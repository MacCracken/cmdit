# cmdit — Current State

> Refreshed every release. CLAUDE.md is preferences/process/procedures
> (durable); this file is **state** (volatile).

## Version

**1.2.6** — built 2026-10-14. **Maintenance only.** Toolchain pin `6.6.2` → `6.6.20` and a full
`cyrius lib sync --full --relock` (113 files, byte-identical to `6.6.20/lib`). `cyrius.lock` is
now tracked. `dist/` was regenerated: `distlib` now emits a compile-verified `# Requires` include
block, and the `dist/cmdit.deps` sidecar dropped from 10 entries to 4 leaves (alloc, string,
args, io). The only source edit is a doc-comment reflow for 6.6.20's stricter deferral lint
(byte-neutral). The one stdlib change that reaches cmdit is `getenv`'s OOM guard. **345/345**.

**1.2.5** — built 2026-09-11. Toolchain `6.5.35` → `6.6.2`, no source change. cmdit builds no
`Result`/`Option`/`Either`, so the 6.6.0 value form has nothing to bite on. All three `callptr`
sites were traced to their full target sets, and none returns a pair.

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

- **Cyrius pin**: `6.6.20` (in `cyrius.cyml [package].cyrius`) — CI derives the
  toolchain from it, so there is no version to sync in the workflow YAML.
- **Vendored `lib/`**: 113 files, exactly `6.6.20/lib` (`cyrius lib sync --full --relock`),
  hash-pinned in `cyrius.lock`. A drift in either direction warns on every build, and both
  warnings are currently silent.
- **On a pin move**, run `cyrius lib sync --full --relock` *before* the first build. A build
  under the new pin writes the lock from the old files, and a plain `--full` then refuses.

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
  (new at 1.2.3; since 1.2.6 it lists 4 compile-verified leaves, not the declared
  set); `cyrius deps` consumes it consumer-side.

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
  `cyrius build` prepends this whole declared set even though cmdit has no external
  `[deps.X]`. A `--no-deps` build of smoke is ~30 KB smaller, and a normal build carries
  `assert_*` functions that smoke never includes. `programs/` and `tests/` also include
  their stdlib explicitly; the duplicates dedupe.
- **What the library itself needs** is the 4 compile-verified leaves in `dist/cmdit.deps`
  and in the `# Requires` block at the top of `dist/cmdit.cyr`: alloc, string, args, io.
  `"args"` is for the `cmdit_argv`/`cmdit_parse` /proc read. `"io"` is for `getenv`, which
  `cmdit_parse` references unconditionally via `cmdit_finalize`; pure
  `cmdit_parse_argv`-only callers link it but never invoke it. The other 6 declared modules
  serve cmdit's own programs, tests, and benchmarks.

## Consumers

Six repos consume `dist/cmdit.cyr` via `[deps.cmdit]`. Pins are as read from each
consumer's `cyrius.cyml` on 2026-10-14, and every one is on **≥ 1.2.4**, so all carry the
1.2.4 completion-injection fix.

- **stiva 3.0.22** — pinned `1.2.4`. The most demanding consumer (35 verbs) and the source
  of the 1.2.0/1.2.1/1.2.2 work: verb introspection + completions, the two
  completion-generator fixes, and the `--` forwarding + `verb_trailing_after` pair.
- **kii 1.6.0** — pinned `1.2.5`. The first consumer and the faithful-extraction proof: it
  dropped its hand-rolled flag-set on stdlib `flags` + `build_argv_array`.
- **anuenue 1.4.0** — pinned `1.2.4`. The second worked migration, and the rich-help
  consumer that surfaced `cmdit_help_flags`.
- **ifran 2.2.1** — pinned `1.2.5`.
- **dig 0.4.1** — pinned `1.2.4`. Tier-3 from the adoption review.
- **commandress 1.1.8** — pinned `1.2.4`.

Only stiva calls `cmdit_completions`, and its handle is `cmdit_new("stiva")` (a literal).
None of the six was ever exposed to the 1.2.4 `argv[0]` injection.

## Next

v1.0.0 shipped and the API is frozen; 1.1–1.2 are all append-only on it. 1.2.6 is a
toolchain/lib refresh with no behaviour change.

Open, in rough order:

- **Consumer re-pins to 1.2.6 are optional.** The library code is unchanged. A consumer's
  `getenv` comes from its *own* vendored stdlib, not cmdit's `lib/`. The visible difference is
  the bundle's new `# Requires` block and the 4-leaf sidecar.
- **`docs/benchmarks.md` is stale.** The 1.2.6 capture reads `cmdit_new_floor` 11.3 µs vs the
  documented 13.8 µs, and `dispatch_before_verb` 164 ns vs 203 ns. The table is a v1.0
  capture (pin 6.2.44, 1.0.0 source). Since 6.6.x, `bench` also subtracts a measured
  clock-read floor, so old and new numbers are not directly comparable. Re-capturing honestly
  needs a bisect; tracked in [`roadmap.md`](roadmap.md#open-maintenance).
- **Tooling noise is at zero; keep it there.** `cyrius fmt --check` and `lint` are clean on
  all seven sources under 6.6.20. `doc --check` is clean on `src/cmdit.cyr` (54/54 public
  fns). It flags only `main` and local print helpers in `programs/`/`tests/`, which 6.6.2
  flagged too. 6.6.20's `lint` is stricter about deferral cross-references, and 1.2.6 fixed
  the one site it caught.
- **v2 backlog and adoption** — see [`roadmap.md`](roadmap.md).

See [`roadmap.md`](roadmap.md) + `agnosticos/docs/development/planning/cmdit.md`.
