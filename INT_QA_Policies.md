# ForgeRock/PingAM Interview Questions — Policy-Based Authorization (Lab 5)

---

## Authorization Model

### Q1: Explain AM's authorization model — Resource Types, Policy Sets, and Policies

**Answer**: AM uses a 3-layer authorization model:

```
Resource Type          -->  "What CAN be protected" (URL patterns + actions)
    |                       Just a template. Protects nothing by itself.
    v
Policy Set             -->  "Group of policies for an application"
(Application)               Links to one or more Resource Types.
    |                       Still protects nothing by itself.
    v
Policy                 -->  "Actual access control rule"
                            WHO (subjects) + WHAT (resources) + WHEN (conditions)
                            = ALLOW or DENY
```

**Enforcement** requires a **Policy Enforcement Point (PEP)**:
- AM Web Agent (installed on app server)
- PingGateway / IG (reverse proxy)
- Application calling `/policies?_action=evaluate` REST endpoint

AM itself acts as the **Policy Decision Point (PDP)** — it evaluates policies and returns allow/deny decisions. But it does NOT intercept traffic. Something else (the PEP) must ask AM for a decision.

**Interview answer**: "AM's authorization separates concerns into three layers. Resource Types define the vocabulary — URL patterns and HTTP actions. Policy Sets group policies for an application. Policies define the actual rules with subjects, conditions, and actions. But none of this enforces anything until a PEP — like a Web Agent or PingGateway — intercepts requests and queries AM's PDP for a decision."

---

## Resource Types

### Q2: What is a Resource Type and how should it be designed?

**Answer**: A Resource Type defines:
- **Patterns**: URL templates that policies can reference (with wildcards)
- **Actions**: HTTP methods (GET, POST, PUT, DELETE, etc.) or custom actions

**Pattern syntax**:

| Pattern | Matches |
|---------|---------|
| `\*://\*:\*/\*` | Any URL (any scheme, host, port, path) |
| `\*://\*:\*/\*?\*` | Any URL with a query string |
| `http://api.techcorp.com/\*` | Any path under api.techcorp.com |
| `http://api.techcorp.com/data/\*` | Any path under /data/ specifically |

Breaking down the wildcard pattern:

```
scheme :// host : port / path
  *    ://  *   :  *   /  *
```

**Real-world design — Banking App example**:

BAD (too broad, no isolation):
- Resource Type: "URL" with pattern matching any URL
- Problem: Policy writer for banking API could accidentally affect admin portal

GOOD (scoped per application):

| Resource Type | Patterns | Actions |
|---------------|----------|---------|
| Internet Banking API | `https://api.mybank.com/accounts/\*`, `https://api.mybank.com/transfers/\*`, `https://api.mybank.com/payments/\*` | GET, POST, PUT, DELETE |
| Admin Portal | `https://admin.mybank.com/\*` | GET, POST |
| Public Website | `https://www.mybank.com/public/\*` | GET |

**Why scope Resource Types per application?**

| Concern | Broad (match-all) | Scoped (per-app patterns) |
|---------|-------------------|---------------------------|
| Least privilege | Policy can reference any URL | Policy limited to app's URL space |
| Separation of concerns | Banking team could write admin portal rules | Each team manages their own Resource Type |
| Audit | Must read every policy to find banking rules | Query policies by Resource Type |
| Accidental exposure | Misconfigured policy could grant access anywhere | Blast radius limited to the application |
| Action control | All actions available everywhere | Public site only has GET, no DELETE |

**When the default URL type is acceptable**:
- Lab / development environments
- Small organizations with few applications
- When a single team manages all policies

**Interview answer**: "In production, I'd create separate Resource Types per application. The banking API gets its own patterns with REST actions, the admin portal gets its own with only GET/POST. This enforces least privilege at the template level — a policy writer for the API physically cannot create rules affecting the admin portal. The default URL type is fine for labs but too broad for production."

---

### Q3: What is the difference between PDP and PEP in AM's architecture?

**Answer**:

| Component | Full Name | Role | AM Component |
|-----------|-----------|------|--------------|
| PDP | Policy Decision Point | Evaluates policies, returns allow/deny | AM server, `/policies?_action=evaluate` |
| PEP | Policy Enforcement Point | Intercepts requests, enforces the decision | Web Agent, PingGateway, or custom app |
| PAP | Policy Administration Point | Manages policies (CRUD) | AM Console / REST API |
| PIP | Policy Information Point | Provides attributes for conditions | User profile, session properties, environment |

**How they work together**:

```
User Request
    |
    v
PEP (Web Agent/Gateway) ---> "Can user X do GET on /accounts/123?"
    |
    v
PDP (AM Server) evaluates:
  - Subjects (WHO)
  - Resources (WHAT)
  - Conditions (WHEN)
    |
    v
ALLOW or DENY
    |
    v
PEP enforces (allow through or return 403)
```

**Critical point**: AM (PDP) does NOT intercept traffic. It only makes decisions when asked. Without a PEP, policies exist but are never enforced.

**Interview answer**: "AM is the PDP — it evaluates policies and returns decisions. But it doesn't sit in the request path. You need a PEP like a Web Agent installed on Apache/IIS, or PingGateway as a reverse proxy, to intercept requests and ask AM for authorization decisions. In modern architectures, applications can also act as their own PEP by calling AM's policy evaluation REST API directly."

---

### Q4: Are Resource Types fixed or user-defined? How many can you have?

**Answer**: Resource Types are **fully user-defined**. There is no fixed list. You can create as many as you need.

AM ships with two **defaults**:

| Default Resource Type | Created By | Patterns | Actions |
|---|---|---|---|
| URL | AM core (pre-installed) | Match-all wildcard patterns | GET, POST, PUT, DELETE, HEAD, OPTIONS, PATCH |
| OAuth2 Scope | Auto-created when OAuth2 Provider is configured | Scope names | GRANT/DENY |

But you can create custom ones with any name, any patterns, any actions:

```
Examples:
  "Banking API"       --> https://api.bank.com/*        --> GET, POST, PUT, DELETE
  "LDAP Resources"    --> ldap://ds.example.com/*        --> READ, WRITE, DELETE
  "Custom App"        --> myapp://resource/*              --> OPEN, CLOSE, EXECUTE
  "Microservice Mesh" --> https://svc-*.internal.com:*/* --> GET, POST
```

**Key points**:
- The **name** is just a label — AM uses an internal **UUID** to reference it. "URL" is not a special keyword.
- **Patterns** use wildcards — you define the URL structure your app uses
- **Actions** are string labels — HTTP methods for web resources, but can be anything for non-URL resources
- One Resource Type can be used by **multiple Policy Sets**
- A Policy Set can reference **multiple Resource Types**

**Interview answer**: "Resource Types are user-defined templates, not fixed types. AM ships with a default URL type for convenience, but in production I'd create application-specific Resource Types — 'Banking API' with specific URL patterns and REST actions. This scopes what each Policy Set can protect, enforcing separation of concerns between application teams."

---

### Q5: What is the OAuth2 Scope Resource Type and how does it differ from URL?

**Answer**: The OAuth2 Scope Resource Type is auto-created when you configure an OAuth2 Provider. It allows you to use AM's policy engine to control **which scopes are granted to which clients/users** — turning scope decisions into policy decisions.

| Aspect | URL Resource Type | OAuth2 Scope Resource Type |
|--------|-------------------|---------------------------|
| Patterns | URL templates (e.g. `https://api.com/\*`) | Scope names (`openid`, `profile`, `email`) |
| Actions | HTTP methods (GET, POST, etc.) | GRANT / DENY |
| Protects | Web resources / API endpoints | OAuth2 scope issuance |
| PEP | Web Agent, PingGateway, or app | AM's OAuth2 Provider (built-in) |
| Use case | "Can user X access /accounts/123?" | "Can client Y get the 'admin' scope for user Z?" |

**Example**: A policy using OAuth2 Scope Resource Type could say:
- "Only users in the `admins` group can be granted the `admin:write` scope"
- "The `financial-data` scope is only available during business hours"

**Interview answer**: "The OAuth2 Scope Resource Type lets you govern scope issuance through policies instead of static client configuration. This is powerful for fine-grained consent — for example, a high-privilege scope like `transfer:funds` could require MFA authentication level, while `profile:read` is available to anyone authenticated."

---

## Policy Evaluation

### Q6: How does AM evaluate policies when multiple policies match a resource?

**Answer**: When a resource matches multiple policies, AM combines the results:

**Rules**:
1. **Allow** from any policy grants the action (union of allows)
2. **Deny** from any policy overrides allow (deny wins)
3. **No matching policy** = implicit deny (access denied by default)

**Example from our lab**:

```
Resource: http://api.techcorp.com:80/data/reports
User: demo (member of employees group)

Policy 1: AllowEmployeeRead
  Resource: http://api.techcorp.com:80/*     <-- matches (wildcard)
  Actions: GET=Allow
  Subject: employees group                   <-- demo is a member

Policy 2: AllowEmployeeWrite
  Resource: http://api.techcorp.com:80/data/* <-- matches
  Actions: POST=Allow, PUT=Allow
  Subject: employees group                   <-- demo is a member

Combined result for demo on /data/reports:
  GET=Allow (from Policy 1) + POST=Allow, PUT=Allow (from Policy 2)
  DELETE = not mentioned anywhere = implicit deny
```

**For bob (not in employees group)**:

```
Neither policy matches (subject condition fails)
--> No actions returned
--> Implicit deny on everything
```

**Interview answer**: "AM uses a 'deny overrides' combining algorithm. If any matching policy explicitly denies an action, that overrides any allows. If no policy matches at all, access is implicitly denied — AM is deny-by-default. This means you don't need explicit deny policies for most cases; simply not granting access is sufficient. Explicit deny policies are useful when you need to carve out exceptions — like 'all employees can read, but NOT during maintenance windows.'"

---

### Q7: How do you evaluate policies via the REST API?

**Answer**: Use the `/policies?_action=evaluate` endpoint.

**Request**:

```
POST /am/json/realms/root/realms/{realm}/policies?_action=evaluate

Headers:
  iPlanetDirectoryPro: <admin_token>
  Content-Type: application/json
  Accept-API-Version: resource=2.0, protocol=1.0
```

**Request Body**:

```json
{
  "resources": [
    "http://api.techcorp.com:80/users",
    "http://api.techcorp.com:80/data/reports"
  ],
  "application": "TechCorpAPI",
  "subject": {
    "ssoToken": "<user_session_token>"
  }
}
```

**Response for an employee**:

```json
[
  {
    "resource": "http://api.techcorp.com:80/users",
    "actions": { "GET": true }
  },
  {
    "resource": "http://api.techcorp.com:80/data/reports",
    "actions": { "GET": true, "POST": true, "PUT": true }
  }
]
```

**Response for a non-employee (bob)**:

```json
[
  {
    "resource": "http://api.techcorp.com:80/users",
    "actions": {}
  },
  {
    "resource": "http://api.techcorp.com:80/data/reports",
    "actions": {}
  }
]
```

**Key points**:
- Empty `actions` object = implicit deny (no matching policy)
- The admin token is needed for API access; the `subject.ssoToken` is the user being evaluated
- You can evaluate multiple resources in a single call
- The `application` field is the Policy Set **ID** (not the display name)

**Interview answer**: "In production, the PEP — whether a Web Agent, PingGateway, or the application itself — calls the policy evaluation endpoint with the user's session token and the requested resource. AM evaluates all matching policies and returns the combined allow/deny decisions. The application then enforces the result. This separates the decision (AM) from the enforcement (PEP), making it easy to change authorization rules without modifying application code."

---

### Q8: What is implicit deny and when would you use explicit deny?

**Answer**:

| Aspect | Implicit Deny | Explicit Deny |
|--------|---------------|---------------|
| How | No policy grants access | Policy explicitly sets action to Deny |
| When | User doesn't match any policy subject | User matches policy but action is set to Deny |
| Override | Can be "fixed" by adding an Allow policy | Overrides ALL Allow policies (deny wins) |
| Use case | Default security posture | Exceptions, maintenance windows, revocations |

**When to use explicit deny**:
- Block a specific user despite group membership: "Alice is in employees but is suspended"
- Time-based restrictions: "No write access during maintenance window (2am-4am)"
- IP-based restrictions: "Deny access from outside corporate network"
- Emergency kill switch: "Deny all access to /admin during incident response"

**Interview answer**: "AM is deny-by-default — if no policy grants access, it's denied. This is the principle of least privilege. You only need explicit deny policies for exceptions: blocking specific users who would otherwise match an allow policy, enforcing time-based blackouts, or creating emergency access controls. Since deny always overrides allow, an explicit deny policy is a powerful override that can't be bypassed by other allow policies."

---

## Environment Conditions

### Q9: What environment conditions are available in AM policies?

**Answer**: Environment conditions control **when** a policy applies. They add context beyond just "who" and "what."

**Time conditions**:

| Type | Use Case |
|------|----------|
| SimpleTime | Business hours, maintenance windows, day-of-week restrictions |
| Session | Max session duration, terminate after idle time |

**Network conditions**:

| Type | Use Case |
|------|----------|
| IPv4 / IPv6 | Allow only from corporate network, block foreign IPs |
| ResourceEnvIP | IP + DNS name based conditions |

**Authentication conditions**:

| Type | Use Case |
|------|----------|
| AuthLevel | Require MFA (level 2+) for sensitive resources |
| LEAuthLevel | Less-than-or-equal auth level check |
| AuthScheme | Specific auth module required |
| AuthenticateToRealm | Must be authenticated to a specific realm |
| AuthenticateToService | Must have used a specific auth tree/chain |

**Advanced conditions**:

| Type | Use Case |
|------|----------|
| Script | Custom JavaScript/Groovy condition (most flexible) |
| LDAPFilter | Match user's LDAP attributes |
| SessionProperty | Check session property values |
| OAuth2Scope | OAuth2 scope-based conditions |
| Transaction | Require transaction token (step-up) |

**Logical operators**: AND, OR, NOT — to combine multiple conditions.

**SimpleTime condition fields**:

| Field | Format | Example |
|-------|--------|---------|
| Start Time | HH:MM | 09:00 |
| End Time | HH:MM | 17:00 |
| Start Day | Day name | Mon |
| End Day | Day name | Fri |
| Start Date | YYYY:MM:DD | 2026:01:01 |
| End Date | YYYY:MM:DD | 2026:12:31 |
| Time Zone | TZ name | IST, GMT, America/New_York |

**Real-world examples**:

1. **Maintenance Window Deny**: SimpleTime 02:00-04:00, Deny all writes -- "No changes during nightly maintenance"
2. **Corporate Network Only**: IPv4 = 10.0.0.0/8, Allow -- "Only accessible from internal network"
3. **MFA Required for Admin**: AuthLevel >= 2, Allow DELETE -- "Must use MFA to delete resources"
4. **Custom Script**: Script checks user's department attribute -- "Only finance department can access /payments"

**Interview answer**: "Environment conditions add contextual access control. The most common are SimpleTime for business hours restrictions, IPv4 for network-based access, and AuthLevel for step-up authentication. For complex logic, scripted conditions provide full flexibility — for example, checking a user's department attribute or querying an external risk engine. Conditions can be combined with AND/OR/NOT operators for complex rules."

---

### Q10: Explain time-based policy enforcement with a real example

**Answer**: Time-based conditions use SimpleTime to restrict **when** a policy is active.

**Our lab setup**:

```
Policy: DenyWriteOutsideBusinessHours
  Resources: http://api.techcorp.com:80/*
  Actions: POST=Deny, PUT=Deny, DELETE=Deny
  Subject: Authenticated Users (everyone)
  Condition: SimpleTime 18:00 - 08:00 IST
```

**During business hours (08:00-18:00 IST)**:

```
DenyWriteOutsideBusinessHours:
  Time condition 18:00-08:00 --> NOT active now
  Result: Policy DOES NOT APPLY

AllowEmployeeWrite:
  Subject: employees --> demo matches
  Result: POST=Allow, PUT=Allow

Combined: POST=ALLOW, PUT=ALLOW
```

**Outside business hours (18:00-08:00 IST)**:

```
DenyWriteOutsideBusinessHours:
  Time condition 18:00-08:00 --> ACTIVE now
  Result: POST=Deny, PUT=Deny, DELETE=Deny

AllowEmployeeWrite:
  Subject: employees --> demo matches
  Result: POST=Allow, PUT=Allow

Combined: POST=DENY, PUT=DENY (deny wins!)
          GET=ALLOW (not denied by any policy)
```

**Key insight**: The deny policy targets POST/PUT/DELETE but NOT GET. So employees can still **read** data outside business hours — they just can't **modify** it. This is a common production pattern for change freeze windows.

**Interview answer**: "We use time-based deny policies for maintenance windows and change freeze periods. The deny policy has a SimpleTime condition that makes it active only outside business hours. Since deny overrides allow in AM's evaluation, even employees with write access are blocked during that window. But read-only access remains unaffected because the deny only targets write actions. This is the principle of least restriction — only deny what's necessary."

---

## Testing and Production

### Q11: How does policy evaluation work end-to-end in production?

**Answer**: The full flow from user request to access decision:

```
1. User sends request
   GET http://api.techcorp.com/data/reports
   Cookie: iPlanetDirectoryPro=<session_token>

2. PEP intercepts (Web Agent / PingGateway)
   Extracts: resource URL, HTTP method, session token

3. PEP calls AM Policy API
   POST /am/json/realms/.../policies?_action=evaluate
   Body: resources, application, subject ssoToken

4. AM (PDP) evaluates
   a. Identify user from session token
   b. Find matching policies in TechCorpAPI policy set
   c. Check subject conditions
   d. Check environment conditions
   e. Combine actions

5. AM returns decision
   {"actions": {"GET": true, "POST": true, "PUT": true}}

6. PEP enforces
   GET requested, actions.GET = true --> ALLOW REQUEST THROUGH
   DELETE requested, not in actions  --> DENY, return 403 Forbidden
```

**Testing without a PEP** (our lab):
- We call the policy evaluation REST API directly
- We act as both the PEP and the application
- Two tokens needed: admin token (API access) + user token (who to evaluate)
- In production, the PEP has its own agent profile in AM for API access

**Postman testing steps**:
1. POST /authenticate (root realm) -- get admin tokenId
2. POST /authenticate (techcorp realm) -- get user tokenId
3. POST `/policies?_action=evaluate` with admin token in header, user token in body
4. Inspect response: actions map shows allow/deny per HTTP method

**Interview answer**: "In production, a Web Agent or PingGateway sits in front of the application and acts as the PEP. It intercepts every request, extracts the session token and resource URL, and calls AM's policy evaluation API. AM evaluates all matching policies considering subjects, conditions, and actions, then returns the combined decision. The PEP enforces it — allowing or blocking the request. The application never needs to know about authorization logic. For testing, we call the evaluation API directly via REST, acting as our own PEP."

---

### Q12: Explain every field in a policy evaluation response

**Answer**: A policy evaluation response contains these fields per resource:

```json
{
  "resource": "http://api.techcorp.com:80/data/reports",
  "actions": {},
  "attributes": {},
  "advices": {},
  "ttl": 9223372036854775807
}
```

| Field | Meaning | Example |
|-------|---------|---------|
| resource | The URL that was evaluated | `http://api.techcorp.com:80/users` |
| actions | Map of action to allow/deny. Empty = implicit deny | `{"GET": true, "POST": true}` or `{}` |
| attributes | Response attributes configured on the policy (custom key-value pairs sent to PEP) | `{"role": ["admin"]}` |
| advices | Hints for the PEP on how user could gain access | `{"AuthLevelConditionAdvice": ["2"]}` |
| ttl | How long (ms) the PEP can cache this decision. Max long = cache indefinitely | `9223372036854775807` |

**Understanding `actions`**:

| actions value | Meaning | PEP action |
|---------------|---------|------------|
| `{"GET": true}` | GET is explicitly allowed | Allow GET requests through |
| `{"GET": false}` | GET is explicitly denied | Block GET requests (403) |
| `{}` (empty) | No policy matched — implicit deny | Block ALL requests (403) |
| `{"GET": true, "POST": false}` | GET allowed, POST denied | Allow GET, block POST |

**Understanding `advices`**:

When empty, there's nothing the user can do to gain access. When populated, it hints at what could help:

| Advice | Meaning | PEP can do |
|--------|---------|------------|
| `AuthLevelConditionAdvice: ["2"]` | User needs auth level 2 (e.g., MFA) | Trigger step-up authentication |
| `AuthenticateToServiceConditionAdvice: ["MFATree"]` | User needs to authenticate via specific tree | Redirect to that auth tree |
| `SessionConditionAdvice` | Session doesn't meet requirements | Force re-authentication |

**Understanding `ttl`**:

| Value | Meaning |
|-------|---------|
| `9223372036854775807` | Java Long.MAX\_VALUE — "decision won't change, cache indefinitely" |
| `60000` | Cache for 60 seconds, then re-evaluate |
| `0` | Don't cache, always re-evaluate |

In practice, PEPs usually have their own cache TTL configured that overrides this.

**Real example from our lab**:

Bob (contractor, not in employees group):
```
AllowEmployeeRead:     Subject=employees --> bob NOT a member --> SKIP
AllowEmployeeWrite:    Subject=employees --> bob NOT a member --> SKIP
DenyWriteOutside...:   Subject=Authenticated Users --> bob matches
                       BUT only has Deny actions (no Allow to override)

Result: actions={} --> no actions granted --> PEP blocks everything
        advices={} --> no hint to gain access --> bob is simply not authorized
```

Demo (employee):
```
AllowEmployeeRead:     Subject=employees --> demo IS a member --> GET=Allow
AllowEmployeeWrite:    Subject=employees --> demo IS a member --> POST=Allow, PUT=Allow

Result: actions={"GET":true, "POST":true, "PUT":true}
        PEP allows GET, POST, PUT requests through
```

**Interview answer**: "The policy evaluation response tells the PEP exactly what to do. The `actions` map shows which HTTP methods are allowed or denied — empty means implicit deny. The `advices` field is powerful for step-up authentication — if a policy requires auth level 2 but the user only has level 1, AM returns an advice telling the PEP to trigger MFA. The `ttl` field controls caching — high-security resources can set a short TTL to force frequent re-evaluation, while static policies use Long.MAX\_VALUE for performance."

---

*Add more questions as lab progresses*
