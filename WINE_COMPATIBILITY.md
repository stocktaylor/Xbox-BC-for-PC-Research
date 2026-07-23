# Linux / Wine / Proton Compatibility Notes

This is a static-analysis assessment of how well this Original Xbox BC package would
likely run under Wine/Proton on Linux, based solely on what's observable in the shipped
files (see [TECHNICAL_FINDINGS.md](TECHNICAL_FINDINGS.md) for how these were found).
**Nothing here was actually tested under Wine or Proton** — this is a prediction of
where the friction is likely to be, not a compatibility report.

## The emulator core itself is not the problem

The actual translated compatibility/game code (`xefu_*.dll`, `xeo3_*.dll`) is ordinary
native **x86-64 Windows PE code** — see [TECHNICAL_FINDINGS.md §2](TECHNICAL_FINDINGS.md#2-the-x64-dlls-are-statically-recompiled-code-not-a-runtime-cpu-emulator).
There's no ARM-style instruction translation or exotic architecture involved; this part
is exactly the kind of native x64 code Wine is built to run. The friction is almost
entirely in the Windows-platform plumbing wrapped around it, not the emulator core.

## Likely hard blockers

### 1. MSIX + Windows App SDK
`EmuMenu` is a WinUI 3 app built on `Microsoft.WindowsAppRuntime.1.8`
(`Microsoft.WindowsAppRuntime.Bootstrap*.dll`, `Microsoft.WinUI.dll`), and the whole
package is distributed as an MSIX-style appx (`appxmanifest.xml`, `<Identity>`,
`unvirtualizedResources`/`customInstallActions` capabilities — see
[TECHNICAL_FINDINGS.md §7](TECHNICAL_FINDINGS.md#7-manifestconfig-details-worth-calling-out)).
Wine's AppX activation and Windows App SDK bootstrap support is minimal and inconsistent.
This class of app (MSIX + Windows App SDK) is one of the worst-supported categories on
Wine in general, independent of anything Xbox-specific. *(See "Possible Mitigation:
Replace EmuMenu" below — this specific blocker may be avoidable.)*

### 2. WebView2
`EmuMenu` embeds WebView2 interop (`Microsoft.Web.WebView2.Core.dll`,
`WebView2Loader.dll`) but does **not** ship the WebView2 runtime itself — it expects a
real Edge/Chromium WebView2 component installed on the host. Getting a functioning
WebView2 runtime working inside a Wine prefix is a known source of fragility.

### 3. Xbox Live licensing / identity
`appxmanifest.xml` requests the `appLicensing` capability, registers protocol
`ms-xbl-<titleid>`, and the package carries an `MSAAppId` and `StoreId`
(`MicrosoftGame.config`). The embedded Xbox 360 dashboard flash image includes
`signin.xex` and `PlayReady.xex` (both visible in `SystemExtPartition/system.manifest`'s
file table — see [TECHNICAL_FINDINGS.md §3](TECHNICAL_FINDINGS.md#3-a-trimmed-xbox-360-dashboard-is-embedded-not-just-game-code)).
Together this points to a runtime entitlement/sign-in check against Xbox Live, most
likely brokered through Windows' Web Account Manager (WAM) — a Windows-only
authentication component with no real Wine equivalent. This is a recurring, well-known
pain point for GDK titles on Proton generally, separate from any graphics or API
translation concern.

### 4. Encrypted game payload
`Content/Game/DefaultPackage.data/Data0000...` is high-entropy/encrypted (see
[TECHNICAL_FINDINGS.md §6](TECHNICAL_FINDINGS.md#6-game-payload)). Whatever key material
unlocks it is almost certainly tied to a legitimate license/account rather than being
derivable from the files alone — so even a flawless Wine/Proton environment likely
doesn't yield a playable game without genuine entitlement through the real Xbox app
infrastructure.

## Softer / uncertain risks

### 5. GameInput API
`Installers/GameInputRedist.msi` installs Microsoft's relatively new GameInput API
(referenced in `MicrosoftGame.config`'s `CustomInstallActions`). Wine/Proton support for
GameInput is recent and still partial as of current builds, so controller input
specifically could be flaky even if everything else launched successfully.

### 6. libHttpClient.GDK.dll
Used for Xbox Live networking (`Content/libHttpClient.GDK.dll`). Other GDK titles run
under Proton reasonably often (e.g. Forza Horizon, Halo Infinite), so this alone isn't
disqualifying — but it's one more dependency in an already dependency-heavy stack.

## What's probably *not* a problem

- **D3D12** — the package ships its own `D3D12Core.dll` (Agility SDK-style
  self-contained D3D12), which VKD3D-Proton generally handles well.
- **VC++ Redistributable** — trivial under Wine.
- No unusual kernel-mode drivers, anti-cheat, or other especially exotic system
  dependencies were observed in the shipped files.

## The missing `Emu.exe`

`EmuMenuLaunchArguments.txt` points at `exe=..\Emu.exe`, but that binary is not present
in this package (see the README's
[Emulator Menu Launch Arguments](README.md#emulator-menu-launch-arguments) section) —
it's presumably installed once, system-wide, as a shared component by the Xbox app
itself rather than bundled per-title. Since it's distributed through that same
MSIX/Windows App SDK/Store machinery described above, simply locating a standalone copy
of the file is unlikely to sidestep the compatibility issues above — it would come
wrapped in the same platform requirements as everything else in this package.

## Possible Mitigation: Replace EmuMenu (Speculative)

**Everything in this section is conjecture — it has not been verified against `Emu.exe`,
which isn't in this package, and no replacement has been built or tested.** It's recorded
here as a plausible avenue worth checking once `Emu.exe` is obtained, not as a working
plan.

`EmuMenu` (per its own DLL contents — see the README's
[Emulator Menu Launch Arguments](README.md#emulator-menu-launch-arguments) section)
looks like a thin settings/branding shell: it reads/writes an INI-based settings store
(`SettingsIni`, `SettingDisplay`, `Resolution`/`Resolutions`, `SaveSettings`), shows a
title background and name from `EmuMenuLaunchArguments.txt`, and — per that same file's
`exe=..\Emu.exe` line — ultimately spawns the real emulation host, `Emu.exe`, as a
separate process. All of the WinUI 3 / Windows App SDK / WebView2 dependencies are
declared in `EmuMenu.deps.json` / `EmuMenu.runtimeconfig.json`, scoped to that one
process — nothing observed suggests `Emu.exe` itself shares those dependencies.

If that separation holds, blocker #1 (and possibly #2) may be specific to the menu shell
rather than the emulator, which would mean:
- A minimal, non-WinUI3 replacement for `EmuMenu` (even a plain console/CLI launcher)
  that sets up the same working directory and reads `LaunchArguments.txt` could
  potentially invoke `Emu.exe` directly with equivalent arguments, sidestepping MSIX/
  Windows App SDK/WebView2 entirely.
- MSIX packaging mainly governs install/entry-point discovery through the Xbox App —
  a `Windows.FullTrustApplication`-type exe like `Emu.exe`, once files are already on
  disk, is not obviously re-checked against AppX activation on every launch the way
  `EmuMenu.exe` (a WinUI3/Windows App SDK process) would be.

Key unknowns that would need to be confirmed against the real `Emu.exe` before this is
anything more than a guess:
- Its actual invocation contract — command-line arguments, environment variables, or a
  config file path — is unknown; the assumption that it accepts something derived from
  `LaunchArguments.txt` is inferred from file naming/adjacency, not confirmed.
- Whether `Emu.exe` itself calls into Xbox Live/licensing APIs independently of
  `EmuMenu` (see blocker #3) — if so, replacing the UI shell doesn't address that
  separate, likely harder problem.
- Whether `Emu.exe` has its own undocumented dependency on Windows App SDK/WinRT APIs
  that just aren't visible from `EmuMenu`'s side.

## Bottom line

The compiled emulator code itself isn't architecturally hostile to Wine — it's native
x64. But it's wrapped in nearly every category of modern Windows platform plumbing that
Wine struggles with most (MSIX, Windows App SDK, WebView2, Xbox Live/MSA licensing), and
that surrounding stack is a substantially bigger obstacle than the emulator itself.

---

*This document reflects analysis only — no attempt was made to actually install or run
any part of this package under Wine or Proton.*
