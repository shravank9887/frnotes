### CTS setup 
  Do CTS setup in Top Level Realm (the root realm, usually just /).

  CTS is a global service - it's configured once at the top level and shared across all realms.      

  Why Top Level Realm:
  - CTS stores tokens for ALL realms (sessions, OAuth tokens, SAML assertions)
  - One CTS store serves the entire AM instance
  - Cannot have different CTS stores per realm

  Configuration Path:
  AM Console → Configure (top menu) → Global Services → CTS

  NOT:
  Top Level Realm → Services ❌

  Realms are used for:
  - OAuth providers (per-realm)
  - SAML entities (per-realm)
  - User stores (per-realm)
  - Authentication trees (per-realm)
  - Applications/policies (per-realm)

  Global Services (like CTS) are for:
  - Session token storage (CTS)
  - Platform settings
  - Logging
  - Monitoring

  So: Configure CTS once at global level, use it from all realms.


### important reference material
https://docs.pingidentity.com/pingam/8/security/securing-administration.html


amAdmin is hardcoded in the am source code

cn=dsameuser,ou=DSAME Users,%ROOT_SUFFIX%|cn=amService-UrlAccessAgent,ou=DSAME Users,%ROOT_SUFFIX%

/path/to/am/security/secrets/userpasswords

org.forgerock.openam.secrets.special.user.passwords.di

Capabilities that require a user profile 
Advance Server property - com.sun.identity.authentication.super.user 