# AddUnit Block Opcode

`{82}` is not an independently-dispatched opcode — it is a sub-token of the `{49}` AddUnitStart … `{4A}` AddUnitEnd block. `{49}` spawns a dedicated block-processor task (`FUN_8013edd8`) and skips the main PC past `{4A}`; the processor walks the block's `{45}` AddUnit and `{82}` tokens, and `{82}` performs control-point / midpoint geometry setup for the just-added units (`FUN_8013e81c`). Static + dynamic analysis complete (2026-07-10, scenario 6 idx 436): a main-ladder slot for a standalone `{82}` also exists (`0x80145b4c`), but it never fires in scenario 6 — community reports of `{82}` "simulating a brief wait" / crashing come from misusing it outside a block, where it runs `FUN_8013e81c` with the interpreter's register state. Godot treats `{49}`/`{4A}` as pure brackets and skips `{82}` (add-unit + `ScenarioPathMotion` cover the path/geometry init).

## Points

- **`{82}` is not an independently-dispatched opcode — it is a sub-token of the `{49}` AddUnitStart … `{4A}` AddUnitEnd block: `{49}` (dispatch `0x80145910`) allocates a task slot via `FUN_80149bec(0x10)`, spawns the block processor `FUN_8013edd8` handed the block pointer, and advances the main PC past `{4A}` via `FUN_80149ebc`; the processor's inner dispatch (`0x8013ee30`) calls `FUN_8013e81c` on token `0x82` — Catmull-Rom-style control-point / midpoint geometry setup (special-casing first/last points) that writes tangents/positions into a per-actor path struct at `+0x94`/`+0x9c` (indexed by `+0x84`) — and `FUN_80180c90` on `{45}` AddUnit; `FUN_8013e81c` is also reachable from the main-ladder slot at `0x80145b4c`, where a standalone `{82}` outside a block runs with the interpreter's register state (the source of the community "brief wait / crash" reports).** — `[S·D] 2/3`
  - S: `{49}` dispatch `0x80145910`, block processor `FUN_8013edd8` @ `0x8013EDD8`, inner dispatch `0x8013ee30` (`jal` at `0x8013ee40`), `{82}` action `FUN_8013e81c` @ `0x8013E81C`, `{45}` action `FUN_80180c90` @ `0x80180C90`, ladder slot `0x80145b4c` (`project-assets/fft-rom/battle_disassembly.txt`)
  - D: scenario-6 live run — the dispatch BP at `0x80145b4c` never fired although the interpreter demonstrably ran through offset 2562; a filtered trace of the executed opcode stream (`BP 0x80143d34`, s8 ∈ [2540,2672]) shows the dispatcher jumping straight `{3D}` → `{49}` → `{F1}` (0x82 and the inner `{45}`/`{4A}` never appear); the `FUN_8013e81c` entry BP fired with `ra=0x8013EE48` (the `jal` at `0x8013ee40` inside the block processor, not the ladder site `0x80145b50`); live chunk bytes at 2562/2563 = `0x82`/`0x4A` (2026-07-10)
  - R: none — AddUnit-block sub-token semantics not implemented in godot-learning; `0x49`/`0x4A` bound to `_op_skip` ("pure brackets in the ROM") and `0x82` (`ADD_UNIT_PATH_SETUP`) bound to `_skip` ("covered by add-unit + ScenarioPathMotion") in `godot-learning/src/scenarios/ScenarioVM.gd` (probed `godot-learning/src/`, `godot-learning/tests/`)
  - src: `research/working_documents/SCENARIO6_UNKNOWN_OPCODES_6D_71_7C_82_INVESTIGATION.md`

## Notes

(empty — user territory)

## Related

- [[Event Opcode Catalog]]
- [[Block Execution]]
- [[Wait Add Unit Opcode]]
- [[Add Ghost Unit Opcode]]
- [[Unit Visibility Flag]]
