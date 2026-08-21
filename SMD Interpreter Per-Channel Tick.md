# SMD Interpreter Per-Channel Tick

FFT's `smd_interpreter_tick` walks the 8 channel structs (stride 0x160) in one per-channel loop; each iteration passes two selectivity gates (chan_word_0 == 0, note_duration != 0) before the bytecode walker, and every exit — both gate fails and the post-walker fall-through — converges on the loop-continue label `LAB_80015814`, so a breakpoint there counts every channel iteration. PCSX's `tick_entry` (BP @ 0x8001536C, loop top) and `gate_skip` (BP @ 0x80015814) probes therefore track each other 1:1. The Godot port's `cadence_body` mirrors that shape: the `smd_interpreter_tick_entry` emit fires at the top of the function before all early-returns, and `smd_interpreter_gate_skip` fires at every exit (including the Godot-only `stream_end_fired` early-return, classified by state, and the post-walker fall-through). An EndBar (0x90) with no active loop target zeroes a channel's `chan_word_0` but leaves the struct in the array, so such non-null channels keep iterating and hit gate-1 immediately.

## Points

- **The PCSX `smd_interpreter_tick` per-channel loop (body 0x8001536C–0x80015828, 8 channel structs, stride 0x160) reaches the loop-continue label LAB_80015814 via exactly three paths: gate-1 fail (`beq v0,zero` @ 0x80015374, chan_word_0 == 0), gate-2 fail (`bne v0,zero` @ 0x80015384, note_duration != 0), and the post-walker fall-through (`beq s4,zero` @ 0x80015718, walker-completed exit) — so on PCSX every channel iteration, including the walker path, lands on the gate-skip BP.** — `[S·D·R] 3/3`
  - S: `0x80015374` (gate-1 `beq`), `0x80015384` (gate-2 `bne`), `0x80015718` (post-walker `beq`), `LAB_80015814` (XREF[3]: 80015374(j), 80015384(j), 80015718(j); `addiu s0,s0,0x160`) — `project-assets/fft-rom/scus_disassembly.txt`
  - D: reraise_no_music last_run capture (2026-05-17): `probe_smd_interpreter_gate_skip` PCSX 2502 rows, `which_gate` distribution Counter({1: 1290, 2: 1212, 3: 0})
  - R: `smd-player/addons/exmateria_sound/runtime/shared/dispatcher.gd` `cadence_body` (gate-2 else-branch + post-walker fall-through emits) + `runtime/effect_sound/probes/probe_emit.gd::emit_smd_interpreter_gate_skip_for_state` — validated by the `probe_smd_interpreter_gate_skip` pair entry in `smd-player/workspace/orchestrator/probe_validation_manifest.py`
  - src: `research/effect_sound/working_documents/SMD_INTERPRETER_GATE_SKIP_EARLY_RETURN_DEFICIT.md`
- **On PCSX the two per-channel probes track each other 1:1 by construction: `probe_smd_interpreter_tick_entry` (BP @ 0x8001536C) fires once per channel iteration at the loop top, before both gates, and `probe_smd_interpreter_gate_skip` (BP @ 0x80015814) fires once per iteration at any of the three convergence paths, so both equal the per-channel loop-iteration count (reraise_no_music: 2501 vs 2502, Δ+1 init artifact; the post-walker fall-through adds no extra tick_entry row because that BP fires only at the loop top and the walker runs inside the iteration).** — `[S·D·R] 3/3`
  - S: `0x8001536C` (tick_entry BP, loop top), `0x80015814` (gate_skip BP) — `smd-player/workspace/probes/probe_smd_interpreter_tick_entry.lua` + `smd-player/workspace/probes/probe_smd_interpreter_gate_skip.lua`, `project-assets/fft-rom/scus_disassembly.txt`
  - D: reraise_no_music last_run capture (2026-05-17): tick_entry PCSX 2501 vs Godot 2464 (−37), gate_skip PCSX 2502 vs Godot 2379 (−123), while `probe_per_channel_tick_entry` (BP @ 0x800151B4) pairs 2496/2496
  - R: `smd-player/addons/exmateria_sound/runtime/shared/dispatcher.gd` — the `smd_interpreter_tick_entry` emit now lives at the top of `cadence_body`, before both early-returns, mirroring the pre-gate BP placement — validated by the `probe_smd_interpreter_tick_entry` pair entry in `smd-player/workspace/orchestrator/probe_validation_manifest.py`
  - src: `research/effect_sound/working_documents/SMD_INTERPRETER_GATE_SKIP_EARLY_RETURN_DEFICIT.md`
- **An EndBar (0x90) with no active loop target zeroes the channel's `chan_word_0` halfword (`sh zero, 0x0(a2)` @ PC 0x800159CC, inside `smd_end_bar` @ LAB_800158F8) but the channel struct stays in the 8-slot chan array: PCSX keeps iterating it and such non-null cw0==0 channels hit gate-1 immediately — on reraise_no_music 42 of PCSX's 1290 gate-1 rows are these post-EndBar non-null channels, the remaining 1248 being null slots.** — `[S·D·R] 3/3`
  - S: `0x800159CC` (`sh zero, 0x0(a2)`, XREF from 800159a8(j)), `LAB_800158F8` (0x90 handler) — `project-assets/fft-rom/scus_disassembly.txt`
  - D: reraise_no_music gate_skip distribution (2026-05-17): gate-1 = 1248 null-slot rows + 42 non-null cw0==0 rows
  - R: `smd-player/addons/exmateria_sound/runtime/shared/opcodes/endbar.gd` (clear path sets `channel.channel_word_0 = 0`, mirroring L80015924..L800159CC) + `runtime/shared/dispatcher.gd` (the `channel_word_0 == 0` early-return now emits which_gate=1 before returning) — validated by the `probe_opcode_endbar` + `probe_smd_interpreter_gate_skip` pair entries in `smd-player/workspace/orchestrator/probe_validation_manifest.py`
  - src: `research/effect_sound/working_documents/SMD_INTERPRETER_GATE_SKIP_EARLY_RETURN_DEFICIT.md`
- **Godot's `cadence_body` now fires `smd_interpreter_tick_entry` once at the top of the function (before any early-return) and `smd_interpreter_gate_skip` at every exit — the Godot-only `stream_end_fired` early-return (no PCSX equivalent; set when the bytecode exhausts with note_duration == 0) is classified by state (which_gate=1 if cw0==0, else 2), the `cw0 == 0` early-return emits which_gate=1, and the gate-2 else-branch plus the post-walker fall-through both go through the state-classifying helper — so each channel iteration contributes exactly 1 tick_entry row + 1 gate_skip row matching the PCSX BP shape; all emits only read channel state, making this a trace-fidelity fix with no audio-affecting state change.** — `[D·R] 2/3`
  - D: pre-fix reraise_no_music capture (2026-05-17): tick_entry Godot 2464 vs PCSX 2501 (−37), gate_skip Godot 2379 vs 2502 (−123 = −42 gate-1 non-null + −81 gate-2 incl. ~84 post-walker fall-through rows, matching `probe_smd_interpreter_post_gates` 84/84); post-fix expectation 2496/2496
  - R: `smd-player/addons/exmateria_sound/runtime/shared/dispatcher.gd` (top-of-function tick_entry emit + gate_skip emits at each exit path) + `runtime/effect_sound/probes/probe_emit.gd::emit_smd_interpreter_gate_skip_for_state` — validated by the `probe_smd_interpreter_tick_entry` / `probe_smd_interpreter_gate_skip` pair entries in `smd-player/workspace/orchestrator/probe_validation_manifest.py`
  - src: `research/effect_sound/working_documents/SMD_INTERPRETER_GATE_SKIP_EARLY_RETURN_DEFICIT.md`

## Notes

(empty — user territory)

## Related

- [[SFX Index]]
- [[KON KOFF IRQ Phasing]]
- [[Post-Walker Lookahead]]
- [[MIPS SPU Interleaving]]
- [[LFO Handler Probe Cadence]]
- [[GOLD Probe Validation]]
- [[SMD Opcodes]]
