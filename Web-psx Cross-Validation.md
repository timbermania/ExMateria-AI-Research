# Web-psx Cross-Validation

An external cross-validation pass over this vault by **web-psx**, a PlayStation
emulator written as a programmable PSX debugger and FFT reverse-engineering
harness (JIT-to-JavaScript CPU/GTE, WebGPU GPU, WebAudio SPU; a Lean 4 sibling
emulator, lean-psx, is the semantics reference). Our evidence standard is an
instrumented emulator running the retail game off the real disc image: a claim is
*graded* by re-deriving it two ways — statically out of `BATTLE.BIN` /
`SCUS_942.21` / the `EVENT` and `EFFECT` overlays on the disc, and dynamically by
running the game with a shadow reimplementation *beside* it and byte-diffing every
write (`tools/fxsim.ts` for `EFFECT/*.BIN`, `tools/eventsim.ts` for the event VM),
plus GPU packet capture at `DrawOTag` and a hardware-verified CPU/GTE. Every
number below is reproducible from a disc image and a BIOS with no hand-driving,
so these claims grade to this vault's **D** as well as **S**.

**Every incorporable claim in this vault was run against our emulator and disc
before this note was written**, which is why the balance below is what it is: the
material here is good, and it corrected us in more places than we corrected it.
The single largest item is a claim of yours we had **twice** got wrong on our own
side and have now adopted — see the sprite-pose point below. This note carries
(a) the confirmations, (b) a summary of the corrections landed as `⚠ SUPERSEDED`
sub-bullets on the specific points, (c) a proposal for a mutual camera
cross-validation, and (d) tooling notes for raising `[S] 1/3` to `[D]`.
Contributed 2026-08-19; contact is via the pull request that carries this note.

## Points

- **The biggest single correction runs in this vault's favour: a `.SEQ` animation resolves at the **quarter** turn out of sub-tables E and F, exactly as *Sprite Cardinal Pose Selection* says, and web-psx had it wrong twice — `DAT_800a7786` is the camera term of the pose angle, and that identification was right too.** — `[S·D] 2/3`
  - S: `raw = (s16[800a7786h] + facing) & 0xfff`, `sixteenth = raw >> 8` into `actor+0x6e`, `quarter = raw >> 10` into `actor+0x6c`, at `BATTLE.BIN+1edcc..1edf4` — the two are one reading of one angle, `quarter == sixteenth >> 2`. A **standing** actor (request code 2) resolves at the sixteenth out of A and B; everything else at the quarter out of E and F, and the EVTCHR branch takes F as well
  - D: over 7,170 drawn actors on 9 recorded tapes — `actor+0x6c == actor+0x6e >> 2` in 7,170 of 7,170, standing id `== A[sixteenth]` in 735 of 735, animation parity `== E[quarter]` in 6,364 of 6,364, animation mirror `== F[quarter]` in **6,434 of 6,435** against `== B[sixteenth]` in 6,358 of 6,434 — **76 records wrong, every one at sixteenth 8**. The fix is merged. What did not survive is only the *characterisation* of `DAT_800a7786` as a 4-way rotation on `0x400` steps: `{−0x100, 0, 0x100, 0x200}` spans `0x300` and shifts the sixteenth by −1/0/+1/+2 (web-psx `docs/combat-tables.md` [combat.animation.quarter], `docs/scene-viewport.md` [viewport.sprite.quarter])
  - src: external contribution — web-psx `docs/combat-tables.md` [combat.animation.quarter]
- **The 46-opcode effect script table is confirmed and adopted outright — substituted for our 14, the static walk goes from 400 files to **401 of 401** with 0 unknown, and the entry check gets better rather than worse.** — `[S] 1/3`
  - S: ours (14 opcodes) walks 400 with 1 unknown, 266 of 267 entry indices resolving inside their own file's code, 195 entry-table slots claimed; theirs (46) walks **401**, **0 unknown**, **267 of 268**, 196 claimed. All 14 of ours appear in the 46 unchanged, and the corpus uses 16 opcodes only their table can decode. Their 9-bit opcode reading is not decidable on this disc — bit 8 is set in **none** of the 7,015 opcode words the walk decodes — but the high field is real and is not an opcode (web-psx `docs/effect-format.md` [effect.xref.script])
  - src: external contribution — web-psx `docs/effect-format.md` [effect.xref.script]
- **Entry `[7]` is the whole timeline with the camera inside it, and the camera tables are confirmed by an arithmetic nobody arranged: 12 gaps, 1 keyframe count per table, and 59 keyframes a file × 401 files = 23,659 — the same total their own corpus walk reports.** — `[S] 1/3`
  - S: at 2 bytes an end-frame or command entry and 6 bytes a data entry, the gaps between their offsets divide with no residue: FOR-EACH-TARGET 17 17 17 17, MAIN 21 21 21 21, CLEANUP 21 21 21 21. Over all 401 files: 23,659 of 23,659 end frames at or under `0x2000`; 5,018 keyframes with a non-zero command word; 256 shake keyframes in 69 files, all 256 reading as 3 `int16` within ±4096. `timeline_channel_base = timeline + 8` is confirmed from the sound-track side (web-psx `docs/effect-format.md` [effect.xref.timeline])
  - src: external contribution — web-psx `docs/effect-format.md` [effect.xref.timeline]
- **The flags section is confirmed field for field including a prediction — bit 7 of the flags word is never set — and the 4 sound-channel configs' mode byte is in 0..4 in **1,596 of 1,596** slots with all 5 modes occurring.** — `[S] 1/3`
  - S: over the 399 files with the section, bits set by bit `0:200 1:109 2:1 3:102 4:31 5:82 6:44`, no flags word above `0xff`, `default_frame_delay` 0 in 316 of 399 with 100 its commonest explicit value. Bits 5 and 6 at 82 and 44 are the same 2 counts the time-scale enables produce, so the claims corroborate each other. Their emitter record checks too — 3,227 records over 401 files at `0x14 + 196n`, byte 0 zero in all 3,227, **byte 5 zero in 3,226 and `0x77` in exactly 1**, which is the same lone outlier they report from a corpus they counted differently (web-psx `docs/effect-format.md` [effect.xref.flags], [effect.xref.emitter])
  - src: external contribution — web-psx `docs/effect-format.md` [effect.xref.flags]
- **Their curve-index map hits 6 times against 3 transcriptions written with no knowledge of it, each hit a nibble in `0x08..0x0F` paired with the `int16` field pair it drives — and the callback-parameter naming is adopted in our source.** — `[S·D] 2/3`
  - S: the grid's scroll speed reads the high nibble of `0x0B` and lerps `0x5c`/`0x60` (their radial velocity); its scroll angle the low nibble of `0x09` with `0x2e`/`0x34` (velocity base angle); the ring fan's angle the low nibble of `0x0A` with `0x44`/`0x48` (inertia); its texture scroll the high nibble of `0x0C` with `0x7c`/`0x88` (drag); its centre `0x14`/`0x1a`, `0x16`/`0x1c`, `0x18`/`0x1e` (start/end position); its angular acceleration the low nibble of `0x0F` with `0xb4`/`0xb6` (spawn interval). The colour path matches their `color_curve_enable` and their `u32(+0x10) & 0xfff` split 4/4/4
  - D: those transcriptions run beside the console's own copy and byte-diff every write — 3 shapes, 52 files, 100,000+ brackets, **0 disagree** (web-psx `docs/effect-format.md` [effect.xref.emitter], [effect.hle.score])
  - src: external contribution — web-psx `docs/effect-format.md` [effect.xref.emitter]
- **Both camera pre-patchers are confirmed, and watched moving the operands live rather than only read out of the binary.** — `[S·D] 2/3`
  - S: `{73}` is `BATTLE.BIN+e04a4` and `{38}` is `+e0780`, both reading and writing through `LoadHalfword`/`StoreHalfword`; `{73}` writes operands 1..7 and `{38}` writes operand 8 alone
  - D: `tools/eventsim.ts --patchwatch` snapshots the following `Camera`'s 8 operands out of the live script buffer at each dispatch — `{38}` moved operand 7 and only 7 on all 8 observations, `{73}` moved operands inside 0..6 and never 7, and both wrote the same `Camera` in one run without colliding. Three refinements are in [[Scenario Camera Opcodes]]: the distance law is `max(ΣΔpos², ΣΔrot²/2)` with zoom in neither sum, the `{38}` operand is a **speed** divisor rather than a time, and the peek past an intervening `{38}` is dead code on this disc (web-psx `docs/event-seam.md` [event.static.prepatch])
  - src: external contribution — web-psx `docs/event-seam.md` [event.static.prepatch]
- **`V == 0` broadcasting to every existing unit is confirmed exactly — the only claim of theirs we checked that needed no correction of any kind.** — `[S] 1/3`
  - S: `ClassifyUnitSpec` at `BATTLE.BIN+e0928`, shared by **11 rungs** — `0x11`, `0x2C`, `0x2D`, `0x32`, `0x53`, `0x69`, `0x6C`, `0x6D`, `0x80`, `0x81`, `0x83` — and `{47}` is not one of them. A model reading `Units=0, Multi=0` as "unit 0" would aim 96 of the disc's 6,505 `0x11`s, 89 of its 620 `0x32`s and 27 of its 808 `0x2D`s at one unit instead of all of them (web-psx `docs/event-seam.md` [event.hle.units])
  - src: external contribution — web-psx `docs/event-seam.md` [event.hle.units]
- **The `{6B}` ramp was watched arriving at the sound driver exactly as their reading predicts, and their `{6A}`/`{6B}` decode is right end to end including "byte 1 is StartVol, not Echo".** — `[S·D] 2/3`
  - S: `BATTLE.BIN+e29ac` and `+e2a54` decompiled — operands `[Sound, StartVol, Volume, Stacking, Time]`, task id `0x35`, `handle = 0x10000 | Sound`, bank 1 being `SOUND/ENV.SED`, which is the namespace our `.SED` decoder predicts from the header's `+0ah`
  - D: handle `0x00010001` stepping **1 → 24 in 24 single steps over exactly 255 frames**, one call per display frame, 375 of its 379 inter-call gaps a single frame. The whole ambient bed is 997 set-level calls in a 16,000-frame window (web-psx `docs/event-seam.md` [event.hle.sound], `docs/audio-doorlog.md` [doorlog.vocabulary.setlevel])
  - src: external contribution — web-psx `docs/event-seam.md` [event.hle.sound]
- **This vault corrected *us*, twice, in ways that mattered.** — `[S·D] 2/3`
  - S: `SCUS_942.21+336c` is set-volume-by-handle, not the conditional stop our own documentation called it — and it is our highest-frequency game-side sound entrypoint, so a mislabelled verb sat in front of every native consumer we have. Separately, their screen-tint destination was right and ours was the wrong label: the 3 words at `0x800F5B40` are the **live global screen tint**, which we exported as `LIGHT_AMBIENT` and called "the map chunk's three ambient bytes"; the map's authored ambient is a different number that disagrees on most frames of a battle
  - D: 997 calls in a battle window and 1,677 over a recorded session for the first; for the second, `tint >> 16` and the GTE back colour move in lockstep over 11,956 frames, and the cache, the accumulator and op `0x5A` agree with no residue. The constant is `GTE_BACK_COLOUR` now (web-psx `docs/audio-seam.md` [audio.door.doors], `docs/scene-viewport.md` [viewport.tint])
  - src: external contribution — web-psx `docs/scene-viewport.md` [viewport.tint]
- **Also confirmed, each in its own note: the 40-halfword direction block byte for byte; the charging pose sets; the weapon table's values and `unit+0x13b` as its index; the sequencer's 66-entry switch table at `0x80067F20`; the `{47}` ghost id rule and its shared queue; the damage popup's phase machine and its 1/7/12 stagger; all 6 scenario-record field meanings, 2 of them by the drive; and the `E040` palette measurement to the byte.** — `[S·D] 2/3`
  - S: `BATTLE.BIN+0x10dc` dumped whole (F `0,0,2,2`, E `0,1,1,0`, B `0×9,2×6,0`, A `1,2,2,3,3,4,4,5,5,4,4,3,3,2,2,1`); `+0x2cde8` sets 0/1/7/10 byte-correct; `+0x2d364` values byte-correct; the switch table bound-checked by an `sltiu` against `0x42`; `Charge+X` reading the no-file sentinel for all 8 of 406–413, which are exactly skillset 8, with 12 of 13 config-10 fields exact
  - D: the scenario songs came from the drive — warped to 218 it delivered `MUSIC_74` then `MUSIC_16`, the record's `+5` and `+6` in order, and `MUSIC_51` (scenario 1's, displaced) was fetched by none of the warps; the popup phase, stagger and blend-on frame were watched over an auto-battle; `E040`'s palettes read 378 and 249 non-zero bytes, their two numbers exactly (web-psx `docs/warp.md` [warp.identifies.fields], `docs/scene-viewport.md` [viewport.number.ramp], `docs/effect-format.md` [effect.xref.palette], [effect.charge])
  - src: external contribution — web-psx `docs/warp.md` [warp.identifies.fields]
- **Corrections landed as `⚠ SUPERSEDED` sub-bullets, sprite and ability domain: the height table's entries, the weapon-type index ordering, the Ability Animation Table's row count and RAM base, the headline pose claim, the layer table's stride, and the damage popup's ramp base.** — `[S] 1/3`
  - S: [[Unit Sprite Height Table]] (4 distinct heights, not 5), [[Weapon Animation System]] (GUN 10, BOW 11–12, INSTRUMENT 13, BOOK 14, SPEAR 15–16, BAG 17–18), [[Ability Animation Table]] (454 rows not 512; 70 with `effect_anim_id == 0` not 96; RAM `0x80093C10` not `0x8003CE10`), [[Sprite Cardinal Pose Selection]] (`FUN_8006bbfc` has no camera term and is a different routine), [[Damage Number Popup System]] (the layer table is 24 × 4 `u32`; the scale ramp is at `+0xbac` in 2 families of 3 rows, not at `+0xc00` in 1) — each with the dump inline (2026-08-19)
  - src: external contribution — web-psx `docs/combat-tables.md` [combat.animation]
- **Corrections landed as `⚠ SUPERSEDED` sub-bullets, effect and sound domain: the particle tpage formula, the effect file buffer's prefix and load base, the texture `+0x400` word, the two-palettes inference, the time-scale value alphabet, the `.SED` stream extent, and 3 FEDS header readings.** — `[S·D] 2/3`
  - S: [[Particle Emitter Format]], [[Effect File Buffer]], [[Effect Texture Upload]], [[Effect Frame Pacing]], [[Effect Sound Timing]], [[FEDS Sound Definition Format]]
  - D: 4,919 GPU packets refute `(animation_word & 0xE0) | 0x08`; 183,176 words of DMA provenance put an effect file's byte 0 at `0x801C2500`; bytes 0–2 of the texture word read as a VRAM Y under 512 in **0 of 401** and byte 3 is non-zero in exactly 56 files, all 56 of them 8bpp — the opposite of the claimed direction; the selector bit `0x10` is set in 1,806 framesets and **1,806 of 1,806 are 4bpp**; **no** time-scale nibble in 157,200 samples is 0 or 1 and 15 are 10; a stream ends on its own `0x90` bar, so the next-offset rule truncates 94 of 2,388 (web-psx `docs/effect-format.md` [effect.xref.texture], [effect.xref.palette], [effect.xref.pacing], `docs/audio-seam.md` [audio.smd.sed], [audio.smd.feds])
  - src: external contribution — web-psx `docs/effect-format.md` [effect.xref.texture]
- **Corrections landed as `⚠ SUPERSEDED` sub-bullets, event and scenario domain: `{1E}`'s handler, the `{7E}`/`{7F}` CONTESTED pair, the block skip's tail, the `{63}` low nibble, the ENTD facing labels, the `{91}` overlay's causal story, the `{47}` idempotency gate, the write gate's direction, the scenario table's length, and 4 SEQ arities.** — `[S·D] 2/3`
  - S: [[Event Opcode Catalog]] (`0x8013db9c` is `{1D}`'s builder; `{1E}` has no rung), [[Wait Value Opcode]] (`0x7E` and `0x7F` have distinct case bodies), [[Block Execution]] (the forward scan falls into the advance tail), [[Scenario Camera Opcodes]] (`curve == 1` is decelerate; `group == 1` eases the rotations only), [[ENTD Unit Deployment Table]] (the cardinal labels contradict this vault's own wheel), [[DarkScreen Opcode]] (the `0x801CA664` prologue is `REQUIRE.OUT` landing 19 frames later), [[Add Ghost Unit Opcode]] (the loop is a free-id allocator, so a duplicate spawns a **second** ghost), [[Event Variable File]] (the gate discards; `0x2C` is a range and `0x19` is a mirror on the store path), [[Scenario Table]] (480 records; 490 is the largest id), [[EVTCHR Script VM]] (`0xFFCA` = 1, `0xFFE5` = 2, `0xFFE6` = 2, `0xFFF9` = 2)
  - D: `{91}`'s builder is `EVENT/ATTACK.OUT+0xAD68` — the resident bytes match `ATTACK.OUT` at **125,037 of 125,956 (99.3%)** with a 57,436-byte exact leading run, against 13–41% for every rival — and the opcode is `Show Map Title`, occurring 8 times on the whole disc (web-psx `docs/event-seam.md` [event.hle.spawn], [event.hle.ghost], [event.hle.vars.gate], [event.seq.arity])
  - src: external contribution — web-psx `docs/event-seam.md` [event.hle.dispatch]

## Reproducing A Continuous Camera Pose Track

[[Scenario Beat Capture]] is a good protocol and it concedes its own limit:
*"camera framing at the same PC still differs (scenario director)"*. A beat is a
freeze-frame; a cinematic camera is a **track**. The measurement that closes the
gap is a per-frame pose capture, and your existing PCSX-Redux Lua setup can
already do it — you run a non-pausing exec BP on the per-vsync camera ticker
`0x801439c0`, which is how the `{63}` ease fixture was measured. Keep it, dump
the whole live pose per tick instead of one field, park on the chapel savestate,
and let the cinematic run untouched.

Two readings of the pose are available and both are worth dumping, because a
disagreement between them is itself a finding:

- The **scratch struct** this vault already names: `+0x68` X, `+0x6c` Y, `+0x70`
  Z, `+0x74` pitch, `+0x78` yaw, `+0x7c` roll, `+0x80` zoom.
- The **event variables the `{19}` task itself reads and writes** — the authored
  pose in the script's own units. The task takes its ids from `CAMERA_VAR_IDS` at
  `80165ED4h` against the variable array at `*(0x80165F9C)`: ids
  `0x1A`/`0x1B`/`0x1C` are X/Z/Y **stored `<<10`**, and `0x1D`/`0x1E`/`0x1F`/
  `0x20` are Angle/MapRot/CamRot/Zoom raw. `0x1E` is pinned as Map Rotation by
  the task's own one-shot unwrap, which reads variable `1Eh` by name at
  `0x801461c8`. Seven words a tick.

**And for the chapel there is a third source, which makes this a three-way
check.** We looked, because a relative orbit would have been a tidy explanation
for a scene that "opens already moving" out of a script whose numbers look
absolute — and it is not the explanation. The traced frames are event 2's chain,
slots 2 and 4, which hold **12 and 9 `Camera` instructions and none of the 3
pre-patchers between them**: 0 `Focus`, 0 `Focus Speed`, 0 `Camera Move
Relative`. Only slot 5, which the cut at 5587 hands over to, has any, and those
are `Focus` and `Focus Speed`. So **every pose in the chapel opening is the
number on the disc**, readable statically by all parties with no machine at all —
and a consumer replaying the segment needs no live-pose feedback to reproduce it.
Three independent sources for one camera: your Lua trace of the live fields, our
wire trace of what a consumer is sent, and the script's own `{19}` operands.

**Our numbers, published as the cross-check.** We captured scenario 1 / MAP062 /
ENTD 256 (aux opens event 2) and read the camera off the producer's own wire
bytes: **1,911 records over frames 3819–5999**, ~36 s, covering the opening
sweep, the whole *"God, please forgive us sinful children of Ivalice"* narration,
and the cut past it.

| frame | eye | centre | yaw | zoom |
| --- | --- | --- | --- | --- |
| 3819 | (1655, 1217, −1657) | (392, 322, −392) | 3584 | 0.999 |
| 3943 | (604, **2201**, 119) | (−341, 440, 121) | 3074 | **1.888** |
| 4048 | (974, 2035, 107) | (−223, 434, 120) | 3079 | 1.664 |
| 4151 | (1371, 1804, 44) | (−76, 427, 108) | 3101 | 1.400 |
| 4252 | (1804, 1478, −421) | (150, 439, −4) | 3233 | 1.028 |
| 4355 | (1707, 1361, −1146) | (227, 420, −188) | 3446 | 0.954 |
| **4457** | **(1444, 1271, −1531)** | **(182, 377, −267)** | **3584** | 0.999 |
| … | *(unchanged for 16 s)* | | | |
| 5484 | (1248, 1271, −1711) | (139, 377, −309) | 3659 | 0.999 |
| 5587 | (−1309, 1271, −1911) | (−427, 377, −356) | 4432 | 0.999 |

Four claims a Lua pose track either reproduces or does not:

1. **It opens already moving** — by 3943 the camera is at a close framing (zoom
   1.888, nearly twice the resting scale) high above the chapel at eye Y = 2,201,
   yaw 3074.
2. **It descends and pulls out** over ~300 frames (5 s): eye Y
   2201 → 1478 → 1271, zoom 1.888 → 1.028 → 0.999. The descent is **930 units**.
3. **It turns 45° while it falls** — yaw 3074 → 3584, i.e. **510 of 4096**.
4. **It arrives at frame 4503 and holds for 945 more records** — 16 s at eye
   (1444, 1271, −1531), yaw 3584, zoom 0.999; the whole narration is spoken at
   one camera. Then it **cuts** at 5587 to yaw 4432 (336 mod 4096, a 74° turn)
   and 2,750 units across, settling at 5599 and holding 198 more. Both arrivals
   are settle-detected (still to within half a unit and a thousandth of a turn,
   held a second), not eyeballed.

**How to compare.** Our poses are the *consumer's* camera in world units rather
than the raw event variables, and our frame indices are our corpus run's — so
align on the first camera move, and compare the quantities that are the game's
own rather than anyone's projection: **yaw in 4096ths**, the **zoom ratio**, and
the **shape of the track**. Since the segment is absolute-authored, a fourth
comparison is free: the `{19}` operands in slots 2 and 4 should predict the whole
track, up to the task's interpolation.

**The decoder for what you capture** — five `{19}` behaviours that make a raw
pose track legible, all detailed in [[Scenario Camera Opcodes]]:

- the `0x2710` sentinel is per-field on **every** component (tested at
  `0x801462FC`, `0x801465CC`, `0x8014661C`), and the `{73}` pre-patcher applies
  the same rule per field;
- a **one-shot `+4096` map-rotation unwrap** on the first Camera of a scene,
  gated on `0x801696F8`, which startup sets to 1 exactly once — an unexplained
  4096-unit jump at the start of a track is this;
- there are **two dead zones** — a move under 1536 in 1/256 units (6 script units) on
  X/Z/Y, or under 96 in 1/64 units (1.5 units) on the rotations and zoom, is
  snapped in the epilogue rather than animated;
- a **genuine rounding bug at `0x80146538`**, masking with `0x0FF` against a
  6-bit fixed point, so `(v & 0xFF) < 33` only passes when `(v >> 6) & 3 == 0`
  and three times in four it adds a whole unit. **We think this is the ±1/1024
  residual** reported in that note: the measured fixture moves MapRot 1536 →
  2560, a 1024-unit move, so ±1 unit *is* ±1/1024 of it;
- the epilogue **writes the exact parameters regardless**, so intermediate frames
  need only be approximate but the landing must be exact.

One clock question while you are in there: we read `Time` as **task ticks** — one
pass of the 16-slot fiber scheduler at `0x8014CA80`, which during an event is one
game frame, two video frames — rather than vblanks. The `{63}` fixture moved
1536 → 2560 over "exactly 64 frames", which agrees if those samples were game
frames and disagrees if they were vblanks. The pose track settles it in one run.

## Tooling Notes

Offered because a few `[S] 1/3` points here are one cheap run from `[D]`, and
because the corrections above split cleanly into ones a shadow diff finds by
itself and ones only a census finds.

- Prefer to **bound a capture by the machine's clock, not by wall time.** Capture
  identifiers here are shaped like "90 s chapel capture" and "30 s". A wall-clock
  window is not reproducible across hosts and cannot be diffed against a later
  run; an emulated frame count or an instruction count can. Ours name a frame
  range (`frames 3819–5999`) or a packet count, and a re-run reproduces them.
- Consider grading **`[R]` claims with a diff shadow rather than with eyeballs.**
  Run the reimplementation *beside* the emulated game and diff every byte it
  writes into the structures the game writes. A Godot port cannot live inside the
  emulator, but the shape works across a socket: emit per-frame state from both
  and diff. Our numbers for the shape: 3 effect shapes, 52 files, 100,000+
  brackets, 0 disagree.
- Remember that **a census over the whole corpus is the cheapest falsifier there is**, and it
  is what settled most of the corrections above: 401 files, 3,227 emitter
  records, 17,453 framesets, 157,200 nibbles, 2,388 voice streams. Several claims
  here are right about a mechanism and wrong about its *alphabet* — the
  time-scale values, the texture depth byte, the palette selector — and a
  histogram catches exactly that class in one run over the disc.
- Prefer **a static walk that has to close.** A script slot's first word is its
  text-section offset, so a width-table walk must land on it exactly — 304 of 304
  do, validating all 176 widths at once. The same trick on table offsets — each
  table's offset plus its length landing on the next one's — is what shows the
  Ability Animation Table stops at 454 rows. And the strongest single check in
  this whole pass was of that kind: 12 gaps between camera-table offsets dividing
  to one keyframe count each.
- Take **savestates at instruction boundaries** so a state composes with an input
  recording and a moment can be *resumed* rather than re-navigated.
- Read **the dispatch rather than transcribing it.** Every event-VM attribution
  error corrected above falls out of walking the interpreter's comparison chain
  and classifying each case body by what it calls. The same lesson applies one
  subsystem over: 4 of the sequencer's 66 arities were wrong in both of our
  transcriptions *and* in the community table they came from, and no shipped
  animation exercises any of the 4 — a corpus is blind to exactly the rows it
  does not contain.

## Round 2 — the 8 AI-Research issues (2026-08-21)

- **Round 2 adjudicated: 8 issues, every load-bearing claim re-derived off `project-assets/fft-extract` rather than accepted on assertion. The strike rate is high and the misses are informative — 2 claims were wrong, and in 4 places the vault already contained the answer and contradicted itself.** — `[S] 1/3`
  - S: confirmed independently — the SMD arg-size table's `0xFE` entry reads `0x02` at file `0x1950C` of `SCUS_942.21`; the noop sink `LAB_8001586C` takes exactly 37 of 128 jumptable slots and that set equals the arg-size zero set; `FUN_80018140` is `SpuSetReverbModeParam` (mode bound-checked `< 10`, libspu's 10-entry work-area size table at `ram:80028F9C` with index 4 = `0x6FE0` = Studio Large); `{DB}` reaches `LoadNextEvent` and `bne v0,zero -> 80143CDC`; `allocate_task_slot` returns `n` unchanged for `n < 16`; EVTCHR filler starts at `0x6D80` in 136 of 137 blocks (= `0x980 + 256×200/2`), the exception being block 16; `FUN_80184E98`'s absorb and half arms fall through and only cancel returns; `FUN_8001ACF0` gates the `DAT_80037080` stores on bit 0 of `0x8002AD3C`; config 12's `direction_flags` is `0x0400` and its radius pair is `(2112, 1872)`
  - R: none — knowledge-base round; the two code consequences (`spu.gd`'s `RAM_INSTRUMENT_BASE`, `apply_element_defense`) are flagged in their notes and deliberately not changed here
  - src: external contribution — web-psx, ExMateria-AI-Research#1–#8
- **`ExMateria-AI-Research#8` was filed as an unresolved disagreement with a fixture being staged on their side; it is decidable statically and their reading wins, so neither fixture is needed to settle it.** — `[S] 1/3`
  - S: `FUN_80184E98` — absorb ORs `0x400` and falls through (`ram:80184ED0`→`ram:80184ED8`); cancel is the single early return (`j 0x80184F8C` at `ram:80184F00`); half rewrites `CurActData+0x4` with a truncating halve and falls through (`ram:80184F44`→`ram:80184F48`); double ORs `0x800` and shifts left. Our own `apply_element_defense` implements the refuted else-ladder
  - src: external contribution — web-psx `docs/rules.md` [rules.damage]
- **Two of their claims did not survive, and both are worth returning: `{DB}`'s chaining body is shared with `{E3}` rather than being `{DB}`'s alone, and `Dialogue Font Palette.md` is not self-contradictory — its older reading is correctly marked `⚠ SUPERSEDED` by its newer one, which is the convention working.** — `[S] 1/3`
  - S: the dispatcher is a compare chain whose delay slots load the *next* constant — `ram:801442AC beq s4,v0` against `v0 = 0xDB` and `ram:801442B4 bne s4,v0` against `v0 = 0xE3` both arrive at `0x801442BC`, so the two terminators share one body. Separately, the restore pass at `0x8014429C` that this vault attributed to `{DB}` belongs to **`{12}`** (chain constant `ori v0,zero,0x12` at `ram:80144254`), an opcode the catalog still lists as having no confirmed behaviour
  - src: external contribution — web-psx `docs/event-seam.md`
- **Counter-corrections owed back on the sound census: their header table's `0x18` row is mislabelled, and `0x18` is a live varying byte rather than a constant 0.** — `[S] 1/3`
  - S: over the 100 `SOUND/MUSIC_*.SMD` files, `0x18` takes **20 distinct values in 40..127** (mode 127, ×28) while it is `0x19` that is `0` ×100. `FUN_800136C0` copies the halfword at header `+0x18` into `ctx+0x1A` (`ram:80013728`–`ram:80013730`), and the driver's own defaults routine initialises `ctx+0x1A` to `0x7F` (`ram:800137B4`–`ram:800137C4`) — a max-valued default with a 40..127 spread, which reads as a per-song **master volume**. Offered as a hypothesis, not a finding: we have not traced `ctx+0x1A`'s consumer
  - src: external contribution — web-psx `docs/audio-seam.md` [audio.reverb.preset]

## Notes

(empty — user territory)

## Related

- [[Ability Animation Table]]
- [[Add Ghost Unit Opcode]]
- [[Block Execution]]
- [[Color Tint Luma Modes]]
- [[Combat Color Appliers]]
- [[Damage Number Popup System]]
- [[DarkScreen Opcode]]
- [[ENTD Unit Deployment Table]]
- [[EVTCHR Script VM]]
- [[Effect Camera System]]
- [[Effect Execution Model]]
- [[Effect File Buffer]]
- [[Effect Frame Pacing]]
- [[Effect Sound Timing]]
- [[Effect Texture Upload]]
- [[Event Opcode Catalog]]
- [[Event Sound OpCodes]]
- [[Event Unit Selector]]
- [[Event Variable File]]
- [[FEDS Sound Definition Format]]
- [[Particle Curve Indices]]
- [[Particle Emitter Format]]
- [[Scenario Beat Capture]]
- [[Scenario Camera Opcodes]]
- [[Scenario Table]]
- [[Sprite Cardinal Pose Selection]]
- [[TRAP Charge Particle System]]
- [[Unit Sprite Height Table]]
- [[Wait Value Opcode]]
- [[Weapon Animation System]]
