# Dead Unit Fade

The scenario-6 corpse-clear beat: when the Chapter-1 battle ends, instruction 7 of the abduction cutscene — `{0x43} Call Function`, `Function=4` — dispatches in `event_scenario_interpreter` to a 21-slot sweep `FUN_80147cf0` that fades out and removes every on-field unit on the non-persisting team side (`unit+0x1BA & 0x30 != 0`, a stable ENTD-sourced 2-bit team/side index — not hp==0, not a death latch), then halts the event VM for 120 frames. The fade is a two-pass palette fade driven by a 3-state removal machine on the sprite's `+0x13e` flag (pass 1 mode-4 Δ(−31,−31,0) kills R/G; pass 2 mode-4 Δ(−31,−31,−31) from the per-frame updater `FUN_800870ac` goes fully black; then hide/despawn), collapsing the corpse CLUT to near-black + STP. The whole beat is statically RE'd (Ghidra) and byte-verified live on PCSX-Redux (2026-07-17), and is ported in `godot-learning` on branch `import-godot-game` (TDD `ScenarioDeadUnitFadeTest` 22/0 + headful `--scenario=6`).

## Points

- **Scenario-6 instruction 7 is `{0x43} Call Function` with `Function=4` (raw `43 04`, 1-byte operand confirmed by the live in-RAM size table `DAT_8014D170[0x43] == 1`) — the engine hook that fades/removes the KO'd enemy units at the end of the Chapter-1 battle, before the staging warps and Ovelia's `{10} Display Message`.** — `[S·D·R] 3/3`
  - S: instr 7 of `godot-learning/assets/scenarios/chunks/scenario_006_chunk.json` (raw `43 04`), opcode size table `DAT_8014D170` (`battle_decompilation.c`)
  - D: live RAM prologues + size-table check from `scenario6_abduct_princess_pre_events.sstate` on port 8080; a plain `resume` drives the battle (which auto-resolves, no input) into the transition in ~13 s (2026-07-17)
  - R: `godot-learning/src/scenarios/ScenarioVM.gd` `_op_call_function` (0x43 → `_begin_dead_unit_fade` for Function=4; rest of the function space keeps the loud non-halting stub — branch `import-godot-game`) + `godot-learning/tests/ScenarioCallFunctionStubTest.gd` (non-halt stub assertion retargeted to Function=1) + `ScenarioDeadUnitFadeTest._test_handler_bound`
  - src: `research/working_documents/SCENARIO6_DEAD_UNIT_FADE.md`
- **The `Function==4` branch in `event_scenario_interpreter` @ `0x80143BD8` is exactly `FUN_80147cf0()` followed by `event_fiber_yield_n(0x78)` — a 120-frame (~2 s) event-VM halt; since the tint ramp is only ~8 frames, most of the yield is a hold on the faded corpses before the cutscene's warps and Ovelia's "Let go of me!" box run.** — `[S·D·R] 3/3`
  - S: `0x80143BD8` (the `Function==4` case; `battle_decompilation.c`)
  - D: the hold observed live — after the sweep the scene stays in the yield, then advances to the warps + Ovelia box (scenario 6, pcsx-redux port 8080, 2026-07-17)
  - R: `godot-learning/src/scenarios/ScenarioVM.gd` `_current_ctx.wait_ticks = _DEAD_FADE_HOLD_TICKS` (= 120, non-halting — `_running` stays true) + `godot-learning/tests/ScenarioDeadUnitFadeTest.gd` `_test_holds_vm_120_ticks`
  - src: `research/working_documents/SCENARIO6_DEAD_UNIT_FADE.md`
- **`FUN_80147cf0` @ `0x80147CF0` (size 0xA8) sweeps the 21 unit slots 0..0x14 and fades a slot only if it has a live on-field sprite (`unit_sprite_object_exists` @ `0x8008CBB4` + `get_unit_portrait_index` @ `0x8008CDD0` != −1) AND `unit+0x1BA & 0x30 != 0`, calling `unit_graphic_palette_revert(slot, 2, −0x1F, −0x1F, 0)` @ `0x8008D26C` per hit — the `addunit_add_action(...) != −2` sub-condition is a near-tautology (a slot with a live sprite never returns −2) and does not meaningfully filter.** — `[S·D·R] 3/3`
  - S: `FUN_80147cf0 @ 0x80147CF0` and its callees (`battle_decompilation.c`)
  - D: paused at `0x80147CF0` from the isolated `scenario6_pre_deadunit_fade.sstate` (PCSX save slot 9, fires in ~5 s with zero confounding fan-out calls); 21-slot survey and the `0x8008D26C` BP fired exactly for slots 5–9 with `p2=2, R=−31, G=−31, B=0` — byte-exact (2026-07-17)
  - R: `godot-learning/src/scenarios/ScenarioVM.gd` `_begin_dead_unit_fade` + `_is_dead_unit_fade_target` (`scenario_team_color != 0 AND scenario_present`, the byte-aligned mirror of the gate) + `ScenarioDeadUnitFadeTest._test_selection_partition` / `_test_persisting_side_and_staged_untouched`
  - src: `research/working_documents/SCENARIO6_DEAD_UNIT_FADE.md`
- **`unit+0x1BA & 0x30` is a stable 2-bit team/side index (bits 4–5), not hp==0 and not a death latch: it is a cached mirror rewritten into the unit record every frame by a generic byte-`memcpy` (`FUN_8019AB48`, store @ `0x8019AB58`) from the authoritative record, is never written by death, and partitions the field into persisting side 0 (`&0x30==0` — Ramza's party + guests) and cleared side 1 (`&0x30==0x10` — the enemy soldiers plus the abduction's incoming actors) — so `Call Function 4` = "fade out every on-field unit NOT on the persisting side"; the reader shape `(x&0x30)>>4` indexes the per-side array `DAT_8018f5f4`, and the initial `+0x1BA` is stored at battle-load by the setup-overlay writers `0x8017F4EC`/`0x8018E53C`.** — `[S·D·R] 3/3`
  - S: reader shape into `DAT_8018f5f4`, writers `0x8017F4EC`/`0x8018E53C` (Ghidra)
  - D: write-BP on `+0x1BA` (slots 3, 5–9) across the whole battle→fade window: dead slot 3 keeps `&0x30=0` (kept), slots 5–9 hold `0x94` (`&0x30=0x10`), and full-HP off-field staged slots 12/13 carry the flag but are skipped (no live sprite) (2026-07-17)
  - R: `godot-learning` mirrors the same ENTD team byte as `scenario_team_color` (`ScenarioPlayerScene.gd`/`ScenarioWorld.gd`; scn6 loads ENTD record 387: the five enemy corpses = uids 134–138, `team_color 1` + present → fade; staged actors uids 5/139 `team_color 1` but `scenario_present==false` → spared, then re-Added by instrs 8–12) + `ScenarioDeadUnitFadeTest._test_selection_partition`
  - src: `research/working_documents/SCENARIO6_DEAD_UNIT_FADE.md`
- **The fade is a TWO-PASS palette fade + a 3-state removal machine on the sprite's `+0x13e`: pass 1 (from the sweep) `unit_graphic_palette_revert` → `FUN_8008945C` @ `0x8008945C` sets `+0x13e=1`, `+0x298=0`, `+0x12 = (x & 0xFF9F) | 0x21` and fires `color_unit_fanout(4, 2, slot, −31, −31, 0)` (kills R/G, keeps B); ~8 frames later the per-frame updater `FUN_800870ac` @ `0x800870AC` sees `+0x13e==1`, sets `+0x13e=2` and fires pass 2 `color_unit_fanout(4, 4, slot, −31, −31, −31)` (all channels → black); at `+0x13e==2` the unit is hidden (`obj[4] == DAT_80096118` → `unit_sprite_object_hide()`) or despawned (`FUN_80086f2c`, `+0x13e=3`) — all inside the Call Function 4 120-frame hold.** — `[S·D·R] 3/3`
  - S: `0x8008945C`, `0x800870AC`, `color_unit_fanout @ 0x800933C4`, `DAT_80096118`, `0x80086f2c` (`battle_decompilation.c`)
  - D: fan-out capture at BP `0x800933C4` (ra distinguishes call sites): pass 1 `ra=0x800894F8` slots 5..9 ascending, pass 2 `ra=0x80087134` slots 9..5 descending ~8 frames later (2026-07-17)
  - R: `godot-learning/src/scenarios/ScenarioVM.gd` `_advance_dead_unit_fades` — pass 2 via `ScenarioColorTint.apply(4,-31,-31,-31,Time=4)` the moment pass 1's ramp lands (`ScenarioColorTint.is_ramping()`, new pure query), then hide (`visible=false`) + `godot-learning/tests/ScenarioDeadUnitFadeTest.gd` `_test_two_pass_fade_to_black_then_hide`
  - src: `research/working_documents/SCENARIO6_DEAD_UNIT_FADE.md`
- **The faded corpse's end state is byte-captured: its CLUT in `dynamic_clut_view_strip` @ `0x800E4EA4` (layout `bank*0x100+slot*0x10+c`, BGR555) collapses to bank 3 entry 0 = `0000`, entries 1–15 = `8400` = (R0, G0, B1) + STP, and bank 4 = `FFFF` (31,31,31) + STP — a near-black semi-transparent silhouette — while living units' banks keep their varied real palettes.** — `[S·D] 2/3`
  - S: `dynamic_clut_view_strip @ 0x800E4EA4` (bank stride 0x100; `battle_decompilation.c`)
  - D: byte-dump of faded slot 5's strip from the scenario-6 run on port 8080 (2026-07-17)
  - R: none — the CLUT end-state byte-capture is PSX-only; the godot-learning fade ends at `visible=false` with no CLUT collapse (probed `godot-learning/src/scenarios/`, `godot-learning/tests/`)
  - src: `research/working_documents/SCENARIO6_DEAD_UNIT_FADE.md`

## Notes

(empty — user territory)

## Related

- [[Event End Opcode]]
- [[Event Opcode Catalog]]
- [[Color Tint Luma Modes]]
- [[Event Unit Selector]]
- [[Display Message Opcode]]
- [[Scenario 6 Ride Off]]
- [[ENTD Unit Deployment Table]]
- [[Unit Sprite Object Struct]]
