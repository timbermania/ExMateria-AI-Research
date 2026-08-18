# Gambit Encoder Safety Net

The godot-learning `GambitEncoder` injects one fixed "attack nearest enemy" safety-net gambit into the last slot (`MAX_USER_GAMBITS`, 5) of every unit's gambit buffer at pack time — `ALWAYS` condition, `NEAREST_ENEMY` target (`TargetSelector.enemies()`), `ATTACK` action, action-target the triggering unit. It is a pure buffer-injection concern (ADR-0048): the domain layer (`Gambit` / `GambitList` / UI editor) stays unaware and authors at most `MAX_USER_GAMBITS` slots. Consequence: any unit with no authored gambits attacks the nearest enemy by default — so the "attack nearest" gambit the Orbonne game-state navigator needs is zero new code.

## Points

- **The `GambitEncoder` auto-appends one fixed safety-net gambit per unit at slot `MAX_USER_GAMBITS` (5): `ALWAYS` + `NEAREST_ENEMY` target + `ATTACK` — the authored region is padded so it always lands in slot 5, it survives a 6th authored gambit (never displaced/overwritten), and the packed GPU buffer carries it; the domain/UI layer never sees it (ADR-0048), so gambit-less units attack the nearest enemy by default.** — `[R] 1/3`
  - R: `godot-learning/src/gpu/GambitEncoder.gd` (`_safety_net_config` + auto-append at encode) + `godot-learning/tests/GambitSafetyNetTest.gd` (asserts slot 5, ATTACK / NEAREST_ENEMY / ALWAYS, incl. packed buffer)
  - src: `research/working_documents/HANDOFF_navigator_build_ready.md`

## Notes

(empty — user territory)

## Related

- [[Ability Execution Index]]
- [[Battle Entry And Party Selection]]
