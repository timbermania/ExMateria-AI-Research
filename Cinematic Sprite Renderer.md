# Cinematic Sprite Renderer

The chapel's cinematic unit-sprite renderer: `FUN_80085c0c` (decomp symbol `update_and_animate_unit_wep_eff`), a per-vsync per-roster-slot sibling of the combat renderer `FUN_8017a290` — both read the same unit roster at `0x800B7308 + slot*0x440`, but the combat side is fully dormant in the chapel (0 hits in 30 s) while the cinematic side fires ~227/s. Decoded 2026-06-27 from the disassembly plus live probes: single-arg `a0` = roster-slot pointer, sole caller is the linked-list walker `FUN_8008719c` over `DAT_80098a54`, a 4-branch dispatch on the `+0x0c` anim slot (idle / SEQ / rotate-stepper / EVTCHR re-derive), one-shot EVTCHR pulse semantics (~8 renderer hits = 1 frame), camera-relative pose, and a 40-halfword per-direction table.

## Points

- **The chapel cinematic runs on `FUN_80085c0c` (decomp symbol `update_and_animate_unit_wep_eff`), a per-vsync per-roster-slot sibling of the combat renderer `FUN_8017a290`; both read the same unit roster at `0x800B7308 + slot*0x440`, and in a 30 s chapel capture the combat renderer fires 0 times while the cinematic one fires 6802 (~227/s) and the dispatcher `FUN_80084818` fires 60.** — `[S·D] 2/3`
  - S: `FUN_80085c0c` (`update_and_animate_unit_wep_eff`), `FUN_8017a290`, roster base `0x800B7308` stride `0x440` (`battle_disassembly.txt`)
  - D: probe `probe_layer4_render.lua` 30 s breakpoint hit table, `orbonne_prayer_mid_dialog.sstate` (2026-06-27): FUN_8017a290=0, FUN_8006bbfc=0, FUN_8017fddc=0, FUN_801810a0=3, FUN_80084818=60, FUN_80085c0c=6802
  - src: `research/working_documents/chapel_opcode_trace/SPRITE_PIPELINE_INVESTIGATION.md`
- **`FUN_80085c0c` takes a single caller-passed arg `a0` = roster-slot pointer (live-verified: 8 unique `a0` values across 4 s, stride exactly `0x440`, all aligning to `0x800B7308 + slot*0x440`); `a1`/`a2` are register residue, not caller-passed (`a1` is recomputed inside the function at `0x80085df0` as pose_octant), and `ra = 0x800871c0` on every call.** — `[S·D] 2/3`
  - S: `FUN_80085c0c` call site + `0x80085df0` (`battle_disassembly.txt`)
  - D: probe `probe_cinematic_actor.lua` (2026-06-27): 8 unique `a0` across 4 s, stride exactly `0x440`
  - src: `research/working_documents/chapel_opcode_trace/SPRITE_PIPELINE_INVESTIGATION.md`
- **The shared roster entry (stride `0x440`) maps: `+0x00..03` linked-list next/prev pointer (null at the slot-12 tail), `+0x04` slot index, `+0x05` partner byte, `+0x06..07` sprite-set ID (chapel: Agrias `0x0034` AGURI, Simon `0x0013`, Ovelia `0x000c` HIME), `+0x0C..0D` anim slot, `+0x0E` palette index (`0x16`), `+0x40..45` world XYZ s16 Q1, `+0x6C..6F` tile coords, `+0x70..71` 12-bit facing (`0x0C00` at the rotate-cascade start), `+0x7C/7D/7E` pose bytes (`0x05/0x03/0x00`).** — `[S·D] 2/3`
  - S: roster field offsets decoded at `0x80085c0c..0x80085f80` (`battle_disassembly.txt`)
  - D: live roster dump, chapel slot 1 = Agrias (2026-06-27)
  - src: `research/working_documents/chapel_opcode_trace/SPRITE_PIPELINE_INVESTIGATION.md`
- **`+0x0C` is the shared anim-slot field (there is no separate cinematic slot — the event script writes high-range values into it to flip the renderer into EVTCHR), and `FUN_80085c0c` dispatches on it: `0` → `LAB_80086308` (zero-anim/idle), `3` → `LAB_80085e14` (rotate-stepper reset), `1..11` ≠ 3 (generally 1..0x1F4) → `LAB_80085ec4` (SEQ continuation), `≥ 0x1F5` → `LAB_80086164` (re-derive), with the EVTCHR boundary `slti v0, (anim-1), 0x1f4` at `0x80085f6c..0x80085f70`.** — `[S·D] 2/3`
  - S: `0x80085f6c..0x80085f70` (slti boundary), `LAB_80085ec4`/`LAB_80085e14`/`LAB_80086164`/`LAB_80086308` (`battle_disassembly.txt`)
  - D: live `+0x0c` values across the chapel actors (2026-06-27)
  - src: `research/working_documents/chapel_opcode_trace/SPRITE_PIPELINE_INVESTIGATION.md`
- **`LAB_80086164` is not purely EVTCHR — it is the universal "re-derive `+0x1dc` from current anim and pose, prime `FUN_80084818`, clear `+0x0c`" pathway with an inner SEQ-vs-EVTCHR dispatch at `0x80086180..0x800861b8`, the EVTCHR-specific behaviour being only the bypass of the cardinal table lookup (`+0x1dc = anim-1`); the renderer treats `+0x0c` as a one-shot pulse field with three clear sites: `0x80085ea4` (anim=3 reset), `0x80086238` (re-derive), `0x80086304` (SEQ-flag bypass).** — `[S] 1/3`
  - S: `0x80086180..0x800861b8` (inner dispatch), clear sites `0x80085ea4`/`0x80086238`/`0x80086304` (`battle_disassembly.txt`)
  - src: `research/working_documents/chapel_opcode_trace/SPRITE_PIPELINE_INVESTIGATION.md`
- **EVTCHR anim IDs are written to `+0x0c` as one-shot pulses, not sustained state: the field jumps to the pose ID for ~8 consecutive renderer hits (= 1 frame; 488 Hz call rate / 60 Hz vsync ≈ 8.1 calls/vsync/slot) then resets to 0 — live chapel IDs: slot 1 Agrias `0x025d`, slot 12 Hime `0x025f`, slots 2/3/4 → `0x0262`/`0x0261`/`0x0264`, while overlay slots 5/6 keep `+0x0c = 0x0004` (SEQ) with the EVTCHR ID (`0x0262`/`0x0264`) carried in their `+0x06` sprite-set field instead.** — `[D] 1/3`
  - D: `probe_cinematic_actor_perframe.lua` + `capture_cinematic_perframe.py` (~40 s, 18888 hits / 8 unique actors / 95 state changes, 2026-06-27)
  - src: `research/working_documents/chapel_opcode_trace/SPRITE_PIPELINE_INVESTIGATION.md`
- **`FUN_8008719c` (@ `0x8008719c..0x800871f0`) is the sole caller of `FUN_80085c0c` (single `jal` hit at `0x800871b8`) — a linked-list walker over head `DAT_80098a54` (entries linked by `+0x00`, slots pushed/popped by writers at `0x8007a828`/`0x8008703c`), so the rendered set is the enqueued-active subset and LL insertion order = per-tick render order; per entry it calls `FUN_80085c0c` (a0 only) → `FUN_8007f1d4` → `FUN_800870ac`, the last being a finite-state animation-phase machine on `+0x13e` (`0` no-op, `1`→2, `2` steps `FUN_8017fcc8`/`FUN_80086f2c`, `3` → `FUN_8008d18c`); the walker itself has 7 sibling per-vsync ticker callers (`0x80074bd0`, `0x80076460`, `0x77428`, `0x80078528`, `0x800785d0`, `0x80078fd8`, `0x80088818`).** — `[S] 1/3`
  - S: `0x800871b8` (sole `jal`), `DAT_80098a54` (LL head), `FUN_800870ac` (`battle_disassembly.txt`)
  - src: `research/working_documents/chapel_opcode_trace/SPRITE_PIPELINE_INVESTIGATION.md`
- **`+0x0c = 3` is the rotate stepper's per-tick "advance rotate" signal, not a renderable anim: `FUN_80085c0c` consumes it at `LAB_80085e14` (zeroes pose-state, calls `FUN_80084818` to step the rotate, clears `+0x0c` at `0x80085ea4`) — live Agrias cascade: facing 0xC00→0x400 in −0x100 steps with anim pulsing 3→0 at each step.** — `[S·D] 2/3`
  - S: `LAB_80085e14`, clear site `0x80085ea4` (`battle_disassembly.txt`)
  - D: `probe_cinematic_actor_perframe.lua` per-call capture (2026-06-27): slot 1 facing 0xC00→0x400, anim 3→0 pulse at each step
  - src: `research/working_documents/chapel_opcode_trace/SPRITE_PIPELINE_INVESTIGATION.md`
- **Further roster fields decoded in the cinematic path: `+0x1e2` anim-frame counter (decremented per tick; at 0 it calls `FUN_80084818` to advance the anim), `+0x1f8` SEQ-range bounds struct (`+0x00` table ptr, `+0x04` entry count; checked at `0x80085f80..f94` to reject out-of-range anim indices), `+0x80 & 0x01000000` force-per-tick-decrement gate, `+0x80 & 0x20000000` rearm-allowed gate (cleared at `LAB_80086224` after each rearm), `+0x130`/`+0x131` companion mode/ID (mode `== 2` → `FUN_8007a6e4` resolves the companion's roster slot, armed via `FUN_80084818`, sites `0x80086234..0x80086270`), `+0x6e` cached pose_octant (the pose-change comparator at `LAB_80086308`).** — `[S] 1/3`
  - S: `0x80085f80..f94`, `LAB_80086224`, `0x80086234..0x80086270`, `LAB_80086308` (`battle_disassembly.txt`)
  - src: `research/working_documents/chapel_opcode_trace/SPRITE_PIPELINE_INVESTIGATION.md`
- **The dormant combat sibling resolves frames through a sprite-table cell index: `FUN_801810a0` computes `cell_idx = (halfword[+0x48]>>15)*0x100 + byte[+0x48]*GRID_WIDTH + byte[+0x47]` (+0x47 = column, +0x48 = row, bit 15 = page; the same formula appears at the `battle_decompilation.c` writer site), and `FUN_8017a290` (called from `0x8017aa94`) indexes the 8-byte per-cell table at `0x8018f8cc` (bytes [2]/[3] = SEQ frame_id components) from per-slot base `0x801908cc` at stride `0x1C0` (2×0xE0, two sprite-table entries per unit slot, 21 slots); `DAT_800e4e9c`/`DAT_800e4ea0` hold the per-sprite GRID_WIDTH/GRID_HEIGHT set by `FUN_80183ea0` from SHP-setup bytes [0]/[1], so each cardinal step increments `cell_idx` by GRID_WIDTH and lands on a different frame_id row.** — `[S·D] 2/3`
  - S: `FUN_801810a0`, `0x8017aa94`, `0x8018f8cc`, `0x801908cc`, `DAT_800e4e9c`, `FUN_80183ea0` (`battle_disassembly.txt`); same formula at the `battle_decompilation.c` writer site
  - D: probe `probe_layer4_render.lua` 30 s hit table (2026-06-27): `FUN_801810a0` = 3 hits, all `ra=0x8017f73c` — a one-shot caller, the combat SEQ path fully dormant in the cinematic
  - src: `research/working_documents/chapel_opcode_trace/SPRITE_PIPELINE_INVESTIGATION.md`

## Notes

(empty — user territory)

## Related

- [[Cinematic Atlas Row]]
- [[EVTCHR Script VM]]
- [[Sprite Cardinal Pose Selection]]
- [[Rotate Unit Interpolation]]
- [[Unit Anim Opcode]]
