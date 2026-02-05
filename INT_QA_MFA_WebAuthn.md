# Interview Q&A: MFA, OATH TOTP, WebAuthn & FIDO2

## OATH / TOTP

### Q1: What is OATH and how does it relate to TOTP/HOTP?

**OATH** (Initiative for Open Authentication) is an industry collaboration that defines open standards for strong authentication. It specifies two OTP algorithms:

- **TOTP** (Time-based One-Time Password, RFC 6238): Generates a code based on a shared secret + current time. Code changes every 30 seconds (configurable). Used by Google Authenticator, ForgeRock Authenticator, Authy, etc.
- **HOTP** (HMAC-based One-Time Password, RFC 4226): Generates a code based on a shared secret + a counter. Counter increments on each use. Less common because server/client counters can desync.

Both use HMAC-SHA1 (or SHA-256/SHA-512) to derive a 6-8 digit code from the secret.

**HMAC** = **Hash-based Message Authentication Code**. It combines a cryptographic hash function (like SHA-1 or SHA-256) with a secret key to produce a message authentication code. In OATH TOTP, HMAC-SHA1 takes the shared secret + current time counter as inputs and produces the 6-digit code.

### Q2: How does OATH TOTP work in PingAM?

Three components are needed:

1. **ForgeRock Authenticator (OATH) Service** — Realm-level service that enables OATH device profiles. Configures algorithm (TOTP/HOTP), code length, time step, skew tolerance, and maximum retry attempts.
2. **OATH Registration node** — Displays a QR code containing `otpauth://` URI. User scans with an authenticator app. The shared secret is stored encrypted in the user's AM profile (`oathDeviceProfiles` attribute).
3. **OATH Token Verifier node** — Prompts user for a 6-digit code and validates it against the shared secret + current time.

**Tree wiring**: Registration Success must connect to Token Verifier, not directly to tree Success. Otherwise the user registers a device but never proves possession of it.

### Q3: What is the `otpauth://` URI format in the QR code?

```
otpauth://totp/Issuer:username?secret=BASE32SECRET&issuer=Issuer&algorithm=SHA1&digits=6&period=30
```

- `totp` — algorithm type
- `Issuer:username` — label shown in authenticator app
- `secret` — Base32-encoded shared secret
- `issuer` — organization name
- `algorithm` — HMAC algorithm (SHA1, SHA256, SHA512)
- `digits` — code length (6 or 8)
- `period` — time step in seconds (default 30)

### Q4: What is OATH "skew" tolerance?

Clock skew accounts for time drift between server and client device. A skew of 1 means the server accepts codes from the previous, current, and next time step (90-second window for a 30-second step). Higher skew = more tolerant but less secure.

---

## WebAuthn

### Q5: What is WebAuthn?

**WebAuthn** (Web Authentication API) is a W3C standard that defines a JavaScript API for browsers to interact with authenticators (hardware keys, biometric sensors, platform authenticators). It enables:

- **Passwordless login** — authenticate with fingerprint, face, or security key instead of password
- **Second-factor authentication** — use a security key or biometric as MFA after password
- **Phishing resistance** — credentials are bound to the origin (domain), so they cannot be phished

WebAuthn uses **public-key cryptography**: the authenticator generates a key pair, stores the private key, and sends the public key to the server during registration. Authentication involves signing a challenge with the private key.

### Q6: What is FIDO2? How does it relate to WebAuthn?

**FIDO2** is an umbrella standard from the FIDO Alliance consisting of two components:

| Component | What it is | Scope |
|-----------|-----------|-------|
| **WebAuthn** | W3C browser JavaScript API | Browser ↔ Relying Party (server) |
| **CTAP** (Client to Authenticator Protocol) | FIDO Alliance protocol | Browser ↔ Authenticator (hardware) |

So: **FIDO2 = WebAuthn + CTAP**

- WebAuthn defines how the browser talks to the server (registration ceremonies, authentication ceremonies, credential management)
- CTAP defines how the browser talks to the authenticator device (USB, NFC, BLE, internal platform authenticator)

They are **not the same thing** — WebAuthn is one half of FIDO2. You can use WebAuthn without caring about CTAP (the browser abstracts it), but FIDO2 encompasses both.

### Q7: What is FIDO U2F and how does it relate to FIDO2?

**FIDO U2F** (Universal 2nd Factor) was the predecessor standard:

| | FIDO U2F | FIDO2/WebAuthn |
|--|----------|----------------|
| **Spec** | FIDO Alliance only | W3C + FIDO Alliance |
| **Use case** | Second factor only | Passwordless + second factor + MFA |
| **Browser API** | Separate U2F API (deprecated) | WebAuthn API (standard) |
| **Authenticator** | Hardware key required | Hardware key, platform biometric, or hybrid |
| **Credential type** | Single key per origin | Discoverable (resident) keys possible |

FIDO2 is backwards-compatible with U2F — existing U2F security keys work with WebAuthn.

### Q8: What are "platform authenticators" vs "cross-platform authenticators"?

- **Platform authenticator** (internal): Built into the device — Windows Hello (face/fingerprint/PIN), macOS Touch ID, Android biometrics. Cannot be moved to another device.
- **Cross-platform authenticator** (roaming): External hardware — YubiKey, Titan Key, phone-as-security-key. Can be used across multiple devices via USB/NFC/BLE.

In AM's WebAuthn Registration node, you can filter which type to accept.

### Q9: What is a "relying party" in WebAuthn?

The **Relying Party (RP)** is the server/service that wants to authenticate the user. In our setup, PingAM is the Relying Party. The RP ID is typically the domain name (e.g., `localhost` or `example.com`). The browser enforces that credentials are bound to the RP ID — a credential created for `example.com` cannot be used on `evil.com`. This is why WebAuthn is **phishing-resistant**.

### Q10: Why does WebAuthn require HTTPS (secure context)?

The browser's `navigator.credentials.create()` and `navigator.credentials.get()` APIs are only available in a **secure context**:

- HTTPS on any domain
- HTTP on `localhost` (special exception for development)

This prevents:
- Man-in-the-middle attacks during registration/authentication
- Credential theft via insecure transport
- Origin spoofing

In our Docker lab, AM redirects `localhost:8081` to `pingam:8081` (AM's configured FQDN). Since `pingam` is not `localhost` and the connection is HTTP, the browser blocks WebAuthn API calls entirely.

### Q11: How does WebAuthn registration work (step by step)?

1. **Server → Browser**: Server sends a challenge (random bytes), RP info (id, name), user info (id, name), and credential creation options (authenticator type, algorithms, attestation preference)
2. **Browser → Authenticator**: Browser calls `navigator.credentials.create()`, which triggers the authenticator (e.g., fingerprint scan, key touch)
3. **Authenticator → Browser**: Authenticator generates a new key pair, signs the challenge with the private key, returns the public key + attestation + credential ID
4. **Browser → Server**: Browser sends the response (public key, attestation, credential ID, client data)
5. **Server**: Validates the attestation, stores the public key + credential ID associated with the user

### Q12: How does WebAuthn authentication work?

1. **Server → Browser**: Server sends a challenge + list of allowed credential IDs for the user
2. **Browser → Authenticator**: Browser calls `navigator.credentials.get()`, authenticator finds matching credential, user verifies (biometric/touch/PIN)
3. **Authenticator → Browser**: Authenticator signs the challenge with the private key
4. **Browser → Server**: Browser sends the signed assertion
5. **Server**: Verifies the signature using the stored public key

### Q13: What is "attestation" in WebAuthn?

Attestation is the authenticator's proof of its own identity and trustworthiness during registration:

- **None**: No attestation — server trusts any authenticator (most common for consumer apps)
- **Direct**: Authenticator sends its attestation certificate — server can verify the make/model
- **Indirect**: Attestation may be anonymized by a privacy CA
- **Enterprise**: Full device identification (for managed corporate devices)

AM supports configuring attestation preference in the WebAuthn Registration node.

---

## WebAuthn in PingAM

### Q14: What services/nodes does PingAM provide for WebAuthn?

**Service**: WebAuthn Profile Encryption Service (realm-level) — Required before using WebAuthn nodes. Configures encryption of device credential profiles stored in user identity.

**Authentication nodes**:
- **WebAuthn Registration Node** — Triggers `navigator.credentials.create()`, stores the public key credential in the user's profile. Outcomes: Success, Failure, Client Error, Unsupported.
- **WebAuthn Authentication Node** — Triggers `navigator.credentials.get()`, verifies the signed challenge. Outcomes: Success, Failure, Client Error, Unsupported.
- **WebAuthn Device Storage Node** — Optional node to persist device metadata after registration.

### Q15: What is the typical MFA tree structure using WebAuthn?

```
Start → Data Store Decision → [check if device registered]
                                    |
                    ┌────────────────┴────────────────┐
                    ↓ (no device)                      ↓ (has device)
          WebAuthn Registration              WebAuthn Authentication
                    |                                  |
                    ↓ Success                          ↓ Success
               Success                            Success
```

In practice, you can combine with OATH or add fallback paths.

### Q16: What are the WebAuthn node outcomes and when do they fire?

| Outcome | When |
|---------|------|
| **Success** | User completed registration/authentication successfully |
| **Failure** | User cancelled, timed out, or verification failed |
| **Client Error** | JavaScript error in browser (often: not a secure context, API not available) |
| **Unsupported** | Browser does not support WebAuthn API at all |

In our lab, we hit **Client Error** because AM redirected to `pingam:8081` (non-secure context).

---

## General MFA Concepts

### Q17: What is MFA and why is it important?

**Multi-Factor Authentication** requires two or more independent factors:

| Factor | Category | Examples |
|--------|----------|----------|
| Something you **know** | Knowledge | Password, PIN, security question |
| Something you **have** | Possession | Phone (TOTP app), security key, smart card |
| Something you **are** | Inherence | Fingerprint, face, iris, voice |

MFA prevents account takeover when one factor is compromised (e.g., password stolen via phishing, but attacker doesn't have the security key).

### Q18: Compare OATH TOTP vs WebAuthn/FIDO2 for MFA.

| Aspect | OATH TOTP | WebAuthn/FIDO2 |
|--------|-----------|----------------|
| **Factor type** | Possession (phone with app) | Possession + Inherence (key + biometric) |
| **Phishing resistance** | No — user can be tricked into entering code on fake site | Yes — credentials bound to origin |
| **Shared secret** | Yes — server and client both know the secret | No — asymmetric keys, server only has public key |
| **Server breach risk** | High — leaked secrets allow code generation | Low — public keys are useless without private key |
| **User experience** | Open app, read code, type 6 digits | Touch key or scan fingerprint |
| **Offline support** | Yes — TOTP works offline | Depends — platform authenticator yes, some hybrid no |
| **Setup complexity** | Low — scan QR code | Medium — requires secure context, browser support |
| **Cost** | Free (any TOTP app) | Free (platform) or $25-50 (hardware key) |

### Q19: What is "step-up authentication" and how does it relate to MFA?

Step-up authentication increases the authentication level when accessing sensitive resources. For example:

1. User logs in with password → auth level 5 → can browse dashboard
2. User tries to transfer money → policy requires auth level 10 → AM prompts for TOTP
3. User enters TOTP → auth level 10 → transfer proceeds

In AM, this is implemented using:
- **Authentication Level** on each tree/chain (set in tree properties)
- **AuthLevel environment condition** on policies
- **Transactional Authorization** trees (re-authenticate for sensitive actions)

### Q20: What is "passwordless" authentication and how does FIDO2 enable it?

**Passwordless** means the user never types a password — they authenticate purely through a cryptographic credential + biometric or PIN on their device.

#### Why passwords are the problem

Passwords are the #1 attack vector: phishing, credential stuffing, brute force, password reuse. Even with MFA (password + TOTP), the password is still phishable. Passwordless eliminates the password entirely.

#### How FIDO2 makes passwordless work — the practical flow

**One-time registration** (user enrolls their device):
1. User logs in with existing credentials (password, one-time link, admin enrollment)
2. AM sends a registration challenge to the browser
3. Browser calls `navigator.credentials.create()` → OS prompts for fingerprint/face/PIN
4. The authenticator (Windows Hello, Touch ID, YubiKey) generates a **key pair**:
   - **Private key** — stays on the device, never leaves, never transmitted
   - **Public key** — sent to AM and stored in the user's profile
5. Crucially, the authenticator stores the credential **with the user ID** — this is a **discoverable credential** (also called "resident key"). The device remembers "this key belongs to user X on site Y"

**Every subsequent login** (no password, no username):
1. User navigates to login page — **no username field**, just a "Sign in" button
2. AM sends an authentication challenge (random bytes) — with **no user hint** (it doesn't know who's logging in yet)
3. Browser calls `navigator.credentials.get()` → OS shows a picker: "Which account?" (lists all discoverable credentials for this site)
4. User selects their account → fingerprint/face/PIN verification
5. Authenticator signs the challenge with the private key → sends signed assertion + credential ID + **user handle** back to AM
6. AM looks up the public key by credential ID, verifies the signature, reads the user handle to identify who just authenticated
7. Session created — user is in

**The key insight**: In traditional WebAuthn MFA, the server already knows who the user is (they typed their username/password). It sends the credential IDs it has on file and says "prove you own one of these." In **passwordless**, the server doesn't know who's logging in — the **authenticator tells the server** who the user is via the discoverable credential. That's the fundamental difference.

#### How this looks in PingAM authentication trees

**Traditional MFA tree** (password + WebAuthn):
```
Start → Page Node (Username + Password) → Data Store Decision → WebAuthn Authentication → Success
```
AM knows the user after Data Store Decision, so it sends that user's registered credential IDs to the WebAuthn node.

**Passwordless tree** (no password at all):
```
Start → WebAuthn Authentication Node → Success
```
No username collection, no Data Store Decision. The WebAuthn Authentication node is configured with `requireResidentKey: true`. The authenticator provides the user identity through the discoverable credential. AM resolves the user from the returned `userHandle`.

#### Real-world deployment example

*"In our environment, we rolled out passwordless for internal employees accessing our SSO portal. Here's what we did:*

1. *First phase — we deployed FIDO2 as MFA alongside passwords. Every employee registered their Windows Hello (laptop fingerprint/face) or a YubiKey through a self-service registration tree. The tree was: Password Login → check if WebAuthn device registered → if no, prompt registration → if yes, WebAuthn verify → Success.*

2. *Second phase — once 90%+ adoption, we switched the default tree to passwordless. The login page has no username/password fields — just "Sign in with your device." Employees touch their fingerprint reader or tap their YubiKey. For the 10% who hadn't registered, we kept a fallback link: "Sign in with password instead" that routes to the old tree.*

3. *For contractors/BYOD users who can't use platform authenticators, we issued YubiKey 5 NFC keys as cross-platform authenticators. They tap the key on USB or NFC.*

4. *Recovery flow: if someone loses their device, helpdesk verifies identity out-of-band, then issues a one-time magic link that lets them register a new device. We built this as a separate AM tree: Magic Link verification → WebAuthn Registration → Success."*

#### Why passwordless is phishing-proof

With TOTP MFA, an attacker can set up a fake login page at `evil-bank.com`, proxy the real `bank.com` login, and relay the TOTP code in real-time (real-time phishing proxy attack).

With FIDO2 passwordless, this fails because:
- The credential is **bound to the origin** (`bank.com`). The browser sends the origin as part of the signed data.
- When the user is on `evil-bank.com`, the browser tells the authenticator "origin is evil-bank.com"
- The authenticator has no credential for `evil-bank.com` — authentication fails
- Even if the attacker somehow gets a signed assertion, it's signed for `evil-bank.com`, not `bank.com` — the real server rejects it

**No shared secrets, no codes to type, no relay attacks possible.**

### Q21: What is "passkey" and how does it differ from traditional WebAuthn?

**Passkeys** are discoverable FIDO2 credentials that sync across devices via cloud (iCloud Keychain, Google Password Manager, etc.):

| | Traditional WebAuthn | Passkeys |
|--|---------------------|----------|
| **Storage** | Single device (hardware key or TPM) | Synced across devices via cloud |
| **Recovery** | Lost key = locked out (need backup) | Recover via cloud account |
| **Portability** | Need physical key on each device | Available on all synced devices |
| **Security model** | Hardware-bound (highest assurance) | Cloud-synced (convenient but cloud account = single point of failure) |

AM 7.3+ / PingAM 8.0 supports passkeys through the standard WebAuthn nodes — the distinction is handled by the authenticator/platform, not by AM.

### Q22: In PingAM, how are MFA device profiles stored?

Device profiles (OATH secrets, WebAuthn public keys, Push registration data) are stored as **multi-valued attributes on the user's identity entry in DS**:

- `oathDeviceProfiles` — OATH device registrations (encrypted JSON)
- `webauthnDeviceProfiles` — WebAuthn credential public keys (encrypted JSON)
- `pushDeviceProfiles` — Push notification device registrations (encrypted JSON)

The **WebAuthn Profile Encryption Service** configures the encryption keys used for these stored profiles. Without it, WebAuthn nodes fail because they can't encrypt/decrypt stored credentials.

### Q23: Can you combine multiple MFA methods in a single tree?

Yes. Common patterns:

1. **MFA with choice**: Page Node with Select Identity Provider-style chooser → branch to OATH or WebAuthn path
2. **MFA with fallback**: Try WebAuthn → Unsupported/Client Error → fall back to OATH TOTP
3. **Progressive MFA**: First login = OATH Registration → subsequent logins = OATH Verify. Scripted Decision checks if user has registered device.
4. **Step-up MFA**: Separate tree for step-up that only does the MFA verification (no primary auth)

---

## Practical Insights from Lab 8

### Q24: What went wrong with WebAuthn in the Docker lab?

AM is configured with site URL `http://pingam:8081/am`. When a user accesses `http://localhost:8081/am`, AM redirects to `http://pingam:8081/am` (its canonical FQDN). This causes two problems for WebAuthn:

1. **Not localhost**: The redirect changes the origin from `localhost` to `pingam`, losing the localhost secure-context exception
2. **Not HTTPS**: The connection is plain HTTP

The browser blocks `navigator.credentials.create()` / `navigator.credentials.get()` in non-secure contexts, causing the WebAuthn node to hit the "Client Error" outcome.

**Fix options** (not implemented in lab):
- Configure AM site URL as `https://localhost:8081/am` + set up TLS in Tomcat
- Use a reverse proxy (nginx) with TLS termination in front of AM
- In production: always use HTTPS with proper certificates

### Q25: Why must OATH Registration connect to Token Verifier before Success?

If Registration → Success directly, the flow is:

1. User sees QR code
2. User scans QR code (or not!) → clicks "next"
3. Tree succeeds → user is logged in

The user could skip scanning the QR code entirely and still log in. The OATH secret is stored but never verified. On next login, the user may not have the code.

Correct wiring: Registration Success → Token Verifier → Success. This forces the user to prove they successfully set up their authenticator app by entering a valid code immediately after registration.

---

## Real-World Interview Questions: "Tell me about your FIDO2/WebAuthn experience"

### Q26: "How did you implement FIDO2/WebAuthn in your organization?"

*Sample answer:*

"We used PingAM (ForgeRock AM) as our identity provider. The implementation had three phases:

**Phase 1 — MFA rollout**: We added WebAuthn as a second factor to our existing password-based authentication tree. The tree flow was: username/password → Data Store Decision → check if user has a registered WebAuthn device (Scripted Decision node) → if not registered, route to WebAuthn Registration → if registered, route to WebAuthn Authentication → Success. This was non-disruptive — users who hadn't enrolled yet still logged in with just password.

**Phase 2 — Enrollment push**: We configured a Scripted Decision node that checked if the user had `webauthnDeviceProfiles` populated. If empty, it forced registration before granting access. Within 2-3 weeks, all active users had registered at least one authenticator. We encouraged registering both a platform authenticator (Windows Hello/Touch ID) and a backup security key.

**Phase 3 — Passwordless option**: We created a separate tree that starts directly with WebAuthn Authentication (no username/password collection). The node was configured with `requireResidentKey: true` and `userVerification: required`. We exposed this as a "Sign in without password" button on the login page. Users who preferred passwords still had that option.

For the AM configuration, we needed:
- WebAuthn Profile Encryption Service enabled at the realm level
- WebAuthn Registration and Authentication nodes in authentication trees
- Relying Party ID set to our domain (e.g., `sso.company.com`)
- HTTPS mandatory — WebAuthn doesn't work without a secure context
- We set attestation to `none` because we didn't need to verify the make/model of authenticators"

### Q27: "What challenges did you face deploying WebAuthn?"

*Sample answer:*

"Several practical challenges:

1. **HTTPS requirement**: Our dev/staging environments were on HTTP with internal hostnames. WebAuthn silently fails without a secure context — the browser just won't call the API. We had to set up TLS certificates even for dev environments, or use `localhost` for testing. In PingAM, the site URL must match the origin exactly, so we configured proper FQDN + TLS for each environment.

2. **Device loss and recovery**: The biggest operational concern. If a user loses their only registered device, they're locked out. We solved this by:
   - Requiring users to register at least 2 authenticators (platform + backup key)
   - Building a recovery tree: helpdesk generates a one-time link → user clicks it → verifies with security questions → registers a new device
   - For high-security accounts, we required in-person identity verification before re-enrollment

3. **Cross-browser inconsistencies**: Safari, Chrome, and Firefox handle WebAuthn prompts differently. Some versions had bugs with certain authenticator types. We had to test across browsers and document supported browser versions for our helpdesk.

4. **Corporate laptops with shared accounts**: Some departments had shared workstations. Platform authenticators (Windows Hello) are tied to the OS user profile. We issued hardware security keys (YubiKeys) for these shared environments instead.

5. **Mobile devices**: iOS and Android handle WebAuthn differently. iOS requires Safari for platform authenticator (Face ID/Touch ID). Android works in Chrome. We had to adjust our user guides per platform."

### Q28: "How does FIDO2/WebAuthn compare to TOTP in a real deployment? Why choose one over the other?"

*Sample answer:*

"We actually deployed both and here's the practical difference:

**TOTP (we used ForgeRock Authenticator OATH service)**:
- Pros: Works everywhere — any browser, HTTP or HTTPS, any device with a TOTP app. Zero infrastructure requirements beyond AM configuration. Users understand the concept ('type the code from your app').
- Cons: Phishable in real-time proxy attacks. Users occasionally lose their phone and can't access their codes. Code entry adds 10-15 seconds to every login. Shared secret stored server-side is a liability.

**WebAuthn/FIDO2**:
- Pros: Phishing-proof (origin-bound credentials). Faster UX (tap fingerprint vs open app and type 6 digits). No shared secrets on server. Can enable passwordless.
- Cons: Requires HTTPS. Doesn't work on older browsers. Device management overhead. More complex recovery flows.

**Our decision**: We used TOTP as the baseline MFA for all users (easy rollout, universal support) and offered WebAuthn as an upgrade path. For privileged accounts (admins, developers with production access), we mandated FIDO2 security keys because phishing resistance was critical. For regular employees, we offered passwordless via platform authenticators as an opt-in convenience feature.

The key insight: TOTP raises the bar (attacker needs your phone), but FIDO2 changes the game (attacker can't phish the credential at all). For most organizations, the pragmatic answer is TOTP as baseline, FIDO2 for high-value accounts, and passwordless as the long-term goal."

### Q29: "A user reports WebAuthn isn't working. How do you troubleshoot?"

*Sample answer:*

"I'd follow this checklist:

1. **Secure context check**: Is the page loaded over HTTPS? Or is it `localhost`? Open browser DevTools console — if you see errors about `navigator.credentials` being undefined, it's a secure context issue. This was our most common problem. In AM, verify the site URL matches the HTTPS FQDN.

2. **Browser support**: Check if the browser version supports WebAuthn (`navigator.credentials` exists in console). Very old browsers or restricted corporate browsers may not support it.

3. **Relying Party ID mismatch**: AM's WebAuthn node uses the RP ID derived from the site URL domain. If the user registered on `sso.company.com` but is now accessing `login.company.com`, the RP ID won't match and the authenticator won't find the credential. Check AM site configuration.

4. **Authenticator type filter**: The WebAuthn Registration node can be configured to require `platform` or `cross-platform` authenticators. If set to `platform` only and the user is trying a YubiKey, registration fails. Check the node configuration.

5. **AM debug logs**: Check `/opt/am-config/var/debug/Authentication` for WebAuthn-specific errors. Common ones: `NotAllowedError` (user denied or timed out), `InvalidStateError` (credential already registered), `SecurityError` (origin mismatch).

6. **User's device profile**: Check if the user has `webauthnDeviceProfiles` populated in their identity entry. If empty, they never successfully registered. If populated but auth fails, the stored public key may be corrupted or the credential was revoked on the authenticator side.

7. **Timeout**: WebAuthn has a configurable timeout (default 60 seconds). If the user takes too long (finding their key, positioning their finger), the ceremony times out. Increase the timeout in the node configuration if needed."

### Q30: "How would you design an authentication system that supports password, TOTP, and FIDO2 in a single PingAM tree?"

*Sample answer:*

"I'd build it as a progressive authentication tree with fallbacks:

```
Start
  → Page Node [Username + Password]
  → Data Store Decision
      → Failure → Retry/Lockout
      → Success → Scripted Decision (check MFA enrollment status)
          → 'webauthn_registered' → WebAuthn Authentication
              → Success → Tree Success
              → Client Error / Unsupported → OATH Token Verifier → Success
              → Failure → Tree Failure
          → 'oath_registered' → OATH Token Verifier
              → Success → Tree Success
              → Failure → Tree Failure
          → 'none' → MFA Registration Choice Page
              → 'webauthn' → WebAuthn Registration → WebAuthn Auth → Success
              → 'totp' → OATH Registration → OATH Token Verifier → Success
```

The Scripted Decision node reads the user's `webauthnDeviceProfiles` and `oathDeviceProfiles` attributes. If WebAuthn is registered, prefer it (better security). If WebAuthn fails due to client error (non-secure context, unsupported browser), fall back to OATH. If nothing is registered, force enrollment.

Key design decisions:
- **WebAuthn preferred over TOTP** when both are registered (phishing resistance)
- **Graceful fallback** from WebAuthn to TOTP (don't lock out users on unsupported browsers)
- **Forced enrollment** for users with no MFA (compliance requirement)
- **Registration choice** lets users pick their preferred method
- **Separate passwordless tree** available as an alternative for WebAuthn-registered users who want to skip passwords entirely"

### Q31: "What is attestation and did you use it? Why or why not?"

*Sample answer:*

"Attestation is the authenticator's proof of its own identity during WebAuthn registration. The authenticator can send a certificate chain that lets the server verify the make, model, and firmware version of the device.

**We set attestation to `none` in our deployment.** Reasons:

1. **Privacy**: Direct attestation reveals the exact device model. Some users and privacy regulations consider this a concern. `none` means the server doesn't know if it's a YubiKey 5 or a MacBook Touch ID — it just knows the public key works.

2. **Compatibility**: Some authenticators don't support attestation, or their attestation certificates aren't in public trust chains. Requiring attestation would block these devices.

3. **Our threat model didn't need it**: We trusted that if a user completed registration on a managed corporate device, the authenticator was legitimate. We weren't trying to prevent users from using specific authenticator types.

**When you WOULD use attestation**: If you need to enforce a policy like 'only FIPS-certified hardware keys' or 'only YubiKey 5 series or newer.' Government and high-security financial environments sometimes require this. You'd set attestation to `direct`, then validate the attestation certificate against a known list of approved authenticator models (using FIDO Metadata Service or a custom allowlist).

In AM, the WebAuthn Registration node has an attestation preference setting — `none`, `indirect`, `direct`, or `enterprise`."
