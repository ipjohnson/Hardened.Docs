# Lambda Testing

Hardened provides dedicated testing utilities for each Lambda runtime type. These test helpers simulate the Lambda execution environment locally, using the same DI container and request pipeline as production, so your tests exercise the full handler chain without deploying to AWS.

---

## Packages

| Lambda type | Testing package | Test helper class |
|---|---|---|
| Function (request/response) | `Hardened.Amz.Function.Lambda.Testing` | `LambdaTestApp` |
| SQS batch processor | `Hardened.Amz.Function.Sqs.Testing` | `TestSqsApp` |
| DynamoDB Streams | `Hardened.Amz.Function.DDB.Testing` | `TestDynamoDbStream` |
| Web (API Gateway) | `Hardened.Web.Testing` | `ITestWebApp` |

---

## Test Project Setup

### 1. Create the test project

```bash
dotnet new xunit -n MyLambda.Tests
cd MyLambda.Tests
dotnet add reference ../MyLambda/MyLambda.csproj
```

### 2. Add the testing package

=== "Function Testing"

    ```bash
    dotnet add package Hardened.Amz.Function.Lambda.Testing --prerelease
    ```

=== "SQS Testing"

    ```bash
    dotnet add package Hardened.Amz.Function.Sqs.Testing --prerelease
    ```

=== "DDB Stream Testing"

    ```bash
    dotnet add package Hardened.Amz.Function.DDB.Testing --prerelease
    ```

=== "Web Lambda Testing"

    ```bash
    dotnet add package Hardened.Web.Testing --prerelease
    ```

### 3. Create the Bootstrap file

The bootstrap file tells the test framework which `Application` module to use as the composition root:

```csharp title="Bootstrap.cs"
using Hardened.Shared.Testing.Attributes;

[assembly: HardenedTestEntryPoint(typeof(Application))]
```

!!! note
    For web Lambda testing, also add the `[assembly: WebTesting]` attribute from `Hardened.Web.Testing`.

---

## LambdaTestApp (Function Runtime)

`LambdaTestApp` simulates Lambda function invocations. It serializes your payload, invokes the handler through the full execution pipeline, and deserializes the response.

### Basic Usage

```csharp
using Hardened.Shared.Testing.Attributes;
using Hardened.Amz.Function.Lambda.Testing;

public class OrderHandlerTests
{
    [HardenedTest]
    public async Task ProcessOrder_ReturnsProcessedStatus(LambdaTestApp testApp)
    {
        var request = new OrderRequest
        {
            OrderId = "ORD-001",
            CustomerId = "CUST-123",
            Amount = 99.99m
        };

        var response = await testApp.Invoke<OrderResponse>(
            "process-order",
            request);

        Assert.Equal("ORD-001", response.OrderId);
        Assert.Equal("Processed", response.Status);
    }
}
```

### API Reference

`LambdaTestApp` provides two `Invoke` overloads:

```csharp
// Invoke and deserialize the response to T
Task<T> Invoke<T>(string lambdaName, object payload, Action<TestLambdaContext>? contextAction = null)

// Invoke and return the raw response stream
Task<Stream> Invoke(string lambdaName, object payload, Action<TestLambdaContext>? contextAction = null)
```

| Parameter | Description |
|---|---|
| `lambdaName` | The function name to route to (must match the `[HardenedFunction("name")]` value) |
| `payload` | The request object, serialized to JSON as the Lambda invocation payload |
| `contextAction` | Optional callback to customize the `TestLambdaContext` before invocation |

### Testing Named Functions

When you have multiple `[HardenedFunction]` handlers, pass the function name to route to the correct one:

```csharp
[HardenedTest]
public async Task CreateOrder_Succeeds(LambdaTestApp testApp)
{
    var response = await testApp.Invoke<OrderResponse>(
        "create-order",
        new CreateOrderRequest { CustomerId = "CUST-123" });

    Assert.Equal("Created", response.Status);
}

[HardenedTest]
public async Task GetOrder_ReturnsOrder(LambdaTestApp testApp)
{
    var response = await testApp.Invoke<OrderResponse>(
        "get-order",
        new GetOrderRequest { OrderId = "ORD-001" });

    Assert.Equal("ORD-001", response.OrderId);
}
```

### Customizing Lambda Context

Use the `contextAction` parameter to set custom context values:

```csharp
[HardenedTest]
public async Task Process_WithCustomContext(LambdaTestApp testApp)
{
    var response = await testApp.Invoke<MyResponse>(
        "my-function",
        new MyRequest(),
        context =>
        {
            context.FunctionName = "custom-name";
            context.RemainingTime = TimeSpan.FromSeconds(30);
        });
}
```

### Raw Stream Response

For handlers that return non-JSON data or when you need to inspect the raw output:

```csharp
[HardenedTest]
public async Task Process_RawStream(LambdaTestApp testApp)
{
    await using var stream = await testApp.Invoke("my-function", new MyRequest());

    using var reader = new StreamReader(stream);
    var rawResponse = await reader.ReadToEndAsync();

    Assert.Contains("expected-content", rawResponse);
}
```

---

## TestSqsApp (SQS Runtime)

`TestSqsApp` simulates SQS batch invocations. It wraps your messages in an `SQSEvent`, invokes the handler, and returns the `SQSBatchResponse` so you can verify partial failure handling.

### Basic Usage

```csharp
using Hardened.Shared.Testing.Attributes;
using Hardened.Amz.Function.Sqs.Testing;

public class NotificationHandlerTests
{
    [HardenedTest]
    public async Task Process_SingleMessage_Succeeds(TestSqsApp testApp)
    {
        var response = await testApp.SendMessage(
            new NotificationMessage
            {
                UserId = "user-1",
                Subject = "Welcome",
                Body = "Hello!"
            });

        Assert.Empty(response.BatchItemFailures);
    }
}
```

### API Reference

```csharp
Task<SQSBatchResponse> SendMessage<T>(params T[] messages)
```

| Parameter | Description |
|---|---|
| `messages` | One or more message objects. Each is serialized to JSON and wrapped as an `SQSEvent.SQSMessage` body. |

The method returns an `SQSBatchResponse`. Check `BatchItemFailures` to verify which messages failed.

### Testing Multiple Messages

```csharp
[HardenedTest]
public async Task Process_BatchOfMessages(TestSqsApp testApp)
{
    var response = await testApp.SendMessage(
        new OrderMessage { OrderId = "ORD-001", Amount = 10.00m },
        new OrderMessage { OrderId = "ORD-002", Amount = 25.50m },
        new OrderMessage { OrderId = "ORD-003", Amount = 5.99m });

    Assert.Empty(response.BatchItemFailures);
}
```

### Testing Partial Failures

If your handler throws for certain messages, the batch response will contain the failed message IDs:

```csharp
[HardenedTest]
public async Task Process_PartialFailure_ReportsFailedMessages(TestSqsApp testApp)
{
    var response = await testApp.SendMessage(
        new OrderMessage { OrderId = "valid-order", Amount = 10.00m },
        new OrderMessage { OrderId = "invalid-order", Amount = -1m });

    // One message should have failed
    Assert.Single(response.BatchItemFailures);
}
```

---

## TestDynamoDbStream (DDB Stream Runtime)

`TestDynamoDbStream` simulates DynamoDB Streams batch events. You construct `DynamodbStreamRecord` objects and pass them to `ProcessUpdates`.

### Basic Usage

```csharp
using Amazon.Lambda.DynamoDBEvents;
using Hardened.Shared.Testing.Attributes;
using Hardened.Amz.Function.DDB.Testing;

public class StreamHandlerTests
{
    [HardenedTest]
    public async Task ProcessRecord_HandlesInsert(TestDynamoDbStream stream)
    {
        var response = await stream.ProcessUpdates(new DynamoDBEvent.DynamodbStreamRecord
        {
            Dynamodb = new DynamoDBEvent.StreamRecord
            {
                NewImage = new Dictionary<string, DynamoDBEvent.AttributeValue>
                {
                    ["PK"] = new() { S = "USER#123" },
                    ["Name"] = new() { S = "Alice" }
                },
                OldImage = new Dictionary<string, DynamoDBEvent.AttributeValue>()
            }
        });

        Assert.Empty(response.BatchItemFailures);
    }
}
```

### API Reference

```csharp
Task<StreamsEventResponse> ProcessUpdates(params DynamoDBEvent.DynamodbStreamRecord[] records)
```

| Parameter | Description |
|---|---|
| `records` | One or more `DynamodbStreamRecord` objects, each containing a `StreamRecord` with `OldImage` and/or `NewImage` data. |

### Testing Modifications

```csharp
[HardenedTest]
public async Task ProcessRecord_HandlesModify(TestDynamoDbStream stream)
{
    var response = await stream.ProcessUpdates(new DynamoDBEvent.DynamodbStreamRecord
    {
        Dynamodb = new DynamoDBEvent.StreamRecord
        {
            OldImage = new Dictionary<string, DynamoDBEvent.AttributeValue>
            {
                ["PK"] = new() { S = "USER#123" },
                ["Status"] = new() { S = "Active" }
            },
            NewImage = new Dictionary<string, DynamoDBEvent.AttributeValue>
            {
                ["PK"] = new() { S = "USER#123" },
                ["Status"] = new() { S = "Inactive" }
            }
        }
    });

    Assert.Empty(response.BatchItemFailures);
}
```

### Testing Multiple Records

```csharp
[HardenedTest]
public async Task ProcessRecord_HandlesBatch(
    TestDynamoDbStream stream,
    CountingService countingService)
{
    var response = await stream.ProcessUpdates(
        new DynamoDBEvent.DynamodbStreamRecord
        {
            Dynamodb = new DynamoDBEvent.StreamRecord
            {
                NewImage = new Dictionary<string, DynamoDBEvent.AttributeValue>
                {
                    ["PK"] = new() { S = "ITEM#1" }
                },
                OldImage = new Dictionary<string, DynamoDBEvent.AttributeValue>()
            }
        },
        new DynamoDBEvent.DynamodbStreamRecord
        {
            Dynamodb = new DynamoDBEvent.StreamRecord
            {
                NewImage = new Dictionary<string, DynamoDBEvent.AttributeValue>
                {
                    ["PK"] = new() { S = "ITEM#2" }
                },
                OldImage = new Dictionary<string, DynamoDBEvent.AttributeValue>()
            }
        });

    Assert.Equal(2, countingService.Count);
    Assert.Empty(response.BatchItemFailures);
}
```

---

## Injecting Additional Services

Because `[HardenedTest]` supports DI, you can inject any registered service alongside the test helper:

```csharp
[HardenedTest]
public async Task Process_VerifySideEffect(
    LambdaTestApp testApp,
    IOrderRepository orderRepository)
{
    await testApp.Invoke<OrderResponse>("create-order", new CreateOrderRequest
    {
        OrderId = "ORD-001",
        Amount = 50.00m
    });

    var savedOrder = await orderRepository.Get("ORD-001");
    Assert.NotNull(savedOrder);
    Assert.Equal(50.00m, savedOrder.Amount);
}
```

---

## Mocking Dependencies

Use the `[Mock]` attribute to replace real services with NSubstitute mocks:

```csharp
using NSubstitute;
using Hardened.Shared.Testing.Attributes;
using Hardened.Amz.Function.Lambda.Testing;

public class OrderHandlerTests
{
    [HardenedTest]
    public async Task Process_CallsService(
        LambdaTestApp testApp,
        [Mock] IOrderService mockOrderService)
    {
        mockOrderService.ProcessOrder(Arg.Any<OrderRequest>())
            .Returns(new OrderResponse { Status = "Mocked" });

        var response = await testApp.Invoke<OrderResponse>(
            "process-order",
            new OrderRequest { OrderId = "ORD-001" });

        Assert.Equal("Mocked", response.Status);
        await mockOrderService.Received(1).ProcessOrder(Arg.Any<OrderRequest>());
    }
}
```

---

## Web Lambda Testing

Lambda web applications (API Gateway) are tested using `ITestWebApp` from `Hardened.Web.Testing`, exactly like ASP.NET Core applications:

```csharp title="Bootstrap.cs"
using Hardened.Shared.Testing.Attributes;
using Hardened.Web.Testing;

[assembly: WebTesting]
[assembly: HardenedTestEntryPoint(typeof(Application))]
```

```csharp
using Hardened.Shared.Testing.Attributes;
using Hardened.Web.Testing;

public class BookControllerTests
{
    [HardenedTest]
    public async Task GetBook_ReturnsOk(ITestWebApp testWebApp)
    {
        var response = await testWebApp.Get("/api/books/tolkien/hobbit");

        response.Assert.Ok();

        var book = response.Deserialize<BookResponse>();
        Assert.Equal("tolkien", book.Author);
    }
}
```

See [Web Testing](../../framework/testing/web-testing.md) for the full `ITestWebApp` API.

---

## Next Steps

- [Function Runtime](function-runtime.md) -- handler patterns and DI
- [DynamoDB Testing](../clients/dynamodb-testing.md) -- integration testing with `[LocalDynamoDb]`
- [Mocking](../../framework/testing/mocking.md) -- deep dive into `[Mock]` and NSubstitute patterns
- [HardenedTest](../../framework/testing/hardened-test.md) -- test framework fundamentals
