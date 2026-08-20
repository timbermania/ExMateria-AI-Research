# Battle Action SFX

How a melee/"regular" attack picks its sound: pinned down 2026-06-11 via PCSX capture (knight basic Attack savestate) plus BATTLE.BIN disassembly. The dispatcher FUN_80082620 derives a sound-class index s2 from the byte at unit+0x1ab (clamped at 0x20); s2 indexes three 32-entry global-SFX-bank tables (cursor, swing, alternate) selected by the 0x18e hit-entry-type switch, with a hardcoded 0x30 and a target-conditioned 0x72. godot-learning reimplements the lookup as AttackSfxResolver (weapon graphic -> sound class -> swing/hit/block slugs, ROM table baked into attack_sounds.json) driven from CombatLoop; smd-player reimplements the playback half (key -> resource -> SMD -> SPU).

## Points

- **The battle action SFX dispatcher FUN_80082620 (BATTLE.BIN) is called per action phase with a0 = unit/action ctx, a1 = phase: it reads sprite_id from u8[unit+0x1ab], derives the sound-class index s2 = u8[FUN_8005a884(sprite_id) + 5], clamps s2 to 1 when s2 >= 0x20, and s2 indexes the battle SFX tables.** — `[S·D·R] 3/3`
  - S: BATTLE.BIN disasm, symbols FUN_80082620 and FUN_8005a884, per `research/effect_sound/working_documents/BATTLE_SFX_ATTACK_SOUNDS.md`
  - D: knight basic Attack PCSX capture (savestate `.pcsx-state-win/SCUS94221.sstate9`, 2026-06-11): sprite_id 0x13 -> s2 3, live play_sound args 0x1A then 0x13
  - R: `godot-learning/src/audio/AttackSfxResolver.gd` (weapon graphic -> sound_class from the ROM table) + `godot-learning/tests/AttackSfxResolverTest.gd` (knife=1, sword=2, rune blade=3, fists=0, gun=5, bow=6)
  - src: `research/effect_sound/working_documents/BATTLE_SFX_ATTACK_SOUNDS.md`
- **Battle SFX are chosen from three 32-entry sound-class tables populated only for s2 0x00..0x12 (rest 0x00): DAT_80093d40 (cursor/select, phase 0), DAT_80093d60 (swing, used by entry-type cases 9/c), DAT_80093d80 (alternate, cases 2/3/a); table values are global SFX bank indices (resource_id 0), not per-item data.** — `[S·D·R] 3/3`
  - S: BATTLE.BIN data at DAT_80093d40 / DAT_80093d60 / DAT_80093d80 (PSX-physical 0x2CD40 / 0x2CD60 / 0x2CD80), per `research/effect_sound/working_documents/BATTLE_SFX_ATTACK_SOUNDS.md`
  - D: knight basic Attack capture (2026-06-11): DAT_80093d40[3] = 0x1A (cursor) and DAT_80093d60[3] = 0x13 (swing) both matched the live play_sound args exactly
  - R: `godot-learning/src/audio/AttackSfxResolver.gd` + `godot-learning/assets/audio/sfx_banks/attack_sounds.json` (bakes weapon graphic -> class -> ROM swing/hit/block ids from the same tables, incl. specials 0x30 Blade Grasp and 0x72) + `godot-learning/tests/AttackSfxResolverTest.gd`
  - src: `research/effect_sound/working_documents/BATTLE_SFX_ATTACK_SOUNDS.md`
- **In phase 1 the attack-swing loop switches on each hit entry's type byte u8[entry+0x18e] (1..0xd): cases 2,3,a play DAT_80093d80[s2]; cases 1,4,7,b,d play hardcoded 0x30; cases 9,c play DAT_80093d60[s2] when u8[target+0x18d] == 0, else the special 0x72; cases 6,8 and every other phase play no sound.** — `[S·D] 2/3`
  - S: 0x18e switch and target+0x18d condition in FUN_80082620, BATTLE.BIN disasm, per `research/effect_sound/working_documents/BATTLE_SFX_ATTACK_SOUNDS.md`
  - D: knight basic Attack capture (2026-06-11) confirms the swing-table case firing (0x13 = DAT_80093d60[3])
  - R: none — 0x18e entry-type switch not present in godot-learning (probed godot-learning/src, godot-learning/tests, smd-player/addons/exmateria_sound; `godot-learning/src/gpu/CombatLoop.gd` approximates it by playing the resolver's swing/hit/block slugs keyed off animation phase instead)
  - src: `research/effect_sound/working_documents/BATTLE_SFX_ATTACK_SOUNDS.md`
- **Battle SFX playback runs play(idx) = FUN_8006b960(ctx, idx) -> FUN_80044018 (wrapper) -> FUN_800125a8 (play_sound) -> SCUS SMD interpreter -> SPU, where idx is the global SFX bank index (resource_id 0) forming the lower 16 bits of the play_sound key.** — `[S·D·R] 3/3`
  - S: BATTLE.BIN disasm symbols FUN_8006b960, FUN_80044018, FUN_800125a8, per `research/effect_sound/working_documents/BATTLE_SFX_ATTACK_SOUNDS.md`
  - D: knight basic Attack capture (2026-06-11) — live play_sound args observed at the chain tail
  - R: `smd-player/addons/exmateria_sound/runtime/effect_sound/play_sound.gd` (reimplements the playback half: resource_id = key >> 16 bank lookup, resource_id << 16 SPU VRAM bank base, resource_id 0 = global SFX bank)
  - src: `research/effect_sound/working_documents/BATTLE_SFX_ATTACK_SOUNDS.md`

## Notes

(empty — user territory)

## Related

- [[SFX Index]]
- [[SPU Voice Engine]]
- [[Effect Sound Timing]]
- [[Projectile Arc System]]
