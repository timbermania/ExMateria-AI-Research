# Scenario Camera Opcodes

The scenario (cutscene) camera is driven by a matched opcode family in the event VM: `{19}` Camera spawns a per-vblank lerp task that eases the camera toward 8 s16 pose params (X, Z, Y, Angle, MapRot, CamRot, Zoom, Time), `{63}` CameraSpeedCurve arms a persistent ease-curve byte, and `{73}` Camera Move Relative pre-patches the following `{19}`'s operands to live-pose + delta — the mechanism behind the game's camera orbits (e.g. scenario 6's Delita "tough talk" tableau at PC 386–388). As of 2026-07-06 the full mechanism is reverse-engineered with byte-exact dynamic confirmation (pcsx-agent on the scenario 6 savestate); the Godot port was still pending at the doc's date. The `{1F}` Focus / `{38}` Focus Speed pair is the camera-focus-on-unit family: Focus is a bytecode patcher that rewrites the following `{19}`'s position operands to the target unit(s)' midpoint (camera units = 4× unit units, 112 vs 28 per tile) and optionally auto-fits the zoom, while Focus Speed overrides that Camera's Time — both RE'd byte-exact live (2026-07-02) and ported to Godot with `ScenarioFocusTest`. The live-pose scratch stores position/zoom as opcode × 1024 (pitch/yaw/roll unshifted raw i16), and the pitch latcher writes at 30 Hz — every other NTSC vsync.

## Points

- **`{63}` CameraSpeedCurve (case `0x80144c98`, store `0x80144ca4`) writes its whole parameter byte to the persistent speed-curve global `0x80166054`, whose only reader is the `{19}` Camera task body at `0x80146118`; the global is reset to 0 (linear default) at camera-setup sites `0x8013f444`/`0x8013f474`/`0x8013f4dc`.** — `[S·D] 2/3`
  - S: `sw s2, DAT_80166054` @ `0x80144ca4`, reset sites `0x8013f444`/`0x8013f474`/`0x8013f4dc`, sole reader `0x80146118` (`battle_disassembly.txt`)
  - D: exec BP at `0x80144ca0`/`0x80144ca8` with `0x80166054` poked to 0 and verified as `0xAA` after the step (capture `camera_rotation_63_73_19_captures`, 2026-07-06)
  - src: `research/working_documents/CAMERA_ROTATION_OPCODES_63_73_19_INVESTIGATION.md`
- **`{73}` Camera Move Relative is handled by `FUN_801474a4`, a bytecode pre-patcher that rewrites the following `{19}` Camera's first 7 operand slots to live-camera-field + signed delta (peeking the next opcode at `0x801474c8` to skip an intervening `{38}`), honoring the sentinel `0x2710` ("keep unchanged") per field and leaving the `{19}` Time operand untouched.** — `[S·D] 2/3`
  - S: `FUN_801474a4` @ `0x801474a4` (peek `lbu v1,0xe(s1)` @ `0x801474c8`, add `target = live + delta` @ `0x80147534`, halfword store `FUN_80146094` @ `0x80146094`) (`battle_disassembly.txt`)
  - D: byte-exact pre-patch capture — `{19}` operands `00 00 80 00 00 00 80 00 00 04 00 00 00 00 40 00` (static) → `08 03 44 FF 98 05 8E 01 00 0A 00 00 00 10 40 00` (patched), exec BP at task body `0x80146110` (2026-07-06)
  - src: `research/working_documents/CAMERA_ROTATION_OPCODES_63_73_19_INVESTIGATION.md`
- **In a `{63}`/`{73}`-preceded context the `{19}` Camera behaves as a relative orbit move — the camera settles exactly at start + params (Angle 270+128=398, MapRotation 1536+1024=2560) — because the "relative" behaviour is owned by `{73}`'s live+delta pre-patch, while `{19}` itself stays absolute and just consumes the patched operands.** — `[S·D] 2/3`
  - S: `target = live + delta` add in `FUN_801474a4` @ `0x80147534` (`battle_disassembly.txt`)
  - D: angle-mirror diff before vs settled shot, `0x800A7784` 270→398 and `0x800A7786` 1536→2560 (capture `camera_rotation_63_73_19_captures`, 2026-07-06)
  - src: `research/working_documents/CAMERA_ROTATION_OPCODES_63_73_19_INVESTIGATION.md`
- **A `{19}` Zoom operand of 0 in this context means "no zoom change": after the `{73}` pre-patch the Zoom slot holds live zoom + 0 (4096 = 1.0×), so the orbit shot keeps the current zoom instead of zooming to 0×.** — `[S·D] 2/3`
  - S: `FUN_801474a4` adds the live zoom scratch field (+0x80) to the Zoom delta (`battle_disassembly.txt`)
  - D: patched Zoom operand 0 → 4096 in the byte-exact capture (2026-07-06)
  - src: `research/working_documents/CAMERA_ROTATION_OPCODES_63_73_19_INVESTIGATION.md`
- **The `{19}` Camera task body (`0x80146110`, spawned via `FUN_80149bec`) reads the `{63}` curve byte from `0x80166054`, interpolates the camera scratch toward the (possibly pre-patched) operands over Time frames, and yields one vblank per frame (`event_fiber_yield` @ `0x80146598`); with the curve unset (0) the per-field step is a plain linear lerp (`muldiv_64`, path `LAB_801463bc`), position fields shifting >>2 and rotation/zoom fields >>6.** — `[S·D] 2/3`
  - S: `lw DAT_80166054` @ `0x80146118`, linear path `LAB_801463bc`, `event_fiber_yield` @ `0x80146598` (`battle_disassembly.txt`)
  - D: exec BP at `0x80146110` fires once, after `{73}` has patched the operands (2026-07-06)
  - src: `research/working_documents/CAMERA_ROTATION_OPCODES_63_73_19_INVESTIGATION.md`
- **The `{63}` ease curve has a bit-exact closed form — prog(t) = (16−I)/16·t + I/16·easeQuad(t), with I = (byte & 0xF0) >> 4 and easeQuad the symmetric quadratic (2t² for t<0.5, 1−2(1−t)² for t≥0.5), t = frame/Time — so the intensity nibble linearly blends pure linear motion with full symmetric quadratic ease-in-out; the measured 0xAA curve reproduces hardware within ±1/1024.** — `[S·D] 2/3`
  - S: task body `0x80146454`/`0x801464b0` multiplies the per-field delta by `s8 = (16 − intensity)` and by `intensity` and sums the two weighted terms (`battle_disassembly.txt`)
  - D: per-vblank yaw samples via non-pausing exec BP on ticker `0x801439c0`, 1536→2560 over exactly 64 frames, midpoint frame 32 exactly halfway (2026-07-06); raw 65-sample fixture archived in the doc
  - src: `research/working_documents/CAMERA_ROTATION_OPCODES_63_73_19_INVESTIGATION.md`
- **The `{63}` low nibble selects ease shape and field gating: A = byte & 3 (0 = linear, 1 = ease-in, 2/3 = symmetric ease-in-out) and B = (byte >> 2) & 3 (B=0 eases only position fields, B≠0 eases all 7); 35 distinct `{63}` bytes with 9 distinct low nibbles occur across all scenario chunks.** — `[S·D] 2/3`
  - S: branch tree `0x80146388`–`0x80146448` (linear `beq A,zero` @ `0x80146388`, field gate `slti field,3` @ `0x801463a0`) (`battle_disassembly.txt`); static scan of all scenario chunks
  - D: 0xAA (A=2, B=2) matches the measured ease-in-out S-curve (2026-07-06)
  - src: `research/working_documents/CAMERA_ROTATION_OPCODES_63_73_19_INVESTIGATION.md`
- **`{38}` Focus Speed is handled by `FUN_80147780` (not `FUN_801474a4`, which is the `{73}` handler): it derives a pan speed of sqrt(Σ(target−live)²)/Time (sqrt via `SUB_8001bf38`) and patches the following Camera's Time operand (+0xe) through `FUN_80146094`.** — `[S·D·R] 3/3`
  - S: dispatch `{0x38}` @ `LAB_80144be0` → `jal FUN_80147780` @ `0x80144bf8`; `SUB_8001bf38`; `FUN_80146094` (`battle_disassembly.txt`)
  - D: live dispatch of `{38}` confirmed in the scenario-4 tail → battle-intro opcode stream (`{DB}@0 … {1F}@12 {38}@15 {19}@32`) (resume of `orbonne_darkscreen_dispatch.sstate`, 2026-07-02)
  - R: `godot-learning/src/scenarios/ScenarioVM.gd` FOCUS_SPEED → `ScenarioCameraDirector._op_focus_speed` (stashes the Time override, consumed by the next `_op_camera`; absent → the authored Camera Time stands) (`godot-learning/tests/ScenarioFocusTest.gd` — `_test_focus_speed_overrides_time_and_lerps`)
  - src: `research/working_documents/CAMERA_ROTATION_OPCODES_63_73_19_INVESTIGATION.md`
  - src: `research/working_documents/FOCUS_OPCODE_1F_INVESTIGATION.md`
- **The scenario interpreter `event_scenario_interpreter` (`0x80143BD8`) dispatches opcodes through a linear `bne s4,v0` compare-chain (not a jump table), reading the opcode byte at `0x80143d30`, and the opcode-indexed size table at `0x8014d170` gives [0x19]=0x10, [0x63]=0x01, [0x73]=0x0e, [0x1F]=0x05, [0x38]=0x02.** — `[S·D] 2/3`
  - S: compare-chain cases `{0x63}` @ `0x80144c98`, `{0x73}` @ `LAB_80144c08`, `{0x19}` @ `LAB_80144c2c`, `{0x38}` @ `LAB_80144be0`; size table `0x8014d170` (`battle_disassembly.txt`)
  - D: Read watchpoint on the `{63}` opcode byte (`0x8004afb3`) halted exactly at the opcode-fetch instruction `0x80143D28` with the interpreter cursor in `s1` (2026-07-06)
  - src: `research/working_documents/CAMERA_ROTATION_OPCODES_63_73_19_INVESTIGATION.md`
- **The live camera pose lives in a scratch struct — +0x68 pos X, +0x6c pos Y, +0x70 pos Z, +0x74 pitch, +0x78 yaw, +0x7c roll, +0x80 zoom — and the per-vsync ticker `0x801439c0` copies it each vblank into the render/GTE camera mirror (`0x8016e3f0…e408`) without any curve math; all velocity shaping happens in the `{19}` task body.** — `[S·D] 2/3`
  - S: scratch offsets +0x68…+0x80, ticker `0x801439c0`, mirror `0x8016e3f0…e408` (`battle_disassembly.txt`)
  - S: latch subroutines `FUN_8008ba60` (pitch) / `FUN_8008b834` (zoom) / `FUN_8008b30c` (position) after the copy to `DAT_8016e3f0..DAT_8016e414`; scratch base ptr `DAT_80165f9c = 0x8005771c` (`battle:801439c0..80143a98`, `battle_disassembly.txt`)
  - D: yaw/pitch mirrors `0x800A7786`/`0x800A7784` sampled every vblank via non-pausing exec BP on `0x801439c0` (2026-07-06)
  - D: `probe_camera_chain_writes.jsonl` (2026-06-26) — 952 ticker hits over 30 s of chapel cinematic, each capture carrying all 7 scratch fields
  - src: `research/working_documents/CAMERA_ROTATION_OPCODES_63_73_19_INVESTIGATION.md`
  - src: `research/working_documents/scenario_1_captures/cinematic_camera_motion_decode.md`
- **The live camera-pose scratch stores position and zoom as opcode value × 1024 (10-bit fractional shift) while pitch/yaw/roll are unshifted raw i16 — chapel idle scratch X = 2088960 = 2040 × 1024 matches the PC=56 Camera op's `X = 2040`, and the seg-1 target 1564672 ≈ 1528 × 1024.** — `[D·R] 2/3`
  - D: `probe_camera_chain_writes.jsonl` (2026-06-26) — scratch X 2088960 at chapel idle; seg-1 target 1564672
  - R: `godot-learning/src/scenarios/CameraChainSpline.gd` `finalize_output` (position scratch = q12 >> 2 = opcode × 1024; rotation scratch = raw opcode) + `godot-learning/tests/ScenarioChapelChainSplineTest.gd`
  - src: `research/working_documents/scenario_1_captures/cinematic_camera_motion_decode.md`
- **The live pitch register is at RAM `0x800A7784` (i16 pitch/yaw/roll triplet `0x800A7784..0x800A7788`), written by the pitch latcher `FUN_8008ba60` (entry `0x8008BA68`, called from `FUN_801439C0` at ra `0x80143A04`); pitch updates every ~1,128,800 CPU cycles = 33.3 ms = 30 Hz (every other NTSC vsync), with per-tick Δ overwhelmingly ±1, occasional ±2, and never zero mid-segment.** — `[S·D] 2/3`
  - S: `0x800A7784..0x800A7788`, `FUN_8008ba60` entry `0x8008BA68` (`battle_disassembly.txt`)
  - D: Write-BP capture at `0x800A7784` — 441 writes over ~10 s of chapel cinematic (2026-06-24)
  - R: none — 30 Hz pitch latching not implemented in godot-learning (the capture is only documented in `ScenarioCameraDirector.gd`'s `camera_ease_mode` docstring; probed `godot-learning/src/`, `godot-learning/tests/`)
  - src: `research/working_documents/scenario_1_captures/cinematic_camera_motion_decode.md`
- **In the ISO, `{73}`'s 7 halfwords mirror the following `{19}` Camera's first 6 params (X, Z, Y, Angle, MapRot, CamRot) byte-identically in both scenario-6 occurrences, because `{73}` overwrites those `{19}` operand slots at runtime — the authored `{19}` values are don't-cares (compiler filler).** — `[S·D] 2/3`
  - S: PC 387/388 and PC 428/429 operand pairs in `scenario_006_chunk.json`
  - D: runtime overwrite proven by the patched-operand capture (2026-07-06)
  - src: `research/working_documents/CAMERA_ROTATION_OPCODES_63_73_19_INVESTIGATION.md`
- **In scenario 6 the live pose that `{73}` reads at PC 387 is exactly the pose set by the last prior `{19}` Camera (PC 218: X=776, Z=−316, Y=1432, Angle=270, MapRot=1536, CamRot=0, Zoom=4096) — no camera move intervenes, so "last applied camera params" is byte-exact as the pre-patch base pose.** — `[S·D] 2/3`
  - S: static scan of `scenario_006_chunk.json` — `{19}` @ PC 218 byte-identical to the implied live pose
  - D: implied live pose recovered from the byte-exact `{73}` patch capture (2026-07-06)
  - src: `research/working_documents/CAMERA_ROTATION_OPCODES_63_73_19_INVESTIGATION.md`
- **`{1F}` Focus is a bytecode patcher, not a camera-global writer: it resolves its two `Unit` operands to runtime unit ids (`FUN_80133158`, absent sentinel `0x7d0` → abort, marking the target opcode dead), fetches each unit's world position (`FUN_801330e4`/`FUN_8008c410`, x@+0, z@+2, y@+4), and overwrites the position operands of the *following* `{19}` Camera in place with `2·(pos1+pos2)` per axis via the u16 patch-store `FUN_80146094` — the Camera then executes normally and the camera centres on the unit(s); the authored angle / map-rotation / camera-rotation are left alone.** — `[S·D·R] 3/3`
  - S: handler `FUN_80147584` @ `0x80147584` (reached from the interpreter's Focus case, sub-VM `0x8013dd24`), `FUN_80133158`, `FUN_801330e4`/`FUN_8008c410`, `FUN_80146094` (`battle_disassembly.txt`)
  - D: handler BP `0x80147584` (1 hit during the battle intro) + patch-store BP on `FUN_80146094` captured writes `X=728`, mid=−400, `Z=728`, `zoom=0xE00`, +0xB into the chunk at `0x8004A6CC..DA`; Focus operands `[17 00 17 00 00]` = Unit(1)=Unit(2)=0x17, Unknown=0, both resolving to runtime unit 1 (resume of `orbonne_darkscreen_dispatch.sstate`, 2026-07-02)
  - R: `godot-learning/src/scenarios/ScenarioCameraDirector.gd` `_op_focus` (stashes a pending focus, no immediate move — mirrors the PSX patching a *future* Camera op) + `_op_camera`/`_focus_midpoint_godot` (consumes it, re-aims; absent units → authored pose stands, mirroring the `0x7d0` abort) + `ScenarioDecode.focus` (`godot-learning/tests/ScenarioFocusTest.gd`)
  - src: `research/working_documents/FOCUS_OPCODE_1F_INVESTIGATION.md`
- **The `4×` in Focus's patch is exactly the unit→camera unit conversion: the per-axis write `2·(pos1+pos2)` equals `4×midpoint` for a single-unit focus because the Camera-opcode units are 112/tile while unit-position units are 28/tile (`112 = 4·28`) — the patched value is literally the unit's position expressed in camera units.** — `[S·D·R] 3/3`
  - S: per-axis `2·(pos1+pos2)` combine in `FUN_80147584` (`battle_disassembly.txt`)
  - D: D5/D6 live — slot 1 world-x = 182 → patched Camera X 728 = 4·182; camera lands on the unit's tile (resume of `orbonne_darkscreen_dispatch.sstate`, 2026-07-02)
  - R: `godot-learning/src/scenarios/ScenarioCameraDirector.gd` `_op_camera` reverse-converts the Godot-world midpoint to opcode units (`opcode_X = mid.x·112`, `opcode_Z = −mid.y·112`, `opcode_Y = (map_size_z − mid.z)·112` — the ADR-0052 depth flip) (`godot-learning/tests/ScenarioFocusTest.gd`)
  - src: `research/working_documents/FOCUS_OPCODE_1F_INVESTIGATION.md`
- **When `Unknown == 0`, Focus additionally patches the Camera's Zoom to an auto-zoom-to-fit-both-units value via `FUN_80147318` — a screen-space discrete fit-search, not a formula: it projects both units' live screen anchors (`DAT_8016e40e`/`410`) and iterates the zoom-level table `DAT_80169718` = {0x0E00, 0x0A00, …}, calling the projection/clip test `FUN_8018401c` per candidate and keeping the tightest zoom that still fits both anchors on-screen; `Unknown != 0` leaves the authored zoom alone.** — `[S·D] 2/3`
  - S: `FUN_80147318`, screen anchors `DAT_8016e40e`/`410`, zoom-level table `DAT_80169718`, fit test `FUN_8018401c` (`battle_disassembly.txt`)
  - D: live Focus patch wrote zoom `0xE00` = 3584 = 0.875× into the following Camera (D5, resume of `orbonne_darkscreen_dispatch.sstate`, 2026-07-02)
  - R: none — auto-zoom-to-fit not present in godot-learning (the `auto_zoom` flag is decoded in `ScenarioDecode.FocusIntent` but never applied; authored zoom kept as a documented GAP — probed `godot-learning/src/`, `godot-learning/tests/`)
  - src: `research/working_documents/FOCUS_OPCODE_1F_INVESTIGATION.md`
- **The Camera's `Map Rotation` operand (chapel instr 99: 4432) is render-only — it never touches a unit's base/offset: an axis-by-axis dual-trace of the walk-in slides matched Godot's fixed-axis Sprite Move remap, so unit coordinates stay in fixed map space while the map mesh rotates.** — `[D·R] 2/3`
  - D: chapel dual-trace, axis-by-axis PSX (base+offset) deltas vs Godot deltas (2026-06-28)
  - R: `godot-learning/src/scenarios/ScenarioDecode.gd` `sprite_move_offset` (fixed-axis remap X→world X, Z→world −Y, Y→world Z) + `ScenarioApply.gd` `sprite_move` — validated by `godot-learning/tests/ScenarioSpriteMoveTest.gd`
  - src: `research/working_documents/scenario_1_captures/HANDOFF_unit_tile_alignment.md`

## Notes

(empty — user territory)

## Related

- [[Event Opcode Catalog]]
- [[Effect Camera System]]
- [[GTE World-to-Screen Transform]]
- [[Event End Opcode]]
