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

## Notes

(empty — user territory)

## Related

- [[SPU Voice Engine]]
- [[WAVESET Instrument Bank]]
- [[SFX Index]]
