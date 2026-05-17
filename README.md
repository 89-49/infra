# PGSG Infrastructure

## 개요

PGSG 서비스의 데이터베이스 및 모니터링 인프라를 Docker Compose 기반으로 구성한 레포지토리입니다.

---

## 구성

```
infra/
├── monitoring/
│   ├── rules/                  # Prometheus 알림 규칙
│   ├── alertmanager.yml        # Alertmanager 설정
│   ├── docker-compose.yml      # 모니터링 서비스 컴포즈
│   ├── prometheus.yml          # Prometheus 수집 대상 설정
│   └── promtail-config.yml     # Promtail 로그 수집 설정
├── postgres/                   # PostgreSQL 설정 (고가용성 구성 예정)
├── redis/                      # Redis 설정 (고가용성 구성 예정)
├── .env.template               # 환경변수 템플릿
├── .gitignore
└── docker-compose.infra.yaml   # DB 서비스 컴포즈
```

---

## 서비스 구성

### 데이터베이스 (docker-compose.infra.yaml)

| 서비스 | 이미지 | 포트 | 설명 |
|--------|--------|------|------|
| PostgreSQL | postgres:17 | 5432 | 메인 데이터베이스 |
| Redis | redis:7-alpine | 6379 | 캐시 및 세션 저장소 |

> 현재 단일 인스턴스로 운영 중이며, 부하테스트 결과를 기반으로 PostgreSQL Replication 및 Redis Sentinel을 활용한 고가용성 구성으로 전환 예정입니다.

### 모니터링 (monitoring/docker-compose.yml)

| 서비스 | 이미지 | 포트 | 설명 |
|--------|--------|------|------|
| Prometheus | prom/prometheus | 9090 | 메트릭 수집 |
| Grafana | grafana/grafana | 3000 | 시각화 대시보드 |
| Loki | grafana/loki | 3100 | 로그 저장소 |
| Promtail | grafana/promtail | - | 로그 수집기 |
| Alertmanager | prom/alertmanager | 9093 | 알림 관리 |
| Zipkin | openzipkin/zipkin | 9411 | 분산 트레이싱 |

### Grafana 초기 접속 정보

| 항목 | 값 |
|------|-----|
| 접속 URL | http://모니터링서버IP:3000 |
| 초기 계정 | admin |
| 초기 비밀번호 | .env의 GRAFANA_PASSWORD 값 |

**기본 대시보드**

| 대시보드 | ID | 설명 |
|---------|-----|------|
| JVM (Micrometer) | 4701 | JVM 메모리, GC, 스레드, HTTP 지표 |
| Spring Boot APM | 12900 | Spring Boot 애플리케이션 전반 지표 |
| k6 Load Testing | 19665 | k6 부하테스트 결과 시각화 |

---

## 아키텍처

```
각 서비스 VM
  └── Promtail             → Loki        (로그 수집)
  └── /actuator/prometheus → Prometheus  (메트릭 수집)
  └── Zipkin Endpoint      → Zipkin      (트레이싱)

모니터링 VM
  └── Grafana
        ├── Prometheus (메트릭 조회)
        └── Loki       (로그 조회)
```

---

## 환경변수 설정

`.env.template`을 복사하여 `.env` 파일 생성 후 값을 채워주세요.

```bash
cp .env.template .env
```

### DB 환경변수

```env
DB_USERNAME=      # PostgreSQL 유저명
DB_PASSWORD=      # PostgreSQL 비밀번호
DB_DATABASE=      # PostgreSQL 데이터베이스명
```

### 모니터링 환경변수

```env
GRAFANA_PASSWORD=         # Grafana 관리자 비밀번호
ZIPKIN_DB_USER=           # Zipkin DB 유저명
ZIPKIN_DB_PASSWORD=       # Zipkin DB 비밀번호
ZIPKIN_DB_ROOT_PASSWORD=  # Zipkin DB root 비밀번호
DISCORD_WEBHOOK=          # Discord 알림 웹훅 URL
```

---

## 실행 방법

### DB 서비스 실행

```bash
# 네트워크 생성 (최초 1회)
docker network create pgsg-network

# DB 실행
docker compose -f docker-compose.infra.yaml up -d
```

### 모니터링 서비스 실행

```bash
cd monitoring
docker compose up -d
```

### Prometheus 수집 대상 추가

`monitoring/prometheus.yml`의 `scrape_configs`에 서비스를 추가해주세요.

```yaml
scrape_configs:
  - job_name: 서비스명
    static_configs:
      - targets: ['서버IP:포트']
    metrics_path: /actuator/prometheus
```

### 각 서비스 모니터링 연동 설정

`application.yml`에 아래 내용을 추가해주세요.

```yaml
management:
  metrics:
    tags:
      application: ${spring.application.name}
  tracing:
    sampling:
      probability: 1.0
  zipkin:
    tracing:
      endpoint: http://모니터링서버IP:9411/api/v2/spans
```

### Promtail 각 서비스에 추가

로그 수집이 필요한 서비스 VM에 `promtail-config.yml` 파일을 생성해주세요.

```yaml
server:
  http_listen_port: 9080

clients:
  - url: http://모니터링서버IP:3100/loki/api/v1/push

scrape_configs:
  - job_name: 서비스명
    static_configs:
      - targets: [localhost]
        labels:
          job: 서비스명
          __path__: /로그경로/*.log
```

해당 서비스의 `docker-compose.yml`에 Promtail 컨테이너를 추가해주세요.

```yaml
promtail:
  image: grafana/promtail:2.9.1
  container_name: promtail
  volumes:
    - ./promtail-config.yml:/etc/promtail/config.yml
    - /로그경로:/logs
  command: -config.file=/etc/promtail/config.yml
```

### Alertmanager Discord 연동

`.env`에 Discord 웹훅 URL을 설정해주세요.

```env
DISCORD_WEBHOOK=https://discord.com/api/webhooks/...
```

Discord 웹훅 URL은 Discord 서버 설정 → 연동 → 웹후크에서 생성할 수 있습니다.

---

## 향후 개선 계획

- PostgreSQL Primary-Replica 구성으로 읽기 부하 분산
- Redis Sentinel 구성으로 자동 장애 복구
- 부하테스트 결과에 따라 GCP Cloud SQL, Memorystore 전환 검토
