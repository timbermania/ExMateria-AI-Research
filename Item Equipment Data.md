# Item Equipment Data

The cross-screen item/equipment data domain behind the formation equip picker: the 12-byte item record table (RAM `0x80062EB8`, = `items.json`), id-range slot legality, the candidate-list builder with its job-equip bitmask, per-category most-recent-first display ordering, and the icon/CLUT/type-glyph data plumbing. All claims are disassembly-address-cited and live-validated on pcsx-redux (RE round 49, 2026-08-11), and they correct two FORMATION_SCREEN.md §15.26/§15.28 misreads: the off-by-4 record base (`0x80062EBC` → `0x80062EB8`) and the 0x4000 candidate-flag semantics ("not equippable by this unit's job", not "equipped by this unit"). godot-learning has ported the data layer (`items.json` via `extract_items.py` + `ItemDatabase`) and per-palette icon sheets, but the picker's slot gate, candidate builder, ordering lists, and row rendering are not ported.

## Points

- **The 12-byte item record table sits at RAM base `0x80062EB8` (SCUS) with `scus_item_record 0x8005A884(id)` returning `0x80062EB8 + (id&0xff)·12`; field map is rec[0]=palette, rec[1]=graphic, rec[2]=enemy_level, rec[3]=slot_flags, rec[4]=second_table_id, rec[5]=item_type_id (the "class byte"), rec[8..9]=price, rec[10]=shop_availability — the `0x80062EBC` base written in §15.26-b/§15.28 was an off-by-4 misread.** — `[S·D·R] 3/3`
  - S: `scus_item_record 0x8005A884` / `scus_item_record_table 0x80062EB8` (`labels_scus.tsv`; SCUS bytes at file 0x4B084, base `0x8000F800`, disassembled by hand with `mipsdis.py`); icon placer `0x800EA990` @+0x18/+0x28 (`lbu 0x1(v0)`, `lbu 0x0(v0)`); class read `world_item_class_byte 0x80124FD0` = `lbu(0x80062EBD + (id&0x3ff)·12)` = base+5
  - D: RE round 49 (2026-08-11, artifacts `/tmp/item_re/`): RAM `0x80062EB8..+0xC00` byte-identical to SCUS file `0x536B8` (= `extract_items.py ITEMS_OFFSET`); all 254 items × {palette, graphic, item_type_id, slot_flags} byte-compared RAM vs `items.json`: 0 mismatches (oracles `SCUS94221.sstate0` + pad-injected Head/Body/Accessory picker states)
  - R: `godot-learning/tools/extract_items.py` (`ITEMS_OFFSET = 0x536B8`, `parse_common_item` fields data[0..10]) + `godot-learning/src/data/ItemDatabase.gd` (items.json loader); validated by `godot-learning/tests/UnitWeaponBindTest.gd` (item_type_id-driven weapon sprite selection)
  - src: `research/working_documents/ITEM_EQUIPMENT_DATA.md`
- **The equip picker's slot legality is a pure item-id range gate (`world_item_slot_category 0x80125374`): R.Hand/L.Hand accept category 0 (weapon) OR 1 (shield) in either hand, Head/Body/Accessory accept exactly 2/3/4, and the picker never reads the rec[3] `slot_flags` byte (it agrees with the ranges 254/254 but is other consumers' data); the head/body boundary inside the extractor's combined "armor" category is id 0x90–0xAB / 0xAC–0xCF.** — `[S·D] 2/3`
  - S: slot gate `0x80124C54` @0x80124D24-0x80124DC8; `world_item_slot_category 0x80125374`; rec[4] stat sub-tables confirmed against our own disassembly: weapons `0x80063AB8` (stride 8), shields `0x80063EB8` (stride 2), armor `0x80063ED8` (stride 2, indexed id−0x90), accessories `0x80063F58` — exactly `extract_items.py`'s offsets
  - D: RE round 49 (2026-08-11): the four slot pickers listed exactly [19,2,1] / [157] / [186] / [208] for the party; the causal poke test proved a shield candidate IS accepted by the R.Hand picker
  - R: none — equip picker slot gate not present in godot-learning (probed godot-learning/src + tests, 2026-08-18; `ItemDatabase.get_items_by_category` uses the items.json `category` string, not the ROM slot gate)
  - src: `research/working_documents/ITEM_EQUIPMENT_DATA.md`
- **`world_equip_candidate_builder 0x80124C54` (the formation picker passes mode 1) includes an item only when its party total > 0 (`world_item_party_total 0x801237E4` = per-item stock ledger byte `DAT_800596E0[id]` + `world_item_equipped_count 0x80123764` over roster `DAT_801CD5EC` units, slot at unit+0x54+slot·2), gates per slot, and job-checks via `world_unit_can_equip 0x8012457C` which bit-tests the unit's 5-byte MSB-first job equip mask at `unitStruct+0x73` indexed by the class byte rec[5]; mode 1 keeps not-equippable items and flags them 0x4000 — candidate flag 0x4000 means "NOT equippable by this unit's job", not "equipped-by-this-unit" (correction to §15.26 A); the list is emitted to `DAT_801CD230` as `id | flags` shorts, −1-terminated, with the total count in `DAT_801CD824`.** — `[S·D] 2/3`
  - S: `0x80124C54` (call site `0x8011B0A4-0x8011B0C8` inside `world_equip_picker_statemachine 0x8011AF4C`; bit-stream readers `0x8012B1B4`/`0x8012B1EC`; focused slot from `FUN_8012BA14(5,2,…)`, cursor `DAT_8018CA66`)
  - D: sstate0 (Boyce, Squire; job mask `d2 40 04 be c0` → Knife✓ Sword✓ Rod✗ Shield✗ Hat✓): Broad Sword *equipped on* the unit is unflagged, while poked-in Escutcheon and Rod both came back 0x4000 and Long Sword didn't (`ram_poked2.bin`); `picker_poked.png` shows flagged rows render name + "NN/NN" counts through the grey bank (count digits CLUT 0x7FA4 vs 0x7C3C normal) with icon + type glyph full-color (2026-08-11)
  - R: none — equip candidate builder / job equip bitmask not present in godot-learning (probed godot-learning/src + tests, 2026-08-18)
  - src: `research/working_documents/ITEM_EQUIPMENT_DATA.md`
- **Candidate display order is the party's per-category runtime acquisition list (most-recent-first), not static: `world_candidate_order_pass 0x801220D8` rebuilds the list following the pointer array `world_item_display_list_ptrs 0x8018D7D8` (5 words) to SCUS lists `0x80057B5C` (kind 0 = weapons+shields shared) / `0x80057BE8` (head) / `0x80057C08` (body) / `0x80057C30` (accessory) / `0x80057C54` (chemist); the 0xFF-terminated byte lists are empty in SCUS, populated in RAM, front-inserted on acquisition (`world_item_list_front_insert 0x801222D0`), and a full re-sync (`world_item_list_sync 0x801221D8`) yields descending item id; the picker does NOT sync at open (causally proven), and the order pass drops candidates missing from the list; saves round-trip the lists.** — `[S·D] 2/3`
  - S: `0x801220D8`, `0x8018D7D8`, `0x801221D8`, `0x801222D0`, `0x80122338` (remove), `0x8012227C` (contains); save serialization `0x80130114`/`0x80130648` into save buffer `DAT_801CD1EC+0x1CBC`; new-game initializer `0x800455B0+` empties the lists and seeds starting stock; slot codes ≥5 (shop/Item modes) sort via `FUN_80121654` instead
  - D: RE round 49 poke test (2026-08-11, `ram_poked.bin`): poking only the stock ledger left the picker at [19,2,1]; additionally poking the list to [128,51,20,19,2,1] made the picker show exactly that order — order comes from the list, not the stock
  - R: none — per-category acquisition display list not present in godot-learning (probed godot-learning/src + tests, 2026-08-18)
  - src: `research/working_documents/ITEM_EQUIPMENT_DATA.md`
- **The picker item icon's CLUT is keyed on record byte rec[0] (= items.json `palette`), not the class byte: `GetClut(0x280 + (palette%8)·16, 0xFE + palette/8)` (constants `DAT_80153198 = 0x280 = 640`, `DAT_8015319A = 0xFE = 254`) gives CLUT id = `0x3FA8 + palette` (palette 0–7, VRAM row 254) / `0x3FE8 + (palette−8)` (palette 8–15, row 255); the provider chain is slot 3: `0x8011AAA4` (reads `DAT_801CD230[row]` low byte) → `FUN_80112878` (descriptor assembly, W=H=16, tpage `0x1E`) → `0x800EA990`.** — `[S·D·R] 3/3`
  - S: `0x800EA990` @+0x18 (`lbu 0x1(v0)` graphic) / @+0x28 (`lbu 0x0(v0)` palette); `GetClut` `SUB_80023A54` with live-read constants; `world_item_icon_provider 0x8011AAA4` / `world_item_icon_descriptor_fill 0x80112878` (`renames_high_world.tsv`)
  - D: RE round 49 live picker icon prims (2026-08-11): Broad/Long Sword, Escutcheon, Battle Boots (pal 0) → 0x3FA8; Rod (pal 5) → 0x3FAD; Clothes (pal 6) → 0x3FAE; Leather Hat (pal 11) → 0x3FEB; every UV decoded to the items.json `graphic` (12/13/71/132/35/116/90)
  - R: `godot-learning/tools/extract_item_sprites.py` (ITEM.BIN 4bpp region + 16 palettes → gitignored `assets/items/item_icons.tga` / `item_icons_pal%d.tga`) consumed per-palette by `godot-learning/src/projectiles/Projectile3D.gd` (`item_icons_pal%d.tga` selected by the record palette byte)
  - src: `research/working_documents/ITEM_EQUIPMENT_DATA.md`
- **The picker window shows at most 5 visible rows (16px pitch; icon rows at y 147/163/179/195/211 in draw-env space) with 6th+ candidates scrolling (`DAT_801CD54C` scroll value, `DAT_801CD824` = total candidate count); the "NN/NN" row counts are equipped-by-team (`world_item_equipped_count 0x80123764`) / party total (`world_item_party_total 0x801237E4`), and the count provider `0x8011A96C+` forwards the 0x4000 job flag as bit 0x40000000 on the count value.** — `[S·D] 2/3`
  - S: `DAT_801CD54C`, `DAT_801CD824`, `0x80123764`, `0x801237E4`, `0x8011A96C+`
  - D: RE round 49 (2026-08-11): observed live with 6 candidates — 5 icon rows drawn, scrollbar at the window's right edge; sstate0 "04/04" Broad Sword / "00/01" Mythril Knife matches the §15.26 d/e digit decode
  - R: none — picker window / row scroll / count rendering not present in godot-learning (probed godot-learning/src + tests, 2026-08-18)
  - src: `research/working_documents/ITEM_EQUIPMENT_DATA.md`

## Notes

(empty — user territory)

## Related

- [[Equip Sub Screen]]
- [[Equip And Ability Panel]]
- [[Equip Stat Delta Preview]]
- [[Formation Ability Picker]]
