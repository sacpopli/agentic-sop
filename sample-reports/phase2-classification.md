# Phase 2 — Domain Classification

## Domain Model
- **customer-service** → Customer & Party Management
- **account-service** → Product & Account Servicing
- **payments-service** → Payments & Financial Transactions

---

## Classification Matrix

| Module / Class | Methods | Domains Touched | Action | Target Service(s) | Notes |
|---|---|---|---|---|---|
| `BankingApplication` | `main` | none | `move` | customer-service (replicated per service) | Entry point — each service gets its own copy |
| `CustomerController` | `register`, `get`, `verifyKyc`, `updateProfile`, `suspend`, `close` | Customer & Party Management | `facade` | customer-service | Thin controller delegating entirely to CustomerService; flagged for team review per SOP |
| `AccountController` | `open`, `get`, `getByCustomer`, `debit`, `credit`, `applyInterest`, `close` | Product & Account Servicing | `facade` | account-service | Thin controller delegating entirely to AccountService; flagged for team review per SOP |
| `PaymentController` | `initiate`, `validate`, `execute`, `status`, `cancel`, `history` | Payments & Financial Transactions | `facade` | payments-service | Thin controller delegating entirely to PaymentService; flagged for team review per SOP |
| `CustomerService` | `registerCustomer`, `verifyKyc`, `getCustomer`, `updateCustomerProfile`, `suspendCustomer`, `closeCustomer`, `findById` | Customer & Party Management + Product & Account Servicing | `split` | customer-service, account-service | `registerCustomer` writes AccountRepository (cross-domain); `suspendCustomer` writes AccountRepository (cross-domain); `closeCustomer` writes AccountRepository (cross-domain) — these cross-domain writes become REST notification events |
| `AccountService` | `openAccount`, `getAccount`, `getAccountsByCustomer`, `applyInterest`, `debitAccount`, `creditAccount`, `closeAccount` | Product & Account Servicing + Customer & Party Management | `split` | account-service, customer-service | `openAccount` reads CustomerRepository for KYC/status check (cross-domain read) — replaced with synchronous REST call to customer-service |
| `PaymentService` | `initiatePayment`, `validatePayment`, `executePayment`, `getPaymentStatus`, `cancelPayment`, `getPaymentHistory`, `findByReference` | Payments & Financial Transactions + Customer & Party Management + Product & Account Servicing | `split` | payments-service, customer-service, account-service | `initiatePayment` reads CustomerRepository + AccountRepository (cross-domain reads); `executePayment` calls AccountService.debitAccount/creditAccount (cross-domain service calls) — all replaced with REST calls |
| `Customer` | entity fields + enums + lifecycle hooks | Customer & Party Management | `move` | customer-service | All fields belong to customer domain; cross-domain `accounts` list removed |
| `Account` | entity fields + enums + lifecycle hooks | Product & Account Servicing | `move` | account-service | `customer` @ManyToOne replaced with plain `customerId: Long` scalar |
| `PaymentOrder` | entity fields + enums + lifecycle hooks | Payments & Financial Transactions | `move` | payments-service | Already uses plain scalar `customerId` — no ORM FK to remove |
| `Transaction` | entity fields + enums + lifecycle hooks | Payments & Financial Transactions | `move` | payments-service | `account` @ManyToOne replaced with plain `accountId: Long` scalar |
| `CustomerRepository` | `findByEmail`, `findByNationalId`, `findByStatus`, `existsByEmail` | Customer & Party Management | `move` | customer-service | All methods belong to customer domain |
| `AccountRepository` | `findByAccountNumber`, `findByCustomerId`, `findByCustomerIdAndStatus`, `existsByAccountNumber` | Product & Account Servicing | `move` | account-service | All methods belong to account domain |
| `PaymentOrderRepository` | `findByOrderReference`, `findByCustomerId`, `findByStatus`, `findByDebitAccountNumber` | Payments & Financial Transactions | `move` | payments-service | All methods belong to payments domain |
| `TransactionRepository` | `findByReferenceNumber`, `findByAccountId`, `findByAccountIdAndInitiatedAtBetween` | Payments & Financial Transactions | `move` | payments-service | All methods belong to payments domain |
| `SecurityConfig` | `filterChain` | none (cross-cutting) | `shared-kernel` | shared-kernel | Zero domain business logic; adapted per service to also permit `/internal/events` |
| `UserDetailsConfig` | `passwordEncoder`, `userDetailsService` | none (cross-cutting) | `shared-kernel` | shared-kernel | Zero domain business logic; dev/demo in-memory users |
| `AuditLogger` | `log` | none (cross-cutting) | `shared-kernel` | shared-kernel | Zero domain business logic |
| `ReferenceGenerator` | `accountNumber`, `transactionRef`, `paymentRef` | none (cross-cutting) | `shared-kernel` | shared-kernel | Zero domain business logic; used by all 3 services |
| `DataInitializer` | `run` | Customer & Party Management + Product & Account Servicing | `omit` | — | Dev/demo seed data loader (CommandLineRunner); not production code |

---

## Split Detail

### `CustomerService` — split
| Method | Domain | Target Service | Action |
|---|---|---|---|
| `registerCustomer` | Customer & Party Management (save customer) + Product & Account Servicing (save default account) | customer-service | `extract-method`: customer save stays in customer-service; account creation extracted — replaced with `AccountCreationRequestedEvent` REST notification to account-service |
| `verifyKyc` | Customer & Party Management | customer-service | `move` — pure customer domain |
| `getCustomer` | Customer & Party Management | customer-service | `move` — pure customer domain |
| `updateCustomerProfile` | Customer & Party Management | customer-service | `move` — pure customer domain |
| `suspendCustomer` | Customer & Party Management (suspend customer) + Product & Account Servicing (freeze accounts) | customer-service | `extract-method`: customer suspend stays in customer-service; account freeze extracted — replaced with `CustomerSuspendedEvent` REST notification to account-service |
| `closeCustomer` | Customer & Party Management (close customer) + Product & Account Servicing (close accounts) | customer-service | `extract-method`: customer close stays in customer-service; account close extracted — replaced with `CustomerClosedEvent` REST notification to account-service |
| `findById` | Customer & Party Management | customer-service | `move` — private helper, stays in customer-service |

### `AccountService` — split
| Method | Domain | Target Service | Action |
|---|---|---|---|
| `openAccount` | Product & Account Servicing (create account) + Customer & Party Management (KYC/status read) | account-service | `extract-method`: account creation stays in account-service; customer KYC/status check replaced with synchronous REST GET call to customer-service |
| `getAccount` | Product & Account Servicing | account-service | `move` — pure account domain |
| `getAccountsByCustomer` | Product & Account Servicing | account-service | `move` — pure account domain |
| `applyInterest` | Product & Account Servicing | account-service | `move` — pure account domain |
| `debitAccount` | Product & Account Servicing | account-service | `move` — pure account domain |
| `creditAccount` | Product & Account Servicing | account-service | `move` — pure account domain |
| `closeAccount` | Product & Account Servicing | account-service | `move` — pure account domain |

### `PaymentService` — split
| Method | Domain | Target Service | Action |
|---|---|---|---|
| `initiatePayment` | Payments & Financial Transactions (create order) + Customer & Party Management (customer status read) + Product & Account Servicing (account status read) | payments-service | `extract-method`: order creation stays in payments-service; customer status check replaced with synchronous REST GET to customer-service; account status check replaced with synchronous REST GET to account-service |
| `validatePayment` | Payments & Financial Transactions | payments-service | `move` — pure payments domain |
| `executePayment` | Payments & Financial Transactions (update order) + Product & Account Servicing (debit/credit accounts) | payments-service | `extract-method`: order status update stays in payments-service; debit/credit calls replaced with synchronous REST POST calls to account-service |
| `getPaymentStatus` | Payments & Financial Transactions | payments-service | `move` — pure payments domain |
| `cancelPayment` | Payments & Financial Transactions | payments-service | `move` — pure payments domain |
| `getPaymentHistory` | Payments & Financial Transactions | payments-service | `move` — pure payments domain |
| `findByReference` | Payments & Financial Transactions | payments-service | `move` — private helper |

---

## Ambiguous Items
None.

---

## Summary
| Action | Count | Classes |
|---|---|---|
| `move` | 9 | `Customer`, `Account`, `PaymentOrder`, `Transaction`, `CustomerRepository`, `AccountRepository`, `PaymentOrderRepository`, `TransactionRepository`, `BankingApplication` |
| `split` / `extract-method` | 3 | `CustomerService`, `AccountService`, `PaymentService` |
| `facade` | 3 | `CustomerController`, `AccountController`, `PaymentController` |
| `shared-kernel` | 4 | `SecurityConfig`, `UserDetailsConfig`, `AuditLogger`, `ReferenceGenerator` |
| `omit` | 1 | `DataInitializer` |
