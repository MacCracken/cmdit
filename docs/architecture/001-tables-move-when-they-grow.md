# 001 — The flag and verb tables move when they grow

**Since:** 1.4.0.

`cmdit_new` allocates room for 16 flags, and the first `cmdit_verb` room for 16 verb rows.
Each table doubles when it fills, up to its cap (64 flags, 128 verb rows). A growth
allocates a new table, copies the used part, zeroes the rest, and points the handle at the
new one. The old table is left behind; the stdlib allocator never frees.

What follows from that:

- **Address entries and rows by index, never by a pointer kept across a registration.**
  `_cmdit_entry(h, i)` and `_cmdit_verb_row(h, i)` compute the address from the handle each
  time. A pointer taken before a `cmdit_bool` / `cmdit_verb` call can point into the old
  table afterwards. The old table still holds the old bytes, so the bug shows up as stale
  data, not as a crash.
- **The handle keeps exactly one entry pointer: `err_entry` (ctx +96).** `_cmdit_add` moves
  it with the table, so `cmdit_err_flag` still answers correctly when flags are registered
  after a failed parse. A new field that stores an entry or row pointer must be moved the
  same way, or store an index instead.
- **A new entry starts at zero only in its modifier fields because the grown tail is
  zeroed.** `_cmdit_add` writes an entry's first six fields; attributes, choices, env name,
  range, repeat list, metavar and second long name rely on the zeroed tail. (The stdlib's
  `alloc` also returns zeroed memory, so no test can tell the zeroing is there.) A verb row's
  fields are all written by `_cmdit_verb_add`.
- **The caps did not change.** Registering past 64 flags or 128 verb rows still returns -1.
  The public `CMDIT_FLAGS_MAX` / `CMDIT_VERB_MAX` are informational, as they have been since
  1.3.2.
