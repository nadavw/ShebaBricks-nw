# Genie Agent Developer Guide

A practical, 9-step guide for building and maintaining Genie agents on Databricks.

---

## Step 1 — Define Subjects & Questions

Decide **what** the agent is for before touching any data.

- List the **business subjects** the agent should cover (e.g., hospitalizations, ER visits, length of stay, readmissions).
- For each subject, write 3–5 **example questions** a business user would ask in plain Hebrew (default language).
  - *"מה היו 10 המחלקות עם מספר האשפוזים הגבוה ביותר ברבעון האחרון?"*
  - *"אילו מחלקות היו בעלות אחוז חזרות לאשפוז (30 יום) הגבוה ביותר ב-2024?"*
- Identify the **audience** (department heads, medical staff, hospital management) and their expected vocabulary.
- Keep the scope **narrow and focused** — one Genie per domain beats one mega-Genie.

**Deliverable:** A one-page scope document with subjects, sample questions, and audience.

---

## Step 2 — Verify Data Readiness

Genie can only answer what the data supports — and only *well* when the data is self-describing.

- Confirm every **table** and **column** the agent needs has a **Unity Catalog comment**.
  ```sql
  ALTER TABLE catalog.schema.hospitalizations
    COMMENT 'תיעוד אשפוזים יומי לפי מחלקה וחולה.';
  
  ALTER TABLE catalog.schema.hospitalizations
    ALTER COLUMN length_of_stay COMMENT 'משך האשפוז בימים, מחושב כתאריך שחרור פחות תאריך קבלה.';
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
| *אשפוזים לפי מחלקה* | ✅ `hospitalizations` table | None |
| *אשפוזים לפי מחוז* | ⚠️ No `district` column | Need a mapping table or view |
| *סיכון לחזרה לאשפוז* | ❌ Not in data | Add a model output table or exclude |

- Note **synonyms** users might use (*"אשפוז"* vs *"קבלה"*, *"מחלקה"* vs *"מרפאה"*).
- Note **business logic** that isn't obvious from column names (e.g., *"חזרה לאשפוז"* = readmission within 30 days of discharge).
- Note **filters** users always apply (e.g., exclude transfer-only episodes, only current fiscal year).

**Deliverable:** A gap matrix listing every clarification Genie will need.

---

## Step 4 — Bridge the Gap

Fill the gaps from Step 3 with **instructions**, **skills**, and **examples**.


### Semantic View
If a gap requires combining multiple tables or embedding business logic, consider **building a semantic view** — a pre-joined, aggregated view with clear Hebrew column definitions and comments, so Genie can query it directly without complex multi-hop joins.

### Instructions (System Prompt)
Write clear, declarative rules Genie follows when generating SQL. **Instruction should be small, focused, global and organized.** for example:

> - "אשפוז" always means an episode in the `hospitalizations` table where `admission_type` is not 'ER observation'.
> - "חזרה לאשפוז" = readmission within 30 days of discharge.
> - Always exclude `is_transfer_only = true` records.
> - Fiscal year starts January 1.




### Examples (Q&A Pairs)
Add verified example questions with their expected SQL and a short explanation:

> **Q:** "5 המחלקות עם משך האשפוז הממוצע הארוך ביותר השנה"
> **SQL:**
> ```sql
> SELECT department_name, AVG(length_of_stay) AS avg_los
> FROM catalog.schema.hospitalizations
> WHERE YEAR(admission_date) = YEAR(CURRENT_DATE)
>   AND is_transfer_only = false
> GROUP BY department_name
> ORDER BY avg_los DESC
> LIMIT 5
> ```
> *מחשב ממוצע משך אשפוז לפי מחלקה, למעט העברות פנימיות, לשנה הנוכחית.*

Add **at least 5–10 examples** covering the most common question patterns.

### SQL Snippets - Filters, Measures, Fields and Joins
Provide reusable SQL fragments for non-obvious logic:

```sql
-- חישוב רבעון פיסקלי מתאריך קבלה
CASE
  WHEN MONTH(admission_date) IN (1,2,3)  THEN 'Q1'
  WHEN MONTH(admission_date) IN (4,5,6)  THEN 'Q2'
  WHEN MONTH(admission_date) IN (7,8,9) THEN 'Q3'
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
| 1 | סך אשפוזים Q1 2024 | 12,450 אשפוזים | שכחה לא לכלול העברות פנימיות |
| 2 | 3 המחלקות עם משך האשפוז הארוך ביותר | כירורגיה, פנימית, טיפול נמרץ | שימוש בתאריך קבלה במקום שחרור לחישוב משך |

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

## Step 7 — Stakeholder Validation

Before finalizing, ask a **business stakeholder** — someone with domain knowledge but no technical/SQL background — to test the agent in their own words. This step is **mandatory**.

- Have the stakeholder ask **5–10 questions** phrased the way they naturally would (not the benchmark wording).
- After each answer, have the stakeholder use **Genie's built-in rating** (👍 thumbs up / 👎 thumbs down) and add comments when rating down.
- Observe whether Genie understands the vocabulary and returns correct, useful answers.
- Capture **new synonyms, phrasings, or edge cases** the stakeholder reveals that benchmarks missed.
- If the stakeholder's questions surface gaps, loop back to Step 4 to add instructions, examples, or a semantic view.
- **Do not proceed to production** until the stakeholder confirms all answers are satisfactory (no unresolved 👎 ratings).

> **Rule of thumb:** If a business user can't get a correct answer in plain language, the agent isn't ready yet.

**Deliverable:** Stakeholder feedback summary with thumbs up/down ratings and any new instructions or examples added.

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
| 👎 (thumbs down) ratings | ↓ trending | Genie feedback history |
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
| 7 | Stakeholder validation | Business-user feedback + 👍/👎 ratings |
| 8 | Move to production | Agent live with access + announcement |
| 9 | Monitor | Weekly/monthly review cadence |