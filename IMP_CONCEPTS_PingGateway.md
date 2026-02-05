# PingGateway (IG) — Core Concepts Deep Dive

## 1. The Mental Model

PingGateway is a **programmable reverse proxy**. Every request flows through this pipeline:

```
                          ┌─── Route Match ───┐
                          │                    │
Client ──► PingGateway ──►│ Filter → Filter ──►│──► Handler ──► Backend
                          │   ▲         ▲      │       │
                          │   │         │      │       ▼
                          │ modify   modify    │    Response
                          │ request  request   │       │
                          │   │         │      │       ▼
Client ◄── PingGateway ◄──│ Filter ← Filter ◄─│◄── Handler
                          │ modify   modify    │
                          │ response response  │
                          └────────────────────┘
```

**Key insight**: Filters are **bidirectional**. They process the request on the way in AND the response on the way out. A Handler is the **terminal point** — it produces the response (either from a backend or statically).

---

## 2. The Three Config Files

### `config.json` — The Entry Point (loaded once at startup)

This is what PingGateway reads first. It defines the **top-level handler** and **global heap objects**.

```json
{
  "handler": {
    "type": "Router",
    "name": "_router",
    "capture": "all"
  },
  "heap": [
    {
      "name": "JwtSession",
      "type": "JwtSession"
    },
    {
      "name": "capture",
      "type": "CaptureDecorator",
      "config": {
        "captureEntity": true,
        "captureContext": true
      }
    }
  ]
}
```

Breaking it down:

| Field | What it does |
|---|---|
| `handler` | The top-level handler that receives ALL incoming requests. `Router` type scans `config/routes/` for route files. |
| `handler.capture` | `"all"` enables CaptureDecorator logging for every request (debugging) |
| `heap` | Shared objects available to ALL routes. Defined once, referenced by name. |
| `heap[0]` JwtSession | Enables JWT-based session cookies (PingGW can cache auth state) |
| `heap[1]` CaptureDecorator | Logs request/response bodies + context to log files |

**Think of it as**: The `main()` function. The Router is the dispatcher that reads route files and delegates.

### `admin.json` — The Management API (loaded once at startup)

```json
{
  "prefix": "openig",
  "allowedHosts": ["localhost", "pinggw"]
}
```

| Field | What it does |
|---|---|
| `prefix` | URL prefix for admin API → `/openig/api/info`, `/openig/api/system/objects` |
| `allowedHosts` | Security restriction — only these hosts can access admin API |

The admin API runs on the **same port** as routes (8080). It's protected by host restriction, not a separate port.

### `routes/*.json` — Individual Request Rules (auto-reloaded ~10s)

Each file = one routing rule. Evaluated in **alphabetical order** by filename (hence `01-`, `02-` prefixes).

---

## 3. Anatomy of a Route

Every route has three parts: **condition**, **handler**, and optionally **heap** + **filters**.

### Simplest possible route — just a handler:

```json
{
  "name": "04-headers",
  "condition": "${find(request.uri.path, '^/headers')}",
  "handler": {
    "type": "StaticResponseHandler",
    "config": {
      "status": 200,
      "headers": { "Content-Type": ["application/json"] },
      "entity": "{\"ip\": \"${contexts.client.remoteAddress}\"}"
    }
  }
}
```

```
Request: GET /headers
         │
         ▼
    Condition: path starts with /headers? → YES
         │
         ▼
    StaticResponseHandler → returns hardcoded JSON
         │
         ▼
Response: 200 {"ip": "172.28.0.1"}
```

### Route with filters — Chain handler:

```json
{
  "name": "01-simple-proxy",
  "condition": "${find(request.uri.path, '^/sample')}",
  "handler": {
    "type": "Chain",
    "config": {
      "filters": [
        {
          "type": "UriPathRewriteFilter",
          "config": {
            "mappings": { "/sample": "/" }
          }
        }
      ],
      "handler": {
        "type": "ReverseProxyHandler",
        "baseURI": "http://sample-app:8081"
      }
    }
  }
}
```

```
Request: GET /sample/home
         │
         ▼
    Condition: path starts with /sample? → YES
         │
         ▼
    Chain
    ├── Filter 1: UriPathRewriteFilter
    │   rewrites /sample/home → /home
    │         │
    │         ▼
    └── Handler: ReverseProxyHandler
        sends GET /home to http://sample-app:8081
                  │
                  ▼
        Backend returns HTML
                  │
                  ▼
    Response flows back through filters (no modification)
         │
         ▼
Response: 200 <html>...</html>
```

### Route with heap objects — AM integration:

```json
{
  "name": "02-sso",
  "condition": "${find(request.uri.path, '^/sso')}",
  "heap": [
    {
      "name": "AmService-1",
      "type": "AmService",
      "config": {
        "url": "http://pingam:8080/am",
        "realm": "/techcorp",
        "agent": {
          "username": "ig_agent",
          "passwordSecretId": "agent.password"
        },
        "secretsProvider": { ... }
      }
    }
  ],
  "handler": {
    "type": "Chain",
    "config": {
      "filters": [
        {
          "type": "SingleSignOnFilter",
          "config": { "amService": "AmService-1" }
        },
        {
          "type": "HeaderFilter",
          "config": {
            "messageType": "REQUEST",
            "add": {
              "X-IG-User": ["${contexts.ssoToken.info.uid}"]
            }
          }
        }
      ],
      "handler": {
        "type": "ReverseProxyHandler",
        "baseURI": "http://sample-app:8081"
      }
    }
  }
}
```

```
Request: GET /sso/home (no AM cookie)
         │
         ▼
    Condition: path starts with /sso? → YES
         │
         ▼
    Chain
    ├── Filter 1: SingleSignOnFilter
    │   checks for iPlanetDirectoryPro cookie
    │   → MISSING → redirect to AM login
    │
    │   (user logs in, gets cookie, browser retries)
    │
    ├── Filter 1: SingleSignOnFilter (retry with cookie)
    │   validates cookie against AM REST API
    │   → VALID → populates contexts.ssoToken
    │         │
    │         ▼
    ├── Filter 2: HeaderFilter
    │   adds X-IG-User: demo (from contexts.ssoToken.info.uid)
    │         │
    │         ▼
    └── Handler: ReverseProxyHandler
        sends request to backend WITH X-IG-User header
```

---

## 4. Handlers — The Terminal Points

A Handler **produces a response**. It's the end of the chain. Every route must have exactly one handler at the end.

### ReverseProxyHandler (most common)

Forwards the request to a backend server and returns its response.

```json
{
  "type": "ReverseProxyHandler",
  "baseURI": "http://backend:8080",
  "config": {
    "connectionTimeout": "10 seconds",
    "soTimeout": "10 seconds"
  }
}
```

**Use case**: The default. Proxy traffic to your application.

### StaticResponseHandler

Returns a hardcoded response without calling any backend.

```json
{
  "type": "StaticResponseHandler",
  "config": {
    "status": 200,
    "headers": { "Content-Type": ["application/json"] },
    "entity": "{\"status\": \"ok\"}"
  }
}
```

**Use cases**: Health checks, maintenance pages, mock responses, debugging.

### Chain (the glue)

Not really a handler — it's a **container** that wraps filters around a handler.

```json
{
  "type": "Chain",
  "config": {
    "filters": [ ... ],     ← processed in order, request IN
    "handler": { ... }       ← terminal point
  }                          ← filters processed in reverse, response OUT
}
```

**Every time you need filters, you need a Chain.**

### Router

Reads route files from a directory and dispatches requests to matching routes.

```json
{
  "type": "Router",
  "config": {
    "directory": "${openig.configDirectory}/routes",
    "scanInterval": "10 seconds"
  }
}
```

**Use case**: Only in `config.json` as the top-level handler. You rarely define this in routes.

### DispatchHandler

Routes to different handlers based on conditions (like a switch statement).

```json
{
  "type": "DispatchHandler",
  "config": {
    "bindings": [
      {
        "condition": "${request.method == 'GET'}",
        "handler": { "type": "ReverseProxyHandler", "baseURI": "http://read-service:8080" }
      },
      {
        "condition": "${request.method == 'POST'}",
        "handler": { "type": "ReverseProxyHandler", "baseURI": "http://write-service:8080" }
      }
    ]
  }
}
```

**Use case**: Route GETs to read replicas, POSTs to primary. Route by header values, methods, etc.

### ScriptableHandler

Custom Groovy/JavaScript logic that generates a response.

```json
{
  "type": "ScriptableHandler",
  "config": {
    "type": "application/x-groovy",
    "file": "CustomHandler.groovy"
  }
}
```

```groovy
// CustomHandler.groovy
import org.forgerock.http.protocol.*

return new Response(Status.OK)
    .setEntity([message: "Hello from Groovy", user: contexts.ssoToken?.info?.uid])
```

**Use case**: Complex response generation, aggregation of multiple backends, transformation.

### ClientHandler / ForgeRockClientHandler

Makes outbound HTTP calls (used inside filters to call external APIs).

```json
{ "type": "ForgeRockClientHandler" }
```

**Use case**: Not a terminal route handler — used inside filters when they need to call AM, introspection endpoints, external APIs.

---

## 5. Filters — The Processing Pipeline

Filters **modify requests and/or responses** as they pass through. They're always inside a `Chain`.

### Execution order matters:

```json
"filters": [
  { "type": "FilterA" },   ← 1st on request, 3rd on response
  { "type": "FilterB" },   ← 2nd on request, 2nd on response
  { "type": "FilterC" }    ← 3rd on request, 1st on response
]
```

Request:  FilterA → FilterB → FilterC → Handler
Response: FilterA ← FilterB ← FilterC ← Handler

### Category 1: Authentication Filters — "Who are you?"

#### SingleSignOnFilter — AM Session Validation

```json
{
  "type": "SingleSignOnFilter",
  "config": {
    "amService": "AmService-1"
  }
}
```

What it does:
1. Looks for `iPlanetDirectoryPro` cookie
2. No cookie → **302 redirect** to AM login page (with `goto` return URL)
3. Has cookie → calls AM REST API to validate session
4. Invalid → redirect to login
5. Valid → populates `contexts.ssoToken` with session data, continues chain

**This is the #1 most important filter** — it's what replaces Web Agents.

#### OAuth2ResourceServerFilter — Bearer Token Validation

```json
{
  "type": "OAuth2ResourceServerFilter",
  "config": {
    "scopes": ["profile", "email"],
    "requireHttps": false,
    "accessTokenResolver": {
      "type": "TokenIntrospectionAccessTokenResolver",
      "config": {
        "amService": "AmService-1"
      }
    }
  }
}
```

What it does:
1. Extracts `Authorization: Bearer <token>` header
2. No token → **401 Unauthorized** with `WWW-Authenticate: Bearer` header
3. Calls AM's `/oauth2/introspect` endpoint
4. Token invalid or expired → **401**
5. Token valid but missing required scopes → **403 Forbidden**
6. Valid → populates `contexts.oauth2.accessToken` with token info

Token resolver options:
- `TokenIntrospectionAccessTokenResolver` — calls AM per request (works with opaque tokens)
- `OpenIdConnectTokenResolver` — validates JWT locally via JWKS (no AM call — faster)
- `StatelessAccessTokenResolver` — validates stateless tokens locally

#### OAuth2ClientFilter — Acts as OIDC Relying Party

```json
{
  "type": "OAuth2ClientFilter",
  "config": {
    "clientEndpoint": "/oauth2/callback",
    "loginHandler": {
      "failureHandler": { "type": "StaticResponseHandler", "config": { "status": 401 } }
    },
    "registrations": ["OidcRegistration"]
  }
}
```

What it does:
1. User hits protected page → PingGateway redirects to AM's `/authorize` endpoint
2. AM authenticates user, returns authorization code
3. PingGateway exchanges code for tokens, stores in IG session
4. Populates `contexts.oauth2.accessToken` and `contexts.openid.idToken`

**Key difference from SingleSignOnFilter**: SSO uses AM cookies, this uses standard OAuth2/OIDC flow.
**Use case**: Web apps that need OIDC login but can't implement it themselves.

#### CrossDomainSingleSignOnFilter — CDSSO for different domains

```json
{
  "type": "CrossDomainSingleSignOnFilter",
  "config": {
    "amService": "AmService-1",
    "authCookie": {
      "name": "ig-token"
    }
  }
}
```

Same as SingleSignOnFilter but handles the cross-domain cookie problem. Creates its own cookie on PingGateway's domain after validating with AM.
**Use case**: AM on `sso.company.com`, apps on `apps.company.com`.

#### HttpBasicAuthenticationClientFilter — Basic auth on outbound requests

```json
{
  "type": "HttpBasicAuthenticationClientFilter",
  "config": {
    "username": "ig_agent",
    "passwordSecretId": "agent.password",
    "secretsProvider": { ... }
  }
}
```

Adds `Authorization: Basic base64(user:pass)` header to outbound calls.
**Not for authenticating users** — for PingGateway authenticating itself to other services (AM introspection, backends).

### Category 2: Authorization Filters — "Are you allowed?"

#### PolicyEnforcementFilter — AM Policy Evaluation

```json
{
  "type": "PolicyEnforcementFilter",
  "config": {
    "pepRealm": "/techcorp",
    "application": "TechCorpAPI",
    "ssoTokenSubject": "${contexts.ssoToken.value}",
    "amService": "AmService-1"
  }
}
```

What it does:
1. Sends policy evaluation request to AM (resource URL + subject + environment)
2. AM returns allow/deny per action (GET, POST, PUT, DELETE)
3. Deny → **403 Forbidden**

**Must come after an auth filter** — needs `contexts.ssoToken` or `contexts.oauth2`.

#### TokenTransformationFilter — Exchange one token type for another

```json
{
  "type": "TokenTransformationFilter",
  "config": {
    "amService": "AmService-1",
    "idToken": "${contexts.oauth2.accessToken.info.id_token}",
    "instance": "oidc-to-saml"
  }
}
```

Uses AM's Security Token Service (STS). **Use case**: Frontend uses OIDC, backend requires SAML assertion.

### Category 3: Request Modification Filters — "Change what goes to the backend"

#### HeaderFilter — Add/Remove/Modify HTTP headers

```json
{
  "type": "HeaderFilter",
  "config": {
    "messageType": "REQUEST",
    "add": {
      "X-Custom-Header": ["value1"],
      "X-Forwarded-User": ["${contexts.ssoToken.info.uid}"]
    },
    "remove": ["Authorization"],
    "set": {
      "X-Request-Id": ["${java.util.UUID.randomUUID()}"]
    }
  }
}
```

- `messageType`: `REQUEST` (to backend) or `RESPONSE` (to client)
- `add`: Add headers (preserves existing)
- `remove`: Strip headers
- `set`: Replace headers (removes existing, adds new)

**Use case**: Inject user identity into backend requests. Strip sensitive headers. Add CORS headers on responses.

#### UriPathRewriteFilter — Path Manipulation

```json
{
  "type": "UriPathRewriteFilter",
  "config": {
    "mappings": {
      "/sample": "/",
      "/api/v2": "/api"
    }
  }
}
```

Rewrites the request path before forwarding. `/sample/home` → `/home`.
**Use case**: Gateway path prefix differs from backend path.

#### AssignmentFilter — Set arbitrary values on request/response

```json
{
  "type": "AssignmentFilter",
  "config": {
    "onRequest": [
      { "target": "${request.uri.query}", "value": "api_key=internal123" },
      { "target": "${request.headers['X-Request-Id'][0]}", "value": "${java.util.UUID.randomUUID()}" }
    ],
    "onResponse": [
      { "target": "${response.headers['X-Powered-By']}", "value": "" }
    ]
  }
}
```

Low-level manipulation of any request/response property.
**Use case**: Inject API keys, add request IDs, strip server identity headers.

#### EntityExtractFilter — Parse response body and extract values

```json
{
  "type": "EntityExtractFilter",
  "config": {
    "messageType": "response",
    "target": "${attributes.extractedToken}",
    "bindings": [
      { "key": "token", "pattern": "name=\"csrf_token\" value=\"(.+?)\"" }
    ]
  }
}
```

Regex extraction from HTML/response body.
**Use case**: Extract CSRF tokens from legacy login pages (used with PasswordReplayFilter).

#### PasswordReplayFilter — Submit credentials to legacy login forms

```json
{
  "type": "PasswordReplayFilter",
  "config": {
    "loginPage": "${find(request.uri.path, '/login')}",
    "loginPageContentMarker": "<form",
    "request": {
      "method": "POST",
      "uri": "http://legacy-app:8080/j_security_check",
      "form": {
        "j_username": ["${contexts.ssoToken.info.uid}"],
        "j_password": ["${attributes.userPassword}"]
      }
    }
  }
}
```

Detects a legacy app's login page → submits credentials automatically.
**Use case**: SSO for apps that can't be modified. One of PingGateway's most valuable differentiators.

### Category 4: Security Filters — "Protect the system"

#### ThrottlingFilter — Rate Limiting

```json
{
  "type": "ThrottlingFilter",
  "config": {
    "requestGroupingPolicy": "${contexts.oauth2.accessToken.info.client_id}",
    "rate": {
      "numberOfRequests": 100,
      "duration": "1 minute"
    }
  }
}
```

Group by client_id, IP, user, or any expression. Exceeds limit → **429 Too Many Requests**.

#### CorsFilter — Cross-Origin Resource Sharing

```json
{
  "type": "CorsFilter",
  "config": {
    "policies": [
      {
        "acceptedOrigins": ["https://app.company.com"],
        "acceptedMethods": ["GET", "POST", "PUT", "DELETE"],
        "acceptedHeaders": ["Authorization", "Content-Type"],
        "exposedHeaders": ["X-Request-Id"],
        "maxAge": 3600
      }
    ]
  }
}
```

Handles preflight OPTIONS requests. Adds `Access-Control-Allow-*` headers.
**Use case**: APIs called from browser JavaScript on different origins.

#### ContentTypeFilter — Validate Content-Type

```json
{
  "type": "ContentTypeFilter",
  "config": {
    "messageType": "REQUEST",
    "allowedMediaTypes": ["application/json", "application/xml"]
  }
}
```

Rejects requests with unexpected content types → **415 Unsupported Media Type**.

### Category 5: Conditional / Flow Control Filters — "When to apply"

#### ConditionalFilter — Apply a filter only when condition is true

```json
{
  "type": "ConditionalFilter",
  "config": {
    "condition": "${request.method == 'POST' or request.method == 'PUT'}",
    "delegate": {
      "type": "ScriptableFilter",
      "config": { "file": "ValidateBody.groovy" }
    }
  }
}
```

Condition false → filter is skipped, request passes through.
**Use case**: Only validate body on write operations. Only rate-limit anonymous users.

#### SwitchFilter — Different behavior based on conditions (like switch/case)

```json
{
  "type": "SwitchFilter",
  "config": {
    "onRequest": [
      {
        "condition": "${find(request.uri.path, '^/admin')}",
        "handler": { "type": "StaticResponseHandler", "config": { "status": 403 } }
      },
      {
        "condition": "${find(request.uri.path, '^/health')}",
        "handler": { "type": "StaticResponseHandler", "config": { "status": 200, "entity": "OK" } }
      }
    ]
  }
}
```

**Use case**: Different behavior for different paths within the same route.

### Category 6: Debugging / Observability Filters — "What's happening?"

#### CaptureDecorator — Log full request/response details

Technically a decorator, not a filter. Defined in heap:

```json
{
  "name": "capture",
  "type": "CaptureDecorator",
  "config": {
    "captureEntity": true,
    "captureContext": true
  }
}
```

Then on any handler or filter: `"capture": "all"` or `"capture": ["request", "response"]`.
Logs go to `/var/ig/logs/route-*.log`.

#### CaptureFilter — Capture as an inline filter (instead of decorator)

```json
{
  "type": "CaptureFilter",
  "config": {
    "captureEntity": true
  }
}
```

**Use case**: Log request/response at a specific point in the chain (between two filters).

### Category 7: Custom Logic — "Do anything"

#### ScriptableFilter — Write Groovy/JavaScript

```json
{
  "type": "ScriptableFilter",
  "config": {
    "type": "application/x-groovy",
    "file": "CustomRiskFilter.groovy",
    "args": {
      "riskThreshold": 75,
      "blockCountries": ["XX", "YY"]
    }
  }
}
```

```groovy
// CustomRiskFilter.groovy — example risk scoring filter
import org.forgerock.http.protocol.*

// Access request data
def clientIp = contexts.client.remoteAddress
def userAgent = request.headers['User-Agent']?.firstValue
def path = request.uri.path

// Access args passed from route JSON
def threshold = args.riskThreshold

// Custom risk logic
def riskScore = 0
if (clientIp?.startsWith("10.")) riskScore += 10      // internal
if (userAgent?.contains("curl")) riskScore += 30       // scripted
if (path?.contains("admin")) riskScore += 40           // sensitive

// Add risk info as headers for backend
request.headers.put("X-Risk-Score", riskScore.toString())
request.headers.put("X-Client-IP", clientIp ?: "unknown")

// Block high-risk requests
if (riskScore > threshold) {
    def response = new Response(Status.FORBIDDEN)
    response.entity.json = [error: "Risk score too high", score: riskScore]
    return response    // Short-circuit — never reaches handler
}

// Continue to next filter/handler
return next.handle(context, request)
```

**Key concepts**:
- `next.handle(context, request)` — passes to next filter/handler (MUST call this to continue)
- Return a `Response` directly to short-circuit (block, redirect, mock)
- `request` — the HTTP request (headers, URI, entity/body, method)
- `context` — runtime context tree (ssoToken, oauth2, client, session)
- `args` — values passed from the route JSON config
- Can modify request on way in AND response on way out:

```groovy
// Modify request, call next, then modify response
def start = System.currentTimeMillis()
def response = next.handle(context, request).get()
response.headers.put("X-Processing-Time", (System.currentTimeMillis() - start).toString())
return response
```

**This is the most powerful filter** — your org's `CustomRiskFilter` and similar are all ScriptableFilters.

### Filter Cheat Sheet — Interview Priority

| Priority | Filter | Category | One-liner |
|---|---|---|---|
| 1 | SingleSignOnFilter | Auth | Replaces Web Agents — AM cookie SSO |
| 2 | OAuth2ResourceServerFilter | Auth | API protection — validates bearer tokens |
| 3 | ScriptableFilter | Custom | Custom business logic in Groovy |
| 4 | HeaderFilter | Request Mod | Injects identity info for backends |
| 5 | PolicyEnforcementFilter | Authz | AM policy evaluation (PEP) |
| 6 | OAuth2ClientFilter | Auth | PingGW acts as OIDC Relying Party |
| 7 | ThrottlingFilter | Security | Rate limiting |
| 8 | PasswordReplayFilter | Request Mod | Legacy app SSO via form submission |
| 9 | ConditionalFilter | Flow | Conditional filter application |
| 10 | CorsFilter | Security | CORS handling for browser APIs |
| 11 | AssignmentFilter | Request Mod | Low-level request/response property manipulation |
| 12 | EntityExtractFilter | Request Mod | Regex extraction from response body |
| 13 | SwitchFilter | Flow | Switch/case routing within a route |
| 14 | ContentTypeFilter | Security | Content-Type validation |
| 15 | CaptureFilter | Debug | Log request/response at a specific chain point |
| 16 | HttpBasicAuthenticationClientFilter | Auth | Basic auth on PingGW's outbound calls |
| 17 | CrossDomainSingleSignOnFilter | Auth | CDSSO for cross-domain |
| 18 | TokenTransformationFilter | Authz | Token exchange (OIDC → SAML) |

---

## 6. Heap Objects — Shared Dependencies

Heap objects are **reusable components** defined once, referenced by name. They live in `config.json` (global) or in a route's `heap` array (route-local).

### AmService — Connection to PingAM

```json
{
  "name": "AmService-1",
  "type": "AmService",
  "config": {
    "url": "http://pingam:8080/am",
    "realm": "/techcorp",
    "agent": {
      "username": "ig_agent",
      "passwordSecretId": "agent.password"
    },
    "secretsProvider": {
      "type": "Base64EncodedSecretStore",
      "config": {
        "secrets": {
          "agent.password": "Y2hhbmdlaXQ="
        }
      }
    },
    "sessionProperties": ["uid", "cn"]
  }
}
```

Referenced by filters: `"amService": "AmService-1"`. Handles:
- Agent authentication to AM
- Session validation calls
- Token introspection calls
- Policy evaluation calls
- Notification subscriptions (session logout events)

### Where to define heap objects:

| Location | Scope | When to use |
|---|---|---|
| `config.json` → `heap` | All routes | AmService shared by multiple routes |
| Route file → `heap` | That route only | Route-specific objects |
| Inline in filter/handler | That filter only | One-off objects |

---

## 7. Expressions — The `${}` Syntax

PingGateway uses **expression language** (EL) everywhere for dynamic values.

### Request data:
```
${request.uri.path}                          → /api/users/123
${request.uri.query}                         → name=demo&role=admin
${request.method}                            → GET
${request.headers['Authorization'][0]}       → Bearer eyJhbG...
${request.headers['User-Agent'][0]}          → Mozilla/5.0...
${request.form['username'][0]}               → demo (POST form data)
${request.entity.json.name}                  → (parsed JSON body)
```

### Context data (populated by filters):
```
${contexts.client.remoteAddress}             → 172.28.0.1
${contexts.ssoToken.value}                   → AQIC5w... (session cookie value)
${contexts.ssoToken.info.uid}                → demo
${contexts.ssoToken.info.cn}                 → Demo User
${contexts.oauth2.accessToken.info.sub}      → demo
${contexts.oauth2.accessToken.info.client_id}→ techcorp-app
${contexts.oauth2.accessToken.scopes}        → [profile, email]
```

### Environment:
```
${env['AM_URL']}                             → value of AM_URL env variable
${system['ig.agent.password']}               → JVM system property
```

### Functions:
```
${find(request.uri.path, '^/api')}           → true/false (regex match)
${matches(request.uri.path, '^/api/(.*)')}   → captures groups
${toString(request.headers['Accept'])}       → joins array to string
${empty contexts.ssoToken}                   → true if no SSO token
```

---

## 8. Context Chain — How Data Flows Between Filters

Each filter can **add data to the context** that downstream filters and handlers can read:

```
Request arrives
    │
    ▼
SingleSignOnFilter
    adds: contexts.ssoToken.info.uid = "demo"
    adds: contexts.ssoToken.value = "AQIC5w..."
    │
    ▼
OAuth2ResourceServerFilter (if used instead)
    adds: contexts.oauth2.accessToken.info.sub = "demo"
    adds: contexts.oauth2.accessToken.scopes = ["profile"]
    │
    ▼
PolicyEnforcementFilter
    reads: contexts.ssoToken.value (to identify user)
    adds: contexts.policyDecision.actions = {GET: true, POST: false}
    │
    ▼
HeaderFilter
    reads: contexts.ssoToken.info.uid → injects as X-IG-User header
    │
    ▼
ScriptableFilter
    reads: anything from contexts
    adds: custom attributes
    │
    ▼
Handler → Backend receives request with injected headers
```

---

## 9. Route Evaluation Order

```
Request: GET /api/users

Router scans routes/ alphabetically:
  01-simple-proxy.json → condition: ^/sample → NO MATCH
  02-sso.json          → condition: ^/sso    → NO MATCH
  03-oauth2-rs.json    → condition: ^/api    → MATCH → process this route
  04-headers.json      → (never checked)
```

- First match wins
- Use numeric prefixes for explicit ordering
- More specific conditions should come first
- A catch-all route (`99-default.json` with no condition) handles unmatched requests

---

## 10. How Your Org's ScriptableFilters Relate

Your org's `CustomRiskFilter` and similar are just ScriptableFilters with business logic:

```
Route JSON                           Groovy Script
─────────                            ─────────────
{                                    // CustomRiskFilter.groovy
  "filters": [
    {                                // Read request data
      "type": "ScriptableFilter",   def ip = contexts.client.remoteAddress
      "config": {                   def token = contexts.oauth2.accessToken
        "file": "CustomRisk.groovy",
        "args": {                    // Business logic
          "threshold": 75            def score = calculateRisk(ip, token)
        }
      }                              // Decision
    }                                if (score > args.threshold) {
  ]                                    return new Response(Status.FORBIDDEN)
}                                    }
                                     return next.handle(context, request)
```

The `systemFilters` and `systemHandlers` in your org are likely:
- **systemFilters**: Groovy filters applied to all routes (logging, tracing, correlation IDs)
- **systemHandlers**: Custom handlers for special endpoints (health, metrics)

These would be registered as heap objects in `config.json` and referenced across routes.

---

## 11. Complete Request Lifecycle (Putting It All Together)

```
1. Browser: GET http://localhost:8083/sso/dashboard

2. PingGateway receives request on port 8080

3. config.json handler = Router
   Router scans routes/:
   → 02-sso.json condition "${find(request.uri.path, '^/sso')}" → MATCH

4. Route heap: AmService-1 initialized (connection to AM)

5. Chain processes filters IN ORDER:

   Filter 1: SingleSignOnFilter
   → Checks for iPlanetDirectoryPro cookie
   → NOT FOUND → returns 302 redirect to:
     http://pingam:8080/am/XUI/#login/&goto=http://pinggw:8080/sso/dashboard

6. Browser follows redirect → AM login page
   User enters demo/password → AM creates session → sets cookie
   AM redirects back to: http://pinggw:8080/sso/dashboard

7. Browser retries GET /sso/dashboard (now with cookie)

8. Filter 1: SingleSignOnFilter
   → Cookie FOUND → calls AM: POST /am/json/sessions?_action=validate
   → AM returns: { "valid": true, "uid": "demo", "cn": "Demo User" }
   → Populates contexts.ssoToken

9. Filter 2: HeaderFilter
   → Reads contexts.ssoToken.info.uid → "demo"
   → Adds header: X-IG-User: demo
   → Adds header: X-IG-Session: AQIC5w...

10. Handler: ReverseProxyHandler
    → Forwards to http://sample-app:8081/sso/dashboard
    → Backend reads X-IG-User header, knows it's "demo"
    → Returns HTML response

11. Response flows back through filters (no modification in this case)

12. Browser receives 200 with dashboard HTML
```

---

## Quick Reference: Filter Cheat Sheet

| Filter | Input | Output | Typical Position |
|---|---|---|---|
| SingleSignOnFilter | Cookie | contexts.ssoToken | First (authenticates) |
| OAuth2ResourceServerFilter | Bearer header | contexts.oauth2 | First (authenticates) |
| PolicyEnforcementFilter | ssoToken/oauth2 | Allow/Deny | After auth filter |
| HeaderFilter | contexts.* | Modified headers | After auth, before handler |
| UriPathRewriteFilter | request.uri | Modified path | Before handler |
| ScriptableFilter | Anything | Anything | Anywhere |
| ThrottlingFilter | client/token info | 429 or pass | Early (before heavy processing) |
| ConditionalFilter | Condition | Delegates | Wraps another filter |
| PasswordReplayFilter | Session data | Form submission | After SSO filter |

## Quick Reference: Handler Cheat Sheet

| Handler | Does What | When to Use |
|---|---|---|
| ReverseProxyHandler | Proxies to backend | Default — 90% of routes |
| StaticResponseHandler | Returns hardcoded response | Health checks, mocks, debugging |
| Chain | Wraps filters + handler | Whenever you need filters |
| DispatchHandler | Conditional routing | Read/write split, method-based routing |
| Router | Reads route files | Only in config.json |
| ScriptableHandler | Custom Groovy response | Complex logic, aggregation |
| ForgeRockClientHandler | HTTP client | Inside filters (not terminal) |
