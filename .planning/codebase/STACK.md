# Technology Stack

**Analysis Date:** 2026-03-26

## Languages

**Primary:**
- C# 12 targeting `net8.0` for the engine, platform layer, server, launcher, utility, and tests in `OpenRA.Game/`, `OpenRA.Platforms.Default/`, `OpenRA.Server/`, `OpenRA.Launcher/`, `OpenRA.Utility/`, and `OpenRA.Test/`.

**Secondary:**
- Lua 5.1 for mission and gameplay scripting in `mods/*/maps/*/*.lua` and `mods/*/scripts/*.lua`; syntax-checked by `Makefile` and `make.ps1`.
- YAML / MiniYAML for mod manifests, rules, installer manifests, and settings-oriented data in `mods/*/mod.yaml`, `mods/*/rules/*.yaml`, and `mods/*-content/installer/*.yaml`.
- Fluent FTL for localization bundles in `mods/*/fluent/*.ftl`.
- POSIX shell, Bash, PowerShell, and CMD for build, launch, install, and packaging automation in `Makefile`, `make.ps1`, `launch-*.sh`, `launch-*.cmd`, and `packaging/**/*.sh`.
- GLSL for rendering shaders in `glsl/*.frag` and `glsl/*.vert`.
- Objective-C/C and Python 3 for platform packaging helpers in `packaging/macos/*.m`, `packaging/macos/apphost.c`, `packaging/format-docs.py`, and `packaging/windows/fixlauncher.py`.

## Runtime

**Environment:**
- .NET 8 SDK is the build baseline; `Directory.Build.props` sets `TargetFramework` to `net8.0` and `LangVersion` to `12`, while `INSTALL.md` and `.github/workflows/ci.yml` both install/use `.NET 8`.
- Runtime executables are the generic launcher, dedicated server, and utility defined by `OpenRA.Launcher/OpenRA.Launcher.csproj`, `OpenRA.Server/OpenRA.Server.csproj`, and `OpenRA.Utility/OpenRA.Utility.csproj`.
- Native runtime dependencies are SDL2, OpenAL, FreeType, and Lua 5.1, provided either through NuGet-backed wrapper packages or through system libraries configured by `configure-system-libraries.sh`; see `INSTALL.md`, `OpenRA.Platforms.Default/OpenRA.Platforms.Default.csproj`, and `configure-system-libraries.sh`.

**Package Manager:**
- NuGet via `dotnet build`, `dotnet test`, and `dotnet publish`; Windows CI explicitly adds `https://api.nuget.org/v3/index.json` in `.github/workflows/ci.yml`.
- Dependency versions are declared directly in `Directory.Build.props` and the `*.csproj` files across the solution.
- Lockfile: not detected. `packages.lock.json`, `NuGet.Config`, and `global.json` are absent from the repo root.

## Frameworks

**Core:**
- SDK-style `.NET` projects (`Microsoft.NET.Sdk`) organized in `OpenRA.sln`.
- The custom OpenRA engine/mod runtime implemented in `OpenRA.Game/`, `OpenRA.Mods.Common/`, `OpenRA.Mods.Cnc/`, and `OpenRA.Mods.D2k/`.
- SDL2/OpenGL/OpenAL/FreeType wrapper stack for windowing, rendering, input, audio, and text through `OpenRA.Platforms.Default/OpenRA.Platforms.Default.csproj` and `glsl/`.
- Embedded Lua scripting through `OpenRA-Eluant` in `OpenRA.Game/OpenRA.Game.csproj` and the script/runtime code under `OpenRA.Game/Scripting/`.
- Fluent localization loading through `Linguini.Bundle` in `OpenRA.Game/OpenRA.Game.csproj` and `.ftl` bundles under `mods/*/fluent/`.

**Testing:**
- NUnit 4.3.2 with `Microsoft.NET.Test.Sdk`, `NUnit.Console`, and `NUnit3TestAdapter` in `OpenRA.Test/OpenRA.Test.csproj`.
- Lua syntax checks via `luac -p` wired into `Makefile` and `make.ps1`.
- MiniYAML validation and engine-specific lint commands executed through `OpenRA.Utility` from `Makefile` and `make.ps1`.

**Build/Dev:**
- GNU Make on Unix and `make.ps1` on Windows as the top-level developer interface in `Makefile` and `make.ps1`.
- `dotnet build`, `dotnet test`, and `dotnet publish` as the actual compilation, test, and publish toolchain in `Makefile` and `packaging/functions.sh`.
- Static analysis with `StyleCop.Analyzers`, `Roslynator.Analyzers`, and `Roslynator.Formatting.Analyzers` from `Directory.Build.props`.
- Platform packaging toolchains in `packaging/linux/buildpackage.sh`, `packaging/windows/buildpackage.sh`, and `packaging/macos/buildpackage.sh`.

## Key Dependencies

**Critical:**
- `OpenRA-SDL2-CS` 1.0.43 / 1.0.42 for cross-platform windowing, input, and launcher/native interop in `OpenRA.Platforms.Default/OpenRA.Platforms.Default.csproj` and `OpenRA.WindowsLauncher/OpenRA.WindowsLauncher.csproj`.
- `OpenRA-OpenAL-CS` 1.0.22 for the audio backend in `OpenRA.Platforms.Default/OpenRA.Platforms.Default.csproj`.
- `OpenRA-Freetype6` 1.0.11 for font rasterization in `OpenRA.Platforms.Default/OpenRA.Platforms.Default.csproj`.
- `OpenRA-Eluant` 1.0.22 for the embedded Lua runtime in `OpenRA.Game/OpenRA.Game.csproj`.
- `Linguini.Bundle` 0.8.1 for Fluent/FTL localization bundle loading in `OpenRA.Game/OpenRA.Game.csproj`.
- `Mono.NAT` 3.0.4 for UPnP/NAT-PMP discovery and port forwarding in `OpenRA.Game/OpenRA.Game.csproj` and `OpenRA.Game/Network/Nat.cs`.
- `DiscordRichPresence` 1.2.1.24 for local Discord Rich Presence support in `OpenRA.Mods.Common/OpenRA.Mods.Common.csproj` and `OpenRA.Mods.Common/DiscordService.cs`.

**Infrastructure:**
- `SharpZipLib` 1.4.2 for ZIP handling used by GeoIP and package workflows in `OpenRA.Game/OpenRA.Game.csproj` and `OpenRA.Game/Network/GeoIP.cs`.
- `rix0rrr.BeaconLib` 1.0.2 for LAN game discovery/advertising in `OpenRA.Mods.Common/OpenRA.Mods.Common.csproj` and `OpenRA.Mods.Common/ServerTraits/MasterServerPinger.cs`.
- `Microsoft.Extensions.DependencyModel` 9.0.0 and `System.Runtime.Loader` 4.3.0 for runtime assembly/mod loading in `OpenRA.Game/OpenRA.Game.csproj`.
- `MP3Sharp`, `NVorbis`, `TagLibSharp`, and `Pfim` for audio and image asset decoding in `OpenRA.Mods.Common/OpenRA.Mods.Common.csproj`.
- `OpenRA-FuzzyLogicLibrary` 1.0.1 for AI/fuzzy-logic support in `OpenRA.Mods.Common/OpenRA.Mods.Common.csproj`.

## Configuration

**Environment:**
- No `.env` files are detected in the repository. Runtime and build configuration come from command-line arguments, settings files in the support directory, and a small set of environment variables.
- Core runtime overrides include `ENGINE_DIR` and `MOD_SEARCH_PATHS` in `OpenRA.Utility/Program.cs` and `OpenRA.Server/Program.cs`.
- Platform/display overrides include `OPENRA_DISPLAY_SCALE`, `OPENRA_DESKTOP_FILENAME`, `GDK_SCALE`, and `XDG_CONFIG_HOME` in `OpenRA.Platforms.Default/Sdl2PlatformWindow.cs` and `OpenRA.Game/Platform.cs`.
- Optional release/integration variables include `ITCHIO_API_KEY` in `OpenRA.Mods.Common/ItchIntegration.cs` and signing/notarization variables consumed by `packaging/macos/buildpackage.sh` and `.github/workflows/packaging.yml`.
- User settings, logs, maps, replays, downloaded content, and auth profiles are stored under `Platform.SupportDir` as defined in `OpenRA.Game/Platform.cs`.

**Build:**
- Shared MSBuild settings live in `Directory.Build.props`.
- Solution and launcher configuration live in `OpenRA.sln`, `OpenRA.Launcher/Properties/launchSettings.json`, `OpenRA.Launcher/App.config`, and `OpenRA.WindowsLauncher/App.config`.
- Build/test wrappers live in `Makefile`, `make.ps1`, `launch-game.sh`, `launch-dedicated.sh`, and `utility.sh`.
- CI and packaging configuration live in `.github/workflows/*.yml`, `packaging/functions.sh`, `packaging/linux/buildpackage.sh`, `packaging/windows/buildpackage.sh`, and `packaging/macos/buildpackage.sh`.

## Platform Requirements

**Development:**
- Windows, macOS, and Linux are supported build targets per `INSTALL.md`, `Directory.Build.props`, and `Makefile`.
- The required baseline toolchain is `.NET 8 SDK`; Lua 5.1 is additionally required when running script syntax checks in `Makefile`, `make.ps1`, and `.github/workflows/ci.yml`.
- On Unix, `TARGETPLATFORM=unix-generic` expects system `SDL2`, `OpenAL`, `FreeType`, and `liblua 5.1`, with symlink setup handled by `configure-system-libraries.sh`.
- Packaging adds host-specific tools: AppImage tooling for Linux in `packaging/linux/buildpackage.sh`, `makensis`/`ImageMagick`/`wine64`/`python3` for Windows in `packaging/windows/buildpackage.sh`, and `clang`/`hdiutil`/`xcrun` for macOS in `packaging/macos/buildpackage.sh`.

**Production:**
- End-user builds are published as self-contained Windows launchers/installers, macOS `.app` bundles inside a DMG, and Linux AppImages or local installs; see `packaging/functions.sh`, `packaging/windows/buildpackage.sh`, `packaging/macos/buildpackage.sh`, and `packaging/linux/buildpackage.sh`.
- Dedicated server deployments run the `OpenRA.Server` executable with `Game.Mod`, `Server.*`, and `Engine.SupportDir` arguments as shown in `launch-dedicated.sh` and `launch-dedicated.cmd`.
- The repo is not arranged as a web service or container image. The production targets are desktop/game binaries plus a dedicated server binary.

---

*Stack analysis: 2026-03-26*
