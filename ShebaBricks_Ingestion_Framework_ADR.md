# Architecture Decision Record: ShebaBricks Multi-Domain Ingestion Framework

---

## **Status**

✅ **Approved** | Date: 2026-09-03 | Owner: Data Platform Team

---

## **Decision**

We will build a **scalable, governed data ingestion framework** using:
- **Lakeflow Connect** for SQL Server ingestion (Query-Based Capture)
- **Spark Declarative Pipelines** for bronze qualified transformations
- **DAB per domain** for independent deployment
- **Configuration-as-data**: Generate DAB YAML from 4 Unity Catalog config tables (`bundles`, `pipelines`, `tables`, `jobs`)
- **Two-layer architecture**: bronze_raw → bronze_qualified
- **Unity Catalog** for governance (lineage, quality metrics, audit logs)

---

## **Context**

The **ShebaBricks Ingestion Framework** is a multi-domain data ingestion framework for Databricks. We need a **configuration-driven approach** to generate DABs (Declarative Automation Bundles) from Unity Catalog tables instead of manually writing YAML files. This enables UI-based DAB management, configuration reuse across domains, and centralized governance.

---

## **Scope**

This configuration model supports:
- **Lakeflow Connect** managed ingestion from SQL Server (Query-Based Capture)
- **SDP (Spark Declarative Pipelines)** for bronze qualified transformations
- **Two-layer architecture**: bronze_raw → bronze_qualified
- **DAB per domain** with independent deployment

---

## **Requirements**

### **Functional**
- Incremental ingestion from SQL Server (Query-Based Capture)
- Bronze raw: Land data with minimal transformation
- Bronze qualified: Column comments + data quality expectations
- Per-domain isolation: Independent deployment, ownership, scheduling
- Ad-hoc full refresh capability via UI
- Multiple source systems within a domain

### **Non-Functional**
- Modularity: One SQL file per table
- No embedded code in DAB: All SQL in external files
- Cost optimization: Use serverless where possible
- Auditability: Clear lineage and data quality metrics
- Scalability: Easy to add new domains and tables

---

## **Design Principles**

1. **KISS (Keep It Simple)**: Minimal tables, flat structure
2. **Configuration, not code**: DAB YAML generated from tables; all SQL logic in external files
3. **Self-service**: Domain teams configure pipelines via SQL inserts
4. **Maintainable**: Easy to query, understand, and extend
5. **Auditable**: Track who changed what and when

---

## **Data Model: 4 Core Tables**

### **Schema**: `main.dab_config`

---

## **Architecture Overview**

### **Layered Architecture**

```
SQL Server Sources (N systems)
        ↓
Lakeflow Connect (Classic Compute)
  • Query-Based Capture (QBC)
  • Incremental sync with watermark
  • Per-table configuration
        ↓
Bronze Raw (Unity Catalog)
  • main.bronze_raw.<table>_raw
  • Delta tables, minimal transformation
  • LIMITATION: No expectations or comments (Lakeflow Connect restriction)
        ↓
Spark Declarative Pipelines (Serverless)
  • STREAM() source
  • Data quality expectations
  • Column documentation
  • Triggered mode
        ↓
Bronze Qualified (Unity Catalog)
  • main.bronze_qualified.<table>_qualified
  • Validated, documented data
  • Quality metrics tracked
```

### **Why Two Bronze Layers?**

**Lakeflow Connect Limitation**: Managed ingestion does NOT support:
- Data quality expectations on target tables
- Column comments/documentation

**Solution**: Two-layer bronze architecture

**Bronze Raw** (`bronze_raw`):
- ✅ Catch all source system issues
- ✅ Technical layer: Move data, no business logic
- ✅ Data model mirrors source schema (same columns, types, structure)
- ✅ Technical expectations only (NOT NULL, data types, format validation)

**Bronze Qualified** (`bronze_qualified`):
- ✅ Add column documentation
- ✅ Add data quality expectations
- ✅ Track quality metrics
- ✅ Same data model as raw (no reshaping, joins, or aggregations)

### **Why Not Add Expectations in Silver Layer?**

**Decision**: Data quality expectations belong in bronze_qualified, NOT silver

**Reason**: **Isolate source data quality from transformation logic**

**Bronze Qualified**:
- ✅ Data model mirrors source schema (1:1 table structure)
- ✅ No joins, aggregations, or transformations
- ✅ Quality failures = **source system issues**
- ✅ Clear accountability: Data quality problems trace back to source

**Silver Layer**:
- ❌ Different data model (reshaped, joined, aggregated)
- ❌ Business logic and transformations applied
- ❌ Quality failures could be from **transformations** OR source
- ❌ Mixed concerns: Hard to distinguish source issues from transformation bugs

**Debugging Benefit**:
- If bronze_qualified ✅ passes but silver ❌ fails → Transformation logic issue
- If bronze_qualified ❌ fails → Source system data quality issue
- Clear separation = faster root cause analysis

---

## **Key Design Decisions**

### **1. DAB per Domain**
✅ **Decision**: Separate DAB per business domain (Sales, Inventory, Finance, etc.)

**Why**: Independent deployment, clear ownership, reduced blast radius, domain-level audit trails

**Rejected**: Single monolithic DAB (merge conflicts, unclear ownership); DAB per table (too granular)

---

### **2. Lakeflow Connect for Ingestion**
✅ **Decision**: Use Lakeflow Connect with Query-Based Capture (QBC) from SQL Server

**Why**: Managed service with built-in monitoring, automatic watermark management, no code required, schema evolution support

**Compute**: Classic compute (autoscaling) - regulatory restriction prevents serverless

**Rejected**: 
- SQL Server CDC (Change Data Capture): Organization previously rejected CDC due to internal reasons; decision deferred to avoid delays and enable fast delivery. Will be revisited with DBA team in later phase
- Manual JDBC queries (maintenance burden)
- Auto Loader (file-based, not DB QBC)

---

### **3. Spark Declarative Pipelines (SDP) for Bronze Qualified**
✅ **Decision**: SDP streaming tables for bronze qualified layer

**Why**: Built-in data quality expectations, column-level lineage, incremental STREAM() processing, observability dashboard, column documentation support

**Scope**: Technical layer only
- Data movement from raw → qualified
- Technical expectations: NOT NULL, type validation, format checks, referential integrity
- NO business logic: No joins, aggregations, calculations, or business rules
- Data model matches source schema (1:1 table structure)

**Compute**: Serverless (faster startup, auto-scaling, cost-efficient for triggered workloads)

**Mode**: Triggered (not continuous) - aligns with batch schedules, cost optimization

**Rejected**: DBSQL Materialized Views (no expectations); Notebooks (no quality tracking); Continuous mode (cost)

---

### **4. Triggered Orchestration with Explicit Dependencies**
✅ **Decision**: Job orchestrates ingestion → transformation with explicit depends_on

**Why**: Predictable execution, cost control (compute only when needed), clear workflow DAG

**Pattern**: 2-task job (ingest → transform)

**Rejected**: Continuous bronze qualified (cost); Separate scheduled jobs (ordering complexity)

---

### **5. Unity Catalog for Data Governance**
✅ **Decision**: All data in Unity Catalog with three-part namespace

**Convention**:
- Bronze raw: `main.bronze_raw.<table>_raw`
- Bronze qualified: `main.bronze_qualified.<table>_qualified`

**Why**: Unified governance (permissions, lineage, audit logs), workspace independence, data quality tracking, discoverability

---

## **Entity Relationships**

```
bundles (1) ──< (*) pipelines
          └──< (*) jobs

pipelines (1) ──< (*) tables

jobs.ingestion_pipeline_id ──> pipelines
jobs.transformation_pipeline_id ──> pipelines
```

**Simplified Foreign Key Structure**:
- `pipelines.bundle_id` → `bundles.bundle_id`
- `tables.pipeline_id` → `pipelines.pipeline_id`
- `jobs.bundle_id` → `bundles.bundle_id`
- `jobs.ingestion_pipeline_id` → `pipelines.pipeline_id`
- `jobs.transformation_pipeline_id` → `pipelines.pipeline_id`

---

## **Framework Structure**

### **DAB Organization**

```
/Users/<username>/dabs/
├── sales_domain/
│   ├── databricks.yml
│   ├── pipelines/
│   │   ├── bronze_qualified_customers.sql
│   │   ├── bronze_qualified_orders.sql
│   │   └── bronze_qualified_invoices.sql
│   └── validate_bundle.py
│
├── inventory_domain/
│   ├── databricks.yml
│   ├── pipelines/
│   │   ├── bronze_qualified_products.sql
│   │   └── bronze_qualified_warehouse_stock.sql
│   └── validate_bundle.py
│
└── finance_domain/
    ├── databricks.yml
    ├── pipelines/
    │   └── bronze_qualified_accounts.sql
    └── validate_bundle.py
```

### **Key Files**
- **`databricks.yml`**: Generated from config tables
- **`pipelines/*.sql`**: Transformation logic, column comments, expectations
- **`validate_bundle.py`**: Validates bundle config before deployment (checks SQL file paths exist, validates cron expressions, verifies pipeline references). Runs via `databricks bundle validate`

---

## **Standard Pattern**

Each domain bundle contains multiple pipelines and jobs:

```
1 Bundle (e.g., "sales_domain")
  ├─ N Ingestion Pipelines (one per SQL Server connection)
  ├─ M Transformation Pipelines (SDP, serverless)
  └─ K Jobs (different schedules: hourly, daily, weekly)
```

**Configuration per domain**:
- 1 row in `bundles`
- N+M rows in `pipelines`
- X rows in `tables`
- K rows in `jobs`

---

## **What's NOT in Config Tables**

### **In SQL files** (referenced by `tables.sql_file_path`):
- Column comments
- Data quality expectations (`CONSTRAINT ... EXPECT`)
- Transformation logic (SELECT statements)

### **In Unity Catalog** (pre-created manually):
- SQL Server connection definitions
- Source system credentials

---

## **DAB Generation Flow**

1. Query config tables for `bundle_id`
2. Generate `databricks.yml` from config + SQL file references
3. Write YAML to `/Users/<username>/dabs/<domain>/databricks.yml`
4. Deploy via `databricks bundle deploy --target dev`

---

## **Benefits**

✅ Simple: 4 tables, flat structure  
✅ Self-service: SQL inserts to configure  
✅ Auditable: Timestamps on all changes  
✅ Scalable: Add domains by adding rows  
✅ Flexible: Multiple pipelines/jobs per bundle  

---

## **Constraints**

- SQL files must exist at specified paths
- UC connections pre-created manually
- One bundle = one domain
- Column metadata in SQL files, not config tables
- Ingestion requires classic compute (regulatory restriction)
- Each job = 2-task pattern (ingestion → transformation)

---

## **1. `bundles`**

**What it stores**: Bundle identity, domain assignment, UC catalog, compute configuration per target (dev/prod), ownership, deployment status

**Key**: One row = one domain's DAB bundle

---

## **2. `pipelines`**

**What it stores**: Pipeline identity and type, parent bundle reference, target UC schema, execution mode (continuous/triggered), source connection (ingestion only), deployment status

**Key**: Ingestion uses classic compute (regulatory restriction), transformation uses serverless

---

## **3. `tables`**

**What it stores**:
- **Ingestion tables**: Source system details (schema, table), primary keys, incremental column (NULL for full refresh), destination location
- **Transformation tables**: Path to SQL file with transformation logic, destination table name
- **All tables**: Parent pipeline reference, deployment status

**Key**: Column metadata (comments, expectations) lives in SQL files, not here

---

## **4. `jobs`**

**What it stores**: Job identity, parent bundle reference, schedule configuration (cron expression), which pipelines to run (ingestion + transformation), deployment status

**Columns**: `job_id`, `bundle_id`, `job_name`, `ingestion_pipeline_id`, `transformation_pipeline_id`, `schedule_cron`, `is_active`, `created_at`, `updated_at`

**Key**: Fixed 2-task pattern (ingestion → transformation), multiple jobs per bundle with different schedules

---

## **Benefits by Stakeholder**

### **For Data Engineers**
✅ Clear patterns to follow (templates)

✅ Minimal boilerplate (DAB abstracts infrastructure)

✅ Pure SQL (no complex Python/Scala code)

✅ Built-in observability (SDP dashboards)

✅ Easy debugging (clear lineage, quality metrics)

---

### **For Domain Teams**
✅ Independent deployment (no coordination needed)

✅ Clear ownership (one DAB per domain)

✅ Self-service (add tables without platform team)

✅ Flexible scheduling (per-domain cadence)

---

### **For Platform Team**
✅ Standardized patterns (easier to support)

✅ Reduced operational burden (managed services)

✅ Governance by default (Unity Catalog)

✅ Scalable architecture (add domains easily)

✅ Cost visibility (per-domain tagging)

---

### **For Organization**
✅ Regulatory compliance (audit trails, lineage)

✅ Data quality tracking (expectations, metrics)

✅ Fast time-to-value (templates accelerate development)

✅ Reduced maintenance cost (managed services)

---

## **Conclusion**

The **ShebaBricks Ingestion Framework** provides a scalable, governed, and maintainable approach to data ingestion using modern Databricks capabilities. The DAB-per-domain pattern balances independence with standardization, while Lakeflow Connect + SDP provides robust, observable data pipelines with minimal code.

### **Key Principles**

✅ **Domain ownership and independence**

✅ **Configuration over code**

✅ **Managed services over custom builds**

✅ **Governance by default**

✅ **Scalability through standardization**