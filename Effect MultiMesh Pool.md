# Effect MultiMesh Pool

The ADR-0040 particle-batching pool behind the Godot reimplementation's combat effect particles: each effect borrows 5 MultiMeshes from `EffectMultiMeshPool` — `RM_OPAQUE` + `RM_MODE0..3` — and the per-frame renderer loop routes every particle frame into the bucket whose fixed `render_mode` matches its (pass, blend_mode), packing per-instance state into the MultiMesh instance buffers (corners + depth_mode in the transform basis, world position in the origin, uv_rect in INSTANCE_CUSTOM, modulate + semi_trans_on in COLOR) so O(particles×passes×effects) draw calls collapse from 600 to ~50 and the per-instance `set_shader_parameter` storm dies. Ordering rides three layers — L1 `psx_ot_depth` depth write, L2 instance-buffer write order, L3 opaque-queue-then-transparent-queue — and mode bucketing trades cross-mode submission order (all adds contiguous, all subs contiguous) for that win, with occlusion coming from the depth buffer rather than draw order. A 2026-07-19 compositor spike confirmed the MultiMesh RD instance buffer's 20-floats/instance layout at runtime and proved the buffer is drawable instanced from a `CompositorEffect` — the basis for folding the blend buckets into the display-space fold.

## Points

- **Each effect borrows 5 MultiMeshes from the pool — `RM_OPAQUE` + `RM_MODE0..3` — and the per-frame loop (`update_particles` → `_write_instance`) walks particles in build order and packs each frame into the MultiMesh whose fixed `render_mode` matches its (pass, blend_mode): `RM_OPAQUE` runs `depth_draw_opaque` (writes depth = the canvas + depth buffer) while `RM_MODE0..3` run `blend_mix / blend_add / blend_sub / blend_add` with `depth_draw_never` (read depth, hardware-blend in the scene transparent pass in LINEAR space — the color-space defect the display-space fold exists to fix); the pool's whole reason for existing is killing the O(particles×passes×effects) draw calls (600→50) and the per-instance `set_shader_parameter` storm.** — `[R] 1/3`
  - R: `godot-learning/src/effects/EffectMultiMeshPool.gd` + `godot-learning/src/effects/EffectParticleRenderer.gd` (`update_particles` → `_write_instance`), `godot-learning/assets/shaders/effect_particle_opaque.gdshader` (`depth_draw_opaque`), `godot-learning/tools/probe_shaders/effect_particle_mode{0..3}.gdshader` (`blend_mix`/`blend_add`/`blend_sub`/`blend_add`, `depth_draw_never`)
  - src: `research/working_documents/COMPOSITOR_MULTIMESH_INTEGRATION_SCOPING.md`
- **Per-instance state is packed into the MultiMesh instance buffers: corners + depth_mode in the transform basis, world position in the origin, uv_rect in INSTANCE_CUSTOM, modulate + `semi_trans_on` in COLOR (alpha) — unpacked by `effect_particle_stp.gdshaderinc`, which reads basis[0]=(tl.x, tl.y, tr.x), basis[1]=(tr.y, bl.x, bl.y), basis[2]=(br.x, br.y, depth_mode) and billboards the particle at the MODEL_MATRIX origin.** — `[R] 1/3`
  - R: `godot-learning/assets/shaders/effect_particle_stp.gdshaderinc` (packing header + unpack), `godot-learning/src/effects/EffectParticleRenderer.gd` (`_write_instance`)
  - src: `research/working_documents/COMPOSITOR_MULTIMESH_INTEGRATION_SCOPING.md`
- **Ordering rides three layers — L1 the per-prim `psx_ot_depth` DEPTH write, L2 the instance-buffer write order, L3 opaque-queue-then-transparent-queue — and mode bucketing destroys cross-mode submission order (all adds contiguous, all subs contiguous) while preserving the blend sequence within a mode as the instance-buffer order; occlusion comes from the depth buffer (opaque `RM_OPAQUE` depth + per-prim `psx_ot_depth`), not from submission order.** — `[R] 1/3`
  - R: `godot-learning/src/effects/EffectParticleRenderer.gd` (per-frame instance write in build order), `godot-learning/assets/shaders/psx_ot_depth.gdshaderinc`
  - src: `research/working_documents/COMPOSITOR_MULTIMESH_INTEGRATION_SCOPING.md`
- **A MultiMesh's RD instance buffer is a GPU storage buffer with a 20-floats/instance layout — [0..11] transform 3×4 row-major (origin at f[3], f[7], f[11]), [12..15] color rgba as unpacked floats (NOT rgba8), [16..19] custom_data — confirmed at runtime by the 2026-07-19 compositor spike, so the ADR-0040 packing (corners in basis, world pos in origin, uv in custom, modulate + stp in color) is readable verbatim in a compositor shader and the mode-shader unpack ports 1:1.** — `[R] 1/3`
  - R: throwaway spike `tools/compositor_prototype/proto_mm_compositor{.gd,.glsl,_effect.tscn}` (headful, RTX 5090), runtime layout + pixel readback, 2026-07-19
  - src: `research/working_documents/COMPOSITOR_MULTIMESH_INTEGRATION_SCOPING.md`

## Notes

(empty — user territory)

## Related

- [[Display Space Blend Fold]]
- [[Particle Runtime State]]
