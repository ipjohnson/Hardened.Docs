# Defining Canaries

Canaries are xUnit test methods decorated with the `[HardenedCanary]` attribute. Each canary defines a health check that runs on a schedule in production. Because canaries are standard xUnit facts, you can also run them locally with `dotnet test` during development.

---

## The [HardenedCanary] attribute

`[HardenedCanary]` extends xUnit's `FactAttribute` through the `HardenedTestDiscoverer`. Every public method with this attribute becomes both a runnable xUnit test and a scheduled canary.

```csharp
public class MyCanary
{
    [HardenedCanary]
    public async Task CheckSomething(ITestContext context)
    {
        await context.Step(async () =>
        {
            // health check logic
        }, "Verify something works");
    }
}
```

### Attribute properties

| Property | Type | Default | Description |
|---|---|---|---|
| `Name` | `string` | Method name | Display name for the canary in dashboards and logs |
| `Frequency` | `int` | `1` | How often the canary runs |
| `Unit` | `CanaryFrequencyUnit` | `Minute` | Time unit for the frequency |
| `ReportMetric` | `bool` | `true` | Whether to publish results to CloudWatch |
| `FlightStyle` | `CanaryFlightStyle` | `Loose` | How strictly the runtime enforces timing |
| `AllowConcurrentExecution` | `bool` | `false` | Whether multiple instances can run simultaneously |

---

## Frequency and units

The `Frequency` and `Unit` properties control how often a canary executes. Together they produce a `CanaryFrequency` record with a `Duration` property that returns the equivalent `TimeSpan`.

### CanaryFrequencyUnit enum

| Value | Description |
|---|---|
| `Second` | Frequency measured in seconds |
| `Minute` | Frequency measured in minutes (default) |
| `Hour` | Frequency measured in hours |
| `Day` | Frequency measured in days |

### Examples

```csharp
// Every 30 seconds
[HardenedCanary(Frequency = 30, Unit = CanaryFrequencyUnit.Second)]

// Every 5 minutes (most common)
[HardenedCanary(Frequency = 5, Unit = CanaryFrequencyUnit.Minute)]

// Every hour
[HardenedCanary(Frequency = 1, Unit = CanaryFrequencyUnit.Hour)]

// Once a day
[HardenedCanary(Frequency = 1, Unit = CanaryFrequencyUnit.Day)]
```

!!! warning
    Very high-frequency canaries (e.g., every few seconds) increase Lambda invocation costs and DynamoDB throughput. For most health checks, a frequency of 1-5 minutes provides a good balance between detection speed and cost.

---

## Flight styles

The `FlightStyle` property controls how the runtime handles canary execution timing.

### CanaryFlightStyle enum

| Value | Behavior |
|---|---|
| `Loose` (default) | The canary is considered failed only if it exceeds the timeout. The timeout defaults to the frequency duration. A canary scheduled every 5 minutes can take up to 5 minutes to complete without being flagged. |
| `Strict` | The canary is considered failed if its execution time exceeds its frequency. This is useful when you want tight guarantees that canaries complete well within their scheduling window. |

```csharp
// Loose: only fails if execution exceeds the 5-minute timeout
[HardenedCanary(
    Frequency = 5,
    Unit = CanaryFrequencyUnit.Minute,
    FlightStyle = CanaryFlightStyle.Loose)]
public async Task RelaxedCheck(ITestContext context) { ... }

// Strict: fails if execution takes longer than the 5-minute frequency
[HardenedCanary(
    Frequency = 5,
    Unit = CanaryFrequencyUnit.Minute,
    FlightStyle = CanaryFlightStyle.Strict)]
public async Task StrictCheck(ITestContext context) { ... }
```

!!! tip
    Use `Loose` (the default) for most canaries. Reserve `Strict` for checks where execution time is a critical signal -- for example, latency-sensitive API calls where slow responses indicate degradation.

---

## ITestContext

Every canary method receives an `ITestContext` parameter that provides structured test execution capabilities.

### Step()

Use `context.Step()` to organize your canary into named steps. Each step is logged and timed individually, making it easier to identify which part of a canary failed.

```csharp
[HardenedCanary(Frequency = 5, Unit = CanaryFrequencyUnit.Minute)]
public async Task CheckOrderWorkflow(ITestContext context)
{
    string orderId = null!;

    await context.Step(async () =>
    {
        using var client = new HttpClient();
        var response = await client.PostAsJsonAsync(
            "https://api.example.com/orders",
            new { Item = "Widget", Quantity = 1 });
        response.EnsureSuccessStatusCode();

        var result = await response.Content.ReadFromJsonAsync<OrderResponse>();
        orderId = result!.OrderId;
    }, "Create order");

    await context.Step(async () =>
    {
        using var client = new HttpClient();
        var response = await client.GetAsync(
            $"https://api.example.com/orders/{orderId}");
        response.EnsureSuccessStatusCode();
    }, "Retrieve order");
}
```

### Retry engine

`ITestContext` exposes a `Retry` property of type `IRetryEngine` for polling-style assertions. This is useful when you need to wait for an eventually-consistent state.

| Method | Description |
|---|---|
| `TillTrue(Func<Task<bool>>)` | Retries until the function returns `true` |
| `TillFalse(Func<Task<bool>>)` | Retries until the function returns `false` |
| `TillValue(Func<Task<T>>)` | Retries until the function returns a non-default value |

```csharp
[HardenedCanary(Frequency = 10, Unit = CanaryFrequencyUnit.Minute)]
public async Task CheckAsyncProcessing(ITestContext context)
{
    string jobId = null!;

    await context.Step(async () =>
    {
        using var client = new HttpClient();
        var response = await client.PostAsJsonAsync(
            "https://api.example.com/jobs",
            new { Type = "process-data" });
        var result = await response.Content.ReadFromJsonAsync<JobResponse>();
        jobId = result!.JobId;
    }, "Submit job");

    await context.Step(async () =>
    {
        await context.Retry.TillTrue(async () =>
        {
            using var client = new HttpClient();
            var response = await client.GetFromJsonAsync<JobStatus>(
                $"https://api.example.com/jobs/{jobId}");
            return response?.State == "completed";
        });
    }, "Wait for job completion");
}
```

### Logger

Access `context.Logger` to write diagnostic output. Log messages appear in CloudWatch Logs alongside the canary execution.

```csharp
[HardenedCanary(Frequency = 1, Unit = CanaryFrequencyUnit.Hour)]
public async Task CheckDatabaseConnectivity(ITestContext context)
{
    await context.Step(async () =>
    {
        context.Logger.LogInformation("Checking database connectivity...");
        // database check logic
    }, "Connect to database");
}
```

### CancellationRequest

`context.CancellationRequest` provides a `CancellationToken` that is triggered when the canary exceeds its timeout. Use it with async operations that accept cancellation:

```csharp
[HardenedCanary(Frequency = 5, Unit = CanaryFrequencyUnit.Minute)]
public async Task CheckWithTimeout(ITestContext context)
{
    await context.Step(async () =>
    {
        using var client = new HttpClient();
        var response = await client.GetAsync(
            "https://api.example.com/slow-endpoint",
            context.CancellationRequest);
        response.EnsureSuccessStatusCode();
    }, "Call slow endpoint");
}
```

---

## Multiple canaries in one class

A single class can contain multiple canary methods, each with its own schedule. This is useful for grouping related health checks.

```csharp
public class PaymentServiceCanary
{
    [HardenedCanary(
        Name = "PaymentGatewayHealth",
        Frequency = 2,
        Unit = CanaryFrequencyUnit.Minute)]
    public async Task CheckGatewayHealth(ITestContext context)
    {
        await context.Step(async () =>
        {
            using var client = new HttpClient();
            var response = await client.GetAsync(
                "https://payments.example.com/health");
            response.EnsureSuccessStatusCode();
        }, "Check payment gateway");
    }

    [HardenedCanary(
        Name = "PaymentProcessingE2E",
        Frequency = 15,
        Unit = CanaryFrequencyUnit.Minute)]
    public async Task CheckPaymentProcessing(ITestContext context)
    {
        await context.Step(async () =>
        {
            // end-to-end payment test with test credentials
        }, "Process test payment");

        await context.Step(async () =>
        {
            // verify payment was recorded
        }, "Verify payment record");
    }

    [HardenedCanary(
        Name = "RefundCapability",
        Frequency = 1,
        Unit = CanaryFrequencyUnit.Hour)]
    public async Task CheckRefundCapability(ITestContext context)
    {
        await context.Step(async () =>
        {
            // verify refund endpoint is responsive
        }, "Check refund endpoint");
    }
}
```

Each method is discovered and scheduled independently. The `Name` property controls how the canary appears in dashboards and metrics -- if omitted, the method name is used.

---

## Dependency injection in canaries

Canary classes support constructor injection. Services registered with `[Expose]`, `[Singleton]`, or `[Scoped]` in your application module are available.

```csharp
public class DatabaseCanary
{
    private readonly IDatabaseClient _db;

    public DatabaseCanary(IDatabaseClient db)
    {
        _db = db;
    }

    [HardenedCanary(Frequency = 5, Unit = CanaryFrequencyUnit.Minute)]
    public async Task CheckDatabaseHealth(ITestContext context)
    {
        await context.Step(async () =>
        {
            var result = await _db.PingAsync();
            Assert.True(result.IsHealthy, "Database ping failed");
        }, "Ping database");
    }

    [HardenedCanary(Frequency = 1, Unit = CanaryFrequencyUnit.Hour)]
    public async Task CheckDatabaseReadWrite(ITestContext context)
    {
        var testKey = $"canary-test-{Guid.NewGuid()}";

        await context.Step(async () =>
        {
            await _db.PutAsync(testKey, "canary-value");
        }, "Write test record");

        await context.Step(async () =>
        {
            var value = await _db.GetAsync(testKey);
            Assert.Equal("canary-value", value);
        }, "Read test record");

        await context.Step(async () =>
        {
            await _db.DeleteAsync(testKey);
        }, "Clean up test record");
    }
}
```

!!! note
    Canary classes are instantiated per invocation, so constructor-injected services receive a fresh DI scope for each run. Singleton services are shared across invocations within the same Lambda instance.

---

## Controlling metric reporting

By default, all canaries report metrics to CloudWatch (`ReportMetric = true`). Set `ReportMetric = false` to suppress metric reporting for canaries used purely for logging or diagnostics.

```csharp
// Reports metrics (default)
[HardenedCanary(Frequency = 5, Unit = CanaryFrequencyUnit.Minute)]
public async Task CriticalCheck(ITestContext context) { ... }

// No metrics -- diagnostic only
[HardenedCanary(Frequency = 1, Unit = CanaryFrequencyUnit.Hour, ReportMetric = false)]
public async Task DiagnosticCheck(ITestContext context) { ... }
```

---

## Running canaries locally

Because `[HardenedCanary]` extends `FactAttribute`, canaries run as standard xUnit tests:

```bash
dotnet test
```

This lets you verify your canary logic during development before deploying to Lambda. Locally, the scheduling and CloudWatch integration are not active -- only the test method body executes.

!!! tip
    Use environment variables or configuration to switch between production and test endpoints, so your canaries can run locally against a staging environment.

---

## Next steps

- [Flight Control](flight-control.md) -- understand how canaries are scheduled and executed
- [CloudWatch Integration](cloudwatch-integration.md) -- set up metrics and dashboards for your canaries
