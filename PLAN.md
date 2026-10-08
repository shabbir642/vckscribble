# vckscribble — Build Plan

A Skribbl.io-style multiplayer drawing-and-guessing game, with three deviations
from the original:

1. The drawer gets 3 suggested words **and** can write their own if none appeal.
2. The dictionary is corporate slang, not generic nouns.
3. The owner can extend the dictionary through an admin panel.

Those three requirements, not the drawing engine, are what shape the
architecture below.

---

## 1. Architectural consequences of the customizations

Worth stating up front, because each one rules out an "obvious" simpler choice.

**Requirement 3 forces persistence.** The original Skribbl is entirely
ephemeral — rooms live in memory and die with the process. An owner-curated
dictionary cannot. This adds a database, a migration story, and an
authenticated admin surface that otherwise would not exist. Everything else
(rooms, strokes, scores) stays in memory and remains disposable.

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
guess-rate data in the admin panel (§7).

**Requirement 1 cannot be pre-moderated.** A custom word is used *immediately* —
the round starts the instant it is submitted, so there is no window for owner
review before it reaches players. Moderation is therefore synchronous
(validation + profanity filter at submit time) with an in-game report button as
the backstop. Owner review happens *after* play, when deciding whether the word
earns a permanent place in the dictionary.

---

## 2. Stack

| Layer | Choice | Why |
|---|---|---|
| Language | TypeScript throughout | One shared event-contract module; wire protocol cannot drift between client and server |
| Client | React + Vite | Fast HMR; canvas work is plain DOM API regardless of framework |
| Server | Node + Express + Socket.IO | Socket.IO gives rooms, acks, and reconnection handling for free |
| DB | SQLite + Drizzle | Single owner, single instance. A managed Postgres would be operational overhead for what is one table of words |
| Deploy | Single container, Fly.io or Railway | WebSocket-friendly with a persistent volume. **Not** Vercel/Lambda — serverless cannot hold socket connections |

Workspace layout (pnpm):

```
client/   React app
server/   Socket.IO + Express + Drizzle
shared/   Socket event types, word normalization (imported by BOTH sides)
```

`shared/` holds the guess-normalization function specifically so the client can
preview "is this a valid guess" with exactly the server's logic, with the server
still authoritative.

---

## 3. Data model

```
words
  id, text, category, drawability(1-3), enabled,
  source('seed'|'owner'|'player'), times_drawn, times_guessed, created_at

suggestions
  id, text, submitted_by, room_code, created_at,
  status('pending'|'approved'|'rejected'), times_used
```

The seed dictionary (~200 entries) lives in `server/src/db/words.seed.ts` under
version control, not as raw INSERTs — so dictionary changes are reviewable in
diffs. Seeding is idempotent on `text`, runs on boot, and never overwrites
owner edits to an existing row.

`suggestions` is deliberately a separate table from `words`. Player-submitted
text is untrusted until you approve it; mixing it into the live dictionary with
a status flag invites a query that forgets to filter.

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
- reject if it duplicates a disabled word (someone retired it for a reason)

A 15s selection timer auto-picks a candidate if the drawer stalls, so one AFK
player cannot freeze the room.

Every accepted custom word is written to `suggestions` as `pending`. **This is
what links requirement 1 to requirement 3:** play itself generates your
candidate vocabulary, and the admin queue is where you promote what worked.
A word submitted by several different players, and actually guessed, is a
strong candidate — which is why `suggestions` tracks `times_used`.

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
rather than emitting per-event. Stroke count is capped per round as a
memory guard.

Tools: brush (3 sizes), color palette, fill, eraser, undo, clear.

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
already guessed correctly are moved to a separate post-guess channel — otherwise
the first correct guesser trivially spoils the round.

**Scoring:** guessers earn by time remaining; the drawer earns per correct
guess, so drawing clearly is rewarded over drawing obscurely.

---

## 7. Owner admin (`/admin`)

Auth: password from an env var → signed, httpOnly session cookie, rate-limited
login. No user table, no registration, no password reset — you are the only
account, and every one of those features would be attack surface for zero gain.

Capabilities:

- browse/search the dictionary, filter by category, source, enabled state
- add words singly or by bulk paste
- enable/disable without deleting (preserves play stats)
- review the pending-suggestion queue; approve promotes into `words` as
  `source='player'`
- **per-word stats: draw count and guess rate.** This is the feedback loop that
  fixes the drawability ratings from §1 — a word drawn 40 times and guessed
  twice is miscategorized or simply unfun, and should be retired on evidence
  rather than instinct.

---

## 8. Build timeline

Estimates are solo-developer working days. They assume the customizations are
built *into* each phase rather than appended — §4 and §6 in particular are
cheap now and expensive later.

| # | Milestone | Days | Depends on |
|---|---|---|---|
| M0 | Scaffold: workspace, TS config, shared event contract, CI lint/build | 0.5 | — |
| M1 | DB, schema, seed dictionary (~200 corporate slang entries, rated), admin auth skeleton | 1.5 | M0 |
| M2 | Rooms: create/join by code, lobby, player list, host settings, reconnect-by-token | 1.0 | M0 |
| M3 | Canvas: stroke log, drawing sync, tools, late-joiner replay, undo | 2.0 | M2 |
| M4 | Game loop: round state machine, server timer, word selection, **custom-word entry** | 2.0 | M1, M2 |
| M5 | Guessing: phrase normalization, matching, hints, scoring, channel separation | 1.5 | M4 |
| M6 | Admin panel: CRUD, bulk import, suggestion queue, stats | 1.0 | M1, M4 |
| M7 | Polish: avatars, sound, mobile touch, end-game podium | 2.0 | M5 |
| M8 | Deploy: container, volume, env config, load/abuse hardening | 1.0 | M7 |

**Total ≈ 12.5 working days ≈ 3 calendar weeks solo.**

Three checkpoints where it is worth stopping to play it before continuing:

- **After M3** — two browsers, one draws, the other watches. Proves the hardest
  technical risk (sync + replay) before any game logic is layered on it.
- **After M5** — the game is fully playable end to end. This is where the
  corporate-slang dictionary gets its first real test, and where drawability
  ratings will need their first correction.
- **After M6** — you can run the dictionary yourself; everything past this point
  is polish and can be cut if you want to ship early.

M2 and M3 are the critical path; M1/M6 (dictionary and admin) are independent of
the drawing work and could run in parallel with a second developer.

### Principal risks

| Risk | Mitigation |
|---|---|
| Canvas perf degrades over a long round | Offscreen caching, per-frame batching, stroke cap (M3) |
| Round-transition race conditions | Server-authoritative state machine, single timer source, clients render only what they're told (M4) |
| Mobile touch handling | Pointer Events API from the start rather than mouse events retrofitted (M3) |
| Custom-word abuse | Sync validation, profanity filter, rate limit, in-game report (M4) |
| Slang dictionary skews stale or regional | Stats-driven retirement in admin; seed file is version-controlled and reviewable (M7) |
