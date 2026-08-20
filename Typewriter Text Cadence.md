# Typewriter Text Cadence

How FFT types out event dialog: the active dialog's tick (`event_dialogue_tick` @ `0x8012F6D4`) reveals exactly one glyph per call, and its per-glyph emit loop (`0x8012FB84..0x8012FBEC`) sleeps `a2·(3−throttle)` frames between reveals via `event_fiber_yield_n(1)` fiber yields. `a2` is the sticky per-glyph budget (`local_58`): default 1 at message init, re-set by each inline `{Delay NN}` (`0xE2 NN`) until the next one — so the Orbonne prayer overlay's leading `{Delay 05}` yields 10 VBlanks/glyph, the boxed dialog's default yields ~1 VBlank/glyph net (bunched pairs), and a `{Delay NN}` itself costs `NN·2` VBlanks at the default throttle. Throttle is the single global `DAT_80165F88` (0 fastest … 3 paused; 1 default; sole writer `FUN_8013da00`). Everything is clocked off the game-loop VBlank wait (`frame_sync` `FUN_80093a98` → `FUN_8001dba8`), not rendered frames, and the PSX never gates event-script dispatch on the typewriter — the `Dialog=0x09` overlay is fire-and-forget (Map Darkness and Ovelia's Unit Anim dispatch mid-prayer). Grounded in live pcsx captures of scenario 1 (2026-06-30/07-01); Godot mirrors it with the sticky-budget `TypewriterController` clocked off the 60 Hz `ScenarioVM._tick_once`, a non-blocking `DialogueOverlay`, and the boxed `DialogueBox` at throttle 2.

## Points

- **The dialog tick's per-glyph emit loop (`0x8012FB84..0x8012FBEC`) runs `a2·(3−throttle)` iterations, each a single `event_fiber_yield_n(1)` — `event_dialogue_tick` (`0x8012F6D4`) sleeps that many frames between reveals and reveals exactly one glyph per call, so the tick-entry cadence IS the glyph cadence and `a2` sets frames-per-glyph directly.** — `[S·D·R] 3/3`
  - S: emit loop `0x8012FB84..0x8012FBEC`, yield wrapper `0x8014c858` (`event_fiber_yield_n`), doc §2.B/§7.0
  - D: live entry BP `0x8012F6D4` (log `cyc,ra,a2`) + blit BP `0x8012FB38` (log `cyc,y`): base-overlay ticks at `a2=5` gap 10.00 VBlanks, 5 clean instances (2026-06-30, `scenario_1_captures/last_run/probe_typewriter_a2_overlay.csv`)
  - R: `godot-learning/src/scenarios/TypewriterController.gd` — frame-discrete walker, one glyph per tick, `_glyph_cooldown = _budget * _factor() − 1` (header cites the `0x8012FB84` emit loop) — validated by `godot-learning/tests/DialogueOverlayTest.gd` `_test_production_cadence_golden`
  - src: `research/working_documents/TYPEWRITER_TEXT_CADENCE.md`
- **The overlay-vs-boxed speed difference is the tick's `a2` glyph budget (3rd arg of `event_dialogue_tick`): `a2=5` for the prayer overlay, `a2=1` for the boxed dialog, both entering through the same caller `ra=0x80132728` (jal `0x80132720`) — the overlay is ~9× slower (10 VBlanks/glyph vs net ~1.16) at the same throttle.** — `[S·D·R] 3/3`
  - S: overlay caller `0x80132570`, boxed caller `0x80132720` (both jal `0x80132720`), doc §2.B/§7.0
  - D: live a2 trace (2026-06-30, `probe_typewriter_a2_{overlay,boxed}.csv`; savestates `orbonne_prayer_pre_scenario_load`→START / `orbonne_prayer_mid_dialog`→CIRCLE): overlay base `a2=5` (10.00-VBlank tick gaps), boxed `a2=1` (bunched pairs, net 1.16 VBlanks/glyph)
  - R: `godot-learning/src/scenarios/DialogueOverlay.gd` (throttle default 1 → factor 2; budget driven by `{Delay}` tokens) + `godot-learning/src/ui3/assemblies/DialogueBox.gd` (`throttle = 2` → factor 1, reproducing the ROM's net boxed cadence) — validated by `godot-learning/tests/DialogueOverlayTest.gd` `_test_production_cadence_golden` + `godot-learning/tests/DialogueBoxTest.gd`
  - src: `research/working_documents/TYPEWRITER_TEXT_CADENCE.md`
- **At throttle 1 the Orbonne prayer overlay reveals every printable glyph at a steady 10 VBlanks (median 9.999 over the captured 41 glyphs), and a `{Newline}` costs 0 extra VBlanks (line 2 starts at the base gap).** — `[D·R] 2/3`
  - D: live glyph-blit BP `0x800248FC` filtered `ra==0x8012FB40` — 41-glyph prayer capture (2026-06-30, `scenario_1_captures/last_run/probe_typewriter_glyph_timeline.tsv`): median inter-glyph gap 9.999 VBlanks; newline gap (y=8→24) 10.0
  - R: `godot-learning/src/scenarios/TypewriterController.gd` (newline drained free, glyphs step at `_budget·_factor()`) + `DialogueOverlay.gd` production throttle 1 — validated by `godot-learning/tests/DialogueOverlayTest.gd` `_test_production_cadence_golden` / `_test_chapel_prayer_full_advance`
  - src: `research/working_documents/TYPEWRITER_TEXT_CADENCE.md`
- **The space token (`0xFA`) gets a hard-coded 4 px X-advance that ignores the width table, yet still consumes one full cursor/cadence step — a lone space reads as base + one extra 10-VBlank tick, no blit.** — `[S·D·R] 3/3`
  - S: space advance `0x80132578..0x80132588` (hard-codes 4 px), doc §2.B
  - D: 41-glyph timeline (2026-06-30, `probe_typewriter_glyph_timeline.tsv`): all 5 lone-space gaps = 20.0 VBlanks = base10 + space10
  - R: `godot-learning/src/scenarios/TypewriterController.gd` (space is one text-char step at `_budget·_factor()`) — validated by `godot-learning/tests/DialogueOverlayTest.gd` `_test_production_cadence_golden` (space typed at f=46)
  - R: layout-side of the same 4px override — `godot-learning/src/scenarios/DialogueOverlay.gd` `_SPACE_PSX_WIDTH := 4.0` (L50, used by `_glyph_width` L348) + `src/ui3/assemblies/DialogueBox.gd` `SPACE_WIDTH_PX := 4.0` (L91) applied through the `UIText.space_width` export (`src/ui3/elements/UIText.gd` L51; default −1 falls back to the font's natural 10px space, `char_width = 10` in `assets/fonts/font_meta.json`) — box suites `DialogueBoxTest.tscn` / `ScenarioBoxedDialogTest.tscn` stay green; no dedicated space-width assertion yet
  - src: `research/working_documents/TYPEWRITER_TEXT_CADENCE.md`
  - src: `research/working_documents/scenario_1_captures/handoff_boxed_dialog_polish.md`
- **`{Delay NN}` (charmap `0xE2 NN`) costs `NN·(3−throttle)` VBlanks — 2 VBlanks/unit at the throttle-1 default — because the delay operand becomes the tick's `a2` budget and runs through the same emit loop as a glyph: the overlay's `{Delay 0F}` (15u) fires one tick at `a2=15` that sleeps exactly 30.00 VBlanks = 15·2.** — `[S·D·R] 3/3`
  - S: `0xE2 NN` handler writes the budget (`0x80132368` `lbu t0,0x0(s4)` → `0x80132374` `sh t0,local_58`), loaded to `a2` at `0x80132538`, doc §7.1
  - D: live a2 trace (2026-07-01, `probe_typewriter_a2_overlay.csv`): `{Delay 0F}` step fires exactly one tick `a2=15`, 30.00-VBlank sleep; the 5 base glyphs obey the same `a2·(3−throttle)` mapping (`a2=5` → 10.00)
  - R: `godot-learning/src/scenarios/TypewriterController.gd` — delay token sets `_budget = NN` and arms `_delay_frames_left = _budget * _factor()` (throttle 1 → ×2) — validated by `godot-learning/tests/DialogueOverlayTest.gd` `_test_delay_token_cost_is_budget_times_factor` + `_test_production_cadence_golden`
  - src: `research/working_documents/TYPEWRITER_TEXT_CADENCE.md`
- **The per-glyph budget (`local_58`) is STICKY: the `0xE2` handler's write persists across subsequent glyphs until the next `{Delay}` — the default budget 1 is set once at message init (`FUN_801310c0`, constant writers `0x80131400`/`0x80131C2C`/`0x8013213C`), so the prayer's steady `a2=5` is just its leading `{Delay 05}` persisting; there is no separate delay data table to parse.** — `[S·D·R] 3/3`
  - S: `0x80132368`/`0x80132374` (`0xE2` write) + `0x80132538` (load to `a2`) + default writers `0x80131400`/`0x80131C2C`/`0x8013213C`, init `FUN_801310c0` @ `0x801310C0` (doc §7.1, resolved 2026-07-10)
  - D: live a2 trace (2026-06-30, `probe_typewriter_a2_overlay.csv`): base glyphs all `a2=5` (= leading D05 persisting), `a2=15` only at the `{Delay 0F}` step, then back to 5 — the operand-driven, non-constant budget
  - R: `godot-learning/src/scenarios/TypewriterController.gd` — `_budget` starts at 1, each `{Delay NN}` sets it and it persists; every glyph AND delay costs `_budget·(3−throttle)` (header cites this doc §7.1) — validated by `godot-learning/tests/DialogueOverlayTest.gd` `_test_sticky_budget_little_money` (scenario-8 `{Delay 19}` run types slow at 50 VBlanks/glyph) + `_test_default_budget_types_at_factor`
  - src: `research/working_documents/TYPEWRITER_TEXT_CADENCE.md`
- **A `{Delay}` immediately before the final glyph is NOT applied on the PSX: the Orbonne prayer's trailing `{Delay 3C}` (60u) sits before the last `.` yet the `.` blits only 10 VBlanks after the preceding `e` (predicted 70 if applied) — the final glyph flushes at base cadence (mechanism, end-of-message flush vs emit batching, unconfirmed).** — `[D] 1/3`
  - D: 41-glyph blit timeline (2026-06-30, `probe_typewriter_glyph_timeline.tsv`): final `.` 10.0 VBlanks after `e`; Σ(token costs − trailing D60) = 496 ≈ the observed 491 VBlanks from `0x10` dispatch to last glyph
  - R: none — the final-glyph flush exception is not present in godot-learning; `TypewriterController.gd` applies the trailing delay inline (doc §6.1 notes a ~1 s harmless tail overshoot)
  - src: `research/working_documents/TYPEWRITER_TEXT_CADENCE.md`
- **The PSX does NOT gate event-script dispatch on the typewriter — the `Dialog=0x09` overlay is a fire-and-forget fiber: in the Orbonne prayer, Map Darkness (`0x1a`) fires 1.4 s after dispatch (~32 % typed) and Ovelia's next Unit Anim (`0x11`) 3.9 s after (~40 % typed), while the last glyph blits at +8.2 s.** — `[D·R] 2/3`
  - D: opcode-dispatch BP `0x80143D34` (log `cyc, opcode s4, ip v0`) timeline (2026-06-30, `probe_typewriter_opcode_timeline.tsv`): `0x10` @ 25.071 s, `0x1a` @ 26.504 s, `0x11` @ 28.971 s, last glyph @ 33.254 s (cyc0-relative)
  - R: `godot-learning/src/scenarios/ScenarioVM.gd` `_op_display_message` — the `Dialog==0x09` overlay is non-blocking (the old `_arm_wait_until(not overlay.is_active())` + `_OVERLAY_TYPEWRITER_WATCHDOG_TICKS` arm was removed 2026-07-01) — validated by `godot-learning/tests/ScenarioPrayerOverlayTest.gd` `_test_overlay_does_not_gate_dispatch`
  - src: `research/working_documents/TYPEWRITER_TEXT_CADENCE.md`
- **The Godot typewriters are clocked off the game-loop rate, not rendered frames: `DialogueOverlay` is pumped one `advance_frames(1)` per `ScenarioVM._tick_once` (the fixed 60 Hz logical tick, wall-clock-locked like the console VBlank) and `DialogueBox` uses a `delta·60` accumulator — replacing `Engine.get_frames_drawn()` deltas that made type-out speed scale with monitor refresh (144 Hz typed 2.4× faster).** — `[R] 1/3`
  - R: `godot-learning/src/scenarios/DialogueOverlay.gd` (self-clocking `_process` removed; `advance_frames` documented off the VM 60 Hz tick) + `godot-learning/src/scenarios/ScenarioVM.gd` `_tick_once` (`dialogue_overlay.advance_frames(1)`) + `godot-learning/src/ui3/assemblies/DialogueBox.gd` (`_reveal_accum += dt * _TICK_HZ`) — validated by `godot-learning/tests/DialogueOverlayTest.gd` `_test_production_cadence_golden`; headful `ScenarioPlayer.tscn` run (2026-07-01) confirmed Map Darkness + Unit Anim fire mid-prayer
  - src: `research/working_documents/TYPEWRITER_TEXT_CADENCE.md`
- **The typewriter is clocked by the game loop, not rendered frames: `frame_sync` (`FUN_80093a98` @ `0x80093A98`) runs at the end of every game-loop iteration and calls the kernel VBlank wait `FUN_8001dba8(N)` @ `0x8001DBA8`; in the chapel event-script cinematic `N = DAT_80045980`, measured live = 1 — one game-loop iteration (one dialog tick) per VBlank.** — `[S·D] 2/3`
  - S: `FUN_80093a98` @ `0x80093A98`, `FUN_8001dba8` @ `0x8001DBA8`, `DAT_80045980` @ `0x80045980` (doc §2.A, per `EFFECT_TIMING_SYSTEM.md` decompilation)
  - D: live global sampling at the Orbonne prayer (2026-06-30): `DAT_800960e4=0x34` (else branch), complexity 0, `g_frame_pacing_value`=0, `DAT_80045980`=1
  - R: none — the `frame_sync`/VBlank-wait loop is not present in godot-learning; the faithful analogue is `ScenarioVM._tick_once`'s fixed 60 Hz logical tick (see the re-clock point)
  - src: `research/working_documents/TYPEWRITER_TEXT_CADENCE.md`
- **The text-speed throttle is a single global, `DAT_80165f88` @ `0x80165F88` (0 = fastest, 1 = default, 2 = slow, 3 = paused), with `FUN_8013da00` @ `0x8013DA00` (`sw a0, DAT_80165f88`) as sole writer; the Orbonne prayer runs the whole 41-glyph message at throttle 1 (no text-speed opcode runs before PC 42), making the `a2·(3−throttle)` cost factor 2.** — `[S·D·R] 3/3`
  - S: `DAT_80165f88` @ `0x80165F88`, sole writer `FUN_8013da00` @ `0x8013DA00`, doc §2.B
  - S: caller walk on `battle_disassembly.txt` (Q6, 2026-06-28): the 7 callers of `FUN_8013da00` (single writer of `DAT_80165F88` @ `0x8013da0c`) are `0x8013bdf4` (a0=0x2, `FUN_8013bdcc` entry — pause during dialog-handle allocation), `0x8013bef8` (a0=0x1, near exit — resume normal), `0x8013f644` (a0 live from caller, generic state-restore, unverified), `0x8014329c` / `0x801432dc` / `0x80143cdc` (a0=0x1, dialog-state-machine teardown resets), `0x80145250` (a0=s2 = event-script text-speed opcode payload; the living doc labels this 0x76, the vault pins text-speed to 0x75 — see [[DarkScreen Opcode]])
  - D: live global sampling (2026-06-30, Orbonne prayer): `DAT_80165f88=1` throughout the capture (throttle column of `probe_typewriter_glyph_timeline.tsv`)
  - R: `godot-learning/src/scenarios/TypewriterController.gd` `throttle` (default 1, factor `3 − throttle` floored at 1; `{75}` Set Text Speed not wired yet) + `DialogueBox.gd` `throttle = 2` export — validated by `godot-learning/tests/DialogueOverlayTest.gd` `_test_default_budget_types_at_factor` + `tests/DialogueBoxTest.gd`
  - src: `research/working_documents/TYPEWRITER_TEXT_CADENCE.md`
- **`0x8014c858` is `event_fiber_yield_n` — a "sleep N frames" wrapper that loops the real yield primitive `event_fiber_yield` @ `0x8014ca80` (full context save into the coroutine table `PTR_DAT_80165f98`) `a0` times; called with `a0=1` from the emit loop it yields once.** — `[S] 1/3`
  - S: disasm-confirmed 2026-07-01 (doc §7.0/§9): `0x8014c858` loops `0x8014ca80`; context table `PTR_DAT_80165f98`
  - R: none — `event_fiber_yield_n` / `event_fiber_yield` are not present in godot-learning
  - src: `research/working_documents/TYPEWRITER_TEXT_CADENCE.md`
- **Each revealed dialog glyph is one `GsLoadImage` fire — `jal 0x800248FC` at call site `0x8012FB38` (ret `ra=0x8012FB40`) uploading one glyph cell to the VRAM text strip; the prayer overlay's glyphs land at VRAM y=8/y=24 and the boxed dialog's at y=16/y=32, cleanly segmenting one BP's captures by mode.** — `[S·D] 2/3`
  - S: `jal 0x800248FC` @ `0x8012FB38` (ret `ra=0x8012FB40`), doc §2.B/§8
  - D: live Exec BP at `0x800248FC` filtered `ra==0x8012FB40` — 41 prayer glyphs at y=8/24 + 35 boxed glyphs at y=16/32 in one capture (2026-06-30, `probe_typewriter_glyph_timeline.tsv`)
  - R: none — the `GsLoadImage` glyph blit is not present in godot-learning (the overlay renders from the BATTLE.BIN font atlas; see [[Display Message Opcode]])
  - src: `research/working_documents/TYPEWRITER_TEXT_CADENCE.md`
- **The boxed dialog reveals glyphs in PAIRS at throttle 1 — two tick entries within ~0.03 VBlank, then a ~2-VBlank fiber sleep (gap histogram {0: 15, 2: 19}) — for a net 1.12–1.16 VBlanks/glyph, ~9× faster than the overlay's 10.** — `[D·R] 2/3`
  - D: live boxed a2 trace (2026-06-30, `probe_typewriter_a2_boxed.csv`): `a2=1`, pairs ~2 VBlanks apart, net 1.16 VBlanks/glyph (header line 1.09, body line 1.18)
  - R: `godot-learning/src/ui3/assemblies/DialogueBox.gd` `throttle = 2` (factor 1 → one glyph per tick; net rate identical to the ROM pairing, pairing imperceptible) — validated by `godot-learning/tests/DialogueBoxTest.gd`
  - src: `research/working_documents/TYPEWRITER_TEXT_CADENCE.md`
- **The boxed dialog's live reveal clock must be reset on every box transition: while the box is inactive during the advance-gate dwell, `_process` returns early (`!is_active()`) and the clock freezes, so without the reset the next box's first `_advance_delta()` is the entire dwell in frames and instant-types the whole text (the "only the first box types" bug) — fixed by `_reset_reveal_clock()` on every `show_dialog` and `swap_text`.** — `[R] 1/3`
  - R: `godot-learning/src/ui3/assemblies/DialogueBox.gd` `_reset_reveal_clock()` (called from `_render_current_page`, which both `show_dialog` and `swap_text` run) — no dedicated test: unit tests drive `advance_frames(n)` directly and never exercised the live clock; the doc flags verifying this in a real multi-box scenario, not the test rig
  - src: `research/working_documents/scenario_1_captures/handoff_dialogue_box_geometry_REFINEMENT.md`

## Notes

(empty — user territory)

## Related

- [[Display Message Opcode]]
- [[Effect Frame Pacing]]
- [[Map Darkness Opcode]]
- [[Unit Anim Opcode]]
- [[Scenario Wait Semantics]]
