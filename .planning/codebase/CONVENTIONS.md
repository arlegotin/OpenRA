# Coding Conventions

**Analysis Date:** 2026-03-26

## Naming Patterns

**Files:**
- C# source files use `PascalCase.cs`, usually named for the main feature or type: `OpenRA.Game/Actor.cs`, `OpenRA.Game/Support/ExceptionHandler.cs`, `OpenRA.Mods.Cnc/Traits/PortableChrono.cs`.
- A single C# file can contain multiple closely related types, so the filename matches the feature, not necessarily every contained type: `OpenRA.Mods.Cnc/Traits/PortableChrono.cs`, `OpenRA.Game/IUtilityCommand.cs`.
- Test files live in the separate `OpenRA.Test` project and usually use `<Subject>Test.cs`, with a few pluralized `*Tests.cs` files: `OpenRA.Test/OpenRA.Game/CPosTest.cs`, `OpenRA.Test/OpenRA.Game/PngTest.cs`, `OpenRA.Test/OpenRA.Game/Sha1Tests.cs`.
- YAML files are lowercase and functional, often using canonical names like `map.yaml`, `rules.yaml`, `mod.yaml`, or feature names: `mods/all/mod.yaml`, `mods/cnc/chrome/mainmenu.yaml`, `mods/cnc/maps/gdi01/map.yaml`.
- Lua files are lowercase mission or script identifiers, with `-AI.lua` for companion AI scripts: `mods/cnc/maps/gdi01/gdi01.lua`, `mods/ra/maps/allies-04/allies04-AI.lua`, `mods/common/scripts/utils.lua`.

**Functions:**
- C# methods, properties, and local functions use `PascalCase`: `OpenRA.Game/Support/ExceptionHandler.cs`, `OpenRA.Game/Support/Log.cs`, `OpenRA.Mods.Cnc/Traits/PortableChrono.cs`.
- Explicit interface implementations are common when the interface is marked with `RequireExplicitImplementationAttribute`: `OpenRA.Mods.Common/UtilityCommands/CheckYaml.cs`, `OpenRA.Mods.Common/UtilityCommands/CheckExplicitInterfacesCommand.cs`, `OpenRA.Mods.Cnc/Traits/PortableChrono.cs`.
- Lua mission hooks exposed to the engine also use `PascalCase` globals: `WorldLoaded`, `Tick`, `Reinforce`, `SendNodPatrol` in `mods/cnc/maps/gdi01/gdi01.lua`.

**Variables:**
- Parameters, locals, and private/protected instance fields use `camelCase` with no leading underscore: `orientation` in `OpenRA.Mods.Common/Traits/HitShape.cs`, `warningAsError` in `OpenRA.Mods.Common/UtilityCommands/CheckYaml.cs`, `chargeTick` in `OpenRA.Mods.Cnc/Traits/PortableChrono.cs`.
- Public fields, public properties, constants, and static readonly fields use `PascalCase`, not `ALL_CAPS`: `CreateLogFileMaxRetryCount` and `FlushInterval` in `OpenRA.Game/Support/Log.cs`, `InvalidConditionToken` in `OpenRA.Game/Actor.cs`.
- YAML-bound trait fields keep `PascalCase` names even when initialized to `null` defaults: `OpenRA.Game/Traits/World/Faction.cs`, `OpenRA.Game/Manifest.cs`, `OpenRA.Mods.Cnc/Traits/PortableChrono.cs`.
- When compatibility requires an exception, use a narrow pragma instead of changing surrounding style: `OpenRA.Game/Manifest.cs`, `OpenRA.Test/OpenRA.Game/FieldLoaderTest.cs`.

**Types:**
- Classes, structs, records, enums, and attributes use `PascalCase`: `ChannelData` in `OpenRA.Game/Support/Log.cs`, `DamageState` in `OpenRA.Game/Traits/TraitsInterfaces.cs`, `TraitLocationAttribute` in `OpenRA.Game/Traits/LintAttributes.cs`.
- Interfaces use `I` + `PascalCase`: `OpenRA.Game/Traits/TraitsInterfaces.cs`, `OpenRA.Game/IUtilityCommand.cs`.
- Trait definitions usually pair `FooInfo` and `Foo` in the same file: `OpenRA.Game/Traits/World/Faction.cs`, `OpenRA.Mods.Common/Traits/HitShape.cs`, `OpenRA.Mods.Cnc/Traits/PortableChrono.cs`.
- Enum members use `PascalCase`: `RunStatus` in `OpenRA.Game/Support/Program.cs`, `PlayerRelationship` in `OpenRA.Game/Traits/TraitsInterfaces.cs`.

## Code Style

**Formatting:**
- Formatting is analyzer-driven through `.editorconfig`, `StyleCop.Analyzers`, and `Roslynator.Formatting.Analyzers`, not Prettier/ESLint/Biome: `.editorconfig`, `Directory.Build.props`.
- Use LF newlines, trim trailing whitespace, and keep a final newline: `.editorconfig`.
- Use 4-column tab indentation for `*.cs`, `*.csproj`, `*.yaml`, `*.lua`, `*.sh`, and `*.ps1`: `.editorconfig`. This is visible in `OpenRA.Game/Actor.cs`, `mods/cnc/chrome/mainmenu.yaml`, and `mods/cnc/maps/gdi01/gdi01.lua`.
- Use block-scoped namespaces. File-scoped namespaces are configured as warnings: `.editorconfig`, `OpenRA.Game/Actor.cs`, `OpenRA.Utility/Program.cs`.
- Prefer `var`, collection expressions, object/collection initializers, and modern pattern matching where it improves clarity: `.editorconfig`, `OpenRA.Game/Support/Log.cs`, `OpenRA.Game/Exts.cs`, `OpenRA.Mods.Common/Traits/HitShape.cs`.
- Nullable reference types are disabled repo-wide. `null` is still used as a normal sentinel/default value in C# APIs and YAML-bound fields: `Directory.Build.props`, `OpenRA.Game/Traits/World/Faction.cs`, `OpenRA.Game/Manifest.cs`.
- Semicolons are required. Standard C# double-quoted strings, interpolated strings, and verbatim strings are all common: `OpenRA.Game/Support/ExceptionHandler.cs`, `OpenRA.Test/OpenRA.Game/MiniYamlTest.cs`.
- Single-line control flow without braces is accepted and common. Keep braces when the block is multiline or the logic is non-trivial: `OpenRA.Game/Support/Log.cs`, `OpenRA.Game/Exts.cs`, `OpenRA.Test/OpenRA.Game/PlatformTest.cs`.
- No explicit maximum line length is configured in `.editorconfig`.

**Linting:**
- The authoritative in-repo rules are `.editorconfig`, `Directory.Build.props`, `Makefile`, and `make.ps1`. `CONTRIBUTING.md` points to an external coding-standard wiki, but the local analyzer configuration is what CI enforces: `CONTRIBUTING.md`, `.editorconfig`, `Directory.Build.props`.
- Run `make check` on Unix-like hosts or `./make.ps1 check` on Windows before treating C# changes as ready: `Makefile`, `make.ps1`.
- `make check` performs a Debug build with `-warnaserror`, then runs custom utility validations for explicit interface implementation and conditional trait overrides: `Makefile`, `OpenRA.Mods.Common/UtilityCommands/CheckExplicitInterfacesCommand.cs`, `OpenRA.Mods.Common/UtilityCommands/CheckConditionalTraitInterfaceOverrides.cs`.
- CI runs `make check`, `make tests`, `make check-scripts`, and `make TREAT_WARNINGS_AS_ERRORS=true test` on Linux, with PowerShell equivalents on Windows: `.github/workflows/ci.yml`, `Makefile`, `make.ps1`.
- Code style enforcement belongs in Debug/check flows. Release builds remove analyzers for compile-time performance: `Directory.Build.props`.

## Import Organization

**Order:**
1. BCL/System namespaces first: `System`, `System.Collections.*`, `System.Linq` in `OpenRA.Game/Actor.cs` and `OpenRA.Mods.Cnc/Traits/PortableChrono.cs`
2. Third-party namespaces next when present: `Eluant` in `OpenRA.Game/Actor.cs`, `NUnit.Framework` in `OpenRA.Test/OpenRA.Game/CPosTest.cs`
3. OpenRA namespaces last: `OpenRA.Graphics`, `OpenRA.Traits`, `OpenRA.Mods.Common.*` in `OpenRA.Mods.Common/Traits/HitShape.cs`

**Grouping:**
- Keep `using` directives at the top of the file, outside the namespace: `OpenRA.Game/Support/Log.cs`, `OpenRA.Utility/Program.cs`.
- Sort within each logical block roughly alphabetically.
- Blank lines between groups are optional. Many files keep one continuous `using` block even when mixing framework, third-party, and internal namespaces: `OpenRA.Game/Actor.cs`, `OpenRA.Mods.Cnc/Traits/PortableChrono.cs`, `OpenRA.Test/OpenRA.Game/FieldLoaderTest.cs`.

**Path Aliases:**
- Not used. Imports rely on real namespaces and project references: `OpenRA.sln`, `OpenRA.Test/OpenRA.Test.csproj`.

## Error Handling

**Patterns:**
- Throw specific exception types at the point of failure with contextual messages: `OpenRA.Game/FieldLoader.cs`, `OpenRA.Game/Actor.cs`, `OpenRA.Mods.Common/Traits/HitShape.cs`.
- Catch broad exceptions only at program boundaries, utility boundaries, or linter/plugin boundaries where the code can log, print, exit, or rethrow cleanly: `OpenRA.Server/Program.cs`, `OpenRA.Launcher/Program.cs`, `OpenRA.Utility/Program.cs`, `OpenRA.Mods.Common/UtilityCommands/CheckYaml.cs`.
- Command-line tools commonly fail with `Environment.Exit(1)` after printing a summary: `OpenRA.Utility/Program.cs`, `OpenRA.Mods.Common/UtilityCommands/CheckYaml.cs`, `OpenRA.Mods.Common/UtilityCommands/CheckExplicitInterfacesCommand.cs`, `OpenRA.Mods.Common/UtilityCommands/CheckConditionalTraitInterfaceOverrides.cs`.
- Plugin-style loops catch exceptions per pass to downgrade failures into reported lint errors instead of crashing the whole checker: `OpenRA.Mods.Common/UtilityCommands/CheckYaml.cs`.

**Error Types:**
- Use `YamlException` for YAML/config parsing and trait-loading failures: `OpenRA.Game/FieldLoader.cs`, `OpenRA.Mods.Common/Traits/HitShape.cs`, `OpenRA.Mods.Cnc/Traits/SupportPowers/IonCannonPower.cs`.
- Use `ArgumentException` and `ArgumentOutOfRangeException` for invalid API inputs: `OpenRA.Game/Map/CellCoordsRegion.cs`, `OpenRA.Game/StreamExts.cs`, `OpenRA.Mods.Cnc/FileSystem/MixFile.cs`.
- Use `InvalidDataException` for malformed external formats and serialized content: `OpenRA.Mods.Cnc/AudioLoaders/VocLoader.cs`, `OpenRA.Mods.Cnc/FileFormats/VqaVideo.cs`, `OpenRA.Game/Widgets/WidgetLoader.cs`.
- Use `InvalidOperationException` for invariant violations and illegal engine state: `OpenRA.Game/Activities/Activity.cs`, `OpenRA.Game/Actor.cs`, `OpenRA.Server/Program.cs`.
- Returning `null` is acceptable for expected "not available" cases because nullable annotations are disabled: `OpenRA.Mods.Cnc/Traits/PortableChrono.cs`, `OpenRA.Game/Manifest.cs`.

## Logging

**Framework:**
- Use the custom channel-based `Log` system for engine/runtime logging: `OpenRA.Game/Support/Log.cs`.
- Use `Console.WriteLine` and `Console.Error.WriteLine` for launcher, utility, linter, and migration command output: `OpenRA.Server/Program.cs`, `OpenRA.Utility/Program.cs`, `OpenRA.Mods.Common/UtilityCommands/CheckYaml.cs`, `OpenRA.Mods.Cnc/UtilityCommands/ImportRedAlertMapCommand.cs`.

**Patterns:**
- Register channels during startup with descriptive names like `debug`, `perf`, `server`, `utility`, `exception`, `nat`, and `geoip`: `OpenRA.Server/Program.cs`, `OpenRA.Utility/Program.cs`, `OpenRA.Game/Support/ExceptionHandler.cs`.
- Log exceptions through `Log.Write(channel, exception)` when the caller wants the stack trace preserved: `OpenRA.Utility/Program.cs`, `OpenRA.Mods.Common/ItchIntegration.cs`, `OpenRA.Game/Map/MapPreview.cs`.
- Prefer channel-based text logging over structured payloads. Channel names communicate purpose more than formal severity levels: `OpenRA.Game/Support/Log.cs`.
- CLI validators temporarily change `Console.ForegroundColor` to highlight warnings and errors: `OpenRA.Mods.Common/UtilityCommands/CheckYaml.cs`, `OpenRA.Mods.Common/UtilityCommands/CheckExplicitInterfacesCommand.cs`, `OpenRA.Mods.Common/UtilityCommands/CheckConditionalTraitInterfaceOverrides.cs`.

## Comments

**When to Comment:**
- Start new C# files with the standard GPL header region used throughout the repo: `OpenRA.Game/Actor.cs`, `OpenRA.Utility/Program.cs`, `OpenRA.Mods.Cnc/Traits/PortableChrono.cs`.
- Use short inline comments for engine constraints, compatibility quirks, and performance-sensitive choices. Prefixes `TODO`, `HACK`, `PERF`, and `SAFETY` are normal: `OpenRA.Game/Actor.cs`, `OpenRA.Game/Exts.cs`, `OpenRA.Server/Program.cs`, `OpenRA.Mods.Common/Activities/Resupply.cs`.
- Use `[Desc(...)]` attributes to document YAML-exposed fields and command-line usage instead of adding separate doc wrappers: `OpenRA.Game/Traits/World/Faction.cs`, `OpenRA.Mods.Cnc/Traits/PortableChrono.cs`, `OpenRA.Mods.Common/UtilityCommands/CheckYaml.cs`.
- Lua files carry the same license intent in block comments and then use short mission-script comments sparingly: `mods/cnc/maps/gdi01/gdi01.lua`.

**JSDoc/TSDoc:**
- XML doc comments are selective, not mandatory. Add them for public engine APIs, complex editor/map-generator code, or places that benefit from generated docs: `OpenRA.Game/Activities/Activity.cs`, `OpenRA.Game/Exts.cs`, `OpenRA.Mods.Common/MapGenerator/MultiBrush.cs`.
- Do not add XML docs only to satisfy tooling. Documentation analyzer rules such as SA1600 are disabled in `.editorconfig`.

**TODO Comments:**
- Use plain `TODO`, `HACK`, or `PERF` comments without usernames or issue IDs: `OpenRA.Game/Support/ExceptionHandler.cs`, `OpenRA.Mods.Cnc/UtilityCommands/ImportGen1MapCommand.cs`, `OpenRA.Mods.Common/Widgets/WidgetUtils.cs`.
- Keep warning suppressions narrowly scoped and explain them inline when possible: `OpenRA.Game/Manifest.cs`, `OpenRA.Game/GameRules/ActorInfo.cs`, `OpenRA.Test/OpenRA.Game/FieldLoaderTest.cs`.

## Function Design

**Size:**
- Prefer small, single-purpose methods, but keep hot-path engine code together when splitting would obscure invariants or hurt performance: `OpenRA.Game/Actor.cs`, `OpenRA.Game/Exts.cs`, `OpenRA.Mods.Common/UtilityCommands/CheckYaml.cs`.
- Use nested helper methods or local functions when a small helper only matters to one code path or one test: `OpenRA.Game/Support/ExceptionHandler.cs`, `OpenRA.Test/OpenRA.Game/PngTest.cs`, `OpenRA.Test/OpenRA.Game/FieldLoaderTest.cs`.

**Parameters:**
- Keep parameter names descriptive and `camelCase`.
- Put `CancellationToken` last when present. CA1068 is enabled in `.editorconfig`.
- Use `in` parameters for larger value types when avoiding copies matters: `OpenRA.Mods.Common/Traits/HitShape.cs`, `OpenRA.Mods.Cnc/Traits/PortableChrono.cs`.

**Return Values:**
- Prefer guard clauses and early returns: `OpenRA.Game/Support/Log.cs`, `OpenRA.Mods.Common/Traits/HitShape.cs`, `OpenRA.Utility/Program.cs`.
- Use `yield return` and `yield break` for lazy enumerations of orders, renderables, positions, and other streamed results: `OpenRA.Mods.Cnc/Traits/PortableChrono.cs`, `OpenRA.Mods.Common/Traits/HitShape.cs`.
- Returning `null` is part of normal control flow in some APIs because nullable reference types are disabled: `OpenRA.Mods.Cnc/Traits/PortableChrono.cs`, `OpenRA.Game/Manifest.cs`.

## Module Design

**Exports:**
- There are no barrel files. Types are consumed through namespaces and direct project references: `OpenRA.sln`, `OpenRA.Test/OpenRA.Test.csproj`.
- It is normal for a file to contain multiple related runtime types, especially an `Info` class plus its runtime trait and helper targeters/generators: `OpenRA.Mods.Cnc/Traits/PortableChrono.cs`, `OpenRA.Game/IUtilityCommand.cs`, `OpenRA.Test/OpenRA.Game/ActorInfoTest.cs`.
- When implementing interfaces tagged with `RequireExplicitImplementationAttribute`, prefer explicit interface members so command/trait APIs do not leak onto the public class surface: `OpenRA.Game/IUtilityCommand.cs`, `OpenRA.Mods.Common/UtilityCommands/CheckYaml.cs`, `OpenRA.Mods.Cnc/Traits/PortableChrono.cs`.

**Barrel Files:**
- Not used in the current codebase.
- Keep helper types in the same file when they are tightly coupled to one feature. Split only when the helper has its own meaningful namespace surface or lifecycle.

---

*Convention analysis: 2026-03-26*
