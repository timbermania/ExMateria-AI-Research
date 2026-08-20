# Weather Rain Renderer

The BATTLE.BIN map-graphics particle system (`FUN_800eeaf8` → `FUN_800eeecc`) renders FFT's rain as a **world-space 3D particle volume** over the battlefield: a fixed 32 drops scattered across the map's tile footprint, each a two-endpoint streak transformed by the live isometric camera through a single shared GTE `MVMVA` (orthographic — no `RTPS`, no terrain/tile lookup), that on straddling its position-indexed ground line in one frame converts into a pinned ground-ripple splat (one connected fall→splat lifecycle per slot). Primitives are POLY_FT3 textured triangles under additive-quarter blend (abr=3, no alpha), occluded purely by the scene's shared ordering-table depth sort; Strength scales only the velocity/scatter triple, never density, and rain-vs-snow is a per-map flag. The 2026-07-02 session-8 live re-verification (§15) corrected the render model to screen-vertical constant-screen-size streaks, additive grey-ramp heads, and conditional (~58 %) dim diamond-ring splats; the Godot port re-derived against it (`ScenarioWeather` + `ScenarioWeatherTest` 27/0 + F3 "Scenario Weather" panel) re-verified headful the same day.

## Points

- **The rain path (spawner `a0=0x90`, `LAB_800ef2e8`) spawns exactly 32 drops, template-free — the 64-record rain array is 32 drops × 2 endpoints (bottoms `DAT_800fc358[0..31]`, tops `DAT_800fc458[0..31]`, live-verified sharing X/Z) — while the snow path (`a0=0x8f`, `LAB_800eef80`) spawns 64 particles using the 3-emitter template `DAT_800e6b50`; spawn/respawn scatters `X = rand() % (DAT_800f6860·28)`, `Z = rand() % (DAT_800f6864·28)`, `Y = −(rand()%384) − 0x180` with 28 = sub-units/tile, so the scatter box is exactly the map tile footprint (live Orbonne `f6860/f6864 = 10/14` → 10×14 tiles) and every landing re-scatters X/Z across the whole box.** — `[S·D·R] 3/3`
  - S: spawner `FUN_800eeecc` / rain init loop `0x800ef310` / respawn `0x800ea0e0`, record bases `0x800fc358`/`0x800fc458`, map dims `0x800f6860`/`0x800f6864` (`battle_disassembly.txt`)
  - D: live drop-array distribution `X[12..277] Y[−362..−48] Z[0..368]` + all-32 endpoint X/Z sharing on `orbonne_rain_battle_active.sstate` (2026-07-02)
  - R: `godot-learning/src/scenarios/ScenarioWeather.gd` (`DROP_COUNT=32`, `SUBUNITS_PER_TILE=28.0`, map-width/depth scatter box) + `tests/ScenarioWeatherTest.gd` `_test_sim_falls_lands_splats_and_respawns`
  - src: `research/working_documents/WEATHER_OPCODE_3C_INVESTIGATION.md`
- **Both streak endpoints integrate the same per-frame Y velocity `vy = (fa6a4 + layer)·DAT_80045980` with `DAT_80045980 = 2` (live); the two parallax depth layers are assigned by SLOT INDEX, not Z — slots 0..15 take the near bonus `fa6a8`, slots 16..31 the far bonus `fa6ac` — giving Strength-3 live rates of 22 px/frame near and 28 px/frame far, matching the observed `ΔY≈+22/sample`.** — `[S·D·R] 3/3`
  - S: velocity block `0x800ea2e4`, frame scalar `0x80045980` (`battle_disassembly.txt`)
  - D: live-confirmed near-layer rate vs §11.8 time-series `ΔY≈+22/sample` (2026-07-02, GAP 3)
  - R: `godot-learning/src/scenarios/ScenarioWeather.gd` `tick()` `vy = (fa6a4 + layer) * ANIM_SPEED_SCALAR`, `NEAR_LAYER_SLOTS=16` + `tests/ScenarioWeatherTest.gd` `_test_sim_falls_lands_splats_and_respawns`
  - src: `research/working_documents/WEATHER_OPCODE_3C_INVESTIGATION.md`
- **The rain is a world-space 3D particle volume — neither a flat 2D screen overlay nor per-tile-projected: each vertex is transformed by the shared GTE `MVMVA` helper `SUB_8001d578` (`RotMatrix·(X,Y,Z) + TR`, byte-decoded from live RAM; no `RTPS` → orthographic) using the live isometric battle camera matrix (`R/4096 = [[+.707,0,−.707],[−.315,+.894,−.316],[+.631,+.447,+.632]]`, `TR≈(220,353,517)` — the same matrix that draws the map), and the whole weather region `0x800e9000–0x800ef000` contains 0 GTE ops except that one shared `MVMVA`, so no terrain-height / tile-mesh lookup exists.** — `[S·D·R] 3/3`
  - S: `0x8001d578`, weather draw calls `0x800ea430`/`0x800ea498` (drop endpoints) and `0x800ea654` (splat), region `0x800e9000–0x800ef000` (6145 instrs) (`battle_disassembly.txt`)
  - D: D10 weather-draw BP at `0x800ea430` — captured GTE rotation matrix byte-identical on every hit and identical to the map camera (2026-07-02)
  - R: `godot-learning/src/scenarios/ScenarioWeather.gd` Node3D world-space system parented under the game camera + `tests/ScenarioWeatherTest.gd`
  - src: `research/working_documents/WEATHER_OPCODE_3C_INVESTIGATION.md`
- **Occlusion comes from participation in the scene's ordering-table (painter's-algorithm) depth sort: each drop's rotated Z becomes `otz = (Z>>2)` clipped to `1..0x17f` (drops outside the window are skipped — the coarse near/far cull), the drop prim is `AddPrim`'d into the SAME ordering table as the map geometry (`SUB_80023bb4`, the psyq OT linked-list insert), and the GPU renders strictly back-to-front so a nearer wall paints over a deeper drop; a GTE-overflow flag cull and projected-screen-rect clamp also apply — no Z-buffer, no collision test.** — `[S·D·R] 3/3`
  - S: drop cull `0x800ea4a8`–`c0`, OT insert `SUB_80023bb4 @0x80023bb4` (`battle_disassembly.txt`)
  - D: live rain frame-diff overlay — density peaks over the courtyard mid-rows, thins into the sky, cuts out behind the chapel tower = occlusion, not a bounding test (2026-07-02)
  - R: `godot-learning` depth buffer occludes the world-space `ScenarioWeather` particles against the map mesh for free + headful verify (2026-07-02, session 9)
  - src: `research/working_documents/WEATHER_OPCODE_3C_INVESTIGATION.md`
- **Drop and splat prims are POLY_FT3 textured triangles (code `0x26`, semi-transparent) with `clut=0x7880` (VRAM x=0, y=482) and `tpage=0x007F` (4bpp page base VRAM x=960, y=256, abr=3 = additive-quarter `dst + src/4`); GPU vertex colour modulates drops `0x64` grey and splats `0x80` grey.** — `[S·D·R] 3/3`
  - S: static D6 prediction; CLUT/tpage/semi-trans helpers `SUB_80023a54`/`SUB_8002398c`/`SUB_80023c68` (`battle_disassembly.txt`)
  - D: D6 — live GPU packet at `0x8010ab84` matched the static prediction exactly (2026-07-02)
  - R: `godot-learning/src/scenarios/ScenarioWeather.gd` additive-quarter spatial shader (`blend_add`) + headful rasterize verify (2026-07-02)
  - src: `research/working_documents/WEATHER_OPCODE_3C_INVESTIGATION.md`
- **The drop texture is a 1-D grayscale gradient strip in 4bpp page `(960,256)`, texel row `v=175, u=216→247`: a ~5-texel bright head (grey 224 = CLUT idx 1, `u219–223`, with a 3-texel anti-aliased cap) followed by a long linear fade 200→24; the shared `CLUT(0,482)` is a linear 16-step white(224)→black(8) ramp with idx 0 transparent; drops draw as ~1px-wide vertical triangles of projected length 4–75 px (mostly 16–68 px).** — `[S·D·R] 3/3`
  - S: static prediction + live decode of the gradient strip (`battle_disassembly.txt` / VRAM layout)
  - D: D7 live VRAM capture of the drop strip + §15.2 texel-dump re-verification (2026-07-02)
  - R: `godot-learning/src/scenarios/ScenarioWeather.gd` `_make_drop_texture` procedural gradient strip (no ISO assets) + headful verify
  - src: `research/working_documents/WEATHER_OPCODE_3C_INVESTIGATION.md`
- **The sky-streak and the ground-ripple are ONE particle lifecycle: when a falling streak crosses its per-particle ground line, the drop converts to a splat at the same slot — the splat inherits the drop's X/Z (splat position words `DAT_8011a1d4/d8` each have exactly one writer, `0x800ea5e0`/`0x800ea5f8`, so no independent splat spawner exists), its Y is PINNED to the position-indexed ground line (`s2 − 8`, not inherited), its `active` is armed to 16, and the drop's state byte `DAT_800fc35e` marks the conversion.** — `[S·D·R] 3/3`
  - S: conversion block `0x800ea538`–`0x800ea620`, XREF census of splat records `0x8011a1d4/d6/d8` (`battle_disassembly.txt`)
  - D: D11 live BP at the conversion store `0x800ea5fc` — all 12 hits `splat.X==drop.X && splat.Z==drop.Z` at the same idx with `active←0x10`; single-slot time series shows fall→land→respawn (2026-07-02)
  - R: `godot-learning/src/scenarios/ScenarioWeather.gd` `_spawn_splat` pinned to landing X/Z + ground height + `tests/ScenarioWeatherTest.gd` `_test_sim_falls_lands_splats_and_respawns`
  - src: `research/working_documents/WEATHER_OPCODE_3C_INVESTIGATION.md`
- **Splat ripples run a 16-frame `active` countdown; the UV frame is refreshed only when `active&3==0` (every 4th frame), frame index `(active<<1)&0x18` into `[u64,u48,u32,u16]` — `active∈{12,8,4,0}` → `u∈{16,32,48,64}`: impact dot → small ring → wide horizontal ellipse → dissipating broken ring, each held 4 game-frames — one impact→expand→dissipate ripple per landing.** — `[S·D·R] 3/3`
  - S: splat draw handler `0x800ea740` (`battle_disassembly.txt`)
  - D: live BP at `0x800ea788` — 3969 hits yielding the exact `active→u` map (2026-07-02, GAP 2)
  - R: `godot-learning/src/scenarios/ScenarioWeather.gd` `splat_frame(active) = clampi(3 - ((active+3) >> 2), 0, 3)` + `tests/ScenarioWeatherTest.gd` `_test_splat_frame_cadence`
  - src: `research/working_documents/WEATHER_OPCODE_3C_INVESTIGATION.md`
- **Splat spawning is CONDITIONAL, not every landing: conversion requires `state==0` and the streak SEGMENT straddling the ground line within one frame (`bottomY > s2` AND `topY < s2`) — a streak shorter than the per-frame fall step (near 22 / far 28 px) can jump over `s2` between frames and skip its splat; it is a geometric gate, not RNG, and live ≈55–65 % of landings splat (~40 % skip).** — `[S·D·R] 3/3`
  - S: conversion gate `0x800ea51c`–`0x800ea590` byte-decode (`battle_disassembly.txt`)
  - D: live cool counter-BP at the arm site `0x800ea5c0` — 118 splat-arms while ~200 falls occur over ~90 frames (2026-07-02, §15.3)
  - R: `godot-learning/src/scenarios/ScenarioWeather.gd` `tick()` straddle gate (splat only when the bottom crossed the ground line and the top still straddles it) + `tests/ScenarioWeatherTest.gd` `_test_straddle_gate_skips_overshoot_but_hits_straddle`
  - src: `research/working_documents/WEATHER_OPCODE_3C_INVESTIGATION.md`
- **Rain streaks are PERFECTLY SCREEN-VERTICAL at any FFT camera angle: the live GTE rotation has `R12 = 0` exactly, so world-Y contributes zero screen-X, and `dScreenX = 0` for all 32 measured drops (angle-from-vertical `0.0°`); the scale is orthographic and constant (obj-Y projects at `R22/4096 = 0.894` px/sub-unit, obj-X/Z at 0.707) with no perspective foreshortening, so near and far drops of equal length draw equal — slanted "comet" streaks are Godot perspective-camera artifacts on world-vertical ribbons, not faithful behaviour.** — `[S·D·R] 3/3`
  - S: `R12` analysis of the byte-decoded camera matrix, no-`RTPS` draw path (`battle_disassembly.txt`)
  - D: §15.1 — live dump of all-32 endpoint record arrays + hand-projection through the live GTE matrix, screen streak lengths 4–75 px (mostly 16–68) (2026-07-02)
  - R: `godot-learning/src/scenarios/ScenarioWeather.gd` screen-space clip-space billboard streaks (constant screen size, `PX_PER_SUBUNIT_Y=0.894`, `REF_HEIGHT=240`) + headful verify (2026-07-02, session 9)
  - src: `research/working_documents/WEATHER_OPCODE_3C_INVESTIGATION.md`
- **PSX drops carry NO alpha channel: the per-drop brightness variation is additive-quarter intensity over a varying background (visible over the dark chapel arch, invisible over pale stone) plus the head→tail grey ramp — only the bright head (~5–8 px, +56 to the framebuffer) adds visibly, and the tail (grey 24–104 → +6..+26) is near-invisible, which is why drops read as small marks rather than long streaks.** — `[S·D·R] 3/3`
  - S: blend abr=3 (`dst + src/4`) from D6 static/live decode; no alpha path in the weather region (`battle_disassembly.txt`)
  - D: §15.2 live VRAM texel dump of the drop gradient strip (2026-07-02)
  - R: `godot-learning/src/scenarios/ScenarioWeather.gd` additive grey-ramp drop material (albedo = grey-ramp · intensity, texture alpha ignored) + headful verify (2026-07-02, session 9)
  - src: `research/working_documents/WEATHER_OPCODE_3C_INVESTIGATION.md`
- **The real splats are thin, DIM, sparse diamond-ring outlines, not clean bright filled ellipses: the four 16×16 cells (4bpp page `(960,256)`, `v=160–175`, `u∈{16,32,48,64}`) each hold a main ground ring + a tiny bright accent — impact blob (13/256 texels) → small ring (23) → wide broken ring (46) → dissipating four corner dots (9); ring texels are grey 40–120 (only the tiny accent reaches 224), and with vertex colour `0x80` + additive-quarter a grey-72 ring pixel adds only +18 to the frame.** — `[S·D·R] 3/3`
  - S: texel structure of the 4 splat cells (VRAM layout decoded from the live capture)
  - D: §15.4 live dump of all 4 splat UV cells as index grids (2026-07-02)
  - R: `godot-learning/src/scenarios/ScenarioWeather.gd` dim diamond-ring splat texture/material (intensity ~0.15–0.45 pre-additive) + headful verify (2026-07-02, session 9)
  - src: `research/working_documents/WEATHER_OPCODE_3C_INVESTIGATION.md`
- **The ground line `s2` that drops cross into splats is a per-particle threshold from a coarse per-map 2-D ground-height grid at `0x8018f8cc`, indexed as a grid cell from the drop's X/Z (`index = (Z-cell)·mapWidth + (X-cell)`, `mapWidth = DAT_800f6860`), stride-8 records: `+2` = floor level, `+3` top bit = edge/off-map flag, `s2 = −((level + flagBit)·12)`; edge cells resolve to the default `−48` line (the uniform Y=−48 conversion threshold seen live); the grid is per-map data built at map load — exact fill routine still untracked.** — `[S·D·R] 3/3`
  - S: table `0x8018f8cc`, index site `0x800ea174` (`battle_disassembly.txt`)
  - D: live dump of the first 16 cells = height ramp `0,0,0,5,6,7,8,9,9,9,0,0,0,6,6,9` rising front→back (2026-07-02, GAP 4)
  - R: not reproduced — `godot-learning/src/scenarios/ScenarioWeather.gd` `_sample_ground` samples the map-tile height once at spawn (`MapComposer.get_tile`) as the modernized stand-in for the baked grid, validated by `tests/ScenarioWeatherTest.gd` sim
  - src: `research/working_documents/WEATHER_OPCODE_3C_INVESTIGATION.md`

## Notes

(empty — user territory)

## Related

- [[Weather Opcode]]
- [[GTE World-to-Screen Transform]]
- [[Ordering Table & AddPrim]]
- [[PSX GPU Primitives]]
- [[PSX Texture Page Register]]
- [[Effect Frame Pacing]]
