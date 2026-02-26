# Dependency Injection

Hardened provides a compile-time dependency injection system powered by source generators. Instead of registering services manually at runtime, you decorate your classes with attributes and the source generator writes all the DI registration code at build time.

**Package:** `Hardened.Shared.Runtime` (namespace `Hardened.Shared.Runtime.Attributes`)

---

## [Expose] Attribute

The `[Expose]` attribute is the primary way to register a class in the DI container. When the source generator encounters this attribute, it emits code that registers the class with `Microsoft.Extensions.DependencyInjection`.

### Definition

```csharp
namespace Hardened.Shared.Runtime.Attributes;

public class ExposeAttribute : Attribute {
    public ExposeAttribute(params Type[] forServices) {
        ForServices = forServices;
    }

    public Type[] ForServices { get; }
    public bool Try { get; set; } = false;
}
```

### Basic Usage

Register a class as itself:

```csharp
using Hardened.Shared.Runtime.Attributes;

[Expose]
public class OrderProcessor {
    public void Process(Order order) {
        // ...
    }
}
```

The generated code registers `OrderProcessor` as a **transient** service (the default lifetime).

### ForServices -- Registering for Interfaces

Use the constructor parameter to specify which interfaces or base types the class should be registered for:

```csharp
public interface IOrderRepository {
    Task<Order?> GetById(string id);
    Task Save(Order order);
}

[Expose(typeof(IOrderRepository))]
public class DynamoDbOrderRepository : IOrderRepository {
    public Task<Order?> GetById(string id) { /* ... */ }
    public Task Save(Order order) { /* ... */ }
}
```

This registers `DynamoDbOrderRepository` as the implementation for `IOrderRepository`. You can register a class for multiple interfaces:

```csharp
public interface IReader { string Read(); }
public interface IWriter { void Write(string data); }

[Expose(typeof(IReader), typeof(IWriter))]
public class FileStore : IReader, IWriter {
    public string Read() { /* ... */ }
    public void Write(string data) { /* ... */ }
}
```

### Try -- Conditional Registration

Set `Try = true` to register the service only if no existing registration exists for the same service type. This is useful for providing default implementations that consumers can override:

```csharp
[Expose(typeof(IEmailSender), Try = true)]
public class DefaultEmailSender : IEmailSender {
    public Task Send(string to, string body) {
        // Default implementation -- can be overridden
    }
}
```

If another module has already registered an `IEmailSender`, this registration is skipped.

---

## Lifecycle Attributes

By default, services registered with `[Expose]` use a **transient** lifecycle -- a new instance is created each time the service is requested. Use `[Singleton]` or `[Scoped]` to change this.

### [Singleton]

A single instance is created and shared for the lifetime of the application:

```csharp
namespace Hardened.Shared.Runtime.Attributes;

public class SingletonAttribute : Attribute { }
```

```csharp
[Expose(typeof(ICacheService))]
[Singleton]
public class InMemoryCacheService : ICacheService {
    private readonly ConcurrentDictionary<string, object> _cache = new();

    public void Set(string key, object value) => _cache[key] = value;
    public object? Get(string key) => _cache.GetValueOrDefault(key);
}
```

### [Scoped]

A single instance is created per scope (typically per HTTP request in web applications):

```csharp
namespace Hardened.Shared.Runtime.Attributes;

public class ScopedAttribute : Attribute { }
```

```csharp
[Expose(typeof(IUserContext))]
[Scoped]
public class UserContext : IUserContext {
    public string? UserId { get; set; }
    public string? TenantId { get; set; }
}
```

### Lifecycle Summary

| Attribute | Lifetime | When to Use |
|---|---|---|
| *(none)* | Transient | Lightweight, stateless services |
| `[Singleton]` | Singleton | Shared state, caches, connection pools |
| `[Scoped]` | Scoped | Per-request state, unit-of-work patterns |

!!! warning
    Avoid injecting scoped or transient services into singletons. This is a common mistake that leads to captured dependencies and memory leaks.

---

## [ForEnvironment] -- Environment-Specific Registration

The `[ForEnvironment]` attribute restricts a service registration to specific environments. This is useful for swapping implementations between development, staging, and production.

### Definition

```csharp
namespace Hardened.Shared.Runtime.Attributes;

[AttributeUsage(AttributeTargets.Class, AllowMultiple = true, Inherited = false)]
public class ForEnvironmentAttribute : Attribute {
    public ForEnvironmentAttribute(string environment) {
        Environment = environment;
    }

    public string Environment { get; }
}
```

### Usage

```csharp
public interface IPaymentGateway {
    Task<PaymentResult> Charge(decimal amount, string token);
}

[Expose(typeof(IPaymentGateway))]
[ForEnvironment("Production")]
public class StripePaymentGateway : IPaymentGateway {
    public Task<PaymentResult> Charge(decimal amount, string token) {
        // Real Stripe API calls
    }
}

[Expose(typeof(IPaymentGateway))]
[ForEnvironment("Development")]
[ForEnvironment("Test")]
public class FakePaymentGateway : IPaymentGateway {
    public Task<PaymentResult> Charge(decimal amount, string token) {
        // Always succeeds, no real charges
        return Task.FromResult(new PaymentResult { Success = true });
    }
}
```

Since `[ForEnvironment]` supports `AllowMultiple = true`, you can apply it multiple times to register a service for several environments.

!!! note
    Environment matching is case-insensitive. `[ForEnvironment("production")]` and `[ForEnvironment("Production")]` are equivalent.

---

## How It Works -- Compile-Time Registration

When you build your project, the Hardened source generator scans for all classes decorated with `[Expose]` and generates DI registration code. This means:

1. **No runtime reflection** -- service discovery happens at compile time
2. **Fast startup** -- registration code is pre-generated, not discovered at runtime
3. **Strong type safety** -- mismatches between `ForServices` types and the class are caught at build time

The generated code is equivalent to calling `IServiceCollection` methods directly:

```csharp
// What you write:
[Expose(typeof(IOrderRepository))]
[Singleton]
public class DynamoDbOrderRepository : IOrderRepository { }

// What the source generator produces (conceptually):
services.AddSingleton<IOrderRepository, DynamoDbOrderRepository>();
```

You can inspect the generated files under `obj/Debug/net8.0/generated/` in your project directory after building.

---

## Combining Attributes

Attributes compose naturally. Here is a complete example showing common patterns:

```csharp
// Transient service registered for its interface
[Expose(typeof(IOrderValidator))]
public class OrderValidator : IOrderValidator {
    public ValidationResult Validate(Order order) { /* ... */ }
}

// Singleton cache registered for multiple interfaces
[Expose(typeof(IReadCache), typeof(IWriteCache))]
[Singleton]
public class RedisCache : IReadCache, IWriteCache {
    public Task<string?> Get(string key) { /* ... */ }
    public Task Set(string key, string value) { /* ... */ }
}

// Environment-specific, try-register default
[Expose(typeof(IFeatureFlagService), Try = true)]
[Singleton]
[ForEnvironment("Development")]
public class LocalFeatureFlagService : IFeatureFlagService {
    public bool IsEnabled(string flag) => true;
}

// Scoped per-request service
[Expose(typeof(IAuditLogger))]
[Scoped]
public class AuditLogger : IAuditLogger {
    public void Log(string action, string entityId) { /* ... */ }
}
```

---

## Related Pages

- [Configuration](configuration.md) -- configure services with `[ConfigurationModel]`
- [Application Lifecycle](application-lifecycle.md) -- how modules and startup services interact with DI
- [Environment](environment.md) -- how environment names drive `[ForEnvironment]` matching
- [Architecture: Dependency Injection](../../architecture/dependency-injection.md) -- high-level DI architecture
