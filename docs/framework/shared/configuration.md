# Configuration

Hardened provides a compile-time configuration system built on source generators. You define configuration as interfaces, and the source generator creates the implementation classes, environment variable binding, and DI registration automatically.

**Package:** `Hardened.Shared.Runtime` (namespace `Hardened.Shared.Runtime.Attributes`, `Hardened.Shared.Runtime.Configuration`)

---

## [ConfigurationModel]

The `[ConfigurationModel]` attribute marks an interface as a configuration model. The source generator creates a concrete implementation class, registers it in the DI container as a singleton, and wires up property defaults and environment variable bindings.

### Definition

```csharp
namespace Hardened.Shared.Runtime.Attributes;

public class ConfigurationModelAttribute : Attribute { }
```

### Basic Usage

```csharp
using Hardened.Shared.Runtime.Attributes;

[ConfigurationModel]
public interface IDatabaseConfig {
    string ConnectionString { get; }
    int MaxPoolSize { get; }
    int CommandTimeoutSeconds { get; }
}
```

The source generator produces a concrete class implementing this interface. The configuration model is automatically registered in the DI container as a singleton, so you can inject it anywhere:

```csharp
[Expose(typeof(IOrderRepository))]
public class SqlOrderRepository : IOrderRepository {
    private readonly IDatabaseConfig _config;

    public SqlOrderRepository(IDatabaseConfig config) {
        _config = config;
    }

    public async Task<Order?> GetById(string id) {
        // Use _config.ConnectionString, _config.MaxPoolSize, etc.
    }
}
```

---

## [FromEnvironmentVariable]

Bind a configuration property to an environment variable. The source generator emits code that reads the value from `System.Environment.GetEnvironmentVariable()` at startup.

### Definition

```csharp
namespace Hardened.Shared.Runtime.Attributes;

public class FromEnvironmentVariableAttribute : Attribute {
    public FromEnvironmentVariableAttribute(string environmentVariable) {
        EnvironmentVariable = environmentVariable;
    }

    public string EnvironmentVariable { get; }
}
```

### Usage

```csharp
[ConfigurationModel]
public interface IDatabaseConfig {
    [FromEnvironmentVariable("DB_CONNECTION_STRING")]
    string ConnectionString { get; }

    [FromEnvironmentVariable("DB_MAX_POOL_SIZE")]
    int MaxPoolSize { get; }

    int CommandTimeoutSeconds { get; }
}
```

When the application starts, `ConnectionString` is populated from the `DB_CONNECTION_STRING` environment variable and `MaxPoolSize` from `DB_MAX_POOL_SIZE`. Properties without `[FromEnvironmentVariable]` use their type's default value unless configured through `IAppConfig`.

!!! tip
    This pattern works especially well with AWS Lambda and containerized deployments where configuration is passed through environment variables.

---

## [HideConfigurationField]

Prevents a property from appearing in the generated configuration implementation. This is useful when you want to define a computed property or a property that is always provided programmatically.

### Definition

```csharp
namespace Hardened.Shared.Runtime.Attributes;

public class HideConfigurationFieldAttribute : Attribute { }
```

### Usage

```csharp
[ConfigurationModel]
public interface IAppSettings {
    string AppName { get; }

    [HideConfigurationField]
    string Version { get; }
}
```

The `Version` property will not be included in the generated implementation class. You would provide its value through a custom `IConfigurationPackage` or `IAppConfig.ProvideValue`.

---

## IAppConfig

`IAppConfig` allows you to programmatically provide or modify configuration values. It is available during application startup and is typically used in module configuration or startup code.

### Definition

```csharp
namespace Hardened.Shared.Runtime.Configuration;

public interface IAppConfig {
    IAppConfig ProvideValue<TInterface, TImpl>(
        Func<IHardenedEnvironment, TImpl> valueProvider)
        where TImpl : class, TInterface;

    IAppConfig Amend<TImpl>(
        Action<TImpl> amendAction,
        string environment = "")
        where TImpl : class;

    IAppConfig Amend<TImpl>(
        Func<IHardenedEnvironment, TImpl, TImpl> amendFunc)
        where TImpl : class;
}
```

### ProvideValue -- Supplying Configuration

Use `ProvideValue` to supply a complete configuration instance:

```csharp
appConfig.ProvideValue<IDatabaseConfig, DatabaseConfig>(env => new DatabaseConfig {
    ConnectionString = "Server=localhost;Database=mydb",
    MaxPoolSize = 10,
    CommandTimeoutSeconds = 30
});
```

The `IHardenedEnvironment` parameter lets you vary configuration by environment:

```csharp
appConfig.ProvideValue<IDatabaseConfig, DatabaseConfig>(env => {
    if (env.Matches("Production")) {
        return new DatabaseConfig {
            ConnectionString = Environment.GetEnvironmentVariable("PROD_DB")!,
            MaxPoolSize = 50
        };
    }
    return new DatabaseConfig {
        ConnectionString = "Server=localhost;Database=dev",
        MaxPoolSize = 5
    };
});
```

### Amend -- Modifying Configuration

Use `Amend` to modify an existing configuration value. This is useful when you want to adjust defaults without replacing the entire configuration:

```csharp
// Amend for all environments
appConfig.Amend<DatabaseConfig>(config => {
    config.CommandTimeoutSeconds = 60;
});

// Amend only for a specific environment
appConfig.Amend<DatabaseConfig>(config => {
    config.MaxPoolSize = 100;
}, environment: "Production");
```

The overload accepting a `Func<IHardenedEnvironment, TImpl, TImpl>` gives you access to the environment:

```csharp
appConfig.Amend<DatabaseConfig>((env, config) => {
    if (env.Matches("Production")) {
        config.MaxPoolSize = 100;
    }
    return config;
});
```

!!! note
    `Amend` calls are applied in order. If multiple modules amend the same configuration, later amendments see the results of earlier ones.

---

## IConfigurationPackage

`IConfigurationPackage` is an interface for bundling configuration providers and amenders into a reusable package. This is the preferred way to organize configuration logic in library modules.

### Definition

```csharp
namespace Hardened.Shared.Runtime.Configuration;

public interface IConfigurationPackage {
    IEnumerable<IConfigurationValueProvider> ConfigurationValueProviders(
        IHardenedEnvironment env);

    IEnumerable<IConfigurationValueAmender> ConfigurationValueAmenders(
        IHardenedEnvironment env);
}
```

### Usage

```csharp
using Hardened.Shared.Runtime.Attributes;
using Hardened.Shared.Runtime.Application;
using Hardened.Shared.Runtime.Configuration;

[Expose(typeof(IConfigurationPackage))]
[Singleton]
public class DatabaseConfigPackage : IConfigurationPackage {
    public IEnumerable<IConfigurationValueProvider> ConfigurationValueProviders(
        IHardenedEnvironment env) {
        // Return providers that supply initial configuration values
        yield break;
    }

    public IEnumerable<IConfigurationValueAmender> ConfigurationValueAmenders(
        IHardenedEnvironment env) {
        // Return amenders that modify configuration values
        yield break;
    }
}
```

!!! tip
    `IConfigurationPackage` is automatically discovered through DI. Register it with `[Expose]` and Hardened will invoke it during configuration setup.

---

## Complete Example

Here is a full example showing the configuration system in action:

```csharp
// 1. Define the configuration interface
[ConfigurationModel]
public interface IApiConfig {
    [FromEnvironmentVariable("API_BASE_URL")]
    string BaseUrl { get; }

    [FromEnvironmentVariable("API_TIMEOUT_MS")]
    int TimeoutMs { get; }

    int MaxRetries { get; }

    [FromEnvironmentVariable("API_KEY")]
    string ApiKey { get; }
}
```

```csharp
// 2. Amend defaults in your module
public partial class Application {
    private static void ConfigureApplication(IAppConfig appConfig) {
        appConfig.Amend<ApiConfig>(config => {
            config.MaxRetries = 3;
            config.TimeoutMs = 5000;
        });

        appConfig.Amend<ApiConfig>(config => {
            config.TimeoutMs = 15000;
        }, environment: "Production");
    }
}
```

```csharp
// 3. Inject and use the configuration
[Expose(typeof(IExternalApiClient))]
public class ExternalApiClient : IExternalApiClient {
    private readonly IApiConfig _config;
    private readonly HttpClient _http;

    public ExternalApiClient(IApiConfig config, HttpClient http) {
        _config = config;
        _http = http;
        _http.BaseAddress = new Uri(_config.BaseUrl);
        _http.Timeout = TimeSpan.FromMilliseconds(_config.TimeoutMs);
    }

    public async Task<ApiResponse> Call(string endpoint) {
        for (int i = 0; i <= _config.MaxRetries; i++) {
            try {
                var response = await _http.GetAsync(endpoint);
                response.EnsureSuccessStatusCode();
                return await response.Content.ReadFromJsonAsync<ApiResponse>();
            } catch when (i < _config.MaxRetries) {
                await Task.Delay(100 * (i + 1));
            }
        }
        throw new InvalidOperationException("Unreachable");
    }
}
```

---

## Configuration Resolution Order

Configuration values are resolved in the following order, with later values overriding earlier ones:

1. Type defaults (e.g., `0` for `int`, `null` for `string`)
2. `[FromEnvironmentVariable]` values
3. `IConfigurationPackage` providers
4. `IAppConfig.ProvideValue` calls
5. `IAppConfig.Amend` calls (applied sequentially)

---

## Related Pages

- [Dependency Injection](dependency-injection.md) -- how `[Expose]` registers services
- [Application Lifecycle](application-lifecycle.md) -- when configuration is initialized
- [Environment](environment.md) -- how `IHardenedEnvironment` drives configuration
- [Architecture: Configuration System](../../architecture/configuration-system.md) -- high-level configuration architecture
