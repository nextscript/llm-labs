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

## The complete game file

This is the entire game in one file. Save it as `snake.html`:

```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>Snake</title>
<style>
  html,body{margin:0;padding:0;width:100%;height:100%;background:#8BAC0F;overflow:hidden;}
  #game{display:block;width:100vw;height:100vh;}
</style>
</head>
<body>
<canvas id="game"></canvas>
<script>
(function(){
  'use strict';
  const canvas=document.getElementById('game');
  canvas.width=window.innerWidth;
  canvas.height=window.innerHeight;
  const ctx=canvas.getContext('2d');
  const width=canvas.width, height=canvas.height;

  function runFrame(canvas, ctx, width, height){
// Snake overlay — canvas2d filter for GVF
// Main menu + highscore screen + ESC to menu
// Highscores are stored locally (localStorage) — no server
// Sound is synthesized via Web Audio (no external assets)

const COLS = 30, ROWS = 20;
const SZ   = Math.floor(Math.min(width / (COLS + 8), height / ROWS) * 0.95);
const OX   = Math.floor((width  - SZ * (COLS + 7)) / 2);
const OY   = Math.floor((height - SZ * ROWS)       / 2);

const HS_KEY     = 'gvf_snake_hs';
const CHARS      = 'ABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789';
const MENU_ITEMS = ['GAME START', 'HIGHSCORE'];

function loadHS() {
    try { return JSON.parse(localStorage.getItem(HS_KEY)) || []; }
    catch (_) { return []; }
}
function saveHS(list) {
    try { localStorage.setItem(HS_KEY, JSON.stringify(list)); } catch (_){}
}

function fetchHS() {
    return loadHS();
}

function postHS(name, score) {
    const l = loadHS();
    l.push({
        name: String(name || '').toUpperCase().slice(0, 6).padEnd(6, ' '),
        score: score | 0
    });
    l.sort((a, b) => b.score - a.score);
    l.splice(5);
    saveHS(l);
    return l;
}

function rndFood(snake) {
    let pos;
    do {
        pos = {
            x: Math.floor(Math.random() * COLS),
            y: Math.floor(Math.random() * ROWS)
        };
    } while (snake.some(s => s.x === pos.x && s.y === pos.y));
    return pos;
}

function newGameState(resets) {
    const snake = [{ x: Math.floor(COLS / 2), y: Math.floor(ROWS / 2) }];
    return {
        snake,
        dir: { x: 1, y: 0 },
        nextDir: { x: 1, y: 0 },
        food: rndFood(snake),
        score: 0,
        resets: resets || 0,
        lastMove: Date.now(),
        interval: 120,
        dead: false
    };
}

function initState(prev) {
    return {
        mode: 'menu', // menu | play | name | highscore
        menuIndex: 0,

        game: newGameState((prev && prev.game && prev.game.resets) || 0),

        nameChars: [0, 0, 0, 0, 0, 0],
        nameCursor: 0,

        hsList: loadHS(),
        hsLoaded: false,
        hsLoading: false,
        hsMessage: '',
        hsFlashTime: 0
    };
}

function startGame(S) {
    const resets = (S && S.game ? S.game.resets : 0);
    S.game = newGameState(resets);
    S.mode = 'play';
}

function backToMenu(S) {
    S.mode = 'menu';
    S.menuIndex = 0;
}

async function ensureHSLoaded(S) {
    if (S.hsLoading || S.hsLoaded) return;
    S.hsLoading = true;
    S.hsMessage = 'LOADING...';
    try {
        S.hsList = await fetchHS();
        S.hsLoaded = true;
        S.hsMessage = '';
    } catch (_) {
        S.hsMessage = 'LOAD FAILED';
    }
    S.hsLoading = false;
}

async function submitScore(S) {
    const name = S.nameChars.map(i => CHARS[i]).join('');
    S.hsMessage = 'SAVING...';
    S.hsList = await postHS(name, S.game.score);
    S.hsLoaded = true;
    S.hsMessage = 'SAVED';
    S.hsFlashTime = Date.now();
    S.mode = 'highscore';
}

function openHighscore(S) {
    S.mode = 'highscore';
    ensureHSLoaded(S);
}

// ── Init once ────────────────────────────────────────────────────────────────
if (!canvas._gvfSnake) {
    canvas._gvfSnake = initState(null);
    ensureHSLoaded(canvas._gvfSnake);
}
const S = canvas._gvfSnake;

// ── Sounds ───────────────────────────────────────────────────────────────────
if (!canvas._gvfSnakeSounds) {
    const AC = (typeof window !== 'undefined') ? (window.AudioContext || window.webkitAudioContext) : null;
    let actx = null;
    const ensure = () => {
        if (actx) return actx;
        if (!AC) return null;
        try { actx = new AC(); } catch (_) { return null; }
        return actx;
    };
    const tone = (freq, dur, type, vol, slideTo) => {
        const c = ensure();
        if (!c) return;
        const t = c.currentTime;
        const o = c.createOscillator();
        const g = c.createGain();
        o.type = type;
        o.frequency.setValueAtTime(freq, t);
        if (slideTo) o.frequency.exponentialRampToValueAtTime(Math.max(1, slideTo), t + dur);
        g.gain.setValueAtTime(0.0001, t);
        g.gain.linearRampToValueAtTime(vol, t + 0.01);
        g.gain.exponentialRampToValueAtTime(0.0001, t + dur);
        o.connect(g);
        g.connect(c.destination);
        o.start(t);
        o.stop(t + dur + 0.02);
    };
    canvas._gvfSnakeSounds = {
        food: 'food',
        gameOver: 'gameOver',
        play: (key) => {
            const c = ensure();
            if (!c) return;
            if (c.state === 'suspended') c.resume();
            if (key === 'food') {
                tone(660, 0.12, 'square', 0.3, 990);
            } else if (key === 'gameOver') {
                tone(440, 0.7, 'sawtooth', 0.3, 70);
            }
        }
    };
}
const SND = canvas._gvfSnakeSounds;

function playSound(a) {
    try { SND.play(a); } catch (_) {}
}

// ── Keyboard ─────────────────────────────────────────────────────────────────
if (canvas._gvfSnakeHandler) {
    document.removeEventListener('keydown', canvas._gvfSnakeHandler, { capture: true });
    canvas._gvfSnakeHandler = null;
}

{
    const _handler = function (e) {
        if (!canvas.isConnected) {
            document.removeEventListener('keydown', _handler, { capture: true });
            canvas._gvfSnakeHandler = null;
            return;
        }

        const S = canvas._gvfSnake;
        if (!S) return;

        // ESC always returns to main menu
        if (e.key === 'Escape') {
            e.preventDefault();
            if (S.mode === 'play') {
                S.game.resets++;
            }
            backToMenu(S);
            return;
        }

        // MENU
        if (S.mode === 'menu') {
            if (e.key === 'ArrowUp' || e.key === 'w' || e.key === 'W') {
                e.preventDefault();
                S.menuIndex = (S.menuIndex - 1 + MENU_ITEMS.length) % MENU_ITEMS.length;
                return;
            }
            if (e.key === 'ArrowDown' || e.key === 's' || e.key === 'S') {
                e.preventDefault();
                S.menuIndex = (S.menuIndex + 1) % MENU_ITEMS.length;
                return;
            }
            if (e.key === 'Enter' || e.key === ' ') {
                e.preventDefault();
                if (S.menuIndex === 0) startGame(S);
                else if (S.menuIndex === 1) openHighscore(S);
                return;
            }
            return;
        }

        // NAME ENTRY
        if (S.mode === 'name') {
            e.preventDefault();

            if (e.key === 'ArrowUp') {
                S.nameChars[S.nameCursor] = (S.nameChars[S.nameCursor] + 1) % CHARS.length;
                return;
            }
            if (e.key === 'ArrowDown') {
                S.nameChars[S.nameCursor] = (S.nameChars[S.nameCursor] - 1 + CHARS.length) % CHARS.length;
                return;
            }
            if (e.key === 'ArrowRight') {
                S.nameCursor = Math.min(5, S.nameCursor + 1);
                return;
            }
            if (e.key === 'ArrowLeft') {
                S.nameCursor = Math.max(0, S.nameCursor - 1);
                return;
            }
            if (e.key === 'Enter' || e.key === ' ') {
                if (S.nameCursor < 5) {
                    S.nameCursor++;
                } else {
                    submitScore(S);
                }
                return;
            }
            if (e.key.length === 1 && /[a-zA-Z0-9]/.test(e.key)) {
                const idx = CHARS.indexOf(e.key.toUpperCase());
                if (idx >= 0) {
                    S.nameChars[S.nameCursor] = idx;
                    if (S.nameCursor < 5) S.nameCursor++;
                }
                return;
            }
            return;
        }

        // HIGHSCORE SCREEN
        if (S.mode === 'highscore') {
            if (e.key === 'Enter' || e.key === ' ') {
                e.preventDefault();
                backToMenu(S);
                return;
            }
            return;
        }

        // PLAY
        if (S.mode === 'play') {
            if ((e.key === 'ArrowUp' || e.key === 'w' || e.key === 'W') && S.game.dir.y !== 1) {
                e.preventDefault();
                S.game.nextDir = { x: 0, y: -1 };
                return;
            }
            if ((e.key === 'ArrowDown' || e.key === 's' || e.key === 'S') && S.game.dir.y !== -1) {
                e.preventDefault();
                S.game.nextDir = { x: 0, y: 1 };
                return;
            }
            if ((e.key === 'ArrowLeft' || e.key === 'a' || e.key === 'A') && S.game.dir.x !== 1) {
                e.preventDefault();
                S.game.nextDir = { x: -1, y: 0 };
                return;
            }
            if ((e.key === 'ArrowRight' || e.key === 'd' || e.key === 'D') && S.game.dir.x !== -1) {
                e.preventDefault();
                S.game.nextDir = { x: 1, y: 0 };
                return;
            }
        }
    };

    document.addEventListener('keydown', _handler, { capture: true });
    canvas._gvfSnakeHandler = _handler;
}

// ── Game tick ────────────────────────────────────────────────────────────────
const _now = Date.now();

if (S.mode === 'play' && _now - S.game.lastMove >= S.game.interval) {
    S.game.lastMove = _now;
    S.game.dir = S.game.nextDir;

    const head = {
        x: S.game.snake[0].x + S.game.dir.x,
        y: S.game.snake[0].y + S.game.dir.y
    };

    if (
        head.x < 0 || head.x >= COLS ||
        head.y < 0 || head.y >= ROWS ||
        S.game.snake.some(s => s.x === head.x && s.y === head.y)
    ) {
        playSound(SND.gameOver);
        S.mode = 'name';
        S.nameChars = [0, 0, 0, 0, 0, 0];
        S.nameCursor = 0;
    } else {
        S.game.snake.unshift(head);

        if (head.x === S.game.food.x && head.y === S.game.food.y) {
            playSound(SND.food);
            S.game.score += 10;
            S.game.food = rndFood(S.game.snake);
            S.game.interval = Math.max(60, 120 - Math.floor(S.game.score / 50) * 5);
        } else {
            S.game.snake.pop();
        }
    }
}

// ── Nokia 3210 Farben ─────────────────────────────────────────────────────────
// Display: grünliches LCD, dunkle Pixel für Objekte
const NK_BG      = '#8BAC0F'; // helles LCD-Grün (Hintergrund)
const NK_DARK    = '#0F380F'; // dunkelgrün (Pixel an)
const NK_MID     = '#306230'; // mittelgrün (Schatten/Gitter)
const NK_LIGHT   = '#9BBC0F'; // hellstes Grün (Highlight)

const PX = OX + SZ * COLS + 10;
const fs = Math.max(10, Math.round(SZ * 0.9));

// Ganzes Canvas mit LCD-Farbe füllen
ctx.imageSmoothingEnabled = false;
ctx.fillStyle = NK_BG;
ctx.fillRect(0, 0, width, height);

function nokiaText(t, x, y, size, align) {
    ctx.fillStyle = NK_DARK;
    ctx.font = `900 ${size}px monospace`;
    ctx.textAlign = align || 'left';
    ctx.fillText(t, x, y);
    ctx.textAlign = 'left';
}

function nokiaTextMuted(t, x, y, size, align) {
    ctx.fillStyle = NK_MID;
    ctx.font = `${size}px monospace`;
    ctx.textAlign = align || 'left';
    ctx.fillText(t, x, y);
    ctx.textAlign = 'left';
}

function drawBoardAndSide() {
    // Spielfeld-Rahmen
    ctx.strokeStyle = NK_DARK;
    ctx.lineWidth = 2;
    ctx.strokeRect(OX - 2, OY - 2, SZ * COLS + 4, SZ * ROWS + 4);

    // Gitter (sehr dezent)
    ctx.strokeStyle = NK_MID;
    ctx.lineWidth = 0.5;
    for (let r = 0; r <= ROWS; r++) {
        ctx.beginPath();
        ctx.moveTo(OX, OY + r * SZ);
        ctx.lineTo(OX + SZ * COLS, OY + r * SZ);
        ctx.stroke();
    }
    for (let c = 0; c <= COLS; c++) {
        ctx.beginPath();
        ctx.moveTo(OX + c * SZ, OY);
        ctx.lineTo(OX + c * SZ, OY + SZ * ROWS);
        ctx.stroke();
    }

    // Side panel Rahmen
    ctx.strokeStyle = NK_DARK;
    ctx.lineWidth = 2;
    ctx.strokeRect(PX - 2, OY - 2, SZ * 6 + 4, SZ * ROWS + 4);
}

function drawGame() {
    drawBoardAndSide();

    // Snake — ausgefüllte dunkle Blöcke, pixelig
    S.game.snake.forEach((seg, i) => {
        ctx.fillStyle = i === 0 ? NK_DARK : NK_MID;
        ctx.fillRect(OX + seg.x * SZ, OY + seg.y * SZ, SZ, SZ);
        // innerer heller Punkt für Kopf
        if (i === 0) {
            ctx.fillStyle = NK_LIGHT;
            ctx.fillRect(
                OX + seg.x * SZ + Math.floor(SZ * 0.25),
                OY + seg.y * SZ + Math.floor(SZ * 0.25),
                Math.max(1, Math.floor(SZ * 0.2)),
                Math.max(1, Math.floor(SZ * 0.2))
            );
        }
    });

    // Food — kleines ausgefülltes Quadrat (Nokia-Style: kleines X)
    const fx = OX + S.game.food.x * SZ;
    const fy = OY + S.game.food.y * SZ;
    const fp = Math.max(1, Math.floor(SZ * 0.2));
    ctx.fillStyle = NK_DARK;
    ctx.fillRect(fx + fp, fy + fp, SZ - fp * 2, SZ - fp * 2);
    ctx.fillStyle = NK_BG;
    ctx.fillRect(fx + fp * 2, fy + fp * 2, SZ - fp * 4, SZ - fp * 4);

    // Side panel Inhalt
    const sx = PX + 4;
    let sy = OY + fs;
    nokiaTextMuted('SCORE', sx, sy, fs - 3);
    sy += fs * 1.2;
    nokiaText(S.game.score, sx, sy, fs);
    sy += fs * 1.6;
    nokiaTextMuted('LEN', sx, sy, fs - 3);
    sy += fs * 1.2;
    nokiaText(S.game.snake.length, sx, sy, fs);
    sy += fs * 1.6;
    nokiaTextMuted('RST', sx, sy, fs - 3);
    sy += fs * 1.2;
    nokiaText(S.game.resets, sx, sy, fs);
    sy += fs * 1.6;
    nokiaTextMuted('SPD', sx, sy, fs - 3);
    sy += fs * 1.2;
    nokiaText(Math.round((120 - S.game.interval) / 60 * 100) + '%', sx, sy, fs);

    // Highscores unten
    const hsMax = 3;
    const hsList = (S.hsList || []).slice(0, hsMax);
    const lineH = fs + 2;
    const hsBottom = OY + SZ * ROWS - 4;
    const hsTop = hsBottom - hsList.length * lineH - fs;
    nokiaTextMuted('BEST', sx, hsTop, fs - 3);
    hsList.forEach((e, i) => {
        nokiaText(`${e.name}`, sx, hsTop + fs + i * lineH, fs - 3);
        nokiaText(`${e.score}`, PX + SZ * 6 - 4, hsTop + fs + i * lineH, fs - 3, 'right');
    });
}

function drawMenu() {
    drawBoardAndSide();

    const cx = OX + SZ * COLS / 2;
    const cy = OY + SZ * ROWS / 2;
    const bfs = Math.max(14, Math.round(SZ * 1.6));

    nokiaText('SNAKE', cx, cy - bfs * 2.8, bfs * 1.3, 'center');
    nokiaTextMuted('- - - - - - - -', cx, cy - bfs * 1.6, bfs * 0.5, 'center');

    for (let i = 0; i < MENU_ITEMS.length; i++) {
        const y = cy - bfs * 0.3 + i * (bfs * 1.2);
        const active = i === S.menuIndex;

        if (active) {
            ctx.fillStyle = NK_DARK;
            ctx.fillRect(OX + SZ * 2, y - bfs * 0.85, SZ * (COLS - 4), bfs * 1.0);
            ctx.fillStyle = NK_BG;
            ctx.font = `900 ${bfs * 0.7}px monospace`;
            ctx.textAlign = 'center';
            ctx.fillText('> ' + MENU_ITEMS[i] + ' <', cx, y);
            ctx.textAlign = 'left';
        } else {
            nokiaTextMuted(MENU_ITEMS[i], cx, y, bfs * 0.7, 'center');
        }
    }

    nokiaTextMuted('W/S  ENTER', cx, OY + SZ * ROWS - 6, fs - 3, 'center');
}

function drawNameEntry() {
    // Hintergrund mit Spielfeld
    drawBoardAndSide();

    const cx = OX + SZ * COLS / 2;
    const cy = OY + SZ * ROWS / 2;
    const bfs = Math.max(12, Math.round(SZ * 1.4));

    nokiaText('GAME OVER', cx, OY + bfs * 1.4, bfs, 'center');

    nokiaTextMuted(`SCORE: ${S.game.score}`, cx, OY + bfs * 2.6, bfs * 0.65, 'center');
    nokiaTextMuted('ENTER NAME:', cx, cy - bfs * 0.6, bfs * 0.65, 'center');

    const boxW = bfs * 1.1;
    const gap = Math.max(2, Math.floor(SZ * 0.3));
    const totalW = 6 * boxW + 5 * gap;
    const bx0 = cx - totalW / 2;

    for (let i = 0; i < 6; i++) {
        const bx = bx0 + i * (boxW + gap);
        const active = i === S.nameCursor;

        if (active) {
            ctx.fillStyle = NK_DARK;
            ctx.fillRect(bx, cy, boxW, bfs * 1.2);
            ctx.fillStyle = NK_BG;
        } else {
            ctx.strokeStyle = NK_DARK;
            ctx.lineWidth = 1.5;
            ctx.strokeRect(bx, cy, boxW, bfs * 1.2);
            ctx.fillStyle = NK_DARK;
        }

        ctx.font = `900 ${bfs}px monospace`;
        ctx.textAlign = 'center';
        ctx.fillText(CHARS[S.nameChars[i]], bx + boxW / 2, cy + bfs * 1.0);
        ctx.textAlign = 'left';
    }

    nokiaTextMuted('UP/DN  LT/RT  ENTER', cx, OY + SZ * ROWS - 6, Math.max(8, fs - 4), 'center');
}

function drawHighscore() {
    drawBoardAndSide();

    const cx = OX + SZ * COLS / 2;
    const bfs = Math.max(12, Math.round(SZ * 1.4));

    nokiaText('HIGH SCORES', cx, OY + bfs * 1.4, bfs, 'center');

    // Trennlinie
    ctx.fillStyle = NK_DARK;
    ctx.fillRect(OX + SZ * 2, OY + bfs * 1.8, SZ * (COLS - 4), 2);

    if (S.hsLoading) {
        nokiaTextMuted('LOADING...', cx, OY + bfs * 3.5, bfs * 0.75, 'center');
    } else if (!S.hsList || !S.hsList.length) {
        nokiaTextMuted('NO SCORES YET', cx, OY + bfs * 3.5, bfs * 0.75, 'center');
    } else {
        S.hsList.forEach((e, i) => {
            const y = OY + bfs * 2.6 + i * (bfs * 1.1);
            const rank = `${i + 1}.`;
            nokiaText(rank, OX + SZ * 3, y, bfs * 0.8);
            nokiaText(e.name, OX + SZ * 5, y, bfs * 0.8);
            nokiaText(String(e.score), OX + SZ * (COLS - 3), y, bfs * 0.8, 'right');
        });
    }

    if (S.hsMessage) {
        nokiaTextMuted(S.hsMessage, cx, OY + SZ * ROWS - bfs, bfs * 0.5, 'center');
    }

    nokiaTextMuted('ENTER / ESC', cx, OY + SZ * ROWS - 6, Math.max(8, fs - 3), 'center');
}

// ── Draw ─────────────────────────────────────────────────────────────────────
if (S.mode === 'menu') {
    drawMenu();
} else if (S.mode === 'play') {
    drawGame();
} else if (S.mode === 'name') {
    drawNameEntry();
} else if (S.mode === 'highscore') {
    drawHighscore();
}
  }

  function loop(){
    runFrame(canvas, ctx, canvas.width, canvas.height);
    requestAnimationFrame(loop);
  }
  requestAnimationFrame(loop);
})();
</script>
</body>
</html>

```

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
