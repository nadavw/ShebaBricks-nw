# Genie Agent Developer Guide

A practical, 7-step guide for building and maintaining Genie agents on Databricks.

---

## Step 1 — Define Subjects & Questions

Decide **what** the agent is for before touching any data.

- List the **business subjects** the agent should cover (e.g., revenue, inventory, churn).
- For each subject, write 3–5 **example questions** a business user would ask in plain language.
  - *"What were our top 10 products by revenue last quarter?"*
  - *"Which regions had the highest return rate in 2024?"*
- Identify the **audience** (executives, analysts, ops) and their expected vocabulary.
- Keep the scope **narrow and focused** — one Genie per domain beats one mega-Genie.

**Deliverable:** A one-page scope document with subjects, sample questions, and audience.

---

## Step 2 — Verify Data Readiness

Genie can only answer what the data supports — and only *well* when the data is self-describing.

- Confirm every **table** and **column** the agent needs has a **Unity Catalog comment**.
  ```sql
  ALTER TABLE catalog.schema.sales
    COMMENT 'Daily sales transactions by product and store.';
  
  ALTER TABLE catalog.schema.sales
    ALTER COLUMN revenue COMMENT 'Net revenue in USD after discounts and returns.';
  ```
- Check that column names are **human-readable** — rename cryptic names where possible.
- Verify the data is **fresh, complete, and accurate** for the questions in scope.
- Genie agents are **only allowed to read tables from the Silver and Gold layers** — never expose Bronze or raw source tables.
- All Unity Catalog **table and column comments must be written in Hebrew** to match the audience.
- Only **add the tables and columns needed for the domain** to prevent confusing Genie

**Deliverable:** A checklist confirming all in-scope tables/columns have comments and the data is current.

---

## Step 3 — Map the Gap

Compare Step 1 (questions) against Step 2 (data). Find where Genie will need help.

| Question | Data available? | Gap / Clarification needed |
|---|---|---|
| *Top products by revenue* | ✅ `sales` table | None |
| *Revenue by region* | ⚠️ No `region` column | Need a mapping table or view |
| *Churn risk score* | ❌ Not in data | Add a model output table or exclude |

- Note **synonyms** users might use (*"turnover"* vs *"revenue"*, *"SKU"* vs *"product_id"*).
- Note **business logic** that isn't obvious from column names (e.g., *"active customer"* = last purchase within 90 days).
- Note **filters** users always apply (e.g., exclude test accounts, only current fiscal year).

**Deliverable:** A gap matrix listing every clarification Genie will need.

---

## Step 4 — Bridge the Gap

Fill the gaps from Step 3 with **instructions**, **skills**, and **examples**.

### Instructions (System Prompt)
Write clear, declarative rules Genie follows when generating SQL:

> - "Revenue" always means the `revenue` column in `sales`, already net of returns.
> - "Active customer" = at least one purchase in the last 90 days.
> - Always exclude `is_test_account = true` records.
> - Fiscal year starts February 1.

### Semantic View
If a gap requires combining multiple tables or embedding business logic, consider **building a semantic view** — a pre-joined, aggregated view with clear Hebrew column definitions and comments, so Genie can query it directly without complex multi-hop joins.



### Examples (Q&A Pairs)
Add verified example questions with their expected SQL and a short explanation:

> **Q:** "Top 5 customers by revenue this year"
> **SQL:**
> ```sql
> SELECT customer_name, SUM(revenue) AS total
> FROM catalog.schema.sales
> WHERE YEAR(transaction_date) = YEAR(CURRENT_DATE)
>   AND is_test_account = false
> GROUP BY customer_name
> ORDER BY total DESC
> LIMIT 5
> ```
> *Groups by customer, sums net revenue, excludes test accounts, current calendar year.*

Add **at least 5–10 examples** covering the most common question patterns.

### SQL Snippets - Filters, Measures, Fields and Joins
Provide reusable SQL fragments for non-obvious logic:

```sql
-- Fiscal quarter from date column
CASE
  WHEN MONTH(date_col) IN (2,3,4)  THEN 'Q1'
  WHEN MONTH(date_col) IN (5,6,7)  THEN 'Q2'
  WHEN MONTH(date_col) IN (8,9,10) THEN 'Q3'
  ELSE 'Q4'
END
```
Prefer a **Semantic View** over SQL snippets.

**Deliverable:** Populated Instructions, Skills, and Examples sections in the Genie configuration.

---

## Step 5 — Add Benchmarks

Define **expected correct answers** for a set of representative questions so you can measure quality over time.

- Choose **10–20 benchmark questions** spanning all subjects and difficulty levels.
- For each, record:
  - The **expected SQL** or a description of the correct result.
  - The **expected answer shape** (columns, row count, sort order).
  - Any **common mistakes** to watch for.
- Store these as a simple checklist or spreadsheet.

| # | Question | Expected result | Common mistake |
|---|---|---|---|
| 1 | Total revenue Q1 2024 | $4.2M (net) | Forgetting to exclude test accounts |
| 2 | Top 3 regions by orders | West, Central, East | Using gross instead of net revenue |

**Deliverable:** A benchmark set with ≥ 10 verified questions and expected answers.

---

## Step 6 — Test & Iterate

Run the benchmarks and fix what breaks.

1. **Ask** each benchmark question to the Genie agent (fresh conversation each time).
2. **Compare** the generated SQL and results against your expected answers.
3. For each failure, determine the root cause. For example:
   - **Missing data** → go back to Step 2.
   - **Ambiguous column** → improve the column comment.
   - **Missing business logic** → add or refine an instruction.
   - **Complex pattern** → add a skill or example.
   - **Synonym not recognized** → add to instructions.
4. **Update** instructions, skills, examples, or comments — then re-test.
5. Repeat until all benchmarks pass.

> **Rule of thumb:** If Genie gets it wrong twice, don't tweak the prompt — add an example.

**Deliverable:** All benchmark questions passing; change log of what was fixed.

---

## Step 7 — Stakeholder Validation (Optional)

Before finalizing, ask a **business stakeholder** — someone with domain knowledge but no technical/SQL background — to test the agent in their own words.

- Have the stakeholder ask **5–10 questions** phrased the way they naturally would (not the benchmark wording).
- Observe whether Genie understands the vocabulary and returns correct, useful answers.
- Capture **new synonyms, phrasings, or edge cases** the stakeholder reveals that benchmarks missed.
- If the stakeholder's questions surface gaps, loop back to Step 4 to add instructions, examples, or a semantic view.

> **Rule of thumb:** If a business user can't get a correct answer in plain language, the agent isn't ready yet.

**Deliverable:** Stakeholder feedback summary with any new instructions or examples added.

---

## Step 8 — Move to Production

Once benchmarks pass and stakeholders are satisfied, promote the Genie agent to production.

- **Final review** — confirm all instructions, examples, skills, and semantic views are clean and up to date.
- **Permissions** — grant access to the intended audience groups; verify no over-exposure of tables or columns.
- **Announce** — notify users that the agent is live, including a short guide on what it covers and how to phrase questions.
- **Request Feedback** - ask end users to give feedback (Thumb up\down, add comments)
- **Archive artifacts** — save the scope document, gap matrix, benchmark set, and stakeholder feedback in a shared location for future reference.
- **Schedule the first monitoring review** (Step 9) for one week after go-live to catch early issues quickly.


**Deliverable:** Agent live in production with documented access, announcement, and a scheduled first review.

---

## Step 9 — Monitor

Genie agents drift as data and usage evolve. Set up an ongoing review cadence.

- **Weekly:** Review the Genie conversation history for failed or low-quality answers.
- **Monthly:** Re-run the full benchmark suite; fix regressions.
- **Quarterly:** Re-evaluate scope — are there new subjects or questions users are asking that aren't covered?
- **Ongoing triggers for review:**
  - New tables or columns added to the underlying data.
  - Business logic changes (new fiscal calendar, new exclusion rules).
  - Users reporting wrong answers.

**Quick health metrics:**

| Metric | Target | Source |
|---|---|---|
| Benchmark pass rate | ≥ 90% | Re-run Step 5 |
| Failed/questioned conversations | ↓ trending | Genie history |
| New example questions added | Monthly | Gap analysis |

**Deliverable:** A monitoring cadence and a living document tracking quality over time.

---

## Summary Checklist

| Step | What | Key Output |
|---|---|---|
| 1 | Define scope | Subjects + sample questions |
| 2 | Verify data | Comments on all tables/columns |
| 3 | Map gaps | Gap matrix |
| 4 | Bridge gaps | Instructions + skills + examples |
| 5 | Benchmark | 10–20 verified Q&A pairs |
| 6 | Test & iterate | All benchmarks passing |
| 7 | Stakeholder validation *(optional)* | Business-user feedback + refinements |
| 8 | Move to production | Agent live with access + announcement |
| 9 | Monitor | Weekly/monthly review cadence |