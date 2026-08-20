# Projectile Arc System

Handler 20 (FUN_801b3938, 0x801b3938–0x801b40f4) is FFT's custom two-point linear trajectory renderer for projectile weapons — crossbow bolts, thrown stones, and ninja throwing weapons. It is not a config-driven particle system: triggered by SEQ opcode 0xD9 through anim_types 0x03/0x06/0x10, it interpolates a position along the caster→target vector every frame via FUN_801af6d8 (pure linear — no parabolic arc call) and draws a variant-specific 3D wireframe model (LineF2 primitives + sparkle sprites, or a sprite sub-effect for most ninja weapons) through render_charge_effect (0x801ae340). The handler runs a 3-state machine on slot+0x08, fires the impact sound two frames before arrival with ability-ID exclusion ranges, and terminates on frame-countdown 0 with state 3 (return 0) so the dispatcher removes the slot. godot-learning mirrors the straight-line flight, the stone tumble rates, and the crossbow/stay-straight vs bow/arcs distinction in its 3D projectile system.

## Points

- **Handler 20 (FUN_801b3938, 0x801b3938–0x801b40f4, 0x7BC bytes, 0xa0-byte stack frame) is a custom two-point trajectory renderer for crossbow bolts, thrown stones, and ninja throwing weapons — no config table, no spawner loop, no physics pipeline: it interpolates a position along the caster→target vector each frame and calls render_charge_effect (0x801ae340) to draw a 3D wireframe model, in contrast to the config-driven particle handlers (8, 9, 12, 13).** — `[S·R] 2/3`
  - S: 0x801b3938–0x801b40f4, render_charge_effect 0x801ae340, per research/working_documents/handler_20_projectile_arc_system.md
  - R: godot-learning/src/projectiles/ProjectileManager.gd + Projectile3D.gd (per-frame flight along the caster→target line with Godot 3D meshes, not PSX LineF2) + tests/GPUThrowStoneTest.gd (ThrowStone ID 148 ranged flight)
  - src: `research/working_documents/handler_20_projectile_arc_system.md`
- **Trigger pipeline: the attacker's SEQ reaches 0xD9 → FUN_800689a4 (with position data from FUN_800687e0) routes the attack — ability 0x94 (Throw Stone) → anim_type 0x06, ability 0x17E (Ninja Throw) and abilities 0x170–0x189 → 0x10, all others → DAT_800943c4[weapon_type] (Crossbow 0x0B → 0x03) — then FUN_801ade7c allocates the charge slot; all three anim_types reach handler 20 via DAT_801b84dc records (0x03 @ 0x801b84e8, 0x06 @ 0x801b84f4, 0x10 @ 0x801b851c), and the signed halfword in aux bytes [2:3] (g_duration_param_table = 0x801b84de) selects the variant: 0x0800 = Arrow, 0x0801 = Stone, 0xFFFF (−1) = Special.** — `[S] 1/3`
  - S: FUN_800689a4 (0x800689a4), FUN_800687e0, FUN_801ade7c, DAT_800943c4, DAT_801b84dc records at 0x801b84e8/0x801b84f4/0x801b851c, per research/working_documents/handler_20_projectile_arc_system.md
  - R: none — 0xD9 / FUN_800689a4 anim_type routing not present in godot-learning (probed godot-learning/src, godot-learning/tests)
  - src: `research/working_documents/handler_20_projectile_arc_system.md`
- **Caster and target world positions are read from six halfword globals (g_caster_pos_x 0x801b925c, +y 0x801b925e, +z 0x801b9260; g_target_pos_x 0x801b9264, +y 0x801b9266, +z 0x801b9268), written before the first handler call by FUN_8008c468 (unit ID → world coordinates): tile coords at unit struct +0x47/+0x48, pos_x = pos_z = tile·28 + 14, pos_y from tile characteristics; targets without a unit ID (target_id == −1) are computed inline with the same formula.** — `[S] 1/3`
  - S: 0x801b925c–0x801b9268, FUN_8008c468 (0x8008c468), per research/working_documents/handler_20_projectile_arc_system.md
  - R: none — FUN_8008c468 tile·28+14 world-coordinate conversion not present in godot-learning (probed godot-learning/src, godot-learning/tests)
  - src: `research/working_documents/handler_20_projectile_arc_system.md`
- **Handler 20's trajectory is purely linear, not parabolic: it never calls evaluate_parabolic_arc (0x801af59c — all four of that function's XREFs, 0x801afc78/0x801b1260/0x801b128c/0x801b12b0, fall outside 0x801b3938–0x801b40f4), so the flight for all variants (arrow, stone, ninja) is straight-line interpolation via FUN_801af6d8, and the apparent arc is 3D perspective projection when caster and target are at different heights.** — `[S·R] 2/3`
  - S: FUN_801af6d8 (0x801af6d8), evaluate_parabolic_arc XREFs, per research/working_documents/handler_20_projectile_arc_system.md
  - R: godot-learning/src/projectiles/ProjectileManager.gd (crossbow item_type 11 keeps arc_height = 0.0 straight-line; ThrowStone and ninja throws also arc_multiplier 0.0; only bow item_type 12 computes an arc) + tests/GPUThrowStoneTest.gd
  - src: `research/working_documents/handler_20_projectile_arc_system.md`
- **Handler 20 runs a 3-state machine on slot+0x08: state 1 = init (LAB_801b39a0, runs once), state 2 = per-frame rendering (LAB_801b3d38, checked first at dispatch), state 3 = complete (LAB_801b40c4, returns 0 so the dispatcher removes the effect from the active linked list); any other state passes param_1 through unchanged.** — `[S] 1/3`
  - S: dispatch 0x801b3938–0x801b399c; labels LAB_801b39a0, LAB_801b3d38, LAB_801b40c4, LAB_801b40cc, per research/working_documents/handler_20_projectile_arc_system.md
  - R: none — handler slot state machine not present in godot-learning (probed godot-learning/src, godot-learning/tests)
  - src: `research/working_documents/handler_20_projectile_arc_system.md`
- **Init (state 1) computes the 3D distance as fixed_point_sqrt((Δx² + Δy² + Δz²) << 12) and writes the arc globals: total distance → DAT_801b8b40, progress accumulator 0 → DAT_801b8b44, step size 0x8000 → DAT_801b8b48, frame countdown → DAT_801b8b4c (loaded from g_effect_duration, 0x801bbf3c, set at slot allocation — NOT from the duration_param aux bytes), delta vector → DAT_801b8b50/8b54/8b58; it then frees the old buffer at slot+0x50, allocates the variant buffer, sets slot+0x08 = 2 (0x801b3a84), calls FUN_8008dee8(1), and returns 1.** — `[S] 1/3`
  - S: 0x801b39a0–0x801b3d34, fixed_point_sqrt 0x8001c268, per research/working_documents/handler_20_projectile_arc_system.md
  - R: none — DAT_801b8b40 arc-progress globals not present in godot-learning (probed godot-learning/src, godot-learning/tests)
  - src: `research/working_documents/handler_20_projectile_arc_system.md`
- **Variant buffer allocation in init keys off the duration_param low byte / −1: Arrow (0x00) allocates 0xC88 = 2 groups × (4 LineF2 @ 0x1C + 13 sparkle sprites @ 0x24) + 0x808 vertex/transform workspace for render_charge_effect; Stone (0x01) allocates 0x2A0 = 2 × 12 LineF2; Special (0xFFFF, ninja) allocates 0x380 = 2 × 16 LineF2 only when slot+0x24 (ability ID) is in {0x7A, 0x7B, 0x7C}, otherwise allocates no GPU buffer and instead takes a sub-effect slot via FUN_801adb3c, initializing a sprite struct at DAT_801b9278 + index·0xCC (medium gray 0x80, tile size 0x1E, primary entry duplicated to +0x5C); LineF2 comes from SetLineF2 (0x80023ce0) and sparkles from init_sparkle_particle (0x80023d30).** — `[S] 1/3`
  - S: 0x801b39a0–0x801b3d34, SetLineF2 0x80023ce0, init_sparkle_particle 0x80023d30, alloc 0x80044414, per research/working_documents/handler_20_projectile_arc_system.md
  - R: none — PSX GPU primitive buffer layout not present in godot-learning (probed godot-learning/src, godot-learning/tests; projectiles build Godot meshes instead)
  - src: `research/working_documents/handler_20_projectile_arc_system.md`
- **Per-frame math: FUN_801af6d8 (0x801af6d8–0x801af730) does 12-bit fixed-point linear interpolation with scale = (progress << 8) / (distance >> 4) and output[i] = (scale · delta[i]) >> 12; the flight direction orientation uses yaw = ratan2(−Δz, Δx) (negated Z for PSX handedness, 0x801b3ddc) and pitch = ratan2(Δy, horizontal distance) + 0x400 (90° in PSX angle units where 0x1000 = 360°, 0x801b3e14–0x801b3e24).** — `[S·R] 2/3`
  - S: FUN_801af6d8 0x801af6d8–0x801af730, ratan2 0x8001d8e8, per research/working_documents/handler_20_projectile_arc_system.md
  - R: godot-learning/src/projectiles/Projectile3D.gd (float progress t ∈ [0,1] advanced per frame along the start→end segment, with `_update_arrow_rotation` pointing the arrow along the velocity direction — float analogue of the linear interpolation, not a fixed-point port); no interpolation-specific test named (flight validated end-to-end by tests/GPUThrowStoneTest.gd)
  - src: `research/working_documents/handler_20_projectile_arc_system.md`
- **Variant per-frame rendering differs in color and rotation: Arrow is static dim gray RGB (0x40,0x40,0x40) with zero rotation (points along the trajectory); Stone is bright gray (0xC0,0xC0,0xC0) and tumbles as it flies — rotation_y = countdown·256, rotation_x = countdown·128, both shrinking each frame; Special LineF2 spins in reverse on X (rotation_x = −countdown·256) and renders with model_type 3; the Special sprite sub-path renders via FUN_801af108 (sprite renderer, 0x801af108).** — `[S·R] 2/3`
  - S: 0x801b3e30–0x801b40c0, per research/working_documents/handler_20_projectile_arc_system.md
  - R: godot-learning/src/projectiles/Projectile3D.gd (`STONE_TUMBLE_Y_RATE_PER_SEC = 256/4096·TAU·0.5·60` and `STONE_TUMBLE_X_RATE_PER_SEC = 128/4096·TAU·0.5·60` mirror the countdown·256 / countdown·128 PSX steps at half speed; WEAPON_SPRITE spin for ninja throws) + tests/GPUThrowStoneTest.gd
  - src: `research/working_documents/handler_20_projectile_arc_system.md`
- **The impact sound fires when the frame countdown (DAT_801b8b4c) equals 2 — two frames before the projectile reaches the target — and only if g_sound_effect_id (0x801b8b64) != −1, slot+0x24 is NOT in {0x7D, 0x7E, 0x7F} or {0xF0–0xFD}, and slot+0x28 != 0, via trigger_effect_sound (0x80068cc4) passing caster_id (slot+0x12) and target_id (slot+0x1c); the exclusion ranges suppress sound for abilities that handle their own impact sounds.** — `[S] 1/3`
  - S: 0x801b3f3c–0x801b3fb0, trigger_effect_sound 0x80068cc4, per research/working_documents/handler_20_projectile_arc_system.md
  - R: none — countdown-2 impact sound trigger not present in smd-player, fft-sound-driver, or godot-learning (probed smd-player/addons/exmateria_sound, fft-sound-driver, godot-learning/src, godot-learning/tests)
  - src: `research/working_documents/handler_20_projectile_arc_system.md`
- **Termination at countdown 0 runs: step 1, free the sprite sub-effect (FUN_801adc24, clear slot+0x30) if present; step 2, impact reaction — Path A (slot+0x1a == 0 and slot+0x1d not in {3, 4}) calls FUN_801a1880 (impact animation init) + FUN_8008dee8(2), Path B otherwise checks FUN_801a18c8() == 1 && slot+0x28 != 0 to play trigger_effect_sound, then FUN_801a18a4 (impact cleanup); step 3, set slot+0x08 = 3 and return 1 for this frame; state 3 then returns 0 so the dispatcher removes the effect from the active list (0x801b3fe8–0x801b40f4).** — `[S] 1/3`
  - S: 0x801b3fe8–0x801b40f4, FUN_801adc24 0x801adc24, FUN_801a1880/801a18c8/801a18a4, per research/working_documents/handler_20_projectile_arc_system.md
  - R: none — sub-effect cleanup / impact reaction paths not present in godot-learning (probed godot-learning/src, godot-learning/tests)
  - src: `research/working_documents/handler_20_projectile_arc_system.md`

## Notes

(empty — user territory)

## Related

- [[Bow Arrow Arc System]]
- [[Spell Charge Effect System]]
- [[PSX GPU Primitives]]
- [[Isometric Coordinate System]]
- [[Effect System Index]]
