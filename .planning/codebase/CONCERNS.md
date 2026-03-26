# Codebase Concerns

**Analysis Date:** 2026-03-26

## Tech Debt

**Static `Game` state leaked into headless and utility workflows:**
- Issue: server startup, map importers, documentation extractors, update commands, and lint/diagnostic utilities still have to populate `Game.Settings` and `Game.ModData` to satisfy engine assumptions.
- Files: `OpenRA.Server/Program.cs`, `OpenRA.Mods.Common/UtilityCommands/UpdateModCommand.cs`, `OpenRA.Mods.Common/UtilityCommands/UpdateMapCommand.cs`, `OpenRA.Mods.Common/UtilityCommands/MapCommand.cs`, `OpenRA.Mods.Common/UtilityCommands/CheckYaml.cs`, `OpenRA.Mods.Common/UtilityCommands/FuzzMapGeneratorCommand.cs`, `OpenRA.Mods.Cnc/UtilityCommands/ImportGen1MapCommand.cs`, `OpenRA.Mods.Cnc/UtilityCommands/ImportGen2MapCommand.cs`, `OpenRA.Mods.Cnc/UtilityCommands/LegacyRulesImporter.cs`, `OpenRA.Mods.Cnc/UtilityCommands/LegacySequenceImporter.cs`, `OpenRA.Mods.Cnc/UtilityCommands/LegacyTilesetImporter.cs`, `OpenRA.Mods.D2k/UtilityCommands/ImportD2kMapCommand.cs`
- Why: engine services were built around global singletons instead of explicit dependency injection.
- Impact: isolated tooling, background processing, and testability are all weaker than they should be; future refactors can easily break server or utility mode while gameplay still works.
- Fix approach: pass settings and `ModData` through explicit service boundaries, then remove static reads from shared engine code before changing server or utility command flows.

**Renderer ownership and palette handling are held together by compatibility hacks:**
- Issue: palette lookup is duplicated across renderables because palettes live on traits instead of sequences, old overlay grouping is preserved by a hack, and `WorldRenderer.Dispose()` still disposes the `World` even though it does not own it.
- Files: `OpenRA.Game/Graphics/WorldRenderer.cs`, `OpenRA.Game/Graphics/SpriteRenderer.cs`, `OpenRA.Game/Graphics/SpriteRenderable.cs`, `OpenRA.Game/Graphics/TerrainSpriteLayer.cs`, `OpenRA.Game/Graphics/UISpriteRenderable.cs`
- Why: rendering abstractions grew around legacy trait-driven palette selection and long-lived ownership assumptions.
- Impact: rendering changes are cross-cutting, disposal order is fragile, and seemingly local palette changes often require touching multiple engine layers.
- Fix approach: move palette ownership closer to sequence/render data, separate world lifetime from renderer lifetime, and delete the duplicated RGBA palette workarounds in one coordinated pass.

**Aircraft, docking, and resupply behavior depend on cancellation-era workarounds:**
- Issue: repair, rearm, docking, landing, and takeoff rules are distributed across activities with explicit hacks for cancel paths, host reservation, height normalization, and preview/render edge cases.
- Files: `OpenRA.Mods.Common/Activities/Resupply.cs`, `OpenRA.Mods.Common/Activities/Air/Fly.cs`, `OpenRA.Mods.Common/Activities/Air/Land.cs`, `OpenRA.Mods.Common/Traits/Repairable.cs`, `OpenRA.Mods.Cnc/Traits/Render/WithVoxelUnloadBody.cs`
- Why: aircraft support, repair docking, and voxel-specific presentation were layered onto the activity system incrementally.
- Impact: future behavior changes around aircraft, repair pads, or docking can introduce stuck units, blocking on host footprints, or incorrect altitude transitions.
- Fix approach: centralize docking and resupply into a single state model before modifying landing, repair, or cancel logic again.

**Legacy content tooling is still important but intentionally hard to maintain:**
- Issue: classic map importers and Westwood-era crypto/format helpers are explicitly described as bitrotted or direct C ports.
- Files: `OpenRA.Mods.Cnc/UtilityCommands/ImportGen1MapCommand.cs`, `OpenRA.Mods.Cnc/UtilityCommands/ImportGen2MapCommand.cs`, `OpenRA.Mods.Cnc/UtilityCommands/LegacyRulesImporter.cs`, `OpenRA.Mods.Cnc/UtilityCommands/LegacySequenceImporter.cs`, `OpenRA.Mods.Cnc/UtilityCommands/LegacyTilesetImporter.cs`, `OpenRA.Mods.Cnc/FileFormats/BlowfishKeyProvider.cs`
- Why: these paths preserve compatibility with older assets and tooling, but are not a day-to-day gameplay focus.
- Impact: importer or archive changes are expensive to reason about, and regressions are likely to surface only when niche migration workflows are used.
- Fix approach: either deprecate low-value import flows or add fixture-based tests around the supported ones before making algorithmic changes.

## Known Bugs

**Multiplayer save/load remains incomplete and singleplayer-biased:**
- Symptoms: save metadata assumes spectator orders belong to the first bot, humans are auto-filled into saved slots, and viewport state is stored on the first bot trait.
- Files: `OpenRA.Game/Network/GameSave.cs`, `OpenRA.Game/Server/Server.cs`, `OpenRA.Mods.Common/Traits/Player/GameSaveViewportManager.cs`, `OpenRA.Game/World.cs`
- Trigger: attempting multiplayer saves, expanding save support beyond the current singleplayer/non-dedicated use case, or changing bot/spectator handling.
- Workaround: keep save/load work scoped to the currently supported singleplayer path; do not assume multiplayer save semantics exist.
- Root cause: the current protocol does not carry enough player/bot ownership information, and several save-path assumptions are explicitly marked as TODOs for multiplayer.
- Blocked by: a redesign of save/load identity mapping, spectator handling, and lobby slot restoration.

**Audio pause/resume can race with asynchronous track loading:**
- Symptoms: music can play or rewind from the wrong state if the user pauses or resumes at exactly the wrong point during async loading.
- Files: `OpenRA.Platforms.Default/OpenAlSoundEngine.cs`
- Trigger: pausing/resuming while a new source is being attached after the silent placeholder buffer.
- Workaround: user retry; the code comments describe the consequences as minor, but it is still a real race.
- Root cause: source state is checked and then acted on in a non-atomic sequence.

**Host/admin identity still leaks into player ownership paths:**
- Symptoms: map-owned players default to the admin client index, admin reassignment is incomplete once the game has started, and local-server endpoint handling contains a documented PlayerIndex race if multiple loopback endpoints are exposed.
- Files: `OpenRA.Game/Player.cs`, `OpenRA.Game/Server/Server.cs`, `OpenRA.Game/Game.cs`
- Trigger: admin disconnects, save/load remapping, or endpoint changes in local server creation.
- Workaround: current code limits local servers to a single loopback endpoint and only reassigns admin during waiting-lobby state on dedicated servers.
- Root cause: player identity, host/admin privileges, and transport endpoint selection are still entangled in the server state machine.

## Security Considerations

**The asset and installer parser surface is large, custom, and only partly covered by tests:**
- Risk: malformed PNGs, CAB archives, VQA videos, compression streams, or legacy asset formats can still crash the process or cause denial-of-service behavior during import/load paths.
- Files: `OpenRA.Game/FileFormats/Png.cs`, `OpenRA.Mods.Common/FileFormats/InstallShieldCABCompression.cs`, `OpenRA.Mods.Cnc/FileFormats/VqaVideo.cs`, `OpenRA.Mods.Cnc/FileFormats/LZOCompression.cs`, `OpenRA.Mods.Cnc/FileFormats/BlowfishKeyProvider.cs`, `OpenRA.Test/OpenRA.Game/PngTest.cs`
- Current mitigation: loaders do reject bad input in places with explicit `InvalidDataException` and length checks, and PNG parsing has dedicated unit tests in `OpenRA.Test/OpenRA.Game/PngTest.cs`.
- Recommendations: treat binary loaders as a fuzzing target, add corpus-based tests for CAB/VQA/audio/archive code, and keep parser failures isolated from long-lived gameplay state where possible.

**Native/platform interop is a trust boundary with little automated safety net:**
- Risk: font, input, windowing, graphics, and audio paths use `DllImport`, raw pointers, and manual memory layout assumptions; malformed runtime state or platform drift can cause hard crashes instead of recoverable errors.
- Files: `OpenRA.Platforms.Default/FreeTypeFont.cs`, `OpenRA.Platforms.Default/Sdl2PlatformWindow.cs`, `OpenRA.Platforms.Default/Sdl2GraphicsContext.cs`, `OpenRA.Platforms.Default/OpenAlSoundEngine.cs`, `OpenRA.Platforms.Default/OpenGL.cs`, `OpenRA.Platforms.Default/Texture.cs`, `Directory.Build.props`
- Current mitigation: the codebase stays in managed .NET for most engine logic, allows unsafe blocks globally in `Directory.Build.props`, and uses one platform implementation assembly loaded through the engine bootstrap.
- Recommendations: add platform smoke tests, minimize pointer arithmetic during future refactors, and assume any change in this layer requires real cross-platform runtime validation.

**External mod and assembly loading expands the trust boundary beyond Lua scripting:**
- Risk: platform DLLs and mod assemblies are loaded dynamically, while external mod registrations persist `LaunchPath` metadata that can later be executed or presented by the mod switcher.
- Files: `OpenRA.Game/ObjectCreator.cs`, `OpenRA.Game/Support/AssemblyLoader.cs`, `OpenRA.Game/Game.cs`, `OpenRA.Game/ExternalMods.cs`, `OpenRA.Game/Scripting/ScriptContext.cs`
- Current mitigation: Lua scripts are sandboxed in `OpenRA.Game/Scripting/ScriptContext.cs`, and external mod metadata is sanity-checked and cleaned in `OpenRA.Game/ExternalMods.cs`.
- Recommendations: treat external mod registration as trusted-local input only, prefer stricter validation of launch paths and metadata sources, and avoid broadening dynamic assembly discovery without a stronger trust model.

## Performance Bottlenecks

**Lobby/network sync still scales by rebroadcasting full state and buffering with static latency:**
- Problem: every lobby client or slot change triggers full list serialization, ping updates are special-cased, and game start injects empty frames per client to satisfy static order buffering.
- Files: `OpenRA.Game/Server/Server.cs`
- Measurement: not instrumented in-repo, but the implementation currently assumes a default order lag of 3 net ticks / 360ms and includes TODOs to replace full syncs and static buffering.
- Cause: the protocol has no general delta-sync path for lobby state and no dynamic order buffering system.
- Improvement path: add delta updates for clients/slots/global settings, then replace static `OrderLatency` handling with adaptive buffering before attempting larger multiplayer or save/load changes.

**Map generation and pathfinding are algorithmically dense hotspots with limited guardrails:**
- Problem: `Terraformer`, `MatrixUtils`, `TilingPath`, and `HierarchicalPathFinder` are some of the largest engine files, and movement code still contains workarounds for non-shortest paths and repeated candidate-cell scans.
- Files: `OpenRA.Mods.Common/MapGenerator/Terraformer.cs`, `OpenRA.Mods.Common/MapGenerator/MatrixUtils.cs`, `OpenRA.Mods.Common/MapGenerator/TilingPath.cs`, `OpenRA.Mods.Common/Traits/World/ClassicMapGenerator.cs`, `OpenRA.Mods.Common/Pathfinder/HierarchicalPathFinder.cs`, `OpenRA.Mods.Common/Activities/Move/MoveWithinRange.cs`, `OpenRA.Mods.Common/Activities/Move/Move.cs`, `OpenRA.Mods.Common/Traits/AutoCrusher.cs`
- Measurement: `OpenRA.Mods.Common/MapGenerator/Terraformer.cs` is 2305 lines, `OpenRA.Mods.Common/MapGenerator/MatrixUtils.cs` is 1771 lines, `OpenRA.Mods.Common/Pathfinder/HierarchicalPathFinder.cs` is 1284 lines, and `MoveWithinRange` already caches per-tick annulus queries as an explicit perf workaround.
- Cause: generation and navigation logic operate on large matrix/grid structures with partial heuristics for dynamic obstacles.
- Improvement path: add benchmark fixtures for representative maps, isolate the slowest transforms, and avoid behavior changes in pathfinding without profiling both correctness and runtime cost.

## Fragile Areas

**Server, lobby, save, and identity flow are concentrated in a few monolithic classes:**
- Files: `OpenRA.Game/Server/Server.cs`, `OpenRA.Game/Network/GameSave.cs`, `OpenRA.Game/Player.cs`, `OpenRA.Game/Game.cs`
- Why fragile: connection validation, command interpretation, save/load, admin reassignment, lobby sync, and game start all share mutable server state and lock ordering.
- Common failures: wrong player indices, broken lobby updates, save/load regressions, and multiplayer desyncs when protocol assumptions drift.
- Safe modification: add or extend high-level server tests before changing protocol or slot logic; keep changes localized to one transition at a time instead of combining lobby, save, and order-buffer edits.
- Test coverage: `OpenRA.Test/` contains 20 C# test files and none target `Server`, `Network`, `GameSave`, or lobby synchronization paths.

**Platform and renderer code can fail catastrophically from small lifetime or ABI mistakes:**
- Files: `OpenRA.Platforms.Default/FreeTypeFont.cs`, `OpenRA.Platforms.Default/OpenAlSoundEngine.cs`, `OpenRA.Platforms.Default/Sdl2PlatformWindow.cs`, `OpenRA.Platforms.Default/Sdl2GraphicsContext.cs`, `OpenRA.Game/Graphics/WorldRenderer.cs`
- Why fragile: the code mixes native handles, manual offsets, async audio state, and engine-owned lifetime assumptions.
- Common failures: startup crashes, use-after-dispose bugs, missing device features, cross-platform regressions, and hard-to-reproduce race conditions.
- Safe modification: validate on every target OS/runtime combination touched by the change and preserve disposal order until ownership is made explicit.
- Test coverage: no tests in `OpenRA.Test/` reference `OpenRA.Platforms.Default`, SDL/OpenGL, or renderer classes.

**Aircraft and resupply behavior still depends on timing-sensitive activity chaining:**
- Files: `OpenRA.Mods.Common/Activities/Resupply.cs`, `OpenRA.Mods.Common/Activities/Air/Fly.cs`, `OpenRA.Mods.Common/Activities/Air/Land.cs`, `OpenRA.Mods.Common/Traits/Repairable.cs`
- Why fragile: activity cancellation, terrain altitude, reservation, repair, and docking state are encoded across separate activities rather than a single coordinator.
- Common failures: aircraft getting stuck, units blocking resupply pads, or landing/repair flows behaving differently after unrelated movement changes.
- Safe modification: test cancel, pause, repair, and idle-land cases together; do not change one activity in isolation unless you understand the full chain.
- Test coverage: `OpenRA.Test/` has no movement, aircraft, or docking behavior tests.

**Legacy importer and updater toolchains are easy to break silently:**
- Files: `OpenRA.Mods.Cnc/UtilityCommands/ImportGen1MapCommand.cs`, `OpenRA.Mods.Cnc/UtilityCommands/ImportGen2MapCommand.cs`, `OpenRA.Mods.Common/UtilityCommands/UpdateModCommand.cs`, `OpenRA.Mods.Common/UtilityCommands/UpdateMapCommand.cs`, `OpenRA.Mods.Common/UpdateRules/UpdateUtils.cs`
- Why fragile: these commands depend on the same global state hacks as the engine, and some inputs are old file formats or transformation rules with low day-to-day usage.
- Common failures: broken migrations, incomplete update output, or legacy assets importing with incorrect metadata.
- Safe modification: capture representative fixture inputs before refactoring; treat importer behavior as compatibility code rather than dead code.
- Test coverage: no dedicated importer/update command tests exist in `OpenRA.Test/`.

## Scaling Limits

**Multiplayer order buffering and lobby sync are tuned for small-session assumptions:**
- Current capacity: server latency quality thresholds are explicitly keyed to the default 3 net tick / 360ms order lag in `OpenRA.Game/Server/Server.cs`.
- Files: `OpenRA.Game/Server/Server.cs`
- Limit: larger or noisier lobbies pay for full client and slot rebroadcasts, while fixed order latency becomes a blunt instrument for variable network conditions.
- Symptoms at limit: more lobby churn traffic, harder-to-debug sync conflicts, and growing pressure to keep special-case ping and admin logic aligned.
- Scaling path: ship a general lobby delta protocol and dynamic order buffering before optimizing around higher player counts.

**Map generation cost grows with full-map matrix transforms rather than incremental work:**
- Current capacity: the generator builds and transforms entire matrix layers in memory through `Terraformer`, `MatrixUtils`, and `ClassicMapGenerator`.
- Files: `OpenRA.Mods.Common/MapGenerator/Terraformer.cs`, `OpenRA.Mods.Common/MapGenerator/MatrixUtils.cs`, `OpenRA.Mods.Common/Traits/World/ClassicMapGenerator.cs`
- Limit: larger maps or more generator options increase CPU and memory cost non-linearly, and some logic still carries acknowledged mathematical shortcuts.
- Symptoms at limit: long editor stalls, expensive retries, and harder debugging when generation fails or produces poor terrain.
- Scaling path: add profiling hooks and deterministic benchmark scenarios before attempting new generator features or larger generated maps.

## Dependencies at Risk

**Native wrapper packages are critical but operationally sensitive:**
- Risk: `OpenRA-Freetype6`, `OpenRA-OpenAL-CS`, and `OpenRA-SDL2-CS` sit directly on the platform boundary, and the surrounding code already contains manual offsets, `DllImport`, and race-condition comments.
- Files: `OpenRA.Platforms.Default/OpenRA.Platforms.Default.csproj`, `OpenRA.Platforms.Default/FreeTypeFont.cs`, `OpenRA.Platforms.Default/OpenAlSoundEngine.cs`, `OpenRA.Platforms.Default/Sdl2PlatformWindow.cs`
- Impact: if runtime behavior changes, the engine can lose text rendering, audio, input, or startup entirely on one platform before CI notices.
- Migration plan: keep platform bindings behind narrow abstractions and add smoke coverage before swapping package versions or changing the platform bootstrap path.

**LAN/NAT/media dependencies are lightly covered compared with their user impact:**
- Risk: `Mono.NAT`, `rix0rrr.BeaconLib`, `MP3Sharp`, `NVorbis`, `TagLibSharp`, and `Pfim` affect hosting, discovery, audio decoding, and image loading, but there is little automated coverage around those integration points.
- Files: `OpenRA.Game/OpenRA.Game.csproj`, `OpenRA.Mods.Common/OpenRA.Mods.Common.csproj`, `OpenRA.Game/Network/Nat.cs`, `OpenRA.Mods.Common/ServerTraits/MasterServerPinger.cs`, `OpenRA.Mods.Common/Widgets/Logic/ServerListLogic.cs`, `OpenRA.Mods.Common/AudioLoaders/Mp3Loader.cs`, `OpenRA.Mods.Common/AudioLoaders/OggLoader.cs`, `OpenRA.Mods.Common/SpriteLoaders/DdsLoader.cs`, `OpenRA.Mods.Common/SpriteLoaders/TgaLoader.cs`
- Impact: regressions show up as “can’t host”, “can’t discover”, or “can’t play/load media” issues that are often platform- or data-specific.
- Migration plan: wrap these integrations in higher-level engine services and add fixture-driven tests before dependency upgrades.

## Missing Critical Features

**Proper multiplayer save support:**
- Problem: the current save system is explicitly scoped away from multiplayer; slot reassignment, spectator orders, and viewport persistence all rely on singleplayer or skirmish assumptions.
- Files: `OpenRA.Game/Server/Server.cs`, `OpenRA.Game/Network/GameSave.cs`, `OpenRA.Mods.Common/Traits/Player/GameSaveViewportManager.cs`, `OpenRA.Game/World.cs`
- Current workaround: enable saves only for non-dedicated sessions with one non-bot client.
- Blocks: persistent multiplayer matches, reconnect-friendly campaign/co-op flows, and safer future save-system refactors.
- Implementation complexity: High

**General delta-based lobby sync and adaptive order buffering:**
- Problem: the network layer still depends on full lobby rebroadcasts and fixed order latency.
- Files: `OpenRA.Game/Server/Server.cs`
- Current workaround: special-case sync methods for clients, slots, global settings, and connection quality, plus fixed `OrderLatency`.
- Blocks: better scaling, lower-latency multiplayer, and clean handling of mid-game admin or save/load state transitions.
- Implementation complexity: High

## Test Coverage Gaps

**Server/network/save flow is effectively untested by the automated test project:**
- What's not tested: connection validation, lobby synchronization, admin reassignment, order buffering, desync/replay edges, and game save load paths.
- Files: `OpenRA.Game/Server/Server.cs`, `OpenRA.Game/Network/GameSave.cs`, `OpenRA.Game/Game.cs`, `OpenRA.Game/Player.cs`, `OpenRA.Test/OpenRA.Test.csproj`, `.github/workflows/ci.yml`, `Makefile`
- Risk: regressions in multiplayer or save/load behavior can land even while `make check`, `make test`, and `make tests` pass.
- Priority: High
- Difficulty to test: high because current tests are unit-focused and the repo has no integration harness for full server/client state transitions.

**Map generation, pathfinding, and movement behavior have almost no automated correctness coverage:**
- What's not tested: `Terraformer`, `ClassicMapGenerator`, hierarchical pathfinding, shortest-path edge cases, and aircraft/docking/resupply flows.
- Files: `OpenRA.Mods.Common/MapGenerator/Terraformer.cs`, `OpenRA.Mods.Common/Traits/World/ClassicMapGenerator.cs`, `OpenRA.Mods.Common/Pathfinder/HierarchicalPathFinder.cs`, `OpenRA.Mods.Common/Activities/Move/MoveWithinRange.cs`, `OpenRA.Mods.Common/Activities/Air/Fly.cs`, `OpenRA.Mods.Common/Activities/Air/Land.cs`, `OpenRA.Mods.Common/Activities/Resupply.cs`, `OpenRA.Mods.Common/UtilityCommands/FuzzMapGeneratorCommand.cs`
- Risk: gameplay regressions or performance regressions are likely to surface only through manual playtesting.
- Priority: High
- Difficulty to test: high because these systems need deterministic fixtures and scenario-style assertions rather than simple parser tests.

**Platform, graphics, installer, and importer paths are outside the current test envelope:**
- What's not tested: SDL/OpenGL/OpenAL/FreeType integration, installer source resolvers, legacy asset importers, and most binary format loaders beyond PNG.
- Files: `OpenRA.Platforms.Default/FreeTypeFont.cs`, `OpenRA.Platforms.Default/OpenAlSoundEngine.cs`, `OpenRA.Platforms.Default/Sdl2PlatformWindow.cs`, `OpenRA.Mods.Common/Installer/SourceResolvers/SteamSourceResolver.cs`, `OpenRA.Mods.Common/Installer/SourceResolvers/GogSourceResolver.cs`, `OpenRA.Mods.Common/Installer/SourceResolvers/RegistryDirectorySourceResolver.cs`, `OpenRA.Mods.Common/Installer/SourceResolvers/DiscSourceResolver.cs`, `OpenRA.Mods.Cnc/UtilityCommands/ImportGen1MapCommand.cs`, `OpenRA.Mods.Cnc/FileFormats/VqaVideo.cs`, `OpenRA.Mods.Common/FileFormats/InstallShieldCABCompression.cs`
- Risk: platform-specific or format-specific regressions can ship unnoticed because the current `OpenRA.Test/` suite covers 20 C# test files and 4,222 lines against 1,486 production C# files and 227,875 lines.
- Priority: Medium
- Difficulty to test: medium to high because many of these flows require fixture archives, legacy assets, or cross-platform execution.

---

*Concerns audit: 2026-03-26*
*Update as issues are fixed or new ones discovered*
