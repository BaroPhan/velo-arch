# 03 — MVP Requirements: In / Out / Gaps

## Explicit MVP requirements (from the brief)

| ID | Requirement | Source in brief |
|---|---|---|
| R1 | **Velo Transaction Account** — zero-fee everyday account, interest on positive balances, FCS-protected | "The Business — Velo" section |
| R2 | **Customer Onboarding** — registration + **offline/manual** ID verification by Ops team; automated KYC is roadmap | "Customer Onboarding" |
| R3 | **Inbound Direct Entry (DE) credits** via sponsor bank | "Deposits" |
| R4 | **Inbound NPP / PayID credits** via sponsor bank | "Deposits" |
| R5 | **Real-time Internal Transfers** between Velo customers with immediate funds availability | "Internal Transfers" |

## Implicit MVP requirements (must-have but not spelled out)

These are the "invisible" requirements without which R1–R5 are unsafe, unauditable, or non-compliant. **Call these out explicitly in the interview** — the panel is watching for whether you separate "the feature" from "the plumbing the feature demands."

| ID | Requirement | Why it must be in MVP |
|---|---|---|
| R6 | **Customer identity, AuthN, MFA, session management** | R2 through R5 all require an authenticated user |
| R7 | **Double-entry ledger** | R3, R4, R5 all move money; you cannot ship a bank without a ledger |
| R8 | **Account issuance** — allocating a BSB + account number from the sponsor's range | R1 has no value if the account can't be credited |
| R9 | **Sponsor bank integration** — SFTP for DE files, REST/webhooks for NPP messages | R3 and R4 both need it |
| R10 | **Ops portal with RBAC & audited actions** | R2's manual KYC review, plus exception handling for R3–R5 |
| R11 | **Reconciliation** — daily sponsor-position vs Velo-ledger reconciliation with break management | Non-negotiable for an ADI |
| R12 | **Notifications** — email/push for onboarding decisions and money-in events | Baseline UX and customer trust |
| R13 | **Interest accrual & posting** on positive balances | Explicit in R1; needs daily accrual + monthly capitalisation |
| R14 | **Immutable audit trail + long-term event retention** | AUSTRAC (7 years), APRA CPS 234 |
| R15 | **Basic observability + alerting on money movement** | You cannot operate a bank blind |
| R16 | **Statements / transaction history in-app** | Implied by "modern banking experience" and by R1's "everyday account" |

## Explicitly out of MVP scope (from the brief's roadmap)

| ID | Explicitly deferred | Why the brief says it's OK to defer |
|---|---|---|
| D1 | **Automated KYC / KYB** | "Automated KYC is a future roadmap item — the MVP does not require it" |
| D2 | **Outbound / external transfers** (DE + NPP) | Listed under Future Roadmap |
| D3 | **Transaction enrichment / merchant categorisation** | Roadmap |
| D4 | **Fraud monitoring (real-time rules engine)** | Roadmap |
| D5 | **Full AML/CTF automation** (TTR / SMR / IFTI reporting) | Roadmap; brief only demands "mindful" design |
| D6 | **Full back-office ops portal** | MVP portal only needs onboarding-review + manual break workflows |

## Implicit deferrals (things the brief doesn't mention that you should also defer)

Call these out to demonstrate scoping discipline.

| ID | Deferred | Rationale for deferring |
|---|---|---|
| D7 | **Cards (debit card, Apple/Google Pay)** | Requires card scheme (Visa/MC), BIN sponsor, tokenisation — huge scope creep, no revenue impact for an MVP |
| D8 | **Joint accounts, trust accounts, minors' accounts** | Product complexity; can be a variant of the base account later |
| D9 | **Multiple accounts per customer / savings pockets** | Model supports it (customer → many accounts) but only 1 issued in MVP |
| D10 | **BPAY** | Separate scheme, sponsor bank has it but scope for later |
| D11 | **Statement PDFs / eStatements** | In-app transaction history is enough for MVP; PDFs later |
| D12 | **Customer support inbox / in-app chat** | Email + phone via ops team is fine at MVP scale |
| D13 | **Referral / rewards / gamification** | Not core banking |
| D14 | **Native mobile apps (iOS + Android) at day one** | Ship a **PWA + responsive web** or a single React Native codebase; do not staff two native teams pre-launch |
| D15 | **CDR (Open Banking) data sharing** | RADI is exempt initially; design events so CDR is a downstream consumer later |
| D16 | **Direct debit authorities (customer being the payer)** | Requires outbound DE first |

## What's missing / ambiguous in the brief (assumptions you must declare)

The interviewer wants you to identify these; they're deliberate gaps.

| # | Ambiguity | Assumption I'll defend |
|---|---|---|
| A1 | **Which sponsor bank?** | Assume a mid-tier sponsor (e.g. Cuscal / Volt-style pattern). Integration surface: SFTP for BECS files, REST + webhooks for NPP, batch reports for reconciliation. |
| A2 | **BSB / account number issuance** | Sponsor delegates a **BSB (or sub-BSB) and an account-number range**; Velo issues numbers from that range using a locally-generated, checksum-validated scheme. |
| A3 | **Are we the account holder of record with the sponsor?** | Yes — Velo holds a **pool / trust account** at the sponsor into which all customer funds flow; Velo maintains the sub-ledger of individual customer balances. |
| A4 | **Real-time vs eventual balance for inbound NPP** | NPP is near-real-time; we credit the customer's ledger balance the moment the sponsor confirms the pain.001/pacs.008 (typically < 60s). |
| A5 | **DE credit timing** | BECS is batch: file arrives from sponsor mid-morning (typically ~09:00 AEST); funds available same business day. |
| A6 | **Interest rate** | Assume a single published rate (e.g. 4.50% p.a.) accrued daily on the end-of-day balance, capitalised monthly. Rate configurable via ops portal. |
| A7 | **FX / non-AUD** | Out of scope entirely for MVP — AUD only. |
| A8 | **Cheque / RTGS / HVCS** | Out of scope — retail digital-only. |
| A9 | **Card acquiring / merchant payments** | Out of scope. |
| A10 | **Business customers / joint** | Individuals only, Australian residents ≥ 18. |
| A11 | **Regulatory reporting cadence** | Deferred to roadmap; events are captured so back-fill is trivial. |
| A12 | **SLA with sponsor bank** | Assume: NPP webhooks within 60s, DE files by 10:00 AEST business days, reconciliation report by 11:00 AEST. |
| A13 | **How does the customer share their account details externally?** | In-app "Receive" screen displays BSB + account number **and** their PayID (email/mobile) once verified. |

## MVP acceptance criteria (one sentence per requirement)

- **R1**: A verified customer sees an active account with a BSB + account number and a current AUD balance in cents.
- **R2**: A prospect can submit KYC in ≤ 5 minutes; an Ops user can approve/reject within the portal with a full audit trail.
- **R3**: A DE credit lodged into the sponsor pool for a Velo customer is reflected in the customer's balance the same business day, exactly once.
- **R4**: An NPP/PayID payment to a Velo customer is reflected in the customer's balance within 60 seconds of the sponsor's callback, exactly once.
- **R5**: A transfer from Velo customer A to Velo customer B debits A and credits B atomically in one transaction, with both balances immediately visible in-app.
- **R6–R16**: Each has its own SLO, called out in the workflow docs.
