# RisuAI Network

RisuAI 서비스들의 네트워크 구성을 관리하는 프로젝트.
Caddy 리버스 프록시 설정과 Docker Compose 오케스트레이션을 통해
sync, with-sqlite, risuai 서비스 간의 연결을 정의한다.

## 프로젝트 성격

이 프로젝트는 코드를 작성하는 프로젝트가 아니라, **인프라 구성 파일**을 관리하는 프로젝트다.
Caddyfile, docker-compose.yml, .env 예시 등이 핵심 산출물이다.

## 의존 관계

이 프로젝트는 아래 서비스들의 구성에 영향을 받는다. 해당 서비스의 변경 시 이 프로젝트도 확인·수정해야 한다:

- **RisuAI 본체** (`Risuai/`): Dockerfile, 노출 포트, 도메인(`SITE_DOMAIN`) 변경 시
- **Sync 서버** (`risu-files/custom-codes/sync/`): 리슨 포트, UPSTREAM 인터페이스 변경 시
- **DB Proxy** (`risu-files/custom-codes/with-sqlite/`): 리슨 포트, UPSTREAM 인터페이스, 엔드포인트 추가/제거 시
- **Risuai 루트의 Caddyfile** (`Risuai/Caddyfile`): Caddy 설정이 이 프로젝트와 중복되므로, 어느 한쪽이 변경되면 다른 쪽도 동기화해야 한다

## 개별 서비스 재배포

특정 서비스만 변경했을 때, 해당 서비스만 빌드·재시작하려면 `build`와 `up`을 분리한다.
`up --build`를 쓰면 `depends_on` 체인 때문에 의존 서비스까지 재시작될 수 있다.

```bash
# 예: sync만 재배포
docker compose -f docker-compose.yml build sync && \
docker compose -f docker-compose.yml --profile sync up -d sync
```

### BuildKit 캐시 주의

`docker compose build`는 BuildKit 레이어 캐시를 적극 활용한다.
특히 risuai 본체처럼 빌드 컨텍스트가 큰 서비스는, 파일이 변경되었는데도
캐시된 레이어가 사용되어 빌드 산출물이 반영되지 않는 경우가 있다.

컨테이너가 시작 직후 즉시 종료(exit 0)되거나 예상과 다르게 동작하면,
**`--no-cache`로 풀 빌드**를 시도한다:

```bash
docker compose -f docker-compose.yml build --no-cache risuai
```

## 문서

- 서비스 구성, 포트, 프로필, 환경변수 등이 변경되면 README.md도 함께 업데이트한다.

### 변경 시 체크리스트

- [ ] 서비스의 리슨 포트가 바뀌었는가? → Caddyfile, docker-compose.yml 업데이트
- [ ] 새로운 서비스가 프록시 체인에 추가되었는가? → Caddyfile fallback 순서, docker-compose.yml profiles 업데이트
- [ ] 도메인이 변경되었는가? → Caddyfile site block 업데이트
- [ ] 서비스의 health check 경로가 바뀌었는가? → Caddyfile health_uri 업데이트
