# `_cmdit_print_esc` passes C1 control pairs to the terminal — RESOLVED

**Status:** RESOLVED in cmdit **1.5.1** (2026-11-06). `C2 80..9F` and any bare `80..9F` now show as one `?` each, and every
well-formed character is kept.

One rule below was sharpened. "`80..9F` that does not follow a lead byte" would mangle `E0 A4 9B`, whose `9B` follows a
continuation byte. So the string is walked a character at a time against Unicode's table of well-formed sequences, which
is what commandress `_sanitize_bytes` does. That also refuses the overlong forms of ESC and CSI (`C0 9B`, `E0 82 9B`), a
surrogate, and anything past U+10FFFF. commandress needs no change beyond the pin.

**Discovered:** 2026-11-06, commandress audit item D-06 ([2026-11-06 audit](https://github.com/MacCracken/commandress/blob/main/docs/audit/2026-11-06-audit.md)).
**Severity:** Low. Argv is rarely an attacker position. It is one wherever a tool's arguments come from a file name or a
script, and this function exists for that case.
**Affects:** cmdit 1.5.0 (verified; also current `main`, 7671a28): `src/cmdit.cyr`, `_cmdit_print_esc` at line 1715, called
by `_cmdit_eprint_tail` and `_cmdit_eprint_quoted`.

## Summary

The comment above `_cmdit_print_esc` says it stops a token putting "terminal escape sequences into whatever reads the
output". It filters C0 and DEL, and passes every byte from 0x80 up so that non-ASCII names stay legible. But C1 controls
are escape sequences too. `C2 9B` is U+009B, which is CSI in one character, and it needs no ESC byte for the filter to catch.

## Reproduction

```sh
cmdrs $'--\xc2\x9b31m' 2>&1 >/dev/null | od -c | head -3
# cmdrs: unknown flag: -- 302 233 3 1 m
```

Run on commandress 1.5.1, which pins cmdit 1.5.0. `C2 9B` reaches stderr unchanged. A terminal that honours C1 in UTF-8
reads it as CSI.

## Consumer and workaround

commandress has no workaround: `cmdit_print_error` writes straight to stderr. Both of its shell adapters pass a fixed argv,
which is why this is not exposed in practice.

## The smallest fix

**Not a plain test for 0x80..0x9F.** Those are also UTF-8 continuation bytes, so that test would mangle real names: `ě` is
`C4 9B`. The byte must be judged by what comes before it:

- `C2` followed by `80..9F` is a C1 control. Show `?` for the pair.
- `80..9F` that does not follow a lead byte is a bare C1 byte. Show `?`.

Each control is replaced and not removed, so a pair cannot reassemble across it, which is the trap in a filter that drops.

commandress `src/color.cyr` `_sanitize_bytes` is a worked example of the first two rules, with an exhaustive test over
813,615 strings in its suite. The limit is the same as there: a terminal in 8-bit mode reads the second byte of real UTF-8 as a
control, and both cannot be kept. UTF-8 is the case that matters.

The output stays one `?` per control, so the change is append-only on the frozen surface.
