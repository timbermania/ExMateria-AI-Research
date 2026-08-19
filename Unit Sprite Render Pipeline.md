# Unit Sprite Render Pipeline

The battle/scenario unit-sprite path from event opcode to on-screen pixels, verified stage-by-stage against the scn6 abduction carry (2026-07-08): `{3B}` Sprite Move arms a target offset that `FUN_8006af7c` integrates per frame through fixed-point accumulators (tile = accum ÷ 0x1C000, the same cadence constant as Walk To) driving render offsets `+0x60/+0x62/+0x64`; `FUN_80086b44` projects the unit world→screen with a GTE **orthographic** MVMVA (screen pos at `unit+0x120/+0x122`); `sprite_subframe_assemble` writes the sprite descriptor at `*(unit+0x204)` (piece count `[+3]`, CLUT `[+6..7]`, per-piece x/y/U/V at `14+idx*7`, piece-0 U/V at `[+18]/[+19]`); a mode-0 consumer at `0x80086a68` places the primitive in the ordering table; the GPU draws the OT into the hidden DRAWENV buffer and pacer `FUN_80093a98` swaps the display via PutDispEnv — DRAWENV/DISPENV share the toggling index `_DAT_8004597C` but point at OPPOSITE VRAM blocks, with two 256×240 display blocks stacked at TOP y[0,240] / BOTTOM y[240,480]. The carry's ~2px "203→204" sprite shift resolved as double-buffer pipeline lag, not a state change, so the settled-state Godot render is faithful.

## Points

- **Sprite Move motion integrates per unit per frame in `FUN_8006af7c` (@0x8006AF7C): it advances the fixed-point position accumulators `+0x18/+0x28`, `+0x20/+0x30` (tile = accum ÷ 0x1C000, the same cadence constant as Walk To) by the per-frame velocity, driving the render offsets `+0x60` (+X) / `+0x62` (+Z, height) / `+0x64` (+Y, depth) toward the target until settled — the scn6 pc200 carry settles exactly at its operand target `(-4,-3,8)`.** — `[S·D·R] 3/3`
  - S: `FUN_8006af7c` @0x8006AF7C, unit offsets +0x18/+0x28/+0x20/+0x30/+0x60/+0x62/+0x64 (BATTLE.BIN, per INSTRUCTION_TO_RENDER.md)
  - D: scn6 abduction-carry pc200/pc203 per-vblank captures — offset frozen at (-4,-3,8), the exact pc200 target, by park 203 (2026-07-08)
  - R: `godot-learning/src/scenarios/ScenarioApply.gd` (`sprite_move` arms the per-unit straight-line motion; {6F} barrier stays on the VM) + `src/scenarios/ScenarioMotion.gd` (bit-exact easing curve, ROM-verified) + `godot-learning/tests/ScenarioSpriteMoveTest.gd`
  - src: `research/working_documents/INSTRUCTION_TO_RENDER.md`
- **Unit world→screen projection is `FUN_80086b44` (@0x80086B44): per frame it walks the unit list `DAT_80098A54` and applies a GTE **orthographic** MVMVA (mx=0, sf=1 — NOT perspective) on world-vec + offset, writing the screen position to `unit+0x120` (X) / `unit+0x122` (Y), with screen offsets `+0x58/+0x5a` and H/V-flip flags `+0x12`.** — `[S·D] 2/3`
  - S: `FUN_80086b44` @0x80086B44, `DAT_80098A54` (BATTLE.BIN, per INSTRUCTION_TO_RENDER.md)
  - D: worked against the scn6 abduction carry (Sprite Move@pc200), per-vblank captures (2026-07-08)
  - R: none — `0x80086b44` MVMVA unit projection not present in godot-learning (probed `godot-learning/src/`, `godot-learning/tests/`)
  - src: `research/working_documents/INSTRUCTION_TO_RENDER.md`
- **`sprite_subframe_assemble` (`FUN_80084214` @0x80084214 → `FUN_8007b4ec` @0x8007B4EC) resolves the current anim/frame (`unit+0x1DC` anim id, `unit+0x1E0` SEQ frame) into the sprite descriptor at `*(unit+0x204)`: header `[+3]` = piece count (`(frame_byte0&7)+1`, written `sb v1,0x3(s5)` @0x800842a0), `[+6..7]` = CLUT; then per piece at offset `14 + idx*7`: `[0]=x, [1]=y, [4]=U, [5]=V` (piece 0's U/V = descriptor bytes `[+18]`/`[+19]`), and the UV picks the frame COLUMN in the EVTCHR/SEQ sheet (±32px/frame); cinematic anims (≥0x1F4) select the EVTCHR sheet/table — same engine.** — `[S·D] 2/3`
  - S: `FUN_80084214`/`FUN_8007b4ec`, `sb v1,0x3(s5)` @0x800842a0 (BATTLE.BIN, per INSTRUCTION_TO_RENDER.md)
  - D: scn6 abduction carry — live descriptor reads + hide-unit de-confounding probe (zeroing `*(*(unit+0x204)+3)` persists only within a held pose; the descriptor is rebuilt) (2026-07-08)
  - R: none — runtime sprite descriptor `*(unit+0x204)` not present in godot-learning (probed `godot-learning/src/`, `godot-learning/tests/` for `0x204`/piece count; the disc-side SHP tile-UV formula is separately R-verified in [[EVTCHR Frame Resolution]])
  - src: `research/working_documents/INSTRUCTION_TO_RENDER.md`
- **The mode-0 consumer loop at `0x80086a68` (runs when `unit+0x130==0`) combines the descriptor's local quad + UV with the projected screen position (`unit+0x120/+0x122`) into a GPU primitive placed in the ordering table; OT/primitive workspaces observed at `0x800FC000` / `0x8010A000` / `0x80119000`.** — `[S·D] 2/3`
  - S: consumer @0x80086a68, workspaces 0x800FC000/0x8010A000/0x80119000 (BATTLE.BIN, per INSTRUCTION_TO_RENDER.md)
  - D: worked against the scn6 abduction carry (Sprite Move@pc200), per-vblank captures (2026-07-08)
  - R: none — unit-sprite OT insertion @0x80086a68 not present in godot-learning (probed `godot-learning/src/`, `godot-learning/tests/`)
  - src: `research/working_documents/INSTRUCTION_TO_RENDER.md`
- **The GPU draws the OT into the hidden buffer `DRAWENV[_DAT_8004597c]` (DRAWENV table base `0x8004EA14`, stride `0x5c`); pacer `FUN_80093a98` (@0x80093A98; normal wait count `g_gameloop_vblank_wait_count` @0x80045980 = 1, selector `g_gameloop_pacing_state` @0x800960E4) then `PutDispEnv(DISPENV[_DAT_8004597c])` (DISPENV table base `0x8004EACC`, stride `0x14`) — buffer index @0x8004597C toggles 0/1, and DRAWENV and DISPENV use the SAME index but point at OPPOSITE VRAM blocks = proper double-buffer.** — `[S·D] 2/3`
  - S: DRAWENV 0x8004EA14/stride 0x5c, DISPENV 0x8004EACC/stride 0x14, `_DAT_8004597C` @0x8004597C, 0x80045980, 0x800960E4 (BATTLE.BIN, per INSTRUCTION_TO_RENDER.md)
  - D: per-vblank capture from park 203 — "post" is drawn into TOP@f11, into BOTTOM@f20; swap shows TOP on-screen at f15 (2026-07-08)
  - R: none — DRAWENV/DISPENV double-buffer swap not present in godot-learning (probed `godot-learning/src/`, `godot-learning/tests/`)
  - src: `research/working_documents/INSTRUCTION_TO_RENDER.md`
- **`DISPENV[idx].disp.y` selects which 256×240 block is scanned out: the two blocks are stacked at TOP y[0,240] and BOTTOM y[240,480] (y240, NOT y256); idx0 → disp.y=240 = BOTTOM visible, idx1 → disp.y=0 = TOP visible.** — `[S·D] 2/3`
  - S: DISPENV table @0x8004EACC (BATTLE.BIN, per INSTRUCTION_TO_RENDER.md)
  - D: per-vblank capture from park 203 — TOP@f11 / BOTTOM@f20 / f15 swap pin the block mapping (2026-07-08)
  - R: none — stacked 256×240 display-block geometry not present in godot-learning (probed `godot-learning/src/`, `godot-learning/tests/`)
  - src: `research/working_documents/INSTRUCTION_TO_RENDER.md`
- **In the scn6 carry the pc200 motion STATE is settled by park pc203 (offset frozen at (-4,-3,8), one position from park 203 on), so the ~2px "203→204 change" is the tail of the pc200 slide surfacing through the double buffer — pc203 park = vblank f0 (screen PRE), pc204 park = vblank f15 (latch consumes 511→0 AND swap, screen POST) — a rendering-pipeline lag, not a state change; the game state (offset/anim/OT) is settled and correct, so a state-driven engine that shows the settled position (Godot) is faithful, and the staggered per-buffer catch-up + swap lag is a PSX rendering artifact, not authored motion.** — `[D·R] 2/3`
  - D: per-vblank frame-step capture from park 203: "post" drawn into TOP@f11, into BOTTOM@f20, swapped on-screen at f15 (2026-07-08)
  - R: `godot-learning/src/scenarios/ScenarioApply.gd` (`sprite_move` — endpoint absolute-from-home, repeated moves don't accumulate; shows the settled position) + `godot-learning/tests/ScenarioSpriteMoveTest.gd` (pins eased settle to target, {6F} blocks until done)
  - src: `research/working_documents/INSTRUCTION_TO_RENDER.md`

## Notes

(empty — user territory)

## Related

- [[Event Opcode Catalog]]
- [[Walk To Opcode]]
- [[GTE World-to-Screen Transform]]
- [[Ordering Table & AddPrim]]
- [[EVTCHR Frame Resolution]]
- [[Effect Frame Pacing]]
- [[Cinematic Sprite Renderer]]
- [[Scenario Beat Capture]]
