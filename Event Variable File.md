# Event Variable File

The FFT event variable file is the backing store for the `0xB0`–`0xBE` arithmetic writers and the `0x7E` Wait Value / `0xA0`–`0xA5` / `0xD0`–`0xD5` readers: a base pointer at `*0x80165f9c` (live value `0x8005771c`) holding 128 32-bit word variables (ids `0x00`–`0x7F` at `base + id*4`), 1-bit flag variables (ids `0x80`–`0x35F`, 32 per word), and 4-bit packed nibble variables (ids `0x360`–`0x3FF`, 8 per word); ids ≥ `0x400` are out of range and abort via `event_fiber_mark_complete`. Id→address resolution is `FUN_8014a2ec` (bit index from `FUN_8014a39c`, `-1` for word vars). Certain "special" destination ids — including var 87 — trigger a notify/recompute path through `FUN_8013b590` after a write, which is why var 87 is continually re-driven by the Orbonne prayer scene rather than holding a static value. Static decode was verified byte-for-byte against `battle_disassembly.txt` and the layout confirmed against live RAM (2026-06-30).

## Points

- **The event variable file's base pointer lives at `*0x80165f9c` (live value `0x8005771c`), and ids `0x00`–`0x7F` are 128 32-bit word variables at `base + id*4` (var 87 → `0x80057878`, var 96 → `0x8005789c`); id→address is resolved by `FUN_8014a2ec`, with the bit index from `FUN_8014a39c` returning `-1` for word vars.** — `[S·D] 2/3`
  - S: base `*0x80165f9c`, resolvers `FUN_8014a2ec` / `FUN_8014a39c` (`project-assets/fft-rom/battle_disassembly.txt`)
  - D: register hijack (documented 2026-06-30): base `== 0x8005771c`; var87 → `0x80057878`, var96 → `0x8005789c`
  - src: `research/wiki_articles/event_instruction_b0_be_variable_math.md`
- **Variable ids `0x80`–`0x35F` are 1-bit flags, 32 per word, at `base + 0x200 + ((id-0x80)>>5)*4`, bit `id & 0x1f`; ids `0x360`–`0x3FF` are 4-bit packed, 8 per word, at `base + 0x25c + ((id-0x360)>>3)*4`, nibble `(id&7)<<2`; ids ≥ `0x400` are out of range and abort via `event_fiber_mark_complete`.** — `[S] 1/3`
  - S: resolvers `FUN_8014a2ec` / `FUN_8014a39c` (`battle_disassembly.txt`)
  - src: `research/wiki_articles/event_instruction_b0_be_variable_math.md`
- **Writing to certain destination ids triggers a notify path through `FUN_8013b590`: ranges `0x70`–`0x8f`, `0x56`–`0x5a` (includes var 87 / `0x57`), `0x32`–`0x39`, `0x1fc`–`0x1ff`, plus singles `0x66`, `0x2c`, `0x30`, `0x53`, `0x19` and a `<0x3c0` gate — var 87 is one of these "special" vars, and the game recomputes it after a write.** — `[S·D] 2/3`
  - S: notify path `FUN_8013b590` (`battle_disassembly.txt`)
  - D: register hijack (documented 2026-06-30): var `0x57` recomputed after write (notify path confirmed); live capture (2026-06-30): var 87 continually re-driven by the prayer scene rather than holding a static value
  - src: `research/wiki_articles/event_instruction_b0_be_variable_math.md`

- **The special-destination path is a **discard**, not a notify/recompute: every failing path falls to `0x8014a2c0` and returns without storing, and the function calls nothing that could notify.** — `[S] 1/3`
  - S: read off `0x8014a1b8..0x8014a2c0`. The only routines the whole function calls are the address decode, the shift, `GetEventVariable`, the `0x21` generator and *exit the current fiber* — there is nothing to notify with, and the return value on a dropped write is whatever the last comparison left in `$v0`, so it is not a status either. Two gates in order: (1) `id >= 0x3C0` **and** its current value 12 or more → discarded, with `id < 0x3C0` skipping the test entirely — so the note's "under `0x3C0`" figure is that *bypass* rather than a whitelist entry; (2) `0x1FC` non-zero → discarded unless the id is in the list. Two corrections to the list itself: **`0x2C` is not a single id** — `sltiu $v0, $s3, 0x2c` at `0x8014a258` admits the whole range `0..0x2B`, 44 ids, including `0x27`, the variable a scenario is *started* by, so a game with `0x1FC` set can still advance its own story; and **`0x19` is not in the list at all** — it appears at `0x8014a2b8`, on the **store** path, where the value written to `0x19` is mirrored into the word at `0x80165ef4`. A gate and a mirror read alike in a dump and do opposite things to a model. `0x1FC` is itself in the list, which is what makes the gate escapable (web-psx `docs/event-seam.md` [event.hle.vars.gate]) (2026-08-19)
  - src: external contribution — web-psx `docs/event-seam.md` [event.hle.vars.gate] (see [[Web-psx Cross-Validation]])

## Notes

(empty — user territory)

## Related

- [[Variable Math Opcodes]]
- [[Wait Value Opcode]]
- [[Variable Comparison Opcodes]]
