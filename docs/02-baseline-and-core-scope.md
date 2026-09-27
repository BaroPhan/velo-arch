# 02 — Baseline (what's done) & Core Scope (what you build on)

## Part A — What has already been accomplished (given to you)

These are stated in the brief. **You don't design or build these**, but every design decision has to respect them.

| # | Accomplished | Why it matters to your design |
|---|---|---|
| 1 | **Seed funding secured** | You have runway but not big-bank runway — pick managed services over self-hosted; avoid vendor lock-in only where it's cheap to avoid |
| 2 | **APRA Restricted ADI pathway established** | Deposit caps (~$2M per depositor, $100M aggregate) → your account model must support per-account and aggregate exposure checks; the RADI status expires (2 years) → the platform must survive an ADI upgrade with schema evolution, not rewrite |
| 3 | **Sponsor bank partnership signed** | You are *not* a direct clearer on BECS/NPP/HVCS. All external payments enter/leave through the sponsor bank's rails; your customer BSB will typically be sponsor-owned; your ES (Exchange Settlement) account with RBA is effectively the sponsor's for now |
| 4 | **Scheme memberships in place** (via sponsor) | BECS (AusPayNet BECS framework), NPP (NPPA), PayID, likely Osko — all accessible **through** the sponsor bank |
| 5 | **Founders + regulatory relationships** | Amy (core banking / CDR) and Daniel (AUSTRAC / AML/CTF) — the domain knowledge is on the founding team, not something you have to source |

**Implication:** the boundary between "our platform" and "the rest of the Australian financial system" is drawn at **the sponsor bank's API surface**. That is the single most important integration point in your architecture and deserves its own anti-corruption layer.

---

## Part B — The core business you must build on

The brief is unambiguous: **one product, four capabilities**.

### The product
**Velo Transaction Account** — a zero-fee everyday deposit account, competitive interest on positive balances, FCS-protected.

### The four capabilities (MVP)
1. **Customer Onboarding** — registration + offline (manual) ID verification by the Ops team.
2. **Inbound Deposits — Direct Entry (DE)** via BECS through the sponsor bank.
3. **Inbound Deposits — NPP / PayID** via NPP through the sponsor bank.
4. **Internal (intra-Velo) Transfers** — real-time, immediate funds availability.

Everything you build in the MVP is one of these four capabilities plus the **cross-cutting foundations** they require:

### The cross-cutting foundations (implicit but required)
The brief lists MVP features but does not enumerate the invisible plumbing they demand. You must call these out explicitly because they are what makes the MVP *safe*, not just functional.

| Foundation | Why it must exist in MVP | Where it lives |
|---|---|---|
| **Double-entry ledger** | You are a bank — every credit must have a matching debit against a controlled account (customer liability vs sponsor pool asset); balances must be reconstructable | Ledger service (core) |
| **Customer identity + AuthN/AuthZ** | App login, MFA, session management, device binding | Identity service / Cognito |
| **Ops portal with RBAC & segregation of duties** | Manual KYC review, ID doc handling, exception queues, refunds/reversals | Ops portal (internal web app) |
| **Sponsor bank integration adapter** | One place that talks SFTP for DE files and REST/webhooks for NPP; hides the sponsor's schema | Anti-corruption layer service |
| **Notifications** | Onboarding decisions, inbound deposit alerts, transfer confirmations | Notifications service (email + push + SMS) |
| **Reconciliation** | Daily match of sponsor bank's statements/positions vs Velo's ledger — non-negotiable for an ADI | Reconciliation job + break queue |
| **Immutable audit trail + event log** | Every state change is an event; retained ≥ 7 years (AUSTRAC) | Kafka topic → S3 with Object Lock |
| **Secrets, keys, and encryption** | Envelope encryption for PII (ID docs, TFN if captured), KMS CMKs, IAM boundaries | AWS KMS + Secrets Manager |
| **Observability** | Metrics, structured logs, distributed traces, alerting on money movement anomalies | CloudWatch + OpenTelemetry + Grafana/Datadog |
| **CI/CD + change management** | Reproducible builds, approvals for prod, DB migration gates | GitHub Actions + Terraform + Flyway/Prisma migrations |

### The regulatory frame (design *around*, don't implement in full)
| Regulation | MVP-visible impact |
|---|---|
| **APRA CPS 234** (Info Security) | Encryption, key management, IAM boundaries, incident response runbook, third-party security assessment on sponsor bank + all vendors |
| **APRA CPS 231** (Outsourcing) | Cloud provider (AWS) documented as a material outsource; DR plan; exit strategy |
| **APRA CPS 220** (Risk Management) | Risk register, change management, segregation of duties in ops portal |
| **AUSTRAC / AML-CTF** | Customer ID retained, all transactions captured as events, ability to hold/freeze accounts, hooks for TTR/SMR/IFTI (built later) |
| **Privacy Act / APPs** | PII minimisation, consent, data-subject access request (DSAR) capability, breach notification |
| **ePayments Code** | Clear transaction records, dispute handling, unauthorised transaction process — hooks now, workflow later |

---

## Part C — The line between "core scope" and "roadmap"

The brief explicitly lists roadmap items. You must **build the seams** for them without building them.

| Roadmap item | What "not blocking it" means in your MVP design |
|---|---|
| **Automated KYC/KYB** | Onboarding state machine has a pluggable `verification_provider` step; today it's `manual`, tomorrow it's `frankieone`/`greenid` |
| **External transfers (outbound DE + NPP)** | Payments service has a symmetric `payments-in` and `payments-out` module split; ledger already handles debits + credits |
| **Transaction enrichment** | Every `posting` emits an event on Kafka; an enrichment consumer can annotate without touching the ledger |
| **Fraud monitoring** | Every money-movement event carries device/session/geo context; a rules engine (e.g. Feedzai / in-house) subscribes to the same events |
| **AML/CTF reporting (TTR/SMR/IFTI)** | Events are retained ≥ 7 years in S3 with Object Lock; reporting is a downstream batch consumer |
| **Back-office ops portal (full)** | Ops portal MVP already uses RBAC + segregation of duties + audited action log — expanding it is just more screens |

**Rule of thumb:** if a roadmap feature would force a schema change to a MVP table, you got the MVP model wrong. Re-do it.
