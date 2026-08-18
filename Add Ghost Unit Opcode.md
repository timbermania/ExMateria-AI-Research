# Add Ghost Unit Opcode

Event opcode `0x47` "Add Ghost Unit" (8-byte body `xSP, x00, xID, X, Y, xEL, xFD, xDR`) spawns a sprite-only "ghost" actor — no ENTD roster record, no stats — through the same unit-add queue as `0x45` Add-Unit, the ENTD-unit-ptr queue field being `0` the only difference. Fully grounded: the dispatch case in the large event executor `FUN_80143bd8`, the authoritative operand decode (`xDR` Draw inverted from the first draft; the ghost's control unit-id = `xID + 0x64`), `xSP` resolved as a standard `sprite_set` id through the shared drain, and placement = operand tile (the ghost lives in the sprite-object list, not the combat roster). Implemented in godot-learning (bound handler + spawn path, tested); in scenario 6 it draws three held, off-frame generic escort sprites (`0x61/0x62/0x63`) for the Orbonne abduction tableau.

## Points

- **`0x47` Add Ghost Unit has an 8-byte body `xSP, x00, xID, X, Y, xEL, xFD, xDR` (9 bytes total), and the interpreter advances the PC by size-table[`0x47`]+1 from the opcode size table at `0x8014D170` (`[0x47]=8`, `[0x45]=3`, `[0x44]=2`, `[0x46]=2`, `[0x48]=0`).** — `[S·D] 2/3`
  - S: size table `0x8014D170`, case body `code_r0x801456ac` (`battle_disassembly.txt`)
  - D: scenario 6 live RAM read of the size table + raw bytecode dump `47 61 00 …` @ `0x8004AFFE` (2026-07-04)
  - src: `research/working_documents/ADD_GHOST_UNIT_OPCODE_47.md`
- **`0x47` is dispatched by the large event executor `FUN_80143bd8` @ `0x80143BD8` (case label `code_r0x801456ac` @ `0x801456AC`), not by the small top-level interpreter `FUN_8013e904` which has no `0x47` case; a live dispatch fire with `ra=0x80143D3C` confirms the large executor.** — `[S·D] 2/3`
  - S: `FUN_80143bd8`, `code_r0x801456ac`, `FUN_8013e904` (`battle_disassembly.txt`)
  - D: scenario 6 non-pausing dispatch BP @ `0x801456AC` fire, `ra=0x80143D3C` (2026-07-04)
  - src: `research/working_documents/ADD_GHOST_UNIT_OPCODE_47.md`
- **A ghost unit is an EVTCHR-sprite-only actor with no ENTD roster record: `0x47` and `0x45` Add-Unit (`evt_add_unit_sister_handler` @ `0x8008D05C`) both enqueue through `evtchr_queue_enqueue` @ `0x8008E540` — a 0x14-byte entry at `0x80049C1C` (X, Y, Z, Facing i16, Spritesheet i16, reserved, Slot i16, ENTD-unit-ptr u32, Draw u32, queue caps at 16 pending) — and the only semantic difference is the `+0x0C` ENTD-unit-ptr field: `0` for a ghost (drain builds a sprite-only actor with no roster/stat backing), the real unit struct for a normal unit.** — `[S·D] 2/3`
  - S: `evtchr_queue_enqueue` `0x8008E540`, queue base `0x80049C1C`, `evt_add_unit_sister_handler` `0x8008D05C` (`battle_disassembly.txt`)
  - D: scenario 6 spawn BP @ `0x8008CF78` live args — Spritesheet 97/98/99, slot 5/6/7, Draw 1 (2026-07-04)
  - src: `research/working_documents/ADD_GHOST_UNIT_OPCODE_47.md`
- **The `0x47` handler allocates the first free slot by scanning `FUN_8008cbb4` over slots `0..0x14` and records it in the tracking table `0x80165FE8 + xID*2`; `xID` is not the slot, and the ghost's event-facing control unit-id is `xID + 0x64` (scenario 6: `0/1/2 → 0x64/0x65/0x66`), which is how later ops address the ghost.** — `[S·D] 2/3`
  - S: `FUN_8008cbb4`, table `0x80165FE8` (`battle_disassembly.txt`, case body `code_r0x801456ac`)
  - D: scenario 6 capture — live slots 5/6/7, table `[0]=5 [1]=6 [2]=7` (2026-07-04)
  - src: `research/working_documents/ADD_GHOST_UNIT_OPCODE_47.md`
- **Spawning is gated on the slot having no sprite yet — `FUN_8007a6e4(slot)==0` inside `FUN_8008cf78` @ `0x8008CF78` — so re-running `0x47` on an already-spawned ghost is a no-op (idempotency gate).** — `[S] 1/3`
  - S: `FUN_8008cf78` `0x8008CF78`, `FUN_8007a6e4` `0x8007A6E4` (`battle_disassembly.txt`)
  - src: `research/working_documents/ADD_GHOST_UNIT_OPCODE_47.md`
- **The `xDR` (Draw) operand is `0x00` = draw immediately, `0x01` = hold in memory and reveal later via `{44} Draw Unit` (inverted from the first draft's guess); all three scenario-6 ghosts are added held (`xDR=1`) around msgids 15/16 and become drawn at the msgid-17 "…Delita??" beat.** — `[D] 1/3`
  - D: scenario 6 raw bytecode (xDR=1 ×3, 2026-07-04) + PSX tableau captures `/tmp/gu_tableau.png` / `2026-07-05_gu_tableau.png` showing the ghosts drawn at msgid 17 (2026-07-05); semantics per the authoritative community/FFHacktics `{47}` reference quoted in the doc §3.1
  - src: `research/working_documents/ADD_GHOST_UNIT_OPCODE_47.md`
- **`xSP` is a standard unit-sprite / `sprite_set` id (not an EVTCHR segment index): `0x47` and `0x45` both resolve queue `+0x06` in the shared drain `FUN_80088904` @ `0x80088904` (clamping `> 0x9E → 1`) against the shared 8-byte-stride CD file-descriptor table `&DAT_80094cd8` (~0x9F entries, `evtchr_unit_clut_writer` @ `0x80087A28` uploads), so `0x61/0x62/0x63` are the SPR files Female Squire / Male Chemist / Female Chemist.** — `[S·R] 2/3`
  - S: `FUN_80088904`, `&DAT_80094cd8`, `evtchr_unit_clut_writer` `0x80087A28` (`battle_disassembly.txt`, static shared-resolver trace §5)
  - R: `godot-learning` `ScenarioPlayerScene._spawn_ghost_actor` (body_sprite_id = xSP through the normal SPR pipeline, not `evtchr_frames.json`), validated by `ScenarioApplyTest`
  - src: `research/working_documents/ADD_GHOST_UNIT_OPCODE_47.md`
- **A ghost is placed at its operand tile X/Y (scenario 6 `0,0` = map corner, off the tableau camera); there is no cinematic-anchor table — the ghost's free-slot scan walks the sprite-object list from head `DAT_80098a54` (the "slots" 5/6/7 are sprite-object ids), not the combat roster `0x801908CC + slot*0x1C0`, and a static enumeration of every roster tile-byte `+0x47/+0x48` writer found nothing on the ghost path writing them.** — `[S·R] 2/3`
  - S: sprite-object list head `DAT_80098a54`, roster base `0x801908CC` (stride 0x1C0), `0x80165FE8` holds only slot indices (`battle_disassembly.txt`, writer enumeration §4.5a)
  - R: `godot-learning` `ScenarioApply.add_ghost_unit` (placement = operand tile), validated by `ScenarioApplyTest` + headful scn6 probe (2026-07-05, ghosts register at the map corner)
  - src: `research/working_documents/ADD_GHOST_UNIT_OPCODE_47.md`
- **Godot binds `0x47` as `_op_add_ghost_unit` → `ScenarioApply.add_ghost_unit` → `ScenarioWorld.spawn_ghost_unit` → host `ScenarioPlayerScene._spawn_ghost_actor`: ghosts register under control-id `xID+0x64` (`0x64/0x65/0x66`), sprite_set = `xSP` via the normal SPR pipeline, placed at the operand tile, with `xDR`-gated visibility (held ghosts stay hidden — the PSX reveal trigger is untraced) and the PSX idempotency gate as a `units_by_id.has(control_id)` no-op; `0x47` is marked `verified:true`.** — `[R] 1/3`
  - R: `godot-learning/src/scenarios` `ScenarioVM.gd` / `ScenarioApply.gd` / `ScenarioWorld.gd` / `ScenarioPlayerScene.gd`, validated by `ScenarioApplyTest` (4 ghost cases) + `ScenarioEventInstructionCoverageTest`; headful scn6 probe confirms registration at the map corner (2026-07-05)
  - src: `research/working_documents/ADD_GHOST_UNIT_OPCODE_47.md`
- **In Godot, scenario 6's msgid-17 back-of-castle tableau closely matches the PSX reference — unit 5 (Delita) spawns, animates through the struggle, and is Removed at the correct beat — so `0x47` is faithful polish (three off-frame escort sprites), not a rescue of an empty scene; the earlier "empty courtyard" observation was the opaque `FadeLayer/FadeRect` overlay captured in `-s` SceneTree mode, not a black scene.** — `[R] 1/3`
  - R: `godot-learning/tools/probe_scenario6_roster.gd` headful auto-advance probe, `/tmp/s6_tableau_nodlg.png` vs PSX `research/working_documents/scenario_6_captures/2026-07-05_gu_tableau.png` (2026-07-05)
  - src: `research/working_documents/ADD_GHOST_UNIT_OPCODE_47.md`
- **In scenario 6 the "Delita" of the abduction tableau is normal unit 5 (ENTD 387 sprite_set `0x05`), animated via Warp/Sprite-Move and Removed at chunk offset 2361 — not a `0x47` ghost; the animated cast are ordinary units (5/12/52/139/2), while the three ghosts are generic escort sprites `0x61/0x62/0x63`.** — `[D] 1/3`
  - D: scenario 6 chunk JSON `godot-learning/assets/scenarios/chunks/scenario_006_chunk.json` + PSX/Godot probes tracking unit 5 (2026-07-05)
  - src: `research/working_documents/ADD_GHOST_UNIT_OPCODE_47.md`
- **The chunk JSON stores opcode ids in decimal — `opcode 71` = `0x47` Add Ghost Unit — and the separate `opcode 113` (`0x71`) is a different, still-Unknown op with its own dispatcher case → `FUN_8007a7b8`.** — `[S] 1/3`
  - S: dispatcher `0x71` case → `FUN_8007a7b8` (`battle_disassembly.txt`); chunk JSON decimal encoding
  - src: `research/working_documents/ADD_GHOST_UNIT_OPCODE_47.md`

## Notes

(empty — user territory)

## Related

- [[Event Opcode Catalog]]
- [[Sprite Set Resolution]]
- [[ENTD Unit Deployment Table]]
- [[Wait Add Unit Opcode]]
