# Full document skeleton

Copy this and fill it in. Placeholders in `<angle brackets>`. Delete sections that genuinely
don't apply (see "Adapting the format" in SKILL.md) rather than leaving empty headings.

`<PREFIX>` is a short project tag used for both step IDs and commit messages — e.g. `RS` with
commits `rs-step-1:`, or `DP` with commits `dp-step-1:`.

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
> **One step = one commit = one reviewable unit. <Agent> never commits** — it hands over a
> paired commit+push block and stops.

---

## Progress Tracker

> Each session flips its own row to ✅ as the final action of the step, in the same commit as
> the change. Read this table at the start of a session to know where to resume. (Status
> proposed by the session; the developer's commit is the real confirmation.)

<!-- Add a `Repo` column for cross-repo plans: | Step | Repo | Scope | Model | Status | -->

| Step | Scope | Model | Status |
|------|-------|-------|--------|
| <PREFIX>1 | <what this step delivers, one line> | <model> | ⬜ todo |
| <PREFIX>2 | <…> | <model> | ⬜ todo |

**Overall status: <what's done, what's next, and the shape of the sequence.>**

**Model note:** <which steps need the stronger model and WHY — usually the contract/shared
types and the correctness-critical layer, because a wrong shape there ripples into every later
step. Name the step most likely to become a spike.>

**Session split** — natural review checkpoints:
1. **<PREFIX>1 → <PREFIX>N** — <phase name and what to review at the boundary>
2. **<PREFIX>N+1 → …** — <…>

One session can carry several steps back-to-back; the phases are suggested checkpoints.
Never start a step with uncommitted changes in the tree.

---

## How To Run This

1. Open a **new session** in `<repo path>`. <Companion doc> auto-loads.
2. **Paste the Opening prompt below first.** Wait for its green light.
3. Paste the **PROMPT** for the next unfinished step.
4. The session does the work, runs `<verify command>`, ticks its tracker row, and shows a diff
   + a paired commit+push block. **It never commits.**
5. You review, run it where relevant, and commit if it looks right.

### Opening prompt — every step

```text
You are building <project>, <one-line description>, in <repo path>.
Before any work:
1. Read <companion doc> and <plan path> in this repo, in full.
2. Run `git status` and `git branch --show-current`. Confirm the tree is clean and the
   branch is <branch>. If the repo/branch doesn't exist yet, tell me the commands from
   "0. Before Starting" and STOP until I've run them.
3. <Any environment precondition — OS, tool availability, a service being reachable.>
4. Restate, in one line each, the rules you will not break this session:
   (a) you never run `git commit` or `git push` — you hand me a paired block and stop;
   (b) <the project's other non-negotiable rule>.
5. Read the Progress Tracker and tell me which step is next (the first ⬜ todo).
Then wait. Do not implement anything until I paste the step prompt.
```

### Who verifies what

| Check | Who | "Green" looks like |
|---|---|---|
| `<verify command>` | <Agent> | No errors, clean exit |
| <test suite> | <Agent> | All tests pass; <the specific guarantee they prove> |
| **Nothing regressed** | you | <name the specific existing features to re-check — not "the app still works"> |
| <release constraint, if any> | <Agent> | <e.g. no new native dependency, no schema migration — stated explicitly in every step's diff summary> |
| <thing the agent CANNOT check> | **you** | <what to look at, specifically> |

<One or two sentences stating plainly what the agent can self-verify and what structurally
falls to the human, and committing the agent to always say which is which.>

---

## Ground Rules

1. **Branch:** `<branch>`. Tag `<pre-tag>` right after the first commit.
2. **One step = one commit**, message prefixed `<prefix>-step-N:`.
3. **Verify before every commit.** `<verify command>` must be green. Never commit red.
4. **<Agent> never commits** — paired commit+push block, then stop. Never runs `git push` either.
5. <Project-specific invariant — the thing that, if violated, breaks the product's premise.>
6. <…more invariants that apply to EVERY step, so they need not be repeated in each prompt.>
N. **Every step:** ask questions if genuinely uncertain, and update <companion doc> + this
   tracker + any architecture docs.

### Rollback

| Situation | Command |
|---|---|
| Step wrong, uncommitted | `git restore . && git clean -fd` |
| A committed step is bad | `git revert <sha>` |
| Abandon the whole thing | `git reset --hard <pre-tag>` |

---

## 0. Before Starting

<!-- USUAL CASE: a new feature on an existing project. -->

Run once, before the first step (handed over — the agent does not run git for you):

```bash
cd <repo path>
git status                               # must be clean
git checkout main && git pull            # start from current main
git checkout -b <branch>
git tag <pre-tag>
```

<!-- CROSS-REPO VARIANT: repeat per repo. -->
<!-- "In **each** repo, before its first step:" then the same block, noting which steps
     belong to which repo. -->

<!-- BRANCH-ALREADY-EXISTS VARIANT: replace the block above with a read-only pre-flight.
     "The branch already exists. Do not recreate it."

```bash
cd <repo path>
git switch <branch>
git pull --ff-only
git status --short --branch
git branch --show-current
git log -1 --oneline main                # read-only: establishes base state
git diff --stat main...HEAD
git tag <pre-tag>                        # if it already exists, inspect — don't move it
```
-->

<!-- GREENFIELD VARIANT (new project only — rarer):

```bash
cd <repo path>
git init
git add <the scope docs>
git commit -m "<prefix>-step-0: scope docs"
git tag <pre-tag>
git branch -M main
git checkout -b <branch>
# git remote add origin <url> && git push -u origin <branch>
```
-->

---

## Design Decisions Already Made

<!-- OPTIONAL — include when decisions were settled in conversation and must not be
     relitigated mid-execution. Also seen as "## Locked architecture for this plan". -->

- **<Decision>.** <What was chosen and why the alternative was rejected.> Do not redesign this
  mid-plan; if a step seems to require changing it, STOP and raise it.

---

## Steps

### <PREFIX>1 — <short title> · <model>

**Verify:** <concrete and checkable — a command, an observable output, a specific behavior.
Never "it works".>
**Commit:** `<prefix>-step-1: <message>`

```text
<Full copy-paste prompt. Written for a session with zero context from the planning
conversation. Name real paths. Say what NOT to do and why. Explain intent. Give an escalation
path for the likely failure. Invite questions on genuine ambiguity. Ask for observable output
to be pasted back.>

Run <verify command>. Flip <PREFIX>1 to ✅ in <plan path> in the same change, then show me the
diff and a single block with BOTH the git commit ("<prefix>-step-1: <message>") AND git push.
Do not commit.
```

---

### <PREFIX>2 — <short title> · <model>

**Verify:** <…>
**Commit:** `<prefix>-step-2: <message>`

```text
<…>

Run <verify command>. Flip <PREFIX>2 to ✅ in <plan path> in the same change, then show me the
diff and a single block with BOTH the git commit ("<prefix>-step-2: <message>") AND git push.
Do not commit.
```

---

## Deferred follow-ups (cross-step, live tracker)

> Aggregates flagged-but-not-fixed items that survive individual steps — things a step noticed
> but was out of its own scope. Kept here so they don't get buried in per-step notes. Cross off
> (strike through) when genuinely handled; don't silently delete lines.

- **<Item>** (surfaced by <PREFIX>N). <What it is, why it wasn't fixed there, what fixing it
  would take.>
- *(Steps add to this list as they surface out-of-scope friction.)*

---

## Landmines

Populated from real friction, not speculation:

- **<The trap>.** <What actually happens, how it was discovered, and which step or ground rule
  addresses it.>
- **<…>**

---

## Later — designed, NOT built in <milestone>

Everything below is intentionally out of scope. Recorded so the boundary is explicit and so
nobody half-builds one of these while working through the steps.

| Item | Design | What it needs |
|---|---|---|
| **<Feature>** | <how it would work, given the architecture already chosen> | <the concrete work> |

**When Later work starts:** <suggested order and why>.
````

---

## Post-execution additions

Once steps start executing, two things get appended. Neither is written at planning time.

**Implementation notes**, directly under the step they belong to:

```markdown
**Implementation notes (YYYY-MM-DD):**

- **<What differed from the prompt, and why.>** State plainly when the prompt was wrong — that
  correction is often the most valuable line in the document.
- **<Decision made by judgment call, with the reasoning.>**
- **<Bug found in testing and how it was fixed.>**
- **Not verified by the session that built it** — <what still needs a human check>.
```

**Mid-plan steps**, numbered to insert without renumbering:

```markdown
### <PREFIX>9.1 — <title> · <model>

> **Added mid-plan YYYY-MM-DD** — not part of the original scope. <Why it was added.>
```
