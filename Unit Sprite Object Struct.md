# Unit Sprite Object Struct

The battle unit-sprite object (struct A) lives in an array at base `0x800B7308`, stride `0x440` (~16 slots), singly linked from `unit_sprite_list_head @ 0x80098A54`. The fields that read a unit's pose live: `+0x04` slot index, `+0x06` unit id (the sprite-type index `FUN_80082eec` branches on), `+0x0C` pending-pose latch (u16 = anim_id+1, normally 0), `+0x70` 12-bit facing, `+0x140/+0x144` status flag words, `+0x1DC` active anim id, `+0x1E0` SEQ frame index, `+0x1E2` frame timer (nonzero = cycling, 0 = static hold) — so "is this unit marching in place?" = `+0x1DC` is an idle anim (6/7 in Gariland) AND `+0x1E2 ≠ 0`. Animation changes commit in two stages: `set_unit_animation_with_flags @ 0x80081978` latches `+0x0C` now, and the next-frame consumer `FUN_80085C0C` paints `+0x1DC` and clears the latch — verified on the single PC23→PC24 March release in Gariland (2026-08-01). The movement half of the same struct is shared with battle: `+0x38` is the movement-speed magnitude that event `{28} Walk To` writes as `Speed<<9` (landing on battle velocity constants), and `+0x40` is the `tile×28 + 14` screen projection, not a tile index.

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
- **The 2026-05-10 pipeline decode adds the sprite/animation-field span of the battle unit struct consumed by the renderers (same object as the +0x70/+0x1DC/+0x1E0/+0x1E2 fields above; ~0x300 in use): +0x0E tpage_base (TYPE1 TPAGE), +0x10 clut_base, +0x12 render_flags (bit 1 flip_H, bit 2 flip_V), +0x14 layer_priority_index (indexes the 4-entry layer config; set by SEQ 0xE2), +0x50/+0x52/+0x54 world-space move offsets, +0x58/+0x5A screen-space offsets, +0x6C/+0x6E facing quadrant / pose octant, +0x80 status_flags (0x1000000 frozen gate, 0x20000000 forced-anim gate), +0x87 motion_type / +0x88 motion_counter / +0x8C motion_param (distortion handler, armed by 0xC1), +0x120/+0x122 final screen X/Y, +0x128 screen depth, +0x130 mount_mode (0 normal / 1 hidden / 2 mounted) with +0x131 mount id, +0x13A wep_v_offset_idx / +0x13B wep_frame_base, +0x13F flip_xor_mask, +0x1D8 main anim state block (0x30 bytes), +0x1DE seq_offset, +0x1F4 shp_ptr, +0x1F8 per-unit SEQ pointer table, +0x204 sprite buffer pointer, +0x2D0 distort_flag, +0x2E8 reaction pointer.** — `[S] 1/3`
  - S: offsets decoded from `0x80085C0C` / `0x80084818` / `0x80086640` decompilation (BATTLE.BIN, per `research/working_documents/PSX_UNIT_SPRITE_RENDERING.md` §2/§9/§10)
  - R: none — ROM unit-sprite struct offsets not present in godot-learning (port keeps its own `Unit`/`ScenarioWorld` state; probed `godot-learning/src/`, `godot-learning/tests/`)
  - src: `research/working_documents/PSX_UNIT_SPRITE_RENDERING.md`
- **The three WEP/EFF sub-anim slots (+0x208/+0x238/+0x268) are 0x30 bytes each: +0x00 active, +0x02 sprite_type (1=WEP, 2=EFF), +0x04 seq_anim_id, +0x06 seq_offset, +0x08 prev_frame, +0x0A wait_timer, +0x0C loop_counter (0xD5), +0x0E SEQ pointer table, +0x10 SHP frame pointer table, +0x16 loop_count (0xFC), +0x18 flip_flags (XORed by 0xEB/0xEC), +0x1C shp_ptr, +0x24 sprite buffer pointer, +0x28 frame_offset, +0x2A v_offset_base; the render dispatch reaches slot buffers at `((slot−1)*0x30 + unit+0x22C)`.** — `[S] 1/3`
  - S: 0x30-slot layout consumed by `0x8008526C` / `0x80085C0C` / `0x80086640` (BATTLE.BIN, per `research/working_documents/PSX_UNIT_SPRITE_RENDERING.md` §2/§5/§9)
  - R: none — 0x30-byte WEP/EFF slot struct not present in godot-learning (port keeps WEP1/EFF1 as animation layers in `SpriteLayerManager`; probed `godot-learning/src/animation/`)
  - src: `research/working_documents/PSX_UNIT_SPRITE_RENDERING.md`
- **`unit+0x38` is the movement-speed magnitude field shared game-wide: event `{28} Walk To` writes `Speed<<9` into it, and those event values land exactly on the battle walk-speed constants (init/floor `0x1000`, normal walk `0x2000`, Haste band `> 0x3000`) — Speed 16 writes the same `0x2000` battle uses for a normal walk, so walk-vs-run is a game-wide concept, not a walk-only scalar.** — `[S·D] 2/3`
  - S: arm write `0x8008C77C` (`unit+0x38 = Speed<<9`) in `FUN_8008c664` (`battle_disassembly.txt`); battle constants per ffhacktics `BATTLE.BIN_Data_Tables` RAM map
  - D: scenario 6 chase live polling, Agrias = slot 0 @ `0x800B7308`, pcsx-redux port 8080 (2026-07-06) — `unit+0x38 = 0x2000` @ Speed 16, flat across the whole walk (no ease-in)
  - R: none — ROM unit-struct `+0x38` velocity field not present in godot-learning (probed `godot-learning/src/`, `godot-learning/tests/`)
  - src: `research/working_documents/SCENARIO6_CHASE_WALK_TIMING.md`
- **`unit+0x40` is the `tile×28 + 14` screen projection, not a "tile index": with the tile pitch `0x1C000 = 28 × 0x1000` folded into `+0x18`, the stepper's `unit[0x40] = unit[0x18] >> 0xC` yields tile×28, dropping the `+14` half-tile centering term — so the public RAM map's "`+0x18 = tile×0x1000`" is an oversimplification (the ×28 is folded into `+0x18`).** — `[S·D] 2/3`
  - S: stepper `FUN_8006AF7C` tile-boundary literal `0x1C000` + `+0x40` write (`battle_disassembly.txt`); `+0x40 = tile*28 + 14` per ffhacktics RAM map
  - D: scenario 6 chase live polling (2026-07-06) — measured 14 f/tile × `0x2000` velocity = `0x1C000`/tile, proving the ×28 folded into `+0x18`
  - R: none — no `0x1C000` sub-tile / `+0x40` screen-projection field in godot-learning (probed `godot-learning/src/`)
  - src: `research/working_documents/SCENARIO6_CHASE_WALK_TIMING.md`

## Notes

(empty — user territory)

## Related

- [[March Opcode]]
- [[Event Unit Selector]]
- [[Unit Anim Opcode]]
- [[Walk To Opcode]]
- [[Unit Sprite Render Pipeline]]
- [[Unit Sprite SEQ Opcodes]]
