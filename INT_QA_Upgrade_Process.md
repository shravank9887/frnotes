# Interview Q&A: PingAM Upgrade Process — Complete Walkthrough

*Generated from Lab 14 — practical upgrade walkthrough using a production-like environment with trees, OAuth2, SAML, policies, custom nodes, MFA*

---

## Q1: Walk me through the AM upgrade process at a high level.

**A:** Five phases:

1. **Pre-upgrade assessment** — Inventory config (Amster export), review release notes for breaking changes, identify high-risk areas (custom nodes, SAML federation)
2. **Prepare** — Full backups (Amster export + DS LDIF + DS binary), build staging environment, recompile custom nodes against new AM APIs
3. **Execute** — Upgrade DS first (schema), then AM (WAR deployment), then deploy recompiled custom nodes, then IDM if applicable
4. **Verify** — Run through verification checklist: auth trees, OAuth2 flows, SAML SSO, policy evaluation, custom node visibility
5. **Rollback plan** — If something breaks: blue-green switch, or restore DS from LDIF + redeploy old WAR

The order is critical: DS before AM (new AM needs new DS schema), AM before custom nodes (nodes deploy into AM).

---

## Q2: What is the first thing you do before any upgrade?

**A:** Inventory what you have. Run Amster `export-config` and commit to Git as a baseline snapshot:

```
connect -i http://am.prod.example.com/am
export-config --path ./pre-upgrade-inventory
:exit
```

Then analyze the export — count trees, OAuth2 clients, SAML entities, policies, custom nodes. You need to know exactly what exists so you can verify it all works after the upgrade.

Many teams skip this and discover post-upgrade that something is missing. The Amster export is your source of truth.

---

## Q3: What are the breaking changes for each upgrade path?

**A:**

### AM 6.5 → AM 7.x (MAJOR — biggest breaking change in AM history)

| Area | Breaking Change | Impact |
|---|---|---|
| Embedded DS removed | AM no longer bundles DS, must have external DS | Need separate DS infrastructure |
| Config moved to DS | File-based `.properties` → DS `am-config` backend | Must run config migration tool |
| Auth chains deprecated | Chains still work but deprecated, trees are primary | Any legacy chains need review |
| Custom node API changed | Package renames, new annotations | All custom nodes must be recompiled |
| REST API versioning | Some endpoints changed | Check all curl scripts, integrations |
| Session service changes | New session schema in CTS | CTS DS schema must be upgraded |
| SAML2 metadata format | Minor schema changes | Re-verify federation after upgrade |

### AM 7.x → AM 7.5 (MINOR — mostly additive)

| Area | Change | Impact |
|---|---|---|
| New node types | Intelligence nodes, Device Binding | No breaking change — additive |
| OAuth2 spec updates | PAR, CIBA improvements | Existing clients unaffected |
| DS 7.x required | Must upgrade DS too | DS upgrade is separate step |

### AM 7.5 → PingAM 8.0 (MODERATE)

| Area | Change | Impact |
|---|---|---|
| Tomcat 9 → Tomcat 10 | javax.* → jakarta.* namespace | Custom nodes must switch to jakarta.inject |
| Java 11 → Java 17/21 | Minimum Java version bump | Update base image |
| Rebranding | ForgeRock AM → PingAM | Package/class names may change |
| DS 7.5+ required | DS must be compatible version | DS upgrade first |

---

## Q4: Why are custom nodes the highest risk during upgrades?

**A:** Custom nodes are compiled Java code with direct dependencies on AM's internal APIs. Unlike config (which AM can auto-migrate), custom code has no automated migration path.

Specific risks:

**AM 6.5 → 7.x:** API signatures changed, package renames, some deprecated methods removed.

**AM 7.x → PingAM 8.0:** `javax.inject` → `jakarta.inject` (Tomcat 10 = Jakarta EE). Every `@Inject` annotation in every custom node must change.

**The fix process:**
1. Get new AM version's JARs (auth-node-api, openam-annotations, etc.)
2. Update `pom.xml` dependencies to new versions
3. Fix import statements and any deprecated API calls
4. Recompile: `mvn clean package`
5. Run unit tests
6. Deploy to staging and test each node in its tree

**Risk assessment by node complexity:**

| Node Type | Risk | Why |
|---|---|---|
| Simple AbstractDecisionNode | Low | Minimal API surface, boolean logic |
| Nodes using ExternalRequestContext | Medium | HTTP request API may change |
| Nodes with custom OutcomeProvider | Medium | OutcomeProvider interface may change |
| Nodes calling external APIs | Low | External calls don't depend on AM APIs |
| Nodes using AM internal services | High | Internal services are not stable API |

---

## Q5: Why is SAML federation particularly fragile during upgrades?

**A:** SAML involves **two parties** with a cryptographic trust relationship. Three things can break:

**1. Signing certificates:**
- If AM regenerates keystores during upgrade, the IdP signing cert changes
- Every SP that trusts your IdP must re-import your metadata
- Symptom: SP rejects SAML assertions — "signature validation failed"

**2. Metadata endpoints:**
- If AM's URL structure changes, metadata URLs may break
- External partners who fetch metadata dynamically will get errors

**3. Configuration details:**
- NameID Value Maps, Attribute Maps, Relay State URL Lists — these are config in DS
- They should survive if `am-config` DS is preserved, but must be verified

**Pre-upgrade SAML checklist:**
1. Export IdP metadata XML and save it
2. Export SP metadata XML
3. Document all NameID maps, attribute maps, relay state URL lists
4. Save the signing keystore separately (file-level backup)
5. After upgrade: verify metadata endpoints, test SP-initiated and IdP-initiated SSO
6. If certs changed: re-distribute metadata to all federation partners

**Interview phrasing:** "SAML is our highest-risk config during upgrades because it involves external trust relationships. If the signing certificate changes, we need to coordinate with every federation partner to re-import metadata. We always save the keystore separately and test SSO flows in staging before touching production."

---

## Q6: What backups do you take before an upgrade and why three types?

**A:**

```bash
# 1. Amster config export (human-readable, version-controllable)
amster export-config --path ./backup-am-config-$(date +%Y%m%d)

# 2. DS LDIF export (portable text backup)
export-ldif --backendName amConfig --ldifFile am-config.ldif
export-ldif --backendName amIdentityStore --ldifFile am-identity.ldif

# 3. DS binary backup (fast restore)
dsbackup create --backendName amConfig --backendName amIdentityStore --backendName amCts
```

**Why three types:**

| Backup Type | Pros | Cons |
|---|---|---|
| Amster export | Human-readable JSON, version-controllable, can import to any AM version | Missing secrets/passwords (exported as null), missing users/sessions |
| LDIF export | Portable across DS versions, includes everything (config + users), text-based | Slower restore, larger files |
| DS binary backup | Fastest restore, exact point-in-time copy | DS-version-specific, can't restore to different DS version |

You need all three because they cover different failure scenarios:
- Amster export → restore specific config entities without full DS restore
- LDIF → restore to a different DS version (if DS upgrade also fails)
- Binary → fastest recovery when DS version is unchanged

---

## Q7: What is the correct upgrade order and why?

**A:**

```
1. DS first       → schema must be compatible with new AM
2. AM second      → needs new DS schema to start
3. Custom nodes   → deploy recompiled JAR into new AM
4. IDM last       → depends on AM being stable
```

**Why DS first?** New AM may require new DS schema (indexes, object classes, attribute types). If DS has old schema, AM startup fails with schema errors. DS schema upgrade adds new elements — it's backwards compatible for the old AM, so the old AM can still run against upgraded DS during the transition.

**Why custom nodes after AM?** You need AM running with the new version to verify nodes load correctly. Also, if AM fails to start, you want to know it's an AM issue, not a custom node issue.

**CRITICAL: DS schema upgrade is ONE-WAY.** Once you upgrade DS schema, you cannot downgrade it. The only way back is restoring from LDIF backup. This is why you must take DS backups before starting.

---

## Q8: What is in-place vs blue-green (side-by-side) upgrade?

**A:**

**In-place upgrade (simpler, has downtime):**
```bash
# Stop old AM
docker stop pingam
# Deploy new AM WAR
cp AM-8.0.2.war /opt/tomcat/webapps/am.war
# Start AM — it detects old config version, runs auto-upgrade
docker start pingam
# Watch logs for "Configuration upgrade complete"
```

- Pros: Simpler, one environment
- Cons: Downtime during upgrade, rollback is harder (must restore DS + redeploy old WAR)

**Blue-green / side-by-side (zero downtime, recommended for prod):**
```
                    ┌─────────────┐
Users ──> LB ──────>│ AM 7.5 (old)│──> DS (upgraded)
                    └─────────────┘
                    ┌─────────────┐
         (standby)  │ AM 8.0 (new)│──> DS (same)
                    └─────────────┘

Step 1: New AM 8.0 starts, connects to same upgraded DS
Step 2: Test AM 8.0 internally (QA, smoke tests)
Step 3: Switch load balancer to AM 8.0
Step 4: If problems → switch LB back to AM 7.5 (instant rollback)
Step 5: Decommission AM 7.5
```

- Pros: Zero downtime, instant rollback, can test new version with real data
- Cons: Need double infrastructure temporarily, both AMs share DS (must be schema-compatible)

**Interview answer:** "We always use blue-green for production upgrades. The instant rollback capability is worth the extra infrastructure cost. For staging and dev, in-place is fine."

---

## Q9: What is the post-upgrade verification checklist?

**A:** Verify every category of config, starting with highest risk:

### Authentication Trees
| Test | How | Pass Criteria |
|---|---|---|
| Each tree by name | Authenticate via REST API | Get tokenId |
| Trees visible in console | Open Authentication → Trees | All trees listed |
| Retry/lockout logic | Fail N times | Lockout triggers |
| MFA tree | Authenticate + TOTP code | QR + verify works |

### OAuth2/OIDC
| Test | How | Pass Criteria |
|---|---|---|
| Client Credentials flow | `POST /access_token` | Get access token |
| Token introspection | `POST /introspect` | `active: true`, correct scopes |
| Authorization Code flow | Browser flow end-to-end | Get code, exchange for tokens |
| id_token | Decode JWT | Valid claims, correct signing alg |

### SAML2 Federation
| Test | How | Pass Criteria |
|---|---|---|
| IdP metadata endpoint | `GET /saml2/jsp/exportmetadata.jsp` | Valid XML |
| SP-initiated SSO | Hit SP SSO init URL | "SSO succeeded" |
| IdP-initiated SSO | Hit IdP SSO init URL | "SSO succeeded" |
| NameID mapping | Check assertion content | Correct attribute mapped |

### Policies
| Test | How | Pass Criteria |
|---|---|---|
| Policy evaluation | `POST /policies?_action=evaluate` | Correct allow/deny results |
| Policy sets visible | Console → Authorization | All sets listed |

### Custom Nodes
| Test | How | Pass Criteria |
|---|---|---|
| Nodes in palette | Open tree designer | All custom nodes listed |
| Node functionality | Authenticate through trees using each node | Correct routing/enrichment |

---

## Q10: What are the rollback scenarios and how do you handle each?

**A:**

### Scenario A: AM upgrade broke something, DS is fine
```bash
# Redeploy old AM WAR
docker stop pingam
# Replace AM-8.0.2.war with AM-7.5.war
docker start pingam
# Old AM reads old config from DS — should work
```

### Scenario B: DS schema upgrade broke something
```bash
# DS schema upgrade is irreversible — must restore from backup
docker stop pingds
# Restore from LDIF backup
import-ldif --backendName amConfig --ldifFile am-config.ldif
docker start pingds
# Redeploy old AM WAR
```

### Scenario C: Custom nodes crash AM
```bash
# Remove the custom node JAR entirely — AM starts without custom nodes
docker exec pingam rm /opt/tomcat/webapps/am/WEB-INF/lib/techcorp-custom-nodes-2.0.jar
docker restart pingam
# Trees using custom nodes fail, but AM itself is up
# Other trees still work
# Fix node code, recompile, redeploy
```

### Scenario D: Blue-green deployment
```bash
# Switch load balancer back to old AM instance
# Zero downtime, zero risk — this is why blue-green is the production standard
```

---

## Q11: What is the diligence priority order during upgrades?

**A:** If you can only be thorough about a few things, prioritize:

1. **Custom nodes** — highest breakage risk, requires code changes, no automated migration
2. **SAML federation** — involves external parties, certificate changes break trust, hard to debug remotely
3. **DS schema upgrade** — irreversible, must have LDIF backup before proceeding
4. **OAuth2 clients** — token format/signing changes can break all API consumers silently
5. **Authentication trees** — usually survive cleanly, but verify each one works
6. **Policies** — rarely break during upgrades, but verify evaluation results match

---

## Q12: How do you handle OAuth2 client upgrades specifically?

**A:** OAuth2 clients are medium risk because changes can be **silent** — the client still gets a token, but the token format or claims may have changed, breaking downstream API consumers.

**What can change:**
- Default token signing algorithm (e.g., HS256 → RS256 in newer versions)
- Token claim structure (new claims added, old claims renamed)
- Client secret storage format (hashing algorithm change)
- Scope handling (stricter validation in newer specs)

**Pre-upgrade:** Save each client's config:
```bash
curl -s ".../realm-config/agents/OAuth2Client/techcorp-app" > oauth2-client-backup.json
```

**Post-upgrade verification:**
1. Test Client Credentials flow — verify token returned
2. Introspect the token — verify claims, scopes, expiry match expectations
3. Test Authorization Code flow end-to-end — verify redirect, code exchange, id_token
4. Have downstream APIs validate tokens — ensure they accept the new format
5. Compare `statelessTokensEnabled`, `tokenSigningAlgorithm` settings against backup

---

## Q13: What's different about upgrading when you have IDM integration?

**A:** IDM adds complexity because it depends on AM for authentication:

**Upgrade order with IDM:**
1. DS (all backends: am-config, am-identity-store, am-cts, idm-repo)
2. AM (must be stable before IDM can authenticate)
3. IDM (connects to AM via OAuth2 — AM must be working first)
4. Custom nodes (deploy into running AM)

**IDM-specific risks:**
- `idm-provisioning` OAuth2 client config must survive — AM-IDM integration depends on it
- IDM Provisioning signing keys must survive
- If AM's OAuth2 provider changes token format, IDM's rsFilter may reject tokens
- IDM's own DS (`idm-repo` profile) needs separate schema upgrade

**Post-upgrade IDM verification:**
- IDM Admin UI accessible
- Recon runs successfully (LDAP connector to AM's DS still works)
- AM → IDM provisioning (if rsFilter is configured)

---

## Q14: How does the upgrade process differ for Kubernetes/cloud-native deployments?

**A:**

**Traditional (VM/Docker):**
- In-place WAR upgrade or blue-green with load balancer
- DS backup + restore for rollback
- Manual verification steps

**Kubernetes with FBC:**
- Config is in Git as JSON files
- Build new Docker image with new AM version + config files
- `kubectl apply` deploys new pods
- Kubernetes handles rolling update (zero downtime)
- Rollback: `kubectl rollout undo` → previous image with old AM version + old config
- DS runs as StatefulSet — schema upgrade via init container or job

**Advantages of K8s approach:**
- Immutable images = exact reproducibility
- Rolling updates = zero downtime by default
- Rollback is one command
- Git history = full audit trail of config + version changes

**Interview phrasing:** "In our Kubernetes environment, upgrades are just a new Docker image build with the updated AM WAR and FBC config files. We deploy via rolling update, and rollback is `kubectl rollout undo`. DS schema upgrade runs as a Kubernetes Job before the AM deployment."

---

## Q15: Give the complete interview answer for "How do you handle AM upgrades?"

**A:**

> "When upgrading AM, we follow a structured process. First, we inventory everything with Amster export and review release notes for breaking changes — custom nodes and SAML federation get the most scrutiny because they have the highest breakage risk.
>
> We take three types of backups: Amster config export for human-readable JSON we can diff, DS LDIF export for portable restore, and DS binary backup for fast recovery.
>
> Custom nodes get recompiled against the new AM APIs before the upgrade window. For the AM 7 to 8 upgrade, the main change was javax.inject to jakarta.inject because of the Tomcat 10 migration.
>
> We upgrade DS first (schema must be compatible), then AM. We always test in staging first with our full verification suite — auth tree tests for every tree, OAuth2 flow tests for every client, SAML SSO tests with federation partners in staging, and policy evaluation tests.
>
> For production, we use blue-green deployment so we can instantly switch the load balancer back if something breaks. The most common issues we watch for are custom node API changes, SAML certificate changes breaking federation trust, and DS schema upgrades being irreversible.
>
> After the upgrade, we verify everything against our pre-upgrade Amster export — if the entity count doesn't match, something was lost in the migration."