# LDAP Replication - Quick Reference Card

## 📋 One-Page Cheat Sheet

---

## Core Concepts

| Term | Definition |
|------|------------|
| **Replica** | Copy of directory data on a server |
| **Multi-Master** | All servers accept reads AND writes |
| **CSN** | Change Sequence Number - unique ID for each change |
| **Changelog** | Log of all directory modifications |
| **Replication Port** | Port used for server-to-server sync (default: 8989) |
| **Server ID** | Unique identifier for each server in topology |

---

## Essential Commands

### Enable Replication (First Time Setup)
```bash
dsreplication enable \
  --host1 pingds1 --port1 4444 --bindDN1 "cn=Directory Manager" --bindPassword1 "Passw0rd123" --replicationPort1 8989 \
  --host2 pingds2 --port2 4444 --bindDN2 "cn=Directory Manager" --bindPassword2 "Passw0rd123" --replicationPort2 8989 \
  --baseDN "ou=identities" \
  --adminUID admin --adminPassword "AdminRepl123" \
  --trustAll --no-prompt
```

### Initialize Replica (Copy Data)
```bash
dsreplication initialize \
  --hostSource pingds1 --portSource 4444 \
  --hostDestination pingds2 --portDestination 4444 \
  --baseDN "ou=identities" \
  --adminUID admin --adminPassword "AdminRepl123" \
  --trustAll --no-prompt
```

### Check Status
```bash
dsreplication status \
  --hostname pingds1 --port 4444 \
  --adminUID admin --adminPassword "AdminRepl123" \
  --trustAll
```

### Disable Replication
```bash
dsreplication disable \
  --hostname pingds1 --port 4444 \
  --baseDN "ou=identities" \
  --adminUID admin --adminPassword "AdminRepl123" \
  --trustAll --disableAll
```

---

## Status Indicators

### Healthy Replication
```
Status             : Normal
Missing Changes    : 0 (or < 10)
Approximate Delay  : 0 seconds (or < 1 second)
Connected To       : Shows all peer servers
```

### Unhealthy Replication
```
Status             : Degraded / Not Connected
Missing Changes    : > 100
Approximate Delay  : > 5 seconds
Connected To       : Missing servers
```

---

## Common Issues & Fixes

| Problem | Quick Fix |
|---------|-----------|
| **Not Connected** | `docker restart pingds1 pingds2` |
| **High Missing Changes** | Re-initialize lagging server |
| **Replication Delay** | Check network, increase memory |
| **Data Inconsistency** | Re-initialize or check for conflicts |
| **Port Already in Use** | Change replication port in config |

---

## Testing Workflow

```bash
# 1. Add user on Server1
ldapmodify on pingds1 → Add uid=test

# 2. Verify on Server2 (should appear within seconds)
ldapsearch on pingds2 → Search uid=test

# 3. Modify on Server2
ldapmodify on pingds2 → Change attribute

# 4. Verify on Server1
ldapsearch on pingds1 → See updated attribute

# 5. Delete on Server1
ldapdelete on pingds1 → Remove uid=test

# 6. Verify on Server2 (should be gone)
ldapsearch on pingds2 → No results
```

---

## Docker Commands (for our setup)

```bash
# Start containers
docker-compose up -d

# Check health
docker ps | grep pingds

# View logs
docker-compose logs -f pingds1

# Enter container
docker exec -it pingds1 bash

# Check replication status (from host)
docker exec pingds1 /opt/opendj/bin/dsreplication status \
  --hostname pingds1 --port 4444 \
  --adminUID admin --adminPassword "AdminRepl123" --trustAll
```

---

## Port Mapping (Our Setup)

### PingDS1
| Service | Container | Host |
|---------|-----------|------|
| LDAP | 1389 | 1389 |
| LDAPS | 1636 | 1636 |
| Admin | 4444 | 4444 |
| Replication | 8989 | 8989 |

### PingDS2
| Service | Container | Host |
|---------|-----------|------|
| LDAP | 1389 | 2389 |
| LDAPS | 1636 | 2636 |
| Admin | 4444 | 5444 |
| Replication | 8989 | 8990 |

---

## Replication Topology Patterns

### Full Mesh (Recommended)
```
A ←→ B
↕ ✕ ↕
C ←→ D
```
Fastest convergence, most resilient

### Hub-and-Spoke
```
   HUB
  ↙ ↓ ↘
 A  B  C
```
Simpler, but hub is single point of failure

### Cascading
```
A → B → C → D
```
Minimal connections, slowest convergence

---

## Monitoring Script

Create `monitor-replication.sh`:
```bash
#!/bin/bash
while true; do
  clear
  docker exec pingds1 /opt/opendj/bin/dsreplication status \
    --hostname pingds1 --port 4444 \
    --adminUID admin --adminPassword "AdminRepl123" \
    --trustAll
  sleep 10
done
```

---

## Emergency Procedures

### Server Out of Sync
```bash
# Re-initialize from healthy server
dsreplication initialize \
  --hostSource pingds1 --hostDestination pingds2 \
  --baseDN "ou=identities" \
  --adminUID admin --adminPassword "AdminRepl123" \
  --trustAll --no-prompt
```

### Server Crashed
```bash
# 1. Restart server
docker restart pingds2

# 2. Check if replication resumes
dsreplication status ...

# 3. If not, re-initialize
dsreplication initialize ...
```

### Network Partition
```bash
# Wait for network to recover, then:
# 1. Check status
dsreplication status ...

# 2. Resolve conflicts if any
ldapsearch "(ds-sync-conflict=*)"

# 3. Re-initialize if needed
```

---

## Best Practices

✅ **DO:**
- Monitor replication status regularly
- Test failover scenarios
- Keep servers in sync (Missing Changes < 10)
- Use consistent server IDs
- Secure replication traffic (SSL/TLS)
- Backup regularly (replication ≠ backup!)

❌ **DON'T:**
- Assume replication works without testing
- Ignore high missing changes
- Use same server ID on different servers
- Rely only on replication for disaster recovery
- Skip monitoring

---

## Study Tips

1. **Understand the "why"**: Replication = High Availability + Performance
2. **Practice the workflow**: Enable → Initialize → Verify → Test
3. **Monitor continuously**: Status should always be "Normal"
4. **Test failures**: Stop a server, verify failover works
5. **Know the ports**: Replication uses port 8989 by default

---

## Additional Resources

- `REPLICATION_CONCEPTS.md` - Detailed theory and concepts
- `REPLICATION_COMMANDS.md` - Complete command reference
- `REPLICATION_SETUP_GUIDE.md` - Step-by-step setup instructions

---

**Print this page and keep it handy for quick reference!**
