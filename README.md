# I Am Emacs

Incremental modernization of the GNU Emacs C core with Zig.

No GPU glitter. Looks the same. Never freezes.

## Why

Emacs freezes. Everyone who uses it knows this. A long-running `org-agenda` rebuild blocks keyboard input. GC pauses stall the display. A slow network operation locks the entire editor.

These problems live in the C core — a single-threaded event loop written decades ago. Elisp packages cannot fix what the C layer forces: blocking I/O, stop-the-world GC, and a command loop that cannot yield.

## What

This project replaces performance-critical internals of Emacs 30.2
with Zig, one function at a time:

- **I/O thread separation** — move `pselect()` blocking out of the
  main thread so keyboard input is never starved
- **Incremental GC** — replace stop-the-world mark-sweep with sliced
  incremental marking, keeping GC pauses under 10ms
- **Command loop yield points** — insert checkpoints in `Feval` /
  `Fprogn` so `C-g` responds instantly even during heavy computation

## How

- **Zero API changes** — C function signatures are untouched; only
  internal implementations are replaced with Zig
- **Elisp untouched** — all Elisp code remains exactly as-is; 100%
  package compatibility
- **Test-first** — establish comprehensive test coverage for existing
  C logic before converting anything
- **`make check` always passes** — the original Emacs 30.2 test suite
  is the invariant; it must never break

The architecture follows an event-driven pattern: a typed event union,
a blocking queue between threads, and pure function dispatch on the
main thread.

## Status

**Research phase.** No C code has been modified yet. Based on
`emacs-30.2` tag.

We have completed a thorough analysis of the actual codebase and
studied every prior attempt to solve these problems. This section
documents what we found and where we currently stand.

### What the codebase already has

The emacs-30.2 C core is more mature than expected:

- `thread_select()` already releases the global lock during
  `pselect()` — the infrastructure for I/O thread separation
  partially exists
- `mark_stack` (alloc.c:7104) already provides stack-based GC
  traversal — but incremental slicing requires write barriers,
  which is a fundamental change
- `maybe_quit()` exists in 40 files (111 call sites) — but cannot
  fire during system calls, GC, or C-internal loops

### Prior art: what failed and why

| Project | Approach | Outcome | Lesson |
|---------|----------|---------|--------|
| [Remacs](https://github.com/remacs/remacs) | C→Rust 1:1 rewrite | Dead (2021) | Language swap alone solves nothing |
| [emacs-ng](https://github.com/emacs-ng/emacs-ng) | C + Rust + JS (Deno) | Stalled (2025) | Scope creep kills; event loop integration is the hard problem |
| [Neomacs](https://github.com/eval-exec/neomacs) | Full Rust rewrite + GPU | Alpha (6 days old) | GPU is easy; eval/GC/bytecode is hard |
| [Rune](https://github.com/CeleritasCelery/rune) | Full Rust rewrite | Active | Closest architectural vision (CSP channels, per-thread GC) |

### The `feature/igc` factor

GNU Emacs has an official incremental GC branch (`feature/igc`) that
has been in development since 2024. It uses the
[MPS](https://www.ravenbrook.com/project/mps/) library with
SIGSEGV-based page protection as a write barrier. As of January 2026,
the merge process into master has begun. Benchmarks show 2.7x faster
GC-heavy workloads at the cost of 3-6x memory usage.

**This changes our scope significantly.** If `feature/igc` merges,
the GC problem is officially solved upstream. We should not duplicate
that work.

### Where the real gap is

After studying the upstream mailing lists (emacs-devel), we found:

- **GC** → `feature/igc` is handling this
- **Threading/GIL** → 10 years of discussion, no consensus, no
  solution (Tom Tromey's cooperative threads shipped in Emacs 26,
  but Chris Wellons found "many data races, completely
  untrustworthy")
- **I/O thread separation** → **nobody is working on this**

Moving `pselect()` blocking out of the main thread into a dedicated
I/O thread with event queue communication — this is the gap that no
project has filled. This is where I Am Emacs can contribute something
genuinely new.

### What we are thinking about

Emacs is an organism. Elisp's single-threaded logic is its
bloodstream — it must flow in one direction, never backflow. The
question is not "how to add threads" but "how to keep the bloodstream
healthy while the organism grows."

The event-driven pattern (typed event union + blocking queue + pure
dispatch) respects this constraint: I/O threads only *produce* events,
the main thread only *consumes* them, and Elisp never runs outside
the main thread.

We are still deliberating on the approach. The architecture design
must come before the code.

## Build

Requires NixOS with flakes enabled.

```bash
nix build    # (planned — flake.nix not yet implemented)
```

Fallback (standard Emacs build):

```bash
./autogen.sh && ./configure && make -j$(nproc)
make check
```

## License

GNU Emacs is free software under the GNU General Public License v3.0
or later. See [COPYING](COPYING) for details.

All new Zig code in this project is contributed under the same license.

## Related Links

### Prior art and references

- [Ghostty(Zig+libxev+flake.nix)](https://github.com/ghostty-org/ghostty)
- [Neomacs](https://github.com/eval-exec/neomacs)
- [Emacs-ng](https://github.com/emacs-ng/emacs-ng)
- [EAF](https://github.com/emacs-eaf/emacs-application-framework)
- [Rune — Experimental Emacs core in Rust](https://github.com/CeleritasCelery/rune)
- [Troy Hinckley: A Vision of a Multi-threaded Emacs](https://coredumped.dev/2022/05/19/a-vision-of-a-multi-threaded-emacs/)
- [feature/igc — Emacs incremental GC branch](https://cgit.git.savannah.gnu.org/cgit/emacs.git/tree/README-IGC?h=feature/igc)

### Upstream discussions

- [MPS/IGC proposal (emacs-devel, 2024-02)](https://lists.gnu.org/archive/html/emacs-devel/2024-02/msg00758.html)
- [Merging feature/igc (emacs-devel, 2026-01)](https://lists.gnu.org/archive/html/emacs-devel/2026-01/msg00613.html)
- [Concurrency, again (emacs-devel, 2016-10)](https://lists.gnu.org/archive/html/emacs-devel/2016-10/msg00209.html)
- [EmacsConf 2023: GC stats](https://emacsconf.org/2023/talks/gc/)
- [EmacsConf 2024: Rune](https://emacsconf.org/2024/talks/rust/)

### Maintainer

- [GLG's Digital Garden](https://notes.junghanacs.com)
