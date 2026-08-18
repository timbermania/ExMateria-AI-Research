# Unit Build Pipeline

How godot-learning composes a playable unit from reference data: stat growth and effective-stat derivation run end-to-end from `base_stats.json` / `jobs.json` through `StatCalculator` + `UnitProgression` (the same proven path that stats the project's own roster units, so ENTD-seeded battle units reuse it as-is), and post-spawn wiring reuses the roster primitives (`configure_spawned_unit` binds progression + gambits by reference and derives equipped abilities from the job's skill set). The ENTD-specific pieces — a `Character.from_entd_slot` factory and a `start_entd_battle` bridge — are planned (Orbonne navigator handoff, tasks T3) and not yet built as of 2026-08-18.

## Points

- **FFT's level-up growth formula is implemented as `StatCalculator.grow_stat(current_raw, job_constant, level)`: new_raw = current_raw + current_raw/(job_constant + level), in fixed-point ×16384; `UnitProgression.level_up` applies it per stat (hp/mp/speed/pa/ma) with the job's `*_constant`.** — `[R] 1/3`
  - R: `godot-learning/src/data/StatCalculator.gd::grow_stat` (:17) + `src/units/UnitProgression.gd` (:114–118), validated by `godot-learning/tests/ProgressionTesterTest.gd` (level_up / add_experience / stat-query steps)
  - src: `research/working_documents/HANDOFF_navigator_run_1to7.md`
- **Effective (displayed) stat = raw stat × job multiplier/100 + equipment bonuses: each `UnitProgression.get_effective_{hp,mp,speed,pa,ma}` is `StatCalculator.get_effective_stat(raw, job.<stat>_multiplier)` plus `get_equipment_stat_bonuses()`.** — `[R] 1/3`
  - R: `godot-learning/src/units/UnitProgression.gd` (:215–273) + `src/data/StatCalculator.gd::get_effective_stat` (:41), validated by `godot-learning/tests/ProgressionTesterTest.gd` (`_test_stat_queries`, `_test_equipment_elements_statuses`)
  - src: `research/working_documents/HANDOFF_navigator_run_1to7.md`
- **A spawned roster unit is wired post-spawn by `BaseRoster.configure_spawned_unit`: it binds the roster entry's progression and gambit list by reference (edits persist on the entry), derives equipped abilities from the job's `skill_set_id` via `AbilityDatabase.get_skill_set_actions` (max 3, skips placeholder 256 "Nothing", filters by MP cost), and resets HP/MP to full.** — `[R] 1/3`
  - R: `godot-learning/src/roster/BaseRoster.gd::configure_spawned_unit` (:233, incl. `_setup_job_abilities`) + `src/data/AbilityDatabase.gd::get_skill_set_actions` (:19371); no test names `configure_spawned_unit` directly (probed godot-learning/tests/)
  - src: `research/working_documents/HANDOFF_navigator_run_1to7.md`

## Notes

(empty — user territory)

## Related

- [[Battle Entry And Party Selection]]
- [[ENTD Unit Deployment Table]]
- [[Ability Execution Index]]
- [[Unit Index]]
