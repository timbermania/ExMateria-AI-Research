# Effect Studio Editability

The Godot Effect Studio's editable-coverage state over E###.BIN bytes: which subsystems have write-back wired into `studio_save`, tracked per-subsystem by an editability manifest against a sample effect (E019.BIN, 52,100 bytes, 14 emitters). Editable (write-back wired) is 9,570 of 52,100 bytes (18.4%) of the sample; the remainder is read-only display (targetable gaps: texture 33,796 B, frames 5,024 B, animation curves 2,400 B, sequences 776 B, emitters 112 B, effect flags 24 B, particle-system header 16 B), ROM-fixed/derived, or deferred. Emitter parameters are authored as semantic two-axis groups (randomness × evolution) rather than raw technobabble fields.

## Points

- **The Godot Effect Studio authors emitter parameters as semantic two-axis groups — most parameters vary along a randomness axis (each particle rolls a value between min and max at spawn) and an evolution axis (the min–max range slides start→end over the emitter's active window, shaped by that parameter's assigned curve) — via EmitterParamRows (edit-cell/enum/int rows) mapped onto EmitterChannel, with byte-exact save via write_effect_emitters.py + EffectEmitterSaver layered into studio_save (ADR-0089 LANDED).** — `[S·R] 2/3`
  - S: two-axis structure, EmitterParamRows→EmitterChannel shape, and E019.BIN emitter region 0x01724–0x021DC (2,744 B = 14 emitters × 196 B), per `godot-learning/docs/adr/0089-emitter-parameters-author-as-semantic-two-axis-groups-edited-at-the-reference.md` + `godot-learning/src/effects/studio/editability_manifest.json` (feature/effect-studio-authoring)
  - R: godot-learning/src/effects/studio/EffectEmitterSaver.gd + tools/write_effect_emitters.py, validated by tests/EffectEmitterSaverTest.gd (tests/run_all_tests.sh) + tools/test_write_effect_emitters.py (feature/effect-studio-authoring)
  - src: `research/working_documents/EFFECT_STUDIO_COVERAGE.md`
- **As of 2026-08-17, studio write-back (studio_save) covers 9,570 of 52,100 bytes (18.4%) of E019.BIN: particles/emitters partially (2,548 of 2,744 B), script bytecode (64 B), time scale (600 B), timeline header (6 of 12 B), particle timeline (1,920 B), colour tracks (palette/field tint 1,800 B + screen tint/gradient 900 B), sound timeline (342 B), camera timeline partially (1,062 of 1,306 B), and FEDS sound defs (328 B) — each through a dedicated Saver delegating to a byte-exact tools/write_effect_*.py writer.** — `[R] 1/3`
  - R: godot-learning/src/effects/studio/{EffectEmitterSaver,EffectScriptSaver,EffectTimeScaleSaver,EffectTimelineHeaderSaver,EffectParticleTimelineSaver,EffectPaletteSaver,EffectSoundSaver,EffectCameraSaver}.gd + tools/write_effect_*.py, validated by tools/test_write_effect_*.py + tests/run_all_tests.sh (feature/effect-studio-authoring)
  - src: `research/working_documents/EFFECT_STUDIO_COVERAGE.md`

## Notes

(empty — user territory)

## Related

- [[Effect File Format]]
- [[Particle Emitter Format]]
- [[FEDS Sound Definition Format]]
