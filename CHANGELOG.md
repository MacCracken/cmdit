# Changelog

Format: [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased]

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
