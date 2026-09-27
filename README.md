# Velo — System Design Interview Prep

Prep pack for the **Automation Consulting — Senior Software Engineer** System Design Interview based on the brief `Velo_System_Design_Interview_Brief_Updated.pdf`.

This repo is a set of markdown documents you can review, edit, and lift into your submission email. Nothing runs here — it's a thinking/decision log plus diagrams (Mermaid).

---

## 1. What the PDF actually asks for

### Format
- **Take-home prep + AI-led video interview** (~45 minutes).
- **Prep time:** 2 days from receipt of the brief.
- **Submission deadline:** by email **1 day before** the scheduled interview.
- **Send to:** `tech@automationconsulting.com.au`
- **Subject line:** `[Your Full Name] - Velo Senior Engineer Interview Submission`
- The AI interviewer platform **does not support file uploads on the day**, hence the pre-submission.

### On-the-day discussion areas (what the AI will explore)
The brief tells you to be ready to present and defend:
1. **Approach** — how you broke the problem down, constraints identified, assumptions made.
2. **High-level architecture** — services, components, external integration points.
3. **Data model** — customers, accounts, balances, transactions (+ ERD).
4. **Technology choices** — cloud provider, frontend framework, backend framework, database, architectural style (monolith vs microservices).
5. **CI/CD pipeline and testing strategy.**
6. **Trade-offs** — what you accepted, what you deferred, and why.

### Required pre-submission deliverables (by email)
| # | Deliverable | Notes |
|---|---|---|
| 1 | **Entity Relationship Diagram (ERD)** | Customers, accounts, balances, transactions |
| 2 | **High-Level Architecture Diagram** | Services, components, external integration points |
| 3 | **Sequence Diagram** | End-to-end flow for **one** key MVP feature (e.g. inbound NPP payment, internal transfer, or customer onboarding) |
| 4 | **AI Transcript** (if applicable) | Include if you used an AI assistant during prep |
| 5 | **Supporting notes / materials** | Anything else you want the panel to have as context |

> The brief encourages AI use — you must **declare it at the start of the interview** and be ready to explain any part of your design in your own words.

---

## 2. What you're being asked to design (one-liner)

Design and deliver the **MVP for Velo** — a cloud-native, Australian digital ADI (Restricted ADI pathway) that runs on a **sponsor bank** for settlement/clearing, and initially ships four capabilities: a **zero-fee transaction account**, **customer onboarding (manual KYC)**, **inbound deposits (Direct Entry + NPP/PayID)**, and **real-time internal transfers**.

The MVP must be **compliant, scalable, and future-proof** — future roadmap items (KYC/KYB, external transfers, fraud, AML/CTF, back-office portal, enrichment) must not be architecturally blocked.

---

## 3. Document map

Read in order. Each doc is self-contained and reflects one of the bullets in the user question.

| # | File | Answers |
|---|---|---|
| 1 | [`docs/01-product-and-business.md`](docs/01-product-and-business.md) | **What the product is, the business, the high-level requirement.** |
| 2 | [`docs/02-baseline-and-core-scope.md`](docs/02-baseline-and-core-scope.md) | **What has already been accomplished; what is the core business you must build on.** |
| 3 | [`docs/03-mvp-requirements.md`](docs/03-mvp-requirements.md) | **Full MVP requirement list — what is in, what is left out, and the gaps you must call out.** |
| 4 | [`docs/workflows/`](docs/workflows/) | **Per-requirement workflow docs**, each with (a) a sample case, (b) a sequence diagram covering Customer ↔ Velo ↔ Sponsor Bank ↔ Australian scheme, (c) an ERD delta showing the models it touches, (d) how our services interact at the cloud level. |
| 5 | [`docs/05-erd-consolidated.md`](docs/05-erd-consolidated.md) | **Consolidated first-cut ERD** (Mermaid) with every entity, key attributes, and relationships. This is the ERD you attach to the email. |
| 6 | [`docs/06-cloud-architecture.md`](docs/06-cloud-architecture.md) | **High-level architecture diagram** (Mermaid) plus a service-by-service breakdown, external integrations, VPC / network posture, CI/CD, and testing strategy. This is the architecture diagram you attach to the email. |
| 7 | [`docs/07-explainers.md`](docs/07-explainers.md) | **Background glossary** — Australian payment rails (BSB, PayID, NPP, DE), FCS + what "auditable" really means, RADI vs Full ADI, double-entry / sub-ledger / idempotency / exactly-once explained with worked examples, trade-offs compared in tables, PII + data residency, TLS/KMS/MFA/WAF/SoD explained, **Velo vs ZaloPay/MoMo (Vietnam e-wallets)**, and how the sponsor bank actually plugs in. Read this first if any AU-banking term in the other docs is unfamiliar. |

### Workflow docs (one per MVP requirement)

| File | MVP feature | Australian rail |
|---|---|---|
| [`docs/workflows/01-customer-onboarding.md`](docs/workflows/01-customer-onboarding.md) | Customer onboarding (offline/manual KYC) | Ops portal / manual review |
| [`docs/workflows/02-inbound-direct-entry.md`](docs/workflows/02-inbound-direct-entry.md) | Inbound deposit — DE credit | BECS / Direct Entry via sponsor bank |
| [`docs/workflows/03-inbound-npp-payid.md`](docs/workflows/03-inbound-npp-payid.md) | Inbound deposit — NPP / PayID | NPP + PayID + Osko via sponsor bank |
| [`docs/workflows/04-internal-transfer.md`](docs/workflows/04-internal-transfer.md) | Real-time internal (Velo → Velo) transfer | None — settles on Velo's ledger |
| [`docs/workflows/05-interest-accrual.md`](docs/workflows/05-interest-accrual.md) | Interest accrual & posting on positive balances | Internal (regulated by APRA/tax rules) |

---

## 4. How to use this pack

1. Skim `docs/01-product-and-business.md` and `docs/03-mvp-requirements.md` to lock in scope in your head.
2. Read the five workflow docs — each one seeds the sequence diagram you might use for deliverable #3. Pick one to elevate into your submission (recommendation: **inbound NPP/PayID** — it exercises the sponsor bank boundary, the async settlement callback, and the fraud/AML hooks).
3. Use `docs/05-erd-consolidated.md` as the ERD you export/screenshot for the email.
4. Use `docs/06-cloud-architecture.md` as the architecture diagram you export/screenshot for the email.
5. Bring `docs/02-baseline-and-core-scope.md` mentally into the interview — it maps every "we accepted this / we deferred this" trade-off to a founder concern.

---

## 5. Assumptions I baked in (call these out on the day)

- **Cloud:** AWS in `ap-southeast-2 (Sydney)` with `ap-southeast-4 (Melbourne)` as DR. Justifies APRA CPS 234 data residency and CPS 231 outsourcing due diligence.
- **Architecture style:** **Modular monolith on day one, with service boundaries drawn along the future microservice seams** (customer, account/ledger, payments-in, payments-out placeholder, notifications, ops portal). Cheaper to run, easier to reason about at seed stage, non-blocking for the roadmap.
- **Backend:** TypeScript (NestJS) or Kotlin (Spring Boot). Docs assume **NestJS + TypeScript** for concreteness — swap freely.
- **Frontend:** Next.js (customer app) + Next.js (ops portal), shadcn/ui, Tailwind.
- **Database:** **PostgreSQL (Aurora)** for the ledger and OLTP; **Kafka (MSK)** for domain events and payment ingest; **S3** for immutable evidence (ID docs, statements); **KMS** for envelope encryption; **Cognito or Auth0** for customer identity; **AWS SES + Twilio** for notifications.
- **Sponsor bank integration:** SFTP for BECS/DE files (batch) + REST/webhooks for NPP messages (near-real-time). The sponsor bank is Velo's connection to **BECS, NPP, PayID, HVCS/RITS**.
- **Money is stored in minor units (cents) as `BIGINT`** with a currency column, never as float.
- **Every posting is a double-entry event**; balances are a **materialised projection** of the ledger, not the source of truth.

If any of these are wrong for the panel's context, they're easy to swap without redesigning the model.
