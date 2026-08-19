# Equip Stat Delta Preview

The equip/compare panel's numeric before/after stat deltas are reverse-engineered end-to-end: for the item under the picker cursor the ROM runs a dry-run equip into scratch pseudo-unit `0x14` (`FUN_80124428`, skipping stock ledger + derived-stat rebuild), computes signed per-stat deltas into `equip_stats_src_picker @ 0x8018AB40` (base array `0x8018AB00`), and renders sign→colour via a software 4bpp CLUT-index-bias blitter — positive +0xC → blue sub-ramp, negative +0x8 → red sub-ramp, zero → dash — in FRAME.BIN palette 15 (CLUT 0x7FFC; [9] red, [13] blue, [1] tan). Weapon deltas show as a two-number WP/W-EV pair in the Weap.Power row; armor/accessory HP·MP deltas route to the vitals numerators. Every claim is Ghidra-address-rooted and validated on four 2026-08-12 savestate framebuffer/RAM oracles, which also confirmed the previously-unconfirmed item-stat subtraction table (FORMATION_SCREEN.md §15.26 C). In this checkout godot-learning has the data layer (items.json + ItemDatabase, FRAME palette reader in parse_fft_font.py) but the detail-scene preview panel itself is not present, despite the doc's 2026-08-13 update claiming it landed.

## Points

- **The item stat getter `FUN_80121568` @ 0x80121568 returns one stat byte for an item id by dispatching on slot category via `FUN_80125374` @ 0x80125374, a pure id-range classifier: <0x7A weapon(0), 0x7A–0x7F throwable(5), 0x80–0x8F shield(1), 0x90–0xAB head(2), 0xAC–0xCF body(3), 0xD0–0xEF accessory(4), ≥0xF0 chemist(5)** — `[S·D·R] 3/3`
  - S: 0x80121568, 0x80125374 (WORLD.BIN decompiles at `project-assets/fft-rom/WORLD/functions/*.c`; the earlier handoff's 0x8012157C is mid-function — the slot-classifier call 0x14 bytes into FUN_80121568)
  - D: sstate2 savestate (captured 2026-08-12), live RAM `getMemPtr` reads of the weapon/armor sub-tables — live ROM bytes == items.json == the values the panel displays
  - R: `godot-learning/src/data/ItemDatabase.gd` (`get_weapon_power` / `get_evade_percent` / `get_hp_bonus` / `get_mp_bonus` over `assets/items/items.json`) + generator `godot-learning/tools/extract_items.py` (offsets FFTPatcher-confirmed); validated by `godot-learning/tests/UnitWeaponBindTest.gd`
  - src: `research/working_documents/EQUIP_STAT_PREVIEW.md`
- **The item stat sub-tables live at fixed RAM addresses: item records 0x80062EBC (stride 0xC, rec[4] = second_table_id), weapon sub-table 0x80063AB8 (stride 8, +4 = weapon_power, +5 = evade%), shield sub-table 0x80063EB8 (stride 2), head/body armor sub-table 0x80063ED8 (stride 2, index (id−0x90)×2); SCUS RAM base = 0x8000F800 (file 0x536B8 → 0x80062EB8)** — `[S·D·R] 3/3`
  - S: two-step indirection in `FUN_80121568` (disasm 0x801215a8 → 0x801215bc; decomp const −0x7ff9c544 = 0x80063ABC = base+4, pointing straight at the weapon_power byte) in `world_disassembly.txt` / WORLD decompiles
  - D: sstate2 live RAM (2026-08-12): Broad Sword (id 19) WP=4/EV=5 @ 0x80063B50 (= 0x80063AB8 + 19×8), Dagger (id 1) @ 0x80063AC0, Clothes (id 186) HP=5/MP=0 @ 0x80063F2C (= 0x80063ED8 + (0xBA−0x90)×2)
  - R: `godot-learning/tools/extract_items.py` (weapon_power = data[4], evade_percent = data[5], hp/mp_bonus = armor/accessory data[0]/data[1]) + `godot-learning/assets/items/items.json` + `godot-learning/src/data/ItemDatabase.gd`; validated by `godot-learning/tests/UnitWeaponBindTest.gd`
  - src: `research/working_documents/EQUIP_STAT_PREVIEW.md`
  - ⚠ SUPERSEDED (2026-08-18) by: the item record table base is `0x80062EB8`, not `0x80062EBC` (off-by-4 base misread) — `scus_item_record 0x8005A884(id)` = `0x80062EB8 + (id&0xff)·12`, rec[0]=palette / rec[1]=graphic / rec[5]=item_type_id; RAM table byte-identical to SCUS file 0x536B8 and items.json 254/254 (ITEM_EQUIPMENT_DATA.md §1)
- **The equip dry-run/preview is a pseudo-unit: `world_equip_commit` `FUN_80124428` @ 0x80124428 with `unit == 0x14` skips the party-stock ledger adjust `FUN_801208B8` and the live derived-stat rebuild `FUN_801212B8`, but still writes `item_id & 0x3ff` into `(&DAT_801cd5ec)[0x14] + 0x54 + slot*2` (per-unit equip table, 5 slots × 2 bytes) — scratch state only, no side effects on real stock or the real unit's stats** — `[S·D] 2/3`
  - S: 0x80124428, 0x801208B8, 0x801212B8, DAT_801cd5ec + 0x54 (WORLD.BIN decompiles)
  - D: sstate1–4 savestates (captured 2026-08-12, SCUS94221.sstateN, 256×240 native framebuffer) — displayed preview deltas match the dry-run model end-to-end
  - R: none — `DetailScene` / `EquipStatDelta` not present in godot-learning (probed godot-learning/src + tests, plus smd-player, fft-sound-driver, effect-editor)
  - src: `research/working_documents/EQUIP_STAT_PREVIEW.md`
- **The delta array is `equip_stats_src_picker @ 0x8018AB40` (signed per-stat deltas) with parallel base array `equip_stats_src_base @ 0x8018AB00`; the delta computer `FUN_80123430` @ 0x80123430 (called from the picker statemachine at disasm 0x8011b3c4) loops all 5 slots — `FUN_801231CC` @ 0x801231CC resolves each item's stat contribution and `FUN_80123288` @ 0x80123288 performs the raw subtraction (`dst = base − k·preview`, k==1 ⇒ exactly `preview − base` across ~15 s16 fields)** — `[S·D] 2/3`
  - S: 0x80123430, 0x801231CC, 0x80123288, 0x8018AB40, 0x8018AB00, caller 0x8011b3c4 (WORLD.BIN decompiles / world_disassembly.txt)
  - D: sstate4 (Broad Sword equipped, cursor on Dagger, 2026-08-12): displayed `−1` = 3−4 (Dagger WP 3 − Broad Sword WP 4) proves the value is a delta, not the raw item value
  - R: none — stat-delta computation not present in godot-learning (probed godot-learning/src + tests)
  - src: `research/working_documents/EQUIP_STAT_PREVIEW.md`
- **The compare panel builder `world_equip_compare_panel_builder` `FUN_80111EC4` @ 0x80111EC4 does not compute numbers — it dereferences the pre-populated arrays through value-job tables in two dispatch modes: the delta panel (slot 0xb) → classifier `FUN_801119C0` with job table `equip_stats_value_jobs_picker @ 0x8018B588` (src_ptr → 0x8018AB40); the base panel (slot 0xc) → plain renderer `FUN_800FDF38` with job table `equip_stats_value_jobs_base @ 0x8018B4D4` (src_ptr → 0x8018AB00), no sign/colour classification** — `[S] 1/3`
  - S: 0x80111EC4, 0x801119C0, 0x800FDF38, 0x8018B588, 0x8018B4D4 (WORLD.BIN decompiles; the two overlapping panels, 2px offset, were established by FORMATION_SCREEN.md §15.26)
  - R: none — compare-panel value plumbing not present in godot-learning (probed godot-learning/src + tests)
  - src: `research/working_documents/EQUIP_STAT_PREVIEW.md`
- **The sign classifier `FUN_801119C0` @ 0x801119C0 maps each signed delta to a 32-bit colour word + format flag: negative → `0x88888888` + flag 0x800 (dash/minus), zero → `0x00000000` + 0x800 (dash), positive → `0xCCCCCCCC` + 0x400 (plus/up), passing `abs(delta)` + flags to the glyph emitter `FUN_80111BC0`** — `[S·D] 2/3`
  - S: 0x801119C0 (WORLD.BIN decompiles)
  - D: sstate2/sstate4 framebuffer colour samples (2026-08-12): positive +4/+5 = 0x4142 blue, negative −1 = 0x08AD red — the sign→colour outcome is proven numerically from the raw 256×240 buffer, not by eye
  - R: none — sign→colour preview painting not present in godot-learning (probed godot-learning/src + tests)
  - src: `research/working_documents/EQUIP_STAT_PREVIEW.md`
- **The flag→glyph emitter `FUN_80111BC0` @ 0x80111BC0 turns format flags into glyph cells of the FRAME.BIN number font (atlas `DAT_801cd830`; scratch descriptor `DAT_8018b660`, low byte selects the U cell): flag 0x800 → dash cell 0xBA, 0x400 → plus cell 0xC8, 0x8000 → kerned slash 0xB4 (the `/` in `+4 / +5`), 0x100 → dot-leader 0x78/0x1A; digits at U = 0x78 + 6×digit; the colour word is saved/zeroed/restored around the slash/dots so only the digits and the ± marker carry the colour bias** — `[S] 1/3`
  - S: 0x80111BC0, DAT_801cd830, DAT_8018b660 (WORLD.BIN decompiles)
  - R: none — glyph-cell rendering not present in godot-learning (probed godot-learning/src + tests)
  - src: `research/working_documents/EQUIP_STAT_PREVIEW.md`
- **`FUN_800FEFF0` @ 0x800FEFF0 is a software 4bpp blitter, not a GPU-prim builder — its inner loop writes only NON-ZERO source nibbles (ink) with `dest_index = (src_ink_nibble + bias) mod 16`, where the "colour word" is a per-nibble additive bias (+0xC selects CLUT sub-ramp 0xC–0xF = blue, +0x8 sub-ramp 0x8–0xB = red, +0 base) — hence a GPU-prim scan of the panel buffers finds no coloured digit prims (they are blitted, then the composited region is drawn with the panel's font CLUT 0x7ffc/0x7f7d)** — `[S·D] 2/3`
  - S: 0x800FEFF0 (WORLD.BIN decompiles); the colour word's all-0x8/all-0xC/0 nibble pattern is the +8/+12/0 bias
  - D: sstate2/sstate4 raw 256×240 framebuffer RGB555 samples (2026-08-12): positive 0x4142 (16,80,128) blue, negative 0x08AD (104,40,16) red — mechanism (code) and outcome (framebuffer) agree
  - R: none — 4bpp index-bias blitting not present in godot-learning (probed godot-learning/src + tests)
  - src: `research/working_documents/EQUIP_STAT_PREVIEW.md`
- **The delta colour palette is exactly CLUT 0x7FFC = VRAM row 511 = EVENT/FRAME.BIN palette #15 (file offset 0x91E0 = 0x9000 + 15×32; FRAME.BIN holds 22 × 16-entry RGB555 CLUTs at 0x9000), byte-identical VRAM ↔ ROM: entry [1] = tan base (48,40,32; plain numbers + zero-dash, bias +0), [9] = red (104,40,16; negative = ink 1 + bias 0x8), [13] = blue (16,80,128; positive = ink 1 + bias 0xC), with entries 2–4 / 10–12 / 14–15 as anti-alias ramps** — `[S·D·R] 3/3`
  - S: ROM trace — 32-byte signature search of FRAME.BIN@0x91E0 gives a single hit; colour-bias words 0xCCCCCCCC / 0x88888888 from 0x801119C0
  - D: live VRAM dump over HTTP `GET /api/v1/gpu/vram/raw` + framebuffer samples landing exactly on entries 9 and 13 (sstate2/sstate4, 2026-08-12)
  - R: `godot-learning/tools/parse_fft_font.py:read_frame_palette` (reads the full 16-entry palette at 0x9000 + idx×32; the port is prescribed to read [9] red, [13] blue, [1] tan)
  - src: `research/working_documents/EQUIP_STAT_PREVIEW.md`
- **The Weap.Power row packs two fields — left = weapon_power delta, right = evade% (W-EV) delta, separated by the kerned slash — while armor/accessory HP·MP deltas route to the vitals HP/MP numerators instead of the stats band (hp_bonus → HP numerator, mp_bonus → MP numerator; the vitals sink uses its own cooler font CLUT, +5 sampled 0x774C cyan, same sign→bias logic)** — `[S·D] 2/3`
  - S: slash flag 0x8000 → cell 0xB4 in `FUN_80111BC0`; classifier zero→0x800→dash fires for the unchanged AT · C-EV · S-EV · A-EV columns (WORLD.BIN decompiles)
  - D: sstate2 `+4/+5` = WP 4 / EV 5 (Broad Sword into empty R.Hand); sstate4 `-1/-` = WP −1 / EV 0-as-dash; sstate3 Clothes → HP numerator +5 in the vitals window with the stats band's Weap.Power/AT/EV columns dashed (all captured 2026-08-12)
  - R: none — equip compare panel not present in godot-learning (probed godot-learning/src + tests)
  - src: `research/working_documents/EQUIP_STAT_PREVIEW.md`
- **Per the doc's 2026-08-13 update, the port's full-band preview covers all 10 fields — `EquipStatDelta.compute` returns `{wp, wev, hp, mp, move, jump, speed, pa, s_ev, a_ev}` (pure two-item contribution deltas) and `DetailScene._build_stats_text` paints a signed delta on the Move/Jump/Speed rows and the AT · S-EV · A-EV columns, with C-EV job-only → always dashed and AT reflecting PA contribution only; guards are `EquipStatDeltaTest` (compute), `DetailStatsDeltaCoverageTest` (render readback), `FormationEquipShieldPreviewTest` (end-to-end shield → S-EV); A-EV stays 0 until items.json parses an accessory evade field** — `[ ] 0/3`
  - R: none — `EquipStatDelta` / `DetailScene` / the three named tests not present in this checkout (probed godot-learning/src, godot-learning/tests, smd-player, fft-sound-driver, effect-editor)
  - src: `research/working_documents/EQUIP_STAT_PREVIEW.md`

## Notes

(empty — user territory)

## Related

- [[Formation Screen Index]]
- [[Formation Screen Compositing]]
- [[Formation Ability Picker]]
