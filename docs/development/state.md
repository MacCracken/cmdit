# cmdit — Current State

> Refreshed every release. CLAUDE.md is preferences/process/procedures
> (durable); this file is **state** (volatile).

## Version

**1.4.1** — built 2026-10-28. **Maintenance only; `src/cmdit.cyr` untouched.** Toolchain pin
`6.7.9` → `6.7.18` and a full `cyrius lib sync --full --relock` (113 files, byte-identical to
`6.7.18/lib`). Output is byte-identical (51 demo invocations, stdout, stderr and exit codes),
and all 13 benchmarks are within ±4% (layout noise). Binaries are +4,120 B, all but 24 B of it
unreachable stdlib. The one stdlib change on cmdit's path is `args_init` reading to EOF
(6.7.12), which resolves cmdit's SEC-7 issue; cmdit's own 2 MiB check stays for consumers on
older pins. **887/887**.

**1.4.0** — built 2026-10-14. **Refactor and optimization sweep**; no API, error-code or
output change (a 20,724-byte fixture of every renderer diffed against 1.3.7 after each step).

- Refactor: named field offsets (proven by byte-identical binaries), one token-to-flag path
  for both parse cores, shared verb guards, and the test suite split into 113 group functions.
- Speed (interleaved medians against 1.3.7):
  - `cmdit_new` 2.03 → 0.76 µs, a parse 2.44 → 1.16 µs, and a verb tool's setup 3.29 → 0.81
    µs: the flag and verb tables now grow on demand from 16;
  - dispatch after the verb 242 → 177 ns: one lookup per flag, not two;
  - `--help` 33.8 → 2.9 µs, an error line 4.75 → 0.77 µs, and three completion scripts
    161 → 14.9 µs: one write(2) per call;
  - a 36-flag lookup 396 → 152 ns.

  `get_enum_rewalk` is +6 ns from code layout alone.

**786 → 887 assertions**; 27/31 seeded reverts caught, and the
4 survivors are unobservable. The roadmap is now an open intake with no scheduled items.

**1.3.7** — built 2026-10-14. **Flag-table introspection, and per-verb flags in completions**,
the last pinned 1.3.x item.

- `cmdit_flag_*_at` accessors, plus `cmdit_flag_find_short` / `_find_long` (the parser's own
  lookup, span-based), for kriya's help renderer, `--help=json` and pre-parse expander.
- `cmdit_verb_flags(h, id, vh)` / `cmdit_verb_flags_at`: attach a verb's own handle. All three
  completion scripts then offer that verb's flags after it (ark). Without an attachment the
  scripts are byte-identical to 1.3.6.
- Fixed: fish quoting of a help text containing a backslash.

Verb row 40 → 48 B (`verb_new_floor` +0.2 µs, once per process). **694 → 786 assertions**,
38/39 seeded reverts caught (the survivor is unobservable), fuzz invariant I9.

**1.3.6** — built 2026-10-14. **Four small modifiers**, one per surveyed tool:

- `cmdit_once` (cyim): a second occurrence is the new `CMDIT_ERR_REPEATED` (14);
- `cmdit_enum_nocase` (phylax);
- `cmdit_long_alias` (ark): a second long name;
- `cmdit_reject_flag_value` (hapi F-019): `--root --dry-run` is a missing value.

The flag entry grows 112 → 120 B. **629 → 694 assertions**, 16/16 mutation-proven.

**1.3.5** — built 2026-10-14. **Help on a chosen stream, and hidden verbs.**
`cmdit_help_fd` / `cmdit_help_flags_fd` / `cmdit_verbs_help_fd(h, fd)`; the stderr forms are
byte-identical wrappers. `cmdit_verb_hidden` / `cmdit_verb_is_hidden`: a verb that dispatches
but stays out of help, the unknown-command list and completions. Verb row 32 → 40 B. The verb
list's content is now pinned; its tests checked only the return code. Dispatch is +6–10 ns
(code layout, stride ruled out), and a verb tool's construction +0.2 µs. **596 → 629
assertions**, 12/12 mutation-proven.

**1.3.4** — built 2026-10-14. **POSIX stop-at-first-positional**, `cmdit_stop_at_positional(h,
on)`. Option recognition stops at the first positional (a lone `-` included), and a `--`
after it stays data. It is not the per-verb trailing mark, which consumes `--` at its
threshold, the bug that cost kriya a directory under `xargs rm --`. On a verb handle, globals
apply only before the verb. Shakti's last blocker. ctx 184 → 192 B. **562 → 596 assertions**,
7/7 mutation-proven.

**1.3.3** — built 2026-10-14. **The offending token.** `cmdit_err_token` (the argv token
behind the last error, as typed, matching stdlib `flags_error_arg`) and `cmdit_err_value`
(the value a BAD_INT / BAD_ENUM / OUT_OF_RANGE / UNEXPECTED_VALUE rejected) are append-only
additions; ctx grows 176 → 184 B. `cmdit_print_error` now prints them, with every argv-derived
byte escaped (control bytes → `?`). It also fixes 1.3.2's five untracked lint deferrals,
which shipped despite its notes. **503 → 562 assertions**, 18/18 mutation-proven.

**1.3.2** — built 2026-10-14. **Audit sweep + one security fix**
([`audit/2026-10-14-audit.md`](../audit/2026-10-14-audit.md)).

- **Security:** a program name starting with `-` (attacker-chosen `argv[0]`) became options
  to `complete` in generated completion scripts.
- **Attacker-reachable:** a command line over 2 MiB, or unreadable, parsed as a silent
  prefix; it is now `CMDIT_ERR_ARGV` (13).
- **Memory, program misuse:** slot-count size wraps; public layout vars read live (now a
  private enum); out-of-range aliases; 0-handle crashes on ~25 entry points.
- **Contract:** five verb-dispatch fixes, and three in completions.
- **Speed:** verb-tool construction 8.4 → 2.8 µs.

**400 → 503 assertions**, 33/33 seeded reverts caught.

**1.3.1** — built 2026-10-14. **Stdlib parity.** Positionals and repeat lists double on demand
instead of silently dropping past 128 / 64, which reverses audit SEC-3. A bool given `=value`
is `CMDIT_ERR_UNEXPECTED_VALUE` (11) instead of turning the flag on. A refused growth is
`CMDIT_ERR_NOMEM` (12). Both defects were fixed in the stdlib in 6.6.5, and the 1.3.0
adoption survey found cmdit still had them. **357 → 400 assertions**, 10/10 mutants caught.

**1.3.0** — built 2026-10-14. Toolchain pin `6.6.20` → `6.7.9` (`lib/` byte-identical to
`6.7.9/lib`). **`cmdit_new` ~6× faster**: 1.97 µs, was ~11.5, from zeroing the entry table
word-at-a-time. **`cmdit_argv` linear on Linux**: it was quadratic, re-walking
`/proc/self/cmdline` per index, so 20,000 arguments cost 2.6 s; now 11 ms. **The fuzz harness is
real now**: it had been a stub that called nothing since 0.1.0, and it now runs 6,400 seeded
argvs per run through both pure cores, mutation-proven 8/9. Benchmarks were re-captured with a
1.0.0 → 1.3.0 bisect. `cyrius audit` is clean on every stage. No API change. **357/357**.

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
`cmdit_completions(h, shell)` for bash/zsh/fish. Surfaced by the stiva adoption (36 verbs).
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

- **Cyrius pin**: `6.7.18` (in `cyrius.cyml [package].cyrius`) — CI derives the
  toolchain from it, so there is no version to sync in the workflow YAML.
- **Vendored `lib/`**: 113 files, exactly `6.7.18/lib` (`cyrius lib sync --full --relock`),
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
  `cmdit_completions`; 1.2.2 `verb_trailing_after`; the 1.3.x survey items through 1.3.7's
  flag introspection and per-verb completion flags). Entry **120 B**, ctx **192 B**, verb
  row **48 B**, all append-only. Every field is addressed through the private
  `CmditOffset` names (1.4.0); the layout comment at the top of the source has every
  offset. The flag table starts at 16 entries and the verb table at 16 rows, each doubling
  to its cap of 64 / 128 (1.4.0; see
  [`../architecture/001-tables-move-when-they-grow.md`](../architecture/001-tables-move-when-they-grow.md)).
  Every renderer writes through an output session, one write(2) per call
  ([`../architecture/002-every-write-goes-through-an-output-session.md`](../architecture/002-every-write-goes-through-an-output-session.md)).
  Both parse cores apply flags through one `_cmdit_take_flag`. The pure `cmdit_parse_argv` core is
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

- `tests/cmdit.tcyr` — **887/887**, in 113 group functions (1.4.0; it was one ~2,300-line
  `main` whose fixture names all had to be unique). The 0.1 pure-core + 0.2 modifier + 0.3 verb
  groups, plus the 1.0.0 hardening: the full output/renderer surface
  (`cmdit_print_error` over every `CmditErr` branch incl. the short-flag sbuf path,
  `cmdit_help`, `cmdit_verbs_help`, `cmdit_version`), int-overflow rejection +
  i64-max boundary, getter idx guards, negative-int parse, unknown-short /
  short-missing-value, enum prefix rejection, dispatch pre-verb-unknown + dispatch
  missing-value, `CMDIT_FLAGS_MAX`/`CMDIT_VERB_MAX` cap returns, `cmdit_new(0)`
  argv[0] adoption, and the C5/C3/C1 groups (`help_short`/`version_short` remap+disable,
  metavar, `require_positionals_max`/`_exact`). Env positives guard on `getenv("HOME")`.
  1.4.0 adds the output-session, table-growth and lookup-equivalence groups.
- `tests/cmdit.bcyr` — **13 real benchmarks** (`cmdit_new` floor, register/parse
  subtraction, enum re-walk, dispatch before/after-verb parity, repeat accumulate, the
  1.3.2 `verb_new_floor`, the 1.3.7 `flag_find_long`, and 1.4.0's `flag_find_long_36` over
  attn11's real flag names plus the three output paths to /dev/null);
  see [`benchmarks.md`](../benchmarks.md).
- `tests/cmdit.fcyr` — **real fuzz harness** (1.3.0; it was a stub before). 16 seeds × 400 argvs
  through `cmdit_parse_argv` and `cmdit_dispatch_argv`, mixing a 54-token pool with random bytes
  and checking nine contract invariants (I1–I9 in its header; I7 since 1.3.3, I8 since 1.3.4,
  I9, the flag lookups against a brute-force walk of the accessors, since 1.3.7). Since 1.3.1,
  every 50th argv is long (up to 400 tokens), so the growth paths past 128 / 64 are fuzzed
  too. Mutation-proven at 1.3.0: 8 of 9 seeded source bugs caught. The survivor (the remainder
  slice's NULL terminator) is unobservable because fresh `alloc` memory is already zero. I9
  alone catches each of three seeded lookup bugs (1.3.7). Since 1.4.0 both fuzz handles are
  padded past 16 flags / 16 verb rows, so the tables grow inside every iteration. ~0.6 s.
- **Poison gates:** `cyrius fuzz --poison=ab` and `CYRIUS_POISON=1 cyrius test` both pass, so
  there is no redzone overwrite in cmdit. Neither can observe `cmdit_new`'s zeroing loop:
  poison fills only the redzones.

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

- **stiva 3.0.22** — pinned `1.2.4`. The most demanding consumer (36 verbs) and the source
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

v1.0.0 froze the API; 1.1–1.4 are all append-only on it. Every gap the 2026-10-14 survey
pinned shipped in 1.3.x, and 1.4.0 was the refactor and optimization sweep. **Nothing is
scheduled.** The [roadmap](roadmap.md) is an open intake for consumer requests and issues.

Open, in rough order:

- **Consumer re-pins to 1.4.1 are recommended**; its bundle is 1.4.0's code. They bring the
  misparse fixes, the audit repairs, the token-naming error lines and 1.4.0's speed (a parse
  about 2× faster, `--help` over 10×). None of the six breaks. 1.4.0's output is
  byte-identical to 1.3.7's; it now arrives in one write per call.
  - stiva, ifran, commandress and dig call `cmdit_print_error`, so their stderr lines now
    name the token; no test of theirs pins the old text.
  - kii and anuenue's `cmdit_error` switches print their generic line for
    `UNEXPECTED_VALUE` and `ARGV`; `cmdit_err_token` lets them name the token.
  - stiva should re-pin first, for its marked `exec`. It is the one consumer that generates
    completions; its scripts are unchanged until it attaches verb handles, apart from 1.3.7's
    fish fix for a verb help ending in a backslash.

  The bundle compiles on pins as old as 6.6.2.
- **Resolved in cyrius 6.7.12:** stdlib `args_init` returned a prefix of a command line over
  2 MiB with no error (cyrius
  `docs/development/issues/archived/2026-10-14-args-init-truncates-argv-past-2-mib.md`). It now
  reads to EOF. cmdit's fail-closed 2 MiB check (audit SEC-7) is still in place and is due a
  revisit; see the [roadmap](roadmap.md#watching).
- **Adoption notes are filed in the 18 hand-rolled candidates' roadmaps** (2026-10-14). Each
  is a site-by-site map from that repo's CLI code to cmdit calls. klug is not worth migrating.
  - **kriya and ark** carry an appended "Update — cmdit 1.4.0" (2026-10-14): kriya's
    expander and help renderer can use `cmdit_flag_*` instead of the stdlib flags layout,
    and ark's per-verb completion lists can come from `cmdit_verb_flags`.
  - **The other sixteen predate 1.3.3 and 1.3.4**, and are left unedited at the user's
    request. They still list the token gap as open (shakti, attn11, owl, chakshu, hapi,
    phylax, agnova) and stop-at-positional as a blocker (shakti). shakti's note calls it
    blocked, but neither of its two cmdit blockers remains.
  - bannermanor, klug, shakti, yo, kriya and cyim also carry an "Update — cmdit 1.3.1".
- **Tooling noise is at zero; keep it there.** Under 6.7.18, `cyrius audit` is clean on every
  stage, and `fmt --check` / `lint` / `doc --check` are clean on all seven sources.
- **v2 backlog** — see [`roadmap.md`](roadmap.md).

See [`roadmap.md`](roadmap.md) + `agnosticos/docs/development/planning/cmdit.md`.
