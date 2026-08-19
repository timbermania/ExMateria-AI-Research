# Unit White Flash Reaction Handler

The non-TRAP unit white flash on hit: the flinch reaction animation arms reaction handler type 14 via SEQ opcode 0xC1, and the per-frame unit update dispatches to FUN_8008a924 — an 8-phase palette state machine (SFX + white flash → darken → dark snap → re-flash → animated fade) driven by tick thresholds and attack/item sync flags. This is the flash path for melee attacks and items; the TRAP hit-cloud effect instead drives its white flash inline in charge_effect_handler_2 ([[TRAP Hit Effect Particle System]]), and breakpoint probing confirmed FUN_8008a924 never runs during a TRAP hit.

## Points

- **Trigger and dispatch: PostGenericAttack → execute_unit_reaction_pose plays the flinch animation, whose SEQ data contains opcode 0xC1 that writes the reaction type to unit+0x87 (arg0 + 2 = 14 for a hit flinch); the per-frame unit update dispatches through the reaction handler table at PTR_LAB_80093da0 (18 entries), and type 14 resolves to FUN_8008a924.** — `[S] 1/3`
  - S: trigger path and handler table, `research/working_documents/TRAP_PARTICLE_SYSTEM_DEEP_DIVE.md` §14
  - R: none — reaction-handler type 14 / unit+0x87 dispatch not present in godot-learning (probed `godot-learning/src/` + `tests/` for 0x8008a924; the hit white flash is instead implemented as the TRAP hit effect's inline palette flash in `godot-learning/src/effects/TrapPaletteController.gd`)
  - src: `research/working_documents/TRAP_PARTICLE_SYSTEM_DEEP_DIVE.md`
- **FUN_8008a924 is an 8-phase machine: phase 0 (immediate) plays SFX 0x6A and turns the white flash ON — apply_unit_palette_effect(4, 2, unit, +31, +31, +31) → phase 2; phase 2 (tick ≥ 17) darkens — (4, 4, unit, −31) → phase 3; phase 3 (sync flag == 0) does terrain setup and a dark snap — (4, 0, unit, −31) → phase 5; phase 5 (tick ≥ 17) re-flashes — (4, 4, unit, +31) → phase 6; phase 6 (tick ≥ 33) removes the shift — (8, 2, unit, 0, 0, 0) → phase 7; phase 7 (sync flag == 0) clears the handler and completes. Phases 1 and 4 are sync-wait phases for attack ('A') and item ('I') hit types. This hardcoded sequence is the equivalent of five keyframes of an E###.BIN timeline color track — the same apply_unit_palette_effect calls a script color track would keyframe — including the mode-4 −31 dark phase that the TRAP inline path lacks.** — `[S] 1/3`
  - S: phase table, `research/working_documents/TRAP_PARTICLE_SYSTEM_DEEP_DIVE.md` §14 + "White Flash = Hardcoded Timeline Color Track"
  - R: none — not present in godot-learning (probed `godot-learning/src/` + `tests/`; see the trigger point above)
  - src: `research/working_documents/TRAP_PARTICLE_SYSTEM_DEEP_DIVE.md`
