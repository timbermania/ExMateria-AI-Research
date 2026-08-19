# Sprite Move Opcode

Event instruction `{3B}`/`{6E}` Sprite Move: the arm step does not move the unit — it spawns a per-frame lerp coroutine (`FUN_80146940`, task kind `0xB`) into a free scheduler slot and returns; the coroutine writes the unit's `+0x60/+0x62/+0x64` offset triple one lerp step per yielded frame (true per-frame lerp, not snap-at-Wait), and `{6F} Wait Sprite Move` is a pure main-thread barrier that spins until that unit's move coroutine ends. The offsets change only at elapsed-frame boundaries, and PSX and Godot spend the slide on the same parks (202/206/210/213/216/225 on the scn6 carry).

## Points

- **[S·D·R] 3/3** — The `{3B}` Sprite Move arm step changes nothing: it spawns a per-frame interpolation coroutine (`FUN_80146940 @ 0x80146940`, task kind `0xB`) into a free slot and returns; the coroutine's loop `do { yield(); offset = lerp(start, target, frame/Time); } while (frame < Time)` writes `unit+0x60/62/64` one lerp step per yielded frame (true per-frame lerp, not snap-at-Wait) — and `{6F} Wait Sprite Move` (`FUN_80146F5C @ 0x80146F5C`) is a pure main-thread barrier, `do { yield(); } while (any slot kind == 0xB && unit matches)`: it gates the main thread, it does not create the motion.
  - S: `FUN_80146940 @ 0x80146940` (coroutine loop), `FUN_80146F5C @ 0x80146F5C` (barrier spin) (battle_disassembly.txt)
  - D: single-pass `scn_step` sweep pc203→225 (2026-07-08): no field changes at the `{3B}` arm parks (204/208/211/214); bit-exact live trace in `scenario_1_captures/SPRITE_MOVE_INVESTIGATION.md`
  - R: `godot-learning/src/scenarios/ScenarioDecode.gd` (`sprite_move_intent` / `sprite_move_offset`) + `godot-learning/src/scenarios/ScenarioApply.gd` (`sprite_move`, `{6F}` barrier) + `godot-learning/tests/ScenarioSpriteMoveTest.gd`
  - src: research/working_documents/SCENARIO_WAIT_SEMANTICS.md

- **[D·R] 2/3** — PSX and Godot spend the Sprite-Move slide on the same parks: every `{3B}` offset change lands on the `{6F}` that spins its coroutine, or on the immediately-following `{F1}` when no `{6F}` follows (`{3B}`@211 spends its frame in `Wait(4)@212`) — matching park-for-park across parks 202/206/210/213/216/225; the Round-0 eyeball "no move" at the `{6F}`@205/@209 was actually a perceptually tiny lift (Y −3→−5, Z +8→+16) that the eye reads as stillness.
  - D: single-pass PSX `scn_step` sweep pc200→225, base `scenario6_letgo_full_base` (2026-07-08), read against Godot's `probe_beat_sweep`
  - R: `godot-learning/src/scenarios/ScenarioVM.gd` (post-§8k step-boundary fix) + `godot-learning/tools/probe_beat_sweep.gd`
  - src: research/working_documents/SCENARIO_WAIT_SEMANTICS.md

## Notes

(empty — user territory)

## Related

- [[Event Opcode Catalog]]
- [[Scenario Wait Semantics]]
- [[Unit Sprite Object Struct]]
- [[Unit Sprite Render Pipeline]]
- [[Walk To Opcode]]
