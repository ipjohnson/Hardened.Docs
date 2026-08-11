# Project Organization

Well-organized Hardened projects are easier to navigate, test, and maintain. This page covers opinionated recommendations for structuring projects, drawing boundaries between modules, and composing multi-project solutions.

---

## Recommended Directory Layout

A single Hardened project -- whether a web API, Lambda function, or canary -- should follow this structure:

```
MyService/
  MyService/
    Application.cs            # [HardenedModule] composition root
    Program.cs                # Entry point (web apps only)
    Controllers/              # Route handlers ([Get], [Post], etc.)
    Handlers/                 # Lambda function handlers ([HardenedFunction])
    Services/                 # Business logic with registration attributes
    Models/                   # Request/response DTOs, domain models
    Config/                   # [ConfigurationModel] interfaces
    Filters/                  # IExecutionFilter implementations
    MyService.csproj
  MyService.Tests/
    Bootstrap.cs              # [assembly: HardenedTestEntryPoint]
    Controllers/              # Mirror of main project structure
    Services/
    MyService.Tests.csproj
  nuget.config
  MyService.sln
```

!!! tip "Controllers vs. Handlers"
    Use `Controllers/` for web projects with HTTP route attributes (`[Get]`, `[Post]`, etc.) and `Handlers/` for Lambda function projects with `[HardenedFunction]`. This naming signals the entry point style and makes the project's purpose immediately clear.

### Why This Layout Works

| Directory | Responsibility | Guideline |
|---|---|---|
| `Application.cs` | Composition root | One per project. Keep it minimal -- let the source generator do the work. |
| `Controllers/` | HTTP route definitions | One controller per resource (e.g., `OrderController`, `ProductController`). |
| `Services/` | Business logic | Interface + implementation pairs. All DI registrations live here. |
| `Models/` | Data shapes | Request DTOs, response DTOs, domain entities, configuration models. |
| `Config/` | Typed configuration | `[ConfigurationModel]` interfaces for environment variables and settings. |
| `Filters/` | Cross-cutting concerns | `IExecutionFilter` implementations (auth, logging, metrics). |

---

## Application.cs Composition Patterns

The `Application.cs` file is the most important file in any Hardened project. It defines the composition root where all source-generated wiring converges.

### Keep It Minimal

For most projects, the Application class should contain only the `[HardenedModule]` attribute and the minimum boilerplate for your runtime:

```csharp title="Web API"
[HardenedModule]
[AspNetCoreRuntime.Module]
public partial class Application
{
    public static WebApplicationBuilder CreateBuilder(string[] args)
    {
        var hardenedApp = new Application();
        var environment = new EnvironmentImpl(arguments: args);

        var builder = WebApplication.CreateBuilder(args);
        builder.Services.AddTransient<IHardenedEnvironment>(_ => environment);
        hardenedApp.ConfigureModule(environment, builder.Services);

        return builder;
    }
}
```

```csharp title="Lambda Function"
[HardenedModule]
public partial class Application { }
```

!!! warning "Do Not Put Business Logic in Application.cs"
    The Application class is a composition root, not a service class. Resist the temptation to add methods, fields, or logic here. If you need manual DI registrations, implement `IApplicationModule` to keep them organized, but extract actual logic into dedicated service classes.

### Manual Module Configuration

When you need to register services that the source generator cannot discover (third-party types, factory methods, etc.), implement `IApplicationModule`:

```csharp title="Application.cs"
[HardenedModule]
[AspNetCoreRuntime.Module]
public partial class Application : IApplicationModule
{
    public void ConfigureModule(
        IHardenedEnvironment environment,
        IApplicationModule.ServiceRegistrationDelegate register)
    {
        register.Register<HttpClient>(
            ServiceLifetime.Singleton,
            _ => new HttpClient { Timeout = TimeSpan.FromSeconds(30) });
    }
}
```

Use this sparingly. If you find yourself registering many services manually, consider whether those services should have registration attributes instead.

---

## Module Boundaries

Knowing when to extract code into a separate library module is one of the most impactful design decisions in a Hardened solution.

### When to Create a Separate Module

Create a library module when:

- **Multiple projects share the same services.** If your web API and Lambda function both need `IOrderService`, extract it into a shared library.
- **You want independent versioning.** A library module can be published as a NuGet package and consumed by multiple solutions.
- **The domain boundary is clear.** If a set of services, models, and configuration form a cohesive unit (e.g., "payments", "user management"), a module boundary makes sense.
- **You need to control registration order.** Library modules are composed in a deterministic order, giving you control over which registrations take priority.

### When to Keep Things in One Project

Keep services in the main project when:

- **Only one host consumes them.** A service used exclusively by a single web API does not need its own module.
- **The overhead is not worth it.** Every module requires its own `Application.cs`, `.csproj`, and source generator reference. For small codebases, this is unnecessary ceremony.
- **You are prototyping.** Start with one project and extract modules when the boundaries become clear.

!!! note "Rule of Thumb"
    If two or more host projects (web APIs, Lambda functions) need the same service, extract it into a library module. Otherwise, keep it local.

---

## Multi-Module Projects

In a multi-module solution, one or more library modules provide shared services, and host projects compose them into a running application.

```
MyPlatform/
  MyPlatform.Core/
    Application.cs            # [HardenedModule] for shared services
    Services/
    Models/
    Config/
    MyPlatform.Core.csproj    # Hardened.Library.SourceGenerator
  MyPlatform.Api/
    Application.cs            # [HardenedModule] + [AspNetCoreRuntime.Module]
    Program.cs
    Controllers/
    MyPlatform.Api.csproj     # References MyPlatform.Core
  MyPlatform.OrderProcessor/
    Application.cs            # [HardenedModule]
    Handlers/
    MyPlatform.OrderProcessor.csproj  # References MyPlatform.Core
  MyPlatform.Tests/
    Bootstrap.cs
    Controllers/
    Handlers/
    MyPlatform.Tests.csproj
  nuget.config
  MyPlatform.sln
```

### Library Module Setup

Library modules use `Hardened.Library.SourceGenerator` instead of the runtime-specific generators:

```xml title="MyPlatform.Core.csproj"
<ItemGroup>
  <PackageReference Include="Hardened.Library.SourceGenerator" Version="..." />
</ItemGroup>
```

```csharp title="MyPlatform.Core/Application.cs"
using Hardened.Shared.Runtime.Attributes;

[HardenedModule]
public partial class Application { }
```

Every service in the library marked with a registration attribute is automatically discovered and composed into any host project that references it. No additional wiring is required in the host.

### Composition Is Automatic

When the host project's source generator runs, it discovers all `[HardenedModule]` classes in referenced assemblies and composes them:

```csharp title="MyPlatform.Api/Application.cs"
[HardenedModule]
[AspNetCoreRuntime.Module]
public partial class Application
{
    // MyPlatform.Core's services are automatically included.
    // No explicit reference to the library module is needed here.
}
```

!!! tip "Overriding Library Registrations"
    If a library module registers `IFooService` and the host project also registers `IFooService`, the host's registration wins (last registration takes precedence). Use `Try = true` in library modules for default implementations that consumers can override.

---

## Shared Libraries with Library.SourceGenerator

The `Hardened.Library.SourceGenerator` is purpose-built for reusable modules. It generates DI registrations and configuration implementations but does not generate runtime-specific code (no Lambda bootstrap, no ASP.NET Core middleware).

### Guidelines for Library Modules

1. **Keep libraries runtime-agnostic.** A library module should not depend on `Hardened.Web.Runtime` or `Hardened.Amz.Function.Lambda.Runtime`. It should only depend on `Hardened.Shared.Runtime` and `Hardened.Requests.Abstract` (if it needs filters).

2. **Use `Try = true` for default implementations.** This lets host projects override registrations without conflicts:

    ```csharp
    [SingletonService(As = typeof(IEmailSender), Using = RegistrationType.Try)]
    public class SmtpEmailSender : IEmailSender { }
    ```

3. **Expose configuration interfaces.** Library modules can define `[ConfigurationModel]` interfaces that host projects provide values for:

    ```csharp
    [ConfigurationModel]
    public interface IDatabaseConfig
    {
        [FromEnvironmentVariable("DB_CONNECTION_STRING")]
        string ConnectionString { get; }
    }
    ```

4. **Document required configuration.** Since library consumers need to set environment variables or configuration values, make the requirements clear in code comments or companion documentation.

---

## Separation of Concerns

### Controllers: Thin and Focused

Controllers should be thin orchestrators. They receive requests, delegate to services, and return responses:

```csharp
public class OrderController
{
    private readonly IOrderService _orderService;

    public OrderController(IOrderService orderService)
    {
        _orderService = orderService;
    }

    [Post("/orders")]
    public async Task<OrderResponse> CreateOrder(OrderRequest request)
    {
        return await _orderService.CreateOrder(request);
    }

    [Get("/orders/{orderId}")]
    public async Task<OrderResponse> GetOrder(string orderId)
    {
        return await _orderService.GetOrder(orderId);
    }
}
```

!!! warning "Avoid Business Logic in Controllers"
    If a controller method grows beyond a few lines of delegation, extract the logic into a service. Controllers should not contain validation, data transformation, or business rules.

### Services: Where Logic Lives

Services contain the business logic and are the primary unit of reuse and testing:

```csharp
public interface IOrderService
{
    Task<OrderResponse> CreateOrder(OrderRequest request);
    Task<OrderResponse> GetOrder(string orderId);
}

[TransientService(As = typeof(IOrderService))]
public class OrderService : IOrderService
{
    private readonly IOrderRepository _repository;
    private readonly IOrderValidator _validator;

    public OrderService(
        IOrderRepository repository,
        IOrderValidator validator)
    {
        _repository = repository;
        _validator = validator;
    }

    public async Task<OrderResponse> CreateOrder(OrderRequest request)
    {
        _validator.Validate(request);
        var order = new Order(request);
        await _repository.Save(order);
        return order.ToResponse();
    }

    public async Task<OrderResponse> GetOrder(string orderId)
    {
        var order = await _repository.GetById(orderId)
            ?? throw new NotFoundException($"Order {orderId} not found");
        return order.ToResponse();
    }
}
```

### Filters: Cross-Cutting Only

Use `IExecutionFilter` for concerns that span multiple handlers -- authentication, logging, error handling, metrics. Do not use filters as a replacement for service logic:

```csharp
[SingletonService]
public class RequestLoggingFilter : IExecutionFilter
{
    public int Order => ExecutionFilterOrder.Init + 1;

    public async Task Execute(IExecutionChain chain)
    {
        var context = chain.Context;
        var logger = context.Services.GetRequiredService<ILogger<RequestLoggingFilter>>();

        logger.LogInformation("Request started: {Path}", context.Request.Path);
        var sw = Stopwatch.StartNew();

        await chain.Next();

        logger.LogInformation(
            "Request completed: {Path} in {ElapsedMs}ms",
            context.Request.Path,
            sw.ElapsedMilliseconds);
    }
}
```

---

## Test Project Organization

Mirror the structure of the project under test. This makes it easy to find tests for any given class:

```
MyService.Tests/
  Bootstrap.cs
  Controllers/
    OrderControllerTests.cs     # Tests for OrderController
    ProductControllerTests.cs
  Services/
    OrderServiceTests.cs        # Tests for OrderService
  Filters/
    RequestLoggingFilterTests.cs
```

### Single Bootstrap File

Every test project needs exactly one `Bootstrap.cs` file with assembly-level attributes:

```csharp title="Bootstrap.cs (web project tests)"
using Hardened.Web.Testing;
using Hardened.Shared.Runtime.Attributes;

[assembly: WebTesting]
[assembly: HardenedTestEntryPoint(typeof(Application))]
```

```csharp title="Bootstrap.cs (Lambda project tests)"
using Hardened.Shared.Runtime.Attributes;

[assembly: HardenedTestEntryPoint(typeof(Application))]
```

!!! tip "One Test Project Per Host"
    If your solution has multiple host projects (e.g., `MyPlatform.Api` and `MyPlatform.OrderProcessor`), create a separate test project for each. Each test project references its host and uses that host's `Application` class as the entry point.

---

## File Naming Conventions

| Convention | Example | Rationale |
|---|---|---|
| Interface + Implementation | `IOrderService.cs`, `OrderService.cs` | Pairs are easy to find alphabetically |
| Controller suffix | `OrderController.cs` | Signals HTTP routing responsibility |
| Handler suffix | `OrderHandler.cs` | Signals Lambda function handling |
| Filter suffix | `AuthFilter.cs`, `LoggingFilter.cs` | Signals cross-cutting pipeline behavior |
| Config prefix with `I` | `IAppConfig.cs`, `IDatabaseConfig.cs` | Matches `[ConfigurationModel]` interface pattern |
| Tests suffix | `OrderServiceTests.cs` | Standard .NET testing convention |

---

## Next Steps

- [Dependency Injection Best Practices](dependency-injection.md) -- optimize your DI registrations
- [Testing Strategy](testing-strategy.md) -- organize tests across the testing pyramid
- [Lambda Performance](lambda-performance.md) -- tune Lambda deployments for fast cold starts
