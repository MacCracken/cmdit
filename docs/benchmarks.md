# cmdit benchmarks

> Captured for **1.3.7** (2026-10-14) on toolchain **6.7.9**, on an idle host (load average
> ~1.3). Harness:
> [`tests/cmdit.bcyr`](../tests/cmdit.bcyr) (stdlib `lib/bench.cyr`). Reproduce with
> `cyrius build tests/cmdit.bcyr build/cmdit-bench && ./build/cmdit-bench`.
> The v1.0 capture this replaces is accounted for in [the bisect](#where-the-v10-numbers-went) below.

## Method

Each line is one `bench_batch_start → tight loop → bench_batch_stop(n)` pair
reporting the per-iteration average. The harness measures its own clock-read floor
and subtracts it per batch, so over a batch of `n` iterations that correction is
negligible. Two iteration regimes, because `cmdit_new` allocates ~7.8 KB into the
**free-less bump allocator**:

- **Alloc-heavy** benches (`cmdit_new_floor`, `setup_baseline`, `parse_argv_mixed`,
  `repeat_accumulate_32`) build a fresh handle each iteration, so they run a bounded
  **4096 iters** (~32 MB of bump memory).
- **Pure-leaf** benches (`get_enum_rewalk`, `dispatch_*`, `flag_find_long`) reuse one
  handle and run **200k–1M iters**.

Costs are isolated by **subtraction**, since `cmdit_new` is shared:

| Quantity | Formula |
|---|---|
| flag-registration cost | `setup_baseline − cmdit_new_floor` |
| parse-loop cost | `parse_argv_mixed − setup_baseline` |
| single-pass dispatch proof | `dispatch_after_verb ≈ dispatch_before_verb` (not 2×) |

**Every number below is the median of 11 runs, interleaved with the other variants
being compared**, not a single "representative" run. (The 1.3.2 capture and the bisect below
used 7.) Interleaving spreads slow drift
(thermal, frequency, background load) across all variants instead of letting it land
on whichever ran last.

## Results (1.3.7)

Host: AMD Ryzen 7 5800H, x86_64 Linux 7.2, `powersave` governor. Interleaved with the 1.3.6
source, which is the right-hand column.

| Benchmark | median | min–max | 1.3.6 | iters | what it measures |
|---|---|---|---|---|---|
| `cmdit_new_floor` | **2.05 µs** | 2.00–2.15 | 2.02 | 4096 | handle alloc + the 7680-byte entry-table zero + 2 auto registrations |
| `setup_baseline` | 1.99 µs | 1.89–2.10 | 2.04 | 4096 | `cmdit_new` + register bool/int/str/enum (no parse) |
| `parse_argv_mixed` | 2.42 µs | 2.35–2.45 | 2.52 | 4096 | the above + parse a 9-token argv (bool, short-int, `=value` str, enum, 2 positionals) |
| `get_enum_rewalk` | **95 ns** | 93–96 | 98 | 1 000 000 | one `cmdit_get_enum` choice-set re-walk (worst case = last of 3) |
| `dispatch_before_verb` | **177 ns** | 171–201 | 176 | 200 000 | single-pass dispatch, global flag **before** the verb |
| `dispatch_after_verb` | **248 ns** | 242–253 | 246 | 200 000 | single-pass dispatch, global flag **after** the verb |
| `repeat_accumulate_32` | 3.39 µs | 3.27–3.50 | 3.40 | 4096 | `cmdit_new` + 32 `-H` occurrences (lazy list alloc + 32 pushes) |
| `verb_new_floor` | **3.36 µs** | 3.23–3.48 | 3.13 | 4096 | `cmdit_new` + the first `cmdit_verb` (allocates and zeroes the 6144-byte verb table) |
| `flag_find_long` | **80 ns** | 78–81 | — | 1 000 000 | `cmdit_flag_find_long` on the last of six flags, from a `--name=value` span (new in 1.3.7) |

**Against 1.3.6**, every row is within ±4% except `verb_new_floor` (+7%). 1.3.7 grew the verb
row 40 → 48 B, so the table is 6,144 B, and its share (`verb_new_floor − cmdit_new_floor`) went
1.11 → 1.31 µs, the 20% the table grew. 1.3.6 itself was never cleanly captured (its release
run was on a saturated host); this run measures it as within noise of 1.3.7 everywhere else.

**Against 1.3.2** (the previous capture, median of 7): construction, registration, parsing,
enum and dispatch are within ±3%, `repeat_accumulate_32` is +7%, and `verb_new_floor` is
2.83 → 3.36 µs, the sum of 1.3.5's and 1.3.7's row growth (4,096 → 6,144 B). 1.3.6 grew the
flag entry 112 → 120 B, so `cmdit_new` zeroes 7,680 B instead of 7,168; that is not visible
at this resolution.

## What the numbers say

- **Handle construction is now ~2 µs, down from ~11.5 µs.** 1.3.0 zeroes the
  entry table (64 × 112 = 7168 bytes then, 64 × 120 = 7680 since 1.3.6) with 896 `store64`s instead of 7168 `store8`s. That
  one loop *was* the cost of `cmdit_new`: the interleaved A/B on the same build reads
  **11.56 → 1.94 µs (−83%)**, and every alloc-heavy bench drops by the same ~9.5 µs.
  The v1.0 doc predicted "up to ~8×"; the measured gain is ~6×, because the handle
  alloc and the two auto registrations are now what remains.
- **Registration and parsing are nearly free.** `setup_baseline − cmdit_new_floor` is
  inside run-to-run noise. The parse loop over a representative 9-token argv costs
  **~0.43 µs**. With construction no longer swamping it, that is now ~18% of a full
  `cmdit_new` + register + parse, not ~3%.
- **A verb tool's construction is ~3.4 µs, down from ~8.4 µs at 1.3.1.** The first
  `cmdit_verb` zeroes the verb table (128 × 32 = 4096 bytes at 1.3.2; 128 × 48 = 6144 since
  1.3.7), and that loop was still byte-at-a-time after 1.3.0 fixed the entry table's. 1.3.2
  made it a word loop. `verb_new_floor − cmdit_new_floor ≈ 1.3 µs` is now the whole cost of
  the verb table, and it scales with the table's size, not with how many verbs a tool has. The dispatch benches build their handle outside the
  timed loop, which is why nothing measured this until 1.3.2 added the bench.
- **A repeat push is ~42 ns** (`repeat_accumulate_32 − cmdit_new_floor ≈ 1.34 µs` over
  32 pushes, including the lazy list alloc).
- **Enum dispatch is ~95 ns** for a 3-choice set, re-walked per `cmdit_get_enum` call.
  It is a linear scan of the choices cstr, not O(1), so the source header's advice to
  **cache the index after parse** rather than re-call in a loop still holds.
- **The single-pass dispatch design holds.** `dispatch_after_verb` is **1.40×**
  `dispatch_before_verb`, not the ~2× a double scan would cost (ark's original bug).
  The after-verb path pays an extra `_cmdit_disp_find` + auto-flag check. The 1.0.0
  source measures 1.48× on this host (bisect below); the v1.0 table's 1.3× came from a
  single run.
- **A flag lookup is ~80 ns** over six flags, the linear scan the parser does for every
  `--name` token. kriya's expander calls it once per flag-shaped token.

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
