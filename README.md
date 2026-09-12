# logging-core-nv

The half of structured logging a `core` library can reach: a record with
typed fields, a filter, a formatter family, and the `LgSink[e]` trait
whose effect parameter means a log call costs exactly what its
destination costs.

**Status: NOT IMPLEMENTED — interface only.**  Every `pub fn` body is a
`todo()`, so the signatures, the effect rows and the tests are published
and nothing is implemented.  The first implementation is the `0.1.0`
published over this.

## Why this package exists

[logging-nv](https://registry.novo-lang.org/logging-nv) declares
`layer = "host"`, because three of its nine modules perform something: a
console sink is `[io]`, a file sink is `[fs]`, a ring behind a slot is
`[mutate]`.  A `core` package may depend only on `core` packages.

So **the trait designed to let a library log at `[]` was out of reach of
exactly the libraries it was designed for.**  logging-nv's own README
named that as "the one thing in this design that does not fit its box"
and named the row that would close it; this is that row.

Nothing was redesigned to make the split.  Every declaration here was
already `[]` in logging-nv, and the line was already drawn in its module
table.

## What moved, what stayed

| declaration | here (`core`) | logging-nv (`host`) | why |
| --- | --- | --- | --- |
| `lgrecord` — `LgLevel`, `LgValue`, `LgField`, `LgRecord` | **moved** | — | every row `[]`; a record is the value that travels |
| `lgfilter` — `LgRule`, `LgFilter`, `LgFilterFault`, the spec grammar | **moved** | — | every row `[]`; a filter is two integers compared |
| `lgformat` — `LgFormat`, `LgHumanStyle`, `LgTheme`, `LgStamp`, `render_into` | **moved** | — | every row `[]`; rendering is arithmetic over a value the caller holds |
| `lgsink` — **`LgSink[e]`**, `LgSinkFault`, `LgNullSink`, `LgFanout`, `emit_if`, `emit_all`, `emit_and_flush` | **moved** | — | the declaration and the `[]` impl; `[e]` is a bound parameter and inside a `core` budget |
| `lgring` — `LgRing`, `push`, `drain`, `snapshot`, `len`, `dropped_count`, `is_full`, `clear`, `dump_into` | **moved** | — | the ring as a VALUE is `[]` |
| `lgring` — `LgRingSink`, `ring_sink`, `ring_in`, `drain_in` | — | **stays** | `impl LgSink[mutate]`; a sink that accumulates must own mutable state |
| `lgwrite` — `LgStdSink`, `LgStream`, `is_terminal` | — | **stays** | `[io]` |
| `lgfile` — `LgFileSink`, `LgRotate`, `LgFilePolicy`, the rotation arithmetic | — | **stays** | `[fs]` |
| `lgdeflog` — the device bridge | — | **stays** | needs deflog-decoder and deflog-parser, and their four-package closure |
| `lglog` — `LgLogger`, `now`, `to_std_log`, `emit_to_console_and_file` | — | **stays** | `[time]`, `[io]`, `[fs]` |

**Every name is the one logging-nv already published**, so nothing
downstream renames: `LgRecord` is still `LgRecord`, `lgfilter.parse_spec`
is still `lgfilter.parse_spec`, and a program that moves from one
package to the other changes one `use` line per module and nothing else.

## The one-line manifest change logging-nv makes at its next version

```toml
[dependencies]
logging-core-nv = { path = "../logging-core-nv" }   # a range at 0.0.2
```

…and deletes `src/lgrecord.nv`, `src/lgfilter.nv`, `src/lgformat.nv` and
`src/lgsink.nv`, which then resolve through the dependency.  Its
`layer = "host"` is unchanged and correct: what remains is the three
modules that perform.

**One rename is unavoidable and it is the smaller of the two available
ones.**  `lgring` splits, and two packages may not both ship a module of
that name.  Keeping `lgring` here — nine functions and the type — and
renaming logging-nv's remaining three (`ring_sink`, `ring_in`,
`drain_in`) into a module of its own is three call sites moved;
the other way round is nine.  `lgslot` is the suggested name, since what
those three have in common is the slot the caller owns.  That is
logging-nv's change to make and this lane did not make it.

## The load-bearing interface

```novo norun:pseudo
pub trait LgSink[e]
    fn emit(self, r: LgRecord) -> Result<Unit, LgSinkFault> [e]
    fn flush(self) -> Result<Unit, LgSinkFault> [e]
    fn accepts(self, l: LgLevel) -> Bool [e]
```

**`LgSink[e]` is the package**, and the effect parameter is what lets
one generic emit be written once and charged what the caller's own sink
costs — `[io]` over a terminal, `[fs]` over a rotating file, `[mutate]`
over a ring buffer, and **nothing at all** over the null sink or a
buffer a test drains.

The standard library's `Logger` shows what the alternative costs.  Its
`LogSink` is an enum, so — as
[its own page](https://novo-lang.org/docs/stdlib/log) says —
`Logger.info` is charged the union over the variants, `[io]`, "even when
the installed sink is `SinkNull`".  A program that logs into a buffer
and asserts on the bytes is charged for a console it never touches, and
a caller with no `[io]` to give cannot log at all.

**The sink takes the record, not the line.**  The obvious surface —
`emit(self, line: Str)` — is wrong twice.  A ring buffer on a device
would have to format on the target, which is the entire cost the
deferred-logging story exists to avoid.  And a program writing to a
terminal and a collector would have to render the same record twice, in
two formats, in a caller that has no reason to know there are two.

## The one example that will work

```novo
use lgfilter
use lgrecord
use lgsink

// A `core` library, logging at `[]` into whatever its caller brought.
fn served<S: lgsink.LgSink[e]>(to: S, f: LgFilter, status: Int) -> Result<Bool, LgSinkFault> [e]
    lgsink.emit_if(to, f,
                   lgrecord.with_field(
                       lgrecord.record(LgInfo, "http.server", "request served"),
                       lgrecord.field_int("status", status)))

fn main() [io]
    match served(lgsink.null_sink(), lgfilter.filter(LgInfo), 200)
        Ok(written) => println("${written}")
        Err(e)      => println(e.message())
```

`main` above is `[io]` only because it prints.  `served` — the library
function — costs nothing, and that is the whole argument.

## Adding it, and checking it

```console
$ novo pkg add logging-core-nv
$ novo pkg build
$ novo test tests/lgsink_tests.nv
```

The suites are **red on purpose**: every body is a `todo()`, so every
assertion reaches `not implemented: logging-core-nv.<module>.<fn>`.
That is what an interface release looks like from the outside, and it is
how the first implementation will know it is finished.

## The layer, and why

`core`, and every row is `[]` or the `[e]` that `LgSink[e]` binds.  The
`effect-budget` audit row counts a bound effect parameter as inside the
budget, which is the rule `docs/publishing.md` states: a `core` function
that names `[e]` has not spent `[io]`, it has said "whatever you hand
me".

### No device claim, and why

There is no `tests/embedded_probe.nv`.  An `LgRecord` holds a `Str`
message, a `Str` target and a list of `LgField`, every one of which
allocates; `lgformat.render_into` appends to a `[u8]` a caller grew.
None of that is a firmware shape.

What a device logs is deflog's **interned index** — a number and some
raw bytes, with no formatter linked into the image at all — and turning
that into an `LgRecord` is a host-side conversion that logging-nv's
`lgdeflog` already owns.  The two halves meet on the host, which is
where they should.

## What widened, and what did not

- **Nothing widened.**  Every declaration that moved was already `[]` in
  logging-nv, which is what made the split a move rather than a
  redesign.
- **The "buffer sink" the row asked for cannot be a sink at `core`, and
  this is the one place the plan's wording and the budget disagree.**  A
  sink is a place records are *put*; putting into a value is mutation,
  and `[mutate]` is outside `core`.  So what a `core` caller gets is
  `LgNullSink` — a real `impl LgSink[]` — plus `LgRing` as a value it
  threads, plus `dump_into` to spill that ring into whatever sink its
  host brought.  `LgRingSink` stays at `host`.  The capability is intact
  and the shape is a value rather than a sink; saying so is more useful
  than publishing a `[mutate]` row in a package that claims `[]`.
- **`lgring` had to keep its module name here**, which forces a rename
  on logging-nv's remaining three functions.  Two packages may not both
  ship `lgring.nv`; three call sites move instead of nine.
- **`LgSinkFault` names `IoError`**, which is a standard library type
  and costs a `core` consumer nothing — checked, because a fault type
  that dragged an effect into the budget would have sunk the split.
- **`lgring.dump_into` is in a `core` package although every sink it can
  reach today is in a `host` one.**  That is not an oversight: the
  function costs nothing and the caller pays for the sink it brought,
  which is precisely what a bound effect parameter is for.

## What `std.log` keeps

Unchanged from logging-nv's own answer, and it applies to both halves:
the global functions stay, `std.log`'s `Logger` keeps the whole
no-dependency case, and `render_at` keeps being the reference for the
text format — the human format here pads the level tag to the same
column nine, so a program migrating one subsystem at a time produces
output that still lines up.

## Reference

Rust's [`log`](https://docs.rs/log) facade,
[`tracing-subscriber`](https://docs.rs/tracing-subscriber)'s filter
grammar, and Python's
[`logging`](https://docs.python.org/3/library/logging.html).  The
specification-shaped pieces are their own: the logfmt convention, and
one JSON object per line.

## Licence

Apache-2.0.
