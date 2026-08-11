# Attribute Reference

This page is a comprehensive reference for every attribute in the Hardened framework, organized by package. Each entry includes the attribute's target, properties, namespace, and a usage example.

---

## Service Registration (DependencyModules.Runtime.Attributes)

Registration is provided by [DependencyModules](https://github.com/ipjohnson/DependencyModules). Hardened's
generator reads these attributes against its own `[HardenedModule]` entry point, so they work in a
Hardened application without declaring a `[DependencyModule]`.

### [TransientService]

Register a class for dependency injection with a transient lifetime. The source generator emits the `IServiceCollection` call at build time. `[SingletonService]` and `[ScopedService]` are identical apart from the lifetime.

| Property | Type | Default | Description |
|---|---|---|---|
| `As` | `Type?` | `null` | The service type to register as. When unset, the first non-capability interface the class implements is inferred; failing that, the class itself. |
| `Key` | `object?` | `null` | Registers as a keyed service. |
| `Using` | `RegistrationType` | `Add` | Which registration method to use: `Add`, `Try`, `TryEnumerable` or `Replace`. |
| `Realm` | `Type?` | `null` | The module realm the registration belongs to. |

**Target:** Class, Method

**Namespace:** `DependencyModules.Runtime.Attributes`

```csharp
using DependencyModules.Runtime.Attributes;

// Service type inferred from the first non-capability interface
[TransientService]
public class OrderService : IOrderService { }

// Register only for a specific interface
[TransientService(As = typeof(IPaymentGateway))]
public class StripeGateway : IPaymentGateway, IDisposable { }

// Try-register: only if no prior registration exists
[TransientService(As = typeof(ILogger), Using = RegistrationType.Try)]
public class DefaultLogger : ILogger { }
```

---

### [SingletonService]

Register a service with a singleton lifetime -- one instance for the entire application.

**Target:** Class, Method
**Properties:** `As`, `Key`, `Using`, `Realm`
**Namespace:** `DependencyModules.Runtime.Attributes`

```csharp
[SingletonService]
public class CacheService : ICacheService { }
```

---

### [ScopedService]

Register a service with a scoped lifetime -- one instance per request scope.

**Target:** Class, Method
**Properties:** `As`, `Key`, `Using`, `Realm`
**Namespace:** `DependencyModules.Runtime.Attributes`

```csharp
[ScopedService]
public class UnitOfWork : IUnitOfWork { }
```

---

### [IfEnvironment]

Restrict a service registration to specific environment names. The class is only registered in the DI container when `IHardenedEnvironment.Name` matches the specified value.

| Property | Type | Default | Description |
|---|---|---|---|
| `Environment` | `string` | *(required)* | The environment name to match (case-insensitive). |

**Target:** Class (AllowMultiple = true)
**Namespace:** `Hardened.Shared.Runtime.Attributes`

```csharp
[TransientService(As = typeof(IEmailSender))]
[IfEnvironment("Production")]
public class SmtpEmailSender : IEmailSender { }

[TransientService(As = typeof(IEmailSender))]
[IfEnvironment("Development", "Testing")]
public class FakeEmailSender : IEmailSender { }
```

---

### [ConfigurationModel]

Mark an interface or partial class as a configuration model. The source generator creates a concrete implementation class, registers it as a singleton in the DI container, and wires up environment variable bindings.

**Target:** Interface or Class
**Properties:** None
**Namespace:** `Hardened.Shared.Runtime.Attributes`

```csharp
// Interface-based configuration model
[ConfigurationModel]
public interface IDatabaseConfig {
    [FromEnvironmentVariable("DB_CONNECTION_STRING")]
    string ConnectionString { get; }

    int MaxPoolSize { get; }
}

// Partial-class configuration model with private field defaults
[ConfigurationModel]
public partial class CanaryConfig {
    [FromEnvironmentVariable("CANARY_DATA_TABLE")]
    private string _dynamoDataTable = "canary-data-table";

    private int _retentionDays = 180;
}
```

!!! info "See Also"
    For a full guide on the configuration system, see the [Configuration Reference](configuration.md).

---

### [FromEnvironmentVariable]

Bind a configuration model property to an environment variable. The source generator reads the value from `System.Environment.GetEnvironmentVariable()` at startup and performs type conversion automatically.

| Property | Type | Default | Description |
|---|---|---|---|
| `EnvironmentVariable` | `string` | *(required)* | The name of the environment variable to read. |

**Target:** Property (on a `[ConfigurationModel]` interface) or Field (on a `[ConfigurationModel]` partial class)
**Namespace:** `Hardened.Shared.Runtime.Attributes`

```csharp
[ConfigurationModel]
public interface IApiConfig {
    [FromEnvironmentVariable("API_BASE_URL")]
    string BaseUrl { get; }

    [FromEnvironmentVariable("API_TIMEOUT_MS")]
    int TimeoutMs { get; }

    int MaxRetries { get; }  // No env var binding; uses default or IAppConfig
}
```

---

### [HideConfigurationField]

Exclude a property from the generated configuration implementation. Use this for computed properties or properties that are always provided programmatically.

**Target:** Property
**Properties:** None
**Namespace:** `Hardened.Shared.Runtime.Attributes`

```csharp
[ConfigurationModel]
public interface IAppSettings {
    string AppName { get; }

    [HideConfigurationField]
    string Version { get; }  // Not included in generated implementation
}
```

---

### [ConfigurationProvider]

Mark a class as a configuration provider. Configuration providers participate in the configuration resolution pipeline.

**Target:** Class
**Properties:** None
**Namespace:** `Hardened.Shared.Runtime.Attributes`

```csharp
[ConfigurationProvider]
public class CustomConfigProvider { }
```

---

### [HardenedModule]

Mark a partial class as the application module entry point. The source generator emits the `ConfigureModule` method into this class, which registers all discovered services, routes, configuration, and module dependencies.

**Target:** Class
**Properties:** None
**Namespace:** `Hardened.Shared.Runtime.Attributes`

!!! warning "Partial Class Required"
    The class must be declared `partial` so the source generator can extend it with the generated `ConfigureModule` method.

```csharp
using Hardened.Shared.Runtime.Attributes;

[HardenedModule]
[AspNetCoreRuntime.Module]
public partial class Application
{
    public static WebApplicationBuilder CreateBuilder(string[] args)
    {
        var hardenedApp = new Application();
        var environment = new EnvironmentImpl(arguments: args);

        var builder = WebApplication.CreateBuilder(args);
        hardenedApp.ConfigureModule(environment, builder.Services);

        return builder;
    }
}
```

---

## Requests (Hardened.Requests.Abstract.Attributes)

These attributes operate at the request pipeline level and are used across all runtime types (web, Lambda, DDB streams, SQS, canaries).

### [HardenedFunction]

Mark a method as a request handler. This is the generic handler registration attribute used by Lambda and other non-web runtimes.

| Property | Type | Default | Description |
|---|---|---|---|
| `FunctionName` | `string?` | `null` | Optional name for the function. When `null`, the method name is used. |

**Target:** Method
**Namespace:** `Hardened.Requests.Abstract.Attributes`

```csharp
using Hardened.Requests.Abstract.Attributes;

public class OrderProcessor
{
    [HardenedFunction("process-order")]
    public async Task<OrderResult> ProcessOrder([FromBody] OrderRequest request)
    {
        // Handle the order
    }
}
```

---

### [FromBody]

Deserialize the request body into the parameter type. The framework uses the configured serializer (System.Text.Json by default) to parse the body stream.

**Target:** Parameter
**Properties:** None
**Namespace:** `Hardened.Requests.Abstract.Attributes`

```csharp
[Post("/orders")]
public async Task<Order> CreateOrder([FromBody] CreateOrderRequest request)
{
    return await _orderService.Create(request);
}
```

---

### [FromServices]

Resolve the parameter from the DI container. This is also auto-detected for parameters whose type is an interface, so explicit use is only required when disambiguation is needed.

**Target:** Parameter
**Properties:** None
**Namespace:** `Hardened.Requests.Abstract.Attributes`

```csharp
[Get("/status")]
public string Status([FromServices] IHealthChecker checker)
{
    return checker.GetStatus();
}
```

!!! tip
    Interface-typed parameters are automatically resolved from DI without needing `[FromServices]`. Use it explicitly only when the parameter type is a concrete class that could also be bound from the request.

---

### [AuthorizeActivities]

Require authorization for specific activities before the handler executes.

| Property | Type | Default | Description |
|---|---|---|---|
| `Activities` | `string[]` | *(required)* | One or more activity names that must be authorized. |

**Target:** Method
**Namespace:** `Hardened.Requests.Abstract.Attributes`

```csharp
[Get("/admin/users")]
[AuthorizeActivities("admin:read-users")]
public async Task<IEnumerable<User>> ListUsers()
{
    return await _userService.GetAll();
}
```

---

### [RawResponse]

Skip response serialization. The handler's return value is written directly to the response body as the specified content type.

| Property | Type | Default | Description |
|---|---|---|---|
| `contentType` | `string` | `"text/plain"` | The content type to set on the response. |

**Target:** Method
**Namespace:** `Hardened.Requests.Abstract.Attributes`

```csharp
[Get("/health")]
[RawResponse("text/plain")]
public string HealthCheck()
{
    return "OK";
}
```

---

### [Template]

Associate a compiled template with a handler method. The handler's return value is passed to the template engine as the data model.

| Property | Type | Default | Description |
|---|---|---|---|
| `TemplateName` | `string` | *(required)* | The name of the compiled template to render. |

**Target:** Method
**Namespace:** `Hardened.Requests.Abstract.Attributes`

```csharp
[Get("/dashboard")]
[Template("dashboard.html")]
public DashboardModel GetDashboard()
{
    return new DashboardModel { Title = "Dashboard", Items = _items };
}
```

---

## Web (Hardened.Web.Runtime.Attributes)

These attributes are specific to web (HTTP) runtimes -- ASP.NET Core and Lambda Web (API Gateway).

### [Get]

Register an HTTP GET route handler.

| Property | Type | Default | Description |
|---|---|---|---|
| `Path` | `string` | `""` | The route path. Supports `{parameter}` tokens. |
| `SuccessStatus` | `int` | `200` | HTTP status code for successful responses. |
| `ValidationErrorStatus` | `int` | `400` | HTTP status code for validation errors. |
| `NullReturnStatus` | `int` | `404` | HTTP status code when the handler returns `null`. |
| `ErrorStatus` | `int` | `500` | HTTP status code for unhandled errors. |

**Target:** Method
**Namespace:** `Hardened.Web.Runtime.Attributes`

```csharp
using Hardened.Web.Runtime.Attributes;

[Get("/products/{id}")]
public async Task<Product?> GetProduct(string id)
{
    return await _productRepo.FindById(id);
    // Returns 200 with product, or 404 if null
}

[Get("/items", NullReturnStatus = 200)]
public async Task<IEnumerable<Item>?> ListItems()
{
    return await _itemRepo.GetAll();
    // Returns 200 even if null (e.g., empty list)
}
```

---

### [Post]

Register an HTTP POST route handler.

| Property | Type | Default | Description |
|---|---|---|---|
| `Path` | `string` | `""` | The route path. |

**Target:** Method
**Namespace:** `Hardened.Web.Runtime.Attributes`

```csharp
[Post("/products")]
public async Task<Product> CreateProduct([FromBody] CreateProductRequest request)
{
    return await _productRepo.Create(request);
}
```

---

### [Put]

Register an HTTP PUT route handler.

| Property | Type | Default | Description |
|---|---|---|---|
| `Path` | `string` | `""` | The route path. |
| `SuccessStatus` | `int` | `200` | HTTP status code for successful responses. |
| `ValidationErrorStatus` | `int` | `400` | HTTP status code for validation errors. |
| `NullReturnStatus` | `int` | `404` | HTTP status code when the handler returns `null`. |
| `ErrorStatus` | `int` | `500` | HTTP status code for unhandled errors. |

**Target:** Method
**Namespace:** `Hardened.Web.Runtime.Attributes`

```csharp
[Put("/products/{id}")]
public async Task<Product> UpdateProduct(string id, [FromBody] UpdateProductRequest request)
{
    return await _productRepo.Update(id, request);
}
```

---

### [Delete]

Register an HTTP DELETE route handler.

**Target:** Method
**Namespace:** `Hardened.Web.Runtime.Attributes`

```csharp
[Delete("/products/{id}")]
public async Task DeleteProduct(string id)
{
    await _productRepo.Delete(id);
}
```

---

### [Patch]

Register an HTTP PATCH route handler.

**Target:** Method
**Namespace:** `Hardened.Web.Runtime.Attributes`

```csharp
[Patch("/products/{id}")]
public async Task<Product> PatchProduct(string id, [FromBody] PatchProductRequest request)
{
    return await _productRepo.Patch(id, request);
}
```

---

### [BasePath]

Set a URL prefix for all routes in a class or assembly. When applied to a class, all route attributes in that class are prefixed. When applied at the assembly level, all routes in the assembly are prefixed.

| Property | Type | Default | Description |
|---|---|---|---|
| `Path` | `string` | *(required)* | The URL prefix. |

**Target:** Class or Assembly (AllowMultiple = false)
**Namespace:** `Hardened.Web.Runtime.Attributes`

```csharp
// Class-level base path
[BasePath("/api/v1")]
public class ProductController
{
    [Get("/products")]       // Effective path: /api/v1/products
    public Task<IEnumerable<Product>> List() { }

    [Get("/products/{id}")] // Effective path: /api/v1/products/{id}
    public Task<Product?> Get(string id) { }
}
```

```csharp
// Assembly-level base path
[assembly: BasePath("/api")]
```

---

### [FromHeader]

Bind a method parameter from an HTTP request header.

| Property | Type | Default | Description |
|---|---|---|---|
| `Name` | `string?` | `null` | The header name. When `null`, the parameter name is used. |

**Target:** Parameter
**Namespace:** `Hardened.Web.Runtime.Attributes`

```csharp
[Get("/protected")]
public string Protected([FromHeader("Authorization")] string authToken)
{
    return $"Token: {authToken}";
}

// When Name is omitted, uses the parameter name as the header name
[Get("/info")]
public string Info([FromHeader] string accept)
{
    return $"Accept: {accept}";
}
```

---

### [FromQueryString]

Bind a method parameter from a URL query string value.

| Property | Type | Default | Description |
|---|---|---|---|
| `Name` | `string?` | `null` | The query parameter name. When `null`, the parameter name is used. |

**Target:** Parameter
**Namespace:** `Hardened.Web.Runtime.Attributes`

```csharp
[Get("/search")]
public async Task<SearchResult> Search(
    [FromQueryString] string q,
    [FromQueryString("page_size")] int pageSize = 20)
{
    return await _searchService.Search(q, pageSize);
}
```

---

### [CacheControl]

Set HTTP cache control headers on the response.

| Property | Type | Default | Description |
|---|---|---|---|
| `MaxAge` | `int` | `0` | Cache max-age in seconds. |
| `Type` | `CacheControlEnum` | `MaxAge \| Public` | Cache control directives (flags enum). |

**Target:** Method
**Namespace:** `Hardened.Web.Runtime.Attributes`

**CacheControlEnum values:** `MaxAge`, `NoCache`, `NoStore`, `NoTransform`, `Public`, `Private`

```csharp
// Cache for 5 minutes, public
[Get("/catalog")]
[CacheControl(MaxAge = 300)]
public async Task<Catalog> GetCatalog() { }

// Private cache, 60 seconds
[Get("/profile")]
[CacheControl(MaxAge = 60, Type = CacheControlEnum.MaxAge | CacheControlEnum.Private)]
public async Task<Profile> GetProfile() { }

// No caching
[Get("/live-data")]
[CacheControl(Type = CacheControlEnum.NoStore)]
public async Task<LiveData> GetLiveData() { }
```

---

### [WebLibrary]

Mark an assembly as a Hardened web library. This attribute enables the web source generator to discover and process routes defined in library assemblies.

**Target:** Assembly
**Properties:** None
**Namespace:** `Hardened.Web.Runtime.Attributes`

```csharp
[assembly: WebLibrary]
```

---

## Testing (Hardened.Shared.Testing.Attributes)

These attributes support the Hardened testing framework, built on top of xUnit.

### [HardenedTest]

Mark a method as a Hardened test. This attribute extends xUnit's `FactAttribute` with custom test discovery that supports DI parameter injection, mock creation, and environment configuration.

**Target:** Method (AllowMultiple = false)
**Properties:** Inherits all properties from `FactAttribute` (e.g., `DisplayName`, `Skip`, `Timeout`)
**Namespace:** `Hardened.Shared.Testing.Attributes`

```csharp
using Hardened.Shared.Testing.Attributes;

public class OrderServiceTests
{
    [HardenedTest]
    public async Task CreateOrder_ReturnsOrder(IOrderService orderService)
    {
        var result = await orderService.Create(new CreateOrderRequest { Item = "Widget" });
        Assert.NotNull(result);
    }
}
```

---

### [HardenedTestEntryPoint]

Specify the application module entry point for tests. This tells the test framework which `[HardenedModule]` class to use as the composition root, giving tests access to the full DI container and pipeline.

| Property | Type | Default | Description |
|---|---|---|---|
| `EntryPoint` | `Type` | *(required)* | The `[HardenedModule]` class to use. |

**Target:** Assembly, Class, or Method
**Namespace:** `Hardened.Shared.Testing.Attributes`

```csharp
// Assembly-level (most common): applies to all tests in the assembly
[assembly: HardenedTestEntryPoint(typeof(Application))]

// Class-level: override the entry point for a specific test class
[HardenedTestEntryPoint(typeof(AlternateApplication))]
public class SpecialTests { }
```

---

### [Mock]

Create an NSubstitute mock for the parameter type and register it as a singleton in the test's DI container. The mock is injected into the test method and can be configured using standard NSubstitute syntax.

**Target:** Parameter
**Properties:** None
**Namespace:** `Hardened.Shared.Testing.Attributes`

```csharp
[HardenedTest]
public async Task CreateOrder_CallsPaymentGateway(
    IOrderService orderService,
    [Mock] IPaymentGateway paymentGateway)
{
    paymentGateway.ChargeAsync(Arg.Any<decimal>())
        .Returns(new PaymentResult { Success = true });

    var order = await orderService.Create(new CreateOrderRequest { Amount = 99.99m });

    Assert.NotNull(order);
    await paymentGateway.Received(1).ChargeAsync(99.99m);
}
```

---

### [EnvironmentName]

Set the environment name for a test method or all tests in a class.

| Property | Type | Default | Description |
|---|---|---|---|
| `Name` | `string` | *(required)* | The environment name to use. |

**Target:** Method or Class
**Namespace:** `Hardened.Shared.Testing.Attributes`

```csharp
[HardenedTest]
[EnvironmentName("Production")]
public async Task ProductionConfig_UsesCorrectTimeout(IApiConfig config)
{
    Assert.Equal(15000, config.TimeoutMs);
}
```

---

### [EnvironmentValue]

Set an environment variable value for the duration of a test.

| Property | Type | Default | Description |
|---|---|---|---|
| `Variable` | `string` | *(required)* | The environment variable name. |
| `Value` | `string` | *(required)* | The value to set. |

**Target:** Method or Class
**Namespace:** `Hardened.Shared.Testing.Attributes`

```csharp
[HardenedTest]
[EnvironmentValue("API_BASE_URL", "https://staging.example.com")]
[EnvironmentValue("API_KEY", "test-key-123")]
public async Task ApiClient_UsesConfiguredUrl(IApiConfig config)
{
    Assert.Equal("https://staging.example.com", config.BaseUrl);
    Assert.Equal("test-key-123", config.ApiKey);
}
```

---

## AWS (Hardened.Amz.*)

Attributes for AWS Lambda runtimes, DynamoDB Streams, and related services.

### [LambdaWebApplication]

Mark a class as a Lambda web application entry point. Used with API Gateway (REST or HTTP API) integration.

| Property | Type | Default | Description |
|---|---|---|---|
| `Version` | `ProxyIntegrationType` | `ApiGateway` | The API Gateway proxy integration type. |

**Target:** Class
**Namespace:** `Hardened.Amz.Web.Lambda.Runtime`

**ProxyIntegrationType values:** `ApiGateway`, `HttpApiV2`

```csharp
using Hardened.Amz.Web.Lambda.Runtime;
using Hardened.Shared.Runtime.Attributes;

[HardenedModule]
[LambdaWebApplication(Version = ProxyIntegrationType.HttpApiV2)]
public partial class LambdaApplication { }
```

---

### [OldImage]

Bind a DynamoDB Streams handler parameter to the old image (the item state before the change) from a stream record.

**Target:** Parameter
**Properties:** None
**Namespace:** `Hardened.Amz.Function.DDB.Runtime.Attributes`

```csharp
using Hardened.Amz.Function.DDB.Runtime.Attributes;
using Amazon.DynamoDBv2.Model;

[HardenedFunction]
public async Task HandleDelete(
    [OldImage] Dictionary<string, AttributeValue> oldImage)
{
    var deletedId = oldImage["Id"].S;
    // Process the deleted item
}
```

---

### [NewImage]

Bind a DynamoDB Streams handler parameter to the new image (the item state after the change) from a stream record.

**Target:** Parameter
**Properties:** None
**Namespace:** `Hardened.Amz.Function.DDB.Runtime.Attributes`

```csharp
using Hardened.Amz.Function.DDB.Runtime.Attributes;
using Amazon.DynamoDBv2.Model;

[HardenedFunction]
public async Task HandleInsert(
    [NewImage] Dictionary<string, AttributeValue> newImage)
{
    var insertedId = newImage["Id"].S;
    // Process the new item
}
```

---

### [LocalDynamoDb]

Start a local DynamoDB container (via Testcontainers) for integration testing. The attribute creates a shared DynamoDB Local container and registers a test `IDynamoDbClientProvider` in the DI container.

**Target:** Method or Class
**Properties:** None
**Namespace:** `Hardened.Amz.DynamoDbClient.Testing`

```csharp
using Hardened.Amz.DynamoDbClient.Testing;
using Hardened.Shared.Testing.Attributes;

[HardenedTest]
[LocalDynamoDb]
public async Task CreateTable_Succeeds(IDynamoDbClientProvider clientProvider)
{
    var client = clientProvider.GetClient();

    await client.CreateTableAsync(new CreateTableRequest
    {
        TableName = "TestTable",
        KeySchema = new List<KeySchemaElement>
        {
            new("PK", KeyType.HASH)
        },
        AttributeDefinitions = new List<AttributeDefinition>
        {
            new("PK", ScalarAttributeType.S)
        },
        BillingMode = BillingMode.PAY_PER_REQUEST
    });

    var tables = await client.ListTablesAsync();
    Assert.Contains("TestTable", tables.TableNames);
}
```

---

## Canaries (Hardened.Amz.Canaries.Runtime.Attributes)

Attributes for the canary testing framework.

### [HardenedCanary]

Define a canary test method. Canaries are automated health checks that run on a schedule as AWS Lambda functions. This attribute extends xUnit's `FactAttribute`, so canaries can also be run as local unit tests.

| Property | Type | Default | Description |
|---|---|---|---|
| `Name` | `string` | `""` | The canary name. When empty, the method name is used. |
| `Frequency` | `int` | `1` | How often to execute the canary. |
| `Unit` | `CanaryFrequencyUnit` | `Minute` | The frequency unit: `Second`, `Minute`, `Hour`, or `Day`. |
| `ReportMetric` | `bool` | `true` | Whether to report metrics to CloudWatch. |
| `FlightStyle` | `CanaryFlightStyle` | `Loose` | `Strict` = fail if execution exceeds frequency. `Loose` = only fail on timeout. |
| `AllowConcurrentExecution` | `bool` | `false` | Whether multiple instances of this canary can run concurrently. |

**Target:** Method (AllowMultiple = false)
**Namespace:** `Hardened.Amz.Canaries.Runtime.Attributes`

```csharp
using Hardened.Amz.Canaries.Runtime.Attributes;

public class ApiCanaries
{
    [HardenedCanary(Frequency = 5, Unit = CanaryFrequencyUnit.Minute)]
    public async Task ApiHealthCheck(ITestContext testContext, HttpClient client)
    {
        await testContext.Step(async () =>
        {
            var response = await client.GetAsync("https://api.example.com/health");
            response.EnsureSuccessStatusCode();
        }, "Check API health endpoint");
    }

    [HardenedCanary(
        Name = "database-canary",
        Frequency = 1,
        Unit = CanaryFrequencyUnit.Hour,
        FlightStyle = CanaryFlightStyle.Strict)]
    public async Task DatabaseConnectivity(ITestContext testContext, IDbConnection db)
    {
        await testContext.Step(async () =>
        {
            await db.OpenAsync();
        }, "Verify database connectivity");
    }
}
```

---

## Quick Reference Table

A consolidated lookup table of every attribute, sorted by namespace.

| Attribute | Namespace | Target | Key Properties |
|---|---|---|---|
| `[TransientService]` | `DependencyModules.Runtime.Attributes` | Class/Method | `As`, `Key`, `Using`, `Realm` |
| `[SingletonService]` | `DependencyModules.Runtime.Attributes` | Class/Method | `As`, `Key`, `Using`, `Realm` |
| `[ScopedService]` | `DependencyModules.Runtime.Attributes` | Class/Method | `As`, `Key`, `Using`, `Realm` |
| `[CrossWireService]` | `DependencyModules.Runtime.Attributes` | Class/Method | `Lifetime`, `Key`, `Using`, `Realm` |
| `[IfEnvironment]` | `DependencyModules.Runtime.Attributes` | Class/Method | `Environments` |
| `[ConfigurationModel]` | `Hardened.Shared.Runtime.Attributes` | Interface/Class | -- |
| `[FromEnvironmentVariable]` | `Hardened.Shared.Runtime.Attributes` | Property/Field | `EnvironmentVariable` |
| `[HideConfigurationField]` | `Hardened.Shared.Runtime.Attributes` | Property | -- |
| `[ConfigurationProvider]` | `Hardened.Shared.Runtime.Attributes` | Class | -- |
| `[HardenedModule]` | `Hardened.Shared.Runtime.Attributes` | Class | -- |
| `[HardenedFunction]` | `Hardened.Requests.Abstract.Attributes` | Method | `FunctionName` |
| `[FromBody]` | `Hardened.Requests.Abstract.Attributes` | Parameter | -- |
| `[FromServices]` | `Hardened.Requests.Abstract.Attributes` | Parameter | -- |
| `[AuthorizeActivities]` | `Hardened.Requests.Abstract.Attributes` | Method | `Activities` |
| `[RawResponse]` | `Hardened.Requests.Abstract.Attributes` | Method | `contentType` |
| `[Template]` | `Hardened.Requests.Abstract.Attributes` | Method | `TemplateName` |
| `[Get]` | `Hardened.Web.Runtime.Attributes` | Method | `Path`, `SuccessStatus`, `NullReturnStatus`, `ValidationErrorStatus`, `ErrorStatus` |
| `[Post]` | `Hardened.Web.Runtime.Attributes` | Method | `Path` |
| `[Put]` | `Hardened.Web.Runtime.Attributes` | Method | `Path`, `SuccessStatus`, `NullReturnStatus`, `ValidationErrorStatus`, `ErrorStatus` |
| `[Delete]` | `Hardened.Web.Runtime.Attributes` | Method | -- |
| `[Patch]` | `Hardened.Web.Runtime.Attributes` | Method | -- |
| `[BasePath]` | `Hardened.Web.Runtime.Attributes` | Class/Assembly | `Path` |
| `[FromHeader]` | `Hardened.Web.Runtime.Attributes` | Parameter | `Name` |
| `[FromQueryString]` | `Hardened.Web.Runtime.Attributes` | Parameter | `Name` |
| `[CacheControl]` | `Hardened.Web.Runtime.Attributes` | Method | `MaxAge`, `Type` |
| `[WebLibrary]` | `Hardened.Web.Runtime.Attributes` | Assembly | -- |
| `[HardenedTest]` | `Hardened.Shared.Testing.Attributes` | Method | *(inherits FactAttribute)* |
| `[HardenedTestEntryPoint]` | `Hardened.Shared.Testing.Attributes` | Assembly/Class/Method | `EntryPoint` |
| `[Mock]` | `Hardened.Shared.Testing.Attributes` | Parameter | -- |
| `[EnvironmentName]` | `Hardened.Shared.Testing.Attributes` | Method/Class | `Name` |
| `[EnvironmentValue]` | `Hardened.Shared.Testing.Attributes` | Method/Class | `Variable`, `Value` |
| `[LambdaWebApplication]` | `Hardened.Amz.Web.Lambda.Runtime` | Class | `Version` |
| `[OldImage]` | `Hardened.Amz.Function.DDB.Runtime.Attributes` | Parameter | -- |
| `[NewImage]` | `Hardened.Amz.Function.DDB.Runtime.Attributes` | Parameter | -- |
| `[LocalDynamoDb]` | `Hardened.Amz.DynamoDbClient.Testing` | Method/Class | -- |
| `[HardenedCanary]` | `Hardened.Amz.Canaries.Runtime.Attributes` | Method | `Name`, `Frequency`, `Unit`, `ReportMetric`, `FlightStyle`, `AllowConcurrentExecution` |

---

## Related Pages

- [Interfaces Reference](interfaces.md) -- key interfaces and their members
- [Configuration Reference](configuration.md) -- the `[ConfigurationModel]` system in detail
- [Architecture Overview](../architecture/overview.md) -- how attributes are consumed by source generators
