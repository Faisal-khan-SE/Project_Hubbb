# ADR 0001: Modular monolith on Spring Boot + PostgreSQL

- **Status:** Accepted
- **Date:** 2026-10-28
- **Deciders:** Founder A, Founder B

## Context
Two founders, building lean with AI-assisted coding, need a marketplace with escrow money flows, strict
business rules (fees, penalties, cancellations), background jobs and audit trails. Money correctness and
transactional consistency matter more than independent scaling. The team must be able to change rules
quickly and test them thoroughly.

## Decision
- One deployable backend: Java 21, Spring Boot 3, Maven, PostgreSQL 16, Flyway migrations.
- Modular monolith: one package per module under `com.marketplace.<module>` (users, projects, bidding,
  delivery, payments, messaging, rules, notifications, reviews, disputes, admin, jobs, invoices).
- Each module has three layers: `presentation/` (controllers, DTOs, no rules), `business/` (services,
  domain, rules, repository interfaces, ports), `data/` (JPA repositories, adapters). Enforced by ArchUnit tests.
- External systems (payment gateway, payouts, AI, storage, email, virus scan, alerts) are always behind port
  interfaces so they can be mocked or replaced.
- Money: `NUMERIC(12,2)` / `BigDecimal` only, append-only payments rows (D1). Business rules are data in tables (fees, penalties, tax).

## Alternatives considered
- **Microservices:** independent scaling, but distributed transactions around escrow, more infra and more failure modes. Too much for two people now.
- **Node.js/NestJS backend:** one language with the frontend, but weaker fit for the team's typed/transactional preference and BigDecimal-style money handling.
- **MySQL:** familiar, but PostgreSQL gives partial unique indexes, JSONB, CHECK constraints and full-text search we rely on.

## Consequences
- Simple transactions for money movements; one database to back up.
- Module boundaries must be protected by tests and reviews, or the monolith turns into a tangle.
- If one module (e.g. messaging) later needs separate scaling, its port boundaries make extraction possible.
- The AI work lives in a separate Python service (ADR 0002), so the monolith never depends on Python libraries.
