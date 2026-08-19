# Unit Event Hold Opcode

Event opcodes `{6D}` Set Unit Event Hold / `{6C}` Clear Unit Event Hold set/clear bit 26 (`0x04000000`) in the per-unit node's flags word at `node+0x80` — a "held under event control" flag that suppresses part of the unit's automatic per-frame update (the `FUN_800822bc` sub-step inside the main-loop per-unit update `FUN_80082468`). Static + dynamic analysis complete (2026-07-10, scenario 6 "Abducting the Princess"): scenario 6 sets the hold on units 5/12 before `Reset Palette` and on unit 52 before `Warp Unit`, and never emits `{6C}` (the flag is torn down at event end). Godot intentionally no-ops these: cinematic units are already VM-driven (`is_cinematic_unit`), so there is no automatic update to suppress.

## Points

- **Event opcodes `{6D}` / `{6C}` set / clear bit 26 (`0x04000000`) in the per-unit node's flags word at `node+0x80` (node = `0x800B7308 + index*0x440`, via the shared resolver `FUN_8007a6e4`): both dispatch into the same handler `FUN_8014968c` (`{6D}` flag=1 @ `0x8014513c`, `{6C}` flag=0 @ `0x80145120`), which fans out over up to 21 resolved unit slots into the setter `FUN_8008ccac` (`*(node+0x80) |= 0x04000000`) / clearer `FUN_8008cce8` (`&= ~0x04000000`); in scenario 6 the bit is set on units 5/12 before `Reset Palette` and on unit 52 before `Warp Unit`, and scenario 6 never emits `{6C}` — the flag is torn down at event end.** — `[S·D] 2/3`
  - S: dispatch cases `0x8014513c` / `0x80145120`, handler `FUN_8014968c` @ `0x8014968C`, setter `FUN_8008ccac` @ `0x8008CCAC`, clearer `FUN_8008cce8` @ `0x8008CCE8`, resolver `FUN_8007a6e4` @ `0x8007A6E4`, roster base `0x800B7308` stride `0x440` (`project-assets/fft-rom/battle_disassembly.txt`)
  - D: BP on the setter store `0x8008ccd4`, scenario-6 pre-events live run — `SET node=800B8848 newflags=04000B08` (unit 5), `SET node=800BA608 newflags=24000B08` (unit 12); handler-head BP logged unit=5/12/52 at pc 397/400/1488 (2026-07-10)
  - R: none — the bit-26 hold semantics are not implemented in godot-learning; `0x6D`/`0x6C` are named `SET_UNIT_EVENT_HOLD`/`CLEAR_UNIT_EVENT_HOLD` and bound to `_skip` in `godot-learning/src/scenarios/ScenarioVM.gd` (cinematic units are already VM-driven via `Unit.is_cinematic_unit`; probed `godot-learning/src/`, `godot-learning/tests/`)
  - src: `research/working_documents/SCENARIO6_UNKNOWN_OPCODES_6D_71_7C_82_INVESTIGATION.md`
- **The bit-26 flag is consumed by the per-unit update `FUN_80082468`, a routine the main battle loop calls every frame (callers `0x80069534`, `0x8006DDAC`, `0x80083D14`, `0x80083EC4`): the test-and-skip at `0x8008248c` skips the `FUN_800822bc` sub-step while the bit is set, so a held unit's automatic per-frame update is partially suppressed while its palette is reset or it is warped ("held under event control"); what `FUN_800822bc` itself does (reads `unit+0x13e`) is not yet fully characterised.** — `[S] 1/3`
  - S: consumer `FUN_80082468` @ `0x80082468`, test-and-skip `0x8008248c`, caller list (static only — the doc's §6 D7; `project-assets/fft-rom/battle_disassembly.txt`)
  - R: none — no per-frame combat update loop / bit-26 gate in godot-learning (probed `godot-learning/src/`, `godot-learning/tests/` for `0x04000000` / suppress-update — only the `0x6D`/`0x6C` `_skip` bindings exist)
  - src: `research/working_documents/SCENARIO6_UNKNOWN_OPCODES_6D_71_7C_82_INVESTIGATION.md`

## Notes

(empty — user territory)

## Related

- [[Event Opcode Catalog]]
- [[Reset Palette Opcode]]
- [[Unit Sprite Object Struct]]
- [[Scenario 6 Carry Composition]]
