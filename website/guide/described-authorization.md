# Described authorization

What a contract says an operation requires of its caller, and what Hardened does with it. This is
the contract-first half of [Authorization](/guide/authorization); everything about attributes,
policies and conventions there applies to generated routes unchanged.

The short version: a description carries **authorization**, not **authentication**. Hardened
reads scopes and ignores schemes. The scheme decides one thing, whether the entry carries scopes
at all; which issuer or token format a caller proves themselves against stays
[application configuration](/guide/authentication).

## What is read

| declaration | becomes |
|---|---|
| `oauth2: ["pets:read"]` | `Requirement.Grant("pets:read")` |
| `oauth2: ["pets:read", "pets:write"]` | `Grant("pets:read") & Grant("pets:write")` |
| `apiKey: []` — any scheme with no scopes | `Requirement.Authenticated()` |
| `[{ oauth2: [...] }, { apiKey: [] }]` | `AnyOf(...)` — the array is an OR |
| `{ oauth2: [...], apiKey: [] }` | `AllOf(...)` — keys in one entry are an AND |
| `security: []` on an operation | **nothing** |
| no `security` at all | the document-level default, or nothing |

Only `oauth2` and `openIdConnect` may carry scopes. The specification requires every other type
to declare an empty array, so those say "be someone" and nothing more.

A scope name is read as a grant name, unchanged. No prefixing and no namespacing.

### An empty array means two different things

- **`{ oauth2: [] }`** — an empty *scope* array. The caller must be authenticated and needs no
  particular permission.
- **`security: []`** — an empty *security* array. The specification's way of opting one
  operation out of a document-level default. It derives **nothing**.

### An unscoped entry is a requirement, not the absence of one

```yaml
security:
  - oauth2: ["pets:write"]
  - apiKey: []
```

The second entry becomes `Authenticated()`, so the alternative stays an alternative. Reading it
as "requires nothing" would satisfy the OR for everybody, making a document that reads as
protective weaker than declaring none at all.

## It narrows, and can never open

A described requirement arrives as one more entry in the handler's metadata, alongside anything
the implementation declared, and every entry conjoins:

- A contract can **narrow** a route.
- A contract can **never widen** one. `security: []` does not strip an `[AuthorizeGrants]`
  somebody wrote on the implementation.
- `[AllowAnonymous]` remains the single thing that cancels a requirement, the same rule an
  attribute or a convention is held to.

`security: []` therefore derives nothing rather than `[AllowAnonymous]`. An author who wants a
route anonymous says so in code, where whoever reads the handler can see it.

## Enforcement needs no opt-in

A handler that declares something is guarded on the next build. `[RequireAuthorization]` decides
only what happens to a handler that declares nothing.

::: warning Adding `security` to a published contract is a breaking change for its callers
An operation that answered 200 to an anonymous request answers 401 once the description says it
needs a scope. That is the correct behaviour and it is not a quiet one. It is worth a release
note when it happens to a published contract.
:::

## Smithy carries less, and that is the language

Smithy has no equivalent of an OAuth scope. A model can say a caller must be authenticated; it
cannot say what they must hold.

| declaration | becomes |
|---|---|
| service declares `@httpBearerAuth` (or any scheme) | `Requirement.Authenticated()` |
| operation carries `@optionalAuth` | nothing |
| `@auth([])` on the operation or the service | nothing |
| service declares no scheme | nothing |

To require particular grants on a Smithy-generated route, put `[AuthorizeGrants]` on the
implementation. A contract can narrow a route and never widen one, so that composes.

## Diagnostics

A `security` entry naming a scheme that `components.securitySchemes` does not declare is
reported at build time. The operation falls back to requiring an authenticated caller and none
of the permissions it names.

## Next

- [Authentication](/guide/authentication) — establishing the caller these requirements judge
- [Authorization](/guide/authorization) — the requirement algebra the derived entries join
