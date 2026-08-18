# E317 Choco Ball Callback System

E317.BIN is the Choco Ball (Chocobo skill) effect: a 43 KB CODE-format file whose two embedded MIPS callbacks — CB91 (trail projectile) and CB92 (bicone burst, two slots) — drive the visible action alongside 9 particle emitters (3 callback-routed, 6 decorative). CB91 manages a pool of up to 8 sub-projectiles that fly on their initial velocity but whose trail positions cosine-ease back toward the target; it renders a glowing additive POLY_GT4 ribbon with spatial + temporal fades and re-implements child ball-sprite spawning itself because callback routing bypasses the normal particle spawn path. CB92 is a pure geometry renderer expanding a faceted 8-sided bicone at the target (additive, purple→red, ~16-frame fade, no particles). The convergence "flash" is keyframe choreography — bicones and TARGET-anchored decorative scatters are pre-positioned 12–18 frames before the ~t=65 convergence — with only CB91's on-death child emission causally linked to ball arrival.

## Points

- **E317.BIN (Choco Ball) is a 43,536-byte CODE-format effect embedding two MIPS callback functions across three slots — slot 0 → callback 91 (FUN_801c2500, trail projectile) and slots 1 and 2 → callback 92 (FUN_801c44a0, bicone burst) — alongside 9 particle emitters (3 routed to callbacks, 6 decorative), loaded at 0x801C2500.** — `[S] 1/3`
  - S: load base 0x801C2500, slot table 0x801C2500/0x801C44A0 (file offsets 0x0000/0x1FA0), per `research/working_documents/E317_callback_system.md` (E317.BIN disassembly)
  - src: `research/working_documents/E317_callback_system.md`
- **A particle channel's `action_flags` bits 0–2 route that channel to a MIPS callback INSTEAD of normal particle spawning — the if/else at 0x801A40F0 (`andi v0,v0,0x7` + `beq v0,zero` at 0x801A4108) invokes the callback instead of `emitter_control_routine`, never both — so on a build without MIPS callback support all three E317 callback channels fall to the else-path and spawn identical ball sprites (anim 0 → frameset 1).** — `[S] 1/3`
  - S: 0x801A40F0 / 0x801A4108, per `research/working_documents/E317_callback_system.md` (E317.BIN disassembly)
  - src: `research/working_documents/E317_callback_system.md`
- **All nine E317 emitters use anim_idx=0, and animation 0 (E317.BIN file offset 0x3B9E) plays frameset 1 on loop, so every emitter renders the 16×16 ball sprite centered at (16,32) regardless of its own `frameset_idx` field.** — `[S] 1/3`
  - S: animation 0 data at E317.BIN offset 0x3B9E, frameset 1 UV (8,24) 16×16, per `research/working_documents/E317_callback_system.md`
  - src: `research/working_documents/E317_callback_system.md`
- **CB91 (FUN_801c2500) manages a pool of up to 8 sub-projectile slots (slot stride 0x3AC, allocation 0x1D8C bytes via alloc_particle 0x801A4DE8), each holding 12.12 fixed-point position/velocity and a 9-entry XYZ trail ring buffer at slot+0x368; the per-frame spawn count comes from the lerp'd `particle_count` field of emitter config 2 (3→1), not a hardcoded MIPS constant.** — `[S] 1/3`
  - S: FUN_801c2500, alloc_particle 0x801A4DE8, per-slot layout offsets, per `research/working_documents/E317_callback_system.md`
  - src: `research/working_documents/E317_callback_system.md`
- **CB91's sub-projectiles fly on their initial velocity vectors but their trail positions are eased toward the target by cosine_ease (0x801A8834) over each projectile's lifetime — trail head reaches the target at frame == lifetime, full convergence at lifetime + 8 — with no homing involved (emitter config 2 has homing_strength = 0).** — `[S] 1/3`
  - S: cosine_ease 0x801A8834, FUN_801c2500, per `research/working_documents/E317_callback_system.md`
  - src: `research/working_documents/E317_callback_system.md`
- **CB91's trail renders as up to 8 POLY_GT4 quads per sub-projectile at constant screen-space width (~15 px; rcos × callback_param_5(15) >> 13 ≈ 7.5 px half-width), each quad sampling a 3-px UV column (callback_param_4 = 24 → 3 px/segment) walking U=8→32, V=8–23 head-to-tail, with adjacent quads sharing edge vertices at the angle-averaged perpendicular (flipped when the angle difference exceeds 0x1000 to prevent twisting).** — `[S] 1/3`
  - S: E317 emitter config 2 callback params (0xA8=24, 0xAA=15), ribbon construction, per `research/working_documents/E317_callback_system.md`
  - src: `research/working_documents/E317_callback_system.md`
- **CB91's trail fades two complementary ways: a spatial head→tail brightness gradient driven by the 9-entry lookup table at 0x801C5BB0 (head reads the highest index; entry 0 = linear ramp 0→4096), and a temporal shortening after convergence where `convergence_age` drops the visible-quad count by 1 per frame while the surviving quads walk to later texture U columns.** — `[S] 1/3`
  - S: DAT_801c5bb0 (E317.BIN file offset 0x36B0), FUN_801c2500, per `research/working_documents/E317_callback_system.md`
  - src: `research/working_documents/E317_callback_system.md`
- **Because callback routing bypasses the normal spawn path, CB91 re-implements child spawning itself, calling emitter_control (0x801A60AC) at the trail-ring-head position: every frame mid-life via `child_emitter_mid_life` (emitter offset 0xC1 → emitter 0) and on death at frame_counter == lifetime + 8 via `child_emitter_on_death` (offset 0xC0 → emitter 1) — the on-death burst is the only causally triggered component of the convergence flash.** — `[S] 1/3`
  - S: emitter_control 0x801A60AC, routing if/else 0x801A40F0, per `research/working_documents/E317_callback_system.md`
  - src: `research/working_documents/E317_callback_system.md`
- **Both CB91 and CB92 double-buffer their GPU primitives: each slot holds two banks of POLY_GT4 data plus a flag toggled (`*flag = 1 - *flag`) at frame end, so the GPU renders one bank while the callback writes the other, preventing tearing mid-frame.** — `[S] 1/3`
  - S: double-buffer flag at allocation +0x00 (cleanup flags at +0x1D88/+0x1A18), per `research/working_documents/E317_callback_system.md` (E317.BIN decompilation)
  - src: `research/working_documents/E317_callback_system.md`
- **CB92 (FUN_801c44a0, two slots) is a pure geometry renderer: it builds a faceted 8-sided bicone — two cones glued base-to-base along Y, 5 cross-sections mirrored ±Y → 9 depth positions, 64 POLY_GT4 quads per bank × 2 banks, allocation 0x1A1C — and makes zero `emitter_control` calls, so it produces no ball sprites or particles; the two slots stay independent via `param_2` → `sprite_ptrs[slot]` (EffectState+0xE4+slot×4) and `callback_state` (EffectState+0x22+slot).** — `[S] 1/3`
  - S: FUN_801c44a0, EffectState offsets 0xE4/0x22, per `research/working_documents/E317_callback_system.md`
  - src: `research/working_documents/E317_callback_system.md`
- **CB92's bicone radius profile comes from the embedded table DAT_801c5d60 [4096, 3965, 3547, 2709, 0] (widest ring at the Y=0 center, point at the tips), and its growth is driven not by the callback_param_4→param_6 lerp (curve index −1 → no interpolation) but by the `angular_velocity_accum` accumulator, which adds +1536/frame from emitter config 4's acceleration field (offset 0x64; >>8 ≈ 6 world units/frame), reaching a visible radius of ~2→~98 world units (~3.5 tiles) before the color curves zero out at ~frame 16.** — `[S] 1/3`
  - S: DAT_801c5d60, FUN_801c44a0, per `research/working_documents/E317_callback_system.md`
  - src: `research/working_documents/E317_callback_system.md`
- **CB92 repurposes emitter-config fields for its GPU parameters: blend config from `spread_z_start` at offset 0x24 (E317 value 0x0001 → additive ABR mode 1, semi-transparency on, TPAGE 0xA6, CLUT 0x7B00 — the same additive blending as CB91's ribbon, which instead reads its blend bits from callback_param_0 at 0x4C), UV from the `velocity_base_angle` lerps (x: 32→0, y: 8→0) and spread fields (u_step = spread_x = 8, v_step = spread_y = 8), and color curves R=14/G=12/B=13 that tint the burst purple→red over ~16 frames (green dies by frame 2, leaving R+B purple until a faint red trace).** — `[S] 1/3`
  - S: E317 emitter config 4 field values, brightness table DAT_801c5c88, per `research/working_documents/E317_callback_system.md`
  - src: `research/working_documents/E317_callback_system.md`
- **E317's convergence "flash" at the target is keyframe choreography, not causal triggering: the two CB92 bicones are pre-positioned at t=47/t=53 (12–18 frames before the ~t=65 convergence), the TARGET-anchored decorative scatter emitters fire at t=39 (emitter 3) and t=53 (emitter 5), and the only causally linked component is CB91's on-death child emission — with emitter config 2's lifetime lerp 25→16 compressing the window so late-spawned balls still arrive inside the bicones' brightness windows.** — `[S] 1/3`
  - S: timeline channel/keyframe layout (ch0 KF2 @t=40, ch4 KF2 @t=47, ch2 KF4 @t=53), lifetime fields at emitter config 2 offsets 0x94–0x9B, per `research/working_documents/E317_callback_system.md`
  - src: `research/working_documents/E317_callback_system.md`

## Notes

(empty — user territory)

## Related

- [[Embedded MIPS Effect Code]]
- [[Effect Callback Mesh]]
- [[GTE World-to-Screen Transform]]
- [[Ordering Table & AddPrim]]
- [[Particle Emitter Format]]
- [[Effect Execution Model]]
