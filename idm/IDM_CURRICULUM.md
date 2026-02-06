# PingIDM / ForgeRock IDM - Lead Engineer Curriculum

## Target Role
ForgeRock/Ping Identity Engineer Lead - IDM Expert (7-8 years experience)

---

## Core Topics (Must Know)

### 1. IDM Architecture & Fundamentals ✅
- Managed objects vs external resources
- ICF connectors and object types
- Repository architecture (DS-based vs JDBC)
- Config overlay pattern and file structure
- Reconciliation engine (full recon vs LiveSync vs implicit sync)
- Link table and relationship management
- Workflows and scripting model

### 2. Connector Development & Configuration
- ICF connector framework deep dive
- LDAP connector (complex filters, schema mapping)
- Database Table connector (JDBC)
- REST connector (modern API integration)
- CSV connector (bulk import/export)
- Custom connector development (ICF SPI)
- Connection pooling and performance tuning
- Error handling and retry logic

### 3. Mapping & Synchronization
- Mapping configuration (source → target)
- Attribute transformations (scripts, expressions)
- Situation policies (FOUND, ABSENT, UNQUALIFIED, etc.)
- Correlation logic and query design
- Link qualifiers (multi-account scenarios)
- Bi-directional sync patterns
- Conflict resolution strategies
- Mapping validation and testing

### 4. Reconciliation Deep Dive
- Full reconciliation process (source phase, target phase)
- LiveSync architecture (change log tailing)
- Implicit sync vs explicit sync
- Recon performance optimization
- Large dataset strategies (batching, threading)
- Failure recovery and restart
- Audit trail and reporting
- Scheduling and orchestration

### 5. Workflows & Business Logic
- Workflow engine architecture
- Approval processes and escalation
- Task management and delegated administration
- Scripted workflows vs declarative
- Integration with external BPMN engines
- Workflow versioning and migration
- Performance considerations
- Common workflow patterns (joiner/mover/leaver)

### 6. Self-Service & End User Experience
- User registration flows
- Password reset and forgot username
- Profile management (read vs write)
- Progressive profiling
- Account linking and unlinking
- Dashboard customization
- Theming and branding
- Mobile and responsive design

### 7. Roles, Entitlements & Provisioning
- Role model design (business vs IT roles)
- Temporal constraints (start/end dates)
- Conditional roles (assignment scripts)
- Entitlement management
- Provisioning policies
- De-provisioning and cleanup
- Role mining and certification
- RBAC vs ABAC patterns

### 8. Security & Authentication
- Internal authentication (managed/user)
- Delegated authentication (pass-through to AM)
- OAuth2 / OIDC integration
- SAML integration
- API security (bearer tokens, mutual TLS)
- Secrets management (keystores, encrypted properties)
- Audit logging and compliance
- GDPR and data privacy considerations

### 9. Integration Patterns
- AM-IDM integration (Platform mode)
- IDM as authoritative source vs aggregator
- Federation with external IdPs
- SCIM 2.0 provisioning
- HR feed integration patterns
- SaaS application provisioning (Okta, Azure AD, etc.)
- Legacy system integration
- Event-driven architecture (webhooks, message queues)

### 10. Performance & Scalability
- Horizontal scaling (clustered deployment)
- Database tuning (indexes, connection pools)
- Caching strategies
- Recon job optimization
- Workflow performance tuning
- Monitoring and metrics (Prometheus, custom endpoints)
- Capacity planning
- Load testing strategies

### 11. Production Operations
- Deployment strategies (blue-green, canary)
- Configuration promotion (dev → test → prod)
- Backup and restore procedures
- Disaster recovery planning
- Log management and troubleshooting
- Health checks and monitoring
- Alerting and incident response
- Change management processes

### 12. Upgrade & Migration
- IDM version upgrade paths (6.x → 7.x → 8.x)
- Schema migration
- Configuration migration (update vs replace)
- Custom code compatibility
- Rollback strategies
- Zero-downtime upgrades
- Data migration patterns
- Testing and validation

### 13. DevOps & Automation
- Docker and Kubernetes deployment
- Helm charts and Kustomize
- CI/CD pipelines (config validation, automated testing)
- Infrastructure as Code (Terraform, Ansible)
- GitOps workflows
- Secret injection (Vault, K8s Secrets)
- Immutable infrastructure patterns
- Monitoring stack integration

### 14. Advanced Scripting
- Groovy scripting in IDM
- JavaScript policy scripts
- Script contexts and bindings
- Script libraries and reuse
- Performance optimization
- Error handling and logging
- Testing strategies
- Common script patterns

### 15. Troubleshooting & Debugging
- Log analysis (openidm0.log, audit logs)
- Debug mode configuration
- REST API debugging
- Recon troubleshooting
- Connector debugging
- Performance profiling
- Common errors and solutions
- Support case patterns

### 16. Design & Architecture
- Multi-tenant design patterns
- Data residency and sovereignty
- High availability architecture
- Network topology design
- Security zones and DMZ placement
- API gateway integration
- Microservices vs monolithic
- Trade-off analysis

### 17. Compliance & Governance
- Audit trail requirements
- Access certification campaigns
- Segregation of duties (SoD)
- Compliance reporting
- Data lineage tracking
- Privacy by design
- Retention policies
- Regulatory requirements (SOX, HIPAA, etc.)

### 18. Team Leadership & Best Practices
- Code review standards
- Documentation requirements
- Knowledge transfer strategies
- Mentoring junior engineers
- Capacity planning for teams
- Technical debt management
- Proof of concept methodologies
- Vendor engagement strategies

---

## Learning Path Order

**Phase 1: Foundation** (Sessions 1-6)
1. Architecture & Fundamentals ← START HERE
2. Connector Development & Configuration
3. Mapping & Synchronization
4. Reconciliation Deep Dive
5. Workflows & Business Logic
6. Self-Service & End User Experience

**Phase 2: Advanced** (Sessions 7-12)
7. Roles, Entitlements & Provisioning
8. Security & Authentication
9. Integration Patterns
10. Performance & Scalability
11. Production Operations
12. Upgrade & Migration

**Phase 3: Expert** (Sessions 13-18)
13. DevOps & Automation
14. Advanced Scripting
15. Troubleshooting & Debugging
16. Design & Architecture
17. Compliance & Governance
18. Team Leadership & Best Practices

---

## Notes Organization

All Q&A files will be created under `frnotes/idm/`:
- `INT_QA_IDM_<Topic>_<Number>.md` (15 questions max per file)
- Index table at the top of each file
- Practical examples from hands-on labs
- Production-ready answers for lead-level interviews

---

*Created: 2026-02-06*
*Target: Lead Engineer (7-8 years experience)*
