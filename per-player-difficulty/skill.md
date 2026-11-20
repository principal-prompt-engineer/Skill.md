---
name: per-player-difficulty
description: Use when a same-screen race or arcade game needs fixed, per-player difficulty levels (for example Tiny, Easy, Normal, Pro) so young children and adults can play the same round together. Covers what a level changes (steering assist, catch-up boost, crash penalty, speed floor, enemy speed, one-crash death), the exact reference values and formulas, class leaderboards, and how levels plug into the lobby. Use it whenever you build, tune or describe fixed difficulty levels, handicaps or "easier for the little ones" options, even if the user does not say "difficulty".
---

# Fixed per-player difficulty levels

Use with `multiplayer-lobby` (the level is picked in step 2 of the per-player setup). A player's level is chosen before the race and never changes during it. Reference implementation: the "Ski Run" downhill race (steer left and right, tuck, brake, jump; trees and rocks to avoid; a yeti that hunts the skiers). The numbers below come from that game. Keep the structure and retune the numbers for another game.

## 1. Principle
- Difficulty belongs to the player, not the room. A five-year-old and a parent ski the same course (same seed) at the same moment, so everything that makes the game easier or harder is a per-player parameter.
- A level is one row in a small parameter table that is read every tick from the player's level. Only the special cases in section 4 may branch on the level itself. This keeps adding a level cheap.
- Levels give help, never different content: the course, obstacles and win conditions are identical for everyone.

## 2. Parameter table (reference values)

| Level  | Steering assist `as` | Catch-up cap `cu` | Crash stun | Speed floor | Yeti speed factor | Dies on crash |
|--------|----------------------|-------------------|------------|-------------|-------------------|---------------|
| Tiny   | 1                    | 0.7               | 0 s (soft) | 280         | 0 (no yeti)       | no            |
| Easy   | 0.5                  | 0.3               | 0.5 s      | none        | 0.7               | no            |
| Normal | 0                    | 0                 | 1.2 s      | none        | 1                 | no            |
| Pro    | 0                    | 0                 | -          | none        | 1                 | yes           |

- Survival mode has no finish line and must put a yeti on everybody, so it uses its own yeti factors: Tiny 0.6, Easy 0.75, Normal 1, Pro 1.
- Show a one-line description per level under the level chips in the lobby and again in Settings > Players. Keep them in one list so both places stay in sync.

## 3. What each parameter does

**Steering assist (`as`, 0 to 1).** Active only while the skier is not stunned. The player's own steering always works (heading changes by 2.1 rad/s, clamped to +/-1.35 rad); assist adds to it:
- Self-centring: with no steering input the heading decays towards straight, `a *= 1 - min(1, dt * 2 * as)`.
- Obstacle avoidance: for trees and rocks within the next 5 rows, less than 170 px ahead and less than 30 px to either side, nudge the heading away from the obstacle: `a += away * as * 3.2 * dt * (1 - ahead/170)`.
- Star hunt mode: pull towards uncollected stars up to 220 px ahead and 100 px to the side: `as * 2.4 * dt * (1 - ahead/220)`.
- Slalom mode: steer towards the next gate when it is within 420 px ahead and more than 20 px off-centre: `as * 1.8 * dt * min(1, offset/150)`.
- Every term scales with `as`, so a fractional value is a valid, weaker assist.

**Catch-up boost (`cu`).** Multiplies the target speed by `1 + clamp((lead - y) / 600, 0, cu)`, where `y` is the distance down the course in px and `lead` is the largest `y` among all players who have played (every level, including finished and eliminated players; the reference code also counts the human yeti player). The boost grows with the gap and is capped by the level's `cu`. Normal and Pro never get it.

**Crash.** Hitting a tree or rock while on the ground, not stunned and not in the post-crash grace period:
- Place the skier beside the obstacle, so the same obstacle cannot hit twice.
- Pro: the skier is out immediately (cause "crash"), with the full elimination animation.
- Other levels: stun for `stun` seconds. If `stun > 0` speed drops to 0 and the heading resets to straight when the stun ends. If `stun = 0` (Tiny) the skier keeps going at half speed. The grace period is `stun + 0.9 s`.
- Nobody except Pro dies from crashing. Jumping over an obstacle (air time) skips the collision.

**Speed floor.** Tiny only: after the base speed is computed and before the catch-up boost, `target = max(target, 280)`. A small child who points sideways or brakes by accident never grinds to a halt.

**Yeti speed factor.** The yeti spawns 460 px behind and 260 px to the side of a skier once they pass the trigger distance (900 m by default, 300 m in survival; it is a setting), if the factor is above 0 and the AI yeti is switched on. Its speed is `(330 + min(110, (dist - trigger) * 0.06)) * factor * globalYetiSpeed` (survival: `330 + (dist - 300) * 0.12`, no cap). It catches a skier within 22 px. Factor 0 means no yeti at all. In Yeti hunter mode a human plays the yeti instead, and that player's speed comes from their own level: 300, 330, 360, 400 for Tiny, Easy, Normal, Pro.

## 4. Base movement the parameters plug into
- Target speed: `(70 + 330 * cos(heading)^1.6) * (tuck 1.3 | brake 0.35 | 1) * (snow drift 0.5 | 1)`, then the Tiny floor, then the catch-up boost.
- The actual speed eases towards the target: `v += (target - v) * min(1, dt * 1.4)`.
- The only places that read the level by name are the Tiny speed floor, Pro's death flag, the per-level yeti tables for the human yeti and for survival mode, and the class leaderboards.

## 5. Results and profiles
- Show an overall leaderboard plus one per level that actually played, strongest class first, so a younger player can win their own class.
- At the end of a round (more than one player raced) give the overall winner an overall win and the best finisher of each level a class win, for named players only. Profiles keep `wins: {Tiny, Easy, Normal, Pro}` and the version `v`.

## Checklist
- [ ] Level is a parameter row, read per player every tick
- [ ] Steering assist, catch-up, crash stun, speed floor and yeti factor all implemented as above
- [ ] Pro is the only level that dies from a crash
- [ ] Survival mode and human-yeti mode have their own per-level tables
- [ ] Level descriptions listed once, shown in lobby and Settings
- [ ] Class leaderboards and per-level win counters
