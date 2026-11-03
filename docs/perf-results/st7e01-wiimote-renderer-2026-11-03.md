# Fortune Street (ST7E01): Wii Remote input and renderer choice — 2026-11-03

## Question

With emulated Wii Remote input added (patch 0068), short button taps on the
first "Create a save file?" dialog moved the cursor only some of the time. Was
input being lost between the browser and the game, or was something else
going on?

## Setup

- Machine: Intel i9-14900KF, NVIDIA RTX 5080, Windows, Chrome 156 (headed).
- Core: `517dbbc4df9cea6f07dd8b2ad0752610840e8759eeaaa2ae78ccb4a35c6e8e7f`
  (Wii Remote input, no diagnostics).
- Disc: Fortune Street (USA), `ST7E01`, NTSC-U. Not committed.
- Harness: Playwright-driven headed Chrome. Keys are held for a fixed time with
  `keyboard.down`/`keyboard.up`; the Yes/No selection is read back from
  screenshots of the canvas.
- URLs:
  - software: `http://127.0.0.1:8090/?metrics=1`
    (before this change the default renderer was software for this disc)
  - hardware: `http://127.0.0.1:8090/?metrics=1&video=wgpu`

## Input path is lossless

A diagnostic build (not shipped) counted every Wii Remote report at the
emulated Bluetooth layer:

| Measure | Result |
|---|---|
| Software renderer, 10 × 120 ms taps on the dialog | 10 of 10 press edges delivered over L2CAP; cursor moved 3 times |
| Hardware renderer, whole session (menus to board) | 96 of 96 browser press edges delivered over L2CAP |
| Reports dropped by the 100-entry ACL queue | 0 (peak depth 12) |
| Report rate | ~187/s (200 Hz emulated) |

Every press reached the game. The misses happened inside it.

## The game was running at ~4 frames per second

On the software renderer the dialog and title screen changed only 2–4 times a
second (`visualChangeFps`) while VI/core FPS sat at ~60 and game speed at
~100%. The game logic itself was only advancing a few times a second, so a tap
shorter than its sampling interval was never seen.

| Renderer | Scene | Unique visual FPS | Game speed |
|---|---|---|---|
| software | title globe | 4 | 96–107% |
| hardware (`video=wgpu`) | boot / save dialog | 26–30 | 94–103% |
| hardware (`video=wgpu`) | title, main menu | 30 | 99–104% |

Cursor moves per 10 alternating Left/Right taps on the save dialog:

| Hold | software | hardware |
|---|---|---|
| 40 ms | 1 | 9 (+1 unreadable mid-transition shot) |
| 80 ms | 0 | 10 |
| 120 ms | 3 | 10 |
| 200 ms | 4 | 10 |
| 300 ms | 6 | — |

Fortune Street renders at 30 fps on hardware, as it does on a Wii.

## Frame check

Checked by eye on hardware: health/strap screen, save dialog, title globe,
title logo, main menu, save-slot and Mii selection, rules and main menu,
character select, board select, settings, the castle intro and the Castle
Trodain board with HUD. The title globe and save dialog match the software
render.

## Decision

`ST7E01` defaults to the hardware renderer in `src/game-profiles.js`.

## Open: board speed

On the Castle Trodain board, with a cold JIT cache, hardware averaged ~72% game
speed over a 60 s sample (range 25–108%; ~45–66% with the main board camera,
~100% on static screens). That still needs profiling, and is the next target.
