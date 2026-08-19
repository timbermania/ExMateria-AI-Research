# Unit Shadow Opcode

Event opcode `{4E}` Unit Shadow is a render-enable toggle for a unit's ground drop shadow: the inline handler in the large event executor validates the unit id, then writes the shadow show-flag `unit+0x298` — 1 when `Disable==0` (shadow renders), 0 when `Disable!=0` (shadow suppressed) — the same byte the shadow renderer `FUN_8007d5d0` gates on and that anim opcodes `0xE0` HideShadow / `0xE1` ShowShadow write, making `{4E}` the third writer of `+0x298`. Scenario scripts kill the shadow when a unit leaves the ground so no ground-locked blob floats under a lifted/carried/ridden unit. Bound and wired in godot-learning onto the existing per-unit `UnitShadow` mesh, validated in the scenario 6 ride-off (2026-07-05).

## Points

- **`{4E}` Unit Shadow is handled inline (no standalone symbol) at `0x80145a94` (guard `LAB_80145a8c`, opcode preloaded by `80145a78 _ori v0,zero,0x4e`) with s3 = unit id (u16) and s6 = Disable byte: the id is validated via `FUN_80133158` (sentinel `0x7d0` → handler bails), then `Disable==0` (enable) → `FUN_8008c268` writes `sb 1, 0x298(unit)` @ `0x8008c28c` (shadow shows), `Disable!=0` (disable) → `FUN_8008c2a4` writes `sb 0, 0x298(unit)` @ `0x8008c2c4` (shadow suppressed).** — `[S·D·R] 3/3`
  - S: inline body `0x80145a94`, validator `FUN_80133158`, setters `FUN_8008c268` / `FUN_8008c2a4` with sb sites `0x8008c28c` / `0x8008c2c4` (`battle_disassembly.txt`, BATTLE.BIN base `0x80067000`)
  - D: scenario 6 ride-off probe log — `Unit Shadow 0x0C → OFF` (unit 12 @ pc 225), `0x05 → OFF` (unit 5 @ pc 226), `0x8B → OFF` (unit 139 @ pc 384), matching the ROM's `sb 0` on each, no halt (2026-07-05)
  - R: `godot-learning/src/scenarios/ScenarioVM.gd` `_op_unit_shadow` → `ScenarioApply.unit_shadow` (resolve id, skip if unspawned) → `ScenarioWorld.set_unit_shadow` — validated by `ScenarioApplyTest._test_unit_shadow_toggles` (Disable→enabled inversion, missing-unit skip)
  - src: `research/working_documents/FACE_TILE_UNIT_SHADOW_WAIT_ADD_UNIT.md`
- **The unit-struct shadow show-flag is `unit+0x298` — non-zero ⇒ the drop-shadow dispatch calls renderer `FUN_8007d5d0` — and `{4E}` is the third writer of `+0x298` besides anim opcodes `0xE0` HideShadow / `0xE1` ShowShadow; scenario scripts use it to kill the shadow when a unit leaves the ground (constant-size, ground-locked blob) so no blob floats under a lifted/carried/ridden unit (scenario 6: Ovelia 12 and Delita 5 lifted at pc 225/226 with carry-pose anim 526 + +Y sprite move; chocobo 139 riding off at pc 384).** — `[S·D·R] 3/3`
  - S: sb write sites `0x8008c28c` / `0x8008c2c4`, renderer gate `FUN_8007d5d0` (`battle_disassembly.txt`; flag semantics per `UNIT_SHADOW_RENDERING.md`)
  - D: scenario 6 ride-off capture — the shadow-off set (units 12/5/139) is exactly the lifted/carried/riding units (2026-07-05)
  - R: `godot-learning/src/units/Unit.gd` `$ShadowMesh` + `UnitShadow.set_enabled` (per-unit shadow already shipped; `{4E}` just drives it) — validated by `ScenarioApplyTest._test_unit_shadow_toggles`
  - src: `research/working_documents/FACE_TILE_UNIT_SHADOW_WAIT_ADD_UNIT.md`

## Notes

(empty — user territory)

## Related

- [[Event Opcode Catalog]]
- [[Face Tile Opcode]]
- [[Wait Add Unit Opcode]]
- [[Unit Shadow Rendering]]
