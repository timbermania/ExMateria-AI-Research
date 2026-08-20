# EVTCHR Frame Resolution

The frame-resolution half of the EVTCHR cinematic problem: which pixels of a resident EVTCHR block a `{11}` Unit Anim draws for a unit, resolved as a parse-time transform instead of at runtime. As of the 2026-07-09 scn6 carry-beat validation the whole chain is confirmed dynamic+static+pixel: `{58}` `Slot` picks the file block (entry) and `Block` picks a 2-entry VRAM rect, the scn6 carry is file-block 1 at VRAM (256,0) = segment 1; a per-unit `(unit+0x05, +0x06, +0x07)` tuple locates the sprite descriptor; `sprite_subframe_assemble`'s static tile-UV formula matches live PSX primitives and the Godot parser byte-for-byte; and the `+7` frame-index rewrite (disc `0xCB..0xF2` to runtime `0xD2..0xF9`) is sound. The one confirmed defect is Godot's baked pixel page sitting 1 row too high vs PSX VRAM (`dx=0, dy=+1`), suspected to be the `0x0A00` vs `0x980` pixel-page origin. The segment-to-character half lives in [[EVTCHR Character Attribution]]; the load/save/VRAM opcode mechanics in [[EVTCHR Script VM]].

## Points

- **The scn6 carry loads exactly one EVTCHR sheet: its only `Load EVTCHR` is chunk instr 4 `{Block:0, Slot:1}` → file-block 1 at VRAM `(256,0)`, unchanged across buffer swaps, so the correct Godot entry is segment 1 (the earlier "SEG 0 → Ramza" render was a forced-wrong-entry artifact, not a frame bug).** — `[S·D·R] 3/3`
  - S: opcode decode per `research/scenario_1_captures/evtchr_load_save_decode.md` (`Open_EVTCHR` `0x8013C7C4`)
  - D: live VRAM read at the scn6 "letgo" carry beat, parked via `scenario-event-debugger` `scn_swap` (2026-07-09): sheet at (256,0), entry 1, unchanged
  - D: 28/28 first-frame-id byte-match between the live PSX mid-table slots 0..27 and Godot's `assets/sprites/animations/cinematic_seq.json["1"]` local 0..27 (live scn6 poll, 2026-07-05) — settles the scn6 half of issue #124's Slot→segment indirection empirically (identity here; the chapel's non-identity `Slot 0x88 → seg 0` proves it is not globally identity)
  - R: `godot-learning/src/scenarios/ScenarioVM.gd` (`_resolve_cinematic_segment` tracks the live Load-EVTCHR Slot) + `godot-learning/tests/ScenarioCinematicBandRoutingTest.gd` (pins scn6 Slot 1 → seg 1)
  - src: `research/working_documents/EVTCHR_FRAME_RESOLUTION.md`
- **Each EVTCHR.BIN file block (0x7800; 137 blocks × 0x7800 = the whole file) holds: cinematic SEQ table @ `0x0000` (`0x500`), a 40-entry frame-pointer table (LE32) @ `0x0500`, SHP-TYPE1 4-byte tile descriptors @ `0x05A0`, 16 palettes × 16 colors (BGR555) @ `0x0780`, and a 256×200 4bpp pixel page (`0x200`) with `0xFF` padding @ `0x7000`; a "frame" is one frame-pointer entry → a run of SHP-TYPE1 tile descriptors → tile UVs into that page, so "row/column" is the tile-UV grid.** — `[S·R] 2/3`
  - S: `research/scenario_1_captures/evtchr_load_save_decode.md` §file format (static+dynamic decode); `137×0x7800` = EVTCHR.BIN file size
  - R: `godot-learning/tools/parse_evtchr.py` (`NUM_SEGMENTS = 137`, `SEGMENT_SIZE = 30_720`, `PALETTE_OFFSET = 1920`, `PIXEL_BYTES = 25_600`) + `godot-learning/tools/test_parse_evtchr.py` (`ParseEvtchrSegmentTracerTest`)
  - src: `research/working_documents/EVTCHR_FRAME_RESOLUTION.md`
- **The renderer resolves an EVTCHR sprite from a per-unit tuple: `unit+0x05` = EVTCHR decode working-slot index (0x32D6-byte working buffers @ `0x800C7CE8`), `unit+0x06` = sheet-type / sprite-id (`0x9B`–`0x9E` reserved for the fixed built-in sheets at `DAT_800BF790`/`DAT_800BF830`/`DAT_800BF8D0`/`DAT_800BF990`, else an EVTCHR slot), `unit+0x07` = frame / sub-part index (0x20-byte descriptor stride); the EVTCHR descriptor address is `DAT_800CABDE + unit[+0x07]*0x20 + unit[+0x05]*0x32D6`.** — `[S·D] 2/3`
  - S: `FUN_80082110` @ `0x80082110` (resolver, BATTLE.BIN disassembly, via `evtchr_load_save_decode.md`)
  - D: live RAM read at the scn6 "letgo" carry beat (2026-07-09): Ovelia `+05=0 +06=0x0C +07=0`, Delita `+05=5 +06=0x05 +07=0`
  - R: none — no 0x32D6 working-slot tuple in godot-learning (probed `godot-learning/src/`, `godot-learning/tests/`, `godot-learning/tools/` for `0x32d6`/`0x800c7ce8`)
  - src: `research/working_documents/EVTCHR_FRAME_RESOLUTION.md`
- **The static tile-UV formula of `sprite_subframe_assemble` (`0x80084214`, function base `0x800842A0`) — per 4-byte SHP piece: `U = (bitfield & 0x1F) << 3`, `V = unit[+0x7a] + (((bitfield >> 5) & 0x1F) << 3)`, piece size from bits 10–13 (`sprite_piece_{width,height}_table`), flip bits 14/15, tpage from `unit+0x0e` — is faithful dynamic+static+pixel at the scn6 beat `anim 510 / seqfrm 228 (0xE4)`: the live Ovelia primitive (`unit+0x204`) gives piece[0] `UV (128,80) 32×40 loc(-17,-35)` for 0xE4 (`(160,80)` for 0xE5; consecutive frames step `u += 32` along `v = 80`), and Godot resolves byte-for-byte identically (`anim 510 → local 10 → LoadFrameWait 0xE4 → evtchr_frames["1"][228]` = rect `(128,80) 32×40 loc(-17,-35)`).** — `[S·D·R] 3/3`
  - S: `sprite_subframe_assemble` @ `0x80084214`/`0x800842A0` (BATTLE.BIN disassembly)
  - D: live PSX primitive read + pixel diff at the scn6 carry beat (2026-07-09, `/tmp/sxs/psx_beat_BOTTOM.png`, `/tmp/vram_beat.bin`)
  - R: `godot-learning/tools/parse_evtchr_frames.py` (`decode_block`: `tile_x=(flags&0x1F)*8`, `tile_y=((flags>>5)&0x1F)*8`, size `(flags>>10)&0xF`) + `godot-learning/tools/test_parse_evtchr_frames.py` (`DecodeBlockUnit`, `Segment0KneelFrame`)
  - src: `research/working_documents/EVTCHR_FRAME_RESOLUTION.md`
- **The EVTCHR block-pointer table's runtime frame index is `disc_byte + 7`: the disc table is authored for frame bytes `0xCB..0xF2` but the runtime rewrites the data to `0xD2..0xF9` because the dispatcher `FUN_80083F18` uses `frame_byte >= 0xD2` (`sltiu v0,s2,0xd2`) as the EVTCHR-vs-TYPE1.SHP threshold, so `block_frame_idx = frame_id − 0xD2`; it is a frame-INDEX offset, not an X shift, and never touches `tile_x`.** — `[S·D·R] 3/3`
  - S: `FUN_80083F18` (EVTCHR/TYPE1.SHP dispatcher, BATTLE.BIN disassembly, per `cinematic_frame_offset_decode.md`)
  - D: frame selection validated at the scn6 carry beat (2026-07-09): 0xE4 → correct rect `(128,80)` against the live primitive
  - R: `godot-learning/tools/parse_evtchr_frames.py` (`FRAME_ID_BASE = 0xD2`, `block_frame_idx = frame_id − 0xD2`) + `godot-learning/tools/test_parse_evtchr_frames.py`
  - src: `research/working_documents/EVTCHR_FRAME_RESOLUTION.md`
- **Godot's baked EVTCHR pixel page is 1 row too HIGH vs PSX VRAM, uniformly: whole-sheet mask correlation of the PSX-VRAM sheet against Godot `segment_001.tga` aligns at `dx=0, dy=+1` (identical 14059 non-black px; first content row: PSX VRAM row 5 vs Godot TGA row 4; horizontal is clean at `dx=0`); suspected cause is the parser's `PIXEL_OFFSET = 0x0A00` missing the page's top row — the true page likely starts at `0x980` (the 128-byte `0x980..0x0A00` gap IS the top row) — fix `PIXEL_OFFSET 0x0A00 → 0x980`, re-bake, re-verify.** — `[D·R] 2/3`
  - D: whole-sheet pixel correlation at the scn6 carry beat (2026-07-09, `/tmp/vram_beat.bin` vs baked `segment_001.tga`; PSX framebuffer `/tmp/sxs/psx_beat_BOTTOM.png`)
  - R: `godot-learning/tools/parse_evtchr.py` (`PIXEL_OFFSET`; docstring + commit `ed5c43c48` record the 0x980 pixel-perfect verification) + `godot-learning/tools/test_parse_evtchr.py`
  - src: `research/working_documents/EVTCHR_FRAME_RESOLUTION.md`
- **Scenario 6's mid-cinematic table holds single-frame pose holds: every live slot 0–27 (anims 500–527) at `0x800A77D8` is a `frame, 2; PauseAnimation` entry — the classic cinematic "freeze on a pose" — with frame ids spanning 0xD9–0xF3 (217–243), all inside the runtime EVTCHR frame range [0xD2, 0xF9]; `slot = anim − 0x1F4` confirmed live (anim 500 → slot 0, 527 → slot 27).** — `[D] 1/3`
  - D: live mid-table bytecode dump at scn6 (2026-07-05): slot 0 → `0x800A78E0` = `DA 02 FF FF` (frame 0xDA, wait 2, PauseAnimation), slot 27 → `0x800A7952` = `D9 02 FF FF` (frame 0xD9)
  - R: none — the PSX mid-table pose-hold content is not modeled in godot-learning (probed `godot-learning/src/`, `godot-learning/tests/`, `godot-learning/tools/` for `0x800A77D8`: only band-base constants and the baked-asset parser reference it; the frames are consumed via `cinematic_seq.json`)
  - src: `research/working_documents/SCENARIO6_CARRY_POSE_EVTCHR_RENDER.md`
- **An EVTCHR block is a cinematic BODY-POSE atlas with no face cells: rendered blocks 0/1/2 show ~32px full-body poses (Ovelia kneeling, priest, knights) and no 31×48 face; scenario 1's chapel loads exactly ONE block (`{58} Block=1, Slot=0` — the only real Load EVTCHR before `0xDB Event End` at chunk pc 403; `Block=92/Slot=27457` etc. are string-table garbage), so a block is a per-SCENE pose atlas with no block→character map, and EVTCHR's real consumer is `{11}` Unit Anim with cinematic anim IDs (0x1F4–0x233 / 0x258–0x297) — NOT the dialog portrait (that is the speaker's unit-SPR face, see [[Event Dialogue Portrait System]]).** — `[S·D·R] 3/3`
  - S: `{58}` loader `Open_EVTCHR` (~`0x8013c7d8`; LBA = `Block*15+0x1d4c`) + `Current_EVTCHR_block` resident gate (`battle_decompilation.c`); static-agent renders of blocks 0/1/2 (2026-06-27)
  - D: visual confirm `last_run/vram_portrait_column.png` (2026-06-27): recognizable faces at the X=832 portrait column, far from the EVTCHR VRAM slots (256,0)/(320,0)
  - R: `godot-learning/tools/parse_evtchr.py` + `tools/bake_evtchr_textures.py` (segment decode → baked pose atlases) — validated by `godot-learning/tools/test_parse_evtchr.py`
  - src: `research/working_documents/scenario_1_captures/boxed_dialog_decode.md`

## Notes

(empty — user territory)

## Related

- [[EVTCHR Script VM]]
- [[EVTCHR Character Attribution]]
- [[Event Dialogue Portrait System]]
- [[EVTCHR CLUT Resolution]]
- [[Unit Anim Opcode]]
- [[Cinematic Sprite Renderer]]
