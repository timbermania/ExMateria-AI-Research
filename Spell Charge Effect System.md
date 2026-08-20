# Spell Charge Effect System

FFT renders in-memory charge VFX (no E###.BIN file) for abilities that use a charging animation. When an ability is cast, the effect system looks up the caster's secondary animation ID in two tables — the charge-effect table at 0x801b84ac (does this animation get a charge visual, and of what type) and the effect-function table at 0x801b84dc (which handler renders it) — and allocates one 0x54-byte charge-effect slot from the array at 0x801b8b9c via allocate_sprite_animation_slot (0x801add54). A per-frame loop (FUN_801b47e0) then walks the active slot linked list, dispatching each slot to a render/update handler through the jump table at 0x801b8900 until the handler reports completion, at which point the slot is freed. This note covers the routing tables, slot layout, and dispatch loop; individual particle handlers are documented in [[TRAP Charge Particle System]]. Handler 1 (0x801b0ffc) — despite the Ghidra name render_standard_spell_charge — is the bow arrow parabolic-arc renderer, documented in [[Bow Arrow Arc System]]. Of the 23 handlers in the jump table, func_ids 5, 7, 10, 11, 14, and 16 are unreachable: handler 0 (anim_type 0x00, the default for most poses in DAT_801b84ac) is an active 8-byte no-op, func_ids 7/10/11/14/16 are `jr ra; nop` placeholder stubs, and func_id 5 holds ~396 bytes of likely-dead real code.

## Points

- **The secondary-animation effect table at 0x801b84ac (20 bytes, indexed by Secondary Animation ID 0x00–0x13) maps each charge animation to a charge effect type: only anim 0x01 (spell charging) maps to 0x02 (spell charge lines) and anim 0x02 (summon charging) maps to 0x05 (summon charge orbs); every other ID maps to 0x00 (no visual effect).** — `[S] 1/3`
  - S: DAT_801b84ac table with 16-byte memory snapshot (BATTLE.BIN memory dump, per the doc)
  - R: none — 0x801b84ac / secondary-anim charge table not present in godot-learning (charge VFX routes by ability charging_pose_id instead; probed godot-learning/src, godot-learning/tests)
  - src: `research/working_documents/SPELL_CHARGE_EFFECT_SYSTEM.md`
- **The effect function table at 0x801b84dc holds 4 bytes per Secondary Animation ID; byte 0 is the effect function ID used to dispatch the render/update handler (snapshot: anim 0x00 → 0, anim 0x01 → 1, anim 0x02 → 4) and bytes 1–3 are additional parameters.** — `[S] 1/3`
  - S: DAT_801b84dc table with memory snapshot (BATTLE.BIN memory dump, per the doc)
  - R: none — 0x801b84dc table not present in godot-learning (probed godot-learning/src, godot-learning/tests)
  - src: `research/working_documents/SPELL_CHARGE_EFFECT_SYSTEM.md`
- **Charge effect slots live at 0x801b8b9c as 16 slots of 0x54 bytes, chained as a linked list (next/prev at +0x00/+0x01); key fields: effect function ID at +0x03, secondary animation ID at +0x04, element at +0x06, phase state at +0x08 (0=inactive, 1=initialized, 3=ending), frame counter at +0x0C, weapon ID at +0x24, allocated sprite animation slot pointer at +0x50; phase is read via FUN_801adae0 and set to 3 (ending) by FUN_801adb0c.** — `[S] 1/3`
  - S: 0x801b8b9c slot structure and phase helpers in the doc (BATTLE.BIN disassembly + memory dump)
  - R: none — 0x54-byte slot array not present in godot-learning (probed godot-learning/src, godot-learning/tests)
  - src: `research/working_documents/SPELL_CHARGE_EFFECT_SYSTEM.md`
  - ⚠ SUPERSEDED (2026-08-19) by: handler 18's slot-usage table places element_id at +0x02 and initial_frame at +0x06 (per research/working_documents/handler_18_summon_charge_lines.md, 2026-05-10; equal [S] 1/3 scores, newer doc date)
- **allocate_sprite_animation_slot (0x801add54) initializes a charge effect slot: it takes the current slot index from FUN_801ad828, copies the function ID from 0x801b84dc into slot +0x03, copies position/weapon data from the effect data structure, and sets the phase state at +0x08 to 1 (initialized).** — `[S·R] 2/3`
  - S: FUN_801add54 with pseudo-C decompilation, per the doc
  - R: godot-learning/src/effects/EffectManager.gd (`spawn_charge_vfx` / `_spawn_charge_standard`) mirrors the per-charge allocation lifecycle (one active charge effect per unit, force-killed and replaced on re-spawn, freed on completion); no charge-specific test named (tests/GPUBreakTrapTest.gd validates the shared TrapEffect pipeline only)
  - src: `research/working_documents/SPELL_CHARGE_EFFECT_SYSTEM.md`
- **The per-frame charge effect loop (FUN_801b47e0) walks the active slot linked list from DAT_801b9130, stores the current slot pointer in DAT_801bc098, dispatches to the handler selected by the slot's function ID (+0x03) through the jump table at 0x801b8900, and calls FUN_801ad944 to free the slot when the handler returns 0 (return 1 = still active); the next slot index is read before the handler call so handlers may mutate the list.** — `[S·R] 2/3`
  - S: FUN_801b47e0, jump table base 0x801b8900, DAT_801b9130, DAT_801bc098, per the doc
  - R: godot-learning/src/effects/EffectManager.gd (`_on_charge_vfx_finished` / `stop_charge_vfx`) mirrors the finish-cleanup semantics (animation_finished → free the effect node); no charge-specific test named
  - src: `research/working_documents/SPELL_CHARGE_EFFECT_SYSTEM.md`
- **The charge handler jump table at 0x801b8900 maps dispatch index to handler: 0x00 (0x801b0fec) = no effect, returns 0 immediately; 0x01 (0x801b0ffc) = spell charge lines (rising sparkle lines); 0x02 (0x801b153c) = magic circle / summon preparation; 0x05 (0x801b27dc) = summon charge orbs (orbiting spheres).** — `[S] 1/3 CONTESTED`
  - S: 0x801b8900 jump table, per the doc (BATTLE.BIN disassembly)
  - S: 0x801b0fec is an 8-byte active no-op (`jr ra; _clear v0`, returns 0, removed on the first dispatch tick) reached via anim_type 0x00, the default for all but pose indices 1 and 2 in DAT_801b84ac, per `research/working_documents/stub_handlers_reference.md`
  - R: none — the jump-table index-to-address mapping is not present in godot-learning (its spell charge lines / orbital summon orb effects are implemented as TrapChargeLineEffect / TrapOrbitalEffect under its own handler numbering; probed godot-learning/src, godot-learning/tests)
  - src: `research/working_documents/SPELL_CHARGE_EFFECT_SYSTEM.md`
  - ⚠ SUPERSEDED (2026-08-20) by: summon charge orbs are func_id 22 at 0x801b4234 (g_charge_effect_handlers[22] at 0x801b8958), routed via DAT_801b84dc[0x05] → func_id 22; jump-table index 5 (0x801b27dc) is unreachable dead code, not the orb handler (per research/working_documents/summon_orb_orbital_system.md, 2026-05-10; higher [S·R] 2/3 score, newer doc date vs 2026-04-16)
  - ⚠ SUPERSEDED (2026-08-19) by: handler 1 at 0x801b0ffc is the bow arrow parabolic-arc renderer (bow-only, anim_type 0x01); spell charge effects route via anim_types 0x02/0x04 to handler 4 (charge_effect_orbs), not handler 1, despite the Ghidra name render_standard_spell_charge (per research/working_documents/handler_1_spell_charge_arc_system.md, 2026-05-10; explicit correction in the new doc)
- **On ability cast, effect initialization at 0x801a1694 looks up DAT_801b84ac[secondary_anim_id]; if nonzero it calls allocate_sprite_animation_slot and stores the active slot index in DAT_801bbf64, otherwise it sets DAT_801bbf64 = 0 (no charge effect).** — `[S·R] 2/3`
  - S: 0x801a1694 initialization flow, DAT_801bbf64, per the doc
  - R: godot-learning/src/gpu/CombatLoop.gd + godot-learning/src/effects/EffectManager.gd (`spawn_charge_vfx` on LOGICAL_ACTIVITY_SPELL_CHARGING entry) mirror cast-time charge routing; no charge-specific test named
  - src: `research/working_documents/SPELL_CHARGE_EFFECT_SYSTEM.md`
- **The DAT_801b84dc routing table is 84 bytes (21 entries × 4 bytes) mapping anim_types 0x00–0x14 to func_ids: 0x00→0, 0x01→1, 0x02→4, 0x03→20, 0x04→4, 0x05→22, 0x06→20, 0x07→2, 0x08→2, 0x09→8, 0x0A→18, 0x0B→17, 0x0C→15, 0x0D→13, 0x0E→12, 0x0F→9, 0x10→20, 0x11→6, 0x12→21, 0x13→19, 0x14→3; func_ids 5, 7, 10, 11, 14, and 16 appear in no entry, so those handlers are unreachable.** — `[S] 1/3`
  - S: DAT_801b84dc (0x801b84dc, 21 entries) complete mapping, per `research/working_documents/stub_handlers_reference.md` (BATTLE.BIN disassembly)
  - R: none — DAT_801b84dc anim_type→func_id table not present in godot-learning (charge VFX routes by ability charging_pose_id instead; probed godot-learning/src, godot-learning/tests, smd-player/addons/exmateria_sound, fft-sound-driver — no matches)
  - src: `research/working_documents/stub_handlers_reference.md`
- **Charge handlers 7, 10, 11, 14, and 16 (0x801b2968, 0x801b2bfc, 0x801b2c04, 0x801b2e60, 0x801b3118) are unreachable 8-byte `jr ra; nop` stubs with undefined return values: no anim_type in DAT_801b84dc maps to these func_ids and no other code path indexes g_charge_effect_handlers with these values; each stub sits exactly 8 bytes before its successor handler in the table layout.** — `[S] 1/3`
  - S: 0x801b2968 / 0x801b2bfc / 0x801b2c04 / 0x801b2e60 / 0x801b3118 stub disassembly and 8-byte gap analysis, per `research/working_documents/stub_handlers_reference.md`
  - R: none — stub func_ids 7/10/11/14/16 not present in godot-learning (TrapEffect.gd HANDLER_CONFIGS covers func_ids 2, 3, 4, 6, 8, 9, 12, 13, 15, 17, 19, 21, 22 only; tests/GPUBreakTrapTest.gd validates the shared TrapEffect pipeline only)
  - src: `research/working_documents/stub_handlers_reference.md`
- **Handler 5 (FUN_801b27dc, 0x801b27dc) is also unreachable — no anim_type in DAT_801b84dc routes to func_id 5 — but is not a stub: it contains ~396 bytes of real code, possibly dead code from a removed feature.** — `[S] 1/3 CONTESTED`
  - S: 0x801b27dc (FUN_801b27dc) reachability and ~396-byte size, per `research/working_documents/stub_handlers_reference.md`
  - R: none — func_id 5 / 0x801b27dc not present in godot-learning (no handler-5 entry in TrapEffect.gd HANDLER_CONFIGS)
  - src: `research/working_documents/stub_handlers_reference.md`

## Notes

(empty — user territory)

## Related

- [[Bow Arrow Arc System]]
- [[Spell Charge Lines System]]
- [[Summon Charge Lines System]]
- [[Summon Orb Orbital System]]
- [[TRAP Charge Particle System]]
- [[Animation Event System]]
- [[Effect System Index]]
