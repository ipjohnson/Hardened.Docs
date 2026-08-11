# Recipe: Web API CRUD

Build a complete CRUD (Create, Read, Update, Delete) REST API for managing Todo items using Hardened's compile-time routing and dependency injection.

**What you will build:**

- A `TodoController` with full CRUD endpoints under `/api/todos`
- A `TodoService` registered as a singleton via compile-time DI
- Automatic parameter binding for path tokens, query strings, and request bodies
- Custom status code handling for missing resources

---

## Prerequisites

- [.NET 8 SDK](https://dotnet.microsoft.com/download/dotnet/8.0) or later
- [NuGet configured for GitHub Packages](../getting-started/nuget-setup.md)

---

## Project Setup

```bash
dotnet new web -n TodoApi
cd TodoApi
dotnet add package Hardened.Web.AspNetCore.Runtime --prerelease
dotnet add package Hardened.Web.SourceGenerator --prerelease
dotnet add package Hardened.DependencyModules.SourceGenerator --prerelease
```

---

## Complete Code

### Models

```csharp title="Models/Todo.cs"
public class Todo
{
    public int Id { get; set; }
    public string Title { get; set; } = string.Empty;
    public string? Description { get; set; }
    public bool IsComplete { get; set; }
    public DateTime CreatedAt { get; set; }
    public DateTime? CompletedAt { get; set; }
}
```

```csharp title="Models/CreateTodoRequest.cs"
public class CreateTodoRequest
{
    public string Title { get; set; } = string.Empty;
    public string? Description { get; set; }
}
```

```csharp title="Models/UpdateTodoRequest.cs"
public class UpdateTodoRequest
{
    public string? Title { get; set; }
    public string? Description { get; set; }
    public bool? IsComplete { get; set; }
}
```

### Service

```csharp title="Services/ITodoService.cs"
public interface ITodoService
{
    List<Todo> GetAll();
    Todo? GetById(int id);
    Todo Create(CreateTodoRequest request);
    Todo? Update(int id, UpdateTodoRequest request);
    bool Delete(int id);
}
```

```csharp title="Services/TodoService.cs"
using DependencyModules.Runtime.Attributes;

[SingletonService]
public class TodoService : ITodoService
{
    private readonly List<Todo> _todos = new();
    private int _nextId = 1;

    public List<Todo> GetAll()
    {
        return _todos.ToList();
    }

    public Todo? GetById(int id)
    {
        return _todos.FirstOrDefault(t => t.Id == id);
    }

    public Todo Create(CreateTodoRequest request)
    {
        var todo = new Todo
        {
            Id = _nextId++,
            Title = request.Title,
            Description = request.Description,
            IsComplete = false,
            CreatedAt = DateTime.UtcNow
        };

        _todos.Add(todo);
        return todo;
    }

    public Todo? Update(int id, UpdateTodoRequest request)
    {
        var todo = _todos.FirstOrDefault(t => t.Id == id);
        if (todo == null) return null;

        if (request.Title != null)
            todo.Title = request.Title;

        if (request.Description != null)
            todo.Description = request.Description;

        if (request.IsComplete.HasValue)
        {
            todo.IsComplete = request.IsComplete.Value;
            todo.CompletedAt = request.IsComplete.Value ? DateTime.UtcNow : null;
        }

        return todo;
    }

    public bool Delete(int id)
    {
        var todo = _todos.FirstOrDefault(t => t.Id == id);
        if (todo == null) return false;

        _todos.Remove(todo);
        return true;
    }
}
```

!!! note
    `[SingletonService]` ensures only one instance of `TodoService` exists for the lifetime of the application. This is important here because the service holds the in-memory list of todos. In a real application, you would use a database and the service could be `[TransientService]` or `[ScopedService]` instead.

### Controller

```csharp title="Controllers/TodoController.cs"
using Hardened.Web.Runtime.Attributes;

[BasePath("/api/todos")]
public class TodoController
{
    private readonly ITodoService _todoService;

    public TodoController(ITodoService todoService)
    {
        _todoService = todoService;
    }

    [Get]
    public List<Todo> GetAll()
    {
        return _todoService.GetAll();
    }

    [Get("{id}", NullReturnStatus = 404)]
    public Todo? GetById(int id)
    {
        return _todoService.GetById(id);
    }

    [Post(SuccessStatus = 201)]
    public Todo Create([FromBody] CreateTodoRequest request)
    {
        return _todoService.Create(request);
    }

    [Put("{id}", NullReturnStatus = 404)]
    public Todo? Update(int id, [FromBody] UpdateTodoRequest request)
    {
        return _todoService.Update(id, request);
    }

    [Delete("{id}", SuccessStatus = 204)]
    public bool Delete(int id)
    {
        return _todoService.Delete(id);
    }
}
```

### Application Module

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

### Entry Point

```csharp title="Program.cs"
var builder = Application.CreateBuilder(args);
var app = builder.Build();

app.UseHardened();
app.Run();
```

---

## Explanation

### Routing with `[BasePath]`

The `[BasePath("/api/todos")]` attribute on the controller class prefixes all route paths within that controller. The individual route attributes then specify the relative path:

| Attribute | Resulting Route |
|---|---|
| `[Get]` | `GET /api/todos` |
| `[Get("{id}")]` | `GET /api/todos/{id}` |
| `[Post]` | `POST /api/todos` |
| `[Put("{id}")]` | `PUT /api/todos/{id}` |
| `[Delete("{id}")]` | `DELETE /api/todos/{id}` |

### Path Token Binding

The `{id}` token in the route path is automatically bound to the `int id` parameter by name. The source generator handles type conversion from the string path segment to `int`.

### Body Binding with `[FromBody]`

Complex types in `[Post]` and `[Put]` handlers are annotated with `[FromBody]` to indicate they should be deserialized from the JSON request body. For `[Post]` and `[Put]` methods, complex type parameters are automatically treated as body parameters, so the `[FromBody]` attribute is optional but makes the intent explicit.

### Status Code Customization

| Property | Purpose |
|---|---|
| `NullReturnStatus = 404` | If the handler returns `null`, respond with a 404 instead of 200 |
| `SuccessStatus = 201` | On success, return 201 Created instead of the default 200 |
| `SuccessStatus = 204` | On success, return 204 No Content (useful for delete operations) |

!!! tip
    `NullReturnStatus` is particularly useful for `GetById` patterns where a missing resource should return 404 rather than a null JSON body with a 200 status.

---

## Testing

Create a test project:

```bash
cd ..
dotnet new xunit -n TodoApi.Tests
cd TodoApi.Tests
dotnet add reference ../TodoApi/TodoApi.csproj
dotnet add package Hardened.Web.Testing --prerelease
```

### Bootstrap

```csharp title="Bootstrap.cs"
using Hardened.Web.Testing;
using Hardened.Shared.Runtime.Attributes;

[assembly: WebTesting]
[assembly: HardenedTestEntryPoint(typeof(Application))]
```

### Tests

```csharp title="TodoControllerTests.cs"
using Hardened.Shared.Runtime.Attributes;
using Hardened.Web.Testing;

public class TodoControllerTests
{
    [HardenedTest]
    public async Task CreateTodo_Returns201(ITestWebApp testWebApp)
    {
        var request = new CreateTodoRequest
        {
            Title = "Buy groceries",
            Description = "Milk, eggs, bread"
        };

        var response = await testWebApp.Post("/api/todos", request);

        response.Assert.StatusCode(201);

        var todo = response.Deserialize<Todo>();
        Assert.Equal("Buy groceries", todo.Title);
        Assert.False(todo.IsComplete);
    }

    [HardenedTest]
    public async Task GetTodo_ReturnsExistingTodo(ITestWebApp testWebApp)
    {
        // Create a todo first
        var createRequest = new CreateTodoRequest { Title = "Test todo" };
        var createResponse = await testWebApp.Post("/api/todos", createRequest);
        var created = createResponse.Deserialize<Todo>();

        // Retrieve it
        var response = await testWebApp.Get($"/api/todos/{created.Id}");

        response.Assert.Ok();

        var todo = response.Deserialize<Todo>();
        Assert.Equal("Test todo", todo.Title);
    }

    [HardenedTest]
    public async Task GetTodo_Returns404ForMissing(ITestWebApp testWebApp)
    {
        var response = await testWebApp.Get("/api/todos/9999");

        response.Assert.StatusCode(404);
    }

    [HardenedTest]
    public async Task UpdateTodo_ModifiesExistingTodo(ITestWebApp testWebApp)
    {
        // Create a todo
        var createRequest = new CreateTodoRequest { Title = "Original title" };
        var createResponse = await testWebApp.Post("/api/todos", createRequest);
        var created = createResponse.Deserialize<Todo>();

        // Update it
        var updateRequest = new UpdateTodoRequest
        {
            Title = "Updated title",
            IsComplete = true
        };

        var response = await testWebApp.Put(
            $"/api/todos/{created.Id}", updateRequest);

        response.Assert.Ok();

        var updated = response.Deserialize<Todo>();
        Assert.Equal("Updated title", updated.Title);
        Assert.True(updated.IsComplete);
        Assert.NotNull(updated.CompletedAt);
    }

    [HardenedTest]
    public async Task DeleteTodo_Returns204(ITestWebApp testWebApp)
    {
        // Create a todo
        var createRequest = new CreateTodoRequest { Title = "To delete" };
        var createResponse = await testWebApp.Post("/api/todos", createRequest);
        var created = createResponse.Deserialize<Todo>();

        // Delete it
        var response = await testWebApp.Delete($"/api/todos/{created.Id}");

        response.Assert.StatusCode(204);

        // Verify it is gone
        var getResponse = await testWebApp.Get($"/api/todos/{created.Id}");
        getResponse.Assert.StatusCode(404);
    }
}
```

Run the tests:

```bash
dotnet test
```

---

## Next Steps

- [Routing](../framework/web/routing.md) -- explore advanced routing patterns
- [Filters](../framework/requests/filters.md) -- add cross-cutting concerns like authentication
- [Lambda Web API](lambda-web-api.md) -- deploy this same API as a serverless Lambda function
