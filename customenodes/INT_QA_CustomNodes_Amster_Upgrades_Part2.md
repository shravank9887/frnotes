# ForgeRock/PingAM Interview Questions — Custom Nodes, Amster, Upgrades (Part 2)

*Continuation of INT_QA_CustomNodes_Amster_Upgrades.md*

---

## Part 1 Continued: Real-World Custom Node Examples

### Q11: Real-world example — Custom risk scoring node

**Answer**:

A risk scoring node evaluates authentication risk based on user behavior, device characteristics, IP geolocation, and other signals, then routes to different authentication paths based on risk level.

```java
package com.example.nodes;

import com.google.inject.assistedinject.Assisted;
import org.forgerock.json.JsonValue;
import org.forgerock.openam.annotations.sm.Attribute;
import org.forgerock.openam.auth.node.api.*;
import org.forgerock.util.i18n.PreferredLocales;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import javax.inject.Inject;
import java.util.List;
import java.util.ResourceBundle;

@Node.Metadata(
    outcomeProvider = RiskScoringNode.RiskOutcomeProvider.class,
    configClass = RiskScoringNode.Config.class,
    tags = {"risk", "security"}
)
public class RiskScoringNode implements Node {

    private final Logger logger = LoggerFactory.getLogger(RiskScoringNode.class);
    private final Config config;
    private final RiskEngine riskEngine;

    public interface Config {

        @Attribute(order = 10)
        default int lowRiskThreshold() {
            return 30;
        }

        @Attribute(order = 20)
        default int mediumRiskThreshold() {
            return 70;
        }

        @Attribute(order = 30)
        default boolean enableIPGeolocation() {
            return true;
        }

        @Attribute(order = 40)
        default boolean enableDeviceFingerprinting() {
            return true;
        }

        @Attribute(order = 50)
        default boolean enableVelocityChecks() {
            return true;
        }
    }

    @Inject
    public RiskScoringNode(@Assisted Config config, RiskEngine riskEngine) {
        this.config = config;
        this.riskEngine = riskEngine;
    }

    @Override
    public Action process(TreeContext context) throws NodeProcessException {

        String username = context.sharedState.get("username").asString();
        String ipAddress = context.request.getHeader("X-Forwarded-For");

        if (ipAddress == null || ipAddress.isEmpty()) {
            ipAddress = context.request.getRemoteAddr();
        }

        logger.debug("Calculating risk for user: {}, IP: {}", username, ipAddress);

        try {
            // Build risk context
            RiskContext riskContext = RiskContext.builder()
                .username(username)
                .ipAddress(ipAddress)
                .userAgent(context.request.getHeader("User-Agent"))
                .enableIPCheck(config.enableIPGeolocation())
                .enableDeviceCheck(config.enableDeviceFingerprinting())
                .enableVelocityCheck(config.enableVelocityChecks())
                .build();

            // Calculate risk score (0-100)
            RiskScore riskScore = riskEngine.calculateRisk(riskContext);

            int totalScore = riskScore.getTotalScore();
            logger.info("Risk score for {}: {} (IP: {}, Device: {}, Velocity: {})",
                username, totalScore,
                riskScore.getIpScore(), riskScore.getDeviceScore(), riskScore.getVelocityScore());

            // Store risk details in shared state for downstream nodes or logging
            JsonValue updatedSharedState = context.sharedState.copy()
                .put("riskScore", totalScore)
                .put("riskLevel", getRiskLevel(totalScore))
                .put("riskFactors", riskScore.toJson());

            // Route based on risk level
            if (totalScore < config.lowRiskThreshold()) {
                return goTo("low-risk")
                    .replaceSharedState(updatedSharedState)
                    .putSessionProperty("authLevel", "1")
                    .build();
            } else if (totalScore < config.mediumRiskThreshold()) {
                return goTo("medium-risk")
                    .replaceSharedState(updatedSharedState)
                    .build();
            } else {
                return goTo("high-risk")
                    .replaceSharedState(updatedSharedState)
                    .build();
            }

        } catch (RiskEngineException e) {
            logger.error("Risk engine failed for user: " + username, e);

            // Fail-open or fail-closed based on config
            if (config.failOpen()) {
                logger.warn("Risk engine failed, failing open (allow low-risk)");
                return goTo("low-risk")
                    .replaceSharedState(context.sharedState.put("riskEngineError", true))
                    .build();
            } else {
                throw new NodeProcessException("Risk calculation failed", e);
            }
        }
    }

    private String getRiskLevel(int score) {
        if (score < config.lowRiskThreshold()) return "LOW";
        if (score < config.mediumRiskThreshold()) return "MEDIUM";
        return "HIGH";
    }

    public static class RiskOutcomeProvider implements OutcomeProvider {
        @Override
        public List<Outcome> getOutcomes(PreReqCheck preReqCheck, Class<? extends Node> nodeType) {
            return ImmutableList.of(
                new Outcome("low-risk", "Low Risk"),
                new Outcome("medium-risk", "Medium Risk"),
                new Outcome("high-risk", "High Risk")
            );
        }
    }
}
```

**RiskEngine implementation**:
```java
public class RiskEngine {

    private final IPGeoLocationService geoService;
    private final DeviceFingerprintService deviceService;
    private final VelocityCheckService velocityService;

    @Inject
    public RiskEngine(IPGeoLocationService geoService,
                      DeviceFingerprintService deviceService,
                      VelocityCheckService velocityService) {
        this.geoService = geoService;
        this.deviceService = deviceService;
        this.velocityService = velocityService;
    }

    public RiskScore calculateRisk(RiskContext context) throws RiskEngineException {
        int ipScore = 0;
        int deviceScore = 0;
        int velocityScore = 0;

        // IP Geolocation check (0-40 points)
        if (context.isEnableIPCheck()) {
            GeoLocation geo = geoService.lookup(context.getIpAddress());
            ipScore += checkIPRisk(geo, context.getUsername());
        }

        // Device fingerprint check (0-30 points)
        if (context.isEnableDeviceCheck()) {
            DeviceProfile device = deviceService.analyze(context.getUserAgent());
            deviceScore += checkDeviceRisk(device, context.getUsername());
        }

        // Velocity check (0-30 points)
        if (context.isEnableVelocityCheck()) {
            VelocityStats stats = velocityService.checkVelocity(context.getUsername());
            velocityScore += checkVelocityRisk(stats);
        }

        int totalScore = ipScore + deviceScore + velocityScore;

        return new RiskScore(totalScore, ipScore, deviceScore, velocityScore);
    }

    private int checkIPRisk(GeoLocation geo, String username) {
        int score = 0;

        // Check if country is on blocklist
        if (isBlockedCountry(geo.getCountry())) {
            score += 40;
        }
        // Check if IP is on threat intel feed
        else if (isMaliciousIP(geo.getIpAddress())) {
            score += 35;
        }
        // Check if location changed drastically from last login
        else if (isImpossibleTravel(geo, username)) {
            score += 25;
        }
        // Check if ASN is known for fraud
        else if (isSuspiciousASN(geo.getAsn())) {
            score += 15;
        }

        return score;
    }

    private int checkDeviceRisk(DeviceProfile device, String username) {
        int score = 0;

        // Check if device is recognized for this user
        if (!device.isRecognized(username)) {
            score += 15;
        }

        // Check for suspicious user agent patterns (bots, scrapers)
        if (device.isSuspiciousUserAgent()) {
            score += 20;
        }

        // Check for automation tools (Selenium, Puppeteer)
        if (device.hasAutomationIndicators()) {
            score += 25;
        }

        return score;
    }

    private int checkVelocityRisk(VelocityStats stats) {
        int score = 0;

        // Multiple failed logins in short time
        if (stats.getFailedLogins() > 3 && stats.getTimeWindowMinutes() < 5) {
            score += 20;
        }

        // Logins from multiple IPs in short time
        if (stats.getUniqueIPs() > 2 && stats.getTimeWindowMinutes() < 10) {
            score += 25;
        }

        return score;
    }
}
```

**Interview answer**: "I built a risk scoring node that integrates IP geolocation, device fingerprinting, and velocity checks to calculate a 0-100 risk score. The node has three outcomes: low-risk (0-29, straight-through auth), medium-risk (30-69, SMS OTP required), and high-risk (70+, step-up MFA + fraud review). Each risk factor contributes points — for example, a login from a blocklisted country adds 40 points, a new device adds 15 points. The risk score and detailed factors are stored in shared state for audit logging. The node fails open by default if the risk engine is down, but that's configurable. This replaced our previous approach where MFA was all-or-nothing."

---

### Q12: Real-world example — Custom MFA node calling external API

**Answer**:

A custom MFA node that integrates with a third-party SMS provider (e.g., Twilio) to send one-time codes.

```java
@Node.Metadata(
    outcomeProvider = SMSOTPNode.OTPOutcomeProvider.class,
    configClass = SMSOTPNode.Config.class,
    tags = {"mfa", "otp", "sms"}
)
public class SMSOTPNode implements Node {

    private final Logger logger = LoggerFactory.getLogger(SMSOTPNode.class);
    private final Config config;
    private final TwilioSMSService smsService;
    private final OTPGenerator otpGenerator;

    public interface Config {

        @Attribute(order = 10)
        String twilioAccountSid();

        @Attribute(order = 20)
        String twilioAuthToken();

        @Attribute(order = 30)
        String fromPhoneNumber();

        @Attribute(order = 40)
        default int otpLength() {
            return 6;
        }

        @Attribute(order = 50)
        default int otpValidityMinutes() {
            return 5;
        }

        @Attribute(order = 60)
        default int maxRetries() {
            return 3;
        }
    }

    @Inject
    public SMSOTPNode(@Assisted Config config,
                      TwilioSMSService smsService,
                      OTPGenerator otpGenerator) {
        this.config = config;
        this.smsService = smsService;
        this.otpGenerator = otpGenerator;
    }

    @Override
    public Action process(TreeContext context) throws NodeProcessException {

        String username = context.sharedState.get("username").asString();
        String phoneNumber = getUserPhoneNumber(context);

        // Check if OTP already sent (user is submitting code)
        if (context.transientState.isDefined("otpCode")) {
            return verifyOTP(context);
        }

        // First time — generate and send OTP
        String otp = otpGenerator.generate(config.otpLength());
        long expiryTime = System.currentTimeMillis() + (config.otpValidityMinutes() * 60 * 1000);

        try {
            // Send SMS via Twilio
            smsService.sendSMS(
                config.fromPhoneNumber(),
                phoneNumber,
                String.format("Your verification code is: %s. Valid for %d minutes.",
                    otp, config.otpValidityMinutes())
            );

            logger.info("Sent OTP to user: {}, phone: {}", username, maskPhoneNumber(phoneNumber));

            // Store OTP and expiry in transient state (encrypted, not sent to client)
            JsonValue transientState = context.transientState.copy()
                .put("otpCode", otp)
                .put("otpExpiry", expiryTime)
                .put("otpRetries", 0);

            // Ask user to enter OTP
            return Action.send(new PasswordCallback("Enter OTP code", false))
                .replaceTransientState(transientState)
                .build();

        } catch (SMSException e) {
            logger.error("Failed to send OTP to user: " + username, e);
            throw new NodeProcessException("Failed to send verification code", e);
        }
    }

    private Action verifyOTP(TreeContext context) throws NodeProcessException {

        // Get stored OTP from transient state
        String expectedOTP = context.transientState.get("otpCode").asString();
        long expiryTime = context.transientState.get("otpExpiry").asLong();
        int retries = context.transientState.get("otpRetries").defaultTo(0).asInt();

        // Get user-submitted OTP from callback
        List<PasswordCallback> callbacks = context.getCallbacks(PasswordCallback.class);
        if (callbacks.isEmpty()) {
            throw new NodeProcessException("No OTP callback found");
        }

        String submittedOTP = new String(callbacks.get(0).getPassword());

        // Check if OTP expired
        if (System.currentTimeMillis() > expiryTime) {
            logger.warn("OTP expired for user: {}",
                context.sharedState.get("username").asString());
            return goTo("expired")
                .replaceSharedState(context.sharedState.put("otpError", "Code expired"))
                .build();
        }

        // Verify OTP
        if (expectedOTP.equals(submittedOTP)) {
            logger.info("OTP verified successfully for user: {}",
                context.sharedState.get("username").asString());

            return goTo("success")
                .putSessionProperty("authLevel", "2")  // MFA completed
                .build();
        } else {
            retries++;

            // Max retries exceeded
            if (retries >= config.maxRetries()) {
                logger.warn("Max OTP retries exceeded for user: {}",
                    context.sharedState.get("username").asString());
                return goTo("locked")
                    .replaceSharedState(context.sharedState.put("otpError", "Too many attempts"))
                    .build();
            }

            // Allow retry
            logger.debug("Invalid OTP submitted, retry {}/{}",
                retries, config.maxRetries());

            return Action.send(new PasswordCallback(
                    String.format("Invalid code. Try again (%d/%d):",
                        retries, config.maxRetries()), false))
                .replaceTransientState(
                    context.transientState.put("otpRetries", retries))
                .build();
        }
    }

    private String getUserPhoneNumber(TreeContext context) throws NodeProcessException {
        // Try to get from shared state first (set by previous node)
        if (context.sharedState.isDefined("phoneNumber")) {
            return context.sharedState.get("phoneNumber").asString();
        }

        // Fall back to user profile
        String username = context.sharedState.get("username").asString();
        try {
            AMIdentity identity = coreWrapper.getIdentity(username, realm);
            Set<String> phones = identity.getAttribute("telephoneNumber");
            if (phones != null && !phones.isEmpty()) {
                return phones.iterator().next();
            }
        } catch (Exception e) {
            throw new NodeProcessException("Failed to retrieve phone number", e);
        }

        throw new NodeProcessException("No phone number found for user: " + username);
    }

    private String maskPhoneNumber(String phone) {
        if (phone.length() < 4) return "***";
        return "****" + phone.substring(phone.length() - 4);
    }

    public static class OTPOutcomeProvider implements OutcomeProvider {
        @Override
        public List<Outcome> getOutcomes(PreReqCheck preReqCheck, Class<? extends Node> nodeType) {
            return ImmutableList.of(
                new Outcome("success", "OTP Verified"),
                new Outcome("expired", "OTP Expired"),
                new Outcome("locked", "Too Many Attempts")
            );
        }
    }
}
```

**Interview answer**: "I built a custom SMS OTP node integrated with Twilio. On first execution, it generates a 6-digit code, stores it in transient state (encrypted), and sends it via the Twilio REST API. It then shows a PasswordCallback to collect the code. When the user submits, the node compares the submitted code to the stored one, checking for expiry (5 minutes) and max retries (3 attempts). It has three outcomes: success (OTP verified, auth level set to 2), expired (OTP timed out), and locked (too many failed attempts). This approach keeps sensitive data (the OTP itself) in transient state so it's never visible in the authId JWT. In production, we cache the Twilio client to avoid recreating HTTP connections on every request."

---

### Q13: Real-world example — Database lookup node

**Answer**:

A custom node that queries a legacy database to enrich user profile with attributes not stored in LDAP.

```java
@Node.Metadata(
    outcomeProvider = SingleOutcomeNode.OutcomeProvider.class,
    configClass = DatabaseLookupNode.Config.class,
    tags = {"database", "attribute", "enrichment"}
)
public class DatabaseLookupNode extends SingleOutcomeNode {

    private final Config config;
    private final DatabaseService dbService;

    public interface Config {

        @Attribute(order = 10)
        String jdbcUrl();  // jdbc:mysql://legacy-db:3306/users

        @Attribute(order = 20)
        String username();

        @Attribute(order = 30)
        String password();

        @Attribute(order = 40)
        default String sqlQuery() {
            return "SELECT employee_id, department, cost_center, manager_email " +
                   "FROM employees WHERE username = ?";
        }

        @Attribute(order = 50)
        default int queryTimeoutSeconds() {
            return 5;
        }
    }

    @Inject
    public DatabaseLookupNode(@Assisted Config config, DatabaseService dbService) {
        this.config = config;
        this.dbService = dbService;
    }

    @Override
    public Action process(TreeContext context) throws NodeProcessException {

        String username = context.sharedState.get("username").asString();

        try (Connection conn = dbService.getConnection(
                config.jdbcUrl(), config.username(), config.password());
             PreparedStatement stmt = conn.prepareStatement(config.sqlQuery())) {

            stmt.setQueryTimeout(config.queryTimeoutSeconds());
            stmt.setString(1, username);

            ResultSet rs = stmt.executeQuery();

            if (rs.next()) {
                // Enrich shared state with DB attributes
                JsonValue enrichedState = context.sharedState.copy()
                    .put("employeeId", rs.getString("employee_id"))
                    .put("department", rs.getString("department"))
                    .put("costCenter", rs.getString("cost_center"))
                    .put("managerEmail", rs.getString("manager_email"));

                logger.info("Enriched user profile from DB: {}", username);

                return goToNext()
                    .replaceSharedState(enrichedState)
                    .build();
            } else {
                logger.warn("User not found in legacy DB: {}", username);

                // Continue anyway (user might be new)
                return goToNext().build();
            }

        } catch (SQLException e) {
            logger.error("Database query failed for user: " + username, e);

            // Fail gracefully — continue without enrichment
            return goToNext()
                .replaceSharedState(context.sharedState.put("dbLookupFailed", true))
                .build();
        }
    }
}
```

**Interview answer**: "We have a legacy Oracle database with employee attributes not in LDAP — employee ID, cost center, department. I built a node that queries this database via JDBC, enriches the shared state with those attributes, and continues. If the DB is down or the query times out, it fails gracefully and continues without the attributes. Downstream nodes check if the attributes exist before using them. This avoided a costly LDAP schema extension and data migration project. We use HikariCP connection pooling in the DatabaseService to avoid opening a new DB connection on every authentication."

---

### Q14: Common patterns — Calling external REST APIs from a node

**Answer**:

**Pattern 1: Synchronous REST call with timeout**

```java
public class ExternalAPINode extends AbstractDecisionNode {

    private final Config config;
    private final HttpClient httpClient;

    @Inject
    public ExternalAPINode(@Assisted Config config) {
        this.config = config;
        this.httpClient = HttpClient.newBuilder()
            .connectTimeout(Duration.ofMillis(config.connectionTimeout()))
            .build();
    }

    @Override
    public boolean process(TreeContext context) throws NodeProcessException {

        String username = context.sharedState.get("username").asString();

        try {
            HttpRequest request = HttpRequest.newBuilder()
                .uri(URI.create(config.apiEndpoint() + "/check"))
                .header("Authorization", "Bearer " + config.apiToken())
                .header("Content-Type", "application/json")
                .POST(HttpRequest.BodyPublishers.ofString(
                    String.format("{\"username\":\"%s\"}", username)
                ))
                .timeout(Duration.ofMillis(config.requestTimeout()))
                .build();

            HttpResponse<String> response = httpClient.send(request,
                HttpResponse.BodyHandlers.ofString());

            if (response.statusCode() == 200) {
                JSONObject json = new JSONObject(response.body());
                return json.getBoolean("allowed");
            } else {
                logger.warn("API returned non-200: {}", response.statusCode());
                return handleAPIError(response, context);
            }

        } catch (InterruptedException | IOException e) {
            logger.error("API call failed", e);
            return config.failOpen(); // Fail open or fail closed
        }
    }
}
```

**Pattern 2: Caching API responses**

```java
public class CachedAPINode extends AbstractDecisionNode {

    private final LoadingCache<String, Boolean> resultCache;

    @Inject
    public CachedAPINode(@Assisted Config config, APIClient apiClient) {
        this.config = config;

        // Caffeine cache — 10 minute TTL, max 10k entries
        this.resultCache = Caffeine.newBuilder()
            .expireAfterWrite(10, TimeUnit.MINUTES)
            .maximumSize(10_000)
            .build(username -> apiClient.checkUser(username));
    }

    @Override
    public boolean process(TreeContext context) throws NodeProcessException {
        String username = context.sharedState.get("username").asString();

        try {
            // Cache hit → no API call
            // Cache miss → calls API, caches result
            return resultCache.get(username);
        } catch (ExecutionException e) {
            throw new NodeProcessException("Cached API call failed", e.getCause());
        }
    }
}
```

**Pattern 3: Async API call with CompletableFuture**

```java
public class AsyncAPINode extends AbstractDecisionNode {

    private final APIClient apiClient;
    private final ExecutorService executor;

    @Inject
    public AsyncAPINode(@Assisted Config config, APIClient apiClient) {
        this.apiClient = apiClient;
        this.executor = Executors.newFixedThreadPool(10);
    }

    @Override
    public boolean process(TreeContext context) throws NodeProcessException {

        String username = context.sharedState.get("username").asString();

        try {
            CompletableFuture<Boolean> future = CompletableFuture.supplyAsync(
                () -> apiClient.checkUser(username),
                executor
            );

            // Block with timeout
            return future.get(config.timeoutMs(), TimeUnit.MILLISECONDS);

        } catch (TimeoutException e) {
            logger.warn("API call timed out for user: {}", username);
            return config.failOpen();
        } catch (Exception e) {
            throw new NodeProcessException("Async API call failed", e);
        }
    }

    @PreDestroy
    public void shutdown() {
        executor.shutdown();
    }
}
```

**Interview answer**: "For external APIs, I use Java 11's HttpClient with connection and request timeouts to prevent hanging authentication flows. I always implement fail-open/fail-closed logic — if the API is down, should we allow or deny? For high-traffic nodes, I use Caffeine cache to avoid hammering the external API. For truly async workflows, I use CompletableFuture with a thread pool, but I still block with a timeout because AM's tree model is synchronous. The key is defensive programming — APIs will fail, networks will timeout, so every external call needs error handling and fallback logic."

---

### Q15: ForgeRock Marketplace and node examples

**Answer**:

The **ForgeRock Marketplace** (now Ping Identity Marketplace) is a catalog of custom nodes, integrations, and solutions built by ForgeRock, partners, and the community.

**Access**: https://backstage.forgerock.com/marketplace/

#### **Notable Marketplace Nodes**:

**1. Socure Authentication Nodes**
- **Purpose**: Identity verification and fraud prevention
- **Nodes**: Socure ID+ Node (verify user attributes), Socure DeviceID Collector (device fingerprinting), Socure Predictive DocV (document verification)
- **Use case**: Verify government-issued IDs during registration
- **GitHub**: https://github.com/ForgeRock/Socure-Auth-Nodes

**2. RSA SecurID Authentication Nodes**
- **Purpose**: Integrate with RSA Cloud Authentication Service or RSA Authentication Manager
- **Use case**: Use existing RSA SecurID MFA infrastructure from AM trees
- **GitHub**: https://github.com/ForgeRock/Rsa-SecurId-Auth-Tree-Nodes

**3. Have I Been Pwned Node**
- **Purpose**: Check if a user's password appears in known data breaches
- **Use case**: Force password reset if compromised password detected
- **GitHub**: https://github.com/ForgeRock/haveibeenpwned-auth-tree-node

**4. Google reCAPTCHA Node**
- **Purpose**: Render Google reCAPTCHA widget to prevent bot attacks
- **Use case**: Add CAPTCHA challenge to login or registration trees

**5. Get Profile Property Node**
- **Purpose**: Fetch user profile attributes and copy to shared state
- **Use case**: Enrich shared state with user attributes for downstream nodes

**6. Knowledge-Based Authentication (KBA) Node**
- **Purpose**: Challenge users with security questions
- **Use case**: Step-up authentication or account recovery
- **GitHub**: https://github.com/ForgeRock/knowledge-based-auth-tree-node

**7. MFA SAML Selector Node**
- **Purpose**: Route MFA based on SAML `spEntityId`
- **Use case**: Different SPs require different MFA methods
- **GitHub**: https://github.com/ForgeRock/mfa-saml-selector-auth-tree-node

---

#### **How to use Marketplace nodes**:

1. **Browse catalog**: https://backstage.forgerock.com/marketplace/catalogDisplay
2. **Download JAR or clone GitHub repo**
3. **Build if necessary**: `mvn clean package`
4. **Deploy to AM**: Copy JAR to `WEB-INF/lib/`
5. **Restart AM**
6. **Configure in tree**: Node appears in Components pane

---

#### **Community Node Development**:

**ForgeRock encourages community contributions**:
- Open-source nodes on GitHub
- Share on Marketplace or ForgeRock Community
- Follow the authentication node development guide
- Use consistent naming and tagging

**Popular GitHub repos**:
- https://github.com/ForgeRock — Official nodes
- Search "forgerock auth node" on GitHub for community nodes

**Interview answer**: "The ForgeRock Marketplace is a catalog of pre-built authentication nodes — some from ForgeRock, some from partners like Socure and RSA, some from the community. Notable examples include the Have I Been Pwned node that checks if passwords are compromised, the Socure ID+ node for identity verification, and the RSA SecurID integration. All marketplace nodes are open-source on GitHub, so I can review the code before deploying. When I need a new integration, I check the Marketplace first before building from scratch. Deploying a marketplace node is the same as a custom node — download the JAR, copy to `WEB-INF/lib/`, restart AM."

**Sources**:
- [Marketplace catalog](https://backstage.forgerock.com/marketplace/api/catalog/entries/AWAnFI7lT81PD1YoPuXo)
- [Ping Identity Marketplace](https://backstage.forgerock.com/marketplace/)
- [Socure Auth Nodes GitHub](https://github.com/ForgeRock/Socure-Auth-Nodes)
- [RSA SecurID Nodes GitHub](https://github.com/ForgeRock/Rsa-SecurId-Auth-Tree-Nodes)

---

## Part 2: Amster (AM CLI Automation)

### Q16: What is Amster?

**Answer**:

**Amster** (Access Management Steroids) is a command-line interface for ForgeRock Access Management that allows administrators to configure, export, import, and automate AM deployments.

**Key features**:
- **Export/import configuration**: Realm config, trees, policies, OAuth2 clients, scripts
- **Scripting support**: Groovy-based automation
- **CI/CD integration**: Automate config promotion across environments (dev → staging → prod)
- **Built on REST API**: Wraps AM's REST API in a CLI

**Why "Steroids"**: Amster supercharges AM configuration management, especially for DevOps workflows.

---

#### **When to use Amster**:

| Use Case | Why Amster |
|---|---|
| **Environment promotion** | Export config from dev, import to staging/prod |
| **CI/CD pipelines** | Automate config deployment as part of release process |
| **Backup/restore** | Export entire realm config for disaster recovery |
| **Bulk operations** | Create 100 OAuth2 clients from a CSV file |
| **Infrastructure as Code** | Store AM config in Git, version control changes |
| **Cluster setup** | Quickly configure multiple AM instances identically |

---

#### **Installing Amster**:

**Download**:
```bash
# From Backstage (requires login)
wget https://backstage.forgerock.com/downloads/amster-7.4.0.zip

# Unzip
unzip amster-7.4.0.zip
cd amster

# Run
./amster
```

**Docker** (unofficial):
```bash
docker run -it --rm \
  -v $(pwd)/config:/config \
  forgerock/amster:7.4.0
```

---

#### **Basic Amster workflow**:

```groovy
// Connect to AM
:connect http://pingam:8081/am -k /path/to/admin/.keyId

// Export entire config
:export-config /tmp/am-config --failOnError true

// Import config to another instance
:connect http://staging-am:8081/am -k /path/to/admin/.keyId
:import-config /tmp/am-config --clean true --failOnError false

// Disconnect
:exit
```

**Interview answer**: "Amster is AM's CLI tool for configuration automation. It's built on the REST API and supports export/import of realm configs, trees, policies, scripts, and OAuth2 clients. I use it for environment promotion — export config from dev, check it into Git, then import to staging and prod as part of the CI/CD pipeline. It's also critical for disaster recovery — we export the entire config nightly and store it in S3. Amster supports Groovy scripting, so I can automate bulk operations like creating 50 OAuth2 clients from a CSV file."

**Sources**:
- [Amster User Guide](https://backstage.forgerock.com/docs/amster/7/user-guide/index.html)
- [Import Configuration Data](https://backstage.forgerock.com/docs/amster/7/user-guide/amster-import-config.html)
- [Export Configuration Data](https://backstage.forgerock.com/docs/amster/7/user-guide/amster-export-config.html)

---

### Q17: Amster vs REST API vs Console — when to use each?

**Answer**:

| Tool | When to Use | Pros | Cons |
|---|---|---|---|
| **AM Console (UI)** | Ad-hoc changes, learning, testing | Visual, intuitive, no scripting needed | Not automatable, prone to human error, slow for bulk ops |
| **REST API** | Custom automation, app integration | Full programmatic control, language-agnostic | Requires HTTP client, verbose JSON, no built-in export/import |
| **Amster (CLI)** | DevOps automation, config promotion, scripting | Export/import, Groovy scripting, repeatable, version-controllable | Learning curve, not interactive like Console |

---

#### **Scenario-based decision**:

**Scenario 1: Create one OAuth2 client for testing**
→ **Use Console**: Fastest for one-off tasks

**Scenario 2: Create 50 OAuth2 clients with different redirect URIs**
→ **Use Amster script**: Loop through CSV, create via scripting

**Scenario 3: Promote realm config from dev to prod**
→ **Use Amster**: Export-import workflow

**Scenario 4: App needs to create OAuth2 clients dynamically**
→ **Use REST API**: Called from application code (Java, Python, etc.)

**Scenario 5: Daily backup of AM config**
→ **Use Amster in cron job**: Scheduled export

---

#### **Real-world workflow**:

```
1. Developer changes tree in DEV → AM Console (visual testing)
2. Export DEV config → Amster export-config
3. Commit to Git → Version control
4. CI/CD pipeline runs → Amster import-config to STAGING
5. QA tests in STAGING → Automated tests
6. Promote to PROD → Amster import-config to PROD
7. Rollback if needed → Import previous config from Git
```

**Interview answer**: "I use the Console for learning and ad-hoc testing. For bulk operations like creating 50 OAuth2 clients or updating all trees in a realm, I use Amster scripts. For config promotion across environments, Amster's export/import is essential — I export the dev config, commit it to Git, and my CI/CD pipeline imports it to staging and prod. For custom application logic, like a registration portal that creates OAuth2 clients dynamically, I call the REST API directly. Each tool has its place — Console for humans, Amster for DevOps, REST API for apps."

---

(Continued in final response...)
