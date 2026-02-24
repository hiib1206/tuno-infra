# tuno-infra

Docker Compose 기반 멀티 컨테이너 인프라 구성

## 왜 만들었는가?

프론트엔드, 백엔드, WebSocket, AI API, 배치 서버 등 여러 서비스를 하나의 서버에서 운영해야 했습니다. 각 서비스를 독립적으로 배포하면서도 서비스 간 통신, 의존성 관리, 리버스 프록시를 일관되게 구성할 필요가 있었습니다.

Docker Compose로 전체 서비스를 정의하고, Nginx로 도메인별 라우팅과 SSL 처리를 담당하는 구조로 구성했습니다.

## 주요 기능

- **멀티 컨테이너 오케스트레이션** - 7개 서비스를 단일 Compose 파일로 관리
- **서비스 의존성 관리** - healthcheck 기반 시작 순서 보장
- **리버스 프록시** - 도메인별 라우팅 (tunoinvest.com, api.tunoinvest.com, ws.tunoinvest.com)
- **Rate Limiting** - API 과부하 방지 (초당 10회, 버스트 20)
- **WebSocket 프록시** - Connection Upgrade 처리, 24시간 타임아웃

## 기술 스택

| 분류          | 기술                          |
| ------------- | ----------------------------- |
| Container     | Docker, Docker Compose        |
| Reverse Proxy | Nginx                         |
| Database      | MariaDB 11.8                  |
| Cache         | Redis 7                       |
| SSL           | Cloudflare Origin Certificate |

## 아키텍처

```
                    ┌─────────────────────────────────────┐
                    │             Nginx                   │
                    │         (Reverse Proxy)             │
                    │   :80 → :443 redirect               │
                    │   SSL termination                   │
                    └─────────────┬───────────────────────┘
                                  │
        ┌─────────────────────────┼─────────────────────────┐
        │                         │                         │
        ▼                         ▼                         ▼
┌───────────────┐       ┌───────────────┐       ┌───────────────┐
│   Frontend    │       │    Backend    │       │   WebSocket   │
│  (Next.js)    │       │   (Express)   │       │     (ws)      │
│ tunoinvest.com│       │api.tunoinvest │       │ws.tunoinvest  │
└───────────────┘       └───────┬───────┘       └───────┬───────┘
                                │                       │
                    ┌───────────┴───────────┐           │
                    ▼                       ▼           │
              ┌──────────┐            ┌──────────┐      │
              │ MariaDB  │            │  Redis   │◀─────┘
              └──────────┘            └──────────┘
                    ▲                       ▲
                    │                       │
              ┌─────┴─────┐           ┌─────┴─────┐
              │  AI API   │           │   Batch   │
              │ (FastAPI) │           │(Scheduler)│
              └───────────┘           └───────────┘
```

## 서비스 구성

| 서비스   | 이미지         | 포트    | 설명                  |
| -------- | -------------- | ------- | --------------------- |
| frontend | tuno-frontend  | 3000    | Next.js 프론트엔드    |
| backend  | tuno-backend   | 4000    | Express API 서버      |
| ws       | tuno-ws        | 8080    | WebSocket 릴레이 서버 |
| ai-api   | tuno-ai-api    | 8000    | FastAPI 추론 서버     |
| batch    | tuno-batch     | -       | 데이터 수집 스케줄러  |
| mariadb  | mariadb:11.8   | 3306    | 데이터베이스          |
| redis    | redis:7-alpine | 6379    | 캐시/Pub-Sub          |
| nginx    | nginx:alpine   | 80, 443 | 리버스 프록시         |

## Nginx 설정

### 도메인 라우팅

```
tunoinvest.com      → frontend:3000
api.tunoinvest.com  → backend:4000
ws.tunoinvest.com   → ws:8080
```

### 주요 설정

- **HTTP → HTTPS 리다이렉트** - 모든 HTTP 요청을 HTTPS로 전환
- **Gzip 압축** - text, json, javascript 등 압축 전송
- **Rate Limit** - API 서버에 초당 10회 제한 적용
- **WebSocket 타임아웃** - 24시간 연결 유지
- **보안 헤더** - X-Frame-Options, X-Content-Type-Options, X-XSS-Protection

## 관련 저장소

- [tuno-frontend](https://github.com/hiib1206/tuno-frontend) - Next.js 프론트엔드
- [tuno-backend](https://github.com/hiib1206/tuno-backend) - Express API 서버
- [tuno-ws](https://github.com/hiib1206/tuno-ws) - WebSocket 릴레이 서버
- [tuno-ai](https://github.com/hiib1206/tuno-ai) - AI 분석 서버
