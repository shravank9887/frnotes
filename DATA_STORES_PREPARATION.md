# Preparing PingDS for PingAM Data Stores

## Table of Contents
1. [Overview](#overview)
2. [Data Store Requirements](#data-store-requirements)
3. [Directory Structure Setup](#directory-structure-setup)
4. [Schema Installation](#schema-installation)
5. [Service Accounts Creation](#service-accounts-creation)
6. [Indexes Configuration](#indexes-configuration)
7. [SSL/TLS Certificate Setup](#ssltls-certificate-setup)
8. [Verification Steps](#verification-steps)

---

## Overview

PingAM requires properly configured LDAP data stores. We'll configure our existing **PingDS instance** to provide:

1. **Configuration Store** - AM settings and configuration
2. **CTS Store** - Session tokens and transient data
3. **User Store** - User identities (already exists!)
4. **Policy/Application Store** - Authorization policies

### Our Strategy

We'll use **PingDS setup profiles** which automatically create:
- Required backends
- Proper schemas
- Service accounts
- Necessary indexes

This is the **recommended approach** vs. manual setup.

---

## Data Store Requirements

### 1. Configuration Store Requirements

| Requirement | Value |
|-------------|-------|
| **Backend ID** | `am-config` |
| **Base DN** | `ou=am-config` |
| **Service Account** | `uid=am-config,ou=admins,ou=am-config` |
| **Connection** | LDAPS (port 1636) |
| **Privileges** | Read/Write |

**Purpose**: Stores AM configuration, realms, services, policies

---

### 2. CTS Store Requirements

| Requirement | Value |
|-------------|-------|
| **Backend ID** | Can share with `am-config` |
| **Base DN** | `ou=tokens` |
| **CTS Base DN** | `ou=famrecords,ou=openam-session,ou=tokens` |
| **Service Account** | `uid=openam_cts,ou=admins,ou=famrecords,ou=openam-session,ou=tokens` |
| **Connection** | LDAPS (port 1636) |
| **Privileges** | Read/Write |
| **Performance** | High - frequent add/delete operations |

**Purpose**: Stores SSO sessions, OAuth tokens, SAML assertions

---

### 3. User Store Requirements (Already Exists!)

| Requirement | Value |
|-------------|-------|
| **Backend ID** | `userRoot` |
| **Base DN** | `ou=identities` |
| **Service Account** | `uid=am-identity-bind-account,ou=admins,ou=identities` |
| **Connection** | LDAPS (port 1636) |
| **Privileges** | Read (minimum), Read/Write (for self-service) |

**Purpose**: Stores user accounts and credentials

**Note**: We already have this! Just need to add schema extensions and service account.

---

## Directory Structure Setup

### Option A: Using PingDS Setup Profiles (Recommended)

PingDS provides built-in setup profiles that automatically configure everything for AM.

#### Profile 1: AM Config Store Profile

Creates configuration and CTS stores automatically.

```bash
# This would be run during DS initial setup
/opt/opendj/setup \
  --serverId am-config-server \
  --deploymentId forgerock \
  --deploymentIdPassword Passw0rd123 \
  --rootUserDN "cn=Directory Manager" \
  --rootUserPassword Passw0rd123 \
  --monitorUserPassword Passw0rd123 \
  --hostname pingds \
  --ldapPort 1389 \
  --ldapsPort 1636 \
  --httpsPort 8443 \
  --adminConnectorPort 4444 \
  --profile am-config \
  --set am-config/amConfigAdminPassword:Passw0rd123 \
  --acceptLicense
```

**What this creates:**
- Backend: `am-config`
- Base DN: `ou=am-config`
- Service account: `uid=am-config,ou=admins,ou=am-config`
- CTS structure: `ou=tokens`
- All required schemas
- All required indexes

---

### Option B: Manual Setup (For Our Existing Instance)

Since we already have a running PingDS instance, we'll **manually add** the AM configuration store.

**Steps:**
1. Create new backend for `am-config`
2. Import base structure LDIF
3. Add schema extensions
4. Create service accounts
5. Add indexes

---

## Manual Setup Steps

### Step 1: Create AM Config Backend

```bash
# Create backend for am-config
docker exec pingds /opt/opendj/bin/dsconfig create-backend \
  --hostname pingds \
  --port 4444 \
  --bindDN "cn=Directory Manager" \
  --bindPassword "Passw0rd123" \
  --backend-name amConfig \
  --type je \
  --set enabled:true \
  --set base-dn:ou=am-config \
  --trustAll \
  --no-prompt

# Create backend for tokens (CTS)
docker exec pingds /opt/opendj/bin/dsconfig create-backend \
  --hostname pingds \
  --port 4444 \
  --bindDN "cn=Directory Manager" \
  --bindPassword "Passw0rd123" \
  --backend-name amTokens \
  --type je \
  --set enabled:true \
  --set base-dn:ou=tokens \
  --trustAll \
  --no-prompt
```

---

### Step 2: Create Base Structure

Create LDIF file: `pingds/am-base-structure.ldif`

```ldif
# Base structure for AM Configuration Store
dn: ou=am-config
objectClass: top
objectClass: organizationalUnit
ou: am-config

# Admin container for config store service account
dn: ou=admins,ou=am-config
objectClass: top
objectClass: organizationalUnit
ou: admins

# Services container
dn: ou=services,ou=am-config
objectClass: top
objectClass: organizationalUnit
ou: services

# Base structure for CTS (Token Store)
dn: ou=tokens
objectClass: top
objectClass: organizationalUnit
ou: tokens

# OpenAM session container
dn: ou=openam-session,ou=tokens
objectClass: top
objectClass: organizationalUnit
ou: openam-session

# FAM records container (CTS data)
dn: ou=famrecords,ou=openam-session,ou=tokens
objectClass: top
objectClass: organizationalUnit
ou: famrecords

# Admin container for CTS service account
dn: ou=admins,ou=famrecords,ou=openam-session,ou=tokens
objectClass: top
objectClass: organizationalUnit
ou: admins

# Admin container in identities for user store service account
dn: ou=admins,ou=identities
objectClass: top
objectClass: organizationalUnit
ou: admins
```

Import:
```bash
docker cp pingds/am-base-structure.ldif pingds:/tmp/
docker exec pingds /opt/opendj/bin/ldapmodify \
  --hostname pingds \
  --port 1636 \
  --useSSL \
  --trustAll \
  --bindDN "cn=Directory Manager" \
  --bindPassword "Passw0rd123" \
  --filename /tmp/am-base-structure.ldif
```

---

### Step 3: Install AM Schema Extensions

After deploying PingAM WAR file, schema LDIF files will be available at:
`/path/to/tomcat/webapps/am/WEB-INF/template/ldif/opendj/`

**Required Schema Files:**

1. **opendj_config_schema.ldif** - Configuration store schema
2. **opendj_user_schema.ldif** - User store schema extensions
3. **opendj_user_index.ldif** - User store indexes

**Installation Process:**

```bash
# Copy schema files from AM to PingDS
docker cp pingam:/usr/local/tomcat/webapps/am/WEB-INF/template/ldif/opendj/opendj_config_schema.ldif pingds:/tmp/
docker cp pingam:/usr/local/tomcat/webapps/am/WEB-INF/template/ldif/opendj/opendj_user_schema.ldif pingds:/tmp/
docker cp pingam:/usr/local/tomcat/webapps/am/WEB-INF/template/ldif/opendj/opendj_user_index.ldif pingds:/tmp/

# Import config schema
docker exec pingds /opt/opendj/bin/ldapmodify \
  --hostname pingds \
  --port 1636 \
  --useSSL \
  --trustAll \
  --bindDN "cn=Directory Manager" \
  --bindPassword "Passw0rd123" \
  --filename /tmp/opendj_config_schema.ldif

# Import user schema extensions
docker exec pingds /opt/opendj/bin/ldapmodify \
  --hostname pingds \
  --port 1636 \
  --useSSL \
  --trustAll \
  --bindDN "cn=Directory Manager" \
  --bindPassword "Passw0rd123" \
  --filename /tmp/opendj_user_schema.ldif
```

---

### Step 4: Create Service Accounts

Create LDIF file: `pingds/am-service-accounts.ldif`

```ldif
# Service account for AM Configuration Store
dn: uid=am-config,ou=admins,ou=am-config
objectClass: top
objectClass: person
objectClass: organizationalPerson
objectClass: inetOrgPerson
uid: am-config
cn: AM Config Service Account
sn: Service Account
userPassword: AMConfig@2024
description: Service account for PingAM configuration store

# Service account for CTS Store
dn: uid=openam_cts,ou=admins,ou=famrecords,ou=openam-session,ou=tokens
objectClass: top
objectClass: person
objectClass: organizationalPerson
objectClass: inetOrgPerson
uid: openam_cts
cn: OpenAM CTS Service Account
sn: Service Account
userPassword: AMCTS@2024
description: Service account for PingAM Core Token Service

# Service account for User Store
dn: uid=am-identity-bind-account,ou=admins,ou=identities
objectClass: top
objectClass: person
objectClass: organizationalPerson
objectClass: inetOrgPerson
uid: am-identity-bind-account
cn: AM Identity Bind Account
sn: Service Account
userPassword: AMIdentity@2024
description: Service account for PingAM identity repository
```

Import:
```bash
docker cp pingds/am-service-accounts.ldif pingds:/tmp/
docker exec pingds /opt/opendj/bin/ldapmodify \
  --hostname pingds \
  --port 1636 \
  --useSSL \
  --trustAll \
  --bindDN "cn=Directory Manager" \
  --bindPassword "Passw0rd123" \
  --filename /tmp/am-service-accounts.ldif
```

---

### Step 5: Grant Permissions to Service Accounts

Service accounts need appropriate privileges to access their respective stores.

**For Config Store Account:**
```bash
docker exec pingds /opt/opendj/bin/dsconfig set-access-control-handler-prop \
  --hostname pingds \
  --port 4444 \
  --bindDN "cn=Directory Manager" \
  --bindPassword "Passw0rd123" \
  --add global-aci:"(target=\"ldap:///ou=am-config\")(targetattr=\"*\")(version 3.0; acl \"AM Config Access\"; allow (all) userdn=\"ldap:///uid=am-config,ou=admins,ou=am-config\";)" \
  --trustAll \
  --no-prompt
```

**For CTS Account:**
```bash
docker exec pingds /opt/opendj/bin/dsconfig set-access-control-handler-prop \
  --hostname pingds \
  --port 4444 \
  --bindDN "cn=Directory Manager" \
  --bindPassword "Passw0rd123" \
  --add global-aci:"(target=\"ldap:///ou=tokens\")(targetattr=\"*\")(version 3.0; acl \"CTS Access\"; allow (all) userdn=\"ldap:///uid=openam_cts,ou=admins,ou=famrecords,ou=openam-session,ou=tokens\";)" \
  --trustAll \
  --no-prompt
```

**For User Store Account (Read-Only):**
```bash
docker exec pingds /opt/opendj/bin/dsconfig set-access-control-handler-prop \
  --hostname pingds \
  --port 4444 \
  --bindDN "cn=Directory Manager" \
  --bindPassword "Passw0rd123" \
  --add global-aci:"(target=\"ldap:///ou=identities\")(targetattr=\"*\")(version 3.0; acl \"AM Identity Read Access\"; allow (read,search,compare) userdn=\"ldap:///uid=am-identity-bind-account,ou=admins,ou=identities\";)" \
  --trustAll \
  --no-prompt
```

---

### Step 6: Create Indexes for Performance

CTS requires specific indexes for optimal performance.

```bash
# Index for CTS: coreTokenExpirationDate
docker exec pingds /opt/opendj/bin/dsconfig create-local-db-index \
  --hostname pingds \
  --port 4444 \
  --bindDN "cn=Directory Manager" \
  --bindPassword "Passw0rd123" \
  --backend-name amTokens \
  --index-name coreTokenExpirationDate \
  --set index-type:ordering \
  --trustAll \
  --no-prompt

# Rebuild index
docker exec pingds /opt/opendj/bin/rebuild-index \
  --hostname pingds \
  --port 4444 \
  --bindDN "cn=Directory Manager" \
  --bindPassword "Passw0rd123" \
  --baseDN ou=tokens \
  --index coreTokenExpirationDate \
  --trustAll
```

---

## SSL/TLS Certificate Setup

PingAM requires secure LDAPS connections to all data stores.

### Export DS Certificate

```bash
# Export CA certificate from PingDS
docker exec pingds /opt/opendj/bin/dskeymgr export-ca-cert \
  --deploymentId forgerock-eval \
  --deploymentIdPassword Passw0rd123 \
  --outputFile /tmp/ds-ca-cert.pem

# Copy to host
docker cp pingds:/tmp/ds-ca-cert.pem ./shared-certs/
```

### Import into PingAM Truststore

This step will be done after PingAM container is running:

```bash
# Import DS certificate into AM's truststore
docker exec pingam keytool -importcert \
  -file /opt/certs/ds-ca-cert.pem \
  -alias pingds-ca \
  -keystore /usr/local/tomcat/conf/truststore.jks \
  -storepass changeit \
  -noprompt
```

---

## Verification Steps

### Verify Base Structure

```bash
# Check am-config base
docker exec pingds /opt/opendj/bin/ldapsearch \
  --hostname pingds \
  --port 1636 \
  --useSSL \
  --trustAll \
  --bindDN "cn=Directory Manager" \
  --bindPassword "Passw0rd123" \
  --baseDN "ou=am-config" \
  --searchScope base \
  "(objectClass=*)" \
  dn

# Check tokens base
docker exec pingds /opt/opendj/bin/ldapsearch \
  --hostname pingds \
  --port 1636 \
  --useSSL \
  --trustAll \
  --bindDN "cn=Directory Manager" \
  --bindPassword "Passw0rd123" \
  --baseDN "ou=tokens" \
  --searchScope base \
  "(objectClass=*)" \
  dn
```

---

### Verify Service Accounts

```bash
# Test config store account
docker exec pingds /opt/opendj/bin/ldapsearch \
  --hostname pingds \
  --port 1636 \
  --useSSL \
  --trustAll \
  --bindDN "uid=am-config,ou=admins,ou=am-config" \
  --bindPassword "AMConfig@2024" \
  --baseDN "ou=am-config" \
  --searchScope base \
  "(objectClass=*)"

# Test CTS account
docker exec pingds /opt/opendj/bin/ldapsearch \
  --hostname pingds \
  --port 1636 \
  --useSSL \
  --trustAll \
  --bindDN "uid=openam_cts,ou=admins,ou=famrecords,ou=openam-session,ou=tokens" \
  --bindPassword "AMCTS@2024" \
  --baseDN "ou=tokens" \
  --searchScope base \
  "(objectClass=*)"

# Test identity bind account
docker exec pingds /opt/opendj/bin/ldapsearch \
  --hostname pingds \
  --port 1636 \
  --useSSL \
  --trustAll \
  --bindDN "uid=am-identity-bind-account,ou=admins,ou=identities" \
  --bindPassword "AMIdentity@2024" \
  --baseDN "ou=identities" \
  --searchScope sub \
  "(uid=jdoe)" \
  dn cn mail
```

---

## Quick Reference

### Service Account Credentials

| Store | Bind DN | Password |
|-------|---------|----------|
| **Config Store** | `uid=am-config,ou=admins,ou=am-config` | `AMConfig@2024` |
| **CTS Store** | `uid=openam_cts,ou=admins,ou=famrecords,ou=openam-session,ou=tokens` | `AMCTS@2024` |
| **User Store** | `uid=am-identity-bind-account,ou=admins,ou=identities` | `AMIdentity@2024` |

### Connection Parameters

| Parameter | Value |
|-----------|-------|
| **Hostname** | `pingds` |
| **LDAPS Port** | `1636` |
| **Connection Type** | SSL/TLS (LDAPS) |
| **Trust All Certs** | Yes (for development) |

### Base DNs

| Store | Base DN |
|-------|---------|
| **Config Store** | `ou=am-config` |
| **CTS Store** | `ou=famrecords,ou=openam-session,ou=tokens` |
| **User Store** | `ou=identities` |

---

## Troubleshooting

### Issue: Cannot create backend

**Error:** Backend already exists

**Solution:**
```bash
# Check existing backends
docker exec pingds /opt/opendj/bin/dsconfig list-backends \
  --hostname pingds \
  --port 4444 \
  --bindDN "cn=Directory Manager" \
  --bindPassword "Passw0rd123" \
  --trustAll
```

---

### Issue: Service account authentication fails

**Error:** Invalid credentials

**Solution:**
```bash
# Reset password
docker exec pingds /opt/opendj/bin/ldappasswordmodify \
  --hostname pingds \
  --port 1636 \
  --useSSL \
  --trustAll \
  --bindDN "cn=Directory Manager" \
  --bindPassword "Passw0rd123" \
  --authzID "uid=am-config,ou=admins,ou=am-config" \
  --newPassword "AMConfig@2024"
```

---

### Issue: Schema import fails

**Error:** Attribute already exists

**Solution:** Schema may already be present. Verify:
```bash
docker exec pingds /opt/opendj/bin/ldapsearch \
  --hostname pingds \
  --port 1636 \
  --useSSL \
  --trustAll \
  --bindDN "cn=Directory Manager" \
  --bindPassword "Passw0rd123" \
  --baseDN "cn=schema" \
  --searchScope base \
  "(objectClass=*)" \
  attributeTypes | grep -i "coretoken"
```

---

## Next Steps

1. ✅ Data stores prepared
2. → Read `PINGAM_INSTALLATION_GUIDE.md` for AM container setup
3. → Configure AM to use these data stores
4. → Test authentication with existing users

---

**Important Notes:**
- Always use LDAPS (port 1636) for production
- Keep service account passwords secure
- CTS store requires performance tuning for production
- Back up PingDS before making schema changes
