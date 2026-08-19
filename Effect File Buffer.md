# Effect File Buffer

FFT's effect file buffer is a fixed 0x1CB00-byte (~115 KB) RAM region at 0x801C2500–0x801DEFFF into which E###.BIN files are loaded byte-for-byte; the game computes the available space at runtime and hands it to the loader. The buffer carries a 4-byte size prefix before the file data, header pointer values resolve against the buffer base, and the buffer is not cleared between loads. Every effect file in the ROM was designed to fit; the largest (E242.BIN) is 103.3 KB.

## Points

- **FFT's PSX RAM layout: main executable SCUS_942.41 at 0x80010000–0x80066800, BATTLE.BIN battle overlay at 0x80067800–0x801BC858 (contains the particle system code; Ghidra import base 0x80067800), effect system globals/state at 0x801C0000–0x801C24FF, effect file buffer at 0x801C2500–0x801DEFFF, and dynamic data (heap) at 0x801DF000–0x807FFFFF.** — `[S] 1/3`
  - S: PSX RAM layout map, per `research/key_documents/MEMORY_LAYOUT_REFERENCE.md`
  - src: `research/key_documents/MEMORY_LAYOUT_REFERENCE.md`
- **The effect file buffer is a fixed-size 0x1CB00-byte (117,504-byte, ~115 KB) region from 0x801C2500 to 0x801DF000 — the end address is the constant DAT_8001000c — into which E###.BIN files are loaded byte-for-byte and all must fit.** — `[S·D] 2/3`
  - S: buffer start 0x801C2500, end 0x801DF000 (DAT_8001000c), size 0x1CB00, per `research/key_documents/MEMORY_LAYOUT_REFERENCE.md`
  - D: E063.BIN memory dump — 256/256 bytes matched at 0x801C2500 (2026-04-16 doc)
  - src: `research/key_documents/MEMORY_LAYOUT_REFERENCE.md`
- **The game calculates available effect buffer space at 0x801A1974 (caseD_3 of effect_system_main_loop): it loads the effect file base pointer from DAT_801BBF80 and the buffer end constant (0x801DF000) from DAT_8001000c, subtracts, and passes the available size to FUN_801A4D9C.** — `[S] 1/3`
  - S: 0x801A1974, DAT_801BBF80, DAT_8001000c, FUN_801A4D9C, per `research/key_documents/MEMORY_LAYOUT_REFERENCE.md`
  - src: `research/key_documents/MEMORY_LAYOUT_REFERENCE.md`
- **The loaded effect buffer carries a 4-byte size prefix before the file data — 0x801C2500 holds 0x0001CB00 (buffer size) and the actual E###.BIN header starts at 0x801C2504 — so extracting pure .bin data from RAM requires skipping the first 4 bytes.** — `[S] 1/3`
  - S: size prefix at 0x801C2500 and header start at 0x801C2504, per `research/key_documents/MEMORY_LAYOUT_REFERENCE.md`
  - src: `research/key_documents/MEMORY_LAYOUT_REFERENCE.md`
- **In the loaded image, header field locations are read from base+4+field_offset, but the header's pointer values are relative to the buffer base (NOT base+4), so a section's RAM address = base + header_pointer (e.g. effect_data 0x2E8 → 0x801C2500 + 0x2E8 = 0x801C27E8).** — `[S] 1/3`
  - S: addressing rules and the 0x801C2500+0x2E8=0x801C27E8 example, per `research/key_documents/MEMORY_LAYOUT_REFERENCE.md`
  - src: `research/key_documents/MEMORY_LAYOUT_REFERENCE.md`
- **All effect files were designed to fit within the ~115 KB buffer; the largest is E242.BIN at 103.3 KB (E070.BIN 100.8 KB, E065.BIN Shiva 96.4 KB CODE format, E019.BIN Fire 4 50.9 KB, E001.BIN Cure 43.8 KB).** — `[S] 1/3`
  - S: effect file size table, per `research/key_documents/MEMORY_LAYOUT_REFERENCE.md`
  - src: `research/key_documents/MEMORY_LAYOUT_REFERENCE.md`
- **The effect buffer is NOT cleared between effect loads — unused buffer space contains zeros or stale data from previous effects, and each new effect simply overwrites from the start.** — `[ ] 0/3`
  - src: `research/key_documents/MEMORY_LAYOUT_REFERENCE.md`

## Notes

(empty — user territory)

## Related

- [[E001.BIN Memory Mapping]]
- [[Effect File Format]]
- [[Effect Texture Upload]]
- [[Embedded MIPS Effect Code]]
