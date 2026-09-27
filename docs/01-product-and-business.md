# 01 — Product & Business (high-level)

## The product in one paragraph

**Velo** is a next-generation, cloud-native **digital bank** aimed at the **underbanked Australian market**. It is being built from the ground up as an **Authorised Deposit-taking Institution (ADI)** under APRA's **Restricted ADI** pathway, using a **sponsor bank** for settlement and clearing while it stands up its own capability. The MVP is a **single-product bank** — a zero-fee everyday transaction account with interest on positive balances — delivered through a mobile-first digital experience.

## The business context

| Dimension | Detail |
|---|---|
| Market | Australian retail, "underbanked" segment (people poorly served by Big Four) |
| Regulator | **APRA** (prudential), **AUSTRAC** (AML/CTF), **OAIC** (Privacy Act / APPs), **ASIC** (ePayments Code) |
| Licensing posture | **Restricted ADI** (RADI) — 2-year window with deposit caps (~$2M per depositor, $100M aggregate) and mandatory transition to full ADI |
| Funding | Seed round secured |
| Distribution | Direct-to-consumer via native mobile app + web |
| Business model (MVP) | Interest margin on customer deposits (float held at sponsor bank / RBA ES account) — **zero fees to the customer** |
| Guarantee | Deposits protected under the **Financial Claims Scheme (FCS)** up to $250k per account holder per ADI |

## The founders (what they optimise for)

- **Amy (CEO)** — ex-Head of Digital Transformation at a Big Four. Core-banking modernisation, **Open Banking / CDR**. She will care about: extensibility, event-driven architecture, clean domain boundaries, CDR-readiness, avoiding a legacy core-banking millstone.
- **Daniel (COO)** — ex-compliance director at a payments scheme. **AUSTRAC**, **AML/CTF**, cross-border compliance. He will care about: auditability, immutable ledger, transaction monitoring hooks, RBAC & segregation of duties, evidence retention, SAR/TTR/IFTI reporting pipes.

Design decisions should trace back to at least one of these two people's concerns.

## The high-level requirement (what "success" looks like)

At the end of the MVP, a founding Velo customer can:

1. **Sign up** in the app, submit ID, and be manually approved by the Ops team.
2. **See a Velo Transaction Account** with a real BSB + account number (issued via the sponsor bank's BSB range).
3. **Receive money** into that account from any Australian bank — via **Direct Entry (BECS)** batch credits *and* via **NPP/PayID** near-real-time credits.
4. **Send money instantly** to another Velo customer (internal transfer) with immediate funds availability.
5. **See the resulting balance and transaction history** in-app in real time.
6. **Earn interest** on positive balances, accrued daily, credited monthly.
7. **Trust** that the ledger is correct, auditable, and that deposits are FCS-protected.

Everything else (outbound DE/NPP, cards, joint accounts, KYB, fraud, transaction enrichment) is explicitly **out of MVP scope** but must not be architecturally excluded.

## Non-functional posture (what the architecture must respect)

| NFR | Target for MVP | Why |
|---|---|---|
| **Availability** | 99.9% for customer app; 99.95% for ledger APIs | Below true "big bank" SLA but honest for RADI; scale up per roadmap |
| **Data residency** | All PII and financial data resident in **Australia** | APRA CPS 234 / CPS 231; regulator + customer expectation |
| **Auditability** | Every ledger event is immutable, append-only, and reproducible from source events | AUSTRAC + APRA; Daniel's #1 concern |
| **Security** | Encryption in transit (TLS 1.2+) and at rest (KMS-CMK); MFA on customer login; WAF; segregation of duties in ops portal | CPS 234 |
| **Recoverability** | RPO ≤ 5 min, RTO ≤ 4 h for the ledger | APRA CPS 232 Business Continuity (implied) |
| **Extensibility** | Roadmap features (KYC, outbound payments, fraud) plug in via events, not schema changes | Founder alignment |
| **Cost** | Runs on ~seed budget until scale demands more | It's seed-funded; no big-bank cheque book |
| **Compliance-by-default** | Every write path emits a domain event that is capturable for AML/reporting | AUSTRAC hooks even before AUSTRAC features ship |

## What the panel is really testing (and how to signal it)

The brief openly states: *"This isn't about finding a single correct answer. We want to understand how you think..."*. So signal:

- **You understand Australian retail banking rails** — BECS, NPP, PayID, RITS, sponsor-bank pattern, RADI constraints. This is domain literacy.
- **You know what a bank's data model actually needs** — double-entry ledger, sub-ledger vs GL, idempotency on credits, exactly-once settlement.
- **You cut scope ruthlessly and honestly** — MVP is *four* features, not eight; you name the deferrals.
- **You separate "what I built" from "what I made non-blocking"** — the roadmap should be reachable from your architecture without a rewrite.
- **You have opinions on trade-offs** — monolith vs microservices, self-hosted vs managed, sponsor-bank abstraction, Kafka vs SNS/SQS.
