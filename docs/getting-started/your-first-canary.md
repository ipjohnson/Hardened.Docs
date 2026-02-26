# Your First Canary

This tutorial walks you through building a canary test with Hardened. Canaries are automated health checks that run on a schedule as AWS Lambda functions, verifying your production services are functioning correctly.

**What you will build:**

- A canary that checks a health endpoint on a recurring schedule
- A multi-step canary with structured test steps

**Time:** ~10 minutes

---

## Prerequisites

- [.NET 8 SDK](https://dotnet.microsoft.com/download/dotnet/8.0) or later
- [NuGet configured for GitHub Packages](nuget-setup.md)

---

## Step 1: Create the project

```bash
dotnet new classlib -n ServiceCanary
cd ServiceCanary
```

Add the Hardened Canaries runtime:

```bash
dotnet add package Hardened.Amz.Canaries.Runtime --prerelease
```

---

## Step 2: Create the Application module

```csharp title="Application.cs"
using Hardened.Shared.Runtime.Attributes;

[HardenedModule]
public partial class Application { }
```

---

## Step 3: Create your first canary

```csharp title="Canaries/HealthCheckCanary.cs"
using Hardened.Amz.Canaries.Runtime.Attributes;
using Hardened.Amz.Canaries.Runtime;

public class HealthCheckCanary
{
    [HardenedCanary(Frequency = 5, Unit = CanaryFrequencyUnit.Minute)]
    public async Task CheckHealthEndpoint(ITestContext context)
    {
        await context.Step(async () =>
        {
            using var httpClient = new HttpClient();
            var response = await httpClient.GetAsync("https://api.example.com/health");
            response.EnsureSuccessStatusCode();
        }, "Call health endpoint");

        await context.Step(async () =>
        {
            using var httpClient = new HttpClient();
            var response = await httpClient.GetAsync("https://api.example.com/health");
            var body = await response.Content.ReadAsStringAsync();

            if (!body.Contains("\"status\":\"healthy\""))
            {
                throw new Exception("Health check response did not contain healthy status");
            }
        }, "Verify health response body");
    }
}
```

---

## Understanding `[HardenedCanary]`

The `[HardenedCanary]` attribute extends xUnit's `[Fact]` attribute, which means canary methods work both as scheduled Lambda functions in production **and** as standard xUnit tests during development.

### Properties

| Property | Type | Description |
|---|---|---|
| `Frequency` | `int` | How often the canary runs |
| `Unit` | `CanaryFrequencyUnit` | Time unit: `Minute`, `Hour` |
| `FlightStyle` | `CanaryFlightStyle` | Execution mode (see below) |

### Frequency examples

```csharp
// Run every 5 minutes
[HardenedCanary(Frequency = 5, Unit = CanaryFrequencyUnit.Minute)]

// Run every hour
[HardenedCanary(Frequency = 1, Unit = CanaryFrequencyUnit.Hour)]

// Run every 15 minutes
[HardenedCanary(Frequency = 15, Unit = CanaryFrequencyUnit.Minute)]
```

### Flight styles

The `FlightStyle` property controls how the canary executes:

| Style | Description |
|---|---|
| `CanaryFlightStyle.Heartbeat` | Runs on a fixed schedule (default) |

```csharp
[HardenedCanary(
    Frequency = 5,
    Unit = CanaryFrequencyUnit.Minute,
    FlightStyle = CanaryFlightStyle.Heartbeat)]
public async Task MyCanary(ITestContext context)
{
    // ...
}
```

---

## Step 4: Using test steps

The `ITestContext.Step` method structures your canary into discrete, named steps. Each step:

- Is reported individually to CloudWatch
- Provides clear failure diagnostics when something goes wrong
- Can be monitored and alarmed on independently

```csharp title="Canaries/ApiCanary.cs"
using Hardened.Amz.Canaries.Runtime.Attributes;
using Hardened.Amz.Canaries.Runtime;

public class ApiCanary
{
    [HardenedCanary(Frequency = 10, Unit = CanaryFrequencyUnit.Minute)]
    public async Task VerifyOrderFlow(ITestContext context)
    {
        string orderId = string.Empty;

        await context.Step(async () =>
        {
            using var client = new HttpClient();
            var content = new StringContent(
                """{"customerId": "test-123", "amount": 1.00}""",
                System.Text.Encoding.UTF8,
                "application/json");

            var response = await client.PostAsync(
                "https://api.example.com/orders", content);
            response.EnsureSuccessStatusCode();

            var body = await response.Content.ReadAsStringAsync();
            // Parse orderId from response
        }, "Create test order");

        await context.Step(async () =>
        {
            using var client = new HttpClient();
            var response = await client.GetAsync(
                $"https://api.example.com/orders/{orderId}");
            response.EnsureSuccessStatusCode();
        }, "Retrieve created order");
    }
}
```

---

## Step 5: Using dependency injection

Canary classes support constructor injection, just like any other Hardened component:

```csharp title="Services/IApiClient.cs"
public interface IApiClient
{
    Task<HttpResponseMessage> GetHealth();
}
```

```csharp title="Services/ApiClient.cs"
using Hardened.Shared.Runtime.Attributes;

[Expose]
public class ApiClient : IApiClient
{
    private readonly HttpClient _httpClient;

    public ApiClient(HttpClient httpClient)
    {
        _httpClient = httpClient;
    }

    public async Task<HttpResponseMessage> GetHealth()
    {
        return await _httpClient.GetAsync("https://api.example.com/health");
    }
}
```

```csharp title="Canaries/HealthCheckCanary.cs"
using Hardened.Amz.Canaries.Runtime.Attributes;
using Hardened.Amz.Canaries.Runtime;

public class HealthCheckCanary
{
    private readonly IApiClient _apiClient;

    public HealthCheckCanary(IApiClient apiClient)
    {
        _apiClient = apiClient;
    }

    [HardenedCanary(Frequency = 5, Unit = CanaryFrequencyUnit.Minute)]
    public async Task CheckHealth(ITestContext context)
    {
        await context.Step(async () =>
        {
            var response = await _apiClient.GetHealth();
            response.EnsureSuccessStatusCode();
        }, "Call health endpoint");
    }
}
```

---

## Step 6: Using configuration

Use `[ConfigurationModel]` to pull configuration from environment variables:

```csharp title="Config/ICanaryConfig.cs"
using Hardened.Shared.Runtime.Attributes;

[ConfigurationModel]
public interface ICanaryConfig
{
    [FromEnvironmentVariable("API_BASE_URL")]
    string ApiBaseUrl { get; }
}
```

The source generator creates an implementation that reads `API_BASE_URL` from the environment at runtime. Inject `ICanaryConfig` wherever you need it.

---

## Step 7: Build and run locally

Since `[HardenedCanary]` extends xUnit's `[Fact]`, you can run your canaries as standard unit tests during development:

```bash
dotnet test
```

This executes each canary method as a test, making it easy to iterate locally before deploying.

---

## What just happened?

At compile time, the Hardened source generators:

1. Found `[HardenedCanary]` methods and generated Lambda entry points for each
2. Generated scheduling metadata (frequency, unit, flight style) for CDK deployment
3. Discovered `[Expose]` services and generated DI registrations
4. Wired `ITestContext` injection for step-based execution

The canary runs as a Lambda function on a CloudWatch Events schedule, with each step reported as a metric.

---

## Next steps

- [Defining Canaries](../canaries/defining-canaries.md) -- advanced canary patterns
- [Flight Control](../canaries/flight-control.md) -- managing canary execution
- [CloudWatch Integration](../canaries/cloudwatch-integration.md) -- metrics, alarms, and dashboards
- [Project Structure](project-structure.md) -- organize canary projects
