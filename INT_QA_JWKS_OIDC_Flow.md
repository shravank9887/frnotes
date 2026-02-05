# Interview Q&A — JWKS URI & OpenID Connect Flow

---

## Part 1: JWKS URI (JSON Web Key Set URI)

### Q1: What is a JWKS URI and why does it exist?

**A:** It's a publicly accessible endpoint that exposes an authorization server's public signing keys in JWK (JSON Web Key) format. It exists so resource servers can verify JWT signatures without pre-sharing certificates. The keys are fetched on demand and cached, enabling zero-downtime key rotation.

In PingAM, the endpoint is at:
```
/am/oauth2/realms/{realm}/connect/jwk_uri
```

---

### Q2: Walk me through how a resource server validates a JWT using JWKS URI.

**A:**
1. RS reads the `kid` (Key ID) from the JWT header
2. Fetches the JWKS URI
3. Finds the matching key by `kid`
4. Uses that public key to verify the signature cryptographically
5. If valid, the claims are trusted

The RS caches the JWKS response and only re-fetches when it encounters an unknown `kid` — which signals key rotation.

---

### Q3: What's the difference between verifying tokens via JWKS URI vs token introspection?

**A:**

| | JWKS URI | Token Introspection |
|---|---|---|
| **Token type** | JWT (self-contained) | Opaque or JWT |
| **Network call** | Once (cache keys) | Every token |
| **Revocation** | Not detected until expiry | Real-time check |
| **Performance** | Fast (local verification) | Slower (HTTP per request) |
| **Use case** | Stateless APIs | When revocation matters |

Use JWKS for stateless high-throughput APIs. Use introspection when immediate revocation matters (financial, healthcare).

---

### Q4: How does key rotation work with JWKS URI?

**A:**
1. Generate new key B
2. Publish both old (A) and new (B) in JWKS
3. Start signing with key B
4. Resource servers encountering new `kid` re-fetch JWKS and find it
5. Once all tokens signed with A have expired, remove A from JWKS

Zero downtime, no coordination needed. Old tokens still verify during transition because key A is still published.

In PingAM, key rotation is configured in the OAuth2 Provider under **Signing & Encryption**.

---

### Q5: What happens if a resource server can't reach the JWKS URI?

**A:** It should use its cached copy of the keys. This is why caching is critical. If there's no cache and JWKS is unreachable, the RS cannot verify tokens and must reject them (fail-closed) or degrade gracefully depending on policy. This is a production availability concern — JWKS URI should be highly available.

---

### Q6: In PingAM, where is the JWKS URI published?

**A:** It's advertised in the OpenID Connect discovery document at `/.well-known/openid-configuration` under the `jwks_uri` field.

In the AM Console: **Realms → {realm} → Services → OAuth2 Provider → Advanced** — the JWKs URI field.

The actual endpoint:
```
http://pingam:8081/am/oauth2/realms/root/realms/techcorp/connect/jwk_uri
```

---

### Q7: Why does PingAM sign id_tokens with RS256 but access_tokens with HS256 by default?

**A:**
- **id_tokens** are meant to be verified by any relying party — RS256 (asymmetric) lets anyone verify using the public key from JWKS URI without sharing secrets.
- **access_tokens** are typically sent to resource servers that can use introspection, or if the client needs to verify them locally, the client already has the shared secret (client_secret) for HS256.
- RS256 access tokens are used when third-party resource servers need to verify without introspection.

---

### Q8: What fields are in a JWK entry and what do they mean?

**A:**

| Field | Meaning |
|-------|---------|
| `kty` | Key type — RSA, EC, oct |
| `kid` | Unique key identifier — matched against JWT header |
| `use` | Purpose — `sig` (signing) or `enc` (encryption) |
| `alg` | Algorithm — RS256, ES256, etc. |
| `n` | RSA modulus (public key component) |
| `e` | RSA exponent |

For EC keys, you'd see `x` and `y` curve coordinates instead of `n` and `e`.

Example JWKS response:
```json
{
  "keys": [
    {
      "kty": "RSA",
      "kid": "wU3ifIIaLO...",
      "use": "sig",
      "alg": "RS256",
      "n": "0vx7agoebG...",
      "e": "AQAB"
    }
  ]
}
```

---

### Q9: What's the relationship between the `kid` in a JWT header and the JWKS?

**A:** The JWT header contains `{"alg": "RS256", "kid": "abc123"}`. The `kid` tells the verifier which specific key from the JWKS to use. This enables key rotation — multiple keys coexist in JWKS, and each token declares which key signed it.

---

### Q10: In a microservices architecture, how does JWKS URI change the trust model?

**A:** Without JWKS URI, every microservice needs the signing secret or a cert copy — tight coupling, manual rotation. With JWKS URI, services only need one URL. They independently fetch and cache keys. The authorization server is the single source of truth. Adding a new service requires zero key distribution. Rotating keys requires zero coordination.

---

### Q11: Is the JWKS URI per-client or per-realm?

**A:** Per realm. All OAuth2 clients in the same realm share the same JWKS URI because the OAuth2 Provider (realm-level service) owns the signing keys.

However, clients can have their own JWKS URI for **client authentication** (`private_key_jwt`) — the client uploads its public key so AM can verify client-signed JWTs. This is the reverse direction:
- **AM's JWKS URI** — AM publishes keys so others verify tokens AM signed (one per realm)
- **Client's JWKS URI** — Client publishes keys so AM verifies client authentication JWTs (one per client)

---

## Part 2: OpenID Connect Authorization Code Flow

### Q12: What are the four roles in OIDC and how do they map to OAuth2?

**A:**

| OAuth2 Term | OIDC Term | Example | Role |
|-------------|-----------|---------|------|
| Resource Owner | End-User | The human | Logs in, grants consent |
| Client | Relying Party (RP) | Web app | Wants to know who the user is |
| Authorization Server | OpenID Provider (OP) | PingAM | Authenticates users, issues tokens |
| Resource Server | — | Backend API | Accepts access_tokens |

---

### Q13: What makes a flow OIDC instead of plain OAuth2?

**A:** The `scope=openid` parameter. Without it, AM returns only an `access_token` (OAuth2 — authorization). With `openid` in the scope, AM also returns an `id_token` (OIDC — authentication).

**One-liner:** "OAuth2 is authorization (what can you do). OIDC is authentication built on top of OAuth2 (who are you). The `openid` scope is the switch."

---

### Q14: Walk through every HTTP call in the OIDC Authorization Code flow.

**A:**

**Call 1 — Authorization Request (browser redirect to AM):**
```
GET /am/oauth2/realms/root/realms/techcorp/authorize?
  response_type=code
  &client_id=techcorp-app
  &redirect_uri=http://localhost:3000/callback
  &scope=openid profile email
  &state=random123
  &nonce=unique456
  &code_challenge=Base64URL(SHA256(verifier))
  &code_challenge_method=S256
```

| Parameter | Purpose |
|-----------|---------|
| `response_type=code` | Authorization Code flow |
| `client_id` | Identifies the RP |
| `redirect_uri` | Where AM sends the code — must match registered URI exactly |
| `scope=openid` | Triggers OIDC mode (id_token returned) |
| `scope=profile email` | Request additional claims |
| `state` | CSRF protection — random value, client verifies on return |
| `nonce` | Replay protection — embedded in id_token, client verifies |
| `code_challenge` | PKCE security |
| `code_challenge_method` | `S256` = SHA-256 |

**Call 2 — AM authenticates user, redirects back with code:**
```
HTTP 302 Location:
http://localhost:3000/callback?code=g7h8i9j0k1&state=random123
```

Client MUST verify `state` matches what it sent.

**Call 3 — Token exchange (backend server-to-server, not browser):**
```
POST /am/oauth2/realms/root/realms/techcorp/access_token
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code
&code=g7h8i9j0k1
&redirect_uri=http://localhost:3000/callback
&client_id=techcorp-app
&client_secret=T3chC0rp!
&code_verifier=original_random_string
```

**Call 4 — Token response:**
```json
{
  "access_token": "eyJ0eXAi...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "scope": "openid profile email",
  "id_token": "eyJhbGci...",
  "refresh_token": "r1s2t3u4v5"
}
```

**Call 5 — Validate id_token via JWKS URI:**
```
GET /am/oauth2/realms/root/realms/techcorp/connect/jwk_uri
```

**Call 6 — UserInfo (optional, for more claims):**
```
GET /am/oauth2/realms/root/realms/techcorp/userinfo
Authorization: Bearer eyJ0eXAi...
```

---

### Q15: What are the three tokens returned and what is each for?

**A:**

| Token | Audience | Signed With | Verified By | Purpose |
|-------|----------|-------------|-------------|---------|
| `access_token` | Resource Server (API) | HS256 (symmetric) | Introspection or shared secret | Authorization — what can the user do |
| `id_token` | Client (RP) only | RS256 (asymmetric) | JWKS URI public key | Authentication — who is the user |
| `refresh_token` | AM only | N/A | AM validates internally | Get new access_token without re-login |

**Critical:** The id_token is for the CLIENT, not the API. Never send an id_token to a resource server.

---

### Q16: What claims are in an id_token and what must the client validate?

**A:**

**Claims:**
```json
{
  "sub": "demo",              // Subject — who
  "iss": "http://pingam:..",  // Issuer — who issued it
  "aud": "techcorp-app",      // Audience — who it's for
  "exp": 1706745600,          // Expiry
  "iat": 1706742000,          // Issued at
  "auth_time": 1706741990,    // When user actually authenticated
  "nonce": "unique456",       // Replay protection
  "azp": "techcorp-app",      // Authorized party
  "at_hash": "x5b9...",       // Binds id_token to access_token
  "c_hash": "y6c0...",        // Binds id_token to authorization code
  "acr": "0",                 // Authentication context class
  "sid": "session-id..."      // Session ID
}
```

**Client MUST validate:**

| Check | Why |
|-------|-----|
| `iss` matches AM's issuer | Not from a rogue server |
| `aud` contains your `client_id` | Token is meant for you |
| `exp` > current time | Not expired |
| `nonce` matches what you sent | Not a replayed token |
| Signature valid via JWKS URI | Not tampered with |
| `at_hash` matches access_token hash | Tokens are bound together |

---

### Q17: Why is the token exchange done server-to-server and not in the browser?

**A:** Because the `client_secret` must never be exposed to the browser. JavaScript is fully inspectable by the user. The authorization code is safe to pass through the browser (it's one-time-use, short-lived, and bound to the client via PKCE), but the secret that proves client identity must stay on the backend.

---

### Q18: What is PKCE and why is it needed?

**A:** PKCE (Proof Key for Code Exchange) prevents authorization code interception attacks.

1. Client generates random `code_verifier` (43-128 chars)
2. Computes `code_challenge = Base64URL(SHA256(code_verifier))`
3. Sends `code_challenge` + `code_challenge_method=S256` in authorization request
4. Sends original `code_verifier` in token exchange
5. AM computes `SHA256(code_verifier)`, compares to stored `code_challenge`
6. Match → issue tokens. No match → reject.

An attacker can intercept the code but doesn't have the `code_verifier`, so they can't complete the exchange.

**Important:** OAuth 2.1 makes PKCE mandatory for ALL clients (public and confidential), not just mobile apps.

---

### Q19: What are the OIDC standard scopes and what claims does each return?

**A:**

| Scope | Claims Returned |
|-------|----------------|
| `openid` | `sub` only (required for OIDC) |
| `profile` | `name`, `given_name`, `family_name`, `picture`, `locale`, etc. |
| `email` | `email`, `email_verified` |
| `address` | `address` (structured object) |
| `phone` | `phone_number`, `phone_number_verified` |

OAuth2 has **no standard scopes**. OIDC defines these standard claim mappings. This is a key interview distinction.

---

### Q20: What is the UserInfo endpoint and when would you use it?

**A:** `GET /oauth2/{realm}/userinfo` with a Bearer access_token. Returns user profile claims based on granted scopes.

Use it when:
- id_token doesn't contain all the claims you need (AM may put minimal claims in id_token)
- You need fresh data (id_token claims are fixed at issuance)
- You want claims not included in the id_token by default

Don't use it if the id_token already has everything you need — it's an extra network call.

---

### Q21: What is the `state` parameter and what attack does it prevent?

**A:** `state` is a random, unguessable value the client generates and sends in the authorization request. AM returns it unchanged with the authorization code. The client MUST verify the returned `state` matches.

It prevents **CSRF attacks** — an attacker could trick a victim's browser into submitting a forged authorization request. Without `state` verification, the client would accept an attacker-controlled authorization code.

---

### Q22: What is the `nonce` parameter and how is it different from `state`?

**A:**
- **`state`** — protects the authorization request/response (CSRF). Validated by comparing the returned query parameter.
- **`nonce`** — protects the id_token (replay). Embedded inside the id_token JWT. Client validates by decoding the token and checking the claim.

`state` prevents someone from forging the redirect. `nonce` prevents someone from replaying a stolen id_token.

---

### Q23: What optional parameters can you send in the authorization request?

**A:**

| Parameter | Purpose |
|-----------|---------|
| `prompt=login` | Force re-authentication even if session exists |
| `prompt=consent` | Force consent screen even if previously consented |
| `prompt=none` | Fail if not already logged in (silent auth check) |
| `max_age=3600` | Force re-login if last auth was more than N seconds ago |
| `acr_values` | Request specific authentication level (e.g., MFA) |
| `login_hint=demo` | Pre-fill the username field |
| `ui_locales=en fr` | Preferred language for login UI |

---

### Q24: What is the `.well-known/openid-configuration` discovery endpoint?

**A:** A standardized URL that returns a JSON document describing everything about the OpenID Provider:

```
GET /am/oauth2/realms/root/realms/techcorp/.well-known/openid-configuration
```

Returns:
```json
{
  "issuer": "http://pingam:8081/am/oauth2/realms/root/realms/techcorp",
  "authorization_endpoint": "...",
  "token_endpoint": "...",
  "userinfo_endpoint": "...",
  "jwks_uri": "...",
  "scopes_supported": ["openid", "profile", "email", ...],
  "response_types_supported": ["code", "token", "id_token", ...],
  "id_token_signing_alg_values_supported": ["RS256", "PS256", ...],
  "token_endpoint_auth_methods_supported": ["client_secret_basic", "client_secret_post", ...]
}
```

A client only needs this ONE URL to discover all endpoints, supported algorithms, and JWKS URI. This is how OIDC libraries auto-configure themselves.

---

### Q25: How does OIDC differ from SAML2 for authentication?

**A:**

| | OIDC | SAML2 |
|---|---|---|
| **Protocol** | REST/JSON over HTTPS | XML over HTTP POST/Redirect |
| **Token format** | JWT (compact, URL-safe) | XML Assertion (verbose) |
| **Discovery** | `.well-known/openid-configuration` | Metadata XML |
| **Key distribution** | JWKS URI (dynamic) | Embedded in metadata XML (static) |
| **Best for** | APIs, SPAs, mobile | Enterprise SSO, legacy apps |
| **Key rotation** | Easy (JWKS URI) | Hard (re-distribute metadata) |
| **Consent** | Built-in (scopes) | Not standard |

Both achieve SSO. OIDC is the modern choice for new applications. SAML2 dominates enterprise legacy.

---

---

## Part 3: OAuth2/OIDC Application Onboarding

### Q26: A new application team wants to integrate with ForgeRock via OAuth2/OIDC. What information do you send them?

**A:** After registering their client in AM, I send them:

**1. OIDC Discovery URL** (the single most important thing):
```
https://am.company.com/am/oauth2/realms/root/realms/<realm>/.well-known/openid-configuration
```
A well-built OIDC client library only needs this to auto-configure all endpoints, supported algorithms, and JWKS URI.

**2. Client credentials** (sent securely — never via email/chat):
- Client ID: `their-app-client`
- Client Secret: `<generated>` (for confidential clients only; public clients don't get a secret)

**3. Key endpoints** (also in discovery, but often sent explicitly for convenience):

| Endpoint | URL |
|----------|-----|
| Authorization | `/am/oauth2/realms/.../authorize` |
| Token | `/am/oauth2/realms/.../access_token` |
| UserInfo | `/am/oauth2/realms/.../userinfo` |
| JWKS URI | `/am/oauth2/realms/.../connect/jwk_uri` |
| End Session (logout) | `/am/oauth2/realms/.../connect/endSession` |

**4. Token details**:
- Access token format — opaque or JWT (determines how they validate)
- ID token signing algorithm — RS256 (verify against JWKS URI)
- Token lifetimes — access_token (e.g., 3600s), refresh_token (if granted), id_token
- Available scopes — `openid`, `profile`, `email`, plus any custom scopes

**5. Claims mapping** — what claims they'll receive:
```
openid  → sub
profile → given_name, family_name, name
email   → email, email_verified
+ any custom claims mapped to DS attributes
```

**6. Realm and environment info**:
- Which realm their client is registered in
- Environment-specific URLs (dev, staging, prod)
- Cookie domain (if SSO with other apps is relevant)
- Logout behavior (front-channel, back-channel, or RP-initiated)

---

### Q27: What information do you collect FROM the application team before onboarding?

**A:** I use an intake form or kickoff meeting to collect:

**Required information:**

| Item | Why | Example |
|------|-----|---------|
| **Redirect URI(s)** | AM validates the exact redirect — prevents open redirect attacks | `https://app.company.com/callback` |
| **Post-logout redirect URI(s)** | Where to land after logout | `https://app.company.com/logged-out` |
| **Grant type(s)** | Determines the OAuth2 flow | Authorization Code, Client Credentials, etc. |
| **Scopes needed** | What user data they need | `openid profile email` |
| **Application type** | Confidential (server-side) vs Public (SPA/mobile) | Web app = confidential, SPA = public |
| **PKCE support** | Required for public clients, recommended for all | S256 method |

**Architecture questions I ask:**

| Question | Impact on AM Configuration |
|----------|---------------------------|
| Web app, SPA, or mobile? | Determines grant type and confidential vs public |
| Will it call APIs with access tokens? | Need to configure resource server (introspection or JWT validation) |
| Need refresh tokens? | Must enable refresh_token grant type on the client |
| Need silent/background token renewal? | Affects token lifetimes and `prompt=none` support |
| Multiple environments? | Separate client registrations per env with different redirect URIs |
| Need specific claims in tokens? | Custom OIDC claims script or scope-to-claim mapping |
| SSO with other apps? | Cookie domain, session sharing, same realm |
| Specific auth level required? | `acr_values` parameter, MFA tree |

**Security requirements:**

| Item | Options |
|------|---------|
| Token endpoint auth method | `client_secret_basic` (header), `client_secret_post` (body), `private_key_jwt` (most secure) |
| Consent required? | Implied consent for internal apps, explicit for third-party |
| Session requirements | SSO, single logout, max session time |
| IP restrictions | Restrict token endpoint to specific CIDRs (via policy or IG) |

---

### Q28: Walk through the steps you perform in AM Console to onboard an OAuth2/OIDC client.

**A:**

**Pre-requisite**: OAuth2 Provider service must exist in the realm.

**Step 1 — Register the client:**
Realms → /{realm} → Applications → OAuth 2.0 → Clients → Add Client
- Client ID, Client Secret (auto-generate or provided)

**Step 2 — Core tab:**
- **Redirect URIs** — exact list from app team (no wildcards in production)
- **Scopes** — only what they requested (least privilege)
- **Default Scopes** — scopes granted without explicit request (usually just `openid`)

**Step 3 — Advanced tab:**
- **Grant Types** — only what's needed (e.g., Authorization Code only — don't enable all)
- **Response Types** — `code` for auth code flow
- **Token Endpoint Auth Method** — `client_secret_basic` or `private_key_jwt`
- **Implied Consent** — ON for internal trusted apps, OFF for third-party

**Step 4 — Overrides (if needed):**
- **Access Token Lifetime** — per-client override (e.g., 300s for sensitive APIs)
- **Refresh Token Lifetime** — if refresh tokens are enabled
- **ID Token Signing Algorithm** — usually inherit from OAuth2 Provider (RS256)

**Step 5 — Verify:**
```bash
# Test the client can get tokens
curl -s -X POST "https://am.company.com/am/oauth2/realms/root/realms/{realm}/access_token" \
  -d "grant_type=client_credentials" \
  -d "client_id=their-app-client" \
  -d "client_secret=<secret>" \
  -d "scope=openid"
```

**Step 6 — Send credentials securely** to the app team (vault, encrypted channel — never plaintext email).

---

### Q29: How do you decide between confidential and public client types?

**A:**

| | Confidential Client | Public Client |
|---|---|---|
| **Can keep a secret** | Yes — server-side code | No — browser JS or mobile app |
| **Examples** | Java backend, Node.js server, Python API | React SPA, Angular app, iOS/Android app |
| **Auth method** | client_secret_basic, client_secret_post, private_key_jwt | None (no secret) |
| **PKCE** | Recommended | **Required** |
| **Refresh tokens** | Yes — stored securely on server | Careful — if granted, use rotation + short lifetime |
| **AM setting** | Default | Set "Public Client" toggle ON |

**Key rule**: If the client runs in the browser or on a user's device, it's public — the secret can be extracted. If it runs on a server you control, it's confidential.

**Interview-ready answer**: "When onboarding a SPA, I register it as a public client with no client_secret, enforce PKCE with S256, and use the Authorization Code flow. For server-side web apps, I register a confidential client with `client_secret_basic` authentication. For service-to-service (no user), I use a confidential client with Client Credentials grant. The decision always starts with: can this application keep a secret?"

---

### Q30: What are common mistakes when onboarding OAuth2/OIDC applications?

**A:**

| Mistake | Consequence | Fix |
|---------|-------------|-----|
| Enabling all grant types | Widens attack surface unnecessarily | Only enable the specific grant type needed |
| Wildcard redirect URIs | Open redirect vulnerability — attacker can steal authorization codes | Use exact redirect URIs, no wildcards |
| Sharing client_secret via email | Secret compromised in transit | Use vault, encrypted channel, or rotate after first use |
| Not enforcing PKCE for public clients | Authorization code interception attacks | Enable PKCE requirement on the client |
| Granting excessive scopes | App gets more user data than needed | Principle of least privilege — only approved scopes |
| Same client across environments | Prod secret exposed in dev logs | Separate client registrations per environment |
| Not setting token lifetimes | Default may be too long for sensitive apps | Set per-client overrides matching security requirements |
| Skipping post-logout redirect validation | Open redirect after logout | Configure allowed post-logout redirect URIs |
| Using Implicit grant for SPAs | Tokens exposed in URL fragment, no refresh | Use Authorization Code + PKCE instead (OAuth 2.1 deprecates Implicit) |

**Interview-ready answer**: "The most critical onboarding mistakes I watch for are: enabling unnecessary grant types, using wildcard redirect URIs, and not enforcing PKCE for public clients. I treat client registration like a firewall rule — only open what's explicitly needed. I also ensure every environment (dev/staging/prod) has its own client registration with separate credentials, so a dev leak doesn't compromise production."

---

*Updated: 2026-02-04 — Added Q26-Q30 covering OAuth2/OIDC application onboarding: what to send, what to collect, AM Console steps, confidential vs public clients, common mistakes.*
