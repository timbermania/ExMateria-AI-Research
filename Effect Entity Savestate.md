# Effect Entity Savestate

sstate-derived entity state: lifting the effect-sound entity linked list (rooted at `DAT_80032A50`) out of PCSX-Redux .sstate RAM dumps and using it to seed smd-player's effect-sound seeder. Phase 0 verification against 6 no_music effect savestates (2026-05-16) established the invariant 3-entity list shape (two silent drivers at the head, the active effect entity at the tail), the silent drivers being bit-identical and never advancing (gate ≥ 0 skips the per-entity body at PC 0x80014BC4), the bit-15 gate arming happening after sstate capture (gate 2 → −32766), the arming IRQ re-seeding slot+0x58 from wrap_reset, the N-decrement entity_acc gap between sstate and the canonical cad=0 entry, and the warmup rule (subtract entity_budget until acc < budget; verified on all 6 sessions). smd-player's `_seed_entities_from_state` now applies lift + warmup + arm from the sstate seed, replacing the per-session `--entity-acc-cad0` back-calc and the hardcoded `_silent_entities` literals.

## Points

- **All 6 captured no_music effect savestates hold the same 3-entity linked list under `DAT_80032A50`: head 0x80038B80 (silent driver, channel_count 2) → 0x800387F0 (silent driver, channel_count 2) → 0x800370E0 (active effect entity, channel_count 8) → null, walked via the +0x00 next pointer — shape and addresses invariant across cure, cure_3, cure_4, protect, protect_hit, reraise.** — `[S·D·R] 3/3`
  - S: DAT_80032A50 list head; entity addresses 0x80038B80 / 0x800387F0 / 0x800370E0 — Ghidra symbol + .sstate RAM dump (doc Q1/Q3)
  - D: 6 .sstate captures (cure_no_music, cure_3_no_music, cure_4_no_music, protect_no_music, protect_no_music_hit, reraise_no_music) walked by `research/tools/sstate_entity_extract.py` (2026-05-16)
  - R: `smd-player/addons/exmateria_sound/runtime/effect_sound/play_sound.gd::_seed_entities_from_state` + `smd-player/workspace/orchestrator/extract_entity_state.py` (sstate → JSON seed) — validated by the `probe_per_entity_iter` / `probe_per_entity_pass` pairs (`probe_validation_manifest.py`)
  - src: `research/effect_sound/working_documents/ENTITY_ACC_SAVESTATE_VERIFICATION.md`
- **The two silent-driver entities are bit-identical across all 6 sstate captures (ent0 @ 0x80038B80: gate=1 chs=2 acc=0x10000 sub=1 pass=1 measure=0 slot_58=3 wrap=48; ent1 @ 0x800387F0: gate=1 chs=2 acc=0xDA00 sub=48 pass=17 measure=1 slot_58=0 wrap=48) and never advance at runtime: gate ≥ 0 makes the bgez at PC 0x80014BC4 skip the per-entity body, so they are bit-exact between sstate and the cad=1 runtime probe.** — `[S·D·R] 3/3`
  - S: bgez pass-gate PC 0x80014BC4 (doc Q4); entity addresses + field values from the sstate RAM dump
  - D: 6 sstate captures vs `probe_entity_dump.jsonl` cad=1 rows, protect_no_music (2026-05-16) — silent entities bit-exact, active entity diverging
  - R: `smd-player/addons/exmateria_sound/runtime/effect_sound/play_sound.gd::_seed_entities_from_state` seeds `_silent_entities` verbatim (no warmup for channel_count ≤ 2; code comment cites this doc's Q2) — `probe_per_entity_iter` / `probe_per_entity_pass` pairs
  - src: `research/effect_sound/working_documents/ENTITY_ACC_SAVESTATE_VERIFICATION.md`
- **FFT's effect-load path arms bit 15 (0x8000) of slot+0x10 (gate) after sstate capture: protect_no_music's sstate gate is 2 (pre-arm) while the runtime probe sees −32766 (= 0x8002, bits 15+1, post-arm) — the sstate-time gate cannot drive the runtime catchup without re-arming.** — `[D·R] 2/3`
  - D: protect_no_music sstate gate 2 vs probe_entity_dump gate −32766 (2026-05-16)
  - R: `smd-player/addons/exmateria_sound/runtime/effect_sound/play_sound.gd` — `_ENTITY_GATE_ARM_BIT = 0x8000`; `_seed_entities_from_state` ORs bit 15 into gate and stores it in `_cure_slot_10` (default −32766) — `probe_per_entity_iter` / `probe_per_entity_pass` pairs
  - src: `research/effect_sound/working_documents/ENTITY_ACC_SAVESTATE_VERIFICATION.md`
- **The arming IRQ re-seeds slot+0x58 (post_cadence_flag) from wrap_reset: on protect_no_music sstate slot_58 = 0 while the post-arm probe shows slot_58 = 48 (= wrap_reset), matching the LAB_80014DD0 re-arm path that fires when post-cadence_flag hits 0 and the catchup loop completes.** — `[S·D] 2/3`
  - S: LAB_80014DD0; slot+0x58 read sites 0x80014D70 / 0x80014DD0 — Ghidra (doc Q5)
  - D: protect_no_music sstate (slot_58=0) vs probe_entity_dump cad=1 (slot_58=48) (2026-05-16)
  - R: none — post_cadence_flag / slot+0x58 not modeled in smd-player's entity_state.gd (catchup subset only; probed smd-player, godot-learning, fft-sound-driver)
  - src: `research/effect_sound/working_documents/ENTITY_ACC_SAVESTATE_VERIFICATION.md`
- **The sstate active-entity entity_acc is exactly N outer-decrement steps ahead of the canonical cad=0 entry, because the savestate was captured at an arbitrary moment after effect load: protect_no_music sstate 0xB200 is 1 step ahead (IRQ A no-fire: 0xB200 − 0x6600 = 0x4C00 = cad=0 entry; IRQ B one-fire: 0x4C00 − 0x6600 + 0x10000 = 0xE600 = the cad=1 probe value); cure_no_music sstate 0xF800 is 2 steps ahead of the back-calc cad=0 init 0x2C00 (0xF800 → 0x9200 → 0x2C00).** — `[D] 1/3`
  - D: protect_no_music + cure_no_music sstate vs probe_entity_dump / known back-calc (2026-05-16)
  - R: none — the capture-timing gap itself is not modeled; smd-player's `_seed_entities_from_state` applies the consequence (warmup) instead (probed smd-player, godot-learning, fft-sound-driver)
  - src: `research/effect_sound/working_documents/ENTITY_ACC_SAVESTATE_VERIFICATION.md`
- **Warmup rule (sstate → cad=0 init): repeatedly subtract entity_budget (0x6600) from the sstate active-entity entity_acc until acc < budget; subcounter / pass / measure stay unchanged (no sub-loop fires during the warmup). Per-session: cure 0xF800→0x2C00 (2 decrements, matches the prior hardcoded 0x2C00), cure_3 0xCE00→0x0200 (2; anomalously low — the cad=0-entry IRQ won't fire, flagged for the validator), cure_4 0x9A00→0x3400 (1, matches the play_sound.gd:127 comment), protect 0xB200→0x4C00 (1, matches probe back-calc 0xE600 − 0x9A00), protect_hit 0xC800→0x6200 (1), reraise 0xAA00→0x4400 (1).** — `[D·R] 2/3`
  - D: 6 sstate captures (2026-05-16); matches the prior `probe_entity_dump` back-calc values
  - R: `smd-player/addons/exmateria_sound/runtime/effect_sound/play_sound.gd::_seed_entities_from_state` warmup loop (`while entity_acc >= entity_budget: entity_acc -= entity_budget`) for the channel_count > 2 entity — validated by the `probe_per_entity_iter` / `probe_per_entity_pass` pairs (`probe_validation_manifest.py`)
  - src: `research/effect_sound/working_documents/ENTITY_ACC_SAVESTATE_VERIFICATION.md`
- **Active-entity discriminator: an entity is active iff channel_count > 2 (silent drivers always channel_count == 2; in current data equivalently address 0x800370E0 or last in the list). The sstate-time gate value is not a reliable discriminator because the bit-15 effect-load arming hasn't happened yet.** — `[D·R] 2/3`
  - D: 6 sstate captures (2026-05-16)
  - R: `smd-player/addons/exmateria_sound/runtime/effect_sound/play_sound.gd::_seed_entities_from_state` (`if channel_count > 2:` → active, else `_silent_entities`) — `probe_per_entity_iter` / `probe_per_entity_pass` pairs
  - src: `research/effect_sound/working_documents/ENTITY_ACC_SAVESTATE_VERIFICATION.md`

## Notes

(empty — user territory)

## Related

- [[SPU Voice Engine]]
- [[Effect Sound Timing]]
- [[Effect Sound Slot Allocator]]
- [[Cure 4 Audio Parity]]
- [[Savestate Residue Voice]]
