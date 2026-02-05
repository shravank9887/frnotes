`# Lab 1: Realm Architecture & DS Storage

## Lab Completion Status
✅ **Completed**: Created `/techcorp` realm and explored DS storage structure

---

## What You Accomplished

### 1. Created TechCorp Realm
- **Realm Path**: `/techcorp`
- **Alias**: `techcorp.example.com`
- **Status**: Active
- **Access**: http://pingam:8081/am/console (navigate to `/techcorp` realm)

### 2. Understood Directory Server Storage

#### Realm Storage Location
```
DN: o=techcorp,ou=services,ou=am-config

Attributes:
- objectClass: sunRealmService
- objectClass: top
- o: techcorp
- sunOrganizationAlias: techcorp.example.com
- sunOrganizationStatus: Active
```

#### Directory Structure
```
ou=am-config                                    (AM Config Store Root)
├── ou=services
│   ├── ou=iPlanetAMAuthService               (Global auth service configs)
│   ├── ou=authenticationTreesService         (Global auth tree configs)
│   └── o=techcorp                             ⭐ YOUR REALM
│       ├── [objectClass: sunRealmService]
│       ├── [sunOrganizationAlias: techcorp.example.com]
│       └── ou=services                        (Realm-specific services)
│           ├── ou=authenticationTreesService  (Auth trees for this realm)
│           ├── ou=sunIdentityRepositoryService (User store config)
│           ├── ou=DataStoreDecisionNode
│           ├── ou=UsernameCollectorNode
│           ├── ou=PageNode
│           └── ... (24 total services)
├── ou=admins                                  (AM admin users)
├── ou=tokens                                  (UMA tokens)
```

---

## Key Concepts Learned

### 1. AM Uses 3 Separate Data Stores in DS

#### a) Configuration Store (ou=am-config)
- **Purpose**: Stores AM configuration
- **Contains**:
  - Realm definitions and hierarchy
  - Authentication trees and chains
  - Services configuration
  - Policies and policy sets
  - OAuth2/SAML configurations

#### b) Identity Store (ou=identities)
- **Purpose**: Stores user identities
- **Contains**:
  - Users (uid=username,ou=people,ou=identities)
  - Groups
  - Roles
  - Can be external (LDAP/AD)

#### c) CTS Store (ou=tokens)
- **Purpose**: Core Token Service - stores session & token data
- **Contains**:
  - AM session tokens
  - OAuth2 access/refresh tokens
  - SAML assertions
  - Blacklisted tokens
  - Temporary data with TTL

---

## Realm Architecture Concepts

### What is a Realm?
A **realm** is a logical partition/namespace in ForgeRock AM that provides:
- **Isolated Authentication**: Each realm has its own auth trees, chains, modules
- **Isolated Authorization**: Separate policies and policy sets
- **Isolated Users**: Can have separate identity stores or share them
- **Isolated Applications**: OAuth2 clients, SAML entities

### Realm Hierarchy
Realms can be nested:
```
/ (Top Level Realm)
├── /techcorp
│   ├── /techcorp/employees
│   └── /techcorp/contractors
├── /partners
└── /customers
```

### Realm Aliases & DNS Routing
- **Aliases** allow automatic realm routing based on domain
- Example: `techcorp.example.com` → `/techcorp` realm
- User accesses: `https://am.example.com/am/XUI/?realm=techcorp`
- Or via alias: `https://techcorp.example.com/am/XUI/`

### Services in Realms
Each realm inherits global services but can override them:
- **Global Services** (ou=services,ou=am-config): Defaults for all realms
- **Realm Services** (ou=services,o=techcorp,...): Overrides for specific realm

---

## Interview Questions & Answers

### Q1: When would you use multiple realms?
**Answer**:
- **Multi-tenancy**: SaaS applications serving multiple customers
- **Separation of concerns**: B2B vs B2C authentication
- **Different security requirements**: High-security vs standard authentication
- **Organizational boundaries**: Employees vs partners vs customers
- **Development/Testing**: Separate realms for dev/test/prod in same AM instance

### Q2: Can realms share users?
**Answer**: Yes! Multiple realms can point to the same identity repository (LDAP/AD). The identity store is configured per-realm via the "sunIdentityRepositoryService". This allows:
- Central user management
- SSO across realms (if configured)
- Different authentication flows for same users

### Q3: How is realm configuration stored in DS?
**Answer**:
- Realm base entry: `o={realm-name},ou=services,ou=am-config`
- ObjectClass: `sunRealmService`
- Realm-specific services under: `ou=services,o={realm-name},ou=services,ou=am-config`
- Each service is an OU with configuration stored as attributes
- Authentication trees stored in: `ou=authenticationTreesService,ou=services,o={realm-name},...`

### Q4: What's the difference between Config Store, Identity Store, and CTS Store?
**Answer**:
- **Config Store**: AM configuration (realms, trees, policies) - changes infrequently
- **Identity Store**: User/group data - can be external LDAP/AD
- **CTS Store**: High-volume, short-lived tokens and sessions - optimized for performance with TTL-based cleanup

### Q5: How do you verify a realm was created correctly?
**Answer**:
1. **Via Console**: Navigate to realm, check services and configuration
2. **Via LDAP**:
   ```bash
   ldapsearch -b "ou=am-config" "(sunOrganizationAlias={realm-alias})"
   ```
3. **Via Amster**:
   ```
   connect http://am:8080/am -k /path/to/key
   query-realms
   ```
4. **Via REST**:
   ```bash
   curl -X GET "http://am:8080/am/json/realms?_queryFilter=true" \
     -H "iPlanetDirectoryPro: {admin-token}"
   ```

---

## Practical Commands Used

### Check Realm in DS
```bash
docker exec pingds bash -c '/opt/opendj/bin/ldapsearch \
  --hostname localhost --port 1636 --useSsl --trustAll \
  --bindDN "cn=Directory Manager" --bindPassword "Passw0rd123" \
  --baseDN "ou=am-config" --searchScope sub \
  "(sunOrganizationAlias=techcorp)" \
  dn o sunOrganizationAlias objectClass'
```

### List All Realms
```bash
docker exec pingds bash -c '/opt/opendj/bin/ldapsearch \
  --hostname localhost --port 1636 --useSsl --trustAll \
  --bindDN "cn=Directory Manager" --bindPassword "Passw0rd123" \
  --baseDN "ou=am-config" --searchScope sub \
  "(objectClass=sunismanagedorganization)" \
  dn o sunOrganizationAlias'
```

### Check Realm Services
```bash
docker exec pingds bash -c '/opt/opendj/bin/ldapsearch \
  --hostname localhost --port 1636 --useSsl --trustAll \
  --bindDN "cn=Directory Manager" --bindPassword "Passw0rd123" \
  --baseDN "ou=services,o=techcorp,ou=services,ou=am-config" \
  --searchScope one "(objectClass=*)" dn'
```

---

## Next Steps

Proceed to **Lab 2**: Create Authentication Trees/Journeys in the `/techcorp` realm

Topics to cover:
- Understanding authentication trees vs chains
- Creating a simple username/password journey
- Using nodes: UsernameCollector, PasswordCollector, DataStoreDecision
- Testing the authentication flow
- Viewing tree configuration in DS
`