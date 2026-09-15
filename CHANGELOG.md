# Changelog

All notable changes to logging-core-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.0.2 — 2026-09-15

README rewritten to the package README style guide (docs/writing-a-readme.md); no change to the interface.

## 0.0.1 — 2026-09-12

The **interface**: every signature and every effect row, and no bodies.
`stability = "draft"`, and the release is recorded `implemented = false`.

### Added

- `lgsink` — `LgSink[e]`, the load-bearing interface, with
  `LgSinkFault`, `LgNullSink`, `LgFanout` and the three generic emits
  that bind `[e]`.
- `lgrecord` — `LgLevel`, `LgValue`, `LgField`, `LgRecord`, and the
  builders.
- `lgfilter` — `LgRule`, `LgFilter`, the longest-prefix segment rule,
  and the one-line specification grammar.
- `lgformat` — human with colour, logfmt and JSON lines, into a
  caller's buffer.
- `lgring` — the ring as a VALUE, and `dump_into` to spill it into
  whatever sink the caller brought.

### Notes

- **Every name is the one logging-nv already published.**  The split is
  a move and not a redesign: every declaration here was already `[]`
  over there, and nothing downstream renames.
- **Why the row exists**: logging-nv is `host`, a `core` package may
  depend only on `core` packages, and the trait designed to let a
  library log at `[]` was therefore out of reach of exactly the
  libraries it was designed for.
- **logging-nv's next version** adds one dependency line and deletes
  four modules.  One rename is unavoidable — `lgring` splits and two
  packages may not both ship that module name — and keeping it here
  moves three call sites instead of nine.
- **The buffer sink the plan asked for cannot be a sink at `core`**, and
  the README says so: putting into a value is mutation.  What a `core`
  caller gets is the null sink, the ring as a value, and `dump_into`.
- **No device claim.**  A record holds a `Str` and a list; what a device
  logs is deflog's interned index, and the conversion is host-side.

### Design notes

- `LgSinkFault` names `IoError`, which is a standard library type and
  costs a `core` consumer no effects.  A fault type that dragged an
  effect into the layer's budget would have prevented the split.
- `lgring.dump_into` is published here although every sink it can reach
  today lives in logging-nv.  The function itself costs nothing, and the
  caller pays for the sink it brought.  That is what a bound effect
  parameter is for.
- logging-nv's next version adds one dependency line on this package and
  deletes `src/lgrecord.nv`, `src/lgfilter.nv`, `src/lgformat.nv` and
  `src/lgsink.nv`, which then resolve through the dependency.  Its
  `layer = "host"` is unchanged.  Two packages may not both ship a
  module named `lgring`, so logging-nv's remaining three ring functions
  (`ring_sink`, `ring_in`, `drain_in`) need a module of their own;
  `lgslot` is the suggested name, and moving those three is three call
  sites rather than the nine the other split would move.
