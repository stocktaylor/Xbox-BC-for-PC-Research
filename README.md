# Original Xbox Backward Compatibility Project - Fuzion Frenzy®

This project is research into the Original Xbox backward compatibility package that allows Original Xbox games to run on Windows 10/11 using an emulator. The specific package analyized is designed to run the game "Fuzion Frenzy®" on PC.

Microsoft announced this functionality on July 22nd, 2026 and made 4 Original Xbox games available.  The games can be downloaded via the Xbox app, and the file structure below is what is contained within the installation folder.

See [TECHNICAL_FINDINGS.md](TECHNICAL_FINDINGS.md) for a deep-dive binary analysis, [WINE_COMPATIBILITY.md](WINE_COMPATIBILITY.md) for notes on what would likely make this difficult to run under Wine/Proton on Linux, and [XENIA_COMPARISON.md](XENIA_COMPARISON.md) for how this compares architecturally to the open-source Xenia Canary Xbox 360 emulator.

Binary analysis of the shipped files (see [TECHNICAL_FINDINGS.md](TECHNICAL_FINDINGS.md) for the full deep dive) shows this isn't a from-scratch Original Xbox emulator: it repackages the **Xbox 360's own built-in Original Xbox backward compatibility layer**, internally codenamed **"Fusion"**. Several files that ship in this package (`xefu.xex`, `xefutitle.xex`, and everything under `Flash/`) are genuine Xbox 360 XEX executables — the same PowerPC code that has run on real Xbox 360 consoles since 2006 to play OG Xbox discs. That Xbox 360 code, along with a trimmed copy of the Xbox 360 dashboard it depends on, has been statically recompiled from PowerPC to native x64 (via an internal Microsoft toolchain named Ficl/"Fission") and shipped as the `xeo3_*.dll` / `xefu_*.dll` files, so the original Xbox title effectively runs "inside" a virtualized Xbox 360 on PC, the same way it did on real Xbox 360 hardware.

## Project Structure

```
target/
├── BC3EA99F-9654-445F-8E1C-100B0CE16047          # Main game files
├── Content/                                     # Game content and configuration
│   ├── appxmanifest.xml                         # Application manifest for Windows Store
│   ├── background_launcher.png                  # Background image for launcher
│   ├── BootAnim.mp4                             # Boot animation
│   ├── D3D12Core.dll                            # Direct3D 12 core library
│   ├── Default.ico                              # Default icon
│   ├── DX12DirectResolveShaders.sbin            # Direct3D 12 resolve shaders
│   ├── DX12EdramResolveShaders.sbin             # Direct3D 12 EDRAM resolve shaders
│   ├── dxcompiler.dll                           # DirectX shader compiler
│   ├── gamelaunchhelper.exe                     # Game launch helper executable
│   ├── LaunchArguments.txt                      # Launch arguments for the game
│   ├── layout_eadf947c-f1d1-f8b7-aeee-62fc4f9f047f.xml  # Layout configuration
│   ├── libHttpClient.GDK.dll                    # HTTP client library for GDK
│   ├── Logo.png                                 # Game logo
│   ├── MicrosoftGame.config                     # Microsoft Game configuration
│   ├── resources.pri                            # Resource file
│   ├── SmallLogo.png                            # Small logo
│   ├── SplashScreen.png                         # Splash screen
│   ├── StoreLogo.png                            # Store logo
│   ├── Thumbs.db                                # Thumbnail cache
│   ├── VGPUDX12.dll                             # Virtual GPU DirectX 12 library
│   ├── WideLogo.png                             # Wide logo
│   ├── xefu_*.dll                               # Statically recompiled Xbox 360 compat-layer code (see System Libraries below)
│   ├── xeo3_*.dll                               # Statically recompiled Xbox 360/Original Xbox game code (see System Libraries below)
│   ├── EmuMenu/                                 # Emulator menu application
│   │   ├── EmuMenu.exe                          # Emulator menu executable
│   │   ├── EmuMenu.dll                          # Emulator menu library
│   │   ├── EmuMenu.deps.json                    # Dependency file
│   │   ├── EmuMenu.runtimeconfig.json           # Runtime configuration
│   │   ├── EmuMenuLaunchArguments.txt           # Emulator menu launch arguments
│   │   └── ...                                  # Additional .NET libraries
│   ├── Flash/                                   # Flash memory files
│   │   ├── hud.xex                              # HUD executable
│   │   ├── huduiskin.xex                        # HUD UI skin executable
│   │   ├── xam.xex                              # Xbox Advanced Media executable
│   │   ├── xboxkrnlcf.bin                       # Xbox kernel configuration file
│   │   ├── xboxkrnlcf.hvdata                    # Xbox kernel hypervisor data
│   │   ├── xenonjklatin.xtt                     # Xbox font file
│   │   └── ximecore.xex                         # XIME core executable
│   ├── Game/                                    # Game files
│   │   ├── DefaultPackage                       # Default package
│   │   └── DefaultPackage.data/                 # Game data files
│   ├── Installers/                              # Installer files
│   │   └── GameInputRedist.msi                  # Game input redistributable
│   ├── Strings/                                 # Localization files
│   │   └── Resources.resw                       # Resource file for localization
│   ├── SystemAuxPartition/                      # Auxiliary system partition (seen empty in the Fuzion Frenzy package; absent entirely from the Conker package — appears optional/title-dependent)
│   ├── SystemExtPartition/                      # Extended system partition
│   ├── SystemPartition/                         # System partition
│   └── XeO3_ShaderCache/                        # Shader cache
├── Shaders/                                     # Shader files
└── *.xsp, *.smd, *.xct, *.xvi, *.xvs, *.xss     # Game-specific files
```

## Key Files and Components

### Main Configuration Files
- **appxmanifest.xml**: Windows application manifest that defines the application identity, dependencies, and capabilities
- **MicrosoftGame.config**: Game configuration file that specifies game identity, executable list, and platform dependencies
- **LaunchArguments.txt**: Command-line arguments used when launching the game

### Emulator Components
- **EmuMenu/**: The emulator menu application built with .NET technologies
- **EmuMenuLaunchArguments.txt**: Arguments for launching the emulator menu
- **gamelaunchhelper.exe**: Helper executable for launching games

### Game Assets and Content
- **Flash/**: Contains Xbox executable files (.xex) and system files needed for the game to run
- **Game/**: Main game data and package files
- **Strings/**: Localization files for multiple languages

### System Libraries
- **xefu_*.dll** and **xeo3_*.dll**: Native x64 DLLs produced by statically recompiling the Xbox 360's Original Xbox compatibility XEXs (PowerPC) ahead of time — the actual translated compatibility/game code, not a generic runtime CPU emulator
- **D3D12Core.dll** and **VGPUDX12.dll**: DirectX 12 graphics libraries for rendering; `VGPUDX12.dll` also owns the `XeO3_ShaderCache/` directory
- **SystemPartition/Compatibility/**, **SystemExtPartition/**, **Flash/*.xex**: genuine Xbox 360 XEX executables (`xefu.xex`, `xefutitle.xex`, `Xam.Community.xex`, `xam.xex`, `hud.xex`, `huduiskin.xex`, `ximecore.xex`) plus `xboxkrnlcf.bin`, a PowerPC big-endian kernel-compatibility shim — together these form the trimmed Xbox 360 + Original Xbox kernel environment the game actually boots into

### Per-Title vs. Shared Components
Comparing the Fuzion Frenzy and Conker: Live and Reloaded packages byte-for-byte shows the `xeo3_*.dll` / `xefu_*.dll` set splits cleanly into two categories:
- **Shared engine files** — the 8 `xeo3_<hash>.dll` (+ `_no` twin) pairs and `xefu_69c41281_00027bcf.dll` (the recompiled `xb1krnl` kernel shim) are **byte-identical** (same filenames, same SHA-256) between both packages. These are the generic Xbox 360/Original-Xbox compatibility engine, reused as-is across titles.
- **One per-title file** — each package carries exactly one additional `xefu_<hash>.dll` that differs per game (`xefu_ad64d751_..._81821c8d.dll` for Fuzion Frenzy vs. `xefu_70648536_..._ad43dc7e.dll` for Conker). Debug strings inside it point at the title's own recompiled `default.xbe` (e.g. `...\ppc\titles\ConkerLiveandReloaded_4D530051\GAM_0\...`) — this is the one file that is genuinely game-specific.
- Even though the shared files are byte-identical, the internal build-tree branch label embedded in their debug strings differs per package (`20F914` for the Fuzion Frenzy package vs. `20F912` for the Conker package) — the shared engine appears to be *rebuilt* for every title package rather than copied from one master build, but converges on identical output when the underlying Xbox 360 XEX hasn't changed. The per-title DLL is always built on its own separate branch (`20F919` for Fuzion Frenzy, `20F917` for Conker).

## Game Information
Values below are per-title; both packages observed so far are listed for comparison (see [TECHNICAL_FINDINGS.md](TECHNICAL_FINDINGS.md) for how these were extracted).

| Field | Fuzion Frenzy® | Conker: Live and Reloaded |
|---|---|---|
| Package/Store Title ID (`MicrosoftGame.config`) | 057442C0 | 27028FBB |
| Original Xbox Title ID (`LaunchArguments.txt`) | 4D530856 | 4D530051 |
| Store ID | C2P985H1H42H | BVFB8CBS75R6 |
| Package Version | 2607.1523.1.0 | 2607.1623.1.0 |
| MSAAppId | 00000000482787E9 | 000000004C270AF2 |
| SaveGameStorage SCID / `configurationId` | 74340100-b1d0-46db-885f-e6bc057442c0 | 33f30100-1908-4e64-bd9a-cab427028fbb |
| Protocol registered | `ms-xbl-057442c0` | `ms-xbl-27028fbb` |
| `Content/Game/DefaultPackage.data/` chunk count | 13 (~2.0 GB) | 29 (~4.6 GB) |

Publisher is `Gaming Platform Team` for both. Both Original Xbox title IDs start with `4D53` (`"MS"` in ASCII) — the publisher code Microsoft used for first-party Original Xbox titles (Fuzion Frenzy was Microsoft Game Studios; Conker: Live and Reloaded was Rare, which Microsoft owned and self-published under the same prefix). Game payload chunk size is a fixed 170,459,136 bytes (~162.56 MiB) per chunk in both packages, only the final chunk is shorter — chunk *count* simply scales with the game's disc size.

## Platform Support
This package is designed for Windows 10/11 desktop platforms with:
- Minimum Windows version: 10.0.19044.3086
- Maximum tested version: 10.0.26200.8117
- Processor architecture: x64

## Installation Requirements
- Microsoft Visual C++ Redistributable
- Windows App Runtime 1.8
- GameInput redistributable (installed via MSI)

## Technical Details

### Launch Arguments
`Content/LaunchArguments.txt` is **not a fixed template** — each title ships its own tailored set of flags. Comparing the two packages observed:

**Fuzion Frenzy®:**
```
aaBoostOn
aaBoostTargetMsaa=1
disableAudioOnConstrained
fusion
mediaId=b812b25c-ff51-40f1-80ef-e0e1ddca4f38
scalingResolutions=720x0 844x0 1280x0 0x240 0x480 0x256
titleId=4D530856
configurationId=74340100-b1d0-46db-885f-e6bc057442c0
```

**Conker: Live and Reloaded:**
```
aaBoostOn
aaBoostTargetMsaa=1
fusion
mediaId=4d530051-0000-0000-0000-000000000000
titleId=4D530051
configurationId=33f30100-1908-4e64-bd9a-cab427028fbb
```

Takeaways from the diff:
- `aaBoostOn`, `aaBoostTargetMsaa=1`, `fusion`, `mediaId`, `titleId`, and `configurationId` are common to both — likely always present.
- `disableAudioOnConstrained` and `scalingResolutions=...` appear **only** for Fuzion Frenzy. These read as per-title compatibility tuning flags (audio behavior under constrained/low-power conditions, and a fixed list of display-scaling resolution buckets the original title rendered at) that Microsoft opts individual titles into rather than applying universally.
- `fusion` references the internal **"Fusion"** codename for the Xbox 360-derived compatibility subsystem — see [TECHNICAL_FINDINGS.md](TECHNICAL_FINDINGS.md).
- `mediaId` format differs meaningfully: Fuzion Frenzy has a genuine random-looking GUID (presumably the original disc's real XGD media ID), while Conker's is a **synthesized** GUID — literally the hex title ID (`4d530051`) padded with zeros (`4d530051-0000-0000-0000-000000000000`). This suggests that when a title's real original-disc media ID either isn't tracked or isn't needed, the build pipeline fabricates a placeholder GUID from the title ID instead of leaving the field blank.
- `configurationId` always matches the `SCID` from that title's `MicrosoftGame.config`.

### Emulator Menu Launch Arguments
`Content/EmuMenu/EmuMenuLaunchArguments.txt` follows a fixed template that only swaps the title name, e.g. for Fuzion Frenzy:
```
exe=..\Emu.exe
titlename=Fuzion Frenzy®
background=..\background_launcher.png
```
(Conker: Live and Reloaded's copy is identical apart from `titlename=Conker: Live and Reloaded`.)

`Emu.exe` itself is not shipped in either package, which suggests it's a shared component installed once system-wide rather than bundled per-title. `EmuMenu` is a .NET 8 / WinUI 3 desktop app that also embeds WebView2 and DirectX interop (via Vortice/SharpGen), and persists its own settings through an INI-based `SettingsIni` system.

## How It Works
This project uses an Original Xbox emulator to run Original Xbox games on Windows. The package includes:
1. The emulator menu (EmuMenu) that provides the user interface
2. Game executables and assets in the Flash directory
3. Compatibility libraries that translate Original Xbox calls to Windows APIs
4. Configuration files that define how the game should be launched and run

Under the hood, this backward compatibility approach doesn't emulate Original Xbox hardware directly. It repackages the Original Xbox compatibility layer that has shipped inside every Xbox 360 since 2006: an Xbox 360 kernel-level shim (`xboxkrnlcf.bin`) hosts a recompiled Original Xbox kernel (`xb1krnl`), which runs the actual Original Xbox title (`default.xbe`) inside a trimmed, virtualized Xbox 360 environment (dashboard, sign-in, guide, etc., all present as real Xbox 360 XEX modules). That whole PowerPC stack — Xbox 360 compatibility layer plus the Original Xbox title's code — is statically recompiled ahead of time to native x64 and shipped as the `xefu_*.dll` / `xeo3_*.dll` files, with `VGPUDX12.dll` translating the original GPU command stream to Direct3D 12. See [TECHNICAL_FINDINGS.md](TECHNICAL_FINDINGS.md) for the full evidence trail (embedded build paths, XEX headers, shader cache title IDs, etc.).
