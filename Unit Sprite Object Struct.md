# Unit Sprite Object Struct

The battle unit-sprite object (struct A) lives in an array at base `0x800B7308`, stride `0x440` (~16 slots), singly linked from `unit_sprite_list_head @ 0x80098A54`. The fields that read a unit's pose live: `+0x04` slot index, `+0x06` unit id (the sprite-type index `FUN_80082eec` branches on), `+0x0C` pending-pose latch (u16 = anim_id+1, normally 0), `+0x70` 12-bit facing, `+0x140/+0x144` status flag words, `+0x1DC` active anim id, `+0x1E0` SEQ frame index, `+0x1E2` frame timer (nonzero = cycling, 0 = static hold) — so "is this unit marching in place?" = `+0x1DC` is an idle anim (6/7 in Gariland) AND `+0x1E2 ≠ 0`. Animation changes commit in two stages: `set_unit_animation_with_flags @ 0x80081978` latches `+0x0C` now, and the next-frame consumer `FUN_80085C0C` paints `+0x1DC` and clears the latch — verified on the single PC23→PC24 March release in Gariland (2026-08-01).

## Points

- **The battle unit-sprite object (struct A) array is base `0x800B7308`, stride `0x440`, ~16 slots, singly linked from `unit_sprite_list_head @ 0x80098A54`; key fields: `+0x04` slot index (u8, matches the array index), `+0x06` unit id (u8, origin/ENTD id = sprite-type index in `FUN_80082eec`), `+0x0C` pending-pose latch (u16 = anim_id + 1, normally 0), `+0x70` facing (u16, 12-bit angle), `+0x140/+0x144` status flag words (u32, branched on by `FUN_80082eec`), `+0x1DC` active anim id (u16, the painted current anim), `+0x1E0` SEQ frame index, `+0x1E2` frame timer (nonzero ⇒ cycling, 0 ⇒ static hold) — so "is this unit marching in place?" = `+0x1DC` is an idle anim (6/7 in Gariland) AND `+0x1E2 ≠ 0`, while a frozen cinematic pose reads a held anim (e.g. 4) with `+0x1E2 == 0`.** — `[S·D] 2/3`
  - S: latch writer `0x8008197C` (`+0x0C` = anim_id+1), facing write `0x80081984` (`+0x70`), per-frame SEQ-advance write `0x800851FC` (`+0x1DC`), list head `0x80098A54` (BATTLE.BIN disassembly)
  - D: live RAM reads across `before_magic_city`, `magic_city_pc_0_jump_back`, `magic_city_march_pc_22` (`fft-monorepo-game/reference-assets/`, pcsx-redux port 8080, 2026-08-01)
  - R: none — ROM unit-sprite struct offsets not present in godot-learning (port keeps its own `Unit`/`ScenarioWorld` state; probed `godot-learning/src/`, `godot-learning/tests/`)
  - src: `research/working_documents/MARCH_OPCODE_80_SEMANTICS.md`
- **Animation changes commit in two stages — latch now, paint next frame: `set_unit_animation_with_flags @ 0x80081978` writes the `+0x0C` latch (`anim_id + 1`) at `0x8008197C`; the next-frame consumer `FUN_80085C0C` paints the real anim id into `+0x1DC` (write at `0x800861B8`) and then clears the latch (write at `0x80086238`); after a release the per-frame SEQ advance keeps writing `+0x1DC` at `0x800851FC` — the full observed PC23→PC24 order is handler `0x80149490` → latch `0x8008197C` → paint `0x800861B8` → latch clear `0x80086238` → cycling writes `0x800851FC`.** — `[S·D] 2/3`
  - S: `0x8008197C`, `FUN_80085C0C`, `0x800861B8`, `0x80086238`, `0x800851FC` (BATTLE.BIN disassembly)
  - D: Write/Exec-BP capture of the single PC23→PC24 March transition, chronological by CPU cycle (Gariland, pcsx-redux port 8080, 2026-08-01) — matches the latch/paint mechanism documented in `SCENARIO_WAIT_SEMANTICS.md`
  - R: none — two-stage latch/paint anim commit not present in godot-learning (probed `godot-learning/src/`, `godot-learning/tests/`)
  - src: `research/working_documents/MARCH_OPCODE_80_SEMANTICS.md`

## Notes

(empty — user territory)

## Related

- [[March Opcode]]
- [[Event Unit Selector]]
- [[Unit Anim Opcode]]
- [[Walk To Opcode]]
- [[Unit Sprite Render Pipeline]]
