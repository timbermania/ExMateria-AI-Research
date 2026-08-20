# Damage Number Popup System

The PSX floating damage number is a per-unit sprite popup, not a menu-font glyph: combat writes the amount into the unit's pending-popup array (`pp+0x190`, `pp = *(unit+0x134)`), the flag translator `FUN_800803e4` arms the kind bit (`pp+0x1b1 & 0x80` → `unit+0x1b8` bit 0x1, the default type-0 popup), and the master per-unit renderer `FUN_800810a4` draws three 8×16 digit quads from a shared right-edge anchor over the target unit. The whole ~61-frame lifecycle is driven by a per-unit phase counter (`unit+0x2c2`, +1/frame via `DAT_80045980=1`): grow = per-digit Q12 matrix scale from a ROM ramp table (0.30→…→1.50 overshoot→…→1.00 settle) whose 1/7/12-phase per-digit stagger is the ones-first right-to-left reveal; fade = the primitive switching opaque→additive at phase 0x32 while the CLUT-curve engine (`0x8008f710`) ramps the number's single-pass palette (VRAM `0x7d7c`: idx1 dark outline, idx2 white fill) to black at −31/channel/frame. The EXP/JP popup is a separate glyph/text renderer, and the Godot port reproduces the whole trending phase-driven from the parsed `number_popup_trend.json` asset.

## Points

- **The floating damage number is drawn as textured sprite quads (POLY_FT4) from a VRAM font page through the general sprite/OT path — the menu/glyph number routines (`0x8014bd88`, `0x8014aec0`, `0x8014ac30`, `0x8014ab58`) are never called during the popup.** — `[D] 1/3`
  - D: round-2 live capture — stepping the static 777 frame (save-6 state) fired zero calls to any of the four number routines; framebuffer/VRAM proof `/tmp/dmg777_fb.png`, `/tmp/dmg777_vram.png` (2026-07-24)
  - src: `research/working_documents/DAMAGE_NUMBER_DISPLAY_INVESTIGATION.md`
- **The per-unit popup descriptor is a pooled sub-struct at `unit+0x2b8`: `+0x2bc` active flag, `+0x2be` type selector, `+0x2c0` display value, `+0x2c4/2c8/2cc` three per-digit record pointers (22-byte records spaced 0x16, +0x06 CLUT, +0x0E placement stepping 7px per digit), `+0x2c2` phase counter — with the kind flag word at `unit+0x1b8`, the pending-popup pointer at `unit+0x134`, and the queue count at `unit+0x1b7`.** — `[S·D] 2/3`
  - S: `lw a1,0x134(s6) @0x8007f62c`, `lw v1,0x1b8(s6) @0x8007f628`, `addiu s4,s6,0x2bc @0x8007f638`, `sh v0,0x4(s4) @0x8007f8e4`, per `battle_disassembly.txt`
  - D: live save-6 descriptor dump @0x800BA480 (unit base 0x800BA1C8, i.e. unit+0x2B8); 22-byte per-digit records @0x800BA596/5AC/5C2 (2026-07-24)
  - src: `research/working_documents/DAMAGE_NUMBER_DISPLAY_INVESTIGATION.md`
- **The damage amount travels a two-hop handoff: the damage calc writes it into the pending-popup value array at `pp+0x190` (u16 slots at computed offset `pp+kind*2`; static writer `FUN_8018df0c`), `FUN_8007f5f8` (the tail call of `FUN_800803e4`) copies it into the global value table `DAT_800962d4`, and the per-bit dispatch stamps `DAT_800962d4[bit*2]` into the descriptor value `unit+0x2c0` — re-copying every frame while the popup is active.** — `[S·D] 2/3`
  - S: `lhu v0,0x190(a1) @0x8007f63c`, `sh v0,DAT_800962d4 @0x8007f644` (in `FUN_8007f5f8 @0x8007f5f8`, tail-called from `FUN_800803e4` at `0x800808a0`), `lhu v0,0x62d4(at) @0x8007f8dc`, `sh v0,0x4(s4) @0x8007f8e4`, per `battle_disassembly.txt`
  - D: live write catch — value-table [+00]=777 (cyc 112543936815), descriptor +0x2C0=777, writes carried s6=0x800BA1C8, s4=0x800BA484 (s6+0x2bc), ra=0x800808A8 (2026-07-24)
  - src: `research/working_documents/DAMAGE_NUMBER_DISPLAY_INVESTIGATION.md`
- **The cream damage number is the default popup kind: the type selector `+0x2be` reads 0x0000 every frame of the popup's life, so it renders through the `FUN_800810a4` phase path and takes no 0x40–0x100 type code.** — `[D] 1/3`
  - D: pacer-sampled the 777 descriptor across its full 60-frame life — `+0x2be`=0x0000, `+0x2c0`=0x0309, `+0x2bc`=1 throughout (2026-07-24)
  - src: `research/working_documents/DAMAGE_NUMBER_DISPLAY_INVESTIGATION.md`
- **The pending-struct byte `pp+0x1b1` is the primary popup-kind byte: bit 0x80 arms the HP-damage number (`unit+0x1b8` bit 0x1), the other high bits arm sibling numeric popups, and bit 0x08 diverts into the status-effect icon path (icon-glyph ids from `DAT_80093d18`); `FUN_800803e4` translates the pending flag bytes into the `unit+0x1b8` bit field, full flag→bit table decoded.** — `[S·D] 2/3`
  - S: flag→bit table inside `FUN_800803e4` (0x800803e4–0x800808b4), per `battle_disassembly.txt`
  - D: R5-D1 live — the 777 popup armed via bit 0x1, type 0 (2026-07-24)
  - src: `research/working_documents/DAMAGE_NUMBER_DISPLAY_INVESTIGATION.md`
- **The HIGH-bit popup family is rendered by the `FUN_800808b8` sub-renderer, invoked from inside the master `FUN_800810a4` draw (not a separate per-frame task): `+0x1b8` bits 0x02000000/0x04000000 → types 0x40/0x50 with byte values from `pp+0x1b0`/`pp+0x1b1` (armed by `FUN_80080f44`), bits 0x08000000/0x10000000/0x20000000 → types 0xe0/0xf0/0x100 as word glyphs (armed by `FUN_80080fec` kinds 1/2/3), with the value split into ≤4 digits via the ÷10/÷1000 reciprocals.** — `[S] 1/3`
  - S: `FUN_800808b8` type table + value stamping; arming helpers `FUN_80080f44`, `FUN_80080fec`, per `battle_disassembly.txt`
  - src: `research/working_documents/DAMAGE_NUMBER_DISPLAY_INVESTIGATION.md`
- **The digit record's CLUT field (+0x06) is computed, not a literal: the unit's base VRAM CLUT `*(unit+0x10)` bumped by 0x100 to the number-font palette row, stamped once at slot construction — which is why no 0x79CB immediate exists in the ROM.** — `[S·D] 2/3`
  - S: `lhu v0,0x10(s2); addiu v0,v0,0x100; sh v0,0x6(record) @0x80087d88–0x80087d9c`, per `battle_disassembly.txt`
  - D: live digit records all carry CLUT 0x79CB → VRAM (x=176, y=487), identical across the three digits (2026-07-24)
  - src: `research/working_documents/DAMAGE_NUMBER_DISPLAY_INVESTIGATION.md`
- **The on-screen damage number is ONE textured pass: every colour in the steady-frame capture maps to VRAM CLUT `0x7d7c` (idx0 transparent, idx1 dark outline (41,41,33), idx2 white fill (239,239,239), idx3/5/6 cool-grey AA edges), so the dark border is the glyph's own baked-in idx1 — there is no separate shadow sprite.** — `[D·R] 2/3`
  - D: live VRAM read of CLUT 0x7d7c + pixel-exact mapping of the `dmg_steady.png` capture, `battle_wizard_melee_777_number_onscreen` save state (2026-07-25)
  - R: `godot-learning/tools/parse_range_tiles.py` (`DIGIT_CLUT_BGR555`, byte-exact fixture + unit test) → `RANGETILE.json` `digits.colors` → `RangeTileAtlas.digit_palette_texture()` → `DamageNumber3D` `palette_tex` (use_palette=true)
  - src: `research/working_documents/DAMAGE_NUMBER_DISPLAY_INVESTIGATION.md`
- **FFT has essentially one battle number font: the damage-popup digit art and the vitals HP/MP/CT digit art are the same glyphs (identical topology, differing only in index→CLUT convention — RANGETILE outline=idx1/fill=idx2..6, FRAMEFONT outline=idx4/fill=idx1..3), drawn through different CLUTs per context — vitals menu-grey 0x7CBC, damage cream 0x7D7C — while the EXP gold is a Gouraud tint via `FUN_800810b8` (RGB 0x96,0x8C,0x14), not a CLUT.** — `[S·D] 2/3`
  - S: `FUN_800810b8` (EXP gold Gouraud tint), per `battle_disassembly.txt`
  - D: RANGETILE '7' vs FRAMEFONT '7' bitmap comparison + live VRAM CLUT reads (2026-07-25)
  - src: `research/working_documents/DAMAGE_NUMBER_DISPLAY_INVESTIGATION.md`
- **The vitals panel numbers are drawn by `draw_number_small` (0x8014aec0, 6×10 glyph), the "01/12" unit-count by `draw_number_big` (0x8014ac30, 8×16 glyph), and the panel text labels by the event glyph decoder (0x8014bd88) — a distinct number-font system from the sprite-based damage popup.** — `[S·D] 2/3`
  - S: `draw_number_small 0x8014aec0` (digit U = digit*6+0x78, V=16, 6×10) / `draw_number_big 0x8014ac30` (pitch 8, 8×16) / `event_text_glyph_decoder 0x8014bd88`, per `renames_high.tsv` + `battle_disassembly.txt`
  - D: round-1 Exec-BP logs — small font `a0`=762(HP)/999(MP)/100(CT)/99(Lv)/63(Exp)/73(Brave)/50(Faith), big font `a0`=1/12 (unit count) (2026-07-24)
  - src: `research/working_documents/DAMAGE_NUMBER_DISPLAY_INVESTIGATION.md`
- **The entire popup lifecycle is driven by a single per-unit u16 phase counter at `unit+0x2c2`, incremented every frame by global `DAT_80045980` (=1): grow phase 0x00–0x14, steady 0x15–0x31, fade 0x32–0x3c, teardown ≥0x3d where the active flag `+0x2bc` clears — ≈61 frames ≈ 1.0 s @60 Hz.** — `[S·D·R] 3/3`
  - S: branch ladder 0x800815f4–0x80081828; `sb zero,0x2bc(s1) @0x80081828`; per-frame accumulator `@0x80081934–0x80081948` (phase += DAT_80045980), per `battle_disassembly.txt`
  - D: 73-frame pacer log — phase steps 0x01→0x3C by +1/frame with act=1, frame 61 phase=0x3D and act flips 1→0 (2026-07-24)
  - R: `godot-learning/src/ui3/elements/NumberPopupTrend.gd` + `DamageNumber3D.gd` (integer `_phase` +1 per rendered frame at 60 Hz), validated by `DamageNumberTrendTest`
  - src: `research/working_documents/DAMAGE_NUMBER_DISPLAY_INVESTIGATION.md`
- **The grow is a per-digit uniform matrix scale (`SUB_80042b1c`, a1=a2=a3=scale) driven by a ROM Q12 scale ramp (BATTLE.BIN 0x80067c00–0x80067cb4) shaped 0.30→0.60→0.90→1.20→1.50 overshoot→1.20→0.90→0.95→1.00 settle, stored as three per-digit rows (0x80067c30/0x80067c5c/0x80067c88) whose leading-zero padding staggers the digits 1/7/12 phases apart — that stagger IS the ones-first right-to-left reveal, and the per-digit records stay byte-static during display (scale applied at render time).** — `[S·D·R] 3/3`
  - ⚠ SUPERSEDED (2026-08-19) by: The table starts at **`BATTLE.BIN+0xbac`** (`0x80067bac`), 84 bytes before the window used here, and it is **6 rows of 22 halfwords at stride `0x2c`** in **2** families rather than 3 rows in 1 — so the rows at `c30`/`c5c`/`c88` are family 2 only, and starting at `c00` lands inside row 2 and misses family 1 (`bac`/`bd8`/`c04`) entirely. Family 2's curve and its 1/7/12 stagger are as described; family 1 is a **sine quarter-wave that settles through a dip** — 0.259, 0.500, 0.707, 0.873, 0.966, 1.000, 0.875, 0.875, 1.000 — staggered **1/7/13**. Read the halfwords rather than the decimals: family 2 ships `04cc 0999 0e65 1331 1800 1331 0e65 0f3c 1000`, so 0.900 is `0e65` and not `0e66`, 1.200 is `1331` and not `1333`, and the ninth value is `0f3c` = 0.9521
  - S: copy loops 0x80081464–0x800815c8 from ROM constants (DAT_80067c5c, DAT_80067c88); grow branch `FUN_800810a4 @0x800815f4` (table read 0x800812a8–0x800815c8), per `battle_disassembly.txt`
  - D: phase-triggered framebuffer triptych — `dmg_grow.png` @ phase 0x08 (tiny/forming), `dmg_steady.png` @ 0x28 (full cream 777), `dmg_fade.png` @ 0x37 (dimmed to near-invisible); record byte-identical frames 13→253 (2026-07-24)
  - R: `godot-learning/tools/parse_number_popup.py` → `assets/sprites/number_popup_trend.json` (byte-verify — raises if BATTLE.BIN doesn't reproduce the ramp); `test_parse_number_popup.py` pins the curve + R→L stagger {ones:1, tens:7, hundreds:12}; `NumberPopupTrend.gd` reproduces the asset's `scale_by_phase_q12` byte-for-byte per `NumberPopupTrendTest`
  - src: `research/working_documents/DAMAGE_NUMBER_DISPLAY_INVESTIGATION.md`
- **The three digits are laid out as offsets (8×16 cells, 7px pitch, common baseline) from a single shared anchor at the number's right edge, and each digit's offset+size is scaled about that anchor by its own phase factor — at scale 0 all digits collapse onto the same point, so the number grows and reveals outward-LEFT from the anchor, which sits over the target unit's projected screen position.** — `[S·D·R] 3/3`
  - S: grow branch `FUN_800810a4 @0x800815f4` transforms per-digit records `unit+0x2c4/2c8/2cc` via `SUB_80044a60` before `poly_ft4_packet_builder @0x8007af44` emits the quads, per `battle_disassembly.txt`
  - D: phase-0 packet capture — all three quads degenerate to w=h=0 at (272,114); steady geometry hundreds/tens/ones at x251/258/265, common baseline y130–146 (2026-07-24)
  - R: `godot-learning/src/ui3/elements/DamageNumber3D.gd` — each digit a `MeshInstance3D` at the node origin (the shared right-edge anchor), `inst.scale=(s,s,1)` per phase; `DamageNumberTrendTest` asserts anchor collapse + R→L reveal
  - src: `research/working_documents/DAMAGE_NUMBER_DISPLAY_INVESTIGATION.md`
- **The fade is two mechanisms together: at phase 0x32 the primitive switches opaque GP0 0x2C → semi-transparent additive 0x2E (tpage ABR bit 0x1F→0x3F, blend B+F), while in parallel `color_tint_blend_apply` (0x8008f710, the CLUT-curve engine shared with prayer darkening) ramps the number's palette RGB toward black at −0x1f/channel/frame — additive blend + palette→black means the digit adds nothing to the framebuffer, a genuine fade-to-nothing.** — `[S·D·R] 3/3`
  - S: `0x8008f710` walks an RGB byte table based at `DAT_80099676` (target black (0,0,0)), per `battle_disassembly.txt`
  - D: live POLY_FT4 packet decode across bands (byte-exact) — grow 0x08 / steady 0x28: code 0x2C, tpage 0x001F, ABR 0; fade 0x37: code 0x2E, tpage 0x003F, ABR 1 = B+F additive (2026-07-24)
  - R: `DamageNumber3D.gd` swaps digit materials at phase 0x32 from `feedback_hud_sprite.gdshader` (opaque) → `feedback_hud_sprite_additive.gdshader` (additive) + `brightness` 1→0; `DamageNumberTrendTest` asserts the additive shader swap
  - src: `research/working_documents/DAMAGE_NUMBER_DISPLAY_INVESTIGATION.md`
- **The yellow EXP/JP gain popup is a separate glyph/text renderer that does not route through the per-unit sprite popup system — the `unit+0x2b8` descriptor and the global value table `DAT_800962d4` stay quiet during it, so `FUN_800803e4`'s kind table enumerates damage + status popups only.** — `[D] 1/3`
  - D: R5-D3 live test — loaded slot 2 (`post_damage_pre_xp`), armed `DAT_800962d4` watchpoint + unit-region descriptor scan across ~9 s resume: no fresh value-table write, no active popup descriptor anywhere (2026-07-24)
  - src: `research/working_documents/DAMAGE_NUMBER_DISPLAY_INVESTIGATION.md`
- **The damage number is a SEPARATE per-unit draw — `FUN_800810a4` called @0x800866e8 with its own descriptor `unit+0x2b8` via `sprite_buffer_write_piece`/POLY_FT4 — a sibling of the 4-layer `unit_layer_ordering_table` composite loop (0x800867d0, `SUB_80044a60`, sub-structs at `unit+(v0-1)*0x30`); it is unit-attached, not a standalone screen popup.** — `[S·D] 2/3`
  - S: call site 0x800866e8 (per-unit sprite render dispatch ~0x80086640) vs the 4-layer composite loop 0x800867d0–0x8008680c, per `battle_disassembly.txt`
  - D: the standalone-popup model (the `0x801396a0` type-state-machine path) fired zero times in the 777 replay (2026-07-24)
  - src: `research/working_documents/DAMAGE_NUMBER_DISPLAY_INVESTIGATION.md`
- **The per-unit sprite composite draws 4 ordered layers (back→front) from the layer-priority table at RAM 0x80094548 (BATTLE.BIN 0x2D548) — 24 entries × 4 slots, FFHacktics-named 0=unit body, 1=weapon, 2=effect, 3=status/damage text graphic — and the entry index is what the SEQ `SetLayerPriority` opcode (0xFFE2) sets; the engine reads slot values in the loop at 0x800867d0 and skips zero slots.** — `[S·R] 2/3`
  - S: `t2 = 0x80094548` (Ghidra label `unit_layer_ordering_table`), composite loop 0x800867d0–0x8008680c, per `battle_disassembly.txt`
  - R: `godot-learning/tools/parse_layer_priority.py` → `assets/sprites/layer_priority.json` (24×4); `SpriteLayerManager.gd` implements only slots 0–2 (TYPE1/WEP1/EFF1) and drops token 3
  - ⚠ SUPERSEDED (2026-08-19) by: The 4 slots are 32-bit words, so a row is 16 bytes and the table spans 384 bytes — a parser using a 4-byte row stride only covers the first 6 rows and returns aliased garbage for rows 6..23; token 3 is the damage-text track this note's own popup points prove is a real layer
  - src: `research/working_documents/DAMAGE_NUMBER_DISPLAY_INVESTIGATION.md`
- **The game-loop pacer `FUN_80093a98` fires exactly once per rendered frame (waits vblank at `kernel_vblank_wait @0x8001DBA8`, then PutDispEnv-swaps the double buffer; buffer index `_DAT_8004597C`) — the cool 1/frame breakpoint used for all per-frame sampling of the popup.** — `[S·D] 2/3`
  - S: `FUN_80093a98`, `kernel_vblank_wait @0x8001DBA8`, `_DAT_8004597C`, per `battle_disassembly.txt`
  - D: Exec BP @ 0x80093A98 pacer used for the phase logs and packet captures, rounds 3–7 (2026-07-24/25)
  - src: `research/working_documents/DAMAGE_NUMBER_DISPLAY_INVESTIGATION.md`
- **The layer-priority table is 24 rows of 4 `u32` — `row = 0x80094548 + u16[actor + 0x14] * 16` — and it is stored front-first, so a painter's-order consumer reverses it.** — `[S] 1/3`
  - S: `BATTLE.BIN+0x2D548` dumped whole off the US retail disc image. At a 16-byte stride every row is a clean permutation of 0..3 as little-endian words — row 0 `00 00 00 00 | 01 00 00 00 | 02 00 00 00 | 03 00 00 00`, row 1 `0,1,3,2`, row 2 `0,2,1,3`, row 6 `1,0,2,3` — while at a 4-byte stride the same bytes read as `0,1,2,3,0,1,3,2,…`, which is the first six rows' low bytes repeating. The indexing site is `build_one_actor_sprite` at `BATTLE.BIN+1f640` (2026-08-19)
  - src: external contribution — web-psx `tools/combatdata.ts` `readTrackOrder` (see [[Web-psx Cross-Validation]])

- **The scale-in tables are 6 rows of 22 halfwords at `BATTLE.BIN+0xbac`, in 2 curve families of 3 rows each, and the ladder constant `0x15` is the row length rather than a magic number.** — `[S·D] 2/3`
  - S: `sub_800810a4` opens with 6 inlined `memcpy`s whose source bases are the whole answer — `80067bac`, `bd8`, `c04`, `c30`, `c5c`, `c88`, stride `0x2c` = 44 bytes, of which 42 are copied and the 22nd halfword of each row is a `0000` no copy reaches. Q12, `0x1000` = 1.0. Family 1 (rows 0–2) is a sine quarter-wave settling through a dip, delays 1/7/13; family 2 (rows 3–5) is the overshoot to 1.5 and back, delays 1/7/12. 21 halfwords of each 22 are copied, indices 0..20, and `0x15` is 21 — one past the end. Both families hold at 1.0 from the end of their ramp, except the *last* row of family 1, whose ramp is cut off by the row boundary at 0.875 and never reaches 1.0
  - D: watched arriving over a warped auto-battle — the phase runs 2 to `0x3c` in steps of 2, one step every 2 emulated frames, so a number's life is 60 frames of screen time; the 3 rect pointers are 22 bytes apart (the bank's first 3 records, contiguous); **7 px per digit right-to-left with the glyph at `u = 0xa8 + 8·digit`** (the figure 110 reads ones 0 → `a8`, tens 1 → `b0`, hundreds 1 → `b0`); a **leading zero is suppressed by a zero-width rectangle** rather than by a count; and the blend turns on at exactly phase `0x32`, rect 0's flag byte going `00` → `01` and the record's byte at `+4` going `1f` → `3f`, with nothing else in the 22 bytes moving for the whole episode. The figure itself is 3 digits clamped at 999 (web-psx `docs/scene-viewport.md` [viewport.number.ramp]) (2026-08-19)
  - src: external contribution — web-psx `docs/scene-viewport.md` [viewport.number.ramp] (see [[Web-psx Cross-Validation]])

## Notes

(empty — user territory)

## Related

- [[Ability Execution State Flow]]
- [[Combat Color Appliers]]
- [[Ordering Table & AddPrim]]
- [[PSX GPU Primitives]]
- [[Web-psx Cross-Validation]]
