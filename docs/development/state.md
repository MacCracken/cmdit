# cmdit — Current State

> Refreshed every release. CLAUDE.md is preferences/process/procedures
> (durable); this file is **state** (volatile).

## Version

**1.1.0** — built 2026-06-25. Append-only, non-breaking on the frozen 1.0.0 surface:
adds **`cmdit_help_flags(h)`**, the table-only flag renderer (just the `  -x, --long
<type>` rows, no Usage/Options wrapper) for tools that frame their own help (intro +
custom usage + examples) around a generated flag list. `cmdit_help` composes it; its
output is unchanged. Surfaced by the anuenue adoption pilot (the second rich-help
consumer after the re-survey under-counted the need). **231 tests**; `dist` regenerated.

**1.0.0** — Public API frozen (built 2026-06-25). M4 completeness delta + the v1.0
freeze, grounded by a 23-repo re-survey of hand-rolled CLI code (confirmed the surface
complete) and an adversarial security/readiness audit. Added: `cmdit_help_short` /
`cmdit_version_short` (remap/disable the auto `-h`/`-V`), `cmdit_metavar` + shown string
defaults in help, `cmdit_require_positionals_max` / `_exact` (`TOO_MANY_POSITIONAL = 10`).
Fixed (audit): `_cmdit_parse_int` overflow range-bypass, unguarded getter idx. Entry
struct grown 104→112 B (metavar at +104; append-only). The whole output/renderer surface
is now tested. **230 tests** green; benchmarks ([`benchmarks.md`](benchmarks.md)) +
security audit ([`audit/2026-06-25-audit.md`](audit/2026-06-25-audit.md)); the
public/internal constant boundary is delimited in [`../adr/0003-v1-freeze.md`](../adr/0003-v1-freeze.md).
`dist/cmdit.cyr` regenerated.

**0.3.0** — built 2026-06-25. Verb / subcommand dispatch (M3):
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
  flag modifiers; 0.3.0 verb dispatch; 1.0.0 `help_short`/`version_short` + `metavar`
  + `require_positionals_max`/`_exact`). Entry struct 112 B, ctx 160 B (append-only);
  verbs are a separate lazy-allocated table. The pure `cmdit_parse_argv` core is
  `/proc`- and env-free; `cmdit_dispatch_argv` is its pure verb-dispatch sibling;
  env + required run in `cmdit_finalize` (chained by `cmdit_parse`/`cmdit_dispatch`).
- `programs/smoke.cyr` — single-command demo (enum/range/repeat/env + flag-named errors).
- `programs/verbs.cyr` — verb-dispatch demo (verb/alias/global-before-after/
  per-verb scope/require_positionals/unknown-verb list).
- `dist/cmdit.cyr` — generated bundle (`cyrius distlib`); consumers import this.

## Tests

- `tests/cmdit.tcyr` — **230/230**. The 0.1 pure-core + 0.2 modifier + 0.3 verb
  groups, plus the 1.0.0 hardening: the full output/renderer surface
  (`cmdit_print_error` over every `CmditErr` branch incl. the short-flag sbuf path,
  `cmdit_help`, `cmdit_verbs_help`, `cmdit_version`), int-overflow rejection +
  i64-max boundary, getter idx guards, negative-int parse, unknown-short /
  short-missing-value, enum prefix rejection, dispatch pre-verb-unknown + dispatch
  missing-value, `CMDIT_FLAGS_MAX`/`CMDIT_VERB_MAX` cap returns, `cmdit_new(0)`
  argv[0] adoption, and the C5/C3/C1 groups (`help_short`/`version_short` remap+disable,
  metavar, `require_positionals_max`/`_exact`). Env positives guard on `getenv("HOME")`.
- `tests/cmdit.bcyr` — **8 real benchmarks** (`cmdit_new` floor, register/parse
  subtraction, enum re-walk, dispatch before/after-verb parity, repeat accumulate);
  see [`benchmarks.md`](benchmarks.md). `tests/cmdit.fcyr` — fuzz stub.

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

**v1.0.0 is freeze-ready — all six roadmap criteria met** (API frozen + documented,
230/230, benchmarks, kii green, CHANGELOG dated, security audit). The remaining action
is the user's: tag `1.0.0` (git is user-owned). Post-v1: adoption (the Tier-1/2/3
drop-ins are demand-gated; kii green) and the confirmed-deferred v2 backlog (mutex/
required-if sugar, optional-value flags, bundled shorts, `--no-foo`, named-positional
help, config cascades — see [`../adr/0003-v1-freeze.md`](../adr/0003-v1-freeze.md)). A
known, deferred micro-opt: `cmdit_new`'s byte-at-a-time entry memzero (the ~13.8 µs
handle-construction floor) could be word-at-a-time (see [`benchmarks.md`](benchmarks.md)).
See [`roadmap.md`](roadmap.md) + `agnosticos/docs/development/planning/cmdit.md`.
