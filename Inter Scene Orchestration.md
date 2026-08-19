# Inter Scene Orchestration

The layer above a single scenario's playback: who computes the next scenario_id, and which code tears down the current world and boots the next one. As of the 2026-07-10 scn6→7/8 capture the cinematic chain is resolved — the ATTACK.OUT overlay (not BATTLE.BIN) owns a selector that linear-scans the live scenario table and chains on the matching record's `post_scenario_step`/`next_scenario_id` (field theory confirmed live: scn6's `0x81`/`7` drives the chain to the root-7 group, settling on member 8). The battle path instead picks its successor from BattleConditionals bytecode (`{0x0019} Run Scenario N`), and the handoff is gated by a scene-end fiber-join barrier (`FUN_80145f78`) that owns the `0x8016A014` execution-context struct — low byte = scenario id at rest, churned by the fiber scheduler in transit. Still open: the `0x80` world-map fork, the selector's exact match predicate, and the single instruction that commits the id flip.

## Points

- **The cinematic-chain selector lives in the ATTACK.OUT overlay, not BATTLE.BIN: the reader at pc `0x801C3740` (ra `0x801C370C`, overlay base `0x801BF000`) linear-scans the 24-byte scenario table (RAM base `0x801CF938` = file offset 0x10938, stride 0x18) by `scenario_id` (lo @+0x00, hi @+0x01), stops at `scenario_id==0` (all-zero padding — the 480-cut), and the matching record's `post_scenario_step` (@+0x14) + `next_scenario_id` (@+0x12) drive the successor — scn6's `0x81`/`7` chains to the root-7 group, promoting the field theory to live-confirmed.** — `[D] 1/3`
  - D: Read-BPs on scn6's `next_scenario_id` @ `0x801CF9C2` and `post_scenario_step` @ `0x801CF9C4` fired 3× from the same reader (`0x801C3740`/`0x801C3744`/`0x801C3748`, ra `0x801C370C`); live-decoded selector disasm `0x801C3704`–`0x801C374C` (savestate `scenario_6_end_transition.sstate`, 2026-07-10)
  - R: none — the overlay selector not present in godot-learning (group exits are statically derived into `assets/scenarios/scenario_groups.json` by `tools/export_scenario_groups.py` and stored in `src/scenarios/ScenarioGroupDatabase.gd`; inter-group exit deferred, ADR-0054; probed `src/scenarios/` + `tests/`)
  - src: `research/working_documents/INTER_SCENE_ORCHESTRATION.md`
- **`0x8016A014` is the event-VM execution-context base, NOT a clean story scenario counter: its low byte equals the current scenario id at rest, but during a transition the fiber scheduler churns the whole dword (observed sweeping 4..15), so a Write-BP on it cannot isolate "the write that selects scenario 7"; the context struct is owned by the scene-end join fiber — its return sites `0x80145f9c`/`0x80145fc4`/`0x80145fe0` are exactly the pointers stashed in the struct at +0x1C/+0x20.** — `[S·D] 2/3`
  - S: BATTLE.BIN `0x8016A014` struct — +0x1C/+0x20 code pointers `0x80145fe0`/`0x80145fc4`, `0xDB`/`0xE3 08` bytes at +0x28/+0x2C (battle_disassembly.txt)
  - D: Write-BP @ `0x8016A014` fired 14× during the <1 s transition, values sweeping 4..15 (`scenario_6_end_transition.sstate`, 2026-07-10); 4-way snapshot diff 4→5→5→6 across two real transitions (2026-06-20)
  - R: none — the event-VM execution-context struct @ `0x8016A014` not present in godot-learning (cited only as the PSX write target in `src/scenarios/ScenarioDirector.gd`'s doc comment; no struct mirrored; probed `src/scenarios/` + `tests/`)
  - src: `research/working_documents/INTER_SCENE_ORCHESTRATION.md`
- **`FUN_80145f78` is the scene-end join barrier — a cooperative event-VM fiber that loops slots s0=4..14 calling `FUN_80149cbc(s0)`, yielding and rescanning until all slots report idle, then finalizes via `FUN_8013b590(0x27)` (ra `0x80145fc4`) + `FUN_8013c9c0(s0)` (ra `0x80145fe0`), the latter setting `DAT_801696fc` (init `0xFFFFFFFF`; holds the finishing scenario id — 6 at the scene-boot hit); `FUN_80149cbc` reads `g_event_current_task_slot` (`0x80174038`) and walks the fiber slot array `DAT_80169cb8` at stride 0x400, and the doc identifies the capture as the `{E5}` Wait-For-Instruction / fiber-join barrier at scn6's end.** — `[S·D] 2/3`
  - S: `FUN_80145f78` (battle:80145f78), `event_fiber_yield` @ `80145f8c`, `event_fiber_mark_complete` @ `80145f3c`, `FUN_80149cbc`, `0x80174038`, `DAT_80169cb8`, `FUN_8013b590`/`FUN_8013c9c0`, `DAT_801696fc` (battle_disassembly.txt)
  - D: Write-BP @ `0x8016A014` return addresses cluster into `FUN_80145f78` (14 fires at sites `0x80145f9c`/`0x80145fc4`/`0x80145fe0`) (`scenario_6_end_transition.sstate`, 2026-07-10)
  - R: none — the scene-end join/finalize fiber not present in godot-learning (the `{E5}` Wait-For-Instruction opcode itself is mirrored in `src/scenarios/ScenarioVM.gd` `_op_wait_for_instruction` + `tests/ScenarioWaitForInstructionTest.gd`, but the scn-end join path is not; probed `src/scenarios/` + `tests/`)
  - src: `research/working_documents/INTER_SCENE_ORCHESTRATION.md`
- **The observed handoff chain is `6 → (7) → 8`, ending on scenario 8: after scn6's cinematic ends, `0x8016A014`'s low byte settles at 8 within <1 s of emulator time (stable ≥10 s) — scenario 7 (the group root/setup) is transient and scn8 is the settled member, matching `scenarios.json` group 7 = {7 setup root, 8 member}.** — `[D] 1/3`
  - D: steady-state `ffi` poll of `0x8016A014` low byte 6 → 8, stable ≥10 s (`scenario_6_end_transition.sstate`, 2026-07-10)
  - R: none — the inter-group 6→7/8 transition not driven in godot-learning (`tests/ScenarioPathTest.gd` covers Orbonne group 3 root→members only; probed `tests/` + `src/scenarios/`)
  - src: `research/working_documents/INTER_SCENE_ORCHESTRATION.md`
- **The root-world boot entry `BATTLE_open_ATTACK_OUT_deployment` @ `0x8013D320` fires exactly once during the handoff, called from `ra=0x80079328` inside a one-shot scene-boot routine; the scenario id is still 6 at the hit — so the deployment-open is downstream of the id flip, not the selector — and the DMA load chain is `0x8013D320` → thunk `0x8014CF28` → `*0x80173CA8 = FUN_80044954` → DMA3 GO @ `0x800206FC`.** — `[S·D] 2/3`
  - S: `BATTLE_open_ATTACK_OUT_deployment` @ `0x8013D320`, thunk `0x8014CF28`, `FUN_80044954`, DMA3 GO @ `0x800206FC` (BATTLE.BIN, battle_disassembly.txt)
  - D: Exec-BP @ `0x8013D320` fired once from `ra=0x80079328` with the scenario id still 6 and `DAT_801696fc` = 6 (capture pass 2, 2026-07-10)
  - R: none — the ATTACK.OUT deployment-open / scene-boot not present in godot-learning (probed `src/` + `tests/`)
  - src: `research/working_documents/INTER_SCENE_ORCHESTRATION.md`
- **Battle-outcome chaining is the BattleConditionals bytecode instruction `{0x0019}` "Run Scenario N": BC[1] is resident at RAM `0x80049A18` (= `BTLEVT.BIN` +0x542, byte-exact), with `Run Scenario 5` @ BC[1]+0x2E on battle-start and `Run Scenario 6` @ +0x3A on victory — matched the observed `0x8016A014` counter ticks 4→5→5→6, so a battle scenario picks its successor from its BC set.** — `[D·R] 2/3`
  - D: BC[1] capture byte-exact vs `BTLEVT.BIN`+0x542, result offsets +0x2E/+0x3A, counter ticks matched (2026-06-20)
  - R: `godot-learning/src/scenarios/ScenarioDirector.gd` `next_scenario()` (first-match-wins → `Run Scenario N`; `RUN_SCENARIO = 0x0019` in `src/scenarios/BattleConditionalOpcode.gd`) + `tests/ScenarioDirectorTest.gd` (Orbonne bc 1 → 4/5/6), `tests/ScenarioPathTest.gd` (Orbonne group root→members)
  - src: `research/working_documents/INTER_SCENE_ORCHESTRATION.md`
- **Scenario 1's auto-advance despite `next=0`/`step=0` is not driven by the ATTACK.OUT overlay selector — it is the setup-stub special case handled off that path (scn1 is a setup-stub triplet); the overlay selector drives the non-zero-field chains (scn6), which resolves the `SCENARIO_LOADING.md` §3.2.3 anomaly and supersedes the "Event ID counter" hypothesis.** — `[D] 1/3`
  - D: selector-capture resolution — the 2026-07-10 Read-BP capture shows the overlay reader driving the non-zero-field chain while scn1's zero-field record cannot drive it (original anomaly observed 2026-06-20)
  - R: none — the scenario-1 setup-stub advance not present in godot-learning (probed `tests/` + `src/scenarios/`)
  - src: `research/working_documents/INTER_SCENE_ORCHESTRATION.md`
- **The in-RAM event-script chunk slot @ `0x8004A6BC` (8 KB, rotating) is overwritten per transition by a 4-sector CD load, caller `ra=0x8007A018`.** — `[D] 1/3`
  - D: transition capture — 4-sector CD load overwrites the chunk slot, caller `ra=0x8007A018` (2026-06-20, per doc §2 citing `SCENARIO_LOADING.md` §3.2.5)
  - R: none — chunk refresh in godot-learning is JSON-file based (`load_chunk_json` from `assets/scenarios/chunks/`), not CD-DMA (probed `src/scenarios/`)
  - src: `research/working_documents/INTER_SCENE_ORCHESTRATION.md`
- **On the scenario 1→2 transition, 5 file loads fire in order via the per-event loader (`ra=0x8007A018`): `/BATTLE/64.SPR` + `/BATTLE/65.SPR` (32 sectors each → `0x801DF000`, LBAs 233264/233296 — extra unit sprites for scenario 2), `/EVENT/ATTACK.OUT` reloaded (64 sectors → `0x801BF000`, LBA 2448 — the scenario-table refresh), `/EVENT/UNIT.BIN` (32 sectors → `0x801DF000`, LBA 5739 — unit metadata: party rosters, names), and `/EVENT/WLDFACE.BIN` (64 sectors → `0x801DF000`, LBA 6330 — full-size character portraits); scenario 2 then starts immediately from the same rotating 8 KB chunk slot `0x8004A6BC`, and scenarios 1 and 2 share that chunk's string table (messages 1..0x12 decode the same dialogue across both scenarios).** — `[D] 1/3`
  - D: 1→2 transition capture — 5-load table + 1640-byte chunk diff A→B (2026-06-20 PM; `scenario_1_captures/file_load_capture_scenario_2_transition.json` + `scenario_post_transition_chunk_0x8004A6BC.bin`)
  - R: none — the transition load sequence not present in godot-learning (chunk refresh is JSON-file based; probed `src/scenarios/`, `tests/`)
  - src: `research/working_documents/SCENARIO_LOADING.md`

## Notes

(empty — user territory)

## Related

- [[Scenario Table]]
- [[Scenario Transition Graph]]
- [[Event End Opcode]]
- [[Block Execution]]
- [[Event Opcode Catalog]]
- [[CDROM DMA Load Pipeline]]
