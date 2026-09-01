# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Running the game

No build step or dependencies. Open directly or serve with any static server:

```bash
open index.html                  # macOS direct open
python3 -m http.server 8000      # then visit http://localhost:8000
```

## Architecture

Three files, no framework, no bundler:

- **`index.html`** — DOM structure: `<canvas id="board">` (300×600px) for the playfield, `<canvas id="next-canvas">` (120×120px) for the preview, sidebar HUD (`#score`, `#lines`, `#level`), and `#overlay` — a container of named `.screen[data-screen="…"]` panels (`start`, `message`), only one visible at a time.
- **`style.css`** — Dark/retro arcade theme; uses CSS variables, flexbox, and `backdrop-filter` on overlays.
- **`game.js`** — All game logic (`'use strict'`, no modules).

### game.js internals

| Concern | Key identifiers |
|---|---|
| Board state | `board` — `ROWS×COLS` matrix; `0` = empty, `1–8` = piece color index |
| Piece representation | `{ type, shape, x, y }` where `shape` is a 2-D matrix |
| Rotation | `rotateCW(shape)` — transpose + reverse; `tryRotate()` applies wall kicks `[0,±1,±2]` |
| Collision | `collide(shape, ox, oy)` — checks bounds and board occupancy |
| Game loop | `loop(ts)` via `requestAnimationFrame`; `dropAccum` tracks elapsed ms against `dropInterval` |
| Line clear | `clearLines()` — iterates board bottom-up, splices full rows, prepends empty row, returns the count cleared |
| Scoring | `LINE_SCORES = [0,100,300,500,800]` × `level`; hard drop +2/cell, soft drop +1/row |
| Speed | `dropInterval = speedForLevel(level)` → `max(100, 1000 − (level−1) × 90)` ms; level = `floor(lines/10) + 1` |
| Combo | `combo`/`maxCombo` — `lockPiece()` increments `combo` when `clearLines()` returns > 0, else resets it to 0 |
| Ghost piece | `ghostY()` — projects current piece down until collision; drawn at `globalAlpha = 0.2` |
| Screens | `showScreen(name)` shows one `#overlay .screen[data-screen=name]` and hides the rest; `hideOverlay()` hides the whole overlay |
| Persistence | `store.get(key, fallback)` / `store.set(key, value)` — `try/catch`-wrapped `localStorage` JSON helper; keys are `tetris-*` |
| State flags | `paused`, `gameOver`, `menuOpen`, `animId` (RAF handle); `gameInputEnabled()` gates the `keydown` handler on all three |
| Stats snapshot | `getStats()` → `{ score, lines, level, maxCombo }` |

### Game flow

Boot shows the `start` screen (no auto-start). `init(startLevel = 1)` → `spawn()` → `requestAnimationFrame(loop)`. Each frame: accumulate dt → auto-drop or `lockPiece()` → `draw()`. `lockPiece()` = `merge()` + `clearLines()` (updates combo) + `spawn()`. If `spawn()` immediately collides → `endGame()` → `showScreen('message')`. `loop()` returns before `draw()`/the next RAF request once `gameOver` is set, so no frame renders after the final lock — keep that early return when touching the loop.

## Tunable constants (top of game.js)

`COLS` (10), `ROWS` (20), `BLOCK` (30 px), `COLORS`/`PIECES` (arrays indexed 1–8), `LINE_SCORES`. If you change `COLS`/`ROWS`/`BLOCK`, update the canvas `width`/`height` attributes in `index.html` to match (`COLS×BLOCK` and `ROWS×BLOCK`). Adding a piece touches three places in lockstep: `COLORS[i]`, `PIECES[i]` (whose non-zero cells must hold the value `i`), and the hardcoded `8` in `randomPiece()`.
