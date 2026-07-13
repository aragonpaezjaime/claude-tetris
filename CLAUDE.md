# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Vanilla JavaScript Tetris — HTML5 Canvas + CSS, no build step, no dependencies, no `package.json`, no tests. Three files: `index.html`, `style.css`, `game.js`. UI text is Spanish.

## Run

Open `index.html` directly, or serve statically:

```bash
python3 -m http.server 8000   # then open http://localhost:8000
```

No build, lint, or test commands exist.

## Architecture (`game.js`)

Single global-scope script. All game state lives in module-level `let` vars (`board`, `current`, `next`, `score`, `lines`, `level`, `dropInterval`, ...) reset by `init()`. No classes, no modules.

- **Board**: `ROWS × COLS` matrix; each cell is `0` (empty) or a color index `1–7` keying into `COLORS` / `PIECES`. The same index identifies piece type, color, and stored block.
- **Pieces**: square matrices in `PIECES` (index 1–7). Rotation = transpose + row-reverse in `rotateCW` (no SRS state; rotation is destructive to `current.shape`).
- **`collide(shape, x, y)`**: the single source of truth for legality — bounds + overlap. Movement, rotation, ghost projection, and lock detection all gate on it. Cells with `ny < 0` (above board) are allowed so pieces can spawn/rotate at the top.
- **Wall kicks** (`tryRotate`): tries x-offsets `[0,-1,1,-2,2]` against the rotated shape; first non-colliding offset wins, else rotation is dropped.
- **Game loop** (`loop`): `requestAnimationFrame` + time accumulator (`dropAccum`); steps the piece down one row when `dropAccum >= dropInterval`. `dropInterval = max(100, 1000 - (level-1)*90)`.
- **Lock cycle**: `lockPiece()` → `merge()` (write shape into `board`) → `clearLines()` → `spawn()`. `spawn()` promotes `next` to `current`, generates a new `next`, and calls `endGame()` if the new piece already collides.
- **Scoring**: `LINE_SCORES[cleared] * level`; soft drop +1/row, hard drop +2/row. Level rises every 10 lines.
- **Rendering**: `draw()` repaints each frame — grid, board, ghost (`ghostY()` at `alpha 0.2`), then current piece. `drawNext()` paints the preview canvas only on spawn.

## Editing notes

- `index.html` `#board` canvas is hardcoded `300×600` = `COLS*BLOCK × ROWS*BLOCK`. Changing `COLS`, `ROWS`, or `BLOCK` in `game.js` requires updating the canvas `width`/`height` attributes to match.
- Adding a piece type means adding aligned entries to both `PIECES` and `COLORS` at the same index.
- `endGame()` and `togglePause()` both share `#overlay`; they set `overlay-title`/`overlay-score` text and toggle the `.hidden` class.
