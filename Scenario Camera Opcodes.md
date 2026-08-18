# Scenario Camera Opcodes

The scenario (cutscene) camera is driven by a matched opcode family in the event VM: `{19}` Camera spawns a per-vblank lerp task that eases the camera toward 8 s16 pose params (X, Z, Y, Angle, MapRot, CamRot, Zoom, Time), `{63}` CameraSpeedCurve arms a persistent ease-curve byte, and `{73}` Camera Move Relative pre-patches the following `{19}`'s operands to live-pose + delta — the mechanism behind the game's camera orbits (e.g. scenario 6's Delita "tough talk" tableau at PC 386–388). As of 2026-07-06 the full mechanism is reverse-engineered with byte-exact dynamic confirmation (pcsx-agent on the scenario 6 savestate); the Godot port was still pending at the doc's date.

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
- **`{38}` Focus Speed is handled by `FUN_80147780` (not `FUN_801474a4`, which is the `{73}` handler): it derives a pan speed of sqrt(Σ(target−live)²)/Time (sqrt via `SUB_8001bf38`) and patches the following Camera's Time operand (+0xe) through `FUN_80146094`.** — `[S] 1/3`
  - S: dispatch `{0x38}` @ `LAB_80144be0` → `jal FUN_80147780` @ `0x80144bf8`; `SUB_8001bf38`; `FUN_80146094` (`battle_disassembly.txt`)
  - src: `research/working_documents/CAMERA_ROTATION_OPCODES_63_73_19_INVESTIGATION.md`
- **The scenario interpreter `event_scenario_interpreter` (`0x80143BD8`) dispatches opcodes through a linear `bne s4,v0` compare-chain (not a jump table), reading the opcode byte at `0x80143d30`, and the opcode-indexed size table at `0x8014d170` gives [0x19]=0x10, [0x63]=0x01, [0x73]=0x0e, [0x1F]=0x05, [0x38]=0x02.** — `[S·D] 2/3`
  - S: compare-chain cases `{0x63}` @ `0x80144c98`, `{0x73}` @ `LAB_80144c08`, `{0x19}` @ `LAB_80144c2c`, `{0x38}` @ `LAB_80144be0`; size table `0x8014d170` (`battle_disassembly.txt`)
  - D: Read watchpoint on the `{63}` opcode byte (`0x8004afb3`) halted exactly at the opcode-fetch instruction `0x80143D28` with the interpreter cursor in `s1` (2026-07-06)
  - src: `research/working_documents/CAMERA_ROTATION_OPCODES_63_73_19_INVESTIGATION.md`
- **The live camera pose lives in a scratch struct — +0x68 pos X, +0x6c pos Y, +0x70 pos Z, +0x74 pitch, +0x78 yaw, +0x7c roll, +0x80 zoom — and the per-vsync ticker `0x801439c0` copies it each vblank into the render/GTE camera mirror (`0x8016e3f0…e408`) without any curve math; all velocity shaping happens in the `{19}` task body.** — `[S·D] 2/3`
  - S: scratch offsets +0x68…+0x80, ticker `0x801439c0`, mirror `0x8016e3f0…e408` (`battle_disassembly.txt`)
  - D: yaw/pitch mirrors `0x800A7786`/`0x800A7784` sampled every vblank via non-pausing exec BP on `0x801439c0` (2026-07-06)
  - src: `research/working_documents/CAMERA_ROTATION_OPCODES_63_73_19_INVESTIGATION.md`
- **In the ISO, `{73}`'s 7 halfwords mirror the following `{19}` Camera's first 6 params (X, Z, Y, Angle, MapRot, CamRot) byte-identically in both scenario-6 occurrences, because `{73}` overwrites those `{19}` operand slots at runtime — the authored `{19}` values are don't-cares (compiler filler).** — `[S·D] 2/3`
  - S: PC 387/388 and PC 428/429 operand pairs in `scenario_006_chunk.json`
  - D: runtime overwrite proven by the patched-operand capture (2026-07-06)
  - src: `research/working_documents/CAMERA_ROTATION_OPCODES_63_73_19_INVESTIGATION.md`
- **In scenario 6 the live pose that `{73}` reads at PC 387 is exactly the pose set by the last prior `{19}` Camera (PC 218: X=776, Z=−316, Y=1432, Angle=270, MapRot=1536, CamRot=0, Zoom=4096) — no camera move intervenes, so "last applied camera params" is byte-exact as the pre-patch base pose.** — `[S·D] 2/3`
  - S: static scan of `scenario_006_chunk.json` — `{19}` @ PC 218 byte-identical to the implied live pose
  - D: implied live pose recovered from the byte-exact `{73}` patch capture (2026-07-06)
  - src: `research/working_documents/CAMERA_ROTATION_OPCODES_63_73_19_INVESTIGATION.md`

## Notes

(empty — user territory)

## Related

- [[Event Opcode Catalog]]
- [[Effect Camera System]]
- [[GTE World-to-Screen Transform]]
