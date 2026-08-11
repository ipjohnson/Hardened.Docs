# Interfaces Reference

This page is a comprehensive reference for the key interfaces in the Hardened framework. Each entry includes the full signature, package, description, and usage guidance.

---

## Shared Layer (Hardened.Shared.Runtime)

### IApplicationModule

The module contract that every `[HardenedModule]` partial class implements. The source generator emits the `ConfigureModule` method, which registers all discovered services, routes, and configuration for the module.

**Package:** `Hardened.Shared.Runtime`
**Namespace:** `Hardened.Shared.Runtime.Application`

```csharp
public interface IApplicationModule {
    void ConfigureModule(IHardenedEnvironment environment, IServiceCollection serviceCollection);
}
```

| Member | Return Type | Description |
|---|---|---|
| `ConfigureModule(IHardenedEnvironment, IServiceCollection)` | `void` | Register all services, routes, and configuration for this module. |

**Usage:** You rarely implement this interface directly. Instead, apply `[HardenedModule]` to a partial class and the source generator produces the implementation.

```csharp
[HardenedModule]
public partial class Application { }

// Later, at startup:
var app = new Application();
var env = new EnvironmentImpl(arguments: args);
app.ConfigureModule(env, services);
```

---

### IApplicationRoot

The root of an initialized Hardened application. Provides access to the root `IServiceProvider` and supports async disposal for cleanup.

**Package:** `Hardened.Shared.Runtime`
**Namespace:** `Hardened.Shared.Runtime.Application`

```csharp
public interface IApplicationRoot : IAsyncDisposable {
    IServiceProvider Provider { get; }
}
```

| Member | Return Type | Description |
|---|---|---|
| `Provider` | `IServiceProvider` | The root service provider for the application. |
| `DisposeAsync()` | `ValueTask` | Dispose the application and all registered services (inherited from `IAsyncDisposable`). |

---

### IHardenedEnvironment

Provides environment information including the environment name, command-line arguments, and key-value configuration. Used throughout the framework for environment-aware behavior.

**Package:** `Hardened.Shared.Runtime`
**Namespace:** `Hardened.Shared.Runtime.Application`

```csharp
public interface IHardenedEnvironment {
    string Name { get; }
    IReadOnlyList<string> Arguments { get; }
    T? Value<T>(string name, T? defaultValue = default);
    T? CustomData<T>(string name, T? defaultValue = default);
}
```

| Member | Return Type | Description |
|---|---|---|
| `Name` | `string` | The environment name (e.g., "Development", "Production"). |
| `Arguments` | `IReadOnlyList<string>` | Command-line arguments passed at startup. |
| `Value<T>(string, T?)` | `T?` | Retrieve a typed environment value by name, with an optional default. |
| `CustomData<T>(string, T?)` | `T?` | Retrieve custom data attached to the environment by name. |

**Extension methods** (from `IEnvironmentExtensions`):

| Method | Return Type | Description |
|---|---|---|
| `Matches(params string[])` | `bool` | Check if the environment name matches any of the given values (case-insensitive). |
| `MatchesVariable(string, string)` | `bool` | Check if an environment variable matches a specific value. |

```csharp
// In a configuration callback
appConfig.Amend<DatabaseConfig>((env, config) => {
    if (env.Matches("Production", "Staging")) {
        config.MaxPoolSize = 50;
    }
    return config;
});
```

---

### IStartupService

An async startup hook that runs after the DI container is built but before the application begins accepting requests. Use this for initialization logic such as database migrations, cache warming, or health checks.

**Package:** `Hardened.Shared.Runtime`
**Namespace:** `Hardened.Shared.Runtime.Application`

```csharp
public interface IStartupService {
    Task<bool> Startup(IServiceProvider rootProvider);
}
```

| Member | Return Type | Description |
|---|---|---|
| `Startup(IServiceProvider)` | `Task<bool>` | Perform startup logic. Return `true` to indicate success, `false` to signal startup failure. |

```csharp
[SingletonService(As = typeof(IStartupService))]
public class CacheWarmer : IStartupService {
    public async Task<bool> Startup(IServiceProvider rootProvider) {
        var cache = rootProvider.GetRequiredService<ICacheService>();
        await cache.WarmUp();
        return true;
    }
}
```

---

### IAppConfig

Fluent API for providing and modifying configuration values at startup. Typically used inside a `[HardenedModule]` partial class's static `ConfigureApplication` method.

**Package:** `Hardened.Shared.Runtime`
**Namespace:** `Hardened.Shared.Runtime.Configuration`

```csharp
public interface IAppConfig {
    IAppConfig ProvideValue<TInterface, TImpl>(
        Func<IHardenedEnvironment, TImpl> valueProvider)
        where TImpl : class, TInterface;

    IAppConfig Amend<TImpl>(
        Action<TImpl> amendAction, string environment = "")
        where TImpl : class;

    IAppConfig Amend<TImpl>(
        Func<IHardenedEnvironment, TImpl, TImpl> amendFunc)
        where TImpl : class;
}
```

| Member | Return Type | Description |
|---|---|---|
| `ProvideValue<TInterface, TImpl>(Func<IHardenedEnvironment, TImpl>)` | `IAppConfig` | Supply a complete configuration value for the given interface. |
| `Amend<TImpl>(Action<TImpl>, string)` | `IAppConfig` | Modify an existing configuration. Optionally restrict to a specific environment. |
| `Amend<TImpl>(Func<IHardenedEnvironment, TImpl, TImpl>)` | `IAppConfig` | Modify configuration with access to the environment. |

!!! info "See Also"
    For full configuration system documentation, see the [Configuration Reference](configuration.md).

```csharp
public partial class Application {
    private static void ConfigureApplication(IAppConfig appConfig) {
        appConfig
            .Amend<DatabaseConfig>(config => {
                config.MaxPoolSize = 10;
                config.CommandTimeoutSeconds = 30;
            })
            .Amend<DatabaseConfig>(config => {
                config.MaxPoolSize = 100;
            }, environment: "Production");
    }
}
```

---

### IConfigurationPackage

Bundle configuration providers and amenders into a reusable package. This is the preferred way to organize configuration logic in library modules.

**Package:** `Hardened.Shared.Runtime`
**Namespace:** `Hardened.Shared.Runtime.Configuration`

```csharp
public interface IConfigurationPackage {
    IEnumerable<IConfigurationValueProvider> ConfigurationValueProviders(
        IHardenedEnvironment env);
    IEnumerable<IConfigurationValueAmender> ConfigurationValueAmenders(
        IHardenedEnvironment env);
}
```

| Member | Return Type | Description |
|---|---|---|
| `ConfigurationValueProviders(IHardenedEnvironment)` | `IEnumerable<IConfigurationValueProvider>` | Return providers that supply initial configuration values. |
| `ConfigurationValueAmenders(IHardenedEnvironment)` | `IEnumerable<IConfigurationValueAmender>` | Return amenders that modify configuration values. |

```csharp
[SingletonService(As = typeof(IConfigurationPackage))]
public class MyLibraryConfigPackage : IConfigurationPackage {
    public IEnumerable<IConfigurationValueProvider> ConfigurationValueProviders(
        IHardenedEnvironment env) {
        yield break;
    }

    public IEnumerable<IConfigurationValueAmender> ConfigurationValueAmenders(
        IHardenedEnvironment env) {
        yield break;
    }
}
```

---

### IConfigurationValueProvider

Provides the initial value for a configuration model type. Implementations are returned from `IConfigurationPackage.ConfigurationValueProviders()`.

**Package:** `Hardened.Shared.Runtime`
**Namespace:** `Hardened.Shared.Runtime.Configuration`

```csharp
public interface IConfigurationValueProvider {
    Type InterfaceType { get; }
    Type ImplementationType { get; }
    object ProvideValue(IHardenedEnvironment environment,
        Action<IHardenedEnvironment, object> amender);
}
```

| Member | Return Type | Description |
|---|---|---|
| `InterfaceType` | `Type` | The configuration interface type. |
| `ImplementationType` | `Type` | The concrete implementation type. |
| `ProvideValue(IHardenedEnvironment, Action<IHardenedEnvironment, object>)` | `object` | Create and return the configuration instance. |

---

### IConfigurationValueAmender

Modifies an existing configuration value. Implementations are returned from `IConfigurationPackage.ConfigurationValueAmenders()`.

**Package:** `Hardened.Shared.Runtime`
**Namespace:** `Hardened.Shared.Runtime.Configuration`

```csharp
public interface IConfigurationValueAmender {
    object ApplyConfiguration(IHardenedEnvironment environment, object configurationValue);
}
```

| Member | Return Type | Description |
|---|---|---|
| `ApplyConfiguration(IHardenedEnvironment, object)` | `object` | Receive the current configuration value and return the modified version. |

---

## Requests Layer (Hardened.Requests.Abstract)

### IExecutionContext

The central object for a single request execution. Holds the request, response, DI scope, handler instance, metrics, and cancellation token.

**Package:** `Hardened.Requests.Abstract`
**Namespace:** `Hardened.Requests.Abstract.Execution`

```csharp
public interface IExecutionContext {
    IExecutionContext Clone(
        IExecutionRequest? request = null,
        IExecutionResponse? response = null,
        IServiceProvider? serviceProvider = null,
        IMetricLogger? metricLogger = null);

    IServiceProvider RootServiceProvider { get; }
    IKnownServices KnownServices { get; }
    IServiceProvider RequestServices { get; }
    IExecutionRequest Request { get; }
    IExecutionResponse Response { get; }
    object? HandlerInstance { get; set; }
    IExecutionRequestHandlerInfo? HandlerInfo { get; set; }
    DefaultOutputFunc? DefaultOutput { get; set; }
    IMetricLogger RequestMetrics { get; }
    MachineTimestamp StartTime { get; }
    CancellationToken CancellationToken { get; }
}
```

| Member | Type | Description |
|---|---|---|
| `RootServiceProvider` | `IServiceProvider` | The root (application-scoped) service provider. |
| `KnownServices` | `IKnownServices` | Pre-resolved common services (serialization, string conversion). |
| `RequestServices` | `IServiceProvider` | The request-scoped service provider (created per request). |
| `Request` | `IExecutionRequest` | The inbound request data. |
| `Response` | `IExecutionResponse` | The outbound response data. |
| `HandlerInstance` | `object?` | The instantiated handler class for this request. |
| `HandlerInfo` | `IExecutionRequestHandlerInfo?` | Metadata about the matched handler (path, method, parameters). |
| `DefaultOutput` | `DefaultOutputFunc?` | Default output function (used for template rendering). |
| `RequestMetrics` | `IMetricLogger` | Metric logger scoped to this request. |
| `StartTime` | `MachineTimestamp` | Timestamp when the request started. |
| `CancellationToken` | `CancellationToken` | Cancellation token for cooperative cancellation. |
| `Clone(...)` | `IExecutionContext` | Create a copy with optional overrides. |

---

### IExecutionRequest

Represents the inbound request data -- method, path, body, headers, query string, path tokens, and cookies.

**Package:** `Hardened.Requests.Abstract`
**Namespace:** `Hardened.Requests.Abstract.Execution`

```csharp
public interface IExecutionRequest {
    IExecutionRequest Clone(
        string? method = null,
        string? path = null,
        IDictionary<string, StringValues>? headers = null,
        IQueryStringCollection? queryString = null,
        IReadOnlyList<string>? cookies = null);

    string Method { get; }
    string Path { get; }
    string? ContentType { get; }
    string? Accept { get; }
    IExecutionRequestParameters? Parameters { get; set; }
    Stream Body { get; set; }
    IDictionary<string, StringValues> Headers { get; }
    IQueryStringCollection QueryString { get; }
    IPathTokenCollection PathTokens { get; set; }
    IReadOnlyList<string> Cookies { get; }
}
```

| Member | Type | Description |
|---|---|---|
| `Method` | `string` | HTTP method or request method (e.g., "GET", "POST"). |
| `Path` | `string` | The request path. |
| `ContentType` | `string?` | The Content-Type header value. |
| `Accept` | `string?` | The Accept header value. |
| `Parameters` | `IExecutionRequestParameters?` | Bound method parameters (populated during parameter binding). |
| `Body` | `Stream` | The request body stream. |
| `Headers` | `IDictionary<string, StringValues>` | Request headers. |
| `QueryString` | `IQueryStringCollection` | Parsed query string parameters. |
| `PathTokens` | `IPathTokenCollection` | Parsed path tokens (from route parameters like `{id}`). |
| `Cookies` | `IReadOnlyList<string>` | Request cookies. |

---

### IExecutionResponse

Represents the outbound response -- status code, body, headers, content type, cookies, and error information.

**Package:** `Hardened.Requests.Abstract`
**Namespace:** `Hardened.Requests.Abstract.Execution`

```csharp
public interface IExecutionResponse {
    IExecutionResponse Clone(IHeaderCollection? headerCollection = null);

    string? ContentType { get; set; }
    object? ResponseValue { get; set; }
    string? TemplateName { get; set; }
    int? Status { get; set; }
    bool ShouldCompress { get; set; }
    Stream Body { get; set; }
    IDictionary<string, StringValues> Headers { get; }
    Exception? ExceptionValue { get; set; }
    bool ResponseStarted { get; }
    bool IsBinary { get; set; }
    ICookieSetCollection Cookies { get; }
    bool ShouldSerialize { get; set; }
}
```

| Member | Type | Description |
|---|---|---|
| `ContentType` | `string?` | Response content type. |
| `ResponseValue` | `object?` | The handler's return value (before serialization). |
| `TemplateName` | `string?` | Template name for rendering (set by `[Template]`). |
| `Status` | `int?` | HTTP status code. |
| `ShouldCompress` | `bool` | Whether to apply compression. |
| `Body` | `Stream` | The response body stream. |
| `Headers` | `IDictionary<string, StringValues>` | Response headers. |
| `ExceptionValue` | `Exception?` | Any unhandled exception from the handler. |
| `ResponseStarted` | `bool` | Whether the response has already begun writing. |
| `IsBinary` | `bool` | Whether the response body is binary data. |
| `Cookies` | `ICookieSetCollection` | Response cookies. |
| `ShouldSerialize` | `bool` | Whether the response value should be serialized. |

---

### IExecutionFilter

A filter in the execution pipeline. Filters wrap handler execution and provide cross-cutting concerns like logging, metrics, serialization, authentication, and error handling.

**Package:** `Hardened.Requests.Abstract`
**Namespace:** `Hardened.Requests.Abstract.Execution`

```csharp
public interface IExecutionFilter {
    Task Execute(IExecutionChain chain);
}
```

| Member | Return Type | Description |
|---|---|---|
| `Execute(IExecutionChain)` | `Task` | Execute this filter's logic, typically calling `chain.Next()` to continue the pipeline. |

**ExecutionFilterOrder** controls the position of a filter in the pipeline:

| Order | Value | Description |
|---|---|---|
| `Init` | `-10000` | Initialization filters. |
| `FullRequestMetrics` | `-7000` | Full request metrics collection. |
| `RetryFilter` | `-5000` | Retry logic. |
| `BeforeSerialize` | `-1` | Before parameter serialization. |
| `BindParameters` | `0` | Parameter binding. |
| `First` | `1` | First user filter slot. |
| `Second` | `2` | Second user filter slot. |
| `Third` | `3` | Third user filter slot. |
| `Normal` | `100` | Default user filter slot. |
| `Last` | `int.MaxValue` | Last filter in the chain. |

```csharp
[SingletonService(As = typeof(IExecutionFilter))]
public class RequestLoggingFilter : IExecutionFilter {
    private readonly ILogger _logger;

    public RequestLoggingFilter(ILogger<RequestLoggingFilter> logger) {
        _logger = logger;
    }

    public async Task Execute(IExecutionChain chain) {
        _logger.LogInformation("Request: {Method} {Path}",
            chain.Context.Request.Method,
            chain.Context.Request.Path);

        await chain.Next();

        _logger.LogInformation("Response: {Status}",
            chain.Context.Response.Status);
    }
}
```

---

### IExecutionChain

The mechanism for progressing through the filter chain. Each filter calls `Next()` to invoke the next filter (or the handler, if it is the last filter).

**Package:** `Hardened.Requests.Abstract`
**Namespace:** `Hardened.Requests.Abstract.Execution`

```csharp
public interface IExecutionChain {
    Task Next();
    IExecutionContext Context { get; }
    IExecutionChain Fork(IExecutionContext context);
    bool IsLastFilter { get; }
}
```

| Member | Type | Description |
|---|---|---|
| `Next()` | `Task` | Execute the next filter or handler in the chain. |
| `Context` | `IExecutionContext` | The current execution context. |
| `Fork(IExecutionContext)` | `IExecutionChain` | Create a copy of the chain (with a new or cloned context) for parallel or retry execution. |
| `IsLastFilter` | `bool` | `true` if the current filter is the last in the chain before the handler. |

---

### IExecutionRequestHandlerInfo

Metadata about a matched request handler -- the route path, HTTP method, handler type, method name, and parameter definitions.

**Package:** `Hardened.Requests.Abstract`
**Namespace:** `Hardened.Requests.Abstract.Execution`

```csharp
public interface IExecutionRequestHandlerInfo {
    string Path { get; }
    string Method { get; }
    Type HandlerType { get; }
    string InvokeMethod { get; }
    int? SuccessStatus => null;
    int? FailureStatus => null;
    int? NullResponseStatus => null;
    IReadOnlyList<IExecutionRequestParameter> Parameters { get; }
}
```

| Member | Type | Description |
|---|---|---|
| `Path` | `string` | The route path pattern. |
| `Method` | `string` | The HTTP method or request method. |
| `HandlerType` | `Type` | The type of the handler class. |
| `InvokeMethod` | `string` | The name of the handler method. |
| `SuccessStatus` | `int?` | Configured success status code (default `null`). |
| `FailureStatus` | `int?` | Configured failure status code (default `null`). |
| `NullResponseStatus` | `int?` | Configured null-response status code (default `null`). |
| `Parameters` | `IReadOnlyList<IExecutionRequestParameter>` | The handler method's parameter definitions. |

---

### IExecutionRequestParameters

Provides indexed and named access to bound request parameters during execution.

**Package:** `Hardened.Requests.Abstract`
**Namespace:** `Hardened.Requests.Abstract.Execution`

```csharp
public interface IExecutionRequestParameters {
    bool TryGetParameter(string parameterName, out object? parameterValue);
    bool TrySetParameter(string parameterName, object parameterValue);
    IReadOnlyList<IExecutionRequestParameter> Info { get; }
    object this[int index] { get; set; }
    int ParameterCount { get; }
    IExecutionRequestParameters Clone();
    object this[string parameterName] { get; set; }
}
```

| Member | Type | Description |
|---|---|---|
| `TryGetParameter(string, out object?)` | `bool` | Try to retrieve a parameter value by name. |
| `TrySetParameter(string, object)` | `bool` | Try to set a parameter value by name. |
| `Info` | `IReadOnlyList<IExecutionRequestParameter>` | Parameter metadata (name, index, type). |
| `this[int]` | `object` | Access parameter by index. |
| `this[string]` | `object` | Access parameter by name. |
| `ParameterCount` | `int` | Number of parameters. |
| `Clone()` | `IExecutionRequestParameters` | Create a copy of the parameters. |

---

### IKnownServices

Pre-resolved common services that are accessed frequently during request processing, avoiding repeated service provider lookups.

**Package:** `Hardened.Requests.Abstract`
**Namespace:** `Hardened.Requests.Abstract.Execution`

```csharp
public interface IKnownServices {
    IContextSerializationService ContextSerializationService { get; }
    IStringConverterService StringConverterService { get; }
}
```

| Member | Type | Description |
|---|---|---|
| `ContextSerializationService` | `IContextSerializationService` | Serialization service for request/response bodies. |
| `StringConverterService` | `IStringConverterService` | Service for converting strings to typed values (used in parameter binding). |

---

## Web Layer (Hardened.Web.Testing)

### ITestWebApp

A test client for sending HTTP requests through the Hardened web pipeline without a real HTTP server. Extends `ITestContext` with HTTP methods.

**Package:** `Hardened.Web.Testing`
**Namespace:** `Hardened.Web.Testing`

```csharp
public interface ITestWebApp : ITestContext {
    IServiceProvider RootServiceProvider { get; }

    Task<TestWebResponse> Get(string path, Action<TestWebRequest>? webRequest = null);
    Task<TestWebResponse> Post(object value, string path, Action<TestWebRequest>? webRequest = null);
    Task<TestWebResponse> Put(object value, string path, Action<TestWebRequest>? webRequest = null);
    Task<TestWebResponse> Patch(object value, string path, Action<TestWebRequest>? webRequest = null);
    Task<TestWebResponse> Delete(string path, Action<TestWebRequest>? webRequest = null);
    Task<TestWebResponse> Request(string method, object? value, string path,
        Action<TestWebRequest>? webRequest = null);
}
```

| Member | Return Type | Description |
|---|---|---|
| `RootServiceProvider` | `IServiceProvider` | The root service provider for the test application. |
| `Get(string, Action<TestWebRequest>?)` | `Task<TestWebResponse>` | Send a GET request. |
| `Post(object, string, Action<TestWebRequest>?)` | `Task<TestWebResponse>` | Send a POST request with a body. |
| `Put(object, string, Action<TestWebRequest>?)` | `Task<TestWebResponse>` | Send a PUT request with a body. |
| `Patch(object, string, Action<TestWebRequest>?)` | `Task<TestWebResponse>` | Send a PATCH request with a body. |
| `Delete(string, Action<TestWebRequest>?)` | `Task<TestWebResponse>` | Send a DELETE request. |
| `Request(string, object?, string, Action<TestWebRequest>?)` | `Task<TestWebResponse>` | Send a request with any HTTP method. |

**TestWebRequest** -- configure request details:

| Property | Type | Description |
|---|---|---|
| `Headers` | `IDictionary<string, StringValues>` | Request headers to send. |
| `Token` | `CancellationToken?` | Optional cancellation token. |

**TestWebResponse** -- inspect response details:

| Member | Type | Description |
|---|---|---|
| `StatusCode` | `int` | The response HTTP status code (defaults to 200). |
| `Headers` | `IDictionary<string, StringValues>` | Response headers. |
| `Body` | `Stream` | The response body stream. |
| `Assert` | `IWebAssertThat` | Fluent assertion helper (e.g., `Assert.Ok()`). |
| `Deserialize<T>()` | `T` | Deserialize the response body as JSON. |

```csharp
[HardenedTest]
public async Task GetProduct_Returns200(ITestWebApp testWebApp) {
    var response = await testWebApp.Get("/products/123");

    response.Assert.Ok();
    var product = response.Deserialize<Product>();
    Assert.Equal("123", product.Id);
}

[HardenedTest]
public async Task CreateProduct_WithHeaders(ITestWebApp testWebApp) {
    var response = await testWebApp.Post(
        new { Name = "Widget", Price = 9.99 },
        "/products",
        req => {
            req.Headers["Authorization"] = "Bearer test-token";
        });

    response.Assert.Ok();
}
```

---

## Testing Layer (Hardened.Shared.Testing)

### ITestContext

Base interface for test contexts. Provides retry/polling capabilities, structured step logging, and cancellation.

**Package:** `Hardened.Shared.Testing`
**Namespace:** `Hardened.Shared.Testing`

```csharp
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

| Member | Type | Description |
|---|---|---|
| `Retry` | `IRetryEngine` | Retry/polling engine for eventual consistency testing. |
| `CancellationRequest` | `CancellationToken` | Token that signals test cancellation. |
| `Step(...)` | `void` / `T` / `Task` / `Task<T>` | Execute a named step with structured logging (sync and async overloads). |
| `Logger` | `ILogger` | Logger instance for the test. |

```csharp
[HardenedTest]
public async Task OrderWorkflow(ITestContext testContext, IOrderService orderService) {
    var order = await testContext.Step(
        async () => await orderService.Create(new CreateOrderRequest { Item = "Widget" }),
        "Create order");

    await testContext.Step(
        async () => await orderService.Confirm(order.Id),
        "Confirm order {OrderId}", order.Id);
}
```

---

### IRetryEngine

A retry/polling utility for testing eventual consistency. Repeats a function until a condition is met or the timeout expires.

**Package:** `Hardened.Shared.Testing`
**Namespace:** `Hardened.Shared.Testing`

```csharp
public interface IRetryEngine {
    int Delay { get; set; }

    Task TillTrue(Func<Task<bool>> testFunc,
        string description, params object[] parameters);

    Task TillFalse(Func<Task<bool>> testFunc,
        string description, params object[] parameters);

    Task<T> TillValue<T>(Func<Task<T>> value,
        string description, params object[] parameters);
}
```

| Member | Type | Description |
|---|---|---|
| `Delay` | `int` | Delay between retries in milliseconds (default: 1000). |
| `TillTrue(...)` | `Task` | Retry until the function returns `true`. |
| `TillFalse(...)` | `Task` | Retry until the function returns `false`. |
| `TillValue<T>(...)` | `Task<T>` | Retry until the function returns a non-default value. |

```csharp
[HardenedTest]
public async Task EventualConsistency(ITestContext testContext, IOrderService orderService) {
    var orderId = await testContext.Step(
        async () => (await orderService.Create(new CreateOrderRequest())).Id,
        "Create order");

    testContext.Retry.Delay = 500;

    await testContext.Retry.TillTrue(
        async () => {
            var order = await orderService.GetById(orderId);
            return order?.Status == "Confirmed";
        },
        "Wait for order {OrderId} to be confirmed", orderId);
}
```

---

## AWS Layer (Hardened.Amz.*)

### IDynamoDbClientProvider

Factory for obtaining configured `AmazonDynamoDBClient` instances. Supports named clients for multi-region or multi-table configurations.

**Package:** `Hardened.Amz.DynamoDbClient`
**Namespace:** `Hardened.Amz.DynamoDbClient`

```csharp
public interface IDynamoDbClientProvider {
    AmazonDynamoDBClient GetClient(string clientName = "");
}
```

| Member | Return Type | Description |
|---|---|---|
| `GetClient(string)` | `AmazonDynamoDBClient` | Get a DynamoDB client by name. Empty string returns the default client. |

```csharp
[TransientService(As = typeof(IOrderRepository))]
public class DynamoOrderRepository : IOrderRepository {
    private readonly AmazonDynamoDBClient _client;

    public DynamoOrderRepository(IDynamoDbClientProvider clientProvider) {
        _client = clientProvider.GetClient();
    }

    public async Task<Order?> GetById(string id) {
        var response = await _client.GetItemAsync(new GetItemRequest {
            TableName = "Orders",
            Key = new Dictionary<string, AttributeValue> {
                ["PK"] = new(id)
            }
        });
        return response.Item.Count > 0 ? MapToOrder(response.Item) : null;
    }
}
```

---

### ISqsClient

Client for sending messages to SQS queues. Messages are serialized automatically.

**Package:** `Hardened.Amz.SqsClient`
**Namespace:** `Hardened.SqsClient`

```csharp
public interface ISqsClient {
    Task SendMessage<TMessage>(TMessage message);
}
```

| Member | Return Type | Description |
|---|---|---|
| `SendMessage<TMessage>(TMessage)` | `Task` | Serialize and send a message to the configured SQS queue. |

```csharp
[Post("/orders")]
public async Task<Order> CreateOrder(
    [FromBody] CreateOrderRequest request,
    ISqsClient sqsClient) {
    var order = new Order { Id = Guid.NewGuid().ToString(), Item = request.Item };

    await sqsClient.SendMessage(new OrderCreatedEvent {
        OrderId = order.Id,
        Timestamp = DateTime.UtcNow
    });

    return order;
}
```

---

## Templates Layer (Hardened.Templates.Abstract)

### ITemplateHelper

Defines a custom helper function that can be called from within compiled templates.

**Package:** `Hardened.Templates.Abstract`
**Namespace:** `Hardened.Templates.Abstract`

```csharp
public interface ITemplateHelper {
    ValueTask<object> Execute(ITemplateExecutionContext handlerDataContext, params object[] arguments);
}
```

| Member | Return Type | Description |
|---|---|---|
| `Execute(ITemplateExecutionContext, params object[])` | `ValueTask<object>` | Execute the helper with the given template context and arguments. |

```csharp
[SingletonService(As = typeof(ITemplateHelper))]
public class FormatDateHelper : ITemplateHelper {
    public ValueTask<object> Execute(
        ITemplateExecutionContext context, params object[] arguments) {
        if (arguments.Length > 0 && arguments[0] is DateTime date) {
            return new ValueTask<object>(date.ToString("yyyy-MM-dd"));
        }
        return new ValueTask<object>("");
    }
}
```

---

## Related Pages

- [Attribute Reference](attributes.md) -- complete attribute reference
- [Configuration Reference](configuration.md) -- the configuration model system
- [Architecture Overview](../architecture/overview.md) -- how interfaces fit into the layer architecture
