# PingAM + PingDS Quick Reference Guide

## 📋 One-Page Cheat Sheet

---

## Architecture at a Glance

```
PingAM (pingam:8081) ←LDAPS:1636→ PingDS (pingds)
       │
       ├─ Config Store: ou=am-config
       ├─ CTS Store: ou=tokens
       ├─ User Store: ou=identities
       └─ Policy Store: ou=services,ou=am-config
```

---

## Connection Details

### PingDS Connection (from PingAM)

| Parameter | Value |
|-----------|-------|
| **Hostname** | `pingds` |
| **LDAPS Port** | `1636` |
| **SSL/TLS** | Enabled |
| **Trust Mode** | Trust All (dev) / Certificate (prod) |

### PingAM Access URLs

| Service | URL |
|---------|-----|
| **AM Console** | `http://localhost:8081/am/console` |
| **AM Base** | `http://localhost:8081/am` |
| **XUI (End User)** | `http://localhost:8081/am/XUI` |
| **REST API** | `http://localhost:8081/am/json` |

---

## Service Account Credentials

| Store | Bind DN | Password |
|-------|---------|----------|
| **Config Store** | `uid=am-config,ou=admins,ou=am-config` | `AMConfig@2024` |
| **CTS Store** | `uid=openam_cts,ou=admins,ou=famrecords,ou=openam-session,ou=tokens` | `AMCTS@2024` |
| **User Store** | `uid=am-identity-bind-account,ou=admins,ou=identities` | `AMIdentity@2024` |

### AM Admin Credentials

| User | Password |
|------|----------|
| `amAdmin` | `Passw0rd123` (default - change!) |

---

## Base DNs

| Store | Base DN |
|-------|---------|
| **Config Store** | `ou=am-config` |
| **CTS Store** | `ou=famrecords,ou=openam-session,ou=tokens` |
| **User Store** | `ou=identities` |
| **Policy Store** | `ou=services,ou=am-config` |

---

## Docker Commands

### Start/Stop Services

```bash
# Start all services
docker-compose up -d

# Start only AM (DS must be running)
docker-compose up -d pingam

# Stop AM
docker-compose stop pingam

# Restart AM
docker restart pingam

# Stop all
docker-compose down
```

### View Logs

```bash
# Follow AM logs
docker logs -f pingam

# Last 100 lines
docker logs --tail 100 pingam

# Follow DS logs
docker logs -f pingds
```

### Container Access

```bash
# Enter AM container
docker exec -it pingam bash

# Enter DS container
docker exec -it pingds bash

# Run command in AM
docker exec pingam <command>
```

---

## AM REST API Examples

### Authenticate User

```bash
curl -X POST \
  'http://localhost:8081/am/json/authenticate' \
  -H 'Content-Type: application/json' \
  -H 'X-OpenAM-Username: jdoe' \
  -H 'X-OpenAM-Password: TestUser@2024' \
  -H 'Accept-API-Version: resource=2.0, protocol=1.0'
```

### Get Server Info

```bash
curl -s 'http://localhost:8081/am/json/serverinfo/*' \
  -H 'Accept-API-Version: resource=1.0'
```

### Validate Token

```bash
TOKEN="<token-from-authenticate>"
curl -s "http://localhost:8081/am/json/sessions/${TOKEN}?_action=validate" \
  -X POST \
  -H 'Content-Type: application/json' \
  -H 'Accept-API-Version: resource=3.1'
```

### Logout

```bash
curl -X POST \
  "http://localhost:8081/am/json/sessions/?_action=logout" \
  -H "Content-Type: application/json" \
  -H "iplanetDirectoryPro: ${TOKEN}"
```

---

## PingDS Verification Commands

### Test Service Accounts

```bash
# Test config store account
docker exec pingds /opt/opendj/bin/ldapsearch \
  --hostname pingds --port 1636 --useSSL --trustAll \
  --bindDN "uid=am-config,ou=admins,ou=am-config" \
  --bindPassword "AMConfig@2024" \
  --baseDN "ou=am-config" --searchScope base "(objectClass=*)"

# Test CTS account
docker exec pingds /opt/opendj/bin/ldapsearch \
  --hostname pingds --port 1636 --useSSL --trustAll \
  --bindDN "uid=openam_cts,ou=admins,ou=famrecords,ou=openam-session,ou=tokens" \
  --bindPassword "AMCTS@2024" \
  --baseDN "ou=tokens" --searchScope base "(objectClass=*)"

# Test identity account
docker exec pingds /opt/opendj/bin/ldapsearch \
  --hostname pingds --port 1636 --useSSL --trustAll \
  --bindDN "uid=am-identity-bind-account,ou=admins,ou=identities" \
  --bindPassword "AMIdentity@2024" \
  --baseDN "ou=identities" "(uid=jdoe)" dn cn
```

### View CTS Tokens (Active Sessions)

```bash
docker exec pingds /opt/opendj/bin/ldapsearch \
  --hostname pingds --port 1636 --useSSL --trustAll \
  --bindDN "cn=Directory Manager" \
  --bindPassword "Passw0rd123" \
  --baseDN "ou=famrecords,ou=openam-session,ou=tokens" \
  --searchScope sub "(objectClass=frCoreToken)" \
  dn coreTokenExpirationDate
```

### Count Entries

```bash
# Count users
docker exec pingds /opt/opendj/bin/ldapsearch \
  --hostname pingds --port 1636 --useSSL --trustAll \
  --bindDN "cn=Directory Manager" --bindPassword "Passw0rd123" \
  --baseDN "ou=identities" \
  "(objectClass=inetOrgPerson)" dn | grep "^dn:" | wc -l

# Count active sessions (CTS tokens)
docker exec pingds /opt/opendj/bin/ldapsearch \
  --hostname pingds --port 1636 --useSSL --trustAll \
  --bindDN "cn=Directory Manager" --bindPassword "Passw0rd123" \
  --baseDN "ou=tokens" \
  "(objectClass=frCoreToken)" dn | grep "^dn:" | wc -l
```

---

## Common Tasks

### Reset amAdmin Password

```bash
# Via AM console (logged in as amAdmin)
# Navigate: Realms > Top Level Realm > Subjects > amAdmin > Password

# Via ssoadm (command line)
docker exec pingam /opt/am/ssoadm set-identity-password \
  -u amAdmin \
  -w /path/to/password/file \
  -i amAdmin \
  -p <new-password>
```

### Add New User to PingDS

```bash
# Create LDIF
cat > /tmp/newuser.ldif << 'EOF'
dn: uid=testuser,ou=people,ou=identities
changetype: add
objectClass: top
objectClass: person
objectClass: organizationalPerson
objectClass: inetOrgPerson
uid: testuser
cn: Test User
sn: User
mail: testuser@example.com
userPassword: Test123
EOF

# Import
docker cp /tmp/newuser.ldif pingds:/tmp/
docker exec pingds /opt/opendj/bin/ldapmodify \
  --hostname pingds --port 1636 --useSSL --trustAll \
  --bindDN "cn=Directory Manager" --bindPassword "Passw0rd123" \
  --filename /tmp/newuser.ldif
```

### Verify User Can Authenticate in AM

```bash
curl -X POST \
  'http://localhost:8081/am/json/authenticate' \
  -H 'X-OpenAM-Username: testuser' \
  -H 'X-OpenAM-Password: Test123' \
  -H 'Accept-API-Version: resource=2.0, protocol=1.0'
```

---

## Certificate Management

### Export DS Certificate

```bash
docker exec pingds /opt/opendj/bin/dskeymgr export-ca-cert \
  --deploymentId forgerock-eval \
  --deploymentIdPassword Passw0rd123 \
  --outputFile /tmp/ds-ca-cert.pem
```

### Import into AM Truststore

```bash
# Copy cert to shared volume
docker cp pingds:/tmp/ds-ca-cert.pem ./shared-certs/

# Import into AM
docker exec pingam keytool -importcert \
  -file /opt/certs/ds-ca-cert.pem \
  -alias pingds-ca \
  -keystore /usr/local/tomcat/conf/keystores/truststore.jks \
  -storepass changeit \
  -noprompt

# Restart AM to apply
docker restart pingam
```

---

## Health Checks

### Check AM Status

```bash
# Health endpoint
curl -f http://localhost:8081/am/isAlive.jsp

# Should return: HTTP 200 with "true"

# Server info
curl -s http://localhost:8081/am/json/serverinfo/* \
  -H 'Accept-API-Version: resource=1.0' | jq
```

### Check DS Status

```bash
docker exec pingds /opt/opendj/bin/status \
  --hostname pingds --port 4444 \
  --bindDN "cn=Directory Manager" \
  --bindPassword "Passw0rd123" \
  --trustAll
```

### Check Container Health

```bash
# Docker health status
docker ps --filter name=pingam --format "table {{.Names}}\t{{.Status}}"
docker ps --filter name=pingds --format "table {{.Names}}\t{{.Status}}"

# Resource usage
docker stats --no-stream pingam pingds
```

---

## Troubleshooting Quick Fixes

### AM Won't Start

```bash
# Check logs
docker logs --tail 50 pingam

# Check if WAR deployed
docker exec pingam ls -lh /usr/local/tomcat/webapps/

# Increase memory
# Edit docker-compose.yaml: JAVA_OPTS=-Xmx2g (or higher)

# Rebuild
docker-compose down
docker-compose up -d --build pingam
```

### Cannot Connect to DS

```bash
# Test connectivity
docker exec pingam ping -c 3 pingds
docker exec pingam nc -zv pingds 1636

# Check DS is listening
docker exec pingds netstat -an | grep 1636

# Verify service account
docker exec pingds /opt/opendj/bin/ldapsearch \
  --hostname pingds --port 1636 --useSSL --trustAll \
  --bindDN "uid=am-config,ou=admins,ou=am-config" \
  --bindPassword "AMConfig@2024" \
  --baseDN "ou=am-config" --searchScope base "(objectClass=*)"
```

### User Authentication Fails

```bash
# Verify user exists
docker exec pingds /opt/opendj/bin/ldapsearch \
  --hostname pingds --port 1636 --useSSL --trustAll \
  --bindDN "cn=Directory Manager" --bindPassword "Passw0rd123" \
  --baseDN "ou=identities" "(uid=jdoe)" dn userPassword

# Check AM data store configuration
# Navigate: AM Console > Realms > Top Level Realm > Data Stores
# Click on user store > Test Connection

# Check AM authentication chain
# Navigate: AM Console > Realms > Top Level Realm > Authentication > Chains
```

---

## File Locations

### Inside PingAM Container

| Path | Purpose |
|------|---------|
| `/usr/local/tomcat/webapps/am` | AM application |
| `/usr/local/tomcat/logs` | Tomcat logs |
| `/opt/am-config` | AM configuration files (FBC) |
| `/usr/local/tomcat/conf/keystores` | Truststores/keystores |

### Inside PingDS Container

| Path | Purpose |
|------|---------|
| `/opt/opendj` | DS installation |
| `/opt/opendj/bin` | DS commands |
| `/opt/opendj/config` | DS configuration |
| `/opt/pingds-data` | DS data (mounted volume) |

---

## Port Mapping

| Service | Container Port | Host Port | Purpose |
|---------|---------------|-----------|---------|
| **PingDS** | 1389 | 1389 | LDAP |
| **PingDS** | 1636 | 1636 | LDAPS |
| **PingDS** | 4444 | 4444 | Admin |
| **PingAM** | 8080 | 8081 | HTTP |
| **PingAM** | 8443 | 8444 | HTTPS |

---

## Environment Variables (docker-compose.yaml)

### PingAM Key Variables

```yaml
# Java options
JAVA_OPTS: -server -Xmx2g -XX:+UseG1GC

# Truststore (for LDAPS)
-Djavax.net.ssl.trustStore=/usr/local/tomcat/conf/keystores/truststore.jks
-Djavax.net.ssl.trustStorePassword=changeit
```

---

## Backup & Restore

### Backup AM Configuration

```bash
# File-based config
docker exec pingam tar -czf /tmp/am-config-backup.tar.gz /opt/am-config
docker cp pingam:/tmp/am-config-backup.tar.gz ./backups/

# LDAP-based config
docker exec pingds /opt/opendj/bin/backup \
  --backendID amConfig \
  --backupDirectory /opt/backups \
  --hostname pingds --port 4444 \
  --bindDN "cn=Directory Manager" --bindPassword "Passw0rd123" \
  --trustAll
```

### Backup PingDS Data

```bash
# All backends
docker exec pingds /opt/opendj/bin/backup \
  --backupAll \
  --backupDirectory /opt/backups \
  --hostname pingds --port 4444 \
  --bindDN "cn=Directory Manager" --bindPassword "Passw0rd123" \
  --trustAll
```

---

## Study Resources

### Documentation Files

- `PINGAM_OVERVIEW.md` - Concepts and architecture
- `DATA_STORES_PREPARATION.md` - DS setup for AM
- `PINGAM_INSTALLATION_GUIDE.md` - Step-by-step install
- `PINGAM_QUICK_REFERENCE.md` - This file!

### Online Resources

- Ping Identity Docs: https://docs.pingidentity.com/pingam/8/
- Community: https://community.pingidentity.com/
- GitHub: https://github.com/ForgeRock/

---

## Common Scenarios

### Scenario 1: Fresh Start

```bash
# Stop everything
docker-compose down -v

# Remove volumes (WARNING: deletes data!)
docker volume rm forgerock_pingam_config forgerock_pingds_data

# Start fresh
docker-compose up -d
```

### Scenario 2: Update AM Configuration

```bash
# Make changes in AM Console
# Changes auto-save to config store (LDAP) or /opt/am-config (FBC)

# For LDAP-based: No additional action needed
# For FBC: Backup /opt/am-config
docker exec pingam tar -czf /tmp/config.tar.gz /opt/am-config
docker cp pingam:/tmp/config.tar.gz ./backups/
```

### Scenario 3: Add More Users

```bash
# Use existing sample-users.ldif or create new one
docker cp pingds/new-users.ldif pingds:/tmp/
docker exec pingds /opt/opendj/bin/ldapmodify \
  --hostname pingds --port 1636 --useSSL --trustAll \
  --bindDN "cn=Directory Manager" --bindPassword "Passw0rd123" \
  --filename /tmp/new-users.ldif

# Users immediately available in AM!
```

---

**Pro Tip:** Bookmark this page for quick reference during development and troubleshooting!
