# Gambit Encoder Safety Net

The godot-learning `GambitEncoder` injects one fixed "attack nearest enemy" safety-net gambit into the last slot (`MAX_USER_GAMBITS`, 5) of every unit's gambit buffer at pack time — `ALWAYS` condition, `NEAREST_ENEMY` target (`TargetSelector.enemies()`), `ATTACK` action, action-target the triggering unit. It is a pure buffer-injection concern (ADR-0048): the domain layer (`Gambit` / `GambitList` / UI editor) stays unaware and authors at most `MAX_USER_GAMBITS` slots. Consequence: any unit with no authored gambits attacks the nearest enemy by default — so the "attack nearest" gambit the Orbonne game-state navigator needs is zero new code. The user-facing list itself is separately padded by `GambitList.ensure_fixed_size` to exactly `VISIBLE_SLOTS` (4) with "Always: Wait on Self" gambits.

## Points

- **The `GambitEncoder` auto-appends one fixed safety-net gambit per unit at slot `MAX_USER_GAMBITS` (5): `ALWAYS` + `NEAREST_ENEMY` target + `ATTACK` — the authored region is padded so it always lands in slot 5, it survives a 6th authored gambit (never displaced/overwritten), and the packed GPU buffer carries it; the domain/UI layer never sees it (ADR-0048), so gambit-less units attack the nearest enemy by default.** — `[R] 1/3`
  - R: `godot-learning/src/gpu/GambitEncoder.gd` (`_safety_net_config` + auto-append at encode) + `godot-learning/tests/GambitSafetyNetTest.gd` (asserts slot 5, ATTACK / NEAREST_ENEMY / ALWAYS, incl. packed buffer)
  - src: `research/working_documents/HANDOFF_navigator_build_ready.md`
- **Unauthored gambit slots pad to "Always: Wait on Self", not empty: `GambitList.ensure_fixed_size` pads (and trims) a unit's user gambit list to exactly `VISIBLE_SLOTS` (4) with empty "Always: Wait on Self" gambits — the idle fallback for unauthored slots in the domain-layer list; called from `Unit` and from the combat test harness before battle start.** — `[R] 1/3`
  - R: `godot-learning/src/data/GambitList.gd::ensure_fixed_size` (:71–80; call sites `src/units/Unit.gd:1521` and `src/data/GambitList.gd:121`), exercised by `godot-learning/tests/GPUCombatTestBase.gd` (:444, pads the list before `start_battle` across the GPU combat suite)
  - src: `research/working_documents/HANDOFF_navigator_run_1to7.md`

## Notes

(empty — user territory)

## Related

- [[Ability Execution Index]]
- [[Battle Entry And Party Selection]]
