# Chan 0x08 Residue Gate

The runtime-patched sound-engine blob at RAM `0x80150AB0` gates the `protect_no_music` per-cadence `chan+0x92` clear (the volume-zero mute of voice 20) on the *previous* voice's `chan+0x08` "last sound state" residue: residue `0` (never used by an SFX since boot/savestate) or `0x011B` fires the clear (`beq t0, zero` at PC `0x80150ACC`, `beq t0, 0x011B` at PC `0x80150AD8`, action `sh zero, 0x92(s0)` at `0x80150AEC`); any other value skips the mute. The 2026-05-15 `protect_no_music` capture showed the gate's deciding input is savestate-determined: slot pair 1 (voices 18/19) was idle when the savestate was taken and PCSX-Redux's `silenceAllVoices()` does not reset WRAM channel structs, so voice 19's residue read `0x0000`, the gate fired, and voice 20's `chan_92` was cleared — while a real PSX reaching `protect_no_music` from an arbitrary gameplay moment almost always carries a non-zero, non-`0x011B` residue in voice 19 from a prior SFX, leaving voice 20 audible. smd-player's orchestrator now primes every SFX predecessor voice whose residue sits in the trigger set to `0x010E` by default before capture (the effect-editor test cycle does the same), which operationalized the doc's decisive test and identified the "data-dependent trigger" of the `0x80150AEC` clear previously flagged in [[SPU Voice Engine]].

## Points

- **The `protect_no_music` per-cadence `chan+0x92` clear (PC `0x80150AEC`, previously "data-dependent trigger, not yet identified") fires from a gate in the runtime-patched sound-engine blob at RAM `0x80150AB0` that tests the previous voice's `chan+0x08` low halfword ("last sound state" residue): residue `0` or `0x011B` fires the clear — `beq t0, zero` at PC `0x80150ACC`, `beq t0, 0x011B` at PC `0x80150AD8` — and any other value skips the mute; the gate asks "did some SFX recently use the previous voice?", not "is this a PARITY_AB voice?", so voice 20's (the PARITY_AB driver, `chan_92` initialized to 24576) mute is decided by voice 19's state at trigger time — the orchestrator's savestate left slot pair 1 (voices 18/19) idle so voice 19's residue was `0x0000` and the gate fired, whereas a real PSX reaching `protect_no_music` from an arbitrary gameplay moment almost always carries a non-zero, non-`0x011B` residue in voice 19 (e.g. `cure_4_no_music` allocates voices 18/19 per the doc §6 chan_92 writer table) and would leave voice 20's `chan_92` at 24576 with its sparkle audible in the mix.** — `[S·D·R] 3/3`
  - S: gate PCs `0x80150ACC` (`beq t0, zero`) / `0x80150AD8` (`beq t0, 0x011B`), gate entry `0x80150AB0`, mute action `sh zero, 0x92(s0)` at `0x80150AEC` (runtime-patched / dynamic blob, not statically reachable from SCUS — provenance established in [[SPU Voice Engine]]) — doc §1/§7
  - D: `protect_no_music` orchestrator capture (2026-05-15) — voice 20's gate read voice 19's `chan+0x08` = `0x0000`, mute fired; WRAM values from `research/captures/ram_80150A00_4kb.bin` + 2 MiB `/tmp/pcsx_wram_full.bin` (PCSX-Redux `GET /api/v1/cpu/ram/raw`)
  - R: `smd-player/workspace/orchestrator/effect_capture_orchestrator.lua` (default-on `prime_sfx_residue` block: primes voices 15–21 whose `chan+0x08` low halfword is in the trigger set {0, `0x011B`} to `0x010E`, skips `0x80` music-marker voices — doc §3 option 2 implemented as default capture behaviour) + `effect-editor/state.lua` (`test_prime_sfx_residue = true`) + `effect-editor/commands/workflow.lua` (primes before the test cycle) — validated by the `protect_no_music` orchestrator capture runs
  - src: `research/effect_sound/working_documents/VOICE_19_CHAN_08_SAVESTATE_RESIDUE.md`
- **`chan+0x08` (per-voice channel struct, voice 19 base `0x800375BA`) is a 32-bit word with three fields: bits 31..24 (`chan+0x0B`) = voice-type marker — `0x20` = SFX voice, `0x80` = MUSIC voice — boot-time init, sticky; bits 23..16 (`chan+0x0A`) = pool index (0..7 within the voice-type pool), boot-time init, sticky; and the low halfword (`chan+0x08`/`0x09`) = the "last sound state" residue, written lazily each time an SFX uses the voice — 2026-05-15 dump values: v16/v17 = `0x010E` (prior-SFX residue, paired), v20/v21 = `0x0005` (the live `protect_no_music` pair), v18/v19 = `0x0000` (slot pair 1 never played since the state reset), v22/v23 = `0x0000` (music silenced), v15 fully uninitialized (`c+0x04`/`c+0x08` = `0x00000000`).** — `[S·D·R] 3/3`
  - D: live WRAM dump `research/captures/ram_80150A00_4kb.bin` + 2 MiB `/tmp/pcsx_wram_full.bin` via PCSX-Redux `GET /api/v1/cpu/ram/raw`, `protect_no_music` session (2026-05-15)
  - S: engine init writes `chan+0x08` via the SCUS path `0x80013A50`–`0x80013A6C` (doc §6, citing `PROTECT_CHAN_92_INIT_TABLE_PLAN.md` §3; the same init writes `chan+0x92` at `0x80013D2C` — [[SPU Voice Engine]])
  - R: `smd-player/workspace/orchestrator/effect_capture_orchestrator.lua` prime block encodes the layout (per-voice base table, `+0x0B` marker read, `+0x08`/`+0x09` residue halfword, `0x80` music skip) — validated by the `protect_no_music` orchestrator capture runs
  - src: `research/effect_sound/working_documents/VOICE_19_CHAN_08_SAVESTATE_RESIDUE.md`
- **In the effect-sound channel pool voices 22 and 23 are the music voices (`chan+0x0B` type byte `0x80`, pool #6/#7, residue `0x0000` under music silencing) while voices 16–21 are SFX voices (type `0x20`, pool #0–5) — music silencing in the `*_no_music` parity sessions keeps voices 22/23 idle and does not directly touch the SFX voices' `chan+0x08` state, so the savestate residue of the SFX slot pairs, not the music silencing, is the divergence variable for voice 19.** — `[S·D·R] 3/3`
  - D: live WRAM dump (2026-05-15) — v22 `0x8006 0000` / v23 `0x8007 0000` (MUSIC #6/#7) vs v16–v21 `0x20XX …` (SFX #0–#5)
  - S: same engine-init writer path SCUS `0x80013A50`–`0x80013A6C` (doc §6)
  - R: `smd-player/addons/exmateria_sound/runtime/effect_sound/pool.gd` (slots 6–7 reserved for the music sequencer) + the orchestrator prime block's `0x80` music skip — consistent with the captured split
  - src: `research/effect_sound/working_documents/VOICE_19_CHAN_08_SAVESTATE_RESIDUE.md`

## Notes

(empty — user territory)

## Related

- [[SPU Voice Engine]]
- [[Savestate Residue Voice]]
- [[Dormant Slot Residue]]
- [[Effect Sound Slot Allocator]]
- [[Cure 4 Audio Parity]]
- [[SFX Index]]
