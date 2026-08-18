# Effect Frame Pacing

FFT's per-frame time-scale system for E###.BIN effects: an effect can slow or speed up its own playback (Fire 4's slow-motion explosion climax) via nibble-packed per-frame curves in the effect file, enabled per-pattern by Effect Flags bits, stored through set_frame_pacing into a global pacing value that the vsync code compares against a countdown timer to decide frame skipping.

## Points

- **The Effect Flags section (header[0x18]) is 24 bytes: flags_word at 0x00, default_frame_delay at 0x04 (0 = use 0x64 = 100), and sound_channels[4] at 0x08–0x17 (4 bytes each).** — `[S] 1/3`
  - S: Effect Flags section layout (flags_word 0x00, default_frame_delay 0x04, sound_channels 0x08–0x17), per `research/key_documents/STRUCTURE_DEFINITIONS.md`
  - src: `research/key_documents/STRUCTURE_DEFINITIONS.md`
- **Effect Flags flags_word bit 3 (0x08) TERRAIN_HEIGHT_ADJUST adjusts anchor Y for terrain slope in get_anchor_position (0x801A90D0), bit 4 (0x10) AUDIO_FADE fades audio at effect end (fade_effect_audio 0x801A1540/0x801A1B58), bit 5 (0x20) enables Pattern 1 time scale (curve offset 0x00), bit 6 (0x40) enables Pattern 2 (curve offset 0x12C); bits 0–2 are set in many files but no code reads them and bit 7 (0x80) is never observed.** — `[S] 1/3`
  - S: flags-bit table with get_anchor_position 0x801A90D0 and fade_effect_audio 0x801A1540/0x801A1B58, per `research/key_documents/STRUCTURE_DEFINITIONS.md`
  - src: `research/key_documents/STRUCTURE_DEFINITIONS.md`
- **Time scale data at header[0x14] holds two ~150-byte nibble-packed sections — +0x000 for Pattern 1 (flag 0x20) and +0x12C for Pattern 2 (flag 0x40) — with even frames in the low nibble and odd frames in the high nibble; value 0 = no pacing, 1–9 valid (1 fastest, 9 slowest), 10–15 clamped by set_frame_pacing.** — `[S] 1/3`
  - S: time-scale data format (two nibble sections at header[0x14] +0x000/+0x12C), per `research/key_documents/STRUCTURE_DEFINITIONS.md`
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

## Notes

(empty — user territory)

## Related

- [[Effect Execution Model]]
- [[Effect File Format]]
- [[Color Track Interpolation]]
