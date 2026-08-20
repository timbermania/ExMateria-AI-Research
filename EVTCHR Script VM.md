# EVTCHR Script VM

The per-unit animation script VM: `FUN_80084818` is a bytecode interpreter (not an anim_id → frame lookup) that arms a unit's body frame `+0x1dc` for EVTCHR animations — 66 control opcodes (0xbe..0xff) dispatched through the switch table at `0x80067f20`, with scripts selected from runtime-populated tables (`0x800A77D8` for anim IDs 0x1F4..0x257, `0x800AED3C` for 0x258+) and each script a two-bytes-per-entry stream of frame-step entries and control opcodes. Decoded 2026-06-27 from the disassembly and validated against a live RAM dump of the chapel script table: the chapel exercises exactly 2 of the 66 control opcodes (LOOP + EMIT-AND-YIELD), and the scripts are shared-suffix tails in one bytecode blob at `0x800AEE44`. The per-unit base-frame offset `+0x14` (`unit+0x1EC`) is live-verified zero for all 16 chapel slots, so the displayed frame equals the raw script byte and routing is the downstream `FUN_80083F18` band split.

## Points

- **`FUN_80084818` is a bytecode interpreter (not an anim_id → frame lookup) with 66 control opcodes (`0xbe..0xff`) dispatched via the switch table at `0x80067f20`: its dispatch loop (`0x800848d0..0x80084930`) consumes two bytes per entry — `byte_A ≠ 0xff` is a frame-step (emit `frame_id = unit+0x14 + byte_A`, set `duration = byte_B + unit+0x12`, call `FUN_80084214`, yield) while `byte_A == 0xff` dispatches control opcode `0x80067f20[byte_B − 0xbe]`; most control cases loop back to `LAB_800848d0` without yielding (`caseD_c2`), only `caseD_ff` exits (`LAB_80085200`).** — `[S·D] 2/3`
  - S: `0x800848d0..0x80084930` (dispatch loop), `0x80067f20` (switch table) (`battle_disassembly.txt` @ 0x80084818..0x800851fc)
  - D: probe `probe_layer4_render.lua` 30 s hit table (2026-06-27): 60 fires during the chapel cinematic
  - src: `research/working_documents/chapel_opcode_trace/SPRITE_PIPELINE_INVESTIGATION.md`
- **The script table is selected by anim ID: `< 0x1F4` → the unit's own slot table (`lw 0x20(s5)`), `0x1F4..0x257` → `0x800A77D8`, `≥ 0x258` → `0x800AED3C` with `s4 = anim_id − 0x258` and the script pointer read at `lw 0x8(v0)` — a base-minus-8 convention, so the effective pointer array starts at `0x800AED44`; an entry of `−1` fires the on-demand load stub `SUB_80044a08`.** — `[S] 1/3`
  - S: `0x80084850..0x800848b8` (table selection + `lw 0x8(v0)`), `0x800A77D8`, `0x800AED3C`/`0x800AED44`, `SUB_80044a08` (`battle_disassembly.txt`)
  - src: `research/working_documents/chapel_opcode_trace/SPRITE_PIPELINE_INVESTIGATION.md`
- **Both script tables are runtime-populated (neither has a static initialiser in BATTLE.BIN) — filled by event opcodes (0x58 Load EVTCHR and the deferred-loader dispatcher at `0x80143790`) from the loaded EVTCHR.BIN image; in the chapel savestate `0x800A77D8` is all-zero (the 0x1F4..0x257 band unused) and `0x800AED3C..43` is zero-initialised filler with a 16-byte preamble at `0x800AED34`.** — `[S·D] 2/3`
  - S: opcode 0x58 (Load EVTCHR), `0x80143790` (deferred-loader dispatcher) (`battle_disassembly.txt`)
  - D: session-6 RAM dump `/tmp/dump_evtchr_table.py` from `orbonne_prayer_mid_dialog.sstate` → `evtchr_table_dump.json` (2026-06-27)
  - src: `research/working_documents/chapel_opcode_trace/SPRITE_PIPELINE_INVESTIGATION.md`
- **Per-unit VM state (u16 fields on the unit struct): `+0x4` anim_id (script selector), `+0x6` PC (script-relative byte offset), `+0x8` cached last-emit frame_id, `+0xa` current target duration (`byte_B + unit+0x12`, clamped to 0x100, cleared by `caseD_ff`), `+0xc` loop counter (incremented by `caseD_d5`), `+0x12` duration-base accumulator, `+0x14` base frame (delta-additive: emit = base + byte_A) — and the script's emit is one indirection from `+0x1dc`: the script computes `s0 = unit+0x14 + byte_A`, passes it to `FUN_80084214`, which writes `+0x1dc`.** — `[S·D] 2/3`
  - S: `FUN_80084818` dispatch-path field accesses, `FUN_80084214` (`battle_disassembly.txt`)
  - D: live anim_state RAM dump, `orbonne_prayer_mid_dialog` after one Cross press (2026-06-25): slot 1 `01 00 00 00 5c 02 02 00 e7 00 08 00` (`+0x04` anim_id `0x025C`, `+0x06` PC 2, `+0x08` last-emit `0xE7`, `+0x0A` duration `0x08`), slot 12 `+0x04` `0x0259` / `+0x08` `0xD5` — corroborates `+0x04`/`+0x06`/`+0x08`/`+0x0A`
  - src: `research/working_documents/chapel_opcode_trace/SPRITE_PIPELINE_INVESTIGATION.md`
- **The chapel EVTCHR scripts are shared-suffix tails in one bytecode blob at `0x800AEE44`: consecutive anim IDs point 4 or 6 bytes further into the same blob (anim N+1 = the suffix of anim N, dropping one frame-step entry or one frame-step + control entry from the front), and every script tail-ends in the common `ff d5 ff ff` sequence.** — `[D] 1/3`
  - D: session-6 dump `evtchr_table_dump.json` (pointer deltas + blob bytes, 2026-06-27)
  - src: `research/working_documents/chapel_opcode_trace/SPRITE_PIPELINE_INVESTIGATION.md`
- **The chapel scripts exercise exactly 2 of the 66 control opcodes: `caseD_d5` @ `0x80084a00` (LOOP: `unit+0xc += 1`, `PC := 0`, continue dispatching from the start) and `caseD_ff` @ `0x80084974` (EMIT-AND-YIELD: `sh zero, 0xa(s5)` clears the duration, `j LAB_80085200` exits `FUN_80084818`); the frame-step `byte_A` values seen are 0x02, 0x06, 0x08, 0x0a, 0x0c, 0xd2..0xd9, 0xdb..0xde, 0xe7, 0xe8, 0xea, 0xeb, 0xee..0xf4, with `byte_B` durations of 2..12 ticks per step.** — `[S·D] 2/3`
  - S: `caseD_d5` @ `0x80084a00`, `caseD_ff` @ `0x80084974` (`battle_disassembly.txt`)
  - D: session-6 dump `evtchr_table_dump.json` bytecode statistics over anim IDs 0x025d..0x0264 (2026-06-27)
  - src: `research/working_documents/chapel_opcode_trace/SPRITE_PIPELINE_INVESTIGATION.md`
- **EVTCHR VRAM has exactly two slots: `{58} {Block, Slot}` reads file-block `Slot` (== the 137-segment id, 1:1) into VRAM slot `Block`; the rect table has exactly 2 entries — Block 0 → (256,0), Block 1 → (320,0) — and `{59}` commits, `{5A}` clears, `{5B}` resets; so at most two segments are ever live, and a multi-character cinematic's un-`{7F}`-bound extras are false positives, not extra sheets.** — `[S·D·R] 3/3`
  - S: opcodes `0x58`/`0x59`/`0x5A`/`0x5B` + two-entry rect table (per `research/scenario_1_captures/evtchr_load_save_decode.md`, the load/save decode)
  - S: `Open_EVTCHR` `0x8013C7C4` (`LBA = Slot*15 + 0x1D4C` selects the file block), VRAM rect table `0x800880A8`, `{59}` commit `FUN_8008D5C8` (`SYS_LoadImage`) (BATTLE.BIN disassembly, via `EVTCHR_FRAME_RESOLUTION.md` §1)
  - D: live VRAM read at the scn6 "letgo" carry beat (2026-07-09): chunk instr 4 `{Block:0,Slot:1}` → file-block 1 resident at (256,0), unchanged across buffer swaps
  - R: `godot-learning/tools/event_asset_derivation.py` (models the currently-resident VRAM blocks incl. `{5A}`/`{5B}` clears) + `godot-learning/tools/test_event_asset_derivation.py` (`DeriveTierTest.test_clearing_a_block_disambiguates_a_two_load_chunk`)
  - src: `research/working_documents/EVTCHR_CHARACTER_ATTRIBUTION.md`

- **[S] 1/3** — An EVTCHR sheet (re)load is a one-shot decode+upload, not a per-frame op: the sheet selector `unit+0x05` (allocated as `0x80|team` by `evtchr_slot_allocator @ 0x80083CD4`; source image `DAT_800C7CEA + unit[+0x05]*0x32D6`) is decoded by `FUN_8007AA34 @ 0x8007AA34` into scratch `DAT_800A8928 + slot*0x7564` and LoadImage-uploaded by `func_0x800248fc` to rect `DAT_800A77D0 + slot*0x3AB2` with state gate `DAT_800A77C4[slot]` flipping 0xFF→0xFE; the VRAM tile position is X = `(sel>>3)*0x40 + 0x340`, Y = `(sel&7)*0x20 + 0x100`; the event-side request enters via `DAT_80173CAC = (unit<<8)|slot` and is pumped by `FUN_80143418 @ 0x80143418` → `FUN_8008D708 @ 0x8008D708`, while the per-frame animation path only asks the consumer `FUN_80085C0C` to re-resolve the sheet via `FUN_80085A18 @ 0x80085A18` when a new one is needed.
  - S: `evtchr_slot_allocator @ 0x80083CD4`, `DAT_800C7CEA`, `FUN_8007AA34 @ 0x8007AA34`, `DAT_800A8928`, `func_0x800248fc`, `DAT_800A77D0`, `DAT_800A77C4`, `DAT_80173CAC`, `FUN_80143418 @ 0x80143418`, `FUN_8008D708 @ 0x8008D708`, `FUN_80085A18 @ 0x80085A18` (battle_disassembly.txt)
  - R: none — no runtime EVTCHR sheet (re)load in godot-learning (the port pre-parses the sheet into `cinematic_seq.json` / baked TGA at build time; probed `godot-learning/src/`, `godot-learning/tools/`)
  - src: research/working_documents/SCENARIO_WAIT_SEMANTICS.md

- **The per-unit base-frame offset is zero for all 16 Orbonne chapel roster slots: a live scan of `unit+0x1EC` (VM `+0x14`; VM state at `unit+0x1D8`, roster `0x800B7308` stride `0x440`) at `orbonne_prayer_mid_dialog` after one Cross press shows `0x0000` in every slot (incl. Ovelia slot 1 and Gafgarion slot 12 per the capture doc), and the emit disasm pins the formula at `lhu v1,0x14(s5)` @ `0x80085188` + `s0 = v1 + v0` @ `0x80085198` with `s5 = param_1 + 0x1d8` — so the displayed frame equals the raw script byte and the atlas-vs-TYPE1 routing decision happens entirely in the downstream `FUN_80083F18` band split; `FUN_80086218` zeroes the field before each `FUN_80084818` call, and the non-zero writer remains untraced (GitHub #124).** — `[S·D] 2/3`
  - S: `0x80085188` (`lhu v1,0x14(s5)`), `0x8008518c` (`lbu v0,0(v0)`), `0x80085198` (add into `s0`), `FUN_80086218` pre-call zero site (`battle_decompilation.c` + `battle_disassembly.txt`)
  - D: live RAM scan of all 16 slots, `orbonne_prayer_mid_dialog` after one Cross press (2026-06-25): `+0x1ec = 0x0000` in every slot
  - R: none — per-unit base-frame offset (`+0x14` / `unit+0x1EC`) not present in godot-learning; the per-unit row shift exists only as a debug knob `atlas_y_offset` (default 0, per-uid F3 map) in `godot-learning/src/animation/SpriteLayerManager.gd` (`load_cinematic_frame`) (probed `godot-learning/src/`, `godot-learning/tests/`)
  - src: `research/working_documents/scenario_1_captures/cinematic_frame_offset_decode.md`

## Notes

(empty — user territory)

## Related

- [[Cinematic Sprite Renderer]]
- [[Unit Anim Opcode]]
- [[EVTCHR Frame Resolution]]
