# Sample Data Import Guide

## Files Created

1. **LDAP_STRUCTURE_GUIDE.md** - Complete LDAP command and structure reference
2. **sample-users.ldif** - 10 sample user accounts
3. **sample-groups.ldif** - 10 sample groups
4. **This file** - Instructions for importing and testing

---

## Sample Users Overview

| Username | Full Name | Title | Department | Employee Type |
|----------|-----------|-------|------------|---------------|
| jdoe | John Doe | Engineering Manager | 1000 | Full-Time |
| jsmith | Jane Smith | Senior Software Developer | 1000 | Full-Time |
| bjohnson | Bob Johnson | DevOps Engineer | 1000 | Full-Time |
| awilliams | Alice Williams | QA Lead | 1001 | Full-Time |
| cbrown | Charlie Brown | Product Manager | 2000 | Full-Time |
| dmartinez | Diana Martinez | UX Designer | 2001 | Full-Time |
| edavis | Edward Davis | System Administrator | 3000 | Full-Time |
| fgarcia | Fiona Garcia | HR Manager | 4000 | Full-Time |
| gwilson | George Wilson | Junior Developer | 1000 | Intern |
| handerson | Helen Anderson | Security Analyst | 3000 | Full-Time |

**Default Password**: `Passw0rd123` (for all users)

---

## Sample Groups Overview

| Group CN | Description | Members |
|----------|-------------|---------|
| Engineering | Engineering Department | jdoe, jsmith, bjohnson, gwilson |
| Developers | Software Developers | jsmith, gwilson |
| DevOps | DevOps Team | bjohnson, edavis |
| QA | Quality Assurance | awilliams |
| Product | Product & Design | cbrown, dmartinez |
| ITOps | IT Operations | edavis, handerson |
| Security | Security Team | handerson |
| Managers | All Managers | jdoe, awilliams, cbrown, fgarcia |
| FullTime | Full-Time Employees | 9 users (all except gwilson) |
| Admins | System Admins | edavis, handerson |

---

## How to Import Sample Data

### Step 1: Copy LDIF Files to Container

From your host machine (in the `pingds` directory):

```bash
# Copy users LDIF to container
docker cp sample-users.ldif pingds:/tmp/sample-users.ldif

# Copy groups LDIF to container
docker cp sample-groups.ldif pingds:/tmp/sample-groups.ldif
```

### Step 2: Import Users

Enter the container and import users:

```bash
# Enter container
docker exec -it pingds bash

# Import users
/opt/opendj/bin/ldapmodify \
  --hostname pingds \
  --port 1636 \
  --useSSL \
  --trustAll \
  --bindDN "cn=Directory Manager" \
  --bindPassword "Passw0rd123" \
  --filename /tmp/sample-users.ldif
```

Expected output:
```
Processing ADD request for uid=jdoe,ou=people,ou=identities
ADD operation successful for DN uid=jdoe,ou=people,ou=identities
Processing ADD request for uid=jsmith,ou=people,ou=identities
ADD operation successful for DN uid=jsmith,ou=people,ou=identities
...
```

### Step 3: Import Groups

```bash
# Import groups
/opt/opendj/bin/ldapmodify \
  --hostname pingds \
  --port 1636 \
  --useSSL \
  --trustAll \
  --bindDN "cn=Directory Manager" \
  --bindPassword "Passw0rd123" \
  --filename /tmp/sample-groups.ldif
```

### Alternative: Import from Host (Pipe Method)

```bash
# From host, pipe the file directly
docker exec -i pingds /opt/opendj/bin/ldapmodify \
  --hostname pingds \
  --port 1636 \
  --useSSL \
  --trustAll \
  --bindDN "cn=Directory Manager" \
  --bindPassword "Passw0rd123" \
  < sample-users.ldif

docker exec -i pingds /opt/opendj/bin/ldapmodify \
  --hostname pingds \
  --port 1636 \
  --useSSL \
  --trustAll \
  --bindDN "cn=Directory Manager" \
  --bindPassword "Passw0rd123" \
  < sample-groups.ldif
```

---

## Verification Commands

### Verify Users Were Added

```bash
# List all users in ou=people
/opt/opendj/bin/ldapsearch \
  --hostname pingds \
  --port 1636 \
  --useSSL \
  --trustAll \
  --bindDN "cn=Directory Manager" \
  --bindPassword "Passw0rd123" \
  --baseDN "ou=people,ou=identities" \
  --searchScope one \
  "(objectClass=inetOrgPerson)" \
  dn cn mail title

# Count users
/opt/opendj/bin/ldapsearch \
  --hostname pingds \
  --port 1636 \
  --useSSL \
  --trustAll \
  --bindDN "cn=Directory Manager" \
  --bindPassword "Passw0rd123" \
  --baseDN "ou=people,ou=identities" \
  --searchScope one \
  "(objectClass=inetOrgPerson)" \
  dn | grep "^dn:" | wc -l
```

### Verify Groups Were Added

```bash
# List all groups
/opt/opendj/bin/ldapsearch \
  --hostname pingds \
  --port 1636 \
  --useSSL \
  --trustAll \
  --bindDN "cn=Directory Manager" \
  --bindPassword "Passw0rd123" \
  --baseDN "ou=groups,ou=identities" \
  --searchScope one \
  "(objectClass=groupOfUniqueNames)" \
  dn cn description

# Show group membership for Engineering group
/opt/opendj/bin/ldapsearch \
  --hostname pingds \
  --port 1636 \
  --useSSL \
  --trustAll \
  --bindDN "cn=Directory Manager" \
  --bindPassword "Passw0rd123" \
  --baseDN "ou=groups,ou=identities" \
  --searchScope sub \
  "(cn=Engineering)" \
  dn cn uniqueMember
```

---

## Testing User Authentication

### Test User Login (Bind)

```bash
# Try to authenticate as jdoe
/opt/opendj/bin/ldapsearch \
  --hostname pingds \
  --port 1636 \
  --useSSL \
  --trustAll \
  --bindDN "uid=jdoe,ou=people,ou=identities" \
  --bindPassword "TestUser@2024" \
  --baseDN "uid=jdoe,ou=people,ou=identities" \
  --searchScope base \
  "(objectClass=*)" \
  dn cn mail title

# If successful, you'll see John Doe's entry
# If password is wrong, you'll get: "Invalid Credentials"
```

  # 1. Copy the ACI file to the container
  docker cp pingds/access-control.ldif pingds:/tmp/

  # 2. Import the ACIs using ldapmodify
  docker exec pingds /opt/opendj/bin/ldapmodify \
    --hostname pingds \
    --port 1636 \
    --useSSL \
    --trustAll \
    --bindDN "cn=Directory Manager" \
    --bindPassword "Passw0rd123" \
    --filename /tmp/access-control.ldif

  # 3. Test with user authentication (should work now)
  docker exec pingds /opt/opendj/bin/ldapsearch \
    --hostname pingds \
    --port 1636 \
    --useSSL \
    --trustAll \
    --bindDN "uid=jdoe,ou=people,ou=identities" \
    --bindPassword "TestUser@2024" \
    --baseDN "uid=jdoe,ou=people,ou=identities" \
    --searchScope base \
    "(objectClass=*)" \
    dn cn mail title


### Test Wrong Password

```bash
# This should FAIL with "Invalid Credentials"
/opt/opendj/bin/ldapsearch \
  --hostname pingds \
  --port 1636 \
  --useSSL \
  --trustAll \
  --bindDN "uid=jdoe,ou=people,ou=identities" \
  --bindPassword "WrongPassword" \
  --baseDN "uid=jdoe,ou=people,ou=identities" \
  --searchScope base \
  "(objectClass=*)"
```

---

## Useful LDAP Queries

### Search by Department

```bash
# Find all employees in department 1000 (Engineering)
/opt/opendj/bin/ldapsearch \
  --hostname pingds \
  --port 1636 \
  --useSSL \
  --trustAll \
  --bindDN "cn=Directory Manager" \
  --bindPassword "Passw0rd123" \
  --baseDN "ou=people,ou=identities" \
  --searchScope sub \
  "(departmentNumber=1000)" \
  dn cn title departmentNumber
```

### Search by Job Title

```bash
# Find all managers
/opt/opendj/bin/ldapsearch \
  --hostname pingds \
  --port 1636 \
  --useSSL \
  --trustAll \
  --bindDN "cn=Directory Manager" \
  --bindPassword "Passw0rd123" \
  --baseDN "ou=people,ou=identities" \
  --searchScope sub \
  "(title=*Manager*)" \
  dn cn title
```

### Search by Employee Type

```bash
# Find all interns
/opt/opendj/bin/ldapsearch \
  --hostname pingds \
  --port 1636 \
  --useSSL \
  --trustAll \
  --bindDN "cn=Directory Manager" \
  --bindPassword "Passw0rd123" \
  --baseDN "ou=people,ou=identities" \
  --searchScope sub \
  "(employeeType=Intern)" \
  dn cn title employeeType
```

### Find Users with Managers

```bash
# Find all users who have a manager assigned
/opt/opendj/bin/ldapsearch \
  --hostname pingds \
  --port 1636 \
  --useSSL \
  --trustAll \
  --bindDN "cn=Directory Manager" \
  --bindPassword "Passw0rd123" \
  --baseDN "ou=people,ou=identities" \
  --searchScope sub \
  "(manager=*)" \
  dn cn manager
```

### Search with AND/OR

```bash
# Find Full-Time employees in Engineering (dept 1000)
/opt/opendj/bin/ldapsearch \
  --hostname pingds \
  --port 1636 \
  --useSSL \
  --trustAll \
  --bindDN "cn=Directory Manager" \
  --bindPassword "Passw0rd123" \
  --baseDN "ou=people,ou=identities" \
  --searchScope sub \
  "(&(employeeType=Full-Time)(departmentNumber=1000))" \
  dn cn title employeeType departmentNumber
```

### Find Group Members

```bash
# Find what groups jsmith belongs to
/opt/opendj/bin/ldapsearch \
  --hostname pingds \
  --port 1636 \
  --useSSL \
  --trustAll \
  --bindDN "cn=Directory Manager" \
  --bindPassword "Passw0rd123" \
  --baseDN "ou=groups,ou=identities" \
  --searchScope sub \
  "(uniqueMember=uid=jsmith,ou=people,ou=identities)" \
  dn cn description
```

---

## Modifying User Data

### Change User's Phone Number

```bash
# Create a modify LDIF
cat > /tmp/modify-user.ldif << 'EOF'
dn: uid=jdoe,ou=people,ou=identities
changetype: modify
replace: telephoneNumber
telephoneNumber: +1-555-9999
EOF

# Apply the modification
/opt/opendj/bin/ldapmodify \
  --hostname pingds \
  --port 1636 \
  --useSSL \
  --trustAll \
  --bindDN "cn=Directory Manager" \
  --bindPassword "Passw0rd123" \
  --filename /tmp/modify-user.ldif
```

### Add an Attribute

```bash
# Add a new attribute (e.g., postalAddress)
cat > /tmp/add-attribute.ldif << 'EOF'
dn: uid=jdoe,ou=people,ou=identities
changetype: modify
add: postalAddress
postalAddress: 123 Main St, San Francisco, CA 94105
EOF

/opt/opendj/bin/ldapmodify \
  --hostname pingds \
  --port 1636 \
  --useSSL \
  --trustAll \
  --bindDN "cn=Directory Manager" \
  --bindPassword "Passw0rd123" \
  --filename /tmp/add-attribute.ldif
```

### Delete an Attribute

```bash
# Remove an attribute
cat > /tmp/delete-attribute.ldif << 'EOF'
dn: uid=jdoe,ou=people,ou=identities
changetype: modify
delete: mobile
EOF

/opt/opendj/bin/ldapmodify \
  --hostname pingds \
  --port 1636 \
  --useSSL \
  --trustAll \
  --bindDN "cn=Directory Manager" \
  --bindPassword "Passw0rd123" \
  --filename /tmp/delete-attribute.ldif
```

---

## Deleting Sample Data

### Delete All Sample Users

```bash
# Delete users one by one
/opt/opendj/bin/ldapdelete \
  --hostname pingds \
  --port 1636 \
  --useSSL \
  --trustAll \
  --bindDN "cn=Directory Manager" \
  --bindPassword "Passw0rd123" \
  "uid=jdoe,ou=people,ou=identities" \
  "uid=jsmith,ou=people,ou=identities" \
  "uid=bjohnson,ou=people,ou=identities" \
  "uid=awilliams,ou=people,ou=identities" \
  "uid=cbrown,ou=people,ou=identities" \
  "uid=dmartinez,ou=people,ou=identities" \
  "uid=edavis,ou=people,ou=identities" \
  "uid=fgarcia,ou=people,ou=identities" \
  "uid=gwilson,ou=people,ou=identities" \
  "uid=handerson,ou=people,ou=identities"
```

### Delete All Sample Groups

```bash
/opt/opendj/bin/ldapdelete \
  --hostname pingds \
  --port 1636 \
  --useSSL \
  --trustAll \
  --bindDN "cn=Directory Manager" \
  --bindPassword "Passw0rd123" \
  "cn=Engineering,ou=groups,ou=identities" \
  "cn=Developers,ou=groups,ou=identities" \
  "cn=DevOps,ou=groups,ou=identities" \
  "cn=QA,ou=groups,ou=identities" \
  "cn=Product,ou=groups,ou=identities" \
  "cn=ITOps,ou=groups,ou=identities" \
  "cn=Security,ou=groups,ou=identities" \
  "cn=Managers,ou=groups,ou=identities" \
  "cn=FullTime,ou=groups,ou=identities" \
  "cn=Admins,ou=groups,ou=identities"
```

---

## Common LDAP Operations Cheat Sheet

### Search Operations

| Task | Command Pattern |
|------|-----------------|
| Search all entries | `--baseDN "dc=example,dc=com" --searchScope sub "(objectClass=*)"` |
| Search specific user | `--baseDN "ou=people,ou=identities" "(uid=jdoe)"` |
| Search with wildcard | `"(cn=John*)"` or `"(mail=*@example.com)"` |
| Count results | Add `\| grep "^dn:" \| wc -l` to end |

### Modify Operations

| Operation | changetype | Action |
|-----------|-----------|---------|
| Add entry | `add` | Create new entry |
| Modify attribute | `modify` + `replace:` | Change existing value |
| Add attribute | `modify` + `add:` | Add new attribute |
| Delete attribute | `modify` + `delete:` | Remove attribute |
| Delete entry | Use `ldapdelete` | Remove entire entry |

### LDIF Format

```ldif
# Add new entry
dn: uid=newuser,ou=people,ou=identities
changetype: add
objectClass: inetOrgPerson
uid: newuser
cn: New User
sn: User

# Modify existing entry
dn: uid=jdoe,ou=people,ou=identities
changetype: modify
replace: telephoneNumber
telephoneNumber: +1-555-1234
```

---

## Troubleshooting

### Error: "Entry Already Exists"

If you try to import twice:
```
MODIFY operation failed: 68 (Entry Already Exists)
```

**Solution**: Delete the existing entry first, or use `ldapmodify` with `changetype: modify`

### Error: "No Such Object"

If the parent OU doesn't exist:
```
ADD operation failed: 32 (No Such Object)
```

**Solution**: Make sure parent OUs exist. In our case, `ou=people,ou=identities` should already exist from setup.

### Error: "Invalid Credentials"

```
The LDAP bind request failed: 49 (Invalid Credentials)
```

**Solution**: Check:
- Correct bind DN format
- Correct password
- For CTS account, password is `Password123` not `Passw0rd123`

### Verify Entry Exists

```bash
/opt/opendj/bin/ldapsearch \
  --hostname pingds \
  --port 1636 \
  --useSSL \
  --trustAll \
  --bindDN "cn=Directory Manager" \
  --bindPassword "Passw0rd123" \
  --baseDN "uid=jdoe,ou=people,ou=identities" \
  --searchScope base \
  "(objectClass=*)"
```

---

## Next Steps

1. **Import the sample data** following the instructions above
2. **Practice LDAP queries** using the examples in this guide
3. **Experiment with modifications** - change user attributes, add new users
4. **Integrate with ForgeRock AM** - use these users for authentication testing
5. **Create your own LDIF files** based on these templates

## Additional Resources

- **LDAP_STRUCTURE_GUIDE.md** - Detailed explanation of LDAP commands and structure
- **sample-users.ldif** - Template for creating users
- **sample-groups.ldif** - Template for creating groups
- PingDS Documentation: https://backstage.forgerock.com/docs/ds/

