# 002 — Every write goes through an output session

**Since:** 1.4.0.

Each public renderer opens an output session on its stream. The renderers are
`cmdit_help_fd`, `cmdit_help_flags_fd`, `cmdit_verbs_help_fd`, `cmdit_version`,
`cmdit_print_error` and `cmdit_completions`, plus the fd-2 wrappers through them. Every
fragment goes through `_cmdit_write`, and the session flushes before the renderer returns,
so one call is normally one write(2): one for `--help`, one per error line, one per
completion script under 4 KB.

What follows from that:

- **Never call `syscall(1, ...)` directly in the library.** Go through `_cmdit_write`, or
  `_cmdit_fprint` / `_cmdit_oprint` / `_cmdit_eprint`, which call it. A direct write inside
  a session lands *before* text the session is still holding, out of order.
- **A new public renderer must open and close a session:** `var prev =
  _cmdit_out_begin(fd);` at the start, and `_cmdit_out_end(prev);` on every return path.
  Opening one is what makes its output a single write. A path that returns without closing
  leaves the next renderer's output buffered behind it; the suite's `ob_idle()` checks
  catch that for the existing renderers.
- **Sessions nest.** `cmdit_help_fd` renders through `cmdit_help_flags_fd` and
  `cmdit_verbs_help_fd`, and `cmdit_print_error` through `cmdit_verbs_help`. An inner session
  on the same stream adds to the outer one's buffer. An inner session on another stream
  flushes the outer buffer first and gives the stream back when it closes.
- **The order against the caller's own writes is unchanged.** Nothing is held past the
  renderer's return. A piece longer than the 4096-byte buffer goes out directly, after what
  was buffered.
- **The final write retries a short write and EINTR**, and stops at any other error, as the
  ignored write results did before 1.4.0.
