# Face Unit Opcode

Event opcode `{53} Face Unit` makes the selected unit(s) rotate to look at the Faced Unit's tile. `evt0x53_face_unit_handler` @ `0x80148084` (dispatched with `a1 = 1` from the per-unit interpreter case body `0x8013ec18`) computes a 12-bit look-at facing per affected unit from the affected-minus-faced tile delta via the PSX arctangent `SUB_8001d8e8`, then arms a non-blocking 7-byte rotate command in the shared `pending_rotate_command_array` @ `0x8016d9d8 + handle*7` — the same queue and per-vsync consumer `FUN_8013f20c` that `{2D} Rotate Unit` uses — and returns without animating. Fully decoded from `battle_disassembly.txt` and dynamically validated against live PSX RAM at the scenario 1 Orbonne beat (the written `0x04` facing nibble matches the math), the 12-bit arctangent wheel characterized by controlled calls, operand layout confirmed byte-exact, and the look-at math ported to godot-learning (2026-06-29) with regression tests. Companion opcode `{2C} Face Unit 2` is the SAME handler dispatched from the scenario interpreter with `a1 = 0`, which additionally turns the faced unit back so the two units face each other — ported and headful-verified 2026-07-02.

## Points

- **`{53}` Face Unit's operand layout is opcode + 7 payload bytes: `Faced Unit` u16 LE, `Units`, `Multi`, `Direction` (0=shortest, 1=CW, 2=CCW), `Speed`, `Delay` — confirmed byte-exact against `event_opcodes.json` and the live payload; the wiki's separate "x00" operand is just the high byte of the u16 Faced Unit.** — `[S·D·R] 3/3`
  - S: `evt0x53_face_unit_handler` @ `0x80148084` operand reads (`event_bytecode_reader_c` @ `0x801480ac`/`0x801480b8`), `event_opcodes.json` `0x53` row (`battle_disassembly.txt`, BATTLE.BIN base `0x80067000`)
  - D: scenario 1 Orbonne capture — savestate `reference-assets/orbonne_female_knight_face_unit.sstate`, live payload `84 00 0c 00 00 00 00` via `_fu_capture1.py` (2026-06-29)
  - R: `godot-learning/src/scenarios/ScenarioVM.gd` `_op_face_unit` → `ScenarioApply.face_unit` (reads `Faced Unit`/`Units`/`Multi`/`Direction`/`Speed`/`Delay`) + `godot-learning/tests/ScenarioFaceUnitTest.gd`
  - src: `research/working_documents/scenario_1_captures/face_unit_decode.md`
- **Face Unit is non-blocking, fire-and-arm: the handler computes each affected unit's look-at facing (skipping affected units equal to the faced unit) and writes a 7-byte rotate command into the shared `pending_rotate_command_array` @ `0x8016d9d8 + handle*7` (the same queue `{2D}` Rotate Unit writes), then returns without animating — the per-vsync consumer `FUN_8013f20c` steps the facing one 16-direction notch per frame, and `{64}`/`{65} Wait Rotate` block on the slot's +4 active flag.** — `[S·D·R] 3/3`
  - S: command write `0x801481cc`–`0x80148238` (stores into `0x8016d9d8 + handle*7`), affected==faced skip @ `0x80148128` (`battle_disassembly.txt`)
  - D: `_fu_capture1.py` `WRITE2 cmd=8016da2c handle=12 facing=04` — unit `0x0c` → handle 12, `0x8016d9d8 + 12*7 = 0x8016da2c` (2026-06-29)
  - R: `godot-learning/src/scenarios/ScenarioVM.gd` `_do_face_unit` → `Unit.scenario_rotate` / `_tick_rotate` stepper (`godot-learning/src/units/Unit.gd`) + `godot-learning/tests/ScenarioFaceUnitTest.gd`
  - src: `research/working_documents/scenario_1_captures/face_unit_decode.md`
- **The target facing is `facing12 = ((0x1400 − atan) & 0xF00)` where `atan = SUB_8001d8e8(Δx, Δy)` is the PSX 12-bit arctangent of the (affected − faced) tile delta in PSX-internal coords — Δx = affected.x − faced.x, Δy = affected.y − faced.y; the low nibble of `facing12` (`>>8`) is the 16-direction wheel index that lands in the command's +0 byte.** — `[S·D·R] 3/3`
  - S: `0x80148144`–`0x80148178` (delta reads, `atan` call @ `0x80148158`, `0x1400 − atan` fold + `& 0xF00`) (`battle_disassembly.txt`)
  - D: scenario 1 capture geometry — affected (126,182) vs faced (126,42): Δx=0, Δy=+140 → atan `0x000` → `((0x1400−0)&0xF00)>>8 = 0x04` = 12-bit `0x400`, exactly the nibble PCSX wrote (`_fu_capture1.py`, 2026-06-29)
  - R: `godot-learning/src/scenarios/PsxNum.gd` `look_at_12bit` + `godot-learning/src/scenarios/ScenarioVM.gd` `face_unit_look_at_12bit_psx` / `_face_unit_look_at_12bit` (`godot-learning/tests/PsxNumTest.gd`, `godot-learning/tests/ScenarioFaceUnitTest.gd` §8 octant-table assertions)
  - src: `research/working_documents/scenario_1_captures/face_unit_decode.md`
- **`SUB_8001d8e8(Δx, Δy)` is a 12-bit clockwise arctangent over `0..0xFFF == 360°`: `0` along +Δy, `0x400` along +Δx, `0x800` along −Δy, `0xC00` along −Δx (clockwise wheel from +Δy toward +Δx).** — `[S·D·R] 3/3`
  - S: call site `0x80148158` in `evt0x53_face_unit_handler` (`battle_disassembly.txt`)
  - D: controlled-call characterization `_fu_atan_char.py` — emulator paused, `pc=0x8001d8e8` forced with `a0=Δx, a1=Δy`, `v0` read back; 8-row octant table incl. (100,100)→`0x0200`, (−100,−100)→`0xfa00` (2026-06-29)
  - R: `godot-learning/src/scenarios/PsxNum.gd` `look_at_12bit` reproduces the wheel (`godot-learning/tests/PsxNumTest.gd` pins it against `ScenarioVM.face_unit_look_at_12bit_psx`)
  - src: `research/working_documents/scenario_1_captures/face_unit_decode.md`
- **PSX-internal coordinates (returned by `FUN_80133088`) are the transpose of the chunk's Warp `X`(column)/`Y`(row) param naming — PSX-x = depth/row, PSX-y = lateral/column — so the Godot→PSX look-at mapping is PSX_Δy = +ΔGodot_x, PSX_Δx = −ΔGodot_z (ADR-0052: PSX +depth → Godot −Z); a naive no-transpose mapping yields `0x000` (South) instead of the correct `0x400` (East).** — `[D·R] 2/3`
  - D: scenario 1 capture (2026-06-29): two onlookers on the SAME chunk row read back with identical PSX-x = 126 and differing PSY-y = 42/182; live Godot run (affected tile (6,5) vs faced tile (0,5)) → PSX (Δx=0, Δy=+6) → `0x400`, byte-matching PCSX's written nibble and the sibling `{2D}` ops in the same beat
  - R: `godot-learning/src/scenarios/ScenarioVM.gd` `_face_unit_look_at_12bit` (Godot→PSX transpose mapper) + `godot-learning/tests/ScenarioFaceUnitTest.gd` (pins the captured-geometry transpose and the no-transpose `0x000` bug case)
  - src: `research/working_documents/scenario_1_captures/face_unit_decode.md`
- **`{2C} Face Unit 2` is not a distinct handler: `event_scenario_interpreter` @ `0x80143bd8` dispatches it at `LAB_80144dec` by calling the SAME `evt0x53_face_unit_handler` @ `0x80148084` with `a1 = 0` (`_clear a1` @ `0x80144dfc`) and the same 7-byte operand layout; `a1 = 0` additionally writes the FACED unit's rotate command (extra block @ `0x80148180`, executed only when `s5 == 0` at the `0x80148178` branch) with facing `F_A = F_B + 180°`, so the two units end up facing each other (the faced unit's command is overwritten each loop iteration, so with a team set it faces the last affected unit).** — `[S·R] 2/3`
  - S: dispatch `LAB_80144dec`–`0x80144dfc`, handler branch `0x80148178` (`bne s5,zero,LAB_801481cc`), faced-unit write block `0x80148180` (`battle_disassembly.txt`)
  - R: `godot-learning/src/scenarios/ScenarioVM.gd` `_op_face_unit_2` → `_do_face_unit(inst, true)` + `Unit.scenario_rotate` (`godot-learning/tests/ScenarioFaceUnitTest.gd` `_test_face_unit_single_vs_mutual` — 20/0; `godot-learning/tests/ScenarioApplyTest.gd` `_test_face_unit_single_and_mutual`; headful scenario 8 live 2026-07-02: `affected=0x84 faced=0x81 → aff=0xC00 faced=0x400` and `affected=0x85 faced=0x80 → aff=0x400 faced=0xC00`, mutual 180° pairs)
  - src: `research/working_documents/scenario_1_captures/face_unit_decode.md`
- **The `Delay` operand is NOT indexed into the wiki's Delay table — the handler computes `cmd+6 = (Delay × counter) >> 2` (≈ `Delay >> 2` frames in the common single-unit case where counter = 1); the wiki's "Delay → frames" column (`0x00→1 … 0xFC→64`) is exactly `(Delay >> 2) + 1`, a faithful description of the consumer's countdown expressed as a table.** — `[S·R] 2/3`
  - S: `0x80148218`–`0x80148238` (`mult` with loop counter, `sra v0,v0,0x2`, store to `cmd+6`) (`battle_disassembly.txt`)
  - R: `godot-learning/src/units/Unit.gd` `scenario_rotate` arms `delay_remaining = delay >> 2` (line 1632) + `godot-learning/tests/ScenarioFaceUnitTest.gd`
  - src: `research/working_documents/scenario_1_captures/face_unit_decode.md`

## Notes

(empty — user territory)

## Related

- [[Rotate Unit Interpolation]]
- [[Wait Rotate Unit Opcode]]
- [[Face Tile Opcode]]
- [[Event Unit Selector]]
- [[Unit Anim Rotate Opcode]]
- [[Event Opcode Catalog]]
