# Worked examples and quality bar

Read this before drafting if you want a concrete sense of the difference between a step that
executes cleanly and one that produces a mess.

The examples below are illustrative and self-contained — they are here so you never need to go
hunting through other projects on the machine for a reference. Project names and paths in them
are stand-ins; copy the *shape*, not the specifics.

## Contents

- [A full worked step (new project)](#a-full-worked-step-new-project)
- [A worked step (feature on an existing project)](#a-worked-step-feature-on-an-existing-project)
- [Verify lines: weak vs strong](#verify-lines-weak-vs-strong)
- [Prompts: weak vs strong](#prompts-weak-vs-strong)
- [Tracker rows](#tracker-rows)
- [Landmines: real vs speculative](#landmines-real-vs-speculative)
- [Honest verification](#honest-verification)

---

## A full worked step (new project)

From a macOS disk-analysis tool whose central premise is that it never deletes anything — it
prints commands for a human to run. Note how the premise is restated at the exact points where
it could be violated, how the "do NOT" carries its reason, and how the annotation requirements
are specific enough to check.

````markdown
### RS6 — Simulator devices (reset) + SIP reporter · Sonnet 5

**Verify:** `npm run typecheck` + `npm test` green; on this Mac, bloated simulator devices
surface with `reclaim.kind: "reset"` and command `xcrun simctl erase <UUID>` (NOT delete), and
17 MB-class devices are filtered out by a size threshold; `/Library/Updates` surfaces as
`level: "blocked"` with a `blockedReason` and **no** reclaim command.
**Commit:** `rs-step-6: simulator-devices (reset-over-delete) + sip-reporter (blocked, no command)`

```text
Two detectors that prove two core concepts end-to-end: reset-over-delete, and
blocked-no-command.

1. detectors/simulatorDevices.ts — id "simulator-devices", cost "moderate".
   - `xcrun simctl list devices --json` to enumerate devices; size each device's data dir
     under ~/Library/Developer/CoreSimulator/Devices/<UUID> via ctx.dirSize.
   - Only surface devices above a threshold (e.g. > 200 MB) — in the audit every non-bloated
     device was ~17 MB; don't spam the list with them.
   - reclaim: xcrun simctl erase <UUID>, kind "RESET" — this keeps the device and drops the
     bloat. Say so explicitly in whatItIs/ifRemoved: "resets the device to factory state; the
     device itself stays, installed apps + data are cleared".

2. detectors/sipReporter.ts — id "sip-reporter", cost "fast".
   - For each target that exists AND ctx.isSipRestricted is true: emit level "blocked",
     blockedReason explaining SIP prevents removal even with sudo, NO reclaim field at all.
   - This detector's entire job is to stop the user (and any future contributor) wasting a
     round-trip trying to rm a restricted path. If a target isn't restricted after all, don't
     emit a blocked finding for it — hand it to whatever normal detector owns it, or skip.

Verify on this machine: erase (not delete) for sim devices; /Library/Updates blocked with no
command. Paste the relevant CLI output.

Run npm run typecheck and npm test, then npm run scan. Flip RS6 to ✅ in
docs/BUILD_PLAN.md in the same change, then show me the diff and a single block with BOTH the
git commit ("rs-step-6: simulator-devices (reset-over-delete) + sip-reporter (blocked, no
command)") AND git push. Do not commit.
```
````

What makes it work: real file paths and identifiers, a numeric threshold with the observation
that justifies it, the reason a rule exists ("stop the user wasting a round-trip"), an
instruction for the case where the assumption fails ("if a target isn't restricted after all"),
a request for pasted-back evidence, and the exact closing handoff.

---

## A worked step (feature on an existing project)

Same format, different pressures. This one adds a screen to a shipped mobile app. Notice what
changes: it reuses existing components by name instead of inventing them, it says which files
it must *not* touch and why, it carries a release constraint, and its verify line includes a
regression check — none of which a greenfield step needs.

````markdown
### FS3 — Saved-items list screen · Sonnet 5

**Verify:** `npm run typecheck`; the Saved tab renders items from `SavedContext` using the
existing `ItemCard` (no new card variant); empty state shows the shared `EmptyState` component,
not a bare screen; **nothing regressed** — the Home feed, pull-to-refresh, and deep-link
navigation still work. **Release-safe:** no new native dependency, no manifest change.
**Commit:** `fs-step-3: saved-items list screen`

```text
Build the Saved tab's list screen. The tab route and its placeholder already exist (FS1); the
persistence layer already exists (FS2). This step only builds the list UI.

1. app/(tabs)/saved.tsx — replace the FS1 placeholder body:
   - Read items via useSaved() from contexts/SavedContext.tsx (FS2). Do NOT read storage
     directly — the context exists so two screens can't hold divergent copies of the same
     state, which was a real bug in the downloads feature.
   - Render with the existing components/ItemCard.tsx, unchanged. If a card variant seems
     necessary, STOP and tell me — a second card variant is a design decision, not an
     implementation detail.
   - Empty state: components/EmptyState.tsx with copy "Nothing saved yet."
   - List bottom padding must include TAB_BAR_CLEARANCE from lib/tabBar.ts — the floating tab
     bar is absolutely positioned and does not reserve its own space, so the last row hides
     behind it otherwise.

2. Do NOT touch contexts/SavedContext.tsx (FS2 owns it, it's already verified) or the Home
   feed. Keeping this diff to one screen is what makes it reviewable.

3. Release constraint: this must stay shippable without a native rebuild. No new dependencies.
   If something here seems to need one, that's out of scope — STOP and raise it.

Verify: npm run typecheck; run the app; save an item from Home, open the Saved tab, confirm it
appears; remove it, confirm the empty state renders. Then re-check the Home feed and
pull-to-refresh still behave. Screenshot the Saved tab back to me.

Run npm run typecheck. Flip FS3 to ✅ in docs/SAVED_ITEMS_PLAN.md in the same change, then show
me the diff and a single block with BOTH the git commit ("fs-step-3: saved-items list screen")
AND git push. Do not commit.
```
````

The feature-mode tells: it names existing files it depends on and the step that produced them,
it justifies a constraint with a bug that actually happened, it draws an explicit line around
the diff, it states the release constraint, and its verification includes re-checking things
this step never touched.

---

## Verify lines: weak vs strong

A verify line is checked by someone who wasn't in the planning conversation. It must be
falsifiable.

| Weak | Strong |
|---|---|
| The detectors work | `npm run scan` prints findings with last-used dates sourced from tool metadata (not mtime), and sizes within ~10% of the audit table |
| Tests pass | `npm test` green, **including** tests asserting the exec allowlist rejects `rm`, `simctl delete`, and `npm cache clean` **without spawning** |
| The UI looks right | The list renders grouped by level with a colored accessory per group; the reclaim command appears **only** as a Copy action — grep confirms no `child_process` import in the extension |
| Search is implemented | Typing 2+ chars fires one debounced request after 450ms; results render as cards; clearing the field restores the recent list |

The pattern: name the command, name the observable, and name the specific thing that would be
wrong if someone implemented it carelessly.

---

## Prompts: weak vs strong

**Weak** — assumes context, invites drift:

```text
Add the caching layer we talked about. Make it fast and clean it up if it gets too big.
Update the docs.
```

Nothing here survives a cold session: "we talked about" refers to a conversation that no longer
exists, "fast" and "too big" are unmeasurable, "the docs" is unnamed.

**Strong** — self-contained, bounded, explains itself:

```text
Add a scan-result cache so the UI paints instantly instead of waiting for a full scan.

1. core/src/store.ts: write a completed ScanResult to
   ~/Library/Caches/<app>/last-scan.json (serialize Dates as ISO strings; version the file
   with a schema tag so a future diff mode can detect shape changes). Read it back typed.
   This is pure IO of our own data — no reclaim semantics, no exec.

2. Raycast: on command open, load last-scan.json first for an instant list, THEN run scan() in
   the background and replace when it finishes. If no cache exists yet, scan directly.

3. Do NOT build diff/history — that's deferred. Just prove in a test that two ScanResults can
   be compared by finding id, so the deferred work has a stable contract to build on.

If the cache path needs to differ on Intel vs Apple Silicon, tell me rather than guessing.
```

Every instruction names a real path, the parenthetical explains *why* (a future consumer needs
schema detection), the "Do NOT" carries its reason, and genuine ambiguity is routed back as a
question instead of a guess.

---

## Tracker rows

Scope text should say what the step *delivers*, not what it touches:

| Weak | Strong |
|---|---|
| Types work | **The contract** — `types.ts` (`Finding`, `Detector`, `DetectorContext`), registry, engine scan loop (streams findings, gates slow detectors). No detectors yet |
| Detectors | `simulatorDevices` (reset-over-delete via `simctl erase`) + `sipReporter` (`/Library/Updates` → blocked, no command) |
| Cleanup | Scan-result cache + `--json` — responsiveness + the foundation the deferred diff mode reads. Diff itself NOT built |

Bold the milestone steps so the phases are visible when skimming the table.

---

## Landmines: real vs speculative

Only include friction you actually hit or confirmed. Speculation trains readers to skim.

**Real** (each was discovered the hard way, and names the step that handles it):

- **SIP restricted blocks even `sudo rm`.** `/Library/Updates` is SIP-restricted — `sudo rm`
  fails. The manual session wasted a full round-trip offering a command for it. That's why
  `sipReporter` reports these as `blocked` with **no command**.
- **`atime` is unreliable on macOS.** Frequently disabled or updated lazily, so an
  atime-derived "last used" can be wildly wrong. Prefer tool-reported metadata, fall back to
  mtime, and always label the source.
- **APFS clones make apparent size ≠ reclaimable size.** Snapshots share blocks with the live
  filesystem, so deleting one frees far less than its nominal size. Never overpromise
  reclaimed space.

**Speculative** (cut these):

- The API might change in a future version
- Performance could become a problem at scale
- There may be edge cases with unicode

---

## Honest verification

The most common way this format fails is a plan that implies verification it can't perform.
State the boundary plainly:

> Claude self-verifies everything the CLI can print (it runs on this Mac). The **only** checks
> that fall to you are the Raycast *visual* render and the final "do the annotations read
> right?" judgment — Claude will always tell you exactly what to look at.

And inside the step that hits the boundary:

> I cannot see the Raycast UI render — after typecheck passes and I've confirmed the code has
> no execute path, run `ray develop` yourself and tell me whether the list, the detail, and the
> copy action look right. Tell me exactly what to click.

Compare with the failure mode: a step whose Verify says "the UI renders correctly" and whose
session reports "✅ verified" having never seen a pixel.
