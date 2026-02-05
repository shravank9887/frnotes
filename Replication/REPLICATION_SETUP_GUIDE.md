# PingDS Replication Setup Guide

## Overview

This guide walks you through setting up basic replication between two PingDS instances using Docker containers.

**Current Setup:** You have 1 PingDS instance running
**Goal:** Add a second instance and configure multi-master replication

---

## Table of Contents
1. [Prerequisites](#prerequisites)
2. [Architecture Overview](#architecture-overview)
3. [Step 1: Update Docker Compose](#step-1-update-docker-compose)
4. [Step 2: Start Second Instance](#step-2-start-second-instance)
5. [Step 3: Enable Replication](#step-3-enable-replication)
6. [Step 4: Initialize Replication](#step-4-initialize-replication)
7. [Step 5: Verify Replication](#step-5-verify-replication)
8. [Step 6: Test Replication](#step-6-test-replication)
9. [Troubleshooting](#troubleshooting)

---

## Prerequisites

✅ **What You Have:**
- One running PingDS instance (pingds)
- Sample users and groups already imported
- Network: `fr-net`
- Admin password: `Passw0rd123`

✅ **What You Need:**
- Docker and Docker Compose
- Access to ports: 8989, 8990 (for replication)
- Basic understanding of LDAP

---

## Architecture Overview

### Before (Current)
```
┌─────────────────────┐
│      pingds         │
│   (Single Server)   │
│  Port 1389, 1636    │
└─────────────────────┘
```

### After (With Replication)
```
┌─────────────────────┐         ┌─────────────────────┐
│      pingds1        │←──────→│      pingds2        │
│   (Master Server)   │  Sync  │   (Master Server)   │
│  LDAP: 1389, 1636   │  8989  │  LDAP: 2389, 2636   │
│  Admin: 4444        │        │  Admin: 5444        │
│  Repl: 8989         │        │  Repl: 8990         │
└─────────────────────┘         └─────────────────────┘
```

**Key Points:**
- Both servers accept read AND write operations (multi-master)
- Changes on either server replicate to the other
- Port 8989 and 8990 are used for replication communication

---

## Step 1: Update Docker Compose

### Create New docker-compose File

Replace your existing `docker-compose.yaml` with this updated version:

```yaml
# yaml-language-server: $schema=https://raw.githubusercontent.com/compose-spec/compose-spec/master/schema/compose-spec.json

services:
  # First PingDS Instance
  pingds1:
    build:
      context: ./pingds
      dockerfile: Dockerfile
    image: pingds:latest
    container_name: pingds1
    hostname: pingds1
    networks:
      fr-net:
        aliases:
          - pingds1
          - ds1.example.com
    ports:
      - "1389:1389"   # LDAP
      - "1636:1636"   # LDAPS
      - "4444:4444"   # Admin connector
      - "8080:8080"   # HTTP
      - "8443:8443"   # HTTPS
      - "8989:8989"   # Replication port
    environment:
      - DS_HOSTNAME=pingds1
      - DS_SERVER_ID=ds-server-01
      - DS_DEPLOYMENT_ID=forgerock-eval
      - DS_DEPLOYMENT_PASSWORD=Passw0rd123
      - DS_ROOT_PASSWORD=Passw0rd123
      - DS_MONITOR_PASSWORD=Passw0rd123
      - AM_CONFIG_PASSWORD=Passw0rd123
      - AM_IDENTITY_PASSWORD=Passw0rd123
      - OPENDJ_JAVA_ARGS=-server -Xmx1g -XX:+UseG1GC
    volumes:
      - pingds1-data:/opt/pingds-data
      - shared-certs:/opt/certs
      - pingds1-backups:/opt/backups
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "/opt/opendj/bin/status", "--hostname", "pingds1", "--port", "4444", "--bindDN", "cn=Directory Manager", "--bindPassword", "Passw0rd123", "--trustAll"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 90s

  # Second PingDS Instance
  pingds2:
    build:
      context: ./pingds
      dockerfile: Dockerfile
    image: pingds:latest
    container_name: pingds2
    hostname: pingds2
    networks:
      fr-net:
        aliases:
          - pingds2
          - ds2.example.com
    ports:
      - "2389:1389"   # LDAP (mapped to 2389 on host)
      - "2636:1636"   # LDAPS (mapped to 2636 on host)
      - "5444:4444"   # Admin connector (mapped to 5444 on host)
      - "9080:8080"   # HTTP (mapped to 9080 on host)
      - "9443:8443"   # HTTPS (mapped to 9443 on host)
      - "8990:8989"   # Replication port (mapped to 8990 on host)
    environment:
      - DS_HOSTNAME=pingds2
      - DS_SERVER_ID=ds-server-02
      - DS_DEPLOYMENT_ID=forgerock-eval
      - DS_DEPLOYMENT_PASSWORD=Passw0rd123
      - DS_ROOT_PASSWORD=Passw0rd123
      - DS_MONITOR_PASSWORD=Passw0rd123
      - AM_CONFIG_PASSWORD=Passw0rd123
      - AM_IDENTITY_PASSWORD=Passw0rd123
      - OPENDJ_JAVA_ARGS=-server -Xmx1g -XX:+UseG1GC
    volumes:
      - pingds2-data:/opt/pingds-data
      - shared-certs:/opt/certs
      - pingds2-backups:/opt/backups
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "/opt/opendj/bin/status", "--hostname", "pingds2", "--port", "4444", "--bindDN", "cn=Directory Manager", "--bindPassword", "Passw0rd123", "--trustAll"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 90s

volumes:
  pingds1-data:
    name: forgerock_pingds1_data
    driver: local

  pingds2-data:
    name: forgerock_pingds2_data
    driver: local

  pingds1-backups:
    name: forgerock_pingds1_backups
    driver: local

  pingds2-backups:
    name: forgerock_pingds2_backups
    driver: local

  shared-certs:
    name: forgerock_shared_certs
    driver: local

networks:
  fr-net:
    external: true
    name: fr-net
```

### Save Existing Data (IMPORTANT!)

Before making changes:

```bash
# 1. Backup your existing container data
docker exec pingds /opt/opendj/bin/backup \
  --hostname pingds \
  --port 4444 \
  --bindDN "cn=Directory Manager" \
  --bindPassword "Passw0rd123" \
  --backupDirectory /opt/backups \
  --backendID userRoot \
  --trustAll

# 2. Stop the current container
docker-compose down

# 3. Rename the volume to match new name
docker volume create forgerock_pingds1_data
docker run --rm -v forgerock_pingds_data:/from -v forgerock_pingds1_data:/to alpine sh -c "cd /from && cp -av . /to"
```

---

## Step 2: Start Second Instance

### Launch Both Containers

```bash
# Start both containers
docker-compose up -d

# Wait for both to be healthy (check logs)
docker-compose logs -f pingds1
docker-compose logs -f pingds2

# Verify both are running
docker ps | grep pingds
```

### Expected Output:
```
pingds1   Up 2 minutes (healthy)
pingds2   Up 2 minutes (healthy)
```

### Verify Connectivity

```bash
# Test pingds1
docker exec pingds1 /opt/opendj/bin/status \
  --hostname pingds1 \
  --port 4444 \
  --bindDN "cn=Directory Manager" \
  --bindPassword "Passw0rd123" \
  --trustAll

# Test pingds2
docker exec pingds2 /opt/opendj/bin/status \
  --hostname pingds2 \
  --port 4444 \
  --bindDN "cn=Directory Manager" \
  --bindPassword "Passw0rd123" \
  --trustAll
```

---

## Step 3: Enable Replication

### Enable Replication Between Servers

Run this command from **either** container (we'll use pingds1):

```bash
docker exec -it pingds1 /opt/opendj/bin/dsreplication enable \
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
  --adminPassword "AdminRepl123" \
  --trustAll \
  --no-prompt
```

### What This Does:
1. Configures replication on both servers
2. Sets up replication port 8989 on both servers
3. Creates replication admin user: `admin` with password `AdminRepl123`
4. Enables replication for base DN: `ou=identities`

### Expected Output:
```
Establishing connections ..... Done.
Checking registration information ..... Done.
Configuring Replication port on server pingds1:4444 ..... Done.
Configuring Replication port on server pingds2:4444 ..... Done.
Updating replication configuration for baseDN ou=identities on server pingds1:4444 ..... Done.
Updating replication configuration for baseDN ou=identities on server pingds2:4444 ..... Done.
Updating registration configuration on server pingds1:4444 ..... Done.
Updating registration configuration on server pingds2:4444 ..... Done.
Updating replication configuration for baseDN cn=schema on server pingds1:4444 ..... Done.
Updating replication configuration for baseDN cn=schema on server pingds2:4444 ..... Done.

Replication has been successfully enabled.
```

---

## Step 4: Initialize Replication

### Copy Data from Server1 to Server2

```bash
docker exec -it pingds1 /opt/opendj/bin/dsreplication initialize \
  --hostSource pingds1 \
  --portSource 4444 \
  --hostDestination pingds2 \
  --portDestination 4444 \
  --baseDN "ou=identities" \
  --adminUID admin \
  --adminPassword "AdminRepl123" \
  --trustAll \
  --no-prompt
```

### What This Does:
- Copies ALL data from `ou=identities` on pingds1 to pingds2
- Includes all users and groups you imported earlier
- Ensures both servers start with identical data

### Expected Output:
```
Initializing base DN ou=identities with the contents from pingds1:4444:
123 entries processed (100 % complete).

Base DN initialized successfully.
```

---

## Step 5: Verify Replication

### Check Replication Status

```bash
docker exec pingds1 /opt/opendj/bin/dsreplication status \
  --hostname pingds1 \
  --port 4444 \
  --adminUID admin \
  --adminPassword "AdminRepl123" \
  --trustAll
```

### Expected Output:
```
Suffix DN          : ou=identities
Server             : pingds1:4444
Server ID          : 16861
Replication Port   : 8989
Connected To       : pingds2:8989
Status             : Normal
Entries            : 123
Missing Changes    : 0
Approximate Delay  : 0 seconds

Suffix DN          : ou=identities
Server             : pingds2:4444
Server ID          : 16862
Replication Port   : 8989
Connected To       : pingds1:8989
Status             : Normal
Entries            : 123
Missing Changes    : 0
Approximate Delay  : 0 seconds
```

### Key Indicators of Success:
- ✅ **Status**: Normal (not Degraded or Error)
- ✅ **Missing Changes**: 0 (servers are in sync)
- ✅ **Approximate Delay**: < 1 second
- ✅ **Connected To**: Shows the other server
- ✅ **Entries**: Same count on both servers

---

## Step 6: Test Replication

### Test 1: Add User on Server1, Verify on Server2

```bash
# 1. Add user on pingds1
docker exec pingds1 /opt/opendj/bin/ldapmodify \
  --hostname pingds1 \
  --port 1636 \
  --useSSL \
  --trustAll \
  --bindDN "cn=Directory Manager" \
  --bindPassword "Passw0rd123" << 'EOF'
dn: uid=repltest,ou=people,ou=identities
changetype: add
objectClass: top
objectClass: person
objectClass: organizationalPerson
objectClass: inetOrgPerson
uid: repltest
cn: Replication Test
sn: Test
mail: repltest@example.com
userPassword: Test123
EOF

# 2. Verify user exists on pingds2 (should replicate within seconds)
docker exec pingds2 /opt/opendj/bin/ldapsearch \
  --hostname pingds2 \
  --port 1636 \
  --useSSL \
  --trustAll \
  --bindDN "cn=Directory Manager" \
  --bindPassword "Passw0rd123" \
  --baseDN "ou=identities" \
  "(uid=repltest)" \
  dn cn mail
```

### Expected Result:
```
dn: uid=repltest,ou=people,ou=identities
cn: Replication Test
mail: repltest@example.com
```

---

### Test 2: Modify User on Server2, Verify on Server1

```bash
# 1. Modify user on pingds2
docker exec pingds2 /opt/opendj/bin/ldapmodify \
  --hostname pingds2 \
  --port 1636 \
  --useSSL \
  --trustAll \
  --bindDN "cn=Directory Manager" \
  --bindPassword "Passw0rd123" << 'EOF'
dn: uid=repltest,ou=people,ou=identities
changetype: modify
replace: telephoneNumber
telephoneNumber: +1-555-9999
EOF

# 2. Verify change on pingds1
docker exec pingds1 /opt/opendj/bin/ldapsearch \
  --hostname pingds1 \
  --port 1636 \
  --useSSL \
  --trustAll \
  --bindDN "cn=Directory Manager" \
  --bindPassword "Passw0rd123" \
  --baseDN "ou=identities" \
  "(uid=repltest)" \
  telephoneNumber
```

### Expected Result:
```
dn: uid=repltest,ou=people,ou=identities
telephoneNumber: +1-555-9999
```

---

### Test 3: Delete User on Server1, Verify on Server2

```bash
# 1. Delete user on pingds1
docker exec pingds1 /opt/opendj/bin/ldapdelete \
  --hostname pingds1 \
  --port 1636 \
  --useSSL \
  --trustAll \
  --bindDN "cn=Directory Manager" \
  --bindPassword "Passw0rd123" \
  "uid=repltest,ou=people,ou=identities"

# 2. Verify deleted on pingds2
docker exec pingds2 /opt/opendj/bin/ldapsearch \
  --hostname pingds2 \
  --port 1636 \
  --useSSL \
  --trustAll \
  --bindDN "cn=Directory Manager" \
  --bindPassword "Passw0rd123" \
  --baseDN "ou=identities" \
  "(uid=repltest)"
```

### Expected Result:
```
# No entries found (user successfully deleted and replicated)
```

---

## Troubleshooting

### Issue 1: Servers Not Connected

**Symptom:** Status shows "Not Connected" or missing peer

**Solutions:**
```bash
# 1. Check network connectivity
docker exec pingds1 ping -c 3 pingds2

# 2. Check replication port is open
docker exec pingds1 nc -zv pingds2 8989

# 3. Check firewall rules (if using firewalld/iptables)

# 4. Restart replication
docker restart pingds1 pingds2
```

---

### Issue 2: High Missing Changes

**Symptom:** Missing Changes > 100

**Solutions:**
```bash
# Re-initialize the lagging server
docker exec pingds1 /opt/opendj/bin/dsreplication initialize \
  --hostSource pingds1 \
  --portSource 4444 \
  --hostDestination pingds2 \
  --portDestination 4444 \
  --baseDN "ou=identities" \
  --adminUID admin \
  --adminPassword "AdminRepl123" \
  --trustAll \
  --no-prompt
```

---

### Issue 3: Replication Delay > 5 Seconds

**Symptom:** Approximate Delay shows high value

**Possible Causes:**
1. Network latency between containers
2. High load on one server
3. Disk I/O bottleneck

**Solutions:**
```bash
# 1. Check container resource usage
docker stats pingds1 pingds2

# 2. Increase memory allocation in docker-compose.yaml
# Change: -Xmx1g to -Xmx2g

# 3. Check changelog size
docker exec pingds1 /opt/opendj/bin/dsconfig get-replication-server-prop \
  --hostname pingds1 \
  --port 4444 \
  --bindDN "cn=Directory Manager" \
  --bindPassword "Passw0rd123" \
  --provider-name "Multimaster Synchronization" \
  --trustAll
```

---

### Issue 4: Data Inconsistency

**Symptom:** Same entry has different values on different servers

**Solutions:**
```bash
# 1. Check for conflicts
docker exec pingds1 /opt/opendj/bin/ldapsearch \
  --hostname pingds1 \
  --port 1636 \
  --useSSL \
  --trustAll \
  --bindDN "cn=Directory Manager" \
  --bindPassword "Passw0rd123" \
  --baseDN "ou=identities" \
  "(ds-sync-conflict=*)"

# 2. Force re-initialization
docker exec pingds1 /opt/opendj/bin/dsreplication initialize \
  --hostSource pingds1 \
  --portSource 4444 \
  --hostDestination pingds2 \
  --portDestination 4444 \
  --baseDN "ou=identities" \
  --adminUID admin \
  --adminPassword "AdminRepl123" \
  --trustAll \
  --no-prompt
```

---

## Monitoring Replication

### Create Monitoring Script

Save as `pingds/scripts/monitor-replication.sh`:

```bash
#!/bin/bash
# Monitor replication status continuously

while true; do
  clear
  echo "==================================="
  echo "Replication Status - $(date)"
  echo "==================================="

  docker exec pingds1 /opt/opendj/bin/dsreplication status \
    --hostname pingds1 \
    --port 4444 \
    --adminUID admin \
    --adminPassword "AdminRepl123" \
    --trustAll \
    --script-friendly

  echo ""
  echo "Refreshing in 10 seconds... (Ctrl+C to exit)"
  sleep 10
done
```

Usage:
```bash
chmod +x pingds/scripts/monitor-replication.sh
./pingds/scripts/monitor-replication.sh
```

---

## Next Steps

### Add Third Server (Optional)

To create a 3-server topology:

1. Add `pingds3` service to docker-compose.yaml
2. Enable replication: `pingds1 ↔ pingds3`
3. Enable replication: `pingds2 ↔ pingds3`
4. Initialize pingds3 from pingds1

### Configure Client Applications

Update application LDAP URLs to use both servers:
```
ldap://pingds1:1389,pingds2:2389/ou=identities??sub?
```

Most LDAP clients support failover automatically.

---

## Quick Command Reference

```bash
# Check status
docker exec pingds1 /opt/opendj/bin/dsreplication status \
  --hostname pingds1 --port 4444 \
  --adminUID admin --adminPassword "AdminRepl123" --trustAll

# Re-initialize
docker exec pingds1 /opt/opendj/bin/dsreplication initialize \
  --hostSource pingds1 --portSource 4444 \
  --hostDestination pingds2 --portDestination 4444 \
  --baseDN "ou=identities" \
  --adminUID admin --adminPassword "AdminRepl123" \
  --trustAll --no-prompt

# Disable replication
docker exec pingds1 /opt/opendj/bin/dsreplication disable \
  --hostname pingds1 --port 4444 \
  --baseDN "ou=identities" \
  --adminUID admin --adminPassword "AdminRepl123" \
  --trustAll --disableAll
```

---

## Summary

✅ You now have:
- Two PingDS instances running in Docker
- Multi-master replication configured
- Data synchronized between both servers
- High availability setup

✅ Both servers can:
- Accept read and write operations
- Automatically sync changes
- Fail over if one server goes down

✅ You learned:
- How to configure replication
- How to verify replication status
- How to test replication
- How to troubleshoot issues

**Congratulations!** You have a working replicated LDAP environment! 🎉
