# Codebase Structure

**Analysis Date:** 2026-03-26

## Directory Layout

```text
OpenRA/
├── OpenRA.Game/              # Shared engine library and runtime
├── OpenRA.Platforms.Default/ # SDL/OpenGL/OpenAL/Freetype platform adapter
├── OpenRA.Mods.Common/       # Shared gameplay, UI, installer, and tooling code
├── OpenRA.Mods.Cnc/          # C&C/RA/TS-specific gameplay and asset-loader code
├── OpenRA.Mods.D2k/          # Dune 2000-specific gameplay and asset-loader code
├── OpenRA.Launcher/          # Standard desktop launcher executable
├── OpenRA.WindowsLauncher/   # Windows wrapper launcher executable
├── OpenRA.Server/            # Dedicated server executable
├── OpenRA.Utility/           # Utility, import, lint, and documentation CLI
├── OpenRA.Test/              # NUnit test project
├── mods/                     # Data-driven official mods, shared packages, and content installers
├── glsl/                     # GLSL shader sources loaded by the default platform
├── packaging/                # Release packaging assets and per-OS packaging scripts
├── .github/workflows/        # CI, docs, and packaging automation
├── OpenRA.sln                # Solution wiring all .NET projects together
├── Directory.Build.props     # Shared MSBuild defaults and output-path settings
└── Makefile                  # Primary cross-platform build, test, and install entry point
```

## Directory Purposes

**OpenRA.Game/**
- Purpose: Shared engine and runtime library used by every executable and mod assembly.
- Contains: C# source in subsystem folders such as `Activities/`, `GameRules/`, `Graphics/`, `Map/`, `Network/`, `Server/`, `Traits/`, `Widgets/`
- Key files: `OpenRA.Game/Game.cs`, `OpenRA.Game/Manifest.cs`, `OpenRA.Game/ModData.cs`, `OpenRA.Game/World.cs`, `OpenRA.Game/FileSystem/FileSystem.cs`
- Subdirectories: `GameRules/` rule/model loaders, `Map/` map and preview/cache code, `Network/` client transport and order handling, `Server/` shared server runtime, `Graphics/` renderer-side abstractions, `Traits/` engine-level trait interfaces.

**OpenRA.Platforms.Default/**
- Purpose: Default runtime platform adapter loaded by `Game.CreatePlatform()`.
- Contains: Platform implementation files, native-interop helpers, and renderer/audio backends.
- Key files: `OpenRA.Platforms.Default/DefaultPlatform.cs`, `OpenRA.Platforms.Default/Sdl2PlatformWindow.cs`, `OpenRA.Platforms.Default/OpenAlSoundEngine.cs`, `OpenRA.Platforms.Default/OpenRA.Platforms.Default.dll.config`
- Subdirectories: None. The project is flat and organized by class name.

**OpenRA.Mods.Common/**
- Purpose: Shared gameplay, UI, installer, server, scripting, and utility extensions consumed by all official mods.
- Contains: Traits, activities, widget logic, load screens, file-system loaders, server traits, scripting globals/properties, sprite loaders, update helpers, and utility commands.
- Key files: `OpenRA.Mods.Common/FileSystem/ContentInstallerFileSystemLoader.cs`, `OpenRA.Mods.Common/LoadScreens/BlankLoadScreen.cs`, `OpenRA.Mods.Common/ServerTraits/LobbyCommands.cs`, `OpenRA.Mods.Common/Scripting/LuaScript.cs`, `OpenRA.Mods.Common/UtilityCommands/CheckYaml.cs`
- Subdirectories: `Traits/`, `Activities/`, `Widgets/Logic/`, `ServerTraits/`, `UtilityCommands/`, `Installer/`, `Scripting/`, `SpriteLoaders/`, `UpdateRules/`.

**OpenRA.Mods.Cnc/**
- Purpose: C&C-family runtime extensions shared by Tiberian Dawn, Red Alert, and Tiberian Sun data mods.
- Contains: C&C-specific traits, load screens, file and video loaders, widget logic, graphics helpers, and import utilities.
- Key files: `OpenRA.Mods.Cnc/CncLoadScreen.cs`, `OpenRA.Mods.Cnc/FileSystem/MixFile.cs`, `OpenRA.Mods.Cnc/VideoLoaders/VqaLoader.cs`, `OpenRA.Mods.Cnc/UtilityCommands/ImportTiberianSunMapCommand.cs`
- Subdirectories: `Traits/`, `Widgets/Logic/`, `UtilityCommands/`, `SpriteLoaders/`, `VideoLoaders/`, `Graphics/`, `Installer/`.

**OpenRA.Mods.D2k/**
- Purpose: Dune 2000-specific runtime extensions and asset loaders.
- Contains: D2K traits, graphics sequence loaders, package loaders, widget logic, projectiles, warheads, and import utilities.
- Key files: `OpenRA.Mods.D2k/Graphics/D2kSpriteSequence.cs`, `OpenRA.Mods.D2k/PackageLoaders/D2kSoundResources.cs`, `OpenRA.Mods.D2k/UtilityCommands/ImportD2kMapCommand.cs`
- Subdirectories: `Traits/`, `Widgets/Logic/`, `PackageLoaders/`, `SpriteLoaders/`, `UtilityCommands/`, `Warheads/`.

**OpenRA.Launcher/**
- Purpose: Standard interactive client launcher executable.
- Contains: Minimal host code plus launch settings metadata.
- Key files: `OpenRA.Launcher/Program.cs`, `OpenRA.Launcher/App.config`, `OpenRA.Launcher/Properties/launchSettings.json`
- Subdirectories: `Properties/` for IDE and local launch profiles.

**OpenRA.WindowsLauncher/**
- Purpose: Windows wrapper executable for packaged builds and fatal-error dialogs.
- Contains: A single host class and app config.
- Key files: `OpenRA.WindowsLauncher/Program.cs`, `OpenRA.WindowsLauncher/App.config`
- Subdirectories: None.

**OpenRA.Server/**
- Purpose: Dedicated server executable.
- Contains: A single host class plus project file.
- Key files: `OpenRA.Server/Program.cs`, `OpenRA.Server/OpenRA.Server.csproj`
- Subdirectories: None.

**OpenRA.Utility/**
- Purpose: Utility CLI host that discovers `IUtilityCommand` implementations from the selected mod.
- Contains: A single host class, project file, and launch settings.
- Key files: `OpenRA.Utility/Program.cs`, `OpenRA.Utility/Properties/launchSettings.json`
- Subdirectories: `Properties/`.

**OpenRA.Test/**
- Purpose: Automated test project for engine code and selected shared mod code.
- Contains: NUnit tests grouped by referenced assembly.
- Key files: `OpenRA.Test/OpenRA.Test.csproj`, `OpenRA.Test/OpenRA.Game/*.cs`, `OpenRA.Test/OpenRA.Mods.Common/*.cs`
- Subdirectories: `OpenRA.Game/`, `OpenRA.Mods.Common/`.

**mods/**
- Purpose: Shipped data-driven mods, shared data packages, and content-installer mods.
- Contains: `mod.yaml`, YAML rules/chrome/hotkeys/weapons, Fluent translations, maps, scripts, UI bits, and installer manifests.
- Key files: `mods/all/mod.yaml`, `mods/ra/mod.yaml`, `mods/cnc/mod.yaml`, `mods/d2k/mod.yaml`, `mods/ts/mod.yaml`, `mods/ra-content/mod.yaml`
- Subdirectories: `common/` shared chrome/scripts/hotkeys, `common-content/` shared installer assets, `ra/`, `cnc/`, `d2k/`, `ts/` playable mod data, `*-content/` installer mods, `all/` hidden aggregate mod.

**packaging/**
- Purpose: Release packaging, installer templates, icons, and distribution scripts.
- Contains: Shell, Python, NSIS, Objective-C, and C helper files plus artwork.
- Key files: `packaging/package-all.sh`, `packaging/functions.sh`, `packaging/linux/buildpackage.sh`, `packaging/macos/buildpackage.sh`, `packaging/windows/buildpackage.sh`, `packaging/windows/OpenRA.nsi`
- Subdirectories: `linux/`, `macos/`, `windows/`, `source/`, `artwork/`.

**glsl/**
- Purpose: Shader programs consumed by the default platform renderer.
- Contains: Vertex and fragment shader source files.
- Key files: `glsl/combined.vert`, `glsl/combined.frag`, `glsl/model.vert`, `glsl/postprocess_textured_vortex.frag`
- Subdirectories: None.

**.github/workflows/**
- Purpose: Repository automation for CI, docs, packaging, and itch publishing.
- Contains: GitHub Actions workflow YAML files.
- Key files: `.github/workflows/ci.yml`, `.github/workflows/documentation.yml`, `.github/workflows/packaging.yml`, `.github/workflows/itch.yml`
- Subdirectories: None.

## Key File Locations

**Entry Points:**
- `launch-game.sh`: POSIX launcher wrapper for the interactive client.
- `launch-game.cmd`: Windows launcher wrapper for the interactive client.
- `OpenRA.Launcher/Program.cs`: Primary client entry point.
- `OpenRA.WindowsLauncher/Program.cs`: Packaged Windows wrapper entry point.
- `OpenRA.Server/Program.cs`: Dedicated server entry point.
- `launch-dedicated.sh`: POSIX dedicated-server wrapper.
- `OpenRA.Utility/Program.cs`: Utility CLI entry point.
- `utility.sh`: POSIX utility wrapper.

**Configuration:**
- `Directory.Build.props`: Shared framework, analyzer, and output-path settings.
- `.editorconfig`: Repository-wide formatting and code-style policy.
- `OpenRA.sln`: Solution graph for all projects.
- `Makefile`: Cross-platform build, lint, test, and install orchestration.
- `OpenRA.Platforms.Default/OpenRA.Platforms.Default.dll.config`: Native binding config for the default platform adapter.
- `mods/*/mod.yaml`: Per-mod manifests that declare assemblies, loaders, content, and runtime data files.

**Core Logic:**
- `OpenRA.Game/`: Engine bootstrap, manifests, world simulation, networking, rendering, UI, settings, and file system code.
- `OpenRA.Mods.Common/`: Shared mod code for traits, widgets, load screens, installers, scripting, and utility commands.
- `OpenRA.Mods.Cnc/`: C&C-family runtime extensions used by `mods/cnc`, `mods/ra`, and `mods/ts`.
- `OpenRA.Mods.D2k/`: Dune 2000 runtime extensions used by `mods/d2k`.
- `mods/`: Data packs, rules, chrome layouts, translations, maps, and scripts that feed the engine at runtime.
- `glsl/`: Shader source files loaded by the default platform renderer.

**Testing:**
- `OpenRA.Test/`: NUnit tests for engine and shared mod code.
- `.github/workflows/ci.yml`: CI execution path for `make check`, `make tests`, `make check-scripts`, and `make test`.

**Documentation:**
- `README.md`: Repository overview and contribution entry point.
- `INSTALL.md`: Build and run instructions for supported platforms.
- `CONTRIBUTING.md`: Patch and review workflow expectations.
- `COPYING`: Project license text.

## Naming Conventions

**Files:**
- `PascalCase.cs`: C# source files generally match the primary type name, e.g. `OpenRA.Game/World.cs`, `OpenRA.Mods.Common/Scripting/LuaScript.cs`.
- `Program.cs`: Executable host entry points, e.g. `OpenRA.Launcher/Program.cs`, `OpenRA.Server/Program.cs`.
- `<ProjectName>.csproj`: Project files mirror the directory name, e.g. `OpenRA.Game/OpenRA.Game.csproj`.
- `mod.yaml`, `map.yaml`, `rules/*.yaml`, `chrome/*.yaml`, `weapons/*.yaml`: Data-driven runtime configuration and content manifests under `mods/`.
- `*.sh`, `*.cmd`, `make.ps1`: Platform-specific launch, build, and utility wrappers at the repo root.

**Directories:**
- Dotted `PascalCase` project roots for .NET assemblies, e.g. `OpenRA.Game/`, `OpenRA.Mods.Common/`, `OpenRA.Platforms.Default/`.
- `PascalCase` subsystem folders inside projects, e.g. `Graphics/`, `Network/`, `Widgets/Logic/`, `Traits/World/`.
- Lowercase mod ids for data packages under `mods/`, e.g. `mods/ra/`, `mods/cnc/`, `mods/d2k/`, `mods/ts/`.
- Lowercase `-content` suffix for installer-only mods, e.g. `mods/ra-content/`, `mods/ts-content/`.

**Special Patterns:**
- `Traits/<domain>/` folders hold runtime behavior types that are referenced from YAML by trait name and usually pair a `*Info` class with a runtime class in the same file, e.g. `OpenRA.Mods.Common/Traits/World/SpawnMapActors.cs`.
- `Widgets/Logic/` holds UI logic classes while chrome layouts live in `mods/*/chrome/*.yaml`.
- `*Loader.cs` files implement package, file-system, sprite, sequence, sound, or video loader interfaces chosen by manifest strings.
- `maps/<map-id>/` packages typically contain `map.yaml`, `map.bin`, optional `*.lua`, optional `map.png`, and map-specific rule overrides.
- `Properties/launchSettings.json` appears only in executable projects that define local debug launch profiles.

## Where to Add New Code

**New Engine Feature:**
- Primary code: `OpenRA.Game/<subsystem>/`
- Tests: `OpenRA.Test/OpenRA.Game/`
- Config if needed: `mods/*/mod.yaml` or mod YAML under `mods/<mod-id>/` if the feature must be wired into manifests or data.

**New Shared Gameplay or UI Component:**
- Implementation: `OpenRA.Mods.Common/Traits/`, `OpenRA.Mods.Common/Activities/`, `OpenRA.Mods.Common/Widgets/Logic/`, `OpenRA.Mods.Common/ServerTraits/`, or `OpenRA.Mods.Common/Scripting/`
- Types and runtime wiring: same file or nearby domain folder inside `OpenRA.Mods.Common/`
- Tests: `OpenRA.Test/OpenRA.Mods.Common/` if the behavior is testable without full runtime assets.

**New RA / CNC / TS-Specific Feature:**
- Implementation: `OpenRA.Mods.Cnc/`
- Data and config: `mods/cnc/`, `mods/ra/`, or `mods/ts/`
- Tests: `OpenRA.Test/` only when the logic can be isolated into a focused unit test.

**New D2K-Specific Feature:**
- Implementation: `OpenRA.Mods.D2k/`
- Data and config: `mods/d2k/`
- Tests: `OpenRA.Test/` only when the logic can be isolated into a focused unit test.

**New Tool or Utility Command:**
- Definition: `OpenRA.Game/UtilityCommands/`, `OpenRA.Mods.Common/UtilityCommands/`, `OpenRA.Mods.Cnc/UtilityCommands/`, or `OpenRA.Mods.D2k/UtilityCommands/` depending scope
- Host: `OpenRA.Utility/Program.cs` already discovers commands reflectively
- Tests: `OpenRA.Test/` if the core logic can be extracted and asserted independently.

**New Mod Data, UI Asset, or Map:**
- Playable mod files: `mods/<mod-id>/`
- Content-installer files: `mods/<mod-id>-content/`
- Shared chrome, scripts, and hotkeys: `mods/common/` or `mods/common-content/`
- Maps: `mods/<mod-id>/maps/<map-id>/`

## Special Directories

**mods/common/**
- Purpose: Shared data package referenced by multiple playable mods.
- Source: Mounted from playable manifests such as `mods/ra/mod.yaml` and `mods/cnc/mod.yaml`
- Committed: Yes

**mods/common-content/**
- Purpose: Shared content-installer chrome, metrics, cursors, and translations.
- Source: Mounted by hidden installer mods such as `mods/ra-content/mod.yaml` and `mods/cnc-content/mod.yaml`
- Committed: Yes

**mods/all/**
- Purpose: Hidden aggregate mod that loads all official code assemblies for utility, lint, and documentation workflows.
- Source: `mods/all/mod.yaml`
- Committed: Yes

**bin/**
- Purpose: Compiled output directory for all projects.
- Source: Generated by MSBuild per `Directory.Build.props`
- Committed: No, ignored in `.gitignore`

**obj/**
- Purpose: Per-project intermediate build artifacts.
- Source: Generated by MSBuild
- Committed: No, ignored in `.gitignore`

**Support/**
- Purpose: Optional local support, content, and log directory when present beside the engine root.
- Source: Runtime-created and runtime-managed outside source control
- Committed: No, ignored in `.gitignore`

---

*Structure analysis: 2026-03-26*
