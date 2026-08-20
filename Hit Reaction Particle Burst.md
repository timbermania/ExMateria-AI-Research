# Hit Reaction Particle Burst

The hit/reaction particle burst is charge-effect handler 12 (FUN_801b2c0c), which fires when a unit takes damage during battle action resolution (DAT_80096208 bit 3 set, via the reaction-pose path — not footstep dispatch and not a SEQ opcode): it plays battle SFX 0x22 on init, then spawns a few short-lived particles on a fixed 15-frame lifetime through a 7-frame spawn window, launched in a wide upward cone (SPHERICAL_RANDOM mode with uniform ±52° angular spread) around the target, and lets gravity arc them back down. Its 0x2E-byte config (entry 8 at 0x801b86d4) sits in the in-memory config table shared with the TRAP hit clouds (handler 2), the Charge+X effect (handler 6), and the footstep handlers 8/9. In godot-learning the config data and the shared SPHERICAL_RANDOM physics are implemented (`emitters.json` index 8, `TrapEffect.gd` `HANDLER_CONFIGS[12]`), but no gameplay trigger wires handler 12 up yet.

## Points

- **Handler 12 is triggered from the battle action resolution, not footstep dispatch: FUN_80071ee8 checks DAT_80096208 bit 3 (0x8), and when set calls execute_unit_reaction_pose(0, unit) then FUN_80068ad4(unit) → FUN_80068a20(unit, 0x0E) → allocate_sprite_animation_slot(0x0E, …), which resolves anim_type 0x0E via DAT_801b84dc[0x0E·4] = func_id 12 to g_charge_effect_handlers[12] = 0x801b2c0c; the unit+0x134 null guard always passes because ability execution sets +0x134 via FUN_80087a28; anim_type 0x0E is hit/reaction-triggered, not "SEQ assigned" as the wiki label claims.** — `[S] 1/3`
  - S: trigger chain (0x80071ee8, DAT_80096208 bit 3, 0x80068ad4, 0x80068a20, DAT_801b84dc[0x0E·4] = {12,0,0,0}, g_charge_effect_handlers[12] at 0x801b8930 → 0x801b2c0c), `research/working_documents/handler_12_particle_system.md` §1
  - R: none — reaction-pose / handler-12 trigger not present in godot-learning (probed `godot-learning/src/` + `tests/` for execute_unit_reaction_pose, 0x80071ee8, DAT_80096208; impact hits only route to handlers 2/21 via `EffectManager.gd`)
  - src: `research/working_documents/handler_12_particle_system.md`
- **Config 8 (0x801b86d4, 0x2E bytes): frame_table_index 2 (animation sequence 2), spawn window [0, 7) (frames 0–6), max_particles 32, direction_flags 0x0600 (target-anchored + directional), velocity_mode 0x0000 (SPHERICAL_RANDOM), pos_scatter (0, −17, 0) — static 17-unit upward offset, runtime-modified by nothing — velocity ellipsoid semi-axes (8, 8, 8), vel_range (2048, 0, 0), scatter_half (1184, 1184, 1184), weight −528…−496 (upward, anti-gravity), radius 1044/1092, spawn_rate 3, spawn_count 1, lifetime 15/15 (fixed 15 frames, not animation-driven like configs 6/7); spawned with clut_param 0x0C → CLUT 0x7ACC.** — `[S·R] 2/3`
  - S: 46-byte hex dump + field table, `research/working_documents/handler_12_particle_system.md` §2 (+ §7 comparison: CLUT 0x7ACC, sound 0x22 on init)
  - R: `godot-learning/assets/effects/trap/emitters.json` index 8 ("shimmer_glow"; `raw.hex` byte-identical to the doc's config-8 dump, `config_addr` 0x801B86D4) + `godot-learning/src/effects/TrapEffect.gd` `HANDLER_CONFIGS[12] = [8]` ("Hit/reaction dust"); no handler-12-specific test
  - src: `research/working_documents/handler_12_particle_system.md`
- **SPHERICAL_RANDOM (velocity_mode 0x0000) dispatches to LAB_801a8408 in FUN_801a7f5c (dispatch: 0x000/0x400/0x410 → SPHERICAL_RANDOM at LAB_801a8408, 0x010 → SCATTER at LAB_801a850c; disasm ram 0x801a83c4–0x801a83d4) and runs in two separate phases: PHASE 1 POSITION uses three fully random angles to place the particle at a random point on an ellipsoidal shell whose semi-axes are the config's "velocity" fields (config 8: ±8 on all axes), centered on the anchor + pos_scatter (0, −17, 0); PHASE 2 VELOCITY uses speed = rand(radius_min, radius_max) with the base vector (0, speed, 0) rotated by vel_range + rand(−scatter_half/2 … +scatter_half/2) per axis — config 8's vel_range (2048, 0, 0) is a 180° X flip making (0, −speed, 0) upward, and scatter_half (1184, 1184, 1184) adds ±52° on all axes, so particles burst outward-and-upward in a wide cone; with spawn_rate 3 over the 7-frame window only ~2–3 particles ever spawn.** — `[S·R] 2/3`
  - S: dispatch table + two-phase walkthrough (LAB_801a8408, 0x801a83c4–0x801a83d4), `research/working_documents/handler_12_particle_system.md` §3
  - R: `godot-learning/src/effects/TrapEffect.gd` `_init_particle_position` (random unit-sphere direction × velocity semi-axes + pos_scatter, shared by all modes) and `_init_particle_velocity` SPHERICAL_RANDOM branch (vel_range ± scatter_half/2 cone angles, speed from radius, `cone_basis * Vector3(0, -speed, 0)`); no handler-12-specific test
  - src: `research/working_documents/handler_12_particle_system.md`
- **Handler 12 is a 3-state machine: state 1 (LAB_801b2c58) zeroes active_particle_count/frame_counter/fade_counter, plays battle SFX 0x22, and transitions to state 2 — the only behavioural difference from the silent footstep handlers 8/9; state 2 (LAB_801b2c74) spawns via FUN_801b0cf0(config 8, clut_param 0x0C) then falls through to state 3's pipeline; state 3 (LAB_801b2c80) runs the shared physics pipeline (FUN_801a9a24, set_inertia_threshold(0x230), gravity (0, 0x1000, 0), FUN_801b0f08(8), FUN_801a9a3c, FUN_801a99f0) with no spawning, identical to handlers 8/9; termination uses the same fade_counter XOR-1 trick as handlers 8/9 — once active_count is 0 and frame_counter > spawn_check_lo (DAT_801b86d6 = 0), the handler returns ACTIVE for one extra frame then DONE, and the dispatch loop deallocates the slot.** — `[S] 1/3`
  - S: state addresses 0x801b2c58/0x801b2c74/0x801b2c80 + termination logic, `research/working_documents/handler_12_particle_system.md` §4–§6
  - R: none — handler-12 state machine / SFX-0x22-on-init not present in godot-learning (probed `godot-learning/src/` + `tests/`; shared spawn/physics/cleanup exist in `TrapEffect.gd` but there is no per-handler init sound or XOR termination, and no gameplay call site plays handler 12; also probed `smd-player/addons/exmateria_sound/` + `fft-sound-driver/` for SFX 0x22 — not present)
  - src: `research/working_documents/handler_12_particle_system.md`

## Notes

(empty — user territory)

## Related

- [[TRAP Hit Effect Particle System]]
- [[TRAP Charge Particle System]]
- [[Unit White Flash Reaction Handler]]
