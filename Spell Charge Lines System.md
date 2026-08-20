# Spell Charge Lines System

Handler 4 (FUN_801b1c04) renders the spell charge line VFX: a 2D screen-space effect in which up to 10 gouraud-shaded LINE_G2 trails (16-slot pool) contract from a radius-256 spawn ring around the caster's head inward to the caster via cosine-eased position history, alongside up to 14 SCATTER-mode sparkle particles (config 12, CLUT 0x7ACF) that implode toward the head. This note holds the disassembly-derived model — state machine, 14-byte parameter table at 0x801b8880, 7-entry position-history ring, easing function FUN_801a8834, element color palette at 0x801b84c0, fade curves, and 1-pixel perpendicular drift — with every data table cross-verified against BATTLE.BIN (charge-effect data section at file 0x151000; file offset = RAM − 0x80067000). Handler 4 is reimplemented in godot-learning as `TrapChargeLineEffect` with matching constants; slot/dispatch routing lives in [[Spell Charge Effect System]] and the summon-side sibling renderer (handler 18) in [[Summon Charge Lines System]].

## Points

- **Handler 4 (FUN_801b1c04) is the spell charge line renderer: spell charging (charging pose 0x01) sets secondary_anim_id 0x01, DAT_801b84ac[0x01] = 0x02 (effect type), and the slot's function ID resolves to 4, so the dispatch loop at 0x801b47e0 calls g_charge_effect_handlers[4] (0x801b8910) = 0x801b1c04 once per game tick; handler 4 reads the 0x54-byte charge slot's +0x04 (secondary_anim_id), +0x06 (element_flags), +0x08 (phase_state), and +0x12 (caster_id).** — `[S·R] 2/3`
  - S: routing chain 0x801b1c04 / DAT_801b84ac / handler pointer 0x801b8910 (BATTLE.BIN disassembly + handler table byte-verified at file 0x151910), per `research/working_documents/spell_charge_lines_system.md` §1/§17
  - R: `godot-learning/src/effects/EffectManager.gd` `spawn_charge_vfx` (charging_pose_id 1 → `_spawn_charge_lines` → `TrapChargeLineEffect`, "PSX TRAP handler 4"); no validating test named
  - src: `research/working_documents/spell_charge_lines_system.md`
- **Handler 4 keeps all state in a private global block based at 0x801bade0: view-space center i16 pair at +0x00/+0x02, 16 line slots × 0x20 bytes at 0x801bbcfc (7 i16 position-history pairs, u16 age, particle ID), double-buffered groups of 96 × 0x14-byte LINE_G2 primitives starting at 0x801badfc with the write group selected by g_primitive_buffer_index at 0x801bc090, scalar state at 0x801bbf0c–0x801bbf38 (line/sparkle counters, spawn-angle accumulator, table-loaded params, g_spawning_enabled), 2 line-particle IDs at 0x801bbefc, and 14 sparkle IDs at 0x801bbefe.** — `[S] 1/3 CONTESTED`
  - S: data block layout per `research/working_documents/spell_charge_lines_system.md` §3 (s5 base loaded at 0x801b1c14–0x801b1c18; BATTLE.BIN disassembly)
  - R: none — the PSX global data block (0x801bade0 / 0x801bbcfc / 0x801badfc) not present in godot-learning (only the 16-slot / 7-history state model is mirrored in `TrapChargeLineEffect.gd`; probed godot-learning/src, godot-learning/tests)
  - src: `research/working_documents/spell_charge_lines_system.md`
- **Handler 4 is a 3-state machine: state 1 (0x801b1c7c–0x801b1ee8, one-shot) loads the parameter table, initializes both LINE_G2 groups via FUN_80023df8 + FUN_80023c68 (semi-transparent), clears all slots and counters, computes the caster head's view-space center (get_camera_position + GTE rotation), writes −(height+8) to DAT_801b87c6 and to config 12's pos_scatter_y (DAT_801b8798), plays sound 0x1F, and moves to state 2; state 2 (0x801b1eec–0x801b2734) runs spawn-lines → spawn-sparkles → update/render each tick; state 3 (set externally when the charge animation ends; routed at 0x801b1c68 to LAB_801b21e8) skips all spawning, renders remaining lines until active_line_count == 0, then releases the pool via FUN_80012990(0x1F) and returns 0 to free the slot.** — `[S·R] 2/3`
  - S: state entries 0x801b1c7c / 0x801b1eec / 0x801b1c68–LAB_801b21e8 (BATTLE.BIN disassembly), per `research/working_documents/spell_charge_lines_system.md` §4
  - R: `godot-learning/src/effects/TrapChargeLineEffect.gd` State enum {INIT, ACTIVE, ENDING, DONE} + `start_fade()` mirror the init→active→ending→free flow; no validating test named
  - src: `research/working_documents/spell_charge_lines_system.md`
- **The charge-line parameter table at 0x801b8880 holds 14-byte entries indexed via g_duration_param_table[secondary_anim_id × 4]: entry 0 (default) = velocity_scale 0, angle_random_range 512, random_angle_offset 128, line_max_lifetime 32, spawn_radius 256, spawn_chance_divisor 2, max_concurrent_lines 10, fade_curve_index 0; entry 1 (0x801b888E) differs only in spawn_chance_divisor = 258; values load into globals at 0x801bbf14–0x801bbf34.** — `[S·R] 2/3`
  - S: param table 0x801b8880 (bytes cross-verified against BATTLE.BIN file 0x151880) and loads 0x801b1cbc–0x801b1d70, per `research/working_documents/spell_charge_lines_system.md` §2
  - R: `godot-learning/src/effects/TrapChargeLineEffect.gd` constants MAX_CONCURRENT_LINES = 10, LINE_MAX_LIFETIME = 32, SPAWN_CHANCE_DIVISOR = 2 mirror entry 0; no validating test named
  - src: `research/working_documents/spell_charge_lines_system.md`
- **Line spawning (state 2, 0x801b1eec–0x801b20f8) is gated on g_spawning_enabled == 0, active count < max_concurrent (default 10), and rand() % divisor == 0 (50% per tick with the default divisor 2); the spawn angle = (rand() & 0x1FF) + accumulator where the accumulator += 0x571 per spawn (≈122.4°, golden-angle-like distribution that prevents clustering); the position lies on the radius-256 circle in 12.12 fixed point in 2D screen space around the head center; the velocity angle = base + rand() % 512 − 128 − 0x800 (the −0x800 turns velocity inward), but velocity_scale defaults to 0 so lines don't physically move — contraction comes entirely from the easing function.** — `[S·R] 2/3`
  - S: spawn gates 0x801b1ef8–0x801b1f54, golden angle 0x801b1f8c–0x801b1f9c, fixed-point spawn 0x801b1fa0–0x801b209c (BATTLE.BIN disassembly), per `research/working_documents/spell_charge_lines_system.md` §5
  - R: `godot-learning/src/effects/TrapChargeLineEffect.gd` `_try_spawn_line` mirrors the free-slot scan, the (randi() & 0x1FF) + accumulator ring spawn (GOLDEN_ANGLE_INCREMENT = 1393), and the 50% gate; no validating test named
  - src: `research/working_documents/spell_charge_lines_system.md`
- **Each line keeps 7 i16 position-history pairs inside its 0x20-byte slot (base 0x801bbcfc); the u16 age doubles as the ring-buffer write index via age % 7 (MIPS multiply-by-reciprocal, magic constant 0x92492493 at 0x801b2354–0x801b239c), one entry is overwritten each tick with the new eased head position, and the trail is drawn as 6 LINE_G2 segments between consecutive history entries.** — `[S·R] 2/3`
  - S: slot layout 0x801bbcfc and ring-index math 0x801b2354–0x801b239c (BATTLE.BIN disassembly), per `research/working_documents/spell_charge_lines_system.md` §6
  - R: `godot-learning/src/effects/TrapChargeLineEffect.gd` HISTORY_SIZE = 7 with write_index = (write_index + 1) % HISTORY_SIZE; no validating test named
  - src: `research/working_documents/spell_charge_lines_system.md`
- **FUN_801a8834 (0x801a8834–0x801a8894) is the contraction easing: scaled angle = (age << 11) / max_lifetime, result = start + (end − start) × (4096 − rcos(scaled_angle)) / 8192 with round-toward-zero on negative deltas — a cosine ease-in-out that yields start at age 0, the midpoint at max_lifetime/2, and end at max_lifetime; handler 4 applies it per axis from each line's spawn point toward the caster's view-space center.** — `[S·R] 2/3`
  - S: FUN_801a8834 (BATTLE.BIN disassembly), per `research/working_documents/spell_charge_lines_system.md` §7
  - R: `godot-learning/src/effects/TrapChargeLineEffect.gd` `_process_tick` uses factor = (1.0 − cos(t·π)) / 2.0 ("cosine ease-in-out contraction"); no validating test named
  - src: `research/working_documents/spell_charge_lines_system.md`
- **LINE_G2 vertex colors = palette × fade / 256: the palette is the 9-entry per-element RGB table at 0x801b84c0 (None 0xA0A0A0, Fire 0xFF5040, Lightning 0x40C0C0, Ice 0xE08830, Wind 0x40FF50, Earth 0xA0A0A0, Water 0xC040C0, Holy 0x4050FF, Dark 0xC0C040) indexed by element bitmask via FUN_801adfac; the fade value comes from the 3-curve × 7-entry table at 0x801b88c0 (default curve 0 = [0, 25, 50, 75, 100, 125, 255] — dim tail → bright head; curve 1 steeper ramp, curve 2 inverted with dim head).** — `[S·R] 2/3`
  - S: palette 0x801b84c0 and fade table 0x801b88c0 (bytes cross-verified against BATTLE.BIN file 0x1514C0 / 0x1518C0), palette loads 0x801b22b0–0x801b22dc, per `research/working_documents/spell_charge_lines_system.md` §9–10
  - R: `godot-learning/src/effects/TrapChargeLineEffect.gd` ELEMENT_COLORS (identical 9 RGB values) + FADE_CURVE = [0, 25, 50, 75, 100, 125, 255]; no validating test named
  - src: `research/working_documents/spell_charge_lines_system.md`
- **Each segment gets a 1-pixel perpendicular thickness offset: dir_index = (ratan2(dy, dx) + 0x900) >> 9 quantizes the segment angle to the 8-entry drift table at 0x801b889c (i16 dx/dy pairs: →(1,0), ↗(1,1), ↑(0,1), ↖(−1,1), ←(−1,0), ↙(−1,−1), ↓(0,−1), ↘(1,−1)); the +0x900 pre-rotation makes the offset perpendicular to the line direction, and degenerate segments (start == end) are skipped before AddPrim (0x801b2600).** — `[S] 1/3`
  - S: drift table 0x801b889c (bytes cross-verified against BATTLE.BIN file 0x15189C) and segment loop 0x801b2440–0x801b26cc (BATTLE.BIN disassembly), per `research/working_documents/spell_charge_lines_system.md` §8.3/§11
  - R: none — the ratan2/drift-table quantization not present in godot-learning (line thickness is achieved with a perpendicular cylinder basis in `TrapChargeLineEffect.gd` `_render_lines` instead; probed godot-learning/src, godot-learning/tests)
  - src: `research/working_documents/spell_charge_lines_system.md`
- **Lines expire when age > line_max_lifetime + 7 (default 39): the +7 lets all 7 history entries cycle through the ring so the full trail has converged to the center before the particle is freed (FUN_801adc24 at 0x801b26ec) and the slot cleared, preventing visual popping; once age passes line_max_lifetime, ring writes snap the entry straight to the center instead of easing.** — `[S·R] 2/3`
  - S: expiration 0x801b26d0–0x801b2714 and center-snap 0x801b23fc–0x801b2430 (BATTLE.BIN disassembly), per `research/working_documents/spell_charge_lines_system.md` §8.4/§13
  - R: `godot-learning/src/effects/TrapChargeLineEffect.gd` EXPIRATION_AGE = LINE_MAX_LIFETIME + HISTORY_SIZE (= 39); no validating test named
  - src: `research/working_documents/spell_charge_lines_system.md`
- **Handler 4 spawns sparkle particles alongside the lines: up to 14 in a dedicated ID array (0x801bbefe), at most 1 per tick (DAT_801b87b6 = 1), each allocated via FUN_801adb3c, initialized from config 12 at 0x801b878C via FUN_801a7f5c, and rendered via FUN_801b0c88(12, 0x7ACF, …) (overbright white CLUT); updates run the standard particle loop (FUN_801aa7dc, freed via FUN_801adc24 on death). Config 12 is SCATTER mode (velocity_mode 0x0010) with speed range radius_min/max = 2112/1872 and fixed 10-tick lifetime, so the sparkles fly inward from their spawn scatter toward the caster — a visible implosion.** — `[S·R] 2/3`
  - S: sparkle spawn 0x801b20fc–0x801b21e4, update loop 0x801b21fc–0x801b22a8, config 12 at 0x801b878C (velocity_mode 0x0010, radius 2112/1872, lifetime 10/10 cross-verified against BATTLE.BIN file 0x15178C), per `research/working_documents/spell_charge_lines_system.md` §12
  - R: `godot-learning/src/effects/TrapChargeLineEffect.gd` `_sparkle_effect` (TrapEffect instance, SPARKLE_EMITTER_INDEX = 12, restarted on completion) co-spawns the sparkles; no validating test named
  - src: `research/working_documents/spell_charge_lines_system.md`
- **The whole effect is anchored to the caster's head, not the feet: state 1 computes head_offset = FUN_8008dc74(caster_id) (sprite height from DAT_8009474b[sprite_type × 4]) + 8 and stores −head_offset at DAT_801b87c6 and in config 12's pos_scatter_y (DAT_801b8798), so the line convergence center sits at head height (GTE-rotated view-space, screen_y = cam_pos[2] − head_offset) and the sparkle spawn cube is shifted up by the unit's full height.** — `[S·R] 2/3`
  - S: height lookup 0x801b1e4c, DAT_801b8798 / DAT_801b87c6 writes 0x801b1e64–0x801b1ec8, GTE rotation 0x801b1e94–0x801b1eac (BATTLE.BIN disassembly), per `research/working_documents/spell_charge_lines_system.md` §4/§12.2
  - R: `godot-learning/src/effects/TrapChargeLineEffect.gd` HEIGHT_OVERSHOOT = 8.0 ("PSX adds 8 units above head") + `_convergence_y` head-anchored convergence; no validating test named
  - src: `research/working_documents/spell_charge_lines_system.md`

## Notes

(empty — user territory)

## Related

- [[Spell Charge Effect System]]
- [[Summon Charge Lines System]]
- [[Summon Orb Orbital System]]
- [[TRAP Charge Particle System]]
- [[Unit Sprite Height Table]]
- [[PSX GPU Primitives]]
- [[Ordering Table & AddPrim]]
- [[Effect System Index]]
