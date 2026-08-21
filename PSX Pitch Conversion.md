# PSX Pitch Conversion

How the FFT SPU turns a Note (key, octave, fine_tune) into a raw pitch value: `FUN_80017424` evaluates the ROM pitch table at `0x800290d8` on the adjusted value `key*256 + fine_tune + pitch-bend accumulator`, with fine_tune supplied by the WAVESET instrument entry. Verified bit-for-bit against the real PSX (13/13 tested input/output pairs), and the SMD Octave value is used raw — no −1 shift — with absolute pitch carried by the instrument's fine_tune.

## Points

- **The FFT SPU pitch conversion (`FUN_80017424`, PC `0x80017340`–`0x80017364`) evaluates the ROM pitch table at `0x800290d8` on the adjusted value `key*256 + fine_tune + pitch-bend accumulator` (fine_tune from the WAVESET instrument entry); all 13 tested input/output pairs match the real PSX bit-for-bit.** — `[S·D·R] 3/3`
  - S: `FUN_80017424` decompilation recorded in `research/working_documents/SYNTH_ACCURACY.md` (2026-04-16); ROM pitch table at `0x800290d8` (transcribed into `smd-player/.../pitch_table.gd` via `probe_constants_invariant.lua`)
  - D: Lua `PITCH_IN` trace — 13 I/O pairs bit-for-bit (doc 2026-04-16)
  - R: `smd-player/addons/exmateria_sound/runtime/shared/note_handler/compute_pitch.gd` + `smd-player/addons/exmateria_sound/runtime/pitch_table.gd` (ROM table dump) + smd-player music parity Gate D / effect-sound Gate B probe pairs (`smd-player/workspace/regression/verify_all.sh`)
  - src: `research/working_documents/SYNTH_ACCURACY.md`
- **The PSX uses the raw SMD octave value with no −1 adjustment — the instrument's fine_tune carries the absolute pitch, so no conventional MIDI octave shift is applied (verified via Lua PITCH_IN traces).** — `[D·R] 2/3`
  - D: Lua `PITCH_IN` trace (doc 2026-04-16)
  - R: `smd-player/addons/exmateria_sound/runtime/sequencer/opcodes/octave.gd` (raw param → `channel.octave`, `bmidi_baseline_byte = octave * 12`; no −1)
  - src: `research/working_documents/SYNTH_ACCURACY.md`
- **On `cure_4` voice 18 (Note-before-Octave, `bmidi_baseline_byte=0`) Godot's pre-fix pitch formula landed `raw_pitch=128` because `pre_pitch_baseline = −19` (inst[1].fine_tune) put `adjusted = −19 < 0` and the then-current `PitchTable.note_to_pitch` clamped `adjusted` to 0 (pitch_table.gd:241-242 at the time), yielding `TABLE[0] >> 6 = 128` — the clamp-to-zero was the immediate emitter of the 32× divergence; the clamp has since been removed, and current `note_to_pitch` lets negative `adjusted` reach the ROM lookups unmasked (matching FFT's `andi` semantics).** — `[D·R] 2/3`
  - D: pre-fix `spu_voice_events.jsonl` voice 18 first KEYON `raw_pitch=128` vs PCSX 4078 + `diag_pitch_formula_inputs` trace (2026-05-14)
  - R: `smd-player/addons/exmateria_sound/runtime/pitch_table.gd` (clamp was :241-242; now absent — `note_to_pitch` :582+ passes negative `adjusted` unmasked) + `runtime/shared/note_handler/compute_pitch.gd` — validated by the pre-fix render + `diag_pitch_formula_inputs` trace
  - src: `research/effect_sound/working_documents/CURE_4_CH2_VOICE_18_PITCH_DEFAULT_INVESTIGATION.md`

## Notes

(empty — user territory)

## Related

- [[SPU Voice Engine]]
- [[WAVESET Instrument Bank]]
- [[SFX Index]]
