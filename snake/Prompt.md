# Snake — Self-Contained Build Guide

This single document is the **complete reference**. Save the game file below
and you have the whole game — no other files, folders, or prior context are
needed. Everything (the game, its look, controls, and how it works) is in here.

## What this is

A classic **Snake** game with a **Nokia-3210 LCD** look, that runs from **one
single HTML file** — no build step, no network, no external images or sounds.
All rendering is procedural on a 2D canvas; all sound is synthesized with the
Web Audio API.

## Quick start

1. Copy the full file below into a new file named `snake.html`.
2. Open `snake.html` in any modern browser (Chrome, Edge, Firefox).
3. Play. (Sound starts after your first key press — browser autoplay rule.)

## The game

A classic Snake game. You steer a snake around a 30×20 grid, eating food to
grow longer. Hitting a wall or your own body ends the run.

### Screens

- **Menu** — the start screen with two items: `GAME START` and `HIGHSCORE`.
  Move with `↑`/`↓` (or `W`/`S`), select with `Enter`/`Space`.
- **Play** — the grid with the snake and food, plus a side panel showing
  `SCORE`, `RESCUE` (number of runs started), and the `HIGH SCORE`.
- **Name entry** — after a game over you enter a 6-character name for the
  highscore. `←`/`→` move the cursor, `↑`/`↓` change the character under it,
  letters/digits type directly; `Enter`/`Space` submits.
- **Highscore** — the top-5 list (rank, name, score). `Enter`/`Space` returns
  to the menu.

### Gameplay

- The snake starts as a **single segment** in the center of the grid, moving
  right.
- Each tick (every `interval` milliseconds) the head advances one cell in the
  current direction.
- **Eating food**: +10 points, the snake grows by one segment, a new food
  spawns on a random empty cell, and the speed increases.
- **Speed**: starts at 120 ms/tick, drops by 5 ms per 50 points, minimum
  60 ms/tick.
- **Collision**: the head leaving the grid (wall) or landing on any body
  segment ends the game and moves to name entry.
- **Score**: +10 per food. The run's score is submitted to the local
  highscore list (top 5, stored in `localStorage` key `gvf_snake_hs`).

### Visuals (Nokia 3210 LCD)

- Background: LCD green `#8BAC0F`.
- Objects (snake, food, text): dark green `#0F380F`.
- Grid lines / shadows: medium green `#306230`.
- Highlights: light green `#9BBC0F`.
- The snake is drawn as connected rounded blocks (head slightly larger), the
  food as a small block, and the board as a subtle grid.

### Audio (Web Audio synth)

- **food** — short ascending blip (square wave, 660→990 Hz, 0.12 s).
- **gameOver** — descending tone (sawtooth, 440→70 Hz, 0.7 s).
- Synthesized in code; no audio files. The `AudioContext` is created lazily
  on the first user gesture.


## How it works (technical)

- **One HTML file** — inline HTML/CSS/JS, no external files.
- **Canvas 2D** — `<canvas id="game">` sized to the window; the game draws a
  green LCD background and dark-pixel sprites each frame.
- **runFrame harness** — the game script runs inside `runFrame(canvas, ctx,
  width, height)`, driven by a `requestAnimationFrame` loop. All game state
  hangs off `canvas._gvfSnake` (persists across frames).
- **Local highscores** — top 5, stored in `localStorage` (key `gvf_snake_hs`).
  No server, no network.
- **Layout** — the cell size `SZ` is derived from the window, so the board
  and side panel always fit (no clipping on any window size).

## Controls

| Key | Action |
|---|---|
| `←` `→` `↑` `↓` or `A` `D` `W` `S` | move the snake |
| `Enter` / `Space` | select in menu / confirm |
| `ESC` | back to main menu (always) |
| `↑`/`↓`/`←`/`→` + letters/digits | name entry (after game over) |

## Notes

- **No network** — the game makes zero network requests; highscores are local.
- **No assets** — every sprite and sound is generated in code at runtime.
- **Browser** — any modern browser; the canvas fills the whole window.
