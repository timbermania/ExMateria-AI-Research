# Scenario Transition Graph

The entire FFT story graph is latent in three parsed artifacts (the ATTACK.OUT scenario table, the BattleConditionals sets, the deployment-zone table) and folds into one derived graph: 480 scenarios in 155 groups, 375 guarded BC edges plus 162 deterministic ATTACK edges. A battle is not a state that alternates with scenarios — it is a group whose setup/root node owns a BC set that routes cinematic members at battle moments, and the terminal (victory) member carries the group's one ATTACK exit to the next group's root. The graph is not one connected chain: 105 of 155 roots are reachable only from the world map, and the 106 maximal "runs" between map returns are mostly single-group; the world map is the mandatory dispatch backbone and the biggest open RE gap. All edge-role invariants and counts were re-verified against the committed godot-learning artifacts on 2026-08-18.

## Points

- **A battle is a scenario group, not a separate state: the group's setup/root node owns a BattleConditionals set that routes the cinematic members at battle moments (Var509==1 opener, mid-combat predicates, Victory terminal), and the terminal member carries the group's one ATTACK exit (0x81 next group / 0x80 world map / 0x82 reset) — so the spine alternates group-to-group, with cinematic beats woven inside each battle.** — `[S·R] 2/3`
  - S: ATTACK.OUT fields `post_scenario_step`@0x14 / `next_scenario_id`@0x12 / `battle_conditionals_id`@0x16 (scenario table @0x10938, retail US) + BC set bytecode; re-verified 2026-08-18 against godot-learning/assets/scenarios/{scenarios.json, scenario_groups.json, battle_conditionals.json} — 145 setup nodes own all 375 BC edges, only terminal members carry the 162 ATTACK exits
  - R: `godot-learning/src/scenarios/ScenarioDirector.gd` (first-match-wins BC evaluation) + `src/data/BattleConditionalDatabase.gd`; `tests/ScenarioPathTest.gd` drives the Orbonne group (bc_id 1, members 3–6) root-to-member against the committed artifacts (debug planner, not the live combat loop)
  - src: `research/working_documents/GAME_STATE_TRANSITIONS.md`
- **Edges partition perfectly by node role: all 375 BC edges live on the group's setup/root node (0 on members), all 162 ATTACK edges live on members (0 on roots: 97×0x80 world-map sink, 52×0x81 next-scenario, 13×0x82 reset), no ATTACK edge stays in-group, and every inter-group 0x81 edge lands on the next group's root.** — `[S] 1/3`
  - S: re-verified 2026-08-18 against godot-learning/assets/scenarios/{scenarios.json, scenario_groups.json, battle_conditionals.json} — 0 exceptions across all 480 nodes (reproduces the doc's 2026-07-16 role-crosscheck against scenario_groups.json)
  - R: none — invariants not mechanized in godot-learning (no test asserts them; doc notes they are "prose-verified, not mechanized")
  - src: `research/working_documents/GAME_STATE_TRANSITIONS.md`
- **The 50 cross-group BC edges are cinematic reuse, not story branching: 49 splice the nine Deep Dungeon floor battles (roots 95, 97, 99, 101, 103, 105, 107, 109, 114) into group 91's two shared members — scenario 93 "Deep Dungeon Panel Found" under Team Unit Location (×40) and scenario 94 "Deep Dungeon (Victory — Used for all Floors)" under Victory (×9) — and 1 splices grp367 (Bethla Sluice) into single-node group 490's reused hint under Unit Present (×1); a cross-group target is a specific member to play once, not a group to enter at its root.** — `[S] 1/3`
  - S: re-verified 2026-08-18 against godot-learning/assets/scenarios/{battle_conditionals.json, scenario_groups.json} — 50 cross-group BC edges from 10 source roots; 49 land on a member, 1 on a setup; guard tally Team Unit Location 40 · Victory 9 · Unit Present 1
  - R: none — cross-group BC routing not present in godot-learning (probed src/scenarios/, src/data/, src/debug/, tests/)
  - src: `research/working_documents/GAME_STATE_TRANSITIONS.md`
- **The ten Deep Dungeon floor-groups have no ATTACK exit (post_scenario_step 0x00 / Unknown) — floor-to-floor descent is not encoded in the scenario transition graph; group 91 exits via GoToWorldMap and the DD's own descent logic drives the rest.** — `[S] 1/3`
  - S: scenario_groups.json `exit` field re-verified 2026-08-18 — roots 95–114 (odd) all exit=Unknown_0x00, root 91 exit=GoToWorldMap
  - R: none — Deep Dungeon progression not present in godot-learning
  - src: `research/working_documents/GAME_STATE_TRANSITIONS.md`
- **Only 72 of the 155 group roots are actual battles — 45 standard (BC set contains Victory()) plus 27 scripted (combat predicates without Victory(): boss-HP threshold / unit-death / positional end conditions); the other 73 BC-owning roots are pure cinematics whose BC is only the Var509==1 launch latch, and 10 roots own no BC set at all; every BC-owning group launches its first member with the same Var509==1 latch.** — `[S] 1/3`
  - S: re-verified 2026-08-18 against godot-learning/assets/scenarios/{battle_conditionals.json, scenario_groups.json} — 72 roots with non-latch combat predicates (45 with Victory), 73 latch-only, 10 no-BC
  - S: `kind` tags on the 480-node transition graph — vocabulary battle/cinematic_latch/quiet/cinematic_linear/exit_worldmap/reset, regenerated by `tools/build_transition_graph.py`; doc §8 confirms 72 `kind=battle` nodes, not 145 (the pre-classifier count); graph artifact + classifier live on branch `import-godot-game` (commit 92711cfe6), not in main's worktree
  - R: none — the battle-vs-latch kind split is not in godot-learning's main worktree (classifier `tools/build_transition_graph.py` + `kind` tags exist only on branch import-godot-game, commit 92711cfe6)
  - src: `research/working_documents/GAME_STATE_TRANSITIONS.md`
- **The story is not one connected chain: 105 of 155 group roots have in-degree 0 (reachable only from the world map or boot), 94 groups exit via GoToWorldMap, 50 chain onward, 10 have no exit and 1 resets; chaining by 0x81 yields 106 maximal "runs" between map returns — 67 are single-group and the longest is 5 (Riovanes 291→295→297→328→330) — so the world map is the mandatory dispatch backbone and the biggest open RE gap.** — `[S] 1/3`
  - S: re-verified 2026-08-18 against godot-learning/assets/scenarios/{scenarios.json, scenario_groups.json, battle_conditionals.json} — exits {GoToWorldMap 94, GoToNextScenario 50, Unknown 10, ResetGame 1}; 105 roots in-degree 0 counting ATTACK+BC; 106 runs with lengths {1:67, 2:31, 3:6, 4:1, 5:1}; opening run 1→3→7→9 hands to the world map after Gariland
  - R: none — world-map dispatcher state not present in godot-learning
  - src: `research/working_documents/GAME_STATE_TRANSITIONS.md`
- **In godot-learning's main worktree the state transitions are not wired: the scenario VM never launches GPUArena, the BC evaluator (ScenarioDirector) is consumed only by the debug path planner, and GPUArena._on_loop_victory only prints a result and optionally auto-resets — no post-battle scenario routing; combat team routing hard-codes team0=PartyRoster / team1=EnemyRoster.** — `[R] 1/3`
  - R: verified on main 2026-08-18 — `src/scenes/GPUArena.gd::_on_loop_victory` (print + optional `DebugConfig.combat_auto_reset` only), `src/scenarios/ScenarioDirector.gd` referenced only from `src/debug/ScenarioPathDebugPanel.gd`, team hard-code at `src/scenes/GPUArena.gd` lines 233–236; no test asserts a live-loop transition (probed tests/)
  - src: `research/working_documents/GAME_STATE_TRANSITIONS.md`
- **The canonical top-level state list is {TITLE, SCENARIO, BATTLE, DEPLOYMENT, WORLD_MAP, FORMATION, RESET}, machine-readable in game_states.json with GameState.for_node_kind/for_sink mapping transition-graph node kinds and sinks onto the owning state.** — `[R] 1/3`
  - R: branch `import-godot-game` (commit 92711cfe6): `godot-learning/src/scenarios/GameState.gd` + `assets/scenarios/game_states.json` (source of truth), guarded by `tests/GameStateTest.gd` (doc reports 29/0 after the classifier fix); files not yet in main's worktree
  - src: `research/working_documents/GAME_STATE_TRANSITIONS.md`
- **The Orbonne opening run is three chained groups: group 1 "Chapel of Orbonne" = scn 1 (setup, prayer-cinematic stub) + scn 2 (member), BC=0, exit GoToNextScenario→3; group 3 "Orbonne Monastery" = scn 3 (setup, kind=battle, ENTD 387) + scn 4–6 (members), BC=1, exit→7; group 7 "Military Academy" = scn 7 (setup, kind=cinematic_latch) + scn 8 (member), BC=2, exit→9 — walking 1→3→7 chains whole groups by those exits and needs no world-map or formation system.** — `[S] 1/3`
  - S: ATTACK.OUT scenario table @0x10938 (per-record exit / BC-id fields), parsed into godot-learning/assets/scenarios/{scenario_groups.json, scenarios.json} — the doc's "Confirmed group structure" block (groups 1/3/7 members, BC ids 0/1/2, GoToNextScenario exits)
  - R: none — the opening-run 1→3→7 walk not present in godot-learning (probed tests/ and src/scenarios/; tests/ScenarioPathTest.gd drives only Orbonne group 3 root→members; the headless 1→3→7 walk test is task T1 of this doc, not yet built)
  - src: `research/working_documents/HANDOFF_navigator_run_1to7.md`

## Notes

(empty — user territory)

## Related

- [[Scenario Table]]
- [[ENTD Unit Deployment Table]]
- [[Battle Entry And Party Selection]]
- [[Formation Screen Index]]
