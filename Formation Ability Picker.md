# Formation Ability Picker

The formation-screen "Ability" panel and its ○-opened picker are now reverse-engineered end-to-end: ○ on a slot row routes through `FUN_8011DC98` into one of three substates (Set/Remove/Learn = 0xa/0xb/0xc), a unit's five ability slots (primary/secondary skillset + Reaction/Support/Movement) live at `+0x5e` of the per-unit editable record with learned state in two MSB-first bitmasks (`+0x77`, `+0x7a`), and the candidate list is built by `FUN_801228F0` as learned-bits ANDed against a four-way slot-class id-range gate that matches the parsed `abilities.json` `ability_type` split exactly (512 records). Picker rows are name-text only (no per-row glyph); the five category icons are per-slot panel icons (RANGETILE SPRTs, CLUT 0x7DFC), and the title tab is a baked RANGETILE quad. godot-learning models the data side (equipped R/S/M slots, learned sets, type-filtered popups) but the formation picker screen itself is not yet ported.

## Points

- **○ on an Ability slot row dispatches through `world_ability_submenu_dispatcher` `FUN_8011DC98`, which sets substate `DAT_8018ba1c = row + 10`: 0xa "Set" (the slot picker `FUN_8011E630`, slot cursor `DAT_8018d187` 0..4), 0xb "Remove" (per-slot clear committing id 0; `FUN_8011E0B4`, bulk-clear helper `FUN_8011E4B4`), and 0xc "Learn" (`FUN_8011ED18` → skillset chooser `FUN_8011F090` → ability list `FUN_8011F5F0`; installs window slot 0xf, not 0xe)** — `[S·D] 2/3`
  - S: 0x8011DC98, 0x8011E630, 0x8011E0B4, 0x8011ED18 (WORLD decompiles at `project-assets/fft-rom/WORLD/functions/`, base 0x800E0000)
  - D: oracle sstate0 "Set/Remove/Learn" popup crop (`/tmp/ability_re/menu_crop.png`) and sstate1 `DAT_8018ba1c=0x0a`, `DAT_8018d187=1` (pcsx-agent live session, 2026-08-12)
  - R: none — formation Ability picker not present in godot-learning (probed godot-learning/src/ui3 + tests; only the battle-side UIActionAbilityPopup/UIPassiveAbilityPopup/UIEquipmentPopup exist)
  - src: `research/working_documents/ABILITY_PICKER.md`
- **A unit's five ability slots live at `(&DAT_801cd5ec)[unit] + 0x5e` (5×2 shorts): slot 0 = primary skillset id (job-fixed, unselectable), slot 1 = secondary skillset id, slots 2/3/4 = Reaction/Support/Movement ability ids — slots 0/1 hold small skillset ids while slots 2–4 hold ability ids (0x1A6–0x1FD); commit `world_ability_slot_commit` `FUN_801253D4` writes `+0x5e+slot*2` (0x80125408), recomputes derived stats, and with refresh≠0 re-validates equipment since a skillset change can invalidate a held item** — `[S·D·R] 3/3`
  - S: record base `DAT_801cd5ec`, commit `FUN_801253D4` @ 0x80125408 (WORLD decompiles)
  - D: live Boyce sstate1 slots `[5, 0, 0, 0x1E0, 0]` → Basic Skill / — / — / Equip Change / — (oracle, 2026-08-12)
  - R: `godot-learning/src/units/UnitProgression.gd` (equipped_reaction/support/movement, sub_job_id, `set_equipped_*`) + `godot-learning/tests/ProgressionTesterTest.gd`
  - src: `research/working_documents/ABILITY_PICKER.md`
- **Per-unit learned state is two MSB-first bitmasks plus per-skillset JP: the action-ability learned mask at `+0x77` (1 bit/ability, index `id-0x4a`, scanned by `FUN_8012257C`), the per-skillset learned masks at `+0x7a + skillset*3` (3 bytes = 24 bits per skillset; bit set by `FUN_801255E4` @ 0x80125610), and JP spent on Learn at `+0xbe + skillset*2`** — `[S·D·R] 3/3`
  - S: 0x80122594 (`FUN_8012257C` seeks +0x77), 0x801229ac (`FUN_801228F0` seeks +0x7a+ss*3), 0x80125610 (`FUN_801255E4` sets the bit) (WORLD decompiles)
  - D: live Boyce sstate1 `+0x77` = `C0 00 00` (2 learned) (oracle, 2026-08-12)
  - R: `godot-learning/src/units/UnitProgression.gd` (`learned_abilities` dict + `job_jp` — dict representation, not the ROM bitmasks) + `godot-learning/tests/ProgressionTesterTest.gd`
  - src: `research/working_documents/ABILITY_PICKER.md`
- **The ability candidate list is built by `world_ability_candidate_builder` `FUN_801228F0` into `DAT_801cd230` (short[], 0xffff-terminated — the same array the equip picker fills): skillset action ids from SCUS helper `func_0x8005a638`, ANDed with learned bits at `+0x7a+ss*3`, then gated by the slot-class id range (param3 0 → 0x001..0x1A5 Action @ 0x80122950, 1 → 0x1A6..0x1C5 Reaction @ 0x8012295c, 2 → 0x1C6..0x1E5 Support @ 0x80122968, 3 → 0x1E6..0x1FD Movement @ 0x80122974); R/S/M slots merge+sort+dedupe across every job via `FUN_80122C20`, and the picker `FUN_8011E630` branches at 0x8011E6EC to skillset logic for slot 1 vs `FUN_80122C20` for slots 2–4** — `[S·D·R] 3/3`
  - S: 0x801228F0, 0x80122950/0x8012295c/0x80122968/0x80122974, 0x80122C20, 0x8011E6EC (WORLD decompiles); `func_0x8005a638` is a SCUS helper outside the WORLD dump (inferred)
  - D: sstate1 `DAT_801cd824=1`, `DAT_801cd20c=0`, `DAT_801cd230={0x0006}` → one candidate, skillset 6 "Item" (oracle, 2026-08-12)
  - R: `godot-learning/src/ui3/UIPassiveAbilityPopup.gd` (build_rows filters by class via `AbilityDatabase.get_views_by_type(slot_type)`; no learned-bit or id-range filter) + `godot-learning/tests/PickerContractTest.gd`
  - src: `research/working_documents/ABILITY_PICKER.md`
- **The ROM's four-way classification id-range gate matches the parsed-asset `ability_type` boundaries exactly over all 512 abilities: Action kinds 0x001..0x1A5 (421 records: Normal/Item/Throwing/Jumping/Charging/Arithmetick), Reaction 0x1A6..0x1C5 (32), Support 0x1C6..0x1E5 (32), Movement 0x1E6..0x1FD (24) — so the slot-class inventory already exists in the extracted assets and needs no new extractor** — `[S·D·R] 3/3`
  - S: ROM range gates 0x80122950/0x8012295c/0x80122968/0x80122974 (WORLD decompiles) vs `godot-learning/assets/abilities/abilities.json` `ability_type` (verified over all 512 records)
  - D: live Boyce sstate1 slot[3]=0x1E0 decodes to ability 480 "Equip Change"/Support matching the left-panel text, and slot[0]=5 → skillset 5 "Basic Skill" (oracle, 2026-08-12)
  - R: `godot-learning/assets/abilities/abilities.json` + `skill_sets.json` (`actions[]` + `rsm[]`) + `assets/jobs/jobs.json` (`skill_set_id`, `innate_abilities`), consumed by `src/data/AbilityDatabase.gd` / `AbilityType.gd`; validated by `godot-learning/tests/PickerContractTest.gd` (drives the type-filtered R/S/M popup)
  - src: `research/working_documents/ABILITY_PICKER.md`
- **The ability picker rows are name-text only — no per-row category glyph: the window template at `0x8018d100` (live bytes: an op-0x12 FONT/value-stream cell plus CLUT-setup cells, terminated by op 0x1C, with no op-0x06 provider cell) feeds the shared list-row renderer `FUN_80128CE0` (dispatch `(&PTR_FUN_8018dfb8)[op]`), and the equip type-glyph LUT/provider (`0x8018D7FC`, `world_equip_type_glyph_provider 0x8011AA34`) has no ability-side analog** — `[S·D·R] 3/3`
  - S: 0x8018d100 (live template bytes), 0x80128CE0, 0x8018D7FC / 0x8011AA34 (WORLD decompiles)
  - D: 2026-08-12 oracle sstate1 prim scan + enlarged framebuffer: glove cursor immediately left of the "Item" row text, no icon between cursor and glyphs (the "small icon" first suspected in the row was the glove) (/tmp/ability_re/)
  - R: `godot-learning/src/ui3/UIPassiveAbilityPopup.gd` (single NameText column + "---" clear row; no glyph column) + `godot-learning/tests/PickerContractTest.gd`
  - src: `research/working_documents/ABILITY_PICKER.md`
- **The five category icons are per-slot panel icons on the left Ability panel (Action/Action on slots 0/1, Reaction, Support, Movement), not per-row picker icons: RANGETILE SPRT primitives (command 0x66) at cells Action (136,0,12,16), Reaction (148,0,12,16), Support (136,16,16,16), Movement (152,16,16,16), emitted CLUT 0x7DFC — correcting the doc's earlier 0x7D7C, which is the idle-bank global `DAT_801cd5e4`, not the emitted-prim CLUT; the oracle's live 5-element SPRT array sits at RAM 0x801C4214, and the installer `FUN_800ea990` is shared with the detail screen (`detail_abl_icon_descriptors 0x80155F88`)** — `[S·D] 2/3`
  - S: 0x80155F88, 0x800ea990, 0x801C4214 (WORLD decompiles / oracle RAM)
  - D: byte-confirmed live 2026-08-12 — tag-validated prim scan of oracle sstate1; rendering from those cells through their CLUT reproduces the ⚡⚡↳↻👣 glyphs pixel-for-pixel against the framebuffer (/tmp/ability_re/cellpanel.png, recon.png)
  - R: none — ability category-icon cells / CLUT 0x7DFC not present in godot-learning (probed godot-learning/src + tests)
  - src: `research/working_documents/ABILITY_PICKER.md`
- **The picker title tab ("Ability", cream) is one baked RANGETILE POLY_FT4 quad, not a FONT string: screen position (38,45), size 80×16, source uv(0,200), tpage 0x0b, idle CLUT 0x7FFC, emitted by `world_emit_textured_quad 0x8012C6A8` from `FUN_8011F5F0`+126 / twin `FUN_8011FD28` — the same "baked cream header cell" convention as the equip "Eqp."/"ALL" tabs, a distinct cell at v=200** — `[S·D] 2/3`
  - S: 0x8012C6A8, 0x8011F5F0 (WORLD decompiles)
  - D: oracle sstate1 settled-picker screenshot (2026-08-12)
  - R: none — formation Ability picker UI not present in godot-learning (the doc's port-deviation note cites `AbilityPickerMenu._build_header`; no such file exists in the repo — probed godot-learning/src + tests)
  - src: `research/working_documents/ABILITY_PICKER.md`
- **Picker row name text takes the per-row name id from `*(short*)(record*2 + unit_record + 0xbe)` (`FUN_8011EF08`) and renders it with the FONT message-stream blitter `FUN_8012A5C0 → 0x8012A0E8 → 0x80129E4C` from base `DAT_801cd8bc` (a runtime-loaded WORLD message resource, a distinct decompressed stream from the item name table); the row highlight is the glove sprite (`FUN_801282DC`), not a CLUT swap, row text glyphs scan the idle text bank 0x7C3C, and row geometry is a 16px stride with first-row base y=144 (0x8011eb90/0x8011e15c — identical to the equip picker)** — `[S·D] 2/3`
  - S: 0x8011EF08, 0x8012A5C0, 0x801282DC, 0x8011EB90 (WORLD decompiles)
  - D: oracle sstate1 settled screen shows the "Item" row with the glove cursor at the expected geometry (2026-08-12)
  - R: none — the ROM name-stream/blitter mechanism not present in godot-learning (probed godot-learning/src + tests; the port's popups render their own name-text rows via UIPassiveAbilityPopup)
  - src: `research/working_documents/ABILITY_PICKER.md`

## Notes

(empty — user territory)

## Related

- [[Formation Screen Compositing]]
- [[Ability Execution Index]]
