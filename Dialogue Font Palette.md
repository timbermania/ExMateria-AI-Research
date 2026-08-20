# Dialogue Font Palette

The `{10}` Display Message font-palette mechanism, settled by static + dynamic analysis (2026-06-27): the only palette input is the `{Color NN}` string opcode `0xE3` — the handler's message walk latches it into `DAT_8016dae2` and passes it to the 2bpp→4bpp glyph rasterizer `FUN_8014bd88 @ 0x8014BD88`, where `final_index = glyph_pixel(1..3) + {Color NN}` is an additive offset into a 16-color dialog CLUT (`{Color 00}` default = slots 1–3 body, `{Color 08}` = slots 9–11 speaker name). Box type (`Dialog & 0x70`) drives box geometry/anchoring only and never touches the glyph CLUT. Which 16-color row a glyph actually samples is decided by the primitive's CLUT id, and all 22 FRAME.BIN palettes are resident in VRAM simultaneously (pal 0–15 at X=960 rows 496–511, pal 16–21 at X=976 rows 506–511): the chapel prayer (`Dialog=0x09`) carries CLUT id `0x7cbc` = VRAM (960,498) = FRAME pal 2 (off-white ramp) while the boxed dialog's glyphs carry `0x7c3c` = VRAM (960,496) = FRAME pal 0 (dark body (49,41,32), red speaker (106,41,16)) — the box path never takes the conditional `0x7cbc` store @ `0x80130FFC`, and the prayer renders through a separate pre-rasterized path that skips the rasterizer + `GsLoadImage` upload, so box body ≠ prayer body. `godot-learning` implements the box with a 3-level nearest-neighbor CUSTOM shader remap to the dumped CLUT (`DialogueBox` + `ui_font_char.gdshader`) and the prayer/menu text with the baked off-white atlas (FRAME pal 2).

## Points

- **String opcode `0xE3` (`{Color NN}`) is the ONLY palette input to the dialog font: the `FUN_801308c0` message-string walk writes `local_d4` (default 0) solely from `0xE3`, latches it to `DAT_8016dae2`, and passes it as `param_4` to the 2bpp→4bpp glyph rasterizer `FUN_8014bd88 @ 0x8014BD88`, which computes `final_4bpp_index = glyph_2bpp_pixel(1..3) + {Color NN}` — an additive offset into a single 16-color dialog CLUT, where `{Color 00}` (default) uses slots 1–3 (body ramp) and `{Color 08}` uses slots 9–11 (speaker-name ramp); box type (`Dialog & 0x70`) drives box geometry/anchoring only and never touches the glyph CLUT (statically proven).** — `[S·D·R] 3/3`
  - S: `0xE3`/`0xE2` string walk inside `FUN_801308c0 @ 0x801308C0` (`local_d4` latch → `DAT_8016dae2`) + the index-offset lines in rasterizer `FUN_8014bd88 @ 0x8014BD88` (size 0x150) (`battle_decompilation.c` / `battle_disassembly.txt`)
  - D: D1/D2 cross-checks on `orbonne_prayer_mid_dialog.sstate` (2026-06-27) — the D1 CLUT dump pins slots 1–3 / 9–11 as the body / speaker ramps and the D2 on-screen read confirms the `{Color 00}` body and `{Color 08}` speaker land in different ramps
  - R: `godot-learning/src/scenarios/TypewriterController.gd` `typewriter_set_palette` + `src/ui3/elements/UIText.gd` `set_char_clut_run` + the 3-level nearest-match remap in `src/ui3/shaders/ui_font_char.gdshader` (`use_palette`, `palette_dark/mid/light`) — validated by `godot-learning/tests/DialogueBoxTest.gd::_test_palette_runs_split_header_body`
  - src: `research/working_documents/scenario_1_captures/display_message_dialog_type_and_palette_decode.md`
- **All 22 FRAME.BIN palettes are uploaded and resident in VRAM SIMULTANEOUSLY — pal 0–15 at X=960 rows 496–511, pal 16–21 at X=976 rows 506–511 — so a glyph's CLUT id just selects a row (`id = (Y<<6) | (X/16)`): VRAM (960,498) is NOT a per-context "compose slot" that gets different RGB depending on what's open; the box and the prayer differ because their glyph primitives carry different CLUT ids, and the static `sh 0x7cbc` store @ `0x80130FFC` (Ghidra label `dialog_font_clut_id_pal2_store`) is one conditional branch (`v1==0`) that the box path never takes.** — `[S·D] 2/3`
  - S: `sh 0x7cbc` @ `0x80130FFC` (`battle_decompilation.c`) + Ghidra label `dialog_font_clut_id_pal2_store` (renames_high.tsv)
  - D: VRAM byte-match of all 22 resident FRAME palettes, `orbonne_prayer_mid_dialog.sstate` (2026-06-27)
  - R: none — the all-22-palettes-resident-simultaneously property is not modeled in godot-learning (probed `godot-learning/src/`, `godot-learning/tools/`, `godot-learning/tests/`)
  - src: `research/working_documents/scenario_1_captures/display_message_dialog_type_and_palette_decode.md`
- **The chapel-prayer dialog's 16-color CLUT — CLUT id `0x7cbc` = `(498<<6)|60` = VRAM (960,498) = FRAME pal 2 — was dumped: slot 0 transparent, body-ramp slots 1–3 = (238,238,230) / (156,156,148) / (82,82,74) off-white, speaker-ramp slots 9–11 = (156,148,123) / (115,106,90) / (139,123,106) tan; the speaker ramp is NOT a scalar multiple of the body ramp (slot 11 is brighter than slot 10, unlike the body where px3 < px2), so a white `font_color` tint can never reproduce it; the prayer (`Dialog=0x09`, Open Type `0`) carries no `{Color}` token at all and rides the default palette 0 = body ramp.** — `[S·D·R] 3/3`
  - S: CLUT id `0x7cbc` stored @ `0x80130FFC` (`battle_decompilation.c`); Ghidra `dialog_font_clut_id_pal2_store` (renames_high.tsv)
  - D: `scripts/probe_d1_dialog_clut.py` dumped VRAM (960,498) from `orbonne_prayer_mid_dialog.sstate` (2026-06-27); scenario-1 capture (`scenario_1_chunk.json`) — the prayer (idx 42) is the lone box-type-0 call and the only one with no `{Color}` token
  - R: `godot-learning/tools/parse_fft_font.py` `ATLAS_PALETTE`/`DEFAULT_PALETTE` (the off-white slots 0–3 above) baked into `font_atlas.tga`, consumed by `src/scenarios/DialogueOverlay.gd` (atlas blit, untinted) — validated by `godot-learning/tests/DialogueOverlayTest.gd::_test_color_and_newline_drain_free` (`{Color}` token routing on the overlay path; the RGB values are pinned in the bake constants)
  - src: `research/working_documents/scenario_1_captures/display_message_dialog_type_and_palette_decode.md`
- **The boxed-dialog font samples FRAME pal 0 @ VRAM (960,496) (CLUT id `0x7c3c`) instead of the prayer's off-white pal 2 — dark body stroke (49,41,32), red speaker stroke (106,41,16) — and the prayer instead renders via a separate pre-rasterized path that skips the `FUN_8014bd88` rasterizer + the `GsLoadImage @ 0x800248FC` upload inside `event_dialogue_tick @ 0x8012F6D4`, so box body ≠ prayer body; the doc's initial "one shared 0x7cbc CLUT, box body == prayer body" framing is wrong, corrected by the doc's own 2026-06-27 VRAM byte-match.** — `[S·D·R] 3/3`
  - S: conditional `sh 0x7cbc` @ `0x80130FFC` (the box path does not take it) + `GsLoadImage @ 0x800248FC` in `event_dialogue_tick @ 0x8012F6D4` (box path only) (`battle_decompilation.c`)
  - D: live VRAM byte-match, `orbonne_prayer_mid_dialog.sstate` (2026-06-27) — box glyph CLUT id = `0x7c3c`, pal-0 row byte-matched; on-screen prayer stays near-white (pal 2)
  - R: `godot-learning/src/ui3/assemblies/DialogueBox.gd` (BODY_STROKE/BODY_HILITE/BODY_AA + SPEAKER_STROKE/SPEAKER_HILITE/SPEAKER_AA constants, `set_char_clut_run` recolor, no font_color tint) + `tools/parse_fft_font.py` `DIALOG_CLUT` — validated by `godot-learning/tests/DialogueBoxTest.gd::_test_palette_runs_split_header_body`
  - src: `research/working_documents/scenario_1_captures/display_message_dialog_type_and_palette_decode.md`

## Notes

(empty — user territory)

## Related

- [[Display Message Opcode]]
- [[Dialogue Box Geometry]]
- [[Dialogue Pagination]]
- [[Event Dialogue Portrait System]]
