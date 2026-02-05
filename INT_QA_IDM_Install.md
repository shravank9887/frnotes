# PingIDM Installation & Architecture — Interview Q&A

## Q1: What is PingIDM and how does it differ from PingAM?

**Answer:**

| Aspect | PingAM | PingIDM |
|--------|--------|---------|
| Purpose | Authentication & Authorization (Who are you? What can you access?) | Identity Lifecycle Management (Provisioning, sync, workflows) |
| Core Function | SSO, Federation, Policies, MFA | User provisioning, reconciliation, connectors, managed objects |
| Data Ownership | Consumes identities for authentication | Manages identity lifecycle (create, update, disable, delete) |
| Runs On | Apache Tomcat (WAR deployment) | Standalone Jetty (embedded, OSGi-based) |
| Config Format | AM Console + DS backend | JSON files in /conf directory + DS or JDBC repository |
| Default Port | 8080 | 8080 (we changed to 8082 to avoid conflict) |

**Key Insight**: AM answers "who is this user?" — IDM answers "where did this user come from, where should they exist, and what happens when they leave?"

---

## Q2: Why does PingIDM need its own DS instance?

**Answer:**

IDM stores fundamentally different data than AM:

**AM's DS (pingds)** stores:
- `ou=am-config` — AM server configuration
- `ou=identities` — user accounts for authentication
- `ou=tokens` — CTS session/OAuth tokens

**IDM's DS (pingds-idm)** stores:
- `dc=openidm,dc=forgerock,dc=com` — IDM repository root
- `ou=users,ou=internal` — IDM internal admin users
- `ou=roles,ou=internal` — IDM internal roles
- `ou=generic` — generic managed objects
- `ou=managed` — explicit managed objects (users, roles)
- `ou=links` — reconciliation link table (maps managed objects to external resources)
- `ou=cluster` — cluster state
- `ou=scheduler` — scheduled jobs

**Why separate?**
1. **Schema isolation** — IDM requires `fr-idm-*` object classes that AM doesn't need
2. **Blast radius** — IDM repo corruption doesn't affect AM authentication
3. **Independent scaling** — IDM repo can be sized differently than AM's DS
4. **Upgrade independence** — upgrade IDM without touching AM's data

In production, ForgeRock recommends dedicated DS instances per purpose (am-config, am-cts, am-identity, idm-repo).

---

## Q3: What is the `idm-repo` DS setup profile?

**Answer:**

PingDS ships with built-in setup profiles that pre-configure the directory for specific ForgeRock products:

| Profile | Purpose | Base DN |
|---------|---------|---------|
| `am-config` | AM server configuration | `ou=am-config` |
| `am-identity-store` | User identities for AM auth | `ou=identities` |
| `am-cts` | Core Token Service (sessions, tokens) | `ou=tokens` |
| `idm-repo` | IDM managed objects, links, internal users | `dc=openidm,dc={domain}` |

The `idm-repo:8.0` profile creates:
- All IDM-specific LDAP schema (object classes, attributes)
- Base DN structure (`dc=openidm,dc=forgerock,dc=com`)
- Indexes optimized for IDM query patterns
- Sub-trees: `ou=generic`, `ou=managed`, `ou=links`, `ou=internal`, `ou=cluster`, etc.

**Setup command**:
```bash
./setup --profile idm-repo:8.0 \
        --set idm-repo/domain:forgerock.com
```

The `domain` parameter becomes the base DN suffix. `forgerock.com` becomes `dc=openidm,dc=forgerock,dc=com`.

---

## Q4: How does IDM connect to its DS repository?

**Answer:**

IDM's DS connection is configured in `conf/repo.ds.json`:

```json
{
  "ldapConnectionFactories": {
    "bind": {
      "connectionSecurity": "none",
      "primaryLdapServers": [
        {"hostname": "pingds-idm", "port": 2389}
      ]
    },
    "root": {
      "inheritFrom": "bind",
      "authentication": {
        "simple": {
          "bindDn": "cn=Directory Manager",
          "bindPassword": "Passw0rd123"
        }
      }
    }
  }
}
```

Two connection factories:
- **bind** — for anonymous/simple operations, defines server list + connection security
- **root** — inherits from bind, adds authentication credentials for privileged operations

**Connection security options**: `none`, `startTLS`, `ssl`

In production, always use `startTLS` or `ssl` with proper certificates.

---

## Q5: Explain the IDM two-DS architecture for user sync

**Answer:**

```
┌─────────────────────────────────────────────────────────┐
│                      PingIDM                             │
│                                                          │
│  ┌──────────────┐   Reconciliation   ┌───────────────┐  │
│  │ Managed Users │ ◄══════════════► │ LDAP Connector │  │
│  │ (IDM objects) │   (sync engine)   │ (to pingds)   │  │
│  └──────┬───────┘                    └──────┬────────┘  │
└─────────┼──────────────────────────────────┼────────────┘
          │                                  │
          ▼                                  ▼
   ┌─────────────┐                    ┌─────────────┐
   │  pingds-idm  │                    │   pingds     │
   │  (IDM repo)  │                    │  (AM users)  │
   │              │                    │              │
   │ Stores:      │                    │ Stores:      │
   │ - managed/user│                   │ - uid=demo   │
   │ - links table │                   │ - uid=alice  │
   │ - audit logs  │                   │ - uid=bob    │
   └─────────────┘                    └─────────────┘
```

**The flow for self-registration:**
1. User fills registration form in AM (browser)
2. AM's tree calls IDM API via `Create Object` node
3. IDM creates `managed/user` object in pingds-idm
4. IDM mapping auto-provisions user to pingds (LDAP connector)
5. User now exists in AM's identity store
6. AM creates session, user is logged in

**Key insight**: The two DS instances never talk to each other directly. IDM sits in the middle as the orchestrator.

---

## Q6: What is IDM's config overlay pattern?

**Answer:**

IDM uses a **file-based configuration** system:
- Default configs ship in `/opt/openidm/conf/`
- Custom configs are overlaid at startup from `/opt/openidm/conf-overlay/`
- The overlay pattern copies custom files on top of defaults

This is different from AM's approach (AM stores config in DS, managed via Console/REST).

**Files we customized:**
| File | Purpose | Change |
|------|---------|--------|
| `repo.ds.json` | DS repository connection | hostname: `pingds-idm`, port: `2389`, bindDn: `cn=Directory Manager` |
| `boot.properties` | Server startup config | HTTP port: `8082` (avoid conflict with pingds) |
| `webserver.listener-http.json` | HTTP listener | port: `8082` |

**Interview tip**: IDM's JSON config files are hot-reloadable. Changes to files in `conf/` take effect without restart (for most files). This is because IDM uses OSGi's ConfigAdmin service with a file-based installer that polls for changes.

---

## Q7: What ports does each component use in a full ForgeRock deployment?

**Answer:**

| Component | Container | LDAP | LDAPS | Admin | HTTP | HTTPS |
|-----------|-----------|------|-------|-------|------|-------|
| DS (AM) | pingds | 1389 | 1636 | 4444 | 8080 | 8443 |
| DS (IDM) | pingds-idm | 2389 | 2636 | 5444 | 9080 | 9443 |
| AM | pingam | - | - | - | 8081 (host) → 8080 (container) | 8444 → 8443 |
| IDM | pingidm | - | - | - | 8082 | 9443 |
| Mailpit | mailpit | - | - | - | 8025 (UI), 1025 (SMTP) | - |

**Port conflict avoidance**: Each DS instance uses different ports. IDM uses 8082 instead of default 8080.

---

## Q8: What is the difference between `managed/user` and LDAP users?

**Answer:**

| Aspect | managed/user (IDM) | LDAP user (DS) |
|--------|-------------------|----------------|
| Location | pingds-idm (`dc=openidm,...`) | pingds (`ou=people,ou=identities`) |
| Format | JSON object | LDAP entry (inetOrgPerson) |
| Access | IDM REST API (`/openidm/managed/user/`) | LDAP protocol or AM REST API |
| Purpose | Canonical identity record for lifecycle management | Authentication identity |
| Contains | All user properties + metadata + relationships | LDAP attributes (uid, cn, mail, userPassword) |
| Created by | IDM (self-service, admin, reconciliation) | IDM provisioning or direct LDAP |

**The link table** in pingds-idm maps: `managed/user/{id}` ↔ `uid=demo,ou=people,ou=identities` (in pingds). This bidirectional link enables IDM to keep both sides in sync.

---

## Q9: What is reconciliation in IDM?

**Answer:**

Reconciliation ("recon") is IDM's process for comparing and synchronizing identities between two systems:

**Three types:**
1. **Full reconciliation** — Compare all records on both sides, create/update/delete as needed
2. **LiveSync** — Monitor DS changelog for real-time incremental sync
3. **Implicit sync** — When IDM creates/updates a managed object, the mapping immediately provisions to external systems

**Recon situations:**
| Source has | Target has | Action |
|-----------|-----------|--------|
| User A | No User A | Create in target (provision) |
| User A (updated) | User A (stale) | Update target |
| No User A | User A | Flag for review or delete (deprovision) |
| User A | User A (matching) | Skip (no action) |

**Production example**: HR system creates employee → IDM recon creates AD account + email + app accounts → Employee leaves → HR disables → IDM recon disables all linked accounts.

---

## Q10: Why use Docker Compose separation for IDM components?

**Answer:**

We use a **separate compose file** (`docker-compose.idm.yaml`) rather than adding to the main compose file:

**Safety reasons:**
1. `docker compose down` only tears down IDM containers, not AM/DS
2. No risk of accidental edits to AM/DS service definitions
3. `docker compose up --build` only rebuilds IDM images
4. Human error is isolated to IDM scope

**How it still works together:**
- All compose files share the `fr-net` external Docker network
- Containers resolve each other by hostname (pingam, pingds, pingds-idm, pingidm)
- No `depends_on` across files, but we manage startup order manually

**The tradeoff**: We lose cross-file `depends_on` (can't say "pingidm depends_on pingds"), but gain complete isolation. For a learning lab, manual startup order is fine. In production, Kubernetes handles this with readiness probes.

---

## Q11: What is the IDM admin interface?

**Answer:**

PingIDM provides two web interfaces:

1. **Admin UI** — `http://pingidm:8082/admin/` — Full management console for:
   - Managed objects (users, roles)
   - Connectors (LDAP, AD, SCIM, scripted)
   - Mappings (sync definitions)
   - Reconciliation (run and monitor)
   - Schedules, workflows, audit

2. **End User UI** — `http://pingidm:8082/enduser/` — Self-service portal for:
   - Profile management
   - Password change
   - Consent management
   - Dashboard

**Default admin credentials**: `openidm-admin` / `openidm-admin`

**REST API**: `http://pingidm:8082/openidm/` — All IDM operations available via REST (managed objects, system objects, recon, scheduler, etc.)

---

## Q12: How does AM integrate with IDM for self-service trees?

**Answer:**

AM's "Identity Management" tree nodes (Platform Username, Platform Password, Create Object, Patch Object) are **IDM integration nodes**. They call IDM's REST API internally.

**Configuration required:**
1. AM needs to know IDM's URL — configured in AM's global services
2. IDM needs to accept AM's requests — configured in IDM's `authentication.json`
3. Both need to share the same DS identity store — IDM's LDAP connector points to AM's DS

**What happens without IDM:**
- Platform Username/Password/Create Object nodes throw `NullPointerException` ("url" is null)
- AM has no IDM URL configured, so the integration handler gets null
- Must use legacy User Self-Service service instead (service-based, not tree-based)

**What happens with IDM:**
- Create Object → `POST /openidm/managed/user` → IDM creates user → LDAP connector provisions to DS → AM can authenticate user
- Platform Password → IDM enforces password policy → stores hashed password

This is why ForgeRock's modern deployment always includes both AM + IDM together — the self-service features depend on the integration.

---

## Q13: How does TLS certificate trust work between PingIDM and PingDS?

**Answer:**

PingIDM connects to PingDS using **startTLS** (upgrades plain LDAP to encrypted). For this to work, IDM must **trust** the DS server's certificate.

### Certificate Chain

```
DS Setup (dskeymgr)
  → Generates deployment key pair
  → Creates self-signed CA certificate
  → Signs server certificate with CA
  → Server cert stored in DS keystore

DS Export (dskeymgr export-ca-cert)
  → Exports CA cert as PEM file
  → Shared via Docker volume

IDM Import (keytool -import)
  → Imports CA cert into IDM's truststore (JKS)
  → IDM now trusts any cert signed by this CA
  → startTLS handshake succeeds
```

### Key Files

| File | Location | Purpose |
|------|----------|---------|
| `ds-idm-ca-cert.pem` | `/opt/certs/` (shared volume) | DS CA certificate in PEM format |
| `truststore` | `/opt/openidm/security/truststore` | IDM's JKS truststore (CA cert imported here) |
| `storepass` | `/opt/openidm/security/storepass` | Password for IDM's truststore |

### repo.ds.json Trust Configuration

```json
{
  "security": {
    "trustManager": "file",
    "fileBasedTrustManagerType": "JKS",
    "fileBasedTrustManagerFile": "&{idm.install.dir}/security/truststore",
    "fileBasedTrustManagerPasswordFile": "&{idm.install.dir}/security/storepass"
  },
  "ldapConnectionFactories": {
    "bind": {
      "connectionSecurity": "startTLS",
      "primaryLdapServers": [{"hostname": "pingds-idm", "port": 2389}]
    }
  }
}
```

### trustManager Options

| Value | Meaning |
|-------|---------|
| `file` | Read trust certs from a file-based truststore (JKS/PKCS12). You control exactly which CAs are trusted. |
| `jvm` | Use the JVM's default truststore (`cacerts`). Only trusts public CAs — won't trust DS self-signed certs. |
| `blind` | Trust everything (NEVER use in production — defeats TLS purpose) |

### startTLS vs LDAPS

| Protocol | Port | How TLS starts |
|----------|------|----------------|
| startTLS | 2389 (LDAP port) | Client sends StartTLS extended operation, then upgrades to TLS on same port |
| LDAPS | 2636 | TLS from first byte — dedicated encrypted port |

**startTLS is preferred** because:
1. Single port for both plain and encrypted (simpler firewall rules)
2. Can detect misconfiguration (plain LDAP still works if TLS fails)
3. ForgeRock's default in repo.ds.json

### Docker Volume Pattern for Certificate Sharing

```yaml
# pingds-idm writes certs
volumes:
  - shared-certs-idm:/opt/certs

# pingidm reads certs (read-only)
volumes:
  - shared-certs-idm:/opt/certs:ro
```

The `:ro` mount on IDM prevents accidental modification. DS owns the certificate lifecycle — IDM only consumes.

### Startup Ordering (Chicken-and-Egg Problem)

1. `pingds-idm` starts → runs DS setup → starts DS server
2. Background task waits for DS to be ready → exports CA cert to shared volume
3. `pingidm` starts (after DS healthcheck passes) → waits up to 90s for cert file
4. IDM imports CA cert into truststore → starts IDM → connects to DS via startTLS

This is a common pattern in containerized deployments. In Kubernetes, you'd use init containers or cert-manager instead.

---

## Q14: What is the difference between `__ACCOUNT__` and `account` in the LDAP connector?

**Answer:**

The LDAP connector exposes multiple **object types**. Two are easily confused:

| Object Type | REST Path | LDAP objectClass | Used For |
|---|---|---|---|
| `__ACCOUNT__` | `/system/ldap/__ACCOUNT__` | `inetOrgPerson` | Standard user accounts (AM's users) |
| `account` | `/system/ldap/account` | `account` (POSIX) | POSIX accounts (rarely used) |

`__ACCOUNT__` and `__GROUP__` are **ICF (Identity Connector Framework) standard types** — abstract types that every connector maps to its native equivalents. The double-underscore convention indicates they are ICF-reserved.

**Common mistake**: Querying `/system/ldap/account` and getting empty results, when users are actually `inetOrgPerson` entries accessible via `/system/ldap/__ACCOUNT__`.

**Key insight**: Always use `__ACCOUNT__` for standard user queries against AM's DS. The plain `account` type is a separate POSIX objectClass that AM does not use.

---

## Q15: What is an IDM mapping and how do you create one?

**Answer:**

A **mapping** defines how identities flow between two resources (source → target). It includes:

1. **Source**: Where data comes from (e.g., `system/ldap/__ACCOUNT__`)
2. **Target**: Where data goes (e.g., `managed/user`)
3. **Attribute grid**: Which source attributes map to which target properties
4. **Behaviors/Situation policies**: What to do in each sync scenario
5. **Scheduling**: When to run automatic reconciliation

**Mapping naming convention**: `{source}{target}` — e.g., `systemLdap__ACCOUNT__managedUser`

**Creating a mapping in IDM Admin UI:**
1. Configure → Mappings → New Mapping
2. Select source connector + object type (e.g., LDAP → `__ACCOUNT__`)
3. Select target (e.g., Managed → User)
4. IDM auto-generates the mapping name
5. Optionally set a **Linked Mapping** (reverse direction for bidirectional sync)
6. Configure attribute mappings and situation policies

**Linked Mapping**: If you have `LDAP→managed/user` and also `managed/user→LDAP`, linking them tells IDM they share the same association links. Changes synced by one mapping won't trigger the reverse mapping unnecessarily.

---

## Q16: What are IDM mapping situation policies (Behaviors)?

**Answer:**

Situation policies define what IDM does when it encounters different sync scenarios during reconciliation:

| Situation | Meaning | Typical Action |
|---|---|---|
| **ABSENT** | Source record exists, no matching target | `CREATE` (provision new user) |
| **FOUND** | Source and target both exist and are linked | `UPDATE` (sync changes) |
| **MISSING** | Target exists, no matching source | `DELETE` or `IGNORE` or `UNLINK` |
| **UNQUALIFIED** | Record doesn't meet mapping conditions | `IGNORE` |
| **AMBIGUOUS** | Source matches multiple targets | `IGNORE` (needs manual resolution) |
| **SOURCE_MISSING** | Previously linked source record is gone | `DELETE` or `EXCEPTION` |
| **UNASSIGNED** | Target exists but isn't linked to any source | `IGNORE` |

**Read-only policy**: The default policy is read-only — reconciliation runs but makes no changes. You must change situation policies to enable create/update/delete actions.

**Interview insight**: Situation policies are what make IDM flexible. A conservative setup might `IGNORE` missing sources (don't auto-delete), while an aggressive one might `DELETE` to enforce authoritative source compliance.

---

## Q17: What does "Persist Associations" mean during reconciliation?

**Answer:**

When triggering a reconciliation, IDM asks whether to **persist associations** (`&persistAssociations=true`).

**Associations (links)** are records stored in IDM's repository (pingds-idm, `ou=links`) that map a managed object to its external counterpart:

```
managed/user/abc-123  ↔  system/ldap/__ACCOUNT__/entryUUID=xyz-789
```

| Setting | Behavior | Use Case |
|---|---|---|
| **ON** (persist) | IDM saves link records after recon. Future recons use these links to match records, enabling updates instead of duplicate creates. | Production / real sync |
| **OFF** (don't persist) | IDM runs recon but doesn't save links. Next recon treats all records as new. | Dry-run / testing |

**Always enable for real synchronization.** Without persisted links, IDM cannot track which managed user corresponds to which LDAP entry, and subsequent reconciliations would attempt to create duplicates.

**Where links are stored**: `ou=links,dc=openidm,dc=forgerock,dc=com` in pingds-idm.

---

## Q18: What are the essential attribute mappings for LDAP → managed/user?

**Answer:**

The `managed/user` schema has 4 **required** fields. Each must be mapped from an LDAP source attribute:

| LDAP Attribute (source) | managed/user Property (target) | Notes |
|---|---|---|
| `uid` | `userName` | Primary login identifier |
| `givenName` | `givenName` | First name |
| `sn` | `sn` | Last name |
| `mail` | `mail` | Email address |

**Optional but useful mappings:**
| LDAP Attribute | managed/user Property |
|---|---|
| `telephoneNumber` | `telephoneNumber` |
| `description` | `description` |
| `__PASSWORD__` | `password` |

**`__PASSWORD__`** is another ICF reserved attribute. It represents the password in a connector-safe way (`JAVA_TYPE_GUARDEDSTRING`). Mapping it enables password sync between systems, but handle carefully — bidirectional password sync can cause loops.

**"Add Missing Required Properties"** button in the IDM UI auto-adds the 4 required target fields. You still need to manually set each one's source attribute.

---

## Q19: Why does LDAP → managed/user reconciliation fail with "Policy validation failed"?

**Answer:**

LDAP attributes are **always multi-valued** by protocol design. The LDAP schema for `uid`, `sn`, `givenName`, `mail` all lack the `SINGLE-VALUE` flag, so the ICF LDAP connector correctly models them as `"type": "array"` and returns data like `["bob"]` instead of `"bob"`.

However, `managed/user` properties (`userName`, `sn`, `givenName`, `mail`) are defined as `"type": "string"`. When IDM tries to create a managed user with array values, policy validation rejects the type mismatch:

```json
{
  "policyRequirement": "VALID_TYPE",
  "params": {
    "invalidType": "array",
    "validTypes": ["string"]
  }
}
```

**Fix**: Add a **transformation script** to each attribute mapping that extracts the first element:

```javascript
source[0]
```

This is configured in the IDM Admin UI: Properties tab → edit (pencil icon) on each mapping row → add Transformation Script → `source[0]`.

**Why this is the correct approach (not a hack):**
1. LDAP defines these attributes as multi-valued (no `SINGLE-VALUE` in schema) — the connector is correct to return arrays
2. The connector's object type schema explicitly defines them as `"type": "array"` with `"items": {"type": "string"}`
3. PingIdentity documentation confirms using JavaScript transforms (e.g., `source[0]`, `source.join(',')`) to convert array attributes to strings for managed objects
4. `readSchema: true` on the connector would not help — these attributes are genuinely multi-valued in the LDAP spec

**Also watch for**: Empty arrays (e.g., `givenName: []`) will cause `source[0]` to return `undefined`, which also fails validation. Ensure source LDAP data has values for all required fields, or use a safer transform like `source && source.length > 0 ? source[0] : ""`.

---

## Q20: What data quality issues can cause reconciliation failures?

**Answer:**

Even with correct mappings and transforms, recon can fail if source LDAP data is incomplete:

| Issue | Example | Result |
|---|---|---|
| Missing required attribute | `givenName: []` (empty) | `VALID_TYPE` or `minimum-length` policy failure |
| Null attribute | Attribute not present on LDAP entry | Transform returns `undefined` → policy failure |
| Duplicate userName | Two LDAP users with same `uid` | `unique` policy failure on `userName` |

**Debugging approach:**
1. Check IDM logs: `docker.exe logs pingidm` — look for "Failed to create target object" and "Policy validation failed"
2. Test manually: `POST /openidm/managed/user?_action=create` with the exact data the mapping would send — the error response includes `failedPolicyRequirements` with the specific field and policy that failed
3. Query source data: `GET /openidm/system/ldap/__ACCOUNT__?_queryFilter=true&_fields=uid,sn,givenName,mail` — verify all required fields have values

---

## Q21: How do you configure AM-IDM integration for Platform tree nodes?

**Answer:**

AM's Platform nodes (Platform Username, Platform Password, Create Object, Patch Object) call IDM's REST API internally. AM needs to know where IDM is.

**Configuration**: AM Console → Configure → **Global Services** → **IDM Provisioning**

| Field | Value | Purpose |
|---|---|---|
| Enabled | ON | Activates the integration |
| Deployment URL | `http://pingidm:8082` | IDM's internal URL (container-to-container) |
| Deployment Path | `openidm` | REST API base path |
| IDM Provisioning Client | `idm-provisioning` | OAuth2 client name (default) |
| useInternalOAuth2Provider | OFF | For lab; production would use OAuth2 between AM and IDM |
| provisioningClientScopes | `fr:idm:*` | Scope for IDM API access |

**Key points:**
- This is a **global** (root-level) service, not per-realm. All realms share the same IDM connection.
- Without this config, Platform nodes throw `NullPointerException` ("url" is null).
- In production, `useInternalOAuth2Provider` would be ON — AM obtains an OAuth2 token to authenticate to IDM, rather than using basic auth.

---

## Q22: How do you build a user registration tree with IDM integration?

**Answer:**

A registration tree uses AM's identity management nodes to collect user data and provision via IDM.

### Tree Structure

```
Start
  │
  ▼
Page Node
  ├── Platform Username      (collects username)
  ├── Platform Password      (collects + confirms password)
  └── Attribute Collector    (collects givenName, sn, mail)
  │
  ▼
Create Object
  ├── Created → Success
  └── Failed  → Failure
```

### Node Details

| Node | Category | Purpose |
|---|---|---|
| **Page Node** | Container | Groups child nodes into a single form page |
| **Platform Username** | Identity Management | Captures username, checks uniqueness against IDM |
| **Platform Password** | Identity Management | Captures password with confirmation, enforces IDM password policies |
| **Attribute Collector** | Identity Management | Collects arbitrary user profile fields |
| **Create Object** | Identity Management | `POST /openidm/managed/user` — creates the user in IDM |

### Attribute Collector Configuration

- **Attributes to Collect**: `givenName`, `sn`, `mail`
- **Identity Attribute**: `userName` (maps to managed/user's userName property)
- **All Attributes Required**: ON (ensures all fields are filled)
- **Validate Input**: Optional (enables IDM-side validation of field formats)

### Create Object Configuration

- **Identity Resource**: `managed/user` — tells the node which IDM managed object to create

### What happens end-to-end

1. User navigates to `http://localhost:8081/am/XUI/?realm=/techcorp&service=TechCorpRegistration`
2. AM renders the Page Node form (username, password, givenName, sn, mail)
3. User submits → AM calls IDM: `POST /openidm/managed/user`
4. IDM creates `managed/user` in pingds-idm repository
5. IDM's LDAP mapping (if implicit sync is configured) provisions user to AM's DS (pingds)
6. Create Object returns "Created" → tree reaches Success
7. User now exists in both IDM and AM's identity store

### Key Interview Insight

The registration tree demonstrates the **AM + IDM integration pattern**: AM handles the authentication flow (tree, callbacks, session), while IDM handles the identity lifecycle (create user, enforce policies, provision to external systems). Neither can do the other's job alone — AM can't provision, IDM can't run auth trees.

---

## Q23: What signing/encryption keys does the IDM Provisioning service require?

**Answer:**

The IDM Provisioning global service needs signing and encryption configuration for secure AM-IDM communication:

| Field | Value | Purpose |
|---|---|---|
| Signing Key Alias | `selfservice` | Key in AM's keystore used to sign JWT tokens for IDM |
| Encryption Key Alias | `openidm-selfservice-key` | Key used to encrypt payloads sent to IDM |
| Signing Algorithm | `HS256` | HMAC-SHA256 for JWT signing |
| Encryption Algorithm | `RSAES_PKCS1_V1_5` | RSA encryption for key wrapping |
| Encryption Method | `A128CBC_HS256` | AES-128 with HMAC-SHA256 for content encryption |

**Without these keys configured**, AM throws `NullPointerException` at `RequestHandlerProvider.ensureEndingSlash()` — the IDM deployment URL returns null from the config because the service isn't fully initialized.

**Key insight**: These key aliases (`selfservice`, `openidm-selfservice-key`) are expected to exist in AM's JCEKS keystore. In a full Ping Identity Platform deployment, the installer creates these automatically. In standalone deployments, they may need to be manually created or imported.

---

## Q24: What is `useInternalOAuth2Provider` in IDM Provisioning?

**Answer:**

This toggle controls how AM authenticates when calling IDM's REST API:

| Setting | Behavior |
|---|---|
| **ON** | AM uses its own internal OAuth2 provider to issue a `client_credentials` token, then sends it to IDM as a Bearer token |
| **OFF** | AM uses basic authentication or an external OAuth2 provider |

**When ON**, the following must be true:
1. The **root realm** must have an OAuth2 Provider service
2. An OAuth2 client named `idm-provisioning` must exist in the root realm
3. That client must have `client_credentials` in its allowed grant types
4. The `provisioningClientSecret` must match the client's secret
5. The `provisioningClientScopes` (e.g., `fr:idm:*`) must be in the client's allowed scopes

**Common pitfall**: AM auto-creates an `idm-provisioning` entry as an `agentonly` type when you save the IDM Provisioning config. This agent shares the OAuth2 client namespace but may not have `client_credentials` grant type configured — causing `UnauthorizedClientException`.

---

## Q25: Where are AM's runtime debug logs?

**Answer:**

AM's debug logs are **not** streamed to container stdout (Docker logs). They live inside the container at:

```
/opt/am-config/var/debug/
```

**Key log files:**

| File | Content |
|---|---|
| `Authentication` | Auth tree execution errors, node failures, callback issues |
| `Federation` | SAML2/OIDC federation errors |
| `OAuth2Provider` | OAuth2 token issuance, client validation errors |
| `CoreSystem` | AM core framework errors |
| `IdRepo` | Identity repository (DS) interaction errors |
| `Configuration` | Service/config loading issues |
| `Session` | Session creation/validation errors |

**How to read:**
```bash
docker.exe exec pingam tail -100 //opt/am-config/var/debug/Authentication
```

**Audit logs** are at `/opt/am-config/var/audit/` — structured JSON logs for access events, activity, and authentication outcomes.

**Interview insight**: When troubleshooting AM issues, always check debug logs first — container stdout only shows Tomcat startup/shutdown. The debug log level is controlled per component in AM Console → Configure → Global Services → Debugging.

---

## Q26: What is the AM agent namespace conflict with OAuth2 clients?

**Answer:**

AM stores both **agents** (Web Agent, J2EE Agent, OAuth2 client) and **OAuth2 clients** under the same `AgentService` in the config store. They share a flat namespace — you cannot have an agent and an OAuth2 client with the same name.

**Example conflict:**
1. IDM Provisioning config auto-creates `idm-provisioning` as an `agentonly` agent in the root realm
2. Attempting to create an OAuth2 client named `idm-provisioning` fails: *"Identity idm-provisioning of type agentonly already exists"*

**Resolution options:**
1. **Modify the existing agent** directly in DS via `ldapmodify` to add OAuth2 properties (grant types, scopes)
2. **Delete the agent first**, then recreate as a proper OAuth2 client
3. **Use a different client name** in the IDM Provisioning config

**DS path for the agent:**
```
ou=idm-provisioning,ou=default,ou=OrganizationConfig,ou=1.0,ou=AgentService,ou=services,ou=am-config
```

Grant types are stored as:
```
sunKeyValue: com.forgerock.openam.oauth2provider.grantTypes=[0]=authorization_code
sunKeyValue: com.forgerock.openam.oauth2provider.grantTypes=[1]=client_credentials
```

---

## Q13: What are the three layers needed for AM→IDM registration tree integration?

**Answer:**

The TechCorpRegistration tree (Page Node → Create Object) requires three configuration layers:

1. **AM IDM Integration Service** (Global Services):
   - `idmDeploymentUrl`: IDM URL (e.g., `http://pingidm:8082`)
   - `idmProvisioningClient`: OAuth2 client name (`idm-provisioning`)
   - `provisioningClientSecret`: Shared secret (AM-encrypted)
   - `provisioningSigningKeyAlias` / `provisioningEncryptionKeyAlias`: JWT signing/encryption keys
   - `useInternalOAuth2Provider=true`: AM uses its own root realm OAuth2 provider

2. **Root realm idm-provisioning OAuth2 client**:
   - Must have `client_credentials` grant type (auto-created entry only has `authorization_code`)
   - Must have `fr:idm:*` scope
   - Password must match `provisioningClientSecret` in the IDM Integration Service

3. **IDM rsFilter authentication** (the Platform mode switch):
   - IDM's `authentication.json` must use `rsFilter` instead of `serverAuthContext`
   - rsFilter calls AM's `/oauth2/introspect` endpoint to validate bearer tokens
   - Requires a second OAuth2 client (`idm-resource-server`) with `am-introspect-all-tokens` scope
   - `staticUserMapping` maps `(age!idm-provisioning)` → `internal/user/idm-provisioning` with `platform-provisioning` role

In a ForgeRock Platform deployment, the installer configures all three layers automatically. In a manual Docker setup, each layer is a separate configuration task.

---

## Q14: What is the difference between IDM's `serverAuthContext` and `rsFilter` authentication?

**Answer:**

**serverAuthContext** (standalone mode):
- Uses `authModules` array: STATIC_USER, MANAGED_USER, etc.
- Authenticates via `X-OpenIDM-Username` / `X-OpenIDM-Password` headers
- IDM handles its own authentication — no dependency on AM
- Default in a fresh IDM installation

**rsFilter** (Platform/integrated mode):
- Replaces serverAuthContext entirely
- All requests must carry `Authorization: Bearer <token>` header
- IDM calls AM's token introspection endpoint to validate tokens
- `staticUserMapping` maps AM subjects (e.g., `(age!idm-provisioning)`, `(usr!amadmin)`) to IDM internal users/roles
- `subjectMapping` maps regular user tokens to `managed/user` entries
- Username/password auth headers stop working — including `openidm-admin`

**When to use which:**
- Standalone IDM (no AM): `serverAuthContext`
- ForgeRock Platform (AM + IDM integrated): `rsFilter`
- Switching from standalone to rsFilter is a breaking change — must be planned

---

## Q15: Why does AM auto-create the idm-provisioning agent, and why is it problematic in manual setups?

**Answer:**

When you configure the IDM Integration Service (Global Services), AM automatically creates an `idm-provisioning` entry under AgentService in the root realm. This auto-creation has several issues in manual setups:

1. **Incomplete grant types**: The auto-created entry only has `authorization_code` grant type, missing `client_credentials` which is required for the internal OAuth2 flow.

2. **agentonly identity conflict**: The entry creates an internal identity of type `agentonly`. This blocks creating a proper `OAuth2Client` with the same name through the AM Console or REST API — you get "Identity of type agentonly already exists."

3. **Console "Not found" error**: The entry appears in the OAuth2 Clients listing but returns "Not found error" when you click on it, because the console's detail page uses the OAuth2Client-specific REST path which returns 404 for agentonly identities.

4. **Auto-recreation on delete**: If you delete the DS entry, the IDM Integration Service re-creates it automatically. Must modify in place, not delete-and-recreate.

5. **ldapmodify is the only fix**: Since console and REST API can't manage the entry, the only option is direct DS manipulation via ldapmodify, followed by an AM restart to flush the Guava cache.

**Fix command:**
```bash
# Write LDIF inside container (heredoc through docker exec gets mangled by Git Bash)
docker.exe exec pingds bash -c 'echo "dn: ou=idm-provisioning,...,ou=am-config
changetype: modify
add: sunKeyValue
sunKeyValue: com.forgerock.openam.oauth2provider.grantTypes=[1]=client_credentials" > /tmp/mod.ldif && \
/opt/opendj/bin/ldapmodify --hostname localhost --port 1636 --useSsl --trustAll \
  --bindDN "cn=Directory Manager" --bindPassword "Passw0rd123" /tmp/mod.ldif'
```
