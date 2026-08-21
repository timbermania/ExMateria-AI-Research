# KON KOFF Mask Dispatch

How FFT's per-voice KON/KOFF on/off state reaches the PSX SPU hardware: the game never writes the KON_LO/HI/KOFF_LO/HI hardware masks directly from its KON/KOFF dispatcher — the masks are built per-tick in RAM (per-channel `slot+0x64` voice masks, global-flag globals feeding the KOFF mask) and pushed to the hardware at the SPU IRQ flush from `DAT_80037080–86`. The 2026-05-12 two-stage probe investigation (write-BPs on the global-flag globals plus the dispatcher entry) confirmed that for `cure_no_music` the global SPU registers — KON/KOFF masks, SPUCNT, noise enable, main volume, reverb enable/config — are set once at sound-system init and never written at runtime. Godot's effect-sound runtime mirrors the per-voice KON/KOFF dispatch: direction and per-voice pairing match; only the bundled-mask event count differs (6 vs 9, Godot splits a bundled KOFF-before-KON into multiple emitted events).

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

## Notes

(empty — user territory)

## Related

- [[SPU Voice Engine]]
- [[PSX SPU Reverb]]
- [[Effect Sound Audio Divergence]]
- [[KON KOFF IRQ Phasing]]
- [[Ice V16 2-Cadence Pre-Arm]]
- [[SFX Index]]
