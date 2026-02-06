# ForgeRock/PingAM Interview Questions — Custom Nodes, Amster, Upgrades (Part 3)

*Final section: Amster commands, scripting, CI/CD, and AM Upgrades*

---

## Part 2 Continued: Amster Commands and Automation

### Q18: Key Amster commands — connect, import-config, export-config

**Answer**:

#### **:connect** — Connect to AM instance

```groovy
// Connect with admin credentials
:connect http://pingam:8081/am -k /path/to/admin/.keyId

// Connect with username/password (less secure)
:connect http://pingam:8081/am amadmin changeit

// Connect to specific realm
:connect http://pingam:8081/am -k /path/to/admin/.keyId -r /techcorp
```

**Key parameters**:
- `-k` or `--private-key`: Path to admin user's `.keyId` file (more secure than password)
- `-r` or `--realm`: Realm to operate on (default: root realm)
- `--truststore`: Path to custom truststore for HTTPS

---

#### **:export-config** — Export AM configuration to JSON files

```groovy
// Export entire AM config
:export-config --path /tmp/am-config

// Export with options
:export-config \
  --path /tmp/am-config \
  --failOnError true \      // Fail immediately on any error
  --clean true              // Clean target directory before export

// Export specific realm only
:export-config --path /tmp/techcorp-config --realm /techcorp
```

**What gets exported**:
- Global services (Authentication, Session, OAuth2 Provider)
- Realm services (Trees, Policies, Scripts, OAuth2 clients)
- Configuration is written to JSON files organized by type

**Directory structure after export**:
```
/tmp/am-config/
├── global/
│   ├── GlobalServices.json
│   └── ...
└── realms/
    └── root/
        ├── realm-config/
        │   ├── authentication/
        │   │   ├── authenticationtrees/
        │   │   │   ├── TechCorpLogin.json
        │   │   │   └── TechCorpMFA.json
        │   │   └── scripts/
        │   │       └── RiskScoringScript.groovy
        │   ├── authorization/
        │   │   ├── policysets/
        │   │   │   └── TechCorpAPI.json
        │   │   └── resourcetypes/
        │   │       └── URL.json
        │   └── services/
        │       ├── OAuth2Provider.json
        │       └── EmailService.json
        └── techcorp/
            └── realm-config/
                └── ...
```

**What is NOT exported**:
- User data (identity store users)
- Session data (CTS)
- Audit logs
- Secret keys (only references)

---

#### **:import-config** — Import configuration from JSON files

```groovy
// Import config
:import-config --path /tmp/am-config

// Import with clean (remove existing config first)
:import-config \
  --path /tmp/am-config \
  --clean true \              // Delete existing config before import
  --failOnError false         // Continue on errors

// Import to specific realm
:import-config --path /tmp/techcorp-config --realm /techcorp
```

**--clean option**:
- `--clean true`: Deletes ALL existing configuration in the target realm before importing
- **Dangerous in production** — use with caution
- Recommended workflow: Export first (backup), then import with clean

**Error handling**:
- `--failOnError true`: Stop immediately on first error (default)
- `--failOnError false`: Log errors but continue importing

---

#### **Other key commands**:

**:create** — Create a single entity
```groovy
// Create OAuth2 client
:create OAuth2Client \
  --realm /techcorp \
  --id my-app \
  --data '{"coreOAuth2ClientConfig":{"redirectionUris":["http://localhost:3000/callback"],"scopes":["openid","profile"]}}'
```

**:update** — Update an entity
```groovy
:update OAuth2Client \
  --realm /techcorp \
  --id my-app \
  --data '{"coreOAuth2ClientConfig":{"scopes":["openid","profile","email"]}}'
```

**:delete** — Delete an entity
```groovy
:delete OAuth2Client --realm /techcorp --id my-app
```

**:query** — Query entities
```groovy
:query OAuth2Client --realm /techcorp
```

**Interview answer**: "The core Amster workflow is connect, export-config, import-config. I connect to an AM instance with `:connect` using an admin key file for security. `:export-config` writes the entire realm configuration to a directory of JSON files — trees, policies, scripts, OAuth2 clients. `:import-config` reads those files and creates/updates the configuration in the target AM instance. The `--clean` option is powerful but dangerous — it deletes all existing config before importing, so it's critical to export first as a backup. For individual entity operations, I use `:create`, `:update`, `:delete`, and `:query` commands."

**Sources**:
- [Export Configuration Data](https://backstage.forgerock.com/docs/amster/7/user-guide/amster-export-config.html)
- [Import Configuration Data](https://backstage.forgerock.com/docs/amster/7/user-guide/amster-import-config.html)
- [Amster FAQ](https://backstage.forgerock.com/knowledge/kb/article/a99944796)

---

### Q19: Amster scripting with Groovy

**Answer**:

Amster supports **Groovy scripting** for automation. You can create script files (`.amster`) containing commands and variable declarations.

#### **Example 1: Environment promotion script**

**promote-to-staging.amster**:
```groovy
// Variables
def devAM = "http://dev-am:8081/am"
def stagingAM = "http://staging-am:8081/am"
def adminKey = "/path/to/admin/.keyId"
def exportPath = "/tmp/dev-export"

// Export from DEV
println "Connecting to DEV..."
:connect ${devAM} -k ${adminKey}

println "Exporting DEV config..."
:export-config --path ${exportPath} --failOnError true

:exit

// Import to STAGING
println "Connecting to STAGING..."
:connect ${stagingAM} -k ${adminKey}

println "Importing config to STAGING..."
:import-config --path ${exportPath} --clean false --failOnError false

println "Promotion complete!"
:exit
```

**Run the script**:
```bash
./amster promote-to-staging.amster
```

---

#### **Example 2: Bulk create OAuth2 clients from CSV**

**CSV file** (`clients.csv`):
```
clientId,redirectUri,scopes
app1,http://app1.example.com/callback,"openid,profile"
app2,http://app2.example.com/callback,"openid,profile,email"
app3,http://app3.example.com/callback,"openid"
```

**Groovy script** (`create-clients.amster`):
```groovy
// Read CSV
def csvFile = new File('/path/to/clients.csv')
def lines = csvFile.readLines()

// Connect to AM
:connect http://pingam:8081/am -k /path/to/admin/.keyId

// Skip header, iterate rows
lines.drop(1).each { line ->
    def (clientId, redirectUri, scopes) = line.split(',')

    println "Creating client: ${clientId}"

    :create OAuth2Client \
      --realm /techcorp \
      --id ${clientId} \
      --data """{
        "coreOAuth2ClientConfig": {
          "redirectionUris": ["${redirectUri}"],
          "scopes": ${scopes.split(';').collect { "\"${it}\"" }},
          "defaultScopes": ["openid"],
          "clientType": "Confidential",
          "tokenEndpointAuthMethod": "client_secret_post"
        },
        "advancedOAuth2ClientConfig": {
          "grantTypes": ["authorization_code", "refresh_token"]
        }
      }"""
}

println "Created ${lines.size() - 1} OAuth2 clients"
:exit
```

---

#### **Example 3: Variable substitution with environment files**

**Environment-specific variables**:

**dev.properties**:
```properties
am.url=http://dev-am:8081/am
realm=/techcorp
admin.key=/keys/dev-admin.keyId
```

**prod.properties**:
```properties
am.url=https://sso.example.com/am
realm=/techcorp
admin.key=/keys/prod-admin.keyId
```

**Amster script** (`deploy.amster`):
```groovy
// Load environment properties
def env = System.getenv("ENVIRONMENT") ?: "dev"
def props = new Properties()
props.load(new FileInputStream("${env}.properties"))

// Connect using environment variables
:connect ${props['am.url']} -k ${props['admin.key']}

// Import config
:import-config --path /config/${env} --realm ${props['realm']}

:exit
```

**Run with environment selection**:
```bash
ENVIRONMENT=prod ./amster deploy.amster
```

---

#### **Example 4: Conditional logic and error handling**

```groovy
:connect http://pingam:8081/am amadmin changeit

// Check if realm exists before creating
def realmExists = false
try {
    :query Realm --filter realm=/techcorp
    realmExists = true
    println "Realm /techcorp exists"
} catch (Exception e) {
    println "Realm /techcorp does not exist"
}

if (!realmExists) {
    println "Creating realm /techcorp"
    :create Realm \
      --realm / \
      --id techcorp \
      --data '{"name":"techcorp","active":true}'
} else {
    println "Skipping realm creation"
}

:exit
```

---

#### **Variables and expressions**:

Amster supports **configuration expressions** in exported JSON files:

**Exported tree JSON**:
```json
{
  "nodes": {
    "RiskScoringNode": {
      "apiEndpoint": "&{am.risk.api.url|http://localhost:8080/risk}",
      "apiKey": "&{am.risk.api.key}",
      "highRiskThreshold": "&{am.risk.threshold.high|75}"
    }
  }
}
```

**Expression syntax**:
- `&{variable}` — Required variable (import fails if not defined)
- `&{variable|default}` — Optional with default value

**Define variables when importing**:
```groovy
// Set variables before import
System.setProperty("am.risk.api.url", "https://risk.example.com")
System.setProperty("am.risk.api.key", "sk-12345")
System.setProperty("am.risk.threshold.high", "80")

:import-config --path /config
```

**Or via environment variables**:
```bash
export am.risk.api.url=https://risk.example.com
export am.risk.api.key=sk-12345
./amster import.amster
```

**Interview answer**: "Amster scripts are Groovy files with `.amster` extension. I use them for environment promotion — connect to dev, export config, connect to staging, import config. For bulk operations, I read a CSV file and loop through it, creating OAuth2 clients with `:create` commands. Amster also supports configuration expressions in exported JSON files — `&{variable|default}` syntax lets me parameterize environment-specific values like API URLs or thresholds. I store environment properties in separate files (`dev.properties`, `prod.properties`) and load them in the script. This makes the same Amster script work across all environments."

**Sources**:
- [Amster Scripting](https://backstage.forgerock.com/docs/amster/7/user-guide/amster-usage-scripts.html)
- [Configuration Expressions](https://backstage.forgerock.com/docs/amster/7/user-guide/amster-usage-expressions.html)

---

### Q20: How to export an entire realm config and import to another environment

**Answer**:

**Step-by-step workflow**:

#### **Step 1: Export from source environment (DEV)**

```bash
# Start Amster
./amster

# Connect to DEV AM
:connect http://dev-am:8081/am -k /path/to/admin/.keyId

# Export entire realm
:export-config --path /tmp/dev-export --failOnError true

# Disconnect
:exit
```

**Step 2: Review exported files**

```bash
cd /tmp/dev-export
tree .
```

Output:
```
.
├── global/
│   ├── GlobalServices.json
│   └── ...
└── realms/
    └── root/
        ├── realm-config/
        │   ├── authentication/
        │   │   ├── authenticationtrees/
        │   │   │   ├── TechCorpLogin.json
        │   │   │   └── TechCorpMFA.json
        │   │   ├── chains/
        │   │   └── scripts/
        │   ├── authorization/
        │   │   ├── policysets/
        │   │   └── resourcetypes/
        │   ├── services/
        │   │   ├── OAuth2Provider.json
        │   │   └── EmailService.json
        │   └── applications/
        │       └── oauth2Clients/
        │           ├── app1.json
        │           └── app2.json
        └── techcorp/
            └── realm-config/
                └── ...
```

---

#### **Step 3: Replace environment-specific values (optional)**

Edit exported JSON files to parameterize environment-specific values:

**Before** (`OAuth2Provider.json`):
```json
{
  "issuerName": "http://dev-am:8081/am/oauth2"
}
```

**After** (with expression):
```json
{
  "issuerName": "&{am.issuer.url}"
}
```

---

#### **Step 4: Commit to Git (version control)**

```bash
cd /tmp/dev-export
git init
git add .
git commit -m "Export AM config from DEV — 2026-01-31"
git remote add origin https://github.com/myorg/am-config.git
git push -u origin main
```

---

#### **Step 5: Import to target environment (STAGING)**

```bash
# Start Amster
./amster

# Connect to STAGING AM
:connect http://staging-am:8081/am -k /path/to/staging-admin/.keyId

# Set environment variables for expressions
export am.issuer.url=http://staging-am:8081/am/oauth2

# Import config
:import-config --path /tmp/dev-export --clean false --failOnError false

# Review errors (if any)
# Amster logs to console and amster.log

:exit
```

**Important considerations**:

| Concern | Solution |
|---|---|
| **Secrets** (API keys, passwords) | Don't commit to Git — use expressions + environment variables or secret managers |
| **Environment-specific URLs** | Use configuration expressions: `&{am.base.url}` |
| **User data** | Amster does NOT export users — sync users separately via DS replication or IDM |
| **CTS data** | Session tokens are NOT exported — users will need to re-authenticate |
| **--clean flag** | Use `--clean false` for staging/prod to avoid deleting existing config |

---

#### **Production-ready workflow**:

**CI/CD pipeline** (GitHub Actions example):

```yaml
name: Promote AM Config to Staging

on:
  push:
    branches: [main]
    paths: ['am-config/**']

jobs:
  promote:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Download Amster
        run: |
          wget https://backstage.forgerock.com/downloads/amster-7.4.0.zip
          unzip amster-7.4.0.zip

      - name: Import to Staging AM
        env:
          AM_URL: ${{ secrets.STAGING_AM_URL }}
          ADMIN_KEY: ${{ secrets.STAGING_ADMIN_KEY }}
          am_issuer_url: ${{ secrets.STAGING_ISSUER_URL }}
        run: |
          echo ":connect ${AM_URL} -k ${ADMIN_KEY}" > import.amster
          echo ":import-config --path ./am-config --failOnError false" >> import.amster
          echo ":exit" >> import.amster

          amster/amster import.amster

      - name: Verify import
        run: |
          curl -s "${AM_URL}/json/serverinfo/*" | jq .version
```

---

**Interview answer**: "I export the entire realm config from DEV using Amster's `:export-config` command, which writes all trees, policies, scripts, and OAuth2 clients to JSON files. I parameterize environment-specific values using configuration expressions like `&{am.base.url}`. Then I commit the exported config to Git for version control. My CI/CD pipeline automatically imports the config to staging when I push to main — it sets environment variables for the expressions and runs `:import-config`. I use `--clean false` in staging and prod to avoid accidentally deleting existing config. Secrets like API keys are injected via GitHub Secrets, never committed to Git. This approach gives us repeatable, version-controlled AM deployments."

**Sources**:
- [Export Configuration Data](https://backstage.forgerock.com/docs/amster/7/user-guide/amster-export-config.html)
- [Import Configuration Data](https://backstage.forgerock.com/docs/amster/7/user-guide/amster-import-config.html)
- [How to export and import Service configurations](https://backstage.forgerock.com/knowledge/kb/article/a53282600)

---

### Q21: CI/CD pipeline integration with Amster

**Answer**:

Integrating Amster into CI/CD pipelines enables **Infrastructure as Code** for AM configuration.

#### **Common CI/CD patterns**:

**Pattern 1: Pull-based (GitOps)**
```
1. Dev exports config → Commits to Git (feature branch)
2. Pull request created → Code review
3. Merge to main → CI pipeline triggered
4. Pipeline imports config to STAGING
5. Automated tests run
6. Manual approval → Pipeline imports to PROD
```

**Pattern 2: Push-based**
```
1. Dev changes config in AM Console
2. Nightly job exports config from DEV → Commits to Git
3. If changes detected → Create automated PR
4. After review/merge → Import to STAGING/PROD
```

---

#### **Example: Jenkins Pipeline**

**Jenkinsfile**:
```groovy
pipeline {
    agent any

    environment {
        AMSTER_HOME = '/opt/amster'
        AM_CONFIG_REPO = 'https://github.com/myorg/am-config.git'
    }

    stages {
        stage('Checkout Config') {
            steps {
                git branch: 'main', url: "${AM_CONFIG_REPO}"
            }
        }

        stage('Deploy to Staging') {
            when {
                branch 'main'
            }
            steps {
                script {
                    sh """
                        ${AMSTER_HOME}/amster <<EOF
:connect ${STAGING_AM_URL} -k ${STAGING_ADMIN_KEY}
:import-config --path ./am-config --failOnError false
:exit
EOF
                    """
                }
            }
        }

        stage('Integration Tests') {
            steps {
                sh './run-integration-tests.sh'
            }
        }

        stage('Deploy to Prod') {
            when {
                expression { currentBuild.result == 'SUCCESS' }
            }
            input {
                message "Deploy to PROD?"
                ok "Deploy"
            }
            steps {
                script {
                    sh """
                        ${AMSTER_HOME}/amster <<EOF
:connect ${PROD_AM_URL} -k ${PROD_ADMIN_KEY}
:import-config --path ./am-config --failOnError false
:exit
EOF
                    """
                }
            }
        }
    }

    post {
        failure {
            mail to: 'devops@example.com',
                 subject: "AM Config Deploy Failed: ${env.JOB_NAME}",
                 body: "Check ${env.BUILD_URL}"
        }
    }
}
```

---

#### **Example: GitLab CI**

**.gitlab-ci.yml**:
```yaml
stages:
  - validate
  - deploy-staging
  - test
  - deploy-prod

variables:
  AMSTER_VERSION: "7.4.0"

before_script:
  - wget -q https://backstage.forgerock.com/downloads/amster-${AMSTER_VERSION}.zip
  - unzip -q amster-${AMSTER_VERSION}.zip
  - export PATH=$PATH:$(pwd)/amster

validate-config:
  stage: validate
  script:
    - echo "Validating JSON syntax"
    - find am-config -name "*.json" -exec jq empty {} \;

deploy-staging:
  stage: deploy-staging
  only:
    - main
  script:
    - |
      amster <<EOF
      :connect ${STAGING_AM_URL} -k ${STAGING_ADMIN_KEY}
      :import-config --path ./am-config --failOnError false
      :exit
      EOF

integration-tests:
  stage: test
  dependencies:
    - deploy-staging
  script:
    - ./run-tests.sh ${STAGING_AM_URL}

deploy-prod:
  stage: deploy-prod
  when: manual
  only:
    - main
  script:
    - |
      amster <<EOF
      :connect ${PROD_AM_URL} -k ${PROD_ADMIN_KEY}
      :import-config --path ./am-config --failOnError false
      :exit
      EOF
```

---

#### **Best Practices**:

| Practice | Why |
|---|---|
| **Validate JSON before import** | Catch syntax errors early (`jq empty *.json`) |
| **Use --failOnError false in pipelines** | Don't halt deployment on minor warnings |
| **Store secrets in CI/CD secret managers** | Never commit admin keys or passwords |
| **Export config nightly from DEV** | Catch manual changes made in Console |
| **Tag releases in Git** | Easy rollback to previous config versions |
| **Run integration tests after import** | Verify config works before promoting to prod |
| **Use manual approval for prod** | Human gate before production changes |
| **Notification on failure** | Email/Slack alerts for failed deployments |

---

**Interview answer**: "In our CI/CD pipeline, AM config lives in Git. When a developer merges config changes to main, GitLab CI kicks off: it downloads Amster, validates the JSON syntax, imports the config to staging, runs integration tests, and waits for manual approval to deploy to prod. We use GitLab's secret manager for admin keys and AM URLs. The entire config promotion is automated and repeatable — no more manual exports and imports. If a deployment fails, the pipeline sends a Slack notification. We tag each prod release in Git, so rolling back is just importing an older tag."

**Sources**:
- [Automation with ForgeRock AM 6.5](https://codingdaddy.dobbs.technology/2019/11/15/automation-with-forgerock-am-6-5/)
- [ForgeRock AM Configuration as an Artefact (Medium)](https://medium.com/@patrick.diligent/forgerock-am-configuration-as-an-artefact-38471981819e)

---

### Q22: Amster vs ForgeOps/CDK approach

**Answer**:

ForgeRock provides two DevOps approaches for managing AM configuration:

| Aspect | Amster | ForgeOps (CDK) |
|---|---|---|
| **What** | CLI tool, export/import JSON | Kubernetes-native deployment, Docker images |
| **Config storage** | JSON files in Git | JSON files baked into Docker images |
| **Deployment** | Import via Amster CLI | Deploy Kubernetes manifests |
| **Use case** | Traditional VM/bare-metal deployments | Cloud-native Kubernetes deployments |
| **Complexity** | Low — just a CLI tool | High — requires Kubernetes knowledge |
| **Immutability** | Mutable — import overwrites config | Immutable — new image per config change |
| **Speed** | Slow — import at runtime | Fast — config pre-baked in image |

---

#### **Amster approach** (traditional):

```
1. Export config from DEV
2. Commit JSON files to Git
3. CI/CD imports JSON to STAGING/PROD via Amster
4. AM runtime reads config from DS
```

**Pros**:
- Simple — no Kubernetes required
- Works on VMs, bare metal, Docker Compose
- Config changes don't require image rebuild

**Cons**:
- Import at runtime (slower deployments)
- Mutable — config can drift if someone uses Console

---

#### **ForgeOps/CDK approach** (cloud-native):

```
1. Export config from DEV (or use CDK)
2. Commit JSON files to Git
3. CI/CD builds custom AM Docker image with config baked in
4. Deploy new image to Kubernetes
5. Kubernetes rolling update replaces old pods
```

**Dockerfile**:
```dockerfile
FROM gcr.io/forgerock-io/am:7.4.0

# Bake config into image
COPY am-config/ /opt/am/config/

# AM reads config from filesystem at startup
```

**Kubernetes deployment**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: am
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: am
        image: myregistry/am:1.2.3  # Versioned image with config
```

**Pros**:
- Immutable — config can't drift
- Fast deployments (no import at runtime)
- Kubernetes-native (rollback, scaling, health checks)
- Config versioned with Docker image tags

**Cons**:
- Requires Kubernetes
- Config changes require image rebuild
- More complex tooling (Docker, K8s, Skaffold)

---

#### **ForgeOps tools**:

**CDK (Cloud Development Kit)**:
- Minimal Kubernetes deployment for dev/test
- Includes AM, DS, IDM, IG
- Uses Kustomize for config overlays
- `forgeops` CLI for common operations

**Amster in ForgeOps**:
- ForgeOps includes an `amster` pod
- Used for initial config import at cluster bootstrap
- After bootstrap, config is managed via Docker images

---

#### **When to use each**:

| Deployment Type | Recommended Approach |
|---|---|
| **On-prem VMs** | Amster |
| **Docker Compose** | Amster |
| **Kubernetes (cloud-native)** | ForgeOps (CDK) |
| **Hybrid (VMs + K8s)** | Amster + custom scripts |

---

**Interview answer**: "Amster is a CLI tool for traditional deployments — you export config to JSON, commit to Git, and import via Amster in the CI/CD pipeline. It's great for VM or bare-metal deployments. ForgeOps is the Kubernetes-native approach — you bake the config into a custom Docker image and deploy to K8s. ForgeOps is immutable and faster because config is pre-loaded, but it requires Kubernetes expertise and image rebuilds for every config change. In my current project, we use ForgeOps in prod (running on EKS) and Amster for our legacy staging environment (still on VMs). Both read from the same Git repo of exported config."

**Sources**:
- [ForgeOps Documentation](https://backstage.forgerock.com/docs/forgeops/7.3/start/start-here.html)
- [CDK Architecture](https://backstage.forgerock.com/docs/forgeops/7.1/cdk/architecture.html)
- [Types of configuration (ForgeOps)](https://backstage.forgerock.com/docs/forgeops/7.2/cdk/develop/fr-data.html)

---

### Q23: Real-world use — config promotion across environments (dev → staging → prod)

**Answer**:

**Real-world scenario**: A large enterprise with 3 environments (DEV, STAGING, PROD) needs to promote new authentication trees from development to production safely.

#### **Requirements**:
- Changes must be reviewed before production
- Rollback capability if production deployment fails
- Audit trail of who approved what
- No manual Console clicking in prod

---

#### **Solution: GitOps + Amster + CI/CD**

**Architecture**:
```
DEV AM ────export───> Git (feature branch) ────PR───> Git (main)
                                                         │
                                                         ├──> STAGING AM (auto)
                                                         │     │
                                                         │     ├──> Integration Tests
                                                         │     │
                                                         │     └──> Manual Approval
                                                         │
                                                         └──> PROD AM (manual)
```

---

#### **Step 1: Developer workflow (DEV)**

```bash
# Developer makes changes in AM Console (DEV)
# - Creates new "LoginWithBiometrics" tree
# - Adds WebAuthn nodes
# - Tests in DEV realm

# Export config
./amster export-dev.amster

# Review changes
git status
git diff am-config/realms/root/techcorp/authentication/authenticationtrees/LoginWithBiometrics.json

# Create feature branch
git checkout -b feature/biometric-login
git add am-config/
git commit -m "Add biometric login tree"
git push origin feature/biometric-login

# Create pull request
gh pr create --title "Add biometric login" --body "Implements WebAuthn-based login"
```

---

#### **Step 2: Code review**

- Senior engineer reviews the exported JSON
- Checks for:
  - Correct node wiring
  - No hardcoded secrets
  - Proper error outcomes
  - Environment-specific values parameterized
- Approves or requests changes

---

#### **Step 3: Merge triggers STAGING deployment**

**GitLab CI** (.gitlab-ci.yml):
```yaml
deploy-staging:
  stage: deploy-staging
  only:
    - main
  script:
    - echo "Deploying to STAGING"
    - amster <<EOF
      :connect ${STAGING_AM_URL} -k ${STAGING_ADMIN_KEY}
      :import-config --path ./am-config --failOnError false
      :exit
      EOF
  artifacts:
    reports:
      dotenv: deploy.env

integration-tests:
  stage: test
  dependencies:
    - deploy-staging
  script:
    - npm test -- --baseUrl=${STAGING_AM_URL}
  artifacts:
    when: always
    reports:
      junit: test-results.xml
```

---

#### **Step 4: Manual approval for PROD**

**GitLab CI** (continued):
```yaml
deploy-prod:
  stage: deploy-prod
  when: manual  # Requires manual trigger
  only:
    - main
  script:
    - echo "Deploying to PROD"
    - amster <<EOF
      :connect ${PROD_AM_URL} -k ${PROD_ADMIN_KEY}
      :import-config --path ./am-config --failOnError false
      :exit
      EOF
  environment:
    name: production
    url: https://sso.example.com/am
```

**Approval workflow**:
1. Tech lead reviews STAGING test results
2. Clicks "Deploy to Prod" button in GitLab
3. Audit trail: GitLab logs who approved, when, and which commit

---

#### **Step 5: Rollback if needed**

**Option 1: Revert Git commit**
```bash
# Find last working commit
git log --oneline

# Revert to previous version
git revert HEAD

# Push
git push origin main

# CI/CD re-deploys previous config
```

**Option 2: Import previous export**
```bash
# Find previous export (nightly backups)
ls -ltr /backups/am-config/

# Import manually
amster <<EOF
:connect ${PROD_AM_URL} -k ${PROD_ADMIN_KEY}
:import-config --path /backups/am-config/2026-01-30 --clean false
:exit
EOF
```

---

#### **Monitoring and alerting**:

```yaml
post-deploy-verify:
  stage: verify
  script:
    # Check AM is healthy
    - curl -f ${PROD_AM_URL}/json/serverinfo/version || exit 1

    # Check tree exists
    - curl -f ${PROD_AM_URL}/json/realm-config/authentication/authenticationtrees/LoginWithBiometrics || exit 1

    # Smoke test login
    - ./smoke-test.sh ${PROD_AM_URL}

  only:
    - main
```

---

**Interview answer**: "In our environment, config changes start in DEV. When a developer is satisfied, they export the config with Amster and commit to a feature branch in Git. After code review, the branch merges to main, triggering our CI/CD pipeline. The pipeline automatically imports the config to STAGING and runs integration tests. If tests pass, it waits for manual approval to deploy to PROD. We tag each prod release in Git for easy rollback. This approach gives us version control, peer review, automated testing, and an audit trail. No one touches the PROD AM Console — all changes go through Git and CI/CD."

---

## Part 3: AM Upgrades

### Q24: Major version upgrade paths (AM 6.x → 7.x → 7.5 → PingAM 8.0)

**Answer**:

ForgeRock AM (now PingAM) has specific supported upgrade paths. You **cannot skip major versions** — you must upgrade sequentially.

#### **Supported upgrade paths**:

```
AM 5.5.x → AM 6.0.x → AM 6.5.x → AM 7.0.x → AM 7.1.x → AM 7.2.x → AM 7.3.x → AM 7.4.x → AM 7.5.x → PingAM 8.0.x
```

**Key version milestones**:

| Version | Key Changes |
|---|---|
| **AM 6.0** | Authentication Trees introduced (replacing Chains) |
| **AM 6.5** | Last version supporting embedded DS in production |
| **AM 7.0** | External DS required, REST API v3, Amster rewrite |
| **AM 7.1** | OAuth2 introspection improvements |
| **AM 7.3** | Session blacklisting |
| **AM 7.4** | File-based configuration (preview), migration utility |
| **AM 7.5** | File-based configuration (GA) |
| **PingAM 8.0** | Rebranding to Ping Identity, Java 17 support |

---

#### **Upgrade path examples**:

**Example 1: AM 6.5 → AM 7.5**
```
AM 6.5.x → AM 7.0.x → AM 7.5.x
```
(Can skip 7.1-7.4 minor versions)

**Example 2: AM 5.5 → AM 7.5**
```
AM 5.5.x → AM 6.0.x → AM 6.5.x → AM 7.0.x → AM 7.5.x
```

**Example 3: AM 7.3 → PingAM 8.0**
```
AM 7.3.x → AM 7.5.x → PingAM 8.0.x
```

---

#### **Upgrade constraints**:

| Constraint | Details |
|---|---|
| **AM 6.5 to AM 7.x** | Must migrate embedded DS to external DS first |
| **AM 6.x to AM 7.x** | Configuration migrates from file-based to DS-based |
| **AM 7.x to AM 7.4+** | Can optionally migrate to file-based config |
| **Any to PingAM 8.0** | Must be on AM 7.5 first |

---

**Interview answer**: "ForgeRock AM upgrade paths are strictly sequential for major versions. You can't jump from AM 6.5 to AM 7.5 directly — you must go through AM 7.0 first. For minor versions (7.0, 7.1, 7.2), you can skip directly to a later minor version like 7.5. The biggest breaking change was AM 6.5 to AM 7.0, which removed support for embedded DS in production and moved configuration storage from files to DS. Before upgrading to PingAM 8.0, you must first be on AM 7.5."

**Sources**:
- [Plan the upgrade (PingAM 8.0.1)](https://backstage.forgerock.com/docs/am/8/upgrade-guide/upgrade-planning.html)
- [Supported upgrade paths (AM 7.0.2)](https://backstage.forgerock.com/docs/am/7/upgrade-guide/supported-upgrades.html)
- [Supported upgrade paths (PingAM 7.5.0)](https://backstage.forgerock.com/docs/am/7.5/upgrade-guide/supported-upgrades.html)

---

(Final sections continued in next file due to length limits...)
