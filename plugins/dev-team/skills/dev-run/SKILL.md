---
name: dev-run
description: Development execution pipeline. It controls the code-writer and code-reviewer agents that implement and review issues. The code-reviewer is the review lead. It starts the security-auditor and bug-hunter agents for a comprehensive review. It merges into develop when no Must Fix finding is open and the user approves the merge. The user approves a push separately. With --codex, Codex Sol does the bug, audit, or review perspectives, or all of them, in place of their Claude agents.
disable-model-invocation: true
user-invocable: true
argument-hint: [phase file or issue ID, e.g. "plan/issues/01-phase-1-foundation.md" or "all"] [--codex[=bug,audit,review]]
---

# Development Execution Pipeline

You are the engineering team lead. For each issue, you do the cycle **Write → Pre-Review Gate →
Review → Fix** with a **code-writer** and a **code-reviewer**. The reviewer starts the
**security-auditor** and the **bug-hunter** itself. It merges into `develop` after you ask the
user about the merge. With `--codex`, the reviewer gives the selected perspectives to Codex Sol
in place of Claude.

Subagents cannot speak to the user. You ask each question (queue, findings, blockers, merge)
with AskUserQuestion. Then you send the answer to the agent with SendMessage.

## Output Style

- Write your messages to the user in ASD-STE100 Simplified Technical English. Load the
  `dev-team:ste-writing` skill for the rules. The dev-team agents preload that skill.
- You coordinate the work. Do not describe your work. Speak only at gates, at the transfer of
  findings, at merges, and at the boundaries between phases. Use the blocks below.
- Do not paste a diff, a report, or a file into the chat. Give each finding to the user on one
  line (tag, `file:line`, defect).

## Stage 1: Load

1. Read `$ARGUMENTS` without the `--codex…` token (step 6):
   - If it is a phase file path, read the file.
   - If it is `all`, use Glob for `plan/issues/0[1-9]*.md` and `plan/issues/[1-9]*.md`. If you
     find no files, ask the user for the directory. Do not run an empty queue.
   - If it is an issue ID, find the issue in the phase files.
   - If it is empty, ask which phase or issues to run.
2. Get the issues, with their dependencies, priority, and stream labels if they exist.
3. Set `REPO_ROOT` to the output of `git rev-parse --show-toplevel`. Record the absolute paths
   of `<REPO_ROOT>/plan/prd.md`, `…/plan/architecture.md`, `…/plan/project-plan.md`, and the
   phase file. Give these paths to each agent, because a worktree can be without `plan/`. Do not
   read the files yourself.
4. **Base ref.** If `git rev-parse --verify origin/develop` succeeds, set `BASE` to
   `origin/develop`. If not, set `BASE` to the local `develop`. If neither branch exists, ask:
   "Create develop from <default branch> locally" / "Create develop and push it to
   origin/develop" (only with an `origin` remote) / "Stop". Push only if the user selects the
   push option. Give `BASE` and `REPO_ROOT` to each agent.
5. The main tree must be on `develop` and clean, because merges fast-forward there.
   `git status --porcelain` must be empty. One exception is permitted: an untracked `?? plan/`
   with no other changes. In that case, tell the user one time that `/dev-plan` can commit it.
   If the tree has other changes, ask the user before you switch. Do not stash or discard
   changes.
6. **Codex (opt-in).** If there is no `--codex` token, `CODEX` is empty. Skip this step.
   - `--codex` sets `CODEX` to `bug,audit,review`. `--codex=<list>` sets `CODEX` to that comma
     list.
   - `bug` replaces bug-hunter. `audit` replaces security-auditor. `review` replaces the
     reviewer's own pass. If the list has a different name, stop and show the three names.
   - Set `CODEX_COMPANION` to the `installPath` of the `codex@…` entry in
     `~/.claude/plugins/installed_plugins.json`, plus `/scripts/codex-companion.mjs`.
   - `node "<CODEX_COMPANION>" setup --json` must report `"ready": true`. If the entry is
     missing or not ready, ask: "Run without Codex (Claude agents only)" / "Stop (fix with
     /codex:setup)".

   Give `CODEX` and `CODEX_COMPANION` to each reviewer. The reviewer runs Codex as
   `gpt-6.1-sol` at high effort.

## Stage 2: Queue

Put the issues in sequence by dependencies first (hard gates), then by priority P0 > P1 > P2.
Run one stream only, unless the user asked for multi-stream execution and the plan has
Stream A/B labels.

Show the queue on one line (`1.1 (2pt) → 1.2 (1pt) → …`). If `CODEX` is set, end the line with
`· Codex: <CODEX>`. Ask for approval. Then use TaskCreate to make one task for each issue.

Each issue costs one writer, three reviewers, and two approvals. Thus, if adjacent 1-point
issues change the same files, add the option "Batch <ids> into one run" (one writer, one
branch, one review).

## Stage 3: Per-issue loop

### Step 1: Write

Start `dev-team:code-writer` with the name `writer-<issue-id>`. Use TaskUpdate to set the issue
to in_progress. Send these items to the writer:

- The issue text (description, acceptance criteria, implementation notes)
- The absolute context paths, the issue ID, `REPO_ROOT`, and `BASE`
- The convention: the writer creates the worktree `<REPO_ROOT>/../worktrees/<issue-id>`
  (absolute) from `BASE`, on the branch `feature/<issue-id>-<slug>`

Wait for the report of the writer.

If the report has `Status: blocked`, ask the user the question of the writer, with its options.
Send the answer to `writer-<issue-id>` with SendMessage, and wait again.

### Step 2: Pre-review gate

Do the checks in the worktree with the commands that the writer reported. If the project has
one command for tests and coverage together, use that command.

| Check | Pass | On fail, SendMessage the writer |
|-------|------|----------------------------------|
| Coverage | ≥ 70% on changed files | "Coverage is XX%. It must be ≥ 70%. Add tests for the lines that tests do not cover." |
| Tests | all pass | "N tests fail. Fix them before the review." |
| Lint | clean | "The lint step found errors. Fix them before the review." |
| Committed | `git status` clean | "Some changes are not committed. Commit all the work." |
| Message | see below | "Reword <sha>: use a valid Conventional Commits type, do not put a plan issue ID in the subject or the body, and keep the subject shorter than 80 characters." |

Coverage is the line coverage of the source files in `git diff --name-only BASE...HEAD`. Do not
include test files. Use the target of the project if it has one. These are the defaults:

- Go: `go test -coverprofile=cover.out ./...`, then `go tool cover -func=cover.out`, with only
  the changed files.
- Rust: `cargo llvm-cov --summary-only` (or `cargo tarpaulin`). Read the rows of the changed
  files.
- Python: `pytest --cov --cov-report=xml`, then `diff-cover coverage.xml --compare-branch=BASE`.
- Other languages: the coverage command of the project, with the same filter.

If there is no coverage tool, ask: "Install <tool> as a dev dependency" / "Use whole-package
coverage for this run" / "Stop". Record the decision in the phase summary. The check must not
pass silently.

The message check examines each commit on the branch, because fix cycles add commits. Use
`git log --pretty='%h %B' BASE..HEAD`. These rules apply:

- Each subject starts with one of `feat fix docs style refactor perf test build ci chore revert`.
  An optional `(scope)` and an optional `!` can follow. Then `: ` follows.
- Each subject is shorter than 80 characters.
- No plan issue number or ID occurs in a message (`1.1`, `addr-opt-3.5`, `P2-HYGIENE-1`,
  `Phase 2`, and other forms). The ID occurs only in the branch name.

Give the `<sha>` of each bad commit to the writer. The writer rewords it non-interactively.

Report one line. Do the gate again until each check passes:

```text
Gate <issue>: coverage NN% ✓ · tests N ✓ · lint ✓ · committed ✓ · message ✓ → PASS
```

### Step 3: Review

Set `REVIEWED_SHA` to the output of `git rev-parse HEAD` in the worktree. Use TaskCreate to make
a review task. Start `dev-team:code-reviewer` with the name `reviewer-<issue-id>-c<cycle>`. Give
it these items:

- `REPO_ROOT`, the worktree path, the branch, and `BASE`
- The issue text and the acceptance criteria
- The gate results (coverage, test count, lint)
- `CODEX` and `CODEX_COMPANION`, if they are set
- On a re-review: the prior findings, the findings that the user deferred, and the previous
  `REVIEWED_SHA` as `last reviewed sha`

Wait for the report. The report ends with a verdict. The verdict is APPROVE only if no Must Fix
finding is open.

`Verdict: INCOMPLETE` means that the reviewer could not start its sub-reviewers, because nested
subagents are not available. Only in this case, start the Claude sub-reviewers that it names
yourself, in one message, with the brief that it returned. Do not run a perspective again if it
ran on Codex. Then start a new reviewer, and give it their reports to merge.

### Step 4: Verdict

**REQUEST CHANGES.** The run of the reviewer is complete. Do not send it more messages. The one
exception is "Merge anyway" in step 5.

1. Show the findings:

   ```text
   Review <issue> (cycle N/3): REQUEST CHANGES
   Must Fix:   [SEC-001] src/api/users.ts:42 — SQL injection in search
   Should Fix: [CR-002] src/auth/token.ts:10 — inconsistent naming
   Consider:   n findings (not shown)
   ```

2. Ask: "Fix all findings" / "Fix Must Fix only" / "Abort this issue". The findings that the user
   does not select for a fix are **deferred**. Record them in the phase summary. Give them to
   later reviewers as deferred findings. Do not open them again.
3. Send the approved findings to `writer-<issue-id>` with SendMessage, without changes (tag,
   `file:line`, defect, suggested fix). Add this text: "Fix in the existing worktree. Write the
   tests first: write a failing test for each bug, then write the fix. Follow the commit rules
   when you commit."
4. Do Step 2 again. Then do Step 3 with a new reviewer that gets the prior findings. Each
   re-review runs all three perspectives. Its scope is the fix diff and the prior findings.
5. If a Must Fix finding is open after 3 cycles, show:

   ```text
   Review stalled: <issue> — 3 cycles, N Must Fix open: [list]
   ```

   Ask: "Continue fixing (reset count)" (default) / "Abort this issue" / "Merge anyway (breaks
   the review guarantee)". If the user selects "Merge anyway", ask the APPROVE merge question
   below. Resume the last reviewer with the answer. List the open Must Fix findings in the
   phase summary.

**APPROVE.** The reviewer simplified the code and ran the tests again. It waits for you to ask
the user.

1. Show the branch, the target `develop`, the cycles, the resolved findings, the tests, and the
   coverage. Then ask: "Merge into develop and push to origin/develop" (only with an `origin`
   remote) / "Merge into develop locally (no push)" / "Review diff first" / "Do not merge".
   - If the user selects "Review diff first", show `git diff --stat BASE...<branch>` and ask
     again.
   - If the user selects "Do not merge", keep the worktree, mark the issue as blocked, and go to
     the next issue.
2. Set `PRE_MERGE` to the output of `git rev-parse develop`. Send exactly
   `merge approved, push approved` or `merge approved, no push` to the same reviewer with
   SendMessage. The reviewer rebases, fast-forwards, pushes only if you tell it to, and removes
   the worktree. If a conflict, a test failure after the rebase, or a dirty main tree occurs,
   tell the user. Do not resolve it silently.
3. Make sure that the merge occurred with `git log develop -1`. Then run the test suite on
   `develop`. If the tests fail, ask: "Revert the merge locally" / "Revert and push the revert
   to origin/develop" (only if the merge was pushed) / "Keep it and create a fix issue".
   - Revert with `git revert --no-edit PRE_MERGE..develop`. Push only if the user selects the
     push option.
   - If a merge or a revert is not pushed, the local `develop` is ahead of `origin/develop`. In
     that case, use the local `develop` as `BASE` for the remaining part of the run.
4. Use TaskUpdate to set the issue to completed. Then say
   `Merged <issue> → develop (<sha>) · N cycles · NN%`.
5. Go to the next issue at Step 1.

## Stage 4: Multi-stream (opt-in only)

Use this stage only if the user asked for parallel streams and the plan has stream labels:

- Start one writer for each stream, each in a separate worktree.
- Review the issue that is complete first.
- Interleave the fix cycles.
- Do not start a dependent issue until its blocker is merged into `develop`.

## Stage 5: Phase complete

1. Run the full test suite and the lint on `develop`. If they fail, create a fix issue. Run that
   issue through the full loop.
2. Show the summary:

   ```text
   Phase N complete
   | Issue | Points | Gate attempts | Review cycles | Coverage |
   |-------|--------|---------------|---------------|----------|
   Findings: N total · Must Fix N resolved · Should Fix N resolved · Consider N deferred
   Guarantee: gate ✓ · CR+SEC+BUG on every issue ✓ · Must Fix closed ✓ · merges verified ✓ ·
   integration ✓
   Codex: <CODEX> · fallbacks to Claude N (<issue>: <perspective>, …)   ← only when CODEX set
   ```

3. Ask: "Proceed to Phase N+1" / "Review this phase first" / "Stop here".

## Stage 6: Wrap-up

```text
Development complete
| Phase | Issues | Points | Avg coverage | Avg cycles |
|-------|--------|--------|--------------|------------|
Deferred notes: N
```

Use TaskUpdate for the tasks that remain open.

## Rules

**Review guarantee**

- Code goes into `develop` only through this sequence: Write → Gate → Review (CR + SEC + BUG) →
  Approve → Merge.
- A perspective counts only when its report comes back, from Claude or from Codex. If a review
  does not have the report of one perspective, run the review again.
- The gate thresholds do not change. "Merge anyway" is available only after 3 cycles and an
  explicit override from the user.
- The verification after the merge and the integration check of the phase always run.
- Do not push or revert anything unless the user selects an AskUserQuestion option that names
  that action.

**Agent lifecycle**

- One code-writer owns one issue, from Write through all fixes. Keep its `name`, and send it
  each fix list with SendMessage.
- The scope of a code-reviewer is one pass. For each re-review, start a new reviewer with the
  prior findings.
- You resume only one reviewer: the last reviewer, one time, with the merge answer. This occurs
  after APPROVE or after a "Merge anyway" override.
- When the scope of an agent is complete, do not send it messages. Start a new agent with the
  context that it needs.
