# PingAM (Access Manager) - Overview and Concepts

## Table of Contents
1. [What is PingAM?](#what-is-pingam)
2. [Core Capabilities](#core-capabilities)
3. [Architecture](#architecture)
4. [Data Stores Explained](#data-stores-explained)
5. [Deployment Models](#deployment-models)
6. [Integration with PingDS](#integration-with-pingds)

---

## What is PingAM?

**PingAM (Access Manager)** is an enterprise-grade, centralized **Identity and Access Management (IAM)** solution that provides:

- **Authentication**: Verifying user identities
- **Authorization**: Controlling access to resources
- **Single Sign-On (SSO)**: One login for multiple applications
- **Federation**: Linking identities across different systems
- **Policy Management**: Centralized access control policies

### Beyond Just SSO

While many know PingAM for SSO, it provides much more:
- Adaptive risk-based authentication
- OAuth 2.0 and OpenID Connect support
- SAML 2.0 federation
- API security and protection
- IoT device authentication
- WebAuthn passwordless authentication

---

## Core Capabilities

### 1. **Authentication**
Verify who users are through multiple methods:
- Username/password
- Multi-factor authentication (MFA)
- Biometric authentication
- Social login (Google, Facebook, etc.)
- OATH tokens, push notifications
- WebAuthn (FIDO2)

### 2. **Authorization**
Control what authenticated users can access:
- Policy-based access control
- Role-based access control (RBAC)
- Attribute-based access control (ABAC)
- Resource-level permissions

### 3. **Federation**
Link identities across organizations:
- SAML 2.0 for enterprise SSO
- OpenID Connect for modern apps
- Cross-domain identity sharing

### 4. **Session Management**
Track and manage user sessions:
- Centralized session storage (CTS)
- Session timeout controls
- Single logout across applications

### 5. **Adaptive Risk**
Context-aware security decisions:
- Device fingerprinting
- Geo-location analysis
- Behavioral analytics
- Risk-based step-up authentication

---

## Architecture

### High-Level Components

```
┌──────────────────────────────────────────────────┐
│                   Users/Clients                  │
└────────────────────┬─────────────────────────────┘
                     │
                     ▼
┌──────────────────────────────────────────────────┐
│              PingAM Server (WAR)                 │
│  ┌────────────────────────────────────────────┐  │
│  │  Authentication Trees & Policy Engine     │  │
│  └────────────────────────────────────────────┘  │
│  ┌────────────────────────────────────────────┐  │
│  │  REST APIs & Admin Console                │  │
│  └────────────────────────────────────────────┘  │
└──────┬──────────┬──────────┬──────────┬─────────┘
       │          │          │          │
       ▼          ▼          ▼          ▼
┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐
│ Config   │ │   CTS    │ │   User   │ │ Policy/  │
│  Store   │ │  Store   │ │  Store   │ │App Store │
│ (PingDS) │ │ (PingDS) │ │ (PingDS) │ │ (PingDS) │
└──────────┘ └──────────┘ └──────────┘ └──────────┘
```

### Deployment Artifact

PingAM is distributed as a **WAR file** (`AM-8.0.2.war`) that deploys to a Java servlet container:
- **Apache Tomcat** (most common)
- JBoss/WildFly
- IBM WebSphere Liberty

### Technology Stack

- **Language**: 100% Java
- **Protocols**: HTTP, HTTPS, LDAP, LDAPS
- **Standards**: SAML 2.0, OAuth 2.0, OpenID Connect 1.0, SCIM
- **APIs**: REST, SOAP, XML

---

## Data Stores Explained

PingAM requires **multiple data stores** to function. Understanding these is crucial!

### 1. Configuration Store (Config Store)

**Purpose**: Stores PingAM's configuration settings

**What it stores:**
- Realm configurations
- Service configurations
- Authentication chain definitions
- Policy definitions
- Server settings

**Default Base DN**: `ou=am-config`

**Service Account**: `uid=am-config,ou=admins,ou=am-config`

**Requirements:**
- Dedicated DS instance (or dedicated backend on shared instance)
- Secure connection (LDAPS)
- Read/Write access

**Think of it as**: The "brain" of PingAM - all settings live here

---

### 2. CTS Store (Core Token Service)

**Purpose**: Stores session tokens and transient data

**What it stores:**
- User session tokens (SSO sessions)
- OAuth 2.0 tokens
- SAML assertions
- CSRF tokens
- Authentication state

**Default Base DN**: `ou=famrecords,ou=openam-session,ou=tokens`

**Service Account**: `uid=openam_cts,ou=admins,ou=famrecords,ou=openam-session,ou=tokens`

**Requirements:**
- High-performance DS instance
- Heavy read/write operations
- Can share DS instance with config store
- Requires proper tuning for performance

**Characteristics:**
- **High Volume**: Constantly creates/deletes entries
- **Short-lived Data**: Tokens expire and are removed
- **Performance Critical**: Slow CTS = slow logins

**Think of it as**: The "memory" of PingAM - active sessions live here

---

### 3. User Store (Identity Repository)

**Purpose**: Stores user identities and credentials

**What it stores:**
- User accounts (uid, cn, sn, mail, etc.)
- User passwords
- User attributes (phone, address, role, etc.)
- Group memberships
- Device profiles

**Default Base DN**: `ou=identities` (customizable)

**Service Account**: `uid=am-identity-bind-account,ou=admins,ou=identities`

**Requirements:**
- Can be existing LDAP directory
- Can be shared with other applications
- Supports: PingDS, Active Directory, Oracle UD, etc.
- Read access (minimum), Read/Write (for self-registration)

**Think of it as**: The "user database" - who your users are

---

### 4. Policy/Application Store

**Purpose**: Stores authorization policies and application definitions

**What it stores:**
- Resource-based policies
- Application configurations
- Policy sets
- Privilege definitions

**Default Base DN**: Often combined with config store (`ou=services,ou=am-config`)

**Requirements:**
- Can share backend with config store
- Secure connection required

**Think of it as**: The "rule book" - what users can access

---

## Data Store Strategy Options

### Option 1: All-in-One (Development/Testing)
```
┌─────────────────────────────────┐
│         Single PingDS           │
│  ┌────────────────────────────┐ │
│  │ Backend: am-config         │ │
│  │ (Config + CTS + Policy)    │ │
│  └────────────────────────────┘ │
│  ┌────────────────────────────┐ │
│  │ Backend: identities        │ │
│  │ (User Store)               │ │
│  └────────────────────────────┘ │
└─────────────────────────────────┘
```

**Pros:** Simple, easy to manage, minimal resources
**Cons:** Single point of failure, performance limitations

---

### Option 2: Separated CTS (Production)
```
┌──────────────────┐  ┌──────────────────┐
│ PingDS Instance 1│  │ PingDS Instance 2│
│                  │  │                  │
│ Config Store     │  │ CTS Store        │
│ Policy Store     │  │ (High Performance│
│ User Store       │  │  Optimized)      │
└──────────────────┘  └──────────────────┘
```

**Pros:** CTS performance optimized separately, better scalability
**Cons:** More complex, requires 2 DS instances

---

### Option 3: Fully Distributed (Enterprise)
```
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│Config Store  │ │  CTS Store   │ │  User Store  │
│  (PingDS)    │ │  (PingDS)    │ │  (AD/PingDS) │
└──────────────┘ └──────────────┘ └──────────────┘
```

**Pros:** Maximum flexibility, independent scaling, fault isolation
**Cons:** Most complex, highest resource usage

---

## Deployment Models

### 1. Traditional Deployment (LDAP-Based Config)

```
PingAM reads config from → LDAP Config Store
Changes made in UI → Saved to LDAP
Multiple AM servers → Share same LDAP config
```

**Use Case:** Traditional enterprise deployments
**Benefits:** Centralized config, easy multi-server setup
**Drawbacks:** Requires LDAP for all config

---

### 2. File-Based Configuration (FBC)

```
PingAM reads config from → JSON files on disk
Changes made in UI → Saved to local JSON files
Multiple AM servers → Each has own config files
```

**Use Case:** Docker/Kubernetes, DevOps, cloud-native
**Benefits:** Config as code, version control, faster startup
**Drawbacks:** No automatic config sync between servers

---

## Integration with PingDS

### Our Setup Plan

We'll configure PingDS to provide **all three data stores** in a single instance (Option 1):

```
PingDS Instance (pingds)
│
├─ Backend: am-config
│  ├─ ou=am-config          (Config Store)
│  ├─ ou=services           (Policy/Application Store)
│  └─ ou=famrecords,ou=openam-session,ou=tokens (CTS Store)
│
└─ Backend: identities
   └─ ou=identities         (User Store - ALREADY EXISTS!)
```

### Why This Works

1. **User Store Ready**: We already have `ou=identities` with users!
2. **Single Instance**: Easier to manage for learning
3. **Resource Efficient**: One DS container instead of 3-4
4. **Production-Like**: Same structure as production, just combined

---

## Key Concepts Summary

| Concept | Description |
|---------|-------------|
| **AM Server** | Java WAR file running in Tomcat |
| **Config Store** | Where AM stores its configuration |
| **CTS Store** | Where AM stores active sessions/tokens |
| **User Store** | Where AM looks up users for authentication |
| **Policy Store** | Where AM stores authorization policies |
| **Authentication Tree** | Visual workflow defining login process |
| **Realm** | Logical division of users/policies (like a tenant) |
| **Service Account** | LDAP user AM uses to connect to stores |

---

## Authentication Flow Example

1. **User visits app** → App redirects to PingAM
2. **PingAM shows login** → User enters credentials
3. **PingAM queries User Store (PingDS)** → Validates username/password
4. **PingAM creates session** → Stores in CTS Store (PingDS)
5. **PingAM checks policies** → Reads from Policy Store (PingDS)
6. **PingAM grants access** → Issues token to user
7. **User accesses app** → App validates token with PingAM

All these steps involve reading/writing to PingDS!

---

## Next Steps

1. Read `DATA_STORES_PREPARATION.md` to prepare PingDS for AM
2. Read `PINGAM_INSTALLATION_GUIDE.md` for step-by-step setup
3. Read `PINGAM_QUICK_REFERENCE.md` for command cheat sheet

---

## Quick Facts

- **Release**: PingAM 8.0.2 (we'll use this version)
- **License**: Commercial (evaluation mode for learning)
- **Platform**: Java-based, runs on Linux/Windows
- **Ports**: 8080 (HTTP), 8443 (HTTPS)
- **Admin User**: `amAdmin` (default)
- **Container**: Apache Tomcat 9.x recommended

---

**Remember**: PingAM is the **gatekeeper** for your applications. It decides **who can log in** (authentication) and **what they can access** (authorization). Everything it needs to make these decisions is stored in PingDS!
