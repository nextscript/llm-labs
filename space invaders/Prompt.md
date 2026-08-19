# Prompt — Build a self-contained Space Invaders game

> Hand this prompt to an agent (or follow it yourself) to rebuild the game from scratch.
> Source game logic lives in `projects/test.txt` (JSON, `code` field). The finished file is `space_invaders.html`.

## Goal

Build a complete **Space Invaders** game that runs from **one single HTML file**. The file must be fully self-contained and work offline with no build step.

## Hard constraints

- **One HTML file.** All HTML, CSS, and JavaScript inline. No separate files.
- **No external libraries** — no CDNs, no npm packages, no frameworks.
- **No internet resources** — no remote scripts, styles, fonts, images, or audio.
- **No image or audio assets.** All sprites are drawn procedurally on a 2D canvas; all sound is synthesized at runtime with the Web Audio API.
- **No server.** Highscores are stored locally in `localStorage` only.

## Tech

- HTML5 + CSS + vanilla JavaScript.
- Rendering: **Canvas 2D** (`<canvas id="game">`), pixel-art sprites generated in code.
- Sound: **Web Audio API**, fully synthesized.
- The canvas is sized to the window (`window.innerWidth` × `window.innerHeight`); layout is responsive.

## Game features

- Ship-select menu, booster/level phase, boss phase, and a sound menu.
- Levels, score, lives, power-ups, shields, enemy swarms with formation types.
- Local highscore list (top 5), persisted in `localStorage` (key `gvf_si_hs_cache`).
- Audio settings (mute, music volume, SFX volume) persisted in `localStorage` (key `gvf_si_audio_settings`).

## Controls

| Key | Action |
|---|---|
| `←` `→` or `A` / `D` | move |
| `↑` or `W` or `SPACE` | shoot |
| `ESC` | sound menu (mute, music/SFX volume) |

Audio unlocks on the first user gesture (browser autoplay rule).

## Sound design (Web Audio synth)

Fully synthesized, no assets. Lazy `AudioContext` created on user gesture; gain chain `master` → `musicGain` / `sfxGain`.

- **One-shot SFX** (oscillators + noise):
  - `shoot` — laser blip (square, 880→320 Hz).
  - `invaderKilled` — explosion (filtered noise burst + low sine drop).
  - `swarmMove` — UFO warble (sine with LFO on frequency).
- **Looping music** via a lookahead scheduler (`setInterval`):
  - `bg` — 108 BPM, triangle bass + sine arpeggio.
  - `bossBg` — 150 BPM, sawtooth bass + square arps + kick/hat.

Represent sound as a symbolic-key map so call sites read `playOne(SND_URLS.shoot, 0.12)`:

```js
const SND_URLS = { shoot:'shoot', invaderKilled:'invaderKilled', swarmMove:'swarmMove', bg:'bg', bossBg:'bossBg' };
```

The sound system exposes: `unlocked`, `pendingLoop`, `muted`, `musicVolume`, `sfxVolume`, and methods `save()`, `applyVolumes()`, `stopAllLoops()`, `resetForNewRun()`, `playOne(url, baseVolume)`, `ensureUnlocked()`, `applyLoop(kind)`, `setMuted(v)`, `toggleMuted()`, `adjustMusic(delta)`, `adjustSfx(delta)`.

## Layout

The game field has aspect ratio `GH = GW*1.35` and is centered, with a right-side panel. Constrain the field width so it fits the window height (no top/bottom clipping on wide windows):

```js
const GW = Math.min(width*0.65, height/1.35);   // ensures GH = GW*1.35 <= height
const GH = GW*1.35;
const GX = Math.floor((width-GW)/2);
const GY = Math.floor((height-GH)/2);
```

## Harness

Wrap the game script in `runFrame(canvas, ctx, width, height)` and drive it with a `requestAnimationFrame` loop:

```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>Space Invaders</title>
<style>
  html,body{margin:0;padding:0;width:100%;height:100%;background:#000;overflow:hidden;}
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
    // … game script …
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

## Build steps

1. Extract the `code` field from `projects/test.txt` → flat JS (~2500 lines). It expects globals `canvas`, `ctx`, `width`, `height`; state hangs off `canvas._gvfSI`.
2. **Strip remote highscores** — remove `HS_API`, `loadHSRemote`, `addHSRemote`, `syncHSFromServer`, `syncAddHS`, the `hsRemoteTried` flag, and all their call sites. Keep only the local cache (`sanitizeHSList`, `loadHSLocal`/`saveHSLocal`, `loadHS`/`saveHS`/`addHS`).
3. **Replace audio** — turn `SND_URLS` into the symbolic-key map and rewrite `createSoundSystem` as the Web Audio implementation above. Keep the exact interface (used as `S.audio.*`).
4. **Wrap** — put the script in `runFrame` + the rAF loop (harness above).
5. **Fix layout** — apply the `GW` constraint so the field fits the window height.
6. **Assemble** the single HTML file (head/CSS + `<canvas id="game">` + IIFE script).
7. **Verify** — `node --check` on the extracted JS → `SYNTAX_OK`. Save the result in the project folder, not `/tmp`.

## Acceptance

- Opens in a browser with no network requests and no console errors.
- Renders the game field (no top/bottom clipping), runs the rAF loop.
- Input works (move/shoot/sound menu); sound plays after the first gesture.
- Highscores and audio settings persist across reloads via `localStorage`.
