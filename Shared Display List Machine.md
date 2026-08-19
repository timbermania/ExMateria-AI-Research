# Shared Display List Machine

The shared display-list state machine is the render engine behind every menu list picker: the driver `0x801264DC` walks a byte template of descriptor records through a 28-entry state-function table, entered by `FUN_80126570` from the formation mainloop `0x80113748` — one engine for the formation job list, equipment, ability, and Learn pickers. The state table, the 0xD9-byte job-picker template `0x8018D1A4`, the record grammar, the palette set `FUN_801298C0` (variant 0 always), and the render-global semantics are now fully mapped and live-validated. The engine is separate from the 0x400-byte window table at `0x80195CD0` — the Learn picker never uses that table (its window-9 block is a stale leftover of an earlier screen).

## Points

- **Every menu list picker (formation job list, equipment, ability, Learn) renders through the shared display-list state machine `0x801264DC`, entered by `FUN_80126570(template, pad, flag)` called from the formation mainloop `0x80113748`; the Learn picker never touches the 0x400-byte window table — its "window 9" identity lives purely in the 6-byte record, and the slot-9 block `0x801980D0` is a stale leftover of an earlier screen (active=0, user=0, generic close handler)** — `[S·D] 2/3`
  - S: 0x80126570, 0x801264DC, 0x80113748, 0x8018D1A4 (WORLD decompiles at `project-assets/fft-rom/WORLD/functions/`, base 0x800E0000); stale win9 block 0x801980D0
  - D: round 2 live (2026-08-15): win9 block active=0, user=0, handler=0x800FFE54 (generic close); the picker's live render attributes to no 0x400 slot
  - R: none — display-list machine not present in godot-learning (probed godot-learning/src + tests; the UI is a separate Godot implementation)
  - src: `research/working_documents/LEARN_PICKER.md`
- **The driver `FUN_801264DC` walks the template as `state = *template` (first byte = initial state), then `while (state != 0x1C) state = *fn_table[state]()` — each state fn takes the record pointer and returns the pointer to the next-state byte it just wrote; default stride is `a0 + rec[1]` (s22 is 1 byte → a0+1; s16 spans a 14-byte header + 15-record sub-block whose pointer resets per row; s1/s2 return past their conditional blocks); state 8 = null trap, unreachable by well-formed templates** — `[S] 1/3`
  - S: 0x801264DC + all 28 state fns read (WORLD decompiles; the four tiny fns with no decompile verified from `world_disassembly.txt` labels LAB_80127BA4 / LAB_80127B24 / LAB_80127C1C / LAB_80129B34); machine-verified walk (round 3, 2026-08-15)
  - R: none — not present in godot-learning (probed godot-learning/src + tests)
  - src: `research/working_documents/LEARN_PICKER.md`
- **The state-function table `0x8018DFB8` (28 entries, stop 0x1C) is fully mapped: state 15 `0x80127C1C` = OT-slot setter, 16 `0x80128CE0` (0xB94 bytes) = the list-row renderer, 18 = open-animation wrapper (scale% + aperture window ring [0x17]), 13/25/26/27 `0x80127C34` = per-row text column, 22 `0x80129B34` = aperture-stage ++, 23 `0x801280FC` = right-aligned number column** — `[S] 1/3`
  - S: 0x8018DFB8 + all 28 state fns (WORLD decompiles; round 3 static pass, 2026-08-15)
  - R: none — not present in godot-learning (probed godot-learning/src + tests)
  - src: `research/working_documents/LEARN_PICKER.md`
- **The job-picker template `0x8018D1A4` is 0xD9 bytes: byte0 `0x0F` starts at state 15 (OT slot 0x0A); stop byte `0x1C` @ 0x8018D27C; the final s15 (offset @0x49) writes OT slot 0x0E — last-write-wins, matching live `c9e88`=0x0E; s16's 14-byte header decodes to 7 top-level sub-records/row, 16-px row pitch, 11 visible rows, 80-px column width; s2's conditional on provider[13] = the JP gate (val==0 → Total/Next columns; else the Jp column)** — `[S·D] 2/3`
  - S: full 0xD9-byte template walk (machine-verified round 3); s16 header @0x53, s2 @0x90, s15 @0x49 relative to 0x8018D1A4 (WORLD decompiles)
  - D: round 2 live `c9e88`=0x0E at settle (2026-08-15)
  - R: none — not present in godot-learning (probed godot-learning/src + tests)
  - src: `research/working_documents/LEARN_PICKER.md`
- **The palette set `FUN_801298C0`: idle (param 0) sets c9e84/85/86 = 0x80 and copies 32-bit words from 2-byte-UNALIGNED rodata whose lower 16 bits are the CLUTs (0x7FFC←0x8018DFB2, 0x7CBC←0x8018DF92, 0x7C3C←0x8018DF8C, 0x7FA4←0x8018DF94, 0x7D7C←0x8018DF9A, 0x7DBC←0x8018DF9C); the active variant (c9ec8≠0) never fires because `c9ec8` is written unconditionally every frame from `0x80126594` with `DAT_8015330C`=0 — variant 0 always** — `[S·D] 2/3`
  - S: 0x801298C0, rodata sources 0x8018DF8C.., 0x80126594 (WORLD decompiles)
  - D: round 2 live — each lower-16 matches a live CLUT word exactly; round 4 live `c9ec8`=0 (2026-08-15)
  - R: none — not present in godot-learning (probed godot-learning/src + tests)
  - src: `research/working_documents/LEARN_PICKER.md`
- **`c9e84` (`0x801C9E84`, live 0x80) is a color/shade pointer, not a CLUT — the static prediction that s4 header quads would use CLUT 0x0080 is a confirmed FAIL: the corrected pool-wide on-screen histogram shows NO CLUT 0x0080 anywhere; the header CLUT comes from `c9eb8`, written by the s10 record immediately before the header quads** — `[S·D] 2/3`
  - S: s4 `pal = &c9e84` (0x80126968) vs s10 record @0x14 of the template (WORLD decompiles)
  - D: round 4 corrected on-screen histogram (2026-08-15): 0x7C3C=247, 0x7CBC=3, 0x7D7C=1, 0x7DBC=1, 0x7FFC=1, no 0x0080
  - R: none — not present in godot-learning (probed godot-learning/src + tests)
  - src: `research/working_documents/LEARN_PICKER.md`
- **Prims form a linked list via tagged next pointers (high byte = tag, low 24 bits = RAM offset; the & 0x1FFFFF mask strips the tag when following chains); every writer ends with the OT back-link prepend `prim->OT = (old & 0xFF000000) | OTtable[slot] & 0xFFFFFF; OTtable[slot] = (old & 0xFF000000) | prim & 0xFFFFFF` (slot = c9e88, c9e88−1 for the aperture window `FUN_8012D25C` ring [0x17], c9e88+1 for s21); `DAT_801CD528` word 0 is a rolling pool/list cursor, NOT an OT table pointer** — `[S·D] 2/3`
  - S: writer-tail prepend pattern (0x8012C6A8 / 0x8012D25C / 0x8012D340 + s21) and prim-ring table 0x801CD528 (WORLD decompiles)
  - D: rounds 2/4 live (2026-08-15): `cd528[0]` 0x801FFD10 → 0x801FFDFC after ~1 s, then STABLE at settle; the region holds code pointers, not an OT table
  - R: none — not present in godot-learning (probed godot-learning/src + tests)
  - src: `research/working_documents/LEARN_PICKER.md`
- **The Set/Remove/Learn popup is 0x400-table window 0xf: user struct `0x8018D0B4` (0x100 bytes, row at +0x18 low word 0x0002 = "Learn"), installed on substate 2 via `FUN_8012AB78`, ticked by `0x8011241C`, input-handled by `0x8011254C`; the table @ `0x80195CD0` is 16 slots with +0x00 user ptr, +0x30 tick cb, +0x44 input handler, +0x48 active flag** — `[S·D] 2/3`
  - S: 0x801998D0 (win 0xf block), 0x8018D0B4, 0x8012AB78, 0x8011241C, 0x8011254C (WORLD decompiles)
  - D: round 2 live win-0xf block (2026-08-15): ss0 active=1 / user=0x8018D0B4 / handler=0x8011254C; ss1/ss4 active=0, handler=0x800FFE54
  - R: none — not present in godot-learning (probed godot-learning/src + tests)
  - src: `research/working_documents/LEARN_PICKER.md`
- **The glove cursor: `FUN_8012A5C0` builds one hand sprite per visible row from the per-row x-table (y starts at 0x130, +0x10 per row); at list edges s16 slides the hand off the edge (down move: y = c9e90·c9e94 + 0x130; up move: 0x120)** — `[S] 1/3`
  - S: 0x8012A5C0, s16 0x80128CE0, geometry+glove setup 0x8012895C (WORLD decompiles; round 3 static pass)
  - R: none — glove x-table construction not present in godot-learning (probed godot-learning/src + tests)
  - src: `research/working_documents/LEARN_PICKER.md`

## Notes

(empty — user territory)

## Related

- [[Learn Job Picker]]
- [[Formation Ability Picker]]
- [[Start Action Menu]]
- [[Menu Window Box Open]]
