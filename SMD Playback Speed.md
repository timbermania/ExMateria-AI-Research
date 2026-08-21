# SMD Playback Speed

The FFT effect cadence runs at exactly 240 Hz on both sides — PCSX-Redux (confirmed via the SPU RCnt2 IRQ's `cpu_cycle` delta) and Godot (30 Hz audition tick × 8 dispatches per tick) — but the two sides advance different SPU-sample counts per cadence: PCSX captures produce ~252 samples/cadence because the SPU thread drains its audio output buffer at 44100 samples per realtime second, untied to the CPU emulation pace (60480 samples per emulated second under Lua-probe instrumentation at ~0.73× realtime), while Godot's effect-sound render advances 183 samples per sub-tick (`44100/30/8`, truncated from 183.75; 43920 effective samples/sec). The 2026-05-16 envelope-tail re-investigation of `protect_no_music` (audible voices 20+21) localized the entire 1.343× wallclock tail-gap (PCSX `spu_mix.wav` 3.190 s vs Godot 2.376 s) to this samples-per-cadence mismatch: the per-sample ADSR decay rate is bit-exact on both sides, and in cadence-time Godot's envelope decays *behind* PCSX, not ahead. The harness now accepts an orchestrator-supplied `samples_per_sub` override (default remains the real-PSX 183) so Godot's WAV aligns temporally with whatever pace PCSX captured at.

## Points

- **The `protect_no_music` wallclock tail-gap (PCSX `spu_mix.wav` 3.190 s vs Godot 2.376 s, ratio 1.343×; voice-21 stem 3.079 s vs 2.304 s) is a samples-per-cadence mismatch, not an SPU envelope bug: PCSX produces ~252 SPU samples per FFT cadence, Godot produces 183 (`SAMPLE_RATE / TICK_HZ / FLUSH_PER_DISPATCH` = 44100/30/8, truncated from 183.75), and the per-sample ADSR decay rate is bit-exact on both sides (voice 21: −2.5000 vs −2.4998/sample; voice 20: −1.2500 vs −1.2503/sample) — the two WAVs play at different effective rates (60480 vs 43920 samples/sec), drift out of phase within the first cadence, and bias every full_mix RMS/cos_dist comparison regardless of cadence-level bytecode parity.** — `[D·R] 2/3`
  - D: `probe_cadence_wallclock` cad 50→500 (PCSX 252.00 vs Godot 183.00 samples/cadence), `probe_envelope_tail` per-sample rate fits, `spu_mix.wav` + voice-21 stem duration pairs (2026-05-16)
  - R: `smd-player/workspace/harness/render_effect_sound.gd` (`samples_per_sub = samples_per_tick / FLUSH_PER_DISPATCH`; `--samples-per-sub=` / `cfg.samples_per_sub` override — the orchestrator measures PCSX's samples-per-cadence from `probe_cadence_wallclock.jsonl` and passes it in, implementing the doc §11e option 2; default remains 183)
  - src: `research/effect_sound/working_documents/ENVELOPE_TAIL_INVESTIGATION.md`
- **Both PCSX and Godot run the FFT effect cadence at exactly 240 Hz: PCSX confirmed by the `cpu_cycle` delta over 450 cadences (63,505,393 cycles ÷ 33.8688 MHz = 1.875 s = 450/240), Godot by its hardcoded sub-tick rate (30 Hz audition tick × 8 dispatches per tick).** — `[S·D·R] 3/3`
  - S: PC `0x80017A6C` (RCnt2 init write, target=17640, mode sysclk/8; 4,233,600 ÷ 17640 = 240.00 Hz) + `FUN_80022114` (FFT's SetRCnt wrapper at PC 0x80022114) — disassembly PCs cited in doc §3.1/§10.5
  - D: `probe_cadence_wallclock` `cpu_cycle` delta cad 50→500 (2026-05-16)
  - D: `diag_cadence_wallclock` BP @ `FUN_800149DC` (RCnt2 IRQ handler), `protect_no_music` Run-5 1680-pulse capture — `cpu_cycle` slope 141,125.26 cycles/cadence = 239.99 Hz vs expected 141,120 = 33,868,800 ÷ 240 (0.004% off); Godot `godot_sample_count` slope 183.00/cadence = 240.98 Hz; ratio 1.0000 (2026-05-15)
  - R: `smd-player/workspace/harness/render_effect_sound.gd` (`TICK_HZ := 30`, `FLUSH_PER_DISPATCH := 8` — code comment: matches FFT's 240 Hz per-IRQ cadence)
  - src: `research/effect_sound/working_documents/ENVELOPE_TAIL_INVESTIGATION.md`
  - src: `research/effect_sound/working_documents/PROTECT_NO_MUSIC_AUDIBLE_GAP_RESOLUTION.md`
- **FFT's RCnt2 (Clock 2) timer is configured exactly once, at the boot callsite PC `0x80017A6C` (target 17640, mode sysclk/8 = 240 Hz), and never reconfigured during gameplay: `diag_rcnt_config` logged zero calls to the SetRCnt wrapper `FUN_80022114` across the 1680-pulse `protect_no_music` window, so the timer was set pre-savestate and the 240 Hz cadence is the engine-init default for the whole run — on this effect the `chan+0x78` tempo-state writers exist and operate as documented, but the `chan+0x7C`/`chan+0x8A` writers never fire (no 0xA0/0xA1 opcodes in the bytecode, no LFO-mode-2 activity observed in the run).** — `[S·D] 2/3`
  - S: PC `0x80017A6C` (RCnt2 init write), `FUN_80022114` (SetRCnt wrapper at PC 0x80022114) — disassembly PCs cited in doc §3.1/§7
  - D: `diag_rcnt_config` Exec-BP on `FUN_80022114` logging every call's `(counter_idx, target, mode, callsite_pc)`, `protect_no_music` capture (2026-05-15): zero calls in the capture window
  - R: none — RCnt2 reconfiguration not present in godot-learning or smd-player (probed both for `17640`/`RCnt`; Godot pins the 240 Hz cadence by construction instead)
  - src: `research/effect_sound/working_documents/PROTECT_NO_MUSIC_AUDIBLE_GAP_RESOLUTION.md`
- **The `protect_no_music` audible KEYON sequence is bit-exact between PCSX-Redux and Godot: voice 20 fires 8 KEYONs at PCSX cadences 1/21/41/61/82/242/302/423 (Godot 0/20/40/60/81/241/301/422) and voice 21 fires 4 at 1/242/302/415 (Godot 0/241/301/414) — every event within 1 cadence of the other side, the off-by-1 being the known PCSX `FIRST_OPCODE_FIRED`-gate vs Godot `_cadence_anchored`-latch anchor-zero drift — and the surrounding dispatch checkpoints all match too (1680 RCnt2 pulses, 204 cadence_body fires, 103 opcode dispatches, 1632 per-channel tick entries on both sides), so there is no FFT-side divergence to fix.** — `[S·D·R] 3/3`
  - S: `FUN_8001ACF0` (KEYON-mask writer) with the per-voice ledger BP at PC `0x8001ACF4`, one instruction past entry with `a0`/`a1` (counter mode + voice mask) and `ra` (caller) still live — disassembly cited in doc §3.6
  - D: `diag_keyon_per_voice` per-voice KEYON ledger, `protect_no_music` Run-5 1680-pulse capture (2026-05-15) — 8/8 and 4/4 KEYON counts at matching cadence_indices; the earlier "Godot fires 8 vs PCSX 5 on v20" reading was the rising-edge artifact of `spu_voice_events.jsonl` (a re-keyed voice keeps its `on` bit high), see [[PCSX-Redux Capture Rig]]
  - R: `smd-player/addons/exmateria_sound/runtime/shared/flush_tick.gd::_commit_kon` (per-committed-KEYON `keyon_per_voice` emit, one row per committed KEYON inside the gated commit loop; the doc cites the pre-move `smd-player/scripts/effect_sound/` path) — validated by the `probe_keyon_per_voice` layer-5 pair (`bp_addr 0x8001ACF4`, `smd-player/workspace/orchestrator/probe_validation_manifest.py` + `diff_keyon_per_voice.py`)
  - src: `research/effect_sound/working_documents/PROTECT_NO_MUSIC_AUDIBLE_GAP_RESOLUTION.md`
- **PCSX-Redux's SPU thread samples untied from CPU emulation pace — `MainThread` drains the audio output buffer at 44100 samples per realtime second — so under Lua-probe instrumentation (CPU at ~0.73× realtime) a capture yields 60480 SPU samples per emulated second (252/cadence at 240 Hz) instead of the canonical 44100; the ~252 samples/cadence is host-pace-dependent, not a hardware constant, which is why the Godot-alignment target is measured per capture rather than hardcoded.** — `[D] 1/3`
  - D: `probe_cadence_wallclock` cad 50→500: 113,400 SPU samples in 1.875 s of emulator time (2026-05-16)
  - D: `cadence_calibration.json` `samples_per_cadence` 190–320 range observed across effect sessions, vs canonical 183 (2026-05-24)
  - D: `protect_no_music` Run 1 vs Run 2 (same deterministic input, 2026-05-15): `spu_sample` slope 234.508 → 254.240 samples/cadence (+9% between consecutive runs — the 80+ probe BPs slow the emulated CPU while the SPU thread keeps producing at wall-clock rate) while `cpu_cycle` holds 141,125 cycles/cadence; deterministic timing must use `cpu_cycle` (LuaJIT FFI `uint64_t` cdata needs `tonumber()` to serialize)
  - R: none — PCSX-Redux SPU-thread pacing (`vendor/pcsx-redux/src/spu/spu.cc::MainThread`) is oracle-side code; not present in godot-learning, smd-player, fft-sound-driver, or effect-editor
  - src: `research/effect_sound/working_documents/ENVELOPE_TAIL_INVESTIGATION.md`
  - src: `research/effect_sound/working_documents/ENV_VOL_WITHIN_IRQ_KEY_ON_TIMING.md`
  - src: `research/effect_sound/working_documents/PROTECT_NO_MUSIC_AUDIBLE_GAP_RESOLUTION.md`
- **In cadence-time Godot's `protect_no_music` voice-21 envelope decays SLOWER than PCSX's — more cadences to reach each env_vol threshold (env_vol ≤ 30000 at cad 420 PCSX vs 422 Godot; ≤ 10000 at 453 vs 465; 0 at 469 vs 487) — because each Godot cadence carries only 183 envelope ticks vs PCSX's 252, inverting the original "Godot decays 2.9× faster" sample-window read.** — `[D] 1/3`
  - D: `probe_envelope_tail` voice-21 cadence-time threshold crossings (2026-05-16)
  - R: none — cadence-time env_vol threshold crossings not present in godot-learning (probed godot-learning/src, godot-learning/tests; smd-player holds only the diagnostic/parity plumbing for `protect_no_music`)
  - src: `research/effect_sound/working_documents/ENVELOPE_TAIL_INVESTIGATION.md`
- **The ~3-cadence (~90 ms) attack-start offset at `protect_no_music` onset (PCSX reaches Sustain at cad 6, Godot at cad 9) is a probe-emit timing artifact, not a divergence: the PCSX SPU-IRQ probe can fire before FFT has seeded ADSR1/ADSR2 so its first row shows the prior (savestate-residue) envelope state, while Godot's emit begins one tick after `_Trace._post_anchor` already holding the new ADSR config and starting the attack from 0.** — `[D·R] 2/3`
  - D: `probe_envelope_tail` cad 2/6/9 rows (voice 20 at cad 2: PCSX sample 810 env_vol 19440 mid-attack vs Godot sample 26718 env_vol 4392) (2026-05-16)
  - R: `smd-player/addons/exmateria_sound/runtime/effect_sound/play_sound.gd::_emit_envelope_tail_for_runtime` (emit gated on the post-anchor `_cadence_anchored` pulse)
  - src: `research/effect_sound/working_documents/ENVELOPE_TAIL_INVESTIGATION.md`
- **The `protect_no_music` sustain plateau (cad 9 through ~85: `adsr_state == 2`, `env_vol == 32767` at every common cadence) and the transient Sustain→Attack→Sustain re-KEYON cycles at cad 19/22, 44/47, 63/67, 84/87 align at matching `cadence_index` values on both sides — the instrument-driven KEYON re-triggers are fully cadence-aligned.** — `[D] 1/3`
  - D: `probe_envelope_tail` cadence-aligned diff over 720 common (voice, cadence) rows (2026-05-16)
  - R: none — `protect_no_music` not present in godot-learning (probed godot-learning/src, godot-learning/tests; the Godot side is smd-player's capture harness — see the points above)
  - src: `research/effect_sound/working_documents/ENVELOPE_TAIL_INVESTIGATION.md`

## Notes

(empty — user territory)

## Related

- [[SPU Voice Engine]]
- [[Effect Sound Audio Divergence]]
- [[Effect Sound Timing]]
- [[PCSX-Redux Capture Rig]]
- [[SFX Index]]
