# Equip Sub Screen

The START-"Item" → Equip sub-screen: an in-place sub-state of the persistent formation controller, its 2D sprite-slide transition, slot focus, and the item picker windows.

## Points

- **The START-"Item" → equipment sub-screen is an IN-PLACE sub-state switch inside the persistent formation controller `FUN_8010CCA4` (screen id `0x39`), NOT a new full-screen dispatch: the menu window is content-swapped in place (the same slot-6 list menu, 5 items → 4 items: Equip/Best/Remove/List) and there is no backgrounding/blue tint.** — `[S·D] 2/3`
  - S: `research/working_documents/FORMATION_SCREEN.md` §15.23 (in-place sub-state switch; controller + screen id; 5→4 menu items)
  - D: oracle A/B of the START-Item press (same window slot, content-swapped; no CLUT backgrounding)
  - R: none — not present in godot-learning (probed main@054e6849e, 2026-08-18: src/ + tests/)
  - src: `research/working_documents/FORMATION_SCREEN.md`
- **The equipment sub-screen transition is a 2D SPRITE SLIDE over a static background, NOT a camera dolly: non-selected units slide off to the right while the selected unit slides into position with no size change; this supersedes RE20's "kernel camera dolly via `gte_camera_translation_vec 0x80037100`" — that address is the sound driver (entity 2 `0x800370E0`), not a camera (RE round 21 correction, 2026-08-06).** — `[S·D] 2/3`
  - S: `research/working_documents/FORMATION_SCREEN.md` §15.24 / RE21 correction (2D sprite slide; retracted camera names)
  - D: frame-by-frame transition capture (static terrain, sliding sprites, constant sprite size)
  - R: none — not present in godot-learning (probed main@054e6849e, 2026-08-18: src/ + tests/)
  - src: `research/working_documents/FORMATION_SCREEN.md`
- **On the settled equipment screen the orb + gold box disappear; the floor spotlight pool instead tracks the sliding unit (`0x8018BAD0` eased to (164,177)) — the sweep must sample the unit's live box centre, not the statically docked cell.** — `[S·D] 2/3`
  - S: `research/working_documents/FORMATION_SCREEN.md` §15.24 (orb/box removal on the equip screen; pool retarget `(164,177)` − body-anchor offset)
  - D: settled-equip-screen oracle (no orb/box prims in the OT; pool centred on the standing unit)
  - R: none — not present in godot-learning (probed main@054e6849e, 2026-08-18: src/ + tests/)
  - src: `research/working_documents/FORMATION_SCREEN.md`
- **The Eqp slot-focus sub-state: ○ on the "Equip" row transfers input focus to the slot list; the glove anchors to the panel frame's left edge (`x = 13 − 12 + bob ≈ 1`, half off-screen), `y = 132 + row·16 + 10`; the list menu backgrounds via the discrete CLUT swap `0x7C3C→0x7D3C` (the luminance inversion is the fingerprint of a CLUT index remap, not a tint); the "Remove"/"Blank" menu rows are FONT.BIN glyphs through the shared list-menu glyph path, NOT baked RANGETILE words.** — `[S·D] 2/3`
  - S: `research/working_documents/FORMATION_SCREEN.md` §15.25 (slot-focus sub-state, glove anchor, CLUT-swap backgrounding, row glyph source)
  - D: oracle slot-focus captures (glove half off-screen; tan→blue-grey swap on the list)
  - R: none — not present in godot-learning (probed main@054e6849e, 2026-08-18: src/ + tests/)
  - src: `research/working_documents/FORMATION_SCREEN.md`
- **The equipment picker is two INDEPENDENT WORLD windows plus two overlapping stats panels: the item-list window is slot `0xF` (builder `0x8011241C`, row renderer `0x80128CE0`); the stats/compare panel builder `0x80111EC4` is registered in slots `0xB` (base) and `0xC` (delta); the stats-panel frame is a 12-cell gradient LINE frame (`world_gradient_line_frame` `0x800E3358`), NOT a 9-slice; the picker state machine is `0x8011AF4C`; row Y = `idx·16 + 144`.** — `[S·D] 2/3`
  - S: `research/working_documents/FORMATION_SCREEN.md` §15.26 (picker window topology, builders, gradient LINE frame, state machine, row geometry)
  - D: live window-slot reads (slots 0xb/0xc/0xf) + frame-capture of the gradient LINE frame
  - R: none — not present in godot-learning (probed main@054e6849e, 2026-08-18: src/ + tests/)
  - src: `research/working_documents/FORMATION_SCREEN.md`
- **The picker's item icon = ITEM.BIN `POLY_FT4`, tpage `0x1E` → VRAM (896,256): `U = (graphic%15)·16`, `V = (graphic/15)·16 + 0x20`, 16×16, with `graphic = itemRecord[1]` (`world_item_icon_place 0x800EA990`); the per-type icon CLUT = `GetClut(DAT_80153198 + (type%8)·16, DAT_8015319A + type/8)` with `DAT_80153198/9A = 640/254` (= `0x3FA8 + (type&7)·0x10 + (type>>3)·0x40`); the type glyph is a 12×12 cell via LUT `0x8018D7FC` at idle CLUT `0x7C3C` — round-47 revision: the "ALL" header cell is `(45,120,16,8)` and the glyph is a 13×8 at `U+1, V+6` through CLUT `0x7D7C`.** — `[S·D] 2/3`
  - S: `research/working_documents/FORMATION_SCREEN.md` §15.26/§15.30 (icon formula + CLUT formula, `type = itemRecord[0]`; round-47 type-glyph LUT revision incl. the initially-missed ALL cell)
  - D: live-proven — the formula predicts the measured prims exactly (u192,v32 → graphic 12; u16,v32 → 1; u0,v128 → 90); causal CLUT-swap proof: overwriting CLUT `0x3FA8` @VRAM(640,254) with green turned exactly the picker icons + equipped mini-icons green while the type-glyph column (`0x7DFC`) stayed grey; live-proven with pal 0/5/6/11
  - R: `godot-learning/tools/extract_item_sprites.py` (ITEM.BIN 4bpp region + 16 palettes → `assets/items/item_icons.tga`) + `godot-learning/assets/items/items.json` (the `palette`/`graphic`/`slot_flags` fields the formulas consume)
  - src: `research/working_documents/FORMATION_SCREEN.md`
- **The ability sub-screen is the mirror of the equipment one: panel `LOWER_FRAME_ABILITY` `Rect2(13,132,117,94)`, the ability window repositioned to the left, and the menu container at (200,132,56,64) with 3 rows Set/Remove/Learn; the Eqp-only panel rect is measured at `Rect2(13,132,125,94)` (RE round 23).** — `[S·D] 2/3`
  - S: `research/working_documents/FORMATION_SCREEN.md` §15.27 (ability mirror geometry) + §15.24 RE round 23 (Eqp-only panel rect)
  - D: oracle captures of the ability sub-screen vs the equipment sub-screen (mirror layout confirmed)
  - R: none — not present in godot-learning (probed main@054e6849e, 2026-08-18: src/ + tests/)
  - src: `research/working_documents/FORMATION_SCREEN.md`

## Notes

(empty — user territory)

## Related

- [[Start Action Menu]]
- [[Equip And Ability Panel]]
- [[Equip Stat Delta Preview]]
- [[Selection Orb And Floor Spotlight]]
