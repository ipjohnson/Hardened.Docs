# Your First Lambda Function

This tutorial walks you through building an AWS Lambda function with Hardened. You will create a simple request/response handler, wire up dependency injection, and write a test using `LambdaTestApp`.

**What you will build:**

- A Lambda function that processes a request and returns a response
- A service registered via compile-time DI
- A unit test using `LambdaTestApp`

**Time:** ~10 minutes

---

## Prerequisites

- [.NET 8 SDK](https://dotnet.microsoft.com/download/dotnet/8.0) or later
- [NuGet configured for GitHub Packages](nuget-setup.md)

---

## Step 1: Create the project

```bash
dotnet new classlib -n OrderProcessor
cd OrderProcessor
```

Add the Hardened Lambda source generator:

```bash
dotnet add package Hardened.Amz.Function.Lambda.SourceGenerator --prerelease
```

---

## Step 2: Define request and response models

Create a `Models/` directory with your data types:

```csharp title="Models/OrderRequest.cs"
public class OrderRequest
{
    public string OrderId { get; set; } = string.Empty;
    public string CustomerId { get; set; } = string.Empty;
    public decimal Amount { get; set; }
}
```

```csharp title="Models/OrderResponse.cs"
public class OrderResponse
{
    public string OrderId { get; set; } = string.Empty;
    public string Status { get; set; } = string.Empty;
    public DateTime ProcessedAt { get; set; }
}
```

---

## Step 3: Create the Application module

```csharp title="Application.cs"
using Hardened.Shared.Runtime.Attributes;

[HardenedModule]
public partial class Application { }
```

For a basic Lambda function, you only need `[HardenedModule]`. No runtime module attribute is required -- the Lambda source generator handles the runtime wiring.

---

## Step 4: Create a service

```csharp title="Services/IOrderService.cs"
public interface IOrderService
{
    Task<OrderResponse> ProcessOrder(OrderRequest request);
}
```

```csharp title="Services/OrderService.cs"
using DependencyModules.Runtime.Attributes;

[TransientService]
public class OrderService : IOrderService
{
    public Task<OrderResponse> ProcessOrder(OrderRequest request)
    {
        var response = new OrderResponse
        {
            OrderId = request.OrderId,
            Status = "Processed",
            ProcessedAt = DateTime.UtcNow
        };

        return Task.FromResult(response);
    }
}
```

---

## Step 5: Create the function handler

```csharp title="Handlers/OrderHandler.cs"
using Hardened.Amz.Function.Lambda.Runtime.Attributes;

public class OrderHandler
{
    private readonly IOrderService _orderService;

    public OrderHandler(IOrderService orderService)
    {
        _orderService = orderService;
    }

    [HardenedFunction]
    public async Task<OrderResponse> Process(OrderRequest request)
    {
        return await _orderService.ProcessOrder(request);
    }
}
```

The `[HardenedFunction]` attribute marks this method as the Lambda entry point. The source generator:

1. Generates the Lambda bootstrap code
2. Deserializes the incoming event into `OrderRequest`
3. Resolves `OrderHandler` from DI (injecting `IOrderService`)
4. Calls `Process` and serializes the `OrderResponse` back

!!! note
    A project should have exactly one `[HardenedFunction]` method. If you need multiple Lambda functions, create separate projects for each.

---

## Step 6: Build

```bash
dotnet build
```

If the build succeeds, the source generator has produced the Lambda entry point and all DI wiring.

---

## Step 7: Add tests

Create a test project:

```bash
cd ..
dotnet new xunit -n OrderProcessor.Tests
cd OrderProcessor.Tests
dotnet add reference ../OrderProcessor/OrderProcessor.csproj
dotnet add package Hardened.Amz.Function.Lambda.Testing --prerelease
```

### Bootstrap file

```csharp title="Bootstrap.cs"
using Hardened.Shared.Runtime.Attributes;

[assembly: HardenedTestEntryPoint(typeof(Application))]
```

### Write a test

```csharp title="OrderHandlerTests.cs"
using Hardened.Shared.Runtime.Attributes;
using Hardened.Amz.Function.Lambda.Testing;

public class OrderHandlerTests
{
    [HardenedTest]
    public async Task Process_ReturnsProcessedOrder(LambdaTestApp testApp)
    {
        var request = new OrderRequest
        {
            OrderId = "ORD-001",
            CustomerId = "CUST-123",
            Amount = 99.99m
        };

        var response = await testApp.Invoke<OrderRequest, OrderResponse>(request);

        Assert.Equal("ORD-001", response.OrderId);
        Assert.Equal("Processed", response.Status);
    }
}
```

`LambdaTestApp` simulates the Lambda runtime locally. It uses the same DI container and request pipeline as production, so your tests exercise the full handler chain.

Run the tests:

```bash
dotnet test
```

---

## SQS Lambda variant

To build a Lambda that processes SQS messages, use the SQS runtime instead:

```bash
dotnet add package Hardened.Amz.Function.Sqs.Runtime --prerelease
```

```csharp title="Application.cs"
using Hardened.Shared.Runtime.Attributes;

[HardenedModule]
[SqsLambda.Module]
public partial class Application { }
```

```csharp title="Handlers/SqsHandler.cs"
using Hardened.Amz.Function.Lambda.Runtime.Attributes;

public class SqsHandler
{
    [HardenedFunction]
    public async Task Process(OrderRequest message)
    {
        // Process each SQS message body, deserialized as OrderRequest
        await Task.CompletedTask;
    }
}
```

The SQS runtime handles batch processing, individual message deserialization, and partial failure reporting automatically.

---

## DynamoDB Stream Lambda variant

To process DynamoDB Stream events:

```bash
dotnet add package Hardened.Amz.Function.DDB.Runtime --prerelease
```

```csharp title="Application.cs"
using Hardened.Shared.Runtime.Attributes;

[HardenedModule]
[DynamoStreamLambda.Module]
public partial class Application { }
```

```csharp title="Handlers/StreamHandler.cs"
using Amazon.DynamoDBv2.Model;
using Hardened.Amz.Function.Lambda.Runtime.Attributes;
using Hardened.Amz.Function.DDB.Runtime.Attributes;

public class StreamHandler
{
    [HardenedFunction]
    public async Task ProcessRecord(
        [OldImage] Dictionary<string, AttributeValue> oldImage,
        [NewImage] Dictionary<string, AttributeValue> newImage)
    {
        // Process each stream record
        await Task.CompletedTask;
    }
}
```

The `[OldImage]` and `[NewImage]` attributes bind the DynamoDB Stream record images to your method parameters.

---

## What just happened?

At compile time, the Hardened source generators:

1. Found `[HardenedFunction]` and generated the Lambda bootstrap entry point
2. Generated serialization/deserialization code for your request and response types
3. Discovered `[TransientService]` on `OrderService` and generated DI registrations
4. Wired everything together in the `Application` partial class

No `Amazon.Lambda.Core` boilerplate. No `FunctionHandler` string configuration. The source generator handles it all.

---

## Next steps

- [Project Structure](project-structure.md) -- organize larger Lambda projects
- [Lambda Function Runtime](../aws/lambda/function-runtime.md) -- advanced function configuration
- [SQS Processing](../aws/lambda/sqs-processing.md) -- deep dive into SQS batch handling
- [DDB Stream Processing](../aws/lambda/ddb-stream.md) -- DynamoDB Stream event processing
- [Lambda Testing](../aws/lambda/testing.md) -- advanced testing patterns with `LambdaTestApp`
