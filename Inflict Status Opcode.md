# Inflict Status Opcode

The event instruction `{92}` Inflict Status applies one status to a scenario unit from its Status byte operand. A re-runnable PCSX-Redux rig (2026-07-05) poked the Status byte at all three live scenario-6 call sites and swept values, confirming that on stock ROM only Status 0 (revive/normalise, the `v1==2` path, no status bits touched), 1 (Crystal: unit+0x58=0x40, +0x3b1=0x02), and 2 (Poison+Critical: unit+0x5b=0x80, forced anim 0x16, +0x3b1=0x03) are live — the `0x80+` single-status range is dead code without the EIUH hack (`SS=0x91` Frog observed no-op). The runtime status-index order for that dead range is `idx = SS − 0x80`, `byte*8 + (7−bit)` (idx 0x05=Dead, 0x06=Crystal, 0x10=Critical, 0x1f=Poison, all four exercised). `godot-learning` implements Status 0/1/2 in `ScenarioApply.inflict_status` and fails loud on any other value.

## Points

- **The runtime status-index order for the `{92}` `0x80+` range is `idx = SS − 0x80`, `byte*8 + (7−bit)` — idx 0x05=Dead, 0x06=Crystal, 0x10=Critical, 0x1f=Poison, all four exercised indices confirm it.** — `[S·D] 2/3`
  - S: three `{92}` Status-byte sites `0x8004A713 / 0x8004A719 / 0x8004A71F` (op+3), scenario 6 event code — named in `research/working_documents/op92_status_probe/README.md`
  - D: op92_status_probe sweep `0x00 0x01 0x02 0x91`, before/after status dumps in `captures/op92_ss_<XX>/log.txt` (2026-07-05)
  - R: none — 0x80+ status-index table not present in godot-learning (probed `godot-learning/src` + `godot-learning/tests`; the `{92}` handler in `src/scenarios/ScenarioVM.gd` special-cases Status 0/1/2 only)
  - src: `research/working_documents/op92_status_probe/README.md`
- **`SS=1` (Crystal) crystallizes the unit: unit+0x58 = 0x40 and unit+0x3b1 = 0x02.** — `[D·R] 2/3`
  - D: op92_status_probe `SS=1` capture, `captures/op92_ss_01/` (2026-07-05)
  - R: `godot-learning/src/scenarios/ScenarioApply.gd` `inflict_status` (Status=1 → `world.inflict_crystal`) + `src/scenarios/ScenarioWorld.gd` `inflict_crystal` (body → animated diamond, keyed on +0x58 & 0x40) — validated by `tests/ScenarioInflictStatusTest.gd` `_test_status1_inflicts_crystal` / `_test_status1_crystallizes_and_hides_body`
  - src: `research/working_documents/op92_status_probe/README.md`
- **`SS=2` (Poison+Critical) renders the unit prone/green: unit+0x5b = 0x80, animation forced to 0x16, unit+0x3b1 = 0x03.** — `[D·R] 2/3`
  - D: op92_status_probe `SS=2` capture, `captures/op92_ss_02/` (2026-07-05)
  - R: `godot-learning/src/scenarios/ScenarioApply.gd` `inflict_status` (Status=2 → `world.inflict_poison_critical`, `src/scenarios/ScenarioWorld.gd`), Critical kneel forced via `src/animation/AnimationStateController.gd` `to_critical_idle` — validated by `tests/ScenarioInflictStatusTest.gd` `_test_status2_inflicts_poison_critical` / `_test_status2_poisons_and_kneels`
  - src: `research/working_documents/op92_status_probe/README.md`
- **`SS=0x91` (Frog) is a no-op on stock ROM — it falls through, empirically confirming the `0x80+` single-status range is dead code without the EIUH hack.** — `[D·R] 2/3`
  - D: op92_status_probe `SS=0x91` capture, `captures/op92_ss_91/` (2026-07-05)
  - R: `godot-learning/src/scenarios/ScenarioVM.gd` `_op_inflict_status` encodes the 0x80+ range as dead code on stock and fails loud + halts instead of no-op (issue #154) — validated by `tests/ScenarioInflictStatusTest.gd` `_test_unmodelled_status_fails_loud`
  - src: `research/working_documents/op92_status_probe/README.md`
- **`SS=0` on healthy units takes the `v1==2` path — no status bits touched.** — `[D·R] 2/3`
  - D: op92_status_probe `SS=0` capture, `captures/op92_ss_00/` (2026-07-05)
  - R: `godot-learning/src/scenarios/ScenarioApply.gd` `inflict_status` (Status=0 → `world.revive_and_normalise`: revive-if-dead + normalise to Standing, explicitly "NOT a status bit") — validated by `tests/ScenarioInflictStatusTest.gd` `_test_status0_does_not_halt` / `_test_living_unit_normalises_without_revive`
  - src: `research/working_documents/op92_status_probe/README.md`
- **Scenario 6 (abduct princess, pre-events) holds three `{92}` Inflict Status calls, the Status operand byte at `0x8004A713`, `0x8004A719`, `0x8004A71F` (op+3).** — `[S·D·R] 3/3`
  - S: `0x8004A713 / 0x8004A719 / 0x8004A71F` (op+3 Status-byte sites), scenario 6 event code — named in `research/working_documents/op92_status_probe/README.md`
  - D: op92_status_probe poked all three sites, captures (2026-07-05)
  - R: `godot-learning/tests/ScenarioInflictStatusTest.gd` `_test_real_chunk6_three_sites_do_not_halt`
  - src: `research/working_documents/op92_status_probe/README.md`

## Notes

(empty — user territory)

## Related

- [[Event VM Index]]
- [[Event Opcode Catalog]]
