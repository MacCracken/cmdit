# `cmdit_completions` scripts complete nothing in the first argument of a tool with no verbs — RESOLVED

**Status:** RESOLVED in cmdit **1.5.1** (2026-11-06). A tool with no verb to offer (none registered, every one hidden,
or every name dropped by the whitelist) now gets a script for flags and files. Run in real bash, zsh and fish.

As proposed below, with these differences:

| Proposed | Shipped | Why |
|---|---|---|
| fish `-r -a 'a b c'` for an enum | `-x -a 'a b c'` | On fish 4.9.3, `-r` lists every file beside the choices; `-x` lists the choices alone |
| bash: `--color <TAB>` | also `--color=<TAB>` and `--color=al<TAB>` | `COMP_WORDBREAKS` splits at the `=`, so a value after it needed the same handling, for files too |
| `cmdit_verb_count(h) == 0`, or every verb hidden | also every name dropped by the whitelist | The same empty list, by another route |

- **Not in this fix:** a tool WITH verbs keeps its 1.5.0 script byte for byte, so `--color <TAB>` on a verb tool still
  offers verbs. The issue covers verb-less tools, and two consumers ship the verb scripts as they are.
- **owl** can now add `cmdit_enum(h, 0, "completions", 0, "bash|zsh|fish", …)` and one call. Note that a registered
  `--completions` shows up in its own completion list, like any other flag.

**Discovered:** 2026-10-31 while adopting cmdit for owl 1.6.0's CLI.
**Severity:** Medium. A wrong answer with no workaround short of writing your own completion script. For a
single-command tool, installing the generated script is worse than having none: it replaces the shell's
default file completion with one that offers nothing for the first argument.
**Affects:** cmdit 1.5.0 (verified; also current `main`, 7671a28): `src/cmdit.cyr` `_cmdit_completions_bash`
(line 2628), `_cmdit_completions_zsh` (2674) and `_cmdit_completions_fish` (2726).

## Summary

All three emitters assume the first non-flag word is a verb. Until a word has been seen in that slot, they
complete from the verb list. A tool that registers no verbs (owl, cat- or ls-shaped tools: flags plus files)
has an empty verb list, so:

- **bash:** `if [[ $seen_verb -eq 0 ]]; then COMPREPLY=( $(compgen -W "$verbs" ...) ); return 0; fi` with
  `verbs=''`. The first file never completes. The second does, because the first file counts as the "verb".
- **zsh:** `if (( ! seen )); then _describe -t commands 'command' verbs` with an empty `verbs` array, so the
  first argument has the same problem. zsh also never completes a flag. Its script carries no flag list,
  only `valueflags` for skipping.
- **fish:** `complete -c owl -f` turns files off globally, and the only `-F` is guarded by
  `not __fish_use_subcommand`. That guard is true until a non-option word exists, so the first argument has
  the same problem. fish completes no flags either.

## Reproduction

A scratch tool with one flag and no verbs, or owl's 1.6.0 flag table, which has 24 flags and no verbs:

```sh
cd "$(mktemp -d)" && touch alpha.txt beta.rs
tool --completions=bash > t.bash      # the tool calls cmdit_completions(h, "bash")
bash -c 'source ./t.bash
  for line in "tool " "tool al" "tool -n al" "tool alpha.txt b"; do
    read -ra COMP_WORDS <<< "$line"; [[ "$line" == *" " ]] && COMP_WORDS+=("")
    COMP_CWORD=$((${#COMP_WORDS[@]}-1)); COMPREPLY=(); _tool_complete
    printf "%-18s -> [%s]\n" "$line" "${COMPREPLY[*]}"
  done'
```

Observed with owl's table:

```
owl                -> []
owl al             -> []
owl -n al          -> []
owl alpha.txt b    -> [beta.rs]
```

Expected: `alpha.txt beta.rs`, `alpha.txt`, `alpha.txt`, `beta.rs`.

## Proposed fix

When the handle has no visible verbs (`cmdit_verb_count(h) == 0`, or every verb hidden), emit the verb-less
shape:

- **bash:** skip the verb walk and fall straight to the file branch. The `--…` flag branch already works.
- **zsh:** `_arguments` over the flag table (`'--color=[when to colour]:when:(auto always never)'` from
  `cmdit_flag_choices_at`, bools as plain `'--plain[…]'`), then `'*:file:_files'`.
- **fish:** no global `-f`. One `complete -c tool -l name -d help` per flag, plus `-r -a 'a b c'` for an enum
  flag's choices.

Choices for enum flags would be welcome in bash too. `--color <TAB>` completing `auto always never` follows
directly from `cmdit_flag_choices_at`.

## Consumer-side workaround

None that is worth having. owl 1.6.0 parses its CLI with cmdit but does not expose `--completions` until this
is fixed. The alternative was an owl-side generator over the same flag table, which would duplicate cmdit.
owl's `src/main.cyr` (CLI section) carries the note. Once cmdit ships the fix, owl adds
`cmdit_enum(h, 0, "completions", 0, "bash|zsh|fish", …)` and one call.
