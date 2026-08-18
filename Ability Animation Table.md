# Ability Animation Table

FFT's Ability Animation Table (BATTLE.BIN `0x2CE10`) gives every ability a 3-byte entry governing its whole body-animation lifecycle: byte 0 selects a charging pose set through a two-level lookup into the Charging Pose Set Table (`0x2CDE8`), byte 1 (`effect_anim_id`) selects the execution-phase animation — `0x00` routing to the weapon animation dispatcher instead of a fixed SEQ — and byte 2 is the battle action text index. Charging start, sustain, and CT-wait are resolved by `FUN_80082b1c` / `FUN_80082b80` / `FUN_80082eec`, and 96 abilities (Knight Breaks, Sword Skills, Cloud Limits, Charge+, basic Attack, monster attacks, and more) carry `effect_anim_id == 0x00`.

## Points

- **The Ability Animation Table sits at BATTLE.BIN offset 0x2CE10 (RAM 0x8003CE10) with 3 bytes per ability across 512 entries (offset = ability_id × 3): byte 0 = charging_pose_set_id, byte 1 = effect_anim_id (execution-phase animation), byte 2 = battle action text index.** — `[S] 1/3`
  - S: table base 0x2CE10 / RAM 0x8003CE10, per `research/working_documents/ability_execution_animation_system.md`
  - src: `research/working_documents/ability_execution_animation_system.md`
- **Charging uses a two-level lookup: byte 0 indexes the Charging Pose Set Table (0x2CDE8, RAM 0x8003CDE8, 20 sets × 2 bytes) into (start_anim, sustain_anim), with common pose sets 0x00 Crouch (start 0x2A Casting Start, sustain 0x22 Charging Loop), 0x07 Crouch (no start, sustain 0x22), 0x0A monster attack (0x2A/0x22), and 0x01 Spell Cast (0x2A/0x2B Casting Mid).** — `[S] 1/3`
  - S: Charging Pose Set Table 0x2CDE8 / RAM 0x8003CDE8, per `research/working_documents/ability_execution_animation_system.md`
  - src: `research/working_documents/ability_execution_animation_system.md`
- **Start charging (battle state 0x1C) plays the Level 2 start_anim via FUN_80082b1c (0x80082b1c) — or the default idle animation (anim 2) when pose_set_id is 0 — and sustain charging (state 0x1E) plays the sustain_anim via FUN_80082b80 (0x80082b80), which also applies a crystal/treasure state override (anim 0x09) checked by FUN_8008278c.** — `[S] 1/3`
  - S: FUN_80082b1c (0x80082b1c), FUN_80082b80 (0x80082b80), FUN_8008278c, per `research/working_documents/ability_execution_animation_system.md`
  - src: `research/working_documents/ability_execution_animation_system.md`
- **The CT-wait idle resolver FUN_80082eec (0x80082eec) runs every frame per unit and, when the unit's status word at unit+0x140 has the Charging (0x100) or Performing (0x200) bit set, restores the ability_id from the action context (unit+0x134[0x170]) and re-runs the sustain lookup FUN_80082b80, holding the unit in its charging pose while other units act.** — `[S] 1/3`
  - S: FUN_80082eec (0x80082eec), per `research/working_documents/ability_execution_animation_system.md`
  - src: `research/working_documents/ability_execution_animation_system.md`
- **96 abilities carry effect_anim_id == 0x00 and route their execution phase through the weapon animation dispatcher — Knight Breaks 0x08A–0x091, Sword Skills 0x09B–0x0A5, Cloud Limits 0x101–0x108, Sniper/Aim 0x0D5–0x0D7, Oracle skills 0x0EA–0x0F7, Ruin skills 0x0C4–0x0C7, Charge+ 0x196–0x19D, monster attacks (0x109 ChocoAttack … 0x150 TailSwing), basic Attack 0x16F, and support/movement abilities 0x1DB–0x1F2.** — `[S] 1/3`
  - S: Ability Animation Table byte 1 values, per `research/working_documents/ability_execution_animation_system.md`
  - src: `research/working_documents/ability_execution_animation_system.md`
- **Oracle skills (0x0EA–0x0F7) and Ruin skills (0x0C4–0x0C7) pair spell-style charging (pose set 0x01, arms raised with Casting Mid sustain) with weapon-swing execution (effect_anim_id 0x00), so the execution animation depends entirely on the unit's equipped weapon — a rod produces a staff swing, a knife a knife slash.** — `[S] 1/3`
  - S: pose set 0x01 + effect_anim_id 0x00 entries, per `research/working_documents/ability_execution_animation_system.md`
  - src: `research/working_documents/ability_execution_animation_system.md`

## Notes

(empty — user territory)

## Related

- [[Weapon Animation System]]
- [[Ability Execution State Flow]]
- [[Unit Anim Opcode]]
