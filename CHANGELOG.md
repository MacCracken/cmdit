# Changelog

Format: [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased]

## [0.1.0] - Unreleased

**The extraction cut.** cmdit is the stdlib flag parser (`lib/flags.cyr`)
productized as a standalone distlib + the universal boilerplate every structured
consumer hand-rolled. Surfaced by the 2026-06-25 ecosystem CLI review (no
dedicated CLI lib existed; ~40 binaries hand-roll `argc`/`argv`).

### Added
- **getopt-long parser core** — forked from cyrius stdlib `lib/flags.cyr`
  (byte-compatible types `CMDIT_BOOL/INT/STR` and error codes), renamed to the
  `cmdit_*` surface. `cmdit_new` / `cmdit_bool` / `cmdit_int` / `cmdit_str` /
  `cmdit_parse_argv` / `cmdit_get_*` / `cmdit_positional*` / `cmdit_error`.
  Syntax: `--name`, `--name=value`, `--name value`, `-x`, `-x value`, `--`
  terminator, lone `-` positional.
- **`cmdit_argv`** — the argc/argv → contiguous NULL-terminated `cstr*` materialize
  bridge (calls `args_init` once, cached on the ctx). Absorbs the
  `build_argv_array` / `_materialize_argv` boilerplate found in kii/yo/agnova/dig/kriya.
- **`cmdit_parse`** — convenience: materialize then parse the process argv.
- **Auto `--help`/`-h` + `--version`/`-V`** — registered by `cmdit_new`;
  `cmdit_parse*` short-circuit to `CMDIT_HELP` / `CMDIT_VERSION`.
- **`cmdit_help`** — generated `Usage:` + flag list (kills hand-written-help drift);
  **`cmdit_version`** — centralized machine-readable version line; **`cmdit_print_error`**
  — standard `<prog>: <message>` stderr decode.
- **`CMDIT_EXIT_OK/RUN/USAGE`** — the `0`/`1`/`2` AGNOS userland exit convention as
  named constants.
- **`cmdit_raw_argv` / `cmdit_raw_argc`** — escape hatch for non-getopt grammars.
- `FLAGS_MAX` raised 32 → 64 (attn11 has ~35 flags, exceeding the stdlib cap).
- **Tests** — `tests/cmdit.tcyr`, 26/26 over the pure `cmdit_parse_argv` core
  (bool/int/str/positional, `=value`, `--` terminator, lone `-`, unknown / missing-value
  / bad-int / bundled errors, auto help/version short-circuit, default preservation).
- **Smoke/demo** — `programs/smoke.cyr` exercises real-argv parsing end to end.

### Deferred
- 0.2.0: `cmdit_enum` / `cmdit_repeat` / `cmdit_required` / `cmdit_range` / `cmdit_env`.
- 0.3.0: verb / subcommand dispatch.
- Out of scope: bundled shorts `-abc`, attached short value `-xvalue`, count `-vvv`,
  `--no-foo` negation, interactive/REPL command DSLs.

Toolchain pin: cyrius 6.2.44.
