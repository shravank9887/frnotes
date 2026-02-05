# Lab 2: Authentication Trees (Journeys)

## Objective
Create and understand authentication trees in the `/techcorp` realm.

---

## Pre-requisite: Create Test Users

Before testing authentication, create test users in the techcorp realm.

### Method 1: Via AM Console (Easiest)

1. Go to: http://pingam:8081/am/console
2. Select realm: **techcorp**
3. Go to: **Identities** → **+ Add Identity**
4. Create users:

| Username | Password | Email |
|----------|----------|-------|
| demo | changeit | demo@techcorp.com |
| alice | P@ssw0rd | alice@techcorp.com |
| bob | P@ssw0rd | bob@techcorp.com |

### Method 2: Via AM REST API

```bash
# First, get admin SSO token
TOKEN=$(curl -s -X POST "http://pingam:8081/am/json/realms/root/authenticate" \
  -H "Content-Type: application/json" \
  -H "X-OpenAM-Username: amadmin" \
  -H "X-OpenAM-Password: Passw0rd123" \
  -H "Accept-API-Version: resource=2.0, protocol=1.0" | jq -r '.tokenId')

# Create user 'demo' in techcorp realm
curl -X POST "http://pingam:8081/am/json/realms/root/realms/techcorp/users?_action=create" \
  -H "Content-Type: application/json" \
  -H "iPlanetDirectoryPro: $TOKEN" \
  -H "Accept-API-Version: resource=4.0, protocol=2.1" \
  -d '{
    "username": "demo",
    "userpassword": "changeit",
    "mail": ["demo@techcorp.com"],
    "givenName": ["Demo"],
    "sn": ["User"]
  }'

# Create user 'alice'
curl -X POST "http://pingam:8081/am/json/realms/root/realms/techcorp/users?_action=create" \
  -H "Content-Type: application/json" \
  -H "iPlanetDirectoryPro: $TOKEN" \
  -H "Accept-API-Version: resource=4.0, protocol=2.1" \
  -d '{
    "username": "alice",
    "userpassword": "P@ssw0rd",
    "mail": ["alice@techcorp.com"],
    "givenName": ["Alice"],
    "sn": ["Smith"]
  }'

# Create user 'bob'
curl -X POST "http://pingam:8081/am/json/realms/root/realms/techcorp/users?_action=create" \
  -H "Content-Type: application/json" \
  -H "iPlanetDirectoryPro: $TOKEN" \
  -H "Accept-API-Version: resource=4.0, protocol=2.1" \
  -d '{
    "username": "bob",
    "userpassword": "P@ssw0rd",
    "mail": ["bob@techcorp.com"],
    "givenName": ["Bob"],
    "sn": ["Jones"]
  }'
```

### Method 3: Via LDAP (Direct to DS)

```bash
# Create LDIF file for users
cat > /tmp/users.ldif << 'EOF'
dn: uid=demo,ou=people,ou=identities
objectClass: iplanet-am-managed-person
objectClass: inetuser
objectClass: sunFMSAML2NameIdentifier
objectClass: inetorgperson
objectClass: devicePrintProfilesContainer
objectClass: kbaInfoContainer
objectClass: organizationalperson
objectClass: top
objectClass: person
objectClass: sunAMAuthAccountLockout
objectClass: iplanet-am-user-service
objectClass: forgerock-am-dashboard-service
objectClass: oathDeviceProfilesContainer
objectClass: pushDeviceProfilesContainer
objectClass: webauthnDeviceProfilesContainer
uid: demo
cn: Demo User
sn: User
givenName: Demo
mail: demo@techcorp.com
userPassword: changeit
inetUserStatus: Active

dn: uid=alice,ou=people,ou=identities
objectClass: iplanet-am-managed-person
objectClass: inetuser
objectClass: inetorgperson
objectClass: organizationalperson
objectClass: top
objectClass: person
uid: alice
cn: Alice Smith
sn: Smith
givenName: Alice
mail: alice@techcorp.com
userPassword: P@ssw0rd
inetUserStatus: Active

dn: uid=bob,ou=people,ou=identities
objectClass: iplanet-am-managed-person
objectClass: inetuser
objectClass: inetorgperson
objectClass: organizationalperson
objectClass: top
objectClass: person
uid: bob
cn: Bob Jones
sn: Jones
givenName: Bob
mail: bob@techcorp.com
userPassword: P@ssw0rd
inetUserStatus: Active
EOF

# Add users to DS
docker exec -i pingds /opt/opendj/bin/ldapmodify \
  --hostname localhost --port 1636 --useSsl --trustAll \
  --bindDN "cn=Directory Manager" --bindPassword "Passw0rd123" \
  --add < /tmp/users.ldif
```

### Verify Users Created

```bash
# Via REST API
curl -s "http://pingam:8081/am/json/realms/root/realms/techcorp/users?_queryFilter=true" \
  -H "iPlanetDirectoryPro: $TOKEN" \
  -H "Accept-API-Version: resource=4.0, protocol=2.1" | jq '.result[].username'

# Via LDAP
docker exec pingds bash -c '/opt/opendj/bin/ldapsearch \
  --hostname localhost --port 1636 --useSsl --trustAll \
  --bindDN "cn=Directory Manager" --bindPassword "Passw0rd123" \
  --baseDN "ou=people,ou=identities" \
  --searchScope sub "(objectClass=person)" uid cn mail'
```

---

## Part 1: Understanding Trees vs Chains

### Authentication Chains (Legacy)
- Linear, step-by-step authentication
- Each module succeeds or fails
- Limited branching logic
- Being phased out in favor of trees

### Authentication Trees (Modern - "Journeys")
- **Visual, node-based design**
- Non-linear flows with branching
- Reusable inner trees
- Better user experience
- Supports callbacks for progressive data collection

---

## Part 2: Create Your First Tree

### Step 1: Navigate to Authentication Trees

1. Go to **AM Console**: http://pingam:8081/am/console
2. Select realm: **techcorp**
3. Go to: **Authentication** → **Trees**
4. You'll see the default tree: `login` (used for XUI login)

### Step 2: Create a Simple Login Tree

Click **+ Create Tree** and name it: `TechCorpLogin`

**Build this flow**:
```
[Start]
   ↓
[Page Node] ─────────────────┐
   │                         │
   ├── Username Collector    │
   │                         │ (Groups nodes into single page)
   └── Password Collector    │
         ↓                   │
─────────────────────────────┘
         ↓
[Data Store Decision]
   │
   ├── True → [Success]
   │
   └── False → [Failure]
```

### Step 3: Adding Nodes

1. **Page Node** (drag from palette)
   - Groups multiple collectors into one login page
   - Drag inside it:
     - **Username Collector Node**
     - **Password Collector Node**

2. **Data Store Decision Node**
   - Validates credentials against the identity store
   - Connect Page Node → Data Store Decision

3. **Connect Outcomes**
   - Data Store Decision → True → Success (green circle)
   - Data Store Decision → False → Failure (red circle)

4. Click **Save**

---

## Part 3: Test Your Tree

### Method 1: Direct URL
```
http://pingam:8081/am/XUI/?realm=techcorp&service=TechCorpLogin
```

### Method 2: REST API Testing
```bash
# Step 1: Start authentication (get callbacks)
curl -X POST "http://pingam:8081/am/json/realms/root/realms/techcorp/authenticate?authIndexType=service&authIndexValue=TechCorpLogin" \
  -H "Content-Type: application/json" \
  -H "Accept-API-Version: resource=2.0, protocol=1.0"
```

This returns callbacks for username/password. Then submit with credentials:
```bash
curl -X POST "http://pingam:8081/am/json/realms/root/realms/techcorp/authenticate?authIndexType=service&authIndexValue=TechCorpLogin" \
  -H "Content-Type: application/json" \
  -H "Accept-API-Version: resource=2.0, protocol=1.0" \
  -d '{
    "authId": "<authId-from-previous-response>",
    "callbacks": [
      {
        "type": "NameCallback",
        "output": [{"name": "prompt", "value": "User Name"}],
        "input": [{"name": "IDToken1", "value": "demo"}]
      },
      {
        "type": "PasswordCallback",
        "output": [{"name": "prompt", "value": "Password"}],
        "input": [{"name": "IDToken2", "value": "changeit"}]
      }
    ]
  }'
```

---

## Part 4: Examine Tree in Directory Server

### Find the Tree Configuration
```bash
docker exec pingds bash -c '/opt/opendj/bin/ldapsearch \
  --hostname localhost --port 1636 --useSsl --trustAll \
  --bindDN "cn=Directory Manager" --bindPassword "Passw0rd123" \
  --baseDN "ou=authenticationTreesService,ou=services,o=techcorp,ou=services,ou=am-config" \
  --searchScope sub "(objectClass=*)" dn'
```

### View Tree Structure
```bash
docker exec pingds bash -c '/opt/opendj/bin/ldapsearch \
  --hostname localhost --port 1636 --useSsl --trustAll \
  --bindDN "cn=Directory Manager" --bindPassword "Passw0rd123" \
  --baseDN "ou=authenticationTreesService,ou=services,o=techcorp,ou=services,ou=am-config" \
  --searchScope sub "(sunServiceID=TechCorpLogin)"'
```

### Tree Storage Structure
```
Full DN path:
ou=TechCorpLogin,ou=default,ou=OrganizationConfig,ou=1.0,ou=authenticationTreesService,ou=services,o=techcorp,ou=services,ou=am-config

Key attributes (sunKeyValue):
├── entryNodeId=<UUID>           ← Starting node
├── innerTreeOnly=false          ← Can be called directly
├── enabled=true
├── staticNodes={...}            ← UI positioning (Success/Failure nodes)
└── nodes={...}                  ← Node definitions and connections
    ├── "nodeUUID": {
    │     "displayName": "Page Node",
    │     "nodeType": "PageNode",
    │     "connections": {"outcome": "nextNodeUUID"}
    │   }
    └── ...
```

### Actual TechCorpLogin Tree in DS
```json
{
  "a916a8e4-...": {
    "displayName": "Page Node",
    "nodeType": "PageNode",
    "connections": {"outcome": "bc5c798d-..."}
  },
  "bc5c798d-...": {
    "displayName": "Data Store Decision",
    "nodeType": "DataStoreDecisionNode",
    "connections": {"true": "Success", "false": "Failure"}
  }
}
```

---

## Part 5: Key Node Types to Know

### Collector Nodes (Gather Input)
| Node | Purpose |
|------|---------|
| Username Collector | Collects username |
| Password Collector | Collects password |
| Attribute Collector | Collects custom attributes |
| Platform Username | Collects with username hints |
| Platform Password | Collects with password managers support |

### Decision Nodes (Make Choices)
| Node | Purpose |
|------|---------|
| Data Store Decision | Validates against identity store |
| LDAP Decision | Validates against external LDAP |
| Scripted Decision | Custom JS/Groovy logic |
| Inner Tree Evaluator | Calls another tree |

### Action Nodes (Do Something)
| Node | Purpose |
|------|---------|
| Set Session Properties | Add values to session |
| Increment Login Count | Track login attempts |
| Create Object | Create user/resource |
| Patch Object | Update user attributes |

### Utility Nodes
| Node | Purpose |
|------|---------|
| Page Node | Groups collectors into single page |
| Message Node | Display messages to user |
| Retry Limit Decision | Limit authentication attempts |

---

## Part 6: Exercise - Enhanced Login Tree

Create a more sophisticated tree: `TechCorpSecureLogin`

**Requirements**:
1. Allow 3 login attempts before showing lockout message
2. Display custom message on failure
3. Log successful logins

**Flow**:
```
[Start]
   ↓
[Retry Limit Decision] ────────────────┐
   │                                   │
   │ (retry)                           │ (reject - attempts exceeded)
   ↓                                   ↓
[Page Node]                       [Message Node] → [Failure]
   │                              "Too many attempts.
   ├── Username Collector          Please try again later."
   └── Password Collector
         ↓
[Data Store Decision]
   │
   ├── True → [Set Session Properties] → [Success]
   │          (set: loginTime, loginIP)
   │
   └── False → (back to Retry Limit)
```

### Configuration Tips:
- **Retry Limit Decision Node**:
  - Retry Limit: 3
  - Lock Behavior: "Increment and Save in Tree State"

- **Message Node**:
  - Message: `{"en": "Account temporarily locked. Please try again in 5 minutes."}`

- **Set Session Properties**:
  - Key: `loginTimestamp`, Value: `${now}`

---

## Interview Questions & Answers

### Q1: What's the difference between Authentication Trees and Chains?
**Answer**:
- **Chains** are linear, module-by-module authentication (legacy)
- **Trees** are visual, node-based with branching logic (modern)
- Trees support:
  - Non-linear flows
  - Reusable inner trees
  - Better UX with callbacks
  - Progressive profiling
- ForgeRock recommends trees for all new implementations

### Q2: How do authentication callbacks work?
**Answer**:
The tree uses a callback mechanism:
1. AM sends callback requests (what input is needed)
2. Client collects input and sends callback responses
3. AM processes responses and moves to next node
4. Repeats until Success or Failure

Callback types: `NameCallback`, `PasswordCallback`, `TextOutputCallback`, `ChoiceCallback`, etc.

### Q3: What is an Inner Tree and when would you use it?
**Answer**:
An **Inner Tree** is a reusable authentication tree that can be called from other trees via the "Inner Tree Evaluator" node.

Use cases:
- **MFA flows** - Call MFA tree from multiple login trees
- **Step-up authentication** - Reusable verification flow
- **Common validation** - Terms acceptance, CAPTCHA
- **Modularity** - DRY principle for auth flows

### Q4: How do you debug authentication tree issues?
**Answer**:
1. **Enable Debug Logging**:
   - Go to: Debug → Configuration → Instance → "Authentication"
   - Set to "MESSAGE" level

2. **Check AM Logs**:
   ```bash
   docker logs pingam | grep -i "auth"
   ```

3. **Test via REST API** to see exact callbacks and responses

4. **Use Message Nodes** to display debug info during development

5. **Check Node Configuration** in DS if tree doesn't appear

### Q5: How are trees stored in the directory?
**Answer**:
Trees are stored in DS under:
```
ou=authenticationTreesService,ou=services,o={realm},ou=services,ou=am-config
```

Each tree has:
- `sunServiceID`: Tree name
- `sunKeyValue`: JSON containing:
  - Tree nodes with UUIDs
  - Node connections (edges)
  - Entry point node
  - Node-specific configuration

---

## Practical Verification Commands

### List All Trees in Realm
```bash
docker exec pingds bash -c '/opt/opendj/bin/ldapsearch \
  --hostname localhost --port 1636 --useSsl --trustAll \
  --bindDN "cn=Directory Manager" --bindPassword "Passw0rd123" \
  --baseDN "ou=authenticationTreesService,ou=services,o=techcorp,ou=services,ou=am-config" \
  --searchScope sub "(objectClass=sunService)" sunServiceID'
```

### View Specific Tree Config
```bash
docker exec pingds bash -c '/opt/opendj/bin/ldapsearch \
  --hostname localhost --port 1636 --useSsl --trustAll \
  --bindDN "cn=Directory Manager" --bindPassword "Passw0rd123" \
  --baseDN "ou=authenticationTreesService,ou=services,o=techcorp,ou=services,ou=am-config" \
  --searchScope sub "(sunServiceID=TechCorpLogin)" sunKeyValue' | head -100
```

---

## Checklist

- [ ] Created `TechCorpLogin` tree
- [ ] Tested via XUI URL
- [ ] Tested via REST API
- [ ] Viewed tree in DS
- [ ] (Optional) Created `TechCorpSecureLogin` with retry logic
- [ ] Understand callback mechanism
- [ ] Know key node types

---

## Next Lab

**Lab 3**: OAuth2/OIDC Provider Setup
- Configure OAuth2 provider in techcorp realm
- Create OAuth2 client applications
- Understand authorization code flow
- Test with Postman/curl
