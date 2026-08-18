# Variable Comparison Opcodes

Event instructions `0xA0`–`0xA5` are the event VM's `if` conditions: a 2-register signed ALU over two *fixed* scratch words — operand A = `var[0]` (`0x8005771C`), operand B = `var[1]` (`0x80057720`) — that read no operand bytes of their own (handler `FUN_80149f10`) and write the boolean result back into `var[0]` for the `0xD0` Jump Forward If Zero to test. Comparing a named variable therefore requires staging the operands into `var0`/`var1` first via the `0xB0`–`0xBE` writers. The odd/even immediate-vs-Var distinction lives only in the writer handler, and the resolver `FUN_8014a2ec` has exactly 2 callers (both in the writer). Static decode completed 2026-06-30; neither `scenario_1` nor `scenario_0001_setup` ever executes one of these opcodes (all such bytes sit in the mis-decoded dialog-text region).

## Points

- **`0xA0`–`0xA5` read NO operand bytes: handler `FUN_80149f10` (dispatched at `0x80143d60` with `a0` = opcode byte, pre-filtered by `addiu v0,s4,-0xa0; sltiu v0,v0,6`) is a 2-register ALU over two fixed scratch words — operand A = `var[0]` at `0x8005771C`, operand B = `var[1]` at `0x80057720` — loaded from the variable-file base `*0x80165f9c`, the same pointer the `0xB0`–`0xBE` writers use (Ghidra's `camera_scratch_struct_base` / `DAT_80057720` labels are misnomers; this is the event-script comparison register block).** — `[S] 1/3`
  - S: `FUN_80149f10`, dispatch `0x80143d60`, scratch words `0x8005771C`/`0x80057720`, base `*0x80165f9c` (`project-assets/fft-rom/battle_disassembly.txt`)
  - src: `research/wiki_articles/event_instruction_a0_d5_variable_readers.md`
- **The opcode→comparison mapping (A = `[base+0]`, B = `[base+4]`) is `0xA0` = ≤, `0xA1` = ≥, `0xA2` = ==, `0xA3` = !=, `0xA4` = <, `0xA5` = >, all signed (`slt`) with the operand load order swapped per block to realize the sense (e.g. `0xA0`: `slt v1,B,A; xori v1,1` = !(B<A); `0xA5`: `slt v1,B,A` = B<A = A>B).** — `[S] 1/3`
  - S: `0x80149f34`/`0x80149f3c` (≤), `0x80149f60`/`0x80149f68` (≥), `0x80149f8c`/`0x80149f94` (==), `0x80149fb8`/`0x80149fc0` (!=), `0x80149fe4` (<), `0x8014a008` (>) (`battle_disassembly.txt`)
  - S: FFTPatcher `EventCommands.xml` master catalog — {A0} Variable <=, {A1} Variable >=, {A2} Variable ==, {A3} Variable !=, {A4} Variable <, {A5} Variable >
  - src: `research/wiki_articles/event_instruction_a0_d5_variable_readers.md`
- **All six comparison blocks fall into a single shared store at `0x8014a00c` that writes the boolean result back into `var[0]` (overwriting operand A) — no global `DAT_` flag, no meaningful return value — so comparing a named variable requires staging the operands into `var0`/`var1` first via the `0xB0`-family writers (e.g. `Zero(0); AddVar(0,87)` to copy var 87 → `var0`, `Zero(1); Add(1,k)` to load a constant → `var1`, then `0xA2`, with the boolean landing in `var0`).** — `[S] 1/3`
  - S: shared writeback `0x8014a00c` (`sw v1, 0x0(v0)` with `v0` = `0x8005771C` = `var[0]`) (`battle_disassembly.txt`)
  - src: `research/wiki_articles/event_instruction_a0_d5_variable_readers.md`
- **The odd/even immediate-vs-Var operand distinction (the `andi …,1` test) exists only in the `0xB0`–`0xBE` writer handler `FUN_8014a018` (at `0x8014a050`) — not in the `0xA0`–`0xA5` comparison handler.** — `[S] 1/3`
  - S: writer `FUN_8014a018`, odd/even test at `0x8014a050` (`battle_disassembly.txt`)
  - src: `research/wiki_articles/event_instruction_a0_d5_variable_readers.md`
- **`FUN_8014a2ec` (var id → address resolver) has exactly 2 callers, both inside the writer `FUN_8014a018` (`0x8014a07c`, `0x8014a0c4`); the sibling bit-resolver `FUN_8014a39c` likewise has 2 callers (`0x8014a088`, `0x8014a0d0`), again both in the writer.** — `[S] 1/3`
  - S: `FUN_8014a2ec` callers `0x8014a07c`/`0x8014a0c4`; `FUN_8014a39c` callers `0x8014a088`/`0x8014a0d0` (`battle_disassembly.txt`)
  - src: `research/wiki_articles/event_instruction_a0_d5_variable_readers.md`

## Notes

(empty — user territory)

## Related

- [[Event Jump Opcodes]]
- [[Wait Value Opcode]]
- [[Event Opcode Catalog]]
