# Event Dialogue Portrait System

Event-dialogue box portraits are driven by three opcodes acting on a shared 8×8 face grid in `EVTFACE.BIN`: `{50}` "Portrait Row" latches a row-block (8 × 32×48 4bpp portraits) and uploads it to a fixed VRAM strip at (448,206) with its CLUTs at (448,254); the `Portrait` byte of `{10} Display Message` (and the in-place swap of `{51} Change Dialog`) picks a *column* within that strip, `col = byte − 1` (byte 0 = no EVTFACE face). The face is drawn as a single `POLY_FT4` whose UV/CLUT derive from the column alone. The engine has multiple portrait sources and routes by scene/opcode path: EVTFACE is an *override* — when its gate fails, the box falls back to the speaker's own unit-SPR portrait (in-battle dialogue), and the world map uses the separate `WLDFACE.BIN` system. Byte-proven live on the Balbanes deathbed scene (scenario 14, EVTFACE row 0 = Balbanes) via PCSX-Redux capture on 2026-07-11, and implemented end-to-end in `godot-learning` (`{50}` no longer halts the VM). File layouts: [[Event Face File Format]].

## Points

- **`{50}` "Portrait Row" takes a 1-byte operand that is the `EVTFACE.BIN` row-block index (0-based); its handlers `FUN_8013c748` (sync) / `FUN_8013c8a8` (fiber) read the row-block from disc at `sector = Row*4 + 0x164B` (0x2000 B) and eagerly upload all 8 portraits (each 32×48 4bpp) into the fixed VRAM strip at (448,206) with the 8 CLUTs to (448,254) — it is not a persistent "column latch"; the pixels now resident in the strip *are* the current face set.** — `[S·D·R] 3/3`
  - S: `FUN_8013c748` @ 0x8013C748 / `FUN_8013c8a8` @ 0x8013C8A8, VRAM RECT templates `DAT_80165EBC` (448,206) / `DAT_80165EC4` (448,254), CD-read thunk `FUN_8014ceb4` @ 0x8014CEB4 (`battle_decompilation.c` / `battle_disassembly.txt`)
  - D: scenario 14 capture via `scenario_14_start_maybe.sstate` (2026-07-11): strip empty before `{50}`, then populated byte-exactly with `EVTFACE.BIN` row 0 after resume
  - R: `godot-learning/src/scenarios/ScenarioVM.gd` `_op_portrait_row` (binds `EventInstruction.PORTRAIT_ROW`, latches `_portrait_row`) — validated by `godot-learning/tests/ScenarioPortraitRowTest.gd`
  - src: `research/working_documents/PORTRAIT_ROW_OPCODE_50_EVTFACE.md`
- **The `Portrait` byte of `{10} Display Message` is operand byte [7] and selects the *column* within the resident EVTFACE strip: the handler latches it to the dialog-box field +0x0C and computes `col = Portrait_byte − 1`, so `byte 0` ⇒ col −1 ⇒ no EVTFACE portrait (the box falls back to the speaker's unit-SPR face), `byte 1` ⇒ col 0, …, `byte 9` ⇒ col 8, out of range, also no face; an inline `{EC nn}` in the message text can override the column mid-message.** — `[S·D·R] 3/3`
  - S: `event_display_message_handler` @ 0x801308C0 — operand byte [7] → fiber field `DAT_80169878` (box +0x0C), `col = *(slot+0xC) − 1` (`battle_decompilation.c`)
  - D: scenario 14 capture (2026-07-11): `Portrait=0x01` box renders Balbanes = EVTFACE col 0 (col 1 is Teta, a different face); `Portrait=0x00` boxes (instrs 43/47) render no EVTFACE face
  - R: `godot-learning/src/scenarios/ScenarioDecode.gd` `portrait_column` (`byte − 1`, `0 ⇒ −1`) — validated by `godot-learning/tests/ScenarioPortraitRowTest.gd`
  - src: `research/working_documents/PORTRAIT_ROW_OPCODE_50_EVTFACE.md`
- **The EVTFACE portrait is rendered by `FUN_8012e65c` as a single `POLY_FT4` whose UV/TPAGE/CLUT derive entirely from the column: `U = col*32 … col*32+31`, `V = 206…254`, 4bpp TPAGE at VRAM x=448 (page (7,0)), CLUT at `(448 + (col&3)*16, 254 + (col>>2))` (one 16-colour CLUT per column, packed 4-across × 2 rows).** — `[S·D·R] 3/3`
  - S: `FUN_8012e65c` @ 0x8012E65C (`battle_decompilation.c`)
  - D: scenario 14 capture (2026-07-11): live VRAM pixels and CLUTs at exactly these coordinates byte-match `EVTFACE.BIN` row 0
  - R: `godot-learning/src/ui3/elements/UIPortrait.gd` `display_evtface` (upright, un-rotated RGBA path + `assets/shaders/evtface_portrait_3d.gdshader`; distinct from the unit-SPR indexed/rotated path) — validated by `godot-learning/tests/ScenarioPortraitRowTest.gd`
  - src: `research/working_documents/PORTRAIT_ROW_OPCODE_50_EVTFACE.md`
- **`{51} Change Dialog` (opcode 0x51, 5-byte body `Sub, Message(u16 LE), Portrait(u16 LE)`) spawns no box — it walks the dialog-fiber records and mutates the already-active box in place: new message id (`0xFFFF` = keep, else stored as `msg − 1`) and new portrait column written raw to box +0x0C; the `{50}` row / VRAM strip stays resident, so the column (and message) can be re-picked between boxes without re-issuing `{50}`.** — `[S·D·R] 3/3`
  - S: interpreter `0x51` case (`battle_decompilation.c`); fiber records `DAT_8016e440` / `DAT_8016e450` (stride 0x230, active-box kind 0x33), box fields `DAT_80169870` (+0x04 message) / `DAT_80169878` (+0x0C portrait)
  - D: scenario 14 capture (2026-07-11): chunk bytes `51 02 FF FF 00 00` at instr 41 = keep message, Portrait 0 ⇒ col −1 (no portrait)
  - R: `godot-learning/src/scenarios/ScenarioVM.gd` `_op_change_dialog` + `ScenarioDialogueBoxPool._resolve_evtface` (shared by the `{10}` show path and the `{51}` swap; col-0 swaps leave the face untouched, `Message==0xFFFF` close semantics preserved) — validated by `godot-learning/tests/ScenarioBoxedDialogTest.gd` `_test_change_dialog_swap_updates_portrait` / `_test_change_dialog_swap_col0_keeps_portrait`
  - src: `research/working_documents/PORTRAIT_ROW_OPCODE_50_EVTFACE.md`
- **EVTFACE portrait visibility gate: a box shows its EVTFACE face iff the Dialog byte's mode bit 4 is set (`Dialog & 0x10` — Elidibs, scn112 cell (3,6), carries `Dialog=0x70` and still shows its face, so the strict `(Dialog & 0x70) == 0x10` form is too narrow) AND `portrait_byte ∈ [1,8]`; EVTFACE is an *override* source, not the only source — when the gate fails the box falls back to the speaker's own unit-SPR portrait, so `portrait_byte == 0` does not blank the face, it only suppresses the EVTFACE override (scn14: only Balbanes, speaker `0x80`, speaks with `Portrait=1`; the sons speak with `Portrait=0` and show their own unit faces).** — `[S·D·R] 3/3`
  - S: `FUN_801308c0` @ 0x801308C0 — box-mode read (`Dialog & 0x70`), `col − 1` clamp (`local_ac < 8`), speaker unit-face id `FUN_8008cdd0` → box field +4 (`battle_decompilation.c`)
  - D: scenario 14 capture + user A/B (2026-07-11): Portrait=1 ⟺ Balbanes EVTFACE face; Elidibs `Dialog=0x70` cell (3,6) case from the 43/43 identity validation (2026-07-20)
  - R: `godot-learning/src/scenarios/ScenarioDecode.gd` `portrait_visible` + `ScenarioDialogueBoxPool._resolve_evtface` (routes EVTFACE when a `{50}` row is active, else the unit-SPR path; the runtime gate is still the strict `&0x70==0x10` form) — validated by `godot-learning/tests/ScenarioPortraitRowTest.gd`
  - src: `research/working_documents/PORTRAIT_ROW_OPCODE_50_EVTFACE.md`
- **In scenario 14 the `{50}` disc read does not fire at all: `EVTFACE.BIN` is loaded into a RAM buffer during scenario setup (before the savestate), and `{50}` / the render only run the LoadImage RAM→VRAM path; the static `Row*4+0x164B` sector read is the cold path for when EVTFACE is not already cached (the CD reads that did fire from the setup region were `EVTCHR.BIN`, e.g. sector 0x1D79).** — `[S·D] 2/3`
  - S: CD-read thunk `FUN_8014ceb4` @ 0x8014CEB4 gated on the sector argument; `sector = Row*4 + 0x164B` in `FUN_8013c8a8` (`battle_decompilation.c`)
  - D: scenario 14 capture (2026-07-11): non-pausing BP at 0x8014CEB4 gated to the EVTFACE window [0x164B, 0x166B] caught zero EVTFACE reads while the strip still populated byte-exactly
  - R: none — no disc pre-load / warm path in godot-learning (Godot parses the face file directly; probed godot-learning)
  - src: `research/working_documents/PORTRAIT_ROW_OPCODE_50_EVTFACE.md`
- **The `(row, col) → who` identity of an EVTFACE cell is recoverable from the game's own script: every scripted EVTFACE portrait is shown by a `{10}` message whose decoded text opens with the speaker's `{Color 08}` highlighted name header, so cell `(row, Portrait−1)` = that header's name; the mapping is one-to-one (43 of the 64 cells are ever shown — byte-for-byte the hand-authored per-event Row/Column table, the other 21 cells are filler/duplicates) while identity → cells is one-to-many (e.g. Dycedarg owns (0,3) and (5,1)); this supersedes the earlier ENTD-speaking-Unit axis, which is wrong because the Unit byte is scene-local and misses SPR-less identities (Balbanes, Draclau, Goltana, Elidibs).** — `[R] 1/3`
  - R: `godot-learning/tools/evtface_identity.py` (`derive_evtface_identities` → `assets/scenarios/faces/evtface_identities.json`) — validated by `godot-learning/tools/test_evtface_identity.py` (11 tests, incl. the bit-4 / `0x70` Elidibs case); artifacts present in the `import-godot-game` worktree
  - src: `research/working_documents/PORTRAIT_ROW_OPCODE_50_EVTFACE.md`

## Notes

(empty — user territory)

## Related

- [[Display Message Opcode]]
- [[Event Face File Format]]
- [[Event Opcode Catalog]]
- [[EVTCHR Character Attribution]]
