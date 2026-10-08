# Day 2: HTTP Contracts and Request Lifecycle

This lesson expands Day 2 of the
[backend learning roadmap](../BACKEND_LEARNING_ROADMAP.md#day-2-http-contracts-and-request-lifecycle).
It assumes the Day 1 in-memory product store exists. External references are
optional; everything required for this lesson is explained here.

## Learning outcomes

By the end of this lesson, you should be able to:

- trace an HTTP request from a client to a Go handler and back;
- choose methods and status codes from their protocol meaning;
- distinguish safe, idempotent, and cacheable operations;
- configure a bounded `http.Server`;
- decode JSON with content-type, size, shape, and trailing-data checks;
- separate transport validation from business rules;
- order middleware deliberately;
- return stable problem details without exposing internals;
- test handlers with `httptest`;
- explain graceful shutdown and cancellation propagation.

Budget about three focused hours. Build only the endpoints listed here. PostgreSQL,
authentication, and telemetry arrive on later days.

## 1. Follow one request through the system

When a client requests `https://api.example.test/v1/products`, the main stages are:

1. DNS resolves the host name to an IP address.
2. The client opens a TCP connection to that address.
3. TLS authenticates the server and establishes encrypted transport.
4. A reverse proxy or load balancer may terminate TLS, enforce limits, and select
   an application instance.
5. Go's HTTP server parses the request and invokes the matching handler.
6. Middleware adds cross-cutting behavior around the handler.
7. The handler validates the HTTP representation and calls application behavior.
8. The response travels back over a reusable connection when both sides permit it.

Each stage has bounded resources: DNS time, connection slots, file descriptors,
memory for headers and bodies, handler goroutines, and downstream calls. Backend
reliability begins by placing limits on those resources.

HTTP keep-alive lets several requests reuse one connection. Reuse avoids repeated
TCP and TLS setup, but an application must close response bodies when acting as an
HTTP client. Failure to close them can prevent connection reuse and eventually
exhaust sockets or file descriptors.

```go
resp, err := client.Do(req)
if err != nil {
    return err
}
defer resp.Body.Close()
```

## 2. Methods express semantics

Choose a method because its semantics match the operation, not because routing is
convenient.

| Method | Intended meaning | Safe | Idempotent |
| --- | --- | --- | --- |
| `GET` | Retrieve a representation | Yes | Yes |
| `HEAD` | Retrieve headers without a response body | Yes | Yes |
| `POST` | Submit data or create under server control | No | Not inherently |
| `PUT` | Replace the state at a known URI | No | Yes |
| `PATCH` | Apply a partial modification | No | Not inherently |
| `DELETE` | Remove the target state | No | Yes |

**Safe** means the client is not asking to change server state. Logging and metrics
may still change internally. **Idempotent** means repeating the same request has the
same intended effect as sending it once. It does not require identical response
codes or timestamps.

`POST /v1/orders` is not naturally idempotent: repeating it could create two
orders. Day 6 of the implementation roadmap later adds an idempotency-key contract.

Caching is separate from safety and idempotency. A response is cacheable only when
the method, status, headers, and authorization context allow it. Do not add caching
to today's endpoints; first learn to make their semantics explicit.

## 3. Status codes, headers, and representations

Use a small, consistent status vocabulary:

| Situation | Status |
| --- | --- |
| Successful `GET` | `200 OK` |
| Resource created | `201 Created` plus `Location` |
| Successful request with no body | `204 No Content` |
| Malformed JSON or invalid request fields | `400 Bad Request` |
| Missing or invalid authentication | `401 Unauthorized` |
| Authenticated but not allowed | `403 Forbidden` |
| Resource absent or deliberately concealed | `404 Not Found` |
| State or uniqueness conflict | `409 Conflict` |
| Unsupported request media type | `415 Unsupported Media Type` |
| Body exceeds the configured limit | `413 Content Too Large` |
| Unexpected server failure | `500 Internal Server Error` |
| Dependency temporarily unavailable | `503 Service Unavailable` |

For JSON responses, set `Content-Type: application/json` before calling
`WriteHeader`. Headers written afterward may be ignored because the status and
headers have already been sent.

For a created product, return a representation and its URI:

```text
HTTP/1.1 201 Created
Content-Type: application/json
Location: /v1/products/product-123
```

Never return Go error text, stack traces, SQL, filesystem paths, or secrets in a
public response.

## 4. Start a bounded Go HTTP server

`http.ListenAndServe` with a zero-value server omits important production limits.
Construct an `http.Server` explicitly:

```go
server := &http.Server{
    Addr:              ":8080",
    Handler:           handler,
    ReadHeaderTimeout: 5 * time.Second,
    ReadTimeout:       10 * time.Second,
    WriteTimeout:      15 * time.Second,
    IdleTimeout:       60 * time.Second,
    MaxHeaderBytes:    1 << 20,
}
```

These are starting values, not universal truths. Measure real payload and latency
needs later. The important property is that header parsing, request reads, response
writes, idle connections, and header memory are not silently unbounded.

Go 1.22 and later supports method-aware patterns on `http.ServeMux`:

```go
mux := http.NewServeMux()
mux.HandleFunc("GET /health/live", healthHandler)
mux.HandleFunc("GET /v1/products", productHandler.List)
mux.HandleFunc("POST /v1/products", productHandler.Create)
```

Keep `main` as the composition root: create the store, handler, middleware chain,
and server there. Handlers should not read global variables to discover them.

## 5. Decode JSON strictly and within a limit

Untrusted bodies need several independent checks:

1. Require a supported content type.
2. Limit bytes before decoding.
3. Reject unknown fields.
4. Decode exactly one JSON value.
5. Validate field-level transport rules.

Define a transport-only request type:

```go
type createProductRequest struct {
    SKU  string `json:"sku"`
    Name string `json:"name"`
}
```

Check the media type with `mime.ParseMediaType`; a valid request may include a
parameter such as `application/json; charset=utf-8`.

```go
func requireJSON(r *http.Request) error {
    mediaType, _, err := mime.ParseMediaType(r.Header.Get("Content-Type"))
    if err != nil || mediaType != "application/json" {
        return errors.New("Content-Type must be application/json")
    }
    return nil
}
```

Bound and decode the body:

```go
func decodeJSON(w http.ResponseWriter, r *http.Request, dst any) error {
    const maxBodyBytes = 1 << 20

    r.Body = http.MaxBytesReader(w, r.Body, maxBodyBytes)
    decoder := json.NewDecoder(r.Body)
    decoder.DisallowUnknownFields()

    if err := decoder.Decode(dst); err != nil {
        return fmt.Errorf("decode JSON: %w", err)
    }

    var extra any
    if err := decoder.Decode(&extra); !errors.Is(err, io.EOF) {
        if err == nil {
            return errors.New("request body must contain one JSON value")
        }
        return fmt.Errorf("decode trailing JSON: %w", err)
    }

    return nil
}
```

The body limit protects memory and time. Unknown-field rejection catches misspelled
client fields instead of silently ignoring them. The second decode rejects
`{"sku":"A"}{"sku":"B"}` and other trailing JSON.

In a complete handler, classify `*http.MaxBytesError` as 413 and other malformed
input as 400. Return a stable public problem; log diagnostic detail privately.

## 6. Separate validation responsibilities

Transport validation asks whether the request is a valid member of the HTTP API:

- is the body JSON?
- are required fields present and correctly shaped?
- is a string within the public length limit?
- is an identifier syntactically valid?

Business validation asks whether an action is allowed in the current domain state:

- is the SKU already assigned?
- may this actor create products?
- is there enough inventory to place the order?
- is the order in a state that can be canceled?

The handler owns transport concerns and maps outcomes to HTTP. The product use case
or store owns today's duplicate-SKU invariant. Do not put HTTP status codes in the
product package.

## 7. Return consistent problem details

Use one public error shape rather than ad hoc messages:

```go
type problem struct {
    Type     string `json:"type"`
    Title    string `json:"title"`
    Status   int    `json:"status"`
    Detail   string `json:"detail,omitempty"`
    Instance string `json:"instance,omitempty"`
    Code     string `json:"code,omitempty"`
}
```

Example response:

```json
{
  "type": "https://example.test/problems/invalid-request",
  "title": "Invalid request",
  "status": 400,
  "detail": "name is required",
  "instance": "/v1/products",
  "code": "invalid_request"
}
```

Set `Content-Type: application/problem+json`. The `code` is the stable value a
client can branch on. Do not make clients parse `detail`; wording may change.

Map known errors deliberately:

```go
switch {
case errors.Is(err, product.ErrDuplicateSKU):
    writeProblem(w, r, http.StatusConflict, "duplicate_sku", "SKU already exists")
default:
    writeProblem(w, r, http.StatusInternalServerError, "internal_error", "Internal server error")
}
```

Log the unexpected error with its request ID. The client receives no internal
diagnostic detail.

## 8. Middleware order is observable behavior

Middleware wraps a handler. If constructed as:

```go
handler := requestID(accessLog(recoverPanic(mux)))
```

Then request ID runs first and can be used by access logging and recovery. Access
logging wraps recovery, so it observes the final 500 status written after a panic.
Think in both directions: request execution travels from the outer wrapper inward;
response completion unwinds outward.

A useful initial order is:

1. Request ID creation or validation.
2. Access logging.
3. Panic recovery.
4. Future authentication.
5. Routing and handlers.

Request IDs must be bounded and validated before trusting a caller-supplied value.
Generate one when absent or invalid. Store it in request context with an unexported,
typed key, not a plain string key.

Recovery should catch a panic at the request boundary, log it with stack and request
ID, and return a generic 500 if the response has not already been committed. It is
not a substitute for fixing defects.

Access logs should record method, route template, status, response bytes, duration,
and request ID. Avoid raw query strings and bodies because they may contain secrets
or high-cardinality personal data.

## 9. Liveness, readiness, and shutdown

`GET /health/live` answers only whether the process can serve HTTP. It should not
query PostgreSQL or another dependency. If a database outage makes liveness fail,
an orchestrator may restart healthy application processes repeatedly without fixing
the database.

Readiness is added with the database on Day 3. It answers whether this instance
should receive traffic and may use a tightly bounded dependency check.

For graceful shutdown:

1. Receive `SIGINT` or `SIGTERM` in `main`.
2. Mark the instance unready before or while traffic is removed.
3. Call `server.Shutdown` with a deadline.
4. Stop accepting new connections and wait for active handlers.
5. Close owned dependencies after handlers finish.
6. Exit nonzero only for an actual startup or server failure.

```go
shutdownCtx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
defer cancel()

if err := server.Shutdown(shutdownCtx); err != nil {
    return fmt.Errorf("shutdown HTTP server: %w", err)
}
```

Request contexts are canceled when clients disconnect, requests are canceled by
HTTP/2, or the server stops the connection. Pass `r.Context()` into blocking use
cases so work can stop when its result is no longer useful.

## 10. Guided build

Add the smallest project-appropriate HTTP structure:

```text
cmd/api/main.go
internal/platform/httpserver/
    errors.go
    middleware.go
internal/product/
    handler.go
    handler_test.go
```

Complete the work in this order:

1. Add `GET /health/live` returning 200 and a small JSON body.
2. Add `GET /v1/products` using the Day 1 store.
3. Add `POST /v1/products` with strict JSON decoding.
4. Return 201 plus `Location` for successful creation.
5. Map duplicate SKU to a stable 409 problem.
6. Add request-ID, recovery, and access-log middleware.
7. Configure explicit server timeouts and graceful shutdown.

Do not add a framework. The purpose is to see `net/http` contracts directly.

## 11. Handler tests with `httptest`

Test a handler without binding a real port:

```go
func TestCreateProduct(t *testing.T) {
    store := product.NewStore()
    handler := NewHandler(store)

    body := strings.NewReader(`{"sku":"TEA-001","name":"Green tea"}`)
    request := httptest.NewRequest(http.MethodPost, "/v1/products", body)
    request.Header.Set("Content-Type", "application/json")
    response := httptest.NewRecorder()

    handler.Create(response, request)

    if response.Code != http.StatusCreated {
        t.Fatalf("status = %d, want %d; body = %s", response.Code, http.StatusCreated, response.Body.String())
    }
}
```

Cover at least:

- a valid create and list;
- missing or wrong content type;
- malformed JSON;
- unknown JSON field;
- two JSON values;
- body over the limit;
- missing required field;
- duplicate SKU;
- handler panic converted to a generic 500;
- request ID present in response and logs.

Use a fake logger or buffer in tests; do not assert unstable timestamps or entire
log lines when individual structured fields are the contract.

## 12. Optional stretch: opaque cursor pagination

An offset says how many rows to skip and becomes slow or unstable when data changes.
A cursor identifies the last position in a deterministic order.

For in-memory products ordered by `(sku, id)`, encode the last pair as an opaque,
URL-safe value. On the next request, return records strictly after that pair. Validate
decoded cursor shape and enforce a maximum page size. Do not expose a database
offset disguised as base64.

## 13. Knowledge check

1. Why can a safe method still write logs?
2. Why is `DELETE` idempotent even if the second response is 404?
3. What five checks belong in strict JSON decoding?
4. Which validation belongs in a handler, and which belongs in a use case?
5. Why must request ID wrap recovery and access logging?
6. Why must liveness avoid checking PostgreSQL?
7. What happens to existing handlers during `Server.Shutdown`?
8. Why should clients branch on a problem code rather than its detail text?

### Expected answers

1. Safety describes the requested application semantics, not incidental monitoring.
2. After either request, the intended resource state is absent.
3. Media type, byte limit, unknown fields, one JSON value, and field validation.
4. The handler validates the HTTP representation; the use case validates business
   rules and state transitions.
5. Both need the identifier even when a downstream handler panics.
6. A dependency outage should not trigger pointless application restart loops.
7. New connections stop while active handlers receive time to finish before the
   shutdown deadline.
8. Codes are stable machine contracts; detail text is for people and may change.

## 14. Completion evidence

Day 2 is complete when:

- handler tests cover the happy path and malformed, oversized, unknown-field,
  duplicate, and panic paths;
- the server has explicit header, read, write, idle, and shutdown limits;
- every public error has a stable problem shape and no internal details;
- you can trace one request from DNS through the handler and response;
- you can explain middleware execution order in both directions;
- graceful shutdown has a bounded test or manual demonstration;
- `gofmt`, `go test`, `go test -race`, and `go vet` pass.

Record one failure and why the final boundary prevents it. Then continue to Day 3.
