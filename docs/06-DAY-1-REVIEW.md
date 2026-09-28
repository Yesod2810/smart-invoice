# SMART INVOICE — DAY 1 DOCUMENTATION REVIEW

**Document Type:** Cross-Document Consistency Audit  
**Version:** 1.0  
**Date:** 28/09/2026  
**Scope of Review:** README.md, 01-PROJECT-SCOPE.md, 02-USER-STORIES.md, 03-BUSINESS-RULES.md, 04-SYSTEM-ARCHITECTURE.md, 05-DATABASE-DRAFT.md  
**Constraint:** MVP stack stays Java 21, Spring Boot, PostgreSQL, Docker, Modular Monolith. No microservices, Kafka, Kubernetes, or Redis.

---

## 1. Summary

The five documents describe the same MVP. The main risks are in financial integrity: rounding order, discount allocation, idempotency contracts, and locking strategy. Before this review, these were described only as intentions ("must be deterministic", "may use locking or idempotency"). Implementation could therefore diverge between services and tests.

This review found **7 Critical**, **13 Major**, and **13 Minor** issues.

- All Critical issues and most Major issues are **resolved in the documents** by this review (status **Fixed**).
- Issues that need a business decision are **not** resolved by inventing requirements. They are listed as open decisions in §6.

Legend: **Fixed** means the documents were corrected in this review. **Open** means a decision is required and is listed in §6.

---

## 2. Critical Issues

| ID | Issue | Affected Documents / Sections | Impact | Correction | Status |
| --- | --- | --- | --- | --- | --- |
| C-01 | **Rounding sequence undefined.** BR-CAL-009 requires HALF_UP to whole VND but does not say *where* rounding happens: per line or per invoice, and on tax or on taxable amount. | BR-CAL-005/009/010, DB §21 | Two correct-looking implementations can produce different grand totals, so invoice totals would not reconcile with line sums. | Added **BR-CAL-012**, a fixed 7-step sequence: line rounding, then allocation, then line tax rounding. Invoice tax is the sum of rounded line taxes. DB §21.3 now states that final VND amounts are stored at scale 0. | Fixed |
| C-02 | **Discount allocation remainder undefined.** "Deterministic" and "account for rounding differences" were not specified. | BR-CAL-005, BR-CAL-010 | Allocated discounts may not sum to the invoice discount, so money is silently lost or created, which violates BR-CAL-010. | BR-CAL-012 step 3 now defines the method: proportional allocation, floor, then largest remainder, with ties broken by `line_number`. Σ allocations = D exactly. | Fixed |
| C-03 | **"Taxable lines" ambiguity.** BR-CAL-005 allocated the discount only across "taxable lines", while the default policy is proportional to all line amounts. | BR-CAL-005 | Allocating only to taxed lines changes the tax owed when 0% lines exist. | The discount is now allocated across **all** lines, including 0% lines, which is consistent with the documented proportional default. | Fixed |
| C-04 | **Duplicate-issuance contract contradicts itself.** BR-ISS-004 says to return the existing result *or* 409. BR-ERR-006 lists duplicate issuance as 409. US-08 says only "no new number". | US-08 AC-02, BR-ISS-004, BR-ERR-006 | The API behaviour is untestable, and a client retrying after a timeout cannot tell success from failure. | The same `Idempotency-Key` now replays the stored response. Any other issue request on a non-DRAFT invoice returns 409 `INVOICE_NOT_DRAFT`. All three documents are aligned. | Fixed |
| C-05 | **Idempotency not enforced at the database for concurrent requests.** `idempotency_records` had no unique constraint, and the key scope was unspecified. | DB §17 | Two concurrent requests with the same key could both run, producing duplicate payments or email jobs. | Added `UNIQUE (operation_type, idempotency_key)` and status values. The `Idempotency-Key` header is **required** for payments and send, and optional for issue. `payments.idempotency_key` UNIQUE is kept as a backstop. | Fixed |
| C-06 | **Payment and issuance concurrency mechanism undecided.** The architecture says "optimistic or pessimistic", BR-ISS-003 says "may use …", and DB §23 says "invoice locking". | Arch §10.2–10.3, BR-ISS-003, BR-PAY-009, DB §22–23 | Optimistic locking alone produces user-visible 409s on concurrent payments. An undecided mechanism leads to inconsistent code. | **Pessimistic `SELECT … FOR UPDATE` on the invoice row** is the chosen mechanism for issuance and payment, and the transaction steps are documented. `version` remains in use for draft edits. | Fixed |
| C-07 | **Invoice balance invariants not enforced by constraints.** Only the individual columns were non-negative. | DB §11.3 | A bug could persist `balance_due ≠ grand_total − paid_amount` or an issued invoice without a number or snapshot. | Added CHECK constraints: the balance identity, the grand-total identity, discount ≤ subtotal, issued-invoice completeness, and due_date ≥ issue_date. | Fixed |

---

## 3. Major Issues

| ID | Issue | Affected Documents / Sections | Impact | Correction | Status |
| --- | --- | --- | --- | --- | --- |
| M-01 | **Invoice number year basis undefined.** The sequence is keyed by year, but not by which date or timezone. The DB had no issue *date*, only `issued_at` TIMESTAMPTZ. | BR-ISS-002, BR-DATA-007, DB §11, §16 | Invoices issued near midnight UTC on 31 Dec could be numbered in the wrong year, and the PDF "Issue date" would be ambiguous. | Added `invoices.issue_date DATE` in the business timezone. `sequence_year` is derived from it. | Fixed |
| M-02 | **Gap-free numbering not specified.** Using a PostgreSQL SEQUENCE would leave gaps on rollback. | DB §16.5 | Gaps in commercial invoice numbering are hard to explain in an audit. | The sequence row is updated or locked inside the issuance transaction. A DB `SEQUENCE` is explicitly disallowed. | Fixed |
| M-03 | **Name collision on `payment_status`.** `payments.payment_status` (COMPLETED/REVERSED) collides with `invoices.payment_status` (UNPAID/…). "Valid payment" was undefined. | DB §13, BR-PAY-003, US-12 | Confusion in mapping and queries, and an undefined balance formula. | Renamed to `payments.status`. A valid payment is defined as COMPLETED in BR-PAY-003, US-12, and DB §13. | Fixed |
| M-04 | **Email job status semantics undefined.** It was unclear whether FAILED is terminal or retryable, and how stuck PROCESSING jobs recover. | BR-EMAIL-004/005, Arch §12, DB §14 | Jobs could be retried forever or never. A crashed worker would leave jobs stuck. | Defined PENDING (including retry waiting), PROCESSING with `locked_until`, SENT (terminal), and FAILED (terminal). Claiming uses `FOR UPDATE SKIP LOCKED`, with separate claim, deliver, and record steps so SMTP runs outside transactions. | Fixed |
| M-05 | **Circular module dependency.** The email sequence ran through `InvoiceController`/`InvoiceService`, while Email reads the Invoice snapshot. Payment wrote the invoice table directly. | Arch §6, §12.1 | Invoice ↔ Email and Invoice ↔ Payment cycles violate §3.3 "no circular dependencies". | Added **Arch §6.10 Module Dependency Rules**. Invoice exposes a query interface and a payment-summary method. The send endpoint moved to the Email module. | Fixed |
| M-06 | **P0 endpoints missing from the API list.** Draft update, draft cancel, payment history (US-12 AC-06), and customer/product get/put were missing. | Arch §8, US-09, US-12, BR-INV-010/012 | P0 acceptance criteria had no documented endpoint. | Added the endpoints to Arch §8 and US-09 / US-12. | Fixed |
| M-07 | **Staff account and business profile management have no user story.** Scope §4.1 and US §2.1 give ADMIN "manage staff accounts" and "configure business information", but no US, BR, or endpoint exists. Issuance needs a `business_profiles` row. | Scope §4.1, US §2.1, DB §10 | Issuance cannot work without an issuer profile, and there is no defined way to create the first ADMIN. | Flagged — **DEC-06**. | Open |
| M-08 | **No role permission matrix.** "Administrator-only endpoints" are never listed. Scope says Staff "manage products", while US-05 allows Staff to create and update products. "Record-level access" and "permitted invoices/payments" are undefined. | US-02, US-09, US-12, BR-AUTH-003, Arch §13.3 | US-02 AC-02 (403 for Staff) is untestable. | Flagged — **DEC-05**. | Open |
| M-09 | **When product prices are captured is contradictory.** BR-PRO-004 says "price at the time of issuance". US-06/US-07 calculate at draft creation. DB item snapshot columns are NOT NULL at insert. US-05 AC-04 says "future drafts" use a new price. | BR-PRO-004, US-05 AC-04, US-06, DB §12 | It is unclear whether a draft re-prices at issue, so a customer could be quoted one amount and invoiced another. | Wording aligned and the question flagged — **DEC-01**. | Open |
| M-10 | **Zero-total invoice payment status conflicts.** When grand total is 0, both BR-PAY-005 (paid = total → PAID) and BR-PAY-006 (paid = 0 → UNPAID) apply. Zero payments are rejected (BR-PAY-002). | BR-CAL-008, BR-PAY-005/006 | Undefined state. The invoice may stay UNPAID forever. | Flagged — **DEC-02**. | Open |
| M-11 | **Due date optionality.** US-06 supplies `dueDate`. `invoices.due_date` is NULL-able. BR-AUTO-001 overdue detection and PDF (BR-PDF-002) require a due date. | BR-INV-007, BR-PDF-002, DB §11 | Issued invoices without a due date cannot be printed per BR-PDF-002 and never become overdue. | Flagged — **DEC-03**. | Open |
| M-12 | **Monetary input precision.** It was not stated whether 1000.5 VND is rejected or rounded. | BR-CAL-001/009, US-06 example | Silent rounding of client input creates reconciliation differences. | Added **BR-CAL-013**: inputs above the currency scale are rejected with 400 and parsed straight to BigDecimal. | Fixed |
| M-13 | **Recipient address for invoice email.** It was unclear whether to use the customer snapshot email, the current customer email, or a request override. | US-11, BR-EMAIL-001, DB §14 | A wrong recipient means a data leak or a missed delivery. | Flagged — **DEC-04**. | Open |

---

## 4. Minor Issues

| ID | Issue | Affected Documents / Sections | Correction | Status |
| --- | --- | --- | --- | --- |
| m-01 | The Scope header said "Java" and not "Java 21". | Scope header | Corrected. | Fixed |
| m-02 | The Java package `com/smart-invoice` is invalid because hyphens are not allowed. | Arch §7 | Renamed to `com/smartinvoice`. | Fixed |
| m-03 | The architecture core-table list (8 tables) disagreed with the DB (11 tables), and the architecture ERD was missing relationships. | Arch §9.1–9.2 | Synced. The DB document is declared authoritative. | Fixed |
| m-04 | Scope step 10 said the "invoice reaches PAID status", which conflicts with the separated lifecycle and payment status (ADR-007). | Scope §6 | Reworded. | Fixed |
| m-05 | Product DELETE was "may be soft", while the DB said "should deactivate". | US-05, BR-PRO, DB §9.4 | Added BR-PRO-006: DELETE always deactivates. | Fixed |
| m-06 | The DB note allowed a draft with zero items, US-06 AC-03 rejects empty drafts, and draft-edit rules were silent. | DB §6, BR-INV-010 | BR-INV-010: a draft update must keep ≥ 1 item and only DRAFT can be modified. | Fixed |
| m-07 | Audit is "Should have" (Scope) and P1 (US-13), but BR-AUDIT-001 says "must" and the table is in the core schema. | Scope §5.2, BR-AUDIT-001 | Clarified: the table and issuance/payment events are MVP, and the query API is P1. | Fixed |
| m-08 | The email polling worker needs scheduling (P0), while the Scheduler module is P1 (US-14). | Arch §6.8, §12 | Clarified: `EmailJobProcessor` polling is part of P0 email, and the reminder scheduler is P1. | Fixed |
| m-09 | `customers.customer_code UNIQUE` is not backed by any requirement. | DB §8 | Marked optional and nullable. | Fixed |
| m-10 | `invoices.business_profile_id` was nullable. | DB §11 | Set to NOT NULL. | Fixed |
| m-11 | README.md was empty. | README.md | Added a project summary and document index. | Fixed |
| m-12 | Traceability schedules Auth/RBAC on days 19–20, after all business APIs, so security tests would come late. P1 US-14 (day 17) is also scheduled before P0 US-12 (day 18). | US §12 | Recommendation: move the security skeleton (US-01/02) to about day 7 and put P1 after all P0. The schedule itself is left unchanged. | Recommendation |
| m-13 | Not specified: token lifetime and refresh, fractional quantities, whether billing address is required, tax-exempt vs 0%, reminder period, and idempotency-record retention. | Arch §13.1, BR-INV-003, BR-CUS-001, BR-CAL-005, BR-AUTO-003, DB §17 | Flagged — DEC-07 to DEC-12. | Open |

---

## 5. Verification Checklists

### 5.1 P0 User Story → Architecture / Database Coverage

| US | Module | Endpoint(s) documented | Tables | Transaction boundary | Status |
| --- | --- | --- | --- | --- | --- |
| US-01 Login | Auth | ✔ | users | n/a | Ready (token lifetime: DEC-10) |
| US-02 RBAC | Security | ✔ (filters) | users.role | n/a | **Blocked on DEC-05** |
| US-03/04 Customer | Customer | ✔ | customers | single-row | Ready (DEC-08 minor) |
| US-05 Product | Product | ✔ | products | single-row | Ready |
| US-06 Draft | Invoice | ✔ | invoices, invoice_items | Arch §10.1 | Ready (DEC-01 affects repricing) |
| US-07 Calculation | Invoice | n/a | invoice_items columns | pure function | Ready (BR-CAL-012) |
| US-08 Issue | Invoice | ✔ | invoices, invoice_sequences, idempotency_records, business_profiles | Arch §10.2 | **Needs DEC-06** (issuer profile source) |
| US-09 Search | Invoice | ✔ | indexes present | read | Ready |
| US-10 PDF | PDF | ✔ | snapshots | read, outside txn | Ready (DEC-03 due date) |
| US-11 Email | Email | ✔ | email_jobs, idempotency_records | Arch §12.1 | Ready (DEC-04 recipient) |
| US-12 Payment | Payment | ✔ | payments, invoices, idempotency_records | Arch §10.3 | Ready (DEC-02 zero total) |

### 5.2 Database Checklist

| Item | Result |
| --- | --- |
| Primary keys | BIGINT identity on all 11 tables ✔ |
| Foreign keys | Documented, no cascade on financial records ✔. `invoice_items.product_id` restrict added ✔ |
| Unique constraints | products.sku, invoices.invoice_number, (invoice_id, line_number), (business_profile_id, sequence_year), payments.idempotency_key, email_jobs.idempotency_key, (operation_type, idempotency_key) ✔. Users email is case-insensitive unique ✔ |
| Monetary precision | NUMERIC(19,4) storage, VND values at scale 0, tax rate NUMERIC(7,4) as a percentage ✔ |
| Invoice snapshots | customer_snapshot, issuer_snapshot (versioned JSONB), item snapshot columns, and a completeness CHECK ✔ |
| Invoice numbering | Per business and year, gap-free, row-lock allocation, DB unique ✔ |
| Payment concurrency | Invoice row lock and balance CHECKs ✔ |
| Idempotency | Header contract, unique key per operation, payload-hash mismatch returns 409 ✔ |
| Email job processing | Durable, SKIP LOCKED claim, lease, bounded retry, terminal states ✔ |

### 5.3 Financial Integrity Checklist

| Item | Result |
| --- | --- |
| BigDecimal | Mandatory. String/exact construction, JSON parsed to BigDecimal ✔ |
| Rounding consistency | One centralized HALF_UP policy with a fixed sequence (BR-CAL-012) ✔ |
| Discount allocation | Largest-remainder, deterministic, exact sum ✔ |
| Invoice immutability | Service-level guard, DRAFT-only updates, issued CHECK. A DB trigger is **not** required for MVP; integration tests must cover it ✔ |
| Payment balance reconciliation | `balance_due = grand_total − paid_amount` CHECK. `paid_amount` is kept equal to Σ COMPLETED payments in the same transaction; an integration test must assert this ✔ |
| Duplicate issuance prevention | Row lock, DRAFT re-check, unique number, idempotency replay ✔ |
| Transaction rollback | Creation, issuance, and payment are each a single transaction, and SMTP and PDF run outside ✔ |

### 5.4 MVP Scope Control

No new infrastructure is introduced. Every fix uses PostgreSQL features: row locks, `SKIP LOCKED`, CHECK and UNIQUE constraints. Spring `@Scheduled` covers background work. Redis, message brokers, Kafka, Kubernetes, and microservices remain excluded, as the existing documents already state.

---

## 6. Open Business Decisions (Not Invented — Owner Must Decide)

| ID | Question | Options | Recommendation (non-binding) | Blocks |
| --- | --- | --- | --- | --- |
| DEC-01 | Does an existing DRAFT re-price when the catalogue price or tax rate changes? | (a) Values captured when the line is added or edited, and frozen at issue. (b) Re-read from the catalogue at issuance. | (a). The staff member sees the same numbers they confirm. BR-PRO-004 wording would then read "at the time the line is added to the draft". | US-06, US-08 |
| DEC-02 | What is the payment status of a zero-total invoice? | PAID at issuance, or UNPAID. | PAID at issuance. | US-12 |
| DEC-03 | Is the due date mandatory at issuance? Is there a default term? | Mandatory, or default N days. | Mandatory at issuance (≥ issue_date). | US-08, US-10, US-14 |
| DEC-04 | Which email address receives the invoice? | Snapshot, current customer, or request override. | Current customer email by default, with an optional validated override stored in `email_jobs.recipient_email`. | US-11 |
| DEC-05 | What is the role matrix? Which endpoints are ADMIN-only? Is there record-level restriction for STAFF? | — | MVP: ADMIN has everything. STAFF has everything except user and business-profile administration and the audit log. No record-level restriction in MVP. | US-02 |
| DEC-06 | How are the first ADMIN and the business profile created? Are staff-account and business-profile APIs in the MVP? | Flyway/env seed, or an admin API as a new P0 story. | Seed one ADMIN and one business profile from environment variables at startup. Admin APIs become a P1 story. | US-08, US-02 |
| DEC-07 | Are fractional quantities allowed? | Integer only, or up to 4 dp. | Up to 4 dp (the schema already allows it). | US-06 |
| DEC-08 | Is billing address required for a customer? | — | Required. It appears on the PDF. | US-03 |
| DEC-09 | What is a "reminder period" (P1)? | — | Decide before US-14. | US-14 |
| DEC-10 | Token lifetime, refresh token, and logout. | — | Access token only, 1 hour expiry, no refresh in MVP. | US-01 |
| DEC-11 | Must tax-exempt ("not subject to tax") be distinguished from 0%? | — | Treat as 0% in MVP. Revisit with regulated e-invoicing. | US-07 |
| DEC-12 | How long are idempotency records retained? | — | 24 h to 7 days. `payments.idempotency_key` stays unique forever. | Non-blocking |

---

## 7. Changes Applied in This Review

- **01-PROJECT-SCOPE.md** — Java 21 header. Audit scope clarification. Step 10 lifecycle/payment wording.
- **02-USER-STORIES.md** — US-05 soft delete. US-08 AC-02 idempotency contract. US-09 draft update and cancel endpoints. US-11 idempotency header and 202. US-12 payment history endpoint, `Idempotency-Key`, and valid-payment definition.
- **03-BUSINESS-RULES.md** — BR-PRO-004 note and new BR-PRO-006. BR-INV-010 draft rules. BR-CAL-005 all-lines allocation. New BR-CAL-012 (calculation sequence) and BR-CAL-013 (input precision). BR-ISS-004 contract. BR-ISS-006 locking. BR-PAY-003/008/009. BR-EMAIL-004/006. BR-ERR-006 codes. BR-AUDIT-001 scope.
- **04-SYSTEM-ARCHITECTURE.md** — Package name. Table list and ERD. Complete endpoint table. Invoice module interfaces. New §6.10 dependency rules. Transaction steps for issuance and payment. Email worker claim, deliver, and record steps.
- **05-DATABASE-DRAFT.md** — `issue_date`. NOT NULL `business_profile_id`. Invoice CHECK constraints. Item constraints. `payments.status` rename. Email claim semantics. Sequence allocation rules. `idempotency_records` unique constraint and statuses. Concurrency table.
- **README.md** — Project summary and document index.

---

## 8. Documentation Readiness Assessment

| Area | Readiness | Notes |
| --- | --- | --- |
| Requirements (Scope, US, BR) | **Ready with open decisions** | Contradictions resolved. DEC-01 to DEC-06 need an owner answer. |
| Architecture | **Ready** | Modules, dependencies, and transaction boundaries are defined. |
| Database | **Ready** | Constraints support every critical rule. Migrations can be written from DB §25. |
| Financial integrity | **Ready** | The calculation sequence can be specified as unit tests directly from BR-CAL-012. |

**Overall verdict: CONDITIONALLY READY.**

Implementation of Customer, Product, Calculation, and Invoice-draft work (US-03 to US-07, US-09) can start immediately.

The following must wait for their decisions to be recorded in this document:

- **US-02** (RBAC) — waits on DEC-05.
- **US-08** (issuance) — waits on DEC-01, DEC-03, and DEC-06.
- **US-11 / US-12** — wait on DEC-04 and DEC-02.

These are expected before their planned days, 13 to 18.

---

**Document Status:** Day 1 review complete.
