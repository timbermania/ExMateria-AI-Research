# Unit Anim Opcode

Event instruction `{11}` Unit Anim plays an animation on one unit or on a team (selected by the Multi Targeting byte): the half-word Animation ID either names a unit SEQ animation (x0000–x003B idle/locomotion/state, x003C–x007D weapon attacks that require a clear EVTCHR block) or an EVTCHR atlas animation (Block 1 x01F4–x0233, Block 2 x0258–x0297, slot-specific). The four ID buckets map to two live runtime paths — SEQ animations are driven by the combat per-unit ticker (`FUN_8017a290` → `FUN_801810a0`) and EVTCHR animations by the cinematic dispatcher (`FUN_80084818` / `update_and_animate_unit_wep_eff`, `FUN_80085c0c`) — and the Orbonne chapel cinematic runs entirely on the EVTCHR path (ticker 0 fires, dispatcher ~6800 fires over 30 s). The spec is the ffhacktics wiki `Event_Instruction_11` article, pasted 2026-06-27 during the chapel sprite-pipeline investigation; the opcode is implemented in the Godot reimplementation (`_op_unit_anim` in `ScenarioVM`), per the master-catalog snapshot (2026-06-30) and verified in current `godot-learning/src/scenarios/ScenarioVM.gd`. The unit's current animation ID is readable in RAM as a uint16 at `unit+0x1DC`, and PSX and Godot (`Unit.current_anim_id`) agree exactly at the same parked beat PC (verified at scn6 PC210: Delita 520, Ovelia 510).

## Points

- **Event instruction `{11}` Unit Anim's body is, after the opcode byte: Unknown (1B, "mostly seems to be x00 and sometimes x01"), Affected Units (1B), Multi Targeting (1B), Animation ID (half-word) — where the Animation ID loads an animation sequence from the corresponding SEQ file for a unit, or an EVTCHR animation.** — `[ ] 0/3`
  - src: `research/wiki_articles/event_instruction_11_unit_anim.md`
- **The Multi Targeting byte selects how Affected Units is interpreted — x00: one target, Affected Units is the ID of the unit specified in the ENTD; x01: team codes (x00 Player's Team (blue), x01 Player's Team (blue) unless incapacitated (dead, critical, etc.), x02 All Enemy Teams (red/green/lightblue), x03 All Enemy Teams unless incapacitated, x04 All Teams); x02: target all teams.** — `[ ] 0/3`
  - src: `research/wiki_articles/event_instruction_11_unit_anim.md`
- **Animation IDs x0000–x003B are unit SEQ animations (x0000/x0032/x003B Vanish, x0001/x0002 Standing, x0003 Walk, x000D Run, x0012–x0014 Fly-to, x0015 Spinning, x0016 Critical/Bow, x0017 Defend, x0018 Dodge, x0019 In Pain (loop), x001A Dead, x001C Level Up (loop), x001D Job Level Up (loop), x0021/x0022 Charge, x0025 Confusion, x0034 Character Death, x0035/x0036 Bow, x0064 Flip Switch On Ground), and a "Nothing" ID (x003A) is a buffer that pauses on whatever the unit's last animation frame was.** — `[ ] 0/3`
  - src: `research/wiki_articles/event_instruction_11_unit_anim.md`
- **Animation IDs x003C–x007D are SEQ weapon/attack animations (unarmed attacks x003D–x003F, weapon attacks x0040–x0042, throws x0043–x004C, spear x004D–x004F, gunshot x0050–x0052, bow x0053, book strike x0056, weapon guards x0058–x005A, shield recoil x005B–x0063, Lancer Jump x0075/x0076, female weapon x0077–x0079, Goblin Punch x007D) and every one of them requires an EVTCHR block to be clear — with two EVTCHR blocks Loaded and Saved you cannot use a weapon attack animation; the wiki directs `LoadEVTCHRClear`/`SaveEVTCHRClear`.** — `[ ] 0/3`
  - src: `research/wiki_articles/event_instruction_11_unit_anim.md`
- **EVTCHR animations occupy IDs x01F4–x0233 (Block 1, 64 slots, red) and x0258–x0297 (Block 2, 64 slots, blue) and are slot-specific — each animation index maps to a specific grid cell of the EVTCHR atlas, with `EVTCHR_LOCATIONS.png` as the canonical visual slot map for both blocks.** — `[ ] 0/3`
  - src: `research/wiki_articles/event_instruction_11_unit_anim.md`
- **The four Unit Anim ID buckets map to two live runtime paths: SEQ animations (x0000–x003B, x003C–x007D) go through the combat per-unit ticker (`FUN_8017a290` → `FUN_801810a0`, the sprite pipeline's "Layer 4"), while EVTCHR animations (x01F4–x0233, x0258–x0297) go through the cinematic dispatcher (`FUN_80084818` / `update_and_animate_unit_wep_eff` = `FUN_80085c0c`).** — `[S] 1/3`
  - S: `FUN_8017a290` → `FUN_801810a0` (per `SPRITE_PIPELINE_INVESTIGATION.md` Layer 4); `FUN_80084818` / `update_and_animate_unit_wep_eff` = `FUN_80085c0c` (per `reference_cinematic_seq_table_2026_06_25`)
  - src: `research/wiki_articles/event_instruction_11_unit_anim.md`
- **During the Orbonne chapel cinematic the combat ticker `FUN_8017a290` fires 0 times while the cinematic dispatcher `FUN_80085c0c` fires ~6800× over 30 s — the chapel renderer is the EVTCHR path.** — `[S·D] 2/3`
  - S: `FUN_8017a290`, `FUN_80085c0c`
  - D: probe `reference_cinematic_seq_table_2026_06_25` (2026-06-25)
  - D: probe `probe_layer4_render.lua` 30 s breakpoint hit table, `orbonne_prayer_mid_dialog.sstate` (2026-06-27): FUN_8017a290=0, FUN_80085c0c=6802, FUN_80084818=60
  - src: `research/wiki_articles/event_instruction_11_unit_anim.md`
- **The `{11}` Unit Anim handler is `0x80149398` (`evt0x11_unit_anim_handler`), which reads the chunk and dispatches on the Animation ID.** — `[S] 1/3`
  - S: `0x80149398` (`evt0x11_unit_anim_handler`) (`battle_disassembly.txt`, master catalog `EventCommands.xml` row {11})
  - src: `research/wiki_articles/event_instructions.md`
- **A unit's current animation ID is stored as a uint16 at `unit+0x1DC` (scn6: Delita unit record base `0x800B8848`, Ovelia `0x800BA608`), and read at a parked beat the PSX value equals Godot's `Unit.current_anim_id` at the same PC — verified at scn6 PC210: Delita 520, Ovelia 510, exact match.** — `[S·D·R] 3/3`
  - S: anim ID at `uint16 @ unit+0x1DC`, unit record bases `0x800B8848`/`0x800BA608` (doc)
  - D: anim-ID read at the scn6 PC210 park — exact PSX/Godot match, Delita 520 / Ovelia 510 (2026-07-07)
  - R: `godot-learning/src/units/Unit.gd` (`current_anim_id`) (no named test)
  - src: `research/working_documents/CAPTURE_SCENARIO_BEAT_HOWTO.md`

## Notes

(empty — user territory)

## Related

- [[ENTD Unit Deployment Table]]
- [[Event Opcode Catalog]]
- [[Scenario Beat Capture]]
- [[EVTCHR Script VM]]
- [[Cinematic Sprite Renderer]]
