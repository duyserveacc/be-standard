# Day 5: Identity and API Security

This lesson expands Day 5 of the
[backend learning roadmap](../BACKEND_LEARNING_ROADMAP.md#day-5-identity-and-api-security).
It treats the API as an OAuth 2.0 resource server: an external identity provider
issues access tokens, while this service verifies them and enforces authorization.

## Learning outcomes

By the end of this lesson, you should be able to:

- separate authentication from authorization;
- identify the OAuth 2.0 and OpenID Connect participants relevant to an API;
- validate an access token's signature and security claims;
- handle JWKS caching and key rotation without disabling verification;
- introduce a trusted principal into request context;
- enforce role and object-level authorization;
- explain JWT, opaque-token, CORS, CSRF, and browser-session tradeoffs;
- prevent over-posting and sensitive-data leakage;
- test missing, invalid, expired, wrong-role, and wrong-owner access;
- write a compact threat model.

## 1. Authentication and authorization are separate decisions

**Authentication** answers: who or what is making this request, and how was that
identity verified?

**Authorization** answers: may that principal perform this action on this resource?

A valid token does not authorize every operation. Authentication middleware can
establish a principal; each use case or authorization boundary must still evaluate
the required role, scope, tenant, and object ownership.

Hiding a button in a frontend is not authorization. Attackers can call the API
directly.

## 2. Participants and token purpose

For this service:

- the **resource owner** is normally the user;
- the **client** is the frontend or other caller;
- the **authorization server / OpenID Provider** authenticates the user and issues
  tokens;
- the **resource server** is this Order API;
- an **access token** authorizes API access;
- an **ID token** tells the client about the authenticated session and should not be
  accepted by the API merely because it is a JWT.

OAuth 2.0 defines delegated authorization. OpenID Connect adds identity semantics
and discovery around OAuth 2.0. This project consumes identity; it does not implement
password storage, login UI, token issuance, recovery, or multifactor authentication.

## 3. What access-token validation must prove

For a signed JWT access token, validation is a conjunction of checks:

1. Parse the compact token with strict size and syntax limits.
2. Accept only explicitly configured signing algorithms.
3. Select a trusted key from the issuer's JWKS.
4. Verify the cryptographic signature.
5. Require the exact configured issuer (`iss`).
6. Require this API in the audience (`aud`).
7. Require expiration (`exp`) and reject expired tokens.
8. Honor `nbf` when present and apply only small configured clock skew.
9. Require a stable subject (`sub`).
10. Validate any application-required scope, role, tenant, or token-type claims.

Every check is necessary. Signature verification alone proves only that a holder of
the selected key signed the bytes. Without issuer and audience validation, a token
created for another system may be accepted here.

Never choose the validation algorithm from the token header without an allowlist.
Never skip signature validation because a token's payload looks plausible.

Use a maintained OIDC/JWT library rather than implementing cryptography or token
parsing manually. Pin and review the dependency when implementation begins.

## 4. Discovery and JWKS rotation

OIDC discovery publishes issuer metadata, including the JWKS URI. JWKS is a set of
public signing keys, commonly identified by `kid`.

A safe verifier:

- loads discovery only from the configured trusted issuer;
- requires the discovered issuer to match exactly;
- fetches JWKS over TLS;
- caches keys for a bounded period;
- refreshes on normal expiry;
- may trigger one rate-limited refresh when an unknown `kid` appears;
- keeps still-valid cached keys during a brief refresh failure;
- never accepts an unverifiable token merely because the issuer is unavailable;
- bounds response size, network time, cache size, and refresh concurrency.

During key rotation, the issuer should publish old and new public keys for an
overlap period. The service must accept tokens signed by either currently trusted
key. A test issuer can demonstrate rotation:

1. Publish key A and validate a token signed by A.
2. Publish keys A and B.
3. Validate a new token signed by B without restarting the service.
4. Remove A only after tokens signed by A have expired.
5. Confirm old A tokens then fail according to the expected lifecycle.

Do not fetch keys separately for every request; that turns authentication into an
unbounded network dependency and creates an easy denial-of-service path.

## 5. Build a trusted principal

After complete token validation, map only required claims into an application type:

```go
type Principal struct {
    Subject string
    Roles   map[string]struct{}
}

func (p Principal) HasRole(role string) bool {
    _, ok := p.Roles[role]
    return ok
}
```

Store the principal in request context with an unexported typed key. Expose helper
functions rather than allowing packages to depend on raw claim maps.

```go
type principalContextKey struct{}

func WithPrincipal(ctx context.Context, principal Principal) context.Context {
    return context.WithValue(ctx, principalContextKey{}, principal)
}

func PrincipalFromContext(ctx context.Context) (Principal, bool) {
    principal, ok := ctx.Value(principalContextKey{}).(Principal)
    return principal, ok
}
```

Context carries the request-scoped verified identity, not an arbitrary bag of
optional parameters.

Authentication middleware behavior:

- missing bearer token: return 401 with a stable problem;
- malformed scheme or token: return 401;
- invalid signature or claims: return 401;
- valid token: attach the principal and continue;
- verifier infrastructure failure that prevents verification: fail closed with a
  generic 503 and log the internal reason. Do not misreport it as bad credentials.

Do not log the Authorization header or the token.

## 6. Role authorization belongs near the action

Product creation requires an administrator. The handler can require an authenticated
principal, but the application operation should enforce the role so another adapter
cannot bypass it accidentally.

```go
var ErrForbidden = errors.New("forbidden")

func (s *Service) CreateProduct(ctx context.Context, actor Principal, input NewProduct) (Product, error) {
    if !actor.HasRole("admin") {
        return Product{}, ErrForbidden
    }
    // Validate and create.
}
```

Return 401 when authentication is absent or invalid. Return 403 when authentication
succeeded but the known principal lacks permission. In cases where revealing a
resource's existence would leak information, return the same 404 for wrong owner
and nonexistent object.

Roles should be narrowly defined. Avoid a universal `admin` bypass scattered across
handlers; centralize which operation requires which capability.

## 7. Object-level authorization belongs in data access

The customer order query must include ownership:

```sql
SELECT id, customer_id, status, total_cents, created_at
FROM orders
WHERE id = $1
  AND customer_id = $2;
```

The service passes `principal.Subject` as the trusted customer identity. It never
accepts `customer_id` from the request body or query string for a customer endpoint.

For list access:

```sql
SELECT id, customer_id, status, total_cents, created_at
FROM orders
WHERE customer_id = $1
  AND (created_at, id) < ($2, $3)
ORDER BY created_at DESC, id DESC
LIMIT $4;
```

This prevents insecure direct object references even if an attacker guesses an
order ID. Test the SQL-backed behavior, not only a handler check.

## 8. JWTs and opaque tokens have different tradeoffs

A signed JWT can be validated locally, reducing per-request issuer calls. Its claims
remain valid until expiration unless the system adds a revocation mechanism. Keep
access-token lifetimes bounded and avoid sensitive data in the payload: JWT payloads
are encoded, not encrypted by default.

An opaque token carries no locally readable claims. The resource server may call an
introspection endpoint or use a gateway-provided verified identity. This can support
centralized revocation but adds latency, availability, caching, and credential
requirements.

Choose based on the identity provider and threat model. Do not infer token type by
whether a string happens to contain dots.

## 9. Browser sessions, CORS, and CSRF

CORS is a browser rule controlling whether frontend JavaScript may read or send
certain cross-origin requests. It is not authentication and does not stop curl,
server-to-server callers, or same-origin attackers.

Configure exact allowed origins, methods, and headers. Do not combine wildcard
origins with credentials. Keep preflight behavior tested.

CSRF matters when a browser automatically attaches credentials, especially cookies.
Mitigations include appropriate `SameSite` cookies, CSRF tokens, origin checking,
and avoiding unsafe state changes through safe methods. A bearer token explicitly
attached by JavaScript is not automatically sent like a cookie, but XSS and token
storage risks remain.

Cookies carrying sessions should be `Secure`, `HttpOnly` when JavaScript access is
unnecessary, have an appropriate `SameSite` policy, narrow path/domain scope, and a
bounded lifetime. Logout must define whether it clears only the browser session or
also revokes server-side state.

## 10. Prevent common API data exposures

### Over-posting and mass assignment

Decode into a request DTO containing only client-writable fields. Never decode
directly into a persistence or domain type with fields such as role, owner, price,
status, or audit timestamps.

### Injection

Continue using parameterized SQL. Token claims and authenticated IDs are still data,
not trusted SQL syntax.

### Logging and telemetry

Do not record:

- bearer tokens or cookies;
- passwords, authorization codes, or refresh tokens;
- raw request/response bodies by default;
- full personal data;
- private signing material;
- database URLs containing credentials.

Log bounded identifiers needed for diagnosis: request ID, route template, public
error code, subject only when policy permits, and authorization decision category.

### Secrets and least privilege

Load secrets from the deployment environment or secret manager, not source control.
The API database role should have only the required schema and statement privileges.
Migration credentials can be separate because schema modification is more powerful.

### Abuse controls

Set body, header, timeout, page-size, and concurrency limits. Rate limits should be
bounded by a meaningful key such as trusted client or principal, consider proxy
address trust, and return a clear 429 response. A rate limiter complements rather
than replaces authorization.

## 11. Security test matrix

Add deterministic tests for:

| Case | Expected result |
| --- | --- |
| Missing token | 401 |
| Malformed bearer scheme | 401 |
| Invalid signature | 401 |
| Disallowed algorithm | 401 |
| Wrong issuer | 401 |
| Wrong audience | 401 |
| Expired token | 401 |
| Token not yet valid beyond allowed skew | 401 |
| Valid customer token | Customer endpoint proceeds |
| Customer calls admin product creation | 403 |
| Admin creates product | Operation proceeds |
| Customer reads own order | 200 |
| Customer reads another customer's order | Same 404 as nonexistent order |
| Signing key rotates | Old and new valid during overlap |
| JWKS refresh temporarily fails | Cached valid keys continue; unknown key fails closed |

Capture logs in tests and assert that token text and authorization headers do not
appear. Generate test keys locally; never use production keys in fixtures.

## 12. Compact threat model

Record at least this table and adapt it to the implementation:

| Asset | Threat | Boundary | Mitigation | Evidence |
| --- | --- | --- | --- | --- |
| Customer orders | ID guessing reads another user's order | Client to API | Owner-scoped query | Wrong-owner integration test |
| Admin operations | Customer forges or reuses privilege | Token to use case | Signature, claims, and role checks | Wrong-role tests |
| Access tokens | Token leaks through logs | API to telemetry | Redaction and field allowlist | Captured-log test |
| Inventory | Concurrent order oversells | API to PostgreSQL | Atomic conditional update | Last-unit test |
| Signing trust | Unknown key accepted during outage | API to issuer | Bounded cache, refresh, fail closed | Rotation/outage tests |
| Database | Compromised app modifies schema | API to database | Least-privilege runtime role | Privilege verification |

For each threat, name the attacker capability, entry point, impact, prevention,
detection, and recovery. A threat model is useful only when it changes design or
produces verifiable controls.

## 13. Guided build

Complete the work in this order:

1. Add an `auth` platform package containing principal and verifier boundaries.
2. Configure one trusted issuer and audience through validated startup config.
3. Use a maintained library to validate signed access tokens and claims.
4. Add bounded discovery/JWKS caching and a local rotating test issuer.
5. Attach a verified principal in authentication middleware.
6. Enforce admin authorization inside product creation.
7. Pass the principal subject into place/list/get order use cases.
8. Scope order SQL by customer ID.
9. Add the complete negative test matrix and log-redaction checks.
10. Write the compact threat model.

Do not add password login or token issuance. Those are separate security products
and distract from learning resource-server responsibilities.

## 14. Knowledge check

1. What is the difference between authentication and authorization?
2. Why must an API validate audience after verifying a JWT signature?
3. Why is an ID token not automatically an API access token?
4. What must happen when an unknown signing key appears?
5. Why should role authorization also exist in the application operation?
6. Why scope the order query instead of loading then checking ownership?
7. Why does CORS not protect an API from non-browser callers?
8. When is CSRF relevant?
9. Why is decoding JSON into a database model dangerous?

### Expected answers

1. Authentication establishes identity; authorization decides whether that identity
   may perform a particular action on a resource.
2. A correctly signed token may have been issued for a different resource server.
3. It communicates authentication to a client and can have a different audience and
   purpose.
4. Perform at most a bounded, rate-limited JWKS refresh; accept only if verification
   succeeds with a trusted key, otherwise fail closed.
5. It protects the behavior when called by a non-HTTP adapter or a future refactor.
6. Unauthorized data never enters application memory and the check cannot be
   accidentally omitted later.
7. CORS is enforced by browsers, not by curl or arbitrary servers.
8. When a browser automatically attaches ambient credentials to a state-changing
   request.
9. It may allow clients to set ownership, role, status, price, or audit fields they
   must not control.

## 15. Completion evidence

Day 5 is complete when:

- token validation requires trusted algorithm, signature, issuer, audience, time,
  and subject claims;
- JWKS rotation succeeds without accepting unverifiable tokens;
- product administration is enforced in the use case;
- all order reads and lists are scoped by the authenticated subject;
- wrong-owner and nonexistent order responses are indistinguishable externally;
- the complete negative-token and wrong-role test matrix passes;
- captured telemetry contains no tokens, credentials, or raw personal payloads;
- the threat model maps every listed threat to implementation evidence;
- formatting, tests, race checks, vet, and vulnerability checks pass.

Then continue to Day 6.
