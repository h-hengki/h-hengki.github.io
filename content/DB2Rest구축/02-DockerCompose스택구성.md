# 02. Docker Compose 스택 구성 (다중 DB)

목표: `~/gateway/` 아래에 docker-compose.yml 하나로 전체 스택을 정의하고 기동한다.
DB2Rest는 처음부터 **다중 DB 구성**으로 설정한다 — 테스트용 sampledb(컨테이너) 1개 +
실환경 PostgreSQL 10대를 함께 등록하고, URL의 dbId로 구분해 호출한다.

> 실행 주체: 별도 표기 없으면 전부 `[aigateway]` 계정, 작업 위치는 `~/gateway/`

## 구성 요약

| 서비스 | 이미지 | 호스트 포트 | 역할 |
|---|---|---|---|
| apisix | apache/apisix:3.13.0-debian | 9080(프록시), 127.0.0.1:9180(Admin API) | API Gateway — 유일한 외부 진입점 |
| etcd | quay.io/coreos/etcd:v3.5.21 | 없음(내부) | APISIX 설정 저장소 |
| keycloak | quay.io/keycloak/keycloak:26.4 | 8180 | 인증 서버 (관리 콘솔 포함) |
| db2rest | kdhrubo/db2rest:latest | 127.0.0.1:8081 (검증용) | DB→REST 변환 (다중 DB) |
| sampledb | postgres:16 | 없음(내부) | 테스트용 DB (흐름 검증용, 유지) |

포트 원칙:
- 외부(다른 서버/PC)에서 접근 가능한 포트는 **9080(APISIX)**, **8180(Keycloak 콘솔)** 두 개뿐
- DB2Rest(8081)와 APISIX Admin API(9180)는 `127.0.0.1` 바인딩 — 서버 내부에서만 접근
- sampledb, etcd는 호스트 포트 자체를 열지 않음 (컨테이너 네트워크 내부 전용)

## 등록 대상 DB 목록

dbId는 접속 계정명(시스템 코드) 기준. API 경로: `GET /v1/rdbms/{dbId}/{테이블명}`

| dbId | IP | Port | User | schemas | 비고 |
|---|---|---|---|---|---|
| sampledb | (컨테이너 내부) | 5432 | sample | (public) | 테스트용 |
| smsp | 172.31.209.166 | 6444 | smsp | l2melt | |
| sccm | 172.31.209.167 | 6444 | sccm | l2ccm | |
| lmsp | 172.31.209.168 | 6444 | lmsp | l2melt2 | (구 lmsp1) |
| lccm | 172.31.209.169 | 6444 | lmsp | l2btccm | (구 lmsp2, 비밀번호 별도) |
| slbm | 172.31.209.170 | 6444 | slbm | l2lbm | |
| ssbm | 172.31.209.171 | 6444 | ssbm | l2sbm | |
| ssbmfce | 172.31.209.172 | 6444 | ssbmfce | l2sbmfce | |
| bica | 172.31.209.173 | 6444 | bica | bica | |
| sicb | 172.31.209.174 | 6444 | sicb | sicb | |
| sici | 172.31.209.175 | 6444 | sici | sicinsp | |
| mes_test | (Oracle 19c) | 1521 | (접수 대기) | — | ORA1_* .env 필요 |

> 2026-07-16 dbId 최종 확정 (서버 databases.yml 기준).
> 2026-07-20 환경변수명 정리: `LMSP1_*`→`LMSP_*`, `LMSP2_*`→`LCCM_*` (dbId와 일치하도록 —
> .env / databases.yml / gen_openapi.sh 3곳 모두 새 이름 기준).

> ⚠️ 확인 필요: ① 각 서버의 **database 이름** (아래 설정은 계정명과 동일하다고 가정 —
> 다르면 `.env`의 `*_URL` 마지막 부분만 수정) ② 당초 11대로 파악되었으나 접수된 것은 10대 —
> 잔여 1대 여부 ③ **Oracle 19c 접속 정보 미접수** — 접수 시 databases.yml에 `type: ORACLE`
> 항목 하나만 추가하면 된다 (하단 주석 참고)

## 0단계. 실환경 DB 통신 사전 점검

Gateway 서버에서 실DB 서버(6444 포트)로 네트워크가 열려 있는지 먼저 확인:

```bash
for ip in 166 167 168 169 170 171 172 173 174 175; do
  timeout 2 bash -c "</dev/tcp/172.31.209.$ip/6444" 2>/dev/null \
    && echo "172.31.209.$ip:6444 OK" || echo "172.31.209.$ip:6444 FAIL"
done
```

`FAIL`이 나오는 서버는 방화벽/ACL 개방 요청이 필요하다 (Gateway 서버 IP → 해당 DB 서버 6444).

## 1단계. 환경변수 파일 작성

`~/gateway/.env` — 스택 공통 + DB 접속 정보 전체. **비밀번호 포함이므로 반드시 `chmod 600`**:

```bash
cat > ~/gateway/.env <<'EOF'
# ── 스택 공통 (비밀번호 3개는 반드시 변경) ──
KEYCLOAK_ADMIN=admin
KEYCLOAK_ADMIN_PASSWORD=ChangeMe_Keycloak!
APISIX_ADMIN_KEY=ChangeMe-ApisixAdminKey-Long-Random
SAMPLE_DB_USER=sample
SAMPLE_DB_PASSWORD=ChangeMe_Sample!

# ── 실환경 PostgreSQL 10대 (DB명은 계정명과 동일 가정 — 확인 후 수정) ──
SMSP_URL=jdbc:postgresql://172.31.209.166:6444/smsp
SMSP_USER=smsp
SMSP_PASSWORD=smsp#2025

SCCM_URL=jdbc:postgresql://172.31.209.167:6444/sccm
SCCM_USER=sccm
SCCM_PASSWORD=sccm#2025

LMSP_URL=jdbc:postgresql://172.31.209.168:6444/lmsp
LMSP_USER=lmsp
LMSP_PASSWORD=lmsp#2025

LCCM_URL=jdbc:postgresql://172.31.209.169:6444/lmsp
LCCM_USER=lmsp
LCCM_PASSWORD=lmsp#2025

SLBM_URL=jdbc:postgresql://172.31.209.170:6444/slbm
SLBM_USER=slbm
SLBM_PASSWORD=slbm#2025

SSBM_URL=jdbc:postgresql://172.31.209.171:6444/ssbm
SSBM_USER=ssbm
SSBM_PASSWORD=ssbm#2025

SSBMFCE_URL=jdbc:postgresql://172.31.209.172:6444/ssbmfce
SSBMFCE_USER=ssbmfce
SSBMFCE_PASSWORD=ssbmfce#2025

BICA_URL=jdbc:postgresql://172.31.209.173:6444/bica
BICA_USER=bica
BICA_PASSWORD=bica#2025

SICB_URL=jdbc:postgresql://172.31.209.174:6444/sicb
SICB_USER=sicb
SICB_PASSWORD=sicb#2025

SICI_URL=jdbc:postgresql://172.31.209.175:6444/sici
SICI_USER=sici
SICI_PASSWORD=sici#2025

# ── Oracle 19c (접속 정보 접수 시 주석 해제 후 수정) ──
#ORA1_URL=jdbc:oracle:thin:@<IP>:1521/<SERVICE_NAME>
#ORA1_USER=
#ORA1_PASSWORD=
EOF
chmod 600 ~/gateway/.env
```

> 참고: db2rest 컨테이너에 `env_file: .env`로 전체가 전달된다. 스택 공통 변수까지 함께
> 들어가지만 동일 운영자가 관리하는 단일 스택이므로 허용. 분리가 필요해지면
> `db2rest/db.env`로 DB 접속 정보만 분리한다.

## 2단계. APISIX 설정 파일 작성

`~/gateway/apisix/conf/config.yaml`:

```bash
cat > ~/gateway/apisix/conf/config.yaml <<'EOF'
apisix:
  node_listen: 9080

deployment:
  role: traditional
  role_traditional:
    config_provider: etcd
  admin:
    allow_admin:
      - 0.0.0.0/0        # Admin API는 127.0.0.1 바인딩이므로 실질 접근은 서버 내부만 가능
    admin_key:
      - name: admin
        key: "${{APISIX_ADMIN_KEY}}"
        role: admin
  etcd:
    host:
      - "http://etcd:2379"
    prefix: /apisix
    timeout: 30
EOF
```

> `${{APISIX_ADMIN_KEY}}`는 APISIX가 컨테이너 환경변수에서 읽는 문법 (compose에서 주입).

## 3단계. 샘플 DB 초기화 SQL 작성

`~/gateway/db2rest/init.sql` — AI 학습용 공정 파라미터를 흉내낸 테스트 테이블:

```bash
cat > ~/gateway/db2rest/init.sql <<'EOF'
CREATE TABLE process_param (
    id           BIGSERIAL PRIMARY KEY,
    equipment_id VARCHAR(20)  NOT NULL,
    param_name   VARCHAR(50)  NOT NULL,
    param_value  NUMERIC(12,4) NOT NULL,
    collected_at TIMESTAMP    NOT NULL DEFAULT now()
);

INSERT INTO process_param (equipment_id, param_name, param_value, collected_at) VALUES
('EAF-01', 'temperature',  1620.5000, now() - interval '10 min'),
('EAF-01', 'power_kw',     42000.0000, now() - interval '10 min'),
('EAF-01', 'o2_flow',      1850.2500, now() - interval '5 min'),
('LF-02',  'temperature',  1585.0000, now() - interval '3 min'),
('LF-02',  'ar_flow',      120.7500,  now());
EOF
```

## 4단계. DB2Rest 다중 DB 설정 파일 작성

`~/gateway/db2rest/databases.yml` — 등록 DB 목록. 접속 정보는 전부 `.env` 참조:

```bash
cat > ~/gateway/db2rest/databases.yml <<'EOF'
app:
  databases:
    - id: sampledb
      type: POSTGRESQL
      url: jdbc:postgresql://sampledb:5432/sampledb
      username: ${SAMPLE_DB_USER}
      password: ${SAMPLE_DB_PASSWORD}
      maxConnections: 5
    - id: smsp
      type: POSTGRESQL
      url: ${SMSP_URL}
      username: ${SMSP_USER}
      password: ${SMSP_PASSWORD}
      maxConnections: 5
      schemas: [l2melt]
    - id: sccm
      type: POSTGRESQL
      url: ${SCCM_URL}
      username: ${SCCM_USER}
      password: ${SCCM_PASSWORD}
      maxConnections: 5
      schemas: [l2ccm]
    - id: lmsp
      type: POSTGRESQL
      url: ${LMSP_URL}
      username: ${LMSP_USER}
      password: ${LMSP_PASSWORD}
      maxConnections: 5
      schemas: [l2melt2]
    - id: lccm
      type: POSTGRESQL
      url: ${LCCM_URL}
      username: ${LCCM_USER}
      password: ${LCCM_PASSWORD}
      maxConnections: 5
      schemas: [l2btccm]
    - id: slbm
      type: POSTGRESQL
      url: ${SLBM_URL}
      username: ${SLBM_USER}
      password: ${SLBM_PASSWORD}
      maxConnections: 5
      schemas: [l2lbm]
    - id: ssbm
      type: POSTGRESQL
      url: ${SSBM_URL}
      username: ${SSBM_USER}
      password: ${SSBM_PASSWORD}
      maxConnections: 5
      schemas: [l2sbm]
    - id: ssbmfce
      type: POSTGRESQL
      url: ${SSBMFCE_URL}
      username: ${SSBMFCE_USER}
      password: ${SSBMFCE_PASSWORD}
      maxConnections: 5
      schemas: [l2sbmfce]
    - id: bica
      type: POSTGRESQL
      url: ${BICA_URL}
      username: ${BICA_USER}
      password: ${BICA_PASSWORD}
      maxConnections: 5
      schemas: [bica]
    - id: sicb
      type: POSTGRESQL
      url: ${SICB_URL}
      username: ${SICB_USER}
      password: ${SICB_PASSWORD}
      maxConnections: 5
      schemas: [sicb]
    - id: sici
      type: POSTGRESQL
      url: ${SICI_URL}
      username: ${SICI_USER}
      password: ${SICI_PASSWORD}
      maxConnections: 5
      schemas: [sicinsp]
#    - id: mes_test                 # Oracle 19c — .env에 ORA1_* 설정 후 주석 해제
#      type: ORACLE                 # (미설정 상태로 해제하면 placeholder 오류로 기동 실패)
#      url: ${ORA1_URL}
#      username: ${ORA1_USER}
#      password: ${ORA1_PASSWORD}
#      maxConnections: 5
EOF
chmod 600 ~/gateway/db2rest/databases.yml
```

### 스키마 지정 (schemas)

각 DB 항목에 `schemas`를 지정하면 해당 스키마의 테이블만 스캔·노출한다:

```yaml
    - id: smsp
      type: POSTGRESQL
      url: ${SMSP_URL}
      username: ${SMSP_USER}
      password: ${SMSP_PASSWORD}
      maxConnections: 5
      schemas:
        - l2melt
```

- 미지정 시 계정이 접근 가능한 **모든 스키마**를 스캔 — 기동 느려짐, 불필요한 테이블 노출,
  스키마 간 동명 테이블 충돌(마지막 것만 인식) 위험.
- 지정 시 `Accept-Profile` 헤더 없이 테이블명만으로 조회 가능.
- 확인된 스키마: smsp = `l2melt` (2026-07-16). 나머지 DB는 스키마 조사 후 반영.
- 스키마 조사 쿼리: `select nspname from pg_namespace where nspname not like 'pg_%' and nspname <> 'information_schema';`

## 5단계. docker-compose.yml 작성

`~/gateway/docker-compose.yml`:

```bash
cat > ~/gateway/docker-compose.yml <<'EOF'
name: ai-data-gateway

services:
  etcd:
    image: quay.io/coreos/etcd:v3.5.21
    restart: unless-stopped
    command:
      - etcd
      - --name=etcd
      - --data-dir=/etcd-data
      - --advertise-client-urls=http://etcd:2379
      - --listen-client-urls=http://0.0.0.0:2379
    volumes:
      - etcd_data:/etcd-data
    networks: [gateway]

  apisix:
    image: apache/apisix:3.13.0-debian
    restart: unless-stopped
    depends_on: [etcd]
    environment:
      APISIX_ADMIN_KEY: ${APISIX_ADMIN_KEY}
    ports:
      - "9080:9080"
      - "127.0.0.1:9180:9180"
    volumes:
      - ./apisix/conf/config.yaml:/usr/local/apisix/conf/config.yaml:ro
    networks: [gateway]

  keycloak:
    image: quay.io/keycloak/keycloak:26.4
    restart: unless-stopped
    command: start-dev
    environment:
      KC_BOOTSTRAP_ADMIN_USERNAME: ${KEYCLOAK_ADMIN}
      KC_BOOTSTRAP_ADMIN_PASSWORD: ${KEYCLOAK_ADMIN_PASSWORD}
    ports:
      - "8180:8080"
    volumes:
      - keycloak_data:/opt/keycloak/data
    networks: [gateway]

  db2rest:
    image: kdhrubo/db2rest:latest
    restart: unless-stopped
    depends_on: [sampledb]
    # 한국어 문자셋(KO16MSWIN949) Oracle 지원: orai18n.jar를 loader.path로 로딩
    # (orai18n 불필요 시 command 줄과 orai18n.jar 볼륨 줄은 생략 가능)
    command: ["java", "-cp", "/opt/app/db2rest.jar", "-Dloader.path=/opt/oracle", "org.springframework.boot.loader.launch.PropertiesLauncher"]
    env_file: .env
    environment:
      SPRING_CONFIG_ADDITIONAL_LOCATION: "file:/config/"
    volumes:
      - ./db2rest/databases.yml:/config/application.yml:ro
      - ./db2rest/orai18n.jar:/opt/oracle/orai18n.jar:ro
    ports:
      - "127.0.0.1:8081:8080"
    networks: [gateway]

  sampledb:
    image: postgres:16
    restart: unless-stopped
    environment:
      POSTGRES_DB: sampledb
      POSTGRES_USER: ${SAMPLE_DB_USER}
      POSTGRES_PASSWORD: ${SAMPLE_DB_PASSWORD}
    volumes:
      - sampledb_data:/var/lib/postgresql/data
      - ./db2rest/init.sql:/docker-entrypoint-initdb.d/init.sql:ro
    networks: [gateway]

networks:
  gateway:

volumes:
  etcd_data:
  keycloak_data:
  sampledb_data:
EOF
```

> Keycloak은 검증 단계라 `start-dev`(내장 H2 DB) 모드. 운영 전환 시 production 모드 + 외부 DB로
> 변경한다 (별도 문서에서 다룸).

## 6단계. 방화벽 포트 개방 `[root]`

```bash
firewall-cmd --permanent --add-port=9080/tcp   # APISIX 프록시
firewall-cmd --permanent --add-port=8180/tcp   # Keycloak 관리 콘솔
firewall-cmd --reload
firewall-cmd --list-ports
```

## 7단계. 스택 기동

```bash
cd ~/gateway
docker compose up -d
docker compose ps        # 5개 서비스 모두 Up 확인 (최초 이미지 pull에 수 분 소요)

# DB2Rest 기동 로그에서 11개 DB 연결 확인 (DB 수가 많아 기동에 시간이 걸릴 수 있음)
docker compose logs -f db2rest | grep -iE "database|datasource|connect|error"
```

## 8단계. 서비스별 동작 확인

```bash
# 1) APISIX 프록시 — 라우트가 없으므로 404가 정상
curl -i http://127.0.0.1:9080/
# HTTP/1.1 404 ... {"error_msg":"404 Route Not Found"}

# 2) APISIX Admin API — 인증키로 라우트 목록 조회 (빈 목록 정상)
source ~/gateway/.env
curl -s http://127.0.0.1:9180/apisix/admin/routes -H "X-API-KEY: ${APISIX_ADMIN_KEY}"

# 3) Keycloak — 관리 콘솔 (사무실 PC 브라우저에서)
#    http://<서버IP>:8180  → admin / .env의 비밀번호로 로그인

# 4) DB2Rest 헬스체크
curl -s http://127.0.0.1:8081/actuator/health
# {"status":"UP"}

# 5) 다중 DB 조회 검증 — dbId로 구분해서 호출
# 테스트 DB
curl -s "http://127.0.0.1:8081/v1/rdbms/sampledb/process_param" | head -40
# → init.sql로 넣은 5건이 JSON 배열로 반환되면 성공

# 실환경 DB (테이블명은 각 시스템 담당자에게 확인한 것으로 교체)
curl -s "http://127.0.0.1:8081/v1/rdbms/smsp/<테이블명>" | head -40
curl -s "http://127.0.0.1:8081/v1/rdbms/sicb/<테이블명>" | head -40
```

## 참고사항

- **dbId 명명**: 접속 계정명(시스템 코드) 기준으로 확정 — smsp, sccm, lmsp1, lmsp2, slbm,
  ssbm, ssbmfce, bica, sicb, sici. AI 서버(인이지) 쪽에 API 경로 규칙으로 공유할 것.
- **DB 계정 권한**: 제공받은 계정이 읽기 전용인지 확인 권장. DB2Rest는 기본적으로
  쓰기(INSERT/UPDATE/DELETE) API도 노출하므로, 쓰기 권한이 있는 계정이면 이후
  APISIX 라우트에서 GET만 허용하도록 제한한다 (doc/03에서 처리).
- **기동 시간**: DB 11개 메타데이터 로딩으로 기동이 느릴 수 있다. 로그로 완료 확인.
- **Oracle 19c**: 접속 정보 접수 시 `.env`와 `databases.yml`의 주석 항목을 해제하고 재기동
  (`docker compose up -d db2rest --force-recreate`). DB2Rest는 Oracle 공식 지원.
- **테이블명 충돌**: 다른 DB에 같은 테이블명이 있어도 dbId로 구분되므로 문제없다.
  같은 DB 안의 다중 스키마는 `Accept-Profile` 헤더로 스키마 지정.
- **일부 DB 접속 실패 시**: 특정 DB만 연결 실패해도 나머지는 동작한다. 실패 DB는
  0단계 통신 점검 → 계정/DB명 확인 순으로 조치.

## 트러블슈팅

| 증상 | 조치 |
|---|---|
| `bitnami/etcd not found` | Bitnami 무료 이미지 제공 중단(2025)으로 발생. `quay.io/coreos/etcd` 공식 이미지 사용 (현재 compose에 반영됨) |
| db2rest `maxPoolSize cannot be less than 1` 로 재시작 반복 | databases.yml의 각 DB 항목에 `maxConnections` 누락. 항목마다 `maxConnections: 5` 추가 (생략 시 0으로 처리되어 기동 실패) |
| 컨테이너가 Restarting 반복 | `docker compose logs <서비스명>` 으로 원인 확인 |
| apisix 기동 실패 (etcd 연결) | etcd가 먼저 healthy 되어야 함. `docker compose restart apisix` |
| db2rest → sampledb 연결 실패 | sampledb 기동 완료 전 접속 시도. `docker compose restart db2rest` |
| db2rest → 실DB 연결 실패 | 0단계 통신 점검, DB명/계정 확인. 필요 시 해당 DB 서버 방화벽에 Gateway 서버 IP 개방 요청 |
| Oracle `ORA-17056: Non-supported character set (KO16MSWIN949)` | 한국어 문자셋 Oracle 연결 시 발생. `orai18n.jar`(Maven Central `com/oracle/database/nls/orai18n` 23.x) 다운로드 → `/opt/oracle`에 마운트 + PropertiesLauncher `command`로 로딩 (compose 표준본에 반영됨, 2026-07-20 bestmes 조치). ⚠️ `-Xbootclasspath` 방식은 JDK21에서 `NoClassDefFoundError: java/sql/SQLException` 유발 — 사용 금지 |
| compose 수정이 반영 안 됨 (COMMAND가 그대로) | `docker compose restart`는 설정 변경을 반영하지 않음 — 반드시 `docker compose up -d --force-recreate <서비스>`. 적용 확인: `docker inspect <컨테이너> --format '{{json .Config.Cmd}}'` |
| 9080/8180 외부 접근 불가 | 방화벽 확인(6단계), 회사망 방화벽/보안그룹의 서버 인바운드 정책 확인 |
| init.sql 반영 안 됨 | 최초 볼륨 생성 시에만 실행됨. `docker compose down -v` 후 재기동 (데이터 삭제 주의) |

## 완료 체크리스트

- [x] 0단계 통신 점검 — 실DB 10대 6444 포트 도달 확인 (2026-07-16, 전부 OK)
- [x] `.env` 작성 (공통 비밀번호 3개 변경) + 권한 600
- [x] `databases.yml` 작성 (PG 11개 등록 + 스키마 지정) + 권한 600
- [x] `docker compose ps` 5개 서비스 모두 Up
- [x] APISIX 프록시(9080) 404 응답 확인
- [x] APISIX Admin API(9180) 인증키 동작 확인
- [x] Keycloak 콘솔(8180) 로그인 성공 (2026-07-16, `http://aigateway.l2:8180` — 서버 DNS 별칭 확인)
      ※ "temporary admin user" 경고 → doc/03에서 영구 관리자 계정 생성 후 임시 계정 삭제
- [x] `/v1/rdbms/sampledb/process_param` JSON 5건 조회 성공 (2026-07-16)
- [x] 실환경 DB 조회 성공 — `/v1/rdbms/smsp/smsp1010` 실데이터 JSON 반환 확인 (2026-07-16)
- [ ] Oracle(mes_test) 연결 — **보류**: 172.17.40.185:1521 방화벽 미개방(ORA-12170), 개방 요청 후 주석 해제

## 구축 이력 (2026-07-14 ~ 07-16)

| 날짜 | 이슈 | 조치 |
|---|---|---|
| 07-14 | bitnami/etcd 이미지 소멸 | quay.io/coreos/etcd:v3.5.21로 교체 |
| 07-14 | db2rest `maxPoolSize cannot be less than 1` | 각 DB 항목에 `maxConnections: 5` 추가 |
| 07-16 | db2rest SASL 인증 실패 | 원인: lmsp2(169) 비밀번호 상이. 신규 비밀번호 수령·적용 |
| 07-16 | smsp 테이블이 l2melt 스키마에 존재 | 전체 DB에 `schemas` 지정 + dbId 확정 (lmsp/lccm 등) |
| 07-16 | Oracle ORA-12170 (172.17.40.185:1521 타임아웃) | 방화벽 미개방. mes_test 주석 처리로 배제, 개방 요청 예정 |

다음 문서: `03-APISIX라우트및Keycloak인증연동.md` (라우트 등록 → Keycloak 클라이언트 생성 → 토큰 인증 연동)
