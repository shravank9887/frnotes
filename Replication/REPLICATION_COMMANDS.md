# PingDS/OpenDJ Replication Commands Reference

## Table of Contents
1. [Command Design Patterns](#command-design-patterns)
2. [Enable Replication](#enable-replication)
3. [Initialize Replication](#initialize-replication)
4. [Monitor Replication](#monitor-replication)
5. [Troubleshoot Replication](#troubleshoot-replication)
6. [Advanced Operations](#advanced-operations)

---

## Command Design Patterns

### Pattern 1: Interactive Mode (Recommended for Learning)
```bash
/opt/opendj/bin/dsreplication enable
# System prompts you for:
# - First server hostname, port, credentials
# - Second server hostname, port, credentials
# - Base DN to replicate
# - Replication port
```

**Pros:** Safe, guided, hard to make mistakes
**Cons:** Not automatable, slower for multiple servers

---

### Pattern 2: Non-Interactive Mode (For Automation)
```bash
/opt/opendj/bin/dsreplication enable \
  --host1 server1.example.com \
  --port1 4444 \
  --bindDN1 "cn=Directory Manager" \
  --bindPassword1 password1 \
  --replicationPort1 8989 \
  --host2 server2.example.com \
  --port2 4444 \
  --bindDN2 "cn=Directory Manager" \
  --bindPassword2 password2 \
  --replicationPort2 8989 \
  --baseDN "ou=identities" \
  --adminUID admin \
  --adminPassword adminPassword \
  --no-prompt \
  --trustAll
```

**Pros:** Scriptable, repeatable, fast
**Cons:** Complex syntax, easy to make mistakes

---

### Pattern 3: Batch Configuration (Multiple Base DNs)
```bash
# Enable replication for multiple suffixes in one command
/opt/opendj/bin/dsreplication enable \
  ... (connection params) ... \
  --baseDN "ou=identities" \
  --baseDN "ou=applications" \
  --baseDN "ou=devices" \
  --no-prompt
```

---

## Enable Replication

### Command: `dsreplication enable`

Establishes replication relationship between two servers.

### Basic Syntax
```bash
dsreplication enable \
  --host1 <hostname> \
  --port1 <admin-port> \
  --bindDN1 <bind-dn> \
  --bindPassword1 <password> \
  --replicationPort1 <repl-port> \
  --host2 <hostname> \
  --port2 <admin-port> \
  --bindDN2 <bind-dn> \
  --bindPassword2 <password> \
  --replicationPort2 <repl-port> \
  --baseDN <suffix-dn> \
  --adminUID <admin-id> \
  --adminPassword <admin-pwd> \
  [--trustAll] \
  [--no-prompt]
```

### Parameter Explanation

| Parameter | Description | Example |
|-----------|-------------|---------|
| `--host1` | First server hostname/IP | `pingds1` or `192.168.1.10` |
| `--port1` | First server admin port | `4444` (default) |
| `--bindDN1` | Admin DN for first server | `cn=Directory Manager` |
| `--bindPassword1` | Admin password | `Passw0rd123` |
| `--replicationPort1` | Replication port on first server | `8989` (default) |
| `--host2` | Second server hostname/IP | `pingds2` |
| `--port2` | Second server admin port | `4444` |
| `--bindDN2` | Admin DN for second server | `cn=Directory Manager` |
| `--bindPassword2` | Admin password | `Passw0rd123` |
| `--replicationPort2` | Replication port on second server | `8989` |
| `--baseDN` | Base DN to replicate | `ou=identities` |
| `--adminUID` | Replication admin username | `admin` |
| `--adminPassword` | Replication admin password | `AdminPass123` |
| `--trustAll` | Trust all SSL certificates | (flag, no value) |
| `--no-prompt` | Non-interactive mode | (flag, no value) |

### Example: Enable Replication Between Two Servers
```bash
dsreplication enable \
  --host1 pingds1 \
  --port1 4444 \
  --bindDN1 "cn=Directory Manager" \
  --bindPassword1 "Passw0rd123" \
  --replicationPort1 8989 \
  --host2 pingds2 \
  --port2 4444 \
  --bindDN2 "cn=Directory Manager" \
  --bindPassword2 "Passw0rd123" \
  --replicationPort2 8989 \
  --baseDN "ou=identities" \
  --adminUID admin \
  --adminPassword "AdminPass123" \
  --trustAll \
  --no-prompt
```

---

## Initialize Replication

### Command: `dsreplication initialize`

Copies data from one server (source) to another (destination).

### When to Use
- After enabling replication for the first time
- When a server is out of sync
- When adding a new replica to existing topology

### Basic Syntax
```bash
dsreplication initialize \
  --hostSource <source-host> \
  --portSource <source-port> \
  --hostDestination <dest-host> \
  --portDestination <dest-port> \
  --baseDN <suffix-dn> \
  --adminUID <admin-id> \
  --adminPassword <admin-pwd> \
  [--trustAll] \
  [--no-prompt]
```

### Example: Initialize from Server1 to Server2
```bash
dsreplication initialize \
  --hostSource pingds1 \
  --portSource 4444 \
  --hostDestination pingds2 \
  --portDestination 4444 \
  --baseDN "ou=identities" \
  --adminUID admin \
  --adminPassword "AdminPass123" \
  --trustAll \
  --no-prompt
```

**Important:** This OVERWRITES all data on the destination server for the specified base DN!

---

## Monitor Replication

### Command: `dsreplication status`

Shows current replication status for all servers in the topology.

### Basic Syntax
```bash
dsreplication status \
  --hostname <server> \
  --port <admin-port> \
  --adminUID <admin-id> \
  --adminPassword <admin-pwd> \
  [--trustAll] \
  [--script-friendly]
```

### Example: Check Replication Status
```bash
dsreplication status \
  --hostname pingds1 \
  --port 4444 \
  --adminUID admin \
  --adminPassword "AdminPass123" \
  --trustAll
```

### Sample Output
```
Suffix DN          : ou=identities
Server             : pingds1:4444
Server ID          : 1
Replication Port   : 8989
Connected To       : pingds2:8989
Status             : Normal
Entries            : 1523
Missing Changes    : 0
Approximate Delay  : 0 seconds

Suffix DN          : ou=identities
Server             : pingds2:4444
Server ID          : 2
Replication Port   : 8989
Connected To       : pingds1:8989
Status             : Normal
Entries            : 1523
Missing Changes    : 0
Approximate Delay  : 0 seconds
```

### Key Fields to Monitor

| Field | Meaning | Good Value | Bad Value |
|-------|---------|------------|-----------|
| Status | Server status | Normal | Degraded, Not Connected |
| Missing Changes | Number of changes not yet applied | 0 or low number | High number (> 1000) |
| Approximate Delay | Time lag behind other servers | < 1 second | > 5 seconds |
| Connected To | Other servers in topology | Shows all peers | Missing servers |

---

## Troubleshoot Replication

### Command: `dsreplication pre-external-initialization`

Prepares server to be initialized from external backup.

```bash
dsreplication pre-external-initialization \
  --hostname <server> \
  --port <admin-port> \
  --baseDN <suffix-dn> \
  --adminUID <admin-id> \
  --adminPassword <admin-pwd> \
  --trustAll
```

### Command: `dsreplication post-external-initialization`

Finalizes server after initialization from external backup.

```bash
dsreplication post-external-initialization \
  --hostname <server> \
  --port <admin-port> \
  --baseDN <suffix-dn> \
  --adminUID <admin-id> \
  --adminPassword <admin-pwd> \
  --trustAll
```

### Command: `dsreplication clear-changelog`

Clears the replication changelog (use with caution!).

```bash
dsreplication clear-changelog \
  --hostname <server> \
  --port <admin-port> \
  --adminUID <admin-id> \
  --adminPassword <admin-pwd> \
  --trustAll
```

**Warning:** Only use when directed by support or documentation!

---

## Advanced Operations

### Disable Replication

```bash
dsreplication disable \
  --hostname <server> \
  --port <admin-port> \
  --baseDN <suffix-dn> \
  --adminUID <admin-id> \
  --adminPassword <admin-pwd> \
  --trustAll \
  --disableAll
```

### Enable Assured Replication

Modify configuration to require acknowledgment before returning success.

```bash
dsconfig set-replication-domain-prop \
  --hostname <server> \
  --port <admin-port> \
  --bindDN "cn=Directory Manager" \
  --bindPassword <password> \
  --provider-name "Multimaster Synchronization" \
  --domain-name <suffix-dn> \
  --set assured-type:safe-data \
  --set assured-sd-level:1 \
  --trustAll \
  --no-prompt
```

### View Replication Configuration

```bash
dsconfig get-replication-domain-prop \
  --hostname <server> \
  --port <admin-port> \
  --bindDN "cn=Directory Manager" \
  --bindPassword <password> \
  --provider-name "Multimaster Synchronization" \
  --domain-name <suffix-dn> \
  --trustAll
```

### List All Replication Servers

```bash
dsconfig list-replication-servers \
  --hostname <server> \
  --port <admin-port> \
  --bindDN "cn=Directory Manager" \
  --bindPassword <password> \
  --provider-name "Multimaster Synchronization" \
  --trustAll
```

---

## Common Workflows

### Workflow 1: Add New Server to Existing Topology

```bash
# Step 1: Enable replication between existing server and new server
dsreplication enable \
  --host1 pingds1 --port1 4444 \
  --host2 pingds3 --port2 4444 \
  --baseDN "ou=identities" \
  --adminUID admin --adminPassword "AdminPass123" \
  --no-prompt --trustAll

# Step 2: Initialize new server from existing server
dsreplication initialize \
  --hostSource pingds1 --portSource 4444 \
  --hostDestination pingds3 --portDestination 4444 \
  --baseDN "ou=identities" \
  --adminUID admin --adminPassword "AdminPass123" \
  --no-prompt --trustAll

# Step 3: Verify status
dsreplication status \
  --hostname pingds1 --port 4444 \
  --adminUID admin --adminPassword "AdminPass123" \
  --trustAll
```

---

### Workflow 2: Fix Out-of-Sync Replica

```bash
# Step 1: Check status to identify problem
dsreplication status \
  --hostname pingds1 --port 4444 \
  --adminUID admin --adminPassword "AdminPass123" \
  --trustAll

# Step 2: Re-initialize the problematic server
dsreplication initialize \
  --hostSource pingds1 --portSource 4444 \
  --hostDestination pingds2 --portDestination 4444 \
  --baseDN "ou=identities" \
  --adminUID admin --adminPassword "AdminPass123" \
  --no-prompt --trustAll

# Step 3: Verify sync is restored
dsreplication status \
  --hostname pingds1 --port 4444 \
  --adminUID admin --adminPassword "AdminPass123" \
  --trustAll
```

---

### Workflow 3: Monitor Replication Continuously

```bash
# Check status every 30 seconds
watch -n 30 'dsreplication status \
  --hostname pingds1 --port 4444 \
  --adminUID admin --adminPassword "AdminPass123" \
  --trustAll --script-friendly'
```

---

## Command Comparison Table

| Task | Command | Key Options |
|------|---------|-------------|
| Set up replication | `dsreplication enable` | `--host1`, `--host2`, `--baseDN` |
| Copy data to replica | `dsreplication initialize` | `--hostSource`, `--hostDestination` |
| Check replication health | `dsreplication status` | `--hostname` |
| Remove replication | `dsreplication disable` | `--disableAll` |
| View configuration | `dsconfig get-replication-domain-prop` | `--domain-name` |
| Modify configuration | `dsconfig set-replication-domain-prop` | `--domain-name`, `--set` |

---

## Error Handling

### Common Errors and Solutions

**Error:** `Connection refused on port 8989`
```bash
# Solution: Check if replication port is open and not blocked by firewall
netstat -an | grep 8989
```

**Error:** `Servers already replicated`
```bash
# Solution: Use initialize instead of enable
dsreplication initialize ...
```

**Error:** `Server ID conflict`
```bash
# Solution: Each server must have unique server ID
# Check current IDs with status command
dsreplication status --hostname pingds1 --port 4444 ...
```

**Error:** `Missing changes > 10000`
```bash
# Solution: Re-initialize the lagging server
dsreplication initialize ...
```

---

## Best Practices

1. **Always verify status after changes**
   ```bash
   dsreplication status ...
   ```

2. **Use script-friendly mode for automation**
   ```bash
   dsreplication status --script-friendly ...
   ```

3. **Initialize after enabling replication**
   - Don't assume data will sync automatically
   - Explicitly initialize to ensure consistency

4. **Monitor regularly**
   - Set up automated monitoring
   - Alert on high missing changes or delays

5. **Test failover**
   - Stop one server and verify others continue working
   - Practice recovery procedures

---

## Quick Reference Card

```bash
# Enable replication
dsreplication enable --host1 <h1> --host2 <h2> --baseDN <dn> ...

# Initialize replica
dsreplication initialize --hostSource <src> --hostDestination <dst> --baseDN <dn> ...

# Check status
dsreplication status --hostname <host> --adminUID admin ...

# Disable replication
dsreplication disable --hostname <host> --disableAll ...
```

---

**Next Steps:** Proceed to `REPLICATION_SETUP_GUIDE.md` for hands-on setup with your existing PingDS instance.
