# ForgeRock/PingAM Interview Questions — Complete Index

*Quick reference for all interview Q&A files*

---

## File Organization

| File | Topics Covered | Questions |
|------|----------------|-----------|
| **INTERVIEW_QUESTIONS.md** | Authentication, Federation, OAuth2/OIDC, Sessions | Q1-Q41 |
| **INT_QA_SAML.md** | SAML2 Metadata, SSO Onboarding, SLO, Bindings + moved Q8-Q22 from INTERVIEW_QUESTIONS.md | Q1-Q22 |
| **INT_QA_SAML_2.md** | SLO Bidirectionality, NameID (basics, formats, value map, account mapper, negotiation) | Q1-Q6 |
| **INT_QA_Policies.md** | Authorization, Policies, Resource Types, PDP/PEP | Q1-Q12 |
| **INT_QA_IDM_Install.md** | PingIDM Setup, LDAP Connector, Reconciliation | Q1-Q12 |
| **INT_QA_MFA_WebAuthn.md** | MFA, OATH TOTP, WebAuthn, FIDO2, Passkeys | Q1-Q31 |
| **INT_QA_Sessions_CTS.md** | Session Management, CTS, Stateful vs Stateless | Q1-Q19 |
| **INT_QA_CustomNodes_Amster_Upgrades.md** (Part 1) | Custom Node Fundamentals | Q1-Q10 |
| **INT_QA_CustomNodes_Amster_Upgrades_Part2.md** (Part 2) | Real-World Examples, Amster Basics | Q11-Q17 |
| **INT_QA_CustomNodes_Amster_Upgrades_Part3.md** (Part 3) | Amster Scripting, CI/CD, Upgrade Paths | Q18-Q24 |
| **INT_QA_CustomNodes_Amster_Upgrades_Part4_FINAL.md** (Part 4) | Upgrade Details, Rollback, Common Issues | Q25-Q30 |

---

## Quick Topic Lookup

### Authentication & Sessions
- Trees vs Chains → **INTERVIEW_QUESTIONS.md** Q1
- Callback mechanism → **INTERVIEW_QUESTIONS.md** Q2
- Page Node → **INTERVIEW_QUESTIONS.md** Q3
- authId vs tokenId → **INTERVIEW_QUESTIONS.md** Q7
- Stateful vs Stateless sessions → **INTERVIEW_QUESTIONS.md** Q9
- CTS architecture → **INTERVIEW_QUESTIONS.md** Q8

### Federation (SAML2)
- IdP Metadata contents (detailed) → **INT_QA_SAML.md** Q1
- SP Metadata contents (detailed) → **INT_QA_SAML.md** Q2
- Info shared with app owner after SP-initiated SSO → **INT_QA_SAML.md** Q3
- Info shared with app owner after IdP-initiated SSO → **INT_QA_SAML.md** Q4
- SLO (Single Logout) and configuration → **INT_QA_SAML.md** Q5
- Pre-SSO onboarding checklist → **INT_QA_SAML.md** Q6
- SAML binding types (Redirect, POST, SOAP, Artifact) → **INT_QA_SAML.md** Q7
- Why both IdP and SP metadata have SLO URLs → **INT_QA_SAML_2.md** Q1
- What is NameID → **INT_QA_SAML_2.md** Q2
- NameID formats (persistent, transient, email, unspecified) → **INT_QA_SAML_2.md** Q3
- IdP NameID Value Map → **INT_QA_SAML_2.md** Q4
- SP Account Mapper (consuming NameID) → **INT_QA_SAML_2.md** Q5
- NameID format negotiation → **INT_QA_SAML_2.md** Q6
- SP-Initiated SSO flow → **INTERVIEW_QUESTIONS.md** Q26
- IdP-Initiated vs SP-Initiated → **INTERVIEW_QUESTIONS.md** Q27
- Circle of Trust (CoT) → **INTERVIEW_QUESTIONS.md** Q28
- SAML metadata → **INTERVIEW_QUESTIONS.md** Q29
- NameID formats → **INTERVIEW_QUESTIONS.md** Q30
- MetaAlias → **INTERVIEW_QUESTIONS.md** Q31
- RelayState → **INTERVIEW_QUESTIONS.md** Q32
- SP architectures (3 types) → **INTERVIEW_QUESTIONS.md** Q33
- Debugging SAML → **INTERVIEW_QUESTIONS.md** Q34
- Internal vs External URLs → **INTERVIEW_QUESTIONS.md** Q36
- Cross-domain cookie problem → **INTERVIEW_QUESTIONS.md** Q41

### OAuth2 / OIDC
- Client Credentials flow → **INTERVIEW_QUESTIONS.md** Q15
- Token introspection → **INTERVIEW_QUESTIONS.md** Q16
- Opaque vs JWT tokens → **INTERVIEW_QUESTIONS.md** Q17
- Stateless OAuth2 tokens → **INTERVIEW_QUESTIONS.md** Q18
- CTS storage location → **INTERVIEW_QUESTIONS.md** Q19
- Identity prefixes (age!, usr!, grp!) → **INTERVIEW_QUESTIONS.md** Q20
- Authorization Code flow → **INTERVIEW_QUESTIONS.md** Q21
- Authorization Code via REST → **INTERVIEW_QUESTIONS.md** Q22
- access_token vs id_token → **INTERVIEW_QUESTIONS.md** Q23

### Authorization (Policies)
- Authorization model (Resource Type → Policy Set → Policy) → **INT_QA_Policies.md** Q1
- Resource Type design → **INT_QA_Policies.md** Q2
- PDP vs PEP → **INT_QA_Policies.md** Q3
- Resource Types: user-defined vs fixed → **INT_QA_Policies.md** Q4
- OAuth2 Scope Resource Type → **INT_QA_Policies.md** Q5
- Multiple policy matching → **INT_QA_Policies.md** Q6
- Policy evaluation REST API → **INT_QA_Policies.md** Q7
- Implicit deny vs explicit deny → **INT_QA_Policies.md** Q8
- Environment conditions → **INT_QA_Policies.md** Q9
- Time-based policy enforcement → **INT_QA_Policies.md** Q10
- End-to-end production flow → **INT_QA_Policies.md** Q11
- Policy evaluation response fields → **INT_QA_Policies.md** Q12

### PingIDM
- IDM vs AM → **INT_QA_IDM_Install.md** Q1
- Two-DS architecture → **INT_QA_IDM_Install.md** Q2
- idm-repo profile → **INT_QA_IDM_Install.md** Q3
- LDAP connector configuration → **INT_QA_IDM_Install.md** Q6
- ICF object types (`__ACCOUNT__`) → **INT_QA_IDM_Install.md** Q7
- Reconciliation → **INT_QA_IDM_Install.md** Q10

### MFA (OATH, WebAuthn, FIDO2)
- OATH vs TOTP vs HOTP → **INT_QA_MFA_WebAuthn.md** Q1
- OATH TOTP in PingAM → **INT_QA_MFA_WebAuthn.md** Q2
- WebAuthn fundamentals → **INT_QA_MFA_WebAuthn.md** Q5
- FIDO2 vs WebAuthn vs U2F → **INT_QA_MFA_WebAuthn.md** Q6-Q7
- Passwordless authentication → **INT_QA_MFA_WebAuthn.md** Q20
- Passkeys → **INT_QA_MFA_WebAuthn.md** Q21
- TOTP vs WebAuthn comparison → **INT_QA_MFA_WebAuthn.md** Q18
- Real-world FIDO2 deployment → **INT_QA_MFA_WebAuthn.md** Q26-Q31

### Sessions & CTS
- What is CTS → **INT_QA_Sessions_CTS.md** Q1
- CTS architecture in production → **INT_QA_Sessions_CTS.md** Q2
- Stateful vs stateless sessions → **INT_QA_Sessions_CTS.md** Q5
- Session lifecycle → **INT_QA_Sessions_CTS.md** Q8
- Three session timeouts → **INT_QA_Sessions_CTS.md** Q9
- Session quotas → **INT_QA_Sessions_CTS.md** Q11
- CTS sizing and monitoring → **INT_QA_Sessions_CTS.md** Q12
- Client-side session security → **INT_QA_Sessions_CTS.md** Q15
- Global vs Realm services → **INT_QA_Sessions_CTS.md** Q19

### Custom Authentication Nodes
- What is a custom node? → **Part 1** Q1
- Java API (AbstractDecisionNode, TreeContext, Action) → **Part 1** Q2
- Maven archetype → **Part 1** Q3
- Node lifecycle (OSGi plugin) → **Part 1** Q4
- Key interfaces (Node, SingleOutcomeNode, AbstractDecisionNode) → **Part 1** Q5
- OutcomeProvider → **Part 1** Q6
- Shared state, transient state, callbacks → **Part 1** Q7
- Config interface (@Attribute) → **Part 1** Q8
- Packaging and deployment → **Part 1** Q9
- Testing (unit and integration) → **Part 1** Q10
- Real-world examples:
  - Risk scoring node → **Part 2** Q11
  - SMS OTP node → **Part 2** Q12
  - Database lookup node → **Part 2** Q13
  - External REST API patterns → **Part 2** Q14
  - ForgeRock Marketplace nodes → **Part 2** Q15

### Amster (CLI Automation)
- What is Amster? → **Part 2** Q16
- Amster vs REST API vs Console → **Part 2** Q17
- Key commands (connect, export-config, import-config) → **Part 3** Q18
- Amster Groovy scripting → **Part 3** Q19
- Export/import realm config → **Part 3** Q20
- CI/CD pipeline integration → **Part 3** Q21
- Amster vs ForgeOps/CDK → **Part 3** Q22
- Real-world config promotion → **Part 3** Q23

### AM Upgrades
- Upgrade paths (AM 6.x → 7.x → 8.0) → **Part 3** Q24
- Pre-upgrade checklist → **Part 4** Q25
- In-place vs side-by-side upgrades → **Part 4** Q26
- Configuration migration (file-based → DS-based → FBC) → **Part 4** Q27
- Custom node compatibility across versions → **Part 4** Q28
- Rollback strategy → **Part 4** Q29
- Common upgrade issues → **Part 4** Q30

---

## Interview Preparation Strategy

### 1. Core Topics (High Priority)
- Authentication Trees (callbacks, shared state, tree design)
- SAML2 Federation (SP-initiated SSO, NameID mapping, debugging)
- OAuth2/OIDC (grant flows, token types, introspection)
- Authorization (Resource Types, Policy Sets, Policies, PDP/PEP)
- Custom Authentication Nodes (API, lifecycle, testing, real-world examples)
- Amster (export/import, scripting, CI/CD)

### 2. Advanced Topics (Medium Priority)
- PingIDM integration (LDAP connector, reconciliation)
- AM Upgrades (paths, pre-upgrade checklist, rollback)
- Session management (stateful vs stateless, CTS)
- ForgeOps/CDK (Kubernetes deployments)

### 3. Practical Demonstrations
Prepare to demonstrate:
- **Curl commands**: Authentication REST API, policy evaluation, token introspection
- **Amster scripts**: Export config, import to another environment
- **Custom node code**: Show a simple node implementation
- **Troubleshooting**: How to read AM logs, debug SAML, check DS entries

### 4. Real-World Scenarios
Be ready to discuss:
- "How did you implement MFA in your project?"
- "Describe a time you debugged a SAML federation issue"
- "How do you promote config from dev to prod?"
- "What's your upgrade strategy for AM?"
- "How do you handle custom node testing?"

### 5. Key Phrases for Interviews
- "In production, we use..."
- "The trade-off is..."
- "For troubleshooting, I would..."
- "Best practice is..."
- "The risk is... so we mitigate by..."

---

## Sources Reference

All content sourced from official documentation:
- [ForgeRock Backstage Documentation](https://backstage.forgerock.com/docs/)
- [Ping Identity Documentation](https://docs.pingidentity.com/)
- [ForgeRock Community](https://community.forgerock.com/)

Key documentation pages:
- **Authentication Nodes**: https://backstage.forgerock.com/docs/am/7/auth-nodes/
- **Amster User Guide**: https://backstage.forgerock.com/docs/amster/7/user-guide/
- **AM Upgrade Guide**: https://backstage.forgerock.com/docs/am/7/upgrade-guide/
- **ForgeOps Documentation**: https://backstage.forgerock.com/docs/forgeops/7.3/

---

## Quick Stats

**Total Interview Questions**: ~133 questions across all files

**Topics Covered**:
- Authentication & Sessions: 14 questions
- Federation (SAML2): 16 questions
- OAuth2/OIDC: 11 questions
- Authorization (Policies): 12 questions
- PingIDM: 15 questions
- MFA / WebAuthn / FIDO2: 31 questions
- Sessions & CTS: 19 questions
- Custom Authentication Nodes: 15 questions
- Amster: 8 questions
- AM Upgrades: 6 questions

**Real-World Examples**: 40+ practical examples with code

**Best Practices**: 60+ production-ready patterns and recommendations

---

*Last Updated: 2026-01-31*
*For the most detailed content, refer to the individual Q&A files*
