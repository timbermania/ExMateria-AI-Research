# Particle Runtime State

The runtime side of FFT's particle system: the doubly-linked Particle instances (12.12 fixed-point positions/velocities, inertia/weight/drag physics, lifetime and homing behaviour, one ParticleAnimState each) that emitters spawn, the 36-byte ParticleAnimState render/animation state, and the global physics, animation-state-pool, and pipeline addresses that update and render them every frame. The emitter CAMERA anchor (0x0A) resolves to the map center — get_camera_position 0x8008DF48 (map dimensions ×14) plus the emitter offset — which is what gives summon effects their screen-relative positioning.

## Points

- **A ParticleAnimState is a 36-byte (0x24) render/animation state referenced by `Particle.anim_state` (offset 0x54), allocated from a pool whose slots are linked through pool_prev_index (0x00) and pool_next_index (0x02) and managed by allocate/cleanup_animation_state.** — `[S] 1/3`
  - S: ParticleAnimState struct and pool-link fields, per `research/key_documents/STRUCTURE_DEFINITIONS.md`
  - src: `research/key_documents/STRUCTURE_DEFINITIONS.md`
- **ParticleAnimState render state: flags 0x06 (bit 0 = sprite definition changed, bits 1–2 render mode 0=3D GTE / 2=2D screen / 4=3D GTE / 6=depth-sorted 2D), sprite_offset_x/y 0x08/0x0A, render_position_x/y/z 0x0C/0x0E/0x10 (copied from the particle's 12.12 position via >>12 each frame), screen_rotation_angle 0x12 (billboard rotation within the screen plane, not 3D — 0x000=0°, 0x400=90°, 0x800=180°), and depth_mode 0x14 (0=Z>>2, 1=Z>>2−8, 2=fixed 8, 3=fixed 0x17E, 4=fixed 0x10, 5=Z>>2−0x10).** — `[S] 1/3`
  - S: ParticleAnimState field offsets 0x06–0x14, per `research/key_documents/STRUCTURE_DEFINITIONS.md`
  - src: `research/key_documents/STRUCTURE_DEFINITIONS.md`
- **ParticleAnimState animation playback: frame_timer 0x16 (decremented by 2 each render call, advances the sequence when ≤0), sequence_data_ptr 0x18 (pointer into E###.BIN animation sequence data), frame_counter 0x1C (current byte offset in the sequence), sprite_variant_offset 0x1E (×2 into the variant table), sprite_frame_index 0x1F, and sprite_data_ptr 0x20 (GPU sprite primitive — RGB at bytes 0–2, quad pointers at +8).** — `[S] 1/3`
  - S: ParticleAnimState field offsets 0x16–0x20, per `research/key_documents/STRUCTURE_DEFINITIONS.md`
  - src: `research/key_documents/STRUCTURE_DEFINITIONS.md`
- **A Particle is a runtime instance stored in a doubly-linked list (prev 0x00, next 0x04, NULL = list end), updated every frame by integrate_particle_motion and update_all_particles, with position (0x0C–0x14), velocity (0x18–0x20), and acceleration (0x24–0x2C) as 12.12 fixed-point int32s.** — `[S] 1/3`
  - S: Particle struct and 12.12 fixed-point layout, per `research/key_documents/STRUCTURE_DEFINITIONS.md`
  - src: `research/key_documents/STRUCTURE_DEFINITIONS.md`
- **Particle physics parameters: inertia 0x08 with formula `velocity = ((inertia − 560) × velocity + accel) / inertia` (higher inertia resists both slowing and speeding up), weight 0x0A with `velocity_y += weight` (0 = floats, 10 = falls 10 units/frame), drag coefficients 0x30–0x38, and homing target_x/y/z 0x3C–0x40 (int16, computed at spawn from anchor + offset).** — `[S] 1/3`
  - S: Particle physics fields and inertia/weight formulas, per `research/key_documents/STRUCTURE_DEFINITIONS.md`
  - src: `research/key_documents/STRUCTURE_DEFINITIONS.md`
- **Particle lifetime and behaviour: lifetime_counter 0x42 (decremented each frame; 0xFFFF = die by animation duration instead), homing_curve_index 0x45, color curve indices 0x46–0x48 (lookup 0x80 = normal, <0x80 = darker, >0x80 = brighter), homing_strength 0x4A (0 = simple physics, !=0 = enable homing — NOT a countdown), motion_flags 0x4C (copied from emitter bytes 0x02–0x03; bit 1 align_sprite_to_screen_velocity, bits 12–15 handler_index), behavior_flags 0x4E (copied from emitter bytes 0x06–0x07), anim_frame_counter 0x50 (wraps at 160), child_emitter_on_death/mid_life 0x52/0x53, and anim_state 0x54.** — `[S] 1/3`
  - S: Particle field offsets 0x42–0x54, per `research/key_documents/STRUCTURE_DEFINITIONS.md`
  - src: `research/key_documents/STRUCTURE_DEFINITIONS.md`
- **align_sprite_to_velocity (emitter byte 0x02 bit 1) is computed at runtime in update_particle_render_state (0x801A5EA4): the velocity is projected to screen space via rotate_vector and `screen_rotation = ratan2(screen_velocity_y, screen_velocity_x)` is stored in anim_state->screen_rotation_angle, so sprite orientation (sparks/debris aligning along their flight path) is controlled by the particle's velocity.** — `[S] 1/3`
  - S: align_sprite_to_velocity runtime code at 0x801A5EA4 (rotate_vector + ratan2), per `research/key_documents/STRUCTURE_DEFINITIONS.md`
  - src: `research/key_documents/STRUCTURE_DEFINITIONS.md`
- **The particle/physics globals are: gravity vector at 0x801B8A40–0x801B8A48 (typically (0, 0x1000, 0)), inertia_threshold at 0x801B8A4C (typically 560); the loader chain is opcode 39 → op_init_physics_params (0x801A3148) with set_gravity_vector (0x801A9984) and set_inertia_threshold (0x801A99AC) reading the ParticleSystemHeader values.** — `[S] 1/3`
  - S: physics globals 0x801B8A40–0x801B8A4C and loader functions 0x801A3148/0x801A9984/0x801A99AC, per `research/key_documents/STRUCTURE_DEFINITIONS.md`
  - src: `research/key_documents/STRUCTURE_DEFINITIONS.md`
- **The particle render/cleanup stage is: deallocate_particle (0x801A5D74) frees a particle, update_particle_render_state (0x801A5EA4) copies the physics position into the anim_state render_position, and render_particle_sprite (0x801AA1F8) processes the animation sequence, transforms the coordinates, and submits the sprite to the GPU.** — `[S] 1/3`
  - S: pipeline functions 0x801A5D74/0x801A5EA4/0x801AA1F8 and render-pipeline description, per `research/key_documents/STRUCTURE_DEFINITIONS.md`
  - src: `research/key_documents/STRUCTURE_DEFINITIONS.md`
- **The animation-state pool is at base 0x801C00A4, with the free-list head at 0x801BF00A and the allocated-list head at 0x801C24DA; slots are linked through the ParticleAnimState pool_prev_index/pool_next_index fields.** — `[S] 1/3`
  - S: pool addresses 0x801C00A4/0x801BF00A/0x801C24DA, per `research/key_documents/STRUCTURE_DEFINITIONS.md`
  - src: `research/key_documents/STRUCTURE_DEFINITIONS.md`
- **The particle system's CAMERA anchor (emitter anchor 0x0A) resolves to the map center, not to a unit: it calls get_camera_position 0x8008DF48 (map width DAT_800E4E9C / depth DAT_800E4EA0, single bytes), scales X and Z by ×14 (midpoint of the N×28 world span), and adds the emitter offset; final Y is the emitter offset only (camera Y unused). Same function and ×14 scaling as the camera EFFECT_CTR source, so EFFECT_CTR + CAMERA anchor both anchor to the map center — screen-relative summon positioning (Shiva E065: 13 of 16 emitters CAMERA-anchored, main sprite at map_center + [192, -256, 0]).** — `[S] 1/3`
  - S: disassembly at 0x801A65A4 (JAL 0x8008DF48, sll/subu/sll ×14 sequence) and E065.BIN emitter table, per `research/working_documents/CAMERA_SYSTEM.md`
  - src: `research/working_documents/CAMERA_SYSTEM.md`

## Notes

(empty — user territory)

## Related

- [[Particle Emitter Format]]
- [[Effect Execution Model]]
- [[Effect Camera System]]
