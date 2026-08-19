# Embedded MIPS Effect Code

Executable MIPS code embedded at the start of CODE-format E###.BIN effect files — the summon-style effects (Ifrit, Shiva, Bahamut) — renders 3D geometry procedurally: no vertex or mesh data is stored, so vertices are computed every frame with sin/cos and lerp, projected through the GTE COP2 (RTPT), and submitted as runtime-built GPU primitives (POLY_F4) via AddPrim. The script interpreter reaches this code through callback opcodes 0x06/0x07, which dispatch through resolve_callback (0x801a1288) and a fixed lookup table (0x801b50d0, 64 entries) of pre-computed pointers into the effect load area at 0x801C2500. The MIPS payload is a library of callback functions that the timeline invokes INSTEAD of normal particle emitter spawning; the 107 MIPS files — detected by an `addiu sp, sp, −X` prologue at file offset 0x0000 — split into two archetypes: 80 direct GPU renderers that bypass the particle system entirely, and 27 enhanced emitter callers that spawn through it after custom lerp/trig calculations.

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
- **MIPS/CODE-format effect files are identified by an `addiu sp, sp, −X` function prologue (first word 0x27BDXXXX) at file offset 0x0000; a ROM-wide scan classifies 107 of 398 E###.BIN files as MIPS/CODE format.** — `[S·R] 2/3`
  - S: prologue word 0x27BDXXXX at file offset 0x0000, 107-of-398 scan, per `research/working_documents/MIPS_CALLBACK_SYSTEM.md`
  - R: `effect-editor/core/parser.lua:45-50` (CODE vs DATA detection, first word 0x27BD0000–0x27BE0000) + `effect-editor/mips/disassembler.lua:408` (no automated test)
  - src: `research/working_documents/MIPS_CALLBACK_SYSTEM.md`
- **The callback opcodes' operand encoding: op_load_callback (opcode 0x06, 0x801A23A8) reads packed_slot at byte +1 (= slot × 4, so slot = packed_slot >> 2) and callback_id as int16 at +2, resolves it via resolve_callback, and stores the address in the runtime callback_slots array at 0x801B9144 + slot × 4, advancing the script by 4; op_invoke_callback (opcode 0x07, 0x801A2414) resolves and immediately invokes callback(effect_target, slot, 0, 0), saving the address to effect_state->last_callback at offset 0xF4.** — `[S·R] 2/3`
  - S: disassembly 0x801A23A8–0x801A23F8 (lbu packed_slot @+1, lh callback_id @+2, sw to 0x801B9144 + slot × 4) and 0x801A2414–0x801A2458 (jalr, sw v0, 0xf4(s1)), per `research/working_documents/MIPS_CALLBACK_SYSTEM.md`
  - R: `godot-learning/src/effects/EffectData.gd:141-148` (parses load_callback ops into callback_slots {slot, callback_id}) + `godot-learning/src/effects/callbacks/CallbackManager.gd` (4-slot array, registry dispatch) + `godot-learning/tests/IfritTest.gd` (E067 CB19 lifecycle monitor; playback test, no hard asserts)
  - src: `research/working_documents/MIPS_CALLBACK_SYSTEM.md`
- **The callback lookup table at 0x801B50D0 holds 64 entries (callback_id 0–63); multiple IDs alias to the same file offset (e.g. 0x14F8 for IDs 3/8/20/30/39/56, 0x1FA0 for IDs 0/24), and entries pointing to the load base 0x801C2500 carry no code ("no callback" in the table).** — `[S] 1/3`
  - S: extended 64-entry table (IDs 0–63 with RAM addresses/file offsets), per `research/working_documents/MIPS_CALLBACK_SYSTEM.md`
  - R: none — 64-entry ROM lookup table not present in godot-learning or effect-editor (callback ids map to Godot callback classes via CallbackRegistry instead)
  - src: `research/working_documents/MIPS_CALLBACK_SYSTEM.md`
- **MIPS effect files must place function prologues at the fixed file offsets the lookup table points at — 0x0774, 0x077C, 0x07EC, 0x0B6C, 0x14F8, 0x15D0, 0x177C, 0x1CE4, 0x1EF0, 0x1FA0, 0x213C, 0x2AC8, 0x2C74, 0x371C — verified by `addiu sp, sp, −X` bytes: 0x1FA0 in 9 files (E070, E242, E317, E412, E461…), 0x0774 in 8, 0x14F8 in 7, 0x177C in 6 (E033, E035, E077, E245, E384 — E033.BIN+0x177C = 0x27BDFF30 = addiu sp, sp, −208), down to single files (E071@0x07EC, E074@0x1CE4, E065@0x1EF0 stack=1664, E080@0x0B6C/0x213C, E077@0x2C74 stack=440).** — `[S] 1/3`
  - S: per-offset prologue bytes and file counts (1–9 files per offset), per `research/working_documents/MIPS_CALLBACK_SYSTEM.md`
  - R: none — required prologue offsets not present in godot-learning or effect-editor
  - src: `research/working_documents/MIPS_CALLBACK_SYSTEM.md`
- **The 107 MIPS callbacks split into two archetypes: Type A (80 files) direct GPU renderers that bypass the particle system entirely — get a GPU packet buffer via get_gpu_buffer (0x80044A60), configure primitives with gpu_prim_setup (0x80023D44), apply 3D matrix transforms and RGB colour manipulation, then write packets directly; Type B (27 files) enhanced emitter callers that compute custom positions with lerp/trig, then call emitter_control (0x801A60AC) so the normal pipeline renders the spawn. Across all 107 files the callbacks call only 15 battle.bin functions (top callers: lerp ×1597, rsin ×395, rcos ×395, cleanup_sprite_4 ×353, alloc_particle ×177, coord_transform ×166, interp_position ×122, cosine_ease ×78, emitter_control ×55, vec3_magnitude ×52).** — `[S] 1/3`
  - S: call-frequency table (gpu_prim_setup 0x80023D44, get_gpu_buffer 0x80044A60, colour helper labelled 0x80023BB4 in this doc, matrix ops 0x8001D0A8/0x8001D138, emitter_control 0x801A60AC), per `research/working_documents/MIPS_CALLBACK_SYSTEM.md`
  - R: none — Type A/B split not present in godot-learning (callbacks re-implemented per-effect in Godot mesh/GPU code, not as the ROM's two pipeline paths); probed `godot-learning/src/effects/callbacks/`
  - src: `research/working_documents/MIPS_CALLBACK_SYSTEM.md`

## Notes

(empty — user territory)

## Related

- [[Effect File Buffer]]
- [[Effect File Format]]
- [[Effect Execution Model]]
- [[E001.BIN Memory Mapping]]
- [[E317 Choco Ball Callback System]]
