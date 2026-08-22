# Battle Action SFX

How a melee/"regular" attack picks its sound: pinned down 2026-06-11 via PCSX capture (knight basic Attack savestate) plus BATTLE.BIN disassembly. The dispatcher FUN_80082620 derives a sound-class index s2 from the byte at unit+0x1ab (clamped at 0x20); s2 indexes three 32-entry global-SFX-bank tables (cursor, swing, alternate) selected by the 0x18e hit-entry-type switch, with a hardcoded 0x30 and a target-conditioned 0x72. godot-learning reimplements the lookup as AttackSfxResolver (weapon graphic -> sound class -> swing/hit/block slugs, ROM table baked into attack_sounds.json) driven from CombatLoop; smd-player reimplements the playback half (key -> resource -> SMD -> SPU). BATTLE.BIN SFX is not attack-only: the 2026-05-18 death V21 investigation showed the per-frame battle-event-object tick `FUN_8006b960` fires a global-bank status-effect SFX (sound_id 0x45) unconditionally at `0x8006B97C`, a trigger outside the effect renderer's parity envelope (now covered by smd-player catalog replay and godot-learning's SfxRouter `combat.unit_died` cue), and exposed a Ghidra export artifact at that PC (disassembly text `lw v0, 0x20(s0)` vs live RAM `jal 0x80044018`). The 2026-05-19 zombie resolution extends the death precedent: zombie's status-effect overlay SFX are per-target BATTLE.BIN dispatches from `FUN_8006FA18` (JALs `0x8006F274`/`0x8006F464` → `Hyp_play_sound_thin_wrapper_44018`, sound_id 0xc/0xd) that are gameplay-incidental rather than spell audio, and are resolved by NOPing those JAL sites on PCSX via the same `BATTLE_BIN_SFX_NOPS` table (full_mix.cos_dist 0.0791 → 0.0033) instead of replaying them on Godot — the canonical pattern for any spell whose savestate triggers secondary BATTLE.BIN-driven SFX.

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
- **Death's (E030, effect_id 30) 3rd `play_sound` fires from BATTLE.BIN at PC `0x8006B97C` inside `FUN_8006b960` — the per-frame battle-event-object tick that unconditionally JALs the thin play_sound wrapper `0x80044018` with sound_id `0x45` — NOT from death.bin's effect-script bytecode: the raw small-int `a0` (vs the `0xC0xxxx` high-byte-routing shape of the two timeline-driven calls at cad 184) marks it as a direct global-bank sound_id request, structurally outside the effect-only renderer's parity envelope.** — `[S·D·R] 3/3`
  - S: BATTLE.BIN `FUN_8006b960` (per-frame battle-event-object tick: sub-tick `FUN_80069dfc`, position updates at s0+0x18/0x20/0x40/0x44, then indirect dispatch through the function-pointer table at `0x80067188` on the state byte at s0+0x7f) and the live RAM instruction word `0x0C011006` (`jal 0x80044018`) at `0x8006B97C`, per `research/effect_sound/working_documents/DEATH_V21_MISSING_THIRD_PLAY_SOUND_DEFICIT.md` §6.3
  - D: `probe_play_sound_call_stack`, death_no_music capture (2026-05-18): thin_44018 row at cad 254 with jal -> 0x80044018, ra=0x8006b984, a0=0x45, anchored=true (calls 1–2 at cad 184 carry a0 0xC00001/0xC00002, anchored=false)
  - R: `smd-player/addons/exmateria_sound/runtime/effect_sound/play_sound.gd` (`battle_sfx_replay` catalog-replay path, active when `feds.resource_id == 0` and `(sound_id >> 16) & 0xFFFF == 0`) + `godot-learning/src/audio/SfxRouter.gd` (`combat.unit_died` cue resolved per BaseStatType into the global system bank) + `godot-learning/tests/SfxRouterTest.gd` (MALE/FEMALE/MONSTER slot resolution + cue observability)
  - src: `research/effect_sound/working_documents/DEATH_V21_MISSING_THIRD_PLAY_SOUND_DEFICIT.md`
- **Ghidra's export of BATTLE.BIN mis-decodes `0x8006B97C` as `lw v0, 0x20(s0)` while live PSX RAM holds instruction word `0x0C011006` (`jal 0x80044018`) — for that overlay-aware region the probe-captured `jal_inst` is authoritative over the disassembly text, which Ghidra's current export did not reanalyze.** — `[S·D] 2/3`
  - S: `0x8006B97C` — BATTLE.BIN disassembly (`scus_disassembly.txt`) vs live RAM instruction word, per `research/effect_sound/working_documents/DEATH_V21_MISSING_THIRD_PLAY_SOUND_DEFICIT.md` §6.3
  - D: `probe_play_sound_call_stack` (death_no_music, 2026-05-18) captured `jal_inst` 0x0C011006 at the thin_44018 site
  - R: none — capture-side disassembly artifact; no Ghidra/RAM-mismatch handling present in godot-learning, smd-player, or fft-sound-driver
  - src: `research/effect_sound/working_documents/DEATH_V21_MISSING_THIRD_PLAY_SOUND_DEFICIT.md`
- **Zombie's status-effect overlay SFX are dispatched per zombie'd target unit by BATTLE.BIN's per-target status-effect handler `FUN_8006FA18`, whose two JAL sites — `0x8006F274` (sound_id 0xc) and `0x8006F464` (sound_id 0xd) — JAL the thin play_sound trampoline `Hyp_play_sound_thin_wrapper_44018` at `0x80044018`, firing per target after the spell's foreground animation completes (PCSX cad ~1211/~1342); these status-overlap SFX are incidental gameplay feedback, not the spell's intended audio (the intended audio is the timeline-driven cad 1–773 chunk).** — `[S·D·R] 3/3`
  - S: BATTLE.BIN disassembly — `FUN_8006FA18` (per-target status-effect handler containing both JALs), JAL sites 0x8006F274/0x8006F464, trampoline 0x80044018 — per `research/effect_sound/working_documents/ZOMBIE_CATALOG_CADENCE_ANCHOR_DELTA_FIX_PLAN.md` §3/§7.4
  - D: `probe_play_sound_call_stack` capture (zombie_no_music, pre-suppression, 2026-05-19): thin_44018 rows at cad 1211/1342 with jal_pc 0x8006F274 (a0=0xc) and 0x8006F464 (a0=0xd)
  - R: `smd-player/workspace/orchestrator/effect_capture_orchestrator.lua` (BATTLE_BIN_SFX_NOPS `["zombie_no_music"] = { 0x8006F274, 0x8006F464 }`) + `smd-player/addons/exmateria_sound/runtime/effect_sound/play_sound.gd` (the BATTLE.BIN-driven catalog-replay path mirroring FUN_8006FA18's fresh-entity allocation per target — dormant for parity scoring) — validated by the post-suppression zombie parity run (full_mix cos_dist 0.0033)
  - src: `research/effect_sound/working_documents/ZOMBIE_CATALOG_CADENCE_ANCHOR_DELTA_FIX_PLAN.md`
- **Zombie's BATTLE.BIN status-effect SFX are suppressed death-style rather than replayed on Godot: writing four zero bytes (MIPS NOP) at 0x8006F274/0x8006F464 before probe arming and regenerating the catalog (3 TIMELINE rows at cads 152/550/550, 0 BATTLE_BIN rows; Godot's catalog-replay filter replays only BATTLE_BIN-region rows) drops `full_mix.cos_dist` from 0.0791 to 0.0033 (24×) and `pair_rate` from 0.2632 to 0.3846, putting zombie in the same parity range as cure/ice/death (all four sessions full_mix.cos_dist < 0.01).** — `[S·D·R] 3/3`
  - S: NOP-target JAL PCs 0x8006F274/0x8006F464 and the death precedent 0x8006B97C — BATTLE.BIN disassembly, per `research/effect_sound/working_documents/ZOMBIE_CATALOG_CADENCE_ANCHOR_DELTA_FIX_PLAN.md` §4/§7.4
  - D: pre/post-suppression captures (zombie_no_music, 2026-05-19): pre pair_rate 0.2632 (10/38), full_mix.cos_dist 0.0791; post `probe_play_sound_call` pcsx=3 at cads {128, 527, 527}, `audio_score.json` full_mix.cos_dist 0.0033
  - R: `smd-player/workspace/orchestrator/effect_capture_orchestrator.lua` (BATTLE_BIN_SFX_NOPS table + NOP-write loop; `--no-suppress-battle-bin-sfx` override in `smd-player/workspace/orchestrator/run_effect_iteration.py`) + `smd-player/workspace/harness/render_effect_sound.gd` (catalog-replay region filter skips non-BATTLE_BIN rows) — validated by the post-suppression parity run (full_mix 0.0033)
  - src: `research/effect_sound/working_documents/ZOMBIE_CATALOG_CADENCE_ANCHOR_DELTA_FIX_PLAN.md`

## Notes

(empty — user territory)

## Related

- [[SFX Index]]
- [[SPU Voice Engine]]
- [[Effect Sound Timing]]
- [[Effect Sound Audio Divergence]]
- [[Projectile Arc System]]
