# ForgeRock/PingAM Interview Questions — AM Upgrades (Part 4 - FINAL)

*Final section: AM Upgrade details, rollback, testing, and common issues*

---

## Part 3 Continued: AM Upgrades (Detailed)

### Q25: Pre-upgrade checklist (backup DS, backup config, check deprecated features, check custom code compatibility)

**Answer**:

A comprehensive pre-upgrade checklist ensures safe upgrades and provides rollback options.

#### **1. Backup Strategy**

| What to Backup | How | Why |
|---|---|---|
| **DS data** | LDIF export | Contains all config and runtime data |
| **DS binary backup** | File system snapshot | Faster restore than LDIF import |
| **AM config** | Amster export-config | Rollback to previous config |
| **Keystores** | Copy `.jks`, `.p12`, `.keystore` files | Encryption/signing keys not in DS |
| **Custom code** | JAR files from `WEB-INF/lib/` | Custom nodes, plugins, post-auth hooks |
| **Tomcat config** | `server.xml`, `setenv.sh`, `context.xml` | JVM settings, datasource configs |
| **Advanced properties** | Screenshot AM Console → Server Defaults → Advanced | Lost during upgrade |

**Backup commands**:

```bash
# 1. LDIF export of DS (all backends)
docker exec pingds /opt/opendj/bin/export-ldif \
  --hostname localhost \
  --port 4444 \
  --bindDN "cn=Directory Manager" \
  --bindPassword "Passw0rd123" \
  --backendID am-config \
  --ldifFile /tmp/am-config-backup.ldif

docker exec pingds /opt/opendj/bin/export-ldif \
  --backendID am-cts \
  --ldifFile /tmp/am-cts-backup.ldif

docker exec pingds /opt/opendj/bin/export-ldif \
  --backendID userRoot \
  --ldifFile /tmp/userRoot-backup.ldif

# Copy LDIF files out of container
docker cp pingds:/tmp/am-config-backup.ldif ./backups/
docker cp pingds:/tmp/am-cts-backup.ldif ./backups/
docker cp pingds:/tmp/userRoot-backup.ldif ./backups/

# 2. File system backup of DS data
docker exec pingds /opt/opendj/bin/stop-ds
tar -czf ds-backup-$(date +%Y%m%d).tar.gz /path/to/opendj/data
docker exec pingds /opt/opendj/bin/start-ds

# 3. Amster export
amster <<EOF
:connect http://pingam:8081/am -k /path/to/admin/.keyId
:export-config --path /backups/am-config-$(date +%Y%m%d)
:exit
EOF

# 4. Backup keystores
cp -r /opt/tomcat/am-config/keystores /backups/keystores-$(date +%Y%m%d)

# 5. Backup custom JARs
cp /opt/tomcat/webapps/am/WEB-INF/lib/custom-* /backups/custom-jars/

# 6. Export advanced properties
curl -s -H "iPlanetDirectoryPro: <ADMIN_TOKEN>" \
  "http://pingam:8081/am/json/global-config/servers/am01?_fields=advanced" \
  > /backups/advanced-properties.json
```

---

#### **2. Review Deprecated Features**

Check the release notes for deprecated features that will be removed:

**AM 7.0 deprecations** (from AM 6.5):
- Embedded DS for production use
- Authentication Chains (replaced by Trees)
- File-based configuration (moved to DS)
- Some old REST API versions

**Action**: Identify if you use deprecated features and plan migration.

**Example check**:
```bash
# Check if any authentication chains are still in use
docker exec pingds /opt/opendj/bin/ldapsearch \
  --hostname localhost --port 1636 --useSsl --trustAll \
  --bindDN "cn=Directory Manager" --bindPassword "Passw0rd123" \
  --baseDN "ou=am-config" \
  "(objectClass=sunIdentityRepositoryService)" dn | grep -i chain
```

---

#### **3. Check Custom Code Compatibility**

Custom authentication nodes, post-auth plugins, and policy plugins may break if they use deprecated APIs.

**Checklist**:
- [ ] List all custom JARs in `WEB-INF/lib/`
- [ ] Identify API version they were compiled against
- [ ] Check ForgeRock Public API Javadoc for breaking changes
- [ ] Recompile custom code against target AM version
- [ ] Test in staging environment

**API compatibility check**:
```bash
# Extract custom JARs
cd /opt/tomcat/webapps/am/WEB-INF/lib/
ls -1 custom-*.jar

# Check manifest for implementation version
unzip -p custom-risk-node-1.0.0.jar META-INF/MANIFEST.MF

# Expected output:
# Implementation-Version: 1.0.0
# Built-Against-AM-Version: 7.3.0

# If built against AM 7.3, likely compatible with AM 7.4, 7.5
# If built against AM 6.5, must recompile for AM 7.x
```

**Known breaking changes** (AM 6.x → AM 7.x):
- `TreeContext` method signatures changed
- Some utility classes moved packages
- `@Supported` APIs unchanged, but `@Evolving` APIs may break

**Action**: Recompile and retest all custom nodes in staging with target AM version before production upgrade.

---

#### **4. Document Current State**

| Item | How to Document |
|---|---|
| **Realms** | List all realms, their purposes |
| **Trees** | Screenshot tree diagrams, export JSON |
| **Policies** | Count policies per realm |
| **OAuth2 clients** | List client IDs and their apps |
| **Custom nodes** | List JARs and versions |
| **Integrations** | Document external systems (LDAP, SAML SPs, OAuth2 clients) |

**Example documentation**:
```markdown
## Current AM Configuration (Pre-Upgrade)

**Version**: AM 7.3.2
**DS Version**: DS 7.3.2
**JVM**: OpenJDK 11.0.16

**Realms**:
- `/` (root) — Global config
- `/techcorp` — Employee authentication
- `/partner` — Partner federation

**Authentication Trees** (techcorp):
- TechCorpLogin (Username/Password)
- TechCorpMFA (TOTP)
- TechCorpRiskBased (Custom risk scoring node)

**Custom Nodes**:
- `risk-scoring-node-1.2.0.jar` (compiled against AM 7.3 API)
- `database-lookup-node-1.0.0.jar` (compiled against AM 7.2 API — needs recompile)

**SAML2 Entities**:
- techcorp-idp (hosted IdP in /techcorp)
- partner-sp (remote SP, Salesforce)

**OAuth2 Clients** (techcorp): 47 clients
- techcorp-app (main web app)
- mobile-app (iOS/Android app)
- ...

**Known Issues**:
- Database lookup node occasionally times out (5s limit)
- High CTS token volume (10M+ tokens, need cleanup)
```

---

#### **5. Test in Staging First**

**Critical rule**: Never upgrade production without testing in staging.

**Staging test plan**:
1. Clone production config to staging (Amster export/import)
2. Clone production DS data (LDIF export/import or replication)
3. Perform upgrade in staging
4. Run regression tests (authentication, federation, authorization)
5. Check logs for errors or warnings
6. Verify custom nodes work
7. Performance test (load, latency)
8. Document issues and fixes

---

**Interview answer**: "My pre-upgrade checklist starts with backups — LDIF exports of all DS backends, Amster export of AM config, and file system backups of keystores and custom JARs. I review the release notes for deprecated features and check if we use any. I verify that all custom authentication nodes compile against the target AM version and recompile if needed. I document the current state — realms, trees, policies, custom nodes, integrations. Then I test the entire upgrade process in staging with production-like data and traffic. Only after staging is verified do I schedule the production upgrade. This process has saved us multiple times when custom nodes broke on AM 7.4 due to API changes."

**Sources**:
- [Plan the upgrade (AM 7.2.2)](https://backstage.forgerock.com/docs/am/7.2/upgrade-guide/upgrade-planning.html)
- [Plan the upgrade (AM 7.4.0)](https://backstage.forgerock.com/docs/am/7.4/upgrade-guide/upgrade-planning.html)
- [Plan the upgrade (PingAM 8.0.1)](https://backstage.forgerock.com/docs/am/8/upgrade-guide/upgrade-planning.html)

---

### Q26: The upgrade process (in-place vs side-by-side)

**Answer**:

ForgeRock supports two upgrade strategies: **in-place** and **side-by-side**.

#### **In-Place Upgrade**

Upgrade the existing AM installation by replacing the WAR file and running the upgrade tool.

**Pros**:
- Simpler — reuses existing DS and config
- Faster (no data migration)
- Lower infrastructure cost (no parallel environment)

**Cons**:
- Downtime required
- Harder to rollback (must restore backups)
- Risky for production (no parallel testing)

**Process**:

```bash
# 1. Stop AM
systemctl stop tomcat

# 2. Backup current AM WAR
cp /opt/tomcat/webapps/am.war /backups/am-7.3.2.war.backup

# 3. Deploy new AM WAR
cp /downloads/AM-7.5.0.war /opt/tomcat/webapps/am.war

# 4. Run upgrade tool (automatically runs on first startup)
systemctl start tomcat

# Monitor catalina.out for upgrade progress
tail -f /opt/tomcat/logs/catalina.out

# 5. Verify upgrade
curl http://localhost:8080/am/json/serverinfo/version
```

**Upgrade tool**:
- AM detects version mismatch between WAR and DS config
- Automatically runs `amupgrade` embedded tool
- Migrates DS schema and config
- Updates version number in DS

---

#### **Side-by-Side Upgrade (Blue/Green)**

Deploy new AM version alongside the old version, migrate traffic after testing.

**Pros**:
- Zero downtime (cutover is instant)
- Easy rollback (switch traffic back to blue)
- Thorough testing in production environment
- Safer for critical systems

**Cons**:
- More complex (requires load balancer, parallel infrastructure)
- Higher cost (2x AM instances during migration)
- Must handle DS config migration carefully

**Architecture**:

```
                Load Balancer
                      |
         ┌────────────┴────────────┐
         |                         |
   Blue (AM 7.3)            Green (AM 7.5)
         |                         |
         └─────────┬───────────────┘
                   |
            DS (upgraded to 7.5)
```

**Process**:

```bash
# 1. Deploy new AM 7.5 instance (green) in parallel
# - New Tomcat server
# - Same DS backend (shared)
# - Load balancer routes 100% traffic to blue (AM 7.3)

# 2. Stop AM 7.5 (green)
systemctl stop tomcat-green

# 3. Upgrade DS (while blue is still serving traffic)
# DS upgrade is backward-compatible — AM 7.3 can read 7.5 DS schema

# 4. Start AM 7.5 (green)
systemctl start tomcat-green

# AM 7.5 detects DS is already upgraded, starts normally

# 5. Smoke test AM 7.5 (green) — internal tests only
curl -H "Host: green.internal" http://localhost:8080/am/json/serverinfo/version

# 6. Route 10% traffic to green via load balancer
# Monitor logs, errors, performance

# 7. Gradually increase traffic: 25% → 50% → 100%

# 8. Decommission blue (AM 7.3) after green is stable
systemctl stop tomcat-blue
```

**Rollback**:
```bash
# If green has issues, route 100% traffic back to blue
# Blue (AM 7.3) can still read DS 7.5 schema (backward compatible)
```

---

#### **Which to use?**

| Environment | Strategy | Reason |
|---|---|---|
| **Dev/Staging** | In-place | Simpler, downtime acceptable |
| **Production (low-traffic)** | In-place | Planned maintenance window |
| **Production (high-availability)** | Side-by-side | Zero downtime required |
| **Kubernetes** | Blue/green via deployments | Native K8s rolling updates |

---

**Interview answer**: "For dev and staging, I use in-place upgrades — stop AM, replace the WAR, start AM, and the upgrade tool runs automatically. It's simpler and downtime doesn't matter. For production, we use side-by-side (blue/green) deployments. We deploy the new AM version in parallel, upgrade the shared DS, smoke test the new AM instance, then gradually shift traffic via the load balancer — 10%, 25%, 50%, 100%. If anything breaks, we route traffic back to the old version instantly. After a week of stable green traffic, we decommission blue. This approach gives us zero downtime and safe rollback."

**Sources**:
- [Upgrade AM Instances (AM 7.0.2)](https://backstage.forgerock.com/docs/am/7/upgrade-guide/upgrade-servers.html)
- [Upgrade the platform from version 7.2 to 7.3 (ForgeOps)](https://backstage.forgerock.com/docs/forgeops/7.3/how-to/72to73.html)

---

### Q27: Configuration migration (AM 6 file-based .properties → AM 7+ DS-based config)

**Answer**:

**AM 6.x and earlier**: Configuration stored in **flat files** (`.properties`, XML) under `~/openam/` directory.

**AM 7.x+**: Configuration stored in **Directory Server** under `ou=am-config`.

This was a major architectural change requiring migration during upgrade.

---

#### **AM 6.x configuration storage**:

```
~/openam/
├── config/
│   ├── bootstrap.properties
│   ├── AMConfig.properties
│   └── ...
├── opensso-upgrade/
└── ...
```

**Key files**:
- `AMConfig.properties` — Server-specific config (port, deployment URI)
- `bootstrap.properties` — DS connection settings

---

#### **AM 7.x+ configuration storage**:

**Everything in DS** under `ou=services,ou=am-config`:

```
ou=am-config
├── ou=services                        ← Global services
│   ├── ou=platform
│   ├── ou=default-config               ← Default server config
│   └── o=techcorp,ou=services         ← Realm services
│       ├── ou=authenticationTreesService
│       ├── ou=iPlanetAMAuthService
│       ├── ou=sunEntitlementService   ← Policies
│       └── ...
├── ou=tokens                          ← CTS
└── ...
```

**Advantages of DS-based config**:
- Multi-master replication (high availability)
- Consistent config across AM cluster
- Transactional updates (ACID properties)
- Centralized backup (LDIF export includes config + data)

**Disadvantages**:
- Harder to version control (not plain text files)
- Requires DS infrastructure
- Performance: network latency for config reads

---

#### **Migration during AM 6 → AM 7 upgrade**:

The upgrade tool automatically:
1. Reads old `.properties` files
2. Converts to LDAP entries
3. Writes to DS under `ou=am-config`
4. Leaves old files in place (for rollback)

**Manual migration**:
```bash
# AM 6.5 → AM 7.0 upgrade

# 1. Export AM 6.5 config via Amster (optional backup)
amster-6.5 <<EOF
:connect http://am65:8080/openam amadmin password
:export-config --path /backups/am65-config
:exit
EOF

# 2. Stop AM 6.5
systemctl stop tomcat

# 3. Upgrade DS to 7.0 (install new DS with am-config profile)
# DO NOT reuse AM 6.x DS — schema incompatible

# 4. Deploy AM 7.0 WAR
cp AM-7.0.0.war /opt/tomcat/webapps/am.war

# 5. Start AM 7.0
systemctl start tomcat

# 6. AM 7.0 first startup:
#    - Detects ~/openam/ directory
#    - Reads bootstrap.properties for DS connection
#    - Runs upgrade tool
#    - Migrates file-based config to DS
#    - Stores "config version" in DS

# 7. Verify in DS
docker exec pingds /opt/opendj/bin/ldapsearch \
  --baseDN "ou=am-config" \
  --searchScope sub \
  "(objectClass=*)" dn | head -20
```

---

#### **AM 7.4+ File-Based Configuration (FBC)**

AM 7.4 introduced **file-based configuration** as a **technology preview**, allowing config to be stored in JSON files again (but different from AM 6.x):

**Why FBC**:
- Easier version control (JSON files in Git)
- Faster CI/CD (no DS config import at runtime)
- Immutable deployments (bake config into Docker images)
- Simpler local development

**Migration AM 7.x DS-based → FBC**:
```bash
# Use amupgrade tool in migration mode
/opt/am/amupgrade \
  --migrationMode \
  --sourceDir /opt/ds/config \
  --targetDir /opt/am/config-fbc

# Output: JSON files + LDIF file for data that couldn't migrate
```

**Limitations**:
- Not all config types supported in FBC (technology preview in 7.4, GA in 7.5)
- Dynamic data (users, sessions) still in DS
- Running data (audit logs, CTS tokens) still in DS

---

**Interview answer**: "AM 6.x stored configuration in flat files under `~/openam/` — `.properties` files and XML. Starting with AM 7.0, all config moved to Directory Server under `ou=am-config`. This enables multi-master replication for high availability and consistent config across clusters. During the AM 6 → AM 7 upgrade, the upgrade tool automatically migrates the file-based config to DS. You cannot reuse the old DS — you must install a new DS 7.0 with the `am-config` profile because the schema is incompatible. AM 7.4+ introduced file-based configuration (FBC) as a new option — JSON files instead of DS — but it's for a different purpose: version control and immutable deployments. The AM 6.x `.properties` files are not compatible with AM 7.4 FBC; they serve different architectures."

**Sources**:
- [Migrate to file-based configuration (AM 7.4.1)](https://backstage.forgerock.com/docs/am/7.4/upgrade-guide/migrate-to-fbc.html)
- [Migrate to file-based configuration (AM 7.5.0)](https://backstage.forgerock.com/docs/am/7.5/upgrade-guide/migrate-to-fbc.html)
- [Best practice for upgrading to AM 7.x](https://backstage.forgerock.com/knowledge/kb/article/a92587780)

---

### Q28: Custom auth node compatibility across versions (API changes between AM 6/7/8)

**Answer**:

Custom authentication nodes compiled for one AM version may break when deployed to a different version due to API changes.

#### **API Stability Guarantees**:

ForgeRock uses annotations to indicate API stability:

| Annotation | Stability | Can Break? |
|---|---|---|
| `@Supported` | Stable — won't break across minor versions | No (backward compatible) |
| `@Evolving` | May change in minor versions | Yes (recompile needed) |
| `@Deprecated` | Will be removed in future major version | Yes (migrate away) |

**Best practice**: Only use `@Supported` APIs in custom nodes.

---

#### **Breaking changes by version**:

**AM 6.x → AM 7.0** (major breaking changes):

| Change | Impact |
|---|---|
| `TreeContext` signature changed | Must update `process()` method signature |
| `NodeState` API refactored | `putShared()` / `putTransient()` methods changed |
| Some utility classes moved packages | Update `import` statements |
| `@Node.Metadata` annotation fields added | Add new fields (`tags`) |

**Example breaking change**:

**AM 6.5 code**:
```java
@Override
public Action process(TreeContext context) throws NodeProcessException {
    // AM 6.5 API
    context.sharedState.put("key", "value");
    return goTo(true).build();
}
```

**AM 7.0+ code**:
```java
@Override
public Action process(TreeContext context) throws NodeProcessException {
    // AM 7.0+ API
    return goTo(true)
        .replaceSharedState(context.sharedState.put("key", "value"))
        .build();
}
```

**Fix**: Recompile node against AM 7.0 SDK:
```xml
<dependency>
    <groupId>org.forgerock.am</groupId>
    <artifactId>auth-node-api</artifactId>
    <version>7.0.0</version>
    <scope>provided</scope>
</dependency>
```

---

**AM 7.0 → AM 7.5** (minor changes):

Most custom nodes compiled for AM 7.0 work on AM 7.5 without changes if they only use `@Supported` APIs.

**Known issues**:
- Nodes with `version = 0.0.0` in metadata cause tree loading errors (fixed by setting proper version)
- Some `@Evolving` callback types changed between 7.2 and 7.3

---

**AM 7.x → PingAM 8.0** (moderate changes):

| Change | Impact |
|---|---|
| Package renames (`com.forgerock` → `com.pingidentity`) | Update imports (rare, mostly internal packages) |
| Java 17 requirement | Recompile with JDK 17 |
| Some deprecated APIs removed | Migrate to new APIs |

**Compatibility statement from ForgeRock**:
> "Nodes from previous product versions should be compatible with subsequent product versions if you have only used @Supported APIs. For example, a node built against AM 7.5 APIs can be deployed in an AM 8 instance."

---

#### **Testing node compatibility**:

**Step 1: Recompile against target AM version**
```bash
# Update pom.xml
<properties>
    <am.version>7.5.0</am.version>
</properties>

<dependency>
    <groupId>org.forgerock.am</groupId>
    <artifactId>auth-node-api</artifactId>
    <version>${am.version}</version>
    <scope>provided</scope>
</dependency>

# Rebuild
mvn clean compile

# If compile succeeds → API-compatible
# If compile fails → breaking changes, must fix code
```

**Step 2: Deploy to staging with new AM version**
```bash
cp target/custom-node-1.0.0.jar /opt/am75-staging/webapps/am/WEB-INF/lib/
systemctl restart tomcat-staging
```

**Step 3: Test in staging**
- Add node to test tree
- Execute tree via REST API
- Check logs for exceptions
- Verify outcomes work as expected

---

#### **Upgrade strategy for custom nodes**:

**Option 1: Recompile before production upgrade**
```
1. Clone custom node repo
2. Update AM API dependency to target version
3. Recompile and test in staging
4. Deploy recompiled JAR during production upgrade
```

**Option 2: Test existing JAR in staging**
```
1. Deploy existing JAR to staging (target AM version)
2. If it works → no recompile needed
3. If it breaks → recompile and fix
```

**Recommendation**: Always recompile, even if not strictly required. Ensures you're using the latest API and avoids runtime surprises.

---

**Interview answer**: "Custom authentication nodes compiled for one AM version may not work on another version due to API changes. ForgeRock marks APIs as `@Supported` (stable) or `@Evolving` (may change). The biggest breaking change was AM 6.x to AM 7.0 — the `TreeContext` and `NodeState` APIs changed significantly, requiring code updates and recompilation. For AM 7.x to 8.0, most nodes work if they only use `@Supported` APIs, but I still recompile to be safe. My upgrade process: recompile all custom nodes against the target AM version, test in staging, and deploy the new JARs alongside the AM upgrade. This has prevented production issues multiple times."

**Sources**:
- [Upgrade nodes and change node configuration (AM 7.1)](https://backstage.forgerock.com/docs/am/7.1/auth-nodes/node-upgrade.html)
- [Upgrade nodes and change node configuration (AM 7.4.0)](https://backstage.forgerock.com/docs/am/7.4/auth-nodes/node-upgrade.html)
- [Prepare for development (PingAM 7.5.0)](https://backstage.forgerock.com/docs/am/7.5/auth-nodes/preparing-for-nodes.html)

---

### Q29: Rollback strategy

**Answer**:

A rollback strategy is critical for recovering from failed upgrades.

#### **Rollback methods**:

| Method | When to Use | Speed | Data Loss Risk |
|---|---|---|---|
| **Restore from backup** | In-place upgrade failed | Slow (restore DS from LDIF) | Lose changes since backup |
| **Switch traffic to blue** | Side-by-side (blue/green) | Instant (load balancer cutover) | None |
| **Redeploy previous Docker image** | Kubernetes/ForgeOps | Fast (K8s rolling update) | Depends on DS state |
| **Reinstall previous AM version** | Catastrophic failure | Very slow | Lose changes since backup |

---

#### **Rollback Scenario 1: In-Place Upgrade Failed**

**Problem**: AM 7.5 upgrade completed but authentication fails.

**Rollback**:
```bash
# 1. Stop broken AM 7.5
systemctl stop tomcat

# 2. Remove AM 7.5 WAR
rm -rf /opt/tomcat/webapps/am
rm -f /opt/tomcat/webapps/am.war

# 3. Restore DS from LDIF backup (made before upgrade)
docker exec pingds /opt/opendj/bin/import-ldif \
  --hostname localhost --port 4444 \
  --bindDN "cn=Directory Manager" --bindPassword "Passw0rd123" \
  --backendID am-config \
  --ldifFile /backups/am-config-pre-upgrade.ldif \
  --replaceExisting

# 4. Restore AM 7.3 WAR
cp /backups/am-7.3.2.war /opt/tomcat/webapps/am.war

# 5. Restore custom JARs
cp /backups/custom-jars/*.jar /opt/tomcat/webapps/am/WEB-INF/lib/

# 6. Start AM 7.3
systemctl start tomcat

# 7. Verify
curl http://localhost:8080/am/json/serverinfo/version
# Should show AM 7.3.2
```

**Time**: 30-60 minutes (depends on DS size)

**Data loss**: Changes made between backup and rollback are lost (users created, config changes, sessions)

---

#### **Rollback Scenario 2: Blue/Green Deployment**

**Problem**: AM 7.5 (green) has performance issues after cutover.

**Rollback**:
```bash
# 1. Route 100% traffic back to AM 7.3 (blue) via load balancer
# AWS ALB example:
aws elbv2 modify-rule \
  --rule-arn arn:aws:elasticloadbalancing:... \
  --actions Type=forward,TargetGroupArn=<BLUE_TARGET_GROUP>

# 2. Verify blue is serving traffic
curl http://load-balancer.example.com/am/json/serverinfo/version
# Should show AM 7.3.2

# 3. Investigate green issues offline
# Green instance still running, can debug without user impact

# 4. Fix green or decide to retry upgrade later
```

**Time**: < 1 minute (instant traffic switch)

**Data loss**: None (DS is shared, all changes preserved)

---

#### **Rollback Scenario 3: Kubernetes Deployment**

**Problem**: AM 8.0 deployment has errors.

**Rollback**:
```bash
# 1. Rollback to previous deployment
kubectl rollout undo deployment/am

# Or rollback to specific revision
kubectl rollout history deployment/am
kubectl rollout undo deployment/am --to-revision=5

# 2. Verify rollback
kubectl rollout status deployment/am
kubectl get pods -l app=am

# 3. Check version
kubectl exec am-0 -- curl -s localhost:8080/am/json/serverinfo/version
```

**Time**: 2-5 minutes (Kubernetes rolling update)

**Data loss**: Depends on DS state. If DS was upgraded, AM 7.5 pods can still read AM 8.0 DS schema (backward compatible).

---

#### **Rollback Limitations**:

**DS schema is not reversible**:
- Once DS is upgraded (e.g., 7.3 → 7.5), you cannot downgrade DS
- Old AM versions can read newer DS schemas (forward-compatible)
- But some new features won't work with old AM

**Example**:
```
Before upgrade: AM 7.3 + DS 7.3
After upgrade:  AM 7.5 + DS 7.5
Rollback AM:    AM 7.3 + DS 7.5  ← This works
Rollback DS:    DS 7.5 → 7.3      ← This does NOT work
```

**Workaround**: Restore DS from LDIF backup taken before upgrade.

---

#### **Rollback Decision Tree**:

```
Upgrade failed?
    │
    ├─ Side-by-side (blue/green)?
    │   └─ Yes → Switch traffic to blue (instant rollback)
    │
    ├─ Kubernetes?
    │   └─ Yes → kubectl rollout undo (fast rollback)
    │
    └─ In-place upgrade?
        └─ Yes → Restore DS from LDIF + redeploy old AM WAR (slow rollback)
```

---

**Interview answer**: "My rollback strategy depends on the deployment method. For side-by-side (blue/green) deployments, rollback is instant — just switch the load balancer back to the blue target group. For Kubernetes, I use `kubectl rollout undo` to revert to the previous deployment. For in-place upgrades, rollback is slower — I restore DS from the LDIF backup taken before the upgrade and redeploy the old AM WAR. The critical limitation is that DS schema upgrades are not reversible, so I always take LDIF backups before upgrading DS. In one incident, we rolled back from AM 7.5 to AM 7.3 after discovering a custom node incompatibility. Because we used blue/green, the rollback took 30 seconds."

**Sources**:
- [Plan the upgrade (AM 7.0.2)](https://backstage.forgerock.com/docs/am/7/upgrade-guide/upgrade-planning.html)
- [Upgrade the platform from version 7.2 to 7.3 (ForgeOps)](https://backstage.forgerock.com/docs/forgeops/7.3/how-to/72to73.html)

---

### Q30: Common upgrade issues (deprecated features, changed REST API versions, UI changes)

**Answer**:

Common issues encountered during ForgeRock AM upgrades and how to resolve them.

---

#### **Issue 1: Deprecated Features Removed**

**Problem**: Authentication chains no longer supported in AM 7.x, but old deployment still uses them.

**Symptom**:
- Login pages show error: "Authentication module not found"
- AM logs: `Module chain 'ldapService' not found`

**Fix**:
- Migrate all authentication chains to authentication trees before upgrading
- Use AM Console → Authentication → Trees to recreate chain logic
- Or use Amster to export chains and manually convert to tree JSON

**Prevention**: Review deprecation notices in release notes before upgrading.

---

#### **Issue 2: REST API Version Changes**

**Problem**: Custom applications call old REST API versions that are removed.

**Example**:
```bash
# Old app code (AM 6.5)
curl http://am:8080/openam/json/authenticate

# AM 7.0 response:
# 404 Not Found — /openam context path changed to /am
```

**Fix**:
```bash
# Update app to use new API
curl http://am:8080/am/json/authenticate \
  -H "Accept-API-Version: resource=2.0, protocol=1.0"
```

**Changes**:
- AM 7.0: `/openam` → `/am` (default context path)
- AM 7.0: `Accept-API-Version` header now required
- Some older API versions (`resource=1.0`) deprecated

**Prevention**: Check API version compatibility matrix in upgrade guide.

---

#### **Issue 3: Embedded DS Deprecated**

**Problem**: AM 6.5 uses embedded DS (single-server mode), but AM 7.0 requires external DS.

**Symptom**:
- Upgrade wizard prompts for external DS connection
- Cannot complete upgrade without migrating to external DS

**Fix**:
```bash
# 1. Install external DS 7.0
/opt/ds/setup \
  --profile am-config \
  --set am-config/amConfigAdminPassword:password \
  --hostname ds.example.com \
  --ldapPort 1389 \
  --ldapsPort 1636 \
  --adminConnectorPort 4444 \
  --acceptLicense

# 2. Export embedded DS data (from AM 6.5)
/opt/openam/opends/bin/export-ldif \
  --backendID userRoot \
  --ldifFile /tmp/userRoot.ldif

# 3. Import to external DS
/opt/ds/bin/import-ldif \
  --backendID userRoot \
  --ldifFile /tmp/userRoot.ldif

# 4. Configure AM 7.0 to use external DS during upgrade
```

**Prevention**: Migrate to external DS before upgrading to AM 7.0.

---

#### **Issue 4: UI Changes (XUI vs Classic UI)**

**Problem**: Custom UI extensions (CSS, JavaScript) break after upgrade.

**Example**:
- Custom login page CSS no longer loads
- JavaScript errors in browser console

**Fix**:
- Review UI customization guide for new AM version
- Update CSS selectors (XUI DOM structure changed)
- Test UI in staging before production upgrade

**Best practice**: Minimize UI customizations. Use theming APIs instead of direct CSS hacks.

---

#### **Issue 5: Custom Nodes with version=0.0.0**

**Problem**: Trees with custom nodes that have `version = "0.0.0"` in `@Node.Metadata` fail to load.

**Symptom**:
- Tree doesn't appear in AM Console tree list
- Logs: `NullPointerException` when loading tree

**Fix**:
```java
// Before (broken in AM 7.3+)
@Node.Metadata(
    outcomeProvider = MyNode.OutcomeProvider.class,
    configClass = MyNode.Config.class,
    version = "0.0.0"  // ← BROKEN
)

// After (working)
@Node.Metadata(
    outcomeProvider = MyNode.OutcomeProvider.class,
    configClass = MyNode.Config.class,
    version = "1.0.0"  // ← Use semantic versioning
)
```

**Prevention**: Always set proper semantic versions on custom nodes.

---

#### **Issue 6: Advanced Properties Lost**

**Problem**: Custom advanced server properties configured in AM Console are lost after upgrade.

**Location**: Configure → Server Defaults → Advanced

**Examples**:
- `com.sun.identity.session.maxSessions=10`
- `org.forgerock.openam.session.stateless.signing.type=RSA`

**Symptom**:
- Custom session limits no longer enforced
- Performance tuning parameters reset to defaults

**Fix**:
- Document all advanced properties before upgrade (screenshot or export)
- Re-add them manually after upgrade via AM Console or REST API

**Prevention**: Export advanced properties via REST before upgrading:
```bash
curl -H "iPlanetDirectoryPro: <ADMIN_TOKEN>" \
  "http://am:8080/am/json/global-config/servers/am01?_fields=advanced" \
  > advanced-properties.json
```

---

#### **Issue 7: Java Version Incompatibility**

**Problem**: PingAM 8.0 requires Java 17, but production runs Java 11.

**Symptom**:
- AM fails to start: `UnsupportedClassVersionError`

**Fix**:
```bash
# Upgrade JVM to Java 17
sudo apt install openjdk-17-jdk
sudo update-alternatives --config java

# Verify
java -version
# openjdk version "17.0.1"

# Restart AM
systemctl restart tomcat
```

**Prevention**: Check JVM requirements in release notes before upgrading.

---

**Interview answer**: "Common upgrade issues include deprecated features like authentication chains being removed in AM 7.x, REST API version changes that break custom applications, and advanced server properties being lost after upgrade. One tricky issue is custom nodes with `version=0.0.0` causing trees to fail loading in AM 7.3+. To prevent issues, I review the release notes thoroughly, test the upgrade in staging with production-like data, document all custom configurations, and check that custom applications use supported REST API versions. In one upgrade, we discovered our mobile app was calling the old `/openam/json/authenticate` endpoint, which broke in AM 7.0 when the context path changed to `/am`. Because we tested in staging first, we caught and fixed it before production."

**Sources**:
- [Best practice for upgrading to AM 7.x](https://backstage.forgerock.com/knowledge/kb/article/a92587780)
- [Plan the upgrade (AM 7.4.0)](https://backstage.forgerock.com/docs/am/7.4/upgrade-guide/upgrade-planning.html)

---

## Summary

This comprehensive guide covered:

**Custom Authentication Nodes**:
- Node API (AbstractDecisionNode, TreeContext, Action)
- Maven archetype and project setup
- Plugin lifecycle and OSGi deployment
- Config interface and @Attribute annotations
- OutcomeProvider for multi-outcome nodes
- Testing strategies (unit and integration tests)
- Real-world examples (risk scoring, MFA, external APIs, database lookups)
- ForgeRock Marketplace nodes

**Amster (AM CLI Automation)**:
- Core commands (connect, export-config, import-config)
- Groovy scripting for automation
- CI/CD pipeline integration
- Environment promotion (dev → staging → prod)
- Amster vs ForgeOps/CDK
- Configuration expressions and variables

**AM Upgrades**:
- Upgrade paths (AM 6.x → 7.x → 7.5 → PingAM 8.0)
- Pre-upgrade checklist (backups, deprecated features, custom node compatibility)
- In-place vs side-by-side (blue/green) upgrades
- Configuration migration (file-based → DS-based → file-based)
- Custom node API compatibility across versions
- Rollback strategies
- Common upgrade issues and fixes

---

**Interview Preparation Tips**:

1. **Know the "why"**: Explain not just how to do something, but why it's done that way
2. **Real-world examples**: Reference specific projects or incidents where you applied the knowledge
3. **Trade-offs**: Discuss pros/cons of different approaches (e.g., in-place vs blue/green upgrades)
4. **Troubleshooting**: Show how you debug issues (check logs, test in staging, use curl)
5. **Best practices**: Demonstrate knowledge of production-ready patterns (version control, CI/CD, testing)

Good luck with your interview!

---

*Generated: 2026-01-31*
*Sources: ForgeRock Backstage Documentation, Ping Identity Documentation, ForgeRock Community*
