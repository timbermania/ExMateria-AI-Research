# Formation Sort Tab Header

The formation roster's sort-tab header row (y≈14–29): L2/R2 buttons, the six sort labels, the active-tab highlight, and the per-cell HP/sort fill swatch.

## Points

- **The sort-tab header is three separate pieces on three different draw paths, not one bar: (A) the L2/R2 buttons — textured `POLY_FT4` from the atlas page (960,256) tpage `0x1F` CLUT `0x7D7C`, emitted EVERY frame by `menu_sprite_setter`, each button a 3-slice window frame (caps atlas 176/180,128 + body 184,128) plus caption cells, with pressed state = Y+1 shift and CLUT swap `0x7D7C→0x7E7C`; (B) the six sort labels — CACHED textured word-cells (`Hp (168,32,16×9)`, `Mp (184,32,16×9)`, `Ct (200,32,16×8)`, `Lv. (48,16,14×8)`, `Exp. (146,32,20×10)`, `Br (216,32,10×9)`, `Fa (25,16,9×8)`), inactive CLUT `0x7C3C` (stroke idx1 = dark ink (48,40,32), fill idx4 = the bar's exact tan (152,144,120) — a real 2-tone emboss) and active CLUT `0x7CBC` (white (232,232,224) strokes on a dark (32,24,16) fill); (C) a near-black (40,40,32) rounded highlight box (~x60–82, y16–27) drawn behind the active tab; all sitting on a tan beveled window bar (fill (152,144,120), x≈60–194, brighter mid (168,160,136), top bevel (208,200,168)).** — `[S·D] 2/3`
  - S: `research/working_documents/FORMATION_SCREEN.md` §12.3.2 (VRAM-oracle re-derivation: per-piece atlas U/V, the four real VRAM CLUTs read from the dump, static emitter `label_glyph_emitter @0x80115d6c` + geometry table `0x8018C230`)
  - D: full-VRAM + framebuffer oracle (sstate0 settled roster) + `menu_sprite_setter` BP descriptor log; rendering the label cells through the measured CLUTs reproduces the oracle pixel-exactly (`RECON_labels7x.png` vs `ORACLE_hdr7x.png`)
  - R: `godot-learning/tools/parse_range_tiles.py` + `godot-learning/src/ui3/elements/RangeTileAtlas.gd` (the shared RANGETILE atlas + `WORD_LABELS` pipeline; the formation header itself is not ported in this checkout)
  - src: `research/working_documents/FORMATION_SCREEN.md`
- **The header is CACHED, not redrawn per frame: `header_caller @0x801128e0` (driving `sort_tab_header_builder`) fired 0× in 5 steady-state frames but 22× during an injected R2 press, gated by the `DAT_80153334 = 5` latch — later renamed `formation_slot_change_anim_countdown`, a 5-frame countdown kicked by any formation-stack slot's `+0x48` field changing (−1/frame at `0x800f6c74`), NOT a view-mode flag; and the mislabelled `sort_tab_header_builder` `FUN_80112C88` does NOT draw the header at all — it builds the two full-width 256px row backdrops (see [[Formation Vitals And Nameplate]]).** — `[S·D] 2/3`
  - S: `research/working_documents/FORMATION_SCREEN.md` §12.3.2 (cache latch) + §15.24/RE round 21 (countdown rename, `FUN_80100164` = `world_formation_slot_field48_get` @ `0x80195CD0 + slot·0x400 + 0x48`)
  - D: BP call-count log (0× steady-state vs 22× after injected R2) + 30-frame live capture of the counter oscillating 4↔3 then settling to 0
  - R: none — not present in godot-learning (probed main@054e6849e, 2026-08-18: src/ + tests/)
  - src: `research/working_documents/FORMATION_SCREEN.md`
- **The per-cell HP/sort fill swatch is a FIXED-size textured `POLY_FT4` — `cellX, cellY+8`, W=`0x25` (37) × H=3, sampling the shared bar swatch at VRAM (216,202) (U=0xD8 V=0xCA) — with NO HP-proportional length and no casing/track prim (unlike `draw_vitals_bars`); its CLUT is swapped by the active sort column (Hp/Mp/Ct, per-column CLUTs at `0x8019DFA6/A8/AA`).** — `[S·D] 2/3`
  - S: `research/working_documents/FORMATION_SCREEN.md` §12.3.4 (swatch prim at `0x80118440`, BSS tpage `0x8019DF88`) — including the correction of the earlier "fill length f(HP%4)" misread: the `(arg&3)*16+0x140` value feeds `GsGetClut` as a CLUT column selector, not an HP width (that pattern is the monster-only texture-column variant, tpage abr2)
  - D: oracle sstate captures of the swatch across sort keys; fixed 37px width confirmed at full HP and low HP
  - R: none — not present in godot-learning (probed main@054e6849e, 2026-08-18: src/ + tests/)
  - src: `research/working_documents/FORMATION_SCREEN.md`
