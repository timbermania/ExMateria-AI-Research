# Particle Coloring System

How FFT colors battle-effect particles: two independent sources multiplied by the GPU hardware — a per-pixel texture palette (CLUT) and a per-vertex color value that, when color curves are enabled (emitter behavior_flags bit 6, 0x40), is a per-channel sample from the effect's animation-curve table. Particles are submitted as GT4 (Gouraud-shaded textured quad) primitives whose vertex color is that curve value (or neutral 0x80), and the PSX computes `final = texture × vertex / 128` per channel. Verified against disassembly and real effect files: E001 Cure is palette-only, E173 tints red by fading G/B curves, E016 Fire overbrights with a 255-start R curve. godot-learning reimplements the curve-modulation side in `EffectParticleRenderer` (per-emitter curve cache, per-particle-age sampling, neutral-white default).

## Points

- **Particle final color is a per-channel GPU multiply of the CLUT palette color and the GT4 vertex color, `final = texture × vertex / 128`: vertex value 0x00 = ×0.0, 0x40 = ×0.5, 0x80 = ×1.0 (neutral), 0xC0 = ×1.5, 0xFF ≈ ×2.0 (overbright) — the palette supplies the base hue and the vertex value scales brightness.** — `[S·R] 2/3`
  - S: color-curve lookup and color pass in `update_particle_render_state` (0x801A5EA4), vertex-color store in `render_particle_to_sprite` (0x801AA1F8), GT4 primitive build in `submit_sprite_to_ordering_table` (0x801A5394), per `research/working_documents/PARTICLE_COLORING_SYSTEM.md`
  - R: `godot-learning/src/effects/EffectParticleRenderer.gd` `_setup_emitter_caches` / `_compute_color_modulate` (per-emitter R/G/B color-curve cache gated on `color_curve_enabled`, sampled per particle age; modulate color multiplies the palette texture; neutral white when disabled) + `godot-learning/tools/parse_effect.py:583-588` (color-curve nibble decode); validated by `godot-learning/tests/EffectSoundCaptureTest.gd` (drives the parsed E001 pipeline)
  - src: `research/working_documents/PARTICLE_COLORING_SYSTEM.md`

- **Color-curve indices sit in emitter bytes 0x10–0x11 (R = low nibble of 0x10, G = high nibble of 0x10, B = low nibble of 0x11); index 0 means "no curve — constant 0x80" and index 1–15 selects animation curve (index − 1); curves apply only when behavior_flags bit 6 (0x40) is set, otherwise all channels are forced to neutral 0x80.** — `[S·R] 2/3`
  - S: curve-index bytes 0x10–0x11 and the 0x40 enable gate at the color lookup in `update_particle_render_state` (0x801A5EA4), per `research/working_documents/PARTICLE_COLORING_SYSTEM.md`
  - R: `godot-learning/tools/parse_effect.py:583-588` (same nibble layout: r = 0x10 & 0x0F, g = 0x10 >> 4 & 0x0F, b = 0x11 & 0x0F) and `parse_effect.py:748-752` (color_curve_enabled = flags & 0x40) + `godot-learning/src/effects/EffectParticleRenderer.gd` (gate honored; note the reimplementation uses the nibble directly as the curve index without the doc's 0-skip / −1 mapping); validated by `godot-learning/tests/EffectSoundCaptureTest.gd`
  - src: `research/working_documents/PARTICLE_COLORING_SYSTEM.md`

- **`emitter_control_routine` (0x801A634C) copies the color-curve nibbles from the emitter into the particle at bytes 0x46 (R), 0x47 (G), 0x48 (B) at spawn time; the per-frame value is then read in `update_particle_render_state` (0x801A5EA4) as `*(anim_table_ptr + 4 + curve_index * 0xA0 + anim_frame)` — 160 uint8 values per curve, one per frame of the effect cycle.** — `[S] 1/3`
  - S: nibble copy in `emitter_control_routine` (0x801A634C) and the 0xA0-stride table lookup in `update_particle_render_state` (0x801A5EA4), per `research/working_documents/PARTICLE_COLORING_SYSTEM.md`
  - R: none — godot-learning keeps color curves per-emitter (`EffectParticleRenderer` caches one curve array per emitter; no per-particle 0x46–0x48 copy); probed godot-learning/src + godot-learning/tests
  - src: `research/working_documents/PARTICLE_COLORING_SYSTEM.md`

- **E001 (Cure) is palette-only coloring: every emitter has color curves disabled (behavior_flags & 0x40 = 0) with neutral 0x80 vertex color, and the effect looks blue because its CLUT palette is blue/cyan (avg R=51.8, G=141.5, B=211.5 over the 16-color palette).** — `[S] 1/3`
  - S: E001.BIN emitter flag bytes and texture-section CLUT (16-bit colors at the header texture_ptr), per `research/working_documents/PARTICLE_COLORING_SYSTEM.md`
  - R: none — E001 palette-average analysis not present in godot-learning (probed godot-learning/src + godot-learning/tests)
  - src: `research/working_documents/PARTICLE_COLORING_SYSTEM.md`

- **E173 tints red by curve subtraction: emitter 2 has behavior_flags = 0x40 with R_curve = 0 (constant 0x80), G_curve = curve 1 and B_curve = curve 0 (both fade to 0), so green/blue channels fade out over ~60 frames while red stays at full brightness, over a near-white/neutral palette (avg R=222.5, G=215.0, B=209.3).** — `[S] 1/3`
  - S: E173.BIN emitter 2 flags/curve bytes and animation-curve values, per `research/working_documents/PARTICLE_COLORING_SYSTEM.md`
  - R: none — E173 curve configuration not present in godot-learning (probed godot-learning/src + godot-learning/tests)
  - src: `research/working_documents/PARTICLE_COLORING_SYSTEM.md`

- **E016 (Fire) opens with an overbright flash: emitter 0 has curves enabled with R_curve = curve 1 starting at 255 (≈ ×2.0 red multiplier at frame 0), G/B on rising fade curves, over an orange/warm palette (avg R=244.4, G=217.0, B=149.4).** — `[S] 1/3`
  - S: E016.BIN emitter 0 flags/curve bytes, curve value data, and texture-section palette, per `research/working_documents/PARTICLE_COLORING_SYSTEM.md`
  - R: none — E016 curve configuration not present in godot-learning (probed godot-learning/src + godot-learning/tests)
  - src: `research/working_documents/PARTICLE_COLORING_SYSTEM.md`

## Notes

(empty — user territory)

## Related

- [[Particle Emitter Format]]
- [[Particle Runtime State]]
- [[Particle Curve Indices]]
- [[Effect Texture Upload]]
- [[PSX GPU Primitives]]
- [[Frameset Header Flags]]
