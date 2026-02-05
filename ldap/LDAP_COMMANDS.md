
  This removes the dictionary word check while keeping other security checks:

  # Run inside the container
  /opt/opendj/bin/dsconfig set-password-validator-prop \
    --hostname pingds \
    --port 4444 \
    --bindDN "cn=Directory Manager" \
    --bindPassword "Passw0rd123" \
    --validator-name "Dictionary" \
    --set enabled:false \
    --trustAll \
    --no-prompt

  /opt/opendj/bin/dsconfig set-password-validator-prop \
    --hostname pingds \
    --port 4444 \
    --bindDN "cn=Directory Manager" \
    --bindPassword "Passw0rd123" \
    --validator-name "Common passwords" \
    --set enabled:false \
    --trustAll \
    --no-prompt

  ---
  Option 2: Skip Validation for Admin Imports (Better for Testing)

  This allows Directory Manager to import passwords without validation:

  # Allow administrators to bypass password validation
  /opt/opendj/bin/dsconfig set-password-policy-prop \
    --hostname pingds \
    --port 4444 \
    --bindDN "cn=Directory Manager" \
    --bindPassword "Passw0rd123" \
    --policy-name "Default Password Policy" \
    --set skip-validation-for-administrators:true \
    --trustAll \
    --no-prompt

  ---
  Option 3: View Current Password Policy Settings

  To see what validators are active:

  # List all password validators
  /opt/opendj/bin/dsconfig list-password-validators \
    --hostname pingds \
    --port 4444 \
    --bindDN "cn=Directory Manager" \
    --bindPassword "Passw0rd123" \
    --trustAll

  ---
  Option 4: Completely Disable Password Validation (Quick & Easy)

  For a sandbox environment, this is the easiest:

  # Get current password policy settings
  /opt/opendj/bin/dsconfig get-password-policy-prop \
    --hostname pingds \
    --port 4444 \
    --bindDN "cn=Directory Manager" \
    --bindPassword "Passw0rd123" \
    --policy-name "Default Password Policy" \
    --trustAll

  # Remove all password validators
  /opt/opendj/bin/dsconfig set-password-policy-prop \
    --hostname pingds \
    --port 4444 \
    --bindDN "cn=Directory Manager" \
    --bindPassword "Passw0rd123" \
    --policy-name "Default Password Policy" \
    --remove password-validator:Dictionary \
    --remove password-validator:Character\ Set \
    --remove password-validator:Length-Based\ Password\ Validator \
    --trustAll \
    --no-prompt

  ---
  My Recommendation for Your Sandbox

  Run Option 2 - it's the cleanest approach for testing:

  /opt/opendj/bin/dsconfig set-password-policy-prop \ 
    --hostname pingds \
    --port 4444 \
    --bindDN "cn=Directory Manager" \
    --bindPassword "Passw0rd123" \
    --policy-name "Default Password Policy" \
    --set skip-validation-for-administrators:true \
    --trustAll \
    --no-prompt

  Why?
  - ✅ Keeps password policy active for regular users
  - ✅ Allows admin (Directory Manager) to import any password
  - ✅ Good for sandbox/testing environments
  - ✅ One simple command

  ---
  Complete Workflow

  Here's the full sequence to import your users:

  # 1. Enter the container
  docker exec -it pingds bash

  # 2. Disable password validation for administrators
  /opt/opendj/bin/dsconfig set-password-policy-prop \
    --hostname pingds \
    --port 4444 \
    --bindDN "cn=Directory Manager" \
    --bindPassword "Passw0rd123" \
    --policy-name "Default Password Policy" \
    --set skip-validation-for-administrators:true \
    --trustAll \
    --no-prompt

  # 3. Import the users
  /opt/opendj/bin/ldapmodify \
    --hostname pingds \
    --port 1636 \
    --useSSL \
    --trustAll \
    --bindDN "cn=Directory Manager" \
    --bindPassword "Passw0rd123" \
    --filename /tmp/sample-users.ldif

  # 4. Import the groups
  /opt/opendj/bin/ldapmodify \
    --hostname pingds \
    --port 1636 \
    --useSSL \
    --trustAll \
    --bindDN "cn=Directory Manager" \
    --bindPassword "Passw0rd123" \
    --filename /tmp/sample-groups.ldif

  ---
  Verify It Worked

  After importing, verify users can authenticate:

  # Test authentication for jdoe
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

  Expected output:
  dn: uid=jdoe,ou=people,ou=identities
  cn: John Doe
  mail: jdoe@example.com
  title: Engineering Manager

  ### list-local-db-indexes
/opt/opendj/bin/dsconfig list-local-db-indexes \
    --hostname pingds --port 4444 \
    --bindDN "cn=Directory Manager" --bindPassword "Passw0rd123" \
    --backend-name amTokens \
    --trustAll

  ### create-local-db-index


  ### rebuild-index


### list backends
  Step 1: Check PingDS Version

  /opt/opendj/bin/dsconfig --version

  Step 2: List Available Backends (Correct Syntax)

  /opt/opendj/bin/dsconfig list-backends \
    --hostname pingds \
    --port 4444 \
    --bindDN "cn=Directory Manager" \
    --bindPassword "Passw0rd123" \
    --trustAll

    Backend         : Type : enabled : base-dn              : confidentiality-enabled
----------------:------:---------:----------------------:------------------------
adminRoot       : ldif : false   : cn=admin data        : -
amCts           : je   : true    : ou=tokens            : false
amIdentityStore : je   : true    : ou=identities        : false
cfgStore        : je   : true    : ou=am-config         : false
monitorUser     : ldif : true    : uid=Monitor          : -
rootUser        : ldif : true    : cn=Directory Manager : -