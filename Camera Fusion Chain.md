# Camera Fusion Chain

The `{1D}`/`{1E}` camera-fusion bracket does not execute its queued `{19}` Cameras op-by-op: the `{1E}` handler `FUN_8013db9c` pre-computes the whole bracket into a 0x47c-byte, 7-axis queue buffer, and the per-axis, per-tick step `FUN_8013dfb0` blends across segment boundaries (q12, Bezier-style with midpoint blended targets/durations) so PSX velocity decays smoothly instead of cliffing at a boundary. Non-fusion `{19}` Cameras take the separate `FUN_80146110` task body (1.6 fixed-point polynomial; linear at mode 0x00). Decoded against the scenario-1 chapel swoop (2026-06-26, `orbonne_prayer_cinematic.sstate`); the user-visible "speeds way up" near the end is PSX-faithful (peak per-segment velocity at seg 4), and Godot's `CameraChainSpline` port (on by default) matches the PSX probe within ±2 opcode units per tick where the old strict-linear lerp had a 6× velocity cliff at seg-5→6.

## Points

- **The 0x47c fusion-queue buffer built by `FUN_8013db9c` is 7 axes × 0xA4 bytes: seven 0x10-byte slot entries at 0x00..0x70 per axis (initial + 6 segments), each `cumulative_time @ +0`, `target @ +4`; per-axis state `total_segs @ 0x80`, `cur_seg @ 0x84`, derived blending state at 0x88..0x9c; position targets stored as opcode units (scratch >> 10), rotation targets as raw scratch.** — `[D·R] 2/3`
  - D: 0x47c buffer dump (`probe_chain_spline_buffer.bin`, 2026-06-26 — Q-B closed)
  - R: `godot-learning/src/scenarios/CameraChainSpline.gd` `AxisBuf` mirrors the on-disc layout (segments 0x00..0x70, state 0x80..0xa0) + `godot-learning/tests/ScenarioChapelChainSplineTest.gd` (per-tick output within ±2 opcode units of the PSX trace over 514 ticks)
  - src: `research/working_documents/scenario_1_captures/cinematic_camera_motion_decode.md`
- **The fusion chain's per-tick interpolation is `FUN_8013dfb0(axis_buf, axis_idx)`, called from `FUN_8013db9c`'s post-build loop at `battle:8013debc..8013df3c` — one call per axis per tick, result stored to `scratch[axis]`; non-fusion `{19}` Cameras instead run the `FUN_80146110` task body (probe-confirmed: 1 hit in 30 s of chapel, mode=0x00 linear, time=1 snap) — a 1.6 fixed-point polynomial evaluator: `s2 = (target − current) << 6`, polynomial `s2×s8 + (s2×t0)×2 + (s6²×coeff)` via `FUN_8014ccb8`, summed with `current << 6`, rounded to nearest 0x40, then `>> 6`.** — `[S·D·R] 3/3`
  - S: `battle:8013debc..8013df3c` (post-build loop), `battle:80146110` body disasm at `0x80146354..0x8014657c`, `FUN_8014ccb8` (`battle_disassembly.txt`)
  - D: `probe_camera_chain_writes.jsonl` (2026-06-26) — `FUN_80146110` fires once (non-fusion snap); observed Δ-x/tick at swoop end decays smoothly through −4434, −2687, −1904, −1402 (vsyncs 1359/1374/1389/1404)
  - R: `godot-learning/src/scenarios/CameraChainSpline.gd` (port of `FUN_8013dfb0`; on by default via `ScenarioCameraDirector.use_camera_chain_spline`) + `godot-learning/tests/ScenarioChapelChainSplineTest.gd` (±2 opcode units/tick vs PSX trace, velocity windows at each segment boundary)
  - src: `research/working_documents/scenario_1_captures/cinematic_camera_motion_decode.md`
- **`FUN_8013dfb0` does cross-boundary spline blending: blended target = `target_cur + (target_next − target_cur) / 2` (midpoint into next segment), falling back to `target_cur` when `cur_seg == 0` or `total_segs < 4`; blended duration = `dur_cur + dur_next/2` (first seg), `(dur_cur + dur_next)/2` (mid), `dur_next + dur_cur/2` (second-to-last), `dur_cur + dur_next` (short chains, total < 4); blended target ×4096 stored at axis-state offset 0x94 (1.12 fixed-point).** — `[S·D·R] 3/3`
  - S: `battle:8013e0b8..8013e0f8` (blended target), `battle:8013e030..8013e08c` (blended duration) (`battle_disassembly.txt`)
  - D: `probe_camera_chain_writes.jsonl` (2026-06-26) — the cross-boundary Δ-x decay at swoop end (vsyncs 1359..1404) is consistent with blending rather than sharp per-segment re-targeting
  - R: `godot-learning/src/scenarios/CameraChainSpline.gd` (Bezier through the blended control points; `AxisBuf` v94/blended_dur state) + `godot-learning/tests/ScenarioChapelChainSplineTest.gd`
  - src: `research/working_documents/scenario_1_captures/cinematic_camera_motion_decode.md`
- **The chapel fusion bracket is 6 `{19}` Cameras at chunk offsets 122/139/156/173/190/207 with `Time = {260, 80, 68, 56, 32, 48}` ticks and ΔX = {−512, −80, −192, −256, −128, −32} opcode units; the immediate `{19}` snap at chunk offset 56 (X=2040, Angle=750, Zoom=8192, Time=1) sets the first segment's start, and a non-fusion `Camera Time=4` at PC=48 is the bracket's cleanup op.** — `[D·R] 2/3`
  - D: `godot-learning/assets/scenarios/scenario_1_chunk.json` chunk decode (`tools/disasm_event.py`, 2026-06-26)
  - R: `godot-learning/tests/ScenarioChapelChainTraceTest.gd` drives a real `ScenarioVM` through the same sequence (PC=56 snap, fusion start, the 6 cameras at 122/139/156/173/190/207, fusion end)
  - src: `research/working_documents/scenario_1_captures/cinematic_camera_motion_decode.md`
- **The chapel swoop's "starts at an OK speed, then suddenly speeds way up" is PSX-faithful: per-segment velocity peaks at seg 4 (ΔX −256 in 56 ticks ≈ 2.5× seg-1's), the probe confirms peak Δx = −4973/tick scratch at vsync ~1329 against the expected −4681, and both PSX and Godot show the 1.6× per-tick velocity jump at the seg-3→4 boundary.** — `[D·R] 2/3`
  - D: `probe_camera_chain_writes.jsonl` (2026-06-26) — peak Δx = −4973 at vs=1329 (~7.4 s in); `diff_godot_vs_psx_camera.py` diff table (identical 1.6× seg-3→4 jump in both implementations)
  - R: `godot-learning/tests/ScenarioChapelChainTraceTest.gd` (Godot matches the expected linear per-segment Δx/tick exactly: seg 1 −1.969, seg 4 −4.571 opcode units/tick)
  - src: `research/working_documents/scenario_1_captures/cinematic_camera_motion_decode.md`
- **The PSX chain undershoots its final target: last motion at vsync 1405 leaves scratch X=895574, not the seg-6 target 860160 (X=898 vs target 872 at end of seg-5) — consistent with a floor-truncating step accumulator losing ~½ a step per recomputation; the queue ticks slightly shorter than Σ Time_i = 544.** — `[D] 1/3`
  - D: `probe_camera_chain_writes.jsonl` (2026-06-26) — last motion vsync 1405, final scratch X=895574
  - R: none — final-target undershoot not asserted in any godot-learning test (probed `godot-learning/src/`, `godot-learning/tests/`)
  - src: `research/working_documents/scenario_1_captures/cinematic_camera_motion_decode.md`
- **Godot's original `_advance_camera_lerp` was strict per-segment linear (`lerp(start, target, done)`), exact on per-segment Δx/tick but carrying a 6× velocity cliff at the seg-5→seg-6 boundary (−4096/tick → −682/tick) that PSX's cross-segment blend smooths away; the fix is the `FUN_8013dfb0` port `CameraChainSpline` (default ON), with `camera_ease_mode` LINEAR retained as the verified non-fusion curve.** — `[D·R] 2/3`
  - D: `probe_camera_chain_writes.jsonl` (2026-06-26) — PSX Δx/tick decays smoothly through −4434 → −2687 → −1904 → −1402 at swoop end, no 6× cliff
  - R: `godot-learning/src/scenarios/CameraChainSpline.gd` + `godot-learning/src/scenarios/ScenarioCameraDirector.gd` (`use_camera_chain_spline = true`) — validated by `godot-learning/tests/ScenarioChapelChainSplineTest.gd`
  - src: `research/working_documents/scenario_1_captures/cinematic_camera_motion_decode.md`
- **The 0x1D/0x1E camera-fusion bracket dispatches atomically in the main executor: the 0x1d case (gate `LAB_80144c50`, body `0x80144c58`) allocates a fiber slot (`FUN_80149bec(0x10)`), spawns the queue-builder task (`event_task_spawn`, callback `0x8013db9c`), writes the current bytecode pointer into the slot, then retargets the VM cursor by forward-scanning (`FUN_80149ebc(s8, 0x1e)` over size table `0x8014d170`) so the loop leaves the entire 0x1d..0x1e span in one pass — the six inner `{19}`s and the 0x1e are never individually dispatched; the 0x1e case itself is a no-op gate (`beq @ 0x80144c90` to `evt_advance_loop`); the queued camera lerps drain on a background fiber while the VM races ahead to post-bracket opcodes.** — `[S·D·R] 3/3`
  - S: main dispatcher `FUN_80143bd8` / loop head `LAB_80143d0c` (opcode lbu `0x80143d30`); compare-chain bodies per `battle_disassembly.txt`: 0x19 @ `LAB_80144c2c` (registers callback `0x80146110`), 0x1d @ `LAB_80144c50` (body `0x80144c58` to `LAB_80145944`: `FUN_80149bec(0x10)` + `event_task_spawn` cb `0x8013db9c` + slot write + `FUN_80149ebc` scan with delay-slot `_move s8, v0`), 0x1e no-op @ `0x80144c90`; scanner `FUN_80149ebc` + size table `0x8014d170` ([0x1d]=0, [0x1e]=0, [0x19]=0x10); queue builder `FUN_8013db9c` (Ghidra annotation `camera_fusion_end_queue_build`: scans from the stored pointer counting 0x19s to the matching 0x1e, pre-computes 7-axis segment tables, drains via `0x8013debc..0x8013df3c`). Note: the doc's Q11 static list attributes these bodies to 0x1d@LAB_80144c2c / 0x1e@LAB_80144c50 — off by one case slot (MIPS branch-delay-slot misread); the mapping above is the verified one.
  - D: `probe_camera_vs_dialog_timing.py` (`orbonne_prayer_pre_scenario_load.sstate`, 2026-06-26): vsync 41 chunk PC=121 (0x1d) @ cycle 3628178805 to PC=225 @ 3628179524 (719 cycles, one vsync) to PC=247 (`0xF1` Wait 180) @ 3628180517; `0x8013db9c` fiber fires @ 3628183054 (same vsync, later); Display Message (chunk PC=256) fires at vsync 225–226 with the camera still mid-swoop
  - R: `godot-learning/src/scenarios/ScenarioCameraDirector.gd` `_op_camera_fusion_start` / `_op_camera_fusion_end` (queue mode `_in_camera_fusion`, `_camera_queue`), `_apply_camera_pose_no_block` chain handoff (overshoot carry + target-as-start), `_start_next_queued_camera` — validated by `godot-learning/tests/ScenarioCameraFusionTest.gd`
  - src: `research/working_documents/scenario_1_captures/display_message_overlay_decode.md`

## Notes

(empty — user territory)

## Related

- [[Scenario Camera Opcodes]]
- [[Scenario Camera Framing]]
- [[Event Opcode Catalog]]
