# Routing

Hardened provides compile-time HTTP routing through method-level attributes. The source generator scans for `[Get]`, `[Post]`, `[Put]`, `[Delete]`, and `[Patch]` attributes and builds a routing tree at compile time -- no runtime route registration is needed.

**Package:** `Hardened.Web.Runtime` (namespace `Hardened.Web.Runtime.Attributes`)

---

## HTTP Method Attributes

Each HTTP method has a corresponding attribute. Apply it to any public method to create a route.

### [Get]

```csharp
namespace Hardened.Web.Runtime.Attributes;

public class GetAttribute : Attribute {
    public GetAttribute(string path = "") { }

    public string Path { get; }
    public int SuccessStatus { get; set; } = 200;
    public int ValidationErrorStatus { get; set; } = 400;
    public int NullReturnStatus { get; set; } = 404;
    public int ErrorStatus { get; set; } = 500;
}
```

### [Post]

```csharp
public class PostAttribute : Attribute {
    public PostAttribute(string path = "") { }
    public string Path { get; }
}
```

### [Put]

```csharp
public class PutAttribute : Attribute {
    public PutAttribute(string path = "") { }
    public string Path { get; }
}
```

### [Delete]

```csharp
public class DeleteAttribute : Attribute {
    public DeleteAttribute(string path = "") { }
    public string Path { get; }
}
```

### [Patch]

```csharp
public class PatchAttribute : Attribute {
    public PatchAttribute(string path = "") { }
    public string Path { get; }
}
```

---

## Basic Routing

```csharp
using Hardened.Web.Runtime.Attributes;

public class ProductController {
    [Get("/api/products")]
    public Task<IReadOnlyList<Product>> ListProducts(
        IProductRepository repo) {
        return repo.GetAll();
    }

    [Get("/api/products/{id}")]
    public Task<Product?> GetProduct(
        string id,
        IProductRepository repo) {
        return repo.GetById(id);
    }

    [Post("/api/products")]
    public async Task<Product> CreateProduct(
        CreateProductRequest request,
        IProductService service) {
        return await service.Create(request);
    }

    [Put("/api/products/{id}")]
    public async Task<Product> UpdateProduct(
        string id,
        UpdateProductRequest request,
        IProductService service) {
        return await service.Update(id, request);
    }

    [Delete("/api/products/{id}")]
    public async Task DeleteProduct(
        string id,
        IProductService service) {
        await service.Delete(id);
    }

    [Patch("/api/products/{id}")]
    public async Task<Product> PatchProduct(
        string id,
        PatchProductRequest request,
        IProductService service) {
        return await service.Patch(id, request);
    }
}
```

!!! note
    Unlike ASP.NET Core, there is no `ControllerBase` to inherit from and no `[ApiController]` attribute needed. Any class with methods decorated with HTTP attributes becomes a route handler.

---

## Path Tokens

Use `{paramName}` syntax in route paths to define dynamic segments. Parameters with matching names are automatically bound from the URL.

```csharp
[Get("/api/tenants/{tenantId}/orders/{orderId}")]
public Task<Order?> GetOrder(
    string tenantId,
    string orderId,
    IOrderRepository repo) {
    return repo.GetOrder(tenantId, orderId);
}
```

A request to `/api/tenants/t-1/orders/o-42` binds `tenantId` to `"t-1"` and `orderId` to `"o-42"`.

Path tokens support type conversion:

```csharp
[Get("/api/items/{itemId}")]
public Task<Item?> GetItem(int itemId, IItemRepository repo) {
    // itemId is parsed as an integer from the path
    return repo.GetById(itemId);
}
```

See [Parameter Binding](../requests/parameter-binding.md) for the full set of binding sources.

---

## [BasePath] -- Route Prefix

The `[BasePath]` attribute adds a prefix to all routes in a class. It can also be applied at the assembly level to prefix all routes in the project.

### Definition

```csharp
namespace Hardened.Web.Runtime.Attributes;

[AttributeUsage(
    AttributeTargets.Class | AttributeTargets.Assembly,
    AllowMultiple = false,
    Inherited = false)]
public class BasePathAttribute : Attribute {
    public BasePathAttribute(string path) { }
    public string Path { get; }
}
```

### Class-Level Prefix

```csharp
[BasePath("/api/v2")]
public class OrderController {
    [Get("/orders")]           // Resolves to /api/v2/orders
    public Task<IReadOnlyList<Order>> List(IOrderRepository repo) {
        return repo.GetAll();
    }

    [Get("/orders/{id}")]      // Resolves to /api/v2/orders/{id}
    public Task<Order?> Get(string id, IOrderRepository repo) {
        return repo.GetById(id);
    }

    [Post("/orders")]          // Resolves to /api/v2/orders
    public Task<Order> Create(CreateOrderRequest req, IOrderService svc) {
        return svc.Create(req);
    }
}
```

### Assembly-Level Prefix

Apply `[BasePath]` at the assembly level to prefix every route in the project:

```csharp
// In any file, typically AssemblyInfo.cs or Program.cs
using Hardened.Web.Runtime.Attributes;

[assembly: BasePath("/api/v1")]
```

Now all routes in the assembly are prefixed:

```csharp
public class UserController {
    [Get("/users")]            // Resolves to /api/v1/users
    public Task<IReadOnlyList<User>> List(IUserRepository repo) {
        return repo.GetAll();
    }
}
```

!!! tip
    When both assembly-level and class-level `[BasePath]` are present, the class-level prefix takes precedence for that class. They do not stack.

---

## Status Code Customization

The `[Get]` attribute exposes properties for customizing HTTP status codes returned in different scenarios. Other HTTP method attributes (`[Post]`, `[Put]`, etc.) share the same path pattern but return 200 by default.

### Available Properties on [Get]

| Property | Default | Description |
|---|---|---|
| `SuccessStatus` | 200 | Status code when the handler returns a non-null value |
| `NullReturnStatus` | 404 | Status code when the handler returns `null` |
| `ValidationErrorStatus` | 400 | Status code for validation errors |
| `ErrorStatus` | 500 | Status code for unhandled exceptions |

### Usage

```csharp
[Get("/api/orders/{id}", NullReturnStatus = 204)]
public Task<Order?> GetOrder(string id, IOrderRepository repo) {
    // Returns 200 with the order, or 204 (No Content) if null
    return repo.GetById(id);
}

[Get("/api/health", SuccessStatus = 200, ErrorStatus = 503)]
public HealthStatus CheckHealth(IHealthService health) {
    return health.Check();
}
```

---

## Route Organization

### Multiple Controllers

You can split routes across as many classes as you want. The source generator discovers all methods with HTTP attributes across the entire assembly:

```csharp
public class OrderController {
    [Get("/api/orders")]
    public Task<IReadOnlyList<Order>> ListOrders(IOrderRepository repo)
        => repo.GetAll();

    [Post("/api/orders")]
    public Task<Order> CreateOrder(CreateOrderRequest req, IOrderService svc)
        => svc.Create(req);
}

public class CustomerController {
    [Get("/api/customers")]
    public Task<IReadOnlyList<Customer>> ListCustomers(ICustomerRepository repo)
        => repo.GetAll();

    [Get("/api/customers/{id}")]
    public Task<Customer?> GetCustomer(string id, ICustomerRepository repo)
        => repo.GetById(id);
}
```

### Nested Resources

Model nested resources naturally with path tokens:

```csharp
[BasePath("/api")]
public class OrderItemController {
    [Get("/orders/{orderId}/items")]
    public Task<IReadOnlyList<OrderItem>> ListItems(
        string orderId, IOrderItemRepository repo)
        => repo.GetByOrder(orderId);

    [Post("/orders/{orderId}/items")]
    public Task<OrderItem> AddItem(
        string orderId,
        AddItemRequest request,
        IOrderItemService service)
        => service.Add(orderId, request);

    [Delete("/orders/{orderId}/items/{itemId}")]
    public Task RemoveItem(
        string orderId,
        string itemId,
        IOrderItemService service)
        => service.Remove(orderId, itemId);
}
```

---

## How It Works -- Compile-Time Routing

The source generator builds a routing tree at compile time:

1. It scans for all methods with `[Get]`, `[Post]`, `[Put]`, `[Delete]`, or `[Patch]`
2. It resolves `[BasePath]` prefixes
3. It generates a trie-based routing table as C# code
4. At runtime, route matching is a direct tree traversal -- no regex, no dictionary lookup

This produces:

- **Zero startup cost** -- routes are compiled into the binary
- **Deterministic matching** -- route conflicts are caught at build time
- **Small memory footprint** -- no runtime route collections

---

## Related Pages

- [Parameter Binding](../requests/parameter-binding.md) -- how path tokens, query strings, and headers bind to parameters
- [ASP.NET Core Integration](aspnetcore-integration.md) -- connecting routes to ASP.NET Core
- [Filters](../requests/filters.md) -- middleware that runs before/after route handlers
