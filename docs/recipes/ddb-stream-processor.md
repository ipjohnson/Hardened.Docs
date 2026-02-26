# Recipe: DynamoDB Stream Processor

Build a Lambda function that processes DynamoDB Stream events, comparing old and new images to detect changes and trigger downstream actions.

**What you will build:**

- A DynamoDB Stream processor using `[DynamoStreamLambda.Module]`
- A handler that receives `[OldImage]` and `[NewImage]` for each record
- Change detection logic based on image comparison
- Error handling for stream processing
- Tests using `LambdaTestApp`

---

## Prerequisites

- [.NET 8 SDK](https://dotnet.microsoft.com/download/dotnet/8.0) or later
- [NuGet configured for GitHub Packages](../getting-started/nuget-setup.md)

---

## Project Setup

```bash
dotnet new classlib -n OrderStreamProcessor
cd OrderStreamProcessor
dotnet add package Hardened.Amz.Function.Lambda.SourceGenerator --prerelease
dotnet add package Hardened.Amz.Function.DDB.Runtime --prerelease
dotnet add package AWSSDK.DynamoDBv2
```

---

## Complete Code

### Models

```csharp title="Models/OrderRecord.cs"
public class OrderRecord
{
    public string OrderId { get; set; } = string.Empty;
    public string CustomerId { get; set; } = string.Empty;
    public string Status { get; set; } = string.Empty;
    public decimal TotalAmount { get; set; }
    public DateTime UpdatedAt { get; set; }
}
```

```csharp title="Models/OrderChangeEvent.cs"
public class OrderChangeEvent
{
    public string OrderId { get; set; } = string.Empty;
    public string? PreviousStatus { get; set; }
    public string CurrentStatus { get; set; } = string.Empty;
    public ChangeType ChangeType { get; set; }
}

public enum ChangeType
{
    Created,
    Updated,
    Deleted
}
```

### Services

```csharp title="Services/IOrderChangeService.cs"
public interface IOrderChangeService
{
    Task HandleOrderCreated(OrderRecord newOrder);
    Task HandleOrderUpdated(OrderRecord oldOrder, OrderRecord newOrder);
    Task HandleOrderDeleted(OrderRecord oldOrder);
}
```

```csharp title="Services/OrderChangeService.cs"
using Hardened.Shared.Runtime.Attributes;
using Microsoft.Extensions.Logging;

[Expose]
public class OrderChangeService : IOrderChangeService
{
    private readonly ILogger<OrderChangeService> _logger;

    public OrderChangeService(ILogger<OrderChangeService> logger)
    {
        _logger = logger;
    }

    public Task HandleOrderCreated(OrderRecord newOrder)
    {
        _logger.LogInformation(
            "New order created: {OrderId} for customer {CustomerId}, " +
            "amount: {Amount}",
            newOrder.OrderId, newOrder.CustomerId, newOrder.TotalAmount);

        // Send welcome notification, initialize fulfillment, etc.
        return Task.CompletedTask;
    }

    public Task HandleOrderUpdated(OrderRecord oldOrder, OrderRecord newOrder)
    {
        _logger.LogInformation(
            "Order {OrderId} updated: {OldStatus} -> {NewStatus}",
            newOrder.OrderId, oldOrder.Status, newOrder.Status);

        if (oldOrder.Status != newOrder.Status)
        {
            _logger.LogInformation(
                "Status transition detected for order {OrderId}",
                newOrder.OrderId);

            // Trigger status-specific actions
            if (newOrder.Status == "Shipped")
            {
                // Send shipping notification
            }
            else if (newOrder.Status == "Delivered")
            {
                // Send delivery confirmation
            }
            else if (newOrder.Status == "Cancelled")
            {
                // Process refund
            }
        }

        return Task.CompletedTask;
    }

    public Task HandleOrderDeleted(OrderRecord oldOrder)
    {
        _logger.LogWarning(
            "Order {OrderId} was deleted. Previous status: {Status}",
            oldOrder.OrderId, oldOrder.Status);

        // Archive or clean up related resources
        return Task.CompletedTask;
    }
}
```

### Helper for Image Conversion

```csharp title="Helpers/DynamoDbImageHelper.cs"
using Amazon.DynamoDBv2.Model;

public static class DynamoDbImageHelper
{
    public static OrderRecord? ToOrderRecord(
        Dictionary<string, AttributeValue>? image)
    {
        if (image == null || image.Count == 0)
            return null;

        return new OrderRecord
        {
            OrderId = image.GetValueOrDefault("OrderId")?.S ?? string.Empty,
            CustomerId = image.GetValueOrDefault("CustomerId")?.S
                ?? string.Empty,
            Status = image.GetValueOrDefault("Status")?.S ?? string.Empty,
            TotalAmount = decimal.TryParse(
                image.GetValueOrDefault("TotalAmount")?.N, out var amount)
                ? amount : 0m,
            UpdatedAt = DateTime.TryParse(
                image.GetValueOrDefault("UpdatedAt")?.S, out var date)
                ? date : DateTime.MinValue
        };
    }
}
```

### Stream Handler

```csharp title="Handlers/OrderStreamHandler.cs"
using Amazon.DynamoDBv2.Model;
using Hardened.Amz.Function.Lambda.Runtime.Attributes;
using Hardened.Amz.Function.DDB.Runtime.Attributes;
using Microsoft.Extensions.Logging;

public class OrderStreamHandler
{
    private readonly IOrderChangeService _changeService;
    private readonly ILogger<OrderStreamHandler> _logger;

    public OrderStreamHandler(
        IOrderChangeService changeService,
        ILogger<OrderStreamHandler> logger)
    {
        _changeService = changeService;
        _logger = logger;
    }

    [HardenedFunction]
    public async Task ProcessRecord(
        [OldImage] Dictionary<string, AttributeValue>? oldImage,
        [NewImage] Dictionary<string, AttributeValue>? newImage)
    {
        var oldRecord = DynamoDbImageHelper.ToOrderRecord(oldImage);
        var newRecord = DynamoDbImageHelper.ToOrderRecord(newImage);

        try
        {
            if (oldRecord == null && newRecord != null)
            {
                // INSERT: new item was added
                await _changeService.HandleOrderCreated(newRecord);
            }
            else if (oldRecord != null && newRecord != null)
            {
                // MODIFY: existing item was updated
                await _changeService.HandleOrderUpdated(oldRecord, newRecord);
            }
            else if (oldRecord != null && newRecord == null)
            {
                // REMOVE: item was deleted
                await _changeService.HandleOrderDeleted(oldRecord);
            }
            else
            {
                _logger.LogWarning(
                    "Stream record with no old or new image -- skipping");
            }
        }
        catch (Exception ex)
        {
            _logger.LogError(ex,
                "Error processing stream record for order {OrderId}",
                newRecord?.OrderId ?? oldRecord?.OrderId ?? "unknown");

            // Re-throw to signal Lambda that processing failed.
            // The record will be retried based on the stream configuration.
            throw;
        }
    }
}
```

### Application Module

```csharp title="Application.cs"
using Hardened.Shared.Runtime.Attributes;

[HardenedModule]
[DynamoStreamLambda.Module]
public partial class Application { }
```

---

## Explanation

### Stream Record Images

DynamoDB Streams provide two images for each record change:

| Image | Attribute | Description |
|---|---|---|
| Old Image | `[OldImage]` | The item **before** the change (null for INSERT events) |
| New Image | `[NewImage]` | The item **after** the change (null for REMOVE events) |

By examining which images are present, you can determine the type of change:

| Old Image | New Image | Event Type |
|---|---|---|
| `null` | present | INSERT |
| present | present | MODIFY |
| present | `null` | REMOVE |

!!! note
    Both `[OldImage]` and `[NewImage]` parameters should be nullable (`Dictionary<string, AttributeValue>?`) since one or the other will be null depending on the event type.

### Error Handling

When a stream record fails processing, you have two options:

1. **Re-throw the exception** -- Lambda retries the batch. Use this when the failure is transient (network issues, throttling).
2. **Catch and log** -- Skip the record and continue. Use this when the failure is permanent (bad data, business rule violation).

!!! warning
    DynamoDB Streams guarantee ordered delivery per partition key. If you re-throw an exception, the entire batch is retried, which blocks processing of subsequent records for that shard. Design your error handling strategy accordingly.

### DynamoDB Table Configuration

For the stream processor to receive events, your DynamoDB table must have streams enabled with the appropriate view type:

| View Type | Old Image | New Image |
|---|---|---|
| `KEYS_ONLY` | Keys only | Keys only |
| `NEW_IMAGE` | Not available | Full item |
| `OLD_IMAGE` | Full item | Not available |
| `NEW_AND_OLD_IMAGES` | Full item | Full item |

For this recipe, use `NEW_AND_OLD_IMAGES` to receive both the before and after state.

---

## Testing

Create a test project:

```bash
cd ..
dotnet new xunit -n OrderStreamProcessor.Tests
cd OrderStreamProcessor.Tests
dotnet add reference ../OrderStreamProcessor/OrderStreamProcessor.csproj
dotnet add package Hardened.Amz.Function.DDB.Testing --prerelease
```

### Bootstrap

```csharp title="Bootstrap.cs"
using Hardened.Shared.Runtime.Attributes;

[assembly: HardenedTestEntryPoint(typeof(Application))]
```

### Tests

```csharp title="OrderStreamHandlerTests.cs"
using Amazon.DynamoDBv2.Model;
using Hardened.Shared.Runtime.Attributes;
using Hardened.Amz.Function.Lambda.Testing;

public class OrderStreamHandlerTests
{
    private static Dictionary<string, AttributeValue> CreateOrderImage(
        string orderId, string customerId, string status, decimal amount)
    {
        return new Dictionary<string, AttributeValue>
        {
            ["OrderId"] = new AttributeValue { S = orderId },
            ["CustomerId"] = new AttributeValue { S = customerId },
            ["Status"] = new AttributeValue { S = status },
            ["TotalAmount"] = new AttributeValue
            {
                N = amount.ToString()
            },
            ["UpdatedAt"] = new AttributeValue
            {
                S = DateTime.UtcNow.ToString("O")
            }
        };
    }

    [HardenedTest]
    public async Task ProcessRecord_Insert_HandlesNewOrder(
        LambdaTestApp testApp)
    {
        var newImage = CreateOrderImage(
            "ORD-001", "CUST-123", "Pending", 99.99m);

        // INSERT event: old image is null, new image is present
        await testApp.InvokeDynamoDbStream(
            oldImage: null,
            newImage: newImage);

        // Verify processing completed without exceptions
    }

    [HardenedTest]
    public async Task ProcessRecord_Modify_HandlesStatusChange(
        LambdaTestApp testApp)
    {
        var oldImage = CreateOrderImage(
            "ORD-001", "CUST-123", "Pending", 99.99m);
        var newImage = CreateOrderImage(
            "ORD-001", "CUST-123", "Shipped", 99.99m);

        // MODIFY event: both images present
        await testApp.InvokeDynamoDbStream(
            oldImage: oldImage,
            newImage: newImage);

        // Verify processing completed without exceptions
    }

    [HardenedTest]
    public async Task ProcessRecord_Remove_HandlesDeletedOrder(
        LambdaTestApp testApp)
    {
        var oldImage = CreateOrderImage(
            "ORD-001", "CUST-123", "Cancelled", 99.99m);

        // REMOVE event: old image is present, new image is null
        await testApp.InvokeDynamoDbStream(
            oldImage: oldImage,
            newImage: null);

        // Verify processing completed without exceptions
    }
}
```

!!! tip
    For more thorough testing, use `[Mock]` to inject a mock `IOrderChangeService` and verify that the correct methods are called with the expected arguments for each event type.

Run the tests:

```bash
dotnet test
```

---

## Next Steps

- [DDB Stream Processing](../aws/lambda/ddb-stream.md) -- advanced stream processing patterns
- [Integration Testing DynamoDB](integration-testing-dynamodb.md) -- test with a real DynamoDB table using Testcontainers
- [Custom Execution Filter](custom-execution-filter.md) -- add retry logic or dead-letter handling
