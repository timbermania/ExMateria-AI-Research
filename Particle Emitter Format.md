# Particle Emitter Format

The on-disk 196-byte (0xC4) ParticleEmitter record embedded in each E###.BIN's Particle System section (at `effect_data_ptr + 0x14 + index*0xC4`, the section holding a uint16 emitter count at +0x02 and a 16-byte header block at +0x04). Emitters are pure static configuration — a 16-byte core control block, per-parameter nibble curve indices, and int16 min/max start/end range pairs for every physics parameter (position, spread, velocity angle/spread, inertia, weight, radial velocity, acceleration, drag, lifetime, target offset, spawn, homing). Callback parameters and child-emitter slots round out the record; blend mode is NOT here (it is baked into the animation sequence word). A corpus analysis of 1945 emitters across 291 DATA-format effect files confirms the control-byte layout (byte_00/byte_05 constant, one 0x77 outlier), characterises real-world usage of the anchor, spread, and flag fields, and bounds the int16 physics-range fields (value sentinels, 4096 angle units, and which fields effects actually use); a 2026-04-16 runtime byte-mutation test on E001 confirmed the anim_index, anchor-mode, and behavior-flag behaviour of the core control bytes. A 2026-04-16 working document offers a competing byte-level reading (behavior flags, section-enable flags, RGBA color modulation, single-purpose parameter fields, and a hypothetical particle-count formula), recorded below as low-evidence points; the code- and corpus-backed model above remains current.

## Points

- **A ParticleEmitter is a 196-byte (0xC4) static record at `effect_data_ptr + 0x14 + emitter_index * 0xC4`, whose core control bytes 0x00–0x0F are: byte_00 unused padding, anim_index 0x01 (Entry Table index at header[0x00]; each 2-byte entry offsets into animation sequence data holding frame timing, sprite indices, and blend mode), motion_type_flag 0x02 (bit 1 align_sprite_to_velocity, bits 5–7 target anchor mode), animation_target_flag 0x03 (bit 0 spread mode 0=spherical/1=box, bits 1–3 emitter anchor mode, bits 4–7 sprite variant bits), anim_param 0x04 (sprite_variant passed to init_particle_animation), byte_05 unused.** — `[S·D·R] 3/3`
  - S: ParticleEmitter core-control table, per `research/key_documents/STRUCTURE_DEFINITIONS.md`
  - D: runtime byte-0x01 mutation on E001.BIN emitter 2 at 0x801C2985 via PCSX-Redux (2026-04-16): different anim_index values display different sprite animations
  - R: `godot-learning/tools/parse_effect.py:553` (`anim_index = read_u8(data, offset + 0x01)`) + `godot-learning/src/effects/EffectEmitter.gd` (`anim_index` var, set at line 99); `godot-learning/tests/EffectSoundCaptureTest.gd` exercises the parsed E001 pipeline
  - src: `research/key_documents/STRUCTURE_DEFINITIONS.md`
  - src: `research/working_documents/EMITTER_FIELD_TESTING_RESULTS.md`
- **Emitter flag bytes 0x06/0x07 decode as: bits 0–1 child_death_mode, bits 2–3 child_midlife_mode, bit 4 velocity_inward, bit 5 vestigial (set in 723 emitters but no code reads it), bit 6 color_curve_enable; high byte bits 0–1 homing_arrival_threshold (0=disabled, 1=16, 2=32, 3=48 units) and bit 2 align_to_unit_facing (rotate velocities to caster's facing).** — `[S·D·R] 3/3`
  - S: emitter flag-bit tables, per `research/key_documents/STRUCTURE_DEFINITIONS.md`
  - D: runtime flags-lo/flags-hi mutation on E001.BIN emitter 2 via PCSX-Redux (2026-04-16): 0x06=0x04 increased spawn-routine breakpoint hits (mid-life child spawning), 0x06=0x10 kept particles at the feet (inward velocity), 0x07=0x04 produced strong directional shoot-out motion (facing alignment)
  - R: `godot-learning/tools/parse_effect.py:748-752` (child_midlife_enabled 0x0C, velocity_inward 0x10, color_curve_enabled 0x40, align_to_facing 0x04, homing_arrival_threshold = flags_hi & 0x03)
  - src: `research/key_documents/STRUCTURE_DEFINITIONS.md`
  - src: `research/working_documents/EMITTER_FIELD_TESTING_RESULTS.md`
- **Curve-index nibble bytes 0x08–0x0F pair low/high nibbles per parameter: 0x08 emitter_position/particle_spread, 0x09 velocity_base_angle/velocity_direction_spread, 0x0A inertia/(dead code), 0x0B weight/radial_velocity, 0x0C acceleration/drag, 0x0D lifetime/target_offset, 0x0E (unused)/particle_count, 0x0F spawn_interval/homing_strength/homing_adjustment; color-curve indices sit at 0x10 (R low nibble, G high nibble) and 0x11 (B low nibble), each channel independent.** — `[S] 1/3`
  - S: curve-index byte table, per `research/key_documents/STRUCTURE_DEFINITIONS.md`
  - src: `research/key_documents/STRUCTURE_DEFINITIONS.md`
- **Emitter parameters are int16 min/max start/end range pairs: start/end position 0x14–0x1F, spread 0x20–0x2B, velocity base angles 0x2C–0x37, velocity direction spread 0x38–0x43, inertia 0x44–0x4B, weight 0x54–0x5B, radial velocity 0x5C–0x63, acceleration 0x64–0x7B, drag 0x7C–0x93, lifetime 0x94–0x9B, target offset 0x9C–0xAF, particle count/spawn interval 0xB0–0xB7, homing strength 0xB8–0xBF.** — `[S] 1/3`
  - S: emitter field offset tables, per `research/key_documents/STRUCTURE_DEFINITIONS.md`
  - S: independent decompilation trace — `emitter_control_routine` lerps 0x94–0x9A into particle+0x42 (lifetime countdown) and 0xB8–0xBE into particle+0x4A (homing param), confirming both range-pair fields, per `research/working_documents/LIFETIME_CORRECTION.md`
  - src: `research/key_documents/STRUCTURE_DEFINITIONS.md`
  - src: `research/working_documents/LIFETIME_CORRECTION.md`
- **Callback parameters 0x4C–0x53 and 0xA8–0xAE are not read by the particle system (it lerps start/end but discards the result); MIPS callbacks read them directly — e.g. E317 CB91 uses 0x4C/0x4E for CLUT blend-mode bits and brightness-table index, and 0xA8/0xAA for trail UV step range (value>>3 = pixels/segment) and ribbon width.** — `[S] 1/3`
  - S: callback_param fields and E317 CB91/CB92 semantics, per `research/key_documents/STRUCTURE_DEFINITIONS.md`
  - S: dead-code lerp of reserved_30_5B +0x1c/+0x1e → +0x20/+0x22 (callback params 0x4C–0x52) driven by the byte 0x0A high nibble (bits 20–23) in spawn routine FUN_801a60ac, result discarded, per `research/working_documents/CURVE_ANALYSIS.md`
  - S: E317 CB92 reads callback_param_4/param_6 (0xA8/0xAC) as bicone radius start/end — in E317 the lerp curve index is −1 so no interpolation happens and growth comes from an accumulator instead; parameter semantics are callback-specific (the same offset means different things per callback), per `research/working_documents/E317_callback_system.md`
  - src: `research/key_documents/STRUCTURE_DEFINITIONS.md`
  - src: `research/working_documents/CURVE_ANALYSIS.md`
  - src: `research/working_documents/E317_callback_system.md`
- **Child emitters: byte 0xC0 child_emitter_on_death (enabled by emitter_flags_lo & 0x03) and 0xC1 child_emitter_mid_life (enabled by emitter_flags_lo & 0x0C); 0xC2/0xC3 are unused.** — `[S] 1/3`
  - S: child-emitter bytes and enable masks, per `research/key_documents/STRUCTURE_DEFINITIONS.md`
  - src: `research/key_documents/STRUCTURE_DEFINITIONS.md`
- **Target anchor modes (emitter byte 0x02 bits 5–7, where particles home TOWARD): 0x00/0x20 DIRECT_OFFSET (target_offset only), 0x40 CAMERA (camera_position×14 + offset), 0x60 ORIGIN (effect_origin_position + offset), 0x80 TARGET (effect_target_position + offset), 0xA0 CURSOR (cursor_tile×28 + tile_height×−12 + offset); emitter anchor modes (byte 0x03 bits 1–3, where particles spawn FROM): 0x00 WORLD, 0x02 CURSOR, 0x04 ORIGIN (caster), 0x06 TARGET, 0x08 PARENT_PARTICLE, 0x0A CAMERA, 0x0C TRACKED_ENTITY.** — `[S·D·R] 3/3`
  - S: anchor-mode value tables, per `research/key_documents/STRUCTURE_DEFINITIONS.md`
  - D: runtime byte-0x03 mutation on E001.BIN emitter 2 at 0x801C2987 via PCSX-Redux (2026-04-16): 0x04 places the emitter on the caster unit, 0x06 on each target — matches the ORIGIN/TARGET anchor entries
  - R: `godot-learning/tools/parse_effect.py:745-747` (target_anchor_mode from byte 0x02 bits 5–7, emitter_anchor_mode ANCHOR_MODES from byte 0x03 bits 1–3) + `godot-learning/src/effects/EffectEmitter.gd:212` `get_emitter_anchor_mode()`; validated by `godot-learning/tests/GPUCallbackE065Test.gd`
  - src: `research/key_documents/STRUCTURE_DEFINITIONS.md`
  - src: `research/working_documents/EMITTER_FIELD_TESTING_RESULTS.md`
- **Velocity modes (bytes 0x06–0x07 combined): 0x0000 OUTWARD (standard angular velocity), 0x0010 INWARD (toward emitter center), 0x0400 SKIP (UNIMPLEMENTED — skips velocity calc), 0x0410 OUTWARD_UNIT_ORIENTED (outward rotated to unit facing).** — `[S] 1/3`
  - S: velocity-mode table, per `research/key_documents/STRUCTURE_DEFINITIONS.md`
  - S: identical VEL_MODES table (master_parser.py:1596–1597: 0x0000 OUTWARD, 0x0010 INWARD, 0x0400 SKIP, 0x0410 OUTWARD_UNIT_ORIENTED), per `research/working_documents/MASTER_PARSER_GAPS.md` (2026-05-27)
  - src: `research/key_documents/STRUCTURE_DEFINITIONS.md`
- **PSX semi-transparency blend mode is NOT in the emitter structure — it is baked into the animation sequence word (bit 5 = STP enable, bits 6–7 = ABR: 0=50% average, 1=additive, 2=subtractive, 3=25% additive); the code path at 0x801a59ec computes `tpage = (animation_word & 0xE0) | 0x08`, so particles from the same emitter with different animation indices can render with different transparency.** — `[S] 1/3`
  - S: blend-mode bits and code at 0x801a59ec, per `research/key_documents/STRUCTURE_DEFINITIONS.md`
  - src: `research/key_documents/STRUCTURE_DEFINITIONS.md`
- **Corpus analysis of 291 DATA-format effect files (1945 emitters) confirms the emitter control-byte layout: byte_00 (0x00) is always 0x00, and byte_05 is always 0x00 except one outlier emitter holding 0x77.** — `[ ] 0/3`
  - src: `research/working_documents/CONTROL_BYTE_OPTIONS.md`
- **Emitter anchor mode (byte 0x03 bits 1–3, where particles spawn from) is dominated in the 1945-emitter corpus by TARGET 64.2%, ORIGIN 17.8%, PARENT 12.6% (WORLD 0.9%, CURSOR 2.5%, CAMERA 1.2%, TRACKED 0.8%).** — `[ ] 0/3`
  - src: `research/working_documents/CONTROL_BYTE_OPTIONS.md`
- **Corpus usage of the remaining emitter control fields: spread mode sphere 75.7% vs box 24.3%; color_curve_enable on in 80.7% (constant RGB 0x80/0x80/0x80 otherwise); velocity_inward on in 32.0%; align_to_velocity on in 16.9%; align_to_unit_facing on in 3.2%; child_death_mode disabled 90.3% (mode 1 7.1%, mode 2 2.6%, value 3 disabled); child_midlife_mode disabled 94.4% (mode 1 5.0%, mode 2 0.6%, value 3 disabled); homing_arrival_threshold disabled 99.1% (16 units 0.8%, 32 units 0.1%, 48 units 0.1%).** — `[ ] 0/3`
  - src: `research/working_documents/CONTROL_BYTE_OPTIONS.md`
- **anim_index (byte 0x01) peaks at 0 (23.0%), 1 (19.2%), 2 (18.0%) in the corpus; anim_param (byte 0x04) is 0 for 99.2% of emitters (valid range 0–2).** — `[ ] 0/3`
  - src: `research/working_documents/CONTROL_BYTE_OPTIONS.md`
- **Byte 0x03 bits 4–7 (called render_handler in this analysis) index a handler table whose entries all point to the same function, so the field is vestigial with no per-emitter effect.** — `[ ] 0/3`
  - src: `research/working_documents/CONTROL_BYTE_OPTIONS.md`
- **The Particle System (emitter control) section holds a uint16 emitter count at section offset +0x02, then a 16-byte header block at +0x04, with the first 196-byte emitter record starting at +0x14.** — `[R] 1/3`
  - R: `godot-learning/tools/parse_effect.py` (`emitter_count = read_u16(data, offset + 0x02)`, `PARTICLE_HEADER_SIZE = 0x14`) + `godot-learning/tests/EffectSoundCaptureTest.gd` (exercises the parsed E001 pipeline)
  - src: `research/working_documents/E001_debug_guide.md`
- **Emitter byte 0x0F is a special-function flag (values 0/1) that halves the particle count.** — `[ ] 0/3`
  - R: none — no 0x0F particle-count halving in godot-learning (`parse_effect.py` decodes the 0x0F high nibble as the homing_strength curve index)
  - src: `research/working_documents/E001_debug_guide.md`
  - src: `research/working_documents/FFT_VFX_COMPLETE_TECHNICAL_REFERENCE.md`
- **Emitter offset 0x2C is an 8-bit direction_primary field (values 0-15, clock face, 22.5° per step).** — `[ ] 0/3`
  - R: none — no 8-bit clock-face direction in godot-learning (`parse_effect.py` reads 0x2C as the int16 velocity base-angle start)
  - src: `research/working_documents/E001_debug_guide.md`
  - src: `research/working_documents/FFT_VFX_COMPLETE_TECHNICAL_REFERENCE.md`
  - ⚠ SUPERSEDED (2026-08-17) by: velocity base-angle fields 0x2C–0x37 are signed int16 fixed-point with 4096 units = 1 full rotation (corpus P98 up to ~7 rotations in spiral effects)
- **Changing byte 0x03 moves the emitter itself to a new anchor position and particles spawn relative to the emitter — the 'particle position' shifts observed on E001 E2 are the spawn origin moving, not particle motion.** — `[D·R] 2/3`
  - D: PCSX-Redux runtime memory-modification test of E001.BIN emitter 2, byte 0x03 at 0x801C2987 (2026-04-16): 0x02 places the emitter on the target tile, 0x04 on the caster unit, 0x06 on each target
  - R: `godot-learning/src/effects/EffectEmitter.gd:212` `get_emitter_anchor_mode()` (emitter anchor positioned per byte 0x03 bits 1–3) + `godot-learning/tests/GPUCallbackE065Test.gd` (particles positioned as anchor + offset)
  - src: `research/working_documents/EMITTER_FIELD_TESTING_RESULTS.md`
- **Lifetime fields use 65535 as the infinite sentinel: corpus P98 of lifetime_min/max_start is 65535 (infinite/animation-driven lifetimes are common), while lifetime_min/max_end P98 is only ~20–21 because finite end-lifetimes are the norm.** — `[S·R] 2/3`
  - S: lifetime fields 0x94–0x9B, P2–P98 percentiles across 291 effect files / 1945 emitters, per `research/working_documents/EMITTER_SLIDER_RANGES.md`
  - R: `godot-learning/tools/parse_effect.py:2108-2115` (lifetime >= 65535 converted to -1, "In FFT, lifetime = -1 means particle dies when animation completes") + `godot-learning/src/effects/ActiveEmitter.gd:149-154` (preserves -1 for animation-driven lifetimes, floors positive values to 1); validated by `godot-learning/tests/EffectSoundCaptureTest.gd` (drives E001/E065 through the parsed pipeline)
  - src: `research/working_documents/EMITTER_SLIDER_RANGES.md`
- **Velocity base angles (0x2C–0x37) are signed int16 fixed-point with 4096 units = 1 full rotation; real spiral/vortex effects use up to 7 rotations (E003.BIN and E266.BIN = 7.0 maximum, E001.BIN Cure healing spiral = 3.0).** — `[S·R] 2/3`
  - S: velocity base-angle P2–P98 values and multi-rotation counts across 291 effect files, per `research/working_documents/EMITTER_SLIDER_RANGES.md`
  - R: `godot-learning/tools/parse_effect.py:60` (`ANGLE_TO_RADIANS = math.tau / 4096.0`, "FFT angle (0-4096 = 0-360°)") + `godot-learning/src/effects/ActiveEmitter.gd:121-141` (lerps velocity_base_angle start→end and applies it to the velocity direction); validated by `godot-learning/tests/EffectSoundCaptureTest.gd`
  - src: `research/working_documents/EMITTER_SLIDER_RANGES.md`
- **Inertia clusters at exactly 4096 in the corpus (P2 = P98 = 4096 across 1945 emitters) with an observed actual range of 1–13632.** — `[S·R] 2/3`
  - S: inertia fields 0x44–0x4B, P2–P98 percentiles across 291 effect files, per `research/working_documents/EMITTER_SLIDER_RANGES.md`
  - R: `godot-learning/src/effects/EffectEmitter.gd:39-41` (inertia defaults 4096.0) + `godot-learning/src/effects/ActiveEmitter.gd:167-169` (lerps inertia over the particle lifetime); validated by `godot-learning/tests/EffectSoundCaptureTest.gd`
  - src: `research/working_documents/EMITTER_SLIDER_RANGES.md`
- **Acceleration and drag are rarely used (98% of emitters all zero) and, when used, affect only the Y axis (accel_y_*/drag_y_*); observed accel_y range is -14992..4448 and drag_y -3840..1440.** — `[S·R] 2/3`
  - S: acceleration 0x64–0x7B / drag 0x7C–0x93 P2–P98 percentiles across 291 effect files, per `research/working_documents/EMITTER_SLIDER_RANGES.md`
  - R: `godot-learning/tools/parse_effect.py:520-530` (reads accel s16s at 0x64–0x70, drag s16s at 0x7C–0x88, Y-flipped) + `godot-learning/src/effects/ActiveEmitter.gd:175-182` (applies both as per-frame vec3 corrections); validated by `godot-learning/tests/EffectSoundCaptureTest.gd`
  - src: `research/working_documents/EMITTER_SLIDER_RANGES.md`
- **Homing strength fades over the particle lifetime: start values run high (corpus P98 = 65504, often near max) while end values are almost always 0.** — `[S·R] 2/3`
  - S: homing fields 0xB8–0xBF, P2–P98 percentiles across 291 effect files, per `research/working_documents/EMITTER_SLIDER_RANGES.md`
  - R: `godot-learning/src/effects/ActiveEmitter.gd:185-187` (homing_strength lerped from start to end over t); validated by `godot-learning/tests/EffectSoundCaptureTest.gd`
  - src: `research/working_documents/EMITTER_SLIDER_RANGES.md`
- **Radial velocity decelerates over the particle lifetime: start values run very high (P98 ≈ 65332) while end values are much lower (P98 ≈ 3706–4559) — most emitters launch fast and settle.** — `[S·R] 2/3`
  - S: radial-velocity fields 0x5C–0x63, P2–P98 percentiles across 291 effect files, per `research/working_documents/EMITTER_SLIDER_RANGES.md`
  - R: `godot-learning/src/effects/ActiveEmitter.gd:113-115` (radial_vel lerped start→end over t) + `godot-learning/tools/parse_effect.py:82-84` (radial ×8/4096/28 conversion to Godot units/frame); validated by `godot-learning/tests/EffectSoundCaptureTest.gd`
  - src: `research/working_documents/EMITTER_SLIDER_RANGES.md`
- **Spawn control typical values: particle_count P2–P98 = 1–8 (max observed 27, end values ≤ 10) and spawn_interval P2–P98 = 1–49 frames (max observed 208, end values ≤ 33).** — `[S·R] 2/3`
  - S: spawn fields 0xB0–0xB7, P2–P98 percentiles across 291 effect files, per `research/working_documents/EMITTER_SLIDER_RANGES.md`
  - R: `godot-learning/tools/parse_effect.py:676-678` (particle_count_start at 0xB0, interval_start at 0xB4) + `godot-learning/src/effects/ActiveEmitter.gd:255-268` (curve-interpolated spawn pacing); validated by `godot-learning/tests/EffectSoundCaptureTest.gd`
  - src: `research/working_documents/EMITTER_SLIDER_RANGES.md`
- **Weight (0x54–0x5B) is a signed gravity multiplier — FFT applies `gravity_effect = gravity * weight >> 12` (÷4096) — with corpus start-range P98 ≈ -1984..1234 and end-range P98 ≈ -64..512.** — `[S·R] 2/3`
  - S: weight fields 0x54–0x5B, P2–P98 percentiles across 291 effect files, per `research/working_documents/EMITTER_SLIDER_RANGES.md`
  - R: `godot-learning/tools/parse_effect.py:627-629` (weight kept raw, "used as gravity multiplier, divided by 4096 in formula") + `godot-learning/src/effects/ActiveEmitter.gd:171-173` (lerps weight over t); validated by `godot-learning/tests/EffectSoundCaptureTest.gd`
  - src: `research/working_documents/EMITTER_SLIDER_RANGES.md`

- **The 2026-04-16 working document reinterprets the core control bytes: byte 0x00 = enable/disable flag, byte 0x01 = color palette index, byte 0x02 = motion type flag (0x00 static, 0x02 linear lerp, 0x60 parabolic arc, 0x80 complex), byte 0x03 = display/target flag (0x01 bitflag mode, 0x02 target panel/tile, 0x04 source unit, 0x06/0x07 sequential/simultaneous targets, 0x08 absolute map coordinates, 0x0B special targeting mode, 0x26 special targeting variant for Truth/Un-Truth/Fire 4), byte 0x04 = animation speed, byte 0x05 = display mode, byte 0x06 = color-masking/motion flags (0x04 trail, 0x10 radial force, 0x40 color modulation using 0x10–0x13), byte 0x07 bit 0x01 = stop the emitter at the target position.** — `[ ] 0/3`
  - R: none — godot-learning `parse_effect.py`/`EffectEmitter.gd` keep this note's 3/3 model (0x01 anim_index, 0x02/0x03 anchor bitfields, 0x04 anim_param, 0x06/0x07 child/homing flags)
  - src: `research/working_documents/FFT_VFX_COMPLETE_TECHNICAL_REFERENCE.md`
- **The working document reads bytes 0x08–0x0F as section-enable flags ("special_func_08–0F") instead of curve-index nibbles: 0x08/0x09 enable the coordinate/direction data section, 0x0A enables 8 uint16 complex-motion parameters at 0x44–0x52, 0x0B enables 4 uint16 motion parameters at 0x54–0x5A, 0x0C enables 24 uint16 timing parameters at 0x64–0x92 plus propagation speeds at 0xB8–0xBE, 0x0D/0x0E unknown, and 0x0F non-zero halves the particle count.** — `[ ] 0/3`
  - R: none — godot-learning `parse_effect.py` decodes 0x08–0x0F as paired 4-bit curve indices, resolved at spawn time in FUN_801a60ac
  - src: `research/working_documents/FFT_VFX_COMPLETE_TECHNICAL_REFERENCE.md`
- **The working document defines bytes 0x10–0x13 as 8-bit RGBA color modulation values (color_mask_r/g/b/a, 0x00 = no effect, /255.0 scale), referenced only when byte 0x06's 0x40 color-modulation flag is set; this note's model reads 0x10/0x11 as per-channel 4-bit color-curve indices.** — `[ ] 0/3`
  - R: none — godot-learning `parse_effect.py` keeps 0x10/0x11 as 4-bit curve indices driving per-channel color curves (color_curve_enable = 0x06 bit 6)
  - src: `research/working_documents/FFT_VFX_COMPLETE_TECHNICAL_REFERENCE.md`
- **The working document reads the emitter parameter region as single-purpose fields instead of int16 min/max range pairs: int16 start position at 0x14–0x18 and int16 end position at 0x1A–0x1E (world space, FFT units ≈ 1/28 tile; end order z@0x1A, y@0x1C, x@0x1E); int16 spread values at 0x20–0x26 (diagonal UL-LR, vertical, diagonal UR-LL, horizontal); 8-bit clock-face direction plus 7 int16 variants at 0x2C–0x33; int16 randomness factors (±variance) at 0x34–0x3A; 8 uint16 complex-motion parameters at 0x44–0x52; 4 uint16 motion parameters at 0x54–0x5A; int16 vertical arc-velocity pair (upward + variant) at 0x5C–0x5E; 24 uint16 timing parameters at 0x64–0x92 (12 low-nibble at 0x64–0x7A, 12 high-nibble at 0x7C–0x92); 4 uint16 fade-timing values at 0x94–0x9A; 4 int16 target-relative directional offsets at 0x9C–0xA2; and 4 uint16 motion-propagation speeds at 0xB8–0xBE (early/late × 2, used when 0x0C is set).** — `[ ] 0/3`
  - R: none — godot-learning `parse_effect.py` decodes these offsets as int16 start/end range pairs (4096-unit angles, inertia, weight/gravity, radial velocity, …), backed by corpus statistics
  - src: `research/working_documents/FFT_VFX_COMPLETE_TECHNICAL_REFERENCE.md`
- **The working document names 0xB0 particle_count_base, 0xB2 particle_count_factor, 0xB4 a non-linear multiplier (0x0001 ≈ "multiplied by a ton" (100×), 0x0002+ tapering diminishing returns), and 0xB6 an adjustment value, with a particle-count formula it labels hypothetical: (base × factor × mult_curve) + (0xB6 − 0xB4 × 0xB4), halved when 0x0F is non-zero, clamped to a minimum of 1.** — `[ ] 0/3`
  - R: none — godot-learning `parse_effect.py` uses 0xB0/0xB4 as particle_count_start/interval_start; the formula is a working-document hypothesis
  - src: `research/working_documents/FFT_VFX_COMPLETE_TECHNICAL_REFERENCE.md`

## Notes

(empty — user territory)

## Related

- [[Effect File Format]]
- [[Effect Execution Model]]
- [[E001.BIN Memory Mapping]]
- [[E001 Emitter Interaction]]
- [[Particle Runtime State]]
- [[E317 Choco Ball Callback System]]
