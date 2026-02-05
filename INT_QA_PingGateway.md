# Interview Q&A: PingGateway (Identity Gateway)

## Q1: What is PingGateway and what problem does it solve?

**A:** PingGateway (formerly ForgeRock Identity Gateway / IG / OpenIG) is a **standalone reverse proxy** that sits between clients and backend applications, providing identity-aware request handling. It solves:

- **Protecting legacy apps** that can't be modified to add authentication/authorization
- **Centralizing security enforcement** — instead of modifying every application, PingGateway intercepts all traffic
- **API protection** — validates OAuth2 tokens before requests reach APIs
- **Migration path from agents** — replaces Web/Java agents with a more flexible, application-agnostic solution

PingGateway is a **Java application** (runs on embedded Jetty), not a WAR file. It reads **JSON route files** from a config directory and auto-reloads them without restart.

---

## Q2: Why choose PingGateway over Web Agents or Java Agents?

**A:** Key advantages:

| Aspect | Web/Java Agents | PingGateway |
|--------|----------------|-------------|
| **Installation** | Native module in web server (Apache, IIS, Tomcat) | Standalone proxy, no app modification |
| **Language** | C/C++ (Web), Java servlet filter (Java) | Java standalone |
| **App coupling** | Tightly coupled to web server version | Decoupled — works with any backend |
| **Protocol support** | HTTP only | HTTP, WebSocket, gRPC |
| **OAuth2/OIDC** | Limited (session-based SSO focus) | Full OAuth2 RS, token exchange, OIDC RP |
| **Legacy apps** | Require web server plugin support | Works with anything that speaks HTTP |
| **Microservices** | One agent per service | One gateway for many services |
| **Maintenance** | Upgrade with web server | Independent upgrade cycle |

**When to use agents**: Simple SSO for a few apps on supported web servers.
**When to use PingGateway**: Modern architectures, API protection, legacy apps, microservices, or when you can't install agents.

---

## Q3: How does PingGateway's route-based configuration work?

**A:** PingGateway uses **JSON route files** in the `config/routes/` directory:

```json
{
  "name": "my-route",
  "condition": "${find(request.uri.path, '^/api')}",
  "handler": {
    "type": "ReverseProxyHandler",
    "baseURI": "http://backend:8080"
  }
}
```

Key concepts:
- **Condition**: Expression that determines if this route handles the request (path matching, headers, etc.)
- **Handler**: What to do with the request (proxy, static response, chain of filters)
- **Auto-reload**: Routes are scanned every 10 seconds by default — no restart needed
- **Ordering**: Routes are evaluated by filename (alphabetical) — use numeric prefixes (`01-`, `02-`)
- **Heap objects**: Shared objects (like AmService) defined once and referenced across routes

This is fundamentally different from agents, where all config is centralized in AM. PingGateway config lives **with the gateway**, enabling GitOps and infrastructure-as-code patterns.

---

## Q4: What is the AmService heap object?

**A:** AmService is the **connection configuration** between PingGateway and PingAM. It contains:

```json
{
  "name": "AmService-1",
  "type": "AmService",
  "config": {
    "url": "http://pingam:8080/am",
    "realm": "/techcorp",
    "agent": {
      "username": "ig_agent",
      "password": "changeit"
    },
    "sessionProperties": ["uid", "cn"]
  }
}
```

- **url**: AM's base URL (internal network URL, not browser-facing)
- **realm**: Which AM realm to authenticate against
- **agent**: IG agent credentials (registered in AM as an Identity Gateway agent)
- **sessionProperties**: Which AM session attributes to fetch

AmService is used by filters like `SingleSignOnFilter`, `OAuth2ResourceServerFilter`, and `PolicyEnforcementFilter`. It handles agent session management, caching, and connection pooling.

---

## Q5: How does PingGateway's SSO flow work (SingleSignOnFilter)?

**A:** The `SingleSignOnFilter` is the **core SSO mechanism** — equivalent to what a Web Agent does:

1. Client sends request to PingGateway
2. SingleSignOnFilter checks for AM session cookie (`iPlanetDirectoryPro`)
3. **If no cookie**: Redirects client to AM login page (with `goto` parameter for return URL)
4. User authenticates in AM, gets session cookie
5. AM redirects back to PingGateway with the cookie
6. **If cookie present**: PingGateway validates the session against AM (REST call)
7. If valid, populates `contexts.ssoToken` with session info (uid, groups, etc.)
8. HeaderFilter injects user identity into backend request headers
9. Backend app receives `X-IG-User: demo` — knows who the user is without any SSO code

```
Client → PingGW → [has cookie?] → No → Redirect to AM login
                                 → Yes → Validate with AM → Proxy to backend
```

**Interview key point**: This is the **primary use case for PingGateway replacing Web Agents**. The backend app needs zero modification — PingGateway handles all authentication externally.

---

## Q6: How does PingGateway protect APIs with OAuth2 (OAuth2ResourceServerFilter)?

**A:** For API (non-browser) traffic, PingGateway acts as an **OAuth2 Resource Server**:

```json
{
  "type": "OAuth2ResourceServerFilter",
  "config": {
    "scopes": ["profile"],
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

Flow:
1. API client sends request with `Authorization: Bearer <token>` header
2. `OAuth2ResourceServerFilter` extracts the bearer token
3. `TokenIntrospectionAccessTokenResolver` calls AM's `/oauth2/introspect` endpoint
4. AM validates the token, returns `active: true/false`, scopes, subject, etc.
5. If valid and scopes match, request proceeds. Otherwise → `401 Unauthorized`
6. Token info is available in `contexts.oauth2.accessToken` for header injection

**Token resolution options**:
- **TokenIntrospectionAccessTokenResolver**: Calls AM introspection (works with opaque tokens)
- **OpenIdConnectTokenResolver**: Validates JWT locally using JWKS (no AM call per request — better performance)
- **StatelessAccessTokenResolver**: Validates stateless tokens locally

**Interview key point**: APIs don't validate tokens themselves — PingGateway does it as a sidecar/gateway. This is the **modern replacement for agents in API architectures**.

---

## Q7: How do you register a PingGateway (IG) agent in PingAM?

**A:** Same concept as Web/Java agent registration:

1. **AM Console** → Realms → (select realm) → Applications → Agents → **Identity Gateway**
2. Create new agent: agent ID (e.g., `ig_agent`) and password (e.g., `changeit`)
3. The agent profile is stored in AM's DS (am-config store)

The agent credentials are used by PingGateway's `AmService` to:
- Authenticate to AM's REST API
- Validate session cookies
- Perform token introspection
- Evaluate policies

**Agent types in AM**:
- **Web Agent**: For Apache, Nginx, IIS
- **Java Agent**: For Tomcat, JBoss
- **Identity Gateway**: For PingGateway (IG)

All three types serve the same PEP role but integrate differently.

---

## Q8: What is PolicyEnforcementFilter and how does it work?

**A:** `PolicyEnforcementFilter` (PEF) is PingGateway's **policy enforcement point** — it calls AM's policy engine to authorize requests:

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

Flow:
1. User is authenticated (via SingleSignOnFilter or OAuth2ResourceServerFilter)
2. PEF sends policy evaluation request to AM: resource URL + subject token + environment
3. AM's PDP evaluates matching policies (Resource Type → Policy Set → Policy)
4. Returns allow/deny per action (GET, POST, PUT, DELETE)
5. PEF enforces the decision — blocks denied requests with 403

**Interview key point**: This completes the PDP-PEP architecture:
- AM = PDP (decides)
- PingGateway + PolicyEnforcementFilter = PEP (enforces)
- Policies are managed centrally in AM, enforced at the gateway

---

## Q9: What is PasswordReplayFilter and when would you use it?

**A:** `PasswordReplayFilter` handles **legacy apps that require form-based login**. PingGateway captures credentials from AM's authentication and replays them to the backend:

Use case: A 20-year-old Java app with its own login form. You can't modify it. Solution:
1. PingGateway intercepts the request
2. SingleSignOnFilter ensures the user is authenticated with AM
3. PasswordReplayFilter retrieves stored credentials from AM session
4. PingGateway submits the legacy app's login form on behalf of the user

This enables **SSO for legacy applications** that can't be modified. The user logs in once to AM, and PingGateway handles the credential replay transparently.

**Interview key point**: This is one of PingGateway's most valuable differentiators — no other component can do credential replay for legacy apps. Web Agents can inject headers but can't submit forms.

---

## Q10: How does Cross-Domain SSO (CDSSO) work with PingGateway?

**A:** When PingGateway and AM are on **different domains**, the AM session cookie can't be shared directly (browser same-origin policy). CDSSO solves this:

1. User accesses app via PingGateway (e.g., `apps.company.com`)
2. PingGateway detects no session, redirects to AM (e.g., `sso.company.com`)
3. User authenticates with AM
4. AM creates a **CDSSO token** (short-lived, single-use) and redirects back to PingGateway's CDSSO endpoint
5. PingGateway exchanges the CDSSO token for a full AM session
6. PingGateway sets its own session cookie on its domain

This is configured via the `CrossDomainSingleSignOnFilter` in PingGateway. It's needed whenever:
- AM and the gateway are on different domains
- You can't use a shared parent domain for cookies
- Production deployments with separate AM and app infrastructure

---

## Q11: How do you troubleshoot PingGateway?

**A:** Key debugging tools:

1. **CaptureDecorator**: Logs full request/response details including headers, body, and context
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
   Add `"capture": "all"` to any handler or filter to enable logging.

2. **Admin API** (`/openig/api/`):
   - `GET /openig/api/info` — version and status
   - `GET /openig/api/system/objects` — all heap objects
   - `GET /openig/api/system/objects/_router/routes` — loaded routes

3. **Log levels**: Configure via `config/logback.xml` — set `org.forgerock.openig` to DEBUG

4. **Common issues**:
   - Route not matching → check condition expression and route file ordering
   - 502 Bad Gateway → backend unreachable (check DNS, ports, network)
   - 401 from AM → agent credentials wrong or agent not registered
   - Cookie domain mismatch → CDSSO needed for cross-domain

---

## Q12: What is PingGateway's architecture for high availability in production?

**A:** Production patterns:

1. **Stateless design**: PingGateway itself holds no session state. All state is in AM sessions or tokens. Multiple PingGateway instances can run behind a load balancer.

2. **Horizontal scaling**: Add more PingGateway containers/pods. No sticky sessions needed (stateless).

3. **Configuration management**:
   - Route files in Git → CI/CD deploys to all instances
   - ConfigMap/Secrets in Kubernetes
   - Volume mounts for route hot-reload

4. **Secrets management**:
   - Never hardcode passwords in route files
   - Use environment variables: `"password": "${env['IG_AGENT_PASSWORD']}"`
   - Use KeyStoreSecretStore for certificates
   - Kubernetes Secrets mounted as files

5. **Monitoring**:
   - Admin API for health checks
   - Prometheus metrics endpoint
   - Structured JSON logging (logback-json)
   - Grafana dashboards (sample dashboards provided in distribution)

6. **Deployment topology**:
   ```
   LB → PingGW-1 → Backend Pool
      → PingGW-2 ↗
      → PingGW-3 ↗
   All → PingAM (session validation)
   ```

---

## Q13: How does PingGateway fit in microservices architecture?

**A:** PingGateway serves as the **API Gateway / Security Gateway** in microservices:

1. **Sidecar pattern**: One PingGateway per service (Kubernetes pod). Handles auth/authz for that service.

2. **Central gateway pattern**: One PingGateway at the edge. Routes to multiple microservices based on path. Each route can have different auth requirements.

3. **Token exchange**: PingGateway can exchange external tokens (from IdP) for internal tokens (for microservices).

4. **Service mesh integration**: PingGateway can complement Istio/Linkerd by adding identity-aware routing at the application layer.

Common microservices route patterns:
```
/api/users/*   → UserService (requires profile scope)
/api/orders/*  → OrderService (requires orders scope)
/api/admin/*   → AdminService (requires admin scope + policy check)
/api/public/*  → PublicService (no auth required)
```

**Interview key point**: PingGateway is the **identity-aware API gateway**. Generic API gateways (Kong, AWS API Gateway) handle routing and rate limiting. PingGateway adds **AM integration** — SSO, OAuth2, policy enforcement, and session management.

---

## Q14: What is the difference between PingGateway's config.json and route files?

**A:**

- **config.json**: The **main configuration file**. Defines the top-level handler (usually a Router), global heap objects, and default settings. Loaded once at startup.

- **admin.json**: Admin endpoint configuration (port, prefix). Provides the management API.

- **Route files** (`routes/*.json`): Individual request handling rules. Each route defines a condition, handler, and optionally local heap objects. Auto-reloaded without restart.

Hierarchy:
```
config.json (global)
├── Router (reads routes/*.json)
│   ├── 01-simple-proxy.json
│   ├── 02-sso.json
│   └── 03-oauth2-rs.json
└── Global heap objects (shared across routes)
```

Heap objects can be defined globally (in config.json) or locally (in a route). Local objects are only visible to that route. Global objects are shared — define AmService globally if multiple routes use the same AM connection.

---

## Q15: Compare PingGateway, Web Agents, and direct OAuth2 integration. When do you use each?

**A:** This is an **architectural decision question** — common in senior-level interviews:

| Approach | Best For | Drawbacks |
|----------|----------|-----------|
| **Web Agent** | Traditional web apps on supported servers (Apache, IIS, Tomcat). Simple SSO. | Platform-specific, upgrade coupling, limited to HTTP, no OAuth2 RS support |
| **Java Agent** | Java apps on app servers. Session-based SSO. | Java-only, servlet filter integration required, same limitations as Web Agent |
| **PingGateway** | Legacy apps (no modification), API protection, microservices, migration from agents | Additional infrastructure (reverse proxy), latency hop, operational overhead |
| **Direct OAuth2/OIDC** | Modern apps built with OAuth2/OIDC. SPA, mobile, API-to-API. | Requires app modification, each app implements OIDC client library |

**Decision framework**:
1. Can you modify the app? → Yes: Direct OAuth2/OIDC (cleanest)
2. Is it a legacy app? → PingGateway with PasswordReplayFilter
3. Simple web SSO on supported server? → Web/Java Agent (if already deployed)
4. API protection? → PingGateway with OAuth2ResourceServerFilter
5. Microservices? → PingGateway as API gateway
6. Greenfield? → Direct OIDC integration + PingGateway for legacy/API

---

## Q16: What are the key filters in PingGateway and their purposes?

**A:**

| Filter | Purpose | Use Case |
|--------|---------|----------|
| `SingleSignOnFilter` | Validates AM session cookie, redirects to login | SSO for web apps |
| `OAuth2ResourceServerFilter` | Validates OAuth2 bearer tokens | API protection |
| `PolicyEnforcementFilter` | Calls AM policy engine for authorization | Fine-grained access control |
| `OAuth2ClientFilter` | Acts as OIDC Relying Party | Web apps needing OIDC login |
| `PasswordReplayFilter` | Replays credentials to legacy login forms | Legacy app SSO |
| `CrossDomainSingleSignOnFilter` | CDSSO for cross-domain SSO | Different domains |
| `HeaderFilter` | Adds/removes/modifies HTTP headers | Passing identity to backends |
| `ScriptableFilter` | Custom Groovy/JavaScript logic | Complex routing, transformations |
| `ThrottlingFilter` | Rate limiting | API protection |
| `CaptureDecorator` | Logs request/response details | Debugging |
| `ConditionalFilter` | Applies filter based on condition | Conditional security policies |

---

## Q17: How would you answer "Tell me about PingGateway" in an interview?

**A:** "PingGateway, formerly Identity Gateway, is a standalone reverse proxy that provides identity-aware security for web applications and APIs. It sits between clients and backend applications, enforcing authentication and authorization without requiring application modifications.

The key use cases are:
1. **Replacing Web Agents** — PingGateway handles SSO via SingleSignOnFilter, validating AM sessions and injecting user identity headers into backend requests
2. **API protection** — OAuth2ResourceServerFilter validates bearer tokens via AM introspection, so APIs don't implement token validation themselves
3. **Legacy app SSO** — PasswordReplayFilter enables SSO for apps that can't be modified, by replaying credentials to their login forms

Configuration is entirely **route-based** using JSON files. Routes auto-reload every 10 seconds, making PingGateway ideal for GitOps workflows. Each route has a condition (URL path matching), filters (security logic), and a handler (backend proxy).

In production, PingGateway is stateless and horizontally scalable. It connects to PingAM via an `AmService` heap object using registered IG agent credentials. The agent registration is similar to Web/Java agents — done in AM Console under Applications → Agents → Identity Gateway.

For microservices, PingGateway serves as the identity-aware API gateway, handling token validation at the edge so individual services focus on business logic. It supports multiple token validation strategies: introspection for opaque tokens, JWKS for JWT validation, and can enforce AM policies through PolicyEnforcementFilter."

---

## Q18: What are PingGateway's secrets management options?

**A:**

1. **Inline** (development only): Passwords directly in route JSON — never in production
2. **Environment variables**: `"password": "${env['IG_AGENT_PASSWORD']}"}"`
3. **System properties**: `"password": "${system['ig.agent.password']}"`
4. **KeyStoreSecretStore**: Java keystore for certificates and keys
5. **FileSystemSecretStore**: Secrets in files (Kubernetes Secrets mounted as volumes)
6. **Base64EncodedSecretStore**: Base64-encoded secrets in config
7. **HsmSecretStore**: Hardware Security Module integration

Production pattern:
```json
{
  "type": "FileSystemSecretStore",
  "config": {
    "directory": "/var/run/secrets",
    "mappings": [{
      "secretId": "agent.password",
      "format": "PLAIN"
    }]
  }
}
```

---

## Q19: How does PingGateway handle session management?

**A:** PingGateway itself is **stateless by default** — it doesn't maintain sessions. It delegates session management to AM:

- **SingleSignOnFilter**: Validates AM's `iPlanetDirectoryPro` cookie against AM on each request
- **JwtSession**: PingGateway can create its own JWT-based sessions (stored in a cookie) for caching authentication state, reducing calls to AM
- **SessionInfoContext**: After SSO validation, session properties (uid, groups, auth level) are available in `contexts.ssoToken`

With `JwtSession`, PingGateway can cache the validation result in an encrypted JWT cookie, avoiding repeated AM calls. This is a **performance optimization** — not a replacement for AM sessions.

---

## Q20: Describe a PingGateway deployment you would design for a large enterprise.

**A:** "For a large enterprise with 50+ applications:

**Architecture**:
- 3+ PingGateway instances behind an L7 load balancer
- Separate route files per application team (GitOps — each team manages their routes)
- Central AmService config pointing to AM cluster

**Route organization**:
```
routes/
├── 01-health.json          (health check, no auth)
├── 10-legacy-erp.json      (PasswordReplayFilter for SAP/Oracle)
├── 20-api-orders.json      (OAuth2ResourceServerFilter, orders scope)
├── 20-api-users.json       (OAuth2ResourceServerFilter, profile scope)
├── 30-internal-portal.json (SingleSignOnFilter + PolicyEnforcementFilter)
└── 99-default-deny.json    (catch-all 403)
```

**Security layers**:
1. TLS termination at load balancer
2. PingGateway validates tokens/sessions
3. PolicyEnforcementFilter for URL-level authorization
4. HeaderFilter injects identity for backend apps
5. ThrottlingFilter for rate limiting

**Operational**:
- Prometheus metrics → Grafana dashboards
- CaptureDecorator in staging (disabled in prod)
- Route changes via CI/CD (merge to main → deploy to all instances)
- Secrets via Kubernetes Secrets / HashiCorp Vault
- Auto-scaling based on request rate"

---

## Q21: What changed with secrets in PingGateway 2025.11.1?

**A:** Starting with PingGateway 2025.11.1, **inline passwords are no longer accepted** in AmService and other objects. You must use a `secretsProvider` with a `passwordSecretId`:

**Old (no longer works)**:
```json
{
  "type": "AmService",
  "config": {
    "agent": {
      "username": "ig_agent",
      "password": "changeit"
    }
  }
}
```

**New (required)**:
```json
{
  "type": "AmService",
  "config": {
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
    }
  }
}
```

The error message is: `secretsProvider: Expecting a value`. PingGateway logs a warning that `Base64EncodedSecretStore` should NOT be used in production — use `FileSystemSecretStore` or `KeyStoreSecretStore` instead.

**Interview key point**: This reflects the broader industry trend of never storing secrets in plain text in config files — even in development. PingGateway enforces this at the framework level.

---

## Q22: What is UriPathRewriteFilter and when do you need it?

**A:** When PingGateway proxies requests to a backend, the **full request path is forwarded** by default. If PingGateway matches on `/sample/*` and the backend serves at `/*`, you need `UriPathRewriteFilter` to strip the prefix:

```json
{
  "type": "UriPathRewriteFilter",
  "config": {
    "mappings": {
      "/sample": "/"
    }
  }
}
```

This rewrites `/sample/home` → `/home` before sending to the backend. Without it, the backend receives `/sample/home` and returns 404.

Common use cases:
- Multi-app gateway: `/app1/*` → Service A at `/*`, `/app2/*` → Service B at `/*`
- API versioning: `/v2/api/*` → new backend at `/api/*`
- Context path differences: gateway path doesn't match backend path

---

## Q23: What is the difference between config that auto-reloads and config that requires restart?

**A:**

| Config File | Auto-Reload? | Notes |
|---|---|---|
| `config/routes/*.json` | Yes (~10 seconds) | Add, modify, delete routes live |
| `config/config.json` | No — restart required | Global handler, heap objects |
| `config/admin.json` | No — restart required | Admin endpoint settings |

**Interview key point**: Route hot-reload is one of PingGateway's operational advantages. You can deploy new routes, update conditions, add filters — all without downtime. This enables blue-green route deployments and A/B testing at the gateway level.

---

## Q24: Does PingGateway have a web UI?

**A:** No. PingGateway is a **headless infrastructure component** — similar to Nginx or HAProxy, not like PingAM which has a full admin console. Configuration is done entirely through:

1. **JSON route files** — edited directly on disk or deployed via CI/CD
2. **Admin REST API** — programmatic access at `/openig/api/` (localhost-only by default for security)

The admin API provides:
- `GET /openig/api/info` — version and build info
- `GET /openig/api/system/objects` — all loaded heap objects
- `GET /openig/api/system/objects/_router/routes` — all active routes

The admin endpoint only responds to requests from localhost inside the container (returns 403 to external requests). Access it via: `docker exec pinggw curl http://localhost:8080/openig/api/info`

Older versions had "IG Studio" — a basic web UI for building routes. It was removed in recent versions in favor of the declarative JSON approach.

---

## Q25: What is the Context chain in PingGateway? Where does context come from?

**A:** Context is a **chain of information objects** that travels with every request through the gateway. Think of it as a growing backpack — each filter can add data to it, and downstream filters/handlers can read it.

**Automatic contexts** (always present, created by the gateway itself):
- `contexts.client` — client IP, port, certificates (from TCP connection)
- `contexts.router` — which route matched, route name

**Filter-added contexts** (only present after specific filters run):
- `contexts.ssoToken` — added by `SingleSignOnFilter` after validating AM session
- `contexts.oauth2` — added by `OAuth2ResourceServerFilter` after token introspection
- `contexts.policyDecision` — added by `PolicyEnforcementFilter` after policy evaluation

**Key insight**: If you reference `${contexts.ssoToken.info.uid}` in a HeaderFilter but no SingleSignOnFilter ran before it, the value is null/empty — there's no error, just missing data. Filter ordering determines what context is available.

---

## Q26: What is the filter execution order? Does the filter run first or the handler?

**A:** **Filters ALWAYS run before the handler.** The execution model is:

```
Request arrives
  → Filter 1 (request phase)
    → Filter 2 (request phase)
      → Filter 3 (request phase)
        → Handler executes (terminal — sends to backend or generates response)
      ← Filter 3 (response phase)
    ← Filter 2 (response phase)
  ← Filter 1 (response phase)
Response sent
```

This is the **chain of responsibility pattern**. Each filter wraps around the next. Filters are bidirectional — they can act on the request (before calling next) and/or the response (after calling next).

In a ScriptableFilter, this is explicit:
```groovy
// Request phase — runs BEFORE handler
request.headers.add("X-Custom", "value")

// Call next filter or handler
return next.handle(context, request).thenOnResult { response ->
    // Response phase — runs AFTER handler
    response.headers.add("X-Response-Time", "42ms")
}
```

---

## Q27: How does a filter know whether to act on request or response?

**A:** Different filter types handle this differently:

1. **MessageType config** — Filters like `HeaderFilter` use `"messageType": "REQUEST"` or `"RESPONSE"` to specify which phase
2. **Hardcoded behavior** — `SingleSignOnFilter` always acts on request (validate session). `UriPathRewriteFilter` always rewrites the request URI
3. **Both phases built-in** — `CaptureDecorator` logs both request AND response automatically
4. **ScriptableFilter** — You control it explicitly: code before `next.handle()` = request phase, code in `.thenOnResult` callback = response phase

There's no generic "phase" flag on filters — each filter type defines its own behavior.

---

## Q28: Why does PingGateway need an IG Agent? Why not just use an OAuth2 client_credentials grant?

**A:** First principles — **AM doesn't trust unknown callers.** When PingGateway calls AM to validate a session or introspect a token, AM needs to know *who is asking* and *what they're allowed to do*.

**Why not OAuth2 client_credentials?**
OAuth2 clients get an access token with scopes — they're designed to access APIs. But PingGateway doesn't need to "access an API." It needs to:

| Operation | OAuth2 Client | IG Agent |
|---|---|---|
| Validate AM session tokens | ❌ No endpoint access | ✅ Built-in permission |
| Introspect OAuth2 tokens | ❌ Needs separate registration | ✅ Built-in permission |
| Evaluate authorization policies | ❌ No policy endpoint access | ✅ Built-in permission |
| Receive session notifications | ❌ Not supported | ✅ Built-in (logout propagation) |
| Get user profile attributes | ❌ Needs scope negotiation | ✅ Direct access via session |

An IG Agent is a **purpose-built identity** with exactly the permissions a gateway needs — session validation, token introspection, policy evaluation, and notification subscription. OAuth2 clients would require complex workarounds to achieve the same.

---

## Q29: Does the IG Agent have admin-level access like amadmin?

**A:** No. The IG Agent has **scoped, read-only security permissions** — completely different from amadmin:

| Capability | amadmin | IG Agent |
|---|---|---|
| Create/delete realms | ✅ | ❌ |
| Create/modify users | ✅ | ❌ |
| Configure authentication trees | ✅ | ❌ |
| Register OAuth2 clients | ✅ | ❌ |
| Validate sessions | ✅ | ✅ |
| Introspect tokens | ✅ | ✅ |
| Evaluate policies | ✅ | ✅ |
| Receive session notifications | ✅ | ✅ |

The agent can **read security state** but cannot **modify AM configuration**. It's the principle of least privilege — the gateway only gets what it needs.

---

## Q30: Why define AmService as a heap object? Why inside the route vs config.json?

**A:** First principles — **heap objects exist because multiple filters need the same dependency.**

If `SingleSignOnFilter`, `PolicyEnforcementFilter`, and `OAuth2ResourceServerFilter` all need an AM connection, you don't configure the URL, agent credentials, and realm three times. You define it once as a heap object and reference it by name.

**Where to define it:**

| Location | Scope | Use When |
|---|---|---|
| `config.json` heap | Global — all routes share it | Production: most routes use the same AM instance |
| Route-level heap | Local — only this route uses it | Lab/testing: isolate configuration per route |

**Real-world pattern**: In production, AmService goes in `config.json` because 20+ routes all connect to the same AM. In our lab, it's in the route because we want each route to be self-contained for learning.

---

## Q31: Is the `config` property optional in handlers and filters?

**A:** Yes. Every handler and filter has **Java-defined defaults**. If you omit `config`, you get those defaults.

Example — these are equivalent:
```json
// Explicit config
{ "type": "ReverseProxyHandler", "config": { "connectionTimeout": "10 seconds", "soTimeout": "10 seconds" }, "baseURI": "http://backend:8080" }

// Defaults applied automatically
{ "type": "ReverseProxyHandler", "baseURI": "http://backend:8080" }
```

Defaults are defined in the Java source code of each class — they're not in any config file. The official docs list default values for each property.

---

## Q32: What is the security benefit of PingGateway as a reverse proxy (even without AM filters)?

**A:** The core principle: **the backend is never directly exposed to the internet.**

```
Internet → PingGateway (:8083) → Backend (:8081, internal only)
```

Even without authentication filters, this gives you:
- **Attack surface reduction** — backend port not exposed, only gateway is internet-facing
- **URL control** — gateway decides which paths are accessible
- **Centralized logging** — all traffic flows through one point
- **Header sanitization** — gateway can strip/add headers before forwarding
- **Rate limiting** — ThrottlingFilter at the gateway, no backend changes needed
- **TLS termination** — one certificate on the gateway, HTTP internally

---

## Q33: What are the top-level attributes of a PingGateway route?

**A:** A route JSON file supports these top-level attributes:

| Attribute | Required | Purpose |
|---|---|---|
| `name` | No (defaults to filename) | Human-readable route name |
| `comment` | No | Documentation — not used at runtime |
| `condition` | No (matches everything) | Expression that determines if this route handles the request |
| `handler` | **Yes** | The handler (or Chain) that processes the request |
| `heap` | No | Route-local shared objects (AmService, etc.) |
| `secrets` | No | Secret stores for this route |
| `session` | No | Session implementation (default: JwtSession) |
| `auditService` | No | Audit logging configuration |
| `capture` | No | Debug capture settings |

Only `handler` is truly required — everything else has sensible defaults.

---

## Q34: What is the `loginEndpoint` property in SingleSignOnFilter?

**A:** By default, `SingleSignOnFilter` constructs the AM login redirect URL automatically from the `AmService` URL. The `loginEndpoint` property lets you **override** this redirect URL — critical when:

- The internal AM URL (container-to-container) differs from the browser-facing URL
- You need custom query parameters (realm, service/tree, locale)
- The default redirect doesn't work in your network topology

Example:
```json
{
  "type": "SingleSignOnFilter",
  "config": {
    "amService": "AmService-1",
    "loginEndpoint": "http://pingam:8081/am/?goto=${urlEncodeQueryParameterNameOrValue(contexts.router.originalUri)}%3F_ig%3Dtrue&realm=/techcorp"
  }
}
```

**Critical detail**: The `_ig=true` parameter must be appended to the goto URL (URL-encoded as `%3F_ig%3Dtrue`). This is how SingleSignOnFilter recognizes that the user is returning from AM login. Without it, PingGateway enters an **infinite redirect loop** — it can't distinguish a post-login callback from a fresh unauthenticated request.

---

## Q35: What is the `_ig=true` callback marker in PingGateway SSO?

**A:** When SingleSignOnFilter redirects to AM login, it appends `?_ig=true` to the goto URL. After AM authentication, AM redirects the browser back to PingGateway with `?_ig=true` in the URL. PingGateway uses this marker to know:

1. **This is a post-login callback** — check for the AM session cookie now
2. **Strip the marker** — forward the clean URL to the backend

Without `_ig=true`:
```
User → PingGW (no cookie) → redirect to AM → login → AM redirect to PingGW → PingGW (no cookie??) → redirect to AM → infinite loop
```

With `_ig=true`:
```
User → PingGW (no cookie) → redirect to AM → login → AM redirect to PingGW/?_ig=true → PingGW recognizes callback, reads cookie → forward to backend
```

**Interview insight**: This is a subtle but important detail about PingGateway's SSO implementation. If you manually configure `loginEndpoint`, you must include `_ig=true` in the goto parameter or SSO breaks silently.

---

## Q36: What happens when AM's cookie domain doesn't match PingGateway's domain?

**A:** This is a **common production issue**. AM sets the `iPlanetDirectoryPro` session cookie with a specific domain:

- If AM's cookie domain list is empty (`"domains":[]`), the cookie is set on the **exact hostname** of the requesting URL
- If set to `.example.com`, the cookie is shared across all `*.example.com` subdomains

**Problem scenario**:
- User logs in at `sso.company.com` (AM) → cookie set on `sso.company.com`
- PingGateway at `apps.company.com` → browser doesn't send the cookie (different hostname)
- PingGateway sees no cookie → redirects to AM again → infinite loop

**Solutions**:
1. **Shared parent domain**: Set cookie domain to `.company.com` — works for `sso.company.com` and `apps.company.com`
2. **CDSSO**: Use `CrossDomainSingleSignOnFilter` — AM creates a one-time token, PingGateway exchanges it for a session
3. **Same hostname**: Put AM and PingGateway behind the same hostname with path-based routing

**Interview key point**: Cookie domain configuration is one of the most common SSO debugging issues. Always check: What domain is the cookie set on? Does the gateway's domain match?

---

## Q37: What is the AM FQDN validation and how does it affect PingGateway?

**A:** AM validates the **Fully Qualified Domain Name** (FQDN) of incoming requests against a list of recognized FQDNs. If the hostname in the request URL doesn't match, AM rejects it with `FQDN "xxx" is not valid`.

**Impact on PingGateway**:
- AM is configured with FQDN `pingam` (or `am.example.com`)
- If PingGateway's `loginEndpoint` redirects the browser to `localhost:8081/am`, AM rejects it
- Must redirect to the AM-recognized FQDN: `pingam:8081/am`

**Where FQDN is configured**:
- AM Console → Configure → Server Defaults → General → Recognized FQDNs
- Or via Amster/REST API

This matters because PingGateway has **two network paths** to AM:
1. **Internal** (container-to-container): `http://pingam:8080/am` — used by AmService for API calls
2. **Browser-facing**: `http://pingam:8081/am` — used by loginEndpoint for redirects

The internal URL uses the Docker network. The browser URL must resolve on the user's machine (via `/etc/hosts` or DNS).

---

## Q38: Why might AM show the user profile page instead of redirecting to the goto URL?

**A:** Several possible causes (all debugged hands-on):

1. **Default Success Login URL override**: AM's Post Authentication Processing can set a default URL (e.g., `/am/console`) that overrides the goto parameter. Check: Realm → Authentication → Settings → Post Authentication Processing.

2. **AM XUI hash-based routing**: AM's XUI SPA uses `#` fragments for internal routing. The `goto` parameter may be in the query string (`?goto=URL`) but the XUI reads its routing from the hash fragment (`#login/`, `#profile/details`). After login, XUI navigates to `#profile/details` ignoring the goto.

3. **goto URL validation**: AM validates the goto URL against a list of allowed domains (Validation Service). If the goto URL doesn't match, AM silently ignores it and shows the default page.

4. **Cookie domain mismatch**: If the goto redirect goes to a different domain than where the cookie was set, the target won't have the session cookie.

**Debugging steps**:
1. Check browser Network tab — look at the POST to `/authenticate` and its response
2. Check if `successUrl` in the authenticate response matches your goto
3. Check AM's Validation Service for allowed goto URLs
4. Check Default Success Login URL in Post Authentication Processing
5. Try direct AM login with `?goto=URL` (no PingGateway) to isolate the issue

---

## Q39: What are the two network paths PingGateway uses to communicate with AM?

**A:** This is a critical architectural concept:

```
┌─────────────────────────────────────────────────┐
│ Docker Network (fr-net)                         │
│                                                 │
│  PingGW ──(1) API calls──→ pingam:8080/am      │
│                                                 │
└─────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────┐
│ Host Machine / Browser                          │
│                                                 │
│  Browser ──(2) Redirects──→ pingam:8081/am      │
│                                                 │
└─────────────────────────────────────────────────┘
```

**Path 1 — Backend API calls** (AmService `url`):
- Container-to-container: `http://pingam:8080/am`
- Used for: session validation, token introspection, policy evaluation
- Never seen by the browser

**Path 2 — Browser redirects** (SingleSignOnFilter `loginEndpoint`):
- Host-mapped port: `http://pingam:8081/am`
- Used for: redirecting user to AM login page
- Must resolve on the user's machine (DNS or `/etc/hosts`)

**Common mistake**: Using the internal URL for browser redirects → browser can't resolve `pingam:8080`. Or using the external URL for API calls → PingGateway can't reach `localhost:8081` from inside the container.

---

## Q40: What happens if PingGateway routes fail to load at startup?

**A:** If a route references an external dependency (like an AmService with an IG agent) that doesn't exist yet, the route **fails to build** and is skipped. Other routes continue to load normally.

**Example from our lab**:
- Routes 02-sso and 03-oauth2-rs reference `ig_agent` in AM
- At startup, `ig_agent` wasn't registered in AM yet
- These routes failed with authentication errors during initialization
- Routes 01 and 04 (no AM dependency) loaded fine

**Fix**: Register the dependency (ig_agent), then restart PingGateway. On restart, the routes rebuild successfully.

**Important**: Route auto-reload (every 10 seconds) does NOT retry failed routes. Failed routes stay failed until PingGateway restarts or the route file is modified (triggering a rebuild).

**Interview key point**: This is why route dependencies should be provisioned before deploying PingGateway in production. In CI/CD: register agents first, then deploy gateway.

---

## Q41: What is the Cross-Domain SSO (CDSSO) problem? Why can't regular SSO work across domains?

**A:** Standard SSO uses AM's session cookie (`iPlanetDirectoryPro`). Browsers enforce the **same-origin policy** on cookies:

```
Cookie set on: sso.company.com
Available to:  sso.company.com        ✅
Available to:  apps.company.com       ❌ (different hostname)
Available to:  partner.othercorp.com  ❌ (different domain entirely)
```

If AM is at `sso.company.com` and your app is at `apps.othercorp.com`, the AM session cookie **never reaches the app**. The browser simply doesn't send it. SSO breaks.

**Simple fix (not CDSSO)**: If both AM and apps share a parent domain (e.g., `sso.company.com` and `apps.company.com`), set AM's cookie domain to `.company.com`. The cookie is shared across all subdomains. This is regular SSO with a broad cookie domain.

**CDSSO is needed when**: Domains are completely different (`company.com` vs `partner.com`) — no shared parent domain exists.

---

## Q42: How does CDSSO work? Walk through the flow.

**A:** CDSSO uses a **redirect-based token exchange** to bridge the domain gap:

```
Step 1: User hits App (apps.partner.com)
        → No AM cookie → Redirect to AM

Step 2: AM (sso.company.com) authenticates user
        → Creates one-time CDSSO token (short-lived JWT called LARES)
        → Redirects back to App with token in URL

Step 3: App/Gateway receives token
        → Validates token with AM (back-channel REST call)
        → Creates its own session (local cookie on partner.com)
        → User is authenticated
```

**Detailed flow with PingGateway**:

```
                       ┌──────────────────┐
User ─── https://app.partnerco.com ──────>│  PingGateway      │
                                          │  (partnerco.com)  │
                                          └────────┬──────────┘
                                                   │
                          No session cookie? ───────┘
                                                   │
                          Redirect to: ─────────────┘
                          https://sso.techcorp.com/am/CDCServlet
                          ?goto=https://app.partnerco.com/cdsso/callback
                                                   │
                       ┌──────────────────┐        │
                       │  PingAM          │<───────┘
                       │  (techcorp.com)  │
                       │  Authenticates   │
                       │  Creates CDSSO   │
                       │  token (LARES)   │
                       └────────┬─────────┘
                                │
                          Redirect back to:
                          https://app.partnerco.com/cdsso/callback
                          ?LARES=<jwt-token>
                                │
                       ┌────────v─────────┐
                       │  PingGateway     │
                       │  Validates LARES │
                       │  Sets local      │
                       │  session cookie  │
                       └──────────────────┘
```

Key components:
- **CDCServlet**: AM's cross-domain entry point (`/am/cdcservlet`) — different from the normal login page
- **LARES token**: Liberty Alliance Resource — a short-lived, signed JWT containing the session reference
- **Back-channel validation**: PingGateway validates the LARES token directly with AM (server-to-server), not via the browser
- **Local session cookie**: PingGateway creates its own cookie on the app's domain — the browser can now send it on subsequent requests

---

## Q43: How do you configure CDSSO with PingGateway?

**A:** Use `CrossDomainSingleSignOnFilter` instead of `SingleSignOnFilter`:

```json
{
  "type": "CrossDomainSingleSignOnFilter",
  "config": {
    "amService": "AmService-1",
    "authCookie": {
      "name": "ig-session",
      "domain": ".partnerco.com"
    }
  }
}
```

Key differences from regular SingleSignOnFilter:

| Aspect | SingleSignOnFilter | CrossDomainSingleSignOnFilter |
|---|---|---|
| **AM endpoint** | Standard login page | CDCServlet |
| **Token type** | AM session cookie | LARES JWT in URL |
| **Cookie** | AM's `iPlanetDirectoryPro` | Gateway's own cookie (`ig-session`) |
| **Domain requirement** | Same domain as AM | Any domain |
| **Validation** | Cookie validated with AM each request | LARES validated once, local session cached |

**AM-side configuration for CDSSO**:
1. **Configure → Global Services → Platform → Cookie Domains**: Add AM's domain (`.techcorp.com`)
2. **Realms → Authentication → Settings → Core**: Enable "Organization Authentication Signing Secret" (signs the CDSSO token)
3. **CDCServlet URL**: `https://sso.techcorp.com/am/cdcservlet` — the cross-domain entry point

---

## Q44: Give a practical example of CDSSO across three domains.

**A:**

**Scenario**: TechCorp acquires PartnerCo and AcmeCorp. All three have existing apps on different domains.

```
Domain 1: sso.techcorp.com       (AM — identity provider)
Domain 2: apps.techcorp.com      (internal apps — same parent domain)
Domain 3: portal.partnerco.com   (acquired company — different domain)
Domain 4: hr.acmecorp.com        (acquired company — different domain)
```

**PingGateway route strategy**:
- **Domain 2** → `SingleSignOnFilter` (shared `.techcorp.com` cookie — regular SSO)
- **Domains 3 and 4** → `CrossDomainSingleSignOnFilter` (CDSSO token exchange)

**Route for Domain 2 (regular SSO)**:
```json
{
  "name": "internal-apps",
  "condition": "${find(request.uri.host, 'apps.techcorp.com')}",
  "handler": {
    "type": "Chain",
    "config": {
      "filters": [
        { "type": "SingleSignOnFilter", "config": { "amService": "AmService-1" } }
      ],
      "handler": { "type": "ReverseProxyHandler", "baseURI": "http://internal-app:8080" }
    }
  }
}
```

**Route for Domain 3 (CDSSO)**:
```json
{
  "name": "partner-portal",
  "condition": "${find(request.uri.host, 'portal.partnerco.com')}",
  "handler": {
    "type": "Chain",
    "config": {
      "filters": [
        {
          "type": "CrossDomainSingleSignOnFilter",
          "config": {
            "amService": "AmService-1",
            "authCookie": { "name": "ig-session", "domain": ".partnerco.com" }
          }
        }
      ],
      "handler": { "type": "ReverseProxyHandler", "baseURI": "http://partner-app:8080" }
    }
  }
}
```

**Real-world production patterns**:
- **Enterprise merger**: Company A acquires Company B. Both have apps on different domains. CDSSO bridges the gap without migrating domains.
- **Partner federation**: SaaS vendor provides SSO to customer-hosted apps. Vendor runs AM, customers run apps on their own domains.
- **Multi-region**: Global company with `company.co.uk`, `company.co.jp`, `company.com`. AM on `company.com`, CDSSO provides SSO to regional domains.

---

## Q45: CDSSO vs SAML vs OIDC — when do you use each for cross-domain SSO?

**A:** All three solve cross-domain authentication, but with different trade-offs:

| Feature | CDSSO | SAML | OIDC |
|---------|-------|------|------|
| **Protocol** | AM-proprietary | Standard | Standard |
| **Token** | LARES JWT | SAML Assertion XML | ID Token JWT |
| **Use case** | AM-to-AM or AM-to-Gateway | Cross-org federation | Modern apps, APIs |
| **Setup complexity** | AM config only | Metadata exchange, CoT | Client registration |
| **App modification** | None (gateway handles it) | SP library needed | OIDC library needed |
| **Interoperability** | ForgeRock/Ping only | Any SAML IdP/SP | Any OIDC provider |
| **API support** | No (browser-based) | No (browser-based) | Yes (bearer tokens) |
| **Standards body** | None | OASIS | OpenID Foundation |

**Decision framework**:

1. **Both sides are PingAM/PingGateway?** → CDSSO (simplest, no app changes)
2. **Other side is a different vendor (Okta, Azure AD, etc.)?** → SAML or OIDC (standards-based interoperability)
3. **Partner wants formal federation with metadata exchange?** → SAML (established enterprise standard)
4. **Modern app or SPA that can use a library?** → OIDC (cleanest, supports APIs too)
5. **Legacy app you can't modify?** → CDSSO + PingGateway (zero app changes)
6. **API-to-API authentication needed?** → OIDC/OAuth2 (CDSSO and SAML are browser-only)

**Interview key point**: CDSSO is the **path of least resistance** for cross-domain SSO within a ForgeRock/Ping deployment. It requires zero application changes — PingGateway handles everything. But it's proprietary — if you need interoperability with non-Ping systems, use SAML or OIDC.

---

## Q46: How would you explain CDSSO in an interview?

**A:** "Cross-Domain Single Sign-On solves the cookie boundary problem. When AM and applications are on different domains, the AM session cookie can't cross the domain boundary due to browser same-origin policy.

CDSSO uses a redirect-based token exchange: the user is redirected to AM's CDCServlet, authenticates, and AM creates a short-lived LARES token. AM redirects the user back to the application with this token in the URL. The application — typically PingGateway with a CrossDomainSingleSignOnFilter — validates the token back-channel with AM and creates its own local session cookie on the application's domain.

The key advantage is zero application modification. PingGateway handles the entire CDSSO flow. The backend application just receives authenticated requests with identity headers, same as regular SSO.

In production, I've seen CDSSO used for enterprise mergers where two companies with different domains need unified SSO, and for multi-region deployments where regional domains need SSO from a central AM instance. For cross-vendor federation — like integrating with Okta or Azure AD — I'd recommend SAML or OIDC instead, since CDSSO is specific to the ForgeRock/Ping ecosystem."
