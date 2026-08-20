# Dialogue Pagination

The standard `{10}` portrait dialogue box (Dialog kind `0x10`) is a fixed 3-line render window that does not grow to fit its text: when a single `0xFE`-terminated message exceeds the window, the game paginates it in place — O/Circle advances to the next page within the same box, page 1 shows the red name line plus the first 2 dialogue lines, and each later page shows up to 3 dialogue lines with the name row dropped while the portrait persists. While a next page is pending, an animated page-corner-curl icon (4 frames from the shared RANGETILE atlas, drawn as a separate per-frame GPU primitive over the once-uploaded box texture) animates at the box's bottom-right and is absent on the last page; the advance poll tests bit `0x20` (FFT's internal mask remaps Circle there) and writes the advance action `0x8` to `0x80166080` (idle `0x2`). Closed 2026-07-01 with address-cited statics + live button injection + committed page-1/page-2 reference states; implemented and tested in `godot-learning` (`DialogueBox._paginate`, `ScenarioVM._advance_dialog`, the page-turn icon quad).

## Points

- **The standard `{10}` portrait box is a fixed 3-line window that does not grow — a message longer than the window page-breaks within a single `0xFE`-terminated string: page 1 = the red name line (`{Color 08}`) + the first 2 dialogue lines, each later page up to 3 dialogue lines with the name row dropped, the portrait persists, and the page icon is absent on the last page (Orbonne's Gafgarion 5-line speech = page 1 name + lines 1–2, page 2 lines 3–5).** — `[S·D·R] 3/3`
  - S: kind-`0x10` clamped to exactly 3 lines at `0x80130a60` (`ori v0,zero,0x3`, clamp table `0x80130a30`–`0x80130ae0`); line count measured by `FUN_8013018c` — `0xF8` `{Newline}` scan @ `0x8013026c`, lines = newline count + 1 (battle_disassembly.txt)
  - D: Orbonne reference states `orbonne_dialogue_page1_more_icon.sstate` / `orbonne_dialogue_page2_continuation.sstate` + screenshots in `dialogue_pagination_images/` (user-captured 2026-07-01)
  - R: `godot-learning/src/ui3/assemblies/DialogueBox.gd::_paginate` (page 1 = name + first 2 dialogue lines, page ≥2 = up to 3 with name dropped, fixed 3-line window) + `godot-learning/tests/DialogueBoxTest.gd::_test_pagination_splits_name_plus_three_lines`, `_test_pagination_single_page_no_icon`, `_test_pagination_no_name_three_per_page`
  - src: `research/working_documents/scenario_1_captures/dialogue_pagination_and_page_icon_decode.md`
- **O/Circle advances the paginated box to the next page: `event_dialogue_circle_input_check @ 0x8012fbf8` polls the pad via `SUB_8001db58` (returns `~button_invert_mask` — bits set for pressed buttons), tests bit `0x20` (`andi v0,v0,0x20` @ `0x8012fc00`), and stores the advance action `0x8` @ `0x8012fc3c` into the global at `0x80166080` (idle value `0x2`); only Circle (override bit 13) flips `0x80166080` from `0x2` to `0x8` — Cross, Right, Triangle, Square, Start leave it `0x2` — and holding Circle visually flips page 1 to page 2, proving FFT's internal mask remaps Circle to bit `0x20` (raw-pad bit `0x20` is D-pad Right).** — `[S·D·R] 3/3`
  - S: `0x8012fbf8` / `0x8012fc00` / `0x8012fc3c`, `SUB_8001db58 @ 0x8001db58` (battle_disassembly.txt)
  - D: button-injection table on `orbonne_dialogue_page1_more_icon.sstate` — `PCSX.SIO0.slots[1].pads[1].setOverride(bit)` with `0x80166080` read before/after + visual page flip (2026-07-01)
  - R: `godot-learning/src/scenarios/ScenarioVM.gd::_advance_dialog` (typing → finish; `has_more_pages()` → in-place `advance_page()` with portrait/frame persisting, no re-open tween; else release the gate) + `godot-learning/tests/ScenarioBoxedDialogTest.gd::_test_pagination_advance_press_ladder`, `_test_pagination_auto_advance_pages`
  - src: `research/working_documents/scenario_1_captures/dialogue_pagination_and_page_icon_decode.md`
- **While a next page is pending, the animated page icon is drawn each frame as its own GPU ordering-table primitive over a static box — the box texture (frame + glyphs) is composited and uploaded once when the page is built and is NOT re-uploaded while waiting: on an idle-waiting page-1 state the box compositor `0x8014c18c`, `GsLoadImage 0x800248fc`, and glyph rasterizer `0x8014bd88` all take 0 hits while the heartbeat BP `0x80014590` takes 148 in 0.6 s.** — `[D·R] 2/3`
  - D: idle-waiting Exec-BP differential on `orbonne_dialogue_page1_more_icon.sstate` (~1.2 s, ~72 frames) (2026-07-01)
  - R: `godot-learning/src/ui3/assemblies/DialogueBox.gd::_setup_page_icon` — the icon is a separate 10×12 quad (its own MeshInstance3D, visible only while `has_more_pages()`), never baked into the box frame — validated by `godot-learning/tests/DialogueBoxTest.gd::_test_pagination_splits_name_plus_three_lines`
  - src: `research/working_documents/scenario_1_captures/dialogue_pagination_and_page_icon_decode.md`
- **The page-corner-curl icon's 4 frames (phase mask `andi …,0x3`, 10×12 cells at a fixed 16 px pitch) live in the shared RANGETILE atlas — the LBA-`0xE68` raw-sector disc asset, 256×256 4bpp, loaded by `FUN_80045154` → LoadImage `0x80045178` to VRAM `(960,256)` tpage `0x3F` (the same sheet holding the tile cursor, HUD digits, and Hp/Mp/Ct labels) — NOT in `EVENT/FRAME.BIN` (its `(176,24)` region is item/gem icons): the cells sit just above the Hp/Mp/Ct labels with top-lefts `(172,19)`, `(188,19)`, `(204,19)`, `(220,19)` (UVs inclusive), each an 8×10 page body with a 1–2 px idx-14/15 drop-shadow whose bottom-left corner folds up 0→3 then resets; the phase selector `DAT_8016dafc` is written at `0x8012f8c4` from counter `DAT_8016dad4 & 0x3` at `0x8012f81c`, and the UVs are runtime-assembled rather than literal SPRT rodata, so no source-rect table exists in the disassembly to cite.** — `[S·D·R] 3/3`
  - S: `FUN_80045154 @ 0x80045154` / LoadImage `0x80045178`; phase write `0x8012f8c4` (from `DAT_8016dad4 & 0x3` @ `0x8012f81c`) into `DAT_8016dafc` (write-only in the Ghidra export — its reader is on the un-cross-referenced OT-build path) (battle_disassembly.txt)
  - D: visual characterization from `page1_more_icon` + full-cycle frame-stepped capture `dialogue_pagination_images/page_icon_animation_frames.png`; source rects pinned from the ISO atlas (2026-07-01)
  - R: `godot-learning/tools/parse_range_tiles.py` → `RANGETILE.json` `page_turn_icon` → `assets/ui/page_turn_icon.json` (4 cells + atlas, ISO fixture-guarded); `DialogueBox.gd` samples index→CLUT through `src/ui3/shaders/vitals_sprite.gdshader` with the shared menu/label CLUT `0x7cbc` (the icon's exact ROM CLUT unpinned) and a tunable `page_icon_frames_per_phase` (default 8; the PSX counter rate unpinned) — validated by `godot-learning/tests/DialogueBoxTest.gd::_test_pagination_splits_name_plus_three_lines`
  - src: `research/working_documents/scenario_1_captures/dialogue_pagination_and_page_icon_decode.md`
- **The dialogue-slot globals separate "more pages pending" from "last page": `0x8016609c` reads `0x02` with a next page pending (gates the icon) vs `0x00` on the last page; `0x801660a0` is the lines-on-this-page marker (`0x08` = 2 lines, `0x04` = 1 line, ×4); `0x80166090` is the per-slot glyph/typewriter counter; and the per-slot struct at `0x8016db00` (stride `0x54`) holds the composed text bytes — the `2` at `0x8016609c` does not map 1:1 to "1 remaining line" (page count, next-page line count, or a small state enum; disambiguating needs a longer multi-page capture).** — `[D] 1/3`
  - D: page1 (`orbonne_dialogue_page1_more_icon`) vs page2 (`orbonne_dialogue_page2_continuation`) RAM differential dump of the `0x80166080` array + per-slot struct region (2026-07-01)
  - R: none — the RAM slot fields `0x8016609c`/`0x801660a0`/`0x8016db00` are not present in godot-learning (probed `godot-learning/src/`, `godot-learning/tests/`; the port keeps its own page state in `DialogueBox._pages`/`_page_idx` instead)
  - src: `research/working_documents/scenario_1_captures/dialogue_pagination_and_page_icon_decode.md`

## Notes

(empty — user territory)

## Related

- [[Display Message Opcode]]
- [[Dialogue Box Geometry]]
- [[Concurrent Dialogue Boxes]]
- [[Event Dialogue Portrait System]]
- [[Typewriter Text Cadence]]
- [[Pad Input Handler]]
