# Lambda Function Runtime

The Lambda function runtime (`Hardened.Amz.Function.Lambda.Runtime`) enables you to build AWS Lambda functions using the Hardened request pipeline. Handlers are plain C# classes with constructor-injected dependencies, and the source generator produces all bootstrap and serialization code at compile time.

---

## Setup

Add the source generator package to your project:

```bash
dotnet add package Hardened.Amz.Function.Lambda.SourceGenerator --prerelease
```

This pulls in the function runtime as a transitive dependency.

---

## Application Module

Every Lambda function project needs an `Application.cs` that declares the Hardened module:

```csharp title="Application.cs"
using Hardened.Shared.Runtime.Attributes;

[HardenedModule]
public partial class Application { }
```

The class must be `partial`. The source generator emits a `ConfigureModule` method and all DI wiring into the generated partial. For a basic Lambda function, no additional module attribute is required -- the source generator handles the runtime configuration automatically.

---

## Defining Handlers

### Basic Handler

A handler is any class with a method annotated with `[HardenedFunction]`. The method's parameter is deserialized from the Lambda invocation payload, and the return value is serialized back as the response.

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

The source generator:

1. Discovers `[HardenedFunction]` at compile time
2. Generates the Lambda bootstrap entry point
3. Deserializes the incoming event stream into `OrderRequest`
4. Resolves `OrderHandler` from the DI container (injecting `IOrderService`)
5. Calls `Process` and serializes `OrderResponse` back to the response stream

### Named Functions (Multi-Handler Routing)

You can define multiple `[HardenedFunction]` methods across different classes, each with a unique name. The Lambda runtime routes invocations to the correct handler based on the function name.

```csharp title="Handlers/CreateOrderHandler.cs"
public class CreateOrderHandler
{
    private readonly IOrderService _orderService;

    public CreateOrderHandler(IOrderService orderService)
    {
        _orderService = orderService;
    }

    [HardenedFunction("create-order")]
    public async Task<OrderResponse> Create(CreateOrderRequest request)
    {
        return await _orderService.Create(request);
    }
}
```

```csharp title="Handlers/GetOrderHandler.cs"
public class GetOrderHandler
{
    private readonly IOrderService _orderService;

    public GetOrderHandler(IOrderService orderService)
    {
        _orderService = orderService;
    }

    [HardenedFunction("get-order")]
    public async Task<OrderResponse> Get(GetOrderRequest request)
    {
        return await _orderService.Get(request);
    }
}
```

When deploying, the Lambda function name determines which handler is invoked. The runtime matches `ILambdaContext.FunctionName` against the name provided to `[HardenedFunction]`.

!!! note
    Multi-handler routing requires that each `[HardenedFunction]` has a distinct name string. If you only have one handler, the name is optional.

---

## Dependency Injection

Handlers are resolved from the DI container, so all constructor dependencies are injected automatically. Register services using the standard Hardened DI attributes:

```csharp title="Services/OrderService.cs"
using Hardened.Shared.Runtime.Attributes;

[Expose]
public class OrderService : IOrderService
{
    private readonly IDynamoDbClientProvider _dynamoDb;

    public OrderService(IDynamoDbClientProvider dynamoDb)
    {
        _dynamoDb = dynamoDb;
    }

    public async Task<OrderResponse> Create(CreateOrderRequest request)
    {
        var client = _dynamoDb.GetClient();
        // ... DynamoDB operations
        return new OrderResponse { OrderId = request.OrderId, Status = "Created" };
    }
}
```

| Attribute | Lifetime | Description |
|---|---|---|
| `[Expose]` | Transient | New instance per resolution |
| `[Singleton]` | Singleton | Single instance for the Lambda lifetime |
| `[Scoped]` | Scoped | One instance per Lambda invocation |

!!! tip
    Lambda functions maintain a singleton container across warm invocations. Use `[Singleton]` for services that should be reused (such as HTTP clients or DynamoDB clients), and `[Scoped]` for per-invocation state.

---

## Lambda Context Access

You can access the raw `ILambdaContext` in your handler using the `[FromContext]` attribute on a parameter:

```csharp
using Hardened.Amz.Function.Lambda.Runtime;

public class MyHandler
{
    [HardenedFunction]
    public async Task<string> Process(
        MyRequest request,
        [FromContext] string functionName)
    {
        return $"Handled by {functionName}";
    }
}
```

The `[FromContext]` attribute extracts values from the Lambda context. You can optionally specify a property name: `[FromContext("FunctionName")]`.

---

## Method Signatures

The handler method supports several signature patterns:

### Request and Response

```csharp
[HardenedFunction]
public async Task<OrderResponse> Process(OrderRequest request)
```

### Request Only (Fire and Forget)

```csharp
[HardenedFunction]
public async Task Process(OrderRequest request)
```

### No Parameters

```csharp
[HardenedFunction]
public async Task<HealthCheckResponse> Check()
```

---

## Execution Pipeline

Lambda functions run through the Hardened execution pipeline, which means you can apply `IExecutionFilter` implementations for cross-cutting concerns:

```csharp
using Hardened.Requests.Abstract.Execution;
using Hardened.Shared.Runtime.Attributes;

[Expose]
[Singleton]
public class LoggingFilter : IExecutionFilter
{
    private readonly ILogger<LoggingFilter> _logger;

    public LoggingFilter(ILogger<LoggingFilter> logger)
    {
        _logger = logger;
    }

    public async Task Execute(IExecutionChain chain)
    {
        _logger.LogInformation("Lambda invocation started: {Path}", chain.Context.Request.Path);

        await chain.Next();

        _logger.LogInformation("Lambda invocation completed: {Status}", chain.Context.Response.Status);
    }
}
```

Filters execute in order around the handler, just like in the web runtime.

---

## Error Handling

By default, exceptions thrown from a handler propagate to the Lambda runtime, which marks the invocation as failed. You can customize error handling with an execution filter or by applying the `[ThrowException]` attribute to control exception behavior.

For batch-processing runtimes (SQS, DDB Streams), see the dedicated documentation for partial failure handling.

---

## Project Structure

A typical Lambda function project looks like this:

```
OrderProcessor/
  Application.cs
  Models/
    OrderRequest.cs
    OrderResponse.cs
  Handlers/
    OrderHandler.cs
  Services/
    IOrderService.cs
    OrderService.cs
OrderProcessor.Tests/
  Bootstrap.cs
  OrderHandlerTests.cs
```

---

## Next Steps

- [Lambda Testing](testing.md) -- test functions with `LambdaTestApp`
- [Web Runtime](web-runtime.md) -- deploy web APIs behind API Gateway
- [DDB Stream Processing](ddb-stream.md) -- process DynamoDB Streams events
- [SQS Processing](sqs-processing.md) -- consume SQS batches
