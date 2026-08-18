# Event Sound OpCodes

Event instructions `{21}` Sound Effect and `{6B}` Background Sound play from FFT's two **global** SFX banks — not per-effect `E###.BIN` audio: the **system** bank (`SOUND/SYSTEM.SED` → `system.feds`, category 0x0000, 167 sounds) for one-shot UI/combat/cinematic cues, and the **env** bank (`SOUND/ENV.SED` → `env.feds`, category 0x0001, 23 of 27 IDs in use) for ambient, mostly-looping weather/environment sound. The `{6B}` semantics are now fully reverse-engineered and live-verified (2026-07-01): it is an asynchronous fire-and-forget opcode that spawns a cooperative task (kind `0x35`) playing an env-bank sound under handle `0x10000 | Sound` and linearly ramping its volume from the byte-1 **start volume** to the target over `Time` frames — the wiki's "Echo" byte is the ramp's *start volume* (the opcode has no echo parameter), and Stacking selects tracked (replaceable) vs. overlay playback; `{6A}` Edit BG Sound re-runs the same ramp on the playing sound. Godot implements both (2026-07-02) with a fading, looping, replaceable bg-sound channel; `{7C}` EndSound remains unimplemented. The meaning of each bank slot is wiki catalog knowledge (not recoverable from the extracted `.feds`/`.json` data) and is kept hand-authored in the project (`sfx_bank_names.json` behind the runtime `SfxCatalog.gd`). The `{21}` handler landed in Godot (2026-06-28) as a fire-and-forget `SfxRouter.play_system_by_id` call, verified across the Orbonne chapel cinematic (7 cues, incl. system slot 72 'Locked Door' at PC 92). The `{60}` Fade Sound semantics were reverse-engineered end-to-end and live-verified on 2026-06-28 (pcsx-agent probes D1–D6 on `orbonne_priest_walk`): a music-only latch (`DAT_80165fc0`) that always ramps `forcePlayedMUS` to silence over `(Time << (Shift+2)) & 0x3FFC` sequencer ticks via the shared `Calc_Mus_VolChange` ramp engine — the decay steps on the music-sequencer tick (~490 Hz at scenario-1 tempo), not the video frame; godot-learning implements it as `ScenarioVM` 0x60 → `MusicPlayer.fade_out` → the smd-player sequencer master fade.

## Points

- **Event instruction `{21}` Sound Effect plays one SFX from the global **system** bank (not per-effect `E###.BIN` audio) and takes its Sound ID as a half-word (16-bit) parameter.** — `[R] 1/3`
  - R: `godot-learning/src/audio/SfxCatalog.gd` + `godot-learning/assets/audio/sfx_banks/sfx_bank_names.json` (runtime catalog of the {21} system-bank slot names; no named test)
  - R: `godot-learning/src/scenarios/ScenarioVM.gd` `_op_sound_effect` + `godot-learning/src/audio/SfxRouter.gd` `play_system_by_id` (system slot N → `_play_slot("system", N)`, emits `cue_requested`; chapel trace run 2026-06-28: 7 `{21}` cues dispatched, 6 one-shot SFX voices allocated, no halt; no named test)
  - src: `research/wiki_articles/event_instructions_sound.md`
  - src: `research/working_documents/chapel_opcode_trace/HANDOFF_sound_effect_opcode.md`
- **The system SFX bank is the ISO-derived blob `SOUND/SYSTEM.SED` → `system.feds` (category 0x0000) with 167 sounds whose slot meanings (UI, combat, cinematic cues) come from the wiki catalog, a subset flagged loop/continuous (e.g. 0x04 Rolling Numbers, 0x0B/0x0C Clockwise/Anti-Clockwise Rotation, 0x1E/0x1F Magic Charging, 0x21 Summoning).** — `[R] 1/3`
  - R: `godot-learning/assets/audio/sfx_banks/index.json` (bank metadata: category 0x0000, source `SOUND/SYSTEM.SED`, 167 sounds) + `sfx_bank_names.json` (167 labeled slots with loop flags)
  - src: `research/wiki_articles/event_instructions_sound.md`
- **The env SFX bank is the ISO-derived blob `SOUND/ENV.SED` → `env.feds` (category 0x0001) with 23 of its 27 IDs in use, carrying ambient/looping weather and environment cues (rain, thunder, wind, night, windmill, ...).** — `[R] 1/3`
  - R: `godot-learning/assets/audio/sfx_banks/index.json` (bank metadata: category 0x0001, source `SOUND/ENV.SED`, 23 of 27) + `sfx_bank_names.json` (27 labeled slots, mostly loop-flagged)
  - src: `research/wiki_articles/event_instructions_sound.md`
- **`{6B}` Background Sound plays an env-bank ambient and — unlike `{21}` — carries shaping parameters: Echo (signed byte, echo strength/duration), Volume (signed byte, relative to the sound's original level), Sound Stacking (byte: `0x00` kills any currently-playing background sounds, `0x01` plays over the ones already playing), Time (unsigned byte, frames to ramp to max volume; 1 frame = 1/60 s, so 60/120/180 = 1/2/3 s).** — `[ ] 0/3`
  - src: `research/wiki_articles/event_instructions_sound.md`
  - ⚠ SUPERSEDED (2026-08-15) by: `{6B}` byte 1 is the ramp's START volume, not "Echo" (the opcode has no echo/reverb parameter), Stacking=0 means tracked-slot replacement + per-handle stop (not a blanket kill of currently-playing background sounds), and Volume is the ramp's target
- **The meaning of each global SFX bank slot is wiki knowledge, NOT recoverable from the extracted `.feds`/`.json` bank data — the project keeps it hand-authored in `sfx_bank_names.json` behind the runtime catalog `SfxCatalog.gd` (`name_for`/`is_loop`), and env slots are currently auditioned one-shot like system sounds; full `{6B}` playback semantics (loop, echo, volume ramp, stacking) are unimplemented.** — `[R] 1/3`
  - R: `godot-learning/src/audio/SfxCatalog.gd` + `godot-learning/assets/audio/sfx_banks/sfx_bank_names.json` (no named test)
  - src: `research/wiki_articles/event_instructions_sound.md`
  - ⚠ SUPERSEDED (2026-08-15) by: Godot implements `{6B}`/`{6A}` with a fading, looping, replaceable bg-sound channel (`ScenarioBgSound.gd` + `SfxRouter.gd`/`EffectSfxEngine.gd`, landed 2026-07-02, live-verified booting scenario 4; byte 1 is the ramp start volume, not "echo")
- **The sound-instruction family has two related global-sound opcodes: `{6A}` (sound; see wiki) and `{7C}` (stop sound).** — `[ ] 0/3`
  - src: `research/wiki_articles/event_instructions_sound.md`
- **Event instruction `{6B}` BGSound is a 6-byte opcode `[6B, Sound, StartVol, Volume, Stacking, Time]` dispatched at `0x8014537c` (BATTLE.BIN), which does NOT play the sound inline: it allocates a free event-task slot (`FUN_80149bec(0x10)`, slots 1..15) and spawns a cooperative task (`event_task_spawn`) whose body is `FUN_801499ac`, so the interpreter continues immediately (no Wait barrier) while the sound fades in over subsequent frames.** — `[S·D·R] 3/3`
  - S: dispatcher arm `0x8014537c`, slot allocator `FUN_80149bec`, task body `FUN_801499ac` (`battle_disassembly.txt`)
  - D: Orbonne battle (scenario 4) live run — exec BP at `0x8014537c` fired with s4=0x6B and the task body ran (2026-07-01)
  - R: `godot-learning/src/scenarios/ScenarioVM.gd` `_op_bg_sound` + `godot-learning/tests/ScenarioBgSoundTest.gd` (35 assertions, in run_all_tests.sh)
  - src: `research/working_documents/BGSOUND_OPCODE_6B_INVESTIGATION.md`
- **`{6B}` plays its ambient from the ENV bank under the SPU handle `0x10000 | Sound` — the `0x10000` high word IS the env-bank category selector (category 0x0001) — whereas `{21}` Sound Effect plays from the SYSTEM bank (category 0x0000) with the raw id as handle.** — `[S·D·R] 3/3`
  - S: handle build `s0 = s2 + 0x10000` at `0x801499e0` in `FUN_801499ac` (`battle_disassembly.txt`)
  - D: Orbonne battle live run — `FUN_80013b20` play trigger observed with `a1=0x10001` (Rain, Sound 1) and `a1=0x10012` (Windmill, Sound 0x12) (2026-07-01)
  - R: `godot-learning/src/audio/SfxRouter.gd` bg channel (env sound N → FEDS pair N-1) + `godot-learning/tests/ScenarioBgSoundTest.gd`
  - src: `research/working_documents/BGSOUND_OPCODE_6B_INVESTIGATION.md`
- **The `{6B}` fade is a linear volume ramp from StartVol to Volume over `Time` frames (one step per `event_fiber_yield`, ~`Time`/60 s — frame-stepped, unlike `{60}` Fade Sound which steps on the music-sequencer tick): intermediates are floored at 1, the final frame sets the exact target, and `Time=0` sets the volume immediately.** — `[S·D·R] 3/3`
  - S: ramp worker `FUN_80149a54`, loop `LAB_80149aac` (`battle_disassembly.txt`)
  - D: Orbonne battle live run — rain fade volume observed stepping 1,2,…,24 then holding (2026-07-01)
  - R: `godot-learning/src/scenarios/ScenarioBgSound.gd` (pure ramp model) + `godot-learning/tests/ScenarioBgSoundTest.gd`
  - src: `research/working_documents/BGSOUND_OPCODE_6B_INVESTIGATION.md`
- **`{6B}` byte 1 (op[1]) is the ramp's START volume, NOT the "Echo" the FFTorama wiki claims — the opcode has no echo/reverb parameter at all: the min-1 guard (a vol of 0 would key the voice off via `FUN_80012b6c`) proves it is a volume, and the live run touched no reverb register (only play + set-volume driver calls).** — `[S·D·R] 3/3`
  - S: min-1 guard at `0x80149a18` in `FUN_801499ac` (`battle_disassembly.txt`); `FUN_80012b6c` vol==0 ⇒ key-off (`scus_decompilation.c`)
  - D: Orbonne battle live run — Start=0 gave initial vol 1 (min-1 guard) ramping to 24, with only play + set-vol calls observed (2026-07-01)
  - R: `godot-learning/src/scenarios/ScenarioBgSound.gd` (initial vol = max(1, Start)) + `godot-learning/tests/ScenarioBgSoundTest.gd`
  - src: `research/working_documents/BGSOUND_OPCODE_6B_INVESTIGATION.md`
- **`{6B}` byte 3 Stacking selects how the sound is registered: `0` = the tracked background sound (driver caches the handle at `DAT_8004599c`; a later tracked `{6B}` replaces it) vs `≠0` = untracked overlay; either way the driver first stops voices already playing the SAME handle (`FUN_800440f4` → `FUN_80012990` — a per-handle stop, not a blanket kill of every ambient voice).** — `[S·D·R] 3/3`
  - S: Stacking branch in `FUN_801499ac`, `FUN_800440f4`/`FUN_80044038` (cache `DAT_8004599c`) (`battle_disassembly.txt` / `scus_decompilation.c`)
  - D: Orbonne battle live run — tracked path (Rain, Stack=0) played via `Hyp_play_sound_variant_b` (a0=0x6004) vs overlay path (Windmill, Stack=1) via `FUN_800125a8` (a0=0x2002) (2026-07-01)
  - R: `godot-learning/src/audio/SfxRouter.gd` tracked-vs-overlay bookkeeping (`_bg_tracked_handle`, `_bg_by_sound`) + `godot-learning/tests/ScenarioBgSoundTest.gd`
  - src: `research/working_documents/BGSOUND_OPCODE_6B_INVESTIGATION.md`
- **`{6A}` Edit BG Sound dispatches (`0x801453a0`) directly to `FUN_80149a54` — the same volume-ramp worker `{6B}` uses — so "Edit BG Sound" re-ramps the volume of an already-playing background sound (Start→Target over Time frames) without re-triggering the voice.** — `[S·R] 2/3`
  - S: dispatcher arm `0x801453a0` → `FUN_80149a54` (`battle_disassembly.txt`)
  - R: `godot-learning/src/scenarios/ScenarioVM.gd` `_op_edit_bg_sound` + `godot-learning/tests/ScenarioBgSoundTest.gd` (positional parse incl. the `{6A}` two-"Unknown" param collision)
  - src: `research/working_documents/BGSOUND_OPCODE_6B_INVESTIGATION.md`
- **The `{6B}` fade task is tagged event-task kind `0x35` (`event_task_set_kind(0x35)` at `0x801499b0`) and runs cooperatively one step per event tick, so a subsequent `{E5}` Wait For Instruction barrier in the same scenario blocks on the still-fading task until `event_fiber_mark_complete` fires after `Time` frames.** — `[S·D·R] 3/3`
  - S: `0x801499b0` in `FUN_801499ac` (`battle_disassembly.txt`)
  - D: Orbonne battle live run — task observed running cooperatively across ~255 frames (2026-07-01)
  - R: `godot-learning/src/scenarios/ScenarioVM.gd` (`TASK_BGSOUND=53` in the `{E5}` liveness registry) + `godot-learning/tests/ScenarioBgSoundTest.gd` (kind-0x35 liveness)
  - src: `research/working_documents/BGSOUND_OPCODE_6B_INVESTIGATION.md`
- **The Orbonne battle (scenario 4) opens with `{6B}` at PC 5 (chunk offset 0x15, `6b 01 00 18 00 ff`): env Sound 1 "Rain 1", tracked (Stacking=0), fading 0 → 24 over 255 frames (~4.25 s), immediately followed by a Windmill Part 1 `{6B}` (Sound 0x12, Stacking=1 overlay, flat volume 127) — the live chunk at `0x8004A6BC` is byte-for-byte identical to godot's `scenario_004_chunk.json`, and `{6B}` was the sole PC-5 blocker keeping the battle from loading.** — `[S·D·R] 3/3`
  - S: `godot-learning/assets/scenarios/chunks/scenario_004_chunk.json` (PC 5 bytes); live chunk base `0x8004A6BC`
  - D: Orbonne battle live run — operands `[01 00 18 00 ff]` confirmed at `0x8004A6D2` and live chunk cmp'd byte-for-byte against the godot chunk (2026-07-01)
  - R: `godot-learning` boots scenario 4 past PC 5 (VM log `[ScenarioVM] BG Sound id=0x01 'Rain 1' start=0 vol=24 stack=0 time=255 handle=1`) + `godot-learning/tests/ScenarioBgSoundTest.gd`
  - src: `research/working_documents/BGSOUND_OPCODE_6B_INVESTIGATION.md`
- **`{21}` Sound Effect is fire-and-forget: the single u16 sound ID is the entire operand (no volume or pan — those are intrinsic to the SED bank entry), it does not consult unit state, and there is no wait barrier, so the event VM keeps walking immediately after the cue.** — `[R] 1/3`
  - R: `godot-learning/src/scenarios/ScenarioVM.gd` `_op_sound_effect` (returns immediately, no barrier; chapel trace run 2026-06-28 walked 2138/2161 instructions past PC 92 without halting; no named test)
  - src: `research/working_documents/chapel_opcode_trace/HANDOFF_sound_effect_opcode.md`
- **The Orbonne chapel scenario (scenario 1) chunk fires `{21}` Sound Effect at bytecode offset PC 92 (chunk offset `0x0211`) with operand `0x48` (72) = system-bank slot 72 "Locked Door"; the chapel trace run exercised 7 `{21}` cues in total.** — `[S·D] 2/3`
  - S: `research/working_documents/chapel_opcode_trace/static_chunk.tsv` (pc 92 / offset `0x0211`: `Sound Effect`, `Sound=72`)
  - D: chapel trace `research/working_documents/chapel_opcode_trace/godot_run.jsonl` — record `"opcode":"Sound Effect","params":{"Sound":72},"pc":92` (2026-06-28)
  - src: `research/working_documents/chapel_opcode_trace/HANDOFF_sound_effect_opcode.md`
- **`{60} FadeSound(ShiftAmount, Time)` always fades the currently-playing forced music (`forcePlayedMUS`) to volume 0 — the target volume is hard-wired to 0 (the low 16 bits of the latched fade word `Time << (16+ShiftAmount)` are always 0), and the opcode has no "fade to X" form; it operates on the main/forced music only, is a no-op when nothing is playing, and does not touch the BGSound/`{6B}` ambient channel.** — `[S·D·R] 3/3`
  - S: handler `0x801453f4` (latch write `DAT_80165fc0 = Time << (16+ShiftAmount)`), flush `0x8014398c..0x801439a4` (`a0 = word & 0xFFFF` is always 0), bridge `SUB_80043be8` @ `0x80043be8` (`forcePlayedMUS` @ `0x8004d960` gate) — `battle_disassembly.txt` / `hacktics_disassembly.txt`
  - D: live run `orbonne_priest_walk`, probes D1/D3/D6 (2026-06-28) — `forcePlayedMUS=0x800387F0`, `shiftTime=0`, music observed silenced
  - R: `godot-learning/src/scenarios/ScenarioVM.gd` `_op_fade_sound` → `ScenarioApply.fade_sound` → `ScenarioWorld.fade_music` → `MusicPlayer.fade_out` (no-op if nothing playing; target always silence) + `godot-learning/tests/ScenarioApplyTest.gd` `_test_fade_sound_fades_music`
  - src: `research/working_documents/FADESOUND_OPCODE_60_INVESTIGATION.md`
- **The `{60}` fade duration is `changeVol = (Time << (ShiftAmount + 2)) & 0x3FFC` music-sequencer ticks — `Time × 4` for the common ShiftAmount=0; a nonzero ShiftAmount multiplies the tick count by `2^n` until significant bits push above the 14-bit `0x3FFC` window and are masked off (the source of the wiki's "x0A–x0F = immediately kill / x0F = much longer" confusion — there is no special-case logic), and `Time = 0` ⇒ `changeVol = 0` ⇒ immediate silence (no gradual ramp).** — `[S·D·R] 3/3`
  - S: flush extraction `0x80143998..0x801439a4` (`battle_disassembly.txt`); `Calc_Mus_VolChange` immediate branch @ `0x80012f08` (`hacktics_disassembly.txt`)
  - D: probe D2, live run `orbonne_priest_walk` (2026-06-28) — injected `0x1e0000` (Time=0x1e, Shift=0) produced `SUB_80043be8(a0=0, a1=120)` = `0x1e << 2`
  - R: `godot-learning/src/scenarios/PsxNum.gd` `fade_ramp_ticks` (`(time << (shift + 2)) & 0x3FFC`) + `ScenarioDecode.fade_sound` + `godot-learning/tests/ScenarioDecodeTest.gd` (Time=4 Shift=0 → 16 ticks; Time=2 Shift=1 → 16 ticks)
  - src: `research/working_documents/FADESOUND_OPCODE_60_INVESTIGATION.md`
- **`DAT_80165fc0` is a per-tick audio-command latch: the event-processor init at `0x80143084` resets it to `-1` every frame, and the per-tick flush at `0x80143878..0x801439a4` (immediately before the camera per-vsync ticker `0x801439c0`) drains every non-`-1` latch — a one-shot producer/consumer-per-frame model (an external poke is wiped before the flush reads it); the whole sound-latch bank (`{21}`=`0x80165fb4`, `{22}`=`0x80165fc8`, `{5e}`=`0x80165fca`, `{60}`=`0x80165fc0`, plus `0x80165fb8`) is reset together at `0x80143028..0x80143084`.** — `[S·D] 2/3`
  - S: init reset `0x80143084`, flush read `0x8014398c` (`battle_disassembly.txt`)
  - D: probe D1, live run `orbonne_priest_walk` (2026-06-28) — external pokes wiped every frame (proving the per-frame reset); an Exec BP at `0x80143988` (one instruction before the flush) writes the latch exactly as the real handler would, and the flush at `0x8014398c` consumes it the same frame
  - R: none — `DAT_80165fc0` not present in godot-learning (probed `godot-learning/src/`, `godot-learning/tests/`)
  - src: `research/working_documents/FADESOUND_OPCODE_60_INVESTIGATION.md`
- **`Calc_Mus_VolChange` @ `0x80012f08` (SCUS) is the shared music volume-ramp engine — called by the `{60}` bridge `SUB_80043be8` and by the SMD sequencer's Expression/Dynamics opcodes (`0x8001223c`/`0x800122cc`): ramp state lives in the MusDataAll struct at `+0x94` (running-volume accumulator, `>>8` = current vol), `+0x98` (per-tick delta, `<<8` fixed point), `+0x9c` (remaining tick count); `changeVol == 0` writes the target volume immediately, otherwise `step = (target − current) / changeVol` per tick.** — `[S·D·R] 3/3`
  - S: `0x80012f08`, callers `0x8001223c`/`0x800122cc` (`hacktics_disassembly.txt`, symbol `Calc_Mus_VolChange`)
  - D: probe D4, live run `orbonne_priest_walk` (2026-06-28) — after `Calc_Mus_VolChange(MusData=0x800387F0, shiftTime=0, changeVol=120)`: `[0x9c]=120`, `[0x98]=0xFEF11200` (negative step, target 0)
  - R: `smd-player/addons/exmateria_sound/runtime/sequencer.gd` master-volume ramp (`_volramp_ticks_left`/`_volramp_step`/`_volramp_target`, the GDScript mirror of the MusData[0x94/0x98/0x9c] triple, driven via `start_master_fade`) + `godot-learning/src/audio/MusicPlayer.gd` `fade_out` (no named smd-player test; godot side validated by `ScenarioApplyTest.gd` `_test_fade_sound_fades_music`)
  - src: `research/working_documents/FADESOUND_OPCODE_60_INVESTIGATION.md`
- **The `{60}` ramp decays on the music-sequencer tick, NOT the 60 Hz video frame: the music-tick applier `0x80014c28` reads `MusData[0x9c]` and, while ≠ 0, calls `FUN_80014f18` (the "SuzukiVolStore" servicer) to add `[0x98]` to `[0x94]` and decrement `[0x9c]` each tick (the same servicer also services the parallel ramp slots `+0x88`/`+0xa0`/`+0xac`); at the scenario-1 tempo the applier fired ~490/s, so `Time=0x1e` (changeVol=120) silenced the music in ~0.3–0.5 s wall-clock — the wiki's "Time≈frames, 30≈0.5 s" is coincidentally in the right ballpark, but the unit is sequencer ticks, not video frames.** — `[S·D·R] 3/3`
  - S: applier `0x80014c28`, servicer `FUN_80014f18` (`hacktics_disassembly.txt`)
  - D: probe D5, live run `orbonne_priest_walk` (2026-06-28) — vol `8323072→2635716→416260→0`, ticks `120→30→0`, applier ~490/s (dormant in `mid_dialog`/`cinematic` savestates: applier 0×, `MusData[0x10] & 0x100` clear)
  - R: `smd-player/addons/exmateria_sound/runtime/sequencer.gd` (the `_volramp_*` block advances per sequencer tick, not per 60 Hz frame; no named smd-player test)
  - src: `research/working_documents/FADESOUND_OPCODE_60_INVESTIGATION.md`
- **The scenario 1 (Orbonne Prayer) chunk fires `{60}` as byte triple `[60 00 1e]` at RAM `0x8004ac6b` (chunk offset `+05AF`): ShiftAmount=0, Time=0x1e(30) ⇒ fade the "Pray" theme to silence over `0x1e << 2 = 120` ticks, followed by `{22} Switch Track` at `+062E` — fade the current song out, then swap.** — `[S·D·R] 3/3`
  - S: live scenario-1 bytecode `research/working_documents/scenario_1_captures/event_script_disasm.txt` (offsets +05AF, +062E)
  - D: scenario 1 live run `orbonne_priest_walk` (2026-06-28) — the live-music instance the D1–D6 probes validated against
  - R: `godot-learning/assets/scenarios/scenario_1_chunk.json` (offset `0x5af`: `60001e`; `0x62e`: `22016000`) + 0x60 handler bound in `godot-learning/src/scenarios/ScenarioVM.gd` (no chunk-specific test named)
  - src: `research/working_documents/FADESOUND_OPCODE_60_INVESTIGATION.md`

## Notes

(empty — user territory)

## Related

- [[Event Opcode Catalog]]
- [[FEDS Sound Definition Format]]
- [[Scenario Table]]
- [[SPU Voice Engine]]
