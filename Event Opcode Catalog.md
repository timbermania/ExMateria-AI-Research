# Event Opcode Catalog

The master inventory of the vanilla PSX FFT event (scenario/cinematic) instruction set: a flat stream of 1-byte opcodes with fixed parameter blocks, decoded by the BATTLE.BIN event VM. The authoritative catalog is FFTPatcher's `EventCommands.xml` — 176 opcodes spanning `{10}`–`{F2}` — which our `disasm_event.py` and `ScenarioVM` consume. The ffhacktics wiki's "Event Instructions" index is a different, EIU-hack-flavoured set whose extra opcodes are not in the retail ROM. The live dispatcher, handler families, and per-opcode case handlers grounded so far are catalogued below, together with the fixed in-RAM locations of the event chunk and VM state.

## Points

- **An FFT event (scenario/cinematic) is a flat stream of 1-byte opcodes, each followed by a fixed number of parameter bytes (catalog param counts are in stream order, opcode byte not counted); the in-game interpreter (BATTLE.BIN) walks the stream and dispatches per opcode — distinct from BattleConditionals, which use 2-byte little-endian opcodes and a different catalog (`BattleConditionalCommands.xml`).** — `[S] 1/3`
  - S: opcode tables `godot-learning/tools/data/EventCommands.xml` vs `BattleConditionalCommands.xml`; `SCENARIO_LOADING.md` §3.2.5
  - src: `research/wiki_articles/event_instructions.md`
- **Two non-matching opcode sets circulate: the vanilla PSX set (retail US 1.0 ROM — 176 opcodes `{10}`–`{F2}`) and the EIU (Event Instruction Upgrade) hack set that is the wiki's primary index; the EIU-only opcodes — `{00}`–`{0E}`, `{15}`, `{17}` (1.37+), `{2F}`, `{8D}`, `{9A}`–`{9F}`, `{AC}`/`{AE}`/`{AF}`, `{BF}` (gains a MODUL param), `{C1}`–`{C7}`, `{CC}`–`{CF}`, `{E0}` (1.4+), `{E6}`–`{EF}` — are not present in the retail ROM and have no handler in `battle_disassembly.txt`.** — `[S] 1/3`
  - S: FFTPatcher `EventCommands.xml` (176 entries); ffhacktics "Event Instructions" EIU index (verbatim in the doc's §4 appendix)
  - src: `research/wiki_articles/event_instructions.md`
- **FFTPatcher catalog names are the naming our tooling emits as `op_code_name`, and the EIU wiki has renamed several vanilla opcodes: `{21}` Sound→Sound Effect, `{22}` Music→Switch Track, `{5E}` EndSong→End Song, `{84}` PlaySong→Play Song, `{B8}` DIVHI→Modulo, `{B9}` DIVHIVar→Modulo Variable, `{D0}` JumpForwardIfNot→Jump Forward If Zero; `{63}`/`{7B}`/`{7C}`/`{90}` were previously unnamed in the wiki.** — `[S] 1/3`
  - S: opcode table `EventCommands.xml` (FFTPatcher names); wiki "History" annotations
  - src: `research/wiki_articles/event_instructions.md`
- **Per the wiki's "Unused/Unknown" table these catalog IDs have no confirmed behavior — `{12}`, `{23}`, `{24}`, `{25}`, `{26}`, `{37}`, `{39}`, `{3A}`, `{3F}`, `{40}`, `{5C}`, `{5D}`, `{61}`, `{62}`, `{66}`, `{6C}`, `{6D}`, `{71}`, `{72}`, `{73}`, `{74}`, `{75}`, `{81}`, `{82}`, `{8F}`, `{93}`, `{95}`, `{C0}`, `{D8}`, `{D9}`, `{DA}`, `{DC}`, `{F0}` — of which `{23}`, `{24}`, `{25}`, `{26}`, `{37}`, `{3F}`, `{5C}`, `{5D}`, `{61}`, `{62}`, `{6C}`, `{72}`, `{74}`, `{81}`, `{95}`, `{C0}`, `{D8}`, `{D9}`, `{DA}`, `{DC}` are "Never used" in any retail event.** — `[S] 1/3`
  - S: ffhacktics "Event Instructions" index (verbatim in doc §4)
  - src: `research/wiki_articles/event_instructions.md`
- **The live opcode dispatcher in BATTLE.BIN (load base `0x80067000`) reads the opcode byte from the in-RAM event chunk at lbu sites `0x8012F5B0`/`0x8012F5D4` under `event_script_opcode_dispatcher_caller` (ra `0x8012F564`); four handler families read the chunk at fixed offsets (family A lbu `0x80130268`/`0x80130740`, ra `0x801305E0`, chunk +0xB20; family B lbu `0x801307FC`/`0x801314D4`, ra `0x80132764`, +0xB20–0xB40; family C lbu `0x80132624`, ra `0x801323D0`; family D lbu `0x8013ED6C`, ra `0x8013ED50`, +0x880 region) and bytecode readers `0x80143D24`/`0x80146078`/`0x8014607C` load opcode/param bytes, with the whole event subsystem co-located in `0x8012F000`–`0x80146000`.** — `[S·D] 2/3`
  - S: lbu sites `0x8012F5B0`/`0x8012F5D4`, caller `0x8012F564`, handler-family ra sites `0x801305E0`/`0x80132764`/`0x801323D0`/`0x8013ED50`, readers `0x80143D24`/`0x80146078`/`0x8014607C` (`battle_disassembly.txt` via `SCENARIO_LOADING.md` §3.2.1)
  - D: scenario 1 chapel-prayer cinematic, confirmed live (event-chunk capture 2026-06-20)
  - src: `research/wiki_articles/event_instructions.md`
- **The in-RAM event chunk lives at fixed address `0x8004A6BC` (captured as `cinematic_event_chunk_0x8004A6BC.bin`), and the VM state struct `event_script_execution_context` is sketched at base `0x8016A000`.** — `[S·D] 2/3`
  - S: chunk base `0x8004A6BC` (`SCENARIO_LOADING.md` §3.2.4); VM struct base `0x8016A000` (§3.2.6)
  - D: `research/working_documents/scenario_1_captures/cinematic_event_chunk_0x8004A6BC.bin` (2026-06-20)
  - src: `research/wiki_articles/event_instructions.md`
- **`{19}` Camera (params X:2, Z:2, Y:2, Angle:2, Map Rotation:2, Camera Rotation:2, Zoom:2, Time:2) is handled at `0x80146110` (`camera_immediate_non_fusion_step`).** — `[S·D] 2/3`
  - S: `0x80146110` (`camera_immediate_non_fusion_step`) (`battle_disassembly.txt`, master catalog row {19})
  - D: exec BP at task body `0x80146110` fires once, after `{73}` has pre-patched the operands (capture `camera_rotation_63_73_19_captures`, 2026-07-06)
  - src: `research/wiki_articles/event_instructions.md`
  - src: `research/working_documents/CAMERA_ROTATION_OPCODES_63_73_19_INVESTIGATION.md`
- **`{1D}` Camera Fusion Start is a bracket-opener consumed by `{1E}` Camera Fusion End, whose handler is `0x8013db9c` (`camera_fusion_end_queue_build`) with the fusion spline at `0x8013dfb0`.** — `[S] 1/3`
  - S: `0x8013db9c` (`camera_fusion_end_queue_build`), spline `0x8013dfb0` (`battle_disassembly.txt`, master catalog rows {1D}/{1E})
  - src: `research/wiki_articles/event_instructions.md`
- **`{2D}` Rotate Unit is handled at `0x80148284` (`evt0x2D_rotate_unit_handler`), a 16-direction 22.5° facing wheel.** — `[S] 1/3`
  - S: `0x80148284` (`evt0x2D_rotate_unit_handler`) (`battle_disassembly.txt`, master catalog row {2D})
  - src: `research/wiki_articles/event_instructions.md`
- **`{53}` Face Unit is handled at `0x80148084` (`evt0x53_face_unit_handler`), case body `0x8013ec18`.** — `[S] 1/3`
  - S: `0x80148084` (`evt0x53_face_unit_handler`), case body `0x8013ec18` (`battle_disassembly.txt`, master catalog row {53})
  - src: `research/wiki_articles/event_instructions.md`
- **`{1A}` Map Darkness (Blend:1, Red:1, Green:1, Blue:1, Time:1) is handled via case `0x80144f2c` → worker `FUN_801466b4` → applier `FUN_80090840` (`apply_screen_tint`, 11 blend modes; tint globals `0x800a1b58`/`0x800a1b64`/`0x800a1b70`, duration = Time×8).** — `[S·D] 2/3`
  - S: case `0x80144f2c`, `FUN_801466b4`, `FUN_80090840`, globals `0x800a1b58`/`64`/`70` (disassembly, per `map_darkness_oxide_decode.md`)
  - D: scenario 1 map-darkness ("oxide") capture, live-verified per `map_darkness_oxide_decode.md` (2026-07-05)
  - src: `research/wiki_articles/event_instructions.md`
- **The field/3D-object task pipeline — `{54}` Use 3D Object (ID:1, State:1) handler body `0x80144790` → `FUN_8008e0bc` → `FUN_800f0be0(cmd=0x80)`; `{55}` Use Field Object handler `0x801447fc` stores the ID at `0x80174058` and consumer `FUN_80143418` → `FUN_8008e11c` → `FUN_800f0be0(cmd=0x83)` plays textured-anim #ID from the map Mesh Resource (fully decoded + live-validated); `{56}` Wait 3D Object barrier body `0x801447c4` spins on `0x8016606e`; `{57}` Wait Field Object barrier `0x80144830` spins on `0x80166070`, cleared by consumer poll `FUN_800f0be0(cmd=0x82)`.** — `[S·D] 2/3`
  - S: `0x80144790`, `0x801447fc`, `0x80174058`, `FUN_80143418`, `0x801447c4`/`0x8016606e`, `0x80144830`/`0x80166070`, `FUN_800f0be0` cmds 0x80/0x82/0x83 (disassembly, per `use_field_object_decode.md`)
  - D: scenario 1 use-field-object capture, live-validated per `use_field_object_decode.md` (2026-06-29)
  - src: `research/wiki_articles/event_instructions.md`
- **`{50}` Portrait Row (Row:1) sets the portrait sheet row used by `{10}`/`{51}`, and `{51}` Change Dialog (Target:1, Message:2, Portrait Column:1, Portrait Palette:1) is the box-close/swap opcode referenced by the `{10}` dialog modes.** — `[S] 1/3`
  - S: opcode table `EventCommands.xml` rows {50}/{51}
  - src: `research/wiki_articles/event_instructions.md`
- **The wait/barrier tail of the catalog — `{E5}` Wait For Instruction (Task:2) is a barrier that blocks until the named task completes; `{F0}` Wait One Cycle; `{F1}` Wait (Time:2) blocks T frames; `{F2}` No-op.** — `[S] 1/3`
  - S: opcode table `EventCommands.xml` rows {E5}/{F0}/{F1}/{F2}
  - src: `research/wiki_articles/event_instructions.md`
- **`disasm_event.py` walks the event chunk and emits one JSON record per opcode — `{op_code_id, op_code_name, params, dialogue}` — reading opcode names and param layouts from `EventCommands.xml` (the 176-opcode vanilla catalog).** — `[R] 1/3`
  - R: `godot-learning/tools/disasm_event.py` (reads `godot-learning/tools/data/EventCommands.xml`; no named test)
  - src: `research/wiki_articles/event_instructions.md`
- **The Godot `ScenarioVM` binds handlers by `op_code_name` string (not the raw byte) as `_op_<snake_case>(params)` methods; unimplemented opcodes log-and-skip in play-through mode or halt in step mode.** — `[R] 1/3`
  - R: `godot-learning/src/scenarios/ScenarioVM.gd` (no named test)
  - src: `research/wiki_articles/event_instructions.md`
- **As of the doc's snapshot (added 2026-06-27) the Godot reimplementation has implemented — Camera (`{19}`/`{1D}`/`{1E}`), Map Darkness (`{1A}`), Reveal (`{4D}`), Wait (`{F1}`), Unit Anim (`{11}`), Warp Unit (`{5F}`), Rotate/Face (`{2D}`/`{53}`), and the Display Message overlay variant (`{10}`, `Dialog=0x09`) via `DialogueOverlay.gd`.** — `[R] 1/3`
  - R: `godot-learning/src/scenarios/ScenarioVM.gd` + `godot-learning/src/scenarios/DialogueOverlay.gd` (snapshot 2026-06-27; no named test)
  - src: `research/wiki_articles/event_instructions.md`
- **`{47}` Add Ghost Unit (xSP:1, x00:1, xID:1, X:1, Y:1, xEL:1, xFD:1, xDR:1 — 8-byte body) spawns a sprite-only "ghost" actor with no ENTD record through the same unit-add queue as `{45}` Add-Unit; its case lives in the large event executor `FUN_80143bd8` at `0x801456AC`, not in the small interpreter `FUN_8013e904`.** — `[S·D] 2/3`
  - S: case `0x801456AC` in `FUN_80143bd8` (`battle_disassembly.txt`)
  - D: scenario 6 dispatch/spawn breakpoint fires, all three ghost spawns captured (2026-07-04)
  - src: `research/working_documents/ADD_GHOST_UNIT_OPCODE_47.md`
- **In a default (non play-through) `ScenarioVM` run an unregistered opcode halts the VM — the dispatch loop logs `[ScenarioVM] stopping at unhandled opcode '<name>' at offset 0x%X` and clears `_running` — which is why the then-unimplemented `Sound Effect` at Orbonne chapel PC 92 froze the prayer cinematic until the handler was registered (observed 2026-06-28).** — `[R] 1/3`
  - R: `godot-learning/src/scenarios/ScenarioVM.gd` dispatch loop (unhandled path sets `_running = false`; chapel halt observed in the 2026-06-28 trace run and absent after the `_op_sound_effect` landing; no named test)
  - src: `research/working_documents/chapel_opcode_trace/HANDOFF_sound_effect_opcode.md`
- **The scenario 1 (Orbonne chapel) event chunk at `0x8004A6BC` is 200 opcodes long — the chapel trace walks PC 0–199 of it.** — `[S] 1/3`
  - S: per-PC opcode table `static_chunk.tsv` (200 rows) for the chunk at `0x8004A6BC`
  - src: `research/working_documents/chapel_opcode_trace/report.md`

## Notes

(empty — user territory)

## Related

- [[Scenario Camera Opcodes]]
- [[Add Ghost Unit Opcode]]
- [[Display Message Opcode]]
- [[Unit Anim Opcode]]
- [[Wait Value Opcode]]
- [[Event Jump Opcodes]]
- [[Variable Comparison Opcodes]]
- [[Variable Math Opcodes]]
- [[Map Tint]]
- [[Scenario Table]]
- [[Event Sound OpCodes]]
- [[Reset Palette Opcode]]
