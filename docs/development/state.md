# cmdit — Current State

> Refreshed every release. CLAUDE.md is preferences/process/procedures
> (durable); this file is **state** (volatile).

## Version

**0.1.0** — Unreleased (built 2026-06-25). The extraction cut: stdlib `lib/flags.cyr`
productized as a standalone distlib + the universal additions (materialize bridge,
auto `--help`/`--version`, generated help, `CMDIT_EXIT_*`, raw-argv escape;
`FLAGS_MAX` 32→64). Smoke + **26/26 tests** green; `dist/cmdit.cyr` generated.
Surfaced by the 2026-06-25 ecosystem CLI review (`agnosticos/docs/development/planning/cmdit.md`).

## Toolchain

- **Cyrius pin**: `6.2.44` (in `cyrius.cyml [package].cyrius`)

## Source

- `src/cmdit.cyr` — the library (the `[lib]` module; forked from cyrius stdlib
  `lib/flags.cyr`, byte-compatible types/error-codes, renamed to `cmdit_*`, +
  materialize bridge / auto help+version / exit constants / raw-argv escape).
- `programs/smoke.cyr` — real-argv demo + compile-link smoke.
- `dist/cmdit.cyr` — generated bundle (`cyrius distlib`); consumers import this.

## Tests

- `tests/cmdit.tcyr` — 26/26 over the pure `cmdit_parse_argv` core (bool/int/str/
  positional, `=value`, `--` terminator, lone `-`, unknown/missing-value/bad-int/
  bundled errors, auto help/version short-circuit, default preservation).
- `tests/cmdit.bcyr` / `.fcyr` — benchmark / fuzz stubs.

## Dependencies

- stdlib — string, fmt, alloc, io, vec, str, syscalls, **args**, assert, bench.
  Note: cmdit is **stdlib-only** (no external `[deps.X]`), so `cyrius build` does
  not auto-prepend stdlib — `programs/smoke.cyr` and `tests/cmdit.tcyr` include
  their stdlib (`lib/args.cyr`, `lib/string.cyr`, …) explicitly. Consumers of
  `dist/cmdit.cyr` get stdlib via their own dep resolution (need `"args"` for the
  `cmdit_argv`/`cmdit_parse` /proc read).

## Consumers

_None yet — **kii** is the planned first re-fold (drops its in-repo flag-set
wrapper for `[deps.cmdit]`)._

## Next

0.2.0 flag modifiers (enum/repeat/required/range/env); 0.3.0 verb dispatch.
See [`roadmap.md`](roadmap.md) + `agnosticos/docs/development/planning/cmdit.md`.
