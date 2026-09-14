# Architecture Decision Record: ShebaBricks Silver & Gold Engineering Standard

---

## **Status**

✅ **Approved** | Date: 2026-09-14 | Owner: Data Architecture Team

---

## **Decision**

We will establish a **comprehensive, organization-wide standard** for Silver and Gold layer data products using:
- **Lakeflow Spark Declarative Pipelines (SDP)** as the default framework
- **SQL-first approach** with explicit schemas
- **Serverless compute** by default
- **Materialized Views** for most Silver/Gold datasets
- **Unity Catalog** for governance, metadata, and security
- **Git + Declarative Automation Bundles** for version control and deployment
- **Explicit data quality rules** on all Silver tables
- **Rich metadata and documentation** for AI/analytics consumption
- **Group-based permissions** and ABAC policies

---

## **Context**

ShebaBricks required a comprehensive, organization-wide standard for designing, implementing, governing, and operating data products in the Silver and Gold layers of the lakehouse architecture.

### **Requirements**

#### **Functional**
* Establish Silver as the authoritative, reusable enterprise data model
* Define Gold as a thin, consumer-oriented presentation layer
* Standardize technology stack and implementation patterns
* Ensure all datasets are well-documented, governed, and AI-ready
* Enable consistent data quality enforcement

#### **Non-Functional**
* Support modern lakehouse patterns with Unity Catalog governance
* Mandate version control and repeatable deployment
* Provide clear guidance on compute, performance, and operations
* Cost optimization through serverless and managed services
* Auditability and compliance with lineage tracking

---

## **Scope**

This standard applies to:
- **Silver layer**: Clean, integrated, conformed, modeled business data (enterprise dimensional model) - simple, fast-loading, information-focused
- **Silver insights sub-layer**: Complex transformations, derived metrics, and time-intensive analytics built on Silver foundation
- **Gold layer**: Thin, consumer-ready presentation datasets (aggregated, simplified, optimized)
- All new Silver/Gold implementations

**Out of Scope**:
- Bronze layer ingestion patterns (covered in separate ADR)
- Ad-hoc analytics notebooks
- ML feature engineering (separate guidance)
- Streaming use cases requiring sub-second latency

---

## **Design Principles**

1. **Silver as Source of Truth**: Silver is the authoritative, reusable enterprise model with complete historical data
2. **Gold is Thin**: Gold is disposable presentation layer, not a duplicate dimensional model; may subset history for performance or business needs
3. **Silver is Simple**: Silver contains information (cleaned, integrated data), not insights; complex analytics and derived metrics belong in silver_insights sub-layer
4. **Explicit Over Implicit**: Explicit schemas, quality rules, and documentation required
5. **SQL-First**: Prefer SQL for readability and maintainability; Python only when justified
6. **Governance by Default**: Unity Catalog metadata, lineage, and permissions are mandatory
7. **Configuration Over Code**: Use declarative patterns (SDP, Materialized Views, DABs)
8. **Version Everything**: All production code in Git with proper CI/CD
9. **Quality is Non-Negotiable**: Every Silver table has explicit quality rules
10. **AI-Ready**: Rich metadata enables Databricks Genie and AI-powered analytics

---

## **Architectural Pattern**

### **Layered Architecture**

```
Bronze Layer (Raw/Qualified)
  • Lakeflow Connect or Auto Loader ingestion
  • Minimal transformation
  • Schema enforcement
        ↓
Silver Layer (Enterprise Model)
  • Lakeflow SDP or Materialized Views
  • Clean, integrated, conformed business data
  • Dimensional model (facts + dimensions)
  • Complete historical data retention
  • Simple transformations, fast loading
  • Information, not insights
  • Explicit data quality rules
  • Rich metadata (comments, tags, lineage)
  • Serverless compute
        ↓
Silver Insights Sub-Layer (Complex Analytics)
  • Complex transformations and computations
  • Longer processing time acceptable
  • Derived metrics, statistical calculations
  • Advanced analytics and aggregations
  • Builds on Silver foundation
        ↓
Gold Layer (Presentation)
  • Lakeflow SDP or Materialized Views
  • Consumer-specific aggregations
  • Simplified relationships
  • Subset of Silver columns and/or history
  • Historical filtering for performance/business needs
  • Primary/foreign keys documented
  • Serverless compute
        ↓
Consumers (BI, ML, Apps)
  • Dashboards, reports, notebooks
  • ML feature stores
  • API endpoints
```

### **Deployment Model**

* **Git**: All production SQL/Python code
* **Declarative Automation Bundles (DABs)**: Infrastructure as code
* **Environment Promotion**: DEV → TEST → PROD

---

## **Key Design Decisions**

### **1. Lakeflow SDP as Default Framework**
✅ **Decision**: Use Lakeflow Spark Declarative Pipelines (SDP) as the default framework for Silver and Gold transformations

**Why**: 
- Built-in data quality Expectations with severity levels
- Column-level lineage and documentation
- Incremental processing with STREAM() and LIVE()
- Observability dashboard out of the box
- Declarative SQL reduces complexity
- Serverless support for cost optimization

**Rejected**: 
- Notebooks (no quality tracking, manual lineage, operational burden)
- Custom Spark jobs (maintenance overhead, reinventing the wheel)
- Airflow + Spark (additional orchestration complexity)

---

### **2. SQL-First with Explicit Schemas**
✅ **Decision**: SQL is the default language

**Why**:
- Readability and maintainability for broader team
- Explicit schemas surface breaking changes immediately
- Forces deliberate decisions on schema evolution
- Enables proper column-level documentation
- Prevents unintended data exposure

**Rejected**:
- Python-first (higher complexity, smaller talent pool)

**Exception**: Python allowed when justified (complex custom logic, ML preprocessing, external library integration)

---

### **3. Serverless Compute by Default**
✅ **Decision**: Use serverless compute for all Silver and Gold pipelines unless technically infeasible

**Why**:
- Faster startup (seconds vs minutes)
- Automatic scaling based on workload
- Cost-efficient for triggered/scheduled workloads
- No cluster management overhead
- Latest Databricks Runtime and optimizations

**Rejected**:
- Classic compute as default (slower startup, manual tuning, over-provisioning)

**Exception**: Classic compute allowed when serverless limitations encountered (legacy libraries, specific Spark configs, regulatory constraints)

---

### **4. Materialized Views as Default Dataset Type**
✅ **Decision**: Prefer Materialized Views for most Silver/Gold datasets; use Streaming Tables when incremental semantics are natural

**Why**:
- Simpler mental model (like a cached view)
- Automatic incremental refresh where possible
- Lower operational complexity
- Sufficient for most batch transformation patterns
- Definition in Git (unlike tables)

**Rejected**:
- Streaming Tables for everything (unnecessary complexity for simple batch transforms)
- Views without materialization (poor query performance)
- Tables - Definition not in Git, alter requires explicit command

**Exception**: Streaming Tables when STREAM() semantics are essential (CDC, event-time processing, watermarking)

---

### **5. Explicit Data Quality on All Silver Tables**
✅ **Decision**: Every Silver table must have explicitly defined data quality rules using SDP Expectations

**Why**:
- Surfaces data issues early in the pipeline
- Creates observable quality metrics
- Prevents garbage-in-garbage-out scenarios
- Documents expected data contracts
- Enables quarantine/remediation workflows

**Quality Rule Categories**:
- Not Null, Data Type, Domain, Uniqueness, Referential Integrity, Business Rule, Range/Validity, Freshness

**Severity Levels**:
- INFO/WARN (track but don't block)
- REJECT (quarantine invalid records)
- CRITICAL (fail pipeline)

**Rejected**:
- Ad-hoc quality checks in notebooks (not observable, not centralized)
- Failing entire pipelines on single invalid records (too brittle)

---

### **6. Git + Declarative Automation Bundles**
✅ **Decision**: All production code in Git; all deployments via Declarative Automation Bundles (DABs)

**Why**:
- Version control and audit trail
- Code review before production changes
- Repeatable, automated deployments
- Infrastructure as code
- Environment consistency (DEV/TEST/PROD use same code)
- Rollback capability

**Workflow**: Branch → Development → Test → Pull Request/Review → Deploy

**Rejected**:
- Workspace notebooks as production systems (no version control, manual changes, drift)
- Manual deployments (error-prone, not repeatable)
- Terraform/CloudFormation (DABs are Databricks-native)

---

### **7. Rich Metadata for AI Readiness**
✅ **Decision**: Every Silver and Gold table/column must have meaningful comments; Unity Catalog tags recommended

**Why**:
- Enables Databricks Genie (AI-powered SQL generation)
- Self-documenting data catalog
- Reduces tribal knowledge
- Improves discoverability
- Supports data governance

**Required Metadata**:
- Table comment (business meaning, not technical description)
- Column comments (semantic meaning, not field name repetition)

**Recommended UC Tags**:
- business_domain, source_system, data_owner, technical_owner, sensitivity, pii_classification, refresh_frequency

**Rejected**:
- Metadata in external wikis/documentation (becomes stale, not discoverable in UC)
- Generic comments ("Customer ID" for customer_id column)

---

### **8. Group-Based Permissions and ABAC**
✅ **Decision**: Group-based permissions only; ABAC policies for consistent row filtering and column masking

**Why**:
- Scalable permission management
- Audit-friendly (group membership changes tracked)
- Consistent security across tables
- Centralized policy enforcement
- Supports least privilege access

**Pattern**:
- Datasets organized by business domain
- Every dataset has business owner and technical owner
- RLS/CLS in Unity Catalog where centralized enforcement needed

**Rejected**:
- Individual user permissions (doesn't scale, audit nightmare)
- View-based security everywhere (proliferation of security views, maintenance burden)

**Exception**: Service principals may receive direct grants

---

### **9. Gold as Thin Presentation Layer**
✅ **Decision**: Gold selects only required columns, aggregates appropriately, and simplifies Silver relationships; does not recreate the dimensional model. Gold may subset historical data when needed.

**Why**:
- Avoids duplicate business logic
- Silver remains single source of truth with complete history
- Gold is disposable and regenerable
- Reduces data duplication
- Enables consumer-specific optimizations
- Historical filtering improves query performance

**Gold Standards**:
- Define primary key or composite key
- Document foreign key relationships
- Select only needed columns
- Aggregate to consumer granularity
- Filter history when appropriate (e.g., "last 12 months", "current year", "active records only")

**Historical Subsetting in Gold**:
- Performance optimization (reduce scan size for BI dashboards)
- Business requirement ("current view" vs. "historical analysis")
- Compliance (archive old data differently)
- User experience (faster dashboard loads)

**Silver Always Retains Full History**:
- Complete audit trail
- Historical analysis capability
- Point-in-time reconstruction
- Regulatory compliance
- Gold can be regenerated with different time windows as needed

**Rejected**:
- Gold as another dimensional model (duplicate Silver, confusion about source of truth)
- Duplicating entire Silver datasets for "convenience"
- Archiving/deleting history from Silver to "optimize" (Silver must be complete)

---

### **10. Silver Insights Sub-Layer for Complex Transformations**
✅ **Decision**: Separate complex, time-intensive transformations into a `silver_insights` sub-layer; keep core Silver simple and fast-loading

**Why**:
- Silver should load quickly and provide foundational information
- Complex computations (statistical analysis, heavy aggregations, derived metrics) can slow Silver refresh
- Separation enables different refresh schedules and compute resources
- Consumers needing raw information access Silver; consumers needing insights access silver_insights
- Prevents Silver from becoming a bottleneck

**Silver Layer**:
- Simple joins, filters, type conversions
- Data cleaning and conformance
- Dimensional modeling (facts + dimensions)
- Information: "What happened?"
- Fast refresh (minutes)

**Silver Insights Sub-Layer**:
- Complex calculations (rolling windows, statistical metrics)
- Heavy aggregations across large time periods
- Advanced analytics (trend analysis, scoring)
- Insights: "What does it mean?"
- Longer refresh acceptable (hours)

**Rejected**:
- Mixing complex analytics in core Silver (creates bottleneck, slows all consumers)
- Making everything a Gold dataset (loses reusability, duplicates logic)

**Exception**: Simple aggregations that are fast and widely reused may remain in Silver

---

### **11. Change Data Feed (CDF) Not Mandatory**
✅ **Decision**: CDF is optional; use only when required by specific use case semantics

**Why**:
- SDP and Materialized Views handle most incremental processing naturally
- CDF adds storage overhead and complexity
- Not needed unless downstream requires explicit insert/update/delete operations

**Use CDF When**:
- SCD Type 2 or bitemporal tracking required
- Downstream system needs CDC semantics
- Low-latency streaming with change operations

**Rejected**:
- CDF by default (unnecessary overhead for most use cases)

---

## **Implementation Standards Summary**

| Category | Standard |
| --- | --- |
| **Framework** | Lakeflow Spark Declarative Pipelines (SDP) |
| **Language** | SQL (Python when justified) |
| **Compute** | Serverless |
| **Dataset Type** | Materialized Views (Streaming Tables for true streaming) |
| **Schema** | Explicit columns; `SELECT *` prohibited |
| **Quality** | Explicit Expectations on all Silver tables |
| **Metadata** | Table and column comments mandatory |
| **Tags** | UC tags for domain, owner, sensitivity, PII |
| **Security** | Group-based permissions, ABAC policies |
| **Version Control** | Git for all production code |
| **Deployment** | Declarative Automation Bundles |
| **Silver Insights** | Complex transformations in sub-layer (optional) |
| **Gold** | Thin presentation layer with PK/FK documentation |
| **Operations** | Idempotent, observable, alerting on failures |

---

## **Benefits**

✅ **Consistency**: Standardized patterns across all teams and projects

✅ **Quality**: Explicit schema contracts and comprehensive data quality enforcement

✅ **Governance**: Clear ownership, group-based security, Unity Catalog integration

✅ **AI Readiness**: Rich metadata enables Databricks Genie and AI-powered analytics

✅ **Maintainability**: Git-based version control and automated deployment

✅ **Observability**: SDP dashboards, quality metrics, lineage tracking

✅ **Performance**: Serverless auto-scaling, Databricks-managed optimization

✅ **Reusability**: Silver as authoritative model eliminates duplicate logic

✅ **Agility**: Thin Gold layer enables rapid consumer implementations

✅ **Cost Efficiency**: Serverless compute, no data duplication

✅ **Developer Experience**: SQL-first, declarative, less code

✅ **Compliance**: Lineage, audit logs, RLS/CLS, data classification

---

## **Constraints**

- Git repository and DAB configuration required
- Data quality rules must be defined upfront
- Metadata documentation is mandatory
- Code review process for all changes
- Exception process for deviations


---

## **Exceptions**

**All deviations from this standard require documented technical or business justification.**

Valid exceptions may include:

* Legacy systems during transition period (with sunset date)
* Specialized workloads requiring classic compute features
* External constraints (vendor limitations, regulatory requirements)
* Technical limitations of declarative pipelines for specific use cases
* Python requirements for complex custom logic or external libraries
* Performance requirements not met by standard patterns (after profiling)

---

## **Benefits by Stakeholder**

### **For Data Engineers**
✅ Clear patterns to follow (reduce decision fatigue)

✅ SQL-first approach (lower complexity than Spark Python)

✅ Built-in observability (SDP dashboards, quality metrics)

✅ Easy debugging (lineage, expectations, quarantine datasets)

✅ No infrastructure management (serverless, managed services)

✅ Less boilerplate (declarative > imperative)

---

### **For Analytics Engineers**
✅ Reliable, well-documented Silver layer (trustworthy source)

✅ Fast Gold creation (thin layer, reuse Silver logic)

✅ Databricks Genie support (rich metadata enables AI SQL)

✅ Unity Catalog lineage (understand upstream dependencies)

✅ Quality metrics (know when data issues occur)

---

### **For Data Scientists**
✅ Clean, modeled data (no wrangling raw files)

✅ Rich metadata (understand data meaning without tribal knowledge)

✅ Quality guarantees (Silver expectations prevent garbage data)

✅ Fast access (optimized Silver/Gold layers, no raw file parsing)

---

### **For BI Developers**
✅ Gold layer tailored to use case (aggregated, simplified)

✅ Primary/foreign keys documented (proper joins)

✅ Fast query performance (materialized, optimized)

✅ Stable schemas (explicit columns, controlled evolution)

---

### **For Data Platform Team**
✅ Standardized patterns (easier to support)

✅ Reduced operational burden (managed services, serverless)

✅ Governance by default (Unity Catalog, lineage, quality)

✅ Scalable architecture (add domains/teams easily)

✅ Cost visibility (tagged resources, serverless efficiency)

✅ Security enforcement (group-based, ABAC policies)

---

### **For Organization**
✅ Regulatory compliance (lineage, audit trails, data classification)

✅ Data quality tracking (measurable, reportable)

✅ Fast time-to-value (templates, standards accelerate delivery)

✅ Reduced maintenance cost (managed services, less custom code)

✅ AI/ML readiness (governed, documented, quality data)

✅ Talent flexibility (SQL skills more common than deep Spark expertise)

---

## **Conclusion**

The **ShebaBricks Silver & Gold Engineering Standard** provides a comprehensive, modern approach to building governed, high-quality, AI-ready data products in the lakehouse. By standardizing on Lakeflow SDP, serverless compute, explicit schemas, and Unity Catalog governance, we enable teams to deliver reliable data products faster while maintaining consistency, quality, and compliance.

### **Key Principles**

✅ **Silver as Source of Truth** (authoritative enterprise model with complete history)

✅ **Gold is Thin** (disposable presentation layer; may subset history)

✅ **Silver is Simple** (information not insights; complex analytics in silver_insights)

✅ **Explicit Over Implicit** (schemas, quality, documentation)

✅ **SQL-First** (readability and maintainability)

✅ **Governance by Default** (Unity Catalog, lineage, permissions)

✅ **Configuration Over Code** (declarative patterns)

✅ **Version Everything** (Git + DABs)

✅ **Quality is Non-Negotiable** (explicit expectations)

✅ **AI-Ready** (rich metadata for Genie)

✅ **Performance Through Simplicity** (serverless, managed optimization)

---

## **Related Decisions**

* [ADR: ShebaBricks Multi-Domain Ingestion Framework](#) (Bronze layer standards)
* Unity Catalog Naming Conventions and Organization Structure
* CI/CD Pipeline Configuration and Deployment Automation
* Monitoring and Alerting Infrastructure
* Data Quality Framework and Quarantine Processes
* Exception Request and Approval Process

---

## **References**

* [Databricks Lakeflow Spark Declarative Pipelines Documentation](https://docs.databricks.com/en/delta-live-tables/index.html)
* [Unity Catalog Best Practices](https://docs.databricks.com/en/data-governance/unity-catalog/best-practices.html)
* [Declarative Automation Bundles Guide](https://docs.databricks.com/en/dev-tools/bundles/index.html)
* [Databricks SQL Materialized Views](https://docs.databricks.com/en/sql/language-manual/sql-ref-syntax-ddl-create-materialized-view.html)
* [Databricks Serverless Compute](https://docs.databricks.com/en/serverless-compute/index.html)
* ShebaBricks Silver & Gold Engineering Standard (detailed implementation guide)

---

**Document Owner**: Data Architecture Team  
**Last Updated**: 2026  
**Review Cycle**: Quarterly or as major platform capabilities evolve
