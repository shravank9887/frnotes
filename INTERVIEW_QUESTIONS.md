# ForgeRock/PingAM Interview Questions

## Authentication Trees (Lab 2)

### Q1: What's the difference between Authentication Trees and Chains?

| Trees (Modern) | Chains (Legacy) |
|----------------|-----------------|
| Visual, node-based design | Linear, step-by-step |
| Non-linear flows with branching | Each module succeeds/fails sequentially |
| Reusable via Inner Trees | No reuse mechanism |
| Callbacks for progressive data collection | Collects all data upfront |
| Recommended for new implementations | Being phased out |

**Interview tip**: "We use Trees because they allow us to create complex, conditional authentication flows - like MFA only for high-risk logins - which chains can't easily support."

---

### Q2: Explain the callback mechanism

```
1. Client POSTs to /authenticate (empty body)
2. AM returns authId (JWT with state) + callbacks array
3. Client fills in callback input values and POSTs back
4. AM processes, may return more callbacks (MFA, etc.) or final result
5. On success: returns tokenId (SSO session token)
```

**Key point**: The `authId` JWT maintains tree state - AM is stateless between requests.

---

### Q3: What is a Page Node and why use it?

**Answer**: Groups multiple collector nodes into a single page/request. Without it, each collector would be a separate callback round-trip.

Example: Page Node groups Username + Password → single login page instead of two separate pages.

---

### Q4: How does Retry Limit Decision work?

**Answer**:
- Tracks attempts in tree state (stored in authId JWT)
- **Retry** outcome: allows another attempt
- **Reject** outcome: max attempts exceeded
- Connect Data Store Decision (False) back to Retry Limit to create loop

---

### Q5: How are trees stored in DS?

**Answer**:
```
DN Path:
ou={TreeName},ou=default,ou=OrganizationConfig,ou=1.0,
   ou=authenticationTreesService,ou=services,o={realm},
   ou=services,ou=am-config

Key attributes (sunKeyValue):
- entryNodeId: Starting node UUID
- nodes: JSON with node definitions and connections
- enabled: true/false
- innerTreeOnly: whether it can be called directly
```

---

### Q6: What are Inner Trees?

**Answer**: Reusable authentication trees called via "Inner Tree Evaluator" node.

Use cases:
- MFA flow reused across multiple login trees
- Common validation (CAPTCHA, terms acceptance)
- Step-up authentication
- DRY principle for auth flows

---

### Q7: What's the difference between authId and tokenId?

| authId | tokenId |
|--------|---------|
| **Temporary** JWT during authentication | **Persistent** session token after successful auth |
| Contains tree state (current node, attempts, etc.) | References session stored in CTS |
| Lives only during auth flow (short-lived ~5 min) | Lives for session duration (30min idle, 2hr max default) |
| Stateless - all state encoded in JWT | Stateful - points to session in CTS datastore |
| Used in `/authenticate` endpoint | Used as `iPlanetDirectoryPro` cookie |
| Cannot access protected resources | Grants access to protected resources |

**authId structure** (decoded JWT):
```json
{
  "authIndexType": "service",
  "authIndexValue": "TechCorpLogin",
  "realm": "/techcorp",
  "sessionId": "...",
  "otk": "one-time-key",
  "exp": 1769616984
}
```

**tokenId** (SSO Token):
```
cd4CkpcOSCPPdfIm4ZIfNfj2jIY.*AAJTSQACMDEAAlNLABx...*
```
- First part: token reference
- After `*`: encrypted session info (for stateless mode) or CTS reference

---

### Q8: How is session information stored? (CTS - Core Token Service)

**Answer**: Sessions are stored in the **CTS (Core Token Service)** which uses DS as its backend.

**CTS Token Store location in DS**:
```
ou=famrecords,ou=openam-session,ou=tokens,dc=cts,dc=example,dc=com
```

Or in embedded/external DS:
```
ou=tokens (root suffix for CTS)
├── ou=openam-session
│   └── ou=famrecords     ← Session tokens stored here
├── ou=oauth2-access      ← OAuth2 access tokens
├── ou=oauth2-refresh     ← OAuth2 refresh tokens
└── ou=saml2              ← SAML2 assertions
```

**Session Token Attributes**:
| Attribute | Description |
|-----------|-------------|
| `coreTokenId` | Unique session identifier |
| `coreTokenUserId` | User who owns the session |
| `coreTokenType` | Token type (SESSION) |
| `coreTokenExpirationDate` | When token expires |
| `coreTokenObject` | Serialized session data (encrypted) |

**LDAP Query to view sessions**:
```bash
# Find CTS base DN first
/opt/opendj/bin/ldapsearch \
  --hostname localhost --port 1636 --useSsl --trustAll \
  --bindDN "cn=Directory Manager" --bindPassword "Passw0rd123" \
  --baseDN "dc=openam,dc=forgerock,dc=org" \
  --searchScope sub "(objectClass=frCoreToken)" dn coreTokenId coreTokenUserId
```

**Alternative - Check via AM REST API** (easier):
```bash
# List all sessions for a user
curl -s "http://pingam:8081/am/json/sessions?_action=getSessionInfo" \
  -H "iPlanetDirectoryPro: <tokenId>" \
  -H "Accept-API-Version: resource=4.0"
```

---

### Q9: Stateful vs Stateless Sessions

| Stateful (Default) | Stateless |
|--------------------|-----------|
| Session stored in CTS (DS) | Session encoded in token itself |
| tokenId is a reference/pointer | tokenId contains full session |
| CTS lookup on each request | No backend lookup needed |
| Can be invalidated server-side | Harder to invalidate (blacklist needed) |
| Better for: security, logout | Better for: high scale, geo-distributed |

**How to identify stateless token**: Much longer tokenId with encrypted payload.

**Lab Observation**: Your sndbx1 environment uses stateless sessions - CTS `ou=famrecords` is empty, all session data is in the tokenId itself.

---

### Q11: Explain the AM SSO Token Format (tokenId)

**Token Structure**:
```
gTMxO9ytpd5gFl3eKE0FmSbpm5Q.*AAJTSQACMDEAAlNLABx...AAA.*
└──────────┬──────────────────┘ └─────────────┬─────────────┘
      Token Handle                  Encrypted Session Blob
      (random ID)                   (Base64 encoded)
```

**This is NOT a JWT** - it's AM's proprietary binary format:

| Component | Description |
|-----------|-------------|
| Token Handle | Random unique identifier |
| `.*` | Delimiter |
| Session Blob | Base64-encoded, **encrypted** session data |

**What's inside the blob** (encrypted):
- Session ID, User principal, Realm
- Auth level, Session properties
- Expiration times, Encryption signature

---

### Q12: Stateless Token Security - Can intercepted tokens be read?

**Key Points**:

| Question | Answer |
|----------|--------|
| Is token stored in CTS/DS? | **NO** - Only exists on client side |
| Can attacker READ the token? | **NO** - Encrypted with AM's secret key |
| Can attacker USE the token? | **YES** - Session hijacking possible |
| Who can decrypt? | **Only AM** - Has the encryption key |

**Security Model**:
```
┌─────────────────────────────────────────────────────────────┐
│ Client stores token (cookie: iPlanetDirectoryPro)          │
│                         ↓                                   │
│ Client sends token with each request                        │
│                         ↓                                   │
│ AM decrypts token → validates → grants access               │
└─────────────────────────────────────────────────────────────┘
```

**Attack Vectors & Mitigations**:

| Risk | Mitigation |
|------|------------|
| Token interception (MITM) | Use HTTPS/TLS only |
| Session hijacking (stolen token) | Short session timeouts, IP binding |
| Token replay | Idle timeout, `HttpOnly` + `Secure` cookie flags |
| Token tampering | Encryption + signature verification by AM |

**Interview Answer**:
> "The stateless SSO token is encrypted using AM's secret key, so an attacker who intercepts it cannot READ the session data - only AM can decrypt it. However, they CAN USE the token for session hijacking if intercepted. That's why we enforce HTTPS, use HttpOnly/Secure cookie flags, and implement short session timeouts. For high-security environments, we might use stateful sessions where tokens are just references to server-side session data that can be immediately invalidated."

---

### Q13: Stateless vs Stateful - When to use which?

| Use Stateless When | Use Stateful When |
|--------------------|-------------------|
| High scale / many users | Security-critical apps |
| Geo-distributed deployment | Need instant logout/revocation |
| Performance is priority | Session data must stay server-side |
| Microservices architecture | Compliance requires audit trail |

**Hybrid Approach**: Some deployments use stateless for general sessions but stateful for admin/privileged sessions.

---

### Q14: What encryption does AM use for stateless session tokens?

**Answer**: AM uses **symmetric encryption (AES)** for stateless session tokens.

| Type | Used For | Why |
|------|----------|-----|
| **Symmetric (AES)** ✓ | Session tokens | AM is both encryptor & decryptor, fast |
| Asymmetric (RSA) | JWT signing (OAuth2), SAML | When others need to verify |

**Why not asymmetric for session tokens?**
- AM creates the token AND validates it (same party)
- Symmetric is ~100x faster than RSA
- Asymmetric is for when sender ≠ receiver

**Key Configuration**:
```
Location: AM keystore (.jceks, .p12, or secrets store)
Algorithm: AES-128 or AES-256
```

**Cluster Requirement**:
```
┌──────────┐     ┌──────────┐     ┌──────────┐
│   AM-1   │     │   AM-2   │     │   AM-3   │
│ [SECRET] │     │ [SECRET] │     │ [SECRET] │
└──────────┘     └──────────┘     └──────────┘
      ↑               ↑               ↑
      └───────────────┴───────────────┘
           MUST be the same key!

If AM-1 issues token, AM-2 must decrypt it
→ Shared symmetric key across cluster
```

**Key Rotation**: AM supports multiple active encryption keys for zero-downtime rotation.

---

### Q10: How do you debug authentication tree issues?

**Answer**:
1. **Enable Debug Logging**:
   - Debug → Configuration → Instance → "Authentication" → MESSAGE level

2. **Check AM Logs**:
   ```bash
   docker logs pingam | grep -i "auth"
   ```

3. **Test via REST API**: See exact callbacks and responses

4. **Use Message Nodes**: Display debug info during development

5. **Check DS**: Verify tree exists and is enabled
   ```bash
   ldapsearch ... "(sunServiceID=TreeName)" sunKeyValue
   ```

---

## Realm Architecture (Lab 1)

### Q1: What is a realm in AM?

**Answer**: A realm is an administrative boundary that contains:
- Users/Identities
- Authentication configuration
- Authorization policies
- Applications/Agents
- Services configuration

Think of it as a tenant or namespace.

---

### Q2: How is realm data stored in DS?

**Answer**:
```
ou=services,ou=am-config              ← Global/root realm config
├── o=techcorp,ou=services,ou=am-config  ← Sub-realm config
│   ├── ou=authenticationTreesService
│   ├── ou=iPlanetAMAuthService
│   └── ...
```

User identities stored separately:
```
ou=people,ou=identities               ← Root realm users
ou=people,o=techcorp,ou=identities    ← techcorp realm users (if external)
```

---

## OAuth2/OIDC (Lab 3)

### Q15: Explain the Client Credentials grant flow

**Answer**: A 2-legged flow where the client authenticates directly with the authorization server - no user involved.

```
┌──────────┐                         ┌──────────┐
│  Client   │──── client_id +  ─────→│    AM     │
│ (Service) │     client_secret       │ (OAuth2  │
│           │     grant_type=         │ Provider)│
│           │     client_credentials  │          │
│           │←── access_token ────────│          │
└──────────┘                         └──────────┘
```

**Use cases**: Service-to-service communication, backend APIs, scheduled jobs, microservices calling each other.

**Key points**:
- No `id_token` returned (no user identity)
- No `refresh_token` by default (client can just request a new token)
- The `sub` claim identifies the client as an agent: `(age!client-name)`
- Token endpoint: `/am/oauth2/realms/root/realms/{realm}/access_token`

**REST call**:
```bash
curl -X POST ".../access_token" \
  -d "grant_type=client_credentials" \
  -d "client_id=myapp" \
  -d "client_secret=secret" \
  -d "scope=profile"
```

---

### Q16: What is token introspection and when is it used?

**Answer**: Token introspection (RFC 7662) allows a resource server to validate an opaque access token by calling the authorization server's `/introspect` endpoint.

**When it's needed**:
- Opaque tokens (resource server can't decode them locally)
- Need to check if a token has been revoked
- Need token metadata (scopes, expiry, client)

**When it's NOT needed**:
- JWT access tokens (self-contained, validate signature locally)

**Introspection response**:
```json
{
  "active": true,
  "scope": "profile",
  "realm": "/techcorp",
  "client_id": "techcorp-app",
  "token_type": "Bearer",
  "exp": 1769707140,
  "sub": "(age!techcorp-app)",
  "iss": "http://pingam:8081/am/oauth2/realms/root/realms/techcorp"
}
```

**Interview tip**: "If `active: false` is returned, the token is expired, revoked, or invalid. The resource server should reject the request."

---

### Q17: Opaque tokens vs JWT access tokens - what's the difference?

| | Opaque | JWT |
|---|---|---|
| **Format** | Short random string (e.g., `sKYB_tIMvf65...`) | Three base64 sections: `header.payload.signature` |
| **Content** | Contains no information - just a reference | Self-contained with claims (sub, scope, exp, etc.) |
| **Validation** | Must call `/introspect` endpoint | Verify signature locally (no network call) |
| **Size** | Small (~27 chars) | Larger (~500+ chars) |
| **Revocation** | Delete from CTS - immediate | Must use token blacklist or wait for expiry |
| **AM setting** | "Use Client-Side Access Tokens" = disabled | "Use Client-Side Access Tokens" = enabled |

**JWT access token decoded example**:
```json
{
  "sub": "(age!techcorp-app)",
  "cts": "OAUTH2_STATELESS_GRANT",
  "iss": "http://pingam:8081/am/oauth2/realms/root/realms/techcorp",
  "token_type": "Bearer",
  "grant_type": "client_credentials",
  "scope": ["profile"],
  "exp": 1769709202,
  "jti": "VLf8RkYTekY391NzETlNA87rNF0",
  "aud": "techcorp-app"
}
```

**Key claim**: `cts: OAUTH2_STATELESS_GRANT` confirms stateless mode.

**Interview answer**: "We use opaque tokens when we want centralized control and easy revocation. JWT tokens are better for distributed architectures where resource servers need to validate tokens without calling AM, reducing latency and AM load."

---

### Q18: Stateless vs stateful OAuth2 tokens in AM

| | Stateful | Stateless (Client-Side) |
|---|---|---|
| **Token format** | Opaque reference | JWT (signed, optionally encrypted) |
| **CTS storage** | Source of truth | Stored for revocation only |
| **Validation** | CTS/DS lookup | Signature verification |
| **Revocation** | Delete from CTS | Token blacklist |
| **Scalability** | Limited by DS | Better (no backend lookup) |
| **AM setting** | "Use Client-Side Access Tokens" = off | "Use Client-Side Access Tokens" = on |
| **Signing** | N/A | Configured via "OAuth2 Token Signing Algorithm" (default HS256) |

**Important**: AM stores a CTS record in **both** modes. For stateless, the CTS record exists for revocation/blacklisting purposes, not for validation.

**CTS attributes for OAuth2 tokens**:
| Attribute | Meaning |
|---|---|
| `coreTokenType` | `OAUTH` (vs `SESSION` for AM sessions) |
| `coreTokenId` | The access token string |
| `coreTokenString09` | Client ID |
| `coreTokenString12` | Grant type |
| `coreTokenString01` | Scope |
| `coreTokenString08` | Realm |
| `coreTokenString10` | Token name (access_token) |

---

### Q19: Where are OAuth2 tokens stored in DS?

**Answer**: In the CTS (Core Token Service) under:
```
ou=famrecords,ou=openam-session,ou=tokens,ou=am-config
```

**LDAP query to view all OAuth2 tokens**:
```bash
docker exec pingds /opt/opendj/bin/ldapsearch \
  --hostname localhost --port 1636 --useSsl --trustAll \
  --bindDN "cn=Directory Manager" --bindPassword "Passw0rd123" \
  --baseDN "ou=famrecords,ou=openam-session,ou=tokens,ou=am-config" \
  --searchScope sub "(objectClass=frCoreToken)"
```

**Note**: The base DN varies by deployment. Use `namingContexts` search to find the correct suffix if unsure:
```bash
ldapsearch --baseDN "" --searchScope base "(objectClass=*)" namingContexts | grep token
```

---

### Q20: What are AM identity subject prefixes?

**Answer**: AM uses prefixes in the `sub` (subject) claim to indicate the type of identity:

| Prefix | Meaning | Example | When You See It |
|---|---|---|---|
| `age!` | Agent (OAuth2 client, web/J2EE agent) | `(age!techcorp-app)` | Client Credentials flow |
| `usr!` | User | `(usr!demo)` | Authorization Code flow |
| `grp!` | Group | `(grp!employees)` | Group-based policies |

**Why it matters**:
- In **Client Credentials** flow: `sub` = `(age!client-name)` because the client authenticates as itself (no user)
- In **Authorization Code** flow: `sub` = `(usr!username)` because a user authenticated and consented

**DS storage**: Agents are stored under `ou=agent` in the realm:
```
id=techcorp-app,ou=agent,o=techcorp,ou=services,ou=am-config
```

Users are stored under `ou=people` in the identity store.

**Interview tip**: "If I see `age!` in a token's subject, I know it was issued via Client Credentials or a similar machine-to-machine flow. If I see `usr!`, a human user authenticated."

---

### Q21: Explain the Authorization Code grant flow

**Answer**: A 3-legged flow involving the **user**, **client application**, and **authorization server**. The most common and secure flow for web applications.

```
┌──────────┐      ┌──────────┐      ┌──────────┐
│   User    │      │  Client   │      │    AM     │
│ (Browser) │      │   (App)   │      │ (OAuth2) │
└────┬─────┘      └────┬─────┘      └────┬─────┘
     │  1. Click Login  │                 │
     │─────────────────→│                 │
     │                  │ 2. Redirect to  │
     │                  │    /authorize   │
     │←─────────────────│─────────────────→
     │  3. Login page   │                 │
     │←────────────────────────────────────│
     │  4. User enters  │                 │
     │     credentials  │                 │
     │─────────────────────────────────────→
     │  5. Consent      │                 │
     │←────────────────────────────────────│
     │  6. Allow        │                 │
     │─────────────────────────────────────→
     │  7. Redirect with code             │
     │←───────────────────────────────────│
     │─────────────────→│                 │
     │                  │ 8. Exchange     │
     │                  │    code for     │
     │                  │    tokens       │
     │                  │─────────────────→
     │                  │ 9. access_token │
     │                  │    id_token     │
     │                  │    refresh_token│
     │                  │←────────────────│
     │  10. Logged in!  │                 │
     │←─────────────────│                 │
```

**Key points**:
- Authorization code is **single-use** and **short-lived** (~120 seconds)
- Code is exchanged server-side (client_secret never exposed to browser)
- Returns `access_token` + `id_token` (if `openid` scope) + `refresh_token`
- `sub` claim shows `(usr!username)` since a user authenticated

**Authorize endpoint**:
```
GET /am/oauth2/realms/root/realms/{realm}/authorize
  ?response_type=code
  &client_id=techcorp-app
  &redirect_uri=http://localhost:3000/callback
  &scope=openid profile email
  &state=abc123
```

**Token exchange**:
```bash
curl -X POST ".../access_token" \
  -d "grant_type=authorization_code" \
  -d "code=<AUTHORIZATION_CODE>" \
  -d "redirect_uri=http://localhost:3000/callback" \
  -d "client_id=techcorp-app" \
  -d "client_secret=T3chC0rp!"
```

---

### Q22: How do you simulate the Authorization Code flow via REST (without a browser)?

**Answer**: Three-step process:

1. **Authenticate the user** → get `tokenId` (iPlanetDirectoryPro session cookie)
2. **GET /authorize** with session cookie → AM returns consent HTML page containing a `csrf` token
3. **POST /authorize** with `csrf` + `decision=allow` + session cookie → AM returns `302 redirect` with `code` in the `Location` header
4. **POST /access_token** with `code` → get tokens

**Why the csrf?**: AM requires a CSRF token to prevent cross-site request forgery on the consent approval. The csrf is generated per-request and embedded in the consent page HTML.

**Implied Consent note**: Even with "Implied Consent" enabled on the client, the REST flow still returns the consent page (HTTP 200). You must POST `decision=allow` with the csrf. Implied Consent only auto-approves in the **browser XUI flow**.

**Interview tip**: "In production, you'd never do this via REST - the browser handles the redirects. But understanding the underlying HTTP flow is important for debugging OAuth2 issues."

---

### Q23: What is the difference between access_token and id_token?

| | access_token | id_token |
|---|---|---|
| **Purpose** | Authorize access to protected resources (APIs) | Prove the user's identity to the client |
| **Audience** | Resource server (API) | Client application |
| **Signing algorithm** | HS256 (symmetric - shared secret) | RS256 (asymmetric - public/private key) |
| **Who validates** | Resource server (via introspect or signature) | Client app (using AM's public key) |
| **Contains** | Scopes, grant type, expiry | User identity, auth context, session |
| **Used as** | `Authorization: Bearer <token>` header | Decoded client-side for user info |
| **Standard** | OAuth2 | OIDC (OpenID Connect) |

**Why different signing algorithms?**
- **access_token (HS256)**: AM and the resource server share a secret. Symmetric = fast.
- **id_token (RS256)**: Any client can verify using AM's **public key** (from JWKS endpoint). No shared secret needed. This is critical because id_tokens may be verified by many different clients.

**id_token key claims**:
| Claim | Meaning |
|---|---|
| `sub` | Subject - the authenticated user, e.g., `(usr!demo)` |
| `aud` | Audience - the client this token is for |
| `azp` | Authorized party - the client that requested it |
| `auth_time` | When the user actually authenticated |
| `acr` | Authentication Context Class Reference (auth level) |
| `at_hash` | Hash of access_token (binds id_token to access_token) |
| `c_hash` | Hash of authorization code (binds id_token to code) |
| `sid` | Session ID |
| `iss` | Issuer (AM's OAuth2 provider URL) |

**Interview answer**: "The access_token is for the API - it says what the bearer is allowed to do. The id_token is for the client application - it says who the user is. They serve different purposes and go to different audiences, which is why they use different signing algorithms."

---

### Q24: What is the `state` parameter in OAuth2 and why is it important?

**Answer**: The `state` parameter is a **CSRF protection mechanism** for the OAuth2 Authorization Code flow.

**How it works**:
1. Client generates a random `state` value and includes it in the `/authorize` request
2. AM includes the same `state` in the redirect back to the client
3. Client verifies the `state` matches what it sent

**Why it matters**:
- Without `state`, an attacker could craft a malicious `/authorize` URL and trick a user into authorizing access to the attacker's account
- The client should reject any callback where `state` doesn't match

**Interview tip**: "The `state` parameter prevents CSRF attacks on the OAuth2 flow. It's technically optional in the spec but should always be used in production. Some implementations also use it to maintain application state across the redirect."

---

### Q25: Compare all OAuth2 grant types

| Grant Type | Legs | User Involved | Use Case |
|---|---|---|---|
| **Authorization Code** | 3 | Yes | Web apps, mobile apps (with PKCE) |
| **Client Credentials** | 2 | No | Service-to-service, backend APIs |
| **Implicit** | 2 | Yes | *Deprecated* - was for SPAs |
| **Resource Owner Password** | 2 | Yes (credentials shared) | *Legacy* - migration scenarios |
| **Device Code** | 3 | Yes (on separate device) | Smart TVs, CLI tools |
| **JWT Bearer** | 2 | No | Token exchange, federated identity |

**What's returned by each**:
| Grant Type | access_token | id_token | refresh_token |
|---|---|---|---|
| Authorization Code | Yes | Yes (with `openid`) | Yes |
| Client Credentials | Yes | No | No |
| Implicit | Yes | Yes (with `openid`) | No |
| Resource Owner Password | Yes | Yes (with `openid`) | Yes |

**Interview tip**: "For new applications, we recommend Authorization Code with PKCE (even for public clients like SPAs). Implicit is deprecated due to token exposure in the browser URL. Client Credentials is for machine-to-machine only."

---

## SAML2 Federation (Lab 4)

### Q26: Explain the SAML2 SP-Initiated SSO flow

**Answer**:

```
1. User visits SP application (e.g., partner app)
2. SP generates SAML AuthnRequest
3. SP redirects browser to IdP's SingleSignOnService endpoint
   (via HTTP-Redirect or HTTP-POST binding)
4. IdP checks for existing session:
   - If session exists → skip login (SSO!)
   - If no session → show login page
5. User authenticates at IdP
6. IdP builds SAML Response containing:
   - Assertion with NameID (user identifier)
   - AuthnStatement (how they authenticated)
   - Conditions (validity period, audience restriction)
   - Optionally: AttributeStatement (user attributes)
7. IdP signs the assertion with its private key
8. IdP POSTs the SAML Response to SP's Assertion Consumer Service (ACS)
9. SP validates:
   - XML signature (using IdP's certificate from metadata)
   - Conditions (time validity, audience)
   - Extracts NameID
10. SP maps NameID to local user (Account Mapper)
11. SP creates local session
12. User is redirected to the target resource (RelayState URL)
```

**Interview tip**: "SP-initiated is the most common flow. The key security mechanism is the XML digital signature — the SP trusts the assertion because only the IdP has the private key that matches the certificate in the metadata."

---

### Q27: What is the difference between SP-Initiated and IdP-Initiated SSO?

| | SP-Initiated | IdP-Initiated |
|---|---|---|
| **Starts at** | Service Provider (app) | Identity Provider |
| **AuthnRequest** | Yes — SP sends one | No — IdP sends unsolicited assertion |
| **Use case** | User clicks login on app | User clicks app link from IdP portal |
| **Security** | More secure (SP can verify InResponseTo) | Less secure (no request to correlate) |
| **RelayState** | Set by SP (where to redirect after SSO) | Set by IdP (target app URL) |
| **Common in** | Most web apps | Enterprise portals, dashboards |

**SP-Initiated URL**:
```
/am/saml2/jsp/spSSOInit.jsp?metaAlias=/partner/partner-sp&idpEntityID=techcorp-idp
```

**IdP-Initiated URL**:
```
/am/saml2/jsp/idpSSOInit.jsp?metaAlias=/techcorp/idp&spEntityID=partner-sp
```

**Interview tip**: "IdP-initiated SSO is considered less secure because there's no AuthnRequest to correlate the response to, making it potentially vulnerable to unsolicited response attacks. We prefer SP-initiated when possible."

---

### Q28: What is a Circle of Trust (CoT)?

**Answer**: A Circle of Trust is a logical grouping of SAML entity providers (IdPs and SPs) that have agreed to trust each other for federated SSO.

**Key points**:
- Both IdP and SP must be in the same CoT (or linked CoTs)
- CoT is configured per-realm in AM
- An entity can be in multiple CoTs
- In cross-realm federation, each realm has its own CoT definition, but they reference each other's entities

**Our lab setup**:
```
/techcorp realm CoT: "techcorp-cot"
  - techcorp-idp (hosted IdP)
  - partner-sp (remote SP)

/partner realm CoT: "techcorp-cot"
  - partner-sp (hosted SP)
  - techcorp-idp (remote IdP)
```

**Interview answer**: "A Circle of Trust establishes which identity and service providers trust each other. In AM, you configure it per realm. For cross-realm federation, each realm's CoT includes the local hosted entity and remote copies of the partner entities imported via metadata."

---

### Q29: What is SAML metadata and why is it important?

**Answer**: SAML metadata is an XML document that describes an entity provider's configuration — endpoints, certificates, supported bindings, and NameID formats.

**Key elements in IdP metadata**:
| Element | Purpose |
|---|---|
| `EntityDescriptor` | Root element with entity ID |
| `SingleSignOnService` | SSO endpoint URLs (POST and Redirect bindings) |
| `SingleLogoutService` | SLO endpoint URLs |
| `ArtifactResolutionService` | For artifact binding |
| `KeyDescriptor (signing)` | X.509 certificate for signature verification |
| `KeyDescriptor (encryption)` | X.509 certificate for assertion encryption |
| `NameIDFormat` | Supported NameID formats |

**Key elements in SP metadata**:
| Element | Purpose |
|---|---|
| `AssertionConsumerService` | ACS endpoint (receives assertions) |
| `SingleLogoutService` | SLO endpoints |
| `KeyDescriptor` | Signing and encryption certificates |
| `NameIDFormat` | Supported NameID formats |

**AM metadata export URL**:
```
/am/saml2/jsp/exportmetadata.jsp?entityid=<ENTITY_ID>&realm=<REALM>
```

**Interview tip**: "Metadata exchange is the foundation of SAML trust. In production, we exchange metadata files (or URLs) between IdP and SP during onboarding. The certificates in the metadata are what allow signature verification — the SP validates the IdP's assertion signature using the public key from the IdP's metadata."

---

### Q30: Explain NameID formats — when to use which?

| Format | Value | Use Case |
|---|---|---|
| **persistent** | Opaque random ID (e.g., `3f7a9b2c...`) | Privacy-preserving, long-term federation link |
| **transient** | Random ID per session | Anonymous/pseudonymous access, no permanent mapping |
| **unspecified** | Application-defined (e.g., username) | Simple mapping, shared identity stores |
| **emailAddress** | Email (e.g., `demo@example.com`) | When SP identifies users by email |

**How NameID mapping works in AM**:

```
IdP Side (what to SEND):
  Hosted IdP → Assertion Content → NameID Value Map
  Maps format → user attribute
  Example: unspecified = cn  →  IdP reads user's cn attribute

SP Side (what to DO with it):
  Hosted SP → Assertion Processing → Account Mapper
  Maps NameID → local user
  "Use Name ID as User ID" = ON  →  treats NameID as uid for lookup
```

**Gotcha**: AM's identity API treats `uid` as the identity name, not a readable profile attribute. Use `cn` or other attributes in the NameID Value Map instead.

**Interview answer**: "We use `persistent` for privacy-sensitive federations where users shouldn't be trackable across providers. `Transient` for anonymous access. `Unspecified` when IdP and SP share an identity store or have a known attribute mapping. `emailAddress` is common for SaaS integrations like Salesforce or Google Workspace."

---

### Q31: What is the MetaAlias and how does it work?

**Answer**: The MetaAlias is a URL path fragment that identifies a SAML entity within AM. AM uses it to route SAML messages to the correct entity in the correct realm.

**Structure**: `/<realm-path>/<local-alias>`

| Realm | Local Alias | Full MetaAlias |
|---|---|---|
| `/techcorp` | `idp` | `/techcorp/idp` |
| `/partner` | `partner-sp` | `/partner/partner-sp` |
| `/` (root) | `my-sp` | `/my-sp` |

**Where it appears**: In all SAML endpoint URLs:
```
http://pingam:8081/am/SSORedirect/metaAlias/techcorp/idp
http://pingam:8081/am/Consumer/metaAlias/partner/partner-sp
```

**Common mistake**: Setting the local alias to `/techcorp-idp` in the `/techcorp` realm creates a double-slash: `/techcorp//techcorp-idp`. The alias should NOT include the realm path.

**Interview tip**: "MetaAlias is how AM routes incoming SAML messages to the correct entity. When debugging SAML issues, always check the metaAlias in the URL matches the entity configuration. A double-slash in the metaAlias is a common misconfiguration symptom."

---

### Q32: What is RelayState in SAML?

**Answer**: RelayState is an opaque parameter carried through the SAML flow that tells the SP where to redirect the user after successful SSO.

**Flow**:
```
1. SP sets RelayState = "https://app.example.com/dashboard"
2. RelayState is included in the AuthnRequest redirect to IdP
3. IdP echoes RelayState back in the SAML Response POST to SP
4. SP redirects user to the RelayState URL after processing the assertion
```

**Security**: AM validates RelayState URLs against the SP's **Relay State URL List** (Advanced tab). This prevents open redirect attacks where an attacker could craft a SAML flow that redirects to a malicious site after login.

**Configuration**: Hosted SP → Advanced → Relay State URL List.

**Validation behavior**: AM validates the RelayState before sending the AuthnRequest (at SP initiation time, not after assertion). Some AM versions do strict matching rather than prefix matching. Best practice: add all exact URLs your app might use as RelayState (e.g., both `http://localhost:3000` and `http://localhost:3000/protected`).

**When validation fails**: AM returns HTTP 400 "Server Error" — the Federation debug log shows `SAML2Exception: Invalid Relay State URL specified` at `SPSSOFederate.initiateAuthnRequest`.

**Interview answer**: "RelayState preserves the user's original destination through the SAML redirect flow. It's critical for user experience — without it, users would land on a generic page after SSO instead of the page they originally requested. AM validates RelayState URLs to prevent open redirects."

---

### Q33: Explain the 3 SP architectures for SAML federation

**Architecture 1: App has built-in SAML SP** (Salesforce, AWS, Google Workspace)
```
IdP (PingAM) --SAML assertion--> App with built-in SP
```
- App handles SAML directly. Just configure IdP metadata in the app.
- Examples: Salesforce, AWS IAM, Jira, ServiceNow

**Architecture 2: AM acts as SP gateway** (Our lab)
```
IdP (PingAM /techcorp) --SAML assertion--> AM as SP (/partner) --session--> Custom App
```
- App has NO SAML support. AM receives assertion, creates session.
- App validates AM session via REST API, Web Agent, or PingGateway.
- Cross-domain cookie problem: AM cookie on `pingam`, app on different domain.

**Architecture 3: Federation Hub**
```
Multiple IdPs --> AM (SP + Hub) --> Multiple Apps
```
- AM federates with many external IdPs and provides SSO to many internal apps.
- Centralizes federation management.

**Interview answer**: "Architecture 2 is common in enterprises with legacy apps. AM acts as the SAML SP, handles assertion validation, and creates a session. The app then validates that session — in production via a Web Agent or PingGateway, which sits on the same domain and passes user info via HTTP headers. In our lab, we used REST API validation with manual token entry due to the cross-domain constraint."

---

### Q34: How do you debug SAML federation issues in PingAM?

**Answer**: Systematic debugging approach:

1. **Check the Federation debug log**:
   ```
   /opt/am-config/var/debug/Federation
   ```
   This logs all SAML processing — AuthnRequest generation, assertion validation, NameID mapping errors.

2. **Common errors and fixes**:

   | Error | Cause | Fix |
   |---|---|---|
   | `No values provided for a request parameter` | Malformed metaAlias (double slash) | Recreate entity with correct meta alias |
   | `Invalid Relay State URL specified` | RelayState URL not in allowlist | Add URL to SP's Relay State URL List |
   | `No local user being mapped` | SP can't map NameID to local user | Configure Account Mapper or Auto Federation |
   | `Unable to generate NameID value` | IdP can't read user attribute for NameID | Check NameID Value Map attribute exists (use `cn` not `uid`) |
   | `Signature verification failed` | Certificate mismatch | Re-import remote entity metadata |

3. **Use a SAML tracer**: Browser extension (SAML-tracer for Firefox/Chrome) to inspect AuthnRequest and Response XML in real-time.

4. **Check metadata**: Export and compare IdP/SP metadata for correct endpoints and certificates.

5. **Verify CoT membership**: Both entities must be in the same Circle of Trust.

6. **Test without RelayState first**: Isolate SAML flow issues from application redirect issues.

**Interview tip**: "My debugging approach is: check the Federation debug log first for the exact error, then use a SAML tracer to inspect the AuthnRequest and Response XML. Common issues are metaAlias misconfiguration, missing NameID mappings, and certificate mismatches after key rotation."

---

### Q35: Hosted entity vs Remote entity — what's the difference?

| | Hosted Entity | Remote Entity |
|---|---|---|
| **What** | Entity that AM operates locally | Entity that AM knows about (partner) |
| **Configuration** | Full config (mappers, keys, all settings) | Imported metadata only (endpoints, certs) |
| **Created by** | AM admin in the local realm | Importing partner's metadata XML/URL |
| **Example** | `techcorp-idp` in `/techcorp` realm | `partner-sp` registered in `/techcorp` realm |
| **When to re-import** | N/A | Only when metadata changes (endpoints, certs) |

**Key insight**: Internal configuration (NameID Value Map, Attribute Mapper, Account Mapper, Relay State URL List) is all on the **hosted** entity. Changing these does NOT require re-importing the remote entity on the partner side.

---

### Q36: Internal vs External URLs in AM SAML deployments

**Answer**: In containerized/proxied deployments, AM has two URL contexts:

| Context | URL | Used By |
|---|---|---|
| **External** (browser-facing) | `http://pingam:8081/am` | SAML redirects, login pages, user-facing URLs |
| **Internal** (container-to-container) | `http://pingam:8080/am` | Backend REST API calls, session validation |

**AM Site Configuration**: In production, this is managed via AM's Site Configuration:
- **Site URL**: What browsers see (load balancer/proxy URL)
- **Server URL**: Internal AM server URL

**Our lab example** (sample-app):
```javascript
const AM_INTERNAL = 'http://pingam:8080/am';  // Backend REST calls (inside Docker network)
const AM_EXTERNAL = 'http://pingam:8081/am';   // Browser redirects (host-mapped port)
```

**Interview answer**: "In production AM deployments behind a load balancer, the Site URL is the external FQDN (e.g., `https://sso.example.com/am`) while the server URL is the internal address. SAML metadata must use the external URL because browsers follow the endpoints. Internal services use the server URL for backend API calls. Misconfiguring this is a common deployment issue."

---

### Q37: What happens when the IdP reuses an existing session during SAML SSO?

**Answer**: This is the core of SSO — "Single Sign-On" means authenticating once and accessing multiple SPs without re-entering credentials.

**Flow**:
```
1. User logs into SP-A via IdP → IdP creates session (iPlanetDirectoryPro cookie)
2. User visits SP-B → SP-B redirects to IdP
3. IdP finds existing session cookie → SKIPS login
4. IdP immediately generates assertion for SP-B using the existing session's user
5. SP-B receives assertion → creates local session
```

**Gotcha**: The IdP generates the NameID for **whatever user currently has the session**. If an admin is logged in (e.g., `amadmin`) and triggers a SAML flow, the IdP tries to generate a NameID for `amadmin`, which may fail if that user doesn't have the mapped attributes.

**Interview answer**: "The IdP checks for an existing session before prompting for login. This is SSO working as designed. But it means the NameID mapping must work for ALL users who might initiate the flow, not just test users. In debugging, a stale admin session is a common cause of unexpected NameID generation failures."

---

### Q38: When does AM validate the RelayState URL — at SP initiation or after receiving the assertion?

**Answer**: AM validates RelayState **at SP initiation time** — before the AuthnRequest is even sent to the IdP.

```
1. App redirects to AM SP with RelayState=http://app.example.com/dashboard
2. AM SP checks RelayState against Relay State URL List  ← VALIDATION HERE
3. If invalid → HTTP 400 immediately (flow never reaches the IdP)
4. If valid → SP sends AuthnRequest to IdP
```

**Why this matters**:
- The error appears **before** the user sees a login page
- The user sees "Server Error" on the AM SP page, not an IdP error
- The Federation debug log shows: `SAML2Exception: Invalid Relay State URL specified` at `SPSSOFederate.initiateAuthnRequest`

**Validation strictness**:
- Some AM versions do **strict matching** (exact URL must be in the list)
- Others do **prefix matching** (`http://app.com` matches `http://app.com/dashboard`)
- Best practice: add all exact URLs your application might use as RelayState targets

**Configuration**: Hosted SP → Advanced tab → Relay State URL List

**Interview answer**: "RelayState is validated at SP initiation time, before the AuthnRequest is sent. If the URL isn't in the SP's Relay State URL List, AM returns a 400 error immediately — the user never reaches the IdP login page. This is a security control to prevent open redirect attacks via crafted SAML flows. When debugging, if you see a 400 on the spSSOInit.jsp page, check the Relay State URL List first."

---

### Q39: Where is SAML entity configuration stored, and does it survive restarts?

**Answer**: SAML entity configuration is stored in **DS (Directory Server)** under the `am-config` backend, not in AM's memory or filesystem.

**Storage structure**:
```
ou=services,ou=am-config
├── o=techcorp,ou=services,ou=am-config       ← techcorp realm
│   └── (SAML2 entity configs: IdP, remote SP, CoT)
├── o=partner,ou=services,ou=am-config         ← partner realm
│   └── (SAML2 entity configs: SP, remote IdP, CoT)
```

**What is stored in DS**:
| Config | Stored In DS? | Survives Restart? |
|--------|---------------|-------------------|
| Entity metadata (endpoints, certs) | Yes | Yes |
| NameID Value Map | Yes | Yes |
| Account Mapper settings | Yes | Yes |
| Relay State URL List | Yes | Yes |
| Circle of Trust membership | Yes | Yes |
| Attribute Mapper config | Yes | Yes |

**When does config NOT survive?**
- If DS volume is not mounted/persisted (Docker ephemeral storage)
- If AM is using embedded DS and the container is recreated without volume

**Interview answer**: "All SAML entity configuration — metadata, NameID mappings, CoT membership, relay state lists — is stored in the DS configuration backend. As long as DS data is persisted (via external DS or mounted volumes), the configuration survives AM restarts. This is why in production we use external DS instances with replication for high availability of the configuration store."

---

### Q40: How does the AM REST authentication callback mechanism work internally?

**Answer**: AM's `/authenticate` REST endpoint uses a **stateless callback loop** where the `authId` JWT carries all tree state between requests.

**Detailed flow**:
```
Client                                    AM
  │                                        │
  │  POST /authenticate (empty body)       │
  │───────────────────────────────────────→│
  │                                        │  AM creates auth session
  │  { authId: JWT, callbacks: [           │  Encodes tree state in JWT
  │    {type: NameCallback,                │
  │     output: [{prompt: "User Name"}],   │
  │     input: [{name: IDToken1, value:""}]│
  │    },                                  │
  │    {type: PasswordCallback,            │
  │     output: [{prompt: "Password"}],    │
  │     input: [{name: IDToken2, value:""}]│
  │    }                                   │
  │  ] }                                   │
  │←───────────────────────────────────────│
  │                                        │
  │  POST /authenticate                    │
  │  { authId: JWT, callbacks: [           │  ← Must include BOTH
  │    {type: NameCallback,                │    output AND input fields
  │     output: [{prompt: "User Name"}],   │
  │     input: [{name: IDToken1,           │
  │              value: "demo"}]           │
  │    },                                  │
  │    {type: PasswordCallback,            │
  │     output: [{prompt: "Password"}],    │
  │     input: [{name: IDToken2,           │
  │              value: "secret"}]         │
  │    }                                   │
  │  ] }                                   │
  │───────────────────────────────────────→│
  │                                        │  AM decodes JWT, processes
  │  { tokenId: "SSO_TOKEN..." }           │  callbacks, authenticates
  │←───────────────────────────────────────│
```

**Critical implementation detail**: When submitting credentials, you must return the **full callback structure** including `output` fields (prompts), not just the `input` fields. AM's `RestAuthNameCallbackHandler` uses the `output.prompt` value to reconstruct the `NameCallback` object. If `output` is missing, it passes `null` to `NameCallback(prompt)` → `IllegalArgumentException: null` → HTTP 500.

**Common mistakes**:
| Mistake | Error | Fix |
|---------|-------|-----|
| Send only `input` fields, omit `output` | HTTP 500 (IllegalArgumentException: null) | Return full callback structure from step 1 |
| Send only `X-OpenAM-Username`/`Password` headers | HTTP 401 (some AM versions) | Use callback-based flow |
| Reuse expired `authId` JWT | HTTP 500 or 401 | authId expires in ~5 min, get a fresh one |

**Interview answer**: "The REST authenticate endpoint is stateless — AM encodes tree position and state in the authId JWT. Clients must echo back the full callback structure with values filled in. The output fields aren't just display hints; AM uses them internally to reconstruct Java callback objects. Stripping them causes a server error."

---

### Q41: What is the cross-domain cookie problem in SAML federation, and how do you solve it?

**Answer**: When AM (acting as SP) and the application are on different domains, the AM session cookie (`iPlanetDirectoryPro`) set after SAML SSO is not accessible to the application.

**The problem**:
```
1. SAML SSO completes → AM SP creates session
2. AM sets cookie: iPlanetDirectoryPro on domain "sso.example.com"
3. Browser redirects to app at "app.otherdomain.com"
4. App cannot read the cookie (different domain = browser blocks it)
5. App has no way to know the user is authenticated
```

**Production solutions**:

| Solution | How It Works | Pros | Cons |
|----------|-------------|------|------|
| **AM Web Agent** | Agent installed on app server intercepts requests, validates session with AM, passes user info via HTTP headers | Transparent to app, same domain | Requires agent install per app server |
| **PingGateway (IG)** | Reverse proxy in front of app, handles session validation and header injection | No app server changes, powerful routing/transformation | Extra infrastructure component |
| **Same domain** | App and AM share a parent domain (e.g., `*.example.com`) | Simplest, cookie sharing works natively | Not always possible (SaaS, multi-org) |
| **Token exchange** | After SAML SSO, exchange AM session for app-specific token (e.g., OAuth2) | Standards-based, works cross-domain | More complex flow |
| **CDSSO (Cross-Domain SSO)** | AM feature that transfers session across domains via URL parameters | Built into AM | Legacy approach, being replaced |

**Architecture 2 pattern (our lab)**:
```
Browser → AM SP (pingam:8081) → SAML SSO → AM sets cookie on pingam domain
Browser → App (localhost:3000) → App CANNOT read pingam cookie
Workaround: User manually pastes token (lab only)
Production: Web Agent or PingGateway handles this transparently
```

**Interview answer**: "The cross-domain cookie problem is fundamental to Architecture 2 SAML deployments where AM acts as the SP gateway. In production, we solve this with PingGateway or Web Agents — they sit on the same domain as the app, validate the AM session on behalf of the app, and pass authenticated user info via HTTP headers like `X-Forwarded-User`. For modern deployments, we often combine SAML federation with OAuth2 token exchange — after SAML SSO, the app gets an OAuth2 access token it can use directly, avoiding the cookie problem entirely."

---

*Add more questions as you progress through labs*
