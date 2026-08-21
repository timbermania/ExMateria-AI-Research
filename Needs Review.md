# Needs Review

The index `AGENTS.md` requires: *"Unresolvable contradictions get `CONTESTED`
appended to both badges and are listed in a `Needs Review` index note."* This is
that note. It is a **register of open contradictions**, not an adjudication —
each entry says what is in dispute and where both sides live, so a future pass
can pick one up without re-finding it.

Census taken **2026-08-21**: 33 `CONTESTED` badges across 15 notes, grouped
below into 13 open disputes plus 1 resolved pair; a 14th dispute was added the
same day out of [[Web-psx Cross-Validation]]. Reproduce with:

```bash
grep -cE '`\[[^]]*\] [0-9]/3 CONTESTED`' *.md | grep -v ":0"
```

Two cautions on reading this note. First, the vault marks each side of a
contradiction but does **not** record which side pairs with which, so the
groupings below are a reading of the claims, not metadata — where a badge has no
obvious counterpart it is filed alone and said to be. Second, a badge going
stale is invisible: the `Wait Value Opcode` pair below was settled on 2026-08-19
and still carried live `CONTESTED` badges two days later. **When you resolve a
dispute, clear the badges and move the entry to `## Resolved` in the same edit.**

## Open — Effect Runtime

- **Effect-runtime global pointer addresses.** `timeline_channel_base` is
  `0x801BBF80` in [[Effect Sound Timing]] and `0x801BBF84` in
  [[Effect Execution Model]] / [[Effect File Format]]. Separately,
  `effect_anim_tbl_ptr` is `0x801BBF7C` in [[Effect File Format]]'s own table
  (agreeing with effect-editor's `known_functions.lua`) and `0x801BBF88` in the
  2026-04-16 working document, which is where `effect_data_ptr` lives on the
  other reading. Both are 4-byte disagreements in one contiguous pointer block,
  so one careful re-read of the initialisation site settles all four badges.
- **Effect-file header shape.** [[E001.BIN Memory Mapping]] carries two
  incompatible headers: nine section pointers at file offsets `0x00–0x24`, and
  ten `uint32` section offsets totalling 40 bytes with a different field
  assignment. [[Effect Texture Upload]] inherits the disagreement downstream —
  the texture section is reached via the header pointer at `+0x28` on one point
  and `header[0x24]` on another. web-psx's corpus walk over all 401 files is the
  natural falsifier (see [[Web-psx Cross-Validation]]).
- **Effect-script instruction sizes.** [[Effect Execution Model]] says the
  handler advances `script_position` by 2, 4, or 6 bytes, and separately
  tabulates 8-byte (3-arg) forms for opcodes 8, 11 and 14. Both cannot hold.
  Note that web-psx's 46-opcode table — adopted outright, 401 of 401 files
  walking with 0 unknown — supersedes the 14-opcode reading these points were
  written against, so re-deriving beats adjudicating here.
- **Pattern 2 corpus count.** [[Effect Execution Model]] reports 136 files on
  one point and 138 on another for the same sweep. A recount over the corpus
  settles it; the `[R] 1/3` side is the reimplementation's number.
- **E001.BIN emitter spread values.** [[E001.BIN Memory Mapping]] states the
  non-zero spreads two ways — "emitter 5 spread start (10, 16, 10) decaying to
  zero" versus "E5 vertical ±Y = 16 with both diagonal spreads = 10". Both are
  `[ ] 0/3`, i.e. neither side carries locatable evidence; a direct read of
  E001.BIN outranks both.
- **E001.BIN emitter start frames.** Same note, same evidence problem: "only
  emitter 5 starts at frame 50, all others at frame 0" versus "timing section 0
  spawns emitter 0 at frame 10 and emitter 1 at frame 25". Both `[ ] 0/3`.

## Open — Charge VFX

- **Ownership of the global block at `0x801bade0`.** Three notes put different
  handlers' private state at the same base: [[Spell Charge Lines System]]
  (handler 4 — view-space centre, 16 line slots, LINE_G2 groups),
  [[Summon Orb Orbital System]] (handler 22 — ring write index, orbital radius,
  3×10 ring buffer), and [[TRAP Charge Particle System]] (handler 17, scanning
  `DAT_801bade8` for a free byte and bypassing the shared spawner). Either the
  block is genuinely shared scratch reused by whichever handler is live — which
  would be a finding worth stating — or two of the three transcriptions are
  mis-attributed.
- **Charge-handler dispatch numbering.** [[Spell Charge Effect System]] maps
  jumptable index `0x05` to `0x801b27dc` = summon charge orbs, and separately
  says handler 5 at `FUN_801b27dc` is unreachable because no `anim_type` in
  `DAT_801b84dc` routes to func_id 5. The other charge notes number their
  handlers 17, 18 and 22, so the two numbering schemes need reconciling before
  either claim can be checked.
- **Where charge VFX anchor.** [[Summon Charge Lines System]] has handler 18
  rendering at `target.Y − height/2` (the target's torso), against
  [[Unit Sprite Height Table]]'s claim that the charge VFX handlers anchor at the
  caster's head, `height + 8`. These may simply be different handlers doing
  different things — in which case the contradiction is in the *scope* of the
  height-table claim, not in either measurement.

## Open — Sprite & Dialogue

- **Unit sprite height table cross-references.** [[Unit Sprite Height Table]]'s
  "17+ sites" call-site list is badged `CONTESTED` with no counterpart in the
  vault. Its `src:` is `STRUCTURE_DEFINITIONS.md`; the dispute is most likely
  against a later count. **Filed alone — the other side is unlocated.**
- **SHP file section layout.** [[Unit Sprite Render Pipeline]] gives the
  `TYPE1.SHP` frame-pointer array as a fixed `0x400`-byte `uint32` array after
  an 8-byte Section 1, and separately as words at `0x40 + 4·i` that are absolute
  file offsets only for `i = 81–167` and `242`, with entries `0–80` holding a
  compact `0x7C + 6·i` progression. The second is byte-verified against the file
  and the first is `[R] 1/3` from the reimplementation, which is the weaker
  position under the vault's own precedence rules.
- **Which FRAME palette the dialogue box samples.** [[Dialogue Font Palette]]
  carries two `[S·D·R] 3/3` points on the box-versus-prayer palette split. Both
  are heavily evidenced, which is what makes this one worth a careful read
  rather than a recount. web-psx's round-2 review concluded this note is **not**
  self-contradictory — that its older reading is correctly marked
  `⚠ SUPERSEDED` by its newer one — so these two badges may already be stale.
  See [[Web-psx Cross-Validation]].

## Open — Scenario Camera

- **What clock is `{19}`'s `Time` operand in?** [[Scenario Camera Opcodes]]
  reads the task body as yielding one vblank per interpolation frame; web-psx
  reads `Time` as **task ticks** — one pass of the 16-slot fiber scheduler at
  `0x8014CA80`, which during an event is one game frame and so two video frames.
  A 2× error either way, and it is decidable in one run: the `{63}` fixture
  moved MapRot 1536 → 2560 over "exactly 64 frames", which agrees with the
  task-tick reading if those samples were game frames and with the vblank
  reading if they were vblanks. Recorded 2026-08-21 while dissolving
  [[Web-psx Cross-Validation]] — it had sat outside that note's `## Points`
  section, which is why an earlier coverage pass over the points alone missed
  it.

## Open — Capture Rig

- **Is PCSX-Redux a faithful audio oracle for `protect_no_music`?**
  [[PCSX-Redux Capture Rig]] says no on one point — ear-A/B against a real-PSX
  recording puts PCSX 30–50% slow with voice 20 attenuated, making Godot the
  closer oracle — and effectively yes on another, where the 66 immediate-mode
  vol-zero writes are shown ~2,650× sparser than the 44.1 kHz latch window, so
  real hardware must mute voice 20 too. This is the one dispute here that
  changes what a parity number *means*, so it is the highest-value entry in this
  note. Both sides are `[D] 1/3`.

## Resolved

- **`0x7E` / `0x7F` handler attribution** — [[Wait Value Opcode]], settled
  2026-08-19. Reading the interpreter's comparison chain mechanically gives
  `0x7E` a wait body at `0x80145afc` calling `FUN_8014a3f8`, and `0x7F` a
  separate body at `0x80145b14` calling `0x80133158` / `0x8008dac0`. The master
  catalog had the wrong half of the pair, and no delay-slot mislabel argument is
  needed. Both badges cleared 2026-08-21.

## Notes

(empty — user territory)

## Related

- [[AI Research Index]]
- [[Web-psx Cross-Validation]]
- [[Scenario Camera Opcodes]]
