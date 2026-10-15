# Full document skeleton (autopilot)

Copy this and fill it in. Placeholders in `<angle brackets>`. Delete sections that genuinely
don't apply (see "Adapting the format" in SKILL.md) rather than leaving empty headings.

`<PREFIX>` is a short project tag used for both step IDs and commit messages — e.g. `RS` with
commits `rs-step-1:`.

---

````markdown
# <Project> — <Plan title>

> <One paragraph: what is being built and why, in plain language.>
>
> **In scope for <milestone>:** <the concrete deliverables>.
>
> **Explicitly NOT in scope** (designed, deferred — see **Later**): <list>.
>
> **<Any non-negotiable standing rule, stated once, prominently.>**
>
> **Autopilot:** one step = one commit, made by the agent only when verify is green. One
> phase = one pull request, reviewed and merged by a human. The agent stops at checkpoints, at
> the end of each phase, and at any stop condition.

---

## Progress Tracker

> Each session flips its own row to ✅ in the same commit as the step's change. Read this table
> at the start of a session to know where to resume. `auto` steps run back to back;
> `checkpoint` steps end with a stop for a human check.

<!-- Add a `Repo` column for cross-repo plans: | Step | Repo | Scope | Model | Gate | Status | -->

| Step | Scope | Model | Gate | Status |
|------|-------|-------|------|--------|
| <PREFIX>1 | <what this step delivers, one line> | <model> | auto | ⬜ todo |
| <PREFIX>2 | <…> | <model> | checkpoint | ⬜ todo |

**Overall status: <what's done, what's next, and the shape of the sequence.>**

**Model note:** <which phases need the stronger model and WHY. A session runs on one model, so
the model is chosen per phase.>

**Phases** — one pull request each:
1. **<PREFIX>1 → <PREFIX>N** · <model> — <phase name; what the reviewer should focus on in this PR>
2. **<PREFIX>N+1 → …** · <model> — <…>

Never start a phase with uncommitted changes in the tree.

---

## How To Run This

1. Open a **new session** in `<repo path>` on the model the next phase names. <Companion doc>
   auto-loads.
2. **Paste the Opening prompt.** It reports which phase is next and waits.
3. **Paste the Phase prompt** with that phase's step range, and say go.
4. The session runs each step: does the work, runs `<verify command>`, ticks its tracker row,
   and commits — one commit per step, only when green. It stops at a checkpoint, at the end of
   the phase, or at any stop condition, and reports.
5. At the end of the phase it pushes and opens or updates the PR. You review the PR, do the
   checks listed under **Needs human check**, and merge — or reply with fixes.

### Opening prompt — every session

```text
You are building <project>, <one-line description>, in <repo path>.
Before any work:
1. Read <companion doc> and <plan path> in this repo, in full.
2. Run `git status`, `git branch --show-current`, and `git log -1 --oneline`. Confirm the
   tree is clean and the branch is <branch>. If the branch doesn't exist yet, run the
   commands in "0. Before Starting" and show me the output.
3. <Any environment precondition — tool availability, `gh auth status`, a service being up.>
4. Restate, in one line each, the rules you will not break this session:
   (a) you commit only when verify is green, one commit per step, never on <base branch>;
   (b) you never force-push, rewrite pushed history, or merge your own PR;
   (c) you stop at checkpoints, at the end of the phase, and at every stop condition in
       Ground Rules — list them;
   (d) <the project's other non-negotiable rule>.
5. Read the Progress Tracker and tell me which phase is next and which steps it covers.
Then wait for me to say go.
```

### Phase prompt

```text
Execute steps <PREFIX>a → <PREFIX>b from <plan path>, in order.

For each step:
1. Read the step's prompt in the plan and follow it exactly. Do not start work the prompt
   doesn't ask for; record anything out of scope under "Deferred follow-ups".
2. Run its verify. If green: flip its tracker row to ✅, add implementation notes under the
   step if anything differed from the prompt, and commit everything as one commit with the
   step's commit message.
3. Paste a short report: the verify output and `git log -1 --stat`.
4. If the step's gate is `checkpoint`, push, update the PR, tell me exactly what to check,
   and stop. If it is `auto`, continue to the next step.

Stop immediately at any stop condition in Ground Rules and tell me which one.
After the last step of the range: push the branch, open the PR (draft if CI hasn't run yet)
or update the existing one using the PR description template, then stop.
```

### PR description template

```markdown
## <Project> — phase <n>: <phase name>

Steps: <PREFIX>a → <PREFIX>b (plan: <plan path>)

| Step | Commit | Verify evidence |
|---|---|---|
| <PREFIX>a | <short sha> <message> | <command → result, one line> |

### Implementation notes
- <what differed from the prompts, and why>

### Added to Deferred follow-ups
- <item, or "none">

### Needs human check
- [ ] <exactly what to look at, and where — or "nothing beyond the diff">
- [ ] Nothing regressed: <the specific existing features to re-check>
```

### Who verifies what

| Check | Who | "Green" looks like |
|---|---|---|
| `<verify command>` | <Agent> | No errors, clean exit |
| <test suite> | <Agent> | All tests pass; <the specific guarantee they prove> |
| CI on the PR | CI | All required checks pass before the PR leaves draft |
| **Nothing regressed** | you, in the PR | <the specific existing features to re-check — not "the app still works"> |
| <thing the agent CANNOT check> | **you** — makes <PREFIX>N a checkpoint | <what to look at, specifically> |

<One or two sentences stating plainly what the agent self-verifies and what structurally falls to
the human. Every row assigned to the human corresponds to a checkpoint step.>

---

## Ground Rules

1. **Branch:** `<branch>`, cut from `<base branch>`. Tag `<pre-tag>` before the first step.
   The agent never commits to `<base branch>`.
2. **One step = one commit**, message prefixed `<prefix>-step-N:`, made by the agent **only when
   `<verify command>` is green.** Never commit red.
3. **Push and PR:** the agent pushes `<branch>` at checkpoints and phase ends, and opens or
   updates the phase PR. It never force-pushes, never rewrites pushed history, and never merges
   its own PR.
4. **Never weaken verification** to get green — no skipped tests, loosened assertions, new lint
   ignores, or swallowed errors.
5. <Project-specific invariant — the thing that, if violated, breaks the product's premise.>
6. <…more invariants that apply to EVERY step.>
N. **Every step:** update <companion doc> + this tracker + any architecture docs it affects, in
   the same commit.

### Stop conditions

The agent stops, reports which condition fired, and waits:

- Verify is still red after two honest fix attempts.
- Getting green would mean weakening verification (Ground Rule 4).
- The step needs something outside its prompt — a file it didn't name, a new dependency, a
  schema change, a design decision.
- The diff is much larger than the step implies.
- The prompt contradicts the code (a named file or interface doesn't exist as described).
- Anything destructive or irreversible, or anything touching CI, branch protection, or
  production.
- A checkpoint step, or the end of the phase.
- <Project-specific stop, e.g. "any change under migrations/".>

### Rollback

| Situation | Command |
|---|---|
| Step wrong, not yet committed | `git restore . && git clean -fd` |
| A committed step is bad | `git revert <sha>` (each step is one green commit, so this is clean) |
| A whole phase is bad, PR not merged | Close the PR; `git reset --hard <phase-start-sha>` locally, then push with `--force-with-lease` — **the human runs this**, never the agent |
| Abandon the whole thing | Close the PR, delete `<branch>`; `<pre-tag>` marks where it started |

---

## 0. Before Starting

<!-- USUAL CASE: a new feature on an existing project. The agent may run this block itself
     in the opening session and paste the output. -->

```bash
cd <repo path>
git status                               # must be clean
git switch <base branch> && git pull
git switch -c <branch>
git tag <pre-tag>
git push -u origin <branch>
```

<!-- BRANCH-ALREADY-EXISTS VARIANT: replace the block above with a read-only pre-flight.
     "The branch already exists. Do not recreate it."

```bash
cd <repo path>
git switch <branch>
git pull --ff-only
git status --short --branch
git log -1 --oneline <base branch>
git diff --stat <base branch>...HEAD
```
-->

<!-- GREENFIELD VARIANT:

```bash
cd <repo path>
git init && git branch -M main
git add <the scope docs>
git commit -m "<prefix>-step-0: scope docs"
git remote add origin <url> && git push -u origin main
git tag <pre-tag>
git switch -c <branch> && git push -u origin <branch>
```
-->

---

## Design Decisions Already Made

<!-- OPTIONAL — include when decisions were settled in conversation and must not be
     relitigated mid-execution. -->

- **<Decision>.** <What was chosen and why the alternative was rejected.> Do not redesign this
  mid-plan; if a step seems to require changing it, that is a stop condition.

---

## Steps

### <PREFIX>1 — <short title> · <model> · auto

**Verify:** <concrete and checkable — a command, an observable output, a specific behavior.
Never "it works". This is the only review an auto step gets before the PR.>
**Commit:** `<prefix>-step-1: <message>`

```text
<Full prompt. Written for a session with zero context from the planning conversation. Name
real paths. Say what NOT to do and why. Explain intent. Turn each judgment call into either a
decision or a named stop. Ask for observable output in the report.>

Run <verify command>. If it is green, flip <PREFIX>1 to ✅ in <plan path> and commit
everything as one commit: "<prefix>-step-1: <message>". Paste the verify output and the output
of `git log -1 --stat`. If it is still red after two honest fix attempts, stop and report —
never commit red, never weaken a check to get green.
```

---

### <PREFIX>2 — <short title> · <model> · checkpoint

**Verify:** <…>
**Human check:** <exactly what to look at, and where>
**Commit:** `<prefix>-step-2: <message>`

```text
<…>

Run <verify command>. If it is green, flip <PREFIX>2 to ✅ in <plan path> and commit
everything as one commit: "<prefix>-step-2: <message>". Paste the verify output and the output
of `git log -1 --stat`. If it is still red after two honest fix attempts, stop and report —
never commit red, never weaken a check to get green.
This is a checkpoint: push the branch, update the PR, tell me exactly what to check and where,
then stop.
```

---

## Deferred follow-ups (cross-step, live tracker)

> Things a step noticed but were out of its own scope. Kept here so they don't get buried in
> per-step notes. Strike through when genuinely handled; don't silently delete lines.

- **<Item>** (surfaced by <PREFIX>N). <What it is, why it wasn't fixed there, what fixing it
  would take.>
- *(Steps add to this list as they surface out-of-scope friction.)*

---

## Landmines

Populated from real friction, not speculation:

- **<The trap>.** <What actually happens, how it was discovered, and which step, ground rule, or
  stop condition addresses it.>

---

## Later — designed, NOT built in <milestone>

Everything below is intentionally out of scope, recorded so nobody half-builds it.

| Item | Design | What it needs |
|---|---|---|
| **<Feature>** | <how it would work, given the architecture already chosen> | <the concrete work> |

**When Later work starts:** <suggested order and why>.
````

---

## Post-execution additions

Once steps start executing, two things get appended — by the session that ran the step, in the
same commit. Neither is written at planning time.

**Implementation notes**, directly under the step they belong to:

```markdown
**Implementation notes (YYYY-MM-DD):**

- **<What differed from the prompt, and why.>** State plainly when the prompt was wrong.
- **<Decision made by judgment call, with the reasoning.>**
- **<Bug found in testing and how it was fixed.>**
- **Not verified by the session that built it** — <what still needs a human check>.
```

**Mid-plan steps**, numbered to insert without renumbering — proposed by a session that stopped,
added by the human or a planning session, never self-inserted into a running phase:

```markdown
### <PREFIX>9.1 — <title> · <model> · <gate>

> **Added mid-plan YYYY-MM-DD** — not part of the original scope. <Why it was added.>
```
