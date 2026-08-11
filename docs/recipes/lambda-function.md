# Recipe: Lambda Function

Build an AWS Lambda function that processes orders, with dependency injection and full test coverage using `LambdaTestApp`.

**What you will build:**

- A Lambda function with a typed request/response model
- A service layer with compile-time DI
- A validation layer
- Unit tests using `LambdaTestApp`

---

## Prerequisites

- [.NET 8 SDK](https://dotnet.microsoft.com/download/dotnet/8.0) or later
- [NuGet configured for GitHub Packages](../getting-started/nuget-setup.md)

---

## Project Setup

```bash
dotnet new classlib -n OrderProcessor
cd OrderProcessor
dotnet add package Hardened.Amz.Function.Lambda.SourceGenerator --prerelease
```

---

## Complete Code

### Models

```csharp title="Models/OrderRequest.cs"
public class OrderRequest
{
    public string OrderId { get; set; } = string.Empty;
    public string CustomerId { get; set; } = string.Empty;
    public List<OrderItem> Items { get; set; } = new();
    public string ShippingAddress { get; set; } = string.Empty;
}
```

```csharp title="Models/OrderItem.cs"
public class OrderItem
{
    public string ProductId { get; set; } = string.Empty;
    public string ProductName { get; set; } = string.Empty;
    public int Quantity { get; set; }
    public decimal UnitPrice { get; set; }
}
```

```csharp title="Models/OrderResponse.cs"
public class OrderResponse
{
    public string OrderId { get; set; } = string.Empty;
    public string Status { get; set; } = string.Empty;
    public decimal TotalAmount { get; set; }
    public DateTime ProcessedAt { get; set; }
    public string? ErrorMessage { get; set; }
}
```

### Services

```csharp title="Services/IOrderValidationService.cs"
public interface IOrderValidationService
{
    (bool IsValid, string? ErrorMessage) Validate(OrderRequest request);
}
```

```csharp title="Services/OrderValidationService.cs"
using DependencyModules.Runtime.Attributes;

[TransientService]
public class OrderValidationService : IOrderValidationService
{
    public (bool IsValid, string? ErrorMessage) Validate(OrderRequest request)
    {
        if (string.IsNullOrEmpty(request.OrderId))
            return (false, "OrderId is required");

        if (string.IsNullOrEmpty(request.CustomerId))
            return (false, "CustomerId is required");

        if (request.Items.Count == 0)
            return (false, "Order must contain at least one item");

        if (string.IsNullOrEmpty(request.ShippingAddress))
            return (false, "ShippingAddress is required");

        foreach (var item in request.Items)
        {
            if (item.Quantity <= 0)
                return (false, $"Invalid quantity for product {item.ProductId}");

            if (item.UnitPrice <= 0)
                return (false, $"Invalid price for product {item.ProductId}");
        }

        return (true, null);
    }
}
```

```csharp title="Services/IOrderService.cs"
public interface IOrderService
{
    Task<OrderResponse> ProcessOrder(OrderRequest request);
}
```

```csharp title="Services/OrderService.cs"
using DependencyModules.Runtime.Attributes;
using Microsoft.Extensions.Logging;

[TransientService]
public class OrderService : IOrderService
{
    private readonly IOrderValidationService _validationService;
    private readonly ILogger<OrderService> _logger;

    public OrderService(
        IOrderValidationService validationService,
        ILogger<OrderService> logger)
    {
        _validationService = validationService;
        _logger = logger;
    }

    public Task<OrderResponse> ProcessOrder(OrderRequest request)
    {
        _logger.LogInformation(
            "Processing order {OrderId} for customer {CustomerId}",
            request.OrderId, request.CustomerId);

        var (isValid, errorMessage) = _validationService.Validate(request);

        if (!isValid)
        {
            _logger.LogWarning(
                "Order {OrderId} validation failed: {Error}",
                request.OrderId, errorMessage);

            return Task.FromResult(new OrderResponse
            {
                OrderId = request.OrderId,
                Status = "ValidationFailed",
                ErrorMessage = errorMessage,
                ProcessedAt = DateTime.UtcNow
            });
        }

        var totalAmount = request.Items.Sum(
            item => item.Quantity * item.UnitPrice);

        var response = new OrderResponse
        {
            OrderId = request.OrderId,
            Status = "Processed",
            TotalAmount = totalAmount,
            ProcessedAt = DateTime.UtcNow
        };

        _logger.LogInformation(
            "Order {OrderId} processed successfully. Total: {Total}",
            request.OrderId, totalAmount);

        return Task.FromResult(response);
    }
}
```

### Handler

```csharp title="Handlers/OrderHandler.cs"
using Hardened.Amz.Function.Lambda.Runtime.Attributes;

public class OrderHandler
{
    private readonly IOrderService _orderService;

    public OrderHandler(IOrderService orderService)
    {
        _orderService = orderService;
    }

    [HardenedFunction("process-order")]
    public async Task<OrderResponse> Process(OrderRequest request)
    {
        return await _orderService.ProcessOrder(request);
    }
}
```

### Application Module

```csharp title="Application.cs"
using Hardened.Shared.Runtime.Attributes;

[HardenedModule]
public partial class Application { }
```

---

## Explanation

### Function Declaration

The `[HardenedFunction("process-order")]` attribute marks the method as a Lambda entry point. The string parameter is the function name used to identify the handler. The source generator produces:

1. The Lambda bootstrap entry point
2. JSON deserialization of the incoming event into `OrderRequest`
3. DI resolution of `OrderHandler` and all its dependencies
4. JSON serialization of the returned `OrderResponse`

### Service Registration

Both services use `[TransientService]`. Each Lambda invocation gets a fresh instance of both `OrderValidationService` and `OrderService`. Constructor injection works exactly as you would expect -- the framework resolves the full dependency graph automatically.

### Logging

`ILogger<T>` is available via the standard Microsoft.Extensions.Logging integration. In Lambda, logs are sent to CloudWatch Logs by default.

!!! tip
    For Lambda functions, keep handler classes thin. Put business logic in services so it can be tested independently and reused across different entry points.

---

## Testing

Create a test project:

```bash
cd ..
dotnet new xunit -n OrderProcessor.Tests
cd OrderProcessor.Tests
dotnet add reference ../OrderProcessor/OrderProcessor.csproj
dotnet add package Hardened.Amz.Function.Lambda.Testing --prerelease
```

### Bootstrap

```csharp title="Bootstrap.cs"
using Hardened.Shared.Runtime.Attributes;

[assembly: HardenedTestEntryPoint(typeof(Application))]
```

### Tests

```csharp title="OrderHandlerTests.cs"
using Hardened.Shared.Runtime.Attributes;
using Hardened.Amz.Function.Lambda.Testing;

public class OrderHandlerTests
{
    [HardenedTest]
    public async Task ProcessOrder_ValidOrder_ReturnsProcessed(
        LambdaTestApp testApp)
    {
        var request = new OrderRequest
        {
            OrderId = "ORD-001",
            CustomerId = "CUST-123",
            ShippingAddress = "123 Main St",
            Items = new List<OrderItem>
            {
                new OrderItem
                {
                    ProductId = "PROD-A",
                    ProductName = "Widget",
                    Quantity = 2,
                    UnitPrice = 19.99m
                },
                new OrderItem
                {
                    ProductId = "PROD-B",
                    ProductName = "Gadget",
                    Quantity = 1,
                    UnitPrice = 49.99m
                }
            }
        };

        var response = await testApp.Invoke<OrderRequest, OrderResponse>(
            "process-order", request);

        Assert.Equal("ORD-001", response.OrderId);
        Assert.Equal("Processed", response.Status);
        Assert.Equal(89.97m, response.TotalAmount);
        Assert.Null(response.ErrorMessage);
    }

    [HardenedTest]
    public async Task ProcessOrder_EmptyItems_ReturnsValidationFailed(
        LambdaTestApp testApp)
    {
        var request = new OrderRequest
        {
            OrderId = "ORD-002",
            CustomerId = "CUST-123",
            ShippingAddress = "123 Main St",
            Items = new List<OrderItem>()
        };

        var response = await testApp.Invoke<OrderRequest, OrderResponse>(
            "process-order", request);

        Assert.Equal("ValidationFailed", response.Status);
        Assert.Equal("Order must contain at least one item",
            response.ErrorMessage);
    }

    [HardenedTest]
    public async Task ProcessOrder_MissingCustomerId_ReturnsValidationFailed(
        LambdaTestApp testApp)
    {
        var request = new OrderRequest
        {
            OrderId = "ORD-003",
            CustomerId = "",
            ShippingAddress = "123 Main St",
            Items = new List<OrderItem>
            {
                new OrderItem
                {
                    ProductId = "PROD-A",
                    ProductName = "Widget",
                    Quantity = 1,
                    UnitPrice = 10.00m
                }
            }
        };

        var response = await testApp.Invoke<OrderRequest, OrderResponse>(
            "process-order", request);

        Assert.Equal("ValidationFailed", response.Status);
        Assert.Equal("CustomerId is required", response.ErrorMessage);
    }
}
```

Run the tests:

```bash
dotnet test
```

!!! note
    `LambdaTestApp` simulates the full Lambda runtime locally, including DI resolution and the execution pipeline. Your tests exercise the same code path as production invocations.

---

## Next Steps

- [Lambda Function Runtime](../aws/lambda/function-runtime.md) -- advanced Lambda configuration
- [Custom Execution Filter](custom-execution-filter.md) -- add logging, metrics, or error handling to your Lambda
- [SQS Consumer](sqs-consumer.md) -- trigger this function from an SQS queue
