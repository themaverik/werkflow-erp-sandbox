<div align="center">
  <img src="public/logo.png" alt="WERP Logo" width="300" />
</div>

# Werkflow ERP Sandbox

A sandbox system of record for testing and demonstrations. It serves HR, Finance, Procurement and
Inventory data over a real REST API, so workflows in the [Werkflow](https://github.com/themaverik/werkflow)
platform can be exercised end to end against something that behaves like a real ERP.

**This is not an ERP, and it is not an official Werkflow product.** It exists so that connectors,
BPMN examples and conformance tests have a deterministic target to run against. Werkflow deliberately
does not own a data layer — business data belongs in your own system of record, reached through a
registered connector. Point the connectors at your ERP and this repository stops being needed.

Do not deploy it as a production ERP or hold real business data in it.

| Property        | Value                                                          |
|-----------------|----------------------------------------------------------------|
| Type            | Spring Boot microservice (Java 21)                            |
| Port            | 8084                                                           |
| Context Path    | `/api/v1`                                                      |
| Database        | PostgreSQL 5433                                                |
| Authentication  | OIDC JWT (Keycloak, Auth0, Azure AD, AWS Cognito) or API Key  |
| Multi-Tenancy   | Yes                                                            |

## What This Service Does

Provides CRUD APIs for five domains:

- **HR**: Employees, departments, leave, attendance, payroll, performance reviews
- **Finance**: Budget plans, expenses, approval thresholds
- **Procurement**: Vendors, purchase requests, orders, receipts
- **Inventory**: Assets, categories, custody, transfers, maintenance
- **Identity**: User profile cache (OIDC), custody-owner-to-candidate-group mappings

Does not implement business approval logic, notifications, or workflow routing — those belong to the caller.

## What This Service Is Not

It is not a production ERP and carries none of the guarantees one needs. There is no migration path
off it, no data retention or backup story, no hardening review, and no commitment to a stable schema
between releases.

Treat every row in it as disposable test data.

## Prerequisites

- Docker and Docker Compose
- Java 21+
- Maven 3.8+

Shared services (must be running):

```bash
cd ../werkflow/infrastructure/docker
docker compose up -d postgres keycloak mailpit
```

## Quick Start

```bash
# Build
mvn clean install -DskipTests

# Run
docker compose up -d

# Verify
curl -s http://localhost:8084/api/v1/actuator/health | jq .
```

Swagger UI: http://localhost:8084/api/v1/swagger-ui.html

## Configuration

Environment variables in `config/env/`:

| File             | Purpose                         |
|------------------|---------------------------------|
| `.env.shared`    | Database and Keycloak URLs      |
| `.env.business`  | Service port and log level      |

Key variables: `POSTGRES_HOST`, `POSTGRES_PORT`, `KEYCLOAK_URL`, `KEYCLOAK_REALM`, `SERVER_PORT`.

## API Key Authentication

Requests may authenticate with `X-API-Key: <raw-key>` instead of a Bearer token. Keys are validated against a SHA-256 hash stored in the `api_keys` table — raw keys are never persisted.

Generate a key via `POST /api/v1/api-keys/generate` (requires `ADMIN`, `SUPER_ADMIN`, or `ENGINE_SERVICE` role). Store the returned `rawKey` securely (e.g. OpenBao) — it cannot be retrieved again.

See [docs/how-to/API-Usage-Guide.md](./docs/how-to/API-Usage-Guide.md) for manual key registration and revocation steps.

## Documentation

| Document | Description |
|----------|-------------|
| [docs/explanation/Architecture-Overview.md](./docs/explanation/Architecture-Overview.md) | Architecture, design principles, and business flow diagrams |
| [docs/how-to/API-Usage-Guide.md](./docs/how-to/API-Usage-Guide.md) | Step-by-step API examples for all domains |
| [docs/how-to/Werkflow-Integration-Guide.md](./docs/how-to/Werkflow-Integration-Guide.md) | Connector setup, BPMN workflow examples |
| [docs/project/qa/Independence-Checklist.md](./docs/project/qa/Independence-Checklist.md) | PR review checklist and anti-pattern guide |

## License

Proprietary — All rights reserved
