---
name: adaptive-assist-difficulty
description: Use when a same-screen race game needs a difficulty that keeps mixed-skill players close together by changing each player's help live, based on how far they are behind the leader (rubber-band assist). Defines the "Adaptive" level where the leader gets Normal, the last Adaptive player gets full Tiny-level help and everyone in between is blended. Includes the formulas, edge cases, tuning constants, HUD and the lobby and profile changes. Use it whenever someone asks for dynamic difficulty, catch-up mechanics, rubber banding or "everyone should finish together", even if they do not say "adaptive".
---

# Adaptive assist level (dynamic difficulty)

Read `per-player-difficulty` first: it defines the Tiny and Normal parameter sets that this level blends between. Also use with `multiplayer-lobby`. Adaptive is a fifth level the player picks in the lobby. Unlike the fixed levels, its parameters change every tick.

## 1. What it does
Adaptive players get exactly as much help as they need to stay near the leader.
- In the lead: Normal, so no help.
- Last among Adaptive players: Tiny's full help.
- In between: proportional to how far back they are.
- As the pack closes up the help fades, so a tight race plays like Normal and a spread-out race helps the back markers most.

Measure by distance behind the leader, not by rank. A continuous measure never jumps when two skiers swap places, and it lets help fade smoothly as the gap closes.

## 2. Who counts
- **The lead skier** is the front-most skier of any level (Tiny, Easy, Normal, Pro or Adaptive). Skiers who have finished still count, because they really are ahead. Ignore eliminated skiers: they are frozen where they died, and chasing a ghost would hand out help for nothing. Ignore the human yeti in Yeti hunter mode, who is not racing.
- **The last player** is the Adaptive skier with the biggest gap, considering only Adaptive skiers who are still racing (not finished, not eliminated). Skiers on fixed levels never influence it. Their help cannot change, so letting a far-behind fixed-level player define "last" would only water down everyone else's help.
- Position is the distance down the course in every mode, even modes scored by stars or gates. It is the one measure all modes share. If you would rather rubber-band on score in those modes, swap the gap measure and keep everything else.

## 3. Each tick
Do this once per tick, from a snapshot of positions taken before anyone moves. Otherwise skiers processed earlier in the loop see different positions from those processed later, and update order decides who gets help.

```
skiers = players who have played, excluding the human yeti
lead   = max y over skiers that are not eliminated          // finished skiers included
group  = Adaptive skiers that are not eliminated and not finished
span   = max(FULL_GAP, max(lead - y) over group)
for p in group:
    target   = clamp((lead - p.y) / span, 0, 1)
    p.assist += (target - p.assist) * min(1, dt * RATE)      // p.assist is t, 0..1
```

- `FULL_GAP = 300` px (30 m on the distance meter, about 0.75 s at top speed). Without it a lone Adaptive player 20 px behind the leader would count as "last", receive full Tiny help, shoot past, lose all help in front, crash and drop back, over and over. With it, full help only appears when the last Adaptive skier is at least this far back. Set it to 0 for the pure "last gets full help" rule.
- `RATE = 1.5` per second (about 0.7 s to react). Smoothing stops help from jumping when the leader changes or a skier is eliminated.
- `t` starts at 0 for everyone at the start of every round and is never saved. Freeze it while paused and during the countdown.

## 4. Blending the level parameters
`t = p.assist`. At 0 the values equal Normal, at 1 they equal Tiny, so retuning those two table rows retunes Adaptive too.

| Parameter           | Normal (t = 0) | Tiny (t = 1) | Blended value                                    |
|---------------------|----------------|--------------|--------------------------------------------------|
| Steering assist `as`| 0              | 1            | `t`                                              |
| Catch-up cap `cu`   | 0              | 0.7          | `0.7 * t`                                        |
| Crash stun          | 1.2 s          | 0 s          | `1.2 * (1 - t)`                                  |
| Speed floor         | none           | 280          | `280 * t`                                        |
| Yeti speed factor   | 1              | 0            | `1 - t` (survival mode: `1 - 0.4 * t`)           |
| Dies on crash       | no             | no           | never                                            |

Rules for the awkward rows:
- **Crash stun:** if the blended stun is under 0.2 s, use Tiny's soft crash (keep going at half speed, no stop). A stop of a few milliseconds followed by a heading reset feels like a glitch, not help.
- **Yeti:** spawn it only while the factor is at least 0.1. Once spawned it never despawns; it simply slows as help rises and speeds up as help falls. In Yeti hunter mode there is no AI yeti. An Adaptive player who is the human yeti moves at Normal speed (360).
- **Catch-up distance:** use the same live lead as above, `1 + clamp((lead - y) / 600, 0, cu)`, for Adaptive skiers only. Fixed levels keep their existing behaviour exactly.
- Everything else (base speed, obstacle handling, steering rate) is unchanged.

## 5. Edge cases
- One Adaptive player: at the front they get Normal. Behind, help is `gap / FULL_GAP` and never more.
- Everyone Adaptive: the leader gets Normal, the last gets Tiny, a pack inside `FULL_GAP` gets only partial help.
- The leader finishes: they stay the lead, the others fall into the large-gap range and get strong help until the results timeout. That is intended; it helps them finish.
- A skier finishes or is eliminated: they leave the Adaptive group at once, so the next skier back becomes "last".
- Survival mode (no finish line): same rule, the lead is the farthest live skier.

## 6. Feedback to players
- Lobby chip "Adaptive" with a short description, for example: "Help adjusts as you race: lots when you trail the leader, none when you lead, so the whole group stays close. Never dies from crashing."
- In each Adaptive player's HUD pane show the live help level next to the level name, for example "Adaptive - 62%". It reads 0% at the front, so players and parents can see the help working.
- Explain it in Settings > Players next to the other levels.

## 7. Plugging it into the lobby and profiles
- Append Adaptive as the fifth level (index 4) so existing level ids and saved data keep their meaning. Chip cycling and any loops use the level count, not a hard-coded 4. Keep the class leaderboard display order separate from the ids, for example Pro, Normal, Adaptive, Easy, Tiny.
- Adaptive is its own class on the leaderboard and gets its own win counter.
- Profiles: add an `Adaptive` counter under `wins`, bump the structure version `v`, and treat a missing counter as 0 so old saves load.
- Per-level tables indexed by level (human yeti speed, survival yeti factor) get a fifth entry copying Normal; the Adaptive code path overrides it as in section 4.

## 8. Tuning
These are starting values. Playtest with a mix of one strong player, one Adaptive player and one child on Tiny, then adjust.
- `FULL_GAP`: smaller means a tighter pack and a more visible rubber band; larger means freer racing. Useful range 200 to 600 px.
- `RATE`: faster feels twitchy, slower lets gaps open before help arrives.
- To drop a blended row (for example keep the yeti unchanged), remove it from the table in section 4; nothing else depends on it.

## Checklist
- [ ] Lead = front-most live or finished skier of any level; last = Adaptive skiers still racing only
- [ ] Assist `t` is gap-based, normalised by `max(FULL_GAP, last gap)`, smoothed, reset each round
- [ ] Targets computed once per tick from a position snapshot
- [ ] Steering assist, catch-up, stun, speed floor and yeti blended as in the table; never dies from a crash
- [ ] Fixed levels behave exactly as before
- [ ] Fifth level appended; profiles migrated; own class on the leaderboard
- [ ] Live help percentage shown in the HUD
