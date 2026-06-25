# 0002 — Verb dispatch by identify-and-return, with nesting as re-entry

**Status**: Accepted
**Date**: 2026-06-25

## Context

The M3 milestone adds subcommand/verb dispatch — `tool add ...`, `tool remote
add ...`, the `kriya`-style argv[0] multiplexer — to cover the ~9 subcommand
tools the ecosystem survey identified (ark, sit, takumi, hapi, phylax, agnova,
hadara, hoosh, cyim). A site-by-site re-survey of their hand-rolled dispatch
forced several decisions:

- All of them are an `streq(argv[1], ...)` if/elif chain that **identifies** the
  verb and then runs inline code or a per-verb `cmd_*` handler. Their handler
  signatures are irreconcilable — ark returns a command object, takumi/hapi/sit
  return `i64`, hadara passes typed positionals. There is no one signature cmdit
  could call.
- Global flags appear **before or after** the verb (`tool --verbose add` and
  `tool add --verbose`). ark literally scans argv twice to handle this — the
  "double-scan" the roadmap names as the thing to fix.
- Production nesting is depth ≤ 2 (`phylax rules list`); nobody has a deep verb
  tree.
- The pure `cmdit_parse_argv` core and its synthetic-argv test suite must stay
  intact (the 0.1/0.2 contract).

## Decision

**Identify-and-return dispatch over a separate lazy verb table, with a pure
dispatch sibling and nesting delivered as re-entry — not handler fn-ptrs and not
an in-core verb tree.**

- `cmdit_verb` / `cmdit_verb_alias` register rows in a lazily-allocated verb
  table (ctx +104..); the entry/flag struct is untouched. `cmdit_dispatch_argv`
  is a **new pure sibling** of `cmdit_parse_argv` (not an edit to it). It makes a
  **single pass**: registered global flags are applied wherever they appear
  (before or after the verb — the double-scan fix), the first non-flag token is
  matched against the table, and everything else is compacted into a
  globals-removed **remainder slice** (`argv[0]` = verb name).
- The caller switches on `cmdit_verb_matched(h)` (the canonical id, mirroring
  `cmdit_get_enum`) and re-parses the remainder in a fresh per-verb handle
  (`cmdit_parse_verb`). cmdit never calls a handler.
- **Nested sub-verbs are re-entry**: a verb handler builds its own handle, calls
  `cmdit_dispatch_argv` on the parent's remainder, and recurses. Unbounded depth,
  zero extra API. The roadmap's "nested sub-verbs" wording bent to this.
- `--help`/`--version` **defer** when they follow the verb (so `tool build
  --help` reaches verb-scoped help) and fire globally when they precede it.
- The argv[0] multiplexer is `cmdit_basename` + the existing `cmdit_raw_argv`,
  caller-side and opt-in.

## Consequences

- **Positive** — the pure core and 0.1/0.2 tests are byte-for-byte unaffected; no
  Cyrius function-pointer dependency; consumers keep their heterogeneous handler
  styles; global flags become position-insensitive in one pass; every flag
  modifier (enum/range/repeat/required/env) works unchanged in verb scope because
  the remainder is a valid argv the unchanged core parses.
- **Negative** — re-entry allocates a fresh handle (a few KB) per matched verb
  (fine for one-shot CLIs; noted for hot multiplexers); a space-separated global
  STR flag placed immediately before the verb can swallow the verb token
  (`tool --config build install` → `--config=build`), an inherent getopt
  ambiguity resolved with `--config=build` or after-verb placement; ctx grows
  104→160 B.
- **Neutral** — the verb-row `+24` slot is reserved for a future in-core sub-verb
  table if real demand ever appears; `CMDIT_VERB_MAX` is 128 because aliases
  consume rows.

## Alternatives considered

- **Handler fn-ptrs (`cmdit_dispatch` calls `verb->handler`).** Rejected: the
  five surveyed handler signatures are irreconcilable, and the codebase never
  uses fn-ptrs — it would be a new language-feature dependency for zero benefit.
- **An in-core recursive verb tree for nesting.** Rejected: production depth is
  ≤ 2 and re-entry on the remainder delivers unbounded nesting with no new API
  and physical (per-handle) flag-scope isolation for free.
- **Editing `cmdit_parse_argv` to also do verb matching.** Rejected outright:
  it would risk the pure-core purity test and the 0.1/0.2 byte-compat contract.
  A separate `cmdit_dispatch_argv` keeps the core frozen.
- **Returning the verb id directly from `cmdit_dispatch`.** Rejected: id 0 is a
  valid verb and would collide with `CMDIT_OK=0`; the result code and the id are
  split (`cmdit_verb_matched`).
