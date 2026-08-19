# Space Invaders — Wiederaufbau-Handbuch

Dieses Handbuch beschreibt, wie das Spiel aus der Quelldatei als **eine einzige, self-contained HTML-Datei** aufgebaut wird. Damit kannst du es jederzeit nachbauen.

## Was ist das?

Eine vollständige Space-Invaders-Web-App als **eine einzige HTML-Datei** (`space_invaders.html`):

- HTML + CSS + JavaScript **inline** (eine Datei)
- **Canvas-2D**-Rendering, Pixel-Art-Sprites (prozedural erzeugt, keine Bilddateien)
- **Web-Audio**-Synth: alle Sounds und Musik werden zur Laufzeit synthetisiert — keine Audio-Dateien
- **Keine** externen Libraries, **keine** Internet-Ressourcen, **keine** Bild-/Audio-Assets
- Highscores nur **lokal** (`localStorage`), kein Server
- Ship-Select-Menü, Booster-/Level-Phase, Boss-Phase, Sound-Menü (ESC)

## Dateien

| Datei | Zweck |
|---|---|
| `projects/test.txt` | **Quelle**: JSON (`custom-code-player-project`) mit dem kompletten Spielcode im `code`-Feld |
| `space_invaders.html` | **Ergebnis**: die fertige, spielbare Datei |
| `create.md` | dieses Handbuch |

## Steuerung

| Taste | Aktion |
|---|---|
| `←` `→` oder `A` / `D` | bewegen |
| `↑` oder `W` oder `SPACE` | schießen |
| `ESC` | Sound-Menü (Mute, Musik-/SFX-Lautstärke) |

> Audio startet erst nach der ersten Eingabe (Browser-Autoplay-Regel).

## Der Aufbau (Transformation)

Die Quelle in `test.txt` ist ein **flaches JS-Script** für die „GVF-Player-Harness" (~2500 Zeilen). Es erwartet, dass ihm die Globals `canvas`, `ctx`, `width`, `height` bereitstehen. Der Spielzustand hängt an `canvas._gvfSI`.

Um daraus die self-contained HTML zu machen, werden **4 Schritte** gemacht. Das eigentliche Spiel-Logik-Code bleibt dabei unverändert.

### Schritt 0 — Code extrahieren
Das `code`-Feld aus `test.txt` (JSON) extrahieren → flaches JS-Script.

### Schritt 1 — Remote-Highscores entfernen
Alles, was mit dem Server (`space_invaders_hs.php`) zu tun hat, wird entfernt. Behalten bleibt nur der lokale Cache.

Entfernt:
- `HS_API` (Konstante), `loadHSRemote`, `addHSRemote`, `syncHSFromServer`, `syncAddHS`
- die `hsRemoteTried`-Flag und alle `syncHSFromServer()`-/`syncAddHS()`-Aufrufstellen

Behalten:
- `sanitizeHSList`, `loadHSLocal` / `saveHSLocal` (localStorage-Key `gvf_si_hs_cache`)
- `loadHS` / `saveHS` / `addHS`

### Schritt 2 — Audio → Web-Audio-Synth
Die Originalquelle spielte 5 **ferne Audio-URLs** ab (`SND_URLS` mit echten URLs). Das wird durch einen reinen Web-Audio-Synth ersetzt:

1. `SND_URLS` wird eine **Symbol-Key-Map** (die Call-Sites `playOne(SND_URLS.X, 0.12)` bleiben unverändert):
```js
const SND_URLS = {
    shoot: 'shoot',
    invaderKilled: 'invaderKilled',
    swarmMove: 'swarmMove',
    bg: 'bg',
    bossBg: 'bossBg'
};
```
2. `createSoundSystem(initialSettings)` wird eine **Web-Audio-Implementierung** (siehe unten). Das Interface wird exakt repliziert, da es als `S.audio.*` genutzt wird:
   - Props: `unlocked`, `pendingLoop` (`'bg'`/`'boss'`), `muted`, `musicVolume`, `sfxVolume`
   - Methoden: `save()`, `applyVolumes()`, `stopAllLoops()`, `resetForNewRun()`, `playOne(url, baseVolume)`, `ensureUnlocked()`, `applyLoop(kind)`, `setMuted(v)`, `toggleMuted()`, `adjustMusic(delta)`, `adjustSfx(delta)`

### Schritt 3 — In `runFrame` + Harness-Wrapper wickeln
Das Script wird in eine Funktion `runFrame(canvas, ctx, width, height)` gewrappt und pro Frame aufgerufen. Die fertige HTML-Struktur:

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
    // … hier steht das (transformierte) Spiel-Script …
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

`width`/`height` = Canvas-Pixelmaße (= Fenstergröße). Der Canvas wird auf `window.innerWidth` × `window.innerHeight` gesetzt.

### Schritt 4 — Layout-Fix (kein Clipping oben/unten)
Das Spielfeld hat Seitenverhältnis `GH = GW*1.35` und wird zentriert. Damit es bei breitem Fenster nicht über die Fensterhöhe hinausragt (Clipping oben/unten), wird `GW` zusätzlich auf `height/1.35` begrenzt:

```js
// FALSCH (überläuft vertikal):
const GW = Math.min(width*0.65, height*0.85);
// RICHTIG (passt exakt in die Höhe):
const GW = Math.min(width*0.65, height/1.35);
const GH = GW*1.35;
const GX = Math.floor((width-GW)/2);
const GY = Math.floor((height-GH)/2);
```

## Web-Audio-Sound-System (Schritt 2, Kern)

Vollständig synthetisiert, keine externen Assets. Lazy `AudioContext` auf User-Geste, Gain-Ketten `master` → `musicGain` / `sfxGain`, One-Shot-SFX via Oszillatoren/Noise, Looping-Musik via Lookahead-Scheduler.

```js
function createSoundSystem(initialSettings){
    const AC = (typeof window!=='undefined') ? (window.AudioContext||window.webkitAudioContext) : null;
    let actx=null, master=null, musicGain=null, sfxGain=null;
    let musicTimer=null, stepIndex=0, nextNoteTime=0, currentKind=null;

    const midi=function(m){ return 440*Math.pow(2,(m-69)/12); };

    // Musik-Pattern: 8tel-Schritte. b=Bass(midi), a=Arp(midi), k=Kick, h=Hat
    const BG_PATTERN=[
        {b:45,a:57},{a:60},{a:64},{a:69},{b:45,a:67},{a:64},{a:60},{a:67},
        {b:43,a:55},{a:57},{a:60},{a:64},{b:43,a:62},{a:60},{a:57},{a:62}
    ];
    const BOSS_PATTERN=[
        {b:45,k:1},{b:45},{b:45,h:1},{b:45},{b:45,k:1},{b:48},{b:45,h:1},{b:45},
        {b:43,k:1},{b:43},{b:43,h:1},{b:45},{b:45,k:1},{b:48},{b:43,h:1},{b:43}
    ];

    function ensureCtx(){
        if(actx) return true;
        if(!AC) return false;
        try{
            actx=new AC();
            master=actx.createGain(); master.gain.value=1; master.connect(actx.destination);
            musicGain=actx.createGain(); musicGain.connect(master);
            sfxGain=actx.createGain(); sfxGain.connect(master);
            return true;
        }catch(_){ actx=null; return false; }
    }
    function stopMusic(){
        if(musicTimer){ clearInterval(musicTimer); musicTimer=null; }
        currentKind=null;
    }
    function tone(freq,t,dur,type,dest,vol){
        if(!actx) return;
        const o=actx.createOscillator(), g=actx.createGain();
        o.type=type; o.frequency.setValueAtTime(freq,t);
        g.gain.setValueAtTime(0,t);
        g.gain.linearRampToValueAtTime(vol,t+0.012);
        g.gain.setValueAtTime(vol,t+dur*0.7);
        g.gain.linearRampToValueAtTime(0,t+dur);
        o.connect(g); g.connect(dest);
        o.start(t); o.stop(t+dur+0.02);
    }
    function scheduleStep(step,t,stepDur,kind){
        if(!actx) return;
        if(step.b){
            if(kind==='boss') tone(midi(step.b),t,stepDur*1.8,'sawtooth',musicGain,0.8);
            else tone(midi(step.b),t,stepDur*3.5,'triangle',musicGain,0.7);
        }
        if(step.a){
            tone(midi(step.a),t,stepDur*0.95,(kind==='boss'?'square':'sine'),musicGain,kind==='boss'?0.5:0.6);
        }
        if(step.k){
            const o=actx.createOscillator(), g=actx.createGain();
            o.type='sine'; o.frequency.setValueAtTime(120,t); o.frequency.exponentialRampToValueAtTime(45,t+0.12);
            g.gain.setValueAtTime(0.7,t); g.gain.exponentialRampToValueAtTime(0.001,t+0.18);
            o.connect(g); g.connect(musicGain); o.start(t); o.stop(t+0.2);
        }
        if(step.h){
            const buf=actx.createBuffer(1,Math.max(1,actx.sampleRate*0.05)|0,actx.sampleRate);
            const d=buf.getChannelData(0);
            for(let i=0;i<d.length;i++) d[i]=(Math.random()*2-1)*Math.pow(1-i/d.length,2);
            const s=actx.createBufferSource(); s.buffer=buf;
            const f=actx.createBiquadFilter(); f.type='highpass'; f.frequency.value=6000;
            const g=actx.createGain(); g.gain.setValueAtTime(0.4,t);
            s.connect(f); f.connect(g); g.connect(musicGain); s.start(t);
        }
    }
    function startMusic(kind){
        stopMusic();
        if(!ensureCtx()) return;
        const bpm=kind==='boss'?150:108;
        const stepDur=60/bpm/2;
        const pattern=kind==='boss'?BOSS_PATTERN:BG_PATTERN;
        stepIndex=0;
        nextNoteTime=actx.currentTime+0.06;
        currentKind=kind;
        musicTimer=setInterval(function(){
            if(!actx) return;
            while(nextNoteTime<actx.currentTime+0.18){
                scheduleStep(pattern[stepIndex%pattern.length],nextNoteTime,stepDur,kind);
                nextNoteTime+=stepDur;
                stepIndex++;
            }
        },25);
    }
    function playSfx(key,baseVolume){
        if(!ensureCtx()) return;
        const t=actx.currentTime;
        const vol=clamp(baseVolume/0.12,0,1);
        if(key==='shoot'){
            const o=actx.createOscillator(), g=actx.createGain();
            o.type='square'; o.frequency.setValueAtTime(880,t); o.frequency.exponentialRampToValueAtTime(320,t+0.09);
            g.gain.setValueAtTime(0.0001,t); g.gain.linearRampToValueAtTime(vol*0.6,t+0.008); g.gain.exponentialRampToValueAtTime(0.0001,t+0.12);
            o.connect(g); g.connect(sfxGain); o.start(t); o.stop(t+0.14);
        } else if(key==='invaderKilled'){
            const dur=0.45;
            const buf=actx.createBuffer(1,Math.max(1,actx.sampleRate*dur)|0,actx.sampleRate);
            const d=buf.getChannelData(0);
            for(let i=0;i<d.length;i++) d[i]=(Math.random()*2-1)*Math.pow(1-i/d.length,1.5);
            const s=actx.createBufferSource(); s.buffer=buf;
            const f=actx.createBiquadFilter(); f.type='lowpass'; f.frequency.setValueAtTime(2500,t); f.frequency.exponentialRampToValueAtTime(200,t+dur);
            const g=actx.createGain(); g.gain.setValueAtTime(vol*0.8,t); g.gain.exponentialRampToValueAtTime(0.0001,t+dur);
            s.connect(f); f.connect(g); g.connect(sfxGain); s.start(t);
            const o=actx.createOscillator(), g2=actx.createGain();
            o.type='sine'; o.frequency.setValueAtTime(160,t); o.frequency.exponentialRampToValueAtTime(40,t+0.3);
            g2.gain.setValueAtTime(vol*0.6,t); g2.gain.exponentialRampToValueAtTime(0.0001,t+0.35);
            o.connect(g2); g2.connect(sfxGain); o.start(t); o.stop(t+0.4);
        } else if(key==='swarmMove'){
            const o=actx.createOscillator(), g=actx.createGain();
            const lfo=actx.createOscillator(), lfoG=actx.createGain();
            o.type='sine'; o.frequency.setValueAtTime(1100,t);
            lfo.type='sine'; lfo.frequency.setValueAtTime(9,t); lfoG.gain.setValueAtTime(180,t);
            lfo.connect(lfoG); lfoG.connect(o.frequency);
            g.gain.setValueAtTime(0.0001,t); g.gain.linearRampToValueAtTime(vol*0.5,t+0.02); g.gain.exponentialRampToValueAtTime(0.0001,t+0.35);
            o.connect(g); g.connect(sfxGain);
            lfo.start(t); o.start(t); o.stop(t+0.4); lfo.stop(t+0.4);
        }
    }

    const sys={
        unlocked:false,
        pendingLoop:'bg',
        muted: !!initialSettings.muted,
        musicVolume: clamp(initialSettings.musicVolume,0,1),
        sfxVolume: clamp(initialSettings.sfxVolume,0,1),
        save:function(){
            saveAudioSettings({muted:this.muted,musicVolume:this.musicVolume,sfxVolume:this.sfxVolume});
        },
        applyVolumes:function(){
            if(!actx) return;
            const t=actx.currentTime;
            musicGain.gain.setValueAtTime(this.muted?0:this.musicVolume,t);
            sfxGain.gain.setValueAtTime(this.muted?0:this.sfxVolume,t);
        },
        stopAllLoops:function(resetTime){ stopMusic(); },
        resetForNewRun:function(){
            stopMusic();
            this.pendingLoop='bg';
            this.applyVolumes();
            if(this.unlocked && !this.muted){ startMusic('bg'); }
        },
        playOne:function(url,baseVolume){
            if(!this.unlocked || this.muted) return;
            playSfx(url,baseVolume);
        },
        ensureUnlocked:function(){
            if(this.unlocked) return;
            this.unlocked=true;
            ensureCtx();
            this.applyVolumes();
            this.applyLoop(this.pendingLoop||'bg');
        },
        applyLoop:function(kind){
            this.pendingLoop=kind;
            this.applyVolumes();
            if(!this.unlocked) return;
            if(!ensureCtx()) return;
            if(currentKind===kind) return;
            startMusic(kind);
        },
        setMuted:function(v){ this.muted=!!v; this.applyVolumes(); this.save(); },
        toggleMuted:function(){ this.setMuted(!this.muted); },
        adjustMusic:function(delta){
            this.musicVolume=clamp(Math.round((this.musicVolume+delta)*100)/100,0,1);
            this.applyVolumes(); this.save();
        },
        adjustSfx:function(delta){
            this.sfxVolume=clamp(Math.round((this.sfxVolume+delta)*100)/100,0,1);
            this.save();
        }
    };
    sys.applyVolumes();
    return sys;
}
```

> `clamp` und `saveAudioSettings`/`loadAudioSettings` (localStorage-Key `gvf_si_audio_settings`) kommen aus dem Spiel-Script bzw. werden dort bereitgestellt.

## Wieder aufbauen (Checkliste)

1. **Quelle prüfen**: `projects/test.txt` vorhanden (JSON, `code`-Feld mit dem Spiel-Script).
2. **Code extrahieren**: `code`-Feld → flaches JS.
3. **Remote-Highscores streifen** (Schritt 1): `HS_API`, `loadHSRemote`, `addHSRemote`, `syncHSFromServer`, `syncAddHS` + alle Aufrufstellen entfernen; nur lokaler Cache bleibt.
4. **Audio ersetzen** (Schritt 2): `SND_URLS` → Symbol-Key-Map; `createSoundSystem` → Web-Audio (oben).
5. **Wrapper + Layout** (Schritt 3 + 4): in `runFrame` + rAF-Loop wickeln; `GW = Math.min(width*0.65, height/1.35)`.
6. **HTML zusammenstellen**: Kopf (CSS) + `<canvas id="game">` + IIFE-Script (siehe Struktur oben).
7. **Verifizieren**: `node --check` auf dem extrahierten JS → `SYNTAX_OK`; Datei im Projektordner (`C:/Users/Stavr/Desktop/player`) speichern, nicht in `/tmp`.

## Hinweise
- Alles läuft im Projektordner `C:/Users/Stavr/Desktop/player` — keine Tests/Dateien in `/tmp` oder `C:/tmp`.
- Das Spiel-Logik-Code (Level-Konfiguration, Sprites, Kollisionen, Phasen) stammt unverändert aus `test.txt`; nur die obigen 4 Punkte sind die Anpassungen.
- Der fertige Stand ist `space_invaders.html` — dieses Handbuch beschreibt, wie er entstanden ist und wie er neu erzeugt wird.
