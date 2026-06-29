# lumen-dbg

A graphical diagnostic probe for the [lumen](https://github.com/AspisOS/lumen)
compositor on **AspisOS**, a capability-based, no-ambient-authority operating
system built on the from-scratch [Aegis](https://github.com/AspisOS/Aegis) kernel.

lumen-dbg ships the `lumen-probe` diagnostic: a standalone `/bin` binary
(installed as `/bin/lumen-probe`) that waits for the compositor's socket to
appear, performs a `lumen_connect()` handshake, and reports the result. It exists
to exercise the Lumen client path end-to-end on test/diagnostic boots. It is a
component of the Lumen desktop, distributed as a
[herald](https://github.com/AspisOS/AspisOS) package.

## Role in the system

- Started by the `vigil` service manager on a graphical boot as a `oneshot`
  service (`mode=graphical`) — it runs once rather than respawning.
- Test-gated: it reads `/proc/cmdline` and runs **only** when the
  `bastion_autologin` test hook is present (i.e. on `aegis-installer-test.iso`).
  On production graphical boots it exits silently, so users never see a probe
  window.
- When it does run it waits briefly (so `bastion` can authenticate and bring
  Lumen up), then retries `lumen_connect()` until the compositor's
  `/run/lumen.sock` accepts the connection (or it times out after ~30s), logging
  progress to `/dev/console`. It backs the `lumen_connect_probe_test` and
  `lumen_proxy_uaf_test` regression tests, which verify the AF_UNIX handshake and
  proxy-window teardown.

## Capabilities

lumen-dbg ships **no** cap policy file — it has no `pkg/etc/aegis/caps.d/`
entry — so it runs under only the baseline `service` policy. It needs no extra
capabilities: it is a plain Lumen client that connects over the compositor's
AF_UNIX socket and holds no ambient authority of its own.

Because its herald package id (`lumen-dbg`) differs from the binary it installs
(`/bin/lumen-probe`) and it installs a `/bin` binary plus a vigil service,
lumen-dbg is a `class=system` package: first-party and signature-trusted,
installed verbatim by herald.

## Building

lumen-dbg fetches a pinned [glyph](https://github.com/AspisOS/glyph) toolkit
artifact (the GUI libraries it links: glyph + libaudio + libauth + libcitadel)
and builds against it, then packs a signed herald package.

```sh
make MUSL_CC=/path/to/musl-gcc HERALD_KEY=/path/to/signing.key
```

- `GLYPH_VERSION` pins the toolkit release fetched by `tools/fetch-glyph.sh`.
- `MUSL_CC` is the musl cross-compiler (the only toolchain assumption — point it
  at an Aegis-native `cc` to build on-device in the future).
- `HERALD_KEY` signs the `.hpkg`.

Output: `lumen-dbg.hpkg` (a `class=system` herald package) + `lumen-dbg.hpkg.sig`.

## Package payload

```
/bin/lumen-probe                        the diagnostic probe binary
/etc/vigil/services/lumen-probe/        the vigil service (mode=graphical, oneshot)
```

## Repository layout

```
src/        lumen-probe source
pkg/        install-tree skeleton shipped verbatim (vigil service)
tools/      fetch-glyph.sh (toolkit fetch) + pack.sh (build the signed .hpkg)
Makefile    fetch toolkit -> build -> pack
VERSION         this component's version
GLYPH_VERSION   the pinned glyph toolkit version it builds against
```

## Dependencies

`depends=lumen` — the probe is a Lumen client and connects to the compositor, so
installing it pulls [lumen](https://github.com/AspisOS/lumen).
