# Installation

Hardened is organized into focused packages so you only pull in what you need. All packages are hosted on [GitHub Packages](nuget-setup.md) and use **pre-release versioning** (`0.1.0-Alpha*` for framework packages, `1.0.*` for AWS and canary packages).

!!! note
    Make sure you have [configured the GitHub Packages NuGet source](nuget-setup.md) before running any install commands.

---

## Package map

Choose the packages that match your project type:

### Web applications

| Package | Description |
|---|---|
| `Hardened.Web.AspNetCore.SourceGenerator` | ASP.NET Core web API — brings in all framework dependencies |

```bash
dotnet add package Hardened.Web.AspNetCore.SourceGenerator --prerelease
```

### Libraries and modules

| Package | Description |
|---|---|
| `Hardened.Library.SourceGenerator` | Reusable library/module with DI and configuration |

```bash
dotnet add package Hardened.Library.SourceGenerator --prerelease
```

### Console applications

| Package | Description |
|---|---|
| `Hardened.Console.SourceGenerator` | Console app with DI, configuration, and lifecycle |

```bash
dotnet add package Hardened.Console.SourceGenerator --prerelease
```

### AWS Lambda

| Use case | Package |
|---|---|
| Lambda function (plain) | `Hardened.Amz.Function.Lambda.SourceGenerator` |
| Lambda web API (API Gateway) | `Hardened.Amz.Web.Lambda.SourceGenerator` |
| Lambda DynamoDB Stream processor | `Hardened.Amz.Function.DDB.Runtime` |
| Lambda SQS processor | `Hardened.Amz.Function.Sqs.Runtime` |

=== "Function"

    ```bash
    dotnet add package Hardened.Amz.Function.Lambda.SourceGenerator --prerelease
    ```

=== "Web (API Gateway)"

    ```bash
    dotnet add package Hardened.Amz.Web.Lambda.SourceGenerator --prerelease
    ```

=== "DynamoDB Stream"

    ```bash
    dotnet add package Hardened.Amz.Function.DDB.Runtime --prerelease
    ```

=== "SQS"

    ```bash
    dotnet add package Hardened.Amz.Function.Sqs.Runtime --prerelease
    ```

### AWS Clients

| Package | Description |
|---|---|
| `Hardened.Amz.DynamoDbClient` | DynamoDB client with `IDynamoDbClientProvider` |
| `Hardened.Amz.SqsClient` | SQS client with `ISqsClient` |

```bash
dotnet add package Hardened.Amz.DynamoDbClient --prerelease
dotnet add package Hardened.Amz.SqsClient --prerelease
```

### Canaries

| Package | Description |
|---|---|
| `Hardened.Amz.Canaries.Runtime` | Canary framework with `[HardenedCanary]` attribute |

```bash
dotnet add package Hardened.Amz.Canaries.Runtime --prerelease
```

---

## Testing packages

Add the matching testing package alongside your runtime package:

| Project type | Testing package |
|---|---|
| Web API | `Hardened.Web.Testing` |
| Lambda function | `Hardened.Amz.Function.Lambda.Testing` |
| DynamoDB Stream | `Hardened.Amz.Function.DDB.Testing` |
| SQS processor | `Hardened.Amz.Function.Sqs.Testing` |
| DynamoDB integration tests | `Hardened.Amz.DynamoDbClient.Testing` |

=== "Web Testing"

    ```bash
    dotnet add package Hardened.Web.Testing --prerelease
    ```

=== "Lambda Testing"

    ```bash
    dotnet add package Hardened.Amz.Function.Lambda.Testing --prerelease
    ```

=== "DynamoDB Stream Testing"

    ```bash
    dotnet add package Hardened.Amz.Function.DDB.Testing --prerelease
    ```

=== "SQS Testing"

    ```bash
    dotnet add package Hardened.Amz.Function.Sqs.Testing --prerelease
    ```

=== "DynamoDB Integration Testing"

    ```bash
    dotnet add package Hardened.Amz.DynamoDbClient.Testing --prerelease
    ```

!!! tip
    `Hardened.Amz.DynamoDbClient.Testing` uses [Testcontainers](https://dotnet.testcontainers.org/) to spin up a local DynamoDB instance for integration tests. Make sure Docker is available in your test environment.

---

## Verifying installation

After adding packages, restore and build to trigger source generation:

```bash
dotnet restore
dotnet build
```

If the build succeeds, the source generators are active and producing code. You can inspect the generated files under `obj/Debug/net8.0/generated/` in your project directory.

!!! warning
    If you see `CS0246: The type or namespace name 'HardenedModule' could not be found`, verify that the source generator package was added correctly and that `dotnet restore` completed without authentication errors.

---

## Next steps

Ready to build something? Pick a tutorial:

- [Your First Web App](your-first-web-app.md) -- build an ASP.NET Core API with Hardened
- [Your First Lambda](your-first-lambda.md) -- build an AWS Lambda function with Hardened
- [Your First Canary](your-first-canary.md) -- build a canary health check with Hardened
