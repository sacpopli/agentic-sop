# Phase 1 — Codebase Inventory

## Parameters
| Key | Value |
|---|---|
| monolith_path | ../banking-monolith |
| output_path | ../decomposed_e2e_v3 |
| language | java |
| build_tool | Maven |
| framework_version | Spring Boot 3.2.0 |
| runtime_version | Java 17 |
| db_type | H2 (in-memory, same-as-monolith) |
| domain_model | Customer & Party Management, Product & Account Servicing, Payments & Financial Transactions |
| target_services | customer-service, account-service, payments-service |

---

## 1. Build Descriptor — `pom.xml`
| Dependency | Version / Scope |
|---|---|
| spring-boot-starter-parent | 3.2.0 (parent) |
| spring-boot-starter-web | managed by parent / compile |
| spring-boot-starter-data-jpa | managed by parent / compile |
| spring-boot-starter-security | managed by parent / compile |
| spring-boot-starter-validation | managed by parent / compile |
| com.h2database:h2 | managed by parent / runtime |
| org.projectlombok:lombok | managed by parent / optional |
| spring-boot-maven-plugin | managed by parent / build |

**No test framework dependency found → Phase 6 will be skipped.**

---

## 2. Package & Layer Structure
```
com.bank/
├── BankingApplication.java                  [entry-point]
├── config/
│   ├── DataInitializer.java                 [shared-kernel / omit — dev seed data]
│   ├── SecurityConfig.java                  [shared-kernel — cross-cutting security]
│   └── UserDetailsConfig.java               [shared-kernel — cross-cutting auth]
├── controller/
│   ├── AccountController.java               [account-service — Product & Account Servicing]
│   ├── CustomerController.java              [customer-service — Customer & Party Management]
│   └── PaymentController.java               [payments-service — Payments & Financial Transactions]
├── model/
│   ├── Account.java                         [account-service — Product & Account Servicing]
│   ├── Customer.java                        [customer-service — Customer & Party Management]
│   ├── PaymentOrder.java                    [payments-service — Payments & Financial Transactions]
│   └── Transaction.java                     [payments-service — Payments & Financial Transactions]
├── repository/
│   ├── AccountRepository.java               [account-service — Product & Account Servicing]
│   ├── CustomerRepository.java              [customer-service — Customer & Party Management]
│   ├── PaymentOrderRepository.java          [payments-service — Payments & Financial Transactions]
│   └── TransactionRepository.java           [payments-service — Payments & Financial Transactions]
├── service/
│   ├── AccountService.java                  [account-service + cross-domain calls from customer/payments]
│   ├── CustomerService.java                 [customer-service + cross-domain write to accounts]
│   └── PaymentService.java                  [payments-service + cross-domain calls to account-service]
└── util/
    ├── AuditLogger.java                     [shared-kernel]
    └── ReferenceGenerator.java              [shared-kernel]
```

---

## 3. API Entry Points
| HTTP Method | Path | Controller | Domain |
|---|---|---|---|
| POST | /api/customers | CustomerController.register | Customer & Party Management |
| GET | /api/customers/{id} | CustomerController.get | Customer & Party Management |
| PUT | /api/customers/{id}/kyc | CustomerController.verifyKyc | Customer & Party Management |
| PUT | /api/customers/{id}/profile | CustomerController.updateProfile | Customer & Party Management |
| PUT | /api/customers/{id}/suspend | CustomerController.suspend | Customer & Party Management |
| DELETE | /api/customers/{id} | CustomerController.close | Customer & Party Management |
| POST | /api/accounts | AccountController.open | Product & Account Servicing |
| GET | /api/accounts/{accountNumber} | AccountController.get | Product & Account Servicing |
| GET | /api/accounts/customer/{customerId} | AccountController.getByCustomer | Product & Account Servicing |
| POST | /api/accounts/{accountNumber}/debit | AccountController.debit | Product & Account Servicing |
| POST | /api/accounts/{accountNumber}/credit | AccountController.credit | Product & Account Servicing |
| POST | /api/accounts/{accountNumber}/interest | AccountController.applyInterest | Product & Account Servicing |
| DELETE | /api/accounts/{accountNumber} | AccountController.close | Product & Account Servicing |
| POST | /api/payments | PaymentController.initiate | Payments & Financial Transactions |
| PUT | /api/payments/{orderReference}/validate | PaymentController.validate | Payments & Financial Transactions |
| PUT | /api/payments/{orderReference}/execute | PaymentController.execute | Payments & Financial Transactions |
| GET | /api/payments/{orderReference} | PaymentController.status | Payments & Financial Transactions |
| PUT | /api/payments/{orderReference}/cancel | PaymentController.cancel | Payments & Financial Transactions |
| GET | /api/payments/customer/{customerId} | PaymentController.history | Payments & Financial Transactions |

No batch jobs, message listeners, or scheduled tasks found.

---

## 4. Entity Models & Enums

### `Customer` — table: `customers`
| Field | Type | Notes |
|---|---|---|
| id | Long | @Id @GeneratedValue IDENTITY |
| firstName | String | @NotBlank |
| lastName | String | @NotBlank |
| email | String | @Email @Column(unique=true) |
| phone | String | |
| nationalId | String | used for KYC |
| dateOfBirth | LocalDate | |
| status | CustomerStatus | @Enumerated(STRING) |
| type | CustomerType | @Enumerated(STRING) |
| createdAt | LocalDateTime | set in @PrePersist |
| updatedAt | LocalDateTime | set in @PrePersist / @PreUpdate |
| kycVerified | boolean | |
| kycVerifiedAt | LocalDateTime | |
| kycDocumentRef | String | |
| accounts | List\<Account\> | @OneToMany @JsonIgnore — cross-domain ref |

**Enums (verbatim):**
```java
public enum CustomerStatus { PENDING_KYC, ACTIVE, SUSPENDED, CLOSED }
public enum CustomerType   { RETAIL, CORPORATE, SME }
```

### `Account` — table: `accounts`
| Field | Type | Notes |
|---|---|---|
| id | Long | @Id @GeneratedValue IDENTITY |
| accountNumber | String | @Column(unique=true, nullable=false) |
| type | AccountType | @Enumerated(STRING) |
| status | AccountStatus | @Enumerated(STRING) |
| balance | BigDecimal | |
| overdraftLimit | BigDecimal | |
| currency | String | |
| customer | Customer | @ManyToOne @JoinColumn(customer_id) @JsonIgnore — cross-domain FK |
| openedAt | LocalDateTime | set in @PrePersist |
| closedAt | LocalDateTime | |
| updatedAt | LocalDateTime | set in @PrePersist / @PreUpdate |
| productCode | String | |
| interestRate | BigDecimal | |
| transactions | List\<Transaction\> | @OneToMany @JsonIgnore |

**Enums (verbatim):**
```java
public enum AccountType   { CURRENT, SAVINGS, DEPOSIT, LOAN, CREDIT }
public enum AccountStatus { ACTIVE, DORMANT, CLOSED, FROZEN }
```

### `PaymentOrder` — table: `payment_orders`
| Field | Type | Notes |
|---|---|---|
| id | Long | @Id @GeneratedValue IDENTITY |
| orderReference | String | @Column(unique=true, nullable=false) |
| type | PaymentType | @Enumerated(STRING) |
| status | PaymentStatus | @Enumerated(STRING) |
| amount | BigDecimal | |
| currency | String | |
| customerId | Long | plain scalar — no FK annotation |
| customerName | String | denormalised snapshot |
| debitAccountNumber | String | plain scalar ref to accounts |
| creditAccountNumber | String | plain scalar ref to accounts |
| beneficiaryName | String | |
| beneficiaryBank | String | |
| beneficiaryBankCode | String | BIC/SWIFT or ABA routing |
| remittanceInfo | String | |
| initiatedAt | LocalDateTime | set in @PrePersist |
| processedAt | LocalDateTime | |
| transactionId | Long | plain scalar ref to transactions |

**Enums (verbatim):**
```java
public enum PaymentType   { ACH, WIRE, INTERNAL, CARD }
public enum PaymentStatus { INITIATED, VALIDATED, PROCESSING, COMPLETED, FAILED, CANCELLED }
```

### `Transaction` — table: `transactions`
| Field | Type | Notes |
|---|---|---|
| id | Long | @Id @GeneratedValue IDENTITY |
| referenceNumber | String | @Column(unique=true, nullable=false) |
| type | TransactionType | @Enumerated(STRING) |
| status | TransactionStatus | @Enumerated(STRING) |
| amount | BigDecimal | |
| currency | String | |
| description | String | |
| account | Account | @ManyToOne @JoinColumn(account_id) @JsonIgnore |
| initiatedAt | LocalDateTime | set in @PrePersist |
| completedAt | LocalDateTime | |
| paymentRail | String | ACH, SWIFT, INTERNAL, CARD |
| counterpartyAccount | String | |
| counterpartyBank | String | |
| counterpartyName | String | |

**Enums (verbatim):**
```java
public enum TransactionType   { CREDIT, DEBIT }
public enum TransactionStatus { PENDING, COMPLETED, FAILED, REVERSED }
```

---

## 5. Database Schema

### Tables by domain
| Table | Domain | Notes |
|---|---|---|
| customers | Customer & Party Management | owned by customer-service |
| accounts | Product & Account Servicing | owned by account-service |
| payment_orders | Payments & Financial Transactions | owned by payments-service |
| transactions | Payments & Financial Transactions | owned by payments-service |

### Cross-domain foreign keys
| FK | From | To | Strategy |
|---|---|---|---|
| accounts.customer_id | Product & Account Servicing | Customer & Party Management | Replace with plain scalar `customerId` + read-model via CustomerProfileUpdated event |
| transactions.account_id | Payments & Financial Transactions | Product & Account Servicing | Replace with plain scalar `accountId` + denormalised ref |
| payment_orders.customerId | Payments & Financial Transactions | Customer & Party Management | Already plain scalar in monolith — no ORM FK |
| payment_orders.transactionId | Payments & Financial Transactions | Payments & Financial Transactions | Same domain — no change needed |

---

## 6. Configuration Properties
| Property | Value | Domain |
|---|---|---|
| spring.datasource.url | jdbc:h2:mem:bankdb;DB_CLOSE_DELAY=-1 | all services (each gets own DB) |
| spring.datasource.driver-class-name | org.h2.Driver | all services |
| spring.datasource.username | sa | all services |
| spring.datasource.password | (empty) | all services |
| spring.h2.console.enabled | true | all services (dev only) |
| spring.h2.console.path | /h2-console | all services (dev only) |
| spring.jpa.hibernate.ddl-auto | create-drop | all services |
| spring.jpa.show-sql | true | all services |
| spring.jpa.database-platform | org.hibernate.dialect.H2Dialect | all services |
| server.port | 8080 | monolith; each service gets own port |
| logging.level.com.bank | DEBUG | all services |

---

## 7. Dependency Map
```
CustomerController
  └─► CustomerService
        ├─► CustomerRepository          [Customer & Party Management]
        ├─► AccountRepository           [cross-domain] ← CustomerService writes accounts on register/suspend/close
        └─► AuditLogger                 [shared-kernel]

AccountController
  └─► AccountService
        ├─► AccountRepository           [Product & Account Servicing]
        ├─► CustomerRepository          [cross-domain] ← AccountService reads customer for KYC/status check
        ├─► TransactionRepository       [Product & Account Servicing]
        └─► AuditLogger                 [shared-kernel]

PaymentController
  └─► PaymentService
        ├─► PaymentOrderRepository      [Payments & Financial Transactions]
        ├─► AccountRepository           [cross-domain] ← PaymentService reads account for validation
        ├─► CustomerRepository          [cross-domain] ← PaymentService reads customer for status check
        ├─► AccountService              [cross-domain] ← PaymentService calls debitAccount/creditAccount
        └─► AuditLogger                 [shared-kernel]
```

Cross-domain calls summary:
- `CustomerService.registerCustomer` → writes to `AccountRepository` (creates default account) [cross-domain]
- `CustomerService.suspendCustomer` → writes to `AccountRepository` (freezes accounts) [cross-domain]
- `CustomerService.closeCustomer` → writes to `AccountRepository` (closes accounts) [cross-domain]
- `AccountService.openAccount` → reads `CustomerRepository` (KYC/status check) [cross-domain]
- `PaymentService.initiatePayment` → reads `CustomerRepository` + `AccountRepository` [cross-domain]
- `PaymentService.executePayment` → calls `AccountService.debitAccount` + `AccountService.creditAccount` [cross-domain]

---

## 8. Existing Tests
No `src/test/` directory found. No test framework dependency in build descriptor (`pom.xml` has no `junit`, `mockito`, `spring-boot-starter-test`, or similar). **Phase 6 will be skipped.**
