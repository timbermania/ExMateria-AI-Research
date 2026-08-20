# Event Unit Selector

The event unit-selector operand (`V = Units | Multi<<8`) shared by the 10 unit-target event handlers is resolved by `resolve_event_unit_handle @ 0x80147928` into a mode: `V < 0x100` → single unit id (`FUN_80133158`), `V == 0` → mode 1 (all existing units), `V ≥ 0x100` → mode `V − 0xFE` (mode = 2 + Units for `Multi=1`: 2 player, 3 player+alive, 4 enemy, 5 enemy+alive; any mode ≥ 6, incl. `Multi=2`, hits the iterator's default arm → all present/alive). The per-candidate test `FUN_801479ac` loops the 21-slot roster `DAT_80169720`; team is `structB[+0x5]&0x30` on the flat `0x801908cc`/`0x1C0` array (NOT the `0x800B7308`/`0x440` facing array), and modes 3/5 additionally drop units whose `structB[+0x58+i]` overlaps the RAM-only alliance mask `DAT_800662f8 = [0x74, 0xC1, 0x01, 0x00, 0x00]`. Presence = a live node in `unit_sprite_list` (head `0x80098a54`), independent of the `+0xa` render flag — proven at scn6 pc31, where hidden-but-allocated Ovelia is still rotated by the mode-3 broadcast. Godot's `ScenarioWorld.resolve_unit_set` mirrors this as a shared broadcast across all 5 unit-target applies (team from ENTD `team_color`, presence from `scenario_present`), with `PLAYER_ALIVE`/`ENEMY_ALIVE` as accepted team+alive supersets (no alliance source in the ENTD pipeline).

## Points

- **`resolve_event_unit_handle @ 0x80147928` maps selector `V` (`Units | Multi<<8`) to a mode: `0 < V < 0x100` → mode 0 (single unit id V, loop breaks after one), `V ≥ 0x100` → mode `V − 0xFE` (team-set broadcast), `V == 0` → mode 1 (broadcast — all units).** — `[S·D·R] 3/3`
  - S: `resolve_event_unit_handle @ 0x80147928` (`battle_decompilation.c`)
  - D: scenario-8 read-only RAM/VRAM capture (2026-07-10): 81 view-3 unit-palette entries consistent with an all-present-units broadcast
  - D: mode-3 Rotate exec-BP trace, scn6 idx 31 `2D 01 01 08`, BP1 @0x80147a08 / BP2 @0x801482f0 (2026-07-07) — selector→mode-3 path exercised end-to-end
  - D: live Gariland (scenario 10) March pc capture (2026-08-01): selector `0x0080` → SINGLE unit `0x80` (slot 0) — the only unit released by PC23 March; March handler `FUN_80149490` joins the shared-`resolve_event_unit_handle` unit-target family
  - R: `godot-learning/src/scenarios/ScenarioDecode.gd` `unit_set_mode`, validated by `godot-learning/tests/ScenarioDecodeTest.gd`
  - src: `research/working_documents/COLOR_TINT_LUMA_MODE_SEPIA.md`
- **`FUN_801479ac @ 0x801479AC` is the per-candidate membership test for the 21-slot selector loop: mode 1 = any existing unit (`unit_sprite_object_exists` only, no team/alive filter), 2 = team A/player (`(unit+5) & 0x30 == 0`), 3 = team A + alive, 4 = team B/enemy (`(unit+5) & 0x30 != 0`), 5 = team B + alive.** — `[S·D] 2/3`
  - S: `FUN_801479ac @ 0x801479AC` (`battle_decompilation.c`)
  - S: roster `DAT_80169720[0..20]` (u16 unit handles, loop bound `slti 0x15`), Rotate handler `0x80148284` (BATTLE.BIN disassembly)
  - D: exec-BP trace, scn6 idx 31 (2026-07-07): 21 slots exercised — 5 kept / 2 team-excluded (0x05/0x06) / 2 alliance-excluded (0x03/0x04) / 12 absent
  - src: `research/working_documents/COLOR_TINT_LUMA_MODE_SEPIA.md`
- **An event `{32}` reaches many units via the selector membership loop in `event_color_command_processor @ 0x801495E0` — read selector, resolve, loop up to 21 candidate slots, one `color_unit_fanout` call per member, break if mode==0 — which is distinct from `color_unit_fanout`'s own all-16 branch (param ≥ 0x10) that fires only for effect/charge callers; the event path always passes a single resolved handle per call.** — `[S·D] 2/3`
  - S: `event_color_command_processor @ 0x801495E0`, `color_unit_fanout @ 0x800933C4` (`battle_decompilation.c`)
  - D: live handler capture, Orbonne door-exit fade (2026-06-30, savestate `orbonne_three_actors_walk_in.sstate`): six `0x800933C4` fires, each carrying one resolved slot (`0x2/0x3/0x4` = event units `2/23/131`) — the event path passes a single resolved handle per call
  - src: `research/working_documents/COLOR_TINT_LUMA_MODE_SEPIA.md`
  - src: `research/working_documents/UNIT_FADE_COLOR_UNIT_OPCODE.md`
- **scn8 PC29's `{32}` has `Units=0, Multi=0` → `V=0` → mode 1, so it tints ALL present units (PSX cannot single-target unit 0 — id 0 always broadcasts); Godot's `ScenarioDecode.unit_set_mode(0, 0)` now returns `UnitSetMode.ALL` (was `SINGLE`), and because this is the shared `{11}`/`{2D}`/`{53}`/`{32}`/March selector, `(0,0)` now broadcasts for all of them with zero new failures in a full scenario regression sweep.** — `[S·D·R] 3/3`
  - S: `resolve_event_unit_handle @ 0x80147928` V==0→mode-1 fall-through + PC29 operands (`battle_decompilation.c`, `godot-learning/assets/scenarios/chunks/scenario_008_chunk.json`)
  - D: scenario-8 read-only RAM/VRAM capture (2026-07-10): 81 view-3 entries (many units) confirm the all-present-units broadcast
  - R: `godot-learning/src/scenarios/ScenarioDecode.gd` `unit_set_mode`, validated by `godot-learning/tests/ScenarioDecodeTest.gd` `_test_unit_set_mode` (`(0,0)→ALL`)
  - src: `research/working_documents/COLOR_TINT_LUMA_MODE_SEPIA.md`
- **`FUN_801479ac` membership adds two constraints not in the earlier decode: modes 3/5 also require no runtime ALLIANCE overlap (`structB[+0x58+i] & DAT_800662f8[i] ≠ 0` excludes the unit, i=0..4, mode-5 slot 0 additionally masked `&0xEF`), and any mode ≥ 6 falls through the iterator's default arm (`LAB_80147b20`) → all present/alive units.** — `[S·D·R] 3/3`
  - S: `FUN_801479ac @ 0x801479ac`, default arm `LAB_80147b20` (BATTLE.BIN disassembly)
  - D: mode-3 Rotate exec-BP trace, idx 31 `2D 01 01 08` (2026-07-07): team-0 units 0x03/0x04 excluded by alliance overlap (`0x20&0x74`, `0x01&0x01`), Ovelia 0x0C kept; pc40 empirical set (slot space): rotated {0,1,2,6,7,9,10,12}, skipped {3,4,5,8}
  - R: `godot-learning/src/scenarios/ScenarioDecode.gd` `unit_set_member` (PLAYER_ALIVE/ENEMY_ALIVE as team+alive supersets; alliance overlap not modeled — accepted gap) + `godot-learning/tests/ScenarioDecodeTest.gd` `_test_unit_set_member`
  - src: `research/working_documents/EVENT_UNIT_SET_RESOLUTION.md`
- **Alliance mask table `DAT_800662f8 = [0x74, 0xC1, 0x01, 0x00, 0x00]` (i=0..4); it lives in RAM only (absent from the code-only disasm) and is load-bearing — team-0 units are excluded from the `_ALIVE` sets whenever `structB[+0x58+i]` overlaps the mask (scn6 idx-31: units 0x03/0x04 dropped, Ovelia kept).** — `[S·D] 2/3`
  - S: `DAT_800662f8` (BATTLE.BIN disassembly symbol; values read from RAM)
  - D: exec-BP trace BP1 `@0x80147a08` / BP2 `@0x801482f0` + `/tmp/survey_a.lua` RAM survey (2026-07-07)
  - R: none — `0x800662f8`/alliance not present in godot-learning (superset gap documented at `ScenarioDecode.unit_set_member`)
  - src: `research/working_documents/EVENT_UNIT_SET_RESOLUTION.md`
- **Per-unit data splits across two distinct structs: presence uses struct A (linked-list node under `unit_sprite_list`, head `0x80098a54`; fields `+0x12`/`+0x80`), while team/alliance use struct B — a flat array at base `0x801908cc`, stride `0x1C0` (448), 21 entries, fetched by `FUN_80180afc(FUN_8008cdd0(h))` where `FUN_8008cdd0` bridges handle→B-index via struct A `+0x134 → +0x18a` and returns −1 when absent; struct B is NOT the `0x800B7308`/`0x440` facing/anim array (which reads team `+0x5`/`+0x58` all-zero in cutscenes).** — `[S·D] 2/3`
  - S: `0x801908cc`, `FUN_80180afc`, `FUN_8008cdd0`, `0x80098a54` (BATTLE.BIN disassembly)
  - D: `/tmp/survey_b.lua` struct-B RAM survey (2026-07-07)
  - R: none — `0x801908cc`/struct B not present in godot-learning (team color plumbed from ENTD `team_color` instead)
  - src: `research/working_documents/EVENT_UNIT_SET_RESOLUTION.md`
- **The whole unit-target opcode family shares the resolver: all 10 handlers — `0x801480c8`, `0x80148284` (Rotate Unit), `0x80148d14`, `0x8014920c`, `0x801493b8`, `0x801494ac`, `0x80149548`, `0x801495fc`, `0x801496a4`, `0x801497e8` — call `resolve_event_unit_handle` then loop `FUN_801479ac` over the 21-slot roster `DAT_80169720[0..20]`.** — `[S·D·R] 3/3`
  - S: the 10 handler addresses + `DAT_80169720` (BATTLE.BIN disassembly)
  - D: Rotate handler (`0x80148284`) traced end-to-end at scn6 idx 31 (2026-07-07)
  - R: shared `godot-learning/src/scenarios/ScenarioWorld.gd` `resolve_unit_set` broadcast across `ScenarioApply.gd`'s 5 unit-target applies (rotate_unit/unit_anim/color_unit/face_unit/face_tile) + `godot-learning/tests/ScenarioApplyTest.gd` per-opcode broadcast tests
  - src: `research/working_documents/EVENT_UNIT_SET_RESOLUTION.md`
- **`Multi=2` has no separate branch in `resolve_event_unit_handle`: `V = Units|0x200` gives mode `Units + 0x102` ≥ 6, hitting the iterator's default arm → all present/alive units (aliases mode 6 = `Multi=1, Units=4`); no dead-inclusive variant is reachable. scn1's five `Multi!=0` rows (0x012E/0x013A/0x0125/0x0136/0x012C → modes 0x30/0x3C/0x27/0x38/0x2E) all degenerate to the same default arm (PSX-park confirmation still open; Godot boot-verified error-free).** — `[S·R] 2/3`
  - S: `resolve_event_unit_handle @ 0x80147928`, default arm `LAB_80147b20` (BATTLE.BIN disassembly)
  - R: `godot-learning/src/scenarios/ScenarioDecode.gd` `unit_set_mode` (Multi≥2 → `ALL`) + `ScenarioDecodeTest` `_test_unit_set_mode`, `ScenarioApplyTest` `_test_unit_anim_broadcasts_all`
  - src: `research/working_documents/EVENT_UNIT_SET_RESOLUTION.md`
- **scn6 pc31's mode-3 Rotate `[Units=1,Multi=1,Facing=8]` is Ovelia's (id 0x0C) only rotate source: she parks at facing `+0x70` = 0x0C00 and settles to 0x0800 (WEST) after pc31, with no individual rotate before it (pc25/27/29 are `Multi=0` rotates of units 2/52/23; pc32–37 is camera/focus/wait), so `0xC00→0x800` is attributable to pc31 alone; Delita (enemy team) stays at 0x0400.** — `[S·D·R] 3/3`
  - S: facing field `+0x70` (struct A; BATTLE.BIN disassembly), Rotate handler `0x80148284`
  - D: facing-delta capture (2026-07-09) — scenario-event-debugger, base `scenario6_letgo_full_base`, `scn_jump(31)` + `scn_step`: Ovelia node `0x800BA608` 0x0C00 → 0x0800; Delita `0x800B8848` holds 0x0400; Agrias `0x800B7748` 0x0000 → 0x0800
  - R: live scn6 pc31 in godot-learning: Ovelia 0x0C rotates to 0x800, Delita/enemies excluded (2026-07-07) + `godot-learning/tests/ScenarioApplyTest.gd` `_test_rotate_unit_broadcasts_player_team`, `godot-learning/tests/ScenarioBroadcastRotateHeldTest.gd`
  - src: `research/working_documents/EVENT_UNIT_SET_RESOLUTION.md`
- **Presence ≠ render visibility: at scn6 pc31 Ovelia reads show flag `+0xa` = 0 (her `{44} Draw Unit [Unit=12]` is at pc104) yet built-primitive ptr `+0x204` = 0x800BA924 (non-null → allocated) with live anim (`+0x1DC` = 6) — held/hidden but allocated — and the mode-3 broadcast rotates her regardless of `+0xa`; the broadcast roster must key on presence (allocated in `unit_sprite_list`), not the render flag.** — `[S·D·R] 3/3`
  - S: struct-A offsets `+0xa` (show), `+0x204` (built-primitive ptr), `+0x1DC` (anim) (BATTLE.BIN disassembly)
  - D: pc31-parked capture, `/tmp/scn6_pc31_capture.py` (2026-07-09)
  - R: `godot-learning/src/scenarios/ScenarioWorld.gd` `_unit_roster` keys on `scenario_present` (spawn for always-present units + {44}/{45}/{47}; independent of `visible`) + `godot-learning/tests/ScenarioBroadcastRotateHeldTest.gd`
  - src: `research/working_documents/EVENT_UNIT_SET_RESOLUTION.md`
- **Godot replaced the wrong `PsxNum.u16_lohi(Units, Multi)` single-id model with a shared multi-unit broadcast: `ScenarioWorld.resolve_unit_set(Units, Multi)` feeds all 5 unit-target applies (`rotate_unit`/`unit_anim`/`color_unit`/`face_unit`/`face_tile`); team is plumbed from ENTD `team_color` at spawn (`Unit.scenario_team_color` == ROM `structB[+0x5]&0x30 >> 4`, 0 = blue/player); live scn6 pc31 rotates {0x02,0x34,0x17,0x84,0x85,0x0C} (team-0 present, incl. Ovelia → 0x800) and excludes Delita/enemies/hidden — the ROM set ∪ the alliance-superset {0x84,0x85}.** — `[D·R] 2/3`
  - D: live scn6 pc31 verification (2026-07-07; re-confirmed 2026-07-09 after the chunk-derived visibility fix made Ovelia spawn hidden)
  - R: `godot-learning/src/scenarios/ScenarioWorld.gd` `resolve_unit_set` + `ScenarioApply.gd` applies — validated by `godot-learning/tests/ScenarioApplyTest.gd` (per-opcode broadcast) and `ScenarioDecodeTest.gd` (mode table + §5.1 truth table)
  - src: `research/working_documents/EVENT_UNIT_SET_RESOLUTION.md`
- **Rotate Unit's target-angle byte (the doc/wiki calls it `Direction`): x00–0x0F = absolute wheel step (byte·0x100, e.g. 8 → 0x800 WEST), x10 = camera-relative, x11/x12/x13 = relative +90/180/270, x14 = reset to pre-event facing.** — `[S·D·R] 3/3`
  - S: Rotate handler `0x80148284` + facing-mode dispatch `FUN_8008c0ac`/`FUN_8008c114` (BATTLE.BIN disassembly); wiki `{2D}` RotateUnit table (user-provided 2026-07-07)
  - D: scn6 pc31 Facing=8 → 0x800 WEST, exact seed formula (2026-07-09)
  - R: `godot-learning/src/scenarios/ScenarioDecode.gd` `resolve_rotate_target_12bit` (0x10 camera-relative; 0x11–0x13 `current + N·0x100`; 0x14 hold; else `PsxNum.facing_byte_to_12bit`) + `ScenarioApplyTest` `_test_rotate_unit_relative_target` — note: Godot's relative step is N·0x100 (N = low nibble), the wiki lists +90/180/270
  - src: `research/working_documents/EVENT_UNIT_SET_RESOLUTION.md`
- **The chunk `Unit` operand resolves with no remap: `FUN_80133158` passes the operand through unchanged to the exact-match resolver `FUN_80180c90`, which scans the roster for `+0x161 == id` (roster table `0x801908cc + (slot+0x4)·0x1C0 + 0x161`) — breakpoint-verified identity 2→2, 23→23, 131→131 on the chapel walk-in, so the chunk Unit space is the event unit-id space, not a re-mapped one.** — `[S·D·R] 3/3`
  - S: `FUN_80133158`, `FUN_80180c90`, roster base `0x801908cc` stride `0x1C0` (BATTLE.BIN symbols per the doc)
  - D: breakpoint probe of `FUN_80133158` via pcsx-agent, chapel scenario 1 (2026-06-28)
  - R: `godot-learning/src/scenarios/ScenarioVM.gd` `_resolve_unit_key` (direct identity + low-byte fallback, no remap) — chapel dual-trace confirms identical walk-in seats
  - src: `research/working_documents/scenario_1_captures/HANDOFF_unit_tile_alignment.md`

- **`V == 0` broadcasting to every existing unit is confirmed exactly, and it is the one claim in this vault that needed no correction at all.** — `[S] 1/3`
  - S: the routine is `ClassifyUnitSpec` at `BATTLE.BIN+e0928`, and **11 rungs share it** — `0x11`, `0x2C`, `0x2D`, `0x32`, `0x53`, `0x69`, `0x6C`, `0x6D`, `0x80`, `0x81`, `0x83` — while `{47}` is *not* one of them. The `V == 0` arm is 2 instructions: `0x80147944` branches to `0x80147988`, which sets membership mode 1 and returns, and mode 1's per-slot predicate is `FindActor(i) != 0` over `i = 0..0x14`, i.e. *exists*. For `0 < V < 0x100` the resolved id is written back into the spec word **in place**, and `0x7D0` coming back is the routine's only false. A model reading `Units=0, Multi=0` as "unit 0" would aim 96 of the disc's 6,505 `0x11`s, 89 of its 620 `0x32`s and 27 of its 808 `0x2D`s at one unit instead of at all of them (web-psx `docs/event-seam.md` [event.hle.units]) (2026-08-19)
  - src: external contribution — web-psx `docs/event-seam.md` [event.hle.units] (see [[Web-psx Cross-Validation]])

## Notes

(empty — user territory)

## Related

- [[Color Tint Luma Modes]]
- [[Color Unit Opcode]]
- [[Event Opcode Catalog]]
- [[Unit Anim Opcode]]
- [[Rotate Unit Interpolation]]
- [[ENTD Unit Deployment Table]]
