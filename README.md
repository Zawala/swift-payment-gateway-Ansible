# FortKnox Payment Stack — System Documentation

Two Spring Boot services that together form a secure ISO 20022 payment processing gateway.

---

## Services at a Glance

| | ZW | Capital |
|---|---|---|
| **Role** | Auth gateway + rate limiter | ISO 20022 payments processor |
| **Port** | 8080 | 8460 |
| **Default profile** | `dev` (H2) | `dev` (H2) |
| **Package** | `fortknox.zw` | `capital.one.capital` |

---

## Architecture

```
External Client
      │
      │  POST /auth/login  →  { accessToken, refreshToken }
      ▼
┌─────────────────────────────────┐
│         ZW  :8080               │
│  Auth + Rate Limiter + Gateway  │
│  JJWT HS256 · Bucket4j          │
└────────────┬────────────────────┘
             │  Authorization: Bearer <accessToken>
             │  (routes to internal services)
             ▼
┌─────────────────────────────────┐
│       Capital  :8460            │
│  ISO 20022 Payments Processor   │
│  JAXB · pacs · acmt · JPA       │
└─────────────────────────────────┘
             │  XML over HTTP
             ▼
    Destination Institution
    (or /mock/* in dev)
```

ZW sits at the edge. Every inbound request is rate-limited and JWT-validated before reaching Capital. Capital validates the same JWT independently using the shared secret — no callback to ZW at request time.

---

## ZW — Auth & API Gateway

### What it does

- Registers and stores users (Spring Data JPA)
- Issues short-lived access tokens (15 min) and long-lived refresh tokens (7 days) as HMAC-SHA256 JWTs
- Rotates refresh tokens on every use — a stolen token can only be used once
- Enforces single active session per user — login revokes all previous refresh tokens
- Rate-limits all endpoints per client IP using Bucket4j token bucket (50 req/60s dev, 100 req/60s uat/prod)

### Auth endpoints (public)

| Method | Path | Description |
|---|---|---|
| `POST` | `/auth/register` | Create a new user account |
| `POST` | `/auth/login` | Authenticate; receive `accessToken` + `refreshToken` |
| `POST` | `/auth/refresh` | Exchange refresh token for a new pair (old token revoked) |
| `POST` | `/auth/logout` | Revoke a refresh token |

### Protected endpoints

| Method | Path | Description |
|---|---|---|
| `GET` | `/ping` | Health check — confirms token is valid |

### Token lifecycle

```
POST /auth/login
      │
      ├─▶ accessToken   (15 min, stateless — validate with secret only)
      └─▶ refreshToken  (7 days, stored in DB — revocable immediately)

POST /auth/refresh
      │
      ├─▶ new accessToken
      ├─▶ new refreshToken
      └─▶ old refreshToken immediately revoked
```

### Default seed user

| Username | Password |
|---|---|
| `zawala` | `changeme` |

Change before any non-local deployment.

### Profiles

| Profile | DB | `ddl-auto` | Rate limit |
|---|---|---|---|
| `dev` (default) | H2 in-memory | `create-drop` | 50 req/60s |
| `uat` | PostgreSQL `fortknox_uat` | `validate` | 100 req/60s |
| `prod` | PostgreSQL `fortknox` | `validate` | 100 req/60s |

### ZW test script (`test-auth-flow.sh`)

Exercises the full auth lifecycle:

1. Register a new user
2. Attempt duplicate registration (expect 409)
3. Login — capture `accessToken` + `refreshToken`
4. Access `/ping` with valid token (expect 200)
5. Access `/ping` without token (expect 401)
6. Refresh — receive new token pair
7. Reuse old refresh token (expect 401 — revoked)
8. Access `/ping` with new token (expect 200)
9. Logout
10. Attempt refresh after logout (expect 401)

```bash
./test-auth-flow.sh
# or against a different host:
BASE_URL=http://authserver:8080 ./test-auth-flow.sh
```

---

## Capital — ISO 20022 Payments Processor

### What it does

Accepts simple JSON requests, builds standards-compliant ISO 20022 XML messages using JAXB-generated classes from the official XSD schemas, forwards them to the destination institution over HTTP, and persists every interaction for audit and reconciliation.

### REST endpoints (all require `Authorization: Bearer <token>` in uat/prod)

| Method | Path | ISO 20022 message | Description |
|---|---|---|---|
| `POST` | `/api/v1/avs/verify` | `acmt.023.001.04` → `acmt.024.001.04` | Verify account holder name |
| `POST` | `/api/v1/payments/transfer` | `pacs.008.001.14` → `pacs.002.001.16` | Initiate credit transfer |
| `GET` | `/api/v1/payments/status?msgId=` | `pacs.028.001.07` → `pacs.002.001.16` | Query payment status |
| `GET` | `/api/v1/payments/return?msgId=&reasonCode=` | `pacs.004.001.15` → `pacs.002.001.16` | Return a payment |

### Payment flow (per request)

```
JSON request
    │
    ▼  Builder service populates ISO 20022 object graph
    │
    ▼  Iso20022MarshallingService marshals to XML (JAXB)
    │
    ├─▶ DB: write row as PENDING
    │
    ▼  RestTemplate POST XML → destination institution
    │
    ├─▶ DB: update row to SENT (2xx) or FAILED (error/non-2xx)
    │
    ▼  Return raw XML response to caller
```

### Audit tables

**`transfer_log`** — one row per payment attempt
**`avs_log`** — one row per AVS verification

Both tables are created automatically in `dev` (`create-drop`). For `uat`/`prod` apply schema migrations manually before first run (`ddl-auto=validate`).

### Dev mock downstream

In the `dev` profile a `MockDownstreamController` runs on the same port and intercepts all outbound ISO calls locally — no external institution needed.

| Mock path | Simulates | Returns |
|---|---|---|
| `POST /mock/credit` | Creditor agent receiving pacs.008 | pacs.002 `TxSts=ACCC` |
| `POST /mock/avs` | Bank receiving acmt.023 | acmt.024 `verified=true` |
| `POST /mock/status` | Bank receiving pacs.028 | pacs.002 `TxSts=ACCC` |
| `POST /mock/return` | Bank receiving pacs.004 | pacs.002 `TxSts=ACCP` |

The mock controller is excluded from `uat` and `prod` builds (`@Profile("dev")`).

### Security

All API endpoints (except `/actuator/health`, `/h2-console/**` in dev, and `/mock/**` in dev) require a valid ZW-issued JWT in the `Authorization: Bearer` header. Capital validates the signature and expiry locally using the shared `jwt.secret` — no network call to ZW.

| Profile | Auth required |
|---|---|
| `dev` | Yes (JWT), `/h2-console/**` and `/mock/**` open |
| `uat` | Yes (JWT) |
| `prod` | Yes (JWT) |

### Profiles

| Profile | DB | `ddl-auto` |
|---|---|---|
| `dev` (default) | H2 in-memory | `create-drop` |
| `uat` | PostgreSQL `capital_uat` | `validate` |
| `prod` | PostgreSQL `capital_prod` | `validate` |

### Capital test script (`scripts/test-flow.sh`)

Exercises the full end-to-end payment flow. Authenticates with ZW first, then runs against Capital:

0. Login to ZW — obtain Bearer token
1. AVS — verify beneficiary account holder
2. Send — credit transfer (pacs.008)
3. Status — query payment status (pacs.028)
4. Return — return the payment (pacs.004)
5. Auth — token validation tests (no token → 401, malformed → 401, wrong signature → 401, valid → 200)

```bash
cd capital
./scripts/test-flow.sh

# Override endpoints
BASE_URL=http://payments:8460 ZW_URL=http://auth:8080 ./scripts/test-flow.sh

# Skip steps 1-2 if you already have a msgId
MSG_ID=MSG-ABCDEF1234567890 ./scripts/test-flow.sh

# Custom return reason
REASON_CODE=MD06 ./scripts/test-flow.sh
```

---

## Shared Configuration — JWT Secret

Both services must be configured with the **same** `jwt.secret` value. ZW uses it to sign tokens; Capital uses it to verify them.

| Profile | ZW | Capital |
|---|---|---|
| `dev` | `${JWT_SECRET:hCMdSpWeMIn1UhHJd0d9fsNQEanV6BZ5VVQm7wlTqtF}` | `${JWT_SECRET:hCMdSpWeMIn1UhHJd0d9fsNQEanV6BZ5VVQm7wlTqtF}` |
| `uat` | `${JWT_SECRET:hCMdSpWeMIn1UhHJd0d9fsNQEanV6BZ5VVQm7wlTqtF}` | `${JWT_SECRET:hCMdSpWeMIn1UhHJd0d9fsNQEanV6BZ5VVQm7wlTqtF}` |
| `prod` | `${JWT_SECRET}` **(required)** | `${JWT_SECRET}` **(required)** |

Set the `JWT_SECRET` environment variable on both servers when running `uat` or `prod`. The secret must be at least 32 characters.

---

## Deployment

See `deploy.yml` (Ansible) for automated deployment. Manual steps:

```bash
# ZW
cd zw
mvn clean package -P prod -DskipTests
JWT_SECRET=<secret> java -jar target/zw-0.0.1-SNAPSHOT.jar

# Capital
cd capital
mvn clean package -P prod -DskipTests
JWT_SECRET=<secret> java -jar target/capital-0.0.1-SNAPSHOT.jar
```
