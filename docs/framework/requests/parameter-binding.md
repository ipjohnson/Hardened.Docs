# Parameter Binding

Hardened automatically binds handler method parameters from various sources: the request body, DI container, HTTP headers, query strings, and path tokens. The source generator analyzes your method signatures at compile time and emits the binding code.

**Packages:** `Hardened.Requests.Abstract` (namespace `Hardened.Requests.Abstract.Attributes`), `Hardened.Web.Runtime` (namespace `Hardened.Web.Runtime.Attributes`)

---

## Binding Sources

Parameters are bound from these sources, in order of precedence:

| Source | Attribute | Example |
|---|---|---|
| Request body | `[FromBody]` | `[FromBody] CreateOrderRequest request` |
| DI container | `[FromServices]` or auto-detected | `IOrderService orderService` |
| HTTP header | `[FromHeader]` | `[FromHeader("X-Request-Id")] string requestId` |
| Query string | `[FromQueryString]` | `[FromQueryString] int page` |
| Path token | *(automatic)* | `string id` when route is `/{id}` |

---

## [FromBody] -- Request Body

Deserializes the request body into the parameter type. The framework uses the request's `Content-Type` header to select the appropriate deserializer (typically JSON).

### Definition

```csharp
namespace Hardened.Requests.Abstract.Attributes;

public class FromBodyAttribute : Attribute { }
```

### Usage

```csharp
using Hardened.Requests.Abstract.Attributes;
using Hardened.Web.Runtime.Attributes;

[Post("/api/orders")]
public async Task<Order> CreateOrder(
    [FromBody] CreateOrderRequest request,
    IOrderService orderService) {
    return await orderService.Create(request);
}
```

The incoming JSON body is deserialized into a `CreateOrderRequest` object:

```json
{
    "customerId": "cust-123",
    "items": [
        { "productId": "prod-456", "quantity": 2 }
    ]
}
```

!!! tip
    For `[Post]`, `[Put]`, and `[Patch]` methods, if there is exactly one non-service, non-path-token parameter, `[FromBody]` is inferred automatically. You can still add it explicitly for clarity.

---

## [FromServices] -- Dependency Injection

Resolves the parameter from the scoped DI container for the current request.

### Definition

```csharp
namespace Hardened.Requests.Abstract.Attributes;

public class FromServicesAttribute : Attribute { }
```

### Usage

```csharp
[Get("/api/orders/{id}")]
public async Task<Order?> GetOrder(
    string id,
    [FromServices] IOrderRepository repository) {
    return await repository.GetById(id);
}
```

### Auto-Detection

Interface parameters are automatically resolved from DI without needing `[FromServices]`. The following is equivalent:

```csharp
[Get("/api/orders/{id}")]
public async Task<Order?> GetOrder(
    string id,
    IOrderRepository repository) {
    return await repository.GetById(id);
}
```

The source generator detects that `IOrderRepository` is an interface and resolves it from the DI container automatically.

!!! note
    Auto-detection works for interface types. If you need to inject a concrete class, use `[FromServices]` explicitly.

---

## [FromHeader] -- HTTP Headers

Binds a parameter from an HTTP request header.

### Definition

```csharp
namespace Hardened.Web.Runtime.Attributes;

public class FromHeaderAttribute : Attribute {
    public FromHeaderAttribute(string? name = null) {
        Name = name;
    }

    public string? Name { get; }
}
```

### Usage

```csharp
[Get("/api/protected")]
public object GetProtectedData(
    [FromHeader("Authorization")] string authToken,
    [FromHeader("X-Request-Id")] string? requestId,
    IProtectedService service) {
    return service.GetData(authToken);
}
```

When `name` is omitted, the parameter name is used as the header name:

```csharp
[Get("/api/data")]
public object GetData(
    [FromHeader] string authorization) {
    // Reads from the "authorization" header
}
```

---

## [FromQueryString] -- Query String Parameters

Binds a parameter from the URL query string.

### Definition

```csharp
namespace Hardened.Web.Runtime.Attributes;

public class FromQueryStringAttribute : Attribute {
    public FromQueryStringAttribute(string? name = null) {
        Name = name;
    }

    public string? Name { get; }
}
```

### Usage

```csharp
[Get("/api/orders")]
public async Task<PagedResult<Order>> ListOrders(
    [FromQueryString] int page,
    [FromQueryString] int pageSize,
    [FromQueryString("status")] string? orderStatus,
    IOrderRepository repository) {
    return await repository.Search(page, pageSize, orderStatus);
}
```

A request to `/api/orders?page=1&pageSize=20&status=pending` binds `page` to `1`, `pageSize` to `20`, and `orderStatus` to `"pending"`.

When `name` is omitted, the parameter name is used as the query string key:

```csharp
[Get("/api/search")]
public Task<SearchResult> Search(
    [FromQueryString] string q,
    [FromQueryString] int limit) {
    // Reads from ?q=...&limit=...
}
```

---

## Path Tokens -- Automatic Binding

Parameters that match `{paramName}` segments in the route path are automatically bound from the URL. No attribute is needed.

### Usage

```csharp
[Get("/api/orders/{orderId}/items/{itemId}")]
public async Task<OrderItem?> GetOrderItem(
    string orderId,
    string itemId,
    IOrderRepository repository) {
    return await repository.GetItem(orderId, itemId);
}
```

A request to `/api/orders/ord-123/items/item-456` binds `orderId` to `"ord-123"` and `itemId` to `"item-456"`.

Path tokens support type conversion:

```csharp
[Get("/api/users/{userId}")]
public async Task<User?> GetUser(int userId, IUserRepository repo) {
    // userId is automatically parsed from the path as an int
    return await repo.GetById(userId);
}
```

---

## ICustomBindingAttribute -- Custom Binding

For advanced scenarios, implement `ICustomBindingAttribute` to create your own binding logic.

### Definition

```csharp
namespace Hardened.Requests.Abstract.Attributes;

public interface ICustomBindingAttribute {
    ValueTask<T> BindValue<T>(
        IExecutionContext context,
        IExecutionRequestParameter parameter);
}
```

### Example -- Binding from a Custom Header Format

```csharp
using Hardened.Requests.Abstract.Attributes;
using Hardened.Requests.Abstract.Execution;

public class FromBearerTokenAttribute : Attribute, ICustomBindingAttribute {
    public async ValueTask<T> BindValue<T>(
        IExecutionContext context,
        IExecutionRequestParameter parameter) {
        var authHeader = context.Request.Headers
            .GetValueOrDefault("Authorization")
            .ToString();

        if (authHeader?.StartsWith("Bearer ") == true) {
            var token = authHeader.Substring(7);
            var tokenService = context.RequestServices
                .GetRequiredService<ITokenService>();

            var claims = await tokenService.Validate(token);
            return (T)(object)claims;
        }

        throw new UnauthorizedException("Missing or invalid bearer token");
    }
}
```

Apply the custom binding attribute to a parameter:

```csharp
[Get("/api/profile")]
public Task<UserProfile> GetProfile(
    [FromBearerToken] UserClaims claims,
    IUserService userService) {
    return userService.GetProfile(claims.UserId);
}
```

---

## Binding Order and Resolution

When the source generator encounters a handler method, it resolves each parameter using these rules in order:

1. **Custom binding attribute** -- If the parameter has an `ICustomBindingAttribute`, use it
2. **`[FromBody]`** -- Deserialize from the request body
3. **`[FromHeader]`** -- Read from the specified header
4. **`[FromQueryString]`** -- Read from the query string
5. **`[FromServices]`** -- Resolve from the DI container
6. **Path token match** -- If the parameter name matches a `{token}` in the route path, bind from the path
7. **Interface auto-detection** -- If the parameter type is an interface, resolve from DI
8. **`IExecutionContext`** -- If the parameter type is `IExecutionContext`, inject the current context

---

## Complete Example

```csharp
public class OrderController {
    [Post("/api/tenants/{tenantId}/orders")]
    public async Task<Order> CreateOrder(
        string tenantId,                              // Path token
        [FromBody] CreateOrderRequest request,        // Request body
        [FromHeader("X-Idempotency-Key")] string key, // Header
        IOrderService orderService,                   // Auto-detected DI
        IExecutionContext context) {                   // Execution context

        var order = await orderService.Create(
            tenantId, request, key);

        context.Response.Status = 201;
        return order;
    }

    [Get("/api/tenants/{tenantId}/orders")]
    public Task<PagedResult<Order>> ListOrders(
        string tenantId,                               // Path token
        [FromQueryString] int page,                    // Query string
        [FromQueryString] int pageSize,                // Query string
        [FromQueryString("status")] string? status,    // Query string (custom name)
        IOrderRepository repository) {                 // Auto-detected DI

        return repository.Search(tenantId, page, pageSize, status);
    }
}
```

---

## Related Pages

- [Execution Model](execution-model.md) -- `IExecutionContext` and the request/response objects
- [Filters](filters.md) -- filters run before parameter binding
- [Routing](../web/routing.md) -- how routes define path token segments
