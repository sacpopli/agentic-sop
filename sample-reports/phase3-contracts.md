# Phase 3 — Service Boundary & Contract Definition

## 1. Aggregate Roots & Owned Entities
| Service | Aggregate Root | Owned Entities / Tables |
|---|---|---|
| customer-service | Customer | customers |
| account-service | Account | accounts, transactions |
| payments-service | PaymentOrder | payment_orders |

---

## 2. Data Ownership
| Table | Owner Service |
|---|---|
| customers | customer-service |
| accounts | account-service |
| transactions | account-service |
| payment_orders | payments-service |

---

## 3. Cross-Domain Dependencies & Resolution Strategy
| Caller | Callee | Monolith call | Resolution |
|---|---|---|---|
| customer-service | account-service | `CustomerService.registerCustomer` → `accountRepository.save(defaultAccount)` | REST Notification: `CustomerRegisteredEvent` → account-service creates default account |
| customer-service | account-service | `CustomerService.suspendCustomer` → `accountRepository.save(account.FROZEN)` | REST Notification: `CustomerSuspendedEvent` → account-service freezes all accounts for customerId |
| customer-service | account-service | `CustomerService.closeCustomer` → `accountRepository.save(account.CLOSED)` | REST Notification: `CustomerClosedEvent` → account-service closes all accounts for customerId (account-service validates zero balance) |
| account-service | customer-service | `AccountService.openAccount` → `customerRepository.findById` (KYC + status check) | Synchronous REST GET `/api/customers/{customerId}` — freshness required before account creation |
| payments-service | customer-service | `PaymentService.initiatePayment` → `customerRepository.findById` (status check) | Synchronous REST GET `/api/customers/{customerId}` — freshness required before payment initiation |
| payments-service | account-service | `PaymentService.initiatePayment` → `accountRepository.findByAccountNumber` (status + ownership check) | Synchronous REST GET `/api/accounts/{accountNumber}` — freshness required before payment initiation |
| payments-service | account-service | `PaymentService.executePayment` → `accountService.debitAccount(...)` | Synchronous REST POST `/api/accounts/{accountNumber}/debit` — real-time debit required |
| payments-service | account-service | `PaymentService.executePayment` → `accountService.creditAccount(...)` | Synchronous REST POST `/api/accounts/{accountNumber}/credit` — real-time credit required |

---

## 4. Synchronous REST APIs (Query Contracts)

### customer-service exposes (consumed by account-service and payments-service)

**GET /api/customers/{customerId}**
- Request: path variable `customerId` (Long)
- Response 200:
```json
{
  "id": 1,
  "firstName": "Alice",
  "lastName": "Smith",
  "email": "alice@example.com",
  "status": "ACTIVE",
  "kycVerified": true,
  "type": "RETAIL"
}
```
- Response 404: `{ "error": "Customer not found: {customerId}" }`

### account-service exposes (consumed by payments-service)

**GET /api/accounts/{accountNumber}**
- Request: path variable `accountNumber` (String)
- Response 200:
```json
{
  "id": 1,
  "accountNumber": "ACC20260423000001",
  "type": "CURRENT",
  "status": "ACTIVE",
  "balance": 1000.00,
  "currency": "USD",
  "customerId": 1
}
```
- Response 404: `{ "error": "Account not found: {accountNumber}" }`

**POST /api/accounts/{accountNumber}/debit**
- Request body:
```json
{
  "amount": 500.00,
  "currency": "USD",
  "description": "Payment: remittance info",
  "paymentRail": "INTERNAL",
  "counterpartyAccount": "ACC20260423000002",
  "counterpartyName": "Bob Jones"
}
```
- Response 200: Transaction object
- Response 400/409: error message

**POST /api/accounts/{accountNumber}/credit**
- Request body: same shape as debit
- Response 200: Transaction object
- Response 400/409: error message

---

## 5. REST Notification Contracts (DomainEvent payloads)

### Shared DomainEvent DTO (shared-kernel)
```java
package com.bank.shared.event;

import lombok.Data;
import java.time.Instant;
import java.util.Map;
import java.util.UUID;

@Data
public class DomainEvent {
    private String eventId = UUID.randomUUID().toString();
    private String type;
    private String occurredAt = Instant.now().toString();
    private Map<String, Object> payload;
}
```

### Event Catalogue
| Event | Publisher | Receiver | Trigger | Payload fields |
|---|---|---|---|---|
| `CustomerRegistered` | customer-service | account-service | `CustomerService.registerCustomer` completes | `customerId`, `firstName`, `lastName`, `currency` (default "USD"), `productCode` (default "CURR-STD-001") |
| `CustomerSuspended` | customer-service | account-service | `CustomerService.suspendCustomer` completes | `customerId`, `reason` |
| `CustomerClosed` | customer-service | account-service | `CustomerService.closeCustomer` completes | `customerId` |

### Notification routing table
```
POST /internal/events → account-service:
  type=CustomerRegistered  → create default CURRENT account for customerId
  type=CustomerSuspended   → freeze all ACTIVE accounts for customerId
  type=CustomerClosed      → close all accounts for customerId (validate zero balance first)
```

### Circular call chain check
- `customer-service` → `account-service` (CustomerRegistered, CustomerSuspended, CustomerClosed): account-service MUST NOT re-notify customer-service on receipt of these events. No circular chain.
- `account-service` → `customer-service`: account-service makes synchronous REST reads only (GET /api/customers/{id}); it does not send notification events to customer-service. No circular chain.
- `payments-service` → `account-service`: payments-service makes synchronous REST calls only (GET + POST); it does not send notification events. No circular chain.

---

## 6. EventPublisher Stub (customer-service)
```java
package com.bank.customer.event;

import com.bank.shared.event.DomainEvent;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;
import org.springframework.web.client.RestTemplate;

@Component
@RequiredArgsConstructor
@Slf4j
public class CustomerEventPublisher {

    private final RestTemplate restTemplate;

    @Value("${services.account-service.base-url}")
    private String accountServiceBaseUrl;

    public void publish(DomainEvent event) {
        try {
            restTemplate.postForEntity(
                accountServiceBaseUrl + "/internal/events", event, Void.class);
        } catch (Exception e) {
            log.warn("Failed to publish event {} to account-service: {}", event.getType(), e.getMessage());
            // MUST NOT rethrow — failed notification must not roll back originating transaction
        }
    }
}
```

---

## 7. InternalEventController Stub (account-service)
```java
package com.bank.account.controller;

import com.bank.shared.event.DomainEvent;
import com.bank.account.service.AccountEventHandler;
import lombok.RequiredArgsConstructor;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/internal/events")
@RequiredArgsConstructor
public class InternalEventController {

    private final AccountEventHandler eventHandler;

    @PostMapping
    public ResponseEntity<Void> receive(@RequestBody DomainEvent event) {
        switch (event.getType()) {
            case "CustomerRegistered" -> eventHandler.onCustomerRegistered(event);
            case "CustomerSuspended"  -> eventHandler.onCustomerSuspended(event);
            case "CustomerClosed"     -> eventHandler.onCustomerClosed(event);
            default -> log.warn("Unknown event type received: {}", event.getType());
        }
        return ResponseEntity.ok().build();
    }
}
```

---

## 8. Cross-Domain REST Clients

### AccountServiceClient (used by payments-service)
```java
package com.bank.payments.client;

import com.bank.payments.dto.AccountDto;
import com.bank.payments.dto.TransactionDto;
import lombok.RequiredArgsConstructor;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;
import org.springframework.web.client.RestTemplate;
import java.util.Map;

@Component
@RequiredArgsConstructor
public class AccountServiceClient {

    private final RestTemplate restTemplate;

    @Value("${services.account-service.base-url}")
    private String baseUrl;

    public AccountDto getAccount(String accountNumber) {
        return restTemplate.getForObject(baseUrl + "/api/accounts/" + accountNumber, AccountDto.class);
    }

    public TransactionDto debitAccount(String accountNumber, Map<String, Object> body) {
        return restTemplate.postForObject(
            baseUrl + "/api/accounts/" + accountNumber + "/debit", body, TransactionDto.class);
    }

    public TransactionDto creditAccount(String accountNumber, Map<String, Object> body) {
        return restTemplate.postForObject(
            baseUrl + "/api/accounts/" + accountNumber + "/credit", body, TransactionDto.class);
    }
}
```

### CustomerServiceClient (used by account-service and payments-service)
```java
package com.bank.account.client;   // also replicated in com.bank.payments.client

import com.bank.shared.dto.CustomerDto;
import lombok.RequiredArgsConstructor;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;
import org.springframework.web.client.RestTemplate;

@Component
@RequiredArgsConstructor
public class CustomerServiceClient {

    private final RestTemplate restTemplate;

    @Value("${services.customer-service.base-url}")
    private String baseUrl;

    public CustomerDto getCustomer(Long customerId) {
        return restTemplate.getForObject(baseUrl + "/api/customers/" + customerId, CustomerDto.class);
    }
}
```
