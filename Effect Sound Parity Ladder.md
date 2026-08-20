# Effect Sound Parity Ladder

The effect-sound parity ladder is a validation sequence in which a minimal Cure SMD grows one SMD opcode per position; a position is trusted only when every manifest probe PAIRs on the same session and the Godot-rendered audio stays below the 0.09 cos_dist floor against the capture reference. As of iter_0452 (2026-05-19) all 7 positions have PASSED, so the Godot effect-sound synth reproduces `0xBA` ReverbOn, `0xDB`, `0xB0` SlurOn, `0xC4` ADSR_SustainRate, `0xD1` AddPitchBend, and `0xD4` (2-param).

## Points

- **As of iter_0452 all 7 effect-sound parity ladder positions PASS: each position adds one SMD opcode to the minimal Cure SMD — `0xBA` ReverbOn, `0xDB`, `0xB0` SlurOn, `0xC4` ADSR_SustainRate, `0xD1` AddPitchBend, `0xD4` (2-param) — and the Godot synth renders every position with all manifest probes paired on the same session and audio cos_dist below the 0.09 floor.** — `[D·R] 2/3`
  - D: ladder session captures `cure_single_one_note` (iter_0419, baseline 5 events) through `cure_min07_plus_ba_db_b0_c4_d1_d4` (iter_0452), all PASSED (doc 2026-05-19)
  - R: `smd-player/addons/exmateria_sound/runtime/shared/opcodes/_table.gd` (0xB0 SlurOn, 0xBA ReverbOn, 0xC4 Adsr2Sustain, 0xD1 PitchBendAdd, 0xD4 PortamentoInit, 0xDB FlagClearDb) + `runtime/sequencer/opcodes/_table.gd`; validated by Gate B effect-sound PCSX probe pairs (`smd-player/workspace/regression/verify_all.sh`, `dispatcher_refactor_baseline`)
  - src: `research/effect_sound/probes_index.md`

## Notes

(empty — user territory)

## Related

- [[SFX Index]]
- [[Effect Sound Timing]]
- [[FEDS Sound Definition Format]]
- [[PCSX-Redux Capture Rig]]
