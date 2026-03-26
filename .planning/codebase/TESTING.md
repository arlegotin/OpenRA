# Testing Patterns

**Analysis Date:** 2026-03-26

## Test Framework

**Runner:**
- NUnit 4.3.2, hosted by `Microsoft.NET.Test.Sdk` 17.12.0 and `NUnit3TestAdapter` 4.6.0 in `OpenRA.Test/OpenRA.Test.csproj`.
- Config is split across the dedicated test project `OpenRA.Test/OpenRA.Test.csproj`, runtime probing in `OpenRA.Test/App.config`, and execution wrappers in `Makefile`, `make.ps1`, and `.github/workflows/ci.yml`.
- The repository also relies on non-NUnit verification commands for mod data and scripts: `make test`, `make check`, and `make check-scripts` in `Makefile` and `make.ps1`.

**Assertion Library:**
- NUnit's built-in constraint model is the default: `Assert.That(..., Is.EqualTo(...))`, `Does.Contain`, `Has.Count`, `Throws.TypeOf`, and `Assert.Throws<T>()`.
- Direct `Assert.Fail(...)` is used for explicit timeout/failure branches when a richer constraint would be awkward: `OpenRA.Test/OpenRA.Game/ActionQueueTest.cs`, `OpenRA.Test/OpenRA.Game/MiniYamlTest.cs`.

**Run Commands:**
```bash
make tests                         # Build OpenRA.Test in Debug and run NUnit tests on Unix-like hosts
./make.ps1 tests                   # Build OpenRA.Test in Debug and run NUnit tests on Windows
dotnet build OpenRA.Test/OpenRA.Test.csproj -c Debug --nologo -p:TargetPlatform=<rid>  # Direct test build
dotnet test bin/OpenRA.Test.dll --test-adapter-path:.                                    # Direct NUnit execution after build
make check                         # Debug analyzer build + custom interface/trait validation
make check-scripts                 # Lua syntax checking via luac
make test                          # Official mod/map YAML linting via OpenRA.Utility --check-yaml
```
- Watch mode: Not detected in repo scripts or config.
- Coverage command: Not detected in repo scripts or config.

## Test File Organization

**Location:**
- Tests live in the separate `OpenRA.Test` project, not alongside production files: `OpenRA.Test/OpenRA.Game/*.cs`, `OpenRA.Test/OpenRA.Mods.Common/*.cs`.
- The test tree currently targets engine core and `OpenRA.Mods.Common`. There are no dedicated test folders for `OpenRA.Server`, `OpenRA.Utility`, `OpenRA.Launcher`, `OpenRA.Platforms.Default`, `OpenRA.Mods.Cnc`, or `OpenRA.Mods.D2k`.
- Cross-cutting verification for YAML, Lua, and analyzer rules happens outside `OpenRA.Test` through `Makefile`, `make.ps1`, and utility commands in `OpenRA.Mods.Common/UtilityCommands/*.cs`.

**Naming:**
- Most files use `<Subject>Test.cs`: `OpenRA.Test/OpenRA.Game/CPosTest.cs`, `OpenRA.Test/OpenRA.Game/FieldLoaderTest.cs`, `OpenRA.Test/OpenRA.Mods.Common/ShapeTest.cs`.
- A few use pluralized `*Tests.cs` when the subject is utility-like or historical: `OpenRA.Test/OpenRA.Game/Sha1Tests.cs`, `OpenRA.Test/OpenRA.Game/StreamExtsTests.cs`, while `OpenRA.Test/OpenRA.Game/PngTest.cs` contains class `PngTests`.
- All sampled test files use namespace `OpenRA.Test` regardless of the folder below `OpenRA.Test/`.

**Structure:**
```text
OpenRA.Test/
  App.config
  OpenRA.Game/
    ActionQueueTest.cs
    ActorInfoTest.cs
    FieldLoaderTest.cs
    MiniYamlTest.cs
    ...
  OpenRA.Mods.Common/
    PerfGraphWidgetTest.cs
    ShapeTest.cs
```

## Test Structure

**Suite Organization:**
```csharp
[TestFixture]
sealed class ActionQueueTest
{
    [TestCase(TestName = "ActionQueue performs actions in order of time, then insertion order.")]
    public void ActionsArePerformedOrderedByTimeThenByInsertionOrder()
    {
        var list = new List<int>();
        var queue = new ActionQueue();

        queue.Add(() => list.Add(1), 0);
        queue.PerformActions(1);

        Assert.That(list, Is.EqualTo(new[] { 1 }));
    }
}
```
```csharp
static IEnumerable<TestCaseData> GetValue_InvalidValue_TestCases()
{
    return
    [
        new TestCaseData(null) { TypeArgs = [typeof(int)] },
        new TestCaseData("test") { TypeArgs = [typeof(int)] },
    ];
}

[TestCaseSource(nameof(GetValue_InvalidValue_TestCases))]
public void GetValue_InvalidValue<T>(string input)
{
    void Act() => FieldLoader.GetValue<T>("field", input);
    Assert.That(Act, Throws.TypeOf<YamlException>());
}
```

**Patterns:**
- Use one `[TestFixture]` class per subject: `OpenRA.Test/OpenRA.Game/ActionQueueTest.cs`, `OpenRA.Test/OpenRA.Game/FieldLoaderTest.cs`, `OpenRA.Test/OpenRA.Mods.Common/PerfGraphWidgetTest.cs`.
- Prefer descriptive `TestName` values on `[TestCase]` when the method name is generic or reused across many inputs: `OpenRA.Test/OpenRA.Game/ActionQueueTest.cs`, `OpenRA.Test/OpenRA.Mods.Common/PerfGraphWidgetTest.cs`, `OpenRA.Test/OpenRA.Mods.Common/ShapeTest.cs`.
- Parameterize heavily with `[TestCase]` and `[TestCaseSource]` for parsers, serializers, and math-heavy code: `OpenRA.Test/OpenRA.Game/FieldLoaderTest.cs`, `OpenRA.Test/OpenRA.Game/FieldSaverTest.cs`, `OpenRA.Test/OpenRA.Game/PriorityQueueTest.cs`.
- Setup hooks are rare. Only `OpenRA.Test/OpenRA.Game/PlatformTest.cs` uses `[SetUp]`; no `[TearDown]`, `[OneTimeSetUp]`, or `[OneTimeTearDown]` were detected.
- Arrange/Act/Assert comments are optional. Newer tests sometimes use them (`OpenRA.Test/OpenRA.Game/PngTest.cs`), while most legacy tests go straight from setup to assertions (`OpenRA.Test/OpenRA.Game/CPosTest.cs`, `OpenRA.Test/OpenRA.Mods.Common/ShapeTest.cs`).

## Mocking

**Framework:**
- No dedicated mocking library is present in `OpenRA.Test/OpenRA.Test.csproj` or test source.
- Tests use inline fakes, nested helper types, or small custom helper classes instead.

**Patterns:**
```csharp
interface IMock : ITraitInfoInterface { }
class MockTraitInfo : TraitInfo { public override object Create(ActorInitializer init) { return null; } }

sealed class MockAInfo : MockTraitInfo, IMock { }
sealed class MockBInfo : MockTraitInfo, Requires<IMock> { }

var actorInfo = new ActorInfo("test", new MockBInfo(), new MockCInfo());
var ex = Assert.Throws<YamlException>(() => actorInfo.TraitsInConstructOrder());
```
```csharp
sealed class TestStream : Stream
{
    readonly ManualResetEventSlim mres = new();
    readonly List<byte> bytes = [];
}
```

**What to Mock:**
- Inline trait/info doubles for dependency-ordering and metadata tests: `OpenRA.Test/OpenRA.Game/ActorInfoTest.cs`.
- Tiny helper streams or scaffolding types when the production code expects a framework abstraction: `OpenRA.Test/OpenRA.Game/MiniYamlTest.cs`, `OpenRA.Test/OpenRA.Game/FieldLoaderTest.cs`.
- Nothing else unless the fake is smaller than constructing the real object graph.

**What NOT to Mock:**
- Core value types, parsers, serializers, geometry, and collection types are tested against real implementations: `OpenRA.Test/OpenRA.Game/CPosTest.cs`, `OpenRA.Test/OpenRA.Game/FieldLoaderTest.cs`, `OpenRA.Test/OpenRA.Mods.Common/ShapeTest.cs`.
- Mod YAML and Lua verification uses real assets and utility commands, not mocked inputs: `Makefile`, `make.ps1`, `OpenRA.Mods.Common/UtilityCommands/CheckYaml.cs`.

## Fixtures and Factories

**Test Data:**
```csharp
static IEnumerable<TestCaseData> FormatValue_Primitive_TestCases()
{
    return
    [
        new TestCaseData(123),
        new TestCaseData(Color.CornflowerBlue),
        new TestCaseData(new WVec(123, 456, 789)),
    ];
}
```
```csharp
const string FirstYaml =
@"Parent: First
    Child: First
Parent: Second
";
```

**Location:**
- Shared fixture or factory directories are not used.
- Test data usually lives inline as constants, arrays, `TestCaseData` enumerators, helper methods, or nested helper types inside the owning test file: `OpenRA.Test/OpenRA.Game/FieldLoaderTest.cs`, `OpenRA.Test/OpenRA.Game/FieldSaverTest.cs`, `OpenRA.Test/OpenRA.Game/MiniYamlTest.cs`, `OpenRA.Test/OpenRA.Mods.Common/PerfGraphWidgetTest.cs`.
- `OpenRA.Test/App.config` is the only shared runtime config and exists to probe `mods/common` assemblies for tests that need mod-side types.

## Coverage

**Requirements:**
- No coverage target or coverage publishing is configured in `OpenRA.Test/OpenRA.Test.csproj`, `Makefile`, `make.ps1`, or `.github/workflows/ci.yml`.
- CI only requires clean execution of build, analyzer, unit-test, Lua, and YAML-lint commands on Linux and Windows: `.github/workflows/ci.yml`.

**Configuration:**
- No Coverlet, ReportGenerator, or `dotnet test --collect` configuration was detected.
- The practical safety net is the combination of NUnit tests, Debug analyzer builds, Lua syntax checks, explicit-interface checks, conditional-trait override checks, and real mod/map YAML linting: `Makefile`, `make.ps1`, `OpenRA.Mods.Common/UtilityCommands/CheckYaml.cs`, `OpenRA.Mods.Common/UtilityCommands/CheckExplicitInterfacesCommand.cs`, `OpenRA.Mods.Common/UtilityCommands/CheckConditionalTraitInterfaceOverrides.cs`.

**View Coverage:**
```bash
Not detected
```

## Test Types

**Unit Tests:**
- `OpenRA.Test` contains classic unit tests for engine primitives, parsing/loading/saving, serialization, platform helpers, geometry, widget formatting, and selected `OpenRA.Mods.Common` behavior: `OpenRA.Test/OpenRA.Game/*.cs`, `OpenRA.Test/OpenRA.Mods.Common/*.cs`.
- Tests generally execute real production code with in-memory inputs and local helper types instead of isolating the subject behind mocks.

**Integration Tests:**
- Integration-style verification is command-driven instead of living in a dedicated NUnit suite.
- `make check` compiles the engine in Debug with analyzers as errors and runs explicit-interface plus conditional-trait override validation across utility-discoverable assemblies: `Makefile`, `make.ps1`, `OpenRA.Mods.Common/UtilityCommands/CheckExplicitInterfacesCommand.cs`, `OpenRA.Mods.Common/UtilityCommands/CheckConditionalTraitInterfaceOverrides.cs`.
- `make test` loads real official mod data and maps through `OpenRA.Utility --check-yaml`: `Makefile`, `make.ps1`, `OpenRA.Mods.Common/UtilityCommands/CheckYaml.cs`, `OpenRA.Utility/Program.cs`.
- `make check-scripts` runs `luac -p` over Lua files under `mods/*/maps/` and `mods/*/scripts/`: `Makefile`, `make.ps1`, `mods/cnc/maps/gdi01/gdi01.lua`, `mods/common/scripts/utils.lua`.

**E2E Tests:**
- Not detected. No browser automation, gameplay replay harness, or full UI flow framework exists in the repository.

## Common Patterns

**Async Testing:**
```csharp
var readTask = Task.Run(() =>
{
    foreach (var node in MiniYaml.FromStream(stream, ""))
        events.Add(("Saw Node", new[] { node }.WriteToString()));
});

if (!readTask.Wait(TimeSpan.FromSeconds(1)))
    Assert.Fail("Timeout waiting for task completion");
```
- Async and concurrency tests are rare. When they exist, they use real `Task`, `AutoResetEvent`, and timeout-based assertions instead of async test attributes: `OpenRA.Test/OpenRA.Game/MiniYamlTest.cs`.

**Error Testing:**
```csharp
void Act() => FieldLoader.GetValue<object>("field", "test");

Assert.That(Act,
    Throws.TypeOf<NotImplementedException>()
        .And.Message.EqualTo("FieldLoader: Missing field `[Type] test` on `Object`"));
```
```csharp
var ex = Assert.Throws<InvalidDataException>(() => new Png(new MemoryStream(invalidSignature)));
Assert.That("PNG Signature is bogus", Does.Contain(ex.Message));
```
- Use both `Assert.Throws<T>()` and constraint-based `Throws.TypeOf<T>().And.Message...` assertions: `OpenRA.Test/OpenRA.Game/ActorInfoTest.cs`, `OpenRA.Test/OpenRA.Game/PngTest.cs`, `OpenRA.Test/OpenRA.Game/FieldLoaderTest.cs`.

**Snapshot Testing:**
- Not used. String serializers and formatters are verified with direct string equality or round-trip assertions: `OpenRA.Test/OpenRA.Game/MiniYamlTest.cs`, `OpenRA.Test/OpenRA.Game/FieldSaverTest.cs`, `OpenRA.Test/OpenRA.Mods.Common/PerfGraphWidgetTest.cs`.
- Two tests are intentionally ignored with `[Ignore("Failing test should be fixed")]` in `OpenRA.Test/OpenRA.Game/PngTest.cs`. Preserve or change ignored tests deliberately instead of copying the pattern casually.

---

*Testing analysis: 2026-03-26*
