# Architecture

**Analysis Date:** 2026-03-26

## Pattern Overview

**Overall:** Data-driven RTS engine with thin executable hosts and reflection-loaded mod/platform plugins

**Key Characteristics:**
- `OpenRA.Game` is the shared engine/runtime used by the launcher, dedicated server, utility CLI, tests, and mod assemblies.
- Playable behavior is assembled at runtime from `mod.yaml`, YAML rules/chrome/maps, and reflection-discovered C# types instead of hardcoded per-game executables.
- Client, local server, dedicated server, utility tooling, and content installers reuse the same manifest, file-system, ruleset, and trait abstractions.
- Platform APIs are isolated behind `IPlatform` and loaded from `OpenRA.Platforms.*.dll` at runtime.

## Layers

**Bootstrap / Host Layer:**
- Purpose: Parse process arguments, set up fatal error handling, choose a mod, and hand off to shared engine code or tool workflows.
- Location: `OpenRA.Launcher/Program.cs`, `OpenRA.WindowsLauncher/Program.cs`, `OpenRA.Server/Program.cs`, `OpenRA.Utility/Program.cs`, `launch-game.sh`, `launch-dedicated.sh`, `utility.sh`
- Contains: Thin `Main` methods, shell wrappers, Windows wrapper process management.
- Depends on: `OpenRA.Game`, root scripts, `Directory.Build.props`
- Used by: End users, dedicated server operators, CI/package scripts.

**Platform Integration Layer:**
- Purpose: Abstract windowing, input, rendering context, audio, and font services behind a runtime-selected platform assembly.
- Location: `OpenRA.Game/Platform.cs`, `OpenRA.Game/Graphics/PlatformInterfaces.cs`, `OpenRA.Platforms.Default/DefaultPlatform.cs`, `OpenRA.Platforms.Default/Sdl2PlatformWindow.cs`, `OpenRA.Platforms.Default/OpenAlSoundEngine.cs`
- Contains: `IPlatform` contracts, support/bin directory discovery, SDL/OpenGL/OpenAL/Freetype bindings, hardware cursor and graphics context code.
- Depends on: `OpenRA.Game`, native libraries shipped via NuGet or system packages, shader sources in `glsl/`
- Used by: `OpenRA.Game/Game.cs` during `CreatePlatform()` and renderer/sound initialization.

**Core Engine Layer:**
- Purpose: Own global process state, mod loading, rules parsing, world creation, actor composition, networking, rendering, UI, settings, and logging.
- Location: `OpenRA.Game/`
- Contains: `Game.cs`, `Manifest.cs`, `ModData.cs`, `ObjectCreator.cs`, `World.cs`, `Actor.cs`, `GameRules/*.cs`, `Map/*.cs`, `Network/*.cs`, `Server/*.cs`, `Graphics/*.cs`, `Widgets/*.cs`, `Sound/*.cs`, `Support/*.cs`
- Depends on: platform contracts, file/package loaders, reflection, YAML parsing, shared primitives.
- Used by: all executables and every mod assembly.

**Mod Runtime Code Layer:**
- Purpose: Supply reusable gameplay, UI, installer, server, scripting, and asset-loader extensions that the engine instantiates from manifest strings and YAML trait names.
- Location: `OpenRA.Mods.Common/`, `OpenRA.Mods.Cnc/`, `OpenRA.Mods.D2k/`
- Contains: trait implementations, activities, projectiles, widget logic, load screens, package/sprite/video loaders, server traits, scripting globals/properties, utility commands.
- Depends on: `OpenRA.Game`
- Used by: playable mods via `Assemblies:` in `mods/*/mod.yaml`. `mods/ra/mod.yaml` and `mods/ts/mod.yaml` both load `OpenRA.Mods.Cnc.dll`; `mods/d2k/mod.yaml` loads `OpenRA.Mods.D2k.dll`.

**Content / Data Layer:**
- Purpose: Define playable mods, shared assets, chrome layouts, rules, maps, translations, and content-install manifests without recompiling the engine.
- Location: `mods/`, `glsl/`
- Contains: `mod.yaml`, `rules/*.yaml`, `weapons/*.yaml`, `chrome/*.yaml`, `fluent/*.ftl`, `maps/*/map.yaml`, `maps/*/map.bin`, optional map Lua, packaged art/audio.
- Depends on: engine manifest/file-system/rules loaders and mod runtime assemblies.
- Used by: `Manifest`, `FileSystem`, `Ruleset`, `WidgetLoader`, `ChromeProvider`, `MapCache`, installer load screens.

**Tooling / Verification / Release Layer:**
- Purpose: Build, lint, test, document, package, and distribute the repo.
- Location: `Makefile`, `make.ps1`, `OpenRA.Test/`, `packaging/`, `.github/workflows/`
- Contains: solution-wide build targets, NUnit tests, packaging scripts for Linux/macOS/Windows/source, CI workflows.
- Depends on: solution projects, launch/utility scripts, packaging templates.
- Used by: contributors, CI, release automation.

## Data Flow

**Game Startup (client launcher):**

1. The user runs `launch-game.sh` or `launch-game.cmd`, or starts `OpenRA.Launcher`.
2. `OpenRA.Launcher/Program.cs` or `OpenRA.WindowsLauncher/Program.cs` calls `Game.InitializeAndRun(args)`.
3. `OpenRA.Game/Game.cs` resolves `EngineDir` / `SupportDir`, initializes `Settings`, log channels, NAT, installed mods, and platform bindings.
4. `Game.CreatePlatform()` loads `OpenRA.Platforms.Default.dll` (or another configured platform assembly) and creates renderer and sound services.
5. `Game.InitializeMod()` constructs `ModData`, which rebuilds the `Manifest`, mounts packages, loads global mod data, chrome, widgets, hotkeys, and map cache.
6. The mod load screen decides whether to start a replay/server/map immediately or fall back to `Game.LoadShellMap()`.
7. `Game.StartGame()` creates `World` and `WorldRenderer`, then the main loop in `Game.Run()` advances logic and render ticks.

**Mod Materialization:**

1. `InstalledMods` finds mod folders such as `mods/ra/`, `mods/cnc/`, `mods/d2k/`, `mods/ts/`, and hidden installer mods like `mods/ra-content/`.
2. `Manifest` reads `mod.yaml`, expands `Include` nodes, and records assembly names, file-system loader, rules/chrome/maps/hotkeys lists, loader formats, and server trait names.
3. `ModData` builds an `ObjectCreator`, mounts packages through `DefaultFileSystemLoader` or `ContentInstallerFileSystemLoader`, and instantiates `IGlobalModData` modules from manifest sections.
4. `ObjectCreator` resolves trait types, widget classes, loaders, server traits, and utility commands across `OpenRA.Game.dll` plus the assemblies listed in `Assemblies:`.
5. `Ruleset`, `WidgetLoader`, `ChromeProvider`, `SequenceSet`, `MapCache`, and `HotkeyManager` hydrate runtime structures from YAML and package contents.

**Simulation / Rendering Loop:**

1. `OrderManager` receives network/replay/local orders and advances synchronized frame state.
2. `World` owns actors, players, effects, map state, and world actor traits for the current match, editor, or shellmap.
3. Each `Actor` is composed from `ActorInfo` plus ordered `TraitInfo` instances, then cached by `TraitDictionary` for fast trait queries.
4. Activities, orders, and traits mutate synchronized world state; `Sync` can hash marked fields to detect desyncs.
5. `WorldRenderer` queries visible actors/effects, resolves palettes and sequences, and emits terrain, actor, overlay, and annotation renderables.
6. `Ui`, `WidgetLoader`, and chrome YAML drive menus, dialogs, HUD, and installer screens on top of the world render.

**Dedicated Server Runtime:**

1. `launch-dedicated.sh` or direct execution starts `OpenRA.Server/Program.cs`.
2. The server entry point initializes settings/logging, resolves the selected mod, and creates `ModData` without client rendering concerns.
3. `OpenRA.Game/Server/Server.cs` binds listeners, constructs `ServerTrait` instances from `Manifest.ServerTraits`, and manages lobby state, handshake/auth, map status, and replay recording.
4. When games start, the same rules, maps, and manifests drive server-side validation and order processing as on the client.

**State Management:**
- Process-global state lives in static holders such as `OpenRA.Game/Game.cs`, `OpenRA.Game/Platform.cs`, and `OpenRA.Game/Support/Log.cs`.
- Per-mod state lives in `ModData`, `Manifest`, mounted packages, loader registries, and cached map previews.
- Per-match state lives in `World`, `Map`, `OrderManager`, `Session`, and `WorldRenderer`.
- Persistent user and operator state is file-based under `Platform.SupportDir`, including `settings.yaml`, logs, downloaded content, maps, replays, screenshots, and auth profiles.

## Key Abstractions

**Manifest / ModData:**
- Purpose: Turn a mod folder into a runnable engine configuration.
- Examples: `OpenRA.Game/Manifest.cs`, `OpenRA.Game/ModData.cs`, `mods/ra/mod.yaml`, `mods/cnc-content/mod.yaml`
- Pattern: Manifest-driven runtime assembly; YAML declares files, loaders, assemblies, traits, and content-installer behavior.

**ObjectCreator:**
- Purpose: Reflection-based resolver for gameplay and plugin types.
- Examples: `OpenRA.Game/ObjectCreator.cs`, `OpenRA.Mods.Common/LoadScreens/BlankLoadScreen.cs`, `OpenRA.Mods.Common/ServerTraits/LobbyCommands.cs`
- Pattern: Name-to-type resolution using conventions such as `FooInfo`, `FooLoader`, and `FooWidget` across loaded assemblies.

**ActorInfo / TraitInfo / Actor:**
- Purpose: Compose units, buildings, world actors, and player actors from many small behaviors.
- Examples: `OpenRA.Game/GameRules/ActorInfo.cs`, `OpenRA.Game/Actor.cs`, `OpenRA.Mods.Common/Traits/World/SpawnMapActors.cs`, `OpenRA.Mods.Common/Scripting/LuaScript.cs`
- Pattern: YAML-defined component model with dependency-ordered trait construction and interface-based dispatch.

**World / OrderManager:**
- Purpose: Separate synchronized simulation state from transport/order buffering and lobby/session metadata.
- Examples: `OpenRA.Game/World.cs`, `OpenRA.Game/Network/OrderManager.cs`, `OpenRA.Game/Network/Connection.cs`, `OpenRA.Game/Server/Server.cs`
- Pattern: Deterministic lockstep simulation with client, replay, local, and dedicated-server backends.

**Virtual File System / Loaders:**
- Purpose: Mount base engine data, shared data packages, installed game content, maps, and package formats behind a unified path model.
- Examples: `OpenRA.Game/FileSystem/FileSystem.cs`, `OpenRA.Mods.Common/FileSystem/DefaultFileSystemLoader.cs`, `OpenRA.Mods.Common/FileSystem/ContentInstallerFileSystemLoader.cs`, `OpenRA.Mods.Cnc/FileSystem/MixFile.cs`, `OpenRA.Mods.D2k/PackageLoaders/D2kSoundResources.cs`
- Pattern: Manifest-selected loader plus path alias layer using prefixes such as `^EngineDir`, `^SupportDir`, `$ra`, and `ra|...`.

**Widget / Chrome System:**
- Purpose: Build UI from declarative layout YAML and runtime logic objects.
- Examples: `OpenRA.Game/Widgets/WidgetLoader.cs`, `OpenRA.Game/Graphics/ChromeProvider.cs`, `mods/common/chrome/*.yaml`, `mods/ra/chrome/*.yaml`, `OpenRA.Mods.Common/Widgets/Logic/`
- Pattern: Data-defined widget trees with logic objects injected via `WidgetArgs` and reflection.

## Entry Points

**Game Launcher:**
- Location: `OpenRA.Launcher/Program.cs`
- Triggers: `launch-game.sh`, `launch-game.cmd`, IDE/project launch profiles.
- Responsibilities: Start the interactive client, delegate to `Game.InitializeAndRun()`, and route fatal errors through `ExceptionHandler`.

**Windows Wrapper Launcher:**
- Location: `OpenRA.WindowsLauncher/Program.cs`
- Triggers: Windows packaged builds that need a GUI wrapper and crash dialog.
- Responsibilities: Optionally relaunch itself with `Engine.LaunchPath=...`, embed mod metadata, and surface FAQ/log actions on fatal errors.

**Dedicated Server:**
- Location: `OpenRA.Server/Program.cs`
- Triggers: `launch-dedicated.sh`, `launch-dedicated.cmd`, direct `dotnet` execution.
- Responsibilities: Build a headless `ModData`, initialize server settings/listeners, and restart fresh server instances after matches end.

**Utility CLI:**
- Location: `OpenRA.Utility/Program.cs`
- Triggers: `utility.sh`, `utility.cmd`, build/test/package scripts.
- Responsibilities: Resolve a mod, reflectively discover `IUtilityCommand` implementations, and execute lint, import, documentation, and update helpers.

**Build / CI Entrypoints:**
- Location: `Makefile`, `make.ps1`, `.github/workflows/ci.yml`, `packaging/package-all.sh`
- Triggers: developer builds, CI runs, release packaging.
- Responsibilities: compile all projects, run tests and YAML/Lua checks, and build distributable packages.

## Error Handling

**Strategy:** Throw descriptive exceptions at engine and plugin boundaries, catch them at process entry points, write structured logs under `Platform.SupportDir`, and abort on invalid runtime state instead of attempting partial recovery.

**Patterns:**
- Fatal process errors are routed through `OpenRA.Game/Support/ExceptionHandler.cs` from `OpenRA.Launcher/Program.cs`, `OpenRA.WindowsLauncher/Program.cs`, and `OpenRA.Server/Program.cs`.
- Data and config errors surface as `YamlException`, `InvalidDataException`, or missing-type failures during manifest, rules, map, or widget loading in `OpenRA.Game/Manifest.cs`, `OpenRA.Game/GameRules/Ruleset.cs`, `OpenRA.Game/Map/Map.cs`, and `OpenRA.Game/Widgets/WidgetLoader.cs`.
- Multiplayer consistency errors stop the match via sync hashing and sync reports in `OpenRA.Game/Sync.cs` and `OpenRA.Game/Network/OrderManager.cs`.
- Non-fatal platform and audio initialization failures degrade to fallback behavior where possible, e.g. `OpenRA.Platforms.Default/DefaultPlatform.cs` swaps to `DummySoundEngine`.

## Cross-Cutting Concerns

**Logging:**
- `OpenRA.Game/Support/Log.cs` provides asynchronous multi-channel logging to `SupportDir/Logs`.
- Entry points add domain-specific channels such as `debug`, `perf`, `server`, `nat`, `geoip`, `utility`, and `exception` before substantial work begins.

**Validation:**
- `FieldLoader` and `FieldSaver` enforce YAML-to-object contracts across manifests, traits, maps, widgets, and settings.
- `OpenRA.Game/GameRules/ActorInfo.cs` orders trait construction by declared dependencies.
- `OpenRA.Mods.Common/UtilityCommands/CheckYaml.cs` and the `make test` / `make check` flows provide repo-wide offline validation for mods and code-generated docs.

**Authentication:**
- Client identity and profile state is persisted via `LocalPlayerProfile` and `PlayerDatabase` created in `OpenRA.Game/Game.cs`.
- Server-side auth gates are configured in `OpenRA.Game/Settings.cs` and enforced in `OpenRA.Game/Server/Server.cs` and `OpenRA.Mods.Common/ServerTraits/LobbyCommands.cs`.

**Content Installation:**
- Playable mods that depend on original game data use `ContentInstallerFileSystemLoader` and hidden `*-content` mods to switch into installer UX when required assets are missing.
- This path is declared in playable manifests such as `mods/ra/mod.yaml`, `mods/cnc/mod.yaml`, `mods/d2k/mod.yaml`, and `mods/ts/mod.yaml`.

---

*Architecture analysis: 2026-03-26*
