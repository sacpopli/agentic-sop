# Decomposition Report — Banking Monolith

**Generated:** 2026-04-23  
**SOP:** monolith-decomposition.sop.md  
**Config:** decomposition-config.json

---

## 1. Executive Summary

| Item | Value |
|---|---|
| Monolith path | `../banking-monolith` |
| Language | Java 17 / Spring Boot 3.2.0 / Maven |
| Total source files analysed | 20 Java files |
| Modules classified | 20 classes across 6 packages |
| Target services generated | 3 microservices + 1 shared-kernel |
| Domain framework | BIAN (Banking Industry Architecture Network) |
| Output path | `../decomposed_e2e_v3` |

**Target services:**
| Service | Domain | Port |
|---|---|---|
| customer-service | Customer & Party Management | 8081 |
| account-service | Product & Account Servicing | 8082 |
| payments-service | Payments & Financial Transactions | 8083 |
| shared-kernel | Cross-cutting utilities | library (no port) |

All three services are independently deployable, each with its own database, build descriptor, Dockerfile, and configuration. They communicate via synchronous REST calls and REST notification callbacks using the `DomainEvent` DTO defined in shared-kernel.

---

## 2. Domain Classification Matrix

| Module / Class | Methods | Domains Touched | Action | Target Service(s) | Notes |
|---|---|---|---|---|---|
| `BankingApplication` | `main` | none | `move` | all services (replicated) | Entry point per service |
| `CustomerController` | `register`, `get`, `verifyKyc`, `updateProfile`, `suspend`, `close` | Customer & Party Management | `facade` | customer-service | Thin controller; flagged for team review |
| `AccountController` | `open`, `get`, `getByCustomer`, `debit`, `credit`, `applyInterest`, `close` | Product & Account Servicing | `facade` | account-service | Thin controller; flagged for team review |
| `PaymentController` | `initiate`, `validate`, `execute`, `status`, `cancel`, `history` | Payments & Financial Transactions | `facade` | payments-service | Thin controller; flagged for team review |
| `CustomerService` | `registerCustomer`, `verifyKyc`, `getCustomer`, `updateCustomerProfile`, `suspendCustomer`, `closeCustomer` | Customer & Party Management + Product & Account Servicing | `split` | customer-service, account-service | Cross-domain account writes replaced with REST notification events |
| `AccountService` | `openAccount`, `getAccount`, `getAccountsByCustomer`, `applyInterest`, `debitAccount`, `creditAccount`, `closeAccount` | Product & Account Servicing + Customer & Party Management | `split` | account-service, customer-service | `openAccount` customer KYC check replaced with REST GET |
| `PaymentService` | `initiatePayment`, `validatePayment`, `executePayment`, `getPaymentStatus`, `cancelPayment`, `getPaymentHistory` | Payments & Financial Transactions + Customer & Party Management + Product & Account Servicing | `split` | payments-service, customer-service, account-service | All cross-domain calls replaced with REST |
| `Customer` | entity | Customer & Party Management | `move` | customer-service | Cross-domain accounts list removed |
| `Account` | entity | Product & Account Servicing | `move` | account-service | `@ManyToOne customer` → `Long customerId` |
| `PaymentOrder` | entity | Payments & Financial Transactions | `move` | payments-service | Already plain scalars in monolith |
| `Transaction` | entity | Payments & Financial Transactions | `move` | payments-service | `@ManyToOne account` → `Long accountId` |
| `CustomerRepository` | 4 query methods | Customer & Party Management | `move` | customer-service | |
| `AccountRepository` | 4 query methods | Product & Account Servicing | `move` | account-service | |
| `PaymentOrderRepository` | 4 query methods | Payments & Financial Transactions | `move` | payments-service | |
| `TransactionRepository` | 3 query methods | Payments & Financial Transactions | `move` | payments-service | |
| `SecurityConfig` | `filterChain` | none (cross-cutting) | `shared-kernel` | shared-kernel | Adapted to permit `/internal/events` |
| `UserDetailsConfig` | `passwordEncoder`, `userDetailsService` | none (cross-cutting) | `shared-kernel` | shared-kernel | Dev/demo in-memory users |
| `AuditLogger` | `log` | none (cross-cutting) | `shared-kernel` | shared-kernel | |
| `ReferenceGenerator` | `accountNumber`, `transactionRef`, `paymentRef` | none (cross-cutting) | `shared-kernel` | shared-kernel | |
| `DataInitializer` | `run` | — | `omit` | — | Dev seed data; not production code |

**Summary:** 9 move, 3 split/extract-method, 3 facade, 4 shared-kernel, 1 omit.

---

## 3. Ambiguous Modules

None. All 20 classes were classified with clear domain ownership.

---

## 4. Inter-Domain Contracts

### Synchronous REST Query APIs

| Caller | Endpoint | Purpose |
|---|---|---|
| account-service | GET /api/customers/{customerId} (customer-service) | Validate KYC + ACTIVE status before opening account |
| payments-service | GET /api/customers/{customerId} (customer-service) | Validate ACTIVE status before payment initiation |
| payments-service | GET /api/accounts/{accountNumber} (account-service) | Validate account status + ownership before payment initiation |
| payments-service | POST /api/accounts/{accountNumber}/debit (account-service) | Execute debit during payment execution |
| payments-service | POST /api/accounts/{accountNumber}/credit (account-service) | Execute credit during INTERNAL payment execution |

### REST Notification Callback Contracts (DomainEvent payloads)

| Event | Publisher | Receiver | Trigger | Payload |
|---|---|---|---|---|
| `CustomerRegistered` | customer-service | account-service | `CustomerService.registerCustomer` completes | `customerId`, `firstName`, `lastName`, `currency`, `productCode` |
| `CustomerSuspended` | customer-service | account-service | `CustomerService.suspendCustomer` completes | `customerId`, `reason` |
| `CustomerClosed` | customer-service | account-service | `CustomerService.closeCustomer` completes | `customerId` |

**DomainEvent wire format:**
```json
{
  "eventId":    "a3f1c2d4-...",
  "type":       "CustomerRegistered",
  "occurredAt": "2026-04-23T18:00:00Z",
  "payload": {
    "customerId":  1,
    "firstName":   "Alice",
    "lastName":    "Smith",
    "currency":    "USD",
    "productCode": "CURR-STD-001"
  }
}
```

**Circular chain check:** No circular chains. customer-service → account-service (notifications only). account-service → customer-service (synchronous reads only). payments-service → customer-service and account-service (synchronous reads/writes only).

---

## 5. Database Decomposition Summary

### Table Ownership
| Table | Owner Service | Notes |
|---|---|---|
| customers | customer-service | No cross-domain FKs on this table |
| accounts | account-service | `customer_id` FK removed; plain scalar retained |
| transactions | account-service | `account_id` FK kept (same-domain) |
| payment_orders | payments-service | All cross-domain refs already plain scalars in monolith |

### Cross-Domain FK Resolutions
| FK | Strategy | Sync mechanism |
|---|---|---|
| `accounts.customer_id` → `customers.id` | Denormalised scalar — no FK constraint | CustomerRegistered/CustomerSuspended/CustomerClosed REST notification events |
| `payment_orders.customer_id` | Denormalised scalar + `customer_name` snapshot | Synchronous REST read at payment initiation |
| `payment_orders.debit_account_number` | Denormalised scalar (account number string) | Synchronous REST read at payment initiation |
| `payment_orders.credit_account_number` | Denormalised scalar (account number string) | Synchronous REST read at payment initiation |
| `payment_orders.transaction_id` | Denormalised scalar (ID from account-service response) | Populated at payment execution |

---

## 6. Data Migration Options

### Migration Sequence (least-coupled first)
1. Stand up `customer-db`, `account-db`, `payments-db`
2. Run DDL scripts: `V1__customer_schema.sql`, `V1__account_schema.sql`, `V1__payments_schema.sql`
3. Migrate: `customers` → `customer-db` (no transformation)
4. Migrate: `accounts` → `account-db` (drop FK constraint on `customer_id`; value copied as-is)
5. Migrate: `transactions` → `account-db` (no transformation; `account_id` FK is same-domain)
6. Migrate: `payment_orders` → `payments-db` (no transformation; all refs already plain scalars)
7. Validate row counts and checksums after each table

### Large-Table Risk
No tables flagged. Current monolith uses H2 in-memory with dev/demo data only. If production `transactions` or `payment_orders` exceed 10M rows, flag for incremental migration strategy before proceeding.

### Migration Approach Trade-offs
| Approach | Pros | Cons |
|---|---|---|
| Big-bang | Simple, single window | Downtime; high rollback risk |
| Parallel run | Safe comparison | Dual-write complexity |
| Dual-write | Gradual shift | Write amplification |
| Strangler Fig | Lowest risk | Longest timeline; requires routing layer |

Decision on approach is left to the team.

---

## 7. Shared Kernel Contents

| Component | Package | Purpose |
|---|---|---|
| `DomainEvent` | `com.bank.shared.event` | REST notification callback DTO (new — required by decomposition) |
| `AuditLogger` | `com.bank.shared.util` | Cross-cutting audit logging |
| `ReferenceGenerator` | `com.bank.shared.util` | Account number, transaction ref, payment ref generation |
| `SecurityConfig` | `com.bank.shared.config` | Spring Security filter chain (adapted to permit `/internal/events`) |
| `UserDetailsConfig` | `com.bank.shared.config` | In-memory dev/demo users |

---

## 8. Recommended Decomposition Order

Based on coupling analysis (least coupled first):

1. **customer-service** — no runtime dependencies on other decomposed services; only publishes events outbound. Safest to deploy first.
2. **account-service** — depends on customer-service for synchronous reads (openAccount KYC check) and receives REST notification events. Deploy after customer-service is stable.
3. **payments-service** — depends on both customer-service and account-service for synchronous reads and writes. Deploy last.

This is a recommendation, not a prescription. The team's chosen migration strategy (Strangler Fig, parallel run, etc.) may dictate a different order.

---

## 9. Risk Register

| Risk | Severity | Description | Mitigation |
|---|---|---|---|
| Data consistency — distributed transactions | High | `CustomerService.closeCustomer` notifies account-service via REST event; if account-service is down, accounts are not closed. No two-phase commit. | Implement retry/idempotency on event delivery; add compensating transaction logic; consider outbox pattern in future enhancement. |
| Network latency — synchronous REST calls | Medium | `PaymentService.executePayment` makes 2 synchronous REST calls to account-service (debit + credit). Network failures cause payment failures. | Add timeout configuration on RestTemplate; implement retry with backoff; consider circuit breaker pattern (future enhancement per SOP scope). |
| Notification delivery failure | Medium | `CustomerEventPublisher` catches HTTP errors and logs a warning without rethrowing. If account-service is unreachable, accounts are not created/frozen/closed. | Monitor warning logs; implement dead-letter queue or retry mechanism; add health checks between services. |
| Team silos | Medium | Three separate services owned by potentially separate teams increases coordination overhead for cross-domain changes. | Establish API versioning strategy; define contract ownership; set up integration test suite covering cross-service flows. |
| Large-table migration complexity | Low (current) / High (production) | Current monolith uses H2 in-memory; no persistent data. Production volumes in `transactions` may exceed 10M rows. | Assess production row counts before migration; use incremental migration tools (e.g., `pg_dump` with batching, `gh-ost`) if needed. |
| `CustomerService.closeCustomer` race condition | Medium | customer-service marks customer CLOSED before account-service confirms zero balance. If account-service rejects the event, customer is CLOSED but accounts may still be open. | Implement saga pattern: customer-service should wait for account-service confirmation before marking CLOSED (future enhancement). |
| `facade` controllers not split | Low | Three controllers are marked `facade` and generated as-is. They delegate to a single service each, so the risk is low, but the team should review whether to split or keep them. | Team review recommended before production deployment. |

---

## 10. Definition of Done Checklist

### Per Service
| Item | customer-service | account-service | payments-service |
|---|---|---|---|
| Executable code generated | ✅ | ✅ | ✅ |
| Own DB schema (DDL) | ✅ | ✅ | ✅ |
| Build descriptor (pom.xml) | ✅ | ✅ | ✅ |
| Config files (application.yml + local) | ✅ | ✅ | ✅ |
| Dockerfile | ✅ | ✅ | ✅ |
| Unit tests | ⚠️ Skipped — no test framework in monolith build descriptor | ⚠️ Skipped | ⚠️ Skipped |
| CI/CD pipeline | ❌ Not generated — outside SOP scope | ❌ | ❌ |
| Observability (metrics, tracing) | ❌ Not in monolith — outside SOP scope | ❌ | ❌ |
| Integration tests | ❌ Not generated — outside SOP scope | ❌ | ❌ |

### Shared Kernel
| Item | Status |
|---|---|
| Library build descriptor (plain JAR) | ✅ |
| DomainEvent DTO | ✅ |
| Shared utilities (AuditLogger, ReferenceGenerator) | ✅ |
| Shared config (SecurityConfig, UserDetailsConfig) | ✅ |

---

## 11. Recommended Next Steps

| Priority | Action | Effort |
|---|---|---|
| 1 | Add `spring-boot-starter-test` + Mockito to each service's `pom.xml` and generate unit tests (Phase 6 was skipped due to missing test framework in monolith) | M |
| 2 | Add PostgreSQL JDBC driver (`org.postgresql:postgresql`) to each service's `pom.xml` — currently only H2 is present; production deployment requires PostgreSQL | S |
| 3 | Build and run `docker-compose up` to validate all three services start and communicate correctly | S |
| 4 | Set up CI/CD pipeline (GitHub Actions or equivalent) per service with build, test, and Docker image push stages | M |
| 5 | Add observability: Spring Boot Actuator + Micrometer + distributed tracing (e.g., OpenTelemetry) to each service | M |
| 6 | Implement retry + idempotency on `CustomerEventPublisher` to handle transient account-service unavailability | M |
| 7 | Write integration tests covering the three key cross-service flows: customer registration → account creation; customer suspension → account freeze; payment execution → debit + credit | L |
| 8 | Review `facade`-classified controllers (`CustomerController`, `AccountController`, `PaymentController`) and decide whether to split or retain as-is | S |
| 9 | Assess production database row counts and plan data migration strategy (big-bang vs. Strangler Fig vs. parallel run) | L |
| 10 | Replace in-memory `UserDetailsConfig` with OAuth2/OIDC integration for production security | L |

---

## Phase 6 Note

**Phase 6 (Unit Test Generation) was skipped.**

The monolith's `pom.xml` contains no test framework dependency (`spring-boot-starter-test`, `junit`, `mockito`, or similar). Per the SOP hard gate and skills file conflict resolution rules, the `config.test_framework=junit5` parameter does not override the build descriptor signal. Tests were not generated.

To enable test generation: add `spring-boot-starter-test` to the monolith's `pom.xml` (or directly to each decomposed service's `pom.xml`) and re-run Phase 6.
