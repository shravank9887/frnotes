# Lab 4 - SAML2 Federation

## Overview

Configure AM as both a **SAML2 Identity Provider (IdP)** and a **Service Provider (SP)** using two different realms to simulate federation:

| Role | Realm | Meaning |
|------|-------|---------|
| **IdP** (Identity Provider) | `/techcorp` | Authenticates users, issues SAML assertions |
| **SP** (Service Provider) | `/` (root) | Relies on IdP to authenticate users |

This simulates a real-world scenario: "TechCorp manages employee identities, and a partner app trusts TechCorp to authenticate users."

---

## Key SAML2 Concepts

| Term | Meaning |
|------|---------|
| **IdP** | The authority that authenticates users and issues assertions |
| **SP** | The application that trusts the IdP and consumes assertions |
| **Assertion** | XML document with authentication/attribute statements |
| **Circle of Trust (CoT)** | A trust relationship grouping IdP + SP(s) |
| **Metadata** | XML describing endpoints, certificates, bindings for IdP/SP |
| **Binding** | How SAML messages are transported (POST, Redirect, Artifact) |
| **NameID** | The identifier for the user shared between IdP and SP |

---

## Step 1: Create the Hosted IdP in /techcorp Realm

1. Go to **http://pingam:8081/am/console**
2. Log in as `amadmin / Passw0rd123`
3. Select the **techcorp** realm (top-left realm dropdown)
4. Navigate to **Applications → Federation → Entity Providers**
5. Click **Add Entity Provider**
6. Select **Hosted** tab
7. Fill in:
   - **Entity Id**: `techcorp-idp`
   - **Entity Provider Base URL**: `http://pingam:8081/am`
   - Check **IdP** role
   - **Meta Alias (IdP)**: `/techcorp-idp`
8. Click **Create**

Once created, AM auto-generates the IdP metadata with endpoints, signing certificates, etc.

---

## Step 2: Create the Hosted SP in Root Realm

1. Switch to the **Top Level Realm** (root `/`) using the realm dropdown
2. Navigate to **Applications → Federation → Entity Providers**
3. Click **Add Entity Provider**
4. Select **Hosted** tab
5. Fill in:
   - **Entity Id**: `partner-sp`
   - **Entity Provider Base URL**: `http://pingam:8081/am`
   - Check **SP** role
   - **Meta Alias (SP)**: `/partner-sp`
6. Click **Create**

---

## Step 3: Create a Circle of Trust in Both Realms

A Circle of Trust (CoT) links the IdP and SP so they trust each other.

### In the /techcorp realm:
1. Switch to **techcorp** realm
2. Go to **Applications → Federation → Circle of Trust**
3. Click **Add Circle of Trust**
4. Name: `techcorp-cot`
5. After creation, click on `techcorp-cot` to edit it
6. In the **Entity Providers** section, add `techcorp-idp` to the CoT
7. Also add `partner-sp` (it should be available as a remote entity — if not, we'll handle this in the next step)
8. Save

### In the Root realm:
1. Switch to **Top Level Realm**
2. Go to **Applications → Federation → Circle of Trust**
3. Click **Add Circle of Trust**
4. Name: `techcorp-cot` (same name for clarity)
5. Add `partner-sp` to the CoT
6. Also add `techcorp-idp`
7. Save

---

## Step 4: Register Remote Entities (Cross-Realm Trust)

Since IdP and SP are in different realms, each realm needs to know about the other's entity via its **metadata URL**.

### Register the IdP as a Remote Entity in Root Realm:
1. Stay in **Top Level Realm**
2. Go to **Applications → Federation → Entity Providers**
3. Click **Add Entity Provider**
4. Select **Remote** tab
5. For the metadata URL, enter:
   ```
   http://pingam:8081/am/saml2/jsp/exportmetadata.jsp?entityid=techcorp-idp&realm=/techcorp
   ```
6. Click **Create**
7. After import, go to **Circle of Trust → techcorp-cot** and make sure `techcorp-idp` is added

### Register the SP as a Remote Entity in /techcorp Realm:
1. Switch to **techcorp** realm
2. Go to **Applications → Federation → Entity Providers**
3. Click **Add Entity Provider**
4. Select **Remote** tab
5. For the metadata URL, enter:
   ```
   http://pingam:8081/am/saml2/jsp/exportmetadata.jsp?entityid=partner-sp&realm=/
   ```
6. Click **Create**
7. Go to **Circle of Trust → techcorp-cot** and ensure `partner-sp` is added

---

## Step 5: Test SP-Initiated SSO

This is the most common flow: user visits the SP, gets redirected to the IdP to log in, then returns to the SP with a SAML assertion.

Open this URL in your browser:

```
http://pingam:8081/am/saml2/jsp/spSSOInit.jsp?metaAlias=/partner-sp&idpEntityID=techcorp-idp&binding=urn:oasis:names:tc:SAML:2.0:bindings:HTTP-POST
```

**What should happen**:
1. AM (as SP) redirects you to the IdP login page
2. You log in as `demo` (or any techcorp user)
3. IdP creates a SAML assertion and POSTs it back to the SP
4. SP validates the assertion and creates a local session
5. You see a federation success page

---

## Step 6: Test IdP-Initiated SSO

This flow starts at the IdP side:

```
http://pingam:8081/am/saml2/jsp/idpSSOInit.jsp?metaAlias=/techcorp/techcorp-idp&spEntityID=partner-sp&binding=urn:oasis:names:tc:SAML:2.0:bindings:HTTP-POST
```

**What should happen**:
1. If not already logged in, IdP prompts for credentials
2. IdP generates a SAML assertion
3. POSTs it to the SP's Assertion Consumer Service (ACS)
4. SP creates session

---

## Step 7: Examine the SAML Metadata

View the auto-generated metadata to understand what AM exposes:

**IdP Metadata**:
```
http://pingam:8081/am/saml2/jsp/exportmetadata.jsp?entityid=techcorp-idp&realm=/techcorp
```

**SP Metadata**:
```
http://pingam:8081/am/saml2/jsp/exportmetadata.jsp?entityid=partner-sp&realm=/
```

Look for these key elements in the XML:
- `SingleSignOnService` — IdP endpoints (POST and Redirect bindings)
- `AssertionConsumerService` — SP endpoint that receives assertions
- `SingleLogoutService` — SLO endpoints
- `X509Certificate` — Signing/encryption certificates
- `NameIDFormat` — Supported NameID formats

---

## Real-World SP Architecture

### The 3 Architectures - When Does the SP Side Need an Access Manager?

**Architecture 1: App Has Built-In SAML SP (Most Common - Salesforce, AWS, etc.)**

```
    TechCorp (Your Org)              Salesforce / AWS / Udemy
┌─────────────────────┐         ┌──────────────────────────────┐
│  ┌──────────┐       │  SAML   │  ┌────────────────────────┐  │
│  │ PingAM   │───────│────────→│  │  App Server            │  │
│  │ (IdP)    │       │ assertion│  │  ┌──────────────────┐  │  │
│  └──────────┘       │  via     │  │  │ Built-in SAML SP │  │  │
│       │             │ browser  │  │  │ (part of app     │  │  │
│  ┌──────────┐       │         │  │  │  code itself)    │  │  │
│  │   DS     │       │         │  │  └──────────────────┘  │  │
│  │ (Users)  │       │         │  └────────────────────────┘  │
│  └──────────┘       │         │  NO AM/Okta needed here!     │
└─────────────────────┘         └──────────────────────────────┘
```

The app's developers wrote SAML SP logic directly into the product. You just configure:
- IdP Entity ID, Login URL, Signing Certificate
- NameID mapping (which SAML attribute = app username)

| Application | SP Built-In? | Configuration Location |
|---|---|---|
| Salesforce | Yes | Setup → Identity → Single Sign-On Settings |
| AWS Console | Yes | IAM → Identity Providers |
| Google Workspace | Yes | Admin → Security → SSO with third-party IdP |
| ServiceNow | Yes | System Properties → SAML 2.0 |
| Jira/Confluence | Yes | Authentication → SAML |
| Custom Web App | No | Need a SAML library or AM/PingFederate as SP |

---

**Architecture 2: App Has NO SAML Support → AM/Okta Acts as SP Gateway (Our Lab)**

```
    TechCorp (IdP Side)              Partner Org (SP Side)
┌─────────────────────┐         ┌───────────────────────────────┐
│  ┌──────────┐       │  SAML   │  ┌──────────┐   ┌──────────┐ │
│  │ PingAM   │───────│────────→│  │ PingAM   │──→│ Custom   │ │
│  │ (IdP)    │       │ assertion│  │ (as SP)  │   │ App      │ │
│  │ /techcorp│       │         │  │ root /   │   │ (no SAML │ │
│  └──────────┘       │         │  └──────────┘   │ support) │ │
│       │             │         │  Receives SAML, │ port 3000│ │
│  ┌──────────┐       │         │  creates local  └──────────┘ │
│  │   DS     │       │         │  AM session,                 │
│  │ (Users)  │       │         │  app validates               │
│  └──────────┘       │         │  via REST API                │
└─────────────────────┘         └───────────────────────────────┘
```

The partner org uses AM/Okta/PingFederate as a **federation gateway** because their custom app doesn't speak SAML. AM receives the assertion, validates it, creates a session, and the app validates that session via REST API (or Web Agent / PingGateway in production).

**This is what our lab demonstrates** with the sample-app on port 3000.

---

**Architecture 3: Centralized Federation Hub**

```
    TechCorp (IdP)                   Partner Org (Federation Hub)
┌─────────────────────┐         ┌────────────────────────────────┐
│  ┌──────────┐       │  SAML   │  ┌──────────┐                  │
│  │ PingAM   │───────│────────→│  │ PingAM   │   ┌────┐ ┌────┐ │
│  │ (IdP)    │       │         │  │ (SP +    │──→│App1│ │App2│ │
│  └──────────┘       │         │  │  Gateway)│──→└────┘ └────┘ │
│                     │         │  └──────────┘   ┌────┐        │
│                     │         │       │──────────→App3│        │
│                     │         │                  └────┘        │
└─────────────────────┘         └────────────────────────────────┘
```

Large enterprises use this to federate with **many external IdPs** and route users to **many internal apps**. AM acts as both SP (for external IdPs) and session/token provider (for internal apps).

---

**How Production Apps Get the Session (Architecture 2)**

In our lab, the AM session cookie (`iPlanetDirectoryPro`) is set on the `pingam` domain, but the app runs on `localhost`. In production, this cross-domain problem is solved by:

| Solution | How It Works |
|---|---|
| **AM Web Agent** | Apache/Tomcat plugin installed on the app server. Same domain, reads cookie directly. Intercepts requests, validates session, passes user info via HTTP headers. |
| **PingGateway (IG)** | Reverse proxy in front of the app. Validates AM session and injects user info headers before forwarding to the app. |
| **Same domain** | App and AM share a cookie domain (e.g., `*.example.com`). Cookie is accessible to both. |
| **Token exchange** | AM redirects with a one-time token. App exchanges it server-side for user info. |

In our lab, we simulate this by having the user paste the session token manually (from browser DevTools), which the app then validates via AM's REST API.

---

## Lab Setup - Our Architecture 2 Implementation

```
┌─────────────────────────────────────────────────────────┐
│                    Docker Network (fr-net)               │
│                                                          │
│  ┌──────────────────────────────────────────────────┐   │
│  │                    PingAM                         │   │
│  │                                                    │   │
│  │   /techcorp realm          root / realm            │   │
│  │   ┌──────────┐            ┌──────────┐            │   │
│  │   │   IdP    │───SAML────→│    SP     │            │   │
│  │   │ techcorp │  assertion │ partner  │            │   │
│  │   │   -idp   │            │   -sp    │            │   │
│  │   └──────────┘            └────┬─────┘            │   │
│  │        ↑                       │ AM session        │   │
│  │        │ auth                  │ (iPlanetDirectory │   │
│  │   ┌────┴─────┐                │  Pro cookie)      │   │
│  │   │  PingDS  │                ↓                    │   │
│  │   │ (Users)  │         ┌──────────┐               │   │
│  │   └──────────┘         │sample-app│               │   │
│  │                        │ port 3000│               │   │
│  │                        │ validates│               │   │
│  │                        │ session  │               │   │
│  │                        │ via REST │               │   │
│  │                        └──────────┘               │   │
│  └──────────────────────────────────────────────────┘   │
│                                                          │
│  Ports exposed to host:                                  │
│    pingam:8081  (AM Console + SAML endpoints)            │
│    sample-app:3000  (Partner Corp Sample App)            │
└─────────────────────────────────────────────────────────┘
```

### Start the sample app:
```bash
docker compose up -d sample-app
```

### Access the sample app:
```
http://localhost:3000
```

---

*Created: 2026-01-30 - Lab 4 SAML2 Federation*
