# Prayer Screen Tint

The Orbonne prayer darken→untint (scenario 1, "God, please help us sinful") is decoded end-to-end against three PCSX savestates plus live breakpoints (2026-06-30; Q3 closed 2026-07-05): it is a CLUT palette-content fade, not a full-screen quad and not per-unit color modulation. The view k=3 palette (staging `0x800e54a4`) is frozen while its `thr` byte is <4 ("dark held" = the view switched off), then re-armed by the `{33}` Color Field / `{32}` Color Unit color commands that the per-frame processor `FUN_801495e0` re-applies through the view's private `FUN_8008f710` arming (never the `{1A}` screen-blend `FUN_80090840`, which is inert here); the gate opens when the prayer `Display Message` advances — an event-script barrier, not a message-state byte — and the per-frame dirty `LoadImage` flush `FUN_80092f98` uploads staging view k to VRAM row Y=494+k. The transform is a uniform per-channel additive offset with 5-bit clamp (map −4, sprite −8, additive field∘unit composition), and Godot's `ScenarioColorTint` mode-4 `{33}`/`{32}` is faithful to it apart from the ramp trajectory (smooth lerp vs the PSX's 32-step DDA; endpoints and duration match).

## Points

- **The Orbonne prayer darkening is a darkening of the CLUT palette *contents* in VRAM — NOT a semi-transparent full-screen quad and NOT a per-unit color modulation: everything that samples the affected palettes (unit sprites and the non-unit map background) renders dark, and the scene untints when the palettes are restored; the `{4D}` screen-fade subsystem is dormant throughout (fade mode/level/step `0x800960e4`/`0x800961b0`/`0x8009610c` hold `0x34/0/2` and the whole `0x80096000–0x80096200` block is byte-identical across all three states).** — `[S·D·R] 3/3`
  - S: fade-global block `0x80096000–0x80096200`, sprite CLUT word `0x78c0` → VRAM (0,483) (BATTLE.BIN, doc §3/§5)
  - D: three ground-truth savestates `orbonne_prayer_tint_00/01/02.sstate` (2026-06-30): display-256×240 mean luma 2.65 / 2.92 / 4.96 (02 ≈1.7× brighter than 01), VRAM CLUT (0,483) luma 7.42 / 7.42 / 14.47; 2029 RAM addresses byte-equal 00↔01, and the 01→02 untint happens AFTER the prayer text clears
  - R: `godot-learning/src/scenarios/ScenarioApply.gd` `color_field` (palette-space field tint broadcast to every unit CLUT + the map palette — explicitly not a screen-space filter), validated by `godot-learning/tests/ScenarioColorFieldTest.gd` `_test_broadcast_darken_reaches_all_units_and_map`
  - src: `research/working_documents/scenario_1_captures/prayer_screen_tint_quad_decode.md`
- **No per-unit color state moves with the untint: the sprite quads (region `0x800bb800+`) carry Gouraud RGB neutral `0x80,0x80,0x80` in all three states and only their CLUT field swaps `0x78c0 → 0x78cc → 0x78c0` — a double-buffer flip of the palette upload, NOT the `unit[+0x13F]` alternate-palette XOR (that byte is 0 for every occupied roster slot in all states; roster base `0x800b7308`, stride `0x440`, full byte-diff shows only position/scale/facing/anim fields moving); the VRAM diff 01→02 touches Y=483/484 (the sprite-CLUT selection rows) and Y=494/497/498 (the staging rows) together, X 0..223.** — `[S·D] 2/3`
  - S: quad region `0x800bb800+`, roster `0x800b7308` (stride `0x440`), `unit[+0x13F]` +0x13F alt-palette path (`FUN_8013e65c`) (BATTLE.BIN, doc §4/§5)
  - D: 3-state full-2MB RAM + VRAM dumps (`probe_prayer_tint_ramdiff.py`, `probe_prayer_tint_vram.py`, 2026-06-30)
  - R: none — the sprite-CLUT double-buffer flip (0x78c0↔0x78cc) not present in godot-learning (probed `godot-learning/src/` + `godot-learning/tests/` for `0x78c0`/`0x13f`/`483`; the `ScenarioVM.gd` Reset-Palette comment is the closest analogue)
  - src: `research/working_documents/scenario_1_captures/prayer_screen_tint_quad_decode.md`
- **The per-frame color ticker `0x800917b0` walks ~10 independent "views" (outer loop `0x80091c60`; metadata block `0x800995f4 + k·0x982` — +0 type, +2 enable, +3 phase, +4 sub-counter, +5 thr, +6 mode; accumulator `0x80099676 + k·0x982` (256 entries × 7 B); CLUT staging dest `0x800e4ea4 + k·0x200`; shared curve table `0x800956e4`), and the prayer's darkening palette is view k=3 (staging `0x800e54a4`); a view whose `thr` byte < 4 is frozen entirely by the ticker's per-frame gate (`0x80091ca4`: `sltiu thr,4` → branch-skip) — so "dark held" is the view switched off, not a curve holding it dark each frame (k=3's thr at `0x8009b27f` is 0 while dark-held, 4 at untint).** — `[S·D] 2/3`
  - S: `0x800917b0`/`0x80091c60`/`0x80091ca4`, metadata `0x800995f4`, k=3 state `0x8009b27f`/`0x8009b27d`/`0x8009b2fc`, staging `0x800e54a4` (BATTLE.BIN, doc §8.5)
  - D: bit-exact offline RAM-diff against the saved `prayer_tint_ram/{00,01,02}.ram` (2026-06-30, `analyze_prayer_view_state_offline.py`): k=3 accumulator packs R5G5B5 → `0x800e54a4`, all 16 entries match in both states; moving views k=0/k=3/k=4 table (thr 0/0/4, phase 0/0/0x20, curve-idx 0→0x23/0x27/0x17)
  - R: none — the per-view `thr` enable gate / 0x982 view blocks not present in godot-learning (probed `godot-learning/src/` + `godot-learning/tests/`)
  - src: `research/working_documents/scenario_1_captures/prayer_screen_tint_quad_decode.md`
- **The palette curve is armed by `FUN_8008f710` — writer `0x8008ff04` stores the curve-index bytes `(target_ch − current_ch) + 0x1f` (row `0x1f` of the curve table is all-zero = no-op, so `0x27` = +8 brighten, `0x17` = −8 darken) and writer `0x8008ffd8` stores the view's `thr` byte — and NEVER by `FUN_80090840` (the `{1A}` screen-blend track at `0x800a1b58`, byte-identical 01↔02 at the untint); the arming re-runs every frame from the per-frame color-command processor `FUN_801495e0` (list gated by `FUN_801479ac`), which re-applies queued color commands through `FUN_800933c4` — at the untint the callers were `ra=0x8014964c` (8×, one per occupied slot 0,1,2,3,4,5,6,12, a0=4,a1=4) and `ra=0x80145174` (`FUN_80093170` ← the `{33}` sub-dispatcher `0x8013ed78`), while the event dispatcher `0x8013e8e8` fired 0× — the opcodes only enqueue/target; the per-frame loop does the work.** — `[S·D] 2/3`
  - S: `FUN_8008f710`, `0x8008ff04`, `0x8008ffd8`, `FUN_801495e0`/gate-open `0x80149644`, `FUN_801479ac`, `FUN_800933c4`, `FUN_80145174`, sub-dispatcher `0x8013ed78` (BATTLE.BIN, doc §8.6)
  - D: live Write-BP on the darkening view's private state + Exec-BP with `ra`/arg capture at the untint (2026-06-30, `probe_prayer_palette_arming_bp.py`, isolated PCSX port 8089)
  - R: none — the per-frame color-command re-application (`FUN_801495e0` loop) not present in godot-learning (probed `godot-learning/src/scenarios/` + `godot-learning/tests/`; `ScenarioColorTint` ticks its own ramp instead)
  - src: `research/working_documents/scenario_1_captures/prayer_screen_tint_quad_decode.md`
- **The untint is gated on event-script sequencing, not a message-state byte: while the prayer `Display Message` holds, the script is blocked at chunk [42] and the 21-sprite-actor-slot gate (`FUN_801479ac`, called per slot from `FUN_801495e0`) stays closed; when the message advances, the untint color command's type/target opens the gate for exactly slots 0,1,2,3,4,5,6,12 (once, at the untint frame only) → `FUN_800933c4` arms the ramp once → the ticker converges the palettes; live trigger with state 01 loaded and CIRCLE taps: CLUT(0,483) luma holds 7.42 (taps 0–9), 8.19 (tap 10, ramp begins), 13.53 (tap 11 = UNTINT; tap# varies run-to-run 11/12/14/16), with the per-frame writer `0x800921e8` running continuously (≈4992 hits over the window) and the applier `FUN_80090840` NOT re-called at the untint (0 hits).** — `[S·D·R] 3/3`
  - S: `FUN_801495e0`/`FUN_801479ac`, gate-open site `0x80149644` (BATTLE.BIN, doc §6.4/§8.6-tail)
  - D: `probe_prayer_vram_upload_bp.py` (Exec-BP on the gate-open site: opens only at the untint frame, exactly the 8 occupied slots, a0=4,a1=4) + `probe_prayer_untint_trigger.py` CIRCLE-tap luma trace (2026-06-30)
  - R: `godot-learning/src/scenarios/ScenarioVM.gd` — the `{33}` Color Field dispatches after the prayer `Display Message` in bytecode order, and the field ramp completes even while the VM is halted on the dialog (the field tick in `_tick_once` runs before the `_running` gate), validated by `godot-learning/tests/ScenarioColorFieldTest.gd` `_test_ramp_ticks_outside_halt_gate` + `_test_real_chunk_field_ops_do_not_halt`
  - src: `research/working_documents/scenario_1_captures/prayer_screen_tint_quad_decode.md`
- **The per-view offsets compose additively — the map band is field-only (`{33}` −4), a unit's sprite is its own `{32}` tint over the field (field(−4) ∘ unit(−4) = the sprite band's −8), and a separate view (k=4) inverts at +8 — and scenario 1's untint arm is the post-prayer `[52]` Color Unit Color=4 RGB=(0,0,0) Time=4 + `[53]` Color Field Blend=4 RGB=(0,0,0) Time=4, bracketed by `[41]` Map Darkness Blend=4 RGB=(20,31,31) Time=4 (darken) and `[44]` Map Darkness RGB=(0,0,0) Time=4.** — `[S·D·R] 3/3`
  - S: `godot-learning/assets/scenarios/scenario_1_chunk.json` indices [41]/[44]/[52]/[53] + the `{33}`/`{32}` applier `FUN_8008f710` (doc §6.3/§11.2)
  - D: per-view offsets measured across the three savestates — map band Y=480/494 (−4,−4,−4), sprite Y=483/497 (−8,−8,−6), unit Y=484/498 (+8,+8,+6) (2026-06-30; live CLUT-diff re-confirmed 2026-07-05, `probe_q3_clut_diff.py`/`probe_q3_golden_pairs.py`)
  - R: `godot-learning/src/scenarios/ScenarioApply.gd` `color_field` (field broadcast composed per-unit with the active `{32}` tint; pure-identity result dropped), validated by `godot-learning/tests/ScenarioColorFieldTest.gd`
  - src: `research/working_documents/scenario_1_captures/prayer_screen_tint_quad_decode.md`

## Notes

(empty — user territory)

## Related

- [[Map Darkness Opcode]]
- [[Color Unit Opcode]]
- [[Color Tint Luma Modes]]
- [[Map Tint]]
- [[Cinematic Palette Pipeline]]
- [[Display Message Opcode]]
- [[Reveal Opcode]]
- [[Combat Color Appliers]]
