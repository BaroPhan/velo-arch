# Workflow 04 — Real-time Internal (Velo → Velo) Transfer

**MVP requirement:** R5 — Real-time intra-platform transfers between Velo customers with **immediate funds availability**.

**Rail context:** No external rail. Both sides of the transfer are on Velo's ledger. Settlement is purely intra-ledger; the sponsor bank sees nothing (the pool position doesn't change). This is the simplest external picture and the most interesting *internal* picture — it's the first place your ledger design gets stress-tested.

## Sample case

> **Priya** owes her flatmate **Alex** $85 for utilities. Both banked at Velo. Priya opens the Velo app, taps **Send Money**, chooses Alex (already in her Velo contacts because they're both Velo customers), enters $85, notes "Utilities Sep", and confirms with biometrics.
>
> Velo's **Transfers Service** validates the instruction, calls the **Ledger** to execute a single atomic transaction: **DEBIT Priya's customer account, CREDIT Alex's customer account**, both in AUD, both immediately available. The ledger commits in a single serialisable Postgres transaction. Priya's app updates within ~200ms to show the new balance; Alex's phone buzzes ~500ms later with a push notification. Done.
>
> **No sponsor bank involvement, no NPP, no BECS.** Just a book transfer on the sub-ledger.

## Actors

- **Priya** (payer) and **Alex** (payee) — both Velo customers.
- **Velo App** — customer surface.
- **Velo Transfers Service** — validates, enforces limits, orchestrates.
- **Velo Ledger** — atomic double-entry.
- **Velo Account Service** — resolves recipient (name, ability to receive).
- **Fraud hook (stub in MVP)** — a pass-through in MVP; the seam exists for roadmap.
- **Notifications Service** — push/email to both parties.

## Sequence diagram

```mermaid
sequenceDiagram
    autonumber
    actor P as Priya (payer)
    participant App as Velo App
    participant Id as Identity Svc / Cognito
    participant Tx as Transfers Svc
    participant Acc as Account Svc
    participant FR as Fraud stub
    participant Led as Ledger Svc
    participant PG as Aurora Postgres
    participant K as Kafka MSK
    participant Notif as Notifications Svc
    actor A as Alex (payee)

    P->>App: Open Send Money, choose Alex, $85, note
    App->>Id: Step-up MFA if amount>threshold
    Id-->>App: OK
    P->>App: Confirm biometrics
    App->>Tx: POST /transfers {from_account, to_account_or_alias, amount=8500, note, idempotency_key}

    Tx->>Acc: Resolve payee (velo customer, ACTIVE)
    Acc-->>Tx: payee account_id, name for display
    Tx->>Tx: Enforce limits (per-day, per-txn, cool-off if new payee)
    Tx->>FR: Score(source, dest, amount, device, geo)
    FR-->>Tx: allow (MVP: always allow)

    Tx->>Led: POST /transactions {type:INTERNAL_TRANSFER, idempotency_key,\n postings:[DR payer 8500, CR payee 8500]}
    Led->>PG: BEGIN SERIALIZABLE
    Led->>PG: SELECT payer.available_balance FOR UPDATE
    alt sufficient funds
        Led->>PG: INSERT postings x2
        Led->>PG: UPDATE payer.balance -= 8500
        Led->>PG: UPDATE payee.balance += 8500
        Led->>PG: INSERT transaction (POSTED)
        Led->>PG: COMMIT
        Led-->>Tx: OK {transaction_id, new_balances}
        Tx-->>App: 200 OK {tx_id, new_balance_from}
        App-->>P: "Sent $85 to Alex. New balance $X"
        Led->>K: publish "money.moved" (INTERNAL_TRANSFER)
        K-->>Notif: consume x2 subscribers
        Notif-->>P: (optional) confirmation email
        Notif-->>A: Push + email "$85 from Priya — Utilities Sep"
    else insufficient funds
        Led->>PG: ROLLBACK
        Led-->>Tx: 409 INSUFFICIENT_FUNDS
        Tx-->>App: 409
        App-->>P: "Insufficient funds"
    end
```

## Alternative flows

- **Payee not on Velo** — MVP: block ("recipient not found"). Roadmap: fall through to outbound NPP/PayID.
- **Frozen payee account** — reject (`RECIPIENT_UNAVAILABLE`).
- **Frozen payer account** — reject.
- **Amount above per-txn or per-day limit** — soft reject with reason; require step-up.
- **New payee cool-off** — if payee added in last 24 h, cap to $500 first day (fraud control seam).
- **Idempotent retry** — client sends same `idempotency_key`; Ledger returns the previously committed result unchanged.
- **Deadlock / serialisation conflict** — Ledger retries on `40001` up to 3 times with jitter; if all fail, `503` and client re-tries.
- **Reversal** (Ops-initiated) — a new transaction of type `REVERSAL` posts the mirror; original is marked `REVERSED`. Original is **never** deleted; the ledger is append-only.

## Concurrency correctness — the interview-friendly explanation

The internal transfer is a nice bite-sized case to demo your understanding of ledger correctness. Bring up:

1. **Two postings must be atomic.** They live in the same DB transaction. Kafka publish happens **after** commit (outbox pattern).
2. **Sufficient-funds check** uses `SELECT ... FOR UPDATE` on the payer's balance row to prevent race between two concurrent transfers from the same account. Or better: apply the delta with a conditional `UPDATE ... WHERE available_balance >= X` and check `affected rows`.
3. **Serialisable isolation** avoids phantom-read hazards on `SUM(postings)` if you ever recompute the balance from the ledger inside the same transaction.
4. **Balance projection vs ledger truth.** The balance row is a *cache/projection*. If it disagrees with `SUM(postings)`, we trust the postings and repair. The daily recon job asserts equality.
5. **Idempotency key** is client-generated (`UUIDv4` per user tap), unique-indexed on the transactions table. Prevents double-spend on retries.
6. **Kafka outbox.** Ledger writes an `outbox` row in the same DB txn; a Debezium/CDC connector publishes to Kafka. Guarantees exactly-once effect on the ledger with at-least-once delivery downstream.

## ERD delta

No new tables — this workflow exercises the **existing** `TRANSACTION` + `POSTING` + `LEDGER_ACCOUNT`. We add a couple of columns and one auxiliary table:

```mermaid
erDiagram
    LEDGER_ACCOUNT ||--o{ POSTING : "has"
    TRANSACTION ||--|{ POSTING : "composed of"
    TRANSACTION ||--o{ OUTBOX_EVENT : "emits"
    CUSTOMER ||--o{ PAYEE : "saves"

    TRANSACTION {
        uuid id PK
        string type "INTERNAL_TRANSFER|DE_INBOUND|NPP_INBOUND|INTEREST|REVERSAL"
        string status "PENDING|POSTED|REVERSED|FAILED"
        string idempotency_key UK
        uuid reversed_by_tx_id "self-ref, nullable"
        text note "user-entered memo, PII scrubbed for logs"
        jsonb device_context "for fraud/roadmap"
        timestamp created_at
        timestamp posted_at
    }

    OUTBOX_EVENT {
        uuid id PK
        uuid transaction_id FK
        string topic
        jsonb payload
        string status "PENDING|SENT"
        timestamp created_at
        timestamp sent_at
    }

    PAYEE {
        uuid id PK
        uuid customer_id FK "owner"
        uuid payee_account_id FK "if internal"
        string external_bsb "roadmap"
        string external_account "roadmap"
        string alias "friendly name"
        boolean is_internal
        timestamp added_at
    }
```

## How our services interact at the cloud level

```mermaid
flowchart LR
    App[Velo App PWA/Mobile] --> CF[CloudFront + WAF]
    CF --> APIGW[API Gateway / ALB]
    APIGW --> Tx[Transfers Svc\nECS Fargate]
    Tx --> Acc[Account Svc]
    Tx --> FR[Fraud stub\n(future: rules engine)]
    Tx --> Led[Ledger Svc]
    Led --> PG[(Aurora Postgres Multi-AZ)]
    PG -- outbox rows --> CDC[Debezium on MSK Connect]
    CDC --> MSK[[Kafka MSK: money.moved]]
    MSK --> Notif[Notifications Svc]
    Notif --> SES
    Notif --> APNs
    Tx -.-> Redis[(ElastiCache: hot balances + limits)]
```

### Cloud-level interaction narrative
1. **Transfers Service** is a normal HTTP service behind API Gateway; it does not talk to the sponsor bank at all.
2. **Ledger Service** owns the database. All writes go through it. It exposes an internal-only gRPC/HTTP surface, no direct DB access from any other service.
3. **Outbox pattern** on Aurora: the ledger commits `transaction`, `posting`s, and an `outbox_event` row in one transaction. A **Debezium** source connector running on **MSK Connect** tails the WAL and publishes to the `money.moved` topic. Downstream `Notifications` consumer sends push/email/SMS.
4. **Redis** caches the current balance for a customer for fast reads (< 1 ms). Cache is invalidated by the same outbox pipeline. The **ledger** never trusts the cache for correctness — only for reads to the app.
5. **Step-up auth** for higher-value transfers routes back through **Cognito** for a fresh MFA token.
6. **RBAC + segregation of duties** for Ops-initiated **reversals** — always four-eyes, always audited, always via the Ops Portal (never the customer app).
7. **Blast radius**: Transfers Service is stateless and horizontally scaled; a compromise cannot mutate the ledger without a signed request through the Ledger Service's internal API.
8. **DR**: internal transfers are the workflow that must never be down; Aurora Multi-AZ + Ledger Service Multi-AZ + Redis Multi-AZ; app failover is seamless within region.

## Trade-offs

- **We chose Postgres serialisable + row-lock, not an event-sourced ledger** (e.g. Axon, EventStoreDB). Postgres is boring, understood, and fast enough for MVP throughput. Event-sourcing gives replayability but is overkill at seed stage and harder to hire for.
- **Kafka outbox instead of dual-write**. We accept ~sub-second delay between commit and downstream notification in exchange for exactly-once correctness. Fine for UX.
- **Fraud is a stub in MVP.** Interface exists (`FraudService.score(tx)`); default is `ALLOW`. Roadmap plugs in a real engine without touching Transfers or Ledger.
- **Notes/memos are stored in the ledger.** Simpler than a side-table; but we scrub them from logs (PII).
