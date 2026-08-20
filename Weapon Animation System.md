# Weapon Animation System

During the execution phase, `effect_anim_id == 0x00` abilities get their body animation from the weapon animation table (BATTLE.BIN `0x2D364`, 19 weapon types × HIGH/MID/LOW SEQ animation IDs), dispatched by `FUN_80082c10` and selected by `FUN_8008288c` from the caster/target height difference (±0x0B threshold). The weapon-type index is stored at `unit+0x13b` by `FUN_80083758` from the equipped weapon's item record, and each attack plays three synchronized sprite layers (TYPE1 body, WEP1 weapon overlay, EFF1 projectile arc) triggered by `QueueSpriteAnim` opcodes embedded in the body animation. The body animations open with a `SetLayerPriority` (0xFFE2) pick from the 4-layer priority table: front-facing attacks use row 17, back-facing row 11, and front-facing shield blocks row 18 so the shield's WEP layer draws in front of the body. A 2026-04-16 working doc disassembles the six shield-block sequences (slots 176–181): each is SetLayerPriority → QueueSpriteAnim (WEP anim 46–51) → MoveBackward2 → a 16-tick body-frame wait (SHP frames 130–132 front / 179–181 back) → PauseAnimation holding the pose.

## Points

- **The execution-phase animation dispatcher FUN_80082c10 (0x80082c10) reads effect_anim_id from the Ability Animation Table and branches: 0x00 → height-based weapon attack via FUN_8008288c, 0x01 → distance-based Chemist item animation via FUN_80082a44, any other value → plays the SEQ animation directly; units whose sprite category (DAT_80094749 indexed by unit+0x06) is 5–7 bypass the table entirely and hardcode animation 0x2C.** — `[S] 1/3`
  - S: FUN_80082c10 (0x80082c10), FUN_8008288c, FUN_80082a44, DAT_80094749, per `research/working_documents/ability_execution_animation_system.md`
  - src: `research/working_documents/ability_execution_animation_system.md`
- **The Weapon Animation Table at BATTLE.BIN offset 0x2D364 (RAM 0x80094364) holds 19 weapon types × 3 SEQ animation IDs (HIGH/MID/LOW): FISTS 0x3D/0x3E/0x3F, melee types 1–9 (Knife through Hammer) all share the swing set 0x40/0x41/0x42, SPEAR/POLE 0x4D/0x4E/0x4F, GUN 0x50/0x51/0x52, BOW/CROSSBOW 0x53/0x54/0x55, BOOK 0x56 in all three slots, INSTRUMENT 0x57 in all three slots, and BAG/CLOTH 0x77/0x78/0x79.** — `[S] 1/3`
  - S: table base 0x2D364 / RAM 0x80094364, per `research/working_documents/ability_execution_animation_system.md`
  - ⚠ SUPERSEDED (2026-08-19) by: The values are byte-correct but the prose order is not the index order — the shipped rows are 10 GUN, 11–12 BOW/CROSSBOW, 13 INSTRUMENT (0x57), 14 BOOK (0x56), 15–16 SPEAR/POLE, 17–18 BAG/CLOTH; SPEAR is listed before GUN and BOOK before INSTRUMENT, and `unit+0x13b` *is* this index
  - src: `research/working_documents/ability_execution_animation_system.md`
- **Height selection FUN_8008288c (0x8008288c) computes the caster–target height difference and picks the HIGH slot when it is below −0x0B, the LOW slot when it is above +0x0B, and MID otherwise; monsters (unit type > 0x9A) always use MID, and the weapon-type index is read from unit+0x13b.** — `[S] 1/3`
  - S: FUN_8008288c (0x8008288c), height threshold ±0x0B, per `research/working_documents/ability_execution_animation_system.md`
  - src: `research/working_documents/ability_execution_animation_system.md`
- **unit+0x13b — the weapon type used for animation selection — is written only by FUN_80083758 (0x80083758), which reads the weapon type from byte +5 of the item data record (DAT_80062eb8 + id × 0xC); the primary call site FUN_80073eec (0x80073eec, state 0x2b setup) passes the equipped right-hand weapon from unit+0x1ab, and invalid types (≥ 0x20), item-use abilities 0x17E–0x189, and Throw Stone (0x94) fall back to 0x15 (bare hands/generic).** — `[S] 1/3`
  - S: FUN_80083758 (0x80083758), DAT_80062eb8, FUN_80073eec (0x80073eec), per `research/working_documents/ability_execution_animation_system.md`
  - src: `research/working_documents/ability_execution_animation_system.md`
- **Each weapon attack plays three synchronized sprite layers — TYPE1 (body), WEP1 (weapon overlay), EFF1 (projectile arc, ranged only) — with the TYPE1 body animation containing QueueSpriteAnim opcodes that trigger the other layers (QueueSpriteAnim(1, ptr) → WEP1, QueueSpriteAnim(2, ptr) → EFF1); whether a visible projectile appears depends on the weapon equipped at runtime, not the ability, so a Knight executing Holy Explosion with a bow equipped shows an arrow projectile.** — `[S] 1/3`
  - S: QueueSpriteAnim opcodes in TYPE1 body animations, per `research/working_documents/ability_execution_animation_system.md`
  - src: `research/working_documents/ability_execution_animation_system.md`
- **The Weapon Animation Table's 19 rows are, in index order: 0 FISTS `3d/3e/3f`, 1–9 melee `40/41/42`, 10 GUN `50/51/52`, 11–12 BOW `53/54/55`, 13 INSTRUMENT `57/57/57`, 14 BOOK `56/56/56`, 15–16 SPEAR/POLE `4d/4e/4f`, 17–18 BAG/CLOTH `77/78/79` — INSTRUMENT precedes BOOK and GUN precedes SPEAR, which matters because `unit+0x13b` is the row index, not a name.** — `[S] 1/3`
  - S: `BATTLE.BIN+0x2d364` dumped whole off the US retail disc image; the index labels are checked against the item table's own byte 5 rather than against a name list — type 10 is Romanda Gun / Mythril Gun / Stone Gun (GUN), 11 is Bow Gun / Night Killer / Cross Bow (crossbow) and 12 is Long Bow / Silver Bow / Ice Bow (longbow), 13 is Ramia Harp / Bloody Strings / Fairy Harp (INSTRUMENT), 14 is Battle Dict / Monster Dict / Papyrus Plate (BOOK), 15 is Javelin / Spear / Mythril Spear and 16 is Cypress Rod / Battle Bamboo / Musk Rod (SPEAR/POLE), 17 is C Bag / FS Bag / P Bag and 18 is Persia / Cashmere / Ryozan Silk (BAG/CLOTH); the join is reproducible from the disc alone — group every item record by its byte 5 and read the names off the item-name block (2026-08-19)
  - src: external contribution — web-psx `docs/combat-tables.md` [combat.animation] (see [[Web-psx Cross-Validation]])

- **The swing/bow/shield-block SEQ families select their 4-layer draw order with 0xFFE2 SetLayerPriority at animation start: front-facing swings (TYPE1 slots 128/130/132) and front-facing bow/crossbow attacks (slots 166/168/170) set row 17, all back-facing variants (slots 129/131/133, 167/169/171, 177/179/181) set row 11, and front-facing shield blocks (slots 176/178/180) set row 18 — raw rows at BATTLE.BIN+0x2D548 (16-byte row stride, RAM DAT_80094548 + row × 0x10): 11 = [3,0,2,1], 17 = [3,2,0,1], 18 = [1,2,3,0] — which in the established token naming (0=body, 1=weapon, 2=effect, 3=text) reads row 17 as text > effect > body > weapon, row 11 as text > body > effect > weapon, and row 18 as weapon > effect > text > body, i.e. the shield's WEP layer in front of the body for blocks but behind it for attacks.** — `[S·R] 2/3`
  - S: first instructions of TYPE1.SEQ slots 128–133/166–171/176–181 (file offsets 0x0B42–0x1223; raw `FF E2 11`/`FF E2 0B`/`FF E2 12`, slot 133 sets 11 after its 0xFFF2) at project-assets/fft-extract/BATTLE/TYPE1.SEQ (MD5 d996e31554dd5b2da46d24d1680aec44) + BATTLE.BIN rows 11/17/18 @ 0x2D5F8/0x2D658/0x2D668 (project-assets dump, 2026-08-20), per the doc
  - S: rows 11/18 as raw dwords at RAM 0x800945F8 (00000003 00000000 00000002 00000001) and 0x80094668 (00000001 00000002 00000003 00000000), per `research/working_documents/shield_block_seq_shp_priority.md`
  - R: `godot-learning/tools/parse_seq.py` → `type1_seq.json` (slot 128/130/166 → SetLayerPriority 17, 133 → 11, 176 → 18, 177 → 11) + `tools/parse_layer_priority.py` → `assets/sprites/layer_priority.json` (rows 11/17/18 byte-match BATTLE.BIN) + `src/animation/UnitDisplay.gd` `_apply_variant_layer_priority` (uploads the active row as the `priority` shader param) — no dedicated test
  - src: `research/working_documents/seq_shp_priority_analysis.md`

- **TYPE1.SEQ slots 176–181 (0xB0–0xB5) hold the six shield-block sequences — High/Mid/Low × Front/Back, pointer values 0x0DDC/0x0DE9/0x0DF6/0x0E03/0x0E10/0x0E1D at file offsets 0x11E2–0x1223 — each a five-instruction body stream: SetLayerPriority (18 front / 11 back) → QueueSpriteAnim (WEP target 1, anim 46–51: front 46/48/50, back 47/49/51) → MoveBackward2 (0xFFCD) → frame-wait on body SHP frames 130–132 (front) / 179–181 (back), 16-tick wait → PauseAnimation (0xFFFF) holding the block pose.** — `[S·R] 2/3`
  - S: TYPE1.SEQ pointer-table entries 176–181 + per-slot bytecode disassembly at 0x0DDC–0x0E28, per `research/working_documents/shield_block_seq_shp_priority.md`
  - S: body frames 130–132 at TYPE1.SHP pointer-table offsets 0x0248/0x024C/0x0250 → frame data 0x07FC/0x080E/0x0820 (headers 0xF5 0xE0 / 0xEF 0xE3 / 0xED 0xE6; 6/8/6 pieces; rotation index 30/29/29 = 0.0°), per the doc
  - R: `godot-learning/tools/parse_seq.py` → `type1_seq.json` (slots 176–181 parsed via the 0x0406-base pointer table; loaded by `src/animation/AnimationDatabase.gd`) — no dedicated test
  - src: `research/working_documents/shield_block_seq_shp_priority.md`

## Notes

(empty — user territory)

## Related

- [[Ability Animation Table]]
- [[Ability Execution State Flow]]
- [[Unit Anim Opcode]]
- [[Unit Sprite Height Table]]
- [[Web-psx Cross-Validation]]
- [[Unit Sprite SEQ Opcodes]]
- [[Damage Number Popup System]]
- [[SEQ Movement Opcodes]]
- [[Unit Sprite Render Pipeline]]
