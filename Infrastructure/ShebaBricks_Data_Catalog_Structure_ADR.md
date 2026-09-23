# ShebaBricks Data Catalog Structure ADR
---

## **Status**

🔍 **In-Review** | Date: 2026-09-23 | Owner: Data Platform Team

---

## **Decision**

ShebaBricks main Data Catalog Structure:
- **Environments** - catalogs will be duplicated per environment (i.e. development, production)
- **Technical Layer** - in Bronze layer will have one unified catalog (per environement) with schemas per source system (i.e. Namer, SAP)
- **Business Layer** - gold & silver layers will be schemas under a business entity catalog (i.e. Ambulatry)
- **Infrastructure as code** - objects creation will not be done manualy, but defined as code
- **Permissions** - will be granted for groups only, and based on least priviliges needed principal

---

## Context

**ShebaBricks** is the data platform for Sheba Medical Center, a large healthcare organization with multiple operational source systems (e.g. Namer, SAP). Data from these systems must be ingested, curated, and made available for analytics, reporting, and clinical research — all under strict healthcare data governance (patient privacy, PII protection, auditability).

The platform is built on Databricks with **Unity Catalog** as the centralized governance layer for permissions, lineage, and data discovery. This ADR extends that foundation by defining how catalogs, schemas, and tables are organized across all three medallion layers (Bronze, Silver, Gold) and how permissions are assigned.


---

## Access Models

There are three distinct access models that apply at different layers. Understanding the difference is critical for correct permission assignment.

| Access Type | What It Grants | How It Is Controlled | Typical Users |
| --- | --- | --- | --- |
| **Consumer Access** | Ability to consume data through dashboards and AI in Genie One | Read-only Unity Catalog privileges on Gold layer (`SELECT` on `ambulatory.gold`), served via dashboards or sharing | business users |
| **SQL Access** | Ability to run SQL queries against data via warehouses, SQL editor, and create AI\BI dashboards | Unity Catalog privileges (`USE CATALOG`, `USE SCHEMA`, `SELECT`) plus access to a SQL warehouse | data analysts |
| **Workspace Access** | Ability to log in to the Databricks workspace UI, browse assets, create notebooks, and use compute | workspace group membership | data engineers |
| **Admin Access** | Full control over workspace configuration: user and group management, compute, warehouses and permissions | Workspace admin role, account admin role | data platform admins |



---

## Example:

### Development catalogs structures

```
catalog -> schema -> tables

bronze_dev
    namer
        patients
        blood_tests
        patients_qualified
        blood_tests_qualified
    sap
        budget
        transactions
        budget_qualified
        transactions_qualified


ambulatory_dev
    silver
        dim_patients
        fact_blood_tests
        fact_sap_tranactions
    gold
        patient_summary
        blood_test_summary
        blood_test_trends
```

### Production catalogs structures

```
catalog -> schema -> tables

bronze
    namer
        patients
        blood_tests
        patients_qualified
        blood_tests_qualified
    sap
        budget
        transactions
        budget_qualified
        transactions_qualified


ambulatory
    silver
        dim_patients
        fact_blood_tests
        fact_sap_tranactions
    gold
        patient_summary
        blood_test_summary
        blood_test_trends
```


---

## Permissions
### Catalog: `bronze_dev`


**Permissions (group based):**

| Group | Unity Catalog Permission | Workspace Permission |
| --- | --- | --- |
| `data_engineers` | `USE CATALOG`, `USE SCHEMA`, `ALL TABLE PRIVILEGES` | Workspace Access, Compute |
| `data_analysts` | `USE CATALOG`, `USE SCHEMA`, `SELECT` on `table_qualified` (special cases only) | SQL Access |
| `business_users` | No access | No access |
| `data_platform_admins` | `USE CATALOG`, `USE SCHEMA`, `ALL TABLE PRIVILEGES` | Admin Access |

---

### Catalog: `ambulatory` (Business Layer - Silver and Gold)

**Permissions (group based):**

| Group | Unity Catalog Permission | Workspace Permission |
| --- | --- | --- |
| `data_engineers` | `USE CATALOG`, `USE SCHEMA`, `ALL TABLE PRIVILEGES` on `ambulatory.silver` and `ambulatory.gold` | Workspace Access, Compute |
| `data_analysts` | `USE CATALOG`, `USE SCHEMA`, `SELECT` on `ambulatory.silver` and `ambulatory.gold` | SQL Access |
| `business_users` | `USE CATALOG`, `USE SCHEMA`, `SELECT` on `ambulatory.gold` | consumer Access |
| `data_platform_admins` | `USE CATALOG`, `USE SCHEMA`, `ALL TABLE PRIVILEGES` on `ambulatory.silver` and `ambulatory.gold` | Admin Access |

---

## **Key Design Decisions**

### **1. Catalogs Duplicated per Environment**

✅ **Decision**: Each environment (development, production) gets its own set of catalogs with an environment suffix (e.g. `bronze_dev` / `bronze`, `ambulatory_dev` / `ambulatory`)

**Why**: Full isolation between environments — pipeline changes can be validated against dev data without touching production. No risk of accidental writes to production tables from development pipelines. Simplifies environment-specific permission management.

**Rejected**:
- Single catalog with environment-specific schemas (e.g. `bronze.namer_dev`, `bronze.namer_prod`): Weaker isolation, permission boundaries span environments, harder to clean up
- Single catalog with table-level prefixes (e.g. `bronze.namer.patients_dev`): No real isolation, error-prone, complex permission rules

---

### **2. Bronze Organized by Source System**

✅ **Decision**: Bronze catalog contains one schema per source system (e.g. `namer`, `sap`), with raw and qualified tables co-located in the same schema

**Why**: Preserves the source system structure so data quality issues trace directly to the originating system. Co-locating raw and qualified tables in the same schema simplifies pipeline definitions (raw table and its qualified counterpart share a schema). Enables per-source permission isolation.

**Rejected**:
- One schema per source per layer (e.g. `namer_raw`, `namer_qualified`): Doubles the number of schemas, adds unnecessary navigation overhead, no clear benefit since raw and qualified share the same data model
- Single schema for all sources (e.g. `bronze.all`): Loses source system boundaries, makes it impossible to grant per-source access, complicates pipeline ownership
- One catalog per source system (e.g. `bronze_namer`, `bronze_sap`): Catalog sprawl, Unity Catalog has a catalog limit, harder to manage at scale, requires a lot of nandling on CI\CD

---

### **3. Silver and Gold Organized by Business Domain**

✅ **Decision**: Silver and Gold are schemas under domain catalogs (e.g. `ambulatory.silver`, `ambulatory.gold`), not separate catalogs per layer

**Why**: Business consumers think in terms of domains, not source systems or medallion layers. A single domain catalog groups all related data (cleansed and curated) in one place, simplifying discovery and permission management. Adding a new domain = one new catalog with two schemas.

**Rejected**:
- One catalog per layer per domain (e.g. `ambulatory_silver`, `ambulatory_gold`): Catalog sprawl, doubles the number of catalogs, unnecessary since silver and gold share the same domain context
- Single shared catalog for all domains (e.g. `business.silver`, `business.gold` with per-domain schemas): Loses domain boundaries, cross-domain permissions become complex, no clear ownership
- One catalog per source system in silver/gold (e.g. `namer_silver`): Mirrors bronze structure but breaks the domain model — business users need data combined across sources, not split by source

---

### **4. Raw and Qualified as Tables in the Same Schema**

✅ **Decision**: Raw tables (e.g. `patients`) and their qualified counterparts (e.g. `patients_qualified`) live as separate tables within the same source schema in Bronze

**Why**: The qualified table is a 1:1 transform of the raw table with added data quality expectations and column documentation. Keeping them in the same schema maintains the source system grouping and simplifies pipeline references. Table naming convention (`<table>` vs `<table>_qualified`) clearly distinguishes the two stages.

**Rejected**:
- Separate schemas for raw and qualified (e.g. `bronze_dev.namer_raw` and `bronze_dev.namer_qualified`): Doubles schema count, fragments the source system grouping, no benefit since both layers share the same data model
- Views instead of qualified tables: Views cannot enforce data quality expectations or track quality metrics; the qualified layer must be materialized

---

### **5. Group-Based Permissions with Least Privilege**

✅ **Decision**: All permissions are granted to groups, never to individual users. Each group receives only the minimum privileges needed for its role.

**Why**: Group-based permissions scale better than individual grants, reduce administrative overhead, and ensure consistency. Least privilege limits exposure of sensitive healthcare data — business users never touch Bronze, data analysts get SELECT only on qualified tables, and only data engineers have full table privileges.

**Rejected**:
- Individual user grants: Does not scale, error-prone, impossible to audit at organization level
- Broad group grants (e.g. all users get SELECT on all schemas): Violates healthcare data governance requirements, PII exposure risk, no accountability

---

### **6. Four-Tier Access Model (Consumer, SQL, Workspace, Admin)**

✅ **Decision**: Four distinct access types ordered from least to most privilege, each mapped to specific user personas and layers.

**Why**: Clear separation of concerns — consumer access is dashboard-only (Gold layer via Genie), SQL access enables analysts to query data and build dashboards, workspace access enables engineers to use compute and notebooks, admin access is for platform management. Each tier builds on the previous, and permissions tables explicitly define what each group gets at each layer.

**Rejected**:
- Binary access (admin vs user): Too coarse, cannot distinguish between a dashboard consumer and a SQL analyst, leads to over-privileging
- Per-layer access without persona mapping: Permissions become hard to reason about; the persona-based model makes it clear who gets what

---

### **7. Infrastructure as Code for Object Creation**

✅ **Decision**: All catalog, schema, and table creation plus permission grants are defined as code (Declarative Automation Bundles), not created manually.

**Why**: Repeatability across environments (dev catalog structure mirrors prod), full audit trail of changes, no configuration drift, and onboarding new source systems or domains follows a standard code template rather than manual SQL execution.

**Rejected**:
- Manual creation via SQL editor: Error-prone, no audit trail, impossible to reproduce across environments, no version control
- UI-based creation only: Not scriptable, not repeatable, no peer review process

---

## **Conclusion**

This catalog structure provides a clear separation between the technical layer (Bronze, organized by source system) and the business layer (Silver and Gold, organized by domain), with environment isolation and group-based least-privilege permissions. The four-tier access model ensures healthcare data governance requirements are met while enabling self-service analytics for business users.

### **Key Principles**

✅ **Source system isolation in Bronze**

✅ **Domain-driven organization in Silver and Gold**

✅ **Environment separation via catalog duplication**

✅ **Least-privilege group-based permissions**

✅ **Infrastructure as code for all objects and grants**
