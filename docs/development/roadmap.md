# cmdit — Roadmap

> Milestone plan through v1.0. State lives in [`state.md`](state.md);
> this file is the sequencing — what ships, in what order, against
> what dependency gates.

## v1.0 criteria

The bar for tagging cmdit v1.0 (freeze the public API). **All met in the 1.0.0 cut
(M4, 2026-06-25):**

- [x] Public API frozen — every exported symbol documented (README "API reference")
  and tested; the public/internal boundary is delimited in [`../adr/0003-v1-freeze.md`](../adr/0003-v1-freeze.md)
- [x] Test coverage adequate for the surface area — **230/230**, incl. the full
  output/renderer surface (was 157; +73 over renderers, parse/dispatch gaps, and C5/C3/C1)
- [x] Benchmarks captured in [`../benchmarks.md`](../benchmarks.md)
- [x] At least one downstream consumer green — **kii 1.1.0** (the re-fold; 468/468 tests, render verified)
- [x] CHANGELOG complete from v0.1.0 onward (all sections dated 2026-06-25)
- [x] Security audit pass — [`../audit/2026-06-25-audit.md`](../audit/2026-06-25-audit.md)

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

### M2 — flag modifiers (v0.2.0) — ✅ shipped 2026-06-25

The new parse-loop modifiers the survey found hand-rolled everywhere (append-only
— types stay byte-compatible, the error taxonomy only grows). Re-grounded
site-by-site against the consumer repos before landing:
- `cmdit_enum` — choice flags (`auto|always|never`) + `cmdit_get_enum` index
  dispatch; subsumes the nested-streq validation in attn11 `--attn-kind` /
  owl ×5 / cyim / phylax.
- `cmdit_repeat` — accumulate flags (whirl `-H`) + count/get accessors.
- `cmdit_required` — required flags (agora `--handle`, agnova `--i-mean-it`) on a
  SEEN bit (+ `cmdit_seen`), checked in `cmdit_finalize`.
- `cmdit_range` — inclusive int bounds, reject-not-clamp (bannermanor, attn11 caps).
- `cmdit_env` — env-var fallback (`NO_COLOR`, `$ARK_CONFIG`), CLI-wins, non-fatal.
- Plus: flag-named errors, enriched help, the impure/pure split (`cmdit_finalize`
  vs `cmdit_parse_argv`). **88/88 tests** (incl. a parse-core purity test); the
  extended taxonomy documented. See [`../adr/0001-flag-modifiers.md`](../adr/0001-flag-modifiers.md).
- Deferred to 0.3.0: `cmdit_require_positionals` (positional-count validation).

### M3 — verb / subcommand dispatch (v0.3.0) — ✅ shipped 2026-06-25

`cmdit_verb` / `cmdit_verb_alias` / `cmdit_dispatch` (identify-and-return) +
global-flags-before-or-after-verb resolution in one pass (fixes ark's double-scan)
+ `cmdit_verb_matched` / `cmdit_verb_argc/argv` + `cmdit_parse_verb`. Covers the ~9
subcommand tools (ark, sit, takumi, hapi, phylax, agnova, hadara, hoosh, cyim).
Multiplexer (kriya, argv[0] basename) via `cmdit_raw_argv` + a `cmdit_basename`
helper. Plus the deferred `cmdit_require_positionals` and a generated
`cmdit_verbs_help` command list. **157/157 tests**; demo `programs/verbs.cyr`.
See [`../adr/0002-verb-dispatch.md`](../adr/0002-verb-dispatch.md).

**The roadmap bent on "nested sub-verbs":** shipped as re-entry on the remainder
slice (a child handle re-dispatches), not an in-core verb tree — every production
tool is depth ≤ 2 and the re-entry model is unbounded with zero extra API.

### M4 — completeness delta + freeze (v1.0.0) — ✅ shipped 2026-06-25

The final surface a 23-repo re-survey of hand-rolled CLI code found still uncovered,
plus the v1.0 hardening + freeze. Append-only (entry 104→112 B; error codes grow to
`TOO_MANY_POSITIONAL = 10`; the pure cores untouched):
- `cmdit_help_short` / `cmdit_version_short` — remap/disable the auto `-h`/`-V` short
  letters (unblocks the `df -h`/`ls -h` human-readable convention).
- `cmdit_metavar` + shown string defaults in generated help.
- `cmdit_require_positionals_max` / `_exact` — the upper-bound / both-bounds mirrors.
- Two audit fixes (int-overflow range-bypass, unguarded getter idx); the full
  renderer surface tested; **230/230**; benchmarks + security-audit doc; API frozen.

See [`../adr/0003-v1-freeze.md`](../adr/0003-v1-freeze.md). The v2 backlog (mutex/
required-if sugar, optional-value flags, bundled shorts, `--no-foo`, named-positional
help, config cascades) is confirmed-deferred by the re-survey, not dropped.

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
