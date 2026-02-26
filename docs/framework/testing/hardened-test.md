# HardenedTest

`[HardenedTest]` is Hardened's test attribute that extends xUnit's `[Fact]` with automatic dependency injection, test discovery, and integration with the Hardened module system. Test method parameters are resolved from the DI container, giving you full access to your application's services in tests.

**Package:** `Hardened.Shared.Testing` (namespace `Hardened.Shared.Testing.Attributes`)

---

## [HardenedTest] Attribute

### Definition

```csharp
using Xunit;
using Xunit.Sdk;

namespace Hardened.Shared.Testing.Attributes;

[XunitTestCaseDiscoverer(
    "Hardened.Shared.Testing.Impl.HardenedTestDiscoverer",
    "Hardened.Shared.Testing")]
[AttributeUsage(AttributeTargets.Method, AllowMultiple = false)]
public class HardenedTestAttribute : FactAttribute { }
```

`[HardenedTest]` extends xUnit's `FactAttribute`, which means it is discovered by the standard xUnit test runner. The custom discoverer (`HardenedTestDiscoverer`) adds DI parameter injection and Hardened-specific lifecycle hooks.

---

## Bootstrap with [HardenedTestEntryPoint]

Before writing tests, you must tell the test framework which application module to use. Apply `[HardenedTestEntryPoint]` at the assembly, class, or method level.

### Definition

```csharp
namespace Hardened.Shared.Testing.Attributes;

[AttributeUsage(
    AttributeTargets.Assembly | AttributeTargets.Class | AttributeTargets.Method)]
public class HardenedTestEntryPointAttribute : Attribute {
    public HardenedTestEntryPointAttribute(Type entryPoint) { }
    public Type EntryPoint { get; }
}
```

### Assembly-Level Bootstrap (Recommended)

Create a file (e.g., `Bootstrap.cs`) in your test project:

```csharp
using Hardened.Shared.Testing.Attributes;

[assembly: HardenedTestEntryPoint(typeof(Application))]
```

This registers the `Application` module as the entry point for all tests in the assembly. The framework will use it to build the DI container for each test.

### Class-Level Bootstrap

Override the assembly-level entry point for a specific test class:

```csharp
[HardenedTestEntryPoint(typeof(AlternateApplication))]
public class AlternateModuleTests {
    [HardenedTest]
    public void TestSomething(IMyService service) {
        // Uses AlternateApplication's DI container
    }
}
```

### Method-Level Bootstrap

Override for a single test method:

```csharp
public class MixedTests {
    [HardenedTest]
    public void UsesDefaultModule(IMyService service) {
        // Uses the assembly-level entry point
    }

    [HardenedTest]
    [HardenedTestEntryPoint(typeof(MinimalApplication))]
    public void UsesMinimalModule(IMyService service) {
        // Uses MinimalApplication's DI container
    }
}
```

---

## Writing Tests

### Basic Test with DI

```csharp
using Hardened.Shared.Testing.Attributes;
using Xunit;

public class OrderServiceTests {
    [HardenedTest]
    public async Task CreateOrder_ReturnsOrder(IOrderService orderService) {
        var request = new CreateOrderRequest {
            CustomerId = "cust-1",
            Items = new[] { new OrderItem("prod-1", 2) }
        };

        var result = await orderService.Create(request);

        Assert.NotNull(result);
        Assert.Equal("cust-1", result.CustomerId);
        Assert.Single(result.Items);
    }
}
```

The `IOrderService` parameter is resolved from the DI container built from the `[HardenedTestEntryPoint]` module. No manual service setup is required.

### Multiple DI Parameters

You can inject as many services as you need:

```csharp
[HardenedTest]
public async Task ProcessOrder_UpdatesInventory(
    IOrderService orderService,
    IInventoryService inventoryService,
    ILogger<OrderServiceTests> logger) {

    logger.LogInformation("Starting test");

    var order = await orderService.Create(new CreateOrderRequest {
        CustomerId = "cust-1",
        Items = new[] { new OrderItem("prod-1", 5) }
    });

    var stock = await inventoryService.GetStock("prod-1");
    Assert.True(stock < 100); // Assuming initial stock of 100
}
```

### Combining with [Mock]

Use `[Mock]` to inject NSubstitute mocks alongside real services:

```csharp
[HardenedTest]
public async Task ProcessOrder_SendsNotification(
    IOrderService orderService,
    [Mock] INotificationService notifications) {

    var order = await orderService.Create(new CreateOrderRequest {
        CustomerId = "cust-1",
        Items = new[] { new OrderItem("prod-1", 1) }
    });

    await notifications.Received(1).Send(
        Arg.Is<string>(s => s.Contains("cust-1")),
        Arg.Any<string>());
}
```

See [Mocking](mocking.md) for details on the `[Mock]` attribute.

---

## Test Discovery and Execution

The `HardenedTestDiscoverer` processes each test method as follows:

1. **Discover** -- Find the `[HardenedTestEntryPoint]` (method > class > assembly)
2. **Build** -- Create an `IApplicationRoot` from the entry point module
3. **Register** -- Process `[Mock]` and other `IHardenedParameterProviderAttribute` attributes to register additional services
4. **Resolve** -- Create a scoped service provider and resolve each parameter
5. **Execute** -- Invoke the test method with the resolved parameters
6. **Dispose** -- Clean up the scoped provider and application root

Each test gets its own `IApplicationRoot` instance, ensuring test isolation.

---

## Test Project Setup

### Required Packages

```xml
<ItemGroup>
    <PackageReference Include="Hardened.Shared.Testing" Version="0.1.0-Alpha*" />
    <PackageReference Include="xunit" Version="2.*" />
    <PackageReference Include="xunit.runner.visualstudio" Version="2.*" />
    <PackageReference Include="Microsoft.NET.Test.Sdk" Version="17.*" />
</ItemGroup>
```

### Project Structure

```
MyApp.Tests/
    Bootstrap.cs              # [assembly: HardenedTestEntryPoint(typeof(Application))]
    OrderServiceTests.cs
    InventoryServiceTests.cs
    MyApp.Tests.csproj
```

### Bootstrap File

```csharp
// Bootstrap.cs
using Hardened.Shared.Testing.Attributes;

[assembly: HardenedTestEntryPoint(typeof(Application))]
```

!!! tip
    If your test project references the main application project, the `Application` class is available directly. For library modules, reference the module class instead.

---

## Running Tests

Hardened tests run with the standard `dotnet test` command:

```bash
dotnet test
```

They are also compatible with Visual Studio Test Explorer, JetBrains Rider, and any other xUnit-compatible test runner.

---

## Related Pages

- [Mocking](mocking.md) -- `[Mock]` attribute for NSubstitute mocks
- [Web Testing](web-testing.md) -- `ITestWebApp` for HTTP endpoint testing
- [Test Context](test-context.md) -- `ITestContext` for steps, retries, and logging
- [Custom Attributes](custom-attributes.md) -- extending the test framework
