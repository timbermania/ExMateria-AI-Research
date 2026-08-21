# PSX Pitch Conversion

How the FFT SPU turns a Note (key, octave, fine_tune) into a raw pitch value: `FUN_80017424` evaluates the ROM pitch table at `0x800290d8` on the adjusted value `key*256 + fine_tune + pitch-bend accumulator`, with fine_tune supplied by the WAVESET instrument entry. Verified bit-for-bit against the real PSX (13/13 tested input/output pairs), and the SMD Octave value is used raw — no −1 shift — with absolute pitch carried by the instrument's fine_tune. The lookup's internal structure (2026-05-16 table dump): `octave_index = (adjusted & 0x7FFF) >> 8` indexes `SEMITONE_LOOKUP` (ROM `0x80029060`) to pick one of 256-entry fine bins in `PITCH_TABLE` (`0x800290D8`, 8448+ entries), and the s16 base is shifted by `6 − OCTAVE_SHIFT_LOOKUP[octave_index]` then masked to 14 bits; the tail rows (octave_index 121/123/125/127) hold semitone=32, pulling lookups into the extended range reachable only from high `pitch_base` values. A 2026-05-17 `reraise_no_music` bisection (`probe_pitch_formula_stages`) proves the whole pitch block bit-exact between the engines at all five intermediates (437/433 rows, cad=1 trace-artifact deficit only), and pins the block's entry gate: it runs only when `CHAN1_PITCH_PRESTAGE` (chan_word_1 bit 0x200) is set, armed by the Note pre-pass, the per-channel-tick idle drain, and the per-tick LFO handler.

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
- **The three FFT pitch lookup tables sit at `OCTAVE_SHIFT_LOOKUP` (128 B @ `0x80028FE8`), `SEMITONE_LOOKUP` (128 B @ `0x80029060`), and `PITCH_TABLE` (16-bit @ `0x800290D8`, at least 8448 entries = 33 semitone bins); `SEMITONE_LOOKUP[112..127]` is `04 05 06 07 08 00 00 00 00 20 02 20 04 20 06 20` — indices 117–120 are zero padding and 121/123/125/127 hold `0x20` (semitone=32) interleaved with even semitones, not the linear `1..7` continuation a naive transcription assumes.** — `[S·D·R] 3/3`
  - S: ROM addresses `0x80028FE8` / `0x80029060` / `0x800290D8` (BATTLE.BIN), recorded in the doc's §1 diff
  - D: `diag_pitch_table_dump.jsonl` PCSX runtime table dump (2026-05-16) — diff vs the Godot copy: `SEMITONE_LOOKUP` differs only at [121],[123],[125],[127] (Godot had 1,3,5,7; ROM has 32,32,32,32), `OCTAVE_SHIFT_LOOKUP` 0 diffs, `PITCH_TABLE` 0 diffs in the shared 3072 entries (3072–8447 absent from Godot pre-fix)
  - R: `smd-player/addons/exmateria_sound/runtime/pitch_table.gd` (`SEMITONE_LOOKUP`, `OCTAVE_SHIFT_LOOKUP`, `TABLE` = 12288-entry superset) + `smd-player/workspace/probes/probe_constants_invariant.lua` (via `harness_lib/constants_invariant_emit.gd`) validates all three tables byte-for-byte against those ROM addresses
  - src: `research/effect_sound/working_documents/PITCH_TABLE_TRUNCATION.md`
- **`midi_to_spu_pitch_lookup` (`FUN_80017424`, entry PC `0x80017424`) computes the pitch as `octave_index = (a0 & 0x7FFF) >> 8`, `semitone = SEMITONE_LOOKUP[octave_index]`, `base = s16(PITCH_TABLE[semitone*256 + (a0 & 0xFF)])`, `result = (base >> (6 − OCTAVE_SHIFT_LOOKUP[octave_index])) & 0x3FFF` — signed s16 shift, no clamping of negative inputs; worked example a0=32085: octave_index=125, semitone=32, TABLE[8277]=0xFBF8 (−1032), shift 6−5=1, result −516 & 0x3FFF = 15868, matching PCSX bit-for-bit.** — `[S·D·R] 3/3`
  - S: `diag_pitch_lookup_arg.lua` BP at PC `0x80017424` confirms PCSX's encoder receives a0=32085; formula walkthrough in doc §3
  - D: protect_no_music capture (2026-05-16) — voice 21 pitch staging cad 470/472: PCSX 15868/7770 vs pre-fix Godot 5573/9124 on bit-identical inputs; post-fix `probe_pitch_register` / `probe_pitch_staging` / `probe_pitch_inputs` all pair 263/263 rows (drift ±1)
  - R: `smd-player/addons/exmateria_sound/runtime/pitch_table.gd` `note_to_pitch` (s16 signed shift, unclamped negative inputs) + mirrored in `fft-sound-driver/src/shared/fft_pitch_tools.cpp` — validated by the protect_no_music pitch probe pairs
  - src: `research/effect_sound/working_documents/PITCH_TABLE_TRUNCATION.md`
- **The semitone=32 extended rows are reached only when `pitch_base & 0x7FFF` falls in `[0x7900,0x79FF]`, `[0x7B00,0x7BFF]`, `[0x7D00,0x7DFF]`, or `[0x7F00,0x7FFF]` (octave_index 121/123/125/127) — high-pitch-base ranges the cure_4 trace never enters, which is why the 3072-entry truncated `PITCH_TABLE` stayed latent on cure_4 while protect_no_music voice 21 (pitch 32085 = 0x7D0D, cad 471/473) broke on it.** — `[S·D·R] 3/3`
  - S: derived from ROM `SEMITONE_LOOKUP` @ `0x80029060` (doc §5)
  - D: protect_no_music capture (2026-05-16) — voice 21 pitch register 32085 at cad 471/473 hits the 0x7D00–0x7DFF window; doc's cure_4 trace shows no entry into these ranges
  - R: `smd-player/addons/exmateria_sound/runtime/pitch_table.gd` — extended rows now implemented (same pitch probe pairs; voice_21 cos_dist 0.0055 → 0.0039 on protect_no_music)
  - src: `research/effect_sound/working_documents/PITCH_TABLE_TRUNCATION.md`
- **On `reraise_no_music` the pitch block of `FUN_80017118` is bit-exact between PCSX and Godot at all five bisection stages — `pitch_base`, `pitch_bend`, `pool_a2`, `midi_sum`, `pitch_result` (BPs `0x80017344`/`0x80017348`/`0x8001734c`/`0x80017354`/`0x8001736c`) — `probe_pitch_formula_stages` pairs at PCSX 437 / Godot 433 rows with only the known cad=1 trace-artifact deficit, ruling the pitch math out of voice_21's chronic cos_dist (0.384 post-probe, was 0.396; voice_19 rms_spec_err 0.034 stays the highest across voices).** — `[S·D·R] 3/3`
  - S: stage BP PCs `0x80017344`/`0x80017348`/`0x8001734c`/`0x80017354`/`0x8001736c` in the FUN_80017118 pitch block (`project-assets/fft-rom/scus_disassembly.txt`; JAL to `midi_to_spu_pitch_lookup` `FUN_80017424` at `0x80017358`)
  - D: `probe_pitch_formula_stages` 5/5-stage pairing + `audio_score.json` voice metrics — `reraise_no_music` (2026-05-17)
  - R: `smd-player/addons/exmateria_sound/runtime/effect_sound/play_sound.gd::_emit_pitch_formula_stage` (five emits in `_drain_prestage_all_slots`, :1702–1706) + `runtime/shared/dispatcher.gd::_evaluate_pitch_formula` — validated by the `probe_pitch_formula_stages` entry in `smd-player/workspace/orchestrator/probe_validation_manifest.py` + the `reraise_no_music` orchestrator audio score
  - src: `research/effect_sound/working_documents/PROBE_PITCH_FORMULA_STAGES_INTRODUCED.md`
- **The pitch block runs only when `CHAN1_PITCH_PRESTAGE` (chan_word_1 bit 0x200) is set on entry — gate `beq` at PC `0x80017338` on the `andi s1, 0x200` at `0x80017334` — and that bit is armed by three sites: the Note opcode handler pre-pass (PC `0x800153D4`), the per-channel-tick idle-timeout drain (PC `0x800152C8`), and the per-tick LFO handler on LFO-swap fire; `probe_pitch_inputs` / `probe_pitch_staging` both pairing at 437/437 on `reraise_no_music` confirms the gate fires the same number of times on both sides, so any divergence the stage probe would surface is in the values, not the gate timing.** — `[S·D·R] 3/3`
  - S: gate `0x80017334`/`0x80017338`; arming sites `0x800153D4` / `0x800152C8` — `project-assets/fft-rom/scus_disassembly.txt`
  - D: `probe_pitch_inputs` + `probe_pitch_staging` 437/437 pairs — `reraise_no_music` (2026-05-17)
  - R: `smd-player/addons/exmateria_sound/runtime/effect_sound/play_sound.gd:1667` guard `(s1 & _SS.CHAN1_PITCH_PRESTAGE) != 0` + `runtime/runtime.gd:461–464` bit clear + `runtime/sequencer/note_handler/note_handler.gd:136–144` arm — validated by the pitch-probe pairs above
  - src: `research/effect_sound/working_documents/PROBE_PITCH_FORMULA_STAGES_INTRODUCED.md`

## Notes

(empty — user territory)

## Related

- [[SPU Voice Engine]]
- [[WAVESET Instrument Bank]]
- [[SFX Index]]
- [[Effect Sound Audio Divergence]]
