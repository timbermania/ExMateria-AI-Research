# Rotate Unit Interpolation

Event instruction `{2D}` Rotate Unit animates a unit's facing over time rather than snapping: the per-vsync tick consumer `FUN_8013f20c` reads a 7-byte per-handle slot at `0x8016d9d8 + handle*7`, dispatches on the Direction byte (0 = shortest path, 1 = CW, 2 = CCW), and steps the unit's 12-bit facing (wheel S=0x000, E=0x400, W=0x800, N=0xC00 — not uniformly geometric) by one 0x100 direction once `counter >= DAT_80169750[Speed]`, where the speed table `[4, 2, 1, 0]` gives frames per step (Speed 3 = every frame). Statically decoded from the re-exported disassembly (2026-06-26), confirmed against live RAM and the PSX Orbonne-chapel cascades, mirrored in the Godot reimplementation (`Unit._tick_rotate` driven by `ScenarioVM._tick_unit_rotations`), and verified end-state on all 5 Rotate Unit opcodes of the chapel cinematic.

## Points

- **`FUN_8013f20c` is the per-vsync tick consumer for `{2D}` Rotate Unit: it loops 21 handles at `0x8016d9d8 + handle*7`, dispatches on the Direction byte, and steps the unit's facing by one 0x100 direction (±1 on the 16-direction wheel) once `counter >= DAT_80169750[Speed]`.** — `[S·D·R] 3/3`
  - S: `FUN_8013f20c` (consumer entry), `0x8013f330` (Direction branch), `0x8013f388` (facing writer call `FUN_8012db00`), `0x80148188`–`0x801481c8` (handler writes to slot) (`battle_disassembly.txt`, re-exported 2026-06-26)
  - D: PCSX agent live RAM session (port 8087, `-debugger`) + `pcsx_run.jsonl` chapel cascades (2026-06-26)
  - R: `godot-learning/src/units/Unit.gd` (`Unit._tick_rotate`), `godot-learning/src/scenarios/ScenarioVM.gd` (`_tick_unit_rotations`) — validated by chapel-trace rerun (all 5 Rotate Unit opcodes target-aligned)
  - src: `research/working_documents/chapel_opcode_trace/HANDOFF_rotation_interpolation.md`
- **The per-handle rotate slot at `0x8016d9d8 + handle*7` is 7 bytes: +0 target byte, +1 Direction, +2 speed_value, +3 counter, +4 active flag, +6 Delay countdown.** — `[S] 1/3`
  - S: `DAT_8016d9d8` slot table, `FUN_8013f20c` decode (`battle_disassembly.txt`, re-exported 2026-06-26)
  - src: `research/working_documents/chapel_opcode_trace/HANDOFF_rotation_interpolation.md`
- **The speed table `DAT_80169750` is `[4, 2, 1, 0]`: Speed 0..3 → 4/2/1/0 frames per step (Speed 3 = step every frame), and a live RAM read matches the static decode bit-exactly.** — `[S·D] 2/3`
  - S: `DAT_80169750` (`battle_disassembly.txt`)
  - D: live RAM read of `0x80169750` via PCSX agent (2026-06-26) — `[4,2,1,0]` ✓
  - S: same table as BATTLE.BIN `0x00102750` (base `0x80067000` + offset, one byte/value); wiki `{2D}` RotateUnit article lists the frame lengths (user-provided 2026-07-07)
  - src: `research/working_documents/chapel_opcode_trace/HANDOFF_rotation_interpolation.md`
- **Direction byte semantics: 0 = shortest path, 1 = CW (12-bit angle increasing), 2 = CCW (decreasing) — confirmed by the PSX chapel cascades: Agrias (slot 1, uid 0x34, Dir=2) goes 0xC00→0x400 in 8 steps of −0x100 over 16 vsyncs (vs 7341–7356), Simon (slot 0, uid 0x13, Dir=1) goes 0xC00→0x200 in 6 steps of +0x100.** — `[S·D] 2/3`
  - S: `0x8013f330` (Direction branch) (`battle_disassembly.txt`)
  - D: `pcsx_run.jsonl` chapel cascades, slot 1 vs 7341–7356 / slot 0 final @vs=7385 (2026-06-26)
  - src: `research/working_documents/chapel_opcode_trace/HANDOFF_rotation_interpolation.md`
- **The Godot reimplementation mirrors `FUN_8013f20c` per tick: `Unit.scenario_rotate` records the precise 12-bit `facing_angle` and arms `_rotate_state` (delay countdown, counter vs `_ROTATE_SPEED_TABLE`, step via `_pick_step_direction`) instead of teleporting, and `ScenarioVM._tick_unit_rotations` drives it per 60 Hz VM tick — chapel-trace rerun: all 5 Rotate Unit opcodes match the PSX end states (pc 95 Agrias EAST(0x400) @vs=7357, pc 97 Simon EAST(0x200) @vs=7385, pc 193 Agrias WEST(0x900), pc 196 Simon NORTH(0xD00), pc 199 Agrias EAST(0x400)).** — `[D·R] 2/3`
  - D: PSX ground truth from `pcsx_run.jsonl` (2026-06-26)
  - R: `godot-learning/src/units/Unit.gd` (`scenario_rotate`, `_tick_rotate`, `_ROTATE_SPEED_TABLE`, `_pick_step_direction`), `godot-learning/src/scenarios/ScenarioVM.gd` (`_tick_unit_rotations`) — validated by chapel-trace rerun (`report.md`)
  - src: `research/working_documents/chapel_opcode_trace/HANDOFF_rotation_interpolation.md`
- **Chapel trace mode drains each in-flight rotation to target (`_drain_unit_rotations_to_target`) before emitting the Rotate Unit opcode's trace row, so each row's post-state matches the PSX render after the cinematic Wait that follows.** — `[R] 1/3`
  - R: `godot-learning/src/scenarios/ScenarioVM.gd` (`_drain_unit_rotations_to_target`, `--chapel-trace`) — validated by chapel-trace rerun (target-aligned rows in `report.md`)
  - src: `research/working_documents/chapel_opcode_trace/HANDOFF_rotation_interpolation.md`
- **FFT's 12-bit facing wheel is S=0x000, E=0x400, W=0x800, N=0xC00 (16 directions, 0x100 steps) in wheel order S→E→W→N — not uniformly geometric (E and W are 180° apart geographically but only 4 wheel steps apart); Godot's debug arrow maps it by lerping within each 4-byte segment (`facing_angle_to_world_radians`), following the cinematic author's path-of-bytes rather than a great-circle minimum.** — `[R] 1/3`
  - R: `godot-learning/src/units/Unit.gd` (`facing_angle_to_world_radians`), `godot-learning/src/animation/AnimationStateController.gd` (`angle_12bit_to_facing`), `godot-learning/src/scenarios/ScenarioPlayerScene.gd` (`_apply_facing_arrows`) — validated by chapel-trace (EAST(0x400)/EAST(0x200) cells match PSX)
  - ⚠ SUPERSEDED (2026-08-15) by: Godot's facing arrow uses a uniform 22.5°/byte rotation (`Unit.facing_angle_to_world_radians`, commit b91e5342) — the Agrias chapel cascade 0xC→0x4 (8 byte-steps, CCW) rotates 180°, matching the user's PSX observation
  - src: `research/working_documents/chapel_opcode_trace/HANDOFF_rotation_interpolation.md`
- **The unit struct holds its current facing as a signed 16-bit `current_facing` at offset +0x70 — the field stepped by the per-vsync tick consumer.** — `[ ] 0/3`
  - src: `research/working_documents/chapel_opcode_trace/HANDOFF_rotation_interpolation.md`
- **The unit roster slot base is `0x800B7308` with stride `0x440`.** — `[S·D] 2/3`
  - S: roster base `0x800B7308` stride `0x440` (`battle_disassembly.txt`)
  - D: probe `probe_cinematic_actor.lua` (2026-06-27): 8 unique `a0` slot pointers across 4 s, stride exactly `0x440`, all aligning to `0x800B7308 + slot*0x440`
  - src: `research/working_documents/chapel_opcode_trace/HANDOFF_rotation_interpolation.md`
- **Roster slot field +0x161 holds `ENTD.unit_id` (empty in the `orbonne_prayer_pre_scenario_load` sstate — slot↔uid inferred behaviourally in the chapel trace report).** — `[ ] 0/3`
  - ⚠ SUPERSEDED (2026-08-15) by: `+0x161` is NOT the sprite uid — during the active chapel cinematic every occupied roster slot reports `+0x161 = 0x00`; the sprite identifier is the `+0x06..07` sprite-set ID (Agrias `0x0034`)
  - src: `research/working_documents/chapel_opcode_trace/HANDOFF_rotation_interpolation.md`
- **Godot's facing arrow now uses a uniform 22.5°/byte rotation instead of per-segment lerping: `Unit.facing_angle_to_world_radians` (commit b91e5342) rotates the arrow the geometrically-correct amount, and the Agrias chapel cascade 0xC→0x4 (8 byte-steps, Direction=2 CCW) = 180°, matching the user's PSX observation.** — `[D·R] 2/3`
  - D: headful side-by-side (PCSX 8082 + Godot ScenarioPlayer) user observation (2026-06-27): arrow turns 180° on the 0xC→0x4 cascade
  - R: `godot-learning/src/units/Unit.gd` (`facing_angle_to_world_radians`, commit b91e5342) — no named test
  - src: `research/working_documents/chapel_opcode_trace/HANDOFF_sprite_cardinal_mapping.md`
- **The event-handler facing write path: the selector iterator feeds facing-mode dispatch `FUN_8008c0ac`/`FUN_8008c114`, which seeds the unit's facing via `sh facing, 0x70(unit)` @ `0x8008c154`; the per-vsync stepper then walks the byte one step (0x100) per tick.** — `[S·D] 2/3`
  - S: `0x8008c154` (seed write), `FUN_8008c0ac`/`FUN_8008c114` (BATTLE.BIN disassembly)
  - D: live facing values across scn6 beats — PC30 0xC00 / PC40 0x800 / PC117 0x800 / PC205 0x400 (PSX `+0x70`, 2026-07-07)
  - R: none — `0x8008c154`/this write path not present in godot-learning (Godot seeds `facing_angle` through its own `ScenarioApply` path)
  - src: `research/working_documents/EVENT_UNIT_SET_RESOLUTION.md`

## Notes

(empty — user territory)

## Related

- [[Event Opcode Catalog]]
- [[Unit Anim Opcode]]
- [[Wait Value Opcode]]
- [[ENTD Unit Deployment Table]]
- [[Sprite Cardinal Pose Selection]]
- [[Cinematic Sprite Renderer]]
- [[Event Unit Selector]]
- [[Face Tile Opcode]]
