# Dependency Injection Best Practices

Hardened's compile-time DI system generates all service registrations at build time. This eliminates runtime reflection but also means your attribute choices have direct, visible consequences in the generated code. This page covers opinionated recommendations for getting the most out of `[Expose]`, lifecycle attributes, and related patterns.

---

## Choosing the Right Lifecycle

The lifecycle you choose affects memory usage, thread safety requirements, and behavior in serverless environments. Choose deliberately.

### Transient (Default)

A new instance is created every time the service is requested. This is the default when you use `[Expose]` without a lifecycle attribute.

```csharp
[Expose(typeof(IOrderValidator))]
public class OrderValidator : IOrderValidator { }
```

**Use transient when:**

- The service is stateless and lightweight to construct
- The service holds no shared resources
- You want the simplest possible lifecycle with no concurrency concerns

**Avoid transient when:**

- The service is expensive to create (database connections, HTTP clients)
- You need shared state across multiple consumers within the same request

### Singleton

A single instance is shared for the lifetime of the application. In Lambda, this means the lifetime of the container (across warm invocations).

```csharp
[Expose(typeof(ICacheService))]
[Singleton]
public class InMemoryCacheService : ICacheService { }
```

**Use singleton when:**

- The service manages shared state (caches, connection pools)
- The service wraps an expensive-to-create resource (AWS SDK clients, HTTP clients)
- The service is stateless and you want to avoid repeated allocations
- You are running in Lambda and want to reuse the instance across invocations

**Avoid singleton when:**

- The service holds per-request state
- The service is not thread-safe

!!! warning "Thread Safety Requirement"
    Singleton services must be thread-safe. In web applications, multiple requests execute concurrently and share the same singleton instance. Even in Lambda (which processes one request at a time per container), design singletons to be thread-safe so they behave correctly if the hosting model changes.

### Scoped

A single instance is created per scope. In web applications, each HTTP request is a scope. In Lambda, each invocation is a scope.

```csharp
[Expose(typeof(IUserContext))]
[Scoped]
public class UserContext : IUserContext
{
    public string? UserId { get; set; }
    public string? TenantId { get; set; }
}
```

**Use scoped when:**

- The service holds per-request state (user context, correlation IDs, unit-of-work)
- You want a single instance shared across all services within one request
- The service is expensive to create but should not be shared across requests

### Decision Matrix

| Question | Transient | Scoped | Singleton |
|---|---|---|---|
| Holds per-request state? | No | **Yes** | No |
| Expensive to construct? | No | Maybe | **Yes** |
| Wraps shared resource? | No | No | **Yes** |
| Needs thread safety? | No | No | **Yes** |
| Default for most services? | **Yes** | No | No |

---

## The Captive Dependency Problem

Never inject a shorter-lived service into a longer-lived one. This is the most common DI mistake and it leads to subtle, hard-to-diagnose bugs.

```csharp
// BAD: Scoped service injected into singleton
[Expose(typeof(INotificationService))]
[Singleton]
public class NotificationService : INotificationService
{
    private readonly IUserContext _userContext; // Scoped -- captured!

    public NotificationService(IUserContext userContext)
    {
        _userContext = userContext; // This instance is frozen for the app's lifetime
    }
}
```

The `IUserContext` instance injected into the singleton was created in a specific scope. After that scope ends, the captured instance holds stale data and may reference disposed resources.

### The Fix

If a singleton needs access to a scoped or transient service, inject `IServiceProvider` and resolve it per-operation:

```csharp
[Expose(typeof(INotificationService))]
[Singleton]
public class NotificationService : INotificationService
{
    private readonly IServiceProvider _serviceProvider;

    public NotificationService(IServiceProvider serviceProvider)
    {
        _serviceProvider = serviceProvider;
    }

    public async Task Notify(string message)
    {
        // Resolve from the current scope at call time
        var userContext = _serviceProvider.GetRequiredService<IUserContext>();
        // ...
    }
}
```

!!! tip "Preferred Alternative"
    Better yet, restructure so the scoped dependency is passed as a method parameter or resolved in the calling code. Injecting `IServiceProvider` is a code smell that signals a design issue -- use it as a last resort.

---

## Interface Design for DI

### One Interface Per Responsibility

Design narrow, focused interfaces rather than wide ones. This makes mocking easier and dependencies explicit:

```csharp
// GOOD: Focused interfaces
public interface IOrderReader
{
    Task<Order?> GetById(string id);
    Task<IReadOnlyList<Order>> GetByCustomer(string customerId);
}

public interface IOrderWriter
{
    Task Save(Order order);
    Task Delete(string id);
}

[Expose(typeof(IOrderReader), typeof(IOrderWriter))]
[Singleton]
public class DynamoDbOrderRepository : IOrderReader, IOrderWriter { }
```

```csharp
// AVOID: Wide interface forces consumers to depend on methods they don't use
public interface IOrderRepository
{
    Task<Order?> GetById(string id);
    Task<IReadOnlyList<Order>> GetByCustomer(string customerId);
    Task Save(Order order);
    Task Delete(string id);
    Task<int> Count();
    Task BulkImport(IEnumerable<Order> orders);
}
```

### Use Interfaces for All Exposed Services

Always register services against interfaces, not concrete types. This enables mocking in tests and makes implementations swappable:

```csharp
// GOOD: Register against interface
[Expose(typeof(IOrderService))]
public class OrderService : IOrderService { }

// AVOID: Register as concrete type
[Expose]
public class OrderService { }
```

!!! note "When Concrete Registration Is Acceptable"
    Registering as a concrete type is fine for internal helper classes that are never injected into other services and never need to be mocked. But as a rule, if a class is injected into another class, expose it via an interface.

---

## Controlling Registration with ForServices

By default, `[Expose]` with no arguments registers the class for all implemented interfaces. When a class implements multiple interfaces and you only want to expose some of them, use `ForServices`:

```csharp
public interface IOrderService { }
public interface IDisposable { }
public interface IHealthCheck { }

// Without ForServices: registered for IOrderService, IDisposable, and IHealthCheck
[Expose]
public class OrderService : IOrderService, IDisposable, IHealthCheck { }

// With ForServices: registered only for IOrderService
[Expose(typeof(IOrderService))]
public class OrderService : IOrderService, IDisposable, IHealthCheck { }
```

**Use explicit `ForServices` when:**

- The class implements infrastructure interfaces (`IDisposable`, `IAsyncDisposable`) that should not be DI-resolvable
- You want to limit the surface area of what consumers can depend on
- The class implements multiple domain interfaces but only some should be publicly registered

---

## Environment-Specific Registrations

Use `[ForEnvironment]` to swap implementations between environments without conditional logic:

```csharp
public interface IPaymentGateway
{
    Task<PaymentResult> Charge(decimal amount, string token);
}

[Expose(typeof(IPaymentGateway))]
[ForEnvironment("Production")]
[Singleton]
public class StripePaymentGateway : IPaymentGateway
{
    public Task<PaymentResult> Charge(decimal amount, string token)
    {
        // Real Stripe API calls
    }
}

[Expose(typeof(IPaymentGateway))]
[ForEnvironment("Development")]
[ForEnvironment("Test")]
public class FakePaymentGateway : IPaymentGateway
{
    public Task<PaymentResult> Charge(decimal amount, string token)
    {
        return Task.FromResult(new PaymentResult { Success = true });
    }
}
```

### Guidelines for ForEnvironment

1. **Always have a registration for every environment.** If `IPaymentGateway` is registered for Production but not Development, the DI container will throw at startup in Development.

2. **Use `[ForEnvironment]` for infrastructure, not business logic.** Swapping a real Stripe client for a fake is appropriate. Branching business rules by environment is not -- use configuration for that.

3. **Combine with `Try = true` for defaults:**

    ```csharp
    // Default: used unless overridden
    [Expose(typeof(IEmailSender), Try = true)]
    public class ConsoleEmailSender : IEmailSender { }

    // Production override
    [Expose(typeof(IEmailSender))]
    [ForEnvironment("Production")]
    [Singleton]
    public class SesEmailSender : IEmailSender { }
    ```

---

## Optional Registrations with Try

The `Try` property registers a service only if nothing else has already registered for the same service type. This is the foundation for extensible library modules.

```csharp
// In a library module: provides a default
[Expose(typeof(ISerializer), Try = true)]
[Singleton]
public class JsonSerializer : ISerializer { }

// In the host project: overrides the default
[Expose(typeof(ISerializer))]
[Singleton]
public class CustomSerializer : ISerializer { }
```

### When to Use Try

| Scenario | Use `Try`? |
|---|---|
| Library providing a default implementation | Yes |
| Host project registering its primary service | No |
| Fallback for optional features | Yes |
| Overriding a library default | No |

!!! tip "Library Module Pattern"
    If you are building a reusable library module with `Hardened.Library.SourceGenerator`, use `Try = true` on all default implementations. This ensures consumers can override any service without registration conflicts.

---

## Avoiding Circular Dependencies

Circular dependencies cause runtime exceptions during service resolution. Since Hardened generates standard `IServiceCollection` registrations, the same rules apply.

### Detecting Circular Dependencies

If `ServiceA` depends on `ServiceB` and `ServiceB` depends on `ServiceA`, resolution will fail:

```csharp
// CIRCULAR: Will throw at runtime
[Expose(typeof(IServiceA))]
public class ServiceA : IServiceA
{
    public ServiceA(IServiceB serviceB) { }
}

[Expose(typeof(IServiceB))]
public class ServiceB : IServiceB
{
    public ServiceB(IServiceA serviceA) { }
}
```

### Breaking Circular Dependencies

1. **Extract a shared dependency.** If `A` and `B` depend on each other, they likely share a concept that should be its own service:

    ```csharp
    // Extract the shared concern into a third service
    [Expose(typeof(ISharedLogic))]
    public class SharedLogic : ISharedLogic { }

    [Expose(typeof(IServiceA))]
    public class ServiceA : IServiceA
    {
        public ServiceA(ISharedLogic shared) { }
    }

    [Expose(typeof(IServiceB))]
    public class ServiceB : IServiceB
    {
        public ServiceB(ISharedLogic shared) { }
    }
    ```

2. **Use lazy resolution.** As a last resort, inject `IServiceProvider` and resolve one of the services lazily:

    ```csharp
    [Expose(typeof(IServiceA))]
    public class ServiceA : IServiceA
    {
        private readonly IServiceProvider _provider;

        public ServiceA(IServiceProvider provider)
        {
            _provider = provider;
        }

        public void DoWork()
        {
            var b = _provider.GetRequiredService<IServiceB>();
            // ...
        }
    }
    ```

!!! warning "Circular Dependencies Are a Design Smell"
    If two services depend on each other, re-examine the design. In almost every case, the circular dependency reveals a missing abstraction or a responsibility that should be extracted.

---

## Configuration Model Design

`[ConfigurationModel]` interfaces define typed configuration that the source generator implements at compile time. Design them carefully -- they are the contract between your code and its runtime environment.

### One Model Per Concern

Group related configuration together but do not create a single monolithic config interface:

```csharp
// GOOD: Focused configuration models
[ConfigurationModel]
public interface IDatabaseConfig
{
    [FromEnvironmentVariable("DB_TABLE_NAME")]
    string TableName { get; }

    [FromEnvironmentVariable("DB_REGION")]
    string Region { get; }
}

[ConfigurationModel]
public interface INotificationConfig
{
    [FromEnvironmentVariable("SQS_QUEUE_URL")]
    string QueueUrl { get; }

    [FromEnvironmentVariable("NOTIFICATION_TIMEOUT_SECONDS")]
    int TimeoutSeconds { get; }
}
```

```csharp
// AVOID: One giant config interface for everything
[ConfigurationModel]
public interface IAppConfig
{
    string DbTableName { get; }
    string DbRegion { get; }
    string SqsQueueUrl { get; }
    int NotificationTimeout { get; }
    string ApiBaseUrl { get; }
    // ... keeps growing
}
```

### Use Explicit Environment Variable Names

Always use `[FromEnvironmentVariable]` with an explicit variable name. This makes the deployment contract visible in code:

```csharp
[ConfigurationModel]
public interface IApiConfig
{
    [FromEnvironmentVariable("API_BASE_URL")]
    string BaseUrl { get; }

    [FromEnvironmentVariable("API_TIMEOUT_MS")]
    int TimeoutMs { get; }
}
```

---

## Summary: Quick Reference

| Guideline | Do | Do Not |
|---|---|---|
| Default lifecycle | Use transient (no attribute) for stateless services | Make everything singleton "just in case" |
| Singleton | Use for shared state, expensive resources, AWS clients | Inject scoped/transient services into singletons |
| Scoped | Use for per-request state | Use scoped in Lambda unless you need per-invocation isolation |
| Interface registration | `[Expose(typeof(IMyService))]` | `[Expose]` on concrete types that are injected elsewhere |
| Library defaults | `[Expose(typeof(IFoo), Try = true)]` | Hard-register defaults in libraries |
| Environment switching | `[ForEnvironment]` per environment | Conditional logic in service constructors |
| Configuration | One `[ConfigurationModel]` per concern | Monolithic config interfaces |
| Circular deps | Extract shared abstractions | Rely on `IServiceProvider` as a primary pattern |

---

## Next Steps

- [Testing Strategy](testing-strategy.md) -- test services at every layer
- [Lambda Performance](lambda-performance.md) -- lifecycle choices that impact cold start
- [Project Organization](project-organization.md) -- structure projects for clarity
