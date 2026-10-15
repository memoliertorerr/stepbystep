# stepbystep

Agent skills that turn "build me X" into a **plan document later sessions can execute one step at
a time**: a progress tracker, a copy-paste prompt for every step, concrete verify criteria,
rollback, and an explicit list of what is deliberately *not* being built.

Works with **Claude Code** (as skills or as a plugin marketplace) and **Codex**, and with any agent
that can read a markdown file. Comes in four editions, so you can pick the amount of ceremony and
autonomy that fits the work.

---

## The problem

Ask an agent to build something substantial and you usually get one of two failure modes:

- **A roadmap nobody can execute.** Bullet points at the wrong altitude — fine to read, useless to
  act on, and worthless to a session that wasn't in the conversation that produced them.
- **A long session that drifts.** Scope creeps, the plan lives only in the context window, and
  when the session ends or compacts, the thread is gone.

The gap is that plans are usually written for *humans to interpret* rather than for *agents to
execute*. A plan a fresh session can pick up cold, months later, is a different artifact.

## What it produces

One markdown file that is both the plan **and** the status tracker. Every step carries the literal
prompt you paste to do that step's work:

````markdown
### LS3 — Code generation + create + redirect · standard

**Verify:** `npm run typecheck && npm test` green; `curl -X POST localhost:3000/links` returns a
7-character code; `curl -i localhost:3000/<code>` returns 302 with the original URL; an unknown
code returns 404, not a crash.
**Commit:** `ls-step-3: code generation, create endpoint, and redirect`

```text
Wire the first end-to-end flow: create a short link, then follow it. The store from LS2 is done
and tested — use it as-is, do not modify it.

1. src/codes.ts — generateCode(): a 7-character code from an unambiguous alphabet
   (no 0/O/1/l/I). Export the alphabet as a named constant so the test can assert it.
...
```
````

**→ [See a complete example plan](examples/link-shortener-plan.md)** — tracker, ground rules,
worked steps, landmines, and deferred work, end to end.

## Two ideas the format rests on

**The document is the single source of truth for status.** Not a separate tracker, not the git
log, not the conversation. Each session flips its own row to ✅ in the same change as the code, so
the two can't drift. If the document and reality disagree, the document is the bug.

**One step = one commit = one reviewable unit.** There is a review boundary at every step (or, in
the autopilot edition, every phase). That boundary is the point — it's what keeps a long build
correctable instead of discovering at step twelve that step four went sideways.

---

## Editions

| Edition | Best for | Who commits | Model column | Size |
|---|---|---|---|---|
| **`stepbystep`** — default | Projects and features you want to review step by step | **You.** The agent stops after each step and hands you one block with both `git commit` and `git push` | Claude model names (Opus / Sonnet) | Full: SKILL.md + template + examples |
| **`stepbystep-portable`** | Teams, other agents, other model vendors | **Your choice per plan** — human commits, agent commits, or agent commits and pushes | Generic tiers (`standard` / `strong`) | Full |
| **`stepbystep-lite`** | Small features and fixes, about 3–8 steps | You, as in the default | Only when a step needs a stronger model | One file, built-in template |
| **`stepbystep-autopilot`** | Work with a strong test suite and CI | **The agent**, one commit per step and only when verify is green. It opens a PR per phase and stops at checkpoints | Per phase, since a session runs on one model | Full |

**How to choose:** start with the default. Reach for **lite** when the full plan would be longer
than the work, **portable** when the plan will be executed by other people or other agents, and
**autopilot** when your verify command is trustworthy enough to stand in for reviewing every
diff. The autopilot edition also adds **gates** (`auto` / `checkpoint`), a **phase prompt**, a
**PR description template**, and explicit **stop conditions**, so autonomy has hard edges.

Install one edition at a time. They share trigger phrases, so with several installed you should
invoke the one you want by name.

---

## Install

### Claude Code — plugin marketplace (recommended)

```bash
claude plugin marketplace add memoliertorerr/stepbystep
claude plugin install stepbystep@stepbystep
```

Swap the second command for `stepbystep-portable@stepbystep`, `stepbystep-lite@stepbystep`, or
`stepbystep-autopilot@stepbystep` to get another edition. Inside a session, `/plugin` does the
same through a menu. Updates arrive through `claude plugin marketplace update stepbystep`.

### Claude Code — as a plain skill

```bash
git clone https://github.com/memoliertorerr/stepbystep.git
cp -R stepbystep/skills/stepbystep ~/.claude/skills/
```

Copy any folder under `skills/` the same way. Use `.claude/skills/` inside a repo instead to
scope an edition to that one project.

### Codex

```bash
git clone https://github.com/memoliertorerr/stepbystep.git
mkdir -p ~/.agents/skills && cp -R stepbystep/skills/stepbystep ~/.agents/skills/
```

Or use a repo's own `.agents/skills/` to scope it to that repo. If Codex is already running,
restart it.

### Any other agent

Every edition is a self-contained folder of plain markdown with no paths outside itself. Put it
anywhere and point your agent's standing-instructions file at it: *"When asked to plan a project,
follow the skill at `<path>/SKILL.md`."*

---

## Use

Ask for it by name, or just describe the work:

| Where | How |
|---|---|
| Claude Code, installed as a plugin | `/stepbystep:stepbystep` (or `/stepbystep-lite:stepbystep-lite`, …) |
| Claude Code, installed as a skill | `/stepbystep` (or `/stepbystep-lite`, …) |
| Codex | `$stepbystep`, or pick it from `/skills` |
| Anywhere | "Scope this project", "break this into steps", "write me a build plan" |

The skill asks what it can't infer — what's out of scope, what must keep working, what it can't
verify on its own — then writes the plan, usually to `docs/BUILD_PLAN.md`. It handles **new
projects** (starts with scaffolding and a fresh repo) and **features on existing codebases**
(branches from `main`, inherits the project's conventions, adds a regression check).

Then you execute it, one fresh session at a time:

1. Open a new agent session in the repo.
2. Paste the plan's **opening prompt**. The session loads the docs, checks git state, restates
   the rules, and tells you which step is next.
3. Paste that step's prompt. The session does the work, runs the verify command, ticks the
   tracker, and hands back the diff (and, depending on the edition, a commit command or a commit).
4. Review. Repeat.

## What's in a plan

| Section | Why it's there |
|---|---|
| Scope blockquote | What's in, and explicitly what's **not** — scope creep is the format's main failure mode |
| Progress Tracker | Step / scope / model / status. Read first every session |
| Opening prompt | What a fresh session pastes before any step: load docs, check git state, restate the rules, report the next step, then wait |
| Who verifies what | Honest split of agent-checkable vs. human-only. Nothing implies verification that didn't happen |
| Ground Rules | Invariants for every step, so they aren't restated twelve times |
| Rollback | Undo commands, so nobody has to think under pressure |
| Steps | Each with a falsifiable verify line and a self-contained prompt |
| Deferred follow-ups | Out-of-scope things steps noticed, so they aren't buried |
| Landmines | Real friction that actually bit — not speculation |
| Later | Designed but deliberately unbuilt, so nobody half-builds it |

The lite edition keeps the scope, tracker, opening prompt, and steps, and folds the rest into a
short **Notes** section.

---

## Repository layout

```
.claude-plugin/marketplace.json     one plugin per edition, for /plugin installs
skills/
  stepbystep/                       default edition
    SKILL.md                        the workflow, principles, and step anatomy
    references/template.md          the literal fill-in skeleton
    references/examples.md          worked steps + a weak-vs-strong quality bar
  stepbystep-portable/              same shape, tool-neutral
  stepbystep-lite/SKILL.md          one file, template included
  stepbystep-autopilot/             same shape, autonomous
examples/link-shortener-plan.md     a complete plan, end to end
```

Agents load `SKILL.md` when the skill triggers and read the `references/` files only when they
need them, so the context cost stays small.

## Contributing

Issues and pull requests are welcome. A few rules keep the editions coherent:

- Each edition stays **self-contained**: nothing in it may point outside its own folder.
- The `name` in a `SKILL.md` matches its folder name, and the description stays under 1024
  characters.
- A change to a shared principle (the two core ideas, the verify-line contract, cold-session
  prompts) should be carried to every edition it applies to.
- After touching `.claude-plugin/marketplace.json`, run `claude plugin validate .`.

## About

stepbystep was made for agentic coding: it exists because AI agents work best with plans written
for them to execute rather than for people to interpret. It grew out of planning real projects
with AI coding agents, and was written and refined with their help.

## License

[MIT](LICENSE) © 2026 Mehmet Ali Ertorer
