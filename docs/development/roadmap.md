# cmdit — Roadmap

> Forward-looking sequencing only — what is still open, in what order.
> State lives in [`state.md`](state.md); what already shipped lives in
> [`../../CHANGELOG.md`](../../CHANGELOG.md), with the *why* in [`../adr/`](../adr/).

## Shipped

v1.0.0 (2026-06-25) froze the public API. Every v1.0 criterion and milestones M0–M4
are done (extraction, flag modifiers, verb dispatch, completeness delta + freeze). 1.1
and 1.2.x are append-only on the frozen surface. For details, see the CHANGELOG and
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

## Gaps found by the adoption survey (2026-10-14) — open, deferred past 1.3.0

Surveying the 18 hand-rolled candidates site by site found places where cmdit cannot
express a real tool's CLI, or does something wrong. Several are places where the stdlib
`lib/flags.cyr` cmdit was forked from has **since moved past cmdit**. Ranked by how many
repos each blocks or affects:

1. **The 128-positional cap drops silently.** Blocks shakti: a privileged command line would
   be truncated with exit 0, and policy, audit and `execve` would all see the short version.
   Regresses bannermanor and kriya, which already rely on the stdlib's fix (kriya's
   `rm *` over 200 files deleted 128 and exited 0 until stdlib 6.6.5). Also affects cyim's
   `--grepfiles`, which sizes its file list from argc. The stdlib grows its
   array since 6.6.5 and fails with `FLAG_ERR_NOMEM` since 6.6.7. The 1.0.0 audit accepted
   cmdit's drop as SEC-3 *because it matched the stdlib*; that justification no longer holds.
   Remedy: grow on demand (internal, no API). On alloc failure, fail with
   `CMDIT_ERR_TOO_MANY_POSITIONAL`. The same rule caps a repeat flag at 64.
2. **The offending token is not reported.** `cmdit_print_error` says "unknown flag" and keeps
   no copy of the token or rejected value. Seven repos echo it today, and shakti's
   integration test asserts it: shakti, attn11, owl, chakshu, hapi, phylax, agnova. The
   unknown-*verb* token is already stored (ctx +152). The stdlib precedent is
   `flags_error_arg`. Remedy: append-only `cmdit_err_token(h)`. Optionally include the token
   in `cmdit_print_error` too, which would change its output for current consumers.
3. **`--bool=value` is accepted and the value discarded**, so `--verbose=0` turns the flag
   *on*. This affects bannermanor, klug, shakti, kriya and cyim, and the stdlib fixed it in
   6.6.5 (`FLAG_ERR_UNEXPECTED_VALUE`). Remedy: a new `CMDIT_ERR_UNEXPECTED_VALUE = 11`. This
   rejects input that parses today, so it is a behaviour change.
4. **No top-level "stop at the first positional."** Blocks shakti, where the command owns
   every later token (`shakti ls -la`). kriya's `xargs`/`env` need it too. The mechanism
   exists per verb (`cmdit_verb_trailing_after`). The stdlib precedent is
   `flags_set_stop_at_positional`. Remedy: an append-only top-level setter.
5. **Help renders only to stderr.** The survey found 14 of the 18 candidates printing help on
   stdout, but most keep hand-written help for byte identity, so this rarely blocks. anuenue
   chose stderr on purpose, so this is a policy question. Remedy: an append-only fd-taking
   variant.
6. **Weaker, single-repo demand:**
   - Attached short values `-U3` (sit `diff`/`show`; v2 backlog).
   - Optional-value flags: cyim `--wc[=l]`, chakshu `--watch [PATH]`, kriya (v2 backlog).
   - Clusters (kriya; v2 backlog).
   - Refusing a repeated flag, or a per-flag occurrence count (cyim; asserted by 9 smoke cases).
   - Negative-number positionals such as `hadara at -1000`. `cmdit_verb_trailing_after(h, v, 0)`
     already works around it.
   - Hidden verbs, kept out of help and completions (hoosh `bench`).
   - Public flag-table introspection (kriya's expander and `--help=json`).
   - Refusing a flag-shaped value (hapi audit F-019).
   - Case-insensitive enums (phylax).
   - Long-name aliases (ark).
   - Per-verb flags in `cmdit_completions` (ark).
   - argv-order between two different flags (attn11 `--preset`, owl `-n`/`-N`; the v2
     `--no-foo` item).

Items 1–4 would also unblock the "blocked" notes in shakti, bannermanor and kriya. 1.3.0
ships without them by decision (2026-10-14); items 1 and 3 are the first candidates for the
next release.

## v2 backlog — confirmed-deferred, not dropped

The re-survey left these caller-side by choice ([`../adr/0003-v1-freeze.md`](../adr/0003-v1-freeze.md)).
Any of them would be an opt-in, append-only addition:

- Bundled short bools `-abc` and attached short value `-xvalue`. The 2026-10-14 survey found
  more demand than the 1.0 re-survey did: kriya (`-rf`, `-n5`, solved with a pre-parse
  expander) and sit (`-U<N>`), besides darshini.
- Count flags `-vvv`, float flags, `--no-foo` negation.
- Optional-value flags, mutex / required-if sugar.
- Named-positional help, config cascades.

## Out of scope

- Interactive REPL, telnet-session, and slash-command DSLs (agora/agnoshi/thoth).
  Those are app-specific command languages at a different altitude, and cmdit is
  process-argv only.
- Dynamic/large enum lists, cross-field/modulo ranges, pure-env cascades with no
  associated flag, in-core nested-verb trees (use re-entry).
