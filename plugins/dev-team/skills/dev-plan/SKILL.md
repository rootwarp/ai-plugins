---
name: dev-plan
description: Full development planning pipeline. Drives an interactive, gated flow from a project idea to sprint-ready issues — keeping the PRD interview and every LGTM gate in the main loop — while orchestrating the heavy research, architecture, and issue-estimation fan-out through background Workflow runs that reuse the dev-team planning agents. Sizes the pipeline to the work (feature vs project) and sizes issues to the plan's tasks, with no cap on their number. Optionally publishes the approved plan to GitHub as one phase issue per phase with every issue linked as a sub-issue.
disable-model-invocation: true
user-invocable: true
argument-hint: [project description or requirements] [--github]
---

# Development Planning Pipeline

You are the team lead. You take a project idea through eight stages — five LGTM gates, plus two
more if GitHub publishing is on — to sprint-ready issues on disk. Stage 7 (GitHub issues) is
optional; every other stage is mandatory.

The pipeline is **hybrid**: interactive work — the PRD interview and every LGTM gate — runs in
the main loop (you); heavy fan-out — research, architecture, issue estimation, GitHub rendering —
runs as background **Workflow** runs over the dev-team agents. No subagent and no workflow can
reach the user: every question is yours to ask.

| Agent | `agentType` | Role | Runs via |
|-------|-------------|------|----------|
| prd-writer | `dev-team:prd-writer` | write the PRD from your interview | Agent tool (Stage 2) |
| researcher | `dev-team:researcher` | investigate angles, verify claims | Workflow (Stage 3) |
| software-architect | `dev-team:software-architect` | candidate designs, judge, final | Workflow (Stage 4) |
| project-planner | `dev-team:project-planner` | phases, milestones, dependencies | Agent tool (Stage 5) |
| issue-estimator | `dev-team:issue-estimator` | 2-3 day issues; GitHub bodies | Workflow (Stages 6, 7) |

## Output Style

You are a coordinator, not a narrator.

- Say nothing between tool calls. Speak at gates and at wrap-up, within the caps below.
- Gate summaries lead with the numbers, then only what the user must approve — decisions,
  recommendations, recorded assumptions. Never paste a document into the chat.
- Caps: PRD gate ≤ 12 lines; research, architecture, and plan gates ≤ 15; estimation gate = the
  phase table + ≤ 10 lines; GitHub dry-run = the sync table; wrap-up = the Stage 8 block.
- Each gate ends with one question that names `LGTM` as the accept signal.

## Orchestration Rules

### Mechanisms

- **Main loop** for anything that must talk to the user: the PRD interview, every gate, every
  AskUserQuestion. Subagents cannot ask; one that needs an answer returns a `Questions` block —
  ask the user, then resume it (SendMessage) or re-spawn it with the answers.
- **Agent tool** for single sequential agents: the PRD (Stage 2), the project plan (Stage 5), and
  single-file revisions.
- **Workflow tool** for non-interactive fan-out: Stages 3, 4, 6, and the Stage 7 render step.
- **Main loop, sequential, after explicit approval** for every side effect outside the working
  tree — all `gh` writes in Stage 7. Never put a network mutation in a workflow script.

### Workflow stages: compute → launch → wait → read → gate

1. **Compute `args`** in the main loop from the approved artifacts (each stage says what to
   build). Pass it as a real JSON object, never a JSON-encoded string. Every script still starts
   with `const input = typeof args === 'string' ? JSON.parse(args) : args` so either form works;
   keep that line.
2. **Launch** with the embedded script and `args`.
3. **Wait** for the completion notification. Do not start the gate or the next workflow early.
4. **Read the artifacts from disk** — the workflow's return value is only a thin summary.
5. **Gate.** Advance only on `LGTM`.

Every workflow prompt tells the agent it runs unattended: never ask the user, record assumptions
for the gate. House style and line budgets live in each agent's Output Style section; prompts
only name the budget. Keep both when you adapt a prompt.

### Gates

- Never skip a gate. Only an exact `LGTM` (case-insensitive) advances. "ok", "looks fine",
  silence, or partial approval are revision requests — address them and ask again.
- Stage 7 may be declined as a whole; its internal gates cannot be skipped once opted in.

### Revisions — the smallest change that satisfies the request

- Fan-out shape changed (new research angle, new architecture lens, a phase split) → re-launch
  that workflow with revised `args`, pointed at the existing artifacts so it builds on them.
- Localized fix (a name, a wording, one issue) → spawn one agent of the producing type via the
  Agent tool with the file path and the exact change; it edits in place.
- PRD and project plan → spawn a fresh agent of that type with the file and the changes.
- Published GitHub issues → edit with `gh issue edit` / `gh api --method PATCH`; re-running
  Stage 7 only creates what `plan/issues/github-sync.md` lacks.
- Revisions always spawn fresh; never resume a completed workflow. Resume an agent (SendMessage)
  only to answer its `Questions` block.

### General

- Every stage writes files under the plan directory; downstream stages read them by path, and
  `args` always carry the upstream paths.
- Workflows reuse the registered agents by `agentType`; you never redefine them.
- Track progress with an eight-line checklist; there is no team to set up or tear down.

## Stages

### Stage 1: Kickoff (main loop)

1. Read `$ARGUMENTS`. If too vague to brief a PRD, ask once about the problem, the users, and
   known constraints (stack, timeline, team).
2. **Size the work** — it sets every downstream fan-out (table below). `feature`: a change to an
   existing codebase, roughly ≤ 3 weeks of work. `project`: greenfield, or a change that adds or
   reshapes several modules. Glob the repo to decide; unclear → AskUserQuestion. State the scope
   in one line and pass it in every agent prompt.
3. Output directory: `plan/` (workflows write `plan/research/` and `plan/issues/`). Change it
   only on request.
4. Record whether `--github` was passed or GitHub issues were asked for; act on it in Stage 7
   only.

| | `feature` | `project` |
|---|-----------|-----------|
| PRD | ≤ 60 lines; P0 = what the feature needs | ≤ 120 lines |
| Research | 1-2 angles, only open technical questions | 2-5 angles |
| Architecture | one design, `fit-existing` lens, delta only, ≤ 120 lines | three lenses + judge |
| Plan | as few phases as the work needs — usually 1-2 | as few phases as the work needs |
| Issues | one per plan task by default; split or merge on size — no count cap | same |

Oversized plans come from running the `project` pipeline on a `feature`: every extra research
angle, module, and phase turns into issues, and every issue costs `/dev-run` a writer, three
reviewers, and two approvals. When in doubt, pick `feature`. Nothing here limits how many issues
a plan may have — big work gets many issues; the rules only stop small work from being diced.

### Stage 2: PRD (main-loop interview, then Agent tool)

1. **Interview** the user yourself with AskUserQuestion, batches of 1-4, only what `$ARGUMENTS`
   left open: problem and status quo · users · must-have vs nice-to-have · hard constraints ·
   definition of done and metrics · integrations and non-functional needs. Stop as soon as the P0
   requirements and success metrics are clear; the rest becomes Open Questions.
2. Spawn `subagent_type: "dev-team:prd-writer"` with the scope, the constraints, and the
   interview answers verbatim; it writes `plan/prd.md` with P0/P1/P2 requirements within the
   scope's line budget. If it returns a `Questions` block instead of a file, ask them and resume
   it with the answers.
3. **Gate — PRD** (≤ 12 lines: problem, users, metrics, P0/P1/P2 highlights): "Does this PRD
   capture your requirements? Reply `LGTM` to proceed, or describe the changes."

### Stage 3: Research (Workflow — parallel angles, adversarial verification)

**`args`:** read `plan/prd.md` and pick the angles that matter — 1-2 for `feature` scope (only
questions the codebase and the PRD cannot answer), 2-5 for `project` — each `{ key, prompt }` —
`key` a filename-safe slug, `prompt` naming the researcher template and the concrete question:
A technology comparison · B deep dive · C landscape · D best practices · E feasibility. Then
`prdPath: 'plan/prd.md'`, `researchDir: 'plan/research'`.

```javascript
export const meta = {
  name: 'dev-plan-research',
  description: 'Parallel multi-angle research with adversarial verification, synthesized into plan/research/',
  phases: [
    { title: 'Investigate', detail: 'one researcher per angle' },
    { title: 'Verify', detail: 'adversarially check key claims' },
    { title: 'Synthesize', detail: 'merge into 00-overview.md' },
  ],
}

const FINDINGS = {
  type: 'object',
  properties: {
    angle: { type: 'string' },
    summary: { type: 'string' },
    claims: {
      type: 'array',
      items: {
        type: 'object',
        properties: { claim: { type: 'string' }, source: { type: 'string' } },
        required: ['claim'],
      },
    },
    assumptions: { type: 'array', items: { type: 'string' } },
    docPath: { type: 'string' },
  },
  required: ['angle', 'summary', 'claims'],
}

const VERDICT = {
  type: 'object',
  properties: {
    angle: { type: 'string' },
    verifiedClaims: { type: 'array', items: { type: 'string' } },
    refutedClaims: { type: 'array', items: { type: 'string' } },
    notes: { type: 'string' },
  },
  required: ['angle'],
}

const input = typeof args === 'string' ? JSON.parse(args) : args
const { angles, prdPath: prd, researchDir: outDir } = input
const UNATTENDED = 'Unattended run: never ask the user; record assumptions under "Assumptions". '

const results = await pipeline(
  angles,
  (a) => agent(
    'Research ONE angle for a development plan. PRD: ' + prd + '. Angle "' + a.key + '": ' +
    a.prompt + ' Use WebSearch/WebFetch, cite claims inline, end with Sources. Write ' +
    outDir + '/' + a.key + '.md with the matching researcher template (A-E), your Output Style, ' +
    'under 100 lines. ' + UNATTENDED +
    'Return angle, summary, key claims with sources, assumptions, docPath.',
    { agentType: 'dev-team:researcher', label: 'research:' + a.key, phase: 'Investigate', schema: FINDINGS }
  ),
  (r) => r ? agent(
    'Adversarially verify research claims against primary sources. Flag unsupported, outdated, ' +
    'overstated, or contradicted claims; treat single-source significant claims as low-confidence. ' +
    'Angle "' + r.angle + '". Claims: ' + JSON.stringify(r.claims) + '. ' + UNATTENDED +
    'Return angle, verifiedClaims, refutedClaims, and notes (a few lines, no restating).',
    { agentType: 'dev-team:researcher', label: 'verify:' + r.angle, phase: 'Verify', schema: VERDICT }
  ).then((v) => ({ ...r, verdict: v })) : null,
)

const clean = results.filter(Boolean)
const verification = clean.map((r) => ({
  angle: r.angle,
  refuted: r.verdict ? r.verdict.refutedClaims : null,
  notes: r.verdict ? r.verdict.notes : null,
}))

const summary = await agent(
  'Consolidate a multi-angle investigation. Read PRD ' + prd + ' and every doc in ' + outDir +
  '. Write ' + outDir + '/00-overview.md: lead with recommendations (one bullet + one-line ' +
  'reason each), reconcile conflicts, drop or caveat refuted claims: ' +
  JSON.stringify(verification) + '. List consolidated assumptions and link each angle doc. ' +
  'No background, no recap, under 60 lines. ' + UNATTENDED +
  'Return a short summary and open questions.',
  { label: 'synthesize', phase: 'Synthesize' }
)

return { angles: clean.map((r) => r.angle), overview: outDir + '/00-overview.md', summary }
```

**Gate — Research** (≤ 15 lines from `plan/research/00-overview.md`: recommendations,
reconciled conflicts, assumptions): "Does this research cover what the PRD needs? Reply `LGTM`
to proceed, or name areas to investigate further."

### Stage 4: Architecture (Workflow — three candidates, one judge; one design for a feature)

**`args`:** `priorities` = lenses `{ key, prompt }`. `feature` scope — exactly one:

- `fit-existing` — "Extend the existing architecture: reuse its modules, patterns, and stack; add
  a module only when no existing one can own the behaviour; document only the delta."

With a single lens the script writes `plan/architecture.md` directly — no candidates, no judge.
`project` scope — three fixed lenses unless the user's feedback changes them:

- `modular-monolith` — "Optimize for simplicity: the fewest moving parts that satisfy the PRD."
- `scale-first` — "Optimize for scale: independent scaling seams and async boundaries from day
  one."
- `extraction-first` — "Optimize for service extraction: each module becomes a service with no
  rewrite."

Plus `prdPath`, `researchDir`, `architecturePath: 'plan/architecture.md'`, and `scope`.

```javascript
export const meta = {
  name: 'dev-plan-architecture',
  description: 'Three candidate designs, one comparative judge, synthesized into plan/architecture.md',
  phases: [
    { title: 'Design', detail: 'one candidate per priority lens' },
    { title: 'Judge', detail: 'rank all candidates against the quality checklist' },
    { title: 'Synthesize', detail: 'merge the winner + best ideas into plan/architecture.md' },
  ],
}

const CANDIDATE = {
  type: 'object',
  properties: {
    priority: { type: 'string' },
    path: { type: 'string' },
    summary: { type: 'string' },
    keyDecisions: { type: 'array', items: { type: 'string' } },
  },
  required: ['priority', 'path', 'summary'],
}

const RANKING = {
  type: 'object',
  properties: {
    ranked: {
      type: 'array',
      items: {
        type: 'object',
        properties: {
          candidate: { type: 'string' },
          total: { type: 'number' },
          keep: { type: 'array', items: { type: 'string' } },
          weak: { type: 'array', items: { type: 'string' } },
        },
        required: ['candidate', 'total'],
      },
    },
  },
  required: ['ranked'],
}

const input = typeof args === 'string' ? JSON.parse(args) : args
const { priorities, prdPath: prd, researchDir, architecturePath: outPath } = input
const candBase = outPath.replace(/\.md$/, '')   // candidates: <candBase>.<key>.candidate.md
const UNATTENDED = 'Unattended run: never ask the user; record assumptions under Open Questions. '

if (priorities.length === 1) {
  const only = priorities[0]
  const direct = await agent(
    'Design the architecture for a ' + (input.scope || 'feature') + '-scope change. PRD: ' + prd +
    '; research: ' + researchDir + '. Read the existing codebase first. Lens "' + only.key +
    '": ' + only.prompt + ' Follow the software-architect template and your Output Style, no ' +
    'circular dependencies, every new module justified, under 120 lines. Write ' + outPath +
    '. ' + UNATTENDED + 'Return a summary: touched modules, new modules, key decisions.',
    { agentType: 'dev-team:software-architect', label: 'design:' + only.key, phase: 'Design' }
  )
  return { winner: only.key, architecture: outPath, summary: direct }
}

const candidates = (await parallel(priorities.map((p) => () => agent(
  'Design a candidate architecture. PRD: ' + prd + '; research: ' + researchDir + '. ' +
  'Optimize for "' + p.key + '": ' + p.prompt + ' Follow the software-architect template and ' +
  'your Output Style, no circular dependencies, under 200 lines. Write it to ' +
  candBase + '.' + p.key + '.candidate.md. ' + UNATTENDED +
  'Return priority, path, summary, keyDecisions.',
  { agentType: 'dev-team:software-architect', label: 'design:' + p.key, phase: 'Design', schema: CANDIDATE }
)))).filter(Boolean)

const judged = await agent(
  'Judge candidate architectures against the quality checklist: no circular dependencies, ' +
  'single responsibility per module, clear data ownership, testable in isolation, clear ' +
  'extraction path, justified module count, simplicity. Candidates: ' +
  JSON.stringify(candidates.map((c) => ({ priority: c.priority, path: c.path }))) +
  '. Read all of them. ' + UNATTENDED + 'Return them ranked best first, each with a 0-100 ' +
  'total, the ideas worth keeping (keep) and the weakest points (weak), one line each.',
  { agentType: 'dev-team:software-architect', label: 'judge', phase: 'Judge', schema: RANKING }
)

const ranked = judged && judged.ranked && judged.ranked.length
  ? judged.ranked
  : candidates.map((c) => ({ candidate: c.priority, total: 0 }))

const result = await agent(
  'Produce the FINAL architecture. Ranked candidates: ' + JSON.stringify(ranked) + '; files: ' +
  JSON.stringify(candidates.map((c) => c.path)) + '; PRD: ' + prd + '. Start from the ' +
  'top-ranked design and graft in the "keep" ideas from the others. No circular dependencies; ' +
  'run the quality checklist. Write ' + outPath + ' per the software-architect template and ' +
  'your Output Style, under 250 lines; do not narrate the judging. ' + UNATTENDED +
  'Return a summary and which ideas came from which candidate.',
  { agentType: 'dev-team:software-architect', label: 'synthesize', phase: 'Synthesize' }
)

return { winner: ranked[0] ? ranked[0].candidate : null, architecture: outPath, summary: result }
```

**Gate — Architecture** (≤ 15 lines from `plan/architecture.md`: winning lens, module table —
new vs reused modules for a `feature` — merged decisions, assumptions): "Does this architecture
align with your vision? Reply `LGTM` to proceed, or list modules to add, remove, or restructure."

### Stage 5: Project Plan (Agent tool)

Phasing is sequential reasoning; one agent, no workflow.

1. Spawn `subagent_type: "dev-team:project-planner"`: the scope (Stage 1 table); read `plan/prd.md`, `plan/research/`, `plan/architecture.md`; write
   `plan/project-plan.md` with `## Phase N: [Name]` headings and `- [ ] Task N.M` lines;
   issue-sized tasks (2-3 days, vertical slices, no test/docs/config-only tasks); P2 under
   `## Later`; budget ≤ 120 lines.
2. **Gate — Plan** (≤ 15 lines: one line per phase with its task count, the total task count —
   roughly the number of issues to expect — plus dependency highlights): "Does the
   phasing make sense? Reply `LGTM` to proceed, or list phases to reorder, split, or merge."

### Stage 6: Issues (Workflow — pipeline per phase)

**`args`:** `phases` = ordered `{ num, name, taskCount }` from the plan's `## Phase N: [Name]`
headings (stop if empty), `taskCount` = the `- [ ] Task N.M` lines under that heading;
`docs = { prd, researchDir, architecture, projectPlan }`; `issuesDir: 'plan/issues'`. `taskCount`
is a reference point for the estimator and the gate, never a limit: issue count follows size.

```javascript
export const meta = {
  name: 'dev-plan-issues',
  description: 'Estimate issues per phase from the plan tasks, check sizing both ways, revise, then write a summary',
  phases: [
    { title: 'Estimate', detail: 'one estimator per phase, one issue per plan task by default' },
    { title: 'Check', detail: 'adversarial review: oversized, undersized, dependencies' },
    { title: 'Revise', detail: 'apply the check findings in place (only when the check fails)' },
    { title: 'Summary', detail: 'write 00-summary.md across all phases' },
  ],
}

const PHASE_RESULT = {
  type: 'object',
  properties: {
    phase: { type: 'string' },
    file: { type: 'string' },
    issueCount: { type: 'number' },
    totalPoints: { type: 'number' },
  },
  required: ['phase', 'file'],
}

const CHECK = {
  type: 'object',
  properties: {
    phase: { type: 'string' },
    oversizedIssues: { type: 'array', items: { type: 'string' } },
    mergeCandidates: { type: 'array', items: { type: 'string' } },
    missingDeps: { type: 'array', items: { type: 'string' } },
    ok: { type: 'boolean' },
  },
  required: ['phase', 'ok'],
}

const input = typeof args === 'string' ? JSON.parse(args) : args
const { phases, docs, issuesDir: outDir } = input
const UNATTENDED = 'Unattended run: never ask the user; record assumptions in the phase overview. '
const slug = (s) => s.toLowerCase().replace(/[^a-z0-9]+/g, '-').replace(/^-|-$/g, '')
const phaseFile = (p) =>
  outDir + '/' + String(p.num).padStart(2, '0') + '-phase-' + p.num + '-' + slug(p.name) + '.md'

const perPhase = await pipeline(
  phases,
  (p) => agent(
    'Estimate ONE phase. Inputs: PRD ' + docs.prd + ', architecture ' + docs.architecture +
    ', project plan ' + docs.projectPlan + ', research ' + docs.researchDir + '. Break Phase ' +
    p.num + ' (' + p.name + ') into 2-3 day issues per the issue-estimator phase template, ' +
    'Issue Sizing rules, and Output Style: one issue per plan task by default (the plan ' +
    'lists ' + (p.taskCount || 'an unknown number of') + ' tasks) — split or merge only on ' +
    'size, and there is no limit on the number of issues; tests, ' +
    'docs, config, and migrations ship inside the issue that needs them; points 1/2/3/5, ' +
    'target 2-3 (two to three days), split only over 5; single-stream (no streams, ' +
    'ownership maps, or scaffold issues); each issue at most 25 lines with concrete files ' +
    'and functions and checkbox ' +
    'acceptance criteria. Write ' + phaseFile(p) + '. ' + UNATTENDED +
    'Return phase, file, issueCount, totalPoints.',
    { agentType: 'dev-team:issue-estimator', label: 'estimate:p' + p.num, phase: 'Estimate', schema: PHASE_RESULT }
  ),
  (r, p) => r ? agent(
    'Adversarially review the issue breakdown for Phase ' + p.num + ' (' + p.name + ') at ' +
    r.file + '. Review only; write nothing. Flag both directions: oversizedIssues (over 5 ' +
    'points, or a 5 with no stated reason) and mergeCandidates (issues under about a day, ' +
    'issues touching the same files, standalone test/docs/config/refactor issues, anything ' +
    'the plan or codebase does ' +
    'not call for — name the issue to merge into). Judge each issue on its size and ' +
    'cohesion, never on how many issues the phase has. Also flag missing or implicit ' +
    'dependencies and untestable acceptance criteria. ok is true only when every list is ' +
    'empty. ' + UNATTENDED + 'Return phase, oversizedIssues, mergeCandidates, missingDeps, ok.',
    { agentType: 'dev-team:issue-estimator', label: 'check:p' + p.num, phase: 'Check', schema: CHECK }
  ).then((c) => ({ ...r, check: c, tasks: p.taskCount || null })) : null,
  (r, p) => (r && r.check && !r.check.ok) ? agent(
    'Revise the Phase ' + p.num + ' issue file ' + r.file + ' in place to resolve these review ' +
    'findings: ' + JSON.stringify(r.check) + '. Apply each merge and split named there; ' +
    'renumber issues and fix Blocked by / Blocks references and both tables. Change nothing ' +
    'else. ' + UNATTENDED + 'Return phase, file, issueCount, totalPoints.',
    { agentType: 'dev-team:issue-estimator', label: 'revise:p' + p.num, phase: 'Revise', schema: PHASE_RESULT }
  ).then((v) => (v ? { ...r, ...v, revised: true } : r)) : r,
)

const clean = perPhase.filter(Boolean)

const summary = await agent(
  'Write the cross-phase summary ' + outDir + '/00-summary.md from the phase files ' +
  JSON.stringify(clean.map((r) => r.file)) + ' per the issue-estimator summary template: ' +
  'approach, phase table with plan tasks, issue counts, and points, single-stream execution ' +
  'plan, dependency map, ' +
  'risk flags; relative links; tables and bullets only, under 80 lines. ' + UNATTENDED +
  'Return total issues, total points, risk flags.',
  { label: 'summary', phase: 'Summary' }
)

return {
  phases: clean.length,
  summary,
  sizing: clean.map((r) => ({
    phase: r.phase, tasks: r.tasks, issues: r.issueCount, revised: !!r.revised, check: r.check,
  })),
}
```

Every stage callback receives `(previousResult, originalPhase)` — that is how Check and Revise
know `p.num`. `sizing[].check` is the pre-revision review; re-read the phase files for the final
counts.

**Gate — Estimation** (phase table from `plan/issues/00-summary.md` with plan tasks vs issues per
phase + ≤ 10 lines: execution plan, what the check merged or split, and why any phase has
notably more issues than tasks): "Are the estimates reasonable? Reply `LGTM` to proceed, or list
issues to split, merge, or reprioritize."

### Stage 7: GitHub Issues (optional — workflow render, main-loop publish)

One phase issue per phase; every planned issue a sub-issue of it. Opt-in first, then two LGTM
gates: a dry-run of exactly what will be created, and a sync check after publishing.

#### 7.1 Opt in

AskUserQuestion: "Create these issues on GitHub? One phase issue per phase, each issue as a
sub-issue." — "Yes, create GitHub issues" / "Skip — keep the plan local". If `--github` was
passed, confirm rather than re-ask; the dry-run gate still applies. Skip → Stage 8.

#### 7.2 Preflight (Bash)

Any failure: report it and offer to skip the stage.

1. `gh --version` and `gh auth status` — missing or unauthenticated → tell the user to install
   or `gh auth login`; offer to skip.
2. `gh repo view --json nameWithOwner -q .nameWithOwner` — no remote → AskUserQuestion for
   `owner/repo`. Never guess a repository.
3. Read `plan/issues/github-sync.md` if present; everything listed already exists and is
   skipped.
4. No sync file → `gh issue list --state all --limit 100 --json number,title`; match phase and
   issue titles; report collisions at the dry-run gate.
5. `gh label list --limit 200 --json name -q '.[].name'`; create only the missing labels you
   will use (`phase`, `P0`/`P1`/`P2`, `feature`/`bug`/`chore`/`spike`), e.g.
   `gh label create phase --color 0E8A16 --description "Phase-level tracking issue"`.
   `gh issue create --label` fails on unknown labels.

#### 7.3 Render (Workflow)

**`args`:** `phases` = `[{ num, name, file }]` from Glob `plan/issues/0[1-9]*.md` and
`plan/issues/[1-9]*.md` in phase order; `githubDir: 'plan/issues/github'`. No repository name
is needed — the render does no network work.

```javascript
export const meta = {
  name: 'dev-plan-github-render',
  description: 'Render GitHub-ready issue bodies and a per-phase manifest from the approved plan',
  phases: [{ title: 'Render', detail: 'one agent per phase file' }],
}

const MANIFEST = {
  type: 'object',
  properties: {
    phase: { type: 'number' },
    manifestPath: { type: 'string' },
    parent: {
      type: 'object',
      properties: {
        title: { type: 'string' },
        bodyPath: { type: 'string' },
        labels: { type: 'array', items: { type: 'string' } },
      },
      required: ['title', 'bodyPath'],
    },
    children: {
      type: 'array',
      items: {
        type: 'object',
        properties: {
          issueId: { type: 'string' },
          title: { type: 'string' },
          bodyPath: { type: 'string' },
          labels: { type: 'array', items: { type: 'string' } },
          points: { type: 'number' },
          blockedBy: { type: 'array', items: { type: 'string' } },
        },
        required: ['issueId', 'title', 'bodyPath'],
      },
    },
  },
  required: ['phase', 'parent', 'children'],
}

const input = typeof args === 'string' ? JSON.parse(args) : args
const { phases, githubDir: outDir } = input

const rendered = (await parallel(phases.map((p) => () => {
  const slug = String(p.num).padStart(2, '0') + '-phase-' + p.num
  return agent(
    'Render GitHub issue bodies for ONE approved phase from ' + p.file + '. Read plan files ' +
    'and write files only — no gh, git, or network; never ask the user. Copy the phase ' +
    'file\'s wording; do not re-estimate or expand. Write under ' + outDir + ':\n' +
    '1. ' + slug + '.parent.md (under 40 lines): goal, entry/exit criteria, summary table, ' +
    'execution plan, and a "Sub-issues" checklist of "- [ ] <planId> <title>" lines.\n' +
    '2. ' + slug + '.issue-<planId>.md per issue (under 30 lines): a "Plan reference" line ' +
    '(phase file + plan id), Description, Implementation Notes, Acceptance Criteria as "- [ ]" ' +
    'checkboxes, Testing Notes, and a metadata line (points, type, priority, scope, blocked-by ' +
    'plan ids).\n' +
    '3. ' + slug + '.manifest.json with the shape you return.\n' +
    'Titles: parent "Phase ' + p.num + ': ' + p.name + '"; child "[<planId>] <title>"; all ' +
    'under 256 chars. Labels: parent ["phase"]; child its priority (P0/P1/P2) and type ' +
    '(feature/bug/chore/spike) only. Plain GitHub markdown, no front matter. Return the ' +
    'Phase ' + p.num + ' manifest including manifestPath.',
    { agentType: 'dev-team:issue-estimator', label: 'render:p' + p.num, phase: 'Render', schema: MANIFEST }
  )
}))).filter(Boolean)

return {
  phases: rendered.map((r) => ({
    phase: r.phase,
    manifestPath: r.manifestPath,
    parentTitle: r.parent.title,
    childCount: r.children.length,
  })),
  totalIssues: rendered.reduce((n, r) => n + r.children.length, 0),
}
```

Read the `*.manifest.json` files after completion; they drive publishing. The `[1.1]` title
prefix is board traceability only — `/dev-run` still bans plan IDs from commit messages.

#### 7.4 Dry-run (gate)

```text
GitHub sync — <owner/repo>
| Phase issue | Sub-issues | Labels |
|-------------|------------|--------|
Total: N phase issues + N sub-issues = N · labels to create: … · already synced: … ·
possible duplicates: …
```

"Create these N issues on <owner/repo>? Reply `LGTM` to publish, or tell me what to change."
Changes → fix the manifests (re-render a phase, or edit one body file) and re-present. Never act
on an `LGTM` given before this table was shown.

#### 7.5 Publish (main loop, `gh`)

Phase by phase, in order, skipping anything in `plan/issues/github-sync.md`.

1. Phase issue:

   ```bash
   gh api repos/OWNER/REPO/issues -f title='Phase 1: Foundation' \
     -F body=@plan/issues/github/01-phase-1.parent.md -f 'labels[]=phase' \
     --jq '{number, id, url: .html_url}'
   ```

2. Each child the same way with its labels, then link it by database `id` (not number):

   ```bash
   gh api --method POST repos/OWNER/REPO/issues/PARENT_NUMBER/sub_issues -F sub_issue_id=CHILD_ID
   ```

   A parent holds ≤ 100 sub-issues; over that, stop and ask how to split.
3. Append a row to `plan/issues/github-sync.md` after **each** creation — it is the resume
   point:

   ```markdown
   # GitHub Issue Sync

   - Repository: owner/repo
   - Synced: YYYY-MM-DD

   | Plan ID | Kind | GitHub | Title | Parent |
   |---------|------|--------|-------|--------|
   | Phase 1 | phase | #12 | Phase 1: Foundation | — |
   | 1.1 | issue | #13 | [1.1] User login API | #12 |
   ```

4. Once every issue exists, rewrite `blockedBy` plan ids to `#number` in each child's local
   body and push it (`gh api --method PATCH repos/OWNER/REPO/issues/N -F body=@<file>`); append
   a "Plan ID → GitHub issue" table to each parent body the same way.
5. On failure, stop. HTTP 403 secondary rate limit → honour `Retry-After`, retry that call once.
   Anything else → report what was created and point at the sync file. Never re-run the loop
   without reading it first.

**Gate — Sync** (one table: phase issues, URLs, sub-issue counts, skipped, failed): "Are the
GitHub issues correct? Reply `LGTM` to finish, or name what to fix." Fix in place with `gh`;
never delete-and-recreate; Stage 7 never closes or deletes issues.

### Stage 8: Wrap-up (main loop)

```text
Development plan complete
Artifacts: plan/prd.md · plan/research/00-overview.md · plan/architecture.md ·
plan/project-plan.md · plan/issues/00-summary.md · plan/issues/github-sync.md (if synced)
Phases N · issues N · points N · ≈ N days single-stream
GitHub: N phase issues + N sub-issues on owner/repo | not created (Stage 7 skipped; re-runnable)
Next: /dev-run plan/issues/01-phase-1-<name>.md
```

Ask whether anything should change; route changes through that stage's revision handling and
re-run its gate. Do not restart the pipeline.
