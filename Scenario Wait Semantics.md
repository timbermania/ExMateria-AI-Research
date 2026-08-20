# Scenario Wait Semantics

In the PSX scenario event engine a Wait does not "process" a particular subsystem — it yields frames. A frame elapses only when the running task yields to the 16-slot cooperative scheduler (stride `0x400`, current slot `DAT_80174038`, yield `FUN_8014CA80`, main dispatcher `FUN_80143BD8`), and every frame that elapses advances every active task across all event blocks — so "what updates during a Wait" = whatever is active/pending on those frames, independent of the Wait opcode kind. The Wait type only sets the main thread's release condition: `{F1} Wait Time` blocks for exactly N frames, `{6F} Wait Sprite Move` spins until that unit's kind-`0xB` move coroutine ends, `{29} Wait Walk` until the walk ends. Pinned by static disassembly + a single-pass dynamic sweep of the scn6 carry (pc193–225, 2026-07-08) and mirrored in Godot's per-tick advance + dispatch loop with step-consistency guards.

## Points

- **[S·D·R] 3/3** — A scenario Wait does not process a particular set of subsystems — it yields frames: every elapsed frame advances every active task across all event blocks, so what updates during a Wait is only what is active/pending on those frames, independent of the Wait kind; a Wait overlapping no active/pending work shows no change at all.
  - S: yield primitive `FUN_8014CA80 @ 0x8014CA80`, main dispatcher `FUN_80143BD8 @ 0x80143BD8` (runs non-blocking opcodes back-to-back without yielding), current slot `DAT_80174038`, 16 slots stride `0x400` (battle_disassembly.txt)
  - D: single-pass `scn_step` sweep pc203→225, base `scenario6_abduct_punch_pickup_start` (2026-07-08): the `{F1} Wait(16)@213` advances the chocobo (slot 6)'s own walk — `+0x18` 40959→57339 plus walk accumulators — while the main thread waits (its own block context at `0x8016A000+`), and the `{F1} Wait(4)@207` / `Wait(16)@213` change nothing on the static-held Ovelia
  - R: `godot-learning/src/scenarios/ScenarioVM.gd` (`_advance_frame`: per tick `_advance_tick_visuals` + `_tick_once` round-robins all live contexts) + `godot-learning/tests/ScenarioWaitCadenceTest.gd` (10/0) + `godot-learning/tests/ScenarioSteppingConsistencyTest.gd` (49/0)
  - src: research/working_documents/SCENARIO_WAIT_SEMANTICS.md

- **[S·D·R] 3/3** — The Wait opcode only sets the main-thread release condition and never picks which subsystem advances: `{F1} Wait Time` blocks for exactly N frames (inline `do { yield(); } while(--T)`), `{6F} Wait Sprite Move` (`FUN_80146F5C @ 0x80146F5C`) spins `do { yield(); } while (any slot kind == 0xB && unit matches)` until that unit's move coroutine ends, and `{29} Wait Walk` until the walk ends.
  - S: `{6F}` barrier `FUN_80146F5C @ 0x80146F5C`, `{F1}` inline yield loop, `{28}` arm `FUN_8013E5C0 @ 0x8013E5C0` (battle_disassembly.txt)
  - D: single-pass `scn_step` sweep pc200→225 (2026-07-08): the `{6F}@224` releases only after both latches are consumed and both slides settled; the `{6F}@205`/`@209` each spend exactly the 2-frame slide
  - R: `godot-learning/src/scenarios/ScenarioApply.gd` (`sprite_move` + the `{6F}` barrier) + `godot-learning/tests/ScenarioSpriteMoveTest.gd` + `godot-learning/tests/ScenarioWaitCadenceTest.gd`
  - src: research/working_documents/SCENARIO_WAIT_SEMANTICS.md

## Notes

(empty — user territory)

## Related

- [[Event Opcode Catalog]]
- [[Block Execution]]
- [[Walk To Opcode]]
- [[Sprite Move Opcode]]
- [[Unit Anim Opcode]]
- [[Scenario Beat Capture]]
- [[Wait Rotate Unit Opcode]]
