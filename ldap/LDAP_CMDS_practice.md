  ```bash
# view ldap entry
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
  dn cn mail title telephoneNumber



# Create a modify LDIF
cat > /tmp/modify-user.ldif << 'EOF'
dn: uid=jdoe,ou=people,ou=identities
changetype: modify
replace: telephoneNumber
telephoneNumber: +1-555-9977
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

```bash
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
 --bindPassword "PassW0rd123" \
 --filename /tmp/add-attribute.ldif
```

### Delete an attricbute
```bash
cat > /tmp/delete-attribute.ldif << 'EOF'
dn: uid=jdoe,ou=people,ou=identities
changtype: modify
delete: mobile
EOF


```