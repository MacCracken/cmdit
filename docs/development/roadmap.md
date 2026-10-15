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

Each migration follows the kii/anuenue recipe. A gap that a migration surfaces ships
append-only (that is how `cmdit_help_flags`, verb introspection, and
`verb_trailing_after` arrived).

## Open maintenance

- **Benchmark re-capture.** [`../benchmarks.md`](../benchmarks.md) is still the v1.0
  capture (pin 6.2.44, 1.0.0 source). To attribute the drift honestly, bisect 1.0.0 →
  current before re-capturing.
- **`cmdit_new` memzero micro-opt.** It zeroes the entry table one byte at a time;
  doing it a word at a time is the known, measured-cold win.

## v2 backlog — confirmed-deferred, not dropped

The re-survey left these caller-side by choice ([`../adr/0003-v1-freeze.md`](../adr/0003-v1-freeze.md)).
Any of them would be an opt-in, append-only addition:

- Bundled short bools `-abc` and attached short value `-xvalue`. Only darshini needs
  bundling, so the hard error stays until there is demand for an opt-in mode.
- Count flags `-vvv`, float flags, `--no-foo` negation.
- Optional-value flags, mutex / required-if sugar.
- Named-positional help, config cascades.

## Out of scope

- Interactive REPL, telnet-session, and slash-command DSLs (agora/agnoshi/thoth).
  Those are app-specific command languages at a different altitude, and cmdit is
  process-argv only.
- Dynamic/large enum lists, cross-field/modulo ranges, pure-env cascades with no
  associated flag, in-core nested-verb trees (use re-entry).
