---
name: multiplayer-lobby
description: Use when building a local (same-screen) game or app for 4 or more players using gamepads and/or keyboard. Covers detecting and listing controllers, joining, name entry, saved players and profiles, per-player difficulty, auto-start, pause and settings ownership, disconnects, split-screen fairness and leaderboards.
---

# Local 4+ player lobby

Use this whenever more than 3 people share one computer. Reference implementation: the "Ski Run" lobby (up to 5 players, one controller each, plus one keyboard player).

## 1. Detect and list controllers honestly
- Only list controllers that have actually been detected. Never show placeholder rows for pads that do not exist. A single "Press a button on each controller to join" card is shown while nobody has joined.
- Browsers expose a gamepad only after its first button press, so call the header count "N controllers active", not "connected".
- Persistent header: controller count, players joined. Tooltip: the `id` of each pad.
- Settings > Controllers: for each detected pad show index, `id`, `mapping`, button and axis counts, rumble support, the player it belongs to (or "not joined"), live pressed buttons, and a Buzz button (`vibrationActuator.playEffect('dual-rumble', ...)`) so people can tell which physical controller is which.
- Explain the limits: no serial number exists. Identical models share one `id`, and the index follows connection order and can change after a reconnect. Players should put a coloured sticker matching their player colour on each controller. Chrome and XInput commonly cap at 4 pads, so 5+ may need Firefox, DirectInput pads, or a keyboard player.

## 2. Joining
- No player-count picker. A device joins by pressing any button (Enter or Space on the keyboard; only one keyboard player). Identify players by device id (`gamepad.index` or `'kb'`), never by list position.
- Use a fixed array of 5 slots. The slot index sets colour and screen order for the whole session. When someone leaves, leave a gap; do not shift other players. Hide empty slots, and a new joiner takes the first gap.
- Ignore `event.repeat` for keys, and use edge detection for pad buttons so a held button does not skip steps.

## 3. Per-player setup: name, level, ready
Each player card shows three progress dots (name, level, ready), the current step, and a one-line hint for that step. Steps never need the mouse.
1. **Name.** A scrolling list of saved names (about 5 visible, selection highlighted) plus "+ New name" and "Skip". Typing on a gamepad uses a horizontal letter wheel: left/right changes the letter, with many letters shown either side (about 9 each way, fading), held direction auto-repeats (about 380 ms delay then 90 ms), up/down jumps several letters, A adds the letter, B deletes the last letter (cancels if empty), Start finishes. Use the full width of the screen for the typing card. Keyboard users just type, Enter finishes, Esc cancels.
2. **Level.** Chips, with the selected one checked and a short description under it. Left/right changes it, A confirms.
3. **Ready.** A large READY badge; B steps back.
- Button roles: A selects, B goes back, Start opens Settings in the lobby (and pauses in a race). Decide the physical button mapping once (the ski game maps A to gamepad button 1 and B to button 0), and show it in the Controls help.

## 4. Starting
- Start automatically when every player who has joined is ready, after a short cancellable grace period (about 2.5 s) in which a new join or an un-ready cancels the start. Ignore detected pads that have not joined; the player count comes from who joined.
- Then run a countdown with everybody lined up on one start line. Hold the countdown if a joined player's device is missing.

## 5. During play
- All screens show all players (name tags, colours) on the same course (shared seed). Split-screen is equal vertical panes with per-pane HUD, level, and a progress marker for every player.
- Pause with Start or Esc from any device, including during the countdown. Say who paused ("Paused by Ava (Controller 2)"). If the player has no name use "Player N", and if the device has not joined use "Controller N" or "Keyboard". The settings screen shows who opened it the same way. The pause menu must be navigable with the pad (up/down, A select, B resume). Pause automatically if a joined device disconnects.
- When a player is out (died, eaten) hold a visible overlay and the full animation before moving on. Do not end the round until those animations have finished.

## 6. Names, saved players and profiles
- Saved names live in local storage (most recent first). Settings > Players lists every saved name with its stats and a Delete button per name (two-click confirm; delete the name and its profile). Provide "Delete all".
- Profile per name: `{v, dist, races, fin, eaten, overall, wins:{<level>:n}}`. At the end of each round: add distance, count the race, count a yeti death, count an overall win and a class win when the winner finished and more than one player raced. Only named players get a profile. Keep `v` so the structure can be migrated later (for example Grand Prix handicaps).

## 7. Fairness for mixed ages and skill
- Difficulty is per player (assists, crash penalty, catch-up speed, enemy speed), not global.
- Show an overall leaderboard plus one per level, so strong players can still win their own class. Show results when everyone is finished or out, or a timeout after the first finisher, and display that countdown on the racing screens.

## 8. Settings page
Use a category rail plus a scrolling panel (see the universal settings UI skill): Gameplay (enemy behaviour), Players (level explanations, saved players), Controls (controller and keyboard table), Controllers (detected pads), Appearance (theme, accent, density), About (build stamp, roadmap, limitations). It must be fully usable with a gamepad: up/down move, left/right adjust sliders and selects, A activates, B or Start closes. Destructive buttons need an in-page two-click confirm (do not rely on `confirm()`, which can be blocked in embedded pages).

## Checklist
- [ ] Only detected controllers listed; header says "active"
- [ ] Join by button, fixed slot colours with gaps
- [ ] Name (wheel or list), level, ready, per device
- [ ] Auto-start when all joined are ready
- [ ] Pause and settings show who opened them
- [ ] Disconnect pauses; animations finish before results
- [ ] Per-level leaderboards and saved-player stats
- [ ] Settings usable with a gamepad
