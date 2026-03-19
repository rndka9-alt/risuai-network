# RisuAI Network

RisuAI 서비스들의 네트워크 오케스트레이션.
Caddy 리버스 프록시 + Docker Compose profiles로 서비스 조합을 유동적으로 구성한다.

## 서비스 구성

| 서비스 | 포트 | 역할 |
|--------|------|------|
| risuai | 6001 | RisuAI Node 서버 (항상 실행) |
| with-sqlite | 3001 | SQLite 캐싱 프록시 (선택, profile: sqlite) |
| sync | 3000 | 다중 기기 동기화 프록시 (선택, profile: sync) |
| caddy | 443/80/8080/9090 | TLS 종단 + health-check failover |
| cloudflared | - | Cloudflare Tunnel (항상 실행) |

## 네트워크 구조

### 풀 체인 (sync + sqlite)

```
                          ┌─────────────────────────────────────────┐
                          │              Caddy                      │
Client ──HTTPS──→  :443   │   reverse_proxy (lb_policy first)       │
                          │   ┌─ sync:3000                          │
                          │   ├─ with-sqlite:3001   (fallback)      │
                          │   └─ risuai:6001        (fallback)      │
                          └───────┬─────────────────────────────────┘
                                  │
                          ┌───────▼───────┐
                          │  sync:3000    │
                          │  UPSTREAM=    │
                          │  caddy:8080   │──────────────┐
                          └───────────────┘              │
                                                 ┌───────▼────────┐
                                                 │  Caddy :8080   │
                                                 │  (내부 failover)│
                                                 │  ┌─ sqlite:3001│
                                                 │  └─ risuai:6001│
                                                 └──┬─────────────┘
                                                    │
                                           ┌────────▼────────┐
                                           │ with-sqlite:3001│
                                           │ UPSTREAM=       │
                                           │ risuai:6001     │
                                           └────────┬────────┘
                                                    │
                                           ┌────────▼────────┐
                                           │  risuai:6001    │
                                           │  (Node 서버)    │
                                           └─────────────────┘
```

### 핵심 설계: Caddy 이중 역할

Caddy가 여러 역할을 동시에 수행한다:

| 포트 | 역할 | upstream 목록 |
|------|------|--------------|
| `{$SITE_DOMAIN}` | 외부 HTTPS 진입점 | sync → with-sqlite → risuai (fallback) |
| `:80` | HTTP 진입점 | sync → with-sqlite → risuai (fallback) |
| `:9090` | 내부 관리 진입점 | sync → with-sqlite → risuai (fallback) |
| `:8080` | 내부 failover | with-sqlite → risuai (fallback) |

sync의 UPSTREAM이 `caddy:8080`을 가리키므로, **체인 중간(with-sqlite)이 죽어도 Caddy가 risuai로 failover**한다.
이것이 일반적인 체이닝(`sync → with-sqlite → risuai`)에서는 불가능한 중간 노드 장애 복구를 가능하게 한다.

### Cloudflare Tunnel

`cloudflared` 서비스가 Cloudflare Tunnel을 통해 외부에서 Caddy로 트래픽을 전달한다.
`CLOUDFLARE_TUNNEL_TOKEN` 환경변수(.env)로 터널을 설정한다.

## 가용성 시나리오

| 상황 | 실제 경로 | 손실 |
|------|----------|------|
| 정상 | caddy:443 → sync → caddy:8080 → with-sqlite → risuai | 없음 |
| with-sqlite 죽음 | caddy:443 → sync → caddy:8080 → **risuai** | 캐싱만 손실 |
| sync 죽음 | caddy:443 → **with-sqlite** → risuai | 동기화만 손실 |
| 둘 다 죽음 | caddy:443 → **risuai** | 기본 기능 유지 |
| 죽었던 서비스 복구 | health check 통과 → 자동 복귀 | 수동 개입 불필요 |

모든 경우에서 `{$SITE_DOMAIN}` 접근이 유지된다.

## 사용법

### 명령행 조합

```bash
# 풀 체인: sync → with-sqlite → risuai
SYNC_UPSTREAM=http://caddy:8080 \
  docker compose --profile sync --profile sqlite up -d

# sync만: sync → risuai
docker compose --profile sync up -d

# sqlite만: with-sqlite → risuai
docker compose --profile sqlite up -d

# risuai 직통
docker compose up -d
```

파일 수정 없이 환경변수 + `--profile` 플래그만으로 4가지 조합 전환.

### 프로필 조합표

| 명령 | sync | sqlite | SYNC_UPSTREAM |
|------|------|--------|---------------|
| `--profile sync --profile sqlite` | O | O | `http://caddy:8080` |
| `--profile sync` | O | X | (기본값: `http://risuai:6001`) |
| `--profile sqlite` | X | O | - |
| (없음) | X | X | - |

## 설정 파일

### Caddyfile

```
{
    auto_https disable_redirects
}

{$SITE_DOMAIN} {
    encode zstd gzip { ... }
    reverse_proxy sync:3000 with-sqlite:3001 risuai:6001 {
        lb_policy first
        lb_try_duration 750ms
        lb_try_interval 250ms
        fail_duration 10s
        max_fails 1
    }
}

:80 {
    encode zstd gzip { ... }
    reverse_proxy sync:3000 with-sqlite:3001 risuai:6001 { ... }
}

:9090 {
    reverse_proxy sync:3000 with-sqlite:3001 risuai:6001 { ... }
}

:8080 {
    reverse_proxy with-sqlite:3001 risuai:6001 {
        lb_policy first
        lb_try_duration 750ms
        lb_try_interval 250ms
        fail_duration 10s
        max_fails 1
    }
}
```

| 포트 | 역할 | upstream 목록 |
|------|------|--------------|
| `{$SITE_DOMAIN}` | 외부 HTTPS 진입점 | sync → with-sqlite → risuai |
| `:80` | HTTP 진입점 | sync → with-sqlite → risuai |
| `:9090` | 내부 관리용 진입점 | sync → with-sqlite → risuai |
| `:8080` | 내부 failover (sync의 upstream) | with-sqlite → risuai |

### docker-compose.yml

```yaml
name: risuai

services:
  risuai:
    container_name: risuai
    build:
      context: ${RISUAI_DIR:-../../../}
      dockerfile: Dockerfile
    restart: always
    expose:
      - "6001"
    volumes:
      - ${RISUAI_SAVE_DIR:-risuai-save}:/app/save

  with-sqlite:
    container_name: with-sqlite
    build:
      context: ${SQLITE_DIR:-../with-sqlite}
    profiles: [sqlite]
    restart: always
    expose:
      - "3001"
    environment:
      - PORT=3001
      - UPSTREAM=http://risuai:6001
      - DB_PATH=/data/proxy.db
    volumes:
      - ${SQLITE_DATA_DIR:-sqlite-data}:/data
      - ${RISUAI_SAVE_DIR:-risuai-save}:/risuai-save:ro
    depends_on:
      - risuai

  sync:
    container_name: sync
    build:
      context: ${SYNC_DIR:-../sync}
    profiles: [sync]
    restart: always
    expose:
      - "3000"
    env_file:
      - ${SYNC_DIR:-../sync}/.env
    environment:
      - PORT=3000
      - UPSTREAM=${SYNC_UPSTREAM:-http://risuai:6001}
    depends_on:
      - risuai

  caddy:
    image: caddy:alpine
    container_name: caddy
    restart: always
    ports:
      - "127.0.0.1:${CADDY_HTTPS_PORT:-6001}:443"
      - "127.0.0.1:${CADDY_HTTP_PORT:-6002}:80"
      - "127.0.0.1:9090:9090"
    expose:
      - "8080"
    env_file:
      - .env
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile
      - caddy_data:/data

  cloudflared:
    image: cloudflare/cloudflared:latest
    container_name: cloudflared
    restart: always
    command: tunnel run
    environment:
      - TUNNEL_TOKEN=${CLOUDFLARE_TUNNEL_TOKEN}
    depends_on:
      - caddy

volumes:
  sqlite-data:
  caddy_data:
```

## 기존 구성과의 관계

이 프로젝트는 `Risuai/` 루트에 있던 `Caddyfile`과 `docker-compose.yml`을 대체한다.
기존 파일은 sync만 지원하는 단순 구성이었으며, 이 프로젝트는 with-sqlite를 포함한 유동적 조합과 중간 노드 장애 복구를 지원한다.

### 환경변수

| 변수 | 기본값 | 설명 |
|------|--------|------|
| `RISUAI_DIR` | `../../../` | RisuAI 빌드 컨텍스트 경로 |
| `RISUAI_SAVE_DIR` | `risuai-save` | RisuAI 저장소 볼륨/경로 |
| `SQLITE_DIR` | `../with-sqlite` | with-sqlite 빌드 컨텍스트 경로 |
| `SQLITE_DATA_DIR` | `sqlite-data` | SQLite 데이터 볼륨/경로 |
| `SYNC_DIR` | `../sync` | Sync 서버 빌드 컨텍스트 경로 |
| `SYNC_UPSTREAM` | `http://risuai:6001` | Sync의 upstream (풀 체인 시 `http://caddy:8080`) |
| `CADDY_HTTPS_PORT` | `6001` | Caddy HTTPS 호스트 바인딩 포트 |
| `CADDY_HTTP_PORT` | `6002` | Caddy HTTP 호스트 바인딩 포트 |
| `SITE_DOMAIN` | - | Caddy TLS 도메인 |
| `CLOUDFLARE_TUNNEL_TOKEN` | - | Cloudflare Tunnel 인증 토큰 |
