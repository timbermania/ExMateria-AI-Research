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

## Notes

(empty — user territory)

## Related

- [[SPU Voice Engine]]
- [[PSX SPU Reverb]]
- [[Effect Sound Audio Divergence]]
- [[SFX Index]]
