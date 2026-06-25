# cmdit — Current State

> Refreshed every release. CLAUDE.md is preferences/process/procedures
> (durable); this file is **state** (volatile).

## Version

**0.2.0** — Unreleased (built 2026-06-25). Flag modifiers (M2): `cmdit_enum` /
`cmdit_repeat` / `cmdit_required` / `cmdit_range` / `cmdit_env` + the forced
companions `cmdit_get_enum` (index dispatch), `cmdit_seen`, and `cmdit_finalize`
(the impure env+required post-stage, kept out of the pure parse core). Entry
struct grown 48→104 B and ctx 96→104 B (both append-only; types stay
byte-compatible). Flag-named errors + enriched generated help. Smoke + **88 tests**
green; `dist/cmdit.cyr` regenerated. Grounded by a site-by-site re-survey of the
hand-rolled patterns across the consumer repos.

**0.1.0** — Unreleased (built 2026-06-25). The extraction cut: stdlib `lib/flags.cyr`
productized as a standalone distlib + the universal additions (materialize bridge,
auto `--help`/`--version`, generated help, `CMDIT_EXIT_*`, raw-argv escape;
`FLAGS_MAX` 32→64). Smoke + 26/26 tests green. Surfaced by the 2026-06-25 ecosystem
CLI review (`agnosticos/docs/development/planning/cmdit.md`).

## Toolchain

- **Cyrius pin**: `6.2.44` (in `cyrius.cyml [package].cyrius`)

## Source

- `src/cmdit.cyr` — the library (the `[lib]` module; forked from cyrius stdlib
  `lib/flags.cyr`, byte-compatible types/error-codes, renamed to `cmdit_*`, +
  materialize bridge / auto help+version / exit constants / raw-argv escape; 0.2.0
  flag modifiers). Entry struct 104 B, ctx 104 B (append-only). The pure
  `cmdit_parse_argv` core is `/proc`- and env-free; env + required run in the
  separate `cmdit_finalize` (chained by `cmdit_parse`).
- `programs/smoke.cyr` — real-argv demo + compile-link smoke (exercises
  enum/range/repeat/env + flag-named errors).
- `dist/cmdit.cyr` — generated bundle (`cyrius distlib`); consumers import this.

## Tests

- `tests/cmdit.tcyr` — **88/88**. The 26 pure-core cases (bool/int/str/positional,
  `=value`, `--`, lone `-`, unknown/missing-value/bad-int/bundled, auto
  help/version, defaults) plus the 0.2.0 modifier groups: enum (valid/`=value`/
  invalid→BAD_ENUM/default index), range (inclusive bounds/out-of-range), required
  (present/absent/bool absent-vs-false/seen), repeat (accumulate/order/last-wins/
  empty), env (purity boundary, CLI-wins, unset→default, bool presence,
  required-via-env, non-fatal bad value). Env positives guard on `getenv("HOME")`.
- `tests/cmdit.bcyr` / `.fcyr` — benchmark / fuzz stubs.

## Dependencies

- stdlib — string, fmt, alloc, io, vec, str, syscalls, **args**, assert, bench.
  Note: cmdit is **stdlib-only** (no external `[deps.X]`), so `cyrius build` does
  not auto-prepend stdlib — `programs/smoke.cyr` and `tests/cmdit.tcyr` include
  their stdlib (`lib/args.cyr`, `lib/string.cyr`, `lib/io.cyr`, …) explicitly.
  Consumers of `dist/cmdit.cyr` get stdlib via their own dep resolution: need
  `"args"` for the `cmdit_argv`/`cmdit_parse` /proc read, and now **`"io"`** for
  `getenv` (referenced unconditionally by `cmdit_parse` via `cmdit_finalize`;
  pure `cmdit_parse_argv`-only callers link it but never invoke it).

## Consumers

- **kii 1.1.0** — re-fold complete (2026-06-25): dropped its hand-rolled flag-set on
  stdlib `flags` + `build_argv_array` for `[deps.cmdit]`; `cmdit_new`/`cmdit_parse`/
  `cmdit_get_*`/`cmdit_positional` + auto `--help`/`--version`. 468/468 tests green,
  rendering verified. The first consumer + the faithful-extraction proof.

## Next

0.3.0 verb / subcommand dispatch (`cmdit_verb`/`cmdit_dispatch`, nested sub-verbs,
before-or-after globals, multiplexer) + the deferred `cmdit_require_positionals`.
See [`roadmap.md`](roadmap.md) + `agnosticos/docs/development/planning/cmdit.md`.
