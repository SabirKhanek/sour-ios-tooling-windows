# sour-ios-tooling-windows

Self-contained Windows x64 build of libimobiledevice + the pieces upstream
omits and that [jrjr/libimobiledevice-windows](https://github.com/jrjr/libimobiledevice-windows)
doesn't ship.

## What's in the zip

| Tool | What it's for |
|---|---|
| `ideviceinfo.exe`, `idevicebackup2.exe`, `idevicepair.exe`, `idevicediagnostics.exe`, `idevicesyslog.exe`, `idevice_id.exe`, `plistutil.exe`, `iproxy.exe`, `afcclient.exe` | Standard libimobiledevice client tools — same set `jrjr` ships, kept current weekly |
| `ideviceinstaller.exe` | App install / list / uninstall (separate libimobiledevice subproject — jrjr omits this) |
| `irecovery.exe` | DFU / recovery-mode boot-normal (libirecovery — jrjr omits this) |
| `ideviceactivation.exe` | Apple-Configurator-style activation (libideviceactivation — jrjr omits this) |
| `wdi-simple.exe` | libwdi CLI — binds WinUSB driver to iPhone in DFU mode |
| All transitive DLLs | `libimobiledevice*.dll`, `libusbmuxd*.dll`, `libplist*.dll`, `libtatsu*.dll`, `libssl-3-x64.dll`, `libcrypto-3-x64.dll`, `libcurl*.dll`, `libxml2*.dll`, `libzip*.dll`, zlib, etc. |

## Important: iTunes is required at runtime

Every tool in this bundle that talks to a USB-connected iPhone (every
`idevice*.exe`, plus `ideviceinstaller.exe`, plus `irecovery.exe` in
normal-mode use) is a **usbmuxd client**. On Linux/macOS the matching
usbmuxd daemon is installed by the OS. **On Windows, the only matching
daemon is Apple Mobile Device Service (AMDS), which ships with iTunes.**

The upstream libimobiledevice/usbmuxd daemon project is intentionally
Linux-only — see [libimobiledevice/libusbmuxd#153](https://github.com/libimobiledevice/libusbmuxd/issues/153),
[#263](https://github.com/libimobiledevice/libusbmuxd/issues/263),
[#93](https://github.com/libimobiledevice/libusbmuxd/issues/93). We
cannot replace this without writing a Windows port of usbmuxd from
scratch (a project nobody has shipped yet).

So users of any app that consumes this bundle **must have iTunes installed
on their Windows host**. iTunes provides Apple Mobile Device Support
(AMDS), which listens on `127.0.0.1:27015` and speaks the usbmuxd protocol
every tool here connects to.

We recommend consumer apps detect AMDS at startup and surface a clear
"install iTunes" prompt if missing. (See the example service detection in
[SabirKhanek/sourapp/apps/desktop/src/main/services/appleServicesDetector.ts](https://github.com/SabirKhanek/sourapp).)

## Build pipeline

Built weekly (Sunday 00:00 UTC) and on demand via `workflow_dispatch`.

All upstream sources are pulled fresh from their master branches:
`libplist`, `libimobiledevice-glue`, `libtatsu`, `libusbmuxd`,
`libimobiledevice`, `libirecovery`, `ideviceinstaller`,
`libideviceactivation`, [`pbatard/libwdi`](https://github.com/pbatard/libwdi).

The exact commits used per build land in `VERSIONS.md` inside the zip.

## Output

Each successful build publishes a release tagged `v<YYYYMMDD>-<git-sha>`
with one asset:

- `sour-ios-tooling-windows-x64.zip` — flat directory of `.exe` + `.dll`
  files. Drop the contents anywhere; everything's relative-linked via
  MinGW64 conventions.

Plus a `sha256.txt` companion for verification.

## How to consume

In your app's binary-fetch pipeline, point at the latest release tag:

```js
url: `https://github.com/SabirKhanek/sour-ios-tooling-windows/releases/download/${TAG}/sour-ios-tooling-windows-x64.zip`
```

At runtime, your app should:

1. **Detect AMDS** at startup (probe `127.0.0.1:27015`) and surface a
   banner if missing. Do NOT attempt to spawn a competing daemon — none
   exists for Windows.
2. **Run `wdi-simple.exe --vid 0x05ac --pid 0x1227 --type 0`** the first
   time an iPhone is detected in DFU mode, to bind WinUSB.

## License notes

This repo is just a build pipeline. Each upstream project carries its
own license:

| Project | License |
|---|---|
| libimobiledevice / libplist / libusbmuxd / libtatsu / libimobiledevice-glue / libirecovery / ideviceinstaller / libideviceactivation | LGPL-2.1 |
| libwdi | LGPL-3.0 |

LGPL allows redistribution in commercial closed-source apps provided you
don't statically link and you include attribution. We dynamically link
everything; the bundle qualifies. Include a `LICENSES.txt` in your
distributing app.

Consult each project's repository for the exact license terms.

## Smoke test runs as part of every build

Before publishing the release, the CI invokes `--help` on each binary
to catch DLL-link failures. A build with a broken binary fails CI rather
than shipping a half-bundle.
