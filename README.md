# SoftPOS Platform

> Cloud-native, event-driven microservices platform turning an Android device into a software point-of-sale (tap-to-pay) terminal — built with Spring Boot, Keycloak, ISO 8583 and Apache Kafka.

![Java](https://img.shields.io/badge/Java-21-orange)
![Spring Boot](https://img.shields.io/badge/Spring%20Boot-3.3.x-brightgreen)
![Build](https://img.shields.io/badge/build-passing-brightgreen)
![License](https://img.shields.io/badge/license-MIT-blue)
![Status](https://img.shields.io/badge/status-training%20project-yellow)

> **Status:** this is a personal training/practice project for learning production-grade microservices patterns in a fintech context. The acquirer host is fully **mocked** — no live card scheme connectivity is included.

---

## Table of contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Tech stack](#tech-stack)
- [Repository structure](#repository-structure)
- [Prerequisites](#prerequisites)
- [Getting started](#getting-started)
- [Services & ports](#services--ports)
- [Running a single service](#running-a-single-service)
- [Configuration](#configuration)
- [Testing](#testing)
- [API documentation](#api-documentation)
- [Observability](#observability)
- [Roadmap](#roadmap)
- [Contributing](#contributing)
- [License](#license)

---

## Overview

SoftPOS ("software point-of-sale") turns a commercial off-the-shelf smartphone into a card-acceptance terminal, reading contactless (NFC) transactions without dedicated payment hardware. This repository contains the **backend platform**: a set of Spring Boot microservices coordinating merchant/device onboarding, risk scoring, tokenization, ISO 8583 authorization messaging against a mocked acquirer, and event-driven settlement/reporting.

The project is deliberately built with **PCI-DSS-aligned architectural boundaries** (isolated Cardholder Data Environment, no PAN in logs, keys managed via Vault) so that the practices transfer directly to a real-world payments engagement, even though certification itself is out of scope here.

📄 Full architecture rationale, per-service dependency list, and the communication diagram live in [`docs/SoftPOS-Architecture-Specification.pdf`](./docs/SoftPOS-Architecture-Specification.pdf) — this README covers only what you need to **run and develop** the project day to day.

## Architecture

At a glance: the mobile app talks to everything through a single **API Gateway**, secured by **Keycloak**. `transaction-service` acts as a saga orchestrator across risk, tokenization and ISO 8583 authorization. Everything downstream of a completed transaction (receipts, audit, settlement, reporting) is driven asynchronously off a **Kafka** event bus using the transactional outbox pattern.

```
Mobile app → API Gateway → Transaction Service ─┬─→ Risk Service
                                                  ├─→ Tokenization Service → Key Management Service
                                                  └─→ ISO 8583 Gateway → Acquirer Mock

Transaction Service → Kafka ─┬─→ Notification Service
                              ├─→ Audit Service
                              ├─→ Reconciliation Service
                              └─→ Reporting Service
```

See the PDF spec for the full diagram, the communication matrix, and the reasoning behind each boundary.

## Tech stack

| Layer | Technology |
|---|---|
| Language / runtime | Java 21, Spring Boot 3.3.x |
| API gateway | Spring Cloud Gateway |
| Service discovery | Netflix Eureka |
| Centralized config | Spring Cloud Config |
| Identity & access | Keycloak (OAuth2 / OIDC) |
| Messaging | Apache Kafka (KRaft mode) |
| ISO 8583 | jPOS |
| Relational storage | PostgreSQL (one schema per service) |
| Document storage | MongoDB (audit & reporting projections) |
| Caching | Redis |
| Secrets / key management | HashiCorp Vault (Transit engine as mock HSM) |
| Resilience | Resilience4j |
| Tracing | Micrometer Tracing + Zipkin |
| Mobile client | Kotlin, Jetpack Compose, Android NFC Reader Mode |
| Containerization | Docker / Docker Compose (local), Kubernetes-ready |

## Repository structure

```
softpos-platform/
├── infra/
│   ├── config-server/
│   ├── discovery-server/
│   └── api-gateway/
├── services/
│   ├── merchant-service/
│   ├── device-service/
│   ├── transaction-service/
│   ├── risk-service/
│   ├── tokenization-service/
│   ├── key-management-service/
│   ├── iso8583-gateway-service/
│   ├── acquirer-mock-service/
│   ├── notification-service/
│   ├── audit-service/
│   ├── reconciliation-service/
│   └── reporting-service/
├── mobile/
│   └── softpos-android/            # Kotlin / Jetpack Compose client
├── docker/
│   ├── docker-compose.yml          # full local stack
│   └── keycloak/
│       └── realm-export.json       # pre-configured realm, clients, roles
├── docs/
│   └── SoftPOS-Architecture-Specification.pdf
├── .env.example
└── README.md
```

Each service under `services/` and `infra/` is a standalone Maven project generated from [start.spring.io](https://start.spring.io) — see each service's own `README.md` for its specific dependency list.

## Prerequisites

- Java 21 (SDKMAN or Temurin recommended)
- Maven 3.9+
- Docker & Docker Compose
- An IDE with Lombok annotation processing enabled

## Getting started

```bash
# 1. Clone the repository
git clone https://github.com/<your-org>/softpos-platform.git
cd softpos-platform

# 2. Copy environment defaults
cp .env.example .env

# 3. Start infrastructure (Postgres, Kafka, Redis, MongoDB, Vault, Keycloak, Zipkin)
docker compose -f docker/docker-compose.yml up -d

# 4. Import the pre-configured Keycloak realm (clients, roles, test users)
#    already handled automatically by docker/keycloak/realm-export.json on first boot

# 5. Build all services
./mvnw clean install -DskipTests

# 6. Start the platform services in order
./mvnw spring-boot:run -pl infra/config-server
./mvnw spring-boot:run -pl infra/discovery-server
./mvnw spring-boot:run -pl infra/api-gateway
./mvnw spring-boot:run -pl services/merchant-service,services/device-service,services/transaction-service,...
```

> Tip: once you're past initial setup, use the provided `docker/docker-compose.full.yml` to run every service as a container instead of starting each one manually.

## Services & ports

| Service | Port | Notes |
|---|---|---|
| config-server | 8888 | must be first to start |
| discovery-server (Eureka) | 8761 | dashboard at `/` |
| api-gateway | 8080 | single public entry point |
| keycloak | 8180 | admin console at `/admin` |
| merchant-service | 8081 | |
| device-service | 8082 | |
| transaction-service | 8083 | saga orchestrator |
| risk-service | 8084 | |
| tokenization-service | 8085 | internal only — not exposed via gateway |
| key-management-service | 8086 | internal only — not exposed via gateway |
| iso8583-gateway-service | 8087 | HTTP admin API |
| acquirer-mock-service | 8088 / 10000 | 8088 = admin API, 10000 = ISO 8583 TCP socket |
| notification-service | 8089 | |
| audit-service | 8090 | |
| reconciliation-service | 8091 | |
| reporting-service | 8092 | |
| PostgreSQL | 5432 | one database per service |
| Kafka | 9092 | KRaft mode, no Zookeeper |
| Redis | 6379 | |
| MongoDB | 27017 | |
| Vault | 8200 | dev mode locally |
| Zipkin | 9411 | trace UI |

## Running a single service

Every service is independently runnable once `config-server` and `discovery-server` are up:

```bash
cd services/transaction-service
../../mvnw spring-boot:run
```

Local profile defaults (`application-local.yml`) point to the Dockerized infrastructure started above — no additional configuration should be needed for local development.

## Configuration

- Shared, non-secret configuration lives in `config-server`'s backing Git repo (see `infra/config-server/README.md`).
- Secrets (DB credentials, Vault tokens, Kafka credentials) are **never committed** — copy `.env.example` to `.env` and fill in local values, or point Spring Cloud Config at your own Vault instance.
- Each service exposes its config contract in its own `src/main/resources/application.yml`.

## Testing

```bash
# Unit + integration tests for a single service (uses Testcontainers)
cd services/transaction-service
../../mvnw test

# Full platform test suite
./mvnw test
```

Integration tests spin up real Postgres/Kafka/MongoDB containers via Testcontainers rather than mocking the infrastructure — expect Docker to be running.

## API documentation

Each service exposes OpenAPI docs at `/swagger-ui.html` when running locally. The aggregated, gateway-routed view is available at `http://localhost:8080/swagger-ui.html` once the platform is fully up.

## Observability

- **Tracing:** Zipkin UI at `http://localhost:9411`
- **Health:** every service exposes `/actuator/health`
- **Metrics:** Prometheus-formatted metrics at `/actuator/prometheus` (Grafana dashboards in `docker/grafana/`)

## Roadmap

See [`docs/SoftPOS-Architecture-Specification.pdf`](./docs/SoftPOS-Architecture-Specification.pdf), section 10, for the full phased implementation plan. Current focus:

- [x] Platform plumbing (config, discovery, gateway, Keycloak)
- [x] ISO 8583 gateway + acquirer mock
- [ ] Transaction saga (risk + tokenization steps)
- [ ] Kafka outbox + event consumers
- [ ] Reconciliation batch job
- [ ] Android client integration

## Contributing

This is currently a solo learning project, but issues and suggestions are welcome. If you'd like to contribute:

1. Fork the repo and create a feature branch (`git checkout -b feature/my-change`)
2. Follow the existing package structure and naming conventions per service
3. Make sure `./mvnw verify` passes before opening a PR
4. Open a PR with a clear description of the change and why

## License

Distributed under the MIT License. See [`LICENSE`](./LICENSE) for details.