# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Vanilla JavaScript Tetris — HTML5 Canvas + CSS, no build step, no dependencies, no `package.json`, no tests. Three source files: `index.html`, `style.css`, `game.js`. UI text and `README.md` are Spanish.

## Run

Open `index.html` directly, or serve statically:

```bash
python3 -m http.server 8000   # then open http://localhost:8000
```

No build, lint, or test commands exist.

## Architecture (`game.js`)

Single global-scope script (`'use strict'`), loaded via plain `<script src>` at the end of `<body>`. All game state lives in module-level `let` vars (`board`, `current`, `next`, `score`, `lines`, `level`, `dropInterval`, `paused`, `gameOver`, `animId`, ...) reset by `init()`, which runs on load and on the restart button. No classes, no modules.

- **Board**: `ROWS × COLS` matrix; each cell is `0` (empty) or a color index `1–7` keying into `COLORS` / `PIECES`. The same index identifies piece type, color, and stored block.
- **Pieces**: square matrices in `PIECES` (index 1–7). Rotation = transpose + row-reverse in `rotateCW` (no SRS state; `tryRotate` overwrites `current.shape` destructively).
- **`collide(shape, x, y)`**: the single source of truth for legality — bounds + overlap. Movement, rotation, ghost projection, and lock detection all gate on it. Cells with `ny < 0` (above board) are allowed so pieces can spawn/rotate at the top.
- **Wall kicks** (`tryRotate`): tries x-offsets `[0,-1,1,-2,2]` against the rotated shape; first non-colliding offset wins, else rotation is dropped.
- **Game loop** (`loop`): `requestAnimationFrame` + time accumulator (`dropAccum`); steps the piece down one row when `dropAccum >= dropInterval`, then resets `dropAccum` to `0` (not subtracting the remainder). `dropInterval = max(100, 1000 - (level-1)*90)`.
- **Lock cycle**: `lockPiece()` → `merge()` (write shape into `board`) → `clearLines()` → `spawn()`. No lock delay — a piece locks the instant it cannot move down. `spawn()` promotes `next` to `current`, generates a new `next`, and calls `endGame()` if the new piece already collides.
- **Input**: one `keydown` listener on `document`. `KeyP` toggles pause and returns early; every other key is ignored while `paused || gameOver`. `Space` calls `preventDefault()` to stop page scroll.
- **Scoring**: `LINE_SCORES[cleared] * level`; soft drop +1/row, hard drop +2/row. Level rises every 10 lines.
- **Rendering**: `draw()` repaints each frame — grid, board, ghost (`ghostY()` at `alpha 0.2`), then current piece. `drawNext()` paints the preview canvas only on spawn.

## Editing notes

- `index.html` `#board` canvas is hardcoded `300×600` = `COLS*BLOCK × ROWS*BLOCK`. Changing `COLS`, `ROWS`, or `BLOCK` in `game.js` requires updating the canvas `width`/`height` attributes to match.
- The preview is separate: `drawNext()` uses its own local `NB = 30` and centers the shape in a fixed 4×4 grid matching `#next-canvas` at `120×120`. It does not follow `BLOCK`; pieces larger than 4×4 would overflow.
- Adding a piece type means adding aligned entries to both `PIECES` and `COLORS` at the same index, and `randomPiece()`'s hardcoded `* 7` must be updated too.
- `endGame()` and `togglePause()` both share `#overlay`; they set `overlay-title`/`overlay-score` text and toggle the `.hidden` class.
- `endGame()` calls `cancelAnimationFrame(animId)`, but when it is reached from inside `loop` (via `lockPiece` → `spawn`), `loop` still schedules the next frame afterwards, so the loop keeps running after game over — only input is blocked. Touching game-over or restart logic means reckoning with this.
- `README.md` documents controls, scoring table, and the tunable constants in Spanish; keep it in sync when changing any of them.

## GitHub automation

`.github/workflows/` runs `anthropics/claude-code-action@v1` in three places: `claude.yml` (responds to `@claude` mentions in issues/PR comments/reviews), `claude-code-review.yml` (automatic review on PR open/sync), and `issue-triage.yml` (labels and comments on newly opened issues, with a Spanish prompt and a narrow `--allowedTools` allowlist). All authenticate via the `CLAUDE_CODE_OAUTH_TOKEN` secret. Editing a workflow's `allowedTools` or permissions block changes what the bot can do in CI.
