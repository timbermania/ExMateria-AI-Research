# Custom Effect Hooks

Custom GPU rendering hooks for FFT's spell-charge effects, implemented as raw MIPS code injected through a PCSX-Redux Lua session: the first two instructions of render_spell_charge (0x801B0FFC) or render_spell_charge_lines (0x801B1C04) are patched to `j HOOK_ADDR; nop`, and the hook code (CODE_ADDR 0x80150AB0, working data at a non-overlapping DATA_ADDR) runs a bounded frame loop that stages primitives and submits them via AddPrim, then re-executes the two original instructions and jumps to original+8 to hand back to the game; unit positioning goes through the anchor system (unit_id at +0x12 of the context at g_effect_state_ptr, resolved with get_camera_position), not the unreliable static g_caster_pos/g_target_pos.

## Points

- **The spell-charge effect renderers are render_spell_charge (func_id 0x01) at 0x801B0FFC and render_spell_charge_lines (func_id 0x04) at 0x801B1C04; the hook replaces each function's first two instructions with `j HOOK_ADDR; nop`, the hook code (CODE_ADDR 0x80150AB0) then runs a bounded frame-counter loop rendering custom primitives per frame, and on completion resets the counter, re-executes the two original instructions, and jumps to original+8 (e.g. 0x801B1C0C for render_spell_charge_lines).** — `[S] 1/3`
  - S: 0x801B0FFC (render_spell_charge), 0x801B1C04 (render_spell_charge_lines), handoff jump 0x801B1C0C, per `research/key_documents/CUSTOM_EFFECT_HOOKS.md`
  - src: `research/key_documents/CUSTOM_EFFECT_HOOKS.md`
- **The effect rendering globals are g_primitive_buffer_index at 0x801BC090 and g_effect_state_ptr at 0x801BC098 (pointer to the current effect rendering state); the hook stages its working data — frame counter at DATA_ADDR+0x00 followed by per-primitive 16-byte slots — at a DATA_ADDR strictly beyond the code region (e.g. 0x80150E00 after 600 bytes of code from 0x80150AB0), because overlapping code and data makes the hook overwrite its own data, causing crashes or no rendering.** — `[S] 1/3`
  - S: g_primitive_buffer_index 0x801BC090, g_effect_state_ptr 0x801BC098, per `research/key_documents/CUSTOM_EFFECT_HOOKS.md`
  - src: `research/key_documents/CUSTOM_EFFECT_HOOKS.md`
- **The anchor system is how effects find their unit: the spell-charge rendering context pointed to by g_effect_state_ptr (0x801BC098) holds phase_state (int32) at +0x08 and the anchor unit_id (byte) at +0x12 (the caster for spell-charge effects); the hook reads that unit_id, resolves the world position with get_camera_position (0x8008C410, which returns unit_struct+0x40 via get_unit_struct_from_id 0x8007A6E4; X/Y/Z int16 at +0x00/+0x02/+0x04), and re-reads it each frame so the effect tracks the unit — the static g_caster_pos/g_target_pos at 0x801B925C–0x801B9268 are not reliably set (in testing they held the target's position when the caster's was expected).** — `[S] 1/3`
  - S: g_effect_state_ptr 0x801BC098, get_camera_position 0x8008C410, get_unit_struct_from_id 0x8007A6E4, g_caster_pos/g_target_pos 0x801B925C–0x801B9268, per `research/key_documents/CUSTOM_EFFECT_HOOKS.md`
  - src: `research/key_documents/CUSTOM_EFFECT_HOOKS.md`
- **MIPS J instructions in the hook handoff must encode the word address, not the byte address: encoding = 0x08000000 | ((target / 4) & 0x03FFFFFF) — the handoff jump to render_spell_charge_lines + 8 (0x801B1C0C) is 0x0806C703; the naive byte-based encoding 0x08006C83 mis-jumps to 0x8001B20C in the kernel area.** — `[S] 1/3`
  - S: handoff target 0x801B1C0C (original+8 of render_spell_charge_lines 0x801B1C04), per `research/key_documents/CUSTOM_EFFECT_HOOKS.md`
  - src: `research/key_documents/CUSTOM_EFFECT_HOOKS.md`

## Notes

(empty — user territory)

## Related

- [[Embedded MIPS Effect Code]]
- [[GTE World-to-Screen Transform]]
- [[Ordering Table & AddPrim]]
- [[Effect Texture Upload]]
