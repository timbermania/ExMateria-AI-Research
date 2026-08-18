# TRAP Charge Particle System

FFT's in-memory TRAP charge-effect particle system renders VFX for abilities that have no E###.BIN file: the charge VFX init in FUN_801a15b4 routes Charge+X abilities (IDs 406–413) through DAT_801b84dc to g_charge_effect_handlers[6] — handler 6 at 0x801b284c, a 3-state machine that spawns SCATTER-mode particles from the 46-byte config 10 at 0x801b8730 (fixed inward speed 1840, spawn point 18 units above the caster, downward gravity 0x1000) and runs the shared TRAP physics pipeline every tick until a fade_counter XOR-1 termination trick signals DONE. This note is a disassembly-derived reference for handler 6 (no decompilation exists) covering routing, slot layout, config fields, spawn checks, CLUT computation, SCATTER velocity, the per-tick physics pipeline, lifetime management, and termination.

## Points

- **Charge+X abilities (IDs 406–413, hex 0x196–0x19D) have no E###.BIN; the charge VFX init in FUN_801a15b4 takes the `(param_2 − 0x196) < 8` branch and calls allocate_sprite_animation_slot(0x11, 0, pos_data), where DAT_801b84dc[0x11×4] (0x801b8520) yields func_id 6 and g_charge_effect_handlers[6] (0x801b8918) resolves to handler 6 at 0x801b284c, which the process_charge_effects dispatch loop at 0x801b47e4 calls once per game tick.** — `[S] 1/3`
  - S: FUN_801a15b4, DAT_801b84dc[0x11] (0x801b8520), g_charge_effect_handlers[6] (0x801b8918), handler 6 at 0x801b284c, dispatch loop 0x801b47e4 (disassembly), per `research/working_documents/charge_x_particle_system.md`
  - src: `research/working_documents/charge_x_particle_system.md`
- **The 54-byte charge effect slot used by handler 6 holds phase_state at +0x08 (1=init, 2=spawning, 3=wind-down), frame_counter at +0x0C, active_particle_count at +0x2C, fade_counter at +0x2E, and the particle_ids array (pool indices, up to max_particles = 16) at +0x30; state 1 is set externally at slot allocation and state 3 is set externally by the charge effect system when the charge animation ends.** — `[S] 1/3`
  - S: slot field offsets read in FUN_801b284c (0x801b284c–0x801b2964), per `research/working_documents/charge_x_particle_system.md` (layout documented in SPELL_CHARGE_EFFECT_SYSTEM.md)
  - src: `research/working_documents/charge_x_particle_system.md`
- **Config 10 at 0x801b8730 (46 bytes, one of 17 configs in the table at 0x801b8564) configures handler 6's particles: frame_table_index 8, spawn window [0, 9), max_particles 16, direction_flags 0x0400 (DIRECTIONAL — spawn positions rotated by the caster's facing), velocity_mode 0x0010 (SCATTER), pos_scatter (0, −18, 0), scatter_half (0, 0, 0), radius_min = radius_max = 1840 (fixed inward speed), spawn_rate 2, spawn_count 1, lifetime −1 (animation-driven).** — `[S] 1/3`
  - S: config 10 raw bytes at 0x801b8730 (hex dump) and config table base 0x801b8564, per `research/working_documents/charge_x_particle_system.md`
  - src: `research/working_documents/charge_x_particle_system.md`
- **Handler 6 (FUN_801b284c) is a 3-state machine: state 1 (0x801b2898) zeroes the counters, plays sound effect 8, and sets phase_state to 2; state 2 (0x801b28b4) calls FUN_801b0cf0(config 10, clut_param 0x0b) to spawn and then falls through into state 3's physics pipeline; state 3 (0x801b28c0) runs only the physics pipeline with no further spawning.** — `[S] 1/3`
  - S: state dispatch 0x801b284c–0x801b2894 and state entries 0x801b2898 / 0x801b28b4 / 0x801b28c0 (disassembly), per `research/working_documents/charge_x_particle_system.md`
  - src: `research/working_documents/charge_x_particle_system.md`
- **FUN_801b0cf0 gates each spawn attempt with a frame-window check (frame_counter within [spawn_check_lo, spawn_check_hi) — frames 0–8 for config 10, so spawning stops after frame 8) and a particle cap (active_particle_count < max_particles = 16); with spawn_rate 2 and spawn_count 1 this yields roughly 4–5 particles over the 9-frame window, well under the 16 cap.** — `[S] 1/3`
  - S: spawn checks at 0x801b0d48 / 0x801b0d68 / 0x801b0ebc inside FUN_801b0cf0 (0x801b0cf0–0x801b0f04), per `research/working_documents/charge_x_particle_system.md`
  - src: `research/working_documents/charge_x_particle_system.md`
- **The particle CLUT is computed as clut_param + 0x7AC0 (0x0B → 0x7ACB for config 10) with VRAM position clut_x = (CLUT & 0x3F) << 4 = 176 and clut_y = CLUT >> 6 = 491; FUN_801b0c88 writes the CLUT at particle+0x88, neutral RGB (0x80, 0x80, 0x80) at +0x84–0x86, and the animation pointer from DAT_801b8320[frame_table_index × 2] at +0x90.** — `[S] 1/3`
  - S: CLUT computation in `research/working_documents/charge_x_particle_system.md`, FUN_801b0c88 (0x801b0c88–0x801b0cec), frame table DAT_801b8320
  - src: `research/working_documents/charge_x_particle_system.md`
- **In SCATTER mode FUN_801a7f5c aims each particle at the anchor (caster position): spawn = anchor + pos_scatter — fixed at (0, −18, 0), i.e. 18 units above the caster in PSX Y-down coordinates with no random scatter — and velocity = normalized(anchor − spawn) × 1840, which for this config is always straight down (0, +18, 0); the particle is written with position at +0x9C–0xA4 (12.12 fixed point), velocity at +0xA8–0xB0 (×2³), hardcoded inertia 0x1000 at +0x98, and weight 0 at +0x9A.** — `[S] 1/3`
  - S: FUN_801a7f5c (SCATTER physics init; full walkthrough in TRAP_PARTICLE_SYSTEM_DEEP_DIVE.md §17 as referenced), per `research/working_documents/charge_x_particle_system.md`
  - src: `research/working_documents/charge_x_particle_system.md`
- **Every tick in states 2 and 3 the handler runs the shared TRAP physics pipeline: FUN_801a9a24 (frame prepare), set_inertia_threshold(0x230) = 560 (velocity components below the threshold decay by (4096−560)/4096 ≈ 0.863× per tick), gravity {0, 0x1000, 0} (downward) applied via set_gravity_vector, FUN_801b0f08(10) (particle iteration), FUN_801a9a3c (integration step), FUN_801a99f0 (render).** — `[S] 1/3`
  - S: pipeline call sites 0x801b28c0–0x801b2900 (gravity constant at 0x801b28d0), set_inertia_threshold 0x801a69ac, set_gravity_vector 0x801a6984, per `research/working_documents/charge_x_particle_system.md`
  - src: `research/working_documents/charge_x_particle_system.md`
- **FUN_801b0f08 iterates the slot's 16 particle_ids every tick calling FUN_801aa7dc(pid) (physics + animation + render); when it returns 0 — the animation completed, since lifetime −1 means animation-driven — the particle is freed via FUN_801adc24, the slot entry is zeroed, and active_particle_count is decremented.** — `[S] 1/3`
  - S: FUN_801b0f08 (0x801b0f08–0x801b0fe8), FUN_801aa7dc, FUN_801adc24, per `research/working_documents/charge_x_particle_system.md`
  - src: `research/working_documents/charge_x_particle_system.md`
- **Handler 6's termination uses a `fade_counter XOR 1` return value computed before any updates (fade_counter 0 → ACTIVE, 1 → DONE), and fade_counter is incremented only when active_particle_count == 0 and frame_counter > DAT_801b8732 (spawn_check_lo = 0) — so the DONE signal that deallocates the slot in the dispatch loop arrives exactly one frame after the last particle dies.** — `[S] 1/3`
  - S: termination logic 0x801b2904–0x801b2950 (xori at 0x801b2918, DAT_801b8732 load at 0x801b293c), per `research/working_documents/charge_x_particle_system.md`
  - src: `research/working_documents/charge_x_particle_system.md`
- **Config 12 (handler 4, FUN_801b1c04 — spell charge sparkle implosion) also uses SCATTER mode but differs from config 10: max 10 particles, direction_flags 0x0000 (none), pos_scatter (48, 48, 48) instead of (0, −18, 0), randomized radius 1872–2112 instead of fixed 1840, CLUT 0x7ACF instead of 0x7ACB, spawn window [0, 8) instead of [0, 9), and animation sequence 5 instead of 8.** — `[S] 1/3`
  - S: handler 4 FUN_801b1c04 and config table 0x801b8564 (config 10 at 0x801b8730), per `research/working_documents/charge_x_particle_system.md`
  - src: `research/working_documents/charge_x_particle_system.md`

## Notes

(empty — user territory)

## Related

- [[Particle Emitter Format]]
- [[Particle Runtime State]]
- [[Ability Execution State Flow]]
- [[Animation Event System]]
