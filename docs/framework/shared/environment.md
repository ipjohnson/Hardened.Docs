# Environment

The `IHardenedEnvironment` interface provides access to the current runtime environment, including the environment name, command-line arguments, and custom key-value data. It drives environment-specific behavior throughout the framework, including `[ForEnvironment]` filtering and configuration amendments.

**Package:** `Hardened.Shared.Runtime` (namespace `Hardened.Shared.Runtime.Application`)

---

## IHardenedEnvironment

### Definition

```csharp
namespace Hardened.Shared.Runtime.Application;

public interface IHardenedEnvironment {
    string Name { get; }
    IReadOnlyList<string> Arguments { get; }
    T? Value<T>(string name, T? defaultValue = default);
    T? CustomData<T>(string name, T? defaultValue = default);
}
```

### Properties

| Member | Type | Description |
|---|---|---|
| `Name` | `string` | The environment name (e.g., `"Development"`, `"Production"`) |
| `Arguments` | `IReadOnlyList<string>` | Command-line arguments passed to the application |

### Methods

| Method | Description |
|---|---|
| `Value<T>(name, default)` | Retrieve a typed value by name, with an optional default |
| `CustomData<T>(name, default)` | Retrieve custom data by name, with an optional default |

---

## Using IHardenedEnvironment

### Injecting the Environment

`IHardenedEnvironment` is registered in the DI container and can be injected into any service:

```csharp
using Hardened.Shared.Runtime.Application;
using Hardened.Shared.Runtime.Attributes;

[Expose(typeof(IFeatureService))]
[Singleton]
public class FeatureService : IFeatureService {
    private readonly IHardenedEnvironment _env;

    public FeatureService(IHardenedEnvironment env) {
        _env = env;
    }

    public bool IsDebugEnabled() {
        return _env.Name == "Development";
    }
}
```

### Reading Values

Use `Value<T>` to read typed values from the environment:

```csharp
// Read a string value with a default
string region = env.Value<string>("AWS_REGION", "us-east-1")!;

// Read an integer value
int port = env.Value<int>("PORT", 8080);

// Read a boolean flag
bool verbose = env.Value<bool>("VERBOSE", false);
```

### Reading Custom Data

`CustomData<T>` accesses custom data that was set during environment construction. This is typically used by hosting layers to pass runtime-specific information:

```csharp
// Read custom data set by the hosting layer
var lambdaContext = env.CustomData<ILambdaContext>("LambdaContext");
```

### Accessing Arguments

The `Arguments` property provides access to the command-line arguments:

```csharp
[Expose(typeof(IStartupService))]
[Singleton]
public class ArgParser : IStartupService {
    private readonly IHardenedEnvironment _env;

    public ArgParser(IHardenedEnvironment env) {
        _env = env;
    }

    public Task<bool> Startup(IServiceProvider rootProvider) {
        if (_env.Arguments.Contains("--dry-run")) {
            // Configure for dry-run mode
        }
        return Task.FromResult(true);
    }
}
```

---

## Extension Methods

Hardened provides extension methods on `IHardenedEnvironment` for common operations.

### Matches

Check if the current environment name matches one or more values. Comparison is **case-insensitive**.

```csharp
public static class IEnvironmentExtensions {
    public static bool Matches(
        this IHardenedEnvironment environment,
        params string[] environments);
}
```

**Usage:**

```csharp
if (env.Matches("Development", "Test")) {
    // Enable detailed error messages
}

if (env.Matches("Production")) {
    // Use production-grade settings
}
```

### MatchesVariable

Check if a named variable equals a specific value. Comparison is **case-insensitive**.

```csharp
public static class IEnvironmentExtensions {
    public static bool MatchesVariable(
        this IHardenedEnvironment environment,
        string variable,
        string value);
}
```

**Usage:**

```csharp
if (env.MatchesVariable("FEATURE_NEW_UI", "true")) {
    // Enable the new UI feature
}

if (env.MatchesVariable("LOG_LEVEL", "debug")) {
    // Configure debug logging
}
```

---

## Environment in Configuration

`IHardenedEnvironment` is passed to configuration methods, enabling environment-specific configuration:

```csharp
[HardenedModule]
public partial class Application {
    private static void ConfigureApplication(IAppConfig appConfig) {
        appConfig.Amend<DatabaseConfig>((env, config) => {
            if (env.Matches("Production")) {
                config.MaxPoolSize = 100;
                config.CommandTimeoutSeconds = 60;
            } else {
                config.MaxPoolSize = 5;
                config.CommandTimeoutSeconds = 10;
            }
            return config;
        });
    }
}
```

---

## Environment in DI Registration

The `[ForEnvironment]` attribute uses the environment name to conditionally register services:

```csharp
[Expose(typeof(IEmailSender))]
[ForEnvironment("Production")]
public class SmtpEmailSender : IEmailSender { }

[Expose(typeof(IEmailSender))]
[ForEnvironment("Development")]
[ForEnvironment("Test")]
public class ConsoleEmailSender : IEmailSender { }
```

See [Dependency Injection](dependency-injection.md#forenvironment-environment-specific-registration) for details.

---

## Environment in IConfigurationPackage

The `IConfigurationPackage` interface receives `IHardenedEnvironment`, allowing packages to return different providers per environment:

```csharp
[Expose(typeof(IConfigurationPackage))]
[Singleton]
public class MyConfigPackage : IConfigurationPackage {
    public IEnumerable<IConfigurationValueProvider> ConfigurationValueProviders(
        IHardenedEnvironment env) {
        if (env.Matches("Production")) {
            yield return new ProductionConfigProvider();
        }
        yield break;
    }

    public IEnumerable<IConfigurationValueAmender> ConfigurationValueAmenders(
        IHardenedEnvironment env) {
        yield break;
    }
}
```

---

## Common Environment Names

While Hardened does not enforce specific environment names, the following conventions are common:

| Name | Typical Use |
|---|---|
| `Development` | Local development with detailed errors and logging |
| `Test` | Automated test execution |
| `Staging` | Pre-production validation |
| `Production` | Live production workloads |

!!! tip
    Use `Matches()` with multiple values when the same behavior applies to several environments: `env.Matches("Development", "Test")`.

---

## Related Pages

- [Dependency Injection](dependency-injection.md) -- `[ForEnvironment]` for conditional registration
- [Configuration](configuration.md) -- environment-aware configuration with `IAppConfig.Amend`
- [Application Lifecycle](application-lifecycle.md) -- how `IHardenedEnvironment` flows through startup
