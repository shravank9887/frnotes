# IDM Architecture & Fundamentals - Interview Q&A (Part 1)

**Topic:** PingIDM / ForgeRock IDM Architecture & Fundamentals
**Level:** Lead Engineer (7-8 years experience)
**Created:** 2026-02-06 (Session 18)

---

## Index

| # | Question | Topic/Tags |
|---|----------|------------|
| 1 | What are managed objects in IDM and why are they the core concept? | Architecture, Managed Objects |
| 2 | Explain the difference between managed objects and external resources | Data Model, Concepts |
| 3 | What is the link table and why is it critical? | Link Table, Synchronization |
| 4 | Describe IDM's repository architecture. DS vs JDBC - what are the trade-offs? | Repository, DS, JDBC, Architecture |
| 5 | Why does IDM need a separate DS from AM's DS? | Two-DS Architecture, Design |
| 6 | Walk me through the physical structure of IDM's DS repository | DS Schema, Repository |
| 7 | How does IDM store managed object data in DS vs how AM stores config? | Data Storage, JSON |
| 8 | What is the ICF framework and why does IDM use it? | ICF, Connectors |
| 9 | Explain the three synchronization models in IDM | Reconciliation, LiveSync, Implicit Sync |
| 10 | What's the relationship between mappings and the link table? | Mappings, Links, Sync |
| 11 | How does IDM handle optimistic concurrency control? | MVCC, _rev, Concurrency |
| 12 | What is a link qualifier and when would you use it? | Link Qualifiers, Multi-Account |
| 13 | Describe IDM's config file architecture and overlay pattern | Config, JSON Files, Overlay |
| 14 | How would you design an HA/clustered IDM deployment? | High Availability, Clustering, Production |
| 15 | Compare IDM's architecture to competitors (Okta, SailPoint, Azure AD) | Competitive Analysis, Architecture |

---

## Q1: What are managed objects in IDM and why are they the core concept?

**Answer:**

**Managed objects** are IDM's **canonical representation of identity data** — the "golden record" stored in IDM's repository (DS or JDBC). They represent the authoritative identity model for your organization.

**Key characteristics:**
- **Schema-driven**: Defined in `conf/managed.json`, fully customizable
- **Versioned**: Every change creates a new `_rev` (MVCC for optimistic locking)
- **Queryable**: REST API exposes full CRUD operations
- **Synchronized**: External systems sync to/from managed objects via mappings

**Common managed object types:**
- `managed/user` — canonical user identity
- `managed/role` — business/IT roles
- `managed/assignment` — entitlements and access
- `managed/organization` — org hierarchy

**Why they're central:**
> "IDM is not just a sync tool — it's an **identity hub**. All external systems (LDAP, databases, SaaS apps) are synchronized to the canonical managed objects. This inverts the traditional model where AD is authoritative. In IDM, **managed objects are authoritative**, and AD is just another external system."

**Production example:**
```
HR system (source) → reconciliation → managed/user (canonical)
                                          ↓ mappings
                        ┌────────────────────────────────┐
                        ↓                                ↓
                    AD (target)                    Salesforce (target)
```

---

## Q2: Explain the difference between managed objects and external resources

**Answer:**

| Aspect | Managed Objects | External Resources |
|--------|-----------------|-------------------|
| **Location** | Stored in IDM's repository (DS/JDBC) | Stored in external systems (LDAP, DB, SaaS) |
| **Schema** | Fully customizable (conf/managed.json) | Fixed by external system (AD schema, DB columns) |
| **Authoritative** | IDM owns the data | External system owns the data |
| **Access** | REST API: `/openidm/managed/user/123` | Via ICF connector: `/openidm/system/ldap/account/uid=jdoe,...` |
| **Sync Direction** | Can be source or target | Can be source or target |
| **Naming** | `managed/<type>` (e.g., `managed/user`) | `system/<connector>/<objectType>` (e.g., `system/ldap/__ACCOUNT__`) |

**The relationship:**
- **Mappings** define how to sync between managed and external
- **Link table** tracks which external identity corresponds to which managed object
- **Reconciliation** synchronizes data bidirectionally

**Interview insight:**
> "The key architectural decision is: **what is authoritative?** In an IDM-centric model, managed objects are authoritative and HR/AD are external resources. In an AD-centric model, AD might be the source and IDM aggregates data. The link table allows IDM to support both patterns."

**Example link table entry (from our lab):**
```
linkType: systemLdap__ACCOUNT___managedUser
firstId:  8e1c350f-56fe-49f0-9bc1-9c1bc834201e (LDAP user uid=bob)
secondId: 02011b65-100c-42f1-be66-b63691c75f85 (managed/user bob)
qualifier: default
```

This says: "LDAP user bob (firstId) is linked to managed user bob (secondId) via the mapping `systemLdap__ACCOUNT___managedUser`."

---

## Q3: What is the link table and why is it critical?

**Answer:**

The **link table** is IDM's **relationship tracking database** that maps managed objects to external resources. It's stored in the repository at:
- **DS**: `ou=links,dc=openidm,dc=forgerock,dc=com`
- **JDBC**: `links` table

**Link table schema:**
```
uid: <UUID>                     (primary key)
linkType: <mappingName>         (e.g., systemLdap__ACCOUNT___managedUser)
firstId: <sourceObjectId>       (external system _id)
secondId: <targetObjectId>      (managed object _id)
linkQualifier: <qualifier>      (e.g., "default", "privileged", "contractor")
```

**Why it's critical:**

1. **Enables correlation**: IDM knows which external account belongs to which managed user
2. **Supports bi-directional sync**: Both systems can be updated, links track the relationship
3. **Allows multi-account scenarios**: One managed user can have multiple linked accounts (via different qualifiers)
4. **Prevents duplicate creates**: If a link exists, recon updates instead of creating
5. **Audit trail**: Links show historical relationships (when combined with audit logs)

**Production scenarios using link qualifiers:**

| Scenario | Qualifier | Use Case |
|----------|-----------|----------|
| Regular employee account | `default` | Standard AD account |
| Privileged admin account | `privileged` | Separate admin account for same user |
| Contractor vs employee | `contractor` | Different provisioning rules |
| Multi-region | `us-west`, `eu-central` | Separate accounts per region |

**Interview answer:**
> "The link table is what makes IDM a **stateful integration hub**. Without it, IDM would just be a batch sync tool. With it, IDM can track relationships over time, handle renames, detect orphans, and support complex multi-account scenarios. It's the foundation for all reconciliation logic."

**Link table query (production troubleshooting):**
```bash
# Find all links for a specific managed user
curl -u admin:password "http://idm/openidm/repo/link?_queryFilter=secondId+eq+'<managed-user-id>'"

# Find all links for a specific external account
curl -u admin:password "http://idm/openidm/repo/link?_queryFilter=firstId+eq+'<external-id>'"

# Find unqualified links (orphaned links)
curl -u admin:password "http://idm/openidm/repo/link?_queryFilter=linkQualifier+eq+'orphan'"
```

---

## Q4: Describe IDM's repository architecture. DS vs JDBC - what are the trade-offs?

**Answer:**

IDM supports two repository types for storing managed objects, links, config, and internal state:

### 1. DS Repository (what we're using)

**Architecture:**
- LDAP-based storage using PingDS (OpenDJ)
- Schema: `dc=openidm,dc=forgerock,dc=com`
- Data stored as JSON in `fr-idm-json` attribute
- Setup profile: `idm-repo:8.0`

**Pros:**
- **Native ForgeRock integration** — same vendor, same support
- **Replication built-in** — DS's multi-master replication for HA
- **Mature tooling** — LDAP utilities, backup/restore, monitoring
- **ForgeRock's recommended choice** for Platform deployments

**Cons:**
- **Less familiar to DBAs** — LDAP skill set required
- **JSON-in-LDAP overhead** — not as efficient as native JSON storage
- **Vendor lock-in** — tied to ForgeRock stack

---

### 2. JDBC Repository (PostgreSQL, MySQL, Oracle, MSSQL)

**Architecture:**
- Relational database storage
- Tables: `managedobjects`, `links`, `internaluser`, `auditaccess`, etc.
- JSON data stored in `CLOB`/`TEXT` columns (or native JSON in PostgreSQL)

**Pros:**
- **DBA familiarity** — standard RDBMS tooling and skills
- **Enterprise database features** — connection pooling, read replicas, backups
- **Database-agnostic** — easier to migrate between DB vendors
- **Better JSON support** (PostgreSQL native JSON columns, indexing)

**Cons:**
- **HA complexity** — need to manage DB replication separately (not built-in)
- **Performance tuning** — indexes, connection pools, query optimization
- **Schema upgrades** — manual DDL scripts during IDM version upgrades

---

### Trade-off Matrix (Lead Engineer Answer)

| Factor | DS Repository | JDBC Repository |
|--------|---------------|-----------------|
| **HA/Clustering** | Built-in multi-master replication | Requires external DB clustering (PostgreSQL streaming replication, MySQL Galera, Oracle RAC) |
| **Backup/Restore** | LDIF export (`backup-backend`), binary backup | SQL dump, DB-specific backup tools |
| **Performance** | Optimized for LDAP queries, JSON overhead | Native SQL queries, better JSON indexing (PostgreSQL) |
| **Skillset** | LDAP admins | DBAs |
| **ForgeRock Platform** | Recommended default | Supported but not preferred |
| **Cloud-native** | Works well in K8s (StatefulSet) | Easier with managed DB (RDS, Cloud SQL) |

---

### Production recommendation:

**Use DS if:**
- You're deploying ForgeRock Platform (AM + IDM + DS)
- You need out-of-the-box HA without external dependencies
- You have LDAP expertise in-house

**Use JDBC (PostgreSQL) if:**
- You're IDM-only deployment (no AM)
- Your org has strong PostgreSQL DBA team
- You want native JSON column types and better query performance
- You're using managed cloud databases (AWS RDS, GCP Cloud SQL)

**Interview insight:**
> "In production, I'd choose **DS for Platform deployments** (integrated stack, vendor support) and **PostgreSQL for standalone IDM** (DBA familiarity, better JSON, managed cloud DB options). The key is aligning with your team's skills and operational tooling."

---

## Q5: Why does IDM need a separate DS from AM's DS?

**Answer:**

**Short answer:** Different schemas, different data models, different replication requirements, and blast radius isolation.

### Architectural reasons:

**1. Different Schemas**
- **AM DS** uses AM-specific schema:
  - `ou=am-config` — AM configuration
  - `ou=identities` — user authentication identities (inetOrgPerson)
  - `ou=tokens` — CTS session/OAuth2 tokens
- **IDM DS** uses IDM-specific schema:
  - `ou=managed` — managed objects (JSON blobs in `fr-idm-json` attribute)
  - `ou=links` — link table (`fr-idm-link` objectClass)
  - `ou=internal`, `ou=config`, `ou=relationships`, etc.

Mixing these in one DS would create schema conflicts and operational complexity.

---

**2. Different Replication Requirements**

| Aspect | AM DS | IDM DS |
|--------|-------|--------|
| **Replication topology** | Multi-master, globally distributed (CTS replication across DCs) | Multi-master, co-located with IDM instances |
| **Replication lag tolerance** | Low (CTS tokens need near-instant replication) | Higher (managed objects can tolerate seconds of lag) |
| **Replication scope** | Often excludes ou=identities (authentication is local) | All managed objects must replicate |
| **Purge policies** | Aggressive CTS reaper (tokens expire quickly) | Long retention (identities are persistent) |

---

**3. Operational Blast Radius Isolation**

If AM and IDM shared one DS:
- **AM CTS load** (high write rate for session tokens) would impact IDM query performance
- **IDM reconciliation** (bulk reads during recon) would impact AM authentication performance
- **DS outage** would take down both AM and IDM simultaneously (no fault isolation)
- **Backup/restore** would require coordinating both systems (operational complexity)
- **Upgrade timing** — AM upgrades require DS schema changes that would force IDM downtime

---

**4. Security and Access Control**

- **AM DS** is read by AM servers (authentication), written by end users (via AM)
- **IDM DS** is read/written by IDM servers only (no direct user access)
- Separate DS instances allow different ACLs, firewall rules, and network zones

---

**5. The LDAP Connector Pattern**

IDM **reads** AM's DS as an **external resource** via the LDAP connector:
```
IDM → LDAP Connector → AM DS (ou=identities)
      (reconciliation)
```

If they shared one DS, this pattern wouldn't work — IDM can't treat part of its own repository as an external system.

---

### Production topology:

```
┌─────────────────┐         ┌─────────────────┐
│   PingAM (8081) │         │  PingIDM (8082) │
│                 │         │                 │
│  Authentication │         │  Provisioning   │
└────────┬────────┘         └────────┬────────┘
         │                           │
         ↓                           ↓
┌─────────────────┐         ┌─────────────────┐
│   AM DS (1389)  │         │  IDM DS (2389)  │
│                 │←──LDAP──│                 │
│ ou=identities   │ Connector│ ou=managed      │
│ ou=am-config    │         │ ou=links        │
│ ou=tokens       │         │                 │
└─────────────────┘         └─────────────────┘
```

**Interview answer:**
> "Separate DS instances for AM and IDM is a **best practice for production**. It provides schema isolation, performance isolation, independent scaling, and operational flexibility. The LDAP connector allows IDM to sync with AM's identity store while keeping the architectures decoupled. In Platform deployments, ForgeRock provides separate DS setup profiles (`am-config`, `am-cts`, `am-identity-store` for AM; `idm-repo` for IDM) to enforce this separation."

---

## Q6: Walk me through the physical structure of IDM's DS repository

**Answer:**

Based on our lab environment running PingIDM 8.0 + PingDS with `idm-repo:8.0` profile:

```
dc=openidm,dc=forgerock,dc=com (base DN)
│
├── ou=managed (managed objects - the canonical identity store)
│   ├── ou=user          (managed users)
│   ├── ou=role          (managed roles)
│   ├── ou=group         (managed groups)
│   ├── ou=assignment    (entitlement assignments)
│   ├── ou=organization  (org hierarchy)
│   └── ou=application   (application catalog)
│
├── ou=links (link table - tracks managed ↔ external relationships)
│   └── uid=<linkId>
│       ├── fr-idm-link-type: systemLdap__ACCOUNT___managedUser
│       ├── fr-idm-link-firstid: <external-account-id>
│       ├── fr-idm-link-secondid: <managed-user-id>
│       └── fr-idm-link-qualifier: default
│
├── ou=internal (IDM internal users - openidm-admin, idm-provisioning)
│
├── ou=config (IDM runtime config - less used in file-based config mode)
│
├── ou=relationships (managed object relationships)
│
├── ou=recon (reconciliation state)
│   └── Stores recon run status, progress, timestamps
│
├── ou=reconprogressstate (recon checkpoint state for resumability)
│
├── ou=clusteredrecontargetids (distributed recon coordination)
│
├── ou=scheduler (scheduled job state)
│
├── ou=cluster (cluster node membership and coordination)
│   └── Tracks IDM cluster members, heartbeats
│
├── ou=locks (distributed locking for clustered operations)
│
├── ou=sync (synchronization state)
│
├── ou=Security (keystore, truststore)
│
├── ou=ui (UI customization data)
│
├── ou=file (file-based storage for workflows, reports)
│
├── ou=generic (generic object storage)
│
├── ou=updates (update state tracking)
│
├── ou=import (bulk import staging)
│
└── ou=jsonstorage (generic JSON document storage)
```

---

### Key containers explained:

**1. ou=managed** (the heart of IDM)
- Stores all managed objects as JSON blobs
- Each entry has:
  - `uid` = managed object `_id` (UUID)
  - `fr-idm-json` = full object JSON (LDAP single-value attribute)
  - `objectClass: fr-idm-managed-user` (or role, group, etc.)

**2. ou=links** (the glue)
- Each link entry maps one external account to one managed object
- Attributes:
  - `fr-idm-link-type` — mapping name (e.g., `systemLdap__ACCOUNT___managedUser`)
  - `fr-idm-link-firstid` — external system object ID
  - `fr-idm-link-secondid` — managed object ID
  - `fr-idm-link-qualifier` — link qualifier (default, privileged, etc.)

**3. ou=recon + ou=reconprogressstate**
- Stores reconciliation run state (started, completed, failed)
- Checkpoint state allows resuming interrupted recon jobs
- In clustered mode, ensures only one node runs a given recon

**4. ou=cluster + ou=locks**
- Cluster membership (node IDs, heartbeats, timestamps)
- Distributed locks prevent concurrent recon or scheduler jobs

---

### Example managed user entry (DS structure):

```
dn: uid=b38e9187-9606-4dfb-a60f-65ddd5d274e1,ou=user,ou=managed,dc=openidm,dc=forgerock,dc=com
objectClass: fr-idm-managed-user
objectClass: uidObject
uid: b38e9187-9606-4dfb-a60f-65ddd5d274e1
fr-idm-json: {
  "_id": "b38e9187-9606-4dfb-a60f-65ddd5d274e1",
  "_rev": "46e66758-1bd9-474c-92d4-9a7e2f1390cc-1382",
  "userName": "alice",
  "givenName": "alice",
  "sn": "alice",
  "mail": "alice@techcorp.com",
  "accountStatus": "active",
  "effectiveRoles": [],
  "effectiveAssignments": []
}
```

**Interview insight:**
> "IDM uses LDAP as a **JSON document store**, not traditional LDAP attribute storage. This is why you query managed objects via REST API (`/openidm/managed/user`) instead of LDAP. The DS backend is an implementation detail — the JSON structure is what matters."

---

## Q7: How does IDM store managed object data in DS vs how AM stores config?

**Answer:**

**Key difference:** IDM treats DS as a **JSON document store**, while AM stores config as **traditional LDAP attributes**.

### IDM Managed Objects in DS

**Storage model:**
- All managed object data stored in a **single LDAP attribute**: `fr-idm-json`
- Attribute type: `SINGLE-VALUE` text (JSON string)
- Each managed object = one LDAP entry with JSON blob

**Example:**
```ldap
dn: uid=alice-uuid,ou=user,ou=managed,dc=openidm,dc=forgerock,dc=com
objectClass: fr-idm-managed-user
uid: b38e9187-9606-4dfb-a60f-65ddd5d274e1
fr-idm-json: {"_id":"b38e...", "userName":"alice", "givenName":"alice", ...}
```

**Why JSON-in-LDAP?**
- **Schema flexibility** — add/remove attributes without changing LDAP schema
- **Complex data structures** — arrays, nested objects (e.g., `effectiveRoles: [...]`)
- **MVCC versioning** — `_rev` field enables optimistic concurrency
- **REST API alignment** — JSON is the native format for IDM's REST API

---

### AM Config in DS

**Storage model:**
- Config stored as **traditional LDAP attributes**
- Each config property = separate LDAP attribute
- Multi-value attributes for lists

**Example (OAuth2 Provider config):**
```ldap
dn: ou=OAuth2Provider,ou=services,o=techcorp,ou=services,ou=am-config
objectClass: sunServiceComponent
sunKeyValue: coreOAuth2Config.statelessTokensEnabled=false
sunKeyValue: coreOAuth2Config.accessTokenLifetime=3600
sunKeyValue: advancedOAuth2Config.supportedScopes=openid
sunKeyValue: advancedOAuth2Config.supportedScopes=profile
```

**Why traditional LDAP?**
- **AM predates IDM** — used LDAP natively before JSON REST APIs were standard
- **Replication compatibility** — DS's LDAP replication works seamlessly
- **Backward compatibility** — AM 6.x, 7.x, 8.0 all use this model

---

### Trade-offs:

| Aspect | IDM (JSON-in-LDAP) | AM (LDAP Attributes) |
|--------|-------------------|---------------------|
| **Schema changes** | No LDAP schema change needed | Must update LDAP schema |
| **Complex data** | Supports nested JSON | Multi-value attributes only |
| **Query performance** | Must deserialize JSON | Native LDAP indexing |
| **REST API** | JSON is native format | Must convert LDAP → JSON |
| **LDAP tools** | Limited usefulness (`ldapsearch` shows JSON blob) | Full LDAP tooling works |

---

### Production implications:

**For IDM:**
- Use **REST API** for all managed object queries (don't use `ldapsearch`)
- Backup via **IDM's export API** or DS binary backup (not LDIF)
- Indexing is done on JSON fields (via DS's JSON index support)

**For AM:**
- **Amster** exports to JSON (converts LDAP attributes → JSON files)
- **DS replication** handles config distribution across AM instances
- **ldapsearch** is useful for troubleshooting AM config

**Interview answer:**
> "IDM and AM both use DS but store data differently. **IDM uses DS as a JSON document store** (schema-flexible, REST-native), while **AM uses traditional LDAP attributes** (backward-compatible, replication-optimized). This is why you use REST API for IDM and Amster/LDAP for AM. Understanding this distinction is critical for backup, troubleshooting, and migration strategies."

---

*Continued in INT_QA_IDM_Arch_2.md (Q8-Q15)*
