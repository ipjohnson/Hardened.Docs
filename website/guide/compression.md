# Compression

Responses are compressed for clients that accept it, and compressed request bodies are decoded
before anything reads them. Response compression is an opt-in. Request decompression is always on.

## Turn it on

```csharp
[HardenedModule]
[Enable<HardenedCompression>]
[KestrelRuntime]
public partial class Application { }
```

Every response whose media type is on the list below is then compressed for a client whose
`Accept-Encoding` names a coding the server offers. gzip is offered first and Brotli behind it,
both at the fastest level.

```
GET /pets HTTP/1.1
Accept: application/json
Accept-Encoding: gzip, deflate, br, zstd

HTTP/1.1 200 OK
Content-Type: application/json
Content-Encoding: gzip
Vary: Accept-Encoding
Transfer-Encoding: chunked
```

The default rule compresses JSON, problem JSON and any `+json` type; XML and any `+xml` type;
JavaScript; NDJSON; SVG; and `text/*`, except `text/event-stream`. Images, archives, video and
`application/octet-stream` are left alone.

gzip is the default because every client accepts it, including a browser on plain HTTP, which
advertises Brotli only over TLS. Brotli's output is smaller at the same level. The CPU cost of
either at a typical body is tens of microseconds, so CPU does not decide the choice.

## One operation

`[Compress]` applies the same rule to one operation, or to every operation on a class, without
enabling the application-wide default. `Favor` picks the coding to try first when the client
accepts more than one.

```csharp
[Get("/pets")]
[Compress(Favor = CompressionType.Br)]
public Task<List<Pet>> List() => _pets.All();
```

An operation carrying `[Compress]` is left alone by the application-wide default, so the
declaration on the operation is the one that applies. `Favor` reorders the codings the
configuration offers. It cannot enable one the configuration turned off.

One declaration per operation. `[Compress]` on a class and on one of its methods is build error
`HRDW003`.

## A predicate

`[Compress<TPredicate>]` decides from the value the handler returned:

```csharp
[Get("/pets")]
[Compress<ListLargerThan>(50, Favor = CompressionType.Br)]
public Task<List<Pet>> List() => _pets.All();
```

```csharp
public sealed class ListLargerThan : ICompressionPredicate {
    private readonly int _count;

    private ListLargerThan(int count) => _count = count;

    public static ICompressionPredicate Create(object[] args) => args is [int count]
        ? new ListLargerThan(count)
        : throw new ArgumentException("ListLargerThan takes one integer, the count above which the body is compressed.");

    public bool ShouldCompress(object value, IExecutionContext context) =>
        value is System.Collections.ICollection { Count: var n } && n > _count;
}
```

The arguments reach the predicate through `Create`, the same way a cache key provider takes the
values from `[CacheResponse<VaryByQuery>("page")]`. Anything C# admits as an attribute argument
can go there. The predicate is built once per handler as its filter chain is assembled, so a
predicate handed arguments it cannot use fails there, naming the handler. A `params` parameter has
to be last, so `Favor` and anything else on the line is a property set with `=`.

A predicate replaces the media-type rule for its operation. It can opt in a type the default list
leaves out, and a predicate that always answers false is how an operation opts out of the
application-wide default.

A response replayed from the [response cache](#with-the-response-cache) carries no handler value,
so the predicate is not consulted on a hit. The default rule decides there.

## What is left alone

The decision is made on the first write of the body, when the status and the content type are
known. A response is passed through unchanged when:

- it already carries `Content-Encoding`. This is how the OpenAPI document, which is held gzipped,
  and a precompressed static file are left alone without knowing about the filter
- the status is 204, 206 or 304. There is no body, or there is a byte range, and an offset into a
  compressed stream is not an offset into the resource
- the media type is outside the list and no predicate says otherwise
- the client sent no `Accept-Encoding`, or one naming nothing the server offers

When a response is compressed, `Content-Encoding` is written, `Accept-Encoding` is merged into
`Vary`, any `Content-Length` is dropped so the host frames the body, and a strong `ETag` becomes a
weak one, because the bytes it validated are not the bytes being sent.

## Configuration

```csharp
public void ConfigureServices(IServiceCollection services) {
    services.ConfigureCompression(compression => {
        compression.Encodings = ["br", "gzip"];
        compression.Level = CompressionLevel.Optimal;
        compression.MediaTypes.Add("application/wasm");
        compression.MaxDecompressedRequestBytes = 8_000_000;
    });
}
```

| Setting | Default | Meaning |
|---|---|---|
| `Encodings` | `gzip`, `br` | The server's preference order. The first coding the client accepts is used. An operation's `Favor` moves one to the front. |
| `Level` | `Fastest` | The `CompressionLevel` handed to both encoders. |
| `MediaTypes` | the list above | The default rule, as patterns: an exact type, `text/*`, or `application/*+json`. Parameters on the content type are ignored. |
| `ExcludedMediaTypes` | `text/event-stream` | Types the rule leaves alone even where a pattern admits them. |
| `MaxDecompressedRequestBytes` | 30 000 000 | The most a compressed request body may decode to. Past it the request is a 413. |

The configuration is registered by the request module whether or not response compression is
enabled, because the request-side cap applies to every application.

## Where it sits

The filter runs at `FilterOrder.Before + FilterOrder.ResponseCache`, outside the response cache
and inside everything that can refuse a request without reading a body.

### With the response cache

The cache buffers the body inside the compression filter, so a stored entry holds identity bytes
and both a miss and a hit are compressed on the way out. A hit to a client that accepts nothing is
served plain from the same entry. `Content-Encoding` is never stored. `Vary` is.

### With streams

A streamed response is one compressed member. The streaming filter flushes the body after every
item, and a flush through the compression filter is a sync flush on the encoder, so the caller
still reads the first item while the handler is producing the second. Event streams are excluded
by default and go out uncompressed.

### On each host

| Host | What to know |
|---|---|
| Kestrel | Nothing. |
| ASP.NET Core | Do not add ASP.NET's own response compression middleware as well. If it is present it sees the `Content-Encoding` and stands down. |
| Lambda, API Gateway | The filter marks the response binary, so the event processor base64-encodes the body. |
| Lambda, streaming | Nothing. |
| Test client | `Deserialize<T>`, `ReadTextAsync` and `DeserializeAsyncEnumerable<T>` decode. `Body` is the bytes as sent. |

A `HEAD` runs the `GET` handler inside a counting stream, so its `Content-Length` is the compressed
length the `GET` would send.

## Compressed requests

A request body carrying `Content-Encoding: gzip` or `Content-Encoding: br` is decoded before
binding, for every reader: the JSON deserializers, forms, Newtonsoft and raw bodies. The header is
removed on the way, so nothing downstream decodes twice, and `Content-Length` with it, because it
measured the bytes on the wire.

| The request says | The answer |
|---|---|
| nothing, or `identity` | the body is read as it is |
| `gzip` or `br` | the body is decoded, and read up to `MaxDecompressedRequestBytes` |
| past the cap | 413 |
| any other coding | 415, with `Accept-Encoding: gzip, br` naming what the server decodes |

The cap is on the decoded size. A gzip member of a few hundred bytes can decode to gigabytes, and
the host's own request limit measures the bytes on the wire.

The response cache's `ByPayload` strategy hashes the decoded body, so a compressed request and its
plain twin share an entry.

## Testing

The test client sends `Accept-Encoding: gzip` unless a test sets the header, and its decoded
accessors undo whatever coding came back. A test asserting on the value never has to know the
response was compressed. A test asserting on the coding reads `Headers` and `Body`:

```csharp
[HardenedTest]
public async Task AListIsGzipped(ITestWebApp testWebApp) {
    var response = await testWebApp.Get("/pets");

    Assert.Equal("gzip", response.Headers["Content-Encoding"].ToString());
    Assert.Equal(50, response.Deserialize<List<Pet>>().Count);
}

[HardenedTest]
public async Task AClientAcceptingNothingIsServedPlain(ITestWebApp testWebApp) {
    var response = await testWebApp.Get("/pets",
        request => request.Headers["Accept-Encoding"] = "identity");

    Assert.False(response.Headers.ContainsKey("Content-Encoding"));
}
```

## Not in this version

- A size threshold. The length is unknown at the first write, and buffering to learn it delays
  every response. A small body grows by a few bytes.
- A typed predicate, checked against the handler's return type at build time.
- Remembering the predicate's decision on a cache entry, so a hit follows the same rule as the
  miss.
- Quality values in `Accept-Encoding`. A client sending `gzip;q=0` is served gzip.
- `deflate` and `zstd`.
