# Dialogue Box SFX

FFT plays no SFX on dialogue-box open or close/dismiss — confirmed by two static RE passes and a live PCSX capture (all 9 box open/close edges across the Orbonne opening were silent). Dialogue/UI sounds are requested, not directly played: code writes a sound id into the one-word request register `DAT_80165fb4` (idle −1, reset ~every 4 frames by PC `0x8014307c`), which the system SFX dispatcher `SUB_80043ff8` drains; the dialogue-lifecycle ids are 115/`0x73` Text Typing (per glyph, boxed only — the prayer overlay's typewriter is gated silent by `FUN_8013b590(0x53)`), 45/`0x2d` Flip Page (in-message page/line advance), 3 Move Cursor, 1/2/5 Confirm/Cancel/Invalid, plus any authored id via event opcode `{21}`. `godot-learning` wires typing, page flip, and cursor (`SfxRouter` cues `ui.text_typing` / `ui.dialogue_page_flip` / `ui.cursor_move`), but its `DialogueOverlay` types with sound, diverging from the ROM's silent overlay; Confirm/Cancel/Invalid remain unwired.

## Points

- **FFT dialogue-box open and close/dismiss play NO SFX — the box-open allocation and close/dismiss paths write nothing to the sound-request register; confirmed by two static RE passes and live capture (all 9 box opens + 9 box closes across the Orbonne opening were silent).** — `[S·D] 2/3`
  - S: box open `FUN_8012e348`; box close/dismiss `LAB_8013276c` → `FUN_8012e5a4`/`FUN_8012ec90`/`FUN_8012f65c` — no `DAT_80165fb4` write (`battle_disassembly.txt`, per doc)
  - D: probe `probe_dialogue_boundary_sfx.py` polling `0x80165fb4` every vblank across the Orbonne opening (sstate `orbonne_prayer_pre_scenario_load`, port 8080 full ISO) → `last_run/probe_dialogue_boundary_sfx.jsonl`: 9 typing runs (id 115) = 9 boxed dialogue lines, every open edge and close edge had NO sound request (2026-07-03)
  - R: none — no box open/close SFX cue in godot-learning (probed `godot-learning/src/audio/SfxRouter.gd` cue registry + `godot-learning/tests/`; registry holds only `ui.cursor_move`, `ui.text_typing`, `ui.dialogue_page_flip`, `combat.unit_died`) — absence matches the ROM
  - src: `research/working_documents/scenario_1_captures/FINDING_no_dialogue_box_close_sfx.md`
- **Dialogue/UI sounds are request-registered, not directly played: code writes a sound id into the one-word request register `DAT_80165fb4` (idle −1), which PC `0x8014307c` resets to −1 every ~4 frames once the scenario is active (idle heartbeat); the consumer is the system SFX dispatcher `SUB_80043ff8`, which plays whatever non-−1 id it finds, while the battle-action `play_sound` `FUN_800125a8` is a separate path (only 10 call sites, none in dialogue code).** — `[S·D] 2/3`
  - S: register `DAT_80165fb4`, idle reset `0x8014307c`, dispatcher `SUB_80043ff8`, separate `FUN_800125a8` (`battle_disassembly.txt`, per doc)
  - D: same per-vblank register poll (2026-07-03) observed exactly the 9 typing bursts (id 115) + 1 authored `0x48` across the Orbonne opening — nothing reached playback outside this register path
  - R: none — no `DAT_80165fb4` request-register equivalent in godot-learning (probed `godot-learning/src/`, `godot-learning/tests/`); the port fires cues directly via `SfxRouter.play_cue`
  - src: `research/working_documents/scenario_1_captures/FINDING_no_dialogue_box_close_sfx.md`
- **The dialogue-lifecycle sound ids on `DAT_80165fb4` (all 20 writers enumerated): `0x73`/115 Text Typing (per glyph, boxed dialogue only), `0x2d`/45 Flip Page (text-flow control point in the render loop — page/line advance *within* a message, not box open/close), `3` Move Cursor (selectable dialogues), `1`/`2`/`5` Confirm/Cancel/Invalid (selectable dialogues; helpers `FUN_8012dd1c`/`FUN_8012dd30`/`FUN_8012dd44`/`FUN_8012dd58`), and any authored id via event opcode `{21}` (interpreter `FUN_8013e904`).** — `[S·D·R] 3/3`
  - S: full enumeration of all 20 writers of `0x80165fb4` (`battle_disassembly.txt`, per doc)
  - D: Orbonne probe capture (2026-07-03): 9 typing (115) runs + one authored `0x48`/72 "Locked Door" at frame 2933 *between* typing runs (an authored `{21}` story sound, not at any box boundary); no Flip Page / menu sounds appeared (those dialogues were single-page, non-selectable)
  - R: `godot-learning/src/audio/SfxRouter.gd` `ui.text_typing` (system slot 115/0x73, retrigger via `EffectSfxEngine.play_click`), `ui.dialogue_page_flip` (system slot 45/0x2d, fired on a real `DialogueBox.advance_page()` page turn only — never on final advance/close), `ui.cursor_move` (system slot 3/0x03) + `godot-learning/tests/DialogueBoxTest.gd` `_test_page_flip_sfx_on_advance`; Confirm/Cancel/Invalid (1/2/5) unwired
  - src: `research/working_documents/scenario_1_captures/FINDING_no_dialogue_box_close_sfx.md`
- **The prayer overlay typewriter is ALSO silent — its `0x73` Text Typing write is gated off by `FUN_8013b590(0x53)`; only boxed dialogue types with sound.** — `[S] 1/3`
  - S: gate `FUN_8013b590` @ `0x8013B590` arg `0x53` (`battle_disassembly.txt`, per doc)
  - R: none — the overlay-silence gate is not present in godot-learning; `godot-learning/src/scenarios/DialogueOverlay.gd` fires `ui.text_typing` per overlay glyph (`glyph_typed` → `SfxRouter.play_cue`), so the port types the overlay WITH sound (divergence from the ROM; no named test)
  - src: `research/working_documents/scenario_1_captures/FINDING_no_dialogue_box_close_sfx.md`

## Notes

(empty — user territory)

## Related

- [[Event Sound OpCodes]]
- [[Typewriter Text Cadence]]
- [[Display Message Opcode]]
- [[SFX Index]]
