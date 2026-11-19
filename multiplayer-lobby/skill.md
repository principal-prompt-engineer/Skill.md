---
name: multiplayer-lobby
description: Use when building a local (same-screen) game or app for 4 or more players using gamepads and/or keyboard. Covers one shared input layer, exclusive lobby input, generated layout and buttons, detecting controllers, joining, name entry, saved players and profiles, per-player difficulty, auto-start, pause and settings ownership, disconnects, split-screen fairness and leaderboards.
---

# Local 4+ player lobby

Use this whenever more than 3 people share one computer. Reference implementation: the "Ski Run" lobby (up to 5 players, one controller each, plus one keyboard player).

## 1. One input layer for everything
- Define one mapping table from logical action to physical source, and nothing else reads raw button numbers.

| Action | Gamepad | Keyboard |
|---|---|---|
| select / confirm / jump | button 1 ("A") | Enter, Space |
| back / delete | button 0 ("B"), Back (8) | Backspace, Esc |
| menu (Settings in lobby, pause in race) | Start (9) | Tab (lobby), Esc (race) |
| up / down / left / right | D-pad (12 to 15), left stick | arrows (and WASD in a race) |

- Every screen asks for logical actions (`select`, `back`, `menu`, directions, `any`). Hints, help tables and on-screen labels are generated from the same table, so they can never disagree with behaviour. Changing the mapping (for example swapping A and B) is a one-line edit.
- Produce per-device edge events once per frame (pressed this frame). Directions auto-repeat (about 380 ms delay, then 90 ms) and ignore the opposite direction for about 250 ms after a press, because analog sticks rebound past centre when released. All consumers use these events and never poll for themselves.

## 2. Input contexts: lobby buttons are exclusive
- Keep one explicit active context (`lobby`, `countdown`, `race`, `pause`, `settings`, `results`). Only the active context receives events. Gameplay actions (jump, steer, pause) must not fire in the lobby, and lobby actions (select, back, menu) must not fire in a race.
- The same physical button may mean different things per context (A selects in the lobby and jumps in a race). The context decides, and the frame that changes context consumes the event, so the press that opened a screen is not also handled by it (the Start that opens Settings must not also close it, and the A that readies must not also jump).
- While a lobby step waits for input, nothing else may claim its buttons. Do not bind global shortcuts, auto-pause, browser defaults (call `preventDefault` for arrows, Space, Backspace and Tab in the lobby) or any other game feature to buttons the lobby needs. If a global feature needs a button (Start opens Settings), reserve it explicitly in the lobby context and say so in the hint line. Text entry temporarily re-purposes buttons (Start = done, B = delete) and releases them afterwards.
- Ownership: when a player opens pause or settings, only that device's events are delivered until it closes. Release ownership if the device disconnects. A mouse is never blocked.
- Test it: in every context press every mapped button and confirm only the intended action fires, and press a button on the exact frame a context changes.

## 3. The layout is generated from state, one way
- Render the lobby from data only: `slots[]` (device, name, level, step) goes through a single `renderSlot(slot)` function used for every player and every step. No hand-built layout per player, and no special cases for player 1 or for 4+ players.
- Every card has the same structure: header (player badge, device, step dots), body (varies by step but uses the same classes), hint line. All cards sit in one responsive grid (`auto-fit` with `minmax`), so 1 to 5 players lay out identically. Only a typing card spans the full row.
- Order is a pure function of state (alphabetical by name, unnamed players last), not a side effect of joining. Colour comes from the slot index and never changes when the order changes.
- Re-render only when a serialized state key changes. After any sequence of joins and leaves the layout must be exactly what a fresh render of the same state gives.
- Pause, settings and results cards use the same CSS tokens (variables) so themes change every screen the same way.

## 4. Buttons are generated the same way
- Every on-screen button (lobby chips, pause menu items, settings controls, result actions) is built by one helper from `{id, label, action}`, with the same markup, classes, focus highlight and activation logic. Menus are arrays of these items.
- The list a gamepad can navigate is computed from the visible, enabled items, so adding a button makes it reachable by controller automatically.
- Mouse click, gamepad select and keyboard Enter all call the same `action`.
- Destructive buttons use one shared two-click confirm helper (do not rely on `confirm()`, which can be blocked in embedded pages).

## 5. Detect and list controllers honestly
- List only controllers that have been detected. Never show placeholder rows for pads that do not exist. While nobody has joined, show a single "Press a button on each controller to join" card.
- Browsers expose a gamepad only after its first button press, so the header says "N controllers active", not "connected". The header also shows how many players joined, with each pad's `id` in a tooltip.
- Settings > Controllers: for each detected pad show index, `id`, `mapping`, button and axis counts, rumble support, which player owns it (or "not joined"), live pressed buttons, and a Buzz button (`vibrationActuator.playEffect('dual-rumble', ...)`) to find the physical controller.
- Explain the limits: no serial number exists, identical models share one `id`, and the index follows connection order and can change after a reconnect. Players should put a coloured sticker matching their player colour on each controller. Chrome and XInput commonly cap at 4 pads, so 5+ may need Firefox, DirectInput pads, or a keyboard player.

## 6. Joining
- No player-count picker. A device joins by pressing any button (Enter or Space on the keyboard; only one keyboard player). Identify players by device id (`gamepad.index` or `'kb'`), never by list position.
- Use a fixed array of 5 slots. The slot index sets colour for the whole session. When someone leaves, leave a gap and do not shift anyone. Hide empty slots, and give a new joiner the first gap.
- Ignore `event.repeat` for keys, and use edge detection for pad buttons so a held button does not skip steps.

## 7. Per-player setup: name, level, ready
Each card shows three progress dots (name, level, ready), the current step and a one-line hint for it. No step needs the mouse.
1. **Name.** A scrolling list of saved names (about 5 visible, selection highlighted, alphabetical) plus "+ New name" and "Skip". Gamepad typing uses a horizontal letter wheel: left/right changes the letter with about 9 letters shown either side (fading), held direction auto-repeats, up/down jumps several letters, select adds the letter, back deletes the last letter (cancels if empty), menu/Start finishes. Use the full width for the typing card. Keyboard users just type; Enter finishes and Esc cancels.
2. **Level.** Chips with the selected one checked and a short description. Left/right changes it, select confirms.
3. **Ready.** A large READY badge; back steps back.

## 8. Starting
- Start automatically when every player who has joined is ready, after a short cancellable grace period (about 2.5 s) in which a new join or an un-ready cancels the start. Ignore detected pads that have not joined, since the player count comes from who joined.
- Then run a countdown with everybody lined up on one start line. Hold the countdown if a joined player's device is missing.

## 9. During play
- All screens show all players (name tags, colours) on the same course (shared seed). Split-screen is equal vertical panes with a per-pane HUD, level, and a progress marker for every player.
- Pause with the menu action from any device, including during the countdown, and say who paused ("Paused by Ava (Controller 2)"). Use "Player N" if there is no name, and "Controller N" or "Keyboard" if the device has not joined. Settings shows who opened it the same way. The pause menu is navigable with the pad (up/down, select, back to resume). Pause automatically if a joined device disconnects.
- When a player is out (died, eaten) hold a visible overlay and the full animation before moving on, and do not end the round until those animations finish.

## 10. Names, saved players and profiles
- Saved names live in local storage. Settings > Players lists every saved name (alphabetical) with its stats and a Delete button per name (two-click confirm; delete the name and its profile), plus "Delete all".
- Profile per name: `{v, dist, races, fin, eaten, overall, wins:{<level>:n}}`. At the end of each round add distance, count the race, count a yeti death, and count an overall win and a class win when the winner finished and more than one player raced. Only named players get a profile. Keep `v` so the structure can be migrated later (for example Grand Prix handicaps).

## 11. Fairness for mixed ages and skill
- Difficulty is per player (assists, crash penalty, catch-up speed, enemy speed), not global.
- Show an overall leaderboard plus one per level, so strong players can win their own class. Show results when everyone is finished or out, or after a timeout following the first finisher, and display that countdown on the racing screens.

## 12. Settings page
Use a category rail plus a scrolling panel: Gameplay, Players (level explanations, saved players), Controls (table generated from the mapping in section 1), Controllers, Appearance, About. It must be fully usable with a gamepad (up/down move, left/right adjust sliders and selects, select activates, back or menu closes).

## Checklist
- [ ] One mapping table; hints and help generated from it
- [ ] Explicit input contexts; lobby buttons exclusive; context-change frame consumed
- [ ] Browser defaults suppressed for lobby keys
- [ ] Lobby rendered from state by one `renderSlot`, one grid, deterministic order
- [ ] All buttons built by one helper; pad-navigable list computed from the DOM
- [ ] Only detected controllers listed; header says "active"
- [ ] Join by button, fixed slot colours with gaps
- [ ] Name (wheel or list), level, ready, per device
- [ ] Auto-start when all joined are ready
- [ ] Pause and settings show who opened them, and only that device controls them
- [ ] Disconnect pauses; animations finish before results
- [ ] Per-level leaderboards and saved-player stats
