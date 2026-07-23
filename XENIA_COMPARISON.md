# Comparison with Xenia Canary

[Xenia Canary](https://github.com/xenia-canary/xenia-canary) is a well-known open-source
Xbox 360 emulator, pulled into this project (`xenia-canary/`) as a reference point. Since
Microsoft's official Original Xbox BC package works by running the Original Xbox title
*inside the Xbox 360's own compatibility layer* (see
[TECHNICAL_FINDINGS.md §1](TECHNICAL_FINDINGS.md#1-it-is-xbox360-on-pc-running-the-xbox-360s-own-original-xbox-compatibility-layer)),
Xenia — a real Xbox 360 emulator — solves a large overlapping set of problems, and its
public source/docs make a useful point of comparison against what could only be inferred
from binary strings in the official package.

Everything below is a comparison of design/architecture based on reading Xenia's source
and docs against the binary evidence in `target/`; no code from either project was
executed.

## CPU translation: same problem, opposite timing

Xenia's `src/xenia/cpu/ppc/` decodes Xbox 360 PowerPC (Xenon) instructions into an HIR
(intermediate representation — see `ppc_translator.cc`, `ppc_hir_builder.cc`), runs a
series of optimization passes (`cpu/compiler/passes`), then a backend
(`cpu/backend/x64/`, or `cpu/backend/a64/` for ARM64 hosts) emits native machine code.
Per Xenia's own [docs/cpu.md](../xenia-canary/docs/cpu.md), this is explicitly a
**JIT** — translation happens at runtime, and compiled code lives in an in-memory code
cache (`x64_code_cache.cc`); nothing indicates it's persisted to disk between runs.

Microsoft's real pipeline solves the analogous problem (and, one layer further down, the
original x86 Xbox → PowerPC translation the Xbox 360 always needed to run OG Xbox
discs), but does it **ahead of time at build/packaging time**, not at runtime. The
Ficl/"Fission" job blob embedded in `xeo3_11149bba_...dll`
(see [TECHNICAL_FINDINGS.md §2](TECHNICAL_FINDINGS.md#2-the-x64-dlls-are-statically-recompiled-code-not-a-runtime-cpu-emulator))
references a `CompilerExePath` (`ficompiler.exe`), a `PdbFile`, and an `EtlDigestPath` —
language for an offline, traced/profiled static-recompilation job, not a JIT. The output
of that job ships as the `xeo3_*.dll`/`xefu_*.dll` files, and there is no runtime
recompiler in the package at all. Same translation problem (guest ISA → host native
code), solved at opposite ends of the build/run timeline.

## Kernel & dashboard: HLE reimplementation vs. the genuine article

Xenia's `src/xenia/kernel/xboxkrnl/` and `src/xenia/kernel/xam/` are clean-room C++
**reimplementations** of the Xbox 360 kernel and dashboard (XAM) export surfaces.
[docs/kernel.md](../xenia-canary/docs/kernel.md) states this directly: "Xenia implements
all kernel APIs as native functions under the host" — guest imports are patched to
syscalls that dispatch into Xenia's own C++ functions (`SHIM_CALL`/shim conventions).
`kernel/xam/` further reimplements profile management, content/save management, and
achievements (`profile_manager.cc`, `content_manager.cc`, `achievement_manager.cc`) —
i.e. Xenia *fakes* the dashboard's behavior rather than running it.

Microsoft's official BC package doesn't need to reimplement any of this — it ships and
statically recompiles the **actual** Xbox 360 system binaries: `xam.xex`, `hud.xex`,
`huduiskin.xex`, `Xam.Community.xex`, and the kernel-compatibility shim
`xboxkrnlcf.bin`, plus (per debug strings) a recompiled genuine Original Xbox kernel
image (`xb1krnl`) — see
[TECHNICAL_FINDINGS.md §1 and §3](TECHNICAL_FINDINGS.md#3-a-trimmed-xbox-360-dashboard-is-embedded-not-just-game-code).
`SystemExtPartition/system.manifest`'s file table (dash/Guide/Xam `.xex`/`.lex` modules,
`.xzp` resource packages) reads like a real, if trimmed, Xbox 360 flash dashboard image.
Xenia HLEs where it has to, for lack of source; Microsoft's first-party BC stack LLEs —
runs the genuine original code — because it owns it.

## GPU: a near one-to-one match

Xenia's `src/xenia/gpu/` translates Xenos (Xbox 360 GPU) command-buffer packets and
shader microcode to DXBC or SPIR-V at runtime (`dxbc_shader_translator.cc`,
`spirv_shader_translator.cc`, `command_processor.cc`), and
[docs/gpu.md](../xenia-canary/docs/gpu.md) calls out **EDRAM** — the Xenos's on-die
high-speed framebuffer memory — as needing explicit "resolve" operations to copy it into
normal memory. That maps almost directly onto the BC package's `VGPUDX12.dll` plus
`DX12EdramResolveShaders.sbin` and `DX12DirectResolveShaders.sbin` (documented in
[TECHNICAL_FINDINGS.md §2](TECHNICAL_FINDINGS.md#2-the-x64-dlls-are-statically-recompiled-code-not-a-runtime-cpu-emulator)) —
same chip, same EDRAM-resolve concept, same rough division of labor (GPU command
translation + a distinct EDRAM-resolve shader step).

Even the caching strategy rhymes: `docs/gpu.md` notes Xenia's `--dump_shaders` option
writes translated shaders "with names based on input hash (so they'll be stable across
runs)". The BC package's `XeO3_ShaderCache/<titleId>/V0.1.2_JIT_TVP70003_0.pak` files use
the same content-addressed-cache philosophy — and notably, the filename literally
contains `JIT`, unlike the AOT-compiled CPU-side `xeo3_*.dll`s, suggesting the GPU/shader
side of Microsoft's stack — unlike the CPU side — does translate at runtime and caches
the result to disk, much like Xenia's shader cache does.

## Virtual filesystem

Xenia's `src/xenia/vfs/devices/` mounts Xbox 360 disc images (`disc_image_device.cc`,
XISO/GDF-style) and STFS/XContent packages (`xcontent_container_device.cc`,
`stfs_xbox.h`) — the same general category of container the BC package's encrypted
`Content/Game/DefaultPackage.data/Data0000...` chunks
(see [TECHNICAL_FINDINGS.md §6](TECHNICAL_FINDINGS.md#6-game-payload)) almost certainly
are. Notably, Xenia's VFS layer has **no handler for `.xzp`** (the Xbox 360 dashboard
resource-package format found throughout `system.manifest`) — it doesn't need one, since
it fakes dashboard UI/behavior rather than running the genuine dashboard that would
consume those packages.

## Per-title compatibility tuning

Xenia has a dedicated `src/xenia/patcher/` subsystem (`patch_db.cc`, `patcher.cc`,
`plugin_loader.cc`) backing a community-maintained database of per-game patches
(widescreen fixes, timing hacks, etc.). Microsoft's much smaller equivalent is the
per-title flags observed in `LaunchArguments.txt` — `aaBoostOn`,
`disableAudioOnConstrained`, `scalingResolutions=...` — which differ between the two
titles compared in
[TECHNICAL_FINDINGS.md §9](TECHNICAL_FINDINGS.md#9-cross-title-comparison-fuzion-frenzy-vs-conker-live-and-reloaded).
Both projects land on the same conclusion by necessity: no compatibility layer is
perfect out of the box for every title, so both carry a per-title override mechanism —
Xenia's is a general, community-extensible patch database; Microsoft's is a small,
first-party set of build-time flags.

## The big picture

Xenia is what you build when you *don't* have Microsoft's source: a from-scratch
reimplementation, HLE-ing the kernel/dashboard where no genuine binary is available, and
JIT-translating guest code at runtime because there's no build pipeline to do it ahead
of time. Microsoft's official BC stack is what you build when you *do* have the source
and the original binaries: genuine Xbox 360 system code (kernel shim, dashboard, guide,
sign-in) recompiled rather than reimplemented, and the CPU-translation work front-loaded
into the build/packaging pipeline rather than paid at runtime. The two projects converge
strongly on *what* needs solving (PPC→host translation, Xenos GPU command/shader
translation with EDRAM handling, per-title compatibility overrides) while diverging
sharply on *how*, in exactly the way you'd expect from "official first-party reuse of
real assets" vs. "community reverse-engineered reimplementation."

---

*Comparison based on reading Xenia Canary's source/docs (`xenia-canary/`) against the
static binary analysis in [TECHNICAL_FINDINGS.md](TECHNICAL_FINDINGS.md). No code from
either project was built, run, or disassembled beyond what's already documented there.*
