# Event Jump Opcodes

Event instructions `0xD0`/`0xD1`/`0xD3` are label-search jumps, not byte offsets: the only operand consumed is a label id, and `FUN_80149d6c` (dispatched inline in the event loop) scans the bytecode opcode-by-opcode for a label-marker — `0xD2` Forward Target, `0xD4` Forward If Zero Target, `0xD5` Back Target — and overwrites the event PC with `found_index + 2`; a failed forward scan ends the fiber (`0xDB` Event End as the scan terminator). `0xD0` is the conditional half of the comparison family, jumping iff `var[0] == 0` (i.e. the `0xA0`–`0xA5` comparison was false), closing the `if (cond) { block }` pattern. Static decode completed 2026-06-30; the family is never executed in the scenario chunks we own, so any reimplementation test must be authored synthetically.

## Points

- **`0xD0`/`0xD1`/`0xD3` do NOT encode a byte distance — dispatched inline in the event loop (`0x80144494`–`0x801444fc`), all three funnel into `FUN_80149d6c`, which overwrites the event PC (`s8`) with its return: the only operand consumed is `chunk[+1]` = a label id, and `FUN_80149d6c` scans the bytecode opcode-by-opcode (stepping via the per-opcode operand-length table at ~`0x8014d170`) for a label-marker opcode whose operand byte equals the requested id, returning `found_index + 2` as the new PC; "not found" terminates the fiber (`event_fiber_mark_complete`).** — `[S] 1/3`
  - S: event-loop dispatch `0x80144494`–`0x801444fc`, `FUN_80149d6c`, operand-length table ~`0x8014d170` (`project-assets/fft-rom/battle_disassembly.txt`)
  - src: `research/wiki_articles/event_instruction_a0_d5_variable_readers.md`
- **The label-marker (anchor) opcodes are `0xD2` Forward Target (searched by `0xD1` and `0xD0`), `0xD4` Forward If Zero Target (searched by `0xD0`), and `0xD5` Back Target (searched by `0xD3`); `0xDB` Event End acts as the forward-scan terminator ("not found").** — `[S] 1/3`
  - S: `event_opcodes.json` catalog names (Forward Target / Forward If Zero Target / Back Target / Event End) + `FUN_80149d6c` scan (`battle_disassembly.txt`)
  - S: FFTPatcher `EventCommands.xml` master catalog — {D2} Forward Target, {D4} Forward If Zero Target, {D5} Back Target, {DB} Event End
  - src: `research/wiki_articles/event_instruction_a0_d5_variable_readers.md`
- **`0xD0` Jump Forward If Zero jumps iff `var[0] == 0` — i.e. when the `0xA0`–`0xA5` comparison was **false** — reading the exact word `0x8005771C` (= `var[0]`) that `FUN_80149f10` stores its boolean to (`0x801444a4`–`0x801444b0`), then scanning forward from `PC+2` (`s8+2`) for a `0xD2` or `0xD4` label; a true result falls through, giving the pattern `if (cond) { block }` (false → jump past the block).** — `[S] 1/3`
  - S: `0x801444a4`–`0x801444c8` (`var[0]` load, `bne v0,zero` at `0x801444b8`, `ori a1,zero,0xd2` at `0x801444c0`, `ori a2,zero,0xd4` at `0x801444c8`) (`battle_disassembly.txt`)
  - src: `research/wiki_articles/event_instruction_a0_d5_variable_readers.md`
- **`0xD1` is an unconditional forward jump scanning from `PC+2` for a `0xD2` label only (`a2=-1`), while `0xD3` is an unconditional backward jump scanning from index 0 up to the current PC for a `0xD5` label only (passing `a1=0xd5` selects the backward branch in `FUN_80149d6c`).** — `[S] 1/3`
  - S: `FUN_80149d6c` direction/`a2` selection; `0xD1`/`0xD3` dispatch within `0x80144494`–`0x801444fc` (`battle_disassembly.txt`)
  - src: `research/wiki_articles/event_instruction_a0_d5_variable_readers.md`
- **The Godot reimplementation's ScenarioVM pre-bakes each event chunk into an instruction array (`_insts` of name-keyed dicts) and dispatches handlers by opcode name string (from `event_opcodes.json`), with `ctx.pc` an instruction index into `_insts` (NOT a byte offset) and handlers reading params via `_params_dict(inst)`.** — `[R] 1/3`
  - R: `godot-learning/src/scenarios/ScenarioVM.gd` (`_insts`, name-keyed dispatch, `_params_dict`; no named test)
  - src: `research/wiki_articles/event_instruction_a0_d5_variable_readers.md`

## Notes

(empty — user territory)

## Related

- [[Variable Comparison Opcodes]]
- [[Wait Value Opcode]]
- [[Event Opcode Catalog]]
