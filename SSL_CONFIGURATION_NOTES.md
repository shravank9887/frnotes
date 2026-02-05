# PingAM SSL/TLS Configuration Notes

## Overview

This document explains the SSL/TLS configuration between PingAM and PingDS in the Docker lab environment.

---

## The Problem We Solved

### Issue: LDAPS Connection Timeout

When configuring PingAM to connect to PingDS via LDAPS (port 1636), the connection times out with:

```
Cannot connect to Directory Server, the error was: Client-Side Timeout
```

### Root Cause

1. **PingDS generates self-signed certificates** during setup using deployment keys
2. These certificates have a generic subject: `CN=Deployment key, O=ForgeRock.com`
3. They **don't include the hostname** (`pingds`) in the certificate
4. Java's LDAP client performs **strict hostname verification** by default
5. The hostname verification fails → SSL handshake times out

---

## The Solution

### Disable Hostname Verification for LDAP Connections

We added this JVM parameter to PingAM:

```bash
-Dcom.sun.jndi.ldap.object.disableEndpointIdentification=true
```

**Where it's configured:**
- File: `pingam/scripts/docker-entrypoint.sh`
- Function: `configure_tomcat_truststore()`
- Lines: 105-108

**When it's applied:**
- Automatically during container startup
- Appended to `$CATALINA_BASE/bin/setenv.sh`
- Takes effect when Tomcat starts

---

## Is This Safe?

### ✅ YES - For Lab/Development Environments

**Why it's acceptable:**
1. **Isolated Network**: Both containers run on the same Docker network (`fr-net`)
2. **No External Access**: No external network can reach the PingAM ↔ PingDS connection
3. **Man-in-the-Middle Protection**: Docker's bridge network provides isolation
4. **Learning Environment**: This is a sandbox for learning, not production
5. **Internal Communication**: Traffic never leaves the Docker host

**Diagram:**
```
Docker Host Machine
├── fr-net (isolated network)
│   ├── PingDS (172.28.0.2:1636)
│   └── PingAM (172.28.0.3)
│       ↕ LDAPS (encrypted, but hostname verification disabled)
│   No external access possible
```

### ❌ NO - For Production Environments

In production, you should:
1. Generate proper SSL certificates with correct hostnames
2. Include Subject Alternative Names (SANs) in certificates
3. Use a proper Certificate Authority (CA)
4. Keep hostname verification enabled
5. Use certificate pinning if needed

---

## Alternative Solutions

### Option 1: Use Non-SSL (LDAP on port 1389)

**Not possible** in our setup because:
- PingDS requires "Confidentiality Required" for all connections
- Even port 1389 rejects simple binds without SSL
- This is a security policy enforced by the AM profiles during PingDS setup

### Option 2: Regenerate PingDS Certificates with Proper SANs

**Too complex** for a lab environment:
1. Would require recreating the entire PingDS instance
2. Need to customize deployment key generation
3. Requires understanding PingDS key management
4. Overkill for learning purposes

### Option 3: Import Custom Certificates

**Also complex**:
1. Generate certificates with `openssl`
2. Import into PingDS keystore
3. Configure PingDS to use custom certificates
4. Not worth the effort for a lab

---

## What Gets Configured

### Complete SSL/TLS Configuration (from docker-entrypoint.sh)

```bash
# PingDS Truststore Configuration
export CATALINA_OPTS="$CATALINA_OPTS -Djavax.net.ssl.trustStore=/usr/local/tomcat/conf/keystores/truststore.p12"
export CATALINA_OPTS="$CATALINA_OPTS -Djavax.net.ssl.trustStorePassword=changeit"
export CATALINA_OPTS="$CATALINA_OPTS -Djavax.net.ssl.trustStoreType=PKCS12"

# Disable hostname verification for LDAPS connections in Docker network
# NOTE: This is acceptable for lab/dev environments with isolated networks.
# For production, use properly configured certificates with correct SANs.
export CATALINA_OPTS="$CATALINA_OPTS -Dcom.sun.jndi.ldap.object.disableEndpointIdentification=true"
```

### What Each Parameter Does

| Parameter | Purpose |
|-----------|---------|
| `javax.net.ssl.trustStore` | Points to the PKCS12 truststore containing PingDS CA cert |
| `javax.net.ssl.trustStorePassword` | Password for the truststore (`changeit`) |
| `javax.net.ssl.trustStoreType` | Specifies PKCS12 format (not JKS) |
| `com.sun.jndi.ldap.object.disableEndpointIdentification` | **Disables hostname verification** for LDAP |

---

## Verification

### Check if the Configuration is Applied

```bash
# View the setenv.sh file
docker exec pingam cat /usr/local/tomcat/bin/setenv.sh

# Check JVM parameters
docker exec pingam ps aux | grep java | grep -o 'disableEndpointIdentification[^ ]*'

# Verify LDAPS connection works
docker exec pingam openssl s_client -connect pingds:1636 < /dev/null
```

### Test LDAPS Connection from PingAM

```bash
# This should succeed with the configuration
docker exec pingds /opt/opendj/bin/ldapsearch \
  --hostname pingds \
  --port 1636 \
  --useSSL \
  --trustAll \
  --bindDN "cn=Directory Manager" \
  --bindPassword Passw0rd123 \
  --baseDN "ou=identities" \
  "(uid=*)" dn
```

---

## Troubleshooting

### If LDAPS Still Times Out

1. **Check if truststore exists:**
   ```bash
   docker exec pingam ls -la /usr/local/tomcat/conf/keystores/truststore.p12
   ```

2. **Verify certificate is in truststore:**
   ```bash
   docker exec pingam keytool -list -keystore /usr/local/tomcat/conf/keystores/truststore.p12 -storepass changeit -storetype PKCS12
   ```

3. **Check if parameter is set:**
   ```bash
   docker exec pingam cat /usr/local/tomcat/bin/setenv.sh | grep disableEndpointIdentification
   ```

4. **Restart PingAM:**
   ```bash
   docker restart pingam
   ```

### If You Get Certificate Errors

The certificate might not be properly imported. Re-run:

```bash
# Import into Java's default truststore
docker exec -u root pingam keytool -import -noprompt -trustcacerts \
  -alias pingds-ca \
  -file /opt/certs/ds-ca-cert.pem \
  -keystore $JAVA_HOME/lib/security/cacerts \
  -storepass changeit

# Restart PingAM
docker restart pingam
```

---

## Production Recommendations

### For Production Environments

**DO NOT disable hostname verification in production!** Instead:

1. **Use a proper CA:**
   - Get certificates from a trusted Certificate Authority
   - Or set up an internal PKI with your organization's CA

2. **Generate certificates with proper SANs:**
   ```
   Subject: CN=pingds.example.com
   Subject Alternative Names:
     DNS: pingds.example.com
     DNS: pingds
     DNS: ds.example.com
   ```

3. **Configure PingDS with custom certificates:**
   ```bash
   dsconfig set-key-manager-provider-prop \
     --provider-name "PKCS12" \
     --set enabled:true \
     --set key-store-file:/path/to/keystore.p12
   ```

4. **Keep hostname verification enabled:**
   - Remove the `disableEndpointIdentification` parameter
   - Trust the CA certificate in Java's truststore
   - Ensure DNS resolution works correctly

---

## Summary

### What We Did
✅ Disabled hostname verification for LDAPS in the lab environment
✅ Updated docker-entrypoint.sh to configure this automatically
✅ Documented why this is acceptable for labs but not production
✅ Provided production alternatives

### Configuration Files Modified
- `pingam/scripts/docker-entrypoint.sh` (lines 105-108)
- `pingam/notes/SERVER_URL_EXPLAINED.md` (added technical note)
- `pingam/notes/SSL_CONFIGURATION_NOTES.md` (this file)

### When to Rebuild
If you rebuild the PingAM image from scratch, the SSL configuration will be automatically applied by the docker-entrypoint.sh script.

**No manual configuration needed!** 🎉

---

**Last Updated:** December 8, 2025
**Environment:** Docker Lab/Sandbox
**Security Context:** Isolated internal network
