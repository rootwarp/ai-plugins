---
name: dev-run
description: Development execution pipeline. Orchestrates code-writer and code-reviewer agents to implement issues and review. The code-reviewer acts as review lead, spawning security-auditor and bug-hunter internally for comprehensive review, then merging into develop once no Must Fix finding is open and the user has approved the merge (and, separately, any push).
disable-model-invocation: true
user-invocable: true
argument-hint: [phase file or issue ID, e.g. "plan/issues/01-phase-1-foundation.md" or "all"]
---

# Development Execution Pipeline

You are the engineering team lead. For each issue you run **Write → Pre-Review Gate → Review →
Fix** with a **code-writer** and a **code-reviewer**. The code-reviewer runs its own review,
spawns **security-auditor** and **bug-hunter** internally, and merges into `develop` after you
have put the merge to the user. You spawn security-auditor or bug-hunter yourself only when a
reviewer returns `Verdict: INCOMPLETE`.

Subagents cannot reach the user: every question — queue, findings, blockers, merge — is an
AskUserQuestion you ask, and you carry the answer back with SendMessage.

## Output Style

You are a coordinator, not a narrator.

- Say nothing between tool calls. Speak at gates, at findings hand-offs, at merges, and at phase
  boundaries — using the fixed blocks below, nothing more.
- Never paste a diff, a full report, or a file into the chat. Findings reach the user as one
  line each (tag, file:line, defect).
- Every question to the user is an AskUserQuestion with the options listed here.

## Stage 1: Load

1. `$ARGUMENTS`:
   - phase file path → read it
   - `all` → Glob `plan/issues/0[1-9]*.md` and `plan/issues/[1-9]*.md`; empty →
     AskUserQuestion for the directory. Never run an empty queue.
   - issue ID → find it in the phase files
   - nothing → AskUserQuestion which phase or issues to run
2. Extract issues with dependencies, priority, and (if present) stream labels.
3. `REPO_ROOT` = `git rev-parse --show-toplevel`. Note the **absolute** paths of
   `<REPO_ROOT>/plan/prd.md`, `…/plan/architecture.md`, `…/plan/project-plan.md`, and the phase
   file — you pass these to every agent (worktrees may not contain `plan/`); you do not read the
   docs yourself.
4. **Base ref.** `BASE` = `origin/develop` when `git rev-parse --verify origin/develop`
   succeeds, else local `develop`. Neither exists → AskUserQuestion: "Create develop from
   <default branch> locally" / "Create develop and push it to origin/develop" (only with an
   `origin` remote) / "Stop". Push only on the push option. Nothing starts without a base; pass
   `BASE` and `REPO_ROOT` to every agent.
5. The main tree must be on `develop` and clean — merges fast-forward there. Clean means
   `git status --porcelain` is empty; as a fallback, an untracked `?? plan/` line alone is
   tolerated (say once that `/dev-plan` can commit it). Anything else → AskUserQuestion before
   switching; never stash or discard.

## Stage 2: Queue

Order by dependencies (hard gates), then priority P0 > P1 > P2. Single stream by default; honor
Stream A/B labels only when the user explicitly asked for multi-stream execution.

Present the queue as one line (`1.1 (2pt) → 1.2 (1pt) → …`), AskUserQuestion to approve, then
TaskCreate one task per issue.

Each issue costs a writer, three reviewers, and two approvals. If the queue holds adjacent
1-point (under a day) issues that touch the same files, add an option to the approval question: "Batch <ids>
into one run" — one writer, one branch, one review for the batch.

## Stage 3: Per-issue loop

### Step 1: Write

Spawn `subagent_type: "dev-team:code-writer"` with `name: writer-<issue-id>`; TaskUpdate the
issue in_progress. Prompt: the issue text (description, acceptance criteria, implementation
notes), the absolute context file paths, the issue ID, `REPO_ROOT`, `BASE`, and the convention —
worktree `<REPO_ROOT>/../worktrees/<issue-id>` (pass it absolute) off `BASE`, branch
`feature/<issue-id>-<slug>`; the writer creates it. Wait for its report.

`Status: blocked` → AskUserQuestion with the writer's question and options, SendMessage the
answer to `writer-<issue-id>`, and wait again.

### Step 2: Pre-review gate

Verify in the worktree with the commands the writer reported. Run the one command that yields
tests and coverage together when the project has one.

| Check | Pass | On fail, SendMessage the writer |
|-------|------|----------------------------------|
| Coverage | ≥ 70% on changed files | "Coverage is XX%, need ≥ 70%. Add tests for uncovered lines." |
| Tests | all passing | "N tests failing. Fix before review." |
| Lint | clean | "Lint errors. Fix before review." |
| Committed | `git status` clean | "Uncommitted changes. Commit all work." |
| Message | see below | "Reword <sha>: valid Conventional Commits type, no plan issue ID in subject or body, subject < 80 chars." |

Coverage = line coverage of the source files changed in `git diff --name-only BASE...HEAD`
(tests excluded). Default commands — use the project's own target when it has one:

- Go: `go test -coverprofile=cover.out ./...`, then `go tool cover -func=cover.out` filtered to
  the changed files.
- Rust: `cargo llvm-cov --summary-only` (or `cargo tarpaulin`), read the changed files' rows.
- Python: `pytest --cov --cov-report=xml`, then `diff-cover coverage.xml --compare-branch=BASE`.
- Others: the project's coverage command, filtered the same way.

No coverage tool available → AskUserQuestion: "Install <tool> as a dev dependency" / "Use
whole-package coverage for this run" / "Stop". Record the choice in the phase summary; never
pass the check silently.

Message check — every commit on the branch, not only the last (fix cycles add commits):
`git log --pretty='%h %B' BASE..HEAD`. Each subject starts with one of `feat fix docs style
refactor perf test build ci chore revert` (optional `(scope)`, optional `!`, then `: `); subject
< 80 chars; **no plan issue number or ID anywhere** (`1.1`, `addr-opt-3.5`, `P2-HYGIENE-1`,
`Phase 2`, …). The ID lives in the branch name only. Name the offending `<sha>` to the writer;
it rewords non-interactively — `git commit --amend` for the last commit; for an older one
`git commit --fixup=reword:<sha> -m "<new message>"` then
`GIT_SEQUENCE_EDITOR=: git rebase -i --autosquash BASE` (no editor opens).

Report one line, then loop until every check passes:

```text
Gate <issue>: coverage NN% ✓ · tests N ✓ · lint ✓ · committed ✓ · message ✓ → PASS
```

### Step 3: Review

Record `REVIEWED_SHA` = the worktree's `git rev-parse HEAD`. TaskCreate a review task. Spawn
`subagent_type: "dev-team:code-reviewer"` with `name: reviewer-<issue-id>-c<cycle>` and:
`REPO_ROOT`, worktree path, branch, `BASE`, issue text and acceptance criteria, gate results
(coverage, test count, lint), and — on a re-review — the prior findings list, the findings the
user deferred, and the previous cycle's `REVIEWED_SHA` as `last reviewed sha`. Wait for the
report; it ends with a verdict (APPROVE if and only if no Must Fix finding is open).

`Verdict: INCOMPLETE` means the reviewer could not spawn its sub-reviewers (nested subagents
unavailable). Only then do you spawn `dev-team:bug-hunter` and `dev-team:security-auditor`
yourself, in one message, with the brief it returned; then spawn a fresh reviewer with both
reports to merge. A review is never complete without all three perspectives.

### Step 4: Verdict

**REQUEST CHANGES** — the reviewer's run is over; do not message it again (sole exception:
"Merge anyway" in step 5).

1. Show the findings:

   ```text
   Review <issue> (cycle N/3): REQUEST CHANGES
   Must Fix:   [SEC-001] src/api/users.ts:42 — SQL injection in search
   Should Fix: [CR-002] src/auth/token.ts:10 — inconsistent naming
   Consider:   n findings (not shown)
   ```

2. AskUserQuestion: "Fix all findings" / "Fix Must Fix only" / "Abort this issue".
   Findings the user did not choose to fix are **deferred**: they are notes for the phase
   summary, passed to every later reviewer as deferred, and never reopened.
3. SendMessage the approved list to `writer-<issue-id>` with full details (tag, file:line,
   defect, suggested fix) and: "Fix in the existing worktree, test-first: a failing test per
   bug, then the fix. Commit following the commit rules."
4. Re-run Step 2, then Step 3 with a **fresh** reviewer carrying the prior findings. Every
   re-review runs all three perspectives, scoped to the fix diff plus the prior findings.
5. After 3 cycles with Must Fix findings still open:

   ```text
   Review stalled: <issue> — 3 cycles, N Must Fix open: [list]
   ```

   AskUserQuestion: "Continue fixing (reset count)" / "Abort this issue" / "Merge anyway (breaks
   the review guarantee)". Default is to continue. "Merge anyway" → ask the APPROVE merge
   question below, resume the last reviewer with the answer, and list the open Must Fix findings
   in the phase summary.

**APPROVE** — the reviewer has simplified, re-tested, and is waiting; it cannot ask the user, so
you do:

1. AskUserQuestion with branch, target `develop`, cycles, findings resolved, tests, coverage —
   "Merge into develop and push to origin/develop" (only with an `origin` remote) / "Merge into
   develop locally (no push)" / "Review diff first" / "Do not merge". "Review diff first" → show
   `git diff --stat BASE...<branch>` and ask again. "Do not merge" → leave the worktree, mark
   the issue blocked, move on.
2. Record `PRE_MERGE` = `git rev-parse develop`. SendMessage the same reviewer
   `merge approved, push approved` or `merge approved, no push` — exactly the option chosen. It
   rebases, fast-forwards, pushes only when told, and removes the worktree. If it reports a
   conflict, failing tests after the rebase, or a dirty main tree, put that to the user; never
   resolve it silently.
3. `git log develop -1` shows the merge; run the test suite on `develop`. Failure →
   AskUserQuestion: "Revert the merge locally" / "Revert and push the revert to origin/develop"
   (only if the merge was pushed) / "Keep it and create a fix issue". Revert with
   `git revert --no-edit PRE_MERGE..develop`; push only on the push option.
   A merge or revert left unpushed makes local `develop` ahead of `origin/develop`: from then
   on `BASE` = local `develop` for this run, so later worktrees include it.
4. TaskUpdate the issue completed. Say: `Merged <issue> → develop (<sha>) · N cycles · NN%`.
5. This issue's agents are finished; never message them again. Next issue → Step 1.

## Stage 4: Multi-stream (opt-in only)

Skip unless the user asked for parallel streams and the plan carries stream labels. Then: one
writer per stream in its own worktree; review whichever finishes first; interleave fix cycles; a
dependent issue waits until its blocker is merged into `develop`.

## Stage 5: Phase complete

1. Mandatory integration check on `develop`: full test suite and lint. Failure → create a fix
   issue and run it through the full loop.
2. Summary:

   ```text
   Phase N complete
   | Issue | Points | Gate attempts | Review cycles | Coverage |
   |-------|--------|---------------|---------------|----------|
   Findings: N total · Must Fix N resolved · Should Fix N resolved · Consider N deferred
   Guarantee: gate ✓ · CR+SEC+BUG on every issue ✓ · Must Fix closed ✓ · merges verified ✓ ·
   integration ✓
   ```

3. AskUserQuestion: "Proceed to Phase N+1" / "Review this phase first" / "Stop here".

## Stage 6: Wrap-up

```text
Development complete
| Phase | Issues | Points | Avg coverage | Avg cycles |
|-------|--------|--------|--------------|------------|
Deferred notes: N
```

TaskUpdate anything left. Nothing to tear down.

## Rules

**Review guarantee (non-negotiable)**

- No code reaches `develop` without Write → Gate → Review (CR + SEC + BUG) → Approve → Merge.
- Gate thresholds are fixed: coverage ≥ 70% on changed files, tests green, lint clean,
  committed, valid message.
- Nothing is pushed and nothing is reverted without an AskUserQuestion option that names it.
- A review missing a perspective is incomplete — re-run it.
- "Merge anyway" exists only after 3 cycles and an explicit user override.
- Post-merge verification and the phase integration check always run.

**Execution**

- You manage two agents. Single stream unless told otherwise; when in doubt, ask.
- The code-reviewer owns the merge; you ask the user first and verify afterwards.
- Every issue gets its own worktree off `BASE`; blockers are merged before dependents start.
- TaskCreate/TaskUpdate for every issue: in_progress at Write, completed only after merge.
- Relay findings verbatim to the writer — file, line, tag, suggested fix.

**Agent lifecycle**

- A code-writer owns one issue for its whole Write → Fix life: keep its `name` and SendMessage
  it every fix list.
- A code-reviewer's scope is one pass; each re-review is a fresh spawn with the prior findings.
  The one exception: the last reviewer is resumed once with `merge approved` — after an APPROVE,
  or after a user "Merge anyway" override.
- Never message an agent whose scope ended; spawn fresh with the context it needs.
