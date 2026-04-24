# Phase 4f — Data Migration Plan

## Migration Sequence

The recommended sequence moves data from the monolith's shared H2 database into three separate domain databases, starting with the service that has the fewest cross-domain dependencies.

### Step 1 — Stand up domain databases
Provision one new empty database per domain service:
- `customer-db` for customer-service
- `account-db` for account-service
- `payments-db` for payments-service

### Step 2 — Run DDL scripts
Execute each service's DDL script against its target database:
1. `customer-service/src/main/resources/db/migration/V1__customer_schema.sql` → `customer-db`
2. `account-service/src/main/resources/db/migration/V1__account_schema.sql` → `account-db`
3. `payments-service/src/main/resources/db/migration/V1__payments_schema.sql` → `payments-db`

### Step 3 — Migrate data table by table (least-coupled first)

**Order:**
1. `customers` → `customer-db.customers`
   - Simple copy; no transformation required
   - Validate: row count match + checksum on `id`, `email`, `status`

2. `accounts` → `account-db.accounts`
   - **Transformation required**: `customer_id` column is retained as a plain scalar (no FK constraint in target schema). No data transformation needed — value is copied as-is.
   - Validate: row count match + checksum on `id`, `account_number`, `balance`

3. `transactions` → `account-db.transactions`
   - Simple copy; `account_id` FK is same-domain in target schema
   - Validate: row count match + checksum on `id`, `reference_number`, `amount`

4. `payment_orders` → `payments-db.payment_orders`
   - Simple copy; all cross-domain refs are already plain scalars in monolith
   - Validate: row count match + checksum on `id`, `order_reference`, `status`

### Step 4 — Validate
After each table migration:
- Compare row counts between source and target
- Spot-check 10 random rows per table for field-level accuracy
- Verify referential integrity within each service's own schema (e.g., `transactions.account_id` references valid `accounts.id` in `account-db`)

---

## Tables Requiring Data Transformation

| Table | Transformation | Notes |
|---|---|---|
| accounts | Remove FK constraint on `customer_id` | Value copied as-is; only the DDL constraint is dropped. No data transformation. |

All other tables: straight copy, no transformation.

---

## Large-Table Migration Risk

| Table | Estimated Row Count | Risk |
|---|---|---|
| customers | < 10M (dev/demo data) | Low |
| accounts | < 10M | Low |
| transactions | < 10M | Low |
| payment_orders | < 10M | Low |

No tables flagged as `large-table` migration risk (> 10M rows) based on current monolith data. If production data volumes exceed 10M rows in `transactions` or `payment_orders`, flag for incremental/online migration strategy before proceeding.

---

## Migration Approach Trade-offs (informational — decision left to the team)

| Approach | Pros | Cons |
|---|---|---|
| **Big-bang cutover** | Simple; single migration window; no dual-write complexity | Downtime required; high risk if validation fails; rollback is complex |
| **Parallel run** | Both monolith and microservices run simultaneously; compare outputs | Requires dual-write logic; data consistency is complex; higher operational overhead |
| **Dual-write period** | Gradual traffic shift; easy rollback to monolith | Write amplification; consistency window between old and new DBs; more complex application code |
| **Strangler Fig** | Incremental; low risk; monolith remains live | Longest timeline; requires routing layer (API gateway or proxy); team must maintain two codebases during transition |

The team should select the approach based on acceptable downtime window, rollback requirements, and operational maturity. This SOP does not prescribe a cutover strategy.

---

## Notes
- The monolith uses H2 in-memory (`create-drop`); there is no persistent data to migrate in the current dev/demo setup. This migration plan applies when the team switches to a persistent database (e.g., PostgreSQL) for production deployment.
- Each decomposed service's DDL includes a `schema_version` table for independent migration tracking.
