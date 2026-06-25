# 0001 — Flag modifiers as append-only entry fields with a pure/impure parse split

**Status**: Accepted
**Date**: 2026-06-25

## Context

cmdit 0.1.0 forked the stdlib `flags.cyr` getopt-long core with a 48-byte,
byte-compatible entry struct (`+0` short_ch … `+40` help) and a pure
`cmdit_parse_argv` that the whole test suite drives with synthetic argv (no
`/proc`, no env). The 0.2.0 milestone adds five modifiers — `enum`, `repeat`,
`required`, `range`, `env` — that a 2026-06-25 ecosystem survey (re-verified
site-by-site against attn11/owl/agnova/cyim/phylax/whirl/agora/bannermanor/
chakshu/ark) found hand-rolled everywhere.

Three constraints make this a real decision rather than a default:
1. **Byte-compatibility** with `flags.cyr` is a stated invariant — the `CmditType`
   values `{0,1,2}` and the existing entry offsets must not move.
2. Two modifiers need per-flag metadata that doesn't fit the 6-slot entry
   (enum choices, env name, range bounds, a repeat list + count, a required/seen
   bitfield).
3. `required` and `env` are inherently impure (env reads the environment;
   required is a post-parse cross-flag check), but the 26 existing tests depend on
   `cmdit_parse_argv` being deterministic and environment-free.

## Decision

**Grow the entry struct append-only (48 → 104 B) and keep the new impure logic in
a separate `cmdit_finalize`, leaving `cmdit_parse_argv` pure.**

- Modifiers are **attributes/fields on top of the byte-compatible core**, not new
  `CmditType` values: enum/repeat are underlying `CMDIT_STR`; required/range/env
  are post-registration markers. New fields live at `+48..+96`
  (`attrs` bitfield, `choices`, `env_name`, `range_min/max`, `repeat_list/count`);
  `+0..+40` are untouched.
- enum/range validation is **pure** and stays in `cmdit_parse_argv`. The impure
  env fallback (then the required check, so env can satisfy required) lives in
  `cmdit_finalize`; `cmdit_parse` chains `parse_argv → finalize` on a clean parse,
  with help/version short-circuiting first.
- enum exposes a **`cmdit_get_enum` index** (0-based) — the survey showed
  consumers overwhelmingly store the matched choice as an int for dispatch, so a
  string-only enum would not actually replace the streq chains it targets.
- A **SEEN bit** (set by CLI parse or env, never inside `_cmdit_apply`) backs both
  `required` and `cmdit_seen`, so a required bool distinguishes absent from
  explicit-false.
- `range` **rejects** out-of-bounds (`OUT_OF_RANGE`) rather than clamping; `env`
  is **non-fatal** (a bad value falls through to the default); `cmdit_print_error`
  **names the offending flag** for value/modifier errors via a ctx `err_entry`.

In scope: static choice sets, constant int bounds, single flag-bound env vars.
Out of scope (stays caller-side): dynamic/large enum lists, cross-field/modulo
ranges, pure-env cascades with no associated flag.

## Consequences

- **Positive** — byte-compatibility and the deterministic synthetic-argv test
  model are both preserved; the modifiers drop into the existing parse loop with
  no new `CmditType` branches; consumers get index dispatch, flag-named errors,
  and env/required "for free"; the pure core remains unit-testable FD-free.
- **Negative** — per-flag storage more than doubles (64 × 104 B = 6656 B/ctx, plus
  512 B per repeat flag); `cmdit_parse` now references `getenv` unconditionally, so
  every consumer needs stdlib `"io"`; `CMDIT_ENTRY_SIZE`/`CMDIT_CTX_SIZE` are now
  load-bearing single-source-of-truth constants the zero-loop and allocs depend on.
- **Neutral** — SEEN conflates CLI-set and env-set (a CLI-only escape hatch can be
  added later if a consumer needs it); `cmdit_get_enum` re-walks the choices cstr
  per call (callers cache after parse); `cmdit_require_positionals` is deferred to
  0.3.0.

## Alternatives considered

- **New `CmditType` values (CMDIT_ENUM/CMDIT_REPEAT).** Rejected: it would still
  need the side fields, complicates the type dispatch, and risks the
  byte-compatibility story; attributes on `CMDIT_STR` are simpler and append-only.
- **Env/required inside `cmdit_parse_argv`.** Rejected outright: `getenv` in the
  pure core makes every synthetic-argv test environment-dependent and flaky,
  violating the M2 "each FD-free unit-tested" bar. The pure/impure split is the
  load-bearing decision here.
- **String-only enum (no index accessor).** Rejected: consumers would re-`streq`
  the returned string, re-introducing the exact boilerplate cmdit removes.
- **Clamp out-of-range values; hard-error on bad env.** Inverted per the evidence:
  CLI input rejects (only internal config resolution clamps), and hand-rolled env
  fallbacks silently ignore bad values (e.g. bannermanor `term_width_env`).
- **A NULL-terminated `cstr*` choices array.** Rejected for a `'|'`-delimited
  string: one literal at the call site, and it renders directly in help/errors
  with no array-building or `fmt`.
