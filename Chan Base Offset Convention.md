# Chan Base Offset Convention

FFT's chan-side pitch state lives in the main-RAM chan block, and the disassembly exposes two conflicting offset conventions for it: the vol-formula / lfo_handler_tick / pitch-formula prologues run with `s0 = chan_base + 0x2` while the D-family opcode handlers receive the raw `chan_base`, so the same nominal s16 displacement addresses different physical fields in the two contexts. The core pitch fields (raw offsets): `pre_pitch_lo` at chan+0x80 and `pre_pitch_hi` at chan+0x82 (the pitch formula consumes only the high half of the u32 pre-pitch accumulator), `word_86` at chan+0x86 (the slow-modulation accumulator written by the D0/D1/D2/D3 opcode handlers and read as the Note-baseline formula's second addend at L80015444), and `pitch_bend` at chan+0x88 (written by the `lfo_handler_tick` mode-0 commit at `0x800175A4` and read by the pitch formula at `0x80017344`). The 2026-05-24 haste probe investigation pinned a 2-byte mis-pairing in `probe_chan_pitch_state` (raw `s4+0x86` read labeled `pitch_bend`) that had made word_86's D-opcode trajectories look like pitch_bend divergence; smd-player's `chan_pitch_state` probe now emits all four raw-halfword fields separately, voice 20's pitch_bend verified zero on both engines, with voice 19's mode-0 pitch_bend (Godot alternating ±400) still an open divergence.

## Points

- **FFT's chan block holds two adjacent s16 pitch fields with different writers and consumers: `word_86` at raw `chan_base+0x86` — the slow-modulation accumulator written by the D0/D1/D2/D3 opcode handlers (0xD1 `smd_add_pitch_bend` at `0x800162D8` does `lhu/sh 0x86(a2)` on raw chan_base a2) and read as the second addend of the Note-baseline formula (L80015444, `lh v1, 0x84(s0)` with s0 = chan_base+0x2) — and `pitch_bend` at `chan_base+0x88`, which the `lfo_handler_tick` mode-0 commit writes (`sh v0, 0x86(s0)` at `0x800175A4`) and the pitch formula reads (`lhu v0, 0x86(s0)` at `0x80017344`).** — `[S·D·R] 3/3`
  - S: PC `0x80017344` (formula pitch_bend read), `0x800175A4` (mode-0 commit store), L80015444 (word_86 Note-baseline read), `0x800162D8`..`0x80016300` (0xD1 `0x86(a2)` accumulator write) — `scus_disassembly.txt` (doc §1.2, §3, §10.3)
  - D: `probe_chan_pitch_state` + `probe_pitch_formula_stages` cross-check, `haste_no_music` — slot 4 (voice 20, chan_base 0x80037718) word_86 trajectory 0/128/256/384/0 at cadences 194/254/307/322 (D1 byte=4 writes, D0 byte=0) vs pitch_bend zero across all formula rows (2026-05-24)
  - R: `smd-player/addons/exmateria_sound/runtime/shared/channel_state.gd` (word_86 :164, pitch_bend :138, carrying the field-identity comment) + `runtime/shared/opcodes/pitch_bend_add.gd` (0xD1 writes word_86) + `runtime/effect_sound/probes/probe_emit.gd::emit_lfo_handler_probes` (post-fix chan_pitch_state emit of both fields) — validated by the `probe_chan_pitch_state` / `probe_pitch_formula_stages` pairs in `smd-player/workspace/orchestrator/probe_validation_manifest.py`
  - src: `research/effect_sound/working_documents/PROBE_CHAN_PITCH_STATE_OFFSET_FIX.md`
- **FFT's vol-formula / lfo_handler_tick / pitch-formula prologues run with `s0 = chan_base + 0x2` (`addiu s0, s4, 0x2` at `0x800174B4`, `addiu s0, s3, 0x2` at `0x80017188`) while the D-family opcode handlers receive the raw chan_base (a2), so the same nominal halfword displacement addresses different physical fields in the two contexts — `0x86(s0)` in the formula context is chan+0x88, but `0x86(a2)` in the D handler is chan+0x86.** — `[S·D·R] 3/3`
  - S: PC `0x800174AC` (`move s4, a1` — s4 = raw chan_base), `0x800174B4` / `0x80017188` (+2 base setup), `0x800162D8` (0xD1 raw-a2 write) — `scus_disassembly.txt` (doc §1.1–1.2, §10.3)
  - D: empirical — `probe_chan_pitch_state`'s raw `s4+0x86` read observed the D1 byte=4 +128 writes on slot 4, confirming a2 = raw chan_base in the D handlers (`haste_no_music`, 2026-05-24)
  - R: `smd-player/addons/exmateria_sound/runtime/effect_sound/probes/probe_emit.gd` (field-map comment block in raw chan_base offsets, citing this doc) + `runtime/shared/channel_state.gd` (per-field s0-relative offset comments) — validated by the `probe_chan_pitch_state` pair
  - src: `research/effect_sound/working_documents/PROBE_CHAN_PITCH_STATE_OFFSET_FIX.md`
- **The pitch formula's `pitch_base` input is the HIGH halfword of the u32 pre-pitch accumulator stored at `chan_base+0x80..0x83` — the formula reads `lh a0, 0x80(s0)` at PC `0x80017340` with s0 = chan_base+0x2, so the physical field is `chan_base+0x82`, never the low half at `chan_base+0x80` (the pre-fix `probe_chan_pitch_state` emitted the low half as pitch_base, off by 2 bytes).** — `[S·R] 2/3`
  - S: PC `0x80017340` (`lh a0, 0x80(s0)`) — `scus_disassembly.txt` (doc §1.2, §4.4)
  - R: `smd-player/addons/exmateria_sound/runtime/shared/channel_state.gd::pre_pitch_acc_u32` + `runtime/effect_sound/probes/probe_emit.gd` (post-fix emit of both `pre_pitch_lo` and `pre_pitch_hi`) — validated by the `probe_chan_pitch_state` pair
  - src: `research/effect_sound/working_documents/PROBE_CHAN_PITCH_STATE_OFFSET_FIX.md`
- **Voice 20 (slot 4, `haste_no_music`) formula-stage pitch_bend is 0 on both PCSX and Godot across all 45 paired formula rows — zero divergence on the real pitch_bend field; the four non-zero transitions previously reported for this voice's "pitch_bend" were word_86 D-opcode-accumulator writes, and voice 20's pitch_bend sees no writers at all (no mode-0 LFO firings, no others).** — `[S·D·R] 3/3`
  - S: `0x80017344` (formula pitch_bend read) + `0x800175A4` (the sole mode-0 writer, which never fires for this voice) — `scus_disassembly.txt` (doc §2)
  - D: `probe_pitch_formula_stages` 45 paired rows, 0 non-zero on slot 4 (chan_base 0x80037718), `haste_no_music` (2026-05-24)
  - R: smd-player's pitch-formula input `channel.pitch_bend` (`runtime/shared/channel_state.gd:138`) — validated by the `probe_pitch_formula_stages` pair
  - src: `research/effect_sound/working_documents/PROBE_CHAN_PITCH_STATE_OFFSET_FIX.md`
- **Voice 19's LFO mode-0 pitch_bend remains divergent between the engines as of the doc date on `haste_no_music`: Godot emits an alternating ±400 pattern while PCSX shows a different `probe_pitch_formula_stages` pattern — a real, probe-rooted divergence (voice 19 cos_dist 0.029) that the probe offset fix explicitly does not address.** — `[D·R] 2/3`
  - D: `probe_pitch_formula_stages` / `probe_chan_pitch_state` voice-19 rows, `haste_no_music` (2026-05-24)
  - R: `smd-player/addons/exmateria_sound/runtime/shared/per_tick/advance_lfo.gd:228` (mode-0 commit, port of PC `0x800175A4`–`0x800175E0`, writes `channel.pitch_bend`) — divergence open at doc date; no validating test
  - src: `research/effect_sound/working_documents/PROBE_CHAN_PITCH_STATE_OFFSET_FIX.md`

## Notes

(empty — user territory)

## Related

- [[LFO Sub-Slot 0 Pitch LFO]]
- [[PSX Pitch Conversion]]
- [[Noise LFO PRNG]]
- [[Effect Sound Audio Divergence]]
- [[Dormant Slot Residue]]
- [[LFO Handler Probe Cadence]]
