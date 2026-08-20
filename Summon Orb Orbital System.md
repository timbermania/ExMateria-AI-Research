# Summon Orb Orbital System

Handler 22 (FUN_801b4234, 0x801b4234–0x801b47d8) renders the summon charge orbs: 30 particles in 3 rings × 10 slots orbit the caster as comet trails in a flat XZ plane 24 units above the caster origin, driven by a 3-state machine (init → 24-tick ramp-up orbit → 61-tick expand/accelerate/fade) in the 0x54-byte charge-effect slot, with private orbital state (ring buffer at 0x801badfc) in the global block at 0x801bade0 — a block the handler 4 and handler 17 docs also claim as their own storage (CONTESTED). Steady state is radius 23, 69 PSX angle units/tick (~6.06°/tick, ~1.98 s per rotation); the fade flares brightness_scale 128→240, spins to ~130/tick, and expands the radius to ~53 while brightness decays linearly to 0. Reimplemented in godot-learning as `TrapOrbitalEffect` (spawned by charge pose 2 in `EffectManager.gd`), which mirrors the state machine, ring-buffer trail, and brightness weight table but tracks caster movement (the PSX anchor is fixed).

## Points

- **Summon charge (secondary anim 0x02, effect type 0x05) routes to handler 22: DAT_801b84dc[0x05×4] (0x801b84f0) holds func_id 22, g_charge_effect_handlers[22] at 0x801b8958 points to FUN_801b4234 (0x801b4234–0x801b47d8), and the process_charge_effects dispatch loop calls the handler once per game tick until it returns 0.** — `[S·R] 2/3`
  - S: routing chain per doc §1/§13 — DAT_801b84ac[0x02] = 0x05 (0x801b84ae), DAT_801b84dc[0x05×4] → func_id 22 (0x801b84f0), g_charge_effect_handlers[22] at 0x801b8900 + 22×4 = 0x801b8958 = 0x801b4234 (BATTLE.BIN disassembly)
  - R: `godot-learning/src/effects/EffectManager.gd` `CHARGE_POSE_TO_HANDLER` pose 2 → `_spawn_charge_orbital` (TrapOrbitalEffect, comments "handler 22" / "Orbital Summon Orbs"); no orbital-specific test named (GPUBreakTrapTest.gd only counts TrapEffect nodes)
  - src: `research/working_documents/summon_orb_orbital_system.md`
- **Handler 22 keeps all orbital state in a contiguous global block at 0x801bade0 (loaded into s4): ring_write_index i32 +0x00 (0–9, wraps), orbital_radius i32 +0x04, accumulated_angle i32 +0x08, angular_velocity i32 +0x0C, anchor_x/y/z i16 +0x10/+0x12/+0x14, brightness_scale i32 +0x18, and a 3×10 ring buffer of 8-byte cells (x/y/z i16 + pad, y always 0) from +0x1C (0x801badfc), ring stride 0x50, cell stride 8, ring bases 0x801badfc / 0x801bae4c / 0x801bae9c.** — `[S·R] 2/3 CONTESTED`
  - S: block layout per doc §2, cell writes `sh v0, 0x1c(a0)` / `sh v0, 0x20(a0)` at 0x801b452c / 0x801b456c (BATTLE.BIN disassembly)
  - R: `godot-learning/src/effects/TrapOrbitalEffect.gd` mirrors the per-ring ring buffer and write head (RING_COUNT 3, SLOTS_PER_RING 10, `_write_head` advance mod 10); no orbital-specific test named
  - src: `research/working_documents/summon_orb_orbital_system.md`
- **Handler 22 runs a 3-state machine in the 0x54-byte charge-effect slot: phase_state +0x08 (1=init, 2=orbit, 3=fade), frame_counter +0x0C, fade_counter +0x2E, particle_ids +0x30 (30 u8 pool indices: +0x30–0x39 ring 0, +0x3A–0x43 ring 1, +0x44–0x4D ring 2); state 1 is set at slot allocation, 1→2 by the handler, and 3 is set externally when the charge animation ends.** — `[S·R] 2/3`
  - S: handler 22 state logic 0x801b4234–0x801b47d8 and slot offsets per doc §1/§9 (slot layout documented in SPELL_CHARGE_EFFECT_SYSTEM.md)
  - R: `godot-learning/src/effects/TrapOrbitalEffect.gd` `enum State { INIT, ORBIT, FADE, DONE }` + `start()` / `start_fade()` / done-cleanup; no orbital-specific test named
  - src: `research/working_documents/summon_orb_orbital_system.md`
- **State 1 (init): zeroes the orbital globals and clears all 30 ring-buffer cells; allocates 30 particles from pool 0x21 via FUN_801adb3c (each particle 0xCC bytes at DAT_801b9278 + id×0xCC), initializes each from config 14 via FUN_801a7f5c and sets sprite animation + CLUT via FUN_801b0c88(14, 0x7ACC, render_ptr); the anchor is get_anchor_position(caster) with Y − 24 (`addiu ... −0x18`), fixed for the whole effect — it does NOT track unit movement; a sound is played on init.** — `[S·R] 2/3`
  - S: init entry 0x801b42d0, config-14 address load 0x801b437c, FUN_801b0c88 call 0x801b438c (`ori a0, zero, 0x0E` / 0x7ACC), FUN_801adb3c(0x21) allocation, anchor shift per doc §3/§8 (BATTLE.BIN disassembly)
  - R: `godot-learning/src/effects/TrapOrbitalEffect.gd` `start()` mirrors the 24-unit anchor shift (ANCHOR_Y_OFFSET = 24.0/28.0) and ring-buffer clear, but re-reads the target unit's global position every tick (tracks movement) — diverges from the fixed PSX anchor
  - src: `research/working_documents/summon_orb_orbital_system.md`
- **State 2 (orbit) ramps up: while frame_counter < 24, angular_velocity = frame×3 and orbital_radius = frame; while frame_counter < 8, brightness_scale = (frame+1)×16; steady state (from tick 24): radius 23, 69 PSX angle units/tick (~6.06°/tick, ~1.98 s per full rotation at 30 ticks/s), brightness 128.** — `[S·R] 2/3`
  - S: 0x801b4270 (slti v0,v0,0x18), 0x801b4294 (slti v0,v0,8), 0x801b42a0 (frame_counter cap 256), per doc §3/§12 (BATTLE.BIN disassembly)
  - R: `godot-learning/src/effects/TrapOrbitalEffect.gd` `_tick_orbit()` (RAMP_UP_FRAMES = 24, identical ×3 angular-velocity / ×1 brightness ramps, (frame+1)×16 brightness for frame < 8); no orbital-specific test named
  - src: `research/working_documents/summon_orb_orbital_system.md`
- **State 3 (fade), set externally when the charge ends: until fade_counter > 60 — orbital_radius += 1 only in the fade-counter windows where bit 3 is set (& 8: 8–15, 24–31, 40–47, 56–60), angular_velocity += 1 every tick (69 → ~130, ~11.4°/tick at the end), brightness_scale = (60 − fade_counter)×4, which spikes 128 → 240 at the state 2→3 transition (brief flare) and decays linearly to 0; over the 61 ticks radius grows ~23 → ~53; on completion the handler frees all 30 particles and returns 0, deactivating the slot in the dispatch loop.** — `[S·R] 2/3`
  - S: fade logic per doc §3 (0x3D = 61-tick cap, &8 expansion mask, (60−fc)×4 brightness), disassembly 0x801b4234–0x801b47d8
  - R: `godot-learning/src/effects/TrapOrbitalEffect.gd` `_tick_fade()` (FADE_DURATION = 61, FADE_EXPANSION_MASK = 8, FADE_BRIGHTNESS_BASE = 60, stop + animation_finished + queue_free on done); no orbital-specific test named
  - src: `research/working_documents/summon_orb_orbital_system.md`
- **Orbital position computation: PSX 12-bit fixed-point angles (4096 = full circle, rcos/rsin use only the low 12 bits so the 32-bit accumulated_angle wraps naturally); per ring, angle = accumulated_angle + ring×0x555 (120° spacing), x = rcos(angle)×orbital_radius and z = rsin(angle)×orbital_radius, each += 0xFFF when negative then >>12 (truncation toward zero); y is never written (stays 0) — a flat orbit in the XZ plane with the three rings evenly spaced.** — `[S·R] 2/3`
  - S: 0x801b44d4–0x801b4580 — jal rcos 0x801b44f4, jal rsin 0x801b4534, sra v0,v1,0xc at 0x801b4528, ring phase addiu s0,s0,0x555 at 0x801b4580, angle advance 0x801b44e8, per doc §4 (BATTLE.BIN disassembly)
  - R: `godot-learning/src/effects/TrapOrbitalEffect.gd` `_compute_orbital_positions()` (RING_PHASE_OFFSET = 1365, TrapConstants.FULL_CIRCLE_PSX = 4096, cos/sin × radius × PSX_SCALE, Y fixed 0.0); no orbital-specific test named
  - src: `research/working_documents/summon_orb_orbital_system.md`
- **The ring buffer creates a comet trail: each ring's particle slot 0 reads the just-written head (age 0 ticks), each following slot reads one temporal slot further with mod-10 wrap — slot 9 is 1 tick behind the head, slot 1 is the tail end (9 ticks old); at the steady 69 units/tick the trail spans ~54.6° of arc (9 ticks × 69/4096 × 360°).** — `[S·R] 2/3`
  - S: ring-buffer read loop 0x801b4584–0x801b46dc and read-index advance, per doc §5.1–5.2/§9 (BATTLE.BIN disassembly)
  - R: `godot-learning/src/effects/TrapOrbitalEffect.gd` `_update_particle_overrides()` (read_idx starts at the most recent write, +1 mod 10 per slot); no orbital-specific test named
  - src: `research/working_documents/summon_orb_orbital_system.md`
- **World positions are written into the particle physics struct at DAT_801b9278 + pid×0xCC as 12.12 fixed point at +0x9C (x) / +0xA0 (y) / +0xA4 (z) = (anchor + ring-cell offset) × 0x1000, then FUN_801aa7dc(pid) runs physics + animation + render; because config 14 has zero initial velocity, zero weight, and zero acceleration, the physics integration adds nothing (new_pos = handler_pos + 0), so the handler-computed orbital position is the sole source of particle motion for summon orbs.** — `[S·R] 2/3`
  - S: position stores 0x801b469c / 0x801b46b4 / 0x801b46cc, particle base DAT_801b9278 + pid×0xCC, FUN_801aa7dc interaction per doc §5.3–5.4 (BATTLE.BIN disassembly)
  - R: `godot-learning/src/effects/TrapEffect.gd` override mode (`enable_override_mode` / `set_particle_override` — positions and brightness set externally each tick, handler-22 branch); no orbital-specific test named
  - src: `research/working_documents/summon_orb_orbital_system.md`
- **Per-particle brightness = weight[slot] × brightness_scale / 128 (round-toward-zero: raw += 0x7F if negative, then >> 7), written as monochrome R=G=B to particle +0x84/+0x85/+0x86; the 10-byte weight table at DAT_801b88f4 is [0x80, 0x02, 0x06, 0x08, 0x0A, 0x0C, 0x10, 0x14, 0x18, 0x1C] (128, 2, 6, 8, 10, 12, 16, 20, 24, 28) — head slot fully bright (128 = PSX neutral 1.0×), tail near-invisible (2 ≈ 0.02×); during the fade the scale spikes to 240 (head ~1.9× overbright) and decays to 0.** — `[S·R] 2/3`
  - S: DAT_801b88f4 (10 bytes, BATTLE.BIN), table load at 0x801b45c4 (`lbu v1, -0x770c(at)`), brightness formula at 0x801b45d4, per doc §6 (BATTLE.BIN disassembly + byte values)
  - R: `godot-learning/src/effects/TrapOrbitalEffect.gd` `BRIGHTNESS_WEIGHTS = [128, 2, 6, 8, 10, 12, 16, 20, 24, 28]`, color_mod = weight × brightness_scale / 16384 (equivalent neutral-1.0 mapping) pushed via TrapEffect override; no orbital-specific test named
  - src: `research/working_documents/summon_orb_orbital_system.md`
- **Config 14 at 0x801b87E8 (46 bytes, entry 14 of the 17-config table at 0x801b8564) configures the summon orbs: frame_table_index 9, spawn window [0, 8], max_particles 30 (exactly 3 rings × 10), direction_flags 0x0400 (DIRECTIONAL), velocity_mode 0x0000 (SPHERICAL_RANDOM fallthrough), all velocity/scatter/weight/radius fields zero (no physics drift — the handler's orbital computation is the sole motion source), spawn_rate 1, spawn_count 1, lifetime −1 (animation-driven; the handler frees the particles itself when fade_counter > 60).** — `[S·R] 2/3`
  - S: config 14 raw bytes at 0x801b87E8 (hex 09 00 00 08 1e 00 00 04 … 01 01 ff ff) and config table base 0x801b8564, per doc §7 (BATTLE.BIN dump via parse_trap_effect.py)
  - R: `godot-learning/assets/effects/trap/emitters.json` emitter 14 ("pulsing_pair", `raw.config_addr: 0x801B87E8` with matching raw hex, anim_index 9, max_particles 30, zero velocity, weight 0, lifetime −1, spawn_window [0, 8]) + `godot-learning/src/effects/TrapEffect.gd` `HANDLER_CONFIGS[22] = [14]`; no orbital-specific test named
  - src: `research/working_documents/summon_orb_orbital_system.md`
- **The orbs use sprite animation sequence 9: a 2-frame pulsing pair with a 4-tick period — frameset 49 for 2 ticks, frameset 48 for 2 ticks, repeating (twinkle); CLUT 0x7ACC decodes to VRAM (192, 491) via clut_x = (CLUT & 0x3F) << 4, clut_y = CLUT >> 6; the frameset 49 sprite is a 16×16 texture at VRAM UV (104, 120) rendered as a 5×5 world-unit quad (−2,−2) to (3,3) with additive blend (PSX semi-trans mode 1), 4bpp TPAGE x_base 6.** — `[S·R] 2/3`
  - S: sequence 9 / frameset 49 data parsed per doc §7 (parse_trap_effect.py), CLUT argument at 0x801b438c (FUN_801b0c88(0x0E, 0x7ACC, …)), per doc (BATTLE.BIN disassembly + parser output)
  - R: `godot-learning/assets/effects/trap/emitters.json` emitter 14 `anim_index: 9` ("pulsing_pair") + `godot-learning/src/effects/TrapEffect.gd` `HANDLER_PALETTE_OVERRIDES[22] = 12`; no orbital-specific test named
  - src: `research/working_documents/summon_orb_orbital_system.md`
- **Real-world scale: one map tile ≈ 28 world units wide, so the steady-state orbit (radius 23, diameter 46) spans ≈ 1.64 tiles.** — `[S·R] 2/3`
  - S: derived from the state-2 steady-state constants (radius cap 23 at 0x801b4270) and the PSX angle system, per doc §4.5/§11
  - R: `godot-learning/src/effects/TrapConstants.gd` `PSX_SCALE = 1.0 / 28.0` (tile = 28 world units); no orbital-specific test named
  - src: `research/working_documents/summon_orb_orbital_system.md`

## Notes

(empty — user territory)

## Related

- [[Spell Charge Effect System]]
- [[TRAP Charge Particle System]]
- [[Spell Charge Lines System]]
- [[Summon Charge Lines System]]
- [[Particle Runtime State]]
- [[Particle Coloring System]]
- [[Effect System Index]]
