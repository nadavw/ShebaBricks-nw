# Databricks Operability Blueprint
## Enterprise Medical Center · AWS · v3.0 (Condensed)

---

## 1. Overview

**Partners:** AllCloud (infra: AWS, workspace deployment, networking, PrivateLink, KMS, AWS Secrets Manager infra, JFrog infra) · OneDataAI (platform engineering + knowledge transfer to internal data engineer)

**Team:** 50 people — 2 Databricks experts, 48 beginners. No dedicated platform engineer internally; OneDataAI fills this role with transition plan.

**Workloads:** BI/SQL analytics · Data engineering (migrating from SSIS) · Genie Agents · AI/BI Dashboards · Databricks Apps · GenAI/LLM (Phase 1: multi-LLM translation project) · Near real-time (1-min latency)

**Current State:** Workspaces live on AWS, on-prem SQL connections active (Epic/Chameleon, Namer, Q-Flow), PrivateLink configured, no public access, PHI/PII on platform (no strict compliance mandate yet), secrets in AWS Secrets Manager, packages via JFrog.

**Principles:**
1. Secure by default — PHI/PII governance from day one
2. Paved roads over guardrails — templates, policies, CI/CD make the right way the easy way
3. Partners enable, not own — internal team progressively takes ownership
4. Speed with guardrails — Phase 1 in days, not months
5. Observability before optimization — see before you tune
6. Enablement is operability — 48 beginners need training

---

## 2. Responsibility Model (RACI)

### Layer Split

| Layer | Owner | Key Responsibilities |
|---|---|---|
| **Infrastructure** | AllCloud | AWS (VPC, subnets, PrivateLink, firewall), workspace deployment (Terraform), KMS/BYOK, S3 root storage, EntraID SCIM provisioning, AWS Secrets Manager setup + scopes, cluster types (AWS instance types), JFrog server + network access |
| **Workspace & Platform** | OneDataAI -> Internal | UC governance, cluster policies, CI/CD (DABs), JFrog integration, SAT/LakeWatch, AI Gateway, Genie/Dashboards/Apps governance, monitoring, enablement |
| **Shared** | AllCloud + OneDataAI | IaC parameters, initial UC setup, network troubleshooting, on-prem connectivity, incident response, AWS Secrets Manager access policies |

### Value & Delivery

**Value:** Eliminates ambiguity between AllCloud, OneDataAI, and the internal team. Prevents gaps where nobody owns a critical task (e.g., UC governance, secrets, monitoring). Enables faster incident routing -- each party knows immediately if an issue is infra (AllCloud), workspace/platform (OneDataAI), or shared. Without this, a medical center with two external partners gets "I thought they were handling that" on PHI access governance, cluster cost control, or AI Gateway configuration.

**Delivery:** RACI is signed off by all three parties and management. Communicated during onboarding for every new team member. Referenced during incident triage to determine ownership. Reviewed when scope changes (e.g., OneDataAI to internal ownership transition).

---

## 3. Operability Domains

### PlatformOps
Workspace admin, compute governance, connectivity, JFrog, SAT/LakeWatch, Genie/Dashboards/Apps config.
- Cluster policies by team/purpose (min 3: small/medium/no-unrestricted); enforce auto-termination, max workers
- SQL Warehouse right-sizing, auto-stop tuning
- On-prem connections (JDBC to Epic/Chameleon, Namer, Q-Flow), file shares, bidirectional flow
- JFrog: private PyPI via init scripts + pip config; credentials from AWS Secrets Manager; per-env repos (dev/staging/prod)
- SAT: audit log analysis (SecOps), cost analysis (FinOps), job/cluster monitoring (PlatformOps) — deployed by PlatformOps, consumed cross-domain
- LakeWatch: query perf (DataOps), data quality (DataOps), pipeline health (DataOps) — deployed by PlatformOps, consumed cross-domain
- Genie/Dashboards/Apps: governance patterns, entitlements, CI/CD via DABs

### SecOps
Identity, access, data governance, audit, compliance readiness.
- EntraID SCIM -> Databricks groups -> UC grants
- UC catalogs (clinical, operational, finance, sandbox); dynamic views for PHI masking; GRANT-based (no GRANT ALL in prod)
- Data classification tags on PHI/PII columns
- Audit logging via system tables; PHI access monitoring
- AWS Secrets Manager for ALL secrets (on-prem creds, JFrog, LLM API keys) — no hardcoded secrets
- Build for HIPAA/HITECH readiness now even if not mandated

### DataOps
Pipeline reliability, data quality, lineage, observability.
- Databricks Jobs + SDP with expectations (not-null, unique, range checks)
- UC lineage tracking; freshness monitoring (alert on stale Epic syncs)
- Failure handling: retries, alerting (Teams/email), dead-letter patterns
- Schema drift detection for Epic upgrades
- Pipeline health dashboard (LakeWatch-powered)
- Semantic layer: create and maintain UC metric views (business KPIs), UC domains, schema annotations, column descriptions — the data engineering foundation for Genie Ontology

### DevOps
Code-to-production lifecycle, CI/CD, shared libraries, templates.
- All code in Git (GitHub or Azure DevOps); no notebook-only dev in prod
- DABs for environment promotion (dev->staging->prod)
- CI/CD: automated testing, PR-based review, code quality gates (SQLFluff, Black/Ruff)
- Shared libraries via JFrog (on-prem connectors, PHI masking, data quality utilities)
- Starter templates: on-prem SQL extraction, file share ingestion, reporting pipeline, Apps deployment
- Semantic layer CI/CD: metric views, Pages, and domain definitions version-controlled and deployed via DABs

### FinOps
Cost visibility, budget monitoring, optimization, chargeback.
- Cost attribution via system tables (system.billing.usage) + SAT
- Budget alerts by workspace/team; waste detection (orphaned clusters, over-provisioned warehouses)
- Serverless vs. classic evaluation for BI workloads
- Showback by department (Next), chargeback (Later)

### MLOps
TBD — to be defined when ML use cases materialize. See TBD section.

### AIOps (ACTIVE Current)
AI Gateway (Unity Gateway), multi-LLM routing, guardrails, token monitoring.
- AI Gateway: multi-LLM routing (traffic splitting + fallback chains) for translation project
- Guardrails: PHI detection in prompts before egress; output filtering
- Token monitoring: per-request cost tracking, budget alerts
- Audit: all translation requests/responses logged
- Evaluation: BLEU, semantic similarity, human review workflow
- RAG infrastructure (Vector Search) for future GenAI use cases (Next phase)
- LLM API keys in AWS Secrets Manager; egress via controlled firewall (coordinate with AllCloud)

### Data Product Ops
Consumer-facing data products (Genie One, Genie Agents, AI/BI Dashboards, Databricks Apps) — lifecycle, quality, performance, usage analytics, ZeroOps.
- Genie One ops: adoption monitoring (active business users, session frequency), answer quality across all agents, discovery experience (can users find the right agent/dashboard?), ontology integration from consumer perspective
- Genie Agent ops: quality monitoring (answer accuracy, hallucination detection), confidence voting, trusted asset management, drift detection
- Dashboard ops: data freshness, load performance, usage analytics, lifecycle management (create -> update -> retire)
- Apps ops: health monitoring (uptime, response time, error rates), lifecycle management, performance optimization
- Consumer usage analytics: adoption tracking, satisfaction metrics, question success rates
- Consumer-facing SLAs: define and monitor SLAs for Genie response time, dashboard load time, app availability
- Genie ZeroOps: automated self-managing Genie operations — quality monitoring, drift detection, automated remediation (https://www.databricks.com/blog/introducing-genie-zeroops)
- Semantic layer quality: Genie Ontology monitoring — inferred context snippet accuracy, authority scoring, conflict resolution, certification workflows, Pages governance (business term definitions), ontology drift detection (https://docs.databricks.com/aws/en/genie/genie-ontology, https://www.databricks.com/blog/operationalizing-genie-ontology-your-data-stack)

**Semantic Layer Glossary:**

| Term | Plain-Language Explanation |
|---|---|
| **Genie Ontology** | Databricks' unified context layer that gives Genie a business-aware map of your organization. Combines human-defined context (UC semantics) with auto-extracted context (inferred snippets). |
| **Inferred context snippet** | A piece of business knowledge Genie automatically extracts from your existing assets (dashboards, queries, Genie Agents, metric views) — e.g., "revenue = sum of sales_amount where status = closed". |
| **Snippet accuracy** | Whether an auto-extracted snippet correctly represents the business logic it was extracted from. Wrong snippets lead to wrong Genie answers. |
| **Authority score** | A ranking Genie assigns to each snippet based on source (certified metric view > ad-hoc query), usage frequency, and freshness. Higher authority snippets win when there are conflicting definitions. |
| **Conflict resolution** | When two snippets define the same metric differently (e.g., two teams define 'active patient' differently), Genie uses authority score to pick the right one. Data Product Ops monitors and resolves these conflicts. |
| **Certification workflow** | The process of marking a data asset (metric view, table) as trusted/authoritative. Certified assets get higher authority scores, steering Genie toward vetted definitions. |
| **Pages governance** | Pages are governed, human-authored definitions of business concepts (e.g., 'What counts as a readmission?'). Data Product Ops owns the accuracy and lifecycle of these definitions. |
| **Ontology drift detection** | Monitoring whether the semantic layer is becoming stale — e.g., a metric view references a column that was renamed, or a business rule changed but the Page wasn't updated. |

### Notification Strategy
Cross-cutting delivery layer for all operability domains. Destinations configured once by PlatformOps at workspace level; all domains reference them.

**Notification Destinations** — *TBD: no decision yet on which destinations to adopt.*

Databricks supports the following destination types (all admin-configured at workspace level). The routing table below is a proposal — final destination selection per severity is pending.

**Destination Options (HL):**
- **Email** — Built-in, zero-setup. Good for audit trails and non-urgent summaries. Low visibility (easy to ignore). No on-call escalation.
- **Microsoft Teams** — Webhook-based or Genie app. Real-time visibility in team channels. Good for collaborative triage. Requires channel governance (one per domain). Not suitable for on-call paging.
- **Slack** — Webhook-based or Genie app. Similar to Teams but different org stack. Only relevant if Slack is the org standard (TBD vs. Teams).
- **PagerDuty** — Purpose-built for on-call incident management. Acknowledgement, escalation policies, on-call schedules. Overkill for low-severity alerts. Best for critical prod-only.
- **Webhook** — Generic HTTP POST to any endpoint. Maximum flexibility (ITSM, custom runbooks, auto-remediation). Requires custom receiver infrastructure. Phase 3+.

**Proposed Routing Rules (pending destination decision):**

| Severity | Event Examples | Candidate Destination | Domains |
|---|---|---|---|
| Critical | Prod pipeline failure, PHI access violation, AI Gateway down | PagerDuty + Teams/Slack | DataOps, SecOps, AIOps |
| High | Non-prod job failure, budget threshold exceeded, token budget alert | Teams/Slack | DataOps, FinOps, AIOps |
| Medium | DQ expectation failure, stale data, query perf degradation | Teams/Slack (per-domain channel) | DataOps, PlatformOps |
| Low | Cost trend, idle cluster, audit summary | Email | FinOps, PlatformOps, SecOps |

**Event Sources:**
- Job/pipeline built-in notifications (run failure, flow-level failure, maintenance start/complete)
- Databricks SQL Alerts (scheduled query + threshold condition → TRIGGERED / OK / ERROR)
- System tables + SQL Alerts (account-wide: billing spikes, audit anomalies, job-run failures, query duration)
- AI Gateway token/budget alerts

**Open Decision:** Which destination(s) to adopt and in which phase. Input needed from org IT / SecOps on existing collaboration tool (Teams vs. Slack) and incident management tool (PagerDuty vs. existing ITSM).

**Principles (destination-agnostic):**
- One channel per domain for routing clarity (whatever chat tool is chosen)
- Alert on sustained trends, not single points (avoid alert fatigue)
- Notify on recovery (OK) for critical alerts only
- Phase 1: email + chat tool (TBD which) · Phase 2: add on-call paging for prod · Phase 3: add webhook for ITSM

---

## 4. Phased Maturity Roadmap

### Phase 1: 04/10/2026 - 08/10/2026
**Exit criteria:** Users in groups, clusters have policies, pipelines in Git, cost visible, JFrog integrated, SAT deployed, AI Gateway live.

**Key milestones per domain:**

| Domain | Key Milestones | Owner | Days |
|---|---|---|---|
| PlatformOps | SQL Warehouse cluster created via Admin Bundel | Chen | |
| SecOps | Account-level groups created + synced to workspaces (`data_platform_admins`, `data_engineers`) | Chen | |
| DataOps |Creating Catalogs and Schema via Admin Bundle | Chen | |
| DevOps | Creating Example Bundle| Chen | |
| FinOps | Budget alerts configured (`budget_all_dev`: $500/$1K/$2K monthly thresholds) | Chen |
| PlatformOps | Notification destination decision made + configured (TBD which); routing rules documented | Chen |
| MLOps | TBD | | |
| AIOps | | | |
| Data Product Ops | | | |

### Phase 2: 11/10/2026 - 22/10/2026
**Exit criteria:** CI/CD for all code, PHI masked, DQ expectations in pipelines, cost attributed, team trained, LLM translation optimized, Genie ZeroOps approach adopted.

**Key milestones per domain:**

| Domain | Key Milestones | Owner | Days |
|---|---|---|---|
| PlatformOps | Tagging strategy| Chen, Nadav, Avi ||
| SecOps | SQL alerts for rules violations (TBD)| Chen |1|
| DataOps | | ||
| DevOps | Run Dashboard based on pipelines event log | Chen | 2|
| FinOps | Usage Dashboard (By Job/team/workspace) | Chen | 1|
| MLOps |  | |
| AIOps | | |
| Data Product Ops | | |

**Genie ZeroOps:** In next phases, adopt the Genie ZeroOps approach for automated Genie operations — self-managing Genie Agents with automated quality monitoring, drift detection, and remediation. See [Introducing Genie ZeroOps](https://www.databricks.com/blog/introducing-genie-zeroops).


### Phase 3: 25/10/2026 - 05/11/2026
**Exit criteria:** Production environment ready, CI/CD between dev and prod

**Key milestones per domain:**

| Domain | Key Milestones | Owner | Days |
|---|---|---|---|
| PlatformOps | | ||
| SecOps | prod permissions| Chen ||
| DataOps | prod data objects | ||
| DevOps | CI\CD automation, prod monitoring, alerts | Chen | |
| FinOps | prod usage monitoring | Chen | |
| MLOps |  | |
| AIOps | | |
| Data Product Ops | | |

---

## 5. Tool Stack

| Purpose | Tool |
|---|---|
| Infrastructure as Code | Terraform (AllCloud) |
| CI/CD | DABs + GitHub Actions or Azure DevOps |
| Orchestration | Databricks Jobs + SDP |
| Governance | Unity Catalog |
| Semantic Layer (business context) | UC Semantics: Metric Views, Domains, Pages, Certification/Deprecation + Genie Ontology (inferred context) |
| Secrets | AWS Secrets Manager-backed scopes |
| Identity | EntraID + SCIM |
| Package Management | JFrog (Artifactory, private PyPI) |
| Data Quality | SDP Expectations |
| Admin/Monitoring | SAT (audit, cost, jobs) |
| Lakehouse Health | LakeWatch |
| System Monitoring | Databricks System Tables + dashboards |
| ML Lifecycle | MLflow (managed) |
| GenAI Governance | Databricks AI Gateway (Unity Gateway) |
| RAG / Vector Search | Databricks Vector Search |
| Genie Agents | Databricks Genie (AI/BI) |
| Dashboards | Databricks AI/BI Dashboards |
| Apps | Databricks Apps |
| Notifications | Databricks Notification Destinations (TBD which: Email, Teams/Slack, PagerDuty, Webhook) |
| Code Quality | SQLFluff (SQL) + Black/Ruff (Python) |

---

## 6. Enablement Program

| Track | Audience | Timeline |
|---|---|---|
| Databricks Fundamentals | All 48 beginners | Days 1-7 |
| JFrog Package Usage | All developers | Days 5-10 |
| Data Engineering (SDP, Auto Loader, DQ) | Data Engineers | Days 7-21 |
| UC & Governance (PHI masking, lineage) | Data Engineers + Analysts | Days 7-21 |
| DABs & CI/CD | Data Engineers | Weeks 3-4 |
| Genie/Dashboards/Apps | Data Engineers + Analysts | Weeks 3-5 |
| Near Real-Time (Streaming, CDC) | Selected engineers | Weeks 4-8 |
| GenAI/LLM & AI Gateway | Selected engineers | Weeks 4-8 |
| SAT & LakeWatch | Platform Eng + Data Engineers | Days 10-21 |
| Platform Engineering | Internal data engineer (shadowing OneDataAI) | Ongoing |

**Mechanisms:** Weekly office hours (2 experts) · PR-based code review · Template library · Internal wiki · Pair programming on first pipelines · Progressive access (sandbox -> clinical/operational after training) · Databricks Academy for self-paced learning.

---

## 7. Domain Priority by Phase

| Domain | Current (This Week) | Next |
|---|---|---|
| PlatformOps | Critical | Active |
| SecOps | Critical | Critical |
| DataOps | Foundational | Critical |
| DevOps | Foundational | Critical |
| FinOps | Visibility | Active |
| MLOps | TBD | TBD |
| AIOps | CRITICAL (LLM translation) | Critical |
| Data Product Ops | Foundational | Active |

---

## 8. Next Steps

### Day 1
- [ ] Onboard OneDataAI as platform engineer; identify internal data engineer for knowledge transfer
- [ ] Schedule RACI sign-off with AllCloud + OneDataAI + management
- [ ] Confirm dev/staging/prod workspace naming
- [ ] Audit EntraID groups -> map to Databricks groups
- [ ] Confirm JFrog server access + credentials path (coordinate with AllCloud)

### Days 1-7
- [ ] UC catalogs/schemas (clinical, operational, finance, sandbox)
- [ ] Cluster policies (min 3: small/medium/no-unrestricted)
- [ ] First DABs pipeline template (with JFrog consumption)
- [ ] Enable system tables (billing, audit)
- [ ] Cost dashboard (system tables + SAT)
- [ ] Deploy SAT (audit, cost, job monitoring)
- [ ] Coordinate AWS Secrets Manager scopes setup with AllCloud
- [ ] Integrate JFrog (init scripts, pip config, cluster policy refs)
- [ ] Deploy AI Gateway with multi-LLM routing for translation project
- [ ] Configure guardrails (PHI detection) for LLM translation
- [ ] Draft onboarding guide

### End of Current Phase (This Week)
- [ ] RACI signed (AllCloud + OneDataAI + management)
- [ ] UC structure live with grants documented
- [ ] Cluster policies enforced
- [ ] JFrog integrated + documented
- [ ] SAT + LakeWatch deployed
- [ ] AI Gateway live with LLM translation in production
- [ ] CI/CD pipeline operational (one DABs end-to-end)
- [ ] Cost dashboard live
- [ ] Genie/Dashboard/App governance patterns documented
- [ ] First training session completed
- [ ] Audit logging verified (PHI access tracking)
- [ ] Knowledge transfer in motion

### Ongoing
- [ ] Weekly cost review (Next phase)
- [ ] Bi-weekly platform health review
- [ ] Monthly access review (Next phase)
- [ ] Monthly blueprint review and update

---

## 9. TBD — To Be Defined

### Knowledge Transfer Plan
TBD — to be defined with OneDataAI and internal team.

### MLOps
TBD — to be defined when ML use cases materialize.

### Key Milestones per Domain per Phase
TBD — to be defined for each domain in each phase.

### Genie ZeroOps
Adopt Genie ZeroOps approach in next phases for automated Genie operations. Reference: https://www.databricks.com/blog/introducing-genie-zeroops

---

*v3.1 — Ops domains only, 2 phases, MLOps/KT/milestones as TBD, Genie ZeroOps added.*