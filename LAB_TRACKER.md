# ForgeRock Interview Practice Labs - Progress Tracker

## Environment
- **AM Console**: http://pingam:8081/am/console
- **IDM Admin**: http://pingidm:8082/admin/ (openidm-admin / openidm-admin)
- **Mailpit UI**: http://localhost:8025
- **Admin**: amadmin / changeit
- **DS Manager**: cn=Directory Manager / Passw0rd123
- **Practice Realm**: /techcorp (alias: techcorp.example.com)

### Docker Compose Files
| File | Services | Purpose |
|------|----------|---------|
| `docker-compose.yaml` | pingam, pingds | Core AM + DS (DO NOT MODIFY) |
| `docker-compose.idm.yaml` | pingidm, pingds-idm | IDM + dedicated DS repo |
| `docker-compose.mailpit.yaml` | mailpit | SMTP mock for email testing |
| `docker-compose.gw.yaml` | pinggw, sample-app | PingGateway + sample application |

### Interview Q&A File Rules
- Max **15 questions per file**. Exceeding → create sequel: `INT_QA_<topic>_2.md`
- Keep answers concise. Break large concepts into multiple focused questions.
- Existing files grandfathered (don't split retroactively).

---

## Lab Progress

| Lab | Topic | Status | Notes |
|-----|-------|--------|-------|
| 1 | Realm Architecture & DS Storage | ✅ Completed | Created /techcorp realm |
| 2 | Authentication Trees/Journeys | ✅ Completed | Created TechCorpLogin & TechCorpSecureLogin |
| 3 | OAuth2/OIDC Provider Setup | ✅ Completed | OAuth2 Provider, Client Credentials, Authorization Code, token analysis |
| 4 | SAML2 Federation | ✅ Completed | IdP + SP configured, SP-initiated & IdP-initiated SSO working, sample-app integration complete |
| 5 | Policy-Based Authorization | ✅ Completed | Resource Types, Policy Sets, 3 Policies, REST evaluation, time-based deny |
| 6 | User Self-Service (Registration/Password Reset) | 🔧 In Progress | Email Service configured, IDM required for tree-based self-service |
| 6a | PingIDM Installation & Setup | 🔧 In Progress | Separate compose: pingidm + pingds-idm |
| 7 | Social Authentication (Google/Facebook) | ⏭️ Skipped | Requires external Google OAuth2 credentials; concepts understood from research |
| 8 | MFA with OATH/WebAuthn | 🔧 Partial | OATH TOTP working; WebAuthn blocked by HTTP (needs HTTPS) |
| 9 | Session Management & CTS | ✅ Completed | CTS architecture, stateful vs stateless, session lifecycle, quotas, timeouts, denylisting, client-side security |
| 10 | Amster Scripting & Automation | ⏳ Planned | |
| 13 | Amster CLI — Hands-On Config Management | ✅ Completed | Amster connect, export, import, scripts, expressions, FBC/upgrade concepts |
| 14 | AM Upgrade Process — Complete Walkthrough | ✅ Completed | Full upgrade walkthrough: assessment, backup, execution, verification, rollback |
| 15 | Web Agents, Java Agents & PEP | ✅ Completed | Agent types, PEP enforcement, PingGateway comparison, SSO, production patterns |
| 11 | Custom Authentication Nodes (Research) | ✅ Completed | Research: API, Amster, upgrades (Session 11) |
| 12 | Custom Authentication Nodes (Hands-On) | ✅ Completed | Built & deployed 7 nodes to PingAM 8.0.2 |
| 16 | PingGateway (Identity Gateway) | 🔧 In Progress | Built & running on :8083. Routes 01-04 all loading. IG agent registered. SSO goto redirect blocked by AM XUI issue. |

---

## Current Session Context

### Current Lab: Lab 4 - SAML2 Federation (In Progress)

**Objective**: Configure SAML2 IdP + SP federation across realms, integrate with sample-app

**SAML2 Entity Reference**:
- **IdP Entity ID**: `techcorp-idp` (hosted in `/techcorp` realm, meta alias: `/techcorp/idp`)
- **SP Entity ID**: `partner-sp` (hosted in `/partner` realm, meta alias: `/partner/partner-sp`)
- **Circle of Trust**: `techcorp-cot` (in both realms)
- **IdP Metadata URL**: `http://pingam:8081/am/saml2/jsp/exportmetadata.jsp?entityid=techcorp-idp&realm=/techcorp`
- **SP Metadata URL**: `http://pingam:8081/am/saml2/jsp/exportmetadata.jsp?entityid=partner-sp&realm=/partner`
- **SP-Initiated SSO**: `http://pingam:8081/am/saml2/jsp/spSSOInit.jsp?metaAlias=/partner/partner-sp&idpEntityID=techcorp-idp&binding=urn:oasis:names:tc:SAML:2.0:bindings:HTTP-POST`
- **IdP-Initiated SSO**: `http://pingam:8081/am/saml2/jsp/idpSSOInit.jsp?metaAlias=/techcorp/idp&spEntityID=partner-sp&binding=urn:oasis:names:tc:SAML:2.0:bindings:HTTP-POST`
- **Sample App**: `http://localhost:3000` (Node.js, Architecture 2 - AM as SP gateway)
- **Identity Store**: `ou=people,ou=identities` (shared between both realms)

**OAuth2 Client Reference (from Lab 3)**:
- **Client ID**: `techcorp-app`
- **Client Secret**: `T3chC0rp!`
- **Redirect URI**: `http://localhost:3000/callback`
- **Scopes**: `openid`, `profile`, `email`
- **Grant Types**: Authorization Code, Client Credentials (+ others)
- **Implied Consent**: Enabled

---

## Quick Reference Commands

### Check Auth Trees in DS
```bash
docker exec pingds bash -c '/opt/opendj/bin/ldapsearch \
  --hostname localhost --port 1636 --useSsl --trustAll \
  --bindDN "cn=Directory Manager" --bindPassword "Passw0rd123" \
  --baseDN "ou=authenticationTreesService,ou=services,o=techcorp,ou=services,ou=am-config" \
  --searchScope sub "(objectClass=*)" dn'
```

### Test Authentication via REST
```bash
curl -X POST "http://pingam:8081/am/json/realms/root/realms/techcorp/authenticate" \
  -H "Content-Type: application/json" \
  -H "Accept-API-Version: resource=2.0, protocol=1.0"
```

### OAuth2 Client Credentials Flow
```bash
curl -s -X POST "http://pingam:8081/am/oauth2/realms/root/realms/techcorp/access_token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=client_credentials" \
  -d "client_id=techcorp-app" \
  -d "client_secret=T3chC0rp!" \
  -d "scope=profile"
```

### Token Introspection
```bash
curl -s -X POST "http://pingam:8081/am/oauth2/realms/root/realms/techcorp/introspect" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "token=<ACCESS_TOKEN>" \
  -d "client_id=techcorp-app" \
  -d "client_secret=T3chC0rp!"
```

### OAuth2 Authorization Code Flow (via REST/Postman)
```bash
# Step 1: Authenticate user (get tokenId)
curl -s -X POST "http://pingam:8081/am/json/realms/root/realms/techcorp/authenticate" \
  -H "Content-Type: application/json" \
  -H "Accept-API-Version: resource=2.0, protocol=1.0" \
  -H "X-OpenAM-Username: demo" \
  -H "X-OpenAM-Password: <password>"

# Step 2: GET /authorize with session cookie (grab csrf from HTML response)
curl -s "http://pingam:8081/am/oauth2/realms/root/realms/techcorp/authorize?response_type=code&client_id=techcorp-app&redirect_uri=http%3A%2F%2Flocalhost%3A3000%2Fcallback&scope=openid%20profile%20email&state=abc123" \
  -H "Cookie: iPlanetDirectoryPro=<TOKENID>" | grep csrf

# Step 3: POST consent with decision=allow (get code from Location header)
curl -v -d "response_type=code&client_id=techcorp-app&redirect_uri=http://localhost:3000/callback&scope=openid%20profile%20email&state=abc123&csrf=<CSRF>&decision=allow" \
  -H "Cookie: iPlanetDirectoryPro=<TOKENID>" \
  "http://pingam:8081/am/oauth2/realms/root/realms/techcorp/authorize"

# Step 4: Exchange code for tokens
curl -s -X POST "http://pingam:8081/am/oauth2/realms/root/realms/techcorp/access_token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=authorization_code&code=<CODE>&redirect_uri=http://localhost:3000/callback&client_id=techcorp-app&client_secret=T3chC0rp!"
```

### View OAuth2 Tokens in CTS (DS)
```bash
docker exec pingds /opt/opendj/bin/ldapsearch \
  --hostname localhost --port 1636 --useSsl --trustAll \
  --bindDN "cn=Directory Manager" --bindPassword "Passw0rd123" \
  --baseDN "ou=famrecords,ou=openam-session,ou=tokens,ou=am-config" \
  --searchScope sub "(objectClass=frCoreToken)"
```

---

## Interview Topics Checklist

### Authentication
- [x] Trees vs Chains (when to use each)
- [x] Node types and callbacks
- [x] Inner trees / shared trees (concept covered)
- [x] Tree debugging and troubleshooting
- [ ] Zero page login

### Authorization
- [x] Authorization model (Resource Types → Policy Sets → Policies)
- [x] PDP vs PEP vs PAP vs PIP architecture
- [x] Resource Type design (scoped per-app vs broad default)
- [x] Policy sets and applications (TechCorpAPI policy set)
- [x] Policy subjects (Identity/group, Authenticated Users)
- [x] Policy evaluation via REST API (`/policies?_action=evaluate`)
- [x] Implicit deny behavior (no matching policy = deny)
- [x] Explicit deny with deny-overrides combining
- [x] Multiple policy matching (resource matches overlapping policies)
- [x] Environment conditions (SimpleTime for business hours, IPv4, AuthLevel, Script)
- [x] Time-based deny policy (DenyWriteOutsideBusinessHours)
- [x] OAuth2 Scope Resource Type vs URL Resource Type
- [ ] Entitlements vs Policies
- [ ] Response attributes

### Federation
- [x] SAML2 concepts (IdP, SP, assertions, CoT, metadata, bindings, NameID)
- [x] SAML2 SP-initiated SSO flow (tested and debugged)
- [x] SAML2 IdP-initiated SSO flow (tested)
- [x] SAML2 metadata (examined IdP + SP metadata XML, endpoints, certificates)
- [x] SAML2 MetaAlias configuration (debugged double-slash issue)
- [x] SAML2 Circle of Trust (cross-realm trust configuration)
- [x] SAML2 Remote entity registration (cross-realm metadata import)
- [x] SAML2 NameID formats (persistent vs transient vs unspecified)
- [x] SAML2 NameID Value Map (IdP: maps format → user attribute)
- [x] SAML2 Account Mapper (SP: maps NameID → local user)
- [x] SAML2 Attribute Mapper (IdP sends attributes, SP receives them)
- [x] SAML2 RelayState validation (Relay State URL List on SP, strict vs prefix matching)
- [x] SAML2 Architecture patterns (built-in SP, AM as SP gateway, federation hub)
- [x] SAML2 debugging (Federation debug log, common errors)
- [x] SAML2 Complete NameID format list (8 formats: persistent, transient, kerberos, emailAddress, unspecified, X509SubjectName, WindowsDomainQualifiedName, encrypted)
- [ ] SAML2 Single Logout (SLO)
- [ ] SAML2 Assertion encryption/signing
- [x] OAuth2/OIDC - Client Credentials flow
- [x] OAuth2/OIDC - Authorization Code flow (browser + REST/Postman)
- [x] OAuth2/OIDC - OIDC ID tokens (decoded id_token, understood claims)
- [x] OAuth2/OIDC - access_token vs id_token differences (HS256 vs RS256)
- [x] OAuth2/OIDC - Authorization Code flow via REST (3-step: auth → GET /authorize → POST consent → exchange code)
- [x] OAuth2/OIDC - JWKS URI (key publishing, rotation, verification, per-realm vs per-client)
- [x] OAuth2/OIDC - Full OIDC Authorization Code flow (all parameters, all calls, roles, PKCE, nonce, state)
- [x] OAuth2/OIDC - OIDC Discovery endpoint (`.well-known/openid-configuration`)
- [x] OAuth2/OIDC - OIDC standard scopes (openid, profile, email, address, phone → claims mapping)
- [x] OAuth2/OIDC - UserInfo endpoint
- [x] OAuth2/OIDC - OIDC vs SAML2 comparison
- [x] OAuth2/OIDC - Application onboarding process (what to send, what to collect, AM Console steps)
- [x] OAuth2/OIDC - Confidential vs public clients (PKCE, grant type selection)
- [x] OAuth2/OIDC - Common onboarding mistakes (wildcard redirects, excessive scopes, implicit grant)
- [ ] STS (Security Token Service)

### Token Management
- [x] CTS architecture (ou=tokens, ou=famrecords structure)
- [x] Session token types (authId vs tokenId)
- [x] CTS reaper configuration (Global Services → Session → CTS, Advanced properties, DS-side cleanup)
- [x] DS replication (deployment ID model, bootstrapReplicationServer, dsrepl status)
- [x] DS replication topology design (3-tier: config/identity/CTS, cross-DC, affinity routing)
- [x] DS replication troubleshooting (changelog exhaustion, split brain, replication delay)
- [ ] Token blacklisting
- [x] Stateless vs stateful sessions (your env uses stateless)
- [x] Stateless vs stateful OAuth2 tokens (Use Client-Side Access Tokens setting)
- [x] Token introspection endpoint
- [x] Opaque vs JWT access tokens
- [x] CTS token storage location: `ou=famrecords,ou=openam-session,ou=tokens,ou=am-config`
- [x] AM identity subject prefixes (age!, usr!, grp!)

### Identity Management (IDM)
- [x] PingIDM vs PingAM — purpose and architecture differences
- [x] IDM repository — dedicated DS with `idm-repo` profile
- [x] DS setup profiles (am-config, am-cts, am-identity-store, idm-repo)
- [x] Two-DS architecture — IDM repo vs AM identity store
- [x] managed/user vs LDAP users — canonical vs authentication identity
- [x] Reconciliation concepts (full recon, LiveSync, implicit sync)
- [x] LDAP connector — IDM reads/writes AM's DS
- [x] Link table — maps managed objects to external resources
- [x] Config overlay pattern — JSON file-based configuration
- [x] AM-IDM integration nodes (Platform Username/Password, Create Object)
- [x] Legacy vs tree-based self-service (standalone AM vs AM+IDM)
- [x] LDAP connector configuration
- [x] ICF object types: `__ACCOUNT__` (inetOrgPerson) vs `account` (POSIX) — must use `__ACCOUNT__` for AM users
- [x] Mapping creation: `systemLdap__ACCOUNT__managedUser` (LDAP → managed/user)
- [x] Situation policies (Behaviors tab) — read-only default must be changed to enable create/update
- [x] Persist Associations — saves link records for ongoing sync (must enable for real recon)
- [x] Managed object mappings — attribute grid (uid→userName, sn→sn, mail→mail, givenName→givenName) ✅
- [x] LDAP array-to-string transforms — `source[0]` on all 4 properties (LDAP returns arrays, managed/user expects strings) ✅
- [x] Fixed missing givenName on bob/demo LDAP entries (data quality issue) ✅
- [x] First reconciliation succeeded — 3 LDAP users synced to managed/user ✅
- [x] AM-IDM integration — IDM Provisioning global service configured (URL: http://pingidm:8082, path: openidm) ✅
- [x] IDM Provisioning signing/encryption keys configured (selfservice, openidm-selfservice-key, HS256) ✅
- [x] OAuth2 Provider created in techcorp realm ✅
- [x] idm-provisioning OAuth2 client created in techcorp realm (client_credentials grant, fr:idm:* scope) ✅
- [x] Root realm idm-provisioning agent — added client_credentials grant type via ldapmodify ✅
- [x] useInternalOAuth2Provider enabled ✅
- [x] TechCorpRegistration tree created (Page Node → Create Object) ✅
- [ ] User registration tree — BLOCKED: OAuth2 grant type auth error (internal OAuth2 provider vs idm-provisioning client). Parked for now.
- [ ] Forgot password tree
- [ ] IDM workflows and schedules

### Custom Authentication Nodes
- [x] Node interface — `process(TreeContext)` returns `Action`
- [x] AbstractDecisionNode — boolean outcomes, `goTo(true/false)`
- [x] SingleOutcomeNode — single exit path, `goToNext()`
- [x] Implementing Node directly — multi-outcome with custom OutcomeProvider
- [x] Config interface with `@Attribute` — admin-configurable properties in tree designer
- [x] Enum config attributes — rendered as dropdown in UI
- [x] Set<String> config attributes — multi-value input
- [x] Default vs required config — `default` keyword makes properties optional
- [x] `@Node.Metadata` — outcomeProvider, configClass, tags, i18nFile
- [x] `@Inject` + `@Assisted` — Guice DI, per-instance Config from DS
- [x] TreeContext — sharedState, transientState, request (headers, clientIp, cookies)
- [x] Action — `goTo()`, `replaceSharedState()`, `putSessionProperty()`
- [x] ExternalRequestContext — reading HTTP headers (ListMultimap), X-Forwarded-For parsing
- [x] Shared state enrichment pattern — copy, modify, replaceSharedState
- [x] Session properties — written into AM session for policy evaluation
- [x] Custom OutcomeProvider — multi-outcome nodes (4 outcomes: low/medium/high/error)
- [x] Plugin class — `AbstractNodeAmPlugin`, `getNodesByVersion()`, `getPluginVersion()`
- [x] `META-INF/services/org.forgerock.openam.plugins.AmPlugin` — ServiceLoader registration
- [x] Properties file (i18n) — UI labels, help text, `nodeDescription`/`nodeHelp`
- [x] Plugin versioning — adding new nodes requires version bump or AM skips `onInstall()`
- [x] Maven project setup — extracting JARs from container, `mvn install:install-file`
- [x] All dependencies as `<scope>provided</scope>` — AM supplies at runtime
- [x] Key JARs: auth-node-api, openam-annotations, openam-plugin-framework, forgerock-util
- [x] Build → deploy → restart cycle — `mvn package` → `docker cp` → `docker restart`
- [x] `javax.inject` vs `jakarta.inject` — PingAM 8 uses Tomcat 10 (Jakarta EE)
- [x] Hands-on: Built 7 nodes (BusinessHours, HeaderCheck, RiskLevelRouter, IpWhitelist, AuditLog, GeoRouter + original 3)

### Amster & Configuration Management
- [x] Amster CLI — connect, export-config, import-config, CRUD commands
- [x] Amster authentication — password (-i) vs RSA private key (-k) for CI/CD
- [x] AM FQDN requirement — must connect using AM's configured Site URL
- [x] Export structure — global/ (server-wide) + realms/ (per-realm, flattened naming)
- [x] JSON file anatomy — metadata (address) + data (REST body) + _type (singleton vs collection)
- [x] Singleton services vs collections vs node instances (UUID-named)
- [x] Tree export structure — tree wiring (AuthTree/) + node configs (UUID dirs)
- [x] Config sections map to AM Console tabs
- [x] What's exported (config) vs not (users, sessions, tokens, secrets)
- [x] import-config idempotency — creates if missing, updates if exists, safe on live AM
- [x] Dev workflow — Console changes → export → git diff → commit → CI/CD import
- [x] Amster scripts (.amster files) — Groovy, non-interactive, CI/CD pipelines
- [x] Configuration expressions — `&{variable|default}` for environment-specific config
- [x] FBC evolution — AM 6 files → AM 7 DS → AM 7.4+ FBC (JSON filesystem)
- [x] FBC vs DS-based + Amster — immutable cloud-native vs traditional deployments
- [x] Amster for upgrades — pre-upgrade backup, cross-version import, rollback strategy
- [x] DS schema upgrade is one-way — must restore from LDIF backup to rollback
- [x] FBC deep dive — enabling FBC mode, JVM properties, directory structure, boot.json
- [x] FBC vs DS-based vs Amster — when to use each (dev/traditional/K8s)
- [x] FBC + Docker production pattern — baking config into images, immutable containers
- [x] FBC limitations — read-only runtime, Console view-only, restart required
- [x] FBC + expressions — `&{ENV_VAR|default}` for multi-environment deployment
- [x] Amster RSA key auth — CI/CD non-interactive authentication
- [x] Amster dependency ordering — global → realms → services → collections → trees → policies
- [x] Secret management in FBC/Amster — expressions + Vault/K8s Secrets, never plaintext in Git
- [x] ForgeOps/CDK relationship — Amster export → FBC → Docker → K8s
- [x] UUID handling in tree promotion — export as-is, import creates/updates by UUID
- [x] Emergency fix patterns in FBC — fast-track pipeline, rollback, escape hatch to DS-based

### Upgrades
- [x] Upgrade phases: assessment → prepare → execute → verify → rollback plan
- [x] Upgrade order: DS first → AM second → custom nodes → IDM last
- [x] Breaking changes per version (AM 6→7 major, 7→7.5 minor, 7.5→8 moderate)
- [x] Custom nodes — highest risk, javax→jakarta for AM 8, must recompile
- [x] SAML federation — certificate changes break trust, must coordinate with partners
- [x] Three backup types: Amster export + DS LDIF + DS binary (each covers different failure)
- [x] In-place vs blue-green upgrade (zero downtime, instant rollback)
- [x] DS schema upgrade is one-way — irreversible, must backup first
- [x] Post-upgrade verification checklist (trees, OAuth2, SAML, policies, custom nodes)
- [x] Rollback scenarios (AM broke, DS broke, custom nodes broke, blue-green switch)
- [x] OAuth2 silent breakage risk — token format/signing changes break downstream APIs
- [x] IDM upgrade ordering — depends on AM, separate DS schema upgrade
- [x] Kubernetes upgrades — FBC + rolling updates + kubectl rollout undo

### Agents & Policy Enforcement
- [x] Web Agents — native C/C++ modules for Apache, Nginx, IIS
- [x] Java Agents — servlet filters for Tomcat, JBoss, WebLogic
- [x] PEP role — agents enforce AM's PDP decisions (session + policy)
- [x] Agent registration — agent profiles stored in AM, centralized vs local config
- [x] HTTP header injection — X-AM-Username, X-AM-Groups, session properties
- [x] SSO via shared session cookie across agents
- [x] Cross-Domain SSO (CDSSO) — redirect-based for different domains
- [x] Not-Enforced URLs — excluding health checks, public pages
- [x] Agent failure modes — fail-closed vs fail-open vs cache-degraded
- [x] PingGateway (IG) — standalone reverse proxy replacing agents
- [x] Agents vs OAuth2 — traditional (cookie/session) vs modern (token-based)
- [x] Agent → OAuth2 migration trend — legacy apps use agents, new apps use OIDC

### PingGateway (Identity Gateway)
- [x] PingGateway architecture — standalone reverse proxy, Jetty-based
- [x] Route-based JSON configuration — auto-reload, condition/handler/heap
- [x] AmService heap object — connection to AM with IG agent credentials
- [x] SingleSignOnFilter — SSO replacing Web Agents
- [x] OAuth2ResourceServerFilter — API protection via token introspection
- [x] PolicyEnforcementFilter — PEP calling AM's PDP
- [x] IG agent registration in AM Console
- [x] PingGateway vs Web Agent vs Java Agent (architectural decision)
- [x] PasswordReplayFilter — legacy app SSO
- [x] CDSSO with PingGateway — CrossDomainSingleSignOnFilter
- [x] CaptureDecorator for debugging
- [x] Production patterns — HA, secrets, monitoring, GitOps
- [x] Microservices gateway pattern
- [x] Hands-on: Build and start PingGateway container
- [x] Hands-on: Simple proxy route verified (01-simple-proxy with UriPathRewriteFilter)
- [x] Hands-on: Headers route verified (04-headers — StaticResponseHandler)
- [x] PingGateway 2025.11.1 secretsProvider requirement — inline passwords deprecated, must use Base64EncodedSecretStore
- [x] UriPathRewriteFilter — strip path prefix when proxying to backend
- [x] Admin API localhost-only restriction — 403 from outside container, works via docker exec
- [x] No UI — PingGateway is headless, config via JSON files + REST API only
- [x] Route auto-reload — config changes picked up within 10 seconds without restart
- [x] admin.json changes require container restart (not auto-reloaded)
- [x] Core concepts: Handlers (ReverseProxy, Static, Chain, Dispatch, Scriptable, Router)
- [x] Core concepts: Filters (SSO, OAuth2RS, PolicyEnforcement, Header, Scriptable, Throttling, Conditional, PasswordReplay)
- [x] Core concepts: Heap objects — shared dependencies (AmService, JwtSession, CaptureDecorator)
- [x] Core concepts: Expression language — ${request.*}, ${contexts.*}, ${env[*]}
- [x] Core concepts: Context chain — how filters populate data for downstream use
- [x] Core concepts: Route evaluation order — alphabetical, first match wins
- [x] Core concepts: ScriptableFilter — Groovy scripts, next.handle(), short-circuit with Response
- [x] Core concepts: Enterprise project structure — Gradle build, src/main/resources, deployer/overlays (Kustomize)
- [x] Hands-on: Register IG agent in AM (ig_agent in /techcorp, Token Introspection: Realm)
- [x] All 4 routes loading after ig_agent registration + restart
- [ ] Hands-on: Test SSO route — BLOCKED: AM XUI ignores goto query param after login
- [ ] Hands-on: Test OAuth2 RS token introspection
- [x] loginEndpoint configuration in SingleSignOnFilter
- [x] AM FQDN validation and cookie domain behavior
- [x] `_ig=true` callback marker — how SingleSignOnFilter recognizes post-login callbacks

### Self-Service
- [x] Email Service configuration (SMTP transport as secondary config)
- [x] Transport Type pattern (reference to secondary configuration)
- [ ] User registration flow (AM tree → IDM → DS)
- [ ] Forgot password flow (email verification + reset)
- [ ] Profile management

---

## Session Log

### Session 2 - 2026-01-28
**Starting Point**: Lab 2 - Authentication Trees/Journeys
**Goal**: Complete Lab 2, understand authentication trees for interview prep

**Progress**:
- [x] Create test users (demo, alice, bob) in techcorp realm ✅ Already done
- [x] Create `TechCorpLogin` tree (simple username/password) ✅
- [x] Test tree via XUI and REST API ✅ (captured callback flow)
- [x] View tree configuration in DS ✅ (found actual DN path)
- [x] Create `TechCorpSecureLogin` tree (with retry logic) ✅
- [x] Review interview Q&A for authentication trees ✅

**Key Learnings**:
- Callback mechanism (authId JWT for state, tokenId for session)
- Tree storage in DS: `ou={TreeName},...,ou=authenticationTreesService,...`
- Retry Limit Decision node for attempt limiting
- Page Node to group collectors
- CTS structure: `ou=tokens` → `ou=openam-session` → `ou=famrecords`
- **Stateless sessions**: tokenId contains encrypted session (not stored in CTS)
- **Token format**: Not JWT - AM proprietary format (handle + encrypted blob)
- **Encryption**: Symmetric (AES) - same key across AM cluster
- **Security**: Attacker can USE stolen token but cannot READ it

**Created**: INTERVIEW_QUESTIONS.md with Q&A (14 questions covering auth trees, sessions, tokens, encryption)

### Session 3 - 2026-01-29
**Starting Point**: Lab 3 - OAuth2/OIDC Provider Setup
**Goal**: Configure OAuth2 provider, test grant flows, understand token storage

**Progress**:
- [x] Configure OAuth2 Provider service in techcorp realm (via AM Console) ✅
- [x] Create OAuth2 client `techcorp-app` ✅
- [x] Test Client Credentials flow ✅ (got opaque access token)
- [x] Introspect access token ✅ (verified realm, client_id, scope, expiry)
- [x] Explore CTS token storage in DS ✅ (found correct base DN)
- [x] Examine all CTS token entries ✅ (SESSION + OAUTH types)
- [x] Test stateless JWT token ✅ (decoded JWT, saw OAUTH2_STATELESS_GRANT)
- [x] Test Authorization Code flow via browser ✅ (got code in redirect URL)
- [x] Test Authorization Code flow via REST/Postman ✅ (3-step: auth → consent GET → consent POST)
- [x] Exchange authorization code for tokens ✅ (access_token + id_token)
- [x] Decode and analyze id_token claims ✅
- [x] Test with OIDC (openid scope, id_token) ✅

**Key Learnings**:
- OAuth2 Provider created via AM Console → Services → Add Service → OAuth2 Provider
- Client Credentials = 2-legged flow (no user), returns access token only
- Introspection shows: `active`, `scope`, `realm`, `client_id`, `exp`, `sub`
- CTS base DN in this env: `ou=famrecords,ou=openam-session,ou=tokens,ou=am-config`
- `coreTokenType: OAUTH` for OAuth2 tokens vs `coreTokenType: SESSION` for AM sessions
- "Use Client-Side Access Tokens" = stateless toggle (JWT vs opaque)
- Stateless JWT contains `cts: OAUTH2_STATELESS_GRANT` claim
- AM stores CTS record even for stateless tokens (for revocation)
- `sub` claim uses identity prefixes: `(age!techcorp-app)` for agents
- Authorization Code flow via REST requires 3 steps: authenticate user → GET /authorize (get csrf) → POST /authorize with decision=allow (get code)
- `id_token` signed with RS256 (asymmetric) vs access_token signed with HS256 (symmetric)
- id_token contains: `sub`, `aud`, `azp`, `auth_time`, `sid`, `acr`, `at_hash`, `c_hash`
- `at_hash` binds id_token to access_token, `c_hash` binds to authorization code
- Authorization codes are single-use and short-lived (~120 seconds)
- Implied Consent setting skips consent in browser but still shows consent page via REST (need POST with decision=allow + csrf)

### Session 4 - 2026-01-30
**Starting Point**: Lab 4 - SAML2 Federation (continuing from previous session debugging)
**Goal**: Fix SP-initiated SSO, configure full SAML2 federation, integrate sample-app

**Progress**:
- [x] Diagnosed metaAlias double-slash bug (`/techcorp//techcorp-idp`) from exported metadata ✅
- [x] Deleted and recreated hosted IdP with correct meta alias (`/idp` → full path `/techcorp/idp`) ✅
- [x] Re-imported remote IdP entity in partner realm with corrected metadata ✅
- [x] Tested SP-initiated SSO → "Single Sign-on succeeded" ✅
- [x] Tested IdP-initiated SSO → "Single Sign-on succeeded" ✅
- [x] Built and deployed sample-app (Node.js, port 3000) ✅
- [x] Fixed sample-app internal vs external URL issue (AM_INTERNAL:8080 vs AM_EXTERNAL:8081) ✅
- [x] Fixed SP metaAlias in sample-app (`/partner/partner-sp`) ✅
- [x] Debugged "Invalid Relay State URL" error → added `http://localhost:3000` to SP Relay State URL List ✅
- [x] Debugged "No local user being mapped" error → configured NameID Value Map on IdP ✅
- [x] Configured IdP NameID Value Map: `unspecified=cn`, `emailAddress=mail` ✅
- [x] Configured SP NameID Format List: `unspecified` first ✅
- [x] Configured SP "Use Name ID as User ID" = ON ✅
- [x] Debugged "Unable to generate NameID value" → `uid` attribute not readable via AM identity API, switched to `cn` ✅
- [x] Verify end-to-end sample-app SAML flow ✅ (cn-based NameID fix confirmed working)
- [x] Fixed "Invalid Relay State URL" for sample-app ✅ (added exact /protected path to SP Relay State URL List)
- [x] Sample-app login → SAML SSO → /protected page working ✅
- [ ] Examine SAML assertion content (optional - use SAML tracer browser extension)
- [ ] Test with SAML tracer browser extension (optional)

**Key Learnings**:
- **MetaAlias**: In sub-realm `/techcorp`, set meta alias to `/idp` (not `/techcorp-idp`). AM prepends realm path → `/techcorp/idp`. Using `/techcorp-idp` causes double-slash: `/techcorp//techcorp-idp`
- **Internal vs External URLs**: Container-to-container uses internal port (8080), browser redirects use host-mapped port (8081). Same concept as AM Site Configuration in production.
- **Relay State URL List**: AM validates RelayState against an allowlist on the SP entity (Advanced tab). Prevents open redirect attacks. Must include the app's URL.
- **NameID Value Map**: Configured on the hosted IdP. Maps NameID format → user attribute. For `unspecified` format, use `cn` (not `uid` — AM's identity API treats `uid` as the identity name, not a readable profile attribute).
- **Use Name ID as User ID**: Toggle on hosted SP Assertion Processing tab. Tells SP Account Mapper to use the NameID value directly as the uid for local user lookup.
- **Remote entity re-import**: Only needed when metadata changes (endpoints, certs, bindings). Internal config changes (NameID maps, attribute maps, relay state lists) are on the hosted entity and don't require re-import.
- **Identity Store location**: Users at `uid=demo,ou=people,ou=identities` (shared across realms)
- **Federation debug log**: `/opt/am-config/var/debug/Federation` — essential for SAML troubleshooting
- **SP Architecture 2**: App has no SAML support → AM acts as SP gateway → app validates AM session via REST API
- **Cross-domain cookie problem**: AM session cookie (`iPlanetDirectoryPro`) set on `pingam` domain, app on `localhost`. Production solutions: Web Agent, PingGateway, same domain, or token exchange.
- **Common SAML errors debugged**:
  - `No values provided for a request parameter` → malformed metaAlias (double slash)
  - `Invalid Relay State URL specified` → URL not in SP's Relay State URL List
  - `No local user being mapped` → SP can't map NameID to local user (need Account Mapper config)
  - `Unable to generate NameID value` → IdP can't read the attribute from user profile (uid quirk)

---

### Session 5 - 2026-01-30 (continued)
**Starting Point**: Lab 4 - Completing sample-app SAML integration
**Goal**: Fix remaining SAML SSO issues with sample-app

**Progress**:
- [x] Confirmed all SAML entity config persisted across container restarts ✅
- [x] Verified IdP NameID Value Map (`unspecified=cn`) still configured ✅
- [x] Tested basic SP-initiated SSO with demo user → "SSO succeeded" ✅
- [x] Diagnosed sample-app /login error → "Invalid Relay State URL specified" ✅
- [x] Root cause: AM Relay State URL List had `http://localhost:3000` but needed exact path `http://localhost:3000/protected` ✅
- [x] Added `http://localhost:3000/protected` to hosted SP Relay State URL List (partner realm → partner-sp → Advanced tab) ✅
- [x] Sample-app full SAML SSO flow working end-to-end ✅

**Key Learnings**:
- **Git Bash + Docker on Windows**: `docker` command outputs nothing; must use `docker.exe` explicitly. Git Bash's MSYS path conversion also mangles container paths (e.g., `/opt/opendj` → `C:/Program Files/Git/opt/opendj`). Fix: use `MSYS_NO_PATHCONV=1` or use `docker.exe`.
- **AM callback-based auth**: Must preserve full callback structure including `output` fields when submitting credentials. Sending only `input` fields causes HTTP 500 (`NameCallback` constructor throws `IllegalArgumentException: null`).
- **Relay State URL validation strictness**: Some AM versions require the exact URL path in the Relay State URL List, not just a prefix. Adding both `http://localhost:3000` and `http://localhost:3000/protected` is safest.
- **SAML config persistence**: Entity config (NameID maps, Relay State lists, CoT membership) is stored in DS (am-config), so it survives container restarts as long as the DS volume persists.

**Lab 4 Status**: ✅ COMPLETED

---

### Session 6 - 2026-01-30 (continued)
**Starting Point**: Lab 5 - Policy-Based Authorization
**Goal**: Create policies, test evaluation via REST, understand authorization model

**Progress**:
- [x] Created `employees` group with members: demo, alice (bob excluded as contractor) ✅
- [x] Reviewed built-in URL Resource Type — understood pattern syntax and wildcard matching ✅
- [x] Customized URL Resource Type patterns: `http://api.techcorp.com/*`, `http://api.techcorp.com/*?*`, `http://api.techcorp.com/data/*`, `http://api.techcorp.com/data/*?*` ✅
- [x] Configured actions: GET, POST, PUT, DELETE ✅
- [x] Created Policy Set: `TechCorpAPI` (display name: "TechCorp API Access") ✅
- [x] Created Policy: `AllowEmployeeRead` — employees can GET any API resource ✅
- [x] Created Policy: `AllowEmployeeWrite` — employees can POST/PUT to /data/* ✅
- [x] Created Policy: `DenyWriteOutsideBusinessHours` — deny POST/PUT/DELETE 18:00-08:00 IST for all authenticated users ✅
- [x] Tested policy evaluation via REST API ✅
  - demo (employee): GET=Allow on /users, GET/POST/PUT=Allow on /data/reports
  - alice (employee): same as demo
  - bob (contractor): empty actions on both (implicit deny)
- [x] Understood Resource Type → Policy Set → Policy relationship ✅
- [x] Understood PDP vs PEP architecture ✅
- [x] Understood implicit deny vs explicit deny ✅
- [x] Understood multiple policy matching and deny-overrides combining ✅
- [x] Learned Resource Types are user-defined, not fixed types ✅
- [x] Learned about environment conditions: SimpleTime, IPv4, AuthLevel, Script, etc. ✅
- [ ] Test deny policy outside business hours (after 18:00 IST)
- [ ] Test Response Attributes (optional)

**Key Learnings**:
- **Authorization 3-layer model**: Resource Type (template) → Policy Set (application grouping) → Policy (actual rule). None enforce anything without a PEP.
- **PDP vs PEP**: AM is the PDP (decides). Web Agent / PingGateway / app is the PEP (enforces). AM never intercepts traffic.
- **Resource Type design**: Default `*://*:*/*` is too broad for production. Create scoped types per application (e.g., `https://api.bank.com/accounts/*`). Policies can only reference patterns from their Resource Type — this enforces separation of concerns.
- **Resource Types are user-defined**: Name is just a label (AM uses UUID internally). "URL" is not special. Can create any pattern/action combination.
- **Policy resource selection**: Policies pick from Resource Type patterns — you can't type arbitrary URLs. Need to add specific sub-path patterns (e.g., `/data/*`) to the Resource Type first.
- **Implicit deny**: No matching policy = denied. No explicit deny rule needed for bob (contractor) — he simply has no matching policy.
- **Deny overrides allow**: If any policy denies an action, it overrides all allows for that action.
- **Multiple policy matching**: A resource can match multiple policies. Actions are unioned (allows combined), then denies override.
- **Environment conditions**: SimpleTime for business hours, IPv4 for network, AuthLevel for MFA requirements, Script for custom logic.
- **Policy evaluation REST API**: `POST /policies?_action=evaluate` with admin token in header and user ssoToken in body. Returns actions map per resource.
- **Two tokens needed for testing**: Admin token (API access) + user token (subject to evaluate).
- **OAuth2 Scope Resource Type**: Auto-created with OAuth2 Provider. Uses scope names as patterns and GRANT/DENY as actions. Enables policy-driven scope governance.

**Interview Questions Created**: `INT_QA_Policies.md` — Q1-Q11 covering authorization model, Resource Types, PDP/PEP, policy evaluation, implicit/explicit deny, environment conditions, time-based enforcement, and end-to-end production flow.

---

### Session 7 - 2026-01-30 (continued)
**Starting Point**: Lab 6 - User Self-Service → Lab 6a - PingIDM Installation
**Goal**: Configure self-service features, set up PingIDM for tree-based registration

**Progress**:
- [x] Added Mailpit SMTP mock container (separate compose: `docker-compose.mailpit.yaml`) ✅
- [x] Configured Email Service in techcorp realm ✅
  - From: `noreply@techcorp.example.com`
  - SMTP Transport: `mailpit` (secondary configuration)
  - Port: 1025, Non SSL
- [x] Created UserRegistration auth tree (Page Node → Create Object) ✅
- [x] Discovered: Platform Username/Password/Create Object nodes require PingIDM ✅
  - Error: `NullPointerException: Cannot invoke "String.endsWith(String)" because "url" is null`
  - Root cause: IDM integration nodes call IDM REST API, but no IDM URL configured
- [x] Evaluated two approaches: Legacy User Self-Service (service-based) vs IDM integration (tree-based) ✅
- [x] Decision: Set up PingIDM for full tree-based self-service ✅
- [x] Designed two-container IDM architecture (pingidm + pingds-idm) ✅
- [x] Created `pingds-idm/` — Dedicated DS with `idm-repo:8.0` profile ✅
  - Ports: LDAP 2389, LDAPS 2636, Admin 5444
  - Base DN: `dc=openidm,dc=forgerock,dc=com`
- [x] Created `pingidm/` — PingIDM 8.0.1 with custom configs ✅
  - Port: 8082 (HTTP)
  - Config overlay: repo.ds.json → pingds-idm:2389, boot.properties → port 8082
- [x] Created `docker-compose.idm.yaml` (separate from main compose) ✅
- [x] Build and start IDM containers ✅
- [x] Verify IDM admin console accessible ✅
- [x] Configure LDAP connector to pingds (AM identity store) ✅
- [x] Debugged LDAP connector "empty results" — `/system/ldap/account` (POSIX objectClass) vs `/system/ldap/__ACCOUNT__` (inetOrgPerson/ICF standard) ✅
- [ ] Configure managed/user mappings
- [ ] Configure AM-IDM integration
- [ ] Test user registration tree end-to-end

**Key Learnings**:
- **Platform/Create Object nodes are IDM nodes**: They call IDM's REST API internally. Without IDM, they fail with NullPointerException.
- **Legacy vs Tree-based self-service**: AM has two approaches — legacy User Self-Service (service toggles, built-in stages) works standalone. Tree-based uses IDM integration nodes and requires PingIDM.
- **IDM needs its own DS**: IDM repository uses different schema (`fr-idm-*` object classes), different base DN (`dc=openidm,...`). Must be separate from AM's DS for safety.
- **DS `idm-repo` profile**: Built-in setup profile creates all IDM schema, indexes, and base entries. Domain parameter becomes DN suffix.
- **Two DS instances never talk directly**: IDM sits in the middle as orchestrator. LDAP connector reads/writes AM's DS, managed objects live in IDM's DS.
- **Config overlay pattern**: IDM uses file-based JSON config. Custom configs overlaid at container startup from `conf-overlay/` directory.
- **Separate compose for safety**: `docker compose down` on IDM compose doesn't affect AM/DS. Human error isolated.
- **ICF object types vs LDAP objectClasses**: `/system/ldap/account` maps to POSIX `account` objectClass (empty). `/system/ldap/__ACCOUNT__` maps to `inetOrgPerson` (AM's users). Always use `__ACCOUNT__` for AM identity store queries.

**Interview Questions Created**: `INT_QA_IDM_Install.md` — Q1-Q12 covering IDM vs AM, two-DS architecture, idm-repo profile, DS connection config, user sync, config overlay, port allocation, managed vs LDAP users, reconciliation, compose separation, IDM admin UI, AM-IDM integration.

---

### Session 8 - 2026-01-31
**Starting Point**: Lab 6a - AM-IDM Integration (continuing registration tree debugging)
**Goal**: Fix TechCorpRegistration tree NullPointerException, complete user registration

**Progress**:
- [x] Configured IDM Provisioning signing/encryption keys (selfservice, openidm-selfservice-key, HS256, RSAES_PKCS1_V1_5, A128CBC_HS256) ✅
- [x] Enabled useInternalOAuth2Provider ✅
- [x] Created OAuth2 Provider service in techcorp realm ✅
- [x] Created idm-provisioning OAuth2 client in techcorp realm (client_credentials grant, fr:idm:* scope) ✅
- [x] Found NullPointerException resolved — new error: "UnauthorizedClientException: not authorized to use this authorization grant type" ✅
- [x] Discovered debug logs at `/opt/am-config/var/debug/Authentication` (not container stdout) ✅
- [x] Found root realm already had idm-provisioning agent (auto-created, agentonly type) ✅
- [x] Added client_credentials grant type to root realm idm-provisioning via ldapmodify ✅
- [x] Identified root cause chain: useInternalOAuth2Provider=true → AM uses root realm OAuth2 Provider → root realm client missing client_credentials grant → error persists (possibly cached or root OAuth2 Provider config issue) ✅
- [ ] Registration tree still failing — parked for now, moving to next lab

**Key Learnings**:
- **AM debug logs location**: `/opt/am-config/var/debug/` — not streamed to container stdout. Critical files: `Authentication`, `Federation`, `OAuth2Provider`
- **IDM Provisioning auto-creates agent**: When you configure IDM Provisioning global service, AM auto-creates an `agentonly` type entry for `idm-provisioning` in the root realm. This agent identity shares namespace with OAuth2 clients.
- **useInternalOAuth2Provider**: When ON, AM uses its own internal OAuth2 provider (root realm) to issue client_credentials tokens for IDM communication. The idm-provisioning client must exist in root realm with client_credentials grant.
- **agentonly vs OAuth2 client**: AM's agent namespace and OAuth2 client namespace overlap. An existing `agentonly` agent prevents creating an OAuth2 client with the same name. Must modify the agent entry directly in DS.
- **Guava cache**: AM caches schema and config in Guava LoadingCache. Even with `Configuration Cache Duration=0`, some caches persist until restart.
- **Root cause analysis**: NullPointerException → config not loaded (fixed with signing keys) → UnauthorizedClientException → client_credentials not in grant types → added via ldapmodify → still cached. Full Platform deployment likely handles this automatically.

**Lab 6a Status**: 🔧 Partially Complete — LDAP connector, mapping, recon all working. AM-IDM integration configured but registration tree blocked on OAuth2 grant type issue. Moving to next lab.

---

### Session 9 - 2026-01-31 (continued)
**Starting Point**: Lab 6a - Debugging TechCorpRegistration tree (continued from Session 8)
**Goal**: Fix the registration tree OAuth2 grant type error

**Progress**:
- [x] Confirmed root realm idm-provisioning agent only had `authorization_code` grant (ldapmodify from Session 8 didn't persist — heredoc mangled by Git Bash docker exec) ✅
- [x] Fixed ldapmodify approach: write LDIF to file inside container, then run ldapmodify (Git Bash heredoc piping to docker exec silently drops content) ✅
- [x] Added `client_credentials` grant type to root realm idm-provisioning — confirmed persisted in DS ✅
- [x] Restarted PingAM to flush Guava cache — grant type survived restart ✅
- [x] Confirmed OAuth2 `client_credentials` flow works: `POST /am/oauth2/access_token` with idm-provisioning returns access token ✅
- [x] Discovered **real blocker**: IDM rejects AM's bearer token (401 Access Denied) ✅
- [x] Root cause: IDM's `authentication.json` uses `serverAuthContext` (username/password only) — no `rsFilter` configured to validate AM OAuth2 bearer tokens ✅
- [x] Researched rsFilter configuration — requires switching IDM from standalone auth to OAuth2-based auth ✅
- [x] Risk assessment: rsFilter change would break `openidm-admin` password access, IDM Admin UI, and add more moving parts (new `idm-resource-server` OAuth2 client, introspection scopes) ✅
- [x] Decision: Park registration tree, move to next lab — the debugging itself covered all key concepts ✅

**Key Learnings**:
- **Git Bash + Docker exec + heredoc**: Heredocs piped through `docker.exe exec` in Git Bash get silently dropped. The LDAP bind succeeds but the modify request is never sent. Fix: write LDIF to a file inside the container first, then pass the file path to ldapmodify.
- **IDM Integration Service auto-recreates agent**: If you delete the root realm `idm-provisioning` entry from DS, AM's IDM Integration Service automatically re-creates it on demand. Cannot delete-and-recreate; must modify in place.
- **agentonly identity conflict**: The auto-created entry registers an `agentonly` identity that blocks creating a proper OAuth2Client via console or REST API. Console shows "Not found error" when clicking the client. REST API returns 404 for direct access but lists it in query results. The `sunServiceID` in DS says `OAuth2Client` but the internal identity type is `agentonly`.
- **Three-layer fix needed for AM→IDM registration tree**:
  1. ✅ AM IDM Integration Service config (signing keys, OAuth2 provider, client secret)
  2. ✅ Root realm idm-provisioning client with `client_credentials` grant type
  3. ❌ IDM rsFilter authentication (switches IDM from standalone to Platform mode)
- **rsFilter replaces serverAuthContext**: When IDM uses rsFilter, ALL authentication goes through AM OAuth2 token introspection. Username/password auth (`X-OpenIDM-Username/Password`) stops working. Requires `idm-resource-server` OAuth2 client with `am-introspect-all-tokens` scope.
- **Platform installer vs manual setup**: In a ForgeRock Platform deployment, all three layers are configured automatically. Manual Docker setup exposes each layer as a separate problem. This is why ForgeRock sells the Platform as a unified product.
- **IDM internal users**: `internal/user/idm-provisioning` and `internal/role/platform-provisioning` exist out of the box in IDM 8.0 — ready for rsFilter mapping.
- **AM debug level**: Stored in DS at iPlanetAMPlatformService: `com.iplanet.services.debug.level=error`. Set to `message` for detailed logging. Change dynamically without restart (though file-based debug may need restart in AM 8.0).

**Interview Topics Covered**:
- [x] AM-IDM integration architecture (3-layer: AM config → OAuth2 client → IDM rsFilter)
- [x] IDM rsFilter vs serverAuthContext (Platform mode vs standalone mode)
- [x] OAuth2 token introspection for service-to-service auth
- [x] AM agent types (agentonly vs OAuth2Client) and namespace conflicts
- [x] DS direct manipulation with ldapmodify (when console/REST can't manage an entry)
- [x] AM Guava cache behavior and when restarts are needed
- [x] Platform installer automation vs manual configuration complexity

**Lab 6a Final Status**: 🔧 Partially Complete — All AM-side configuration working. Registration tree blocked on IDM rsFilter (requires switching IDM to Platform auth mode). Parked — moving to next lab.

---

### Session 10 - 2026-01-31 (continued)
**Starting Point**: Lab 8 - MFA with OATH/WebAuthn
**Goal**: Configure OATH TOTP and WebAuthn MFA trees

**Progress**:
- [x] Added ForgeRock Authenticator (OATH) Service to /techcorp realm ✅
- [x] Created TechCorpMFA tree: DataStore Decision → OATH Registration → OATH Token Verifier → Success ✅
- [x] Fixed tree wiring: OATH Registration Success must loop back to OATH Token Verifier (not straight to Success) to force TOTP code entry after scanning QR ✅
- [x] Tested OATH TOTP end-to-end — QR scan + code verification working ✅
- [x] Created WebAuthn Profile Encryption Service in /techcorp realm ✅
- [x] Created TechCorpMFA-WebAuthn tree: DataStore Decision → WebAuthn Registration → WebAuthn Authentication → Success ✅
- [x] Discovered WebAuthn requires HTTPS or localhost secure context ✅
- [x] AM redirects localhost:8081 to pingam:8081 (AM's configured FQDN) — breaks WebAuthn origin validation ✅
- [x] Decision: WebAuthn tree built correctly but untestable without TLS — concepts understood ✅

**Key Learnings**:
- **OATH TOTP setup**: Requires ForgeRock Authenticator (OATH) Service at realm level + OATH Registration node (QR code) + OATH Token Verifier node (code input)
- **OATH Registration → Verifier wiring**: Registration Success must connect to Token Verifier, not directly to tree Success. Otherwise user registers device but never proves they have it.
- **WebAuthn requires secure context**: Browser WebAuthn API only available on HTTPS or localhost. AM's site URL redirect (`pingam:8081`) means the origin becomes non-localhost, non-HTTPS — WebAuthn fails silently at DataStore Decision.
- **WebAuthn Profile Encryption Service**: Required before using WebAuthn nodes. Stores encrypted device profiles in user's AM identity.
- **WebAuthn vs FIDO2**: FIDO2 = umbrella standard (WebAuthn API + CTAP). WebAuthn is the browser JavaScript API component. AM implements WebAuthn for FIDO2 passwordless/MFA.
- **AM Site URL impact on WebAuthn**: AM always redirects to its configured FQDN. If FQDN is a Docker hostname (not localhost), WebAuthn breaks. Production deployments use proper HTTPS certificates.

**Lab 8 Status**: 🔧 Partially Complete — OATH TOTP fully working. WebAuthn tree built but blocked by HTTP/hostname limitations.

---

*Last Updated: 2026-02-01 - Session 17. IG agent registered, SSO testing blocked by AM XUI goto issue.*

---

### Session 11 - 2026-01-31 (continued)
**Starting Point**: Interview preparation research
**Goal**: Research custom authentication nodes, Amster CLI, and AM upgrades for interview

**Progress**:
- [x] Researched custom authentication node development (Java API, Maven archetype, lifecycle, testing) ✅
- [x] Compiled real-world examples (risk scoring, MFA, external API, database lookup nodes) ✅
- [x] Researched Amster CLI commands (connect, export-config, import-config) ✅
- [x] Documented Amster Groovy scripting for automation ✅
- [x] Researched CI/CD pipeline integration with Amster ✅
- [x] Documented Amster vs ForgeOps/CDK approaches ✅
- [x] Researched AM upgrade paths (AM 6.x → 7.x → 7.5 → PingAM 8.0) ✅
- [x] Compiled pre-upgrade checklist and best practices ✅
- [x] Documented in-place vs side-by-side upgrade strategies ✅
- [x] Researched custom node compatibility across AM versions ✅
- [x] Documented rollback strategies ✅
- [x] Compiled common upgrade issues and fixes ✅

**Interview Q&A Files Created**:
- `INT_QA_CustomNodes_Amster_Upgrades.md` (Part 1: Q1-Q10 — Custom nodes fundamentals, API, development, testing)
- `INT_QA_CustomNodes_Amster_Upgrades_Part2.md` (Part 2: Q11-Q17 — Real-world examples, Amster basics)
- `INT_QA_CustomNodes_Amster_Upgrades_Part3.md` (Part 3: Q18-Q24 — Amster scripting, CI/CD, upgrade paths)
- `INT_QA_CustomNodes_Amster_Upgrades_Part4_FINAL.md` (Part 4: Q25-Q30 — Upgrade details, rollback, common issues)

**Key Topics Covered**:

**Custom Authentication Nodes** (30 questions total):
- AbstractDecisionNode vs SingleOutcomeNode vs Node interface
- @Attribute annotations and Config interface
- TreeContext, Action, shared state, transient state, callbacks
- Maven archetype for project scaffolding
- OSGi plugin lifecycle and deployment
- OutcomeProvider for defining outcomes
- Unit testing and integration testing strategies
- Real-world examples: risk scoring, SMS OTP, database lookup, external API calls
- ForgeRock Marketplace nodes (Socure, RSA SecurID, Have I Been Pwned)
- Packaging and deployment (.jar → WEB-INF/lib → restart)

**Amster CLI**:
- What is Amster (CLI for AM configuration automation)
- Core commands: connect, export-config, import-config
- Amster vs REST API vs Console (when to use each)
- Groovy scripting for bulk operations and environment promotion
- Configuration expressions (&{variable|default} syntax)
- CI/CD pipeline integration (Jenkins, GitLab CI, GitHub Actions)
- Real-world config promotion workflow (dev → staging → prod)
- Amster vs ForgeOps/CDK approach (traditional vs cloud-native)

**AM Upgrades**:
- Upgrade paths (AM 6.x → 7.x → 7.5 → PingAM 8.0)
- Pre-upgrade checklist (backups, deprecated features, custom code)
- In-place upgrade vs side-by-side (blue/green) deployment
- Configuration migration (AM 6 file-based .properties → AM 7+ DS-based → AM 7.4+ FBC)
- Custom node API compatibility (AM 6 → AM 7 breaking changes, AM 7 → AM 8 compatibility)
- Amster export/import for config promotion
- Rollback strategies (restore DS, switch traffic, K8s rollout undo)
- Common upgrade issues (deprecated chains, REST API changes, embedded DS, custom nodes version=0.0.0, advanced properties lost)

**Key Learnings**:
- Custom nodes extend `AbstractDecisionNode` (boolean outcomes) or `SingleOutcomeNode` (single path) or implement `Node` directly (multi-outcome)
- TreeContext provides access to shared state (non-sensitive), transient state (sensitive/encrypted), and callbacks (user interaction)
- Amster is essential for DevOps automation — export config from dev, commit to Git, import to staging/prod via CI/CD
- ForgeOps/CDK bakes config into Docker images for immutable Kubernetes deployments
- AM 6 → AM 7 was the biggest breaking change (embedded DS removed, config moved to DS, custom node API changed)
- Side-by-side (blue/green) upgrades provide zero downtime and instant rollback
- Always recompile custom nodes when upgrading AM versions
- DS schema upgrades are not reversible — must restore from LDIF backup to rollback DS

**Practical Interview Insights**:
- Focus on real-world examples (risk scoring node, SMS OTP node, database lookup)
- Explain trade-offs (in-place vs blue/green, Amster vs ForgeOps, fail-open vs fail-closed)
- Demonstrate troubleshooting skills (check logs, test in staging, use curl)
- Show DevOps mindset (version control, CI/CD, automated testing, rollback plans)
- Reference specific tools (Maven Shade plugin, Caffeine cache, RestAssured, Playwright)

---

*Last Updated: 2026-01-31 - Session 12. Custom Auth Nodes hands-on lab completed.*

---

### Session 12 - 2026-01-31 (continued)
**Starting Point**: Lab 12 — Custom Authentication Nodes (Hands-On Development)
**Goal**: Build, deploy, and test custom authentication nodes on PingAM 8.0.2

**Progress**:
- [x] Extracted 10 compile-time JARs from pingam container ✅
- [x] Installed all JARs into local Maven repo with `mvn install:install-file -DgeneratePom=true` ✅
- [x] Created Maven project at `sndbx1/custom-nodes/` with pom.xml (all deps as `provided` scope) ✅
- [x] Built Node 1: **BusinessHoursNode** (AbstractDecisionNode) — configurable hours, timezone enum ✅
- [x] Built Node 2: **HeaderCheckNode** (Node direct) — HTTP headers, shared state enrichment ✅
- [x] Built Node 3: **RiskLevelRouterNode** (Node direct) — 4-outcome router, session properties ✅
- [x] Created **TechCorpNodesPlugin** + META-INF/services + .properties files ✅
- [x] First deploy — all 3 nodes visible in AM Console ✅
- [x] Built Node 4: **IpWhitelistNode** (AbstractDecisionNode) — user wrote, X-Forwarded-For parsing ✅
- [x] Learned plugin versioning — must bump version to register new nodes ✅
- [x] Built Node 5: **AuditLogNode** (SingleOutcomeNode) — user wrote, goToNext(), state enrichment ✅
- [x] Built Node 6: **GeoRouterNode** (Node direct) — user created structure, 4-outcome router ✅
- [x] All 7 nodes deployed and visible in AM Console ✅
- [x] Created `INT_QA_CustomNodes_HandsOn.md` — 15 interview Q&A from hands-on experience ✅

**Key Learnings**:
- **3 base class options**: `AbstractDecisionNode` (boolean), `SingleOutcomeNode` (pass-through), `Node` direct (multi-outcome)
- **Config interface**: Methods become UI fields. AM creates dynamic proxy at runtime from DS-stored values
- **`@Inject` + `@Assisted`**: Guice creates per-instance Config. Without `@Assisted`, all nodes share one Config
- **Plugin versioning**: AM stores version in DS. Same version = skip `onInstall()`. Must bump or use `"0.0.0"` for dev
- **`javax.inject` vs `jakarta.inject`**: PingAM 8 = Tomcat 10 = Jakarta EE
- **`AmPlugin` interface**: Lives in `openam-plugin-framework` JAR (not openam-core)
- **Outcome IDs are case-sensitive**: `Action.goTo("Low")` ≠ `new Outcome("low", ...)`
- **Shared state is immutable**: Always `.copy()` then `replaceSharedState()`

**Nodes Built**:
| Node | Base Class | Outcomes | Who Wrote |
|------|-----------|----------|-----------|
| BusinessHoursNode | AbstractDecisionNode | true/false | Claude |
| HeaderCheckNode | Node (direct) | found/missing | Claude |
| RiskLevelRouterNode | Node (direct) | low/medium/high/error | Claude |
| IpWhitelistNode | AbstractDecisionNode | true/false | User |
| AuditLogNode | SingleOutcomeNode | single | User |
| GeoRouterNode | Node (direct) | domestic/intl/blocked/unknown | User + Claude |

**Lab 12 Status**: ✅ Completed

---

### Session 13 - 2026-01-31 (continued)
**Starting Point**: Lab 13 — Amster CLI Hands-On Configuration Management
**Goal**: Hands-on Amster experience — connect, export, import, scripting, expressions, FBC/upgrade concepts

**Progress**:
- [x] Launched Amster 8.0.2 from `sndbx1/amster/amster.bat` ✅
- [x] Connected to AM using `connect -i http://pingam:8081/am` (password mode) ✅
- [x] Learned: must use AM's configured FQDN (`pingam`), not `localhost` — AM redirects ✅
- [x] Exported full AM config: `export-config --path .../amster-exports/full` ✅
- [x] Examined export directory structure: `global/` (server-wide) + `realms/` (per-realm) ✅
- [x] Mapped export structure to AM Console UI (global services, realm services, collections) ✅
- [x] Understood singleton vs collection pattern (`_type.collection: true/false`) ✅
- [x] Understood JSON file structure: `metadata` (address) + `data` (REST body) ✅
- [x] Examined tree JSON wiring (entryNodeId, nodes map, connections, separate node instance files) ✅
- [x] Examined OAuth2 client JSON (config sections map to console tabs) ✅
- [x] Examined policy JSON (subject, actionValues, resourceTypeUuid, applicationName) ✅
- [x] Examined singleton service JSON (OAuth2Provider, Authentication, AuthenticatorWebAuthn) ✅
- [x] Realm-specific export — discovered `--realm` flag doesn't filter (exports everything) ✅
- [x] Created AmsterTestTree in AM Console ✅
- [x] Re-exported, diffed before/after — saw 5 new files (tree + 4 node instances) ✅
- [x] Deleted tree from console, imported from export — tree restored ✅
- [x] Created `scripts/export-techcorp.amster` and `scripts/import-techcorp.amster` ✅
- [x] Ran export script non-interactively: `amster.bat ../scripts/export-techcorp.amster` ✅
- [x] Ran import script — confirmed round-trip works ✅
- [x] Edited OAuth2 client JSON with expression `&{app.callback.url|http://localhost:3000/callback}` ✅
- [x] Imported with expression — default resolved correctly in AM Console ✅
- [x] Covered FBC concepts (history, FBC vs DS-based vs Amster, when to use each) ✅
- [x] Covered Amster for upgrades (pre-upgrade backup, cross-version import, rollback) ✅

**Key Learnings**:
- **Amster connects via REST API** — not LDAP, not direct DS. Gets admin session token, uses it for all calls
- **`-i` flag** for interactive password auth. `-k` for RSA private key (automation/CI/CD)
- **AM FQDN requirement**: Amster must use AM's configured Site URL. `localhost` gets redirected to `pingam`
- **Export structure**: `global/` = Configure → Global Services; `realms/root-{name}` = per-realm config
- **Realm naming**: Flattened with dashes — `/techcorp` → `root-techcorp`, `/techcorp/sub` → `root-techcorp-sub`
- **`--realm` flag on export**: Doesn't filter — still exports full config (cross-realm dependencies)
- **Singleton vs collection**: Standalone JSON (`collection: false`) = one-per-realm service. Directory (`collection: true`) = multiple instances
- **UUID directories**: Node instance configs referenced by trees via UUID
- **JSON structure**: `metadata` (address: entityType + entityId) + `data` (exact REST API response body)
- **Config sections = console tabs**: `coreOAuth2ClientConfig` → Core tab, `advancedOAuth2ClientConfig` → Advanced tab
- **`_type` field**: Always at bottom of `data`, contains `name` (human label) and `collection` (boolean)
- **Each JSON = one REST endpoint entity**: Amster is a REST client with file serialization
- **Diff workflow**: Create tree → re-export → diff shows exactly the new files (tree + node instances)
- **import-config is idempotent**: Creates missing, updates existing. Safe on live AM, no restart needed
- **Script execution**: Must run from amster/ dir (amster.bat needs to find its JAR). Paths in scripts are relative to CWD
- **Configuration expressions**: `&{variable|default}` syntax. Resolved at import time. Enables env-specific config from same files
- **FBC evolution**: AM 6 files → AM 7 DS → AM 7.4+ FBC (JSON filesystem). FBC = immutable cloud-native. Amster = bridge
- **Upgrade safety**: export-config before upgrade, import-config to restore. DS schema upgrade is one-way

**Interview Q&A Created**: `INT_QA_Amster_Hands_On.md` — Q1-Q28 covering Amster connection, export structure, JSON format, singleton vs collection, tree exports, what's exported vs not, dev workflow, import idempotency, --realm flag behavior, all commands, scripts, CI/CD key auth, expressions, pipeline workflow, dependency ordering, secrets handling, FBC evolution, upgrade risks, static vs dynamic config, FBC mode setup, and comprehensive config management answer

**Lab 13 Status**: ✅ Completed

---

### Session 14 - 2026-02-01 (continued)
**Starting Point**: Lab 14 — AM Upgrade Process Walkthrough
**Goal**: Understand complete AM upgrade process with practical examples from our environment

**Progress**:
- [x] Covered pre-upgrade assessment (inventory with Amster, release notes review) ✅
- [x] Analyzed breaking changes for AM 6.5→7.x (major), 7.x→7.5 (minor), 7.5→8.0 (moderate) ✅
- [x] Custom nodes as highest risk — javax→jakarta, API changes, recompile process ✅
- [x] SAML federation fragility — certificate changes, metadata re-distribution, trust relationships ✅
- [x] Three backup types — Amster export, DS LDIF, DS binary (covers different failure scenarios) ✅
- [x] Upgrade order — DS first → AM → custom nodes → IDM ✅
- [x] In-place vs blue-green upgrade strategies ✅
- [x] Post-upgrade verification checklist (trees, OAuth2, SAML, policies, custom nodes) ✅
- [x] Rollback scenarios (AM broke, DS broke, custom nodes broke, blue-green switch) ✅
- [x] Diligence priority order (custom nodes > SAML > DS > OAuth2 > trees > policies) ✅
- [x] OAuth2 silent breakage risk (token format changes) ✅
- [x] IDM upgrade dependencies ✅
- [x] Kubernetes upgrade approach (FBC + rolling update + rollout undo) ✅

**Key Learnings**:
- **Upgrade order matters**: DS first (schema compatibility), AM second, custom nodes third, IDM last
- **DS schema upgrade is irreversible**: Only way back is LDIF restore — take backups before starting
- **Custom nodes = highest risk**: No automated migration, must recompile against new APIs
- **SAML = second highest risk**: Certificate changes break external trust relationships
- **Three backup types**: Amster (readable config), LDIF (portable), binary (fast restore)
- **Blue-green = production standard**: Instant rollback via load balancer switch
- **OAuth2 silent breakage**: Token format changes can break downstream APIs without visible errors
- **Post-upgrade verification**: Must test every tree, OAuth2 flow, SAML SSO, policy evaluation

**Interview Q&A Created**: `INT_QA_Upgrade_Process.md` — Q1-Q15 covering upgrade phases, breaking changes per version, custom node risk, SAML fragility, backup types, upgrade order, in-place vs blue-green, verification checklist, rollback scenarios, priority order, OAuth2 risks, IDM ordering, K8s approach, complete interview answer

**Lab 14 Status**: ✅ Completed

---

### Session 16 - 2026-02-01
**Starting Point**: Lab 16 — PingGateway (Identity Gateway) Installation
**Goal**: Install PingGateway 2025.11.1 in Docker, create routes for SSO and OAuth2 RS

**Progress**:
- [x] Created `pinggw/Dockerfile` — eclipse-temurin:21-jre-jammy, extracts PingGateway zip, iguser UID 1002 ✅
- [x] Created `docker-compose.gw.yaml` — pinggw (:8083) + sample-app (:8084) on fr-net ✅
- [x] Created `pinggw/config/config.json` — Router handler with CaptureDecorator ✅
- [x] Created `pinggw/config/admin.json` — Admin API config ✅
- [x] Created route `01-simple-proxy.json` — ReverseProxyHandler + UriPathRewriteFilter to sample app ✅
- [x] Created route `02-sso.json` — SingleSignOnFilter + AmService + HeaderFilter (needs IG agent) ✅
- [x] Created route `03-oauth2-rs.json` — OAuth2ResourceServerFilter + TokenIntrospectionAccessTokenResolver (needs IG agent) ✅
- [x] Created route `04-headers.json` — StaticResponseHandler showing client IP, path, method, user-agent ✅
- [x] Built and started containers — both running ✅
- [x] Verified route 01 (simple proxy) — `/sample/home` proxies to sample app ✅
- [x] Verified route 04 (headers) — returns JSON with request metadata ✅
- [x] Fixed secretsProvider requirement — PingGateway 2025.11.1 requires Base64EncodedSecretStore for passwords ✅
- [x] Fixed path rewrite — added UriPathRewriteFilter to strip `/sample` prefix ✅
- [x] Discovered admin API is localhost-only (403 from outside container) — security feature ✅
- [x] Confirmed PingGateway has no web UI — headless infrastructure component ✅
- [x] Created `INT_QA_PingGateway.md` — 20 interview Q&A ✅
- [x] Register `ig_agent` in AM Console ✅
- [ ] Test SSO route — BLOCKED: AM XUI goto redirect issue
- [ ] Test OAuth2 RS route — bearer token introspection

**Key Learnings**:
- **PingGateway 2025.11.1 secretsProvider**: Inline `password` field in AmService is no longer accepted. Must use `passwordSecretId` + `Base64EncodedSecretStore` (or FileSystemSecretStore in production). Error: `secretsProvider: Expecting a value`
- **UriPathRewriteFilter**: When proxying `/sample/*` to a backend that serves at `/*`, use UriPathRewriteFilter with `mappings: {"/sample": "/"}` to strip the prefix
- **Route auto-reload**: Route JSON files are scanned every ~10 seconds. Changes take effect without restart. But `config.json` and `admin.json` require container restart.
- **Admin API security**: The admin endpoint (`/openig/api/`) only responds to requests from localhost inside the container. External requests get 403 Forbidden. Access via `docker.exe exec pinggw curl http://localhost:8080/openig/api/info`
- **No UI**: PingGateway is a headless reverse proxy. All configuration via JSON files + REST API. No admin console like AM. Important interview distinction.
- **Sample app**: PingGateway's bundled sample app serves on `/home` (not `/`). Port 8081 internally, mapped to 8084 externally.
- **Base64EncodedSecretStore warning**: Logs warn "should NOT be used in PRODUCTION mode" — use FileSystemSecretStore or KeyStoreSecretStore in production (Kubernetes Secrets, Vault, etc.)
- **Container architecture**: PingGateway runs as embedded Jetty (not Tomcat). Config directory volume-mounted for live editing. No WAR deployment — standalone Java app.

**Interview Q&A Created**: `INT_QA_PingGateway.md` — Q1-Q20 covering architecture, routes, AmService, SingleSignOnFilter, OAuth2ResourceServerFilter, PolicyEnforcementFilter, PasswordReplayFilter, CDSSO, IG agent registration, troubleshooting (CaptureDecorator), HA/production patterns, microservices, secrets management, session handling, enterprise deployment design

**Lab 16 Status**: 🔧 In Progress — Infrastructure built and running. Next: register IG agent, test SSO and OAuth2 RS routes

---

### Session 17 - 2026-02-01 (continued)
**Starting Point**: Lab 16 — PingGateway SSO Testing
**Goal**: Register IG agent, test SingleSignOnFilter SSO flow with AM

**Progress**:
- [x] Added Q25-Q33 to INT_QA_PingGateway.md (context chain, filter ordering, IG agent vs OAuth2, heap objects) ✅
- [x] Registered `ig_agent` in AM Console → Realms → techcorp → Applications → Agents → Identity Gateway ✅
- [x] Set Token Introspection to "Realm" on ig_agent profile ✅
- [x] Restarted PingGateway — all 4 routes loaded (02+03 had failed before ig_agent existed) ✅
- [x] Tested SSO route — AM login page appears via redirect ✅
- [x] Debugged AM goto redirect failure — XUI ignores goto query parameter after login ✅
- [x] Tried multiple loginEndpoint configurations (localhost, pingam, XUI hash routing) ✅
- [x] Identified: AM's `UI/Login` endpoint creates `XUI/?goto=URL#login/` but XUI reads routing from hash fragment, not query params ✅
- [x] Cleared Default Success Login URL (was `/am/console` overriding goto) ✅
- [x] Verified cookie domain behavior — `"domains":[]` means cookie set on exact requesting hostname ✅
- [x] Analyzed HAR file to trace full authentication flow ✅
- [ ] SSO goto redirect — UNRESOLVED: AM XUI SPA does not process goto query parameter after login

**Key Learnings**:
- **AM XUI hash-based routing**: AM's XUI is a SPA that uses `#` fragments for routing (`#login/`, `#profile/details`). The `goto` parameter in the URL query string is passed to `XUI/?goto=URL#login/` but after authentication, the XUI navigates to `#profile/details` ignoring the goto.
- **AM FQDN validation**: AM rejects `localhost` as an invalid FQDN — `FQDN "localhost" is not valid`. Must use AM's configured hostname (`pingam`).
- **Cookie domain behavior**: When AM's cookie domain list is empty (`"domains":[]`), the cookie is set on the exact hostname of the request. Cookie from `pingam:8081` is scoped to `pingam` hostname.
- **`_ig=true` callback marker**: SingleSignOnFilter appends `?_ig=true` to the goto URL. This lets PingGateway recognize when the user is returning from AM login vs making a fresh request. Without it, PingGateway enters infinite redirect.
- **loginEndpoint in SingleSignOnFilter**: Custom redirect URL for browser login. Overrides the default AM login redirect. Must include `_ig=true` in the goto parameter.
- **Default Success Login URL override**: AM's Post Authentication Processing can set a Default Success Login URL that overrides the goto redirect entirely. If set to `/am/console`, all logins redirect there regardless of goto.
- **Docker port change = data loss**: Changing ports in docker-compose.yaml triggers container recreation. PingDS stores data in container filesystem (not volume), so recreation destroys all realms, users, agents, and trees.
- **Routes need ig_agent at startup**: Routes 02-sso and 03-oauth2-rs failed to build at startup because ig_agent wasn't registered yet. Restarting PingGateway after agent registration fixed the route loading.

**Current SSO Route State** (`02-sso.json`):
- AmService URL: `http://pingam:8080/am` (internal container network)
- loginEndpoint: `http://pingam:8081/am/?goto=...%3F_ig%3Dtrue&realm=/techcorp` (browser-facing)
- HeaderFilter: injects X-IG-User and X-IG-Session
- Backend: `http://sample-app:8081`

**Blocker**: AM XUI does not process the `goto` query parameter after authentication. After login, user is shown profile page instead of being redirected. This is a fundamental AM XUI behavior issue, not a PingGateway configuration problem.

**Next Steps**: Analyze HAR file authenticate response body for `successUrl` field. Consider alternative approaches: AM service URL configuration, custom tree with goto handling, or CDSSO.

**Lab 16 Status**: 🔧 In Progress — IG agent registered, routes loading. SSO test blocked by AM goto redirect
