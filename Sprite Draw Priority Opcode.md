# Sprite Draw Priority Opcode

Opcode `{71}` Raise Unit Draw Priority is a Z-order opcode: it unlinks the unit's node from the global sprite draw-order list (`unit_sprite_list_head` @ `0x80098A54`) and re-inserts it at the head, raising that unit's sprite to topmost draw priority (the head is drawn last / topmost). Static + dynamic analysis complete (2026-07-10, scenario 6): it fires on the exact units about to be `Sprite Move`d (the abduction carry/toss), so they layer in front during the motion. Godot no-ops it — sprites depth-sort by real 3D position (ADR-0009), so there is no draw list to reorder.

## Points

- **Opcode `{71}` (operand Unit u16) raises the unit's sprite to the topmost draw priority: dispatch at `0x80144898` resolves the unit id to a roster index via `FUN_80133158` (`0x7d0` = unit not loaded) and `FUN_8007a7b8` unlinks the node from the `unit_sprite_list_head` singly-linked list (`0x80098A54`) and re-inserts it at the head — the list is the sprite draw order, head = drawn last/topmost — so the unit layers on top during the `Sprite Move` that immediately follows it (scenario 6: abduction carry/toss, units 5/12/139 at pcs 1061/1744/1747/1750).** — `[S·D] 2/3`
  - S: dispatch `0x80144898`, resolver `FUN_80133158` @ `0x80133158`, action `FUN_8007a7b8` @ `0x8007A7B8`, list head `unit_sprite_list_head` @ `0x80098A54` (= `0x800A0000 − 0x75AC`; `project-assets/fft-rom/battle_disassembly.txt`)
  - D: BPs before (`0x80144898`) / after (`0x801448bc`) the reorder, scenario-6 live run — `head_before → head_after` flips to the reordered unit's node each call, `ret=1` (2026-07-10)
  - R: none — no draw-list reorder in godot-learning; `0x71` is named `RAISE_UNIT_DRAW_PRIORITY` and bound to `_skip` in `godot-learning/src/scenarios/ScenarioVM.gd` ("Godot 3D depth sort (ADR-0009) already orders sprites"; probed `godot-learning/src/`, `godot-learning/tests/`)
  - src: `research/working_documents/SCENARIO6_UNKNOWN_OPCODES_6D_71_7C_82_INVESTIGATION.md`

## Notes

(empty — user territory)

## Related

- [[Event Opcode Catalog]]
- [[Unit Sprite Render Pipeline]]
- [[Scenario 6 Punch Pickup Throw]]
- [[Scenario 6 Ride Off]]
- [[Scenario 6 Carry Composition]]
