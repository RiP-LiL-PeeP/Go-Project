# Interview Go Project

>  Репозиторий тренировочного проекта: микросервис на Go (REST + gRPC) с Postgres, Redis, брокером сообщений и S3-совместимым хранилищем (MinIO).
---

## Краткое описание

Этот репозиторий содержит шаблон и пример реализации backend-сервиса на Go, который покрывает следующие темы:

* idiomatic Go, модули и тестирование
* concurrency: goroutines, channels, patterns
* REST API + OpenAPI (Swagger) и gRPC (proto)
* PostgreSQL (pgx/database/sql) + миграции
* Redis (кеширование, TTL, pub/sub)
* Message broker (RabbitMQ / Kafka) для асинхронных задач
* Object storage (MinIO / S3): загрузка файлов, presigned URL
* Docker + docker-compose, Kubernetes manifests (basic)
* CI: GitHub Actions (пример pipeline)
* Observability: pprof, Prometheus metrics, structured logs
* Performance: benchmarking, load testing (wrk/hey)

---

## Цели

1. Быстрый запуск локального окружения для разработки и тестирования.
2. Дать возможность показывать и объяснять архитектуру на интервью.
3. Иметь набор скриптов для CI, тестов и нагрузочного профайлинга.

---

## Технологический стек

* Язык: Go (1.20+)
* HTTP framework: net/http
* gRPC + protobuf
* База данных: PostgreSQL (pgx)
* Кеш: Redis
* Broker: RabbitMQ (альтернатива: Kafka)
* Object storage: MinIO (S3 совместимый)
* Контейнеризация: Docker, docker-compose
* Оркестрация: Kubernetes (manifests)
* CI: GitHub Actions 
* Мониторинг: Prometheus

---

## Структура репозитория

```
/cmd/
  /service/        -> main для сервиса
/internal/
  /api/            -> handlers, middleware
  /grpc/           -> grpc server impl
  /store/          -> db/repo layer (postgres, redis)
  /worker/         -> background jobs
  /jobs/           -> message consumer
  /config/         -> конфигурация и env
  /migrations/     -> sql migration files
/pkg/              -> общие пакеты (logger, errors)
/deploy/
  /docker/         -> Dockerfile, docker-compose.yml
  /k8s/            -> manifests
/scripts/          -> helper scripts (run_local.sh, migrate.sh)
/proto/             -> .proto файлы
/tests/             -> интеграционные тесты
README.md
```

---

## Быстрый старт (локально)

**Требуется:** Docker и docker-compose

1. Клонируй репозиторий:

```bash
git clone git@github.com:RiP-LiL-PeeP/Go-Project.git
cd Пo-project
```

2. Скопируй `.env.example` в `.env` и при необходимости поправь переменные.

3. Поднять инфраструктуру (Postgres, Redis, RabbitMQ, MinIO):

```bash
docker-compose -f deploy/docker/docker-compose.yml up --build
```

4. Запустить миграции:

```bash
./scripts/migrate.sh up
# или с использованием мигратора (golang-migrate/goose)
```

5. Запустить сервис (локально в dev режиме):

```bash
go run ./cmd/service
```

6. Проверить endpoints:

* REST: `http://localhost:8080/health`, `http://localhost:8080/api/v1/items`
* gRPC: `localhost:50051` (use grpcurl / evans for testing)

---

## Пример environment variables (`.env.example`)

```
APP_ENV=development
PORT=8080
GRPC_PORT=50051
DATABASE_URL=postgres://user:pass@db:5432/app_db?sslmode=disable
REDIS_ADDR=redis:6379
RABBITMQ_URL=amqp://guest:guest@rabbitmq:5672/
MINIO_ENDPOINT=minio:9000
MINIO_ACCESS_KEY=minioadmin
MINIO_SECRET_KEY=minioadmin
S3_BUCKET=app-bucket
JWT_SECRET=some-secret
LOG_LEVEL=debug
```

---

## Как запускать тесты

* Unit tests:

```bash
go test ./... -v
```

* Интеграционные тесты (в Docker):

```bash
docker-compose -f deploy/docker/docker-compose.test.yml up --build --abort-on-container-exit
```

* Запуск с race detector:

```bash
go test ./... -race
```

---

## CI (примерная схема GitHub Actions)

Pipeline stages:

* `lint` (golangci-lint)
* `unit-test` (go test)
* `build` (multi-stage Docker build)
* `integration-test` (optional, runs in matrix with services)
* `publish` (docker push)

В `/.github/workflows/ci.yml` хранятся примеры шагов.

---

## Dockerfile (рекомендация)

* Использовать multi-stage build
* Минимизировать размер (scratch/distroless)
* Запускать как non-root user

Пример (краткий):

```dockerfile
# build stage
FROM golang:1.20 AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o /app/bin/service ./cmd/service

# final
FROM gcr.io/distroless/static
COPY --from=builder /app/bin/service /service
USER nonroot:nonroot
ENTRYPOINT ["/service"]
```

---

## Kubernetes manifests (deploy/k8s)

Добавить манифесты: Deployment, Service, ConfigMap, Secret, Ingress, HPA. Для stateful сервисов (Postgres, Redis) использовать managed service или StatefulSet.

---

## API спецификация

* Документировать REST endpoints через OpenAPI (yaml/json) и генерировать клиент/стаб.
* Протоколы gRPC описаны в `/proto`.

Пример endpoints:

```
GET /api/v1/health -> { status }
POST /api/v1/items -> create item (auth required)
GET /api/v1/items/{id} -> retrieve item
POST /api/v1/uploads -> presigned upload -> PUT to S3
```

---

## Background jobs & Broker

* Producer: main service кладёт события в очередь при создании ресурса.
* Consumer: worker читает, выполняет длительные операции (обработка, записывание в объектное хранилище).
* Обрабатывать повторные доставки (idempotency), DLQ (dead-letter queue), и дедлагаи.

---

## Observability

* Логирование: structured JSON logs (zap/logrus/zerolog), context-aware logging.
* Метрики: Prometheus metrics (`/metrics`). Основные метрики: request\_latency, requests\_total, errors\_total, goroutines\_count, db\_conn\_count.
* Tracing: OpenTelemetry (traces for HTTP/gRPC + DB calls).
* Профайлинг: pprof endpoints (for dev), инструкция как снимать дамп и анализировать.

---

## Performance testing и профайлинг

* Инструменты: wrk, hey, vegeta
* Запуск: `wrk -t12 -c400 -d30s http://localhost:8080/api/v1/items`
* Собирать pprof: `go tool pprof -http=:8081 <binary> <profile>` или через `pprof` пакет.

---

## Архитектура и design decisions

Здесь опиши ключевые решения, которые важно уметь объяснить на интервью:

* Почему выбран PostgreSQL для основного хранилища, где использован Mongo?
* Как реализован кеш и invalidation strategy?
* Выбранный подход к аутентификации/авторизации (JWT, RBAC)
* Как обеспечивается отказоустойчивость и масштабирование (read replicas, shard, partitioning)
* Patterns: Circuit Breaker, Bulkhead, Retry + Backoff, Idempotency tokens, Saga (если есть распределённые транзакции)

---

## Checklist перед интервью (что показать)

* [ ] Рабочее локальное окружение (`docker-compose up`).
* [ ] Минимальный набор API (CRUD) + примеры curl/requests.
* [ ] gRPC endpoint и proto файл.
* [ ] Unit + Integration tests (покрытие ключевых модулей).
* [ ] CI файл (GitHub Actions).
* [ ] Документы: architecture.md, decisions.md, performance-report.md.

---

## TODOs (шаблон)

* [ ] Добавить примеры кода для аутентификации (JWT + middleware)
* [ ] Реализовать worker + message broker (RabbitMQ)
* [ ] Подготовить simple load-test report
* [ ] Добавить k8s Helm chart
* [ ] Добавить example of DB partitioning/sharding plan

---
