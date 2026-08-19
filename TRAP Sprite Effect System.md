# TRAP Sprite Effect System

The trigger, allocation, and rendering side of FFT's TRAP hit/impact visual system (dust clouds and white flashes at target positions for abilities without E###.BIN effect files, e.g. Throw Stone 0x094). Static-analysis knowledge from the BATTLE.BIN decompilation: the PostGenericAttack SEQ opcode (0xFFDE) is the primary trigger, gated by an effect-execution guard and an ability-ID filter; the position builder selects an effect variant via byte 7 of each position entry, which the charge slot allocator (FUN_801adfec) maps to animation types 7 / 0x12 / 0x14 in the shared 16-slot pool. QueueThrowAnimation (0xFFD9) is the projectile counterpart, using a different allocator (FUN_801ade7c) and an ability-ID-to-projectile-type table. Sprite data comes from the TRAP1 section of BATTLE/WEP.SPR; animation data is compiled into BATTLE.BIN as frame-definition tables rendered by a three-pass GTE pipeline. Particle-handler internals (handler 2, config table, physics) are documented in [[TRAP Hit Effect Particle System]]; the slot pool and dispatch loop in [[Spell Charge Effect System]].

## Points

- **PostGenericAttack (SEQ 0xFFDE) is the only SEQ opcode that directly spawns TRAP hit clouds: for each target struck it runs the target flinch pose, the caster follow-through, and damage numbers, then spawns one hit cloud via FUN_8006894c; the whole handler is skipped while an effect file is executing (`DAT_800960e4 == 0x34` guard), so effect files' own `[2b]` hit-cloud spawns never double up with it.** — `[S·R] 2/3`
  - S: 0xFFDE handler pseudocode with per-target loop and DAT_800960e4 guard, `research/working_documents/TRAP_SPRITE_EFFECT_SYSTEM.md` §3
  - R: `godot-learning/src/animation/AnimationPlayback.gd` (emits the POST_GENERIC_ATTACK side effect) + `godot-learning/src/effects/EffectManager.gd` (`set_pending_trap` / `fire_pending_trap` / `spawn_trap_effect`) + `godot-learning/src/gpu/CombatLoop.gd` (`_on_unit_type1_side_effect` fires the deferred TRAP and physical reaction), validated by `godot-learning/tests/GPUReactDurationTest.gd` and `godot-learning/tests/GPUBreakTrapTest.gd` (in `tests/run_all_tests.sh`)
  - src: `research/working_documents/TRAP_SPRITE_EFFECT_SYSTEM.md`
- **The PostGenericAttack handler only spawns TRAP clouds when the casting ability ID passes its filter: 0x000 (generic/default attack), 0x08A–0x091 (8 physical skills), 0x0D5–0x0D7 (3 physical skills), 0x16F, and 0x18A–0x19D (20 monster/special abilities); outside these ranges the opcode does nothing even when targets were struck.** — `[S] 1/3`
  - S: ability-ID filter inside the 0xFFDE handler, `research/working_documents/TRAP_SPRITE_EFFECT_SYSTEM.md` §3
  - R: none — the ability-ID filter ranges (0x08A–0x091 / 0x0D5–0x0D7 / 0x18A–0x19D) are not present in godot-learning (probed `godot-learning/src/`, `godot-learning/tests/`; the reimplementation routes TRAP spawns via its own HP-change and deferred-weapon-impact paths instead)
  - src: `research/working_documents/TRAP_SPRITE_EFFECT_SYSTEM.md`
- **The hit-cloud spawner FUN_8006894c builds the per-target position/type array via FUN_800687e0, reads the caster's weapon frame base (`unit+0x13a`), maps it through the weapon-type-to-effect-variant table DAT_80063abe (stride 8) to get the effect variant byte, and hands both to the charge slot allocator FUN_801adfec.** — `[S] 1/3`
  - S: FUN_8006894c and the DAT_80063abe lookup, `research/working_documents/TRAP_SPRITE_EFFECT_SYSTEM.md` §3/§8
  - R: none — the DAT_80063abe weapon effect-variant table is not present in godot-learning (probed `godot-learning/src/`, `godot-learning/tests/`)
  - src: `research/working_documents/TRAP_SPRITE_EFFECT_SYSTEM.md`
- **FUN_800687e0 writes byte 7 of each position entry as the effect-type selector: 6 for a sound-flagged reaction (`target_ctx+0x19c & 0x4`), 1 for a standard hit where the target has targets of its own, 0 when it has none; with no targets at all it falls back to the map position from the global coordinates at DAT_800961b4 / DAT_800961b8 / DAT_800961bc (X / Y / elevation).** — `[S] 1/3`
  - S: FUN_800687e0 byte-7 selection logic, `research/working_documents/TRAP_SPRITE_EFFECT_SYSTEM.md` §3
  - R: none — the byte-7 position-entry selector is not present in godot-learning (probed `godot-learning/src/`, `godot-learning/tests/`)
  - src: `research/working_documents/TRAP_SPRITE_EFFECT_SYSTEM.md`
- **The hit-cloud charge slot allocator FUN_801adfec maps position-entry byte 7 to the slot's animation type: 0/1/3/4 → 7 (physical hit cloud, the common case), 5 → 0x14 (extended variant), 6/7 → 0x12 (sound-flagged); it stores animation_type at DAT_801b8ba0+off, element flags from FUN_801adfac() at +0x06, the effect function ID from DAT_801b84dc[animation_type × 4] at +0x03, resets the frame counter to 0, and sets phase state to active (1).** — `[S] 1/3`
  - S: FUN_801adfec byte-7 switch and slot field stores, `research/working_documents/TRAP_SPRITE_EFFECT_SYSTEM.md` §4
  - R: none — the 0x801adfec hit-cloud allocator is not present in godot-learning (probed `godot-learning/src/`, `godot-learning/tests/`; the reimplementation routes break/sword-skill effects through its own `EffectManager.spawn_trap_effect` handler numbering instead)
  - src: `research/working_documents/TRAP_SPRITE_EFFECT_SYSTEM.md`
- **QueueThrowAnimation (SEQ 0xFFD9) spawns caster-to-target projectiles via FUN_800689a4: ability 0x094 (Throw Stone) → projectile type 6, 0x17E or 0x170–0x189 → type 0x10, default from the weapon-type table DAT_800943c4; it allocates through FUN_801ade7c, which stores start and end 3D positions for flight interpolation, in contrast to the static-target-position hit-cloud allocator FUN_801adfec — both allocate from the same 16-slot charge pool.** — `[S] 1/3`
  - S: FUN_800689a4 type mapping, DAT_800943c4, and FUN_801ade7c vs FUN_801adfec, `research/working_documents/TRAP_SPRITE_EFFECT_SYSTEM.md` §5
  - R: none — the DAT_800943c4 projectile-type table / 0x801ade7c are not present in godot-learning (probed `godot-learning/src/`, `godot-learning/tests/`; the reimplementation emits the QUEUE_THROW_ANIMATION side effect and derives projectile type from its ability database in `ProjectileManager.spawn_from_gpu` instead)
  - src: `research/working_documents/TRAP_SPRITE_EFFECT_SYSTEM.md`
- **BATTLE/WEP.SPR packs three sprite sections sequentially: WEP1 at 0x00000 (0x8200 bytes, 256×256 weapon sprites), EFF1 at 0x8200 (0x8200 bytes, effect sprites), TRAP1 at 0x10400 (0x4A00 bytes, trap sprites); each section starts with a 0x200-byte palette (16 sub-palettes × 16 colors × 2 bytes, 15-bit BGR + STP alpha bit) followed by 4bpp pixel data (two pixels per byte).** — `[S·R] 2/3`
  - S: section table, per-section layout, and ShishiSpriteEditor `AllSprites.cs` (WEP3Sprite class, 144px width, for TRAP1), `research/working_documents/TRAP_SPRITE_EFFECT_SYSTEM.md` §2
  - R: `godot-learning/tools/extract_spr.py` (`WEP_SPRITES`: WEP1 0x0000/256×256, EFF1 0x8200/256×256, TRAP1 0x10400/256×144) + `godot-learning/tools/extract_trap_palettes.py` (TRAP1 palette at section start, 512 bytes); extracted textures feed `godot-learning/src/effects/TrapEffect.gd`, validated by `godot-learning/tests/GPUBreakTrapTest.gd`
  - src: `research/working_documents/TRAP_SPRITE_EFFECT_SYSTEM.md`
- **TRAP animation data is compiled into BATTLE.BIN as frame-definition tables (no TRAP.SEQ/TRAP.SHP files): render_charge_effect resolves PTR_DAT_801b8534[animation_type] to the frame data (vertex data pointer at +0x0C, vertex count at +0x10, total frames at +0x20), transforms vertices through three GTE passes (geometry→world, world→camera, camera→screen), and the frame iterator FUN_801a8f14 returns per-frame vertex indices, UVs, RGB colors, and TPAGE/CLUT; the render-mode parameter selects quad+extended quad, tri, tri variant, or standard quad, with each primitive depth-sorted into the ordering table.** — `[S] 1/3`
  - S: render_charge_effect pipeline, frame-field table, and render-mode table, `research/working_documents/TRAP_SPRITE_EFFECT_SYSTEM.md` §7
  - R: none — the 0x801b8534 frame pointer table / FUN_801a8f14 frame iterator are not present in godot-learning (probed `godot-learning/src/`, `godot-learning/tests/`)
  - src: `research/working_documents/TRAP_SPRITE_EFFECT_SYSTEM.md`
- **Effect files can also spawn TRAP hit clouds through a second code path: the `[2b]` opcode handlers call FUN_80068c18 (state setup + target flinch + damage numbers + FUN_8006894c with param_2=1, versus param_2=0 on the PostGenericAttack path); because PostGenericAttack is guarded off during effect execution, the two paths coexist without double-spawning clouds.** — `[S] 1/3`
  - S: FUN_80068c18 and its wrappers (FUN_80068c80, trigger_effect_sound), `research/working_documents/TRAP_SPRITE_EFFECT_SYSTEM.md` §9
  - R: none — the effect-file [2b] hit-cloud callback is not present in godot-learning (probed `godot-learning/src/`, `godot-learning/tests/`)
  - src: `research/working_documents/TRAP_SPRITE_EFFECT_SYSTEM.md`

## Notes

(empty — user territory)

## Related

- [[TRAP Hit Effect Particle System]]
- [[Spell Charge Effect System]]
- [[TRAP Charge Particle System]]
- [[GTE World-to-Screen Transform]]
- [[Effect System Index]]
