# logging-core-nv

**Structured logging** records an event as a value with named fields rather than
as a sentence, so that a program reading the log can find a field by name. The
convention was made ordinary by Rust's [`log`](https://docs.rs/log) facade and
[`tracing-subscriber`](https://docs.rs/tracing-subscriber), and by Python's
[`logging`](https://docs.python.org/3/library/logging.html). This package brings
the part of it that performs no input or output to novo-lang: the record, the
level, the filter, the three text formats, the ring buffer, and the `LgSink[e]`
trait that says where a record goes.
[logging-nv](https://novo-lang.org/packages/logging-nv) is the companion
package that writes: a sink over a standard stream, a rotating file, a ring
behind a mutable slot, and a bridge from a microcontroller's deferred log.

**Status: NOT IMPLEMENTED — interface only.** Every function is declared with
its full signature, but every body is a `todo()` that panics when called. The
package is published so its design can be reviewed and depended on before it is
implemented. Version 0.1.0 will be the first working release.

## What it is

A **record** is one thing that happened. It carries a level, a target, a
message, a time, a list of fields and an optional source location.

A **level** says how loud a record is. There are six, from `LgTrace` to
`LgError`, with `LgOff` above them as a threshold that silences everything.

A **target** is the dotted name of the subsystem that emitted the record, such
as `http.server` or `ml.core`. The caller chooses it. It is the name a filter
matches on.

A **field** is one key and one value. A field value keeps its type: text, whole
number, real number, true or false, or present-and-empty. A consumer that reads
`status` gets a number on every line, so the field can be aggregated.

A **filter** decides whether a record is worth writing, before it is rendered.
It holds a default level and a list of rules, each a target prefix and the level
that prefix governs.

A **format** decides how a record is written. There are three. The human format
is for a person reading a terminal. The logfmt format writes `key=value` pairs
on one line, for a person who also pipes the output through `grep`. The JSON
lines format writes one JSON object per line, for a collector.

A **sink** is where a record goes. `LgSink[e]` is a trait with one effect
parameter. An implementation supplies the effects its own writing costs, and a
function that emits through any sink is charged exactly that. Writing to a
terminal costs `[io]`, writing to a file costs `[fs]`, pushing into a mutable
buffer costs `[mutate]`, and the sink that keeps nothing costs nothing. A
library can therefore log and still declare an empty effect list.

Every function in this package performs no input or output. The clock is never
read: a timestamp arrives as an argument, and `0.0` means the record has none.

## Install

```
novo pkg add logging-core-nv
```

## Example

```novo
use lgfilter
use lgformat
use lgrecord
use lgsink

// A function that logs and performs nothing. Its effect list is empty
// because the sink it was handed is the one that keeps nothing.
fn served(to: LgNullSink, f: LgFilter, status: Int) -> Result<Bool, LgSinkFault> []
    // A record: how loud it is, the subsystem it came from, the message.
    let r = lgrecord.record(LgInfo, "http.server", "request served")
    // One field whose value keeps its type, so a consumer reads a number.
    let full = lgrecord.with_field(r, lgrecord.field_int("status", status))
    // Write it if the filter and the sink both admit it.
    lgsink.emit_if(to, f, full)

fn main() [io]
    // Read the one-line filter a flag or an environment variable carries.
    match lgfilter.parse_spec("info,http=error")
        Err(e) => println(e.message())
        Ok(f)  =>
            // Render a record to text without writing it anywhere.
            println(lgformat.render(lgformat.logfmt(),
                                    lgrecord.record(LgWarn, "db", "slow query")))
            // Log through the sink that keeps nothing.
            match served(lgsink.null_sink(), f, 200)
                Ok(written) => println("${written}")
                Err(e)      => println(e.message())
```

Build and test with `novo pkg build` and `novo test`. Today `novo test` fails on
purpose: every test reaches a `not implemented` panic.

## What the package contains

| Module | Contents |
| --- | --- |
| `lgrecord` | The record and its parts: the six levels, the typed field value, the field, the record, the builders that add to one, and the check for reserved and duplicate keys. |
| `lgfilter` | The filter: a default level, a list of prefix rules, the longest-prefix lookup, the one-line specification grammar, and the report of rules no target can reach. |
| `lgformat` | The three renderings: human with colour, logfmt, and JSON lines. Each writes into a string or into a buffer the caller owns. |
| `lgsink` | The `LgSink[e]` trait, the fault type, the sink that keeps nothing, and the three generic emits that cost whatever the caller's sink costs. |
| `lgring` | The ring buffer as a value: the last N records, the count of those dropped to make room, and the call that spills the ring into another sink. |

## How to choose an entry point

**A library logs through `lgsink.emit_if`.** Take a sink as a bound type
parameter, and the function costs whatever the caller's sink costs. That is the
shape the trait exists for.

```novo ignore
fn served<S: lgsink.LgSink[e]>(to: S, f: LgFilter, status: Int) -> Result<Bool, LgSinkFault> [e]
    lgsink.emit_if(to, f, lgrecord.record(LgInfo, "http.server", "request served"))
```

**A program that owns the destination uses `lgsink.emit_all` or
`emit_and_flush`.** Both take a batch. `emit_and_flush` pushes whatever the sink
was holding, which is the step that decides whether the last seconds before a
crash reach the file.

**A test uses `lgsink.null_sink`.** Its implementation supplies the empty effect
list, so a test can assert that a caller's own effects did not widen.
`null_sink_at` refuses below a level, which is what exercises a caller's own
gating.

**A program that only wants the bytes calls `lgformat.render`.** Rendering costs
nothing, so a test can assert on an exact log line without capturing a file
descriptor. `render_into` appends to a buffer the caller already has, which is
how a sink turns a hundred records into one write.

**A program that only wants the last few records holds an `LgRing`.** Push
records into it, and call `lgring.dump_into` to spill them into a real sink when
something goes wrong. Debug logging then costs its write only on the failures.

## The rules a user needs

1. **A higher level number is louder.** `LgTrace` is 0 and `LgOff` is 5. A
   threshold admits what is at least as loud as itself. Call
   `lgrecord.level_at_least` rather than writing the comparison, which is the
   one written backwards most often.
2. **There are six levels, where `std.log` has five.** `LgTrace` is the extra
   one. It exists so that a record crossing a boundary from Rust's `log`, from
   `tracing` or from syslog has somewhere to go, and so that a filter set to
   debug-but-not-trace can be expressed.
3. **`LgOff` is a threshold and never a record's level.** `lgrecord.record`
   refuses nothing. A record built at `LgOff` is filtered out by every filter.
4. **A filter's longest matching prefix wins, and prefixes match whole
   segments.** `"ml"` governs `ml` and `ml.core`. It does not govern `mlx`. Rule
   order in a specification is therefore not a hidden meaning.
   `lgfilter.prefix_governs` is the rule on its own.
5. **`lgfilter.rule` replaces a rule with the same prefix.
   `lgfilter.parse_spec` refuses a duplicate target.** A program layering
   overrides wants the later call to win. A duplicate in a line a person wrote
   is a mistake they can see.
6. **The specification grammar is `env_logger`'s.** A bare level sets the
   default. Comma-separated `target=level` clauses add rules. Whitespace around
   a clause is ignored. See the `RUST_LOG` section of
   [`env_logger`](https://docs.rs/env_logger)'s documentation.
7. **Check the filter before building the record.** Interpolation in a message
   argument runs before the call, so a debug line in a hot loop costs its
   formatting whether or not it is emitted. `lgfilter.allows` is the per-record
   check. `lgfilter.cheapest_gate` is the one a loop hoists out.
8. **A sink fault is not a program error.** `emit` answers a `Result` so that a
   caller can look at it. Propagating it with `!` would make a failed log line
   abort the request it was describing.
9. **A sink renders the record itself, through the format it holds.** Nothing
   hands a sink a finished line. That is what lets one record reach a terminal
   in colour and a collector as JSON without the caller knowing there are two.
10. **`level`, `logger`, `msg` and `ts` are reserved in the JSON format.** A
    caller's field with one of those keys appears twice in the object. This
    package does not rename it. `lgrecord.reserved_or_duplicate_keys` reports
    it, and `lgrecord.reserved_keys` is the list.
11. **The JSON line's key set is fixed. Its key order is not.** A JSON object is
    unordered, so a consumer must read by name. See
    [JSON Lines](https://jsonlines.org/).
12. **logfmt quotes a value containing a space, an `=`, a quote or a control
    byte, and leaves every other value bare.**
    `lgformat.logfmt_needs_quoting` is the rule a consumer's parser has to
    agree with. See the [logfmt convention](https://brandur.org/logfmt).
13. **`lgformat.render` writes no trailing newline. `render_into` writes one.**
    A file sink needs the separator. A syslog datagram and a test comparing one
    line do not.
14. **A timestamp is a parameter, and `0.0` means there is none.** The value is
    Unix epoch seconds. The human and logfmt formats can write it as that
    number or as an RFC 3339 instant in UTC. The JSON format always writes the
    number.
15. **Colour is a field on the style, not a question this package asks.**
    Nothing here looks at a file descriptor. Set `LgHumanStyle.colour` and
    `LgHumanStyle.depth` from the party that owns the destination.
16. **A full ring drops the oldest record and counts it.**
    `lgring.dropped_count` is that count. A dump that does not report it reads
    as the whole story when it is not.
17. **`lgring.drain` empties the ring and hands back both halves.**
    `lgring.dump_into` does not empty it, so a dump that failed halfway has not
    lost the records it was reporting.
18. **The human format pads the level tag to column nine**, which is the column
    `std.log`'s own text format uses. Output from both surfaces lines up during
    a migration. `lgformat.tag_column` is the number.

## What is not included

- **Sinks that write.** A console, a file with rotation, and a ring behind a
  mutable slot all perform something, so they live in
  [logging-nv](https://novo-lang.org/packages/logging-nv). What is here is the
  trait, the generic emits, and the one implementation that costs nothing.
- **A clock.** Reading the time is an effect, and a record constructor that
  stamped itself would charge every caller for it. `lgrecord.stamped` takes the
  value.
- **A global logger.** A filter is a value the caller threads. One
  process-wide threshold cannot say "debug from the scheduler, error from the
  HTTP client", and two libraries setting it fight over one number.
- **A generic fan-out over two arbitrary sinks.** A function may bind exactly
  one effect parameter, so a signature over two sink types with two different
  effect sets cannot be written. `LgFanout` is the result type for a fan-out,
  and logging-nv's `emit_to_console_and_file` is the concrete pair. A caller
  pairing two other sinks calls `emit_all` twice.
- **A message-text filter.** `env_logger` can match on the rendered message.
  Matching on text would make the gate depend on the rendering, which this
  package defers until after the gate.
- **Local time.** The RFC 3339 stamp is UTC. A local rendering needs a timezone
  database and a clock, and this package has neither.
- **Running on a microcontroller.** A record holds a message, a target and a
  list of fields, all of which allocate. What a device emits is a deferred
  log's interned index, and turning that into a record happens on the host.
- **The device bridge.** Reading a deferred log needs four more packages. That
  module stays in logging-nv, so a library taking this package resolves two
  packages instead of seven.

## Related packages

- [logging-nv](https://novo-lang.org/packages/logging-nv) is the other half: the
  console sink, the rotating file sink, the ring behind a slot, the device
  bridge and the logger that ties them together. Take it when your program owns
  the destination. Take this package when your library does not. **A program
  takes one of the two, not both.** logging-nv declares the same types, the same
  trait and the same module names itself, and an assembly holding both is
  refused with `E2005`.
- [ansi-nv](https://novo-lang.org/packages/ansi-nv) supplies the colour
  attributes the human format uses, and the arithmetic that narrows a
  twenty-four-bit colour to the sixteen a real terminal has.
- [tracing-nv](https://novo-lang.org/packages/tracing-nv) records spans, which
  are events with a duration and a parent. A log line says what happened. A
  span says how long it took and what it was part of.
- `std.log` in the standard library is the no-dependency case. Its sink is an
  enum, so every logging call is charged the union over the variants, `[io]`,
  even when the installed sink discards. Its global functions keep one
  process-wide level.

## Tests

```bash
novo test tests                            # every suite
novo test tests/lgrecord_tests.nv          # the level order and the typed field
novo test tests/lgfilter_tests.nv          # longest prefix, and the segment rule
novo test tests/lgformat_tests.nv          # the three renderings and their escaping
novo test tests/lgsink_tests.nv            # the trait, and what its parameter costs
novo test tests/lgring_tests.nv            # the last N records, and the drop count
```

`novo test` fails on purpose today. Every assertion reaches a `not implemented:
logging-core-nv.<module>.<fn>` panic, because every body is a `todo()`. The
tests are the specification the implementation will have to satisfy.

The expected text comes from the conventions the formats are named after: the
logfmt convention for the `key=value` line, JSON Lines for the one-object-per-line
form, RFC 3339 for the timestamp spelling, and `env_logger`'s `RUST_LOG` grammar
for the filter specification. The level names and the nine-column tag are
`std.log`'s, so output from both surfaces aligns.

The suite asserts that `"ml"` governs `ml.core` and does not govern `mlx`, that
the level comparison admits what is at least as loud as the threshold, that a
caller's `msg` field is reported as a reserved key rather than renamed, that
logfmt quotes exactly the values a consumer's parser expects it to, and that a
full ring counts what it discarded. `tests/lgsink_tests.nv` also carries an
assertion the compiler makes rather than `test.assert`: a function declared with
an empty effect list emits through the null sink, and it would not compile if
the effect parameter did not do what this package claims.

## Implementation status

| Item | Implemented |
| --- | --- |
| `lgrecord.LgLevel`, `.LgValue`, `.LgField`, `.LgRecord` | declared |
| `lgrecord.level_num`, `.level_name`, `.level_tag`, `.level_parse`, `.level_at_least` | no |
| `lgrecord.value_kind`, and the five `field_*` builders | no |
| `lgrecord.record`, `.with_fields`, `.with_field`, `.stamped`, `.located` | no |
| `lgrecord.field_of`, `.has_timestamp`, `.reserved_or_duplicate_keys`, `.reserved_keys` | no |
| `lgfilter.LgRule`, `.LgFilter`, `.LgFilterFault` and `impl Error` | declared |
| `lgfilter.filter`, `.rule`, `.allows`, `.level_for`, `.governing_rule`, `.prefix_governs` | no |
| `lgfilter.parse_spec`, `.spec_of`, `.cheapest_gate`, `.redundant_rules` | no |
| `lgformat.LgHumanStyle`, `.LgStamp`, `.LgFormat`, `.LgTheme` | declared |
| `lgformat.human`, `.human_colour`, `.default_style`, `.logfmt`, `.json` | no |
| `lgformat.format_name`, `.format_parse`, `.default_theme`, `.level_attrs` | no |
| `lgformat.render`, `.render_with`, `.render_into`, `.value_text` | no |
| `lgformat.logfmt_needs_quoting`, `.logfmt_value`, `.json_object`, `.tag_column` | no |
| `lgsink.LgSink[e]`, `.LgSinkFault` and `impl Error`, `.LgNullSink`, `.LgFanout` | declared |
| `impl LgSink[] for LgNullSink`: `emit`, `flush`, `accepts` | no |
| `lgsink.null_sink`, `.null_sink_at`, `.emit_if`, `.emit_all`, `.emit_and_flush` | no |
| `lgring.LgRing` | declared |
| `lgring.ring`, `.push`, `.drain`, `.snapshot`, `.len`, `.dropped_count` | no |
| `lgring.is_full`, `.clear`, `.dump_into` | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
