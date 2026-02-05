# PingAM Installation Guide - Docker Setup

## Table of Contents
1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Architecture](#architecture)
4. [Installation Steps](#installation-steps)
5. [Configuration](#configuration)
6. [Verification](#verification)
7. [Troubleshooting](#troubleshooting)
8. [Next Steps](#next-steps)

---

## Overview

This guide walks through deploying **PingAM 8.0.2** in a Docker container with:
- **Tomcat 10** application server
- **JDK 21** runtime
- **Automated truststore setup** for LDAPS connectivity with PingDS
- **Production-ready** configuration with security best practices

---

## Prerequisites

### Required Components
- ✅ **PingDS container** running and healthy
- ✅ **Docker** and **Docker Compose** installed
- ✅ **fr-net** network created
- ✅ **PingAM WAR file** (AM-8.0.2.war)

### PingDS Must Be Running
Before starting PingAM, ensure PingDS is up and has exported its certificate:

```bash
# Check PingDS status
docker ps | grep pingds

# Verify certificate exists
docker exec pingds ls -la /opt/certs/
# Should show: ds-ca-cert.pem and truststore.p12
```

---

## Architecture

### Container Setup

```
┌──────────────────────────────────────────────────────────────┐
│                     Docker Network: fr-net                    │
│                                                               │
│  ┌─────────────────────┐         ┌─────────────────────┐    │
│  │  PingDS Container   │         │  PingAM Container   │    │
│  │                     │         │                     │    │
│  │  Port 1389 (LDAP)   │         │  Tomcat 10 + JDK 21 │    │
│  │  Port 1636 (LDAPS)  │◄────────┤  Port 8081 (HTTP)   │    │
│  │  Port 4444 (Admin)  │  LDAPS  │  Port 8444 (HTTPS)  │    │
│  │                     │         │                     │    │
│  │  ┌───────────────┐  │         │  Context: /am       │    │
│  │  │ ou=am-config  │  │         │  User: tomcat:1001  │    │
│  │  │ ou=tokens     │  │         │  Heap: 1GB-2GB      │    │
│  │  │ ou=identities │  │         │                     │    │
│  │  └───────────────┘  │         └─────────────────────┘    │
│  │                     │                                     │
│  │  /opt/certs/        │◄────────────────────────────────┐   │
│  │  - ds-ca-cert.pem   │   Shared Volume: shared-certs  │   │
│  │  - truststore.p12   │                                │   │
│  └─────────────────────┘                                │   │
│                                                          │   │
└──────────────────────────────────────────────────────────┘   │
                                                               │
                            Browser: http://localhost:8081/am  │
                                                               │
└──────────────────────────────────────────────────────────────┘
```

### Key Features
- **Automated Certificate Setup**: PingAM automatically imports PingDS truststore
- **Shared Volume**: `/opt/certs` shared between containers (read-only for AM)
- **Security**: Runs as non-root `tomcat` user (UID 1001)
- **JVM Tuning**: 1GB min heap, 2GB max heap, G1GC
- **Health Checks**: Automatic container health monitoring

---

## Installation Steps

### Step 1: Prepare PingAM Directory Structure

```bash
cd /c/PCFolders/Main/Learning/Docker/fr/sndbx1

# Create directory structure
mkdir -p pingam/{software,scripts,notes}
```

### Step 2: Obtain and Place PingAM WAR File

Download PingAM 8.0.2 from Ping Identity and place it in the software directory:

```bash
# Place the WAR file here:
# pingam/software/AM-8.0.2.war

# Verify
ls -lh pingam/software/AM-8.0.2.war
```

**Note**: Download from [Ping Identity Backstage](https://backstage.forgerock.com/) (requires account).

### Step 3: Review Configuration Files

Your setup should have these files:

```
pingam/
├── Dockerfile                      # Container definition
├── scripts/
│   └── docker-entrypoint.sh       # Startup script (handles truststore)
├── software/
│   └── AM-8.0.2.war               # PingAM application
└── notes/
    └── PINGAM_INSTALLATION_GUIDE.md
```

**Dockerfile** - Already created with:
- Tomcat 10 + JDK 21 base image
- JVM configuration (1GB-2GB heap)
- Tomcat user (UID 1001)
- Automatic truststore setup

**docker-entrypoint.sh** - Already created with:
- Waits for PingDS truststore
- Copies truststore to Tomcat keystores
- Configures SSL/TLS settings
- Starts Tomcat

### Step 4: Build PingAM Container

```bash
# From the sndbx1 directory
cd /c/PCFolders/Main/Learning/Docker/fr/sndbx1

# Build the image
docker-compose build pingam
```

Expected output:
```
Successfully built <image-id>
Successfully tagged pingam:latest
```

### Step 5: Start PingAM Container

```bash
# Start PingAM (PingDS should already be running)
docker-compose up -d pingam

# Watch the logs
docker-compose logs -f pingam
```

**What to expect in logs:**

1. **Truststore Setup** (first 30-60 seconds):
   ```
   [AM-ENTRYPOINT] Checking for PingDS truststore...
   [AM-ENTRYPOINT] SUCCESS: PingDS truststore found
   [AM-ENTRYPOINT] Copying truststore to Tomcat keystores directory...
   [AM-ENTRYPOINT] SUCCESS: Truststore copied
   [AM-ENTRYPOINT] SUCCESS: Truststore verification passed
   ```

2. **Tomcat Startup** (30-90 seconds):
   ```
   INFO: Starting Servlet engine: [Apache Tomcat/10.x.x]
   INFO: Deploying web application archive [/usr/local/tomcat/webapps/am.war]
   ```

3. **AM Deployment** (1-2 minutes):
   ```
   INFO: Deployment of web application archive [am.war] has finished in [X] ms
   ```

### Step 6: Verify Container is Running

```bash
# Check container status
docker ps | grep pingam

# Should show: "healthy" status

# Check health
docker inspect pingam | grep -A 5 Health
```

---

## Configuration

### Access PingAM Configuration Wizard

Once the container is healthy, open your browser:

```
http://localhost:8081/am
```

You should see the **PingAM Configuration** page.

---

### Option 1: Custom Configuration (Recommended for Learning)

#### General Settings

| Field | Value |
|-------|-------|
| **Configuration Type** | Custom Configuration |
| **Default User Password** | `Passw0rd123` |
| **amAdmin Password** | `Passw0rd123` |
| **Agent Password** | `Passw0rd123` |

#### Server Settings

| Field | Value |
|-------|-------|
| **Server URL** | `http://pingam:8080/am` |
| **Cookie Domain** | `.example.com` |
| **Platform Locale** | `en_US` |
| **Configuration Directory** | `/opt/am-config` |

---

### 🔴 CRITICAL: Server URL and Cookie Domain Explained

#### Server URL: `http://pingam:8080/am`

**Use:** `http://pingam:8080/am` ✅
**NOT:** `http://localhost:8081/am` ❌

**Why?**
- **`pingam`** = Internal Docker network hostname (containers communicate using this)
- **`8080`** = Internal container port (not 8081, which is only the host port mapping)
- **`localhost`** would only work inside the container, breaking all redirects and callbacks
- PingAM uses this URL for:
  - OAuth 2.0 redirects
  - SAML callbacks
  - Internal references
  - Session management

**How it works:**
```
Your Browser:  http://localhost:8081/am  (external access)
       ↓
Docker Port Mapping: 8081 → 8080
       ↓
Inside Container:  http://pingam:8080/am  (internal reference)
       ↓
PingAM generates URLs using: http://pingam:8080/am
```

#### Cookie Domain: `.example.com`

**Use:** `.example.com` ✅
**NOT:** `localhost` ❌

**Why?**
- The leading `.` enables **subdomain cookie sharing** (required for SSO)
- Allows cookies to work across:
  - `pingam.example.com`
  - `app1.example.com`
  - `app2.example.com`
- **`localhost`** would only work for local browser testing, not a production-like setup
- SSO (Single Sign-On) requires cookies to be shared across multiple applications

**SSO Example:**
```
User logs in at:     pingam.example.com
Session cookie set:  .example.com
User accesses:       app1.example.com  ← Cookie available (SSO works!)
User accesses:       app2.example.com  ← Cookie available (SSO works!)
```

---

### Accessing PingAM from Your Browser

**From your browser, use:** `http://localhost:8081/am`

Docker automatically maps:
- External: `localhost:8081` → Internal: `pingam:8080`

**Optional: Add to hosts file for convenience**

If you want to use `pingam` directly from your browser:

**Windows:** Edit `C:\Windows\System32\drivers\etc\hosts`
**Linux/Mac:** Edit `/etc/hosts`

Add:
```
127.0.0.1  pingam
127.0.0.1  pingam.example.com
```

Then you can access:
- `http://localhost:8081/am` ✅
- `http://pingam:8081/am` ✅
- `http://pingam.example.com:8081/am` ✅

---

### Quick Reference Table

| Scenario | Hostname | Port | Full URL |
|----------|----------|------|----------|
| **Configuration Wizard** | `pingam` | `8080` | `http://pingam:8080/am` |
| **Browser Access** | `localhost` | `8081` | `http://localhost:8081/am` |
| **Internal (Container to Container)** | `pingam` | `8080` | `http://pingam:8080/am` |
| **PingDS Connection** | `pingds` | `1636` | `pingds:1636` |

---

### Summary

✅ **In the wizard, use internal values:**
- Server URL: `http://pingam:8080/am` (internal Docker hostname & port)
- Cookie Domain: `.example.com` (enables SSO)

✅ **In your browser, access via:**
- `http://localhost:8081/am` (external port mapping)

✅ **PingAM will generate redirects using:** `http://pingam:8080/am`

---

#### Configuration Data Store

| Field | Value |
|-------|-------|
| **Store Type** | External Directory Server |
| **SSL/TLS** | ✅ This server uses SSL/TLS |
| **Server Name** | `pingds` |
| **Port** | `1636` |
| **Root Suffix** | `ou=am-config` |
| **Login ID** | `cn=Directory Manager` |
| **Password** | `Passw0rd123` |

**Why these settings work:**
- ✅ `pingds` hostname resolves via Docker network
- ✅ Port `1636` is LDAPS (encrypted)
- ✅ Truststore is already configured in Tomcat
- ✅ Using Directory Manager for initial setup (simplicity)

**Test Connection** - Click this button to verify LDAPS connectivity!

#### User Data Store

| Field | Value |
|-------|-------|
| **Store Type** | External Directory Server |
| **SSL/TLS** | ✅ This server uses SSL/TLS |
| **Server Name** | `pingds` |
| **Port** | `1636` |
| **Root Suffix** | `ou=identities` |
| **Login ID** | `cn=Directory Manager` |
| **Password** | `Passw0rd123` |

**Note**: For production, use dedicated service accounts like `uid=am-identity-bind-account,ou=admins,ou=identities`.

#### Site Configuration

| Field | Value |
|-------|-------|
| **Site Name** | `ForgeRock-Sandbox` |
| **Load Balancer URL** | Leave empty (single server) |

#### Complete Configuration

Click **Create Configuration**

Wait 2-5 minutes for configuration to complete. Watch logs:

```bash
docker logs -f pingam
```

You should see AM writing configuration to PingDS and initializing services.

---

### Option 2: Default Configuration (Quick Start)

1. Select **Create Default Configuration**
2. Set **amAdmin Password**: `Passw0rd123`
3. Click **Create Configuration**

**Note**: Default configuration uses embedded DS (not recommended for learning PingDS integration).

---

## Verification

### Step 1: Access AM Console

After configuration completes:

1. Navigate to: `http://localhost:8081/am/console`
2. Login:
   - **Username**: `amAdmin`
   - **Password**: `Passw0rd123`

You should see the **PingAM Admin Console**.

### Step 2: Verify Data Store Connections

In AM Console:

1. Navigate: **Realms** → **Top Level Realm** → **Data Stores**
2. You should see:
   - `embedded` (if using default config)
   - Or your external DS connections (if custom config)
3. Click on the data store
4. Click **Test Connection** → Should succeed ✅

### Step 3: Verify User Store Integration

In AM Console:

1. Navigate: **Realms** → **Top Level Realm** → **Subjects**
2. Click **Search** (or search for `demo`)
3. You should see users from PingDS if using external user store

### Step 4: Test LDAPS Connectivity

From the AM container:

```bash
# Enter AM container
docker exec -it pingam bash

# Verify truststore
keytool -list -keystore /usr/local/tomcat/conf/keystores/truststore.p12 \
    -storetype PKCS12 \
    -storepass changeit

# Should show "pingds-ca" certificate

# Test LDAPS connection to PingDS
openssl s_client -connect pingds:1636 -showcerts

# Should connect successfully
```

### Step 5: Verify JVM Settings

```bash
# Check Java process
docker exec pingam ps aux | grep java

# Should show:
# -Xms1g -Xmx2g -XX:MetaspaceSize=256m -XX:MaxMetaspaceSize=256m
# -Djavax.net.ssl.trustStore=/usr/local/tomcat/conf/keystores/truststore.p12
```

---

## Troubleshooting

### Issue 1: Container Won't Start

**Symptoms:**
- Container exits immediately
- `docker ps` doesn't show pingam

**Check logs:**
```bash
docker logs pingam
```

**Common causes:**
1. WAR file missing: `ERROR: AM WAR file not found`
2. Port conflict: Port 8081 already in use
3. Memory issues: Insufficient RAM

**Solutions:**
```bash
# Check WAR file
ls -lh pingam/software/AM-8.0.2.war

# Check port
netstat -an | grep 8081

# Rebuild
docker-compose down
docker-compose build --no-cache pingam
docker-compose up -d pingam
```

---

### Issue 2: Truststore Not Found

**Symptoms:**
```
[AM-ENTRYPOINT] ERROR: Truststore not found after 180 seconds
```

**Cause**: PingDS hasn't exported the certificate yet

**Solutions:**
```bash
# Check if PingDS is healthy
docker ps | grep pingds

# Check if certificate exists
docker exec pingds ls -la /opt/certs/

# If missing, manually export
docker exec pingds /opt/scripts/export-certificates.sh

# Restart AM
docker restart pingam
```

---

### Issue 3: LDAPS Connection Failed During Configuration

**Symptoms:**
- "Test Connection" fails
- Error: `Unable to connect to directory server`

**Check from AM container:**
```bash
# Enter AM container
docker exec -it pingam bash

# Test network connectivity
ping -c 3 pingds
nc -zv pingds 1636

# Verify truststore
keytool -list -keystore /usr/local/tomcat/conf/keystores/truststore.p12 \
    -storetype PKCS12 \
    -storepass changeit

# Test LDAPS
openssl s_client -connect pingds:1636
```

**Solutions:**
1. Ensure PingDS is healthy: `docker ps | grep pingds`
2. Verify shared volume: `docker exec pingam ls -la /opt/certs/`
3. Check JVM truststore settings: `docker exec pingam cat /usr/local/tomcat/bin/setenv.sh`

---

### Issue 4: Health Check Failing

**Symptoms:**
- Container shows "unhealthy" status

**Check health endpoint:**
```bash
docker exec pingam curl -f http://localhost:8080/am/isAlive.jsp
```

**Expected response:** HTTP 200 with content

**If 404:**
- AM not yet deployed (wait 2-3 minutes)
- WAR failed to deploy (check Tomcat logs)

**Check Tomcat logs:**
```bash
docker exec pingam tail -100 /usr/local/tomcat/logs/catalina.out
```

---

### Issue 5: Configuration Page Shows "AM Already Configured"

**Cause**: `/opt/am-config` volume has leftover configuration

**Solutions:**

**Option A: Remove the volume and start fresh**
```bash
docker-compose down
docker volume rm forgerock_pingam_config
docker-compose up -d pingam
```

**Option B: Access the existing installation**
```bash
# Just go to console
http://localhost:8081/am/console
```

---

## Advanced Configuration

### Securing the Installation

1. **Change Default Passwords**
   ```bash
   # Login to AM Console
   # Navigate to: Realms → Top Level Realm → Subjects
   # Click: amAdmin → Change password
   ```

2. **Create Service Accounts for PingDS**
   - Use dedicated bind accounts instead of `cn=Directory Manager`
   - Refer to PingDS documentation for creating service accounts

3. **Enable HTTPS**
   - Configure SSL certificate for Tomcat
   - Update Server URL to `https://pingam:8443/am`

### Monitoring and Logs

```bash
# View live logs
docker logs -f pingam

# View Tomcat catalina.out
docker exec pingam tail -f /usr/local/tomcat/logs/catalina.out

# View AM debug logs (after enabling debug)
docker exec pingam tail -f /opt/am-config/debug/*.debug

# Check disk usage
docker exec pingam df -h

# Check memory usage
docker stats pingam
```

### Backup and Restore

**Backup AM Configuration:**
```bash
# Backup volume
docker run --rm -v forgerock_pingam_config:/data -v $(pwd):/backup \
    ubuntu tar czf /backup/pingam-config-backup.tar.gz /data

# Backup database (if using file-based config)
docker exec pingam tar czf /tmp/am-config.tar.gz /opt/am-config
docker cp pingam:/tmp/am-config.tar.gz ./pingam-config-backup.tar.gz
```

**Restore:**
```bash
# Restore volume
docker run --rm -v forgerock_pingam_config:/data -v $(pwd):/backup \
    ubuntu tar xzf /backup/pingam-config-backup.tar.gz -C /
```

---

## Quick Command Reference

```bash
# Start services
docker-compose up -d

# View AM logs
docker logs -f pingam
docker logs --tail 100 pingam

# Restart AM
docker restart pingam

# Enter AM container
docker exec -it pingam bash

# Check AM version
docker exec pingam curl -s http://localhost:8080/am/json/serverinfo/version

# Stop services
docker-compose down

# Remove everything (including volumes)
docker-compose down -v
```

---

## Next Steps

### 1. Configure Authentication

- [ ] Create authentication trees/chains
- [ ] Configure multi-factor authentication (MFA)
- [ ] Set up social login providers
- [ ] Configure password policies

### 2. Set Up OAuth 2.0 / OpenID Connect

- [ ] Register OAuth 2.0 clients
- [ ] Configure scopes and claims
- [ ] Test authorization code flow
- [ ] Implement token introspection

### 3. Configure Policies

- [ ] Create policy sets
- [ ] Define resource types
- [ ] Set up authorization policies
- [ ] Test policy evaluation

### 4. Integrate Applications

- [ ] Configure SAML 2.0 service providers
- [ ] Set up agent-based protection
- [ ] Implement SSO for web applications
- [ ] Configure API gateway integration

### 5. Production Readiness

- [ ] Enable HTTPS everywhere
- [ ] Harden security settings
- [ ] Set up monitoring and alerting
- [ ] Configure log aggregation
- [ ] Plan backup and disaster recovery

---

## Summary

You now have:

✅ **PingAM 8.0.2** running on Tomcat 10 with JDK 21
✅ **Automated truststore setup** for LDAPS connectivity
✅ **Secure configuration** with non-root tomcat user
✅ **Production-ready JVM tuning** (1GB-2GB heap, G1GC)
✅ **Health monitoring** with automatic restarts
✅ **Integrated with PingDS** for configuration and user stores
✅ **Ready for application integration** via OAuth 2.0, SAML, etc.

**Access Points:**
- **Configuration/Login**: `http://localhost:8081/am`
- **Admin Console**: `http://localhost:8081/am/console`
- **REST API**: `http://localhost:8081/am/json`

**Congratulations!** Your ForgeRock/Ping Identity environment is ready! 🎉

---

## Additional Resources

- [Ping Identity Documentation](https://docs.pingidentity.com/)
- [PingAM 8.0 Admin Guide](https://docs.pingidentity.com/r/en-us/am-docs/am-docs)
- [OAuth 2.0 / OpenID Connect](https://docs.pingidentity.com/r/en-us/am-docs/am-oauth2-guide)
- [SAML 2.0](https://docs.pingidentity.com/r/en-us/am-docs/am-saml2-guide)
- [ForgeRock Community](https://community.forgerock.com/)

  ---
  What's Ready Now

  File Structure:

  sndbx1/
  ├── docker-compose.yaml          ✅ Updated with PingAM service
  ├── pingds/
  │   ├── Dockerfile               ✅ Existing
  │   └── scripts/                 ✅ Existing
  ├── pingam/
  │   ├── Dockerfile               ✅ Created (Tomcat 10 + JDK 21)
  │   ├── scripts/
  │   │   └── docker-entrypoint.sh ✅ Created (truststore automation)
  │   ├── software/
  │   │   └── AM-8.0.2.war        ⚠️  YOU NEED TO ADD THIS
  │   └── notes/
  │       └── PINGAM_INSTALLATION_GUIDE.md ✅ Completely rewritten

  ---
  Next Steps

  1. Place the WAR File

  # Download AM-8.0.2.war and place it here:
  sndbx1/pingam/software/AM-8.0.2.war

  2. Build the Image

  cd sndbx1
  docker-compose build pingam

  3. Start PingAM

  docker-compose up -d pingam
  docker-compose logs -f pingam

  4. Access Configuration

  http://localhost:8081/am

  5. Follow the Guide

  - Open pingam/notes/PINGAM_INSTALLATION_GUIDE.md
  - Follow the configuration section
  - Use the exact settings provided for PingDS integration

  ---
  Key Configuration Values (From Guide)

  When configuring PingAM:
  - Server URL: http://pingam:8080/am (internal hostname)
  - PingDS Host: pingds
  - PingDS Port: 1636 (LDAPS)
  - Config Store: ou=am-config
  - User Store: ou=identities
  - All Passwords: Passw0rd123

  The truststore will be automatically configured by the entrypoint script! 🎉

  Your ForgeRock/Ping Identity lab environment is now fully documented and ready to deploy! 🚀