# Chirality Flip

ADR-0052: the Godot map frame is 180°-about-X flipped relative to the PSX map frame — the map mesh and ENTD spawn tiles are flipped at parse time. Because scenario chunk JSON stays committed raw (byte-faithful to PSX RAM, required for the probe diffs), in-chunk unit depth rows (Warp/Walk To operands) are mirrored at the runtime consume-boundary instead: `godot_z = map_size_z − 1 − psx_z`, with `map_size_z` supplied by the host from `terrain.json`. The camera pose was empirically calibrated against the raw frame and was NOT flipped — that raw consume is now superseded for the camera BODY's depth, which is ADR-0052-flipped at the consume boundary (2026-06-29; see [[Scenario Camera Framing]]). Verified on the scenario-1 chapel walk-in (2026-06-28): post-fix seats (3,5)/(2,6)/(2,4) are the mirrors of the raw rows, and the earlier "fixed" state was a false positive that compared Godot's raw tile to PSX's raw tile, skipping the flip.

## Points

- **The Godot map frame (mesh + ENTD spawn) is flipped 180° about X from the PSX frame (ADR-0052), so a unit's Godot depth row must be `map_size_z − 1 − psx_z`; the scenario chunk stays committed raw (== PSX RAM, for probe diffs) and the VM mirrors the in-chunk Warp/Walk depth rows at runtime, while the camera pose is NOT flipped.** — `[D·R] 2/3`
  - D: post-fix chapel trace — Gafgarion (3,5), Ramza (2,6), squire (2,4) = mirrors of raw rows 4/3/5 on the flipped map (2026-06-28); the pre-fix "verified" state had Ramza raw row 3 → stranded 3 tiles shallow
  - R: `godot-learning/src/scenarios/PsxNum.gd` `flip_depth_row` (map frame mirror) + host `map_size_z` from `terrain.json` in `src/scenarios/ScenarioPlayerScene.gd` `_load_map_size_z` — validated by `tests/PsxNumTest.gd` (flip cases) + `tests/ScenarioSpriteMoveTest.gd` `_test_walk_to_consumes_preflipped_rows` (chapel seats, raw→flipped)
  - src: `research/working_documents/scenario_1_captures/HANDOFF_unit_tile_alignment.md`
  - ⚠ SUPERSEDED (2026-08-19) by: the camera body's depth IS ADR-0052-flipped at consume — `godot_pos.z = map_size_z − opcode_Y/112` via `PsxNum.flip_depth_continuous` (`camera_flip_body_depth` default ON), verified to the pixel against the live per-unit screen store (Agrias 107.6 vs 107, Ovelia 148.4 vs 147, priest 66.7 vs 68) — see [[Scenario Camera Framing]]

## Notes

(empty — user territory)

## Related

- [[ENTD Unit Deployment Table]]
- [[Scenario Camera Opcodes]]
- [[Scenario Camera Framing]]
- [[Walk To Opcode]]
- [[Isometric Coordinate System]]
