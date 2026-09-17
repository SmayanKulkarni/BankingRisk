# Banking Risk

A Spring Boot service for accounts, transfers, and risk-alert triage. Transfers are scored with deterministic rules. High-risk transfers are held until an analyst approves or rejects them; medium-risk transfers settle and receive an asynchronous explanation; low-risk transfers settle without an AI call.

The application lives in [`BankingProject`](BankingProject).

## What it does

- Registers users and issues JWTs.
- Maintains one account per user, with deposit and withdrawal endpoints.
- Transfers money between users, with optional idempotency keys.
- Builds a risk context from account, login, transaction, and money-flow data.
- Records risk alerts and lets an analyst approve or reject held transfers.
- Uses Spring AI to explain medium- and high-risk alerts. If the model is unavailable, it stores a rule-based fallback explanation.

## Transfer flow

1. The service creates an `INITIATED` transaction and evaluates its risk context.
2. It saves a `RiskAlert` with the score, triggered rules, and recommended action.
3. A high-risk transfer debits the sender and moves to `PENDING_REVIEW`. The receiver is credited only after analyst approval.
4. A low- or medium-risk transfer debits the sender, credits the receiver, and moves to `COMPLETED`.
5. Medium- and high-risk alerts receive their explanation on an asynchronous worker. The transfer itself does not wait for the model.

Rejecting a held transfer refunds the sender and marks the transaction `FAILED`.

## Risk scoring

`RiskRuleEngine` assigns points for account and transaction signals, including:

- New accounts and recent profile changes
- Failed login attempts and unusual recent transaction volume
- Large transfers relative to the account balance
- High-value transactions
- Combinations associated with account takeover, bust-out, or siphoning patterns

A score of 70 or more is `HIGH`; 35 through 69 is `MEDIUM`; lower scores are `LOW`. Rules are additive, so a transaction can trigger more than one signal.

## Privacy and AI explanations

Before calling the configured chat model, `PiiMaskingService` replaces user IDs with generated tokens. It also masks email addresses, IPv4 addresses, and phone numbers in free text. The service rehydrates model output locally before the alert is stored, then clears its in-memory token registry.

This is application-level masking, not a complete compliance boundary. Run it with appropriate access controls, logging policy, model-provider agreements, and a security review before using it with real customer data.

## Requirements

- Java 17
- Maven Wrapper, included in `BankingProject`
- PostgreSQL
- Redis
- An OpenAI-compatible chat endpoint and API key, or the Docker Compose Ollama setup

## Run locally

Start PostgreSQL and Redis, then set the values used by [`application.properties`](BankingProject/src/main/resources/application.properties). The project imports an optional `.env.properties` file, so a local file is convenient:

```properties
DB_HOST=localhost
DB_PORT=5432
DB_NAME=banking_db
DB_USERNAME=postgres
DB_PASSWORD=password
REDIS_HOST=localhost
REDIS_PORT=6379
JWT_SECRET=replace-with-a-long-random-secret
OPENROUTER_API_KEY=your-api-key
```

Then run the application:

```bash
cd BankingProject
./mvnw spring-boot:run
```

The API listens on `http://localhost:8080`. OpenAPI UI is available at `/swagger-ui/index.html` when the application is running.

### Docker Compose

`BankingProject/docker-compose.yml` starts PostgreSQL, Redis, Ollama, and the application:

```bash
cd BankingProject
docker compose up --build
```

The Compose configuration points Spring AI's OpenAI-compatible client at the bundled Ollama container using the `llama3.2` model. Pull that model into the container before exercising AI explanations.

## API overview

All account, transaction, and analyst endpoints require `Authorization: Bearer <token>`.

| Method | Endpoint | Purpose |
| --- | --- | --- |
| `POST` | `/api/auth/register` | Register a user |
| `POST` | `/api/auth/login` | Get a JWT |
| `GET` | `/api/account` | Get the current user's account |
| `POST` | `/api/account/deposit` | Deposit into the current user's account |
| `POST` | `/api/account/withdraw` | Withdraw from the current user's account |
| `POST` | `/api/transaction/transfer` | Transfer to another user |
| `GET` | `/api/transaction/history` | Get transaction history |
| `GET` | `/api/analyst/alerts` | List risk alerts |
| `POST` | `/api/analyst/alerts/{alertId}/resolve` | Approve or reject a held transfer |

### Create a transfer

`Idempotency-Key` is optional. Reusing a completed key returns the saved response instead of processing the transfer again.

```bash
curl -X POST http://localhost:8080/api/transaction/transfer \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: transfer-2026-001" \
  -d '{
    "toUserId": 2,
    "amount": 15000.00
  }'
```

### Resolve a held transfer

Use `APPROVE` to credit the receiver or `REJECT` to refund the sender.

```bash
curl -X POST http://localhost:8080/api/analyst/alerts/45/resolve \
  -H "Authorization: Bearer <analyst-token>" \
  -H "Content-Type: application/json" \
  -d '{
    "action": "APPROVE",
    "notes": "Verified with the customer."
  }'
```

## Database and operations

Flyway applies the schema migrations in [`src/main/resources/db/migration`](BankingProject/src/main/resources/db/migration). The service uses PostgreSQL for application data and Redis for idempotency and rate limiting. Health, info, and metrics endpoints are exposed through Spring Boot Actuator.

## Tests

```bash
cd BankingProject
./mvnw test
```

The test suite covers account operations, authentication, idempotency, transfer behavior, risk scoring, PII masking, and asynchronous risk analysis.
