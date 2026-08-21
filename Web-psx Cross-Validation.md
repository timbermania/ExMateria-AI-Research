# Web-psx Cross-Validation

**web-psx** is a PlayStation emulator written as a programmable PSX debugger and
FFT reverse-engineering harness — JIT-to-JavaScript CPU/GTE, WebGPU GPU,
WebAudio SPU, with a Lean 4 sibling emulator, `lean-psx`, as the semantics
reference. It is the external contributor behind every `src:` line in this vault
of the form:

```
src: external contribution — web-psx <doc> [<anchor>] (see [[Web-psx Cross-Validation]])
```

This note exists to say what that provenance means. It is **reference material,
not a claim store**: the claims themselves live in the topic notes they are
about, which is where you should read and correct them.

## Their evidence standard

A claim is graded by re-deriving it two ways — statically out of `BATTLE.BIN` /
`SCUS_942.21` / the `EVENT` and `EFFECT` overlays on the disc, and dynamically
by running the retail game with a shadow reimplementation *beside* it and
byte-diffing every write (`tools/fxsim.ts` for `EFFECT/*.BIN`, `tools/eventsim.ts`
for the event VM), plus GPU packet capture at `DrawOTag` and a hardware-verified
CPU/GTE. Every number is reproducible from a disc image and a BIOS with no
hand-driving, which is why their contributions generally grade to this vault's
**D** as well as its **S**.

## What the exchange produced

Two rounds, 2026-08-19 and 2026-08-21. Every incorporable claim in this vault
was run against their emulator and disc before the first round was written, and
the balance ran in this vault's favour more often than not — the single largest
item was a claim of *ours* they had twice got wrong on their own side and have
now adopted (the quarter-turn pose resolution, see
[[Sprite Cardinal Pose Selection]]). Two of their own claims did not survive
round 2 and went back to them.

**None of that is stored here.** Confirmations, corrections and counter-corrections
were all landed on the specific points they are about — confirmations as evidence
sub-bullets, corrections as `⚠ SUPERSEDED` sub-bullets, each with the dump
inline. About 60 point bullets across 30 notes carry a web-psx `src:` line; find
them with:

```bash
grep -l "external contribution — web-psx" *.md
```

Open threads from the exchange that are *not* settled live in [[Needs Review]] —
currently the disagreement over whether `{19}`'s `Time` operand is counted in
task ticks or vblanks. Two code consequences were flagged and deliberately not
acted on in the vault: `spu.gd`'s `RAM_INSTRUMENT_BASE`, and `apply_element_defense`'s
element ladder (see [[Status Element Defense Interplay]]).

## Their methodology notes for this vault

Offered because several `[S] 1/3` points here are one cheap run from `[D]`, and
because the corrections above split cleanly into the ones a shadow diff finds by
itself and the ones only a census finds. This section is kept because it is
advice about *how this vault grades evidence*, which is not a claim about any
FFT subsystem and so has no topic note to live in.

- **Bound a capture by the machine's clock, not by wall time.** Capture
  identifiers in this vault are shaped like "90 s chapel capture" and "30 s". A
  wall-clock window is not reproducible across hosts and cannot be diffed
  against a later run; an emulated frame count or an instruction count can.
  Theirs name a frame range (`frames 3819–5999`) or a packet count, and a re-run
  reproduces them.
- **Grade `[R]` claims with a diff shadow rather than with eyeballs.** Run the
  reimplementation *beside* the emulated game and diff every byte it writes into
  the structures the game writes. A Godot port cannot live inside the emulator,
  but the shape works across a socket: emit per-frame state from both and diff.
  Their numbers for the shape: 3 effect shapes, 52 files, 100,000+ brackets, 0
  disagree.
- **A census over the whole corpus is the cheapest falsifier there is**, and it
  is what settled most of the corrections: 401 files, 3,227 emitter records,
  17,453 framesets, 157,200 nibbles, 2,388 voice streams. Several claims here
  were right about a mechanism and wrong about its *alphabet* — the time-scale
  values, the texture depth byte, the palette selector — and a histogram catches
  exactly that class in one run over the disc.
- **Prefer a static walk that has to close.** A script slot's first word is its
  text-section offset, so a width-table walk must land on it exactly — 304 of
  304 do, validating all 176 widths at once. The same trick on table offsets —
  each table's offset plus its length landing on the next one's — is what showed
  the Ability Animation Table stops at 454 rows. The strongest single check in
  the whole pass was of that kind: 12 gaps between camera-table offsets dividing
  to one keyframe count each.
- **Take savestates at instruction boundaries** so a state composes with an
  input recording and a moment can be *resumed* rather than re-navigated.
- **Read the dispatch rather than transcribing it.** Every event-VM attribution
  error corrected in round 1 falls out of walking the interpreter's comparison
  chain and classifying each case body by what it calls. The same lesson applies
  one subsystem over: 4 of the sequencer's 66 arities were wrong in both of
  their transcriptions *and* in the community table those came from, and no
  shipped animation exercises any of the 4 — a corpus is blind to exactly the
  rows it does not contain.

## Notes

(empty — user territory)

## Related

- [[AI Research Index]]
- [[Needs Review]]
- [[Scenario Beat Capture]]
- [[Scenario Camera Framing]]
- [[Scenario Camera Opcodes]]
- [[Sprite Cardinal Pose Selection]]
