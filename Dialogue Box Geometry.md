# Dialogue Box Geometry

Pixel geometry of the FFT 3-line portrait dialogue box (opcode `{10} Display Message`, Dialog=`0x1X`/`0x9X`): the box dynamically sizes to its text (width = measured text + 0x18 padding, +0x40 with a portrait; height FORCES 3 lines × 16px + 16px chrome, plus a tail term keyed on `Dialog & 0xC` — chapel boxes 72px), text inset is 8px top/left (48px left when the portrait docks left), the speaker triangle is a 16×16 cell from the FRAME atlas at X=0x58 that aims at the unit with a clamped offset, and the frame chrome is a software 9-slice (8/16/8) cut from the FRAME.BIN sprite `(40,0,32,32)`. All decoded by static disassembly of `FUN_801308c0` / `FUN_8014c18c` / `FUN_8014c758` and confirmed live from a scenario-1 VRAM dump (2026-06-27); implemented in `godot-learning` with intentional deviations (tail drawn as an external quad so the frame stays a flat 64px; portrait dock and width reserve trimmed 6px tighter than the ROM).

## Points

- **Box width = measured text width + 0x18 (24px), or + 0x40 (64px) when a portrait is present (portrait row `local_ac < 8`); the box dynamically sizes to its text per message.** — `[S·D·R] 3/3`
  - S: width writes at `0x80130af8`/`0x80130b1c`/`0x80130b34` in `FUN_801308c0` (battle_disassembly.txt)
  - D: scenario-1 chapel-box capture round 1 — live box widths 156/170/162 tracked the text (2026-06-27)
  - R: `godot-learning/src/ui3/assemblies/DialogueBox.gd` (`WIDTH_PAD_PX=0x18` matches ROM; `WIDTH_PAD_PORTRAIT_PX=0x38` deliberately trimmed from the ROM 0x40, 2026-06-27 refinement) + `tests/DialogueBoxTest.gd` `_test_box_geometry`
  - src: `research/working_documents/scenario_1_captures/dialogue_box_geometry_and_fidelity_decode.md`
- **The 0x1X/0x9X box FORCES its line count to 3 regardless of the actual `{Newline}` count (`local_d0==0x10`); height = `3*16 + 16` chrome plus a ROM tail term keyed on `Dialog & 0xC` — `cc==0` (arrow present) → +8 (chapel box = 72px total), `cc==4` (thinking bubble) → +16, `cc==8` (no tail) → +0.** — `[S·D·R] 3/3`
  - S: base `0x80130ae0..0x80130af4`; `cc==0` +8 at `0x80130b24..0x80130b38`; `cc==4` +16 at `0x80130b40..0x80130b54`; `cc==8` at `0x80130b5c` (battle_disassembly.txt)
  - D: scenario-1 chapel-box VRAM dump `last_run/vram_dump_mid_dialog.bin` — live box shows a full empty 3rd line; round-1 composed height `comp_a1=72` (2026-06-27)
  - R: `godot-learning/src/ui3/assemblies/DialogueBox.gd` (`FORCED_LINES=3`; ROM +8/+16 tail terms documented as `ARROW_EXTRA_*` but deliberately not applied to frame height — the tail is drawn as a separate external quad, so the frame stays flat 64px, 2026-06-27) + `tests/DialogueBoxTest.gd` `_test_box_geometry` (asserts 64px for `cc==0/4/8`)
  - src: `research/working_documents/scenario_1_captures/dialogue_box_geometry_and_fidelity_decode.md`
- **Text top inset = `box_top + 8` (override adds another +8 → 16px when `local_d0==0x10 && cc!=8 && ce==2`, the bottom-anchored / upward-tail case); text left inset = `box_left + 8`, or `box_left + 0x30` (48) ONLY when the portrait docks LEFT (`local_b8<0 && local_ac<8`); right-side portrait keeps text at +8 — the box just reserves +0x40 width on the right.** — `[S·D·R] 3/3`
  - S: top `0x80131370..0x8013137c` (+8 override `0x801313e4..0x801313f0`); left `0x80131360..0x8013136c` (48px left-portrait case `0x80131394..0x801313b0`) (battle_disassembly.txt)
  - D: scenario-1 chapel-box VRAM dump render — text hugs the left border at ~8px (2026-06-27)
  - R: `godot-learning/src/ui3/assemblies/DialogueBox.gd` (`TEXT_INSET_PX=8` / `TEXT_INSET_PORTRAIT_PX=0x30` match ROM; `TEXT_TOP_INSET_PX=8` with the `ce==2` +8 deliberately dropped for the external-tail layout, 2026-06-27) + `tests/DialogueBoxTest.gd` `_test_text_and_portrait_insets`
  - src: `research/working_documents/scenario_1_captures/dialogue_box_geometry_and_fidelity_decode.md`
- **Line advance = +0x10 (16px) per `{Newline}`; each glyph is a 10×14 atlas cell drawn 14px tall (`event_dialogue_tick` sets height 0xe), so 16px pitch − 14px glyph = 2px leading, and line N glyph-top = `box_top + 8 + N*0x10` (or `+16 + N*0x10` when `ce==2`).** — `[S·D·R] 3/3`
  - S: line advance `0x80131e24`/`0x80131e2c`; glyph height 0xe set at `0x8012fb20`/`0x8012fb34` (battle_disassembly.txt)
  - D: scenario-1 chapel-box VRAM dump render — two tight lines at 16px pitch (2026-06-27)
  - R: `godot-learning/src/ui3/assemblies/DialogueBox.gd` (`LINE_HEIGHT_PX=16`) + `tests/DialogueBoxTest.gd` `_test_box_geometry`
  - src: `research/working_documents/scenario_1_captures/dialogue_box_geometry_and_fidelity_decode.md`
- **The speaker triangle ("points at the unit") is a 16×16 sprite composited INTO the box buffer from two stacked cells in the resident FRAME atlas at X=0x58 — `(88,0)` = arrow UP (box below unit), `(88,16)` = arrow DOWN (box above) — with the edge chosen by the box-vs-unit side (align low-nibble: align 1 = box above → DOWN on the bottom edge; align 2/0 → UP on the top edge); horizontal aim is `tri_dst_X = (width/2) + local_b8 − 8` clamped to `[0x10, width−0x10]`, mirrored horizontally via `FUN_8014c014` when `local_b8 < 0`.** — `[S·D·R] 3/3`
  - S: clamp at `0x8014c3f4`; mirror `FUN_8014c014 @ 0x8014c014`; composer tail blit `0x8014c2fc..0x8014c448` (cells `DAT_801697b0=0x58/0x68`, dst V `0x38`/`0`) inside `FUN_8014c18c` (battle_disassembly.txt)
  - D: scenario-1 chapel-box capture round 1 — src `(88,0,16,16)`/`(88,16,16,16)`, dst-X 70/52/71 for widths 156/170/162, dst-Y 0 (top) / `0x38`=56 (bottom) (2026-06-27)
  - R: `godot-learning/src/ui3/assemblies/DialogueBox.gd` `_layout_triangle()` — 16×16 up/down TGA, `TRIANGLE_CLAMP_MARGIN_PX=0x10`, `TRIANGLE_EDGE_OVERLAP_PX=8`; the mirror gate keys on the authored arrow operand (`open_type & 0xF0`) rather than the offset sign (decode §1c correction, 2026-06-27) + `tests/DialogueBoxTest.gd` `_test_arrow_up_down_and_remove`, `_test_offset_flips_portrait_arrow_gates_triangle`
  - src: `research/working_documents/scenario_1_captures/dialogue_box_geometry_and_fidelity_decode.md`
- **The box frame is a software 9-slice composite (`FUN_8014c18c` @ `0x8014c18c` via tile-copy `FUN_8014c758` @ `0x8014c758`) from the resident FRAME.BIN atlas (`0x8014d5d4`, uploaded once): source sprite = FRAME.BIN `(40,0,32,32)` with a uniform 8px corner / 16×16 tiled center on both axes — center fill = the `(48..63, 8..23)` 16×16 sub-tile tiled both ways; the per-column arg to `FUN_8014c758` is a HALFWORD index (`sll v1,a3,1` at `0x8014c75c`) so `x_px=(a3%64)*4`, `y=a3/64` — true origin x=40, correcting the prior "+10 bytes → x=20" read (2× off).** — `[S·D·R] 3/3`
  - S: `FUN_8014c18c @ 0x8014c18c` (row math `0x8014c26c..0x8014c2e8`); `FUN_8014c758 @ 0x8014c758` (halfword index `0x8014c75c`, `lhu` offsets `0x8014c764..0x8014c824`); chrome atlas `hud_font_src_ptr @ 0x80173f5c → 0x8014d5d4` (battle_disassembly.txt)
  - D: scenario-1 chapel-box VRAM dump render `last_run/psx_box_from_vram.png` — live border is the rounded beveled tan matching the `(40,0)` file render, NOT the flat `(3,3)` crop (2026-06-27)
  - R: `godot-learning/src/ui3/elements/UIFrame.gd` (9-slice element; shared default is the flat menu-tile crop `(3,3,28,25)`) + `DialogueBox.gd` dialog override `DIALOG_FRAME_REGION=Vector4(40,0,32,32)`, `DIALOG_FRAME_MARGINS=(8,8,8,8)` — no dedicated test for the region
  - src: `research/working_documents/scenario_1_captures/dialogue_box_geometry_and_fidelity_decode.md`

## Notes

(empty — user territory)

## Related

- [[Display Message Opcode]]
- [[Event Dialogue Portrait System]]
- [[Typewriter Text Cadence]]
- [[Concurrent Dialogue Boxes]]
- [[Dialogue Box SFX]]
