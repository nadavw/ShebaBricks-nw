# ShebaBricks-nw

Databricks data platform project for Sheba Medical Center (AWS). This repo contains architecture decisions, engineering standards, and operational blueprints.

## Repository Structure

| Path | Description |
|---|---|
| `operability-blueprint.md` | Operability blueprint — 8 Ops domains (PlatformOps, SecOps, DataOps, DevOps, FinOps, MLOps, AIOps, Data Product Ops), RACI model, phased roadmap, tool stack |
| `ShebaBricks_Ingestion_Framework_ADR.md` | ADR: multi-domain ingestion framework using Lakeflow Connect + SDP + DABs |
| `ShebaBricks - Silver & Gold Engineering Standard ADR.md` | ADR: organization-wide standard for Silver and Gold layer data products |
| `Genie_Agents_Developers_Guide.md` | 9-step practical guide for building Genie Agents (with Hebrew medical examples) |
| `Infrastructure/ShebaBricks_Data_Catalog_Structure_ADR.md` | ADR: Unity Catalog structure (catalogs, schemas, domains) |
| `SSIS_migration/ShebaBricks_SSIS_Migration_Plan.md` | SSIS-to-Databricks migration plan (assessment, complexity levels) |
| `order_ingest_dab/` | DAB project for order ingestion pipeline |
| `skills/` | Databricks Assistant skills directory |

## Partners

- **AllCloud** — Infrastructure (AWS, workspace deployment, networking, PrivateLink)
- **OneDataAI** — Platform engineering + knowledge transfer to internal team

## Environments

Dev / Staging / Prod workspaces on AWS with PrivateLink (no public internet access).