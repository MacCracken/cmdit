# cmdit benchmarks

> Captured for **1.4.0** (2026-10-14) on toolchain **6.7.9**, at load average ~1.9 (one
> unrelated single-core compile was running). Harness:
> [`tests/cmdit.bcyr`](../tests/cmdit.bcyr) (stdlib `lib/bench.cyr`). Reproduce with
> `cyrius build tests/cmdit.bcyr build/cmdit-bench && ./build/cmdit-bench`.
> The v1.0 capture is accounted for in [the bisect](#where-the-v10-numbers-went) below.
>
> **1.4.1 (toolchain 6.7.18, 2026-10-28)** did not change the source. Its builds ran
> interleaved against 6.7.9 builds of the same source, median of 15 at load average ~0.6. All
> 13 benches were within ±4%, the code-layout band, so the 1.4.0 table stands. The per-bench
> numbers are in the CHANGELOG's 1.4.1 entry.

## Method

Each line is one `bench_batch_start → tight loop → bench_batch_stop(n)` pair
reporting the per-iteration average. The harness measures its own clock-read floor
and subtracts it per batch, so over a batch of `n` iterations that correction is
negligible. Three iteration regimes, because `cmdit_new` allocates ~3 KB (~8.9 KB before
1.4.0) from the **free-less bump allocator**:

- **Alloc-heavy** benches (`cmdit_new_floor`, `setup_baseline`, `parse_argv_mixed`,
  `repeat_accumulate_32`, `verb_new_floor`) build a fresh handle each iteration, so they run
  a bounded **4096 iters**.
- **Pure-leaf** benches (`get_enum_rewalk`, `dispatch_*`, `flag_find_long*`) reuse one
  handle and run **200k–1M iters**.
- **Output** benches (`help_fd_devnull`, `print_error_devnull`, `completions_devnull`, new in
  1.4.0) render the same handle to /dev/null **2k–20k times**, so they measure the writing,
  which before 1.4.0 was mostly write(2) calls.

Costs are isolated by **subtraction**, since `cmdit_new` is shared:

| Quantity | Formula |
|---|---|
| flag-registration cost | `setup_baseline − cmdit_new_floor` |
| parse-loop cost | `parse_argv_mixed − setup_baseline` |
| verb-table cost | `verb_new_floor − cmdit_new_floor` |
| single-pass dispatch proof | `dispatch_after_verb ≈ dispatch_before_verb` (not 2×) |

**Every number below is the median of 11 runs, interleaved with the other variants
being compared**, not a single "representative" run. (The 1.3.2 capture and the bisect below
used 7.) Interleaving spreads slow drift (thermal, frequency, background load) across all
variants instead of letting it land on whichever ran last.

## Results (1.4.0)

Host: AMD Ryzen 7 5800H, x86_64 Linux 7.2, `powersave` governor. Interleaved with the 1.3.7
source, which is the right-hand column.

| Benchmark | median | min–max | 1.3.7 | iters | what it measures |
|---|---|---|---|---|---|
| `cmdit_new_floor` | **0.76 µs** | 0.73–0.82 | 2.03 | 4096 | handle allocs + zeroing the first 16 entries + 2 auto registrations |
| `setup_baseline` | 0.75 µs | 0.73–0.78 | 1.99 | 4096 | `cmdit_new` + register bool/int/str/enum (no parse) |
| `parse_argv_mixed` | **1.16 µs** | 1.12–1.18 | 2.44 | 4096 | the above + parse a 9-token argv (bool, short-int, `=value` str, enum, 2 positionals) |
| `get_enum_rewalk` | 98 ns | 97–98 | 92 | 1 000 000 | one `cmdit_get_enum` choice-set re-walk (worst case = last of 3) |
| `dispatch_before_verb` | **167 ns** | 166–174 | 172 | 200 000 | single-pass dispatch, global flag **before** the verb |
| `dispatch_after_verb` | **177 ns** | 176–184 | 242 | 200 000 | single-pass dispatch, global flag **after** the verb |
| `repeat_accumulate_32` | 1.84 µs | 1.81–1.89 | 3.28 | 4096 | `cmdit_new` + 32 `-H` occurrences (lazy list alloc + 32 pushes) |
| `verb_new_floor` | **0.81 µs** | 0.79–0.84 | 3.29 | 4096 | `cmdit_new` + the first `cmdit_verb` (a 16-row verb table) |
| `flag_find_long` | 47 ns | 47–48 | 78 | 1 000 000 | `cmdit_flag_find_long` on the last of six flags, from a `--name=value` span |
| `flag_find_long_36` | 152 ns | 151–153 | 396 | 1 000 000 | the same over attn11's 36 real long names, for the last one (new in 1.4.0) |
| `help_fd_devnull` | **2.91 µs** | 2.88–2.96 | 33.78 | 20 000 | `cmdit_help_fd` for 7 flags and 3 verbs (new in 1.4.0) |
| `print_error_devnull` | 0.77 µs | 0.77–0.79 | 4.75 | 20 000 | one `cmdit_print_error` line, a bad enum value (new in 1.4.0) |
| `completions_devnull` | 14.9 µs | 14.7–15.2 | 161.1 | 2 000 | the bash, zsh and fish scripts for that handle (new in 1.4.0) |

**Against 1.3.7**, every row is faster except `get_enum_rewalk`, which is +6 ns although its
code did not change. 1.4.0's field-offset swap compiles byte-identically, and the functions on
its path are the same. Inserting unused padding functions ahead of `_cmdit_enum_find` moves
the same source between 95 and 101 ns, so the difference is where the code landed. The new
output benches were measured on both versions with the same harness; the 1.3.7 column is the
old per-fragment write(2) cost.

## What the numbers say

- **Handle construction is ~0.76 µs.** It was ~11.5 µs until 1.3.0 zeroed the entry table a
  word at a time (the bisect below), and ~2 µs until 1.4.0 stopped allocating and zeroing all
  64 entries (7,680 B) up front. The table now starts at 16 entries (1,920 B) and doubles as
  flags are registered. What remains is three allocations, that zeroing and the two auto
  registrations.
- **Registration is nearly free, and a parse is ~0.4 µs.** `setup_baseline − cmdit_new_floor`
  is inside run-to-run noise. The parse loop over a representative 9-token argv costs
  **~0.41 µs**, about a third of a full `cmdit_new` + register + parse.
- **A verb tool's construction is ~0.81 µs**, from ~8.4 µs at 1.3.1 and ~3.3 µs at 1.3.7. The
  verb table starts at 16 rows (768 B) and doubles, so `verb_new_floor − cmdit_new_floor` is
  ~0.05 µs, where zeroing all 128 rows cost ~1.3 µs.
- **A repeat push is ~34 ns** (`repeat_accumulate_32 − cmdit_new_floor ≈ 1.08 µs` over
  32 pushes, including the lazy list alloc).
- **Enum dispatch is ~98 ns** for a 3-choice set, re-walked per `cmdit_get_enum` call.
  It is a linear scan of the choices cstr, not O(1), so the source header's advice to
  **cache the index after parse** rather than re-call in a loop still holds.
- **A global flag costs the same before and after the verb.** `dispatch_after_verb` is
  **1.06×** `dispatch_before_verb`. It was 1.40× at 1.3.7, because the after-verb path
  looked each flag up twice (once to spot `--help`, once to apply it); 1.4.0 looks it up
  once. A double scan of argv, ark's original bug, would be ~2×.
- **A flag lookup is ~47 ns over six flags and ~152 ns over attn11's 36.** The scan compares a
  name's first byte before the full compare, so it makes a call only for the names that share
  it: 3 of attn11's 38 entries for `--vision-fd`.
- **Output is one write per call.** `--help` for a small tool is ~2.9 µs, an error line
  ~0.8 µs, and all three completion scripts ~15 µs. Before 1.4.0 each fragment was its own
  write(2): 93 for that `--help`, and one per byte of quoted help in the scripts.

## Where the v1.0 numbers went

The v1.0 capture (pin 6.2.44, 1.0.0 source) read `cmdit_new_floor` **13.8 µs** and
`dispatch_before_verb` **203 ns**. By 1.2.3 a fresh run read ~11.4 µs and ~153 ns, and
nobody could say why. To separate the variables, the **same harness** was compiled
against each tag's `src/cmdit.cyr` (current 6.7.9 compiler + lib), and the **1.0.0
source** was compiled under every installed toolchain. All variants ran interleaved,
median of 7 (ns):

| variant | `new` | `parse` | `disp<` | `disp>` | `rep32` |
|---|---|---|---|---|---|
| 1.0.0 source, toolchain 6.6.2 | 11 445 | 11 832 | 142 | 211 | 12 506 |
| 1.0.0 source, toolchain 6.6.20 | 11 510 | 11 773 | 145 | 213 | 12 457 |
| 1.0.0 source, toolchain 6.7.9 | 11 516 | 11 827 | 145 | 214 | 12 507 |
| 1.1.0 source | 11 502 | 11 960 | 146 | 213 | 12 536 |
| 1.2.0 source | 11 491 | 11 764 | 144 | 214 | 12 526 |
| 1.2.2 source | 11 503 | 11 837 | 162 | 231 | 12 303 |
| 1.2.4 source | 11 464 | 11 848 | 165 | 234 | 12 599 |
| 1.2.6 source | 11 486 | 11 755 | 164 | 233 | 12 691 |
| **1.3.0 source** | **1 971** | **2 339** | **165** | **238** | **3 014** |
| 1.3.2 source (separate run) | 2 023 | 2 421 | 172 | 243 | 3 181 |

What it settles:

- **No toolchain from 6.6.2 to 6.7.9 moved anything.** The identical 1.0.0 source
  measures the same under all three, within noise. (1.2.3 had already shown 6.4.78 →
  6.5.35 to be byte-identical codegen for cmdit.)
- **Source evolution 1.0.0 → 1.2.6 left construction and parsing flat.** The only
  real movement is **dispatch, +12% at 1.2.2** (145 → 162 ns before the verb, 214 → 231
  after). That is the cost of forwarding `--` into the remainder and of the
  `cmdit_verb_trailing_after` check, both correctness fixes from the stiva adoption.
- **The v1.0 table's extra ~2.3 µs and ~60 ns are not reproducible.** The 1.0.0 source
  on any installed toolchain measures 11.4–11.5 µs, not 13.8. The one span left
  unmeasured is toolchain 6.2.44 → 6.6.2, which is not installed (installing it would
  re-point the shared `~/.cyrius`). That span, plus capture conditions (the v1.0 table
  was a single run, not an interleaved median), is where the gap lives. The harness's
  floor subtraction is not it: over a 4096-iteration batch it is well under 1 ns per
  iteration.
