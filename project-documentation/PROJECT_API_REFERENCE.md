# ApexPay - Master API Reference & Contract Specification

This document provides complete, endpoint-by-endpoint API documentation for both microservices powering **ApexPay**:
1. **Core Banking & Payment Gateway Backend:** Node.js / Express (Port `4000`)
2. **Intelligent Pre-Settlement Fraud Service:** Python / FastAPI (Port `8000`)

---

# SECTION 1: CORE BACKEND APIS (PORT 4000)

**Base URL:** `http://localhost:4000`  
**Standard Error Envelope:**
```json
{
  "ok": false,
  "error": "ERROR_CODE",
  "message": "Human readable explanation",
  "details": []
}
```

---

## 1. Authentication & Onboarding APIs (`/api/auth`)

### 1.1 Initiate Registration
- **HTTP Method:** `POST`
- **Endpoint URL:** `/api/auth/register/initiate`
- **Purpose:** Initiates multi-step user onboarding by verifying mobile number presence in the Central Bank Registry and dispatching an OTP.
- **Authentication:** Public
- **Request Headers:** `Content-Type: application/json`
- **Request Body:**
  ```json
  {
    "mobileNumber": "9000000001",
    "firstName": "Aarav",
    "lastName": "Sender"
  }
  ```
  - `mobileNumber` (String, Required): Exactly 10 digits regex `^\d{10}$`.
  - `firstName` (String, Required): Minimum 1 character.
  - `lastName` (String, Required): Minimum 1 character.
- **Backend Processing:**
  1. Validates body via Zod `registerInitiateSchema`.
  2. Queries `users` table: checks if `mobile_number` is already registered; throws `USER_ALREADY_EXISTS` if found.
  3. Queries `bank_accounts` table: checks if `mobileNumber` is linked to any bank; throws `NUMBER_NOT_LINKED_TO_BANK` if absent.
  4. Generates 6-digit cryptographically random OTP via `issueMobileOtp`.
  5. Computes SHA-256 hash of OTP and writes to Redis key `otp:register:<mobile>` with 300s TTL.
- **Database & Redis Operations:**
  - DB: `SELECT id FROM users WHERE mobile_number = $1`
  - DB: `SELECT ... FROM bank_accounts ba JOIN banks b ON b.id = ba.bank_id WHERE ba.mobile_number = $1`
  - Redis: `SET otp:register:<mobile> '{"hash":"...","attempts":0}' EX 300`
- **Success Response:** `200 OK`
  ```json
  {
    "ok": true,
    "otpDebug": "482910",
    "availableBanks": [
      {
        "id": "aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa",
        "bankName": "Apex Bank",
        "accountNumber": "ACT-111111"
      }
    ]
  }
  ```
- **Possible Errors:**
  - `400 Bad Request` - `USER_ALREADY_EXISTS`
  - `400 Bad Request` - `NUMBER_NOT_LINKED_TO_BANK`
  - `400 Bad Request` - `VALIDATION_ERROR` (Invalid mobile regex)

---

### 1.2 Verify Registration OTP
- **HTTP Method:** `POST`
- **Endpoint URL:** `/api/auth/register/verify-otp`
- **Purpose:** Validates the 6-digit OTP submitted by the user during onboarding.
- **Authentication:** Public
- **Request Body:**
  ```json
  {
    "mobileNumber": "9000000001",
    "otp": "482910"
  }
  ```
- **Backend Processing:**
  1. Reads `otp:register:<mobile>` from Redis. Throws `OTP_EXPIRED` if missing.
  2. Enforces attempt threshold (locks out after 3 attempts with `OTP_LOCKED`).
  3. Verifies SHA-256 hash. If invalid, increments attempt counter and throws `OTP_INVALID`.
  4. On match, sets verification flag `otp:verified:register:<mobile> = "true"` with 600s TTL.
  5. Deletes temporary OTP key from Redis.
- **Database & Redis Operations:**
  - Redis: `GET otp:register:<mobile>`
  - Redis: `SET otp:verified:register:<mobile> "true" EX 600`
  - Redis: `DEL otp:register:<mobile>`
- **Success Response:** `200 OK`
  ```json
  {
    "ok": true
  }
  ```
- **Possible Errors:**
  - `400 Bad Request` - `OTP_EXPIRED`
  - `400 Bad Request` - `OTP_INVALID`
  - `400 Bad Request` - `OTP_LOCKED`

---

### 1.3 Set PIN & Complete Registration
- **HTTP Method:** `POST`
- **Endpoint URL:** `/api/auth/register/set-pin`
- **Purpose:** Finalizes user onboarding by verifying the OTP clearance flag, hashing the user's permanent PIN, creating their account profile, and mapping a default VPA.
- **Authentication:** Public (requires verified OTP flag in Redis)
- **Request Body:**
  ```json
  {
    "mobileNumber": "9000000001",
    "firstName": "Aarav",
    "lastName": "Sender",
    "pin": "1234",
    "bankAccountId": "bbbbbbbb-bbbb-bbbb-bbbb-bbbbbbbbbbbb"
  }
  ```
- **Backend Processing:**
  1. Checks `checkOtpVerified(mobileNumber)`. Throws `OTP_NOT_VERIFIED` if missing.
  2. Queries `bank_accounts` by `bankAccountId`: validates ownership and active bank status.
  3. Hashes `pin` using bcrypt (10 salt rounds).
  4. Opens atomic PostgreSQL transaction (`BEGIN`):
     - Inserts record into `users` returning `userId`.
     - Inserts record into `accounts` copying initial balance from `bank_accounts`.
     - Derives default VPA `${firstName.toLowerCase()}${mobile.slice(-4)}@apex` and inserts into `vpas`.
     - Commits transaction (`COMMIT`).
  5. Purges verification flag `otp:verified:register:<mobile>` from Redis.
- **Success Response:** `200 OK`
  ```json
  {
    "ok": true,
    "userId": "11111111-1111-1111-1111-111111111111",
    "vpa": "aarav0001@apex"
  }
  ```
- **Possible Errors:**
  - `400 Bad Request` - `OTP_NOT_VERIFIED`
  - `400 Bad Request` - `INVALID_BANK_ACCOUNT`
  - `400 Bad Request` - `VALIDATION_ERROR`

---

### 1.4 Initiate Login
- **HTTP Method:** `POST`
- **Endpoint URL:** `/api/auth/login/initiate`
- **Purpose:** Initiates 2-factor authentication login by checking user registration and issuing a login OTP.
- **Authentication:** Public
- **Request Body:**
  ```json
  {
    "mobileNumber": "9000000001"
  }
  ```
- **Backend Processing:**
  1. Checks if mobile exists in `users` table; throws `USER_NOT_FOUND` if absent.
  2. Issues 6-digit OTP, stores SHA-256 hash in Redis key `otp:login:<mobile>` with 300s TTL.
- **Success Response:** `200 OK`
  ```json
  {
    "ok": true,
    "otpDebug": "592813"
  }
  ```
- **Possible Errors:**
  - `400 Bad Request` - `USER_NOT_FOUND`

---

### 1.5 Verify Login OTP
- **HTTP Method:** `POST`
- **Endpoint URL:** `/api/auth/login/verify-otp`
- **Purpose:** Validates the login OTP and sets a 5-minute pre-clearance session in Redis.
- **Authentication:** Public
- **Request Body:**
  ```json
  {
    "mobileNumber": "9000000001",
    "otp": "592813"
  }
  ```
- **Backend Processing:**
  1. Verifies SHA-256 hash against `otp:login:<mobile>` in Redis.
  2. Sets temporary flag `login:otp_verified:<mobile> = "true"` with 300s TTL.
- **Success Response:** `200 OK`
  ```json
  {
    "ok": true
  }
  ```
- **Possible Errors:**
  - `400 Bad Request` - `OTP_EXPIRED`
  - `400 Bad Request` - `OTP_INVALID`

---

### 1.6 Verify PIN & Issue JWT
- **HTTP Method:** `POST`
- **Endpoint URL:** `/api/auth/login/verify-pin`
- **Purpose:** Completes login by verifying the user's permanent PIN against `pin_hash` and generating a signed JWT session token.
- **Authentication:** Public (requires verified login OTP flag)
- **Request Body:**
  ```json
  {
    "mobileNumber": "9000000001",
    "pin": "1234"
  }
  ```
- **Backend Processing:**
  1. Verifies that `login:otp_verified:<mobile>` exists in Redis; throws `OTP_NOT_VERIFIED` if absent.
  2. Queries `users` for `pin_hash`.
  3. Compares PIN using `bcrypt.compare`. Throws `INVALID_PIN` on mismatch.
  4. Issues signed JWT with payload `{ userId, mobile, name }` and 24-hour expiration.
  5. Queries primary VPA from `vpas` table.
  6. Deletes Redis login flag.
- **Success Response:** `200 OK`
  ```json
  {
    "ok": true,
    "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "user": {
      "id": "11111111-1111-1111-1111-111111111111",
      "firstName": "Aarav",
      "lastName": "Sender",
      "mobile": "9000000001",
      "vpa": "aarav@apex"
    }
  }
  ```
- **Possible Errors:**
  - `400 Bad Request` - `OTP_NOT_VERIFIED`
  - `400 Bad Request` - `USER_NOT_FOUND`
  - `400 Bad Request` - `INVALID_PIN`

---

## 2. Core Banking Simulation APIs (`/api/bank`)

### 2.1 Create Central Bank Account
- **HTTP Method:** `POST`
- **Endpoint URL:** `/api/bank/accounts`
- **Purpose:** Creates a simulated bank account in the Central Bank Registry (`bank_accounts`).
- **Authentication:** Public
- **Request Body:**
  ```json
  {
    "firstName": "Sujal",
    "lastName": "Dingankar",
    "aadharNumber": "123456789012",
    "mobileNumber": "9876543210",
    "accountNumber": "ACT-998877",
    "bankId": "aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa",
    "ifscCode": "APEX0001",
    "balance": 50000
  }
  ```
- **Success Response:** `200 OK`
  ```json
  {
    "ok": true,
    "accountId": "cccccccc-cccc-cccc-cccc-cccccccccccc"
  }
  ```

---

### 2.2 List Supported Banks
- **HTTP Method:** `GET`
- **Endpoint URL:** `/api/bank/list`
- **Purpose:** Retrieves the list of available banking institutions.
- **Authentication:** Public
- **Success Response:** `200 OK`
  ```json
  {
    "ok": true,
    "banks": [
      {
        "id": "aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa",
        "name": "Apex Bank"
      },
      {
        "id": "bbbbbbbb-bbbb-bbbb-bbbb-bbbbbbbbbbbb",
        "name": "Neo Bank"
      }
    ]
  }
  ```

---

## 3. Payment Processing Gateway APIs (`/payment`)

### 3.1 Initiate Payment
- **HTTP Method:** `POST`
- **Endpoint URL:** `/payment/initiate`
- **Purpose:** Initiates an end-to-end payment transaction. Evaluates multi-dimensional risk signals, queries the ML service, creates transaction records, and determines whether to auto-approve, hold, or reject.
- **Authentication:** Required (`Bearer <token>`)
- **Request Headers:**
  - `Authorization: Bearer <jwt_token>`
  - `Content-Type: application/json`
- **Request Body:**
  ```json
  {
    "idempotencyKey": "req_unique_99812",
    "senderVpa": "aarav@apex",
    "receiverVpa": "isha@neo",
    "receiverMobile": "9000000002",
    "amount": 2500.0,
    "deviceId": "device_web_chrome_1",
    "location": "IN-MH",
    "pin": "1234",
    "paymentNote": "Dinner bill split"
  }
  ```
  *(Note: Either `receiverVpa` or `receiverMobile` must be supplied).*
- **Backend Processing:**
  1. Validates PIN against `users.pin_hash` via `bcrypt.compare` (if supplied).
  2. Idempotency Check: if `idempotencyKey` exists in `transactions`, returns existing transaction status immediately.
  3. Payee Resolution: resolves destination user ID, account ID, and VPA from `vpas` or `users`. Throws `SAME_ACCOUNT` if sender equals receiver.
  4. Behavioral Analytics (PostgreSQL + Redis):
     - Calculates 24h frequency (`getTxnCount24h`).
     - Calculates 30-day mean and standard deviation (`getUserAmountStats30d`).
     - Computes Z-score: $Z = \frac{\text{amount} - \mu}{\sigma}$.
     - Checks device and location anomalies against Redis anchors.
     - Checks payee suspicion score and blacklist status (`receiver_profiles`).
     - Checks first-transaction cold-start flag.
  5. ML Microservice Query: sends 17-feature payload to `POST http://localhost:8000/predict-risk`.
  6. Decision Dispatch:
     - `HIGH`: Sets status `'FAILED'`, `failure_reason = 'RISK_REJECTED'`. Writes audit log, dispatches alert notification.
     - `MEDIUM`: Sets status `'HOLD'`, `otpRequired = true`, sets `hold_expires_at = now() + 45s`. Reserves in-flight funds via Redis (`reserveFunds`). Generates 6-digit challenge OTP.
     - `LOW`: Sets status `'APPROVED'`. Immediately invokes `processTransaction()` for two-phase settlement.
  7. Updates user device and location behavioral anchors in Redis (30-day TTL).
- **Success Response (Auto-Approved - LOW Risk):** `200 OK`
  ```json
  {
    "txnId": "9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d",
    "status": "COMPLETED",
    "decision": "LOW",
    "riskScore": 0.1245,
    "reasons": [
      "Normal transaction behavior",
      "Familiar counterparty history"
    ],
    "fraudType": "UNKNOWN",
    "otpRequired": false
  }
  ```
- **Success Response (Challenged - MEDIUM Risk):** `200 OK`
  ```json
  {
    "txnId": "9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d",
    "status": "HOLD",
    "decision": "MEDIUM",
    "riskScore": 0.5210,
    "reasons": [
      "Noticeable deviation from recent spending pattern (|z| >= 1.5)",
      "New unrecognized device detected"
    ],
    "fraudType": "ACCOUNT_TAKEOVER",
    "holdExpiresAt": "2026-11-02T18:45:00.000Z",
    "otpRequired": true,
    "otpDebug": "829104"
  }
  ```
- **Possible Errors:**
  - `401 Unauthorized` - `UNAUTHORIZED` / `INVALID_TOKEN`
  - `400 Bad Request` - `INVALID_PIN`
  - `400 Bad Request` - `SAME_ACCOUNT`
  - `404 Not Found` - `VPA_NOT_FOUND` / `MOBILE_NOT_FOUND_IN_APEXPAY`
  - `500 Internal Server Error` - `ML_SERVICE_500`

---

### 3.2 Confirm Hold Payment (Step-Up Verification)
- **HTTP Method:** `POST`
- **Endpoint URL:** `/payment/confirm`
- **Purpose:** Verifies the challenge OTP and user PIN for a transaction in `HOLD` status, transitions it to `APPROVED`, and initiates two-phase ledger settlement.
- **Authentication:** Required (`Bearer <token>`)
- **Request Body:**
  ```json
  {
    "txnId": "9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d",
    "otp": "829104",
    "pin": "1234"
  }
  ```
- **Backend Processing:**
  1. Queries transaction status. Validates `status === 'HOLD'`.
  2. Validates `hold_expires_at > now()`. Throws `HOLD_EXPIRED` if deadline passed.
  3. Verifies OTP against `otp:txn:<txnId>` in Redis (max 5 attempts).
  4. Verifies PIN (`1234`).
  5. Updates `transactions` set `status = 'APPROVED'`, `confirmed_at = now()`.
  6. Writes audit log `HOLD_CONFIRMED`.
  7. Invokes `processTransaction({ txnId, asRecovery: false })`.
- **Success Response:** `200 OK`
  ```json
  {
    "id": "9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d",
    "status": "COMPLETED",
    "amount": 2500.0,
    "sender_user_id": "11111111-1111-1111-1111-111111111111",
    "receiver_user_id": "22222222-2222-2222-2222-222222222222",
    "risk_score": 0.5210,
    "decision": "MEDIUM",
    "decision_reason": "Noticeable deviation from recent spending pattern",
    "fraud_type": "ACCOUNT_TAKEOVER",
    "otp_required": true,
    "hold_expires_at": "2026-11-02T18:45:00.000Z",
    "failure_reason": null,
    "retries": 0
  }
  ```
- **Possible Errors:**
  - `400 Bad Request` - `NOT_HOLD`
  - `400 Bad Request` - `HOLD_EXPIRED`
  - `400 Bad Request` - `PIN_INVALID`
  - `400 Bad Request` - `OTP_INVALID` / `OTP_EXPIRED`

---

### 3.3 Cancel Hold Payment
- **HTTP Method:** `POST`
- **Endpoint URL:** `/payment/cancel`
- **Purpose:** Allows a user to voluntarily decline a pending transaction in `HOLD` status.
- **Authentication:** Required (`Bearer <token>`)
- **Request Body:**
  ```json
  {
    "txnId": "9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d"
  }
  ```
- **Backend Processing:**
  1. Validates transaction is in `HOLD` status.
  2. Updates `transactions` set `status = 'FAILED'`, `failure_reason = 'CANCELED_BY_USER'`.
  3. Releases reserved funds from Redis via `releaseReservedFunds`.
  4. Appends `HOLD_CANCELED` entry in `audit_logs`.
  5. Dispatches cancellation notification.
- **Success Response:** `200 OK`
  ```json
  {
    "id": "9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d",
    "status": "FAILED",
    "failure_reason": "CANCELED_BY_USER"
  }
  ```

---

### 3.4 Get Payment Status
- **HTTP Method:** `GET`
- **Endpoint URL:** `/payment/status?id=<txnId>`
- **Purpose:** Polling endpoint for real-time status updates on active transactions.
- **Authentication:** Public
- **Query Parameters:**
  - `id` (String, Required): UUID of the transaction.
- **Success Response:** `200 OK`
  ```json
  {
    "id": "9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d",
    "status": "COMPLETED",
    "amount": 2500.0,
    "sender_user_id": "11111111-1111-1111-1111-111111111111",
    "sender_account_id": "aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa",
    "receiver_user_id": "22222222-2222-2222-2222-222222222222",
    "receiver_account_id": "bbbbbbbb-bbbb-bbbb-bbbb-bbbbbbbbbbbb",
    "risk_score": 0.1245,
    "decision": "LOW",
    "decision_reason": "Normal transaction behavior",
    "fraud_type": "UNKNOWN",
    "otp_required": false,
    "hold_expires_at": null,
    "failure_reason": null,
    "retries": 0
  }
  ```

---

### 3.5 Transaction History
- **HTTP Method:** `GET`
- **Endpoint URL:** `/payment/history`
- **Purpose:** Retrieves complete chronological ledger history for the authenticated user (both debits and credits) with comprehensive counterparty metadata.
- **Authentication:** Required (`Bearer <token>`)
- **Success Response:** `200 OK`
  ```json
  {
    "items": [
      {
        "id": "9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d",
        "amount": 2500.0,
        "status": "COMPLETED",
        "timestamp": "2026-11-02T18:30:00.000Z",
        "paymentNote": "Dinner bill split",
        "type": "DEBIT",
        "name": "Isha Receiver",
        "vpa": "isha@neo",
        "accountNumber": "ACT-222222",
        "bankName": "Neo Bank",
        "senderName": "Aarav Sender",
        "senderVpa": "aarav@apex",
        "senderAccountNumber": "ACT-111111",
        "senderBankName": "Apex Bank",
        "receiverName": "Isha Receiver",
        "receiverVpa": "isha@neo",
        "receiverAccountNumber": "ACT-222222",
        "receiverBankName": "Neo Bank"
      }
    ]
  }
  ```

---

## 4. Account & Notification APIs (`/account`, `/notifications`)

### 4.1 Resolve Payee
- **HTTP Method:** `GET`
- **Endpoint URL:** `/account/resolve?vpa=...` OR `/account/resolve?mobile=...`
- **Purpose:** Resolves recipient full name and VPA before payment initiation to prevent misdirected transfers.
- **Authentication:** Required (`Bearer <token>`)
- **Query Parameters:**
  - `vpa` (String, Optional)
  - `mobile` (String, Optional)
- **Success Response:** `200 OK`
  ```json
  {
    "success": true,
    "name": "Isha Receiver",
    "vpa": "isha@neo"
  }
  ```
- **Possible Errors:**
  - `404 Not Found` - `RECEIVER_NOT_FOUND`

---

### 4.2 Check Secure Balance
- **HTTP Method:** `GET`
- **Endpoint URL:** `/account/balance?pin=1234`
- **Purpose:** Secure balance inquiry requiring mandatory PIN verification.
- **Authentication:** Required (`Bearer <token>`)
- **Query Parameters:**
  - `pin` (String, Required): 4-to-6 digit user PIN.
- **Backend Processing:**
  1. Validates PIN against authenticated user's `pin_hash` using `bcrypt.compare`.
  2. Queries primary linked account balance from `accounts`.
- **Success Response:** `200 OK`
  ```json
  {
    "success": true,
    "vpa": "aarav@apex",
    "balance": 197500.0
  }
  ```
- **Possible Errors:**
  - `400 Bad Request` - `PIN_REQUIRED`
  - `401 Unauthorized` - `INVALID_PIN`

---

### 4.3 List Notifications
- **HTTP Method:** `GET`
- **Endpoint URL:** `/notifications?userId=<uuid>`
- **Purpose:** Returns the 20 most recent user notifications.
- **Authentication:** Public
- **Success Response:** `200 OK`
  ```json
  {
    "items": [
      {
        "id": "e81d723a-f2e1-4569-8d77-90c7490f84a1",
        "type": "PAYMENT_DEBITED",
        "message": "₹2500 debited from your account",
        "created_at": "2026-11-02T18:30:05.000Z"
      }
    ]
  }
  ```

---

## 5. Chaos Engineering & Simulation APIs (`/test`)

### 5.1 Simulate Settlement Failure
- **HTTP Method:** `POST`
- **Endpoint URL:** `/test/settlement-failure`
- **Purpose:** Sets an immediate simulated crash point for the subsequent transaction to test two-phase settlement recovery.
- **Authentication:** Public
- **Request Body:**
  ```json
  {
    "txnId": "00000000-0000-0000-0000-000000000000",
    "mode": "AFTER_DEBIT"
  }
  ```
  - `mode` (Enum): `'AFTER_DEBIT'` | `'BEFORE_CREDIT'`
- **Success Response:** `200 OK`
  ```json
  {
    "ok": true,
    "simulated": "AFTER_DEBIT"
  }
  ```

---

# SECTION 2: MACHINE LEARNING SERVICE APIS (PORT 8000)

**Base URL:** `http://localhost:8000`  
**Framework:** FastAPI / Pydantic v2  

---

## 1. Service Health Check

### `GET /health`
- **Purpose:** Verifies operational readiness of the Python ML engine.
- **Request:** None
- **Success Response:** `200 OK`
  ```json
  {
    "status": "ok",
    "model_version": "2.2.0"
  }
  ```

---

## 2. Model Metrics & Evaluation

### `GET /metrics`
- **Purpose:** Reports verified hold-out test set accuracy, precision, recall, and F1-score from `metrics.json`.
- **Request:** None
- **Success Response:** `200 OK`
  ```json
  {
    "accuracy": 0.9853,
    "precision": 0.9611,
    "recall": 0.9875,
    "f1_score": 0.9741,
    "model_version": "2.2.0"
  }
  ```

---

## 3. Real-Time Fraud & Risk Scoring

### `POST /predict-risk`
- **Purpose:** Executes synchronous 3-layer hybrid risk scoring (Heuristic Rules + Calibrated Random Forest + Directed Multigraph Intelligence).
- **Request Headers:** `Content-Type: application/json`
- **Request Payload Schema (`RiskRequest`):**
  ```json
  {
    "sender_id": "11111111-1111-1111-1111-111111111111",
    "receiver_id": "22222222-2222-2222-2222-222222222222",
    "transaction_id": "9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d",
    "amount": 2500.0,
    "txn_count_24h": 2,
    "device_change": 0,
    "location_anomaly": 0,
    "receiver_suspicion_score": 0.10,
    "receiver_flagged": 0,
    "user_avg_amount_30d": 2100.0,
    "user_std_amount_30d": 450.0,
    "amount_zscore": 0.88,
    "is_first_transaction": false,
    "user_total_txn": 14,
    "account_age_hours": 720.0,
    "relationship_stability_score": 0.85,
    "is_periodic_pattern": false,
    "day_of_month": 2,
    "is_high_amount_global": false,
    "amount_percentile_global": 0.45
  }
  ```
- **Inference Pipeline:**
  1. **Graph Ingestion:** Idempotently records directed edge `sender_id -> receiver_id` in RAM multigraph cache backed by SQLite WAL. Rejects placeholder super-nodes.
  2. **Layer 1 (Heuristic Rules):** Evaluates deterministic baseline rules, tiered amounts, velocity spikes, and legitimacy discounts.
  3. **Layer 2 (Calibrated Random Forest):** Vectorizes 17 standardized features and computes calibrated probability $P(\text{Fraud} = 1 \mid \mathbf{x})$ via Platt sigmoid scaling.
  4. **Smart Force Overrides:** Evaluates critical account takeover ($Z > 8.0$ or new device + location jump + amount $\ge 10\text{k}$) and mule fan-in funnels ($\ge 10$ senders in 24h).
  5. **Layer 3 (Graph Intelligence):** Runs bounded cycle DFS (detecting laundering rings $\ge 5$ nodes while exempting 3-node social splitting) and 14-day half-life neighbor risk decay.
  6. **Multi-Signal Score Fusion:** Blends scores adaptively:
     $$S_{\text{final}} = w_{\text{ML}} S_{\text{ML}} + w_{\text{rule}} S_{\text{rule}} + w_{\text{graph}} S_{\text{graph}}$$
  7. **Decision Gating:**
     - $S_{\text{final}} < 0.42 \implies \text{LOW}$ (Auto-Approve)
     - $0.42 \le S_{\text{final}} < 0.75 \implies \text{MEDIUM}$ (Step-Up Challenge)
     - $S_{\text{final}} \ge 0.75 \implies \text{HIGH}$ (Immediate Block)
- **Success Response:** `200 OK`
  ```json
  {
    "risk_score": 0.1245,
    "combined_risk_score": 0.1245,
    "ml_probability": 0.0812,
    "rule_score": 0.0800,
    "graph_score": 0.0500,
    "decision": "LOW",
    "reasons": [
      "Normal transaction behavior",
      "Familiar counterparty history"
    ],
    "fraud_type": "UNKNOWN",
    "model_version": "2.2.0"
  }
  ```
- **Possible Errors:**
  - `422 Unprocessable Entity` - Pydantic validation failure (negative amount, invalid types)
  - `500 Internal Server Error` - Model inference exception
