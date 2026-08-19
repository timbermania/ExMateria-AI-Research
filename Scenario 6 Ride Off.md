# Scenario 6 Ride Off

The end beat of scenario 6 ("Abducting the Princess", map 56, ENTD 387): after Agrias (unit 52) chases the fleeing Delita with a fire-and-forget `Walk To` (instr 334 — see [[Walk To Opcode]]), the chocobo trio (Delita 5, Ovelia 12, chocobo 139) slides off toward the lower-right exit via chunk instrs 344–401 in three concurrent `Block Start` groups. On PSX **only Ovelia rides — mounted on the chocobo's back** (a `+Z`=45 *descent*: height field `[0x62]` from the ledge's −132 to −87 ≈ the back's −84; `+Y`=66 is depth) **while Delita leaves on foot**. The ROM's `Sprite Move` operand→offset-field pairing (`+X`→`[0x60]`, `+Z`→`[0x62]` height, `+Y`→`[0x64]` depth) is live-proven and Godot's `sprite_move_offset` matches it — the Ovelia↔chocobo gap matches PSX to sub-tile on all three axes, so no axis fix is needed (the prior "swapped `+Y`/`+Z`" root cause is refuted); movement, block concurrency, and Erase/Remove teardown are all faithful, and the ride-off is re-based mid-scene by a pc302/pc303 Warp that Godot now handles by disarming the in-flight slide. The remaining gap is the ride-phase pose: no `{11} Unit Anim` for units 5/12 in the window, so Godot holds the carry poses `0x1FF`/`0x201` (chocobo reads riderless) where PSX shows mounted-rider + walking-Delita. PSX ground truth (2026-07-05/06, pcsx-redux port 8080): the ride-off Sprite-Moves begin ~0.44 s (~26 frames) after the O-press and their sub-tile offsets ramp to exactly the scripted operands, while Agrias keeps a ~1.5-tile head start. Sibling facets in `research/working_documents/`: `SCENARIO6_RIDE_OFF_CHOCOBO.md` (this topic), `SCENARIO6_CARRY_POSE_EVTCHR_RENDER.md`, `SCENARIO6_CARRY_COMPOSITION_DEPTH.md`, `SCENARIO6_CHASE_WALK_TIMING.md`.

## Points

- **PSX scenario-6 ride-off: the ride-off Sprite-Move blocks (instrs 346-401) begin ~0.44 s (~26 frames) after the O-press, run ~1.1 s, and the three units' `[0x60]` sub-tile offset fields ramp from their parked values to exactly the ride-off operands — Delita 5: −10 → −150, chocobo 139: 0 → −140, Ovelia 12: −72 → −212 — then freeze at ~t=1.57 s.** — `[S·D] 2/3`
  - S: ride-off operands −150 / −140 / −212 in `assets/scenarios/chunks/scenario_006_chunk.json` instrs 346-401; unit struct bases Delita `0x800b8848`, chocobo `0x800b8c88`, Ovelia `0x800ba608` (per `SCENARIO6_RIDE_OFF_CHOCOBO.md` §4b)
  - D: BP-free live polling of `[0x60]` every ~0.12 s, scenario 6 on pcsx-redux port 8080, savestate `scenario6_delita_tough_dialogue_pc334.sstate` (2026-07-06); frame sequence `/tmp/psx_chase/`
  - R: none — the PSX ride-off start delay is PSX-only timing; godot-learning fires the ride-off block on the same ~20-tick scripted schedule (probed `godot-learning/src/scenarios/`)
  - src: `research/working_documents/SCENARIO6_CHASE_WALK_TIMING.md`

- **Scenario-6 ride-off (chunk instrs 344–401): only Ovelia (unit 12) is mounted on the chocobo's back (unit 139) — Delita (unit 5) leaves on foot; all three slide off-screen toward the lower-right exit concurrently, driven by three parallel `Block Start`/`Block End` groups (chocobo 346–358, Ovelia 359–367, Delita 368–376), each = `Sprite Move`×3 → `Erase Unit`, with final teardown `Remove Unit {5},{12},{139}` at 399–401.** — `[S·D·R] 3/3`
  - S: `godot-learning/assets/scenarios/chunks/scenario_006_chunk.json` instrs 344–401 (block brackets + Sprite Move/Erase/Remove operands)
  - D: live PSX frame captures `research/working_documents/scenario6_captures/rideoff/psx_rideoff_ovelia_mounted.png` + `psx_rideoff_sequence.png` — Ovelia (red dress) clearly lifted on the yellow chocobo's back, Delita on foot; scenario 6 on pcsx-redux port 8080 (2026-07-05)
  - R: `godot-learning/src/scenarios/ScenarioApply.gd` (`sprite_move`/`erase_unit`/`remove_unit`) + `ScenarioVM` block spawn; headful probe `godot-learning/tools/probe_scenario6_rideoff.gd` logs units 5/12/139 sliding concurrently
  - src: `research/working_documents/SCENARIO6_RIDE_OFF_CHOCOBO.md`

- **Ovelia's "mount" onto the chocobo is a descent, not a lift: the `Sprite Move` operand `+Z`=45 lowers her height field `[0x62]` from the ledge base height −132 to −87 ≈ the chocobo back −84, while operand `+Y`=66 is depth travel; the mid-ride PSX world position (base+offset) puts Ovelia at (88,−87,388) on top of the chocobo at (76,−84,378).** — `[S·D·R] 3/3`
  - S: `godot-learning/assets/scenarios/chunks/scenario_006_chunk.json` Ovelia Sprite Move operands (X −156/−184/−212, +Z 45, +Y 66) at instrs 359–366
  - D: BP-free live offset-field poll from savestate `scenario6_rideoff_start.sstate` parked at instr 344 — Ovelia `[0x60/0x62/0x64]` (0,0,0) → (−212,45,66); live base/home `[0x40..44]` reads confirm `[0x62]` is the height field (leads (238,−132,322), chocobo (154,−84,378)); pcsx-redux port 8080 (2026-07-05)
  - R: `godot-learning/src/scenarios/ScenarioDecode.gd` `sprite_move_offset` routes +Z to world height: Ovelia ΔY = −45/28 = −1.61 (descends onto the chocobo) and ends at world-Y 3.14 ≈ the chocobo's 3.04 — headful probe `godot-learning/tools/probe_scenario6_rideoff.gd`
  - src: `research/working_documents/SCENARIO6_RIDE_OFF_CHOCOBO.md`

- **The chocobo gallop is driven by `{11} Unit Anim {139, 3 → 4 → 48}` at instrs 347/350/352; 48 = 0x30 is a low-range SEQ anim that Godot renders via the cyoko sprite set (SPR 0x86) clock key `str((0x31−1)×2)` = `"96"` (gallop frames + MoveUnitRL present in the cyoko seq data) — the probe shows unit 139 at anim=0x31 throughout the ride.** — `[S·R] 2/3`
  - S: `godot-learning/assets/scenarios/chunks/scenario_006_chunk.json` instrs 347/350/352
  - R: `godot-learning/src/animation/UnitDisplay.gd:288` (`clock_key = str((current_anim_id − 1) * 2)`) + `src/animation/AnimationResolutionMap.gd` "cyoko" sprite type; headful probe `godot-learning/tools/probe_scenario6_rideoff.gd` observes anim=0x31 with the gallop playing (no dedicated test named)
  - src: `research/working_documents/SCENARIO6_RIDE_OFF_CHOCOBO.md`

- **No `{11} Unit Anim` targets units 5/12 in the ride-off window (instrs 344–401 — only the chocobo's 3→4→48), so the ride-phase pose is inherited from the carry beat: the Godot probe holds both leads in carry anims `0x1FF`/`0x201` through the whole ride (chocobo reads riderless, leads clump near the tower) while the live PSX frames show Ovelia as a distinct mounted rider and Delita walking on foot.** — `[S·D·R] 3/3`
  - S: absence of `{11}` for units 5/12 in `godot-learning/assets/scenarios/chunks/scenario_006_chunk.json` instrs 344–401
  - D: PSX capture `research/working_documents/scenario6_captures/rideoff/psx_rideoff_ovelia_mounted.png` (mounted rider + walking Delita) vs Godot `godot_rideoff_chocobo_riderless.png` + `godot_rideoff_leads_carry.png` (2026-07-05)
  - R: `godot-learning/tools/probe_scenario6_rideoff.gd` logs units 5/12 at anim `0x201`/`0x1FF` throughout the ride even with unit 12's `global_position` on the chocobo
  - src: `research/working_documents/SCENARIO6_RIDE_OFF_CHOCOBO.md`

- **The ride-off is re-based mid-scene by a `{24}` Warp: pc302 zeroes Delita's Sprite-Move offset, then pc303 `Warp Unit 5` → tile (6,0) re-plants his base for the ride — so Godot's warp must cancel the in-flight `ScenarioMotion`, or the stray pre-warp slide re-lerps Delita back to the old ledge home and every ride move resolves against the ledge (the "walks back UP the stairs" symptom); `ScenarioApply.warp` now calls `ScenarioWorld.disarm_motion` right after placement — the Godot analogue of the PSX warp zeroing the `+0x60` offset — and post-fix Delita ends at (1.14, 3.82, −0.32), just behind the chocobo (0.50, 3.04, 0.50).** — `[S·R] 2/3`
  - S: `godot-learning/assets/scenarios/chunks/scenario_006_chunk.json` pc302/pc303 (unit-5 warp re-base to tile (6,0))
  - R: `godot-learning/src/scenarios/ScenarioApply.gd` `warp` → `world.disarm_motion` + `clear_actor_home` — validated by `ScenarioApplyTest` 158/0 (+3 disarm asserts across the warp cases), `ScenarioSpriteMoveTest` 79/0, `ScenarioWarpFacingTest` 11/0, `ScenarioActorTest` 41/0; root-caused by `godot-learning/tools/probe_scenario6_freeze.gd` (2026-07-05)
  - src: `research/working_documents/SCENARIO6_RIDE_OFF_CHOCOBO.md`

## Notes

(empty — user territory)

## Related

- [[Walk To Opcode]]
- [[Block Execution]]
- [[Face Tile Opcode]]
- [[Unit Anim Opcode]]
- [[Scenario 6 Punch Pickup Throw]]
- [[Event Opcode Catalog]]
