# Block Execution

The event VM runs parallel `Block Start`/`Block End` (0x2A/0x2B) brackets as genuine cooperative coroutines on a 16-slot fixed-stack scheduler: `Block Start` allocates a free slot, runs a generic block-body interpreter (`FUN_8013e904`) over the bracket's bytes, and the main thread skips its own pointer past the matching `Block End`; `Block End` only terminates the block's own coroutine and never synchronizes the main thread. The static model (LAB_80144e70, FUN_8014ca80, FUN_80149ebc in battle_disassembly.txt) was confirmed bit-for-bit by a live PCSX-Redux run in the chapel scenario (2026-06-27), where three parallel walk-on blocks entered slots 3/4/5 in the same vsync as their `Block Start`s. The Godot `ScenarioVM` mirrors the model — `Block Start` spawns a child context that ticks in the same vsync as the main thread, and `_tick_once` round-robins all live contexts each tick (validated on the scenario-6 ride-off, 2026-07-05).

## Points

- **FFT/PSX runs parallel `Block Start`/`Block End` (0x2A/0x2B) brackets as genuine cooperative coroutines — block contents execute interleaved with the main thread from the same vsync as `Block Start`, and `Block End` merely terminates the block's own coroutine (`FUN_8014c958`) without synchronizing the main thread.** — `[S·D] 2/3`
  - S: `LAB_80144e70`, `FUN_8013e904`, `FUN_8014ca80`, `FUN_8014c958` (`project-assets/fft-rom/battle_disassembly.txt`)
  - D: exec-BP capture at `0x80144ea0`/`0x8013e904`/`0x80149ebc` (`research/lua_scripts/probe_block_execution.lua`), chapel `mid_dialog` savestate, PCSX-Redux port 8082 (2026-06-27)
  - src: `research/working_documents/chapel_opcode_trace/BLOCK_EXECUTION_INVESTIGATION.md`
- **`Block Start` (0x2A) allocates a free slot (1..15) via `FUN_80149bec(0x10)`, initializes that slot's coroutine with the generic block-body interpreter `FUN_8013e904` via `FUN_8014c8a0`, plants the byte right after the 0x2A as the slot's bytecode pointer, and then has the main thread fast-forward its own pointer past the matching 0x2B via `FUN_80149ebc`.** — `[S·D] 2/3`
  - S: `LAB_80144e70`–`0x80144eac`, `FUN_80149bec`, `FUN_80149ebc`–`0x80149f0c` (`project-assets/fft-rom/battle_disassembly.txt`)
  - D: live probe — three back-to-back `Block Start`s at vsync 409 allocated slots 3/4/5 and planted byte pointers 0x8004A8FE/0x8004A94E/0x8004A997, exactly matching static chapel chunk base 0x8004A6BC + offset + 1 (`static_chunk.tsv` cross-check) (2026-06-27)
  - src: `research/working_documents/chapel_opcode_trace/BLOCK_EXECUTION_INVESTIGATION.md`
- **The scheduler is a 16-slot fixed-stack cooperative design: the slot table lives at 0x8016986C (via `DAT_80165f98`) with 0x400-byte slots, the current slot index is `DAT_80174038` (0x10 wraps to 0), and `FUN_8014ca80` saves s0..s7/k0/k1/gp/sp/s8/ra to slot+0x10..0x44, round-robins to the next live slot (alive flag slot+0x48) running the per-tick engine `FUN_80142ca8` between swaps — every blocking opcode handler yields in a loop via `FUN_8014ca80` until its wait condition clears.** — `[S·D] 2/3`
  - S: `DAT_80165f98` → 0x8016986C, `DAT_80174038`, `FUN_8014ca80`, `FUN_80142ca8` (`project-assets/fft-rom/battle_disassembly.txt`)
  - D: live slot-table dump confirming slot layout, alive/wait flags, and cur_slot (chapel `mid_dialog` savestate, 2026-06-27)
  - src: `research/working_documents/chapel_opcode_trace/BLOCK_EXECUTION_INVESTIGATION.md`
- **The generic block-body interpreter `FUN_8013e904` dispatches opcodes from its slot's bytecode pointer, advancing by `DAT_8014d170[opcode]+1`, and exits on 0x2B by calling `FUN_8014c958`; its opcode handlers yield themselves (e.g. `FUN_8013e7d0` busy-waits until a Unit Anim finishes), so a block ticks at whatever rate its own wait conditions allow.** — `[S] 1/3`
  - S: `FUN_8013e904`–`0x8013edd4` (dispatch loop LAB_8013e95c, epilogue LAB_8013ed80), `FUN_8013e7d0` (`project-assets/fft-rom/battle_disassembly.txt`)
  - src: `research/working_documents/chapel_opcode_trace/BLOCK_EXECUTION_INVESTIGATION.md`
- **Among the main interpreter's spawn-coroutine opcodes, `Block Start` (0x2A) is the only one that runs the generic bytecode interpreter `FUN_8013e904`; the others (0x21 SoundEffect → `FUN_8013c710`, 0x2E Wait → `FUN_8014703c`, 0x3B SpriteMove → `FUN_8014672c`, 0x6E SpriteMove(β) → `FUN_80146ee4`, 0x6F WaitSpriteMove → `FUN_80146f20`) each run a hand-written per-opcode coroutine, which is why in-block opcodes like Wait Walk dispatch to the same handler the main thread would use.** — `[S] 1/3`
  - S: spawn cases from `LAB_80144e70` onward (`project-assets/fft-rom/battle_disassembly.txt`)
  - src: `research/working_documents/chapel_opcode_trace/BLOCK_EXECUTION_INVESTIGATION.md`
- **Live slot observation: slot 0 is the always-on system slot, the scenario VM runs on slot 1 (parked on a Wait with wait flag 0x4c=68 in the dump), slot 2 is a long-lived worker, and dead block slots retain their last bytecode pointer because `FUN_8014c958` clears only the alive (0x48) and wait (0x4c) flags, not slot[0].** — `[S·D] 2/3`
  - S: `FUN_8014c958` (`project-assets/fft-rom/battle_disassembly.txt`)
  - D: live slot-table dump after dialog advance (`/tmp/dump_slot_table.lua`), chapel `mid_dialog` savestate (2026-06-27)
  - src: `research/working_documents/chapel_opcode_trace/BLOCK_EXECUTION_INVESTIGATION.md`
- **The opcode-size table at `DAT_8014d170` gives [0x2A Block Start]=0 and [0x2B Block End]=0 (each advances 1 byte), [0x28 Walk To]=8, [0x29 Wait Walk]=2, [0x2D Rotate Unit]=6, [0x2F Block Loop]=0 — so the main thread's forward scan for 0x2B steps over any nested 0x2A one byte at a time with no depth tracking.** — `[S·D] 2/3`
  - S: `FUN_80149ebc` advances via `DAT_8014d170` (`project-assets/fft-rom/battle_disassembly.txt`)
  - D: live opcode-size table dump (`/tmp/check_opsize.lua`), PCSX-Redux port 8082 (2026-06-27)
  - src: `research/working_documents/chapel_opcode_trace/BLOCK_EXECUTION_INVESTIGATION.md`
- **Godot mirrors the PSX block concurrency: `ScenarioVM`'s `Block Start` spawns a child context that begins in the same vsync as the main thread, and `_tick_once` round-robins every live context each tick until none remain — so scenario 6's three ride-off blocks (chocobo/Ovelia/Delita, chunk instrs 346–376) advance their Sprite Moves concurrently, matching the PSX same-vsync slot allocations.** — `[R] 1/3`
  - R: `godot-learning/src/scenarios/ScenarioVM.gd` `_op_block_start` + the same-tick drain in `_tick_once` (in-code comment cites the `BLOCK_EXECUTION_INVESTIGATION.md` live run at vsync=409); concurrent slide observed via `godot-learning/tools/probe_scenario6_rideoff.gd` (no dedicated test named)
  - src: `research/working_documents/SCENARIO6_RIDE_OFF_CHOCOBO.md`

## Notes

(empty — user territory)

## Related

- [[Event Opcode Catalog]]
- [[Wait Value Opcode]]
- [[Scenario Camera Opcodes]]
