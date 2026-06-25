# cmdit

> Sovereign **CLI / argument-parsing** library in [Cyrius](https://github.com/MacCracken/cyrius)
> — getopt-long shaped, zero external code. The one place AGNOS tools register
> flags, parse argv, and print `--help`/`--version`, instead of hand-rolling it
> on top of the bare `args` primitive.

**Status:** v0.1.0 (the extraction cut). **License:** GPL-3.0-only. Distributed as
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
- **Pure, testable core** — `cmdit_parse_argv(h, argc, argv)` never reads `/proc`,
  so tests drive synthetic argv. **Escape hatch** `cmdit_raw_argv` for non-getopt
  grammars (e.g. `dig`'s `@server`).

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
tag = "0.1.0"
modules = ["dist/cmdit.cyr"]
```

`cmdit_argv`/`cmdit_parse` need stdlib `"args"` in the consumer's `[deps] stdlib`.
Callers that pass their own array to `cmdit_parse_argv` don't.

## Roadmap

- **0.1.0** — the extraction: getopt-long core + materialize bridge + auto help/
  version + exit constants + raw-argv escape (this cut).
- **0.2.0** — `cmdit_enum` (choice flags), `cmdit_repeat`, `cmdit_required`,
  `cmdit_range`, `cmdit_env` (the new parse-loop modifiers).
- **0.3.0** — verb / subcommand dispatch (`cmdit_verb` / `cmdit_dispatch`),
  nested sub-verbs, before-or-after global flags.

First consumer (re-fold): **kii** drops its in-repo flag-set wrapper for cmdit.

## Build

```sh
cyrius build programs/smoke.cyr build/cmdit-smoke   # compile-link smoke / demo
cyrius distlib                                       # produce dist/cmdit.cyr
cyrius test                                          # run tests/*.tcyr  (26/26)
```
