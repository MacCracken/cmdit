# cmdit — Roadmap

> Forward-looking sequencing only — what is still open, in what order.
> State lives in [`state.md`](state.md); what already shipped lives in
> [`../../CHANGELOG.md`](../../CHANGELOG.md), with the *why* in [`../adr/`](../adr/).

## Shipped

v1.0.0 (2026-06-25) froze the public API. Every v1.0 criterion and milestones M0–M4
are done (extraction, flag modifiers, verb dispatch, completeness delta + freeze). 1.1,
1.2.x and 1.3.x are append-only on the frozen surface. 1.3.2 was a full audit sweep
([`../audit/2026-10-14-audit.md`](../audit/2026-10-14-audit.md)), and 1.3.3 shipped the
first pinned survey gap, `cmdit_err_token` / `cmdit_err_value`. For details, see the CHANGELOG and
ADRs [0001](../adr/0001-flag-modifiers.md) / [0002](../adr/0002-verb-dispatch.md) /
[0003](../adr/0003-v1-freeze.md).

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

## Pinned for 1.3.x

The 2026-10-14 adoption survey of the 18 hand-rolled candidates found where cmdit cannot
express a real tool's CLI. Its two worst gaps shipped in 1.3.1 (silent positional/repeat caps,
`--bool=0`), and the offending-token accessor (seven repos) in 1.3.3. The rest are pinned
below, one theme per release, ordered by how many repos each unblocks. Every item is
append-only on the frozen 1.0.0 surface.

- [ ] **1.3.4 — stop at the first positional.** A top-level setter so every token after the
  first positional is a positional, reusing the per-verb trailing-verbatim mechanism
  (`cmdit_verb_trailing_after`). The stdlib precedent is `flags_set_stop_at_positional`.
  *Unblocks:* shakti (its last blocker since 1.3.3), kriya's `xargs`/`env`.
- [ ] **1.3.5 — output to a chosen fd, and hidden verbs.**
  - fd-taking variants of `cmdit_help` / `cmdit_help_flags` / `cmdit_verbs_help`. 14 of the
    18 candidates print help on stdout; anuenue's stderr choice stays the default.
  - Hidden verbs, kept out of help, the verb list and completions (hoosh `bench`).
- [ ] **1.3.6 — small modifiers**, each requested by one repo:
  - a once-only flag, or a per-flag occurrence count (cyim; 9 smoke cases assert it);
  - case-insensitive enum matching (phylax `--severity-threshold`);
  - a second long name for a flag (ark `--marketplace|--market`);
  - refusing a flag-shaped value (hapi audit F-019).
- [ ] **1.3.7 — flag-table introspection.**
  - `cmdit_flag_*_at` accessors mirroring the 1.2.0 verb ones (kriya's expander and
    `--help=json`).
  - Per-verb flags in `cmdit_completions`, which needs them (ark).

**Filed against cyrius:** stdlib `args_init` returns a prefix of a command line longer than
2 MiB with no error (ARG_MAX rises with `ulimit -s`). That is
cyrius `docs/development/issues/2026-10-14-args-init-truncates-argv-past-2-mib.md`, for cyrius to place and repair. Since 1.3.2, cmdit fails closed with
`CMDIT_ERR_ARGV` instead of parsing the prefix (audit SEC-7). Revisit that check when cyrius
closes the issue.

**Not pinned:** negative-number positionals such as `hadara at -1000`;
`cmdit_verb_trailing_after(h, v, 0)` already handles them, and 1.3.4 will cover the top level.
Attached short values (sit `-U3`), clusters (kriya), optional-value flags (cyim `--wc[=l]`,
chakshu `--watch [PATH]`) and argv order between two flags (attn11 `--preset`, owl `-n`/`-N`)
are v2 backlog, below.

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
