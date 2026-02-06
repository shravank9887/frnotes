# IDM Architecture & Fundamentals - Interview Q&A (Part 2)

**Topic:** PingIDM / ForgeRock IDM Architecture & Fundamentals
**Level:** Lead Engineer (7-8 years experience)
**Created:** 2026-02-06 (Session 18)
**Continued from:** INT_QA_IDM_Arch_1.md (Q1-Q7)

---

## Index

| # | Question | Topic/Tags |
|---|----------|------------|
| 8 | What is the ICF framework and why does IDM use it? | ICF, Connectors |
| 9 | Explain the three synchronization models in IDM | Reconciliation, LiveSync, Implicit Sync |
| 10 | What's the relationship between mappings and the link table? | Mappings, Links, Sync |
| 11 | How does IDM handle optimistic concurrency control? | MVCC, _rev, Concurrency |
| 12 | What is a link qualifier and when would you use it? | Link Qualifiers, Multi-Account |
| 13 | Describe IDM's config file architecture and overlay pattern | Config, JSON Files, Overlay |
| 14 | How would you design an HA/clustered IDM deployment? | High Availability, Clustering, Production |
| 15 | Compare IDM's architecture to competitors (Okta, SailPoint, Azure AD) | Competitive Analysis, Architecture |

---

## Q8: What is the ICF framework and why does IDM use it?

**Answer:**

**ICF** = **Identity Connector Framework** (originally from Sun Microsystems, now maintained by ForgeRock/Ping Identity)

### What it is:

ICF is a **connector abstraction layer** that provides a standard Java API for integrating with identity systems (LDAP, databases, REST APIs, SaaS apps, mainframes, etc.).

**Architecture:**
```
┌────────────────────────────────────────┐
│           PingIDM                      │
│  ┌──────────────────────────────────┐  │
│  │  Reconciliation Engine           │  │
│  │  Mapping Engine                  │  │
│  └───────────────┬──────────────────┘  │
│                  │                      │
│                  ↓                      │
│  ┌──────────────────────────────────┐  │
│  │  ICF Connector Framework         │  │ ← Standard API
│  └───┬───────┬───────┬───────┬──────┘  │
│      │       │       │       │          │
└──────┼───────┼───────┼───────┼──────────┘
       │       │       │       │
       ↓       ↓       ↓       ↓
   ┌──────┐ ┌────┐ ┌──────┐ ┌──────────┐
   │ LDAP │ │ DB │ │ REST │ │ Custom   │ ← Connector Implementations
   │Conn. │ │Conn│ │Conn. │ │Connector │
   └──────┘ └────┘ └──────┘ └──────────┘
       ↓       ↓       ↓       ↓
   ┌──────┐ ┌────┐ ┌──────┐ ┌──────────┐
   │  AD  │ │ HR │ │Google│ │Mainframe │ ← Target Systems
   └──────┘ └────┘ └──────┘ └──────────┘
```

### Core ICF concepts:

**1. Connector Interface (SPI)**
- `Connector` — base interface
- `CreateOp` — create objects in target system
- `UpdateOp` — update existing objects
- `DeleteOp` — delete objects
- `SearchOp` — query/search objects
- `SyncOp` — detect changes (for LiveSync)
- `TestOp` — test connection

**2. Object Classes**
- `__ACCOUNT__` — user accounts (most common, maps to inetOrgPerson in LDAP)
- `__GROUP__` — groups
- `__ALL__` — query all object types
- Custom object classes (e.g., `department`, `location`)

**3. Connector Configuration**
- Each connector has a config schema (host, port, credentials, base DN, etc.)
- Stored in `conf/provisioner.openicf-<name>.json`

---

### Why IDM uses ICF:

**1. Vendor-neutral abstraction**
- Write sync logic once, works across all connectors
- Swap LDAP vendor (OpenLDAP → AD → PingDirectory) without changing mappings
- Future-proof: new connectors added without core IDM changes

**2. Standard connector ecosystem**
- ForgeRock/Ping provides 30+ connectors out of the box:
  - LDAP (AD, PingDirectory, OpenLDAP)
  - Database Table (JDBC for any SQL database)
  - REST (generic HTTP/REST API integration)
  - CSV (bulk import/export)
  - Scripted SQL, Scripted REST (custom logic)
  - SaaS: Google Workspace, Salesforce, ServiceNow, etc.
- Community/custom connectors follow same interface

**3. Remote connector server support**
- ICF connectors can run remotely (Java or .NET)
- .NET connector server for Windows-only systems (Azure AD, SharePoint)
- Useful for DMZ, firewall-separated environments

**4. Built-in capabilities**
- Connection pooling (reuse LDAP/DB connections)
- Result paging (handle large result sets)
- Attribute filtering (fetch only needed attributes)
- Error handling and retry logic

---

### ICF vs proprietary approaches:

| Approach | Example | Pros | Cons |
|----------|---------|------|------|
| **ICF (IDM)** | Standard connector API | Vendor-neutral, extensible, ecosystem | More abstraction overhead |
| **Native SDK** | Microsoft Graph API for Azure AD | Optimized for specific system | Vendor lock-in, custom code per system |
| **ETL Tools** | Informatica, Talend | Mature data integration | Not identity-aware, no link table |

---

### Production example (LDAP connector config):

**File:** `conf/provisioner.openicf-ldap.json`
```json
{
  "name": "ldap",
  "connectorRef": {
    "connectorName": "org.identityconnectors.ldap.LdapConnector",
    "bundleName": "org.forgerock.openicf.connectors.ldap-connector",
    "bundleVersion": "[1.4.0.0,2.0.0.0)"
  },
  "configurationProperties": {
    "host": "pingds",
    "port": 1636,
    "ssl": true,
    "principal": "cn=Directory Manager",
    "credentials": "Passw0rd123",
    "baseContexts": ["ou=people,ou=identities"],
    "accountObjectClasses": ["inetOrgPerson"],
    "accountUserNameAttributes": ["uid"]
  },
  "objectTypes": {
    "__ACCOUNT__": {
      "$schema": "http://json-schema.org/draft-03/schema",
      "id": "__ACCOUNT__",
      "type": "object",
      "properties": {
        "uid": { "type": "string", "required": true },
        "cn": { "type": "string" },
        "sn": { "type": "string" },
        "givenName": { "type": "string" },
        "mail": { "type": "string" }
      }
    }
  }
}
```

**Interview answer:**
> "ICF is the **abstraction layer** that makes IDM connector-agnostic. It's like JDBC for identity systems — one API, many implementations. This allows IDM to sync with 30+ systems out of the box and makes custom connector development straightforward. In production, ICF's connection pooling, paging, and remote connector server support are critical for performance and security."

---

## Q9: Explain the three synchronization models in IDM

**Answer:**

IDM supports three sync models, each optimized for different use cases:

### 1. Full Reconciliation (Scheduled Batch Sync)

**What it is:**
- Compares **all** objects in source system to **all** objects in target
- Runs on a schedule (e.g., nightly, hourly)
- Two phases: **source phase** and **target phase**

**How it works:**
```
Source Phase:
  For each object in source system:
    1. Query for existing link
    2. If link exists → UPDATE target (situation: FOUND)
    3. If no link → CREATE target (situation: ABSENT)
    4. Run mapping transformations
    5. Create/update link table entry

Target Phase (optional):
  For each object in target system:
    1. Query for existing link
    2. If no link → orphan detection (situation: UNQUALIFIED)
    3. Optional: delete orphans or unlink
```

**Use cases:**
- **Nightly HR feed sync** — batch update from HR system
- **Initial migration** — load 100K users from legacy system
- **Drift detection** — find orphaned accounts monthly
- **Compliance reporting** — full inventory of accounts

**Performance considerations:**
- Large datasets (100K+ users): enable paging, increase Java heap
- Use `_queryFilter` to limit scope (e.g., only active users)
- Run during off-hours (high CPU/memory/network usage)
- Enable checkpointing for resumability

---

### 2. LiveSync (Near Real-Time Event Sync)

**What it is:**
- Connector polls source system's **change log** for delta updates
- Runs continuously (every 15 seconds typical)
- Only syncs **changed** objects (high efficiency)

**How it works:**
```
Every 15 seconds:
  1. Connector queries change log since last sync token
     (e.g., LDAP Persistent Search, DB changelog table, AD DirSync)
  2. For each changed object:
     - CREATE, UPDATE, or DELETE event
  3. Mapping processes the change
  4. Update target system
  5. Save new sync token (checkpoint)
```

**Requirements:**
- Source system must support change tracking:
  - **LDAP**: Persistent Search, `modifyTimestamp` attribute
  - **Database**: Changelog table with timestamps
  - **REST API**: `/changes` endpoint with delta tokens
- Connector must implement `SyncOp` interface

**Use cases:**
- **Active Directory → cloud sync** — near real-time user provisioning
- **Immediate access needs** — new hire shows up in Salesforce within 1 minute
- **Security events** — account disabled in HR → immediately disabled in all systems

**LiveSync config example:**
```json
{
  "enabled": true,
  "source": "system/ldap/__ACCOUNT__",
  "target": "managed/user",
  "schedule": "0/15 * * * * ?",  // Every 15 seconds
  "correlationQuery": {
    "type": "text/javascript",
    "source": "source.uid == target.userName"
  }
}
```

---

### 3. Implicit Sync (On-Demand Sync)

**What it is:**
- Sync triggered **during API operations** (not scheduled)
- Happens when a managed object is created/updated via REST API
- Synchronous (blocks until target systems are updated)

**How it works:**
```
User calls: POST /openidm/managed/user
  1. IDM creates managed/user object
  2. IDM checks for mappings with "implicit sync" enabled
  3. For each mapping:
     - Run transformations
     - Create/update target system object (via connector)
     - Create link table entry
  4. Return response to caller (all systems updated)
```

**Configuration:**
```json
{
  "mappings": [
    {
      "name": "managedUser_systemLdap",
      "source": "managed/user",
      "target": "system/ldap/__ACCOUNT__",
      "enableSync": true  // ← Enables implicit sync
    }
  ]
}
```

**Use cases:**
- **Self-service registration** — user registers → immediately created in AD
- **Delegated admin** — help desk creates user → syncs to all systems instantly
- **Workflow-driven provisioning** — approval completes → user provisioned synchronously

**Trade-offs:**
- **Pros**: Immediate consistency, no delay
- **Cons**: API latency (must wait for all targets), failure handling complexity (rollback?)

---

### Choosing the right model:

| Scenario | Recommended Model | Reasoning |
|----------|------------------|-----------|
| **HR nightly feed** | Full reconciliation | Batch data, no real-time requirement |
| **Active Directory → cloud apps** | LiveSync | Near real-time, change log available |
| **Self-service user registration** | Implicit sync | Immediate provisioning required |
| **Initial migration (legacy → IDM)** | Full reconciliation | One-time bulk load |
| **Orphan detection** | Full reconciliation (target phase) | Periodic audit |
| **SaaS app provisioning (Google, Salesforce)** | LiveSync or implicit | Real-time onboarding |

---

### Production best practice: **Hybrid approach**

```
┌─────────────────────────────────────────────────────────┐
│  HR System (authoritative source)                       │
└────────────┬────────────────────────────────────────────┘
             │
             ↓ Full Reconciliation (nightly 2 AM)
┌─────────────────────────────────────────────────────────┐
│  IDM managed/user (canonical identity)                  │
└─────┬──────────────────────────────┬────────────────────┘
      │                              │
      ↓ LiveSync (15 sec)            ↓ Implicit Sync (on-create)
┌──────────────┐              ┌──────────────────┐
│  Active      │              │  Delegated admin │
│  Directory   │              │  API creates     │
└──────────────┘              └──────────────────┘
```

**Interview answer:**
> "IDM's three sync models address different requirements: **full reconciliation** for batch/scheduled sync, **LiveSync** for near real-time event-driven sync, and **implicit sync** for synchronous API-driven provisioning. In production, you typically use a **hybrid approach** — nightly reconciliation for baseline accuracy, LiveSync for operational changes, and implicit sync for self-service/workflow scenarios. The key is understanding source system capabilities (does it have a change log?) and business requirements (how fast must changes propagate?)."

---

## Q10: What's the relationship between mappings and the link table?

**Answer:**

**Mappings** define the **sync logic**, and the **link table** stores the **sync state**. They work together to enable bi-directional synchronization.

### Mapping = Sync Logic (Declarative)

A mapping defines:
- **Source** and **target** objects (e.g., `system/ldap/__ACCOUNT__` → `managed/user`)
- **Attribute transformations** (e.g., `source.uid` → `target.userName`)
- **Correlation logic** (how to match existing objects)
- **Situation policies** (what to do in each sync scenario)
- **Conditions** (when to sync or skip)

**Mapping config:** `conf/sync.json`
```json
{
  "mappings": [
    {
      "name": "systemLdap__ACCOUNT___managedUser",
      "source": "system/ldap/__ACCOUNT__",
      "target": "managed/user",
      "properties": [
        { "source": "uid", "target": "userName" },
        { "source": "givenName", "target": "givenName" },
        { "source": "sn", "target": "sn" },
        { "source": "mail", "target": "mail" }
      ],
      "correlationQuery": {
        "type": "text/javascript",
        "source": "var query = {'_queryFilter': 'userName eq \"' + source.uid + '\"'}; query;"
      },
      "policies": [
        { "situation": "ABSENT", "action": "CREATE" },
        { "situation": "FOUND", "action": "UPDATE" },
        { "situation": "UNQUALIFIED", "action": "DELETE" }
      ]
    }
  ]
}
```

---

### Link Table = Sync State (Persistent)

The link table stores:
- **Which** external object is linked to **which** managed object
- **What** mapping created the link (`linkType`)
- **What** qualifier applies (`linkQualifier`)

**Link table entry (from our lab):**
```
dn: uid=a92185b0-3768-423e-959a-288f02f2f07e,ou=links,dc=openidm,dc=forgerock,dc=com
fr-idm-link-type: systemLdap__ACCOUNT___managedUser        ← Mapping name
fr-idm-link-firstid: 8e1c350f-56fe-49f0-9bc1-9c1bc834201e  ← External LDAP user _id
fr-idm-link-secondid: 02011b65-100c-42f1-be66-b63691c75f85 ← Managed user _id (bob)
fr-idm-link-qualifier: default                             ← Link qualifier
```

---

### How they work together (reconciliation flow):

```
Step 1: Reconciliation starts
  ↓
Step 2: IDM queries source system (e.g., LDAP users)
  ↓
Step 3: For each source object (e.g., uid=alice):
  ↓
  a. Check link table: "Does a link exist for this source object?"
     Query: linkType = "systemLdap__ACCOUNT___managedUser"
            AND firstId = "<alice-ldap-id>"
  ↓
  b. If link FOUND:
     - Retrieve linked managed user (secondId)
     - Situation = "FOUND"
     - Action = "UPDATE" (per mapping policy)
     - Run attribute transformations
     - Update managed/user
     - Update link table (timestamp, etc.)
  ↓
  c. If link ABSENT:
     - Run correlation query (try to match by userName)
     - If match found:
         Situation = "FOUND" (late binding)
         Create link, then UPDATE
     - If no match:
         Situation = "ABSENT"
         Action = "CREATE"
         Create new managed/user
         Create new link table entry
  ↓
Step 4: Target phase (optional)
  For each managed/user:
    a. Check link table: "Does a link exist for this managed object?"
       Query: linkType = "systemLdap__ACCOUNT___managedUser"
              AND secondId = "<managed-user-id>"
    b. If no link FOUND:
       - Situation = "UNQUALIFIED" (orphaned managed object)
       - Action = "DELETE" or "IGNORE" (per mapping policy)
```

---

### Why the link table is essential:

**1. Correlation without query**
- Without link table: IDM must query target system every time (slow)
- With link table: Direct lookup by ID (fast, efficient)

**2. Rename handling**
- User's `uid` changes from `alice.smith` to `alice.jones`
- Link table still points to same managed user (by UUID)
- IDM knows this is an UPDATE, not a new user

**3. Bi-directional sync**
- Link works in both directions:
  - LDAP → managed (firstId → secondId)
  - managed → LDAP (secondId → firstId)
- Same link entry, two mappings

**4. Multi-account support**
- One managed user can have multiple links (different qualifiers)
- Example:
  ```
  Link 1: alice (managed) → alice.smith (AD) [qualifier: default]
  Link 2: alice (managed) → alice_admin (AD) [qualifier: privileged]
  ```

**5. Audit and troubleshooting**
- Link table shows historical relationships
- "Why does managed user X have no LDAP account?" → Check link table
- "Why is this account orphaned?" → No link entry exists

---

### Production troubleshooting patterns:

**Problem:** Recon creates duplicate users instead of updating

**Root cause:** Link table entry missing or corrupted

**Fix:**
```bash
# Find managed user ID
curl -u admin:password "http://idm/openidm/managed/user?_queryFilter=userName+eq+'alice'"
# Returns: "_id": "abc-123-xyz"

# Check for existing link
curl -u admin:password "http://idm/openidm/repo/link?_queryFilter=secondId+eq+'abc-123-xyz'"
# Returns: empty (no link!)

# Manually create link (advanced)
curl -X POST -u admin:password \
  -H "Content-Type: application/json" \
  "http://idm/openidm/repo/link?_action=create" \
  -d '{
    "linkType": "systemLdap__ACCOUNT___managedUser",
    "firstId": "ldap-user-id-from-connector",
    "secondId": "abc-123-xyz",
    "linkQualifier": "default"
  }'

# Re-run recon → now UPDATE instead of CREATE
```

---

**Interview answer:**
> "The **mapping defines the sync logic** (what to sync, how to transform, when to create vs update), while the **link table stores the sync state** (which objects are related). During reconciliation, IDM queries the link table first — if a link exists, it's an UPDATE; if not, it runs the correlation query or creates new. The link table is what makes IDM **stateful** and enables rename handling, bi-directional sync, and multi-account scenarios. Without it, IDM would be a stateless ETL tool."

---

## Q11: How does IDM handle optimistic concurrency control?

**Answer:**

IDM uses **MVCC (Multi-Version Concurrency Control)** with a `_rev` field to handle concurrent updates without locking.

### The `_rev` Field

Every managed object has a `_rev` (revision) field:
```json
{
  "_id": "b38e9187-9606-4dfb-a60f-65ddd5d274e1",
  "_rev": "46e66758-1bd9-474c-92d4-9a7e2f1390cc-1382",
  "userName": "alice",
  "givenName": "alice",
  "sn": "alice",
  "mail": "alice@techcorp.com"
}
```

**`_rev` format:** `<instance-id>-<timestamp>-<counter>`
- `46e66758-1bd9-474c-92d4-9a7e2f1390cc` — IDM instance UUID
- `1382` — monotonic counter (increments on each update)

---

### How MVCC Works (Optimistic Locking)

**Scenario:** Two API calls try to update the same user simultaneously

```
Time  |  Client A                  |  Client B
------|----------------------------|---------------------------
T0    | GET /managed/user/alice    | GET /managed/user/alice
      | Returns: _rev = "...1382"  | Returns: _rev = "...1382"
      |                            |
T1    | PATCH /managed/user/alice  |
      | If-Match: "...1382"        |
      | Body: { "mail": "a@new" }  |
      | → SUCCESS                  |
      | New _rev = "...1383"       |
      |                            |
T2    |                            | PATCH /managed/user/alice
      |                            | If-Match: "...1382"  ← OLD REV!
      |                            | Body: { "title": "Engineer" }
      |                            | → 412 PRECONDITION FAILED
      |                            |
T3    |                            | Retry:
      |                            | GET /managed/user/alice
      |                            | Returns: _rev = "...1383"
      |                            | PATCH with If-Match: "...1383"
      |                            | → SUCCESS
```

---

### REST API Usage

**Required header:**
```http
If-Match: <_rev-value>
```

**Example:**
```bash
# 1. Read current version
curl -u admin:password "http://idm/openidm/managed/user/alice"
# Response: "_rev": "46e66758-1bd9-474c-92d4-9a7e2f1390cc-1382"

# 2. Update with revision check
curl -X PATCH -u admin:password \
  -H "Content-Type: application/json" \
  -H "If-Match: 46e66758-1bd9-474c-92d4-9a7e2f1390cc-1382" \
  "http://idm/openidm/managed/user/alice" \
  -d '[
    {
      "operation": "replace",
      "field": "/mail",
      "value": "alice.new@techcorp.com"
    }
  ]'

# 3. If another process updated first:
# HTTP 412 Precondition Failed
# {
#   "code": 412,
#   "message": "Version mismatch. Current version: ...1383, expected: ...1382"
# }
```

---

### Why Optimistic Locking?

**Alternative: Pessimistic Locking (traditional databases)**
```
1. BEGIN TRANSACTION
2. SELECT ... FOR UPDATE  ← Locks row
3. Application processes
4. UPDATE ...
5. COMMIT  ← Releases lock
```

**Problems with pessimistic locking in IDM:**
- **REST API is stateless** — no transaction context across HTTP calls
- **High contention** — many clients updating same users (self-service, recon, workflows)
- **Deadlocks** — distributed systems, multiple IDM nodes
- **Performance** — locking blocks concurrent reads

**Optimistic locking advantages:**
- **No locking overhead** — reads are never blocked
- **Detect conflicts late** — only fail if actual concurrent update occurred
- **Scalable** — works in clustered IDM (no distributed locks needed)
- **Retry-friendly** — client can refresh and retry

---

### When `_rev` Fails: Lost Update Problem

**Scenario:** Client ignores `_rev` (doesn't send `If-Match`)

```
Time  |  Client A              |  Client B
------|------------------------|------------------------
T0    | GET /managed/user/alice|
      | mail = "a@old.com"     |
T1    |                        | GET /managed/user/alice
      |                        | mail = "a@old.com"
T2    | PATCH (no If-Match)    |
      | mail = "a@new.com"     |
      | → SUCCESS              |
T3    |                        | PATCH (no If-Match)
      |                        | mail = "a@client-b.com"
      |                        | → SUCCESS (overwrites A!)
```

**Best practice:** **Always** use `If-Match` header in production

---

### Clustered IDM Behavior

**`_rev` includes instance UUID:**
- Each IDM node has unique instance ID
- Repository (DS or JDBC) stores the `_rev` globally
- Any node can update, `_rev` changes globally
- Next read from any node sees new `_rev`

**Example (2-node cluster):**
```
Node 1 updates user:  _rev = "node1-uuid-1234"
Node 2 reads user:    Sees _rev = "node1-uuid-1234"
Node 2 updates user:  _rev = "node2-uuid-1235"
Node 1 reads user:    Sees _rev = "node2-uuid-1235"
```

**Repository consistency:**
- **DS repo**: Multi-master replication (eventual consistency, sub-second lag)
- **JDBC repo**: DB-level consistency (depends on DB replication topology)

---

### Production scenarios:

**1. Self-service profile update + recon conflict**
```
User updates phone number (API):    _rev = "1382"
Recon runs 1 second later:           Reads _rev = "1382", updates title
Recon PATCH with If-Match: "1382":   → 412 PRECONDITION FAILED
Recon retries:                       Reads _rev = "1383", merges changes, succeeds
```

**2. Workflow approval + manual admin update**
```
Workflow reads user:           _rev = "1400"
Admin changes department:      _rev = "1401"
Workflow tries to add role:    If-Match: "1400" → 412 FAILED
Workflow must re-read and merge manually updated data
```

**Interview answer:**
> "IDM uses **optimistic concurrency control via the `_rev` field** (MVCC). Every update increments `_rev`, and clients must send `If-Match` header to detect concurrent modifications. This avoids pessimistic locking (which doesn't work in stateless REST APIs) and allows high concurrency. In clustered IDM, the repository (DS or JDBC) provides global `_rev` consistency. The trade-off is clients must handle 412 errors and retry with latest `_rev`. In production, always use `If-Match` for updates — skipping it risks silent lost updates."

---

*Continued in next response (Q12-Q15)*
