# FinCore Platform — Fintech / Exchange-grade Backend (Java 21)

FinCore is a production-like fintech platform core that demonstrates how banks and crypto exchanges build high-load, reliable systems where money correctness is the top priority.
This is not a CRUD pet project — it focuses on correctness, reliability, and operability.

[Notion link to explore how I build it](https://www.notion.so/Fintech-pet-project-2ec2cbc0e88b803a9765d17d8eb6b55a?source=copy_link)


## What the system does
FinCore implements realistic financial workflows on top of a shared Money Core:

#### Money Core (Ledger)

- Double-entry accounting (debit/credit postings)
- Immutable audit trail (append-only journal)
- Available/Locked balances (holds for orders / withdrawals / repayments)
- Deposits, transfers, fees, holds, releases

#### Lending lifecycle

- Onboarding/KYC → underwriting → offer → disbursement → repayment → overdue → collections
- State machines + background jobs

#### Exchange trading (Spot)

- Orders + matching engine (price-time priority)
- Settlement through ledger postings (real accounting model)

#### Crypto wallets

- Deposits detection + confirmations
- Withdrawals state machine (risk checks + approvals + broadcast + confirm)
- Exactly-once crediting + reorg-aware logic (simplified)

#### Operations & reliability

- Kafka domain events
- Transactional outbox pattern (no lost events)
- Consumer dedup (at-least-once safe)
- Retries/backoff + DLQ + replay/backfill
- Reconciliation (ledger vs external statements)
- Admin ops backend with RBAC + audit log

## Architecture
- Modular monolith with strict bounded contexts (microservice-ready design)
- Event-driven communication via Kafka
- Outbox pattern for transactional messaging
- Idempotency end-to-end (commands + consumer dedup)
- Designed for high-load hot paths (holds, settlement, single-writer matching)

## Tech Stack

- Java 21, Spring Boot 3 (Web, Security, Actuator)
- PostgreSQL + Flyway
- Kafka + DLQ topics
- Redis (idempotency, rate limits, cache)
- Observability: Micrometer/Prometheus/Grafana + OpenTelemetry tracing
- Testing: JUnit + Testcontainers (Postgres/Kafka/Redis)
- Docker Compose local environment

## Quick Start

#### Prerequisites
- Docker + Docker Compose
- JDK 21

#### Run locally
```bash
# 1) Start infrastructure
docker compose up -d

# 2) Run tests
./mvnw test

# 3) Start application
./mvnw spring-boot:run
```

## Endpoints:

- API: http://localhost:8080
- Swagger UI: http://localhost:8080/swagger-ui.html
- Metrics: http://localhost:8080/actuator/prometheus

## What this project proves (interview-ready)

- Money correctness (double-entry ledger, rounding rules, invariants)
- Transactions, locking & consistency
- Event-driven reliability (outbox, at-least-once, dedup)
- Workflow/state machine design (lending & withdrawals)
- Retry/DLQ/replay/backfill patterns
- Reconciliation and compensation flows
- Observability and operability mindset (metrics/tracing/runbooks)

## Resources:
[Link to google folder with resources](https://drive.google.com/drive/folders/1-NpjlIOOgHzVj-e3GZ7gOYUx_Eab5-AJ?usp=sharing)
