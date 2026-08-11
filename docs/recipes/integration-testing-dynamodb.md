# Recipe: Integration Testing with DynamoDB

Build integration tests that run against a real DynamoDB instance using Testcontainers and Hardened's `[LocalDynamoDb]` attribute.

**What you will build:**

- A custom test attribute extending `LocalDynamoDbAttribute` with table setup
- Integration tests using `IDynamoDbClientProvider`
- Table creation and cleanup between tests
- A service that reads and writes DynamoDB items, tested end-to-end

---

## Prerequisites

- [.NET 8 SDK](https://dotnet.microsoft.com/download/dotnet/8.0) or later
- [NuGet configured for GitHub Packages](../getting-started/nuget-setup.md)
- [Docker](https://www.docker.com/) installed and running (required for Testcontainers)

---

## Project Setup

### Application project

```bash
dotnet new classlib -n UserService
cd UserService
dotnet add package Hardened.Amz.Function.Lambda.SourceGenerator --prerelease
dotnet add package Hardened.Amz.DynamoDbClient --prerelease
```

### Test project

```bash
cd ..
dotnet new xunit -n UserService.Tests
cd UserService.Tests
dotnet add reference ../UserService/UserService.csproj
dotnet add package Hardened.Amz.Function.Lambda.Testing --prerelease
dotnet add package Hardened.Amz.DynamoDbClient.Testing --prerelease
```

---

## Complete Code

### Application

```csharp title="UserService/Models/UserRecord.cs"
public class UserRecord
{
    public string UserId { get; set; } = string.Empty;
    public string Email { get; set; } = string.Empty;
    public string DisplayName { get; set; } = string.Empty;
    public DateTime CreatedAt { get; set; }
    public DateTime? LastLoginAt { get; set; }
}
```

```csharp title="UserService/Services/IUserRepository.cs"
public interface IUserRepository
{
    Task<UserRecord?> GetById(string userId);
    Task<List<UserRecord>> GetAll();
    Task Put(UserRecord user);
    Task Delete(string userId);
}
```

```csharp title="UserService/Services/UserRepository.cs"
using Amazon.DynamoDBv2;
using Amazon.DynamoDBv2.Model;
using Hardened.Amz.DynamoDbClient;
using DependencyModules.Runtime.Attributes;

[TransientService]
public class UserRepository : IUserRepository
{
    private readonly IDynamoDbClientProvider _clientProvider;
    private const string TableName = "Users";

    public UserRepository(IDynamoDbClientProvider clientProvider)
    {
        _clientProvider = clientProvider;
    }

    public async Task<UserRecord?> GetById(string userId)
    {
        var client = _clientProvider.GetClient();

        var response = await client.GetItemAsync(new GetItemRequest
        {
            TableName = TableName,
            Key = new Dictionary<string, AttributeValue>
            {
                ["UserId"] = new AttributeValue { S = userId }
            }
        });

        if (!response.IsItemSet)
            return null;

        return MapToUserRecord(response.Item);
    }

    public async Task<List<UserRecord>> GetAll()
    {
        var client = _clientProvider.GetClient();

        var response = await client.ScanAsync(new ScanRequest
        {
            TableName = TableName
        });

        return response.Items.Select(MapToUserRecord).ToList();
    }

    public async Task Put(UserRecord user)
    {
        var client = _clientProvider.GetClient();

        await client.PutItemAsync(new PutItemRequest
        {
            TableName = TableName,
            Item = new Dictionary<string, AttributeValue>
            {
                ["UserId"] = new AttributeValue { S = user.UserId },
                ["Email"] = new AttributeValue { S = user.Email },
                ["DisplayName"] = new AttributeValue
                {
                    S = user.DisplayName
                },
                ["CreatedAt"] = new AttributeValue
                {
                    S = user.CreatedAt.ToString("O")
                },
                ["LastLoginAt"] = user.LastLoginAt.HasValue
                    ? new AttributeValue
                    {
                        S = user.LastLoginAt.Value.ToString("O")
                    }
                    : new AttributeValue { NULL = true }
            }
        });
    }

    public async Task Delete(string userId)
    {
        var client = _clientProvider.GetClient();

        await client.DeleteItemAsync(new DeleteItemRequest
        {
            TableName = TableName,
            Key = new Dictionary<string, AttributeValue>
            {
                ["UserId"] = new AttributeValue { S = userId }
            }
        });
    }

    private static UserRecord MapToUserRecord(
        Dictionary<string, AttributeValue> item)
    {
        return new UserRecord
        {
            UserId = item["UserId"].S,
            Email = item["Email"].S,
            DisplayName = item["DisplayName"].S,
            CreatedAt = DateTime.Parse(item["CreatedAt"].S),
            LastLoginAt = item.ContainsKey("LastLoginAt")
                && !item["LastLoginAt"].NULL
                ? DateTime.Parse(item["LastLoginAt"].S)
                : null
        };
    }
}
```

```csharp title="UserService/Handlers/UserHandler.cs"
using Hardened.Amz.Function.Lambda.Runtime.Attributes;

public class UserHandler
{
    private readonly IUserRepository _userRepository;

    public UserHandler(IUserRepository userRepository)
    {
        _userRepository = userRepository;
    }

    [HardenedFunction("get-user")]
    public async Task<UserRecord?> GetUser(string userId)
    {
        return await _userRepository.GetById(userId);
    }
}
```

```csharp title="UserService/Application.cs"
using Hardened.Shared.Runtime.Attributes;

[HardenedModule]
public partial class Application { }
```

### Test Infrastructure

#### Custom LocalDynamoDb Attribute

The `[LocalDynamoDb]` attribute starts a DynamoDB Local container using Testcontainers. To create tables automatically, extend the attribute and override the `DdbSetup` method:

```csharp title="UserService.Tests/Infrastructure/UserTableSetupAttribute.cs"
using Amazon.DynamoDBv2;
using Amazon.DynamoDBv2.Model;
using Hardened.Amz.DynamoDbClient.Testing;

public class UserTableSetupAttribute : LocalDynamoDbAttribute
{
    protected override async Task DdbSetup(IAmazonDynamoDB client)
    {
        await client.CreateTableAsync(new CreateTableRequest
        {
            TableName = "Users",
            KeySchema = new List<KeySchemaElement>
            {
                new KeySchemaElement
                {
                    AttributeName = "UserId",
                    KeyType = KeyType.HASH
                }
            },
            AttributeDefinitions = new List<AttributeDefinition>
            {
                new AttributeDefinition
                {
                    AttributeName = "UserId",
                    AttributeType = ScalarAttributeType.S
                }
            },
            BillingMode = BillingMode.PAY_PER_REQUEST
        });
    }
}
```

!!! note
    The `DdbSetup` method runs once when the DynamoDB container starts. Use it to create all tables your tests require. The container is shared across all tests in the assembly that use the same attribute, keeping test execution fast.

#### Bootstrap

```csharp title="UserService.Tests/Bootstrap.cs"
using Hardened.Shared.Runtime.Attributes;

[assembly: HardenedTestEntryPoint(typeof(Application))]
```

### Tests

```csharp title="UserService.Tests/UserRepositoryTests.cs"
using Hardened.Amz.DynamoDbClient;
using Hardened.Shared.Runtime.Attributes;

public class UserRepositoryTests
{
    [HardenedTest]
    [UserTableSetup]
    public async Task PutAndGetUser_RoundTrips(
        IUserRepository repository)
    {
        var user = new UserRecord
        {
            UserId = "user-001",
            Email = "alice@example.com",
            DisplayName = "Alice",
            CreatedAt = DateTime.UtcNow
        };

        await repository.Put(user);

        var retrieved = await repository.GetById("user-001");

        Assert.NotNull(retrieved);
        Assert.Equal("alice@example.com", retrieved.Email);
        Assert.Equal("Alice", retrieved.DisplayName);
    }

    [HardenedTest]
    [UserTableSetup]
    public async Task GetById_NonExistent_ReturnsNull(
        IUserRepository repository)
    {
        var result = await repository.GetById("nonexistent");

        Assert.Null(result);
    }

    [HardenedTest]
    [UserTableSetup]
    public async Task GetAll_ReturnsAllUsers(
        IUserRepository repository)
    {
        await repository.Put(new UserRecord
        {
            UserId = "user-001",
            Email = "alice@example.com",
            DisplayName = "Alice",
            CreatedAt = DateTime.UtcNow
        });

        await repository.Put(new UserRecord
        {
            UserId = "user-002",
            Email = "bob@example.com",
            DisplayName = "Bob",
            CreatedAt = DateTime.UtcNow
        });

        var all = await repository.GetAll();

        Assert.True(all.Count >= 2);
        Assert.Contains(all, u => u.UserId == "user-001");
        Assert.Contains(all, u => u.UserId == "user-002");
    }

    [HardenedTest]
    [UserTableSetup]
    public async Task Delete_RemovesUser(
        IUserRepository repository)
    {
        await repository.Put(new UserRecord
        {
            UserId = "user-to-delete",
            Email = "delete@example.com",
            DisplayName = "Delete Me",
            CreatedAt = DateTime.UtcNow
        });

        await repository.Delete("user-to-delete");

        var result = await repository.GetById("user-to-delete");
        Assert.Null(result);
    }

    [HardenedTest]
    [UserTableSetup]
    public async Task Put_UpdatesExistingUser(
        IUserRepository repository)
    {
        var user = new UserRecord
        {
            UserId = "user-update",
            Email = "original@example.com",
            DisplayName = "Original",
            CreatedAt = DateTime.UtcNow
        };

        await repository.Put(user);

        // Update the user
        user.Email = "updated@example.com";
        user.DisplayName = "Updated";
        user.LastLoginAt = DateTime.UtcNow;

        await repository.Put(user);

        var retrieved = await repository.GetById("user-update");
        Assert.NotNull(retrieved);
        Assert.Equal("updated@example.com", retrieved.Email);
        Assert.Equal("Updated", retrieved.DisplayName);
        Assert.NotNull(retrieved.LastLoginAt);
    }
}
```

### Clearing Tables Between Tests

If your tests need a clean table state, use `IDynamoDbClientProvider` to clear the table in a setup step:

```csharp title="UserService.Tests/Infrastructure/DynamoDbTestHelper.cs"
using Amazon.DynamoDBv2;
using Amazon.DynamoDBv2.Model;
using Hardened.Amz.DynamoDbClient;

public static class DynamoDbTestHelper
{
    public static async Task ClearTable(
        IDynamoDbClientProvider clientProvider,
        string tableName,
        string partitionKeyName)
    {
        var client = clientProvider.GetClient();

        var scanResponse = await client.ScanAsync(new ScanRequest
        {
            TableName = tableName,
            ProjectionExpression = partitionKeyName
        });

        foreach (var item in scanResponse.Items)
        {
            await client.DeleteItemAsync(new DeleteItemRequest
            {
                TableName = tableName,
                Key = new Dictionary<string, AttributeValue>
                {
                    [partitionKeyName] = item[partitionKeyName]
                }
            });
        }
    }
}
```

```csharp title="UserService.Tests/UserRepositoryCleanTests.cs"
using Hardened.Amz.DynamoDbClient;
using Hardened.Shared.Runtime.Attributes;

public class UserRepositoryCleanTests
{
    [HardenedTest]
    [UserTableSetup]
    public async Task CountUsers_WithCleanTable(
        IUserRepository repository,
        IDynamoDbClientProvider clientProvider)
    {
        // Clear the table before this test
        await DynamoDbTestHelper.ClearTable(
            clientProvider, "Users", "UserId");

        // Now we know the table is empty
        var all = await repository.GetAll();
        Assert.Empty(all);

        // Add exactly one user
        await repository.Put(new UserRecord
        {
            UserId = "only-user",
            Email = "only@example.com",
            DisplayName = "Only User",
            CreatedAt = DateTime.UtcNow
        });

        all = await repository.GetAll();
        Assert.Single(all);
    }
}
```

---

## Explanation

### How `[LocalDynamoDb]` Works

The `[LocalDynamoDb]` attribute (from `Hardened.Amz.DynamoDbClient.Testing`) uses Testcontainers to:

1. **Start a DynamoDB Local container** -- a Docker container running Amazon's DynamoDB Local
2. **Configure `IDynamoDbClientProvider`** -- automatically points the DynamoDB client to the local container
3. **Share the container** -- a single container instance is shared across all tests in the assembly
4. **Clean up** -- the container is stopped when the test run completes

### Custom Attribute with `DdbSetup`

Extending `LocalDynamoDbAttribute` lets you override the `DdbSetup` method to run one-time initialization:

```csharp
public class MyTableSetupAttribute : LocalDynamoDbAttribute
{
    protected override async Task DdbSetup(IAmazonDynamoDB client)
    {
        // This runs once when the container starts.
        // Create all tables your tests need here.
        await client.CreateTableAsync(/* ... */);
    }
}
```

!!! tip
    Create one custom attribute per test project (or per set of related tables). The `DdbSetup` method runs once per container lifecycle, so all table creation should be done here rather than in individual tests.

### `IDynamoDbClientProvider`

The `IDynamoDbClientProvider` interface is the standard way to obtain a DynamoDB client in Hardened:

```csharp
var client = _clientProvider.GetClient();
```

In production, this returns a client configured for your AWS region. In tests with `[LocalDynamoDb]`, it automatically returns a client pointing to the local container. Your service code does not need to change between environments.

### Container Sharing

Testcontainers starts the DynamoDB Local container once per test run. All tests that use `[LocalDynamoDb]` (or a derived attribute) share the same container instance. This keeps the test suite fast -- container startup is a one-time cost of a few seconds.

!!! warning
    Because the container is shared, tests that modify the same table may interfere with each other if run in parallel. Use `ClearTable` or unique keys per test if you need isolation.

### Tables with Sort Keys

For tables with both partition and sort keys, extend the setup and cleanup logic:

```csharp title="Infrastructure/OrderTableSetupAttribute.cs"
using Amazon.DynamoDBv2;
using Amazon.DynamoDBv2.Model;
using Hardened.Amz.DynamoDbClient.Testing;

public class OrderTableSetupAttribute : LocalDynamoDbAttribute
{
    protected override async Task DdbSetup(IAmazonDynamoDB client)
    {
        await client.CreateTableAsync(new CreateTableRequest
        {
            TableName = "Orders",
            KeySchema = new List<KeySchemaElement>
            {
                new KeySchemaElement
                {
                    AttributeName = "CustomerId",
                    KeyType = KeyType.HASH
                },
                new KeySchemaElement
                {
                    AttributeName = "OrderId",
                    KeyType = KeyType.RANGE
                }
            },
            AttributeDefinitions = new List<AttributeDefinition>
            {
                new AttributeDefinition
                {
                    AttributeName = "CustomerId",
                    AttributeType = ScalarAttributeType.S
                },
                new AttributeDefinition
                {
                    AttributeName = "OrderId",
                    AttributeType = ScalarAttributeType.S
                }
            },
            BillingMode = BillingMode.PAY_PER_REQUEST
        });
    }
}
```

---

## Running the Tests

Make sure Docker is running, then:

```bash
dotnet test
```

The first run will pull the DynamoDB Local Docker image, which takes a few seconds. Subsequent runs reuse the cached image.

!!! note
    If you see connection errors, verify Docker is running with `docker ps`. The Testcontainers library manages the container lifecycle automatically -- you do not need to start it manually.

---

## Next Steps

- [DynamoDB Client](../aws/clients/dynamodb.md) -- advanced client patterns
- [DynamoDB Testing](../aws/clients/dynamodb-testing.md) -- testing utilities reference
- [DDB Stream Processor](ddb-stream-processor.md) -- process DynamoDB Stream events
