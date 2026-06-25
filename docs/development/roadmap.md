# cmdit — Roadmap

> Milestone plan through v1.0. State lives in [`state.md`](state.md);
> this file is the sequencing — what ships, in what order, against
> what dependency gates.

## v1.0 criteria

_Define before tagging v0.1.0:_

- [ ] Public API frozen — every exported symbol documented and tested
- [ ] Test coverage adequate for the surface area
- [ ] Benchmarks captured in `docs/benchmarks.md`
- [ ] At least one downstream consumer green
- [ ] CHANGELOG complete from v0.1.0 onward
- [ ] Security audit pass (`docs/audit/YYYY-MM-DD-audit.md`)

## Milestones

### M0 — Scaffold (v0.1.0) — ✅ shipped 2026-06-25

- `cyrius init` scaffold landed
- Doc-tree per [first-party-documentation.md](https://github.com/MacCracken/agnosticos/blob/main/docs/development/applications/first-party-documentation.md)
- ADRs / architecture notes / guides / examples folders ready

### M1 — the extraction (v0.1.0) — ✅ shipped 2026-06-25

The faithful fork of stdlib `lib/flags.cyr` → `cmdit_*` (byte-compatible types +
error codes) PLUS the three universal additions every structured consumer
hand-rolled: the `cmdit_argv` materialize bridge, auto `--help`/`--version`,
generated help, and the `CMDIT_EXIT_*` convention; `FLAGS_MAX` 32→64; raw-argv
escape hatch. Pure `cmdit_parse_argv` core, **26/26 tests**, real-argv smoke green.

### M2 — flag modifiers (v0.2.0)

The new parse-loop FlagTypes/modifiers the survey found hand-rolled everywhere
(append-only — types stay byte-compatible, the error taxonomy only grows):
- `cmdit_enum` — choice flags (`auto|always|never`); subsumes the nested-streq
  validation in attn11 `--attn-kind` / owl ×5 / agnova / cyim / phylax.
- `cmdit_repeat` — accumulate flags (whirl `-H`).
- `cmdit_required` — required flags (agora `--handle`, agnova `--i-mean-it`).
- `cmdit_range` — int range validation (bannermanor, chakshu, attn11 caps).
- `cmdit_env` — env-var fallback (`NO_COLOR`, `$ARK_CONFIG`).
- Acceptance: each FD-free unit-tested; the extended taxonomy documented.

### M3 — verb / subcommand dispatch (v0.3.0)

`cmdit_verb` / `cmdit_verb_alias` / `cmdit_dispatch` + nested sub-verbs +
global-flags-before-or-after-verb resolution (fixes ark's double-scan). Covers
the ~9 subcommand tools (ark, sit, takumi, hapi, phylax, agnova, hadara, hoosh).
Multiplexer (kriya, argv[0] basename) via the raw-argv slice + a `cmdit_basename`
helper.

## Adoption (post-0.1.0)

1. **kii** re-folds onto cmdit (seed → first consumer; output must stay identical).
2. The review's Tier-1 drop-ins (anuenue, bannermanor, klug, shakti, yo, whirl),
   then Tier-2 (verb tools, gated on 0.3.0), then Tier-3 (rich/edge: attn11, owl,
   chakshu, dig). Demand-gated, not a forced sweep. See
   [`agnosticos/docs/development/planning/cmdit.md`](https://github.com/MacCracken/agnosticos/blob/main/docs/development/planning/cmdit.md).

## Out of scope (for v1.0)

- Bundled short bools `-abc` and attached short value `-xvalue` (only darshini
  needs bundling — keep the hard-error; revisit as an opt-in v2 mode).
- Count flags `-vvv`, float flags, `--no-foo` negation.
- Interactive REPL / telnet-session / slash-command DSLs (agora/agnoshi/thoth) —
  those are app-specific command languages at a different altitude; cmdit is
  process-argv only.
