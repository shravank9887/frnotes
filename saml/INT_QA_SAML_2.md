# ForgeRock/PingAM Interview Q&A — SAML2 Federation (Part 2)

## Index

| # | Question | Topic |
|---|----------|-------|
| Q1 | Why do both IdP and SP metadata have SLO URLs? | SLO Bidirectionality |
| Q2 | What is NameID? | NameID Basics |
| Q3 | NameID Formats — when to use which? | NameID Formats |
| Q4 | How does the IdP produce the NameID value? | NameID Value Map |
| Q5 | How does the SP consume the NameID? | Account Mapper |
| Q6 | How do IdP and SP negotiate the NameID format? | NameID Negotiation |
| Q7 | What are ALL the SAML2 NameID formats? | Complete NameID Format List |

---

## Q1: Why do both IdP and SP metadata have SLO (SingleLogoutService) URLs?

**Answer**: Because **either side can initiate logout**, and the other side needs to know where to send the `LogoutRequest` or `LogoutResponse`.

**Scenario 1 — User logs out at the SP**:
```
User clicks "Logout" on SP app
  → SP sends LogoutRequest to IdP's SLO endpoint (from IdP metadata)
  → IdP terminates session, notifies other SPs
  → IdP sends LogoutResponse back to SP's SLO endpoint (from SP metadata)
```

**Scenario 2 — User logs out at the IdP**:
```
User clicks "Logout" on IdP portal
  → IdP sends LogoutRequest to every SP's SLO endpoint (from SP metadata)
  → Each SP destroys local session
  → Each SP sends LogoutResponse to IdP's SLO endpoint (from IdP metadata)
```

**Summary**:

| Who initiates logout | Sends LogoutRequest to | Uses SLO URL from |
|---|---|---|
| SP | IdP | **IdP metadata** |
| IdP | SP | **SP metadata** |
| Either side | Sends LogoutResponse back | **The other side's metadata** |

**Interview answer**: "SLO is bidirectional — logout can be triggered from either side. The SP needs the IdP's SLO URL to send LogoutRequests, and the IdP needs the SP's SLO URL to propagate logout to all federated applications. Both sides also need each other's SLO URL to send back LogoutResponses. That's why both metadata documents publish SLO endpoints."

---

## Q2: What is NameID?

**Answer**: NameID is the **user identifier inside a SAML assertion**. It tells the SP who authenticated at the IdP.

```xml
<saml:NameID Format="urn:oasis:names:tc:SAML:1.1:nameid-format:emailAddress">
  demo@example.com
</saml:NameID>
```

It has two parts:
- **Format** — what type of identifier (email, persistent, transient, unspecified)
- **Value** — the actual identifier string

The IdP produces it. The SP consumes it to find or create a local user.

---

## Q3: NameID Formats — when to use which?

| Format | Value looks like | Use case |
|---|---|---|
| `persistent` | `3f7a9b2c-8d1e-4f2a...` (opaque) | Long-term federation, privacy-preserving |
| `transient` | Random per-session | Anonymous access, no permanent link |
| `unspecified` | Whatever you want (`demo`, `jsmith`) | Simple setups, shared identity stores |
| `emailAddress` | `user@example.com` | SaaS apps (Salesforce, Google Workspace) |

**Key distinctions**:
- `persistent` — IdP generates a random ID and stores a mapping. Same user always gets the same ID for the same SP, but different SPs see different IDs. Privacy-preserving.
- `transient` — New random ID every session. SP cannot correlate sessions. Used for anonymous/pseudonymous access.
- `unspecified` — IdP sends a real user attribute. Simplest but least privacy.
- `emailAddress` — Most common for SaaS integrations. Easy for SP to match users.

**Interview answer**: "We use `persistent` for privacy-sensitive federations, `transient` for anonymous access, `emailAddress` for SaaS integrations like Salesforce, and `unspecified` when IdP and SP share an identity store or have a known attribute mapping."

---

## Q4: How does the IdP produce the NameID value?

**Answer**: Configured on the **Hosted IdP → Assertion Content → NameID Value Map**.

This maps format to a user attribute:
```
unspecified  = cn     → reads user's cn     → sends "Demo User"
emailAddress = mail   → reads user's mail   → sends "demo@example.com"
persistent   = (auto) → generates opaque ID, stores mapping internally
```

**PingAM gotcha**: Don't use `uid` in the Value Map. AM's identity API treats `uid` as the identity name (lookup key), not a readable profile attribute. Use `cn`, `mail`, or other LDAP attributes instead.

**If the attribute is missing**: AM throws `Unable to generate NameID value` in the Federation debug log. The user profile must have the mapped attribute populated.

**Interview answer**: "The NameID Value Map on the hosted IdP maps each NameID format to a user profile attribute. For example, `emailAddress=mail` tells the IdP to read the user's `mail` attribute and put it in the assertion. In PingAM, avoid mapping to `uid` — it's a special identity key, not a readable attribute."

---

## Q5: How does the SP consume the NameID?

**Answer**: Configured on the **Hosted SP → Assertion Processing → Account Mapper**.

The SP receives the NameID and must map it to a local user:

| SP Setting | What it does |
|---|---|
| **Use Name ID as User ID** = ON | Treats NameID value directly as `uid` for local user lookup |
| **Account Mapper class** | Custom Java class for complex mapping logic |
| **Auto Federation** | Auto-creates/links local account if no match found |

**Example flow**:
```
IdP sends NameID "demo" (format: unspecified)
  → SP Account Mapper looks up uid=demo in local identity store
  → Found → creates local session for demo
  → Not found → error "No local user being mapped"
```

**Auto Federation** (if enabled): When no local user matches, SP can automatically create a federated link using a specified user attribute instead of failing.

**Interview answer**: "The SP's Account Mapper translates the incoming NameID to a local user. The simplest configuration is 'Use Name ID as User ID' which does a direct uid lookup. For complex scenarios, you use a custom Account Mapper class or enable Auto Federation to handle first-time users."

---

## Q6: How do IdP and SP negotiate the NameID format?

**Answer**: The SP declares preferred formats, the IdP picks the best match.

**Negotiation flow**:
```
1. SP metadata lists supported NameID formats (ordered by preference)
2. SP can also override via NameID Format List (Assertion Processing tab)
3. IdP checks its own supported formats
4. IdP picks the first SP-preferred format that it also supports
5. IdP generates the value using its NameID Value Map for that format
```

**Example**:
```
SP prefers:   1. emailAddress  2. unspecified
IdP supports: persistent, transient, unspecified, emailAddress
Result: emailAddress (first SP preference that IdP supports)
```

**Override on SP**: The `NameID Format List` on the hosted SP (Assertion Processing tab) takes priority over the formats listed in SP metadata.

**Override on IdP**: If the IdP's NameID Value Map only has entries for certain formats, it can only produce those — even if the SP requests something else.

**Interview answer**: "The SP advertises preferred NameID formats in its metadata or via the NameID Format List setting. The IdP picks the first match it can actually produce based on its NameID Value Map configuration. If there's a mismatch — the SP requests a format the IdP doesn't have a mapping for — the assertion generation fails."

---

## Q7: What are ALL the SAML2 NameID formats?

**Answer**: The SAML specification defines **8 standard NameID formats** — 3 from SAML 2.0 and 5 inherited from SAML 1.1.

### SAML 2.0 Formats

| Format URN | Use Case | Example Value |
|---|---|---|
| `urn:oasis:names:tc:SAML:2.0:nameid-format:persistent` | Long-term, privacy-preserving federation | `3f7a9b2c-8d1e-4f2a...` |
| `urn:oasis:names:tc:SAML:2.0:nameid-format:transient` | Anonymous, session-only | `_47f9a2bc-8d1e...` |
| `urn:oasis:names:tc:SAML:2.0:nameid-format:kerberos` | Kerberos principal name | `jsmith@CORP.EXAMPLE.COM` |

### SAML 1.1 Formats (still supported in SAML 2.0)

| Format URN | Use Case | Example Value |
|---|---|---|
| `urn:oasis:names:tc:SAML:1.1:nameid-format:emailAddress` | Email-based federation | `demo@example.com` |
| `urn:oasis:names:tc:SAML:1.1:nameid-format:unspecified` | Any format, simple setups | `demo`, `jsmith` |
| `urn:oasis:names:tc:SAML:1.1:nameid-format:X509SubjectName` | X.509 certificate subject DN | `CN=John Smith,OU=IT,O=TechCorp,C=US` |
| `urn:oasis:names:tc:SAML:1.1:nameid-format:WindowsDomainQualifiedName` | Windows Active Directory | `CORP\jsmith` |
| `urn:oasis:names:tc:SAML:1.0:nameid-format:unspecified` | Legacy SAML 1.0 (default) | Same as 1.1 unspecified |

### Special Format

| Format URN | Meaning |
|---|---|
| `urn:oasis:names:tc:SAML:2.0:nameid-format:encrypted` | **Not a NameID format itself** — signals the NameID must be encrypted in the assertion |

### Key Points

**Most common in production**:
- `persistent` — enterprise B2B federations
- `emailAddress` — SaaS integrations (Salesforce, Google Workspace)
- `transient` — anonymous access
- `unspecified` — simple internal setups

**Rarely used**:
- `kerberos` — when federating with Kerberos realms
- `X509SubjectName` — when using certificate-based authentication
- `WindowsDomainQualifiedName` — legacy Active Directory integrations (modern AD uses `emailAddress` or `persistent`)

**Default behavior**: If no Format is specified in the assertion, the default is `urn:oasis:names:tc:SAML:1.0:nameid-format:unspecified`.

**PingAM support**: PingAM supports all 8 formats, but not all may be configured by default in the NameID Value Map. You must explicitly map each format to a user attribute (or auto-generation logic for `persistent`/`transient`).

**Encryption**: The `encrypted` format is a flag, not a standalone format. It tells the SP that the NameID will be encrypted using the SP's encryption certificate. The actual format (`persistent`, `emailAddress`, etc.) is encrypted inside an `<EncryptedID>` element.

**Interview answer**: "SAML 2.0 defines 8 standard NameID formats: 3 native SAML 2.0 formats (`persistent`, `transient`, `kerberos`) and 5 inherited from SAML 1.1 (`emailAddress`, `unspecified`, `X509SubjectName`, `WindowsDomainQualifiedName`, plus the SAML 1.0 unspecified variant). The most common in production are `persistent` for privacy-preserving B2B federation, `emailAddress` for SaaS integrations, `transient` for anonymous access, and `unspecified` for simple internal setups. There's also an `encrypted` format flag that signals the NameID should be encrypted, but it's not a format itself — it wraps another format inside an `<EncryptedID>` element."

---

*Last Updated: 2026-02-04*
