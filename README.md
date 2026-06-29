# lumen-dbg

A graphical diagnostic probe for the [lumen](https://github.com/AspisOS/lumen)
compositor on [AspisOS](https://github.com/AspisOS/AspisOS) — a capability-based,
no-ambient-authority x86-64 operating system built on the from-scratch
[Aegis](https://github.com/AspisOS/Aegis) kernel.

lumen-dbg ships the `lumen-probe` diagnostic: a standalone binary (installed at
`/bin/lumen-probe`) that waits for the compositor to come up, performs a
`lumen_connect()` handshake, opens a probe window, and reports the result to the
console. It exercises the lumen client path end-to-end and backs the compositor's
connection and proxy-teardown regression tests. It is distributed as a
[herald](https://github.com/AspisOS/AspisOS) system package.

## Status

lumen-dbg is an early-stage, intentionally minimal diagnostic. Today it is a
single test probe that runs only on test/diagnostic boots and underpins two
regression tests; it is not a user-facing tool and never appears on a production
desktop. It is expected to grow as AspisOS's graphical test and diagnostic needs
expand — additional probes, more of the window protocol exercised, richer
reporting — so treat the current single-binary shape as a starting point rather
than a finished surface.

## Where it fits in AspisOS

AspisOS is decomposed into independent repositories:

| Repo | Role |
|------|------|
| `AspisOS/Aegis` | The kernel: `AF_UNIX` sockets, the framebuffer, the capability model, the syscalls the compositor and its clients run on. |
| `AspisOS/lumen` | The compositor/display server. Owns the screen; clients connect to `/run/lumen.sock` for a window. |
| `AspisOS/glyph` | The GUI toolkit. Provides the client side of lumen's window protocol (`lumen_client.h`, `lumen_proto.h`) the probe links against. |
| `AspisOS/lumen-dbg` | **This repo.** A minimal lumen client used to validate the compositor connection and window lifecycle. |

The probe is just another lumen client. It holds no display authority of its own;
it connects to lumen over `AF_UNIX` and asks for a window like any GUI app.

## What it does

`src/main.c` is a single-file lumen client built for verification rather than
use:

- **Test-only gate.** It reads `/proc/cmdline` and runs **only** when the
  `bastion_autologin` test hook is present (i.e. on `aegis-installer-test.iso`).
  On a production graphical boot it returns immediately, so users never see a
  probe window.
- **Wait for the compositor.** It sleeps briefly so [bastion](https://github.com/AspisOS/bastion)
  can authenticate and bring lumen up, then retries `lumen_connect()` up to 300
  times at 100 ms intervals (~30 s). Aegis `AF_UNIX` binds live in an in-kernel
  name table with no filesystem entry, so there is nothing to `stat()` for
  readiness — `ECONNREFUSED` is treated as "listener not up yet" and retried;
  any other error is surfaced and aborts.
- **Exercise the window path.** On a successful connect it creates a 200×100
  window (`lumen_window_create`), fills the shared buffer with a solid color, and
  presents one frame — validating the `memfd` handoff and present round-trip. It
  holds the window visible for ~2 s (long enough for a test to capture a screen
  dump) before destroying it and disconnecting.
- **Report.** Every step is logged to `stderr` and `/dev/console` with `[PROBE]`
  lines (`PASS` / `FAIL` plus the connect result and attempt count), which the
  test harness parses. It backs the `lumen_connect_probe_test` (the AF_UNIX
  handshake) and `lumen_proxy_uaf_test` (proxy-window teardown) regression tests.

## Capabilities

lumen-dbg ships **no** cap policy — there is no `pkg/etc/aegis/caps.d/` entry — so
the probe runs under only the baseline `service` profile. It needs nothing more:
it is a plain lumen client that connects over the compositor's `AF_UNIX` socket
and holds no ambient authority of its own.

The herald package id (`lumen-dbg`) differs from the binary it installs
(`/bin/lumen-probe`); that id/exec-name divergence, together with installing a
`/bin` binary and a vigil service, is why it is a `class=system` package:
first-party and signature-trusted, installed verbatim by herald.

## Building

lumen-dbg builds with a musl cross-compiler against a pinned
[glyph](https://github.com/AspisOS/glyph) toolkit artifact, then packs a signed
herald package.

```sh
make MUSL_CC=/path/to/musl-gcc HERALD_KEY=/path/to/signing.key
```

The `Makefile` fetches the toolkit, compiles `src/*.c` against it, and packs the
`.hpkg`:

- `GLYPH_VERSION` pins the glyph release fetched by `tools/fetch-glyph.sh` (local
  `vendor/` cache first, otherwise the GitHub release). Links
  `-lcitadel -laudio -lauth -lglyph` — a static archive only contributes the
  objects actually referenced, so the probe pulls in just the lumen client code.
- `MUSL_CC` is the musl cross-compiler (the only toolchain assumption; defaults
  to `musl-gcc` on `PATH`). Point it at an Aegis-native `cc` to build on-device
  in the future.
- `HERALD_KEY` signs the package (ECDSA P-256).

Output: `lumen-dbg.hpkg` (a `class=system` herald package) plus its detached
`lumen-dbg.hpkg.sig`.

## Package payload

`lumen-dbg.hpkg` is a manifest-first, uncompressed POSIX `ustar` archive with a
detached ECDSA-P256/SHA-256 signature. herald installs the payload tree verbatim
at these paths:

```
/bin/lumen-probe                       the diagnostic probe binary (stripped)
/etc/vigil/services/lumen-probe/       vigil service: mode=graphical, oneshot
```

The vigil service runs the probe once (`oneshot`) as `root` on a graphical boot;
the probe's own cmdline gate means it does real work only on a test ISO.

## Repository layout

```
src/main.c        the lumen-probe diagnostic (connect, window, report)
pkg/              install-tree skeleton shipped verbatim (vigil service)
tools/
  fetch-glyph.sh  fetch + unpack the pinned glyph toolkit artifact
  pack.sh         build + sign lumen-dbg.hpkg
Makefile          fetch toolkit → build → pack
VERSION           this component's version
GLYPH_VERSION     the pinned glyph toolkit version it builds against
```

## Dependencies

`depends=lumen` — the probe is a lumen client and connects to the compositor, so
installing it pulls [lumen](https://github.com/AspisOS/lumen).
