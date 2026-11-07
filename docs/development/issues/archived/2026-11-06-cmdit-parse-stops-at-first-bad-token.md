# `cmdit_parse` stops at the first bad token, so a good flag after it is lost — RESOLVED

**Status:** RESOLVED in cmdit **1.5.1** (2026-11-06). `cmdit_continue_on_error(h, on)`, as proposed below: off by default,
the first error stands, an unknown `--x y` leaves `y` a token of its own, and `--bogus --help` still reports `--bogus`.

Beyond the proposal:

- **Dispatch honours it too.** `_cmdit_take_flag` is shared by both parse cores, and a setter one of them ignored would
  fail silently. An unknown verb fills the verb's slot: `cmdit_verb_matched` stays -1, no later word is read as the
  verb, and a global after it still applies.
- **A registered flag keeps its grammar.** `--count abc` consumes `abc`, as the plain parser does. That has one edge:
  `--count --side=right` takes `--side=right` as the value. `cmdit_reject_flag_value` is how a flag opts out.
- **`CMDIT_ERR_NOMEM` still stops the parse.** It is not a fact about a token.

**commandress** can pin 1.5.1, call `cmdit_continue_on_error(cli, 1)` and drop the note above `cli_side_for`.

**Discovered:** 2026-11-06, commandress audit item D-05 ([2026-11-06 audit](https://github.com/MacCracken/commandress/blob/main/docs/audit/2026-11-06-audit.md)).
**Severity:** Low. A tool that must render something on bad input gets a wrong answer, with no workaround short of
re-entering the parser. Tools that exit on a bad flag never notice.
**Affects:** cmdit 1.5.0 (verified; also current `main`, 7671a28): `src/cmdit.cyr`, the unknown-flag path at line 1047
and the `CMDIT_RESULT_ERR` return out of the `cmdit_parse_argv` loop at line 1121.

## Summary

The loop in `cmdit_parse_argv` returns at the first token it cannot use. Flags before that token are set, and flags after
it are never seen. For most tools that is right: they print the error and exit.

commandress cannot exit. It renders a shell prompt, so "never blank the prompt" is its contract, and it reads whatever
was parsed before the error. A flag it understands, after one it does not, is dropped.

## Reproduction

```sh
cmdrs --side=right --bogus   # the right prompt: --side was parsed first
cmdrs --bogus --side=right   # the LEFT prompt: the parse stopped at --bogus
```

Run on commandress 1.5.1, which pins cmdit 1.5.0. No adapter produces that order, which is why it is Low.

## Consumer and workaround

commandress `src/cli.cyr`, `cli_side_for` and the comment above it. It honours the flags parsed before the error, and
documents the rest as this issue. Nothing else can be done there short of parsing argv a second time, which is what cmdit
exists to prevent.

## The smallest addition

One additive setter, off by default, so no existing tool changes:

```cyrius
# Keep parsing past a bad token. cmdit_parse still returns CMDIT_RESULT_ERR, and
# cmdit_error and friends name the FIRST error.
fn cmdit_continue_on_error(h, on): i64
```

The first error stays the one reported, because `_cmdit_fail` would otherwise overwrite it. It needs one decision: an
unknown `--x y` cannot tell whether `y` was its value. Treating `y` as its own token is the only choice that needs no
knowledge of a flag nobody registered, and `--bogus --side=right` works under it.

A help or version token after a bad flag needs the same care: `CMDIT_HELP` should not replace an error already recorded,
or `--bogus --help` would stop reporting `--bogus`.
