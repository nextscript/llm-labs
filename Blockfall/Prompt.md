Create a complete falling-blocks puzzle game in a single self-contained HTML file. No external libraries, no internet resources, no images or audio assets: everything inline (HTML, CSS, JavaScript, canvas rendering, Web Audio sound).

Requirements:

- 10 wide by 20 tall playfield rendered on a canvas.
- Seven distinct piece shapes, each made of four squares, each with its own color. Every piece — including the horizontal bar — must render and lock correctly.
- Pieces are dealt from a 7-bag randomizer for fair distribution.
- Controls: left/right arrows move, up arrow rotates, down arrow soft-drops, spacebar hard-drops, R restarts.
- Rotation must respect walls and stacked blocks (no clipping through anything).
- A ghost piece shows where the current piece will land (semi-transparent outline).
- Completed horizontal lines clear, rows above fall down, and clearing multiple lines at once scores more.
- Score, lines cleared, and level displayed. Speed increases with level.
- Next-piece preview box.
- Game over when the stack reaches the top, with a visible game-over state and a restart key.
- Sound effects generated with the Web Audio API (no audio files): move, rotate, lock, hard drop, line clear, and game over. A mute toggle on the M key.
- Clean, readable dark visual style.

Output only the complete HTML file.
