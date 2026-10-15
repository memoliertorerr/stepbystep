# Link Shortener — v1 Build Plan

> **This is an example of what the `stepbystep` skills produce**, written with the
> `stepbystep-portable` conventions (generic model tiers, human-commits). It is a complete,
> realistic plan for a small greenfield service, trimmed to five steps so the whole shape fits on
> one page. Nothing here is meant to be built — it exists so you can see the format end-to-end
> before adopting it.
>
> Building a self-hosted URL shortener: a small HTTP service that stores short codes, redirects
> on hit, and counts hits.
>
> **In scope for v1:** the storage layer, code generation, the redirect path, a create endpoint,
> and hit counting.
>
> **Explicitly NOT in scope** (designed, deferred — see **Later**): user accounts, custom
> vanity codes, link expiry, an admin UI, analytics beyond a raw hit count.
>
> **One step = one commit = one reviewable unit.** Commit convention for this plan: the agent
> hands over the commit command and does not commit — see Ground Rule 4.

---

## Progress Tracker

> Each session flips its own row to ✅ as the final action of the step, in the same commit as
> the change. Read this table at the start of a session to know where to resume.

| Step | Scope | Model | Status |
|------|-------|-------|--------|
| LS1 | Scaffolding — project skeleton, strict TS config, test runner, `dev`/`typecheck`/`test` scripts, health endpoint | standard | ✅ done |
| LS2 | **The contract** — `types.ts` (`Link`, `LinkStore`), the in-memory store implementation, and its test suite | **strong** | ✅ done |
| LS3 | **First runnable** — code generation + `POST /links` + `GET /:code` redirect, wired end-to-end | standard | ⬜ todo |
| LS4 | Hit counting — atomic increment on redirect, `GET /links/:code` stats endpoint | standard | ⬜ todo |
| LS5 | SQLite-backed `LinkStore` swapped in behind the same interface; in-memory kept for tests | standard | ⬜ todo |

**Overall status: LS2 done — the store interface and its in-memory implementation are tested and
stable. LS3 is next and is the first runnable milestone.** The plan front-loads the storage
contract (LS2) because every later step is written against it, then reaches a working
create-and-redirect flow at LS3 so the thing can actually be used well before persistence lands.
LS5 proves the interface was right by swapping the implementation without touching any caller.

**Model note:** most steps are **standard** — once the contract exists, each is ordinary
endpoint work against a settled interface. **LS2 is strong.** It designs `LinkStore`, the
boundary every later step and both storage implementations depend on; a wrong shape there is
expensive to unwind, and LS5's whole purpose is to prove it was right.

**Session split** — natural review checkpoints:
1. **LS1 → LS2** — foundation. Stop when the store tests are green and the interface feels
   settled; nothing else should be built until it is.
2. **LS3** — first runnable milestone. Actually create a link and follow the redirect before
   continuing.
3. **LS4 → LS5** — hit counting, then the persistence swap.

Never start a step with uncommitted changes in the tree.

---

## How To Run This

1. Open a **new session** in the repo. The project's standing-instructions file auto-loads.
2. **Paste the Opening prompt below first.** Wait for its green light.
3. Paste the **PROMPT** for the next unfinished step.
4. The session does the work, runs `npm run typecheck && npm test`, ticks its tracker row, and
   shows a diff + the commit command.
5. You review, run it where relevant, and commit if it looks right.

### Opening prompt — every step

```text
You are building a link shortener in this repo.
Before any work:
1. Read the project's standing-instructions file and docs/BUILD_PLAN.md, in full.
2. Run `git status` and `git branch --show-current`. Confirm the tree is clean and the
   branch is feature/shortener-v1. If it doesn't exist, tell me the commands from
   "0. Before Starting" and STOP until I've run them.
3. Restate in one line the commit convention from Ground Rule 4.
4. Read the Progress Tracker and tell me which step is next (the first ⬜ todo).
Then wait. Do not implement anything until I paste the step prompt.
```

### Who verifies what

| Check | Who | "Green" looks like |
|---|---|---|
| `npm run typecheck` | agent | No errors, clean exit |
| `npm test` | agent | All tests pass, including the store contract suite |
| Redirect works in a real browser | **you** | Visiting a short URL lands on the target with a 302 — the agent can verify the status code with `curl` but not the browser's actual navigation |
| Nothing regressed | agent | Existing endpoint tests still pass after each step |

The agent self-verifies everything reachable from `curl` and the test suite. Real browser
behavior is yours; the agent will say exactly which URL to open.

---

## Ground Rules

1. **Branch:** `feature/shortener-v1`. Tag `pre-shortener` right after the first commit.
2. **One step = one commit**, message prefixed `ls-step-N:`.
3. **Verify before every commit.** `npm run typecheck && npm test` must be green. Never commit red.
4. **Commit convention:** the agent never runs `git commit` or `git push`. It shows the diff and
   hands over the commit command, then stops. *(This is this project's choice — the format
   supports agent-commits conventions too.)*
5. **`types.ts` is the contract.** `Link` and `LinkStore` are the boundary between the
   HTTP layer and every storage implementation. A change there ripples to both — update all
   affected code in the same step, never let them drift.
6. **Short codes must be URL-safe and unambiguous.** No characters that look alike (`0`/`O`,
   `1`/`l`) — a code that can't be read aloud correctly is a support burden.
7. **Every step:** ask if genuinely uncertain, and update this tracker plus any affected docs.

### Rollback

| Situation | Command |
|---|---|
| Step wrong, uncommitted | `git restore . && git clean -fd` |
| A committed step is bad | `git revert <sha>` |
| Abandon the whole thing | `git reset --hard pre-shortener` |

---

## 0. Before Starting

```bash
git status                               # must be clean
git checkout main && git pull
git checkout -b feature/shortener-v1
git tag pre-shortener
```

---

## Steps

### LS3 — Code generation + create + redirect · standard

**Verify:** `npm run typecheck && npm test` green; `curl -X POST localhost:3000/links -d
'{"url":"https://example.com"}' -H 'content-type: application/json'` returns a 7-character code;
`curl -i localhost:3000/<code>` returns `302` with a `Location` header of the original URL; an
unknown code returns `404`, not a crash or a redirect to `undefined`.
**Commit:** `ls-step-3: code generation, create endpoint, and redirect`

```text
Wire the first end-to-end flow: create a short link, then follow it. The store from LS2 is done
and tested — use it as-is, do not modify it.

1. src/codes.ts — generateCode(): a 7-character code from an unambiguous alphabet
   (no 0/O/1/l/I, per Ground Rule 6). Export the alphabet as a named constant so the test can
   assert no ambiguous character ever appears. 7 chars of a ~57-char alphabet is ~10^12 codes;
   collisions are rare but not impossible, so generateCode does NOT check uniqueness — the
   caller retries on collision (step 2).

2. src/routes/links.ts — POST /links:
   - Body: { url: string }. Reject anything that isn't a valid http/https URL with 400 and a
     JSON error. Do NOT accept other schemes — a javascript: URL in a redirect is an XSS vector.
   - Generate a code, attempt store.create(); on a collision retry up to 5 times, then 500.
   - Respond 201 with { code, url, shortUrl }.

3. src/routes/redirect.ts — GET /:code:
   - Look the code up. Found: 302 to the stored URL. Not found: 404 with a plain message.
   - Do NOT count hits here yet — that's LS4, kept separate so this diff stays reviewable.
   - Mount this route LAST so it can't shadow /links or /health.

4. Tests: generateCode produces only allowed characters and the right length; POST rejects a
   non-URL and a javascript: URL; a created code redirects to its target; an unknown code 404s.

If mounting order fights the router and /:code starts swallowing /health, stop and tell me
rather than renaming the health route to dodge it.

Run npm run typecheck && npm test, then start the server and run the curls in the Verify line
above and paste me the output. Flip LS3 to ✅ in docs/BUILD_PLAN.md in the same change, then show
me the diff and the commit command for this step (`ls-step-3: code generation, create endpoint,
and redirect`), following the commit convention in Ground Rule 4.
```

---

### LS4 — Hit counting · standard

**Verify:** `npm run typecheck && npm test` green; hitting a code twice then calling
`GET /links/<code>` reports `hits: 2`; the redirect still returns 302 in the same latency ballpark
(counting must not block the redirect); an unknown code's stats endpoint returns 404.
**Commit:** `ls-step-4: hit counting and stats endpoint`

```text
Count redirect hits and expose them.

1. Extend LinkStore in types.ts with incrementHits(code): Promise<void> and include hits on the
   Link type. Update the in-memory implementation AND its test suite in this same step — the
   interface and its implementations must never drift (Ground Rule 5).

2. In the redirect route, increment the count. Do not await the increment before responding —
   a redirect that waits on a write is a slow redirect, and a lost count matters far less than
   latency here. Log a failed increment rather than surfacing it to the visitor.

3. GET /links/:code returns { code, url, hits, createdAt }, or 404. This must be registered
   BEFORE the catch-all /:code redirect route.

4. Tests: two hits produce hits === 2; stats 404s for an unknown code; a store whose increment
   rejects still returns a working redirect.

Run npm run typecheck && npm test. Flip LS4 to ✅ in docs/BUILD_PLAN.md in the same change, then
show me the diff and the commit command for this step (`ls-step-4: hit counting and stats
endpoint`), following the commit convention in Ground Rule 4.
```

---

## Deferred follow-ups (cross-step, live tracker)

> Things a step noticed but was out of its own scope. Strike through when handled; don't delete.

- **No rate limiting on `POST /links`** (surfaced by LS3). An open shortener is an open redirect
  farm for spammers. Not built in v1 because it needs a decision on the limiting strategy
  (per-IP, per-key, or an auth requirement) — that decision belongs with the accounts work in
  **Later**.
- **Collision retry count is a magic number** (surfaced by LS3). Five is a guess; nobody has
  measured the real collision rate. Fine at current scale, worth revisiting if code length ever
  shrinks.

---

## Landmines

- **Route order matters and fails silently.** `GET /:code` matches *everything*, so registering
  it before `/health` or `/links/:code` silently swallows those routes and they 404 in ways that
  look like application bugs. Mount the catch-all last (LS3 step 3, LS4 step 3).
- **`javascript:` and `data:` URLs are an XSS vector.** A shortener that redirects to an
  arbitrary scheme becomes an attack tool. Validate for http/https only, at creation time, not
  at redirect time — by redirect time the bad URL is already stored.
- **Awaiting the hit-count write makes redirects slow.** The whole product is a fast redirect;
  a synchronous counter write puts a database round-trip in the critical path for no user-facing
  benefit. Fire and log (LS4 step 2).
- **Ambiguous characters in codes create support load.** `0`/`O` and `1`/`l`/`I` are
  indistinguishable in many fonts, and short links get read aloud and copied by hand. This is
  why the alphabet is a constant the tests assert on, not an inline string.

---

## Later — designed, NOT built in v1

| Item | Design | What it needs |
|---|---|---|
| **User accounts** | Links gain an `ownerId`; `LinkStore` queries filter by it. | Auth, a users table, and a migration for existing ownerless links. |
| **Vanity codes** | `POST /links` accepts an optional `code`; the same collision path rejects duplicates. | A reserved-word list so nobody claims `/health` or `/links`. |
| **Link expiry** | An `expiresAt` on `Link`; the redirect treats expired as 404. | A sweep job, plus a decision on whether expired codes are reusable. |
| **Rate limiting** | Per-IP bucket on `POST /links`. | The strategy decision noted in Deferred follow-ups. |

**When Later work starts:** rate limiting first — it's the smallest change and the only one that
is a live abuse risk while the service is public. Accounts next, since vanity codes and
per-user limits both depend on them.
