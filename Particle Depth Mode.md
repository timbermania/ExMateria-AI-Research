# Particle Depth Mode

How FFT assigns effect-particle sprites to PSX Ordering Table buckets. depth_mode is per-frame data in byte 2 of each 3-byte animation frame entry, read by the particle sprite renderer into ParticleAnimState (offset 0x14). On 3D GTE paths the base bucket is GTE SZ >> 2 and the mode then biases or fixes the index (full six-mode table lives in [[Ordering Table & AddPrim]]); the 2D screen path (flags & 6 == 2) bypasses both the GTE and depth_mode, using render Z directly as the OT index. Within a single bucket, order is set by spawn iteration: timeline channels are processed 0→4, so later channels draw in front and emitter index is irrelevant. The Godot reimplementation mirrors the model with a calibrated ordering-table depth shader (src/core/DepthMode.gd, ADR-0009).

## Points

- **depth_mode is stored in byte 2 of each 3-byte animation frame entry ([frameset_index][duration][depth_mode], frame opcode byte < 0x80) — per frame, not per emitter or per frameset — and render_particle_to_sprite (0x801AA1F8) loads it into ParticleAnimState.depth_mode (offset 0x14) each tick.** — `[S·R] 2/3`
  - S: render_particle_to_sprite 0x801AA1F8; `lbu depth_mode, 0x14(s1)` @ 0x801aa524, battle disassembly, per `research/working_documents/DEPTH_MODE_RENDER_ORDER_GUIDE.md` §2/§10
  - R: godot-learning/tools/parse_effect.py (FRAME opcode: `depth_mode = read_u8(pos + 2)`, 3-byte advance) + godot-learning/src/effects/ParticleAnimator.gd (baked {frameset, duration, depth_mode} → Particle.depth_mode); no named test; effect-editor/ui/sequences_tab.lua exposes all 6 modes
  - src: `research/working_documents/DEPTH_MODE_RENDER_ORDER_GUIDE.md`
- **The flags bits 1–2 (ParticleAnimState offset 0x06) select the particle render path: flags & 6 == 2 is 2D screen mode, which ignores depth_mode entirely — no GTE transform, no SZ>>2; render_position_z is clamped to [0, 0x17E] and used directly as the OT index, and positions add straight onto screen offsets; modes 0 and 4 both apply the same depth_mode switch (duplicated code), and mode 6 is a depth-sorted 2D variant.** — `[S] 1/3`
  - S: 2D screen path + duplicated depth_mode switch within render_particle_to_sprite (0x801AA1F8), per `research/working_documents/DEPTH_MODE_RENDER_ORDER_GUIDE.md` §3 Path B / §11 + battle disassembly
  - R: none — 2D screen render mode (flags & 6 == 2) not present in godot-learning
  - src: `research/working_documents/DEPTH_MODE_RENDER_ORDER_GUIDE.md`
- **Same-bucket particle order is set by spawn iteration, not emitter identity: timeline channels are processed in order 0 → 1 → 2 → 3 → 4 and each spawn head-inserts into the active particle list, so the render loop iterates channel-4 particles before channel-0 — within one OT bucket, higher channel index (then later spawn, then later batch order) appears in front, and emitter index is irrelevant for render order.** — `[S] 1/3`
  - S: per `research/wiki_articles/particle_system_section.txt` §12.2–12.4 (particle-list insertion + OT insertion), cited in `research/working_documents/DEPTH_MODE_RENDER_ORDER_GUIDE.md` §5
  - R: none — channel-index tie-break not present in godot-learning (ActiveEmitter.channel_index is explicitly "NOT a Z-order key", ADR-0015)
  - src: `research/working_documents/DEPTH_MODE_RENDER_ORDER_GUIDE.md`

## Notes

(empty — user territory)

## Related

- [[Ordering Table & AddPrim]]
- [[Particle Runtime State]]
- [[GTE World-to-Screen Transform]]
- [[Terrain Render Pipeline]]
