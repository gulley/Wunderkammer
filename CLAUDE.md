# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Collaboration context

This is a mystery collaboration between two friends of 30+ years: **rmauceri** and **gulley** (GitHub: gulley). They take turns making changes, both vibe-coding with AI. Neither knows exactly what this is becoming — the project is part art, part discovery, part exploration.

**Gulley** built the original wunderkammer. He's a professional software engineer, active writer, and student of mathematics, art, theater, and music. He values originality, the absurd, and humor. He's fascinated by pseudo-science, alchemy, astrology, and the esoteric. His wife tragically passed from cancer a few years ago; they were high school sweethearts.

**Creative principles for this project:**
- Favor the surprising and original over the polished and expected
- Leave creative seams — don't over-engineer or close off directions the other collaborator might pull on
- The tone is playful on the surface, serious craft underneath
- Embrace absurdity and humor; the wunderkammer is a cabinet of curiosities, not a product
- Both collaborators are technically strong — code can be clever and expressive

## Project overview

Single-file HTML/JS interactive art piece. No build step, no dependencies, no package manager. Open `index.html` directly in a browser to run it.

A mechanical puppet theater (Wunderkammer) where pairs of pencil-sketch characters perform procedurally generated three-act plays with Shakespeare dialogue. The audience drives the performance by turning a crank.

## Architecture

Everything lives in `index.html`. The code is organized into clearly-labeled sections (marked with `=== ... ===` banners):

1. **ROSTER** — defines all characters. Each entry calls `loadRosterEntry(src, gait, name, tags)` which loads the PNG and stores three gait functions, a display name, and personality tags.
2. **GEOMETRY** — one-point perspective stage box. `topPt(s, t)` maps UV coordinates (s∈[0,1] left→right, t∈[0,1] front→back) to canvas pixels via bilinear interpolation of the trapezoidal top face. All stage positions go through this function.
3. **WOBBLY TRACK GENERATOR** — `makeWobblyTrack()` generates a random closed parametric curve using a base oval plus two Fourier harmonics. After generating, it samples 256 points to find bounds, then normalizes the curve to fill the stage (7% margin). The track object exposes a single `pt(a)` method.
4. **PERFORMANCE STATE** — `perf[]` holds the two currently-active puppets. `makeNewPair()` draws two random roster entries and assigns each a fresh track.
5. **OPENING SEQUENCE** — state machine for the lid-opening animation (`closed → opening → open → playing`).
6. **CRANK + BUTTONS** — constants for the three interactive controls (crank, pick, scene).
7. **CHARACTER TRANSITION** — slide-out/slide-in animation when picking a new pair.
8. **BACKDROP** — two backdrop images (castle, forest) drawn behind the stage with multiply blending. The scene button cycles between them.
9. **PLAY SYSTEM** — the generative heart of the piece:
   - **CORPUS** — 200 Shakespeare lines tagged by emotional category (`greeting`, `boast`, `challenge`, `devotion`, `insult`, `despair`, `madness`, `prophecy`, `triumph`, `farewell`) and tone (`royal`, `comic`, `dark`, `gentle`, `wild`).
   - **STAGE_DIRS** — 8 opening directions + 24 mid-play directions, selected without repeats.
   - **ARC_TEMPLATES** — 6 three-act story shapes (The Rivalry, The Romance, The Tragedy, The Comedy, The Dream, The Trial). Each defines a sequence of emotional beats with speaker assignments.
   - **TITLE_FORMATS** — 6 title patterns combining character names and arc names.
   - **generateScript()** — selects an arc, fills beats with corpus lines matching category + character tone preference, avoids repeats.
   - **MOOD_PACE** — per-mood pacing multipliers. Conflict/madness beats are fast; farewell/despair linger.
   - **Cue cards** — floating cream rectangles above the cabinet with Shakespeare text, drawn with fade-in/hold/drift-out lifecycle.
   - **Color suffusion** — speaking character's card tints toward a mood color; backdrop base fill shifts before multiply blending.
   - **Emotion glyphs** — small particles (♥, @#$%!, ★, ?, ·) that spawn from the speaking character's card and float upward.
   - **Wind-down** — after Fin, puppets decelerate to a stop like a vinyl record ending.
10. **STARS** — twinkling stars scattered in the dark canvas area surrounding the stage box.
11. **GRAIN TEXTURE** — pre-baked stipple + fiber stroke texture for wood feel.
12. **AUDIO** — kalimba-like plucked tones (D Phrygian) with mood-driven behavior:
    - **MOOD_MUSIC** — per-mood note selection (register, rate, volume, harmonic ratios).
    - **Structural stings** — ascending arpeggio for title, root+fifth for act titles, single high note for directions, descending arpeggio for Fin.
    - **Crank ticks** — filtered noise bursts simulating mechanical clicks.
13. **DRAWING HELPERS** — polygon fill, perspective-correct line drawing, grain overlay.
14. **SCENE DRAWING** — draw order matters: stars → backdrop → stage top → slot tracks → puppets (z-sorted) → lid → front face → buttons → crank → emotion particles → cue cards.
15. **MAIN LOOP** — `draw(ts)` uses `requestAnimationFrame`. `running` is true only while the mouse is held on the crank.
16. **INPUT** — canvas mouse events; hit-tested against three circles (crank, pick button, scene button). Hold Shift while cranking to reverse.

## Key design facts

**Gait system.** Each character has three functions of angle `a` (the puppet's current position in radians on its oval):
- `bob(a)` — stick-tip vertical offset in px, negative = upward. Shape of the function controls gait character: `pow(|sin|, <1)` = springy/airborne; `pow(|sin|, >1)` = heavy/stomping; amplitude modulated by a second frequency = limp.
- `waggle(a)` — stick rotation (rad) around the stage hole. Frequency × lap speed = swings per circuit.
- `cardTilt(a)` — independent card rotation (rad) around stick tip. Offset bias = constant lean; oscillation = body sway.

**Character personality tags.** Each roster entry has a `tags` array (e.g., `['royal', 'gentle']`). The play generator uses the first tag to prefer corpus lines with a matching tone, falling back to any line in the emotional category.

**Play generation pipeline.** `startPlay()` → `generateScript()` picks a random arc template, fills each beat from the corpus matching category + character tone, wraps with a randomized title card and Fin. The script is an array of beat objects with `type`, `speaker`, `text`, `mood`.

**Crank-driven progression.** The play advances only when cranking. `playCrankAccum` accumulates crank radians; when it exceeds `BEAT_BASE * beatPace(idx)`, the next beat fires. Mood-specific pacing: conflict/madness are fast (0.55-0.65×), farewell is slow (1.4×).

**Color suffusion.** The speaking character's card fill interpolates from cream toward a mood-specific color. The backdrop's base fill (before multiply) also shifts. Non-speaking character stays neutral.

**Track closure guarantee.** The wobbly track uses `cos(n·a)` and `sin(n·a)` terms only. These are all exactly 2π-periodic so the path always closes regardless of harmonic amplitudes.

**Stick/stage intersection illusion.** The stick is drawn first (starting at the stage surface), then the slot track groove is painted over the stick's base. The groove visually consumes the stick at the surface level.

**Card scaling.** `CARD_H = 123` is fixed for all characters. Width is derived from each image's natural aspect ratio on load, so all characters stand the same height.

**Z-sorting.** Each frame `perf` is sorted by the y-coordinate of `track.pt(currentAngle)` before drawing. Lower y = further from viewer = drawn first.

## Adding a character

1. Drop a PNG into `assets/`. Pencil sketch on white/near-white background.
2. Add one `loadRosterEntry('assets/name.png', { bob, waggle, cardTilt }, 'Name', ['tag1', 'tag2'])` entry to `ROSTER`.
   - Gait functions take angle `a` in radians.
   - Tags should be from: `royal`, `comic`, `dark`, `gentle`, `wild`.
3. The character is automatically eligible for random casting and play generation.

## Assets

All PNGs in `assets/` are pencil-sketch stick figures on white/near-white backgrounds. The renderer uses `globalCompositeOperation = 'multiply'` to drop the white backgrounds into the cream card color.

Characters: boney, bromley, george, hag, jester, king, piper, princess, sol, spike.
Backdrops: castle, forest.
