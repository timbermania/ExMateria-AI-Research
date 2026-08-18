# Start Action Menu

The START-key action menu on the formation screen: the 5-mode state machine around it, the generic vertical list menu, the glove cursor, container placement, and the window-backgrounding CLUT swap.

## Points

- **The formation menu state machine `formation_menu_state_machine` `FUN_801022BC` has 5 modes (idle / transition / detail / equip / ability); pressing START posts screen command id `0x1F` → `world_screen_dispatch` `FUN_800F4CDC` → the screen table @ `0x80156750`; each mode has its own entry/cancel handler.** — `[S·D] 2/3`
  - S: `research/working_documents/FORMATION_SCREEN.md` §15.20 (state machine + dispatch path)
  - D: live mode transitions observed across START presses in the oracle
  - R: none — not present in godot-learning (probed main@054e6849e, 2026-08-18: src/ + tests/)
  - src: `research/working_documents/FORMATION_SCREEN.md`
- **The START menu itself is the generic vertical list-menu coroutine `world_menu_list_coroutine` `FUN_800ECF20` (window slot 6): its 5 items come from message ID `0xD800` rendered as one block; the window struct is a 16-slot stack at `0x80195CD0` with stride `0x400`; its box-open goes through the sibling scaler `FUN_800EC7B4` using the same `0x801533B8` curve — the only two curve readers in the ROM are `0x800EC7D0` and `0x800EC9A0`.** — `[S·D] 2/3`
  - S: `research/working_documents/FORMATION_SCREEN.md` §15.20 (list-menu coroutine, message block, 16-slot/0x400 stack, slot-6 confirmation, two curve readers)
  - D: slot-6 confirmation from settled-state RAM (`*(base + DAT_801cd170·0x400)`) + draw-struct @ `0x8018C0D8`
  - R: none — not present in godot-learning (probed main@054e6849e, 2026-08-18: src/ + tests/)
  - src: `research/working_documents/FORMATION_SCREEN.md`
- **The list-menu cursor is a two-sprite glove: a lit glove at uv(168,0) CLUT `0x7D7C` (16×16, opaque) drawn AFTER a subtractive shadow glove at uv(184,0) CLUT `0x7DBC` (tpage `0x5F`, abr=2, gouraud `0x808080`) offset +2,+2 — an opaque lit sprite over a crescent shadow; placement `x = win_x + bob − 0xC`, `y = win_y + row·0x10 + 10`; the bob axis is X and the at-rest wobble is a 46-frame-period `[threshold, int8]` step table @ `0x80156352` — `(16,0)(18,1)(22,2)(28,3)(36,2)(46,1)` → 0..3px; the ADR-0046 render function is `FUN_800EC504`.** — `[S·D·R] 3/3`
  - S: `research/working_documents/FORMATION_SCREEN.md` §15.20 (glove lit/shadow uvs + CLUTs, placement formula, bob table, ADR-0046 function)
  - D: oracle cursor captures in two positions (crescent shadow visible; bob wobble 0..3px)
  - R: `godot-learning/tools/parse_cursor_bob.py` + `godot-learning/src/scenes/CursorBob.gd` + `godot-learning/tests/CursorBobTest.gd` (the glove's threshold-pair encoding parsed into the same JSON; "no consumer yet" as of main@054e6849e)
  - src: `research/working_documents/FORMATION_SCREEN.md`
- **The START menu container flips side by unit position: main-formation menu at top-left (10,32,84,96) or right-side (172,32); the rule is unit-sprite centre-x < 128 → menu on the right (172,32), otherwise left (10,32).** — `[S·D] 2/3`
  - S: `research/working_documents/FORMATION_SCREEN.md` §15.21 (container rects + side-flip rule)
  - D: oracle captures of left-side and right-side units with the menu open
  - R: none — not present in godot-learning (probed main@054e6849e, 2026-08-18: src/ + tests/)
  - src: `research/working_documents/FORMATION_SCREEN.md`
- **"Sending a window to the background" is a baked CLUT SWAP, not an ambient tint or alpha blend: foreground `0x7C3C` ⇄ background `0x7D3C` (plus `0x7D7C→0x7DFC`, `0x7CBC→0x7C7C`, glove shadow `0x7DBC→0x7E3C`); the blue is baked into the ROM palette; the trigger is the per-slot stack flag at `+0x48` (screen-slot struct base `DAT_80195D18`, stride `0x400`); the nameplate + vitals have no background variant and stay tan; this supersedes the earlier "semi-transparent tint" interpretation.** — `[S·D] 2/3`
  - S: `research/working_documents/FORMATION_SCREEN.md` §15.21 (baked-CLUT-swap backgrounding, per-slot `+0x48` flag, palette bytes)
  - D: oracle A/B (menu closed vs open): the "blue" windows are byte-identical geometry through a different CLUT; vitals/nameplate unchanged in both
  - R: none — not present in godot-learning (probed main@054e6849e, 2026-08-18: src/ + tests/)
  - src: `research/working_documents/FORMATION_SCREEN.md`

## Notes

(empty — user territory)

## Related

- [[Menu Window Box Open]]
- [[Equip Sub Screen]]
- [[Formation Vitals And Nameplate]]
