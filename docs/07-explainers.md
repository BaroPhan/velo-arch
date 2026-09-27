# 07 — Explainers (banking, ledger, security, and Velo vs Vietnamese e-wallets)

A background/glossary doc for the terms used across the prep pack. Written for someone who is technically strong but new to Australian retail banking. Vietnamese banking analogies included throughout.

**Contents**
1. [Australian payment rails: BSB + account, PayID, NPP, Direct Entry](#1-australian-payment-rails)
2. [FCS and what "auditable" really means](#2-fcs-and-what-auditable-really-means)
3. [Restricted ADI vs Full ADI](#3-restricted-adi-vs-full-adi)
4. [Double-entry, sub-ledger vs GL, idempotency, exactly-once](#4-ledger-mechanics-double-entry-sub-ledger-vs-gl-idempotency-exactly-once)
5. [Trade-offs, in depth](#5-trade-offs-in-depth)
6. [PII and Australian data residency](#6-pii-and-australian-data-residency)
7. [Security terms: TLS, KMS-CMK, MFA, WAF, Segregation of Duties](#7-security-terms)
8. [Velo vs ZaloPay / MoMo (Vietnamese e-wallets)](#8-velo-vs-zalopay--momo)
9. [Why a sponsor bank, and how it plugs into our system](#9-sponsor-bank)

---

## 1. Australian payment rails

Yes — **BSB + account number**, **PayID**, **NPP**, and **Direct Entry** are all ways to move money in Australia, but they operate at different layers. Think of it like this:

```
Layer 3  |  Addressing:   BSB + account number    PayID
Layer 2  |  Rail:         Direct Entry (BECS)     NPP (24/7 real-time)
Layer 1  |  Settlement:   RITS (batch, RBA)       FSS (instant, RBA)
```

### BSB + account number

- **BSB** = "Bank State Branch". A 6-digit code that identifies a bank + state + (historically) branch.
  - Format: `XXX-XXX`. Example: `062-000` = Commonwealth Bank (062 = CBA), 000 = NSW head office.
  - Managed by **AusPayNet** (Australian Payments Network).
- **Account number** = up to 9 digits, unique **within** that BSB.
- Together, `BSB + account number` uniquely identifies an account in Australia.
- Vietnamese analogy: like the combination of *mã ngân hàng + số tài khoản*. US analogy: routing number + account number.
- BSB+account can be used with **either** rail (NPP or Direct Entry). It's just the address.

### PayID

- An **alias service** launched with NPP in 2018.
- Instead of memorizing your BSB + account number, you register your **phone number, email, ABN, or Organisation ID** as your PayID.
- Under the hood: a national directory maps `"khoi@example.com"` → `"BSB 802-985, Account 100 234 567, Name Khoi Nguyen"`.
- The **payer's app resolves** the alias, shows the payee's **registered account name** for confirmation, then sends via NPP.
- Vietnamese analogy: like registering your phone number for VietQR / Napas 247's account lookup.
- PayID works **only over NPP**, not over Direct Entry.

### NPP (New Payments Platform)

- Australia's **real-time**, **24/7/365** payments rail. Launched Feb 2018.
- Runs on **ISO 20022** messages (`pacs.008` = credit transfer, `pacs.002` = status).
- Settled on the RBA's **Fast Settlement Service (FSS)** — instant, gross settlement in central bank money.
- Per-transaction limit is currently ~$1M for retail (varies by participant).
- Rich data: up to 280 characters of description, ISO-standard remitter info.
- Overlay services on NPP:
  - **Osko** (by BPAY) — the branded consumer-facing name for real-time payments.
  - **PayID** — the addressing service.
  - **PayTo** — a newer overlay (2023+) that replaces direct debits with a mandate model.
- Governed by **NPPA** (NPP Australia Pty Ltd), owned by 13 major ADIs.
- Vietnamese analogy: **Napas 247** — 24/7 instant transfers between Vietnamese banks.

### Direct Entry (DE) / BECS

- **BECS** = Bulk Electronic Clearing System. The **legacy batch** rail. Older than NPP by decades.
- Files (called "DE files") in a fixed-width flat-file format (CEMTEX).
- **Settled in batches**, typically **three times per day** on business days; funds usually **available the same business day**.
- Zero cost or near-zero — this is why bulk payroll and utility direct debits all run over BECS.
- Used for:
  - **Direct Credits** (payroll, refunds, government benefits)
  - **Direct Debits** (utilities, gym memberships, insurance)
- Governed by **AusPayNet** (the BECS Framework).
- Does **not** support PayID. You must use BSB + account number.
- Vietnamese analogy: **UNC hàng loạt** (bulk transfer files) via the old *Citad* clearing — largely superseded by Napas 247 in Vietnam, but Australia still uses BECS heavily for payroll.

### Two more you'll hear on the day, for context

- **HVCS / SWIFT / RITS** — High Value Clearing System, for large corporate transactions, settled in real time on **RITS** (the RBA's real-time gross settlement system). Not relevant for retail MVP.
- **BPAY** — a bill-payment scheme. Merchant registers a "Biller Code"; customer pays via BSB-account-like address. Separate rail, sponsor-bank has it but roadmap for Velo.

### Which one does Velo use in the MVP?

- **Receive DE credits** (payroll into a customer's account) — YES.
- **Receive NPP + PayID payments** — YES.
- **Send anything externally** — NO (roadmap).
- **Internal transfers Velo → Velo** — pure ledger, no external rail at all.

---

## 2. FCS and what "auditable" really means

### What FCS is

- **FCS** = **Financial Claims Scheme**. An Australian government guarantee, activated only if an ADI actually collapses.
- If Velo failed, APRA would pay out each depositor **up to AUD $250,000 per account holder per ADI** within 7 calendar days of activation.
- Funded by taxpayers first (then a levy on the industry).
- Vietnamese analogy: like Vietnam's **Deposit Insurance of Vietnam (DIV/BHTG)**, which currently covers up to VND 125m per person per credit institution — same idea, smaller cap.

### What FCS demands from our design

Being "FCS-protected" is not something you tick a box for. APRA imposes a rule set called **CPS 910** (and related) that says an ADI must, at any point, be able to produce a **Single Customer View (SCV)** file — one row per depositor with:

- Legal name, DOB, address, contact
- Every deposit account they hold with the ADI
- Balance in AUD at the requested point in time
- Any set-offs (loans against deposits — n/a for Velo MVP)
- Ability to hand this to APRA within a very short window in a resolution event

So the practical requirements this imposes on Velo's data model:

| Requirement | What we build |
|---|---|
| Balance at any point in time, per customer | Ledger is append-only; you can reconstruct any historical balance from `SUM(postings WHERE ledger_account_id = X AND posted_at <= T)` |
| Uniquely identify each depositor | `CUSTOMER` PK + verified `kyc_status` |
| Aggregate exposure per depositor | `SUM(available_balance)` across all accounts owned by a customer |
| RADI aggregate exposure | `SUM(available_balance)` across all customer-class ledger accounts |
| Auditor-quality reports | Immutable event log + versioned reference data (rates, statuses) |

### Is "just a transaction history" enough?

**No.** A transaction history alone (one big log of "customer received $X on date Y") gives you the customer view but doesn't give you the properties an ADI needs. Here's what's missing:

1. **No proof the numbers actually balance across the bank.** If we only log credits into customer accounts, where did the money come from? Double-entry forces us to record the matching debit against the sponsor pool asset — now the whole bank sums to zero and any drift is visible.
2. **No immutability guarantee.** Any ORM will happily let you `UPDATE transaction SET amount = 0 WHERE ...`. AUSTRAC requires 7-year retention of the actual events; APRA requires reproducibility. We need **append-only** with **DB-level protections** (roles that lack UPDATE on the postings table, event stream to S3 Object Lock).
3. **No reconciliation.** The sponsor bank holds the real money in its pool account. If Velo's ledger says customers have $50M total but the sponsor's statement says the pool is $49.7M, there's $300k missing — you cannot detect this from a transaction history alone. You need a **daily reconciliation** that compares the two.
4. **No versioned reference data.** If we changed the interest rate mid-month and don't record the version, we cannot reproduce a customer's historical interest calculation.
5. **No actor attribution.** Regulators want to know **who** made each change and **why**. Not just "the system." Ops actions need `actor_id`, `reason_code`, and a signed audit line.
6. **No segregation of concerns.** A "history" table doesn't distinguish customer money (a liability we owe them) from bank equity from interest expense — regulators need the chart of accounts.

### What "auditable" means concretely

An external APRA/AUSTRAC/EY auditor should be able to say:

1. **"Show me every posting for customer X between date A and date B."** → Query on `POSTING` joined to `LEDGER_ACCOUNT` and `ACCOUNT`. Returns immutable rows with `posted_at`, actor, source (rail), external reference.
2. **"Prove the total customer liability equals the sponsor pool balance yesterday."** → Show the reconciliation run for that date, matched to sponsor's statement file (stored in S3 with Object Lock).
3. **"Prove this credit was authorized."** → Show the original NPP webhook payload (also in S3 with Object Lock), the signature verification result, the idempotency-protected posting.
4. **"Prove no one has silently altered a historical transaction."** → Point at S3 Object Lock in Compliance mode, at the append-only postings table (no UPDATE/DELETE grants), and at the WAL-based CDC that ships every commit off-box.
5. **"Prove the ops user who approved this KYC didn't also submit it."** → Show four-eyes in `APPLICATION_DECISION` (submit + confirm are distinct rows with distinct `reviewer_ops_user_id`s).

So: **transaction history is a byproduct**. Auditability is a *combination* of double-entry, immutability, reconciliation, versioned reference data, actor attribution, and segregation of duties.

---

## 3. Restricted ADI vs Full ADI

### Full ADI

- **Authorised Deposit-taking Institution** with an **unrestricted** license from APRA.
- Can:
  - Take unlimited deposits from anyone.
  - Call itself a "bank" (with a caveat around size for "credit union" vs "bank" branding).
  - Apply directly for scheme membership (BECS, NPP, HVCS) and hold its own **Exchange Settlement account** with the RBA.
- Must:
  - Meet full **APRA prudential standards** — capital ratios (CET1 ≥ 4.5% + other buffers, total ~10.5–15%+ depending on approach), liquidity (LCR, NSFR), governance, risk management (CPS 220), operational risk (CPS 230), info security (CPS 234).
  - Submit regular reg reporting (EFS returns, ARF forms).

### Restricted ADI (RADI)

- Introduced by APRA in **2018** as a "training wheels" license so neobanks could get to market faster.
- **Time-limited to 2 years.** At the end you must either **upgrade to full ADI** or **exit** (return deposits, close down).
- **Restrictions during the RADI phase** (subject to APRA's specific determination on each licensee, so numbers vary):
  - Aggregate deposits capped (~$100M at framework introduction).
  - Per-individual-depositor cap (historically ~$2M, sometimes tightened to align with FCS's $250k depending on APRA's condition-setting).
  - Limited product set.
  - Cannot use the word "bank" without APRA consent (Volt got consent early; Xinja too).
- **Simplified prudential requirements** — still real, but scaled to the size and risk of a small institution.
- **Still an ADI** — deposits are FCS-protected from Day 1.
- Examples of past RADIs: **Volt Bank** (surrendered license 2022), **Xinja** (failed 2020), **86 400** (acquired by NAB and merged into UBank), **Judo Bank** (upgraded to full ADI, now listed).

### What the RADI phase means for your architecture

- Every design must **survive the RADI → Full ADI transition without a rewrite** — because the transition happens during peak growth.
- Your data model must support:
  - Per-depositor exposure calculation (for RADI cap enforcement)
  - Aggregate exposure calculation
  - Ability to reject a new deposit when a cap is hit
- Your ops portal needs:
  - Alerts when caps are approached
  - Reporting exportable to APRA formats
- You must have an **exit strategy** documented under CPS 231, whether "exit" means upgrade or wind-down.

Vietnamese analogy: there isn't a direct one. The State Bank of Vietnam doesn't have a "training wheels bank license" tier. In Australia, RADI exists specifically to make **neobank market entry** feasible in less than a decade.

---

## 4. Ledger mechanics: double-entry, sub-ledger vs GL, idempotency, exactly-once

This is the meat of banking engineering. If you internalize this section you'll be fluent in the interview.

### Double-entry bookkeeping

Invented by Luca Pacioli in Venice, **1494**. Still the mathematical bedrock of banking.

**The rule:** every business transaction is recorded as at least **two** entries — one **debit (DR)** and one **credit (CR)** — that sum to zero for a given currency.

**The intuition (banking-flavoured):**
- A customer's balance at Velo is a **liability** to Velo. Velo owes the customer that money. So "customer has more money" = "our liability increased" = **CR** on the customer's liability account.
- Sponsor's pool account is Velo's **asset** with the sponsor. So "money flowed in from outside" = "our asset increased" = but wait, we also need to record where it *came from*. So we DR the sponsor's pool (increasing asset) and CR the customer's liability (increasing liability). Balance sheet stays balanced.

Under a very simplified convention (using +/− against natural balance):

| Event | Debit account (−) | Credit account (+) |
|---|---|---|
| Customer receives $100 via NPP | Sponsor Pool (Asset)  ▲ $100 → recorded as **DR** on the pool | Customer Deposit (Liability) ▲ $100 → recorded as **CR** on the customer |
| Customer A sends $85 internal to B | Customer A (Liability) ▼ $85 → recorded as **DR** | Customer B (Liability) ▲ $85 → recorded as **CR** |
| Monthly interest to customer | Interest Expense (Expense) ▲ $8.42 → **DR** | Customer Deposit ▲ $8.42 → **CR** |
| Customer's card is declined (fee) | Customer Deposit ▼ $0.50 → **DR** | Fee Income (Revenue) ▲ $0.50 → **CR** |

> Sign convention gotcha: in accounting, "DR" and "CR" don't mean minus/plus. They mean "left side" and "right side" of the T-account. Whether it *increases* or *decreases* the balance depends on the account **class** (asset/liability/equity/revenue/expense). Assets and expenses have natural DR balances; liabilities, equity, and revenue have natural CR balances. Above table uses that convention.

**Why we care in code:**
- Every ledger write inserts **≥ 2 `POSTING` rows** in one DB transaction.
- We enforce **`SUM(amount * sign) = 0`** per transaction per currency at write time (application check + DB trigger).
- The `LEDGER_ACCOUNT.balance` column is a **projection** — we can always recompute it from the postings.
- If sponsor and Velo disagree, we have both sides of the truth: the sponsor's real ES account movements and Velo's postings against the sponsor pool.

### Sub-ledger vs General Ledger (GL)

Two different views of the same money at different levels of granularity.

**General Ledger (GL)**
- The **chart of accounts** view. A small number of account rollups.
- Typical Velo GL lines for MVP:
  - Assets: `Cash at Sponsor Pool`, `Interest Receivable`
  - Liabilities: `Customer Deposits`, `Interest Payable`
  - Equity: `Retained Earnings`
  - Revenue: `Interest Income` (from sponsor / RBA), `Fee Income` (future)
  - Expenses: `Interest Expense` (paid to customers), `Operating Costs`
- Total lines: ~30. This is what auditors and the CFO look at.
- Rolls up into the **balance sheet** and **P&L** (income statement).

**Sub-ledger**
- The **detail** view. Millions of rows.
- The customer sub-ledger: one `LEDGER_ACCOUNT` per customer account. `SUM(all customer sub-ledger balances) = GL "Customer Deposits" balance`.
- The interest accrual sub-ledger: one `INTEREST_ACCRUAL` per account per day. `SUM(all accruals in September) = GL "Interest Expense" September`.
- The reconciliation sub-ledger: one row per external message. `SUM(external credits today) = today's movement in the GL "Cash at Sponsor Pool"`.

**Why separate?**
- Different consumers: customers see sub-ledger; the CFO sees GL.
- Different retention: sub-ledger might get archived after 7 years; GL is forever.
- Different query patterns: sub-ledger is high-cardinality, GL is aggregated.
- Different security: fewer people should be able to query the sub-ledger by customer identity.

In our design: `LEDGER_ACCOUNT.account_class` marks whether a row is a **sub-ledger customer account** (`CUSTOMER`) or a **GL control account** (`SPONSOR_POOL`, `INTEREST_EXPENSE`, `SUSPENSE`, ...). A single postings table serves both. The GL "value" is just `SUM(LEDGER_ACCOUNT.balance WHERE account_class = 'X')`.

### Idempotency on credits

**Definition:** an operation is idempotent if performing it *N* times produces the same result as performing it once.

**Why we care:** the world retries. Networks glitch. Sponsors resend files "just to be safe". Webhook receivers time out and the caller retries. If we're not careful, one $100 payment becomes two, three, or five.

**How we make credits idempotent:**

1. **Compute a deterministic idempotency key** for each intended transaction. The key must be:
   - Deterministic from the *source*, not a random UUID generated at receive time.
   - Unique enough to distinguish two legitimate distinct payments with the same amount.
   - Stable across retries.
   
   Examples:
   - NPP inbound: `key = "NPP|" + sponsor_msg_id` (sponsor's message id is unique per NPP message globally).
   - DE inbound: `key = "DE|" + sponsor_file_id + "|" + sequence_within_file`.
   - Internal transfer: `key = "INT|" + client_generated_uuid_at_button_press` (the app generates one UUID per user tap and re-sends it if the request retries).
   - Interest capitalisation: `key = "INTC|" + account_id + "|" + yyyy-mm`.

2. **Unique constraint at the database level.** `UNIQUE (idempotency_key)` on the `TRANSACTION` table.

3. **Insert-first, react to conflict.**
   ```
   BEGIN
     INSERT INTO transaction (id, idempotency_key, ...) VALUES (...)
       ON CONFLICT (idempotency_key) DO NOTHING RETURNING id;
     IF returning row is NULL:
        -- Duplicate. Load the previous transaction, return its result.
     ELSE:
        -- New transaction. Insert postings, update balances.
   COMMIT
   ```

4. **Return the same response for duplicate calls.** Cache the response body against the idempotency key for a window (Redis, 24 h). If the sponsor retries, they get the identical 200 OK.

This is *the* single most important pattern in payments engineering. If you get it right, everything else is recoverable.

### Exactly-once settlement

The (in)famous phrase. Short version:

> **You cannot achieve exactly-once *delivery* in a distributed system. You *can* achieve exactly-once *effect* by combining at-least-once delivery with idempotent processing.**

**What that looks like in Velo:**

- The sponsor's NPP webhook is retried until they get a 2xx from us. **At-least-once delivery.**
- Our webhook handler dedupes by `sponsor_msg_id` via the ledger's unique constraint. **Idempotent processing.**
- Net effect: for one real-world NPP payment, exactly one customer posting exists in our ledger. **Exactly-once effect.**

Same pattern for DE (file might be re-sent), internal transfers (client might retry on a network timeout), and interest capitalisation (job might be re-run).

**What breaks it:**
- Non-deterministic idempotency keys (using `NOW()` as part of the key).
- Publishing an event to Kafka *before* the DB commit (subscriber acts on an event whose transaction rolled back). Solution: **outbox pattern** — publish only after commit, via CDC.
- Manual "let me just re-run that one" without an idempotency key. Ops portal must always supply keys.

---

## 5. Trade-offs, in depth

Everything I flagged as a trade-off, compared and defended.

### Monolith vs microservices

|  | Modular monolith (our choice) | Microservices |
|---|---|---|
| **Deployment** | One binary per bounded context deployed together in a small number of services | One binary per service, each with own CI/CD |
| **Data** | Shared DB with schema per module (or per service inside monolith) | Database per service; cross-service consistency via events |
| **Consistency** | Cross-module transactions are trivial (one DB txn) | Cross-service consistency is hard — you need sagas, compensations, eventual consistency |
| **Ops burden** | Low: fewer things to run, monitor, deploy | High: service mesh, distributed tracing, rate limiting, circuit breakers, cross-cutting auth |
| **Blast radius** | A bug can touch everything | A bug in one service can't touch another's DB |
| **Team topology** | Fine up to ~30 engineers | Necessary above that |
| **Latency** | In-process function calls (µs) | RPC (ms), often 10-100x slower |
| **Cost** | 1-2x compute, 1x DB | 3-10x compute, N DBs |

**Why "modular monolith with service seams" for MVP?**
- Seed-stage team. Ops burden of true microservices swallows the whole engineering budget.
- We *do* keep some services separate (Payments-In, Ops Portal, Sponsor Adapter) because their security/blast-radius profile is very different — but they're still in one repo.
- The service seams are drawn along future microservice boundaries. When Payments-In outgrows the monolith, we split it out with a two-week project, not a two-year rewrite.

### Managed vs self-hosted for data services

|  | Managed (Aurora, MSK, ElastiCache — our choice) | Self-hosted on EC2/EKS |
|---|---|---|
| **Setup** | Days | Weeks |
| **Backups + HA + failover** | Included, tested by AWS | You engineer it, page yourself at 3am |
| **Patching + minor version upgrades** | AWS handles maintenance windows | You handle |
| **Cost** | Higher $ (~40-60% premium) | Cheaper if you have SRE time |
| **Lock-in** | Aurora-specific SQL is minimal; MSK is Kafka-standard; ElastiCache is Redis-standard | Lower |
| **Compliance** | AWS shared responsibility; APRA CPS 231 needs due diligence on AWS but the docs are pre-baked | Your problem entirely |

**Why managed:** at seed stage, engineer time is worth more than the AWS premium. When the AWS bill dominates payroll, that's a different conversation.

### Kafka (MSK) vs SNS/SQS

|  | Kafka on MSK (our choice) | SNS + SQS |
|---|---|---|
| **Semantics** | Durable, ordered log; replayable from any offset | Pub/sub or queue; consumers see each message once (SQS) or fan-out (SNS) |
| **Consumers** | Multiple consumer groups; each group has its own offset | Fan-out via SNS→SQS; each queue is independent |
| **Replay** | Native — set offset to earlier point and re-consume | Not supported; need to re-publish |
| **Ordering** | Per-partition guaranteed | FIFO SQS provides ordering but low throughput |
| **Ops burden** | High: brokers, ZK/KRaft, topics, ACLs | Very low: fully serverless |
| **Cost floor** | Several $/day even idle (brokers) | ~$0 idle |
| **Throughput ceiling** | Very high (millions/sec) | Millions/sec, but structured differently |

**Why Kafka for us:** the audit lake + future consumers (fraud, AML, CDR) need **replay** and multiple **independent consumers** on the same event stream. SNS/SQS makes each consumer a distinct queue that you have to backfill separately — painful for compliance.

**Where we still use SQS:** per-channel notification queues (SES, SMS, push), the SFTP-file-arrived notification. Simple point-to-point, no replay needed.

### PostgreSQL vs purpose-built ledger DB (TigerBeetle, QLDB)

|  | PostgreSQL / Aurora (our choice) | TigerBeetle | AWS QLDB |
|---|---|---|---|
| **Focus** | General purpose OLTP | Purpose-built financial ledger | Immutable, cryptographically verifiable ledger |
| **Throughput** | ~10k–50k txn/sec (with tuning) | Millions of txn/sec (design goal) | Modest |
| **Correctness** | Serialisable isolation, MVCC | Deterministic replicated state machine, strong consistency by design | Verifiable Merkle tree over history |
| **Hireability** | Every engineer knows Postgres | Very few engineers | Few |
| **Managed offering** | Aurora, RDS | Not on AWS as managed yet | Yes, but AWS deprecated it recently |
| **Query flexibility** | Full SQL | Very limited (only debits/credits/accounts) | Limited (PartiQL) |
| **Ecosystem** | Massive | Small | AWS ecosystem |

**Why Postgres for MVP:** boring, deep, understood, transactional. At Velo's projected MVP throughput (< 10 txn/sec sustained), Postgres is comfortably 3 orders of magnitude over-provisioned. As we scale, we can put **TigerBeetle** in front as the hot posting engine and keep Postgres as the reporting/reference DB — but that's a Year-2 problem.

### Cognito vs Auth0 vs custom identity

|  | Cognito (our choice) | Auth0 | Roll our own |
|---|---|---|---|
| **Cost** | Cheapest at scale | $$$ per MAU | Zero infra, huge engineering |
| **Deep AWS integration** | Native (IAM, API Gateway) | Bolt-on | Bolt-on |
| **Developer experience** | Historically clunky | Excellent | Whatever you build |
| **Feature depth (MFA, social, SSO, MFA policies)** | Solid, opinionated | Best-in-class | Whatever you build |
| **Compliance certifications** | AWS certifications inherited | Own certifications | You get them yourself |
| **Vendor risk** | AWS-locked but stable | Okta acquisition dependence | Zero vendor risk, huge implementation risk |

**Why Cognito:** cost + AWS integration matter more than DX at seed. If DX becomes a real blocker post-MVP, migrating to Auth0 is a compartmentalised change (Identity service is behind an interface).

### PWA + React Native vs native iOS + Android

|  | PWA + React Native shared codebase (our choice) | Two native codebases |
|---|---|---|
| **Team size** | 2–4 engineers | 4–8 (iOS team + Android team + shared BE) |
| **Time to market** | Fast | Slower |
| **UX quality** | Very good | Best |
| **Access to native features** | Solid via RN + expo | Native APIs directly |
| **Bank-grade security features** | RN has good stories for keystore, biometrics, jailbreak detection | Native has best |
| **App store presence** | Yes (RN builds to real .ipa/.apk) | Yes |

**Why RN + PWA:** the underbanked segment lives on Android in Australia; a great RN app is a much better use of seed capital than half-finished native apps. Once we have retention data proving native-quality matters, we can swap module-by-module.

### Sponsor bank abstraction (anti-corruption layer) vs direct integration

|  | Anti-corruption layer (our choice) | Direct sponsor SDK calls sprinkled across services |
|---|---|---|
| **Sponsor change** | Rewrite one adapter | Rewrite everything |
| **Testing** | Mock the adapter interface | Mock the sponsor across every service |
| **Vendor coupling** | Contained | Diffused |
| **Extra code** | ~1 service worth | Zero |

**Why ACL:** sponsor changes do happen (Xinja's sponsor exited; Cuscal is being bought by Pemba). The cost of one adapter service is negligible compared to a full rewrite.

### Kafka outbox pattern vs dual write

|  | Outbox + CDC (our choice) | Dual write: `db.commit()` then `kafka.publish()` |
|---|---|---|
| **Consistency** | Exactly-once effect on the ledger | Two-phase failure: db committed but kafka not published → downstream never notified |
| **Ordering** | Preserves DB commit order | Not necessarily |
| **Complexity** | Requires CDC (Debezium or similar) | Simpler code |
| **Latency** | ~sub-second additional lag | Milliseconds |

**Why outbox:** in banking, "we credited the customer but never notified them / never told the fraud consumer" is a bug we cannot ship.

---

## 6. PII and Australian data residency

### PII (Personally Identifiable Information)

Any data that identifies a person **on its own** or **in combination with other data**:

| PII (obvious) | Sensitive PII | Not obvious PII |
|---|---|---|
| Full name | Tax File Number (TFN) | IP address (in some contexts) |
| Email | Medicare number | Device fingerprint |
| Mobile | Passport number | Precise geolocation |
| Address | Driver's licence | Biometric templates |
| Date of birth | Health info | Behavioural patterns (transaction habits) |
| Government IDs | Racial/ethnic origin | Photos of the person |

In Velo, most `CUSTOMER` columns are PII, uploaded KYC docs are **highly sensitive** PII.

### Australian data residency

Two different concepts that often get conflated:

- **Data residency**: the data is *physically stored* in Australia. Achieved by choosing an Australian AWS region (`ap-southeast-2 Sydney`, `ap-southeast-4 Melbourne`) and disabling cross-region replication to non-AU regions.
- **Data sovereignty**: the data is subject to *Australian law only*. Harder — even if data sits in Sydney, the US CLOUD Act theoretically allows US authorities to demand it from AWS. Mitigation: **customer-managed encryption keys** (KMS-CMK) so AWS can't decrypt without you.

### Does this mean "keeping correct track of resident data in Australia"?

Not quite — it's less about *tracking who is resident* and more about **making sure PII and financial data never leaves Australian soil**. Concretely:

1. **AWS region:** primary in Sydney, DR in Melbourne. No `us-east-1` anywhere.
2. **S3 replication:** cross-region replication limited to AU regions only.
3. **DynamoDB Global Tables:** disabled (would replicate to other regions).
4. **CDN caching:** CloudFront caches static assets globally, but *not* dynamic API responses containing PII. Cache-Control headers enforced.
5. **Third-party vendors:**
   - Email (SES) — has an AU region, use it.
   - SMS (Twilio) — has AU numbers/routes; contract clauses require message content stays in AU.
   - Push notifications (APNs/FCM) — cross-border but content is minimal ("You received $X"), no full PII.
   - Observability (Datadog / Grafana Cloud) — must use their AU region, and PII is scrubbed at emit time.
   - Analytics — don't send raw PII to Google Analytics / Mixpanel; use hashed IDs.
6. **Backups:** stored in AU regions only.
7. **Support tools:** support ops uses tooling that runs in AU (no US-hosted Zendesk without an AU option).
8. **Vendor DPAs:** every processor signs a data-processing agreement covering AU residency and breach notification.

### Legal underpinnings

- **Privacy Act 1988 (Cth)** + the **Australian Privacy Principles (APPs)** — 13 principles covering collection, use, disclosure, cross-border transfer (APP 8 makes the sender accountable for what happens overseas), access, correction, security.
- **APRA CPS 234** — requires info security capability commensurate with the size and risk of the ADI.
- **APRA CPS 231** — outsourcing (using AWS, Twilio etc.) requires formal risk assessment and material-outsource documentation.
- **Notifiable Data Breaches (NDB) scheme** under the Privacy Act — must notify OAIC and affected individuals of eligible data breaches.

### PII minimisation and lifecycle

Beyond residency:
- **Collect only what you need.** MVP asks for name, DOB, address, occupation, ID — no TFN unless we're paying interest that needs withholding.
- **Encrypt at column level for the most sensitive fields.** `full_name`, `dob`, `mobile` in `CUSTOMER` encrypted with pgcrypto + KMS.
- **Retain only as long as required.** AUSTRAC requires ~7 years after account closure; APPs require deletion after that.
- **Right to access** (DSAR): a customer can request all data held about them; must be answerable via a single SQL by `customer_id`.
- **Right to correction**: fix stale data; audit the correction.

---

## 7. Security terms

### TLS 1.2+ (Transport Layer Security)

- **What:** encrypts data **in transit** between two parties (client ↔ server, server ↔ server).
- **Version:** TLS 1.2 minimum, TLS 1.3 preferred. **Never TLS 1.0/1.1** (banned by PCI DSS since 2018) and **never SSL** (deprecated 1996+).
- **Where we use it:**
  - Customer app ↔ CloudFront (HTTPS)
  - CloudFront ↔ API Gateway (HTTPS)
  - Service ↔ service (mTLS optionally)
  - Service ↔ Aurora (TLS enforced, `sslmode=require`)
  - Service ↔ Sponsor bank webhook (mTLS — both sides present certs)
  - SFTP ↔ Sponsor bank SFTP (SSH — different protocol, same idea)
- **mTLS (mutual TLS):** both sides authenticate with certificates. Used where an API is high-value and low-cardinality of callers — the sponsor's NPP webhook is the poster child.
- **Certificate management:** ACM for public certs, ACM Private CA for internal, quarterly rotation.

### KMS-CMK (Key Management Service, Customer-Managed Key)

- **KMS** = AWS Key Management Service. A hardware-security-module-backed key management service.
- **Encryption at rest:** all persistent data is encrypted on disk. Aurora, RDS, S3, EBS, Secrets Manager — all use KMS.
- **CMK (Customer-Managed Key) vs AWS-Managed Key:**
  - **AWS-managed key**: AWS controls the key policy; you can't restrict who at AWS can (in principle) access it; can't audit granularly.
  - **Customer-managed key**: you write the policy, you control rotation, you can delete it (bricks the data — powerful).
- **Envelope encryption:** the actual data is encrypted with a per-object **data key**; the data key is encrypted with the **CMK**. Only the encrypted data key travels with the object; KMS decrypts the data key on request. Benefit: KMS never sees plaintext data; API calls are small.
- **Key hierarchy for Velo:**
  - `kms/velo/data-general` — bulk data (Aurora, S3 audit lake, EBS).
  - `kms/velo/data-kyc` — KYC documents; policy restricted to Onboarding + Ops Portal service roles only.
  - `kms/velo/data-ledger` — extra key for column-level encryption on PII columns.
  - `kms/velo/data-secrets` — Secrets Manager.
- **Rotation:** enabled automatically (annual by default).
- **Grants and policies:** IAM + resource-based; the "ledger writer" role has no grant on the KMS key for KYC docs — a compromised ledger service cannot read customer ID photos.

### MFA (Multi-Factor Authentication)

- **Definition:** authentication using ≥ 2 of the 3 factor classes:
  - **Something you know** — password / PIN.
  - **Something you have** — phone (SMS OTP), hardware token, FIDO2 key.
  - **Something you are** — fingerprint, face ID, other biometric.
- **Customer MFA at Velo:**
  - **Signup:** email + SMS OTP verification.
  - **Login:** password + device-bound biometric (Face ID / fingerprint) after first login on that device.
  - **Sensitive actions (step-up):** biometric confirmation for large transfers, adding a new payee, changing PayID, closing account. This is a **step-up**, not a second full login.
- **Ops MFA:**
  - Cognito user pool with mandatory MFA (hardware key preferred, TOTP fallback).
  - Enforced at both the identity layer *and* the Verified Access proxy.
- **Why SMS OTP is deprecated for high-value flows:** SIM-swap attacks. Keep SMS for signup verification, don't use it for step-up.

### WAF (Web Application Firewall)

- **What:** a filter between the internet and your API. Inspects each HTTP request; blocks or challenges bad ones.
- **AWS WAF (attached to CloudFront + API Gateway):**
  - **Managed rule groups:** AWS Managed Rules for the OWASP Top 10 (SQLi, XSS, path traversal), Anonymous IP list, Bot Control.
  - **Rate limiting:** e.g. 100 req/5min per IP on login, 10 req/min on OTP-resend.
  - **Geo-blocking:** for MVP, disallow requests from countries with no Velo customers (defence-in-depth; also compliance signal).
  - **Custom rules:** block requests missing our custom `x-app-version` header (breaks basic bots).
- **Where WAF stops and app-level defence begins:** WAF handles the noisy, generic layer 7 attacks; app code handles business-logic attacks (idempotency abuse, credit-stacking, race conditions).

### Segregation of Duties (SoD) in the Ops portal

- **Principle:** no single person can complete a sensitive action end-to-end.
- **APRA CPS 220** (Risk Management) and **CPS 230** (Operational Risk) essentially demand this for financial institutions.
- **How we enforce it:**
  - **Role separation** in Cognito user pool groups:
    - `OPS_REVIEWER` — can review a KYC application, propose a decision.
    - `OPS_APPROVER` — can confirm/reject a reviewer's proposal (cannot self-approve their own).
    - `OPS_ADMIN` — user management, role assignment (never touches customer money).
    - `TREASURER` — proposes rate changes.
    - `TREASURY_APPROVER` — approves rate changes.
  - **Four-eyes at the application layer:** `APPLICATION_DECISION` has an `is_confirmation` flag; the confirming decision must have a different `reviewer_ops_user_id` than the first decision. Enforced in code + DB check constraint.
  - **Immutable audit log:** every ops action writes an `AUDIT_LOG` row (actor, action, before/after, IP, user agent, timestamp). Append-only, cannot be edited by the ops user themselves.
  - **Break-glass procedure:** in true emergencies, `OPS_ADMIN` + `SECURITY_ADMIN` can jointly grant temporary elevated access, logged at highest priority, retro-reviewed within 24 h.
- **Physical/logical separation of the money mover:** ops users cannot post directly to the ledger; they can only *approve* reversal/hold/freeze actions that flow through the same Ledger service with an idempotency key.

---

## 8. Velo vs ZaloPay / MoMo

Great question — this is the mental-model gap that trips up most non-AU engineers.

### The essence: e-wallet vs bank

**MoMo, ZaloPay, VNPay** in Vietnam are **e-wallets**, regulated as **Trung gian thanh toán** (Payment Intermediaries) by the State Bank of Vietnam (SBV). **Velo is a bank** (an ADI), regulated by APRA + AUSTRAC + ASIC + OAIC in Australia.

The differences are fundamental, not cosmetic:

| Dimension | MoMo / ZaloPay (e-wallet) | Velo (ADI) |
|---|---|---|
| **License class** | Payment Intermediary (SBV) | Authorised Deposit-taking Institution (APRA) |
| **What the balance legally is** | **Stored-value / prepaid balance** at a licensed intermediary | **A deposit** — a legal debt from the bank to you |
| **Deposit insurance** | None (customer money sits in an escrow at a partner bank; if the wallet fails, recovery is via the escrow, complex) | Yes — **FCS up to $250k per depositor per ADI** |
| **Interest paid** | Generally no (SBV historically forbade it; some interest-like products through partners) | Yes — Velo's core value proposition |
| **Own account number** | No — you have a wallet ID; you top up from your linked bank account | Yes — real BSB + account number, receive from anyone |
| **Direct access to national payment rails** | No — routes through partner banks and Napas | Yes — via the sponsor for now, direct later |
| **Can be paid salary directly** | Not natively — salary hits your bank account then you top up | Yes — payroll credits into your Velo BSB + account |
| **Capital / prudential requirements** | Modest — designed for payments, not banking | Full APRA prudential standards (scaled during RADI) |
| **Product breadth** | Payments, top-ups, bill pay, some savings-adjacent products via partners | Deposits, later cards, loans, savings — a full retail bank |
| **Regulator focus** | Fraud, KYC tiers, escrow safety | Solvency, liquidity, capital, deposit protection, AML/CTF |

### What Velo *does* share with MoMo / ZaloPay

- Mobile-first UX.
- Digital onboarding (though Velo currently does manual KYC).
- Real-time UX for money movement (Velo uses NPP; MoMo uses Napas 247 through partner banks).
- Push notifications, in-app history.
- API integrations with merchants and services (roadmap for Velo).

### The closest Vietnamese analog to Velo

Not MoMo/ZaloPay. Look at:
- **Cake by VPBank** — digital-first bank (but backed by a full universal bank).
- **TNEX by MSB** — same pattern.
- **Timo** — digital-first bank, historically a joint venture.
- These are the pattern: **a digital brand riding on a licensed bank's balance sheet**. Velo is trying to become that licensed bank itself, not just a brand.

There is **no direct Vietnamese analog** to the RADI pathway. In VN you either need a full commercial bank license (extremely hard to get) or you settle for the payment-intermediary license — no in-between.

### Why this matters for the interview

If the panel senses you're mentally modelling Velo as "an e-wallet with better UI", you'll get the *ledger* wrong, the *reg-comp posture* wrong, and the *money flows* wrong. Model Velo as **a bank that happens to be digital-first, using a bigger bank as its temporary on-ramp**, and everything else follows.

---

## 9. Sponsor bank

### Why we need one

To move money in and out of Velo customers' accounts, we need access to **BECS**, **NPP**, **PayID**, and (indirectly) **RITS** — plus an **Exchange Settlement (ES) account at the RBA** to actually hold central-bank money.

Getting direct access requires:

- **Full ADI status** (RADI does not grant direct scheme membership).
- **Direct participation in NPPA** — capital + ops maturity requirements, currently ~13 direct participants.
- **BECS Framework membership** — Framework Participant (settles at RBA) vs Non-Framework (settles via another participant).
- **RITS account** — RBA membership.

Standing all of this up **takes years**, not months, and requires a mature operations team we won't have on day one.

A **sponsor bank** is an existing Full ADI that already has all of the above and lets us use their pipes for a fee. Common Australian sponsors: **Cuscal** (specialises in this), **Indue**, historically **Volt** and others.

### What the sponsor provides

| Service | Detail |
|---|---|
| **BSB range** | Either a BSB owned by the sponsor (all Velo customers share it, e.g. `802-985`), or a dedicated BSB assigned to us within their allocation |
| **Account-number range delegation** | We issue our own account numbers within an agreed range; sponsor knows the range is "Velo's" |
| **Pool (trust) account at the sponsor** | One big account in Velo's name holding all customer deposits collectively; the sponsor's own ledger has this as a single liability line |
| **BECS/DE ingestion** | Sponsor receives all DE credits addressed to Velo's BSB + account range; forwards to us as a file |
| **BECS/DE egress (roadmap)** | We instruct sponsor to lodge a DE file; sponsor submits to BECS on our behalf |
| **NPP inbound** | Sponsor receives NPP messages addressed to us; calls our webhook synchronously; awaits accept/reject; issues `pacs.002` back to NPP |
| **NPP outbound (roadmap)** | We call sponsor's API; sponsor submits `pacs.008` to NPP; settles on RBA FSS |
| **PayID directory** | Sponsor registers/deregisters aliases in the national PayID directory on our behalf |
| **HVCS / batch statements / positions** | Sponsor gives us EOD balance + activity reports for reconciliation |
| **Compliance oversight** | Sponsor has veto over our onboarding (they don't want dirty money in *their* pool account) |

### How Velo interacts with the sponsor

Two integration surfaces:

1. **SFTP + PGP** for batch flows (DE files inbound, statements/positions inbound, DE files outbound in roadmap).
2. **REST + mTLS webhooks** for real-time flows (NPP inbound webhook, PayID mgmt, NPP outbound API in roadmap).

We put a **single Sponsor-Bank Adapter service** in front of both to keep the rest of Velo unaware of the sponsor's protocols and quirks. If the sponsor changes API v1 → v2, only the adapter changes.

### The money picture, precisely

Think of it as a two-tier ledger:

```
                         RBA ES account (real central-bank money)
                                  │
                                  │   sponsor holds this
                                  ▼
              ┌──────────────────────────────────────────────┐
              │  Sponsor Bank's own core banking             │
              │                                              │
              │   Velo Pool Account: $50,000,000 (liability │
              │   from sponsor's perspective; asset from    │
              │   Velo's perspective)                        │
              └──────────────────────────────────────────────┘
                                  ▲
                                  │   Velo's ledger tracks
                                  │   who owns what share
                                  │
              ┌──────────────────────────────────────────────┐
              │  Velo's ledger (sub-ledger)                  │
              │                                              │
              │   Alice: $12,000                              │
              │   Bob:   $3,450                               │
              │   ... 5,000 customers ...                     │
              │   Sum: $50,000,000                            │
              └──────────────────────────────────────────────┘
```

- The sponsor sees **one balance** (the pool).
- Velo sees **many balances** (each customer).
- **They must always sum equal**. Daily reconciliation asserts this. A drift = a **regulatory event**.

### What happens on each MVP flow

- **Inbound DE**: sponsor receives money into pool → pool balance ↑ → we credit customer sub-ledger → both sides balance.
- **Inbound NPP**: same, but real-time via webhook.
- **Internal transfer Velo → Velo**: pool balance **unchanged** (money didn't leave the sponsor); Velo shifts balance from Customer A's sub-ledger row to Customer B's. **Sponsor sees nothing**.
- **Outbound DE/NPP (roadmap)**: we instruct sponsor → pool balance ↓ → we debit customer sub-ledger → external bank receives.
- **Interest** (roadmap on the sponsor side): sponsor pays Velo interest on the pool at wholesale rate; Velo pays customers interest at retail rate; the spread is Velo's margin.

### Failure modes

| Failure | Impact | Mitigation |
|---|---|---|
| Sponsor NPP webhook outage | Inbound NPP payments to Velo fail | Sponsor retries; customers see delay; internal transfers unaffected |
| Sponsor SFTP outage | DE ingestion delayed | Same-day catch-up on restore; recon detects |
| Sponsor pool position ≠ Velo ledger sum | **Regulatory event** | Daily recon detects immediately; Ops + treasury investigate; APRA notified if material |
| Sponsor decides to exit / gets acquired | Existential | CPS 231 exit plan documents 6–12 month migration to another sponsor or full ADI upgrade |
| Sponsor compromised (fraud/breach) | Blast radius includes Velo pool | Sponsor is a documented material outsource; contractual liability + insurance; incident-response runbook |

### Compliance framing

- **APRA CPS 231** treats the sponsor as a **material service provider** — formal due diligence, ongoing monitoring, board oversight, and a documented **exit plan**.
- **APRA CPS 234** requires info-security assessment of the sponsor (their vulnerabilities are your vulnerabilities).
- **AUSTRAC** — the sponsor is a **reporting entity**; they have their own AML program; ours must be consistent with theirs where flows cross.
- **The customer** doesn't see the sponsor at all. Their BSB is either the sponsor's or Velo's, either way it's "their Velo account".

### One-liner to remember

> "The sponsor bank is Velo's temporary on-ramp to the plumbing of Australian retail banking. We hold the customer relationship, the ledger truth, and the product. The sponsor holds the money at the RBA and holds our scheme memberships until we get our own."
