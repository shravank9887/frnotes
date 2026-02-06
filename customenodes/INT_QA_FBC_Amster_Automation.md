# Interview Q&A — FBC, Amster Automation & Configuration Management

---

## Part 1: File-Based Configuration (FBC)

### Q1: What is File-Based Configuration (FBC) in PingAM and why was it introduced?

**A:** FBC lets AM read its entire configuration from JSON files on the local filesystem instead of from DS. It was introduced in AM 7.4 for cloud-native deployments where you want immutable containers — config baked into a Docker image at build time, version-controlled in Git, and promoted through CI/CD. No DS dependency for config means one fewer moving part and no risk of runtime config drift.

---

### Q2: Explain the evolution of AM configuration storage.

**A:**
- **AM 6.x** — Config in `.properties` files + embedded OpenDJ. Simple but tightly coupled.
- **AM 7.0** — Config moved to external DS (`am-config` store). Centralized, shared across AM cluster. But DS becomes a dependency for every config read.
- **AM 7.4+** — FBC introduced. Config as static JSON files on the filesystem. DS only needed for identity data and CTS.
- **PingAM 8.0** — Both modes supported. DS-based for traditional, FBC for Kubernetes/cloud-native.

The trend: decouple config from runtime state, make config immutable and version-controllable.

---

### Q3: How do you enable FBC mode in PingAM?

**A:** Two JVM system properties set in `CATALINA_OPTS`:

```
-Dcom.sun.identity.configuration.directory=/opt/am/config
-Dorg.forgerock.openam.configuration.mode=file
```

- `configuration.directory` — path to the JSON config directory
- `configuration.mode=file` — switches from DS-based to file-based (default is `ds`)

Then place the exported config JSON files at that path. AM reads them on startup and never contacts DS for configuration.

---

### Q4: What does the FBC directory structure look like?

**A:**
```
/opt/am/config/
├── global/                    # Server-wide services
│   ├── Authentication.json
│   ├── CtsDataStoreProperties.json
│   └── ...
├── realms/
│   ├── root/                  # Root realm config
│   │   ├── OAuth2Provider.json
│   │   └── ...
│   ├── root-techcorp/         # /techcorp realm
│   │   ├── OAuth2Clients/
│   │   │   └── techcorp-app.json
│   │   ├── AuthTree/
│   │   │   ├── TechCorpLogin.json
│   │   │   └── nodes/
│   │   └── ...
│   └── ...
└── boot.json                  # Bootstrap (FQDN, DS connections)
```

Each JSON file represents one REST endpoint entity. Same structure as Amster export. Realm naming is flattened with dashes (`/techcorp` → `root-techcorp`).

---

### Q5: What are the limitations of FBC?

**A:**
- **Read-only at runtime** — Cannot change config via Console, REST API, or Amster while AM is running
- **Console becomes view-only** — Can browse config but all edit buttons are disabled
- **Restart required** — Config changes need image rebuild and redeployment
- **No dynamic service creation** — Can't add OAuth2 clients or trees via REST API at runtime
- **Boot config needs special handling** — `boot.json` contains FQDN and DS connection info that varies per environment

These are intentional — immutability prevents config drift and ensures all changes go through Git + CI/CD.

---

### Q6: What still requires DS even when using FBC?

**A:**

| Still in DS | Why |
|-------------|-----|
| User identities (`am-identity-store`) | Dynamic data, not config |
| Sessions/CTS (`am-cts`) | Runtime state |
| OAuth2 tokens | Runtime state, created/revoked dynamically |
| SAML assertions | Runtime federation state |

FBC replaces the `am-config` DS store only. AM still needs DS for identity and token storage. So you go from 3 DS profiles (am-config + am-identity-store + am-cts) to 2 (am-identity-store + am-cts).

---

### Q7: How do configuration expressions work in FBC?

**A:** Syntax: `&{ENV_VARIABLE|default_value}`

```json
{
  "redirectionUris": ["&{OAUTH_CALLBACK_URL|http://localhost:3000/callback}"],
  "clientName": ["&{APP_NAME|TechCorp App}"]
}
```

Resolved at AM startup from environment variables. Same config files, different environments:

```bash
# Dev
docker run -e OAUTH_CALLBACK_URL=http://localhost:3000/callback ...
# Production
docker run -e OAUTH_CALLBACK_URL=https://app.techcorp.com/callback ...
```

**Parameterize:** URLs, hostnames, ports, passwords, scope lists — anything that varies per environment.
**Don't parameterize:** Tree structure, policy logic, resource type patterns — structural config that's the same everywhere.

---

### Q8: Describe the complete CI/CD workflow with FBC.

**A:**
1. **Developer** makes config changes in dev AM Console (DS-based dev instance)
2. **Amster export** captures changes as JSON files
3. **Replace** hardcoded values with `&{expressions}` for environment-specific settings
4. **Git commit + push** — config is version-controlled, reviewable, auditable
5. **CI pipeline triggers** — Docker build bakes config into AM image
6. **Automated tests** — smoke test AM startup, verify trees load, OAuth2 works
7. **Push image** to container registry with version tag
8. **Deploy to staging** with staging env vars — verify
9. **Deploy to production** with prod env vars

Config changes follow the same review process as code changes (PR review, approvals, audit trail).

---

### Q9: FBC vs DS-based — when do you choose each?

**A:**

| Scenario | Choice | Why |
|----------|--------|-----|
| Local dev / learning | DS-based + Console | Need to experiment interactively |
| Traditional on-prem | DS-based + Amster | Ops teams expect mutable config |
| Kubernetes / cloud-native | FBC | Immutable containers, GitOps |
| ForgeRock Identity Cloud | FBC (managed) | ForgeRock handles internally |
| Hybrid (some mutable) | DS-based + Amster CI/CD | Best of both worlds |

Rule of thumb: If you're deploying to Kubernetes with a CI/CD pipeline, use FBC. If ops teams need to make emergency config changes via Console, use DS-based.

---

## Part 2: Amster Automation Deep Dive

### Q10: How does Amster connect to AM and what authentication methods does it support?

**A:** Two methods:

**Password authentication** (interactive/dev):
```
am> connect --interactive http://pingam:8081/am
```
Prompts for amadmin password. Gets a session token. Good for development.

**RSA key authentication** (CI/CD):
```
am> connect http://pingam:8081/am -k /path/to/amster_rsa
```
Uses a pre-registered RSA private key. No password needed. The public key is registered in AM under Configure → Global Services → Amster. Required for non-interactive pipelines.

Amster connects via REST API (not LDAP, not direct DS). Must use AM's configured FQDN — `localhost` gets rejected if AM's site URL is different.

---

### Q11: What's the difference between Amster export and FBC config?

**A:** They produce the **same JSON format and directory structure**. The difference is how they're used:

| | Amster Export | FBC Config |
|---|---|---|
| **Source** | Exported from running AM | Same files, placed on filesystem |
| **Used for** | Config promotion, backup, diff | AM startup in file mode |
| **Runtime** | Imported into DS-based AM | Read directly by FBC-mode AM |
| **Mutable** | Yes (import updates DS) | No (read-only filesystem) |

Typical workflow: Amster export → add expressions → commit to Git → bake into Docker image as FBC config.

---

### Q12: Write an Amster script for automated config export with error handling.

**A:**
```groovy
// export-techcorp.amster
import org.forgerock.openam.amster.AmsterException

def amUrl = "&{AM_URL|http://pingam:8081/am}"
def exportPath = "&{EXPORT_PATH|/tmp/amster-export}"
def keyPath = "&{AMSTER_KEY|/opt/amster/amster_rsa}"

try {
    connect amUrl -k keyPath
    println "Connected to ${amUrl}"

    export-config --path exportPath
    println "Export completed to ${exportPath}"

} catch (Exception e) {
    println "ERROR: ${e.message}"
    System.exit(1)
}
```

Run non-interactively:
```bash
./amster export-techcorp.amster
```

---

### Q13: How do you handle secrets in Amster exports and FBC?

**A:** Amster exports **do not include**:
- User passwords
- Client secrets (hashed, not reversible)
- Private keys
- DS bind passwords

For FBC, secrets are handled via:

1. **Environment variables with expressions**: `&{CLIENT_SECRET}` — injected at runtime
2. **Kubernetes Secrets**: Mounted as env vars or files
3. **AM Secret Stores**: FileSystemSecretStore, KeyStoreSecretStore, or HSM-backed
4. **Vault integration**: HashiCorp Vault via AM's secret API

**Never** commit plaintext secrets to Git. Use expressions + external secret management.

---

### Q14: Explain Amster's import-config idempotency and why it matters.

**A:** `import-config` creates entities that don't exist and updates entities that do. It doesn't fail on duplicates or delete entities not in the import set.

Why this matters:
- **Safe on live AM** — won't break existing config
- **Repeatable** — run the same import multiple times with same result
- **Incremental** — import just the changed files, not everything
- **No restart needed** — changes take effect immediately (DS-based mode)

This makes it safe to use in CI/CD pipelines — a failed pipeline can be re-run without side effects.

---

### Q15: How do you promote config from dev → staging → production using Amster?

**A:**

**Step 1 — Dev**: Make changes in AM Console, export with Amster:
```bash
./amster export-config --path ./config-export
```

**Step 2 — Parameterize**: Replace environment-specific values:
```json
// Before (dev)
"redirectionUris": ["http://localhost:3000/callback"]

// After (parameterized)
"redirectionUris": ["&{CALLBACK_URL|http://localhost:3000/callback}"]
```

**Step 3 — Git**: Commit, create PR, review the diff, merge.

**Step 4 — Import to staging** (Amster script):
```groovy
connect http://staging-am:8080/am -k /opt/keys/amster_rsa
import-config --path /opt/config-export
```
Staging AM resolves `&{CALLBACK_URL}` from its environment.

**Step 5 — Test staging**, then repeat import for production.

**For FBC deployments**: Skip the import step — bake config into Docker image and deploy with per-environment env vars.

---

### Q16: What is the `--realm` flag behavior in Amster export and why is it surprising?

**A:** `export-config --realm /techcorp` still exports **everything** — global config and all realms. It doesn't filter. This is because AM config has cross-realm dependencies (global services, shared identity stores, CTS config).

To get realm-specific config, export everything and then manually pick the files you need from `realms/root-techcorp/`. Most CI/CD pipelines export the full config and version-control all of it.

---

### Q17: How do you set up RSA key authentication for Amster in a CI/CD pipeline?

**A:**

**Step 1 — Generate key pair:**
```bash
ssh-keygen -t rsa -b 2048 -f amster_rsa -N ""
```

**Step 2 — Register public key in AM:**
AM Console → Configure → Global Services → Amster → Add the public key content

**Step 3 — Store private key as CI/CD secret:**
- Jenkins: Credentials store
- GitLab CI: CI/CD Variables (masked, file type)
- GitHub Actions: Repository secrets

**Step 4 — Use in pipeline:**
```yaml
# GitLab CI example
deploy-config:
  script:
    - echo "$AMSTER_PRIVATE_KEY" > /tmp/amster_rsa
    - chmod 600 /tmp/amster_rsa
    - ./amster import-script.amster
```

---

### Q18: How does Amster handle configuration dependencies during import?

**A:** Amster processes imports in dependency order automatically:
1. Global services first (authentication, CTS, identity stores)
2. Realm creation
3. Realm services (OAuth2 Provider, SAML)
4. Collections within services (OAuth2 clients, SAML entities)
5. Trees and node instances
6. Policies (depend on policy sets, which depend on resource types)

If you import a partial set (e.g., just one OAuth2 client), the parent service must already exist in AM. Import fails if dependencies are missing.

---

## Part 3: Production Patterns & Architecture

### Q19: Compare all three config management approaches for a production deployment.

**A:**

| | Console + Amster | Amster CI/CD | FBC |
|---|---|---|---|
| **Config store** | DS | DS | Filesystem |
| **Change method** | Console → export → Git | Git → import | Git → Docker build |
| **Runtime mutable** | Yes | Yes | No |
| **Emergency fix** | Console edit | Amster import | Rebuild + redeploy |
| **Audit trail** | AM audit log | Git history + AM audit | Git history only |
| **Rollback** | Amster re-import | Git revert + import | kubectl rollout undo |
| **Complexity** | Low | Medium | High (initial), Low (ongoing) |
| **Best for** | Small/dev | Traditional production | Kubernetes/cloud |

---

### Q20: How would you design a complete AM configuration management system for an enterprise?

**A:** "I'd design a three-stage pipeline:

**Source of truth**: Git repository with Amster-exported JSON config files. All config changes start as PRs with peer review.

**Parameterization**: Environment-specific values use `&{expressions}`. Secrets reference external stores (Vault, K8s Secrets), never committed to Git.

**Pipeline stages**:
1. **Build** — Validate JSON syntax, check for hardcoded secrets (lint), run Amster dry-run import against a disposable AM instance
2. **Staging deploy** — Import config (Amster) or build FBC image (Docker). Run automated smoke tests: authenticate user, get OAuth2 token, evaluate policy, verify SAML SSO
3. **Production deploy** — Same image/config, different env vars. Blue-green deployment for zero downtime. Automated health checks before switching traffic.

**Rollback**: Git revert + re-run pipeline (Amster) or `kubectl rollout undo` (FBC/K8s).

**Monitoring**: Alert on config drift (compare running config hash to Git). AM audit logs to SIEM. OAuth2 token error rates as canary.

For a Kubernetes deployment, I'd use FBC with config baked into the image. For traditional VM-based, I'd use DS-based with Amster CI/CD import."

---

### Q21: What is the relationship between Amster, FBC, and ForgeOps/CDK?

**A:**
- **Amster** — CLI tool for exporting/importing config. Works with DS-based AM. Used for config promotion and backup.
- **FBC** — AM runtime mode that reads config from filesystem instead of DS. The JSON files come from Amster export.
- **ForgeOps/CDK** — ForgeRock's Kubernetes deployment toolkit. Uses FBC internally. Bakes config into Docker images using a custom build process. Adds Kubernetes manifests (Helm charts, Kustomize overlays) for deployment.

The relationship:
```
Amster export → JSON files → FBC config → Docker image → ForgeOps/CDK → Kubernetes
```

ForgeOps/CDK is the highest-level abstraction. It uses Amster and FBC under the hood but adds Kubernetes-native tooling.

---

### Q22: How do you handle tree/journey config promotion when nodes have UUIDs?

**A:** Each node instance in a tree has a UUID identifier. When you export a tree, you get:
- Tree JSON with `entryNodeId` and a `nodes` map (UUID → node type + connections)
- Node instance JSONs in UUID-named directories (actual config values)

For promotion:
- **Same UUIDs work** — Amster import uses the UUID to find/create the node. If the UUID doesn't exist in the target, it creates it. If it exists, it updates it.
- **Don't regenerate UUIDs** — Export from dev, import to staging/prod as-is. The UUIDs are the link between tree wiring and node config.
- **Adding a node in dev** — Creates new UUID. Export captures it. Import to staging creates it there too.
- **Deleting a node in dev** — The old UUID files remain in target unless manually cleaned. Orphaned nodes don't cause errors but waste space.

---

### Q23: What's the boot.json file and why does it need special handling?

**A:** `boot.json` contains bootstrap configuration that AM needs before it can read any other config:
- AM's FQDN / Site URL
- DS connection details (host, port, bind DN, password)
- Encryption key
- Platform mode settings

It needs special handling because:
- **Every environment has different values** (different DS hosts, different FQDNs)
- **Contains secrets** (DS bind password, encryption key)
- **Must be correct before AM can start** — if boot.json is wrong, AM can't read the rest of the config

Solution: Heavy use of `&{expressions}` in boot.json, with all values injected via environment variables or Kubernetes secrets.

---

### Q24: In an interview, how would you explain the full lifecycle of a configuration change in a production FBC deployment?

**A:** "Let's say we need to add a new OAuth2 client.

1. **Developer** creates the client in the dev AM Console (dev runs DS-based for interactive config)
2. **Amster export** — captures the new client as a JSON file in `realms/root-techcorp/OAuth2Clients/new-client.json`
3. **Parameterize** — replace the redirect URI and client secret with `&{NEW_CLIENT_REDIRECT}` and `&{NEW_CLIENT_SECRET}`
4. **PR review** — team reviews the JSON diff in Git. Checks scopes, grant types, token lifetimes
5. **Merge** — triggers CI pipeline
6. **Docker build** — new AM image with the client config baked in
7. **Staging deploy** — Kubernetes rolling update with staging env vars. Automated test: obtain token with new client, verify scopes
8. **Production deploy** — same image, production env vars. Blue-green switch. Monitor token error rates for 15 minutes
9. **Done** — new client is live. Config is in Git, auditable, rollback-able via `kubectl rollout undo`

The client was never created via Console or REST API in production. Production AM is FBC mode — read-only. All changes flow through Git."

---

### Q25: What happens if you need an emergency config fix in production with FBC?

**A:** This is the trade-off of immutability. Options:

1. **Fast-track the pipeline** — make the fix, skip staging, deploy directly to prod. Still goes through Git (emergency PR, skip full review). Takes minutes if pipeline is fast.
2. **Rollback** — if the issue is a recent change, `kubectl rollout undo` reverts to the previous image (previous config) instantly.
3. **Escape hatch** — temporarily switch AM to DS-based mode (change JVM property, restart). Make the fix via Console. Then export, commit to Git, rebuild image, switch back to FBC. This is a last resort.

The discipline required: your CI/CD pipeline must be fast enough that emergency fixes don't tempt people to bypass it.

---

*Created: 2026-02-02 — Covers FBC architecture, enabling FBC, directory structure, expressions, CI/CD workflow, Amster automation, RSA key auth, config promotion, production patterns, ForgeOps/CDK relationship, boot.json, and emergency procedures.*
