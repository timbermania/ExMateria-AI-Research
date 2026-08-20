# Wait Rotate Unit Opcode

Event instructions `{64} Wait Rotate Unit` and `{65} Wait Rotate All` are coroutine barriers that block the calling scenario script until a rotation finishes. Both share one PSX handler, `FUN_801498fc @ 0x801498fc`, which yields a frame at a time and re-polls the `+4` active byte of the target unit's 7-byte pending-rotate command (`0x8016d9dc + handle*7`) until it is zero; that byte is set to 1 by `{2D}` Rotate Unit and cleared to 0 by the per-vsync consumer `FUN_8013f20c` the moment the unit's facing reaches the target. The handler was fully statically decoded (2026-06-28) and the polled signal live-validated, though the handler itself was never caught executing (no available savestate reaches the Setup scenario's dense `{64}` runs). Godot mirrors the barrier as a generic `wait_until` predicate hold on `Unit._rotate_state` with a watchdog force-release that PSX lacks.

## Points

- **`{64}` Wait Rotate Unit and `{65}` Wait Rotate All share handler `FUN_801498fc @ 0x801498fc`: `{64}` is a 3-byte instruction `[64][unitID][00]` (length table `[0x8014d1d4] = 2`; dispatcher case body `0x8013ecfc` loads the unit id from `[s2+1]`), `{65}` is a single byte with `a0` hardcoded `-1` (case body `0x8013ed18`); the id resolves to a roster handle via `FUN_80133158`, the `0x7d0` sentinel (not deployed) returns immediately without blocking, and `{65}` loops all 21 handles, done only when every `+4` flag is 0.** — `[S·R] 2/3`
  - S: `FUN_801498fc @ 0x801498fc` (specific path `0x8014995c`–`0x80149998`, ALL path via `bne a0,-1`), dispatcher case bodies `0x8013ecfc` / `0x8013ed18` (dispatcher `FUN_8013e904`, length table base `0x8014d170`), resolver `FUN_80133158`, sentinel `0x7d0` (static decode on real-RAM disassembly, 2026-06-28, in `HANDOFF_wait_rotate_unit.md`)
  - R: `godot-learning/src/scenarios/ScenarioVM.gd` (`_op_wait_rotate_unit` :3248, `_op_wait_rotate_all` :3255, `_rotate_done`/`_all_rotations_done` with the not-deployed-unit short-circuit) — validated by `godot-learning/tests/ScenarioWaitRotateTest.gd` (holds-then-resumes, absent-unit-does-not-block, all-done)
  - src: `research/working_documents/scenario_1_captures/HANDOFF_wait_rotate_unit.md`
- **The exact "rotation done" signal `{64}`/`{65}` block on is the `+4` active byte at `0x8016d9dc + handle*7`: `{2D}` Rotate Unit arms the command and sets `+4 = 1` (store at `0x80148414`), and the per-vsync consumer `FUN_8013f20c` clears it (`sb zero` @ `0x8013f2e4`) precisely when the current facing `unit+0x70` equals the `+0` target byte.** — `[S·D·R] 3/3`
  - S: `0x80148414` (`{2D}` `sb s4(=1)` into `+4`), consumer active-read `0x8013f2c0`, target compare `0x8013f2d8`, completion store `0x8013f2e4` (static decode on real-RAM disassembly, 2026-06-28, in `HANDOFF_wait_rotate_unit.md`)
  - D: `scripts/probe_wait_rotate.py` completion-signal pass on `orbonne_prayer_mid_dialog.sstate` (2026-06-28): BPs at `0x80148414` (`s4=1`, tgt=04/02) and `0x8013f2e4` (fires when `a0(cur)==target`, `flag_before=01`); the 1→0 flag transitions coincided exactly with the facing reaching target (h1 @ 0x0400, h0 @ 0x0200)
  - R: `godot-learning/src/units/Unit.gd` (`_tick_rotate` clears `_rotate_state` when `cur_byte == target_byte` — the exact analogue of the `+4` clear) — validated by `godot-learning/tests/UnitScenarioRotateTest.gd` and `tests/ScenarioWaitRotateTest.gd`
  - src: `research/working_documents/scenario_1_captures/HANDOFF_wait_rotate_unit.md`
- **The blocking mechanism is a cooperative coroutine poll loop: the handler loops `jal FUN_8014ca80` (`0x8014ca80`) — a real context switch that saves s0–s8/k0/k1/gp/sp/ra to `0x8016986c + thread*0x400 + 0x10`, bumps the thread index `DAT_80174038` mod 16, advances the frame (`FUN_80142ca8`), and resumes the next thread — re-reading `[0x8016d9dc + s0]` on each wake; the handler blocks inside its own frame and the script PC `s2` is not rewound.** — `[S·R] 2/3`
  - S: poll loop `0x80149978`/`0x80149988`/`0x80149990`, yield `FUN_8014ca80 @ 0x8014ca80`, save-block base `0x8016986c`, slot index `DAT_80174038`, frame advance `FUN_80142ca8` (static decode on real-RAM disassembly, 2026-06-28, in `HANDOFF_wait_rotate_unit.md`)
  - R: `godot-learning/src/scenarios/ScenarioVM.gd` (`_hold_wait_until`/`_arm_wait_until` :1636–1676 hold the calling context each VM tick while the predicate is false — the Godot analogue of yield-and-re-poll with the script position left in place) — validated by `godot-learning/tests/ScenarioWaitRotateTest.gd`
  - src: `research/working_documents/scenario_1_captures/HANDOFF_wait_rotate_unit.md`
- **Godot implements `{64}`/`{65}` on the generic `wait_until` predicate-hold primitive (the context is held each tick while the predicate is false) rather than a precomputed remaining-tick duration, and adds a watchdog (`wait_until_deadline_tick` force-release + `push_warning`) because PSX has no watchdog — it trusts the consumer to clear the flag.** — `[R] 1/3`
  - R: `godot-learning/src/scenarios/ScenarioVM.gd` (`_arm_wait_until` :1669, watchdog in `_hold_wait_until` :1641–1658, `{64}`/`{65}` registered :767–768) — validated by `godot-learning/tests/ScenarioWaitRotateTest.gd` (incl. `_test_wait_until_watchdog_force_releases`)
  - src: `research/working_documents/scenario_1_captures/HANDOFF_wait_rotate_unit.md`

## Notes

(empty — user territory)

## Related

- [[Rotate Unit Interpolation]]
- [[Scenario Wait Semantics]]
- [[Event Opcode Catalog]]
- [[Event VM Index]]
