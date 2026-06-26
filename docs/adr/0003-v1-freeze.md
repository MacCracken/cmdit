# 0003 — v1.0 API freeze: the completeness delta, and what is (not) frozen

**Status**: Accepted
**Date**: 2026-06-25

## Context

cmdit reached feature-completeness across the 0.1→0.3 arc (extraction → flag
modifiers → verb dispatch). Tagging v1.0 means **freezing the public API** — names,
signatures, and semantics become stable. Two grounding passes preceded the freeze:

1. A **23-repo re-survey** of hand-rolled CLI code across the consumer ecosystem
   (Tier-1/2/3), to confirm nothing reusable was still left to cmdit's callers. It
   found the surface substantially complete — every high-prevalence pattern
   (cross-flag validation, `--no-foo`, bundled shorts, config cascades) resolves to
   caller-side or already-deferred — but surfaced three genuinely-uncovered,
   reusable gaps: configurable `-h`/`-V` short letters, per-flag metavar + shown
   defaults, and a max/exact positional gate (the min gate already shipped).
2. An **adversarial security + readiness audit**, which (a) found no
   attacker-reachable memory-safety defect, (b) confirmed two low-severity
   correctness bugs (int-overflow range-bypass, unguarded getter idx), and (c)
   flagged two freeze-relevant facts about the *constant* surface: value collisions
   across the result-code and exit-code namespaces, and internal layout constants
   exported under the public `CMDIT_*` prefix.

A freeze is one-way: a wrong signature or an under-specified constant becomes a
permanent contract. So the surface had to be *finished* and *delimited* before
freezing, not frozen as-found.

## Decision

**Land the three-item completeness delta, fix the two confirmed bugs, then freeze —
and explicitly delimit what "frozen public API" covers.**

- **Completeness delta (append-only):** `cmdit_help_short` / `cmdit_version_short`
  (remap/disable the auto short letters), `cmdit_metavar` + shown string defaults in
  generated help, and `cmdit_require_positionals_max` / `_exact`
  (`CMDIT_ERR_TOO_MANY_POSITIONAL = 10`). Entry grows 104→112 B for the metavar;
  `CMDIT_BOOL/INT/STR` = {0,1,2}, all prior offsets, and error codes 0..9 are
  untouched.
- **The colliding result/exit codes are intentional and stay** (`CMDIT_OK` =
  `CMDIT_EXIT_OK` = 0; `CMDIT_HELP` = `CMDIT_EXIT_USAGE` = 2). They are **not
  renumbered**: the result codes are byte-compatible-inherited from stdlib
  `flags.cyr`, and the first consumer (kii 1.1.0) already depends on them. They are
  documented as two distinct namespaces (parse result vs process exit) that must not
  be substituted for one another.
- **Frozen surface ≠ every `CMDIT_*` symbol.** The frozen public API is the function
  set + the result/exit codes + the `CmditType` / `CmditErr` enums (listed in the
  README "API reference"). The layout/cap constants — `CMDIT_ENTRY_SIZE`,
  `CMDIT_CTX_SIZE`, `CMDIT_*_MAX`, `CMDIT_VERB_SIZE`, and the `CMDIT_ATTR_*`
  bitfield — are **internal**: their *values* may change (e.g. a cap raised) and
  consumers must not depend on them. They remain visible only because the library is
  a single included module.
- **Bug fixes land before the freeze** (the guarded behavior is itself part of the
  frozen contract): the `_cmdit_parse_int` overflow rejection and the
  `_cmdit_idx_ok` getter guards.

## Consequences

- **Positive** — the frozen surface is complete (the positional gate, auto-help
  generator, and short-letter binding are now coherent), the two correctness bugs
  are out of the permanent contract, and the public/internal boundary is explicit so
  later internal-constant changes are non-breaking. Coverage is 230/230 incl. the
  full renderer surface; benchmarks and a security-audit pass are recorded.
- **Negative** — the entry struct grew again (104→112 B; 64 × 112 = 7168 B/handle),
  and the result/exit value collisions remain a latent footgun for a consumer that
  conflates the namespaces (mitigated by documentation, not by type).
- **Neutral** — a sizeable v2 backlog stays caller-side by deliberate choice (mutex/
  required-if sugar, optional-value flags, bundled shorts, `--no-foo`, named-positional
  help); each is recorded as confirmed-deferred, not forgotten.

## Alternatives considered

- **Freeze the 0.3.0 surface as-is.** Rejected: the re-survey showed `-h`-hard-binding
  is an active adoption blocker (fights `df -h`/`ls -h`), and a min-only positional
  gate is an asymmetric public API — both cheaper to fix before the freeze than after.
- **Renumber the colliding result/exit codes** to remove the value aliasing. Rejected:
  it breaks `flags.cyr` byte-compatibility and the existing kii consumer for a
  cosmetic gain; documentation resolves the ambiguity without a breaking change.
- **Freeze every `CMDIT_*` constant as public.** Rejected: it would pin internal
  layout/caps (`CMDIT_FLAGS_MAX`, struct sizes) as contract, blocking routine internal
  tuning; an explicit public/internal split keeps the freeze meaningful.
- **A throwaway 0.4.0 for the delta, then 1.0.0.** Rejected: nothing was ever released
  (all prior versions were Unreleased) and the delta + all freeze gates land together,
  so a single 1.0.0 cut is cleaner than an intermediate tag.
