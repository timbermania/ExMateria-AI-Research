# Elemental Puff Particle System

The hit-impact puff is TRAP charge-effect handler 3 (FUN_801b1aec), selected when byte 7 of the target's ability data is 5 in the hit-impact switch at FUN_801adfec (animation_type 20 → func_id 3 via DAT_801b84dc). It is the most minimal handler in the TRAP particle system: config 16 (0x801b8844) has every velocity, scatter, weight, and radius field zeroed, spawns a single particle on a 1-frame window 16 units above the impact point, and lets it fall under pure gravity (no weight counterforce, net 4096/tick downward — the fastest of the documented handlers) while a hardcoded CLUT 0x7ACD renders it; the specific abilities that produce byte7=5 are still unknown. In godot-learning the config data and shared physics are implemented (`emitters.json` index 16, `TrapEffect.gd` `HANDLER_CONFIGS[3]` with fixed palette 13 = 0x7ACD), but no gameplay trigger routes to handler 3 yet.

## Points

- **Handler 3 (FUN_801b1aec) is the hit-impact particle handler selected when byte 7 of the target's ability data (ability_data[0x18e] jump table) is 5 in the hit-impact switch at FUN_801adfec: byte7=5 forces animation_type 20 (0x14), which DAT_801b84dc[0x14·4] at 0x801b852c maps to func_id 3 and g_charge_effect_handlers[3] at 0x801b890c points at 0x801b1aec; byte7=0,1,3,4 routes to handler 2, byte7=6,7 to handler 21 (FUN_801b40f8), and all hit clouds funnel through FUN_8006894c → FUN_800687e0 → FUN_801adfec.** — `[S] 1/3`
  - S: byte7 dispatch table and routing chain, `research/working_documents/handler_3_particle_system.md` §1
  - R: none — byte7=5 routing not present in godot-learning (probed `godot-learning/src/effects/EffectManager.gd` + `tests/`; Godot routes impact by ability formula instead: break/holy-sword → handler 21, all else → handler 2)
  - src: `research/working_documents/handler_3_particle_system.md`
- **Config 16 (0x801b8844, 0x2E bytes) is the most minimal emitter config in the TRAP system: frame_table_index 16, spawn window [0,1) (single frame), max_particles 1, direction_flags 0x0600 (target-anchored + directional), pos_scatter (0, −16, 0), and every velocity, vel_range, scatter_half, weight, and radius field zeroed; spawn_rate 1, spawn_count 1, lifetime −1/−1 (animation-driven).** — `[S·R] 2/3`
  - S: 46-byte hex dump + field table at 0x801b8844, `research/working_documents/handler_3_particle_system.md` §2
  - R: `godot-learning/assets/effects/trap/emitters.json` index 16 ("beam"; raw hex byte-identical to the doc's config-16 dump, `config_addr` 0x801B8844) + `godot-learning/src/effects/TrapEffect.gd` `HANDLER_CONFIGS[3] = [16]` ("Elemental Puffs"); no handler-3-specific test
  - src: `research/working_documents/handler_3_particle_system.md`
- **The spawn runs SPHERICAL_RANDOM velocity mode (velocity_mode 0x0000, dispatched to LAB_801a8408 in FUN_801a7f5c; dispatch check at ram 0x801a83c4–0x801a83d4) but produces exactly zero velocity: the magnitude of vel (0,0,0) is 0 so the random direction × 0 = (0,0,0), vel_range (0,0,0) adds no noise, and speed = rand(radius 0,0) = 0 — the particle spawns stationary at anchor + (0, −16, 0) and moves only under gravity.** — `[S·R] 2/3`
  - S: velocity-mode dispatch 0x801a83c4–0x801a83d4 + zero-magnitude walkthrough, `research/working_documents/handler_3_particle_system.md` §3
  - R: `godot-learning/src/effects/TrapEffect.gd` `_init_particle_velocity` SPHERICAL_RANDOM branch yields zero velocity when velocity and radius are both zero (shared by all emitters; no handler-3-specific test)
  - src: `research/working_documents/handler_3_particle_system.md`
- **Handler 3 is a 3-state machine (dispatch 0x801b1aec–0x801b1b34): state 1 (LAB_801b1b38) zeroes active_particle_count/frame_counter/fade_counter and transitions to state 2 with no SFX (unlike handler 12's sound 0x22 on init); state 2 (LAB_801b1b50) spawns via FUN_801b0cf0(config 16, clut_param 0x0D) then falls through to state 3; state 3 (LAB_801b1b5c) runs the shared physics pipeline only; the particle CLUT is hardcoded 0x7ACD (0x7AC0 + 0x0D; VRAM clut_x = (0x7ACD & 0x3F) << 4 = 208, clut_y = 0x7ACD >> 6 = 491) regardless of the ability's element.** — `[S·R] 2/3`
  - S: state addresses, spawn call, and CLUT derivation, `research/working_documents/handler_3_particle_system.md` §4
  - R: `godot-learning/src/effects/TrapEffect.gd` `HANDLER_PALETTE_OVERRIDES[3] = 13` (fixed palette 13 = 0x7ACD sub-palette, overrides element); no handler-3-specific test
  - src: `research/working_documents/handler_3_particle_system.md`
- **The per-tick physics pipeline (states 2 and 3, ram 0x801b1b5c–0x801b1b9c) is the shared TRAP pipeline — FUN_801a9a24 (frame prepare), set_inertia_threshold(0x230) = 560 (velocity components decay ×(4096−560)/4096 ≈ 0.863/tick), gravity {0, 0x1000, 0} (4096/tick downward) applied via set_gravity_vector (0x801a6984), FUN_801b0f08(0x10) (particle iteration), FUN_801a9a3c (integration), FUN_801a99f0 (render) — and with zero weight the net downward force is the full 4096/tick, the fastest of the documented handlers (8/9/12: ~3568–3600/tick, 13: ~2896–2912/tick).** — `[S·R] 2/3`
  - S: pipeline addresses, gravity-only comparison, `research/working_documents/handler_3_particle_system.md` §5
  - R: `godot-learning/src/effects/TrapEffect.gd` shared pipeline constants (DAMPING = (4096 − 560)/4096, GRAVITY_Y) + `emitters.json` index 16 weight 0; no handler-3-specific test
  - src: `research/working_documents/handler_3_particle_system.md`
- **Termination uses the fade_counter XOR-1 trick (xori v0, a0, 0x1; sltu at 0x801b1bb4): the handler returns ACTIVE while fade_counter ≠ 1 and DONE when it equals 1; once the lone particle dies, if active_count == 0 and frame_counter > DAT_801b8846 (spawn_check_lo = 0) the fade_counter increments each tick — so handler 3 returns ACTIVE for one extra frame after the puff dies, then DONE on the next tick, at which point the dispatch loop deallocates the slot.** — `[S] 1/3`
  - S: XOR pattern at 0x801b1bb4, DAT_801b8846 load at 0x801b1bd8, `research/working_documents/handler_3_particle_system.md` §6
  - R: none — XOR termination not present in godot-learning (probed `godot-learning/src/effects/TrapEffect.gd`; Godot TRAP path keeps its own per-emitter particle arrays and does not mirror the ROM state machine / fade_counter)
  - src: `research/working_documents/handler_3_particle_system.md`

## Notes

(empty — user territory)

## Related

- [[TRAP Hit Effect Particle System]]
- [[Hit Reaction Particle Burst]]
- [[Knight Break Impact Particle System]]
- [[TRAP Charge Particle System]]
