# Event End Opcode

FFT's two event-script terminators: `{DB}` Event End (soft end — a per-unit graphic/palette-restore pass, after which the script simply ends via the interpreter's default fall-through; the chunk's last opcode from the data view) and `{E3}` Event End 2 (hard end — synchronous quiesce: barrier on the pending screen/transition flag, drain of kind-8 block child tasks, unit-state finalization, then an explicit fiber mark-complete). Both are RE'd byte-exact against live RAM on the `orbonne_darkscreen_dispatch.sstate` anchor (2026-07-02 session 3), and both are ported in the `godot-learning` `ScenarioVM` as clean context terminators — no unhandled-opcode halt warning — replacing the false "stopping at unhandled opcode 'Event End'" halt every scenario tail previously produced.

## Points

- **`{DB}` Event End is a per-unit graphic/palette-restore pass, not a fiber stop: case `0x8014429c` (operand size 0) calls `FUN_80147d98`, which walks the 21 active roster slots, reads each unit's event-id byte `+0x161` and flag `+0x5 & 0x30`, and for slots whose event id matches (a0 = sign-extended low16 of s3) calls `FUN_8008d26c(slot, 2, −0x1F, −0x1F)` — a graphic/palette revert (the `∓0x1F` curve-index convention matches the prayer-darkening palette path) — then the interpreter just loops back to fetch; the fiber actually terminates when the chunk's trailing unhandled byte falls through to the default `event_fiber_mark_complete` @ `0x80145f3c`.** — `[S·D·R] 3/3`
  - S: case `0x8014429c`, `FUN_80147d98` @ `0x80147d98`, `FUN_8008d26c`, default stop `LAB_80145f3c` (`battle_disassembly.txt`)
  - D: live dispatch of `{DB}` captured as the first opcode of the battle-intro chunk after the scenario-4 event closes (opcode stream `… {1C}@822 {DB}@0 {F2}@1 …`, resume of `orbonne_darkscreen_dispatch.sstate`, 2026-07-02)
  - R: `godot-learning/src/scenarios/ScenarioVM.gd` `_op_event_end` bound to `EVENT_END` (0xDB) — ends the current context cleanly (`ctx.alive = false`, no `push_warning`), mirroring the PSX "restore the units' graphics, then let the script end" semantics (`godot-learning/tests/ScenarioFocusTest.gd` — `_test_event_end_terminates_cleanly`)
  - src: `research/working_documents/FOCUS_OPCODE_1F_INVESTIGATION.md`
- **`{E3}` Event End 2 is the hard terminator + battle handoff used by the ~37 battle-intro chunks that hand the event straight to battle setup: case `0x801442bc` (operand size 0) takes the `FUN_801460a4` barrier (spin `event_fiber_yield` while `*0x8016606c != 0` — the pending screen/transition flag), drains kind-8 (block) child tasks via `FUN_80149cbc(0x8)`, runs the unit-state finalization loop, then fires an explicit `event_fiber_mark_complete` @ `0x80144484` — everything is quiesced synchronously before the hard stop.** — `[S·D·R] 3/3`
  - S: case `0x801442bc`, barrier `FUN_801460a4` @ `0x801460a4` / flag `0x8016606c`, kind-8 drain `FUN_80149cbc(0x8)`, explicit stop `0x80144484` (`battle_disassembly.txt`)
  - D: all addresses byte-verified against live RAM on the `orbonne_darkscreen_dispatch.sstate` anchor (2026-07-02 session 3)
  - R: `godot-learning/src/scenarios/ScenarioVM.gd` `_op_event_end` bound to `EVENT_END_2` (0xE3) — same clean context-terminator path as `{DB}` (`godot-learning/tests/ScenarioFocusTest.gd` — `_test_event_end_terminates_cleanly`)
  - src: `research/working_documents/FOCUS_OPCODE_1F_INVESTIGATION.md`

## Notes

(empty — user territory)

## Related

- [[Event Opcode Catalog]]
- [[Scenario Camera Opcodes]]
- [[DarkScreen Opcode]]
- [[Block Execution]]
