# 03. APISIX 라우트 등록 및 Keycloak 인증 연동

목표: **사내 API 사용자**(인이지 AI 서버 포함, 그 외 사내 시스템)가 사용할 정식 호출 경로를 완성한다.

```
API 사용자 ──(Bearer 토큰)──▶ APISIX :9080 ──▶ DB2Rest :8080 ──▶ 실환경 DB
(인이지 등)                        │
                                  └─ 토큰 검증 ◀─── Keycloak :8180
```

> 사용자(시스템)마다 Keycloak **클라이언트를 하나씩 발급**한다. APISIX 라우트는 하나로 충분 —
> `aigateway` realm에서 발급된 유효한 토큰이면 어떤 클라이언트든 통과하며, 누가 호출했는지는
> 토큰의 client_id(azp)로 로그에서 구분된다.

- 완성 후: `GET http://172.31.209.71:9080/api/{dbId}/{table}` (토큰 필수, GET만 허용)
- 토큰 없는 호출, GET 외 메서드는 APISIX가 차단 — DB2Rest(8081)는 계속 내부 전용

> 실행 주체: `[aigateway]` 계정, 작업 위치 `~/gateway/`. 브라우저 작업은 사무실 PC.
> **기준 주소는 IP `172.31.209.71`** (사내 DNS `aigateway.l2`는 브라우저 접속용으로만 사용 —
> DNS를 못 쓰는 호출자가 있어 토큰/API 주소는 IP로 통일, 3단계 참고).

## 1단계. Keycloak 보안 정비 (임시 admin 제거)

콘솔 상단 경고("You are logged in as a temporary admin user") 처리.
브라우저에서 `http://aigateway.l2:8180` → admin 로그인 후:

1. 좌측 **Users** → **Add user**
   - Username: `kcadmin` (예시) → **Create**
2. 생성된 사용자 → **Credentials** 탭 → **Set password**
   - 강력한 비밀번호 입력, **Temporary: Off** → Save
3. **Role mapping** 탭 → **Assign role** → Filter를 "Filter by realm roles"로 → **admin** 선택 → Assign
4. 로그아웃 → `kcadmin`으로 재로그인 확인
5. **Users**에서 기존 임시 계정(admin) 삭제

> 이후 `.env`의 KEYCLOAK_ADMIN/KEYCLOAK_ADMIN_PASSWORD는 부트스트랩용이므로 더 이상 사용되지 않는다.

## 2단계. Keycloak 전용 Realm + API 사용자용 클라이언트 생성

master realm은 Keycloak 자체 관리용이므로, 게이트웨이용 realm을 분리한다.

**Realm 생성:**
1. 좌측 상단 realm 선택 드롭다운(현재 "Keycloak" 또는 "master") → **Create realm**
2. Realm name: `aigateway` → **Create**

**클라이언트 생성** (`aigateway` realm에서) — **API 사용자(시스템)마다 아래 절차로 1개씩 발급**.
클라이언트 ID는 사용자를 식별할 수 있게 명명 (예: `api-test`(사내 테스트), `ineeji-ai`(인이지) 등):

1. **Clients** → **Create client**
   - Client type: OpenID Connect / Client ID: `api-test` → Next
2. Capability config:
   - **Client authentication: On** (기밀 클라이언트)
   - Authentication flow: **Service accounts roles만 체크** (Standard flow, Direct access grants 해제)
     → 시스템 간(machine-to-machine) 호출 전용
3. Next → Save
4. 생성된 클라이언트 → **Credentials** 탭 → **Client Secret 복사** (해당 사용자에게 전달할 값)

> 신규 사용자 추가 = 이 절차만 반복하면 끝. APISIX 라우트는 수정할 필요 없다.
> 사용자 차단 = 해당 클라이언트 Disable (콘솔 토글 하나로 즉시 차단).

**토큰 발급 테스트** (서버에서):

```bash
curl -s -X POST "http://172.31.209.71:8180/realms/aigateway/protocol/openid-connect/token" \
  -d "grant_type=client_credentials" \
  -d "client_id=api-test" \
  -d "client_secret=<복사한 시크릿>"
# {"access_token":"eyJhb...","expires_in":300,...} 나오면 성공
```

## 3단계. Keycloak 발급자(issuer) 주소 고정

토큰의 issuer와 APISIX가 검증에 사용할 주소를 일치시키기 위해 Keycloak의 외부 주소를 고정한다.
`docker-compose.yml`의 keycloak 서비스 environment에 추가:

```yaml
  keycloak:
    ...
    environment:
      KC_BOOTSTRAP_ADMIN_USERNAME: ${KEYCLOAK_ADMIN}
      KC_BOOTSTRAP_ADMIN_PASSWORD: ${KEYCLOAK_ADMIN_PASSWORD}
      KC_HOSTNAME: "http://172.31.209.71:8180"    # ← 추가 (2026-07-16 적용)
```

> **기준 주소는 IP(172.31.209.71)로 확정** — 사내 DNS(`aigateway.l2`)를 못 쓰는 외부/타대역
> 호출자도 있기 때문. KC_HOSTNAME을 고정하면 어떤 주소로 토큰을 요청해도 iss가 IP 기준으로
> 통일되어 APISIX 검증과 항상 일치한다. 브라우저 접속은 DNS/IP 아무거나 가능.

적용:

```bash
docker compose up -d --force-recreate keycloak
# 재기동 후 발급 테스트(2단계) 재확인 — 어떤 주소로 요청해도 iss가
# http://172.31.209.71:8180/realms/aigateway 로 고정되면 성공
```

## 4단계. APISIX 라우트 등록 (토큰 검증 + GET 전용)

Admin API(127.0.0.1:9180)로 라우트를 생성한다. 플러그인 구성:
- `openid-connect` — Keycloak 토큰 검증 (bearer_only: 토큰 없으면 401)
- `proxy-rewrite` — 외부 경로 `/api/...` → DB2Rest 내부 경로 `/v1/rdbms/...` 변환
- `methods: GET` — 조회만 허용 (DB 계정에 쓰기 권한이 있어도 게이트웨이에서 차단)

```bash
cd ~/gateway && source .env

curl -s -X PUT http://127.0.0.1:9180/apisix/admin/routes/ai-data-api \
  -H "X-API-KEY: ${APISIX_ADMIN_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "ai-data-api",
    "uri": "/api/*",
    "methods": ["GET"],
    "plugins": {
      "openid-connect": {
        "client_id": "api-test",
        "client_secret": "<2단계에서 복사한 시크릿>",
        "discovery": "http://172.31.209.71:8180/realms/aigateway/.well-known/openid-configuration",
        "bearer_only": true,
        "realm": "aigateway"
      },
      "proxy-rewrite": {
        "regex_uri": ["^/api/(.*)", "/v1/rdbms/$1"]
      }
    },
    "upstream": {
      "type": "roundrobin",
      "nodes": { "db2rest:8080": 1 }
    }
  }'
```

> 등록 확인: `curl -s http://127.0.0.1:9180/apisix/admin/routes -H "X-API-KEY: ${APISIX_ADMIN_KEY}"`

## 5단계. 종단간(End-to-End) 검증

```bash
# 1) 토큰 없이 → 401 Unauthorized 나와야 정상
curl -i "http://172.31.209.71:9080/api/smsp/smsp1010"

# 2) 토큰 발급
TOKEN=$(curl -s -X POST "http://172.31.209.71:8180/realms/aigateway/protocol/openid-connect/token" \
  -d "grant_type=client_credentials" \
  -d "client_id=api-test" \
  -d "client_secret=<시크릿>" | sed 's/.*"access_token":"\([^"]*\)".*/\1/')

# 3) 토큰 포함 → 200 + JSON 데이터
curl -s "http://172.31.209.71:9080/api/smsp/smsp1010" \
  -H "Authorization: Bearer ${TOKEN}" | head -c 500

# 4) GET 외 메서드 → 404/405 차단 확인
curl -i -X DELETE "http://172.31.209.71:9080/api/smsp/smsp1010" \
  -H "Authorization: Bearer ${TOKEN}"
```

## API 사용자 전달 사항

신규 사용자(인이지 등)에게 발급 시 아래 정보를 전달 (client_id/secret만 사용자별로 다름):

| 항목 | 값 |
|---|---|
| 토큰 발급 URL | `http://172.31.209.71:8180/realms/aigateway/protocol/openid-connect/token` |
| grant_type | `client_credentials` |
| client_id / secret | 사용자별 발급 (2단계 절차) — 시크릿은 안전한 채널로 별도 전달 |
| 데이터 API | `GET http://172.31.209.71:9080/api/{dbId}/{table}` + `Authorization: Bearer <token>` |
| dbId 목록 | doc/02 "등록 대상 DB 목록" 표 (smsp, sccm, lmsp, lccm, slbm, ssbm, ssbmfce, bica, sicb, sici) |
| 토큰 만료 | 기본 300초 — 만료 시 재발급 (또는 realm 설정에서 조정) |
| 조회 문법 | [doc/04 DB2Rest 조회 문법 가이드](04-DB2Rest조회문법가이드.md) 함께 전달 |

**발급 현황 관리:**

| client_id | 사용자/용도 | 발급일 | 상태 |
|---|---|---|---|
| test | 초기 검증용 | 2026-07-16 | 시크릿 노출 — 삭제 예정 |
| api-test | 사내 API 사용자 테스트용 | (예정) | |
| (예: ineeji-ai) | 인이지 AI 서버 | (예정) | |

## 트러블슈팅

| 증상 | 조치 |
|---|---|
| 401 (토큰 있는데도) | 토큰 iss와 discovery 주소 불일치 — 3단계 KC_HOSTNAME 적용 여부, discovery URL의 realm 이름 확인 |
| APISIX가 discovery 접근 실패 | apisix 컨테이너에서 `aigateway.l2` DNS 해석 확인: `docker compose exec apisix curl -s http://172.31.209.71:8180/realms/aigateway/.well-known/openid-configuration | head -c 200` |
| 라우트 등록 400 | JSON 문법, X-API-KEY 값 확인 |
| 502 Bad Gateway | upstream `db2rest:8080` 이름/포트 확인 (`docker compose ps`) |

## 완료 체크리스트

- [ ] 영구 관리자 계정 생성 + 임시 admin 삭제 — **확인 필요**
- [x] `aigateway` realm + 클라이언트 생성 (2026-07-16, 검증용 client_id=`test`)
- [x] 토큰 발급 테스트 성공 (2026-07-16)
- [x] KC_HOSTNAME 고정 적용 — `http://172.31.209.71:8180` (2026-07-16)
- [x] APISIX 라우트 등록 (openid-connect + proxy-rewrite + GET 전용) (2026-07-16)
- [x] 토큰 없음 → 401 확인 (2026-07-16)
- [x] 토큰 포함 → smsp1010 실데이터 200 확인 (2026-07-16) — **종단간 검증 완료**
- [x] DELETE 등 쓰기 메서드 차단 확인 (2026-07-16, 404 Route Not Found)
- [ ] 사내 테스트용 클라이언트 `api-test` 신규 생성 (시크릿은 채팅/화면 노출 없이 관리),
      `test` 클라이언트는 시크릿이 노출되었으므로 **삭제**, APISIX 라우트의
      client_id/secret도 `api-test`로 교체
- [ ] 사용자별 클라이언트 발급 체계 운영 — 신규 사용자(인이지 등) 요청 시 2단계 절차로 발급,
      "발급 현황 관리" 표에 기록
