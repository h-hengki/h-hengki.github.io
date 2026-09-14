# 04. DB2Rest 조회 문법 가이드 (사내 API 사용자용)

AI Data Gateway의 데이터 조회 API 사용법 — 사내 API 사용자(인이지 AI 서버 등) 공통 배포용.
DB2Rest **v1.6.8** 소스 기준으로 작성 (2026-07-16). client_id/secret은 사용자별로 별도 발급된다 (doc/03).

## 1. 기본 정보

| 항목 | 값 |
|---|---|
| 기본 URL | `http://172.31.209.71:9080/api/{dbId}/{테이블명}` |
| 허용 메서드 | **GET만** (쓰기 메서드는 게이트웨이에서 차단) |
| 인증 | `Authorization: Bearer <토큰>` 헤더 필수 (없으면 401) |
| 응답 형식 | JSON 배열 (한 행 = 한 객체, 컬럼명은 소문자) |
| 기본 조회 건수 | limit 미지정 시 **100건** (defaultFetchLimit) |

**토큰 발급** (만료 300초 — 만료 시 재발급):

```bash
curl -s -X POST "http://172.31.209.71:8180/realms/aigateway/protocol/openid-connect/token" \
  -d "grant_type=client_credentials" \
  -d "client_id=<발급받은 ID>" \
  -d "client_secret=<발급받은 시크릿>"
```

**dbId 목록** (doc/02 참고): `smsp`, `sccm`, `lmsp`, `lccm`, `slbm`, `ssbm`, `ssbmfce`, `bica`, `sicb`, `sici` (+ 테스트용 `sampledb`)

## 2. 엔드포인트 4종

| 용도 | 경로 | 반환 |
|---|---|---|
| 목록 조회 | `GET /api/{dbId}/{table}` | JSON 배열 |
| PK 단건 조회 | `GET /api/{dbId}/{table}/{pk값}` | JSON 객체 |
| 조건 단건 조회 | `GET /api/{dbId}/{table}/one?filter=...` | JSON 객체 (첫 번째 일치 행) |
| 건수 조회 | `GET /api/{dbId}/{table}/count?filter=...` | `{"count": N}` |

> PK 단건 조회는 테이블에 기본키가 정의되어 있어야 동작한다.

## 3. 쿼리 파라미터

### 3.1 `fields` — 조회 컬럼 선택

기본값 `*`(전체). 콤마로 구분해 필요한 컬럼만 지정 — **필요 컬럼만 지정 권장** (전송량·DB 부하 감소):

```
?fields=irn_code,irn_name,in_date
```

### 3.2 `filter` — 조건 (RSQL 문법)

`컬럼연산자값` 형태. 조건 연결: `;` = AND, `,` = OR. 괄호로 그룹핑 가능.

**비교 연산자:**

| 연산자 | 의미 | 예 |
|---|---|---|
| `==` | 같음 | `use_yn==Y` |
| `!=` | 다름 | `use_yn!=N` |
| `=gt=` / `=ge=` | 초과 / 이상 | `param_value=gt=1600` |
| `=lt=` / `=le=` | 미만 / 이하 | `seq=le=100` |
| `=in=` | 목록 포함 | `irn_larg=in=(A,B,C)` |
| `=out=` / `=notin=` | 목록 제외 | `scrap_gbn=out=(X,Z)` |

**문자열/NULL 연산자:**

| 연산자 | 의미 | 예 |
|---|---|---|
| `=like=` | 부분 일치 | `irn_name=like=CW*` (와일드카드 `*`) |
| `=notlike=` / `=nk=` | 부분 불일치 | `irn_name=nk=TEST*` |
| `=startWith=` | ~로 시작 | `irn_code=startWith=AAB` |
| `=endWith=` | ~로 끝남 | `irn_code=endWith=218` |
| `=isnull=` / `=na=` | NULL임 | `remark=isnull=true` |
| `=isnotnull=` / `=nn=` | NULL 아님 | `in_date=nn=true` |

**날짜/시간**: ISO 8601 문자열로 비교:

```
?filter=in_date=ge=2026-07-01T00:00:00;in_date=lt=2026-07-16T00:00:00
```

**복합 조건 예** — (A그룹이면서 사용중) 또는 VD 대상:

```
?filter=(irn_larg==A;use_yn==Y),vd_yn==Y
```

### 3.3 `sort` — 정렬

`컬럼;방향` 형식 (방향 생략 시 ASC). 여러 컬럼은 sort 파라미터를 반복:

```
?sort=in_date;desc
?sort=irn_larg&sort=seq;desc          # irn_larg ASC, seq DESC
```

> 구분자가 세미콜론(`;`)이므로 URL 인코딩이 필요한 클라이언트에서는 `%3B`로 인코딩:
> `?sort=in_date%3Bdesc`

### 3.4 `limit` / `offset` — 페이징

| 파라미터 | 규칙 |
|---|---|
| `limit` | 양수 = 해당 건수. 미지정/-1 = 기본 100건. **0은 오류** |
| `offset` | 0 이상 = 건너뛸 행 수. 미지정/-1 = 처음부터 |

```
?sort=in_date;desc&limit=50&offset=100     # 최신순 3페이지 (페이지당 50건)
```

> ⚠️ offset 페이징은 반드시 `sort`와 함께 사용할 것 — 정렬 없는 offset은 순서가 보장되지 않는다.
> 대량 수집 시에는 offset 증가 방식보다 `filter=in_date=gt=<마지막수집시각>` 방식(증분 수집)을 권장.

## 4. 예제 모음

```bash
BASE="http://172.31.209.71:9080/api"
AUTH="Authorization: Bearer ${TOKEN}"

# 1) 전체 조회 (기본 100건)
curl -s "$BASE/smsp/smsp1010" -H "$AUTH"

# 2) 컬럼 선택 + 조건 + 정렬 + 5건만
curl -s "$BASE/smsp/smsp1010?fields=irn_code,irn_name,in_date&filter=use_yn==Y&sort=in_date;desc&limit=5" -H "$AUTH"

# 3) 기간 조회 (증분 수집 패턴)
curl -s "$BASE/smsp/smsp1010?filter=up_date=gt=2026-07-01T00:00:00&sort=up_date&limit=1000" -H "$AUTH"

# 4) IN 조건 + OR 조합
curl -s "$BASE/smsp/smsp1010?filter=(irn_larg=in=(A,B);use_yn==Y),vd_yn==Y" -H "$AUTH"

# 5) 건수 확인 (수집 전 데이터량 파악)
curl -s "$BASE/smsp/smsp1010/count?filter=use_yn==Y" -H "$AUTH"

# 6) 단건 조회
curl -s "$BASE/smsp/smsp1010/one?filter=irn_code==AAB218" -H "$AUTH"
```

## 5. 주의사항

- **GET 전용**: 게이트웨이가 GET 외 메서드를 차단한다. DB2Rest의 조인 기능(`/_expand`)은
  POST 방식이라 현재 사용 불가 — 조인이 필요하면 게이트웨이 운영팀(VNTG)과 협의.
- **토큰 만료 300초**: 매 요청 전 발급이 아니라, 만료 시(401 응답) 재발급하는 캐싱 방식 권장.
- **스키마 지정 불필요**: DB별 대상 스키마(smsp→l2melt 등)는 게이트웨이에 이미 설정되어 있다.
  `Accept-Profile` 헤더는 쓸 필요 없음.
- **URL 인코딩**: `;`(%3B), `(`(%28), `)`(%29), 공백(%20) 등은 HTTP 클라이언트에 따라
  인코딩이 필요할 수 있다. curl은 대부분 그대로 동작.
- **응답이 `[]`**: 오류가 아니라 조건에 맞는 데이터가 0건이라는 뜻.
- **404 Route Not Found**: 경로 오타(`/api/` 누락 등) 또는 GET 외 메서드 사용.
- **401**: 토큰 누락/만료. 재발급 후 재시도.
- **테이블 목록 확인**: 각 시스템의 테이블 구조는 해당 DB 담당 부서에 확인
  (게이트웨이는 존재하는 테이블을 그대로 노출할 뿐, 스키마 문서를 제공하지 않는다).

## 6. Python 클라이언트 예제

토큰 캐싱(만료 300초 대응) + 401 시 1회 재발급을 포함한 최소 구현. `pip install requests` 필요.

```python
import time
import requests

TOKEN_URL = "http://172.31.209.71:8180/realms/aigateway/protocol/openid-connect/token"
BASE_URL = "http://172.31.209.71:9080/api"
CLIENT_ID = "<발급받은 client_id>"
CLIENT_SECRET = "<발급받은 시크릿>"      # 운영 코드에서는 환경변수/시크릿 저장소로 관리


class GatewayClient:
    def __init__(self):
        self._token = None
        self._expires_at = 0

    def _get_token(self):
        # 만료 30초 전까지는 기존 토큰 재사용 (매 요청마다 발급하지 않음)
        if self._token and time.time() < self._expires_at - 30:
            return self._token
        resp = requests.post(TOKEN_URL, data={
            "grant_type": "client_credentials",
            "client_id": CLIENT_ID,
            "client_secret": CLIENT_SECRET,
        }, timeout=10)
        resp.raise_for_status()
        body = resp.json()
        self._token = body["access_token"]
        self._expires_at = time.time() + body["expires_in"]
        return self._token

    def get(self, db_id, table, **params):
        """예: client.get("smsp", "smsp1010", filter="use_yn==Y", limit=10)"""
        url = f"{BASE_URL}/{db_id}/{table}"
        resp = requests.get(
            url,
            headers={"Authorization": f"Bearer {self._get_token()}"},
            params=params,
            timeout=30,
        )
        if resp.status_code == 401:      # 토큰 만료 직후 경합 — 1회 재발급 후 재시도
            self._token = None
            resp = requests.get(
                url,
                headers={"Authorization": f"Bearer {self._get_token()}"},
                params=params,
                timeout=30,
            )
        resp.raise_for_status()
        return resp.json()
```

사용 예:

```python
client = GatewayClient()

# 조건 + 정렬 + 컬럼 선택 + 건수 제한
rows = client.get(
    "smsp", "smsp1010",
    filter="use_yn==Y;irn_larg==A",
    sort="in_date;desc",
    fields="irn_code,irn_name,in_date",
    limit=10,
)

# 건수 / 단건
cnt = client.get("smsp", "smsp1010/count", filter="use_yn==Y")     # {"count": N}
row = client.get("smsp", "smsp1010/one", filter="irn_code==AAB218")

# pandas DataFrame으로 (AI 학습 데이터 수집)
import pandas as pd
df = pd.DataFrame(client.get("smsp", "smsp1010", limit=1000))

# 증분 수집 패턴 (offset 페이징보다 권장)
last_ts = "2026-07-16T00:00:00"
new_rows = client.get("smsp", "smsp1010",
                      filter=f"up_date=gt={last_ts}", sort="up_date", limit=1000)
```

- `params=` dict로 넘기면 `filter`의 `==`/`;` 등 URL 인코딩을 requests가 자동 처리한다.
- 토큰은 만료 30초 전까지 재사용 — 5절 "토큰 캐싱 권장"의 구현 예.

## 부록. 오류 응답 형태

| HTTP | 의미 | 대응 |
|---|---|---|
| 401 | 토큰 없음/만료/무효 | 토큰 재발급 |
| 404 | 라우트 없음(경로·메서드) 또는 테이블 없음 | URL·dbId·테이블명 확인 |
| 400 | filter/sort/limit 문법 오류 | 파라미터 문법 확인 (limit=0 금지 등) |
| 5xx | 게이트웨이/DB 장애 | VNTG 운영 담당 문의 |
