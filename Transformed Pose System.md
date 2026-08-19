# Transformed Pose System

The `transformed` action flag (0x40) on an effect keyframe makes the target unit play a context-aware reaction pose: Cure produces a hands-raised healing pose, Fire/damage a damaged/stunned pose. The pose is not determined by the effect file — it is selected by the **ability being cast**: `start_unit_transformed_pose` routes through `FUN_80083978(caster, target)`, which reads the ability data structure at `unit+0x134` and switches on the reaction-type byte at `ability+0x18E`. The godot-learning side implements the same 0x40 flag as `ACTION_FLAG_ABILITY_REACT` and routes the reaction pose through a per-ability `target_reaction_type` field plus a reaction-animation database, but it does not implement the ROM's `FUN_80083978` switch itself.

## Points

- **The `transformed` action flag (0x40) reaction pose is determined by the ability being cast, not by the effect file — Cure plays a hands-raised healing pose at frame 2, while Fire/Damage plays a damaged/stunned pose.** — `[S·R] 2/3`
  - S: `start_unit_transformed_pose` → `FUN_80083978` (BATTLE.BIN), per `research/working_documents/TRANSFORMED_POSE_MYSTERY.md`
  - R: `godot-learning/src/effects/EffectInstance.gd` (`ACTION_FLAG_ABILITY_REACT = 0x0040`, bit 6) + `godot-learning/src/gpu/CombatLoop.gd` (`_on_ability_react` selects the pose from the ability's `target_reaction_type`) + `godot-learning/src/effects/EffectManager.gd` — no test named for the 0x40 flag; adjacent reaction path validated by `godot-learning/tests/GPUEvasionMeleeHitTest.gd`
  - src: `research/working_documents/TRANSFORMED_POSE_MYSTERY.md`
- **`start_unit_transformed_pose(unit_id)` (0x800839XX, exact address still to pin down) routes into `FUN_80083978(caster_unit, target_unit)`; the caster is looked up via `FUN_8007a1d4` (uses `DAT_8009611c`), the target via `FUN_8007a6e4` (unit lookup by ID), and the animation is actually triggered by `FUN_80081988`, with `FUN_80081978` as the alternative trigger.** — `[S] 1/3`
  - S: BATTLE.BIN symbols `FUN_80083978`, `FUN_8007a1d4`, `FUN_8007a6e4`, `FUN_80081988`, `FUN_80081978`, `DAT_8009611c`, per `research/working_documents/TRANSFORMED_POSE_MYSTERY.md`
  - R: none — `start_unit_transformed_pose` / `FUN_80083978` not present in godot-learning (probed godot-learning/src, godot-learning/tests)
  - src: `research/working_documents/TRANSFORMED_POSE_MYSTERY.md`
- **The pose logic reads the ability data structure pointed to by `unit+0x134` (pointer to the current ability/action data): `ability+0x18C` action flag (checked in the default case), `ability+0x18E` reaction-type byte (selects the pose), `ability+0x190–0x196` additional modifier flags, `ability+0x19C` flags (checked for 0x4000, 0x20, 0x02, 0x10, 0x04), and `ability+0x1A4` flag byte (bit 0x80 checked).** — `[S] 1/3`
  - S: `unit+0x134` and ability offsets +0x18C/+0x18E/+0x190–0x196/+0x19C/+0x1A4 read by `FUN_80083978` (BATTLE.BIN), per `research/working_documents/TRANSFORMED_POSE_MYSTERY.md`
  - R: none — reaction-type byte at `ability+0x18E` not present in godot-learning (probed godot-learning/src, godot-learning/tests)
  - src: `research/working_documents/TRANSFORMED_POSE_MYSTERY.md`
- **The reaction-type switch at `ability+0x18E` maps values to reaction animations: 1, 4, 5, 6, 7 → 0x18 (24) damage reaction (only when `ability+0x1A4 & 0x80`); 2, 3 → 0x58/0x59/0x5A (88–90) knock down/back/up based on attacker/target height; 8, 9, 10 → no reaction; 0xB (11), 0xD (13) → 0x1B (27) healing/buff reaction (hands raised); default → 0x19 (25) damaged pose or 0x1B (27), based on +0x18C and the modifier flags.** — `[S·R] 2/3`
  - S: switch in `FUN_80083978` (BATTLE.BIN), per `research/working_documents/TRANSFORMED_POSE_MYSTERY.md`
  - R: `godot-learning/src/data/ReactionType.gd` + `godot-learning/assets/abilities/reaction_animations.json` (SEQ anim ids 24/25/27/88/89/90 = 0x18/0x19/0x1B/0x58–0x5A, per sprite type) + per-ability `target_reaction_type` in `godot-learning/src/data/AbilityDatabase.gd` — validated by `godot-learning/tests/GPUEvasionMeleeHitTest.gd` (all reactions taking_damage) and `godot-learning/tests/UnitDisplayPaintGoldenTest.gd` (`play_reaction_animation` via `ReactionType.get_seq_id`)
  - src: `research/working_documents/TRANSFORMED_POSE_MYSTERY.md`

## Notes

(empty — user territory)

## Related

- [[Ability Execution State Flow]]
- [[Ability Animation Table]]
- [[Effect Execution Model]]
