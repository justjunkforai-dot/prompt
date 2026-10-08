# Prompt: Build a Full Geometry Dash–Style Game

> Copy everything below the line into a new Claude Code session (empty folder) and send it.

---

You are a senior game developer. Build me a **complete, polished, playable Geometry Dash–style rhythm platformer** that runs in the browser. Do not give me a skeleton, a demo, or a "starting point". Deliver a finished game with menus, multiple levels, a level editor, audio, effects, and saved progress. Work autonomously: plan, build, run it, test it, fix what is broken, and only then report back.

Call the game **"Neon Dash"** (original name, original art, original music – do not copy any copyrighted assets, level data, or names from the real Geometry Dash).

---

## 1. Tech stack and project rules

- **Plain HTML + CSS + JavaScript (ES modules), HTML5 Canvas 2D.** No build step, no frameworks, no npm dependencies required to play. It must work by opening `index.html` through any static server (`python3 -m http.server`).
- Audio via the **Web Audio API only**. All music and sound effects are **synthesized in code** (no external audio files, no downloads, no network requests at runtime).
- All graphics are drawn in code (canvas paths/gradients/glow). No external images or fonts (use system fonts + CSS fallback).
- File structure (split into modules, each file under ~500 lines):
  ```
  index.html
  css/style.css
  js/main.js            // bootstrap, game loop, state machine
  js/config.js          // all constants (physics, colors, sizes)
  js/input.js           // keyboard, mouse, touch, gamepad
  js/player.js          // player physics and modes
  js/level.js           // level loading, objects, collision world
  js/objects.js         // object type definitions
  js/camera.js
  js/renderer.js        // drawing, parallax, glow
  js/particles.js
  js/audio.js           // synth engine, SFX, music sequencer
  js/levels/            // built-in level data (JSON-like JS modules)
  js/editor.js          // level editor
  js/ui.js              // menus, HUD, screens
  js/storage.js         // localStorage persistence
  README.md             // how to run, controls, how to add levels
  ```
- Code quality: readable, commented where non-obvious, no global variable soup, no dead code.
- Performance target: **stable 60 FPS** on a mid-range laptop, with a **fixed-timestep physics loop (240 Hz sub-stepping or 120 Hz fixed)** decoupled from render rate so gameplay is identical on 60/120/144 Hz monitors. Use `requestAnimationFrame` with accumulator, clamp big frame gaps (tab switch), render with interpolation.
- HiDPI aware: scale the canvas by `devicePixelRatio`, keep a fixed logical resolution of **1920×1080 (16:9) letterboxed** to any window size.

---

## 2. Core gameplay (must feel like the real genre)

The player is a square-ish icon that **auto-scrolls to the right at a constant speed**. The only control is **one button (jump/click/tap)**. Dying is instant and the level restarts in under ~0.3 seconds. Tight, responsive, fair controls are the #1 priority.

### 2.1 Units and physics
- Define the grid: **1 block = 30 logical units** (document this in `config.js`). Player hitbox is 30×30 (cube). Level is built on this grid.
- Speeds (blocks/second, tunable per level via speed portals): **0.7× slow, 1× normal, 1.3× fast, 1.6× faster, 2× fastest** (base ≈ 10.4 blocks/s at 1×).
- Cube: gravity ≈ constant downward acceleration, jump gives a fixed initial velocity so a jump covers about **~2.2 blocks high max and ~3.8–4 blocks long at 1× speed**. Tune it so that single spikes, double spikes, and 1–2 block gaps are comfortably clearable, and triple spikes need a well-timed jump.
- **Hold to keep jumping:** holding the button on the ground auto-jumps again on landing (like the real game). Releasing and pressing again also works.
- **Input buffering:** a press within ~80 ms before landing still triggers a jump. **Coyote time** ~40 ms (only for orbs/ground fairness; keep it subtle).
- Gravity can flip (gravity portals) and the player is mirrored/rotated appropriately.
- Cube rotation: while airborne the cube rotates smoothly (~ 360° per jump arc), snapping to the nearest 90° on landing with a quick ease.

### 2.2 Game modes (vehicle portals) — implement ALL of these
1. **Cube** – jump on click.
2. **Ship** – hold to fly up, release to fall; smooth tilt toward velocity; ceiling and floor are bounded in ship sections.
3. **Ball** – click to flip gravity while on a surface; rolls along floor/ceiling.
4. **UFO** – each click gives a small hop (mid-air allowed); falls gently.
5. **Wave** – hold to move diagonally up, release for diagonal down at 45°; leaves a glowing trail; dies on touching *any* solid or spike (slopes included).
6. **Robot** – hold to jump higher (variable height), grounded stride animation.
7. **Spider** – click to teleport instantly to the opposite surface (floor↔ceiling), with a short dash animation.
8. **Swing** (optional bonus mode) – click to flip gravity in mid-air with smooth swing physics. Implement if everything else is done.

Each mode has its own tuned physics constants in `config.js`, its own drawn icon (distinct, readable shape), and an animated transition particle burst when entering a portal.

### 2.3 Level objects — implement all of these
**Solid/hazard**
- Blocks (full, half-height, slabs), with proper side/top/bottom collision. Touching the *side* of a solid block while moving right = **death**; landing on top = safe; hitting the bottom while moving up = death (cube) / bounce-stop (ship).
- **Spikes** (up, down, left, right, small and large variants) with *tighter-than-visual* hitboxes (triangle hitbox ~ 35–45% of visual) so it feels fair.
- **Sawblades** (rotating, 3 sizes, some on moving paths), circular hitboxes.
- **Slopes/ramps** (45° up and down, plus 22.5° variants) that launch/slide the player correctly per mode.
- **Fake/decoration blocks** that have no collision.

**Interactive**
- **Jump pads** (yellow – medium launch, pink – small launch, red – huge launch, blue – gravity flip launch).
- **Jump orbs** (yellow – normal jump, pink – small, red – huge, blue – gravity flip, green – flip + jump, black – downward dash, dash orbs – directional dash). Orbs activate on click press while overlapping; each orb can only be used once per overlap.
- **Portals:** gamemode portals (one per mode), gravity up/down, **size mini/normal** (mini = 0.6× hitbox & sprite, different physics scaling), **speed portals** (5 speeds), **mirror portals** (flip horizontal scroll direction visually / flip the screen), **dual-mode portals** (two players at once, mirrored gravity, either dying kills both).
- **Trigger system** (invisible, activated when player x passes them): background color change, ground color change, camera zoom, camera shake, move/rotate/scale object groups, alpha fade groups, toggle groups on/off, pulse (beat-synced color flash), spawn particles. Objects get a `group` id so triggers can target them.
- **Checkpoints** (practice mode only, see below), **secret coins** (3 per level, hidden in risky/hidden spots, saved when you *complete* the level).
- **End wall** that triggers the level-complete sequence.

### 2.4 Collision system (be careful, this is where such games break)
- Use **swept / sub-stepped AABB collision** so the player never tunnels through blocks at 2× speed.
- Spatial partition: uniform grid or chunked buckets so only nearby objects are tested each step. Level of 20,000+ objects must still run at 60 FPS.
- Separate checks for: solid AABBs, slanted surfaces (SAT or line-segment tests), triangle spikes, circle saws, and trigger-zone overlaps (sensors).
- Deterministic: with identical input timings the player must follow identical trajectories every run (no random or frame-rate-dependent physics). Add a small debug test (see §9) proving this.
- A **hitbox debug overlay** toggle (key `H`) draws every hitbox in red/green so I can verify fairness.

---

## 3. Game flow and screens

State machine: `Boot → MainMenu → LevelSelect → Playing ⇄ Paused → LevelComplete / Dead(auto-restart) → …`, plus `Editor`, `Settings`, `Customize`, `Stats/Achievements`, `Credits`.

### 3.1 Main menu
- Animated neon background (parallax scrolling shapes, pulsing to a menu music loop).
- Big animated logo, buttons: **Play, Create (Editor), Customize, Achievements, Settings, Quit/Credits**.
- Shows total stars, coins, and attempts.
- Keyboard + mouse + gamepad navigation, with hover/focus sound and animation.

### 3.2 Level select
- Horizontal carousel of levels with name, difficulty face icon, star count, best % (normal) and best % (practice), coin slots (collected/uncollected), and a progress bar.
- Locked levels shown if the previous isn't beaten (but allow a "Unlock all" checkbox in Settings for convenience).

### 3.3 In-level HUD
- Slim **progress bar** at the top with percent text; attempt counter ("Attempt 37") shown at level start floating in world space; small FPS counter (toggle in settings); pause button.
- Practice-mode indicators and checkpoint diamond buttons (touch) / keys.

### 3.4 Pause menu
- Resume, Restart, Practice Mode toggle, Settings (volume sliders), Exit to Menu. Pausing freezes physics and audio (suspend AudioContext cleanly) and resumes from the exact spot with a **3-2-1 countdown**.

### 3.5 Death sequence
- Cube explodes into ~40 shards/particles + a flash + screen shake + sound; auto-restart after ~0.5 s (time configurable). The music restarts in sync with the level start.

### 3.6 Level complete
- Camera slows, end-wall glow, fireworks/particles, "Level Complete!" banner, stats panel (attempts, time, jumps, coins collected), animated stars/orbs/diamond reward counters, buttons: *Replay, Next Level, Menu*.

### 3.7 Practice mode
- Toggle with a button / key `P`. Player can place a **checkpoint** (key `Z`) and remove the last one (key `X`) or enable **auto-checkpoints** every N seconds. Dying respawns at the latest checkpoint with correct state restored (mode, gravity, speed, size, dual, mirror, camera, active triggers, music position). Practice runs never count for progress/coins/stars but record best practice %.
- Music in practice mode plays a separate, quieter, beat-synced track that seeks to the checkpoint time.

---

## 4. Levels (built-in content)

Create **at least 8 original handcrafted levels** with escalating difficulty, each with its own music track, color scheme, background style, and length (~45–120 seconds):

| # | Name | Difficulty | Theme / New mechanic introduced |
|---|------|-----------|--------------------------------|
| 1 | First Steps | Easy | Cube basics: spikes, blocks, gaps, jump pads |
| 2 | Neon Skyline | Easy | Ship mode, speed portals |
| 3 | Gravity Well | Normal | Gravity portals, ball mode, orbs |
| 4 | Pulse Reactor | Normal | UFO, mini size, pulse triggers |
| 5 | Wave Runner | Hard | Wave mode with tight corridors, slopes |
| 6 | Mirror Maze | Hard | Mirror portals, dual mode, spider |
| 7 | Saw Factory | Harder | Moving sawblades, robot, group move triggers |
| 8 | Final Descent | Insane | All mechanics, 2× speed sections, camera triggers |

Level design requirements:
- Levels must be **beatable and fair**. Every jump/transition has a visible cue. No unavoidable deaths, no blind-guess spots outside of clearly-marked "hard" ones. Add a small automated **"bot" validator** (see §9) that replays a recorded input list for each level to prove it is completable; ship the input recordings with the repo.
- Rhythm design: obstacles are placed on the **beat grid** of that level's music (BPM stored per level, e.g. 128, 140, 150). Jumps land on strong beats. Document the BPM and the beat→x mapping.
- Visual variety per level: different palette, ground style, parallax background layers (stars, city silhouettes, grid, hexagons, gears), and pulsing effects on beat.
- Levels are defined as **data** (`js/levels/level1.js` exporting `{ meta, objects, triggers }`), in a compact format, e.g. `[typeId, x, y, rotation, flipX, flipY, groupId, props]`. Provide a small **level-building helper API** (`block(x,y)`, `spikes(x,y,n)`, `stairs(...)`, `repeat(...)`, `gap(...)`) so authoring levels in code is ergonomic and readable.
- Include a hidden **9th "Bonus/Challenge" level** unlocked by collecting all coins.

---

## 5. Music and sound (all synthesized, no files)

Build an **audio engine** (`audio.js`) in Web Audio:
- Master gain + separate music/SFX buses with volume sliders (0–100%, persisted).
- Synth voices: kick, snare, hat, clap, bass (saw/square + filter envelope), lead (detuned saws + low-pass), pad, arp, plus noise-based risers/impacts.
- A **step sequencer** driven by the AudioContext clock with look-ahead scheduling (no `setInterval` drift): patterns defined as data per level (BPM, scale, chord progression, drum pattern, bass line, lead melody, arp). 8 distinct original tracks (one per level) + menu theme + level-complete jingle + editor ambience.
- The game logic reads music time so **beat-synced effects** (pulse, background throb, ground flash) stay aligned. Handle `AudioContext` autoplay policy (resume on first input).
- SFX: jump, land, orb click, pad launch, portal, death explosion, level complete, coin collect, UI hover/click/back, checkpoint, achievement unlock, countdown ticks, speed-up whoosh.
- The music must restart/seek correctly on death, on checkpoint respawn, on pause/resume, and when the tab is hidden/visible.

---

## 6. Visuals and effects

Make it look like a premium neon game, not a prototype:
- Glow: use `shadowBlur`/layered strokes or an offscreen blurred bloom pass (cached); keep it fast.
- Blocks with beveled neon outlines and subtle inner gradients; spikes with glow; animated saws with rotating teeth; portals with swirling particles and a flare.
- **Parallax**: 3–5 background layers + ground with scrolling tile pattern + ceiling in ship/ball/wave sections.
- **Beat pulse**: background and objects pulse with the kick; color triggers crossfade smoothly.
- **Particles**: jump dust, landing puffs, ship exhaust flame, wave trail (fading ribbon), orb/pad burst, portal swirls, death explosion shards, confetti on completion. Pool particles (no per-frame allocations) with a hard cap (~2,000).
- Camera: smooth follow on Y (deadzone), fixed X offset, zoom triggers, shake, and a slight lookahead; ship/wave sections lock camera to the corridor.
- Player icon is drawn procedurally with customizable **primary color, secondary color, glow color, and 20+ icon shapes** (cube faces, ship hulls, ball, UFO, wave arrow, robot, spider each with ≥ 6 variants), and a trail option.
- Screen transitions: fade/iris wipe between states.
- Respect `prefers-reduced-motion`: reduce shake, flashes, and offer a **"Reduce Flashing"** toggle in Settings.

---

## 7. Level editor (must be really usable)

Full-featured editor reachable from the main menu:
- **Canvas editing:** grid-snapped placement, pan (middle mouse / space+drag / arrow keys), zoom (wheel / +/-), box-select, multi-select (shift), move, rotate (90°), flip H/V, copy/paste, duplicate, delete, undo/redo (≥ 100 steps), snap toggle, layers (background decoration / main / foreground).
- **Palette panel:** tabs for Blocks, Hazards, Pads/Orbs, Portals, Triggers, Decoration; click to pick, hover tooltip with name, keyboard shortcuts for the most-used items.
- **Properties panel** for selected objects: position, rotation, group id, color, z-order, trigger parameters (target group, duration, easing, color, etc.).
- **Playtest** button: play from the start or **from the cursor position** (with correct mode/speed/gravity state computed by scanning earlier portals). Return to the editor at the same view after dying/quitting.
- **Level settings:** name, creator, BPM, starting mode/speed/size, background & ground presets, music track selection (choose from built-in synth tracks), song offset.
- **Save/Load/Export/Import:** levels saved to `localStorage` (multiple slots, list with delete/rename), **export to a downloadable `.json` file** and **import** via file picker / drag-and-drop with validation and friendly error messages. Auto-save every 30 s and a crash-recovery prompt.
- **Validation:** warns when a level has no end wall, no start portal conflicts, > N objects, or invalid triggers.
- Custom user levels appear in a "My Levels" tab in Level Select.

---

## 8. Systems: input, settings, progress

**Input:**
- Jump: `Space`, `ArrowUp`, `W`, left mouse button, touch anywhere. Hold = hold.
- `Esc` pause, `R` restart, `P` practice, `Z/X` checkpoints, `H` hitboxes, `F` FPS toggle, `M` mute, `F11` fullscreen button.
- **Gamepad** support (A / RT / any face button = jump, Start = pause, D-pad/stick = menu nav).
- **Mobile/touch:** responsive layout, large tap target, no 300 ms delay, `touch-action: none`, prevent scroll/zoom, show on-screen pause and practice buttons, handle orientation (suggest landscape).
- Rebindable keys in Settings.

**Settings:** master/music/SFX volume, FPS cap (60/120/Unlimited), vsync-style limiter, show FPS, fullscreen, particle quality (Low/Med/High), glow quality, reduce flashing, screen shake amount, hitbox view, auto-restart delay, key bindings, language placeholder, reset progress (with confirm).

**Progress/Storage (`storage.js`):** versioned save data in `localStorage` with migration and try/catch (works if storage is blocked): per-level best %, attempts, jumps, time played, coins, completion, stars, orbs, unlocked icons/colors, achievements, settings, custom levels. Include **export/import save** as JSON.

**Economy and unlocks:** stars from levels → unlock levels; orbs/diamonds from completions → unlock icons and colors in the Customize screen; secret coins → unlock special icons/trails.

**Achievements (≥ 20):** e.g., "First Jump", "Beat Level 1–8", "No Hit Practice", "Collect 3 Coins", "100 Attempts", "Hold For 60 Seconds", "Create a Level", "Perfect Wave", etc., with toast popups and a gallery screen.

**Accessibility:** colorblind-safe default palette option, large UI scale option, visible focus states, ARIA labels on DOM controls, and no information conveyed by color alone for hazards (hazards have distinct red/white outlines and shape language).

---

## 9. Testing and quality gates (do this before you tell me it's done)

You have a shell and a browser available. Actually run and verify:

1. **Static serve** the project (`python3 -m http.server`) and load it in headless Chromium via Playwright (already installed; use `executablePath: '/opt/pw-browsers/chromium'` if needed, do not run `playwright install`).
2. **No console errors or warnings** on boot, in menus, in each level, and in the editor. Fail the check if any occur.
3. **Automated gameplay tests** (a `tests/` folder runnable with `node` or in the page with `?test=1`):
   - Physics determinism: run the same scripted inputs twice → identical player state hash.
   - Frame-rate independence: simulate at 30, 60, 144, 240 FPS render rates → identical final results.
   - Collision unit tests: no tunneling at 2× speed; spike hitbox; slope behavior; each mode's gravity flip.
   - **Bot completion test:** a recorded input timeline for each built-in level that reaches the end wall. All 8 must pass.
   - Save/load roundtrip for the editor JSON format and for save data.
4. **Screenshots:** take Playwright screenshots of the main menu, level select, a level in cube mode, ship mode, wave mode, the pause menu, the level complete screen, the editor, and the customize screen. **Look at them** and fix visual bugs (overlaps, clipped text, blurry canvas, wrong colors).
5. **Performance check:** measure average frame time over 10 seconds in the heaviest level and report it; fix any hotspots (allocations in the loop, redundant `save/restore`, un-cached gradients).
6. Test at 3 viewport sizes (1920×1080, 1366×768, 390×844 mobile portrait/landscape) and confirm layout is usable.

If any test fails, **fix and re-run** until everything passes. Don't tell me it works unless you ran it.

---

## 10. Working method and deliverables

1. **First**, write a short design doc (`DESIGN.md`): architecture, physics constants with reasoning, level data format, audio engine design, editor architecture. Keep it to ~1–2 pages.
2. **Then build in vertical slices, in this order**, running and testing after each: (a) engine + cube + level 1 + death/restart → (b) audio engine + music for level 1 → (c) remaining modes and objects → (d) menus/HUD/pause/complete/practice → (e) levels 2–8 + bot recordings → (f) editor → (g) customize/achievements/settings/storage → (h) polish, effects, accessibility, performance → (i) test pass and screenshots.
3. Commit to git after each slice with a clear message (conventional-commit style). Don't open a pull request unless I ask.
4. `README.md` must include: how to run, controls, feature list, how to add a level, how to add a track, project structure, known limitations.
5. When finished, give me a **short final report**: what was built, how to run it, test results (pass counts, FPS numbers), screenshots list, and honest known issues. Do not pad it.

---

## 11. Hard constraints and things to avoid

- **No copyrighted material:** no real Geometry Dash level layouts, names, songs, icons, or sprites. Everything original.
- No external network requests at runtime; no analytics, no CDN scripts.
- No placeholder text ("TODO", "lorem ipsum", "coming soon") in the shipped game. If a feature is cut, remove its UI.
- No frame-rate-dependent physics. No `setInterval` for gameplay or audio timing.
- No memory leaks: clean up event listeners and audio nodes when changing states; verify the heap doesn't grow over a 5-minute play session.
- Don't stop to ask me questions about reasonable defaults — decide, note the decision in `DESIGN.md`, and keep going. Only ask me if you are truly blocked.
- If scope forces trade-offs, **protect in this order**: (1) controls feel + collision correctness, (2) levels 1–5 fully polished with music, (3) menus/practice mode, (4) the editor, (5) cosmetics/achievements. Never ship something broken in a higher-priority item to add a lower-priority one.

**Definition of done:** I can open the game, play all 8 levels with music from start to finish with responsive controls on keyboard, mouse, touch, and gamepad; die and retry instantly; use practice mode; customize my icon; build, playtest, save, export, and re-import my own level in the editor; close the tab and come back to find my progress saved — with zero console errors and all automated tests passing.

Start now with the design doc and then slice (a). Keep working until the Definition of Done is met.
