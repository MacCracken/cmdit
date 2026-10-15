# cmdit — Roadmap

> Forward-looking only: what is still open. State lives in [`state.md`](state.md); what
> shipped lives in [`../../CHANGELOG.md`](../../CHANGELOG.md), with the *why* in
> [`../adr/`](../adr/).

## Open — consumer requests and issues

**Nothing is scheduled.** The public API froze at 1.0.0. Every gap the 2026-10-14 adoption
survey pinned shipped in 1.3.x, and 1.4.0 was the refactor and optimization sweep. New work
comes from consumers: a gap a migration runs into, or a defect someone hits.

Each one is filed here as an item that names:

- the consumer;
- the code it would replace (`file:line`);
- the smallest addition that covers it.

Items ship append-only on the frozen surface, the way `cmdit_help_flags`, verb introspection
and `cmdit_verb_trailing_after` arrived. A defect in cyrius or its stdlib is not fixed here.
It is filed as an issue in cyrius `docs/development/issues/`, and cmdit works around it on
its own side if it must.

_No open items._

## Watching

- **cyrius: `args_init` truncates a command line over 2 MiB, with no error** (ARG_MAX rises
  with `ulimit -s`). Filed as cyrius
  `docs/development/issues/2026-10-14-args-init-truncates-argv-past-2-mib.md`, for cyrius to
  place and repair. Since 1.3.2, cmdit fails closed with `CMDIT_ERR_ARGV` instead of parsing
  the prefix (audit SEC-7). Revisit that check when the issue closes.

## Adoption — demand-gated, not a forced sweep

Adopted (`[deps.cmdit]`): kii, anuenue, stiva, ifran, dig, commandress. See
[`state.md`](state.md#consumers) for pins.

Still hand-rolled, from the 2026-06-25 ecosystem CLI review
([`agnosticos/docs/development/planning/cmdit.md`](https://github.com/MacCracken/agnosticos/blob/main/docs/development/planning/cmdit.md)):

1. **Tier-1 drop-ins** — bannermanor, klug, shakti, yo, whirl.
2. **Tier-2 verb tools** — ark, sit, takumi, hapi, phylax, agnova, hadara, hoosh, cyim;
   the kriya `argv[0]` multiplexer via `cmdit_basename` + `cmdit_raw_argv`.
3. **Tier-3 rich/edge** — attn11, owl, chakshu.

Each candidate's `docs/development/roadmap.md` carries an **"Adopting cmdit (CLI parsing)"**
section (filed 2026-10-14). It maps that repo's hand-rolled sites (`file:line`) to cmdit
calls, names the gaps, and lists the tests and goldens that must move. Each migration follows
the kii/anuenue recipe (stiva for verb tools). A gap a migration runs into becomes an item
under "Open" above.

## v2 backlog — confirmed-deferred, not dropped

The re-survey left these caller-side by choice ([`../adr/0003-v1-freeze.md`](../adr/0003-v1-freeze.md)).
Any of them would be an opt-in, append-only addition, and a consumer request moves one up
to "Open":

- Bundled short bools `-abc` and attached short value `-xvalue`. The 2026-10-14 survey found
  more demand than the 1.0 re-survey did: kriya (`-rf`, `-n5`, solved with a pre-parse
  expander that 1.3.7's `cmdit_flag_find_short` now serves) and sit (`-U<N>`), besides
  darshini.
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
