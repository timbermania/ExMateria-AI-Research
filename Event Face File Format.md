# Event Face File Format

FFT's face files: `EVTFACE.BIN` (event-dialogue portraits — the `{50}`/`{10}`/`{51}` face grid), `WLDFACE.BIN` (world-map narration portraits, a distinct WORLD.BIN system), and `WLDFACE4.BIN` (byte-identical copy of WLDFACE set 3). All three are 4bpp / 16-colour; the distinction is which portraits and how palettes are packed — EVTFACE carries one inline 16-colour CLUT per portrait, WLDFACE carries one 64-CLUT table per set. The engine has at least three portrait sources — EVTFACE (event scenes via `{50}`), WLDFACE (world-map narration), and the speaker's unit SPR (in-battle dialogue, where no EVTFACE is even resident in VRAM) — and which one a dialog box uses depends on the scene/opcode path. EVTFACE's layout is byte-verified against both the on-disc file and live VRAM (scenario 14, 2026-07-11); only EVTFACE is implemented in `godot-learning`. Opcode mechanics: [[Event Dialogue Portrait System]].

## Points

- **`EVTFACE.BIN` (65,536 B, LBA 0x164B / 5707) is an 8×8 grid of 32×48 px 4bpp portraits (768 B each, contiguous: 48 scanlines × 16 B), each with an inline 16-colour BGR555 CLUT (32 B) at the +6144 sub-offset of the 8192-B row-block — `pixel_offset = row*8192 + col*768`, `palette_offset = 6144 + row*8192 + col*32`; a row-block is pixels 0x000..0x17FF + CLUTs 0x1800..0x18FF padded to 0x2000, exactly the `{50}` upload unit.** — `[S·D·R] 3/3`
  - S: ISO9660 directory record (LBA 0x164B, size 0x10000) + Ghidra upload unit — 8×0x300 px + 0x800 CLUT at scratch+0x1800 in `FUN_8013c748` (`battle_decompilation.c`)
  - D: scenario 14 capture (2026-07-11): live VRAM strip pixels/CLUTs byte-match the file at exactly the formula-predicted offsets (col 0 at 0x0000, col 1 at 0x0300, …; CLUTs at 0x1800+)
  - R: `godot-learning/tools/parse_evtface.py` (emits 64 `face_r<R>_c<C>.png` + `evtface.json`, registered in `tools/bootstrap_assets.sh`) — validated by `godot-learning/tools/test_parse_evtface.py` (byte-exact golden test)
  - src: `research/working_documents/PORTRAIT_ROW_OPCODE_50_EVTFACE.md`
- **`WLDFACE.BIN` (131,072 B, LBA 0x18BA / 6330) is a *different* system — the world-map narration portraits loaded by a WORLD.BIN routine (file offset 0x00106140: `ori r4, r0, 0x18ba` + chunked `LoadImage` @ 0x800248FC): 4 sets × 5 rows × 8 cols = 160 portraits, each set a 256×240 4bpp sheet (30,720 B, 128-B row stride) + a 64-CLUT table (2,048 B) at the end of the set (set stride 32,768 B); `WLDFACE4.BIN` (32,768 B, LBA 0x18AA) is byte-identical to WLDFACE set 3 (late-game bosses/nobles); and in-battle-map dialogue uses neither face file — the box face comes from the speaker's unit SPR sheet (scenario 1: 0 EVTFACE bytes resident in VRAM), so the engine has ≥3 portrait sources routed by scene/opcode path.** — `[S] 1/3`
  - S: ISO9660 directory records (WLDFACE LBA 0x18BA size 0x20000; WLDFACE4 LBA 0x18AA size 0x8000); WORLD.BIN routine @ file offset 0x00106140 (user-supplied disassembly snippet; `LoadImage` @ 0x800248FC); `research/working_documents/scenario_1_captures/boxed_dialog_decode.md` §7 (unit-SPR path, 0 EVTFACE in VRAM)
  - R: none — WLDFACE not present in godot-learning (probed godot-learning; only the EVTFACE path is implemented)
  - src: `research/working_documents/PORTRAIT_ROW_OPCODE_50_EVTFACE.md`

## Notes

(empty — user territory)

## Related

- [[Event Dialogue Portrait System]]
- [[Event Opcode Catalog]]
