# cmdit — Roadmap

> Forward-looking sequencing only — what is still open, in what order.
> State lives in [`state.md`](state.md); what already shipped lives in
> [`../../CHANGELOG.md`](../../CHANGELOG.md), with the *why* in [`../adr/`](../adr/).

## Shipped

v1.0.0 (2026-06-25) froze the public API. Every v1.0 criterion and milestones M0–M4 are done
(extraction, flag modifiers, verb dispatch, completeness delta + freeze). 1.1, 1.2.x and 1.3.x
are append-only on the frozen surface. 1.3.2 was a full audit sweep
([`../audit/2026-10-14-audit.md`](../audit/2026-10-14-audit.md)), and 1.3.3 shipped the first
pinned survey gap, `cmdit_err_token` / `cmdit_err_value`; 1.3.4 shipped
`cmdit_stop_at_positional`; 1.3.5 the `_fd` help renderers and hidden verbs; 1.3.6
`cmdit_once` / `cmdit_enum_nocase` / `cmdit_long_alias` / `cmdit_reject_flag_value`; 1.3.7
flag-table introspection and per-verb flags in completions, the last pinned item. For
details, see the CHANGELOG and ADRs [0001](../adr/0001-flag-modifiers.md) /
[0002](../adr/0002-verb-dispatch.md) / [0003](../adr/0003-v1-freeze.md).

## Adoption — demand-gated, not a forced sweep

Adopted (`[deps.cmdit]`): kii, anuenue, stiva, ifran, dig, commandress. See
[`state.md`](state.md#consumers) for pins.

Still hand-rolled, from the 2026-06-25 ecosystem CLI review
([`agnosticos/docs/development/planning/cmdit.md`](https://github.com/MacCracken/agnosticos/blob/main/docs/development/planning/cmdit.md)):

1. **Tier-1 drop-ins** — bannermanor, klug, shakti, yo, whirl.
2. **Tier-2 verb tools** — ark, sit, takumi, hapi, phylax, agnova, hadara, hoosh,
   cyim; the kriya `argv[0]` multiplexer via `cmdit_basename` + `cmdit_raw_argv`.
3. **Tier-3 rich/edge** — attn11, owl, chakshu.

Each candidate's own `docs/development/roadmap.md` now carries an **"Adopting cmdit (CLI
parsing)"** section (filed 2026-10-14). Each one maps that repo's hand-rolled sites
(`file:line`) to cmdit calls, names the gaps, and lists the tests and goldens that must move.
Each migration follows the kii/anuenue recipe (stiva for verb tools). A gap that a migration
surfaces ships append-only (that is how `cmdit_help_flags`, verb introspection, and
`verb_trailing_after` arrived).

## 1.3.x survey gaps — all shipped

The 2026-10-14 adoption survey of the 18 hand-rolled candidates found where cmdit could not
express a real tool's CLI. Every gap it pinned has shipped, append-only on the frozen 1.0.0
surface: the silent positional/repeat caps and `--bool=0` (1.3.1), the offending-token
accessor (1.3.3), POSIX stop-at-first-positional (1.3.4), help on a chosen stream and hidden
verbs (1.3.5), four small modifiers (1.3.6), and flag-table introspection with per-verb
completion flags (1.3.7). What the survey found but did not pin is under "Not pinned" below
and in the v2 backlog.

**Filed against cyrius:** stdlib `args_init` returns a prefix of a command line longer than 2
MiB with no error (ARG_MAX rises with `ulimit -s`). That is cyrius
`docs/development/issues/2026-10-14-args-init-truncates-argv-past-2-mib.md`, for cyrius to
place and repair. Since 1.3.2, cmdit fails closed with `CMDIT_ERR_ARGV` instead of parsing the
prefix (audit SEC-7). Revisit that check when cyrius closes the issue.

**Not pinned:** negative-number positionals such as `hadara at -1000`;
`cmdit_verb_trailing_after(h, v, 0)` handles them for a verb, and `cmdit_stop_at_positional`
(1.3.4) once the first positional is past.
Attached short values (sit `-U3`), clusters (kriya), optional-value flags (cyim `--wc[=l]`,
chakshu `--watch [PATH]`) and argv order between two flags (attn11 `--preset`, owl `-n`/`-N`)
are v2 backlog, below.

## 1.4.0 — refactor and optimization sweep (planned)

No new surface. 1.4.0 pays down what the sixteen append-only releases since the 1.0.0 freeze
left in the source, and the costs measured on 2026-10-14 (1.3.7 source, idle host).
**Contract: the public API, the error codes and every byte of output stay as they are** —
help, error lines and all three completion scripts. The suite passes unmodified, except where
a test reaches into an internal helper.

**Baseline** ([`../benchmarks.md`](../benchmarks.md), plus an output-path probe):

- Construction is most of a parse: `cmdit_new` 2.05 µs of `parse_argv_mixed`'s 2.42 µs. The
  first `cmdit_verb` adds 1.31 µs, zeroing a 6,144 B verb table whatever the verb count.
- Output dominates the `--help` path. `cmdit_help_fd` on a 7-flag, 3-verb handle makes **93
  write(2) calls** and costs **~34 µs** to /dev/null, about 14× a whole parse. One
  `cmdit_print_error` line is 13 writes. The completion scripts are 113 (bash), 141 (zsh) and
  238 (fish) writes, because the quoting helpers write one byte per syscall.

### Refactor — no behaviour change

- [ ] **R1 — named offsets.** The source addresses the context, entry and verb row by literal
  offset: 122 lines load or store a ctx field that way, and about 220 `h`/`ep`/`row + N`
  expressions in all. 1.3.5's stride bug
  (`cmdit_verbs_help_fd` read +32 before rows grew to 40 B) is that class. Replace them with
  private enum members (`_CMDIT_H_VERBS = 104`, ...). **Proof:** add the members in one step,
  swap the literals in the next, and require the second build to be byte-identical to the
  first. Checked on 2026-10-14: a swapped enum member compiles to exactly the literal's code.
- [ ] **R2 — one token-to-flag path.** Resolving a flag token, taking its value, the 1.3.6
  flag-shaped-value refusal and the SEEN mark are written three times: the parse core's long
  and short branches, and the dispatcher's `_cmdit_disp_apply_flag`. 1.3.2 fixed a
  BUNDLED/UNKNOWN divergence between them, and 1.3.6 had to add its refusal at all three
  sites. Fold them into one resolver and one take-value-and-apply helper that both cores
  call. Gate: the suite, fuzz I1–I9 (which drive both cores), and an A/B on
  `parse_argv_mixed` and `dispatch_*`. A helper call per flag is the risk, so the budget is
  +3%.
- [ ] **R3 — shared guards and helpers.**
  - A `_cmdit_verb_canon(h, id)` for the guard ladders in `cmdit_verb_trailing_after`,
    `cmdit_verb_trailing_at` and `cmdit_verb_hidden`; 1.3.7's `cmdit_verb_flags` already uses
    the short form. Their `verbs_ptr == 0` check is redundant with the count check, because
    the count is 0 until the table exists.
  - One `_cmdit_zero(p, n)` for the two table-zeroing loops.
  - One fd writer behind `_cmdit_eprint` / `_cmdit_oprint` / `_cmdit_fprint`, which O1
    builds on.
- [ ] **R4 — the test suite in per-group functions.** `tests/cmdit.tcyr` is one ~2,300-line
  `main` whose `var` names must all be unique, and collisions broke the build in several
  1.3.x releases. Split it per group. The assertion count staying at 786 is the check.

### Optimization — each one measured

- [ ] **O1 — buffered output.** Each rendering call fills a fixed buffer, flushes it when
  full, and flushes at return, so ordering against the caller's own writes is unchanged.
  Target: help 93 writes → ~1, an error line 13 → 1, a completion script 100–240 → a few. An
  error line then also reaches a pipe in one write(2), so another process's stderr cannot
  split it. Gate: every capture byte-identical (the suite's capture helpers, plus a fixture
  diff of all three scripts as in 1.3.7), and new `help_fd` / `completions` /
  `print_error` benches to /dev/null.
- [ ] **O2 — lazy verb table.** Allocate 16 rows and double up to the 128 cap, as positionals
  have since 1.3.1, instead of 128 rows on the first `cmdit_verb`. sit has 25 verbs and stiva
  36. Expected: `verb_new_floor` drops to about `cmdit_new_floor` +
  0.2 µs. The cap and the -1-when-full return stay.
- [ ] **O3 — lazy entry table (evaluate; may not ship).** The 64-entry table (7,680 B) is most
  of `cmdit_new`, and most tools register under 16 flags. Unlike the verb table, a growth
  would leave `err_entry` (ctx +96, an entry pointer) dangling. That field would first become
  an index, append-only, or a registration after a parse would have to be shown harmless.
  Ship it only if the A/B shows a clear gain on `cmdit_new_floor` with no other regression.
- [ ] **O4 — long-name lookup.** The lookup is a linear scan with a full compare per entry,
  80 ns over six flags. Compare the first byte, or a stored length, before
  `_cmdit_streq_bounded`. Decide with a new 35-flag bench (attn11's size) first.

### Method and gates

- **Order:** benches first (the output paths, the 35-flag parse) and an idle-host baseline
  (load average < 2), then R1–R4, then O1–O4, one change per step.
- **Per step:**
  - suite, fuzz and poisoned fuzz, `CYRIUS_POISON=1` suite;
  - both lint counts, `fmt` / `doc --check`, `distlib --check`, and the bundle compiled on
    6.6.2;
  - mutation runs for every touched branch, with a no-op control mutant run first;
  - an interleaved A/B of at least 11 runs.
- **Output identity:** capture every renderer and all three completion scripts for a fixture
  set before R1, and diff them after every step.
- **Exit:** a CHANGELOG with before/after per bench, `benchmarks.md` re-captured, and no
  existing bench more than 3% worse.

**Not in 1.4.0:** new API (append-only additions go to 1.4.x or the v2 backlog), any output
change, caching `cmdit_get_enum` (callers cache it, as documented), and a toolchain pin move
unless the sweep needs one.

## v2 backlog — confirmed-deferred, not dropped

The re-survey left these caller-side by choice ([`../adr/0003-v1-freeze.md`](../adr/0003-v1-freeze.md)).
Any of them would be an opt-in, append-only addition:

- Bundled short bools `-abc` and attached short value `-xvalue`. The 2026-10-14 survey found
  more demand than the 1.0 re-survey did: kriya (`-rf`, `-n5`, solved with a pre-parse
  expander) and sit (`-U<N>`), besides darshini.
- Optional-value flags: cyim `--wc[=l]`, chakshu `--watch [PATH]`, kriya's optional-value
  longs.
- Count flags `-vvv`, float flags, `--no-foo` negation (it would settle owl's `-n`/`-N`).
- A per-flag argv position, for precedence between two different flags (attn11 `--preset`
  against explicit dimensions).
- Mutex / required-if sugar.
- Named-positional help, config cascades.

## Out of scope

- Interactive REPL, telnet-session, and slash-command DSLs (agora/agnoshi/thoth).
  Those are app-specific command languages at a different altitude, and cmdit is
  process-argv only.
- Dynamic/large enum lists, cross-field/modulo ranges, pure-env cascades with no
  associated flag, in-core nested-verb trees (use re-entry).
