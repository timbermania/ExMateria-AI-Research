# Cinematic Palette Pipeline

The per-unit palette axis of FFT's cinematic (EVTCHR) sprite rendering: what `unit[+0x0E]` and `unit[+0x10]` actually hold at frame commit, why the PSX per-unit CLUT is correct for everyone (slot-indexed VRAM address plus per-unit uploaded palette content), the decisive bit-match tying each unit's live CLUT to its own SPR palette at row = the ENTD palette byte, and the matching Godot contract (row set from the ENTD palette byte at spawn; the cinematic path swaps only the pixel atlas). Closed and live-verified for the chapel scenario (2026-06-28).

## Points

- **`unit[+0x0E]` is a staging TPAGE byte, not a palette: the frame commit `FUN_80084214` computes `render_state[+0x4] = unit[+0x0E] | (attr & 0x60)` (site `0x800843ec`; the sprite_set < 0x9B path is `0x800843F8`, the >= 0x9B path `0x80084304` is dead in chapel), and the `0x60` mask is the semi-transparency field of a PSX tpage word, so the live values 0x14..0x1b decode as marching texture-page-X.** — `[S·D] 2/3`
  - S: `FUN_80084214`, commit sites `0x800843ec` / `0x800843F8` / `0x80084304` (`battle_disassembly.txt`)
  - D: chapel palette-commit BP `0x800843F8` echoing `v0 = +0x0e` per unit + live VRAM CLUT dump (2026-06-28)
  - src: `research/working_documents/chapel_opcode_trace/HANDOFF_sprite_palette_resolution.md`
- **A unit's real CLUT is `render_state[+0x6] = unit[+0x10]` (site `0x80084408`); live `unit[+0x10] = 0x78c0 | slot` — a slot-indexed CLUT VRAM address (Y=483, X=slot×16), not palette content — and the live VRAM dump shows every chapel slot, including late-adds, holds its own distinct fully-populated 16-color palette, so on PSX the per-unit palette is correct for everyone.** — `[S·D] 2/3`
  - S: commit site `0x80084408` (`battle_disassembly.txt`)
  - D: live VRAM dump via HTTP `/gpu/vram/raw` at Y=483, X=slot*16 (2026-06-28)
  - src: `research/working_documents/chapel_opcode_trace/HANDOFF_sprite_palette_resolution.md`
- **Each cinematic unit's VRAM CLUT equals its own (runtime) SPR's palette at row = its ENTD palette byte — the bit-match is decisive (the matched row sits at a uniform bit-exact floor while the 2nd-best row is 14–469 away in 5-bit-channel L1 across all 158 local SPR palette TGAs), and the row is exactly the combat palette selector (Blue units palette=0 → row 0, Red soldiers palette=2 → row 2, ADR-0022).** — `[D] 1/3`
  - D: session-4 bit-match — each live slot's 16-color CLUT brute-force-matched against all local `<HEX>.palette.tga` sub-palette rows (chapel full cast, scenario_id=4, 2026-06-28; `_bitmatch.py` / `_clut_live.json` in the doc dir)
  - src: `research/working_documents/chapel_opcode_trace/HANDOFF_sprite_palette_resolution.md`
- **`unit[+0x0E]` is written as (allocation counter + 0x14) by `FUN_80087A28` post-allocation — `addiu v0, s3, 0x14; sh v0, 0xe(s2)` at `0x80087BB0` — and chunk-side `{45}` Add Units increment the same counter as ENTD-side adds: chapel boot fired the writer exactly 8× with s3=0..7 (HIME slot 12 → 0x14, SIMON slot 0 → 0x15, AGURI slot 1 → 0x16, late-adds → 0x17..0x1b).** — `[S·D] 2/3`
  - S: `FUN_80087A28`, `0x80087BB0` (`battle_disassembly.txt`)
  - D: allocation BP at `0x80087BB0`, 8 hits s3=0..7 with unit-struct ptrs (2026-06-28, `_capture2_out.json`)
  - src: `research/working_documents/chapel_opcode_trace/HANDOFF_sprite_palette_resolution.md`
- **`{7F}` EVTCHR Palette is a timing gate, not a palette setter: the handler `FUN_8014a3f8` partial-decodes to an `slt v0, v0, s0` gate plus a yield/wait (`FUN_8014ca80`), the chapel cinematic chunk contains ZERO 0x7F opcodes (only Load EVTCHR @ PC 5 and Save EVTCHR @ PC 7), and live `unit[+0x13F]=0` (no XOR override) for all units.** — `[S·D] 2/3`
  - S: `FUN_8014a3f8` (partial decode per `clut_upload_decode.md` §V16, `battle_disassembly.txt`); 0x7F absence by grep of the chapel `static_chunk.tsv`
  - S: dispatch site `0x80145af4`, handler `FUN_8014a3f8` — the `Palette` operand is consumed as a fiber spin-wait threshold, `spin while FUN_8013b590(Block) < Palette` (BATTLE.BIN disassembly, re-verified in `EVTCHR_CLUT_RESOLUTION.md` §1.2, 2026-07-05)
  - D: live roster read `unit[+0x13F]=0` for all 8 cinematic units (2026-06-28)
  - src: `research/working_documents/chapel_opcode_trace/HANDOFF_sprite_palette_resolution.md`
- **In Godot the cinematic palette row is set at spawn from the ENTD palette byte — `unit.body_palette_row = int(slot.get("palette", 0))` in `ScenarioPlayerScene._spawn_units()` (`ScenarioPlayerScene.gd:264`) — and the cinematic path (`SpriteLayerManager.enter_cinematic_mode` / `load_cinematic_frame`, `SpriteLayerManager.gd:539`) swaps ONLY the pixel atlas, deliberately keeping `type1_palette` + `body_palette_row` bound to the unit's own SPR.** — `[R] 1/3`
  - R: `godot-learning/src/scenarios/ScenarioPlayerScene.gd:264` + `godot-learning/src/animation/SpriteLayerManager.gd` (enter_cinematic_mode, load_cinematic_frame :539), validated by `godot-learning/tests/ScenarioPaletteResolutionTest.gd` (green 8/8, 2026-06-28)
  - src: `research/working_documents/chapel_opcode_trace/HANDOFF_sprite_palette_resolution.md`
- **The live PSX VRAM CLUT differs from Godot's `.palette.tga` bake by a uniform Δr=-1, Δg=0, Δb=+1 in 5-bit channels for every non-transparent color (L1 = 30 at 5-bit / 240 at 8-bit, constant across all slots and rows) — a fixed bake round-trip offset that shifts no row match.** — `[D] 1/3`
  - D: session-4 quantization cross-check, live CLUT dump vs `<HEX>.palette.tga` bake (2026-06-28)
  - src: `research/working_documents/chapel_opcode_trace/HANDOFF_sprite_palette_resolution.md`
- **`{7F}` `{Unit:2, Block:1, Palette:1}` (dispatch `0x8014a3f8`) is emitted immediately before a unit's cinematic anim and binds that unit to a VRAM block **and a palette row** — corpus-wide 135 instances across 52 chunks, mostly background generics (binding targets: generic sn255 = 90, other-unique = 39, residue token = 6). Scenario 69: three units share ONE segment (seg26) and are told apart purely by the `{7F}` palette row (2/3/4) — the binding both fixes the segment and supplies the row the whole-CLUT slicer previously left to the consumer.** — `[S·R] 2/3`
  - S: dispatch `0x8014a3f8` (per `research/scenario_1_captures/evtchr_load_save_decode.md`); corpus census 135/52 + scenario-69 row-disambiguation (source doc)
  - R: `godot-learning/tools/event_asset_derivation.py` (`OP_EVTCHR_PALETTE = 0x7F`, unit→(block, palette_row) tracking) + `godot-learning/tools/test_event_asset_derivation.py` (`DeriveTierTest.test_7f_binding_wins_and_supplies_the_palette_row`)
  - src: `research/working_documents/EVTCHR_CHARACTER_ATTRIBUTION.md`

## Notes

(empty — user territory)

## Related

- [[ENTD Unit Deployment Table]]
- [[Sprite Set Resolution]]
- [[Unit Anim Opcode]]
- [[Reset Palette Opcode]]
- [[EVTCHR CLUT Resolution]]
