# Template Helpers

Template helpers extend Hardened's Mustache-style templates with custom logic. Helpers are invoked from templates using the `{{$token args}}` syntax and can perform formatting, localization, conditional rendering, and any other operation.

**Package:** `Hardened.Templates.Abstract` (namespace `Hardened.Templates.Abstract`)

---

## [TemplateHelper]

The `[TemplateHelper]` attribute registers a class as a template helper. It specifies the Mustache token used to invoke the helper and its lifecycle.

### Definition

```csharp
namespace Hardened.Templates.Abstract;

public class TemplateHelperAttribute : Attribute {
    public TemplateHelperAttribute(
        string token,
        TemplateHelperLifecycle lifecycle = TemplateHelperLifecycle.Singleton) {
        MustacheToken = token;
        Lifecycle = lifecycle;
    }

    public string MustacheToken { get; }
    public TemplateHelperLifecycle Lifecycle { get; }
}
```

### Properties

| Property | Description |
|---|---|
| `MustacheToken` | The token used in templates as `{{$token args}}` |
| `Lifecycle` | The helper's lifecycle (Singleton, Scoped, or Transient) |

---

## TemplateHelperLifecycle

Controls how instances of the helper are created and reused.

### Definition

```csharp
namespace Hardened.Templates.Abstract;

public enum TemplateHelperLifecycle {
    Singleton,
    Scoped,
    Transient,
}
```

| Value | Description |
|---|---|
| `Singleton` | One instance shared across all template executions (default) |
| `Scoped` | One instance per scope (typically per request) |
| `Transient` | One instance per template generation |

---

## ITemplateHelper

The interface that all template helpers must implement.

### Definition

```csharp
namespace Hardened.Templates.Abstract;

public interface ITemplateHelper {
    ValueTask<object> Execute(
        ITemplateExecutionContext handlerDataContext,
        params object[] arguments);
}
```

### Parameters

| Parameter | Description |
|---|---|
| `handlerDataContext` | The current template execution context, providing access to the template data and services |
| `arguments` | The arguments passed from the template expression |

---

## Creating a Helper

### Step 1: Implement ITemplateHelper

```csharp
using Hardened.Templates.Abstract;

[TemplateHelper("formatDate")]
public class FormatDateHelper : ITemplateHelper {
    public ValueTask<object> Execute(
        ITemplateExecutionContext handlerDataContext,
        params object[] arguments) {
        if (arguments.Length < 1) {
            return new ValueTask<object>("");
        }

        var date = arguments[0] switch {
            DateTime dt => dt,
            DateTimeOffset dto => dto.DateTime,
            string s => DateTime.Parse(s),
            _ => DateTime.MinValue
        };

        var format = arguments.Length > 1
            ? arguments[1]?.ToString() ?? "yyyy-MM-dd"
            : "yyyy-MM-dd";

        return new ValueTask<object>(date.ToString(format));
    }
}
```

### Step 2: Use in Templates

```html
<p>Created: {{$formatDate createdAt "MMM dd, yyyy"}}</p>
<p>Updated: {{$formatDate updatedAt}}</p>
```

The `$formatDate` token maps to the `FormatDateHelper` class. The arguments after the token (`createdAt`, `"MMM dd, yyyy"`) are passed to the `Execute` method.

---

## Examples

### String Formatting

```csharp
[TemplateHelper("uppercase")]
public class UppercaseHelper : ITemplateHelper {
    public ValueTask<object> Execute(
        ITemplateExecutionContext context,
        params object[] arguments) {
        var value = arguments.FirstOrDefault()?.ToString() ?? "";
        return new ValueTask<object>(value.ToUpperInvariant());
    }
}
```

```html
<h1>{{$uppercase title}}</h1>
```

### Currency Formatting

```csharp
[TemplateHelper("currency")]
public class CurrencyHelper : ITemplateHelper {
    public ValueTask<object> Execute(
        ITemplateExecutionContext context,
        params object[] arguments) {
        if (arguments.Length < 1) {
            return new ValueTask<object>("$0.00");
        }

        var amount = Convert.ToDecimal(arguments[0]);
        var culture = arguments.Length > 1
            ? new CultureInfo(arguments[1].ToString()!)
            : CultureInfo.CurrentCulture;

        return new ValueTask<object>(amount.ToString("C", culture));
    }
}
```

```html
<span>{{$currency price "en-US"}}</span>
```

### Conditional CSS Classes

```csharp
[TemplateHelper("cssIf")]
public class CssIfHelper : ITemplateHelper {
    public ValueTask<object> Execute(
        ITemplateExecutionContext context,
        params object[] arguments) {
        if (arguments.Length < 2) {
            return new ValueTask<object>("");
        }

        var condition = arguments[0];
        var className = arguments[1]?.ToString() ?? "";

        var isTruthy = condition switch {
            bool b => b,
            int i => i != 0,
            string s => !string.IsNullOrEmpty(s),
            null => false,
            _ => true
        };

        return new ValueTask<object>(isTruthy ? className : "");
    }
}
```

```html
<div class="alert {{$cssIf isError "alert-danger"}} {{$cssIf isSuccess "alert-success"}}">
    {{message}}
</div>
```

### Scoped Helper with DI

Helpers with `Scoped` or `Transient` lifecycle can access per-request services:

```csharp
[TemplateHelper("userGreeting", TemplateHelperLifecycle.Scoped)]
public class UserGreetingHelper : ITemplateHelper {
    private readonly IUserContext _userContext;

    public UserGreetingHelper(IUserContext userContext) {
        _userContext = userContext;
    }

    public ValueTask<object> Execute(
        ITemplateExecutionContext context,
        params object[] arguments) {
        var greeting = _userContext.IsAuthenticated
            ? $"Hello, {_userContext.DisplayName}!"
            : "Hello, Guest!";

        return new ValueTask<object>(greeting);
    }
}
```

```html
<header>
    <p>{{$userGreeting}}</p>
</header>
```

!!! note
    Scoped helpers receive a new instance per request scope, so they can safely depend on scoped services like `IUserContext`.

---

## Async Helpers

Since `Execute` returns `ValueTask<object>`, helpers can perform async operations:

```csharp
[TemplateHelper("featureFlag", TemplateHelperLifecycle.Scoped)]
public class FeatureFlagHelper : ITemplateHelper {
    private readonly IFeatureFlagService _flags;

    public FeatureFlagHelper(IFeatureFlagService flags) {
        _flags = flags;
    }

    public async ValueTask<object> Execute(
        ITemplateExecutionContext context,
        params object[] arguments) {
        var flagName = arguments.FirstOrDefault()?.ToString() ?? "";
        var isEnabled = await _flags.IsEnabledAsync(flagName);
        return isEnabled;
    }
}
```

```html
{{#$featureFlag "new-dashboard"}}
    <div class="new-dashboard">...</div>
{{/$featureFlag "new-dashboard"}}
```

---

## SafeString -- Unescaped Output

By default, helper output is HTML-escaped. To return raw HTML, wrap the return value in `SafeString`:

```csharp
using Hardened.Templates.Abstract;

[TemplateHelper("rawHtml")]
public class RawHtmlHelper : ITemplateHelper {
    public ValueTask<object> Execute(
        ITemplateExecutionContext context,
        params object[] arguments) {
        var html = arguments.FirstOrDefault()?.ToString() ?? "";
        return new ValueTask<object>(new SafeString(html));
    }
}
```

!!! warning
    Use `SafeString` with caution. Returning unescaped HTML can expose your application to XSS attacks if the input is not sanitized.

---

## Helper Discovery

Helpers are discovered automatically by the source generator:

1. It scans for classes with the `[TemplateHelper]` attribute
2. It registers them in the template engine with their specified token and lifecycle
3. At template compile time, `{{$token ...}}` expressions are resolved to the corresponding helper

No manual registration is needed -- decorate the class and it is available in all templates.

---

## Related Pages

- [Templates Overview](overview.md) -- Mustache syntax, `[TemplatePackage]`, and `ITemplateExecutionService`
- [Dependency Injection](../shared/dependency-injection.md) -- how helpers are registered in the DI container
