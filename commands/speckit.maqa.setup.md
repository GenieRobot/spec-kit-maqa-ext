---
description: "Idempotent MAQA bootstrap for every Spec Kit AI integration. Preserves user config, verifies command registration and GitHub identity, and creates only missing labels."
---

You are the MAQA Setup role. Configure MAQA without assuming Claude Code or any particular agent runtime. Re-running this command must converge to the same state without overwriting user choices or duplicating files, labels, or configuration keys.

## Why no agent-specific files are created

`specify extension add maqa` already renders every MAQA command into the selected AI's native command or skill format through Spec Kit's command registrar. Do not create `.claude/agents`, `.gemini`, `.agents`, `.github/agents`, or any other provider-specific files here.

The coordinator returns provider-neutral `SPAWN`, `SPAWN_QA`, and `SPAWN_FIX` data. A parent AI with native delegation may run those roles in parallel; an AI without delegation runs the same assignments sequentially or in context.

## Step 1 — Verify the installed extension

From the repository root, require these canonical installed files:

```text
.specify/extensions/maqa/extension.yml
.specify/extensions/maqa/config-template.yml
.specify/extensions/maqa/commands/speckit.maqa.coordinator.md
.specify/extensions/maqa/commands/speckit.maqa.sync.md
.specify/extensions/maqa/commands/speckit.maqa.feature.md
.specify/extensions/maqa/commands/speckit.maqa.qa.md
```

Read `.specify/init-options.json` when present and report the selected AI and whether skills mode is enabled. Do not fail merely because the AI is unknown or future; the canonical extension commands remain the fallback contract.

Check the selected AI's registered commands using Spec Kit's recorded extension registry. If registration is missing, stop with the exact repair command `specify extension update maqa`; do not synthesize provider-specific files.

## Step 2 — Initialize config without clobbering it

If `maqa-config.yml` does not exist, copy the bundled template exactly once.

If it exists, preserve every current value and comment. Add only missing top-level compatibility keys with these defaults:

```yaml
source_of_truth: "github-issues"
github_label_prefix: "maqa"
dispatch_mode: "auto"
board_mirror: "none"
```

Never replace the whole file and never duplicate a key. If `source_of_truth` exists with a value other than `github-issues`, stop and report the conflict instead of changing it.

Treat a repeated run with no missing keys as `config_action: no_op`.

## Step 3 — Lock GitHub identity

Resolve `OWNER/REPO` only from `git remote get-url origin`, supporting normal GitHub HTTPS and SSH URLs. Verify that exact value using:

```bash
gh auth status
gh repo view "$OWNER/$REPO" --json nameWithOwner,url,defaultBranchRef
```

Stop if the remote is not GitHub, authentication fails, or the returned `nameWithOwner` differs. Every following `gh` command must include `--repo "$OWNER/$REPO"`.

## Step 4 — Create only missing workflow labels

Read `github_label_prefix` from config and derive:

```text
<prefix>
<prefix>:in-progress
<prefix>:in-review
<prefix>:blocked
```

Fetch current labels once with `gh label list --repo "$OWNER/$REPO" --limit 1000 --json name`. Create a label only when its exact name is absent. Never use `--force`, never overwrite an existing label's color or description, and never swallow an authentication or API error.

Use these defaults only for newly created labels:

| Suffix | Color | Description |
|---|---|---|
| none | `5319E7` | Managed by MAQA |
| `in-progress` | `FBCA04` | MAQA worker active |
| `in-review` | `0E8A16` | MAQA QA or merge review |
| `blocked` | `B60205` | MAQA work blocked |

## Step 5 — Report legacy Claude setup without deleting it

If `.claude/agents/coordinator.md`, `feature.md`, or `qa.md` exists from MAQA 0.2.x or earlier, report it under `legacy_files`. Do not edit or delete user files. They are no longer required because Spec Kit registers MAQA commands for all AIs.

## Return format

Return only this TOON block:

```text
setup_status: configured | no_op | blocked
agent: <selected AI or unknown>
skills_mode: true | false | unknown
repository: <OWNER/REPO>
config_action: created | updated_missing_keys | no_op
labels_created[N]{name}:
  <label>
labels_existing[N]{name}:
  <label>
legacy_files[N]{path}:
  <path>
next: /speckit.maqa.sync
summary: <one sentence>
```

## Hard rules

- Never write provider-specific agent files.
- Never overwrite user config or duplicate config keys.
- Never recreate or modify an existing GitHub label.
- Never access a repository other than the verified GitHub `origin`.
- A successful second run with unchanged inputs must return `setup_status: no_op`.
