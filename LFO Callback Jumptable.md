# LFO Callback Jumptable

The LFO swap-callback jumptable at `PTR_LAB_80028F54` (ram:0x80028F54–0x80028F80) is how FFT picks the per-tick swap handler for an armed LFO sub-slot: the arm opcodes' (0xE5 vol-LFO / 0xED pan-LFO) `p2 & 0xF` byte indexes the 16-entry table, looked up by the 0xE5 handler at PC `0x800167EC`. idx 3 (`pitch_accum_callback`, 0x80017744) is the biphasic triangle that doubles the countdown on every other swap via its dir bit 0x4; idx 4/5 (`LAB_800177C0`) is sawtooth-with-reset; idx 2 (`LAB_800176E4`) is a uniphasic swap that reloads the countdown to `inner_reload` on every swap and toggles only dir bit 0x8; idx 8..15 are disarm-only stubs (`LAB_80017634`). `reraise_no_music` voice 21's `0xE5 4C 1B 02` is the first known production use of idx 2; while it fell through to the unmodeled triangle handler, Godot's doubled countdown missed PCSX's cad-382 swap and pegged `vol_register` at the SPU max (8839) from cad ~387 — closed by a dedicated `cb_idx == 2` branch in smd-player's `advance_lfo.gd` (also ported to `fft-sound-driver`).

## Points

- **FFT's per-tick LFO swap callback is selected per sub-slot from the 16-entry jumptable at `PTR_LAB_80028F54` (ram:0x80028F54–0x80028F80) by the arm opcode's (0xE5 vol-LFO / 0xED pan-LFO) `p2 & 0xF` byte — the 0xE5 handler performs the lookup at PC `0x800167EC` — with idx 0 = `LAB_80017648`, idx 1 = `LAB_80017690` (mode-1 swap), idx 2 = `LAB_800176E4`, idx 3 = `pitch_accum_callback` (0x80017744, triangle), idx 4/5 = `LAB_800177C0` (sawtooth-with-reset), idx 6 = `FUN_8001780C`, idx 7 = `FUN_80017878`, and idx 8..15 = `LAB_80017634` (disarm-only stub).** — `[S·D·R] 3/3`
  - S: PC `0x800167EC` (0xE5 handler jumptable lookup) + table entries `ram:80028f54`–`ram:80028f80` — doc §3 (ROM dump listing)
  - D: `master_parser.py` decode of `reraise_no_music.bin`'s four SMD channels (2026-05-16) — p2→cb_idx mapping confirmed for every armed sub-slot
  - R: `smd-player/addons/exmateria_sound/runtime/shared/per_tick/advance_lfo.gd` (dispatches cb_idx 2, 4/5, triangle else) + `fft-sound-driver/src/driver/per_tick.cpp` — validated by the `reraise_no_music` orchestrator probe pairs (`smd-player/workspace/orchestrator/run_effect_iteration.py`)
  - src: `research/effect_sound/working_documents/RERAISE_VOICE_21_LFO_CALLBACK_IDX_2_UNMODELED.md`
- **`LAB_800176E4` (jumptable idx 2) is a uniphasic swap callback: on countdown→0 it unconditionally reloads the countdown to `inner_reload` (`_sh v1, 0x10(a0)` in the branch delay slot at PC `0x80017714` — no dir-bit-0x4 doubling), writes step_current = ±step_source gated on dir bit 0x8, and toggles ONLY bit 0x8 of dir_flags (`xori v0, v0, 0x8` at PC `0x80017724`) — so every swap cycle runs at the same rate and step polarity alternates −,+,−,+.** — `[S·D·R] 3/3`
  - S: PC `0x800176E4`–`0x80017740` (reload `0x80017714`, dir commit `0x80017724`) — doc §4.1 MIPS listing
  - D: `probe_lfo_subslot1_state` voice-21 trace, `reraise_no_music` (2026-05-16) — PCSX cd reloads to 76 = inner_reload on every swap (cad 3, 384, 422), dir_flags oscillate 3↔11 (bit 0x4 never set), step_current sign alternates
  - R: `smd-player/addons/exmateria_sound/runtime/shared/per_tick/advance_lfo.gd:242` (cb_idx == 2 branch porting LAB_800176E4) + `fft-sound-driver/src/driver/per_tick.cpp:176` — validated by the `reraise_no_music` orchestrator run (pre-fix `probe_vol_register` 16/167 wrong rows + `probe_lfo_subslot1_state` 76-tick countdown offset on voice 21; post-fix expectations in doc §9)
  - src: `research/effect_sound/working_documents/RERAISE_VOICE_21_LFO_CALLBACK_IDX_2_UNMODELED.md`
- **`LAB_80017744` (jumptable idx 3, `pitch_accum_callback`) is a biphasic triangle: on swap it commits cd = inner_reload and doubles it only when dir bit 0x4 is set (`sll v1, v1, 0x1` at PC `0x8001777C`), then commits `dir = (dir | 0x4) ^ 0x8` (ori at PC `0x8001779C`, xori at `0x800177A0`) — so bit 0x4 is set on every swap and the NEXT swap's countdown reload doubles, giving the 76→152→76→152 swap rhythm that defines the triangle-with-pre-roll waveform.** — `[S·D·R] 3/3`
  - S: PC `0x80017768`/`0x8001777C`/`0x8001779C`/`0x800177A0` — doc §4.2 MIPS listing
  - D: pre-fix `reraise_no_music` voice-21 trace (2026-05-16) — with Godot running the triangle handler on a cb_idx-2 voice, cd reloaded to 152 on swap 2 and dir_flags oscillated 7↔15 (bit 0x4 stuck set), confirming the doubling
  - R: `smd-player/addons/exmateria_sound/runtime/shared/per_tick/advance_lfo.gd` (else branch — triangle mirror) + `fft-sound-driver/src/driver/per_tick.cpp` — validated on cure_4 voice 18/19 (triangle p2=3 arms, per dispatcher.gd:2693–2696 comment) and reraise voice 19 sub-slot 2
  - src: `research/effect_sound/working_documents/RERAISE_VOICE_21_LFO_CALLBACK_IDX_2_UNMODELED.md`
- **`reraise_no_music` voice 21 (slot 5, SMD channel 1) is the first known production use of LFO callback idx 2 — its `0xE5 4C 1B 02` arm (period 0x4C=76, rate 0x1B=27, p2=2) selects sub-slot 1; voice 19 arms sub-slot 2 with 0xED p2=3 (triangle), voices 18/20 arm no LFO sub-slot at all, and the protect/cure/cure_4 parity targets use only cb_idx ∈ {3, 4}.** — `[D] 1/3`
  - D: `master_parser.py` decode of all four `reraise_no_music` SMD channels + the protect/cure/cure_4 arm scan (doc §6/§7.3, 2026-05-16)
  - R: none — LFO arm distribution not present in godot-learning (probed godot-learning/src, godot-learning/tests, smd-player, fft-sound-driver)
  - src: `research/effect_sound/working_documents/RERAISE_VOICE_21_LFO_CALLBACK_IDX_2_UNMODELED.md`
- **The reraise voice 21 LFO step source differs by exactly 1 LSB between the engines (PCSX −5960326 vs Godot −5960325, constant across every `probe_lfo_subslot1_state` row) — a rounding artifact of the shared `pitch_lfo_step_calc` port (FFT PC `0x80016BF8`), judged negligible for vol parity because the cos_dist drift is dominated by swap cadence, not step magnitude.** — `[D·R] 2/3`
  - D: `probe_lfo_subslot1_state` step_source column voice 21, both sides, `reraise_no_music` (2026-05-16)
  - R: `smd-player/addons/exmateria_sound/runtime/shared/helpers/lfo_step_calc.gd` (port of pitch_lfo_step_calc @ 0x80016BF8) — the 1-LSB gap is open, suspected floor-vs-round in the shift step (doc §11 Q5)
  - src: `research/effect_sound/working_documents/RERAISE_VOICE_21_LFO_CALLBACK_IDX_2_UNMODELED.md`

## Notes

(empty — user territory)

## Related

- [[LFO Sub-Slot 0 Pitch LFO]]
- [[LFO Sub-Slot 2 Pan LFO]]
- [[LFO Sub-Slot Period Reset]]
- [[LFO Subslot Residue]]
- [[LFO Handler Probe Cadence]]
- [[Effect Sound Audio Divergence]]
- [[Cure 4 Audio Parity]]
- [[SPU Voice Engine]]
- [[SMD Opcodes]]
