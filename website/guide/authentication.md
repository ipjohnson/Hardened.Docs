# Authentication

Authentication establishes who the caller is. [Authorization](/guide/authorization) decides what
that caller may do, and everything on that page assumes the caller is already established. This
page is the half in front of it: how a request comes to have a principal at all.

The framework's part is deliberately small. It ships the seam, the middleware that runs it, and a
testing source. It does not ship token validation: which issuer, token format or key set a
credential proves itself against is application configuration, and stays yours.

## The principal

Every request carries `IExecutionContext.CallerPrincipal`. It is never null: a request starts
anonymous and stays anonymous unless something establishes a caller. The principal is immutable;
the slot holding it is settable, and replacing it is what authentication does.

`ICallerPrincipal` carries the scheme that authenticated the caller, the subject and issuer the
credential named, the grants the caller holds, and the credential's claims. `IsAuthenticated` is
derived: a principal with no scheme is anonymous, and no principal can claim to be authenticated
without one.

## Declaring a scheme

A scheme is a type. Declaring one is declaring the type; using it anywhere puts it in the
published document:

```csharp
[HttpAuthenticationScheme("bearer", BearerFormat = "JWT")]
public sealed class ApiBearer : IAuthenticationScheme;

[Authorize<ApiBearer>]
[Post("/pets")]
public Task<Pet> CreatePet(CreatePetRequest body) => ...;
```

`[Authorize<ApiBearer>]` requires an authenticated caller and names the scheme the document
declares for the operation. The generator collects every scheme the handlers name into
`components.securitySchemes`, keyed by the type's name and shaped by the attribute on it:
`[HttpAuthenticationScheme]`, `[ApiKeyAuthenticationScheme]` or `[OAuth2AuthenticationScheme]`.

::: warning The scheme attribute describes the scheme type
`[HttpAuthenticationScheme]` and its siblings are read from the type `[Authorize<TScheme>]`
names, and from nowhere else. On a handler, a controller or the application module they publish
nothing and enforce nothing, and the build says so with `HRDSC001`.
:::

## Establishing the caller

`IPrincipalSource` is the seam. Register one or more and the framework's authentication
middleware runs them ahead of the whole handler chain, in registration order, until one answers.
The principal it returns is what every filter behind it judges, both authorization positions
included. An application that registers no source gets no middleware and pays nothing.

A bearer source is the common shape, and it is application code because the validation is:

```csharp
public sealed class BearerSource : IPrincipalSource<ApiBearer> {
    private readonly ITokenValidator _validator;

    public BearerSource(ITokenValidator validator) {
        _validator = validator;
    }

    public async ValueTask<ICallerPrincipal?> Authenticate(IExecutionContext context) {
        if (!context.Request.Headers.TryGetValue("Authorization", out var header)) {
            return null;
        }

        var value = header.ToString();

        if (!value.StartsWith("Bearer ", StringComparison.Ordinal)) {
            return null;
        }

        var token = await _validator.Validate(value.Substring("Bearer ".Length));

        if (token == null) {
            return null;
        }

        return new CallerPrincipal(
            "bearer",
            grants: token.Scopes,
            subject: token.Subject,
            issuer: token.Issuer);
    }
}
```

```csharp
services.AddSingleton<IPrincipalSource, BearerSource>();
```

Null means "this request carries nothing of mine", and the next source is asked; a request no
source answers for continues anonymously and is refused by authorization with its usual
challenge. A source that wants to refuse a present-but-invalid credential immediately throws
`AuthorizationException` with the challenge it means.

`IPrincipalSource<TScheme>` states the tie to the scheme type the operations name, so "find
references" walks from a requirement to the code that terminates its credential. The runtime
does not dispatch on the type parameter; implementing the plain `IPrincipalSource` behaves
identically.

## Testing

`TestGrantsPrincipalSource`, in `Hardened.Requests.Testing`, authenticates a request from an
`X-Test-Grants` header. Register it in the test application and a test states its caller inline:

```csharp
services.AddSingleton<IPrincipalSource, TestGrantsPrincipalSource>();
```

```csharp
var response = await testWebApp.Get(
    "/pets",
    request => request.Headers[TestGrantsPrincipalSource.GrantsHeader] = "pets:read");
```

A request without the header is anonymous. `TestGrantsPrincipalSource.AnonymousGrantsValue`
authenticates a caller who holds no grants at all, which is the third state an authorization
test wants beside "anonymous" and "holds exactly these".

## What the document says

Using a scheme declares it: `components.securitySchemes` carries every scheme the handlers name,
each secured operation carries its `security` requirement, and the operation publishes the 401
the refusal answers, with its `WWW-Authenticate` header. Grants become the requirement's scopes
only where the scheme kind can carry them, which is OAuth2. See
[The OpenAPI document](/guide/openapi-document).

## Next

- [Authorization](/guide/authorization) — what the established caller may do
- [Described authorization](/guide/described-authorization) — what a contract's `security`
  declares, contract-first
