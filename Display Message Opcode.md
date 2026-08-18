# Display Message Opcode

The event instruction `{10}` Display Message is FFT's text/dialogue opcode: a 1-byte opcode + fixed parameter block (Dialog selector, 1-based string-table Message index, Unit, Portrait row, X/Y/Arrow-X offsets, Open Type). The Dialog byte's high nibble picks the box type (3-line portrait boxes, Check/Help boxes, the no-box screen overlay, …) and its low nibble the vertical alignment plus additive arrow flags; Message indexes the per-event string table that follows the `0xDB Event End`, encoded in the FFT charmap with inline control codes such as `0xE2 NN` = `{Delay NN}`. The open/typewriter handler is `FUN_801308c0` with dialog state around `0x80166000`–`0x801661FF`, and glyphs are read one byte at a time by `event_text_glyph_reader` (`0x8014CE80`). Boxed dialog advances on CIRCLE (`0x8012F6D4`) and closes via a later `{51} Change Dialog`, while the screen-overlay (prayer) path is not input-gated and has no close opcode in scenario 1. The `Dialog=0x09` overlay is implemented and tested in the Godot reimplementation; the boxed portrait variants are not.

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
- **The Unit parameter is only meaningful for portrait box types: if a unit with the matching Unit ID is on the map, the box centers over/under it and shows that unit's portrait; otherwise a blank portrait is shown (still occupying the portrait space) and the box renders on the left of the screen; `Portrait 0x09` removes the portrait and reclaims its space; the sign of X and Arrow X governs portrait facing (any negative value mirrors the portrait to the other side — set either to −1 to flip in place), and odd X/Y values may rattle the box.** — `[ ] 0/3`
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
- **Input advance is gated for boxed dialog only: `event_dialogue_tick` at `0x8012F6D4` polls the pad via `SCUS_get_inverted_button_input` (`0x8012FBF8`), tests the CIRCLE bit (`andi v0,0x20` at `0x8012FC00`), and writes the advance flag to `0x80166080`; the screen-overlay/prayer path is not input-gated — in scenario 1 the next opcode is `{F1} Wait` (86 frames), so it auto-advances.** — `[S] 1/3`
  - S: `0x8012F6D4` (`event_dialogue_tick`), `0x8012FBF8` (`SCUS_get_inverted_button_input`), CIRCLE-bit test at `0x8012FC00`, advance flag `0x80166080`
  - src: `research/wiki_articles/event_instruction_10_display_message.md`
- **Page-able (and `*`-marked) box types close on a later `{51} Change Dialog`; the scenario-1 screen overlay (`Dialog=0x09`) has no explicit close opcode — the text simply persists until overwritten or the scene moves on (exact clear trigger is open question A.3 in the living doc).** — `[S] 1/3`
  - S: scenario-1 chunk decode — no `{51}` follows the overlay prayer
  - src: `research/wiki_articles/event_instruction_10_display_message.md`
- **The `Dialog=0x09` screen overlay (Orbonne chapel prayer, "God, please help us sinful children of Ivalice…") is implemented in the Godot reimplementation — `DialogueOverlay.gd`, wired through `ScenarioVM`, rendered from the real BATTLE.BIN font atlas with the typewriter + `{Delay}`/`{Color}`/`{Newline}` token handling.** — `[R] 1/3`
  - R: `godot-learning/src/scenarios/DialogueOverlay.gd` wired via `godot-learning/src/scenarios/ScenarioVM.gd`; validated by `godot-learning/tests/DialogueOverlayTest.gd`
  - src: `research/wiki_articles/event_instruction_10_display_message.md`

## Notes

(empty — user territory)

## Related

- [[Scenario Table]]
- [[Event Opcode Catalog]]
