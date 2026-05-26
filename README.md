# sour-ios-tooling-windows

Self-contained Windows x64 build of the full libimobiledevice ecosystem,
plus the missing pieces that upstream omits and that jrjr/libimobiledevice-windows
doesn't ship.

## Why this exists

[`jrjr/libimobiledevice-windows`](https://github.com/jrjr/libimobiledevice-windows)
publishes the libimobiledevice client tools (`ideviceinfo`, `idevicebackup2`, etc.).
That's a fine start — but it omits four things that are required for a Windows
app to talk to iPhones without depending on iTunes:

1. **`usbmuxd.exe`** — the daemon. Every libimobiledevice client (and every
   pymobiledevice3 invocation) talks to a usbmuxd daemon over a TCP socket
   on `127.0.0.1:27015`. On Linux/macOS this daemon is installed by the
   package manager. On Windows, the only daemon listening on that port by
   default is Apple Mobile Device Service (AMDS), which ships with iTunes.
   **Shipping our own `usbmuxd.exe` is what removes the iTunes dependency.**
2. **`ideviceinstaller.exe`** — separate project from libimobiledevice
   proper. Needed for `apps list / install / uninstall` against iOS devices.
3. **`irecovery.exe`** — from libirecovery, separate project. Needed for
   DFU/recovery-mode boot-normal (the "Exit Recovery" feature).
4. **`ideviceactivation.exe`** — from libideviceactivation, separate
   project. Apple Configurator-style activation.

We also ship `wdi-simple.exe` from [`pbatard/libwdi`](https://github.com/pbatard/libwdi)
so that DFU-mode USB access (which requires the WinUSB driver bound to the
iPhone in DFU mode) can be set up programmatically without users having to
run Zadig manually.

## Output

Each successful build publishes a release tagged `v<YYYYMMDD>-<git-sha>` with
one asset:

- `sour-ios-tooling-windows-x64.zip` — flat directory of `.exe` + `.dll`
  files. Drop the contents anywhere; everything's relative-linked via
  MinGW64 conventions.

Plus a `sha256.txt` companion for verification.

## How to consume

In your app's binary-fetch pipeline, point at the latest release tag:

```js
url: `https://github.com/<OWNER>/sour-ios-tooling-windows/releases/download/${TAG}/sour-ios-tooling-windows-x64.zip`
```

At runtime, spawn `usbmuxd.exe` as a background process before invoking any
`idevice*` tool:

```ts
const usbmuxd = spawn(path.join(toolingDir, "usbmuxd.exe"), [], {
  detached: false,
  stdio: "ignore",
});
process.on("exit", () => usbmuxd.kill());
```

After that, every libimobiledevice (and pymobiledevice3) call works without
iTunes / AMDS installed on the host.

## Build schedule

- **Weekly:** Every Sunday 00:00 UTC (matches jrjr's cadence so we track
  the same upstream cuts).
- **Manual:** Trigger `Actions → build → Run workflow` to cut a release
  on demand.
- **On push:** Pushes to `main` that touch the workflow rebuild.

## Source tracking

All upstream sources are pulled fresh from their respective master branches
at build time (libplist, libimobiledevice-glue, libtatsu, libusbmuxd,
libimobiledevice, usbmuxd, libirecovery, ideviceinstaller, libideviceactivation,
libwdi). The exact commits used for each build land in `VERSIONS.md` inside
the zip.

If we ever hit a "master broke things" event, swap `git clone --depth=1`
for `git clone -b vX.Y.Z --depth=1` per repo to pin to releases. None of
these projects publish prebuilt binaries themselves so master is the
practical version surface.

## License notes

This repo is just a build pipeline. Each upstream project carries its own
license:

| Project | License |
|---|---|
| libimobiledevice / libplist / libusbmuxd / libtatsu / libimobiledevice-glue / usbmuxd / libirecovery / ideviceinstaller / libideviceactivation | LGPL-2.1 |
| libwdi | LGPL-3.0 |

LGPL allows redistribution in commercial closed-source apps provided you
don't statically link and you include attribution. We dynamically link
everything; the bundle qualifies. Include a `LICENSES.txt` in your
distributing app.

## Smoke test runs as part of every build

Before publishing the release, the CI invokes `--help` on each binary to
catch DLL-link failures. A build with a broken binary fails CI rather than
shipping a half-bundle.
