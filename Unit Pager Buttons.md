# Unit Pager Buttons

The ◄L1 / R1► pager buttons that page through roster units, and the sort-tab paging semantics they drive.

## Points

- **Detail mode hides the sort-header row entirely (no header prims in detail); in formation mode the ◄L1 @ (12,14) / R1► @ (224,14) 24×14 pieces are textured `POLY_FT4` on atlas page (960,256), tpage `0x1F`, CLUT `0x7D7C` ⇄ `0x7DFC`, built as 3-slice frames (caps at atlas (176/180,128) + body at (184,128)) plus caption cells `[4,5,218,93,16,5]` / `[5,5,218,88,15,5]`; the L2/R2 pieces are separate overlay pieces (233,88,4,5) whose captions re-texel "1"→"2".** — `[S·D] 2/3`
  - S: `research/working_documents/FORMATION_SCREEN.md` §15.22 (detail-mode header suppression; pager piece geometry, atlas page, tpage, CLUTs, 3-slice frame, caption cells, L2/R2 overlay piece)
  - D: oracle prims in formation vs detail mode (header row absent in detail; pager pieces at the stated coords/CLUTs)
  - R: none — not present in godot-learning (probed main@054e6849e, 2026-08-18: src/ + tests/)
  - src: `research/working_documents/FORMATION_SCREEN.md`
- **Paging through units with ◄/► does NOT re-sort the grid — only the displayed stat column changes; the sort tabs are FIVE keys (`Hp`, `Mp`, `Ct`, `Lv.Exp`, `Br.Fa` — Lv+Exp share one tab, as do Br+Fa), R2 walks through them wrapping to `Hp` and L2 walks the inverse; this overturns the port's old 6-key model; a pressed tab shifts down 1px and swaps CLUT `0x7D7C→0x7E7C` (`formation_sort_tabs_gd` + `world_menu_button_pressed_clut` `0x800E96E4`).** — `[S·D] 2/3`
  - S: `research/working_documents/FORMATION_SCREEN.md` §12.3.5 (no re-sort; five keys; wrap behaviour; port 6-key model overturned)
  - D: oracle paging A/B (grid order byte-stable across page turns; only the stat column values change)
  - R: none — not present in godot-learning (probed main@054e6849e, 2026-08-18: src/ + tests/)
  - src: `research/working_documents/FORMATION_SCREEN.md`

## Notes

(empty — user territory)

## Related

- [[Formation Sort Tab Header]]
- [[Formation Vitals And Nameplate]]
- [[Roster Display Struct]]
