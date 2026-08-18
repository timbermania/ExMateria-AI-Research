# Formation Vitals And Nameplate

The formation screen's bottom-left vitals panel + top-right nameplate (the "unit banner"): content pipeline, glyph sources, the dark bottom band, and the banner's dual-thread coroutine.

## Points

- **The vitals panel's "portrait" is the same idle body frame (TYPE1 sequence 2, "Face Front") drawn LARGER — not a separate face asset; every roster body is that unit's own SEQ/SHP `TYPE1` seq 2 held static and un-mirrored, and the SPR face sprite at `0x8200` is dialogue-only.** — `[D] 1/3`
  - D: `research/working_documents/FORMATION_SCREEN.md` §14.1/§14.2 (oracle body-frame identification + enlarged-portrait match, user-confirmed)
  - R: none — not present in godot-learning (probed main@054e6849e, 2026-08-18: src/ + tests/)
  - src: `research/working_documents/FORMATION_SCREEN.md`
- **The zodiac glyphs in the vitals panel are two RANGETILE rows, and their dither background indices (8/4/12) are treated as transparent.** — `[S·R] 2/3`
  - S: `research/working_documents/FORMATION_SCREEN.md` §14.3 (zodiac glyph rows + dither-index transparency)
  - R: `godot-learning/tools/parse_range_tiles.py` + `godot-learning/src/ui3/elements/RangeTileAtlas.gd` (the RANGETILE atlas pipeline the glyph rows sample; the formation glyph cells themselves are not ported in this checkout)
  - src: `research/working_documents/FORMATION_SCREEN.md`
- **Out-of-battle CT renders as a `---/---` full bar, NOT a zero-padded 0 — and numeric fields are zero-padded to fixed widths (3 digits HP/MP/CT, 2 digits Lv/Exp); the padding is a display formatter, not data.** — `[S·D] 2/3`
  - S: `research/working_documents/FORMATION_SCREEN.md` §14.6.3 (CT dash rule + fixed-field padding widths)
  - D: oracle out-of-battle vitals capture showing `---/---` at full CT
  - R: none — not present in godot-learning (probed main@054e6849e, 2026-08-18: src/ + tests/)
  - src: `research/working_documents/FORMATION_SCREEN.md`
- **The panel's label glyph sources are mixed: name/job = FONT.BIN proportional glyphs (advance via the width table @ `0x801661FC`), numeric values = FRAME.BIN small digits @ `0x8014AEC0` (`U = digit·6 + 0x78`, 6×10), and the `Brave`/`Faith` labels = WHOLE-WORD baked RANGETILE textures — NOT FONT.BIN.** — `[S·R] 2/3`
  - S: `research/working_documents/FORMATION_SCREEN.md` §12.3/§14.6.1 (label source table: "name/job = FONT.BIN glyphs (MAIN size); Brave/Faith labels = WHOLE-WORD baked RANGETILE textures, NOT FONT.BIN; values = FRAME.BIN small digits")
  - R: `godot-learning/tools/parse_fft_font.py` (FONT.BIN) + `godot-learning/tools/parse_frame_font.py` + `godot-learning/src/ui3/elements/NumberFont.gd` (FRAMEFONT small-digit pipeline, `U = digit*6 + 0x78`) + `godot-learning/tests/NumberFontTest.gd`
  - src: `research/working_documents/FORMATION_SCREEN.md`
- **The out-of-battle vitals + nameplate sit on a full-width 256px dark band (y169–236) that is the SECOND element (iteration 2) of `FUN_80112C88`: a chain of ~19 flat monochrome `GP0 0x62` semi-transparent RECTANGLES (libgpu `SetTile`, code `0x60` + semitrans `0x02`) stacked 1px-tall to fake the gradient — NOT a gouraud polygon; foreground ramp `12→120→12` step `0x0C`, subtractive ABR=2, tpage `0x5F` — superseding the §14.7.6 POLY_G4/`SetPolyG4` hypothesis (`0x80023DD0` is `SetTile`, not `SetPolyG4`); the band has no animation of its own (the panels slide open, it does not); byte-exact panel text origin is `vitals_origin_px = (8,177)`.** — `[S·D] 2/3`
  - S: `research/working_documents/FORMATION_SCREEN.md` §14.7.6/§14.7.7 (band = `FUN_ROSTER_MENU_OVERLAY__80112c88` second element; 2026-08-01 round-6 primitive-type correction) + §14.6 (byte-exact origins)
  - D: live GPU packet decode at `0x801C02A8` (core `GP0 0x62` color (120,120,120) y178 256×50; top feather `color(12,12,12) pos(128,169) 256×1` ×9) + framebuffer byte-match incl. "dark pixels clamp to 0 first, edges leak cobble"
  - R: none — not present in godot-learning (probed main@054e6849e, 2026-08-18: src/ + tests/)
  - src: `research/working_documents/FORMATION_SCREEN.md`
- **The band is a CUT/SWAP, never a Y-tween: `FUN_80112C88` reads the slide offset `0x8018BA30` (set by `FUN_801161E8` to `0x90` docked / `0` raised from the mode flag) and picks between two fixed band configurations; the ONLY field tracking the keyframe slide is `0x8018BA30` (144→4→0→0), driven by an `int16[12]` keyframe table @ `0x8018ABE0` (WORLD.BIN file offset `0xAABD6`; first seven keys `144,139,67,31,13,4,0`) at the ~30Hz menu tick; the detail screen gets a fresh fixed-Y band rather than the formation's.** — `[S·D] 2/3`
  - S: `research/working_documents/FORMATION_SCREEN.md` §15.6/§15.12 (the §15.6 correction: the dark band does NOT slide — only the field-tracking slide offset moves; §15.12 static grounding of the cut/swap + keyframe table)
  - D: per-frame field trace across the formation→detail transition (band pixels static; `0x8018BA30` 144→4→0)
  - R: none — not present in godot-learning (probed main@054e6849e, 2026-08-18: src/ + tests/)
  - src: `research/working_documents/FORMATION_SCREEN.md`
- **The stats-band content is a SEPARATE builder from the Eqp/Ability window: `world_status_stats_band_builder` `FUN_800EABCC` (screen id `0x23`) draws its labels from a baked RANGETILE descriptor table `0x80155D88` (18 records, CLUT `0x7C3C` — dark ink on tan), and its values are rendered into a VRAM scratch strip by `world_render_value_list` `FUN_800FDF38` + `world_render_number_small` `FUN_800FE37C` — 6px digits `U = digit·6 + 0x78`, `/` at `U=0xB4`, `%` at `U=0xC0` — then blitted with CLUT `0x7FFC`; the "…" leader is 3 dots baked into the numeric font's 6×4 cell at (120,26).** — `[S·D·R] 3/3`
  - S: `research/working_documents/FORMATION_SCREEN.md` §15.18 (band builder vs window builder split, label table, value pipeline, leader-dot cell)
  - D: oracle band crop + descriptor-table live reads matching the rendered band
  - R: `godot-learning/src/ui3/elements/NumberFont.gd` + `godot-learning/tests/NumberFontTest.gd` (the small-digit `U = digit*6 + 0x78` / 6×10 pipeline the band's values use); the band builder itself is not ported in this checkout
  - src: `research/working_documents/FORMATION_SCREEN.md`
- **The vitals + nameplate "unit banner" renderer `FUN_8010D0CC` is a coroutine executed by TWO threads — thread 8 via `WORLD_toggle_unitBanner` (`FUN_80115460`) and thread 7 via `FUN_801154C4` (through `ParseThreadParamWORLD`), with per-thread control/state structs (`0x8018AA98` thread 8 / `0x8018AABC` thread 7, reached at runtime via `*(PTR_DAT_8015327C + 0x2000)`) — and the generic prim-placement helper `FUN_800FDCF0` has 45 call sites across the whole menu system.** — `[S·D] 2/3`
  - S: `research/working_documents/FORMATION_SCREEN.md` §15.11 (dual-thread banner execution, control-struct fields, 45-call-site helper)
  - D: live thread/struct readback (`/tmp/readdr.py`) showing both threads driving the banner per frame
  - R: none — not present in godot-learning (probed main@054e6849e, 2026-08-18: src/ + tests/)
  - src: `research/working_documents/FORMATION_SCREEN.md`

## Notes

(empty — user territory)

## Related

- [[Formation Sort Tab Header]]
- [[Formation Screen Compositing]]
- [[Unit Pager Buttons]]
- [[Equip And Ability Panel]]
