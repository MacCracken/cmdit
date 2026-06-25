# cmdit — Current State

> Refreshed every release. CLAUDE.md is preferences/process/procedures
> (durable); this file is **state** (volatile).

## Version

**0.3.0** — Unreleased (built 2026-06-25). Verb / subcommand dispatch (M3):
`cmdit_verb` / `cmdit_verb_alias` / `cmdit_dispatch` (identify-and-return) +
`cmdit_verb_matched` / `cmdit_verb_argc/argv` + `cmdit_parse_verb` (per-verb scope
re-entry; nested verbs are re-entry, not an in-core tree) + `cmdit_basename` (argv[0]
multiplexer) + `cmdit_verbs_help` + the deferred `cmdit_require_positionals`.
Globals bind before OR after the verb in one pass (the ark double-scan fix); the
pure `cmdit_parse_argv` core is untouched and `cmdit_dispatch_argv` is its pure
sibling. Ctx grown 104→160 B (append-only; entry struct unchanged). **157 tests**
green; `dist/cmdit.cyr` regenerated; demo `programs/verbs.cyr`. Grounded by a
site-by-site re-survey of the subcommand tools.

**0.2.0** — Unreleased (built 2026-06-25). Flag modifiers (M2): `cmdit_enum` /
`cmdit_repeat` / `cmdit_required` / `cmdit_range` / `cmdit_env` + `cmdit_get_enum`
(index dispatch), `cmdit_seen`, and `cmdit_finalize` (the impure env+required
post-stage). Entry struct grown 48→104 B; flag-named errors + enriched help. 88 tests.

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
  flag modifiers; 0.3.0 verb dispatch). Entry struct 104 B, ctx 160 B (append-only);
  verbs are a separate lazy-allocated table. The pure `cmdit_parse_argv` core is
  `/proc`- and env-free; `cmdit_dispatch_argv` is its pure verb-dispatch sibling;
  env + required run in `cmdit_finalize` (chained by `cmdit_parse`/`cmdit_dispatch`).
- `programs/smoke.cyr` — single-command demo (enum/range/repeat/env + flag-named errors).
- `programs/verbs.cyr` — verb-dispatch demo (verb/alias/global-before-after/
  per-verb scope/require_positionals/unknown-verb list).
- `dist/cmdit.cyr` — generated bundle (`cyrius distlib`); consumers import this.

## Tests

- `tests/cmdit.tcyr` — **157/157**. The 26 pure-core cases + the 0.2.0 modifier
  groups (enum/range/required/repeat/env + purity boundary) + the 0.3.0 verb groups:
  `cmdit_basename`, verb match/alias/unknown-verb, no-verb→-1, global flag
  before/after the verb (+ value consumption), unknown-flag deferral, help
  before→HELP / after→deferred, `--` handling, remainder re-parse in a per-verb
  handle, nested sub-verb re-entry, `require_positionals`, global-required at
  finalize. Env positives guard on `getenv("HOME")`.
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

Toward v1.0: API-freeze pass (every exported symbol documented + tested), benchmarks
in `docs/benchmarks.md`, a security-audit pass, and CHANGELOG completeness. The
0.1→0.3 verb/modifier arc is feature-complete; remaining work is hardening +
adoption (kii green; the Tier-1/2/3 drop-ins are demand-gated). See
[`roadmap.md`](roadmap.md) + `agnosticos/docs/development/planning/cmdit.md`.
