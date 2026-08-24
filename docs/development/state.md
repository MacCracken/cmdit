# cmdit — Current State

> Refreshed every release. CLAUDE.md is preferences/process/procedures
> (durable); this file is **state** (volatile).

## Version

**1.2.4** — built 2026-08-23. **P-1 audit of the whole library.** No API change; every fix is
a guard. One **security** finding: `cmdit_completions` interpolated the program name RAW into
all three emitters (five sites), and on the documented `cmdit_new(0)` path that name is
`argv[0]` — attacker-chosen. An `argv[0]` of `"tool\ntouch /tmp/x\n#"` produced a script that
passed `bash -n` and executed the injected command on `source`. 1.2.1 fixed this class for verb
and flag names and named the program name in its notes, but only ever sanitised it for the shell
FUNCTION name. Plus three memory-safety / contract defects: the four flag modifiers took an
unchecked index and **wrote** through it (`cmdit_required(h, 64)` corrupted a live pointer in the
positional array); `cmdit_repeat_get` dereferenced a list pointer read from out of bounds; and
`err_entry` leaked across parses, so a second parse's error named the first parse's flag. Also
alloc-failure checks, negative-`argc`/`n` validation, and one provably dead guard removed.
**267 → 345 assertions**, every fix **mutation-proven** (17/17 reverts caught; the one unproven
guard is recorded as such). `fmt`/`lint`/`doc` now clean on all seven sources.

**1.2.3** — built 2026-08-23. **Maintenance only; `src/cmdit.cyr` untouched.** Toolchain
pin `6.4.78` → `6.5.35` and a full `cyrius lib sync --full` of the vendored stdlib (108
files, byte-identical to `6.5.35/lib`; 3 new modules + the new `lib/unicode/` subtree,
nothing removed). Both halves A/B'd and both are **byte-neutral**: holding one fixed and
swapping the other, all five artifacts (smoke, verbs, completions, tcyr, bcyr) compile
**byte-identical on sha256**. No stdlib delta reaches cmdit's call surface. **267/267**,
fuzz + 7 benchmarks green. `dist/` regenerated, now with the `dist/cmdit.deps` sidecar
that 6.5.35's `cyrius distlib` emits for consumer-side `cyrius deps`.

**1.2.2** — built 2026-07-26. Verb command-line forwarding, both halves found by
adversarial review of stiva's `exec`. Fixed: `cmdit_dispatch_argv` consumed `--` without
copying it into the remainder, so the documented escape hatch did nothing. Added:
`cmdit_verb_trailing_after(h, verb_id, n)` / `cmdit_verb_trailing_at` — a verbatim
trailing list after `n` positionals, so a global flag inside a forwarded command can no
longer retarget the host tool. `CMDIT_CTX_SIZE` 160 → 168. **250 → 267 assertions.**

**1.2.1** — built 2026-07-25. Completion-generator fixes from the stiva adoption: the
verb-position guard was wrong whenever a global flag preceded the verb (both emitters now
walk preceding words, skipping value-flag arguments), and names are **whitelist-filtered**
to `[A-Za-z0-9_.-]` because `compgen -W` re-expands each word — quoting alone could not
stop a `$(…)` in a name executing at TAB time.

**1.2.0** — built 2026-07-25. Verb introspection + shell completions (append-only):
`cmdit_verb_count` / `_name_at` / `_help_at` / `_is_alias` / `_canonical_at` and
`cmdit_completions(h, shell)` for bash/zsh/fish. Surfaced by the stiva adoption (35 verbs).
Toolchain pin `6.2.44` → `6.4.78`.

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
is now tested. **230 tests** green; benchmarks ([`benchmarks.md`](../benchmarks.md)) +
security audit ([`audit/2026-06-25-audit.md`](../audit/2026-06-25-audit.md)); the
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

- **Cyrius pin**: `6.5.35` (in `cyrius.cyml [package].cyrius`) — CI derives the
  toolchain from it, so there is no version to sync in the workflow YAML.
- **Vendored `lib/`**: 108 files, exactly `6.5.35/lib` (`cyrius lib sync --full`).
  A drift in either direction warns on every build; both warnings are currently silent.

## Source

- `src/cmdit.cyr` — the library (the `[lib]` module; forked from cyrius stdlib
  `lib/flags.cyr`, byte-compatible types/error-codes, renamed to `cmdit_*`, +
  materialize bridge / auto help+version / exit constants / raw-argv escape; 0.2.0
  flag modifiers; 0.3.0 verb dispatch; 1.0.0 `help_short`/`version_short` + `metavar`
  + `require_positionals_max`/`_exact`; 1.1.0 `help_flags`; 1.2.0 verb introspection +
  `cmdit_completions`; 1.2.2 `verb_trailing_after`). Entry struct 112 B, ctx **168 B**
  (append-only; 160 → 168 at 1.2.2 for the trailing-list slot);
  verbs are a separate lazy-allocated table. The pure `cmdit_parse_argv` core is
  `/proc`- and env-free; `cmdit_dispatch_argv` is its pure verb-dispatch sibling;
  env + required run in `cmdit_finalize` (chained by `cmdit_parse`/`cmdit_dispatch`).
- `programs/smoke.cyr` — single-command demo (enum/range/repeat/env + flag-named errors).
- `programs/verbs.cyr` — verb-dispatch demo (verb/alias/global-before-after/
  per-verb scope/require_positionals/unknown-verb list).
- `programs/completions.cyr` — shell-completion demo (1.2.0).
- `dist/cmdit.cyr` — generated bundle (`cyrius distlib`); consumers import this.
- `dist/cmdit.deps` — the stdlib-leaf sidecar `cyrius distlib` emits alongside it
  (new at 1.2.3, from toolchain 6.5.35); `cyrius deps` consumes it consumer-side.

## Tests

- `tests/cmdit.tcyr` — **345/345**. The 0.1 pure-core + 0.2 modifier + 0.3 verb
  groups, plus the 1.0.0 hardening: the full output/renderer surface
  (`cmdit_print_error` over every `CmditErr` branch incl. the short-flag sbuf path,
  `cmdit_help`, `cmdit_verbs_help`, `cmdit_version`), int-overflow rejection +
  i64-max boundary, getter idx guards, negative-int parse, unknown-short /
  short-missing-value, enum prefix rejection, dispatch pre-verb-unknown + dispatch
  missing-value, `CMDIT_FLAGS_MAX`/`CMDIT_VERB_MAX` cap returns, `cmdit_new(0)`
  argv[0] adoption, and the C5/C3/C1 groups (`help_short`/`version_short` remap+disable,
  metavar, `require_positionals_max`/`_exact`). Env positives guard on `getenv("HOME")`.
- `tests/cmdit.bcyr` — **7 real benchmarks** (`cmdit_new` floor, register/parse
  subtraction, enum re-walk, dispatch before/after-verb parity, repeat accumulate);
  see [`benchmarks.md`](../benchmarks.md). `tests/cmdit.fcyr` — fuzz stub.

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

Four repos consume `dist/cmdit.cyr` via `[deps.cmdit]`. Tags are as pinned in each
consumer's `cyrius.cyml` on 2026-08-23 — **none has moved to 1.2.3 yet**, and nothing
forces them to: 1.2.3 is byte-neutral, so a re-pin buys only the fresher vendored stdlib.

- **stiva 3.0.19** — pinned `1.2.2`, the most demanding consumer (35 verbs) and the
  source of the 1.2.0/1.2.1/1.2.2 work: verb introspection + completions, the two
  completion-generator fixes, and the `--` forwarding + `verb_trailing_after` pair.
- **kii 1.4.1** — pinned `1.1.0`. The first consumer + the faithful-extraction proof:
  dropped its hand-rolled flag-set on stdlib `flags` + `build_argv_array`;
  `cmdit_new`/`cmdit_parse`/`cmdit_get_*`/`cmdit_positional` + auto `--help`/`--version`.
- **anuenue 1.2.0** — pinned `1.1.0`. The second worked migration; the rich-help consumer
  that surfaced `cmdit_help_flags`.
- **ifran 2.2.0** — pinned `1.1.0`.

## Next

v1.0.0 shipped and the API is frozen; 1.1–1.2 are all append-only on it. 1.2.3 carries no
source change — it exists to clear the toolchain/lib drift that warned on every build.

Open, in rough order:

- **Consumer re-pins are optional.** stiva sits on 1.2.2, kii/anuenue/ifran on 1.1.0.
  Since 1.2.3 is byte-neutral there is no correctness reason to move; kii/anuenue/ifran
  would gain the 1.2.0 introspection + completions surface if they want it.
- **`docs/benchmarks.md` is stale and not from this release.** A fresh capture reads
  `cmdit_new_floor` 11.4 µs vs the documented 13.8 µs and `dispatch_before_verb` 153 ns
  vs 203 ns, but the 1.2.3 A/B proves the toolchain contributed none of it — the table is
  a v1.0 capture under pin 6.2.44 against 1.0.0 source. Re-capturing honestly needs a
  1.0.0 → 1.2.3 bisect. See [`../benchmarks.md`](../benchmarks.md).
- **Consumers should move to 1.2.4** (it carries the security fix), but **none of the four is
  currently exposed** — checked, not assumed, on 2026-08-23. Exposure needs BOTH a
  `cmdit_completions` call and a program name taken from `argv[0]`. Only stiva calls
  `cmdit_completions` (its `completions` verb), and its top-level handle is
  `cmdit_new("stiva")` — a literal. kii, anuenue and ifran likewise pass literals and never
  generate completions. The re-pin closes the path rather than fixing a live break.
- **`cyrius fmt` / `lint` / `doc --check` are clean on all seven sources** as of 1.2.4 (they
  reported 9 deferrals, 16 warnings and 8 undocumented functions at 1.2.3). Worth keeping at
  zero — the noise is what hid the four real deferral entries.
- **Deferred v2 backlog**: mutex/required-if sugar, optional-value flags, bundled shorts,
  `--no-foo`, named-positional help, config cascades — see
  [`../adr/0003-v1-freeze.md`](../adr/0003-v1-freeze.md). Plus the known micro-opt:
  `cmdit_new`'s byte-at-a-time entry memzero could be word-at-a-time.

See [`roadmap.md`](roadmap.md) + `agnosticos/docs/development/planning/cmdit.md`.
