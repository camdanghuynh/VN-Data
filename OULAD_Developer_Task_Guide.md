# OULAD Pipeline — Developer Task Guide

**Dataset:** https://www.kaggle.com/datasets/rocki37/open-university-learning-analytics-dataset/data
**Reference analysis:** https://www.kaggle.com/code/devassaxd/student-performance-prediction-complete-analysis

**Goal:** build the full pipeline end to end (dlt → dbt → Superset → OpenMetadata → Dagster) using OULAD as practice data, before we connect to real sources.

**Environment:** VM1 `10.224.20.3` (Postgres) / VM2 `10.224.20.4` (devbox) / VM3 `10.224.20.5` (Superset + OpenMetadata)

> Throughout this guide, replace `{dev_user}` with your own identifier (assigned by the team lead, e.g. `triet`, `hieu`).

---

## Before you start

| Prerequisite | Where to find it |
|---|---|
| SSH access to VM2 | `SSH_Key_Setup_Guide.md` |
| Your personal env file `~/.dbt_env_<name>` | Provided by team lead |
| Daily workflow basics (code-server, `docker exec`, git) | `Daily_Dev_Workflow.md` |
| Full reference implementation with complete code | `Dataset_OULAD_Implement.md` |

> This document is the **task list** — what you need to do and why. When you need the actual code, refer to `Dataset_OULAD_Implement.md`, which has every model and script written out in full.

---

## Understand the data first (30 min, do not skip)

Before writing anything, get familiar with the 7 CSV files sitting in `dlt_pipelines/sample_data/oulad/`:

| File | What it holds |
|---|---|
| `courses.csv` | Course + presentation catalogue |
| `assessments.csv` | Assessment definitions per course |
| `vle.csv` | Learning resource catalogue |
| `studentInfo.csv` | Student demographics + final result |
| `studentRegistration.csv` | Registration / unregistration events |
| `studentAssessment.csv` | Individual assessment scores |
| `studentVle.csv` | Daily interaction logs (~10M rows, by far the largest) |

Everything links through `id_student` plus the `code_module` + `code_presentation` pair.

**One trap to be aware of:** the `date` columns are **not real dates** — they're *day offsets* relative to course start (`0` = start day, negative = before the course began). Keep them as integers and name them accordingly (`days_to_registration`, `day_offset`). Do not cast them to date/timestamp.

Have a look at the reference Kaggle notebook too — it's useful for deciding which metrics are worth building in the Gold layer.

---

## Step 1 — Set up your own working schemas

**Why:** all five of us share the same Postgres database. If we all write to the same schema, one person running a full reload wipes out what someone else is working on. So each dev gets their own isolated set of schemas.

**Your schemas:**

| Schema | Written by | Contains |
|---|---|---|
| `bronze_{dev_user}` | your dlt runs | Raw CSV data, untransformed |
| `dbt_{dev_user}_silver` | your dbt runs | Cleaned staging models |
| `dbt_{dev_user}_gold` | your dbt runs | Star schema for dashboards |

You don't need to create these manually — dlt and dbt create them automatically, driven by the `DBT_USER` variable in your env file. Just make sure you always pass `--env-file ~/.dbt_env_<yourname>` when running commands.

**Never write to** `bronze_shared` or `dbt_shared_dev_*` — those belong to Dagster and reflect what's on `main`.

**Verify before moving on:** confirm the 7 CSV files exist in `sample_data/oulad/` and check the header row of one or two of them.

---

## Step 2 — Load raw data with dlt

**What you're building:** a Python script that reads all 7 CSVs and loads them into your Bronze schema.

**Key design points:**

1. **Stream, don't load into memory** — use a generator (`yield`) rather than reading whole files. `studentVle.csv` has ~10M rows and will exhaust RAM otherwise.
2. **Make the target schema dynamic** — read `DBT_USER` from the environment and build the schema name from it (`bronze_{DEV_USER}`), so your run never touches anyone else's data.
3. **Make the pipeline name dynamic too** — dlt stores its own state keyed by pipeline name; if two devs share a name, their state collides.
4. **Use `write_disposition="replace"`** — you'll be reloading many times while developing, and you want a clean slate each run.
5. **Add a row-limit switch** — read something like `OULAD_ROW_LIMIT` from the environment so you can test with 1,000 rows in 30 seconds instead of waiting 20 minutes for the full load.

**Suggested code shape** (the full working version is in `Dataset_OULAD_Implement.md` §B.1):

```python
DEV_USER = os.environ.get("DBT_USER", "shared")
TARGET_SCHEMA = f"bronze_{DEV_USER}"

TABLES = {
    "courses": "courses.csv",
    "student_info": "studentInfo.csv",
    # ... map each table name to its CSV file
}

# One dlt resource per table, generated in a loop rather than
# writing seven near-identical functions by hand.
```

**Credentials:** dlt reads Postgres credentials from `.dlt/secrets.toml` at the repo root. Create it with the connection details for VM1, `chmod 600` it, and confirm `git status` doesn't show it.

**Run it twice:** first with the row limit set to 1000 to verify everything works, then without the limit for the full load.

**Done when:** you can query `bronze_{dev_user}.student_info` and get a row count back.

---

## Step 3 — Build the Silver layer with dbt

**What you're building:** 7 staging models, one per Bronze table. Cleaning only — no joins, no business logic, no aggregation.

**What "cleaning" means here:**
- Cast text columns to their proper types (`integer`, `numeric`, `bigint`)
- Rename columns to something readable (`code_module` → `course_code`)
- Handle empty strings before casting — `nullif(trim(col), '')` is your friend, several of these columns have `''` where you'd expect `NULL`
- Convert `Y`/`N` flags to real booleans
- Drop rows with a null primary key

**Key design points:**

1. **Point at your own Bronze schema** — declare the source schema as a dbt variable driven by `DBT_USER`, not a hardcoded string. Otherwise your models read someone else's data.
2. **Keep the grain identical to the source** — one row in, one row out. No joins at this layer.
3. **Comment the grain at the top of every model** — "1 row = 1 student per presentation". This saves a lot of debugging later.
4. **Materialise `stg_student_vle` as a table, not a view** — it's the only exception. It's 10M rows, and leaving it as a view means every downstream query rescans the whole thing.

**Done when:** `dbt run --select staging` completes and you see 7 objects in `dbt_{dev_user}_silver`.

---

## Step 4 — Build the Gold layer

**What you're building:** a star schema that Superset can query directly, without needing complex joins at dashboard time.

**Suggested models:**

| Model | Type | Grain | Purpose |
|---|---|---|---|
| `dim_student` | dimension | 1 student per presentation | Demographics + final result + registration timing |
| `dim_course` | dimension | 1 course presentation | Course metadata, year/term split out |
| `dim_assessment` | dimension | 1 assessment | Assessment type and weight banding |
| `fct_registration` | fact | 1 registration event | Enrolment and withdrawal analysis |
| `fct_assessment_result` | fact | 1 submission | Scores, lateness, weighted score |
| `agg_student_engagement` | aggregate | 1 student per presentation | Pre-aggregated VLE activity |

**Key design points:**

1. **Reference upstream models with `ref()`, never `source()`** — Gold reads from Silver, not from Bronze. This keeps the lineage clean and means a source change only needs fixing in one place.
2. **Pre-compute what dashboards will need** — banding (`engagement_level`, `registration_timing`, `result_group`), derived flags (`is_late`, `is_unregistered`), and simple arithmetic (`weighted_score`, `days_late`). Doing this once in dbt is far cheaper than doing it in every chart.
3. **`agg_student_engagement` is the important one** — it collapses 10M interaction rows down to one row per student. Without it, every dashboard filter triggers a full scan and Superset will time out.
4. **Watch the join keys** — student-level joins need all three of `student_id` + `course_code` + `presentation_code`. Joining on `student_id` alone will fan out your row counts.

**Sanity check:** `dim_student` will have *more* rows than there are students — that's expected, since a student taking three courses appears three times. If the number looks wildly wrong, check your join keys.

**Done when:** `dbt run --select marts` completes and a `GROUP BY result_group` query on `dim_student` returns sensible numbers.

---

## Step 5 — Add tests and documentation

**What you're building:** a `_marts.yml` file declaring what each model means and what must always be true about it.

**Tests worth adding:**
- `unique` + `not_null` on every dimension's primary key
- `accepted_values` on categorical columns you've derived yourself (`result_group`, `engagement_level`) — these catch logic bugs where a `CASE` statement falls through to an unexpected branch
- `relationships` between facts and dimensions — catches orphaned foreign keys

**Also add a `description` to every model and every non-obvious column.** These descriptions flow through to OpenMetadata later, so business users see them too. Write them for someone who doesn't know the dataset.

**Done when:** `dbt build` (run + test together) passes cleanly, and `dbt docs generate` produces `manifest.json` and `catalog.json` in `target/`.

---

## Step 6 — Build dashboards in Superset

Go to `http://10.224.20.5:8088`.

**Register your Gold tables as datasets:** Data → Datasets → + Dataset, pointing at `dbt_{dev_user}_gold`. Start with `dim_student`, `agg_student_engagement`, and `fct_assessment_result`.

**Suggested charts — each answers a real question:**

| Question | Chart | Dataset |
|---|---|---|
| How do pass/fail rates differ by course? | Bar, `course_code` x `result_group` | `dim_student` |
| Does more VLE activity mean better outcomes? | Bar, `engagement_level` x `result_group` | `agg_student_engagement` |
| Which assessment types are hardest? | Bar, `assessment_type` by `AVG(score)` | `fct_assessment_result` |
| Do late registrants drop out more? | Bar, `registration_timing` x `result_group` | `dim_student` |

Combine them into one dashboard and add filters for `course_code`, `presentation_code`, `gender`, and `age_band` so viewers can slice it themselves.

**Important last step:** export the dashboard and commit the export into `superset_assets/dashboards/`. Dashboards built only in the UI are invisible to version control and get lost.

> When you're building the *official* team dashboard rather than experimenting, point it at `dbt_shared_dev_gold` instead of your personal schema.

---

## Step 7 — Register metadata in OpenMetadata

Go to `http://10.224.20.5:8585`.

**Two separate ingestions are required** — this trips people up:

1. **Postgres metadata ingestion** — discovers tables, columns, types. Run it from the existing Postgres service under Settings → Services.
2. **dbt ingestion** — this is what actually builds the lineage graph. It reads the `manifest.json` and `catalog.json` that `dbt docs generate` produced in Step 5.

Running only the first gives you a table catalogue with no lineage, which is the most common mistake here.

**Note on file access:** OpenMetadata runs on VM3, but your dbt artifacts live on VM2. You'll need to copy the two JSON files across (or upload them through the UI).

**Done when:** opening `dim_student` in the catalogue and clicking the Lineage tab shows the full chain from Bronze through Silver to Gold.

**While you're there:** fill in descriptions and assign owners for the Gold tables. This is the layer business users will actually browse.

---

## Step 8 — Automate with Dagster

Go to `http://10.224.20.4:3000`.

**What you're adding:** a Dagster asset that runs the dlt pipeline, so the whole chain (extract → transform) runs automatically each night against the shared schema.

**Key design points:**

1. **Dagster runs as `shared`, not as you** — set `DBT_USER=shared` in the asset's environment so it writes to `bronze_shared` and `dbt_shared_dev_*`, leaving personal schemas untouched.
2. **Regenerate the dbt manifest after adding models** — Dagster reads `manifest.json` to discover which dbt models exist. New models won't appear until you re-run `dbt parse` and reload the code location.
3. **Always run the job manually once before enabling the schedule.** Never turn on automation for something you haven't watched succeed at least once.

**Done when:** the asset graph in the UI shows the full lineage, a manual run of `rebuild_shared_dev` succeeds, the schedule is toggled to Running, and `dbt_shared_dev_gold.dim_student` has data in it.

---

## Step 9 — Commit and open a PR

Before staging anything, run `git status` and confirm none of these appear:
- `.dlt/secrets.toml`
- `.env`
- `dbt_project/target/` or `dbt_project/logs/`

Then commit your dlt script, dbt models, Dagster asset, and the exported Superset dashboard. Open a PR and ask for a review.

Once it's merged, Dagster picks it up on the next nightly run and the shared schema reflects your work.

---

## Checklist

**dlt**
- [ ] Pipeline runs with a row limit, then runs fully
- [ ] 7 tables present in `bronze_{dev_user}`
- [ ] `secrets.toml` not tracked by git

**dbt**
- [ ] 7 staging models built
- [ ] 6 mart models built
- [ ] All tests pass
- [ ] `manifest.json` and `catalog.json` generated

**Superset**
- [ ] Datasets registered, 4 charts built
- [ ] Dashboard assembled with filters
- [ ] Dashboard exported and committed

**OpenMetadata**
- [ ] Both ingestions run successfully
- [ ] Lineage visible end to end
- [ ] Gold tables documented and owned

**Dagster**
- [ ] Asset appears in the graph
- [ ] Manual run succeeds
- [ ] Schedule enabled
- [ ] Shared schema populated

**Git**
- [ ] No credentials committed
- [ ] PR opened and reviewed

---

## When something breaks

Work outward-in, in this order — it will save you a lot of time:

1. **Network** — can you reach the target at all? (`nc -zv 10.224.20.3 5432`)
2. **Auth** — can you connect with plain `psql`?
3. **Tool** — does the failure only happen inside dbt/dlt?

`Dataset_OULAD_Implement.md` has a troubleshooting table covering the failures we've already hit, including the ones that look confusing (`cannot cast type text to integer`, slow `agg_student_engagement`, empty lineage in OpenMetadata).

Ask in the channel if you're stuck for more than 30 minutes — chances are someone has already run into it.
