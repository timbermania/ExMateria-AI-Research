# Bow Arrow Arc System

FFT renders bow attacks (weapon type 0x0C) as a 3D wireframe arrow lofted on a parabolic arc by charge handler 1 (`render_standard_spell_charge`, 0x801b0ffc–0x801b1538) — the only handler that calls `evaluate_parabolic_arc` (0x801af59c); handler 20's crossbow bolt flies a straight line instead. Arc height is not a preset: a two-candidate quadratic solver (FUN_801af3dc) yields a low (flat) arc that is always tried first, and a high (lofted) arc used only when terrain blocks the low trajectory, with the full flight validated frame-by-frame against terrain and standing units. Each frame handler 1 samples the parabola three times, tilts the arrow along the tangent, projects arc-space to world space through a GTE rotation matrix, and draws the dim gray arrow via `render_charge_effect` (0x801ae340). godot-learning reimplements the low-arc height math and per-frame parabolic flight in its projectile system, and its GPU combat engine uses a parabolic-arc line-of-sight check for bow shots.

## Points

- **Handler 1 (render_standard_spell_charge, 0x801b0ffc–0x801b1538) renders bow arrow trajectories as parabolic arcs and is the only handler that calls evaluate_parabolic_arc (0x801af59c); its three XREFs (0x801b1260, 0x801b128c, 0x801b12b0) all fall inside handler 1, while handler 20 (0x801b3938–0x801b40f4) flies its bolt in a straight line via FUN_801af6d8.** — `[S·R] 2/3`
  - S: 0x801b0ffc–0x801b1538 + XREFs 0x801b1260/0x801b128c/0x801b12b0, per research/working_documents/handler_1_spell_charge_arc_system.md
  - R: godot-learning/src/projectiles/Projectile3D.gd ("Parabolic Arc Flight", per-frame `arc_offset = arc_height * (1.0 - pow(2.0 * progress - 1.0, 2.0))`); no arc-specific test named
  - src: `research/working_documents/handler_1_spell_charge_arc_system.md`
- **Only the Bow (weapon type 0x0C) routes to handler 1: DAT_800943c4[0x0C] = anim_type 0x01, whose DAT_801b84dc record (0x801b84e0) holds func ID 1 and aux bytes 00 00 08, the signed halfword 0x0800 (g_duration_param_table = 0x801b84de + anim_type·4) selecting the Arrow model; spell charge effects use anim_types 0x02/0x04 → handler 4 (charge_effect_orbs), not handler 1, despite the Ghidra name.** — `[S·R] 2/3`
  - S: DAT_800943c4, DAT_801b84dc record at 0x801b84e0, g_duration_param_table 0x801b84de, per research/working_documents/handler_1_spell_charge_arc_system.md
  - R: godot-learning/src/projectiles/ProjectileManager.gd (only bow item_type_id == 12 gets an arc; crossbow 11 and all throw abilities stay straight-line); no arc-specific test named (tests/GPURangedCombatTest.gd covers bow damage, not trajectory routing)
  - src: `research/working_documents/handler_1_spell_charge_arc_system.md`
- **Trigger pipeline: the attacker's SEQ reaches 0xD9 → FUN_800689a4 builds position data via FUN_800687e0, routes the weapon through DAT_800943c4, and calls FUN_801ade7c to allocate the charge slot; no special-case override in FUN_800689a4 routes to anim_type 0x01.** — `[S] 1/3`
  - S: FUN_800689a4 (0x800689a4), FUN_800687e0, FUN_801ade7c, per research/working_documents/handler_1_spell_charge_arc_system.md
  - R: none — 0xD9 charge routing / FUN_800689a4 not present in godot-learning (probed godot-learning/src, godot-learning/tests)
  - src: `research/working_documents/handler_1_spell_charge_arc_system.md`
- **Handler 1 runs a 3-state machine: state 1 = init (LAB_801b1064), state 2 = active arc rendering (LAB_801b1250, checked first at dispatch), state 3 = complete (LAB_801b1508, returns 0 so the dispatcher removes the slot from the active list).** — `[S] 1/3`
  - S: dispatch 0x801b0ffc–0x801b1060; labels LAB_801b1064, LAB_801b1250, LAB_801b1508, per research/working_documents/handler_1_spell_charge_arc_system.md
  - R: none — handler state machine not present in godot-learning (probed godot-learning/src, godot-learning/tests)
  - src: `research/working_documents/handler_1_spell_charge_arc_system.md`
- **Init (state 1) computes only the XZ horizontal distance (no Y component, unlike handler 20's full 3D distance), the horizontal angle via ratan2, and the per-frame arc step `rcos(g_vertical_arc_angle) * (calc_arc_step_factor() >> 8) >> 4`, where calc_arc_step_factor (0x801af734) = fixed_point_sqrt((DAT_801b8878 * DAT_801b8874) >> 12) = fixed_point_sqrt((0xA8000 * 0x53A) >> 12); it also builds the GTE Y-rotation matrix, sets g_frame_countdown from g_effect_duration, and allocates the primitive buffer.** — `[S] 1/3`
  - S: 0x801b1064–0x801b124c; constants DAT_801b8878 = 0x000A8000, DAT_801b8874 = 0x0000053A, per research/working_documents/handler_1_spell_charge_arc_system.md
  - R: none — calc_arc_step_factor / arc step-per-frame not present in godot-learning (probed godot-learning/src, godot-learning/tests; projectile flight time derives from weapon SEQ timing instead)
  - src: `research/working_documents/handler_1_spell_charge_arc_system.md`
- **Arc height is not a fixed preset: FUN_801aff18 (0x801aff18) orchestrates the runtime computation — adjusting the caster Y down by 2/3 of the unit height (arrow fires from chest level), solving the two arc candidates, and always trying the low (flatter) arc first, using the high (lofted) arc only when terrain blocks the low trajectory.** — `[S·R] 2/3`
  - S: FUN_801aff18 (0x801aff18), writes g_arc_height_param (0x801b8b74) and g_effect_duration (0x801bbf3c), per research/working_documents/handler_1_spell_charge_arc_system.md
  - R: godot-learning/src/projectiles/ProjectileManager.gd (`_compute_psx_bow_arc` — "Low arc H (minus sign = flatter trajectory)"; low arc only, no high-arc fallback); no arc-specific test named
  - src: `research/working_documents/handler_1_spell_charge_arc_system.md`
- **FUN_801af3dc (0x801af3dc) derives two arc-height candidates as the two solutions of the quadratic endpoint constraint: high_arc = ((discriminant + ARC_RANGE) * 0x100) / (D >> 4), low_arc = ((ARC_RANGE - discriminant) * 0x100) / (D >> 4), discriminant = fixed_point_sqrt((ARC_RANGE/64 - h/32) * (ARC_RANGE/64) - (D >> 6)²) with ARC_RANGE = 0xA8000; in PSX world units the low-arc peak grows ~distance² (nearly flat at short range), the high-arc peak is ≈ constant 84 world units (7 tile heights, 1h = 12 units), both converge to 42 world units at maximum range, and zero XZ distance falls back to H = 0x1000.** — `[S·R] 2/3`
  - S: FUN_801af3dc (0x801af3dc), terrain height limit via FUN_8008dfac, per research/working_documents/handler_1_spell_charge_arc_system.md
  - R: godot-learning/src/projectiles/ProjectileManager.gd (`_compute_psx_bow_arc` mirrors the low-arc quadratic: `disc = R² - 4·Δy·R - 4·D²/K²`, `H = K²(R - sqrt(disc)) / (2·D)` with K = 4096, R = 336); no arc-height-specific test named (tests/GPURangedCombatTest.gd covers bow damage)
  - src: `research/working_documents/handler_1_spell_charge_arc_system.md`
- **FUN_801afb38 (0x801afb38) validates an arc candidate by simulating the full flight frame-by-frame (frame count = ceil(xz_distance / step)), and at each step FUN_801af8b4 (0x801af8b4) checks terrain (tile size 0x1C = 28; floor = tile_base_height · -12, ceiling = (tile_base_height - tile_depth) · -12, both panels) and all 21 battle unit slots (±8 world units XZ within the unit's height column); on a clear simulation it sets g_effect_duration to the total frame count.** — `[S·R] 2/3`
  - S: FUN_801afb38 (0x801afb38), FUN_801af8b4 (0x801af8b4), per research/working_documents/handler_1_spell_charge_arc_system.md
  - R: godot-learning/src/gpu/shaders/combat_combat.glslinc (`has_line_of_sight_arc` — per-step terrain check against a parabolic arc LOS, ARC_LOS_HEIGHT_PER_TILE = 0.5; approximation of the PSX frame simulation, not a port); validating test: tests/gambit_scenarios/scenarios_H_state_machine.gd (H3: tile height blocks the bow shot)
  - src: `research/working_documents/handler_1_spell_charge_arc_system.md`
- **When both arcs are blocked, selection is team-aware (byte 0x05 bits 4-5 of the caster vs byte 0x1BA bits 4-5 of the obstacle, via FUN_80180afc): a low-arc hit on an ENEMY wins (arrow stops at that enemy); an ally or terrain hit lets the high arc lob over; g_sound_effect_id (0x801b8b64) identifies the obstacle (unit ID, or -1 for terrain), and FUN_801aff18 returns the unit ID the arrow actually hits (-1 = terrain miss).** — `[S] 1/3`
  - S: FUN_801aff18 (0x801aff18) selection logic, team check FUN_80180afc, per research/working_documents/handler_1_spell_charge_arc_system.md
  - R: none — team-based arc obstacle avoidance not present in godot-learning (probed godot-learning/src, godot-learning/tests)
  - src: `research/working_documents/handler_1_spell_charge_arc_system.md`
- **evaluate_parabolic_arc (0x801af59c–0x801af6d4) reduces to f(P) = H·P/K - (H² + K²)·P² / (K³·R) with K = 4096, R = 336, across 4 magnitude-based overflow-safe shift paths that all converge at LAB_801af6ac; the parabola is symmetric (zero at P = 0, peak at P_peak = P_zero/2, P_zero = H·K²·R / (H² + K²)), and its positive values are negated by the caller for PSX Y-down.** — `[S·R] 2/3`
  - S: 0x801af59c–0x801af6d4, convergence label LAB_801af6ac, arc range constant (DAT_801b8878 << 1) >> 12 = 0x150, per research/working_documents/handler_1_spell_charge_arc_system.md
  - R: godot-learning/src/projectiles/Projectile3D.gd + ProjectileManager.gd (symmetric per-frame parabola `h * (1 - (2t - 1)²)`; `_compute_psx_bow_arc` derives h by matching the PSX parabola's B coefficient: (H² + K²)·D² / (4·K³·R)); no arc-specific test named
  - src: `research/working_documents/handler_1_spell_charge_arc_system.md`
- **State 2 samples the parabola three times per frame (prev, current, next); the arrow tilt = ratan2(prev - next, 2 · step_per_frame) + 0x400 (90°) offset, so the arrow points upward during ascent, level at the apex, and downward during descent.** — `[S·R] 2/3`
  - S: 0x801b1250–0x801b1504, per research/working_documents/handler_1_spell_charge_arc_system.md
  - R: godot-learning/src/projectiles/Projectile3D.gd (per-frame arc position + "Rotate arrow to face velocity direction (tangent to arc)" via analytic derivative of the parabola); no arc-specific test named
  - src: `research/working_documents/handler_1_spell_charge_arc_system.md`
- **The arc-space point (g_arc_progress >> 12, -arc_y >> 12, 0) is rotated by the prebuilt Y-rotation GTE matrix (g_effect_rotation_matrix, 0x801bacec) to a world offset from the caster, then through the view matrix (0x80098a24) to screen space; render_charge_effect (0x801ae340) is called with zero rotation offsets, dim gray RGB (0x40, 0x40, 0x40), and model_type 0x0800 (Arrow wireframe, identical to handler 20's Crossbow bolt).** — `[S] 1/3`
  - S: 0x801b1250–0x801b1504, view matrix 0x80098a24, render_charge_effect 0x801ae340, per research/working_documents/handler_1_spell_charge_arc_system.md
  - R: none — GTE arc-space-to-world transform / render_charge_effect not present in godot-learning (probed godot-learning/src, godot-learning/tests; projectiles use Godot 3D math instead)
  - src: `research/working_documents/handler_1_spell_charge_arc_system.md`
- **Handler 1 always allocates 0xC88 (3208) bytes with no variant selection (unlike handler 20): two double-buffered groups of 4 LineF2 wireframe lines (0x1C each, SetLineF2 0x80023ce0 — arrow body) + 13 sparkle sprites (0x24 each, init_sparkle_particle 0x80023d30 — trailing particles) + 0x800 bytes of vertex/transform workspace.** — `[S] 1/3`
  - S: allocation 0x801b11d0–0x801b11f0, per research/working_documents/handler_1_spell_charge_arc_system.md
  - R: none — PSX primitive buffer layout not present in godot-learning (probed godot-learning/src, godot-learning/tests; projectiles build Godot meshes instead)
  - src: `research/working_documents/handler_1_spell_charge_arc_system.md`
- **The impact sound triggers when g_frame_countdown == 2 (two frames before the arrow reaches the target), g_sound_effect_id (0x801b8b64) != -1, and slot+0x28 != 0, via trigger_effect_sound(slot+0x12, slot+0x1c) passing caster/target IDs; handler 1 has no excluded ability-ID ranges (handler 20 excludes 0x7D–0x7F and 0xF0–0xFD).** — `[S] 1/3`
  - S: 0x801b1424–0x801b1474, trigger_effect_sound (0x80068cc4), per research/working_documents/handler_1_spell_charge_arc_system.md
  - R: none — countdown-2 impact triggering not present in smd-player, fft-sound-driver, or godot-learning (probed smd-player/addons/exmateria_sound, fft-sound-driver, godot-learning/src, godot-learning/tests)
  - src: `research/working_documents/handler_1_spell_charge_arc_system.md`
- **Termination fires when the countdown reaches 0 or on terrain contact (world_pos_y >= 0, bltz at 0x801b14a4 — PSX positive Y = below ground), then sets the slot to state 3; when display_type (slot+0x1d) is 3/4/5 or g_sound_effect_id == -1 it sets g_cleanup_flag = 0x80000000 and signals completion, otherwise teardown is deferred to the sound system's impact handler.** — `[S] 1/3`
  - S: 0x801b1478–0x801b1504, g_cleanup_flag 0x801b8b98, signal_effect_complete 0x801ae2c8, per research/working_documents/handler_1_spell_charge_arc_system.md
  - R: none — cleanup flag / deferred teardown not present in godot-learning (probed godot-learning/src, godot-learning/tests)
  - src: `research/working_documents/handler_1_spell_charge_arc_system.md`

## Notes

(empty — user territory)

## Related

- [[Spell Charge Effect System]]
- [[Effect System Index]]
- [[GTE World-to-Screen Transform]]
- [[PSX GPU Primitives]]
