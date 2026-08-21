# KON KOFF Mask Dispatch

How FFT's per-voice KON/KOFF on/off state reaches the PSX SPU hardware: the game never writes the KON_LO/HI/KOFF_LO/HI hardware masks directly from its KON/KOFF dispatcher — the masks are built per-tick in RAM (per-channel `slot+0x64` voice masks, global-flag globals feeding the KOFF mask) and pushed to the hardware at the SPU IRQ flush from `DAT_80037080–86`. The 2026-05-12 two-stage probe investigation (write-BPs on the global-flag globals plus the dispatcher entry) confirmed that for `cure_no_music` the global SPU registers — KON/KOFF masks, SPUCNT, noise enable, main volume, reverb enable/config — are set once at sound-system init and never written at runtime. Godot's effect-sound runtime mirrors the per-voice KON/KOFF dispatch: direction and per-voice pairing match; only the bundled-mask event count differs (6 vs 9, Godot splits a bundled KOFF-before-KON into multiple emitted events). A 2026-05-12 static unwrap of the KOFF assembly loop adds the two KOFF-mask feeders — global `flag_B` (`DAT_80032A20`, five scattered writers) and the per-slot KOFF arm bit — and shows the active-slot list (`DAT_80032A50`) is a separate pool from the global channel pool; the per-IRQ batch invariant (decision on final state, not entry state) is recorded in [[KON KOFF IRQ Phasing]].

## Points

- **FFT performs no runtime global SPU register writes during effect-sound playback (7 s `cure_no_music` window): the KON/KOFF hardware masks, SPUCNT, noise-enable, main volume, reverb-enable and reverb-config registers are all set once at sound-system init — the reverb-enable registers (SPU+0x98/0x9A) are zeroed once at init (PC `0x80018990–94`) and never touched again, and FFT routes reverb in-RAM via per-channel `chan+0x70` instead of the SPU reverb registers.** — `[S·D·R] 3/3`
  - S: `0x80018990–94` (reverb-enable zeroing at init); `FUN_8001920C` (SPU IO write helper) has only 3 callers — both per-voice address writers + one path at PC `0x8001AFE4` that returns without writing — per `research/effect_sound/working_documents/AUDIO_PARITY_FINAL_STATE.md` §3
  - D: `probe_kon_koff_mask` + `diag_kon_koff_globals` write-BPs on `DAT_80032A0C`/`0x20`/`0x08` — zero writes in the cad 340–365 window (commits c6ac231b/ec90d7b1; 2026-05-11/12)
  - R: `smd-player/addons/exmateria_sound/runtime/shared/flush_tick.gd` (`_commit_kon` per-voice KON path) + `probe_kon_koff_mask` pairing
  - src: `research/effect_sound/working_documents/AUDIO_PARITY_FINAL_STATE.md`
- **FFT's final hardware KON/KOFF mask writes happen at the SPU IRQ flush from RAM globals `DAT_80037080–86`, not directly from the KON/KOFF dispatcher `FUN_8001ACF0`: the KOFF mask is assembled per-tick at PC `0x80014E04` from the per-channel `slot+0x64` voice masks, and the global-flag globals that could feed it have zero writes in the cure cadence window — runtime KOFF activity is fully driven by the per-channel voice masks.** — `[S·D·R] 3/3`
  - S: `FUN_8001ACF0` (KON/KOFF mask dispatcher), `DAT_80037080–86` (hardware-mask flush globals), `0x80014E04` (per-tick KOFF-mask assembly), `slot+0x64` (per-channel voice mask) — per `research/effect_sound/working_documents/AUDIO_PARITY_FINAL_STATE.md` §3
  - D: same probe/diag captures (2026-05-11/12)
  - R: Godot's per-voice `_commit_kon` (`smd-player/addons/exmateria_sound/runtime/shared/flush_tick.gd`) splits the bundled KOFF-before-KON pattern across multiple emitted events vs PCSX's single bundled mask — event count differs (6 vs 9) but direction and per-voice dispatches match after commit fbfe2830
  - src: `research/effect_sound/working_documents/AUDIO_PARITY_FINAL_STATE.md`
- **The PCSX KEY-ON commit is `FUN_8001ACF0(1, voice_mask)`, called from the epilogue of `FUN_80017118` (the per-IRQ flush), which runs inside the RCnt2 IRQ handler at `FUN_800149DC` (`process_pulse_tick` per the doc); the commit writes SPU port 0x1F801D88.** — `[S·R] 2/3`
  - S: `FUN_8001ACF0` (KON commit), `FUN_80017118` epilogue, RCnt2 IRQ handler `FUN_800149DC`, SPU port 0x1F801D88 — per the doc's citation of `ARCHITECTURE_MAP.md` §D (stale path `research/effect_alignment/`; the file now at `smd-player/workspace/ARCHITECTURE_MAP.md` corroborates the epilogue call chain in §D/§E)
  - R: `smd-player/addons/exmateria_sound/runtime/shared/flush_tick.gd::flush_kon_commit` (port of `FUN_8001ACF0(1, mask)` in `FUN_80017118`'s epilogue — one KON SPU write per IRQ, after the catchup sub-loop + all dispatcher ticks) + `flush_kon_only_for_slot` (port of `FUN_80017118`, PC 0x80017180–0x800173B8) — validated by the `probe_kon_koff_mask` pair (FFT BP @ 0x8001ACF0) and the `diag_keyon_per_voice` ledger
  - src: `research/effect_sound/working_documents/ENV_VOL_WITHIN_IRQ_KEY_ON_TIMING.md`
- **KON mask emission order differs between the two sides: for the ice cad-104 mask `0x370000`, PCSX emits the per-voice KONs low-to-high (16, 17, 18, 20, 21) while Godot's walker visits slots in slot-allocation order, putting long-lived voice 18 (active since cad 1) ahead of the freshly-armed 16/17 — both sides emit 5 KONs and the SPU processes the KON mask as a single register write, so the emission order has no audio effect (probe-level mismatch only, a red herring for the V16 cos_dist).** — `[S·D] 2/3`
  - S: keyon emission callsite `0x80014AC4`, `probe_keyon_per_voice` BP `0x8001ACF4` — `scus_disassembly.txt`
  - D: `probe_keyon_per_voice` 4 row-order mismatches inside the cad-104 mask-`0x370000` KON (`ice_no_music`, 2026-05-21)
  - R: none — the PCSX low-to-high mask emission order not present in smd-player or godot-learning (probed; `smd-player/addons/exmateria_sound/runtime/shared/spu_irq_walker.gd` visits slots in allocation order)
  - src: `research/effect_sound/working_documents/ICE_V16_TWO_CAD_PRE_ARM_FIX_PLAN.md`
- **`FUN_8001ACF0` (the SPU KON/KOFF dispatcher) takes `a0` = mode (0 = KOFF, 1 = KON, any other value = status-query path) and a 24-bit `a1` voice mask, and writes the memory-mapped pending KON/KOFF accumulators at `DAT_80037080–86` that the SPU IRQ handler later flushes to the real SPU `KON_LO/KON_HI/KOFF_LO/KOFF_HI` registers; its four call sites are `0x8001489C` (per-tick walker primary KON pass, mask accumulated through the per-slot loop), `0x80014AC4` (secondary KON pass), `0x80014EEC` (KOFF, `a1` = the accumulated `s1` KOFF mask), and `0x80017BC0` (stream-end / pool-flush KOFF, `a1 = 0xFFFFFF` = all voices).** — `[S·D·R] 3/3`
  - S: `FUN_8001ACF0` @ `0x8001ACF0`, `DAT_80037080–86`, caller table `0x8001489C` / `0x80014AC4` / `0x80014EEC` / `0x80017BC0` — working doc §"FFT's KON/KOFF model" (helper + callers sections)
  - D: `probe_kon_koff_mask` (BP @ `0x8001ACF0`, commit ec90d7b1) — mode-separated KON/KOFF mask lines, `cure_no_music` (2026-05-12)
  - R: `smd-player/addons/exmateria_sound/runtime/shared/flush_tick.gd::flush_kon_commit` (port of `FUN_8001ACF0(1, mask)`) + `flush_koff_post_loop` (KOFF commit) + `runtime/shared/per_tick/stream_end.gd` (stream-end KOFF via `FLAG_STREAM_END`) — validated by the `probe_kon_koff_mask` pair via `smd-player/workspace/orchestrator/run_effect_iteration.py`
  - src: `research/effect_sound/working_documents/KON_KOFF_STRUCTURAL_ISSUE.md`
- **The KOFF-mask assembly loop at PC `0x80014E04`–`0x80014E80` is fed by two sources: the global `flag_B` at `DAT_80032A20` (direct KOFF-mask contribution; static writer PCs `0x800128F0`, `0x80012B0C`, `0x80013E38`, `0x80015964`, `0x80016F24`, with the EndBar handler at `0x80015964` the likely voice-deactivation writer) and the per-slot `slot+0x10` bit 0x2 KOFF-arm flag (voice mask at `slot+0x64`, ORed in then cleared with `sw zero` after use) — the mask is seeded with `flag_C` (`DAT_80032A08`) | `flag_B`, and the active-slot list is walked from head `DAT_80032A50` with a `bgez` active test on `slot+0x10`; that linked-list pool is NOT the global channel pool `probe_walker_flag_word_entry` observes — write BPs on the global pool's `slot+0x10` never fire for its slots.** — `[S·D] 2/3`
  - S: decoded loop `0x80014E08`..`0x80014E7C` (`flag_A` `DAT_80032A0C`, `flag_B` `DAT_80032A20`, `flag_C` `DAT_80032A08`, list head `DAT_80032A50`, arm test `andi v0, v1, 0x2` @ `0x80014E48`, voice-mask OR @ `0x80014E5C`, clear @ `0x80014E70`) + the five `flag_B` writer PCs — working doc §"Continued investigation: FFT KOFF mask accumulator unwrapped"
  - D: write BPs on the global channel pool's `slot+0x10` do not fire for the `DAT_80032A50` list slots — separate-pool observation (2026-05-12)
  - R: none — `flag_B` / `DAT_80032A20` not present in smd-player or godot-learning (probed both)
  - src: `research/effect_sound/working_documents/KON_KOFF_STRUCTURAL_ISSUE.md`

## Notes

(empty — user territory)

## Related

- [[SPU Voice Engine]]
- [[PSX SPU Reverb]]
- [[Effect Sound Audio Divergence]]
- [[KON KOFF IRQ Phasing]]
- [[Ice V16 2-Cadence Pre-Arm]]
- [[SFX Index]]
