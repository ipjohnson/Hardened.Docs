# JSON serialization

Hardened serializes JSON with `System.Text.Json`, and configures it by **registering an
`IJsonTypeInfoResolver`** — a source-generated `JsonSerializerContext`. That one registration is
read by every JSON serializer in the pipeline: the request body, the response body, and each item
of a streamed response.

```csharp
[JsonSourceGenerationOptions(JsonSerializerDefaults.Web, UseStringEnumConverter = true)]
[JsonSerializable(typeof(Todo))]
[JsonSerializable(typeof(NewTodo))]
public partial class AppJsonContext : JsonSerializerContext;
```

```csharp
[HardenedModule]
[HardenedWebModule]
public partial class AppLibrary : IServiceCollectionConfiguration {

    public void ConfigureServices(IServiceCollection services) {
        services.AddSingleton<IJsonTypeInfoResolver>(AppJsonContext.Default);
    }
}
```

Registered resolvers answer first, in registration order. Reflection covers whatever they do not
declare, on a host where reflection is available at all — so a context does not have to be complete
to be worth registering. The project templates ship this wiring already.

## Enums are numbers unless you say otherwise

This is the default that surprises people, and it is `System.Text.Json`'s rather than Hardened's.
Given a plain C# enum and no configuration:

```csharp
public enum Priority { Low, High }

public record Todo(int Id, string Title, Priority Priority);
```

the response is `{"id":1,"title":"...","priority":0}`, and a client that sends `"priority":"high"`
is answered **400**. Nothing fails at build time; integers are perfectly good JSON.

`UseStringEnumConverter = true` on the context is the fix, and it is set in the templates. With it,
the value written is the **C# member name**:

```json
{ "id": 1, "title": "...", "priority": "High" }
```

PascalCase, inside an otherwise camelCase document. `PropertyNamingPolicy` governs property names
and does not reach enum members — there is no setting that camel-cases them.

### Choosing the wire vocabulary

If the API should say `"high"`, or `"in-progress"`, name a policy through a converter. The
attribute form takes a type with no constructor arguments, so the policy goes in a subclass:

```csharp
public sealed class KebabEnum<T> : JsonStringEnumConverter<T> where T : struct, Enum {
    public KebabEnum() : base(JsonNamingPolicy.KebabCaseLower) { }
}

[JsonConverter(typeof(KebabEnum<Priority>))]
public enum Priority { Low, InProgress }   // "low", "in-progress"
```

Reads stay case-insensitive whichever policy you choose; only the written form changes. **Decide
this before the first client.** Changing an enum's wire vocabulary later breaks every consumer, and
no compiler will tell you it happened.

`[JsonStringEnumMemberName]` expresses the same thing per member and arrives in .NET 9. Hardened
targets `net8.0`, so it is not available here yet.

## Adding your own converter

Four routes, and they do not all reach the same code.

| Route | Reaches | Use when |
|---|---|---|
| `[JsonConverter]` on the type or property | Everything | One type has one wire form, always |
| A `JsonSerializerContext` registered as `IJsonTypeInfoResolver` | Everything | The general answer — AOT-safe, and how the templates do it |
| `Converters` on `[JsonSourceGenerationOptions]` | Everything, via that context | A converter that should apply across the context's types |
| `Hardened.Requests.Runtime.Configuration.JsonSerializerConfiguration` | The wire only | You need to replace `JsonSerializerOptions` wholesale |

The last one takes the whole options object rather than a delta, so setting it drops
`JsonSerializerDefaults.Web` — camelCase, case-insensitive matching, `AllowReadingFromString` —
unless you rebuild all of it. Set both `SerializeOptions` and `DeSerializerOptions` or the
application writes what it will not read.

::: warning A converter in `Options.Converters` outranks `[JsonConverter]` on the type
`System.Text.Json` ranks options-level converters **above** the attribute. Adding a bare
`JsonStringEnumConverter` to the options therefore overrides converters generated from an OpenAPI or
Smithy contract, and writes the C# member name where the document declares something else —
`"ScienceFiction"` for a schema whose value is `science-fiction`, and a 400 reading it back. Attach
converters to types, or scope them to a context.
:::

## Contract-first applications

If your models come from an OpenAPI description or a Smithy model, the enums are already handled.
The build generates a converter per enum carrying the document's own values, attaches it with
`[JsonConverter]`, and registers the generated `IJsonTypeInfoResolver` for you. There is nothing to
configure and nothing to add a second context for.

This is also why the enum default bites code-first applications only.

## Native AOT

A published AOT application has no reflection fallback: every type on the wire must be declared in a
registered context, and one that is missing throws a `NotSupportedException` naming it. That is the
truthful failure — the same call succeeds on a JIT host by reflecting, which is what makes a missing
`[JsonSerializable]` line easy to ship.

**Do not use the non-generic `JsonStringEnumConverter` in an AOT application.** It builds a
converter per enum at run time, which is the one thing AOT cannot do:

```csharp
[JsonConverter(typeof(JsonStringEnumConverter))]        // works on a JIT host, fails when published
[JsonConverter(typeof(JsonStringEnumConverter<Priority>))]  // correct
```

The compiler reports this as `SYSLIB1034` wherever the enum is reachable from a
`JsonSerializerContext` — including when the non-generic form appears in `Converters` on
`[JsonSourceGenerationOptions]`. It is a warning, and a build with
`TreatWarningsAsErrors` turns it into the error it deserves to be. An enum reachable from **no**
context gets no diagnostic at all, which is the case worth being careful about: it works locally
and fails after publishing.

`UseStringEnumConverter = true` has no such problem. The generator has the enum at compile time, so
it emits metadata rather than building a converter.
