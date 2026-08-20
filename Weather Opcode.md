# Weather Opcode

Event opcode `{3C}` "Weather" (bytecode `[3C, Strength, Unknown]`) is an inline latch: the dispatcher arm at `0x80144998` packs both operand bytes into the global `DAT_80173f68 = Strength | (Unknown<<8)`, and the per-frame map-graphics tick at `0x801436b0` uses the Unknown byte as the active gate (`Unknown=0` cancels → global −1) and Strength as the index into the 10-entry type table `DAT_801696e4`. Strength selects the storm power only (distinct animation id → fall-velocity/scatter triple); rain-vs-snow is a per-map flag word `DAT_800b6698` (bit0 snow, bit1 indoor) that adds +5 to the index. The full decode + render RE closed 2026-07-02 with every claim confirmed live against the Orbonne rain battle (D1–D11), and the Godot port (decode gate + velocity triple) is unit-tested in `godot-learning`; the particle rendering itself is tracked in [[Weather Rain Renderer]].

## Points

- **Event opcode `{3C}` Weather (bytecode `[3C, Strength, Unknown]`) is an inline latch — the dispatcher arm at `0x80144998` stores the 16-bit packed operand (s3, a 16-bit LE read of the operand bytes via `event_bytecode_reader_c`) into `DAT_80173f68 = Strength | (Unknown<<8)` and returns with no inline render; `DAT_80173f64`, which a prior handoff misread as the weather latch, is the delay-slot-shifted arm of `{4F}` Set Daytime.** — `[S·D·R] 3/3`
  - S: dispatch arm `0x80144998` (`sh s3,DAT_80173f68`), shared operand head `0x80143d0c`, globals `DAT_80173f64`/`DAT_80173f68` (`battle_disassembly.txt`, BATTLE.BIN base `0x80067000`)
  - D: latch BP `0x801449a4` caught `s3=0x0103, s2=0x03` entering the Orbonne battle (2026-07-02)
  - R: `godot-learning/src/scenarios/ScenarioDecode.gd` `weather()`/`WeatherIntent` + `godot-learning/tests/ScenarioWeatherTest.gd` `_test_decode_active_gate`
  - src: `research/working_documents/WEATHER_OPCODE_3C_INVESTIGATION.md`
- **The per-frame map-graphics tick consumes the latch at `0x801436b0`: `& 0x0f00` (the Unknown byte) is the active gate — `Unknown=0` cancels the weather by clearing the global to `−1` — and `& 0x000f` = Strength indexes the 10-entry type table `DAT_801696e4`; the firing frame clears the Unknown bit, so the started animation self-perpetuates until a cancel `{3C}` or map change.** — `[S·D·R] 3/3`
  - S: consumer block `0x801436b0`–`0x80143728`, type table `0x801696e4` (`battle_disassembly.txt`)
  - D: live D3/D9 — fire site `0x80143724` fired exactly once over ~120 frames of rain (latch=1, fire=1); `s3=0x0103` → Unknown=1/Strength=3 and post-fire `DAT_80173f68` back at `0xFFFF` (−1) (2026-07-02)
  - R: `godot-learning/src/scenarios/ScenarioDecode.gd` `weather()` active gate (`unknown != 0 and strength >= 2`) + `tests/ScenarioWeatherTest.gd` `_test_decode_active_gate`
  - src: `research/working_documents/WEATHER_OPCODE_3C_INVESTIGATION.md`
- **The Strength enum is 0/1 = clear/cancel, 2 = Rain/Snow, 3 = Thunderstorm/Snowstorm, 4 = Strong Thunder/Snowstorm: rain ids `0x7c/0x55/0x7d`, snow ids the +5 entries `0x7e/0x53/0x7f` of `DAT_801696e4`, and clear indices 0/1/5/6 all map to id `0x54` but are unreachable because `Unknown=0` short-circuits to the cancel path — Strength sets the power only; the rain-vs-snow type is a separate per-map flag.** — `[S·D·R] 3/3`
  - S: type table `DAT_801696e4` = `54h,54h,7Ch,55h,7Dh,54h,54h,7Eh,53h,7Fh` (`battle_disassembly.txt`)
  - D: D2 latch BP `0x801449a4` `s3=0x0103` → Strength=3 (2026-07-02)
  - R: `godot-learning/src/scenarios/ScenarioDecode.gd` strength decode + `ScenarioWeather.gd` strength-keyed system, validated by `tests/ScenarioWeatherTest.gd` `_test_decode_active_gate`
  - src: `research/working_documents/WEATHER_OPCODE_3C_INVESTIGATION.md`
- **Strength selects a distinct animation-command id whose case handler writes a fall-velocity / vertical-scatter triple into `DAT_800fa6a4/a6a8/a6ac` before calling the particle spawner `FUN_800eeecc(a0)`: rain (layer `a0=0x90`) 2→(3,5,8) id `0x7c`, 3→(5,6,9) id `0x55`, 4→(5,9,18) id `0x7d`; snow (layer `a0=0x8f`) 2→(1,1), 3→(1,4), 4→(3,16) — the triple scales monotonically with Strength, so heavier storms fall faster/harder, not denser.** — `[S·D·R] 3/3`
  - S: case handlers `0x800eedc8`+ (Strength-3 rain case `0x800eee20`), dispatcher `FUN_800eeaf8` jump table over ids `0x53–0x8c`, globals `0x800fa6a4/a6a8/a6ac` (`battle_disassembly.txt`)
  - D: D5 live `fa6a4/fa6a8/fa6ac = 5/6/9` for Strength-3, bit-exact to the static case (2026-07-02)
  - R: `godot-learning/src/scenarios/ScenarioDecode.gd` `weather_velocity_triple` (2→(3,5,8), 3→(5,6,9), 4→(5,9,18)) + `tests/ScenarioWeatherTest.gd` `_test_decode_velocity_triple`
  - src: `research/working_documents/WEATHER_OPCODE_3C_INVESTIGATION.md`
- **Rain-vs-snow (and ignore/indoor) is a per-map flag word `DAT_800b6698` — bit0 = snow, bit1 = ignore/indoor — set at map/battle init (writers `0x800ee5a4`, `0x800f33c4–3488`), with the renderer adding +5 to the resolved index to pick the snow animation; the weather graphics object and its VRAM texture are allocated per-map at load, so poking the snow bit on a running rain map no-ops (the consumer picks snow id `0x53` and calls the dispatcher, but the particle arrays stay rain-format).** — `[S·D] 2/3`
  - S: +5 test in the consumer `0x801436b0`, flag bit-tests in "Get Weather" `0x8018e660`, writers `0x800ee5a4`/`0x800f33c4` (`battle_disassembly.txt`)
  - D: live poke test — `DAT_800b6698 |= 1` + re-armed latch `0x0103` on the active rain battle no-opped (2026-07-02, GAP 5)
  - R: none — snow variant not present in godot-learning (out of scope per the port scoping; probed `godot-learning/src/scenarios/`, `godot-learning/tests/`)
  - src: `research/working_documents/WEATHER_OPCODE_3C_INVESTIGATION.md`
- **The elemental-weather path reads script variable `0x23` (Weather, clamped [0,4]), which is written ONCE at battle-init by `0x8008ea0c` (`FUN_8013b644(0x23, clamped)`) from a per-battle config byte — not by the `{3C}` opcode — so the render path (`DAT_80173f68` ← `{3C}`) and the elemental path (var `0x23` ← battle-init) are sibling systems that agree by construction; the "Get Weather" routine `0x8018e660` returns the effective weather 0–7 for elemental damage: bit1 (indoor) → 0, Strength <2 passes through, bit0 (snow) adds +3.** — `[S·D] 2/3`
  - S: battle-init writer `0x8008ea0c`, "Get Weather" `0x8018e660`, variable channel `FUN_8013b590`/`FUN_8013b644` (`battle_disassembly.txt`)
  - D: D5 trace resolution — var `0x23` set at battle-init, no `{3C}` bridge exists (2026-07-02)
  - R: none — script variable `0x23` / elemental weather not present in godot-learning (probed `godot-learning/src/`, `godot-learning/tests/`)
  - src: `research/working_documents/WEATHER_OPCODE_3C_INVESTIGATION.md`

## Notes

(empty — user territory)

## Related

- [[Weather Rain Renderer]]
- [[Event Opcode Catalog]]
- [[Map State Selection]]
- [[Map Tint]]
- [[Background Opcode]]
