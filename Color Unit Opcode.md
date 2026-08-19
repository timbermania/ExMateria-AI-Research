# Color Unit Opcode

Event instruction `0x32` "Color Unit" is the per-unit palette tint ramp of the scenario event VM: a 7-byte body (`Units, Multi, Color, R, G, B, Time`) where `Color` is a 0–10 blend mode shared with `0x33`/`0x1A` in `color_tint_blend_apply @ 0x8008F710`, R/G/B are signed 5-bit deltas, and `Time` is the ramp length in frames (`Time==0` snaps). It is the soft "fade" of the Orbonne door exits in scenario 1 — a per-actor `Color=1 Time=0` snap-to-half plus `Color=8 Time=2` ramp back to base, i.e. a dim flash, the sprite vanishing later via the separate `Erase Unit`. The full handler chain (`FUN_8013e904` → `FUN_800933c4` → `FUN_800931c4` → `FUN_8008f710`) is grounded by live PCSX-Redux captures (byte-exact chunk↔RAM match, six handler fires, a golden CLUT vector), and Godot implements it as `ScenarioVM._op_color_unit` + the `ScenarioColorTint` affine `(scale,bias)` model + `unit_tint_scale`/`unit_tint_bias` shader uniforms, validated by the golden-vector `ScenarioColorUnitTest`.

## Points

- **Opcode `0x32` "Color Unit" has a 7-byte body `Units, Multi, Color, R, G, B, Time` — `Color` is a 0–10 blend mode (NOT an RGB value), R/G/B are signed 5-bit deltas (read as `char`), and `Time` is the ramp length in frames with `Time==0` meaning apply instantly (snap) — live-verified by the in-RAM size-table entry `DAT_8014d170[0x32] = 7` (same table gives `[0x33]=5`, `[0x3D]=2`, `[0x46]=2`), which the interpreter uses to advance `pbVar4 += table[*pbVar4] + 1`.** — `[S·D·R] 3/3`
  - S: opcode-size table `DAT_8014d170` + `FUN_8013e904` case `0x32` (`battle_disassembly.txt` / `battle_decompilation.c`)
  - D: live bytecode capture @ `0x8004A8FE` = `32 17 00 01 00 00 00 00`, an exact match to `scenario_1_chunk.json` instr 101 (committed chunk == live PSX RAM); all four handler prologues verified against live RAM (savestate `orbonne_three_actors_walk_in.sstate`, 2026-06-30)
  - R: `godot-learning/src/scenarios/ScenarioDecode.gd` `color_unit` (7-byte decode) + `ScenarioVM.gd` `_op_color_unit` (bound to `EventInstruction.COLOR_UNIT`), validated by `godot-learning/tests/ScenarioColorUnitTest.gd` (registered in `tests/run_all_tests.sh`)
  - src: `research/working_documents/UNIT_FADE_COLOR_UNIT_OPCODE.md`
- **The `0x32` case of the main event interpreter `FUN_8013e904` resolves `Units`/`Multi` to a runtime slot via `FUN_80133158` (absent = sentinel `2000` → no-op), then dispatches the tint with `jal FUN_800933c4` @ `0x8013ED78`, passing `(Color, Time, slot, signed R, G, B)` — G/B are passed on the stack at `0x10(sp)`/`0x14(sp)`; the fan-out `FUN_800933c4` tints the single resolved unit for slot < `0x10` and all 16 units when handed a broadcast token ≥ `0x10`.** — `[S·D·R] 3/3`
  - S: case body `0x8013ED40–0x8013ED80` (compare `bne` @ `0x8013ED40`, `jal` @ `0x8013ED78`), prologue `0x8013E904` (`addiu sp,sp,-0x58`), fan-out `FUN_800933c4` @ `0x800933C4` (`battle_disassembly.txt`)
  - D: six live fires at handler BP `0x800933C4` on the Orbonne door exit, all returning to `0x8013ED80` (the dispatch `jal`), full 6-arg capture: slots `0x2/0x3/0x4` = event units `2/23/131` (savestate `orbonne_three_actors_walk_in.sstate`, 2026-06-30)
  - R: `godot-learning/src/scenarios/ScenarioVM.gd` `_op_color_unit` mirrors resolve→apply (`ScenarioApply.color_unit`); the ≥`0x10` all-16 broadcast fan-out is NOT ported (probed `godot-learning/src/scenarios/` + `godot-learning/tests/`) — validated by `godot-learning/tests/ScenarioColorUnitTest.gd` `_test_handler_registered_and_applies`
  - src: `research/working_documents/UNIT_FADE_COLOR_UNIT_OPCODE.md`
- **The Orbonne door-exit "fade" is a dim flash, not a smooth fade-out: each actor takes a `Color=1, Time=0` snap (per-channel `>>1` halve) immediately followed by a `Color=8, Time=2` ramp that brightens each CLUT entry linearly back to its base value over the 8-frame fast DDA table, landing exactly on base — so the perceived "they fade away" is that dip plus the trailing `Erase Unit` and doorway shadow, not the tint alone (the tint does not hide the sprite).** — `[S·D·R] 3/3`
  - S: mode formulas in `color_tint_blend_apply @ 0x8008F710` (`battle_decompilation.c`); Exit-A pairs at `godot-learning/assets/scenarios/scenario_1_chunk.json` instrs 101/106, 116/119, 129/132
  - D: live golden CLUT capture of unit 23 (slot 3) bank-3 entries, byte-exact halve `0x90A5→0x8842`, `0xD77C→0xA9AE`, `0xCEB7→0xA54B`, then the mode-8/T2 ramp landing exactly on base (savestate `orbonne_three_actors_walk_in.sstate`, 2026-06-30)
  - R: `godot-learning/src/scenarios/ScenarioColorTint.gd` (affine `(scale,bias)` model + fast/slow ramp timing) driven by `ScenarioVM.gd` `_op_color_unit` (ramps tick in `_tick_once`), validated by `godot-learning/tests/ScenarioColorUnitTest.gd` golden-vector tests (`_test_mode1_snap_halves_golden`, `_test_mode8_restores_to_base`, `_test_fast_ramp_is_eight_frames`) + headful `ScenarioPlayer.tscn` dispatch of unit 23 `mode1/T0`+`mode8/T2` (no errors)
  - src: `research/working_documents/UNIT_FADE_COLOR_UNIT_OPCODE.md`
- **Scenario 1 (Orbonne) contains two door-exit fades built from the same opcodes: Exit A (Priest + two attendants, chunk instrs 101–132) gives each actor a `Color=1 Time=0` + `Color=8 Time=2` pair, while Exit B (later group, instrs 310–401) gives each actor a single `Color=1 Time=2` as it reaches the threshold, then a ~21-frame Wait, then `Erase Unit` (`0x46`) — the fade and the disappearance are two distinct events (the sprite stays visible ~20 more frames after the tint fires), with `Remove Unit` (`0x3D`, full despawn) used only at the end of Exit B (instrs 382, 401).** — `[S·D·R] 3/3`
  - S: `godot-learning/assets/scenarios/scenario_1_chunk.json` instrs 101–132 and 310–401
  - D: six live Color Unit fires + raw bytecode @ `0x8004A8FE` matching chunk instr 101; before/after screenshots (Priest + 2 attendants on the dais → next beat's cast) (savestate `orbonne_three_actors_walk_in.sstate`, 2026-06-30)
  - R: `godot-learning/assets/scenarios/scenario_1_chunk.json` drives the same bytecode through `_op_color_unit`; headful `ScenarioPlayer.tscn` replays the Exit-A pair for unit 23 (`0x02`/`0x83` included) matching the live capture — `godot-learning/tests/ScenarioColorUnitTest.gd`
  - src: `research/working_documents/UNIT_FADE_COLOR_UNIT_OPCODE.md`
- **For RGB=(0,0,0) the whole-palette CLUT tint collapses to a single affine map `tinted = base·scale + bias` (mode 1 ⇒ `scale=0.5`; mode 8 ⇒ `scale=1, bias=0`), reproducible within 5-bit rounding without rewriting the CLUT — which is why Godot applies the unit tint as a shader-side affine on the sampled palette colour: `ALBEDO = ALBEDO * unit_tint_scale + unit_tint_bias` in `unit.gdshader` (orthogonal to the spell-flash `unit_tint`), with the non-affine luma modes 2/3/6/7 warned and falling back to additive.** — `[S·D·R] 3/3`
  - S: mode 1 = `(cur>>1)+Δ`, mode 8 = palette absolute — `color_tint_blend_apply @ 0x8008F710` mode switch (`battle_decompilation.c`)
  - D: live golden CLUT capture of unit 23 — the whole-palette uniform scaling under the mode-1 halve and the mode-8 restore-to-base (savestate `orbonne_three_actors_walk_in.sstate`, 2026-06-30)
  - R: `godot-learning/assets/shaders/unit.gdshader` (`unit_tint_scale`/`unit_tint_bias` uniforms applied to the sampled palette colour), `godot-learning/src/scenarios/ScenarioColorTint.gd` (luma-mode fallback), validated by `godot-learning/tests/ScenarioColorUnitTest.gd` (`_test_luma_mode_flagged` + the golden-vector tests)
  - src: `research/working_documents/UNIT_FADE_COLOR_UNIT_OPCODE.md`

## Notes

(empty — user territory)

## Related

- [[Color Tint Luma Modes]]
- [[Color Screen Opcode]]
- [[Reset Palette Opcode]]
- [[Unit Visibility Flag]]
- [[Event Unit Selector]]
- [[Event Opcode Catalog]]
