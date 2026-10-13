# vckscribble — Build Plan

A Skribbl.io-style multiplayer drawing-and-guessing game, with three deviations
from the original:

1. The drawer gets 3 suggested words **and** can write their own if none appeal.
2. The dictionary is corporate slang, not generic nouns.
3. The dictionary can be extended by the owner — via direct SQL, plus a small
   token-protected endpoint added at the very end of the build.

Web only, desktop-first. No admin UI, no owner accounts, no login. See §7.

**Repos** — frontend and backend are deployed and versioned separately:

| Repo | Contents |
|---|---|
| [`vckscribble`](https://github.com/shabbir642/vckscribble) | this plan (umbrella/docs) |
| [`vckscribble-be`](https://github.com/shabbir642/vckscribble-be) | Node + Socket.IO + SQLite. **Owns the socket contract.** |
| [`vckscribble-fe`](https://github.com/shabbir642/vckscribble-fe) | React + Vite. Vendors the contract. |

Locally they sit at `vckscribble/be` and `vckscribble/fe`, each an independent
git repo, gitignored by the umbrella.

---

## 1. Architectural consequences of the customizations

Worth stating up front, because each one rules out an "obvious" simpler choice.

**Requirement 3 needs persistence, but no longer needs a UI.** The original
Skribbl is entirely ephemeral — rooms live in memory and die with the process.
A curated dictionary cannot, and neither can the record of words players invent
during play. That justifies a database. It does *not* justify an admin panel:
a single-operator system where the operator can run SQL is already a complete
management interface. Rooms, strokes, and scores stay in memory and remain
disposable.

**Requirement 2 breaks single-word assumptions.** Corporate slang is mostly
*phrases* — `circle back`, `low-hanging fruit`, `boil the ocean`. Skribbl's
guess matching, letter-reveal hints, and masked-word display all assume one
contiguous token. Phrase handling is cheap if designed in from the start and
genuinely painful to retrofit, so it is specified in §4 and §6 rather than
left to implementation.

**Requirement 2 also redefines "difficulty".** For generic nouns, difficulty
tracks word obscurity. For corporate slang it tracks *drawability*, which is
nearly uncorrelated with length or familiarity: `synergy` is short and
universally known and almost impossible to draw, while `drinking from the
firehose` is long and obscure and draws itself. So difficulty is a
hand-authored drawability rating seeded per word, then corrected against real
guess-rate data — queried with SQL (§7).

**Separate repos remove the compile-time protocol guarantee.** A monorepo can
put the socket contract in a package both sides import, making client/server
drift a build error. Across two repos that is no longer free. Rather than
duplicate the types by hand and hope, the BE owns the contract and the FE
vendors a generated copy; FE CI fails when the two diverge. The guarantee moves
from the compiler to CI, which is weaker — drift is caught at push, not at
keystroke — but it is the honest cost of the split, not something to paper over.

**Requirement 1 cannot be pre-moderated.** A custom word is used *immediately* —
the round starts the instant it is submitted, so there is no window for review
before it reaches players. Moderation is therefore synchronous (validation +
profanity filter at submit time) with an in-game report button as the backstop.
Owner review happens *after* play, when deciding whether a word earns a
permanent place in the dictionary.

---

## 2. Stack

| Layer | Choice | Why |
|---|---|---|
| Language | TypeScript throughout | One shared event-contract module; wire protocol cannot drift between client and server |
| Client | React + Vite | Fast HMR; canvas work is plain DOM API regardless of framework |
| Server | Node + Express + Socket.IO | Socket.IO gives rooms, acks, and reconnection handling for free |
| DB | SQLite + Drizzle | One file, zero ops, and `sqlite3 game.db` *is* the admin panel. Managed Postgres would be overhead for one table of words |
| Deploy | Single container, Fly.io or Railway | WebSocket-friendly with a persistent volume. **Not** Vercel/Lambda — serverless cannot hold socket connections |

### Crossing the repo boundary

`vckscribble-be/src/contract/index.ts` is the single source of truth: socket
event signatures, domain types, and the guess-normalization function. The
frontend vendors it to `src/generated/contract.ts` via `pnpm contract:sync`,
which resolves from `$CONTRACT_SOURCE`, then a sibling `../be` checkout, then
the BE repo's raw GitHub URL — so it works cloned together or alone.

`pnpm contract:check` fails on drift and runs both in CI and in `pnpm build`.
Normalization ships inside the contract specifically so the client can preview
"is this a valid guess" with exactly the server's logic; the server stays
authoritative regardless.

Upgrade path if this gets tiresome: publish the contract to a private registry
and depend on it by version. Not worth the ceremony at this size.

---

## 3. Data model

```
words
  id, text, category, drawability(1-3), enabled,
  source('seed'|'owner'|'player'), times_drawn, times_guessed, created_at

suggestions
  id, text, submitted_by, room_code, created_at, times_used
```

The seed dictionary (~200 entries) lives in `server/src/db/words.seed.ts` under
version control, not as raw INSERTs — so dictionary changes are reviewable in
diffs. Seeding is idempotent on `text`, runs on boot, and never overwrites rows
you have edited by hand.

`suggestions` stays a separate table from `words`. Player-submitted text is
untrusted until promoted; mixing it into the live dictionary behind a status
flag invites a query that forgets to filter. Without an admin UI there is no
`status` column — promotion *is* the INSERT, so a row's presence in `words` is
the only state that matters.

`times_drawn` / `times_guessed` accumulate during play with no UI consumer.
They exist for the retirement query in §7.

---

## 4. Word selection and the custom-word path

Round start → server picks 3 candidates: weighted by drawability, excluding
every word already drawn this game, soft-avoiding recently drawn across all
rooms so popular words don't dominate.

The drawer sees the 3 candidates plus a **"Write my own"** field. On submit:

- normalize (trim, collapse whitespace)
- length bounds (3–40 chars), must contain a letter
- profanity filter, synchronous
- per-player rate limit — prevents `aaaaa` every round
- reject if it duplicates a disabled word (it was retired for a reason)

A 15s selection timer auto-picks a candidate if the drawer stalls, so one AFK
player cannot freeze the room.

Every accepted custom word is written to `suggestions`. **This is what links
requirement 1 to requirement 3:** play itself generates your candidate
vocabulary, and a SQL query is where you promote what worked. A word submitted
by several different players *and* actually guessed is a strong candidate —
which is why `suggestions` tracks `times_used`.

---

## 5. Drawing sync

**The stroke log is the source of truth, not the bitmap.** Each stroke is
`{ id, drawerId, color, size, tool, points[] }`. The client renders by replaying
the log into the canvas, rather than accumulating pixels.

This costs a little performance and buys three things that are otherwise very
hard: undo (drop the last stroke, re-render), late-joiner sync (ship the log,
they see the drawing mid-round), and round replay/export later.

Performance is handled by caching committed strokes to an offscreen canvas and
only re-rendering live on undo/clear. Pointer events batch per animation frame
rather than emitting per-event. Stroke count is capped per round as a memory
guard.

Tools: brush (3 sizes), 30-colour palette, flood fill, eraser, undo, clear.

**Fill deserves a note**, because it is the one tool that is not itself vector
data. It is stored as a seed point plus a colour, and executed as a scanline
flood fill against the bitmap *as it stands at that point in the log*. That is
what keeps it compatible with a vector source of truth: replaying strokes in
order reproduces the same enclosed regions, because each fill sees exactly the
strokes that preceded it. Two consequences fall out of this and are intended —
an unenclosed region floods the whole canvas (a gap in an outline "leaks", as in
the original game), and matching carries a tolerance so antialiased stroke edges
do not halo.

---

## 6. Guessing, hints, and scoring

**Normalization** (in `shared/`, used by both sides): lowercase, strip
diacritics and punctuation, collapse internal whitespace. `Circle-Back` and
`circle back` both match `circle back`. Articles are *not* stripped — too
lenient, and it would make `the ask` and `ask` identical.

**Matching:** exact match on normalized form = correct. Levenshtein distance
within 1 (2 for phrases over 15 chars) = "you're close!" shown privately to
that guesser only.

**Letter reveal, phrase-aware:** the mask renders word structure from the start
(`_ _ _ _ _ _   _ _ _ _`) because word-count and lengths are structural, not a
hint. Letters then reveal progressively as the timer drains, capped at ~35% of
non-space characters, distributed across *all* words in the phrase rather than
front-loading the first one.

**Leak prevention:** the drawer cannot post to chat, and players who have
already guessed correctly move to a separate post-guess channel — otherwise the
first correct guesser trivially spoils the round.

**Scoring:** guessers earn by time remaining; the drawer earns per correct
guess, so drawing clearly is rewarded over drawing obscurely.

---

## 7. Word management — SQL now, endpoint last

No admin panel, no auth, no owner account. Two mechanisms:

### 7a. SQL — available from M1, free

Because the dictionary is a table, it is manageable from the moment the schema
exists. There is nothing to build and nothing to wait for.

```sql
-- Add words by hand
INSERT INTO words (text, category, drawability, enabled, source)
VALUES ('circle back', 'meeting', 2, 1, 'owner');

-- What are players inventing? Most-submitted first
SELECT text, SUM(times_used) AS uses, COUNT(*) AS submissions
FROM suggestions GROUP BY text ORDER BY uses DESC;

-- Promote the good ones
INSERT OR IGNORE INTO words (text, category, drawability, enabled, source)
SELECT DISTINCT text, 'player', 2, 1, 'player'
FROM suggestions WHERE text IN ('eat our own dogfood', 'swim lane');

-- Retire what nobody ever guesses (fixes drawability ratings on evidence)
SELECT text, times_drawn, times_guessed,
       ROUND(100.0 * times_guessed / times_drawn) AS guess_pct
FROM words WHERE times_drawn >= 10 ORDER BY guess_pct ASC;

UPDATE words SET enabled = 0 WHERE text = 'synergy';
```

### 7b. Endpoint — deferred to the end of the build (M9)

`POST /api/words`, bulk-accepting `{ words: [{ text, category, drawability }] }`,
idempotent on `text`, guarded by a static bearer token from an env var.

Note on that token: a shared secret in an env var is meaningfully weaker than a
login, and this endpoint writes to the database. It is the right weight here
only because the blast radius is "someone adds words to a party game" — but it
must not be left unguarded, or it becomes a public spam target the moment the
URL is known.

**Deferring this costs nothing**, which is exactly why it goes last: 7a already
covers every case from M1 onward. The endpoint is a convenience for adding words
without shell access to the volume, not a capability.

---

## 8. Build timeline

Estimates are solo-developer working days. They assume the customizations are
built *into* each phase rather than appended — §4 and §6 in particular are cheap
now and expensive later.

| # | Milestone | Days | Depends on |
|---|---|---|---|
| M0 | Scaffold: workspace, TS config, shared event contract, CI lint/build | 0.5 | — |
| M1 | DB, schema, seed dictionary (~200 corporate slang entries, rated) | 1.0 | M0 |
| M2 | Rooms: create/join by code, lobby, player list, host settings, reconnect-by-token | 1.0 | M0 |
| M3 | Canvas: stroke log, drawing sync, tools, late-joiner replay, undo | 2.0 | M2 |
| M4 | Game loop: round state machine, server timer, word selection, **custom-word entry** | 2.0 | M1, M2 |
| M5 | Guessing: phrase normalization, matching, hints, scoring, channel separation | 1.5 | M4 |
| M6 | Polish: avatars, sound, end-game podium, keyboard shortcuts | 1.5 | M5 |
| M7 | Deploy: container, volume, env config, load/abuse hardening | 1.0 | M6 |
| M8 | `POST /api/words` + bearer token | 0.25 | M7 |

**Total ≈ 10.75 working days ≈ 2.5 calendar weeks solo.**

**All milestones M0–M8 are complete.** The game is playable end to end and the
backend runs in its production container.

Verified: 64 socket-level test cases across three suites (`pnpm smoke`,
`smoke:game`, `smoke:guess`), all passing against both a dev server and the
built Docker image, plus the words endpoint and graceful shutdown.

### Bugs found by exercising the system

Each was invisible to the type checker and to inspection:

- `Result<Record<string, never>>` was unsatisfiable — intersecting `{ok: true}`
  with `Record<string, never>` collapses `ok` to `never`, so no success ack
  could be constructed. (Contract v2.)

- Frontend CI could not fetch the contract at all: `raw.githubusercontent.com`
  cannot authenticate against a private repo, leaving the drift guard inert in
  the one place it mattered. CI now checks the backend out as a sibling.
- `id NOT IN (NULL)` matched **zero** rows, so word selection returned nothing
  and every game ended the instant it started.
- The custom-word rate limiter counted rejected attempts, so three typos locked
  a player out of submitting a valid word.
- The Docker build failed because corepack was unpinned and ignored
  `onlyBuiltDependencies`, so `better-sqlite3` never compiled.

### Known gaps

- **The frontend has no automated tests.** It typechecks and builds against the
  real contract, and every server interaction behind it is covered, but nothing
  drives the UI itself. The six defects below were all found by playing it,
  which is exactly the class of problem this gap leaves open.
- **Profanity filtering is a short hand-written list.** Fine among colleagues;
  swap in a maintained library before exposing this to strangers.
- **Single instance only.** Rooms live in memory, so horizontal scaling needs a
  Redis adapter first.

Two checkpoints where it is worth stopping to play it before continuing:

- **After M3** — two browsers, one draws, the other watches. Proves the hardest
  technical risk (sync + replay) before any game logic is layered on it.
- **After M5** — the game is fully playable end to end. This is where the
  corporate-slang dictionary gets its first real test, and where drawability
  ratings will need their first correction. From M1 onward you can fix them with
  an UPDATE mid-session, without waiting for a tool to be built.

Both checkpoints are now reachable: run both servers and open two browsers.

### Post-release fixes (first play session)

Playing it surfaced six issues, none of which any automated check could have
caught — five were visual, and the sixth only failed away from `localhost`:

- **Fill painted the whole canvas.** It was a `destination-over` wash, an
  honest shortcut taken because a raster fill seemed irreconcilable with a
  vector log. It is not: see §5 above. Replaced with a real flood fill.
- **Multi-word masks ran together.** The server already padded between words;
  HTML collapses a run of spaces, so `_ _ _   _ _ _` rendered as one blank.
  Fixed with `white-space: pre` — the bug was in the display, not the masking.
- **Copy invite link silently did nothing.** `navigator.clipboard` only exists
  in a secure context, so on a LAN address it was `undefined`, the call threw,
  and an empty `catch` swallowed it. This would have worked forever on
  localhost. Now falls back to a textarea, then to revealing the link.
- **The theme followed the viewer's OS setting.** Pinned to light: the canvas
  is transparent and fill floods against what is painted, so a surface that
  differed per viewer would render one stroke log as two different pictures.
- **Guess feedback had no colour.** Added a `tone` field to the contract
  (**v3**) rather than having the client infer intent by matching message text,
  which breaks the moment the wording changes. Covered by two new assertions.
- **No end-of-game payoff.** Winner banner, crown, confetti; ties announce every
  player on the top score; all of it disabled under `prefers-reduced-motion`.

Also from the same session: `CLIENT_ORIGIN` accepts a comma-separated list, and
the client derives the socket host from the page's own origin rather than a
literal `localhost`, so a second device on the wifi reaches the backend on the
dev machine instead of looking for one on itself.

M2 → M3 → M4 → M5 is the critical path. M1 (dictionary) is independent of the
drawing work and could run in parallel with a second developer.

### Principal risks

| Risk | Mitigation |
|---|---|
| Canvas perf degrades over a long round | Offscreen caching, per-frame batching, stroke cap (M3) |
| Pointer input edge cases (stylus, trackpad) | Pointer Events API from the start, not mouse events retrofitted (M3) |
| Round-transition race conditions | Server-authoritative state machine, single timer source, clients render only what they're told (M4) |
| Client/server contract drift across repos | Vendored contract + `contract:check` in FE CI and build (done, M0) |
| Custom-word abuse | Sync validation, profanity filter, rate limit, in-game report (M4) |
| Unguarded write endpoint | Bearer token from env, required at M8 — not optional |
| Slang dictionary skews stale or regional | Stats-driven retirement query in §7a |
