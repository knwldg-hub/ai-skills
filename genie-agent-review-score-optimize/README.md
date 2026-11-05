# genie-agent-review-score-optimize (Genie Code skill, v0.2 pilot)

A Genie Code skill that reviews, scores and optimizes an existing Genie Agent (formerly Genie Space). The skill file is [`SKILL.md`](./SKILL.md).

In the Databricks workspace, Genie Code only loads a skill from `.assistant/skills/<skill-name>/SKILL.md`. The install steps below save the file under that name.

## Install

Pick one option. Each installs the skill for one user. Everyone who runs reviews installs it in their own folder.

### Option 1: ask Genie Code to install it (fastest)

Raw file URL (copy it exactly, with no trailing punctuation):

```
https://raw.githubusercontent.com/knwldg-hub/ai-skills/main/genie-agent-review-score-optimize/SKILL.md
```

Open Genie Code in your workspace and paste:

> Download the file at the raw URL below and show me a summary of its content so I can review it. After I approve, create the folder `/Workspace/Users/<your-email>/.assistant/skills/genie-agent-review-score-optimize/`, save the content there as `SKILL.md`, and confirm the skill is installed.
>
> URL: `https://raw.githubusercontent.com/knwldg-hub/ai-skills/main/genie-agent-review-score-optimize/SKILL.md`

Note: Genie Code has a safety guardrail against writing externally fetched content into its own `.assistant/skills` folder (prompt-injection protection), so it may ask you to review the content first or refuse the write. If it refuses, use option 2 or 3. Also use option 2 or 3 if the workspace can't reach GitHub.

### Option 2: manual, in the workspace UI

1. Go to **Workspace → Users → your folder**.
2. Create the folder `.assistant/skills/genie-agent-review-score-optimize`. Folders that start with a dot may be hidden; turn on hidden files if you don't see them.
3. In that folder, choose **Create → File**, name it `SKILL.md`, and paste the content of [`SKILL.md`](./SKILL.md).

### Option 3: Databricks CLI

```bash
databricks workspace mkdirs "/Workspace/Users/<your-email>/.assistant/skills/genie-agent-review-score-optimize"

databricks workspace import \
  "/Workspace/Users/<your-email>/.assistant/skills/genie-agent-review-score-optimize/SKILL.md" \
  --file ./genie-agent-review-score-optimize/SKILL.md \
  --format AUTO --overwrite
```

Check that it's installed: ask Genie Code "Is the genie-agent-review-score-optimize skill installed?"

## Run a review

Open Genie Code from a notebook, ideally a new empty one, and say one of:

- "Use the genie-agent-review-score-optimize skill to review Genie Agent <agent-id>."
- "Review, score and optimize my Genie Agent <agent-id>."

Other useful prompts during a session:

| You want to... | Say |
|---|---|
| Fix a recommendation | "Start with R3" or "Do the top 3" |
| See the current scorecard | "Show my scorecard" |
| Score again after a deploy | "It's deployed, re-score" |
| Undo changes | "Restore the backup" |
| Add more benchmark questions | "Draft more benchmarks" |
| Finish and get the report | "Finish the review" |

## Prerequisites

- Genie Code enabled in the workspace
- CAN EDIT on the Genie Agent being reviewed. CAN VIEW is enough to review without changes.
- CAN USE on the agent's SQL warehouse
- To apply Unity Catalog description fixes: MODIFY on those tables. Otherwise the skill writes a SQL script for the table owner.

## Testing tip

For the first runs, pick **Mode C (test on a copy)** at step 2. The skill clones the agent and makes all changes on the copy, so the original stays untouched.
