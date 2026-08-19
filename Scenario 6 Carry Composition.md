# Scenario 6 Carry Composition

At the scn6 abduction "Wait!!!" carry beat (Dialog 145 / Message 14, PC 316/320) — Delita carrying Ovelia beside the resting chocobo — live PSX dynamic analysis (hide-one-unit experiment + unit-struct reads on a savestate) settled the composition question: the tableau is NOT a managed mount ensemble or per-piece OTZ interleave, but three ordinary sprites at three flat OT depths, drawn back→front chocobo (bucket 157) → Ovelia (156) → Delita (147). The "chocobo tail poking through in front of Delita" is an occlusion illusion: the tail is genuinely behind Delita, but Delita's authored carry-pose frame is legless (transparent below the waist), so the chocobo behind shows through that empty region. The Godot side correctly models each unit as one flat-depth billboard, but its unit-vs-unit depth order on this beat is inverted (chocobo frontmost, PSX backmost), making the real task a plain depth-order fix (pending as of 2026-07-07).

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

## Notes

(empty — user territory)

## Related

- [[Unit Sprite Render Pipeline]]
- [[Unit Sprite Object Struct]]
- [[Ordering Table & AddPrim]]
- [[EVTCHR Frame Resolution]]
- [[Scenario Beat Capture]]
