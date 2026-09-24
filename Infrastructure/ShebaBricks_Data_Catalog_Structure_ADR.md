# ShebaBricks Data Catalog Structure ADR
---

## **Status**

🔍 **In-Review** | Date: 2026-09-23 | Owner: Data Platform Team

---

## **Decision**

ShebaBricks main Data Catalog Structure:
- **Environments** - catalogs will be duplicated per environment (i.e. development, production)
- **Technical Layer** - in Bronze layer will have 2 unified catalogs - raw and qualified (per environement) with schemas per source system (i.e. Namer, SAP)
- **Business Layer** - the structure of gold & silver layers catalogs will be **decided later based on use cases**
- **Isolated business Catalogs** - A model for a specific use case and a specific audiance (i.e. specific labratory) will have a seperated catalog
- **Infrastructure as code** - objects creation will not be done manualy, but defined as code
- **Permissions** - will be granted for groups only, and based on least priviliges needed principal

---

## Context

**ShebaBricks** is the data platform for Sheba Medical Center, a large healthcare organization with multiple operational source systems (e.g. Namer, SAP). Data from these systems must be ingested, curated, and made available for analytics, reporting, and clinical research — all under strict healthcare data governance (patient privacy, PII protection, auditability).

The platform is built on Databricks with **Unity Catalog** as the centralized governance layer for permissions, lineage, and data discovery. This ADR extends that foundation by defining how catalogs, schemas, and tables are organized across all three medallion layers (Bronze, Silver, Gold) and how permissions are assigned.


---

## Workspace Access Models

There are three distinct access models that apply at different layers. Understanding the difference is critical for correct permission assignment.

| Access Type | What It Grants | How It Is Controlled | Typical Users |
| --- | --- | --- | --- |
| **Consumer Access** | Ability to consume data through dashboards and AI in Genie One | Read-only Unity Catalog privileges on Gold layer (`SELECT` on `ambulatory.gold`), served via dashboards or sharing | business users |
| **SQL Access** | Ability to run SQL queries against data via warehouses, SQL editor, and create AI\BI dashboards | Unity Catalog privileges (`USE CATALOG`, `USE SCHEMA`, `SELECT`) plus access to a SQL warehouse | data analysts |
| **Workspace Access** | Ability to log in to the Databricks workspace UI, browse assets, create notebooks, and use compute | workspace group membership | data engineers |
| **Admin Access** | Full control over workspace configuration: user and group management, compute, warehouses and permissions | Workspace admin role, account admin role | data platform admins |

### Unity Catalog Permission Hierarchy

In Databricks Unity Catalog, SQL access to data follows a hierarchical privilege model. To query a table, a user (or group) must hold all three of the following: `USE CATALOG` on the catalog, `USE SCHEMA` on the schema, and a table-level privilege such as `SELECT` (for read) or `ALL TABLE PRIVILEGES` (for full DML/DDL). The `USE CATALOG` and `USE SCHEMA` privileges act as gatekeepers — without them, table-level grants have no effect. This hierarchy means permissions can be granted at the catalog, schema, or table level, with each level narrowing the scope of access.


---

## Example - dev:

### Development catalogs structures

```
catalog -> schema -> tables

bronze_raw_dev
    namer
        patients
        blood_tests
    sap
        budget
        transactions
bronze_qualified_dev
    namer
        patients
        blood_tests
    sap
        budget
        transactions
```

### Production catalogs structures

```
catalog -> schema -> tables

bronze_raw
    namer
        patients
        blood_tests
    sap
        budget
        transactions

bronze_qualified
    namer
        patients
        blood_tests
    sap
        budget
        transactions
```


---

# Permissions

## Workspace Access
Workspace Access is granted per workspace based on Databricks group membership

### Ingestion Workspace
Only needed for Changes to ingestion procudures (new sources, handle errors)

| Group | Workspace Permission |
| --- | --- |
| `data_ops` | Workspace Access |
| `data_engineers` | None |
| `data_analysts` | None |
| `business_users` | None |
| `data_platform_admins` | Admin Access |

---
### Analytics Workspace

| Group | Workspace Permission |
| --- | --- |
| `data_engineers` | Workspace Access |
| `data_analysts` | SQL Access |
| `business_users` | Consumer Access |
| `data_platform_admins` | Admin Access |

---

### SQL Access (per catalog)

SQL access is granted per catalog via Unity Catalog privileges. To access any table, a group must hold `USE CATALOG` + `USE SCHEMA` + the appropriate table-level privilege.

#### Catalog: `bronze_raw_dev`

| Group | Unity Catalog Permission |
| --- | --- |
| `data_engineers` | `USE CATALOG`, `USE SCHEMA`, `SELECT` |
| `data_analysts` | No access |
| `business_users` | No access |
| `data_platform_admins` | `USE CATALOG`, `USE SCHEMA`, `ALL TABLE PRIVILEGES` |

---

#### Catalog: `bronze_qualified_dev`

| Group | Unity Catalog Permission |
| --- | --- |
| `data_engineers` | `USE CATALOG`, `USE SCHEMA`, `ALL TABLE PRIVILEGES` |
| `data_analysts` | `USE CATALOG`, `USE SCHEMA`, `SELECT` on `table_qualified` (special cases only) |
| `business_users` | No access |
| `data_platform_admins` | `USE CATALOG`, `USE SCHEMA`, `ALL TABLE PRIVILEGES` |



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

✅ **Decision**: Bronze is split into two catalogs — `bronze_raw_dev` and `bronze_qualified_dev` (per environment) — each containing one schema per source system (e.g. `namer`, `sap`). Raw tables live in `bronze_raw_dev.<source>` and their qualified counterparts in `bronze_qualified_dev.<source>`.

**Why**: Separating raw and qualified into different catalogs enforces a clean boundary between the unfiltered landing zone and the quality-checked layer, allowing independent permission policies (e.g. `bronze_raw_dev` is more restricted). Each catalog still preserves the source system structure via per-source schemas, so data quality issues trace directly to the originating system. Pipeline definitions remain straightforward since the raw table and its qualified counterpart share the same schema name across the two catalogs.

**Rejected**:
- Single catalog with both raw and qualified tables co-located in the same schema: Weaker isolation between raw and qualified, harder to apply different permission policies, no clear lifecycle boundary
- Single schema for all sources (e.g. `bronze.all`): Loses source system boundaries, makes it impossible to grant per-source access, complicates pipeline ownership
- One catalog per source system (e.g. `bronze_namer`, `bronze_sap`): Catalog sprawl, Unity Catalog has a catalog limit, harder to manage at scale, requires a lot of handling on CI\CD

---

### **3. Silver and Gold catalog structure**

To be decided later based on actual use cases

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

✅ **Environment separation via catalog duplication**

✅ **Least-privilege group-based permissions**

✅ **Infrastructure as code for all objects and grants**
