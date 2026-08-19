# Walk To Opcode

The event instruction `{28}` Walk To walks a scenario unit along a grid route: dispatch at `0x8013ea1c` spawns a fiber (`FUN_8013e5c0`) that arms the movement (`unit_movement_arm` `FUN_8008c664`, shared with battle move, `a0` = unit id) and a per-frame stepper (`FUN_8006af7c`) integrates position while the per-step arm (`FUN_80069e68`) picks the walk anim from the unit's facing field `+0x70`. The ROM keeps one facing field: the walk re-faces `+0x70` to the travel cardinal at walk start and holds it (no cardinal/precise split) on the 12-bit wheel `0x000=E, 0x400=S, 0x800=W, 0xC00=N` — pinned live in scenario 6 (2026-07-09). Godot had kept that single field as two decoupled stores (`facing_angle` 12-bit + `facing_direction` enum) and the Walk To wrote only the enum, so stale-angle walks rendered sideways across the roster (measured: no VM walk reliably correct); fixed in two stages — Stage 1 (commit `7f89bf37`) routes the walk through `PsxNum.heading_to_12bit` and writes `facing_angle` via `scenario_set_facing`, Stage 2 makes the `facing_direction` setter a forward-converting choke point (`_write_facing_angle`) so the stores can no longer diverge. The walk cadence is now ROM-derived and live-validated: constant velocity, no ease-in, frames per tile = 0x1C000/(Speed<<9) = 224/Speed (14 @ Speed 16), because the Speed operand scales the velocity field (`unit+0x38 = Speed<<9`); Godot uses that cadence via `ScenarioDecode.walk_to_duration_frames`. The Speed byte is signed and lands on the shared battle velocity constants — `+0x08` Walk → 28 f/tile, `+0x10` chase → 14, `+0x24` Run → 6.2 — and the scenario-6 chase walk (instr 334) is deliberately fire-and-forget: no `{29} Wait Walk` until the ride-off `Block Start` (instr 346), so the walk→ride-off gap is a speed relationship, not a scheduling one.

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
- **The `{28} Walk To` ground walk moves at constant velocity with no ease-in: frames per tile = 0x1C000/(Speed<<9) = 224/Speed (14 @ Speed 16; one tile = 0x1C000 position units), and at that cadence the scenario-6 ride-off head-start lands at 1.56 tiles (PSX ≈ 1.5).** — `[S·D·R] 3/3`
  - S: arm `FUN_8008c664` seeds velocity `unit+0x38` @ `0x8008C77C`, per-frame vel scale `>>12` @ `0x80069DFC`, `{28}` case ≈ `0x80144CC8` (`battle_disassembly.txt`)
  - S: `DAT_80045980` (unit-movement delta multiplier) = 1 in event state → `frame_sync` @ `0x80093A98` runs one game-loop iteration per VBlank, so 224/Speed is in VBlanks/tile; the `+0x925`/frame ramp (`DAT_80096128`) exists only in the `+0x1c`/`+0x2c` vertical cases = the jump arc, not ground-walk ease-in (`battle_disassembly.txt`)
  - D: scenario 6 chase live polling, savestate `scenario6_delita_tough_dialogue_pc334.sstate` instr 334 (2026-07-06) — 14.00 stepper-calls/tile @ Speed 16, Agrias = slot 0 @ `0x800B7308`
  - D: scenario 6 chase live polling (2026-07-06) — `DAT_80045980 = 1` measured, confirming the one-iteration-per-VBlank stepper clock
  - R: `godot-learning/src/scenarios/ScenarioDecode.gd` `walk_to_duration_frames` + `ScenarioApply.gd` `walk_to` — validated by `godot-learning/tests/ScenarioDecodeTest.gd` (14 f/tile @ Speed 16, 28 @ 8) + `ScenarioWalkToAnimTest` (`_test_constant_and_formula`)
  - src: `research/working_documents/HANDOFF_walk_to_cadence_derivation.md`
- **The Speed operand of `{28} Walk To` scales the unit velocity field — the arm seeds `unit+0x38 = Speed<<9` — rather than feeding the route pathfinder (the earlier "Speed feeds the pathfinder" fork was a 16-bit-operand misread).** — `[S·D·R] 3/3`
  - S: velocity write `0x8008C77C` in arm `FUN_8008c664`; pathfinder `0x8017813C` reached via `0x801780EC` (`battle_disassembly.txt`)
  - D: scenario 6 chase live validation (2026-07-06) — 14.00 stepper-calls/tile @ Speed 16 matches the `Speed<<9` velocity model
  - R: `godot-learning/src/scenarios/ScenarioApply.gd` `walk_to` (Speed feeds the `ScenarioPathMotion` cadence; the `EventPathfinder` plan stays terrain-only, speed-free) + `ScenarioDecode.gd` `walk_to_duration_frames` — validated by `ScenarioDecodeTest`
  - src: `research/working_documents/HANDOFF_walk_to_cadence_derivation.md`
- **A battle unit's ground-walk position lives in a 32-bit field at `unit+0x18` (tile index read as `+0x18>>12`); the Walk To arm writes the Speed-scaled velocity to `+0x38` and `0x2000` to `+0x3c` (@ `0x8008C77C`/`0x8008C784` in `FUN_8008c664`) plus the path index to `+0x98`.** — `[S·D] 2/3`
  - S: `0x8008C77C` (`unit+0x38`), `0x8008C784` (`unit+0x3c`); arm `FUN_8008c664` writes `unit+0x98` path idx (`battle_disassembly.txt`)
  - D: scenario 6 chase session (2026-07-06) — per-frame `+0x18` polling produced the 14.00/tile rate
  - R: none — PSX unit struct offsets (+0x18/+0x38/+0x3c/+0x98) not present in godot-learning (cited in `ScenarioDecode.gd` comments only)
  - src: `research/working_documents/HANDOFF_walk_to_cadence_derivation.md`
  - ⚠ SUPERSEDED (2026-08-18) by: `+0x18 >> 0xC` yields tile×28, not a tile index — the ×28 sub-tile pitch (`0x1C000 = 28 × 0x1000`) is folded into `+0x18`, and `unit+0x40` is the `tile×28 + 14` screen projection (the `+14` half-tile centering term is dropped by `>>0xC`)
- **The unit array stride `0x440` (roster base `0x800B7308`) holds only for the ride-off units u0–u7 — the live `+0x18` position field of units beyond that group was not located at that stride from that base.** — `[D] 1/3`
  - D: scenario 6 chase session, unit-array stride inspection, savestate `scenario6_delita_tough_dialogue_pc334.sstate` instr 334 (2026-07-06)
  - R: none — 0x440 stride not present in godot-learning
  - src: `research/working_documents/HANDOFF_walk_to_cadence_derivation.md`
- **The `{28} Walk To` case in the event interpreter `FUN_80143bd8` is at ≈ `0x80144CC8`, and its movement stack additionally includes the route pathfinder `FUN_8017813c` (called via `0x801780EC`), the per-frame velocity scale `>>12` at `0x80069DFC`, and the direction setup at `0x800694D8`.** — `[S·R] 2/3`
  - S: `0x80144CC8`, `0x801780EC`/`0x8017813C`, `0x80069DFC`, `0x800694D8` (`battle_disassembly.txt`)
  - R: `godot-learning/src/scenarios/EventPathfinder.gd` (port of ROM `FUN_8017813c` chain `0x801780EC`→`0x8017813C`) + `ScenarioEventPathfinderTest` — covers the pathfinder part only
  - src: `research/working_documents/HANDOFF_walk_to_cadence_derivation.md`
- **The scenario-6 chase `Walk To` (instr 334, unit 52 → tile (4,0), Speed 16) is deliberately fire-and-forget: unlike her two other walks in the chunk (barriered by `Wait Walk` at 56-61 and 264-266), there is no `{29} Wait Walk` between instr 334 and the ride-off `Block Start` at 346, so the main context burns the ~20-tick scripted budget (`Wait 9` + `Wait 10` + Delita's rotate) and reaches the ride-off while Agrias is still walking — the walk→ride-off gap is set purely by walk speed, not by scheduling.** — `[S·D·R] 3/3`
  - S: `assets/scenarios/chunks/scenario_006_chunk.json` instrs 334-346 — 334 `Walk To {unit 52, X 4, Y 0, Speed 16}`, 335 `Walk To Anim`, 338 `Wait 9`, 340 `Wait 10`, 346 `Block Start`; no `{29}` between 334 and 346
  - D: scenario 6 PSX live polling, pcsx-redux port 8080, savestate `scenario6_delita_tough_dialogue_pc334.sstate` (2026-07-06) — ride-off Sprite-Moves begin ~0.44 s (~26 frames) after the O-press, ~18-20 ticks after the walk; Godot probe `tools/probe_scenario6_chase_timing.gd` (2026-07-06) — main PC crosses 333→341 in ~19 ticks with Agrias mid-walk, ride-off arms on schedule
  - R: `godot-learning/src/scenarios/ScenarioVM.gd` — `_op_walk_to` (L3173) arms the motion and returns without arming a wait; `{79}` `WALK_TO_ANIM` → `_op_skip` (L773); `{29}` `WAIT_WALK` → `_op_wait_walk` (L763) exists but is absent from this stream — headful-validated by `tools/probe_scenario6_chase_timing.gd` (ride-off fires on schedule, Agrias mid-walk)
  - src: `research/working_documents/SCENARIO6_CHASE_WALK_TIMING.md`
- **The `{28} Walk To` Speed operand is a signed byte that lands the shared `unit+0x38` velocity magnitude on the battle walk-speed constants: `+0x08` Walk → `0x1000` → 28 f/tile, `+0x10` (chase) → `0x2000` → 14 f/tile, `+0x24` Run → `0x4800` → 6.2 f/tile — the Run byte is disputed between community sources (`0x20` per instructions.php vs `0x24` per Event_Instruction_28), but both exceed the Haste band `0x3000` and read as "run".** — `[S·D·R] 3/3`
  - S: arm write `0x8008C77C` in `FUN_8008c664` (`battle_disassembly.txt`); battle constants (floor `0x1000`, normal walk `0x2000`, Haste `> 0x3000`) per ffhacktics `BATTLE.BIN_Data_Tables` RAM map; speed bytes per ffhacktics `Event_Instruction_28` / `instructions.php`
  - D: scenario 6 chase live polling (2026-07-06) — Speed 16 → `unit+0x38 = 0x2000` measured
  - R: `godot-learning/src/scenarios/ScenarioDecode.gd` `walk_to_duration_frames` — validated by `tests/ScenarioDecodeTest.gd` (224/16 = 14 f/tile; 224/8 = 28 f/tile @ Walk byte `0x08`) + `tests/ScenarioApplyTest.gd` `_test_walk_to_arms_motion_faces_and_clears_home` (224/speed × segments)
  - src: `research/working_documents/SCENARIO6_CHASE_WALK_TIMING.md`
- **The `{28} Walk To` cannot cross water — event scripts use `{70} Jump` for water crossings (community wiki datum; no BATTLE.BIN grounding in this doc).** — `[ ] 0/3`
  - R: none — no water-crossing / `{70} Jump` event semantics in godot-learning (probed `godot-learning/src/`, `godot-learning/tests/`)
  - src: `research/working_documents/SCENARIO6_CHASE_WALK_TIMING.md`

## Notes

(empty — user territory)

## Related

- [[Block Execution]]
- [[Rotate Unit Interpolation]]
- [[Sprite Cardinal Pose Selection]]
- [[Unit Anim Opcode]]
- [[Event Opcode Catalog]]
- [[Scenario 6 Ride Off]]
- [[Unit Sprite Object Struct]]
