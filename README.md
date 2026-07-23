# Original Xbox Backward Compatibility Project - Fuzion Frenzy®

This project is research into the Original Xbox backward compatibility package that allows Original Xbox games to run on Windows 10/11 using an emulator. The specific package analyized is designed to run the game "Fuzion Frenzy®" on PC.

Microsoft announced this functionality on July 22nd, 2026 and made 4 Original Xbox games available.  The games can be downloaded via the Xbox app, and the file structure below is what is contained within the installation folder.

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
│   ├── SystemAuxPartition/                      # Auxiliary system partition
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

## Game Information
- **Package Display Name**: Fuzion Frenzy®
- **Package/Store Title ID**: 057442C0 (`MicrosoftGame.config` — the modern Xbox package identity)
- **Original Xbox Title ID**: 4D530856 (`LaunchArguments.txt` — the title ID from the original 2003 Xbox release; the top two bytes, `4D53`, spell `"MS"`, the publisher code Microsoft used for first-party Original Xbox titles)
- **Store ID**: C2P985H1H42H
- **Publisher**: Gaming Platform Team
- **Version**: 2607.1523.1.0

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
`Content/LaunchArguments.txt` contains:
- `aaBoostOn`
- `aaBoostTargetMsaa=1`
- `disableAudioOnConstrained`
- `fusion` — references the internal "Fusion" codename for the Xbox 360-derived compatibility subsystem, see [TECHNICAL_FINDINGS.md](TECHNICAL_FINDINGS.md)
- `mediaId=b812b25c-ff51-40f1-80ef-e0e1ddca4f38`
- `scalingResolutions=720x0 844x0 1280x0 0x240 0x480 0x256`
- `titleId=4D530856`
- `configurationId=74340100-b1d0-46db-885f-e6bc057442c0`

### Emulator Menu Launch Arguments
`Content/EmuMenu/EmuMenuLaunchArguments.txt` contains:
- `exe=..\Emu.exe`
- `titlename=Fuzion Frenzy®`
- `background=..\background_launcher.png`

`Emu.exe` itself is not shipped in this package, which suggests it's a shared component installed once system-wide rather than bundled per-title. `EmuMenu` is a .NET 8 / WinUI 3 desktop app that also embeds WebView2 and DirectX interop (via Vortice/SharpGen), and persists its own settings through an INI-based `SettingsIni` system.

## How It Works
This project uses an Original Xbox emulator to run Original Xbox games on Windows. The package includes:
1. The emulator menu (EmuMenu) that provides the user interface
2. Game executables and assets in the Flash directory
3. Compatibility libraries that translate Original Xbox calls to Windows APIs
4. Configuration files that define how the game should be launched and run

Under the hood, this backward compatibility approach doesn't emulate Original Xbox hardware directly. It repackages the Original Xbox compatibility layer that has shipped inside every Xbox 360 since 2006: an Xbox 360 kernel-level shim (`xboxkrnlcf.bin`) hosts a recompiled Original Xbox kernel (`xb1krnl`), which runs the actual Original Xbox title (`default.xbe`) inside a trimmed, virtualized Xbox 360 environment (dashboard, sign-in, guide, etc., all present as real Xbox 360 XEX modules). That whole PowerPC stack — Xbox 360 compatibility layer plus the Original Xbox title's code — is statically recompiled ahead of time to native x64 and shipped as the `xefu_*.dll` / `xeo3_*.dll` files, with `VGPUDX12.dll` translating the original GPU command stream to Direct3D 12. See [TECHNICAL_FINDINGS.md](TECHNICAL_FINDINGS.md) for the full evidence trail (embedded build paths, XEX headers, shader cache title IDs, etc.).
