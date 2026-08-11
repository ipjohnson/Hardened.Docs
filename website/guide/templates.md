# Templates

The template engine is Mustache-shaped, and — like everything else here — compiled. A template file
becomes a C# class during the build, so a property that does not exist on the model is a build
failure rather than a blank in the rendered page.

## Adding templates

Templates are `AdditionalFiles`. Four extensions are recognised: `html`, `js`, `css` and `md`.

```xml
<ItemGroup>
    <None Remove="Templates\main.html" />
    <AdditionalFiles Include="Templates\main.html" />
</ItemGroup>
```

The `None Remove` matters: without it the file is also part of the default `None` glob, and the
build carries it twice.

## Writing one

A template declares the model it renders, and the namespaces it needs to name types:

```html
{{model OrderListModel}}
{{using Contoso.Orders.Models}}

<table>
    <thead>
    <tr><td>Reference</td><td>Total</td><td>Placed</td></tr>
    </thead>
    {{#each Orders}}
    <tr>
        <td>{{Reference}}</td>
        <td>{{ Total : "###.00" }}</td>
        <td>{{PlacedAt}}</td>
    </tr>
    {{/each}}
</table>
```

::: v-pre

| Syntax | Meaning |
|---|---|
| `{{model T}}` | The type this template renders. Property tokens resolve against it |
| `{{using N}}` | A namespace import for the generated class |
| `{{Property}}` | A property of the current model |
| `{{ Property : "###.00" }}` | A property with a .NET format string |
| `{{#each Collection}} … {{/each}}` | Iterate; inside the block the model is the element |
| `{{ $Helper arg }}` | Invoke a helper. `.` passes the current model |

:::

## Rendering from a handler

`[Template]` names the template that renders the handler's return value:

```csharp
using Hardened.Requests.Abstract.Attributes;

public class OrderController {
    [Get("/orders")]
    [Template("main")]
    public OrderListModel List() => _orders.Recent();
}
```

The handler returns a model, not markup. Rendering is a step in the
[execution pipeline](/guide/execution-pipeline), which means the same handler can serialise as JSON
for one caller and render as HTML for another without the handler knowing.

## Helpers

A helper is a class marked `[TemplateHelper]` implementing `ITemplateHelper`. It is resolved from the
container, so it can take dependencies:

```csharp
using Hardened.Templates.Abstract;

[TemplateHelper("InvoiceLink")]
public class InvoiceLinkHelper : ITemplateHelper {
    private readonly IBillingConfiguration _configuration;

    public InvoiceLinkHelper(IOptions<IBillingConfiguration> configuration) {
        _configuration = configuration.Value;
    }

    public ValueTask<object> Execute(
        ITemplateExecutionContext context, params object[] arguments) {

        if (arguments.Length != 1 || arguments[0] is not OrderSummary order) {
            throw new Exception("One argument is required: an OrderSummary");
        }

        var link = $"{_configuration.PortalUrl}/invoices/{order.Reference}";

        return new ValueTask<object>(
            new SafeString($"<a href=\"{link}\">invoice</a>"));
    }
}
```

Used as <span v-pre>`{{ $InvoiceLink . }}`</span> — the `$` marks a helper token, and `.` passes the
current model as the single argument.

### Escaping

Returning a `string` escapes it. Returning a `SafeString` does not, which is how a helper emits
markup. That is the right way round: a helper that forgets to think about escaping produces escaped
output, and a helper that wants to inject HTML has to say so.

### Lifecycle

`[TemplateHelper]` takes a lifecycle as its second argument:

```csharp
[TemplateHelper("Now", TemplateHelperLifecycle.Transient)]
public class NowHelper : ITemplateHelper { }
```

| Lifecycle | Instances |
|---|---|
| `Singleton` (default) | One per template environment |
| `Scoped` | One per scope |
| `Transient` | One per template render |

## Package options

`[TemplatePackage]` on the assembly adjusts how templates in it are read:

```csharp
[assembly: TemplatePackage(Extensions = "html", Token = "{{TOKEN}}")]
```

`Extensions` narrows which files are treated as templates, and `Token` changes the delimiter for a
codebase where <span v-pre>`{{ }}`</span> collides with something else it emits.
