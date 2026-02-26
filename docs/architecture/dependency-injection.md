# Dependency Injection

Hardened replaces runtime assembly scanning with **compile-time dependency injection**. You annotate classes with attributes like `[Expose]`, `[Singleton]`, and `[Scoped]`, and the source generator produces direct calls to `Microsoft.Extensions.DependencyInjection` at build time. There is no reflection at runtime.

---

## How It Works

### The Traditional Way

In a standard ASP.NET Core application, you register services manually or through assembly scanning:

```csharp
// Manual registration
services.AddTransient<IMyService, MyService>();
services.AddSingleton<ICacheService, CacheService>();

// Or assembly scanning via third-party libraries
services.Scan(scan => scan
    .FromAssemblyOf<MyService>()
    .AddClasses()
    .AsImplementedInterfaces());
```

Both approaches have drawbacks. Manual registration is tedious and error-prone. Assembly scanning uses runtime reflection, which increases startup time and is incompatible with AOT compilation.

### The Hardened Way

In Hardened, you decorate your class with attributes:

```csharp
[Expose(typeof(IMyService))]
public class MyService : IMyService {
    // ...
}
```

At build time, the source generator emits:

```csharp
// Generated code
serviceCollection.AddTransient(typeof(IMyService), typeof(MyService));
```

No reflection. No scanning. The registration is a plain method call compiled into your application.

---

## Attributes

### `[Expose]`

The primary attribute for registering a service in the DI container.

```csharp
namespace Hardened.Shared.Runtime.Attributes;

public class ExposeAttribute : Attribute {
    public ExposeAttribute(params Type[] forServices) { }

    public Type[] ForServices { get; }
    public bool Try { get; set; } = false;
}
```

#### Basic Usage

Register a class as an implementation of an interface:

```csharp
[Expose(typeof(IOrderService))]
public class OrderService : IOrderService {
    public OrderService(IRepository repository) { }
}
```

**Generated:**
```csharp
serviceCollection.AddTransient(typeof(IOrderService), typeof(OrderService));
```

#### Multiple Interfaces

Register a class for multiple service types:

```csharp
[Expose(typeof(IReader), typeof(IWriter))]
public class ReadWriteService : IReader, IWriter {
    // ...
}
```

**Generated:**
```csharp
serviceCollection.AddTransient(typeof(IReader), typeof(ReadWriteService));
serviceCollection.AddTransient(typeof(IWriter), typeof(ReadWriteService));
```

#### Self-Registration

Omit the type parameter to register the class as itself:

```csharp
[Expose]
public class HelperUtility {
    // ...
}
```

**Generated:**
```csharp
serviceCollection.AddTransient(typeof(HelperUtility), typeof(HelperUtility));
```

#### Try Registration

Set `Try = true` to register only if no other registration exists for the service type. This generates `TryAddTransient` (or `TryAddSingleton`/`TryAddScoped`) instead of `AddTransient`:

```csharp
[Expose(typeof(ILogger)), Try = true]
public class DefaultLogger : ILogger {
    // ...
}
```

**Generated:**
```csharp
serviceCollection.TryAddTransient(typeof(ILogger), typeof(DefaultLogger));
```

This is useful for providing default implementations in library modules that consumers can override.

### `[Singleton]`

Changes the service lifetime to singleton (one instance for the entire application):

```csharp
[Expose(typeof(ICacheService))]
[Singleton]
public class InMemoryCacheService : ICacheService {
    // ...
}
```

**Generated:**
```csharp
serviceCollection.AddSingleton(typeof(ICacheService), typeof(InMemoryCacheService));
```

### `[Scoped]`

Changes the service lifetime to scoped (one instance per request/scope):

```csharp
[Expose(typeof(IUserContext))]
[Scoped]
public class UserContext : IUserContext {
    // ...
}
```

**Generated:**
```csharp
serviceCollection.AddScoped(typeof(IUserContext), typeof(UserContext));
```

### Default Lifetime

When neither `[Singleton]` nor `[Scoped]` is specified, the default lifetime is **Transient** -- a new instance is created each time the service is requested.

### `[ForEnvironment]`

Restricts a service registration to specific environments:

```csharp
[Expose(typeof(IEmailSender))]
[Singleton]
[ForEnvironment("Production")]
public class SmtpEmailSender : IEmailSender {
    // ...
}

[Expose(typeof(IEmailSender))]
[Singleton]
[ForEnvironment("Development")]
[ForEnvironment("Test")]
public class FakeEmailSender : IEmailSender {
    // ...
}
```

**Generated:**
```csharp
if (environment.Matches("Production")) {
    serviceCollection.AddSingleton(typeof(IEmailSender), typeof(SmtpEmailSender));
}
if (environment.Matches("Development", "Test")) {
    serviceCollection.AddSingleton(typeof(IEmailSender), typeof(FakeEmailSender));
}
```

The `[ForEnvironment]` attribute can be applied multiple times. Multiple values on the same class are OR-ed together (the service is registered if the environment matches any of them).

!!! note "Environment Ordering"
    Services without `[ForEnvironment]` are registered first, followed by environment-specific services. This means environment-specific registrations can override defaults when `Try = false` (the default).

---

## Generated Code Structure

The source generator produces a partial class file named `{EntryPoint}.DependencyInjection.cs` containing all DI registrations. Here is a simplified view of what gets generated for a typical application:

```csharp
// Application.DependencyInjection.cs (generated)
public partial class Application : IApplicationModule {

    public class ModuleAttribute : Attribute, IApplicationModuleProvider {
        public IEnumerable<IApplicationModule> ProvideModules() {
            var appInstance = new Application();
            yield return appInstance;
        }
    }

    static Application() { }

    public void ConfigureModule(
        IHardenedEnvironment environment,
        IServiceCollection serviceCollection) {

        if (DependencyRegistry<Application>.ShouldRegisterModule(serviceCollection)) {
            // 1. Register framework-level dependencies
            StandardDependencies.Register(environment, serviceCollection);

            // 2. Process runtime module attributes
            StandardDependencies.ProcessModuleProviders(
                environment, serviceCollection,
                new AspNetCoreRuntime.Module());

            // 3. Process sub-modules (if Modules() method exists)
            StandardDependencies.ProcessModules(
                environment, serviceCollection, Modules());

            // 4. Apply DependencyRegistry registrations (from libraries)
            DependencyRegistry<Application>.ApplyRegistration(
                environment, serviceCollection, this);

            // 5. Register [Expose] services (non-environment-specific first)
            serviceCollection.AddTransient(typeof(IOrderService), typeof(OrderService));
            serviceCollection.AddSingleton(typeof(ICacheService), typeof(InMemoryCacheService));

            // 6. Register environment-specific services
            if (environment.Matches("Production")) {
                serviceCollection.AddSingleton(
                    typeof(IEmailSender), typeof(SmtpEmailSender));
            }

            // 7. Call RegisterDependencies (if defined by user)
            RegisterDependencies(environment, serviceCollection);
        }
    }
}
```

### Registration Order

The generated code follows a specific registration order:

1. **Framework dependencies** -- Core services required by Hardened itself
2. **Runtime module providers** -- Services from runtime attributes (`[AspNetCoreRuntime.Module]`, etc.)
3. **Sub-modules** -- Services from the `Modules()` method
4. **DependencyRegistry registrations** -- Compile-time registrations from library references
5. **Non-environment services** -- `[Expose]` services without `[ForEnvironment]`
6. **Environment-specific services** -- `[Expose]` services with `[ForEnvironment]`
7. **User-defined registrations** -- The `RegisterDependencies()` method

This ordering ensures that more specific registrations (environment-specific, user-defined) can override more general ones.

---

## DependencyRegistry&lt;T&gt; and Library Composition

When a library project uses `[HardenedModule]` with the `Hardened.Library.SourceGenerator`, its `[Expose]` services are registered via `DependencyRegistry<T>`. This enables seamless composition:

```csharp
// In MyLibrary project
[HardenedModule]
public partial class MyLibraryModule { }

[Expose(typeof(IDataAccess))]
[Singleton]
public class SqlDataAccess : IDataAccess { }
```

The library source generator produces a static initializer that calls `DependencyRegistry<MyLibraryModule>.Register(...)`. When the consuming application references this library, those registrations are picked up automatically.

The `ShouldRegisterModule` method on `DependencyRegistry<T>` prevents duplicate registration when the same service collection is configured more than once -- it tracks which `IServiceCollection` instance was last configured using a `WeakReference`.

---

## Contrast with Reflection-Based DI

| Aspect | Reflection-Based | Hardened Compile-Time |
|---|---|---|
| Discovery | Assembly scanning at startup | Source generator at build time |
| Startup cost | O(n) where n = types in assemblies | O(1) -- direct method calls |
| AOT support | Limited / requires trimming hints | Full -- no reflection used |
| Error detection | Runtime `InvalidOperationException` | Build-time compiler errors |
| Debugging | Opaque -- hard to trace registration source | Inspect generated `.cs` files |
| Flexibility | Can register anything at runtime | Attribute-driven with `RegisterDependencies` escape hatch |

!!! tip "The Best of Both Worlds"
    The `RegisterDependencies()` method gives you full access to `IServiceCollection` for cases where attributes are insufficient -- third-party library registration, factory patterns, or conditional logic beyond what `[ForEnvironment]` supports.

---

## Common Patterns

### Interface Segregation

```csharp
public interface IOrderReader {
    Task<Order> GetOrder(string id);
}

public interface IOrderWriter {
    Task SaveOrder(Order order);
}

[Expose(typeof(IOrderReader), typeof(IOrderWriter))]
[Scoped]
public class OrderRepository : IOrderReader, IOrderWriter {
    private readonly IDbConnection _db;

    public OrderRepository(IDbConnection db) {
        _db = db;
    }

    public Task<Order> GetOrder(string id) => ...;
    public Task SaveOrder(Order order) => ...;
}
```

### Default with Override

```csharp
// In a library module -- provides a default
[Expose(typeof(INotificationService)), Try = true]
[Singleton]
public class NoOpNotificationService : INotificationService {
    public Task Notify(string message) => Task.CompletedTask;
}

// In the application -- overrides the default
[Expose(typeof(INotificationService))]
[Singleton]
public class SlackNotificationService : INotificationService {
    public Task Notify(string message) => ...;
}
```

### Environment-Specific Implementations

```csharp
[Expose(typeof(ISecretStore))]
[Singleton]
[ForEnvironment("Production")]
[ForEnvironment("Staging")]
public class AwsSecretStore : ISecretStore { }

[Expose(typeof(ISecretStore))]
[Singleton]
[ForEnvironment("Development")]
public class LocalFileSecretStore : ISecretStore { }
```

---

## Next Steps

- [Module System](module-system.md) -- How modules compose and how `DependencyRegistry<T>` enables library-level DI
- [Configuration System](configuration-system.md) -- Configuration models that are also registered via compile-time DI
- [Execution Pipeline](execution-pipeline.md) -- Filters and handlers that consume injected services
