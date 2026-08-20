# Unit Anim Rotate Opcode

Event opcode `{8C} Unit Anim Rotate` snaps a unit's facing instantly and then plays an animation: handler `0x80147fac` writes the absolute `Direction` facing nibble (12-bit `Direction << 8`, same 16-direction wheel as `{2D}`) into the rotate queue's slot[0] while CLEARING the +4 active flag — so no interpolation happens, unlike `{2D}`/`{53}` — then calls `BATTLE_set_unit_anim_value(Animation)`, the `{11}` Unit Anim path. Decoded from the disassembly (2026-06-29) when it was the next ScenarioVM halt after `{53}` Face Unit in the scenario 1 Orbonne turn-to-face beat, and ported to godot-learning as an instant facing snap plus anim latch; the beat then plays through instructions 261→319 live.

## Points

- **`{8C} Unit Anim Rotate` is handled at `0x80147fac` (dispatched from BOTH the per-unit interpreter `0x8013ed30` and the scenario interpreter `0x80145cec`): it writes the absolute facing nibble (`Direction << 8`, same wheel as `{2D}`) to the rotate queue's slot[0] but clears the +4 active flag → INSTANT facing set (no interpolation), then calls `BATTLE_set_unit_anim_value(Animation)` (= the `{11}` path); the `Unknown` byte toggles a misc move-flag bit (advisory).** — `[S·R] 2/3`
  - S: handler `0x80147fac`, dispatch sites `0x8013ed30` / `0x80145cec` (the only two `jal FUN_80147fac` sites in BATTLE.BIN) (`battle_disassembly.txt`)
  - R: `godot-learning/src/scenarios/ScenarioVM.gd` `_op_unit_anim_rotate` + `godot-learning/src/units/Unit.gd` `scenario_set_facing` (instant snap); validated by the scenario-1 headful run playing the whole turn-to-face beat instr 261→319 (no dedicated test named in the doc)
  - src: `research/working_documents/scenario_1_captures/face_unit_decode.md`

## Notes

(empty — user territory)

## Related

- [[Unit Anim Opcode]]
- [[Face Unit Opcode]]
- [[Rotate Unit Interpolation]]
