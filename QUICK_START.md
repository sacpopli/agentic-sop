# Quick Start Guide: Monolith Decomposition with Industry Standards

## 5-Minute Setup

### Step 1: Identify Your Industry

| Your Application | Use This Standard |
|---|---|
| Banking, Financial Services | `BIAN` |
| Telecommunications | `eTOM` |
| Retail, E-commerce | `ARTS` |
| Healthcare | `HL7` |
| Insurance | `ACORD` |
| Supply Chain | `SCOR` |
| Energy, Utilities | `IEC 61968/61970` |
| Government | `FEA` |
| Other/Custom | `generic` (or omit) |

### Step 2: Define Your Domains

Use terminology from your industry standard:

**BIAN (Banking):**
```
Customer & Party Management
Product & Account Servicing
Payments & Financial Transactions
Credit Risk Management
```

**eTOM (Telecom):**
```
Customer Management
Product Management
Service Management
Resource Management
```

**ARTS (Retail):**
```
Customer
Inventory
Order Management
Payment
Merchandising
```

**HL7 (Healthcare):**
```
Patient Management
Clinical Documentation
Orders & Observations
Billing & Claims
```

**Generic (Custom):**
```
User Management
Content Management
Analytics
Notifications
```

### Step 3: Run the SOP

```bash
run monolith-decomposition.sop.md with \
  monolith_path=./your-app \
  industry_standard=YOUR_STANDARD \
  language=YOUR_LANGUAGE \
  domain_model="Domain1, Domain2, Domain3"
```

---

## Real Examples

### Banking Application (Spring Boot)

```bash
run monolith-decomposition.sop.md with \
  monolith_path=./banking-monolith \
  industry_standard=BIAN \
  language=java \
  domain_model="Customer & Party Management, Product & Account Servicing, Payments & Financial Transactions"
```

**Output:** 3 independent microservices with REST APIs, separate databases, Docker Compose

---

### Telecom Application (Python)

```bash
run monolith-decomposition.sop.md with \
  monolith_path=./telecom-app \
  industry_standard=eTOM \
  language=python \
  db_type=postgresql \
  domain_model="Customer Management, Product Management, Service Management"
```

**Output:** 3 independent microservices with FastAPI, separate PostgreSQL databases

---

### Retail Application (Node.js)

```bash
run monolith-decomposition.sop.md with \
  monolith_path=./retail-app \
  industry_standard=ARTS \
  language=nodejs \
  db_type=mongodb \
  domain_model="Customer, Inventory, Order Management, Payment"
```

**Output:** 4 independent microservices with Express.js, MongoDB databases

---

### Legacy Banking (Java EE)

```bash
run monolith-decomposition.sop.md with \
  monolith_path=./banking-javaee \
  industry_standard=BIAN \
  language=java \
  domain_model="Customer & Party Management, Payments & Financial Transactions" \
  decomposition_mode=module
```

**Output:** Domain-aligned modules within monolith, migration path documented

---

### Custom Application (No Standard)

```bash
run monolith-decomposition.sop.md with \
  monolith_path=./my-app \
  language=java \
  domain_model="User Management, Content Management, Analytics"
```

**Output:** Services based on DDD bounded contexts

---

## What You Get

### Phase 1: Inventory
- Complete codebase analysis
- Tech stack detection
- Dependency mapping

### Phase 2: Classification
- Every class assigned to a domain
- Industry-standard context applied
- Ambiguities documented

### Phase 3: Contracts
- Inter-domain APIs defined
- Event notifications specified
- Integration patterns documented

### Phase 4: Database
- Schema ownership assigned
- Cross-domain FKs resolved
- Migration plan created

### Phase 5: Code Generation
- **Full Mode:** Independent services with REST APIs
- **Module Mode:** Domain packages with clear boundaries

### Phase 6: Tests
- Unit tests per domain
- Mock cross-domain dependencies
- 80%+ coverage target

### Phase 7: Report
- Executive summary
- Classification matrix
- Risk register
- Migration roadmap

---

## Output Structure

### Full Decomposition Mode

```
decomposed/
├── customer-service/
│   ├── src/main/java/...
│   ├── src/test/java/...
│   ├── pom.xml
│   ├── Dockerfile
│   └── README.md
├── account-service/
│   └── ...
├── payment-service/
│   └── ...
├── shared-kernel/
│   └── ...
└── docker-compose.yml
```

### Module Decomposition Mode

```
decomposed/
├── domain-modules/
│   ├── customer/
│   │   ├── model/
│   │   ├── repository/
│   │   ├── service/
│   │   └── README.md
│   ├── account/
│   │   └── ...
│   └── shared/
│       └── ...
├── db/schemas/
│   ├── customer_schema.sql
│   └── account_schema.sql
└── DECOMPOSITION_GUIDE.md
```

---

## Common Scenarios

### Scenario 1: Modern Stack, Known Industry

**You have:** Spring Boot banking app
**You want:** Independent microservices aligned with BIAN

```bash
run monolith-decomposition.sop.md with \
  monolith_path=./banking-app \
  industry_standard=BIAN \
  language=java \
  domain_model="Customer & Party Management, Product & Account Servicing, Payments"
```

**Result:** ✅ Full decomposition with BIAN-aligned boundaries

---

### Scenario 2: Legacy Stack, Known Industry

**You have:** Java EE insurance app
**You want:** Prepare for future microservices

```bash
run monolith-decomposition.sop.md with \
  monolith_path=./insurance-javaee \
  industry_standard=ACORD \
  language=java \
  domain_model="Policy Administration, Claims Management, Underwriting" \
  decomposition_mode=module
```

**Result:** ✅ Module decomposition with domain aligned boundaries + migration path

---

### Scenario 3: Modern Stack, Custom Domains

**You have:** Node.js SaaS app
**You want:** Microservices with custom domains

```bash
run monolith-decomposition.sop.md with \
  monolith_path=./saas-app \
  language=nodejs \
  domain_model="User Management, Workspace Management, Billing, Analytics"
```

**Result:** ✅ Full decomposition with DDD-based boundaries

---

### Scenario 4: Legacy Stack, Custom Domains

**You have:** Struts e-commerce app
**You want:** Logical separation for future migration

```bash
run monolith-decomposition.sop.md with \
  monolith_path=./ecommerce-struts \
  language=java \
  domain_model="Catalog, Cart, Checkout, Fulfillment" \
  decomposition_mode=module
```

**Result:** ✅ Module decomposition with clear boundaries + modernization roadmap

---

## Tips for Success

### 1. Use Industry Standards When Possible
✅ **Do:** `industry_standard=BIAN` for banking
❌ **Don't:** Omit industry_standard for banking (loses context)

### 2. Align Domain Names with Standards
✅ **Do:** `"Customer & Party Management"` (BIAN term)
❌ **Don't:** `"Customer Service"` (ambiguous)

### 3. Let Auto-Detection Work
✅ **Do:** Omit `decomposition_mode` (auto-detects from tech stack)
❌ **Don't:** Force `decomposition_mode=full` on Java EE (won't work)
**Note:** When using auto-detect mode, the outcomes may not be deterministic all the time. Use auto-detect carefully.

### 4. Start with Inventory
Run Phase 1 first to understand your codebase:
```bash
run monolith-decomposition.sop.md with \
  monolith_path=./my-app \
  phase=1
```

### 5. Review Classification
Check Phase 2 output before proceeding:
```bash
run monolith-decomposition.sop.md with \
  monolith_path=./my-app \
  industry_standard=BIAN \
  phase=2
```

---

## Troubleshooting

### "Domain boundary unclear"
→ Check if you're using the right industry standard
→ Consult `INDUSTRY_STANDARDS_GUIDE.md`

### "Tech stack not supported"
→ SOP will auto-select module mode
→ You'll still get domain-aligned modules

### "Classification seems wrong"
→ Verify domain names match industry standard terminology
→ Review Phase 2 classification report

### "Can't deploy independently"
→ Check if tech stack supports it (Spring Boot ✅, Java EE ❌)
→ Module mode provides value even without deployment

---

## Next Steps

1. ✅ Run the SOP with your monolith
2. ✅ Review the classification report
3. ✅ Validate domain boundaries with your team
4. ✅ Proceed with code generation
5. ✅ Follow the migration roadmap

---

## Need Help?

- **Industry Standards:** See `INDUSTRY_STANDARDS_GUIDE.md`
- **Detailed Changes:** See `REFACTORING_SUMMARY.md`
- **All Changes:** See `CHANGES_SUMMARY.md`
- **Full SOP:** See `monolith-decomposition.sop.md`

---

## Quick Reference Card

```bash
# Minimum required parameters
monolith_path=./your-app
domain_model="Domain1, Domain2, Domain3"

# Recommended parameters
industry_standard=YOUR_STANDARD  # BIAN, eTOM, ARTS, HL7, ACORD, generic
language=YOUR_LANGUAGE           # java, python, nodejs, csharp

# Optional parameters
decomposition_mode=auto          # auto, full, module
output_path=./decomposed
report_path=./report.md
db_type=postgresql
test_framework=junit5
phase=all                        # 1, 2, 3, 4, 5, 6, 7, all
```

---

**Ready to decompose? Run the SOP now! 🚀**
