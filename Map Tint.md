# Map Tint

The "Affected Units" palette track (Track 0) tints units and map/terrain from the same keyframe data, but its map path is not a terrain vertex-colour applier: a live-Ghidra xref pass (2026-07-11) found no ×8 vertex-colour scaling and no third tint sink — map tint routes through the two colour appliers (palette CLUT contents or the screen tint quad), never terrain vertex RGB. Script opcodes 0x75/0x76 can disable/re-enable the map tint via bit 2 of effect_render_flags.

## Points

- **Track 0 (Affected Units) tints both units and map/terrain from the same keyframe data; the map/terrain RGB is scaled ×8 (R<<3, G<<3, B<<3) relative to the unit values.** — `[S] 1/3`
  - S: `advance_affected_units_palette_track` at 0x801A45C8, per `research/key_documents/COLOR_TRACK_INTERPOLATION.md`
  - src: `research/key_documents/COLOR_TRACK_INTERPOLATION.md`
  - ⚠ SUPERSEDED (2026-08-14) by: the affected-units palette track advances through advance_affected_units_palette_track at 0x801A41A0; 0x801A45C8 is advance_screen_color_track
  - ⚠ SUPERSEDED (2026-08-16) by: no ×8 map vertex-colour scaling exists — a live-Ghidra xref pass (2026-07-11) found exactly two tint sinks (palette CLUT `color_tint_blend_apply @0x8008F710` and screen `screen_tint_apply @0x80090840`); the map is tinted via CLUT contents or the screen quad, never terrain vertex RGB
- **The scaled map tint RGB is written to register DAT_800f5b58 via FUN_80090dec() → FUN_800e8190(99, &rgb) and applied to terrain polygon vertex colors during map rendering in FUN_8012d2b4.** — `[S] 1/3`
  - S: symbols FUN_80090dec, FUN_800e8190, FUN_8012d2b4 and register DAT_800f5b58, per `research/key_documents/COLOR_TRACK_INTERPOLATION.md`
  - src: `research/key_documents/COLOR_TRACK_INTERPOLATION.md`
  - ⚠ SUPERSEDED (2026-08-16) by: no map vertex-colour applier exists — map tint routes through the two colour appliers (palette CLUT staging `dynamic_clut_view_strip @0x800E4EA4` or the screen tint quad), per the live-Ghidra xref pass (2026-07-11)
- **Map tint applies only while (effect_render_flags & 2) == 0; script opcode 0x75 disables map tint (flag |= 2) and opcode 0x76 re-enables it (flag &= ~2).** — `[S] 1/3`
  - S: opcodes 0x75/0x76 and effect_render_flags bit 2, per `research/key_documents/COLOR_TRACK_INTERPOLATION.md`
  - src: `research/key_documents/COLOR_TRACK_INTERPOLATION.md`

## Notes

(empty — user territory)

## Related

- [[Color Track Interpolation]]
- [[Combat Color Appliers]]
