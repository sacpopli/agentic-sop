# Industry Standards Quick Reference Guide

This guide helps you select the appropriate `industry_standard` parameter for monolith decomposition.

## Why Industry Standards Matter

The same domain name can have different meanings across industries. Using the correct industry standard ensures:
- Accurate domain boundary classification
- Alignment with industry best practices
- Consistent terminology across teams
- Better integration with industry ecosystems
- Compliance with industry regulations

## Supported Industry Standards

### Banking & Financial Services

#### BIAN (Banking Industry Architecture Network)
**Use when:** Decomposing banking or financial services applications

**Key Domains:**
- Customer & Party Management
- Product & Account Servicing
- Payments & Financial Transactions
- Credit Risk Management
- Loan Management
- Card Management
- Regulatory Compliance

**Characteristics:**
- Service-oriented architecture
- Clear separation between party, product, and transaction domains
- Standardized service operations and data models

**Example:**
```bash
industry_standard=BIAN
domain_model="Customer & Party Management, Product & Account Servicing, Payments & Financial Transactions"
```

#### ISO20022
**Use when:** Decomposing payment processing or financial messaging systems

**Key Domains:**
- Payments Clearing & Settlement
- Securities
- Trade Services
- Cards & Related Retail Payments

#### FIBO (Financial Industry Business Ontology)
**Use when:** Decomposing complex financial instruments or derivatives systems

---

### Telecommunications

#### eTOM (enhanced Telecom Operations Map)
**Use when:** Decomposing telecom operations or service provider applications

**Key Domains:**
- Customer Management
- Product Management
- Service Management
- Resource Management
- Supplier/Partner Management

**Process Layers:**
- Strategy, Infrastructure & Product (SIP)
- Operations (Fulfillment, Assurance, Billing)
- Enterprise Management

**Characteristics:**
- Process-centric decomposition
- Clear separation between customer, service, and resource layers
- End-to-end process flows

**Example:**
```bash
industry_standard=eTOM
domain_model="Customer Management, Product Management, Service Management, Resource Management"
```

#### SID (Shared Information/Data Model)
**Use when:** Focus is on telecom data models and information architecture

---

### Retail

#### ARTS (Association for Retail Technology Standards)
**Use when:** Decomposing retail operations, POS, or e-commerce systems

**Key Domains:**
- Customer
- Inventory
- Order Management
- Payment
- Merchandising
- Store Operations
- Pricing & Promotion

**Characteristics:**
- Retail-specific data models
- Store operations focus
- Omnichannel commerce patterns

**Example:**
```bash
industry_standard=ARTS
domain_model="Customer, Inventory, Order Management, Payment, Merchandising"
```

#### GS1
**Use when:** Focus is on supply chain, product identification, or traceability

#### NRF (National Retail Federation)
**Use when:** Focus is on retail business processes and best practices

---

### Healthcare

#### HL7 (Health Level Seven)
**Use when:** Decomposing healthcare information systems

**Key Domains:**
- Patient Management
- Clinical Documentation
- Orders & Observations
- Billing & Claims
- Scheduling
- Pharmacy
- Laboratory

**Characteristics:**
- Healthcare workflow-centric
- Interoperability focus
- Clinical data standards

**Example:**
```bash
industry_standard=HL7
domain_model="Patient Management, Clinical Documentation, Orders & Observations, Billing & Claims"
```

#### FHIR (Fast Healthcare Interoperability Resources)
**Use when:** Modern healthcare APIs and resource-based architecture

**Key Resources:**
- Patient, Practitioner, Organization
- Encounter, Observation, Condition
- Medication, Procedure, DiagnosticReport

---

### Insurance

#### ACORD (Association for Cooperative Operations Research and Development)
**Use when:** Decomposing insurance systems (P&C, Life, Health)

**Key Domains:**
- Policy Administration
- Claims Management
- Underwriting
- Billing & Payments
- Reinsurance
- Agent/Broker Management

**Characteristics:**
- Insurance-specific data models
- Policy lifecycle management
- Claims processing workflows

**Example:**
```bash
industry_standard=ACORD
domain_model="Policy Administration, Claims Management, Underwriting, Billing & Payments"
```

---

### Supply Chain & Manufacturing

#### SCOR (Supply Chain Operations Reference)
**Use when:** Decomposing supply chain management systems

**Key Domains:**
- Plan
- Source
- Make
- Deliver
- Return
- Enable

#### ISA-95
**Use when:** Decomposing manufacturing execution systems (MES)

**Key Domains:**
- Production Operations Management
- Maintenance Operations Management
- Quality Operations Management
- Inventory Operations Management

---

### Energy & Utilities

#### IEC 61968/61970 (Common Information Model)
**Use when:** Decomposing utility management systems

**Key Domains:**
- Asset Management
- Metering
- Network Operations
- Customer Information
- Work Management

#### MultiSpeak
**Use when:** Focus is on utility data integration and interoperability

---

### Government

#### FEA (Federal Enterprise Architecture)
**Use when:** Decomposing government systems (US federal)

**Key Domains:**
- Performance Reference Model
- Business Reference Model
- Service Component Reference Model
- Data Reference Model
- Technical Reference Model

#### TOGAF
**Use when:** Enterprise architecture-driven decomposition

---

### Generic / Custom

#### Use `generic` or omit `industry_standard` when:
- No industry-specific standard applies
- Custom domain model not aligned with any standard
- Internal application with unique domain structure

**Approach:**
- Uses Domain-Driven Design (DDD) bounded context principles
- Business capability mapping
- Code dependency analysis

**Example:**
```bash
# industry_standard defaults to "generic"
domain_model="User Management, Content Management, Analytics, Notifications"
```

---

## How to Choose

### Step 1: Identify Your Industry
What industry does the monolith serve?
- Banking/Finance → BIAN, ISO20022, FIBO
- Telecom → eTOM, SID
- Retail → ARTS, GS1
- Healthcare → HL7, FHIR
- Insurance → ACORD
- Supply Chain → SCOR, ISA-95
- Energy/Utilities → IEC 61968/61970
- Government → FEA, TOGAF
- Other/Custom → generic

### Step 2: Check Domain Alignment
Do your domain names match the standard's terminology?
- **Yes** → Use that industry standard
- **No** → Consider using `generic` or adapting domain names

### Step 3: Verify Standard Applicability
Does the standard cover your application's scope?
- **Full coverage** → Use the standard
- **Partial coverage** → Use the standard, document gaps
- **No coverage** → Use `generic`

---

## Impact on Classification

### Example: "Customer Management" Domain

**BIAN (Banking):**
- Scope: Party data, KYC, relationships, legal entities
- Excludes: Account management, transactions
- Classification: `CustomerService.openAccount()` → **Product & Account Servicing** (not Customer Management)

**eTOM (Telecom):**
- Scope: Customer orders, service requests, billing, support
- Includes: Service activation, trouble tickets
- Classification: `CustomerService.activateService()` → **Customer Management** (correct)

**ARTS (Retail):**
- Scope: Customer profiles, loyalty, preferences
- Excludes: Order processing, inventory
- Classification: `CustomerService.processOrder()` → **Order Management** (not Customer)

**Generic (No Standard):**
- Scope: Determined by code analysis
- Ambiguity: Higher chance of misclassification
- Classification: Relies on method analysis and dependencies

---

## Best Practices

1. **Be Specific**: Use the most specific standard that applies
   - ✅ `industry_standard=BIAN` for banking
   - ❌ `industry_standard=generic` for banking (loses context)

2. **Align Domain Names**: Use terminology from the standard
   - ✅ `domain_model="Customer & Party Management"` (BIAN term)
   - ❌ `domain_model="Customer Service"` (ambiguous)

3. **Document Deviations**: If your domains don't perfectly match the standard, document why

4. **Consult Documentation**: Reference the standard's official documentation for boundary decisions

5. **Validate with Experts**: Have domain experts review the classification results

---

## Resources

- **BIAN**: https://bian.org/
- **eTOM**: https://www.tmforum.org/oda/
- **ARTS**: https://www.omg.org/retail-depository/arts-odm-73/
- **HL7**: https://www.hl7.org/
- **FHIR**: https://www.hl7.org/fhir/
- **ACORD**: https://www.acord.org/
- **SCOR**: https://www.apics.org/apics-for-business/frameworks/scor
- **IEC CIM**: https://cimug.ucaiug.org/

---

## FAQ

**Q: What if my application spans multiple industries?**
A: Choose the primary industry standard and document cross-industry concepts in the classification notes.

**Q: Can I use multiple industry standards?**
A: No, the parameter accepts a single value. Choose the dominant standard and document deviations.

**Q: What if the standard is too rigid for my needs?**
A: Use `generic` and apply DDD principles. Document your custom domain model for future reference.

**Q: How does this affect the generated code?**
A: It primarily affects domain classification and boundary decisions. The generated code structure remains the same.

**Q: Can I change the industry standard mid-project?**
A: Yes, but you'll need to re-run the classification phase. Domain boundaries may shift significantly.
