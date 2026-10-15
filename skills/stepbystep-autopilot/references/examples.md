# Worked examples and quality bar (autopilot)

Read this before drafting if you want a concrete sense of what a step needs to look like when no
human reads it before it is committed.

The examples are illustrative and self-contained, so you never need to go hunting through other
projects on the machine for a reference. Project names and paths are stand-ins; copy the
*shape*, not the specifics.

## Contents

- [An auto step](#an-auto-step)
- [A checkpoint step](#a-checkpoint-step)
- [A phase prompt, filled in](#a-phase-prompt-filled-in)
- [Choosing the gate](#choosing-the-gate)
- [Verify lines: weak vs strong](#verify-lines-weak-vs-strong)
- [Prompts: weak vs strong](#prompts-weak-vs-strong)
- [Stop conditions: weak vs strong](#stop-conditions-weak-vs-strong)
- [Landmines: real vs speculative](#landmines-real-vs-speculative)
- [Honest verification](#honest-verification)

---

## An auto step

From a macOS disk-analysis tool whose premise is that it never deletes anything — it prints
commands for a human to run. Everything this step produces is visible in CLI output and tests,
so it can run unattended. Note how the verify line is specific enough to stand in for a
reviewer, and how the one judgment call ("what if the path isn't restricted?") is decided in the
prompt instead of left as a question.

````markdown
### RS6 — Simulator devices (reset) + SIP reporter · standard · auto

**Verify:** `npm run typecheck` + `npm test` green, including new tests that (a) a simulator
device above 200 MB yields `reclaim.kind: "reset"` with command `xcrun simctl erase <UUID>`,
never `delete`; (b) a device below the threshold yields no finding; (c) a SIP-restricted path
yields `level: "blocked"` with a `blockedReason` and **no** `reclaim` field.
**Commit:** `rs-step-6: simulator-devices (reset-over-delete) + sip-reporter (blocked, no command)`

```text
Two detectors that prove two core concepts end-to-end: reset-over-delete, and
blocked-no-command.

1. detectors/simulatorDevices.ts — id "simulator-devices", cost "moderate".
   - `xcrun simctl list devices --json` to enumerate devices; size each device's data dir
     under ~/Library/Developer/CoreSimulator/Devices/<UUID> via ctx.dirSize.
   - Only surface devices above 200 MB — in the audit every non-bloated device was ~17 MB;
     don't spam the list with them.
   - reclaim: xcrun simctl erase <UUID>, kind "RESET" — this keeps the device and drops the
     bloat. Say so in whatItIs/ifRemoved: "resets the device to factory state; the device
     itself stays, installed apps + data are cleared".

2. detectors/sipReporter.ts — id "sip-reporter", cost "fast".
   - For each target that exists AND ctx.isSipRestricted is true: emit level "blocked",
     blockedReason explaining SIP prevents removal even with sudo, NO reclaim field at all.
   - If a target turns out not to be restricted, skip it — do not emit a blocked finding and
     do not invent a reclaim command for it. That case is decided; it is not a reason to stop.

3. Tests: stub simctl and dirSize in the test context; do NOT call the real simctl from tests.
   Cover the three cases in the Verify line.

Do NOT touch the engine or the registry — RS2 owns them. If wiring a detector seems to need a
change there, that is a stop condition: stop and report what it needs.

Run npm run typecheck && npm test, then npm run scan and include the two new detectors' lines
in your report. If it is green, flip RS6 to ✅ in docs/BUILD_PLAN.md and commit everything as
one commit: "rs-step-6: simulator-devices (reset-over-delete) + sip-reporter (blocked, no
command)". Paste the verify output and the output of `git log -1 --stat`. If it is still red
after two honest fix attempts, stop and report — never commit red, never weaken a check to get
green.
```
````

What makes it safe to run unattended: every claim in the verify line is a test someone could
read in the diff; the only ambiguous case is resolved in the prompt; the boundary with earlier
steps is a named stop, not a request to "ask me".

---

## A checkpoint step

A screen in a shipped mobile app. The code can be typechecked and unit-tested, but whether the
screen *looks right* is something only a human with a device can judge — so the step is a
checkpoint. The agent commits it like any other step, then pushes and stops.

````markdown
### FS3 — Saved-items list screen · standard · checkpoint

**Verify:** `npm run typecheck && npm test` green; a render test asserts the list uses
`ItemCard` and the empty state renders `EmptyState` with "Nothing saved yet."; **nothing
regressed** — Home feed and deep-link tests still pass. **Release-safe:** no new native
dependency, no manifest change.
**Human check:** on a device, save an item from Home, open the Saved tab, confirm it appears
above the floating tab bar; remove it, confirm the empty state.
**Commit:** `fs-step-3: saved-items list screen`

```text
Build the Saved tab's list screen. The tab route and its placeholder exist (FS1); the
persistence layer exists (FS2). This step only builds the list UI.

1. app/(tabs)/saved.tsx — replace the FS1 placeholder body:
   - Read items via useSaved() from contexts/SavedContext.tsx. Do NOT read storage directly —
     the context exists so two screens can't hold divergent copies of the same state.
   - Render with the existing components/ItemCard.tsx, unchanged. If a new card variant seems
     necessary, that is a design decision: stop and report.
   - Empty state: components/EmptyState.tsx with copy "Nothing saved yet."
   - List bottom padding must include TAB_BAR_CLEARANCE from lib/tabBar.ts — the floating tab
     bar doesn't reserve its own space, so the last row hides behind it otherwise.

2. Do NOT touch contexts/SavedContext.tsx or the Home feed.

3. Release constraint: no new dependencies. If something seems to need one, stop.

Run npm run typecheck && npm test. If it is green, flip FS3 to ✅ in
docs/SAVED_ITEMS_PLAN.md and commit everything as one commit: "fs-step-3: saved-items list
screen". Paste the verify output and the output of `git log -1 --stat`. If it is still red
after two honest fix attempts, stop and report — never commit red, never weaken a check to get
green.
This is a checkpoint: push the branch, update the PR, tell me exactly what to check and where,
then stop.
```
````

The checkpoint doesn't change what the agent builds — it changes where the session ends. The
**Human check** line is copied into the PR's "Needs human check" list verbatim.

---

## A phase prompt, filled in

```text
Execute steps FS1 → FS3 from docs/SAVED_ITEMS_PLAN.md, in order.

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

Here FS3 is both the last step and a checkpoint, so the phase ends with one push, one PR, and
one human check — the arrangement to aim for.

---

## Choosing the gate

| The step… | Gate | Why |
|---|---|---|
| Adds a function and its tests | auto | The tests are the review |
| Defines the contract every later step builds on | checkpoint | Mechanically correct can still be the wrong shape, and everything after inherits it |
| Changes something visual | checkpoint | No test sees the pixels |
| Touches auth, payments, or data deletion | checkpoint | Cost of a wrong-but-green commit is too high to find in a PR later |
| Runs a migration or deploy | checkpoint, and usually its own phase | Irreversible; also a stop condition |
| Refactors under full test coverage | auto | Behavior is pinned by the suite |

If more than about a third of the steps are checkpoints, the plan wants the standard edition.

---

## Verify lines: weak vs strong

Under autopilot, the verify line of an `auto` step is the whole review until the PR. It must be
falsifiable by someone who wasn't there.

| Weak | Strong |
|---|---|
| The detectors work | `npm test` green, including tests that a 250 MB device yields `simctl erase` and a 17 MB one yields nothing |
| Tests pass | `npm test` green **including** tests asserting the exec allowlist rejects `rm`, `simctl delete`, and `npm cache clean` **without spawning** |
| Search is implemented | A test asserts typing 2+ chars fires exactly one debounced request after 450 ms, and clearing the field restores the recent list |
| The migration is safe | `npm run migrate:dry` prints exactly the three expected statements; the step is a **checkpoint** and the real run is a stop condition |

The pattern: name the command, name the observable, and name the specific thing that would be
wrong if someone implemented it carelessly — as a test, wherever possible.

---

## Prompts: weak vs strong

**Weak** — assumes context, and leaves ambiguity for an unattended session to guess at:

```text
Add the caching layer we talked about. Make it fast and clean it up if it gets too big.
Ask me if anything is unclear.
```

"Ask me" has no one to ask in the middle of a phase; the session will guess.

**Strong** — self-contained, bounded, with every judgment call resolved or turned into a stop:

```text
Add a scan-result cache so the UI paints instantly instead of waiting for a full scan.

1. core/src/store.ts: write a completed ScanResult to
   ~/Library/Caches/<app>/last-scan.json (serialize Dates as ISO strings; include a
   schemaVersion field so a future diff mode can detect shape changes). Read it back typed.
   If the file exists but its schemaVersion doesn't match, ignore it and rescan — do not
   migrate it.

2. On command open, load last-scan.json first for an instant list, THEN run scan() in the
   background and replace when it finishes. If no cache exists, scan directly.

3. Do NOT build diff/history — that's deferred. Prove in a test that two ScanResults can be
   compared by finding id, so the deferred work has a stable contract.

If the cache directory isn't writable in the test environment, that is a stop condition:
report it rather than writing somewhere else.
```

---

## Stop conditions: weak vs strong

| Weak | Strong |
|---|---|
| Stop if something goes wrong | Stop if verify is still red after two honest fix attempts, and report what fails and what was tried |
| Be careful with tests | Never skip, delete, or loosen a test to get green; if that's the only path, stop |
| Don't go out of scope | If the step needs a file its prompt doesn't name, a new dependency, or a schema change, record it under Deferred follow-ups and stop unless the step is completable without it |
| Ask before risky things | Stop before anything under `migrations/`, any change to CI config, and any command that deletes data |

A stop condition is good when the session can tell, without judgment, whether it has fired.

---

## Landmines: real vs speculative

Only include friction you actually hit or confirmed. Speculation trains readers to skim.

**Real** (each was discovered the hard way, and names what handles it):

- **SIP restricted blocks even `sudo rm`.** `/Library/Updates` is SIP-restricted. That's why
  `sipReporter` reports these as `blocked` with **no command**.
- **`atime` is unreliable on macOS.** Frequently disabled or updated lazily. Prefer
  tool-reported metadata, fall back to mtime, and always label the source.
- **CI runs on Linux, development on macOS.** Path-case and `xcrun` differences made a green
  local run red in CI. Steps that shell out stub the command in tests.

**Speculative** (cut these):

- The API might change in a future version
- Performance could become a problem at scale

---

## Honest verification

The most common way this format fails is a plan that implies verification it can't perform —
and under autopilot, a false "verified" ships straight into a PR. State the boundary plainly:

> The agent self-verifies everything the CLI and the test suite can show. Every visual or
> on-device check is yours; those steps are checkpoints, and the PR lists exactly what to look
> at under **Needs human check**.

And inside the report from a checkpoint step:

> Committed and pushed FS3. I cannot see the rendered screen. On a device: save an item from
> Home, open the Saved tab, and check that the last row clears the floating tab bar. Then
> remove it and check the empty state. Stopping here.

Compare with the failure mode: an `auto` step whose verify says "the UI renders correctly",
committed and followed by three more steps, with no one having seen a pixel.
