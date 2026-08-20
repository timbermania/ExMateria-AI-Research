# Inflict Status Opcode

The event instruction `{92}` InflictStatus is a 6-byte opcode (`Unit`:u16, pad `0x00`, `Status`, `Wait` (usually `0x0C`), pad; opsize table `0x8014d202 = 0x05`) whose live-verified handler `0x80145EE0` does not apply the status inline — it spawns a fire-and-forget cooperative task (`FUN_80149BEC(0x10)`, body `FUN_80148E88`, context from `FUN_8014CA38` at the operand pointer), which is why the status change lands on a later tick and the opcode pairs with `{E5}`. On stock ROM only `Status` 0 (revive/normalise: revive to 1 HP if dead, Standing/Critical pose by HP, block-event-HP-restore flag `DAT_80165FB4 = 0x41`), 1 (Crystal: remove Dead+Critical, add Crystal, +0x3b1=0x02), 2 (Poison+Critical: idx 0x1f, +0x3b1=0x03, anim 0x16) are live — the `0x80+` single-status range is dead code without the EIUH hack (`SS=0x91` Frog observed no-op; all-480-chunk census: 257×0 / 3×1 / 6×2 / 0×`0x80+`). A separate load-time auto-apply (`FUN_8012d8e4` @ `0x80143cb0`) applies any `{92}` at file load to matching Jumping/Floating units even if the instruction is never reached. `godot-learning` implements Status 0/1/2 in `ScenarioApply.inflict_status` and fails loud on any other value.

## Points

- **The runtime status-index order for the `{92}` `0x80+` range is `idx = SS − 0x80`, `byte*8 + (7−bit)` — idx 0x05=Dead, 0x06=Crystal, 0x10=Critical, 0x1f=Poison, all four exercised indices confirm it.** — `[S·D] 2/3`
  - S: three `{92}` Status-byte sites `0x8004A713 / 0x8004A719 / 0x8004A71F` (op+3), scenario 6 event code — named in `research/working_documents/op92_status_probe/README.md`
  - S: live-confirmed render-byte mapping — idx 0x05↔`+0x58 & 0x20`, 0x06↔`+0x58 & 0x40`, 0x10↔`+0x5a & 0x01`, 0x1f↔`+0x5b & 0x80` (`inflict_status_op92_decode.md` §11.1, 2026-07-05)
  - D: op92_status_probe sweep `0x00 0x01 0x02 0x91`, before/after status dumps in `captures/op92_ss_<XX>/log.txt` (2026-07-05)
  - R: none — 0x80+ status-index table not present in godot-learning (probed `godot-learning/src` + `godot-learning/tests`; the `{92}` handler in `src/scenarios/ScenarioVM.gd` special-cases Status 0/1/2 only)
  - src: `research/working_documents/op92_status_probe/README.md`
- **`SS=1` (Crystal) crystallizes the unit: unit+0x58 = 0x40 and unit+0x3b1 = 0x02.** — `[S·D·R] 3/3`
  - S: `FUN_80148E88` SS=1 branch static disasm — three `FUN_80149100` applies (idx 0x05 a2=0 removes Dead, idx 0x10 a2=0 removes Critical, idx 0x06 a2=1 adds Crystal) then `sb unit+0x3b1 = 0x02` (`inflict_status_op92_decode.md` §4.1/§11.5, 2026-07-05)
  - D: op92_status_probe `SS=1` capture, `captures/op92_ss_01/` (2026-07-05)
  - R: `godot-learning/src/scenarios/ScenarioApply.gd` `inflict_status` (Status=1 → `world.inflict_crystal`) + `src/scenarios/ScenarioWorld.gd` `inflict_crystal` (body → animated diamond, keyed on +0x58 & 0x40) — validated by `tests/ScenarioInflictStatusTest.gd` `_test_status1_inflicts_crystal` / `_test_status1_crystallizes_and_hides_body`
  - src: `research/working_documents/op92_status_probe/README.md`
- **`SS=2` (Poison+Critical) renders the unit prone/green: unit+0x5b = 0x80, animation forced to 0x16, unit+0x3b1 = 0x03.** — `[S·D·R] 3/3`
  - S: `FUN_80148E88` SS=2 branch static disasm — single `FUN_80149100(idx 0x1f, a2=1)` (Poison), `sb unit+0x3b1 = 0x03`, anim `0x16` via `BATTLE_set_unit_anim_value` (`inflict_status_op92_decode.md` §4.1, 2026-07-05)
  - D: op92_status_probe `SS=2` capture, `captures/op92_ss_02/` (2026-07-05)
  - R: `godot-learning/src/scenarios/ScenarioApply.gd` `inflict_status` (Status=2 → `world.inflict_poison_critical`, `src/scenarios/ScenarioWorld.gd`), Critical kneel forced via `src/animation/AnimationStateController.gd` `to_critical_idle` — validated by `tests/ScenarioInflictStatusTest.gd` `_test_status2_inflicts_poison_critical` / `_test_status2_poisons_and_kneels`
  - src: `research/working_documents/op92_status_probe/README.md`
- **`SS=0x91` (Frog) is a no-op on stock ROM — it falls through, empirically confirming the `0x80+` single-status range is dead code without the EIUH hack.** — `[S·D·R] 3/3`
  - S: task-body branch structure static disasm — `SS ∉ {0,1,2}` falls through every branch to `event_fiber_yield_n` → `event_fiber_mark_complete`, touching no status bit, no `+0x3b1`, no anim (`inflict_status_op92_decode.md` §11.2, 2026-07-05)
  - D: op92_status_probe `SS=0x91` capture, `captures/op92_ss_91/` (2026-07-05)
  - R: `godot-learning/src/scenarios/ScenarioVM.gd` `_op_inflict_status` encodes the 0x80+ range as dead code on stock and fails loud + halts instead of no-op (issue #154) — validated by `tests/ScenarioInflictStatusTest.gd` `_test_unmodelled_status_fails_loud`
  - src: `research/working_documents/op92_status_probe/README.md`
- **`SS=0` on healthy units takes the `v1==2` path — no status bits touched.** — `[S·D·R] 3/3`
  - S: `FUN_80148E88` HP-state selector static disasm — `v1` from render struct `+0x58 & 0x20` (Dead) / `+0x5a & 0x01` (Critical) / else healthy; `v1==2` registers pose task `FUN_8008be04` and commits, zero bit-writer calls (`inflict_status_op92_decode.md` §11.3, 2026-07-05)
  - D: op92_status_probe `SS=0` capture, `captures/op92_ss_00/` (2026-07-05)
  - R: `godot-learning/src/scenarios/ScenarioApply.gd` `inflict_status` (Status=0 → `world.revive_and_normalise`: revive-if-dead + normalise to Standing, explicitly "NOT a status bit") — validated by `tests/ScenarioInflictStatusTest.gd` `_test_status0_does_not_halt` / `_test_living_unit_normalises_without_revive`
  - src: `research/working_documents/op92_status_probe/README.md`
- **Scenario 6 (abduct princess, pre-events) holds three `{92}` Inflict Status calls, the Status operand byte at `0x8004A713`, `0x8004A719`, `0x8004A71F` (op+3).** — `[S·D·R] 3/3`
  - S: `0x8004A713 / 0x8004A719 / 0x8004A71F` (op+3 Status-byte sites), scenario 6 event code — named in `research/working_documents/op92_status_probe/README.md`
  - S: chunk base `0x8004A6BC` (`g_event_chunk_base @ 0x80173CA4`), sites at chunk offsets `0x54 / 0x5A / 0x60`, 6 bytes apart per `0x8014d202 = 0x05` (`inflict_status_op92_decode.md` §1/§10)
  - D: op92_status_probe poked all three sites, captures (2026-07-05)
  - R: `godot-learning/tests/ScenarioInflictStatusTest.gd` `_test_real_chunk6_three_sites_do_not_halt`
  - src: `research/working_documents/op92_status_probe/README.md`
- **`{92}` is a 6-byte instruction: 5 operand bytes `[Unit:u16][0x00][Status:1][Wait:1][0x00]`, opsize-table entry `0x8014d202 = 0x05`, dispatcher advances the event PC by `optable[op] + 1` at the common exit `0x80145F24`.** — `[S·D·R] 3/3`
  - S: opsize table entry `0x8014d202` live read = `0x05`; PC advance `lbu v1, -0x2E90(at)` at common exit `0x80145F24` (optable base `0x8014D170`) — doc §2/§3.3
  - D: scenario 6 live RAM — three sites `0x8004A710 / 0x8004A716 / 0x8004A71C` exactly 6 bytes apart, chunk base `0x8004A6BC` (`g_event_chunk_base @ 0x80173CA4`) (2026-07-04, doc Added)
  - R: `godot-learning/assets/scenarios/event_instructions.json` `0x92` entry (`Unit:2 / Status:1 / Unknown:2` = 5 operand bytes) parsed by `src/scenarios/EventInstructionSet.gd` `args` — validated by `tests/ScenarioInflictStatusTest.gd` `_test_real_chunk6_three_sites_do_not_halt`
  - src: `research/working_documents/scenario_1_captures/inflict_status_op92_decode.md`
- **The real `{92}` handler is `0x80145EE0`, not the `0x80145EC0` a first static pass tagged: the event-VM dispatcher preloads each block's next compare constant in the previous `bne`'s delay slot, so a block's fall-through belongs to the *previous* constant — proven by breakpoint hit-count (H92=3 vs H98=0 in scenario 6).** — `[S·D] 2/3`
  - S: `0x80145EE0` (handler), `0x80145ED8` (`bne s4, v0(=0x92)` gate), `0x80145EC0` (the `0x98` handler), `FUN_80143bd8` dispatch chain — doc §3.0/§3.1
  - D: scenario 6 Exec-BP hit-counts `0x80145EE0` = 3, `0x80145EC0` = 0 (2026-07-04, doc Added)
  - R: none — MIPS dispatch not present in godot-learning (its `ScenarioVM` is a GDScript table dispatch, `_bind` in `src/scenarios/ScenarioVM.gd`)
  - src: `research/working_documents/scenario_1_captures/inflict_status_op92_decode.md`
- **The `{92}` handler does not apply the status inline — it spawns a fire-and-forget cooperative task: `FUN_80149BEC(0x10)` allocates, `FUN_8014C8A0` registers body `FUN_80148E88`, `FUN_8014CA38` inits the context with `a1` = pointer to the operands (opcode+1); the status change executes on a later scheduler tick.** — `[S·D] 2/3`
  - S: spawn sequence `0x80145EE0-0x80145F14`; `FUN_80149BEC` / `FUN_8014C8A0` / `FUN_8014CA38` / `FUN_80148E88` — doc §3.2
  - D: scenario 6 task-body run — per-site `s1`/`a1` = `0x8004A711 / 0x8004A717 / 0x8004A71D` (= opcode+1), task body `HTASK = 3` (2026-07-04, doc Added)
  - R: none — no cooperative task model in godot-learning; the port applies synchronously inside the VM tick (`src/scenarios/ScenarioApply.gd` `inflict_status`, the `_wait` operand has no analogue)
  - src: `research/working_documents/scenario_1_captures/inflict_status_op92_decode.md`
- **`SS=0` is revive/normalise: revive with 1 HP only if dead, set the Critical pose if HP is low else Standing, play a sound only if the unit was dead, and block later event HP-restoration on that unit within the event — global block flag `DAT_80165FB4` set to `0x41` plus a per-unit marker (`sh` at unit+0x28).** — `[S·D·R] 3/3`
  - S: `FUN_80148E88` SS=0 branch disasm — `0x80149024` (SS!=0 exit), `0x80149070` (`sh v0, 0x28(s0)` HP-restore marker), `0x8014908c`/`0x80149094` (`0x41` → `DAT_80165FB4`); dead sub-path (`v1==0`) removes Dead (idx 0x05, a2=0), adds Critical (idx 0x10, a2=1), `+0x3b1 ← 0x01` — doc §4.1/§11.3
  - D: scenario 6 SS=0 live run — healthy path took 0 bit-writer fires, render stayed `00`, `+0x3b1` stayed `0` (op92 capture, 2026-07-05); the dead sub-path was not dynamically reached (branch + register decode live-confirmed)
  - R: `godot-learning/src/scenarios/ScenarioWorld.gd` `revive_and_normalise` (`unit_stats.revive(1)` only if dead + `revive_to_idle()` IDLE re-resolve; `_hp_restore_blocked` recorded via `is_hp_restore_blocked` with no consumer yet; revive SFX a documented TODO) — validated by `tests/ScenarioInflictStatusTest.gd` `_test_dead_unit_revives_to_1hp_and_stands` / `_test_living_unit_normalises_without_revive`
  - src: `research/working_documents/scenario_1_captures/inflict_status_op92_decode.md`
- **`unit+0x3b1` is the death/pose-state byte: `0x00` healthy, `0x01` SS=0 dead-revive, `0x02` Crystal (SS=1), `0x03` Poison+Critical (SS=2).** — `[S·D] 2/3`
  - S: `FUN_80148E88` branch disasm `sb` writes (doc §4.1/§11.2)
  - D: op92 captures — `ss_00` `+0x3b1` = 0x00, `ss_01` = 0x02, `ss_02` = 0x03 (2026-07-05)
  - R: none — no `+0x3b1` analogue in godot-learning (state tracked via `unit_status` string Set + Unit-owned flags; probed `godot-learning/src` + `godot-learning/tests`)
  - src: `research/working_documents/scenario_1_captures/inflict_status_op92_decode.md`
- **The status-bit writer `FUN_80149100(a0=UnitID, a1=idx, a2=set-selector, a3=1)` splits the index into `byte = idx >> 3`, `bit = 1 << (idx & 7)`, clears both 5-byte scratch sets, then sets the bit into `+0x1a7` when `a2 != 0` (that set commits byte-for-byte to the render field `+0x58..+0x5c`) or into `+0x1ac` when `a2 == 0` (companion mask = remove) — the `beq s2, zero` at `0x80149180` inverts the target from the naive read.** — `[S·D] 2/3`
  - S: `0x80149100` + `0x80149180` disasm; commit via `FUN_8014ceb4` (doc §4.1/§11.5)
  - D: live Crystal `+0x1a7[0] = 0x40 → +0x58 = 0x40`; Poison `+0x1a7[3] = 0x80 → +0x5b = 0x80` (op92 captures, 2026-07-05)
  - R: none — no status-bit writer in godot-learning (statuses are string Sets, `unit_status.add_status`)
  - src: `research/working_documents/scenario_1_captures/inflict_status_op92_decode.md`
- **Any `{92}` in an event file is auto-applied at file load to any matching Jumping (`unit+0x5a & 0x40`) or Floating (`unit+0x58 & 0x04`) unit, even if the instruction is never reached — `FUN_8012d8e4` called once from `0x80143cb0`; the FFHacktics "Nyzer" patch NOPs that `jal` (a faithful port leaves it live).** — `[S] 1/3`
  - S: `0x80143cb0` (`jal FUN_8012d8e4`) + `FUN_8012d8e4` static disasm (doc §6)
  - R: none — no load-time auto-apply present in godot-learning
  - src: `research/working_documents/scenario_1_captures/inflict_status_op92_decode.md`
- **Stock FFT only ever uses `SS = 0/1/2`: a census of all 480 scenario chunks finds 257× `SS=0` (139 scenarios incl. scenario 6), 3× `SS=1` (194/479/480), 6× `SS=2` (354), 0× `0x80+` — the single-status range is never used in stock.** — `[S] 1/3`
  - S: byte-level census across all 480 scenario chunks, filtered to plausible instructions (`Unit < 256`, `SS ∈ {0,1,2}`, `Wait ∈ {1,12}`; the raw grep is polluted by linear-disassembly desync) — doc §9
  - R: none — census not encoded in godot-learning
  - src: `research/working_documents/scenario_1_captures/inflict_status_op92_decode.md`
- **The runtime status field's bit numbering is MSB-first within each byte (`1 << (idx & 7)` at byte `idx >> 3`): physical `+0x5b` bit 7 is Poison (runtime idx 0x1f) even though FFTPatcher's byte-3 bit 7 is "Wall" — the FFTPatcher `Statuses.cs` 40-status byte/bit table is the save-format layout, not the runtime field.** — `[S·D] 2/3`
  - S: live render `+0x5b & 0x80` = Poison (runtime idx 0x1f) vs FFTPatcher `Statuses.cs` byte3 bit7 "Wall" (doc §5.2/§11.1)
  - D: op92 `ss_02` capture `+0x5b = 0x80` (2026-07-05)
  - R: none — no runtime status bitfield present in godot-learning
  - src: `research/working_documents/scenario_1_captures/inflict_status_op92_decode.md`

## Notes

(empty — user territory)

## Related

- [[Event VM Index]]
- [[Event Opcode Catalog]]
- [[Poison Visual Recolor]]
- [[Crystal Status Visual]]
