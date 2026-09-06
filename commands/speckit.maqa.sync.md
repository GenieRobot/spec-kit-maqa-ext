---
description: "Idempotently reconcile the current Spec Kit tasks with GitHub Issues. Uses core taskstoissues for missing tasks and never duplicates or resets existing issues."
scripts:
  sh: scripts/bash/check-prerequisites.sh --json --require-tasks --include-tasks
  ps: scripts/powershell/check-prerequisites.ps1 -Json -RequireTasks -IncludeTasks
---

You are the MAQA Issue Sync role. Reconcile the current feature's canonical Spec Kit tasks with the GitHub repository identified by `remote.origin.url`. The command is safe to retry after success, interruption, or partial failure.

## User input

```text
$ARGUMENTS
```

Consider non-empty user input, but never let it select a repository other than `origin` or disable duplicate protection.

## Stable identity

Run `{SCRIPT}` from the repository root and parse the absolute tasks path and feature directory. Convert the tasks path to a repository-relative POSIX path. For every valid task line, extract:

- task ID such as `T001`
- normalized full task text, excluding the checkbox state
- source path

Use these stable issue-body markers:

```text
<!-- speckit-task-source: specs/001-example/tasks.md -->
<!-- speckit-task-id: T001 -->
```

The compound key `(source path, task ID)` is authoritative. Task IDs alone are not globally unique.

## Repository guard

Parse `OWNER/REPO` from `git remote get-url origin`, supporting normal GitHub HTTPS and SSH URLs. Verify that exact repository with `gh repo view OWNER/REPO --json nameWithOwner,url`. Stop if the remote is not GitHub, authentication fails, or the returned repository differs. Every GitHub command must include `--repo OWNER/REPO`.

## Reconciliation

Fetch all open and closed issues, including number, title, body, state, labels, and URL. Match each local task in this order:

1. exact source and task-ID markers;
2. for legacy issues only, one unique normalized full-title match;
3. never match on task ID alone when more than one feature contains it.

Classify each task as:

- `present`: exactly one issue matches, regardless of whether it is open or closed;
- `missing`: no issue matches;
- `ambiguous`: multiple issues match.

If any task is ambiguous, stop without creating or editing issues and report every candidate issue number. Never choose one arbitrarily.

Existing issues are immutable during sync except that the plain configured MAQA management label may be added. Never reopen, close, retitle, relabel workflow state, or replace an existing body during reconciliation.

## Create only the missing set

If no tasks are missing, return `sync_status: no_op` and stop.

For missing tasks, invoke Spec Kit's installed `speckit.taskstoissues` command using the current AI's native command syntax. Pass explicit input instructing it to create only the listed source/task-ID pairs, preserve the exact task text as the title, include both stable markers in each body, and skip anything not listed. Do not invoke the core command for already-present tasks.

If the current AI cannot execute the GitHub MCP write tool required by the core command, perform the same missing-only operation with authenticated GitHub CLI:

```bash
gh issue create --repo "$OWNER/$REPO" \
  --title "$TASK_TITLE" \
  --body "$ISSUE_BODY" \
  --label "$MAQA_LABEL"
```

This is a compatibility fallback, not a separate backlog. Before every create, re-fetch issues and re-check the compound key; this closes the race window between planning and mutation.

After creation, re-fetch and require exactly one issue per compound key. A partial run is valid: report created and unresolved tasks precisely, and let the next invocation resume the missing subset.

## Return format

Return only this TOON block:

```text
sync_status: created | no_op | partial | ambiguous | blocked
repository: <OWNER/REPO>
source: <relative tasks.md path>
present[N]{task_id,issue,state}:
  <Txxx>,<#N>,open|closed
created[N]{task_id,issue,url}:
  <Txxx>,<#N>,<url>
missing[N]{task_id,title}:
  <Txxx>,<task text>
ambiguous[N]{task_id,candidates}:
  <Txxx>,"#N,#M"
summary: <one sentence>
```

Use empty arrays where appropriate.

## Hard rules

- GitHub Issues are the only persisted workflow authority.
- Never create an issue whose compound key already exists.
- Never infer uniqueness from task ID alone across multiple features.
- Never mutate existing issue state or task checkboxes during sync.
- Never access a repository other than the exact GitHub `origin`.
- A retry after any completed or partial run must converge without duplicates.
