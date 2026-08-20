# Knight Break Impact Particle System

Handler 21 (FUN_801b40f8) is the TRAP charge-effect handler for Knight Break / Holy Sword impact: the hit impact switch forces anim_type 0x12 when the target unit data byte 7 is 6 or 7, config 11 at 0x801b875E drives SCATTER-mode spawning with a negative radius (outward burst) and the strongest positive weight of any handler (+1392/+1424, net 5488–5520/tick downward), so the overbright white particles (CLUT 0x7ACA) spawn on a tiny flat disk, burst outward, and immediately plummet — a brief 5-frame-window "impact splash". The godot-learning reimplementation routes formula 0x25/0x2E impacts to this handler via `TrapEffect` with the config at `emitters.json` index 11 (validated by `GPUBreakTrapTest.gd`); the ROM 3-state machine and XOR termination are not mirrored.

## Points

- **Handler 21 (FUN_801b40f8) renders the Knight Break / Holy Sword impact particles and is triggered when the hit impact switch FUN_801adfec forces anim_type 0x12 on target-unit data byte 7 = 6 or 7 (upstream: SEQ 0xDE target reaction → hit cloud spawner FUN_8006894c → FUN_800687e0); DAT_801b84dc[0x12·4] (0x801b8524) → func_id 21 and g_charge_effect_handlers[21] (0x801b8954) → 0x801b40f8, and the triggering abilities are the Knight Break formula 0x25 set (HeadBreak/ArmorBreak/ShieldBreak/WeaponBreak) and the 4 Holy Sword abilities at formula 0x2E, with the effect suppressed when the targeted slot has no equipment, the target has Maintenance status, or the hit calculation fails.** — `[S·R] 2/3`
  - S: ram 0x801b40f8, 0x801b8524, 0x801b8954, FUN_801adfec byte-7 switch, per `research/working_documents/handler_21_particle_system.md` §1 (+ wiki `charge_effect_particle_config_selection.txt` "0x12 → 21 (KB Impact) byte7=6,7")
  - R: `godot-learning/src/effects/EffectManager.gd` `spawn_trap_effect` (formula 0x25/0x2E → handler 21) + `godot-learning/src/effects/TrapEffect.gd` `HANDLER_CONFIGS[21] = [11]` / `HANDLER_GROUP_NAMES[21] = "Knight Break"`, validated by `godot-learning/tests/GPUBreakTrapTest.gd` (in `tests/run_all_tests.sh`)
  - src: `research/working_documents/handler_21_particle_system.md`
- **Config 11 (0x801b875E, 46 bytes) sets frame_table_index 7, spawn window [0, 5) — shortest of any shared-spawner handler (H12 = 7, H13 = 15, H19 = 48) — max_particles 30, direction_flags 0x0600 (target-anchored + directional), velocity_mode 0x0010 (SCATTER), static pos_scatter (0, −24, 0), spawn ellipsoid (4, 2, 4) with magnitude 6 (smallest of any SCATTER handler), all-zero vel_range and scatter_half, weight +1392/+1424, radius −2384/−2368, spawn_rate 3, spawn_count 1, lifetime −1/−1 (animation-driven), and clut_param 0x0A → CLUT 0x7ACA (overbright white, VRAM clut_x = 160, clut_y = 491).** — `[S·R] 2/3`
  - S: 46-byte hex dump + field table, `research/working_documents/handler_21_particle_system.md` §2
  - R: `godot-learning/assets/effects/trap/emitters.json` index 11 ("scattered_dots"; `raw.hex` byte-identical to the doc's config-11 dump, `config_addr` 0x801B875E) + `godot-learning/src/effects/TrapEffect.gd` `HANDLER_CONFIGS[21] = [11]` / `HANDLER_PALETTE_OVERRIDES[21] = 10`, validated by `godot-learning/tests/GPUBreakTrapTest.gd` (in `tests/run_all_tests.sh`)
  - src: `research/working_documents/handler_21_particle_system.md`
- **Handler 21 carries the strongest positive (downward) weight of any handler (+1392/+1424, ~8× the +176/+192 of handler 19, the only other positive-weight handler), so the net downward force is 5488–5520/tick against 4096/tick gravity (1.34× gravity alone) — the fastest-falling particles of any handler, 29% faster than handler 19 (4272–4288), 53% faster than handlers 8/9/12 (3568–3600), and 89% faster than handler 13 (2896–2912).** — `[S·R] 2/3`
  - S: config 11 +0x22/+0x24 (0x801b875E) and the cross-handler net-force table, `research/working_documents/handler_21_particle_system.md` §4/§6
  - R: `godot-learning/assets/effects/trap/emitters.json` index 11 `weight [1392, 1424]` + `godot-learning/src/effects/TrapEffect.gd` `_init_particle_physics` (per-particle weight sampled from the config range)
  - src: `research/working_documents/handler_21_particle_system.md`
- **The negative SCATTER radius (−2384/−2368, ~67% larger magnitude than handler 13's −1424/−1440) reverses the normalized (center − spawn) vector so velocity points AWAY from center; particles spawn on the tiny (4, 2, 4) ellipsoid around anchor + (0, −24, 0) and burst outward before the positive weight plunges them — an "impact splash" trajectory, in contrast to handler 13's strong anti-gravity rising fountain arcs.** — `[S·R] 2/3`
  - S: FUN_801a7f5c SCATTER branch and dispatch at ram 0x801a83c4–0x801a83d4, `research/working_documents/handler_21_particle_system.md` §3/§4
  - R: `godot-learning/src/effects/TrapEffect.gd` SCATTER branch (`p.velocity = -ellipsoid_offset.normalized() * speed` with speed drawn from the negative radius fields) + `godot-learning/assets/effects/trap/emitters.json` index 11 `radius [-2384, -2368]`
  - src: `research/working_documents/handler_21_particle_system.md`
- **Handler 21 is a 3-state machine: state 1 (LAB_801b414c) zeroes active_particle_count/frame_counter/fade_counter and moves to state 2 with no sound effect (unlike handler 12's SFX 0x22), no config modification (unlike handler 13's animated pos_scatter_y), and no camera offset (unlike handler 9); state 2 (LAB_801b4164) spawns via FUN_801b0cf0(config 11, clut_param 0x0A) then falls through to state 3; state 3 (LAB_801b4170) runs only the shared physics pipeline (FUN_801a9a24, set_inertia_threshold(0x230) = 560, gravity {0, 0x1000, 0}, FUN_801b0f08(0xb), FUN_801a9a3c, FUN_801a99f0), with the 2→3 transition set externally when the hit reaction ends.** — `[S] 1/3`
  - S: state dispatch 0x801b40f8–0x801b4148 and state entries 0x801b414c / 0x801b4164 / 0x801b4170, `research/working_documents/handler_21_particle_system.md` §5–§6
  - R: none — the 3-state phase machine not present in godot-learning (probed `godot-learning/src/effects/TrapEffect.gd` + `tests/`; the Godot TrapEffect runs its own per-emitter lifecycle instead of mirroring the ROM slot states)
  - src: `research/working_documents/handler_21_particle_system.md`
- **Termination (0x801b41b4–0x801b421c) uses the same fade_counter XOR-1 trick as handlers 8/9/12/13/19: the return value is computed BEFORE the update (fade_counter 0 → ACTIVE, 1 → DONE) and fade_counter increments only when active_particle_count == 0 and frame_counter > spawn_check_lo (DAT_801b8760 = config 11 +0x02 = 0), giving a 1-frame delay between the last particle dying and the handler signaling DONE so the dispatch loop deallocates the slot.** — `[S] 1/3`
  - S: termination logic 0x801b41b4–0x801b421c and DAT_801b8760, `research/working_documents/handler_21_particle_system.md` §7
  - R: none — fade_counter XOR termination not present in godot-learning (probed `godot-learning/src/` + `tests/`; `TrapEffect.gd` frees itself on its own all-particles-dead + palette-done check instead)
  - src: `research/working_documents/handler_21_particle_system.md`

## Notes

(empty — user territory)

## Related

- [[TRAP Hit Effect Particle System]]
- [[Hit Reaction Particle Burst]]
- [[TRAP Charge Particle System]]
- [[TRAP Sprite Effect System]]
- [[Spell Charge Effect System]]
