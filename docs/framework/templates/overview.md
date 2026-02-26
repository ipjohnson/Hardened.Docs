# Templates Overview

Hardened includes a compile-time template engine that uses Mustache-style syntax. Templates are compiled into C# code by the source generator at build time, producing fast, type-safe template rendering with no runtime parsing overhead.

**Packages:** `Hardened.Templates.Abstract`, `Hardened.Templates.Runtime`, `Hardened.Templates.SourceGenerator`

---

## [TemplatePackage]

The `[TemplatePackage]` attribute is applied at the assembly level to enable template compilation for the project. It tells the source generator which file extensions to scan and what token format to use.

### Definition

```csharp
namespace Hardened.Templates.Abstract;

public class TemplatePackageAttribute : Attribute {
    public string Extensions { get; set; } = "html";
    public string Token { get; set; } = "{{TOKEN}}";
}
```

### Properties

| Property | Default | Description |
|---|---|---|
| `Extensions` | `"html"` | File extensions to compile as templates (comma-separated) |
| `Token` | `"{{TOKEN}}"` | Token format for Mustache expressions |

### Usage

```csharp
// In AssemblyInfo.cs or any file
using Hardened.Templates.Abstract;

[assembly: TemplatePackage(Extensions = "html", Token = "{{TOKEN}}")]
```

This tells the source generator to find all `.html` files in the project and compile them into template execution functions.

!!! note
    Template files must be included in the project (e.g., as embedded resources or content items) for the source generator to discover them.

---

## Mustache Syntax

Hardened templates use Mustache-style syntax. The double-brace tokens are resolved at compile time into direct property access code.

### Variable Output

```html
<h1>{{title}}</h1>
<p>Welcome, {{user.name}}!</p>
```

### Sections (Conditionals)

Render a block only if the value is truthy:

```html
{{#isLoggedIn}}
    <p>Welcome back, {{userName}}!</p>
{{/isLoggedIn}}
```

### Inverted Sections

Render a block only if the value is falsy:

```html
{{^isLoggedIn}}
    <p>Please <a href="/login">log in</a>.</p>
{{/isLoggedIn}}
```

### Iteration

Iterate over a collection:

```html
<ul>
{{#items}}
    <li>{{name}} - ${{price}}</li>
{{/items}}
</ul>
```

### Template Helpers

Call registered helpers using the `$` prefix:

```html
<p>{{$formatDate createdAt "yyyy-MM-dd"}}</p>
<p>{{$uppercase title}}</p>
```

See [Helpers](helpers.md) for details on creating custom template helpers.

---

## ITemplateExecutionService

The `ITemplateExecutionService` is the runtime API for executing compiled templates.

### Definition

```csharp
namespace Hardened.Templates.Abstract;

public interface ITemplateExecutionService {
    Task<string> Execute(
        string templateName,
        object templateData,
        IServiceProvider serviceProvider);

    Task Execute(
        string templateName,
        object? templateData,
        IServiceProvider serviceProvider,
        ITemplateOutputWriter writer,
        ITemplateExecutionContext? parentContext,
        IExecutionContext? executionContext);

    TemplateExecutionFunction? FindTemplateExecutionFunction(
        string templateName);
}
```

### Simple Execution

The most common usage is the `Execute` overload that returns a string:

```csharp
using Hardened.Templates.Abstract;
using Hardened.Shared.Runtime.Attributes;

[Expose(typeof(IEmailRenderer))]
public class EmailRenderer : IEmailRenderer {
    private readonly ITemplateExecutionService _templates;
    private readonly IServiceProvider _serviceProvider;

    public EmailRenderer(
        ITemplateExecutionService templates,
        IServiceProvider serviceProvider) {
        _templates = templates;
        _serviceProvider = serviceProvider;
    }

    public async Task<string> RenderWelcomeEmail(User user) {
        var data = new {
            userName = user.Name,
            activationLink = $"https://example.com/activate/{user.Token}"
        };

        return await _templates.Execute(
            "welcome-email",
            data,
            _serviceProvider);
    }
}
```

### Template Naming

Template names are derived from the file path relative to the project root, without the extension. For example:

| File Path | Template Name |
|---|---|
| `Templates/welcome-email.html` | `Templates/welcome-email` |
| `Views/orders/detail.html` | `Views/orders/detail` |
| `email.html` | `email` |

---

## Using Templates with Web Routes

Templates integrate with the web routing system through the `TemplateName` property on `IExecutionResponse`. When a handler sets `TemplateName`, the template is rendered with the handler's return value as the data:

```csharp
[Get("/orders/{id}")]
public async Task<OrderViewModel> GetOrderPage(
    string id,
    IOrderRepository repo,
    IExecutionContext context) {
    var order = await repo.GetById(id);

    context.Response.TemplateName = "Views/orders/detail";

    return new OrderViewModel {
        OrderId = order.Id,
        Items = order.Items,
        Total = order.Total
    };
}
```

The template `Views/orders/detail.html` receives the `OrderViewModel` as its data:

```html
<h1>Order {{orderId}}</h1>
<table>
    <tr><th>Item</th><th>Price</th></tr>
    {{#items}}
    <tr><td>{{name}}</td><td>${{price}}</td></tr>
    {{/items}}
</table>
<p>Total: ${{total}}</p>
```

---

## How Compilation Works

The template compilation process happens entirely at build time:

1. The source generator scans for files matching the `Extensions` specified in `[TemplatePackage]`
2. Each template is parsed into an AST (abstract syntax tree)
3. The AST is compiled into a C# method that writes directly to an `ITemplateOutputWriter`
4. Variable references become direct property access via reflection-free generated code
5. Helper calls become invocations of registered `ITemplateHelper` implementations

This produces:

- **Zero parsing at runtime** -- templates are compiled C# code
- **Fast rendering** -- direct property access, no dictionary lookups
- **Build-time error detection** -- malformed templates cause build errors

---

## Complete Example

=== "AssemblyInfo.cs"

    ```csharp
    using Hardened.Templates.Abstract;

    [assembly: TemplatePackage(Extensions = "html")]
    ```

=== "Templates/dashboard.html"

    ```html
    <!DOCTYPE html>
    <html>
    <head><title>{{title}}</title></head>
    <body>
        <h1>{{title}}</h1>

        {{#hasNotifications}}
        <div class="notifications">
            <h2>Notifications</h2>
            <ul>
            {{#notifications}}
                <li>{{message}} - {{$formatDate timestamp "MMM dd"}}</li>
            {{/notifications}}
            </ul>
        </div>
        {{/hasNotifications}}

        {{^hasNotifications}}
        <p>No new notifications.</p>
        {{/hasNotifications}}
    </body>
    </html>
    ```

=== "DashboardController.cs"

    ```csharp
    [Get("/dashboard")]
    public async Task<DashboardViewModel> Dashboard(
        INotificationService notifications,
        IExecutionContext context) {
        var items = await notifications.GetRecent();

        context.Response.TemplateName = "Templates/dashboard";

        return new DashboardViewModel {
            Title = "My Dashboard",
            HasNotifications = items.Any(),
            Notifications = items
        };
    }
    ```

---

## Related Pages

- [Helpers](helpers.md) -- custom template helpers with `[TemplateHelper]`
- [Routing](../web/routing.md) -- connecting templates to web routes
- [Execution Model](../requests/execution-model.md) -- `IExecutionResponse.TemplateName`
