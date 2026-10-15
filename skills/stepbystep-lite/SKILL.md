---
name: stepbystep-lite
description: >-
  Lightweight edition of stepbystep for small, well-understood work: a feature, fix, or
  refactor of roughly 3–8 steps. Writes one compact plan document that later agent sessions
  execute one step at a time: a scope line with an explicit not-doing list, a progress
  tracker, an opening prompt, and per-step copy-paste prompts with concrete verify lines and
  commit messages. Drops the heavier sections (rollback table, landmines, later, cross-repo
  machinery) unless the work actually needs them, so the plan stays short and cheap to load.
  Use when the user wants a quick plan a later session can pick up cold: "plan this
  feature", "break this into a few steps", "quick build plan", "small plan", or an explicit
  /stepbystep-lite. For a large project, multi-repo work, a migration or rollout gate, or
  anything over about eight steps, use the full stepbystep format instead.
---

# Step-by-step plan (lite)

This skill writes a **plan document that is also an executable artifact**: each step carries
the literal prompt a fresh session pastes to do that step's work. Lite keeps exactly the parts
that make that work and cuts everything else. It is for work that is real enough to outlive one
session, but small enough that a 300-line plan would be ceremony.

## What lite keeps — the non-negotiables

1. **The document is the single source of truth for status.** Each step flips its own tracker
   row to ✅ **in the same change** as the code, so the two can never drift.
2. **One step = one commit = one reviewable unit.** The agent never commits. It ends every step
   by handing over a single block with both the `git commit` and the `git push`, then stops.
   The human reviews and commits between steps.
3. **Every step prompt survives a cold session.** A session that never saw the planning
   conversation can execute it.
4. **Every step has a falsifiable verify line.** A command and an observable result, never
   "it works".
5. **An explicit not-doing list.** Scope creep is how small plans turn into large ones; a
   fresh session that doesn't know something was excluded will helpfully build it.

Everything else in the full format is optional here, and is left out unless the work needs it.

## When lite is the wrong tool

Say so and suggest the full `stepbystep` format, rather than cramming, when any of these hold:

- More than about eight steps, or more than two or three sessions of work.
- The work spans more than one repository.
- There is a deploy, migration, or rollout that must happen in a controlled window.
- Design decisions need to be locked in writing before anyone builds, or several people or
  long-lived branches are involved.

## Before you write

Ask only about what you can't infer, and state what you inferred.

- **Mode.** An existing codebase (usual) or a new project. For an existing one, read its
  standing-instructions file (`CLAUDE.md`, `AGENTS.md`) first. The plan must not contradict it.
- **What must keep working.** On an existing project, name the specific features a careless
  change could break. They go in the verify line of the step most likely to break them.
- **The not-doing list.**
- **The verify command.** `npm run typecheck`, `cargo check`, `pytest` — whatever makes the
  tree green.
- **The commit prefix.** A short tag derived from the feature: `sv-step-N:`.
- **What the agent cannot verify.** UI renders, device behavior, live services, anything behind
  credentials. These become the step's "You check" line instead of a claim in its verify line.

Stay inside the target project. Never go looking through unrelated projects for an example; the
one below is enough.

## The document

Write it to `docs/<FEATURE>_PLAN.md` (or `docs/BUILD_PLAN.md` for a new project). Copy this
skeleton; placeholders are in `<angle brackets>`.

````markdown
# <Feature> — plan

> <One or two sentences: what this adds and why.>
>
> **Not doing:** <explicit exclusions, each with a few words of why>.
>
> **Rules:** one step = one commit. The agent never commits — it hands over one block with
> both the commit and the push, then stops. <Any project rule every step must respect.>

## Tracker

| Step | Scope | Status |
|------|-------|--------|
| <P>1 | <what this step delivers, one line> | ⬜ todo |
| <P>2 | <…> | ⬜ todo |

**Next:** <P>1. <One line on the sequence — which step first makes it runnable or demoable.>

## Opening prompt — paste first in every new session

```text
You are working on <feature> in <repo path>.
1. Read <plan path> and <standing-instructions file, if any> in full.
2. Run `git status` and `git branch --show-current`. The tree must be clean and the branch
   must be <branch>. If the branch doesn't exist, give me the command to create it from
   <base branch> and stop until I've run it.
3. Restate in one line: you never run `git commit` or `git push`.
4. Tell me the first ⬜ step in the tracker. Then wait for its prompt.
```

## Steps

### <P>1 — <short title>

**Verify:** <the command, plus the observable result someone who wasn't here can check>
**You check:** <what only the human can see, and exactly where — or "the diff only">
**Commit:** `<p>-step-1: <message>`

```text
<The full prompt. Real paths. What NOT to touch, and why. What to do if the likely failure
happens. A request to paste back the evidence.>

Run <verify command>. Flip <P>1 to ✅ in <plan path> in the same change, then show me the
diff and a single block with BOTH the git commit ("<p>-step-1: <message>") AND git push.
Do not commit.
```

## Notes

<!-- Appended while executing, never at planning time: what differed from a prompt and why,
     and follow-ups a step noticed but kept out of scope. Strike through when handled; don't
     delete. -->
````

Delete the `You check` line from a step only when the diff really is the whole check.

## Writing the steps

The step prompt is the deliverable. Write each one as if the reader has never heard of this
conversation, because it hasn't.

- **Name real paths and identifiers.** `src/routes/saved.ts`, not "the saved route".
- **Say what NOT to do, and why.** "Do NOT touch `SavedContext` — step 2 owns it and it's
  already verified" stops a whole class of scope bleed. The reason is what makes it stick.
- **Give one escalation path.** Name the most likely failure and say "STOP and tell me" for it,
  instead of pretending the step will go smoothly.
- **Ask for evidence.** Where a step produces output, have the session paste it back.
- **End with the closing instruction, word for word.** Flipping the tracker in the same change
  is what keeps the document honest; the commit-and-push block means one copy-paste for the
  human; "Do not commit" restates the rule at the exact moment it would be broken.

A verify line is good when someone else could fail it:

| Weak | Strong |
|---|---|
| The endpoint works | `npm test` green; `curl -i localhost:3000/s/abc` returns `302` with the stored URL; an unknown code returns `404`, not a crash |
| The screen looks right | `npm run typecheck`; **You check:** the Saved tab lists saved items with the existing `ItemCard`, and the empty state shows "Nothing saved yet." |

## Sizing

- **Three to eight steps.** If a step's diff wouldn't be reviewable in one sitting, split it. If
  two steps would always be reviewed together, merge them. At nine or more, switch to the full
  format.
- **Front-load the shape.** The step that defines the types or contract everything else uses
  comes first. If one step deserves a stronger model, add `· strong model` to its heading and
  say why in one line; otherwise leave models out — in a short plan they are noise.
- **Reach something runnable early.** By step two or three the user should be able to run or
  see something, even partially.

## Keeping it alive

- A step added mid-plan is numbered `<P>2.1` so nothing renumbers, and its tracker row says
  *(added YYYY-MM-DD)*.
- What a session learned that the prompt didn't know goes under **Notes**, dated. When a prompt
  turned out to be wrong, say so plainly — that line is often the most useful one in the file.
- A scope decision made in chat goes into the document. If it isn't written down, the next
  session doesn't know it happened.

## Adapting

- **No git?** Drop the commit line and the branch check; replace them with a save or checkpoint
  convention, and keep the "flip the row in the same change" rule.
- **Not code?** Replace the verify command with whatever proves the step landed — a rendered
  document, a passing lint, a review.
- **Outgrowing lite mid-plan?** Stop adding steps here; tell the user and move the plan to the
  full format, keeping the tracker history.

## Example

A three-step plan for adding a "saved items" tab to an existing mobile app. Names are stand-ins;
copy the shape.

````markdown
| Step | Scope | Status |
|------|-------|--------|
| SV1 | Saved-items store — `SavedContext` with add/remove/list, persisted, with tests | ✅ done |
| SV2 | Save button on `ItemCard` wired to the store | ⬜ todo |
| SV3 | Saved tab list screen + empty state | ⬜ todo |

### SV2 — Save button on ItemCard

**Verify:** `npm run typecheck && npm test` green, including a new test that tapping save twice
leaves exactly one saved entry; **nothing regressed** — the Home feed and pull-to-refresh tests
still pass.
**You check:** on a device, the bookmark icon fills on tap and survives an app restart.
**Commit:** `sv-step-2: save button on ItemCard`

```text
Add a save toggle to components/ItemCard.tsx using the store from SV1
(contexts/SavedContext.tsx). SV1 is done and tested — use it as-is, do not modify it.

1. Read saved state with useSaved(); never read storage directly. The context exists so two
   screens can't hold different copies of the same list.
2. The toggle is an icon button in the card's top-right corner, using the existing Icon
   component. If the card layout has no room for it, STOP and tell me — layout changes to a
   shared card are a design decision.
3. Do NOT build the Saved tab; that is SV3, kept separate so this diff stays small.

Run npm run typecheck && npm test and paste the output. Flip SV2 to ✅ in
docs/SAVED_ITEMS_PLAN.md in the same change, then show me the diff and a single block with
BOTH the git commit ("sv-step-2: save button on ItemCard") AND git push. Do not commit.
```
````

## Using this anywhere

This skill is one self-contained file with no paths outside its folder, so it can be copied,
renamed, or handed to another agent without breaking. In Claude Code, put the folder in the
personal or project skills directory and invoke `/stepbystep-lite`. In Codex, put it in
`~/.agents/skills/` or a repo's `.agents/skills/` and mention `$stepbystep-lite`. The plan it
writes is plain markdown with self-contained prompts, so any agent can execute it.
