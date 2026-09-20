# TRI — Frequently Asked Questions

A casual-player companion to [RULES.md](RULES.md). If you want the exact
implementation-level wording, RULES.md is the source of truth — this file
just answers the questions people actually ask when learning the game.

## Basics

### What's the goal of the game?
Capture all 3 of your opponent's Leaders (one Square, one Diamond, and one
Circle Leader). The instant your opponent has zero Leaders left, you win —
it doesn't matter how many Squadrons either side still has.

### What are the three piece types and how do they interact?
Square, Diamond, and Circle form a rock-paper-scissors cycle:
- **Square beats Diamond**
- **Diamond beats Circle**
- **Circle beats Square**

This is the only thing that decides who wins a capture — a piece's type,
not whether it's a Squadron or a Leader.

### What's the difference between a Squadron and a Leader?
Each type (Square, Diamond, Circle) has 5 Squadrons and 1 Leader. Leaders
move a bit farther and "leap" over other pieces instead of sliding, but
type is still all that matters in combat — a Squadron can capture an enemy
Leader just as easily as another Squadron can, as long as its type has the
advantage.

### How many pieces does each side start with?
18 each: 5 Squadrons + 1 Leader for each of the 3 types.

## Movement

### How far can a piece move?
It depends on the piece and whether it's promoted (see below):

| Piece            | Moves            | Range (before / after promotion) |
|------------------|------------------|-----------------------------------|
| Square Squadron  | Orthogonal, slides | 2 / 3 |
| Square Leader    | Orthogonal, leaps   | 3 / 4 |
| Diamond Squadron | Diagonal, slides    | 2 / 3 |
| Diamond Leader   | Diagonal, leaps     | 3 / 4 |
| Circle Squadron  | Any direction, slides | 1 / 2 |
| Circle Leader    | Any direction, leaps  | 2 / 3 |

### What's the difference between "sliding" and "leaping"?
Squadrons **slide**: they move like a chess rook/bishop/queen (depending on
type), and stop the instant they hit any piece. If that first piece is an
enemy, they can capture it; if it's friendly, they're just blocked.

Leaders **leap**: they ignore everything in between and can land on any
legal square within range, friendly pieces and enemies alike. They can even
"jump over" a nearer enemy piece to capture a farther one instead, or fly
past their own pieces freely. They just can't land on a square already
occupied by one of their own pieces.

### Can I ever just pass my turn?
No — but you'll never be forced to make a bad move either. If a piece has
no legal moves, you can always fall back on a **Tactical Swap** (see below)
with an adjacent friendly piece instead. Between "move a piece" and "swap
two adjacent pieces," you always have a legal action available.

## Promotion

### How does a piece get promoted?
By reaching the opponent's home row (row 0 for Player 1, row 8 for Player
2) — whether it gets there by a normal move, a capture, a bonus move, or
even a Tactical Swap.

### What does promotion actually do?
It restores the piece to its full range from the table above. Every piece
actually starts at one *less* than its listed range and only reaches that
full range once promoted. Promotion never changes what type of piece it is
or how it moves — only how far.

### Is promotion permanent?
Yes. Once a piece is promoted, it stays promoted for the rest of the game,
even if it later moves back to its own side of the board.

## Combat

### What happens when I move onto an enemy piece?
Compare types using the rock-paper-scissors cycle:
- **Same type (neutral):** you capture them, turn over, no bonus.
- **Your type beats theirs (advantage):** you capture them, and you get an
  optional bonus action with that same piece.
- **Their type beats yours (disadvantage):** **both pieces are destroyed** —
  neither ends up on the square. This is called mutual annihilation.

### Wait, my piece can die even if I'm the one attacking?
Yes — moving onto an enemy that beats your type is not a good idea!
Attacking is only safe (for your piece) when it's a neutral or advantageous
matchup. Attacking into a disadvantageous matchup trades your piece for
theirs.

### What's a "bonus action"?
After a winning (advantage) capture, you get to do one more thing with the
same piece before your turn ends: move it again (which can even capture a
second piece), swap it with an adjacent friendly piece, or just skip it.
You can never chain a third action, though — one bonus action per turn, max.

### If my bonus move captures another piece, does that chain into a third action?
Only if it's another advantage capture — and even then, no, the rules cap
it at one bonus per turn either way. So: normal move → advantage capture →
bonus action, full stop. If the bonus move itself results in a disadvantage
capture, your own piece is destroyed along with the new target (mutual
annihilation applies to bonus moves too).

## Tactical Swap

### What is a Tactical Swap?
Instead of moving, you can trade the positions of two of your own adjacent
pieces (any of the 8 surrounding squares) — regardless of what type or role
either piece is. It's a full turn on its own and never involves combat.

### Why would I want to do that instead of moving?
Common uses: repositioning a piece that's boxed in, getting a Squadron
"through" a logjam of your own pieces, or — since promotion is checked
after a swap too — sliding a piece onto the enemy's home row via a swap to
promote it.

### Can I move a piece and then swap it, or swap and then move it?
No. A turn is exactly one action: one move, or one swap. Never both.

## Winning, Draws, and Stalls

### How exactly do I win?
The moment your opponent has 0 living Leaders (all 3 types' Leaders
captured), the game ends and you win immediately — this is checked after
every single capture.

### Can the game end in a draw?
Two ways:
1. **Mutual elimination** — a single mutual-annihilation capture happens to
   wipe out *both* players' last Leader at once.
2. **Stalemate by inactivity** — see the next question.

### What stops a game from just going on forever?
If 40 full rounds (both players taking a turn counts as one round) pass
with **no captures at all**, the game ends automatically. Whoever has more
living Leaders at that point wins; if it's tied, it's a draw. This counter
resets to zero the instant any capture happens, so it only triggers during
genuinely stalled games.

### Why 40 rounds and not just "play until someone wins"?
Without a limit, a player who's already behind on Leaders could just stall
indefinitely to force a draw instead of fighting it out. With the 40-round
rule, stalling while behind is a loss, not an escape — so it encourages
both sides to keep contesting captures.

### Can a game get stuck in a position where a capture is literally impossible?
Yes, in one specific case: Diamond pieces only ever move diagonally, so a
Diamond piece can never change the color of square it's standing on. If
both players are down to a single Diamond Leader each, and those two
Leaders happen to sit on opposite-colored squares, they can never reach
each other. This isn't specially detected — it just naturally runs out the
40-round no-progress clock and ends in a draw (since both sides have 1
Leader each).

### Does the game track repeated positions (like chess's threefold repetition)?
The reference implementation does, as a safety net for bot-vs-bot games:
if the exact same position (same pieces, same squares, same promotions,
same player to move) occurs a 3rd time, the game ends right there using the
same "more Leaders wins, equal is a draw" rule. In practice the 40-round
no-progress rule usually ends a stalled game first, since a position can
only repeat if nothing's been captured in between.

## Strategy Tips

### Does having more Squadrons left matter if the game reaches the 40-round limit?
No — only living Leader counts are compared. A big Squadron advantage means
nothing if the game stalls out and both sides still have equal Leaders;
that outcome is a draw regardless of material.

### What's the safest way to think about attacking?
Before moving onto an enemy piece, ask what beats what: attacking a
same-type piece is an even trade (you win, no bonus), attacking your
"prey" type is great (capture + bonus action), and attacking your
"predator" type destroys your own piece too. When in doubt, check the
triangle: Square > Diamond > Circle > Square.

### Why are Leaders more dangerous attackers than Squadrons of the same type?
They're not more dangerous in combat — type is all that matters there — but
their leaping movement makes them harder to defend against, since they
aren't blocked by pieces in between and can strike from farther away in
one move (especially once promoted).
