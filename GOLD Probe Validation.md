# GOLD Probe Validation

The effects-parity worktree defines 8 GOLD probes — PCSX breakpoints each paired with a Godot trace mirror and registered in the `VALIDATION_MANIFEST` — forming a layered validation chain (a layer-N probe is trustworthy only when every lower layer has PAIRED on the same session) that gates every effect-sound ladder session. These 8 probes are the only probe outputs treated as authoritative; the ~76 additional worktree probes are unverified hypotheses whose data may inform forensics but never evidence. As of the 2026-05-03 bridge doc, GOLD covers layers 1–4 (play_sound entry, opcode dispatch, per-opcode handlers, slot-struct field writes; SPU-register and composite layers 5–6 not yet covered), and all 8 PAIR on every ladder session through position 7.

## Points

- **GOLD probe 1 `probe_play_sound_call` (layer 1, BP `0x800125C0` — the audio-enabled gate inside `play_sound` `0x800125A8`) proves trigger cadence: FFT and Godot call `play_sound` the same number of times in the same order; the check is pure cadence because PCSX `a0=sound_id` and Godot `pair_idx` are not comparable.** — `[S·D·R] 3/3`
  - S: `0x800125C0` (gate inside `play_sound` `0x800125A8`) — `ghidra_dump/BATTLE_BIN/functions/` (per PROBE_TO_DISASM.md)
  - D: GOLD probe 1 `probe_play_sound_call.lua` (BP `0x800125C0`, schema `{call_index}`), PAIRED on every ladder session through position 7 (2026-05-03)
  - R: `smd-player/addons/exmateria_sound/runtime/effect_sound/play_sound.gd` (`_probe_play_sound_call_count`, GOLD #1 mirror, emits the `play_sound` trace row) — validated by the `probe_play_sound_call` entry in `smd-player/workspace/orchestrator/probe_validation_manifest.py` + `godot-learning/tests/EffectSoundCaptureTest.gd`
  - src: `research/effect_sound/working_documents/PROBE_TO_DISASM.md`
- **GOLD probe 2 `probe_event_dispatch` (layer 2, BP `0x800153A4` — the per-event byte-fetch site inside `smd_interpreter_tick` `0x80015324`) proves the bytecode walk: FFT dispatches the same opcode events in the same order as Godot; the `byte` field is comparable because both sides walk the same bytecode (FFT `a1` = Godot `NoteEvent.velocity` / `OpcodeEvent.opcode`), and delta-time bytes are consumed inside the Note path at `0x800153D8` and never pass through this fetch site, so the cadence is per-event, not per-byte.** — `[S·D·R] 3/3`
  - S: `0x800153A4` (per-event byte-fetch site), `smd_interpreter_tick` entry `0x80015324`, delta-time consumption `0x800153D8` — `ghidra_dump/BATTLE_BIN/functions/` (per PROBE_TO_DISASM.md)
  - D: GOLD probe 2 `probe_event_dispatch.lua` (schema `{call_index, event_type ∈ {"note","opcode"}, byte}`), PAIRED on every ladder session through position 7 (2026-05-03)
  - R: `smd-player/addons/exmateria_sound/runtime/shared/dispatcher.gd` (GOLD #2 mirror at the `0x800153A4` dispatch site, emits `NoteEvent`/`OpcodeEvent` rows with the walked byte) — validated by the `probe_event_dispatch` entry in `smd-player/workspace/orchestrator/probe_validation_manifest.py` + `godot-learning/tests/EffectSoundCaptureTest.gd`
  - src: `research/effect_sound/working_documents/PROBE_TO_DISASM.md`
- **GOLD probe 3 `probe_note_handler` (layer 3, BP `0x80015428` — confluence point of both branches inside the Note path) proves Note decode: FFT enters the Note handler with the same `delta_time` Godot's `_handle_note` sees, where `delta_time = read16(s0+0x72)` (chan+0x72, written at `L80015418` or `L80015424`) and `relative_key = (chan+0x7B − chan+0x7C) & 0xff = TABLE1[data_byte]` — confirming the FFT byte-transform pipeline (TABLE2 = `DAT_80028D8C` = duration values) is mirrored by Godot's `DELTA_TIME_TABLE`.** — `[S·D·R] 3/3`
  - S: `0x80015428` (Note-path confluence), `L80015418`/`L80015424` (chan+0x72 writers), `DAT_80028D8C` (TABLE2 duration values) — `ghidra_dump/BATTLE_BIN/functions/` (per PROBE_TO_DISASM.md)
  - D: GOLD probe 3 `probe_note_handler.lua` (schema `{call_index, delta_time, relative_key}`), PAIRED on every ladder session through position 7 (2026-05-03)
  - R: `smd-player/addons/exmateria_sound/runtime/shared/note_handler/note_handler.gd` (GOLD #3 mirror, emits `delta_time`/`relative_key`) + `smd-player/addons/exmateria_sound/runtime/smd_opcodes.gd` (`DELTA_TIME_TABLE` = the TABLE2 mirror) — validated by the `probe_note_handler` entry in `smd-player/workspace/orchestrator/probe_validation_manifest.py` + `godot-learning/tests/EffectSoundCaptureTest.gd`
  - src: `research/effect_sound/working_documents/PROBE_TO_DISASM.md`
- **GOLD probe 4 `probe_note_post_state` (layer 4, BP `0x80015494` — FIRST READ of chan+0x0 after the Note handler's bit set at `L80015474`) proves the Note handler's slot-state side effects: after the Note handler runs, bits `0x80` (NOTE_FIRED) + `0x100` (PITCH_REQ) are set in chan+0x0 AND bit `0x200` (CHAN1_VOL_REQ) is set in chan+0x2 — both sides should emit masks `0x180` (chan+0x0) and `0x200` (chan+0x2); the probe compares per-channel masks, not raw memory.** — `[S·D·R] 3/3`
  - S: `0x80015494` (first read of chan+0x0 post-Note), `L80015474` (Note-handler bit set) — `ghidra_dump/BATTLE_BIN/functions/` (per PROBE_TO_DISASM.md)
  - D: GOLD probe 4 `probe_note_post_state.lua` (schema `{call_index, chan_0_mask_180, chan_2_mask_200}`), PAIRED on every ladder session through position 7 (2026-05-03)
  - R: `smd-player/addons/exmateria_sound/runtime/shared/note_handler/note_handler.gd` (GOLD #4 mirror, emits the chan+0x0/chan+0x2 masks) + `smd-player/addons/exmateria_sound/runtime/shared/slot_state.gd` (`FLAG_VOL_UPDATE = 0x200`, set by the Note handler in `dispatcher._handle_note` matching FFT's atomic order) — validated by the `probe_note_post_state` entry in `smd-player/workspace/orchestrator/probe_validation_manifest.py` + `godot-learning/tests/EffectSoundCaptureTest.gd`
  - src: `research/effect_sound/working_documents/PROBE_TO_DISASM.md`
- **GOLD probe 5 `probe_opcode_endbar` (layer 3, BP `0x800158F8` = `smd_end_bar` entry, opcode 0x90 handler) proves 0x90 EndBar dispatch parity: FFT enters the EndBar handler the same number of times as Godot's `_handle_opcode 0x90:` case, and `is_loop_active = (slot+0x1C != 0)` matches Godot's `saved_loop_target_pos != 0` (FFT branch decision at `L80015900`).** — `[S·D·R] 3/3`
  - S: `0x800158F8` (`smd_end_bar` entry), `L80015900` (loop-active branch) — `ghidra_dump/BATTLE_BIN/functions/` (per PROBE_TO_DISASM.md)
  - D: GOLD probe 5 `probe_opcode_endbar.lua` (schema `{call_index, is_loop_active}`), PAIRED on every ladder session through position 7 (2026-05-03)
  - R: `smd-player/addons/exmateria_sound/runtime/shared/opcodes/endbar.gd` (GOLD #5 mirror, emits `is_loop_active`) + `smd-player/addons/exmateria_sound/runtime/shared/channel_state.gd` (`saved_loop_target_pos`) — validated by the `probe_opcode_endbar` entry in `smd-player/workspace/orchestrator/probe_validation_manifest.py` + `godot-learning/tests/EffectSoundCaptureTest.gd`
  - src: `research/effect_sound/working_documents/PROBE_TO_DISASM.md`
- **GOLD probe 6 `probe_opcode_dynamics` (layer 3, BP `0x80016614` = `smd_dynamics` entry, opcode 0xE0 handler) proves 0xE0 Dynamics dispatch parity: FFT enters the 0xE0 Dynamics handler the same as Godot's `0xE0:` case, and `dynamics_byte = read8(a0)` matches Godot's `op.params[0]` bit-exact.** — `[S·D·R] 3/3`
  - S: `0x80016614` (`smd_dynamics` entry) — `ghidra_dump/BATTLE_BIN/functions/` (per PROBE_TO_DISASM.md)
  - D: GOLD probe 6 `probe_opcode_dynamics.lua` (schema `{call_index, dynamics_byte}`), PAIRED on every ladder session through position 7 (2026-05-03)
  - R: `smd-player/addons/exmateria_sound/runtime/shared/opcodes/dynamics.gd` (GOLD #6 mirror, emits `dynamics_byte` from `op.params[0]`) — validated by the `probe_opcode_dynamics` entry in `smd-player/workspace/orchestrator/probe_validation_manifest.py` + `godot-learning/tests/EffectSoundCaptureTest.gd`
  - src: `research/effect_sound/working_documents/PROBE_TO_DISASM.md`
- **GOLD probe 7 `probe_opcode_instrument` (layer 3, BP `FUN_80015E30` = `smd_instrument` entry, opcode 0xAC handler) proves 0xAC Instrument dispatch parity: FFT enters the 0xAC Instrument handler the same as Godot's `0xAC:` case, and `instrument_byte = read8(a0)` (after `move s0,a0`) matches Godot's `op.params[0]` bit-exact.** — `[S·D·R] 3/3`
  - S: `FUN_80015E30` (`smd_instrument` entry) — `ghidra_dump/BATTLE_BIN/functions/` (per PROBE_TO_DISASM.md)
  - D: GOLD probe 7 `probe_opcode_instrument.lua` (schema `{call_index, instrument_byte}`), PAIRED on every ladder session through position 7 (2026-05-03)
  - R: `smd-player/addons/exmateria_sound/runtime/shared/opcodes/instrument.gd` (GOLD #7 mirror, emits `instrument_byte` from `op.params[0]`) — validated by the `probe_opcode_instrument` entry in `smd-player/workspace/orchestrator/probe_validation_manifest.py` + `godot-learning/tests/EffectSoundCaptureTest.gd`
  - src: `research/effect_sound/working_documents/PROBE_TO_DISASM.md`
- **GOLD probe 8 `probe_opcode_octave` (layer 3, BP `LAB_800159F0` = `smd_octave` entry, opcode 0x94 handler) proves 0x94 Octave dispatch parity: FFT enters the 0x94 Octave handler the same as Godot's `0x94:` case, and `octave_byte = read8(a0)` matches Godot's `op.params[0]` bit-exact.** — `[S·D·R] 3/3`
  - S: `LAB_800159F0` (`smd_octave` entry) — `ghidra_dump/BATTLE_BIN/functions/` (per PROBE_TO_DISASM.md)
  - D: GOLD probe 8 `probe_opcode_octave.lua` (schema `{call_index, octave_byte}`), PAIRED on every ladder session through position 7 (2026-05-03)
  - R: `smd-player/addons/exmateria_sound/runtime/shared/opcodes/octave.gd` (GOLD #8 mirror, emits `octave_byte` from `op.params[0]`) — validated by the `probe_opcode_octave` entry in `smd-player/workspace/orchestrator/probe_validation_manifest.py` + `godot-learning/tests/EffectSoundCaptureTest.gd`
  - src: `research/effect_sound/working_documents/PROBE_TO_DISASM.md`

## Notes

(empty — user territory)

## Related

- [[SFX Index]]
- [[Effect Sound Parity Ladder]]
- [[SPU Voice Engine]]
- [[FEDS Sound Definition Format]]
- [[Post-Walker Lookahead]]
- [[PCSX-Redux Capture Rig]]
