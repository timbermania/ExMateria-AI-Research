# Savestate Residue Voice

SPU voices that are already KEY-ON when the parity savestate is captured: PCSX-Redux restores the SPU register state on savestate load, so those voices keep playing from sample 0 — before the captured effect's bytecode runs — until the effect's own KOFF clears them, while Godot starts every voice silent. On `cure_4_no_music` voice 18 is that residue voice: noise mode from the pre-savestate spell (almost certainly armed by `0xB4 Noise(0x3F)`), on from capture sample 0 with `adsr1=13055`/`adsr2=24518`/`raw_pitch=4078`/`start_addr=4112` (`curr_addr` already 16 sample-words advanced), residue KOFF at sample 68977 (cadence 238), cure_4 re-KEYON at 70470. smd-player now replays the residue: the `spu_initial_state.json` `residue_snapshot` plus PCSX's sample-0 anchor rows from `spu_voice_events.jsonl` are seeded into the mixer via `mixer.seed_voice_residue` before `play_feds_pair` — state-preserving (env_state/env_vol kept verbatim) and gated on PCSX's sample-0 `on` — so the residue voice's KOFF/KON land at the same absolute samples as on PCSX and the noise LFSR state at each KEYON aligns. On `haste_no_music` the same seed path (commit f62f2d55) seeds the pre-fire-roll re-keyed voice 20 from PCSX's sample-0 SPU state (voice 21 correctly skipped — still off at sample 0), closing ~62% of the voice-21 cos_dist gap (0.557 → 0.214/0.094); the residual is a ~48-cadence KEYON lag from the timeline cursor origin (see [[Effect Sound Timing]]).

## Points

- **On PCSX `cure_4_no_music`, voice 18 is a savestate-residue voice — already `on=true` at capture sample 0 (the savestate-load moment) with `noise=true`, `adsr1=13055`, `adsr2=24518`, `raw_pitch=4078`, `start_addr=4112`, `loop_addr=48`, `curr_addr=4128` (16 sample-words past start_addr, i.e. mid-playback), `env_state=0`/`env_vol=0` (freshly keyed on, attack still climbing), `vol_l=vol_r=1757`, its noise mode set by the spell that ran before the savestate (almost certainly `0xB4 Noise(0x3F)` at PC `0x80015F44`) — the residue plays on until the cure_4 KOFF at sample 68977 (cadence 238), cure_4's bytecode re-KEYONs it at 70470, and Godot's voice 18 stayed silent until its first KEYON at sample 21462 because Godot's SPU starts every voice off and had no residue model.** — `[S·D·R] 3/3`
  - S: `0x80015F44` (`smd_noise`, opcode 0xB4 — the likely arming of v18's pre-savestate noise mode; §7.1 verified-PC table), `scus_disassembly`
  - D: `spu_voice_events.jsonl` sample-0 rows (`on=true` + adsr/addr/env fields) + `spu_initial_state.json` voice-18 entry (`noise=true`), `cure_4_no_music` capture (2026-05-19)
  - R: `smd-player/workspace/harness/render_effect_sound.gd` (`spu_initial_state.json` `residue_snapshot` loop → `mixer.seed_voice_residue`, gated on PCSX's sample-0 `on` from `spu_voice_events.jsonl`; native binding `smd-player/src/native/fft_spu_mixer_native.cpp:27`) — validated by the `run_effect_iteration.py --session cure_4_no_music` audio-score gates (voice_18.cos_dist 0.337 → target < 0.020, doc §4.2)
  - src: `research/effect_sound/working_documents/CURE_4_V18_SAVESTATE_RESIDUE_AND_NOISE_INIT_FIX.md`
- **The seed fix (commit f62f2d55) extends the residue replay to `haste_no_music`: it seeds the voice that PCSX's pre-fire roll had re-keyed — voice 20, keyed at sample 0 with env_state=0 / env_vol=448 — from PCSX's sample-0 SPU state, correctly skips voice 21 (still off at PCSX sample 0, not yet re-keyed), and thereby closes ~62% of the voice-21 cos_dist gap (0.557 → 0.214; 0.094 on another run), leaving the residual as the ~48-cadence KEYON lag.** — `[D·R] 2/3`
  - D: `haste_no_music` post-seed-fix runs, samples_per_sub=292 (2026-05-24) — voice 21 cos_dist 0.557 → 0.214 (another run 0.094); voice 20 first `on=True` at sample 44 (seeded) vs voice 21 first `on=True` at sample 16444
  - R: `smd-player/workspace/harness/render_effect_sound.gd` (residue-snapshot loop → `mixer.seed_voice_residue`, state-preserving, gated on PCSX sample-0 `on`; commit f62f2d55) + `smd-player/addons/exmateria_sound/runtime/effect_sound_controller.gd` (`stc.start(1, frame_anchor, vm_snapshot)` 3-arg start) — validated by the `haste_no_music` audio-score runs (`smd-player/workspace/orchestrator/run_effect_iteration.py --session haste_no_music`)
  - src: `research/effect_sound/working_documents/HASTE_VOICE_21_KEYON_LAG_INVESTIGATION.md`

## Notes

(empty — user territory)

## Related

- [[SPU Voice Engine]]
- [[Cure 4 Audio Parity]]
- [[Effect Sound Audio Divergence]]
- [[KON KOFF Mask Dispatch]]
- [[Effect Sound Timing]]
- [[SFX Index]]
