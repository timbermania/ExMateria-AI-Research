# Effect Frame Pacing

FFT's per-frame time-scale system for E###.BIN effects: an effect can slow or speed up its own playback (Fire 4's slow-motion explosion climax) via nibble-packed per-frame curves in the effect file, enabled per-pattern by Effect Flags bits, stored through set_frame_pacing into a global pacing value that the vsync code compares against a countdown timer to decide frame skipping. Duration values are in game frames (one per main-loop iteration), not VBlanks: keyframes advance at ~30 FPS without time scales and ~17 FPS with them (trace_phases probe, 2026-04-16), mirrored in godot-learning by EFFECT_FPS 30 plus a BASE_PACING/pacing clock factor. The 2026-04-16 working document claims animation frame durations run at 59 FPS, contradicting the 30 FPS evidence above, recorded as a low-evidence point.

## Points

- **The Effect Flags section (header[0x18]) is 24 bytes: flags_word at 0x00, default_frame_delay at 0x04 (0 = use 0x64 = 100), and sound_channels[4] at 0x08–0x17 (4 bytes each).** — `[S] 1/3`
  - S: Effect Flags section layout (flags_word 0x00, default_frame_delay 0x04, sound_channels 0x08–0x17), per `research/key_documents/STRUCTURE_DEFINITIONS.md`
  - src: `research/key_documents/STRUCTURE_DEFINITIONS.md`
- **Effect Flags flags_word bit 3 (0x08) TERRAIN_HEIGHT_ADJUST adjusts anchor Y for terrain slope in get_anchor_position (0x801A90D0), bit 4 (0x10) AUDIO_FADE fades audio at effect end (fade_effect_audio 0x801A1540/0x801A1B58), bit 5 (0x20) enables Pattern 1 time scale (curve offset 0x00), bit 6 (0x40) enables Pattern 2 (curve offset 0x12C); bits 0–2 are set in many files but no code reads them and bit 7 (0x80) is never observed.** — `[S] 1/3`
  - S: flags-bit table with get_anchor_position 0x801A90D0 and fade_effect_audio 0x801A1540/0x801A1B58, per `research/key_documents/STRUCTURE_DEFINITIONS.md`
  - src: `research/key_documents/STRUCTURE_DEFINITIONS.md`
- **Time scale data at header[0x14] holds two ~150-byte nibble-packed sections — +0x000 for Pattern 1 (flag 0x20) and +0x12C for Pattern 2 (flag 0x40) — with even frames in the low nibble and odd frames in the high nibble; value 0 = no pacing, 1–9 valid (1 fastest, 9 slowest), 10–15 clamped by set_frame_pacing.** — `[S] 1/3`
  - S: time-scale data format (two nibble sections at header[0x14] +0x000/+0x12C), per `research/key_documents/STRUCTURE_DEFINITIONS.md`
  - ⚠ SUPERSEDED (2026-08-19) by: The time scale data block at header[0x14] is exactly 600 bytes (0x258) — bytes 0–299 feed the 0x20-flagged region, bytes 300–599 (+0x12C) the 0x40-flagged region (300 bytes / 600 frames each, two frames per byte: even frame in the low nibble, odd in the high); set_time_scale (0x8008E328) stores only values 1–9, so 0 and 10–15 leave pacing unchanged
  - src: `research/key_documents/STRUCTURE_DEFINITIONS.md`
- **The pacing chain is lookup_time_scale_p1 (0x801A1200) / lookup_time_scale_p2 (0x801A1244) indexed by timeline_frame_counter/anim_progress → set_frame_pacing (0x8008E328) storing 1–9 into g_frame_pacing_value (0x80045998) → vsync code at 0x80093B2C comparing it against frame_pacing_timer (0x8004598C, 0–16); higher values skip more frames (slower), and clear_frame_pacing (0x8008E348) resets the value.** — `[S] 1/3`
  - S: lookup_time_scale_p1/p2 0x801A1200/0x801A1244, set/clear_frame_pacing 0x8008E328/0x8008E348, g_frame_pacing_value 0x80045998, frame_pacing_timer 0x8004598C, vsync comparison at 0x80093B2C, per `research/key_documents/STRUCTURE_DEFINITIONS.md`
  - src: `research/key_documents/STRUCTURE_DEFINITIONS.md`
- **Sound channel config is (mode, sound_id_a, sound_id_b, sound_id_c) and lookup_sound_effect (0x801A32E8) implements five modes (verified 0x801A332C–0x801A3404): 0 DIRECT_A, 1 PARITY_AB (reads byte +1 or +2 by parity — the stored value, not a computed one), 2 FIRST_A_THEN_B, 3 FIRST_A_THEN_BC, 4 CYCLE_ABC, cycling through the per-channel sound_call_count[4] at 0x801B9250 (lbu/sb at 0x801A32F4).** — `[S] 1/3`
  - S: lookup_sound_effect 0x801A32E8, mode table verified at 0x801A332C–0x801A3404, sound_call_count[4] 0x801B9250, per `research/key_documents/STRUCTURE_DEFINITIONS.md`
  - src: `research/key_documents/STRUCTURE_DEFINITIONS.md`
- **Fire 4 (E019) uses flag 0x20 with a ramping time-scale curve — frames 0–20 value 0–2 (fast), 20–40 value 3–4, 40+ value 5+ (slow) — producing the "slow motion explosion" climax.** — `[S] 1/3`
  - S: E019 ramping-curve example, per `research/key_documents/STRUCTURE_DEFINITIONS.md`
  - src: `research/key_documents/STRUCTURE_DEFINITIONS.md`
- **Effect duration is in game frames, not VBlanks or real time: 1 duration unit = 1 game-loop iteration = 1 increment of `timeline_frame_counter`; real time = duration / effective keyframe rate (~30 FPS without time scales, ~17 FPS with).** — `[S·D·R] 3/3`
  - S: game-loop structure (`effect_system_main_loop` increments `timeline_frame_counter`, then `FUN_80093a98` waits for N VBlanks), BATTLE.BIN decompilation per doc "Key Finding"
  - D: trace_phases.lua os.clock probe — E019 Fire 4 Phase 1 96 frames = 5.611 s (17.11 FPS), spawn 38 frames = 1.264 s (30.06 FPS); E001 Cure 12 frames = 0.432 s, 77 frames = 2.568 s (~28–30 FPS) (2026-04-16)
  - R: godot-learning/src/effects/ActiveEmitter.gd (EFFECT_FPS 30.0, FRAME_DURATION 1/30) + src/effects/EffectTimeline.gd (BASE_PACING 2, factor = BASE_PACING/pacing_value) + tests/EffectTimelineTest.gd _test_time_scale
  - src: `research/working_documents/EFFECT_TIMING_SYSTEM.md`
- **In effect-playback states (DAT_800960e4 = 0x33/0x2d) FUN_80093a98 picks the VSync wait: default 4 waits (15 FPS), 3 (20 FPS) while complexity counter DAT_8004598c < 16, 2 (30 FPS) at 0, else the counter decrements; g_frame_pacing_value then overrides to the larger wait when bigger (pacing 1 = no wait). Menus (DAT_80045980 == 1) wait 0; outside effects the wait is DAT_80045980 (2 = 30 FPS).** — `[S·D·R] 3/3`
  - S: FUN_80093a98 decompilation at 0x80093a98 with 0x800960E4 battle state, DAT_80045980/0x8004598C/0x80045998, BATTLE.BIN decompilation per doc "Frame Pacing Logic"
  - D: trace_phases.lua probe — spawn phases measured ~30 FPS (wait 2, counter 0) vs Phase 1 ~17 FPS (pacing override active) (2026-04-16)
  - R: godot-learning/src/effects/EffectTimeline.gd _update_time_scale implements the override rule (pacing valid 1–9 and > BASE_PACING, else factor 1.0) + tests/EffectTimelineTest.gd _test_time_scale; the DAT_8004598c complexity wait tree (0x80093a98) is not present in godot-learning
  - src: `research/working_documents/EFFECT_TIMING_SYSTEM.md`
- **DAT_80045980 (delta-time multiplier, 1 in menus/loading, 2 in battle) scales unit-movement velocity only: particle integration (integrate_particle_motion, 0x801A9BB0) adds velocity without the multiplier, and keyframe/timeline timing is unaffected.** — `[S·R] 2/3`
  - S: integrate_particle_motion decompilation at 0x801A9BB0 (velocity add, no multiplier) and DAT_80045980 value table, BATTLE.BIN decompilation per doc
  - R: godot-learning/src/scenarios/ScenarioWeather.gd (ANIM_SPEED_SCALAR 2.0 mirrors the DAT_80045980 ×2 unit-velocity scalar, §12 GAP 3) + tests/ScenarioWeatherTest.gd; particle path (ActiveEmitter/EffectTimeline) has no 0x80045980-equivalent multiplier
  - src: `research/working_documents/EFFECT_TIMING_SYSTEM.md`
- **Only 35 of 289 DATA-format effects (~12%) carry time-scale flags (0x20/0x40) and run at ~17 FPS: E019 Fire 4 / E023 Ice 4 / E027 Bolt 4 (flag 0x23), E031 Flare (0x41), E066 Holy (0x70), E142 Ultima (0x62), E288–E295 summons (0x60/0x68); the rest run at ~30 FPS.** — `[S·D·R] 3/3`
  - S: effect-header Effect Flags bits 0x20/0x40 across the 289 DATA effect .BIN files; named flags per doc census
  - D: trace_phases.lua probe — E019 (0x23) measured 17.11 FPS vs E001 Cure (flag 0x01) ~28–30 FPS (2026-04-16)
  - R: godot-learning/src/effects/EffectData.gd (loads per-effect time_scale.json) + src/effects/EffectInstance.gd (effect_timeline.setup with time_scale) → EffectTimeline pacing + tests/EffectTimelineTest.gd _test_time_scale
  - src: `research/working_documents/EFFECT_TIMING_SYSTEM.md`

- **The 2026-04-16 working document states animation frame durations are "frames at 59 FPS" (each duration unit = 1/59 s of playback) and particle update runs "each frame @ 59 FPS" — contradicting the ~30 FPS game-frame timing above.** — `[ ] 0/3`
  - R: godot-learning uses EFFECT_FPS = 30.0, and the 3/3 points above measure ~30/~17 FPS real time; EffectTimeline/EffectAnimation model durations as time-scale curve values, not 59 FPS ticks.
  - src: `research/working_documents/FFT_VFX_COMPLETE_TECHNICAL_REFERENCE.md`
  - src: `research/working_documents/VFX_PARTICLES_EMITTERS_DEEP_DIVE.md`
- **The time scale data block at header[0x14] is exactly 600 bytes (0x258) — bytes 0–299 feed the 0x20-flagged region, bytes 300–599 (+0x12C) the 0x40-flagged region (300 bytes / 600 frames each, two frames per byte: even frame in the low nibble, odd in the high); set_time_scale (0x8008E328) stores only values 1–9, so 0 and 10–15 leave pacing unchanged.** — `[S·R] 2/3`
  - S: decompiled lookup_time_scale_curve 0x801A1200 / lookup_time_scale_p2 0x801A1244 (region ±300 offset, nibble select on frame parity) and set_time_scale 0x8008E328 (`if (param_1 - 1U < 9)`), per `research/wiki_articles/effect_file_time_scale.txt` and the doc's decompilations
  - R: effect-editor/core/parser.lua parse_time_scales_from_data + unpack_nibbles (TIME_SCALE_BYTE_COUNT 300, ANIMATE_TICK_BYTE_OFFSET 300, even=low/odd=high) + godot-learning/src/effects/EffectData.gd (time_scale.json load) → src/effects/EffectTimeline.gd _update_time_scale ("0 and 10-15: leave current pacing unchanged")
  - src: `research/working_documents/TIME_SCALE_IMPLEMENTATION_GUIDE.md`
- **The 0x20-flagged region is read only by outer_phases_timeline_tick (opcode 41) during phase-1 (timeline_frame_counter < phase1_duration); the 0x40-flagged region is read every frame by for_each_phase_timeline_tick (opcode 40) via anim_progress; once phase-1 ends the parent stops updating pacing and spawned for-each instances take over through opcode 40.** — `[S·D·R] 3/3`
  - S: andi flag checks at 0x801a4a6c (0x20, phase-1 only, bytes 0–299) and 0x801a3c08 (0x40, bytes 300–599), BATTLE.BIN disassembly per the doc
  - D: trace_phases.lua os.clock probe — Fire 4 E019 phase 1 96 frames = 5.611 s (17.11 FPS, time scale active) vs spawn phase 38 frames = 1.264 s (30.06 FPS, post phase-1, no time scale) (recorded in doc, 2026-05-10)
  - R: godot-learning/src/effects/EffectTimeline.gd _update_time_scale (time_scale_pattern1 requires not phase1_finished, time_scale_pattern2 only after, phase-1→for-each edge resets to BASE_PACING) + tests/EffectTimelineTest.gd _test_time_scale
  - src: `research/working_documents/TIME_SCALE_IMPLEMENTATION_GUIDE.md`
- **Within the for-each tick (opcode 40) the time scale lookup is step 5 of 6 — after the 5 particle channels, 4 color tracks, 3 sound channels, and camera update, just before anim_progress increments — so a newly read value takes effect on the NEXT frame_sync, not the frame it was read in.** — `[S·R] 2/3`
  - S: opcode 40 per-frame processing order, per `research/wiki_articles/timeline_section.txt` §4.2 and the doc (lookup at 0x801a3c08 region)
  - R: godot-learning/src/effects/EffectTimeline.gd (factor recomputed at end of tick, applied to the next accumulator increment) + tests/EffectTimelineTest.gd _test_time_scale ("applies on the NEXT accumulation (one-frame lag)"; frames 0–1 at full speed, factor 0.5 lands after frame 1)
  - src: `research/working_documents/TIME_SCALE_IMPLEMENTATION_GUIDE.md`
- **Both duration systems erode once per game loop: lifetime_counter −1/game frame, frame_timer −2/game frame, preserving the 2:1 ratio at any pacing (equivalent fixed lifetime = total frame_timer / 2); a particle with lifetime=60 lives 2.0 s at pacing 2 and 4.0 s at pacing 4; wall time = frames × pacing/60 (lifetime) or × pacing/120 (frame_timer).** — `[S·R] 2/3`
  - S: erosion rates and duration formulas, per `research/wiki_articles/particle_system_section.txt` + `research/wiki_articles/effect_timing.txt`; integrate_particle_motion 0x801A9BB0 (one integration per loop)
  - R: godot-learning/src/effects/ParticleSubsystem.gd (particle.age += 1 per timeline frame; Particle.is_dead dies at age >= lifetime) + src/effects/ParticleAnimator.gd ("FFT decrements frame_timer by 2 each game frame", display_frames = duration >> 1)
  - src: `research/working_documents/TIME_SCALE_IMPLEMENTATION_GUIDE.md`
- **Time scale changes only the evaluation rate, not the physics step: integrate_particle_motion (0x801A9BB0) runs once per game loop with no delta-time compensation — velocity, acceleration, drag, gravity, and homing integrate identically, so particle travel per real second scales down with 1/pacing.** — `[S·R] 2/3`
  - S: integrate_particle_motion decompilation at 0x801A9BB0 (no time-scale or delta-time term), per `research/wiki_articles/particle_system_section.txt`
  - R: godot-learning/src/effects/ParticleSubsystem.gd (identical per-frame integration, driven by the single EffectTimeline accumulator, ADR-0012) + tests/EffectTimelineTest.gd _test_time_scale
  - src: `research/working_documents/TIME_SCALE_IMPLEMENTATION_GUIDE.md`
- **Time scale does not affect audio: SPU music/SFX playback continues at normal tempo while visuals slow (the cinematic contrast); sound triggers still fire at the same game-frame numbers, so their wall-clock spacing stretches with the visuals.** — `[ ] 0/3`
  - R: none — no time_scale/pacing coupling in smd-player/addons/exmateria_sound or fft-sound-driver (audio driver unaffected by design)
  - src: `research/working_documents/TIME_SCALE_IMPLEMENTATION_GUIDE.md`

## Notes

(empty — user territory)

## Related

- [[Effect Execution Model]]
- [[Effect File Format]]
- [[Color Track Interpolation]]
- [[Effect Sound Timing]]
