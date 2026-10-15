---
name: stepbystep
description: >-
  Generate a step-by-step build plan document that later agent sessions execute one step at a
  time — a single canonical progress tracker with per-step copy-paste prompts, concrete verify
  criteria, per-step model assignments, ground rules, rollback, landmines, and an explicit
  not-building-this section, plus an optional companion CLAUDE.md/AGENTS.md of standing
  instructions. Use this whenever the user wants to scope or plan a project, break work into
  reviewable steps, or set up work that later sessions will pick up cold — including phrasings
  like "plan this out", "scope this project", "break this into steps", "write me a build plan",
  "implementation plan", "roadmap", "phased rollout", or an explicit /stepbystep. Prefer it
  over an ad-hoc TODO list any time the plan will outlive the current conversation, and reach
  for it proactively when a request is large enough that a single session obviously cannot
  finish it.
---

# Step-by-step build plan

This skill produces a **plan document that is also an executable artifact**. It is not a
roadmap someone reads and interprets — each step contains the literal prompt a fresh session
pastes to do that step's work. The document is written to be picked up cold, months later, by
a session with no memory of the conversation that produced it.

Two properties make the whole thing work, and every rule below descends from them:

1. **The document is the single source of truth for status.** Not a separate tracker, not the
   git log, not the conversation. If the document and reality disagree, the document is the
   bug — fix the document.
2. **One step = one commit = one reviewable unit.** The human reviews and commits between
   steps. That review boundary is the point; it is what keeps a long build correctable.

## What you produce

Usually one file — the plan, conventionally `docs/BUILD_PLAN.md` (name it for the project if
that reads better: `DESIGN_PLAN.md`, `MIGRATION_PLAN.md`).

Often a second — a companion `CLAUDE.md` (or `AGENTS.md`, or whatever the target agent reads)
holding **standing instructions** that apply to every session, not just one step: what the
project is, the stack, the file structure, the conventions, and the non-negotiable rules. The
split matters — standing rules live in the companion file so they don't have to be restated in
every step prompt; step-specific work lives in the plan. Offer the companion file when the
project doesn't have one, or when the plan keeps wanting to repeat itself.

## Which mode: new project, or new feature?

Both are fully supported and equally common. Decide which one you're in before writing
anything — it changes several sections, and getting it wrong produces a plan that starts in the
wrong place.

**How to tell:** look at the target directory. An empty or near-empty directory with no VCS
history means greenfield. An existing codebase with commits, a build setup, and probably a
companion doc means feature mode. When it's genuinely ambiguous — a repo exists but the
feature is large enough to feel like its own product — ask.

| | **New project** (greenfield) | **New feature on an existing project** |
|---|---|---|
| `0. Before Starting` | `git init`, first commit, `main`, then branch | `git checkout main && git pull` → `git checkout -b feature/<name>` → `git tag pre-<name>`. **No `git init`** — the repo exists |
| First step | Scaffolding — the skeleton everything else builds on | Straight into real work; the scaffolding is already there |
| Scope blockquote | What the product is and isn't | What this feature adds, **and which existing behavior it must not disturb** |
| Ground Rules | Establishes the conventions | **Inherits** them from the existing companion doc — read it first and don't contradict it |
| Companion doc | Written fresh alongside the plan | Usually exists already — you *update* its file-structure and conventions sections as steps land |
| Regression risk | None yet | Real. The verify table needs a "nothing regressed" row naming the specific existing features to re-check |
| Early runnable milestone | Critical — get something running by step 4–5 | Often already runnable; instead sequence so the feature is *demoable* early, even partially |

Two variants worth knowing, both drawn from real plans in this format:

- **The branch already exists.** Say so explicitly — "The branch already exists. Do not
  recreate it." — and replace the setup block with a *read-only* pre-flight that inspects
  state (`git switch`, `git status --short --branch`, `git log -1 --oneline main`,
  `git diff --stat main...HEAD`) without modifying anything. Note that if the rollback tag
  already exists, inspect it rather than moving it silently.
- **Parallel efforts on other branches.** When another long-lived branch is in flight, add a
  short `## Why this is a separate branch` section with a small topology diagram and the refs
  the branch was cut from, plus guidance on alternating safely between the two. Record hashes
  as historical checkpoints, not permanent expectations — always inspect current refs before a
  step.

**Cross-repo plans are a first-class variant.** When a feature spans repos, one document owns
it (whichever repo owns the contract), the tracker gains a `Repo` column, `0. Before Starting`
is repeated per repo, and there is a **separate opening prompt per repo** so a session in the
wrong repo can't silently do the wrong step. Steps whose work happens in the other repo are
labeled in their heading.

## Before you write

Establish these first. Ask about what you genuinely can't infer; infer the rest and say what
you inferred. A plan built on wrong assumptions wastes far more of the user's time than a
short round of questions.

- **Which mode you're in** (above), and for a feature: the base branch, the feature branch
  name, and whether it already exists.
- **What existing behavior must keep working.** On an established project this is the highest-
  value question, and it becomes the "nothing regressed" row in the verify table.

- **What is in scope, and — more important — what is explicitly NOT.** Scope creep during
  execution is the primary failure mode of this format. A fresh session that doesn't know
  something was deliberately excluded will helpfully build it.
- **Where the first runnable milestone falls.** Ask yourself: at which step does the user get
  to actually run the thing? If the answer is "the last one", resequence. Something runnable
  early is worth more than a tidy dependency order.
- **What the agent cannot verify.** Visual renders, device behavior, third-party services,
  anything needing credentials. This drives the "Who verifies what" table and is where plans
  most often quietly lie.
- **The verify command.** `npm run typecheck`, `cargo check`, `pytest` — whatever makes the
  tree green. Every step ends by running it.
- **The commit prefix.** A short project-specific tag: `rs-step-N:`, `dp-step-N:`. Derive it
  from the project or plan name.
- **Existing conventions.** If the repo has a companion standing-instructions doc, read it —
  the plan must not contradict it.

**Stay inside the target project.** Read and write only within the project being planned. If
you want a worked example of these conventions, use `references/examples.md` in this skill —
never go looking through unrelated projects on the machine for one. When the user explicitly
points you at an outside file as a reference, treat it as strictly read-only: read it, do not
edit it, and say so. Other projects on disk are someone's live work, and a plan is never worth
the risk of touching them. If a plan genuinely needs to span repos, that's the cross-repo
variant below, where every repo involved is named by the user up front.

## Document structure

Use this section order. It is load-bearing: the tracker is first because it is the thing read
at the start of every session, and the run instructions come before the steps because a
session needs the opening prompt before it needs step 4.

```
# <Title>
> Blockquote: what's in scope, what's explicitly NOT, and the standing rules in one line.

## Progress Tracker          table + overall-status prose + model note + session split
## How To Run This           numbered; contains ### Opening prompt (```text fence)
### Who verifies what        table: Check | Who | "Green" looks like
## Ground Rules              numbered invariants that apply to every step
### Rollback                 table: Situation | Command
## 0. Before Starting        the bash block that creates repo/branch
## Steps                     the body — one ### per step
## Deferred follow-ups       live tracker of out-of-scope things steps noticed
## Landmines                 real friction, not speculation
## Later                     designed, explicitly NOT built
```

`references/template.md` holds the full literal skeleton with every table shape — read it when
writing the document and copy from it rather than reconstructing the tables from memory.

**Optional sections**, used when the work calls for them. Each earns its place by removing a
question a session would otherwise have to ask:

| Section | Placement | When to include |
|---|---|---|
| `## The core idea (read before starting)` | before the tracker | The feature has one central mechanism that every step assumes. Explaining it once beats re-explaining it in eight prompts. |
| `## Why this is a separate branch` | before the tracker | Another long-lived branch is in flight and someone could confuse them. |
| `## Design Decisions Already Made` / `## Locked architecture` | after Ground Rules | Decisions were settled in conversation and must not be relitigated mid-execution. This is where "we already chose X, don't redesign it" lives. |
| `## References` | near the top | The plan depends on external docs, mockups, or specs a session must load. |
| A domain-specific spec section (event taxonomy, schema, endpoint table) | before Steps | Multiple steps need one shared reference table. |
| `## Deferred / out of scope` + `## Needs <other repo> work` | after Steps | Adjacent work exists that this plan deliberately doesn't do — especially in another repo. |
| `## <Rollout> gate` | after Steps | A deploy/migration must happen in a controlled window, never during normal steps. |

Plans in this family also tend to name their siblings — "Format mirrors `<other plan>`" in the
opening blockquote. Worth doing: it tells a fresh session where to look for a worked example of
the conventions, and it keeps a family of plans visibly consistent.

A few of these earn their place in ways that aren't obvious:

**Progress Tracker** — columns `Step | Scope | Model | Status`, status `⬜ todo` / `✅ done`.
The Model column exists because steps differ in how much capability they need; assigning it
upfront stops a strong model burning on rote scaffolding and a weak one on the load-bearing
contract. Follow the table with three short prose blocks: overall status (what's done, what's
next), a **model note** justifying the Opus-vs-Sonnet split, and a **session split** proposing
natural review checkpoints.

**Opening prompt** — a fenced `text` block a fresh session pastes *before* any step prompt. It
loads the docs, verifies git state, has the session restate the non-negotiable rules in its own
words, and reports which step is next. It ends with "Then wait. Do not implement anything until
I paste the step prompt." — the human decides when work starts. Restating the rules matters
more than it looks: a session that has just written out "I never run git commit" behaves
differently from one that merely had it in context.

**Who verifies what** — this is where you are honest about your blind spots. If you cannot see
a UI render, the table says so and assigns that row to the user. A plan that implies the agent
verified something it structurally cannot is worse than no plan.

**Landmines** — populate from *real* friction: things that actually bit during investigation,
platform quirks you confirmed, tools that behave differently than assumed. Speculative
landmines ("the API might change") are noise and train the reader to skim the section.

**Later** — features that are designed but deliberately unbuilt. Its job is to stop a future
session half-building one. Say what it needs and why it's deferred.

## Anatomy of a step

Every step takes exactly this shape:

```markdown
### <PREFIX><n> — <short title> · <model>

**Verify:** <concrete and checkable — a command, an observable output, a specific behavior>
**Commit:** `<prefix>-step-<n>: <message>`

```text
<the full copy-paste prompt for that step's session>
```
```

**The Verify line is a contract, not a vibe.** "It works" is useless. "`npm run typecheck`
clean; `npm run app` prints findings grouped by level with last-used dates sourced from tool
metadata, not mtime" is checkable by someone who wasn't there.

**Every step prompt ends with the same closing instruction.** This is the most mechanical and
most important repeated element in the document — it is what keeps the tracker synchronized and
the human in control of commits:

```text
Run <verify command>. Flip <PREFIX><n> to ✅ in <plan path> in the same change, then show me
the diff and a single block with BOTH the git commit ("<commit message>") AND git push.
Do not commit.
```

Three things are doing work there. Flipping the tracker row **in the same change** keeps the
document and the code atomically in sync — a separate "remember to update the tracker" step is
always the one that gets skipped. **BOTH commit AND push in one block** means the user copies
once instead of asking twice. **Do not commit** is the standing rule restated at the exact
moment it would otherwise be violated.

If a push isn't appropriate for a given step, say so explicitly in the prompt rather than
silently omitting it.

## Writing prompts that survive a cold session

The step prompt is the actual deliverable. Everything else is scaffolding around it. Write each
one as if the session reading it has no idea the planning conversation ever happened — because
it doesn't.

- **Name real paths, files, and identifiers.** `src/fs/exec.ts`, not "the exec module".
- **Say what NOT to do, and why.** Steps drift by doing extra. "Do NOT touch the renderer —
  that's step 7, kept separate so its diff stays reviewable" prevents a whole class of scope
  bleed, and the *reason* is what makes it stick.
- **Explain intent, not just instructions.** A session that understands why a subtree must be
  targeted narrowly will handle the case you didn't anticipate. One following a rule blindly
  will not.
- **Give escalation paths.** "If the tooling fights this, STOP and tell me what it wants before
  hacking around it" is better than pretending every step goes smoothly. Name the likely
  failure and what to do about it.
- **Invite questions.** "Ask if X needs a decision I should make" — genuine ambiguity should
  surface as a question, not a guess.
- **Ask for the evidence.** Where a step produces observable output, have the session paste it
  back. That's how the user verifies without re-running everything.

`references/examples.md` shows a full worked step plus weak-vs-strong versions of verify lines
and prompts — read it if you want a concrete quality bar before drafting.

## Sizing and sequencing

**One step = one commit = one reviewable unit.** If a step's diff would be too large to review
in one sitting, split it. If two steps would always be reviewed together, merge them.

Reasonable sizing in practice: scaffolding is its own step; the core contract/types are their
own step; the safety- or correctness-critical layer is its own step; then feature groups of
1–3 related units each.

**Front-load what everything else depends on** (the contract, the shared types, the safety
layer) — these are also the steps that most deserve a stronger model, because a wrong shape
there ripples through every later step.

**Then reach something runnable fast.** A working surface at step 4 or 5, with a partial
feature set, beats a complete feature set that first runs at step 20. Everything after the
runnable milestone is additive and gets verified by the thing you already built.

Typical shape: `scaffold → contract → critical layer → first runnable → feature groups →
polish/persistence`.

## Keeping the document alive

This format is a living document, not a write-once artifact. Later sessions maintain it:

- **Implementation notes.** After a step is executed, append a short
  `**Implementation notes (YYYY-MM-DD):**` block under it recording what differed from the
  prompt, decisions made, and bugs found. When the prompt turned out to be wrong about
  something, say so plainly — that record is often more valuable than the original prompt.
- **Steps get added mid-plan.** Number them `RS9.1`, `RS9.2` to insert without renumbering, and
  mark them `*(added mid-plan YYYY-MM-DD)*`.
- **Deferred follow-ups is append-and-strike.** When something is genuinely handled, strike it
  through and note which step resolved it. Don't silently delete lines — the history of what
  was noticed and when is useful.
- **Scope changes go in the document, not just the conversation.** A decision made in chat that
  isn't written down didn't happen.

## Adapting the format

The skeleton assumes a git repo and a typecheck-able codebase. Adapt rather than force:

- **No git?** Drop `0. Before Starting` and Rollback, replace the commit line with a
  checkpoint/save convention.
- **Not code?** Replace the verify command with whatever proves the step landed — a rendered
  document, a passing lint, a review.
- **Multi-repo?** See the cross-repo variant above — `Repo` column, per-repo setup, and a
  separate opening prompt per repo.
- **Feature on a shipped product?** Add whatever release constraint governs it as a Ground
  Rule and restate it per step — e.g. "every step must state its OTA status", or "no schema
  migration before the rollout gate". A constraint that only appears once gets forgotten by
  step six.

Keep the tracker, the per-step prompts, the verify lines, and the never-commit handoff. Those
are the format. The rest flexes.

## Using this outside Claude Code

The instructions here deliberately avoid tool-specific calls and absolute paths, so they port.
This skill is self-contained: everything needed is in this file and the two files beside it in
`references/`, referred to relatively. Nothing here points at a path outside the skill folder,
which is what lets it be copied, moved, renamed, or handed to someone else without breaking.

- **Claude Code** — drop the folder in the personal skills directory (available in every
  project) or the project-local one (that project only). Invoke by name as a slash command.
- **Another agent (Codex, etc.)** — point it at this file, or reference it from that agent's
  standing-instructions file: "When asked to plan a project, follow the stepbystep skill at
  `<wherever you put it>`." Use a path relative to that agent's own config so it survives being
  moved. There is no slash-command equivalent; ask for it by name.
- **The output is portable regardless.** The plan document is plain markdown and each step
  prompt is a self-contained paste, so a plan written by one agent can be executed by another.
  That is the deeper reason step prompts must never assume conversation context.
