# Phase 4b — Table Ownership

## Table Ownership Assignments

Ownership derived from which domain's service code writes to each table (per Phase 1 dependency map).

| Table | Owner Service | Write Evidence |
|---|---|---|
| customers | customer-service | CustomerService: registerCustomer, verifyKyc, updateCustomerProfile, suspendCustomer, closeCustomer |
| accounts | account-service | AccountService: openAccount, applyInterest, debitAccount, creditAccount, closeAccount. CustomerService cross-domain writes removed — replaced with REST notification events |
| transactions | account-service | AccountService: debitAccount, creditAccount (creates Transaction records) |
| payment_orders | payments-service | PaymentService: initiatePayment, validatePayment, executePayment, cancelPayment |

---

## Cross-Domain FK Resolution

### FK 1: `accounts.customer_id` → `customers.id`
- **Crosses boundary**: account-service → customer-service
- **Chosen strategy**: Denormalised reference column
- **Rationale**: Customer data (KYC status, active status) is read at account-open time via synchronous REST call. The `customer_id` scalar is retained in the `accounts` table as a plain column with no FK constraint. Customer profile changes (suspend/close) are propagated to account-service via REST notification events (`CustomerSuspended`, `CustomerClosed`).
- **Implementation**: Remove `@ManyToOne` / `@JoinColumn` from `Account.customer`; replace with `private Long customerId`. Remove FK constraint from DDL.

### FK 2: `transactions.account_id` → `accounts.id`
- **Crosses boundary**: NO — both `transactions` and `accounts` are owned by account-service
- **Chosen strategy**: Keep as same-domain FK within account-service schema
- **Rationale**: Both tables live in the same service database; a standard FK constraint is safe and correct.

### Cross-domain refs in `payment_orders` (already plain scalars in monolith)
- `payment_orders.customer_id` → denormalised reference; `customer_name` snapshot stored alongside. No FK constraint. Kept in sync via payment initiation (synchronous REST read at initiation time).
- `payment_orders.debit_account_number` → denormalised reference (account number string). No FK constraint.
- `payment_orders.credit_account_number` → denormalised reference (account number string). No FK constraint.
- `payment_orders.transaction_id` → same-domain scalar ref to `transactions.id` within payments-service. Note: `transactions` table is owned by account-service; `payment_orders.transaction_id` stores the ID returned by account-service's debit call. No FK constraint across services.

---

## Schema Version Tracking

Each service schema includes a `schema_version` table for independent migration tracking.

---

## Summary

| Table | Owner | Cross-domain FK removed? | Strategy applied |
|---|---|---|---|
| customers | customer-service | N/A | No cross-domain FKs on this table |
| accounts | account-service | YES — customer_id FK removed | Denormalised scalar + REST notification sync |
| transactions | account-service | NO — account_id FK kept | Same-domain FK (safe) |
| payment_orders | payments-service | N/A — already plain scalars | Denormalised refs; no FK constraints |
