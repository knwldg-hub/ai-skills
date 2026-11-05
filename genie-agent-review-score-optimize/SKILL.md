---
name: genie-agent-review-score-optimize
description: Review, score and optimize an existing Genie Agent (formerly Genie Space). Use when the user asks to review, audit, score, IQ-score, health-check, benchmark, tune, improve the accuracy of, or optimize a Genie Agent or Genie Space. Runs a guided loop - back up config, baseline IQ score plus deep review plus benchmarks, prioritized recommendations, implement approved fixes, then re-score with a before/after comparison.
---

# Genie Agent Review, Score & Optimize

Version 0.1 (pilot). Built for Activision Global Analytics on Databricks on Google Cloud (us-west4).

You are guiding the user through an end-to-end review of one Genie Agent. Work in a notebook as the current user. Go one step at a time. At every ⏸ checkpoint, stop and wait for the user's answer.

## Golden rules (always follow)

1. **No change without approval.** Show the exact change (diff or SQL) and get an explicit "yes" for each change before applying it.
2. **Back up first, verify after.** Save the agent's full config before the first change. After every change, re-fetch the config and check nothing else was lost. If anything was removed unexpectedly, restore the backup immediately and tell the user.
3. **No data values in output.** The data may contain PII. Never print result rows, sample values or example values. Show only counts, column names, data types, SQL text and pass/fail.
4. **Show code before running it**, and run it in notebook cells.
5. **Don't guess.** If an API call fails or a field is missing, explain what happened and offer a manual alternative.
6. **Use GCP documentation links** (docs.databricks.com/gcp/...).
7. **Preserve IDs.** When editing the config, keep existing item IDs and all fields you aren't changing. New items get a new 32-character lowercase hex ID. Copy the structure of existing entries of the same type. If unsure about the config schema, use the built-in genie-space-authoring knowledge.

## Workflow

Step 0 Setup → 1 Pick agent → 2 Apply mode → 3 Backup → 4 Baseline score → 5 Agent-mode benchmarks (optional, manual) → 6 Recommendations → 7 Draft change → 8 Apply → 9 Re-score → 10 Next or finish

### Step 0: Setup

Make sure a notebook is attached to serverless compute or a cluster, then run:

```python
%pip install genie-iq-score-lite --quiet
```

If the install fails (for example, no access to PyPI), continue anyway. Use the built-in rubric in Appendix A and label results "built-in rubric (approximation of the Genie Workbench IQ Scan)".

```python
import json, datetime, os, hashlib
from databricks.sdk import WorkspaceClient
w = WorkspaceClient()
ME = w.current_user.me().user_name
REVIEW_ROOT = f"/Workspace/Users/{ME}/genie_agent_reviews"
```

### Step 1: Pick the agent ⏸

Ask: *"Which Genie Agent should I review? Paste the agent ID (the 32-character ID in the agent's URL), or tell me its name and I'll look it up."*

If they give a name, list the agents they can access (`w.genie.list_spaces()`) and match on title.

Fetch the config:

```python
SPACE_ID = "<agent-id>"
space = w.genie.get_space(SPACE_ID, include_serialized_space=True)
cfg = json.loads(space.serialized_space)
```

Show a summary with counts only: title, description length, number of tables and metric views, text-instruction character count, example SQL count, SQL functions, join specs, SQL snippets (filters, expressions, measures), benchmark question count, and warehouse ID.

Ask: *"Is this the right agent?"*

### Step 2: Choose how changes are applied ⏸

Ask: *"How should I apply approved changes?"*

- **Mode A: edit the agent directly.** Best for agents managed in the UI.
- **Mode B: produce PR-ready config.** Best for agents deployed through CI/CD (DABs, a pipeline, or a deployment service principal). A UI edit would be overwritten by the next deploy. I'll write the updated config file and a change summary for your repo, then re-score after you deploy.
- **Mode C: test on a copy.** I clone the agent (same tables, instructions and benchmarks) as "[REVIEW COPY] <title>" and change only the copy. Good for trying the skill out.

If the agent is owned by a service principal, or the user says it's deployed by CI/CD, recommend Mode B. The user decides. Record the choice. The user can switch modes for any single recommendation.

For Mode C, create the copy:

```python
copy = w.genie.create_space(
    warehouse_id=space.warehouse_id,
    serialized_space=space.serialized_space,
    title=f"[REVIEW COPY] {space.title}",
    description=space.description,
)
```

Continue with the copy's ID. Tell the user the copy's ID, and that they can trash it when done.

### Step 3: Backup

```python
TS = datetime.datetime.utcnow().strftime("%Y%m%dT%H%M%SZ")
DIR = f"{REVIEW_ROOT}/{SPACE_ID}"
os.makedirs(DIR, exist_ok=True)
with open(f"{DIR}/backup_{TS}.json", "w") as f:
    json.dump({"title": space.title, "description": space.description,
               "warehouse_id": space.warehouse_id, "serialized_space": cfg}, f, indent=2)
```

Tell the user where the backup is saved.

### Step 4: Baseline score

Run all three parts, then show one scorecard.

**4a. IQ score (12 checks)**

If `genie-iq-score-lite` is installed:

```python
from genie_iq_score_lite import calculate_score
iq = calculate_score(cfg)
```

Otherwise, score each check in Appendix A yourself. For the description checks (2 and 3), get table and column comments from Unity Catalog with `w.tables.get(full_name)`. Count only; never read data. Agent-level column descriptions in the config also count.

**4b. Deep review (Appendix B)**

Evaluate checks D1 to D15. For each, give a status (✅ pass, ⚠️ warn, ❌ fail) and evidence. Evidence can quote config text or SQL, but never data values.

For D3, validate each example SQL without returning data:

```python
w.statement_execution.execute_statement(
    warehouse_id=space.warehouse_id,
    statement="EXPLAIN " + sql.strip().rstrip(";"),
    wait_timeout="30s",
)
```

**4c. Chat-mode benchmarks (automatic)**

Run the procedure in Appendix C. Before starting, tell the user how many questions will run. It takes about 15 seconds per question, and it uses warehouse compute and Genie usage.

**Scorecard format**

| Area | Baseline |
|---|---|
| IQ score | x/12 (tier: Not Ready / Ready to Optimize / Trusted) |
| Deep review | n ✅ · n ⚠️ · n ❌ |
| Benchmarks (chat mode) | x% (passed/total) |
| Benchmarks (agent mode) | x%, or "not run" |

Below the table, list every failed IQ check and every ❌ or ⚠️ deep-review check, one line each.

Save the scorecard to `{DIR}/scorecard_baseline_{TS}.json`.

### Step 5: Agent-mode benchmarks (optional, manual) ⏸

Explain: *"Genie Code can't start Agent-mode benchmark runs through the API yet. If you'd like, run them in the UI and I'll include the result."*

Ask: *"Do you want to run Agent-mode benchmarks now?"*

If yes, give these steps:
1. Open the Genie Agent and go to the **Benchmarks** tab.
2. If your workspace has a mode option for benchmark runs, choose **Agent mode**.
3. Click **Run all** (or select all questions and run them) and wait for it to finish.
4. Note the overall accuracy and the failing questions.
5. Paste the accuracy and the failing question text back here.

When they reply, record the result in the scorecard and explain what's next: *"I'll use the failing questions to rank the recommendations, then show you the list."*

If no, mark it "not run" and continue. Remind them they can run it at re-score time.

### Step 6: Recommendations ⏸

Build one recommendation per failed or warned item. Combine related items. Each recommendation has:

- **ID** (R1, R2, …) and a short title
- **Why**: the evidence from step 4
- **Fix**: what will change
- **Impact** (High / Medium / Low) and **Effort** (Low / Medium / High). Use Appendix D defaults, adjusted to the evidence.
- **Moves**: which IQ checks, deep-review checks and benchmark questions it should improve
- **Change type**: agent config, Unity Catalog metadata, or both

Sort by impact, then by lowest effort, then by the best-practice order in Appendix D. Show the top 10 as a table:

| ID | Recommendation | Impact | Effort | Moves | Type |
|---|---|---|---|---|---|

Ask: *"Which one should we start with? You can pick one (for example R2), say 'top 3', or ask me to explain any of them."*

### Step 7: Draft the change ⏸

For the chosen recommendation:

- **Agent config changes**: build the updated config in memory. Show a before/after diff of only the affected section.
- **Unity Catalog changes**: write the SQL, for example `COMMENT ON TABLE ...` and `ALTER TABLE ... ALTER COLUMN ... COMMENT '...'`. Descriptions must explain meaning, units and valid ranges in words. They must not contain real data values.
- **New or changed SQL** (example SQL, SQL expressions, functions): validate with `EXPLAIN` before showing it.

Ask: *"Apply this change? (yes / edit / skip)"*

### Step 8: Apply

**Mode A or C:**

```python
w.genie.update_space(SPACE_ID, serialized_space=json.dumps(new_cfg))
after = json.loads(w.genie.get_space(SPACE_ID, include_serialized_space=True).serialized_space)
```

- **Verify the agent config:** every section count must equal the "before" count, plus or minus only the intended change, and existing IDs must still be present. If not, restore the backup and report it.
- **Unity Catalog SQL:** run it only if the user approved it and has `MODIFY` on the tables. Otherwise save it to `{DIR}/proposed/uc_metadata_{TS}.sql` for the table owner.

**Mode B:**

Write these files and tell the user the paths:
- `{DIR}/proposed/serialized_space_{TS}.json`: the full updated config
- `{DIR}/proposed/uc_metadata_{TS}.sql`: if there are Unity Catalog changes
- `{DIR}/proposed/CHANGES_{TS}.md`: what changed, why, and which checks it should improve

Say: *"Commit these to your repo and deploy through your normal process. Tell me 'deployed' when it's live and I'll re-score."* ⏸

### Step 9: Re-score

Re-fetch the config and rerun step 4 (IQ score, deep review, chat-mode benchmarks). Ask whether they want to rerun Agent-mode benchmarks using the step 5 instructions.

Show a before/after table:

| Area | Before | After | Change |
|---|---|---|---|
| IQ score | | | |
| Tier | | | |
| Deep review (✅ / ⚠️ / ❌) | | | |
| Benchmarks, chat mode | | | |
| Benchmarks, agent mode | | | |

Then list each check that changed status. If a benchmark question went from pass to fail, flag it clearly and offer to roll back that change.

Save the scorecard to `{DIR}/scorecard_after_{TS}.json`.

### Step 10: Next or finish ⏸

Ask: *"Want to continue with the next recommendation (R?), pick another, or finish?"*

When they finish, save `{DIR}/review_report_{date}.md` with:
- Agent details, apply mode, and the backup path
- Baseline and final scorecards
- Changes made, and their status (applied, or PR-ready)
- Remaining recommendations, ordered
- How to roll back

Share the path. If Mode C was used, remind them to apply the proven changes to the real agent (Mode A or B) and to trash the copy.

## Rollback

```python
b = json.load(open("<backup path>"))
w.genie.update_space(SPACE_ID, serialized_space=json.dumps(b["serialized_space"]))
```

For Mode B, revert the commit in their repo.

## Appendix A: IQ score rubric (12 checks)

This approximates the Genie Workbench IQ Scan. If `genie-iq-score-lite` is available, its result takes precedence.

| # | Check | Pass when |
|---|---|---|
| 1 | Agent description | Present and meaningful: says what questions it answers and for whom |
| 2 | Table descriptions | ≥ 80% of tables have a description (Unity Catalog or agent-level) |
| 3 | Column descriptions | ≥ 50% of columns have a description |
| 4 | Text instructions | Total length > 50 characters |
| 5 | Join specs | Present whenever there are 2 or more regular tables (metric views excluded) |
| 6 | Data source count | Between 1 and 12 |
| 7 | SQL guidance | At least 1 SQL function, expression, measure, filter or example SQL |
| 8 | Entity matching / format assistance | Enabled on at least 1 column |
| 9 | Benchmarks | At least 10 benchmark questions |
| 10 | Noisy columns | No excess of visible internal or noisy columns (surrogate keys, ETL/audit timestamps, raw JSON) |
| 11 | Optimization run | A benchmark run has completed (from this session or reported by the user) |
| 12 | Accuracy | Best benchmark accuracy ≥ 85% |

**Tiers:**
- **Not Ready**: any of checks 1–10 fails
- **Ready to Optimize**: checks 1–10 pass, but 11 or 12 fails
- **Trusted**: all 12 pass

## Appendix B: Deep-review checks

These come from real Genie Agent reviews.

| # | Check | Flag when |
|---|---|---|
| D1 | Logic in text | Text instructions contain metric formulas, filters, joins or title mappings that could be SQL expressions, join specs or synonyms |
| D2 | Instruction size | Instructions are very long (as a guide, more than about 8,000 characters or 15 sections), or mix persona and tone text with data rules |
| D3 | Example SQL validity | Any example SQL fails `EXPLAIN` |
| D4 | Example SQL logic | Window functions partitioned on the wrong column; averaging rates or averages instead of dividing sums; comments describe a filter the SQL doesn't apply; a date window that doesn't match the stated default (for example `BETWEEN DATE_SUB(CURRENT_DATE(), 8) AND DATE_SUB(CURRENT_DATE(), 1)` is 8 days); `COUNT(*)` used as a user count; a missing `DISTINCT` on user metrics |
| D5 | Stale facts | Instructions contain hard-coded dates, row counts or measured values that will go out of date |
| D6 | Date windows | Defaults are relative to `CURRENT_DATE()` instead of the latest loaded date in each source |
| D7 | Data types | Numeric measures stored as STRING (they need CAST in SQL) |
| D8 | Metadata contradictions | Instructions say column comments are wrong or not to be trusted. Fix them in Unity Catalog instead |
| D9 | Trap columns | Columns the instructions warn against (misleading flags, always-NULL, deprecated) are still visible |
| D10 | Scope and overlap | Sources span several domains or grains; or the description overlaps other agents. Ask the user for related agent IDs and compare descriptions |
| D11 | Benchmark coverage | Fewer than 15 questions; or edge cases documented in the instructions have no benchmark |
| D12 | Description for routing | No sample questions; contains technical detail that doesn't help routing; doesn't say what's out of scope |
| D13 | Response verbosity | Instructions force a long plan or many metadata fields on every answer |
| D14 | Feedback signal | Few or no thumbs up or down, or review comments (ask the user to check the Monitoring tab) |
| D15 | Production readiness | Name contains [STAGE] or [DEV], or the prod agent isn't certified |

## Appendix C: Chat-mode benchmark procedure

For each benchmark question that has an expected SQL answer:

1. Ask the agent the question: `w.genie.start_conversation_and_wait(SPACE_ID, question)`. If that method isn't in the installed SDK, use `POST /api/2.0/genie/spaces/{space_id}/start-conversation` and poll the message.
2. Get the generated SQL and its result from the query attachment.
3. Run the expected SQL on the agent's warehouse with the Statement Execution API.
4. Compare the two results **in memory only**:
   - same column count
   - same row count
   - same values after sorting rows (compare hashes of each row)
   If either result has more than 5,000 rows, compare only column count and row count, and note it.
5. Record pass or fail and the reason (row count, columns, or values). Never print values.
6. Wait about 15 seconds between questions to stay within Genie API limits.

Report: accuracy = passed / total, and the failing **question text** only.

Note: this approximates the UI benchmark evaluation. The Benchmarks tab in the UI is the source of truth. These test conversations will appear in the agent's Monitoring tab under the user's name.

## Appendix D: Fix priority and default impact/effort

The order follows the Databricks best practices: metadata first, text instructions last.

| Order | Fix type | Typical impact | Typical effort |
|---|---|---|---|
| 1 | Unity Catalog metadata: table and column descriptions, correct data types, hide noisy columns | High | Low–Medium |
| 2 | Entity matching, format assistance, column synonyms | High | Low |
| 3 | SQL expressions (measures, filters, dimensions), join specs, metric views | High | Medium |
| 4 | Fix or add example SQL and trusted SQL functions | High | Low–Medium |
| 5 | Shorten text instructions (keep only behavior, scope and tone) | Medium | Medium |
| 6 | Expand benchmarks to 15–20, including edge cases | Medium | Medium |
| 7 | Description, sample questions, redirects to other agents | Medium | Low |
| 8 | Remove [STAGE] from the name and certify | Low (routing boost) | Low |

## References (GCP)

- Curate an effective Genie Agent: https://docs.databricks.com/gcp/en/genie-agents/best-practices
- Tune Genie Agent quality: https://docs.databricks.com/gcp/en/genie-agents/tune-quality
- Benchmarks: https://docs.databricks.com/gcp/en/genie-agents/benchmarks
- Genie Agents API: https://docs.databricks.com/gcp/en/genie-agents/conversation-api
- Metric views: https://docs.databricks.com/gcp/en/uc-semantics/metric-views
