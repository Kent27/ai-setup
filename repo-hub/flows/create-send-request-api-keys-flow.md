# Create API Keys + Rate Limit Flow

This document describes the responsibilities and request flow across repositories for:

- **Creating connector API keys** (with `daily_limit` and `burst_limit_per_minute`)
- **Enforcing rate limits** (MariaDB + Redis)
- **Using the connector raw V2 endpoint** (PHP gateway → main-backend → legacy Data controller)

---

## Repositories and Roles

### `emobiq-main-backend`
- **Owns API key persistence** in MariaDB table `api_keys`.
- **Owns rate limit enforcement**:
  - Validates API key exists + active
  - Applies limits:
    - `daily_limit`
    - `burst_limit_per_minute`
  - Uses Redis counters (if configured) to enforce limits.
- Exposes internal endpoints for:
  - **Create API key** (called after checkout from subscription)
  - **Rate limit check** (called by connector raw V2 gateway)
  - **List API keys for dashboard** (called by subscription UI)

### `orangekloud-subscription`
- **Owns subscription checkout** and persistence of:
  - `orders` (active subscription is represented by `orders.order_status = 1`)
  - `subscriptions` + `subscription_attributes`
- Determines the plan (subscription) and reads the 2 connector-limit attributes:
  - `emobiqSendRequestDailyLimit`
  - `emobiqSendRequestBurstLimitPerMinute`
- Calls main-backend to **create API key** after checkout succeeds.
- Provides “license API” endpoint used by other services to fetch attributes by `idp_id`:
  - `POST /api/open/get_subscription_attr_by_idp`

### `emobiq-v5-platform`
- Provides **raw connector V2 gateway**:
  - Validates presence of API key header (extracts from request)
  - Calls main-backend **rate-limit check**
  - If allowed → delegates to existing legacy controller `controller/Data.php`

---

## Data Model (High-level)

### In `emobiq-main-backend` (MariaDB)
- `users.external_sso_user_id` = the subscription/SSO user id (aka **`idp_id`**)
- `api_keys` contains:
  - `user_id` (internal main-backend user id)
  - `api_key` (generated from code)
  - `daily_limit`
  - `burst_limit_per_minute`
  - `is_active`

### In `orangekloud-subscription` (DB)
- `users.idp_id` = the SSO user id (the value that maps to main-backend `users.external_sso_user_id`)
- Active plan is determined by:
  - `orders.user_id = <subscription user id>`
  - `orders.order_status = 1`
  - `orders.subscription_id != null`
- Plan code stored at:
  - `subscriptions.code` (e.g. `SP0007`)
- Connector limit attributes stored at:
  - `subscription_attributes.subscription_code = subscriptions.code`
  - `subscription_attributes.key` in:
    - `emobiqSendRequestDailyLimit`
    - `emobiqSendRequestBurstLimitPerMinute`

---

## End-to-end Flow

### A) Create API key (after checkout)

**Trigger:** successful checkout in `orangekloud-subscription` (`CartController::postOrder`).

1. Subscription app persists the order
   - `OrdersModel::createOrder(...)` inserts into table `orders` (Eloquent `create()`).
2. Subscription app queries connector limits from subscription attributes:
   - keys:
     - `emobiqSendRequestDailyLimit`
     - `emobiqSendRequestBurstLimitPerMinute`
3. Subscription app calls main-backend to create the API key:
   - **Endpoint:** `POST /v1/sso/user/:external_sso_user_id/api-keys`
   - **Auth:** ConfigAuth (`API-Key` header = `MAIN_BACKEND_API_KEY`)
   - **Body:**
     ```json
     {
       "daily_limit": 100000,
       "burst_limit_per_minute": 100
     }
     ```
4. Main-backend behavior:
   - Finds user by `users.external_sso_user_id = :external_sso_user_id`
   - If an `api_keys` row already exists for that user → **update** `daily_limit` and `burst_limit_per_minute` (same `api_key`)
   - Else inserts `api_keys` row with the provided limits

---

### B) Rate limit enforcement for connector raw V2

**Trigger:** a client calls the connector raw V2 endpoint in `emobiq-v5-platform`.

1. Client calls:
   - `POST /connector/raw/v2/?api=data&a=call` (and other query args as per legacy connector)
   - Must include header:
     - `X-API-Key: <api_key>`
2. Connector raw V2 (PHP) calls main-backend:
   - **Endpoint:** `POST /v1/rate-limit`
   - **Header:** `X-API-Key: <api_key>`
3. Main-backend `POST /v1/rate-limit`:
   - Looks up `api_keys.api_key = <api_key>` (must be active)
   - Reads `daily_limit` and `burst_limit_per_minute`
   - Uses Redis counters (if Redis configured) for:
     - daily key (expires at next midnight SGT)
     - burst per minute key (short TTL)
   - If exceeded → returns `429 Too Many Requests`
4. If allowed:
   - PHP connector delegates to existing legacy `controller/Data.php` which proxies to the external API.

---

## Key Endpoints Summary

### `emobiq-main-backend`
- **Create API key (called from subscription after checkout)**
  - `POST /v1/sso/user/:external_sso_user_id/api-keys`
  - Headers: `API-Key: <MAIN_BACKEND_API_KEY>`
  - Body: `daily_limit`, `burst_limit_per_minute`

- **List API keys (dashboard)**
  - `GET /v1/sso/user/:external_sso_user_id/api-keys`
  - Headers: `API-Key: <MAIN_BACKEND_API_KEY>`

- **Rate limit check (called from connector raw V2 gateway)**
  - `POST /v1/rate-limit`
  - Headers: `X-API-Key: <user_api_key>`

### `orangekloud-subscription`
- **Get subscription attributes by idp**
  - `POST /api/open/get_subscription_attr_by_idp`
  - Body: `{ "idp_id": <int> }`

### `emobiq-v5-platform`
- **Raw connector V2 entry**
  - `/connector/raw/v2/` (folder-based entry point)
  - Calls main-backend rate-limit endpoint before delegating to legacy Data controller

---

## Configuration Notes

### `orangekloud-subscription`
- `.env` must set:
  - `MAIN_BACKEND_URL` (should include `/v1` base, e.g. `http://localhost:8081/v1`)
  - `MAIN_BACKEND_API_KEY`

### `emobiq-main-backend`
- Must have ConfigAuth API key configured (used by subscription service).
- Redis config (if rate limiting enabled) is under `golang/config/database.json` → `redis.connector_raw_rate_limit`.

---
