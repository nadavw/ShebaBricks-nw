# ShebaBricks SSIS Migration Plan

## Phase 1 - Assessment
Collect data about SSIS and SQL assets and their complexity levels

**SSIS complexity levels**
- Level 1 - simple SSIS used to schedule stored procedures (example: dim patiences)
- Level 2 - SSIS packages that include use of SSIS components (i.e. merge, derived columns, converting)
- Level 3 - SSIS packages with script components

###Tasks
- Find a tool to list all SSIS packages in a projects to create an Excel file \ table with
- Find a tool to assess complexity levels (lakebridge?)
- Extract stored procdues and views to sql files

##Phase 2 - Build tools for migration
- **QA library** - compare DWH tables from existing SQL to silver tables automatically and return results. 
######Relevent tests:
1. compare counts
2. compare primary key columns
3. compare rows hash
4. compare actual columns and return where is the difference

- **Genie skill** - Genie skill to extract relevent parts from SSIS packages and stored procedures\views
1. ignore unrelent component - logging, handlers, environment parameters (will be done in Databricks automatically)
2. Aware of disired destination - SDP pipelines with SQL files, catalog and schema, naming conventions

##Phase 3 - Migration
For each model/domain/SSIS project:
1. Upload to Genie - packages + sql files
2. Genie convert to a silver SDP
3. After migration QA tests (using automated tools)

##phase 4 - Refactoring 
- en appropriate refactor to improve usage or performance
## Phase 5 - Gold layer
- When appropriate build gold tables