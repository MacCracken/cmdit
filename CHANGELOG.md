# Changelog

Format: [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased]

## [1.5.0] - 2026-10-28

**The completion scripts complete file names.** After a verb, bash split a file name at
its spaces and glob-expanded it, and fish completed no file at all. Reported by sit (sit
1.12.2's closeout audit, L11), which prints cmdit's script as is. No API or error-code
change. bash's file line becomes three lines and fish gains one; zsh's script is untouched. **892 → 904 assertions**; 9/9 seeded mutants caught, and the control survives.

### Fixed

- **bash: a file name is one candidate.** The file branch was
  `COMPREPLY=( $(compgen -f -- "$cur") )`. The unquoted substitution word-split and globbed
  every name: with `my file.txt` in the directory, `my<TAB>` offered `my` and `file.txt`,
  and a file named `g*` multiplied into duplicates. It now reads compgen's output a line at a
  time (`while IFS= read -r w; do COMPREPLY+=( "$w" ); done < <(compgen -f -- "$cur")`), after
  `COMPREPLY=()`.
  - **`compopt -o filenames`** comes first, so readline quotes the name it inserts and puts a
    `/` after a directory. It is set in the file branch only. On `complete` it would apply to
    verbs too: checked with a directory named `ps` beside the verb `ps`, `p<TAB>` completed
    `ps/` that way, and completes `ps` in 1.5.0.
  - **bash 3.2 keeps working.** A `read` loop rather than `mapfile`, which bash 3.2 lacks;
    `compopt` (bash 4+) is silenced where missing. There the candidates are still right, but
    readline does not quote the inserted name.
- **fish: files come back after the verb.** The script's global `complete -c <prog> -f` kept
  files out of the verb position, and nothing turned them on again. One line now follows it:
  `complete -c <prog> -n 'not __fish_use_subcommand' -F`. fish documents `-F` as allowing a
  file "even if another applicable `complete` specified `--no-files`". fish is not installed
  on the build host, so this half rests on that documentation and on the emitted line, not
  on a run.
- The `cmdit_completions` doc comment said zsh and fish complete "verbs then files"; fish did
  not until now.

### Verification

Real readline, driven through a pty in an interactive `bash --norc`, with
`demo-app run <prefix><TAB>` and a `demo-app` that prints its arguments:

| typed | 1.4.2 | 1.5.0 |
|---|---|---|
| `my<TAB>` (`my file.txt` exists) | `<my>` | `<my file.txt>` |
| `sub<TAB>` (directory `subdir`) | `<subdir>` | `<subdir/>` |
| `b<TAB>` (file `b\x`) | `<bx>`, a different name | `<b\x>` |

- **A new group, `g115_1_5_0_completion_files`**, pins the emitted bash lines in both script
  layouts (with and without per-verb flags), the fish line in both, and zsh's `_files`.
- **The suite now runs the bash script.** A new helper, `run_bash`, forks `/bin/bash -s` and
  feeds it the generated script plus a fixed driver over a pipe; no argv or command string is
  built. In a fresh `mktemp -d`, a name with a space, a name with a `*`, a leading space and a
  backslash each come back as one exact candidate, and the verb position still offers verbs.
  Without `/bin/bash` the run is skipped with a note.
- **Mutation round.** Reverting to 1.4.2's line, quoting the substitution whole, dropping
  `compopt`, `IFS=`, `read -r` or the `COMPREPLY` reset, and, for fish, dropping the `-F` line,
  turning it into `-f` or dropping its `not` are each caught. The bash run alone catches 5 of
  the 6 bash mutants. `compopt` acts only inside a real readline completion, so the text
  check catches that one.
- The demo transcript differs from 1.4.2's in exactly those four script lines; smoke and verbs
  print the same bytes. The suite passes against `dist/cmdit.cyr`, and fuzz, both poison gates
  and `cyrius audit` pass.

### Benchmarks

Interleaved 1.4.2 against 1.5.0, median of 15, at load average ~0.4: `completions_devnull`
13.97 → 14.32 µs (+2.5%), from the longer bash and fish scripts it writes. The other 12
benches are within ±3%, the code-layout band.

### Docs

- The roadmap's 1.5.0 item is shipped. The 1.5.x section stays as the intake, with nothing
  open.

## [1.4.2] - 2026-10-28

**A command line past 2 MiB is read whole on cyrius 6.7.12 and later.** The 1.3.2 check
(audit SEC-7) refused any command line of 2 MiB or more, because stdlib `args_init` cut it
there with no error. cyrius 6.7.12 resolved that issue, so the check now refuses only the one
length that is still ambiguous. Consumers on older pins fail closed exactly as before. No API,
error-code or output change. **887 → 892 assertions**; 6/6 seeded mutants caught, and the
no-op control survives.

### Changed

- **`_cmdit_argv_whole` refuses `len == 2 MiB`, not `len >= 2 MiB`.** A pre-6.7.12 `args_init`
  reads `/proc/self/cmdline` once into a 2,097,152-byte buffer, so it can never return more.
  From 6.7.12 it reads to EOF. So any read longer than 2 MiB is whole under either stdlib, and
  a read of exactly 2 MiB is the only one that may be a cut. That length stays
  `CMDIT_ERR_ARGV`, because `dist/cmdit.cyr` compiles on pins as old as 6.6.2. On 6.7.12+, a
  command line of exactly 2,097,152 bytes is the one false refusal.

  A real command line can now reach 6 MiB, Linux's ARG_MAX ceiling, after an unprivileged
  `ulimit -s` raise. The materialized array's slot cap (2^60 − 1) is far above any such argc.

### Verification

Real command lines under `ulimit -s 65536`, with a flag as the last argument (it shows up in
the error only if the tail was read). Sizes are exact `/proc/self/cmdline` bytes; the
arithmetic was checked by having `cat` read its own `/proc/self/cmdline`.

| command line | 1.4.0, 6.7.9 stdlib | 1.4.2, 6.7.9 stdlib | 1.4.2, 6.7.18 stdlib |
|---|---|---|---|
| 1,900,000 B | whole | whole | whole |
| 2,097,151 B | whole | whole | whole |
| 2,097,152 B | refused | refused | refused |
| 2,097,153 B | refused | refused | **whole** |
| 2,500,000 B | refused | refused | **whole** |
| 4,000,000 B | refused | refused | **whole** |

The middle column is the older-pin consumer: identical to 1.4.0 at every size. The 1.3.2 audit
scenario (30 arguments of 100,000 bytes, then `--loud`) gave exit 0, `loud = 1` and all 30
positionals on 6.7.18. On 6.7.9 it is still `CMDIT_ERR_ARGV`.

- **Tests.** A new group, `g114_1_4_2_argv_past_2_mib`: exactly 2 MiB is refused, and one byte
  past it, a 3 MB read and 6 MiB are whole. A long read never stands in for a missing `argv[0]`.
- **Mutation round.** Reverting to `>=`, dropping the check, moving it one byte either way,
  inverting it to `>`, and weakening `n <= 0` to `n < 0` are each caught by the suite.
- The suite, fuzz, both poison gates and `cyrius audit` pass. The 51-invocation demo transcript
  is byte-identical to 1.4.1's, and the binaries are the same size. No benchmark covers the
  check, which runs once per process.

### Docs

- The roadmap's open items are now planned as **1.5.x**. The sit completions item (bash splits
  file names, and fish completes none) is 1.5.0. The cyrius `args_init` watch item is gone, now
  that it is resolved and acted on. The v2 backlog and out-of-scope sections are unchanged.

## [1.4.1] - 2026-10-28

**Maintenance: toolchain `6.7.9` → `6.7.18` and a full vendored-stdlib refresh.**
`src/cmdit.cyr` is untouched, and every byte cmdit prints is the same: 51 invocations of
smoke, verbs and completions (13,792 bytes of stdout, stderr and exit codes) are identical
between the 6.7.9 and 6.7.18 builds. **887/887** assertions, fuzz, both poison gates, and
`cyrius audit` is clean on every stage.

### Changed

- **Toolchain pin `6.7.9` → `6.7.18`.** CI reads it from `cyrius.cyml`, so no workflow edit.
- **`cyrius lib sync --full --relock`**, run before the first build under the new pin,
  re-vendored `lib/`: 113 files, byte-identical to `6.7.18/lib`. 66 were modified; none are new
  or removed. `cyrius deps --verify` checks all 113 against `cyrius.lock`. cmdit has no
  `[deps.X]`, so the stdlib is its only vendored library.
- **`dist/` regenerated.** Only the `# Version:` line changed: `distlib --check` already found
  the bundle current under 6.7.18. Consumer-side check: the suite and smoke were built against
  `dist/cmdit.cyr` instead of `src/`. All 887 assertions pass, and smoke is the same size
  either way.

### Stdlib delta on cmdit's call surface

cmdit calls six stdlib functions directly (`alloc`, `strlen`, `args_init`, `argc`, `argv`,
`getenv`), which reach 83 across all targets. On Linux x86_64:

- **`args_init`** (stdlib 6.7.12) reads `/proc/self/cmdline` to EOF, growing a heap buffer from
  2 MiB, and on failure returns a negative errno and publishes nothing. This resolves the issue
  cmdit filed from its 1.3.2 audit (SEC-7), now archived in cyrius as resolved. cmdit never read
  the return value; a failed read still leaves `argc()` at 0, which is still `CMDIT_ERR_ARGV`.
  - **cmdit's own 2 MiB check stays.** `_cmdit_argv_whole` still refuses a command line of
    2 MiB or more, even one the stdlib now reads whole. On real command lines under
    `ulimit -s 65536`, 1.9 MB parses and 2.5 MB and 4.0 MB are `CMDIT_ERR_ARGV`, on both
    builds. `dist/cmdit.cyr` compiles on pins as old as 6.6.2, and below 6.7.12 the consumer's
    own `args_init` still truncates silently, so the check is what keeps those consumers failing
    closed. 1.4.2 narrows it to the one ambiguous length.
- **`getenv`** and its loader `_env_load`: `x = x + 1` → `x += 1` rewrites only.
- `alloc`, `strlen`, `argc`, `argv`, `sys_open` and `sys_close` are body-identical.

### Binary size: +4 KB, all but 24 bytes unreachable stdlib

`cmdit-smoke` grew 172,256 → 176,376 B, and each of the six artifacts grew by the same 4,120 B.
With `CYRIUS_DCE=1` the live code went 53,472 → 53,496 B (+24 B). The rest is unreachable
stdlib (513 → 526 unreachable fns), which the opt-in DCE pass drops.

### Benchmarks

The same source built under each toolchain, run interleaved, median of 15, at load average
~0.6 (ns):

| bench | 6.7.9 | 6.7.18 | change |
|---|---|---|---|
| `cmdit_new_floor` | 729 | 752 | +3.2% (ranges overlap: 691–1006 / 709–921) |
| `setup_baseline` | 724 | 723 | −0.1% |
| `parse_argv_mixed` | 1075 | 1085 | +0.9% |
| `get_enum_rewalk` | 96 | 95 | −1.0% |
| `dispatch_before_verb` | 162 | 156 | −3.7% |
| `dispatch_after_verb` | 172 | 167 | −2.9% |
| `repeat_accumulate_32` | 1760 | 1770 | +0.6% |
| `verb_new_floor` | 780 | 772 | −1.0% |
| `flag_find_long` | 46 | 46 | 0 |
| `flag_find_long_36` | 147 | 151 | +2.7% |
| `help_fd_devnull` | 2825 | 2781 | −1.6% |
| `print_error_devnull` | 747 | 758 | +1.5% |
| `completions_devnull` | 14346 | 13941 | −2.8% |

All 13 are within ±4%. That is the code-layout band 1.4.0 measured for unchanged code
(`get_enum_rewalk`, 95–101 ns from padding alone), and the 4 KB of new stdlib shifts the
layout.

## [1.4.0] - 2026-10-14

**Refactor and optimization sweep: no new API, byte-identical output, and faster.** Every
change is internal. The public API and error codes are unchanged, and every byte cmdit
prints is the same. A fixture covering every renderer, 20 error lines and all three
completion scripts over 7 handle shapes (20,724 bytes) was diffed against 1.3.7 after
every step. A parse is about 2× faster, a verb tool's setup 4×, and `--help`, an error line
and a completion script 6–12×. **786 → 887 assertions**; 27 of 31 seeded reverts are caught, and
the 4 survivors are unobservable.

### Refactor

- **Named field offsets.** The 255 literal offsets into the handle, a flag entry and a verb
  row (`h + 104`, `ep + 48`, `row + 16`) are now private `CmditOffset` names
  (`_CMDIT_H_VERBS`, `_CMDIT_E_ATTRS`, `_CMDIT_V_CANON`). A literal is how 1.3.5's verb
  renderer kept reading +32 after the row grew. **Proof:** the names were added in one step and
  swapped in the next, and the seven binaries (smoke, verbs, completions, tests, fuzz, bench
  and the fixture) came out byte-identical. An offset-0 field stays bare (`load64(h)`),
  because `+ 0` costs an instruction.
- **One token-to-flag path.** The parse core resolved and applied a flag at two sites (long and
  short), and the dispatcher at a third, and they had drifted: 1.3.2 fixed a BUNDLED/UNKNOWN
  divergence between them, and 1.3.6 had to add its flag-shaped-value refusal three times. Both
  cores now call one `_cmdit_take_flag`. After the verb, the dispatcher used to look each flag
  up twice (once to spot `--help`, once to apply it); it now looks it up once.
- **Shared guards and helpers.** The verb setters and getters resolve an id through
  `cmdit_verb_canonical_at` instead of four copies of the same guard ladder. One
  `_cmdit_zero` replaces the two table-zeroing loops.
- **The test suite is 113 group functions**, not one ~2,300-line `main` whose fixture names all
  had to be unique (a collision was a build error, several times in 1.3.x). The split was
  checked by identical suite output before and after, and by an identical statement sequence.

### Optimization

Interleaved 1.3.7 against 1.4.0, median of 11 runs each, at load average ~1.9 (one unrelated
single-core compile was running):

| bench | 1.3.7 | 1.4.0 | change | from |
|---|---|---|---|---|
| `cmdit_new_floor` | 2.03 µs | 0.76 µs | −63% | the entry table that grows |
| `setup_baseline` | 1.99 µs | 0.75 µs | −62% | the same |
| `parse_argv_mixed` | 2.44 µs | 1.16 µs | −53% | the same, and the lookup |
| `repeat_accumulate_32` | 3.28 µs | 1.84 µs | −44% | the same |
| `verb_new_floor` | 3.29 µs | 0.81 µs | −75% | both tables growing |
| `dispatch_before_verb` | 172 ns | 167 ns | −3% | |
| `dispatch_after_verb` | 242 ns | 177 ns | −27% | one lookup per flag after the verb, not two |
| `flag_find_long` | 78 ns | 47 ns | −40% | the lookup |
| `flag_find_long_36` | 396 ns | 152 ns | −62% | the same, over attn11's 36 names |
| `help_fd_devnull` | 33.8 µs | 2.9 µs | −91% | buffered output |
| `print_error_devnull` | 4.75 µs | 0.77 µs | −84% | the same |
| `completions_devnull` | 161 µs | 14.9 µs | −91% | the same |
| `get_enum_rewalk` | 92 ns | 98 ns | **+6.5%** | code layout, below |

**`get_enum_rewalk` is the one regression, and its code did not change.** The field-offset
swap compiles byte-identically, and the functions on its path are the same. Inserting unused
padding functions ahead of `_cmdit_enum_find` moves the same source between 95 and 101 ns.
So the 6 ns is where the code landed, not what it does. It is recorded here, not tuned away
with padding that the next edit would undo. On its own, the one token-to-flag path cost a
32-flag parse +3.7% (one more call level per flag token); the growing entry table repays that
many times over.

- **Buffered output.** Every renderer used to make one write(2) per fragment: 93 for one
  `--help`, 13 for one error line, and one per byte of quoted help in a completion script. Each
  public renderer now opens an output session, which flushes before it returns, so one call is
  one write. Its output still lands before anything the caller writes next. An error line now
  reaches a pipe in one write, so another process's stderr cannot split it. The buffer is
  allocated on first use: a global 4 KB array would have added 32 KB to every consumer's
  binary. The final write also retries a short write and EINTR, which the ignored write results
  never did.
- **Tables that grow.** The flag table starts at 16 entries and the verb table at 16 rows,
  each doubling to its unchanged cap of 64 / 128. `cmdit_new` allocated and zeroed all 64
  entries (7,680 B), and the first `cmdit_verb` all 128 rows (6,144 B), for tools with a
  handful. `err_entry`, the one entry pointer a handle keeps, moves with its table, so
  `cmdit_err_flag` stays right when flags are registered after a failed parse.
  - Memory: `cmdit_new` allocates 3,136 B (was 8,896), and the first verb 768 B (was 6,144). A
    tool that fills all 64 flags allocates 13,440 B of entry tables over the moves (was 7,680),
    since the bump allocator never frees the old ones.
- **Long-name lookup.** The parser's `--name` scan compares a name's first byte before calling
  the full compare, and walks a pointer instead of calling `_cmdit_entry` per row. Over attn11's
  36 real flag names (38 entries with `--help` and `--version`), a lookup makes 3 calls
  instead of 76.

### Tests

- **+101 assertions (786 → 887):** output sessions (the order against the caller's writes,
  nesting on one stream and across two, a write to another stream mid-session, pieces up to and
  past the buffer, every renderer closing its session), both tables moving (every modifier,
  mark and attached handle surviving, `err_entry` after a failed parse, the caps), and the
  lookup prefilter (shared first bytes, an empty span).
- **Fuzz:** both handles are padded past 16 flags / 16 verb rows, so the tables grow inside
  every iteration and I9 checks lookups over a moved table.
- **Benchmarks 9 → 13:** `flag_find_long_36` over attn11's real names, and `help_fd_devnull`,
  `print_error_devnull` and `completions_devnull` for the output paths.
- **Mutation, 27/31.** The first round left three real gaps, all closed:
  - the last verb row before a move;
  - a write to another stream mid-session;
  - a piece larger than the buffer being copied into it.

  It also showed one clause in the session close to be redundant, so that clause was removed.
  The four survivors are unobservable:
  - the byte-tail loop of `_cmdit_zero`, since every size is a multiple of 8;
  - the retry of a short write, which needs an interrupted writer;
  - the zeroing of each grown table's tail, because the stdlib's `alloc` already returns
    zeroed memory.
- The bundle passes 887/887 on a 6.6.2 compiler.

### Docs

- **Roadmap:** now forward-facing only. Nothing is scheduled, so it is an open intake for
  consumer requests and issues, plus the watched cyrius issue, adoption, the v2 backlog and
  out-of-scope.
- **Architecture notes**
  [001](docs/architecture/001-tables-move-when-they-grow.md) (the tables move when they
  grow) and [002](docs/architecture/002-every-write-goes-through-an-output-session.md) (every
  write goes through an output session).
- **Adoption notes:** kriya's and ark's are updated for 1.3.x and 1.4.0.

## [1.3.7] - 2026-10-14

**Flag-table introspection, and each verb's own flags in the completion scripts.** This is the
last pinned 1.3.x item. Everything is append-only: 15 functions (13 flag accessors and
lookups, 2 for a verb's flag handle). The verb row grows 40 → 48 B. A handle that uses none of
it produces byte-identical output, completion scripts included. One fix: fish completion
quoting of a help text containing a backslash. **694 → 786 assertions**; 38 of 39 seeded
reverts are caught, and the survivor is unobservable.

### Added — flag-table introspection (kriya)

kriya renders its own OPTIONS page and `--help=json`, and its pre-parse expander splits `-n5`
only after asking whether `-n` takes a value. All three read the stdlib flags table's memory
layout directly (`kriya_short_type`, `kriya_long_type`, `help.cyr`'s `_json_options`), which
kriya's own comment calls a coupling. These accessors answer the same questions through a
supported surface, reading the table the parser reads:

| function | returns |
|---|---|
| `cmdit_flag_count(h)` | registered flags; `cmdit_new`'s auto `--help` and `--version` are rows 0 and 1 |
| `cmdit_flag_short_at` / `_long_at` / `_long_alias_at` / `_help_at` / `_metavar_at` / `_choices_at` `(h, i)` | the registered field, or 0 |
| `cmdit_flag_type_at(h, i)` | `CMDIT_BOOL` / `INT` / `STR`, or -1 for a bad index; an enum or repeat flag is `STR` |
| `cmdit_flag_default_at(h, i)` | the registered default (an int or a cstr), not the parsed value |
| `cmdit_flag_is_required` / `cmdit_flag_is_repeat` `(h, i)` | 1 or 0 |
| `cmdit_flag_find_short(h, ch)` / `cmdit_flag_find_long(h, name, len)` | the index the parser resolves `-x` / `--name` to, or -1 |

- **The set is what `cmdit_help_flags` prints**, so a custom renderer can match it. Env names,
  ranges and the 1.3.6 bits are not exposed, because nothing asked for them. The two
  predicates stand in for a raw attribute accessor because ADR-0003 keeps the `CMDIT_ATTR_*`
  bitfield internal.
- **`cmdit_flag_find_long` takes a span.** kriya passes the bytes of `--name=value` up to the
  `=`, with no copy. It is the parser's own lookup, second long names included, and the first
  registration wins as it does in the parser. A span holding a NUL matches nothing: the shared
  compare checks `len` bytes before it checks the registered name's end, so it would read past
  a shorter name.

### Added — `cmdit_verb_flags(h, verb_id, vh)` / `cmdit_verb_flags_at(h, i)` (ark)

ark's `completions/ark.bash`, `.zsh` and `.fish` hand-write each verb's flags: `install`
`--force --group`, `remove|uninstall` `--purge`, `search` `--source`, `list|ls` `--marketplace
--system --flutter`. A verb tool already builds one handle per verb to re-parse
`cmdit_verb_argv`. `cmdit_verb_flags` attaches that handle to its verb, and the completion
scripts read the verb's flags from it. **Dispatch does not change**: cmdit never parses with
the attached handle. `cmdit_verb_flags_at` reads it back, so a tool can build its verb handles
up front and take the matched one after dispatch with `cmdit_verb_flags_at(h,
cmdit_verb_matched(h))`. An alias row resolves to its verb, and `vh` = 0 detaches.

Once a verb with an attached handle has been typed:

- **bash** adds the verb's long flags to the global list (`install) flags="$flags --force
  --group" ;;`). Under `cmdit_stop_at_positional`, where globals are not applied after the
  verb, the verb's list replaces the global one and keeps its own `--help` / `--version`. The
  walk that finds the verb now runs before the flag branch, since the flag list depends on it.
- **zsh** offers the verb handle's whole list through `compadd` for a word starting with `-`,
  and files otherwise.
- **fish** gets one `complete` line per flag, conditioned on `__fish_seen_subcommand_from
  <verb> <aliases>`: short and long name together, `-r` for a value flag, `-d` with the help.

An alias selects its verb's arm, a hidden verb gets none, and every name still passes the
`[A-Za-z0-9_.-]` whitelist, so an unsafe verb or flag name is dropped as before. One level
only: verbs registered on the attached handle are not completed. zsh and fish still complete
no *global* flags.

**Byte-identical without it.** When no visible verb has a handle attached, all three scripts
are byte for byte the 1.3.6 ones. A fixture's three scripts were diffed against 1.3.6's
output, and the suite pins the 1.3.6 layout. A tool that regenerates checked-in completions
sees no churn.

### Fixed — fish completion quoting

fish reads two escapes inside single quotes, `\'` and `\\`; bash and zsh read none. The fish
emitter used the POSIX quoter for a verb's `-d` help, so a help text ending in a backslash
(`-d 'Install C:\'`) turned the closing quote into a literal one. From there the quoting was
off by one for the rest of the script, and the next verb's help text was read as fish code.
A backslash is now doubled. The text is the program's own help, not argv, so it took a tool
author writing such a string; verb names were never affected, being whitelisted. Help without
a backslash is emitted as before. fish is not installed on the build host: this follows fish's
documented quoting rules, and the suite pins the generated bytes.

### Tests

- **+92 assertions (694 → 786):**
  - every accessor on each flag shape, plus bad indices and handles;
  - the lookups: second long names, `=` spans, prefixes, a NUL in the span, and the
    expander's own question;
  - attaching a verb handle, reading it back through an alias row, detaching it, and a
    dispatch round trip;
  - the exact per-verb lines in each shell, stop mode, a verb with only the auto flags, and
    the unchanged scripts without any attachment.
- **Fuzz invariant I9.** For every flag-shaped argv token, `cmdit_flag_find_long` / `_short`
  must agree with a brute-force walk of the accessors, and the accessors must answer their
  sentinels past the table. The pool grows 52 → 54 with near misses (`--nam`, `--headers=y`).
  Three seeded lookup bugs are each caught by I9 alone.
- **Real shells.** The generated bash script, sourced with simulated `COMP_WORDS`, gives the
  right completions in 14 cases:
  - globals before the verb, the verb's flags after it, an alias;
  - a hidden verb, and a verb with no handle;
  - a value flag that swallows the would-be verb;
  - stop mode.

  zsh, with its completion builtins stubbed, picks the right arm in 8 cases. `bash -n` and
  `zsh -n` pass on the demo program's scripts.
- **Mutation, 38/39.** The first round's one real survivor (a flag with only a short letter
  losing its fish line) was closed with a fixture flag. The remaining survivor is the explicit
  zero of the new row field, which the table's zeroing at allocation makes unobservable.
- The bundle passes 786/786 on a 6.6.2 compiler.

### Performance — measured on an idle host

Interleaved 1.3.6 against 1.3.7, 11 runs each, load average ~1.3. Every bench is within ±4%
except `verb_new_floor`, **3.13 → 3.36 µs (+7%)**. The verb table grows 5,120 → 6,144 B, and
its share (`verb_new_floor − cmdit_new_floor`) went 1.11 → 1.31 µs, the 20% the table grew.
A verb tool pays it once per process. Dispatch is unchanged (176 → 177 ns before the verb,
246 → 248 after). The new `flag_find_long` bench reads 80 ns. `docs/benchmarks.md` now holds
this capture in place of 1.3.2's.

## [1.3.6] - 2026-10-14

**Four small modifiers, each asked for by one tool in the adoption survey.** This is the
fourth pinned 1.3.x item. Everything is append-only: four functions, one error code
(`CMDIT_ERR_REPEATED = 14`) and three attribute bits. The flag entry grows 112 → 120 B for the
alias name. **629 → 694 assertions**, and 16/16 seeded reverts are caught.

| modifier | for | behaviour |
|---|---|---|
| `cmdit_once(h, idx)` | cyim | a second command-line occurrence fails with `CMDIT_ERR_REPEATED`, naming the flag and the repeated token. Last-wins stays the default |
| `cmdit_enum_nocase(h, idx)` | phylax | enum choices match ASCII case-insensitively. `cmdit_get_enum` gives the choice's index, `cmdit_get_str` the text as typed |
| `cmdit_long_alias(h, idx, name)` | ark | a second long name, matched by both parse cores, shown in help and offered by bash completion |
| `cmdit_reject_flag_value(h, idx)` | hapi | a flag-shaped **separate** value is `CMDIT_ERR_MISSING_VALUE` instead of being taken |

### Each one, grounded in the tool that asked

- **cyim's** smoke tests refuse a single flag given twice, with exit 2: `--context=1
  --context=2`, `--wc --wc`, `--regex=ere --regex=ere`, and four more (cases 26, 28, 30, 31,
  41, 66, 87d). `cmdit_print_error` says `--context: may be given only once`. Its two
  *cross-flag* cases (27, 29, `--expect` against `--expect-not`) are mutual exclusion, which
  stays on the v2 backlog. An env fallback never counts as an occurrence; it only applies to
  a flag the command line did not set.
- **phylax** lowercased `--severity-threshold` itself before matching. With the modifier,
  `HiGh` selects `high`, a prefix is still not a match, and a non-choice is still `BAD_ENUM`
  with its value.
- **ark's** `list` accepts `--marketplace|--market` and `--system|--apt`. The alias is matched
  exactly like the first name, never as a prefix, including as a global after the verb. Help
  renders `--marketplace, --market`. bash completion lists both names, and skips the value
  after either one.
- **hapi** audit **F-019**: `--root --dry-run pkg` took `--dry-run` as the root and, on a run
  the user asked to be a no-op, created `$HOME/--dry-run/` at exit 0. Under the modifier, a
  separate value starting with `-` is refused, with three exceptions that keep the guard from
  refusing real input:
  - a lone `-` (stdin);
  - a number such as `-5` for an int flag;
  - an attached `--root=--dry-run`, which the `=` makes deliberate.

  It applies to the long and short forms and to globals in the dispatcher. Without the
  modifier, cmdit still takes the next token, as getopt does.

### Tests

- **+65 assertions (629 → 694):** each modifier through both parse cores, its error lines,
  its boundaries (the exceptions above, prefixes, env interplay, repeat flags, the unchanged
  defaults), and bad handles and indices.
- **The fuzz harness's single-command handle now carries all four modifiers**, with pool
  tokens that exercise them (`--hdr`, `--color=NEVER`, a second `--count`). I2, I3 and I7 now
  cover `CMDIT_ERR_REPEATED`.
- **Mutation-proven, 16/16:**
  - the once check removed, or firing on the first occurrence;
  - case folding removed, or wrong, in each of its two callers;
  - alias lookup, help row, bash flag word and value-skip word removed;
  - the refusal removed at each of its three sites;
  - a lone `-` or `-5` refused;
  - the new message.

  One first-round survivor (the alias missing from bash's flag list) exposed an ambiguous
  assertion, which now pins the exact line.
- The bundle passes 694/694 on a 6.6.2 compiler.

### Performance — not cleanly re-measured

The only hot-path change is the flag entry growing 112 → 120 B, so `cmdit_new` zeroes 7,680 B
instead of 7,168 (word-at-a-time since 1.3.0). The modifier checks are a bit test per applied
flag. The interleaved 1.3.5-vs-1.3.6 run was taken while the host was saturated by unrelated
compiles (load average ~17). Both versions ran ~2.5× their normal times with wide ranges, so
it shows no regression beyond ±5% but measures nothing finer. Re-capture on an idle host
before quoting 1.3.6 numbers; `docs/benchmarks.md` still holds the 1.3.2 capture.

## [1.3.5] - 2026-10-14

**Help on a chosen stream, and hidden verbs.** This is the third pinned 1.3.x item. Five
append-only functions are added; existing output is byte-identical. **596 → 629 assertions**,
and 12/12 seeded reverts are caught.

### Added — `cmdit_help_fd` / `cmdit_help_flags_fd` / `cmdit_verbs_help_fd(h, fd)`

The adoption survey found 14 of the 18 candidate tools printing `--help` on **stdout**. That
is the GNU convention: requested help goes to stdout with exit 0, and a usage error goes to
stderr. The examples are takumi's and hadara's `println` usage and hapi's, phylax's and
agnova's help screens. cmdit's renderers only wrote stderr, so a tool that wanted generated
help had to give up its stream. The three `_fd` forms take the stream:

```
if (r == CMDIT_HELP)       { cmdit_help_fd(h, 1); return CMDIT_EXIT_OK; }      # --help: stdout
if (r == CMDIT_RESULT_ERR) { cmdit_print_error(h); cmdit_help(h); return CMDIT_EXIT_USAGE; }
```

`cmdit_help`, `cmdit_help_flags` and `cmdit_verbs_help` are now the `fd = 2` forms, with
**output byte-identical** to 1.3.4; the suite pins each pair equal. The stream is a call
argument, not a handle setting, so `cmdit_print_error`'s appended command list can never
land on stdout. anuenue's deliberate stderr help is the default and stays as it was.

### Added — `cmdit_verb_hidden(h, verb_id)` / `cmdit_verb_is_hidden(h, i)`

A hidden verb still **dispatches** (its id, its aliases and its trailing mark all work). It
is left out of:

- `cmdit_help` and `cmdit_verbs_help`;
- the command list `cmdit_print_error` shows for an unknown verb;
- all three completion scripts.

Its aliases are hidden with it, and hiding through an alias row hides the verb. This is for
maintenance and diagnostic commands a tool keeps out of its advertised surface, such as hoosh
`bench`, which dispatches but appears in no help. `cmdit_verb_is_hidden` lets a tool that
builds its own listing from the 1.2.0 introspection accessors skip them too. When every verb
is hidden, help prints no empty `Commands:` block.

The verb row grows from 32 to 40 B for the flag at `+32`. Verb-row size is an internal
constant (ADR-0003), and 128 rows now take 5,120 B.

### Fixed — the verb list's content was never tested

Every existing test of `cmdit_verbs_help` checked its return code only. That was exposed when
1.3.5's own first draft read the new hidden flag at `+32` before the row had grown. On
32-byte rows that is the **next** row's name pointer, so every verb but the last was treated
as hidden, and 596 assertions still passed. The suite now pins the exact listing. The
stride-back-to-32 mutant crashes it: the last row's write lands past the table.

### Performance

- **Verb table construction:** handle + first verb **+0.2 µs (2.85 → 3.04 µs)**, from
  zeroing a 25% larger verb table.
- **Dispatch:** +6 to +10 ns per call (`dispatch_before_verb` 178 → 184 ns,
  `dispatch_after_verb` 244 → 254 ns, medians of 11 interleaved runs). It is **not** the
  row stride: 1.3.4 rebuilt with only the stride changed measures dispatch unchanged. Nothing
  in the dispatch path changed either. What tracks it is where the dispatcher lands in
  memory. Moving the two new functions after it, beside the other verb accessors, recovered
  about half (dispatch-after 262 → 254 ns). The rest is attributed to code layout and
  recorded rather than chased.
- Every other bench is within ±2%.

### Tests

- **+33 assertions (596 → 629):**
  - the verb list's exact content;
  - each `_fd` renderer on stdout byte-identical to its stderr form, and `fd = 2` identical
    to the legacy call;
  - hidden verbs dispatching (directly and through an alias), and absent from
    `cmdit_verbs_help`, `cmdit_help`, the unknown-command list, and the bash, zsh and fish
    scripts;
  - hiding through an alias row, the trailing mark on a hidden verb, bounds and a 0 handle;
  - no empty `Commands:` block when every verb is hidden.
- **The fuzz harness's verb handle now has a hidden verb**, so dispatch is fuzzed with one
  present.
- **Mutation-proven, 12/12:**
  - each `_fd` form writing to stderr anyway (rows, verb list, short-flag byte);
  - hidden ignored by the listing, bash, zsh or fish;
  - the alias-row resolution in the setter and the accessor;
  - the bounds check;
  - the empty-block guard;
  - the 32-byte stride.
- The bundle passes 629/629 on a 6.6.2 compiler.

## [1.3.4] - 2026-10-14

**POSIX stop-at-first-positional: `cmdit_stop_at_positional(h, on)`.** This is the second
pinned 1.3.x item, and shakti's last adoption blocker. kriya's `xargs` / `env` need it too.
It is one append-only setter (ctx grows 184 → 192 B). **562 → 596 assertions**, 7/7 seeded
reverts caught, and benches within ±2.5% of 1.3.3.

### Added — `cmdit_stop_at_positional(h, on)`

With `on` non-zero, option recognition stops at the first positional: a token that does not
start with `-`, **a lone `-` included**. That token and every later one are positionals,
whatever they look like, so flags, `--`, `--help` and bundled-looking `-la` after the boundary
are all data. A `--` before the boundary still ends the options and is consumed. Value flags
before the boundary still take their value, so `shakti -u bob ls -la` puts the boundary at
`ls`. That is the property kriya's own notes call "the whole difficulty" of a hand-written
version, and the parse loop gets it for free because it already knows each flag's arity. An
unknown flag before the boundary is still an error, as GNU makes it (`xargs -5 echo`).

On a handle with verbs, the verb **is** the first positional. Globals are applied only before
it, and a global flag after it goes to the verb's scope with everything else (by default
cmdit applies globals before or after the verb). The 1.3.2 help hand-off still works under
the mode, so `tool add --help` is not blocked by a required global.

This matches stdlib `flags_set_stop_at_positional`, with one deliberate difference: there a
lone `-` is a positional but does not stop recognition. POSIX says it is an operand, and
kriya's `kriya_argv_option_end` treats it as one, so in cmdit it stops recognition.

### Why not the per-verb trailing mechanism

The roadmap pinned this as "reusing the per-verb trailing-verbatim mechanism". It cannot, and
the reason is a real bug in kriya's history. Since 1.3.2, `cmdit_verb_trailing_after`
**consumes** a `--` that sits exactly at its threshold (kubectl-style, so `exec c1 -- ls`
runs `ls`). POSIX stop must do the opposite: after the first operand, `--` belongs to the
command. kriya records what eating it did: `ls -1 | xargs rm --` handed `rm` an argument list
beginning `-r` and **recursively deleted a directory the `--` was written to protect**. So the
mode is its own flag (ctx `+184`), checked as each positional is stored, and a test pins
`xargs rm --` keeping its `--`. `cmdit_verb_trailing_after` is unchanged, and remains the tool
for a verb whose first `n` positionals are its own (`exec <container> <cmd>...`).

### Tests

- **+34 assertions (562 → 596):**
  - shakti's grammar end to end, including its pinned case (`-x` after the command is data);
  - kriya's `xargs rm --`;
  - a leading `--`, a lone `-`, an unknown flag before the boundary, `ls -la`;
  - `--help` after the boundary as data and before it as help;
  - off by default, any non-zero `on` enabling it, and a 0 handle;
  - on a verb handle: globals only before the verb, the remainder carrying the after-verb
    flags, and `add --help` past a required global.
- **Fuzz invariant I8:** under the mode, a clean parse's positionals are a contiguous suffix
  of argv, pointer for pointer. A dispatch's remainder is exactly argv from the verb on.
- **Mutation-proven, 7/7:**
  - the stop removed;
  - a lone `-` not stopping;
  - the setter storing `on` raw (caught after a test for `on = 7` was added);
  - globals applied after the verb;
  - the help hand-off hidden by the mode;
  - the null guard;
  - a `--` at the boundary consumed (kriya's bug).
- The bundle passes 596/596 on a 6.6.2 compiler.

## [1.3.3] - 2026-10-14

**The offending token: `cmdit_err_token` / `cmdit_err_value`, and error lines that say what
the user typed.** This is the first of the roadmap's pinned 1.3.x items, and one of shakti's
two adoption blockers. Two append-only accessors are added, and `cmdit_print_error`'s output
changes (decided for this release). **503 → 562 assertions**, and 18/18 seeded reverts of the
new code are caught.

### Added — `cmdit_err_token(h)` and `cmdit_err_value(h)`

The 2026-10-14 adoption survey found seven tools that print the offending text and re-scan
argv to get it. The text they print is one of two things:

- **the token as typed:**
  - shakti `unknown option: <tok>`, which its integration test asserts;
  - attn11 `unknown argument X`, owl `<tok>: unknown option`, chakshu `unknown flag: <flag>`;
  - the unknown command in hapi, phylax and agnova;
- **the rejected value alone:** chakshu's `--sort: unknown key 'zzz'`, `--theme` and `--color`.

So there are two accessors, both returning interior pointers into argv (no copy), or 0:

- **`cmdit_err_token(h)`**, the argv token behind the error, matching stdlib
  `flags_error_arg`. It covers:
  - the unknown flag (`--bogus`, `--bogus=1`), the bundled short (`-vx`) and the unknown verb;
  - the flag whose value was missing or rejected (`--count` in `--count abc`, or `--count=abc`
    as typed);
  - a positional that could not be stored;
  - the first positional past `cmdit_require_positionals_max` / `_exact`.

  It is 0 for REQUIRED_MISSING, MISSING_POSITIONAL, ARGV and an allocation NOMEM. The
  unknown-verb token has lived at ctx `+152` since 0.3.0; that slot is now every error's.
- **`cmdit_err_value(h)`**, the rejected value, for BAD_INT, BAD_ENUM, OUT_OF_RANGE and
  UNEXPECTED_VALUE only: `abc` in `--count abc` or `--count=abc`, and `0` in `--loud=0`. It
  lives at a new ctx `+176` (ctx 176 → 184 B, append-only).

Both are reset at the start of every parse and dispatch, with the other error fields. Every
error path now records all four fields through one function, `_cmdit_fail`, so no path can
leave a field from an earlier error behind. Before this, the three error fields were written
separately at about 20 sites.

### Changed — `cmdit_print_error` names the token and value, escaped

| error | 1.3.2 | 1.3.3 |
|---|---|---|
| UNKNOWN | `tool: unknown flag` | `tool: unknown flag: --bogus=1` |
| BAD_INT | `tool: --count: invalid integer value` | `tool: --count: invalid integer value 'abc'` |
| BAD_ENUM | `tool: --color: invalid value (expected a\|b)` | `tool: --color: invalid value 'purple' (expected a\|b)` |
| OUT_OF_RANGE | `tool: --count: value out of range` | `tool: --count: value '99' out of range` |
| UNEXPECTED_VALUE | `tool: --loud: option takes no value` | `tool: --loud: option takes no value (got '0')` |
| BUNDLED | `tool: bundled/attached short flags not supported (use -x -y)` | `tool: -vx: bundled/attached …` |
| TOO_MANY_POSITIONAL | `tool: too many arguments` | `tool: too many arguments: extra` |

The other codes print as before.

**Every argv-derived byte cmdit writes is escaped:** the token, the value, and the program name
on the `cmdit_new(0)` path in the error line, `cmdit_help`'s Usage line and `cmdit_version`. A
control byte (< 0x20) or DEL becomes `?`, so a token cannot put terminal escape sequences into
the user's terminal, a log or CI output. UTF-8 passes through. This is the convention owl
adopted for its own error lines (its audit FINDING-002). The 1.3.2 audit had accepted the raw
echo of `argv[0]` and the unknown-verb token as Info; both are now escaped too.

⚠ **stderr changes for the four consumers that call `cmdit_print_error`**: stiva (39 sites),
ifran, commandress and dig. None of their tests pins the old text; this was checked, not
assumed. kii and anuenue build their own lines and are unaffected.

### Fixed — 1.3.2 shipped five untracked-deferral lint findings

1.3.2's notes said lint was clean. `cyrius lint src/cmdit.cyr` in fact reported **5 untracked
deferrals**, all on 1.3.2's own `help_deferred` label: the word trips cyrius's
deferral-tracking rule. The release gate read only the last line of the lint output
("0 warnings"); the deferral count prints on the line above it, and `cyrius audit`'s lint stage
reports clean regardless. The label is renamed `help_handoff`, since nothing is deferred to a
backlog; help is handed to the verb. Both counts are now checked, and both are 0 on all seven
sources.

### Tests

- **+59 assertions (503 → 562):**
  - token and value for every error code, on the parse, dispatch-before-verb and
    dispatch-after-verb paths;
  - pointer identity for the token, and an interior pointer for an attached value;
  - no token or value leaking across a re-parse or a re-dispatch;
  - each new `cmdit_print_error` line, captured exactly;
  - escaping of an ESC sequence and DEL in a token and a value, and of a BEL in `argv[0]`
    across the error line, `cmdit_help` and `cmdit_version`, with UTF-8 left intact.
- **The fuzz harness gains I7.** A clean result carries no token or value. A token is always
  pointer-identical to an argv entry, and is present for every error that has one. A value
  points into argv, and is set exactly for the four value errors. I6 now also compares the
  token and value on a reused handle.
- **Mutation-proven, 18/18:** each recording site, both resets, the escaping (ESC and DEL
  separately), the help and version name, and the new print lines. The first round left four
  gaps (three survivors and one caught only by fuzz), each closed with a new assertion before
  the re-run.
- **Performance:** benches are within ±2% of 1.3.2, except `dispatch_before_verb` at +3.6%
  (~6 ns, the `_cmdit_fail` reset at the top of every dispatch). The bundle passes 562/562 on
  a 6.6.2 compiler.

## [1.3.2] - 2026-10-14

**An audit sweep, with one security fix.** It was the first full pass over `src/cmdit.cyr`
since 1.2.4. Three independent finders worked alongside a manual read: memory safety and
integers; contract and state; and the output and completion surface. Every claim was verified
before it counted, and every behavioural finding was first written as a failing test. The full
report is [`docs/audit/2026-10-14-audit.md`](docs/audit/2026-10-14-audit.md). **400 → 503
assertions**, and **33/33** seeded reverts of the fixes are caught (two fixes are unprovable by
any fixture and say so). The error taxonomy gains one append-only code, `CMDIT_ERR_ARGV = 13`.

### Fixed (SECURITY) — a program name starting with `-` became options to `complete`

1.2.4 sanitised the program name in generated completion scripts to `[A-Za-z0-9_.-]`, arguing
`-` is not a metacharacter in command-word position. As the **first** byte of the name given to
bash's `complete`, fish's `complete -c` or zsh's `#compdef`, it is read as options. On the
documented `cmdit_new(0)` path, the name is the attacker-chosen `argv[0]`. Verified on 1.3.1:
with `argv[0] = "-DCtrue"`, the script passed `bash -n`, and merely sourcing it replaced the
user's **default** completion (`complete -p -D` → `complete -C 'true' … -D`). bash then runs
that PATH command on every TAB for any command without its own completion.

The payload is limited to one bare command name, weaker than the 1.2.4 bug, but it has the
same attacker and the same sourced sink. A leading `-` is now mapped to `_` in all three
emitters. No shipped consumer was exposed: only stiva generates completions, and it names
itself with a literal.

### Fixed (attacker-reachable) — a command line too long or impossible to read whole parsed as a prefix

stdlib `args_init` reads `/proc/self/cmdline` once into a fixed 2 MiB buffer, and an
unprivileged `ulimit -s unlimited` raises ARG_MAX past that. Every whole token past 2 MiB was
dropped, and the parse returned `CMDIT_OK`: a wrapper's trailing `--read-only` can be padded
off the end. Verified on 1.3.1: 30 arguments of 100 KB then `--loud` gave exit 0 and
`loud = 0`. With no `/proc` mounted, `argc()` is 0, and the tool ran ignoring every flag.

Both now fail with **`CMDIT_ERR_ARGV` (13)**, "could not read the full command line", on the
6.7.9 and 6.6.2 compilers alike. The stdlib behaviour itself is filed against cyrius as
cyrius `docs/development/issues/2026-10-14-args-init-truncates-argv-past-2-mib.md`.

### Fixed (memory, program misuse)

- **Slot counts past 2^60 wrapped their byte size** to a buffer `alloc` granted:
  `cmdit_dispatch_argv` with `argc = 2^61` wrote past an 8-byte remainder, and growth at a
  2^61 + 1 capacity copied out of 16 bytes. 1.3.1's NOMEM test passed by luck. One bound,
  `_CMDIT_SLOTS_MAX`, now guards every slot-to-size conversion.
- **The public `CMDIT_*_MAX` / `_SIZE` vars were read live**, so mutating one overflowed the
  heap or hung a repeat push. The layout now lives in a private `enum CmditLayout` plus each
  handle's stored caps, and the public vars are read-only in effect, the model stdlib 6.7.9
  adopted for its own bound vars.
- **`cmdit_verb_alias` accepted any id.** An alias of a missing verb read outside the verb
  table when named on the command line. The target must now be a registered, canonical verb.
- **A 0 handle crashed ~25 public entry points**, although 1.2.4's comment said every entry
  point tolerated one. All are guarded now and fail closed: a 0 handle is `cmdit_new`'s OOM, so
  the parse family returns `CMDIT_RESULT_ERR`, `cmdit_error(0)` is `CMDIT_ERR_NOMEM`, and
  `cmdit_print_error(0)` says "out of memory".
- **`_cmdit_repeat_cap` looped forever past 2^62.**

### Fixed (contract)

- **Verb dispatch:**
  - A `cmdit_verb_trailing_after` verb counted its own flags as positionals, so in
    `exec --tty --root /x c1 ls` the global `--root` fell into the payload. Only positionals
    count now; a global inside the payload is still never applied.
  - `tool add --help` was blocked by a required global flag (ctx grows 168 → 176 B for the
    `help_deferred` mark).
  - On a marked verb, `exec c1 -- ls` ran the command `--`. A `--` at the threshold is now
    consumed, kubectl-style.
  - `-vx` before the verb said UNKNOWN; it is BUNDLED, as in the parse core.
  - Marking an alias row trailing silently did nothing; it now resolves to the verb.
- **Completions:**
  - zsh and fish now apply the 1.2.1 name whitelist, which only bash did. fish expands
    `complete -a` arguments at TAB time.
  - Short value flags are now in the skip set (`tool -j 5 <TAB>` took 5 for the verb).
  - An empty program name falls back to `prog`.
- **OOM:** the dispatcher's alloc failure now sets `CMDIT_ERR_NOMEM` (it set no code), and a
  failed argv materialization no longer parses as an empty argv.
- **Repeat flags:** a refused repeat push no longer overwrites the last-wins value.

### Performance

- **The verb table was still zeroed a byte at a time**, which 1.3.0 had fixed only for the
  entry table. Handle + first verb: **8.36 → 2.83 µs (−66%)**, paid on every run of every verb
  tool. A new `verb_new_floor` bench tracks it; every other bench is within ±2.5% of 1.3.1
  (dispatch +2.4%, the new guards). See [`docs/benchmarks.md`](docs/benchmarks.md).
- **argv is materialized once per process**, not once per handle. Each handle used to re-run
  `args_init` and its never-freed 2 MiB buffer.

### Docs

- README: `cmdit_parse` does not "short-circuit" (the whole argv is scanned; an error anywhere
  wins). New verb-dispatch notes: `--` before the verb ends global flags only, help-letter
  remapping is per handle, and the one-pass caveats.
- ADR-0002 gains "Later notes", covering the verb-row `+24` slot and the mirror ambiguity
  (`tool add --name --verbose`).
- Stale source comments are corrected (context layout, `FlagErr` compatibility, the
  NOT-supported list). The roadmap pins the survey's open gaps to 1.3.3–1.3.7.

### Consumers

Re-pinning breaks none of the six. Two behaviours are newly rejected, and both were never
meaningful:

- `-vx` before a verb now reports BUNDLED instead of UNKNOWN;
- a command line over 2 MiB now fails instead of being cut.

kii and anuenue's `cmdit_error` switches print their generic line for the new
`CMDIT_ERR_ARGV`. stiva's marked `exec` now accepts flags before the container, and consumes a
`--` after it.

## [1.3.1] - 2026-10-14

**Stdlib parity: two correctness defects that the parser cmdit was forked from fixed in
6.6.5, and cmdit still had.** Both silently did something other than what was typed. Both were
found by the 1.3.0 adoption survey and reproduced on `cmdit-smoke`. **357 → 400 assertions**,
and every change is mutation-proven: 10/10 seeded reverts are caught. The append-only error
taxonomy grows by two codes; nothing else in the API changes.

### Fixed — positionals past the 128th were dropped silently

`_cmdit_push_positional` stopped storing at `CMDIT_POS_MAX = 128` and reported nothing.
`cmdit-smoke $(seq 1 200)` printed `positionals: 128`, and the caller could not tell, because
`cmdit_positional_count` read exactly 128. That is a shell glob away from data loss:

- **shakti:** the privileged command line would have been truncated with exit 0, so policy
  evaluation, the audit record and `execve` would all see a shorter command than the one
  typed.
- **kriya:** `rm *` over 200 files deleted 128 and exited 0, until it moved to the stdlib's
  fix.

The positional array now **doubles** when full (128 → 256 → 512 …), copying the pointers
across, so every positional is kept in order. Parsing stays linear: 20,000 arguments parse in
2 ms and 100,000 in 5 ms.

⛔ **This reverses the 1.0.0 audit's SEC-3 ("accepted: memory-safe, lossy, documented").**
That acceptance rested on cmdit "matching the stdlib parser's drop-past-cap rule", and the
stdlib has grown its array since 6.6.5. The audit entry now records the revision
(`docs/audit/2026-06-25-audit.md`).

### Fixed — repeat flags past the 64th occurrence were dropped silently

The same rule capped a `cmdit_repeat` list at `CMDIT_REPEAT_MAX = 64`. Lists now double too.
Their capacity is **derived from the count** (64, doubled until it covers it) rather than
stored, so the 112 B entry layout is unchanged. `CMDIT_POS_MAX` and `CMDIT_REPEAT_MAX` are now
*initial* capacities. They were always internal constants, not API (ADR-0003).

### Added — `CMDIT_ERR_NOMEM = 12`: a refused growth is an error, never a drop

If the doubling alloc is refused, the parse fails with `CMDIT_ERR_NOMEM` ("out of memory"). A
repeat list names its flag via `cmdit_err_flag`; the positional array names none. Before
1.3.1, a refused repeat-list alloc *also* dropped silently. This mirrors stdlib
`FLAG_ERR_NOMEM` (6.6.7). It is unreachable through the public API, since the sizes are far
inside `ALLOC_MAX`. The tests reach it by setting a handle's capacity fields to 2^59, so the
doubling alloc must refuse.

### Fixed — `--bool=value` turned the flag ON, whatever the value

`_cmdit_apply`'s bool branch stored 1 and ignored an attached value. So `--verbose=0`, the
opposite of what was typed, *enabled* verbose, and `--loud=nonsense` parsed clean. A bool
given `=value` (including an empty `--flag=`) is now **`CMDIT_ERR_UNEXPECTED_VALUE = 11`**. It
names the flag and renders `prog: --loud: option takes no value`. It applies on every path:
`cmdit_parse_argv`, global flags before or after the verb in `cmdit_dispatch_argv`, verb
scopes via `cmdit_parse_verb`, and the auto `--help` / `--version` (`--help=x` is an error, not
help). **The env fallback is unchanged:** a bool bound with `cmdit_env` keeps presence
semantics, so `NO_COLOR`-style variables still work. This mirrors stdlib
`FLAG_ERR_UNEXPECTED_VALUE` (6.6.5).

⚠ **This rejects input that parsed before.** Any script passing `--some-bool=1` now fails with
exit 2. That input never meant what it looked like, but it is a visible change.

### Tests

- **+43 assertions (357 → 400):**
  - 300 repeat occurrences and 1,000 positionals, each a distinct pointer, checked in order
    through three regrowths, plus a white-box check that capacity doubled (not regrown per
    push);
  - 999 trailing args through verb re-entry;
  - both NOMEM paths and their `err_flag`;
  - `_cmdit_repeat_cap`;
  - `--loud=0`, `--loud=`, `--help=x`, and global bools before and after a verb;
  - the env-bool presence path;
  - the exact stderr lines for both new codes, captured through `pipe(2)` / `dup2(2)`.
- **Mutation-proven, 10/10.** Caught: each growth path reverted to the silent drop, a copy one
  slot short, the capacity not updated, a repeat cap that never doubles, the bool check
  removed, both NOMEM mappings, and both messages.
- **The fuzz harness** accepts the two new codes:
  - I2's bound is now `NOMEM`;
  - `UNEXPECTED_VALUE` joins I3's names-a-flag list;
  - a pool token now puts `--verbose=0` through the dispatch path.

  Every 50th argv is now long (up to 400 tokens) and mostly positional-shaped, with frequent
  `-H`. About 50 parses per run cross 128 positionals, and about 20 cross 64 repeats.
- `cyrius fuzz --poison=ab`, the suite under `CYRIUS_POISON=1`, and `cyrius audit` are all
  clean. The bundle passes 400/400 on a 6.6.2 compiler.

### Consumers

Re-pinning to 1.3.1 breaks none of the six. kii and anuenue switch on `cmdit_error` with an
`else` fallback, so a `--bool=value` gets their generic "argument parse error" line; a branch
for `CMDIT_ERR_UNEXPECTED_VALUE` would name it. anuenue's fuzz bound
`err <= CMDIT_ERR_TOO_MANY_POSITIONAL` still holds, because its pool has no bool-with-value
token, but the taxonomy maximum is now `CMDIT_ERR_NOMEM`.

## [1.3.0] - 2026-10-14

**Toolchain `6.6.20` → `6.7.9`, `cmdit_new` ~6× faster, `cmdit_argv` linear instead of
quadratic, and a fuzz harness that actually fuzzes.** No API change: the frozen 1.0.0 surface
plus the 1.1–1.2 additions behave identically on every input. **357/357** assertions (345 + 12
for the argv fix). The new fuzz harness passes, and so does `cyrius fuzz --poison=ab`, with the
whole suite clean under `CYRIUS_POISON=1`. `cyrius audit` is clean on every stage.

### Changed — `cmdit_new` zeroes its entry table a word at a time

The 64 × 112 = 7168-byte entry table was zeroed with 7168 `store8`s. That one loop was
essentially the whole cost of building a handle. It is now 896 `store64`s, with a byte loop
for any tail if the table size ever stops being a multiple of 8. Interleaved A/B on the same
build, median of 7:

| bench | 1.2.6 | 1.3.0 | |
|---|---|---|---|
| `cmdit_new_floor` | 11.56 µs | **1.94 µs** | −83% |
| `parse_argv_mixed` | 11.77 µs | 2.37 µs | −80% |
| `repeat_accumulate_32` | 12.53 µs | 3.00 µs | −76% |

⚠ **Not covered by a failing test, and recorded as such.** Deleting the loop entirely keeps
all 345 assertions and the fuzz harness green, even under `CYRIUS_POISON=1`. The memory
`alloc` hands out is already zero, and poison mode fills only the redzones. cmdit keeps the
explicit zero because `alloc` does not promise zeroed memory on every target. Like the 1.2.4
alloc-failure check beside it, it is defence-in-depth that no fixture can reach.

### Fixed — `cmdit_argv` was quadratic in the argument count on Linux

`cmdit_argv` asked stdlib `argv(i)` for each index in turn, and on Linux `argv(i)` re-walks
`/proc/self/cmdline` from byte 0 to the i-th NUL on every call. Materializing the process
argv therefore cost O(argc × bytes) before parsing even began. Measured end to end on
`cmdit-smoke` with N positionals:

| N | 1.2.6 | 1.3.0 |
|---|---|---|
| 1,000 | 7 ms | 2 ms |
| 5,000 | 159 ms | 4 ms |
| 20,000 | **2,655 ms** | **11 ms** |
| 100,000 | — | 48 ms |

The arguments sit end to end in that buffer, so one walk finds them all. The new pure helper
`_cmdit_split_flat` does that walk, and `cmdit_argv` uses it under
`#ifdef CYRIUS_TARGET_LINUX`. Other targets keep `argv(i)`: agnos reads a pointer table, which
is already O(1) and whose strings need not be adjacent. The result is pointer-identical to the
old loop, including empty arguments (a bare NUL). Found by the kriya adoption survey: kriya had
already fixed the same defect in its own argv bridge (kriya 1.6.17).

**+12 assertions (345 → 357).** The helper is driven with a synthetic buffer that includes an
empty argument and a sentinel slot. The live process argv must come back pointer-identical to
`argv(i)` and NULL-terminated. Mutation-proven: a walk that skips the separator, a walk that
writes one slot too many, and a Linux branch that starts at the wrong argument are each
caught. A dropped NULL terminator is not, for the same zeroed-memory reason as above.

### Fixed — the fuzz harness was a stub

`tests/cmdit.fcyr` called `fuzz_main("test", 4)`, which returned 0 without touching cmdit.
Every `cyrius fuzz` since 0.1.0 reported PASS on a harness that exercised nothing. It is
replaced by a real one: 6,400 seeded argvs per run (16 seeds × 400) through both pure cores,
`cmdit_parse_argv` and `cmdit_dispatch_argv`. Each argv mixes a 48-token pool (every
registered flag in both spellings, good and bad values, verbs and an alias, `--`, bundled and
attached shorts, `--=`) with random byte strings. It then checks the documented contract,
not particular outcomes:

- **I1** the result code is documented.
- **I2** the error is a `CmditErr`, and is NONE exactly on a clean result.
- **I3** `cmdit_err_flag` names a registered flag for the five errors that name one, and
  is -1 for the rest.
- **I4** every positional and remainder-slice token is pointer-identical to an argv token,
  and the counts fit argc.
- **I5** on a clean parse, the enum index is a choice, the ranged int is in bounds, and the
  repeat list is dense. Then `cmdit_finalize`'s required gate and
  `cmdit_require_positionals_max`.
- **I6** two fresh handles agree, and a handle whose previous parse *failed* reports the same
  error as a fresh one. That is the 1.2.4 `err_entry` leak class, now checked for both cores.

A failure prints the invariant, seed, iteration and the argv, then exits 1.

⭐ **Mutation-proven: 8 of 9 seeded source bugs are caught.** Caught: dropping the 1.2.4 error
reset in either core, MISSING_VALUE no longer naming its flag, a positional stored from the
wrong pointer, the range upper bound disabled, enum validation disabled, the required check
disabled, and the 1.2.4 negative-argc clamp removed. The survivor is dropping the remainder
slice's NULL terminator. That is unobservable for the same reason as the zeroing loop above:
the slot is already zero.

The harness also runs clean under `--poison=ab` (fill 0xA5 and 0x5A, identical output) and the
full suite under `CYRIUS_POISON=1`: no redzone overwrite anywhere in cmdit.

### Changed — toolchain and libs

- **Pin `6.6.20` → `6.7.9`.** This time `cyrius lib sync --full --relock` ran **before** the
  first build, so there was no refusal (see 1.2.6). `lib/` is byte-identical to `6.7.9/lib`:
  113 files, 67 modified, none added or removed. `cyrius.lock` was refreshed.
- **6.7.x's language changes were checked against cmdit's sources, not assumed:** the new
  reserved words (`trait`, `const`, `true`/`false`, `do`, `dyn`); `!` becoming a real operator
  (6.7.3; before it, a lone `!` was silently dropped); narrow `*T` dereference (6.7.8); and
  the unmatched-`}` and for-step refusals (6.7.9). Every hit sits inside a string literal or a
  comment.
- **Artifacts moved by 8 bytes**, except `cmdit-verbs`, which grew 4,104 B. With
  `CYRIUS_DCE=1` its live code also moved 8 B. The difference is one new unreachable stdlib
  function (`sys_ppoll`, additive in 6.7.9) pushing the file over a 4 KB alignment boundary.
- **`dist/` regenerated.** The bundle differs from 1.2.6 only in the version stamp, the zeroing
  loop and the argv walk; the `cmdit.deps` sidecar is unchanged. It was re-verified on a
  **6.6.2** compiler (the full suite through `dist/cmdit.cyr`, 357/357, with the argv fast path
  live: 20,000 arguments in 22 ms), so consumers pinned as far back as 6.6.2 can take it.

### Changed — docs and audit hygiene

- **`docs/benchmarks.md` re-captured, with the v1.0 drift finally attributed.** Every number is
  now a median of 7 interleaved runs. A bisect compiled the same harness against every tag's
  source, and the 1.0.0 source under every installed toolchain:
  - No toolchain from 6.6.2 to 6.7.9 moved anything.
  - From 1.0.0 to 1.2.6, construction and parsing were flat. Dispatch got 12% slower at
    1.2.2, the cost of forwarding `--` and of `verb_trailing_after`.
  - The v1.0 table's 13.8 µs is not reproducible on any installed toolchain (the 1.0.0 source
    measures 11.5 µs everywhere). The gap lives in the uninstalled 6.2.44 → 6.6.2 span or in
    capture conditions.
- **`cyrius audit` flagged 10 undocumented functions** in `programs/` and `tests/` (`main` and
  local print helpers). Each now has a one-line comment, so every audit stage is clean.
- **A stale claim in five entry files was corrected**: "`cyrius build` does not auto-prepend
  stdlib for a stdlib-only project". It does; a normal build carries `assert_*` functions
  that `--no-deps` does not. The explicit includes stay, so the entries still compile
  standalone.
- **Two source comments corrected; code unchanged.** The `CmditErr` comment claimed byte
  compatibility with stdlib `FlagErr`. Codes 0..4 still match, but from 5 the two diverged:
  the stdlib gave 5 to `UNEXPECTED_VALUE` in 6.6.5. `cmdit_required`'s comment cited agnova's
  `--user`, which agnova has since made optional.
- **README: a handle is single-parse.** That was always the model (ADR-0002 builds a fresh
  handle per verb), but nothing said it. A second parse on the same handle resets only the
  error fields (1.2.4); positionals, values and the matched verb carry over.

### Adoption survey — notes filed in 18 repos, and the gaps they found

Each of the 18 candidates from the 2026-06-25 review now has an **"Adopting cmdit (CLI
parsing)"** section appended to its `docs/development/roadmap.md`. Each maps the repo's
hand-rolled CLI sites (`file:line`) to cmdit calls, names the gaps, and lists the tests and
goldens that must migrate. The edits were append-only: no source, manifest or `lib/` was
touched, and the pre-existing uncommitted roadmap edits in agnova, attn11, hapi, phylax and
whirl were preserved byte for byte. klug had no roadmap, so one was created.

Verdicts range from clean drop-ins (yo, whirl, takumi) to **blocked**:

- **shakti** — silent positional cap, no top-level stop-at-positional, and no offending token.
- **kriya** — positional cap, plus clusters and optional values.
- **cyim** — optional-value and once-only flags.
- **bannermanor** — migrating would regress fixes it already has from stdlib `flags`.
- **klug** — its CLI is too small to be worth it.

The gaps, ranked by how many repos they block, are in
[`docs/development/roadmap.md`](docs/development/roadmap.md). The top two are correctness
defects that **the stdlib parser cmdit was forked from fixed in 6.6.5**, and cmdit still has:

- the silent 128-positional cap;
- `--bool=0` setting the flag.

Both reproduce on `cmdit-smoke`. They change behaviour, so they are deliberately **not** in
1.3.0 and are deferred to a later release.

## [1.2.6] - 2026-10-14

**Maintenance: toolchain `6.6.2` → `6.6.20` and a full vendored-stdlib refresh.** No behaviour
change in `src/cmdit.cyr` (one doc-comment reflow, byte-neutral). **345/345** assertions, fuzz,
and the 7 benchmarks are green.

### Changed

- **Toolchain pin `6.6.2` → `6.6.20`.** CI reads it from `cyrius.cyml`, so no workflow edit.
- **`cyrius lib sync --full --relock`** re-vendored `lib/`: 113 files, byte-identical to
  `6.6.20/lib`. 95 were modified and 5 are new (`alloc_cx`, `boxed`, `hashseed`, `poison`,
  `tls_hostid`); none were removed. The shadow warning that every 1.2.5 build printed (12 stale
  bundled libs) is gone.
- **`cyrius.lock` is now tracked.** 6.6.20's build writes it (113 sha256 rows) and checks
  `lib/` against it. The sibling repos already commit theirs.

  ⚠ Upgrade gotcha: the first `cyrius build` under the new pin auto-vendored the declared
  stdlib set (25 files with platform variants) and wrote `cyrius.lock` with **every other file's
  old hash**. After that, a plain `lib sync --full` refused 73 rows as "DISAGREE" and wrote
  nothing. `--relock` is the documented recovery when the pin has just moved. Each refused row
  was checked against the committed previous-pin file before relocking. A changed installed
  snapshot would have needed a reinstall instead.
- **`dist/` regenerated, and `cyrius distlib` changed the bundle's shape.** `dist/cmdit.cyr` now
  opens with a compile-verified `# Requires` block (`include` of `lib/alloc`, `string`, `args`,
  `io`). `dist/cmdit.deps` lists exactly those 4 leaves instead of all 10 `[deps].stdlib`
  entries. The 6 that dropped out (`fmt`, `vec`, `str`, `syscalls`, `assert`, `bench`) are needed
  by cmdit's own programs and tests, not by the library. Fourteen sibling libraries already ship
  this layout. Consumer-side check: smoke and the full suite were built against
  `dist/cmdit.cyr` instead of `src/`. The bundle's includes overlap the program's own, the
  compiler dedupes them (binary size is identical), and all 345 assertions pass.
- **`cmdit_require_positionals` doc comment reflowed.** 6.6.20's `cyrlint` requires a
  deferral's cross-reference on the *same line* as the word "Deferred". 6.6.2 accepted it on the
  next line. Lint is back to 0. All five artifacts are sha256-identical before and after.

### Stdlib delta on cmdit's call surface

`strlen`, `args_init`, `argc` and `argv` are body-identical. Two entry points changed:

- **`getenv`** (stdlib 6.6.8): if its internal copy allocation fails, it now returns 0 (a miss).
  Before, it stored to address 0. This reaches cmdit through `cmdit_env` → `cmdit_finalize`.
  Under OOM, an env fallback now reads as "unset", which cmdit already handles as the ordinary
  path.
- **`alloc`**: only `#ifdef CYRIUS_POISON` instrumentation (the new `lib/poison.cyr`). It is
  inert in a default build.

### Binary size: +30 KB, almost all of it unreachable stdlib

`cmdit-smoke` grew 120,104 → 150,728 B. With `CYRIUS_DCE=1`, the **live** code went 38,184 →
44,232 B (+6 KB). The other ~24.6 KB is unreachable stdlib (387 → 474 unreachable fns), which
the opt-in DCE pass drops. The bundled-lib half of the sync is **byte-neutral**: smoke, verbs,
completions, tcyr and bcyr are sha256-identical before and after `--full`. All of the growth
comes from the compiler plus the declared-stdlib move.

## [1.2.5] - 2026-09-11

### Changed

- **Toolchain `6.5.35` → `6.6.2`.** No source change. cmdit constructs no
  `Result` / `Option` / `Either`, so the 6.6.0 value form has no surface here:
  zero sites the compiler rejects, zero fail-open sites, and no arity collision
  across the compile set. **345 assertions** pass unchanged.

  The three `callptr` sites were traced to their full function-pointer target
  sets; no reachable target is pair-returning, which is the only way to rule that
  class out — the compiler cannot see through an indirect call, so a clean build
  is not evidence for it.

## [1.2.4] - 2026-08-23

**A P-1 audit of the whole library. One finding is a remote-ish code execution in the completion
generator; three more are memory-safety or contract defects.** No API change — every fix is a
guard, and the frozen 1.0.0 surface plus the 1.1–1.2 additions behave identically on every input
that was already valid. **267 → 345 assertions**, and each fix below is **mutation-proven**: the
fix was reverted, and a test failed.

### Fixed (SECURITY) — the program name was injected verbatim into generated completion scripts

`cmdit_completions` interpolated `prog` **raw** into all three emitters, in five places: the bash
`# bash completion for <prog>` comment and its `complete -F _<fn>_complete <prog>` trailer, zsh's
`#compdef <prog>`, and fish's `complete -c <prog> -f` plus one `complete -c <prog>` per verb. A
completion script is *sourced by the user's shell*, so a byte reaching it verbatim is code.

`prog` is `prog_name` from `cmdit_new` — and on the documented, tested `cmdit_new(0)` path it is
adopted from **`argv[0]`**, which is chosen by whoever execs the tool. With
`argv[0] = "tool\ntouch /tmp/x\n#"` the generated file was:

```
# bash completion for tool
touch /tmp/x
#
_tool_touch__tmp_x___complete() {
```

Verified end to end: the result passes `bash -n` — it is a *valid* script — and running
`source` on it executed the injected command.

⛔ **1.2.1 fixed this class for verb and long-flag names and explicitly named the program name in
its own notes — but only ever routed it through `_cmdit_oprint_ident` for the shell FUNCTION
name.** The command-word occurrences were never touched. Anyone reading that changelog would
reasonably conclude the program name was handled; it was not.

**No shipped consumer is currently exposed** — verified rather than assumed. Exposure needs both
a `cmdit_completions` call and a program name taken from `argv[0]`. Of the four consumers, only
stiva calls it, and its top-level handle is `cmdit_new("stiva")`; kii, anuenue and ifran pass
literals and generate no completions. The defect is real and the fix is worth taking, but nothing
downstream is broken today.

New `_cmdit_oprint_cmdname` maps every byte outside `[A-Za-z0-9_.-]` to `_`, so the emitted token
is always exactly one inert word. It **maps rather than drops** (the 1.2.1 remedy for verb names)
because a completion with no command to attach to is useless. `.` and `-` are kept, so ordinary
names survive byte-for-byte — `complete -c cmdit-smoke` still reads `cmdit-smoke`, and the
generated scripts still pass `bash -n` and `zsh -n`.

The alphabet now lives in one predicate, `_cmdit_is_cmdname_byte`, shared with
`_cmdit_name_is_safe`, and the suite asserts it over **all 256 byte values** rather than over the
metacharacters someone happened to think of. A second test captures the emitters' real stdout
through `pipe(2)`/`dup2(2)` and asserts the injected command is absent — testing the predicate
alone would still pass if an emitter simply stopped calling it, and the mutation run confirms all
five call sites are individually covered.

### Fixed (memory safety) — the flag modifiers took an unchecked index and WROTE through it

The 1.0.0 audit added `_cmdit_idx_ok` to the typed getters. The modifiers never got it: only
`idx < 0` was tested, so `cmdit_required` / `cmdit_range` / `cmdit_env` / `cmdit_metavar` turned
an out-of-range index into an out-of-bounds **write**.

`cmdit_required(h, 64)` — one past `CMDIT_FLAGS_MAX` — read-modify-wrote `entries + 7216`, which
the bump allocator places **48 bytes into the positional array**. Demonstrated: a live
`positional[6]` pointer to `"p6"` came back incremented by one, i.e. a valid cstr pointer turned
into one aimed at the middle of a string. All four now share the getters' guard and return -1.

### Fixed (memory safety) — `cmdit_repeat_get` dereferenced a pointer read from out of bounds

It guarded `i` but not `idx`, and computed `_cmdit_entry(h, idx)` before any check — so a bad
`idx` did not merely read out of bounds, it **followed the list pointer it found there**.
`cmdit_repeat_count` was unguarded outright. Both take `_cmdit_idx_ok` now, which also gained a
null-handle test, hardening every getter that routes through it.

### Fixed (contract) — `err_entry` leaked across parses, so errors named the wrong flag

`cmdit_err_flag` documents -1 for errors that name no registered flag (UNKNOWN / BUNDLED), and
`cmdit_print_error` prefixes the flag name for every error except UNKNOWN. Neither parse loop
ever cleared `err_entry`, so that held **only on a virgin handle**. Parse a handle twice — first
hitting `--name` with no value, then a bundled `-xy` — and 1.2.3 reported:

```
prog: --name: bundled/attached short flags not supported (use -x -y)
```

naming a flag from the *previous* parse. `cmdit_parse_argv` and `cmdit_dispatch_argv` now reset
the error trio (`last_error`, `err_entry`, `err_verb`) on entry. ⭐ Per-site clears were tried
first and are **not** in the final fix: mutation testing showed them redundant against the reset —
every such site is downstream of it — so the fix is one mechanism, not two.

### Fixed (input validation)

- `cmdit_verb_trailing_after` accepted a negative `n`. It stores `n + 1`, so `n == -1` silently
  meant "not marked", and `n < -1` stored a negative that `cmdit_verb_trailing_at`'s `v == 0`
  test did not catch — handing the dispatcher a nonsense threshold. Now -1, plus the missing
  null-handle and empty-verb-table guards.
- `cmdit_dispatch_argv` with a negative `argc` called `alloc((argc+1)*8)`, which returns 0 for a
  non-positive size, and then wrote the slice's NUL terminator **to address 0**. `argc` is clamped.

### Hardened — stdlib `alloc` returns 0 on failure and every call site ignored it

`lib/alloc.cyr` returns 0 for OOM or an out-of-range size; it does not abort. `cmdit_new` then
memzeroed 7168 bytes from address 0. Checks added in `cmdit_new` (returns 0), `cmdit_argv`,
`_cmdit_repeat_push`, `_cmdit_verb_add` and `cmdit_dispatch_argv`.

⚠ **Recorded as unproven:** the `cmdit_new` check is the one change here with no failing test.
Its three sizes are compile-time constants well inside `ALLOC_MAX`, so no public API call can
drive `alloc` to return 0 — deleting the line keeps the suite green. It is defence-in-depth
against an exhausted heap, not behaviour any fixture reaches.

### Removed — one guard that was provably dead

`cmdit_repeat_get`'s null-list check is unreachable: the list is allocated by the same push that
makes the count non-zero, so `count == 0` whenever the pointer is 0 and the count test above
already returns. Mutation-verified, not assumed, and replaced by a comment stating the invariant.

### Quality gates — all clean, across every file

`cyrius fmt --check`, `cyrius lint` and `cyrius doc --check` now pass on all seven sources
(`src/`, three `tests/`, three `programs/`). Measured at the 1.2.3 tree, the baseline was **9
untracked deferrals and 16 warnings** across those files, plus 8 undocumented functions; it is
0/0/0 now (the new 1.2.4 tests are held to the same bar):

- **Docs**: 53 documented, **0 undocumented** (was 46/8). The gaps were accessors sharing a block
  comment with a sibling — `cmdit_get_int`, `cmdit_get_str`, `cmdit_repeat_get`,
  `cmdit_positional`, `cmdit_verb_argc`, `cmdit_verb_argv`, `cmdit_raw_argc`,
  `cmdit_version_short` — each now carries its own.
- **Lint**: 0 deferrals, 0 warnings. One deferral was real (the NOT-supported list) and now
  cross-references the roadmap and ADR 0003; the rest were the word "deferred" used in its
  domain sense — a flag deferred into a verb's scope — and carry `#skip-lint` rather than prose
  bent to satisfy a keyword scan.
- **Format**: `cyrius fmt` applied; the diff is whitespace-only (`diff -w` is empty).
- **Corrected a long-standing doc figure**: `docs/development/state.md` has claimed "8 real
  benchmarks" since 1.0.0, and the 1.2.3 entry above repeated it. `tests/cmdit.bcyr` defines
  and runs **7** — which is what `docs/benchmarks.md`'s own table has listed all along. Both
  are now 7.

### No behaviour change on anything that already worked

Every fix is a guard on an input that previously corrupted memory or emitted code, so nothing
valid should move — checked rather than asserted. The 1.2.3 and 1.2.4 builds of all three demo
programs were run over **27 invocations** — help, version, every error arm (bad int, out of
range, bad enum, unknown flag, bundled short, missing value), `-`, `--`, repeats, positionals,
global-flag-before/after-verb, unknown verb, and all four `completions` shells — and stdout,
stderr and exit code are **byte-identical** in every case. `cmdit_help`'s output in particular
is unchanged despite the renderer's `if`/`elif` chain being re-wrapped for the column limit.

## [1.2.3] - 2026-08-23

**Maintenance: toolchain pin 6.4.78 → 6.5.35 and a full vendored-stdlib refresh.** No change to
`src/cmdit.cyr` — the public surface is byte-for-byte the frozen 1.0.0 API plus the 1.1–1.2
append-only additions.

Both halves were overdue and each warned on every single build:

```
warning: cyrius.cyml pins 6.4.78 but cycc is 6.5.35 — toolchain drift
warning: ./lib/ shadows version-pinned .../6.4.78/lib — 12 bundled lib(s) differ
```

- **Pin 6.4.78 → 6.5.35.** CI derives the toolchain from `cyrius.cyml [package].cyrius`, so no
  workflow edit was needed.
- **`cyrius lib sync --full`** re-vendored `lib/` from the new pin: 108 files, now byte-identical
  to `6.5.35/lib`. 70 modified, 3 new top-level modules (`async_macos`, `async_win`,
  `thread_macos`) and the new `lib/unicode/` subtree; nothing removed. The 12 stale bundled libs
  the shadow warning named (bayan, ganita, sakshi, niyama, sigil, sandhi, yukti, patra, vani,
  mabda, sankoch, yantra) are current.

### The upgrade is codegen-neutral for cmdit, and that was measured rather than assumed

⭐ **Both halves were A/B'd separately, and both are byte-neutral.** Holding one variable fixed
and swapping the other, every artifact — `programs/smoke.cyr`, `programs/verbs.cyr`,
`programs/completions.cyr`, `tests/cmdit.tcyr`, `tests/cmdit.bcyr` — comes out **byte-identical
on sha256**:

| A/B | held fixed | swapped | result |
|---|---|---|---|
| compiler | 6.5.35 `lib/` | cycc 6.4.78 ↔ 6.5.35 | 5/5 byte-identical |
| vendored lib | cycc 6.5.35 | `lib/` 6.4.78 ↔ 6.5.35 | 5/5 byte-identical |

So the whole of 1.2.3 is a no-op at the machine-code level for cmdit. 6.5.35's headline is a
register allocator that finally time-shares registers, but its own release notes scope the win to
straight-line regions, and cmdit's hot paths do not hit it.

⚠ **`docs/benchmarks.md` is therefore NOT re-captured in this release, and its numbers are
stale for a different reason.** A fresh capture on this tree reads `cmdit_new_floor` **11.4 µs**
against the documented **13.8 µs**, and `dispatch_before_verb` **153 ns** against **203 ns** —
but the byte-identity above proves none of that delta is this toolchain bump. That table was
captured at the v1.0 cut under pin 6.2.44 against 1.0.0 source, so the gap belongs to the
1.0.0 → 1.2.3 source evolution (ctx 160 → 168 B, verb introspection, completions) and to capture
conditions. Attributing it needs a bisect, which is its own change, not this one.

### No stdlib delta lands on cmdit's call surface, and that surface is smaller than it looks

`string`, `fmt`, `alloc`, `io`, `vec`, `syscalls`, `args`, `assert` and `bench` all moved; `str`
is unchanged. Every public signature delta is **additive** (`vec_sort_by`/`vec_select_nth`,
`arena_new_growable`, the `xmkdir_p`/`xsymlink` family, `signal_default`, the bench clock-overhead
calibration). Two `alloc` internals were removed — `_arena_alloc`, `_arena_reset` — and cmdit
references neither.

The reason nothing reaches it: cmdit's **entire** external surface is `alloc`, `args_init`,
`argc`, `argv`, `getenv`, `strlen`, `load8/64`, `store8/64` and `syscall`. It renders no integer
to output at all — `_cmdit_eprint`/`_cmdit_oprint` are raw `syscall(1, …)` string writes, and an
int/range error prints the flag name and a fixed message, never the value. And every stdlib
entry point it *does* call is **byte-identical** across the two snapshots — `getenv`, `strlen`,
`alloc`, `args_init`, `argc`, `argv`, compared body-for-body; `getenv` merely moved down
`io.cyr`, and `args.cyr`'s sole change is a comment. Which is why the lib-swap A/B above comes
out byte-identical rather than merely equivalent.

⚠ Two changes in the refresh look like they should matter here and **do not** — recorded so the
next reader does not re-derive it:

- **`string.cyr:print_num` i64::MIN** (cyrius 6.5.8) — `0 - n` is a no-op at the minimum, so the
  old `n > 0` loop emitted zero digits and printed a bare `-`. cmdit never calls `print_num`
  (0 references in `src/` and `dist/`).
- **`assert.cyr:assert_eq`** moved its `got`/`expected` output from `fmt_int` (fd 1) to
  `efmt_int` (fd 2). cmdit's harness uses `assert`/`assert_summary`, not `assert_eq`.

**267/267 assertions green, unchanged from 1.2.2**, plus fuzz, all 7 benchmarks, and the three
demo programs. `dist/cmdit.cyr` regenerated at 1.2.3; `cyrius distlib` on 6.5.35 additionally
emits the **`dist/cmdit.deps`** sidecar (the 10 stdlib leaves this fold needs in scope), which
`cyrius deps` consumes on the consumer side — new file, tracked, matching stiva.

## [1.2.2] - 2026-07-26

**A verb could not forward a command line.** Both halves found by adversarial review of stiva's
`exec`, which needs exactly this and could not ship without it.

### Fixed — the `--` terminator was consumed by the dispatcher and never forwarded
`cmdit_dispatch_argv` treated `--` as "stop parsing global flags" and `continue`d **without copying
the token into the remainder slice**, so the verb's own `cmdit_parse_argv` — which handles `--`
correctly — never saw one. The documented escape hatch for a command carrying flags therefore did
nothing, and `prog verb -- cmd -x` failed **identically** to omitting it:

```
stiva exec c1 ls -l       -> "unknown flag", the command never runs
stiva exec c1 -- ls -l    -> the same error
```

A `--` appearing BEFORE the verb is still consumed, because `remainder[0]` must be the verb name
and the terminator has already done its job there.

### Added — `cmdit_verb_trailing_after(h, verb_id, n)`
Marks a verb as taking a **verbatim** trailing argument list after its first `n` positionals.
Past that point nothing is treated as a flag, at either the dispatcher or the verb level, and the
mark propagates from the top-level handle into `cmdit_parse_verb` so a consumer declares it once.

This is a correctness fix, not a convenience. Without it a **global** flag appearing inside a
forwarded command was applied to the host tool:

```
stiva exec c1 mycmd --root /x   -> silently retargeted STIVA's data root
```

No diagnostic, and the payload never saw the argument it was passed. `cmdit_verb_trailing_at`
reports the mark (−1 when unset). Aliases inherit it through the canonical row.

`CMDIT_CTX_SIZE` grows 160 → 168 for the new per-handle slot. Unmarked verbs are byte-for-byte
unchanged. **250 → 267 assertions.**

## [1.2.1] - 2026-07-25

**Completion-generator fixes, all found by adversarial review of the stiva adoption.** 1.2.0's
generated scripts were valid shell but wrong in two ways that matter.

- **The verb-position guard was wrong whenever a global flag preceded the verb.** bash used
  `[[ $COMP_CWORD -eq 1 ]]` and zsh `(( CURRENT == 2 ))`, so `tool --root /x <TAB>` — the ordinary
  invocation for any tool with a global value-flag — completed filenames instead of verbs.
  Verified live against bash 5.3 with the real generated script. Both emitters now walk the words
  before the cursor for the first non-flag token, skipping the argument of a value-taking flag,
  using a new `valueflags` list derived from the entry table's types.
- **The escaping claim in 1.2.0's changelog was false.** Verb names and long-flag names were
  interpolated into `local verbs='…'` with **no** escaping, and the program name was emitted
  outside any quoting. More importantly, escaping alone cannot fix this: `compgen -W "$verbs"`
  re-expands each word, so a `$(…)` or a backtick in a name would execute at TAB time even inside
  correct single quotes. Names are now **whitelist-filtered** to `[A-Za-z0-9_.-]` and dropped from
  the completion if they fail — a missing completion is a papercut; a command substitution running
  in the user's shell is not.

No consumer was exposed: stiva's verb and flag names are all safe literals. This is a library
contract defect, fixed before anything relied on the contract.

## [1.2.0] - 2026-07-25

**Verb introspection + shell completions.** Append-only and non-breaking on the frozen 1.0.0
surface — new functions only.

The verb table already drove `cmdit_verbs_help`, but nothing exposed it, so a consumer could not
generate shell completions from the very verbs it had just registered: it had to keep a second,
hand-maintained list, which drifts the moment a verb is added. Surfaced by the stiva adoption
(35 verbs, a `completions` subcommand among them).

- `cmdit_verb_count(h)` — total rows, aliases included.
- `cmdit_verb_name_at(h, i)` / `cmdit_verb_help_at(h, i)` — name and help. An alias row stores no
  help of its own, so `help_at` returns the **canonical** verb's, or a caller listing aliases
  would render a blank description.
- `cmdit_verb_is_alias(h, i)` / `cmdit_verb_canonical_at(h, i)` — the same
  `canonical_id == own index` test `cmdit_verbs_help` uses to skip alias rows.
  Out-of-range and null-handle inputs return 0 / -1, never a wild read.
- `cmdit_completions(h, shell)` — writes a bash, zsh, or fish completion script for `h`'s
  registered verbs and long flags. Returns -1 for an unrecognised shell **without printing
  anything**, so a caller can report the error without a half-written script on stdout.

Two details that matter for generated shell:

- Output goes to **stdout**, not stderr — the script is meant to be redirected into a completions
  file or eval'd.
- Help text is caller-supplied prose, so every value interpolated into a single-quoted shell
  string has its apostrophes escaped (`'` → `'\''`). An unescaped one would close the quote and
  inject the remainder into the script. Function names derived from the program name are
  sanitised to `[A-Za-z0-9_]` for the same class of reason.

Verified by syntax-checking the generated output with the real parsers: `bash -n` and `zsh -n`
both accept it, including a verb description containing an apostrophe. `programs/completions.cyr`
is the runnable demo.

Also: toolchain pin 6.2.44 → 6.4.78 (the suite is green on it; the old pin warned on every build).

## [1.1.0] - 2026-06-25

**Rich-help support (`cmdit_help_flags`).** Append-only and non-breaking on the
frozen 1.0.0 surface — a new function only. Surfaced by the anuenue adoption pilot:
tools that frame their own help (a brand intro + custom usage synopsis + an Examples
block) need to wrap a *generated* flag list, which the monolithic `cmdit_help`
couldn't provide. The re-survey under-counted this (deemed it beneficiary≈1); the
pilot is the second concrete consumer, so it graduates from caller-side to a tiny
library primitive.

### Added
- **`cmdit_help_flags(h)`** — the table-only flag renderer: prints just the
  `  -x, --long <type>\thelp` rows (no `Usage:`/`Options:` header, no verb list), so a
  consumer can sandwich the generated rows inside its own help prose. `cmdit_help`
  now composes it; `cmdit_help`'s output is byte-for-byte unchanged from 1.0.0.

## [1.0.0] - 2026-06-25

**The v1.0 freeze.** The public API is frozen. This cut lands the final completeness
delta a 23-repo re-survey of hand-rolled CLI code found still uncovered, fixes the two
low-severity correctness issues an adversarial security audit confirmed, and meets the
remaining v1.0 gates (benchmarks, security-audit doc, CHANGELOG, doc-drift). Append-only:
`CMDIT_BOOL/INT/STR` = {0,1,2}, every prior entry/ctx offset, and the 0..9 error codes
are untouched; the entry struct grows 104→112 B for the metavar.

### Added
- **`cmdit_help_short(h, ch)` / `cmdit_version_short(h, ch)`** — remap (any ASCII byte)
  or disable (`ch = 0`) the auto `-h` / `-V` short letters, keeping the `--help`/
  `--version` long forms + detection. Frees `-h` for a consumer's own flag (the
  `df -h`/`ls -h` human-readable convention). No struct/error growth.
- **`cmdit_metavar(h, idx, name)`** — override a value flag's generated-help placeholder
  (`--out <FILE>` vs `<str>`); cosmetic, parsing-neutral, no-op on bool flags. The
  generated help now also renders the registered default of string flags (`(default: …)`;
  cstr only — int defaults stay unrendered, preserving the no-`fmt` renderer contract).
- **`cmdit_require_positionals_max(h, max)` / `cmdit_require_positionals_exact(h, n)`** —
  the upper-bound and both-bounds mirrors of `cmdit_require_positionals`. New error
  **`CMDIT_ERR_TOO_MANY_POSITIONAL = 10`** (`too many arguments`, exit 2); reuses the
  positional count (no struct growth).

### Fixed
- **`_cmdit_parse_int` integer overflow** (audit SEC-1) — a long all-digit argv/env token
  silently wrapped mod 2⁶⁴ and could **bypass a `cmdit_range` bound** (`18446744073709551617`
  → wraps to 1 → passed 1..4096). Now rejected before the multiply (`> (INT64_MAX − d)/10`)
  → `CMDIT_ERR_BAD_INT`; the largest valid i64 still parses. The shared env-int path
  inherits the rejection.
- **Unguarded getter index** (audit SEC-2) — `cmdit_get_bool/int/str/enum` and `cmdit_seen`
  did not bounds-check `idx`, so an ignored `-1` registration return + `cmdit_get_int(h, -1)`
  was a 112-byte OOB read. A shared `_cmdit_idx_ok` guard now returns the type's zero
  sentinel (mirroring `cmdit_positional` / `cmdit_repeat_get`).

### Changed
- Entry struct 104 → 112 B (metavar at +104; append-only, old offsets untouched).

### Hardening (the freeze gates)
- **Tests 157 → 230.** New coverage for the entire output/renderer surface
  (`cmdit_print_error` every `CmditErr` branch, `cmdit_help`, `cmdit_verbs_help`,
  `cmdit_version`), negative-int parse, dispatch pre-verb-unknown + dispatch
  missing-value, enum prefix rejection, `CMDIT_FLAGS_MAX`/`CMDIT_VERB_MAX` cap returns,
  getter idx guards, the overflow fix, and the C5/C3/C1 additions.
- **Benchmarks** captured in [`docs/benchmarks.md`](docs/benchmarks.md) (harness
  `tests/cmdit.bcyr`): handle construction ~13.8 µs (dominated by the entry memzero),
  parse loop sub-µs, single-pass dispatch verified (after-verb ≈ before-verb, not 2×).
- **Security audit** pass in [`docs/audit/2026-06-25-audit.md`](docs/audit/2026-06-25-audit.md):
  no attacker-reachable memory-safety defect on any argv/env path; SEC-1/SEC-2 fixed.
- **Doc drift fixed** — README test count (26→230), the `src/cmdit.cyr` ctx-size header
  comment (104→160 B), `getting-started.md` (`src/main.cyr`→`src/cmdit.cyr`), demo version
  strings (smoke 0.2.0→1.0.0), and this CHANGELOG's dates + the 0.3.0 `88→157` baseline.

### Scope (deferred to a future release — confirmed by the re-survey, not dropped)
- Cross-flag mutex / required-if sugar, duplicate-flag rejection, optional-value
  (getopt `optional_argument`) tri-state, stop-at-first-positional, `--no-foo` negation,
  bundled shorts `-abc`, attached short value `-xvalue`, named-positional help synopsis,
  multi-source env/config cascades. Each resolved to caller-side / already-expressible /
  low-prevalence; none threatens correctness or the frozen surface.

## [0.3.0] - 2026-06-25

**Verb / subcommand dispatch (M3).** Identify-and-return dispatch (mirroring
`cmdit_get_enum`, not handler fn-ptrs — consumer handler signatures are
irreconcilable and the codebase never uses fn-ptrs), grounded in a site-by-site
re-survey of the ~9 subcommand tools (ark/sit/takumi/hapi/phylax/agnova/hadara/
hoosh/cyim) + the kriya multiplexer. Append-only: the entry struct is unchanged
(verbs are a separate lazy-allocated table), ctx grows 104→160 B, error taxonomy
grows. The pure `cmdit_parse_argv` core is untouched.

### Added
- **`cmdit_verb`** / **`cmdit_verb_alias`** — register subcommands (and aliases)
  in a lazily-allocated verb table; `cmdit_verb` returns the verb id, the
  identify-and-return key.
- **`cmdit_dispatch_argv`** — a NEW pure sibling of `cmdit_parse_argv`: one pass
  that applies registered global flags **before OR after** the verb (the fix for
  ark's double-scan), matches the first non-flag token against the verb table, and
  compacts a globals-removed **remainder slice** (`argv[0]` = verb name) for the
  caller to re-parse. **`cmdit_dispatch`** = materialize + dispatch + finalize the
  global scope. **`cmdit_verb_matched`** returns the matched canonical id (-1 = no
  verb token, distinct from an UNKNOWN_VERB error). **`cmdit_verb_argc`** /
  **`cmdit_verb_argv`** expose the remainder.
- **`cmdit_parse_verb`** — sugar: re-parse the remainder in a per-verb handle +
  finalize. **Nested sub-verbs** are just re-entry on the remainder (no in-core
  tree — the roadmap bent here; depth-2 max in real code).
- **`cmdit_basename`** — interior-pointer basename of `argv[0]` for the busybox
  multiplexer (with the existing `cmdit_raw_argv` escape), caller-side and opt-in.
- **`cmdit_require_positionals`** — the deferred-from-0.2.0 minimum positional-count
  gate (`CMDIT_ERR_MISSING_POSITIONAL`); works on the global or any verb handle.
  Subsumes cyim's `if (r_npos < N)` guards.
- **`cmdit_verbs_help`** — generated `Commands:` listing (alias rows skipped);
  appended by `cmdit_help` and printed by `cmdit_print_error` on an unknown verb,
  so the command list never drifts from registration.
- **Help/version deferral** — a `--help`/`--version` *after* the verb defers into
  the verb scope (`tool build --help` → verb help), while *before* the verb it
  fires global help (`tool --help` → global help + command list).
- **Error taxonomy** — `CMDIT_ERR_UNKNOWN_VERB=8` (`<prog>: unknown command: <x>`
  + command list) / `CMDIT_ERR_MISSING_POSITIONAL=9` (`missing required argument`),
  both exit 2.
- **Tests** — `tests/cmdit.tcyr` grown 88 → **157 assertions** over verb
  match/alias/unknown, global-before/after, deferral, `--` handling, remainder
  re-parse, nested re-entry, `require_positionals`, and `cmdit_basename`. Demo:
  `programs/verbs.cyr`.

### Changed
- Context struct 104 → 160 B (append-only; entry struct unchanged at 104 B).

### Scope (not subsumed — stays caller-side)
- In-core nested-verb trees (use re-entry on the remainder).
- The argv[0] multiplexer *policy* (which basenames map to which verbs) — cmdit
  ships only `cmdit_basename`.
- Handler invocation / signatures — cmdit returns an id + slice, never calls a handler.
- Interactive REPL / slash-command DSLs (agora/agnoshi/thoth).

## [0.2.0] - 2026-06-25

**Flag modifiers (M2).** Five append-only parse-loop primitives that subsume the
patterns the 2026-06-25 ecosystem survey found hand-rolled across the consumer
repos (re-verified site-by-site against attn11/owl/agnova/cyim/phylax/whirl/
agora/bannermanor/chakshu/ark before landing). Types stay byte-compatible
(`CMDIT_BOOL/INT/STR` = {0,1,2}); the entry struct grows 48→104 B and the error
taxonomy grows — both append-only, old offsets/codes untouched.

### Added
- **`cmdit_enum`** — choice flags. `'|'`-delimited `choices` (e.g.
  `"auto|always|never"`); the parsed value must be a member or parse fails with
  `CMDIT_ERR_BAD_ENUM`. **`cmdit_get_enum`** returns the 0-based matched index
  (the dispatch every consumer re-derived; attn11 `cfg_*` 0..3, owl 0..2, cyim
  1..6); `cmdit_get_str` still returns the raw choice. Subsumes the nested-streq
  validation in attn11/owl/agnova/cyim/phylax.
- **`cmdit_repeat`** — repeatable flags. Each `--flag value` occurrence appends
  to a per-flag list (cap `CMDIT_REPEAT_MAX` = 64, silently capped like
  positionals); **`cmdit_repeat_count`** / **`cmdit_repeat_get`** read it, and
  `cmdit_get_str` gives a last-wins scalar view. Subsumes whirl's `-H` accumulator.
- **`cmdit_required`** — required flags (post-registration marker, any type incl.
  bool). A new **SEEN** bit (set by CLI parse or env) distinguishes absent from
  explicit-false, so confirmation bools like agnova `--i-mean-it` work; absent at
  finalize → `CMDIT_ERR_REQUIRED_MISSING`. **`cmdit_seen`** exposes the bit.
  Subsumes agora `--handle` / agnova `--device`/`--user`/`--i-mean-it` sentinel checks.
- **`cmdit_range`** — inclusive int bounds `[min,max]`, **rejected** (not clamped)
  → `CMDIT_ERR_OUT_OF_RANGE`. Subsumes bannermanor `--width`/`--pad`, attn11
  `--layers`/`--mtp`/`--bpe` caps.
- **`cmdit_env`** — environment-variable fallback (post-registration marker). At
  finalize, an un-SEEN flag draws from `getenv(env_name)`: bool = presence
  (`NO_COLOR`-style), int/str/enum = value (set-but-empty ignored). CLI always
  wins; env can satisfy required; bad env values are **non-fatal** (fall through
  to the default). Subsumes single flag-bound vars (`NO_COLOR`, `ARK_CONFIG`).
- **`cmdit_finalize`** — the impure post-parse stage (env fallback then required
  check). Kept OUT of `cmdit_parse_argv` so the pure synthetic-argv core stays
  `/proc`- and env-free; **`cmdit_parse`** now chains `parse_argv → finalize` on a
  clean parse (help/version still short-circuit first).
- **Error taxonomy** — `CMDIT_ERR_REQUIRED_MISSING=5` / `BAD_ENUM=6` /
  `OUT_OF_RANGE=7` (0.3.0 reserves `UNKNOWN_VERB=8`).
- **Flag-named errors** — `cmdit_print_error` now names the offending flag
  (`<prog>: --color: invalid value (expected auto|always|never)`) for
  value/modifier errors via a new ctx `err_entry`, matching the near-universal
  convention in agora/bannermanor/cyim/attn11. `UNKNOWN` stays generic; no `fmt`
  dependency (numeric range bounds are not rendered). **`cmdit_err_flag`** exposes
  the offending flag's index for consumer-rendered messages.
- **Generated help** — enum renders `<a|b|c>`, repeat appends `...`, required
  appends `(required)` (all string-only, no `fmt`).
- **Tests** — `tests/cmdit.tcyr` grown 26 → **88 assertions** over the modifiers,
  including a purity test asserting `cmdit_parse_argv` ignores a set env var
  (env only applies at `cmdit_finalize`).

### Changed
- Entry struct 48 → 104 B; context 96 → 104 B (both append-only). `CMDIT_FLAGS_MAX`
  unchanged at 64.
- **New consumer dep**: stdlib `"io"` (for `getenv`) is now referenced
  unconditionally by `cmdit_parse` (via `cmdit_finalize`). Pure callers that drive
  `cmdit_parse_argv` directly and never call `cmdit_finalize` link `getenv` but
  never invoke it.

### Scope (not subsumed — stays caller-side)
- Dynamic / large enum lists (owl theme/lang: 45 langs + runtime user themes).
- Cross-field / modulo ranges (attn11 `--rope-dim<=d_model`, `rope_dim%2`,
  `--expert-topk<=experts`).
- Pure-env cascades with no associated flag (`OWL_PAGER>PAGER`, config-path
  builders, `USER>euid`, `ARK_CONFIG`'s privilege gate).
- Positional-count/`required`-positional validation — candidate for 0.3.0.

## [0.1.0] - 2026-06-25

**The extraction cut.** cmdit is the stdlib flag parser (`lib/flags.cyr`)
productized as a standalone distlib + the universal boilerplate every structured
consumer hand-rolled. Surfaced by the 2026-06-25 ecosystem CLI review (no
dedicated CLI lib existed; ~40 binaries hand-roll `argc`/`argv`).

### Added
- **getopt-long parser core** — forked from cyrius stdlib `lib/flags.cyr`
  (byte-compatible types `CMDIT_BOOL/INT/STR` and error codes), renamed to the
  `cmdit_*` surface. `cmdit_new` / `cmdit_bool` / `cmdit_int` / `cmdit_str` /
  `cmdit_parse_argv` / `cmdit_get_*` / `cmdit_positional*` / `cmdit_error`.
  Syntax: `--name`, `--name=value`, `--name value`, `-x`, `-x value`, `--`
  terminator, lone `-` positional.
- **`cmdit_argv`** — the argc/argv → contiguous NULL-terminated `cstr*` materialize
  bridge (calls `args_init` once, cached on the ctx). Absorbs the
  `build_argv_array` / `_materialize_argv` boilerplate found in kii/yo/agnova/dig/kriya.
- **`cmdit_parse`** — convenience: materialize then parse the process argv.
- **Auto `--help`/`-h` + `--version`/`-V`** — registered by `cmdit_new`;
  `cmdit_parse*` short-circuit to `CMDIT_HELP` / `CMDIT_VERSION`.
- **`cmdit_help`** — generated `Usage:` + flag list (kills hand-written-help drift);
  **`cmdit_version`** — centralized machine-readable version line; **`cmdit_print_error`**
  — standard `<prog>: <message>` stderr decode.
- **`CMDIT_EXIT_OK/RUN/USAGE`** — the `0`/`1`/`2` AGNOS userland exit convention as
  named constants.
- **`cmdit_raw_argv` / `cmdit_raw_argc`** — escape hatch for non-getopt grammars.
- `FLAGS_MAX` raised 32 → 64 (attn11 has ~35 flags, exceeding the stdlib cap).
- **Tests** — `tests/cmdit.tcyr`, 26/26 over the pure `cmdit_parse_argv` core
  (bool/int/str/positional, `=value`, `--` terminator, lone `-`, unknown / missing-value
  / bad-int / bundled errors, auto help/version short-circuit, default preservation).
- **Smoke/demo** — `programs/smoke.cyr` exercises real-argv parsing end to end.

### Deferred
- 0.2.0: `cmdit_enum` / `cmdit_repeat` / `cmdit_required` / `cmdit_range` / `cmdit_env`.
- 0.3.0: verb / subcommand dispatch.
- Out of scope: bundled shorts `-abc`, attached short value `-xvalue`, count `-vvv`,
  `--no-foo` negation, interactive/REPL command DSLs.

Toolchain pin: cyrius 6.2.44.
