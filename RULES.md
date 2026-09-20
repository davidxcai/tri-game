# TRI — Complete Rules Reference

This document is the complete, implementation-level rules specification for
TRI, a two-player, zero-luck strategy game played on a 9x9 grid. It is
written to be sufficient on its own for a fresh implementation — every
mechanic, edge case, and tie-breaking rule used by the reference
implementation in this repo (`game.js`, `ai.js`, `ai-hard.js`) is described
here explicitly.

## 1. Board and Coordinates

- The board is a 9x9 grid of squares, rows and columns indexed 0–8.
- Row 0 is the top of the board and is **Player 2's home rank** (traditionally
  called "Rank 9"). Row 8 is the bottom of the board and is **Player 1's home
  rank** ("Rank 1"). Column 0 is the left edge, column 8 is the right edge.
- Rows 2–6 start empty; all pieces begin on rows 0, 1, 7, and 8.
- Who moves first is decided by the players (mutual agreement or a coin
  toss) before the game starts; the reference implementation always starts
  with Player 1 (there is no first-move advantage baked into the rules
  themselves — the two sides' starting positions are exact 180° rotations
  of each other, see §2.2).

## 2. Pieces

There are 3 piece **types** — informally "Square" (A), "Diamond" (B), and
"Circle" (C) — and 2 **roles** per type — **Squadron** and **Leader**. Every
type/role combination has a fixed movement range, a fixed movement vector
(set of directions), and a fixed movement trait (sliding or leaping):

| Piece            | Range (promoted) | Vector          | Trait   |
|------------------|-------------------|-----------------|---------|
| Square Squadron  | 3                 | Orthogonal      | Sliding |
| Square Leader    | 4                 | Orthogonal      | Leaping |
| Diamond Squadron | 3                 | Diagonal        | Sliding |
| Diamond Leader   | 4                 | Diagonal        | Leaping |
| Circle Squadron  | 2                 | Omnidirectional | Sliding |
| Circle Leader    | 3                 | Omnidirectional | Leaping |

- "Orthogonal" = the 4 directions up/down/left/right.
- "Diagonal" = the 4 diagonal directions.
- "Omnidirectional" = all 8 of the above combined.
- "Range" is the maximum number of squares a piece may travel in one of its
  directions in a single move (see §5 for how promotion affects this).
- Every piece is owned by exactly one player and keeps its type and role for
  the entire game — types and roles never change.

### 2.1 Army composition (per player)

Each player starts with 18 pieces: for each of the 3 types, exactly **1
Leader** and **5 Squadrons** (6 pieces per type × 3 types = 18).

### 2.2 Starting layout

Below, each cell shows `<Type>` for a Squadron and `<Type>L` for a Leader.
Blank cells are empty. Rows 2–6 (not shown) are entirely empty.

```
       col: 0    1    2    3    4    5    6    7    8
row 0 (P2 back / Rank 9):  C   AL    C    B   CL    B    A   BL    A
row 1 (P2 front / Rank 8): B    B    B    A    A    A    C    C    C
row 7 (P1 front / Rank 2): C    C    C    A    A    A    B    B    B
row 8 (P1 back  / Rank 1): A   BL    A    B   CL    B    C   AL    C
```

Notes on the layout:
- Player 2 occupies rows 0–1, Player 1 occupies rows 7–8, facing each other
  across the empty middle.
- **Rotational symmetry**: Player 2's layout is the exact 180° rotation of
  Player 1's layout (row 0 = reverse of row 8, row 1 = reverse of row 7,
  same type codes — not mirrored to the opposing type). Reproduce Player
  1's two rows exactly as given, then generate Player 2's rows by reversing
  column order; do not derive Player 2 independently.
- **How to arrange (design rule, useful for constructing or verifying the
  layout)**: think of the board as three 3-column groups, one per
  Commander. Within a group:
  - The back rank holds `[flank, Commander, flank]`.
  - The front rank holds 3 Squadrons of one type (`[front, front, front]`).
  - The **front-rank type is the type the Commander beats** (i.e. weak
    against the Commander — the Commander could safely capture it).
  - The **flank type (the two back-rank neighbors) is the type that beats
    the Commander** (i.e. strong against the Commander — it threatens the
    Commander, not the other way around).
  - Concretely, per Commander: Square Commander → front is Diamond
    (Square beats Diamond), flank is Circle (Circle beats Square). Diamond
    Commander → front is Circle, flank is Square. Circle Commander → front
    is Square, flank is Diamond.
  - This yields exactly 5 Squadrons of each type per side (3 from being
    the "front" type of their own Commander's group, 2 from being the
    "flank" type of a different Commander's group) plus 1 Leader per type —
    18 pieces total, matching §2.1.

## 3. The Triangle of Advantage

The three types form a rock-paper-scissors cycle used to resolve every
capture:

- **Square beats Diamond**
- **Diamond beats Circle**
- **Circle beats Square**

This applies regardless of role (Squadron or Leader) or promotion status —
only the piece's type matters for combat resolution.

## 4. Movement

On a player's turn, they choose one of their own living pieces and move it,
subject to its role's movement trait:

### 4.1 Sliding (Squadrons)

A sliding piece may move any number of squares, up to its current range (see
§5), along one of its allowed directions, provided the path is clear:

- It may stop on any empty square along the way.
- If the first occupied square encountered in that direction (within range)
  contains an **enemy** piece, the slider may move onto that square,
  **capturing** the enemy piece there (see §6). It may not move further past
  it.
- If the first occupied square encountered contains a **friendly** piece,
  the slider is blocked: it may not move onto or past that square in that
  direction.

### 4.2 Leaping (Leaders)

A leaping piece may move to any square along one of its allowed directions,
up to its current range, **regardless of what occupies the squares in
between** — it flies over both friendly and enemy pieces without interacting
with them:

- It may land on any **empty** square within range.
- It may land on any square within range occupied by an **enemy** piece,
  which **captures** that piece (see §6). Landing is the only way a leaper
  interacts with a piece on its path — flying over a piece (friendly or
  enemy) never captures it and never blocks further movement.
- It may **not** land on a square occupied by one of its own pieces.
- Because occupied squares don't block a leaper, more than one landing
  square along the same direction can be a legal (capturing) destination in
  a single move — e.g. a leader can choose to land on a nearer enemy or fly
  over it to land on/capture a farther one. Only one destination is chosen
  per move.

### 4.3 No pass — every turn is a move or a swap

A player must, on their turn, either move a piece (§4.1/§4.2, possibly with
a capture and bonus action per §6) or perform a Tactical Swap (§7).

A single piece can certainly end up with zero legal *moves* (boxed in by
friendly pieces on every side, or too near the board edge). That is not a
problem on its own. What can never happen is the *player* being left with
zero legal *actions*, because Tactical Swap (§7) is always available as a
fallback: the very thing capable of blocking a piece's movement — an
adjacent friendly piece — is, by definition, also a legal swap partner.
Since every piece has at least one direction that stays on the board, it is
either free to move that way (an empty square is always a legal move, and
an enemy piece is always a legal capture, win or lose — see §6) or blocked
by a friendly piece it can swap with instead. An implementation does not
need a "no legal action, skip turn" case — it cannot occur, as long as
Tactical Swap is implemented as specified in §7.

## 5. Promotion

- Every piece's range from the table in §2 is reduced by 1 by default. A
  piece's **effective range** is therefore `base range - 1` until it is
  promoted, and `base range` (full, from the table) once promoted.
- A piece is promoted the first time it ends up occupying its **opponent's
  home rank** — row 0 for a Player 1 piece, row 8 for a Player 2 piece —
  as a result of any move, capture, or Tactical Swap that places it there.
- Promotion is checked immediately after a piece is placed on a new square,
  including the destination of a bonus move (§6) or either side of a
  Tactical Swap (§7) — a swap that lands a piece on the far rank promotes it.
- Promotion is **permanent** and **per-piece** — it does not transfer, and a
  promoted piece is never demoted.
- Promotion does not change a piece's type, role, movement vector, or
  movement trait — only its range.

## 6. Combat

Combat is resolved whenever a piece moves onto a square occupied by an enemy
piece (whether by sliding onto it or leaping onto it). Resolve using the
Triangle of Advantage (§3) between the **attacker's type** and the
**defender's type**:

- **Neutral** (same type on both sides): the defender is captured (removed
  from the board and from play). The attacker occupies the destination
  square. The turn ends immediately — no bonus action.
- **Advantage** (attacker's type beats defender's type): the defender is
  captured. The attacker occupies the destination square, then the
  attacker's controller may take **one optional bonus action** with that
  same piece (see §6.1). If this capture eliminated the opponent's last
  Leader (i.e. the game is already won), no bonus action is offered.
- **Disadvantage** (defender's type beats attacker's type): **mutual
  annihilation** — both the attacker and the defender are captured/removed
  from the board. Neither ends up occupying the square. The turn ends
  immediately — no bonus action.

### 6.1 Bonus actions

After an advantage capture, the attacking piece's controller chooses exactly
one of the following (or none):

1. **Move again** with the same piece, from its new position, following the
   normal movement rules for its type/role/range (§4). This can itself
   capture:
   - If this second move results in a **disadvantage** capture, mutual
     annihilation applies exactly as in §6 (the piece that just captured is
     now also eliminated, along with the new defender).
   - If this second move results in an **advantage** or **neutral**
     capture, it resolves normally, but **no further bonus action is
     granted** — a single turn may include at most one bonus action ("no
     3rd action").
   - If this second move captures no piece, it's just a repositioning move.
2. **Perform a Tactical Swap** (§7) using the same piece and one of its
   currently-adjacent friendly pieces, in place of moving again. This
   consumes the bonus action and ends the turn immediately — a bonus swap
   does not chain into another bonus action.
3. **Skip** the bonus action, ending the turn with no further effect.

If none of the above are legal (e.g. the piece has no legal moves and no
adjacent friendly pieces to swap with), the bonus action is automatically
skipped.

## 7. Tactical Swap

Instead of moving a piece, a player may perform a Tactical Swap: pick one of
their own pieces and exchange its board position with one of their **own**
pieces that is **immediately adjacent** — one square away in any of the 8
directions (orthogonal or diagonal), regardless of either piece's type,
role, or normal movement vector. Both pieces simply trade squares.

- The adjacent piece must be **friendly** — a unit can never swap with an
  opponent's piece.
- A unit cannot swap if it has no friendly adjacent piece.
- A swap consumes the entire turn (or the entire bonus action, if performed
  as one — see §6.1) — nothing else happens afterward.
- **A unit cannot move and then swap** (or swap and then move) — a normal
  turn is exactly one atomic action: either one move (§4) or one swap, never
  both in sequence.
- A swap never involves combat and can never capture — both pieces are
  friendly and both survive.
- After the swap, each piece independently checks for promotion (§5) based
  on its new square.
- The same mechanic is available as a normal turn action for any piece with
  at least one adjacent friendly piece, and as the alternative bonus action
  described in §6.1.

## 8. Win, Draw, and Stall Resolution

- **Win**: A player wins the moment their opponent has zero living Leaders
  (all 3 of that type — Square, Diamond, Circle — have been captured).
  Check for this after every capture, including bonus-action captures.
- **Draw — mutual elimination**: If a single action (a disadvantage capture,
  see §6) simultaneously eliminates both players' last remaining Leader, the
  game ends in a draw. (Leftover Squadron counts are irrelevant here — the
  game already ended per the Win condition above, which only looks at
  Leaders.)
- **Stall resolution — no progress**: Track the number of consecutive full
  rounds (one round = both players have taken one turn) during which **no
  capture of any kind** has occurred. If this reaches **40 rounds**, the
  game ends immediately, resolved by comparing each player's number of
  **living Leaders** (0–3):
  - Equal Leader counts → the game is a **draw**.
  - Unequal Leader counts → the player with **more** living Leaders **wins**.

  Leftover Squadrons are never counted or compared — the object of the game
  is capturing Commanders, not Squadrons, so a material edge in Squadrons
  alone doesn't decide a stalled game. This also closes off a stalling
  exploit: without this rule, a player who is behind on Leaders could just
  run out the clock for a guaranteed draw instead of fighting for captures;
  with it, running out the clock while behind on Leaders is a loss, not a
  draw, so the trailing player is pushed to actually contest captures. This
  counter resets to 0 every time any piece is captured (neutral, advantage,
  disadvantage, or bonus-action capture).

  This is also what resolves an unusual but real dead position: because
  Diamond pieces (Squadron and Leader alike) only ever move diagonally,
  a Diamond piece can never change the color of square it occupies. If a
  game comes down to both sides' last Leader being a Diamond, and they sit
  on opposite-colored squares, neither can ever reach the other — capture is
  permanently impossible, no matter how the rest of the position develops.
  No special-case detection is needed for this: it's just a position where
  no capture will ever occur, so the no-progress counter runs out in the
  normal way and (with equal Leader counts, 1 each) ends the game in a draw
  within at most 40 rounds. An implementation may optionally detect this
  kind of dead position instantly for a faster/cleaner ending, but it is not
  required for correctness.
- **Stall resolution — repetition (implementation-only safeguard, not part
  of the printed rulebook)**: the reference implementation additionally
  tracks every distinct board position (full set of living pieces' squares
  and promotion states, plus which player is to move next) and, if the
  identical position occurs a 3rd time, ends the game immediately using the
  **same Leader-count comparison** described above (equal → draw, unequal →
  more Leaders wins). This exists only to guarantee termination when two
  deterministic bots would otherwise shuffle forever, and is largely
  superseded by the no-progress rule above: a position can only repeat if no
  capture occurred anywhere in between, so the no-progress counter is
  climbing over that same span and will typically end the game first. A
  from-scratch implementation only needs this rule if it also needs to
  guarantee a fully automated match terminates; a human-vs-human
  implementation can rely on the no-progress rule alone.

## 9. Turn Summary

Each player's turn is exactly one of:

1. Move a piece (§4), possibly capturing (§6), possibly followed by one
   bonus action (§6.1) if the capture was an advantage capture.
2. Perform a Tactical Swap (§7).

(There is no third "pass" case — see §4.3.)

Turns alternate strictly between Player 1 and Player 2, starting with
whichever player was decided to go first (§1), until §8 resolves the game
(win, draw, or a stall resolved by Leader count).

## 10. Tournament Play (Optional)

Everything above (§1–§9) is the complete rule set for a single game and is
all that's needed for casual play — a casual game just needs to know who
won, or that it was a draw, which §8 already fully determines.

This section is an **optional layer for organized play** (a bracket, league,
or any format that scores multiple games against each other). It does not
change how any individual game is played or resolved — it only assigns
points to the outcome §8 already produced, for tallying standings:

| Outcome          | Winner points | Loser points | When it applies |
|------------------|---------------|---------------|------------------|
| Definitive Win   | 3             | 0             | Won by the normal Win condition (§8) — all 3 of the opponent's Commanders captured. |
| Win by Decision  | 2             | 1             | Won via the no-progress or repetition stall-resolution (§8) with an unequal Leader count. |
| Draw             | 1             | 1             | Any draw outcome from §8 (mutual elimination, or a stall-resolution with equal Leader counts). |

Rationale: a Definitive Win requires actually finishing the opponent off,
so it's worth more than winning because you held a Leader-count edge when
the clock ran out without ever forcing the last capture. The trailing
player in a Decision still banks 1 point rather than 0, since their game
ended in a stall, not a defeat.

A casual implementation should surface *which* outcome occurred (the
reference implementation's end-of-game overlay already distinguishes
"eliminated all Leaders" from "won on remaining Leaders after N turns" from
"draw") without needing to track or display point totals at all. An
implementation of organized play can layer the point table above on top of
that same outcome without changing any in-game logic.
