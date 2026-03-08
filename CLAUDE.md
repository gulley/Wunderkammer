# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

Single-file HTML/JS interactive art piece. No build step, no dependencies, no package manager. Open `index.html` directly in a browser to run it.

## Architecture

Everything lives in `index.html`. The code is organized into clearly-labeled sections (marked with `=== ... ===` banners):

1. **ROSTER** — defines all characters. Each entry calls `loadRosterEntry(src, gait)` which loads the PNG and stores three gait functions.
2. **GEOMETRY** — one-point perspective stage box. `topPt(s, t)` maps UV coordinates (s∈[0,1] left→right, t∈[0,1] front→back) to canvas pixels via bilinear interpolation of the trapezoidal top face. All stage positions go through this function.
3. **WOBBLY TRACK GENERATOR** — `makeWobblyTrack()` generates a random closed parametric curve using a base oval plus two Fourier harmonics. After generating, it samples 256 points to find bounds, then normalizes the curve to fill the stage (7% margin). The track object exposes a single `pt(a)` method.
4. **PERFORMANCE STATE** — `perf[]` holds the two currently-active puppets. `pickNewPair()` draws two random roster entries and assigns each a fresh track.
5. **CRANK + PICK BUTTON** — constants for the two interactive controls.
6. **SCENE DRAWING** — draw order matters: box fills → box edges → housings → puppets (z-sorted by y) → slot tracks (drawn *after* puppets so the groove overlaps the stick base, creating the through-stage illusion) → crank arm.
7. **MAIN LOOP** — `draw(ts)` uses `requestAnimationFrame`. `running` is true only while the mouse is held on the crank.
8. **INPUT** — canvas mouse events; hit-tested against two circles (crank and pick button).

## Key design facts

**Gait system.** Each character has three functions of angle `a` (the puppet's current position in radians on its oval):
- `bob(a)` — stick-tip vertical offset in px, negative = upward. Shape of the function controls gait character: `pow(|sin|, <1)` = springy/airborne; `pow(|sin|, >1)` = heavy/stomping; amplitude modulated by a second frequency = limp.
- `waggle(a)` — stick rotation (rad) around the stage hole. Frequency × lap speed = swings per circuit.
- `cardTilt(a)` — independent card rotation (rad) around stick tip. Offset bias = constant lean; oscillation = body sway.

**Track closure guarantee.** The wobbly track uses `cos(n·a)` and `sin(n·a)` terms only. These are all exactly 2π-periodic so the path always closes regardless of harmonic amplitudes.

**Stick/stage intersection illusion.** The stick is drawn first (starting at the stage surface), then the slot track groove is painted over the stick's base. The groove visually consumes the stick at the surface level.

**Card scaling.** `CARD_H = 110` is fixed for all characters. Width is derived from each image's natural aspect ratio on load, so all characters stand the same height.

**Z-sorting.** Each frame `perf` is sorted by the y-coordinate of `track.pt(currentAngle)` before drawing. Lower y = further from viewer = drawn first.

## Adding a character

1. Drop a PNG into `assets/`.
2. Add one `loadRosterEntry('assets/name.png', { bob, waggle, cardTilt })` entry to `ROSTER`. The character automatically becomes eligible for the random cast picker.

## Assets

All PNGs in `assets/` are pencil-sketch stick figures on white/near-white backgrounds. The renderer uses `globalCompositeOperation = 'multiply'` to drop the white backgrounds into the cream card color.
