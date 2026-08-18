# Lua Effect Editor

The `effect-editor/` package: a PCSX-Redux Lua tool (imgui UI over live memory) that is the second E###.BIN editor alongside the Godot Effect Studio. It parses an effect with `core/parser.lua` + `core/field_schema.lua`, edits subsystems through per-subsystem tabs, and repacks the file with `ee_save_bin`. The Structure tab is display-only (all ten header pointers are section-layout derived); Curves, Time Scale, Script, Particles, Sequences, Timeline, Color, Sound, and Camera tabs have write-back; texture replacement is a wholesale ImageMagick-driven swap; FEDS opcode editing exists in the Sound tab but is deferred out of the editability scope.

## Points

- **The effect-editor Structure tab is display-only — it has no input widgets: all ten header section pointers are section-layout derived, and the editor recomputes/rewrites them on structure mutations (add/remove time-scale section shifts downstream sections and rewrites time_scale_ptr at header[0x14]; add/delete frame, frameset, or emitter reindexes and repoints the offset tables), after which ee_save_bin repacks the .BIN.** — `[S·R] 2/3`
  - S: ten derived section pointers, per `research/key_documents/master_parser.py`
  - R: `effect-editor/ui/structure_tab.lua` (zero input widgets) + `effect-editor/ui/time_scale_tab.lua` (add/remove section) + `effect-editor/core/structure_manager.lua` (time_scale_ptr) + `effect-editor/commands/file_ops.lua` (ee_save_bin repack; no automated test)
  - src: `research/working_documents/E_BIN_FIELD_EDITABILITY_INVENTORY.md`
- **The effect-editor Curves tab edits the 156-value animation curves procedurally — generators Constant, Linear, Ease-in, Ease-out, S-Curve, Sine, Triangle, Sawtooth, Pulse plus Invert/Reverse/Reset transforms — and "Apply This Curve" writes the value array straight into live memory.** — `[R] 1/3`
  - R: `effect-editor/ui/curve_generators.lua` (all nine generators + Invert/Reverse) + `effect-editor/ui/curves_tab.lua` (apply-to-memory; no automated test)
  - src: `research/working_documents/E_BIN_FIELD_EDITABILITY_INVENTORY.md`
- **The effect-editor Script tab supports insert/delete/duplicate of 16-bit effect-script instructions and whole-script "Convert to 1-phase / 3-phase" templates (1-phase = single entry-0x00 script; 3-phase = root + for-each entry at 0x24), recomputing branch offsets on edit.** — `[R] 1/3`
  - R: `effect-editor/ui/script_tab.lua` (insert/delete/duplicate + Convert-to-phase buttons; no automated test)
  - src: `research/working_documents/E_BIN_FIELD_EDITABILITY_INVENTORY.md`
- **Effect texture replacement in the effect-editor is a whole-section swap rather than field editing: commands/texture_ops.lua quantizes a source image to the effect's indexed palette via ImageMagick and rewrites the texture section.** — `[R] 1/3`
  - R: `effect-editor/commands/texture_ops.lua` (ImageMagick integration, wholesale texture import; no automated test)
  - src: `research/working_documents/E_BIN_FIELD_EDITABILITY_INVENTORY.md`
- **The effect-editor Sound tab already has full opcode insert/delete/parameter editing for the FEDS stream even though FEDS editing is deliberately deferred out of the editability scope.** — `[R] 1/3`
  - R: `effect-editor/ui/sound_tab.lua` (opcode insert/delete/param controls; no automated test)
  - src: `research/working_documents/E_BIN_FIELD_EDITABILITY_INVENTORY.md`

## Notes

(empty — user territory)

## Related

- [[Effect File Format]]
- [[Effect Studio Editability]]
- [[Effect Texture Upload]]
- [[FEDS Sound Definition Format]]
- [[Effect Animation Sequences]]
