# Web Testing

Hardened provides an in-process HTTP testing framework through `ITestWebApp`. It lets you send requests to your web routes and assert on responses without starting a real HTTP server. Requests flow through the full Hardened pipeline -- routing, filters, parameter binding, and serialization.

**Package:** `Hardened.Web.Testing` (namespace `Hardened.Web.Testing`)

---

## Setup

### [WebTesting] Assembly Attribute

Enable web testing by adding the `[WebTesting]` attribute at the assembly level in your test project:

```csharp
// Bootstrap.cs
using Hardened.Shared.Testing.Attributes;
using Hardened.Web.Testing;

[assembly: HardenedTestEntryPoint(typeof(Application))]
[assembly: WebTesting]
```

The `[WebTesting]` attribute:

1. Wires up the web execution handler for tests
2. Enables `ITestWebApp` parameter injection in test methods
3. Configures the middleware pipeline for in-process request processing

### Required Packages

```xml
<ItemGroup>
    <PackageReference Include="Hardened.Web.Testing" Version="0.1.0-Alpha*" />
    <PackageReference Include="Hardened.Shared.Testing" Version="0.1.0-Alpha*" />
    <PackageReference Include="xunit" Version="2.*" />
    <PackageReference Include="xunit.runner.visualstudio" Version="2.*" />
    <PackageReference Include="Microsoft.NET.Test.Sdk" Version="17.*" />
</ItemGroup>
```

---

## ITestWebApp

`ITestWebApp` is the primary interface for sending test HTTP requests. It extends `ITestContext`, so you also have access to retry, step, and logging utilities.

### Definition

```csharp
namespace Hardened.Web.Testing;

public interface ITestWebApp : ITestContext {
    IServiceProvider RootServiceProvider { get; }

    Task<TestWebResponse> Get(
        string path,
        Action<TestWebRequest>? webRequest = null);

    Task<TestWebResponse> Post(
        object value,
        string path,
        Action<TestWebRequest>? webRequest = null);

    Task<TestWebResponse> Put(
        object value,
        string path,
        Action<TestWebRequest>? webRequest = null);

    Task<TestWebResponse> Patch(
        object value,
        string path,
        Action<TestWebRequest>? webRequest = null);

    Task<TestWebResponse> Delete(
        string path,
        Action<TestWebRequest>? webRequest = null);

    Task<TestWebResponse> Request(
        string method,
        object? value,
        string path,
        Action<TestWebRequest>? webRequest = null);
}
```

### Methods

| Method | Description |
|---|---|
| `Get(path)` | Send a GET request |
| `Post(value, path)` | Send a POST request with a body |
| `Put(value, path)` | Send a PUT request with a body |
| `Patch(value, path)` | Send a PATCH request with a body |
| `Delete(path)` | Send a DELETE request |
| `Request(method, value, path)` | Send a request with a custom HTTP method |

All methods accept an optional `Action<TestWebRequest>` to configure headers and cancellation.

---

## Basic Usage

### GET Request

```csharp
using Hardened.Shared.Testing.Attributes;
using Hardened.Web.Testing;
using Xunit;

public class ProductApiTests {
    [HardenedTest]
    public async Task GetProducts_ReturnsOk(ITestWebApp app) {
        var response = await app.Get("/api/products");

        response.Assert.Ok();
    }

    [HardenedTest]
    public async Task GetProduct_ReturnsProduct(ITestWebApp app) {
        var response = await app.Get("/api/products/prod-1");

        response.Assert.Ok();
        var product = response.Deserialize<Product>();
        Assert.Equal("prod-1", product.Id);
    }

    [HardenedTest]
    public async Task GetProduct_NotFound(ITestWebApp app) {
        var response = await app.Get("/api/products/nonexistent");

        response.Assert.NotFound();
    }
}
```

### POST Request

```csharp
[HardenedTest]
public async Task CreateProduct_ReturnsCreated(ITestWebApp app) {
    var request = new CreateProductRequest {
        Name = "Widget",
        Price = 19.99m
    };

    var response = await app.Post(request, "/api/products");

    response.Assert.Ok();
    var product = response.Deserialize<Product>();
    Assert.Equal("Widget", product.Name);
    Assert.Equal(19.99m, product.Price);
}
```

### PUT Request

```csharp
[HardenedTest]
public async Task UpdateProduct_ReturnsUpdated(ITestWebApp app) {
    var update = new UpdateProductRequest {
        Name = "Updated Widget",
        Price = 24.99m
    };

    var response = await app.Put(update, "/api/products/prod-1");

    response.Assert.Ok();
    var product = response.Deserialize<Product>();
    Assert.Equal("Updated Widget", product.Name);
}
```

### DELETE Request

```csharp
[HardenedTest]
public async Task DeleteProduct_ReturnsOk(ITestWebApp app) {
    var response = await app.Delete("/api/products/prod-1");

    response.Assert.Ok();
}
```

---

## TestWebRequest -- Configuring Requests

The `Action<TestWebRequest>` parameter lets you configure headers and cancellation tokens for each request.

### Definition

```csharp
namespace Hardened.Web.Testing;

public class TestWebRequest {
    public IDictionary<string, StringValues> Headers { get; set; }
    public CancellationToken? Token { get; set; }
}
```

### Adding Headers

```csharp
[HardenedTest]
public async Task ProtectedRoute_WithAuth(ITestWebApp app) {
    var response = await app.Get("/api/admin/settings", req => {
        req.Headers["Authorization"] = "Bearer valid-token";
    });

    response.Assert.Ok();
}

[HardenedTest]
public async Task ProtectedRoute_WithoutAuth(ITestWebApp app) {
    var response = await app.Get("/api/admin/settings");

    response.Assert.Unauthorized();
}
```

### Custom Headers

```csharp
[HardenedTest]
public async Task CreateOrder_WithIdempotencyKey(ITestWebApp app) {
    var order = new CreateOrderRequest { CustomerId = "cust-1" };

    var response = await app.Post(order, "/api/orders", req => {
        req.Headers["X-Idempotency-Key"] = "unique-key-123";
        req.Headers["X-Tenant-Id"] = "tenant-42";
    });

    response.Assert.Ok();
}
```

### Cancellation Token

```csharp
[HardenedTest]
public async Task LongRunning_CanBeCancelled(ITestWebApp app) {
    using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(2));

    var response = await app.Get("/api/slow-endpoint", req => {
        req.Token = cts.Token;
    });
}
```

---

## TestWebResponse

The response object provides access to the status code, headers, body, and assertion helpers.

### Properties

| Property | Type | Description |
|---|---|---|
| `StatusCode` | `int` | HTTP status code (defaults to 200) |
| `Headers` | `IDictionary<string, StringValues>` | Response headers |
| `Body` | `Stream` | Raw response body stream |
| `Assert` | `IWebAssertThat` | Assertion helpers |

### Deserialize

```csharp
var response = await app.Get("/api/products");
var products = response.Deserialize<List<Product>>();
Assert.NotEmpty(products);
```

`Deserialize<T>()` uses `System.Text.Json` with web defaults. It automatically handles compressed responses (GZip and Brotli).

### Assert Helpers

The `Assert` property provides common status code assertions:

| Method | Asserts |
|---|---|
| `Assert.Ok()` | Status code is 2xx |
| `Assert.NotFound()` | Status code is 404 |
| `Assert.BadRequest()` | Status code is 400 |
| `Assert.Unauthorized()` | Status code is 401 |
| `Assert.Forbidden()` | Status code is 403 |

```csharp
[HardenedTest]
public async Task AssertionExamples(ITestWebApp app) {
    (await app.Get("/api/products")).Assert.Ok();
    (await app.Get("/api/products/missing")).Assert.NotFound();
    (await app.Post(new { }, "/api/products")).Assert.BadRequest();
    (await app.Get("/api/admin")).Assert.Unauthorized();
    (await app.Delete("/api/admin/user")).Assert.Forbidden();
}
```

---

## Combining with [Mock]

Use `[Mock]` alongside `ITestWebApp` to mock specific dependencies while testing the full HTTP pipeline:

```csharp
[HardenedTest]
public async Task CreateOrder_MocksPayment(
    ITestWebApp app,
    [Mock] IPaymentGateway payment) {

    payment.Charge(Arg.Any<decimal>(), Arg.Any<string>())
        .Returns(Task.FromResult(new PaymentResult { Success = true }));

    var response = await app.Post(
        new CreateOrderRequest { CustomerId = "cust-1", Total = 50m },
        "/api/orders");

    response.Assert.Ok();
    await payment.Received(1).Charge(50m, Arg.Any<string>());
}
```

---

## Custom HTTP Methods

Use the `Request` method for non-standard HTTP methods:

```csharp
[HardenedTest]
public async Task OptionsRequest(ITestWebApp app) {
    var response = await app.Request("OPTIONS", null, "/api/products");
    // Assert on CORS headers, etc.
}
```

---

## Accessing the Root Service Provider

`ITestWebApp` exposes `RootServiceProvider` for accessing application services directly in your test setup:

```csharp
[HardenedTest]
public async Task TestWithSetup(ITestWebApp app) {
    // Seed data using the real repository
    var repo = app.RootServiceProvider.GetRequiredService<IProductRepository>();
    await repo.Save(new Product { Id = "prod-1", Name = "Widget" });

    // Now test the API
    var response = await app.Get("/api/products/prod-1");
    response.Assert.Ok();

    var product = response.Deserialize<Product>();
    Assert.Equal("Widget", product.Name);
}
```

---

## Complete Example

```csharp
// Bootstrap.cs
using Hardened.Shared.Testing.Attributes;
using Hardened.Web.Testing;

[assembly: HardenedTestEntryPoint(typeof(Application))]
[assembly: WebTesting]
```

```csharp
// OrderApiTests.cs
using Hardened.Shared.Testing.Attributes;
using Hardened.Web.Testing;
using NSubstitute;
using Xunit;

public class OrderApiTests {
    [HardenedTest]
    public async Task CRUD_Workflow(ITestWebApp app) {
        // Create
        var createResponse = await app.Post(
            new { customerId = "cust-1" },
            "/api/orders");
        createResponse.Assert.Ok();
        var order = createResponse.Deserialize<Order>();
        Assert.NotNull(order.Id);

        // Read
        var getResponse = await app.Get($"/api/orders/{order.Id}");
        getResponse.Assert.Ok();

        // Update
        var updateResponse = await app.Put(
            new { status = "confirmed" },
            $"/api/orders/{order.Id}");
        updateResponse.Assert.Ok();

        // Delete
        var deleteResponse = await app.Delete($"/api/orders/{order.Id}");
        deleteResponse.Assert.Ok();

        // Verify deleted
        var verifyResponse = await app.Get($"/api/orders/{order.Id}");
        verifyResponse.Assert.NotFound();
    }
}
```

---

## Related Pages

- [HardenedTest](hardened-test.md) -- test attribute and DI bootstrap
- [Mocking](mocking.md) -- `[Mock]` for NSubstitute integration
- [Test Context](test-context.md) -- `ITestContext` (which `ITestWebApp` extends)
- [Routing](../web/routing.md) -- the routes being tested
