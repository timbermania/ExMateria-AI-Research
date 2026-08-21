# LFO Sub-Slot 2 Pan LFO

FFT's mode-2 (pan-LFO) sub-slot, and the `protect_no_music` vol_inputs deficit that exposed it: an active mode-2 sub-slot fires on every fired-cadence tick, stepping the pan-LFO accumulator into chan+0x8a and co-arming `CHAN1_VOL_PRESTAGE` (chan_word_1 bit 0x100) on the same commit path as mode 1's chan+0x88 writer — so a single active mode-2 sub-slot alone re-arms the vol prestage every tick. On `protect_no_music` voice 21 (ch1) carries an active mode-2 sub-slot on PCSX (chan_8a ramping on 161/167 vol_inputs rows) although neither ch0 nor ch1 bytecode contains any documented LFO-arm opcode (0xD8/0xD9/0xE5/0xED) and 0xEC/0xEF contribute exactly 2 prestage arms; the arming source is unresolved (top candidate: Note handler implicit arm at PC 0x80015458). The missing per-tick co-arm is exactly 91 `vol_inputs` rows on the porta-active ticks of the dt=96 Note hold, and shows up mix-level as a phase-interference fingerprint (full_mix cos_dist 0.090 → 0.155 post-6-opcode-fix). Godot's mirror implements the mode-2 loop but defaults all four sub-slot modes to 0 (FFT pre-seeds [0,1,2,0]), arms sub-slot 2 only from the documented opcodes, and advances LFO state on voiced channels only where PCSX walks all 8.

## Points

- **An active mode-2 (pan-LFO) sub-slot of FFT's `lfo_handler_tick` fires on every fired-cadence tick, independent of opcode dispatch: it updates the sub-slot accumulator, stores it to the pan-LFO accumulator at chan+0x8a (PC `0x800175DC`), and ORs 0x100 (`CHAN1_VOL_PRESTAGE`) into `chan_word_1` on the same commit path as mode 1's chan+0x88 writer (both converge at PC `0x800175E0`) — so a single active mode-2 sub-slot alone re-arms the vol prestage on every tick.** — `[S·D·R] 3/3`
  - S: mode-2 block PC `0x800175CC`–`0x800175D8` (accumulator store `sh v0, 0x8a(s0)` at `0x800175DC`; `ori v1, v1, 0x100` before the `0x800175E0` convergence); mode-1 twin `0x800175B4`–`0x800175C8` (store at `0x800175C8`) — `scus_disassembly.txt` (doc §3.1/§3.2/§7)
  - D: `probe_vol_inputs` on protect_no_music voice 21 (ch1, s0 = 0x8003787A, 2026-05-15) — chan_8a != 0 on 161/167 rows with 33 distinct ramping values while chan_88 == 0 on all 167 rows, i.e. mode 2 firing every tick with mode 1 silent
  - R: `smd-player/addons/exmateria_sound/runtime/shared/per_tick/advance_lfo.gd:233-295` (sub-slot loop for modes 1/2: mode 2 accumulates into `chan_8a_value` then `channel.channel_word_1 |= _SS.CHAN1_VOL_PRESTAGE`; the doc's `smd-player/scripts/effect_sound/dispatcher.gd::_advance_lfo` was later refactored into this module) — validated by the `probe_vol_inputs` / `probe_vol_lr_staging` pairs in `smd-player/workspace/orchestrator/probe_validation_manifest.py`
  - src: `research/effect_sound/working_documents/PROTECT_RESIDUAL_VOL_INPUTS_DEFICIT.md`
- **On `protect_no_music`, voice 21's channel (ch1) carries an active mode-2 sub-slot on PCSX although neither ch0's nor ch1's bytecode contains any of the documented LFO-arm opcodes 0xD8/0xD9/0xE5/0xED and the just-wired 0xEC/0xEF pair contributes exactly 2 `CHAN1_VOL_PRESTAGE` arms total — so the sub-slot-2 active flag is set by something else; the doc's ranked candidates are a Note-handler implicit arm (`smd_note_handler` @ PC `0x80015458`, arming whatever default mode is pre-seeded), an engine-init default of active=1 for sub-slot 2, or an unmodeled opcode (unlikely: the protect dispatch trace at HEAD `ba9acddc` shows zero unhandled opcodes).** — `[S·D] 2/3`
  - S: ch1 bytecode `BA E0 E2 AC 94 EC D4 N(96) E0 AC 94 D4 D5 C2 E0 E2 N(24) E0 E2 N(24) 80 EF E0 E2 D4 AC 94 C4 N(32) 90` (no D8/D9/E5/ED; doc §3.3) + candidate PC `0x80015458` (smd_note_handler) + 0xEC/0xEF handler PCs `0x800168A8` / `0x800168DC` — `scus_disassembly.txt`
  - D: protect_no_music dispatch trace + `probe_lfo_swap` ledger (empty — the active sub-slot is NOT mode 1; mode 2 has no equivalent swap probe, so it is invisible to that ledger), HEAD `ba9acddc` (2026-05-15)
  - R: none — smd-player models no implicit sub-slot-2 arming: `lfo_sub_active[2]` is written only by the documented arm handlers (`runtime/shared/opcodes/lfo_arm_subslot2.gd:57` for 0xEC, `pan_lfo.gd:37`), cleared by `clear_subslot2.gd:27` (0xEF), and seeded only for silent drivers (`play_sound.gd:700`); no note_handler write to the sub-slot active flags (probed smd-player, fft-sound-driver, godot-learning)
  - src: `research/effect_sound/working_documents/PROTECT_RESIDUAL_VOL_INPUTS_DEFICIT.md`
- **The entire 95-row `probe_vol_inputs` deficit on `protect_no_music` (PCSX 233 vs Godot 138) sits on voice 21 (167 → 72 rows; voice 20's 66 rows PAIR) and is confined to the dt=96 Note hold starting at cadence ≈128 with a [3,2,3,2,…] cadence rhythm: on 91 of those ticks Godot's pre-clear `chan_word_1` at the FUN_80017118 entry reads 0x200 (pitch-only) where PCSX holds 0x300 (pitch + vol) — Godot produces the pitch prestage on the right porta-active ticks (0x200 set by the per_channel_tick porta block at PC `0x80015210` after ch1's 0xD4 init + the `ba9acddc`-enabled 0xD5 indefinite porta) but misses the 0x100 co-arm, and the 4 remaining rows are KOFF-tagged variants; the vol drainer is gated on `CHAN1_VOL_PRESTAGE`, so the missing co-arms are exactly the missing vol_inputs rows.** — `[S·D·R] 3/3`
  - S: porta block 0x200 setter PC `0x80015210`; pre-clear `chan_word_1` read at FUN_80017118 entry (bit-0x100 check PC `0x800171B8`, per-slot clear `sh zero, 0x0(s0)` at PC `0x800173B8`) — `scus_disassembly.txt` (doc §2.3/§7)
  - D: `probe_vol_inputs` / `probe_vol_lr_staging` (233 vs 138) + `probe_fun80017118_clear` pre-clear bit distribution (0x200 +91 rows, 0x300 −91 rows, KOFF-tagged variants accounting for the remaining 4 of the 95) on protect_no_music, HEAD `ba9acddc` (2026-05-15)
  - R: `smd-player/addons/exmateria_sound/runtime/effect_sound/play_sound.gd:1551` (the emit gate `if (s1 & _SS.CHAN1_VOL_PRESTAGE) != 0`; the doc's `play_sound.gd:901`) + `runtime/shared/per_tick/advance_lfo.gd:235-238` (sub-slots with `lfo_sub_active == 0` or mode not 1/2 skip the fire, so an un-armed or mode-0 sub-slot 2 produces no co-arm) — no validating test named in the doc
  - src: `research/effect_sound/working_documents/PROTECT_RESIDUAL_VOL_INPUTS_DEFICIT.md`
- **FFT pre-seeds all four LFO sub-slots of each channel at engine init with default modes [0, 1, 2, 0] (slot 2 = mode 2 = pan-LFO), and the active flag (chan+0xFE_for_subslot bit 0x1) defaults to 0 — so whatever implicitly arms sub-slot 2 runs in mode 2 on PCSX; the Godot mirror defaults all four `lfo_sub_mode` to 0 (`_default_mode = [0,0,0,0]`) and its per-tick loop skips sub-slots whose mode is not 1/2, so an armed sub-slot 2 on Godot can never produce the mode-2 dispatch.** — `[R] 1/3`
  - R: `smd-player/addons/exmateria_sound/runtime/shared/channel_state.gd:447` (`_default_mode = PackedByteArray([0, 0, 0, 0])`) + `:400` (`lfo_sub_active = PackedByteArray([0, 0, 0, 0])`) + `runtime/shared/per_tick/advance_lfo.gd:238` (mode 1/2 gate); mirrored at `fft-sound-driver/src/driver/channel_state.h:172` (`lfo_sub_mode{0, 0, 0, 0}`) — no validating test named in the doc
  - src: `research/effect_sound/working_documents/PROTECT_RESIDUAL_VOL_INPUTS_DEFICIT.md`

## Notes

(empty — user territory)

## Related

- [[LFO Sub-Slot 0 Pitch LFO]]
- [[LFO Sub-Slot Period Reset]]
- [[LFO Subslot Residue]]
- [[LFO Handler Probe Cadence]]
- [[SPU Voice Engine]]
- [[Effect Sound Audio Divergence]]
- [[Portamento Tick Ordering]]
- [[KON KOFF IRQ Phasing]]
- [[SFX Index]]
