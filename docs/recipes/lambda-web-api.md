# Recipe: Lambda Web API

Build a serverless REST API that runs behind API Gateway using Hardened's Lambda web runtime. The same controller code works locally during development and in Lambda when deployed.

**What you will build:**

- A REST API running as an AWS Lambda function behind API Gateway
- Controllers with `[Get]`, `[Post]`, `[Put]`, `[Delete]` routes
- Local development harness for testing without deploying
- Unit tests using `ITestWebApp`

---

## Prerequisites

- [.NET 8 SDK](https://dotnet.microsoft.com/download/dotnet/8.0) or later
- [NuGet configured for GitHub Packages](../getting-started/nuget-setup.md)

---

## Project Setup

```bash
dotnet new classlib -n ProductApi
cd ProductApi
dotnet add package Hardened.Amz.Web.Lambda.SourceGenerator --prerelease
```

---

## Complete Code

### Models

```csharp title="Models/Product.cs"
public class Product
{
    public string Id { get; set; } = string.Empty;
    public string Name { get; set; } = string.Empty;
    public string Category { get; set; } = string.Empty;
    public decimal Price { get; set; }
    public int StockCount { get; set; }
}
```

```csharp title="Models/CreateProductRequest.cs"
public class CreateProductRequest
{
    public string Name { get; set; } = string.Empty;
    public string Category { get; set; } = string.Empty;
    public decimal Price { get; set; }
    public int StockCount { get; set; }
}
```

### Service

```csharp title="Services/IProductService.cs"
public interface IProductService
{
    List<Product> GetAll(string? category = null);
    Product? GetById(string id);
    Product Create(CreateProductRequest request);
    bool Delete(string id);
}
```

```csharp title="Services/ProductService.cs"
using Hardened.Shared.Runtime.Attributes;

[Expose]
[Singleton]
public class ProductService : IProductService
{
    private readonly List<Product> _products = new();

    public List<Product> GetAll(string? category = null)
    {
        if (string.IsNullOrEmpty(category))
            return _products.ToList();

        return _products
            .Where(p => p.Category.Equals(
                category, StringComparison.OrdinalIgnoreCase))
            .ToList();
    }

    public Product? GetById(string id)
    {
        return _products.FirstOrDefault(p => p.Id == id);
    }

    public Product Create(CreateProductRequest request)
    {
        var product = new Product
        {
            Id = Guid.NewGuid().ToString("N")[..8],
            Name = request.Name,
            Category = request.Category,
            Price = request.Price,
            StockCount = request.StockCount
        };

        _products.Add(product);
        return product;
    }

    public bool Delete(string id)
    {
        var product = _products.FirstOrDefault(p => p.Id == id);
        if (product == null) return false;

        _products.Remove(product);
        return true;
    }
}
```

### Controllers

```csharp title="Controllers/ProductController.cs"
using Hardened.Web.Runtime.Attributes;

[BasePath("/api/products")]
public class ProductController
{
    private readonly IProductService _productService;

    public ProductController(IProductService productService)
    {
        _productService = productService;
    }

    [Get]
    public List<Product> GetAll([FromQueryString] string? category = null)
    {
        return _productService.GetAll(category);
    }

    [Get("{id}", NullReturnStatus = 404)]
    public Product? GetById(string id)
    {
        return _productService.GetById(id);
    }

    [Post(SuccessStatus = 201)]
    public Product Create([FromBody] CreateProductRequest request)
    {
        return _productService.Create(request);
    }

    [Delete("{id}", SuccessStatus = 204)]
    public bool Delete(string id)
    {
        return _productService.Delete(id);
    }
}
```

```csharp title="Controllers/HealthController.cs"
using Hardened.Web.Runtime.Attributes;

public class HealthController
{
    [Get("/health")]
    public object Health()
    {
        return new
        {
            Status = "Healthy",
            Timestamp = DateTime.UtcNow
        };
    }
}
```

### Application Module

```csharp title="Application.cs"
using Hardened.Shared.Runtime.Attributes;
using Hardened.Amz.Web.Lambda.Runtime;

[HardenedModule]
[LambdaWebApplication(Version = ProxyIntegrationType.HttpApiV2)]
public partial class Application { }
```

!!! note
    `ProxyIntegrationType.HttpApiV2` configures the Lambda to work with API Gateway HTTP APIs (v2 payload format). Use `ProxyIntegrationType.RestApi` if you are using a REST API (v1 payload format) instead.

---

## Explanation

### Lambda Web Application

The `[LambdaWebApplication]` attribute replaces `[AspNetCoreRuntime.Module]`. It configures the application to:

1. Accept API Gateway proxy events as input
2. Route them through Hardened's request pipeline
3. Return API Gateway proxy responses

Your controllers, services, and DI registrations are identical whether you deploy to Lambda or run locally on ASP.NET Core. Only the `Application.cs` module attribute changes.

### Query String Parameters

The `[FromQueryString]` attribute binds a parameter from the URL query string. In this example, `GET /api/products?category=Electronics` passes `"Electronics"` to the `category` parameter. Optional parameters use nullable types with default values.

### Proxy Integration Types

| Type | API Gateway Version | Payload Format |
|---|---|---|
| `ProxyIntegrationType.HttpApiV2` | HTTP API (v2) | Simplified event format |
| `ProxyIntegrationType.RestApi` | REST API (v1) | Full proxy integration event |

---

## Local Development

For local development, use the Lambda web harness to run your API as a local HTTP server without deploying to AWS.

Create a separate console project for the local harness:

```bash
cd ..
dotnet new console -n ProductApi.Harness
cd ProductApi.Harness
dotnet add reference ../ProductApi/ProductApi.csproj
dotnet add package Hardened.Amz.Web.Lambda.Harness --prerelease
```

```csharp title="Program.cs"
using Hardened.Amz.Web.Lambda.Harness;

var harness = new LambdaWebHarness<Application>();
await harness.RunAsync(args);
```

Run the local server:

```bash
dotnet run --project ProductApi.Harness
```

This starts a local HTTP server that mimics API Gateway, routing requests through your Lambda handler. You can use `curl` or a browser to test:

```bash
# Create a product
curl -X POST http://localhost:5000/api/products \
  -H "Content-Type: application/json" \
  -d '{"name": "Widget", "category": "Electronics", "price": 29.99, "stockCount": 100}'

# List all products
curl http://localhost:5000/api/products

# Filter by category
curl "http://localhost:5000/api/products?category=Electronics"

# Health check
curl http://localhost:5000/health
```

!!! tip
    The harness project is only used during development. It is not deployed to AWS. Your `ProductApi` project is the deployment artifact.

---

## Testing

Create a test project:

```bash
cd ..
dotnet new xunit -n ProductApi.Tests
cd ProductApi.Tests
dotnet add reference ../ProductApi/ProductApi.csproj
dotnet add package Hardened.Web.Testing --prerelease
```

### Bootstrap

```csharp title="Bootstrap.cs"
using Hardened.Web.Testing;
using Hardened.Shared.Runtime.Attributes;

[assembly: WebTesting]
[assembly: HardenedTestEntryPoint(typeof(Application))]
```

### Tests

```csharp title="ProductControllerTests.cs"
using Hardened.Shared.Runtime.Attributes;
using Hardened.Web.Testing;

public class ProductControllerTests
{
    [HardenedTest]
    public async Task CreateProduct_Returns201(ITestWebApp testWebApp)
    {
        var request = new CreateProductRequest
        {
            Name = "Widget",
            Category = "Electronics",
            Price = 29.99m,
            StockCount = 100
        };

        var response = await testWebApp.Post("/api/products", request);

        response.Assert.StatusCode(201);

        var product = response.Deserialize<Product>();
        Assert.Equal("Widget", product.Name);
        Assert.Equal("Electronics", product.Category);
        Assert.NotEmpty(product.Id);
    }

    [HardenedTest]
    public async Task GetProducts_FilterByCategory(ITestWebApp testWebApp)
    {
        // Create products in different categories
        await testWebApp.Post("/api/products", new CreateProductRequest
        {
            Name = "Widget",
            Category = "Electronics",
            Price = 29.99m,
            StockCount = 50
        });

        await testWebApp.Post("/api/products", new CreateProductRequest
        {
            Name = "Hammer",
            Category = "Tools",
            Price = 15.00m,
            StockCount = 200
        });

        // Filter by category
        var response = await testWebApp.Get(
            "/api/products?category=Electronics");

        response.Assert.Ok();

        var products = response.Deserialize<List<Product>>();
        Assert.All(products, p => Assert.Equal("Electronics", p.Category));
    }

    [HardenedTest]
    public async Task GetProduct_NotFound_Returns404(ITestWebApp testWebApp)
    {
        var response = await testWebApp.Get("/api/products/nonexistent");

        response.Assert.StatusCode(404);
    }

    [HardenedTest]
    public async Task HealthCheck_ReturnsHealthy(ITestWebApp testWebApp)
    {
        var response = await testWebApp.Get("/health");

        response.Assert.Ok();
    }
}
```

Run the tests:

```bash
dotnet test
```

---

## Deployment

When deploying to AWS, the Lambda function handler is generated by the source generator. Your CDK or SAM template points to the generated handler:

```yaml
# Example SAM template snippet
Resources:
  ProductApiFunction:
    Type: AWS::Serverless::Function
    Properties:
      Handler: ProductApi
      Runtime: dotnet8
      MemorySize: 256
      Timeout: 30
      Events:
        Api:
          Type: HttpApi
          Properties:
            ApiId: !Ref HttpApi
```

!!! note
    The exact handler string depends on your project name and source generator output. Check the generated code in `obj/Debug/net8.0/generated/` for the exact entry point.

---

## Next Steps

- [Web Runtime](../aws/lambda/web-runtime.md) -- advanced Lambda web configuration
- [Web API CRUD](web-api-crud.md) -- full CRUD patterns for controllers
- [Custom Execution Filter](custom-execution-filter.md) -- add middleware to your Lambda web API
