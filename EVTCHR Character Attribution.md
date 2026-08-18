# EVTCHR Character Attribution

The segment→character half of the EVTCHR cinematic problem: which character a `{11}` cinematic-anim frame belongs to. The event stream is the only carrier of that binding (the 137-segment EVTCHR atlas carries no character axis), and attribution runs as a per-unit confidence-tiered replay — `{7F}`-bound units take the bound block's resident segment plus the `{7F}` palette row, a single-resident-segment chunk is attributed to its animating unit, and everything else drops as a false positive. Since the 2026-07-19 redesign the owner is the *resolved SPR*, recovered from the bound CLUT row by palette fingerprint (distance-0 exact match against the flat-store SPR palettes), not the ENTD `special_name` — ENTD tags event-puppets with the real character's `special_name`/`sprite_set` as a decoy. Generic/monster frames pool into a shared per-SPR events store (emit-only), and cutscene-face identities propagate to same-`special_name` sibling slots. The anim→frame half lives in the frame-resolution doc; the load/save/VRAM opcode mechanics are in [[EVTCHR Script VM]].

## Points

- **Per-unit confidence-tiered attribution of cinematic `{11}` frames: `{7F}`-bound → segment = the bound block's resident segment with the `{7F}` palette row (confident); exactly one resident segment → that segment with the full CLUT; ≥2 resident and unbound, or zero resident → DROP as a false positive. Headful-render validated: Ovelia seg-0 @ Orbonne = the real Ovelia (kept), seg-125 @ 429 "Delita's Betrayal" = armored soldiers (dropped), and `delita_6` keeps no frames at all.** — `[R] 1/3`
  - R: `godot-learning/tools/event_asset_derivation.py` (`derive_chunk` tier classification; zero-resident is a hard drop, eliminating the seg-0/no-`{58}` collision) + `godot-learning/tools/test_event_asset_derivation.py` (`DeriveTierTest`: 7F-wins / single-seg / two-resident drop / zero-resident drop / stale-7F fallthrough / clear-disambiguates; 35/35 green 2026-07-18)
  - src: `research/working_documents/EVTCHR_CHARACTER_ATTRIBUTION.md`
- **The correct ambiguity signal is what is *resident at anim time* (modeling `{5A}` clears / `{5B}` resets), not cumulatively `{58}`-loaded: 26 `delita_5` anims fire after a `{5A}` clear with only one segment resident and are correctly kept (multiseg-drops fall 106→80), and an independent clear-aware replay reproduces the shipped `event_asset_map.json` drop counts exactly (ovelia_12: multiseg 101 / zeroseg 7, delita_5: 80, delita_6: 77/12, ramza_2: 50, delita_4: 44, ramza_1: 12, agrias_52: 1).** — `[R] 1/3`
  - R: `godot-learning/tools/event_asset_derivation.py` (currently-resident VRAM-block tracking incl. clears) + `godot-learning/tools/test_event_asset_derivation.py` (`DeriveTierTest.test_clearing_a_block_disambiguates_a_two_load_chunk`)
  - src: `research/working_documents/EVTCHR_CHARACTER_ATTRIBUTION.md`
- **An EVTCHR segment CLUT row is a character's actual palette: matching a `{7F}`-bound (segment, row) against every flat-store SPR palette (`<id>.palette.tga`, symmetric nearest-colour, near-black ignored) yields the owning character as a distance-0 exact match with a wide margin to runner-up — scenario 190 seg48: row0→0x18 Draclau, row1→0x05 Delita, row2→0x24 Vormav, row3→0x0C Ovelia (all dist 0). The resolved SPR (not ENTD `special_name`) is the frame owner, routing the puppets' row-3 frames to `ovelia_12` and the real Delita's row-1 frames to `delita_5`.** — `[R] 1/3`
  - R: `godot-learning/tools/palette_fingerprint.py` (`SegmentOwnerResolver`) + `godot-learning/tools/test_palette_fingerprint.py`; owner routing in `godot-learning/tools/event_asset_derivation.py` (`derive_chunk`, `DeriveSprOwnerTest`)
  - src: `research/working_documents/EVTCHR_CHARACTER_ATTRIBUTION.md`
- **ENTD tags event-puppets with the real character's `special_name`/`sprite_set` as a decoy: scenario 190 (ENTD 284) loads the four-character composite segment 48 and tags the real Delita (`uid5`) **and** two event-puppets (`uid128`/`uid129`) all as `special_name 5` / `sprite_set 0x05`, while the puppets render Ovelia via `{7F}` to CLUT row 3 — so `special_name`-keyed identity mis-attributes composite-sheet frames.** — `[R] 1/3`
  - R: `godot-learning/tools/event_asset_derivation.py` (SPR-owner attribution replaces the special_name axis; `build_spr_to_token` breaks the two 0x16/0x34 decoy collisions by lowest `special_name`) + `godot-learning/tools/test_event_asset_derivation.py` (`DeriveSprOwnerTest`)
  - src: `research/working_documents/EVTCHR_CHARACTER_ATTRIBUTION.md`
- **Expanding the residue template identity from the 11-unit pilot to the whole resolvable story cast (59 tokens total, +48 characters) recovers 633 previously-discarded cinematic `chr` frames across 35 characters under the same tiered replay, with no engine change (top: Alma 71, Dycedarg 54, Algus 54, Mustadio 45+28, Vormav 33, Olan 30); the 13 promoted tokens that yield no frames are all multiseg-unbound or have no cinematic anims.** — `[R] 1/3`
  - R: `godot-learning/tools/derive_template_residue.py` (body sprites from the ENTD dominant `sprite_set`; `--write` merges `template_residue.json` + `template_assets.json` idempotently) + `godot-learning/tools/test_derive_template_residue.py` (logic fixtures + drift invariant: the full resolvable cast stays promoted)
  - src: `research/working_documents/EVTCHR_CHARACTER_ATTRIBUTION.md`
- **Body sprites auto-derive from the ENTD *dominant* `sprite_set` — the sheet the ROM actually loads for the slot, ignoring null/generic-soldier sheets; two slots are flagged `VERIFY` in `template_assets.json`: sn17 Gafgarion loads BARUNA and sn22 Mustadio loads GARU (Gafgarion's own sheet). Both kept faithful (that IS what the ROM loads), and a flagged body taints nothing — a token's EVTCHR event frames are independent of `body_sprite_id` (sourced from the corpus segment atlases).** — `[R] 1/3`
  - R: `godot-learning/tools/derive_template_residue.py` + `godot-learning/tools/test_derive_template_residue.py` (logic fixtures; `VERIFY` flags in `template_assets.json`)
  - src: `research/working_documents/EVTCHR_CHARACTER_ATTRIBUTION.md`
- **482 cinematic anims resolve to real generic/monster SPRs instead of the old `unresolved:anim 255` lump (0x64 Male Knight ×176, 0x63 ×105, 0x72 ×78, 0x65 Female Knight ×35, monsters 0x99/0x97…); they pool per resolved SPR into the shared store `assets/characters/generics/<HEX>/events/chr/NN.tga` + `generic.json` (324 distinct frames across 16 SPRs, empty-composite frames drop on emit) — emit-only: the runtime resolver still returns the flat store for generics, and `event_asset_map.json` carries the `generics` block for provenance.** — `[R] 1/3`
  - R: `godot-learning/tools/event_asset_derivation.py` (`DerivationResult.generic` / `generic_bucket`) + `godot-learning/tools/align_character_templates.py` (`emit_generic_store`)
  - src: `research/working_documents/EVTCHR_CHARACTER_ATTRIBUTION.md`
- **A cutscene character's role often splits across two ENTD puppets sharing one `special_name` — the SPEAKER slot the `{10}` names and a separate ANIMATOR slot that carries the `{7F}` binding + the `{11}` anims — so a speaker-only cutscene-face bridge misses the animator and its frames pool under the ENTD decoy SPR: canonical Elidibs (scn112, ENTD 402), units 128 (speaker) / 129 (animator) both `special_name 119` / `sprite_set 130` → `resolve_sprite(130,151)=0x99` DEMON.SPR, with the human host's 6 seg-123 frames moving `archaic_demon`→`elidibs_face` (`archaic_demon` keeps only the DEMON body). The fix attaches the named speaker's cutscene-face identity to its ENTD `special_name` and propagates it to every sibling slot sharing that `special_name` (the `0xFF` no-special-name sentinel excluded; `elidibs_face/` merges the portrait + host EVTCHR frames).** — `[R] 1/3`
  - R: `godot-learning/tools/event_asset_derivation.py` (`unit_cutscene_faces` gains `uid_slots`; special_name→slug propagation after the direct speaker→slug pass) + `godot-learning/tools/test_event_asset_derivation.py` (`UnitCutsceneFaceSiblingTest` + `DeriveCutsceneFaceSiblingChrTest`)
  - src: `research/working_documents/EVTCHR_CHARACTER_ATTRIBUTION.md`

## Notes

(empty — user territory)

## Related

- [[EVTCHR Script VM]]
- [[Cinematic Sprite Renderer]]
- [[Cinematic Palette Pipeline]]
- [[Unit Anim Opcode]]
- [[ENTD Unit Deployment Table]]
- [[Sprite Set Resolution]]
