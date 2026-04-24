# Phase 4a — Shared Schema Inventory

## All Tables in Monolith Database

### Table: `customers`
| Column | Type | Constraints |
|---|---|---|
| id | BIGINT | PK, AUTO_INCREMENT |
| first_name | VARCHAR | NOT NULL |
| last_name | VARCHAR | NOT NULL |
| email | VARCHAR | UNIQUE, NOT NULL |
| phone | VARCHAR | |
| national_id | VARCHAR | |
| date_of_birth | DATE | |
| status | VARCHAR | (PENDING_KYC, ACTIVE, SUSPENDED, CLOSED) |
| type | VARCHAR | (RETAIL, CORPORATE, SME) |
| created_at | TIMESTAMP | |
| updated_at | TIMESTAMP | |
| kyc_verified | BOOLEAN | |
| kyc_verified_at | TIMESTAMP | |
| kyc_document_ref | VARCHAR | |

Indexes: PK(id), UNIQUE(email)

### Table: `accounts`
| Column | Type | Constraints |
|---|---|---|
| id | BIGINT | PK, AUTO_INCREMENT |
| account_number | VARCHAR | UNIQUE, NOT NULL |
| type | VARCHAR | (CURRENT, SAVINGS, DEPOSIT, LOAN, CREDIT) |
| status | VARCHAR | (ACTIVE, DORMANT, CLOSED, FROZEN) |
| balance | DECIMAL(19,4) | |
| overdraft_limit | DECIMAL(19,4) | |
| currency | VARCHAR | |
| customer_id | BIGINT | FK → customers.id [CROSS-DOMAIN FK] |
| opened_at | TIMESTAMP | |
| closed_at | TIMESTAMP | |
| updated_at | TIMESTAMP | |
| product_code | VARCHAR | |
| interest_rate | DECIMAL(10,6) | |

Indexes: PK(id), UNIQUE(account_number), FK(customer_id)

### Table: `transactions`
| Column | Type | Constraints |
|---|---|---|
| id | BIGINT | PK, AUTO_INCREMENT |
| reference_number | VARCHAR | UNIQUE, NOT NULL |
| type | VARCHAR | (CREDIT, DEBIT) |
| status | VARCHAR | (PENDING, COMPLETED, FAILED, REVERSED) |
| amount | DECIMAL(19,4) | |
| currency | VARCHAR | |
| description | VARCHAR | |
| account_id | BIGINT | FK → accounts.id [SAME-DOMAIN FK — both owned by account-service] |
| initiated_at | TIMESTAMP | |
| completed_at | TIMESTAMP | |
| payment_rail | VARCHAR | |
| counterparty_account | VARCHAR | |
| counterparty_bank | VARCHAR | |
| counterparty_name | VARCHAR | |

Indexes: PK(id), UNIQUE(reference_number), FK(account_id)

### Table: `payment_orders`
| Column | Type | Constraints |
|---|---|---|
| id | BIGINT | PK, AUTO_INCREMENT |
| order_reference | VARCHAR | UNIQUE, NOT NULL |
| type | VARCHAR | (ACH, WIRE, INTERNAL, CARD) |
| status | VARCHAR | (INITIATED, VALIDATED, PROCESSING, COMPLETED, FAILED, CANCELLED) |
| amount | DECIMAL(19,4) | |
| currency | VARCHAR | |
| customer_id | BIGINT | plain scalar — no FK constraint in monolith [CROSS-DOMAIN REF] |
| customer_name | VARCHAR | denormalised snapshot |
| debit_account_number | VARCHAR | plain scalar ref to accounts [CROSS-DOMAIN REF] |
| credit_account_number | VARCHAR | plain scalar ref to accounts [CROSS-DOMAIN REF] |
| beneficiary_name | VARCHAR | |
| beneficiary_bank | VARCHAR | |
| beneficiary_bank_code | VARCHAR | |
| remittance_info | VARCHAR | |
| initiated_at | TIMESTAMP | |
| processed_at | TIMESTAMP | |
| transaction_id | BIGINT | plain scalar ref to transactions [SAME-DOMAIN REF] |

Indexes: PK(id), UNIQUE(order_reference)

---

## Cross-Domain Foreign Keys

| FK | From Table.Column | To Table.Column | Crosses Domain Boundary? | Strategy |
|---|---|---|---|---|
| accounts.customer_id → customers.id | account-service | customer-service | YES | Replace with plain scalar `customer_id` + read-model via CustomerRegistered/CustomerSuspended/CustomerClosed REST notification events |
| transactions.account_id → accounts.id | payments-service (transactions) | account-service (accounts) | NO — both owned by account-service | Keep as same-domain FK within account-service schema |
| payment_orders.customer_id | payments-service | customer-service | YES (already plain scalar in monolith) | Denormalised reference — `customer_id` + `customer_name` snapshot; no FK constraint |
| payment_orders.debit_account_number | payments-service | account-service | YES (already plain scalar in monolith) | Denormalised reference — account number string; no FK constraint |
| payment_orders.credit_account_number | payments-service | account-service | YES (already plain scalar in monolith) | Denormalised reference — account number string; no FK constraint |

---

## Tables Written by More Than One Domain

| Table | Written by |
|---|---|
| accounts | CustomerService (register/suspend/close) + AccountService (all account ops) |
| customers | CustomerService only |
| transactions | AccountService only |
| payment_orders | PaymentService only |

`accounts` is the only table written by more than one domain's service code. After decomposition, `accounts` is owned exclusively by account-service; customer-service communicates account state changes via REST notification events.
