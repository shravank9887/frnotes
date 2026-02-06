# ForgeRock/PingAM Interview Questions — Custom Authentication Nodes, Amster, and Upgrades

---

## Part 1: Custom Authentication Nodes

### Q1: What is a custom authentication node and how does it differ from built-in nodes?

**Answer**:

A custom authentication node is a Java class that implements the `org.forgerock.openam.auth.node.api.Node` interface to extend AM's authentication framework with custom business logic.

| Built-in Nodes | Custom Nodes |
|---|---|
| Shipped with AM (Data Store Decision, Username Collector, etc.) | Developed by organizations or third parties |
| General-purpose authentication logic | Business-specific logic (e.g., call internal fraud API, query proprietary database) |
| Maintained by ForgeRock/Ping | Maintained by the developer |
| Source code not modifiable | Full control over implementation |

**Common use cases for custom nodes**:
- **External API integration**: Call a fraud detection service, CRM system, or risk scoring engine
- **Database lookups**: Query custom user attributes from legacy databases not in LDAP
- **Custom MFA**: Integrate with proprietary MFA systems
- **Business rules**: Implement complex conditional logic specific to the organization (e.g., "if VIP customer from EU during business hours, skip MFA")
- **Attribute enrichment**: Fetch and add user attributes to shared state from external systems

**Interview answer**: "Custom authentication nodes let us extend AM's authentication trees with organization-specific logic. For example, at my last company, we built a custom node that called our internal risk scoring API and adjusted the authentication flow based on the risk level — low risk users got straight-through authentication, high risk users were forced through MFA. Built-in nodes couldn't do this because the risk engine was proprietary."

---

### Q2: Explain the Java API for custom authentication nodes — AbstractDecisionNode, @Attribute, TreeContext, Action, NodeProcessException

**Answer**:

The custom authentication node API consists of several core components:

#### **1. Node Interface**
The base interface all nodes must implement:
```java
public interface Node {
    Action process(TreeContext context) throws NodeProcessException;
}
```

#### **2. Abstract Base Classes**

| Class | When to Use | Outcome Provider |
|---|---|---|
| `SingleOutcomeNode` | Nodes with one exit path (e.g., attribute collector) | Built-in single outcome |
| `AbstractDecisionNode` | Boolean decision nodes (true/false, yes/no, allow/deny) | Built-in true/false outcomes |
| Direct `Node` implementation | Multi-outcome or complex flow control | Custom OutcomeProvider |

**Example extending AbstractDecisionNode**:
```java
@Node.Metadata(
    outcomeProvider = AbstractDecisionNode.OutcomeProvider.class,
    configClass = MyNode.Config.class,
    tags = {"risk", "security"}
)
public class RiskScoringNode extends AbstractDecisionNode {

    @Override
    public Action process(TreeContext context) throws NodeProcessException {
        // Get username from shared state
        String username = context.sharedState.get("username").asString();

        // Call risk API
        int riskScore = callRiskAPI(username);

        // Store score in shared state for downstream nodes
        return goTo(riskScore > 75).replaceSharedState(
            context.sharedState.put("riskScore", riskScore)
        ).build();
    }

    public interface Config {
        @Attribute(order = 10)
        default int riskThreshold() {
            return 75;
        }
    }
}
```

#### **3. @Attribute Annotation**
Used in the Config interface to define configurable properties that appear in the AM Console tree designer:

```java
public interface Config {
    @Attribute(order = 10)
    String apiEndpoint();

    @Attribute(order = 20, validators = SessionPropertyValidator.class)
    default int timeout() {
        return 5000;
    }

    @Attribute(order = 30)
    default boolean enableCaching() {
        return false;
    }
}
```

**Key points**:
- `order` determines the display position in the UI
- `validators` can enforce value constraints (DecimalValidator, HMACKeyLengthValidator, etc.)
- Use `default` methods for optional properties with defaults

#### **4. TreeContext**
Provides access to all authentication state and request context:

```java
public class TreeContext {
    JsonValue sharedState;           // Non-sensitive state shared between nodes
    JsonValue transientState;        // Sensitive state (encrypted, not sent to client)
    HttpServletRequest request;      // HTTP request details
    List<Callback> callbacks;        // Callbacks from previous nodes
    // ... and more
}
```

**Common TreeContext methods**:
```java
// Get shared state value
String username = context.sharedState.get("username").asString();

// Get all callbacks of a specific type
List<NameCallback> nameCallbacks = context.getCallbacks(NameCallback.class);

// Access request headers
String userAgent = context.request.getHeader("User-Agent");
```

#### **5. Action**
Returned by `process()` to control tree flow and state changes:

```java
// Route to "true" outcome, update shared state
return goTo(true)
    .replaceSharedState(context.sharedState.put("authenticated", true))
    .build();

// Add callbacks to collect user input
return Action.send(
    new NameCallback("Enter username"),
    new PasswordCallback("Enter password", false)
).build();

// Route to custom outcome
return Action.goTo("high-risk")
    .putSessionProperty("riskLevel", "HIGH")
    .build();
```

**Action builder methods**:
- `.replaceSharedState()` — update non-sensitive shared state
- `.replaceTransientState()` — update sensitive transient state
- `.putSessionProperty()` — set AM session property
- `.addSessionHook()` — register session lifecycle hooks

#### **6. NodeProcessException**
Thrown when a catastrophic error occurs that should halt the authentication journey:

```java
if (config.apiEndpoint() == null || config.apiEndpoint().isEmpty()) {
    throw new NodeProcessException("API endpoint is required");
}

try {
    callExternalAPI();
} catch (IOException e) {
    // Option 1: Throw exception (harsh — generic error to user)
    throw new NodeProcessException("Failed to connect to risk API", e);

    // Option 2: Handle gracefully (better UX — route to error outcome)
    return goTo("error")
        .replaceSharedState(context.sharedState.put("errorMessage", e.getMessage()))
        .build();
}
```

**Best practice**: Avoid throwing `NodeProcessException` when possible. Instead, add an "error" outcome and route there, storing error details in shared state for downstream logging nodes. This provides a better user experience than a generic error page.

**Interview answer**: "The core API is centered around implementing `process(TreeContext context)` which returns an `Action`. TreeContext gives you access to shared state, transient state, callbacks, and the HTTP request. For simple nodes, you extend `AbstractDecisionNode` for boolean outcomes or `SingleOutcomeNode` for single-path flows. The `@Attribute` annotation in a Config interface lets you define properties that show up in the AM Console. I avoid throwing `NodeProcessException` unless it's truly catastrophic — instead, I add error outcomes and store error details in shared state for graceful handling."

---

### Q3: How do you create a new authentication node project using the Maven archetype?

**Answer**:

ForgeRock provides the `auth-tree-node-archetype` Maven archetype to scaffold a starter project.

**Prerequisites**:
1. **Maven 3.2.5+** and **JDK 11+** installed
2. **Backstage credentials** configured in `~/.m2/settings.xml` for access to ForgeRock's proprietary repositories
3. Access to the `forgerock-private-releases` repository (for custom node development)

**Step-by-step process**:

```bash
# 1. Navigate to your workspace
cd ~/projects

# 2. Run the archetype
mvn archetype:generate \
  -DarchetypeGroupId=org.forgerock.am \
  -DarchetypeArtifactId=auth-tree-node-archetype \
  -DarchetypeVersion=7.4.0

# 3. The archetype will prompt for:
#    - groupId: com.example (your domain in reverse)
#    - artifactId: risk-scoring-node (JAR name, project folder name)
#    - version: 1.0.0-SNAPSHOT
#    - package: com.example.nodes (Java package for generated classes)
```

**What gets generated**:
```
risk-scoring-node/
├── pom.xml                    # Maven build configuration
├── src/
│   └── main/
│       └── java/
│           └── com/
│               └── example/
│                   └── nodes/
│                       ├── RiskScoringNode.java      # Node implementation
│                       └── RiskScoringNodePlugin.java # Plugin class
└── README.md
```

**Alternative approach — Clone existing nodes**:
```bash
# ForgeRock open-sources many nodes on GitHub
git clone https://github.com/ForgeRock/knowledge-based-auth-tree-node.git
cd knowledge-based-auth-tree-node

# Modify the Java classes to match your requirements
# Rename packages, change groupId/artifactId in pom.xml
```

**pom.xml key sections**:
```xml
<dependencies>
    <!-- AM node API (mark as "provided" — AM supplies this at runtime) -->
    <dependency>
        <groupId>org.forgerock.am</groupId>
        <artifactId>auth-node-api</artifactId>
        <version>7.4.0</version>
        <scope>provided</scope>
    </dependency>

    <!-- External dependencies (repackage inside your JAR) -->
    <dependency>
        <groupId>com.google.guava</groupId>
        <artifactId>guava</artifactId>
        <version>31.1-jre</version>
    </dependency>
</dependencies>

<build>
    <plugins>
        <!-- Shade plugin to bundle external dependencies -->
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-shade-plugin</artifactId>
            <version>3.4.1</version>
            <executions>
                <execution>
                    <phase>package</phase>
                    <goals>
                        <goal>shade</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

**Best practices**:
- Mark all ForgeRock AM dependencies as `<scope>provided</scope>` to avoid classpath conflicts
- Use the Maven Shade plugin to repackage external dependencies inside your JAR
- Version your custom node independently from AM (e.g., `1.0.0`, `1.1.0`) for tracking across AM upgrades

**Interview answer**: "I use the `auth-tree-node-archetype` to generate the starter project structure. After configuring my Backstage credentials in Maven settings, I run `mvn archetype:generate` and provide the groupId, artifactId, and package. The archetype creates a complete Maven project with the node class, plugin class, and pom.xml preconfigured. An alternative is to clone one of ForgeRock's open-source node examples from GitHub and modify it — that's faster when you want to learn from a working example."

**Sources**:
- [Prepare for development (AM 7.3.1)](https://backstage.forgerock.com/docs/am/7.3/auth-nodes/preparing-for-nodes.html)
- [Prepare for development (AM 7.4.1)](https://backstage.forgerock.com/docs/am/7.4/auth-nodes/preparing-for-nodes.html)

---

### Q4: Explain the node lifecycle — how does AM load custom nodes as OSGi plugins?

**Answer**:

Custom authentication nodes are deployed as **JAR files** containing both the node implementation and a **plugin class** that registers the node with AM at runtime.

#### **Deployment Process**:

```
1. Build JAR       →  mvn clean package
2. Copy JAR        →  WEB-INF/lib/ in AM webapp
3. Restart Tomcat  →  AM discovers and loads node
4. Node appears    →  AM Console tree designer Components pane
```

**Technical details**:

#### **1. Plugin Class — the entry point**
Every custom node JAR must include a Plugin class that extends `AbstractNodeAmPlugin`:

```java
package com.example.nodes;

import org.forgerock.openam.auth.node.api.AbstractNodeAmPlugin;

public class RiskScoringNodePlugin extends AbstractNodeAmPlugin {

    // Guice module for dependency injection
    @Override
    protected void setupNodeGuiceModule(Binder binder) {
        binder.bind(MyRiskService.class).to(MyRiskServiceImpl.class);
    }

    // Optional: return node classes if not using annotations
    @Override
    public Map<String, Class<? extends Node>> getNodesByName() {
        return ImmutableMap.of(
            "RiskScoringNode", RiskScoringNode.class
        );
    }
}
```

#### **2. META-INF/services discovery**
The JAR must contain a service provider configuration file that tells AM about the plugin:

**File**: `src/main/resources/META-INF/services/org.forgerock.openam.plugins.AmPlugin`

**Content**: Fully qualified class name of your plugin
```
com.example.nodes.RiskScoringNodePlugin
```

This uses the **Java ServiceLoader** mechanism — AM scans all JARs for this file at startup.

#### **3. Startup sequence**:

```
Tomcat starts
    ↓
AM webapp initializes
    ↓
AM scans WEB-INF/lib/*.jar
    ↓
Finds META-INF/services/org.forgerock.openam.plugins.AmPlugin
    ↓
Instantiates RiskScoringNodePlugin
    ↓
Calls setupNodeGuiceModule() → registers dependencies
    ↓
Reads @Node.Metadata annotations → registers node types
    ↓
Node appears in AM Console Components pane
```

#### **4. Why this isn't true OSGi**
While AM historically used **OSGi** (Apache Felix), modern AM (7.x+) uses a **simplified plugin model** that mimics OSGi bundles but doesn't require full OSGi bundle metadata. You don't need `MANIFEST.MF` OSGi headers — just the service provider file.

**Older AM versions (5.x, 6.x)**: True OSGi bundles with `Bundle-Activator` in `MANIFEST.MF`
**Modern AM (7.x+)**: Simplified plugin model using Java ServiceLoader

#### **5. Hot-reload support**
AM does **not** support hot-reloading custom nodes. Changes require:
1. Stop AM (Tomcat)
2. Replace JAR in WEB-INF/lib/
3. Restart AM

In development, this can be automated with a script:
```bash
#!/bin/bash
# deploy-node.sh
mvn clean package
docker cp target/risk-scoring-node-1.0.0.jar pingam:/opt/tomcat/webapps/am/WEB-INF/lib/
docker restart pingam
```

#### **Dependency injection with Guice**
AM uses **Google Guice** for dependency injection. In the plugin's `setupNodeGuiceModule()`, you can bind interfaces to implementations:

```java
@Override
protected void setupNodeGuiceModule(Binder binder) {
    // Bind a service interface to an implementation
    binder.bind(RiskScoringService.class).to(RiskScoringServiceImpl.class);

    // Singleton binding
    binder.bind(CacheService.class).to(CacheServiceImpl.class).in(Singleton.class);
}
```

Then inject into your node:
```java
public class RiskScoringNode extends AbstractDecisionNode {

    private final RiskScoringService riskService;

    @Inject
    public RiskScoringNode(@Assisted Config config, RiskScoringService riskService) {
        this.config = config;
        this.riskService = riskService;
    }
}
```

**Interview answer**: "Custom nodes are packaged as JAR files with a Plugin class that extends `AbstractNodeAmPlugin`. The JAR must include a service provider file at `META-INF/services/org.forgerock.openam.plugins.AmPlugin` containing the fully qualified plugin class name. At startup, AM uses Java's ServiceLoader to discover all plugins in `WEB-INF/lib/`, instantiates them, and registers the node types. It's not true OSGi in AM 7+ — it's a simplified plugin model. Deployment requires restarting AM because there's no hot-reload support. For dependency injection, I use Guice bindings in the plugin's `setupNodeGuiceModule()` method."

**Sources**:
- [Plugin class (PingAM 8)](https://docs.pingidentity.com/pingam/8/auth-nodes/plugin-class.html)
- [Prepare for development (PingAM 8)](https://docs.pingidentity.com/pingam/8/auth-nodes/preparing-for-nodes.html)

---

### Q5: What are the key interfaces and classes — Node, SingleOutcomeNode, AbstractDecisionNode?

**Answer**:

The authentication node API provides a hierarchy of interfaces and abstract classes to simplify node development:

#### **1. Node Interface (base)**
```java
public interface Node {
    /**
     * Main processing method — called by AM during tree execution
     * @param context Provides access to shared state, callbacks, HTTP request
     * @return Action defining next outcome and state changes
     * @throws NodeProcessException on catastrophic errors
     */
    Action process(TreeContext context) throws NodeProcessException;
}
```

**When to implement directly**: Multi-outcome nodes or complex flow control.

---

#### **2. SingleOutcomeNode (abstract class)**
For nodes with **one exit path** (collectors, modifiers, side-effect nodes):

```java
public abstract class SingleOutcomeNode implements Node {

    @Override
    public final Action process(TreeContext context) throws NodeProcessException {
        // Calls your implementation
        return goToNext().build();
    }

    // Implement this instead of process()
    public abstract Action process(TreeContext context);

    // Built-in outcome provider — always returns "outcome" outcome
    public static class OutcomeProvider implements org.forgerock.openam.auth.node.api.OutcomeProvider {
        @Override
        public List<Outcome> getOutcomes(...) {
            return ImmutableList.of(new Outcome("outcome", "Outcome"));
        }
    }
}
```

**Use cases**:
- **Attribute collectors**: Set Session Properties Node
- **Modifiers**: Modify Auth Level Node
- **Side effects**: Logging Node, Analytics Node

**Example — Log Username Node**:
```java
@Node.Metadata(
    outcomeProvider = SingleOutcomeNode.OutcomeProvider.class,
    configClass = LogUsernameNode.Config.class
)
public class LogUsernameNode extends SingleOutcomeNode {

    private final Logger logger = LoggerFactory.getLogger(LogUsernameNode.class);

    @Override
    public Action process(TreeContext context) {
        String username = context.sharedState.get("username").asString();
        logger.info("User authenticating: {}", username);

        // Always goes to the single outcome
        return goToNext().build();
    }
}
```

---

#### **3. AbstractDecisionNode (abstract class)**
For **boolean decision nodes** (true/false, yes/no, allow/deny):

```java
public abstract class AbstractDecisionNode implements Node {

    @Override
    public final Action process(TreeContext context) throws NodeProcessException {
        // Your implementation returns boolean
        boolean decision = process(context);
        return goTo(decision).build();
    }

    // Implement this — return true or false
    public abstract boolean process(TreeContext context) throws NodeProcessException;

    // Built-in outcome provider — returns "true" and "false" outcomes
    public static class OutcomeProvider implements org.forgerock.openam.auth.node.api.OutcomeProvider {
        @Override
        public List<Outcome> getOutcomes(...) {
            return ImmutableList.of(
                new Outcome("true", "True"),
                new Outcome("false", "False")
            );
        }
    }
}
```

**Use cases**:
- **Authentication checks**: Data Store Decision Node
- **Conditional logic**: Account Active Decision Node
- **Risk decisions**: Is Risk High Node

**Example — VIP User Check Node**:
```java
@Node.Metadata(
    outcomeProvider = AbstractDecisionNode.OutcomeProvider.class,
    configClass = VIPCheckNode.Config.class
)
public class VIPCheckNode extends AbstractDecisionNode {

    @Inject
    private CoreWrapper coreWrapper;

    @Override
    public boolean process(TreeContext context) throws NodeProcessException {
        String username = context.sharedState.get("username").asString();

        try {
            AMIdentity identity = coreWrapper.getIdentity(username, realm);
            Set<String> groups = identity.getMemberOf(IdType.GROUP);

            // Return true if user is in "vip-users" group
            return groups.stream()
                .anyMatch(dn -> dn.contains("cn=vip-users"));
        } catch (Exception e) {
            throw new NodeProcessException("Failed to check VIP status", e);
        }
    }
}
```

---

#### **4. Comparison Table**

| Aspect | Node (interface) | SingleOutcomeNode | AbstractDecisionNode |
|---|---|---|---|
| **Outcomes** | Custom (you define) | One: "outcome" | Two: "true", "false" |
| **Outcome Provider** | Must provide custom | Built-in | Built-in |
| **process() returns** | `Action` | `Action` | `boolean` |
| **When to use** | Multi-outcome, complex flow | Collectors, modifiers | Boolean decisions |
| **Examples** | LDAP Decision (3 outcomes: success, locked, expired) | Set Session Properties | Data Store Decision |

---

#### **5. When to use each**

**Use SingleOutcomeNode when**:
- Collecting user input (callbacks)
- Modifying state (auth level, session properties)
- Performing side effects (logging, analytics)
- No conditional branching needed

**Use AbstractDecisionNode when**:
- Binary decision logic (yes/no, allow/deny)
- User exists / doesn't exist
- Account locked / not locked
- Risk high / low (with threshold)

**Implement Node directly when**:
- You need 3+ outcomes (e.g., low/medium/high risk)
- Complex flow control
- Dynamic outcomes based on config

**Example — Multi-outcome Risk Node**:
```java
@Node.Metadata(
    outcomeProvider = RiskNode.RiskOutcomeProvider.class,
    configClass = RiskNode.Config.class
)
public class RiskNode implements Node {

    @Override
    public Action process(TreeContext context) throws NodeProcessException {
        int score = calculateRiskScore(context);

        if (score < 30) return goTo("low").build();
        if (score < 70) return goTo("medium").build();
        return goTo("high").build();
    }

    // Custom outcome provider
    public static class RiskOutcomeProvider implements OutcomeProvider {
        @Override
        public List<Outcome> getOutcomes(...) {
            return ImmutableList.of(
                new Outcome("low", "Low Risk"),
                new Outcome("medium", "Medium Risk"),
                new Outcome("high", "High Risk")
            );
        }
    }
}
```

**Interview answer**: "For simple nodes, I extend `SingleOutcomeNode` if there's one path (like logging or setting a session property) or `AbstractDecisionNode` if it's a boolean decision (like checking if a user exists). These provide built-in outcome providers so I don't have to implement my own. For nodes with 3+ outcomes — like risk scoring with low/medium/high paths — I implement the `Node` interface directly and provide a custom `OutcomeProvider`. This approach keeps the code clean and leverages the framework's abstractions where possible."

**Sources**:
- [The Node Class (AM 7.0.2)](https://backstage.forgerock.com/docs/am/7/auth-nodes/core-class.html)
- [The Node Class (AM 7.1.4)](https://backstage.forgerock.com/docs/am/7.1/auth-nodes/core-class.html)
- [About Authentication Nodes](https://backstage.forgerock.com/docs/am/7/auth-nodes/about-nodes.html)

---

### Q6: How do you define outcomes using OutcomeProvider?

**Answer**:

The `OutcomeProvider` interface tells AM what exit paths a node has and how they appear in the tree designer.

#### **OutcomeProvider Interface**:
```java
public interface OutcomeProvider {
    /**
     * Returns the list of outcomes this node can produce
     * @param preReqCheck: pre-requisite check for node viability
     * @param nodeType: the node class
     * @return List of Outcome objects
     */
    List<Outcome> getOutcomes(PreReqCheck preReqCheck, Class<? extends Node> nodeType);
}
```

#### **Outcome Class**:
```java
public class Outcome {
    private final String id;          // Internal identifier (used in tree JSON)
    private final String displayName; // Label shown in AM Console

    public Outcome(String id, String displayName) {
        this.id = id;
        this.displayName = displayName;
    }
}
```

---

#### **Approach 1: Built-in Outcome Providers**

**SingleOutcomeNode.OutcomeProvider** — for nodes with one exit:
```java
@Node.Metadata(
    outcomeProvider = SingleOutcomeNode.OutcomeProvider.class,
    configClass = LoggingNode.Config.class
)
public class LoggingNode extends SingleOutcomeNode {
    // ... produces outcome "outcome"
}
```

**AbstractDecisionNode.OutcomeProvider** — for boolean decisions:
```java
@Node.Metadata(
    outcomeProvider = AbstractDecisionNode.OutcomeProvider.class,
    configClass = AccountActiveNode.Config.class
)
public class AccountActiveNode extends AbstractDecisionNode {
    // ... produces outcomes "true" and "false"
}
```

---

#### **Approach 2: Custom OutcomeProvider for Multi-Outcome Nodes**

When you need 3+ outcomes or custom outcome names:

```java
@Node.Metadata(
    outcomeProvider = RiskScoringNode.RiskOutcomeProvider.class,
    configClass = RiskScoringNode.Config.class
)
public class RiskScoringNode implements Node {

    @Override
    public Action process(TreeContext context) throws NodeProcessException {
        int riskScore = calculateRisk(context);

        if (riskScore < config.lowThreshold()) {
            return goTo("low-risk").build();
        } else if (riskScore < config.highThreshold()) {
            return goTo("medium-risk").build();
        } else {
            return goTo("high-risk").build();
        }
    }

    // Custom OutcomeProvider inner class
    public static class RiskOutcomeProvider implements OutcomeProvider {

        @Override
        public List<Outcome> getOutcomes(PreReqCheck preReqCheck,
                                         Class<? extends Node> nodeType) {
            return ImmutableList.of(
                new Outcome("low-risk", "Low Risk"),
                new Outcome("medium-risk", "Medium Risk"),
                new Outcome("high-risk", "High Risk")
            );
        }
    }

    public interface Config {
        @Attribute(order = 10)
        default int lowThreshold() { return 30; }

        @Attribute(order = 20)
        default int highThreshold() { return 70; }
    }
}
```

**How this appears in AM Console Tree Designer**:
```
┌──────────────────────┐
│  Risk Scoring Node   │
├──────────────────────┤
│ ● Low Risk           │─────→ [Next node for low risk]
│ ● Medium Risk        │─────→ [Next node for medium risk]
│ ● High Risk          │─────→ [Next node for high risk]
└──────────────────────┘
```

---

#### **Approach 3: Dynamic Outcomes from Config**

Some nodes determine outcomes based on configuration (e.g., Choice Collector Node):

```java
public static class ChoiceOutcomeProvider implements OutcomeProvider {

    @Override
    public List<Outcome> getOutcomes(PreReqCheck preReqCheck,
                                     Class<? extends Node> nodeType) {

        // Read the node's Config to get choices
        ChoiceNode.Config config = preReqCheck.getConfig(ChoiceNode.Config.class);

        List<Outcome> outcomes = new ArrayList<>();
        for (String choice : config.choices()) {
            outcomes.add(new Outcome(choice, choice));
        }

        // Add default "unknown" outcome for error handling
        outcomes.add(new Outcome("unknown", "Unknown"));

        return outcomes;
    }
}

public interface Config {
    @Attribute(order = 10)
    List<String> choices();  // Admin configures: ["Login", "Register", "Forgot Password"]
}
```

This creates outcomes dynamically based on the admin's configuration in the AM Console.

---

#### **Real-World Example: MFA Method Selector**

```java
@Node.Metadata(
    outcomeProvider = MFAMethodSelectorNode.MFAOutcomeProvider.class,
    configClass = MFAMethodSelectorNode.Config.class,
    tags = {"mfa", "authentication"}
)
public class MFAMethodSelectorNode implements Node {

    @Override
    public Action process(TreeContext context) throws NodeProcessException {
        // Get user's registered MFA methods from LDAP
        Set<String> registeredMethods = getUserMFAMethods(context);

        if (registeredMethods.contains("TOTP")) {
            return goTo("totp").build();
        } else if (registeredMethods.contains("WebAuthn")) {
            return goTo("webauthn").build();
        } else if (registeredMethods.contains("SMS")) {
            return goTo("sms").build();
        } else {
            // No MFA registered — route to registration
            return goTo("no-mfa").build();
        }
    }

    public static class MFAOutcomeProvider implements OutcomeProvider {
        @Override
        public List<Outcome> getOutcomes(PreReqCheck preReqCheck,
                                         Class<? extends Node> nodeType) {
            return ImmutableList.of(
                new Outcome("totp", "TOTP (Authenticator App)"),
                new Outcome("webauthn", "WebAuthn (Biometric)"),
                new Outcome("sms", "SMS OTP"),
                new Outcome("no-mfa", "No MFA Registered")
            );
        }
    }
}
```

---

#### **Best Practices**

| Practice | Rationale |
|---|---|
| Use descriptive outcome IDs | `high-risk` vs `outcome3` — clearer in tree JSON |
| Provide user-friendly display names | Tree designer shows these to admins |
| Include error outcomes | `error`, `unknown`, `timeout` for graceful degradation |
| Keep outcomes stable | Changing outcome IDs breaks existing trees |
| Document outcome conditions | JavaDoc explaining when each outcome fires |

**Interview answer**: "The `OutcomeProvider` defines what exit paths a node has. For simple cases, I use the built-in providers from `SingleOutcomeNode` or `AbstractDecisionNode`. For multi-outcome nodes, I implement a custom `OutcomeProvider` as a static inner class. The `getOutcomes()` method returns a list of `Outcome` objects, each with an ID (used internally) and a display name (shown in the tree designer). I always include error outcomes like 'unknown' or 'timeout' for graceful degradation. The Choice Collector Node is a great example of dynamic outcomes — it reads the node's config to generate outcomes at runtime."

**Sources**:
- [The Node Class (AM 7.0.2)](https://backstage.forgerock.com/docs/am/7/auth-nodes/core-class.html)
- [Metadata annotation (PingAM 8.0.1)](https://backstage.forgerock.com/docs/am/8/auth-nodes/core-metadata.html)

---

### Q7: How do you access shared state, transient state, and callbacks in a node?

**Answer**:

Authentication state flows through the tree via three mechanisms: **shared state** (non-sensitive data), **transient state** (sensitive data), and **callbacks** (user interaction).

#### **1. Shared State — Non-Sensitive Data**

Shared state is stored in a `JsonValue` object and is **visible in the authId JWT** sent to the client. Use it for non-sensitive data like username, selected language, or risk scores.

**Reading from shared state**:
```java
public Action process(TreeContext context) {
    // Get string value
    String username = context.sharedState.get("username").asString();

    // Get integer value with default
    int authLevel = context.sharedState.get("authLevel").defaultTo(0).asInt();

    // Check if key exists
    if (context.sharedState.isDefined("riskScore")) {
        int riskScore = context.sharedState.get("riskScore").asInt();
    }

    // Get object/map
    JsonValue userProfile = context.sharedState.get("userProfile");
    String email = userProfile.get("mail").asString();
}
```

**Writing to shared state**:
```java
public Action process(TreeContext context) {
    // Add or update a value
    return goTo(true)
        .replaceSharedState(
            context.sharedState
                .put("username", "demo")
                .put("authLevel", 2)
                .put("riskScore", 45)
        )
        .build();
}
```

**Common shared state keys** (by convention):
| Key | Purpose | Set By |
|---|---|---|
| `username` | User's login name | Username Collector, Data Store Decision |
| `realm` | Authentication realm | Start node |
| `authLevel` | Current authentication level | Authenticators, Modify Auth Level |
| `objectAttributes` | User profile attributes | Data Store Decision |
| `successUrl` | Post-auth redirect URL | URL parameter |

---

#### **2. Transient State — Sensitive Data**

Transient state is **encrypted** and **never sent to the client**. Use it for passwords, tokens, PII, or any data that should not be visible in browser network traces.

**Reading from transient state**:
```java
public Action process(TreeContext context) {
    // Get password entered by user
    String password = context.transientState.get("password").asString();

    // Get OTP code
    String otp = context.transientState.get("otpCode").asString();
}
```

**Writing to transient state**:
```java
public Action process(TreeContext context) {
    // Store sensitive data
    return goTo(true)
        .replaceTransientState(
            context.transientState
                .put("password", "P@ssw0rd!")
                .put("apiToken", "eyJhbGc...")
        )
        .build();
}
```

**Important**: Transient state is **promoted to "secure state"** (encrypted shared state) when:
- A callback to the user is about to occur
- A downstream node requests transient state data as script input

This means transient data can be passed between nodes in the same request, but if the tree needs to collect more input from the user, the data is encrypted before being included in the authId JWT.

---

#### **3. Callbacks — User Interaction**

Callbacks are how nodes collect input from users. The `TreeContext` provides access to callbacks submitted by previous nodes.

**Getting all callbacks**:
```java
public Action process(TreeContext context) {
    List<Callback> allCallbacks = context.getAllCallbacks();

    for (Callback callback : allCallbacks) {
        if (callback instanceof NameCallback) {
            NameCallback nameCallback = (NameCallback) callback;
            String username = nameCallback.getName();
        }
    }
}
```

**Getting callbacks by type** (AM 7.3+):
```java
public Action process(TreeContext context) {
    // Get all NameCallbacks
    List<NameCallback> nameCallbacks = context.getCallbacks(NameCallback.class);

    if (!nameCallbacks.isEmpty()) {
        String username = nameCallbacks.get(0).getName();
    }

    // Get PasswordCallbacks
    List<PasswordCallback> passwordCallbacks = context.getCallbacks(PasswordCallback.class);

    if (!passwordCallbacks.isEmpty()) {
        char[] password = passwordCallbacks.get(0).getPassword();
        String passwordStr = new String(password);
    }
}
```

**Sending callbacks to the user**:
```java
public Action process(TreeContext context) {
    // If no username in shared state, collect it
    if (!context.sharedState.isDefined("username")) {
        return Action.send(
            new NameCallback("Username"),
            new PasswordCallback("Password", false)
        ).build();
    }

    // Username exists, proceed to authentication
    String username = context.sharedState.get("username").asString();
    // ... authenticate
}
```

**Common callback types**:
| Callback Type | Purpose | Example |
|---|---|---|
| `NameCallback` | Collect username or name | Login forms |
| `PasswordCallback` | Collect password | Login forms |
| `ChoiceCallback` | Multiple choice selection | MFA method selector |
| `ConfirmationCallback` | Yes/No confirmation | Terms acceptance |
| `TextOutputCallback` | Display message | Error messages, instructions |
| `HiddenValueCallback` | Hidden form fields | CSRF tokens, state |
| `HttpCallback` | Redirect to external URL | OAuth2, SAML |
| `MetadataCallback` | Metadata (e.g., device info) | Risk assessment |

---

#### **Real-World Pattern: Collect Username, Lookup in LDAP, Enrich Shared State**

```java
@Node.Metadata(outcomeProvider = SingleOutcomeNode.OutcomeProvider.class)
public class UserEnrichmentNode extends SingleOutcomeNode {

    @Inject
    private CoreWrapper coreWrapper;

    @Override
    public Action process(TreeContext context) throws NodeProcessException {

        // 1. Check if username in shared state
        if (!context.sharedState.isDefined("username")) {
            // Collect it via callback
            return Action.send(new NameCallback("Username")).build();
        }

        // 2. Get username from callback (just submitted)
        String username = null;
        List<NameCallback> nameCallbacks = context.getCallbacks(NameCallback.class);
        if (!nameCallbacks.isEmpty()) {
            username = nameCallbacks.get(0).getName();
        } else {
            // Or from shared state (set by previous node)
            username = context.sharedState.get("username").asString();
        }

        // 3. Lookup user in LDAP
        try {
            AMIdentity identity = coreWrapper.getIdentity(username, realm);

            // 4. Enrich shared state with user attributes
            JsonValue sharedState = context.sharedState.copy()
                .put("username", username)
                .put("cn", identity.getAttribute("cn").iterator().next())
                .put("mail", identity.getAttribute("mail").iterator().next())
                .put("telephoneNumber", identity.getAttribute("telephoneNumber").iterator().next());

            return goToNext().replaceSharedState(sharedState).build();

        } catch (IdRepoException | SSOException e) {
            throw new NodeProcessException("User not found: " + username, e);
        }
    }
}
```

---

#### **State Precedence**

When retrieving a value, check in this order:
1. **Transient state** (most recent, sensitive)
2. **Secure state** (promoted transient, encrypted)
3. **Shared state** (non-sensitive, visible)

Some helper methods do this automatically:
```java
// This checks transient → secure → shared, returns first match
String value = context.sharedState.get("key").asString();
```

---

#### **Best Practices**

| Practice | Rationale |
|---|---|---|
| Never put passwords in shared state | Shared state is visible in authId JWT — use transient state |
| Use descriptive keys | `username` not `u`, `riskScore` not `rs` |
| Validate callback input | Check for null, validate format before using |
| Clear sensitive data after use | Overwrite password char arrays with zeros |
| Store callback positions | If reusing callbacks from previous nodes, store positions in shared state |

**Interview answer**: "I use shared state for non-sensitive data like username, auth level, or risk scores — it's a `JsonValue` that flows through the tree. For sensitive data like passwords or tokens, I use transient state, which is encrypted and never sent to the client. Callbacks are how I interact with users — I check if data is already in shared state first, and if not, I send callbacks to collect it. After the user submits, I retrieve the callback values using `context.getCallbacks()` and store them appropriately in shared or transient state. For example, username goes in shared state, password goes in transient state."

**Sources**:
- [Action class (AM 7.4.1)](https://backstage.forgerock.com/docs/am/7.4/auth-nodes/core-action.html)
- [Scripted decision node API](https://backstage.forgerock.com/docs/am/7.1/authentication-guide/scripting-api-node.html)
- [NodeState API documentation](https://docs.pingidentity.com/pingoneaic/latest/_attachments/apidocs/org/forgerock/openam/auth/node/api/NodeState.html)

---

### Q8: How do you configure node properties using @Attribute in the Config interface?

**Answer**:

The `Config` interface (nested inside the node class) defines configurable properties that appear in the AM Console tree designer when an admin adds the node to a tree.

#### **Basic Config Interface**:
```java
@Node.Metadata(
    outcomeProvider = AbstractDecisionNode.OutcomeProvider.class,
    configClass = RiskScoringNode.Config.class
)
public class RiskScoringNode extends AbstractDecisionNode {

    private final Config config;

    @Inject
    public RiskScoringNode(@Assisted Config config) {
        this.config = config;
    }

    @Override
    public boolean process(TreeContext context) {
        // Use config values
        String apiUrl = config.riskApiUrl();
        int timeout = config.timeout();

        // ... call risk API
    }

    /**
     * Configuration for Risk Scoring Node
     */
    public interface Config {

        @Attribute(order = 10)
        String riskApiUrl();

        @Attribute(order = 20)
        default int timeout() {
            return 5000;
        }

        @Attribute(order = 30)
        default int highRiskThreshold() {
            return 75;
        }
    }
}
```

---

#### **@Attribute Annotation Fields**

```java
public @interface Attribute {
    int order();                          // Required: display order in UI
    Class<?>[] validators() default {};   // Optional: validation classes
}
```

**order**: Controls the position in the AM Console config panel:
- `order = 10` appears first
- `order = 20` appears second
- Common practice: use increments of 10 to allow inserting fields later

**validators**: Array of validator classes from `org.forgerock.openam.auth.nodes.validators`:
- `DecimalValidator` — validates numeric input
- `HMACKeyLengthValidator` — validates HMAC key lengths
- `SessionPropertyValidator` — validates session property names
- Custom validators (implement `org.forgerock.openam.auth.nodes.validators.Validator`)

---

#### **Property Types and Defaults**

**String properties**:
```java
@Attribute(order = 10)
String apiEndpoint();  // Required field (no default)

@Attribute(order = 20)
default String apiKey() {
    return "";  // Optional with default
}
```

**Numeric properties**:
```java
@Attribute(order = 30)
default int connectionTimeout() {
    return 5000;
}

@Attribute(order = 40)
default long maxRetries() {
    return 3L;
}
```

**Boolean properties**:
```java
@Attribute(order = 50)
default boolean enableCaching() {
    return false;
}

@Attribute(order = 60)
default boolean strictValidation() {
    return true;
}
```

**Enum properties** (dropdown in UI):
```java
public enum AuthMethod {
    BASIC, OAUTH2, API_KEY
}

@Attribute(order = 70)
default AuthMethod authMethod() {
    return AuthMethod.API_KEY;
}
```

**List properties** (multi-value text fields):
```java
@Attribute(order = 80)
default List<String> allowedDomains() {
    return Collections.singletonList("example.com");
}
```

**Map properties** (key-value pairs):
```java
@Attribute(order = 90, validators = SessionPropertyValidator.class)
Map<String, String> customHeaders();
```

---

#### **Using Validators**

**Built-in validators**:
```java
@Attribute(order = 10, validators = DecimalValidator.class)
default int riskThreshold() {
    return 50;
}

@Attribute(order = 20, validators = SessionPropertyValidator.class)
Map<String, String> sessionProperties();
```

**Custom validator**:
```java
public class UrlValidator implements Validator {
    @Override
    public void validate(Object value) throws NodeProcessException {
        String url = (String) value;
        if (!url.startsWith("https://")) {
            throw new NodeProcessException("API endpoint must use HTTPS");
        }
    }
}

// In Config:
@Attribute(order = 10, validators = UrlValidator.class)
String apiEndpoint();
```

---

#### **Real-World Example: External API Node**

```java
public class ExternalAPINode extends AbstractDecisionNode {

    private final Config config;
    private final Logger logger = LoggerFactory.getLogger(ExternalAPINode.class);

    @Inject
    public ExternalAPINode(@Assisted Config config) {
        this.config = config;
    }

    @Override
    public boolean process(TreeContext context) throws NodeProcessException {
        String username = context.sharedState.get("username").asString();

        try {
            HttpClient client = HttpClient.newBuilder()
                .connectTimeout(Duration.ofMillis(config.connectionTimeout()))
                .build();

            HttpRequest request = HttpRequest.newBuilder()
                .uri(URI.create(config.apiEndpoint() + "/validate"))
                .header("Authorization", config.authMethod().name() + " " + config.apiKey())
                .header("Content-Type", "application/json")
                .POST(HttpRequest.BodyPublishers.ofString(
                    String.format("{\"username\":\"%s\"}", username)
                ))
                .build();

            HttpResponse<String> response = client.send(request,
                HttpResponse.BodyHandlers.ofString());

            if (config.enableLogging()) {
                logger.info("API response: {}", response.body());
            }

            return response.statusCode() == 200;

        } catch (Exception e) {
            if (config.failOpen()) {
                logger.warn("API call failed, failing open", e);
                return true;
            } else {
                throw new NodeProcessException("API call failed", e);
            }
        }
    }

    public interface Config {

        /**
         * The API endpoint URL (must use HTTPS)
         */
        @Attribute(order = 10, validators = UrlValidator.class)
        String apiEndpoint();

        /**
         * Authentication method for API calls
         */
        @Attribute(order = 20)
        default AuthMethod authMethod() {
            return AuthMethod.API_KEY;
        }

        /**
         * API key or bearer token
         */
        @Attribute(order = 30)
        String apiKey();

        /**
         * Connection timeout in milliseconds
         */
        @Attribute(order = 40, validators = DecimalValidator.class)
        default int connectionTimeout() {
            return 5000;
        }

        /**
         * Whether to log API responses (WARNING: may contain PII)
         */
        @Attribute(order = 50)
        default boolean enableLogging() {
            return false;
        }

        /**
         * If API fails, allow authentication to proceed
         */
        @Attribute(order = 60)
        default boolean failOpen() {
            return false;
        }

        /**
         * Custom HTTP headers to include in API requests
         */
        @Attribute(order = 70)
        default Map<String, String> customHeaders() {
            return Collections.emptyMap();
        }
    }

    public enum AuthMethod {
        BASIC, OAUTH2, API_KEY
    }
}
```

**How this appears in AM Console**:

```
┌────────────────────────────────────┐
│ External API Node Configuration   │
├────────────────────────────────────┤
│ API Endpoint:                      │
│ [https://api.example.com/auth__  ] │
│                                    │
│ Authentication Method:             │
│ [API_KEY ▼]                        │
│                                    │
│ API Key:                           │
│ [sk-1234567890abcdef__________   ] │
│                                    │
│ Connection Timeout:                │
│ [5000_____]                        │
│                                    │
│ ☑ Enable Logging                  │
│ ☑ Fail Open                        │
│                                    │
│ Custom Headers:                    │
│ [X-Request-ID] = [req-{{uuid}}]    │
│ [+ Add]                            │
└────────────────────────────────────┘
```

---

#### **Best Practices**

| Practice | Rationale |
|---|---|---|
| Use JavaDoc on Config interface methods | Appears as tooltips in AM Console |
| Provide sensible defaults | Reduces admin configuration burden |
| Use validators for critical fields | Prevent invalid config at save time |
| Group related settings with order | E.g., 10-40 for API settings, 50-70 for behavior |
| Use enums for limited choices | Better UX than free-text fields |
| Mark sensitive fields | Use naming like `apiKey`, `password` — AM may mask these in logs |

**Interview answer**: "I define a nested `Config` interface inside my node class with methods annotated with `@Attribute`. Each method represents a configurable property — the `order` parameter controls the display order in the AM Console. I use `default` methods for optional properties with defaults, and I can add validators like `DecimalValidator` or custom validators to enforce constraints. The admin configures these values in the tree designer, and I access them via the injected config object. For sensitive values like API keys, I use clear naming and rely on AM's logging to mask them."

**Sources**:
- [Config interface (AM 7.2.2)](https://backstage.forgerock.com/docs/am/7.2/auth-nodes/core-config.html)
- [Config interface (AM 7.0.2)](https://backstage.forgerock.com/docs/am/7/auth-nodes/core-config.html)

---

### Q9: How do you package and deploy a custom node (.jar → WEB-INF/lib → restart)?

**Answer**:

Deploying a custom authentication node involves building the JAR, copying it to AM's webapp, and restarting the application server.

#### **Step 1: Build the JAR**

```bash
# Navigate to your node project
cd ~/projects/risk-scoring-node

# Build with Maven (runs tests, packages JAR)
mvn clean package

# Output: target/risk-scoring-node-1.0.0.jar
```

**What `mvn package` does**:
1. Compiles Java classes (`src/main/java`)
2. Copies resources (`src/main/resources`)
3. Runs unit tests (`src/test/java`)
4. Packages into JAR with dependencies (if using Maven Shade plugin)
5. Places JAR in `target/` directory

---

#### **Step 2: Copy JAR to AM's WEB-INF/lib**

**Standalone Tomcat**:
```bash
# Copy to AM webapp
cp target/risk-scoring-node-1.0.0.jar \
   /opt/tomcat/webapps/am/WEB-INF/lib/
```

**Docker container**:
```bash
# Copy into running container
docker cp target/risk-scoring-node-1.0.0.jar \
   pingam:/opt/tomcat/webapps/am/WEB-INF/lib/

# Or mount a volume for development
docker run -d \
  -v ./custom-nodes:/opt/tomcat/webapps/am/WEB-INF/lib/custom-nodes \
  pingam:latest
```

**Kubernetes (ForgeOps)**:
Build the JAR into a custom AM Docker image:
```dockerfile
FROM gcr.io/forgerock-io/am:7.4.0

# Copy custom node JAR
COPY target/risk-scoring-node-1.0.0.jar \
     /usr/local/tomcat/webapps/am/WEB-INF/lib/

# Rebuild and push image
# docker build -t myregistry/am-custom:7.4.0 .
# docker push myregistry/am-custom:7.4.0
```

---

#### **Step 3: Restart AM**

**Standalone Tomcat**:
```bash
# Stop Tomcat
/opt/tomcat/bin/shutdown.sh

# Wait for process to stop
sleep 5

# Start Tomcat
/opt/tomcat/bin/startup.sh

# Monitor logs
tail -f /opt/tomcat/logs/catalina.out
```

**Docker container**:
```bash
# Restart container
docker restart pingam

# Or stop and start
docker stop pingam
docker start pingam

# Watch logs
docker logs -f pingam
```

**Kubernetes**:
```bash
# Delete pod (Deployment recreates it with new image)
kubectl delete pod am-0

# Or rolling update
kubectl rollout restart statefulset/am
```

---

#### **Step 4: Verify Node Appears in AM Console**

1. Log in to AM Console: `http://pingam:8081/am/console`
2. Navigate to: **Realms** → **[Your Realm]** → **Authentication** → **Trees**
3. Create or edit a tree
4. Check the **Components** pane on the left
5. Your node should appear under **Authentication** or the tag you specified

**If node doesn't appear**:
- Check `catalina.out` for errors (JAR classpath issues, missing dependencies)
- Verify `META-INF/services/org.forgerock.openam.plugins.AmPlugin` exists in the JAR
- Confirm the plugin class is in the JAR: `jar tf risk-scoring-node-1.0.0.jar | grep Plugin`
- Check for exceptions during AM startup

---

#### **Development Workflow Automation**

**deploy.sh** (for rapid iteration):
```bash
#!/bin/bash
set -e

echo "Building node..."
mvn clean package -DskipTests

echo "Copying JAR to AM..."
docker cp target/risk-scoring-node-1.0.0.jar \
   pingam:/opt/tomcat/webapps/am/WEB-INF/lib/

echo "Restarting AM..."
docker restart pingam

echo "Waiting for AM to start..."
until $(curl --output /dev/null --silent --head --fail http://localhost:8081/am/console); do
    printf '.'
    sleep 2
done

echo ""
echo "✓ Node deployed successfully!"
echo "Access AM Console at: http://localhost:8081/am/console"
```

Usage:
```bash
chmod +x deploy.sh
./deploy.sh
```

---

#### **Production Deployment Best Practices**

| Environment | Deployment Method | Rollback Strategy |
|---|---|---|
| **Dev** | Manual copy + restart | Delete JAR, restart |
| **Staging** | CI/CD pipeline builds custom AM image | Redeploy previous image tag |
| **Production** | Blue/green deployment with custom AM image | Switch traffic to blue cluster |

**CI/CD Pipeline (GitHub Actions example)**:
```yaml
name: Build and Deploy Custom Node

on:
  push:
    branches: [main]

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Set up JDK 11
        uses: actions/setup-java@v3
        with:
          java-version: '11'

      - name: Build with Maven
        run: mvn clean package

      - name: Build AM Docker image
        run: |
          docker build -t myregistry/am:${{ github.sha }} .
          docker tag myregistry/am:${{ github.sha }} myregistry/am:latest

      - name: Push to registry
        run: |
          docker push myregistry/am:${{ github.sha }}
          docker push myregistry/am:latest

      - name: Deploy to Kubernetes
        run: |
          kubectl set image statefulset/am \
            am=myregistry/am:${{ github.sha }}
```

---

#### **Common Deployment Issues**

| Issue | Cause | Fix |
|---|---|---|
| Node doesn't appear in Console | Plugin not discovered | Check `META-INF/services/...AmPlugin` file |
| ClassNotFoundException | Dependency not in JAR | Use Maven Shade to bundle dependencies |
| Method signature error | AM API version mismatch | Recompile node against deployed AM version |
| Node appears but errors on add | Invalid @Node.Metadata | Check outcomeProvider and configClass |
| JAR locked on Windows | File handle not released | Use `robocopy /MOV` or stop Tomcat first |

---

#### **Versioning Strategy**

Track custom node versions separately from AM:
```
risk-scoring-node-1.0.0.jar    → Initial release (AM 7.3)
risk-scoring-node-1.1.0.jar    → Added caching (AM 7.3)
risk-scoring-node-2.0.0.jar    → Recompiled for AM 7.4 API changes
```

Include version in JAR manifest:
```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-jar-plugin</artifactId>
    <configuration>
        <archive>
            <manifest>
                <addDefaultImplementationEntries>true</addDefaultImplementationEntries>
            </manifest>
            <manifestEntries>
                <Implementation-Version>${project.version}</Implementation-Version>
                <Build-Time>${maven.build.timestamp}</Build-Time>
            </manifestEntries>
        </archive>
    </configuration>
</plugin>
```

**Interview answer**: "I build the node JAR with `mvn clean package`, copy it to `WEB-INF/lib/` in the AM webapp, and restart Tomcat. For Docker, I use `docker cp` and `docker restart`. In production, we bake custom nodes into a custom AM Docker image via CI/CD — when we push to main, GitHub Actions builds the JAR, creates a new AM image with the JAR inside, and pushes it to our registry. Kubernetes then performs a rolling update. For rapid dev iteration, I have a `deploy.sh` script that does the full cycle in one command. The key point is that AM has no hot-reload — you must restart to discover new nodes."

---

### Q10: How do you test custom authentication nodes (unit tests, integration tests)?

**Answer**:

Testing custom authentication nodes requires both **unit tests** (test node logic in isolation) and **integration tests** (test node in a live AM instance).

#### **Unit Testing**

Authentication nodes are well-suited for unit tests because:
- Low number of static dependencies
- `TreeContext` and `Action` are designed to be testable without mocking
- Business logic can be extracted into separate classes

**Example unit test** (using JUnit 5 + Mockito):

```java
package com.example.nodes;

import org.forgerock.json.JsonValue;
import org.forgerock.openam.auth.node.api.Action;
import org.forgerock.openam.auth.node.api.NodeProcessException;
import org.forgerock.openam.auth.node.api.TreeContext;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.BeforeEach;
import org.mockito.Mock;
import org.mockito.MockitoAnnotations;

import javax.security.auth.callback.Callback;
import javax.security.auth.callback.NameCallback;
import java.util.Collections;
import java.util.Optional;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.Mockito.*;

class RiskScoringNodeTest {

    @Mock
    private RiskScoringNode.Config config;

    @Mock
    private RiskScoringService riskService;

    private RiskScoringNode node;

    @BeforeEach
    void setUp() {
        MockitoAnnotations.openMocks(this);
        node = new RiskScoringNode(config, riskService);

        // Set default config values
        when(config.highRiskThreshold()).thenReturn(75);
        when(config.apiEndpoint()).thenReturn("https://api.example.com/risk");
    }

    @Test
    void testLowRiskScore_ReturnsFalse() throws Exception {
        // Arrange
        JsonValue sharedState = JsonValue.json(
            JsonValue.object(
                JsonValue.field("username", "demo")
            )
        );

        TreeContext context = getTreeContext(sharedState);

        when(riskService.calculateRisk("demo")).thenReturn(25);

        // Act
        Action result = node.process(context);

        // Assert
        assertEquals("false", result.outcome);
        verify(riskService).calculateRisk("demo");
    }

    @Test
    void testHighRiskScore_ReturnsTrue() throws Exception {
        // Arrange
        JsonValue sharedState = JsonValue.json(
            JsonValue.object(
                JsonValue.field("username", "demo")
            )
        );

        TreeContext context = getTreeContext(sharedState);

        when(riskService.calculateRisk("demo")).thenReturn(85);

        // Act
        Action result = node.process(context);

        // Assert
        assertEquals("true", result.outcome);
        assertEquals(85, result.sharedState.get("riskScore").asInteger().intValue());
    }

    @Test
    void testMissingUsername_ThrowsException() {
        // Arrange
        JsonValue emptySharedState = JsonValue.json(JsonValue.object());
        TreeContext context = getTreeContext(emptySharedState);

        // Act & Assert
        assertThrows(NodeProcessException.class, () -> {
            node.process(context);
        });
    }

    @Test
    void testAPIFailure_HandlesGracefully() throws Exception {
        // Arrange
        JsonValue sharedState = JsonValue.json(
            JsonValue.object(
                JsonValue.field("username", "demo")
            )
        );

        TreeContext context = getTreeContext(sharedState);

        when(riskService.calculateRisk("demo"))
            .thenThrow(new RuntimeException("API timeout"));

        // Act & Assert
        assertThrows(NodeProcessException.class, () -> {
            node.process(context);
        });
    }

    // Helper to create TreeContext
    private TreeContext getTreeContext(JsonValue sharedState) {
        return new TreeContext(
            sharedState,
            JsonValue.json(JsonValue.object()),  // transientState
            new ExternalRequestContext.Builder().build(),
            Collections.emptyList(),  // callbacks
            Optional.empty()  // request
        );
    }
}
```

**Key unit testing practices**:
- Test all code paths (true, false, error outcomes)
- Test with missing/malformed shared state
- Test error handling (API failures, timeouts)
- Verify shared state is updated correctly
- Test callback handling if applicable
- Mock external dependencies (LDAP, HTTP clients, etc.)

---

#### **Integration Testing**

Integration tests deploy the node to a live AM instance and test via the authentication REST API.

**Approach 1: Manual testing via curl**

```bash
# 1. Start authentication journey
curl -X POST "http://pingam:8081/am/json/realms/root/realms/techcorp/authenticate?authIndexType=service&authIndexValue=RiskScoringTree" \
  -H "Accept-API-Version: resource=2.0, protocol=1.0" \
  | jq .

# 2. Submit username callback
curl -X POST "http://pingam:8081/am/json/realms/root/realms/techcorp/authenticate" \
  -H "Accept-API-Version: resource=2.0, protocol=1.0" \
  -H "Content-Type: application/json" \
  -d '{
    "authId": "<AUTH_ID_FROM_STEP_1>",
    "callbacks": [
      {
        "type": "NameCallback",
        "output": [{"name": "prompt", "value": "User Name"}],
        "input": [{"name": "IDToken1", "value": "demo"}]
      }
    ]
  }' | jq .

# 3. Check if risk node routed correctly
# Look for "stage": "RiskHigh" or "stage": "RiskLow" in response
```

---

**Approach 2: Automated integration tests with RestAssured**

```java
import io.restassured.RestAssured;
import io.restassured.response.Response;
import org.json.JSONObject;
import org.junit.jupiter.api.Test;

import static io.restassured.RestAssured.*;
import static org.hamcrest.Matchers.*;

public class RiskScoringNodeIntegrationTest {

    private static final String AM_URL = "http://localhost:8081/am";
    private static final String REALM = "techcorp";
    private static final String TREE = "RiskScoringTree";

    @Test
    public void testLowRiskUser_SkipsMFA() {
        // Start authentication
        Response response = given()
            .header("Accept-API-Version", "resource=2.0, protocol=1.0")
            .queryParam("authIndexType", "service")
            .queryParam("authIndexValue", TREE)
            .when()
            .post(AM_URL + "/json/realms/root/realms/" + REALM + "/authenticate")
            .then()
            .statusCode(200)
            .extract().response();

        String authId = response.jsonPath().getString("authId");

        // Submit username (low-risk user)
        response = given()
            .header("Accept-API-Version", "resource=2.0, protocol=1.0")
            .header("Content-Type", "application/json")
            .body("{ \"authId\": \"" + authId + "\", " +
                  "\"callbacks\": [" +
                  "  {\"type\": \"NameCallback\", " +
                  "   \"output\": [{\"name\": \"prompt\", \"value\": \"User Name\"}], " +
                  "   \"input\": [{\"name\": \"IDToken1\", \"value\": \"lowrisk-user\"}]}" +
                  "]}")
            .when()
            .post(AM_URL + "/json/realms/root/realms/" + REALM + "/authenticate")
            .then()
            .statusCode(200)
            .body("stage", not(equalTo("MFANode")))  // Should NOT be at MFA stage
            .extract().response();
    }

    @Test
    public void testHighRiskUser_RequiresMFA() {
        // Similar test, but verify high-risk user routes to MFA node
        // ...
    }
}
```

---

**Approach 3: Selenium/Playwright for full UI testing**

```javascript
// Playwright example
const { test, expect } = require('@playwright/test');

test('Low risk user skips MFA', async ({ page }) => {
  await page.goto('http://localhost:8081/am/XUI/#login/&realm=/techcorp&service=RiskScoringTree');

  // Enter username
  await page.fill('input[name="username"]', 'lowrisk-user');
  await page.fill('input[name="password"]', 'password');
  await page.click('button[type="submit"]');

  // Should NOT see MFA prompt
  await expect(page.locator('text=Enter OTP')).not.toBeVisible();

  // Should be logged in
  await expect(page.locator('text=Dashboard')).toBeVisible();
});

test('High risk user requires MFA', async ({ page }) => {
  await page.goto('http://localhost:8081/am/XUI/#login/&realm=/techcorp&service=RiskScoringTree');

  // Enter username
  await page.fill('input[name="username"]', 'highrisk-user');
  await page.fill('input[name="password"]', 'password');
  await page.click('button[type="submit"]');

  // Should see MFA prompt
  await expect(page.locator('text=Enter OTP')).toBeVisible();
});
```

---

#### **Test Environment Setup**

**Dockerized test environment**:
```yaml
# docker-compose.test.yml
version: '3.8'

services:
  pingds:
    image: pingidentity/pingdirectory:latest
    environment:
      - BASE_DN=dc=example,dc=com

  pingam:
    image: myorg/am-with-custom-node:latest
    ports:
      - "8081:8080"
    depends_on:
      - pingds
    volumes:
      - ./test-config:/opt/am/config
```

Run tests:
```bash
# Start test environment
docker-compose -f docker-compose.test.yml up -d

# Wait for AM to be ready
./wait-for-am.sh

# Run integration tests
mvn verify -P integration-tests

# Tear down
docker-compose -f docker-compose.test.yml down
```

---

#### **Test Coverage Strategy**

| Test Type | What to Test | Tool |
|---|---|---|
| **Unit** | Node logic, error handling, state management | JUnit, Mockito |
| **Integration** | Node in live tree, REST API flows | RestAssured, curl |
| **UI** | Full login flows, visual verification | Selenium, Playwright |
| **Performance** | Node latency, external API timeout handling | JMeter, Gatling |
| **Security** | Input validation, XSS prevention | OWASP ZAP, Burp Suite |

**Coverage goals**:
- **Unit tests**: 80%+ code coverage
- **Integration tests**: Cover all normal flows + 1-2 error scenarios
- **UI tests**: Critical user journeys only (expensive to maintain)

---

#### **Best Practices**

| Practice | Rationale |
|---|---|---|
| Extract business logic to separate classes | Easier to unit test without AM dependencies |
| Use Guice for dependency injection | Mock external services in tests |
| Test all outcomes | Each outcome = one test case |
| Test with malformed input | Missing username, null callbacks, etc. |
| Use test fixtures for TreeContext | DRY principle — reusable test data |
| Automate integration tests in CI/CD | Catch regressions before deployment |

**Interview answer**: "I write unit tests for the node logic using JUnit and Mockito, testing all code paths — true outcome, false outcome, errors. The `TreeContext` is designed to be easily constructed in tests without mocking. For integration testing, I deploy the node to a test AM instance and use RestAssured to call the authentication REST API, verifying that the tree routes correctly based on different inputs. For critical flows, we also have Playwright UI tests. In CI/CD, we spin up a Dockerized AM environment, deploy the custom node, run the integration test suite, and tear it down. Unit tests give us 80%+ code coverage, while integration tests verify the node works correctly in a real tree."

**Sources**:
- [Maintaining Authentication Nodes (AM 7.0.2)](https://backstage.pingidentity.com/docs/am/7/auth-nodes/maintaining-nodes.html)
- [Maintain authentication nodes (AM 7.2.2)](https://backstage.forgerock.com/docs/am/7.2/auth-nodes/maintaining-nodes.html)

---

(Continued in next response due to length...)
