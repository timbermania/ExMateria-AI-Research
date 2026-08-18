# Weapon Animation System

During the execution phase, `effect_anim_id == 0x00` abilities get their body animation from the weapon animation table (BATTLE.BIN `0x2D364`, 19 weapon types × HIGH/MID/LOW SEQ animation IDs), dispatched by `FUN_80082c10` and selected by `FUN_8008288c` from the caster/target height difference (±0x0B threshold). The weapon-type index is stored at `unit+0x13b` by `FUN_80083758` from the equipped weapon's item record, and each attack plays three synchronized sprite layers (TYPE1 body, WEP1 weapon overlay, EFF1 projectile arc) triggered by `QueueSpriteAnim` opcodes embedded in the body animation.

## Points

- **The execution-phase animation dispatcher FUN_80082c10 (0x80082c10) reads effect_anim_id from the Ability Animation Table and branches: 0x00 → height-based weapon attack via FUN_8008288c, 0x01 → distance-based Chemist item animation via FUN_80082a44, any other value → plays the SEQ animation directly; units whose sprite category (DAT_80094749 indexed by unit+0x06) is 5–7 bypass the table entirely and hardcode animation 0x2C.** — `[S] 1/3`
  - S: FUN_80082c10 (0x80082c10), FUN_8008288c, FUN_80082a44, DAT_80094749, per `research/working_documents/ability_execution_animation_system.md`
  - src: `research/working_documents/ability_execution_animation_system.md`
- **The Weapon Animation Table at BATTLE.BIN offset 0x2D364 (RAM 0x80094364) holds 19 weapon types × 3 SEQ animation IDs (HIGH/MID/LOW): FISTS 0x3D/0x3E/0x3F, melee types 1–9 (Knife through Hammer) all share the swing set 0x40/0x41/0x42, SPEAR/POLE 0x4D/0x4E/0x4F, GUN 0x50/0x51/0x52, BOW/CROSSBOW 0x53/0x54/0x55, BOOK 0x56 in all three slots, INSTRUMENT 0x57 in all three slots, and BAG/CLOTH 0x77/0x78/0x79.** — `[S] 1/3`
  - S: table base 0x2D364 / RAM 0x80094364, per `research/working_documents/ability_execution_animation_system.md`
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

## Notes

(empty — user territory)

## Related

- [[Ability Animation Table]]
- [[Ability Execution State Flow]]
- [[Unit Anim Opcode]]
- [[Unit Sprite Height Table]]
