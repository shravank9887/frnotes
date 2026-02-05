# Interview Q&A: Amster CLI — Hands-On Configuration Management

*Generated from Lab 13 hands-on experience with Amster 8.0.2 against PingAM 8.0.2*

---

## Q1: What is Amster and how does it connect to AM?

**A:** Amster is a command-line tool for managing PingAM configuration. It connects via AM's **REST API** (not LDAP, not direct DS access). You authenticate as `amadmin` (either password-interactive or RSA private key) and Amster gets a session token which it uses for all subsequent REST calls.

**Hands-on detail:** We used `connect -i http://pingam:8081/am` — the `-i` flag means interactive password mode. Amster must connect using AM's configured FQDN (Site URL). Using `localhost` failed because AM redirects to its Docker hostname `pingam`. In production, this would be `am.example.com` behind a load balancer.

---

## Q2: What does `export-config` produce and how is the output structured?

**A:** `export-config --path <dir>` exports AM's entire configuration as JSON files into two top-level directories:

- **`global/`** — Server-wide settings (Configure → Global Services in console): authentication modules, audit, CORS, secret stores, scripts, etc.
- **`realms/`** — Per-realm configuration. Each realm is a subdirectory named by path: `root` (top-level), `root-techcorp` (/techcorp), `root-partner` (/partner). Nested realms use `root-parent-child` naming.

You can also scope exports: `export-config --path <dir> --realm /techcorp` exports only that realm.

---

## Q3: What is the JSON file structure and how do you read an Amster export file?

**A:** Every exported JSON file has the same two-part structure:

```json
{
  "metadata": {
    "realm": "/techcorp",
    "amsterVersion": "8.0.2",
    "entityType": "AuthTree",
    "entityId": "TechCorpLogin",
    "pathParams": {}
  },
  "data": {
    "_id": "TechCorpLogin",
    ...config fields...,
    "_type": {
      "_id": "...",
      "name": "Authentication Trees",
      "collection": true
    }
  }
}
```

**Reading strategy:**
1. **`metadata.entityType` + `entityId`** → tells Amster which REST endpoint to PUT on import
2. **`data._type.name`** → human-readable name (what you see in AM Console)
3. **`data._type.collection`** → `false` = singleton service (one per realm), `true` = multiple instances
4. **`data._id`** → specific instance identifier
5. **Config sections** map to AM Console tabs (e.g., `coreOAuth2ClientConfig` → Core tab, `advancedOAuth2ClientConfig` → Advanced tab)

**Key insight:** The `data` section is exactly what AM's REST API returns for a GET on that resource. Amster is essentially a REST client with file serialization.

---

## Q4: What's the difference between standalone JSON files and directories in the export?

**A:** This maps to AM's internal distinction between **singleton services** and **collections**:

- **Standalone `.json` files** (e.g., `Authentication.json`, `OAuth2Provider.json`, `EmailService.json`) → **Singleton services** (`collection: false`). One instance per realm. Maps to Realm → Services in the console.

- **Directories with JSON files inside** (e.g., `AuthTree/`, `OAuth2Clients/`, `Policies/`) → **Collections** (`collection: true`). Multiple instances, each with its own JSON file named by `_id`.

- **UUID-named directories** (e.g., `DataStoreDecision/`, `PageNode/`) → **Node instance configs**. These are individual authentication tree node instances, named by UUID because AM assigns IDs internally. They're not visible as a separate list in the console — they're embedded inside tree definitions.

---

## Q5: How do authentication tree exports work? What's the relationship between tree JSON and node JSON?

**A:** Trees are exported as two kinds of files:

1. **Tree definition** (`AuthTree/TechCorpLogin.json`) — Contains:
   - `entryNodeId`: UUID of the first node
   - `nodes`: map of UUID → `{displayName, nodeType, x, y, connections}` — this is the **wiring**
   - `staticNodes`: Success/Failure endpoint positions
   - The `connections` field maps outcome names to the UUID of the next node

2. **Node instance configs** (e.g., `DataStoreDecision/c81e728d-....json`) — Contains the node's **configuration** (what you'd see when clicking the node in the tree designer).

The tree JSON defines **how nodes connect**; the separate node files define **what each node does**. This separation exists because multiple trees can share the same node instance (by UUID reference).

---

## Q6: What does Amster export vs NOT export?

**A:**

**Exported (configuration):**
- Authentication trees, chains, node instances
- OAuth2/OIDC provider settings and clients
- SAML2 entity providers, circles of trust
- Policies, policy sets, resource types
- Realm services (Email, WebAuthn, OATH, etc.)
- Scripts
- Global services, secret store configs
- Agent configurations

**NOT exported (operational data):**
- Users/identities (data in identity store, not config)
- Sessions (transient, in CTS)
- OAuth2 tokens (transient, in CTS)
- Audit logs (operational data)
- Passwords/secrets in plaintext (exported as encrypted references or null)
- CTS token data

**Interview phrasing:** "Amster exports configuration, not data. Users, sessions, and tokens are runtime data in DS — they're managed by replication and backup, not by Amster."

---

## Q7: How do you use Amster for a dev workflow?

**A:** The standard workflow is:

1. **Develop** — Make changes in AM Console (create trees, configure OAuth2 clients, etc.)
2. **Export** — `export-config --path ./config --realm /techcorp`
3. **Diff** — `git diff` to see exactly what changed
4. **Commit** — Check the JSON changes into version control
5. **Promote** — In CI/CD, `import-config --path ./config` into staging/prod AM

This gives you full version control over AM config. You can review changes in PRs, rollback by reverting commits, and audit who changed what.

---

## Q8: What is `import-config` and is it idempotent?

**A:** `import-config --path <dir>` reads all JSON files from the directory and PUTs each one to AM's REST API. It is **idempotent** — creates if missing, updates if exists. Running it twice with the same files produces the same result.

This makes it safe for CI/CD pipelines — you can always re-run an import without side effects.

---

## Q9: What are Amster scripts (.amster files)?

**A:** Amster scripts are **Groovy files** with an `.amster` extension. They contain Amster commands as Groovy code:

```groovy
connect -i http://pingam:8081/am
export-config --path ./exports/techcorp --realm /techcorp
:exit
```

Run with: `amster scripts/export-techcorp.amster`

Used in CI/CD pipelines for automated config promotion. Can include Groovy logic (variables, loops, conditionals) for complex operations.

---

## Q10: What are configuration expressions and why do they matter?

**A:** Configuration expressions use the syntax `&{variable|default}` in exported JSON files:

```json
"redirectionUris": ["&{am.base.url|http://localhost:8081}/am/oauth2/callback"]
```

Amster resolves these at import time. This enables **environment-specific config from the same JSON files** — dev uses `localhost:8081`, staging uses `am.staging.example.com`, prod uses `am.example.com`. You maintain one set of config files and parameterize the differences.

---

## Q11: How does Amster fit in AM upgrades?

**A:**

- **Pre-upgrade**: `export-config` as a backup of all config. If upgrade fails, you have a JSON snapshot to restore from.
- **Post-upgrade**: `import-config` to restore or promote config to the new AM version. Amster handles schema differences between versions.
- **Cross-version**: Export from AM 7.x, import to AM 8.x. Amster maps old config format to new format where possible.
- **Config expressions**: Replace environment-specific values (URLs, hostnames) so the same export works across dev/staging/prod.

**Interview phrasing:** "We use Amster to export config before upgrades as a backup, and to promote config across environments after testing in staging."

---

## Q12: What is File-Based Configuration (FBC) and how does it relate to Amster?

**A:** FBC is PingAM's approach to storing configuration as JSON files on the filesystem instead of in DS.

**History:**
- AM 6.x: File-based `.properties` config
- AM 7.0-7.3: DS-based config (all config moved to Directory Server)
- AM 7.4+: FBC introduced (JSON config on filesystem, version-controllable)
- PingAM 8.x: FBC is the recommended approach for cloud-native/immutable deployments

**FBC vs Amster:**
- **FBC**: Config lives on filesystem, baked into Docker images for immutable deployments. No runtime config changes.
- **Amster**: Config lives in DS, Amster imports/exports via REST API. Supports runtime changes.
- **When to use which**: FBC for Kubernetes/cloud-native (immutable containers). Amster for traditional deployments or as a bridge during migration. Both support version control.

---

## Q13: How does Amster compare to direct REST API calls and the AM Console?

**A:**

| Approach | Best For |
|----------|----------|
| **AM Console** | Interactive exploration, one-off changes, learning |
| **REST API (curl)** | Targeted automation, scripting specific operations, debugging |
| **Amster** | Bulk config management, full exports, environment promotion, CI/CD |

Amster adds value over raw REST because it handles the full dependency graph (exports/imports in correct order), manages file serialization, and supports expressions for parameterization.

---

## Q14: What does the realm naming convention look like in exports?

**A:** Amster flattens the realm hierarchy into directory names using dashes:

| Realm Path | Export Directory |
|---|---|
| `/` (root) | `realms/root` |
| `/techcorp` | `realms/root-techcorp` |
| `/partner` | `realms/root-partner` |
| `/techcorp/sub` | `realms/root-techcorp-sub` |

This flat structure avoids filesystem issues with nested directories while preserving the full realm path.

---

## Q15: What's the relationship between `entityType` in metadata and the AM REST API?

**A:** `entityType` maps directly to an AM REST endpoint path segment. Examples:

| entityType | REST Endpoint |
|---|---|
| `AuthTree` | `/realm-config/authentication/authenticationtrees/trees/{id}` |
| `OAuth2Clients` | `/realm-config/agents/OAuth2Client/{id}` |
| `Policies` | `/json/policies/{id}` |
| `OAuth2Provider` | `/realm-config/services/oauth-oidc` |
| `Authentication` | `/realm-config/authentication` |
| `Saml2Entity` | `/realm-config/federation/entityproviders/{id}` |

Each JSON file = one PUT request to one REST endpoint. This is why Amster is "just a REST client with file I/O."

---

## Q16: Does `export-config --realm /techcorp` only export that realm?

**A:** No. In Amster 8.x, `--realm` on `export-config` still exports the **full configuration** — all global config and all realms. This is by design: config has cross-realm dependencies (e.g., a policy references a global resource type, a SAML entity in `/techcorp` links to a remote entity in `/partner`). Exporting just one realm would produce an incomplete set that could fail on import.

**Practical implication:** If you want to diff only one realm's changes, diff the `realms/root-techcorp/` subdirectory and ignore the rest.

---

## Q17: What are all the Amster commands?

**A:** Two categories:

**Bulk operations** (most commonly used):
- `connect` — authenticate (`-i` password, `-k` private key)
- `export-config` — export all config to JSON files
- `import-config` — import all config from JSON files
- `:exit` — disconnect and quit

**CRUD on individual entities:**
- `create` — create a single entity (e.g., `create Realm --body '{"name":"staging"}'`)
- `read` — read a single entity
- `update` — update a single entity
- `delete` — delete a single entity
- `query` — list entities of a type (e.g., `query AuthTree --realm /techcorp`)

In practice, `export-config`/`import-config` cover 90% of usage. CRUD commands are used for specific automation like realm provisioning or post-deploy validation.

---

## Q18: How do you run Amster in CI/CD without password prompts?

**A:** Use RSA private key authentication instead of `-i` (interactive):

1. Generate RSA key pair
2. Register the public key in AM Console (Deployment → Servers → Amster)
3. Store private key as a CI secret (Jenkins credential, GitHub secret, etc.)
4. Script uses: `connect http://am.example.com/am -k /path/to/private-key`

No password prompt, fully non-interactive. The `.amster` script runs end-to-end without human input.

---

## Q19: What does a real-world config promotion pipeline look like?

**A:**
```
Dev AM ──export──> Git repo ──PR review──> merge
                                              │
Staging AM <──import── CI/CD reads from Git (with staging expressions)
                                              │
Prod AM    <──import── CI/CD (after staging tests pass, with prod expressions)
```

1. Developer makes changes in Dev AM Console
2. Runs Amster `export-config`, commits JSON diff to Git
3. Opens PR — reviewer sees exactly which trees, policies, clients changed
4. After merge, CI/CD runs `import-config` against Staging with staging-specific expressions
5. QA validates on staging
6. Same pipeline promotes to Prod with prod-specific expressions

---

## Q20: What is the import-config dependency order and why does it matter?

**A:** AM config has dependencies — a Policy references a Resource Type (by UUID), an OAuth2 Client requires the OAuth2 Provider service to exist first, a tree references node instances by UUID.

Amster handles this automatically during full imports — it knows the correct dependency order and imports entities in the right sequence. This is a key advantage over raw REST API calls, where you'd have to manage the order yourself.

---

## Q21: How does Amster handle secrets and passwords in exports?

**A:** Passwords are exported as `null` or encrypted references — never plaintext. For example, the OAuth2 client's `userpassword` field exports as `null`.

When importing to a new environment, you may need to:
- Re-set client secrets manually via Console or REST API
- Use AM's Secret Store integration (KeyStore, Vault, etc.) instead of inline passwords
- For CI/CD, set secrets via a separate secure mechanism after config import

---

## Q22: What is the FBC config storage evolution and why does it matter for upgrades?

**A:**

| AM Version | Config Storage | Migration Path |
|---|---|---|
| AM 5.x-6.x | Flat files (.properties, XML) | Manual conversion |
| AM 7.0-7.3 | DS (LDAP) | Auto-migrated on upgrade from 6.x |
| AM 7.4+ | FBC option (JSON filesystem) | Amster export → convert to FBC format |
| PingAM 8.x | FBC recommended for cloud | Native support |

The biggest breaking change was AM 6 → AM 7 (embedded DS removed, config moved to DS). AM 7.4+ introduced FBC as the path toward immutable cloud-native deployments. Amster serves as the bridge — export from DS-based config, convert to FBC format.

---

## Q23: What are the upgrade risks and how does Amster help mitigate them?

**A:**

**Risks:**
- Deprecated auth chains removed (must migrate to trees before upgrade)
- Custom nodes compiled against old API (must recompile)
- DS schema upgrade is one-way (can't downgrade)
- Advanced server properties may move to new locations

**Amster mitigation:**
- Pre-upgrade `export-config` = complete JSON backup
- Cross-version import handles most schema changes automatically
- If upgrade fails: restore DS from LDIF backup + redeploy old WAR + Amster import
- In Kubernetes: `kubectl rollout undo` (if using FBC with immutable images)

---

## Q24: How do you answer "How do you manage AM configuration in your project?"

**A:** The strong answer covers all three approaches:

> "For day-to-day development, we use the AM Console. For config promotion across environments, we use Amster with CI/CD — export from dev, commit to Git, import to staging/prod with configuration expressions for environment-specific values. For Kubernetes deployments, we're moving toward FBC where config is baked into immutable container images. Amster serves as the bridge during that transition."

This demonstrates understanding of the full spectrum — interactive, automated, and cloud-native.

---

## Q25: What is static config vs dynamic config in PingAM?

**A:** AM configuration falls into two categories based on **who changes it and when**:

**Static config (boot-time, set once):**
- Server identity (FQDN, site URL, server ID)
- DS connection strings (am-config, am-cts, am-identity-store endpoints)
- JVM/container settings (heap, GC)
- Encryption keys, keystore paths
- Deployment mode (DS-based vs FBC)

These are read **once at startup** and never change while AM runs. They've always been file-based (`boot.properties`, server config XML, environment variables) — even in DS-based deployments.

**Dynamic config (runtime, changed by admins):**
- Realms
- Authentication trees and node instances
- OAuth2 Provider settings and clients
- SAML2 entities, circles of trust
- Policies, resource types, policy sets
- Scripts, services (Email, WebAuthn, OATH, etc.)

In DS-based mode, all dynamic config lives in the `am-config` DS backend. When you click Save in the AM Console, it writes to DS via LDAP.

**The key insight:** The distinction isn't about the data itself — it's about who changes it and when. Static = set at deploy time, never touched. Dynamic = changed by admins/automation during the system's lifetime.

---

## Q26: Does FBC handle both static and dynamic config?

**A:** Yes, but with a fundamental trade-off: **FBC converts dynamic config into effectively static config.**

In DS-based mode:
- Static config → filesystem (`boot.properties`)
- Dynamic config → DS (`am-config`) — writable at runtime

In FBC mode:
- Static config → filesystem (`boot.properties`)
- Dynamic config → filesystem (JSON files) — **read-only at runtime**

FBC moves all the realm config, trees, clients, policies — everything that currently lives in `am-config` DS — to JSON files on the filesystem. But in an immutable container, those files are read-only. The AM Console becomes a read-only viewer or isn't used at all.

**To change anything in FBC mode:** edit files in Git → rebuild container image → redeploy. No one can make ad-hoc changes to prod by clicking in the Console. This is intentional — immutable infrastructure.

**What still needs DS even with FBC:**
- `am-identity-store` — users/identities (data, not config)
- `am-cts` — sessions, OAuth2 tokens (runtime data)
- Secret rotation — handled by external secret stores (Vault, KMS), not AM config

FBC only replaces the `am-config` DS backend. The other two backends remain regardless.

---

## Q27: How do you install PingAM from scratch in FBC mode?

**A:** You can't start directly in FBC mode because config needs to exist first. The general approach:

**Step 1 — Generate initial config (two approaches):**

*Approach A — DS-based first, then convert:*
1. Install AM normally with DS-based config (traditional setup)
2. Configure everything via Console (trees, clients, policies)
3. Export with Amster: `export-config --path ./fbc-config`
4. Restructure exported JSON into AM's FBC directory layout
5. Switch to FBC mode

*Approach B — ForgeOps/CDK reference images:*
1. Start from ForgeRock Platform's reference Docker images (already FBC-structured)
2. Overlay your custom config files
3. Build and deploy

**Step 2 — Tell AM to use filesystem for config:**

In `boot.properties` or environment variables, set the config store to FILE mode and point to the config directory on the filesystem instead of DS.

**Step 3 — Eliminate am-config DS backend:**

With FBC, the DS setup simplifies:

| DS-based mode | FBC mode |
|---|---|
| am-config (config store) | REMOVED — replaced by files |
| am-identity-store | am-identity-store (stays) |
| am-cts | am-cts (stays) |

**Step 4 — Bake config into container image:**

```dockerfile
FROM pingam:8.0.2
COPY fbc-config/ /opt/am/config/
# AM reads config from filesystem at boot — no DS config store needed
```

**Step 5 — Change management becomes Git-native:**

```
Edit JSON files in Git → CI/CD builds new image → Deploy new container
```

No Amster needed. Config changes are Git commits. Rollback is `git revert` + redeploy.

---

## Q28: What's the comparison between DS-based config and FBC mode?

**A:**

| Aspect | DS-based (traditional) | FBC (cloud-native) |
|---|---|---|
| Config store | DS `am-config` backend | JSON files on filesystem |
| AM Console | Read/write | Read-only or unused |
| Change process | Click Save → writes to DS | Edit files → rebuild image → redeploy |
| Amster role | Export/import for promotion | Not needed (files in Git directly) |
| DS profiles needed | am-config + am-identity-store + am-cts | am-identity-store + am-cts only |
| Runtime flexibility | High (change anything anytime) | None (immutable) |
| Audit trail | DS access logs, AM audit logs | Git commit history |
| Rollback | Amster import or DS restore | `git revert` → rebuild → redeploy |
| Best for | Traditional VMs, dev environments, learning | Kubernetes, cloud-native, production at scale |

**Interview phrasing:** "AM has static config (boot properties, DS connections — always file-based) and dynamic config (realms, trees, clients, policies — stored in DS in traditional mode). FBC moves dynamic config to the filesystem too, making everything file-based. The trade-off is that config becomes immutable at runtime — changes require a new container image. This is ideal for Kubernetes where you want reproducible deployments and Git as the single source of truth. In traditional deployments, DS-based config with Amster for promotion gives you more runtime flexibility."
