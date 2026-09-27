# 09 — Internal System Architecture (Velo application layer)

The **inside** of the Velo backend — how the application code is organised, how modules talk to each other, where data lives, how internal jobs run, and where the boundary to the outside world sits.

Complements `docs/06-cloud-architecture.md`, which is the **outside/deployment** view (AWS regions, VPCs, subnets, edge, DR, CI/CD).

## Scope split with `06-cloud-architecture.md`

| Concern | Lives in | This doc | Cloud arch (06) |
|---|---|---|---|
| Where things run (region, AZ, VPC, subnet) | Infra | — | ✓ |
| Edge, WAF, CDN, DR, cross-region | Infra | — | ✓ |
| CI/CD, testing strategy, deploy topology | DevOps | — | ✓ |
| Which application modules exist and what they own | App | ✓ | brief |
| Internal API contracts between modules | App | ✓ | — |
| Event bus topics + producers + consumers | App | ✓ | brief |
| Postgres schema ownership (who writes which tables) | App | ✓ | — |
| Scheduled jobs (interest, recon, SFTP puller, cleanups) | App | ✓ | — |
| Redis usage patterns (idempotency cache, hot-balance cache) | App | ✓ | — |
| Anti-Corruption Layer to sponsor bank | App | ✓ | brief |
| External integration points (Sponsor, Cognito, SES, ...) | Boundary | mentioned as edges | ✓ full |

**C4 mapping**: doc 06 is the **System Context** + **Container** view. Doc 09 is the **Container (application-focused)** + **Component** view.

---

## Architecture at a glance (one diagram)

```mermaid
flowchart TB
    %% ================= Edges =================
    subgraph Clients[Clients]
      CustApp["Customer App<br/>(PWA + React Native)"]
      OpsWeb["Ops Portal<br/>(internal web)"]
      Sponsor(["Sponsor Bank<br/>(external)"])
    end

    %% ================= API Layer =================
    subgraph APILayer["API Layer  (HTTPS ingress into the backend)"]
      CustAPI["Customer API Gateway<br/>REST/JSON, JWT-authenticated"]
      OpsAPI["Ops Portal API<br/>REST/JSON, Zero-Trust-only"]
      WHIn["Sponsor Webhook Receiver<br/>REST/JSON, mTLS + HMAC"]
    end

    %% ================= Domain Services =================
    subgraph Domain["Domain Services  (Velo backend — modular monolith)"]
      Onb["Onboarding<br/>Service"]
      Id["Identity<br/>Service<br/>(Cognito facade)"]
      Acc["Account<br/>Service"]
      Led["Ledger<br/>Service<br/>(sole writer to postings)"]
      Tx["Transfers<br/>Service"]
      PIn["Payments-In<br/>Service"]
      PSvc["PayID<br/>Service"]
      Int["Interest<br/>Service"]
      Recon["Reconciliation<br/>Service"]
      Notif["Notifications<br/>Service"]
      FraudStub["Fraud<br/>(stub in MVP,<br/>real engine in roadmap)"]
    end

    %% ================= ACL =================
    subgraph ACL["Anti-Corruption Layer"]
      SBA["Sponsor-Bank Adapter<br/>(only place the sponsor SDK / SFTP client lives)"]
    end

    %% ================= Data =================
    subgraph Data["Data Stores"]
      PG[("Postgres<br/>schemas per module")]
      Redis[("Redis<br/>idempotency + hot balance cache")]
      S3In[("S3 inbound-files<br/>raw DE/NPP evidence")]
      S3Kyc[("S3 KYC docs<br/>Object Lock + KMS-CMK")]
      S3Aud[("S3 audit-lake<br/>7-yr Object Lock")]
    end

    %% ================= Messaging =================
    subgraph Bus["Messaging"]
      Outbox[("Postgres outbox table<br/>+ Debezium CDC")]
      Kafka[["Kafka event bus<br/>topics: money.credited, money.debited,<br/>customer.onboarded, application.decided,<br/>account.frozen, rate.changed,<br/>interest.accrued/capitalised,<br/>recon.closed/break"]]
      SqsFiles[/"SQS: s3-inbound-file-arrived"/]
      SqsNotif[/"SQS: per-channel notification queues<br/>(email, sms, push)"/]
      DLQ[/"SQS DLQs<br/>webhook-replay, file-replay, notif-fail"/]
    end

    %% ================= Scheduler =================
    subgraph Sched["Scheduler  (internal cron)"]
      Cron["EventBridge cron<br/>➜ triggers ECS scheduled tasks"]
      JobIntAcc["Job: Interest accrual (daily 23:59 AEST)"]
      JobIntCap["Job: Interest capitalisation (1st of month 00:05)"]
      JobRecon["Job: Reconciliation (EOD ~19:15 AEST)"]
      JobIntRecon["Job: Monthly interest recon (1st of month 06:00)"]
      JobSFTP["Job: SFTP puller (every 1 min business hours)"]
      JobCleanup["Job: Outbox cleanup, RFI-expiry, session-cleanup"]
    end

    %% ================= Client edges =================
    CustApp -->|HTTPS + JWT| CustAPI
    OpsWeb  -->|HTTPS + JWT via Zero-Trust| OpsAPI
    Sponsor ==>|mTLS + HMAC webhook| WHIn
    Sponsor <-. SFTP over VPN .-> JobSFTP
    Sponsor <-. REST + mTLS .-> SBA

    %% ================= API to services =================
    CustAPI --> Onb
    CustAPI --> Id
    CustAPI --> Acc
    CustAPI --> Tx
    CustAPI --> PSvc
    OpsAPI  --> Onb
    OpsAPI  --> Acc
    OpsAPI  --> Led
    OpsAPI  --> Recon
    OpsAPI  --> S3Kyc
    WHIn    --> PIn
    WHIn    --> PSvc

    %% ================= Domain interactions =================
    Onb  --> Acc
    Onb  --> S3Kyc
    Tx   --> Acc
    Tx   --> Led
    Tx   --> FraudStub
    Tx   --> Redis
    PIn  --> SBA
    PIn  --> Acc
    PIn  --> Led
    PIn  --> Redis
    Int  --> Led
    PSvc --> SBA
    Recon --> SBA
    Recon --> Led

    %% ================= Data ownership =================
    Onb  --> PG
    Acc  --> PG
    Led  --> PG
    Tx   --> PG
    PIn  --> PG
    Int  --> PG
    PSvc --> PG
    Recon --> PG

    %% ================= Outbox / events =================
    Led   --> Outbox
    Onb   --> Outbox
    Acc   --> Outbox
    Int   --> Outbox
    Recon --> Outbox
    Outbox -->|Debezium tails WAL| Kafka
    Kafka --> Notif
    Kafka --> S3Aud
    Kafka -.-> FraudStub
    Notif --> SqsNotif

    %% ================= Files =================
    JobSFTP --> S3In
    S3In -->|S3 Event| SqsFiles
    SqsFiles --> PIn

    %% ================= Schedulers =================
    Cron --> JobIntAcc --> Int
    Cron --> JobIntCap --> Int
    Cron --> JobRecon --> Recon
    Cron --> JobIntRecon --> Recon
    Cron --> JobSFTP
    Cron --> JobCleanup

    %% ================= DLQs =================
    PIn  -.-> DLQ
    Notif -.-> DLQ
```

**Read the diagram in three passes:**
1. **Top row** — how the outside world enters (three ingress paths: customer JWT, ops via zero-trust, sponsor via mTLS webhook + SFTP).
2. **Middle band** — the domain services, plus the one **Ledger** service that is the sole writer to the postings tables and the one **Sponsor-Bank Adapter** that is the sole talker to the sponsor.
3. **Bottom band** — how data flows *after* a service acts: Postgres for state, outbox → Kafka for events, S3 for evidence and audit, SQS for point-to-point work queues.

---

## Service inventory

The Velo backend is a **modular monolith** with 10 domain modules + 2 boundary modules. Each has:
- Its **own Postgres schema** (nobody else writes it).
- Its **own package/module** in the repo (`/services/<name>`).
- A well-defined **internal API** (in-process function calls for MVP; the same shape becomes a network API when we later split a service out).

| # | Service | Purpose | Postgres schema (tables owned) | Sync API surface | Events published | Events consumed | Scheduled jobs |
|---|---|---|---|---|---|---|---|
| 1 | **Identity** | Facade over Cognito; JWT verify; MFA orchestration | (none — Cognito is source of truth) | `verifyToken`, `initiateStepUp` | — | — | — |
| 2 | **Onboarding** | Application state machine, KYC docs, 4-eyes decisions | `application`, `application_document`, `application_decision`, `customer`, `customer_address`, `customer_consent` | `createApplication`, `uploadDoc`, `submitDecision`, `confirmDecision`, `getApplication` | `customer.onboarded`, `customer.kyc_rejected`, `application.decided` | — | RFI-expiry sweeper (nightly) |
| 3 | **Account** | Product-level accounts, BSB+account issuance, freeze/unfreeze | `account`, `payee` | `openAccount`, `freezeAccount`, `unfreezeAccount`, `resolveByBSBAcct`, `getAccountsForCustomer` | `account.opened`, `account.frozen`, `account.unfrozen`, `account.closed` | `customer.onboarded` | — |
| 4 | **Ledger** | **Sole writer to postings** — atomic double-entry, idempotency, balance projection | `transaction`, `posting`, `ledger_account`, `outbox_event` | `postTransaction`, `getByIdempotencyKey`, `getBalance`, `getHistory` | `money.credited`, `money.debited`, `money.moved` (via outbox) | — | — |
| 5 | **Transfers** | Customer-initiated internal Velo→Velo transfers | (none — writes go through Ledger) | `createTransfer` | (via Ledger) | — | — |
| 6 | **Payments-In** | Ingest of inbound DE files + NPP webhooks | `inbound_file`, `inbound_message`, `payment_instruction` | `handleWebhook`, `processFile`, `resolveUnmatched` | (via Ledger) + `de.unmatched`, `npp.rejected` | `s3.file_arrived` (SQS) | — |
| 7 | **PayID** | Alias registration/deregistration with sponsor | `payid_registration` | `registerPayID`, `deregisterPayID`, `getPayID` | `payid.registered`, `payid.deregistered` | — | — |
| 8 | **Interest** | Daily accrual + monthly capitalisation + rate registry | `interest_accrual`, `rate_version`, `product` | `accrueForDate`, `capitaliseForMonth`, `proposeRate`, `approveRate` | `interest.accrued.day_closed`, `interest.capitalised.month_closed`, `rate.changed` | — | Accrual (23:59 daily), Capitalisation (1st @ 00:05) |
| 9 | **Reconciliation** | Daily sponsor-vs-Velo recon + monthly interest recon + break management | `recon_run`, `recon_break`, `recon_break_resolution` | `runDailyRecon`, `runMonthlyInterestRecon`, `resolveBreak`, `confirmResolveBreak` | `recon.closed`, `recon.break` | — | Daily EOD recon (~19:15), Monthly interest recon (1st @ 06:00) |
| 10 | **Notifications** | Push/email/SMS delivery, per-channel isolation | `notification_delivery` (audit only) | `sendNotification` | `notification.sent`, `notification.failed` | `money.credited`, `money.debited`, `customer.onboarded`, `application.decided`, `recon.break` | — |
| 11 | **Sponsor-Bank Adapter** *(boundary)* | Only place the sponsor's SDK/SFTP client lives; hides sponsor quirks | (none — stateless client) | `postNPP`, `pullSFTPFiles`, `getStatement`, `registerPayIDWithSponsor` | — | — | SFTP puller (every 1 min business hours) |
| 12 | **Ops Portal API** *(boundary)* | HTTP layer serving Ops web app; wires 4-eyes to underlying services | (none — proxies to owning services) | REST endpoints per capability | (via underlying services) | — | — |
| 13 | *Fraud* | Stub in MVP (`ALLOW` everything); real rules engine in roadmap | (none in MVP) | `scoreTransfer` | (via consumer of `money.moved`) | `money.moved` (roadmap) | — |

**Ownership rule:** Only the service that owns a table may write to it. Others read via the owning service's API. Enforced by:
- One Postgres role per module; only that role has `INSERT/UPDATE` grants on its own tables.
- Code review — cross-schema queries are a smell.
- Tests: a service's tests must not include another service's tables in their setup.

---

## Ingress paths (three, and only three)

Everything the backend does is triggered by one of these three edges:

| Edge | Auth | Target | Latency SLO |
|---|---|---|---|
| **Customer App API** | Cognito JWT (customer pool) + step-up MFA for sensitive ops | Onboarding, Identity, Account, Transfers, PayID | P99 ≤ 300 ms |
| **Ops Portal API** | Cognito JWT (ops pool) + Zero-Trust (Verified Access) + RBAC + 4-eyes on sensitive actions | Onboarding, Account, Ledger (read-only), Recon, S3 KYC (short-lived presigned URLs) | P99 ≤ 500 ms |
| **Sponsor Webhook Receiver** | mTLS (sponsor cert pinned) + HMAC signature + timestamp replay window | Payments-In, PayID | P99 ≤ 400 ms (sponsor SLA ~5s hard cap) |

Plus **one non-request-driven trigger surface**:

| Trigger | Mechanism | Target |
|---|---|---|
| **Scheduled jobs** | EventBridge cron → ECS scheduled task | Interest, Reconciliation, SFTP puller, cleanup jobs |
| **File-arrived events** | S3 event → SQS → subscriber | Payments-In (DE), Reconciliation (statement files) |
| **Domain events** | Kafka topic → consumer group | Notifications, Audit-lake sink, (roadmap) Fraud/AML/CDR |

Any code path that mutates money or state ultimately traces back to one of these five triggers.

---

## Internal API map (who calls whom, synchronously)

Solid arrow = HTTP/gRPC (in the split future) / function call (today). Dotted = event.

```mermaid
flowchart LR
    Onb --> Acc
    Onb --> Id
    Tx  --> Acc
    Tx  --> Led
    Tx  --> Fraud
    PIn --> Acc
    PIn --> Led
    PIn --> SBA
    PSvc --> SBA
    Int --> Led
    Recon --> Led
    Recon --> SBA
    Notif -.-> KafkaTopic
    Fraud -.-> KafkaTopic
    OpsAPI --> Onb
    OpsAPI --> Acc
    OpsAPI --> Led
    OpsAPI --> Recon
```

**Sync-call rules of thumb:**
- **Only three services call `Ledger`**: Transfers, Payments-In, Interest. Nobody else writes money.
- **Only two services call `Sponsor-Bank Adapter`**: Payments-In (webhook flow, statement pulls) and PayID (registration). All external side effects are contained.
- **`Account.resolveByBSBAcct`** is the hottest read path (called on every inbound payment). Cached in Redis with a short TTL and invalidated on account state changes via a Kafka subscriber.
- **`Notifications`** and **`Fraud`** are pure event consumers. They never receive sync calls from other domain services.

---

## Data stores — what lives where, and why

### Postgres (Aurora) — the operational database

- **One logical DB, one schema per module** (`onboarding`, `account`, `ledger`, `transfers`, `payments_in`, `payid`, `interest`, `reconciliation`).
- Each module has its own migration folder and its own Postgres role. Cross-schema `SELECT`s are allowed for read models; cross-schema `INSERT/UPDATE` is forbidden by grants.
- **Ledger schema is special** — it's the only place with `SERIALIZABLE` isolation on writes, WAL-based CDC tailed by Debezium, and DB-level triggers enforcing double-entry (`SUM(posting.amount * sign) = 0` per transaction per currency).

### Redis (ElastiCache) — the cache tier

Two distinct concerns, kept in separate logical databases for clean invalidation:

| Concern | Key shape | Value | TTL | Populated when |
|---|---|---|---|---|
| **Idempotency cache** (webhook + API) | `idem:npp:<sponsor_msg_id>`, `idem:tx:<client_uuid>` | Cached response body | 24h | Only after **successful** commit in Postgres (never on failure — see Workflow 03 Q14) |
| **Hot balance cache** | `bal:<ledger_account_id>` | `{available_cents, last_posted_at}` | 60s | On read miss; invalidated by a Kafka consumer subscribed to `money.moved` |
| **Account resolution cache** | `acc:<bsb>-<acct>` | `{account_id, ledger_account_id, status}` | 300s | On read miss; invalidated on `account.frozen/unfrozen/closed` events |
| **Rate-limit counters** | `rl:<endpoint>:<ip>` | count | window | Every request |

**Correctness rule:** Redis is a **cache**, never a store. Every value is safely recomputable from Postgres. Redis being down = slower, not wrong.

### S3 — three buckets, three purposes

| Bucket | What's in it | Retention | Encryption | Written by |
|---|---|---|---|---|
| `velo-kyc-docs` | Uploaded ID docs, selfies | 7-year Object Lock | KMS-CMK `kms/velo/data-kyc` | Onboarding (write); Ops Portal API (short-lived read via presigned URL) |
| `velo-inbound-files` | Raw DE files, raw sponsor statements, in original format (`.cemtex.pgp`, `.camt053.xml`) | 7-year Object Lock | KMS-CMK `kms/velo/data-general` | SFTP puller (write); Payments-In (read); Reconciliation (read) |
| `velo-audit-lake` | Domain events (JSON, partitioned by topic/date/hour) + raw external artefacts snapshotted for evidence | 7-year Object Lock (Compliance mode) + CRR to Melbourne | KMS-CMK `kms/velo/data-audit` (separate CMK, none of the ledger service roles can decrypt this key) | MSK S3 Sink Connector; snapshot jobs |

---

## Messaging — outbox, Kafka, and SQS

Three distinct messaging surfaces used for different jobs. Confusing them is the number-one MVP anti-pattern to avoid.

### 1. Outbox table + Debezium → Kafka (for domain events)

**Pattern:** every service that publishes an event inserts an `outbox_event` row inside the **same DB transaction** as its business write. Debezium tails Postgres WAL and publishes each row to the matching Kafka topic. The service never calls Kafka directly.

```
BEGIN;
  INSERT/UPDATE business rows;
  INSERT INTO outbox_event (id, topic, payload, status='PENDING');
COMMIT;   -- both or neither
```

**Why:** exactly-once effect. Killing the "dual-write" problem where DB commits but Kafka publish fails and downstream never learns.

**Topics** (see Event Catalog below):
- `money.credited`, `money.debited`, `money.moved`
- `customer.onboarded`, `customer.kyc_rejected`, `application.decided`
- `account.opened`, `account.frozen`, `account.unfrozen`, `account.closed`
- `rate.changed`
- `interest.accrued.day_closed`, `interest.capitalised.month_closed`
- `recon.closed`, `recon.break`
- `payid.registered`, `payid.deregistered`

### 2. SQS (for point-to-point work queues)

**Pattern:** producer writes a message, exactly one consumer processes it.

| Queue | Producer | Consumer | Retry policy |
|---|---|---|---|
| `sqs-inbound-file-arrived` | S3 event on `velo-inbound-files` | Payments-In (DE parser), Reconciliation (statement parser — different prefix) | 5 attempts w/ exponential backoff → DLQ |
| `sqs-notif-email` | Notifications Service | SES sender | 5 attempts → DLQ |
| `sqs-notif-sms` | Notifications Service | Twilio sender | 5 attempts → DLQ |
| `sqs-notif-push` | Notifications Service | APNs/FCM sender | 3 attempts → DLQ |
| `sqs-webhook-replay-dlq` | Payments-In on persistent failure | On-call engineer | manual |
| `sqs-file-replay-dlq` | Payments-In on persistent failure | On-call engineer | manual |

### 3. Kafka (for durable, multi-consumer, replayable event stream)

Same events as (1), but treated as a **log** rather than a queue. Consumers:
- **Notifications** — subscribes to money + status events.
- **S3 Sink Connector** — mirrors every event to the audit lake for compliance retention + Athena queries.
- **(Roadmap) Fraud** — subscribes to `money.moved`, `account.*`, `customer.onboarded`.
- **(Roadmap) AML/CTF** — subscribes to `money.*` and `customer.*`.
- **(Roadmap) CDR data-holder** — subscribes to `customer.onboarded`, `customer.consent.*`.
- **(Roadmap) Analytics warehouse** — subscribes to everything, batch-sinked to a data lake.

**Retention:** 7 days on Kafka itself (buffer for consumer downtime), 7 years in the S3 audit-lake mirror (compliance).

---

## Domain event catalog (v1)

Common envelope on every event:

```json
{
  "event_id": "01HXYZ...",              
  "event_type": "money.credited",
  "event_version": "1.0",
  "occurred_at": "2026-09-27T09:15:23.421Z",
  "producer":  {"service":"payments-in","version":"2.14.3"},
  "actor":     {"type":"system"} ,      
  "aggregate": {"type":"account","id":"acc_..."} ,
  "correlation_id":"req_...",
  "causation_id":"tx_...",
  "payload":   { ... event-specific ... }
}
```

| Topic | Producer | Payload keys | Consumers |
|---|---|---|---|
| `money.credited` | Ledger (via outbox) | `transaction_id`, `amount_cents`, `currency`, `rail`, `customer_id`, `external_ref`, `posted_at` | Notifications, S3 sink, Fraud (roadmap), AML (roadmap) |
| `money.debited` | Ledger | `transaction_id`, `amount_cents`, `currency`, `rail`, `customer_id`, `external_ref`, `posted_at` | Notifications, S3 sink, Fraud, AML |
| `money.moved` | Ledger | full transaction summary | S3 sink, CDR, Analytics |
| `customer.onboarded` | Onboarding | `customer_id`, `application_id`, `onboarded_at` | Account (creates the account), Notifications, S3 sink |
| `customer.kyc_rejected` | Onboarding | `customer_id`, `application_id`, `reason_code` | Notifications, S3 sink |
| `application.decided` | Onboarding | `application_id`, `decision`, `is_confirmation`, `reviewer_id` | S3 sink, Ops metrics |
| `account.opened` | Account | `account_id`, `customer_id`, `bsb`, `account_number` | Notifications, S3 sink |
| `account.frozen` / `.unfrozen` | Account | `account_id`, `reason_code`, `by_user_id` | Payments-In (invalidates resolution cache; starts routing to HOLD), Notifications, S3 sink |
| `rate.changed` | Interest | `product_code`, `rate_version_id`, `rate_pa`, `effective_from` | Interest job (picks new version), S3 sink |
| `interest.accrued.day_closed` | Interest | `date`, `accrual_count`, `sum_cents` | Recon, S3 sink |
| `interest.capitalised.month_closed` | Interest | `month`, `capitalisation_count`, `sum_cents` | Recon, Reporting, S3 sink |
| `recon.closed` / `.break` | Reconciliation | `source`, `business_date`, `status`, `break_details?` | On-call pager (on `.break`), S3 sink |
| `payid.registered` / `.deregistered` | PayID | `account_id`, `alias_type`, `alias_value` | Notifications, S3 sink |

**Versioning discipline:** additive fields = same major version; removals or renames = bump major (`money.credited.v2`) and run both topics in parallel until all consumers migrate.

---

## Scheduled job catalog

All jobs are **idempotent** on their inputs (accrual: `(account_id, date)`; capitalisation: `(account_id, yyyy-mm)`; recon: `(source, business_date)`). Re-running is safe.

| Job | Schedule (AEST) | Owner service | Duration budget | On failure |
|---|---|---|---|---|
| **Interest accrual** | 23:59 daily | Interest | ≤ 10 min at 100k accounts | Retry at 04:00; page on-call if second failure |
| **Interest capitalisation** | 00:05, 1st of month | Interest | ≤ 30 min | Retry at 04:00; page treasury on-call on second failure |
| **Daily reconciliation** (DE, NPP, pool balance) | ~19:15 daily (after sponsor EOD file) | Reconciliation | ≤ 5 min | Retry every 30 min for 2 h; page on-call if still broken; block outbound (roadmap) |
| **Monthly interest recon** | 06:00, 1st of month | Reconciliation | ≤ 15 min | Page treasury; block rate changes until resolved |
| **SFTP puller** | every 1 min, business hours 07:00–20:00 | Sponsor-Bank Adapter | ≤ 30s per run | Alert if backlog grows > 5 files or run takes > 90s |
| **Outbox cleanup** (delete rows `SENT` older than 7 days) | 02:00 daily | Ledger | ≤ 2 min | Best-effort; a stuck outbox row is a bigger problem than a big table |
| **RFI-expiry sweeper** (auto-expire onboarding applications idle > 14 days) | 03:00 daily | Onboarding | ≤ 2 min | Retry next day |
| **Session cleanup** (Cognito refresh-token revocation for inactive) | 04:00 daily | Identity | ≤ 5 min | Retry |
| **Balance projection sanity check** (assert `available_balance == SUM(postings)`) | 05:00 daily | Ledger | ≤ 15 min | On mismatch: alert P1, freeze rate changes, page on-call |
| **CDC health probe** (assert Debezium lag < threshold) | every 5 min | Ledger | seconds | Page on-call if lag > 60s |

---

## Data ownership — the "one-writer" table

Encoded as Postgres roles + grants, so a compromised service literally cannot write another module's tables.

| Owner service | Schema | Writable tables | Read-only from |
|---|---|---|---|
| Onboarding | `onboarding` | `application`, `application_document`, `application_decision`, `customer`, `customer_address`, `customer_consent` | Ops Portal API (via service call) |
| Account | `account` | `account`, `payee` | Payments-In, Transfers, Ops Portal API |
| Ledger | `ledger` | `transaction`, `posting`, `ledger_account`, `outbox_event` | Nobody writes; Reconciliation and Ops Portal read via API |
| Payments-In | `payments_in` | `inbound_file`, `inbound_message`, `payment_instruction` | Ops Portal API (unmatched queue) |
| PayID | `payid` | `payid_registration` | Ops Portal API |
| Interest | `interest` | `interest_accrual`, `rate_version`, `product` | Reconciliation |
| Reconciliation | `reconciliation` | `recon_run`, `recon_break`, `recon_break_resolution` | Ops Portal API |
| Notifications | `notifications` | `notification_delivery` (append-only audit) | — |
| Identity | (none in Postgres) | — | — |

---

## Cross-cutting patterns (in one place)

### Idempotency (3-layer defence)

Every money-moving path has **three independent idempotency checks**:

1. **Source-level** — the sponsor's `sponsor_msg_id` / `sponsor_file_id + record_index` / client-generated UUID for internal transfers.
2. **Ledger-level** — `TRANSACTION.idempotency_key UNIQUE` in Postgres, computed deterministically from source (never `NOW()`).
3. **Redis-level** — cached response for successful posts, TTL 24 h. Speed only; correctness is on (2).

### Outbox pattern (already explained above)

Never dual-write. Always outbox + CDC.

### Serialisable-with-retry (for the ledger)

Ledger writes run in `REPEATABLE READ` with row-level `SELECT ... FOR UPDATE` on the balance projection as the primary concurrency control, and `SERIALIZABLE` as the fallback for any transaction that recomputes from `SUM(postings)`. On `40001`, retry with jitter up to 3 times.

### 4-eyes for all Ops mutations of money or reference data

One row-per-decision model (introduced by `APPLICATION_DECISION`) reused everywhere:
- Onboarding approve/reject
- Rate change proposal/approval
- Recon-break resolution
- Account freeze/unfreeze
- Manual reversal
- KMS-CMK key rotation (via infra pipeline, but same pattern)

### Presigned URLs for KYC document access

Ops never receives raw document bytes. Ops Portal API mints a **≤ 5-minute presigned S3 URL** scoped to a single object. Cognito group + application-review gate must both pass.

### Circuit breakers on external calls

Every call to the Sponsor-Bank Adapter is behind a circuit breaker (Hystrix-style: 50% error rate over 20 requests → open for 30s → half-open probe). Prevents thundering-herd retries when sponsor is degraded.

### Rate limits on customer-facing endpoints

At the API Gateway (WAF layer, see cloud arch doc) *and* in-app at the service (belt-and-braces):
- Login: 5/min/IP
- OTP resend: 2/min/user
- Create transfer: 10/min/user + a per-day amount cap
- KYC upload: 20/hour/application

---

## Repository layout (single-repo, modular monolith)

```
velo-backend/
├── package.json                       # single build, single test suite
├── infrastructure/                    # Terraform / CDK (deploys 06-cloud-architecture)
│
├── services/                          # each folder = a bounded context
│   ├── identity/
│   │   ├── api/                       # HTTP handlers exposed at the API gateway
│   │   ├── domain/                    # pure domain logic (no I/O)
│   │   ├── infra/                     # Cognito clients, repositories
│   │   └── __tests__/
│   ├── onboarding/
│   ├── account/
│   ├── ledger/
│   ├── transfers/
│   ├── payments-in/
│   ├── payid/
│   ├── interest/
│   ├── reconciliation/
│   ├── notifications/
│   └── sponsor-bank-adapter/          # the ONLY module that imports sponsor SDK
│
├── platform/                          # shared infra concerns (no domain logic)
│   ├── db/                            # Postgres pool, migration runner, per-role connection helper
│   ├── cache/                         # Redis clients
│   ├── outbox/                        # generic outbox writer + Debezium config
│   ├── events/                        # envelope, versioning, schema registry
│   ├── auth/                          # JWT verification, RBAC, 4-eyes primitives
│   ├── telemetry/                     # OpenTelemetry setup, structured logging
│   └── errors/                        # standard error taxonomy
│
├── apps/                              # deployable entrypoints (thin process shells)
│   ├── api-customer/                  # customer app HTTP entrypoint
│   ├── api-ops/                       # ops portal HTTP entrypoint
│   ├── api-webhook/                   # sponsor webhook HTTP entrypoint
│   └── scheduled/                     # scheduled-task entrypoints (interest, recon, sftp, cleanup)
│
└── contracts/                         # shared type + schema definitions across services
    ├── events/                        # event schemas (JSON Schema / Avro)
    └── api/                           # OpenAPI specs
```

**Why this shape:**
- **`apps/` are thin.** They wire configuration + start an HTTP or scheduled process; they don't contain business logic. This is what lets us later spin out an app (e.g. `apps/api-webhook`) onto its own binary + service without touching domain code.
- **`services/*/domain/` is pure.** No DB, no HTTP, no Kafka. Testable with a stopwatch. This is where 80% of correctness lives.
- **`services/*/infra/` is the adapter layer.** Postgres repositories, Redis clients, Kafka producers/consumers. Swap for tests.
- **`platform/` is not a service** — it's shared plumbing every service can depend on.
- **`contracts/`** is versioned separately; breaking a contract requires a bumped version + parallel run.

---

## Runtime shape today vs future

**Today (MVP):** the whole backend runs as **~4 deployable binaries**, all built from the same repo:

| Deployable | Contains | Why grouped |
|---|---|---|
| `api-customer` | Identity, Onboarding, Account, Transfers, PayID, (Ledger via calls) | Customer-facing latency profile |
| `api-webhook` | Payments-In webhook handler, PayID sponsor-callback handler | Different security posture (mTLS); different scaling profile (sponsor bursts) |
| `api-ops` | Ops Portal API surface | Different auth (Zero-Trust); different SLA |
| `scheduled` | Interest, Reconciliation, SFTP puller, Cleanups | Batch, non-request-driven |

**Split-later map** (what we spin out first as the business grows):

1. **Ledger** (highest correctness sensitivity + throughput ceiling) → its own binary + hardened deployment.
2. **Payments-In** (highest external-facing burst load + security posture) → already separate; grows autoscaling independently.
3. **Sponsor-Bank Adapter** → its own binary once we introduce a second sponsor.
4. **Notifications** → its own binary once we onboard partners that push notification traffic through it.

Because everything already goes through internal APIs and events, splitting a service = extracting an already-clean interface, wiring an HTTP or gRPC transport, and pointing DNS. Not a rewrite.

---

## What this doc deliberately does not show

- **VPC subnets, IAM roles, KMS keys, ALB, ECS Fargate, WAF, CloudFront** — see `06-cloud-architecture.md`.
- **CI/CD pipeline, testing strategy** — see `06-cloud-architecture.md`.
- **DR / RPO / RTO / cross-region** — see `06-cloud-architecture.md`.
- **Individual UX flows** — see `docs/workflows/`.
- **Data schema in full field detail** — see `docs/05-erd-consolidated.md`.

The reason this doc is small enough to hold in your head: it deliberately stops at "how the application is organised" and lets the other docs answer "how it's deployed" and "what it does step-by-step".
