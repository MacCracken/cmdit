# cmdit

> Sovereign **CLI / argument-parsing** library in [Cyrius](https://github.com/MacCracken/cyrius)
> — getopt-long shaped, zero external code. The one place AGNOS tools register
> flags, parse argv, and print `--help`/`--version`, instead of hand-rolling it
> on top of the bare `args` primitive.

**Status:** v1.3.4 — public API frozen at 1.0.0; everything since is append-only on it
(1.1.0 `cmdit_help_flags`; 1.2.0 verb introspection + shell completions; 1.2.2 verb
command-line forwarding). 1.2.5–1.2.6 are toolchain refreshes; 1.3.0 makes `cmdit_new` ~6×
faster; 1.3.1 stops two silent misparses (positionals past 128, `--bool=0`); 1.3.2 is an
audit sweep with a completion-script security fix; 1.3.3 adds `cmdit_err_token` /
`cmdit_err_value` and error lines that name the offending text; 1.3.4 adds POSIX
`cmdit_stop_at_positional`. **1.2.4 is a security release** — the completion generator
interpolated the program name, which on the `cmdit_new(0)` path comes from `argv[0]`, raw
into the scripts it emits; it also closes three memory-safety/contract defects. No API
change. **Consumers on any earlier version that generate completions should update.**
**License:** GPL-3.0-only. Distributed as
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
  `--` terminator, lone `-` positional; positionals captured in argv order, with no
  count limit (1.3.1). A bool given `--flag=value` is an error (1.3.1), as in getopt_long.
- **The argv materialize bridge** — `cmdit_argv` turns `argc()/argv(i)` into one
  contiguous `cstr*` array (the boilerplate every structured consumer re-copied).
- **Auto `--help`/`-h` + `--version`/`-V`** — registered for free; `cmdit_parse`
  returns `CMDIT_HELP` / `CMDIT_VERSION` instead of `CMDIT_OK`. The whole argv is still
  scanned first, so an error anywhere wins (`--help --bogus` is `CMDIT_ERR_UNKNOWN`).
- **Generated help** — `cmdit_help` prints `Usage:` + the flag list from the table
  (kills hand-written-help drift); `cmdit_version` centralizes the version line.
- **Standard errors + exit codes** — `cmdit_print_error` + `CMDIT_EXIT_OK/RUN/USAGE`
  (the `0`/`1`/`2` AGNOS userland convention as named constants). The error line names
  what the user typed (`tool: unknown flag: --bogus`, `tool: --count: invalid integer value
  'abc'`), with control bytes in any argv-derived text shown as `?` (1.3.3).
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
  subcommand's help, even when a global flag is `cmdit_required` (1.3.2); `tool --help`
  shows global help + the command list. `cmdit_help_short` / `cmdit_version_short` are per
  handle, so remap them on each verb handle too.
- **`--` before the verb** ends *global* flags only: `tool -- add --verbose` does not apply
  `--verbose` as a global, and the verb scope parses it. After the verb, `--` reaches the
  verb scope as its own terminator.
- **`cmdit_verb_trailing_after(h, v, n)`** makes everything after the verb's first `n`
  positionals a verbatim payload (`exec <container> <cmd>...`). Only positionals count
  toward `n`, so verb flags and globals before them still parse (1.3.2), and a `--` right at
  the threshold is consumed, kubectl-style (1.3.2).
- **`cmdit_stop_at_positional(h, 1)`** on a verb handle makes the verb the first operand:
  globals apply only before it, and a global after it goes to the verb's scope (1.3.4).
- **One-pass caveats:** a global value flag right before the verb can take the verb as its
  value (`tool --config build install`), and a verb value flag whose separate value spells a
  global flag gives the global (`tool add --name --verbose`). Write `--config=build` /
  `--name=--verbose` to be unambiguous.
- **Nested sub-verbs** are re-entry: dispatch a child handle on `cmdit_verb_argv(h)`.
- **`cmdit_basename`** + `cmdit_raw_argv` give the argv[0] busybox multiplexer, caller-side.
- **`cmdit_require_positionals(h, min)`** gates minimum positional count.

See `programs/verbs.cyr` for a runnable demo.

## v1.0 completeness delta

The final surface a 23-repo re-survey of hand-rolled CLI code found still uncovered
(all append-only — `CMDIT_BOOL/INT/STR` = {0,1,2} and every prior offset/error code
are untouched; entry grew 104→112 B for the metavar):

- **`cmdit_help_short(h, ch)` / `cmdit_version_short(h, ch)`** — remap (any ASCII
  byte) or disable (`ch = 0`) the auto `-h` / `-V` short letters while keeping the
  `--help`/`--version` long forms. Frees `-h` for a consumer's own flag (the
  `df -h`/`ls -h` "human-readable" convention that hard-binding `-h`=help fights).
- **`cmdit_metavar(h, idx, name)`** — override a value flag's generated help
  placeholder (`--out <FILE>` instead of `<str>`); cosmetic, parsing-neutral. The
  generated help now also shows the registered default of string flags
  (`(default: …)`).
- **`cmdit_require_positionals_max(h, max)` / `cmdit_require_positionals_exact(h, n)`**
  — the upper-bound and both-bounds mirrors of `cmdit_require_positionals`; over the
  max yields `CMDIT_ERR_TOO_MANY_POSITIONAL` (`too many arguments`, exit 2).

## API reference (frozen surface)

The full public surface. **Frozen at v1.0** — names, signatures, and semantics are
stable; `CMDIT_*_SIZE` / `CMDIT_*_MAX` / `CMDIT_ATTR_*` are internal layout/cap
constants, not API (do not depend on their values).

- **Lifecycle:** `cmdit_new(prog)` → handle; `cmdit_argv(h)` (materialize bridge),
  `cmdit_parse(h)` / `cmdit_parse_argv(h, argc, argv)` (pure), `cmdit_finalize(h)`,
  `cmdit_dispatch(h)` / `cmdit_dispatch_argv(h, argc, argv)` (pure). A handle is
  **single-parse**: build a fresh one per argv. A second parse on the same handle resets
  only the error fields (1.2.4); positionals, flag values, the matched verb and a verb
  handle's trailing-verbatim mark carry over. A 0 handle (`cmdit_new`'s OOM result) is safe
  everywhere and reports `CMDIT_ERR_NOMEM` (1.3.2).
- **Register:** `cmdit_bool` / `cmdit_int` / `cmdit_str` / `cmdit_enum` /
  `cmdit_repeat`; modifiers `cmdit_required` / `cmdit_range` / `cmdit_env` /
  `cmdit_metavar`; verbs `cmdit_verb` / `cmdit_verb_alias`; auto-flag config
  `cmdit_help_short` / `cmdit_version_short`; `cmdit_stop_at_positional(h, on)` (1.3.4:
  POSIX — option recognition stops at the first positional, for tools whose operands are
  another command line: `sudo`-shaped, `xargs`, `env`, `nice`).
- **Read:** `cmdit_get_bool` / `cmdit_get_int` / `cmdit_get_str` / `cmdit_get_enum`,
  `cmdit_seen`, `cmdit_repeat_count` / `cmdit_repeat_get`, `cmdit_positional` /
  `cmdit_positional_count`, `cmdit_verb_matched` / `cmdit_verb_argc` /
  `cmdit_verb_argv`, `cmdit_raw_argv` / `cmdit_raw_argc`, `cmdit_basename`.
- **Validate / errors:** `cmdit_require_positionals` / `_max` / `_exact`,
  `cmdit_error` (last `CmditErr`, `CMDIT_ERR_NONE` 0 … `CMDIT_ERR_ARGV` 13),
  `cmdit_err_flag` (offending flag index), `cmdit_err_token` (the argv token behind the
  error, as typed) and `cmdit_err_value` (the value a BAD_INT / BAD_ENUM / OUT_OF_RANGE /
  UNEXPECTED_VALUE rejected) — both 1.3.3, interior pointers into argv or 0.
- **Output:** `cmdit_help` (Usage + flag list) / `cmdit_help_flags` (table-only rows,
  for tools framing their own help) / `cmdit_verbs_help`, `cmdit_version`, `cmdit_print_error`.
- **Result codes:** `CMDIT_OK` / `CMDIT_HELP` / `CMDIT_VERSION` / `CMDIT_RESULT_ERR`.
  **Exit codes:** `CMDIT_EXIT_OK` (0) / `CMDIT_EXIT_RUN` (1) / `CMDIT_EXIT_USAGE` (2).
  Note the two namespaces collide by value on purpose (`CMDIT_OK` = `CMDIT_EXIT_OK`
  = 0; `CMDIT_HELP` = `CMDIT_EXIT_USAGE` = 2) — they are distinct concepts (parse
  result vs process exit); never substitute one for the other.

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
tag = "1.0.0"
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
  re-entry, `cmdit_require_positionals`, `cmdit_basename` multiplexer.
- **1.0.0** — the completeness delta (`cmdit_help_short`/`_version_short`,
  `cmdit_metavar` + shown defaults, `cmdit_require_positionals_max`/`_exact`) plus
  the freeze: API frozen, **230/230 tests**, [benchmarks](docs/benchmarks.md), and a
  [security audit](docs/audit/2026-06-25-audit.md) pass (this cut).
- **1.1.0** — `cmdit_help_flags`, the table-only flag renderer for tools that frame
  their own help around a generated flag list.
- **1.2.0** — verb introspection (`cmdit_verb_count` / `_name_at` / `_help_at` /
  `_is_alias` / `_canonical_at`) + `cmdit_completions` for bash/zsh/fish.
- **1.2.1** — completion-generator fixes: the verb-position guard, and
  whitelist-filtered names (`compgen -W` re-expands, so escaping alone cannot hold).
- **1.2.2** — verb command-line forwarding: `--` is forwarded past the dispatcher, and
  `cmdit_verb_trailing_after` takes a verbatim trailing argument list.
- **1.2.3** — maintenance: toolchain pin 6.5.35 + vendored stdlib refresh, no source change.
- **1.2.4** — **security + P-1 audit**: the completion generator no longer interpolates the
  program name raw (an `argv[0]`-derived name could inject shell into the emitted script);
  plus the flag-modifier and repeat-accessor bounds guards, the `err_entry` reset, and
  alloc-failure/input validation. **345/345 tests**, every fix mutation-proven.
- **1.2.5** — maintenance: toolchain pin 6.6.2, no source change.
- **1.2.6** — maintenance: toolchain pin 6.6.20 + full vendored stdlib refresh; `dist/`
  now carries a compile-verified `# Requires` block and a 4-leaf `cmdit.deps` sidecar.
- **1.3.0** — toolchain pin 6.7.9; `cmdit_new` **~6× faster** (1.97 µs, was ~11.5) by zeroing
  the entry table a word at a time; `cmdit_argv` linear on Linux (20,000 args: 2.6 s → 11 ms);
  **357/357 tests**; a real fuzz harness over both parse cores (the old one
  was a stub); benchmarks re-captured with a 1.0.0 → 1.3.0 bisect. No API change.
- **1.3.1** — stdlib parity: positionals and repeat lists **grow** instead of silently dropping
  past 128 / 64, and a bool given `=value` is `CMDIT_ERR_UNEXPECTED_VALUE` (11) instead of
  turning the flag on. A refused growth is `CMDIT_ERR_NOMEM` (12). **400/400 tests**, every
  change mutation-proven.
- **1.3.2** — **audit sweep + security**: a program name starting with `-` (attacker-chosen
  `argv[0]`) was read as options by `complete` in the generated completion script; a command
  line too long or impossible to read whole parsed as a silent prefix, now
  `CMDIT_ERR_ARGV` (13). Plus slot-count wrap guards, 0-handle safety on every entry point,
  immutable layout constants, the zsh/fish name whitelist, five verb-dispatch contract fixes,
  and a ~3× faster verb-tool construction. **503/503 tests**, 33/33 repairs mutation-proven;
  [audit report](docs/audit/2026-10-14-audit.md).
- **1.3.3** — `cmdit_err_token` / `cmdit_err_value`: the argv token and the rejected value
  behind the last error (seven tools re-scanned argv for these). `cmdit_print_error` now
  prints them, with argv-derived text escaped. **562/562 tests**, 18/18 mutation-proven.
- **1.3.4** — `cmdit_stop_at_positional(h, on)`: POSIX stop at the first positional (a lone
  `-` included); a `--` after it stays data, unlike the per-verb trailing mark. Unblocks
  shakti and kriya's `xargs`/`env`. **596/596 tests**, 7/7 mutation-proven.

Consumers: **kii**, **anuenue**, **ifran**, **stiva**, **dig**, **commandress** — all on `[deps.cmdit]`.

## Build

```sh
cyrius build programs/smoke.cyr build/cmdit-smoke   # compile-link smoke / demo
cyrius distlib                                       # produce dist/cmdit.cyr
cyrius test                                          # run tests/*.tcyr  (596/596)
cyrius build tests/cmdit.bcyr build/cmdit-bench      # build + run the benchmarks
```
