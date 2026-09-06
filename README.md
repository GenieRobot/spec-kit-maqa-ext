# MAQA — Multi-Agent & Quality Assurance

> A [spec-kit](https://github.com/github/spec-kit) extension that adds a **coordinator → feature → QA** multi-agent workflow to any spec-kit project. Works with any language or framework.

## How it works

```text
speckit.maqa.sync          →   core taskstoissues for missing issues only
speckit.maqa.coordinator   →   SPAWN[N] feature assignments in isolated worktrees
                           →   SPAWN_QA[N] QA assignments
                           →   coordinator merged #N   →   re-assess, next batch
```

Each GitHub task issue runs in an **isolated git worktree**. The QA agent validates the committed implementation against that issue. You review and merge; the coordinator closes the issue only after verifying the commit reached the default branch.

GitHub Issues are the source of truth. Spec Kit's `tasks.md` remains read-only implementation and dependency context for worker and QA agents; MAQA does not maintain a competing `.maqa/state.json` backlog. Every command is designed to be safely retried after success, interruption, or partial failure.

## Requirements

- [spec-kit](https://github.com/github/spec-kit) `>=0.3.0`
- `git` with worktree support
- [GitHub CLI](https://cli.github.com/) authenticated for the repository in `remote.origin.url`

## Installation

```bash
specify ext add maqa
```

> Not in the catalog yet? Install directly:
> ```bash
> specify ext add https://github.com/GenieRobot/spec-kit-maqa-ext/archive/refs/tags/maqa-v0.3.1.zip
> ```

## Quick start

```bash
# 1. Install
specify ext add maqa

# 2. Idempotently initialize config, verify the selected AI, and create labels
/speckit.maqa.setup

# 3. Configure your test runner (optional but recommended)
#    Edit maqa-config.yml in your project root

# 4. Generate tasks with Spec Kit
/speckit.tasks

# 5. Accept MAQA's after_tasks prompt. It invokes Spec Kit taskstoissues only
#    for tasks whose GitHub issues do not already exist.
/speckit.maqa.sync

# 6. Run the coordinator
/speckit.maqa.coordinator
```

The coordinator reads state from the issues created through Spec Kit's `taskstoissues` command, correlates each issue with local Spec Kit context, reuses or creates stable worktrees, and returns a provider-neutral SPAWN plan. Spec Kit installs MAQA into the selected AI's native command or skills format. AIs with worker delegation can execute assignments in parallel; all others execute the same plan sequentially.

## Idempotency

MAQA uses stable reconciliation and event keys rather than a private state database:

- Task issues are keyed by the repository-relative `tasks.md` path plus task ID.
- Feature and remediation assignments use stable assignment keys recorded as git commit trailers.
- Worktree paths and branch names are deterministic and reused on retry.
- GitHub comments contain stable hidden event markers; existing transitions and comments are not repeated.
- Setup preserves user configuration and existing label definitions, adding only missing defaults.

This makes the workflow resumable across AI sessions and safe to run from different Spec Kit-supported tools.

## Board mirrors (optional)

Companion extensions remain available for teams that want a board view:

| Tool | Extension | Install |
|---|---|---|
| Trello | [maqa-trello](https://github.com/GenieRobot/spec-kit-maqa-trello) | `specify ext add maqa-trello` |
| Linear | [maqa-linear](https://github.com/GenieRobot/spec-kit-maqa-linear) | `specify ext add maqa-linear` |
| GitHub Projects | [maqa-github-projects](https://github.com/GenieRobot/spec-kit-maqa-github-projects) | `specify ext add maqa-github-projects` |
| Jira | [maqa-jira](https://github.com/GenieRobot/spec-kit-maqa-jira) | `specify ext add maqa-jira` |
| Azure DevOps | [maqa-azure-devops](https://github.com/GenieRobot/spec-kit-maqa-azure-devops) | `specify ext add maqa-azure-devops` |

MAQA 0.3.x never reads these boards as workflow authority and does not auto-select one. Configure `board_mirror` explicitly if a companion supports issue-state mirroring; mirror failures never change GitHub issue state or scheduling.

## CI gate (optional)

```bash
specify ext add maqa-ci
/speckit.maqa-ci.setup
```

With [maqa-ci](https://github.com/GenieRobot/spec-kit-maqa-ci) installed, the coordinator checks pipeline status on the feature branch before moving a card to In Review. Supports GitHub Actions, CircleCI, GitLab CI, and Bitbucket Pipelines.

## Configuration

`maqa-config.yml` is created in your project root by `/speckit.maqa.setup` (or copied manually from `.specify/extensions/maqa/config-template.yml`).

| Field | Default | Description |
|---|---|---|
| `source_of_truth` | `"github-issues"` | The authoritative work source in MAQA 0.3.x. |
| `github_label_prefix` | `"maqa"` | Prefix for issue workflow labels. |
| `dispatch_mode` | `"auto"` | Use native parallel delegation when available, otherwise run assignments sequentially. |
| `test_command` | `""` | Full test suite — e.g. `npm test`, `pytest`, `bundle exec rspec` |
| `test_file_command` | `""` | Single file — e.g. `pytest {file}`, `npm test -- {file}` |
| `tdd` | `false` | Write tests first, then implement. Red is assumed (no pre-run). |
| `auto_push` | `false` | Push feature branch after commit. Recommended: `true` — without it, work only exists in the local worktree and is lost if the worktree is deleted before merging. |
| `qa_cadence` | `"per_feature"` | `per_feature`: QA runs after each feature agent (catches regressions early). `batch_end`: QA runs once when the full batch is done (saves credits). |
| `max_parallel` | `3` | Max concurrent feature agents |
| `worktree_base` | `".."` | Where worktrees are created (relative to repo root) |
| `board_mirror` | `"none"` | Optional one-way companion mirror. Never a source of truth. |
| `qa.text` | `true` | Spelling, grammar, placeholder copy |
| `qa.links` | `true` | Link / route verification |
| `qa.security` | `true` | Unfiltered output, exposed params, missing auth |
| `qa.accessibility` | `false` | WCAG 2.1 AA — enable for web projects |
| `qa.responsive` | `false` | Mobile / responsive layout — enable for web projects |
| `qa.empty_states` | `false` | Empty / error states — enable for UI projects |

## Commands

| Command | Description |
|---|---|
| `/speckit.taskstoissues` | Core Spec Kit command used by MAQA to create task issues |
| `/speckit.maqa.sync` | Idempotently call `taskstoissues` for missing issues only |
| `/speckit.maqa.coordinator` | Assess issues, create worktrees, return SPAWN plan |
| `/speckit.maqa.feature` | Implement one issue in one worktree |
| `/speckit.maqa.qa` | Validate one committed issue implementation |
| `/speckit.maqa.setup` | Agent-neutral, idempotent config and GitHub-label bootstrap |

## AI tool support

| AI capability | Mode |
|---|---|
| Native worker/subagent delegation | Execute provider-neutral SPAWN assignments in parallel |
| No worker delegation | Execute the same assignments sequentially or in context |

The extension contains no Claude-only setup files or runtime assumptions. Command filenames and invocation syntax are rendered by Spec Kit's extension registrar for the selected AI, including future integrations that support the same registrar contract.

## How work is tracked

Issue state and MAQA labels define workflow state:

```
open → maqa:in-progress → maqa:in-review → closed
                 ↘ maqa:blocked
```

An open issue without a MAQA state label is ready/todo. The coordinator applies labels and comments; the issue is closed only after its commit is verified on the default branch.

Explicit issue dependencies are authoritative. When an issue has none, the matched `tasks.md` dependency graph and `[P]` markers guide scheduling without becoming a second state store. Ambiguous matches stop reconciliation; missing matches direct the user to `/speckit.maqa.sync`.

## License

MIT — free for all.
