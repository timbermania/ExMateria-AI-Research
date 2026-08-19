# Particle Curve Indices

Spawn-time behaviour of the emitter's packed curve indices, resolved in the particle spawn routine FUN_801a60ac: each nibble of emitter bytes 0x08–0x0F indexes an entry in the per-effect animation-curve table (0xA0-byte entries; the interpolation factor is the byte at entry_base + anim_frame + 4), and the selected curve lerps the matching emitter start/end field pair into a specific particle field — per-frame spawn count (0xB0/0xB2), spawn center (via FUN_801a8c14 → particle +0x0C–0x14), spread (via FUN_801a8c8c → ±rand scatter), velocity base angle (via FUN_801a8d04 → rotation matrix), velocity-direction cone (0x38–0x43), radial velocity (0x5C–0x63), acceleration (0x64–0x7B → +0x24–0x2C), drag (0x7C–0x93 → +0x30–0x38), lifetime (0x94–0x9A → +0x42), and target/homing offset (0x9C–0xA6 → +0x3C–0x40 with anchor-mode base offsets) — with per-particle randomization between the lerped bounds. The high nibble of byte 0x0F carries two 2-bit fields: a homing-strength curve index (low 2 bits) and a direct homing-curve index (high 2 bits, copied to +0x45 without a lerp). The 2026-04-16 working document claims this section carries a `0x0F000000` header word with 15 × 0xA0 direction-vector subsections, recorded below as a low-evidence point. The particle-count curve is the most fully verified in this note: a 2025-11-20 runtime test on E001 emitter 2 confirmed the 1-based curve indexing and the start/end lerp behaviour, and the godot-learning reimplementation reproduces both the nibble decode and the count lerp.

## Points

- **Each emitter curve resolves against the per-effect animation-curve table (effect_anim_tbl_ptr, global 0x801BBF7C ← header anim_table_ptr): the per-frame interpolation factor for curve index N is the byte at `table_base + N * 0xA0 + anim_frame + 4` — a 0xA0-byte entry per curve, with per-frame bytes following a 4-byte header.** — `[S] 1/3`
  - S: interpolation-factor fetch in the spawn routine FUN_801a60ac (`anim_index * 0xa0 + effect_anim_tbl_ptr + anim_frame + 4`), per `research/working_documents/CURVE_ANALYSIS.md`
  - S: color-curve fetch in `update_particle_render_state` (0x801A5EA4) uses the same `table_base + index*0xA0 + anim_frame + 4` formula (160 uint8 values per curve), per `research/working_documents/PARTICLE_COLORING_SYSTEM.md`
  - src: `research/working_documents/CURVE_ANALYSIS.md`
- **The particle-count curve (high nibble of emitter byte 0x0E) lerps particle_count_start (0xB0) toward particle_count_end (0xB2) by the interpolation factor, and the result is the number of particles spawned that frame — each allocated via FUN_801a5c3c in a loop.** — `[S·D·R] 3/3`
  - S: particle-count lerp_u8 (emitter 0xB0/0xB2) and spawn loop with FUN_801a5c3c allocation in FUN_801a60ac, per `research/working_documents/CURVE_ANALYSIS.md`
  - D: E001 emitter 2 runtime write test (2025-11-20): `write 0x801C2990 0x00 0x00 0x10 0x00` switched the count curve from 2 (step: most frames 1 particle, some frames 20) to 0 (smooth ramp) — observed spawn counts rose gradually 1→19, matching the ramp curve
  - R: `godot-learning/src/effects/ActiveEmitter.gd:264-271` (`_get_particle_count` = `ParticlePhysics.interpolate_simple` → `lerpf(start, end, curve_t)` with curve_t = curve byte /255, floored to int ≥ 1) + `godot-learning/tools/parse_effect.py:577` (particle_count curve = high nibble of byte 0x0E); validated by `godot-learning/tests/EffectSoundCaptureTest.gd` (drives E001 through the parsed pipeline)
  - src: `research/working_documents/CURVE_ANALYSIS.md`
  - src: `research/working_documents/VERIFIED_particle_count_curve.md`
- **The 4-bit curve-index nibbles are 1-based: stored value 0 means no curve and values 1–15 select curves 0–14 (curve_index = nibble − 1) — E001 emitter 2's particle-count nibble (byte 0x0E high nibble) read 0x3 → curve 2, and the written 0x1 → curve 0.** — `[D·R] 2/3`
  - D: E001 emitter 2 runtime write test (2025-11-20): original word `00 00 30 00` at 0x801C2990 (nibble 5 = 0x3, curve 2 = step function) vs written `00 00 10 00` (nibble = 0x1, curve 0 = ramp) changed observed spawn behaviour accordingly
  - R: `godot-learning/tools/parse_effect.py:493-494` (`decode_curve_index`: "0 = none (-1), N = curve N-1"; particle_count read at `:577` from `(curve_bytes[6] >> 4) & 0x0F`) + `godot-learning/src/effects/ActiveEmitter.gd:286` (`_get_curve` maps idx < 0 to null); validated by `godot-learning/tests/EffectSoundCaptureTest.gd`
  - src: `research/working_documents/VERIFIED_particle_count_curve.md`
- **E001's animation-curve table at 0x801C2D58 (pointer global 0x801BBF7C holds `58 2D 1C 80`) holds 15 curves of 160 bytes each, curve N at base + N×0xA0: curve 0 is a smooth ramp 0x00→0xFA over its 160 frames (0%→98%) and curve 2 is a step function (0x00 at frames 4–66, 0xFF at frames 67–129, 3 transition bytes at each end) — burst vs gradual spawning for the particle-count curve.** — `[S·D] 2/3`
  - S: BATTLE.BIN bytes `58 2D 1C 80` at 0x801BBF7C (curve-table pointer = 0x801C2D58), per `research/working_documents/VERIFIED_particle_count_curve.md`
  - D: E001 memory dump during Cure (2025-11-20): curve 0 ramp bytes at dump offset 0x1C2D58, curve 2 step bytes at 0x1C2E98; observed spawn counts matched each curve's shape (1→20 bursts for curve 2, 1→19 ramp for curve 0)
  - R: none — E001 curve-table data not committed in godot-learning (assets/effects/ holds only the trap set; `godot-learning/src/effects/EffectCurve.gd` models a generic 160-sample 0–1 curve with frame-wrap; probed godot-learning/assets/effects/, src/effects/, tests/)
  - src: `research/working_documents/VERIFIED_particle_count_curve.md`
- **The spawn-position curve (low nibble of emitter byte 0x08) is resolved by FUN_801a8c14(emitter, factor, &out), which writes the interpolated spawn-center position into the particle's 12.12 position (+0x0C/+0x10/+0x14).** — `[S] 1/3`
  - S: FUN_801a8c14 call and particle +0x0C/+0x10/+0x14 writes in FUN_801a60ac, per `research/working_documents/CURVE_ANALYSIS.md`
  - src: `research/working_documents/CURVE_ANALYSIS.md`
- **The spread curve (high nibble of emitter byte 0x08) is resolved by FUN_801a8c8c(emitter, factor, &out), whose XYZ output is used as ±rand() bounds that scatter each particle around the interpolated spawn center.** — `[S] 1/3`
  - S: FUN_801a8c8c call and rand() scatter in FUN_801a60ac, per `research/working_documents/CURVE_ANALYSIS.md`
  - src: `research/working_documents/CURVE_ANALYSIS.md`
- **The velocity-direction curve (high nibble of emitter byte 0x09) lerps the emitter's velocity-direction-spread fields (0x38–0x43) and the result is used as ±rand() bounds that randomize each particle's direction vector (the spray cone).** — `[S] 1/3`
  - S: velocity-direction lerp (reserved_30_5B +8..+0x12) and rand() bounds in FUN_801a60ac, per `research/working_documents/CURVE_ANALYSIS.md`
  - src: `research/working_documents/CURVE_ANALYSIS.md`
- **The curve indexed by the low nibble of emitter byte 0x09 (velocity_base_angle per the emitter layout) is resolved by FUN_801a8d04(emitter, factor, &out), whose output feeds the rotation matrix (FUN_8001d658 / FUN_8001d578) as part of the velocity-direction calculation.** — `[S] 1/3`
  - S: FUN_801a8d04 call and rotation-matrix helpers in FUN_801a60ac, per `research/working_documents/CURVE_ANALYSIS.md`
  - src: `research/working_documents/CURVE_ANALYSIS.md`
- **Complete spawn-time velocity construction: the three velocity_base_angle fields (0x2C–0x37) are lerped via their curve, a random direction spread is added (± velocity_direction_spread 0x38–0x43), a rotation matrix is built from the three final angles, and the base vector (0, radial_magnitude, 0) — radial_magnitude lerped from 0x5C–0x63 — is rotated through that matrix; each result component is left-shifted by 3 bits (fixed-point) and written to particle velocity 0x18–0x20.** — `[S·R] 2/3`
  - S: `emitter_control_routine` (0x801a634c) decompilation "Velocity Calculation (Mode 2)", per `research/working_documents/PARTICLE_SYSTEM_ARCHITECTURE.md`
  - R: `godot-learning/src/effects/ParticlePhysics.gd:113-141` (angle_to_direction applies Z/Y/X rotations to the base vector; random_cone_direction adds the ±angular spread) + `godot-learning/src/effects/ActiveEmitter.gd:140-146` (velocity = final_dir × radial_vel); validated by `godot-learning/tests/EffectSoundCaptureTest.gd`
  - src: `research/working_documents/PARTICLE_SYSTEM_ARCHITECTURE.md`
- **The radial-velocity curve (high nibble of emitter byte 0x0B) lerps radial_velocity_min (0x5C→0x60) and radial_velocity_max (0x5E→0x62); each particle then gets a random value in [min, max) (min itself when min == max), 0 = straight-line motion and larger values = wider spirals; it only takes effect while the velocity-mode flag 0x400 (SKIP) is clear, and it controls radial (outward) spread rather than vertical motion.** — `[S] 1/3`
  - S: radial min/max lerp (emitter 0x5C/0x5E → 0x60/0x62) and per-particle rand() in FUN_801a60ac, per `research/working_documents/CURVE_ANALYSIS.md`
  - src: `research/working_documents/CURVE_ANALYSIS.md`
- **The acceleration curve (low nibble of emitter byte 0x0C) lerps six int16 values across the emitter's acceleration fields (0x64–0x7B) and stores the per-particle randomized result into the particle's 12.12 acceleration vector (+0x24/+0x28/+0x2C).** — `[S] 1/3`
  - S: acceleration lerp (reserved_60_AF +4..+0x1a) and particle +0x24–0x2C writes in FUN_801a60ac, per `research/working_documents/CURVE_ANALYSIS.md`
  - src: `research/working_documents/CURVE_ANALYSIS.md`
- **The drag curve (high nibble of emitter byte 0x0C) lerps six int16 values across the emitter's drag fields (0x7C–0x93) and stores the per-particle randomized result into the particle's drag coefficients (+0x30/+0x34/+0x38).** — `[S] 1/3`
  - S: drag lerp (reserved_60_AF +0x1c..+0x32) and particle +0x30–0x38 writes in FUN_801a60ac, per `research/working_documents/CURVE_ANALYSIS.md`
  - src: `research/working_documents/CURVE_ANALYSIS.md`
- **The lifetime curve (low nibble of emitter byte 0x0D) lerps the emitter's lifetime start/end pair (0x94/0x96 → 0x98/0x9A) and randomizes per particle between the two lerped values, storing into the particle's lifetime counter (+0x42).** — `[S] 1/3`
  - S: lifetime lerp (reserved_60_AF +0x34..+0x3a) and per-particle rand() store to +0x42 in FUN_801a60ac, per `research/working_documents/CURVE_ANALYSIS.md`
  - src: `research/working_documents/CURVE_ANALYSIS.md`
- **The target-offset curve (high nibble of emitter byte 0x0D) lerps the emitter's target-offset fields (0x9C/0x9E/0xA0 → 0xA2/0xA4/0xA6) and writes the result into the particle's homing target (+0x3C/+0x3E/+0x40) after adding the anchor-mode base offset (0x00/0x20 direct, 0x40 camera ×14, 0x60 origin, 0x80 target, 0xA0 cursor tile).** — `[S] 1/3`
  - S: target-offset lerp (reserved_60_AF +0x3c..+0x46), anchor-mode store paths, and particle +0x3C–0x40 writes in FUN_801a60ac, per `research/working_documents/CURVE_ANALYSIS.md`
  - src: `research/working_documents/CURVE_ANALYSIS.md`
- **The 2-bit curve index in the low 2 bits of emitter byte 0x0F (curves 0–3 only) selects the curve that lerps the homing-strength start/end pair (0xB8→0xBC, 0xBA→0xBE) and randomizes per particle into the particle's homing_strength (+0x4A).** — `[S] 1/3`
  - S: 2-bit index (`>> 0x1c & 3`) and homing lerp (reserved_B8_C3 +0..+6) store to +0x4A in FUN_801a60ac, per `research/working_documents/CURVE_ANALYSIS.md`
  - src: `research/working_documents/CURVE_ANALYSIS.md`
- **The high 2 bits of emitter byte 0x0F are copied directly (no curve lookup or lerp) into the particle's homing_curve_index (+0x45) as a 2-bit value 0–3.** — `[S] 1/3`
  - S: direct 2-bit copy (`>> 0x1e`) to particle +0x45 in FUN_801a60ac, per `research/working_documents/CURVE_ANALYSIS.md`
  - src: `research/working_documents/CURVE_ANALYSIS.md`

- **The 2026-04-16 working document defines the section at header offset 0x10 (the animation-table pointer slot) as a Coordinate/Direction Data section: a a header word `0x0F000000` followed by 15 subsections of 0xA0 bytes (0x964 total), claiming each subsection provides pre-calculated direction vectors for complex motion linked to the emitter 0x08–0x0F special-function flags; the 0xA0 subsection size matches this note's animation-curve table entry size, but the "direction vectors / 15 subsections" reading is a different interpretation of the section.** — `[ ] 0/3`
  - R: none — godot-learning `EffectCurve` keeps the 0xA0 curve-table entry model (`anim_index * 0xA0 + table + anim_frame + 4`); no `0x0F000000` header word is verified in the repo.
  - src: `research/working_documents/FFT_VFX_COMPLETE_TECHNICAL_REFERENCE.md`

## Notes

(empty — user territory)

## Related

- [[Particle Emitter Format]]
- [[Particle Runtime State]]
- [[Effect Execution Model]]
- [[Effect File Format]]
- [[Particle Coloring System]]
