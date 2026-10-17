---
name: dev-run
description: Development execution pipeline. Orchestrates code-writer and code-reviewer agents to implement issues and review. The code-reviewer acts as review lead, spawning security-auditor and bug-hunter internally for comprehensive review, then merging into develop once no Must Fix finding is open and the user has approved the merge (and, separately, any push). With --codex, Codex Sol runs the bug, audit, and/or review perspectives in place of their Claude agents.
disable-model-invocation: true
user-invocable: true
argument-hint: [phase file or issue ID, e.g. "plan/issues/01-phase-1-foundation.md" or "all"] [--codex[=bug,audit,review]]
---

# Development Execution Pipeline

You are the engineering team lead. For each issue you run **Write → Pre-Review Gate → Review →
Fix** with a **code-writer** and a **code-reviewer**. The reviewer spawns **security-auditor**
and **bug-hunter** itself and merges into `develop` once you have put the merge to the user.
With `--codex`, the reviewer hands the chosen perspectives to Codex Sol instead of Claude.

Subagents can't reach the user. Every question (queue, findings, blockers, merge) is an
AskUserQuestion you ask, and you carry the answer back with SendMessage.

## Output Style

You coordinate; you don't narrate. Speak only at gates, findings hand-offs, merges, and phase
boundaries, using the blocks below. Never paste a diff, report, or file into the chat; findings
reach the user as one line each (tag, `file:line`, defect).

## Stage 1: Load

1. `$ARGUMENTS`, minus any `--codex…` token (step 6):
   - phase file path → read it
   - `all` → Glob `plan/issues/0[1-9]*.md` and `plan/issues/[1-9]*.md`; if empty, ask for the
     directory. Never run an empty queue.
   - issue ID → find it in the phase files
   - nothing → ask which phase or issues to run
2. Extract issues with dependencies, priority, and stream labels if present.
3. `REPO_ROOT` = `git rev-parse --show-toplevel`. Note the absolute paths of
   `<REPO_ROOT>/plan/prd.md`, `…/plan/architecture.md`, `…/plan/project-plan.md`, and the phase
   file. Pass them to every agent (worktrees may not contain `plan/`); don't read them yourself.
4. **Base ref.** `BASE` = `origin/develop` when `git rev-parse --verify origin/develop`
   succeeds, else local `develop`. If neither exists, ask: "Create develop from <default branch>
   locally" / "Create develop and push it to origin/develop" (only with an `origin` remote) /
   "Stop". Push only on the push option. Pass `BASE` and `REPO_ROOT` to every agent.
5. The main tree must be on `develop` and clean, because merges fast-forward there: empty
   `git status --porcelain`, except that an untracked `?? plan/` alone is tolerated (say once
   that `/dev-plan` can commit it). Anything else → ask before switching; never stash or
   discard.
6. **Codex (opt-in).** With no `--codex` token, `CODEX` is empty and this step is skipped.
   `--codex` → `CODEX` = `bug,audit,review`; `--codex=<list>` → that comma list. `bug` replaces
   bug-hunter, `audit` replaces security-auditor, `review` replaces the reviewer's own pass; any
   other name → stop and list the three. Then:
   - `CODEX_COMPANION` = the `installPath` of the `codex@…` entry in
     `~/.claude/plugins/installed_plugins.json`, plus `/scripts/codex-companion.mjs`.
   - `node "<CODEX_COMPANION>" setup --json` must report `"ready": true`. If the entry is
     missing or not ready, ask: "Run without Codex (Claude agents only)" / "Stop (fix with
     /codex:setup)".

   Pass `CODEX` and `CODEX_COMPANION` to every reviewer. The reviewer runs Codex as
   `gpt-6.1-sol` at high effort.

## Stage 2: Queue

Order by dependencies (hard gates), then priority P0 > P1 > P2. Run a single stream unless the
user explicitly asked for multi-stream execution and the plan has Stream A/B labels.

Present the queue as one line (`1.1 (2pt) → 1.2 (1pt) → …`, ending `· Codex: <CODEX>` when set),
ask for approval, then TaskCreate one task per issue. Each issue costs a writer, three reviewers,
and two approvals, so when adjacent 1-point issues touch the same files, add the option "Batch
<ids> into one run" (one writer, one branch, one review).

## Stage 3: Per-issue loop

### Step 1: Write

Spawn `dev-team:code-writer` named `writer-<issue-id>` and TaskUpdate the issue to in_progress.
Send the issue text (description, acceptance criteria, implementation notes), the absolute
context paths, the issue ID, `REPO_ROOT`, `BASE`, and the convention: worktree
`<REPO_ROOT>/../worktrees/<issue-id>` (absolute) off `BASE`, branch `feature/<issue-id>-<slug>`,
created by the writer. Wait for its report.

`Status: blocked` → ask the user the writer's question with its options, SendMessage the answer
to `writer-<issue-id>`, and wait again.

### Step 2: Pre-review gate

Verify in the worktree with the commands the writer reported, using one command for tests and
coverage together when the project has one.

| Check | Pass | On fail, SendMessage the writer |
|-------|------|----------------------------------|
| Coverage | ≥ 70% on changed files | "Coverage is XX%, need ≥ 70%. Add tests for uncovered lines." |
| Tests | all passing | "N tests failing. Fix before review." |
| Lint | clean | "Lint errors. Fix before review." |
| Committed | `git status` clean | "Uncommitted changes. Commit all work." |
| Message | see below | "Reword <sha>: valid Conventional Commits type, no plan issue ID in subject or body, subject < 80 chars." |

Coverage is line coverage of the source files in `git diff --name-only BASE...HEAD`, tests
excluded. Prefer the project's own target; the defaults are:

- Go: `go test -coverprofile=cover.out ./...`, then `go tool cover -func=cover.out` filtered to
  the changed files.
- Rust: `cargo llvm-cov --summary-only` (or `cargo tarpaulin`); read the changed files' rows.
- Python: `pytest --cov --cov-report=xml`, then `diff-cover coverage.xml --compare-branch=BASE`.
- Others: the project's coverage command, filtered the same way.

No coverage tool → ask: "Install <tool> as a dev dependency" / "Use whole-package coverage for
this run" / "Stop". Record the choice in the phase summary; the check never passes silently.

Message check covers every commit on the branch, since fix cycles add commits:
`git log --pretty='%h %B' BASE..HEAD`. Each subject starts with one of `feat fix docs style
refactor perf test build ci chore revert` (optional `(scope)`, optional `!`, then `: `) and is
under 80 chars. No plan issue number or ID appears anywhere (`1.1`, `addr-opt-3.5`,
`P2-HYGIENE-1`, `Phase 2`, …); the ID lives only in the branch name. Name the offending `<sha>`
to the writer, who rewords it non-interactively.

Report one line, and loop until every check passes:

```text
Gate <issue>: coverage NN% ✓ · tests N ✓ · lint ✓ · committed ✓ · message ✓ → PASS
```

### Step 3: Review

Record `REVIEWED_SHA` = the worktree's `git rev-parse HEAD` and TaskCreate a review task. Spawn
`dev-team:code-reviewer` named `reviewer-<issue-id>-c<cycle>` with `REPO_ROOT`, worktree path,
branch, `BASE`, issue text and acceptance criteria, gate results (coverage, test count, lint),
`CODEX` and `CODEX_COMPANION` when set, and on a re-review the prior findings, the findings the
user deferred, and the previous `REVIEWED_SHA` as `last reviewed sha`. Wait for the report; it
ends with a verdict (APPROVE if and only if no Must Fix finding is open).

`Verdict: INCOMPLETE` means the reviewer couldn't spawn its sub-reviewers (nested subagents
unavailable). Only then do you spawn the Claude sub-reviewers it names yourself, in one message,
with the brief it returned (a perspective that ran on Codex is not re-run). Then spawn a fresh
reviewer with their reports to merge.

### Step 4: Verdict

**REQUEST CHANGES.** The reviewer's run is over; don't message it again (the one exception is
"Merge anyway" in step 5).

1. Show the findings:

   ```text
   Review <issue> (cycle N/3): REQUEST CHANGES
   Must Fix:   [SEC-001] src/api/users.ts:42 — SQL injection in search
   Should Fix: [CR-002] src/auth/token.ts:10 — inconsistent naming
   Consider:   n findings (not shown)
   ```

2. Ask: "Fix all findings" / "Fix Must Fix only" / "Abort this issue". Findings the user doesn't
   choose to fix are **deferred**: noted in the phase summary, passed to later reviewers as
   deferred, and never reopened.
3. SendMessage the approved findings to `writer-<issue-id>` verbatim (tag, `file:line`, defect,
   suggested fix) with: "Fix in the existing worktree, test-first: a failing test per bug, then
   the fix. Commit following the commit rules."
4. Re-run Step 2, then Step 3 with a fresh reviewer carrying the prior findings. Every re-review
   runs all three perspectives, scoped to the fix diff plus the prior findings.
5. After 3 cycles with Must Fix still open:

   ```text
   Review stalled: <issue> — 3 cycles, N Must Fix open: [list]
   ```

   Ask: "Continue fixing (reset count)" (default) / "Abort this issue" / "Merge anyway (breaks
   the review guarantee)". On "Merge anyway", ask the APPROVE merge question below, resume the
   last reviewer with the answer, and list the open Must Fix findings in the phase summary.

**APPROVE.** The reviewer has simplified, re-tested, and is waiting for you to ask the user:

1. Ask, showing branch, target `develop`, cycles, findings resolved, tests, and coverage: "Merge
   into develop and push to origin/develop" (only with an `origin` remote) / "Merge into develop
   locally (no push)" / "Review diff first" / "Do not merge". "Review diff first" → show
   `git diff --stat BASE...<branch>` and ask again. "Do not merge" → leave the worktree, mark the
   issue blocked, and move on.
2. Record `PRE_MERGE` = `git rev-parse develop`. SendMessage the same reviewer exactly
   `merge approved, push approved` or `merge approved, no push`. It rebases, fast-forwards,
   pushes only when told, and removes the worktree. Put any conflict, post-rebase test failure,
   or dirty main tree to the user; never resolve one silently.
3. Confirm the merge with `git log develop -1` and run the test suite on `develop`. On failure,
   ask: "Revert the merge locally" / "Revert and push the revert to origin/develop" (only if the
   merge was pushed) / "Keep it and create a fix issue". Revert with
   `git revert --no-edit PRE_MERGE..develop`; push only on the push option. Once a merge or
   revert is left unpushed, local `develop` is ahead of `origin/develop`, so use local `develop`
   as `BASE` for the rest of the run.
4. TaskUpdate the issue to completed and say `Merged <issue> → develop (<sha>) · N cycles · NN%`.
5. Move to the next issue at Step 1.

## Stage 4: Multi-stream (opt-in only)

Only when the user asked for parallel streams and the plan carries stream labels: one writer per
stream in its own worktree, review whichever finishes first, interleave fix cycles, and hold a
dependent issue until its blocker is merged into `develop`.

## Stage 5: Phase complete

1. Run the full test suite and lint on `develop`. On failure, create a fix issue and run it
   through the full loop.
2. Summary:

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

TaskUpdate anything left.

## Rules

**Review guarantee**

- No code reaches `develop` without Write → Gate → Review (CR + SEC + BUG) → Approve → Merge.
  A perspective counts only when its report came back, from Claude or Codex; a review missing
  one is re-run.
- Gate thresholds are fixed. "Merge anyway" exists only after 3 cycles and an explicit user
  override. Post-merge verification and the phase integration check always run.
- Nothing is pushed or reverted without an AskUserQuestion option that names it.

**Agent lifecycle**

- A code-writer owns one issue from Write through every fix: keep its `name` and SendMessage it
  each fix list.
- A code-reviewer's scope is one pass; each re-review is a fresh spawn with the prior findings.
  The only resume is the last reviewer, once, with the merge answer (after APPROVE or a "Merge
  anyway" override).
- Once an agent's scope has ended, don't message it; spawn a fresh one with the context it needs.
