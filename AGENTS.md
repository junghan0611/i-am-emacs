# I Am Emacs — Agent Guidelines

## Principles

- **Baseline: emacs-30.2 tag on `main` branch**
- **C code only** — never touch Elisp files; information overload kills focus
- **100% API compatibility** — no changes to C function signatures; only internals replaced with Zig
- **Elisp stays on the main thread** — 100% package compatibility
- **Tests first** — establish sufficient test coverage before any conversion
- **Event-driven architecture** — Event union + BlockingQueue + pure function dispatch

## Invariants (never break these)

1. No Elisp calls from I/O threads — produce events only
2. No blocking I/O on the main thread — non-blocking + event wait
3. No Elisp execution during GC — resume after GC-complete event
4. No changes to lisp.h public interface
5. No changes to C function signatures — only internal implementation replaced with Zig
6. `make check` must always pass 100% against Emacs 30.2
7. No Elisp file modifications

## Language Policy

**Everything in this repository must be written in English.** This includes: code, comments, commit messages, documentation, test descriptions, and issue/PR titles. The maintainer communicates in Korean during interactive sessions, but all artifacts that enter the repository must be in English.

## Work Rules

- Only look at `src/*.c` and `src/*.h` — ignore everything under `lisp/`
- Read lisp.h for reference only — never modify it
- Write tests before converting any C code to Zig

## Target C Files

### Tier 1 — Core Bottlenecks

| File       | Lines  | Role                                | Tests (cases) |
|------------|--------|-------------------------------------|---------------|
| process.c  | 9,038  | I/O event loop, pselect() blocking  | 1,023 (37)    |
| alloc.c    | 8,358  | GC (stop-the-world mark-sweep)      | 62 (5)        |
| keyboard.c | 14,071 | command_loop, input/timer/redisplay | 79 (2)        |
| eval.c     | 4,553  | Elisp evaluator (Feval, Ffuncall)   | 374 (20)      |

### Tier 2 — Threading Infrastructure

| File        | Lines | Role                                 | Tests (cases) |
|-------------|-------|--------------------------------------|---------------|
| thread.c    | 1,221 | Existing thread support (cooperative + global lock) | 421 (33) |
| systhread.c | 544   | System thread primitives (pthread/win32/stub) | — |
| systhread.h | 113   | Thread interface (11 functions)      | —             |

### Tier 3 — Supporting Files

| File     | Lines | Role                                            |
|----------|-------|-------------------------------------------------|
| sysdep.c | 4,763 | System-dependent code                           |
| emacs.c  | 3,759 | Main entry point                                |
| lisp.h   | 5,939 | Core data structures (read-only, do not modify) |

### Explicitly Excluded

- **xdisp.c** (39,197 lines) — display engine, GPU-related, out of scope
- **src/*.el, lisp/** — all Elisp code, never touch
- **~100 other C files** — consider only after Tier 1-3 are stable

## Build

```bash
# Initial build (autotools, first time only)
./autogen.sh && ./configure && make -j$(nproc)

# Run tests
make check

# Run specific test
make -C test check SELECTOR='process-tests'
```

Build environment: **NixOS flake.nix + build.zig**

## Commit Conventions

- No "Co-Authored-By" or "Generated with Claude" lines
- Format: `component: description` (e.g., `process.c: extract I/O thread draft`)
- Reference beads issue ID in commit body (e.g., `Ref: bd-2ev`)

## Session Checklist

1. Verify branch: `git branch` — must be on `main`
2. Check open issues: `br ready`
3. Review current epic status

## Research Notes (2026-02-09)

This section records findings from analyzing the emacs-30.2 codebase
and studying upstream discussions. It serves as context for
architectural decisions.

### feature/igc changes our GC strategy

GNU Emacs has an official incremental GC on `feature/igc` (since
2024-02, merge started 2026-01). It uses MPS with SIGSEGV-based page
protection as a write barrier. If it merges, our Epic 2 (Incremental
GC) becomes redundant. **Current status: Epic 2 is blocked pending
IGC merge decision.**

### I/O thread separation is the unique gap

After reviewing emacs-devel archives:
- GC → `feature/igc` is handling this upstream
- Threading/GIL → 10 years of discussion, no solution
- I/O thread → **nobody is working on this**

`wait_reading_process_output()` (process.c:5276-8062) calls
`thread_select(pselect, ...)` which already releases `global_lock`
during the blocking call. The pattern exists — it needs to be
extended into a permanent I/O thread with event queue dispatch.

This is where the project can contribute something new.

### org-agenda bottleneck is not in process.c

File I/O goes through `fileio.c` → `emacs_read()`, not through
`pselect()`. The org-agenda freeze is caused by Elisp computation
(eval.c) and GC pauses (alloc.c), not by I/O blocking. The I/O
thread helps different scenarios: compile, LSP, TRAMP, shell.

### maybe_quit() has 111 call sites but structural gaps

`maybe_quit()` exists in 40 C files (111 sites). Bytecode checks
quit every 255th backward branch via `quitcounter`. But it cannot
fire during system calls, GC (`block_input` scope), or C-internal
loops. This is why C-g feels unresponsive on GUI frames.

### Key upstream references

- Troy Hinckley's Rune: CSP channels + per-thread GC + OS threads
  (closest to our event-driven architecture)
- Chris Wellons on Emacs 26 threads: "many data races, completely
  untrustworthy" — the GIL is load-bearing
- Philipp Stephani (2016): Go-style CSP without OS threads worked
  but was not merged

## References

- Ghostty (Zig + libxev + flake.nix): `~/repos/3rd/ghostty/`
- Event pattern reference: `~/repos/work/sks-hub-zig/`
- Rune (Rust Emacs, CSP architecture): https://github.com/CeleritasCelery/rune
- feature/igc (official incremental GC): `feature/igc` branch on Savannah
- Neomacs (GPU approach, reference only): https://github.com/eval-exec/neomacs
- Emacs-ng (stalled, lessons learned): https://github.com/emacs-ng/emacs-ng
