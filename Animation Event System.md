# Animation Event System

FFT connects unit animation state transitions (idle → charging → acting) to per-frame gameplay triggers (particles, animation changes, sounds) through a 40-bit event bitmask per unit. The per-unit animation state struct (0x801908CC + slot × 0x1C0) holds three 5-byte flag fields — permanent (+0x4E), transient (+0x1BB), and derived active (+0x58 = permanent | transient) — and a five-stage pipeline runs the bits from state change to action: FUN_8005e7a8 updates the flags on state change and fires per-bit callbacks, the dispatch (per-change FUN_8018e9bc or bulk FUN_80180fe4/80180f40) calls FUN_80068e80, which routes events 1–32 via a jump table to +0x14C and events 33–40 via the DAT_80094a40 bitmask to +0x148, FUN_80068d08 accumulates them per frame, and FUN_800831b8 checks the flags to trigger actions — the chain ending at handler 15 (FUN_801b2e68), a zero-speed SCATTER drift particle handler whose trigger condition (unit+0x148 & 0x4) appears unreachable per exhaustive static analysis.

## Points

- **The per-unit animation state struct lives at 0x801908CC + unit_slot × 0x1C0 (448 bytes per entry, max 21 units) and holds three 5-byte (40-bit) event flag fields: permanent_flags at +0x4E (loaded from the unit data table), transient_flags at +0x1BB (set/cleared by animation state changes), and active_flags at +0x58, which is always derived byte-wise as permanent | transient and is what event dispatch reads; bit N (0–39) maps to event ID N+1, and within each byte the order is MSB-first (byte 0 = events 1–8, bit 7 = first event in the byte).** — `[S] 1/3`
  - S: animation state array base 0x801908CC (stride 0x1C0), flag offsets +0x4E/+0x1BB/+0x58, per `research/working_documents/animation_event_system.md`
  - src: `research/working_documents/animation_event_system.md`
- **On an animation state change (e.g. idle → charging), FUN_8005e7a8 (0x8005e7a8) looks up the new bit pattern in the jump table PTR_LAB_80059830 indexed by animation_event_type, updates transient_flags via FUN_8005e6cc (0x8005e6cc) — SET/CLEAR/REPLACE on a byte of +0x1BB, then recompute the corresponding +0x58 byte as +0x4E | +0x1BB — XORs old vs new active_flags to detect changed bits, and fires FUN_8018e9bc (0x8018e9bc) for each changed bit with action 1 (SET) for newly set bits and action 0 (CLEAR) for newly cleared ones; FUN_8005e744 (0x8005e744) is the pure recalculation loop (+0x58[i] = +0x4E[i] | +0x1BB[i] for i = 0..4) called from 6 sites.** — `[S] 1/3`
  - S: FUN_8005e7a8 (0x8005e7a8), jump table PTR_LAB_80059830, FUN_8005e6cc (0x8005e6cc), FUN_8005e744 (0x8005e744), FUN_8018e9bc (0x8018e9bc), per `research/working_documents/animation_event_system.md`
  - src: `research/working_documents/animation_event_system.md`
- **Two bulk dispatch paths feed events from the active bitmask: FUN_80180fe4 (0x80180fe4, from the unit setup path 0x80087fa4) iterates all 40 bits of +0x58 MSB-first (byte_index = bit/8, mask = 0x80 >> (bit & 7)) and calls FUN_80068e80(event_id, 1, unit_id) for each set bit, while FUN_80180f40 (0x80180f40, from FUN_8018c288) is the CLEAR mirror that also clears the +0x4E and +0x1BB bytes after dispatch; the per-change path is FUN_8018e9bc (0x8018e9bc), a thin wrapper that checks the guard flag DAT_8018f5fc != 0 before calling FUN_80068e80(event_id, action, unit_id).** — `[S] 1/3`
  - S: FUN_80180fe4 (0x80180fe4), 0x80087fa4, FUN_80180f40 (0x80180f40), FUN_8018c288, FUN_8018e9bc (0x8018e9bc), DAT_8018f5fc, per `research/working_documents/animation_event_system.md`
  - src: `research/working_documents/animation_event_system.md`
- **FUN_80068e80 (0x80068e80) routes each event to per-frame flag words: events 1–32 SET go through a jump table at 0x800670B8 + (event_id − 1) × 4, each entry OR-ing its bit into per_frame_set_lo +0x14C and clearing it from per_frame_clear_lo +0x154 (event 3 additionally calls FUN_80199ec8), and events 33–40 use the bitmask path `bitmask = DAT_80094a40[event_id × 4]` — SET ORs the bitmask into +0x148 and ANDs ~bitmask into +0x150 (LAB_800690a0), CLEAR ORs it into +0x150 and ANDs ~bitmask into +0x148 (LAB_80068f80); the two accumulated words are +0x140 = (+0x140 & ~+0x150) | +0x148 and +0x144 = (+0x144 & ~+0x154) | +0x14C.** — `[S] 1/3`
  - S: FUN_80068e80 (0x80068e80), jump table 0x800670B8, DAT_80094a40, LAB_800690a0, LAB_80068f80, FUN_80199ec8, per `research/working_documents/animation_event_system.md`
  - src: `research/working_documents/animation_event_system.md`
- **The event bitmask table DAT_80094a40 (0x80094a40) has 41 entries (indices 0–40) mapping event IDs to +0x148 bit patterns; the entries for events 33–40 are 0x00008000, 0x00010000, 0x01000000, 0x00000040, 0x00040000, 0x00080000, 0x00000000 (39, unused), 0x04000000 — none has bit 2 (0x4) set, so no event 33–40 can produce the +0x148 & 0x4 condition (value 0x00000004 appears only at indices 3 and 23, both on the events 1–32 jump table path).** — `[S] 1/3`
  - S: DAT_80094a40 (0x80094a40) entries, per `research/working_documents/animation_event_system.md`
  - src: `research/working_documents/animation_event_system.md`
- **Per frame, refresh_unit_tile_state (FUN_80068e30, 35+ callers) calls FUN_80068d08 (0x80068d08), which accumulates the per-frame flag words into the persistent words, runs the call chain through FUN_800831b8 (0x800831b8) that checks the flags and triggers actions, then clears +0x148, +0x14C, +0x150, and +0x154 to zero.** — `[S] 1/3`
  - S: FUN_80068d08 (0x80068d08), FUN_80068e30 (refresh_unit_tile_state), FUN_800831b8 (0x800831b8), per `research/working_documents/animation_event_system.md`
  - src: `research/working_documents/animation_event_system.md`
- **FUN_800831b8 (0x800831b8) checks the per-frame flags and triggers: +0x14C & 0x2 → set_unit_animation_with_flags(3, …) (event 2 SET), +0x154 & 0x2 → FUN_80082eec (event 2 CLEAR), +0x14C & 0x4 → set_unit_animation_with_flags(3, …) (event 3 SET), +0x154 & 0x4 → FUN_80082eec (event 3 CLEAR), +0x14C & 0x9 → FUN_80068a74 (events 1+4 SET), +0x14C & 0x40 → battle-state-dependent set_unit_animation_with_flags (event 7 SET), and +0x148 & 0x4 → the handler 15 trigger block.** — `[S] 1/3`
  - S: FUN_800831b8 (0x800831b8), FUN_80082eec, FUN_80068a74, per `research/working_documents/animation_event_system.md`
  - src: `research/working_documents/animation_event_system.md`
- **The handler 15 trigger block at 0x80083250 fires when unit+0x148 & 0x4 and the unit's sprite type (DAT_80094749[unit+0x06 × 4]) is 2 or 3, calling FUN_80068a94 → FUN_80068a20(unit, 0x0C) → allocate_sprite_animation_slot(0x0C), which resolves DAT_801b84dc[0x0C × 4] to func_id 15 and g_charge_effect_handlers[15] (0x801b893c) = FUN_801b2e68 (0x801b2e68) — the zero-speed SCATTER drift handler running per-frame: state 1 initializes external tracking (DAT_801bc0e0), state 2 spawns 2 particles (config 4 at 0x801b861C, CLUT 0x7ACF), state 3 runs physics until termination; the same +0x148 & 0x4 check appears 3 more times later in FUN_800831b8 for other battle-state branches, triggering set_unit_animation_with_flags(0x1A, …) or set_unit_animation_simple(0x34, …).** — `[S] 1/3`
  - S: trigger block 0x80083250, FUN_80068a94, FUN_80068a20, DAT_801b84dc, g_charge_effect_handlers[15] (0x801b893c), FUN_801b2e68 (0x801b2e68), DAT_80094749, DAT_801bc0e0, config 4 (0x801b861C), per `research/working_documents/animation_event_system.md`
  - src: `research/working_documents/animation_event_system.md`
- **The FUN_8005e7a8 jump table (PTR_LAB_80059830, 9 entries for param_2 = 0–8) maps the animation event type to a 4-bit value replacing the low nibble of transient_flags byte 0 — param_2 0 clears events 5–8 (storing 0xFF to +0x15D), 5/6/7/8 turn on events 5/6/7/8 respectively, and 0xFF is special (reads current bit 0 and shifts it to bit 3) — so this function only ever modifies the low nibble of byte 0 (events 5–8), which all lie in the 1–32 range and therefore always take the jump table path to unit+0x14C, never unit+0x148.** — `[S] 1/3`
  - S: PTR_LAB_80059830 (targets LAB_8005e810/81c/824/82c/838/84c), per `research/working_documents/animation_event_system.md`
  - src: `research/working_documents/animation_event_system.md`
- **Exhaustive static analysis finds no code path that sets bit 2 of unit+0x148 — the DAT_80094a40 entries for events 33–40 (the only writers of +0x148 via FUN_80068e80) all lack 0x4, every sw/sh/sb to offset 0x148 in the disassembly is an initialization (zero), clearing (zero), or the table-entry OR that lacks 0x4, and no indirect store-through-register to 0x148 was found — so handler 15's trigger condition (unit+0x148 & 0x4) appears unreachable, with candidate explanations being dead/vestigial wiring, runtime patching of DAT_80094a40 by disc-loaded data, or a version difference; a breakpoint at 0x80083250 (lw v0, 0x148(s0)) triggering on v0 & 4 != 0 is proposed to confirm.** — `[S] 1/3`
  - S: DAT_80094a40 (0x80094a40), full-disassembly store survey of offset 0x148, trigger load at 0x80083250, per `research/working_documents/animation_event_system.md`
  - src: `research/working_documents/animation_event_system.md`
- **Initial event bitmask population happens in FUN_801488a8 (0x801488a8, called from the unit battle entry 0x80148dcc), which copies from the external unit data table DAT_80173c78 (0x80173c78): bytes unit_id × 5 + i → +0x4E (permanent), unit_id × 5 + i + 0xD2 → +0x1BB (transient), and unit_id × 5 + i + 0x69 → +0x58 (pre-combined), establishing the unit's event flags before any animation transitions occur.** — `[S] 1/3`
  - S: FUN_801488a8 (0x801488a8), 0x80148dcc, DAT_80173c78 (0x80173c78), per `research/working_documents/animation_event_system.md`
  - src: `research/working_documents/animation_event_system.md`
- **During action sequences, FUN_80195f8c (0x80195f8c) temporarily suspends events 1 and 4: the clear step (0x80196108) does anim_state[0x58] &= 0xF6 — clearing bits 0 and 3, also in +0x1BB — and the restore step (0x80196164) puts byte 0 back from the backup at offset +0xED1.** — `[S] 1/3`
  - S: FUN_80195f8c (0x80195f8c), clear 0x80196108, restore 0x80196164, backup offset +0xED1, per `research/working_documents/animation_event_system.md`
  - src: `research/working_documents/animation_event_system.md`

## Notes

(empty — user territory)

## Related

- [[Ability Animation Table]]
- [[Ability Execution State Flow]]
- [[Particle Runtime State]]
- [[Unit Anim Opcode]]
