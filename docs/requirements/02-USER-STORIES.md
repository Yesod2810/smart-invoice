# SMART INVOICE — USER STORIES & ACCEPTANCE CRITERIA

**Project Name:** Smart Invoice — Automated Invoice Management System  
**Document Type:** Software Requirements — User Stories  
**Version:** 1.0  
**Status:** Draft  
**Date:** 28/09/2026  
**Related Document:** 01-PROJECT-SCOPE.md  
**Development Methodology:** Agile / Scrum

---

## 1. Document Purpose

This document defines the functional requirements of the Smart Invoice system from the perspective of its users.

Each user story describes a specific business capability, its expected outcome, acceptance criteria, and priority.

The document serves as a reference for:

- Backend API development.
- Database design.
- Business logic implementation.
- Unit and integration testing.
- Sprint planning.
- Functional acceptance testing.

All implementation work must remain consistent with the approved project scope.

---

## 2. User Roles

### 2.1 Administrator (ADMIN)

The Administrator manages system access and supervises business operations.

Permissions include:

- Manage staff accounts.
- Configure business information.
- Manage customers and products.
- Access all invoices.
- Review payment records.
- Access audit logs.
- Manage operational configurations.

### 2.2 Staff (STAFF)

Staff members perform daily invoice management operations.

Permissions include:

- Create and update customers.
- Create and update products.
- Create invoice drafts.
- Review and issue invoices.
- Generate invoice PDFs.
- Send invoices to customers.
- Record payments.
- View permitted invoice records.

### 2.3 Customer (CUSTOMER)

Customers receive invoices and payment information.

For the initial MVP:

- Customers do not need a system account.
- Customers receive invoices through email.
- Customers cannot access internal administration APIs.

---

## 3. Priority Definitions

| Priority | Meaning | Description |
| --- | --- | --- |
| P0 | Critical | Required for the MVP |
| P1 | High | Important but can follow core functionality |
| P2 | Medium | Future enhancement |

A P0 user story must be implemented and tested before the initial release is considered functionally complete.

---

## 4. EPIC 01 — Authentication & Authorization

### US-01 — User Login

**Priority:** P0  
**Actor:** Administrator / Staff  
**Module:** Authentication

#### User Story

As an authorized user, I want to log in to the system so that I can securely access invoice management features.

#### Acceptance Criteria

##### AC-01 — Successful login

- Given an active user account exists.
- When the user submits valid credentials.
- Then the system authenticates the user.
- And returns an authentication token.

##### AC-02 — Invalid credentials

- Given incorrect credentials are submitted.
- When authentication is attempted.
- Then the system rejects the request.
- And returns a standardized authentication error.

##### AC-03 — Disabled account

- Given a user account is disabled.
- When the user attempts to log in.
- Then access is denied.

##### AC-04 — Protected endpoints

- Given a request does not contain valid authentication credentials.
- When the user accesses a protected endpoint.
- Then the API returns HTTP 401 Unauthorized.

#### Business Rules

- Passwords must never be stored in plaintext.
- Authentication responses must not expose password hashes.
- Credentials must be validated server-side.
- Authentication failures must not reveal whether an email account exists.

#### Expected API

POST /api/v1/auth/login

---

### US-02 — Role-Based Access Control

**Priority:** P0  
**Actor:** Administrator  
**Module:** Security

#### User Story

As an Administrator, I want system access to be restricted according to user roles so that unauthorized operations are prevented.

#### Acceptance Criteria

##### AC-01 — Administrator access

Given an authenticated ADMIN user, when the user accesses an administrator endpoint, then the operation is permitted.

##### AC-02 — Restricted staff access

Given an authenticated STAFF user, when the user accesses an administrator-only endpoint, then the system returns HTTP 403 Forbidden.

##### AC-03 — Unauthenticated access

Given an unauthenticated request, when a protected API is accessed, then the system returns HTTP 401 Unauthorized.

#### Business Rules

- Roles must be verified on the backend.
- Client-side restrictions alone are insufficient.
- Sensitive operations must enforce authorization independently.
- Record-level access restrictions must be enforced where applicable.

---

## 5. EPIC 02 — Customer Management

### US-03 — Create Customer

**Priority:** P0  
**Actor:** Administrator / Staff  
**Module:** Customer

#### User Story

As a Staff member, I want to create customer records so that customer information can be reused when generating invoices.

#### Acceptance Criteria

##### AC-01 — Successful creation

Given valid customer information, when the user submits the creation request, then the system stores the customer and returns a generated customer ID.

##### AC-02 — Required information

Given required fields are missing, when the request is submitted, then the system rejects the request with HTTP 400.

##### AC-03 — Invalid email

Given an invalid email format, when the request is submitted, then validation fails.

##### AC-04 — Database failure

Given a database error occurs, when customer creation is attempted, then the operation is rolled back.

#### Data Requirements

- Customer name.
- Email.
- Phone number.
- Billing address.
- Optional tax identification number.
- Created timestamp.
- Updated timestamp.

#### Expected API

POST /api/v1/customers

---

### US-04 — Update and Retrieve Customer

**Priority:** P0  
**Actor:** Administrator / Staff  
**Module:** Customer

#### User Story

As a Staff member, I want to view and update customer information so that business records remain accurate.

#### Acceptance Criteria

- Existing customer information can be retrieved.
- Valid customer data can be updated.
- Invalid updates are rejected.
- Nonexistent customers return HTTP 404.
- Customer records support pagination and searching.
- Changes to customer information must not modify the historical details of issued invoices.

#### Expected APIs

GET /api/v1/customers

GET /api/v1/customers/{id}

PUT /api/v1/customers/{id}

---

## 6. EPIC 03 — Product Management

### US-05 — Create and Manage Product

**Priority:** P0  
**Actor:** Administrator / Staff  
**Module:** Product

#### User Story

As a Staff member, I want to manage products and services so that invoice items can be selected from a centralized catalogue.

#### Acceptance Criteria

##### AC-01 — Create product

Given valid product information, when a product is created, then it is stored successfully.

##### AC-02 — Unique SKU

Given an SKU already exists, when another product with the same SKU is created, then the operation is rejected.

##### AC-03 — Invalid price

Given a negative product price, when the request is submitted, then validation fails.

##### AC-04 — Update price

Given an existing product, when its price is updated, then future invoice drafts can use the updated price.

##### AC-05 — Historical preservation

Given an invoice has already been issued, when the product price changes, then the issued invoice remains unchanged.

#### Data Requirements

- SKU.
- Product name.
- Description.
- Unit of measurement.
- Unit price.
- Tax configuration.
- Active status.

#### Expected APIs

POST /api/v1/products

GET /api/v1/products

GET /api/v1/products/{id}

PUT /api/v1/products/{id}

DELETE /api/v1/products/{id}

DELETE deactivates the product (`is_active = false`); products are never physically deleted through the API (BR-PRO-003, BR-PRO-006).

---

## 7. EPIC 04 — Invoice Management

### US-06 — Create Invoice Draft

**Priority:** P0  
**Actor:** Administrator / Staff  
**Module:** Invoice

#### User Story

As a Staff member, I want to create an invoice draft for a customer so that I can prepare billing information before issuing the invoice.

#### Acceptance Criteria

##### AC-01 — Successful draft creation

- Given an existing customer and valid invoice items.
- When the user submits an invoice creation request.
- Then the system validates the request.
- And calculates the invoice amounts.
- And stores the invoice with DRAFT status.

##### AC-02 — Invalid customer

- Given the customer does not exist.
- When invoice creation is attempted.
- Then the system returns HTTP 404.

##### AC-03 — Empty invoice

- Given the invoice contains no items.
- When the request is submitted.
- Then the system rejects the request.

##### AC-04 — Invalid quantity

- Given an item quantity is zero or negative.
- When invoice creation is attempted.
- Then validation fails.

##### AC-05 — Transaction rollback

- Given an error occurs while saving invoice items.
- When the transaction fails.
- Then the invoice and associated items are rolled back.

##### AC-06 — Server-side pricing

- Given a product ID and quantity are supplied.
- When the invoice is created.
- Then product pricing is retrieved and validated by the backend.
- And client-supplied totals are not trusted.

#### Example Request

```json
{
  "customerId": 1,
  "currency": "VND",
  "items": [
    {
      "productId": 101,
      "quantity": 2
    },
    {
      "productId": 102,
      "quantity": 1
    }
  ],
  "discountAmount": 50000,
  "dueDate": "2026-10-30"
}
```

#### Expected API

POST /api/v1/invoices

---

### US-07 — Automated Invoice Calculation

**Priority:** P0  
**Actor:** Staff / System  
**Module:** Invoice Calculation

#### User Story

As a Staff member, I want invoice totals to be calculated automatically so that manual calculation errors are reduced.

#### Acceptance Criteria

##### AC-01 — Subtotal calculation

Given valid invoice items, when invoice calculation runs, then the subtotal equals the sum of all item amounts.

##### AC-02 — Discount calculation

Given a valid discount, when invoice calculation runs, then the discount is applied according to the configured discount rules.

##### AC-03 — Tax calculation

Given applicable tax rates, when invoice calculation runs, then taxes are calculated according to the configured line-level tax rules.

##### AC-04 — Final total

Given subtotal, discounts, and applicable taxes, when calculation completes, then the final payable amount is calculated correctly.

##### AC-05 — Decimal precision

Given monetary calculations are performed, then decimal-safe arithmetic must be used.

#### Business Rules

- Java BigDecimal must be used for monetary calculations.
- PostgreSQL NUMERIC must be used for monetary columns.
- Floating-point data types must not be used for money.
- Discounts must not produce a negative payable amount.
- Rounding must follow the configured currency policy.
- Tax calculations must use the applicable item-level configuration.

#### Example

| Field | Amount |
| --- | ---: |
| Subtotal | 1,000,000 VND |
| Discount | 100,000 VND |
| Taxable amount | 900,000 VND |
| Example tax (10%) | 90,000 VND |
| Grand total | 990,000 VND |

The example tax rate is for testing only.

---

### US-08 — Issue Invoice

**Priority:** P0  
**Actor:** Administrator / Staff  
**Module:** Invoice

#### User Story

As a Staff member, I want to issue an approved invoice draft so that it becomes a finalized business document.

#### Acceptance Criteria

##### AC-01 — Successful issuance

Given a valid DRAFT invoice, when issuance is confirmed, then the system assigns a unique invoice number and changes its status to ISSUED.

##### AC-02 — Duplicate issuance

Given an invoice has already been issued, when the issuance request is repeated, then no additional invoice number is generated.

- A retry carrying the same `Idempotency-Key` replays the original successful response.
- Any other issuance request for an invoice that is no longer DRAFT returns HTTP 409 (`INVOICE_NOT_DRAFT`). See BR-ISS-004.

##### AC-03 — Invalid invoice

Given an invoice contains invalid financial information, when issuance is attempted, then the request is rejected.

##### AC-04 — Historical snapshot

Given an invoice is issued, then customer details, item descriptions, prices, discounts, and taxes are preserved.

##### AC-05 — Transaction safety

Given concurrent issuance requests occur, then only one successful issuance is committed.

#### Business Rules

- Invoice numbers must be unique within their defined numbering scope.
- Issued financial data cannot be modified directly.
- Issuance must be transactional.
- Issuance must be idempotent.
- Issued invoices must not be hard-deleted.

#### Expected API

POST /api/v1/invoices/{id}/issue

---

### US-09 — Search and View Invoices

**Priority:** P0  
**Actor:** Administrator / Staff  
**Module:** Invoice

#### User Story

As a Staff member, I want to search and view invoices so that I can retrieve historical billing information efficiently.

#### Acceptance Criteria

- Users can retrieve invoice details.
- Users can filter invoices by status.
- Users can search by invoice number.
- Users can filter by customer.
- Users can filter by issue date.
- Results support pagination.
- Nonexistent invoices return HTTP 404.
- Access restrictions are enforced.

#### Expected APIs

GET /api/v1/invoices

GET /api/v1/invoices/{id}

Draft maintenance (BR-INV-010, BR-INV-012):

PUT /api/v1/invoices/{id} — replace a DRAFT's items, discount, due date, and notes; totals are recalculated server-side; at least one item must remain.

POST /api/v1/invoices/{id}/cancel — DRAFT → CANCELLED only.

---

## 8. EPIC 05 — PDF Generation

### US-10 — Generate Invoice PDF

**Priority:** P0  
**Actor:** Administrator / Staff  
**Module:** PDF

#### User Story

As a Staff member, I want to generate a PDF document from an issued invoice so that the invoice can be downloaded and shared with the customer.

#### Acceptance Criteria

##### AC-01 — Successful generation

Given an issued invoice exists, when PDF generation is requested, then a PDF document is produced.

##### AC-02 — Correct content

The document must display:

- Business name and contact information.
- Customer billing details.
- Invoice number.
- Issue date.
- Item descriptions.
- Quantities.
- Unit prices.
- Discounts.
- Applicable taxes.
- Total payable amount.

##### AC-03 — Historical consistency

Given the original customer or product records have changed, when an old invoice PDF is generated, then the document uses the stored issued snapshot.

##### AC-04 — Character encoding

Given Vietnamese characters exist, when PDF generation is performed, then the characters are displayed correctly.

##### AC-05 — Invalid invoice

Given the invoice does not exist or is not eligible for final PDF generation, then the operation is rejected.

#### Expected API

GET /api/v1/invoices/{id}/pdf

---

## 9. EPIC 06 — Email Delivery

### US-11 — Send Invoice Email

**Priority:** P0  
**Actor:** Administrator / Staff  
**Module:** Email

#### User Story

As a Staff member, I want to send an issued invoice to the customer's email address so that the customer receives the billing document automatically.

#### Acceptance Criteria

##### AC-01 — Successful delivery request

Given a valid issued invoice and customer email, when the send action is requested, then an email delivery job is created.

##### AC-02 — PDF attachment

Given the email job is processed, then the generated PDF is attached to the message.

##### AC-03 — Successful transmission

Given the configured SMTP provider accepts the message, then the delivery job is recorded as SENT.

##### AC-04 — Failed transmission

Given email transmission fails, then the failure is recorded and the job becomes eligible for retry according to the retry policy.

##### AC-05 — Invoice independence

Given email delivery fails, then the issued invoice remains valid and stored in the database.

##### AC-06 — Duplicate prevention

Given the same delivery request is retried with the same idempotency key, then no duplicate delivery job is created.

#### Expected API

POST /api/v1/invoices/{id}/send (requires `Idempotency-Key` header; returns HTTP 202 Accepted)

#### Business Rules

- Email delivery must not occur inside the invoice database transaction.
- Email jobs must have trackable statuses.
- Retry attempts must be limited.
- Duplicate delivery requests must be controlled.
- SENT means the provider accepted the message, not that the recipient read it.

---

## 10. EPIC 07 — Payment Management

### US-12 — Record Payment

**Priority:** P0  
**Actor:** Administrator / Staff  
**Module:** Payment

#### User Story

As a Staff member, I want to record customer payments so that the system can track the outstanding balance of each invoice.

#### Acceptance Criteria

##### AC-01 — Full payment

Given an issued invoice has an outstanding balance, when a payment equal to the outstanding amount is recorded, then the payment status becomes PAID.

##### AC-02 — Partial payment

Given the payment amount is smaller than the outstanding balance, when the payment is recorded, then the status becomes PARTIALLY_PAID.

##### AC-03 — Invalid payment

Given a negative or zero payment amount, when the request is submitted, then validation fails.

##### AC-04 — Overpayment

Given a payment exceeds the outstanding balance, when the payment is submitted, then the operation is rejected.

##### AC-05 — Duplicate payment

Given the same payment request is submitted multiple times with the same idempotency key, then it is recorded only once.

##### AC-06 — Payment history

Given payments exist for an invoice, when payment history is requested, then all permitted payment records are returned.

#### Business Rules

Outstanding Balance = Invoice Total - Sum of Valid Payments

A "valid payment" is a payment record whose record status is COMPLETED (BR-PAY-003).

- Payments must be associated with an existing invoice.
- Payment operations must be transactional.
- Concurrent payments must not produce an invalid outstanding balance.
- Payment records must remain auditable.
- Invoice lifecycle status and payment status must be tracked separately.

#### Expected APIs

POST /api/v1/invoices/{id}/payments (requires `Idempotency-Key` header)

GET /api/v1/invoices/{id}/payments

---

## 11. EPIC 08 — Audit & Automation

### US-13 — Audit Log

**Priority:** P1  
**Actor:** Administrator  
**Module:** Audit

#### User Story

As an Administrator, I want important system operations to be recorded so that I can review business activities and investigate data changes.

#### Acceptance Criteria

- Invoice issuance is logged.
- Payment creation is logged.
- Important record modifications are logged.
- Logs identify the responsible user.
- Logs include timestamps and relevant entity references.
- Only authorized users can access audit records.
- Audit logs must not expose passwords or authentication secrets.

---

### US-14 — Automatic Payment Reminder

**Priority:** P1  
**Actor:** System  
**Module:** Scheduler

#### User Story

As a business operator, I want the system to automatically identify overdue invoices so that customers can receive payment reminders without manual follow-up.

#### Acceptance Criteria

##### AC-01 — Overdue detection

Given an invoice has an outstanding balance and its due date has passed, when the scheduled job runs, then the invoice is identified as overdue.

##### AC-02 — Reminder scheduling

Given an eligible overdue invoice exists, when reminder processing runs, then an email reminder job is created.

##### AC-03 — Paid invoice

Given an invoice is fully paid, when reminder processing runs, then no payment reminder is created.

##### AC-04 — Duplicate prevention

Given a reminder has already been scheduled for the same invoice and reminder period, then duplicate reminder jobs are prevented.

##### AC-05 — Failed delivery

Given reminder delivery fails, then the failure is recorded and retry rules are applied.

---

## 12. User Story Traceability Matrix

| User Story | Module | Priority | Planned Day |
| --- | --- | --- | --- |
| US-01 | Authentication | P0 | 19 |
| US-02 | Authorization | P0 | 20 |
| US-03 | Customer | P0 | 8 |
| US-04 | Customer | P0 | 8 |
| US-05 | Product | P0 | 9 |
| US-06 | Invoice | P0 | 13 |
| US-07 | Calculation | P0 | 12 |
| US-08 | Invoice | P0 | 14 |
| US-09 | Invoice | P0 | 14 |
| US-10 | PDF | P0 | 15 |
| US-11 | Email | P0 | 16 |
| US-12 | Payment | P0 | 18 |
| US-13 | Audit | P1 | 21 |
| US-14 | Scheduler | P1 | 17 |

The planned days represent the initial implementation schedule. Final integration and testing will occur during the testing phase.

---

## 13. Global Acceptance Criteria

The following conditions apply to all relevant user stories.

### API Validation

- Invalid requests must return standardized error responses.
- Required fields must be validated.
- Nonexistent resources must return HTTP 404.
- Unauthorized access must be rejected.
- API responses must not expose internal stack traces.

### Data Integrity

- Database relationships must be enforced.
- Financial operations must use decimal-safe calculations.
- Critical operations must support transaction rollback.
- Issued invoice snapshots must be preserved.

### Security

- Protected endpoints require authentication.
- Authorization must be enforced server-side.
- Sensitive information must not be logged.
- Passwords and secrets must not be exposed through API responses.

### Testing

- Critical business calculations must have unit tests.
- Database operations must have integration tests.
- Authentication and authorization must be tested.
- Error scenarios must be included.
- Duplicate and concurrent requests must be tested where relevant.

---

## 14. Definition of Ready

A user story is ready for development when:

1. The business objective is clearly defined.
2. Its actor is identified.
3. Acceptance criteria are documented.
4. Required input data is identified.
5. Expected output is understood.
6. Dependencies are known.
7. Relevant business rules have been reviewed.
8. The story is small enough to implement and test independently.

---

## 15. Definition of Done

A user story is considered completed when:

1. All acceptance criteria are satisfied.
2. Implementation follows the approved architecture.
3. Unit tests pass.
4. Required integration tests pass.
5. Validation and error handling are implemented.
6. Database changes are migrated using Flyway.
7. API documentation is updated.
8. Code is committed and reviewed.
9. No known critical defect remains.
10. The functionality has been verified through acceptance testing.

---

**Document Status:** Initial User Stories Defined.
