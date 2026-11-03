# ApexPay - Master System Workflows & Sequence Specifications

This document specifies the end-to-end operational workflows, control logic, state machines, and sequence diagrams for all core transaction and security mechanisms implemented in **ApexPay**.

---

## 1. High-Level Transaction State Machine

The lifecycle of every payment in ApexPay transitions through a strict deterministic finite state automaton governed by the `transaction_status` PostgreSQL enum:

```mermaid
stateDiagram-v2
    [*] --> REQUESTED : /payment/initiate

    REQUESTED --> FAILED : Risk == HIGH (RISK_REJECTED)
    REQUESTED --> HOLD : Risk == MEDIUM (Step-Up Challenge)
    REQUESTED --> APPROVED : Risk == LOW (Auto-Approve)

    HOLD --> APPROVED : /payment/confirm (OTP + PIN Valid)
    HOLD --> FAILED : /payment/cancel (Canceled by user)
    HOLD --> FAILED : expireHolds() (45s TTL Elapsed)

    APPROVED --> COMPLETED : processTransaction() [Phase 1 + Phase 2 OK]
    APPROVED --> APPROVED : Crash / Failure (retries < 3)
    APPROVED --> RECOVERED : recoverStuckApproved() [Retry Success]
    APPROVED --> FAILED : rollbackTransaction() [retries >= 3, Refunded]

    COMPLETED --> [*]
    RECOVERED --> [*]
    FAILED --> [*]
```

---

## 2. End-to-End Workflow Specifications

### 2.1 Workflow 1: Central Bank Registry Simulation
**Objective:** Create a verified source-of-truth bank account in the Central Bank Registry before onboarding onto ApexPay.

```mermaid
sequenceDiagram
    autonumber
    actor User as User / Admin
    participant UI as BankPage (Frontend)
    participant BC as BankController
    participant BS as BankService
    participant DB as PostgreSQL (bank_accounts)

    User->>UI: Enter Legal Name, Aadhar, Mobile, Account No, Bank, IFSC, Initial Balance
    UI->>BC: POST /api/bank/accounts
    BC->>BC: Validate CreateAccountSchema via Zod
    BC->>BS: createBankAccount(data)
    BS->>DB: INSERT INTO bank_accounts (...) RETURNING id
    DB-->>BS: accountId
    BS-->>BC: accountId
    BC-->>UI: 200 OK { ok: true, accountId }
    UI-->>User: Display Success Banner ("Account linked to Bank Registry")
```

---

### 2.2 Workflow 2: Multi-Step User Registration & Onboarding
**Objective:** Verify that the user's mobile number exists in the Central Bank Registry, issue an ephemeral SMS OTP, verify the OTP, hash a secure PIN, and create the user's primary linked wallet and default VPA.

```mermaid
sequenceDiagram
    autonumber
    actor User as User
    participant UI as Register Modal (Frontend)
    participant AC as AuthController
    participant AS as AuthService
    participant OS as OtpService
    participant R as Upstash Redis
    participant DB as PostgreSQL

    Note over User,DB: Step 1: Registration Initiation
    User->>UI: Enters Mobile Number, First Name, Last Name
    UI->>AC: POST /api/auth/register/initiate
    AC->>AS: initiateRegistration(mobileNumber, firstName, lastName)
    AS->>DB: SELECT id FROM users WHERE mobile_number = $1
    alt User Already Exists
        AS-->>AC: Error: USER_ALREADY_EXISTS
        AC-->>UI: 400 Bad Request
    end
    AS->>DB: SELECT * FROM bank_accounts WHERE mobile_number = $1
    alt Mobile Not in Central Bank Registry
        AS-->>AC: Error: NUMBER_NOT_LINKED_TO_BANK
        AC-->>UI: 400 Bad Request
    end
    AS->>OS: issueMobileOtp(mobile, 300s, "REGISTER")
    OS->>OS: Generate 6-digit OTP & compute SHA-256 hash
    OS->>R: SET otp:register:<mobile> { hash, attempts: 0 } EX 300
    OS-->>AS: otp
    AS-->>AC: { otpDebug: otp, availableBanks }
    AC-->>UI: 200 OK { availableBanks, otpDebug }

    Note over User,DB: Step 2: OTP Verification
    User->>UI: Enters 6-digit OTP
    UI->>AC: POST /api/auth/register/verify-otp
    AC->>AS: verifyRegistrationOtp(mobile, otp)
    AS->>OS: verifyMobileOtp(mobile, otp, 3, "REGISTER")
    OS->>R: GET otp:register:<mobile>
    OS->>OS: Compare SHA-256(otp) == stored.hash
    alt Hash Matches
        OS->>R: SET otp:verified:register:<mobile> "true" EX 600
        OS->>R: DEL otp:register:<mobile>
        OS-->>AS: OK
        AS-->>AC: { ok: true }
        AC-->>UI: 200 OK { ok: true }
    else Hash Mismatch
        OS->>R: Update attempts += 1
        OS-->>AC: Error: OTP_INVALID
        AC-->>UI: 400 Bad Request
    end

    Note over User,DB: Step 3: PIN Setup & Account Creation
    User->>UI: Selects Bank Account & Enters 4-digit PIN
    UI->>AC: POST /api/auth/register/set-pin
    AC->>AS: setPinAndCompleteRegistration(data)
    AS->>OS: checkOtpVerified(mobile)
    alt OTP Verified Flag Missing in Redis
        AS-->>AC: Error: OTP_NOT_VERIFIED
        AC-->>UI: 400 Bad Request
    end
    AS->>DB: SELECT * FROM bank_accounts WHERE id = bankAccountId
    AS->>AS: bcrypt.hash(pin, 10 rounds)
    AS->>DB: BEGIN TRANSACTION
    AS->>DB: INSERT INTO users (first_name, last_name, mobile_number, pin_hash) RETURNING id
    AS->>DB: INSERT INTO accounts (user_id, bank_id, balance) VALUES (userId, bankId, bankBalance)
    AS->>DB: Derive VPA: firstName + mobile.slice(-4) + '@apex'
    AS->>DB: INSERT INTO vpas (user_id, bank_id, account_id, vpa)
    AS->>DB: COMMIT TRANSACTION
    AS->>R: DEL otp:verified:register:<mobile>
    AS-->>AC: { userId, vpa }
    AC-->>UI: 200 OK { userId, vpa }
    UI-->>User: Registration Complete -> Redirect to Dashboard
```

---

### 2.3 Workflow 3: Two-Factor Authentication Login
**Objective:** Authenticate user via two independent factors: temporary SMS OTP (Possession) and permanent bcrypt PIN (Knowledge).

```mermaid
sequenceDiagram
    autonumber
    actor User as User
    participant UI as Login Modal (Frontend)
    participant AC as AuthController
    participant AS as AuthService
    participant OS as OtpService
    participant R as Upstash Redis
    participant DB as PostgreSQL

    Note over User,DB: Step 1: Login Initiation
    User->>UI: Enters Mobile Number
    UI->>AC: POST /api/auth/login/initiate
    AC->>AS: initiateLogin(mobileNumber)
    AS->>DB: SELECT id FROM users WHERE mobile_number = $1
    alt User Not Found
        AS-->>AC: Error: USER_NOT_FOUND
        AC-->>UI: 400 Bad Request
    end
    AS->>OS: issueMobileOtp(mobile, 300s, "LOGIN")
    OS->>R: SET otp:login:<mobile> { hash, attempts: 0 } EX 300
    OS-->>AS: otp
    AS-->>AC: { otpDebug: otp }
    AC-->>UI: 200 OK { ok: true, otpDebug }

    Note over User,DB: Step 2: Login OTP Verification
    User->>UI: Enters 6-digit OTP
    UI->>AC: POST /api/auth/login/verify-otp
    AC->>AS: verifyLoginOtp(mobile, otp)
    AS->>OS: verifyMobileOtp(mobile, otp, 3, "LOGIN")
    OS->>R: GET otp:login:<mobile>
    OS->>OS: Verify SHA-256 hash
    OS->>R: SET login:otp_verified:<mobile> "true" EX 300
    OS->>R: DEL otp:login:<mobile>
    OS-->>AS: OK
    AS-->>AC: { ok: true }
    AC-->>UI: 200 OK

    Note over User,DB: Step 3: PIN Verification & JWT Issuance
    User->>UI: Enters Permanent 4-to-6 digit PIN
    UI->>AC: POST /api/auth/login/verify-pin
    AC->>AS: verifyPinAndLogin(mobile, pin)
    AS->>R: GET login:otp_verified:<mobile>
    alt Flag != "true"
        AS-->>AC: Error: OTP_NOT_VERIFIED
        AC-->>UI: 400 Bad Request
    end
    AS->>DB: SELECT id, first_name, last_name, pin_hash FROM users WHERE mobile_number = $1
    AS->>AS: bcrypt.compare(pin, user.pin_hash)
    alt PIN Mismatch
        AS-->>AC: Error: INVALID_PIN
        AC-->>UI: 400 Bad Request
    end
    AS->>AS: jwt.sign({ userId, mobile, name }, JWT_SECRET, 24h)
    AS->>DB: SELECT vpa FROM vpas WHERE user_id = userId
    AS->>R: DEL login:otp_verified:<mobile>
    AS-->>AC: { token, user: { id, firstName, lastName, mobile, vpa } }
    AC-->>UI: 200 OK { token, user }
    UI->>UI: Save apexpay_token & apexpay_user in localStorage
    UI-->>User: Logged In -> Dashboard Active
```

---

### 2.4 Workflow 4: Payment Initiation & Pre-Settlement Risk Scoring (LOW Risk Path)
**Objective:** Process a low-risk transaction instantaneously without user friction.

```mermaid
sequenceDiagram
    autonumber
    actor Payer as Payer
    participant UI as Dashboard (Frontend)
    participant PC as PaymentController
    participant PS as PaymentService
    participant BS as BehaviorService
    participant ML as FastAPI ML Service (Port 8000)
    participant DB as PostgreSQL
    participant R as Upstash Redis

    Payer->>UI: Enter Payee VPA, Amount (e.g. ₹500), PIN, Memo
    UI->>PC: POST /payment/initiate (Headers: Bearer <JWT>)
    PC->>PC: authMiddleware: Decodes req.user
    PC->>PS: initiatePayment(payload + senderUserId)
    PS->>DB: Verify sender PIN via bcrypt.compare
    PS->>DB: Resolve sender & receiver accounts and VPAs
    PS->>BS: getTxnCount24h(senderUserId) -> Count
    PS->>BS: getUserAmountStats30d(senderUserId) -> Avg, Std
    PS->>BS: computeDeviceChange & computeLocationAnomaly via Redis
    PS->>DB: Query receiver_profiles (suspicion_score, flagged)
    PS->>PS: Compute amount_zscore = (amount - avg) / std
    PS->>PS: Check is_first_transaction

    PS->>ML: POST /predict-risk (17-feature JSON payload)
    ML->>ML: Layer 1: Heuristic Rules Evaluation
    ML->>ML: Layer 2: Calibrated Random Forest (150 trees)
    ML->>ML: Layer 3: Directed Multigraph & Cycle DFS
    ML->>ML: Adaptive Score Blending & Decision Gating
    ML-->>PS: Response { risk_score: 0.12, decision: "LOW", reasons, fraud_type: "UNKNOWN" }

    PS->>DB: INSERT INTO transactions (..., status: 'APPROVED', risk_score: 0.12) RETURNING txnId
    PS->>DB: INSERT INTO risk_scores (txn_id, risk_score, decision, risk_features)
    PS->>DB: INSERT INTO audit_logs (txn_id, action: 'PAYMENT_INITIATED')
    PS->>R: Update user:last_device & user:last_location (30d TTL)

    Note over PS,DB: Synchronous Two-Phase Settlement Execution
    PS->>PS: processTransaction({ txnId, asRecovery: false })
    PS->>R: acquireLock("user_lock:<sender_id>", TTL: 8000ms)
    
    Note over PS,DB: Phase 1: Sender Debit
    PS->>DB: BEGIN TRANSACTION
    PS->>DB: SELECT balance FROM accounts WHERE id = sender_acc FOR UPDATE
    PS->>DB: UPDATE accounts SET balance = balance - amount WHERE id = sender_acc
    PS->>DB: UPDATE transactions SET failure_reason = 'SENDER_DEBITED' WHERE id = txnId
    PS->>DB: COMMIT TRANSACTION
    PS->>DB: INSERT INTO audit_logs ('SENDER_DEBIT_SUCCESS')
    PS->>DB: INSERT INTO notifications ('PAYMENT_DEBITED')

    Note over PS,DB: Phase 2: Receiver Credit
    PS->>DB: BEGIN TRANSACTION
    PS->>DB: SELECT balance FROM accounts WHERE id = receiver_acc FOR UPDATE
    PS->>DB: UPDATE accounts SET balance = balance + amount WHERE id = receiver_acc
    PS->>DB: UPDATE transactions SET status = 'COMPLETED', failure_reason = NULL
    PS->>DB: COMMIT TRANSACTION
    PS->>DB: INSERT INTO audit_logs ('PAYMENT_COMPLETED')
    PS->>DB: INSERT INTO notifications ('PAYMENT_RECEIVED')
    PS->>R: releaseLock("user_lock:<sender_id>")

    PS-->>PC: { txnId, status: "COMPLETED", decision: "LOW", riskScore: 0.12 }
    PC-->>UI: 200 OK
    UI-->>Payer: Display Green Payment Receipt Modal
```

---

### 2.5 Workflow 5: Payment Initiation with Step-Up Challenge (MEDIUM Risk Path)
**Objective:** Suspend an anomalous transaction on `HOLD` for 45 seconds, reserve in-flight funds, issue an OTP challenge, and settle upon verification.

```mermaid
sequenceDiagram
    autonumber
    actor Payer as Payer
    participant UI as Dashboard (Frontend)
    participant PC as PaymentController
    participant PS as PaymentService
    participant RS as ReserveService
    participant OS as OtpService
    participant ML as FastAPI ML Service
    participant DB as PostgreSQL
    participant R as Upstash Redis

    Payer->>UI: Enters Anomalous Payment (e.g. ₹25,000 from new device)
    UI->>PC: POST /payment/initiate
    PC->>PS: initiatePayment(...)
    PS->>ML: POST /predict-risk
    ML-->>PS: Response { risk_score: 0.58, decision: "MEDIUM", reasons: ["New device", "High Z-Score"] }

    PS->>DB: INSERT INTO transactions (status: 'HOLD', otp_required: true, hold_expires_at: now() + 45s) RETURNING txnId
    PS->>RS: reserveFunds({ accountId, amount, balanceLimit })
    RS->>R: acquireLock("reserve_lock:<accId>")
    RS->>R: GET reserve:account:<accId> -> current
    RS->>RS: Assert (current + amount <= balanceLimit)
    RS->>R: SET reserve:account:<accId> (current + amount) EX 3600
    RS->>R: releaseLock("reserve_lock:<accId>")

    PS->>OS: issueOtp({ txnId, ttlSeconds: 45 })
    OS->>R: SET otp:txn:<txnId> { hash, attempts: 0 } EX 45
    PS->>DB: INSERT INTO notifications ('HOLD_TRIGGERED')
    PS-->>PC: { txnId, status: "HOLD", holdExpiresAt, otpRequired: true, otpDebug }
    PC-->>UI: 200 OK { status: "HOLD", otpDebug }

    UI->>UI: Launch 45-Second Countdown Timer & OTP Input Modal
    UserVerification:
    alt Payer Enters Valid OTP within 45 seconds
        Payer->>UI: Submits OTP + PIN
        UI->>PC: POST /payment/confirm { txnId, otp, pin }
        PC->>PS: confirmPayment(...)
        PS->>DB: SELECT status, hold_expires_at FROM transactions WHERE id = txnId
        PS->>PS: Assert status == 'HOLD' AND hold_expires_at > now()
        PS->>OS: verifyOtp({ txnId, otp, maxAttempts: 5 })
        OS->>R: GET otp:txn:<txnId> & verify SHA-256
        OS->>R: DEL otp:txn:<txnId>
        PS->>DB: UPDATE transactions SET status = 'APPROVED', confirmed_at = now()
        PS->>DB: INSERT INTO audit_logs ('HOLD_CONFIRMED')
        PS->>PS: processTransaction({ txnId, asRecovery: false })
        PS->>RS: releaseReservedFunds({ accountId, amount })
        PS-->>PC: Status: COMPLETED
        PC-->>UI: 200 OK { status: "COMPLETED" }
        UI-->>Payer: Display Success Receipt
    else 45-second Timer Expires Without Verification
        Note over PS,DB: Background Hold Expiration Sweep (Every 5s)
        PS->>DB: SELECT id FROM transactions WHERE status = 'HOLD' AND hold_expires_at < now()
        PS->>DB: UPDATE transactions SET status = 'FAILED', failure_reason = 'HOLD_EXPIRED'
        PS->>RS: releaseReservedFunds({ accountId, amount })
        PS->>DB: INSERT INTO audit_logs ('HOLD_EXPIRED_AUTO')
        PS->>DB: INSERT INTO notifications ('PAYMENT_STATUS', 'HOLD expired')
        UI->>UI: Status polling detects FAILED -> Displays "Transaction Timed Out"
    else Payer Clicks "Cancel Payment"
        Payer->>UI: Clicks Cancel
        UI->>PC: POST /payment/cancel { txnId }
        PC->>PS: cancelPayment(...)
        PS->>DB: UPDATE transactions SET status = 'FAILED', failure_reason = 'CANCELED_BY_USER'
        PS->>RS: releaseReservedFunds({ accountId, amount })
        PS->>DB: INSERT INTO audit_logs ('HOLD_CANCELED')
        PC-->>UI: 200 OK { status: "FAILED" }
    end
```

---

### 2.6 Workflow 6: Crash Resilience, Two-Phase Recovery & Auto-Rollback
**Objective:** Guarantee data consistency when a server crashes after debiting the payer but before crediting the payee.

```mermaid
sequenceDiagram
    autonumber
    participant PS as PaymentService
    participant Rec as recoverStuckApproved() [5s Cron]
    participant DB as PostgreSQL
    participant R as Upstash Redis

    Note over PS,DB: Execution Attempt 1: Server Crashes AFTER_DEBIT
    PS->>PS: processTransaction({ txnId, asRecovery: false })
    PS->>R: acquireLock("user_lock:<sender_id>")
    PS->>DB: Phase 1: Debit sender balance & mark failure_reason = 'SENDER_DEBITED'
    PS->>DB: COMMIT Phase 1
    Note right of PS: Simulated Server Crash (AFTER_DEBIT / Network Drop)
    PS-xPS: Process Dies / Connection Lost
    Note over DB: Transaction remains in status: 'APPROVED', failure_reason: 'SENDER_DEBITED'

    Note over Rec,DB: Recovery Worker Sweeps (Every 5 seconds)
    Rec->>DB: SELECT id, retries FROM transactions WHERE status = 'APPROVED' AND updated_at < now() - interval '5 seconds'
    Rec->>Rec: Finds stuck txnId (retries: 0)
    
    alt Recovery Attempt 1 (retries < 3)
        Rec->>PS: processTransaction({ txnId, asRecovery: true })
        PS->>R: acquireLock("user_lock:<sender_id>")
        PS->>PS: Inspects s.failure_reason == 'SENDER_DEBITED'
        Note over PS,DB: SKIPS Phase 1 Debit! (Payer already debited)
        PS->>DB: Phase 2: Credit receiver balance & UPDATE status = 'RECOVERED'
        PS->>DB: COMMIT Phase 2
        PS->>DB: INSERT INTO audit_logs ('PAYMENT_RECOVERED')
        PS->>R: releaseLock("user_lock:<sender_id>")
        Note over Rec,DB: Successfully Recovered! No Money Lost.
    else Recovery Fails 3 Consecutive Times (retries >= 3)
        Rec->>Rec: Retries count >= 3 reached
        Rec->>PS: rollbackTransaction({ txnId })
        PS->>DB: BEGIN TRANSACTION
        PS->>DB: SELECT status FROM transactions WHERE id = txnId FOR UPDATE
        PS->>DB: Inspects s.failure_reason == 'SENDER_DEBITED'
        PS->>DB: UPDATE accounts SET balance = balance + amount WHERE id = sender_acc (Full Refund!)
        PS->>DB: UPDATE transactions SET status = 'FAILED', failure_reason = 'ROLLBACK_AFTER_MAX_RETRIES'
        PS->>DB: COMMIT TRANSACTION
        PS->>PS: releaseReservedFunds(...)
        PS->>DB: INSERT INTO audit_logs ('PAYMENT_ROLLED_BACK')
        PS->>DB: INSERT INTO notifications ('PAYMENT_REFUNDED', '₹amount refunded to your account')
        Note over Rec,DB: Transaction Safely Reconciled. Funds Restored.
    end
```

---

### 2.7 Workflow 7: Secure Balance Inquiry
**Objective:** Enforce zero-knowledge balance security where account balances are only exposed after active cryptographic PIN verification.

```mermaid
sequenceDiagram
    autonumber
    actor User as User
    participant UI as CheckBalance Component (Frontend)
    participant AC as AccountController
    participant DB as PostgreSQL

    User->>UI: Enters 4-digit PIN in CheckBalance widget
    UI->>AC: GET /account/balance?pin=1234 (Headers: Bearer <JWT>)
    AC->>AC: authMiddleware validates token -> req.user.userId
    AC->>DB: SELECT pin_hash FROM users WHERE id = req.user.userId
    AC->>AC: bcrypt.compare(pin, pin_hash)
    alt PIN Matches
        AC->>DB: SELECT v.vpa, a.balance FROM vpas v JOIN accounts a ON a.id = v.account_id WHERE v.user_id = $1
        DB-->>AC: { vpa: "aarav@apex", balance: 197500.00 }
        AC-->>UI: 200 OK { success: true, vpa, balance }
        UI-->>User: Displays Animated Emerald Balance Card
    else PIN Mismatch
        AC-->>UI: 401 Unauthorized { success: false, error: "INVALID_PIN" }
        UI-->>User: Displays Error: "Incorrect Secure PIN"
    end
```
