# Original Xbox Backward Compatibility Project - Fuzion Frenzy®

This project is research into the Original Xbox backward compatibility package that allows Original Xbox games to run on Windows 10/11 using an emulator. The specific package analyized is designed to run the game "Fuzion Frenzy®" on PC.

Microsoft announced this functionality on July 22nd, 2026 and made 4 Original Xbox games available.  The games can be downloaded via the Xbox app, and the file structure below is what is contained within the installation folder.

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
│   ├── xefu_*.dll                               # Xbox Emulation Framework libraries
│   ├── xeo3_*.dll                               # Xbox Emulation libraries
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
- **xefu_*.dll** and **xeo3_*.dll**: Xbox Emulation Framework libraries that provide compatibility layer
- **D3D12Core.dll** and **VGPUDX12.dll**: DirectX 12 graphics libraries for rendering

## Game Information
- **Game Title**: Fuzion Frenzy®
- **Title ID**: 057442C0
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

## How It Works
This project uses an Original Xbox emulator to run Original Xbox games on Windows. The package includes:
1. The emulator menu (EmuMenu) that provides the user interface
2. Game executables and assets in the Flash directory
3. Compatibility libraries that translate Original Xbox calls to Windows APIs
4. Configuration files that define how the game should be launched and run

The system uses a backward compatibility approach that allows Original Xbox games to run on modern Windows systems by emulating the Original Xbox hardware and operating system environment.

---

This research work was done by qwen3-coder