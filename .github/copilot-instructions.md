# Copilot Instructions for MauiMocks

## Project Overview

MauiMocks is a .NET library (NuGet package `Mocks.Maui`) that enables unit testing of .NET MAUI code without deploying to a device or simulator. It mocks the essential MAUI services (application, dispatcher, device info, device display, font manager, animation manager, etc.) so that MAUI UI classes can be exercised in plain xUnit tests.

## Repository Structure

```
MauiMocks/                  # Main library project (namespace: Microsoft.Maui)
  MauiMocks.cs              # Entry point: MauiMocks.Init() / MauiMocks.Reset()
  MockApplication.cs
  MockApplicationHandler.cs
  MockMauiContext.cs        # IMauiContext backed by MockServiceProvider
  MockServiceProvider.cs    # Registers default mocks for IAnimationManager, IFontManager, etc.
  MockDispatcher.cs
  MockDispatcherProvider.cs
  MockDispatcherTimer.cs
  MockDeviceDisplay.cs
  MockDeviceInfo.cs
  MockFontManager.cs
  MockFontRegistrar.cs
  MockAnimationManager.cs
  MockAnimationHandler.cs   # ViewHandler used to attach mock animations to a view
  AnimationExtensions.cs    # Extension: view.UseMockAnimationHandler()
  Controls/
    MockContentView.cs
    MockListView.cs
  Timers/
    SyncTimer.cs            # Synchronous ITicker (default for animations in tests)
    AsyncTicker.cs

Tests/MauiMocks.Tests/      # xUnit test project targeting net9.0
  MauiMocksTests.cs
  EventToCommandBehaviorTests.cs
```

## Key Concepts

### Initialization Pattern
Every test class that uses MAUI must call `MauiMocks.Init()` before any MAUI API access, and `MauiMocks.Reset()` in teardown (typically `IDisposable.Dispose`). Tests **must not run concurrently** — use an xUnit `[Collection("MauiMocks")]` attribute on each test class to serialize them.

```csharp
[Collection("MauiMocks")]
public class MyTests : IDisposable
{
    public MyTests() => MauiMocks.Init();
    public void Dispose() => MauiMocks.Reset();
}
```

### Overriding Individual Mocks
After `MauiMocks.Init()`, individual services can be replaced with custom mocks using the static helper classes:

```csharp
MauiMocks.DeviceInfo.SetCurrent(myMockDeviceInfo);
MauiMocks.DeviceDisplay.SetCurrent(myMockDeviceDisplay);
```

These use reflection to call `internal static SetCurrent` methods on the MAUI types.

### Animation Testing
Use `view.UseMockAnimationHandler()` (from `AnimationExtensions`) to attach a `MockAnimationHandler` (with `SyncTimer`) to any `IView`. This is necessary for any test that triggers animations.

### MockServiceProvider
`MockServiceProvider` pre-registers defaults for `IAnimationManager`, `IFontRegistrar`, `IFontManager`, and `IDispatcher`. Additional services can be injected at construction via tuples:
```csharp
new MockMauiContext((typeof(IMyService), myServiceInstance))
```

## Target Frameworks & Dependencies

- **Library**: multi-targets `net8.0` and `net9.0`; depends on `Microsoft.Maui.Controls` and `Microsoft.Maui.Controls.Compatibility`
- **Tests**: single-target `net9.0`; uses xUnit 2.x, FluentAssertions 6.x, Moq.AutoMock 3.x, CommunityToolkit.Maui 11.x, coverlet

## Building and Testing

Requires the .NET MAUI workload. To install:
```bash
dotnet workload install maui
```

Build and test from the repo root:
```bash
dotnet build MauiMocks.sln --configuration Release
dotnet test Tests/MauiMocks.Tests/MauiMocks.Tests.csproj --configuration Release
```

The CI pipeline (Azure Pipelines, `azure-pipelines.yml`) runs on `windows-2022`, installs the MAUI workload, builds the solution, runs tests with coverlet coverage, and packs the NuGet package.

## Code Style

Enforced by `.editorconfig`:
- **Indentation**: 4 spaces for `.cs`, 2 spaces for `.csproj`; CRLF line endings
- **Naming**: `camelCase` for private fields and parameters; `PascalCase` for public members, types, and static fields; interfaces prefixed with `I`
- **`this.` qualifier**: required for all field, property, method, and event access
- **`var`**: preferred when type is apparent; required for casts
- **Braces**: always required (`csharp_prefer_braces = true:error`)
- **Null propagation and coalescing**: enforced (`dotnet_style_null_propagation = true:error`)
- **`async` without `await`**: treated as error (`CS4014`)
- Max line length: 140 characters

## Common Patterns

- All mock classes live in the `Microsoft.Maui` namespace (matching the real MAUI namespace), so tests can use real MAUI types and mocks interchangeably without extra `using` statements.
- New mock classes follow the naming convention `Mock<TypeName>` and implement the corresponding MAUI interface.
- When adding a new mock service, register it as a default in `MockServiceProvider` (with a `ContainsKey` guard so callers can override it).
- Test files use Arrange/Act/Assert comment blocks and FluentAssertions (`.Should().Be(...)` etc.).

## Known Constraints

- Tests must be serialized (not parallel) because `MauiMocks.Init/Reset` mutate global static state in the MAUI framework.
- `MockServiceProvider.GetService` throws a `KeyNotFoundException` for unregistered types; callers must ensure required services are registered before use.
- The library uses reflection to call `internal` MAUI methods (`SetCurrent`). These may break if MAUI internals change across versions; check `MauiMocks.cs` → `InvokeStaticMethod` when upgrading MAUI.
