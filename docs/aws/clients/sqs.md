# SQS Client

The `Hardened.Amz.SqsClient` package provides `ISqsClient` -- a simple interface for sending messages to SQS queues. Messages are serialized to JSON and routed to the appropriate queue based on the `[SqsMessage]` attribute on your message type.

---

## Setup

```bash
dotnet add package Hardened.Amz.SqsClient --prerelease
```

The package registers `ISqsClient` in the DI container automatically via the Hardened source generator.

---

## ISqsClient

The core interface for sending SQS messages:

```csharp
public interface ISqsClient
{
    Task SendMessage<TMessage>(TMessage message);
}
```

### Basic Usage

Inject `ISqsClient` into your services and call `SendMessage<T>()`:

```csharp
using Hardened.SqsClient;
using DependencyModules.Runtime.Attributes;

[TransientService]
public class OrderService : IOrderService
{
    private readonly ISqsClient _sqsClient;

    public OrderService(ISqsClient sqsClient)
    {
        _sqsClient = sqsClient;
    }

    public async Task SubmitOrder(Order order)
    {
        // Validate and process the order...

        // Send a message to SQS for async processing
        await _sqsClient.SendMessage(new OrderSubmittedMessage
        {
            OrderId = order.OrderId,
            CustomerId = order.CustomerId,
            Amount = order.Amount,
            SubmittedAt = DateTime.UtcNow
        });
    }
}
```

---

## [SqsMessage] Attribute

The `[SqsMessage]` attribute associates a message type with a specific SQS queue name:

```csharp
using Hardened.SqsClient;

[SqsMessage("order-processing-queue")]
public class OrderSubmittedMessage
{
    public string OrderId { get; set; } = string.Empty;
    public string CustomerId { get; set; } = string.Empty;
    public decimal Amount { get; set; }
    public DateTime SubmittedAt { get; set; }
}
```

When `SendMessage<OrderSubmittedMessage>()` is called, the client:

1. Serializes the message object to JSON
2. Resolves the queue name from the `[SqsMessage]` attribute
3. Sends the message to the specified SQS queue

---

## Configuration

The `SqsClientConfiguration` is a `[ConfigurationModel]` that allows you to customize the SQS client behavior:

```csharp
[ConfigurationModel]
public partial class SqsClientConfiguration { }
```

Configure the client in your module:

```csharp
using Hardened.Shared.Runtime.Attributes;
using Microsoft.Extensions.DependencyInjection;

[HardenedModule]
public partial class Application
{
    public void ConfigureServices(IServiceCollection services)
    {
        services.Configure<ISqsClientConfiguration>(config =>
        {
            // Configure SQS client options
        });
    }
}
```

---

## Message Types

Define message classes as simple POCOs with the `[SqsMessage]` attribute:

```csharp
[SqsMessage("notification-queue")]
public class SendNotificationMessage
{
    public string UserId { get; set; } = string.Empty;
    public string Subject { get; set; } = string.Empty;
    public string Body { get; set; } = string.Empty;
    public string Channel { get; set; } = string.Empty;
}

[SqsMessage("audit-queue")]
public class AuditEventMessage
{
    public string EventType { get; set; } = string.Empty;
    public string EntityId { get; set; } = string.Empty;
    public string PerformedBy { get; set; } = string.Empty;
    public DateTime Timestamp { get; set; }
    public Dictionary<string, string> Metadata { get; set; } = new();
}
```

Each message type can target a different queue. The queue name in `[SqsMessage]` should match the SQS queue name (or URL, depending on your configuration).

---

## Using with Lambda SQS Processing

The SQS client pairs naturally with the [SQS Lambda runtime](../lambda/sqs-processing.md). You can send messages from one Lambda (or web API) and process them in an SQS Lambda handler:

### Producer (Web API or Lambda)

```csharp
[Post("/api/orders")]
public async Task<OrderResponse> CreateOrder(
    CreateOrderRequest request,
    ISqsClient sqsClient)
{
    var order = new Order { OrderId = Guid.NewGuid().ToString(), /* ... */ };

    // Send for async processing
    await sqsClient.SendMessage(new ProcessOrderMessage
    {
        OrderId = order.OrderId,
        Amount = request.Amount
    });

    return new OrderResponse { OrderId = order.OrderId, Status = "Queued" };
}
```

### Consumer (SQS Lambda)

```csharp
[HardenedModule]
[SqsLambda.Module]
public partial class Application { }

public class OrderProcessor
{
    [HardenedFunction]
    public async Task Process(ProcessOrderMessage message)
    {
        // Process the order asynchronously
    }
}
```

---

## Complete Example

```csharp title="Messages/OrderSubmittedMessage.cs"
using Hardened.SqsClient;

[SqsMessage("order-processing-queue")]
public class OrderSubmittedMessage
{
    public string OrderId { get; set; } = string.Empty;
    public string CustomerId { get; set; } = string.Empty;
    public decimal Amount { get; set; }
    public DateTime SubmittedAt { get; set; }
}
```

```csharp title="Services/OrderService.cs"
using Hardened.SqsClient;
using DependencyModules.Runtime.Attributes;

[TransientService]
public class OrderService : IOrderService
{
    private readonly ISqsClient _sqsClient;
    private readonly ILogger<OrderService> _logger;

    public OrderService(ISqsClient sqsClient, ILogger<OrderService> logger)
    {
        _sqsClient = sqsClient;
        _logger = logger;
    }

    public async Task SubmitOrder(string customerId, decimal amount)
    {
        var orderId = Guid.NewGuid().ToString();

        await _sqsClient.SendMessage(new OrderSubmittedMessage
        {
            OrderId = orderId,
            CustomerId = customerId,
            Amount = amount,
            SubmittedAt = DateTime.UtcNow
        });

        _logger.LogInformation("Order {OrderId} submitted for processing", orderId);
    }
}
```

```csharp title="Controllers/OrderController.cs"
using Hardened.Web.Runtime.Attributes;

[BasePath("/api/orders")]
public class OrderController
{
    private readonly IOrderService _orderService;

    public OrderController(IOrderService orderService)
    {
        _orderService = orderService;
    }

    [Post("/")]
    public async Task<object> SubmitOrder(SubmitOrderRequest request)
    {
        await _orderService.SubmitOrder(request.CustomerId, request.Amount);
        return new { Status = "Queued" };
    }
}
```

---

## Next Steps

- [SQS Processing](../lambda/sqs-processing.md) -- consume SQS messages in a Lambda function
- [SQS Testing](../lambda/testing.md) -- test SQS handlers with `TestSqsApp`
- [DynamoDB Client](dynamodb.md) -- access DynamoDB with `IDynamoDbClientProvider`
