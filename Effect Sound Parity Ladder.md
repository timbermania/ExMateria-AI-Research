# Effect Sound Parity Ladder

The effect-sound parity ladder is a validation sequence in which a minimal Cure SMD grows one SMD opcode per position; a position is trusted only when every manifest probe PAIRs on the same session and the Godot-rendered audio stays below the 0.09 cos_dist floor against the capture reference. As of iter_0452 (2026-05-19) all 7 positions have PASSED, so the Godot effect-sound synth reproduces `0xBA` ReverbOn, `0xDB`, `0xB0` SlurOn, `0xC4` ADSR_SustainRate, `0xD1` AddPitchBend, and `0xD4` (2-param).

## Points

- **As of iter_0452 all 7 effect-sound parity ladder positions PASS: each position adds one SMD opcode to the minimal Cure SMD — `0xBA` ReverbOn, `0xDB`, `0xB0` SlurOn, `0xC4` ADSR_SustainRate, `0xD1` AddPitchBend, `0xD4` (2-param) — and the Godot synth renders every position with all manifest probes paired on the same session and audio cos_dist below the 0.09 floor.** — `[S·D·R] 3/3`
  - S: jump-table entries for the two semantics-pending opcodes — 0xDB = `feds_opcode_jump_table[0x5B]` (`0xDB − 0x80`), 0xD4 = `feds_opcode_jump_table[0x54]` — `ghidra_dump/BATTLE_BIN/data_segments.txt` (per OPEN_QUESTIONS, 2026-05-03)
  - D: ladder session captures `cure_single_one_note` (iter_0419, baseline 5 events) through `cure_min07_plus_ba_db_b0_c4_d1_d4` (iter_0452), all PASSED (doc 2026-05-19)
  - D: position-iteration map per `research/effect_sound/working_documents/FEDS_OPCODE_TABLE.md` (2026-05-03): pos 1 = 0x90 EndBar + 0x94 Octave + 0xAC Instrument + 0xE0 Dynamics (iter_0419, cos_dist < 0.09), pos 2 = 0xBA ReverbOn (iter_0435), pos 3 = 0xDB, 0-param (iter_0448), pos 4 = 0xB0 SlurOn (iter_0449), pos 5 = 0xC4 ADSR_SustainRate, first parameterized (iter_0450), pos 6 = 0xD1 AddPitchBend, 1-param tier complete (iter_0451), pos 7 = 0xD4, first 2-param (iter_0452)
  - R: `smd-player/addons/exmateria_sound/runtime/shared/opcodes/_table.gd` (0xB0 SlurOn, 0xBA ReverbOn, 0xC4 Adsr2Sustain, 0xD1 PitchBendAdd, 0xD4 PortamentoInit, 0xDB FlagClearDb) + `runtime/sequencer/opcodes/_table.gd`; validated by Gate B effect-sound PCSX probe pairs (`smd-player/workspace/regression/verify_all.sh`, `dispatcher_refactor_baseline`)
  - src: `research/effect_sound/probes_index.md`

## Notes

(empty — user territory)

## Related

- [[SFX Index]]
- [[Effect Sound Timing]]
- [[FEDS Sound Definition Format]]
- [[PCSX-Redux Capture Rig]]
