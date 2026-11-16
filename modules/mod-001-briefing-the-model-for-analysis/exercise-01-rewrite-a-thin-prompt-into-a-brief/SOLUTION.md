# Exercise 01 — Rewrite a Thin Prompt into a Brief · Reference Solution

Pairs with [`exercise-01-rewrite-a-thin-prompt-into-a-brief.md`](https://github.com/ai-workforce-curriculum/ai-augmented-analyst-learning/blob/main/lessons/mod-001-briefing-the-model-for-analysis/exercises/exercise-01-rewrite-a-thin-prompt-into-a-brief.md) and the [module README](https://github.com/ai-workforce-curriculum/ai-augmented-analyst-learning/blob/main/lessons/mod-001-briefing-the-model-for-analysis/README.md).

## 1. Solution overview

This is a design-and-reflection exercise, not a code exercise. The learner's deliverable is four artifacts: (a) the stakeholder ask captured verbatim, (b) a thin-prompt output, (c) a written brief that replaces every ambiguous word in the ask with a decision, (d) a briefed output plus a diff note. A reference "answer" is one fully worked instance of that pipeline — below — plus a rubric for grading any instance the learner submits.

The grading target is whether the brief **changed the query**, not whether it polished the prose. A brief that leaves the SQL identical has done nothing. A brief that produces a different number but can't name the clause responsible for the difference is only half-done.

## 2. Implementation

The worked brief in the module README uses a hypothetical `fct_feature_events` warehouse. This reference solution deliberately works a *different* domain — the public NYC TLC Yellow Taxi trip records, queried with DuckDB — so the learner can compare the shape of the brief rather than cargo-cult the specifics.

### 2.1 Stakeholder ask, verbatim

> **PM, in Slack, 2026-11-04 09:12 ET:** "hey can you pull how many rides we had last month? also would love to know how it's trending 🙏"

Context the ask does not include but the analyst knows: PM is on the Yellow Taxi product surface; they ignore Green / FHV / FHVHV; "trending" has historically meant intra-month daily pattern in their past asks.

### 2.2 Pass 1 — thin prompt

**Prompt (handed to the model unchanged):**

> hey can you pull how many rides we had last month? also would love to know how it's trending 🙏

**Representative output** (the exact SQL varies by model and run; capture what you actually get):

```sql
SELECT COUNT(*) AS rides_last_month
FROM trips
WHERE pickup_date >= DATE_TRUNC('month', CURRENT_DATE) - INTERVAL '1 month'
  AND pickup_date <  DATE_TRUNC('month', CURRENT_DATE);
```

Narrative the model offered alongside: *"You had approximately N rides last month, trending upward compared to the prior month."*

What the model silently decided for you:

- There is a table called `trips` with a `pickup_date` column.
- "Last month" is the calendar month immediately before today, evaluated at query time.
- "Rides" is every row.
- Dialect is Postgres-flavored (`DATE_TRUNC`, `INTERVAL '1 month'`).
- "Trending" is one month-over-month % figure, not a time series.
- No filters for zero-fare, voided, or duplicate trips.

Every bullet is a defensible guess. None of them was ratified by you.

### 2.3 The brief

> **Runtime:** DuckDB ≥ 0.10 reading NYC TLC Parquet drops locally via `read_parquet()`. No warehouse.
>
> **File:** `yellow_tripdata_2026-10.parquet` from the [NYC TLC trip record data page](https://www.nyc.gov/site/tlc/about/tlc-trip-record-data.page). Published naming convention is `yellow_tripdata_YYYY-MM.parquet`. If the file has not yet been posted for the target month, stop and tell the PM — do not fall back to the prior month silently.
>
> **Grain:** one row per completed Yellow Taxi trip. TLC does not publish a stable primary key; for deduplication use `(VendorID, tpep_pickup_datetime, tpep_dropoff_datetime, PULocationID, DOLocationID)` as a surrogate key.
>
> **Columns we rely on:** `tpep_pickup_datetime` (trip start, NYC local, naive timestamp — no timezone attached), `tpep_dropoff_datetime`, `VendorID`, `passenger_count`, `trip_distance`, `fare_amount`, `total_amount`, `PULocationID`, `DOLocationID`.
>
> **Question, bounded:** Count Yellow Taxi trips where `tpep_pickup_datetime` falls in **October 2026 (NYC local, i.e., the whole month as it appears in the Parquet)**, with `fare_amount > 0`, `trip_distance > 0`, and `passenger_count IS NOT NULL AND passenger_count > 0`. One row per trip after surrogate-key dedup. "Last month" = October 2026 because the ask was made on 2026-11-04.
>
> **Shape of the answer, part A:** a single integer (`rides`).
>
> **Shape of the answer, part B — "trending":** a two-column table `(pickup_date, rides)` at **daily** grain over the same month, same filters. No chart. PM will paste the top row counts into a Slack thread.
>
> **Dialect notes:** DuckDB SQL. Use `TIMESTAMP` literals; TLC timestamps are already in NYC local time with no timezone, so no `AT TIME ZONE` gymnastics. `read_parquet()` globs are available if the file is split.
>
> **Landing:** internal product Slack message. Error budget is ±1%. The number will not be cited externally. If it were a board slide, we would also want (i) a reconciliation against the TLC's own monthly rollups and (ii) an explicit note on the surrogate-key dedup assumption.
>
> **Ruled out:**
> - Green Taxi / FHV / FHVHV Parquet files — the PM's "we" means Yellow medallion, confirmed from prior asks.
> - `total_amount > 0` as the "completed trip" filter — it includes tolls and tips and can legitimately be `$0` for fare disputes. `fare_amount > 0` is the better liveness signal.
> - Any `JOIN` to the TLC taxi-zone lookup — the ask is a count, not a geography breakdown.
> - Month-over-month comparison — the PM's "trending" historically means *intra*-month shape, not a single MoM %.

### 2.4 Pass 2 — briefed prompt

Hand the brief above to the model in a **new session** (don't let Pass 1 context bleed in). Representative output:

```sql
-- DuckDB
WITH raw AS (
  SELECT *
  FROM read_parquet('yellow_tripdata_2026-10.parquet')
  WHERE tpep_pickup_datetime >= TIMESTAMP '2026-10-01 00:00:00'
    AND tpep_pickup_datetime <  TIMESTAMP '2026-11-01 00:00:00'
    AND fare_amount    > 0
    AND trip_distance  > 0
    AND passenger_count IS NOT NULL
    AND passenger_count > 0
),
deduped AS (
  SELECT DISTINCT
         VendorID,
         tpep_pickup_datetime,
         tpep_dropoff_datetime,
         PULocationID,
         DOLocationID
  FROM raw
)
SELECT COUNT(*) AS rides
FROM deduped;
```

```sql
-- Daily "trending" series, same filters, same dedup
WITH raw AS (
  SELECT *
  FROM read_parquet('yellow_tripdata_2026-10.parquet')
  WHERE tpep_pickup_datetime >= TIMESTAMP '2026-10-01 00:00:00'
    AND tpep_pickup_datetime <  TIMESTAMP '2026-11-01 00:00:00'
    AND fare_amount    > 0
    AND trip_distance  > 0
    AND passenger_count IS NOT NULL
    AND passenger_count > 0
),
deduped AS (
  SELECT DISTINCT
         VendorID,
         tpep_pickup_datetime,
         tpep_dropoff_datetime,
         PULocationID,
         DOLocationID
  FROM raw
)
SELECT CAST(tpep_pickup_datetime AS DATE) AS pickup_date,
       COUNT(*) AS rides
FROM deduped
GROUP BY 1
ORDER BY 1;
```

### 2.5 Diff note

| Dimension | Thin pass | Briefed pass | Load-bearing? |
|---|---|---|---|
| Source | `trips` (guessed table) | Specific Parquet file, named pattern | Yes — the thin query would not execute |
| Time window | Today-relative (`CURRENT_DATE - 1 month`) | Pinned to literal `TIMESTAMP '2026-10-01'` – `'2026-11-01'` | Yes — rerunning the thin prompt a month later silently shifts the answer |
| Row filters | None | `fare_amount > 0`, `trip_distance > 0`, positive non-null `passenger_count` | Yes — zero-fare and zero-distance rows exist in the TLC drops and would inflate the count if unfiltered |
| Dedup | `COUNT(*)` on raw rows | `SELECT DISTINCT` on a 5-column surrogate key before counting | Yes — TLC publishes no PK; the thin query would double-count any republished rows |
| Dialect | Postgres (`DATE_TRUNC`, `INTERVAL`) | DuckDB (`read_parquet()`, timestamp literals) | Yes — thin SQL does not run as-is on DuckDB |
| "Trending" | One MoM % in prose | Daily series within the month | Yes — the PM wanted the intra-month shape, which an MoM % hides entirely |

**Winner, honestly called:** briefed pass. The thin query will not run, and even if you hand-edited it to run, it would over-count (no filters, no dedup) and answer a different second question (MoM % instead of a daily series).

**If the numbers had matched:** that would have been luck, not signal — the thin query happened to guess the right table name and the public drops happened to be low on zero-fare rows that month. The analyst should still ship the briefed version because the thin query doesn't survive next month's rerun.

### 2.6 Reflection answers (worked)

- **Which of the six items did you forget on your first draft?** Shape. My first pass had grain, question, dialect, landing, and ruled-out items but asked for "a count"; I didn't notice "trending" was a *second* question with its own shape until I wrote the brief and the two asks stopped collapsing onto each other. The load-bearing one for the analysis I usually do is grain — I have been burned more often by `COUNT(*)` on the wrong row-type than by anything else.
- **Where did the briefed output still need human verification?** The dedup step. The brief declared a surrogate key, but the actual TLC file has no documented guarantee that the five chosen columns are unique together. Before landing the number I would run a quick `GROUP BY ... HAVING COUNT(*) > 1` to size the duplicate rate; if it's non-zero the brief needs a tiebreaker column.
- **Hallway redirect to a briefable question:** *"Before I dig — do you want October specifically, or trailing 30 days? And is this for the Slack update or the business review deck?"* Two questions, 20 seconds, and the brief writes itself from the answers.

## 3. Validation steps

A learner (or the grader) can check their own submission with:

1. **Verbatim check.** Does the submitted ask read like a Slack message or like a tidied-up specification? If it's been cleaned, the exercise has started in the wrong place; re-capture.
2. **Six-item coverage.** Do the brief's sections correspond, 1-to-1, to: schema and grain · bounded question · dialect and runtime · shape · ruled-out · landing? Missing any item is an automatic re-draft.
3. **Ambiguity sweep.** Highlight every ambiguous word in the original ask (`last month`, `rides`, `we`, `trending`, pronouns, plurals, time expressions). For each, point to the sentence in the brief that pins it down. Any un-pinned ambiguity is a gap.
4. **Dialect reality check.** Does the briefed SQL contain functions that exist in the named dialect? A brief that says "BigQuery" and an output that uses `QUALIFY` on Postgres is a brief failure, not a syntax quibble.
5. **Diff diagnostic.** Open both outputs side by side. If you cannot name at least three *clause-level* differences, either the brief did not change the query or the diff note is under-written. Both are findings.
6. **Runnability.** If model access was available, both outputs should have been executed. The thin output is allowed to fail — note *why* it failed (missing table, wrong dialect, etc.) rather than quietly fixing it.
7. **"Lucky thin" case.** If the two queries produced the same number, is the submission explicit about whether that was luck or signal? Silent on this = re-draft.

## 4. Rubric

| Area | 0 (missing) | 1 (partial) | 2 (solid) | 3 (excellent) |
|---|---|---|---|---|
| **Verbatim ask captured** | No ask recorded | Paraphrased or cleaned | Captured as said | Captured with channel, who, when |
| **Brief covers six items** | ≤ 2 items | 3–4 items | 5 items present | All 6; the three load-bearing items (grain, bounded question, landing) are the most specific |
| **Ambiguity pinned to decisions** | No ambiguous word replaced | A couple replaced | Most replaced with a decision | Every ambiguity replaced *and* each decision defended briefly |
| **Two model outputs saved** | One or none | Both but prompts not kept | Both + exact prompts | Both + prompts + environment (model, date, tools, session hygiene) |
| **Diff names clauses** | Adjectives only ("cleaner") | Some differences named | Specific clauses/columns | Clauses + a stakeholder-facing consequence per difference |
| **Honest winner call** | No call made | Picked without reasoning | Picked with reasoning | Picked with reasoning, including the "lucky thin prompt" case if applicable |
| **Reflection** | Not attempted | One-line stubs | All three answered | All three answered with a concrete handoff to the verification module |

**Pass bar:** 2 or better in every row, and 3 in at least three rows including **Diff names clauses**.

## 5. Common mistakes

- **Prose edit masquerading as a brief.** The brief is a decision document, not an English rewrite of the ask. If `last month` or `users` still appears without a definition, nothing has moved.
- **Section-header filler.** Pasting "Schema and grain: our main table." looks like a brief and isn't one. Name the table, the grain, and the dedup key.
- **Tone-based diff.** "The briefed one sounded more confident" is not a diff. If no clause changed, admit it.
- **Chat-history bleed.** Running Pass 2 in the same model session as Pass 1 contaminates it — the model will quietly use context you didn't put in the brief. Open a clean session (or clear history) between passes. The module calls this out as the "silent context" anti-pattern.
- **Hand-tuning the thin output.** Editing Pass 1's SQL to make the comparison "fair" measures your editing, not the brief. Save the raw output; note the failures.
- **Over-briefing throwaway asks.** A brief for a 5-minute exploration is overhead. The signal that briefing is worth the time is in the module: "the number is going somewhere you can't take back." No landing like that → one-liner is fine.
- **Ignoring the secondary ask.** Stakeholders routinely pile a second question onto the first (`"also… trending?"`). Either brief each sub-question's shape separately, or push back that there are two briefs here.
- **Schema dump.** Pasting the whole DDL "so the model can pick". The module's "dump" anti-pattern: it usually finds a wrong table that *sounds* right. Name the specific tables and columns the brief authorizes.
- **Forgetting the dedup when the source has no PK.** Public drops and append-only event tables routinely republish rows; `COUNT(*)` without a dedup is a landmine, not a count.

## 6. References

Primary — grading hangs on these:

- Module 001 README — the six-item brief checklist, worked brief, and anti-patterns the rubric encodes: [`lessons/mod-001-briefing-the-model-for-analysis/README.md`](https://github.com/ai-workforce-curriculum/ai-augmented-analyst-learning/blob/main/lessons/mod-001-briefing-the-model-for-analysis/README.md)
- Exercise 01 spec: [`lessons/mod-001-briefing-the-model-for-analysis/exercises/exercise-01-rewrite-a-thin-prompt-into-a-brief.md`](https://github.com/ai-workforce-curriculum/ai-augmented-analyst-learning/blob/main/lessons/mod-001-briefing-the-model-for-analysis/exercises/exercise-01-rewrite-a-thin-prompt-into-a-brief.md)

Worked-example data source:

- NYC Taxi & Limousine Commission — Trip Record Data (public, monthly Parquet drops, documented column dictionary): https://www.nyc.gov/site/tlc/about/tlc-trip-record-data.page

Peripheral context (not needed to pass the exercise; listed because the module sits inside a wider governance curriculum):

- NIST AI Risk Management Framework — the "where the number lands" step in a brief maps to NIST's MAP function around deployment context: https://www.nist.gov/itl/ai-risk-management-framework
- OWASP Machine Learning Security Top 10 — ambient framing for why ambiguous input is a risk surface, not a prompting manual: https://owasp.org/www-project-machine-learning-security-top-10/
- MITRE ATLAS — adversarial-ML context; not directly applied in this exercise: https://atlas.mitre.org/
