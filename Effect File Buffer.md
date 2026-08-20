# Effect File Buffer

FFT's effect file buffer is a fixed 0x1CB00-byte (~115 KB) RAM region at 0x801C2500–0x801DEFFF into which E###.BIN files are loaded byte-for-byte; the game computes the available space at runtime and hands it to the loader. The buffer carries a 4-byte size prefix before the file data, header pointer values resolve against the buffer base, and the buffer is not cleared between loads. Every effect file in the ROM was designed to fit; the largest (E242.BIN) is 103.3 KB.

## Points

- **FFT's PSX RAM layout: main executable SCUS_942.41 at 0x80010000–0x80066800, BATTLE.BIN battle overlay at 0x80067800–0x801BC858 (contains the particle system code; Ghidra import base 0x80067800), effect system globals/state at 0x801C0000–0x801C24FF, effect file buffer at 0x801C2500–0x801DEFFF, and dynamic data (heap) at 0x801DF000–0x807FFFFF.** — `[S] 1/3`
  - S: PSX RAM layout map, per `research/key_documents/MEMORY_LAYOUT_REFERENCE.md`
  - ⚠ SUPERSEDED (2026-08-19) by: `BATTLE.BIN` loads at `0x80067000`, not `0x80067800` — this vault's own working addresses agree (the height table at file `+0x2D748` is quoted as RAM `0x80094748` and the weapon table at `+0x2D364` as `0x80094364`, both of which imply `0x80067000`); the boot executable is `SCUS_942.21` at `0x80010000`, whose file offsets carry an extra `0x800` of PS-EXE header
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
  - ⚠ SUPERSEDED (2026-08-19) by: There is no 4-byte prefix — an effect file's byte 0 lands at `0x801C2500`, measured from the DMA itself, so every address computed as `base + 4 + field` is 4 bytes high
  - src: `research/key_documents/MEMORY_LAYOUT_REFERENCE.md`
- **In the loaded image, header field locations are read from base+4+field_offset, but the header's pointer values are relative to the buffer base (NOT base+4), so a section's RAM address = base + header_pointer (e.g. effect_data 0x2E8 → 0x801C2500 + 0x2E8 = 0x801C27E8).** — `[S] 1/3`
  - S: addressing rules and the 0x801C2500+0x2E8=0x801C27E8 example, per `research/key_documents/MEMORY_LAYOUT_REFERENCE.md`
  - src: `research/key_documents/MEMORY_LAYOUT_REFERENCE.md`
- **All effect files were designed to fit within the ~115 KB buffer; the largest is E242.BIN at 103.3 KB (E070.BIN 100.8 KB, E065.BIN Shiva 96.4 KB CODE format, E019.BIN Fire 4 50.9 KB, E001.BIN Cure 43.8 KB).** — `[S] 1/3`
  - S: effect file size table, per `research/key_documents/MEMORY_LAYOUT_REFERENCE.md`
  - src: `research/key_documents/MEMORY_LAYOUT_REFERENCE.md`
- **The effect buffer is NOT cleared between effect loads — unused buffer space contains zeros or stale data from previous effects, and each new effect simply overwrites from the start.** — `[ ] 0/3`
  - src: `research/key_documents/MEMORY_LAYOUT_REFERENCE.md`
- **An `E###.BIN` lands at `0x801C2500` with its byte 0 at that address and no size prefix, and the effect's own primitives are allocated from `0x801D6FF4..0x801DEFCC` — inside the claimed buffer extent, so that region is shared with the packet heap rather than reserved for the file.** — `[S·D] 2/3`
  - S: three corroborations. Over every instruction traced in the code half of all 401 files, `801c` is the commonest `lui` immediate by a factor of nine (3,080 against 355 for the scratchpad `1f80`) — these are position-dependent images that materialise their own addresses. ffhacktics' published routine listing for each code-shaped file starts at `001c2500`. And scanning every module image for the literal word `0x801C2500` finds it in one place: **509 of the 512 words at `BATTLE.BIN+14d8d0`**, which is the per-effect content-boundary table
  - D: measured from the DMA rather than from the loader's arguments — a provenance shadow tags every word the drive writes with the disc byte it came from, so `base = destination − offset within the file` is a fact about the transfer. Over one recorded session **all 183,176 words of the five effect files loaded implied `1c2500`**, and each of the five agreed with itself on one base; the same run reproduces the declared module bases as a control (`BATTLE.BIN` and `OPEN.BIN` at `067000`, `WORLD.BIN` at `0e0000`, `ATTACK.OUT`/`REQUIRE.OUT` at `1bf000`) (web-psx `docs/modules.md`; cross-referenced 2026-08-19)
  - src: external contribution — web-psx `docs/modules.md` (see [[Web-psx Cross-Validation]])

## Notes

(empty — user territory)

## Related

- [[E001.BIN Memory Mapping]]
- [[Effect File Format]]
- [[Effect Texture Upload]]
- [[Embedded MIPS Effect Code]]
- [[Web-psx Cross-Validation]]
