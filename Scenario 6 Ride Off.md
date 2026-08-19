# Scenario 6 Ride Off

The end beat of scenario 6 ("Abducting the Princess", map 56, ENTD 387): after Delita's "Tough…Don't blame us. Blame yourself or God." line, Agrias (unit 52) chases the fleeing Delita with a fire-and-forget `Walk To` (instr 334 — see [[Walk To Opcode]]), and the chocobo trio (Delita 5, Ovelia 12, chocobo 139) slides off the dock via the `Block Start` at instr 346. PSX ground truth (2026-07-06, pcsx-redux port 8080): the ride-off Sprite-Moves begin ~0.44 s (~26 frames) after the O-press and their sub-tile offsets ramp to exactly the scripted operands, while Agrias keeps a ~1.5-tile head start. Sibling facets in `research/working_documents/`: `SCENARIO6_RIDE_OFF_CHOCOBO.md` (chocobo walk/carry), `SCENARIO6_CARRY_POSE_EVTCHR_RENDER.md`, `SCENARIO6_CARRY_COMPOSITION_DEPTH.md`, `SCENARIO6_CHASE_WALK_TIMING.md`.

## Points

- **PSX scenario-6 ride-off: the ride-off Sprite-Move blocks (instrs 346-401) begin ~0.44 s (~26 frames) after the O-press, run ~1.1 s, and the three units' `[0x60]` sub-tile offset fields ramp from their parked values to exactly the ride-off operands — Delita 5: −10 → −150, chocobo 139: 0 → −140, Ovelia 12: −72 → −212 — then freeze at ~t=1.57 s.** — `[S·D] 2/3`
  - S: ride-off operands −150 / −140 / −212 in `assets/scenarios/chunks/scenario_006_chunk.json` instrs 346-401; unit struct bases Delita `0x800b8848`, chocobo `0x800b8c88`, Ovelia `0x800ba608` (per `SCENARIO6_RIDE_OFF_CHOCOBO.md` §4b)
  - D: BP-free live polling of `[0x60]` every ~0.12 s, scenario 6 on pcsx-redux port 8080, savestate `scenario6_delita_tough_dialogue_pc334.sstate` (2026-07-06); frame sequence `/tmp/psx_chase/`
  - R: none — the PSX ride-off start delay is PSX-only timing; godot-learning fires the ride-off block on the same ~20-tick scripted schedule (probed `godot-learning/src/scenarios/`)
  - src: `research/working_documents/SCENARIO6_CHASE_WALK_TIMING.md`

## Notes

(empty — user territory)

## Related

- [[Walk To Opcode]]
- [[Block Execution]]
- [[Face Tile Opcode]]
