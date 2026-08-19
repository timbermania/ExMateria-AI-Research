# ENTD Unit Deployment Table

Binary format of `BATTLE/ENTD{1..4}.ENT`, the unit-deployment tables walked at scenario boot: a scenario's `entd_idx` (from ATTACK.OUT) selects one of 512 flat records (128 per file, 640 bytes each), each holding 16 40-byte unit-template slots whose full field layout — sprite/job/ability/item bytes, MSB-first flag bytes, AI fields, and 0xFE randomise sentinels — is now decoded and ported into a tested Godot parser. At runtime, `entd_to_roster_loader_16` in BATTLE.BIN walks the record, enqueues each unit, and the slot allocator's roster-slot choice becomes the CLUT_X seed for the unit's cinematic palette — verified bit-exact against a live capture of the chapel scenario. The slot's `special_name` field doubles as the canonical-vs-generic sourcing key.

## Points

- **BATTLE/ENTD{1..4}.ENT is a flat 512-record unit-deployment namespace split across four files (ENTD1 → entd_idx 0..127, ENTD2 → 128..255, ENTD3 → 256..383, ENTD4 → 384..511); each file is 128 records × 640 bytes = 81920 bytes and each record is 16 slots × 40 bytes.** — `[R] 1/3`
  - R: `godot-learning/tools/parse_entd.py` (concatenates the four files into one indexed dict), validated by `godot-learning/tools/test_parse_entd.py` (test_full_file_dimensions, test_record_count)
  - src: `research/key_documents/ENTD_FORMAT.md`
- **Each ENTD slot is a 40-byte EventUnit with fixed offsets: sprite_set 0x00, flags1 0x01, special_name 0x02, level 0x03, month 0x04, day 0x05, bravery 0x06, faith 0x07, prereq_job 0x08, prereq_job_level 0x09, job 0x0A, secondary_action 0x0B, reaction/support/movement u16 LE at 0x0C/0x0E/0x10, head/body/accessory 0x12-0x14, right_hand/left_hand 0x15/0x16, palette (4-bit in-battle index) 0x17, flags2 0x18 (bits 5..4 = team_color 0=Blue/1=Red/2=Green/3=LightBlue), x 0x19, y 0x1A, facing_raw 0x1B (bit 7 = upper_level, bits 0..1 = facing 0=S/1=W/2=N/3=E), experience 0x1C, skill_set 0x1D, war_trophy 0x1E, bonus_money 0x1F, unit_id 0x20 (0xFF = empty slot), target_x/target_y 0x21/0x22, ai_flags1 0x23, target 0x24, ai_flags2 0x26 (bit 2 = save_ct); level/month/day/bravery/faith/experience use 0xFE as a randomise (or "use story level") sentinel.** — `[R] 1/3`
  - R: field layout in `godot-learning/tools/parse_entd.py`, validated by `godot-learning/tools/test_parse_entd.py` (test_byte_field_mapping, test_team_color_mask)
  - src: `research/key_documents/ENTD_FORMAT.md`
- **The sprite_set byte (0x00) is the SPR table index for named units, or a generic/monster marker (0x80/0x81/0x82) that derives the sprite from job; in the chapel scenario 1, HIME=0x0C, SIMON=0x13, AGURI=0x34.** — `[D·R] 2/3`
  - D: chapel scenario 1 capture (scenario_1_captures, clut_upload_decode.md V17, 2026-06-26)
  - R: `godot-learning/tools/parse_entd.py` (sprite_set field), validated by `godot-learning/tools/test_parse_entd.py` (test_byte_field_mapping)
  - src: `research/key_documents/ENTD_FORMAT.md`
- **FFTPatcher packs ENTD flag bytes MSB-first (first flag name = bit 7, last = bit 0): flags1 = male(7) female(6) monster(5) join_after_event(4) load_formation(3) zodiac_monster(2) blank2(1) save_formation(0); flags2 = always_present(7) randomly_present(6) control(3) immortal(2); ai_flags1 = focus_unit(6) stay_near_xy(5) aggressive(4) defensive(3); ai_flags2 = save_ct(2).** — `[R] 1/3`
  - R: MSB-first decoding in `godot-learning/tools/parse_entd.py`, validated by `godot-learning/tools/test_parse_entd.py` (test_msb_flag_decoding)
  - src: `research/key_documents/ENTD_FORMAT.md`
- **prereq_job (0x08) is a job enum — 0x00 Base, 0x01 Chemist, 0x02 Knight, 0x03 Archer, 0x04 Monk, 0x05 Priest, 0x06 Wizard, 0x07 TimeMage, 0x08 Summoner, 0x09 Thief, 0x0A Mediator, 0x0B Oracle, 0x0C Geomancer, 0x0D Lancer, 0x0E Samurai, 0x0F Ninja, 0x10 Calculator, 0x11 Bard, 0x12 Dancer, 0x13 Mime — with 0xA9 as the Unknown sentinel.** — `[R] 1/3`
  - R: `godot-learning/tools/parse_entd.py`, validated by `godot-learning/tools/test_parse_entd.py` (test_unknown_prereq_job)
  - src: `research/key_documents/ENTD_FORMAT.md`
- **The parser rotates PSX (x, y, facing) 180° around the X axis at parse time (ADR-0052): y_godot = size_z − 1 − y_psx, facing S↔N swapped with E/W unchanged, upper_level bit 7 preserved; empty slots (unit_id 0xFF) are left as-is so positional indices match FFTPatcher's UI 1:1.** — `[R] 1/3`
  - R: `apply_chirality_fix_to_slot` in `godot-learning/tools/parse_entd.py`, validated by `godot-learning/tools/test_parse_entd.py` (ApplyChiralityFixToSlotTest)
  - src: `research/key_documents/ENTD_FORMAT.md`
- **At scenario boot, `entd_to_roster_loader_16` (BATTLE.BIN 0x8017f8a0) walks the ENTD record selected by current_scenario_id and, for each non-empty slot, writes slot[+0x1] = roster_id and slot[+0x161] = unit_id, then calls `evtchr_queue_enqueue` (0x8008e540) at 0x8017fb88 with a0 = roster_id and Stack[+0x18] = unit_id; the queue is drained by `evtchr_unit_queue_drain` (0x80088e04) into `evtchr_slot_allocator` (0x80083cd4), whose choice — slot 12 for unit_id 0x0C, slot 0 for 0x13, slot 1 for 0x34 — is written to slot[+0x4] as roster_slot_idx; all five functions are labeled with [VERIFIED live] anchors in fft-ghidra/content/renames_high.tsv.** — `[S·D] 2/3`
  - S: entd_to_roster_loader_16 0x8017f8a0, evtchr_queue_enqueue 0x8008e540, evtchr_unit_queue_drain 0x80088e04, evtchr_slot_allocator 0x80083cd4, labeled in `fft-ghidra/content/renames_high.tsv`
  - D: chapel scenario 1 live VRAM/unit-slot dump (clut_upload_decode.md V17, 2026-06-26) — bit-exact slot assignment
  - src: `research/key_documents/ENTD_FORMAT.md`
- **`evtchr_unit_clut_writer` (BATTLE.BIN 0x80087a28) seeds the unit's cinematic CLUT_X from the allocator-assigned roster_slot_idx (slot[+0x4]): chapel HIME lands at VRAM CLUT (192, 483), SIMON at (0, 483), AGURI at (16, 483).** — `[S·D] 2/3`
  - S: evtchr_unit_clut_writer 0x80087a28, labeled in `fft-ghidra/content/renames_high.tsv`
  - D: chapel scenario 1 live VRAM dump (clut_upload_decode.md V17, 2026-06-26) — bit-exact CLUT coordinates
  - src: `research/key_documents/ENTD_FORMAT.md`
- **The flags2 `always_present` bit (bit 7) makes a unit spawn visible immediately, and all three chapel mains (HIME, SIMON, AGURI) set it.** — `[D] 1/3`
  - D: chapel scenario 1 live capture (scenario_1_captures, clut_upload_decode.md V17, 2026-06-26)
  - src: `research/key_documents/ENTD_FORMAT.md`
  - ⚠ SUPERSEDED (2026-08-18) by: a boot unit loads held/hidden iff ENTD flags2 & 0xC0 == 0xC0 (always_present AND randomly_present) — Ovelia (0xC4) loads hidden while Agrias (0x84, always_present alone) loads visible; the flag byte is `roster+0x5` = flags2, gated in `entd_to_roster_loader_16` @0x8017f8a0 ([[Unit Visibility Flag]])
- **The sanity anchor entd_idx 256 → ENTD3 record 0 → slot 0 is Princess Ovelia: sprite_set 0x0C, unit_id 0x0C, job 0x5E (Princess), PSX position (8, 4) which is palindromic under the chirality fix (size_z 9), facing 3 (East), flags2.always_present set; the record has 10 used slots then 6 with unit_id 0xFF.** — `[R] 1/3`
  - R: `godot-learning/tools/test_parse_entd.py` (test_scenario_1_ovelia_slot_0, test_entd_256_is_ovelia)
  - src: `research/key_documents/ENTD_FORMAT.md`
- **Scenario 6's resident cinematic cast (event unit ids 131–138) is placed at scenario load from ENTD 387 — loaders `FUN_8017f388` / `entd_to_roster_loader_16` (`0x8017F8A0`) populate the records and `FUN_8017f640` writes their tiles from the ENTD X/Y bytes, presence gated by ENTD flag `+0x05 & 0xC0` (cinematic) with the marker riding `+0x161` — and it never moves across the cutscene, so it is a static backdrop, not script-spawned by event ops.** — `[S·D] 2/3`
  - S: `FUN_8017f388`, `FUN_8017f8a0`, `FUN_8017f640` (`battle_disassembly.txt`, static RE per the doc's §4.5b)
  - D: scenario 6 roster polling over 20+ taps — slots 3–10 (active 0x80/0x81, ids 131–138) unchanged (2026-07-05)
  - src: `research/working_documents/ADD_GHOST_UNIT_OPCODE_47.md`
- **`{45}` Add Unit event handler `FUN_8008cf78` (call site `0x8008d024`, dispatched from `LAB_80145734`) calls the unit-add queue `FUN_8008e540`, whose drain runs `FUN_80088e04 → FUN_80087a28 → FUN_80083cd4(unit_id)` and sets `unit[+0x0E]` — so chunk-side Add Units increment the same allocation counter as ENTD-side adds (chapel's five late-adds land at counter values 3..7, live-verified).** — `[S·D] 2/3`
  - S: `FUN_8008cf78` @ `0x8008d024`, `LAB_80145734`, `FUN_8008e540`, `FUN_80088e04`, `FUN_80087a28`, `FUN_80083cd4` (`battle_disassembly.txt`)
  - D: chapel boot allocation BP at `0x80087BB0` (8 hits s3=0..7; late-adds s3=3..7) (2026-06-28)
  - src: `research/working_documents/chapel_opcode_trace/HANDOFF_sprite_palette_resolution.md`
- **ENTD record 256 (ENTD3 record 0, the chapel record) generic-soldier slots 5–9 (uid 0x83/0x84/0x80/0x81/0x82, sprite_set 0x80/0x81 markers, jobs 0x4A/0x4C/0x4B/0x4C/0x4D) carry palette bytes 0/0/2/2/2, and the palette byte persists through sprite resolution — the Red soldiers (palette=2) render from their resolved SPR palettes (KNIGHT_M/YUMI_M/ITEM_M) at row 2.** — `[D] 1/3`
  - D: session-4 bit-match table, chapel full cast (scenario_id=4, 2026-06-28; `_clut_live.json` in the doc dir)
  - src: `research/working_documents/chapel_opcode_trace/HANDOFF_sprite_palette_resolution.md`
- **In the chapel (scenario 1) cinematic, the PSX roster-slot ↔ chunk unit-id mapping is slot 0↔0x13, 1↔0x34, 2↔0x02, 4↔0x83, 12↔0x0C — inferred from the chunk's cinematic Unit Anim writes.** — `[D] 1/3`
  - D: chapel opcode trace captures `pcsx_run.jsonl` (194 state-change rows) + `godot_run.jsonl` (2161 rows), PC↔vsync anchored on cinematic Unit Anim writes at 188 distinct vsync points (2026-06-27)
  - src: `research/working_documents/chapel_opcode_trace/report.md`
- **`evtchr_slot_allocator` @ `0x80083cd4` assigns the roster slot from the ENTD `unit_id`: direct store when `unit_id < 0x10` (`0x80083d90`), first-empty-slot scan otherwise (`0x80083db8`); roster base `0x800b7308` with stride `0x440`, so `slot[+0x4] == 0x800b730c + slot*0x440`.** — `[S] 1/3`
  - S: `evtchr_slot_allocator` `0x80083cd4`, branch sites `0x80083d90`/`0x80083db8`, roster base `0x800b7308` (BATTLE.BIN disassembly)
  - R: none — roster-slot allocator / `0x440` roster stride not present in godot-learning (probed `godot-learning/src/`, `godot-learning/tests/` for `roster_slot`/`0x440`/`slot_allocator`)
  - src: `research/working_documents/EVTCHR_CLUT_RESOLUTION.md`

- **ENTD slot `special_name` (0x02) is the unit-identity sourcing key: a value present in the ROM story-name table (`UnitNames.xml`, story ids ≈1–72) makes the slot canonical (looked up as a story `Character`); other values (factory ids 120+ / 0xFF) make it factory-generic, whose sprite derives from the `sprite_set` marker (0x80 male / 0x81 female / 0x82 monster) + job.** — `[R] 1/3`
  - R: `godot-learning/tools/data/UnitNames.xml` (ROM story-name table) + `godot-learning/src/characters/Character.gd` (FIXED provenance seeded from `UnitNames.xml`); the `special_name` resolver is pending build, not yet wired
  - src: `research/working_documents/HANDOFF_navigator_build_ready.md`
- **The in-battle unit roster array — distinct from the `0x800b7308`/`0x440` cinematic roster — lives at base ≈ `0x80190908` with a 448-byte (0x1C0) stride: each slot holds the expanded runtime unit struct, with live-confirmed offsets `+0x08` x, `+0x09` y, `+0x0A` facing (0=S, 1=W, 2=N, 3=E), `+0x0B` upper-level bit; the 10 slots active mid-cinematic hold the positions the event script has walked units to, which differ from the ENTD deploy-screen starting positions (ENTD records the deploy positions; this array reflects current positions).** — `[D] 1/3`
  - D: x/y signature scan of live RAM — 10 active slots, all field offsets confirmed across them (sstate2 + Enter, 2026-06-20; dump `scenario_1_captures/unit_roster_array_0x80190908.bin`)
  - R: none — `0x80190908` roster not present in godot-learning (probed `src/`, `tests/`)
  - src: `research/working_documents/SCENARIO_LOADING.md`

## Notes

(empty — user territory)

## Related

- [[Add Ghost Unit Opcode]]
- [[Cinematic Palette Pipeline]]
- [[Unit Visibility Flag]]
- [[EVTCHR CLUT Resolution]]
- [[Rotate Unit Interpolation]]
- [[Unit Build Pipeline]]
