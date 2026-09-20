# TRI Online — Feature & Tech Plan

Status: planning only, nothing implemented yet. This repo stays the simple/reference
version (vanilla JS, local hotseat + greedy CPU). Online multiplayer will live in a
**separate project** that reuses the game rules but not the DOM-rendering code.

## Tech stack

| Layer | Choice | Notes |
|---|---|---|
| Frontend framework | React + Vite | TypeScript from the start |
| Language | TypeScript | Non-negotiable — see "shared engine" below |
| Client state | Zustand | Local game/UI state (selection, animation, bonus-move mode) |
| Data fetching / cache | TanStack Query | For lobby list, match history, leaderboard, profile — **not** for live move state |
| Routing | TanStack Router | Add once there's more than one screen (lobby, `/game/:id`, profile) — not needed for MVP |
| Backend / BaaS | Supabase | Postgres (source of truth), Auth (incl. anonymous), Realtime, Edge Functions |
| Hosting (frontend) | Vercel | Static Vite build |
| Hosting (backend logic) | Supabase Edge Functions | Colocated with DB, used for move validation |

## Core architecture principle: shared, pure game engine

Extract the current `game.js` rules logic (board, legal moves, combat resolution, turn/bonus
state) into a **pure TypeScript module with no DOM dependencies**. This one module gets reused
in three places:

1. Client — optimistic local move preview / animation
2. CPU opponent — search/heuristics run against it
3. Server — Supabase Edge Function imports the same module to validate every move

This is what makes server-validated moves possible without writing the rules twice.

## Anti-cheat / move validation

Because of ranked ELO and a public leaderboard, moves must be **server-validated**, not
client-to-client:

- Client sends an intended move to a Postgres RPC or Edge Function.
- Server runs it through the shared engine, rejects illegal moves, writes the authoritative
  resulting state to Postgres.
- Clients subscribe to `postgres_changes` (not raw Realtime `broadcast`) so both players (and
  spectators) receive the server-confirmed state, not something a modified client fabricated.

## Accounts & guests

- No account required to play CPU or Quickplay.
- Guests play via Supabase anonymous auth (gets a real user id, just unlinked to an identity).
- A guest can be invited to a match by another user even with no account.
- If a guest later creates an account, their anonymous session/user id is upgraded/linked so
  their guest match history carries over (Supabase supports linking an anonymous user to a
  permanent identity — confirm exact flow when implementing).

## CPU opponent

- Current greedy CPU (from this repo) has exploitable patterns — needs a real difficulty
  upgrade (e.g. minimax/negamax with lookahead + eval function, or at least weighted
  heuristics that avoid the obvious greedy traps) before shipping it as the "no account"
  default opponent.

## Matches

- Visibility: **public by default**, can be set to **private**.
- Public matches are spectatable; private matches are not.
- Two ways into a match: **Quickplay** (matchmaking queue) or **direct invite** (works for
  guests too).

## Quickplay matchmaking

Pure FIFO ignores skill and produces blowouts; pure ELO-band matching in a small
player pool can mean long or infinite waits. Resolve this with **progressive
band widening**, the standard approach (lichess/chess.com-style): start narrow,
widen the acceptable rating gap the longer someone waits.

### Algorithm

- Joining Quickplay writes a row to a `matchmaking_queue` table: `user_id`, `elo`,
  `joined_at`.
- A scheduled Edge Function (Supabase cron, every ~2s) scans the queue ordered by
  `joined_at` and tries to pair entries whose rating gap is within the current
  allowed band for their wait time:

  | Waited | Allowed ELO gap |
  |---|---|
  | 0–10s | ±50 |
  | 10–30s | ±150 |
  | 30–60s | ±300 |
  | 60s+ | unbounded — match the longest-waiting pair available |
- On a match, both rows are deleted from the queue and a `games` row is created
  (Quickplay games are always public/spectatable per the default in "Matches").
- FIFO is the tiebreak *within* a satisfied band, not the primary sort — this
  keeps early, easy matches skill-appropriate while guaranteeing nobody waits
  forever once the band opens up.

### Provisional ratings

- New accounts/guests start at a fixed baseline ELO (e.g. 1200) with a **wider
  starting band and higher K-factor** for their first ~10–15 games, same idea as
  chess rating systems — lets a real skill estimate form quickly instead of
  parking new players in a band with wildly mismatched opponents while their
  rating is still meaningless.
- Guests are matched on equal footing with accounts — no separate guest-only
  pool; splitting the queue would only worsen wait times for both groups.

### No-opponent-found fallback

- If a player has waited past a configurable ceiling (e.g. 45–60s) with no match
  found (plausible for an early/small player base), offer a **"play CPU
  instead"** prompt rather than leaving them stuck in an indefinite queue. This
  leans on the CPU-difficulty upgrade already planned above, and the queue entry
  stays live in the background in case a human match still appears.

### Queue abuse guards

- Rate-limit rapid join/leave cycling (queue-dodging to avoid an incoming
  matchup) — e.g. a short cooldown after voluntarily leaving the queue.
- Avoid immediately rematching the same two players twice in a row when other
  candidates are available in-band, to reduce perceived "stuck with the same
  opponent" staleness — soft preference, not a hard rule, so it never overrides
  band-widening's guarantee of eventually finding *a* match.

### Interaction with direct invites

- Invites bypass the queue entirely — they create a `games` row directly (via
  the existing `invites` flow) and never touch `matchmaking_queue`.
- Visibility for invite-created games defaults per the inviter's chosen setting,
  not forced public like Quickplay.

## Match lifecycle: timing, disconnects & forfeits

TRI is turn-based and server-authoritative, so a socket disconnect does **not** by
itself break a match — the server keeps the true state regardless of who's online.
The real design problem is: how long does the game wait for a missing player, and
under what conditions does it end the match on their behalf. This needs to be
resolved before Realtime sync (phase 3) so the schema and Edge Functions are built
around it instead of bolted on after.

### Time control model

- Match creation picks a time control, same idea as chess.com/lichess:
  - **Untimed** — for private/friend matches. No clock; ends via resignation,
    elimination, draw, or abandonment (below).
  - **Timed** — required for Quickplay and ranked. Base time + per-move increment
    (Fischer-style), tracked independently per player, decremented only during
    that player's own turn.
- The clock is **server-authoritative**: the server timestamps move receipt and
  computes remaining time from stored timestamps, never from a client-reported
  duration. Client shows a locally-ticking display purely for UX, reconciled
  against the server value on every synced state update.
- Clock hitting 0 is an automatic loss by timeout — resolved server-side (the
  same Edge Function path that validates moves also needs to check/enforce this,
  e.g. on each move submission and via a scheduled sweep for the idle side).

### Presence vs. the clock

- Track online/offline via Supabase Realtime **Presence**, kept separate from the
  turn clock and **not** persisted to Postgres (ephemeral, per-connection).
- Presence is a UI signal only for timed games ("opponent disconnected" badge) —
  it never triggers a forfeit by itself, since a disconnected player's clock may
  still have plenty of time left and they can reconnect before it expires.
- For **untimed** games, presence is the only signal available, so it does drive
  forfeiture there (see below).

### Untimed-game abandonment

- No clock means a missing player needs an explicit threshold:
  - After inactivity beyond threshold A (e.g. 24–48h, exact value TBD at
    implementation time), the opponent gets a **"claim win"** action.
  - After threshold B (longer, e.g. 7 days) with neither player active, the
    match auto-resolves as **abandoned** — void, does not affect ELO — rather
    than a scored loss, since neither side asserted a claim.
- Requires a scheduled sweep (Supabase cron → Edge Function) since nothing else
  triggers this check for a game nobody is actively looking at.

### Reconnect flow

- No special "resume" logic is needed: state lives in Postgres, so reconnecting
  is just re-subscribing to `postgres_changes` for that game id and re-fetching
  current state.
- Multiple sessions for the same player (second tab, another device) are simply
  allowed — moves are authorized by the player's auth session, not bound to a
  device or socket, so there's no "which session owns the game" problem to solve.
- UX polish still needed: a reconnect banner/spinner, and not discarding
  in-progress local UI state (piece selection, bonus-move mode) on a reconnect
  that turns out to be same-state.

### Forfeit / end-of-match reasons

End state needs to distinguish *why* a match ended, not just that it did:

| Reason | Trigger | Affects ELO? |
|---|---|---|
| Elimination | Normal win condition (opponent's 3 Leaders gone) | Yes |
| Fifty-move-rule draw | Existing local-game rule, carries over | Yes (as a draw) |
| Resignation | Explicit player action | Yes |
| Timeout | Clock hits 0 (timed games only) | Yes |
| Forfeit claim | Opponent claims win after untimed-abandonment threshold A | Yes |
| Abandoned | Neither player active past threshold B | No — void |

### Data model additions (extends the sketch below)

- `games`: `time_control` (jsonb: mode, base_seconds, increment_seconds),
  `clock_p1_remaining_ms`, `clock_p2_remaining_ms`, `last_move_at`, `end_reason`
  (enum matching the table above), `status` gains explicit `waiting` / `active` /
  `completed` values (no separate "paused" — a disconnected-but-not-forfeited
  game is still `active`).

### Open sub-questions

- Exact abandonment thresholds A/B, and whether they differ for casual vs. ranked.
- Whether to add "it's your turn" mobile push notifications (later phase, not MVP).
- Whether repeated abandonment should be rate-limited/penalized to discourage
  queue-and-vanish behavior against Quickplay opponents.

## Player-facing features

- **Leaderboard** — ELO rating, ranked by rating.
- **Match history** — every game recorded; full replay viewable anytime (implies storing the
  move list/log per game, not just final result).
- **Stats** — win/loss ratio, last-10-games record, overall record, current ELO.
- **Spectating** — read-only view of public matches in progress via Realtime.

## Rough data model sketch (to refine at implementation time)

- `profiles` — id, is_guest, display_name, elo, wins, losses
- `games` — id, player1_id, player2_id, visibility (public/private), status, winner_id,
  created_at, time_control, clock_p1_remaining_ms, clock_p2_remaining_ms, last_move_at,
  end_reason (see "Match lifecycle" above)
- `moves` — game_id, move_number, move data (from/to/resulting state or notation), timestamp
- `invites` — id, game_id, inviter_id, invitee (user id or guest link), status
- `matchmaking_queue` — user_id, elo, joined_at (see "Quickplay matchmaking" above)

## Open questions for later

- Rate limiting / abuse prevention on invites to guests
- Exact anonymous → permanent account linking flow in Supabase Auth
- Exact band-widening timings and provisional-rating K-factor/game count for
  Quickplay (starting values proposed above, to tune once there's real traffic)

(Quickplay matchmaking and reconnect/disconnect/forfeit handling are now specced
in their own sections above.)

## Suggested phasing

1. Extract pure TS game engine from current `game.js` (still usable standalone).
2. Scaffold new Vite + React + TS + Zustand project; port local hotseat mode using the shared
   engine; upgrade CPU difficulty.
3. Add Supabase: auth (guest + real accounts), schema above, server-validated move RPC/Edge
   Function, Realtime sync for a single live match.
4. Lobby: quickplay queue + direct invites, public/private visibility.
5. Spectating for public matches.
6. Leaderboard/ELO, match history + replay viewer, TanStack Query for lobby/stats, TanStack
   Router once there are multiple routes.
