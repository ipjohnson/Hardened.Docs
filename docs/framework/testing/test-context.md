# Test Context

`ITestContext` provides utility methods for writing structured, resilient tests. It includes named test steps, a retry engine for eventual consistency assertions, a logger, and a cancellation token. `ITestWebApp` extends `ITestContext`, so all of these features are available in web tests too.

**Package:** `Hardened.Shared.Testing` (namespace `Hardened.Shared.Testing`)

---

## ITestContext

### Definition

```csharp
using Microsoft.Extensions.Logging;

namespace Hardened.Shared.Testing;

public interface ITestContext {
    IRetryEngine Retry { get; }
    CancellationToken CancellationRequest { get; }

    void Step(Action step, string description, params object[] parameters);
    T Step<T>(Func<T> step, string description, params object[] parameters);
    Task Step(Func<Task> step, string description, params object[] parameters);
    Task<T> Step<T>(Func<Task<T>> step, string description, params object[] parameters);

    ILogger Logger { get; }
}
```

### Members

| Member | Description |
|---|---|
| `Retry` | Retry engine for polling-based assertions |
| `CancellationRequest` | Cancellation token for the test |
| `Step(...)` | Named test steps with logging |
| `Logger` | xUnit-integrated logger |

---

## Step -- Named Test Steps

The `Step` method wraps a test action with a descriptive name. Steps are logged to the test output, making test execution easier to follow when diagnosing failures.

### Synchronous Steps

```csharp
[HardenedTest]
public void CalculationTest(ITestContext context) {
    var calculator = new Calculator();

    var a = context.Step(
        () => calculator.Parse("42"),
        "Parse first operand");

    var b = context.Step(
        () => calculator.Parse("8"),
        "Parse second operand");

    context.Step(
        () => Assert.Equal(50, calculator.Add(a, b)),
        "Verify addition result");
}
```

### Async Steps

```csharp
[HardenedTest]
public async Task OrderWorkflow(ITestWebApp app) {
    var order = await app.Step(
        async () => {
            var response = await app.Post(
                new { customerId = "cust-1" },
                "/api/orders");
            response.Assert.Ok();
            return response.Deserialize<Order>();
        },
        "Create a new order");

    await app.Step(
        async () => {
            var response = await app.Get($"/api/orders/{order.Id}");
            response.Assert.Ok();
        },
        "Verify order exists");

    await app.Step(
        async () => {
            var response = await app.Delete($"/api/orders/{order.Id}");
            response.Assert.Ok();
        },
        "Delete the order");
}
```

### Parameterized Descriptions

Step descriptions support format parameters:

```csharp
await context.Step(
    async () => await service.Process(orderId),
    "Process order {0} for customer {1}",
    orderId, customerId);
```

This produces log output like: `Process order ord-123 for customer cust-42`.

---

## IRetryEngine

The retry engine is designed for eventual consistency testing. It repeatedly evaluates a condition until it passes or a timeout is reached. This is essential when testing async operations, event-driven systems, or distributed processes.

### Definition

```csharp
namespace Hardened.Shared.Testing;

public interface IRetryEngine {
    int Delay { get; set; }

    Task TillTrue(
        Func<Task<bool>> testFunc,
        string description,
        params object[] parameters);

    Task TillFalse(
        Func<Task<bool>> testFunc,
        string description,
        params object[] parameters);

    Task<T> TillValue<T>(
        Func<Task<T>> value,
        string description,
        params object[] parameters);
}
```

### Properties

| Property | Default | Description |
|---|---|---|
| `Delay` | 1000 | Milliseconds between retry attempts |

### Methods

| Method | Description |
|---|---|
| `TillTrue(func, desc)` | Retry until the function returns `true` |
| `TillFalse(func, desc)` | Retry until the function returns `false` |
| `TillValue<T>(func, desc)` | Retry until the function returns a non-null, non-default value |

### TillTrue -- Wait for a Condition

```csharp
[HardenedTest]
public async Task OrderProcessing_EventuallyCompletes(
    ITestContext context,
    IOrderService orderService) {
    var orderId = await orderService.Submit(new OrderRequest {
        CustomerId = "cust-1"
    });

    // Poll until the order reaches "Completed" status
    await context.Retry.TillTrue(
        async () => {
            var order = await orderService.GetById(orderId);
            return order?.Status == "Completed";
        },
        "Wait for order {0} to complete", orderId);
}
```

### TillFalse -- Wait for a Condition to Clear

```csharp
[HardenedTest]
public async Task Queue_EventuallyDrains(
    ITestContext context,
    IQueueService queue) {
    await queue.Enqueue(new Message { Body = "test" });

    // Poll until the queue is empty
    await context.Retry.TillFalse(
        async () => {
            var count = await queue.GetCount();
            return count > 0;
        },
        "Wait for queue to drain");
}
```

### TillValue -- Wait for a Non-Null Result

```csharp
[HardenedTest]
public async Task Projection_EventuallyAvailable(
    ITestContext context,
    IProjectionService projections) {
    // TillValue retries until a non-null value is returned
    var summary = await context.Retry.TillValue(
        async () => await projections.GetSummary("report-1"),
        "Wait for projection to be available");

    Assert.True(summary.TotalOrders > 0);
}
```

### Configuring Retry Delay

```csharp
[HardenedTest]
public async Task FastRetry(ITestContext context) {
    context.Retry.Delay = 200; // Check every 200ms instead of 1000ms

    await context.Retry.TillTrue(
        async () => {
            // Fast-polling check
            return await IsReady();
        },
        "Wait for readiness");
}
```

---

## Logger

`ITestContext.Logger` is an `ILogger` instance that writes to the xUnit test output. Use it for diagnostic logging during test execution:

```csharp
[HardenedTest]
public async Task DiagnosticTest(ITestContext context) {
    context.Logger.LogInformation("Starting test at {Time}", DateTime.UtcNow);

    var result = await DoSomething();
    context.Logger.LogDebug("Intermediate result: {@Result}", result);

    Assert.True(result.IsValid);
    context.Logger.LogInformation("Test passed");
}
```

Log output appears in the xUnit test runner output, making it visible in `dotnet test --verbosity detailed`, Visual Studio Test Explorer, and Rider's test runner.

---

## CancellationRequest

`ITestContext.CancellationRequest` provides a `CancellationToken` that signals when the test should abort. This is typically triggered by the test runner's timeout or manual cancellation:

```csharp
[HardenedTest]
public async Task LongRunning_RespectsCancel(ITestContext context) {
    while (!context.CancellationRequest.IsCancellationRequested) {
        var result = await CheckCondition();
        if (result) break;
        await Task.Delay(100, context.CancellationRequest);
    }
}
```

---

## Injecting ITestContext

`ITestContext` is available as a parameter in any `[HardenedTest]` method:

```csharp
[HardenedTest]
public async Task MyTest(
    ITestContext context,
    IOrderService orderService) {
    context.Logger.LogInformation("Running with ITestContext");
    // ...
}
```

Since `ITestWebApp` extends `ITestContext`, web tests have access to all context features:

```csharp
[HardenedTest]
public async Task WebTest(ITestWebApp app) {
    // app.Step, app.Retry, app.Logger, app.CancellationRequest
    // are all available through ITestWebApp
    app.Logger.LogInformation("Starting web test");

    await app.Step(async () => {
        var response = await app.Get("/api/health");
        response.Assert.Ok();
    }, "Check health endpoint");
}
```

---

## Complete Example

```csharp
using Hardened.Shared.Testing;
using Hardened.Shared.Testing.Attributes;
using Hardened.Web.Testing;
using NSubstitute;
using Xunit;

public class EventDrivenTests {
    [HardenedTest]
    public async Task OrderEvent_PublishesAndProcesses(
        ITestWebApp app,
        [Mock] IExternalPaymentApi paymentApi) {

        // Configure mock
        paymentApi.Charge(Arg.Any<ChargeRequest>())
            .Returns(Task.FromResult(new ChargeResult { Success = true }));

        // Step 1: Create order via API
        var order = await app.Step(async () => {
            var response = await app.Post(
                new { customerId = "cust-1", amount = 99.99m },
                "/api/orders");
            response.Assert.Ok();
            return response.Deserialize<Order>();
        }, "Create order via API");

        app.Logger.LogInformation("Order {OrderId} created", order.Id);

        // Step 2: Wait for async processing to complete
        await app.Step(async () => {
            await app.Retry.TillTrue(async () => {
                var response = await app.Get($"/api/orders/{order.Id}");
                var current = response.Deserialize<Order>();
                return current.Status == "Processed";
            }, "Wait for order to be processed");
        }, "Verify async processing completed");

        // Step 3: Verify the mock was called
        app.Step(() => {
            paymentApi.Received(1).Charge(
                Arg.Is<ChargeRequest>(r => r.Amount == 99.99m));
        }, "Verify payment API was called");
    }
}
```

---

## Related Pages

- [HardenedTest](hardened-test.md) -- test attribute and DI bootstrap
- [Web Testing](web-testing.md) -- `ITestWebApp` which extends `ITestContext`
- [Mocking](mocking.md) -- `[Mock]` for NSubstitute integration
- [Custom Attributes](custom-attributes.md) -- extending the test framework
