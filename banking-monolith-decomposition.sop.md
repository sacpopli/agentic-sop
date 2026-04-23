# Monolith Decomposition

## Overview

This SOP guides an agent through the full decomposition of a monolithic application into independently deployable microservices, each aligned to a set of industry-standard functional domains. The agent follows sequential phases — from codebase analysis through to scaffolding, test generation, and reporting.

The SOP is **domain-model agnostic and tech-stack agnostic**. The target domains, service names, and industry framework (e.g., BIAN for banking, eTOM for telecoms, ARTS for retail) are provided as input parameters. The tech-stack-specific scaffolding (build descriptors, entity generation, test setup) is handled by a separate implementation SOP referenced in Steps 5 and 6.

The SOP produces independently deployable units that can co-exist alongside the monolith. The decision of how and when to adopt a migration pattern (e.g., Strangler Fig, parallel run, big-bang cutover) is an architectural and operational decision made **outside this SOP** by the team owning the monolith.

---

## Parameters

Parameters can be supplied in two ways:
1. **Inline** — pass values directly when invoking the SOP
2. **Config file** — pass `config_file=./path/to/config.json` and the agent will read all parameters from that JSON file. Inline values override config file values when both are present.

**Config file format (`decomposition-config.json`):**
```json
{
  "monolith_path": "./my-app",
  "domain_model": "Customer & Party Management, Product & Account Servicing, Payments & Financial Transactions",
  "output_path": "./decomposed",
  "report_path": "./decomposition-report.md",
  "language": "java",
  "db_type": "postgresql",
  "test_framework": "junit5",
  "phase": "all"
}
```

- **config_file** (optional): Path to a JSON config file containing any of the parameters below. Useful for repeatable runs and version-controlled configurations.
- **monolith_path** (required): Root path of the monolith codebase to analyze
- **domain_model** (required): Description of the target domains to decompose into. Provide as a comma-separated list of domain names with their business scope. Example for banking (BIAN): `Customer & Party Management, Product & Account Servicing, Payments & Financial Transactions`. Example for telecoms (eTOM): `Customer Management, Product Management, Service Management`. The agent will use this to drive classification in Step 2.
- **output_path** (optional, default: `./decomposed`): Directory where scaffolded service directories will be created
- **report_path** (optional, default: `./decomposition-report.md`): Path for the final decomposition report
- **language** (optional, default: `java`): Primary programming language of the monolith (e.g., `java`, `python`, `nodejs`, `csharp`). Determines config file formats, source directory conventions, and which tech-stack implementation SOP to reference in Steps 5 and 6.
- **db_type** (optional, default: `same-as-monolith`): Target database engine for the decomposed service schemas. Defaults to the same database type detected in the monolith. Set explicitly to override — e.g., `postgresql`, `mysql`, `h2`, `oracle`, `mssql`.
- **test_framework** (optional, default: derived from `language`): Unit test framework — e.g., `junit5` or `testng` for Java; `pytest` for Python; `jest` or `mocha` for Node.js; `xunit` or `nunit` for C#.
- **phase** (optional, default: `all`): Which phase to execute — `1`, `2`, `3`, `4`, `5`, `6`, `7`, or `all`

**Constraints for parameter acquisition:**
- You MUST read `config_file` first if provided, then apply any inline overrides on top
- You MUST ask for all required parameters upfront in a single prompt rather than one at a time
- You MUST confirm the monolith_path exists and is readable before proceeding
- You MUST NOT proceed if monolith_path is empty or inaccessible because subsequent steps depend entirely on reading the actual codebase
- You MUST derive the list of target service names from `domain_model` — convert each domain name to a kebab-case service name (e.g., "Customer & Party Management" → `customer-service`)

---

## Steps

### 1. Codebase Inventory

Read and map the full structure of the monolith codebase at `monolith_path`.

**Constraints:**
- You MUST list all top-level modules, packages, and layers (e.g., controllers, services, repositories, models)
- You MUST identify all API entry points: REST controllers, batch jobs, message listeners, scheduled tasks
- You MUST document all database schemas, entity models, and shared data objects found in the codebase
- You MUST list every `enum` type found in the codebase with its exact declared values — copy them verbatim from the source file; do NOT infer, rename, or paraphrase enum constants
- You MUST build a dependency map showing which modules call which other modules
- You SHOULD identify the build system (Maven, Gradle, npm, etc.) and note any existing module boundaries
- You MUST inventory all externalised configuration files appropriate for `language` — for Java: `application.yml`, `application.properties`, `application-{profile}.yml`, `logback.xml`, `log4j2.xml`, `persistence.xml`; for Python: `settings.py`, `.env`, `config.yaml`; for Node.js: `.env`, `config.json`, `config.js`; for C#: `appsettings.json`, `appsettings.{env}.json` — and record every property key-value pair found, tagged with the domain it belongs to
- You MUST save the inventory as `{output_path}/phase1-inventory.md` before proceeding to Step 2
- You MUST NOT skip any subdirectory during the scan because missing a module could lead to incorrect domain assignments in later steps

### 2. Domain Classification

Classify every module, class, and database table to one or more domains from `domain_model`, then determine the appropriate decomposition action for each.

**Classification rules:**
- You MUST derive classification rules from the `domain_model` parameter — map each domain's business scope to the code elements that implement it
- You MUST read the monolith source to understand what each class/module does before assigning it to a domain — do NOT classify based on naming conventions alone
- Auth, logging, config, common DTOs, error handling → `shared-kernel` regardless of domain model

**Constraints:**
- You MUST analyse each module/class at the method level, not just the class level, because a single class may contain methods that belong to different domains
- You MUST assign one of the following decomposition actions to every module/class:

  | Action | When to use |
  |---|---|
  | `move` | All methods in the class belong to a single domain — move the whole class as-is |
  | `split` | Methods in the class belong to 2 or more domains — the class must be broken apart into domain-specific classes |
  | `extract-method` | A single method contains interleaved logic for multiple domains — the method must be refactored before it can be moved |
  | `shared-kernel` | The class has zero domain-specific business logic (auth, logging, config, DTOs) — move to shared library |
  | `facade` | The class is a public entry point (controller, API handler) that orchestrates multiple domains — flag for review; the team will decide whether to keep it in the monolith, split it, or replace it as part of their chosen migration strategy |

- You MUST produce the classification as a markdown table with columns: `Module/Class | Methods | Domains Touched | Action | Target Service(s) | Notes`
- For every class marked `split`, you MUST list which methods go to which target service
- For every class marked `extract-method`, you MUST identify the specific method(s) with interleaved logic and describe the proposed split (e.g., "lines 45–80 handle payment validation → payments-service; lines 81–110 update account balance → account-service")
- You MUST flag any method where domain boundary is genuinely unclear as `ambiguous` and document the reason and a recommended ownership decision
- You MUST save the full classification matrix as `{output_path}/phase2-classification.md`
- You MUST NOT assign a class to `shared-kernel` unless it has zero domain-specific business logic because shared-kernel bloat defeats the purpose of decomposition
- You MUST NOT mark a class as `move` without first verifying all its methods belong to the same domain because incorrect moves create hidden cross-domain coupling in the target service

### 3. Service Boundary & Contract Definition

Define what each domain service owns and how the services communicate.

**Constraints:**
- You MUST define the aggregate roots and owned entities for each domain derived from `domain_model`
- You MUST define data ownership: each database table MUST be assigned to exactly one domain
- You MUST identify all cross-domain data reads and propose whether they are resolved via a synchronous REST API call (query) or a REST notification callback (state change)
- You MUST derive the inter-domain contracts by analysing the actual cross-domain calls found in the monolith codebase in Step 1 — do NOT invent contracts that don't reflect real dependencies in the code
- For each cross-domain dependency found, define:
  - **Synchronous REST API** if the caller needs a real-time response (e.g., validation before proceeding)
  - **REST Notification Callback** (`POST /internal/events`) if the caller only needs to inform the other service of a state change
- You MUST generate a `DomainEvent` DTO in `shared-kernel` with fields: `eventId` (UUID), `type` (String), `occurredAt` (ISO8601), `payload` (Map)
- You MUST generate an `EventPublisher` class per service that calls the target service's `/internal/events` endpoint using the HTTP client appropriate for `language`
- You MUST generate an `InternalEventController` (or equivalent handler) per service that receives `POST /internal/events` and routes to the appropriate handler based on `event.type`
- You SHOULD flag any notification callback that could cause a circular call chain and document that the receiver MUST NOT re-notify the original sender
- You MUST save the contract definitions as `{output_path}/phase3-contracts.md`
- You MUST NOT design shared databases between services because shared databases create tight coupling that defeats microservice independence

### 4. Database Decomposition

Decompose the monolith's shared database into domain-owned schemas, one per deployable service. This step produces the migration SQL and the read-model strategy for cross-domain data access.

**4a — Inventory the Shared Schema**

**Constraints:**
- You MUST list every table in the monolith database with its columns, primary keys, foreign keys, and indexes
- You MUST identify every foreign key that crosses a domain boundary (e.g., `accounts.customer_id` references `customers.id` — crosses Customer → Account boundary)
- You MUST identify every table that is written to by more than one domain's service code
- You MUST save the shared schema inventory as `{output_path}/phase4a-shared-schema.md`

**4b — Assign Table Ownership**

**Constraints:**
- You MUST assign every table to exactly one owning domain — derive ownership by analysing which domain's service code writes to each table (identified in Step 1)
- You MUST NOT use a fixed ownership table — ownership must be determined from the actual codebase, not assumed from table names
- You MUST NOT leave any table assigned to multiple domains because shared table ownership is the primary source of deployment coupling
- Tables written exclusively by cross-cutting concerns (audit logs, system events) MUST be assigned to `shared-kernel` and replicated to each service's own schema
- For every cross-domain foreign key found in 4a, you MUST replace it with one of:
  - A **denormalised reference column** (store the remote ID + a snapshot of key fields, no FK constraint) — use when the remote data changes rarely
  - A **read-model table** (local copy of remote data, kept in sync via REST notification callbacks) — use when the remote data is queried frequently
  - A **synchronous API call** at runtime — use only when freshness is critical and latency is acceptable
- You MUST document the chosen strategy for each cross-domain FK in `{output_path}/phase4b-table-ownership.md`

**4c — Generate Domain Schema DDL**

For each domain database, generate a standalone DDL script.

**Constraints:**
- You MUST write each DDL file into the appropriate resources directory inside each service directory, following the convention for `language`:
  - Java/Maven: `{output_path}/{service-name}/src/main/resources/db/migration/V1__{service}_schema.sql`
  - Python: `{output_path}/{service-name}/migrations/V1__{service}_schema.sql`
  - Node.js: `{output_path}/{service-name}/migrations/V1__{service}_schema.sql`
  - C#: `{output_path}/{service-name}/Migrations/V1__{service}_schema.sql`
- You MUST NOT place generated schema files inside the monolith codebase
- Each DDL file MUST be self-contained and executable against a fresh `{db_type}` database with no dependencies on another service's schema
- You MUST add a comment on every column that replaces a cross-domain FK explaining what it references and how it is kept in sync
- You MUST NOT include cross-database foreign key constraints because they create hard runtime coupling between services
- You MUST include a schema version tracking table so migrations can be tracked independently per service

**4d — Generate Domain Schema DDL (dialect-specific)**

Generate the same DDL as 4c but using syntax appropriate for `{db_type}`.

**Constraints:**
- You MUST write each DDL file into the service's resources directory following the `language` convention from Step 4c
- You MUST NOT place generated schema files inside the monolith codebase
- Each DDL file MUST be self-contained and executable against a fresh `{db_type}` database
- You MUST NOT include cross-database foreign key constraints
- You MUST include a schema version tracking table per service

**4e — Define Data Migration Options**

**Constraints:**
- You MUST document the data migration sequence needed to move data from the monolith's shared database into the domain databases, should the team choose to do so:
  1. Stand up one new empty database per domain service
  2. Run each domain DDL script against its target database
  3. Migrate data table by table, starting with the domain that has the fewest cross-domain dependencies
  4. Validate row counts and checksums after each table migration
- You MUST identify any tables that require a data transformation (not just a copy) during migration and describe the transformation
- You MUST flag any table with more than 10 million rows as a `large-table` migration risk requiring an incremental/online migration strategy
- You MUST save the migration options as `{output_path}/phase4e-migration-plan.md`
- You MUST NOT prescribe a specific cutover strategy (dual-write, Strangler Fig, big-bang, parallel run) because that decision is made outside this SOP by the team owning the monolith
- You SHOULD document the trade-offs of common migration approaches (dual-write period, parallel run, big-bang) as informational notes so the team has the context to decide

### 5. Scaffold Deployable Units

Create the skeleton directory structure for each domain service derived from `domain_model`, plus the shared kernel. This step is tech-stack agnostic — the actual build descriptor, source layout, and config file format depend on `language`. For tech-stack-specific scaffolding refer to the implementation SOP for the chosen stack.

**Constraints:**
- You MUST create one service directory per domain in `domain_model`, plus a `shared-kernel` directory:

```
{output_path}/
├── {service-1}/
│   ├── src/
│   │   ├── main/
│   │   └── test/                       ← populated by Step 6
│   ├── resources/
│   │   └── application.{ext}
│   ├── db/
│   │   └── V1__{service-1}_schema.sql  ← generated by Step 4c
│   ├── Dockerfile
│   ├── {build-descriptor}
│   └── README.md
├── {service-2}/
│   └── ...                             ← same structure
├── {service-N}/
│   └── ...                             ← one per domain in domain_model
├── shared-kernel/
│   ├── src/
│   │   ├── main/
│   │   └── test/
│   ├── {build-descriptor}
│   └── README.md
└── docker-compose.yml                  ← N services + N separate DB instances
```

- You MUST run Step 4 (Database Decomposition) before this step because the `db/` directories depend on the DDL files generated in Step 4d
- You MUST NOT create the `db/` directories with placeholder content — they MUST contain the actual generated DDL from Step 4d
- You MUST populate each `README.md` with: domain name, BIAN service domains it covers, owned entities, REST notification events published and consumed, and exposed API endpoints
- You MUST create a configuration file for each service (format depends on `language` and framework) containing: database connection URL, credentials, and the base URLs of downstream services this service calls
- You MUST create a local development configuration override for each service that uses an in-memory or embedded database so the service can start without a running database server
- You MUST NOT put shared database connection strings in service configs because each service must own its own data store
- You MUST add service URL properties to each service's config for every downstream service it calls so the `EventPublisher` and cross-domain clients can resolve target URLs via configuration
- You MUST generate a build descriptor for each service appropriate to `language` that includes all dependencies required to: run the web server, connect to the database, perform schema migration, call other services via HTTP, and run unit tests. Refer to the tech-stack-specific implementation SOP for the exact dependency list.

### 6. Unit Test Generation

**Pre-condition check — MUST be done before generating any tests:**
- You MUST scan `{monolith_path}` for existing test files (e.g., `*Test.java`, `*_test.py`, `*.test.js`, `*.spec.ts`, `*Tests.cs`)
- If **no test files are found** in the monolith, you MUST skip this entire step and note in the report that the monolith had no unit tests and therefore none were generated for the decomposed services
- If **test files are found**, proceed with the steps below — the decomposed services should have tests that mirror the coverage and style of the monolith's existing tests

For every class moved or generated into a decomposed service in Step 5, generate a corresponding unit test class covering all public methods. Tests must be runnable in isolation without depending on other services or shared databases.

**6a — Service Layer Tests**

**Constraints:**
- You MUST generate one test class per service class, named `{ClassName}Test`, placed at the mirrored path under the test source root
- You MUST cover the following test categories for every public method:
  - **Happy path** — valid inputs, expected output or state change
  - **Validation / guard clause** — invalid or missing inputs that should throw exceptions
  - **Domain boundary** — verify the method does NOT directly call another domain's repository or service (assert via mock verification that cross-domain calls go through the defined contract interface only)
- You MUST mock all external dependencies (repositories, cross-domain API clients, event publishers) using the mocking library appropriate for `test_framework`
- You MUST NOT use a real database in unit tests
- You MUST NOT call other services' real implementations in unit tests
- You SHOULD aim for at least 80% line coverage per service class

**6b — Repository / Data Access Layer Tests**

**Constraints:**
- You MUST generate one test class per repository or data access class
- You MUST use an in-memory or embedded database appropriate for `language` and `test_framework`
- You MUST cover: save, findById, findBy custom query methods, and delete operations
- You MUST verify that queries return correct results for both existing and non-existing records
- You MUST NOT connect to the production or shared monolith database

**6c — Controller / API Layer Tests**

**Constraints:**
- You MUST generate one test class per controller or API handler
- You MUST use a lightweight HTTP test harness appropriate for `language`
- You MUST cover for every endpoint: success (200/201), bad request (400), not found (404), and conflict (409) where applicable
- You MUST mock the service layer so controller tests do not execute business logic
- You MUST verify response body structure, not just status codes

**6d — REST Notification Tests**

**Constraints:**
- You MUST generate tests that verify the correct `DomainEvent` payload is sent when a domain action completes
- You MUST mock the HTTP client in `EventPublisher` tests — do not make real HTTP calls
- You MUST generate tests for the notification receiver endpoint that POST a `DomainEvent` body and verify the correct state change occurs
- You MUST verify the failure path: target service unreachable — the publisher MUST log a warning and NOT throw an exception that would roll back the originating transaction

**6e — Shared Kernel Tests**

**Constraints:**
- You MUST generate tests for all public methods in shared utilities
- You MUST NOT test shared kernel classes inside a domain service's test suite — they belong in `shared-kernel/src/test/`

**Test file naming and location summary:**

| Module | Test location | Naming convention |
|---|---|---|
| `{service-1}` | `{service-1}/src/test/` | `{ClassName}Test.{ext}` |
| `{service-N}` | `{service-N}/src/test/` | `{ClassName}Test.{ext}` |
| `shared-kernel` | `shared-kernel/src/test/` | `{ClassName}Test.{ext}` |

### 7. Generate Decomposition Report

Produce a comprehensive report summarising all findings and recommended next steps.

**Constraints:**
- You MUST write the report to `report_path`
- You MUST include the following sections in the report:
  1. **Executive Summary** — monolith size, number of modules classified, target services identified (one per domain in `domain_model`)
  2. **Domain Classification Matrix** — full table from Phase 2
  3. **Ambiguous Modules** — list of flagged items with recommended ownership decisions
  4. **Inter-Domain Contracts** — synchronous REST query APIs and REST notification callback contracts (DomainEvent payloads per event type) from Phase 3
  5. **Database Decomposition Summary** — table ownership assignments, cross-domain FK resolutions, and read-model decisions from Phase 4
  6. **Data Migration Options** — migration sequence, large-table risks, and trade-off notes on migration approaches (dual-write, parallel run, big-bang) from Phase 4e — decision on approach left to the team
  7. **Shared Kernel Contents** — list of cross-cutting components extracted
  8. **Recommended Decomposition Order** — suggested sequence for scaffolding services based on coupling analysis (least coupled first), presented as a recommendation not a prescription
  9. **Risk Register** — at minimum cover: data consistency, distributed transactions, network latency, team silos, large-table migration complexity
  10. **Definition of Done checklist** — per service: code extracted, own DB schema with migration scripts, REST API contract documented, REST notification contract documented (DomainEvent payloads), unit tests generated with >=80% coverage, CI/CD pipeline (including test execution gate), observability, integration tests passing
  11. **Recommended Next Steps** — prioritised action list
- You MUST NOT omit the Risk Register section because unaddressed risks are the primary cause of failed decomposition projects
- You SHOULD include estimated effort (S/M/L) for each recommended next step

---

## Examples

### Example invocations in Kiro CLI

Banking monolith (BIAN domains):
```
run monolith-decomposition.sop.md with monolith_path=./banking-app language=java domain_model="Customer & Party Management, Product & Account Servicing, Payments & Financial Transactions"
```

Telecoms monolith (eTOM domains):
```
run monolith-decomposition.sop.md with monolith_path=./telecom-app language=python domain_model="Customer Management, Product Management, Service Management, Resource Management"
```

Retail monolith (ARTS domains):
```
run monolith-decomposition.sop.md with monolith_path=./retail-app language=nodejs domain_model="Customer, Inventory, Order Management, Payment"
```

### Example classification table output (Phase 2)

The domain names in the table come from `domain_model` — not hardcoded:

| Module / Class | Methods | Domains Touched | Action | Target Service(s) | Notes |
|---|---|---|---|---|---|
| `CustomerController` | `createCustomer`, `getCustomer` | Domain 1 | `move` | domain-1-service | All methods belong to Domain 1 |
| `OrderService` | `createOrder`, `processPayment` | Domain 3, Domain 4 | `split` | domain-3-service, domain-4-service | `createOrder` → domain-3; `processPayment` → domain-4 |
| `OnboardingService` | `register`, `openAccount`, `setupProfile` | Domain 1, Domain 2, Domain 3 | `split` | all domain services | Orchestration — flagged as `facade` |
| `AuditLogger` | `log`, `logError` | none (cross-cutting) | `shared-kernel` | shared-kernel | No domain business logic |

### Example REST notification contract (Phase 3)

The event names and routes are derived from actual cross-domain calls found in the monolith — not hardcoded:

```
POST /internal/events
Host: {target-service}:{port}
Content-Type: application/json

{
  "eventId":    "a3f1c2d4-...",
  "type":       "{EventName}",
  "occurredAt": "2026-04-22T10:15:30Z",
  "payload": {
    "entityId":  123,
    "field1":    "value1",
    "field2":    "value2"
  }
}
```

| Event | Publisher | Receiver | Action on receipt |
|---|---|---|---|
| `{EntityCreated}` | domain-1-service | domain-2-service | Update read model |
| `{StatusChanged}` | domain-2-service | domain-3-service | Trigger downstream workflow |

---

## Troubleshooting

### A single method mixes domain logic with no clean split point
If a method is so deeply interleaved that it cannot be split without a significant rewrite, you MUST mark it `extract-method`, describe the interleaved sections in the notes column, and add it to the Risk Register in the final report as a refactoring prerequisite. You MUST NOT move the method as-is into a target service because it will silently import cross-domain coupling into the new service.

### Monolith has no clear package structure
If the codebase has no module boundaries (everything in one package), you MUST classify at the class level rather than the module level. Use class names and method signatures to infer domain ownership.

### A database table is used by all 3 domains
If a table (e.g., `audit_log`, `notifications`) is read/written by multiple domains, you MUST assign ownership to the domain that writes to it most, and expose read access to other domains via API. Document this decision in the report.

### Circular dependencies between domains
If domain A calls domain B and domain B calls domain A for the same event (e.g., account-service notifies customer-service of `AccountOpened`, and customer-service then notifies account-service back), you MUST add a guard in the receiver's `InternalEventController` to detect and skip re-notification of the original sender. Document the circular chain and the guard logic in the contracts file.

### Language not supported by default Dockerfile base image
If `language` is not one of `java`, `python`, `csharp`, `nodejs`, you SHOULD ask the user to confirm the correct base image before creating Dockerfiles.

### A table has a circular foreign key dependency between domains
If table A (domain X) has a FK to table B (domain Y) and table B also has a FK back to table A, you MUST break one of the FKs by converting it to a denormalised reference column. Document which FK was broken and why in phase4b-table-ownership.md.

### A shared table cannot be cleanly assigned to one domain
If a table is genuinely co-owned (e.g., a `notifications` table written by multiple domains), you MUST split it into domain-specific tables (e.g., `{domain1}_notifications`, `{domain2}_notifications`) rather than keeping a shared table. Each domain owns and writes only to its own table.

### Large table migration exceeds acceptable downtime window
If a table has more than 10 million rows, you MUST flag it as a `large-table` migration risk in the report. You SHOULD document available incremental migration options (e.g., `pt-online-schema-change`, `gh-ost`, application-level backfill) as informational notes. You MUST NOT prescribe which approach to use because that decision depends on the team's operational constraints and chosen migration strategy.
