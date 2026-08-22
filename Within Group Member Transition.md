# Within Group Member Transition

How FFT jump-cuts between the *members* of one scenario battle group (Gariland 10 → mid-combat chat 11 → victory 12): driven not by the ATTACK.OUT scenario table but by the group's BattleConditionals set (`{0x0019} Run Scenario N`) evaluated during combat. The crux question — what scene state a member inherits when it fires mid-combat — is settled for the map palette: combat does NOT reset it (H-KEEP, live-verified 2026-07-11), so Godot's palette preservation across members is faithful; the observed "Gariland member 11 renders blue" bug was a Godot-side `{66}` commit-timing defect, root-caused and fixed 2026-07-11. The PSX BC-evaluator address is closed (2026-08-22, statically: `bc_evaluate 0x801425B0` / `bc_predicate 0x80142694`, a 22-case predicate ladder — see Points); still open: whether a member *replaces* or *overlays* the live battle.

## Points

- **Combat does not reset or reload the map base palette at battle entry: the preceding scenario's `{66}` commit survives the entire battle and is inherited by members firing mid-combat — at Orbonne member 5 (the Active-Turn-gated chat) the base `DAT_80099d76` still carries scn4's committed blue while the raw-load staging `0x800f6ab0` is still the untouched warm MAP056, byte-distinct.** — `[D·R] 2/3`
  - D: Exp A live reads, PCSX-Redux port 8080 (2026-07-11), savestate `scenario6_abduct_princess_pre_events.sstate` parked at member 5: `DAT_80099d76[1]=0x1441` (blue) == `0x800e4ea4[1]`, `0x800f6ab0[1]=0x0864` (warm), `current_scenario_id @0x8016A014 = 5` — the warm staging proves the raw base was never copied back over the committed blue through the battle
  - R: `godot-learning/src/scenarios/ScenarioPathApplier.gd` — each member `start(fresh=false)` concatenates onto one persistent world, keeping the committed palette (`MapComposer.palette_texture`) plus scene overlays, so preserve-across-members is faithful (no dedicated palette-persistence test; verified in the headful walk per doc)
  - src: `research/working_documents/WITHIN_GROUP_MEMBER_TRANSITION.md`
- **The Gariland "member 11 renders blue" bug was Godot's `{66}` commit-timing defect, not palette carry-over: scenario 10 arms a blue `{33}` field (PC6, mode 4, 225/225/0 = Δ(−31,−31,0), Time 0), ramps it back to neutral (PC15–16, Time 8), and commits at PC18 only after the `{E5}` wait on the camera task (PC17) — on PSX the camera move (PC13, Time 196) always outlasts the ramp, so PSX commits the settled neutral; Godot's 64-frame ramp outlasted the fast-forwarded ~30-frame wait, so the commit baked a mid-ramp tint (live diag: bias≈(−0.53,−0.53,0) at ramp 34/64). Fix: `ScenarioColorTint.resolve()` snaps an in-flight ramp to its target, called from `_op_commit_palette` before baking — the walk now commits bias=(0,0,0).** — `[S·D·R] 3/3`
  - S: EVENT/TEST.EVT event 10 chunk (bakes to gitignored `godot-learning/assets/scenarios/chunks/scenario_010_chunk.json` via `tools/export_all_scenario_chunks.py`) — PC5 `{32}`/PC6 `{33}` mode 4 (225,225,0) Time 0, PC13 Camera Time 196, PC15/16 `{32}`/`{33}` (0,0,0) Time 8, PC17 `{E5}` Task 4 (camera), PC18 `{66}` — re-verified 2026-08-19 against `project-assets/fft-extract/EVENT/TEST.EVT`
  - D: live-instrumented root9→11 path walk, `_op_commit_palette` commit diag (2026-07-11): mid-ramp bake bias≈(−0.53,−0.53,0) at ramp_frames_left=34/64; post-fix walk commits bias=(0,0,0)
  - R: `godot-learning/src/scenarios/ScenarioColorTint.gd` `resolve()` + `src/scenarios/ScenarioVM.gd` `_op_commit_palette` (calls `resolve()` before baking; commit 2b761f5d1) — guarded by `godot-learning/tests/ScenarioCommitPaletteTest.gd` `_test_op_commit_palette_resolves_inflight_ramp` (84/0)
  - src: `research/working_documents/WITHIN_GROUP_MEMBER_TRANSITION.md`
- **The Gariland mid-combat member (scenario 11) has no `{4D}` Reveal anywhere in its chunk (56 instructions) — single-booting it in Godot stays all black because the fade-rect never lifts; a faithful walk must leave the screen revealed as a live battle would.** — `[S·D] 2/3`
  - S: EVENT/TEST.EVT event 11 — 56-instruction chunk with zero `{4D}`/dark-screen-lift opcodes (verified 2026-08-19 via `tools/extract_event.py`/`tools/disasm_event.py`; bakes to gitignored `assets/scenarios/chunks/scenario_011_chunk.json`)
  - D: Godot single-boot of member 11 observed all black — fade-rect never lifts (2026-07-11, per doc changelog)
  - R: none — no reveal guarantee for `{4D}`-less members in the godot-learning walk (`ScenarioPathApplier` preserves the dark-screen overlay across `start(fresh=false)`, but nothing in the walk lifts it; probed `src/scenarios/` + `tests/`)
  - src: `research/working_documents/WITHIN_GROUP_MEMBER_TRANSITION.md`

- **The PSX BattleConditionals evaluator is closed (statically, no emulator): `bc_evaluate` (`0x801425B0`) walks records 0..9 of the condition-offset table at `0x800475E0` over the set's bytecode at `0x80049A18`, stepping 2-byte opcodes by the operand-count table at `0x801696BC`; `bc_predicate` (`0x80142694`) is a 22-case ladder — exactly FFTPatcher's 22 opcodes — returning 1 continue / 2 fired / 0 condition failed. The set is installed per battle by the `ATTACK.OUT` file+`0x3ECC` off `battle_conditionals_id` at scenario record `+0x16`; `Run Scenario N` ≡ `event-script variable 0x27 = N` at `0x800577B8` (`*(0x80165F9C) + 0x27*4`) and does not write `0x8016A014` (that address is a fiber-stack mirror, its `current_scenario_id` label superseded); a condition is a conjunction of predicates plus exactly one `Run Scenario`, and firing is first-match-wins over records — verified by stepping all 376 conditions in all 146 sets onto every boundary BTLEVT declares (376/376).** — `[S·R] 2/3`
  - S: BATTLE.BIN static RE closed 2026-08-22 — `bc_evaluate 0x801425B0`, `bc_predicate 0x80142694`, tables `0x800475E0` / `0x80049A18` / `0x801696BC`, installer `ATTACK.OUT`+`0x3ECC`, `Run Scenario` write at `0x800577B8`; full write-up + 31-row check suite in `research/working_documents/battle_end_captures/bceval.py`
  - R: `godot-learning/src/scenarios/ScenarioDirector.gd` (first-match-wins record evaluation over the 22-opcode `BattleConditionalOpcode` catalog from `assets/scenarios/battle_conditional_opcodes.json`) + `tests/ScenarioDirectorTest.gd` (Orbonne bc 1: deploy(4) → mid-battle chat(5) → victory abduction(6))
  - src: `research/working_documents/GAME_STATE_TRANSITIONS.md`

## Notes

(empty — user territory)

## Related

- [[Scenario Transition Graph]]
- [[Inter Scene Orchestration]]
- [[Color Tint Luma Modes]]
- [[Combat Color Appliers]]
- [[Scenario Wait Semantics]]
