# LDAP Replication - Concepts and Theory

## Table of Contents
1. [What is Replication?](#what-is-replication)
2. [Why Use Replication?](#why-use-replication)
3. [Replication Models](#replication-models)
4. [How Replication Works](#how-replication-works)
5. [Key Terminology](#key-terminology)
6. [Replication Topology](#replication-topology)

---

## What is Replication?

**LDAP Replication** is the process of copying and maintaining directory data across multiple LDAP servers (replicas) to ensure:
- **Consistency**: All servers have the same data
- **Availability**: If one server fails, others continue serving requests
- **Performance**: Distribute read/write load across multiple servers

Think of it like having multiple copies of a book in different libraries - if one library closes, you can still access the book at another location.

---

## Why Use Replication?

### 1. **High Availability (HA)**
- If one server crashes, clients automatically fail over to another replica
- No single point of failure
- Example: Authentication services remain available 24/7

### 2. **Load Distribution**
- Multiple servers handle read requests simultaneously
- Reduces load on individual servers
- Improves response time for users

### 3. **Geographic Distribution**
- Place replicas closer to users in different regions
- Reduces network latency
- Example: US office and Asia office each have local LDAP replicas

### 4. **Disaster Recovery**
- If primary datacenter fails, secondary datacenter takes over
- Data is already synchronized, minimal downtime
- Backup strategy for critical directory data

### 5. **Performance Optimization**
- Read operations can be distributed across all replicas
- Write operations can be directed to specific servers
- Caching and indexing can be optimized per replica

---

## Replication Models

### 1. **Single-Master Replication (Traditional)**
```
┌─────────────┐
│   MASTER    │ ← All writes go here
│  (Read/Write)│
└──────┬──────┘
       │ Replicates to ↓
   ┌───┴────┬────────┐
   ▼        ▼        ▼
┌──────┐ ┌──────┐ ┌──────┐
│REPLICA│ │REPLICA│ │REPLICA│
│(Read) │ │(Read) │ │(Read) │
└───────┘ └───────┘ └───────┘
```

**Characteristics:**
- One master server accepts all write operations
- Replicas only handle read operations
- Simple conflict resolution (no conflicts since only one writer)
- Master failure = no writes until recovery

**Use Case:** Simple deployments where write availability is not critical

---

### 2. **Multi-Master Replication (Modern - Used by PingDS/OpenDJ)**
```
┌─────────────┐         ┌─────────────┐
│  MASTER 1   │ ←─────→ │  MASTER 2   │
│ (Read/Write)│  Sync   │(Read/Write) │
└──────┬──────┘         └──────┬──────┘
       │                       │
       │ Replicates            │ Replicates
       ▼                       ▼
┌─────────────┐         ┌─────────────┐
│  MASTER 3   │ ←─────→ │  MASTER 4   │
│ (Read/Write)│  Sync   │(Read/Write) │
└─────────────┘         └─────────────┘
```

**Characteristics:**
- All servers accept read AND write operations
- Changes made on any server replicate to all others
- More complex conflict resolution needed
- No single point of failure for writes

**Use Case:** High-availability production environments (PingDS default)

---

## How Replication Works

### Step-by-Step Process

1. **Change on Server A**
   ```
   User modifies: uid=jdoe,ou=people,ou=identities
   Server A records: Change Sequence Number (CSN)
   ```

2. **Change Log Creation**
   ```
   Server A writes to replication changelog:
   - What changed: uid=jdoe's telephone number
   - When: CSN 001234567890
   - Where: Server A
   ```

3. **Propagation to Other Servers**
   ```
   Server A → Server B: "I have change CSN 001234567890"
   Server B checks: "Do I have this CSN?"
   Server B: "No, send it to me"
   Server A → Server B: [Change details]
   ```

4. **Application of Change**
   ```
   Server B applies the change
   Server B updates its CSN to 001234567890
   Server B confirms: "Change applied successfully"
   ```

5. **Conflict Resolution (if needed)**
   ```
   If Server B also modified the same attribute:
   - Compare CSNs (timestamps)
   - Latest change wins
   - Losing change is rolled back
   ```

---

## Key Terminology

### 1. **Replica**
A copy of the directory database. Each server in a replication topology maintains its own replica.

### 2. **Replication Server**
A lightweight server that manages the replication changelog and coordinates synchronization between directory servers.

### 3. **Change Sequence Number (CSN)**
Unique identifier for each change, includes:
- Timestamp (when change occurred)
- Server ID (which server made the change)
- Sequence number (order of changes on that server)

Example: `000001234567890000000001`

### 4. **Changelog**
A log of all changes (adds, modifies, deletes) maintained by the replication server. Used to synchronize replicas.

### 5. **Replication Domain**
A specific portion of the directory tree being replicated.
Example: `ou=identities` is a replication domain

### 6. **Server ID**
Unique identifier assigned to each directory server in the topology.
Must be unique across all servers (typically 1, 2, 3, etc.)

### 7. **Replication Port**
TCP port used for replication communication between servers.
Default: 8989 (configurable)

### 8. **Assured Replication**
Guarantees that changes are replicated to other servers before acknowledging the client. Two modes:
- **Safe Data**: Change replicated to at least N servers
- **Safe Read**: Change visible on at least N servers

### 9. **Fractional Replication**
Replicating only specific attributes (not entire entries).
Example: Replicate user accounts but exclude sensitive attributes like SSN.

---

## Replication Topology

### 1. **Full Mesh Topology (Recommended)**
```
    A ←→ B
    ↕ ✕ ↕
    C ←→ D
```
Every server replicates with every other server directly.

**Pros:**
- Fastest convergence (changes propagate quickly)
- Most resilient (multiple paths for updates)

**Cons:**
- More network connections
- More complex setup

---

### 2. **Hub-and-Spoke Topology**
```
       HUB
     ↙ ↓ ↘
    A  B  C
```
All servers replicate through a central hub.

**Pros:**
- Fewer connections
- Simpler configuration

**Cons:**
- Hub is single point of failure for replication
- Slower convergence (2-hop propagation)

---

### 3. **Cascading Topology**
```
A → B → C → D
```
Changes flow in one direction through a chain.

**Pros:**
- Minimal connections
- Good for geographically distributed sites

**Cons:**
- Slow convergence
- Chain breaks if any server fails

---

## Replication Configuration Elements

### 1. **Directory Server Configuration**
- Server ID
- Replication port
- Bind DN for replication
- Which base DNs to replicate

### 2. **Replication Server Configuration**
- Replication server ID
- Replication port
- Replication servers to connect to

### 3. **Assured Replication (Optional)**
- Assurance level (safe-data or safe-read)
- Timeout values
- Minimum server count

---

## Common Replication Scenarios

### Scenario 1: Add Second Server for HA
```
Before:  [Server A] ← All clients

After:   [Server A] ←→ [Server B]
              ↑           ↑
           Clients     Clients
```

### Scenario 2: Geographic Distribution
```
US Datacenter:          Europe Datacenter:
[Server US-1]     ←→    [Server EU-1]
     ↑                       ↑
  US Users              EU Users
```

### Scenario 3: Read Replica for Reporting
```
Production:            Reporting:
[Server Prod]    →    [Server Report]
     ↑                     ↑
  Live Apps          Analytics/Reports
```

---

## Best Practices

1. **Use Odd Number of Servers**
   - 3, 5, or 7 servers for better consensus
   - Helps with quorum-based decisions

2. **Monitor Replication Lag**
   - Track delay between servers
   - Alert if lag exceeds threshold (e.g., 5 seconds)

3. **Plan for Network Partitions**
   - What happens if datacenter link fails?
   - Configure appropriate timeouts

4. **Secure Replication Traffic**
   - Use SSL/TLS for replication connections
   - Authentication between servers

5. **Test Failover Scenarios**
   - Practice server failure and recovery
   - Verify clients can connect to alternate servers

6. **Backup Strategy**
   - Replication ≠ Backup
   - Still need regular backups
   - Corrupted data replicates too!

---

## Next Steps

1. Read `REPLICATION_SETUP_GUIDE.md` for practical setup instructions
2. Read `REPLICATION_COMMANDS.md` for command reference
3. Practice with 2-server replication topology
4. Monitor and troubleshoot replication

---

## Quick Reference

| Concept | Purpose |
|---------|---------|
| Replica | Copy of directory data on a server |
| Replication Server | Coordinates synchronization |
| CSN | Unique ID for each change |
| Changelog | Log of all directory changes |
| Server ID | Unique identifier for each server |
| Multi-Master | All servers accept writes |
| Assured Replication | Guarantee change propagation |

---

**Study Tip:** Understand the "why" before the "how". Replication solves availability, performance, and disaster recovery challenges. The complexity is justified by these critical benefits.
