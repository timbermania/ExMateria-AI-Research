# Display Message Opcode

The event instruction `{10}` Display Message is FFT's text/dialogue opcode: a 1-byte opcode + fixed parameter block (Dialog selector, 1-based string-table Message index, Unit, Portrait row, X/Y/Arrow-X offsets, Open Type). The Dialog byte's high nibble picks the box type (3-line portrait boxes, Check/Help boxes, the no-box screen overlay, …) and its low nibble the vertical alignment plus additive arrow flags; Message indexes the per-event string table that follows the `0xDB Event End`, encoded in the FFT charmap with inline control codes such as `0xE2 NN` = `{Delay NN}`. The open/typewriter handler is `FUN_801308c0` with dialog state around `0x80166000`–`0x801661FF`, and glyphs are read one byte at a time by `event_text_glyph_reader` (`0x8014CE80`). Boxed dialog advances on CIRCLE (`0x8012F6D4`) and closes via a later `{51} Change Dialog`, while the screen-overlay (prayer) path is not input-gated and has no close opcode in scenario 1. The `Dialog=0x09` overlay and the boxed portrait variants are implemented and tested in the Godot reimplementation (boxed portraits = EVTFACE override when a `{50}` row is active, else the speaker's unit-SPR face — see [[Event Dialogue Portrait System]]). The scenario-8 chapter-opening narration (`Dialog=0x0B`, box-type-0, valign Center) was captured dynamically on 2026-07-10: `Msg=1` is a multi-page narration (≥3 pages, cleared between) with each page centered at PSX row `116 − 8·n_lines` (pitch 16), and the PC-36 `Task=1` barrier holds until the last page clears, deferring the PC-38 `{33}` Color Field. Godot renders 0x0B through the generalized box-type-nibble test (`(dialog & 0x70) == 0`) with per-page centering and page-break pagination; the boxed portrait variants are implemented (the EVTFACE row/column system, see [[Event Dialogue Portrait System]]).

## Points

- **Opcode `{10}` is a 1-byte opcode followed by a fixed parameter block in stream order — Unknown (1B, the "always seems to be x10" / catalog-default 0x16 programming-error constant the engine ignores), Dialog (1B, box-type/layout selector), Message (2B LE u16, 1-based string-table index), Unit (2B, unit whose portrait + position the box attaches to), Portrait (1B, portrait row; `0x09` removes the portrait and its space), X (2B signed), Y (2B signed), Arrow X (2B signed), Open Type (1B) — and our disassembler decodes exactly this layout (scenario-1 prayer: `Dialog=0x09, Message=0x01, Unit=0x0C, X=0, Y=0x3C, Open Type=0` at chunk offset `0x100`, PC=42).** — `[S·R] 2/3`
  - S: FFTPatcher `EventCommands.xml` parameter block (opcode table); scenario-1 event-chunk decode at offset `0x100` (PC=42)
  - R: `godot-learning/tools/disasm_event.py` decodes exactly this layout
  - src: `research/wiki_articles/event_instruction_10_display_message.md`
- **The Dialog byte's high nibble selects the box type — x0X centered screen overlay (x00 do-not-use), x1X/x7X/x9X 3-line dialog box (includes portrait; x9X closed with `{51}`), x2X/xAX 1–4 line "Check" (xAX supports pages), x3X/xBX 1–4 line "Help" (xBX supports pages), x4X 2-line, x5X/xDX 8-line (xDX no portrait, supports pages), x6X 1–14 line without paging, x8X no box — text over the screen, center-aligned, slow draw, closed with `{51}`, xCX 1–2 line no portrait, supports pages, xEX 1–8 line without paging (≥9 lines won't appear, and `WaitForInstruction(x01)` with ≥9 lines hangs the game), xFX = x1X — and its low nibble selects vertical alignment (1 = top, 2 = bottom, 3 = center; center puts the portrait box over the unit's head with the arrow flipped horizontally) plus additive arrow flags (4 = thinking bubbles, 8 = remove arrow, doesn't work with x9X); if the game doesn't detect a (sufficiently close) `{51}` afterward to close the box it may not display the message at all.** — `[S·R] 2/3`
  - S: ffhacktics "Event Instruction 10" box-type table, corroborated by the scenario-1 chunk decode — across all 18 `{10}` instances `Dialog=0x09` (overlay family, no speaker header) is unique, and every boxed instance is `0x11/0x12/0x91/0x92` with a `{Color 08}Speaker{Newline}{Color 00}` header and a non-zero Open Type
  - R: `godot-learning/tools/disasm_event.py` decoded all 18 scenario-1 instances
  - src: `research/wiki_articles/event_instruction_10_display_message.md`
- **The Message half-word is a 1-based index into the per-event string table, which begins immediately after the `0xDB Event End` opcode that terminates the event bytecode (scenario 1: chunk `+0x8F9`, RAM `0x8004B005+`); there is no Message `0x0000`, and while the full u16 could point at arbitrary RAM lines the wiki notes that is pointless.** — `[S] 1/3`
  - S: scenario-1 string-table location — chunk `+0x8F9`, RAM `0x8004B005+` (per `SCENARIO_LOADING.md` §3.1)
  - S: FFTPatcher `EventCommands.xml` — {DB} Event End (no params; terminates the bytecode, string table follows), {E3} Event End 2
  - src: `research/wiki_articles/event_instruction_10_display_message.md`
- **Event strings are FFT-charmap encoded with inline control codes — `{Delay NN}` (pacing) is encoded as `0xE2 NN` in the byte stream (byte `0xE2` is also a standalone opcode in the bytecode region; context disambiguates), plus `{Newline}`, `{Color NN}` (00 = body, 08 = speaker-name highlight), name macros such as `{Ramza}`, and `{DA xx}` space/word-load helpers (e.g. `{DA73}` adds an invisible side space to fix `{EB}`-loaded word clipping in x8X mode).** — `[S·R] 2/3`
  - S: byte-level encoding in the scenario-1 event string table (RAM `0x8004B005+`)
  - R: `godot-learning/tools/decode_fft_text.py` + `godot-learning/tools/data/psx_charmap.json` resolve these tokens; validated by `godot-learning/tools/test_decode_fft_text.py`
  - src: `research/wiki_articles/event_instruction_10_display_message.md`
- **Each visible glyph is read one byte at a time by `event_text_glyph_reader` at `0x8014CE80` (caller `0x8013284C`), which dispatches the FFT charmap render and is shared by the box and overlay text paths.** — `[S] 1/3`
  - S: `0x8014CE80` (`event_text_glyph_reader`), caller `0x8013284C`
  - src: `research/wiki_articles/event_instruction_10_display_message.md`
- **The Unit parameter is only meaningful for portrait box types: if a unit with the matching Unit ID is on the map, the box centers over/under it and shows that unit's portrait; otherwise a blank portrait is shown (still occupying the portrait space) and the box renders on the left of the screen; `Portrait 0x09` removes the portrait and reclaims its space; the sign of X and Arrow X govern portrait facing (any negative value mirrors the portrait to the other side — set either to −1 to flip in place), and odd X/Y values may rattle the box.** — `[ ] 0/3`
  - src: `research/wiki_articles/event_instruction_10_display_message.md`
- **Open Type is an additive opening-animation/style bitfield: x01 = +50% open speed, x02 = −50% open speed and remove bounce, x04 = darken the dialog box, x10 = toggle arrow (points left if FALSE, right if TRUE); "bounce" is the box growing past its real size (~110%) then settling back to 100%.** — `[ ] 0/3`
  - src: `research/wiki_articles/event_instruction_10_display_message.md`
- **Box type x8X (screen overlay, "prayer" mode) behaves unlike the boxed types: no box — text is drawn directly over the screen, center-aligned, and drawn more slowly (`{Delay 00}` speeds it up, but you need at least one char at `{Delay 02}` or the 6th line briefly glitches); only one can be active at a time, so a `{E5} WaitForInstruction(x01,x00)` must separate two of them; `{EB}` word-loading can clip letter ends, mitigated inconsistently by `{DA73}` padding.** — `[S] 1/3`
  - S: scenario-1 chunk decode — the prayer at chunk offset `0x100` (`Dialog=0x09`, the x0X/overlay family) exhibits the same no-box, raw-body, inline-`{Delay}` character
  - src: `research/wiki_articles/event_instruction_10_display_message.md`
- **The open-and-typewriter handler for `{10}` is `FUN_801308c0` (static-confirmed 2026-06-27); it seeds the dialog-state globals around `0x80166000`–`0x801661FF`, and the per-frame text tick advances the typewriter cursor while consuming `{Delay NN}` (`0xE2 NN`) tokens.** — `[S] 1/3`
  - S: `FUN_801308c0`; dialog-state globals `0x80166000`–`0x801661FF` (static-confirmed 2026-06-27, per `display_message_overlay_decode.md` §C)
  - S: FFTPatcher `EventCommands.xml` master catalog row {10} — `FUN_801308c0` evt0x10 handler
  - src: `research/wiki_articles/event_instruction_10_display_message.md`
- **A single global text-speed throttle is written by `FUN_8013da00` (7 callers); the living doc associates it with a "Set Text Speed" opcode, but the catalog has `0x76 = Dark Screen`, so the opcode-number attribution is pending reconciliation.** — `[S] 1/3`
  - S: `FUN_8013da00` (7 callers in the disassembly)
  - S: FFTPatcher `EventCommands.xml` {76} row — "Dark Screen" (Unknown:1, Shape:1, Screen Expansion Speed:1, Rotation Speed:2, Square Expansion Speed:1); master catalog keeps the disasm cross-ref provisional pending static confirmation of the 0x76 dispatcher case
  - ⚠ SUPERSEDED (2026-08-17) by: the main-executor comparison chain pins `0x75` → `set_event_text_glyph_throttle` (`0x8013da00`) and `0x76` → DarkScreen (`battle:0x80145260`) — the catalog's provisional `0x76 → 0x8013da00` cross-ref was a misattribution
  - src: `research/wiki_articles/event_instruction_10_display_message.md`
- **Input advance is gated for boxed dialog only: `event_dialogue_tick` at `0x8012F6D4` polls the pad via `SCUS_get_inverted_button_input` (`0x8012FBF8`), tests the CIRCLE bit (`andi v0,0x20` at `0x8012FC00`), and writes the advance flag to `0x80166080`; the screen-overlay/prayer path is not input-gated — in scenario 1 the next opcode is `{F1} Wait` (86 frames), so it auto-advances.** — `[S·D] 2/3`
  - S: `0x8012F6D4` (`event_dialogue_tick`), `0x8012FBF8` (`SCUS_get_inverted_button_input`), CIRCLE-bit test at `0x8012FC00`, advance flag `0x80166080`
  - D: live Exec-BP — tick fired 343× (~60 Hz while D held, 0 hits in pure idle) and `0x80166080` held the advance-action code 1/2/8 per pressed button (2026-06-20, per `SCENARIO_LOADING.md` Q#2/§2.5)
  - src: `research/wiki_articles/event_instruction_10_display_message.md`
- **Page-able (and `*`-marked) box types close on a later `{51} Change Dialog`; the scenario-1 screen overlay (`Dialog=0x09`) has no explicit close opcode — the text simply persists until overwritten or the scene moves on (exact clear trigger is open question A.3 in the living doc).** — `[S] 1/3`
  - S: scenario-1 chunk decode — no `{51}` follows the overlay prayer
  - src: `research/wiki_articles/event_instruction_10_display_message.md`
- **The `Dialog=0x09` screen overlay (Orbonne chapel prayer, "God, please help us sinful children of Ivalice…") is implemented in the Godot reimplementation — `DialogueOverlay.gd`, wired through `ScenarioVM`, rendered from the real BATTLE.BIN font atlas with the typewriter + `{Delay}`/`{Color}`/`{Newline}` token handling.** — `[R] 1/3`
  - R: `godot-learning/src/scenarios/DialogueOverlay.gd` wired via `godot-learning/src/scenarios/ScenarioVM.gd`; validated by `godot-learning/tests/DialogueOverlayTest.gd`
  - src: `research/wiki_articles/event_instruction_10_display_message.md`
- **The scenario-8 chapter-opening narration at PC 35 (`{10}` Display Message, chunk offset 198) decodes to `Dialog=0x0B, Msg=1` with 59 tokens ("Delita's name appears…") — box type 0 (no box), valign 3 (Center), Y=0 — and versus the working Orbonne prayer `0x09` the only material decode differences are valign Top→Center (`local_ce` 1→3) plus Y 60→0; palette, cadence, non-blocking, and Task=1-hold are identical.** — `[S·R] 2/3`
  - S: handler `FUN_801308c0`; box-type/valign taxonomy in `scenario_1_captures/display_message_dialog_type_and_palette_decode.md` Part 3; PC35 params in `godot-learning/assets/scenarios/chunks/scenario_008_chunk.json`
  - R: `godot-learning/src/scenarios/ScenarioDecode.gd` `display_message` (`is_overlay = (dialog & 0x70) == 0`, commit `fa88b19b`) + `DialogueOverlay.gd` valign dispatch, validated by `godot-learning/tests/DialogueOverlayTest.gd` `_test_valign_center_placement`
  - src: `research/working_documents/HANDOFF_scenario8_display_message_pc35.md`
- **`Msg=1` (the 59-token scenario-8 narration) is a multi-page message on the PSX — at least three pages that are cleared between pages; the overlay pages forward rather than rendering all 59 tokens as one block or scrolling.** — `[D·R] 2/3`
  - D: dynamic scenario-8 display-message overlay capture, Part Z of `scenario_1_captures/display_message_overlay_decode.md` (2026-07-10)
  - R: `godot-learning/src/scenarios/DialogueOverlay.gd` `_split_pages` / `_PAGE_BREAK_MIN_DELAY` page-break logic, validated by `godot-learning/tests/DialogueOverlayTest.gd` `_test_split_pages_and_content_lines`
  - src: `research/working_documents/HANDOFF_scenario8_display_message_pc35.md`
- **With valign Center, each page is vertically centered: the first line's top sits at PSX row `first_line_top_psx = 116 − 8·n_lines` with line pitch 16 — the Center-Y the box-type-0 RE had never captured, now measured.** — `[D·R] 2/3`
  - D: dynamic scenario-8 display-message overlay capture, Part Z of `scenario_1_captures/display_message_overlay_decode.md` (2026-07-10)
  - R: `godot-learning/src/scenarios/DialogueOverlay.gd` `_first_line_psx_y` (`_CENTER_ANCHOR_PSX=116`, `_LINE_PITCH_PSX=16`, top-margin clamp), validated by `godot-learning/tests/DialogueOverlayTest.gd` `_test_valign_center_placement`
  - src: `research/working_documents/HANDOFF_scenario8_display_message_pc35.md`
- **The PSX holds the PC-36 `{E5} Wait For Instruction Task=1` dialog barrier for the *entire* multi-page narration — the scene RGB is rock-stable while the text is up, and the PC-38 `{33}` Color Field never dispatches until the text has cleared.** — `[D·R] 2/3`
  - D: dynamic scenario-8 display-message overlay capture, Part Z of `scenario_1_captures/display_message_overlay_decode.md` (2026-07-10)
  - R: `godot-learning/src/scenarios/ScenarioVM.gd` `TASK_DIALOG` liveness (`_register_task_kinds` — holds while `dialogue_overlay.is_active()` or the dialog gate is active), validated by `godot-learning/tests/ScenarioWaitForInstructionTest.gd` (Task=1 dialog-gate cycles) + `godot-learning/tests/DialogueOverlayTest.gd` `_test_multipage_advance_holds_active`
  - src: `research/working_documents/HANDOFF_scenario8_display_message_pc35.md`
- **Godot now renders `Dialog=0x0B` as a screen overlay instead of skipping it as unsupported: the overlay-vs-box decision generalized from the prayer-specific `dialog == 0x09` to the box-type nibble `(dialog & 0x70) == 0` (boxed variants are `0x1X/0x7X/0x9X`), so box-type-0 messages all take the no-box overlay path.** — `[S·R] 2/3`
  - S: box-type-nibble split RE-proven — handler `FUN_801308c0`, `scenario_1_captures/display_message_dialog_type_and_palette_decode.md` Part 3
  - R: `godot-learning/src/scenarios/ScenarioDecode.gd` `display_message` (commit `fa88b19b`), validated by `godot-learning/tests/DialogueOverlayTest.gd` (0x0B overlay centering, pagination, self-dismiss) + `godot-learning/tests/ScenarioWaitForInstructionTest.gd`
  - src: `research/working_documents/HANDOFF_scenario8_display_message_pc35.md`
- **The scenario-1 dialogue string table decodes verbatim under the FFT charmap — single-byte ASCII alphanumerics + `!` (`0x00–0x09` digits, `0x0A–0x23` A–Z, `0x24–0x3D` a–z, `0x3E` `!`), control bytes `0xFA` space / `0xF8` soft line break within a page / `0xFE` end of page (next speaker name follows), and 2-byte extended punctuation `0xD9 B6` `.`, `0xDA 74` `,`, `0xD9 C1` `'`, `0xD9 C9` `?`; in this dump page text begins with `0xE3 0x30` and speaker names with `0xE3 0x38` (the charmap labels the 0xE3 NN family `{Color NN}` — the doc reads them as page/speaker boundary markers), pages follow the `{NEXT_SPEAKER_NAME}{NP}{PAGE_TEXT_LINES}{NL}` structure, and the speaker name between the `{E3}8` highlight tag and the next `{NP}` names the NEXT page's speaker.** — `[S·D·R] 3/3`
  - S: FFTPatcher `PSXText.xml` + `CharMap.cs` encoding table (per doc §3.1)
  - D: live RAM decode of the scenario-1 cinematic dialogue verbatim (Ovelia, Agrias, Priest, Gafgarion, Simon, Black Knight) (2026-06-20, `scenario_1_captures/file_load_capture.json` + `dialogue_pages_decoded.txt`)
  - R: `godot-learning/tools/data/psx_charmap.json` (0xFA→space, 0xF8→`{Newline}`, 0xE2 NN→`{Delay NN}`, 0xE3 NN→`{Color NN}`, the four punctuation pairs above) + `godot-learning/tools/decode_fft_text.py` (0xFE/0xFF string terminators, 2-byte lead bytes) — validated by `godot-learning/tools/test_decode_fft_text.py`
  - src: `research/working_documents/SCENARIO_LOADING.md`

## Notes

(empty — user territory)

## Related

- [[Scenario Table]]
- [[Event Opcode Catalog]]
- [[Color Tint Luma Modes]]
- [[Scenario Beat Capture]]
- [[Event Dialogue Portrait System]]
- [[Pad Input Handler]]
