# Snake — Build Specification (Nokia 3210 LCD Style)

This document contains **no code** — only a complete technical
specification. From it, `snake.html` (a single self-contained HTML
document with Canvas2D rendering and Web Audio sound) can be implemented —
no build step, no network, no external images or sounds.

## Target format

- A single `.html` file, inline HTML/CSS/JS.
- Canvas fills the entire browser window (`width: 100vw; height: 100vh`),
  background `#8BAC0F` (prevents flash on load).
- No external requests, no asset loading.

## Architecture

- One `<canvas id="game">`, sized to the window.
- Render loop via `requestAnimationFrame`, calling a function
  `runFrame(canvas, ctx, width, height)` each frame.
- All game state hangs off `canvas._gvfSnake` (survives across frames,
  initialized lazily on first call if not present).
- Sound object lives at `canvas._gvfSnakeSounds`, also lazily initialized.
- A single `keydown` listener (capture phase) is registered globally and
  removed before re-registering on a fresh run, to avoid double bindings.
  It self-terminates (removeEventListener) once `canvas.isConnected` is
  false.

## Layout calculation

- Grid: `COLS = 30`, `ROWS = 20`.
- Cell size `SZ = floor(min(width / (COLS + 8), height / ROWS) * 0.95)`
  — ensures the board and side panel fit at any window size without
  clipping.
- Board origin: `OX = floor((width - SZ * (COLS + 7)) / 2)`,
  `OY = floor((height - SZ * ROWS) / 2)`.
- Side panel starts at `PX = OX + SZ * COLS + 10`, width `SZ * 6`.
- Base font size `fs = max(10, round(SZ * 0.9))`.

## State model

Four modes (`S.mode`): `menu`, `play`, `name`, `highscore`.

**Global state (`S`):**
- `mode`
- `menuIndex` (0 or 1)
- `game` — see below
- `nameChars`: array of 6 indices into `CHARS`
- `nameCursor`: 0–5
- `hsList`, `hsLoaded`, `hsLoading`, `hsMessage`, `hsFlashTime`

**Game state (`S.game`), freshly created per run:**
- `snake`: array of `{x, y}`, starts as **one** segment at
  `{x: floor(COLS/2), y: floor(ROWS/2)}`
- `dir` / `nextDir`: `{x: 1, y: 0}` (starts moving right)
- `food`: a random free cell (see food logic)
- `score`: 0
- `resets`: count of runs started, persists across runs (incremented on
  ESC-abort from `play`)
- `lastMove`: timestamp of the last tick
- `interval`: 120 (ms)
- `dead`: false

## Constants

- `CHARS = 'ABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789'`
- `MENU_ITEMS = ['GAME START', 'HIGHSCORE']`
- `HS_KEY = 'gvf_snake_hs'` (localStorage key)

## Highscore persistence

- Storage: `localStorage`, key `gvf_snake_hs`, value = JSON array.
- Load: `JSON.parse(localStorage.getItem(HS_KEY)) || []`, on error → `[]`.
- Save (`postHS(name, score)`):
  1. Truncate name to 6 chars, uppercase, right-pad with spaces
     (`slice(0,6).padEnd(6,' ')`).
  2. Append entry `{name, score}`.
  3. Sort list descending by `score`.
  4. Truncate to top 5 (`splice(5)`).
  5. Write back to `localStorage`.
- No server, no network — everything is local/synchronous, but modeled as
  `async` functions (for future extensibility), with loading states
  `hsLoading` / `hsMessage` (`'LOADING...'`, `'SAVING...'`, `'SAVED'`,
  `'LOAD FAILED'`).

## Food logic

- `rndFood(snake)`: rolls a random `{x, y}` on the grid until it finds a
  cell not occupied by any snake segment.

## Game logic (tick, only in `play` mode)

Runs when `now - lastMove >= interval`:

1. `lastMove = now`, `dir = nextDir`.
2. New head = `snake[0] + dir`.
3. **Collision check**: head outside the grid (0..COLS-1 / 0..ROWS-1)
   OR head lands on any existing snake segment →
   - Play `gameOver` sound.
   - Mode → `name`, reset `nameChars` to `[0,0,0,0,0,0]`,
     `nameCursor = 0`.
4. Otherwise: insert head at the front (`unshift`).
   - If head hits the food:
     - Play `food` sound.
     - `score += 10`.
     - Roll new food.
     - `interval = max(60, 120 - floor(score / 50) * 5)` (5 ms faster
       every 50 points, minimum 60 ms).
   - Otherwise: remove the last segment (`pop`) — movement without
     growth.

## Controls (keyboard)

A single global `keydown` handler, branching on `S.mode`:

- **Always**: `Escape` → if `mode === 'play'`, `resets++`; then back to
  `menu` (`menuIndex = 0`).
- **Mode `menu`**:
  - `↑`/`W` → decrement `menuIndex` (wraps across 2 entries)
  - `↓`/`S` → increment `menuIndex` (wraps)
  - `Enter`/`Space` → index 0: start a new game (`resets` is preserved);
    index 1: open the highscore screen (loads the list if not already
    loaded)
- **Mode `play`**: arrow keys/`WASD` set `nextDir`, **never** allowing a
  180° reversal (e.g. `↑` is ignored while `dir.y === 1`, i.e. currently
  moving down — analogous for all four directions).
- **Mode `name`**:
  - `↑`/`↓` → cycle the character under the cursor through the `CHARS`
    alphabet forward/backward (wraps)
  - `←`/`→` → move the cursor between the 6 positions (clamped to 0 and
    5, no wrap)
  - Letter/digit keys → set the character directly, cursor then advances
    one position (unless already at position 5)
  - `Enter`/`Space`: if cursor < 5 → cursor++; if cursor === 5 → submit
    the score (`submitScore`) and switch to `highscore` mode
- **Mode `highscore`**: `Enter`/`Space` → back to the menu.

## Sound (Web Audio, synthesized — no files)

- `AudioContext` is created **lazily** on the first needed tone (browser
  autoplay rule); if suspended, `resume()` before playing.
- Tone helper: oscillator + gain node, gain envelope ramps up linearly
  (`0.0001 → vol` over 0.01 s), then decays exponentially (`vol → 0.0001`
  over the tone's duration). The oscillator stops shortly after the
  envelope ends.
- **food**: square wave, start frequency 660 Hz, ramps exponentially to
  990 Hz, duration 0.12 s, volume 0.3.
- **gameOver**: sawtooth wave, start frequency 440 Hz, ramps exponentially
  to 70 Hz, duration 0.7 s, volume 0.3.

## Color palette (Nokia 3210 LCD)

| Role | Hex | Usage |
|---|---|---|
| Background | `#8BAC0F` | Canvas base color |
| Dark | `#0F380F` | Snake head, text, food outline, active menu row |
| Mid | `#306230` | Snake body, grid lines, muted text |
| Light | `#9BBC0F` | Highlight dot inside the snake head |

`ctx.imageSmoothingEnabled = false` for the crisp pixel look.

## Rendering per screen

**Shared (`drawBoardAndSide`)**: dark 2px border around the board and
side panel, a very subtle grid (0.5px, mid green) over the board.

**`menu`**: title "SNAKE" centered at the top, a dashed divider below it,
the two menu items centered one below the other — the active entry
rendered as a filled dark bar row with light text bracketed by `> ... <`,
the inactive entry as muted text. Footer "W/S  ENTER".

**`play`**: the board with the snake (head dark + light inner dot, body
mid-green, each drawn as full cells) and the food (a dark square with a
light "hole" in the middle, looking like a small ring icon). The side
panel shows, stacked vertically: SCORE, LEN (snake length), RST (`resets`
count), SPD (percentage from `(120 - interval) / 60 * 100`, rounded) —
each a muted label above a bold value. Below that, the top-3 highscores
("BEST" label, then name left-aligned / score right-aligned per row).

**`name`**: "GAME OVER" at the top, the current score muted below it,
"ENTER NAME:", then 6 character boxes side by side — the active box
filled dark with a light character, inactive boxes outlined with a dark
character. Footer "UP/DN  LT/RT  ENTER".

**`highscore`**: title "HIGH SCORES", divider, then depending on state
either "LOADING...", "NO SCORES YET", or the list (rank, name, score
right-aligned). If `hsMessage` is set, it's shown above the footer.
Footer "ENTER / ESC".

## Controls reference table

| Key | Action |
|---|---|
| `←` `→` `↑` `↓` or `A` `D` `W` `S` | move the snake |
| `Enter` / `Space` | confirm menu selection |
| `Esc` | back to the main menu, at any time |
| `↑`/`↓`/`←`/`→` + letters/digits | name entry after game over |

## Constraints

- No network access — all highscores are purely local.
- No external assets — every sprite and sound is generated at runtime in
  code.
- Must run in any modern browser (Chrome, Edge, Firefox) with no build
  step; the canvas fills the entire window.
