# Ability Execution State Flow

The battle state machine drives ability execution through 0x1C start charging → 0x1E sustain charging → 0x29 resolve target → 0x2a pre-execution → 0x2b effect loading → 0x2c execution → 0x2e post-effect → 0x15 damage resolution → 0x28 cleanup. In state 0x2c the unit's swing animation and the E###.BIN effect start simultaneously, with the swing holding its final frame until the state machine moves on; the swing SEQ's hit frame fires opcode 0xDE (PostGenericAttack), which triggers the target's reaction in the same frame (reaction pose, Break text, damage numbers, hit sparks), and state 0x2e afterwards restores a height-based recovery pose.

## Points

- **The full battle state flow for a weapon-swing ability is 0x1C start charging (FUN_80073fe0) → 0x1E sustain charging (FUN_800740d4) → 0x29 resolve target (FUN_800739cc) → 0x2a pre-execution (FUN_80073fb8) → 0x2b effect loading (FUN_80073eec) → 0x2c execution (FUN_80073b9c) → 0x2e post-effect (FUN_800732c8) → 0x15 damage resolution (FUN_80071ee8) → 0x28 cleanup (FUN_800734cc); when the ability has no E###.BIN effect file (FUN_801a1814 returns 0), state 0x2b is skipped and the state machine jumps directly from 0x2a to 0x2c, as for basic Attack (0x16F).** — `[S] 1/3`
  - S: state handlers FUN_80073fe0/FUN_800740d4/FUN_800739cc/FUN_80073fb8/FUN_80073eec/FUN_80073b9c/FUN_800732c8/FUN_80071ee8/FUN_800734cc, FUN_801a1814, per `research/working_documents/ability_execution_animation_system.md`
  - src: `research/working_documents/ability_execution_animation_system.md`
- **The weapon swing and the E###.BIN effect play simultaneously, not sequentially: state 0x2b loads the effect (CD read, decompression, VRAM upload) via FUN_801a15b4 (0x801a15b4), and state 0x2c starts both the swing (FUN_80082c10) and the effect per-frame loop (FUN_801a18d8) at the same moment, with the state 0x2c handler FUN_800773f8 (0x800773f8) waiting each frame for both the effect to finish (DAT_8009612c == 0) and the frame-timer condition before transitioning to 0x2e.** — `[S] 1/3`
  - S: FUN_801a15b4 (0x801a15b4), FUN_800773f8 (0x800773f8), DAT_8009612c, per `research/working_documents/ability_execution_animation_system.md`
  - src: `research/working_documents/ability_execution_animation_system.md`
- **The swing animation plays through its SEQ frames and holds on its final frame while the effect continues — the unit does not return to idle until the battle state machine transitions past state 0x2c, the same hold behaviour as casting-pose abilities.** — `[S] 1/3`
  - S: state 0x2c handler FUN_800773f8 (0x800773f8), per `research/working_documents/ability_execution_animation_system.md`
  - src: `research/working_documents/ability_execution_animation_system.md`
- **The attacker's swing SEQ contains opcode 0xDE (PostGenericAttack) at the hit frame, and when the SEQ instruction pointer reaches it the target reaction fires in the same frame: execute_unit_reaction_pose sets the reaction animation, FUN_80083874 adds the "Break" text overlay (Break-range abilities only), trigger_unit_hit_reaction shows damage numbers, and FUN_8006894c spawns hit-spark particles; the reaction animation is 0x18 (evade) or 0x19 (taking damage) for physical hits (effect types 1, 4–7), a height-based shield block 0x58/0x59/0x5A for item/throw (types 2, 3), and 0x1B (receive heal) for healing (types 0xB, 0xD).** — `[S] 1/3`
  - S: SEQ opcode 0xDE (PostGenericAttack), FUN_80083874, FUN_8006894c, per `research/working_documents/ability_execution_animation_system.md`
  - src: `research/working_documents/ability_execution_animation_system.md`
- **Post-flinch recovery is height-based: state 0x2e calls FUN_80082cc4 (0x80082cc4), which uses FUN_8008278c to select recovery pose 0x1C, 0x1D, or 0x09 to return the unit to standing, while state 0x15 handles damage resolution (death, KO, status application).** — `[S] 1/3`
  - S: FUN_80082cc4 (0x80082cc4), FUN_8008278c, per `research/working_documents/ability_execution_animation_system.md`
  - src: `research/working_documents/ability_execution_animation_system.md`

## Notes

(empty — user territory)

## Related

- [[Ability Animation Table]]
- [[Weapon Animation System]]
- [[Effect Execution Model]]
- [[Unit Anim Opcode]]
