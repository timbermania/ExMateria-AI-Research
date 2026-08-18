# Wait Value Opcode

Event instruction `0x7E` Wait Value (`7E vv vv nn nn`, 5 bytes) is the event VM's write-then-poll variable barrier: it blocks the fiber until the named variable reaches `>= value` (signed), re-polling by cooperatively yielding the fiber each frame — it neither writes the variable nor busy-waits, so an overshooting counter still releases. The static decode (2026-06-30) pins the handler as `FUN_8014a3f8` (Ghidra mislabels it `evt0x7f_evtchr_palette_handler` — an attribution now CONTESTED by the master catalog, which assigns `0x8014a3f8` to `0x7F` EVTCHR Palette) and establishes that Wait Value is the *real* consumer of var 87 in the Orbonne prayer scene — a free-running frame-timing counter reset to 0 by `Zero(87); Add(87,0)` and polled at frames 28/30 to schedule the two altar rotations. The compare/jump family (`0xA0`–`0xA5`, `0xD0`–`0xD5`) is fully present in the engine but is never executed in the scenario chunks we own. The live write-BP on the resolved address (2026-06-30) nailed the per-frame producer: every var 87 write in the prayer scene comes from the event VM's own `Zero(87); Add(87, k)` pairs at the word-path writeback `0x8014a2b4`.

## Points

- **Event instruction `0x7E` Wait Value is a 5-byte write-then-poll barrier — `7E vv vv nn nn` = `Wait Value(variable, value)` (catalog `event_opcodes.json`) — handled by `FUN_8014a3f8`, dispatched at `0x80145af4` (which actually tests `0x7E`; the Ghidra label `evt0x7f_evtchr_palette_handler` is a delay-slot mislabel, the `ori 0x7f` stages the *next* comparison) and again from the secondary table at `0x8013e9ec`.** — `[S] 1/3 CONTESTED`
  - S: handler `FUN_8014a3f8`, dispatch `0x80145af4`, secondary table `0x8013e9ec` (`project-assets/fft-rom/battle_disassembly.txt`); encoding per `event_opcodes.json` catalog
  - src: `research/wiki_articles/event_instruction_a0_d5_variable_readers.md`
- **The master catalog instead assigns `0x8014a3f8` to `0x7F` EVTCHR Palette (Unit:2, Block:1, Palette:1): `0x8014a3f8` (`evt0x7f_evtchr_palette_handler`) is a timing-wait on `FUN_8013b590(Block) >= Palette`, i.e. it reads the function as the {7F} handler rather than the {7E} Wait Value handler.** — `[S] 1/3 CONTESTED`
  - S: `0x8014a3f8` (`evt0x7f_evtchr_palette_handler`), `FUN_8013b590` (master catalog `EventCommands.xml` row {7F})
  - S (corroborating, 2026-08-17): `0x8014a3f8` re-pinned as the `{7F}` dispatch with corpus-wide 135 `{7F}` instances across 52 chunks, each emitted immediately before its unit's cinematic `{11}` (`research/scenario_1_captures/evtchr_load_save_decode.md`, via `research/working_documents/EVTCHR_CHARACTER_ATTRIBUTION.md`)
  - src: `research/wiki_articles/event_instructions.md`
- **Wait Value blocks the fiber until `var[id] >= value` (signed `>=`, not `==`): it reads the variable via `FUN_8013b590`, compares with `slt`, and while below the threshold cooperatively yields the fiber (`event_fiber_yield`, `0x8014ca80`) and re-polls next frame — it does not write the variable and does not busy-wait, so a counter that *overshoots* the threshold still releases (matters for a faithful port).** — `[S] 1/3`
  - S: poll loop `0x8014a428`–`0x8014a444` (var read via `jal FUN_8013b590` at `0x8014a428`, `slt` at `0x8014a430`, `beq` exit at `0x8014a434`, `event_fiber_yield` `0x8014ca80`, re-poll `j` at `0x8014a444`) (`battle_disassembly.txt`)
  - src: `research/wiki_articles/event_instruction_a0_d5_variable_readers.md`
- **In the Orbonne prayer scene, var 87 is a free-running frame-timing counter: `Zero(87)` at scenario_1 chunk offset 1863 and `Add(87,0)` at 1866 reset it to 0 as the sync point, then the parallel block spawned at 1871 (`2A` Block Start) polls `Wait Value(87,28)` at 1872 and `Wait Value(87,30)` at 1884 to fire the two altar rotations (Rotate Unit at 1877/1889) at frames 28/30 — a timing-sync mechanism, not a conditional branch; `scenario_0001_setup` uses the identical idiom.** — `[D] 1/3`
  - D: `scenario_1_chunk.json` + `scenario_0001_setup_chunk.json` byte-grounded against the real scenario chunks (2026-06-30)
  - src: `research/wiki_articles/event_instruction_a0_d5_variable_readers.md`
- **Neither `scenario_1` nor `scenario_0001_setup` executes a single `0xA0`–`0xA5` comparison or `0xD0`–`0xD5` jump — every such byte that can be grepped out of those chunks lives in the mis-decoded dialog-text region past `_text_offset` (2297) — so the only reader of var 87 in the scenarios we own is `0x7E` Wait Value, never the comparison/jump family.** — `[D] 1/3`
  - D: `scenario_1_chunk.json` + `scenario_0001_setup_chunk.json` byte-grounded against the real scenario chunks (2026-06-30)
  - src: `research/wiki_articles/event_instruction_a0_d5_variable_readers.md`
- **`FUN_8013b590` is the resolver-mediated event-variable read (base = `*0x80165f9c`): only var `0x22` is special-cased inside it (to a camera angle), var `0x57` (87) takes the generic read path, and the read is read-only (it mirrors var 87 into scratch var 0).** — `[S] 1/3`
  - S: `FUN_8013b590` (called at `0x8014a428`), base pointer `*0x80165f9c` (`battle_disassembly.txt`)
  - src: `research/wiki_articles/event_instruction_a0_d5_variable_readers.md`
- **The per-frame producer of var 87 is the prayer scene's own `Zero(87); Add(87, k)` idiom: a write breakpoint on the resolved address `0x80057878` (Orbonne prayer scene, female-knight-held-by-Simon savestate, advanced by real CIRCLE taps) captured 300 writes — every one at the event VM's word-path writeback `pc=0x8014a2b4`, in strict `ZERO → ADD` pairs ~820 cycles apart (counter climbs 634→658, resets, then 0,1,2,…,124) — so no store outside the event VM drives it.** — `[S·D] 2/3`
  - S: `0x8014a2b4` (`_sw s0, 0x0(s5)` word-path writeback inside `FUN_8014a018`) (`battle_disassembly.txt`)
  - D: write-BP capture on `0x80057878`, `research/working_documents/scenario_1_captures/last_run/var87_zero_add_live.log` — 300 writes, strict ZERO→ADD pairs (2026-06-30)
  - src: `research/wiki_articles/event_instruction_b0_be_variable_math.md`

## Notes

(empty — user territory)

## Related

- [[Variable Comparison Opcodes]]
- [[Event Jump Opcodes]]
- [[Variable Math Opcodes]]
- [[Event Variable File]]
- [[Event Opcode Catalog]]
