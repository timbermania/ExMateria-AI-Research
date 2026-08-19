# PSX Texture Page Register

FFT effect frames carry the complete 16-bit PSX GPU TPage register value in the `texture_page` field at frame offset 0x02–0x03 (VRAM base X/Y, color depth, semi-transparency blend mode) — not a file offset or index. BATTLE.BIN has two rendering code paths that write the primitive's TPage field: the multi-sprite loop copies `texture_page` verbatim, while the single-sprite path ignores it and constructs a partial TPage from the frame's flags word — which is why the frame redundantly carries semi-transparency mode and color depth in both `texture_page` and `flags_byte0`. The godot-learning pipeline bakes the full field (parse_effect.py decodes the complete bit layout) but routes the actual blend passes from the `flags_byte0` copy.

## Points

- **The effect frame's `texture_page` field (offset 0x02–0x03 of the 24-byte frame record) is the complete 16-bit PSX GPU TPage register value — VRAM base X/Y, color depth, semi-transparency mode — NOT a file offset or index; the multi-sprite render loop copies it verbatim into the GPU primitive's TPage field.** — `[S·R] 2/3`
  - S: BATTLE.BIN disassembly 0x801A5650–0x801A5658 (`lhu v0, 0x2(s0)`; `sh v0, 0x16(a0)`), per `research/working_documents/PSX_TPAGE_RESEARCH.md`
  - R: godot-learning/tools/parse_effect.py `parse_frame` (`texture_page = read_u16(data, offset + 2)`, decoded into the frame JSON; the renderer consumes the flags copy instead) — no named validating test
  - src: `research/working_documents/PSX_TPAGE_RESEARCH.md`
- **TPage register (GP0(E1h)) 16-bit layout: bits 0–3 x_base (VRAM X ÷ 64; 0–15 = 0–960 px), bit 4 y_base (rows 0–255 vs 256–511), bits 5–6 semi-transparency blend mode 0–3, bits 7–8 color depth (0 = 4-bit CLUT, 1 = 8-bit CLUT, 2 = 15-bit direct, 3 = reserved), bit 9 24-to-15-bit dither, bit 10 draw-to-display-area, bit 11 extended Y for 2MB arcade VRAM.** — `[S·R] 2/3`
  - S: E019.BIN texture_page 0x00A6 decode + BATTLE.BIN 0x801A5650 verbatim store, per `research/working_documents/PSX_TPAGE_RESEARCH.md`
  - R: godot-learning/tools/parse_effect.py:814–817 (x_base = tp & 0x0F, y_base = tp >> 4 & 0x01, blend = tp >> 5 & 0x03, color_depth = tp >> 7 & 0x03) + effect-editor/core/parser.lua `decode_texture_page` (identical layout, 0=4bit/1=8bit/2=15bit) — no named validating test
  - src: `research/working_documents/PSX_TPAGE_RESEARCH.md`
- **E019.BIN's frame TPage value 0x00A6 decodes to VRAM X base 384 px (TX = 6 × 64), Y base 0 (top half of VRAM), semi-transparency mode 1 (additive B + F), 8-bit CLUT (TP = 1).** — `[S] 1/3`
  - S: E019.BIN frame texture_page value 0x00A6, per `research/working_documents/PSX_TPAGE_RESEARCH.md` (the doc's example line "Bits 7-8: 00 -> 4-bit CLUT" is a decode error: 0x00A6 & 0x0180 = 0x0080, so TP = 1 = 8-bit per the doc's own bit table; the 8-bit reading matches the 0x00A6 example in [[Effect Texture Upload]])
  - R: none — no code or test pins E019's 0x00A6 TPage value (probed godot-learning/src + godot-learning/tests)
  - src: `research/working_documents/PSX_TPAGE_RESEARCH.md`
- **Both rendering code paths write the primitive's TPage field from the same frame data, reading different copies: Path A (multi-sprite loop) copies frame+0x02 verbatim, while Path B (single-sprite) ignores `texture_page` and constructs a partial TPage from the frame flags word (bytes 0–1): `andi v0, v0, 0xE0` keeps bits 5–7 (semi_trans_mode + is_8bpp), `ori v0, v0, 0x08` hardcodes bit 3, `sh v0, 0x16(s3)` stores to the primitive.** — `[S·R] 2/3`
  - S: BATTLE.BIN disassembly 0x801A5650–0x801A5658 (Path A) and 0x801A59E4–0x801A59F4 (Path B), per `research/working_documents/PSX_TPAGE_RESEARCH.md`
  - R: godot-learning/src/effects/EffectParticleRenderer.gd (semi-trans pass routed by frame `semi_trans_on` + `semi_trans_mode` to blend passes 1–4 — the flags copy is the live blend source) — no named validating test
  - src: `research/working_documents/PSX_TPAGE_RESEARCH.md`
- **The branch at 0x801A5534 selects the path by sprite count: sprite_state byte 0x3 > 0 takes Path A (multi-sprite loop), otherwise Path B (single-sprite).** — `[S] 1/3`
  - S: BATTLE.BIN disassembly 0x801A5534, per `research/working_documents/PSX_TPAGE_RESEARCH.md`
  - R: none — no sprite-count branch in godot-learning (single blend-pass routing regardless of sprite count; probed godot-learning/src + godot-learning/tests)
  - src: `research/working_documents/PSX_TPAGE_RESEARCH.md`
- **The effect frame carries semi-transparency mode and color depth redundantly — `flags_byte0` bits 5–6 (semi_trans_mode) + bit 7 (is_8bpp) duplicate `texture_page` bits 5–6 + 7–8 — because Path A and Path B read the different copies, so a mod changing blend mode or color depth must update both fields to keep the two paths consistent.** — `[S·R] 2/3`
  - S: BATTLE.BIN disassembly 0x801A5650 + 0x801A59E4 (the two paths read different copies), per `research/working_documents/PSX_TPAGE_RESEARCH.md`
  - R: godot-learning/tools/parse_effect.py `parse_frame` decodes both copies into the frame JSON (flags_byte0 `semi_trans_mode`/`is_8bpp` + texture_page `blend`/`color_depth`) — no named validating test
  - src: `research/working_documents/PSX_TPAGE_RESEARCH.md`
- **Opcode 5 `set_texture_page` (0x801A2374, `op_set_texture_page`) sets the global `current_texture_page` at 0x801BF000, which some rendering paths use instead of the per-frame field; FUN_800257C8 is a TPage/CLUT construction helper.** — `[S·R] 2/3`
  - S: BATTLE.BIN addresses 0x801A2374, 0x801BF000, 0x800257C8, per `research/working_documents/PSX_TPAGE_RESEARCH.md`
  - R: effect-editor/mips/known_functions.lua (`op_set_texture_page`) + effect-editor/core/parser.lua + godot-learning/tools/parse_effect.py opcode tables ([5] = set_texture_page) — no named validating test
  - src: `research/working_documents/PSX_TPAGE_RESEARCH.md`

## Notes

(empty — user territory)

## Related

- [[PSX GPU Primitives]]
- [[Effect Texture Upload]]
- [[Frameset Header Flags]]
