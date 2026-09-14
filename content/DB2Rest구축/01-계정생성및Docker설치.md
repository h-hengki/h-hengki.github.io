# 01. 서비스 계정 생성 및 Docker 설치

대상 서버: `J-L3-PY-HOSTING` (Rocky Linux 10.2)
목표: 전용 계정 1개로 AI Data Gateway 전체(DB2Rest + APISIX + Keycloak)를 구성·운영한다.

> 실행 주체 표기 — `[root]`: root로 실행, `[aigateway]`: 서비스 계정으로 실행

---

## 1단계. 서비스 계정 생성 `[root]`

```bash
# 계정 생성 (홈 디렉터리 자동 생성)
useradd -m -c "AI Data Gateway service account" aigateway

# 비밀번호 설정
passwd aigateway
```

확인:

```bash
id aigateway
# uid=100X(aigateway) gid=100X(aigateway) groups=100X(aigateway)
```

> 계정명은 `aigateway`로 확정 (2026-07-14 생성 완료). 모든 문서에서 이 계정명을 사용한다.

## 2단계. Docker CE 리포지토리 추가 `[root]`

```bash
dnf -y install dnf-plugins-core

# Docker 공식 리포지토리 추가 (Rocky는 centos 리포지토리 사용)
dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```

- 위 `config-manager` 문법이 안 되면(dnf5 환경):
  `dnf config-manager addrepo --from-repofile=https://download.docker.com/linux/centos/docker-ce.repo`
- el10 패키지가 없다는 오류가 나오면 rhel 리포지토리로 대체:
  `https://download.docker.com/linux/rhel/docker-ce.repo`

## 3단계. Docker CE + Compose 설치 `[root]`

```bash
dnf -y install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

서비스 기동 및 부팅 시 자동 시작:

```bash
systemctl enable --now docker
```

확인:

```bash
docker --version          # Docker version 2x.x.x
docker compose version    # Docker Compose version v2.x.x
systemctl is-active docker  # active
```

## 4단계. 서비스 계정에 Docker 사용 권한 부여 `[root]`

```bash
usermod -aG docker aigateway
```

> **참고(보안)**: `docker` 그룹은 사실상 root 권한과 동급이다.
> 이 서버에 다른 일반 사용자가 있으므로, docker 그룹에는 `aigateway` 외의 계정을 추가하지 않는다.

## 5단계. 동작 확인 `[aigateway]`

```bash
su - aigateway     # 또는 aigateway로 ssh 재접속 (그룹 반영은 재로그인 필요)

docker ps          # 권한 오류 없이 빈 목록이 나오면 정상
docker run --rm hello-world   # 이미지 pull + 실행 테스트 (인터넷 연결 확인 겸용)
```

`permission denied` 오류가 나면 로그아웃 후 재로그인한다 (그룹 추가는 새 세션부터 적용).

## 6단계. 작업 디렉터리 구조 생성 `[aigateway]`

전체 스택 설정을 홈 디렉터리 아래 한 곳에 모은다:

```bash
mkdir -p ~/gateway/{apisix/conf,keycloak,db2rest,scripts,backup}
```

```
/home/aigateway/gateway/
├── docker-compose.yml      # 전체 스택 정의 (다음 문서에서 작성)
├── .env                    # 비밀번호 등 환경변수 (권한 600)
├── apisix/conf/            # APISIX 설정 파일
├── keycloak/               # Keycloak 데이터/설정
├── db2rest/                # DB2Rest 설정
├── scripts/                # 운영 스크립트
└── backup/                 # 설정 백업
```

## 트러블슈팅

| 증상 | 원인/조치 |
|---|---|
| `docker ps` → permission denied | 그룹 반영 전. 재로그인 필요 |
| 이미지 pull 실패 (timeout) | 회사망 프록시 확인. 프록시 사용 시 `/etc/systemd/system/docker.service.d/http-proxy.conf` 설정 필요 |
| el10 패키지 없음 | rhel 리포지토리로 교체 (2단계 참고) |
| SELinux 관련 volume 오류 | compose 볼륨 마운트에 `:z` 또는 `:Z` 옵션 추가 (이후 문서에서 반영) |

## 완료 체크리스트

- [x] `aigateway` 계정 생성됨
- [x] `docker --version`, `docker compose version` 정상 출력
- [x] `systemctl is-active docker` = active
- [x] `aigateway` 계정에서 `docker run --rm hello-world` 성공
- [x] `~/gateway/` 디렉터리 구조 생성됨 (2026-07-14 확인)

## 설치 결과 기록 (2026-07-14)

| 패키지 | 버전 |
|---|---|
| docker-ce | 3:29.6.1-1.el10 |
| docker-ce-cli | 1:29.6.1-1.el10 |
| containerd.io | 2.2.6-1.el10 |
| docker-compose-plugin | 5.3.1-1.el10 |
| docker-buildx-plugin | 0.35.0-1.el10 |
| container-selinux | 4:2.246.0-1.el10 |

- centos 리포지토리에서 el10 패키지 정상 설치됨 (rhel 대체 불필요)
- Docker Hub 이미지 pull 정상 (hello-world 성공) → 인터넷 통신 확인
- `aigateway` 계정 docker 그룹 권한 정상 동작

다음 문서: `02-DockerCompose스택구성.md` (DB2Rest + APISIX + Keycloak compose 작성)
