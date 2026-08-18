# Walk To Opcode

The event instruction `{28}` Walk To walks a scenario unit along a grid route: dispatch at `0x8013ea1c` spawns a fiber (`FUN_8013e5c0`) that arms the movement (`unit_movement_arm` `FUN_8008c664`, shared with battle move, `a0` = unit id) and a per-frame stepper (`FUN_8006af7c`) integrates position while the per-step arm (`FUN_80069e68`) picks the walk anim from the unit's facing field `+0x70`. The ROM keeps one facing field: the walk re-faces `+0x70` to the travel cardinal at walk start and holds it (no cardinal/precise split) on the 12-bit wheel `0x000=E, 0x400=S, 0x800=W, 0xC00=N` — pinned live in scenario 6 (2026-07-09). Godot had kept that single field as two decoupled stores (`facing_angle` 12-bit + `facing_direction` enum) and the Walk To wrote only the enum, so stale-angle walks rendered sideways across the roster (measured: no VM walk reliably correct); fixed in two stages — Stage 1 (commit `7f89bf37`) routes the walk through `PsxNum.heading_to_12bit` and writes `facing_angle` via `scenario_set_facing`, Stage 2 makes the `facing_direction` setter a forward-converting choke point (`_write_facing_angle`) so the stores can no longer diverge.

## Points

- **The ROM's `{28}` Walk To re-faces the single unit facing field `+0x70` to the travel cardinal at walk start and holds it for the whole walk — no cardinal/precise split on hardware, the same `+0x70` holds the Warp value, then the walk value, then the post-walk Rotate value (scenario-6 chocobo: Warp `0x800` W → every walking sample `0x0C00` N while moving +gridX → end sample `0x0E00`, settling into the post-walk Rotate at PC 191).** — `[S·D·R] 3/3`
  - S: `{28}` dispatch `0x8013ea1c` → fiber `FUN_8013e5c0` → stepper `FUN_8006af7c`; per-step arm `FUN_80069e68` = `set_unit_animation_with_flags(0x3c, unit+0x70)`; cardinal quantizer `FUN_8008c1e4` (`battle_disassembly.txt`)
  - D: scenario 6 live capture, slot 6 @ `0x800B8C88` (`research/scripts/probe_chocobo_slot6.py`, PCSX-Redux port 8087, 2026-07-09, ref `scenario6_abduct_punch_pickup_start.sstate`) — pre-walk `+0x70 = 0x0800` undrawn (`+0x6e = 0xFF`), every mst=3/6 sample `0x0C00` (N) with render X `+0x40` climbing `0x…46 → 0x…9a`
  - R: commit `7f89bf37` — `PsxNum.heading_to_12bit(dx,dz)` (`godot-learning/src/scenarios/PsxNum.gd`) + `ScenarioApply.walk_to` / `ScenarioWorld.set_walking`/`set_walk_facing` write `facing_angle` via `scenario_set_facing` — validated by `ScenarioWalkFacingAngleTest`
  - src: `research/working_documents/CHOCOBO_WALK_OCTANT_28.md`
- **The Walk To execution chain is dispatch `0x8013ea1c` → fiber `FUN_8013e5c0` → arm `unit_movement_arm` `FUN_8008c664` (shared with battle move; `a0` is the unit id, resolved internally — NOT a struct pointer) → per-frame stepper `FUN_8006af7c`; the per-step arm `FUN_80069e68` picks the walk anim from `+0x70` each step, and the route consumer `FUN_80069af8` reads the per-tile cardinal (`route_byte >> 6`) to build the step velocity.** — `[S·D] 2/3`
  - S: `0x8013ea1c`, `FUN_8013e5c0`, `FUN_8008c664`, `FUN_8006af7c`, `FUN_80069e68`, `FUN_80069af8` (`battle_disassembly.txt`)
  - D: scenario 6 slot-6 trace (2026-07-09) — tile advance (2,0)→(3,0)→(5,0), mst alternating 3/6, anim id 0004→0000→0003
  - src: `research/working_documents/CHOCOBO_WALK_OCTANT_28.md`
- **The Walk To's X/Y/Z operands are grid-X / grid-Z / height byte: the parser pre-flips Y into the grid-Z row (ADR-0057) and the Z height byte is ignored because the map derives world-Y — scenario-6's `Walk To {X 5, Y 0, Z 0}` walked the chocobo grid x 2→5 along row z=0.** — `[S·D] 2/3`
  - S: `assets/scenarios/chunks/scenario_006_chunk.json` PC 188 raw `28 8b 00 05 00 00 00 08 00` (runtime chunk base `*0x80173CA4 = 0x8004a6bc`), parameter semantics per ADR-0057
  - D: scenario 6 slot-6 trace (2026-07-09) — tile X advanced 2→3→5 with tile Z held at 0
  - src: `research/working_documents/CHOCOBO_WALK_OCTANT_28.md`
- **`Warp Unit` writes its Facing operand into `+0x70` as `facing << 10`: scenario-6 PC 15 `Warp Unit {139, X 2, Y 0, Facing 2}` staged the chocobo at grid (2,0) with `+0x70 = 0x0800` = WEST.** — `[S·D] 2/3`
  - S: `assets/scenarios/chunks/scenario_006_chunk.json` PC 15, runtime chunk base `0x8004a6bc = *0x80173CA4`
  - D: scenario 6 pre-walk sample (2026-07-09) — `+0x70 = 0x0800` with `+0x6e = 0xFF` (undrawn), uniquely matching the chocobo's Warp
  - src: `research/working_documents/CHOCOBO_WALK_OCTANT_28.md`
- **The Godot VM-unit renderer picks the pose octant from the precise `facing_angle`, not the `facing_direction` enum: `UnitDisplay.gd:361` reads `facing_angle` (falling back to `_CARDINAL_TO_12BIT[facing]` only when it is < 0) into `get_pose_octant(fa, psx_cam)`, and the walk anim id 15 (< 0x1F4) takes the low-range SEQ branch that uses `pose_octant`.** — `[R] 1/3`
  - R: `godot-learning/src/animation/UnitDisplay.gd` (view `facing_angle` → `get_pose_octant`), `godot-learning/src/animation/AnimationStateController.gd` (`get_pose_octant`) — no named test
  - src: `research/working_documents/CHOCOBO_WALK_OCTANT_28.md`
- **Measured pre-fix (2026-07-09): because the Walk To wrote only `facing_direction`, every VM walk whose stale `facing_angle` ≠ travel cardinal rendered sideways — 0x8B chocobo (fa=0x800 W, travel N), 0x83 (fa=0x000 E, travel S), 0x02 Delita (fa=0x800 W, travel S), Agrias stair walk (fa=0xC00 N, travel W); only 0x17 and Agrias's first walk (fa=0x800 W, travel W) happened to align, so no VM walk was reliably correct — the stale-write sites were `ScenarioWorld.set_walking`/`set_walk_facing`.** — `[D] 1/3`
  - D: scenario 6 headful via group path (root 3 → member 6), temporary per-frame log in `ScenarioVM._advance_motions` (`/tmp/scen6_walkdbg.log`, 2026-07-09)
  - src: `research/working_documents/CHOCOBO_WALK_OCTANT_28.md`
- **Stage 2 of the fix makes the `Unit.facing_direction` setter a choke point that forward-converts the cardinal and writes `facing_angle` through a single `_write_facing_angle` path (scenario verbs + `_tick_rotate` funnel through it), derives the enum getter from `facing_angle`, and de-overloads the `facing_angle == -1` sentinel into an explicit `Unit.is_cinematic_unit` so a combat unit with a real `facing_angle` still renders combat idle — the two facing stores can no longer diverge.** — `[R] 1/3`
  - R: `godot-learning/src/units/Unit.gd` — validated by `UnitCinematicIdleModeTest` + `UnitOrientationTest`
  - src: `research/working_documents/CHOCOBO_WALK_OCTANT_28.md`
- **The roster walk fields move in lockstep during a Walk To: `+0x7f` move state (walking = 3/6 alternating, idle = 0), `+0x6e` render-facing octant (0xFF = undrawn), `+0x0C` anim id, `+0x40`/`+0x44` render X/Z, `+0x7c`/`+0x7e` tile X/Z (roster base `0x800B7308`, stride `0x440`).** — `[D] 1/3`
  - D: scenario 6 slot-6 trace (2026-07-09) — rfac ff→02→03, mst 00→3/6→00, anim 0004→0000→0003, render X `+0x40` climbing `0x…46 → 0x…9a`; field map per `probe_orientation.lua`
  - src: `research/working_documents/CHOCOBO_WALK_OCTANT_28.md`

## Notes

(empty — user territory)

## Related

- [[Block Execution]]
- [[Rotate Unit Interpolation]]
- [[Sprite Cardinal Pose Selection]]
- [[Unit Anim Opcode]]
- [[Event Opcode Catalog]]
