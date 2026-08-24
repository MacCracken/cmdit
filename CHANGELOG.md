# Changelog

Format: [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased]

## [1.2.4] - 2026-08-23

**A P-1 audit of the whole library. One finding is a remote-ish code execution in the completion
generator; three more are memory-safety or contract defects.** No API change — every fix is a
guard, and the frozen 1.0.0 surface plus the 1.1–1.2 additions behave identically on every input
that was already valid. **267 → 345 assertions**, and each fix below is **mutation-proven**: the
fix was reverted, and a test failed.

### Fixed (SECURITY) — the program name was injected verbatim into generated completion scripts

`cmdit_completions` interpolated `prog` **raw** into all three emitters, in five places: the bash
`# bash completion for <prog>` comment and its `complete -F _<fn>_complete <prog>` trailer, zsh's
`#compdef <prog>`, and fish's `complete -c <prog> -f` plus one `complete -c <prog>` per verb. A
completion script is *sourced by the user's shell*, so a byte reaching it verbatim is code.

`prog` is `prog_name` from `cmdit_new` — and on the documented, tested `cmdit_new(0)` path it is
adopted from **`argv[0]`**, which is chosen by whoever execs the tool. With
`argv[0] = "tool\ntouch /tmp/x\n#"` the generated file was:

```
# bash completion for tool
touch /tmp/x
#
_tool_touch__tmp_x___complete() {
```

Verified end to end: the result passes `bash -n` — it is a *valid* script — and running
`source` on it executed the injected command.

⛔ **1.2.1 fixed this class for verb and long-flag names and explicitly named the program name in
its own notes — but only ever routed it through `_cmdit_oprint_ident` for the shell FUNCTION
name.** The command-word occurrences were never touched. Anyone reading that changelog would
reasonably conclude the program name was handled; it was not.

**No shipped consumer is currently exposed** — verified rather than assumed. Exposure needs both
a `cmdit_completions` call and a program name taken from `argv[0]`. Of the four consumers, only
stiva calls it, and its top-level handle is `cmdit_new("stiva")`; kii, anuenue and ifran pass
literals and generate no completions. The defect is real and the fix is worth taking, but nothing
downstream is broken today.

New `_cmdit_oprint_cmdname` maps every byte outside `[A-Za-z0-9_.-]` to `_`, so the emitted token
is always exactly one inert word. It **maps rather than drops** (the 1.2.1 remedy for verb names)
because a completion with no command to attach to is useless. `.` and `-` are kept, so ordinary
names survive byte-for-byte — `complete -c cmdit-smoke` still reads `cmdit-smoke`, and the
generated scripts still pass `bash -n` and `zsh -n`.

The alphabet now lives in one predicate, `_cmdit_is_cmdname_byte`, shared with
`_cmdit_name_is_safe`, and the suite asserts it over **all 256 byte values** rather than over the
metacharacters someone happened to think of. A second test captures the emitters' real stdout
through `pipe(2)`/`dup2(2)` and asserts the injected command is absent — testing the predicate
alone would still pass if an emitter simply stopped calling it, and the mutation run confirms all
five call sites are individually covered.

### Fixed (memory safety) — the flag modifiers took an unchecked index and WROTE through it

The 1.0.0 audit added `_cmdit_idx_ok` to the typed getters. The modifiers never got it: only
`idx < 0` was tested, so `cmdit_required` / `cmdit_range` / `cmdit_env` / `cmdit_metavar` turned
an out-of-range index into an out-of-bounds **write**.

`cmdit_required(h, 64)` — one past `CMDIT_FLAGS_MAX` — read-modify-wrote `entries + 7216`, which
the bump allocator places **48 bytes into the positional array**. Demonstrated: a live
`positional[6]` pointer to `"p6"` came back incremented by one, i.e. a valid cstr pointer turned
into one aimed at the middle of a string. All four now share the getters' guard and return -1.

### Fixed (memory safety) — `cmdit_repeat_get` dereferenced a pointer read from out of bounds

It guarded `i` but not `idx`, and computed `_cmdit_entry(h, idx)` before any check — so a bad
`idx` did not merely read out of bounds, it **followed the list pointer it found there**.
`cmdit_repeat_count` was unguarded outright. Both take `_cmdit_idx_ok` now, which also gained a
null-handle test, hardening every getter that routes through it.

### Fixed (contract) — `err_entry` leaked across parses, so errors named the wrong flag

`cmdit_err_flag` documents -1 for errors that name no registered flag (UNKNOWN / BUNDLED), and
`cmdit_print_error` prefixes the flag name for every error except UNKNOWN. Neither parse loop
ever cleared `err_entry`, so that held **only on a virgin handle**. Parse a handle twice — first
hitting `--name` with no value, then a bundled `-xy` — and 1.2.3 reported:

```
prog: --name: bundled/attached short flags not supported (use -x -y)
```

naming a flag from the *previous* parse. `cmdit_parse_argv` and `cmdit_dispatch_argv` now reset
the error trio (`last_error`, `err_entry`, `err_verb`) on entry. ⭐ Per-site clears were tried
first and are **not** in the final fix: mutation testing showed them redundant against the reset —
every such site is downstream of it — so the fix is one mechanism, not two.

### Fixed (input validation)

- `cmdit_verb_trailing_after` accepted a negative `n`. It stores `n + 1`, so `n == -1` silently
  meant "not marked", and `n < -1` stored a negative that `cmdit_verb_trailing_at`'s `v == 0`
  test did not catch — handing the dispatcher a nonsense threshold. Now -1, plus the missing
  null-handle and empty-verb-table guards.
- `cmdit_dispatch_argv` with a negative `argc` called `alloc((argc+1)*8)`, which returns 0 for a
  non-positive size, and then wrote the slice's NUL terminator **to address 0**. `argc` is clamped.

### Hardened — stdlib `alloc` returns 0 on failure and every call site ignored it

`lib/alloc.cyr` returns 0 for OOM or an out-of-range size; it does not abort. `cmdit_new` then
memzeroed 7168 bytes from address 0. Checks added in `cmdit_new` (returns 0), `cmdit_argv`,
`_cmdit_repeat_push`, `_cmdit_verb_add` and `cmdit_dispatch_argv`.

⚠ **Recorded as unproven:** the `cmdit_new` check is the one change here with no failing test.
Its three sizes are compile-time constants well inside `ALLOC_MAX`, so no public API call can
drive `alloc` to return 0 — deleting the line keeps the suite green. It is defence-in-depth
against an exhausted heap, not behaviour any fixture reaches.

### Removed — one guard that was provably dead

`cmdit_repeat_get`'s null-list check is unreachable: the list is allocated by the same push that
makes the count non-zero, so `count == 0` whenever the pointer is 0 and the count test above
already returns. Mutation-verified, not assumed, and replaced by a comment stating the invariant.

### Quality gates — all clean, across every file

`cyrius fmt --check`, `cyrius lint` and `cyrius doc --check` now pass on all seven sources
(`src/`, three `tests/`, three `programs/`). Measured at the 1.2.3 tree, the baseline was **9
untracked deferrals and 16 warnings** across those files, plus 8 undocumented functions; it is
0/0/0 now (the new 1.2.4 tests are held to the same bar):

- **Docs**: 53 documented, **0 undocumented** (was 46/8). The gaps were accessors sharing a block
  comment with a sibling — `cmdit_get_int`, `cmdit_get_str`, `cmdit_repeat_get`,
  `cmdit_positional`, `cmdit_verb_argc`, `cmdit_verb_argv`, `cmdit_raw_argc`,
  `cmdit_version_short` — each now carries its own.
- **Lint**: 0 deferrals, 0 warnings. One deferral was real (the NOT-supported list) and now
  cross-references the roadmap and ADR 0003; the rest were the word "deferred" used in its
  domain sense — a flag deferred into a verb's scope — and carry `#skip-lint` rather than prose
  bent to satisfy a keyword scan.
- **Format**: `cyrius fmt` applied; the diff is whitespace-only (`diff -w` is empty).
- **Corrected a long-standing doc figure**: `docs/development/state.md` has claimed "8 real
  benchmarks" since 1.0.0, and the 1.2.3 entry above repeated it. `tests/cmdit.bcyr` defines
  and runs **7** — which is what `docs/benchmarks.md`'s own table has listed all along. Both
  are now 7.

### No behaviour change on anything that already worked

Every fix is a guard on an input that previously corrupted memory or emitted code, so nothing
valid should move — checked rather than asserted. The 1.2.3 and 1.2.4 builds of all three demo
programs were run over **27 invocations** — help, version, every error arm (bad int, out of
range, bad enum, unknown flag, bundled short, missing value), `-`, `--`, repeats, positionals,
global-flag-before/after-verb, unknown verb, and all four `completions` shells — and stdout,
stderr and exit code are **byte-identical** in every case. `cmdit_help`'s output in particular
is unchanged despite the renderer's `if`/`elif` chain being re-wrapped for the column limit.

## [1.2.3] - 2026-08-23

**Maintenance: toolchain pin 6.4.78 → 6.5.35 and a full vendored-stdlib refresh.** No change to
`src/cmdit.cyr` — the public surface is byte-for-byte the frozen 1.0.0 API plus the 1.1–1.2
append-only additions.

Both halves were overdue and each warned on every single build:

```
warning: cyrius.cyml pins 6.4.78 but cycc is 6.5.35 — toolchain drift
warning: ./lib/ shadows version-pinned .../6.4.78/lib — 12 bundled lib(s) differ
```

- **Pin 6.4.78 → 6.5.35.** CI derives the toolchain from `cyrius.cyml [package].cyrius`, so no
  workflow edit was needed.
- **`cyrius lib sync --full`** re-vendored `lib/` from the new pin: 108 files, now byte-identical
  to `6.5.35/lib`. 70 modified, 3 new top-level modules (`async_macos`, `async_win`,
  `thread_macos`) and the new `lib/unicode/` subtree; nothing removed. The 12 stale bundled libs
  the shadow warning named (bayan, ganita, sakshi, niyama, sigil, sandhi, yukti, patra, vani,
  mabda, sankoch, yantra) are current.

### The upgrade is codegen-neutral for cmdit, and that was measured rather than assumed

⭐ **Both halves were A/B'd separately, and both are byte-neutral.** Holding one variable fixed
and swapping the other, every artifact — `programs/smoke.cyr`, `programs/verbs.cyr`,
`programs/completions.cyr`, `tests/cmdit.tcyr`, `tests/cmdit.bcyr` — comes out **byte-identical
on sha256**:

| A/B | held fixed | swapped | result |
|---|---|---|---|
| compiler | 6.5.35 `lib/` | cycc 6.4.78 ↔ 6.5.35 | 5/5 byte-identical |
| vendored lib | cycc 6.5.35 | `lib/` 6.4.78 ↔ 6.5.35 | 5/5 byte-identical |

So the whole of 1.2.3 is a no-op at the machine-code level for cmdit. 6.5.35's headline is a
register allocator that finally time-shares registers, but its own release notes scope the win to
straight-line regions, and cmdit's hot paths do not hit it.

⚠ **`docs/benchmarks.md` is therefore NOT re-captured in this release, and its numbers are
stale for a different reason.** A fresh capture on this tree reads `cmdit_new_floor` **11.4 µs**
against the documented **13.8 µs**, and `dispatch_before_verb` **153 ns** against **203 ns** —
but the byte-identity above proves none of that delta is this toolchain bump. That table was
captured at the v1.0 cut under pin 6.2.44 against 1.0.0 source, so the gap belongs to the
1.0.0 → 1.2.3 source evolution (ctx 160 → 168 B, verb introspection, completions) and to capture
conditions. Attributing it needs a bisect, which is its own change, not this one.

### No stdlib delta lands on cmdit's call surface, and that surface is smaller than it looks

`string`, `fmt`, `alloc`, `io`, `vec`, `syscalls`, `args`, `assert` and `bench` all moved; `str`
is unchanged. Every public signature delta is **additive** (`vec_sort_by`/`vec_select_nth`,
`arena_new_growable`, the `xmkdir_p`/`xsymlink` family, `signal_default`, the bench clock-overhead
calibration). Two `alloc` internals were removed — `_arena_alloc`, `_arena_reset` — and cmdit
references neither.

The reason nothing reaches it: cmdit's **entire** external surface is `alloc`, `args_init`,
`argc`, `argv`, `getenv`, `strlen`, `load8/64`, `store8/64` and `syscall`. It renders no integer
to output at all — `_cmdit_eprint`/`_cmdit_oprint` are raw `syscall(1, …)` string writes, and an
int/range error prints the flag name and a fixed message, never the value. And every stdlib
entry point it *does* call is **byte-identical** across the two snapshots — `getenv`, `strlen`,
`alloc`, `args_init`, `argc`, `argv`, compared body-for-body; `getenv` merely moved down
`io.cyr`, and `args.cyr`'s sole change is a comment. Which is why the lib-swap A/B above comes
out byte-identical rather than merely equivalent.

⚠ Two changes in the refresh look like they should matter here and **do not** — recorded so the
next reader does not re-derive it:

- **`string.cyr:print_num` i64::MIN** (cyrius 6.5.8) — `0 - n` is a no-op at the minimum, so the
  old `n > 0` loop emitted zero digits and printed a bare `-`. cmdit never calls `print_num`
  (0 references in `src/` and `dist/`).
- **`assert.cyr:assert_eq`** moved its `got`/`expected` output from `fmt_int` (fd 1) to
  `efmt_int` (fd 2). cmdit's harness uses `assert`/`assert_summary`, not `assert_eq`.

**267/267 assertions green, unchanged from 1.2.2**, plus fuzz, all 7 benchmarks, and the three
demo programs. `dist/cmdit.cyr` regenerated at 1.2.3; `cyrius distlib` on 6.5.35 additionally
emits the **`dist/cmdit.deps`** sidecar (the 10 stdlib leaves this fold needs in scope), which
`cyrius deps` consumes on the consumer side — new file, tracked, matching stiva.

## [1.2.2] - 2026-07-26

**A verb could not forward a command line.** Both halves found by adversarial review of stiva's
`exec`, which needs exactly this and could not ship without it.

### Fixed — the `--` terminator was consumed by the dispatcher and never forwarded
`cmdit_dispatch_argv` treated `--` as "stop parsing global flags" and `continue`d **without copying
the token into the remainder slice**, so the verb's own `cmdit_parse_argv` — which handles `--`
correctly — never saw one. The documented escape hatch for a command carrying flags therefore did
nothing, and `prog verb -- cmd -x` failed **identically** to omitting it:

```
stiva exec c1 ls -l       -> "unknown flag", the command never runs
stiva exec c1 -- ls -l    -> the same error
```

A `--` appearing BEFORE the verb is still consumed, because `remainder[0]` must be the verb name
and the terminator has already done its job there.

### Added — `cmdit_verb_trailing_after(h, verb_id, n)`
Marks a verb as taking a **verbatim** trailing argument list after its first `n` positionals.
Past that point nothing is treated as a flag, at either the dispatcher or the verb level, and the
mark propagates from the top-level handle into `cmdit_parse_verb` so a consumer declares it once.

This is a correctness fix, not a convenience. Without it a **global** flag appearing inside a
forwarded command was applied to the host tool:

```
stiva exec c1 mycmd --root /x   -> silently retargeted STIVA's data root
```

No diagnostic, and the payload never saw the argument it was passed. `cmdit_verb_trailing_at`
reports the mark (−1 when unset). Aliases inherit it through the canonical row.

`CMDIT_CTX_SIZE` grows 160 → 168 for the new per-handle slot. Unmarked verbs are byte-for-byte
unchanged. **250 → 267 assertions.**

## [1.2.1] - 2026-07-25

**Completion-generator fixes, all found by adversarial review of the stiva adoption.** 1.2.0's
generated scripts were valid shell but wrong in two ways that matter.

- **The verb-position guard was wrong whenever a global flag preceded the verb.** bash used
  `[[ $COMP_CWORD -eq 1 ]]` and zsh `(( CURRENT == 2 ))`, so `tool --root /x <TAB>` — the ordinary
  invocation for any tool with a global value-flag — completed filenames instead of verbs.
  Verified live against bash 5.3 with the real generated script. Both emitters now walk the words
  before the cursor for the first non-flag token, skipping the argument of a value-taking flag,
  using a new `valueflags` list derived from the entry table's types.
- **The escaping claim in 1.2.0's changelog was false.** Verb names and long-flag names were
  interpolated into `local verbs='…'` with **no** escaping, and the program name was emitted
  outside any quoting. More importantly, escaping alone cannot fix this: `compgen -W "$verbs"`
  re-expands each word, so a `$(…)` or a backtick in a name would execute at TAB time even inside
  correct single quotes. Names are now **whitelist-filtered** to `[A-Za-z0-9_.-]` and dropped from
  the completion if they fail — a missing completion is a papercut; a command substitution running
  in the user's shell is not.

No consumer was exposed: stiva's verb and flag names are all safe literals. This is a library
contract defect, fixed before anything relied on the contract.

## [1.2.0] - 2026-07-25

**Verb introspection + shell completions.** Append-only and non-breaking on the frozen 1.0.0
surface — new functions only.

The verb table already drove `cmdit_verbs_help`, but nothing exposed it, so a consumer could not
generate shell completions from the very verbs it had just registered: it had to keep a second,
hand-maintained list, which drifts the moment a verb is added. Surfaced by the stiva adoption
(35 verbs, a `completions` subcommand among them).

- `cmdit_verb_count(h)` — total rows, aliases included.
- `cmdit_verb_name_at(h, i)` / `cmdit_verb_help_at(h, i)` — name and help. An alias row stores no
  help of its own, so `help_at` returns the **canonical** verb's, or a caller listing aliases
  would render a blank description.
- `cmdit_verb_is_alias(h, i)` / `cmdit_verb_canonical_at(h, i)` — the same
  `canonical_id == own index` test `cmdit_verbs_help` uses to skip alias rows.
  Out-of-range and null-handle inputs return 0 / -1, never a wild read.
- `cmdit_completions(h, shell)` — writes a bash, zsh, or fish completion script for `h`'s
  registered verbs and long flags. Returns -1 for an unrecognised shell **without printing
  anything**, so a caller can report the error without a half-written script on stdout.

Two details that matter for generated shell:

- Output goes to **stdout**, not stderr — the script is meant to be redirected into a completions
  file or eval'd.
- Help text is caller-supplied prose, so every value interpolated into a single-quoted shell
  string has its apostrophes escaped (`'` → `'\''`). An unescaped one would close the quote and
  inject the remainder into the script. Function names derived from the program name are
  sanitised to `[A-Za-z0-9_]` for the same class of reason.

Verified by syntax-checking the generated output with the real parsers: `bash -n` and `zsh -n`
both accept it, including a verb description containing an apostrophe. `programs/completions.cyr`
is the runnable demo.

Also: toolchain pin 6.2.44 → 6.4.78 (the suite is green on it; the old pin warned on every build).

## [1.1.0] - 2026-06-25

**Rich-help support (`cmdit_help_flags`).** Append-only and non-breaking on the
frozen 1.0.0 surface — a new function only. Surfaced by the anuenue adoption pilot:
tools that frame their own help (a brand intro + custom usage synopsis + an Examples
block) need to wrap a *generated* flag list, which the monolithic `cmdit_help`
couldn't provide. The re-survey under-counted this (deemed it beneficiary≈1); the
pilot is the second concrete consumer, so it graduates from caller-side to a tiny
library primitive.

### Added
- **`cmdit_help_flags(h)`** — the table-only flag renderer: prints just the
  `  -x, --long <type>\thelp` rows (no `Usage:`/`Options:` header, no verb list), so a
  consumer can sandwich the generated rows inside its own help prose. `cmdit_help`
  now composes it; `cmdit_help`'s output is byte-for-byte unchanged from 1.0.0.

## [1.0.0] - 2026-06-25

**The v1.0 freeze.** The public API is frozen. This cut lands the final completeness
delta a 23-repo re-survey of hand-rolled CLI code found still uncovered, fixes the two
low-severity correctness issues an adversarial security audit confirmed, and meets the
remaining v1.0 gates (benchmarks, security-audit doc, CHANGELOG, doc-drift). Append-only:
`CMDIT_BOOL/INT/STR` = {0,1,2}, every prior entry/ctx offset, and the 0..9 error codes
are untouched; the entry struct grows 104→112 B for the metavar.

### Added
- **`cmdit_help_short(h, ch)` / `cmdit_version_short(h, ch)`** — remap (any ASCII byte)
  or disable (`ch = 0`) the auto `-h` / `-V` short letters, keeping the `--help`/
  `--version` long forms + detection. Frees `-h` for a consumer's own flag (the
  `df -h`/`ls -h` human-readable convention). No struct/error growth.
- **`cmdit_metavar(h, idx, name)`** — override a value flag's generated-help placeholder
  (`--out <FILE>` vs `<str>`); cosmetic, parsing-neutral, no-op on bool flags. The
  generated help now also renders the registered default of string flags (`(default: …)`;
  cstr only — int defaults stay unrendered, preserving the no-`fmt` renderer contract).
- **`cmdit_require_positionals_max(h, max)` / `cmdit_require_positionals_exact(h, n)`** —
  the upper-bound and both-bounds mirrors of `cmdit_require_positionals`. New error
  **`CMDIT_ERR_TOO_MANY_POSITIONAL = 10`** (`too many arguments`, exit 2); reuses the
  positional count (no struct growth).

### Fixed
- **`_cmdit_parse_int` integer overflow** (audit SEC-1) — a long all-digit argv/env token
  silently wrapped mod 2⁶⁴ and could **bypass a `cmdit_range` bound** (`18446744073709551617`
  → wraps to 1 → passed 1..4096). Now rejected before the multiply (`> (INT64_MAX − d)/10`)
  → `CMDIT_ERR_BAD_INT`; the largest valid i64 still parses. The shared env-int path
  inherits the rejection.
- **Unguarded getter index** (audit SEC-2) — `cmdit_get_bool/int/str/enum` and `cmdit_seen`
  did not bounds-check `idx`, so an ignored `-1` registration return + `cmdit_get_int(h, -1)`
  was a 112-byte OOB read. A shared `_cmdit_idx_ok` guard now returns the type's zero
  sentinel (mirroring `cmdit_positional` / `cmdit_repeat_get`).

### Changed
- Entry struct 104 → 112 B (metavar at +104; append-only, old offsets untouched).

### Hardening (the freeze gates)
- **Tests 157 → 230.** New coverage for the entire output/renderer surface
  (`cmdit_print_error` every `CmditErr` branch, `cmdit_help`, `cmdit_verbs_help`,
  `cmdit_version`), negative-int parse, dispatch pre-verb-unknown + dispatch
  missing-value, enum prefix rejection, `CMDIT_FLAGS_MAX`/`CMDIT_VERB_MAX` cap returns,
  getter idx guards, the overflow fix, and the C5/C3/C1 additions.
- **Benchmarks** captured in [`docs/benchmarks.md`](docs/benchmarks.md) (harness
  `tests/cmdit.bcyr`): handle construction ~13.8 µs (dominated by the entry memzero),
  parse loop sub-µs, single-pass dispatch verified (after-verb ≈ before-verb, not 2×).
- **Security audit** pass in [`docs/audit/2026-06-25-audit.md`](docs/audit/2026-06-25-audit.md):
  no attacker-reachable memory-safety defect on any argv/env path; SEC-1/SEC-2 fixed.
- **Doc drift fixed** — README test count (26→230), the `src/cmdit.cyr` ctx-size header
  comment (104→160 B), `getting-started.md` (`src/main.cyr`→`src/cmdit.cyr`), demo version
  strings (smoke 0.2.0→1.0.0), and this CHANGELOG's dates + the 0.3.0 `88→157` baseline.

### Scope (deferred to a future release — confirmed by the re-survey, not dropped)
- Cross-flag mutex / required-if sugar, duplicate-flag rejection, optional-value
  (getopt `optional_argument`) tri-state, stop-at-first-positional, `--no-foo` negation,
  bundled shorts `-abc`, attached short value `-xvalue`, named-positional help synopsis,
  multi-source env/config cascades. Each resolved to caller-side / already-expressible /
  low-prevalence; none threatens correctness or the frozen surface.

## [0.3.0] - 2026-06-25

**Verb / subcommand dispatch (M3).** Identify-and-return dispatch (mirroring
`cmdit_get_enum`, not handler fn-ptrs — consumer handler signatures are
irreconcilable and the codebase never uses fn-ptrs), grounded in a site-by-site
re-survey of the ~9 subcommand tools (ark/sit/takumi/hapi/phylax/agnova/hadara/
hoosh/cyim) + the kriya multiplexer. Append-only: the entry struct is unchanged
(verbs are a separate lazy-allocated table), ctx grows 104→160 B, error taxonomy
grows. The pure `cmdit_parse_argv` core is untouched.

### Added
- **`cmdit_verb`** / **`cmdit_verb_alias`** — register subcommands (and aliases)
  in a lazily-allocated verb table; `cmdit_verb` returns the verb id, the
  identify-and-return key.
- **`cmdit_dispatch_argv`** — a NEW pure sibling of `cmdit_parse_argv`: one pass
  that applies registered global flags **before OR after** the verb (the fix for
  ark's double-scan), matches the first non-flag token against the verb table, and
  compacts a globals-removed **remainder slice** (`argv[0]` = verb name) for the
  caller to re-parse. **`cmdit_dispatch`** = materialize + dispatch + finalize the
  global scope. **`cmdit_verb_matched`** returns the matched canonical id (-1 = no
  verb token, distinct from an UNKNOWN_VERB error). **`cmdit_verb_argc`** /
  **`cmdit_verb_argv`** expose the remainder.
- **`cmdit_parse_verb`** — sugar: re-parse the remainder in a per-verb handle +
  finalize. **Nested sub-verbs** are just re-entry on the remainder (no in-core
  tree — the roadmap bent here; depth-2 max in real code).
- **`cmdit_basename`** — interior-pointer basename of `argv[0]` for the busybox
  multiplexer (with the existing `cmdit_raw_argv` escape), caller-side and opt-in.
- **`cmdit_require_positionals`** — the deferred-from-0.2.0 minimum positional-count
  gate (`CMDIT_ERR_MISSING_POSITIONAL`); works on the global or any verb handle.
  Subsumes cyim's `if (r_npos < N)` guards.
- **`cmdit_verbs_help`** — generated `Commands:` listing (alias rows skipped);
  appended by `cmdit_help` and printed by `cmdit_print_error` on an unknown verb,
  so the command list never drifts from registration.
- **Help/version deferral** — a `--help`/`--version` *after* the verb defers into
  the verb scope (`tool build --help` → verb help), while *before* the verb it
  fires global help (`tool --help` → global help + command list).
- **Error taxonomy** — `CMDIT_ERR_UNKNOWN_VERB=8` (`<prog>: unknown command: <x>`
  + command list) / `CMDIT_ERR_MISSING_POSITIONAL=9` (`missing required argument`),
  both exit 2.
- **Tests** — `tests/cmdit.tcyr` grown 88 → **157 assertions** over verb
  match/alias/unknown, global-before/after, deferral, `--` handling, remainder
  re-parse, nested re-entry, `require_positionals`, and `cmdit_basename`. Demo:
  `programs/verbs.cyr`.

### Changed
- Context struct 104 → 160 B (append-only; entry struct unchanged at 104 B).

### Scope (not subsumed — stays caller-side)
- In-core nested-verb trees (use re-entry on the remainder).
- The argv[0] multiplexer *policy* (which basenames map to which verbs) — cmdit
  ships only `cmdit_basename`.
- Handler invocation / signatures — cmdit returns an id + slice, never calls a handler.
- Interactive REPL / slash-command DSLs (agora/agnoshi/thoth).

## [0.2.0] - 2026-06-25

**Flag modifiers (M2).** Five append-only parse-loop primitives that subsume the
patterns the 2026-06-25 ecosystem survey found hand-rolled across the consumer
repos (re-verified site-by-site against attn11/owl/agnova/cyim/phylax/whirl/
agora/bannermanor/chakshu/ark before landing). Types stay byte-compatible
(`CMDIT_BOOL/INT/STR` = {0,1,2}); the entry struct grows 48→104 B and the error
taxonomy grows — both append-only, old offsets/codes untouched.

### Added
- **`cmdit_enum`** — choice flags. `'|'`-delimited `choices` (e.g.
  `"auto|always|never"`); the parsed value must be a member or parse fails with
  `CMDIT_ERR_BAD_ENUM`. **`cmdit_get_enum`** returns the 0-based matched index
  (the dispatch every consumer re-derived; attn11 `cfg_*` 0..3, owl 0..2, cyim
  1..6); `cmdit_get_str` still returns the raw choice. Subsumes the nested-streq
  validation in attn11/owl/agnova/cyim/phylax.
- **`cmdit_repeat`** — repeatable flags. Each `--flag value` occurrence appends
  to a per-flag list (cap `CMDIT_REPEAT_MAX` = 64, silently capped like
  positionals); **`cmdit_repeat_count`** / **`cmdit_repeat_get`** read it, and
  `cmdit_get_str` gives a last-wins scalar view. Subsumes whirl's `-H` accumulator.
- **`cmdit_required`** — required flags (post-registration marker, any type incl.
  bool). A new **SEEN** bit (set by CLI parse or env) distinguishes absent from
  explicit-false, so confirmation bools like agnova `--i-mean-it` work; absent at
  finalize → `CMDIT_ERR_REQUIRED_MISSING`. **`cmdit_seen`** exposes the bit.
  Subsumes agora `--handle` / agnova `--device`/`--user`/`--i-mean-it` sentinel checks.
- **`cmdit_range`** — inclusive int bounds `[min,max]`, **rejected** (not clamped)
  → `CMDIT_ERR_OUT_OF_RANGE`. Subsumes bannermanor `--width`/`--pad`, attn11
  `--layers`/`--mtp`/`--bpe` caps.
- **`cmdit_env`** — environment-variable fallback (post-registration marker). At
  finalize, an un-SEEN flag draws from `getenv(env_name)`: bool = presence
  (`NO_COLOR`-style), int/str/enum = value (set-but-empty ignored). CLI always
  wins; env can satisfy required; bad env values are **non-fatal** (fall through
  to the default). Subsumes single flag-bound vars (`NO_COLOR`, `ARK_CONFIG`).
- **`cmdit_finalize`** — the impure post-parse stage (env fallback then required
  check). Kept OUT of `cmdit_parse_argv` so the pure synthetic-argv core stays
  `/proc`- and env-free; **`cmdit_parse`** now chains `parse_argv → finalize` on a
  clean parse (help/version still short-circuit first).
- **Error taxonomy** — `CMDIT_ERR_REQUIRED_MISSING=5` / `BAD_ENUM=6` /
  `OUT_OF_RANGE=7` (0.3.0 reserves `UNKNOWN_VERB=8`).
- **Flag-named errors** — `cmdit_print_error` now names the offending flag
  (`<prog>: --color: invalid value (expected auto|always|never)`) for
  value/modifier errors via a new ctx `err_entry`, matching the near-universal
  convention in agora/bannermanor/cyim/attn11. `UNKNOWN` stays generic; no `fmt`
  dependency (numeric range bounds are not rendered). **`cmdit_err_flag`** exposes
  the offending flag's index for consumer-rendered messages.
- **Generated help** — enum renders `<a|b|c>`, repeat appends `...`, required
  appends `(required)` (all string-only, no `fmt`).
- **Tests** — `tests/cmdit.tcyr` grown 26 → **88 assertions** over the modifiers,
  including a purity test asserting `cmdit_parse_argv` ignores a set env var
  (env only applies at `cmdit_finalize`).

### Changed
- Entry struct 48 → 104 B; context 96 → 104 B (both append-only). `CMDIT_FLAGS_MAX`
  unchanged at 64.
- **New consumer dep**: stdlib `"io"` (for `getenv`) is now referenced
  unconditionally by `cmdit_parse` (via `cmdit_finalize`). Pure callers that drive
  `cmdit_parse_argv` directly and never call `cmdit_finalize` link `getenv` but
  never invoke it.

### Scope (not subsumed — stays caller-side)
- Dynamic / large enum lists (owl theme/lang: 45 langs + runtime user themes).
- Cross-field / modulo ranges (attn11 `--rope-dim<=d_model`, `rope_dim%2`,
  `--expert-topk<=experts`).
- Pure-env cascades with no associated flag (`OWL_PAGER>PAGER`, config-path
  builders, `USER>euid`, `ARK_CONFIG`'s privilege gate).
- Positional-count/`required`-positional validation — candidate for 0.3.0.

## [0.1.0] - 2026-06-25

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
