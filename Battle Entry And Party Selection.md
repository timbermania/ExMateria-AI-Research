# Battle Entry And Party Selection

How a battle begins and how its cast is decided. Entering a battle is not a separate load or transition: the group's map + ENTD load once when the ATTACK/BC walk reaches the root, and combat "begins" when the opener beat ends and control falls to the tactical loop. Two ENTD/deployment fields answer the cast question: the ENTD `control` flag decides whether the cast is predetermined (only Orbonne, scn 3, ENTD 387 — 3 player-controlled units baked into the ENTD, no deployment zone) or drawn from the saved formation, and the deployment zone's `max_squad_size` + tiles (reached via `first_squad_deployment_idx`) set the per-battle squad cap (41 of 72 battles allow the full 5). Turn order is not authored anywhere in the battle-init data — it is an engine rule (CT from Speed + initial CT). Verified against the committed godot-learning artifacts on 2026-08-18. Orbonne's full ENTD-387 cast splits 9 Blue (3 control + 6 AI) / 7 Red — 7 canonical story units + 9 factory generics. Combat entry itself is `CombatLoop.start_battle` (simulator boot + arm combat gambits); the ENTD-driven battle bridge that would hand placed units into it is planned, not yet built (task T3 of the Orbonne navigator handoff).

## Points

- **Entering a battle loads nothing new: the map + ENTD load once when the group root boots, the opener cinematic and the combat share the same loaded world, and combat begins when the opener beat ends and control falls through to the tactical loop — "scn 1→2→3→4→battle" is one continuous graph walk with nothing loaded at a "battle boundary".** — `[S·R] 2/3`
  - S: ATTACK.OUT `entd_idx`@0x07 + group structure (one map + one ENTD per group root; re-verified 2026-08-18 against godot-learning/assets/scenarios/{scenarios.json, scenario_groups.json})
  - R: `godot-learning/src/scenarios/ScenarioPlayerScene.gd` — `_spawn_units` runs once against the group-ROOT chunk and members reuse it (`_play_member`: "Units are spawned ONCE against the group-ROOT chunk"); `tests/ScenarioCinematicWalkerTest.gd` validates the member chunk-swap walk
  - src: `research/working_documents/GAME_STATE_TRANSITIONS.md`
- **Orbonne (scn 3, ENTD 387) is the only battle of 72 with player-controlled units baked into the ENTD — flags2 control bit set for 3 units (unit_id 1/2/4, all Blue); every other battle has control = 0, so the player's units are not in the ENTD at all and are inserted from the saved formation into the deployment zone at battle start.** — `[S] 1/3`
  - S: ENTD 387 (field semantics per `research/key_documents/ENTD_FORMAT.md`) — re-verified 2026-08-18 against godot-learning/assets/scenarios/entd.json: control units 1/2/4, all x=0; the 8 other ENTDs with a control bit belong to tutorial/non-battle roots (410–424, 202), leaving Orbonne unique across the 72 battles
  - R: none — ENTD control-flag cast gating not present in godot-learning (probed src/ incl. src/strategy/; deployment-zone data covered only by tests/ScenarioPlacementDataTest.gd)
  - src: `research/working_documents/GAME_STATE_TRANSITIONS.md`
- **The per-battle squad restriction is the deployment zone's `max_squad_size` plus its placement tiles, reached via the scenario record's `first_squad_deployment_idx`; over the 72 battles the distribution is: no zone 1 (Orbonne), 1:1 (Lionel Castle Gate), 2:4, 3:12, 4:13, 5:41.** — `[S] 1/3`
  - S: deployment-zone table (ATTACK.OUT @0xBBD4) via `first_squad_deployment_idx` — re-verified 2026-08-18 against godot-learning/assets/scenarios/{deployment_zones.json, scenarios.json} (Orbonne idx 0 = no zone)
  - R: none — max_squad_size squad-capping not present in godot-learning
  - src: `research/working_documents/GAME_STATE_TRANSITIONS.md`
- **FFT turn order (who acts first) is not authored in the ENTD or anywhere in the battle-init data — it is an engine rule: CT computed from Speed + initial CT.** — `[ ] 0/3`
  - R: none — CT/turn-order stagger not present in godot-learning (initial CT is 0 for all units; engine is auto-battle per the doc)
  - src: `research/working_documents/GAME_STATE_TRANSITIONS.md`
- **In Orbonne the enemies sit at real ENTD tiles but the player/story units sit at x=0 staging — their final positions come from the opener event script (scn 4), not from static ENTD tiles (Mandalia's formation-marker uses the same x=0 convention).** — `[S·R] 2/3`
  - S: ENTD 387 re-verified 2026-08-18 — Red units carry authored tiles (e.g. uid 134 @(6,13)); Blue control units 1/2/4 at x=0 (y 10/4/3)
  - R: `godot-learning/src/scenarios/ScenarioPlayerScene.gd::_spawn_units`/`_spawn_unit_at` (cinematic_place + initial_spawn_facing_12bit, ADR-0052 chirality) + `tests/ScenarioPlacementDataTest.gd` (ENTD/deployment placement data layer)
  - src: `research/working_documents/GAME_STATE_TRANSITIONS.md`

- **ENTD-387 (Orbonne) is 7 canonical story units + 9 factory generics: every canonical slot has `unit_id == special_name` (ids 1/2/4/5/12/23/52) while the generics carry `unit_id` 120+/0xFF with `special_name` 120–139; team split is 9 Blue (3 control + 6 AI) / 7 Red.** — `[R] 1/3`
  - R: `godot-learning/assets/scenarios/entd.json` record "387" (ENTD4.ENT record 3, derived from `BATTLE/ENTD{1..4}.ENT`) + `godot-learning/tests/ScenarioPathTest.gd` (Orbonne group, map 56 / entd 387); 7/9 split cross-validated in the handoff
  - src: `research/working_documents/HANDOFF_navigator_build_ready.md`
- **Combat entry in godot-learning is `CombatLoop.start_battle(team0_units, team1_units, gambits, terrain_index, map, seed)`: it boots the GPU simulator with units loaded and idle, then arms the combat gambits and goes live — the single entry an ENTD-driven battle bridge would hand placed, team-split units into.** — `[R] 1/3`
  - R: `godot-learning/src/gpu/CombatLoop.gd::start_battle` (:226–234: `boot_battle` + `arm_combat_gambits`), validated by `godot-learning/tests/GPUCombatTestBase.gd` (combat harness driving start_battle across the GPU combat test suite)
  - src: `research/working_documents/HANDOFF_navigator_run_1to7.md`

- **The deployment (pre-battle squad-placement) screen is identified at `0x80079320` with its data struct at `0x80098D88` — "identified, not RE'd" on the PSX side; in godot-learning deployment is folded into GPUArena's Strategy phase rather than a separate state.** — `[S·R] 2/3`
  - S: `0x80079320` → struct `0x80098D88`, state-inventory table of the doc (2026-08-22)
  - R: `godot-learning/src/scenes/GPUArena.gd` Strategy phase (`src/strategy/StrategyPhaseManager.gd`), validated by `tests/StrategyPhaseTest.gd`
  - src: `research/working_documents/GAME_STATE_TRANSITIONS.md`

## Notes

(empty — user territory)

## Related

- [[ENTD Unit Deployment Table]]
- [[Scenario Table]]
- [[Scenario Transition Graph]]
- [[Formation Screen Index]]
- [[Unit Build Pipeline]]
