# Equip And Ability Panel

The formation screen's shared Eqp+Ability bottom window: geometry, title tabs, slot-category icons, and the item icons on the status screen.

## Points

- **The Eqp/Ability window is a single MONOLITHIC background panel (not two stacked boxes): 234×92, display origin (12,134), CLUT `0x7C3C`; the two "TILE" bands are flat `rgb(48,48,48)` quads 16×90 at (30,135) and (136,135), tpage `0x5F`, ABR=2, drawn first.** — `[S·D] 2/3`
  - S: `research/working_documents/FORMATION_SCREEN.md` §14.3 (monolithic panel geometry + two flat TILE bands)
  - D: oracle settled-frame measurement (234×92 at (12,134); bands rgb(48,48,48))
  - R: none — not present in godot-learning (probed main@054e6849e, 2026-08-18: src/ + tests/)
  - src: `research/working_documents/FORMATION_SCREEN.md`
- **The "Eqp"/"Ability" title tabs are RANGE-texel slices at (29,131) 18×10 and (131,131) 26×10 (uv (28,32)/(0,32)), CLUT `0x7CBC`, drawn by the window builder `FUN_800EAF3C`; they are the panel's header, not independent windows.** — `[S·D] 2/3`
  - S: `research/working_documents/FORMATION_SCREEN.md` §14.3 (tab texels/positions + builder)
  - D: oracle crop of the settled panel header
  - R: `godot-learning/tools/parse_range_tiles.py` + `godot-learning/src/ui3/elements/RangeTileAtlas.gd` (the RANGETILE atlas the tab slices sample)
  - src: `research/working_documents/FORMATION_SCREEN.md`
- **The 6 equipment slot labels are two-layer EMBOSSED icons: a dark back layer at v=128 (CLUT `0x7C3C`) under a lit front layer at v=140 (CLUT `0x7D7C`); weapons have an extra "2H" patch tile at (48,144) 12×12 and hide the L.hand layer.** — `[S·D] 2/3`
  - S: `research/working_documents/FORMATION_SCREEN.md` §14.3/§15.30 (two-layer emboss, 2H patch, L.hand suppression)
  - D: oracle panel crops showing the two-layer emboss + the 2H patch on two-handed weapons
  - R: none — not present in godot-learning (probed main@054e6849e, 2026-08-18: src/ + tests/)
  - src: `research/working_documents/FORMATION_SCREEN.md`
- **The status-screen item icons use the OTHER (status-screen) icon formula: ITEM.BIN graphics `0x0000–0x7FFF` at VRAM (896,256), palette `0x8000+` at (640,254) = CLUT base `0x3FA8 + palette` (5-bit BGR555 palettes); `U = (graphic%16)·16`, `V = (graphic//16)·16`; SCUS OldItem records are 8 bytes @ `0x536B8+` with palette @+0, graphic @+1; the equipment picker instead uses the 15-column formula (`U=(graphic%15)·16, V=(graphic/15)·16+0x20`).** — `[S·D·R] 3/3`
  - S: `research/working_documents/FORMATION_SCREEN.md` §15.24 RE30 (status-screen formula, VRAM targets, 5-bit BGR555 palette, SCUS OldItem layout, contrast with the picker formula)
  - D: live item-icon capture on the status screen (16×16 quads at the formula's U/V; palettes per the `0x3FA8+pal` rule)
  - R: `godot-learning/tools/extract_item_sprites.py` (ITEM.BIN → `assets/items/item_icons.tga` with 16 palettes) + `godot-learning/assets/items/items.json` (254 items with the `palette`/`graphic`/`slot_flags`/`item_type` fields)
  - src: `research/working_documents/FORMATION_SCREEN.md`

## Notes

(empty — user territory)

## Related

- [[Equip Sub Screen]]
- [[Formation Vitals And Nameplate]]
- [[Menu Window Box Open]]
