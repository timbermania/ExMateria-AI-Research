# Sprite Offset vs Vertex Position

How FFT offsets an effect particle's sprite from the particle position: three additive mechanisms — vertex positions inside FRAMES frames (local to the particle center, rotated by the particle's rotation matrix), the animation-command sprite_offset set by SET_OFFSET (0x82) / ADD_OFFSET (0x83) (screen-space, applied before rotation, frame-synchronized), and particle physics (world space, continuous). sprite_offset carries hand-authored per-frame corrections — bounces, pops, and pixel-precise artwork alignment between framesets — not continuous motion.

## Points

- **sprite_offset (SET_OFFSET 0x82 / ADD_OFFSET 0x83) is applied in screen space before the rotation matrix and does not rotate, while FRAMES vertex positions are local to the particle center and rotate with the particle; the pipeline computes screen_center = render_position + sprite_offset, sets it as the GTE translation vector, and transforms each vertex as rotation_matrix × vertex_local + screen_center.** — `[S·R] 2/3`
  - S: globals screen_center_x/y 0x801BC0B0/0x801BC0B4 and rotation matrix 0x801BC09C, `render_particle_to_sprite` 0x801AA1F8, `submit_sprite_to_ordering_table` 0x801A5394, `init_matrix` 0x8001D138 (loads the GTE translation from matrix-struct offsets 0x14–0x1C)
  - R: `godot-learning/src/effects/ParticleAnimator.gd` (SET_OFFSET overwrites / ADD_OFFSET accumulates into per-frame `anim_offset`) + `godot-learning/src/effects/EffectParticleRenderer.gd:303-311` (adds `anim_offset` to all four sprite corners, a translation separate from the particle's world position) + `effect-editor/core/parser.lua` (SEQUENCE_OPCODES 0x82/0x83); no automated test
  - src: `research/working_documents/SPRITE_OFFSET_VS_VERTEX_POSITION.md`
- **Physics and sprite_offset are additive (final_position = physics render_position + sprite_offset) and two coexisting patterns exist in shipped effect data: "pure sprite animation" where all physics parameters are zero and the particle stays fixed (E024 emitters 0/4, E026 emitter 0, E027 emitters 0/5/6, E028 emitters 0/7, E052/E053/E058) and combined effects where both run together (E017 emitter 1: ADD_OFFSET bounce + WEIGHT=512 gravity; E024 emitter 2: ADD_OFFSET + VEL + WEIGHT=320; E025 emitters 0/4: WEIGHT=240; E026 emitters 3–7: VEL + WEIGHT) — zeroing physics for hand-animated effects like E024's bounce is a design choice, not a requirement.** — `[S·R] 2/3`
  - S: emitter physics parameters + sequence bytecode analysis of E017/E024/E025/E026/E027/E028/E052/E053/E058 .BIN files
  - R: `godot-learning/src/effects/ParticlePhysics.gd` (inertia/weight/gravity) + `godot-learning/src/effects/ActiveEmitter.gd:166-173` (per-particle weight lerp) + `godot-learning/src/effects/ParticleAnimator.gd` (sprite-offset layer) — both layers implemented additively; no automated test
  - src: `research/working_documents/SPRITE_OFFSET_VS_VERTEX_POSITION.md`
- **Most sequences open with SET_OFFSET(0,0) and the trailing LOOP back to the start resets the ADD_OFFSET accumulation to a known state; without it, offsets would accumulate across loops indefinitely.** — `[S·R] 2/3`
  - S: sequence 0 bytecode of E024.BIN and E017.BIN (SET_OFFSET(0,0) at head, trailing LOOP)
  - R: `godot-learning/src/effects/ParticleAnimator.gd` re-reads the baked per-frame offset from frame 0 on wrap (same reset semantics); no automated test
  - src: `research/working_documents/SPRITE_OFFSET_VS_VERTEX_POSITION.md`
- **Tiny ADD_OFFSET values (±1–2 px) used alongside physics are artwork-alignment compensation, not drift or bugs: framesets position their artwork differently relative to the (0,0) anchor, and pixel-calibrated cumulative offsets (no SET_OFFSET reset, persisting for the rest of the animation) make frame switches seamless — E024 animation 2 bridges frame 37 (TL 16,-4) to frame 23 (TL 14,-4) with a one-time ADD_OFFSET(+2,+1), and E026 animation 3 toggles ±2/±1 on every switch between frame 36 (center -2.5,-1.5) and frame 72 (center -0.5,-0.5), repeating 8 times.** — `[S] 1/3`
  - S: E024.BIN animation 2 frames 23/37 and E026.BIN animation 3 frames 36/72 top-left/center coordinates plus ADD_OFFSET bytecode
  - R: none — no artwork-alignment test in godot-learning (probed `godot-learning/src/`, `godot-learning/tests/`; the ADD_OFFSET accumulation mechanism itself is implemented in `godot-learning/src/effects/ParticleAnimator.gd`)
  - src: `research/working_documents/SPRITE_OFFSET_VS_VERTEX_POSITION.md`

## Notes

(empty — user territory)

## Related

- [[Effect Animation Sequences]]
- [[Particle Runtime State]]
- [[GTE World-to-Screen Transform]]
- [[Particle Emitter Format]]
