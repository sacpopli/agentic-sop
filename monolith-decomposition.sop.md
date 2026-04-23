# Monolith Decomposition

## Overview

This SOP guides an agent through the full decomposition of a monolithic application into independently deployable microservices, each aligned to a set of industry-standard functional domains. The agent follows sequential phases — from codebase analysis through to scaffolding, test generation, and reporting.

The SOP is **domain-model agnostic and tech-stack agnostic**. The target domains, service names, and industry framework (e.g., BIAN for banking, eTOM for telecoms, ARTS for retail) are provided as input parameters. The `language` parameter drives all code generation conventions — build descriptors, config files, HTTP clients, ORM patterns, and test frameworks are selected based on what the monolith already uses.

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
- **language** (optional, default: `java`): Primary programming language of the monolith (e.g., `java`, `python`, `nodejs`, `csharp`). This is the key driver for all code generation in Step 5 — it determines build descriptor format, config file format, HTTP client library, ORM style, and test framework. Per-language conventions are inlined in each step.

**Per-language quick reference:**

| Concern | `java` | `python` | `nodejs` | `csharp` |
|---|---|---|---|---|
| Build descriptor | `pom.xml` / `build.gradle` | `pyproject.toml` / `requirements.txt` | `package.json` | `*.csproj` |
| Config file | `application.yml` | `settings.py` / `.env` / `config.yaml` | `.env` / `config.json` | `appsettings.json` |
| HTTP client | `RestTemplate` / `WebClient` | `requests` / `httpx` | `axios` / `node-fetch` | `HttpClient` |
| ORM / data model | JPA `@Entity` | SQLAlchemy / Django Model | TypeORM / Sequelize | EF Core entity |
| Test framework (default) | JUnit5 + Mockito | pytest + unittest.mock | Jest / Mocha | xUnit / NUnit |
| Test file pattern | `*Test.java` | `test_*.py` / `*_test.py` | `*.test.js` / `*.spec.ts` | `*Tests.cs` |
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

Define what each domain service owns and how the services communicate. This step produces the contract specification that Step 5 implements as code.

**Constraints:**
- You MUST define the aggregate roots and owned entities for each domain derived from `domain_model`
- You MUST define data ownership: each database table MUST be assigned to exactly one domain
- You MUST identify all cross-domain data reads and propose whether they are resolved via a synchronous REST API call (query) or a REST notification callback (state change)
- You MUST derive the inter-domain contracts by analysing the actual cross-domain calls found in the monolith codebase in Step 1 — do NOT invent contracts that don't reflect real dependencies in the code
- For each cross-domain dependency found, define:
  - **Synchronous REST API** if the caller needs a real-time response (e.g., validation before proceeding)
  - **REST Notification Callback** (`POST /internal/events`) if the caller only needs to inform the other service of a state change

**Three artifacts to specify for code generation in Step 5:**

**1. Shared Event DTO** — a data transfer object representing a domain event, placed in `shared-kernel`. Fields: `eventId` (UUID), `type` (String), `occurredAt` (ISO8601 timestamp), `payload` (key-value map). Per-language implementation:

| language | Implementation |
|---|---|
| `java` | Plain class with `@Data` (Lombok) or getters/setters; serialised to JSON via Jackson |
| `python` | `dataclass` or `TypedDict`; serialised via `json` stdlib or `pydantic` |
| `nodejs` | Plain JS/TS object or interface; serialised via `JSON.stringify` |
| `csharp` | POCO class with auto-properties; serialised via `System.Text.Json` |

**2. Notification Sender (EventPublisher)** — one per service; sends `POST /internal/events` to downstream services when a state change occurs. Per-language implementation:

| language | HTTP client to use |
|---|---|
| `java` | `RestTemplate` or `WebClient` — whichever is already in the monolith's build descriptor |
| `python` | `requests` or `httpx` — whichever is already in `requirements.txt` |
| `nodejs` | `axios`, `node-fetch`, or built-in `https` — whichever is already in `package.json` |
| `csharp` | `HttpClient` from `System.Net.Http` — already in .NET BCL |

The sender MUST catch all HTTP errors and log a warning — it MUST NOT throw, because a failed notification must not roll back the originating transaction.

**3. Notification Receiver (InternalEventController)** — one per service; exposes `POST /internal/events`, deserialises the event DTO, and routes to the appropriate handler based on `event.type`. Per-language implementation:

| language | Implementation |
|---|---|
| `java` | `@RestController` with `@PostMapping("/internal/events")` |
| `python` | Route handler in Flask/FastAPI/Django — e.g., `@app.post("/internal/events")` |
| `nodejs` | Express/Fastify/Koa route — e.g., `router.post('/internal/events', handler)` |
| `csharp` | Controller action with `[HttpPost("internal/events")]` |

- You SHOULD flag any notification callback that could cause a circular call chain and document that the receiver MUST NOT re-notify the original sender
- You MUST save the contract definitions (event names, payloads, sender→receiver routing) as `{output_path}/phase3-contracts.md`
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

**4c — Generate Domain Data Model Classes**

For each table owned by a domain (from Step 4b), generate the corresponding data model class in the service's source directory. This step is language-agnostic — the class structure mirrors the monolith's model classes, adapted to remove cross-domain ORM relationships.

**Constraints:**
- You MUST read the corresponding model/entity class from the monolith source and reproduce it in the decomposed service — do NOT rewrite from scratch
- You MUST copy all field names, types, and constraints verbatim from the monolith source
- You MUST copy all `enum` types and their constants verbatim — read the actual source file; do NOT infer or substitute values
- You MUST replace all cross-domain ORM relationships with plain scalar reference fields:
  - Remove `@ManyToOne`, `@OneToMany`, `ForeignKey`, or equivalent ORM annotations that reference a table owned by another domain
  - Replace with a plain scalar field holding only the remote ID (e.g., `customerId: Long`, `customer_id: int`, `CustomerId: Guid`)
  - Add a comment on every such field: `// ref: {owning-service}.{table}.{column} — synced via {EventName} REST notification`
- You MUST add serialisation guards on any remaining bidirectional relationship fields to prevent circular serialisation (e.g., `@JsonIgnore` for Java, `exclude=True` for Python Pydantic, `JSON.stringify` replacer for Node.js)
- You MUST generate exactly one model class per table per service — never duplicate mappings to the same table

**Per-language model class conventions:**

| language | ORM / model style | Cross-domain ref replacement |
|---|---|---|
| `java` | JPA `@Entity` + `@Table` + `@Id @GeneratedValue` | Remove `@ManyToOne`/`@OneToMany`; add `private Long {entityId}` |
| `python` | SQLAlchemy `Base` + `Column` / Django `Model` / Pydantic `BaseModel` | Remove `relationship()`; add `{entity}_id: int` column |
| `nodejs` | TypeORM `@Entity` / Sequelize `Model` / Mongoose `Schema` | Remove `@ManyToOne`/`@OneToMany`; add `{entityId}: number` field |
| `csharp` | EF Core `DbContext` entity / Dapper POCO | Remove navigation properties; add `public int {Entity}Id` |

**4d — Generate Domain Schema DDL**

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

**4e — Generate Domain Schema DDL (dialect-specific)**

Generate the same DDL as 4d but using syntax appropriate for `{db_type}`.

**Constraints:**
- You MUST write each DDL file into the service's resources directory following the `language` convention from Step 4c
- You MUST NOT place generated schema files inside the monolith codebase
- Each DDL file MUST be self-contained and executable against a fresh `{db_type}` database
- You MUST NOT include cross-database foreign key constraints
- You MUST include a schema version tracking table per service

**4f — Define Data Migration Options**

**Constraints:**
- You MUST document the data migration sequence needed to move data from the monolith's shared database into the domain databases, should the team choose to do so:
  1. Stand up one new empty database per domain service
  2. Run each domain DDL script against its target database
  3. Migrate data table by table, starting with the domain that has the fewest cross-domain dependencies
  4. Validate row counts and checksums after each table migration
- You MUST identify any tables that require a data transformation (not just a copy) during migration and describe the transformation
- You MUST flag any table with more than 10 million rows as a `large-table` migration risk requiring an incremental/online migration strategy
- You MUST save the migration options as `{output_path}/phase4f-migration-plan.md`
- You MUST NOT prescribe a specific cutover strategy (dual-write, Strangler Fig, big-bang, parallel run) because that decision is made outside this SOP by the team owning the monolith
- You SHOULD document the trade-offs of common migration approaches (dual-write period, parallel run, big-bang) as informational notes so the team has the context to decide

### 5. Generate Deployable Units

Generate fully executable, independently deployable service modules — one per domain in `domain_model` — plus a shared kernel. Each module must be runnable using the same tech stack as the monolith, with no enhancements or new dependencies beyond what the monolith already uses. The `language` parameter drives all code generation conventions throughout this step.

**Core principle: read the monolith, reproduce faithfully, split by domain boundary.**

---

**5a — Read Monolith Build Descriptor**

Before generating any code, read the monolith's build descriptor to capture the exact dependency set.

**Constraints:**
- You MUST read the monolith's build descriptor (`pom.xml` for Java/Maven, `build.gradle` for Gradle, `package.json` for Node.js, `requirements.txt` / `pyproject.toml` for Python, `*.csproj` for C#)
- You MUST record every dependency, version, and scope exactly as declared
- You MUST use these exact versions in every generated service — do NOT upgrade, downgrade, or add dependencies not present in the monolith
- You MUST add only two categories of new dependencies that are justified by decomposition:
  1. A shared-kernel dependency (`com.bank:shared-kernel` or equivalent) — required because decomposition introduces this new artifact
  2. Test dependencies (`spring-boot-starter-test`, `pytest`, `jest`, etc.) — required to run the unit tests generated in Step 6, only if the monolith had tests

---

**5b — Generate Build Descriptor per Service**

Generate a build descriptor for each service and for shared-kernel.

**Constraints:**
- You MUST generate the build descriptor in the format appropriate for `language` and the build tool detected in the monolith
- You MUST include all dependencies carried forward from the monolith's build descriptor
- You MUST NOT add dependencies absent from the monolith (e.g., do not add OpenAPI, circuit breakers, or migration tools if the monolith did not use them)
- You MUST set the parent/base version to match the monolith exactly (e.g., Spring Boot parent version, Node.js engine version)
- For `shared-kernel`, you MUST generate a library build descriptor (not a runnable application) — e.g., plain JAR for Java with no application plugin, a library `package.json` for Node.js

**Per-language build descriptor conventions:**

| language | Build descriptor | Service type | Shared-kernel type |
|---|---|---|---|
| `java` (Maven) | `pom.xml` with `spring-boot-starter-parent` | Executable JAR via `spring-boot-maven-plugin` | Plain JAR, no boot plugin |
| `java` (Gradle) | `build.gradle` | `bootJar` task | `jar` task only |
| `nodejs` | `package.json` | `main` entry + `start` script | `main` entry, no `start` script |
| `python` | `pyproject.toml` or `requirements.txt` | Runnable app entry point | Library package, no entry point |
| `csharp` | `*.csproj` | `<OutputType>Exe</OutputType>` | `<OutputType>Library</OutputType>` |

---

**5c — Generate Application Entry Point**

**Constraints:**
- You MUST generate an application entry point for each service in the style used by the monolith
- You MUST configure the component/bean scan to include both the service's own package and the shared-kernel package so shared utilities are discovered at startup
- You MUST NOT add startup configuration that did not exist in the monolith (e.g., do not add health check endpoints, metrics, or banner customisation unless the monolith had them)

---

**5d — Generate Data Model Classes**

For each table owned by the service (from Step 4b), generate the corresponding data model class.

**Constraints:**
- You MUST read the corresponding entity/model class from the monolith source and reproduce it in the decomposed service — do NOT rewrite from scratch
- You MUST copy all field names, types, annotations, and constraints verbatim from the monolith source
- You MUST copy all `enum` types and their constants verbatim — read the actual source file; do NOT infer or substitute values
- You MUST replace all cross-domain ORM relationships (e.g., `@ManyToOne`, `@OneToMany`, foreign key references) with plain scalar reference fields (e.g., `private Long customerId`) because the referenced entity lives in a different service's database
- You MUST add a comment on every cross-domain reference field explaining what it references and how it is kept in sync (e.g., `// ref: customer-service — synced via CustomerProfileUpdated REST notification`)
- You MUST add `@JsonIgnore` (or language equivalent) on any bidirectional relationship fields to prevent circular serialisation
- You MUST generate exactly one model class per table per service — never duplicate mappings to the same table

---

**5e — Generate Repository / Data Access Layer**

**Constraints:**
- You MUST read each repository or DAO class from the monolith and reproduce it in the owning service
- You MUST carry forward all query methods exactly as declared in the monolith
- You MUST NOT add query methods that did not exist in the monolith

---

**5f — Generate Service Layer**

**Step 2 classification drives every decision in this sub-step.** Before writing any code, read the classification matrix from `{output_path}/phase2-classification.md` and apply the action for each class:

| Step 2 action | What Step 5f does |
|---|---|
| `move` | Read the class from the monolith and write it verbatim to the target service. No changes to method bodies. |
| `split` | Read the class from the monolith. Write only the methods assigned to this service's domain. Replace every cross-domain call in those methods with a REST call through the client interface generated in Step 5h. |
| `extract-method` | Read the method from the monolith. Identify the domain-specific lines per the Step 2 notes. Extract those lines into a new method in this service. Replace the cross-domain portion with a REST call. |
| `shared-kernel` | Do not generate in any domain service — handled in Step 5j. |
| `facade` | Do not generate — flagged for team decision outside this SOP. |
| `ambiguous` | Do not generate — document in report as requiring manual resolution before code can be produced. |

**Constraints:**
- You MUST read the Step 2 classification matrix before generating any service class
- You MUST NOT copy a method that Step 2 assigned to a different domain into this service
- You MUST replace all direct cross-domain repository or service calls with REST API calls through the cross-domain client interface generated in Step 5h
- You MUST NOT add business logic that did not exist in the monolith

---

**5g — Generate REST Controllers**

**Constraints:**
- You MUST read each controller from the monolith and reproduce the domain-relevant endpoints in the owning service
- You MUST carry forward all request mappings, HTTP methods, path variables, request bodies, and response types exactly as declared in the monolith
- You MUST NOT add new endpoints that did not exist in the monolith
- You MUST generate an `InternalEventController` (or language-equivalent handler) per service that exposes `POST /internal/events` to receive REST notification callbacks from other services — this is a new endpoint required by decomposition, not an enhancement

---

**5h — Generate Cross-Domain REST Clients**

For every cross-domain dependency identified in Step 3, generate a client interface and implementation.

**Constraints:**
- You MUST generate one client interface per downstream service dependency (e.g., `CustomerServiceClient`, `AccountServiceClient`)
- You MUST implement the client using the HTTP client library already present in the monolith's build descriptor — do NOT introduce a new HTTP client library
- You MUST implement only the methods that correspond to actual cross-domain calls found in the monolith — do NOT generate speculative methods
- You MUST read the target service's base URL from configuration (not hardcoded) so it can be overridden per environment
- You MUST generate an `EventPublisher` per service that sends REST notification callbacks (`POST /internal/events`) to downstream services for state change events identified in Step 3
- The `EventPublisher` MUST catch HTTP errors and log a warning without throwing — a failed notification MUST NOT roll back the originating transaction

---

**5i — Carry Forward Cross-Cutting Configuration**

**Constraints:**
- You MUST read every cross-cutting config class from the monolith (security config, CORS config, logging config, etc.) and reproduce it in each service that needs it
- You MUST adapt security config to also permit `/internal/events` without authentication — this is the only permitted adaptation, required by decomposition
- You MUST NOT add new config classes that did not exist in the monolith
- You MUST generate a service config file (format per `language`) containing:
  - Database connection URL, username, password for `{db_type}`
  - Base URLs for all downstream services this service calls
  - Only the config properties relevant to this service's domain (identified from the config inventory in Step 1)
- You MUST generate a local development config override using an in-memory or embedded database so the service starts without a running database server
- You MUST copy shared infrastructure properties (logging levels, etc.) to all services
- You MUST flag ambiguous properties with a comment: `# TODO: verify this property is needed by this service`

---

**5j — Generate Shared Kernel**

**Constraints:**
- You MUST generate the shared-kernel module containing all classes classified as `shared-kernel` in Step 2 (cross-cutting utilities, common DTOs, audit logger, reference generators, etc.)
- You MUST generate a `DomainEvent` DTO with fields: `eventId` (UUID), `type` (String), `occurredAt` (ISO8601), `payload` (Map) — this is a new artifact required by decomposition for REST notification callbacks
- You MUST NOT add shared utilities that did not exist in the monolith

---

**5k — Generate docker-compose.yml**

**Constraints:**
- You MUST generate a `docker-compose.yml` at `{output_path}/docker-compose.yml` that wires all services together with one separate database instance per service
- You MUST NOT share a database container between services
- You MUST NOT add a message broker container — all inter-service communication is REST

---

**Output directory structure:**

```
{output_path}/
├── {service-1}/
│   ├── src/main/          ← service layer, controllers, model, repository, clients, config
│   ├── src/test/          ← populated by Step 6 (if monolith had tests)
│   ├── resources/
│   │   ├── application.{ext}
│   │   └── application-local.{ext}
│   ├── db/
│   │   └── V1__{service-1}_schema.sql
│   ├── Dockerfile
│   ├── {build-descriptor}
│   └── README.md
├── {service-N}/
│   └── ...                ← same structure, one per domain
├── shared-kernel/
│   ├── src/main/          ← DomainEvent DTO, shared utilities carried from monolith
│   ├── src/test/
│   └── {build-descriptor}
└── docker-compose.yml
```

### 6. Unit Test Generation

**Pre-condition check — MUST be done before generating any tests:**

Check the monolith's build descriptor for test framework dependencies — this is the definitive signal that the monolith was built with testing in mind:

| language | Build descriptor | Test framework indicators to look for |
|---|---|---|
| `java` | `pom.xml` / `build.gradle` | `junit`, `mockito`, `spring-boot-starter-test`, `testng`, `assertj` |
| `python` | `pyproject.toml` / `requirements.txt` / `setup.cfg` | `pytest`, `unittest`, `mock`, `hypothesis` |
| `nodejs` | `package.json` (`devDependencies`) | `jest`, `mocha`, `chai`, `sinon`, `vitest`, `jasmine` |
| `csharp` | `*.csproj` | `xunit`, `nunit`, `mstest`, `moq`, `fluentassertions` |

- You MUST read the monolith's build descriptor and check for any of the above test framework dependencies
- If **no test framework dependency is found**, you MUST skip this entire step and note in the report that the monolith had no test framework configured and therefore no tests were generated for the decomposed services
- If **a test framework dependency is found**, proceed with the steps below — use that same framework in the decomposed services

For every class moved or generated into a decomposed service in Step 5, generate a corresponding unit test class covering all public methods. Tests must be runnable in isolation without depending on other services or shared databases. Use the test framework appropriate for `language` (see the per-language quick reference table in Parameters).

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
  1. **Executive Summary** — monolith size, number of modules classified, target services generated (one per domain in `domain_model`)
  2. **Domain Classification Matrix** — full table from Phase 2
  3. **Ambiguous Modules** — list of flagged items with recommended ownership decisions
  4. **Inter-Domain Contracts** — synchronous REST query APIs and REST notification callback contracts (DomainEvent payloads per event type) from Phase 3
  5. **Database Decomposition Summary** — table ownership assignments, cross-domain FK resolutions, and read-model decisions from Phase 4
  6. **Data Migration Options** — migration sequence, large-table risks, and trade-off notes on migration approaches (dual-write, parallel run, big-bang) from Phase 4e — decision on approach left to the team
  7. **Shared Kernel Contents** — list of cross-cutting components extracted
  8. **Recommended Decomposition Order** — suggested sequence for scaffolding services based on coupling analysis (least coupled first), presented as a recommendation not a prescription
  9. **Risk Register** — at minimum cover: data consistency, distributed transactions, network latency, team silos, large-table migration complexity
  10. **Definition of Done checklist** — per service: executable code generated, own DB schema, build descriptor with correct dependencies, config files, unit tests (if monolith had tests), CI/CD pipeline, observability, integration tests passing
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
