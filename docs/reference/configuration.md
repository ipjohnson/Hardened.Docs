# Configuration Reference

This page is a comprehensive reference for the Hardened configuration system, including `[ConfigurationModel]`, environment variable binding, `IAppConfig`, and all known configuration models across the framework packages.

---

## How Configuration Models Work

Hardened uses source generators to create compile-time configuration. You define a configuration model as an interface or a partial class decorated with `[ConfigurationModel]`, and the source generator produces:

1. A **concrete implementation class** with backing fields and property accessors
2. **Environment variable binding** for properties decorated with `[FromEnvironmentVariable]`
3. **DI registration** as a singleton in the service container
4. **Type conversion** for non-string properties bound to environment variables

The generated configuration is immutable after startup. All values are resolved during application initialization in a well-defined order.

---

## Defining Configuration Models

### Interface-Based Models

The most common pattern. Define an interface with read-only properties:

```csharp
using Hardened.Shared.Runtime.Attributes;

[ConfigurationModel]
public interface IDatabaseConfig {
    [FromEnvironmentVariable("DB_CONNECTION_STRING")]
    string ConnectionString { get; }

    [FromEnvironmentVariable("DB_MAX_POOL_SIZE")]
    int MaxPoolSize { get; }

    int CommandTimeoutSeconds { get; }
}
```

The source generator creates a concrete class (e.g., `DatabaseConfig`) that:

- Implements `IDatabaseConfig`
- Reads `DB_CONNECTION_STRING` and `DB_MAX_POOL_SIZE` from environment variables at startup
- Provides type-default values for properties without `[FromEnvironmentVariable]` (e.g., `0` for `int`, `null` for `string`)
- Registers both `IDatabaseConfig` and the concrete type as singletons in DI

**Injecting the configuration:**

```csharp
[TransientService(As = typeof(IOrderRepository))]
public class SqlOrderRepository : IOrderRepository {
    private readonly IDatabaseConfig _config;

    public SqlOrderRepository(IDatabaseConfig config) {
        _config = config;
    }
}
```

### Partial-Class Models

For cases where you need default values, complex types, or factory functions, use a partial class with private fields:

```csharp
using Hardened.Shared.Runtime.Attributes;

[ConfigurationModel]
public partial class CanaryConfig {
    [FromEnvironmentVariable("CANARY_DATA_TABLE")]
    private string _dynamoDataTable = "canary-data-table";

    [FromEnvironmentVariable("SEND_TO_CW_METRICS")]
    private bool _sendMetricsToCloudWatch = true;

    [FromEnvironmentVariable("AWS_REGION")]
    private string _region = "us-west-2";

    private int _retentionDays = 180;

    private Func<IServiceProvider, IAmazonSQS> _sqsClientProvider =
        _ => new AmazonSQSClient();
}
```

The source generator creates public properties from the private fields:

- `_dynamoDataTable` becomes `DynamoDataTable` with a public getter and setter
- The field initializer (`"canary-data-table"`) becomes the default value
- `[FromEnvironmentVariable]` overrides the default when the environment variable is set
- Complex types like `Func<IServiceProvider, IAmazonSQS>` are supported as configuration values

!!! info "Naming Convention"
    Private fields following the `_camelCase` convention are converted to `PascalCase` public properties. For example, `_maxRetries` becomes `MaxRetries`.

---

## Configuration Attributes

### [ConfigurationModel]

Mark an interface or partial class as a configuration model.

**Namespace:** `Hardened.Shared.Runtime.Attributes`

```csharp
public class ConfigurationModelAttribute : Attribute { }
```

---

### [FromEnvironmentVariable]

Bind a property or field to an environment variable. The source generator reads the value from `System.Environment.GetEnvironmentVariable()` at startup.

**Namespace:** `Hardened.Shared.Runtime.Attributes`

```csharp
public class FromEnvironmentVariableAttribute : Attribute {
    public FromEnvironmentVariableAttribute(string environmentVariable) {
        EnvironmentVariable = environmentVariable;
    }
    public string EnvironmentVariable { get; }
}
```

**Supported type conversions:**

| Target Type | Conversion |
|---|---|
| `string` | Direct assignment |
| `int` | `int.Parse()` |
| `bool` | `bool.Parse()` (accepts "true"/"false", case-insensitive) |
| `double` | `double.Parse()` |
| `long` | `long.Parse()` |

---

### [HideConfigurationField]

Exclude a property from the generated implementation. The property will exist in the interface but not in the generated class. Provide its value through `IAppConfig.ProvideValue` or a custom `IConfigurationPackage`.

**Namespace:** `Hardened.Shared.Runtime.Attributes`

```csharp
public class HideConfigurationFieldAttribute : Attribute { }
```

```csharp
[ConfigurationModel]
public interface IAppSettings {
    string AppName { get; }

    [HideConfigurationField]
    string ComputedVersion { get; }  // Not generated; must be provided
}
```

---

### [ConfigurationProvider]

Mark a class as a configuration provider that participates in the configuration resolution pipeline.

**Namespace:** `Hardened.Shared.Runtime.Attributes`

```csharp
public class ConfigurationProviderAttribute : Attribute { }
```

---

## IAppConfig API

`IAppConfig` is the primary programmatic API for configuring values at startup. It is available in the static `ConfigureApplication` method of your `[HardenedModule]` partial class.

### Full Interface

```csharp
namespace Hardened.Shared.Runtime.Configuration;

public interface IAppConfig {
    IAppConfig ProvideValue<TInterface, TImpl>(
        Func<IHardenedEnvironment, TImpl> valueProvider)
        where TImpl : class, TInterface;

    IAppConfig Amend<TImpl>(
        Action<TImpl> amendAction, string environment = "")
        where TImpl : class;

    IAppConfig Amend<TImpl>(
        Func<IHardenedEnvironment, TImpl, TImpl> amendFunc)
        where TImpl : class;
}
```

### ProvideValue

Supply a complete configuration instance for a given interface/implementation pair. The factory function receives `IHardenedEnvironment` so you can vary the value by environment.

```csharp
appConfig.ProvideValue<IDatabaseConfig, DatabaseConfig>(env => new DatabaseConfig {
    ConnectionString = "Server=localhost;Database=mydb",
    MaxPoolSize = env.Matches("Production") ? 50 : 5,
    CommandTimeoutSeconds = 30
});
```

### Amend (Action)

Modify an existing configuration value in place. Optionally restrict the amendment to a specific environment.

```csharp
// Amend for all environments
appConfig.Amend<DatabaseConfig>(config => {
    config.CommandTimeoutSeconds = 60;
});

// Amend only for Production
appConfig.Amend<DatabaseConfig>(config => {
    config.MaxPoolSize = 100;
}, environment: "Production");
```

### Amend (Func)

Modify configuration with access to the environment. Return the (possibly new) configuration instance.

```csharp
appConfig.Amend<DatabaseConfig>((env, config) => {
    if (env.Matches("Production")) {
        config.MaxPoolSize = 100;
    }
    return config;
});
```

### Chaining

All `IAppConfig` methods return `IAppConfig`, enabling fluent chaining:

```csharp
public partial class Application {
    private static void ConfigureApplication(IAppConfig appConfig) {
        appConfig
            .Amend<DatabaseConfig>(config => {
                config.MaxPoolSize = 10;
                config.CommandTimeoutSeconds = 30;
            })
            .Amend<DatabaseConfig>(config => {
                config.MaxPoolSize = 100;
            }, environment: "Production")
            .Amend<ApiConfig>(config => {
                config.TimeoutMs = 5000;
            });
    }
}
```

---

## IConfigurationPackage

`IConfigurationPackage` bundles configuration providers and amenders into a reusable unit. This is the preferred pattern for library modules that need to supply default configuration.

### Full Interface

```csharp
namespace Hardened.Shared.Runtime.Configuration;

public interface IConfigurationPackage {
    IEnumerable<IConfigurationValueProvider> ConfigurationValueProviders(
        IHardenedEnvironment env);
    IEnumerable<IConfigurationValueAmender> ConfigurationValueAmenders(
        IHardenedEnvironment env);
}
```

### Supporting Interfaces

**IConfigurationValueProvider** -- supplies the initial value for a configuration type:

```csharp
public interface IConfigurationValueProvider {
    Type InterfaceType { get; }
    Type ImplementationType { get; }
    object ProvideValue(IHardenedEnvironment environment,
        Action<IHardenedEnvironment, object> amender);
}
```

**IConfigurationValueAmender** -- modifies a configuration value after it has been created:

```csharp
public interface IConfigurationValueAmender {
    object ApplyConfiguration(IHardenedEnvironment environment, object configurationValue);
}
```

### Usage

Register a configuration package with `[SingletonService]`:

```csharp
[SingletonService(As = typeof(IConfigurationPackage))]
public class MyLibraryConfig : IConfigurationPackage {
    public IEnumerable<IConfigurationValueProvider> ConfigurationValueProviders(
        IHardenedEnvironment env) {
        yield break;
    }

    public IEnumerable<IConfigurationValueAmender> ConfigurationValueAmenders(
        IHardenedEnvironment env) {
        yield break;
    }
}
```

---

## Configuration Resolution Order

Configuration values are resolved in the following order, with later values overriding earlier ones:

| Priority | Source | Description |
|---|---|---|
| 1 | Type defaults | `0` for `int`, `null` for `string`, field initializers for partial classes |
| 2 | `[FromEnvironmentVariable]` | Values read from environment variables at startup |
| 3 | `IConfigurationPackage` providers | Initial values from registered configuration packages |
| 4 | `IAppConfig.ProvideValue` | Programmatic value providers in `ConfigureApplication` |
| 5 | `IAppConfig.Amend` | Programmatic amendments (applied sequentially in registration order) |

!!! note
    If multiple `Amend` calls target the same configuration type, they are applied in the order they were registered. Later amendments see the results of earlier ones.

---

## Known Configuration Models

The following configuration models are defined across the Hardened ecosystem. Each is automatically registered as a singleton when its package is included.

### CanaryConfigurationModel

**Package:** `Hardened.Amz.Canaries.Runtime`
**Namespace:** `Hardened.Amz.Canaries.Runtime.Configuration`

Configuration for the canary testing framework.

| Field / Property | Type | Default | Environment Variable | Description |
|---|---|---|---|---|
| `DynamoDataTable` | `string` | `"canary-data-table"` | `CANARY_DATA_TABLE` | DynamoDB table for canary run data. |
| `SendMetricsToCloudWatch` | `bool` | `true` | `SEND_TO_CW_METRICS` | Whether to publish metrics to CloudWatch. |
| `LogGroupPrefix` | `string` | `"/canary/"` | `LOG_GROUP_PREFIX` | CloudWatch log group prefix. |
| `SqsInvokeQueue` | `string` | `"sqs-canary-queue"` | `SQS_CANARY_QUEUE` | SQS queue name for canary invocations. |
| `Region` | `string` | `"us-west-2"` | `AWS_REGION` | AWS region. |
| `AccountId` | `string` | `"account-id"` | `AWS_ACCOUNT_ID` | AWS account ID. |
| `MetricsNamespace` | `string` | `"canary-metrics"` | `METRICS_NAMESPACE` | CloudWatch metrics namespace. |
| `EnableMetrics` | `bool` | `true` | `ENABLE_METRICS` | Master toggle for metrics reporting. |
| `RetentionDays` | `int` | `180` | -- | Data retention period in days. |
| `SqsClientProvider` | `Func<IServiceProvider, IAmazonSQS>` | `new AmazonSQSClient()` | -- | Factory for the SQS client. |
| `CloudWatchProvider` | `Func<IServiceProvider, IAmazonCloudWatch>` | `new AmazonCloudWatchClient()` | -- | Factory for the CloudWatch client. |

```csharp
appConfig.Amend<CanaryConfigurationModel>(config => {
    config.RetentionDays = 90;
    config.MetricsNamespace = "my-canaries";
});
```

---

### DynamoDbClientConfiguration

**Package:** `Hardened.Amz.DynamoDbClient`
**Namespace:** `Hardened.Amz.DynamoDbClient`

Configuration for DynamoDB client instances.

| Field / Property | Type | Default | Description |
|---|---|---|---|
| `DefaultClientConfig` | `Func<IServiceProvider, AmazonDynamoDBConfig>?` | `null` | Factory for the default DynamoDB client configuration. |
| `NamedConfigs` | `Dictionary<string, Func<IServiceProvider, AmazonDynamoDBConfig>>` | `{}` | Named client configurations for multi-region or multi-table setups. |

```csharp
appConfig.Amend<DynamoDbClientConfiguration>(config => {
    config.DefaultClientConfig = _ => new AmazonDynamoDBConfig {
        RegionEndpoint = RegionEndpoint.USWest2
    };
});
```

---

### SqsClientConfiguration

**Package:** `Hardened.Amz.SqsClient`
**Namespace:** `Hardened.SqsClient`

Configuration for the SQS client.

```csharp
appConfig.Amend<SqsClientConfiguration>(config => {
    // Configure SQS client options
});
```

---

### NewtonsoftSerializerConfiguration

**Package:** `Hardened.Requests.Serializers.Newtonsoft`
**Namespace:** `Hardened.Requests.Serializers.Newtonsoft`

Configuration for the Newtonsoft.Json serializer (when used as an alternative to System.Text.Json).

| Field / Property | Type | Default | Description |
|---|---|---|---|
| `SerializerProvider` | `Func<IServiceProvider, JsonSerializer>` | `JsonSerializer.CreateDefault()` | Factory for the JSON serializer instance. |

```csharp
appConfig.Amend<NewtonsoftSerializerConfiguration>(config => {
    config.SerializerProvider = _ => {
        var serializer = JsonSerializer.CreateDefault();
        serializer.Formatting = Formatting.Indented;
        return serializer;
    };
});
```

---

## Complete Example

A full example demonstrating the configuration lifecycle from definition through usage:

```csharp
// 1. Define the configuration model
using Hardened.Shared.Runtime.Attributes;

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
// 2. Configure defaults in the module
[HardenedModule]
[AspNetCoreRuntime.Module]
public partial class Application {
    private static void ConfigureApplication(IAppConfig appConfig) {
        appConfig
            .Amend<ApiConfig>(config => {
                config.MaxRetries = 3;
                config.TimeoutMs = 5000;
            })
            .Amend<ApiConfig>(config => {
                config.TimeoutMs = 15000;
            }, environment: "Production");
    }
}
```

```csharp
// 3. Inject and use the configuration
[TransientService(As = typeof(IExternalApiClient))]
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
            }
            catch when (i < _config.MaxRetries) {
                await Task.Delay(100 * (i + 1));
            }
        }
        throw new InvalidOperationException("Unreachable");
    }
}
```

```csharp
// 4. Test with overridden environment values
using Hardened.Shared.Testing.Attributes;

[HardenedTest]
[EnvironmentValue("API_BASE_URL", "https://test.example.com")]
[EnvironmentValue("API_KEY", "test-key")]
public async Task ApiClient_UsesConfiguration(IApiConfig config) {
    Assert.Equal("https://test.example.com", config.BaseUrl);
    Assert.Equal("test-key", config.ApiKey);
    Assert.Equal(3, config.MaxRetries);  // From Amend
}

[HardenedTest]
[EnvironmentName("Production")]
[EnvironmentValue("API_BASE_URL", "https://prod.example.com")]
public async Task Production_HasLongerTimeout(IApiConfig config) {
    Assert.Equal(15000, config.TimeoutMs);  // Production amendment
}
```

---

## Patterns and Best Practices

### Environment-Specific Configuration

Use `IAppConfig.Amend` with the `environment` parameter for simple environment overrides:

```csharp
appConfig
    .Amend<DatabaseConfig>(c => c.MaxPoolSize = 5)  // Default for all
    .Amend<DatabaseConfig>(c => c.MaxPoolSize = 100, environment: "Production");
```

For more complex logic, use the `Func<IHardenedEnvironment, TImpl, TImpl>` overload:

```csharp
appConfig.Amend<DatabaseConfig>((env, config) => {
    if (env.Matches("Production", "Staging")) {
        config.MaxPoolSize = 100;
        config.CommandTimeoutSeconds = 60;
    }
    return config;
});
```

### Factory Properties

Use `Func<IServiceProvider, T>` fields in partial-class models for dependencies that require the service provider:

```csharp
[ConfigurationModel]
public partial class MessagingConfig {
    private Func<IServiceProvider, IAmazonSQS> _sqsClientProvider =
        _ => new AmazonSQSClient();

    private Func<IServiceProvider, IAmazonSNS> _snsClientProvider =
        _ => new AmazonSimpleNotificationServiceClient();
}
```

### Library Configuration

Libraries should define their own `[ConfigurationModel]` and expose defaults that consuming applications can override:

```csharp
// In the library
[ConfigurationModel]
public interface IMyLibraryConfig {
    int CacheTtlSeconds { get; }
    string ServiceEndpoint { get; }
}

// In the consuming application
appConfig.Amend<MyLibraryConfig>(config => {
    config.CacheTtlSeconds = 300;
    config.ServiceEndpoint = "https://api.example.com";
});
```

### Testing Configuration

Use `[EnvironmentValue]` and `[EnvironmentName]` attributes to control configuration in tests:

```csharp
// Override specific environment variables
[HardenedTest]
[EnvironmentValue("DB_CONNECTION_STRING", "Server=localhost;Database=test")]
public async Task Test_WithDatabase(IDatabaseConfig config) {
    Assert.Equal("Server=localhost;Database=test", config.ConnectionString);
}

// Test environment-specific behavior
[HardenedTest]
[EnvironmentName("Production")]
public async Task Production_HasHigherPoolSize(IDatabaseConfig config) {
    Assert.Equal(100, config.MaxPoolSize);
}
```

---

## Related Pages

- [Attribute Reference](attributes.md) -- all attributes including `[ConfigurationModel]` and `[FromEnvironmentVariable]`
- [Interfaces Reference](interfaces.md) -- `IAppConfig`, `IConfigurationPackage`, and related interfaces
- [Dependency Injection](../framework/shared/dependency-injection.md) -- how `[Expose]` registers services
- [Architecture Overview](../architecture/overview.md) -- how configuration fits into the compile-time architecture
