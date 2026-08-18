# Event Unit Selector

The event unit-selector operand (`V = Units | Multi<<8`) shared by `{11}` Unit Anim, `{2D}` Rotate Unit, `{53}`, `{32}` Color Unit, and March is resolved by `resolve_event_unit_handle` into a mode that drives a 21-slot membership loop in the per-frame color command processor; `V == 0` is PSX's broadcast-to-all-existing-units path (mode 1), not single unit 0 — which Godot's `ScenarioDecode.unit_set_mode` now maps to `UnitSetMode.ALL`.

## Points

- **`resolve_event_unit_handle @ 0x80147928` maps selector `V` (`Units | Multi<<8`) to a mode: `0 < V < 0x100` → mode 0 (single unit id V, loop breaks after one), `V ≥ 0x100` → mode `V − 0xFE` (team-set broadcast), `V == 0` → mode 1 (broadcast — all units).** — `[S·D·R] 3/3`
  - S: `resolve_event_unit_handle @ 0x80147928` (`battle_decompilation.c`)
  - D: scenario-8 read-only RAM/VRAM capture (2026-07-10): 81 view-3 unit-palette entries consistent with an all-present-units broadcast
  - R: `godot-learning/src/scenarios/ScenarioDecode.gd` `unit_set_mode`, validated by `godot-learning/tests/ScenarioDecodeTest.gd`
  - src: `research/working_documents/COLOR_TINT_LUMA_MODE_SEPIA.md`
- **`FUN_801479ac @ 0x801479AC` is the per-candidate membership test for the 21-slot selector loop: mode 1 = any existing unit (`unit_sprite_object_exists` only, no team/alive filter), 2 = team A/player (`(unit+5) & 0x30 == 0`), 3 = team A + alive, 4 = team B/enemy (`(unit+5) & 0x30 != 0`), 5 = team B + alive.** — `[S] 1/3`
  - S: `FUN_801479ac @ 0x801479AC` (`battle_decompilation.c`)
  - src: `research/working_documents/COLOR_TINT_LUMA_MODE_SEPIA.md`
- **An event `{32}` reaches many units via the selector membership loop in `event_color_command_processor @ 0x801495E0` — read selector, resolve, loop up to 21 candidate slots, one `color_unit_fanout` call per member, break if mode==0 — which is distinct from `color_unit_fanout`'s own all-16 branch (param ≥ 0x10) that fires only for effect/charge callers; the event path always passes a single resolved handle per call.** — `[S] 1/3`
  - S: `event_color_command_processor @ 0x801495E0`, `color_unit_fanout @ 0x800933C4` (`battle_decompilation.c`)
  - src: `research/working_documents/COLOR_TINT_LUMA_MODE_SEPIA.md`
- **scn8 PC29's `{32}` has `Units=0, Multi=0` → `V=0` → mode 1, so it tints ALL present units (PSX cannot single-target unit 0 — id 0 always broadcasts); Godot's `ScenarioDecode.unit_set_mode(0, 0)` now returns `UnitSetMode.ALL` (was `SINGLE`), and because this is the shared `{11}`/`{2D}`/`{53}`/`{32}`/March selector, `(0,0)` now broadcasts for all of them with zero new failures in a full scenario regression sweep.** — `[S·D·R] 3/3`
  - S: `resolve_event_unit_handle @ 0x80147928` V==0→mode-1 fall-through + PC29 operands (`battle_decompilation.c`, `godot-learning/assets/scenarios/chunks/scenario_008_chunk.json`)
  - D: scenario-8 read-only RAM/VRAM capture (2026-07-10): 81 view-3 entries (many units) confirm the all-present-units broadcast
  - R: `godot-learning/src/scenarios/ScenarioDecode.gd` `unit_set_mode`, validated by `godot-learning/tests/ScenarioDecodeTest.gd` `_test_unit_set_mode` (`(0,0)→ALL`)
  - src: `research/working_documents/COLOR_TINT_LUMA_MODE_SEPIA.md`

## Notes

(empty — user territory)

## Related

- [[Color Tint Luma Modes]]
- [[Event Opcode Catalog]]
- [[Unit Anim Opcode]]
- [[Rotate Unit Interpolation]]
