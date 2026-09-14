# 06. API 내역서 (Swagger) 자동 생성

목표: 각 DB의 실제 테이블 목록을 읽어 **테이블별 API 내역서(OpenAPI 스펙)를 자동 생성**하고,
Swagger UI로 사내에 제공한다. 사용자는 브라우저에서 API 목록·컬럼 구조를 보고,
토큰을 넣으면(Authorize 버튼) 화면에서 바로 호출 테스트까지 가능하다.

```
[생성 스크립트] DB들의 information_schema 조회 → DB별 스펙 생성 (specs/smsp.json ...)
      ↓
[swagger-ui 컨테이너] 드롭다운으로 DB 선택·표시  ←  APISIX(/apidocs) 경유로 사내 공개
```

- 내장 Swagger(`/swagger-ui`)와의 차이: 내장은 제네릭 경로만 표시. 이 방식은
  **dbId/테이블 단위 실제 경로 + 컬럼 스키마**가 전부 나열된다.
- 테이블이 추가/변경되면 스크립트만 재실행하면 된다 (cron 등록 가능).

> 실행 주체: `[aigateway]`, 위치 `~/gateway/`. Oracle(mes)은 psql 미사용으로 이번 버전에서 제외
> (필요 시 수동으로 spec에 추가하거나 추후 스크립트 확장).

## 1단계. 스펙 생성 스크립트 작성

**(1) `~/gateway/scripts/gen_openapi.sh`** — DB별 테이블/컬럼 수집:

```bash
mkdir -p ~/gateway/apidocs ~/gateway/scripts
cat > ~/gateway/scripts/gen_openapi.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail
cd "$(dirname "$0")/.."
source .env
mkdir -p apidocs/tmp && rm -f apidocs/tmp/*.csv

# dbId|URL|USER|PASSWORD|SCHEMA
ENTRIES="
smsp|$SMSP_URL|$SMSP_USER|$SMSP_PASSWORD|l2melt
sccm|$SCCM_URL|$SCCM_USER|$SCCM_PASSWORD|l2ccm
lmsp|$LMSP_URL|$LMSP_USER|$LMSP_PASSWORD|l2melt2
lccm|$LCCM_URL|$LCCM_USER|$LCCM_PASSWORD|l2btccm
slbm|$SLBM_URL|$SLBM_USER|$SLBM_PASSWORD|l2lbm
ssbm|$SSBM_URL|$SSBM_USER|$SSBM_PASSWORD|l2sbm
ssbmfce|$SSBMFCE_URL|$SSBMFCE_USER|$SSBMFCE_PASSWORD|l2sbmfce
bica|$BICA_URL|$BICA_USER|$BICA_PASSWORD|bica
sicb|$SICB_URL|$SICB_USER|$SICB_PASSWORD|sicb
sici|$SICI_URL|$SICI_USER|$SICI_PASSWORD|sicinsp
"

echo "$ENTRIES" | while IFS='|' read -r ID URL USER PW SCHEMA; do
  [ -z "$ID" ] && continue
  HOSTDB=${URL#jdbc:postgresql://}
  HOST=${HOSTDB%%:*}; REST=${HOSTDB#*:}; PORT=${REST%%/*}; DB=${REST#*/}
  echo ">> $ID ($HOST/$DB schema=$SCHEMA)"
  docker run --rm postgres:16 psql \
    "host=$HOST port=$PORT user=$USER password=$PW dbname=$DB" -tA -c \
    "select table_name||','||column_name||','||data_type
       from information_schema.columns
      where table_schema='"$SCHEMA"'
      order by table_name, ordinal_position" > "apidocs/tmp/${ID}.csv"
done

python3 scripts/build_spec.py
echo "생성 완료: apidocs/specs/*.json (DB별 스펙)"
EOF
chmod +x ~/gateway/scripts/gen_openapi.sh
```

**(2) `~/gateway/scripts/build_spec.py`** — 수집 결과를 OpenAPI 3.0 스펙으로 변환:

```bash
cat > ~/gateway/scripts/build_spec.py <<'EOF'
import csv, json, glob, os

GATEWAY = "http://172.31.209.71:9080/api"
TYPEMAP = {
    "integer": ("integer", None), "bigint": ("integer", None), "smallint": ("integer", None),
    "numeric": ("number", None), "double precision": ("number", None), "real": ("number", None),
    "boolean": ("boolean", None),
    "timestamp without time zone": ("string", "date-time"),
    "timestamp with time zone": ("string", "date-time"),
    "date": ("string", "date"),
}

COMMON_PARAMS = [
    {"name": "fields", "in": "query", "schema": {"type": "string"},
     "description": "조회 컬럼 (콤마 구분, 기본 전체)"},
    {"name": "filter", "in": "query", "schema": {"type": "string"},
     "description": "RSQL 조건. 예: use_yn==Y;seq=gt=100 (';'=AND ','=OR)"},
    {"name": "sort", "in": "query", "schema": {"type": "string"},
     "description": "정렬. 예: in_date;desc"},
    {"name": "limit", "in": "query", "schema": {"type": "integer"},
     "description": "조회 건수 (기본 100, 0 금지)"},
    {"name": "offset", "in": "query", "schema": {"type": "integer"},
     "description": "건너뛸 행 수 (sort와 함께 사용)"},
]

OUT = "apidocs/specs"
os.makedirs(OUT, exist_ok=True)

DESC = ("사내 공용 데이터 조회 API. 토큰 발급 후 Authorize 버튼에 입력하면 화면에서 직접 호출 가능. "
        "토큰 발급: POST http://172.31.209.71:8180/realms/aigateway/protocol/openid-connect/token "
        "(grant_type=client_credentials, client_id/secret은 사용자별 발급)")

total = 0
for f in sorted(glob.glob("apidocs/tmp/*.csv")):
    db = os.path.splitext(os.path.basename(f))[0]
    tables = {}
    with open(f, newline="", encoding="utf-8") as fh:
        for row in csv.reader(fh):
            if len(row) < 3:
                continue
            t, c, dt = row[0], row[1], ",".join(row[2:])
            typ, fmt = TYPEMAP.get(dt.strip(), ("string", None))
            prop = {"type": typ}
            if fmt:
                prop["format"] = fmt
            tables.setdefault(t, {})[c] = prop
    if not tables:
        continue
    paths = {}
    for t, cols in sorted(tables.items()):
        item_schema = {"type": "object", "properties": cols}
        paths[f"/{db}/{t}"] = {"get": {
            "summary": f"{t} 목록 조회",
            "parameters": COMMON_PARAMS,
            "security": [{"bearerAuth": []}],
            "responses": {"200": {"description": "OK", "content": {"application/json": {
                "schema": {"type": "array", "items": item_schema}}}},
                "401": {"description": "토큰 없음/만료"}},
        }}
        paths[f"/{db}/{t}/count"] = {"get": {
            "summary": f"{t} 건수",
            "parameters": [COMMON_PARAMS[1]],
            "security": [{"bearerAuth": []}],
            "responses": {"200": {"description": "OK"}},
        }}
    spec = {
        "openapi": "3.0.3",
        "info": {"title": f"{db} API 내역서 ({len(tables)}개 테이블)",
                 "description": DESC, "version": "1.0.0"},
        "servers": [{"url": GATEWAY}],
        "paths": paths,
        "components": {"securitySchemes": {"bearerAuth": {
            "type": "http", "scheme": "bearer", "bearerFormat": "JWT"}}},
    }
    with open(f"{OUT}/{db}.json", "w", encoding="utf-8") as fh:
        json.dump(spec, fh, ensure_ascii=False)
    total += len(paths)
    print(f"{db}: 테이블 {len(tables)}개")
print(f"총 paths: {total} → {OUT}/*.json")
EOF
```

**(3) 실행:**

```bash
~/gateway/scripts/gen_openapi.sh
# >> smsp (...) ... smsp: 테이블 NN개 ... 총 paths: NNN → apidocs/specs/*.json
```

## 2단계. Swagger UI 컨테이너 추가

docker-compose.yml에 서비스 추가:

DB별 스펙 파일(specs/*.json)을 서빙하고, 우측 상단 드롭다운("Select a definition")으로
DB를 전환할 수 있게 `URLS`로 목록을 등록한다:

```yaml
  apidocs:
    image: swaggerapi/swagger-ui
    restart: unless-stopped
    environment:
      BASE_URL: /apidocs
      URLS: >-
        [{url:"specs/smsp.json",name:"smsp"},
         {url:"specs/sccm.json",name:"sccm"},
         {url:"specs/lmsp.json",name:"lmsp"},
         {url:"specs/lccm.json",name:"lccm"},
         {url:"specs/slbm.json",name:"slbm"},
         {url:"specs/ssbm.json",name:"ssbm"},
         {url:"specs/ssbmfce.json",name:"ssbmfce"},
         {url:"specs/bica.json",name:"bica"},
         {url:"specs/sicb.json",name:"sicb"},
         {url:"specs/sici.json",name:"sici"}]
    volumes:
      - ./apidocs/specs:/usr/share/nginx/html/specs:ro
    networks: [gateway]
```

> DB를 추가/제외하면 `URLS` 목록도 함께 수정한다. 이전의 통합 1파일 방식(`SWAGGER_JSON`)을
> 쓰다 전환하는 경우 `SWAGGER_JSON` 환경변수와 기존 openapi.json 마운트는 제거할 것.

```bash
cd ~/gateway
docker compose up -d apidocs
```

## 3단계. APISIX 라우트 추가 (사내 공개)

```bash
source .env
curl -s -X PUT http://127.0.0.1:9180/apisix/admin/routes/apidocs \
  -H "X-API-KEY: ${APISIX_ADMIN_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "apidocs",
    "uri": "/apidocs*",
    "methods": ["GET"],
    "upstream": { "type": "roundrobin", "nodes": { "apidocs:8080": 1 } }
  }'
```

접속: **`http://172.31.209.71:9080/apidocs/`** (사무실 브라우저)

- 우측 상단 **"Select a definition" 드롭다운으로 DB 선택** → 해당 DB의 테이블별 API만 표시
- 테이블별 GET API + 응답 컬럼 스키마 확인
- **Authorize** 버튼에 토큰 붙여넣기(`Bearer` 접두어 없이 토큰만) → **Try it out**으로 실호출 가능
  (스펙의 server가 게이트웨이 `/api` 경로라 동일 출처로 호출됨 — CORS 문제 없음. 토큰 만료 5분 시 재발급)

## 갱신 및 운영

- **테이블 변경 시**: `~/gateway/scripts/gen_openapi.sh` 재실행이면 끝 (컨테이너 재시작 불필요 —
  볼륨 마운트라 새로고침하면 반영).
- 주기 자동화(선택): `crontab -e` → `0 6 * * 1 /home/aigateway/gateway/scripts/gen_openapi.sh`
  (매주 월 06:00 갱신)
- Oracle(mes)은 미포함 — 포함하려면 스크립트에 Oracle 조회(ALL_TAB_COLUMNS) 추가 필요.
- 문서 화면을 특정 사용자만 보게 하려면 apidocs 라우트에도 openid-connect 플러그인 추가.

## 완료 체크리스트

- [ ] gen_openapi.sh 실행 → `apidocs/specs/`에 DB별 json 10개 생성
- [ ] apidocs 컨테이너 Up
- [ ] APISIX 라우트(/apidocs) 등록
- [ ] 브라우저에서 내역서 확인 (드롭다운으로 DB 전환)
- [ ] Authorize + Try it out으로 실호출 1건 검증
