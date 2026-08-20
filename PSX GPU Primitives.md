# PSX GPU Primitives

The PSX GPU's fixed-function primitive formats used by FFT's custom battle-effect rendering: the tag layout shared by all primitives (24-bit OT link, packet-size byte, RGB, command code, then vertex data), the command codes for lines, polygons, and the particle GT4 quad (0x2C/0x2E) in opaque and semi-transparent variants, the exact sizes of POLY_G3 (28 B) and POLY_GT3 (40 B) with their CLUT/TPAGE fields, the GP0(E1h) blend-mode table that governs semi-transparent compositing, and FFT's own quad-packet builder (`FUN_8007af44`) pinning the POLY_FT4 CLUT (+0x0E) and TPAGE (+0x16) fields from the per-unit render state.

## Points

- **LINE_F2 is a 16-byte primitive (3-byte OT link, 1-byte packet size, R, G, B, command code 0x40, then X0/Y0/X1/Y1 as signed int16), and the GPU will not render it unless the packet-size byte (byte 3, number of 32-bit words after the tag) is set before AddPrim, because AddPrim overwrites only tag bytes 0–2 — FFT's hook code stores a packet size of 6 (LINE_F2 nominally needs only 3).** — `[S] 1/3`
  - S: AddPrim 0x80023BB4 overwrites only bytes 0–2, per `research/key_documents/CUSTOM_EFFECT_HOOKS.md`
  - src: `research/key_documents/CUSTOM_EFFECT_HOOKS.md`
- **PSX GPU primitive command codes: LINE_F2 0x40 / semi-trans 0x42, LINE_G2 0x50 / 0x52, POLY_F3 0x20 / 0x22, POLY_G3 0x30 / 0x32, POLY_GT3 0x34 / 0x36, POLY_F4 0x28 / 0x2A, POLY_G4 0x38 / 0x3A — adding 2 to the opaque code selects the semi-transparent variant.** — `[ ] 0/3`
  - src: `research/key_documents/CUSTOM_EFFECT_HOOKS.md`
- **POLY_GT4 (Gouraud-shaded textured quad) is command code 0x2C opaque / 0x2E semi-trans — the primitive effect particles use: `submit_sprite_to_ordering_table` (0x801A5394) writes 0x2C or 0x2E per the quad_def 0x200 semi-trans bit and copies the R/G/B vertex-color bytes into the packet, where the GPU multiplies texture × vertex / 128 per pixel.** — `[S] 1/3`
  - S: GT4 opcode write and vertex-color copy in `submit_sprite_to_ordering_table` (0x801A5394), per `research/working_documents/PARTICLE_COLORING_SYSTEM.md`
  - R: none — godot-learning renders effect particles through Godot MultiMesh materials, not PSX GT4 opcodes (probed godot-learning/src + godot-learning/tests)
  - src: `research/working_documents/PARTICLE_COLORING_SYSTEM.md`
- **POLY_G3 is 28 bytes (7 words; per-vertex RGB bytes at 0x0C–0x0E, 0x14–0x16, 0x1C–0x1E with padding bytes), not 24 — getting this wrong causes primitives to overlap — and POLY_GT3 is 40 bytes (10 words) with CLUT at offset 0x0E and TPAGE at offset 0x1A, both of which must be set for textured primitives to render.** — `[ ] 0/3`
  - src: `research/key_documents/CUSTOM_EFFECT_HOOKS.md`
- **The semi-transparent blend mode is GPU state in GP0(E1h) bits 5–6: mode 0 = 0.5×Back + 0.5×Front (average), mode 1 = 1.0×Back + 1.0×Front (additive), mode 2 = 1.0×Back − 1.0×Front (subtractive), mode 3 = 1.0×Back + 0.25×Front (subtle additive); per the PSX spec the last draw-mode packet received becomes active, so a GP0(E1h) write made before the ordering table is processed is overwritten by the game's own state.** — `[ ] 0/3`
  - src: `research/key_documents/CUSTOM_EFFECT_HOOKS.md`
- **init_line_g2_header (0x80023CE0) initializes a LINE_G2 primitive header — the GPU-primitive helper that sits alongside AddPrim (0x80023BB4) in the effect rendering code.** — `[S] 1/3`
  - S: init_line_g2_header 0x80023CE0, per `research/key_documents/CUSTOM_EFFECT_HOOKS.md`
  - src: `research/key_documents/CUSTOM_EFFECT_HOOKS.md`
- **In semi-transparency mode 0 (0.5×Back + 0.5×Front), fading the foreground RGB to 0 leaves 0.5×background — a 50%-darkened background — so a fade-to-zero effect darkens rather than fading to transparent.** — `[D] 1/3`
  - D: observed with the fade-effect hook in the running game (doc 2026-04-16)
  - src: `research/key_documents/CUSTOM_EFFECT_HOOKS.md`
- **ABR mode 1 (additive) is a per-channel 8-bit display-space add: `dst = clamp(dst + texel·gouraud/128)` — the ROM's per-vertex gouraud is the only modulation, and a pure `texel_srgb·gouraud/128` composite (base gouraud 128, zero per-mesh constants) reproduces the PSX formation orb rim byte-close.** — `[D·R] 2/3`
  - D: formation-screen oracle, sstate1 Ramza/cell 0, pcsx :8080, 256×240 1:1 capture (2026-07-18) — a pure `texel_srgb·gouraud/128` composite (no ×2.2, no pow(1.4), no box_add_level) byte-closes the PSX orb rim
  - R: `godot-learning/src/ui3/shaders/formation_orb_rim_fold.gdshader` + `formation_box_fold.gdshader` (ADR-0074 colour-math: `ALBEDO = texel·gouraud/128`) + `tools/check_no_pow_in_fold.py` (static guard: no sRGB→linear pow reachable from `compositor_layer` shaders)
  - src: `research/working_documents/FORMATION_ORB_ADDITIVE_COLORSPACE.md`
- **FFT's textured-quad GP0 packet packs CLUT at +0x0E and TPAGE at +0x16: `FUN_8007af44` stores `render_state[+0x4] | (caller_flags & 0x60)` to `packet[+0x16]` at `0x8007b2e0` and `render_state[+0x6]` to `packet[+0x0e]` at `0x8007b2f8` — so the per-unit palette pin (CLUT) comes from a separate render-state field than the sprite-atlas selector (`unit[+0x0E]` → `render_state[+0x4]` → TPAGE).** — `[S·D] 2/3`
  - S: `FUN_8007af44` pack sites `0x8007b2e0` (TPAGE) / `0x8007b2f8` (CLUT), POLY_FT4 layout `+0x0e` CLUT / `+0x16` TPAGE (`clut_upload_decode.md` §V11, BATTLE.BIN disassembly)
  - D: `probe_unit_palette_v12.py` exec-BP at `0x8007b2f8` — 906 hits in 5 s, 3 distinct render-state ptrs, per-frame (TPAGE, CLUT) = (0x0015, 0x78C0) / (0x0016, 0x78C1) / (0x0005, 0x78CC) (`orbonne_prayer_cinematic.sstate`, 2026-06-26)
  - R: none — no GP0 quad-packet builder / POLY_FT4 pack site in godot-learning (probed `godot-learning/src/`, `godot-learning/tests/`; only a POLY_FT4 doc-comment in `UnitShadow.gd`)
  - src: `research/working_documents/scenario_1_captures/clut_upload_decode.md`

## Notes

(empty — user territory)

## Related

- [[Ordering Table & AddPrim]]
- [[Effect Texture Upload]]
- [[Embedded MIPS Effect Code]]
- [[Particle Coloring System]]
