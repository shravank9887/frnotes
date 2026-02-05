# Interview Q&A: Session Management & CTS (Core Token Service)

## What is CTS

### Q1: What is CTS and what does it do?

**CTS** (Core Token Service) is AM's persistence layer for all short-lived, transient data. It's the backend store for:

- **Sessions** — active user sessions (stateful mode)
- **OAuth2 tokens** — authorization codes, access tokens, refresh tokens (when server-side)
- **SAML2 artifacts** — assertion references
- **Session denylist** — revoked stateless session entries
- **STS (Security Token Service) tokens**
- **UMA (User-Managed Access) tokens**

CTS is backed by **PingDS (Directory Server)** — it's not a separate product, it's AM using DS as a high-performance token store. In our environment, the CTS data lives in the same DS instance that holds am-config and am-identity-store, under the base DN:

```
ou=famrecords,ou=openam-session,ou=tokens,ou=am-config
```

**Interview-ready answer**: "CTS is AM's token persistence layer backed by DS. Every active session, OAuth2 token, and SAML artifact is stored as a CTS entry in DS. It's what makes AM stateless at the application tier — any AM instance in the cluster can serve any request because the session state lives in the shared CTS store, not in AM's memory."

### Q2: How does CTS architecture work in production?

In production, CTS typically has its own **dedicated DS topology**, separate from AM config and identity stores:

```
                    ┌─────────────┐
                    │  Load       │
                    │  Balancer   │
                    └──────┬──────┘
                    ┌──────┴──────┐
                    │             │
                ┌───┴───┐   ┌───┴───┐
                │  AM1  │   │  AM2  │    AM Cluster (stateless app tier)
                └───┬───┘   └───┬───┘
                    │           │
            ┌───────┴───────────┴───────┐
            │                           │
      ┌─────┴─────┐             ┌──────┴──────┐
      │  CTS DS1  │◄───repl────►│  CTS DS2    │   CTS DS (dedicated)
      └───────────┘             └─────────────┘
```

**Why separate CTS DS?**
- CTS has high write volume (session create/update/delete on every request)
- am-config and identity store have low write volume (config changes, user updates)
- Mixing them causes CTS writes to compete with identity reads
- Separate DS lets you size, tune, and scale CTS independently
- CTS DS needs fast SSDs, lots of RAM for DB cache, and replication tuned for low latency

**In our lab**, we use a single DS instance (`pingds`) for everything — am-config, am-identity-store, and am-cts are all in the same DS. This is fine for development but not for production.

### Q3: What does a CTS token entry look like in DS?

Each CTS token is an LDAP entry under `ou=famrecords,ou=openam-session,ou=tokens`. Key attributes:

| Attribute | Meaning | Example |
|-----------|---------|---------|
| `coreTokenId` | Unique token identifier | `shandle:AM_SESSION_...` |
| `coreTokenType` | Token category | `SESSION`, `OAUTH`, `SAML2` |
| `coreTokenExpirationDate` | When the token expires (UTC) | `20260131180000Z` |
| `coreTokenUserId` | User who owns this token | `uid=demo,ou=people,ou=identities` |
| `coreTokenString*` | Token payload fields | Session properties, token data |
| `coreTokenObject` | Binary/encrypted token body | Session state blob |

**Interview-ready answer**: "CTS entries are LDAP objects in DS. Each entry has a type (SESSION, OAUTH, SAML2), an expiration date, the owning user's DN, and the token payload. AM uses LDAP search/add/modify/delete operations to manage these entries. DS indexes on `coreTokenType`, `coreTokenExpirationDate`, and `coreTokenUserId` for fast lookups and cleanup."

### Q4: How does CTS handle token cleanup/expiry?

Two mechanisms:

1. **AM-side reaper**: AM runs a periodic background thread that queries CTS for expired entries (`coreTokenExpirationDate < now`) and deletes them. The reaper interval is configurable.

2. **DS-side TTL (recommended in newer versions)**: DS can be configured to automatically delete entries based on their `coreTokenExpirationDate` attribute using a backend cleanup task. This offloads the work from AM.

**Why cleanup matters**: Without cleanup, CTS grows indefinitely. In a high-traffic environment, millions of expired tokens accumulate. DS performance degrades as the database grows. In production, monitor CTS entry count and DS disk usage.

---

## Session Types: Stateful vs Stateless (Client-Side)

### Q5: What is the difference between stateful and stateless (client-side) sessions?

| Aspect | Stateful (Server-Side) | Stateless (Client-Side) |
|--------|----------------------|------------------------|
| **Where session lives** | CTS entry in DS | Encrypted in the cookie/token itself |
| **tokenId format** | Opaque handle (reference to CTS entry) | Large encrypted+signed blob containing all session data |
| **Token size** | ~50 chars (handle only) | ~2-4 KB (entire session encoded) |
| **Server lookup per request** | Yes — AM reads CTS on every request | No — AM decrypts the token locally |
| **Scalability** | Limited by CTS DS capacity | Unlimited — no server state |
| **Logout/revocation** | Delete CTS entry — immediate | Need session denylisting (or wait for expiry) |
| **Session modification** | Update CTS entry | Re-issue new token with updated data |
| **Network overhead** | Low (small cookie) | Higher (large cookie sent on every request) |
| **Single point of failure** | CTS DS must be available | No dependency on CTS |
| **AM setting** | Realm → Authentication → Settings → Use Client-Side Sessions = OFF | Use Client-Side Sessions = ON |

**Your /techcorp realm**: Uses **stateful sessions** (Use Client-Side Sessions = OFF). Every session creates a CTS entry in DS.

### Q6: When would you choose stateful vs stateless sessions?

**Choose stateful when:**
- You need immediate session revocation (admin kills a session, user is locked out instantly)
- Session quotas are required (limit concurrent sessions per user)
- Session data is large or changes frequently
- You have a properly sized CTS DS cluster
- Compliance requires server-side session control (banking, healthcare)

**Choose stateless when:**
- You need maximum scalability (millions of concurrent sessions)
- CTS DS is a bottleneck or single point of failure
- You don't need real-time session revocation
- You're running in a cloud/serverless environment where shared state is expensive
- Network bandwidth is not a concern (large tokens on every request)

**Interview-ready answer**: "In our production environment, we used stateful sessions for the customer-facing portal because we needed session quotas (max 3 sessions per user) and immediate revocation (if fraud is detected, kill the session instantly). For our API gateway tier, we used stateless sessions because it was high-throughput and we didn't need real-time revocation — token expiry was sufficient."

### Q7: How does session denylisting work for stateless sessions?

Since stateless sessions live in the cookie (not in CTS), you can't just "delete" them. When a user logs out:

1. AM adds the session's `jti` (JWT Token ID) to a **denylist** in CTS
2. On every subsequent request, AM checks if the presented token's `jti` is in the denylist
3. If denylisted → reject. If not → accept.

From your Global Session config:
- `Enable Session Denylisting = OFF` — **this means logout doesn't actually invalidate stateless sessions in your environment**
- `Denylist Poll Interval = 10s` — how often AM checks CTS for new denylist entries
- `Denylist Cache Size = 10000` — in-memory cache to avoid hitting DS on every request

**The irony**: Denylisting re-introduces CTS dependency for stateless sessions. You still need DS to store the denylist. The benefit is that you only write to CTS on logout (not on every request), so the load is much lower than full stateful sessions.

---

## Session Lifecycle

### Q8: Walk through the complete lifecycle of a stateful session.

```
1. User authenticates
   └─→ AM creates session object in memory
   └─→ AM writes CTS entry to DS (coreTokenType=SESSION)
   └─→ AM returns tokenId (session handle) to browser as cookie

2. User makes subsequent requests
   └─→ Browser sends iPlanetDirectoryPro cookie (tokenId)
   └─→ AM looks up CTS entry by tokenId
   └─→ AM checks: expired? idle timeout? → if valid, serve request
   └─→ AM updates lastAccessTime in CTS (per Latest Access Time Update Frequency)

3. Session times out (idle or max)
   └─→ AM reaper finds expired CTS entries
   └─→ Deletes from DS
   └─→ Next request with old tokenId → AM returns 401

4. User explicitly logs out
   └─→ AM deletes CTS entry from DS immediately
   └─→ Cookie is cleared in browser
   └─→ Session is gone instantly
```

### Q9: What are the three session timeouts and how do they interact?

From your Dynamic Attributes tab:

| Timeout | Your Value | What it does |
|---------|-----------|--------------|
| **Maximum Session Time** | 120 min | **Absolute limit**. Session dies after 2 hours no matter what. Even if the user is actively using the app. Forces re-authentication. |
| **Maximum Idle Time** | 30 min | **Inactivity limit**. If no request for 30 minutes, session expires. Every request resets the idle timer. |
| **Maximum Caching Time** | 3 min | **Cache refresh interval**. AM caches session data locally for 3 minutes before re-reading from CTS. Not a timeout — it's a performance optimization. |

**How they interact**: The session dies when EITHER max session time OR idle time is reached, whichever comes first.

Example timeline:
```
T=0:00   User logs in → session created (max=120min, idle=30min)
T=0:15   User clicks page → idle timer resets to 30min
T=0:45   User clicks page → idle timer resets to 30min
T=1:15   User goes to lunch → no activity
T=1:45   Idle timeout hit (30min since last activity) → session EXPIRED
```

vs:
```
T=0:00   User logs in
T=0:29   User clicks → idle timer resets
T=0:58   User clicks → idle timer resets
T=1:27   User clicks → idle timer resets
T=1:56   User clicks → idle timer resets
T=2:00   Max session time hit → session EXPIRED (even though user is active)
```

**Why max session time matters**: It prevents indefinite sessions. Even if an attacker steals a session token and keeps it alive with activity, it still dies at 2 hours. Forces re-authentication as a security checkpoint.

### Q10: What is the "Latest Access Time Update Frequency" and why is it 60 seconds?

Your value: `60 seconds`

Every time a user makes a request, AM should update the session's `lastAccessTime` to reset the idle timer. But writing to DS on EVERY request is expensive — in a high-traffic app, that's thousands of DS writes per second just for timestamp updates.

**Solution**: AM only writes the timestamp every 60 seconds. Between updates, it tracks the time in memory.

**Tradeoff**:
- If AM crashes, up to 60 seconds of "activity" is lost. The session might appear idle even though the user was active.
- If idle timeout is 30 minutes and update frequency is 60 seconds, worst case a session lives 30min + 60sec before being recognized as expired.
- In production with very short idle timeouts (e.g., 5 minutes), you'd lower this to 10-15 seconds.

---

## Session Quotas

### Q11: How do session quotas work and what are the exhaustion strategies?

Session quotas limit how many **concurrent active sessions** a single user can have. From your config:

- **Enable Quota Constraints**: `OFF` (currently disabled)
- **Active User Sessions**: `5` (default quota — inactive since quotas are off)
- **Resulting behavior if quota exhausted**: `Destroy Next Expiring`

**Exhaustion strategies** (what happens when user has 5 sessions and tries to create a 6th):

| Strategy | Behavior | Use Case |
|----------|----------|----------|
| `DENY_ACCESS` | Block the new login. User must logout elsewhere first. | Strictest — banking, compliance. User gets "max sessions reached" error. |
| `DESTROY_OLD_SESSION` | Kill the **oldest** session, allow new login. | Common for consumer apps — "single device" feel. |
| `DESTROY_NEXT_EXPIRING` | Kill the session that will expire **soonest**. | Preserves the "freshest" sessions. Your current default. |
| `DESTROY_ALL_SESSIONS` | Kill ALL existing sessions, allow new login. | Nuclear option — "one session at a time" behavior. |

**Interview-ready answer**: "We set session quotas to 3 per user with `DESTROY_OLD_SESSION` behavior. If a user logs in on their laptop, phone, and tablet, then tries their work PC — the oldest session (laptop) gets killed. This prevents session hoarding and limits blast radius if credentials are stolen — the attacker can't maintain a session alongside the legitimate user indefinitely."

**Important**: Session quotas only work with **stateful sessions**. For stateless sessions, AM has no central count of active sessions (they live in cookies, not CTS). This is one of the key reasons to choose stateful over stateless.

---

## CTS in Production

### Q12: How did you size and monitor CTS in your company?

*Sample answer:*

"Our CTS architecture:

**Sizing**: We had ~50,000 concurrent sessions at peak. Each CTS session entry is roughly 2-4 KB. Plus OAuth2 tokens (~1 KB each), SAML artifacts, etc. We estimated ~200 MB of active CTS data, but DS needs 3-5x that for indexes, replication changelog, and cache. We provisioned 2 DS nodes with 8 GB RAM each, NVMe SSDs, and 4 cores.

**Topology**: Dedicated CTS DS replication group — two DS instances with multi-master replication. Separate from config DS and identity DS. AM is configured with CTS affinity — each AM instance preferentially talks to one CTS DS but fails over to the other.

**Monitoring**: We tracked:
- `ds_connections_count` — active LDAP connections from AM to CTS DS
- `ds_entries_count` for the CTS backend — number of active tokens
- Replication delay between CTS DS nodes (must be < 100ms)
- AM session creation rate / deletion rate
- CTS DS response time for LDAP operations (p99 < 5ms)

**Alerts**:
- CTS entry count growing faster than shrinking → reaper not keeping up → investigate
- Replication delay > 500ms → network issue or DS overloaded
- DS response time > 10ms → disk I/O bottleneck or need to increase DB cache"

### Q13: What happens when CTS DS goes down?

**With stateful sessions**:
- **Existing sessions**: AM cannot validate them → users get 401/session invalid
- **New logins**: AM cannot create sessions → authentication fails
- **Your config**: `Deny user login when session repository is down = NO` — AM will still attempt logins, but sessions won't persist. This is a degraded mode.

**With stateless sessions**:
- **Existing sessions**: Still work (data is in the cookie, no CTS lookup needed)
- **New logins**: Still work (session is created client-side)
- **Logout**: Denylisting fails (can't write to CTS) — revocation is delayed until CTS recovers
- **Session quotas**: Don't work (no central counter)

**This is the #1 argument for stateless sessions**: CTS DS outage doesn't cause total authentication failure. But you lose revocation and quotas.

**Interview-ready answer**: "We designed for CTS DS failure by having two replicated CTS DS nodes in different availability zones. AM had connection failover configured — if CTS DS1 was unreachable, it automatically switched to CTS DS2 within 5 seconds. We tested this quarterly in our DR exercises. For our API tier, we used stateless sessions as an additional safety net — even if both CTS nodes went down, API authentication continued working."

### Q14: How does CTS differ from the `am-cts` DS setup profile?

The `am-cts` profile in our DS setup (`setup-ds.sh`) creates:
- The CTS backend/suffix in DS
- Schema definitions for CTS object classes (`frCoreToken`, etc.)
- Indexes on `coreTokenId`, `coreTokenType`, `coreTokenExpirationDate`, `coreTokenUserId`, etc.
- Base entries under `ou=tokens,ou=am-config`

**CTS is a concept** (AM's token persistence service). **am-cts is a DS setup profile** (creates the DS structure to support CTS). Think of it as: am-cts creates the database tables, CTS is the application layer that reads/writes them.

---

## Client-Side Session Security

### Q15: How are stateless (client-side) sessions secured?

From your Global Session config, the Client-Side Sessions tab shows the full security chain:

```
Session Data → Sign (HS256) → Encrypt (AES) → Base64 encode → Cookie
```

1. **Signing** (HS256 — HMAC-SHA256): Prevents tampering. If anyone modifies the session data, the signature won't match. All AM instances share the same HMAC secret.

2. **Encryption** (Direct AES): Prevents reading. The session payload is encrypted so even though it's in the browser cookie, the user (or attacker) can't read session attributes like `authLevel`, `userId`, `realm`, etc.

3. **No compression** (your config): `NONE`. Could use DEF (deflate) to shrink the token, but adds CPU overhead.

**Attack scenarios and defenses**:

| Attack | Defense |
|--------|---------|
| Read session data from cookie | AES encryption — data is unreadable |
| Modify session (escalate privileges) | HMAC signature — tampered token is rejected |
| Replay expired session | Expiration timestamp is signed — can't change it |
| Steal session (XSS/network sniffing) | Use HttpOnly + Secure cookie flags + HTTPS |
| Forge new session | Need HMAC shared secret — stored in AM keystore |

**Critical secret**: The `Signing HMAC Shared Secret` and `Encryption Symmetric AES Key` are the crown jewels. If an attacker gets these, they can forge arbitrary sessions as any user with any auth level. These must be rotated periodically and stored securely (AM keystore, HSM in production).

### Q16: What is the session cookie and how does it travel?

The session cookie name is `iPlanetDirectoryPro` (configurable). It's set on the AM domain.

**Stateful**: Cookie contains just the session handle (~50 chars):
```
iPlanetDirectoryPro=AQIC5wM2LY4SfcxvdW...  (opaque handle)
```

**Stateless**: Cookie contains the entire encrypted session (~2-4 KB):
```
iPlanetDirectoryPro=eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJ1aWQ9ZG...  (JWT-like blob)
```

**Cross-domain problem**: The cookie is set on AM's domain (`pingam`). Applications on different domains (`app.company.com`) can't read it. Solutions:
- **Web Agent**: Sits in the app's web server, validates session via AM REST API
- **PingGateway (IG)**: Reverse proxy that validates session and forwards to app
- **Token exchange**: App exchanges AM session for its own token via OAuth2
- **Same domain**: Put AM and app on the same domain (e.g., `sso.company.com` and `app.company.com` with cookie domain `.company.com`)

---

## Practical Lab Observations

### Q17: What did you observe about sessions in your lab environment?

"In our Docker lab:

- **/techcorp realm uses stateful sessions** (`Use Client-Side Sessions = OFF`). Each login creates a CTS entry in DS under `ou=famrecords,ou=openam-session,ou=tokens,ou=am-config`.
- **Session timeouts**: Max 120 minutes absolute, 30 minutes idle. These are global defaults — the /techcorp realm inherits them.
- **Session quotas are OFF** globally. Default quota is 5 but unenforced.
- **Client-side session security**: Even though /techcorp doesn't use stateless sessions, the global config shows HS256 signing + AES encryption ready if a realm enables it.
- **Session denylisting is OFF** — if we switched to stateless, logout wouldn't actually invalidate the session until natural expiry.
- **Single DS instance**: Our `pingds` serves am-config, am-identity-store, AND am-cts. Fine for a lab, but in production these would be separate DS clusters for performance isolation."

### Q18: How would you verify active sessions in CTS via DS?

```bash
# Count all active CTS sessions
MSYS_NO_PATHCONV=1 docker.exe exec pingds /opt/opendj/bin/ldapsearch \
  --hostname localhost --port 1636 --useSsl --trustAll \
  --bindDN "cn=Directory Manager" --bindPassword "Passw0rd123" \
  --baseDN "ou=famrecords,ou=openam-session,ou=tokens,ou=am-config" \
  --searchScope sub "(coreTokenType=SESSION)" dn

# Count OAuth2 tokens
MSYS_NO_PATHCONV=1 docker.exe exec pingds /opt/opendj/bin/ldapsearch \
  --hostname localhost --port 1636 --useSsl --trustAll \
  --bindDN "cn=Directory Manager" --bindPassword "Passw0rd123" \
  --baseDN "ou=famrecords,ou=openam-session,ou=tokens,ou=am-config" \
  --searchScope sub "(coreTokenType=OAUTH)" dn
```

---

## CTS Reaper Configuration

### Q19: What is the CTS reaper and where do you configure it?

The **CTS reaper** is AM's background cleanup process that periodically scans the CTS store in DS and deletes expired tokens (sessions, OAuth2 grants, SAML artifacts, UMA tokens, etc.). Without it, expired entries accumulate and DS performance degrades.

**Where to configure it**:

1. **AM Console → Configure → Global Services → Session → CTS tab**
   - This is the primary location for CTS-related session cleanup settings

2. **AM Console → Configure → Server Defaults → Advanced** (property-level tuning):
   - `org.forgerock.services.cts.reaper.timeout` — max time the reaper task runs per cycle (prevents runaway cleanup blocking AM)
   - `com.iplanet.am.session.purgedelay` — delay (in minutes) after session expiry before the reaper actually deletes the CTS entry. Default: `0`. Setting a delay lets AM handle edge cases where a session is still in transit when it technically expires.

3. **Realm → Authentication → Settings → Session tab**:
   - Timeout values here determine **when** tokens become eligible for reaping (max session time, max idle time)

**How the reaper works**:
```
Every N seconds:
  1. Query DS: SELECT all CTS entries WHERE coreTokenExpirationDate < NOW() - purgeDelay
  2. Delete matching entries in batches
  3. Log cleanup stats
```

**DS-side alternative (DS 7+/8.0)**: DS itself can run a **backend cleanup task** that deletes entries based on `coreTokenExpirationDate`. This offloads the work from AM to DS, which is more efficient because:
- DS doesn't need LDAP round-trips (it reads its own database directly)
- No network overhead between AM and DS
- DS can batch deletes more aggressively

In production, the recommended approach is to use **both**: AM reaper as the primary mechanism, DS-side cleanup as a safety net.

**Interview-ready answer**: "The CTS reaper runs as a scheduled task inside AM that queries DS for expired CTS entries and deletes them. It's configured under Global Services → Session → CTS tab, with fine-tuning via Advanced properties like `purgedelay` and `reaper.timeout`. In DS 7+, you can also enable DS-side backend cleanup as a complementary mechanism — DS deletes expired entries directly without LDAP round-trips, which reduces load on AM. We monitored CTS entry count to ensure the reaper was keeping up with token creation rate."

### Q20: What happens if the CTS reaper falls behind?

If tokens are created faster than the reaper can delete them:

1. **CTS entry count grows** — DS database size increases
2. **DS performance degrades** — larger database means slower searches, more disk I/O
3. **AM response times increase** — session lookups take longer because DS is under pressure
4. **Disk space** — DS data directory grows; if disk fills up, DS crashes

**Symptoms**:
- Rising LDAP operation latency from AM to CTS DS
- Growing `ds_entries_count` metric for the CTS backend
- AM logs showing slow CTS operations
- Users experiencing delayed login or session validation

**Fixes**:
- Increase reaper frequency / reduce batch size
- Enable DS-side cleanup as a parallel mechanism
- Reduce session max time / idle time (fewer tokens to clean up)
- Scale CTS DS (more CPU, faster disks, dedicated instance)
- Check if session quotas are disabled — unlimited sessions per user compounds the problem

---

## DS Replication

### Q21: What is DS replication and why is it needed?

DS replication keeps multiple DS instances synchronized so that:
- **High availability**: If one DS goes down, others continue serving requests
- **Load distribution**: Read-heavy workloads can be spread across replicas
- **Geographic distribution**: DS instances in different data centers for low-latency access
- **CTS failover**: Without CTS DS replication, a single DS failure kills all AM sessions

DS replication is **multi-master** — all replicas accept both reads and writes. Changes propagate asynchronously via a **replication changelog**.

### Q22: How does DS replication work internally?

```
DS1 (writes change)
  │
  ├─→ Writes to local database
  ├─→ Records change in replication changelog with CSN (Change Sequence Number)
  └─→ Sends change to DS2 via replication protocol (port 8989)
        │
        DS2 (receives change)
          ├─→ Applies change to local database
          └─→ Acknowledges receipt back to DS1
```

**Key concepts**:
- **CSN (Change Sequence Number)**: Unique ID for each change — timestamp + serverId + sequence. Used for ordering and conflict detection.
- **Replication changelog**: Persistent log of changes on each DS. Retained for a configurable window (e.g., 24 hours). Enables DS to catch up after being offline.
- **Conflict resolution**: If two DS instances modify the same entry simultaneously, **last writer wins** (based on CSN timestamp). DS also has special handling for multi-valued attributes (merge instead of overwrite).
- **Replication purge delay**: How long changelog entries are retained. Must be longer than the maximum expected downtime for any replica.

### Q23: How do you set up DS replication in DS 8.0 (deployment ID model)?

DS 7.x+ introduced the **deployment ID** model, which simplifies replication setup. Instead of manually enabling replication between pairs of servers, all DS instances sharing the same deployment ID automatically form a replication topology.

**Step 1: Generate a deployment ID** (done once, shared across all DS instances):
```bash
/opt/opendj/bin/dskeymgr create-deployment-id \
  --deploymentIdPassword "Passw0rd123"
# Output: A long base64 deployment ID string
```

In our environment, `setup-ds.sh` does this and saves it to `${DATA_DIR}/.deployment_id`.

**Step 2: Set up the first DS instance** (normal setup with deployment ID):
```bash
/opt/opendj/bin/setup \
  --deploymentId <DEPLOYMENT_ID> \
  --deploymentIdPassword "Passw0rd123" \
  --serverId ds1 \
  --hostname pingds1 \
  --replicationPort 8989 \
  --profile am-config \
  --profile am-cts \
  --profile am-identity-store \
  ...
```

**Step 3: Set up the second DS instance** (bootstrap from the first):
```bash
/opt/opendj/bin/setup \
  --deploymentId <SAME_DEPLOYMENT_ID> \
  --deploymentIdPassword "Passw0rd123" \
  --serverId ds2 \
  --hostname pingds2 \
  --replicationPort 8989 \
  --bootstrapReplicationServer pingds1:8989 \
  --profile am-config \
  --profile am-cts \
  --profile am-identity-store \
  ...
```

The `--bootstrapReplicationServer` flag tells the new DS where to find an existing topology member. DS handles replication initialization automatically.

**Step 4: Verify replication status**:
```bash
/opt/opendj/bin/dsrepl status \
  --hostname pingds1 --port 4444 \
  --bindDN "uid=admin" --bindPassword "Passw0rd123" \
  --trustAll --no-prompt
```

### Q24: What is the difference between the old `dsreplication` command and the DS 8.0 deployment ID approach?

| Aspect | Legacy `dsreplication` (DS 6.x and earlier) | Deployment ID (DS 7+/8.0) |
|--------|---------------------------------------------|---------------------------|
| **Setup** | Manual: `dsreplication enable` between each pair | Automatic: same deployment ID = same topology |
| **Adding a server** | Run `dsreplication enable` + `initialize` for each existing server | Just `setup` with `--bootstrapReplicationServer` |
| **Key management** | Manual key exchange between servers | Shared deployment ID generates consistent keys |
| **Topology discovery** | Static — must configure each pair | Dynamic — DS discovers peers via bootstrap server |
| **Command** | `dsreplication enable/initialize/status` | `dsrepl status` (note: shortened command name in DS 7+) |
| **Typical use** | Traditional on-premise deployments | Cloud-native, Docker, Kubernetes deployments |

**Interview-ready answer**: "DS 8.0 uses the deployment ID model for replication. All DS instances sharing the same deployment ID automatically form a replication topology — you don't need to manually enable replication between pairs like in older versions. When adding a new DS instance, you run `setup` with `--bootstrapReplicationServer` pointing to any existing member. DS handles key exchange, changelog synchronization, and data initialization automatically. This is much simpler for container-based deployments where instances are ephemeral."

### Q25: How does AM connect to replicated DS instances?

AM supports multiple DS connection strategies:

**1. Connection string with multiple servers** (AM config):
```
host1:1636,host2:1636
```
AM tries the first server; if it fails, falls over to the next.

**2. Affinity-based load balancing**:
- AM preferentially uses one DS (primary) for CTS operations
- Falls over to secondary only on failure
- Reduces cross-datacenter replication conflicts

**3. In AM Console** — Configure → Global Services → External CTS Store (if using external CTS):
- **CTS Token Store**: Choose "External Token Store" (instead of embedded/same DS)
- **Connection String**: `host1:1636,host2:1636`
- **Max Connections**: Pool size per server
- **Heartbeat**: Interval to check DS availability

**4. For our lab** — AM connects to a single `pingds:1636` (no replication). In production, you'd configure AM to connect to the CTS DS cluster:
```
cts-ds1.prod:1636,cts-ds2.prod:1636
```

### Q26: What are the common DS replication problems and how do you troubleshoot them?

| Problem | Symptom | Fix |
|---------|---------|-----|
| **Replication delay** | Users see stale data, session created on DS1 not found on DS2 | Check network latency, DS load, changelog size. Monitor `ds_replication_*` metrics. |
| **Split brain** | Both DS instances accept conflicting writes | DS handles via conflict resolution (last writer wins). Check for `conflict` entries in DS. Rare with proper networking. |
| **Changelog exhausted** | DS was offline longer than purge delay. Can't catch up. | Must re-initialize: `dsrepl initialize` to copy all data from a healthy replica. |
| **Replication port blocked** | Firewall blocking port 8989 between DS instances | Open replication port in firewall/security group rules. |
| **Certificate mismatch** | DS instances can't authenticate to each other | Ensure same deployment ID. Re-export and import certificates if needed. |

**Monitoring commands**:
```bash
# Check replication status and delay
/opt/opendj/bin/dsrepl status --hostname ds1 --port 4444 --trustAll

# Check replication backlog (pending changes)
/opt/opendj/bin/dsrepl status --showReplicas --hostname ds1 --port 4444 --trustAll
```

**Interview-ready answer**: "The most common replication issue we encountered was changelog exhaustion — a DS replica went offline during maintenance longer than the replication purge delay (24 hours), and when it came back, it couldn't replay the missed changes. We had to re-initialize it from a healthy replica using `dsrepl initialize`. After that, we increased the purge delay to 48 hours and set up monitoring on replication backlog size to catch delayed replicas early."

### Q27: How would you design a production DS replication topology for AM?

**Recommended production topology** (3-tier):

```
┌─────────────────────────────────────────────────┐
│  Data Center 1              Data Center 2       │
│                                                 │
│  ┌──────────┐              ┌──────────┐         │
│  │ Config   │◄────repl────►│ Config   │         │
│  │ DS-1     │              │ DS-2     │         │
│  └──────────┘              └──────────┘         │
│  am-config store (low write, high read)         │
│                                                 │
│  ┌──────────┐              ┌──────────┐         │
│  │ Identity │◄────repl────►│ Identity │         │
│  │ DS-1     │              │ DS-2     │         │
│  └──────────┘              └──────────┘         │
│  am-identity-store (medium write/read)          │
│                                                 │
│  ┌──────────┐              ┌──────────┐         │
│  │ CTS      │◄────repl────►│ CTS      │         │
│  │ DS-1     │              │ DS-2     │         │
│  └──────────┘              └──────────┘         │
│  am-cts store (HIGH write, high read)           │
└─────────────────────────────────────────────────┘
```

**Design principles**:
- **Separate DS clusters per profile**: Config, Identity, and CTS each get their own DS pair. Different write patterns need different tuning.
- **CTS DS is the most critical**: Needs fastest disks (NVMe), most RAM (for DB cache), and lowest replication latency.
- **Cross-datacenter replication**: Each cluster spans two data centers. If DC1 goes down, DC2 continues.
- **AM affinity**: Each AM instance prefers the local-DC DS. Failover to remote-DC only on local failure.
- **Minimum 2 replicas per cluster**: For HA. 3 replicas for large deployments (2 active + 1 standby).

**Interview-ready answer**: "In production, I'd deploy three separate DS replication groups — one for am-config, one for am-identity-store, and one for am-cts. Each group has at least two replicated DS instances across data centers. CTS DS gets the most resources because it handles the highest write volume — every session create, update, and delete hits CTS. AM is configured with affinity-based routing to prefer local DS instances with automatic failover. This gives us zero-downtime capability for both planned maintenance and unplanned failures."

---

## Session Configuration Inheritance

### Q28: Global Services vs Realm Services — how does session configuration inheritance work?

```
Configure → Global Services → Session
    │
    │  Sets defaults for ALL realms:
    │  Max Session Time: 120 min
    │  Max Idle Time: 30 min
    │  Quotas: OFF
    │  Client-Side Sessions security (signing/encryption keys)
    │
    ▼
Realms → /techcorp → Authentication → Settings
    │
    │  Realm-level OVERRIDES:
    │  Use Client-Side Sessions: OFF (realm chooses stateful)
    │  Default Auth Level: 0
    │
    │  What it INHERITS (cannot override here):
    │  Max Session Time, Max Idle Time → comes from Global
    │  Session Quotas → comes from Global
    │  Signing/Encryption keys → comes from Global
```

**Key distinction**:
- **Global Session service**: Controls timeouts, quotas, CTS behavior, client-side session crypto. These are cluster-wide settings.
- **Realm Authentication Settings**: Controls whether a realm uses client-side sessions, auth level, identity types. These are per-realm choices.
- **Not all session settings are overridable at realm level.** Timeouts and quotas are global. The realm only chooses stateful vs stateless.

**Why this design**: Session timeouts and crypto keys must be consistent across the AM cluster (all AM instances must agree). But stateful vs stateless can vary per realm because it's a per-session decision at authentication time.

---

*Updated: 2026-02-02 — Added Q19-Q27 covering CTS reaper configuration, DS replication (deployment ID model, legacy dsreplication, topology design, troubleshooting, AM connection strategies).*
