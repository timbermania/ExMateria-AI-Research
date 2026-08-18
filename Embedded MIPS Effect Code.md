# Embedded MIPS Effect Code

Executable MIPS code embedded at the start of CODE-format E###.BIN effect files — the summon-style effects (Ifrit, Shiva, Bahamut) — renders 3D geometry procedurally: no vertex or mesh data is stored, so vertices are computed every frame with sin/cos and lerp, projected through the GTE COP2 (RTPT), and submitted as runtime-built GPU primitives (POLY_F4) via AddPrim. The script interpreter reaches this code through callback opcodes 0x06/0x07, which dispatch through resolve_callback (0x801a1288) and a fixed lookup table (0x801b50d0) of pre-computed pointers into the effect load area at 0x801C2500.

## Points

- **The CODE-format effect files are the "summon" style effects (Ifrit, Shiva, Bahamut, etc.) — 107 files, ~27% of the corpus — that need complex 3D geometry and animation beyond the standard particle/sprite system.** — `[ ] 0/3`
  - src: `research/key_documents/MIPS_EMBEDDED_CODE.md`
- **CODE-format effects store no pre-defined vertex/mesh data: all 3D geometry is computed procedurally at runtime (sin/cos for circular/arc paths, lerp for keyframe transitions) and projected to screen space by the GTE, with vertices generated on-the-fly every frame.** — `[ ] 0/3`
  - src: `research/key_documents/MIPS_EMBEDDED_CODE.md`
- **The CODE-format prologue encodes the function stack frame (0x10000 − (first_word & 0xFFFF)), which indicates code complexity: ~120 bytes for simple callbacks, 1,664 bytes for complex summons (Ifrit, Shiva).** — `[ ] 0/3`
  - src: `research/key_documents/MIPS_EMBEDDED_CODE.md`
- **Effect script bytecode invokes embedded MIPS code via two opcodes — 0x06 (op_load_callback, 0x801a23a8) loads a function pointer into a slot and 0x07 (op_invoke_callback, 0x801a2414) executes it via JALR, with the effect state passed in $a0.** — `[S] 1/3`
  - S: op_load_callback 0x801a23a8, op_invoke_callback 0x801a2414, per `research/key_documents/MIPS_EMBEDDED_CODE.md`
  - src: `research/key_documents/MIPS_EMBEDDED_CODE.md`
- **resolve_callback (0x801a1288) resolves a callback ID through the callback lookup table at 0x801b50d0, which holds pre-computed function pointers into the effect load area based at 0x801C2500 (entry 0 → 0x801C44A0, entry 1 → base); effect files must place their functions at the offsets the table entries expect.** — `[S] 1/3`
  - S: resolve_callback 0x801a1288, callback_lookup_table 0x801b50d0, load base 0x801C2500, per `research/key_documents/MIPS_EMBEDDED_CODE.md`
  - src: `research/key_documents/MIPS_EMBEDDED_CODE.md`
- **Effect code computes vertex positions with ROM helpers: lerp (0x801A8BE0, result = start + ((end − start) × t) >> 8 with t = 0–255 normalized time) for keyframe interpolation, and rsin (0x8001BB5C) / rcos (0x8001BC28) lookup tables (4096 entries = full circle) scaled by radius for circular/arc motion.** — `[S] 1/3`
  - S: lerp 0x801A8BE0, rsin 0x8001BB5C, rcos 0x8001BC28, per `research/key_documents/MIPS_EMBEDDED_CODE.md`
  - S: lerp 0x801A8BE0, rsin 0x8001BB5C, per `research/key_documents/working_documents/MIPS_EDITOR_EXPLORATION.md`
  - src: `research/key_documents/MIPS_EMBEDDED_CODE.md`
- **Computed vertices are projected by the GTE COP2 — RTPT (0x480DF800) applies rotation matrix + translation + perspective projection to three vertices (V0–V2) with screen coordinates stored via swc2 $12–$14 — after which the code builds GPU primitives at runtime (e.g. 24-byte POLY_F4 flat-shaded quads, primitive tag type 0x28) and submits them to the ordering table via AddPrim (0x80023BB4).** — `[S] 1/3`
  - S: AddPrim 0x80023BB4, RTPT/MVMVA/CTC2/MTC2/SWC2 usage, per `research/key_documents/MIPS_EMBEDDED_CODE.md`
  - S: AddPrim 0x80023BB4, per `research/key_documents/working_documents/MIPS_EDITOR_EXPLORATION.md`
  - src: `research/key_documents/MIPS_EMBEDDED_CODE.md`
- **E067.BIN (Ifrit) layout: MIPS code at 0x0000–0x1C68 (7,272 bytes), data tables (animation curves) from 0x1C70 (~3 KB), then standard effect data (emitters, textures, ~85 KB).** — `[ ] 0/3`
  - src: `research/key_documents/MIPS_EMBEDDED_CODE.md`
- **E067's (Ifrit) callback runs a state machine tracking animation phase and computes fire-swirl paths with sin/cos: per invocation it calls lerp 12 times, rsin and rcos 3 times each, 9 RTPT instructions (27 vertices), and AddPrim once per main-loop iteration.** — `[ ] 0/3`
  - src: `research/key_documents/MIPS_EMBEDDED_CODE.md`
- **The MIPS inside CODE-format effects is hand-written assembly with timing-sensitive sections — branch delay slots and register allocation matter — so a decompile-to-C-and-recompile round trip cannot reproduce byte-identical code.** — `[ ] 0/3`
  - src: `research/key_documents/working_documents/MIPS_EDITOR_EXPLORATION.md`
- **rsin/rcos calling convention: the angle is passed in $a0 as 0–4095 (0–360°) and the result is returned in $v0 as fixed-point scaled ×4096 (e.g. $v0 = cos(angle) × 4096).** — `[ ] 0/3`
  - src: `research/key_documents/working_documents/MIPS_EDITOR_EXPLORATION.md`
- **The MIPS code in E067.BIN (Ifrit) carries editable parameters at fixed code offsets: 0x0640 `li a1, 0x0100` radius scale (256), 0x0710 `li a0, 0x002D` frame count (45), 0x0820 `li t0, 0xFF80` colour R (255).** — `[ ] 0/3`
  - src: `research/key_documents/working_documents/MIPS_EDITOR_EXPLORATION.md`

## Notes

(empty — user territory)

## Related

- [[Effect File Format]]
- [[Effect Execution Model]]
- [[E001.BIN Memory Mapping]]
- [[E317 Choco Ball Callback System]]
