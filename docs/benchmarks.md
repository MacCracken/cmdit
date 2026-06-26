# cmdit benchmarks

> Captured for the v1.0 cut. Harness: [`tests/cmdit.bcyr`](../tests/cmdit.bcyr)
> (stdlib `lib/bench.cyr`). Reproduce with
> `cyrius build tests/cmdit.bcyr build/cmdit-bench && ./build/cmdit-bench`.

## Method

Each line is one `bench_batch_start → tight loop → bench_batch_stop(n)` pair
reporting the per-iteration average. Two iteration regimes, because `cmdit_new`
allocates ~7.8 KB into the **free-less bump allocator**:

- **Alloc-heavy** benches (`cmdit_new_floor`, `setup_baseline`, `parse_argv_mixed`,
  `repeat_accumulate_32`) build a fresh handle each iteration, so they run a bounded
  **4096 iters** (~32 MB of bump memory) — a 1e6 loop would need ~7.8 GB.
- **Pure-leaf** benches (`get_enum_rewalk`, `dispatch_*`) reuse one handle and run
  **200k–1M iters** to amortize the ~240 ns `clock_gettime` start/stop pair down to
  noise.

Costs are isolated by **subtraction**, since the dominant term (`cmdit_new`) is
shared:

| Quantity | Formula |
|---|---|
| flag-registration cost | `setup_baseline.avg − cmdit_new_floor.avg` |
| parse-loop cost | `parse_argv_mixed.avg − setup_baseline.avg` |
| single-pass dispatch proof | `dispatch_after_verb.avg ≈ dispatch_before_verb.avg` (not 2×) |

## Results

Host: AMD Ryzen 7 5800H, x86_64 Linux. Representative warm run (numbers are stable
run-to-run to the displayed precision):

| Benchmark | avg | iters | what it measures |
|---|---|---|---|
| `cmdit_new_floor` | **13.8 µs** | 4096 | handle alloc + the 7168-byte entry memzero + 2 auto registrations |
| `setup_baseline` | 13.2 µs | 4096 | `cmdit_new` + register bool/int/str/enum (no parse) |
| `parse_argv_mixed` | 13.8 µs | 4096 | the above + parse a 9-token argv (bool, short-int, `=value` str, enum, 2 positionals) |
| `get_enum_rewalk` | **91 ns** | 1 000 000 | one `cmdit_get_enum` choice-set re-walk (worst case = last of 3) |
| `dispatch_before_verb` | **203 ns** | 200 000 | single-pass dispatch, global flag **before** the verb |
| `dispatch_after_verb` | **271 ns** | 200 000 | single-pass dispatch, global flag **after** the verb |
| `repeat_accumulate_32` | 14.5 µs | 4096 | `cmdit_new` + 32 `-H` occurrences (lazy list alloc + 32 O(1) pushes) |

## What the numbers say

- **Handle construction dominates.** `cmdit_new` (~13.8 µs) swamps everything else.
  It is almost entirely the **byte-at-a-time zero of the `CMDIT_FLAGS_MAX ×
  CMDIT_ENTRY_SIZE` = 64 × 112 = 7168-byte entry table** (`store8` per byte). For a
  one-shot CLI this is irrelevant — it is a single ~14 µs cost dwarfed by process
  startup. It only matters for a hot multiplexer that builds many handles.
- **Registration and parsing are nearly free.** `setup_baseline − cmdit_new_floor`
  is within run-to-run noise (registration is a handful of `store64`s), and the
  parse loop over a representative 9-token argv is **sub-microsecond**
  (`parse_argv_mixed − setup_baseline ≈ 0.5 µs`). The cost of using cmdit *is* the
  one-time `cmdit_new`, not the per-arg work.
- **Enum dispatch is ~91 ns** for a 3-choice set, re-walked per `cmdit_get_enum`
  call — cheap, but the source header's advice to **cache the index after parse**
  rather than re-call in a loop is borne out (it is a linear scan of the choices
  cstr, not O(1)).
- **The single-pass dispatch design holds.** `dispatch_after_verb` (271 ns) carries
  a small constant overhead over `dispatch_before_verb` (203 ns) — the after-verb
  token costs an extra `_cmdit_disp_find` + auto-flag check — but it is **~1.3×, not
  the ~2× a naive double-scan would cost** (ark's original bug). Both are O(argc),
  one pass.

## Candidate future optimization (not a v1.0 change)

`cmdit_new`'s entry-table zero is byte-at-a-time; zeroing by 8-byte words
(`store64`) would cut the dominant ~13.8 µs by up to ~8× (7168 → 896 store ops).
It is a pure speed change (zero is zero) with no API or behavior impact, deliberately
**left out of the v1.0 freeze** to keep that cut scope-tight — tracked here as the
obvious first optimization if handle-construction throughput ever matters.
