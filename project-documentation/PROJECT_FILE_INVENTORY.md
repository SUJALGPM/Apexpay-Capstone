# ApexPay - Master Project File Inventory

This inventory documents every file across all tiers of the **ApexPay** repository (`https://github.com/Shaikh-Aasim/ApexPay`). Each file is classified by its physical location, architectural role, exported symbols, functional responsibilities, and dependency links.

---

## 1. Root Directory & Orchestration Scripts

| File Path | Component / Role | Major Functions / Symbols | Purpose & Relationships |
|---|---|---|---|
| `ApexPay/Run.txt` | Operational Guide | Documentation | Outlines startup sequence (ML service -> backend -> frontend) and summarizes multi-step auth and central bank simulation workflows. |
| `ApexPay/.gitignore` | Git Configuration | Ignore rules | Configures Git to ignore `node_modules`, `.env`, build artifacts (`dist/`, `.next/`), virtual environments (`venv/`), and runtime logs. |
| `ApexPay/add_test_user.ts` | Database Seeding Script | `addTestUser()` | Standalone TypeScript script utilizing `backend/utils/db.ts` to insert test user "Rahul Tester" (`rahul@apex`) with ₹5,000,000 balance into `users`, `accounts`, and `vpas`. |
| `ApexPay/test_balance.js` | API Testing Script | `test()` | Node.js fetch script issuing an HTTP GET request to `http://localhost:4000/account/balance?vpa=aarav@apex` to verify balance retrieval. |

---

## 2. Database Tier (`database/`)

| File Path | Component / Role | Major Schema Entities | Purpose & Relationships |
|---|---|---|---|
| `ApexPay/database/schema.sql` | PostgreSQL DDL & Seed Script | Enums: `transaction_status`, `fraud_type`<br>Tables: `banks`, `users`, `bank_accounts`, `accounts`, `vpas`, `receiver_profiles`, `transactions`, `risk_scores`, `audit_logs`, `notifications`<br>Triggers: `prevent_audit_log_mutation()` | The single source of relational truth. Defines strict foreign keys, check constraints (`balance >= 0`, `amount > 0`), indexing on hot transaction paths, immutability triggers for audit compliance, and baseline seed data for Apex Bank, Neo Bank, and test user accounts. |

---

## 3. Backend Tier (`backend/`)

### 3.1 Configuration & Entry Point
| File Path | Component / Role | Major Functions / Symbols | Purpose & Relationships |
|---|---|---|---|
| `ApexPay/backend/config/env.ts` | Environment Validator | `env`, Zod schema | Parses and strictly validates environment variables (`PORT`, `DATABASE_URL`, `ML_SERVICE_URL`, `UPSTASH_REDIS_REST_URL`, `UPSTASH_REDIS_REST_TOKEN`, `HOLD_TTL_SECONDS`, `LOCK_TTL_MS`, `JWT_SECRET`) at boot. Fails fast if configuration is incomplete. |
| `ApexPay/backend/index.ts` | Express HTTP Server | `startServer()`, Express application instance | Initializes Express, mounts security middleware (`helmet`, `cors`), registers API routes (`/api/auth`, `/api/bank`, `/payment`, `/account`, `/notifications`, `/test`), runs automatic schema migrations on startup, and launches background sweep intervals (Hold expiry sweep and Stuck payment recovery sweep every 5 seconds). |
| `ApexPay/backend/package.json` | Node Manifest | Scripts: `dev`, `build`, `start`, `typecheck`<br>Dependencies: `express`, `@upstash/redis`, `pg`, `bcrypt`, `jsonwebtoken`, `zod`, `helmet`, `cors` | Specifies runtime dependencies, TypeScript build configuration, and development execution scripts via `ts-node-dev`. |
| `ApexPay/backend/tsconfig.json` | Compiler Config | TypeScript compiler flags | Configures TypeScript transpilation targets (`ES2022`), module resolution (`node`), strict type-checking, and output directory (`dist/`). |

### 3.2 Utilities (`backend/utils/`)
| File Path | Component / Role | Major Functions / Symbols | Purpose & Relationships |
|---|---|---|---|
| `ApexPay/backend/utils/db.ts` | Database Connection Pool | `pool` (`pg.Pool`) | Instantiates PostgreSQL connection pool with maximum 10 clients. Automatically enables SSL with `rejectUnauthorized: false` when connected to Supabase instances. |
| `ApexPay/backend/utils/redis.ts` | Upstash Redis REST Client & Lock Utilities | `redis`, `acquireLock()`, `releaseLock()`, `getJson()`, `setJson()` | Wraps `@upstash/redis` HTTP REST client. Implements atomic distributed mutual exclusion locking via Redis `SET ... NX PX` and release via Lua script comparison (`if redis.call('get', KEYS[1]) == ARGV[1] then return redis.call('del', KEYS[1]) else return 0 end`). |
| `ApexPay/backend/utils/migrate.ts` | Database Schema Migrator | `runMigrations()` | Executes programmatic migration on application boot: validates table schemas for `users` and `bank_accounts`, creates `banks` and `bank_accounts` if absent, ensures polymorphic columns (`payment_note`, IFSC code, Aadhar) exist, and seeds core banking simulation accounts. |

### 3.3 Middleware (`backend/middleware/`)
| File Path | Component / Role | Major Functions / Symbols | Purpose & Relationships |
|---|---|---|---|
| `ApexPay/backend/middleware/authMiddleware.ts` | JWT Guard | `authMiddleware()`, `AuthRequest` interface | Extracts `Bearer <token>` from HTTP `Authorization` header, validates signature with `JWT_SECRET` via `jsonwebtoken`, and injects decoded identity payload (`userId`, `mobile`, `name`) into `req.user`. Returns `401 UNAUTHORIZED` on failure. |

### 3.4 Controllers (`backend/controllers/`)
| File Path | Component / Role | Major Functions / Symbols | Purpose & Relationships |
|---|---|---|---|
| `ApexPay/backend/controllers/authController.ts` | Authentication Controller | `registerInitiate()`, `registerVerifyOtp()`, `registerSetPin()`, `loginInitiate()`, `loginVerifyOtp()`, `loginVerifyPin()`, `handleError()` | Validates incoming payloads via Zod schemas (`registerInitiateSchema`, `verifyOtpSchema`, `setPinSchema`, etc.) and delegates registration and multi-factor login workflows to `authService.ts`. |
| `ApexPay/backend/controllers/bankController.ts` | Core Banking Controller | `createAccount()`, `getBanks()` | Exposes simulated central bank registry endpoints. Validates input schemas (`CreateAccountSchema`) and routes creation to `bankService.ts`. |
| `ApexPay/backend/controllers/paymentController.ts` | Payment Gateway Controller | `initiate()`, `confirm()`, `cancel()`, `status()`, `history()` | Validates payment initiation parameters (VPA or mobile destination, positive amount, device, location, PIN) and orchestrates payment lifecycle actions via `paymentService.ts`. |
| `ApexPay/backend/controllers/accountController.ts` | Account & Balance Controller | `resolveReceiver()`, `getBalance()` | Resolves payee full name and VPA by query parameter (`vpa` or `mobile`). Retrieves account balance for authenticated user with mandatory bcrypt PIN verification. |
| `ApexPay/backend/controllers/notificationController.ts` | Notification Controller | `list()` | Fetches in-app notification logs for a specific user ID via `notificationService.ts`. |
| `ApexPay/backend/controllers/testController.ts` | Chaos & Testing Controller | `delay()`, `crash()`, `simulateSettlementFailure()` | Injects controlled latency or simulated server crashes (`BEFORE_DEBIT`, `AFTER_DEBIT`, `BEFORE_CREDIT`) into Redis state via `chaosService.ts` for resilience testing. |

### 3.5 Routes (`backend/routes/`)
| File Path | Component / Role | Registered Routes | Purpose & Relationships |
|---|---|---|---|
| `ApexPay/backend/routes/authRoutes.ts` | Router | `POST /register/initiate`<br>`POST /register/verify-otp`<br>`POST /register/set-pin`<br>`POST /login/initiate`<br>`POST /login/verify-otp`<br>`POST /login/verify-pin` | Maps authentication endpoints to `authController.ts`. Mounted at `/api/auth` in `index.ts`. |
| `ApexPay/backend/routes/bankRoutes.ts` | Router | `POST /accounts`<br>`GET /list` | Maps simulated core banking endpoints to `bankController.ts`. Mounted at `/api/bank` in `index.ts`. |
| `ApexPay/backend/routes/paymentRoutes.ts` | Router | `POST /initiate` (Protected)<br>`POST /confirm` (Protected)<br>`POST /cancel` (Protected)<br>`GET /history` (Protected)<br>`GET /status` (Public) | Maps payment engine endpoints to `paymentController.ts`. Enforces `authMiddleware` on state-modifying actions. Mounted at `/payment` in `index.ts`. |
| `ApexPay/backend/routes/accountRoutes.ts` | Router | `GET /balance` (Protected)<br>`GET /resolve` (Protected) | Maps user account inquiry endpoints to `accountController.ts`. Mounted at `/account` in `index.ts`. |
| `ApexPay/backend/routes/notificationRoutes.ts` | Router | `GET /` | Maps notification query endpoint to `notificationController.ts`. Mounted at `/notifications` in `index.ts`. |
| `ApexPay/backend/routes/testRoutes.ts` | Router | `POST /delay`<br>`POST /crash`<br>`POST /settlement-failure` | Maps testing and chaos engineering endpoints to `testController.ts`. Mounted at `/test` in `index.ts`. |

### 3.6 Services (`backend/services/`)
| File Path | Component / Role | Major Functions / Symbols | Purpose & Relationships |
|---|---|---|---|
| `ApexPay/backend/services/authService.ts` | Authentication Business Logic | `initiateRegistration()`, `verifyRegistrationOtp()`, `setPinAndCompleteRegistration()`, `initiateLogin()`, `verifyLoginOtp()`, `verifyPinAndLogin()` | Coordinates registration (verifying existence in simulated bank accounts, issuing OTP, hashing PIN, creating user, account, and default VPA) and login (2FA OTP + bcrypt PIN verification, JWT issuance). |
| `ApexPay/backend/services/otpService.ts` | Ephemeral OTP Manager | `issueOtp()`, `issueMobileOtp()`, `verifyMobileOtp()`, `checkOtpVerified()`, `clearOtpVerified()`, `verifyOtp()`, `hashOtp()` | Generates 6-digit random numeric tokens, hashes them with SHA-256 before storing in Redis with strict TTLs, and enforces maximum verification retry limits (3 to 5 attempts) to prevent brute force attacks. |
| `ApexPay/backend/services/bankService.ts` | Simulated Core Banking Logic | `findBankAccountsByMobile()`, `getBankAccountById()`, `createBankAccount()`, `getAllBanks()` | Interacts with `bank_accounts` and `banks` tables. Acts as the simulated central banking authority (CBS/NPCI registry) during user onboarding. |
| `ApexPay/backend/services/paymentService.ts` | Core Payment & Settlement Engine | `initiatePayment()`, `confirmPayment()`, `cancelPayment()`, `getPaymentStatus()`, `getTransactionHistory()`, `processTransaction()`, `rollbackTransaction()`, `expireHolds()`, `recoverStuckApproved()` | Core engine: verifies sender PIN, resolves VPAs/mobiles, computes behavioral features, queries ML microservice, enforces 3-tier risk decisions (`LOW`, `MEDIUM`, `HIGH`), manages two-phase settlement (`SENDER_DEBITED` -> credit), Redis concurrency locking, hold expirations, and background recovery sweeps. |
| `ApexPay/backend/services/reserveService.ts` | In-Flight Balance Reservation | `reserveFunds()`, `releaseReservedFunds()` | Implements Redis-based in-flight fund locking (`reserve:account:${accountId}`) protected by distributed locks (`reserve_lock:${accountId}`) to prevent double-spending during transaction `HOLD` windows. |
| `ApexPay/backend/services/behaviorService.ts` | Behavioral Analytics Engine | `getTxnCount24h()`, `getUserAmountStats30d()`, `computeDeviceChange()`, `computeLocationAnomaly()`, `updateBehaviorAnchors()` | Extracts historical transaction frequencies and 30-day mean/standard deviation from PostgreSQL; evaluates device and location shifts against 30-day anchors stored in Redis. |
| `ApexPay/backend/services/receiverRiskService.ts` | Receiver Reputation Service | `getReceiverRisk()` | Queries `receiver_profiles` to fetch payee suspicion scores and blacklist flags for ML risk payload compilation. |
| `ApexPay/backend/services/chaosService.ts` | Fault Injection Service | `setDelayMs()`, `getDelayMs()`, `setCrashPoint()`, `getCrashPoint()` | Manages simulated network latency and crash injection points (`BEFORE_DEBIT`, `AFTER_DEBIT`, `BEFORE_CREDIT`) via Redis keys `chaos:delay_ms` and `chaos:crash_point`. |
| `ApexPay/backend/services/mlService.ts` | ML Service Client | `predictRisk()`, `fraudTypeSchema`, `decisionSchema`, `riskRequestSchema` | Issues synchronous HTTP POST requests to the Python FastAPI microservice (`/predict-risk`) and validates the response using Zod schemas. |
| `ApexPay/backend/services/notificationService.ts` | In-App Notification Service | `notifyUser()`, `getUserNotifications()` | Inserts notification records (`PAYMENT_DEBITED`, `PAYMENT_RECEIVED`, `HOLD_TRIGGERED`, `RISK_REJECTED`) into PostgreSQL `notifications` table. |
| `ApexPay/backend/services/auditService.ts` | Immutable Audit Logger | `auditLog()` | Writes immutable audit trail entries (`PAYMENT_INITIATED`, `SENDER_DEBIT_SUCCESS`, `PAYMENT_COMPLETED`, etc.) into `audit_logs` protected by database mutation triggers. |
| `ApexPay/backend/services/accountService.ts` | Account Balance Service | `getBalanceByVPA()` | Resolves account balance from PostgreSQL `accounts` table linked via `vpas`. |

### 3.7 Database Seeding Helpers (`backend/`)
| File Path | Component / Role | Major Functions / Symbols | Purpose & Relationships |
|---|---|---|---|
| `ApexPay/backend/add_test_user.js` | Legacy CommonJS Seeder | `addTestUser()` | Standalone seed script creating user "Rahul Tester" using hardcoded Supabase pool connection. |
| `ApexPay/backend/add_test_user_v2.js` | CommonJS Seeder | `addTestUser()` | Updated standalone script with `.env` loading from backend directory. |

---

## 4. Frontend Tier (`frontend/`)

### 4.1 Framework & Architecture Configuration
| File Path | Component / Role | Purpose & Relationships |
|---|---|---|
| `ApexPay/frontend/package.json` | Dependencies & Scripts | Defines Next.js `16.2.4`, React `19.2.4`, Tailwind CSS `4.2.4`, `@tailwindcss/postcss`, TypeScript `5`, and Supabase SSR client packages. Scripts: `dev`, `build`, `start`, `lint`, `typecheck`. |
| `ApexPay/frontend/next.config.ts` | Next.js Configuration | Next.js runtime configuration settings. |
| `ApexPay/frontend/tsconfig.json` | Compiler Configuration | TypeScript configuration for React 19 / Next.js Pages Router compilation. |
| `ApexPay/frontend/postcss.config.js` | CSS Processor | Configures PostCSS with `@tailwindcss/postcss`. |
| `ApexPay/frontend/postcss.config.mjs` | ES Module PostCSS | Modern PostCSS ESM configuration. |
| `ApexPay/frontend/eslint.config.mjs` | Linting Configuration | ESLint flat configuration extending `eslint-config-next`. |
| `ApexPay/frontend/env.download` | Environment Example | Template defining `NEXT_PUBLIC_BACKEND_URL`, `NEXT_PUBLIC_SUPABASE_URL`, and publishable API keys. |
| `ApexPay/frontend/styles/globals.css` | Styling Entry Point | Imports `@import "tailwindcss";` and defines base font families (Geist Sans, Geist Mono). |

### 4.2 Application Pages (`frontend/pages/`)
| File Path | Component / Role | Major Component / State | Purpose & Relationships |
|---|---|---|---|
| `ApexPay/frontend/pages/_app.tsx` | Next.js App Wrapper | `App({ Component, pageProps })` | Global root component that wraps all pages and imports `globals.css`. |
| `ApexPay/frontend/pages/_document.tsx` | Next.js Document Template | `Document()` | Custom HTML document structure setting language, head meta tags, and body scripts. |
| `ApexPay/frontend/pages/index.tsx` | Primary Dashboard & Single Page Application | `Home()` (1,353 LOC)<br>Tabs: VPA Payment, Mobile Payment, Check Balance, Transaction History<br>Modals: Authentication (Login/Register 2FA), Step-Up OTP Verification, Transaction Details Receipt, Chaos Injection | The primary user interface of ApexPay. Houses the complete client-side state machine: user session management via `localStorage`, payment multi-step flow (`INPUT` -> `CONFIRM_RECEIVER` -> `ENTER_PIN` -> `FINAL_CONFIRM`), 45-second live countdown timer for `HOLD` states, transaction polling (`/payment/status`), filterable transaction history, and live notifications. |
| `ApexPay/frontend/pages/bankPage.tsx` | Core Banking Simulator Interface | `BankPage()` | Independent management portal allowing developers/users to simulate the Central Bank Registry: creates dummy accounts in `bank_accounts` with Aadhar, mobile, IFSC, and balance before registering on ApexPay. |
| `ApexPay/frontend/pages/api/hello.ts` | Next.js API Route | `handler()` | Boilerplate Next.js serverless route returning `{ name: "John Doe" }`. |

### 4.3 UI Components (`frontend/components/`)
| File Path | Component / Role | Major Functions / Symbols | Purpose & Relationships |
|---|---|---|---|
| `ApexPay/frontend/components/CheckBalance.tsx` | Secure Balance Component | `CheckBalance()` | Dedicated secure balance inquiry component requiring 4-to-6 digit PIN entry before invoking `getBalance(undefined, pin)`. Renders interactive balance cards with real-time sync indicators. |

### 4.4 API Communication Layer (`frontend/services/`)
| File Path | Component / Role | Major Functions / Exported Types | Purpose & Relationships |
|---|---|---|---|
| `ApexPay/frontend/services/backend.ts` | HTTP Client & API Gateway Connector | Functions: `loginInitiate()`, `loginVerifyOtp()`, `loginVerifyPin()`, `registerInitiate()`, `registerVerifyOtp()`, `registerSetPin()`, `createBankRegistryAccount()`, `getBankList()`, `initiatePayment()`, `confirmPayment()`, `cancelPayment()`, `getBalance()`, `resolveReceiver()`, `getPaymentStatus()`, `getNotifications()`, `getTransactionHistory()`, `simulateSettlementFailure()`<br>Types: `InitiateRequest`, `InitiateResponse`, `ConfirmRequest`, `TxStatusResponse`, `TransactionHistoryItem`, `AuthUser`, `LoginResponse`, `BalanceResponse` | Centralized typed HTTP service. Reads `localStorage` for `apexpay_token` and injects `Authorization: Bearer <token>` into outgoing requests. Handles JSON parsing, standardizes backend error propagation, and formats query parameters. |

---

## 5. Machine Learning & Fraud Intelligence Tier (`ml-service/`)

### 5.1 Service Entry & Orchestration (`ml-service/api/`)
| File Path | Component / Role | Major Functions / Classes | Purpose & Relationships |
|---|---|---|---|
| `ApexPay/ml-service/api/main.py` | FastAPI Application Instance | `app`, `RiskRequest` (Pydantic v2), `RiskResponse`, `health()`, `metrics()`, `predict()` | Exposes REST endpoints on port 8000: `GET /health`, `GET /metrics`, and `POST /predict-risk`. Formats Pydantic request models and passes payload to `RiskPredictor`. |
| `ApexPay/ml-service/api/predict.py` | Hybrid Predictor Orchestrator | `RiskPredictor`, `_load_model()`, `predict()`, `update_graph()` | Orchestrates 3-layer inference pipeline: graph state ingestion, heuristic rule evaluation (Layer 1), calibrated Random Forest inference (Layer 2), smart force overrides for critical account takeover/extreme deviation, and graph intelligence fusion (Layer 3). |
| `ApexPay/ml-service/api/integration.py` | Multi-Signal Score Fusion & Decision Engine | `combine_scores()`, `get_final_decision()`, `integrate_graph_intelligence()` | Blends ML probability, heuristic rules, and graph scores via adaptive weighting. Evaluates decoupled sender and receiver contexts. Enforces multi-signal confirmation matrix and authoritative decision gating (`LOW`, `MEDIUM`, `HIGH`). Clamps scores strictly within $[0.0, 1.0]$. |

### 5.2 Algorithmic Engines & Feature Extractors (`ml-service/utils/`)
| File Path | Component / Role | Major Functions / Symbols | Purpose & Relationships |
|---|---|---|---|
| `ApexPay/ml-service/utils/features.py` | Feature Definitions & Preprocessor | `FEATURES` (17 feature array), `calculate_zscore()`, `get_feature_vector()` | Defines the canonical 17 features used for ML inference. Implements balanced z-score calculation with log transforms (`log1p`), soft tanh clipping, and legitimacy discounts. |
| `ApexPay/ml-service/utils/risk_rules.py` | Layer 1: Heuristic Rule Engine | `HeuristicResult`, `clamp01()`, `heuristic_risk()` | Evaluates deterministic risk signals: tiered transfer amounts, baseline z-scores, 24h velocities, device and location anomalies, receiver reputation, cold-start limits, periodic pattern exemptions, and critical override blocks. |
| `ApexPay/ml-service/utils/graph_engine.py` | Layer 3: Directed Multigraph & SQLite WAL Store | `is_valid_identity()`, `TransactionRecord`, `EdgeState`, `GraphEngine` | Maintains in-memory directed multigraph with sub-millisecond RAM lookups, backed by persistent SQLite database in Write-Ahead Logging (`WAL`) mode. Enforces transaction idempotency and super-node filtering (`unknown_sender`). |
| `ApexPay/ml-service/utils/graph_features.py` | Graph Topology Feature Extractor | `compute_graph_features()` | Calculates in-degree, out-degree, discrete time-window velocities (1h, 24h, 7d, lifetime), unique sender counts, and half-life temporal risk decay (14-day half-life). |
| `ApexPay/ml-service/utils/graph_rules.py` | Structural Fraud Detectors | `detect_mule_pattern()`, `detect_fraud_ring()` | Implements bounded depth-first search (DFS) for cycle detection (distinguishing 3-node social splitting from $\ge 5$-node circular laundering rings) and fan-in funnel aggregation for money mule detection. |
| `ApexPay/ml-service/utils/graph_score.py` | Graph Score Synthesizer | `calculate_graph_score()` | Fuses in-degree, out-degree, cycle, and mule risks into a unified graph score $S_{\text{graph}} \in [0.0, 1.0]$ with cold-start damping. |
| `ApexPay/ml-service/utils/helpers.py` | String & Math Helpers | Sanitization and validation helpers | Utility functions for data sanitization and mathematical bounding. |

### 5.3 Training Pipeline & Serialized Models (`ml-service/training/`, `ml-service/model/`)
| File Path | Component / Role | Major Functions / Artifacts | Purpose & Relationships |
|---|---|---|---|
| `ApexPay/ml-service/training/train.py` | Training Pipeline | `evaluate_thresholds()`, `analyze_false_positives()`, `main()` | Loads `test_data_large.csv`, executes stratified 80/20 train/test split, performs 5-fold cross-validation on base Random Forest (150 trees, max depth 10), fits `CalibratedClassifierCV` (sigmoid Platt calibration), evaluates precision/recall/F1, and serializes `risk_model.joblib` and `metrics.json`. |
| `ApexPay/ml-service/model/risk_model.joblib` | Serialized Model Bundle | Joblib serialized dict | Bundles calibrated Random Forest classifier, feature names list, and model version (`2.2.0`). |
| `ApexPay/ml-service/model/metrics.json` | Model Performance Report | Serialized JSON | Holds verified model metrics: accuracy (98.53%), precision (96.11%), recall (98.75%), F1-score (97.41%), version 2.2.0. |

### 5.4 Test Suites & Generators (`ml-service/tests/`)
| File Path | Component / Role | Major Functions / Symbols | Purpose & Relationships |
|---|---|---|---|
| `ApexPay/ml-service/tests/test_service_comprehensive.py` | Master Regression Test Suite | `TestApexPayMLService` (13 test cases) | Automated end-to-end regression suite validating normal payments, account takeover overrides, cold-start limits, social triangles, circular laundering rings, mule fan-ins, discrete time windows, idempotency, schema completeness, and inference latency percentiles ($p50 < 300\text{ ms}$). |
| `ApexPay/ml-service/tests/test_data_generator.py` | Canonical Dataset Generator | `generate_large_scale_test_suite()` | Generates 50,000 synthetic transaction records modeling normal spending, account takeover, smurfing, and phishing with mathematical statistical distributions. Outputs `test_data_large.csv`. |
| `ApexPay/ml-service/tests/mini_test_data_generator.py` | Rapid Dataset Generator | `generate_mini_test_suite()` | Generates compact dataset (`test_data_mini.csv`) for fast local validation. |
| `ApexPay/ml-service/tests/noisy_data_generator.py` | Robustness Dataset Generator | Generator function | Injects realistic noise, missing features, and edge cases into dataset (`noisy_test_data.csv`). |
| `ApexPay/ml-service/tests/quick_test_runner.py` | Smoke Test Runner | Test runner | Runs rapid smoke tests against prediction pipeline. |
| `ApexPay/ml-service/tests/scenario_tests.py` | Scenario Validation Suite | Test scenarios | Tests specific threat scenarios (mule rings, sudden credential jumps). |
| `ApexPay/ml-service/tests/graph_test_runner.py` | Graph Engine Unit Tests | Test runner | Exercises DFS cycles, SQLite persistence, and edge decay. |
| `ApexPay/ml-service/tests/noisy_test_runner.py` | Noise Tolerance Runner | Test runner | Evaluates model robustness against noisy input data. |
| `ApexPay/ml-service/tests/metrics.py` | Metric Calculations | Metric helpers | Evaluation utility functions. |

### 5.5 Benchmarks & Technical Documentation (`ml-service/scripts/`, `ml-service/docs/`)
| File Path | Component / Role | Purpose & Relationships |
|---|---|---|
| `ApexPay/ml-service/scripts/benchmark_dataset_mapping.py` | Public Dataset Converter | Converts PaySim mobile money dataset and IEEE-CIS fraud dataset into ApexPay 17-feature schema for benchmark evaluation. |
| `ApexPay/ml-service/docs/API_REFERENCE.md` | API Reference | Complete REST specifications for `/health`, `/metrics`, and `/predict-risk`. |
| `ApexPay/ml-service/docs/ML_ARCHITECTURE.md` | Architecture Specification | Multi-layer hybrid design, component boundaries, and threat models. |
| `ApexPay/ml-service/docs/MODEL_TRAINING.md` | Training & Calibration Doc | Random Forest hyperparameters, Platt sigmoid calibration, and feature importances. |
| `ApexPay/ml-service/docs/RISK_DECISION_POLICY.md` | Risk Governance Policy | 3-tier scoring bands, multi-signal confirmation matrix, and hard security overrides. |
| `ApexPay/ml-service/docs/GRAPH_ENGINE.md` | Graph Engine Specification | Graph topology, cycle DFS, mule fan-in algorithms, and half-life decay equations. |
| `ApexPay/ml-service/docs/DATASET_GENERATION.md` | Dataset Generation Specs | Statistical distributions and synthesis rules for 50,000-sample training dataset. |
| `ApexPay/ml-service/docs/DATASET_FEATURE_MAPPING.md` | Benchmark Mapping Specs | Crosswalk mapping PaySim and IEEE-CIS attributes to ApexPay feature schema. |
| `ApexPay/ml-service/docs/ML_SERVICE_AUDIT.md` | Engineering Audit & Fixes | Audit matrix detailing legacy defects (conflation, super-nodes) and applied engineering remediations. |
| `ApexPay/ml-service/docs/TEST_REPORT.md` | Quality Assurance Report | Complete test execution report and verification benchmarks. |
