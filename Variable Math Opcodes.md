# Variable Math Opcodes

Event instructions `0xB0`–`0xBE` are a single family of variable-arithmetic opcodes (ADD, SUB, MUL, DIV, MOD, AND, OR, plus their `*Var` operand variants and ZERO), all dispatched to one handler, `FUN_8014a018`, which reads and writes the FFT event variable file. The nailed-down members are `0xBE ZERO(var)` (3 bytes) and `0xB0 ADD(var, value)` (5 bytes): the event compiler has no SET opcode, so `ZERO(v); ADD(v, k)` is the "set variable to constant k" idiom. Both were statically decoded byte-for-byte against `battle_disassembly.txt` and dynamically validated in real scenario context (2026-06-30) — the idiom was captured firing live in the Orbonne prayer scene, and the arithmetic was proven via register hijack at the handler entry. The opcode's low bit selects immediate vs `*Var` operand, and inside the handler `s2` holds the opcode (not the var id). ScenarioVM (`godot-learning`) still lacks `_op_zero`/`_op_add` and a variable store, so it halts on the unhandled `Zero` at scenario_1 pc 319.

## Points

- **`0xB0`–`0xBE` is a single family of variable-arithmetic opcodes dispatched to one handler, `FUN_8014a018`: the event reader fetches the opcode plus up to 4 operand bytes (`s4`–`s7`, `0x80143d1c`–`0x80143d34`), and the dispatcher routes the whole range via `sltiu v0, (op-0xb0), 0xf` at `0x80143d70` and `jal` at `0x80143da0`, passing `a0` = opcode, `a1` = var id, `a2` = value, `a3` = 0.** — `[S·D] 2/3`
  - S: opcode fetch `0x80143d1c`–`0x80143d34`, route `0x80143d70`, `jal` `0x80143da0`, handler `FUN_8014a018` (`project-assets/fft-rom/battle_disassembly.txt`)
  - S: FFTPatcher `EventCommands.xml` master catalog — all 15 writers `{B0}`–`{BE}` take Variable:2 + Value:2 (odd `*Var` forms take Variable:2 twice; `{BE}` Zero takes Variable:2 only)
  - D: register-hijack probe at the handler entry `0x8014a018` and the var87 write-BP capture both land in this handler (2026-06-30)
  - src: `research/wiki_articles/event_instruction_b0_be_variable_math.md`
- **`0xBE` is ZERO(var), 3 bytes (`BE vv vv`), `var = 0`: the handler special-cases it before the odd/even operand test (`beq s2, 0xBE` at `0x8014a048`) and skips operand resolution entirely, clearing the result with `_clear s0` at `0x8014a11c`.** — `[S·D] 2/3`
  - S: special-case branch `0x8014a048`, `_clear s0` `0x8014a11c` (`battle_disassembly.txt`)
  - D: var87 write-BP capture (2026-06-30): `pc=0x8014a2b4 s2=0xBE s0=0` → ZERO(87) in the real event stream; register hijack (documented 2026-06-30): ZERO writes 0
  - src: `research/wiki_articles/event_instruction_b0_be_variable_math.md`
- **`0xB0` is ADD(var, value), 5 bytes (`B0 vv vv nn nn`), `var = var + value`: computed by `_addu s0, s0, s1` at `0x8014a12c` and persisted through the `sw` writeback.** — `[S·D] 2/3`
  - S: `_addu` `0x8014a12c`, word-path writeback `0x8014a2b4` (`battle_disassembly.txt`)
  - D: register hijack (documented 2026-06-30): `1 + 7 → 0x8`, then `8 + 0x10 → 0x18`; var87 write-BP capture (2026-06-30): `pc=0x8014a2b4 s2=0xB0 s0=634/635` → ADD(87, k) in the real event stream
  - src: `research/wiki_articles/event_instruction_b0_be_variable_math.md`
- **`ZERO(v); ADD(v, k)` is the event compiler's "set variable to constant k" idiom (there is no single SET opcode): in `scenario_1` it appears at pc 319/320 as `ZERO(87); ADD(87, 0)` — net `var[87] = 0`, right after `Erase Unit 52` + `Wait 15` — and the same pair was captured firing live in the Orbonne prayer scene driving a running counter (climbs 634→658, resets, then 0,1,2,…,124).** — `[D] 1/3`
  - D: `research/working_documents/scenario_1_captures/last_run/var87_zero_add_live.log` — 300 var87 writes, alternating ZERO→ADD at the word-path writeback, Orbonne prayer scene advanced from the female-knight-held-by-Simon savestate (2026-06-30)
  - src: `research/wiki_articles/event_instruction_b0_be_variable_math.md`
- **Inside the handler, the opcode's low bit selects the operand form (`andi v0, s2, 1` at `0x8014a050`): an even opcode takes an immediate value, an odd opcode takes a variable id that is resolved to its current value first — e.g. `0xB0 ADD` adds an immediate while `0xB1 AddVar` adds another variable's value.** — `[S·D] 2/3`
  - S: `0x8014a050` (`battle_disassembly.txt`)
  - D: register hijack (documented 2026-06-30): `0xB1 AddVar` fires through the same op core + writeback
  - src: `research/wiki_articles/event_instruction_b0_be_variable_math.md`
- **ZERO is 3 bytes downstream-clean even though the dispatcher reads a value operand for all `0xB0`–`0xBE`: ZERO's PC advances only 3 bytes, so the trailing 2 bytes are re-parsed as the next opcode, and the live game never desyncs — confirming the 3-byte encoding.** — `[S·D] 2/3`
  - S: opcode fetch reads up to `chunk[+4]` (`0x80143d1c`–`0x80143d34`); ZERO special case `0x8014a048` (`battle_disassembly.txt`)
  - D: live game never desyncs across the 300 captured ZERO→ADD pairs (2026-06-30)
  - src: `research/wiki_articles/event_instruction_b0_be_variable_math.md`
- **The op core covers the rest of the family with the result accumulated in `s0`: SUB `0x8014a13c`, MUL `0x8014a148`, DIV `0x8014a170` (divide-by-zero guard → `event_fiber_mark_complete`), MOD `0x8014a190`, AND `0x8014a1a8`, OR `0x8014a1b4`; writeback is `_sw s0, 0x0(s5)` at `0x8014a2b4` for word vars and `_sw v0, 0x0(s5)` at `0x8014a2a8` for the bit/nibble paths.** — `[S·D] 2/3`
  - S: op core `0x8014a11c`–`0x8014a1b4`, writeback `0x8014a2b4` / `0x8014a2a8` (`battle_disassembly.txt`)
  - D: all 300 captured var87 writes land at the word-path writeback `pc=0x8014a2b4` (2026-06-30)
  - src: `research/wiki_articles/event_instruction_b0_be_variable_math.md`
- **Register naming inside `FUN_8014a018`: `s2` holds the OPCODE (not the var id, despite the dispatcher setting `s2 = chunk[+1]`), the resolved variable address is in `s5`, and the value written is in `s0` — proven by `beq s2, 0xBE` (`0x8014a048`) and `andi v0, s2, 1` (`0x8014a050`), which only make sense if `s2` is the opcode.** — `[S·D] 2/3`
  - S: `0x8014a048`, `0x8014a050` (`battle_disassembly.txt`)
  - D: live capture reads `s2 = 0xBE / 0xB0` (the opcodes), not `0x57`, at writeback time (2026-06-30)
  - src: `research/wiki_articles/event_instruction_b0_be_variable_math.md`
- **The idle input/timer poll loop invokes the variable-arithmetic handler `FUN_8014a018` ~1200×/sec on var 0 — the event VM continuously polls a variable through this handler.** — `[D] 1/3`
  - D: observed while arming an execution breakpoint at the handler entry `0x8014a018` (2026-06-30)
  - src: `research/wiki_articles/event_instruction_b0_be_variable_math.md`

## Notes

(empty — user territory)

## Related

- [[Event Variable File]]
- [[Wait Value Opcode]]
- [[Variable Comparison Opcodes]]
- [[Event Opcode Catalog]]
