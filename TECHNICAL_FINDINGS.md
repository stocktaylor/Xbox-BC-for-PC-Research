# Technical Findings — Original Xbox BC ("Fuzion Frenzy") Deep Dive

This document goes beyond the file-listing overview in [README.md](README.md) /
[PROJECT_SUMMARY.md](PROJECT_SUMMARY.md) and records what can actually be learned by
inspecting the binaries and metadata files shipped in `target/`. Findings below come from
`file`, `strings`, and manual header inspection of the shipped DLLs/XEXs/config files — no
disassembly or execution was performed.

## 1. It is Xbox‑360‑on‑PC running the Xbox 360's *own* original‑Xbox compatibility layer

The most important discovery: several files that ship in this package are not native
Windows code — they're genuine **Xbox 360 XEX executables** (`file` reports them as
`Microsoft Xbox 360 executable`, i.e. the same format used by real 360 discs/dashboards):

| File | Location | Notes |
|---|---|---|
| `xefu.xex` | `Content/SystemPartition/Compatibility/` | 652 KB — the compatibility "engine" |
| `xefutitle.xex` | `Content/SystemPartition/Compatibility/` | 12 KB — thin per-title stub |
| `Xam.Community.xex` | `Content/SystemExtPartition/20426B00/` | Xbox 360 dashboard module |
| `xam.xex`, `hud.xex`, `huduiskin.xex`, `ximecore.xex` | `Content/Flash/` | Xbox 360 dashboard/HUD/IME modules |

In other words, Microsoft isn't emulating the original Xbox from scratch on PC. It is
re-hosting the *Xbox 360's* built-in original-Xbox backward-compatibility subsystem (the
same PowerPC code that ran on real Xbox 360 consoles from 2005 onward to play OG Xbox
discs), plus a slice of the Xbox 360 dashboard/kernel environment that subsystem depends
on. That combined Xbox 360 (PowerPC) payload is what actually gets ported to Windows.

Supporting evidence from embedded debug strings:
- `xeo3_11149bba_...dll` embeds a build-time JSON job description referencing
  `PdbFile: ...\FusionSymbols\xefutitlec.pdb` and tree root
  `D:\btsdx\20F914\xbox\emulator\ppc\system\free\Fusion\` — the compatibility subsystem's
  internal codename is **"Fusion"**, which also explains the bare `fusion` token present in
  `Content/LaunchArguments.txt`.
- `xefu_69c41281_00027bcf.dll` embeds the path
  `...\SystemPartition\Compatibility\xb1krnl.exe` — "xb1krnl" reads as *"Xbox (first‑gen)
  kernel"*, i.e. the original Xbox kernel, recompiled to run inside the Fusion framework.
- `Content/Flash/xboxkrnlcf.bin` is a **PowerPC, big-endian** PE image (`file`: "PE32
  executable for XBOX, PowerPC 64-bit") — "xbox kernel compat framework" — the shim that
  translates original-Xbox kernel calls into Xbox 360 kernel calls. This is the same
  mechanism real Xbox 360 hardware used for OG Xbox BC.
- `xefu_ad64d751_..._81821c8d.dll` embeds the literal path
  `D:\btsdx\20F919\xbox\emulator\ppc\titles\FuzionFrenzy4D530856\GAM_0\default.xbe` —
  proof that Microsoft's build pipeline pulled in the real original-Xbox executable
  (`default.xbe`, `GAM_0` = the disc-layout folder name) for this title and title ID
  **4D530856** matches `titleId=4D530856` in `LaunchArguments.txt`. The top 16 bits of an
  original-Xbox title ID are an ASCII publisher code — `0x4D53` = `"MS"` (Microsoft), the
  standard prefix for first-party OG Xbox titles.

## 2. The x64 DLLs are statically recompiled code, not a runtime CPU emulator

All of the `xeo3_*.dll` / `xefu_*.dll` files under `Content/` are ordinary native **x86-64
Windows PE DLLs**, unlike the XEX files above. One of them
(`xeo3_11149bba_38671b39_48735ee7_1c56d0f9_dc4f8b60.dll`) embeds the actual JSON job
config used to build it, naming a static-recompilation tool called **Ficl**
(`ficompiler.exe`, driven by `Fission_General.ctrl.json` / `XenonXdk.ctrl.json` control
files) whose input is `xefutitle.xex` and whose output is described as "Enlightenment"
data — i.e. this is Microsoft's known **static/ahead-of-time PowerPC→x64 recompiler**
("Fission"), not a general instruction-level CPU emulator. Each oddly-named DLL
(`xeo3_<hash>_<hash>_..._.dll`, often with a `_no` twin) is one precompiled code module
produced from that job; the actual translated Xbox 360/original-Xbox game code ships as
native x64 in this package rather than being recompiled at runtime.

`VGPUDX12.dll` is the companion "virtual GPU" — it translates the Xenon/Xbox GPU command
stream to Direct3D 12 and owns the on-disk shader cache (its strings reference
`\XeO3_ShaderCache` directly). `Content/XeO3_ShaderCache/` contains two title
subfolders, each holding a `V0.1.2_JIT_TVP70003_0.pak` blob:
- `4D530856` — Fuzion Frenzy's own title ID.
- `FFFE07D1` — the well-known Xbox 360 system **Dashboard** title ID, confirming the
  dashboard/Guide UI is genuinely running (and needs its own compiled shaders) alongside
  the game, not just the game.

## 3. A trimmed Xbox 360 dashboard is embedded, not just game code

`Content/SystemExtPartition/system.manifest` is a binary NAND/flash filesystem manifest.
Its embedded file table (readable via `strings`) lists dozens of genuine Xbox 360
dashboard components — `dash.xex`, `Guide.*.xex`, `Dash.*.lex`, `PlayReady.xex`,
`signin.xex`, `updater.xex`, avatar/gamercard art, and many `.xzp` resource packages —
i.e. the actual file listing of a real (if trimmed) Xbox 360 dashboard flash image.
Combined with `Xam.Community.xex` and the Flash/ XEXs, this confirms the original-Xbox
title is booted *inside* a virtualized Xbox 360 environment on PC, exactly as it would
have been on real Xbox 360 hardware — that whole stack (kernel shim, dashboard, guide,
sign-in) has been carried forward.

## 4. EmuMenu is a .NET 8 / WinUI 3 desktop shell with its own DirectX surface

`Content/EmuMenu/` is a self-contained **.NET 8** app (`EmuMenu.runtimeconfig.json` pins
`Microsoft.NETCore.App 8.0.27`; `coreclr.dll`/`hostfxr.dll`/`hostpolicy.dll` present) built
on **WinUI 3** (`Microsoft.WinUI.dll`, `Microsoft.WindowsAppRuntime.Bootstrap*.dll`). It
also embeds:
- **WebView2** (`Microsoft.Web.WebView2.Core.dll`, `WebView2Loader.dll`) — likely for
  store/marketing surfaces inside the menu.
- **Vortice** DirectX interop (`Vortice.DirectX.dll`, `Vortice.DXGI.dll`,
  `Vortice.Mathematics.dll`, `SharpGen.Runtime.dll`) — EmuMenu renders its own D3D/DXGI
  surface directly rather than only hosting XAML.
- An INI-backed settings system — `EmuMenu.dll` contains `SettingsIni`,
  `SettingsIniOperation`, `SettingsIniErrorEventArgs`, `SettingDisplay`, `Resolution`,
  `Resolutions`, `SaveSettings` types, and a `MenuConfig` model.

`EmuMenuLaunchArguments.txt` (`exe=..\Emu.exe`, `titlename=Fuzion Frenzy®`,
`background=..\background_launcher.png`) points at a separate `Emu.exe` that is **not**
shipped in this package — implying it's a shared component installed once system-wide for
all BC titles rather than bundled per-title.

## 5. Loose per-title state files at the package root

The files sitting next to `target/Content/` (named with the install GUID
`BC3EA99F-9654-445F-8E1C-100B0CE16047`) are local runtime/session state, not shipped
package payload — they carry today's file timestamps rather than the package's own dates:

| File | Format observed | Likely purpose |
|---|---|---|
| `*.xct` | Binary; starts with the package GUID (byte-swapped) | compatibility/config token cache |
| `*.xvi` | Binary; ASCII magic `crdi-xvc`, mostly zero-filled | virtual-console index/init marker |
| `*.xvs` | UTF-16LE JSON: `{"Request":{"InstanceId":"{...}"...` | session/request state |
| `*.smd` | Binary; ASCII magic `" PFX"`, embeds the `.xsp` file's GUID | session/metadata descriptor |
| `*.xsp` | All-zero, 7.5 KB | pre-allocated placeholder |
| `Shaders/<guid>.xss` | Plain JSON: `{"Version":1,"StateId":"...","ApplicationData":{"RegistrationState":"Unregistered"},"ShaderFilesResponse":{},"ShaderComponents":{}}` | shader-cache telemetry/registration state (currently unregistered — hasn't synced yet) |

These are almost certainly written by an OS-level BC host service the first time the
title is registered/launched, separate from the static install content under `Content/`.

## 6. Game payload

`Content/Game/DefaultPackage.data/Data0000`–`Data0012` are 13 opaque, high-entropy
(encrypted) chunks — 12 chunks of exactly 170,459,136 bytes (~162.56 MiB) plus one
48,840,704-byte final chunk, ~2.0 GB total. This is the actual original-Xbox Fuzion
Frenzy disc content, encrypted and split into fixed-size chunks. `DefaultPackage` (no
extension) is a small header/manifest tying the chunks together; `Content/Game/Cache/`
is empty until runtime.

## 7. Manifest/config details worth calling out

- `MicrosoftGame.config` / `appxmanifest.xml` — package identity
  `Xbox360BackwardCompatibil.PrimaryFuzionFrenzyFuzio`, entry point is
  `GameLaunchHelper.exe` (`Windows.FullTrustApplication`), registers protocol
  `ms-xbl-057442c0` (title ID in hex, lowercase). Capabilities requested:
  `internetClient`, `runFullTrust`, `appLicensing`, `unvirtualizedResources`,
  `customInstallActions`. `AdvancedUserModel` is explicitly `false` — a comment in the
  config notes PC uses a "simplified user model" that terminates the game if the default
  user signs out.
- `DesktopRegistration` declares `KnownDependency Name="VC14"` and
  `Microsoft.WindowsAppRuntime.1.8` (MinVersion 8000.806.2252.0) as dependencies, and
  runs `GameInputRedist.msi /quiet` as a custom install action.
- `layout_eadf947c-....xml` is a chunked installer manifest (`<Chunk Id="1000"
  Marker="Launch">`) listing every `FileGroup` (destination folder + source glob) needed
  for install — i.e. this is what a Windows/Xbox App-driven streaming install consumes to
  decide what to fetch and where to place it.

## 8. Build provenance

Every internal path recovered from binary strings is rooted at
`D:\btsdx\<branch>\xbox\emulator\...`, with branch labels **`20F914`** (shared/engine
components) and **`20F919`** (this specific title's recompiled code) — consistent with
Microsoft's internal Xbox OS calendar-coded branch naming. The presence of a full PDB/
symbols tree (`FusionSymbols`, `FusionKernel`, `SepSap`) and a dedicated static-recompile
toolchain (Ficl/Fission) points to an internal pipeline that: (1) takes the Xbox 360's
own original-Xbox BC XEX modules plus the target title's original `default.xbe`,
(2) statically recompiles the combined PowerPC code to x64 ahead of time via Ficl,
(3) packages the result together with a trimmed Xbox 360 dashboard flash image, the
original-Xbox kernel shim (`xboxkrnlcf`/`xb1krnl`), and the new WinUI `EmuMenu` front end,
and (4) ships it as an MSIX-style desktop package through the Xbox app. (Section 9 below
shows this pipeline runs per-title, with branch labels that shift between packages even
for otherwise-identical shared components.)

## 9. Cross-title comparison (Fuzion Frenzy® vs. Conker: Live and Reloaded)

The `target/` folder was later swapped to a second title's package — **Conker: Live and
Reloaded** (install GUID `88823EC3-7CFC-4A70-A1A1-CBBDD7186AA9`, package identity
`Xbox360BackwardCompatibil.PrimaryConkerLiveReloade`, Original Xbox title ID `4D530051`,
package/store title ID `27028FBB`, package version `2607.1623.1.0`). The folder layout
is byte-identical in structure to the Fuzion Frenzy package; diffing the two packages'
contents confirms several things that couldn't be proven from a single sample:

**The engine/game DLL split is real and content-addressed.** All 8 `xeo3_<hash>.dll`
(+ `_no` twin) pairs and `xefu_69c41281_00027bcf.dll` (the recompiled `xb1krnl` kernel
shim) are **byte-for-byte identical** (verified via `sha256sum`) between the two
packages — same filenames *and* same hashes. Each package additionally carries exactly
one more `xefu_<hash>.dll` that differs per title
(`xefu_ad64d751_d1837ae9_e9a45c7f_6396b532_81821c8d.dll` for Fuzion Frenzy vs.
`xefu_70648536_7d88890e_d7dee586_06a66283_ad43dc7e.dll` for Conker). The Conker copy's
debug strings point at
`D:\temp\20F917\obj\xbox\emulator\ppc\titles\ConkerLiveandReloaded_4D530051\GAM_0\...`,
mirroring the Fuzion Frenzy pattern (`FuzionFrenzy4D530856\GAM_0\...`) and confirming the
hashed filenames are content-addressed identifiers for compiled modules: the shared
Xbox 360 compatibility engine is one set of modules reused verbatim across titles, and
each package adds one title-specific recompiled-game module.

**But the shared modules are still rebuilt per package, not copied from one master.**
Despite being byte-identical in output, the embedded build-branch label inside the
shared DLLs differs between packages — `20F914` in the Fuzion Frenzy package's copy of
`xeo3_11149bba_...dll` and `xefu_69c41281_00027bcf.dll`, vs. `20F912` in Conker's copy of
the exact same (hash-identical) files. The per-title DLL's own branch differs too
(`20F919` for Fuzion Frenzy vs. `20F917` for Conker). This indicates each title package
is independently rebuilt from Microsoft's internal branch at packaging time, and the
shared engine components simply happen to produce identical output because the
underlying Xbox 360 compatibility XEX hasn't changed between those branch snapshots —
this isn't a single shared binary distributed to every package.

**`LaunchArguments.txt` is per-title, not a fixed template.** Conker's copy drops two
flags present for Fuzion Frenzy (`disableAudioOnConstrained`, `scalingResolutions=...`)
that read as opt-in compatibility tuning rather than universal flags. See the README's
[Launch Arguments](README.md#launch-arguments) section for the full side-by-side.

**`mediaId` isn't always a real disc ID.** Fuzion Frenzy's `mediaId` is a
non-obviously-derived GUID (`b812b25c-ff51-40f1-80ef-e0e1ddca4f38`), plausibly the
original disc's real XGD media ID. Conker's is transparently synthesized from its own
title ID: `mediaId=4d530051-0000-0000-0000-000000000000` — the title ID hex followed by
zeros, not a real per-disc identifier. The packaging pipeline appears to fabricate a
placeholder when a genuine media ID isn't available/needed rather than omitting the
field.

**`SystemAuxPartition/` is optional.** It exists (empty) in the Fuzion Frenzy package
but is entirely absent from the Conker package — this directory is apparently
created only when a given title needs it, not unconditionally.

**Payload size scales with the original disc, chunk size does not.** Conker's
`DefaultPackage.data/` has 29 chunks (~4.6 GB total) vs. Fuzion Frenzy's 13 (~2.0 GB) —
consistent with Conker: Live and Reloaded being a substantially larger original Xbox
game (full voice-acted remake vs. a launch-window party game). In both packages every
chunk but the last is exactly 170,459,136 bytes (~162.56 MiB); only the final chunk is
shorter, confirming a fixed chunk size with the title's true payload size determining
chunk *count*.

**The loose root-level state files (`.xct`/`.xvi`/`.xvs`/`.smd`/`.xss`) follow the same
format for both titles** — identical magic bytes/JSON shapes as documented in section 5
above, just with per-install GUIDs/`StateId`s substituted in, confirming that format is
a general BC-host convention rather than something specific to one title.

---

*Findings derived solely from static inspection (`file`, `strings`, `sha256sum`, header
bytes) of the files present in `target/` as of 2026‑07‑22, across two title packages
(Fuzion Frenzy® and Conker: Live and Reloaded). Nothing here required disassembly,
decryption, or execution of any binary.*
