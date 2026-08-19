# Unit Sprite SEQ Opcodes

The SEQ sprite-animation opcode space (0xBE–0xFF, reached after a 0xFF marker byte) shared by the TYPE1 body interpreter (`seq_interp_type1` @0x80084818) and the WEP/EFF sub-anim interpreter (`seq_interp_wep_eff` @0x8008526C): which opcodes each layer executes, how 0xF2 addresses the sub-anim slots, and how 0xFE/0xFF restore the idle pose.

## Points

- **The TYPE1 SEQ opcode space (0xBE–0xFF) decodes as: 0xBE ClearReaction (unit+0x2E8=0), 0xBF TriggerReaction (0x8007F240), 0xC0 WaitForDistort (stalls unless +0x87==0), 0xC1 QueueDistortAnim (+0x87=arg0+2, +0x88=0, +0x8C=arg1), 0xC2 NOP, 0xC3 ClearDistortFlag, 0xC4 SetDistortParams, 0xC5 SetDistortFlag, 0xC6 SkipIfBattle, 0xCB–0xD2 fixed ±1/±2 movement via the 0x800846B0/0x80084758/0x80084770 helpers, 0xD3 SkipIfNotActive, 0xD4 SetPalette (0x80082620), 0xD5 IncrementLoop (loop counter +1, SEQ offset → 0), 0xD6 SkipIfActive, 0xD8 SetFrameOffset, 0xD9 PlaySound (0x800689A4), 0xDB SetVOffset, 0xDC RestoreSavedAnim / 0xDD SaveAndJumpAnim (save/restore at anim_state+0x0E/+0x10), 0xDE TriggerHitReactions, 0xDF SetYRotation0 (sprite_buffer+0x0C=0), 0xE0/0xE1 Hide/ShowShadow (+0x298), 0xE2 SetLayerPriority (+0x14 = arg), 0xE5 SaveYpin (16-bit rotation override), 0xEB/0xEC flip toggle (XOR 4/2 into slot +0x18), 0xEE/0xEF/0xF0 parameterized move, 0xF2 QueueSpriteAnim, 0xF6 SetAbilityAnim (0x8006B960), 0xF7 3-arg skip, 0xF9 MoveScreenXY (+0x58/+0x5A), 0xFA MoveAll3 (forward + up + lateral), 0xFC WaitLoop (jump offset + repeat count), 0xFD Jump, 0xFE EndAnimation (restore idle pose, zero movement offsets), 0xFF PauseAnimation; plus 1-arg NOPs 0xCA/0xD7/0xE3/0xE4/0xE9/0xED/0xF3/0xF4/0xF5 and 2-arg NOPs 0xC7–0xC9/0xE6–0xE8/0xEA/0xF1/0xF8/0xFB.** — `[S·R] 2/3`
  - S: `0x80084818` opcode dispatch (battle_disassembly.txt; per `research/working_documents/PSX_UNIT_SPRITE_RENDERING.md` §4)
  - R: `godot-learning/src/animation/AnimationOpcodes.gd` (opcode names) + `AnimationPlayback.gd` (side effects: movement, flip, layer priority, distort) — verified by `tests/GPUSEQMovementTest.gd` (movement) + `tests/GPUDashMovementTest.gd` (distort)
  - src: `research/working_documents/PSX_UNIT_SPRITE_RENDERING.md`
- **The WEP/EFF interpreter (0x8008526C) supports a reduced subset of the same opcode space: functional — 0xD5, 0xD8, 0xDB, 0xDC, 0xDD, 0xDF, 0xE2, 0xE5, 0xEB, 0xEC, 0xF2, 0xF6, 0xF9, 0xFC, 0xFD; 1-arg skip — 0xD3, 0xD6, 0xD7, 0xE3, 0xE4, 0xE9, 0xED, 0xEE, 0xEF, 0xF0, 0xF3, 0xF4, 0xF5; 2-arg skip — 0xD9, 0xE6, 0xE7, 0xE8, 0xEA, 0xF1, 0xF8, 0xFB; 3-arg skip — 0xF7, 0xFA; no-arg pass-through — 0xDE, 0xE0, 0xE1; 0xFE deactivates the slot, 0xFF zeroes its wait — so weapon/effect overlays never move the unit (all movement opcodes 0xCB–0xD2/0xEE–0xF0 are skipped).** — `[S] 1/3`
  - S: `0x8008526C` dispatch table (BATTLE.BIN, per `research/working_documents/PSX_UNIT_SPRITE_RENDERING.md` §5)
  - R: none — WEP/EFF reduced opcode set not present in godot-learning (`AnimationPlayback.gd` processes opcodes uniformly across layers; probed `godot-learning/src/animation/`, `godot-learning/tests/`)
  - src: `research/working_documents/PSX_UNIT_SPRITE_RENDERING.md`
- **Opcode 0xF2 QueueSpriteAnim arms a sub-anim slot by target (1, 2, 3 — target 0 is reserved for TYPE1 and errors): slot = `unit + target*0x30 + 0x1D8` (= +0x208/+0x238/+0x268), setting wait_timer=1 (immediate tick), seq_anim_id=arg1, seq_offset=0, frame_offset=0, v_offset=0, loop_count=0, active=1.** — `[S·R] 2/3`
  - S: `0x80084818` 0xF2 case (BATTLE.BIN, per `research/working_documents/PSX_UNIT_SPRITE_RENDERING.md` §4)
  - R: `godot-learning/src/animation/AnimationPlayback.gd` (QUEUE_SPRITE_ANIM side effect, target 1 → WEP1 / 2 → EFF1 via SpriteLayerManager.Layer) — verified by `tests/UnitWeaponBindTest.gd`
  - src: `research/working_documents/PSX_UNIT_SPRITE_RENDERING.md`
- **0xFE EndAnimation / 0xFF PauseAnimation restore the idle pose for TYPE1 slots: the idle frame is looked up from `unit_idle_frame_front_table` @0x800949C4 (front-facing, even slot) or `unit_idle_frame_back_table` @0x800949D0 (back-facing, odd slot), indexed via `sprite_type_idle_index_table[sprite_type*4]` @0x80094748 (first byte of the 4-byte-per-sprite-type entries that [[Unit Sprite Height Table]] uses for heights), then the idle frame is re-assembled and the movement offsets +0x50/+0x52/+0x54 are zeroed.** — `[S] 1/3`
  - S: `0x80084818` 0xFE/0xFF cases, `0x80094748`/`0x800949C4`/`0x800949D0` (BATTLE.BIN, per `research/working_documents/PSX_UNIT_SPRITE_RENDERING.md` §4)
  - R: none — ROM idle-frame-table restore not present in godot-learning (the port ends/pauses playback without a ROM table lookup; probed `godot-learning/src/animation/`)
  - src: `research/working_documents/PSX_UNIT_SPRITE_RENDERING.md`

## Notes

(empty — user territory)

## Related

- [[Unit Sprite Render Pipeline]]
- [[EVTCHR Script VM]]
- [[Unit Anim Opcode]]
- [[Unit Sprite Height Table]]
- [[PSX Rendering Index]]
