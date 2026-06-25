# cmdit

> Sovereign **CLI / argument-parsing** library in [Cyrius](https://github.com/MacCracken/cyrius)
> — getopt-long shaped, zero external code. The one place AGNOS tools register
> flags, parse argv, and print `--help`/`--version`, instead of hand-rolling it
> on top of the bare `args` primitive.

**Status:** v0.3.0 (verb dispatch). **License:** GPL-3.0-only. Distributed as
`dist/cmdit.cyr` — consumers import it via `[deps.cmdit] modules = ["dist/cmdit.cyr"]`.

## Why

An ecosystem review (2026-06-25) found **~40 AGNOS binaries hand-roll** flag/help/
version parsing on the stdlib `args` primitive (`argc`/`argv`), and ~29 hand-write
their own `--help` walls that drift from the parse logic. A stdlib parser
(`lib/flags.cyr`) already existed and ~9 repos consumed it — but it lives in the
toolchain (can't grow `enum`/`repeat`/verb features without a cyrius release) and
the long tail never adopted it. **cmdit is `flags.cyr` productized as a standalone
distlib + extended**: the proven getopt-long core (byte-compatible types + error
codes) plus the three things every structured consumer copies by hand.

## What it does (0.1.0)

- **Typed flags** — `cmdit_bool` / `cmdit_int` / `cmdit_str`, short + long aliases.
- **getopt-long syntax** — `--name`, `--name=value`, `--name value`, `-x`, `-x value`,
  `--` terminator, lone `-` positional; positionals captured in argv order.
- **The argv materialize bridge** — `cmdit_argv` turns `argc()/argv(i)` into one
  contiguous `cstr*` array (the boilerplate every structured consumer re-copied).
- **Auto `--help`/`-h` + `--version`/`-V`** — registered for free; `cmdit_parse`
  short-circuits to `CMDIT_HELP` / `CMDIT_VERSION`.
- **Generated help** — `cmdit_help` prints `Usage:` + the flag list from the table
  (kills hand-written-help drift); `cmdit_version` centralizes the version line.
- **Standard errors + exit codes** — `cmdit_print_error` + `CMDIT_EXIT_OK/RUN/USAGE`
  (the `0`/`1`/`2` AGNOS userland convention as named constants).
- **Pure, testable core** — `cmdit_parse_argv(h, argc, argv)` never reads `/proc`
  or the environment, so tests drive synthetic argv. **Escape hatch**
  `cmdit_raw_argv` for non-getopt grammars (e.g. `dig`'s `@server`).

## Flag modifiers (0.2.0)

Five modifiers that subsume the patterns the ecosystem survey found hand-rolled
(all append-only on the byte-compatible core):

- **`cmdit_enum`** — choice flags: `cmdit_enum(h, 0, "color", "auto", "auto|always|never", help)`.
  Validated at parse (`CMDIT_ERR_BAD_ENUM` otherwise); `cmdit_get_enum` returns the
  0-based index for dispatch, `cmdit_get_str` the raw choice.
- **`cmdit_repeat`** — repeatable flags (`-H a -H b`): `cmdit_repeat_count` /
  `cmdit_repeat_get` read the accumulated list; `cmdit_get_str` is last-wins.
- **`cmdit_required`** — `cmdit_required(h, idx)` marker (any type incl. bool);
  absent → `CMDIT_ERR_REQUIRED_MISSING`. `cmdit_seen(h, idx)` queries presence.
- **`cmdit_range`** — `cmdit_range(h, idx, min, max)` inclusive int bounds, rejected
  (not clamped) → `CMDIT_ERR_OUT_OF_RANGE`.
- **`cmdit_env`** — `cmdit_env(h, idx, "NO_COLOR")` env fallback; CLI wins,
  bool = presence, bad values non-fatal.

The env + required checks run in **`cmdit_finalize`** (chained by `cmdit_parse`);
the pure `cmdit_parse_argv` core stays env-free. `cmdit_print_error` names the
offending flag and lists the valid enum set.

## Verb dispatch (0.3.0)

Subcommands by **identify-and-return** (switch on the id, like `cmdit_get_enum`):

```cyrius
var h = cmdit_new("tool");
cmdit_bool(h, 0, "verbose", "global flag (before OR after the verb)");
cmdit_verb(h, "add", "add a thing");          # id 0
cmdit_verb(h, "remove", "remove a thing");    # id 1
cmdit_verb_alias(h, 1, "rm");

var r = cmdit_dispatch(h);                     # one pass; globals bind before/after the verb
if (r == CMDIT_HELP)       { cmdit_help(h); return CMDIT_EXIT_OK; }      # global help + command list
if (r == CMDIT_RESULT_ERR) { cmdit_print_error(h); return CMDIT_EXIT_USAGE; }   # unknown command + list
var v = cmdit_verb_matched(h);                # -1 if no verb token
if (v == 0) {
    var vh = cmdit_new("tool add");           # per-verb scope: its own flags
    var f_name = cmdit_str(vh, 110, "name", 0, "");
    cmdit_parse_verb(vh, h);                  # re-parse the remainder slice (argv[0] = verb name)
    ...
}
```

- **Globals before OR after the verb** resolve in one pass (the ark double-scan fix).
- **`--help`/`--version` defer** past the verb — `tool add --help` shows the
  subcommand's help; `tool --help` shows global help + the command list.
- **Nested sub-verbs** are re-entry: dispatch a child handle on `cmdit_verb_argv(h)`.
- **`cmdit_basename`** + `cmdit_raw_argv` give the argv[0] busybox multiplexer, caller-side.
- **`cmdit_require_positionals(h, min)`** gates minimum positional count.

See `programs/verbs.cyr` for a runnable demo.

## Usage

```cyrius
fn main(): i64 {
    var h    = cmdit_new("mytool");                              # auto --help/-h, --version/-V
    var f_o  = cmdit_str (h, 111, "output", 0, "output file");   # -o / --output
    var f_n  = cmdit_int (h, 110, "count", 1, "repeat count");   # -n / --count
    var r = cmdit_parse(h);
    if (r == CMDIT_HELP)       { cmdit_help(h);                  return CMDIT_EXIT_OK; }
    if (r == CMDIT_VERSION)    { return cmdit_version(h, "mytool 1.0"); }
    if (r == CMDIT_RESULT_ERR) { cmdit_print_error(h); cmdit_help(h); return CMDIT_EXIT_USAGE; }
    var out = cmdit_get_str(h, f_o);
    var n   = cmdit_get_int(h, f_n);
    var i = 0;
    while (i < cmdit_positional_count(h)) { ...cmdit_positional(h, i)...; i = i + 1; }
    return CMDIT_EXIT_OK;
}
fn _entry(): i64 { var r = main(); sys_exit(r); return 0; }
_entry();   # BARE call — NOT `var r = main();` (agnos init-rsp capture; see docs)
```

Consuming repos add to `cyrius.cyml`:

```cyml
[deps.cmdit]
git = "https://github.com/MacCracken/cmdit"
tag = "0.3.0"
modules = ["dist/cmdit.cyr"]
```

`cmdit_argv`/`cmdit_parse` need stdlib `"args"` in the consumer's `[deps] stdlib`,
and `"io"` (for `getenv`, referenced by `cmdit_parse` via `cmdit_finalize`).
Callers that drive `cmdit_parse_argv` directly don't invoke either.

## Roadmap

- **0.1.0** — the extraction: getopt-long core + materialize bridge + auto help/
  version + exit constants + raw-argv escape (this cut).
- **0.2.0** — `cmdit_enum` (choice flags + index), `cmdit_repeat`, `cmdit_required`,
  `cmdit_range`, `cmdit_env`, `cmdit_finalize` + flag-named errors.
- **0.3.0** — verb / subcommand dispatch (`cmdit_verb` / `cmdit_verb_alias` /
  `cmdit_dispatch`), before-or-after-verb globals, per-verb scopes + nested
  re-entry, `cmdit_require_positionals`, `cmdit_basename` multiplexer (this cut).
- **→ v1.0** — API freeze, benchmarks, security-audit pass.

First consumer (re-fold): **kii** drops its in-repo flag-set wrapper for cmdit.

## Build

```sh
cyrius build programs/smoke.cyr build/cmdit-smoke   # compile-link smoke / demo
cyrius distlib                                       # produce dist/cmdit.cyr
cyrius test                                          # run tests/*.tcyr  (26/26)
```
