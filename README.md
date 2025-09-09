# Expense Tracker — Backend

**Short description (EN):**
A modular microservices backend for an Expense Tracker application. The system is split into independent services that communicate over HTTP (and optionally gRPC). It includes:

* **Auth Service** — JWT based authentication (login, register, token refresh, middleware).
* **User Service** — manages user profiles and preferences.
* **DS Service** — a "data‑science / document‑sanitizer" service that calls external APIs to extract structured data from raw messages (e.g., email/SMS receipts). Uses LangChain (or similar) pipelines to parse message content and returns normalized fields (date, amount, vendor, category, etc.).
* **Expense Service** — stores and manages expense records; consumes the normalized data produced by DS Service.

> This README is written to be a single reference for developers who want to run, test and extend the microservice backend.

---

## Table of Contents

1. [Architecture overview](#architecture-overview)
2. [Tech stack](#tech-stack)
3. [Prerequisites](#prerequisites)
4. [Repository layout (recommended)](#repository-layout-recommended)
5. [Environment variables](#environment-variables)
6. [Run locally (docker-compose)](#run-locally-docker-compose)
7. [Service details & API examples](#service-details--api-examples)
8. [Authentication (JWT)](#authentication-jwt)
9. [DS Service & LangChain integration](#ds-service--langchain-integration)
10. [Testing](#testing)
11. [Deployment notes](#deployment-notes)
12. [Contributing](#contributing)
13. [License](#license)

---

## Architecture overview

The backend follows a microservice pattern where each service is independent and owns its own data. Communication is simple HTTP REST calls. Typical flow for creating an expense from an incoming message:

1. External message arrives (e.g. webhook or user upload).
2. DS Service receives the raw message and calls an external NLP/LLM pipeline (LangChain) to extract structured fields.
3. DS Service returns the normalized payload to the caller (or pushes it to Expense Service via API/event).
4. Expense Service creates an expense record and links it to a user.

*Optional:* Use a messaging layer (RabbitMQ / Kafka) to decouple DS Service from Expense Service for better scalability.

---

## Tech stack

* Language: Java, Python, Sprint-Boot
* Auth: JWT
* LangChain (or equivalent) for NLP/LLM-based parsing
* Database: MySQL
* Containerization: Docker
* Orchestration: Docker Compose (local), Kubernetes (production)


---

## Prerequisites

* Git
* Docker & Docker Compose
* API keys for any third-party LLM provider (OpenAI, Anthropic, etc.) used by DS Service

---

## Repository layout (recommended)

If you keep services in a single "main" repo as submodules or folders, a recommended structure is:

```
microservices-architecture/
├── README.md                # <-- this file
├── docker-compose.yml
├── auth-service/            # Auth service repo (or submodule)
├── user-service/            # User service repo
├── ds-service/              # DS + LangChain service
└── expense-service/         # Expense CRUD service
```

Each service should contain its own README, Dockerfile, and tests.

---

## Environment variables

Create a `.env` or set env vars for each service. Common variables:

```
# Auth service
AUTH_JWT_SECRET=your_jwt_secret
AUTH_TOKEN_EXPIRES_IN=3600

# DS service
LLM_PROVIDER=openai
OPENAI_API_KEY=sk-...
DS_API_TIMEOUT=30

# Database (example for expense service)
EXP_DB_URL=postgres://user:pass@db:5432/expense_db

# General
SERVICE_PORT=3000
```

Place these in service-specific `.env` files or in an orchestrator `docker-compose.env`.

---

## Run locally (docker-compose)

A sample `docker-compose.yml` should bring up the databases and all services. Example commands:

```bash
# clone the main repo (or create a main folder and add submodules)
git clone --recurse-submodules https://github.com/youruser/microservices-architecture.git
cd microservices-architecture

# build & run
docker compose up --build
```

Services will be reachable on the configured ports (see each service README for exact port mapping).

---

## Service details & API examples

### Auth Service (example endpoints)

* `POST /api/auth/register` — Register a new user
* `POST /api/auth/login` — Login & receive JWT
* `POST /api/auth/refresh` — Refresh token
* `GET /api/auth/me` — Get current user (requires `Authorization: Bearer <token>`)

**Register example**

```http
POST /api/auth/register
Content-Type: application/json
{
  "email": "user@example.com",
  "password": "securepassword",
  "name": "John Doe"
}
```

### User Service

* `GET /api/users/:id` — get user profile
* `PUT /api/users/:id` — update profile

### DS Service

* `POST /api/ds/extract` — send raw message content; returns normalized JSON like:

```json
{
  "date": "2025-09-01",
  "amount": 42.50,
  "currency": "USD",
  "vendor": "Cafe ABC",
  "category": "Food",
  "raw_text": "..."
}
```

### Expense Service

* `POST /api/expenses` — create expense (accepts payload from DS Service)
* `GET /api/expenses?userId=...` — list user expenses
* `GET /api/expenses/:id` — get single expense

Include examples of `curl` requests in each service README.

---

## Authentication (JWT)

* Auth Service issues JWT access tokens on login.
* All protected endpoints in other services must validate JWT with the shared `AUTH_JWT_SECRET` (or use a JWKS endpoint if using asymmetric keys).
* Prefer short-lived access tokens and refresh tokens for long sessions.

Sample Authorization header:

```
Authorization: Bearer <jwt_token_here>
```

---

## DS Service & LangChain integration

The DS Service is responsible for converting noisy message content into structured expense data using an LLM pipeline. Implementation notes:

* **Input:** raw text (message body), optional attachments (OCR may be needed).
* **Processing:** send the text through a LangChain pipeline / prompt template that extracts date, amount, currency, vendor, and probable category. Use few-shot examples in the prompt for robust extraction.
* **Output:** normalized JSON with confidence/provenance fields. Example:

```json
{
  "date": "2025-09-01",
  "amount": 42.5,
  "currency": "USD",
  "vendor": "Cafe ABC",
  "category": "Food",
  "confidence": 0.92,
  "provenance": { "prompt": "...", "model": "gpt-4o" }
}
```

* **Security & Cost:** redact sensitive content before sending to third-party LLMs. Cache repeated prompts and consider async processing for long-running operations.

* **Local testing:** if you don't want to use real LLM API during development, mock the DS endpoint to return deterministic parsed payloads.

---

## Testing

* Each service should include unit tests and integration tests for API endpoints.
* Use Postman / Insomnia collections for manual testing and include them in `/docs`.
* Example command (depends on language):

  * `npm test` or `go test ./...` or `pytest`

---

## Deployment notes

* Use CI (GitHub Actions) to build Docker images and push to a container registry.
* Use Kubernetes or managed services for production; ensure secrets (JWT secret, LLM keys, DB credentials) are stored in a secret manager.
* Scale DS Service and Expense Service independently depending on load.

---

## Contributing

1. Fork the repo
2. Create a feature branch
3. Add tests
4. Open a Pull Request

---

## Helpful TODOs (suggested improvements)

* Add end-to-end example: webhook → DS extract → expense creation flow
* Add OpenAPI / Swagger docs for all services
* Setup CI to run tests and linting
* Add role-based access control (RBAC) for expense sharing
* Add email/webhook retry and DLQ for failed DS extractions

---

## License

Add a license file (e.g., MIT) at the repo root.

---

*If you want, I can now:*

* generate a ready-to-use `docker-compose.yml` that wires all 4 services together,
* create a sample `.env.example` for all services,
* or inspect your actual code (you can paste `main` files or the repo tree) and generate a tailored README with exact commands, ports and examples.
