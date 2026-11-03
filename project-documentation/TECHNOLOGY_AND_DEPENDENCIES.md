# ApexPay - Master Technology & Dependency Inventory

This document provides a comprehensive inventory of all technologies, runtimes, frameworks, external libraries, and architectural patterns utilized across the **ApexPay** repository.

---

## 1. Architectural Tier Summary

```
┌────────────────────────────────────────────────────────────────────────┐
│                        1. Frontend Client Tier                         │
│       Next.js 16.2 (Pages Router) | React 19.2 | Tailwind CSS 4.2      │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │ HTTP / JSON (REST)
                                    ▼
┌────────────────────────────────────────────────────────────────────────┐
│                        2. Core Banking Backend                         │
│           Node.js 22 | Express 5.2 | TypeScript 6 | Zod 4.3           │
└──────────────┬────────────────────┬────────────────────┬───────────────┘
               │                    │                    │
               ▼                    ▼                    ▼
┌──────────────────────┐ ┌──────────────────────┐ ┌──────────────────────┐
│  3. Relational DB    │ │  4. Cache & Locks    │ │  5. ML Risk Service  │
│  PostgreSQL (Supabase)│ │  Upstash Redis REST  │ │  FastAPI | Scikit-   │
│  pgcrypto & Triggers │ │  Lua Scripting       │ │  Learn | GraphEngine │
└──────────────────────┘ └──────────────────────┘ └──────────────────────┘
```

---

## 2. Frontend Client Tier (`frontend/package.json`)

| Package / Technology | Version | Architectural Responsibility | Key Integration Points & File References |
|---|---|---|---|
| **Next.js** | `16.2.4` | React application framework utilizing the Pages Router architecture (`pages/_app.tsx`, `pages/index.tsx`, `pages/bankPage.tsx`). | Client-side routing, server rendering, bundling, and hot-module replacement (`frontend/pages/`). |
| **React** | `19.2.4` | Declarative UI component engine. | State hooks (`useState`, `useEffect`, `useMemo`), component composition, and event handling (`frontend/pages/index.tsx`, `frontend/components/CheckBalance.tsx`). |
| **React-DOM** | `19.2.4` | DOM rendering layer for React. | Mounts React virtual DOM into browser runtime (`frontend/pages/_app.tsx`). |
| **Tailwind CSS** | `4.2.4` | Modern utility-first CSS styling engine. | Compiled via `@tailwindcss/postcss`. Houses modern design system with gradients, responsive flexbox/grids, and fluid animations (`frontend/styles/globals.css`). |
| **@tailwindcss/postcss**| `^4` | PostCSS plugin for Tailwind v4 engine. | Configured in `frontend/postcss.config.js` and `frontend/postcss.config.mjs`. |
| **TypeScript** | `^5` | Static type checking and compiler. | Strict type interfaces for API requests/responses (`frontend/services/backend.ts`, `frontend/tsconfig.json`). |
| **@supabase/ssr** | `^0.10.2` | Supabase Server-Side Auth & Session utility. | Provided in package manifest for Supabase session hydration. |
| **@supabase/supabase-js**| `^2.104.0` | Official Supabase JavaScript client SDK. | Manifest dependency for direct database or auth access if required. |
| **ESLint & Config Next** | `^9` / `16.2.4` | Linter and Next.js rule extensions. | Static code quality and rule governance (`frontend/eslint.config.mjs`). |

---

## 3. Core Backend Tier (`backend/package.json`)

| Package / Technology | Version | Architectural Responsibility | Key Integration Points & File References |
|---|---|---|---|
| **Node.js** | `v22.22.2` | Asynchronous JavaScript/TypeScript server runtime. | Base runtime executing the Express application and scheduled workers (`backend/index.ts`). |
| **Express** | `^5.2.1` | Next-generation Node.js HTTP web application framework. | Routing, middleware pipeline, JSON parsing, error handling (`backend/index.ts`, `backend/routes/`). Express 5 natively handles rejected promises in async middleware. |
| **TypeScript** | `^6.0.3` | Type system and transpiler. | Enforces strict compile-time types across models, services, and controllers (`backend/tsconfig.json`). |
| **ts-node-dev** | `^2.0.0` | TypeScript development server with auto-restart on file save. | Development script: `npm run dev` (`backend/package.json`). |
| **Zod** | `^4.3.6` | TypeScript-first runtime schema validation library. | Strict input sanitation on all endpoints and environment loading (`backend/config/env.ts`, `backend/controllers/*`). Rejects malformed payloads with structured `VALIDATION_ERROR`. |
| **pg (node-postgres)** | `^8.20.0` | Non-blocking PostgreSQL database driver and connection pool. | Manages database client pool (max 10), SQL transaction blocks (`BEGIN ... COMMIT / ROLLBACK`), row locks (`FOR UPDATE`), and SSL negotiation (`backend/utils/db.ts`). |
| **@upstash/redis** | `^1.37.0` | Serverless HTTP/REST-based Redis client. | Stateless Redis interaction over HTTP; avoids persistent TCP connection exhaustion. Powers OTP storage, user locks, fund reservation, and chaos injection (`backend/utils/redis.ts`). |
| **bcrypt** | `^5.1.1` | Cryptographic password hashing function using Blowfish cipher. | Hashes user PINs during registration (10 salt rounds) and securely compares PINs during login, payment initiation, and balance inquiry (`backend/services/authService.ts`, `backend/controllers/accountController.ts`). |
| **jsonwebtoken** | `^9.0.2` | JSON Web Token (JWT) implementation (HMAC SHA-256). | Issues cryptographically signed stateless session tokens (24-hour expiry) upon login; verified in incoming requests via `authMiddleware` (`backend/middleware/authMiddleware.ts`, `backend/services/authService.ts`). |
| **helmet** | `^8.1.0` | HTTP security headers middleware. | Sets secure HTTP headers (`X-Content-Type-Options`, `X-Frame-Options`, `Content-Security-Policy`) to safeguard against XSS and clickjacking (`backend/index.ts`). |
| **cors** | `^2.8.6` | Cross-Origin Resource Sharing middleware. | Configured on Express root to allow browser clients on `localhost:3000` to interact with API endpoints (`backend/index.ts`). |
| **dotenv** | `^17.4.2` | Environment configuration loader. | Loads variables from `.env` files into `process.env` at startup (`backend/config/env.ts`). |
| **nanoid** | `^5.1.9` | Secure URL-friendly unique string ID generator. | Generates compact random identifiers for tracing and lock values. |

---

## 4. Machine Learning & Fraud Intelligence Tier (`ml-service/requirements.txt`)

| Package / Technology | Version | Architectural Responsibility | Key Integration Points & File References |
|---|---|---|---|
| **Python** | `3.12.4` | High-performance scientific runtime. | Microservice execution environment (`ml-service/api/main.py`). |
| **FastAPI** | `0.116.1` | High-throughput asynchronous Python web framework based on Starlette and Pydantic. | Synchronous pre-settlement REST API exposing `/health`, `/metrics`, and `/predict-risk` on port 8000 (`ml-service/api/main.py`). |
| **Uvicorn** | `0.35.0` | Lightning-fast ASGI web server implementation using `uvloop` and `httptools`. | Hosts the FastAPI application instance (`ml-service/api/main.py`). |
| **Pydantic** | `2.11.4` | High-performance data validation and serialization based on Rust core. | Validates typed request and response schemas (`RiskRequest`, `RiskResponse`), enforcing numeric bounds ($amount > 0$, probabilities in $[0, 1]$) with zero-cost validation (`ml-service/api/main.py`). |
| **Scikit-Learn** | `1.9.1` (Trained) / `1.5.1` (Runtime)| Machine learning algorithms, probability calibration, and evaluation metrics. | Implements 150-tree `RandomForestClassifier` and Platt probability calibration via `CalibratedClassifierCV(method='sigmoid')` (`ml-service/training/train.py`, `ml-service/api/predict.py`). |
| **Joblib** | `1.4.2` | Fast persistence for NumPy and SciPy arrays. | Serializes and unpickles the trained calibrated model bundle `risk_model.joblib` (`ml-service/training/train.py`, `ml-service/api/predict.py`). |
| **NumPy** | `2.2.6` | Numerical computing and array manipulation. | Feature vector formatting, z-score transformations, tanh soft clipping, percentile evaluation (`ml-service/utils/features.py`, `ml-service/training/train.py`). |
| **Pandas** | `2.2.3` | High-performance tabular data structures and analysis. | Ingestion, feature filtering, and train/test splitting on `test_data_large.csv` (`ml-service/training/train.py`). |
| **SQLite3 (Native)** | Built-in | Embedded relational database engine. | Configured with Write-Ahead Logging (`PRAGMA journal_mode = WAL`) and `PRAGMA synchronous = NORMAL` to power persistent graph store `data/graph_store.db` with sub-millisecond multi-worker transactions (`ml-service/utils/graph_engine.py`). |

---

## 5. Relational Database Tier (PostgreSQL)

| Component | Version / Engine | Architectural Responsibility | Key Implementation Reference |
|---|---|---|---|
| **PostgreSQL** | `14+` / Supabase Cloud | Primary transactional ACID data store. | Enforces relational integrity across 10 tables, check constraints, unique indexes, and two-phase row locks (`ApexPay/database/schema.sql`). |
| **pgcrypto** | PostgreSQL Extension | Cryptographically secure random UUID generation. | Enabled via `create extension if not exists pgcrypto;` to power default UUID primary keys (`gen_random_uuid()`). |
| **PL/pgSQL Triggers** | PostgreSQL Procedural Lang | Immutable regulatory audit logging. | Function `prevent_audit_log_mutation()` attached to `audit_logs` and `risk_scores` before `UPDATE` or `DELETE` to raise runtime exceptions. |

---

## 6. Distributed Caching & Locking Tier (Upstash Redis)

| Capability | Engine | Architectural Responsibility | Key Implementation Reference |
|---|---|---|---|
| **REST-Based Redis** | Upstash Cloud Redis | High-speed ephemeral data store. | Avoids persistent connection limits. Stores SMS OTPs (300s TTL), temporary login clearance flags (300s TTL), user behavioral anchors (30-day TTL), and in-flight balance reservations (`backend/utils/redis.ts`, `backend/services/otpService.ts`). |
| **Distributed Locks (Redlock)**| Redis `SET NX PX` | Mutual exclusion concurrency locking. | Atomically acquires locks (`user_lock:<userId>`, `reserve_lock:<accountId>`) to prevent concurrent race conditions during transactions (`backend/utils/redis.ts`). |
| **Atomic Lua Scripting**| Redis Embedded Lua Engine | Safe atomic unlock without race conditions. | Compares key value against token before deleting: `if redis.call('get', KEYS[1]) == ARGV[1] then return redis.call('del', KEYS[1]) else return 0 end` (`backend/utils/redis.ts`). |

---

## 7. Key Architectural Patterns Visible in Code

1. **Synchronous Pre-Settlement Gating:**
   Unlike traditional batch fraud systems that detect theft after money has left the account, ApexPay synchronously gates settlement. The ledger transfer never executes until the ML microservice issues an authoritative risk clearance.
2. **Two-Phase Distributed Settlement with Progress Checkpointing:**
   Transactions split the movement of money into discrete phases: Phase 1 (Debit Sender) updates `failure_reason = 'SENDER_DEBITED'`. Phase 2 (Credit Receiver) updates `status = 'COMPLETED'`. If a server crashes midway, the recovery worker detects the checkpoint and resumes Phase 2 without double-debiting.
3. **Optimistic In-Flight Fund Reservation:**
   When a transaction is held for OTP verification (`status = 'HOLD'`), the required amount is reserved in Redis (`reserve:account:${accountId}`). If the user attempts another transfer, `reserveFunds` verifies that `current_reserved + new_amount <= balanceLimit`, preventing double-spending while on hold.
4. **Three-Layer Hybrid AI Architecture:**
   Combines rule heuristics (Layer 1) for deterministic regulatory boundaries, calibrated machine learning (Layer 2) for complex non-linear probability estimation, and directed multigraph intelligence (Layer 3) for laundering rings and mule network topology.
5. **Decoupled Identity Context:**
   The ML feature extractor strictly decouples sender attributes (`is_first_transaction`, `account_age_hours`) from receiver attributes (`receiver_is_first_transaction`), ensuring cold-start rules on a new sender never accidentally penalize a legitimate receiver.
6. **Stateless Security with In-Memory Caching:**
   Session state is decentralized via JWT tokens, while high-velocity fraud topology resides in an in-memory directed multigraph backed by SQLite WAL persistence.
