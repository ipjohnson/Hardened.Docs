# Dependency Injection

Hardened provides a compile-time dependency injection system powered by source generators. Instead of registering services manually at runtime, you decorate your classes with attributes and the source generator writes all the DI registration code at build time.

Registration itself is provided by [DependencyModules](https://github.com/ipjohnson/DependencyModules). Hardened's generator reads the same attributes against its own `[HardenedModule]` entry point, so everything on this page works in a Hardened application without declaring a `[DependencyModule]`.

**Package:** `DependencyModules.Runtime` (namespace `DependencyModules.Runtime.Attributes`)

!!! note "Moved from `[Expose]`"
    Earlier versions of Hardened had their own `[Expose]`, `[Singleton]`, `[Scoped]` and `[ForEnvironment]` attributes in `Hardened.Shared.Runtime.Attributes`. Those have been removed. See [Migrating from `[Expose]`](#migrating-from-expose) at the bottom of this page.

---

## Lifetime Attributes

There is one attribute per lifetime, and it is the registration. Unlike `[Expose]`, there is no separate "register this" attribute to combine with a lifetime — you pick the lifetime directly.

```csharp
using DependencyModules.Runtime.Attributes;

[TransientService]
public class OrderProcessor {
    public void Process(Order order) {
        // ...
    }
}
```

| Attribute | Lifetime | When to Use |
|---|---|---|
| `[TransientService]` | Transient | Lightweight, stateless services |
| `[ScopedService]` | Scoped | Per-request state, unit-of-work patterns |
| `[SingletonService]` | Singleton | Shared state, caches, connection pools |

!!! warning
    Avoid injecting scoped or transient services into singletons. This is a common mistake that leads to captured dependencies and memory leaks.

---

## What a Service Is Registered As

When you do not say, the generator infers the service type: **the first interface in the base list that is not a capability interface**; failing that, an interface the base class provides; failing that, the class itself.

```csharp
// Registered as IOrderRepository
[SingletonService]
public class DynamoDbOrderRepository : IOrderRepository { }

// No interfaces - registered as OrderProcessor
[TransientService]
public class OrderProcessor { }
```

Capability interfaces are skipped when inferring, because implementing one describes what a class *does*, not what callers ask for:

```csharp
// Registered as IPool, not IDisposable
[SingletonService]
public class OrderedPool : IDisposable, IPool { }
```

The skipped set is `IDisposable`, `IAsyncDisposable`, `ICloneable`, `IComparable`, `IEquatable`, `IConvertible`, `IFormattable`, `ISpanFormattable`, `IParsable`, `ISpanParsable`, `IEnumerable`, `ISerializable`, `INotifyPropertyChanged`, `INotifyPropertyChanging` and `INotifyCollectionChanged`.

### As -- Naming the Service Type

Set `As` to register for a specific type. Inference is bypassed entirely, so this also names a type that is not an interface:

```csharp
[SingletonService(As = typeof(IOrderRepository))]
public class DynamoDbOrderRepository : IOrderRepository, IHealthCheck { }
```

`As` is how you keep a concrete-type registration when the class also implements interfaces. If anything resolves the class by name, say so:

```csharp
// Resolved as SqsBatchFilter by the startup service that registers it
[SingletonService(As = typeof(SqsBatchFilter))]
public class SqsBatchFilter : BaseBatchExecutionFilter<SQSEvent, SQSEvent.SQSMessage> { }
```

!!! warning
    Without `As` here, inference walks up to `BaseBatchExecutionFilter` and registers an interface off it — and every `GetRequiredService<SqsBatchFilter>()` throws at startup.

---

## [CrossWireService] -- One Instance, Several Interfaces

To expose one implementation under several interfaces, use `[CrossWireService]`. It registers the implementation once and points every interface it implements at that registration, so the same instance is returned for all of them:

```csharp
public interface IReader { string Read(); }
public interface IWriter { void Write(string data); }

[CrossWireService(Lifetime = ServiceLifetime.Singleton)]
public class FileStore : IReader, IWriter {
    public string Read() { /* ... */ }
    public void Write(string data) { /* ... */ }
}
```

```csharp
ReferenceEquals(provider.GetRequiredService<IReader>(),
                provider.GetRequiredService<IWriter>());   // true
```

`[CrossWireService]` takes its lifetime as a property rather than having one attribute per lifetime.

!!! note
    This is the replacement for `[Expose(typeof(IReader), typeof(IWriter))]`, and it behaves better. `[Expose]` emitted an independent registration per interface, so a singleton exposed under two interfaces produced *two* instances.

---

## Using -- How the Registration Is Made

`Using` controls the `IServiceCollection` method used, and applies to every registration attribute:

| Value | Behaviour |
|---|---|
| `RegistrationType.Add` | *(default)* Always adds, even if the service type is already registered |
| `RegistrationType.Try` | Registers only if nothing is registered for the service type |
| `RegistrationType.TryEnumerable` | Adds unless this exact implementation is already registered |
| `RegistrationType.Replace` | Replaces any existing registration |

`Try` is how a package ships a default a consumer can override:

```csharp
[SingletonService(As = typeof(IEmailSender), Using = RegistrationType.Try)]
public class DefaultEmailSender : IEmailSender {
    public Task Send(string to, string body) {
        // Default implementation -- can be overridden
    }
}
```

---

## Environment-Conditional Registration

`[IfEnvironment]` restricts a registration to named environments — useful for swapping implementations between development, staging and production:

```csharp
public interface IPaymentGateway {
    Task<PaymentResult> Charge(decimal amount, string token);
}

[TransientService(As = typeof(IPaymentGateway))]
[IfEnvironment("Production")]
public class StripePaymentGateway : IPaymentGateway {
    public Task<PaymentResult> Charge(decimal amount, string token) {
        // Real Stripe API calls
    }
}

[TransientService(As = typeof(IPaymentGateway))]
[IfEnvironment("Development", "Test")]
public class FakePaymentGateway : IPaymentGateway {
    public Task<PaymentResult> Charge(decimal amount, string token) {
        return Task.FromResult(new PaymentResult { Success = true });
    }
}
```

One attribute takes several names, and the attribute also stacks. The full set:

| Attribute | Registers when |
|---|---|
| `[IfEnvironment("A", "B")]` | The environment name is any of these |
| `[IfNotEnvironment("A")]` | The environment name is none of these |
| `[IfEnvironmentValue("KEY")]` | The environment has any value for `KEY` |
| `[IfEnvironmentValue("KEY", "VALUE")]` | `KEY` equals `VALUE` |
| `[IfNotEnvironmentValue("KEY", "VALUE")]` | `KEY` does not equal `VALUE` |

Stacked conditions combine with **and**. Conditions also apply to conventions and decorators, not only to classes.

!!! note
    Environment **names** compare case-insensitively, matching `IHostEnvironment.IsDevelopment()` — `[IfEnvironment("production")]` and `[IfEnvironment("Production")]` are the same. Environment **values** compare case-sensitively, because a value is data rather than a well-known label.

The environment comes from `IModuleEnvironment`. Hardened's `IHardenedEnvironment` implements it, so the environment your application already has is the one conditions are evaluated against. See [Environment](environment.md).

---

## Registering by Convention

Rather than attributing each class, a module can declare conventions. Implement `IConventionModule` on the module:

```csharp
using DependencyModules.Runtime.Conventions;

[HardenedModule]
public partial class DataModule : IConventionModule {
    void IConventionModule.Conventions(IConventionDefinitions conventions) {
        conventions.RegisterAll<IRepository>().AsScoped();
        conventions.RegisterAll(typeof(IRequestHandler<,>)).AsTransient();
    }
}
```

Every type implementing the named interface is registered, with no attribute on any of them.

!!! warning
    The body of `Conventions` is **never executed**. It is read at compile time and turned into ordinary registrations. Only a chain of the calls declared on `IConventionDefinitions` and `IConventionRegistration` can appear in it — loops, conditionals, locals and calls to your own helpers are reported as `DM0009` rather than silently ignored.

---

## Decorators

`[Decorate]` on the module wraps registrations, including open generics closed over whatever type arguments each registration used:

```csharp
public class LoudHandler<T> : IHandler<T> {
    private readonly IHandler<T> _inner;
    public LoudHandler(IHandler<T> inner) => _inner = inner;
    public string Handle(T value) => _inner.Handle(value).ToUpperInvariant() + "!";
}

[HardenedModule]
[Decorate(typeof(IHandler<>), typeof(LoudHandler<>))]
public partial class HandlerModule : IConventionModule {
    void IConventionModule.Conventions(IConventionDefinitions conventions) {
        conventions.RegisterAll(typeof(IHandler<>)).AsTransient();
    }
}
```

An `IHandler<string>` and an `IHandler<int>` are each wrapped by a `LoudHandler<>` closed over their own argument.

---

## How It Works -- Compile-Time Registration

When you build, Hardened's source generator scans for registration attributes and generates the DI code. This means:

1. **No runtime reflection** -- service discovery happens at compile time
2. **Fast startup** -- registration code is pre-generated, not discovered at runtime
3. **Native AOT clean** -- nothing depends on assembly scanning

The generated code is equivalent to calling `IServiceCollection` methods directly:

```csharp
// What you write:
[SingletonService(As = typeof(IOrderRepository))]
public class DynamoDbOrderRepository : IOrderRepository { }

// What the source generator produces (conceptually):
services.AddSingleton<IOrderRepository, DynamoDbOrderRepository>();
```

You can inspect the generated files under `obj/Debug/net8.0/generated/` in your project directory after building.

---

## Combining Attributes

```csharp
// Transient service, registered as IOrderValidator by inference
[TransientService]
public class OrderValidator : IOrderValidator {
    public ValidationResult Validate(Order order) { /* ... */ }
}

// Singleton shared across two interfaces, one instance
[CrossWireService(Lifetime = ServiceLifetime.Singleton)]
public class RedisCache : IReadCache, IWriteCache {
    public Task<string?> Get(string key) { /* ... */ }
    public Task Set(string key, string value) { /* ... */ }
}

// Environment-specific, try-register default
[SingletonService(As = typeof(IFeatureFlagService), Using = RegistrationType.Try)]
[IfEnvironment("Development")]
public class LocalFeatureFlagService : IFeatureFlagService {
    public bool IsEnabled(string flag) => true;
}

// Scoped per-request service
[ScopedService]
public class AuditLogger : IAuditLogger {
    public void Log(string action, string entityId) { /* ... */ }
}
```

---

## Migrating from [Expose]

`[Expose]`, `[Singleton]`, `[Scoped]` and `[ForEnvironment]` have been removed from `Hardened.Shared.Runtime`.

| Old | New |
|---|---|
| `[Expose]` | `[TransientService]` |
| `[Expose]` + `[Singleton]` | `[SingletonService]` |
| `[Expose]` + `[Scoped]` | `[ScopedService]` |
| `[Expose(typeof(X))]` | `[TransientService(As = typeof(X))]` |
| `[Expose(typeof(X))]` + `[Singleton]` | `[SingletonService(As = typeof(X))]` |
| `[Expose(typeof(A), typeof(B))]` | `[CrossWireService]` |
| `[Expose(Try = true)]` | `Using = RegistrationType.Try` |
| `[ForEnvironment("X")]` | `[IfEnvironment("X")]` |
| `using Hardened.Shared.Runtime.Attributes;` | `using DependencyModules.Runtime.Attributes;` |

!!! warning "Bare `[Expose]` was transient, not singleton"
    `[Expose]` on its own registered a **transient** service. Translating it to `[SingletonService]` silently changes the lifetime, which matters most for the per-request context classes that are the likeliest to carry a bare `[Expose]`.

Two behavioural differences worth checking as you migrate:

- **Service type inference skips capability interfaces.** `[Expose]` took the first entry in the base list whatever it was, including a base *class*. The new inference takes the first non-capability *interface*, so a class whose base list starts with a base class now registers differently. Add `As` where the old behaviour mattered.
- **`[Expose]` with several types made several registrations.** `[CrossWireService]` makes one and points the interfaces at it, so a singleton is genuinely one instance.

---

## Related Pages

- [Configuration](configuration.md) -- configure services with `[ConfigurationModel]`
- [Application Lifecycle](application-lifecycle.md) -- how modules and startup services interact with DI
- [Environment](environment.md) -- how environment names drive `[IfEnvironment]` matching
- [Architecture: Dependency Injection](../../architecture/dependency-injection.md) -- high-level DI architecture
