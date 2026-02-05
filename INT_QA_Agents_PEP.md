# Interview Q&A: Web Agents, Java Agents & Policy Enforcement Points

*Covers agent types, how they enforce AM policies, PingGateway comparison, and production architecture*

---

## Q1: What problem do agents solve?

**A:** AM authenticates users and evaluates policies, but it never intercepts application traffic. Without a Policy Enforcement Point (PEP), applications are unprotected — anyone can bypass AM and hit the app directly.

Agents sit **in front of** (or inside) the application and intercept every request. They check with AM: "Does this user have a valid session? Are they authorized for this resource?" If yes, forward the request. If no, redirect to AM login or return 403.

```
Without agent:
User ──────────────────────────> Application (unprotected)

With agent:
User ────> Agent ──check──> AM ──allow/deny──> Agent ──forward/block──> Application
```

---

## Q2: What is a Web Agent?

**A:** A Web Agent is a **native C/C++ plugin installed into the web server** — Apache (`mod_`), Nginx (module), or IIS (ISAPI filter). It runs in-process with the web server.

**How it works:**
1. User hits `https://app.example.com/protected/resource`
2. Web Agent intercepts the request before it reaches the application
3. Agent checks for `iPlanetDirectoryPro` cookie (AM session)
4. If no cookie → redirect to AM login page
5. If cookie exists → Agent calls AM REST API to validate session and evaluate policies
6. AM returns allow/deny based on policies (e.g., AllowEmployeeRead)
7. Agent forwards request (with identity headers) or returns 403

**Key characteristics:**
- Written in C/C++ — native to the web server platform
- Runs in-process (fast, no extra network hop)
- Protects any application behind that web server — app doesn't need to know about AM
- Injects AM session attributes as HTTP headers (e.g., `X-AM-Username: demo`) so the app knows who the user is

**Use case:** Existing PHP, Python, or static site behind Apache. Install the Web Agent — instant SSO and authorization without touching application code.

---

## Q3: What is a Java Agent?

**A:** A Java Agent is a **servlet filter installed into a Java application server** — Tomcat, JBoss/WildFly, WebLogic, WebSphere, Jetty.

**How it works:** Same flow as Web Agent, but:
1. It's a Java servlet filter in the app server (configured in `web.xml` or as a global filter)
2. Intercepts requests at the Java EE container level before they reach servlets/controllers
3. Calls AM REST API for session validation and policy evaluation
4. Can set `HttpServletRequest` attributes and populate Java security context

**Key characteristics:**
- Written in Java — runs in the JVM alongside the application
- Deployed as a JAR/filter in the app server
- Can access Java EE security context (`request.getRemoteUser()`, `request.isUserInRole()`)
- Works with Java-specific frameworks (Spring Security, J2EE container security)

**Use case:** Java web application on Tomcat. Deploy the Java Agent filter — handles session validation and populates the standard Java security context without rewriting auth logic.

---

## Q4: Web Agent vs Java Agent — when to use which?

**A:**

| Aspect | Web Agent | Java Agent |
|---|---|---|
| **Where it runs** | Web server (Apache, Nginx, IIS) | Java app server (Tomcat, JBoss) |
| **Language** | C/C++ native module | Java servlet filter |
| **Protects** | Any app behind the web server | Java web applications |
| **Installation** | Web server plugin config | Servlet filter in web.xml |
| **App code changes** | None | None (or minimal) |
| **Performance** | Fastest (native C, in-process) | Fast (JVM, in-process) |
| **User identity to app** | HTTP headers (`X-AM-*`) | HTTP headers + Java security context |
| **Typical architecture** | Apache/Nginx reverse proxy → backend apps | Direct Java app protection |

**Decision rule:**
- App behind Apache/Nginx/IIS → **Web Agent**
- Java WAR/EAR on Tomcat/JBoss → **Java Agent**
- Microservices / APIs / cloud-native → **PingGateway** instead

---

## Q5: How do agents relate to AM's policy evaluation?

**A:** Agents are the missing piece that makes policies actually enforce anything. In AM's architecture:

- **PAP (Policy Administration Point)** = AM Console (where you create policies)
- **PDP (Policy Decision Point)** = AM (evaluates policies via REST API)
- **PEP (Policy Enforcement Point)** = Agent / PingGateway (intercepts traffic, enforces decisions)

Without a PEP, AM's policies are just data sitting in DS. The agent is what turns policy decisions into actual access control.

**Example using our lab config:**

When `demo` (employee) hits `http://api.techcorp.com/data/reports`:
1. Agent intercepts → validates demo's session → calls `POST /policies?_action=evaluate`
2. AM evaluates against `AllowEmployeeRead` policy → `GET=Allow`
3. Agent forwards request with headers: `X-AM-Username: demo`, `X-AM-Groups: employees`
4. API receives the request knowing the user is authorized

When `bob` (contractor) hits the same URL:
1. Agent intercepts → validates bob's session → calls policy evaluation
2. AM evaluates → no matching policy → implicit deny
3. Agent returns 403 — request never reaches the API

---

## Q6: What is PingGateway (Identity Gateway) and how does it compare to agents?

**A:** PingGateway (formerly IG / Identity Gateway) is a **standalone reverse proxy** with AM integration. Unlike agents (installed into the app server), PingGateway runs as its own service.

```
Web/Java Agent:                    PingGateway:
┌─────────────────────┐           ┌──────────┐     ┌──────────┐
│ Apache / Tomcat     │           │ PingGW   │────>│ App      │
│ ┌─────────────────┐ │           │ (proxy)  │     │ (any)    │
│ │ Agent (inside)  │ │           └────┬─────┘     └──────────┘
│ └─────────────────┘ │                │
│ App (same server)   │                AM
└─────────────────────┘
```

| Aspect | Web/Java Agent | PingGateway (IG) |
|---|---|---|
| **Deployment** | Installed INTO the web/app server | Separate reverse proxy server |
| **Coupling** | Tightly coupled to server version | Loosely coupled, independent |
| **Upgrade** | Must match web server version | Independent upgrade cycle |
| **Config** | Agent config files on the server | Route-based config, scriptable (Groovy) |
| **Modern APIs** | Basic URL pattern matching | Rich request/response transformation, token exchange |
| **Best for** | Traditional apps, simple SSO | Microservices, APIs, legacy app integration |

**The evolution:**
- 2010s: Web Agents everywhere (Apache + mod_agent was the standard)
- Late 2010s: Java Agents for Java shops
- 2020s: PingGateway replacing both — standalone proxy, no installation into app servers

---

## Q7: How does an agent get registered with AM?

**A:** Agents are registered as **agent identities** in AM. In the Console:

**Realms → [realm] → Applications → Agents → Web/Java**

Each agent gets:
- **Agent name** — unique identifier
- **Agent password** — shared secret for agent-to-AM communication
- **Agent profile** — configuration stored in AM (which URLs to protect, which to exclude, header mappings, etc.)

The agent on the web/app server uses its name + password to authenticate to AM and retrieve its profile. This is why you see `AgentService.json` and `AgentDataStoreDecision/` in Amster exports — agent profiles are AM config.

**Two profile modes:**
- **Centralized** — agent config stored in AM (agent fetches it at startup). Easy to manage centrally.
- **Local** — agent config stored in a local file on the web server. No dependency on AM for config, but harder to manage across many agents.

---

## Q8: What HTTP headers do agents set for the application?

**A:** After validating the session and policy, agents inject identity information as HTTP headers so the application knows who the user is:

| Header | Value | Purpose |
|---|---|---|
| `X-AM-Username` | `demo` | Authenticated user's ID |
| `X-AM-Groups` | `employees,techcorp-users` | User's group memberships |
| `X-AM-Auth-Level` | `0` | Authentication level achieved |
| `X-AM-Realm` | `/techcorp` | Realm the user authenticated to |
| Custom session properties | Any value set by auth tree nodes | E.g., `X-AM-Risk-Level: low` (from your RiskLevelRouterNode) |

The application reads these headers instead of implementing its own authentication. This is how SSO works without modifying app code — the agent handles auth, the app trusts the headers.

**Security concern:** These headers must only come from the agent, never from the end user. The agent strips any incoming `X-AM-*` headers from the client request before adding its own. In production, ensure the app only accepts requests from the agent (network-level restriction or shared secret).

---

## Q9: How do agents handle SSO across multiple applications?

**A:** All agents share the same AM session cookie (`iPlanetDirectoryPro`):

```
User logs into App A (via agent) → gets AM session cookie
User visits App B (different agent, same AM domain) → agent reads same cookie
Agent B validates the session with AM → user is already authenticated → SSO!
```

**Requirements for SSO:**
- All agents point to the same AM instance (or cluster)
- All apps are on the same cookie domain (or use CDSSO for cross-domain)
- Cookie name is the same across all agents (default: `iPlanetDirectoryPro`)

**Cross-Domain SSO (CDSSO):** When apps are on different domains (e.g., `app1.company.com` and `app2.partner.com`), cookies don't travel. CDSSO uses a redirect-based mechanism — the agent redirects to AM with a special URL, AM sets the cookie on its domain, and redirects back with an encoded token that the agent converts to a local cookie.

---

## Q10: What are the common agent configuration settings?

**A:**

| Setting | Purpose | Example |
|---|---|---|
| **Not-Enforced URLs** | URLs that bypass agent protection (public pages) | `/public/*`, `/health`, `/favicon.ico` |
| **Login URL** | Where to redirect unauthenticated users | `http://am.example.com/am/XUI/#login/` |
| **AM URL** | AM server the agent talks to | `http://am.example.com:8081/am` |
| **Agent URL** | The agent's own URL (for redirect callbacks) | `http://app.example.com:8080/` |
| **Profile Attribute Mapping** | Which AM session attributes to pass as headers | `uid=X-AM-Username`, `memberOf=X-AM-Groups` |
| **Policy Cache TTL** | How long to cache policy decisions | 300 seconds (avoid calling AM on every request) |
| **Session Cache TTL** | How long to cache session validation | 60 seconds |
| **FQDN Check** | Validate request hostname matches agent config | Prevents host header injection |

**Not-Enforced URLs are critical:** In production, you must exclude health check endpoints, static assets, and public pages. Otherwise the load balancer's health probe gets redirected to AM login and the app is marked as down.

---

## Q11: What happens when the agent can't reach AM?

**A:** This is a critical production failure scenario:

**Default behavior:** Agent denies all requests (fail-closed). Users get 500 errors or redirect loops.

**Configuration options:**
- **Fail-closed (default):** No AM = no access. Safest from a security perspective.
- **Fail-open (configurable):** Allow requests through without validation. Dangerous — any unauthenticated user gets access. Only used for specific non-sensitive apps where availability > security.
- **Cache-based degraded mode:** Agent has cached session/policy data. If AM is unreachable, serve from cache until TTL expires. Then fail-closed.

**Production mitigation:**
- AM cluster with load balancer (agent configured with multiple AM URLs)
- Agent-side connection pooling and retry logic
- Monitoring: alert when agent-to-AM latency exceeds threshold
- Circuit breaker pattern in PingGateway (more sophisticated than agents)

**Interview answer:** "Our agents were configured fail-closed by default — if AM was unreachable, no traffic passed through. We mitigated this with an AM cluster behind a load balancer, session caching on the agent (60-second TTL), and policy caching (5-minute TTL). If AM was completely down for over 5 minutes, we had a runbook to temporarily switch to fail-open for critical business apps while restoring AM."

---

## Q12: How do agents compare to application-level OAuth2 integration?

**A:** Two different approaches to protecting apps with AM:

**Agent approach (traditional):**
```
User → Agent → AM (session validation + policy) → App
App receives: HTTP headers with user identity
App does: Nothing — agent handles everything
```

**OAuth2 approach (modern):**
```
User → App → AM (OAuth2 /authorize) → App receives tokens
App receives: access_token + id_token
App does: Validates tokens, extracts claims, makes authorization decisions
```

| Aspect | Agent | OAuth2 |
|---|---|---|
| App code changes | None | Must implement OAuth2 flow |
| Protocol | AM-proprietary session cookies | Standard OAuth2/OIDC |
| Interoperability | Only works with AM | Works with any OAuth2 provider |
| API protection | Agent intercepts + evaluates policies | App validates bearer tokens |
| Mobile/SPA | Doesn't work (no cookie support) | Works (token-based) |
| Best for | Legacy web apps, server-rendered pages | APIs, SPAs, mobile apps, microservices |

**The trend:** New applications use OAuth2/OIDC directly. Agents are for legacy apps that can't be modified. PingGateway bridges both — it can act as an agent (session validation) and as an OAuth2 resource server (token validation) simultaneously.

---

## Q13: Give the complete interview answer for "What are Web Agents and Java Agents?"

**A:**

> "Web Agents and Java Agents are Policy Enforcement Points that integrate with PingAM to protect applications. Web Agents are native C modules installed into web servers like Apache or IIS — they intercept HTTP requests and validate them against AM's session and policy services. Java Agents are servlet filters for Java app servers like Tomcat or JBoss — same concept but running in the JVM.
>
> Both work by intercepting every request, checking for a valid AM session cookie, and if the user is authenticated, calling AM's policy evaluation API to determine if the user is authorized for that specific resource and action. They inject user identity as HTTP headers so the application knows who the user is without implementing its own auth.
>
> In modern architectures, PingGateway is largely replacing agents because it runs as a standalone reverse proxy — no installation into the app server, independent upgrade cycle, and it supports both session-based and OAuth2 token-based protection. We still use Java Agents for legacy Java apps that can't be re-architected, but new services use OAuth2 directly or sit behind PingGateway."

---

## Prerequisite: Web Server vs App Server — First Principles

*Understanding this distinction is essential for understanding where Web Agents vs Java Agents are installed.*

### Q14: What is the fundamental difference between a web server and an app server?

**A:**

**Web Server** = Delivers **files**. Serves static content (HTML, CSS, JS, images), handles HTTP requests/responses. That's it.

**App Server** = Executes **code**. Runs business logic, connects to databases, generates dynamic content.

**Analogy:**
- Web Server = **Waiter** — takes your order, brings pre-made items
- App Server = **Kitchen** — cooks custom dishes, processes ingredients

### How they work together

**Static request (web server only):**
```
User requests: example.com/about.html
Web Server: "Here's the about.html file"
(No processing, just file delivery)
```

**Dynamic request (web server + app server):**
```
User requests: example.com/profile?user=alice
Web Server: "I need help, forwarding to app server"
App Server:
  1. Execute code
  2. Query database for Alice
  3. Generate HTML with Alice's data
Web Server: "Here's the generated page"
```

### What each handles

| Request Type | Who Handles It |
|---|---|
| `GET /logo.png` | Web Server |
| `GET /style.css` | Web Server |
| `GET /user/profile` | App Server |
| `POST /login` | App Server |
| `GET /calculate?a=5&b=3` | App Server |

### Common examples

**Web Servers:** Nginx, Apache HTTP Server, IIS

**App Servers:** Tomcat (Java), Node.js, Gunicorn (Python), WildFly/JBoss

### Typical production setup

```
User → Web Server → App Server → Database
        (Nginx)      (Tomcat)      (MySQL)
```

1. Web server handles static files directly
2. Web server **forwards** dynamic requests to app server
3. App server processes logic, returns result
4. Web server sends response to user

### Why use both?

**Web Server is good at:** serving files fast, SSL/TLS termination, load balancing, caching

**App Server is good at:** running code, database connections, business logic, session management

Together: web server handles simple stuff efficiently, app server handles complex processing.

---

### Q15: How does this relate to Web Agents vs Java Agents?

**A:** Now the agent placement makes sense:

- **Web Agent** → installed into the **web server** (Nginx/Apache/IIS) — intercepts requests at the front door, before they even reach the app server. Protects everything behind the web server regardless of what language or framework the app uses.

- **Java Agent** → installed into the **app server** (Tomcat/JBoss) — intercepts requests at the application level, inside the JVM. Only protects Java applications.

**In a typical two-tier setup:**

```
User → Nginx (Web Agent here) → Tomcat → Database
                                  ↑
                          Java Agent could go here instead
```

If you have Nginx in front of Tomcat, you'd typically put the Web Agent in Nginx (catches everything). If Tomcat is exposed directly (no web server in front), you'd use the Java Agent.

**In your lab:** PingAM itself runs on Tomcat (app server). If you wanted to protect a separate application with AM, you'd install an agent on that application's web server or app server — not on AM's own Tomcat.
