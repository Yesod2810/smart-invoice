# Smart Invoice

Automated invoice management backend. It covers customers, products, invoice drafts and issuance, PDF generation, email delivery, and payment tracking.

**Stack:** Java 21, Spring Boot, PostgreSQL, Flyway, Docker Compose. The architecture is a modular monolith.

**Status:** Day 1 — documentation baseline. No application code yet.

## Documentation

| # | Document | Purpose |
| --- | --- | --- |
| 01 | [Project Scope](docs/requirements/01-PROJECT-SCOPE.md) | Objectives, MVP scope, NFRs |
| 02 | [User Stories](docs/requirements/02-USER-STORIES.md) | P0/P1 stories and acceptance criteria |
| 03 | [Business Rules](docs/requirements/03-BUSINESS-RULES.md) | Calculation, lifecycle, payment, email rules |
| 04 | [System Architecture](docs/architecture/04-SYSTEM-ARCHITECTURE.md) | Modules, transactions, API, deployment |
| 05 | [Database Design](docs/database/05-DATABASE-DRAFT.md) | Schema, constraints, concurrency |
| 06 | [Day 1 Review](docs/06-DAY-1-REVIEW.md) | Consistency audit, open decisions, readiness |

The invoice PDFs are commercial billing documents. They are not legally valid electronic tax invoices.
