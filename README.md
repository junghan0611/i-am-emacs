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

**Early stage.** Build infrastructure (NixOS flake.nix + build.zig)
is being set up. No C code has been modified yet.

Based on `emacs-30.2` tag.

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

- [Ghostty(Zig+libxev+flake.nix)](https://github.com/ghostty-org/ghostty)
- [Neomacs](https://github.com/eval-exec/neomacs)
- [Emacs-ng](https://github.com/emacs-ng/emacs-ng)
- [EAF](https://github.com/emacs-eaf/emacs-application-framework)
- [GLG's Digital Garden](https://notes.junghanacs.com)
