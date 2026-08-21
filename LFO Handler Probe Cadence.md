# LFO Handler Probe Cadence

The four `lfo_handler` trace probes (`lfo_handler_entry`, `lfo_subslot0/1/2_state`) mirror a single PCSX breakpoint at the per-channel loop top of `lfo_handler_tick` (PC 0x800174C8), which fires once per channel × entity-iter *before* the `chan_word_0` gate, so all four probes carry identical PCSX row counts (reraise_no_music: 2504 = 313 × 8). The Godot port originally emitted these rows per-IRQ from the LFO state-advance path (gated on `cadence_fired`), leaving a 40-row deficit per probe; the emits were relocated into the per-entity-iter `cadence_body` walk while LFO state mutation stayed per-IRQ. A later haste-session investigation found the top-of-`cadence_body` placement captured pre-dispatch state one cadence stale, so the main emit now runs at the BOTTOM of `cadence_body` after state mutation (see [[Effect Sound Audio Divergence]]). The 2026-05-24 haste pairing investigation found the default per-emit monotonic `call_index` pairing breaks when PCSX (entity-list order) and smd-player (array order) iterate the slot pool differently — a 4-slot shift on haste — and the six lfo_handler-family probes now compute a position-invariant `call_index = cadence_index * 16 + slot_idx` on both sides.

## Points

- **FFT's `lfo_handler_tick` (FUN_800174A8) is invoked once per entity-iter × all 8 channel positions from the entity-accumulator sub-loop (PC 0x80014d34..0x80014d68), and its per-channel loop-top BP @ 0x800174C8 fires BEFORE the `chan_word_0 != 0` gate at PC 0x800174D0 — so null/never-initialized channels still register a row, and all four lfo_handler probes (entry + subslot 0/1/2 state) collocate on that single BP, giving them identical PCSX row counts (reraise_no_music: 2504 = 313 entity-iters × 8 channels).** — `[S·D·R] 3/3`
  - S: `0x800174C8` (`LAB_800174C8`, `lhu v0,0x0(s4)` — per-channel loop top) + `0x800174D0` (`beq v0,zero,LAB_800175F4` — chan_word_0 gate) + invocation site PC 0x80014d34..0x80014d68, per `project-assets/fft-rom/scus_disassembly.txt`
  - D: `probe_lfo_handler_entry` + `probe_lfo_subslot{0,1,2}_state` PCSX captures, reraise_no_music `last_run` (2026-05-17) — 2504 rows each
  - R: `smd-player/addons/exmateria_sound/runtime/effect_sound/probes/probe_emit.gd` `emit_lfo_handler_probes` (docstring: "mirroring PCSX BP @ 0x800174C8 … to fire once per channel per entity-iter") + `runtime/shared/dispatcher.gd::cadence_body` call sites — validated by the `probe_lfo_handler_entry` / `probe_lfo_subslot*_state` paired entries in `smd-player/workspace/orchestrator/probe_validation_manifest.py`
  - D (additive): BP family grew to six probes by the 2026-05-24 haste pairing investigation — `probe_lfo_subslot3_state` and `probe_chan_pitch_state` added to the same BP @ `0x800174C8` family (`PROBE_PAIRING_ITERATION_ORDER_FIX.md`)
  - src: `research/effect_sound/working_documents/LFO_HANDLER_PER_IRQ_GATE_DEFICIT.md`

- **The `lfo_handler_tick` per-channel loop walks 8 chan positions at 0x160 stride; each active channel carries 3 LFO sub-slots (sub-slot 0 base = chan+0xE0, active/dir flag pointer = chan+0xFC, active bit at sub+0xFE, sub-slot stride 0x20) — the sub-slot loop counter initializes to 4 but the 4th iter overruns and is skipped, and inactive sub-slots skip dispatch at the PC 0x800174F0 gate.** — `[S] 1/3`
  - S: `0x800174D8` (`ori s3,zero,0x4` — counter init), `0x800174DC` (sub-slot 0 base chan+0xE0), `0x800174E0` (flag pointer chan+0xFC), `0x800174F0` (sub-slot active gate), `0x800175E4..0x800175F4` (0x20 stride advance), `0x80017604..0x80017608` (outer 0x160 stride loop), per `project-assets/fft-rom/scus_disassembly.txt`
  - R: none — lfo_handler sub-slot stride (0x20) / chan+0xE0 base not present in godot-learning; smd-player tracks sub-slot state in its own channel fields (flat `lfo_*` for sub-slot 0, `lfo_sub_*` arrays for sub-slots 1/2), not ROM offsets
  - src: `research/effect_sound/working_documents/LFO_HANDLER_PER_IRQ_GATE_DEFICIT.md`

- **Godot's four lfo_handler probe rows were originally emitted per-IRQ from `_advance_lfo`, gated on `cadence_fired` — a call-context mismatch with PCSX's per-entity-iter BP that left each probe 40 rows short (Godot 2464 vs PCSX 2504 on reraise_no_music: ~5 multi-iter IRQs × 8 channels under-emitting, plus init-iter drift); the emits were relocated into the per-entity-iter `cadence_body` walk (one row per channel × entity-iter, including the early-return branches) while LFO state mutation stays per-IRQ in the advance_lfo tick script.** — `[S·D·R] 3/3`
  - S: PCSX side of the mismatch — BP @ `0x800174C8` invoked per entity-iter from sub-loop PC 0x80014d34..0x80014d68, per `project-assets/fft-rom/scus_disassembly.txt`
  - D: pre-fix row counts — Godot `lfo_handler_entry` / `lfo_subslot{0,1,2}_state` 2464 each vs PCSX 2504 each, reraise_no_music `last_run` (2026-05-17)
  - R: `smd-player/addons/exmateria_sound/runtime/shared/dispatcher.gd::cadence_body` (`_ProbeEmit.emit_lfo_handler_probes` at the `stream_end_fired` / `chan_word_0 == 0` early-return branches + cadence_body bottom) + `runtime/shared/per_tick/advance_lfo.gd::apply` (per-IRQ `chan_88`/`chan_8a` clear + sub-slot dispatch, untouched) + `runtime/effect_sound/play_sound.gd:1312,1318` (null-slot `emit_lfo_handler_inactive`) — validated by the `probe_lfo_handler_entry` / `probe_lfo_subslot*_state` paired entries in `smd-player/workspace/orchestrator/probe_validation_manifest.py`
  - src: `research/effect_sound/working_documents/LFO_HANDLER_PER_IRQ_GATE_DEFICIT.md`

- **smd-player tracks sub-slot 0 (D9 pitch LFO) in flat channel fields (`lfo_current_output`, `lfo_step_base`, `lfo_countdown`, …), NOT in `lfo_sub_*[0]` (all-zero stubs), while sub-slots 1/2 (volume/pan LFO) are array-tracked — the lfo_handler probe emit maps sub-slot 0 from the flat fields (hard-coding `step_current`/`delay_*`/`depth*` = 0, `mode` = 0) and sub-slots 1/2 from the `lfo_sub_*` arrays.** — `[R] 1/3`
  - R: `smd-player/addons/exmateria_sound/runtime/effect_sound/probes/probe_emit.gd::emit_lfo_handler_probes` (subslot0 block reads `channel.lfo_current_output`/`lfo_step_base`/`lfo_countdown`; subslot1/2 blocks read `lfo_sub_accumulator[1/2]` et al.) — validated by the `probe_lfo_subslot0_state` / `probe_lfo_subslot1_state` / `probe_lfo_subslot2_state` paired entries in `smd-player/workspace/orchestrator/probe_validation_manifest.py`
  - src: `research/effect_sound/working_documents/LFO_HANDLER_PER_IRQ_GATE_DEFICIT.md`

- **The default `call_index` row pairing breaks under the PCSX-vs-smd-player iteration-order mismatch: PCSX's outer walker visits the LFO slot pool in entity-list order (per-entity `lfo_handler_tick` call starting at the active entity's first slot) while smd-player walks channels 0..7 in array order, so the per-emit monotonic `call_index` maps PCSX slot N to Godot channel (N+4) mod 8 on the haste capture — `probe_chan_pitch_state.pre_pitch_hi` shows 1556/1664 "divergent" rows under call_index pairing vs ~390 real audible-slot rows once re-paired by (slot_idx, cadence_index).** — `[S·D·R] 3/3`
  - S: `0x800174A4` (`lfo_handler_tick` body, called per entity by the outer walker) + `0x800174C8` (per-channel inner loop) — PCSX disasm per doc §6.1
  - D: `probe_chan_pitch_state.pre_pitch_hi` mis-pairing — 1556/1664 call_index-paired rows divergent, 4-slot shift table (slot N vs channel (N+4) mod 8), `haste` capture (2026-05-24)
  - R: `smd-player/addons/exmateria_sound/runtime/shared/dispatcher.gd` (per-entity-iter `cadence_body` walk over the `_channels` array in index order) + `runtime/effect_sound/play_sound.gd` (per-channel probe emit) — the mis-pairing itself is exposed by `smd-player/workspace/orchestrator/validate_probe_pair.py`'s call_index-dict pairing
  - src: `research/effect_sound/working_documents/PROBE_PAIRING_ITERATION_ORDER_FIX.md`

- **The fix replaces the per-emit monotonic `call_index` with a position-invariant `call_index = cadence_index * 16 + slot_idx` computed on both engines for all six lfo_handler-family probes — PCSX computes `slot_idx = (chan_base - 0x80037198) / 0x160` from SLOT_TABLE_BASE `0x80037198` / SLOT_STRIDE `0x160`, Godot uses `channel.channel_idx` (0..7), and the null-channel `emit_lfo_handler_inactive` uses the same formula with the null channel's array index so call_index covers the full 0..7 range every IRQ on both sides; the ×16 multiplier (not ×8) leaves expansion headroom and keeps it a power of two, and the max value (1663 × 16 + 7 = 26615 on 1664 IRQs) stays in int32 range.** — `[S·R] 2/3`
  - S: `0x80037198` (SLOT_TABLE_BASE, slot-0 chan_base), `0x160` (SLOT_STRIDE), `0x800174C8` (shared BP of the six-probe family) — PCSX disasm per doc §6.1/§6.2
  - R: `smd-player/addons/exmateria_sound/runtime/effect_sound/probes/probe_emit.gd` (all six active emit sites L78/91/109/127/145/172 + `emit_lfo_handler_inactive` L204/220 use `_Trace._cadence_index * 16 + …`) — validated by the `probe_lfo_handler_entry` / `probe_lfo_subslot{0,1,2,3}_state` / `probe_chan_pitch_state` paired entries in `smd-player/workspace/orchestrator/probe_validation_manifest.py`
  - src: `research/effect_sound/working_documents/PROBE_PAIRING_ITERATION_ORDER_FIX.md`

## Notes

(empty — user territory)

## Related

- [[SFX Index]]
- [[Noise LFO PRNG]]
- [[LFO Sub-Slot 0 Pitch LFO]]
- [[KON KOFF IRQ Phasing]]
- [[Dormant Slot Residue]]
- [[Effect Sound Audio Divergence]]
- [[SPU Voice Engine]]
