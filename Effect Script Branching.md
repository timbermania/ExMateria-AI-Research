# Effect Script Branching

Conditional control flow in E###.BIN effect scripts: besides the unconditional control opcodes (yield, goto, end), the jump table at 0x801B67C8 exposes two branch families — register/count comparisons against the four int16 script registers at EffectState+0x14 (opcodes 16–23, 32, in practice used only by E225.BIN and E306.BIN) and the 4-byte animation-timing/target branches (opcodes 29–31) that read the +0x28 animation frame counter, which op_animate_tick (handler 40, FUN_801a3408) increments at the end of every tick; the register-comparison branches 17–21 use inverted loop-guard semantics (branch when the comparison is FALSE, i.e. keep looping while it holds). Most effect scripts follow a uniform four-phase INIT → MAIN_LOOP → WINDDOWN → END structure that yields to the main loop each frame, with the E001.BIN (Cure) script traced in full below. godot-learning reimplements the effect timeline directly (EffectPhase/PhaseBlock, ADR-0012) and does not implement the script-bytecode interpreter or its branch opcodes.

## Points

- **Effect scripts have two conditional-branching mechanisms: script-register/count comparisons (opcodes 16–23, 32) that are rarely used in practice, and animation-timing branches (opcodes 29–31) which most effects use instead.** — `[S] 1/3`
  - S: jump-table entries at 0x801B67C8, handlers 0x801a2770–0x801a29e0 and 0x801a2c28–0x801a2cfc, per `project-assets/fft-rom/{scus,battle}_decompilation.c`
  - R: none — effect-script branch opcodes not present in godot-learning (probed godot-learning/src, godot-learning/tests, smd-player/addons/exmateria_sound, fft-sound-driver; the effects pipeline models the timeline directly, not script bytecode)
  - src: `research/working_documents/WIP_SCRIPT_REGISTERS.md`
- **Opcodes 29–31 are 4-byte branch instructions whose final 2 bytes are a branch target, not an argument or a separate opcode: 29 op_branch_anim_done (0x801a2c28) exits the main loop when the animation timeline ends, 30 op_branch_anim_done_complex (0x801a2c7c) does the same with extended threshold calculation, and 31 op_branch_target_type (0x801a2cfc) skips the entire effect when the target type is wrong.** — `[S] 1/3`
  - S: 0x801a2c28 / 0x801a2c7c / 0x801a2cfc, per `project-assets/fft-rom/battle_decompilation.c`
  - R: none — opcodes 29–31 not present in godot-learning (probed godot-learning/src, godot-learning/tests, smd-player/addons/exmateria_sound, fft-sound-driver)
  - src: `research/working_documents/WIP_SCRIPT_REGISTERS.md`
- **The animation frame counter at EffectState+0x28 is incremented by op_animate_tick (handler 40, FUN_801a3408) at the end of each tick, and handler 40 also processes 5 emitter channels from the animation timeline data, spawning emitters when timeline triggers fire and playing sound effects on cue.** — `[S] 1/3`
  - S: FUN_801a3408 decompilation, per `project-assets/fft-rom/battle_decompilation.c`
  - R: none — the EffectState+0x28 script-tick counter not present in godot-learning (probed godot-learning/src, godot-learning/tests; the effect clock is driven by PhaseBlock/EffectPhase, not by script opcodes)
  - src: `research/working_documents/WIP_SCRIPT_REGISTERS.md`
- **Every effect script follows a 4-phase lifecycle: INIT (set up textures, validate target type — a wrong target jumps straight to END), MAIN_LOOP (spawn and animate, yielding to the main loop each frame), WINDDOWN (stop spawning, let existing particles finish dying), and END (cleanup and exit via op 4).** — `[S] 1/3`
  - S: E001.BIN trace plus handler addresses (op 31 @ 0x801a2cfc, op 30 @ 0x801a2c7c, op 22 @ 0x801a2998, op 0 @ 0x801a2238, op 4 @ 0x801a236c), per `project-assets/fft-rom/battle_decompilation.c`
  - R: none — no INIT/MAIN_LOOP/WINDDOWN/END script lifecycle present in godot-learning (probed godot-learning/src, godot-learning/tests)
  - src: `research/working_documents/WIP_SCRIPT_REGISTERS.md`
- **E001.BIN (Cure) complete script trace: INIT 0x00–0x06 (op 5 set_texture_page arg 0x27; op 31 branch_target_type target 0x22), MAIN_LOOP 0x08–0x14 (op 30 target 0x16; op 41; op 36; op 37 update_all_particles; op 0 goto_yield target 0x08), WINDDOWN 0x16–0x1E (op 37; op 22 branch_count_eq arg 0 target 0x22; op 0 goto_yield target 0x16), END 0x22 (op 4 end).** — `[S] 1/3`
  - S: E001.BIN script section, parsed with the corrected instruction sizes in `scripts/parse_effect_script.py` (2026-05-10)
  - R: none — E001 script-bytecode trace not present in godot-learning (probed godot-learning/src, godot-learning/tests; E001 is driven through the parsed timeline pipeline, e.g. `godot-learning/tests/EffectSoundCaptureTest.gd`)
  - src: `research/working_documents/WIP_SCRIPT_REGISTERS.md`
- **Particle-count branch opcodes: op 22 branch_count_eq (0x801a2998, 6 bytes) branches when active_particle_count == arg, and op 23 branch_count_gt (0x801a29e0, 6 bytes) when active_particle_count > arg; the typical usage is op 22 arg=0 in the winddown phase as the "are all particles dead?" cleanup test.** — `[S] 1/3`
  - S: 0x801a2998 / 0x801a29e0, per `project-assets/fft-rom/battle_decompilation.c`
  - R: none — branch_count_eq/gt not present in godot-learning (probed godot-learning/src, godot-learning/tests, smd-player/addons/exmateria_sound, fft-sound-driver)
  - src: `research/working_documents/WIP_SCRIPT_REGISTERS.md`
- **EffectState+0x14–0x1B holds 4 general-purpose int16 script registers written by op 16 (0x801a2770, 4 bytes, reg[idx] = arg) and incremented by op 32 (0x801a2d48, 2 bytes, reg[idx]++), with the register index taken from bits 6–7 of opcode byte 1; only E225.BIN and E306.BIN use these in practice, and E306.BIN uses reg[0] as a per-frame loop counter (incremented each frame, branched on at the top of the loop).** — `[S] 1/3`
  - S: 0x801a2770 / 0x801a2d48 decompilations plus E225.BIN / E306.BIN script sections, per `project-assets/fft-rom/battle_decompilation.c`
  - R: none — script registers not present in godot-learning (probed godot-learning/src, godot-learning/tests, smd-player/addons/exmateria_sound, fft-sound-driver)
  - src: `research/working_documents/WIP_SCRIPT_REGISTERS.md`
- **Register-comparison branch opcodes 17–21 use inverted loop-guard semantics — they branch to the target when the comparison is FALSE, i.e. "keep looping while the condition holds, exit when it becomes false": 17 eq @ 0x801a27b0, 18 lt @ 0x801a2810, 19 gt @ 0x801a2870, 20 le @ 0x801a28d4, 21 ge @ 0x801a2938; the decompiled op_branch_reg_lt reads the register index from bits 6–7 of instr[1], the int16 threshold at instr+2, and the u16 branch target at instr+4, consuming 6 bytes.** — `[S] 1/3`
  - S: op_branch_reg_lt 0x801a2810 decompilation (with 0x801a27b0 / 0x801a2870 / 0x801a28d4 / 0x801a2938), per `project-assets/fft-rom/battle_decompilation.c`
  - R: none — opcodes 17–21 not present in godot-learning (probed godot-learning/src, godot-learning/tests, smd-player/addons/exmateria_sound, fft-sound-driver)
  - src: `research/working_documents/WIP_SCRIPT_REGISTERS.md`

## Notes

(empty — user territory)

## Related

- [[Effect Execution Model]]
- [[Particle Runtime State]]
- [[E001.BIN Memory Mapping]]
- [[Effect Frame Pacing]]
- [[Effect Sound Timing]]
