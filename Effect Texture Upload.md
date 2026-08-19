# Effect Texture Upload

Verified procedure for staging and uploading custom-effect textures (e.g. 3D spheres) from a PCSX-Redux Lua session: palette and pixel data are staged in the Effect Buffer at 0x801C2500, pixel data is pushed to VRAM with LoadImage (FUN_800248fc) after a DrawSync, indexed-color CLUTs go through the game's upload_clut (FUN_800926d8) shadow-buffer path, and calls into FFT code use hardcoded JAL encodings. The on-disk texture section of an effect file is the last section (from the header's texture pointer to EOF) and always holds both 256-color BGR555 palettes, a 4-byte VRAM-upload/depth header at +0x400, and indexed pixel data at +0x404; at effect init (FUN_801a0e80) both palettes are unconditionally uploaded to CLUT lines 0x0C/0x0D with the palette choice made per-sprite (bit 4 of the sprite definition), and the effect editor's live texture editing patches the staged RAM from a savestate captured before the state-2 upload (8bpp textures only). E001's extracted texture BMP maps to texture UVs with a fixed (+6, +24) offset. The 2026-04-16 working document additionally decodes the on-disk frame section (24-byte frame records with PS1 VRAM control words and signed UV dimensions) and restates the palette/image layout; the frame-section points gained disassembly + reimplementation evidence on 2026-08-17 (the palette/image restatement remains low-evidence).

## Points

- **Custom effect texture data is staged in the Effect Buffer region 0x801C2500–0x801DF000 — palette 1 at 0x801C2500 (512 B, 256 colors × 2 B BGR555), palette 2 at 0x801C2700 (512 B), pixel data at 0x801C2900 (~115 KB) — which is unused during the spell charge animation before the effect image loads.** — `[S·D] 2/3`
  - S: Effect Buffer addresses 0x801C2500/0x801C2700/0x801C2900/0x801DF000, per `research/key_documents/PCSX_BRIDGE.md`
  - D: test_working_sphere.sh / test_textured_sphere.sh probes (doc 2026-04-16)
  - src: `research/key_documents/PCSX_BRIDGE.md`
- **All VRAM uploads (texture pixels and CLUT palettes) must go through LoadImage (FUN_800248fc); FUN_800926d8 is a palette buffer copy, not a VRAM upload.** — `[S·D] 2/3`
  - S: symbols FUN_800248fc (LoadImage), FUN_800926d8, per `research/key_documents/PCSX_BRIDGE.md`
  - D: test_textured_sphere.sh probe (doc 2026-04-16)
  - src: `research/key_documents/PCSX_BRIDGE.md`
  - ⚠ SUPERSEDED (2026-08-14) by: Indexed-color palette uploads in embedded effect code must use the game's upload_clut (FUN_800926d8) shadow-buffer path (DAT_800e4ea4, flag DAT_800995ec=1) — LoadImage does not take effect for indexed-color CLUTs (it still works for pixel data).
- **DrawSync(0) at 0x800246d4 must be called before a VRAM upload to wait for pending GPU operations to finish.** — `[S·D] 2/3`
  - S: DrawSync address 0x800246d4, per `research/key_documents/PCSX_BRIDGE.md`
  - D: test_textured_sphere.sh probe (doc 2026-04-16)
  - src: `research/key_documents/PCSX_BRIDGE.md`
- **LoadImage takes a 4×int16 RECT {x, y, w, h}; verified upload geometry is CLUT row 1 (0, 492, 256, 1) 512 B, CLUT row 2 (0, 493, 256, 1) 512 B — VRAM (0, 492–493) corresponds to CLUT=0x7B00 — and a 256×256 texture (384, 0, 128, 256) 65536 B, because 8-bit indexed textures pack 2 pixels per VRAM halfword (VRAM width = pixel width / 2).** — `[S·D] 2/3`
  - S: LoadImage RECT usage at 0x800248fc, per `research/key_documents/PCSX_BRIDGE.md`
  - D: test_textured_sphere.sh probe (doc 2026-04-16)
  - src: `research/key_documents/PCSX_BRIDGE.md`
- **JAL instructions to 0x80xxxxxx FFT code must be encoded as 0x0C000000 | ((target & 0x0FFFFFFF) >> 2); the naive 0x0C000000 + target/4 leaves bit 25 set and corrupts the JAL opcode (000011) into BGTZ (000111), yielding a `bgtz $zero, offset` that never branches — e.g. jal 0x800248fc = 0x0C00923F, jal 0x800246d4 = 0x0C0091B5.** — `[S·D] 2/3`
  - S: target addresses 0x800248fc/0x800246d4, per `research/key_documents/PCSX_BRIDGE.md`
  - D: test_textured_sphere.sh probe (doc 2026-04-16)
  - src: `research/key_documents/PCSX_BRIDGE.md`
- **Indexed-color effect palette uploads must use the game's upload_clut (FUN_800926d8, JAL 0x0C0249B6; args palette_ptr, clut_line, sub_palette_idx, full_upload), which writes the shadow buffer DAT_800e4ea4[clut_line×0x100+idx] (via FUN_80092620) and sets DAT_800995ec=1 so the actual VRAM transfer happens later in the rendering pipeline — LoadImage (FUN_800248fc) does not take effect for the CLUT (it still works for pixel data) — with effect CLUT line 0x0C → VRAM (0, 492) → CLUT 0x7B00 (Palette 1) and 0x0D → VRAM (0, 493) → CLUT 0x7B40 (Palette 2).** — `[S] 1/3`
  - S: symbols FUN_800926d8, FUN_80092620, DAT_800e4ea4, DAT_800995ec, per `research/key_documents/CUSTOM_EFFECT_HOOKS.md`
  - src: `research/key_documents/CUSTOM_EFFECT_HOOKS.md`
- **For 8-bit indexed textures (TPAGE TP=1), texture U = (VRAM_X − TPAGE_base_X) × 2 and V = VRAM_Y (no multiplier on Y); texture page width is 256/128/64 pixels in 4/8/16-bit mode respectively (each 16-bit VRAM halfword holds 2 pixels in 8-bit) — verified on the E001 star: VRAM (391, 95) with TPAGE 0x00A6 (base X = 6×64 = 384) gives UV (14, 95).** — `[D] 1/3`
  - D: E001 star UV verification in the PCSX-Redux VRAM viewer (doc 2026-04-16)
  - src: `research/key_documents/CUSTOM_EFFECT_HOOKS.md`
- **CLUT field encoding (16-bit, at primitive offset 0x0E): bits 0–5 = VRAM X ÷ 16, bits 6–14 = VRAM Y (e.g. VRAM (0, 492) = (492 << 6) | 0 = 0x7B00); TPAGE field encoding (16-bit, at offset 0x1A): bits 0–3 = TX (page X = TX×64), bit 4 = TY (page Y = TY×256), bits 5–6 = ABR (semi-transparency mode 0–3), bits 7–8 = TP (color mode: 0=4-bit, 1=8-bit, 2=15-bit) — e.g. an 8-bit texture at VRAM (384, 0) with ABR=1 gives TX=6, TPAGE = 6 + 0 + 32 + 128 = 0x00A6.** — `[ ] 0/3`
  - src: `research/key_documents/CUSTOM_EFFECT_HOOKS.md`
- **The on-disk texture section of an effect file (pointed to by the header's texture pointer at offset 0x28) is laid out as palette 1 (512 B = 256 × 16-bit colors) at +0x000, palette 2 (512 B) at +0x200, and 8-bit pixel indices (variable size) at +0x404.** — `[S] 1/3 CONTESTED`
  - S: texture section layout at header pointer 0x28, per `research/key_documents/CUSTOM_EFFECT_HOOKS.md`
  - S: same palette/pixel offsets (+0x000/+0x200/+0x404) plus a 4-byte upload header at +0x400, per disassembly of FUN_801a0e80 and 0x801a0ed8–0x801a0ef8, in `research/key_documents/TEXTURE_AND_PALETTE_FORMAT.md`
  - src: `research/key_documents/CUSTOM_EFFECT_HOOKS.md`
- **E001.BIN's extracted texture BMP maps to texture UVs with a fixed offset — U = BMP_X + 6, V = BMP_Y + 24 (the offset varies per effect file and must be calibrated against a feature known in both BMP and VRAM) — verified on the star: BMP (18, 95) → UV (24, 119), matching the VRAM-derived (396 − 384) × 2, 119.** — `[D] 1/3`
  - D: E001 star BMP↔UV cross-check against the PCSX-Redux VRAM viewer (doc 2026-04-16)
  - src: `research/key_documents/CUSTOM_EFFECT_HOOKS.md`
- **The texture section is the last section in effect files: it starts at header[0x24] (texture_ptr) and extends to EOF (texture_ptr + texture_size = file_size).** — `[S] 1/3 CONTESTED`
  - S: FUN_801a0e80 (effect texture initialization) reads texture_ptr from effect_base[0x24], per `research/key_documents/TEXTURE_AND_PALETTE_FORMAT.md`
  - src: `research/key_documents/TEXTURE_AND_PALETTE_FORMAT.md`
- **The 4-byte header at +0x400 in the texture section holds the VRAM upload parameters: bytes 0–2 combine little-endian (low | mid<<8 | high<<16) into the VRAM Y coordinate and byte 3 is the depth flag (0 = 8bpp, non-zero = 4bpp); the pixel upload uses a fixed VRAM width of 0x40 bytes (64 pixels) in 8bpp mode or 0x80 bytes (256 pixels) in 4bpp mode, not a width taken from the header.** — `[S] 1/3`
  - S: header byte reads and fixed-width selection at 0x801a0ed8–0x801a0ef8, per `research/key_documents/TEXTURE_AND_PALETTE_FORMAT.md`
  - src: `research/key_documents/TEXTURE_AND_PALETTE_FORMAT.md`
- **Both palettes are ALWAYS uploaded to VRAM at effect initialization (FUN_801a0e80), regardless of whether the effect uses both: palette 1 (texture data + 0x000) to CLUT line 0x0C (VRAM 0x7B00) and palette 2 (texture data + 0x200) to CLUT line 0x0D (VRAM 0x7B40) via upload_clut (FUN_800926d8); the texture data pointer is stored in the global DAT_801bbf80 and the pixel data is uploaded with LoadImage to VRAM X=0x180 (384).** — `[S] 1/3`
  - S: initialization flow at FUN_801a0e80 (texture_ptr read, DAT_801bbf80 store, both upload_clut calls, LoadImage at X=0x180), per `research/key_documents/TEXTURE_AND_PALETTE_FORMAT.md`
  - src: `research/key_documents/TEXTURE_AND_PALETTE_FORMAT.md`
- **Palette selection is per-sprite, not runtime: bit 4 (0x10) of the sprite definition word in the Frames section selects palette 2 (CLUT address 0x7B40 + sub-palette) over palette 1 (0x7B00 + sub-palette), with the sub-palette (0–15) in the lower 4 bits — a static per-sprite choice made in the sprite definition data, not by script opcodes (verified at 0x801a5664).** — `[S] 1/3`
  - S: per-sprite CLUT selection at 0x801a5664, per `research/key_documents/TEXTURE_AND_PALETTE_FORMAT.md`
  - S: clut_y = (quad_def & 0xf) + 0x7B00 (4bpp) / + 0x7B40 (8bpp) computed in `submit_sprite_to_ordering_table` (0x801a5394), per `research/working_documents/PARTICLE_COLORING_SYSTEM.md`
  - src: `research/key_documents/TEXTURE_AND_PALETTE_FORMAT.md`
- **The effect-file corpus splits on the texture depth flag: 0x00 = 8bpp (64-byte VRAM width) in ~220 files (e.g. E001.BIN) and 0x01+ = 4bpp (128-byte VRAM width) in ~71 files (e.g. E121.BIN) — ~220 of the 291 DATA-format files are 8bpp.** — `[S] 1/3`
  - S: bit-depth distribution table, per `research/key_documents/TEXTURE_AND_PALETTE_FORMAT.md`
  - src: `research/key_documents/TEXTURE_AND_PALETTE_FORMAT.md`
- **Effect texture data is raw VRAM-ready format, NOT PSX TIM: there is no TIM header or magic (0x10 0x00 0x00 0x00) — the data is laid out for direct upload.** — `[S] 1/3`
  - S: NOT TIM section, per `research/key_documents/TEXTURE_AND_PALETTE_FORMAT.md`
  - src: `research/key_documents/TEXTURE_AND_PALETTE_FORMAT.md`
- **Each 256-color effect palette is organized as 16 sub-palettes of 16 colors (32 bytes each, offsets 0x000–0x1FF), matching the PSX 4bpp convention where the upper nibble of a pixel byte selects the sub-palette; in E001.BIN sub-palettes 0–12 hold color data and 13–15 are all zeros.** — `[S] 1/3`
  - S: sub-palette structure and E001.BIN observation, per `research/key_documents/TEXTURE_AND_PALETTE_FORMAT.md`
  - src: `research/key_documents/TEXTURE_AND_PALETTE_FORMAT.md`
  - src: `research/working_documents/FFT_VFX_COMPLETE_TECHNICAL_REFERENCE.md`
- **Effect palettes are BGR555 with the STP (semi-transparency) bit at bit 15 (bits 14–10 blue, 9–5 green, 4–0 red): 0x0000 = black (transparent by default), 0x8000 = opaque black (STP=1), 0xFFFF = white with STP.** — `[S] 1/3`
  - S: BGR555 bit layout and special values, per `research/key_documents/TEXTURE_AND_PALETTE_FORMAT.md`
  - src: `research/key_documents/TEXTURE_AND_PALETTE_FORMAT.md`
  - src: `research/working_documents/FFT_VFX_COMPLETE_TECHNICAL_REFERENCE.md`
- **Even 8bpp effects can carry both palettes (depth mode only changes pixel decoding, not palette structure): E040.BIN is an 8bpp 44×256 texture with palette 1 at 378 non-zero bytes and palette 2 at 249 non-zero bytes, letting different sprites in one effect use different color schemes without duplicating pixel data.** — `[S] 1/3`
  - S: E040.BIN dual-palette analysis, per `research/key_documents/TEXTURE_AND_PALETTE_FORMAT.md`
  - src: `research/key_documents/TEXTURE_AND_PALETTE_FORMAT.md`
- **Live texture editing requires the savestate to be captured at the start of effect-system state 2 (0x801a1920), before FUN_801a0e80 uploads the texture from RAM to VRAM: on reload, state 2 re-executes and uploads the patched texture — capturing at state 3 (0x801a1964) fails because VRAM already contains the old texture.** — `[S·R] 2/3`
  - S: state-2 texture upload at 0x801a1920/0x801a1938 (FUN_801a0e80) vs state 3 (0x801a1964), per `research/key_documents/TEXTURE_AND_PALETTE_FORMAT.md`
  - R: `effect-editor/capture.lua` (CASED2_START_ADDRESS 0x801a1920 — breakpoint armed at state-2 start, before the texture upload)
  - src: `research/key_documents/TEXTURE_AND_PALETTE_FORMAT.md`
- **The effect editor's live texture editing (BMP export → Krita → reimport) supports 8bpp textures only (~220 of the 291 DATA-format files); 4bpp textures (two pixels per byte across 16 sub-palettes) are not supported by the import path.** — `[S·R] 2/3`
  - S: 8bpp-only limitation (~220 of 291 files), per `research/key_documents/TEXTURE_AND_PALETTE_FORMAT.md`
  - R: `effect-editor/commands/texture_ops.lua` (BMP import guard: depth_flag != 0 → "4bpp not supported")
  - src: `research/key_documents/TEXTURE_AND_PALETTE_FORMAT.md`

- **The 2026-04-16 working document defines the on-disk frame section (header[0x00] "Frames") as: 1 byte at 0x00 = "number of halfwords before offsets" (affects the parse offset), followed by uint16 offsets to each frame set relative to section_start + 4; each frame set starts with a 4-byte header, then an array of 24-byte frame records.** — `[S·R] 2/3`
  - S: 4-byte frameset header + 24-byte frame stride confirmed at the runtime read sites — `frame_count` loaded at frameset+0x02 (0x801aa3ec), frame pointers built as `frameset_ptr + 4 + i×24` (0x801aa434–0x801aa448), BATTLE.BIN disassembly, per `research/working_documents/FRAMESET_HEADER_FLAGS_ANALYSIS.md`
  - R: godot-learning/tools/parse_effect.py (FRAME_SIZE = 24, 4-byte frameset header, group-offset frameset parse) + godot-learning/src/effects/EffectData.gd (loads baked frames.json / frameset_groups.json); no named validating test
  - src: `research/working_documents/FFT_VFX_COMPLETE_TECHNICAL_REFERENCE.md`
  - src: `research/working_documents/FRAMESET_HEADER_FLAGS_ANALYSIS.md`
- **The working document decodes each 24-byte frame record as: 4 bytes of PS1 VRAM control flags (byte 0: bits 0–3 palette ID for 4bpp, bits 5–6 semi-transparency mode 0=50% blend / 1=additive / 2=subtractive / 3=25% additive, bit 7 color depth 0=4bpp / 1=8bpp; byte 1: bit 1 semi-transparency on/off, bit 4 signed UV width → horizontal flip, bit 5 signed UV height → vertical flip; bytes 2–3 = PS1 VRAM load coordinates, unused in emulation), uint8 top_left_u/top_left_v at 0x04/0x05, signed int8 uv_width/uv_height at 0x06/0x07, then eight int16 screen-space quad corners at 0x08–0x16; negative UV dimensions flip the texture along that axis (used for mirroring sprites).** — `[S·R] 2/3`
  - S: flag bytes read per Frame at frame+0x00 in submit_sprite_to_ordering_table (0x801a5394) — 0x801a55f4 (lhu frame+0x00), 0x801a55fc (andi 0x200 = semi_trans_on), 0x801a565c/0x801a5664 (andi 0xf = palette_id bits 0–3), BATTLE.BIN disassembly, per `research/working_documents/FRAMESET_HEADER_FLAGS_ANALYSIS.md`
  - R: godot-learning/tools/parse_effect.py `parse_frame` (palette_id = flags_byte0 & 0x0F, semi_trans_mode = flags_byte0 >> 5 & 0x03, is_8bpp = flags_byte0 & 0x80, semi_trans_on = flags_byte1 & 0x02, signed UV at 0x04–0x07, four signed int16 corners at 0x08–0x16) + godot-learning/src/effects/EffectParticleRenderer.gd; no named validating test
  - src: `research/working_documents/FFT_VFX_COMPLETE_TECHNICAL_REFERENCE.md`
  - src: `research/working_documents/FRAMESET_HEADER_FLAGS_ANALYSIS.md`
- **The working document defines the palette/image section (header[0x24], its "Palette & Image") as: an 8bpp palette (256 × uint16) at 0x000–0x3FF and a 4bpp palette (16 sub-palettes × 16 colors) at 0x200–0x3FF, colors BGR555 with bit 15 = semi-transparency flag; image data at 0x400 with a uint32 dimension word at 0x400 (0x01010000 = 256×256) and pixel data from 0x404 (4bpp: 2 pixels/byte, low nibble = left pixel; 8bpp: 1 pixel/byte direct palette index).** — `[ ] 0/3`
  - R: none — this note's S points read the +0x400 word as VRAM upload parameters (bytes 0–2 = VRAM Y, byte 3 = depth flag) with fixed pixel-upload widths; the doc's dimension-word reading conflicts at 0/3
  - src: `research/working_documents/FFT_VFX_COMPLETE_TECHNICAL_REFERENCE.md`

## Notes

(empty — user territory)

## Related

- [[E001.BIN Memory Mapping]]
- [[Effect Execution Model]]
- [[Effect File Format]]
- [[Particle Coloring System]]
