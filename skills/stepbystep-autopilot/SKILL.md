---
name: stepbystep-autopilot
description: >-
  Autopilot edition of stepbystep. Generates a step-by-step build plan that agent sessions
  execute with minimal supervision: the agent commits each step itself once its verify
  command is green, runs a whole phase back to back, pushes the feature branch, and opens a
  pull request at each phase boundary — stopping only at human checkpoints, red verifies, or
  scope surprises. The document still carries a progress tracker, per-step prompts, concrete
  verify criteria, model tiers, explicit stop conditions, rollback, landmines, and a
  not-building-this section. Use when the user wants an agent to build something largely on
  its own with CI and PR review as the gate, including phrasings like "plan this so the agent
  can run it", "autonomous build plan", "let it commit as it goes", or an explicit
  /stepbystep-autopilot. Prefer the standard stepbystep when the human wants to review and
  commit every step personally.
---

# Step-by-step build plan (autopilot)

This skill produces a **plan document that is also an executable artifact**. Each step contains
the literal prompt a session follows to do that step's work, and the document is written to be
picked up cold by a session with no memory of the conversation that produced it.

The autopilot edition moves the human review boundary **from every step to every phase**. The
agent commits each step itself when its verify is green and carries on to the next step; the
human reviews a phase at a time, in a pull request. That buys a lot of speed, and it only works
under three properties — every rule below descends from them:

1. **The document is the single source of truth for status.** Not a separate tracker, not the
   git log, not the conversation. Each step flips its own row in the same commit as its code.
   If the document and reality disagree, the document is the bug — fix the document.
2. **One step = one commit; one phase = one pull request.** Steps stay small and individually
   revertable even though nobody reviews them one by one. The PR is the review unit.
3. **The agent stops when it should.** Autonomy is bounded by stop conditions written into the
   plan, and by verify lines strong enough to stand in for a human looking at each diff. A step
   whose correctness only a human can judge is a **checkpoint**, and the agent stops there.

## What you produce

Usually one file — the plan, conventionally `docs/BUILD_PLAN.md` (name it for the project if
that reads better: `DESIGN_PLAN.md`, `MIGRATION_PLAN.md`).

Often a second — a companion `CLAUDE.md` (or `AGENTS.md`, or whatever the target agent reads)
holding **standing instructions** that apply to every session: what the project is, the stack,
the file structure, the conventions, and the non-negotiable rules. Standing rules live there so
they don't have to be restated in every step prompt. Offer the companion file when the project
doesn't have one, or when the plan keeps wanting to repeat itself.

## Is autopilot the right edition?

Autopilot fits when the work can be checked mechanically: a test suite, a typecheck, a build,
CI on pull requests. Push back and suggest the standard edition — where the human reviews and
commits every step — when:

- **Most steps need human eyes** to verify (visual work, device behavior, product judgment). A
  plan that is mostly checkpoints is a standard plan with extra ceremony.
- **There is no test or verify command** strong enough to trust unattended.
- **The codebase is unfamiliar, or the change is risky** (security, data migration, money), and
  the user is still calibrating how much to trust the agent.
- **The agent cannot push or open pull requests** here. Autopilot can still run locally — see
  "Adapting the format" — but say so explicitly.

## Which mode: new project, or new feature?

Both are supported. Decide which one you're in before writing anything.

**How to tell:** an empty or near-empty directory with no VCS history means greenfield. An
existing codebase with commits, a build setup, and probably a companion doc means feature mode.
When it's genuinely ambiguous, ask.

| | **New project** (greenfield) | **New feature on an existing project** |
|---|---|---|
| `0. Before Starting` | `git init`, first commit, `main`, push, then branch | `git switch main && git pull` → `git switch -c feature/<name>` → `git tag pre-<name>` → push the branch |
| First step | Scaffolding — the skeleton everything else builds on, plus CI if there is none | Straight into real work |
| Scope blockquote | What the product is and isn't | What this feature adds, **and which existing behavior it must not disturb** |
| Ground Rules | Establishes the conventions | **Inherits** them from the existing companion doc — read it first and don't contradict it |
| Regression risk | None yet | Real. Every phase's PR runs the full existing suite, and the verify table names the specific features to re-check |
| First phase | Ends at the first runnable milestone | Ends at the first demoable slice of the feature |

**Cross-repo plans** are possible but cut against autopilot: one phase must never span repos,
because a PR can't. Give each repo its own phases, its own opening prompt, and a `Repo` column
in the tracker; make the hand-off between repos a checkpoint.

## Before you write

Establish these first. Ask about what you genuinely can't infer; infer the rest and say what
you inferred.

- **Which mode you're in**, and for a feature: the base branch, the feature branch name, and
  whether it already exists.
- **What existing behavior must keep working.** It becomes the "nothing regressed" row in the
  verify table and a line in every PR description.
- **What is in scope, and — more important — what is explicitly NOT.** Unattended sessions are
  even more prone to helpful scope creep than supervised ones.
- **The verify command, and how much it proves.** Autopilot leans on it entirely between
  checkpoints. If it's weak, strengthening it (tests, a CI job) is often the first step.
- **What the agent cannot verify.** Visual renders, device behavior, third-party services,
  anything needing credentials. Each of these makes the step that produces it a checkpoint.
- **The PR gate.** Is there a remote the agent may push to, and a way to open pull requests
  (`gh`, an API, a web UI the human uses)? What runs on PRs? Who reviews and merges? The agent
  never merges its own PR.
- **The commit prefix.** A short project tag: `rs-step-N:`. Derive it from the project.
- **Existing conventions.** If the repo has a companion doc, read it; the plan must not
  contradict it.

**Stay inside the target project.** Read and write only within the project being planned. If
you want a worked example of these conventions, use `references/examples.md` in this skill —
never go looking through unrelated projects on the machine for one. Other projects on disk are
someone's live work.

## Document structure

Use this section order. The tracker is first because every session reads it first; the run
instructions come before the steps because a session needs the opening and phase prompts
before it needs step 4.

```
# <Title>
> Blockquote: scope, explicitly NOT, and the autopilot rules in one line.

## Progress Tracker          table (with Gate column) + status + model note + phases
## How To Run This           numbered; ### Opening prompt, ### Phase prompt, ### PR description
### Who verifies what        table: Check | Who | "Green" looks like
## Ground Rules              numbered invariants, including the commit and push rules
### Stop conditions          the list that bounds autonomy
### Rollback                 table: Situation | Command
## 0. Before Starting        the bash block that creates the branch and pushes it
## Steps                     the body — one ### per step
## Deferred follow-ups       live tracker of out-of-scope things steps noticed
## Landmines                 real friction, not speculation
## Later                     designed, explicitly NOT built
```

`references/template.md` holds the full literal skeleton with every table shape — read it when
writing the document and copy from it rather than reconstructing the tables from memory.

**Optional sections** earn their place by removing a question a session would otherwise stop to
ask — and in autopilot, every avoided stop matters:

| Section | Placement | When to include |
|---|---|---|
| `## The core idea (read before starting)` | before the tracker | The feature has one central mechanism every step assumes. |
| `## Design Decisions Already Made` | after Ground Rules | Decisions were settled in conversation and must not be relitigated mid-execution. Unattended sessions are especially tempted to "improve" these. |
| `## References` | near the top | The plan depends on external docs or specs a session must load. |
| A domain-specific spec section (schema, endpoint table) | before Steps | Multiple steps need one shared reference. |
| `## <Rollout> gate` | after Steps | A deploy or migration must happen in a controlled window. A rollout is **always** a checkpoint, never an auto step. |

A few sections earn their place in ways that aren't obvious:

**Progress Tracker** — columns `Step | Scope | Model | Gate | Status`, status `⬜ todo` /
`✅ done`, gate `auto` / `checkpoint`. An `auto` step may be committed and followed by the next
one without a human. A `checkpoint` step is committed, pushed, and then the session stops until
the human checks what only they can check. Follow the table with: overall status, a **model
note**, and the **phases**. A session runs on one model, so group steps into phases by the
model they need — a phase that mixes a rote step and a load-bearing one runs entirely on the
stronger model.

**Opening prompt** — pasted at the start of every session. It loads the docs, verifies git
state, has the session restate the rules and stop conditions in its own words, and reports
which phase is next. It ends with "Then wait for me to say go." — the human still decides when
each phase starts. Restating matters: a session that has just written "I never push to main,
never force-push, and stop at checkpoints" behaves differently from one that merely had it in
context.

**Phase prompt** — the one prompt that makes autopilot different. It tells the session to
execute a range of steps in order, reading each step's prompt from the plan itself, committing
each one when green, and stopping at the end of the phase, at a checkpoint, or at any stop
condition. See `references/template.md` for the literal text.

**PR description** — a fixed template the session fills at each phase boundary: the steps
included, the verify evidence for each, implementation notes, anything added to Deferred
follow-ups, and an explicit **Needs human check** list. The reviewer reads this instead of
replaying the session.

**Who verifies what** — be honest about your blind spots. Every row assigned to the human makes
its step a checkpoint. A plan that implies the agent verified something it structurally cannot
is worse than no plan — and under autopilot nobody would notice until the PR.

**Landmines** — populate from *real* friction you hit or confirmed. Speculation trains the
reader to skim.

**Later** — designed but deliberately unbuilt, so no unattended session half-builds it.

## Stop conditions

Write these into Ground Rules, and have the opening prompt restate them. The agent stops,
reports, and waits — never improvises past them:

- **Verify is red after two honest fix attempts.** Report what fails and what was tried.
- **Getting green would mean weakening verification** — skipping or deleting a test, loosening
  an assertion, adding a lint ignore, catching and swallowing an error. Never do it; stop.
- **The step needs something outside its prompt** — another file the prompt didn't mention, a
  new dependency, a schema change, a design decision. Record it in Deferred follow-ups; continue
  only if the step can be completed without it, otherwise stop.
- **The diff is much larger than the step implies.** Something is wrong with the step or the
  approach; a reviewer should see it before it grows.
- **The prompt contradicts the code** — a file it names doesn't exist, an interface it relies on
  has a different shape. The plan is stale; stop so it can be fixed.
- **Anything destructive or irreversible** — deleting data, rewriting pushed history, touching
  production, changing CI or branch protection.
- **A checkpoint step, or the end of the phase.**

## Anatomy of a step

Every step takes exactly this shape:

```markdown
### <PREFIX><n> — <short title> · <model> · <gate>

**Verify:** <concrete and checkable — a command, an observable output, a specific behavior>
**Commit:** `<prefix>-step-<n>: <message>`

```text
<the full prompt for that step>
```
```

**The Verify line is a contract, not a vibe** — and under autopilot it is the only review an
`auto` step gets before the PR. "It works" is useless. "`npm test` green, including a test that
rejects `javascript:` URLs with 400" is checkable by someone who wasn't there.

**Every step prompt ends with the same closing instruction.** It keeps the tracker
synchronized and makes the commit conditional on green:

```text
Run <verify command>. If it is green, flip <PREFIX><n> to ✅ in <plan path> and commit
everything as one commit: "<prefix>-step-<n>: <message>". Paste the verify output and the
output of `git log -1 --stat`. If it is still red after two honest fix attempts, stop and
report — never commit red, never weaken a check to get green.
```

A **checkpoint** step appends: "This is a checkpoint: push the branch, update the PR, tell me
exactly what to check and where, then stop." The **last step of a phase** appends: "This ends
phase <n>: push the branch, open or update the PR using the PR description template, then
stop."

Three things are doing work there. Flipping the row **in the same commit** keeps document and
code atomically in sync. **Commit only when green** makes every commit on the branch a known-good
state, which is what makes per-step `git revert` safe. **The explicit stop** at checkpoints and
phase ends is where the human gets the review boundary back.

## Writing prompts that survive a cold session

The step prompt is the actual deliverable. Under autopilot nobody is standing by to clarify it,
so it has to be complete.

- **Name real paths, files, and identifiers.** `src/fs/exec.ts`, not "the exec module".
- **Say what NOT to do, and why.** "Do NOT touch the renderer — that's step 7, kept separate so
  its diff stays reviewable" prevents scope bleed, and the reason is what makes it stick.
- **Explain intent, not just instructions.** A session that understands why will handle the case
  you didn't anticipate.
- **Turn every judgment call into a decision or a stop.** Where a supervised plan would say "ask
  me if…", an autopilot plan either makes the decision in the prompt or names it as a stop
  condition. Unattended sessions resolve ambiguity by guessing.
- **Make the evidence part of the output.** Have the session paste verify output and command
  results into its report, so they can be copied into the PR description.

`references/examples.md` shows full worked steps — one `auto`, one `checkpoint` — plus a phase
prompt and weak-vs-strong versions of verify lines and stop conditions.

## Sizing and sequencing

**One step = one commit.** Small enough that a `git revert` of it is clean. If two steps would
always be reverted together, merge them.

**One phase = one pull request = 2 to 5 steps.** Small enough to review in one sitting. Put
checkpoint steps at phase ends where you can, so a checkpoint and a PR review happen together.

**Front-load what everything else depends on** — the contract, the shared types, the safety
layer — and run that phase on the stronger model. A wrong shape there ripples through every
unattended step after it, so the first phase often ends in a checkpoint even when its verify is
strong.

**Then reach something runnable fast.** A working surface by the end of phase two beats a
complete feature set that first runs at step 20.

Typical shape: `scaffold + CI → contract (checkpoint) → critical layer → first runnable
(checkpoint) → feature phases → polish`.

## Keeping the document alive

- **Implementation notes.** After a step is executed, the same session appends a short
  `**Implementation notes (YYYY-MM-DD):**` block under it, in the same commit — what differed
  from the prompt, decisions made, bugs found. The PR description summarizes them.
- **Steps get added mid-plan.** Number them `RS9.1`, `RS9.2` so nothing renumbers, mark them
  `*(added mid-plan YYYY-MM-DD)*`, and give them a gate like any other step. An unattended
  session never adds steps to its own phase; it stops and proposes them.
- **Deferred follow-ups is append-and-strike.** Strike through what's handled and name the step
  that resolved it. Don't silently delete lines.
- **Scope changes go in the document, not just the conversation.**

## Adapting the format

- **No remote, no PRs?** Autopilot still runs locally: the phase boundary becomes "stop and
  hand me `git log --oneline <pre-tag>..HEAD` and `git diff --stat <phase-start>..HEAD`". The
  human reviews locally before saying go for the next phase. Nothing is pushed.
- **No git?** Autopilot doesn't fit — every safety property here rests on small revertable
  commits. Use the standard edition with a checkpoint convention.
- **Not code?** Replace the verify command with whatever proves the step landed, and expect more
  checkpoints.
- **Feature on a shipped product?** Add the release constraint as a Ground Rule, restate it in
  every step, and make anything that touches the release path a checkpoint.

Keep the tracker, the gates, the per-step prompts, the verify lines, and the stop conditions.
Those are the format. The rest flexes.

## Using this outside Claude Code

The instructions here avoid tool-specific calls and absolute paths, so they port. This skill is
self-contained: everything needed is in this file and the two files beside it in
`references/`, referred to relatively.

- **Claude Code** — drop the folder in the personal skills directory (every project) or the
  project-local one (that project only), and invoke `/stepbystep-autopilot`.
- **Codex** — drop the folder in `~/.agents/skills/` or a repo's `.agents/skills/`, and mention
  it with `$stepbystep-autopilot`.
- **Any other agent** — point it at this file from that agent's standing-instructions file.
- **The output is portable regardless.** The plan is plain markdown and every prompt is
  self-contained, so a plan written by one agent can be executed by another — which is the
  deeper reason no prompt may assume conversation context.
