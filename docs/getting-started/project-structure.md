# Project Structure

This page covers recommended conventions for organizing Hardened projects. These are not rigid rules -- Hardened's source generators work regardless of file layout -- but following these patterns keeps projects consistent and easy to navigate.

---

## Single-project layout

A typical Hardened web API or Lambda function project:

```
MyService/
  MyService/
    Application.cs            # [HardenedModule] composition root
    Program.cs                # Entry point (web apps only)
    Controllers/
      OrderController.cs      # Route handlers
      ProductController.cs
    Services/
      IOrderService.cs        # Service interfaces
      OrderService.cs         # [Expose] implementations
    Models/
      OrderRequest.cs         # Request/response models
      OrderResponse.cs
    Config/
      IAppConfig.cs           # [ConfigurationModel] interfaces
    Filters/
      LoggingFilter.cs        # IExecutionFilter implementations
    nuget.config
    MyService.csproj
  MyService.Tests/
    Bootstrap.cs              # [assembly: HardenedTestEntryPoint]
    Controllers/
      OrderControllerTests.cs # [HardenedTest] tests
    MyService.Tests.csproj
  MyService.sln
```

### Key conventions

| File/Directory | Purpose |
|---|---|
| `Application.cs` | The `[HardenedModule]` partial class. One per project. Acts as the composition root. |
| `Program.cs` | Minimal entry point for web apps. Lambda functions do not need this. |
| `Controllers/` | Classes with `[Get]`, `[Post]`, `[Put]`, `[Delete]` route methods |
| `Services/` | Business logic. Interfaces paired with `[Expose]` implementations. |
| `Models/` | Request, response, and domain model classes |
| `Config/` | `[ConfigurationModel]` interfaces for typed configuration |
| `Filters/` | `IExecutionFilter` implementations for cross-cutting concerns |
| `Bootstrap.cs` | Test project assembly attributes for test framework setup |

---

## The Application class

The `Application.cs` file is the most important file in a Hardened project. It defines the **composition root** -- the point where the source generator wires everything together.

### Web application

```csharp title="Application.cs"
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
        builder.Services.AddTransient<IHardenedEnvironment>(_ => environment);
        hardenedApp.ConfigureModule(environment, builder.Services);

        return builder;
    }
}
```

### Lambda function

```csharp title="Application.cs"
using Hardened.Shared.Runtime.Attributes;

[HardenedModule]
public partial class Application { }
```

### Library module

```csharp title="Application.cs"
using Hardened.Shared.Runtime.Attributes;

[HardenedModule]
public partial class Application { }
```

!!! note
    The class must be `partial`. The source generator emits the `ConfigureModule` method, DI registrations, and route bindings into the other partial definition.

---

## What the source generator discovers

When you mark a class with `[HardenedModule]`, the source generator scans the assembly for:

| Attribute | What it does |
|---|---|
| `[Expose]`, `[Singleton]`, `[Scoped]` | Registers the class in the DI container |
| `[Get]`, `[Post]`, `[Put]`, `[Delete]` | Registers route handlers |
| `[HardenedFunction]` | Registers the Lambda function entry point |
| `[HardenedCanary]` | Registers canary test methods |
| `[ConfigurationModel]` | Generates a configuration implementation class |

All of this happens at compile time. There is no runtime assembly scanning or reflection.

---

## Multi-project solutions

For larger applications, split shared logic into a library module and reference it from your host project.

```
MyPlatform/
  MyPlatform.Core/
    Application.cs            # [HardenedModule] for the library
    Services/
      IUserService.cs
      UserService.cs          # [Expose]
    Models/
      User.cs
    MyPlatform.Core.csproj    # Uses Hardened.Library.SourceGenerator
  MyPlatform.Api/
    Application.cs            # [HardenedModule] + [AspNetCoreRuntime.Module]
    Program.cs
    Controllers/
      UserController.cs
    MyPlatform.Api.csproj     # Uses Hardened.Web.AspNetCore.SourceGenerator
  MyPlatform.Lambda/
    Application.cs            # [HardenedModule]
    Handlers/
      UserEventHandler.cs
    MyPlatform.Lambda.csproj  # Uses Hardened.Amz.Function.Lambda.SourceGenerator
  MyPlatform.Tests/
    Bootstrap.cs
    ...
  nuget.config
  MyPlatform.sln
```

### Library project setup

The shared library uses `Hardened.Library.SourceGenerator`:

```bash
dotnet add package Hardened.Library.SourceGenerator --prerelease
```

```csharp title="MyPlatform.Core/Application.cs"
using Hardened.Shared.Runtime.Attributes;

[HardenedModule]
public partial class Application { }
```

Services in the library marked with `[Expose]` are automatically available to any project that references the library. The host project's source generator discovers and includes the library module's registrations.

### Host project

The host project (web API, Lambda, etc.) references the library:

```xml title="MyPlatform.Api/MyPlatform.Api.csproj"
<ItemGroup>
  <ProjectReference Include="..\MyPlatform.Core\MyPlatform.Core.csproj" />
</ItemGroup>
```

The host's `[HardenedModule]` automatically composes with any `[HardenedModule]` classes found in referenced projects. No additional wiring is needed.

---

## Lambda project conventions

Lambda projects have a slightly different structure since they do not have a `Program.cs`:

```
OrderProcessor/
  OrderProcessor/
    Application.cs            # [HardenedModule]
    Handlers/
      OrderHandler.cs         # [HardenedFunction]
    Services/
      IOrderService.cs
      OrderService.cs
    Models/
      OrderRequest.cs
      OrderResponse.cs
    OrderProcessor.csproj
  OrderProcessor.Tests/
    Bootstrap.cs
    Handlers/
      OrderHandlerTests.cs
    OrderProcessor.Tests.csproj
```

!!! tip
    Use a `Handlers/` directory (instead of `Controllers/`) for Lambda function projects to signal that these are event handlers, not HTTP route handlers.

---

## Canary project conventions

Canary projects group canary classes by the service or feature they monitor:

```
ServiceCanary/
  ServiceCanary/
    Application.cs            # [HardenedModule]
    Canaries/
      HealthCanary.cs         # [HardenedCanary]
      OrderFlowCanary.cs
      AuthCanary.cs
    Config/
      ICanaryConfig.cs        # [ConfigurationModel]
    Services/
      IApiClient.cs
      ApiClient.cs
    ServiceCanary.csproj
```

---

## Test project conventions

### Web API tests

```csharp title="Bootstrap.cs"
using Hardened.Web.Testing;
using Hardened.Shared.Runtime.Attributes;

[assembly: WebTesting]
[assembly: HardenedTestEntryPoint(typeof(Application))]
```

### Lambda function tests

```csharp title="Bootstrap.cs"
using Hardened.Shared.Runtime.Attributes;

[assembly: HardenedTestEntryPoint(typeof(Application))]
```

### Test file organization

Mirror the structure of the project under test:

```
MyService.Tests/
  Bootstrap.cs
  Controllers/
    OrderControllerTests.cs   # Tests for OrderController
  Services/
    OrderServiceTests.cs      # Tests for OrderService
```

!!! tip
    Use `[HardenedTest]` for tests that need DI injection and the full Hardened pipeline. For pure unit tests with no framework dependencies, standard `[Fact]` works fine.

---

## Configuration files

### nuget.config

Place `nuget.config` at the solution root so all projects share the same package source:

```xml title="nuget.config"
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <packageSources>
    <add key="nuget.org" value="https://api.nuget.org/v3/index.json" />
    <add key="github-ipjohnson-org"
         value="https://nuget.pkg.github.com/ipjohnson-org/index.json" />
  </packageSources>
  <packageSourceCredentials>
    <github-ipjohnson-org>
      <add key="Username" value="YOUR_GITHUB_USERNAME" />
      <add key="ClearTextPassword" value="%GITHUB_TOKEN%" />
    </github-ipjohnson-org>
  </packageSourceCredentials>
</configuration>
```

### .csproj conventions

Enable nullable reference types and implicit usings for a clean experience:

```xml title="MyService.csproj"
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
  </PropertyGroup>
</Project>
```

---

## Next steps

- [Your First Web App](your-first-web-app.md) -- follow a hands-on tutorial
- [Module System](../architecture/module-system.md) -- learn how `[HardenedModule]` composition works
- [Dependency Injection](../architecture/dependency-injection.md) -- understand compile-time DI in depth
