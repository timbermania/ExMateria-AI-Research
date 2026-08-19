# Reset Palette Opcode

Event instruction `0x97` Reset Palette (Unit:2, 3 bytes total) is the synchronous, yield-free opcode that writes the per-unit palette-modifier byte `unit[+0x13F]`: the default path (`FUN_8008cc80`) zeros it — "restore the base palette" — and a mode==1 path (`FUN_8008cc50`) writes 0x02, an alternate palette state that the chapel never reaches because the "mode byte" (param[+2]) is merely the *next opcode* in the 3-byte stream. The byte is consumed at sprite-render time as an XOR delta against the unit's base palette/TPAGE word `unit[+0x12]` (masked `0xFF9F`, bit 0 set) feeding the FT4 polygon packet. The handler `FUN_8013e65c` is dispatched identically from both the main-interpreter and the block-body-interpreter staircase; every static prediction was confirmed bit-exact against live BATTLE.BIN RAM (2026-06-27, PCSX-Redux port 8082, `orbonne_prayer_mid_dialog` savestate), and the Godot mirror (`ScenarioVM._op_reset_palette`) is verified in the chapel playthrough (5 dispatches at PC 86/87/89/90/91, no VM halt, no parallel-block/cinematic-animation regressions). Open: the PSX-side live capture of the handler firing (dialog-gate input injection unsolved), and whether the base word `unit[+0x12]` is an alias of the cinematic-pipeline `unit[+0x0E]`/`unit[+0x144]` fields.

## Points

- **Opcode `0x97` Reset Palette is 3 bytes total (1 opcode byte + Unit:2 little-endian halfword param): the live in-RAM opcode-size table `DAT_8014d170[0x97]` reads 2 param bytes, matching the chapel chunk's 3-byte stride (PC 86 @ `0x01FF` → PC 87 @ `0x0202`), and the param is read by the pure halfword reader `event_bytecode_reader_c` (`0x80146078`), which does not advance the bytecode pointer.** — `[S·D] 2/3`
  - S: opcode-size table `DAT_8014d170[0x97]`, dispatch staircase `LAB_80144860`, reader `0x80146078` (`battle_disassembly.txt`)
  - D: live RAM reads via PCSX-Redux port 8082, savestate `orbonne_prayer_mid_dialog.sstate` — `DAT_8014d170[0x97]=2` (2026-06-27); chapel chunk offsets per `static_chunk.tsv`
  - src: `research/working_documents/chapel_opcode_trace/RESET_PALETTE_INVESTIGATION.md`
- **Reset Palette's handler is `FUN_8013e65c`, dispatched from both the main-interpreter staircase (`LAB_80144860`) and the block-body-interpreter staircase (`LAB_8013eaec`) — both passing `a0` = first param byte — and it is synchronous (no `FUN_8014ca80` yield), so the interpreter races past it in the same vsync it dispatches.** — `[S·D] 2/3`
  - S: `FUN_8013e65c`, `LAB_80144860`, `LAB_8013eaec` (`battle_disassembly.txt`)
  - D: live RAM reads — handler-entry word `0x8013e65c` = `0x27BDFFE8` (`addiu sp,sp,-0x18`), staircase `0x80144864` = `0x34020097` (`_ori v0,zero,0x97`) (2026-06-27)
  - src: `research/working_documents/chapel_opcode_trace/RESET_PALETTE_INVESTIGATION.md`
- **Reset Palette writes the signed byte `unit[+0x13F]`: the default path `FUN_8008cc80` (write site `0x8008cc98`) zeroes it — "reset to the default palette" — while the mode==1 path `FUN_8008cc50` (write site `0x8008cc6c`) writes 0x02, an alternate palette state; the "mode byte" is param[+2], which in the 3-byte form is the *next opcode* in the stream (0x97/0x2E/0x21 in the chapel), so the chapel always takes the zeroing path.** — `[S·D] 2/3`
  - S: `FUN_8008cc80`/`0x8008cc98`, `FUN_8008cc50`/`0x8008cc6c`, mode branch in handler `FUN_8013e65c` (`battle_disassembly.txt`)
  - D: live RAM reads — mode-1 write site `0x8008cc6c` = `0xA062013F` (`sb v0,0x13f(v1)`), default write site `0x8008cc98` = `0xA040013F` (`sb zero,0x13f(v0)`) (2026-06-27)
  - src: `research/working_documents/chapel_opcode_trace/RESET_PALETTE_INVESTIGATION.md`
- **`unit[+0x13F]` is initialized to 0 at unit-add (`FUN_80087c34`, `sb zero,0x13f(s2)` at `0x80087c34`), so every spawned unit starts in the default palette state and Reset Palette's default write is a pure restore.** — `[S·D] 2/3`
  - S: `FUN_80087c34` (`battle_disassembly.txt`)
  - D: live RAM read — `0x80087c34` = `0xA240013F` (2026-06-27)
  - src: `research/working_documents/chapel_opcode_trace/RESET_PALETTE_INVESTIGATION.md`
- **The sprite renderer consumes `unit[+0x13F]` at `0x8007f358` and `0x80086758`: it loads the byte, XORs it with the base palette/TPAGE word at `unit[+0x12]`, masks with `0xFF9F`, sets bit 0, and feeds the result to `poly_ft4_packet_builder` as a CLUT/TPAGE-side field — i.e. the byte is a per-event XOR delta against the base palette, zero meaning "use the base palette unchanged".** — `[S] 1/3`
  - S: `0x8007f358`, `0x80086758`, `poly_ft4_packet_builder` (`battle_disassembly.txt`)
  - src: `research/working_documents/chapel_opcode_trace/RESET_PALETTE_INVESTIGATION.md`
- **The Orbonne chapel chunk issues Reset Palette at PC 86, 87, 89, 90, 91 (PC 88 is the intervening `Wait T=1`) with unit params {0x02, 0x17, 0x80, 0x81, 0x83}.** — `[S·D] 2/3`
  - S: per-PC opcode table `static_chunk.tsv` (chapel chunk @ `0x8004A6BC`)
  - D: `scenario_1_chunk.json` disassembler dump (2161 instructions) + ScenarioVM playthrough dispatch log (2026-06-27)
  - src: `research/working_documents/chapel_opcode_trace/RESET_PALETTE_INVESTIGATION.md`
- **The Godot mirror implements `_op_reset_palette` in `ScenarioVM`: it resolves the unit via `_resolve_unit_key` (log + no-op on miss, mirroring the PSX 0x7d0-handle no-op) and clears the unit's tint — the full chapel-chunk playthrough (`SCENARIO_PLAY_THROUGH=1`) runs without halting at PC 86, logs "tint cleared" for units 0x02/0x17/0x80/0x81/0x83, and the three parallel Block brackets (block@101/115/128), Ovelia's cinematic anims (0x258→0x25A, 0x25D→0x25E) and Argath's anims (0x25B–0x25F) all fire with no regression.** — `[R] 1/3`
  - R: `godot-learning/src/scenarios/ScenarioVM.gd` `_op_reset_palette`; acceptance test: `godot --path . res://assets/scenes/ScenarioPlayer.tscn` full chapel-chunk run, chapel-trace JSONL covers PC 86–145 with `"handled": true` on every Reset Palette row
  - src: `research/working_documents/chapel_opcode_trace/RESET_PALETTE_INVESTIGATION.md`
- **Opcode `0x71` (wiki "Unknown:2") sits next to `0x97` in the same dispatch staircase: its handler `FUN_8013e6c4` has the same shape as Reset Palette (read halfword, resolve unit, conditional write) but calls `FUN_8008cd24`, which writes a different unit-struct offset; the sibling field `unit[+0x130]` is a 3-state (0/1/2) palette-mode selector consumed by a three-way branch at `0x80086790` and also zeroed at unit-add (`0x80087c30`).** — `[S·D·R] 3/3`
  - S: `FUN_8013e6c4`, `FUN_8008cd24`, reader `0x80086790`, init `0x80087c30` (`battle_disassembly.txt`)
  - D: scn6 PC193 `0x71` {5} on Delita — `unit[+0x130]`/`[+0x13F]` read 0 both before the beat (pc~193, op not yet executed) and after the full punch→throw beat (PCs 193–231), pcsx-redux port 8080, savestate `scenario6_abduct_punch_pickup_start` (2026-07-06) — the op does not change Delita's palette-mode in this scene, so the Godot skip is benign here
  - R: godot-learning registrar auto-skips unnamed opcodes — 0x71 ("Unknown") → clean-skip via `EventInstructionSet.is_unknown` / `ScenarioVM` `_op_skip_unknown` (ADR-0059); the beat runs without halting in `tools/probe_scenario6_punch_pickup.gd`
  - src: `research/working_documents/chapel_opcode_trace/RESET_PALETTE_INVESTIGATION.md`
  - src: `research/working_documents/SCENARIO6_PUNCH_PICKUP_THROW.md`
- **Color Unit (`0x32`) is the chapel's tint-setter (PC 101/106/116/119) and dispatches to `FUN_8009349c` from `LAB_801450d4` — the setter for the tint that Reset Palette clears.** — `[S·D] 2/3`
  - S: `LAB_801450d4`, `FUN_8009349c` (`battle_disassembly.txt`)
  - D: chapel chunk dump `static_chunk.tsv` — Color Unit at PC 101/106/116/119 (2026-06-27)
  - src: `research/working_documents/chapel_opcode_trace/RESET_PALETTE_INVESTIGATION.md`

## Notes

(empty — user territory)

## Related

- [[Event Opcode Catalog]]
- [[Cinematic Palette Pipeline]]
- [[Block Execution]]
