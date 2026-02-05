# PingAM Post-Installation Verification & Configuration Guide

**Last Updated:** December 8, 2025
**Environment:** Docker Lab/Sandbox
**PingAM Version:** 8.0.2
**PingDS Version:** 8.0

---

## Table of Contents

1. [Verification Steps](#verification-steps)
2. [Connecting CTS Store](#connecting-cts-store)
3. [Connecting User Store](#connecting-user-store)
4. [Base Configuration for Advanced Use Cases](#base-configuration-for-advanced-use-cases)
5. [SSO with SAML Configuration](#sso-with-saml-configuration)
6. [OAuth 2.0 Use Cases](#oauth-20-use-cases)
7. [Custom Authentication Trees](#custom-authentication-trees)
8. [Troubleshooting](#troubleshooting)

---

## Verification Steps

### 1. Verify PingDS Installation

**Check PingDS is running:**
```bash
docker exec pingds //opt/opendj/bin/status \
  --hostname pingds \
  --port 4444 \
  --bindDN "cn=Directory Manager" \
  --bindPassword Passw0rd123 \
  --trustAll
```

**Expected output:**
```
>>>> Connection handlers
Name  : Port
------:------
LDAP  : 1389
LDAPS : 1636
HTTP  : 8080
HTTPS : 8443
Administration Connector : 4444

>>>> Backends
Name             : Base DN
-----------------:-----------------
amCts            : ou=tokens
amIdentityStore  : ou=identities
cfgStore         : ou=am-config
```

### 2. Verify Base DNs and Entries

**Check AM Config Store:**
```bash
docker exec pingds //opt/opendj/bin/ldapsearch \
  --hostname pingds \
  --port 1636 \
  --useSSL \
  --trustAll \
  --bindDN "cn=Directory Manager" \
  --bindPassword Passw0rd123 \
  --baseDN "ou=am-config" \
  "(objectClass=*)" dn
```

**Check Identity Store:**
```bash
docker exec pingds //opt/opendj/bin/ldapsearch \
  --hostname pingds \
  --port 1636 \
  --useSSL \
  --trustAll \
  --bindDN "cn=Directory Manager" \
  --bindPassword Passw0rd123 \
  --baseDN "ou=identities" \
  "(objectClass=*)" dn
```

**Expected entries:**
```
dn: ou=identities
dn: ou=people,ou=identities
dn: ou=groups,ou=identities
dn: ou=admins,ou=identities
dn: uid=am-identity-bind-account,ou=admins,ou=identities
```

**Check CTS Store:**
```bash
docker exec pingds //opt/opendj/bin/ldapsearch \
  --hostname pingds \
  --port 1636 \
  --useSSL \
  --trustAll \
  --bindDN "cn=Directory Manager" \
  --bindPassword Passw0rd123 \
  --baseDN "ou=tokens" \
  "(objectClass=*)" dn
```

**Expected entries:**
```
dn: ou=tokens
dn: ou=famrecords,ou=openam-session,ou=tokens
dn: ou=openam-session,ou=tokens
dn: ou=admins,ou=famrecords,ou=openam-session,ou=tokens
dn: uid=openam_cts,ou=admins,ou=famrecords,ou=openam-session,ou=tokens
```

### 3. Verify Service Accounts

**AM Config Service Account:**
```bash
docker exec pingds //opt/opendj/bin/ldapsearch \
  --hostname pingds \
  --port 1636 \
  --useSSL \
  --trustAll \
  --bindDN "cn=Directory Manager" \
  --bindPassword Passw0rd123 \
  --baseDN "ou=am-config" \
  "(uid=am-config)" \
  dn userPassword
```

**AM Identity Bind Account:**
```bash
docker exec pingds //opt/opendj/bin/ldapsearch \
  --hostname pingds \
  --port 1636 \
  --useSSL \
  --trustAll \
  --bindDN "cn=Directory Manager" \
  --bindPassword Passw0rd123 \
  --baseDN "ou=identities" \
  "(uid=am-identity-bind-account)" \
  dn userPassword
```

**CTS Service Account:**
```bash
docker exec pingds //opt/opendj/bin/ldapsearch \
  --hostname pingds \
  --port 1636 \
  --useSSL \
  --trustAll \
  --bindDN "cn=Directory Manager" \
  --bindPassword Passw0rd123 \
  --baseDN "ou=tokens" \
  "(uid=openam_cts)" \
  dn userPassword
```

### 4. Verify SSL/TLS Certificates

**Check exported certificates:**
```bash
docker exec pingds ls -la //opt/certs/
```

**Expected files:**
```
-rw-r--r-- 1 pingds pingds ds-ca-cert.pem
-rw-r--r-- 1 pingds pingds truststore.p12
```

**Verify PingAM has the truststore:**
```bash
docker exec pingam ls -la /usr/local/tomcat/conf/keystores/truststore.p12
```

**Check hostname verification is disabled:**
```bash
docker exec pingam cat //usr/local/tomcat/bin/setenv.sh | grep disableEndpointIdentification
```

**Expected output:**
```bash
export CATALINA_OPTS="$CATALINA_OPTS -Dcom.sun.jndi.ldap.object.disableEndpointIdentification=true"
```

### 5. Verify PingAM Installation

**Access PingAM Console:**
- URL: `http://localhost:8081/am/console`
- Username: `amadmin`
- Password: (the password you set during installation)

**Check AM version:**
```bash
docker exec pingam curl -s http://localhost:8080/am/json/serverinfo/* | grep -o '"version":"[^"]*"'
```

---

## Connecting CTS Store

The CTS (Core Token Service) store is used for session tokens, OAuth 2.0 tokens, and SAML assertions.

### Step 1: Navigate to CTS Configuration

1. Login to AM Console: `http://localhost:8081/am/console`
2. Go to: **Configure** → **Global Services** → **CTS**

### Step 2: Configure External CTS Store

1. Click **CTS Token Store** tab
2. Set the following:

```yaml
Store Mode: External Token Store

Connection String #1:
  Server: pingds:1636
  Type: SSL/TLS

Connection String #2:
  Server: pingds:1636
  Type: SSL/TLS

Max Connections: 10

Heartbeat: 10 (seconds)

Root Suffix: ou=tokens

Enable Affinity: Disabled
```

### Step 3: Configure CTS Bind Account

```yaml
Bind DN: uid=openam_cts,ou=admins,ou=famrecords,ou=openam-session,ou=tokens
Password: Passw0rd123
```

### Step 4: SSL/TLS Settings

```yaml
SSL/TLS Enabled: ✓ Enabled

Trust Store: (Leave default - uses JVM truststore)

Hostname Verification: Disabled (already configured in setenv.sh)
```

### Step 5: Save and Test

1. Click **Save Changes**
2. Verify connection:

```bash
# Check CTS tokens are being created
docker exec pingds //opt/opendj/bin/ldapsearch \
  --hostname pingds \
  --port 1636 \
  --useSSL \
  --trustAll \
  --bindDN "cn=Directory Manager" \
  --bindPassword Passw0rd123 \
  --baseDN "ou=tokens" \
  "(coreTokenType=*)" dn
```

3. Login/logout from AM console to generate session tokens
4. Re-run the above command - you should see session tokens

---

## Connecting User Store

The User Store is used for user authentication and identity management.

### Step 1: Navigate to Identity Store Configuration

1. Login to AM Console: `http://localhost:8081/am/console`
2. Go to: **Top Level Realm** → **Identity Stores**
3. You should see the default **embedded** store

### Step 2: Add External User Store

1. Click **Add Identity Store**
2. Select: **LDAP Directory Server**
3. Configure:

```yaml
Name: PingDS-UserStore

LDAP Server:
  Primary Server: pingds
  Primary Port: 1636

  Secondary Server: (leave empty)
  Secondary Port: (leave empty)

Connection Mode: LDAP over SSL

Bind DN: uid=am-identity-bind-account,ou=admins,ou=identities
Bind Password: Passw0rd123

Base DN: ou=identities

User Search Attribute: uid
User Search Filter: (objectClass=inetOrgPerson)
User Object Class: inetOrgPerson

Group Search Attribute: cn
Group Search Filter: (objectClass=groupOfUniqueNames)
Group Object Class: groupOfUniqueNames

Attribute Mapping:
  - Username: uid
  - Email: mail
  - Given Name: givenName
  - Surname: sn
  - Full Name: cn
```

### Step 3: SSL/TLS Configuration

```yaml
SSL/TLS: ✓ Enabled

Trust All Server Certificates: ✓ Enabled (for lab environment)
```

### Step 4: Test Connection

1. Click **Test Connection** button
2. Should show: ✅ **Connection Successful**

### Step 5: Set as Default User Store

1. Go to: **Top Level Realm** → **Authentication** → **Settings**
2. Under **User Profile** → **User Profile Dynamic Creation**:
   ```yaml
   Dynamic Attributes: ✓ Enabled
   Identity Store: PingDS-UserStore
   ```

### Step 6: Verify User Store

**Create a test user via AM Console:**
1. Go to: **Top Level Realm** → **Identities**
2. Click **Add Identity** → **User**
3. Fill in:
   ```yaml
   Username: testuser
   First Name: Test
   Last Name: User
   Email: testuser@example.com
   Password: Passw0rd123
   ```
4. Click **Save**

**Verify in PingDS:**
```bash
docker exec pingds //opt/opendj/bin/ldapsearch \
  --hostname pingds \
  --port 1636 \
  --useSSL \
  --trustAll \
  --bindDN "cn=Directory Manager" \
  --bindPassword Passw0rd123 \
  --baseDN "ou=people,ou=identities" \
  "(uid=testuser)" \
  dn cn mail
```

**Test authentication:**
```bash
curl -X POST 'http://localhost:8081/am/json/authenticate' \
  -H 'Content-Type: application/json' \
  -H 'X-OpenAM-Username: testuser' \
  -H 'X-OpenAM-Password: Passw0rd123' \
  -H 'Accept-API-Version: resource=2.0'
```

---

## Base Configuration for Advanced Use Cases

### 1. Create OAuth 2.0 Provider

**Navigate to:**
- **Top Level Realm** → **Services** → **Add a Service**
- Select: **OAuth2 Provider**

**Basic Configuration:**
```yaml
Authorization Code Lifetime (seconds): 300
Refresh Token Lifetime (seconds): 604800
Access Token Lifetime (seconds): 3600

Issue Refresh Tokens: ✓ Enabled
Issue Refresh Tokens on Refreshing Access Tokens: ✓ Enabled

Supported Scopes:
  - openid
  - profile
  - email
  - address
  - phone

Supported Claims:
  - sub
  - name
  - given_name
  - family_name
  - email
  - email_verified
  - phone_number

Token Signing Algorithm: RS256
```

**Advanced Configuration:**
```yaml
Use Policy Engine for Scope Decisions: ✓ Enabled
Allow Open Dynamic Client Registration: ✗ Disabled (for security)
```

**Save Configuration**

### 2. Create SAML 2.0 Entity Provider

**Navigate to:**
- **Top Level Realm** → **Configure SAMLv2 Provider**

**Create Hosted Identity Provider (IdP):**
1. Click **Create Hosted Identity Provider**
2. Configure:
   ```yaml
   Name: PingAM-IDP

   Signing Key: test (default self-signed)
   Encryption Key: test

   Circle of Trust: cot

   Attribute Mapping:
     - uid → uid
     - mail → email
     - cn → name
     - givenName → firstName
     - sn → lastName
   ```

3. Click **Configure**

**Metadata Location:**
- IdP Metadata: `http://localhost:8081/am/saml2/jsp/exportmetadata.jsp?entityid=PingAM-IDP&realm=/`

### 3. Enable Session Management

**Navigate to:**
- **Top Level Realm** → **Services** → **Session**

**Configuration:**
```yaml
Maximum Session Time: 120 (minutes)
Maximum Idle Time: 30 (minutes)

Session Quota: 5

Enable Session Quota Constraints: ✓ Enabled

Resulting Behavior if Session Quota Exceeded:
  - Deny Access (recommended for production)
  - Destroy Oldest Session (useful for dev)
```

### 4. Configure CORS (for API access)

**Navigate to:**
- **Top Level Realm** → **Services** → **CORS**

**Configuration:**
```yaml
Accepted Origins:
  - http://localhost:*
  - http://127.0.0.1:*

Accepted Methods:
  - GET
  - POST
  - PUT
  - DELETE
  - OPTIONS

Accepted Headers:
  - Content-Type
  - Authorization
  - X-OpenAM-Username
  - X-OpenAM-Password
  - Accept-API-Version

Allow Credentials: ✓ Enabled
```

---

## SSO with SAML Configuration

### Scenario: PingAM as SAML Identity Provider (IdP)

#### Step 1: Configure PingAM as IdP (Already done above)

#### Step 2: Create a Test Service Provider (SP)

For testing, you can use online SAML SP test tools:
- https://samltest.id/
- https://sptest.iamshowcase.com/

Or set up a local SP using SimpleSAMLphp.

#### Step 3: Exchange Metadata

**Export PingAM IdP Metadata:**
```bash
curl http://localhost:8081/am/saml2/jsp/exportmetadata.jsp?entityid=PingAM-IDP&realm=/ > pingam-idp-metadata.xml
```

**Import SP Metadata to PingAM:**
1. Get SP metadata URL from your SP application
2. In AM Console: **Top Level Realm** → **SAML** → **Import Entity**
3. Paste SP metadata or upload file
4. Select **Circle of Trust**: cot
5. Click **Import**

#### Step 4: Configure Attribute Mapping

1. Go to: **Top Level Realm** → **SAML** → **PingAM-IDP**
2. Click **Assertion Content** tab
3. Add attribute mappings:
   ```
   LDAP Attribute → SAML Attribute
   --------------------------------
   uid           → urn:oasis:names:tc:SAML:1.1:nameid-format:unspecified
   mail          → email
   cn            → name
   givenName     → firstName
   sn            → lastName
   ```

#### Step 5: Test SAML SSO Flow

1. Access SP application
2. Click "Login with SAML"
3. Should redirect to PingAM login page: `http://localhost:8081/am/XUI`
4. Login with: `testuser` / `Passw0rd123`
5. Should redirect back to SP with SAML assertion
6. SP validates assertion and creates session

**Check SAML tokens in CTS:**
```bash
docker exec pingds //opt/opendj/bin/ldapsearch \
  --hostname pingds \
  --port 1636 \
  --useSSL \
  --trustAll \
  --bindDN "cn=Directory Manager" \
  --bindPassword Passw0rd123 \
  --baseDN "ou=tokens" \
  "(coreTokenType=SAML2)" dn
```

---

## OAuth 2.0 Use Cases

### Use Case 1: Authorization Code Flow

#### Step 1: Register OAuth Client

1. Go to: **Top Level Realm** → **Applications** → **OAuth 2.0**
2. Click **Add Client**
3. Configure:
   ```yaml
   Client ID: my-web-app
   Client Secret: MySecretPassword123

   Redirection URIs:
     - http://localhost:3000/callback
     - http://localhost:3000/oauth/callback

   Scope(s):
     - openid
     - profile
     - email

   Grant Types:
     - Authorization Code
     - Refresh Token

   Response Types:
     - code
     - token
     - id_token

   Token Endpoint Authentication Method: client_secret_post
   ```

4. Click **Create**

#### Step 2: Test Authorization Code Flow

**1. Get authorization code:**
```bash
# Open in browser:
http://localhost:8081/am/oauth2/authorize?client_id=my-web-app&response_type=code&scope=openid%20profile%20email&redirect_uri=http://localhost:3000/callback&state=randomstate123
```

**2. Login and approve** → You'll be redirected with code:
```
http://localhost:3000/callback?code=AUTHORIZATION_CODE&state=randomstate123
```

**3. Exchange code for tokens:**
```bash
curl -X POST http://localhost:8081/am/oauth2/access_token \
  -d "grant_type=authorization_code" \
  -d "code=AUTHORIZATION_CODE" \
  -d "client_id=my-web-app" \
  -d "client_secret=MySecretPassword123" \
  -d "redirect_uri=http://localhost:3000/callback"
```

**Response:**
```json
{
  "access_token": "eyJ0eXAiOiJKV1...",
  "refresh_token": "eyJ0eXAiOiJKV1...",
  "scope": "openid profile email",
  "id_token": "eyJhbGciOiJSUzI1...",
  "token_type": "Bearer",
  "expires_in": 3599
}
```

**4. Get user info:**
```bash
curl -H "Authorization: Bearer ACCESS_TOKEN" \
  http://localhost:8081/am/oauth2/userinfo
```

### Use Case 2: Client Credentials Flow

**Register machine-to-machine client:**
```yaml
Client ID: api-service
Client Secret: ApiSecret123
Grant Types: Client Credentials
Scope(s): api:read, api:write
```

**Get access token:**
```bash
curl -X POST http://localhost:8081/am/oauth2/access_token \
  -u api-service:ApiSecret123 \
  -d "grant_type=client_credentials" \
  -d "scope=api:read api:write"
```

### Use Case 3: Refresh Token Flow

```bash
curl -X POST http://localhost:8081/am/oauth2/access_token \
  -d "grant_type=refresh_token" \
  -d "refresh_token=REFRESH_TOKEN" \
  -d "client_id=my-web-app" \
  -d "client_secret=MySecretPassword123"
```

**Verify OAuth tokens in CTS:**
```bash
docker exec pingds //opt/opendj/bin/ldapsearch \
  --hostname pingds \
  --port 1636 \
  --useSSL \
  --trustAll \
  --bindDN "cn=Directory Manager" \
  --bindPassword Passw0rd123 \
  --baseDN "ou=tokens" \
  "(coreTokenType=OAUTH*)" dn coreTokenType
```

---

## Custom Authentication Trees

### Prerequisites

1. **Install Scripting Engine:**
   - Go to: **Top Level Realm** → **Scripts**
   - Create test scripts for custom nodes

2. **Understand Default Trees:**
   - Go to: **Top Level Realm** → **Authentication** → **Trees**
   - Examine **Login** tree (default)

### Use Case 1: Multi-Factor Authentication Tree

#### Step 1: Create MFA Authentication Tree

1. Go to: **Top Level Realm** → **Authentication** → **Trees**
2. Click **Create Tree**
3. Name: `MFA-Login`

#### Step 2: Add Nodes

**Drag and drop nodes:**

```
[Start]
   ↓
[Username Collector]
   ↓
[Password Collector]
   ↓
[Data Store Decision]
   ├─[Success]─→ [OTP Email Sender]
   │                ↓
   │             [OTP Collector]
   │                ↓
   │             [Success]
   └─[Failure]─→ [Failure]
```

**Node Configuration:**

**1. Username Collector:**
```yaml
Type: Username Collector
Prompt: Enter your username
```

**2. Password Collector:**
```yaml
Type: Password Collector
Prompt: Enter your password
```

**3. Data Store Decision:**
```yaml
Type: Data Store Decision
(Validates against PingDS User Store)
```

**4. OTP Email Sender:**
```yaml
Type: Email Sender Node
From Email: noreply@example.com
Subject: Your OTP Code
Message: Your OTP code is: {code}
```

**5. OTP Collector:**
```yaml
Type: One-Time Password Collector
Code Length: 6
Code Validity: 300 seconds
```

#### Step 3: Set as Default Tree

1. Go to: **Top Level Realm** → **Authentication** → **Settings**
2. Under **Core** → **Organization Authentication Configuration**:
   ```yaml
   Authentication Tree: MFA-Login
   ```

#### Step 4: Test MFA Flow

```bash
curl -X POST 'http://localhost:8081/am/json/authenticate?authIndexType=service&authIndexValue=MFA-Login' \
  -H 'Content-Type: application/json' \
  -H 'Accept-API-Version: resource=2.0'
```

### Use Case 2: Risk-Based Authentication Tree

**Create tree with:**
- Device fingerprinting
- Geolocation check
- Risk scoring
- Adaptive authentication

**Nodes:**
```
[Start]
   ↓
[Username Collector]
   ↓
[Device Profile Collector]
   ↓
[Scripted Decision Node] ← Check risk score
   ├─[High Risk]─→ [MFA Required]
   │                  ↓
   │               [Success]
   ├─[Low Risk]───→ [Password Only]
   │                  ↓
   │               [Success]
   └─[Blocked]────→ [Failure]
```

### Use Case 3: Custom Authentication Node (Scripted)

#### Step 1: Create Script

1. Go to: **Top Level Realm** → **Scripts**
2. Click **New Script**
3. Configure:
   ```yaml
   Name: Custom-Validation-Script
   Script Type: Authentication Tree Decision Node
   ```

#### Step 2: Write Script Logic

```javascript
// Example: Check if user is from allowed domain
var username = nodeState.get("username");
var allowedDomains = ["example.com", "test.com"];

if (username) {
    var domain = username.split("@")[1];

    if (allowedDomains.includes(domain)) {
        outcome = "true";
    } else {
        outcome = "false";
    }
} else {
    outcome = "false";
}
```

#### Step 3: Use Script in Tree

1. In your authentication tree
2. Add **Scripted Decision Node**
3. Select: **Custom-Validation-Script**
4. Connect outcomes to appropriate nodes

### Creating Custom Nodes (Advanced)

For truly custom nodes, you need to:

1. **Develop custom node JAR:**
   - Use ForgeRock SDK
   - Implement `Node` interface
   - Package as JAR

2. **Deploy to PingAM:**
   ```bash
   # Copy JAR to PingAM container
   docker cp my-custom-node.jar pingam:/usr/local/tomcat/webapps/am/WEB-INF/lib/

   # Restart PingAM
   docker restart pingam
   ```

3. **Use in tree:**
   - Node will appear in tree builder
   - Configure and connect

**Example custom node project structure:**
```
custom-auth-node/
├── pom.xml
├── src/
│   └── main/
│       └── java/
│           └── com/example/am/
│               ├── CustomAuthNode.java
│               └── CustomAuthNodePlugin.java
└── target/
    └── custom-auth-node-1.0.jar
```

---

## Troubleshooting

### CTS Connection Issues

**Symptoms:**
- Sessions not persisting
- OAuth tokens not visible in LDAP

**Check:**
```bash
# View AM logs
docker logs pingam | grep -i "cts"

# Check CTS backend in DS
docker exec pingds //opt/opendj/bin/ldapsearch \
  --hostname pingds --port 1636 --useSSL --trustAll \
  --bindDN "cn=Directory Manager" --bindPassword Passw0rd123 \
  --baseDN "ou=tokens" "(objectClass=*)" dn numSubordinates
```

**Fix:**
- Verify CTS bind account password
- Check SSL/TLS settings
- Ensure hostname verification is disabled

### User Store Connection Issues

**Symptoms:**
- Cannot authenticate users
- Users not appearing in AM console

**Check:**
```bash
# Test bind account
docker exec pingds //opt/opendj/bin/ldapsearch \
  --hostname pingds --port 1636 --useSSL --trustAll \
  --bindDN "uid=am-identity-bind-account,ou=admins,ou=identities" \
  --bindPassword Passw0rd123 \
  --baseDN "ou=people,ou=identities" "(uid=*)" dn
```

**Fix:**
- Verify identity store bind DN and password
- Check base DN is correct: `ou=identities`
- Ensure SSL/TLS is enabled

### SAML Issues

**Symptoms:**
- SAML assertion validation fails
- Metadata import fails

**Check:**
```bash
# View SAML metadata
curl http://localhost:8081/am/saml2/jsp/exportmetadata.jsp?entityid=PingAM-IDP&realm=/

# Check SAML logs
docker logs pingam | grep -i "saml"
```

**Fix:**
- Verify clock synchronization between IdP and SP
- Check certificate validity
- Ensure metadata is up-to-date

### OAuth Issues

**Symptoms:**
- Token validation fails
- Cannot get access token

**Check:**
```bash
# Validate token
curl -H "Authorization: Bearer YOUR_TOKEN" \
  http://localhost:8081/am/oauth2/tokeninfo

# Check OAuth client config
curl -u amadmin:PASSWORD \
  http://localhost:8081/am/json/realm-config/agents/OAuth2Client/my-web-app
```

**Fix:**
- Verify client credentials
- Check redirect URI matches exactly
- Ensure OAuth provider is configured

---

## Next Steps

### For SSO/SAML:
1. Set up a test SP application
2. Configure attribute mappings
3. Test different SAML bindings (POST, Redirect, Artifact)
4. Implement Single Logout (SLO)

### For OAuth 2.0:
1. Implement PKCE for mobile apps
2. Test different grant types
3. Configure token introspection
4. Set up consent management

### For Custom Auth Trees:
1. Create scripted decision nodes
2. Implement device fingerprinting
3. Build risk-based authentication
4. Develop custom authentication nodes

### Advanced Topics:
1. Implement Social Login (Google, Facebook)
2. Configure federation with external IdPs
3. Set up policy-based authorization
4. Implement zero-trust architecture

---

## Useful Resources

**PingAM Documentation:**
- Installation Guide: `sndbx1/pingam/notes/PINGAM_INSTALLATION_GUIDE.md`
- SSL Configuration: `sndbx1/pingam/notes/SSL_CONFIGURATION_NOTES.md`
- Server URL Settings: `sndbx1/pingam/notes/SERVER_URL_EXPLAINED.md`

**API Endpoints:**
- Server Info: `http://localhost:8081/am/json/serverinfo/*`
- Authentication: `http://localhost:8081/am/json/authenticate`
- OAuth 2.0: `http://localhost:8081/am/oauth2/`
- User Info: `http://localhost:8081/am/oauth2/userinfo`
- Token Info: `http://localhost:8081/am/oauth2/tokeninfo`

**Quick Commands:**
```bash
# View all service accounts
docker exec pingds //opt/opendj/bin/ldapsearch \
  --hostname pingds --port 1636 --useSSL --trustAll \
  --bindDN "cn=Directory Manager" --bindPassword Passw0rd123 \
  --baseDN "dc=example,dc=com" "(uid=*)" dn description

# Count CTS tokens
docker exec pingds //opt/opendj/bin/ldapsearch \
  --hostname pingds --port 1636 --useSSL --trustAll \
  --bindDN "cn=Directory Manager" --bindPassword Passw0rd123 \
  --baseDN "ou=tokens" "(coreTokenType=*)" dn | grep -c "^dn:"

# Monitor AM logs
docker logs -f pingam

# Monitor DS logs
docker exec pingds tail -f //opt/opendj/logs/access
```

---

**End of Guide**
