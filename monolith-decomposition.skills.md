# Skills: monolith-decomposition.sop.md

Read this file before executing any phase of the SOP. These rules encode known failure modes and hard gates discovered from real runs.

---

## Hard Gates — MUST block execution if condition is not met

| Gate | Condition | Action if failed |
|---|---|---|
| monolith_path accessible | Directory exists and is readable | STOP. Do not proceed to Phase 1. Ask user to fix the path. |
| Step 6 (unit tests) | Test framework dependency present in build descriptor (`pom.xml`, `package.json`, `pyproject.toml`, `*.csproj`) | SKIP Step 6 entirely. Log in report: "Step 6 skipped — no test framework dependency found in monolith build descriptor." |

---

## Conflict Resolution Rules — when two inputs disagree

| Conflict | Winner | Reason |
|---|---|---|
| `config.test_framework` set vs. no test dep in build descriptor | Build descriptor | SOP states build descriptor is the "definitive signal" for Step 6 |
| `config.db_type` set vs. db detected in codebase | Config value wins | Config is an explicit override |
| Class name implies a domain vs. method analysis says otherwise | Method analysis | SOP requires method-level classification, not name-based |

---

## Pre-condition Checklist — verify before each phase

- **Before Phase 1**: confirm `monolith_path` exists and contains source files. STOP if empty or inaccessible.
- **Before Phase 2**: confirm Phase 1 inventory file was saved. Do not classify from memory.
- **Before Phase 5**: confirm Phase 2 classification matrix was saved. Every code generation decision in Phase 5 must trace back to a Phase 2 action (`move`, `split`, `extract-method`, `shared-kernel`, `facade`).
- **Before Phase 6**: read build descriptor. If no test framework dependency → SKIP. Do not generate tests because `config.test_framework` is set.
- **Before Phase 7**: confirm all prior phase output files exist before writing the report.

---

## Known Failure Modes — do not repeat these

| Failure | What went wrong | Correct behaviour |
|---|---|---|
| Tests generated despite no test dep in build descriptor | `config.test_framework` was treated as permission to generate tests | Config parameter does not override the build descriptor gate. SKIP Step 6 and note it in the report. |
| Proceeding when monolith_path missing | Path not found but execution continued | STOP immediately. Ask user to provide correct path before any further action. |
| Classifying a class as `move` based on its name | Did not read all methods | Read every method in the class before assigning any action. |
| Inventing cross-domain contracts | Contracts not derived from actual code | Every contract in Phase 3 MUST trace to an actual cross-domain call found in Phase 1. |
| Shared-kernel bloat | Assigning domain classes to shared-kernel for convenience | Only assign to shared-kernel if the class has zero domain-specific business logic. |

---

## Parameter Priority Order (highest to lowest)

1. SOP-stated definitive signals (e.g., build descriptor for test framework detection)
2. Inline parameters passed at invocation
3. Config file values (`config_file`)
4. SOP defaults

---

## Output File Checklist — every phase must produce its artifact before the next phase starts

| Phase | Required output file |
|---|---|
| 1 | `{output_path}/phase1-inventory.md` |
| 2 | `{output_path}/phase2-classification.md` |
| 3 | `{output_path}/phase3-contracts.md` |
| 4 | `{output_path}/phase4a-shared-schema.md`, `phase4b-table-ownership.md`, `phase4f-migration-plan.md`, DDL files |
| 5 | All service source files, `docker-compose.yml` |
| 6 | Test files (only if gate passed) OR a logged skip note in the report |
| 7 | `{report_path}` |

---

## Output Templates — use these exact structures every run

These templates are derived from the reference run. Fill in values; do not change headings, column names, or section order.

---

### Phase 1 template

```
# Phase 1 — Codebase Inventory

## Parameters
| Key | Value |
|---|---|
| monolith_path | ... |
| output_path | ... |
| language | ... |
| build_tool | ... |
| framework_version | ... |
| runtime_version | ... |
| db_type | ... |
| domain_model | ... |
| target_services | ... |

---

## 1. Build Descriptor — `{build_descriptor_filename}`
| Dependency | Version / Scope |
|---|---|

{State explicitly: "No test framework dependency found → Phase 6 will be skipped."
 OR "Test framework found: {name} → Phase 6 will execute."}

---

## 2. Package & Layer Structure
{ASCII tree with domain annotation per file}

---

## 3. API Entry Points
| HTTP Method | Path | Controller | Domain |
|---|---|---|---|

{Note any batch jobs / message listeners / scheduled tasks, or state none found}

---

## 4. Entity Models & Enums
{Per entity: field table (Field | Type | Notes) + Enums verbatim from source}

---

## 5. Database Schema

### Tables by domain
| Table | Domain | Notes |
|---|---|---|

### Cross-domain foreign keys
| FK | From | To | Strategy |
|---|---|---|---|

---

## 6. Configuration Properties
| Property | Value | Domain |
|---|---|---|

---

## 7. Dependency Map
{ASCII diagram showing cross-domain calls with [cross-domain] labels}

---

## 8. Existing Tests
{State: "No src/test/ directory found. No test framework dependency in build descriptor. Phase 6 will be skipped."
 OR: "Test framework {name} found. Phase 6 will execute."}
```

---

### Phase 2 template

```
# Phase 2 — Domain Classification

## Domain Model
- **{service-name}** → {Domain}

---

## Classification Matrix
| Module / Class | Methods | Domains Touched | Action | Target Service(s) | Notes |
|---|---|---|---|---|---|

---

## Split Detail

### `{ClassName}` — split / extract-method
| Method | Lines of concern | Domain | Target Service | Action |
|---|---|---|---|---|

{Repeat for each split/extract-method class}

---

## Ambiguous Items
{List OR "None."}

---

## Summary
| Action | Count | Classes |
|---|---|---|
| `move` | ... | ... |
| `split` | ... | ... |
| `extract-method` | ... | ... |
| `facade` | ... | ... |
| `shared-kernel` | ... | ... |
| omit | ... | ... |
```

**Classification decisions that must be consistent across runs:**
- Thin controllers that only delegate to a single service → `facade` (not `move`)
- Dev/demo seed data loaders (e.g., `CommandLineRunner` that creates sample records) → `omit` (not `shared-kernel` or `facade`)
- A service method that mixes domain logic with cross-domain persistence (e.g., saves a record belonging to another domain's table) → `extract-method`; the cross-domain persistence moves to the owning service via a REST notification event
- A lifecycle operation that cascades state changes across domain boundaries (e.g., closing an entity also closes related entities in another domain) → `extract-method`; the cascade is replaced with a REST notification event to the owning service, not synchronous REST calls

---

### Phase 3 template

```
# Phase 3 — Service Boundary & Contract Definition

## 1. Aggregate Roots & Owned Entities
| Service | Aggregate Root | Owned Entities / Tables |
|---|---|---|

## 2. Data Ownership
| Table | Owner Service |
|---|---|

## 3. Cross-Domain Dependencies & Resolution Strategy
| Caller | Callee | Monolith call | Resolution |
|---|---|---|---|

## 4. Synchronous REST APIs (Query Contracts)
{Per service: endpoint, request body, response 200 JSON shape, response 4xx}

## 5. REST Notification Contracts (DomainEvent payloads)

### Shared DomainEvent DTO (shared-kernel)
{Language-appropriate code stub}

### Event Catalogue
| Event | Publisher | Receiver | Trigger | Payload fields |
|---|---|---|---|---|

### Notification routing table
{POST /internal/events → {service}: type=X → action description}

### Circular call chain check
{Explicit statement per potential chain}

## 6. EventPublisher Stub (per service that publishes events)
{Language-appropriate code stub}

## 7. InternalEventController Stub (per service that receives events)
{Language-appropriate code stub}

## 8. Cross-Domain REST Clients
{Language-appropriate code stub per client}
```

**Event catalogue rules that must be consistent across runs:**
- Derive every event from actual cross-domain calls found in Phase 1 — do not invent events
- Every cross-domain state-change cascade found in the monolith must produce a corresponding REST notification event (e.g., entity created → downstream entity created; entity suspended → downstream entities frozen; entity closed → downstream entities closed)
- Every cross-domain persistence call found in a service method (writing to a table owned by another domain) must produce a REST notification event so the owning service can perform the write
- The event catalogue in Phase 3 must account for ALL cross-domain calls identified in the Phase 1 dependency map — missing even one produces an incomplete contract set
