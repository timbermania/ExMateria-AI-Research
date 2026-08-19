# Wait Add Unit Opcode

Event opcode `{48}` Wait Add Unit is a spin barrier in the large event executor: it cooperatively yields (`event_fiber_yield`) and re-polls the "units currently being added" counter `0x8016604e` until it returns to 0. That counter is incremented by the Add-Unit opcodes as they enqueue the async CDROM-DMA sprite load (`{47}` Add Ghost Unit enqueues to `evtchr_unit_add_queue` @ `0x80049c1c`, uploaded by `evtchr_unit_queue_drain` @ `0x80088e04`) and zeroed at `0x8013f064` once the queue drains — on PSX the upload spans frames, so `{48}` is a real barrier there. In the Godot reimplementation the spawn is synchronous, so the barrier is pre-satisfied and `{48}` is bound to a clean skip, exactly like its sibling `{4B}` Wait Add Unit End. Validated in the scenario 6 ride-off (2026-07-05).

## Points

- **`{48}` Wait Add Unit is an inline spin barrier @ `0x801458bc` (guard `LAB_801458b4`, opcode preloaded by `8014584c _ori v0,zero,0x48`; zero operands per `EventCommands.xml`): `jal event_fiber_yield`, `lhu` the counter `0x8016604e`, exit when 0, else loop back; the counter is incremented by the Add-Unit opcodes as they enqueue an async CDROM-DMA sprite load (writes @ `0x801440b0`, `0x8014420c`, `0x801437b8`) and zeroed at `0x8013f064` once the queue drains; the `{47}` path enqueues to `evtchr_unit_add_queue` @ `0x80049c1c` and uploads via `evtchr_unit_queue_drain` @ `0x80088e04`, and the sibling `{4B}` Wait Add Unit End is the adjacent block @ `0x801458e0`.** — `[S·R] 2/3`
  - S: barrier body `0x801458bc`, counter `0x8016604e` (writer sites `0x801440b0`/`0x8014420c`/`0x801437b8`, zero @ `0x8013f064`), queue `0x80049c1c`, drain `0x80088e04`, sibling `0x801458e0` (`battle_disassembly.txt`, BATTLE.BIN base `0x80067000`; cross-ref `ADD_GHOST_UNIT_OPCODE_47.md`)
  - R: `godot-learning/src/scenarios/ScenarioVM.gd` `_bind(EventInstruction.WAIT_ADD_UNIT, Callable(self, "_op_skip"))` (adjacent to the `{4B}` skip) — validated by `ScenarioEventInstructionCoverageTest` (`0x48` `verified:true` in `assets/scenarios/event_instructions.json`, verified count 14→17)
  - src: `research/working_documents/FACE_TILE_UNIT_SHADOW_WAIT_ADD_UNIT.md`
- **In Godot `{48}`'s skip is the faithful model because scenario spawns are synchronous: `add_child()` runs `Unit._ready()` in the same call stack and returns the instantiated node before the handler returns (and `{45}` Add Unit is likewise a no-op — units are pre-spawned from ENTD), so by the time `{48}` executes the pending-add counter's Godot analogue is already 0 and the barrier is pre-satisfied.** — `[R] 1/3`
  - R: `godot-learning/src/scenarios/ScenarioVM.gd` `_op_skip` + `ScenarioPlayerScene._spawn_ghost_actor` (synchronous spawn) — validated by `tools/probe_scenario6_freeze.gd` GREEN (pc 403/405/417 pass, no halt; 2026-07-05)
  - R: this bullet's "`{45}` Add Unit is likewise a no-op" clause is now stale — `{45}` commits the unit graphic and applies the inverted Draw-byte visibility (`godot-learning/src/scenarios/ScenarioApply.gd` `add_unit`, validated by `godot-learning/tests/ScenarioUnitVisibilityTest.gd`; 2026-08-18, see [[Unit Visibility Flag]])
  - src: `research/working_documents/FACE_TILE_UNIT_SHADOW_WAIT_ADD_UNIT.md`

## Notes

(empty — user territory)

## Related

- [[Add Ghost Unit Opcode]]
- [[Event Opcode Catalog]]
- [[Unit Visibility Flag]]
- [[Face Tile Opcode]]
- [[Unit Shadow Opcode]]
