# Face Tile Opcode

Event opcode `{69}` Face Tile rotates the affected unit(s) to look at a map tile (operand X = column, Y = row), reverse-engineered byte-exact from the BATTLE.BIN handler `FUN_8014978c`. It reuses the `{53}` Face Unit / `{2D}` Rotate Unit machinery wholesale — same 12-bit look-at fold, same `pending_rotate_command_array` @ `0x8016d9d8`, speed table @ `0x80169750`, and consumer @ `0x8013f20c` — differing only in the target-angle source (a literal tile instead of a resolved unit). Implemented in godot-learning and validated in the scenario 6 (Ovelia abduction) ride-off on 2026-07-05, where a missing binding for `{69}`/`{4E}`/`{48}` globally halted the ScenarioVM and froze the ride-off trio; the PSX row-flip mapping was triangulated three independent ways, including the chocobo-corner natural experiment. A BP-free PSX poll of unit 2's facing nibble right after pc 377 remains the one open ground-truth follow-up (expected `0x6` = 12-bit `0x600`).

## Points

- **`{69}` Face Tile is handled by `FUN_8014978c` (0x8014978c–0x80149890), dispatched inline from the `event_scenario_interpreter` bne-chain @ `0x80143bd8` (guard `LAB_80144e24`, opcode preloaded by `80144e0c _ori v0,zero,0x69`); it is a distinct function from Face Unit's `evt0x53_face_unit_handler` @ `0x80148084` but reuses the {53}/{2D} rotate machinery — it writes the same 7-byte command into `pending_rotate_command_array` @ `0x8016d9d8` (+0 facing nibble, +1 Direction, +2 `rotate_speed_lookup[Speed]` from table @ `0x80169750`, +3 = 0) for the same consumer `rotate_command_tick_consumer` @ `0x8013f20c`, with only the target angle source differing (literal tile vs. resolved unit).** — `[S·D·R] 3/3`
  - S: `FUN_8014978c`, dispatch `0x80143bd8`, `evt0x53_face_unit_handler` `0x80148084`, `pending_rotate_command_array` `0x8016d9d8`, `rotate_speed_lookup` `0x80169750` (read @ `0x80149888`), consumer `0x8013f20c` (`battle_disassembly.txt`, BATTLE.BIN base `0x80067000`; shared-queue cross-ref `face_unit_decode.md` §6.1)
  - D: scenario 6 ride-off probe `godot-learning/tools/probe_scenario6_freeze.gd` GREEN — zero SKIP-UNHANDLED lines, onlookers 2/23/52 turn toward the trio, ride-off completes to pc≈453 (2026-07-05)
  - R: `godot-learning/src/scenarios/ScenarioVM.gd` `_op_face_tile` → `ScenarioApply.face_tile` → `ScenarioWorld.face_tile_look_at_12bit` + `arm_rotate` (the {2D} `scenario_rotate` stepper) — validated by `ScenarioApplyTest._test_face_tile_rotates_affected_to_tile`
  - src: `research/working_documents/FACE_TILE_UNIT_SHADOW_WAIT_ADD_UNIT.md`
- **`{69}` operand layout is `Units, Multi, X, Y, Unused, Direction, Speed, Delay` at payload +0x0…+0x7 (X=+0x2, Y=+0x3, Direction=+0x5, Speed=+0x6, Delay=+0x7), and the tile → PSX-internal world conversion is `X*28 + 14` / `Y*28 + 14` per axis (28 units per tile, +14 = tile centre); operand X = column (lateral), Y = row (depth) in raw PSX rows — Y=13 hits `size_z−1`, the far row, not flipped — with the actor position resolved via `FUN_80133088` (whose PSX axes are the transpose of chunk X/Y: PSX-x = depth/row, PSX-y = lateral/column) and the 12-bit CW angle from the SCUS arctangent `SUB_8001d8e8`.** — `[S·D·R] 3/3`
  - S: `FUN_8014978c` operand decode + `FUN_80133088` actor resolve + `SUB_8001d8e8` (`battle_disassembly.txt`; operand order per `EventCommands.xml`; axis naming per `face_unit_decode.md` §5)
  - D: scenario 6 probe (2026-07-05): pc 377–379 onlookers 2/23/52 with tile (0,13) face 0x600 SW / 0x800 W / 0x400 S — toward the ride-off corner; pc 380 Multi=1 team-set variant warn-skips (static-only in the ROM, same limitation as {53})
  - R: `godot-learning/src/scenarios/ScenarioVM.gd` `_face_tile_look_at_12bit` + `ScenarioApplyTest._test_face_tile_guards` (Multi≠0 / missing / invalid guards mirroring `face_unit`)
  - src: `research/working_documents/FACE_TILE_UNIT_SHADOW_WAIT_ADD_UNIT.md`
- **The Godot port must take the Face Tile delta in the PSX frame: un-flip the actor's row first (`Δdepth = (size_z−1 − floor(z)) − tile_y`, `Δlateral = floor(x) − tile_x`) and feed the shared look-at fold — the naive `tile_y − floor(z)` no-flip mapping is wrong (faced onlookers EAST, away from the trio, at empty tile (0.5,13.5)), while the flip form reduces exactly to the live-validated `{53}` unit-to-unit look-at and is confirmed by the chocobo-corner natural experiment: flipped tile (0,13) → Godot (0.5,0.5) = exactly unit 139's rest tile (probe: unit 139 at (0.50, 3.04, 0.50) just before ride-off).** — `[S·D·R] 3/3`
  - S: `0x8014978c` ROM math (operand-Y-as-PSX-row subtracted from the actor's PSX-internal row — the only frame-correct mapping); `face_unit_decode.md` §5/§6 (Face Unit live-decoded to 0x400, same fold)
  - D: scenario 6 natural experiment — unit 139 rest-position probe + corrected onlooker facings vs. no-flip target (0.5,13.5) (2026-07-05)
  - R: `godot-learning/src/scenarios/ScenarioVM.gd` `_face_tile_look_at_12bit` → `face_unit_look_at_12bit_psx` (= `PsxNum.look_at_12bit`, uses host-set `map_size_z`) — validated by `ScenarioApplyTest._test_face_tile_rotates_affected_to_tile`
  - src: `research/working_documents/FACE_TILE_UNIT_SHADOW_WAIT_ADD_UNIT.md`

## Notes

(empty — user territory)

## Related

- [[Rotate Unit Interpolation]]
- [[Event Opcode Catalog]]
- [[Unit Shadow Opcode]]
- [[Wait Add Unit Opcode]]
