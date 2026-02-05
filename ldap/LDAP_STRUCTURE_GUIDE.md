# PingDS LDAP Structure Guide

## Table of Contents
1. [LDAP Command Structure](#ldap-command-structure)
2. [PingDS Directory Structure](#pingds-directory-structure)
3. [Understanding Object Classes](#understanding-object-classes)
4. [Sample User Data](#sample-user-data)

---

## LDAP Command Structure

### Basic ldapsearch Command Anatomy

```bash
/opt/opendj/bin/ldapsearch \
    --hostname pingds \              # Server hostname
    --port 1636 \                    # Port (1389=LDAP, 1636=LDAPS, 4444=Admin)
    --useSSL \                       # Use SSL/TLS encryption
    --trustAll \                     # Trust all certificates (dev only!)
    --bindDN "cn=Directory Manager" \ # User DN to authenticate as
    --bindPassword "Passw0rd123" \   # Password for authentication
    --baseDN "ou=identities" \       # Where to start the search
    --searchScope sub \              # Search scope (base|one|sub)
    "(objectClass=person)" \         # LDAP filter (what to search for)
    dn cn mail                       # Attributes to return (optional)
```

---

### Key Options Explained

#### Connection Options

| Option | Description | Example |
|--------|-------------|---------|
| `--hostname` | LDAP server hostname/IP | `--hostname pingds` |
| `--port` | Port number | `--port 1636` (LDAPS) |
| `--useSSL` | Use SSL/TLS encryption | Flag only, no value |
| `--trustAll` | Trust all SSL certificates | For dev/testing only! |

#### Authentication Options

| Option | Description | Example |
|--------|-------------|---------|
| `--bindDN` | Distinguished Name to bind as | `--bindDN "cn=Directory Manager"` |
| `--bindPassword` | Password for the bind DN | `--bindPassword "Passw0rd123"` |

**Anonymous Bind**: Omit `--bindDN` and `--bindPassword` for anonymous access (if allowed).

#### Search Options

| Option | Values | Description |
|--------|--------|-------------|
| `--baseDN` | DN string | Starting point for the search |
| `--searchScope` | `base`, `one`, `sub` | How deep to search |
| | `base` | Only the base DN entry itself |
| | `one` | Base DN + one level below |
| | `sub` | Base DN + all levels below (recursive) |

#### Filter (Search Criteria)

The filter is an LDAP query in parentheses:

| Filter | Meaning |
|--------|---------|
| `(objectClass=*)` | All entries |
| `(objectClass=person)` | All person objects |
| `(uid=jdoe)` | Entries where uid equals jdoe |
| `(cn=John*)` | Entries where cn starts with John |
| `(&(objectClass=person)(mail=*))` | AND: persons with mail attribute |
| `(\|(uid=jdoe)(uid=admin))` | OR: uid is jdoe OR admin |
| `(!(objectClass=group))` | NOT: entries that are not groups |

#### Attributes to Return

After the filter, specify which attributes to return:

```bash
# Return all attributes
ldapsearch ... "(objectClass=person)"

# Return only specific attributes
ldapsearch ... "(objectClass=person)" dn cn mail

# Return all operational attributes
ldapsearch ... "(objectClass=person)" +

# Return all user + operational attributes
ldapsearch ... "(objectClass=person)" * +
```

---

## PingDS Directory Structure

### Overview - Directory Information Tree (DIT)

```
Root ("")
│
├── ou=identities              [User identity store for AM]
│   ├── ou=people              [Regular users go here]
│   ├── ou=groups              [User groups]
│   └── ou=admins              [Admin accounts]
│       └── uid=am-identity-bind-account  [AM service account]
│
├── ou=am-config               [AM configuration store]
│   └── ou=admins
│       └── uid=am-config      [AM config service account]
│
├── ou=tokens                  [Core Token Service (CTS) for sessions]
│   └── ou=openam-session
│       └── ou=famrecords
│           └── ou=admins
│               └── uid=openam_cts  [CTS service account]
│
├── cn=Directory Manager       [Root admin account]
│
└── uid=Monitor                [Server monitoring info]
```

---

### Detailed Structure

#### 1. **ou=identities** - User Identity Store

**Purpose**: Stores user accounts and groups for ForgeRock Access Management (AM)

**Structure**:
- **ou=people**: Regular user accounts (employees, customers, etc.)
- **ou=groups**: User groups and roles
- **ou=admins**: Administrative service accounts

**Object Classes Used**:
- `organizationalUnit` - For OU containers
- `inetOrgPerson` - For user entries (standard LDAP user)
- `groupOfUniqueNames` or `groupOfNames` - For groups

**Typical User Entry**:
```ldif
dn: uid=jdoe,ou=people,ou=identities
objectClass: top
objectClass: person
objectClass: organizationalPerson
objectClass: inetOrgPerson
uid: jdoe
cn: John Doe
sn: Doe
givenName: John
mail: jdoe@example.com
telephoneNumber: +1-555-0100
userPassword: {SSHA}...encrypted...
```

---

#### 2. **ou=am-config** - AM Configuration Store

**Purpose**: Stores ForgeRock Access Management configuration data

**Structure**:
- **ou=admins**: Service accounts for AM configuration access
  - **uid=am-config**: Service account AM uses to read/write configuration

**Authentication**:
- Bind DN: `uid=am-config,ou=admins,ou=am-config`
- Password: `Passw0rd123` (from `AM_CONFIG_PASSWORD`)

**Usage**: AM connects to this DN to store realms, policies, services, etc.

---

#### 3. **ou=tokens** - Core Token Service (CTS)

**Purpose**: Stores session tokens, OAuth2 tokens, SAML assertions

**Structure**:
- **ou=openam-session/ou=famrecords**: Token storage location
- **ou=admins**: CTS service account
  - **uid=openam_cts**: Service account for CTS operations

**Authentication**:
- Bind DN: `uid=openam_cts,ou=admins,ou=famrecords,ou=openam-session,ou=tokens`
- Password: `Password123` ⚠️ (Note: Different from other passwords!)

**Object Class**: `frCoreToken` - ForgeRock-specific token object

**Usage**: High-volume, high-performance token storage with TTL

---

#### 4. **cn=Directory Manager** - Root Administrator

**Purpose**: Superuser account with full directory access

**Authentication**:
- Bind DN: `cn=Directory Manager`
- Password: `Passw0rd123` (from `DS_ROOT_PASSWORD`)

**Usage**:
- Initial setup
- Schema modifications
- Full directory administration
- **Security**: Should NOT be used by applications in production

---

#### 5. **uid=Monitor** - Server Monitoring

**Purpose**: Provides real-time server statistics and monitoring data

**Usage**:
```bash
ldapsearch --baseDN "cn=monitor" --searchScope sub "(objectClass=*)" cn
```

Returns metrics like:
- Connection count
- Operations per second
- Backend statistics
- JVM memory usage

---

## Understanding Object Classes

### Object Class Hierarchy

```
top (abstract)
 ├── person (structural)
 │    ├── organizationalPerson (structural)
 │    │    └── inetOrgPerson (structural) ← Most common for users
 │    └── residentialPerson (structural)
 ├── organizationalUnit (structural) ← For OUs
 ├── groupOfNames (structural) ← For groups
 └── groupOfUniqueNames (structural) ← For groups with unique members
```

### Common Object Classes for Users

#### **inetOrgPerson** (Recommended for users)

**Inherits from**: `organizationalPerson` → `person` → `top`

**Required Attributes** (MUST):
- `cn` - Common name (e.g., "John Doe")
- `sn` - Surname (e.g., "Doe")

**Optional Attributes** (MAY):
- `uid` - User ID (e.g., "jdoe")
- `mail` - Email address
- `telephoneNumber` - Phone number
- `givenName` - First name
- `displayName` - Display name
- `title` - Job title
- `employeeNumber` - Employee ID
- `departmentNumber` - Department
- `manager` - Manager DN
- `userPassword` - Encrypted password
- `jpegPhoto` - Photo
- `mobile` - Mobile phone
- Many more...

### LDAP Attribute Matching Rules

- **Exact Match**: `(uid=jdoe)`
- **Substring**: `(cn=John*)` - starts with
- **Substring**: `(cn=*Doe)` - ends with
- **Substring**: `(cn=*John*)` - contains
- **Presence**: `(mail=*)` - has mail attribute
- **Absence**: `(!(mail=*))` - doesn't have mail attribute

---

## Sample User Data

### Understanding the Examples

I'll create:
1. **Regular users** in `ou=people,ou=identities`
2. **Groups** in `ou=groups,ou=identities`
3. **LDIF format** for easy import

### What is LDIF?

**LDIF** = LDAP Data Interchange Format

- Text file format for LDAP data
- Used to import/export directory data
- Each entry separated by blank line
- Attributes are `name: value` pairs

---

## Next Steps

See the sample LDIF files:
1. `sample-users.ldif` - Sample user accounts
2. `sample-groups.ldif` - Sample groups
3. Instructions on how to import them

