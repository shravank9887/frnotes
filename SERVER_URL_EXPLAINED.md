# Server URL and Cookie Domain - Quick Reference

## Configuration Wizard Values

### Server URL
**Use:** `http://pingam:8080/am` ✅
**NOT:** `http://localhost:8081/am` ❌

### Cookie Domain
**Use:** `.example.com` ✅
**NOT:** `localhost` ❌

---

## Why These Values?

### Server URL: `http://pingam:8080/am`

**Explanation:**
- **`pingam`** = Internal Docker network hostname (how containers talk to each other)
- **`8080`** = Internal container port (NOT 8081 - that's just for your browser)
- **`localhost`** would ONLY work inside the container, breaking all redirects

**How It Works:**
```
Your Browser Access:     http://localhost:8081/am
                                ↓
         Docker Port Mapping: 8081 → 8080
                                ↓
    Inside Container:        http://pingam:8080/am
                                ↓
PingAM Generates URLs:       http://pingam:8080/am
```

**PingAM Uses This URL For:**
- OAuth 2.0 redirects
- SAML callbacks
- Internal service references
- Session management
- API endpoints

---

### Cookie Domain: `.example.com`

**Explanation:**
- The leading `.` enables cookie sharing across subdomains
- Required for Single Sign-On (SSO) to work
- Allows authentication cookies to work across multiple apps

**SSO Example:**
```
User logs in:      pingam.example.com
Cookie set for:    .example.com
↓
User visits app1:  app1.example.com  ← Cookie works! (SSO success)
User visits app2:  app2.example.com  ← Cookie works! (SSO success)
```

**Why NOT `localhost`:**
- Only works for single application
- Cannot share cookies between apps
- Not production-like
- SSO will not work

---

## Browser Access

**From your web browser, use:** `http://localhost:8081/am`

Docker automatically maps:
- External (your browser): `localhost:8081`
- Internal (container): `pingam:8080`

---

## Optional: Hosts File Setup

To use `pingam` directly from your browser:

**Windows:** Edit `C:\Windows\System32\drivers\etc\hosts`
**Linux/Mac:** Edit `/etc/hosts`

**Add these lines:**
```
127.0.0.1  pingam
127.0.0.1  pingam.example.com
```

**Then you can access via:**
- `http://localhost:8081/am` ✅
- `http://pingam:8081/am` ✅
- `http://pingam.example.com:8081/am` ✅

---

## Quick Reference Table

| What You're Configuring | Hostname | Port | Full Value |
|-------------------------|----------|------|------------|
| **Server URL (Wizard)** | `pingam` | `8080` | `http://pingam:8080/am` |
| **Cookie Domain (Wizard)** | `.example.com` | N/A | `.example.com` |
| **Browser Access** | `localhost` | `8081` | `http://localhost:8081/am` |
| **PingDS Connection** | `pingds` | `1636` | `pingds:1636` |

---

## Summary - What to Enter in Wizard

### Server Settings Page:
```yaml
Server URL:              http://pingam:8080/am
Cookie Domain:           .example.com
Platform Locale:         en_US
Configuration Directory: /opt/am-config
```

### Configuration Data Store Page:
```yaml
Server Name:  pingds
Port:         1636     ⚠️ IMPORTANT: The wizard may show 50636 - CHANGE IT to 1636!
SSL/TLS:      ✅ Enabled
Root Suffix:  ou=am-config
Login ID:     uid=am-config,ou=admins,ou=am-config
Password:     Passw0rd123
```

### User Data Store Page:
```yaml
Server Name:  pingds
Port:         1636     ⚠️ IMPORTANT: The wizard may show 50636 - CHANGE IT to 1636!
SSL/TLS:      ✅ Enabled
Root Suffix:  ou=identities
Login ID:     cn=Directory Manager
Password:     Passw0rd123
```

---

## ⚠️ Common Wizard Issue: Port 50636

**If the wizard shows port `50636`:**
- This is INCORRECT for our setup
- Change it to `1636`
- Port 1636 is the LDAPS port that PingDS is actually listening on

**Verify PingDS ports:**
```bash
docker exec pingds /opt/opendj/bin/status --hostname pingds --port 4444 \
  --bindDN "cn=Directory Manager" --bindPassword Passw0rd123 --trustAll
```

**You should see:**
```
>>>> Connection handlers
Name  : Port
------:------
LDAPS : 1636  ← Use this port!
```

---

## Common Mistakes to Avoid

❌ **Don't use:** `http://localhost:8081/am` in Server URL
✅ **Use:** `http://pingam:8080/am`

❌ **Don't use:** `localhost` in Cookie Domain
✅ **Use:** `.example.com`

❌ **Don't use:** Port `8081` in configuration
✅ **Use:** Port `8080` (internal container port)

---

## When You See Errors

**If redirects fail or OAuth doesn't work:**
- Check Server URL is `http://pingam:8080/am` (with port 8080, not 8081)

**If SSO doesn't work between apps:**
- Check Cookie Domain is `.example.com` (with leading dot)

**If browser can't access:**
- Use `http://localhost:8081/am` in browser (port 8081 for external access)
- Or add pingam to hosts file

---

**File Location:** `C:\PCFolders\Main\Learning\Docker\fr\sndbx1\pingam\notes\SERVER_URL_EXPLAINED.md`

**Main Guide:** `PINGAM_INSTALLATION_GUIDE.md` (lines 237-340)
