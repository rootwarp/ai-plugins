---
name: dev-plan
description: Full development planning pipeline. It controls an interactive flow with gates, from a project idea to sprint-ready issues. The PRD interview and each LGTM gate stay in the main loop. Background Workflow runs do the heavy research, architecture, and issue-estimation fan-out, and they reuse the dev-team planning agents. The pipeline size agrees with the work (feature or project). The issue size agrees with the tasks of the plan, and the number of issues has no limit. As an option, it publishes the approved plan to GitHub as one phase issue for each phase, with each issue linked as a sub-issue.
disable-model-invocation: true
user-invocable: true
argument-hint: [project description or requirements] [--github]
---

# Development Planning Pipeline

You are the team lead. You take a project idea through eight stages and five LGTM gates to
sprint-ready issues on disk. GitHub publishing adds two more gates. Stage 7 (GitHub issues) is
optional. All other stages are mandatory.

Interactive work (the PRD interview and each gate) runs in the main loop. Heavy fan-out
(research, architecture, issue estimation, GitHub rendering) runs as background **Workflow** runs
over the dev-team agents. Subagents and workflows cannot speak to the user, so you ask all
questions.

| Agent | `agentType` | Role | Runs via |
|-------|-------------|------|----------|
| prd-writer | `dev-team:prd-writer` | write the PRD from your interview | Agent tool (Stage 2) |
| researcher | `dev-team:researcher` | investigate angles, verify claims | Workflow (Stage 3) |
| software-architect | `dev-team:software-architect` | candidate designs, judge, final | Workflow (Stage 4) |
| project-planner | `dev-team:project-planner` | phases, milestones, dependencies | Agent tool (Stage 5) |
| issue-estimator | `dev-team:issue-estimator` | 2-3 day issues, GitHub bodies | Workflow (Stages 6, 7) |

## Output Style

- Write your messages to the user and each document that you write in ASD-STE100 Simplified
  Technical English. Load the `dev-team:ste-writing` skill for the rules. The dev-team agents
  preload that skill.
- You coordinate the work. Do not describe your work. Speak only at gates and at the wrap-up,
  within these limits.
- Start a gate summary with the numbers. Then give only the items that the user must approve:
  decisions, recommendations, and recorded assumptions. Do not paste a document into the chat.
- Limits: PRD gate ≤ 12 lines. Research, architecture, and plan gates ≤ 15 lines. Estimation
  gate = the phase table + ≤ 10 lines. GitHub dry-run = the sync table. Wrap-up = the Stage 8
  block.
- End each gate with one question that names `LGTM` as the signal to accept.

## Orchestration Rules

### Mechanisms

- **Main loop**: Use it for all work that must speak to the user: the PRD interview, each gate,
  and each AskUserQuestion. Subagents cannot ask questions. If a subagent needs an answer, it
  returns a `Questions` block. Ask the user, then resume the subagent (SendMessage) or start it
  again with the answers.
- **Agent tool**: Use it for single sequential agents: the PRD (Stage 2), the project plan
  (Stage 5), and revisions of one file.
- **Workflow tool**: Use it for non-interactive fan-out: Stages 3, 4, and 6, and the render step
  of Stage 7. If this session has no Workflow tool, do the same stages with the Agent tool:
  - Start the agents of the script with the same `agentType`s and prompts. Start independent
    agents in one message, in parallel.
  - Apply the same stage sequence and the same null handling manually.
  - Then do the gate as usual.
- **Main loop, sequential, after explicit approval**: Use it for each side effect outside the
  working tree. These are all `gh` writes in Stage 7 and the plan commit in Stage 8. Do not put
  a network mutation or a git write in a workflow script.

### Workflow stages: compute → launch → wait → read → gate

1. **Compute `args`** in the main loop from the approved artifacts. Each stage tells what to
   build. Give `args` as a real JSON object, not as a JSON-encoded string. Each script starts
   with `const input = typeof args === 'string' ? JSON.parse(args) : args`, so both forms work.
   Keep that line.
2. **Launch** the workflow with the embedded script and `args`.
3. **Wait** for the completion notification. Do not start the gate or the next workflow before
   it.
4. **Read the artifacts from disk.** The return value of the workflow is only a short summary.
5. **Gate.** Go to the next stage only on `LGTM`.

Each workflow prompt tells the agent that it runs unattended. The agent must not ask the user
questions. It records assumptions for the gate. The Output Style section of each agent gives
the house style and the line limits. The prompts only name the limit. Keep both when you adapt a
prompt.

### Gates

- Do not skip a gate. Only an exact `LGTM` (case-insensitive) moves the pipeline forward. "ok",
  "looks fine", silence, and partial approval are revision requests. Do the revision and ask
  again.
- The user can decline Stage 7 fully. After the user selects Stage 7, you cannot skip its gates.

### Revisions: the smallest change that satisfies the request

- If the fan-out shape changes (a new research angle, a new architecture lens, a phase split),
  launch that workflow again with revised `args`. Point it at the existing artifacts so that it
  builds on them.
- For a local fix (a name, a wording, one issue), start one agent of the type that made the
  file. Use the Agent tool, and give the file path and the exact change. The agent edits the
  file in place.
- For the PRD and the project plan, start a new agent of that type with the file and the
  changes.
- For published GitHub issues, edit with `gh issue edit` or `gh api --method PATCH`. If you run
  Stage 7 again, it creates only the items that are not in `plan/issues/github-sync.md`.
- For each revision, start a new agent or workflow. Do not resume a completed workflow. Resume
  an agent (SendMessage) only to answer its `Questions` block.

### General

- Each stage writes files under the plan directory. Later stages read these files by path, and
  `args` always contain the paths from earlier stages.
- Workflows use the registered agents again by `agentType`. Do not define the agents again.
- Use a checklist of eight lines to track progress. You do not set up or remove a team.

## Stages

### Stage 1: Kickoff (main loop)

1. Read `$ARGUMENTS`. If it does not have enough detail for a PRD brief, ask one time about the
   problem, the users, and the known constraints (stack, timeline, team).
2. **Set the size of the work.** The size sets each fan-out after this step (see the table
   below).
   - `feature`: a change to an existing codebase, of about 3 weeks of work or less.
   - `project`: a greenfield project, or a change that adds or changes the shape of several
     modules.
   - Use Glob on the repo to decide. If the size is not clear, use AskUserQuestion.
   - Give the scope on one line, and put it in each agent prompt.
3. The output directory is `plan/`. The workflows write `plan/research/` and `plan/issues/`.
   Change the directory only if the user asks.
4. Record if the user gave `--github` or asked for GitHub issues. Use this decision only in
   Stage 7.

| | `feature` | `project` |
|---|-----------|-----------|
| PRD | ≤ 60 lines, P0 = what the feature needs | ≤ 120 lines |
| Research | 1-2 angles, only open technical questions | 2-5 angles |
| Architecture | one design, `fit-existing` lens, only the delta, ≤ 120 lines | three lenses + judge |
| Plan | the fewest phases that the work needs, usually 1-2 | the fewest phases that the work needs |
| Issues | by default, one for each plan task, split or merged on size, no count limit | same |

If you are not sure, select `feature`. The `project` pipeline on a feature makes plans too
large. Each extra angle, module, and phase becomes issues. Each issue costs `/dev-run` one
writer, three reviewers, and two approvals. Large work still gets all the issues that it needs.

### Stage 2: PRD (main-loop interview, then Agent tool)

1. **Interview** the user yourself with AskUserQuestion, in batches of 1-4 questions. Ask only
   about the items that `$ARGUMENTS` did not answer:
   - the problem and the current situation
   - the users
   - must-have and nice-to-have items
   - hard constraints
   - the definition of done and the metrics
   - integrations and non-functional needs

   Stop when the P0 requirements and the success metrics are clear. The remaining items become
   Open Questions.
2. Start `subagent_type: "dev-team:prd-writer"`. Give it the scope, the constraints, and the
   interview answers without changes. It writes `plan/prd.md` with P0/P1/P2 requirements, within
   the line limit of the scope. If it returns a `Questions` block and no file, ask the questions
   and resume it with the answers.
3. **Gate: PRD** (≤ 12 lines: problem, users, metrics, P0/P1/P2 highlights): "Does this PRD
   capture your requirements? Reply `LGTM` to proceed, or describe the changes."

### Stage 3: Research (Workflow: parallel angles, adversarial verification)

**`args`:** Read `plan/prd.md` and select the important angles:

- For `feature` scope, select 1-2 angles. Select only questions that the codebase and the PRD
  cannot answer.
- For `project` scope, select 2-5 angles.
- Give each angle as `{ key, prompt }`. `key` is a slug that is safe for a filename. `prompt`
  names the researcher template and the specific question. The templates are A technology
  comparison, B deep dive, C landscape, D best practices, and E feasibility.
- Also give `prdPath: 'plan/prd.md'` and `researchDir: 'plan/research'`.

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
const UNATTENDED = 'Unattended run: do not ask the user questions. Record your assumptions under "Assumptions". '

const results = await pipeline(
  angles,
  (a) => agent(
    'Research ONE angle for a development plan. PRD: ' + prd + '. Angle "' + a.key + '": ' +
    a.prompt + ' Use WebSearch and WebFetch. Cite each claim inline, and end with a Sources list. ' +
    'Write ' + outDir + '/' + a.key + '.md with the matching researcher template (A-E) and your ' +
    'Output Style. Write fewer than 100 lines. ' + UNATTENDED +
    'Return angle, summary, the key claims with sources, assumptions, and docPath.',
    { agentType: 'dev-team:researcher', label: 'research:' + a.key, phase: 'Investigate', schema: FINDINGS }
  ),
  (r) => r ? agent(
    'Adversarially verify research claims against primary sources. Flag each claim that is ' +
    'unsupported, outdated, overstated, or contradicted. If a significant claim has only one ' +
    'source, give it low confidence. Angle "' + r.angle + '". Claims: ' +
    JSON.stringify(r.claims) + '. ' + UNATTENDED +
    'Return angle, verifiedClaims, refutedClaims, and notes (a few lines, with no repeated content).',
    { agentType: 'dev-team:researcher', label: 'verify:' + r.angle, phase: 'Verify', schema: VERDICT }
  ).then((v) => ({ ...r, verdict: v })) : null,
)

const clean = results.filter(Boolean)
if (!clean.length) return { angles: [], overview: null, error: 'every research angle failed' }
const verification = clean.map((r) => ({
  angle: r.angle,
  refuted: r.verdict ? r.verdict.refutedClaims : null,
  notes: r.verdict ? r.verdict.notes : null,
}))

const summary = await agent(
  'Consolidate a multi-angle investigation. Read PRD ' + prd + ' and each document in ' + outDir +
  '. Write ' + outDir + '/00-overview.md. Put the recommendations first: one bullet and a ' +
  'one-line reason for each. Resolve conflicts. Remove each refuted claim, or add a caveat to ' +
  'it: ' + JSON.stringify(verification) + '. List the consolidated assumptions, and link each ' +
  'angle document. Do not add background or a recap. Write fewer than 60 lines. ' + UNATTENDED +
  'Return a short summary and the open questions.',
  { label: 'synthesize', phase: 'Synthesize' }
)

return { angles: clean.map((r) => r.angle), overview: outDir + '/00-overview.md', summary }
```

**Gate: Research** (≤ 15 lines from `plan/research/00-overview.md`: recommendations, resolved
conflicts, assumptions): "Does this research cover what the PRD needs? Reply `LGTM` to proceed,
or name areas to investigate further."

### Stage 4: Architecture (Workflow: three candidates and one judge, or one design for a feature)

**`args`:** `priorities` = lenses `{ key, prompt }`. For `feature` scope, use exactly one lens:

- `fit-existing`: "Extend the existing architecture. Use its modules, patterns, and stack again.
  Add a module only when no existing module can own the behavior. Document only the delta."

With one lens, the script writes `plan/architecture.md` directly. It makes no candidates and
uses no judge.

For `project` scope, use these three fixed lenses, unless the feedback of the user changes them:

- `modular-monolith`: "Optimize for simplicity: the fewest moving parts that satisfy the PRD."
- `scale-first`: "Optimize for scale: independent scaling seams and async boundaries from the
  first day."
- `extraction-first`: "Optimize for service extraction: each module can become a service with
  no rewrite."

Also give `prdPath`, `researchDir`, `architecturePath: 'plan/architecture.md'`, and `scope`.

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
const UNATTENDED = 'Unattended run: do not ask the user questions. Record your assumptions under Open Questions. '

if (priorities.length === 1) {
  const only = priorities[0]
  const direct = await agent(
    'Design the architecture for a ' + (input.scope || 'feature') + '-scope change. PRD: ' + prd +
    '. Research: ' + researchDir + '. Read the existing codebase first. Lens "' + only.key +
    '": ' + only.prompt + ' Follow the software-architect template and your Output Style. Do ' +
    'not make circular dependencies. Give a reason for each new module. Write fewer than 120 ' +
    'lines. Write ' + outPath + '. ' + UNATTENDED +
    'Return a summary: changed modules, new modules, and key decisions.',
    { agentType: 'dev-team:software-architect', label: 'design:' + only.key, phase: 'Design' }
  )
  return { winner: only.key, architecture: outPath, summary: direct }
}

const candidates = (await parallel(priorities.map((p) => () => agent(
  'Design a candidate architecture. PRD: ' + prd + '. Research: ' + researchDir + '. ' +
  'Optimize for "' + p.key + '": ' + p.prompt + ' Follow the software-architect template and ' +
  'your Output Style. Do not make circular dependencies. Write fewer than 200 lines. Write it ' +
  'to ' + candBase + '.' + p.key + '.candidate.md. ' + UNATTENDED +
  'Return priority, path, summary, and keyDecisions.',
  { agentType: 'dev-team:software-architect', label: 'design:' + p.key, phase: 'Design', schema: CANDIDATE }
)))).filter(Boolean)

if (!candidates.length) return { winner: null, architecture: null, error: 'every candidate failed' }

const judged = await agent(
  'Judge candidate architectures against the quality checklist: no circular dependencies, ' +
  'one responsibility for each module, clear data ownership, testable in isolation, a clear ' +
  'extraction path, a reason for the module count, and simplicity. Candidates: ' +
  JSON.stringify(candidates.map((c) => ({ priority: c.priority, path: c.path }))) +
  '. Read all of them. ' + UNATTENDED + 'Return them in rank order, best first. Give each one ' +
  'a 0-100 total, the ideas to keep (keep), and the weakest points (weak), one line each.',
  { agentType: 'dev-team:software-architect', label: 'judge', phase: 'Judge', schema: RANKING }
)

const ranked = judged && judged.ranked && judged.ranked.length
  ? judged.ranked
  : candidates.map((c) => ({ candidate: c.priority, total: 0 }))

const result = await agent(
  'Produce the FINAL architecture. Ranked candidates: ' + JSON.stringify(ranked) + '. Files: ' +
  JSON.stringify(candidates.map((c) => c.path)) + '. PRD: ' + prd + '. Start from the ' +
  'top-ranked design, and add the "keep" ideas from the other candidates. Do not make circular ' +
  'dependencies. Do the quality checklist. Write ' + outPath + ' with the software-architect ' +
  'template and your Output Style. Write fewer than 250 lines. Do not describe how you judged ' +
  'the candidates. ' + UNATTENDED +
  'Return a summary, and tell which ideas came from which candidate.',
  { agentType: 'dev-team:software-architect', label: 'synthesize', phase: 'Synthesize' }
)

return { winner: ranked[0] ? ranked[0].candidate : null, architecture: outPath, summary: result }
```

**Gate: Architecture** (≤ 15 lines from `plan/architecture.md`: the winning lens, the module
table with new and reused modules for a `feature`, the merged decisions, and the assumptions):
"Does this architecture align with your vision? Reply `LGTM` to proceed, or list modules to add,
remove, or restructure."

### Stage 5: Project Plan (Agent tool)

The division into phases is sequential reasoning. Use one agent and no workflow.

1. Start `dev-team:project-planner`. Give it the scope, the inputs `plan/prd.md`,
   `plan/research/`, and `plan/architecture.md`, and the output `plan/project-plan.md` with its
   template. Stage 6 reads its `## Phase N: [Name]` headings and `- [ ] Task N.M` lines.
2. **Gate: Plan** (≤ 15 lines: one line for each phase with its task count, the total task count,
   and the dependency highlights). The total task count is about the number of issues to
   expect. Ask: "Does the phasing make sense? Reply `LGTM` to proceed, or list phases to reorder,
   split, or merge."

### Stage 6: Issues (Workflow: pipeline for each phase)

**`args`:**

- `phases` = the ordered `{ num, name, taskCount }` items from the `## Phase N: [Name]` headings
  of the plan. If there are no headings, stop.
- `taskCount` = the number of `- [ ] Task N.M` lines under that heading.
- `docs = { prd, researchDir, architecture, projectPlan }`
- `issuesDir: 'plan/issues'`

`taskCount` is a reference point for the estimator and the gate. It is not a limit. The issue
count follows the size of the work.

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
const UNATTENDED = 'Unattended run: do not ask the user questions. Record your assumptions in the phase overview. '
const slug = (s) => s.toLowerCase().replace(/[^a-z0-9]+/g, '-').replace(/^-|-$/g, '')
const phaseFile = (p) =>
  outDir + '/' + String(p.num).padStart(2, '0') + '-phase-' + p.num + '-' + slug(p.name) + '.md'

const perPhase = await pipeline(
  phases,
  (p) => agent(
    'Estimate ONE phase. Inputs: PRD ' + docs.prd + ', architecture ' + docs.architecture +
    ', project plan ' + docs.projectPlan + ', research ' + docs.researchDir + '. Divide Phase ' +
    p.num + ' (' + p.name + ') into issues. Follow your phase template, Estimation Rules, Issue ' +
    'Sizing, and Output Style. By default, write one issue for each plan task (the plan lists ' +
    (p.taskCount || 'an unknown number of') + ' tasks). Use a single stream. Write ' +
    phaseFile(p) + '. ' + UNATTENDED + 'Return phase, file, issueCount, and totalPoints.',
    { agentType: 'dev-team:issue-estimator', label: 'estimate:p' + p.num, phase: 'Estimate', schema: PHASE_RESULT }
  ),
  (r, p) => r ? agent(
    'Adversarially review the issue breakdown for Phase ' + p.num + ' (' + p.name + ') at ' +
    r.file + ' against your Estimation Rules and Issue Sizing. Only review. Do not write files. ' +
    'Put in oversizedIssues each issue with more than 5 points, or with 5 points and no reason. ' +
    'Put in mergeCandidates each issue that takes less than about one day, changes the same ' +
    'files as a different issue, is standalone test, docs, config, or refactor work, or is not ' +
    'necessary for the plan or the codebase. Name the issue to merge it into. Put in ' +
    'missingDeps each missing or implicit dependency and each acceptance criterion that you ' +
    'cannot test. Judge each issue on its size and cohesion. Do not judge it on the number of ' +
    'issues in the phase. ok is true only when all lists are empty. ' + UNATTENDED +
    'Return phase, oversizedIssues, mergeCandidates, missingDeps, and ok.',
    { agentType: 'dev-team:issue-estimator', label: 'check:p' + p.num, phase: 'Check', schema: CHECK }
  ).then((c) => ({ ...r, check: c, tasks: p.taskCount || null })) : null,
  (r, p) => (r && r.check && !r.check.ok) ? agent(
    'Revise the Phase ' + p.num + ' issue file ' + r.file + ' in place to resolve these review ' +
    'findings: ' + JSON.stringify(r.check) + '. Do each merge and split that the findings name. ' +
    'Renumber the issues. Correct the Blocked by and Blocks references and both tables. Do not ' +
    'change other content. ' + UNATTENDED + 'Return phase, file, issueCount, and totalPoints.',
    { agentType: 'dev-team:issue-estimator', label: 'revise:p' + p.num, phase: 'Revise', schema: PHASE_RESULT }
  ).then((v) => (v ? { ...r, ...v, revised: true } : r)) : r,
)

const clean = perPhase.filter(Boolean)
if (!clean.length) return { phases: 0, summary: null, error: 'every phase estimate failed', sizing: [] }

const summary = await agent(
  'Write the cross-phase summary ' + outDir + '/00-summary.md from the phase files ' +
  JSON.stringify(clean.map((r) => r.file)) + '. Use the issue-estimator summary template: ' +
  'the approach, a phase table with plan tasks, issue counts, and points, the single-stream ' +
  'execution plan, the dependency map, and the risk flags. Use relative links. Use only tables ' +
  'and bullets. Write fewer than 80 lines. ' + UNATTENDED +
  'Return the total issues, the total points, and the risk flags.',
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

These rules apply to the workflow results:

- If a workflow (Stage 3, 4, or 6) returns `error`, it wrote no usable artifacts. Report the
  error and launch the workflow again. Do not do a gate on missing artifacts.
- Each stage callback receives `(previousResult, originalPhase)`. Thus, Check and Revise know
  `p.num`.
- `sizing[].check` is the review before the revision. Read the phase files again for the final
  counts.

**Gate: Estimation** (the phase table from `plan/issues/00-summary.md`, with the plan tasks and
the issues for each phase, + ≤ 10 lines). The lines give the execution plan, the merges and
splits that the check made, and the reason if a phase has many more issues than tasks. Ask:
"Are the estimates reasonable? Reply `LGTM` to proceed, or list issues to split, merge, or
reprioritize."

### Stage 7: GitHub Issues (optional: workflow render, main-loop publish)

Create one phase issue for each phase. Each planned issue is a sub-issue of its phase issue.
First, the user must opt in. Then there are two LGTM gates: a dry-run of exactly what you will
create, and a sync check after you publish.

#### 7.1 Opt in

Use AskUserQuestion: "Create these issues on GitHub? One phase issue per phase, each issue as a
sub-issue." with the options "Yes, create GitHub issues" / "Skip — keep the plan local".

- If the user gave `--github`, get a confirmation. Do not ask the question again. The dry-run
  gate still applies.
- If the user selects Skip, go to Stage 8.

#### 7.2 Preflight (Bash)

If a check fails, report the failure and offer to skip the stage.

1. Run `gh --version` and `gh auth status`. If `gh` is missing or not authenticated, tell the
   user to install it or to run `gh auth login`. Offer to skip.
2. Run `gh repo view --json nameWithOwner -q .nameWithOwner`. If there is no remote, use
   AskUserQuestion to get `owner/repo`. Do not guess a repository.
3. If `plan/issues/github-sync.md` exists, read it. All the items in it exist, so skip them.
4. If there is no sync file, run `gh issue list --state all --limit 1000 --search 'in:title
   "Phase"' --json number,title`. Also do one search for each distinctive child title. Compare
   the phase titles and the issue titles. Report collisions at the dry-run gate.
5. Run `gh label list --limit 200 --json name -q '.[].name'`. This step only reads. Record
   which of the labels that you will use (`phase`, `P0`/`P1`/`P2`, `feature`/`bug`/`chore`/`spike`)
   are missing. Create them in 7.5, after the dry-run LGTM. Do not create them in this step.

#### 7.3 Render (Workflow)

**`args`:** `phases` = `[{ num, name, file }]` from Glob `plan/issues/0[1-9]*.md` and
`plan/issues/[1-9]*.md`, in phase order. `githubDir: 'plan/issues/github'`. The render does no
network work, so it does not need a repository name.

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
    'Render GitHub issue bodies for ONE approved phase from ' + p.file + '. Only read plan ' +
    'files and write files. Do not use gh, git, or the network. Do not ask the user questions. ' +
    'Copy the wording of the phase file. Do not estimate again or add content. Write these ' +
    'files under ' + outDir + ':\n' +
    '1. ' + slug + '.parent.md (fewer than 40 lines): goal, entry and exit criteria, summary ' +
    'table, execution plan, and a "Sub-issues" checklist of "- [ ] <planId> <title>" lines.\n' +
    '2. ' + slug + '.issue-<planId>.md for each issue (fewer than 30 lines): a "Plan reference" ' +
    'line (phase file and plan id), Description, Implementation Notes, Acceptance Criteria as ' +
    '"- [ ]" checkboxes, Testing Notes, and a metadata line (points, type, priority, scope, ' +
    'blocked-by plan ids).\n' +
    '3. ' + slug + '.manifest.json with the shape that you return.\n' +
    'Titles: the parent title is "Phase ' + p.num + ': ' + p.name + '". A child title is ' +
    '"[<planId>] <title>". Each title has fewer than 256 characters. Labels: the parent gets ' +
    '["phase"]. A child gets only its priority (P0/P1/P2) and its type ' +
    '(feature/bug/chore/spike). Use plain GitHub markdown with no front matter. Return the ' +
    'Phase ' + p.num + ' manifest, with manifestPath.',
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

After the workflow completes, read the `*.manifest.json` files. They control the publish step.
The `[1.1]` title prefix is only for traceability on the board. `/dev-run` still does not permit
plan IDs in commit messages.

#### 7.4 Dry-run (gate)

```text
GitHub sync — <owner/repo>
| Phase issue | Sub-issues | Labels |
|-------------|------------|--------|
Total: N phase issues + N sub-issues = N · labels to create: … · already synced: … ·
possible duplicates: …
```

Ask: "Create these N issues on <owner/repo>? Reply `LGTM` to publish, or tell me what to
change."

- If the user asks for changes, correct the manifests (render a phase again, or edit one body
  file). Then show the table again.
- Do not act on an `LGTM` that the user gave before you showed this table.
- If there are possible duplicates, use AskUserQuestion for each duplicate before the LGTM
  question: "Skip — use existing #N" (record it in `github-sync.md` as existing) / "Create
  anyway".

#### 7.5 Publish (main loop, `gh`)

Do the phases one by one, in sequence. Skip all items in `plan/issues/github-sync.md`.

0. Create the labels first. Create only the missing labels that the approved dry-run names, for
   example `gh label create phase --color 0E8A16 --description "Phase-level tracking issue"`.
   Issue creation fails on unknown labels.
1. Create the phase issue:

   ```bash
   gh api repos/OWNER/REPO/issues -f title='Phase 1: Foundation' \
     -F body=@plan/issues/github/01-phase-1.parent.md -f 'labels[]=phase' \
     --jq '{number, id, url: .html_url}'
   ```

2. Create each child in the same way, with its labels. Then link it by its database `id` (not
   its number):

   ```bash
   gh api --method POST repos/OWNER/REPO/issues/PARENT_NUMBER/sub_issues -F sub_issue_id=CHILD_ID
   ```

   A parent can hold a maximum of 100 sub-issues. If there are more, stop and ask how to split
   them.
3. Add a row to `plan/issues/github-sync.md` after **each** creation. This file is the resume
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

4. When all issues exist, change the `blockedBy` plan ids to `#number` in the local body of each
   child. Then push the body (`gh api --method PATCH repos/OWNER/REPO/issues/N -F body=@<file>`).
   In the same way, add a "Plan ID → GitHub issue" table to each parent body.
5. If a step fails, stop.
   - For an HTTP 403 secondary rate limit, obey `Retry-After`, and try that call one more time.
   - For other failures, report what you created, and point to the sync file.
   - Do not run the loop again before you read the sync file.

**Gate: Sync** (one table: phase issues, URLs, sub-issue counts, skipped items, failed items):
"Are the GitHub issues correct? Reply `LGTM` to finish, or name what to fix." Correct the issues
in place with `gh`. Do not delete an issue and create it again. Stage 7 does not close or delete
issues.

### Stage 8: Wrap-up (main loop)

```text
Development plan complete
Artifacts: plan/prd.md · plan/research/00-overview.md · plan/architecture.md ·
plan/project-plan.md · plan/issues/00-summary.md · plan/issues/github-sync.md (if synced)
Phases N · issues N · points N · ≈ N days single-stream
GitHub: N phase issues + N sub-issues on owner/repo | not created (Stage 7 skipped; re-runnable)
Next: /dev-run plan/issues/01-phase-1-<name>.md
```

Ask if the user wants to change something. Send each change through the revision procedure of
that stage, and do its gate again. Do not start the pipeline again.

Then offer to commit the plan. `/dev-run` needs `plan/` on `develop`, because it makes its
worktrees from `develop` and it needs a clean main tree. Use AskUserQuestion: "Commit plan/ to
develop" / "Commit and push to origin/develop" / "Leave plan/ uncommitted". If the user selects
a commit option:

1. Switch the main tree to `develop`.
   - If `develop` does not exist, ask the user if you can create it from the default
     branch.
   - If there are tracked changes outside `plan/`, report them and stop. Do not stash or discard
     changes.
2. Run `git add plan/ && git commit -m "docs(plan): add development plan"`.
3. Push only if the user selected the push option.

Do not commit or push without that answer.
