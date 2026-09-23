---
name: ssis-to-databricks
description: Guidance for migrating SQL Server SSIS packages to Databricks Lakeflow Spark Declarative Pipelines (SDP). Read this skill when the user asks to convert, migrate, translate, or rewrite SSIS packages, data flows, or ETL logic from SSIS to Databricks. Covers assessment of SSIS package complexity, SSIS component-to-SDP mapping, merging STG and DWH packages into single SDP SQL files, bronze source patterns, Silver layer target standards (Gold only when explicitly requested), data quality expectations, and metadata requirements per the ShebaBricks ADR.
---

# SSIS to Databricks Migration

This skill guides the migration of SQL Server Integration Services (SSIS) packages into Databricks Lakeflow Spark Declarative Pipelines (SDP). All destination code follows the **ShebaBricks Silver & Gold Engineering Standard ADR**.

## Key Migration Rules

1. **Merge STG and DWH into one SDP file**: In SSIS, staging (STG) and data warehouse (DWH) logic are typically split across separate packages. In Databricks, merge both into a **single SDP SQL file** — the Silver layer replaces both STG and DWH stages. Do not create separate staging and DWH pipelines.
2. **Silver only by default**: Convert SSIS packages to the **Silver layer** only. Convert to Gold **only if the user explicitly requests it**.
3. **Ignore logging and event handlers**: SSIS logging, error event handlers, and on-success/on-failure handlers are **not migrated** — SDP handles observability, logging, alerting, and data quality tracking natively. Do not attempt to replicate SSIS logging tables, audit destinations, or event handler logic.
4. **Stored procedure / view SQL files**: Many SSIS packages execute SQL via stored procedures or views. The user should provide this SQL code as `.sql` files in a **sub-folder** alongside the SSIS package description. If any referenced SQL file is missing, **stop and ask the user to provide it** before continuing the migration.
5. **Assessment first**: Before any migration, perform an **assessment phase** that reviews the packages and produces a complexity report (see Assessment Phase section below).
6. **Skip pure orchestrator packages**: If a package only calls (executes) other packages with no data transformation logic of its own, **do not convert it** — SDP handles inter-table dependencies natively via `STREAM()` / `LIVE()` references. If such a package contains additional logic (e.g., pre/post SQL, data quality checks, variable-driven control flow), extract that logic into **separate SQL files** — one per table it creates or updates — and convert those individually.

## Source: Bronze Layer

SSIS packages read from operational source systems (SQL Server, Oracle, files, APIs). In Databricks, these sources are already ingested as **bronze qualified tables** — raw, source-faithful tables in Unity Catalog with complete historical retention.

- Source tables are referenced as `catalog.bronze.<source_table>` (or the workspace's actual bronze catalog/schema).
- Bronze tables preserve the source schema; no cleansing or deduplication has occurred.
- Treat every SSIS source component as a `SELECT` from the corresponding bronze table.
- If a bronze table for an SSIS source does not exist, flag it — the user must create the bronze ingestion first (out of scope for this skill).
- **Stored procedure / view SQL**: SSIS packages often call stored procedures or views. The user should provide these as `.sql` files in a sub-folder next to the package description. If a referenced SQL file is missing, **stop and ask the user to provide it** before continuing.

## Destination: Silver Layer (ADR Standards)

By default, all migrated code targets the **Silver layer only**. Gold layer conversion is performed **only when the user explicitly requests it**. All Silver code **must** comply with the ShebaBricks ADR. Key mandatory rules:

| Standard | Requirement |
| --- | --- |
| **Framework** | Lakeflow SDP (not notebooks, not custom Spark jobs) |
| **Language** | SQL-first (Python only when justified by complex custom logic) |
| **Compute** | Serverless |
| **Dataset Type** | Materialized Views by default; Streaming Tables when STREAM() semantics are needed (CDC, event-time, watermarks) |
| **Schema** | Explicit column lists — `SELECT *` is prohibited |
| **Data Quality** | Explicit SDP Expectations on every Silver table |
| **Metadata** | Table and column comments are mandatory (business meaning, not field-name repetition) |
| **UC Tags** | Recommended: `business_domain`, `source_system`, `data_owner`, `technical_owner`, `sensitivity`, `pii_classification`, `refresh_frequency` |
| **Silver purpose** | Cleaned, conformed enterprise model — information ("what happened"), not insights |
| **Silver history** | Complete historical retention — never subset or delete |


### Silver Layer Conventions

Silver tables clean, conform, and enrich bronze data. Each Silver table must:

1. **Explicit schema** — list every output column with type casts.
2. **Deduplication** — remove duplicates from bronze where appropriate (use `ROW_NUMBER()` or `rank()` patterns).
3. **Data quality Expectations** — at minimum: not-null on key columns, type validation, domain checks. Use severity levels:
   - `INFO` / `WARN` — track but don't block.
   - `REJECT` — quarantine invalid records.
   - `CRITICAL` — fail the pipeline.
4. **Column comments** — every column gets a meaningful business comment.
5. **Table comment** — describes the business purpose of the table.

### Gold Layer Conventions (Only When Explicitly Requested)

> **Gold conversion is optional.** Only convert to Gold when the user explicitly asks for it. The default migration target is Silver.

Gold tables are thin, consumer-ready datasets built on Silver (or Silver Insights). Each Gold table must:

1. **Select only needed columns** — never `SELECT *`.
2. **Aggregate to consumer granularity** — pre-summarize for BI/ML/app needs.
3. **Define and document PK/FK** — primary key and foreign key relationships in comments.
4. **May subset history** — filter by date range or business condition for performance.
5. **Disposable** — Gold is fully regenerable from Silver.

## SSIS-to-SDP Component Mapping

### Control Flow

| SSIS Control Flow Element | Databricks Equivalent |
| --- | --- |
| Sequence Container | Separate SDP streaming table / materialized view in the same pipeline |
| Precedence Constraint (success) | Natural SDP dependency via `STREAM()` or `LIVE()` references |
| Precedence Constraint (failure) | SDP Expectations with `CRITICAL` severity (fail pipeline on bad data) |
| Execute SQL Task (pre/post) | Separate materialized view or a pipeline setup/teardown view |
| Expression Task | N/A — fold logic into the view definition |
| For Loop Container | Not needed — Spark handles parallelism; use set-based SQL |
| Foreach Container | Not needed — use set-based SQL with joins/unions |
| Checkpoint / Restart | SDP built-in checkpointing (idempotent refresh) |

### Data Flow — Sources

| SSIS Source | Databricks Pattern |
| --- | --- |
| OLE DB Source | `SELECT ... FROM catalog.bronze.<table>` (with explicit columns) |
| Flat File Source | Bronze table already ingested the file; reference the bronze table |
| ADO.NET Source | Same as OLE DB — reference the bronze equivalent |
| Raw File Source | Bronze table equivalent; reference it directly |
| XML Source | Bronze table equivalent (parsed into tabular form during ingestion) |
| Script Source | Bronze table equivalent (data already landed) |

### Data Flow — Transformations

| SSIS Transformation | Databricks SQL Pattern |
| --- | --- |
| **Data Conversion** | `CAST(col AS type)` — always use explicit casts in SELECT list |
| **Derived Column** | SQL expression in SELECT: `col_a + col_b AS total_amount` |
| **Copy Column** | Duplicate in SELECT: `col AS alias_col` |
| **Lookup** | `LEFT JOIN` / `INNER JOIN` on the lookup table (usually another bronze or Silver table) |
| **Lookup (no-match redirect)** | `LEFT JOIN ... WHERE lookup_key IS NULL` for no-match rows; use an SDP `WARN` expectation on the join column |
| **Merge Join** | `INNER JOIN` / `LEFT JOIN` / `FULL OUTER JOIN` with matching sort (not needed in Spark — no sort required before join) |
| **Merge** | `UNION ALL` for stacking rows; `MERGE INTO` for upsert (rare in SDP — prefer full refresh with dedup) |
| **Union All** | `UNION ALL` |
| **Merge (Union output)** | `UNION ALL` across the input queries |
| **Aggregate** | `GROUP BY` with aggregate functions (`SUM`, `COUNT`, `AVG`, `MIN`, `MAX`) |
| **Sort** | Generally unnecessary — Spark does not require pre-sorted input. Use `ORDER BY` only in final Gold presentation if needed. Remove Sort components entirely. |
| **Conditional Split** | Multiple `WHERE` clauses or `CASE WHEN` — emit separate views for each output path, or use a single view with a `category` column |
| **Multicast** | Reference the same CTE / subquery multiple times — no physical copy needed |
| **Row Count** | `SELECT COUNT(*)` — or capture as a pipeline metric via an expectation |
| **Row Sampling** | `TABLESAMPLE` or `RAND()` filter in SQL |
| **Pivot** | `PIVOT` clause or conditional aggregation with `CASE WHEN` + `SUM` |
| **Unpivot** | `UNPIVOT` clause or `LATERAL VIEW explode()` for array/map columns |
| **Character Map** | SQL string functions: `UPPER()`, `LOWER()`, `TRIM()`, `REPLACE()` |
| **Audit** | Add metadata columns: `current_timestamp()`, `input_file_name()`, or pipeline run context |
| **Slowly Changing Dimension** | See SCD section below |
| **Script Component (transformation)** | Python UDF in SDP — only when SQL cannot express the logic. Document the justification. |
| **Script Component (source)** | Python-defined streaming table reading from an external source — flag for separate ingestion pipeline |

### Data Flow — Destinations

| SSIS Destination | Databricks Pattern |
| --- | --- |
| OLE DB Destination (Silver) | Materialized View or Streaming Table in Silver schema |
| OLE DB Destination (Gold) | Materialized View in Gold schema (thin, aggregated) |
| Flat File Destination | N/A — output is a UC table; export to files is a separate concern |
| Raw File Destination | N/A — output is a UC table |
| Recordset Destination | N/A — use a temp view or CTE within the pipeline |
| Error Output (redirect rows) | SDP `REJECT` expectation — quarantined records go to the auto-generated quarantine table |

## SSIS Error Handling → SDP Expectations

SSIS redirects error rows to error outputs. In SDP, use **Expectations** with severity levels:

```sql
-- Map SSIS error outputs to Expectations
CONSTRAINT expect_not_null_business_key EXPECT (business_key IS NOT NULL) ON VIOLATION REJECT;
CONSTRAINT expect_valid_amount EXPECT (amount >= 0) ON VIOLATION WARN;
CONSTRAINT expect_critical_id EXPECT (id IS NOT NULL) ON VIOLATION FAIL;
```

### Mapping Severity to SSIS Behavior

| SSIS Error Handling | SDP Severity | Behavior |
| --- | --- | --- |
| Redirect rows to error output | `REJECT` | Quarantined to separate table, pipeline continues |
| Ignore failure (log and proceed) | `WARN` / `INFO` | Logged in quality metrics, pipeline continues |
| Fail component (stop data flow) | `CRITICAL` (FAIL) | Pipeline fails immediately |

## SCD (Slowly Changing Dimension) Patterns

SSIS includes a built-in SCD wizard. In Databricks, handle SCD via SQL patterns in SDP:

### SCD Type 1 (Overwrite)

Simplest — just take the latest record using `ROW_NUMBER()`:

```sql
-- SCD Type 1: keep latest version of each business key
-- Replace col1, col2, ... with explicit column names from the source table
SELECT col1, col2, business_key, ingest_timestamp
FROM (
  SELECT col1, col2, business_key, ingest_timestamp,
         ROW_NUMBER() OVER (PARTITION BY business_key ORDER BY ingest_timestamp DESC) AS rn
  FROM STREAM(catalog.bronze.<source_table>)
) WHERE rn = 1
```

### SCD Type 2 (Historical Tracking)

Use effective-from / effective-to dates. This is an **open question** in the ADR (section 11) — confirm the organization's chosen SCD approach before implementing. General pattern:

```sql
-- SCD Type 2: track historical changes with validity windows
SELECT
  business_key,
  attribute_1,
  attribute_2,
  effective_from,
  effective_to,
  is_current
FROM (
  -- Detect changes and build history windows
  -- (full pattern depends on the organization's SCD decision)
)
```

> **Note**: The ADR explicitly lists row-change handling (SCD Type 1/2, bitemporal) as an open question still under evaluation. Always ask the user which SCD pattern to use before implementing.

## Assessment Phase

Before any migration work, perform an **assessment** of the SSIS packages. The goal is to produce a complexity report — **do not write any migration code during this phase**.

### Assessment Process

1. **Review the uploaded SSIS packages** — the SSIS packages (`.dtsx` files or exported descriptions) will be uploaded to Databricks and made available directly. Read and analyze each package's structure (control flow tasks, data flows, sources, transformations, destinations) from the uploaded files. No need to ask the user to describe them.
2. **Identify STG vs. DWH packages** — note which packages are staging (STG) and which are data warehouse (DWH). These will be merged into a single SDP file during migration.
3. **Catalog external SQL dependencies** — list all stored procedures, views, and functions referenced by each package. Check whether the corresponding `.sql` files are available in the sub-folder. **Flag any missing SQL files** and ask the user to provide them.
4. **Count and classify components** — for each package, tally:
   - Number of control flow tasks
   - Number of data flow tasks
   - Number and types of transformations (simple vs. complex)
   - Whether Script Components are used (adds complexity)
   - Whether SCD logic is present
   - Whether there are complex expressions or parameterized logic
   - **Whether the package is a pure orchestrator** (only Execute Package tasks, no data flow or transformation logic) — mark as **Skip** in the report
5. **Rate complexity** — assign a complexity level per package:

   | Level | Criteria | Migration Effort |
   | --- | --- | --- |
   | **Low** | Package only runs stored procedures (Execute SQL Task calling stored procedures, no data flow tasks) | Direct SQL translation — use the stored procedure `.sql` files as the basis |
   | **Medium** | Package uses SSIS data flow components (sources, transformations, destinations) to move and transform data | Moderate SQL with CTEs, joins, and Expectations |
   | **High** | Script Components, complex SCD logic, many inter-package dependencies, complex expressions | Requires manual review and possible Python UDFs |

6. **Produce the report** — present a summary table:

   | Package Name | STG/DWH | Complexity | Components | External SQL Files | Missing Files | Notes |
   | --- | --- | --- | --- | --- | --- | --- |
   | ... | ... | Low/Medium/High/Skip | count + types | list | list or ✓ | ... |

   Packages marked **Skip** are pure orchestrators and will not be converted. If a skipped package contains side logic, note it for extraction into separate SQL files.

7. **Ask the user to confirm** — after presenting the report, ask the user which packages to migrate first and whether any missing SQL files can be provided.

## Step-by-Step Migration Process

When migrating an SSIS package, follow this process:

### 1. Inventory and Merge STG + DWH Packages

- Identify all **Control Flow** tasks and their precedence constraints.
- For each Data Flow Task, list: sources, transformations (in order), and destinations.
- Note any **variables**, **parameters**, and **expressions** used.
- Identify **error handling** — which components redirect error rows? (Map to Expectations; ignore SSIS logging/event handlers.)
- **Skip pure orchestrators**: If the package only executes other packages with no data transformation logic, **do not convert it**. SDP handles dependencies natively.
- **Extract side logic from orchestrators**: If an otherwise orchestrator-only package contains additional logic (pre/post SQL, checks, variable-driven flow), extract that logic into **separate SQL files** — one per table it creates or updates. Convert those files individually as Silver targets.
- **Merge STG and DWH**: If the SSIS package has separate STG and DWH packages for the same business entity, combine their logic into a **single SDP SQL file**. The STG cleansing steps become CTEs or intermediate views feeding into the Silver target.
- **Check for SQL files**: Verify all stored procedure / view SQL files referenced by the package are available in the sub-folder. If any are missing, **stop and ask the user**.

### 2. Map to Bronze Sources

- For each SSIS source, identify the corresponding bronze qualified table.
- If a bronze table doesn't exist, flag it for the user — bronze ingestion must be completed first.
- Note source columns and types; they may differ from SSIS metadata.

### 3. Classify Destination Layer

- By default, the SSIS package converts to a **Silver** dataset (cleaned, conformed, enterprise model).
- Only convert to **Gold** if the user **explicitly requested it** — otherwise stop at Silver.
- Does it contain **Silver Insights** (complex analytics, heavy aggregations, derived metrics)? If so, flag for the silver_insights sub-layer.
- Use this classification to pick the target schema and apply the right conventions.

### 4. Build the SDP Pipeline

- Create a streaming table or materialized view for each SSIS Data Flow destination.
- Chain transformations as SQL CTEs within the view definition.
- Replace SSIS error outputs with Expectations.
- Add mandatory column and table comments.
- Add recommended UC tags.
- Ensure explicit schema — no `SELECT *`.

### 5. Validate Quality

- Verify every Silver table has at least one Expectation.
- Confirm severity levels match the original SSIS error handling intent.
- Check that deduplication logic handles bronze duplicates.

### 6. Document & Hand Off

- Provide a mapping table showing SSIS component → SDP equivalent for the user.
- Note any SSIS features that have no direct equivalent and how they were handled.
- Flag any open questions (e.g., SCD approach, bronze tables missing).

## Common SSIS Patterns → SQL Templates

### Pattern: Bronze Cleansing with Dedup (Silver)

```sql
-- Mapped from: SSIS Data Flow with Sort + Aggregate + OLE DB Destination
CREATE OR REFRESH STREAMING TABLE catalog.silver.customer_clean (
  -- Explicit schema
  customer_id STRING COMMENT 'Unique identifier for the customer',
  customer_name STRING COMMENT 'Full legal name of the customer',
  email STRING COMMENT 'Primary email address',
  phone STRING COMMENT 'Primary phone number',
  ingest_timestamp TIMESTAMP COMMENT 'When the record was ingested from source',
  -- Data quality expectations
  CONSTRAINT expect_customer_id_not_null EXPECT (customer_id IS NOT NULL) ON VIOLATION REJECT,
  CONSTRAINT expect_email_format EXPECT (email LIKE '%@%.%') ON VIOLATION WARN
)
COMMENT 'Cleansed and deduplicated customer data from bronze source'
TBLPROPERTIES (
  'quality' = 'silver',
  'keys' = 'customer_id'
)
AS SELECT
  DISTINCT
  CAST(cust_id AS STRING) AS customer_id,
  TRIM(full_name) AS customer_name,
  LOWER(TRIM(email_addr)) AS email,
  TRIM(phone_num) AS phone,
  ingest_ts AS ingest_timestamp
FROM STREAM(catalog.bronze.customers);
```

### Pattern: Lookup + Conditional Split (Silver)

```sql
-- Mapped from: SSIS Lookup (dim table) + Conditional Split (valid/invalid)
CREATE OR REFRESH MATERIALIZED VIEW catalog.silver.order_enriched (
  order_id STRING COMMENT 'Unique order identifier',
  customer_id STRING COMMENT 'FK to silver.customer_clean',
  customer_name STRING COMMENT 'Customer name at time of order',
  order_amount DECIMAL(18,2) COMMENT 'Total order value in source currency',
  order_status STRING COMMENT 'Order lifecycle status',
  is_valid_order BOOLEAN COMMENT 'True if customer lookup succeeded',
  CONSTRAINT expect_order_id_not_null EXPECT (order_id IS NOT NULL) ON VIOLATION REJECT,
  CONSTRAINT expect_amount_positive EXPECT (order_amount >= 0) ON VIOLATION WARN
)
COMMENT 'Orders enriched with customer lookup; invalid lookups flagged'
AS SELECT
  o.order_id,
  o.customer_id,
  COALESCE(c.customer_name, 'UNKNOWN') AS customer_name,
  CAST(o.amount AS DECIMAL(18,2)) AS order_amount,
  o.status AS order_status,
  c.customer_id IS NOT NULL AS is_valid_order
FROM LIVE(catalog.bronze.orders) o
LEFT JOIN LIVE(catalog.silver.customer_clean) c
  ON o.customer_id = c.customer_id;
```

### Pattern: Gold Aggregation from Silver (Only When Explicitly Requested)

> **This pattern is only used when the user explicitly asks for Gold conversion.** Default migrations stop at Silver.

```sql
-- Mapped from: SSIS Aggregate + OLE DB Destination (presentation table)
CREATE OR REFRESH MATERIALIZED VIEW catalog.gold.customer_order_summary (
  customer_id STRING COMMENT 'PK — unique customer identifier',
  customer_name STRING COMMENT 'Customer name',
  total_orders INTEGER COMMENT 'Count of all orders for this customer',
  total_revenue DECIMAL(18,2) COMMENT 'Sum of all order amounts',
  avg_order_value DECIMAL(18,2) COMMENT 'Average order value',
  last_order_date DATE COMMENT 'Most recent order date',
  CONSTRAINT expect_customer_id_not_null EXPECT (customer_id IS NOT NULL) ON VIOLATION REJECT
)
COMMENT 'Customer-level order summary for BI dashboards — last 12 months only'
TBLPROPERTIES (
  'quality' = 'gold',
  'keys' = 'customer_id'
)
AS SELECT
  customer_id,
  MAX(customer_name) AS customer_name,
  COUNT(*) AS total_orders,
  SUM(order_amount) AS total_revenue,
  AVG(order_amount) AS avg_order_value,
  MAX(order_date) AS last_order_date
FROM LIVE(catalog.silver.order_enriched)
WHERE order_date >= DATE_ADD(CURRENT_DATE(), -365)
GROUP BY customer_id;
```

## SSIS Variables and Parameters

SSIS uses variables and parameters for dynamic values. In SDP SQL:

| SSIS Construct | Databricks Equivalent |
| --- | --- |
| Package parameter | Pipeline setting or DAB variable |
| Variable (runtime) | SQL expression or CTE value |
| Expression on task | Inline SQL expression |
| Property expression | DAB configuration / pipeline setting |
| Project parameter | DAB variable in `databricks.yml` |

- Pipeline-level parameters (e.g., date ranges, source catalog names) should be configured via **DAB variables** in `databricks.yml`.
- Row-level dynamic values should be computed in SQL using `CURRENT_DATE()`, `current_timestamp()`, etc.

## Checklist Before Presenting Migration Results

Before presenting the migrated pipeline to the user, verify:

- [ ] Pure orchestrator packages skipped (only Execute Package tasks, no data logic). Side logic extracted to separate SQL files if present.
- [ ] SSIS logging and event handlers excluded — SDP handles observability natively.
- [ ] All stored procedure / view SQL files were available and referenced. Missing files flagged to user.
- [ ] No `SELECT *` — every column explicitly listed with casts.
- [ ] Every Silver table has at least one data quality Expectation.
- [ ] Table comment describes business purpose (not technical implementation).
- [ ] Every column has a meaningful business comment.
- [ ] Deduplication applied where bronze may have duplicates.
- [ ] Error outputs mapped to Expectations with appropriate severity.
- [ ] Gold tables created **only if explicitly requested** by the user.
- [ ] If Gold was requested: select only needed columns, aggregate to consumer granularity, document PK/FK.
- [ ] No unnecessary Sort components carried over.
- [ ] Script Component usages have documented justification for Python.
- [ ] SCD approach confirmed with user if the package tracks dimension changes.
- [ ] All bronze source tables exist — flag any missing ones.
- [ ] UC tags recommended in comments or TBLPROPERTIES.

## Limitations & What This Skill Cannot Do

- **Bronze ingestion** is out of scope — this skill assumes bronze qualified tables already exist. If they don't, tell the user to set up ingestion first.
- **SCD Type 1/2** final approach is an open question in the ADR — always confirm with the user before implementing.
- **Script Components** with complex .NET logic may require manual Python rewrite — flag these for user review.
- **Custom SSIS components / third-party extensions** have no automatic mapping — analyze their logic and translate to SQL or Python on a case-by-case basis.
- **Stored procedure / view SQL files** must be provided by the user as `.sql` files in a sub-folder. If a file is missing, **do not proceed** — ask the user to provide it.
- **Pure orchestrator packages** (only Execute Package tasks) are not converted — SDP handles inter-table dependencies natively. Side logic in such packages is extracted into separate SQL files per table.
- **Logging and event handlers** are intentionally ignored — SDP provides native observability, alerting, and data quality tracking.
- **SSIS configuration files** (dtsx XML) are not parsed automatically — the user must describe or share the package structure.
- **Connection managers** (OLE DB, ODBC, file paths) map to bronze table references — the actual connections are handled at the bronze ingestion layer, not here.
