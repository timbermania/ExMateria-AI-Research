# Particle Curve Indices

Spawn-time behaviour of the emitter's packed curve indices, resolved in the particle spawn routine FUN_801a60ac: each nibble of emitter bytes 0x08–0x0F indexes an entry in the per-effect animation-curve table (0xA0-byte entries; the interpolation factor is the byte at entry_base + anim_frame + 4), and the selected curve lerps the matching emitter start/end field pair into a specific particle field — per-frame spawn count (0xB0/0xB2), spawn center (via FUN_801a8c14 → particle +0x0C–0x14), spread (via FUN_801a8c8c → ±rand scatter), velocity base angle (via FUN_801a8d04 → rotation matrix), velocity-direction cone (0x38–0x43), radial velocity (0x5C–0x63), acceleration (0x64–0x7B → +0x24–0x2C), drag (0x7C–0x93 → +0x30–0x38), lifetime (0x94–0x9A → +0x42), and target/homing offset (0x9C–0xA6 → +0x3C–0x40 with anchor-mode base offsets) — with per-particle randomization between the lerped bounds. The high nibble of byte 0x0F carries two 2-bit fields: a homing-strength curve index (low 2 bits) and a direct homing-curve index (high 2 bits, copied to +0x45 without a lerp). The 2026-04-16 working document claims this section carries a `0x0F000000` header word with 15 × 0xA0 direction-vector subsections, recorded below as a low-evidence point.

## Points

- **Each emitter curve resolves against the per-effect animation-curve table (effect_anim_tbl_ptr, global 0x801BBF7C ← header anim_table_ptr): the per-frame interpolation factor for curve index N is the byte at `table_base + N * 0xA0 + anim_frame + 4` — a 0xA0-byte entry per curve, with per-frame bytes following a 4-byte header.** — `[S] 1/3`
  - S: interpolation-factor fetch in the spawn routine FUN_801a60ac (`anim_index * 0xa0 + effect_anim_tbl_ptr + anim_frame + 4`), per `research/working_documents/CURVE_ANALYSIS.md`
  - S: color-curve fetch in `update_particle_render_state` (0x801A5EA4) uses the same `table_base + index*0xA0 + anim_frame + 4` formula (160 uint8 values per curve), per `research/working_documents/PARTICLE_COLORING_SYSTEM.md`
  - src: `research/working_documents/CURVE_ANALYSIS.md`
- **The particle-count curve (high nibble of emitter byte 0x0E) lerps particle_count_start (0xB0) toward particle_count_end (0xB2) by the interpolation factor, and the result is the number of particles spawned that frame — each allocated via FUN_801a5c3c in a loop.** — `[S] 1/3`
  - S: particle-count lerp_u8 (emitter 0xB0/0xB2) and spawn loop with FUN_801a5c3c allocation in FUN_801a60ac, per `research/working_documents/CURVE_ANALYSIS.md`
  - src: `research/working_documents/CURVE_ANALYSIS.md`
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
