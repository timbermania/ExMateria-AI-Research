# Scenario 6 Carry Composition

At the scn6 abduction "Wait!!!" carry beat (Dialog 145 / Message 14, PC 316/320) — Delita carrying Ovelia beside the resting chocobo — live PSX dynamic analysis (hide-one-unit experiment + unit-struct reads on a savestate) settled the composition question: the tableau is NOT a managed mount ensemble or per-piece OTZ interleave, but three ordinary sprites at three flat OT depths, drawn back→front chocobo (bucket 157) → Ovelia (156) → Delita (147). The "chocobo tail poking through in front of Delita" is an occlusion illusion: the tail is genuinely behind Delita, but Delita's authored carry-pose frame is legless (transparent below the waist), so the chocobo behind shows through that empty region. The Godot side correctly models each unit as one flat-depth billboard, but its unit-vs-unit depth order on this beat is inverted (chocobo frontmost, PSX backmost), making the real task a plain depth-order fix (pending as of 2026-07-07). The carry cinematic is scripted as `Load EVTCHR {Block:0, Slot:1}` + `{11}` Unit Anims 500–527 (0x1F4–0x20F) on units 5/12 — the PSX mid-cinematic band that Godot's chapel-only 0x258 gate froze until the 2026-07-05 band-routing fix; the settled carry beat (PC≈219) now measures frame-identical between engines (Delita 0xF2, Ovelia 0xE5).

## Points

- **At the scn6 "Wait!!!" carry beat (Dialog 145 / Message 14, PC 316/320) the tableau is three ordinary sprites at three flat OT depths — Delita bucket 147 (front) / Ovelia 156 / chocobo 157 (back) — painted back→front, and NOT a mount ensemble or per-piece OTZ interleave: `mount_mode == 0` in all 16 slots (the dual-pass mount-pairing path is never entered), `layer_priority == 0` on all three (default layer table; opcode 0xE2 set nothing), and no chocobo piece carries the 0x80 split flag (no per-piece `sprite_scale_z` bucket offset). The "chocobo tail in front of Delita" look is an occlusion illusion: the tail is genuinely behind Delita (157 vs 147), but Delita's carry-pose EVTCHR frame is legless (transparent below the waist), so the chocobo directly behind shows through that empty region; smaller `+0x128` bucket = nearer/front.** — `[S·D·R] 3/3`
  - S: `DAT_80094548` default layer table; `FUN_80084214` (0x80 split-flag setter; no chocobo piece has it); unit fields `+0x14` (layer_priority) / `+0x128` (OT bucket) / `+0x130` (mount_mode) / `+0x131` (mount id); mount dual-pass path in `project-assets/fft-rom/battle_decompilation.c` (per `PSX_UNIT_SPRITE_RENDERING.md` §9)
  - D: hide-one-unit experiment (zeroing sprite-buffer `piece_count` `[unit+0x204] → buf+0x03` per slot, ~3-frame re-render) + full 16-slot unit-struct dump on `reference-assets/scenario6_carry_agrias_wait_msgid14.sstate`, pcsx-redux port 8080; frames archived in `scenario6_carry_images/` (2026-07-05)
  - R: godot-learning models each scenario unit with one flat OT bucket from a single representative point (`src/core/DepthMode.gd` `Mode.UNIT` + `assets/shaders/psx_ot_depth.gdshaderinc`, ADR-0009) — matches PSX's one-bucket-per-unit here; no named test
  - src: `research/working_documents/SCENARIO6_CARRY_COMPOSITION_DEPTH.md`
- **On the same beat, Godot's unit-vs-unit depth order is the INVERSE of PSX: measured Godot OT depths are chocobo −0.6195 (nearest), Ovelia −0.6246, Delita −0.6273 (farthest) — Godot draws the chocobo in front of both riders, where PSX has it backmost. The PSX target order (front→back) is Delita → Ovelia → chocobo, and the correction is a pure depth-order fix: no per-piece tail interleave exists to reproduce, and the terrain/chocobo showing through Delita's lower body is faithful authored transparency, not a bug to "fix".** — `[D·R] 2/3`
  - D: Godot OT depths measured by the PC=316/320 probe (handoff measurements recorded in this doc, 2026-07-05); PSX buckets from the same-day unit-struct dump
  - R: `godot-learning/tools/probe_scenario6_depth.gd` (boots scn6, rewinds to PC 320, measures the chocobo/Ovelia/Delita trio's Godot OT depth the way the unit shader does) + unit depth model `src/core/DepthMode.gd` `Mode.UNIT` — no named test; order correction pending as of 2026-07-07
  - src: `research/working_documents/SCENARIO6_CARRY_COMPOSITION_DEPTH.md`
- **The scn6 carry cinematic is scripted as chunk instr 4 `Load EVTCHR {Block: 0, Slot: 1}` followed by `{11}` Unit Anim ops with Animation 500–527 (`0x1F4`–`0x20F`): Ovelia (unit 12) cycles 500–513 and Delita (unit 5) cycles 513–527 — every carry anim squarely inside the PSX mid-cinematic band `[0x1F4, 0x258)`, never the high band the Orbonne chapel uses.** — `[S·D·R] 3/3`
  - S: `assets/scenarios/chunks/scenario_006_chunk.json` (instr 4 `Load EVTCHR {Block:0, Slot:1}`; instr 105 `Unit Anim {Units:12, Animation:500}`; instr 147 `Unit Anim {Units:5, Animation:516}`; every Animation on units 5/12 in the cinematic falls in [500, 527])
  - D: live scn6 poll (2026-07-05): mid-table slots 0–27 (anims 500–527) populated, `slot = anim − 0x1F4` confirmed live
  - R: `godot-learning/tests/ScenarioCinematicBandRoutingTest.gd` pins the 28 live-verified scn6 slots (500→0 … 527→27) and the scn6 Slot-1 identity
  - src: `research/working_documents/SCENARIO6_CARRY_POSE_EVTCHR_RENDER.md`
- **At the settled carry beat (PC≈219, D519/O511) Godot and PSX render the same frame byte for both leads — Delita's segment-1 frame 0xF2 (rect `(128,160)`, CLUT `0x78C5`) and Ovelia's 0xE5 (rect `(160,80)`, CLUT `0x78CC`) — with matching palette and silhouette orientation (no H-flip: the bent arm is on the same side in both), and with `cam.size 9.147` both engines map 1 tile to 21.68 px, so the carry sprite measures faithful on every axis tested at that beat.** — `[D·R] 2/3`
  - D: frame-identity comparator (2026-07-07): PSX piece[0] UV read live from the assembled buffer, Godot walker rect, and the frozen savestate VRAM framebuffer (`GET /api/v1/gpu/vram/raw`, `(0,0) 256×240 RGB555`) — Delita pixel-identical to seg1 tile 0xF2 recolored with CLUT 0x78C5
  - R: `godot-learning/src/scenarios/ScenarioVM.gd` (`CinematicWalkState.last_fb` frame-byte observability) + `godot-learning/tools/probe_carry_pieces.gd` (frame-byte + PSX piece comparator) + `godot-learning/tools/capture_carry_ab.gd` (`BRIGHT=1` full-ambient, `CAM_SIZE=9.147` PSX-matched)
  - src: `research/working_documents/SCENARIO6_CARRY_POSE_EVTCHR_RENDER.md`

## Notes

(empty — user territory)

## Related

- [[Unit Sprite Render Pipeline]]
- [[Unit Sprite Object Struct]]
- [[Ordering Table & AddPrim]]
- [[EVTCHR Frame Resolution]]
- [[Scenario Beat Capture]]
- [[Unit Anim Opcode]]
