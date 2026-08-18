# PSX GPU Primitives

The PSX GPU's fixed-function primitive formats used by FFT's custom battle-effect rendering: the tag layout shared by all primitives (24-bit OT link, packet-size byte, RGB, command code, then vertex data), the command codes for lines and polygons in opaque and semi-transparent variants, the exact sizes of POLY_G3 (28 B) and POLY_GT3 (40 B) with their CLUT/TPAGE fields, and the GP0(E1h) blend-mode table that governs semi-transparent compositing.

## Points

- **LINE_F2 is a 16-byte primitive (3-byte OT link, 1-byte packet size, R, G, B, command code 0x40, then X0/Y0/X1/Y1 as signed int16), and the GPU will not render it unless the packet-size byte (byte 3, number of 32-bit words after the tag) is set before AddPrim, because AddPrim overwrites only tag bytes 0–2 — FFT's hook code stores a packet size of 6 (LINE_F2 nominally needs only 3).** — `[S] 1/3`
  - S: AddPrim 0x80023BB4 overwrites only bytes 0–2, per `research/key_documents/CUSTOM_EFFECT_HOOKS.md`
  - src: `research/key_documents/CUSTOM_EFFECT_HOOKS.md`
- **PSX GPU primitive command codes: LINE_F2 0x40 / semi-trans 0x42, LINE_G2 0x50 / 0x52, POLY_F3 0x20 / 0x22, POLY_G3 0x30 / 0x32, POLY_GT3 0x34 / 0x36, POLY_F4 0x28 / 0x2A, POLY_G4 0x38 / 0x3A — adding 2 to the opaque code selects the semi-transparent variant.** — `[ ] 0/3`
  - src: `research/key_documents/CUSTOM_EFFECT_HOOKS.md`
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

## Notes

(empty — user territory)

## Related

- [[Ordering Table & AddPrim]]
- [[Effect Texture Upload]]
- [[Embedded MIPS Effect Code]]
