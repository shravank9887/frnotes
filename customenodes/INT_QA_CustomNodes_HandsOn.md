# ForgeRock/PingAM Interview Questions — Custom Authentication Nodes (Hands-On Lab)

*Built from hands-on experience developing and deploying 3 custom nodes to PingAM 8.0.2*

---

## The Big Picture

### Q1: What is a custom authentication node and why would you build one?

**Answer**: A custom authentication node is a Java class that plugs into AM's authentication tree (journey) framework. Each node performs one unit of work during authentication — check a condition, call an external API, collect user input, enrich state, or route the flow.

You build custom nodes when AM's built-in nodes don't cover your requirement:
- **Business logic**: Check business hours, corporate calendar, IP reputation
- **External integration**: Call a risk engine, SMS gateway, or internal API
- **Custom routing**: Multi-outcome decisions beyond simple true/false
- **State enrichment**: Read headers, device fingerprint, or geolocation into shared state

**What you DON'T do**: You don't build nodes for things AM already handles (LDAP auth, OATH TOTP, session management). Use built-in nodes first.

**Interview answer**: "We built custom nodes when built-in ones didn't cover our business requirements. For example, a BusinessHoursNode that restricted authentication to office hours with configurable timezone and thresholds. The key advantage is that admins can configure these nodes in the tree designer without code changes — security team tuned the hours and thresholds themselves."

---

## Core Classes and Interfaces

### Q2: What are the key classes/interfaces you need to know to build a custom node?

**Answer**: There are 6 essential pieces:

| Class/Interface | Package | Purpose |
|----------------|---------|---------|
| `Node` | `org.forgerock.openam.auth.node.api` | Base interface — every node implements this |
| `AbstractDecisionNode` | same | Convenience base class — provides `true`/`false` outcomes |
| `SingleOutcomeNode` | same | Convenience base class — single exit path |
| `TreeContext` | same | Input to `process()` — holds shared state, request, callbacks |
| `Action` | same | Return type of `process()` — controls routing and state |
| `OutcomeProvider` | same | Defines the node's exit connections in the tree designer |

Plus supporting pieces:
- `@Node.Metadata` — annotation that registers the node with AM
- `@Attribute` (`org.forgerock.openam.annotations.sm.Attribute`) — makes Config properties configurable in UI
- `@Inject` / `@Assisted` — Guice dependency injection for Config
- `AbstractNodeAmPlugin` — registers nodes with AM's plugin framework

**Interview answer**: "The core API is the `Node` interface with its `process(TreeContext)` method. TreeContext gives you shared state, HTTP request context, and callbacks. You return an `Action` that tells the tree which outcome to follow and what state changes to make. For simple true/false decisions I extend `AbstractDecisionNode`; for multi-outcome routing I implement `Node` directly with a custom `OutcomeProvider`."

---

### Q3: Explain the three base class options — when do you use each?

**Answer**:

```
Node (interface)
├── AbstractDecisionNode    → 2 outcomes: true / false
├── SingleOutcomeNode       → 1 outcome: always continues
└── (implement directly)    → N outcomes: custom routing
```

**`AbstractDecisionNode`** — Use when your node makes a yes/no decision:
- Business hours check → true (within hours) / false (outside)
- Data Store Decision (built-in) → true (auth success) / false (auth failure)
- Provides `goTo(boolean)` helper method

**`SingleOutcomeNode`** — Use when your node always continues (no branching):
- Set a session property
- Write to shared state
- Call an external API for side effects (logging, audit)
- Provides `goToNext()` helper method

**Implement `Node` directly** — Use when you need more than 2 outcomes:
- Risk router → low / medium / high / error
- Header check → found / missing
- Country router → US / EU / APAC / blocked
- Must provide a custom `OutcomeProvider` and use `Action.goTo("outcomeId")`

**Interview answer**: "I choose the base class based on how many exit paths I need. `AbstractDecisionNode` for boolean decisions — it gives you `goTo(true)` and `goTo(false)`. `SingleOutcomeNode` for enrichment nodes that always pass through. For anything more complex, I implement `Node` directly with a custom `OutcomeProvider` — like our risk router with 4 outcomes."

---

## The Node Class — Anatomy

### Q4: Walk through the anatomy of a custom node class

**Answer**: A custom node has 5 parts:

```java
// 1. @Node.Metadata — registers with AM
@Node.Metadata(
    outcomeProvider = AbstractDecisionNode.OutcomeProvider.class,  // exit paths
    configClass = MyNode.Config.class,                             // admin config
    tags = {"risk"},                                               // UI category
    i18nFile = "com/example/nodes/MyNode"                         // .properties file for labels
)
public class MyNode extends AbstractDecisionNode {

    // 2. Config interface — admin-configurable properties
    public interface Config {
        @Attribute(order = 100)
        default int threshold() { return 50; }
    }

    // 3. Constructor — Guice injects Config
    @Inject
    public MyNode(@Assisted Config config) {
        this.config = config;
    }

    // 4. process() — the logic
    @Override
    public Action process(TreeContext context) throws NodeProcessException {
        // read state, make decision, return Action
        return goTo(true).build();
    }

    // 5. (Optional) Custom OutcomeProvider — for multi-outcome nodes
}
```

**The 5 parts explained**:

1. **`@Node.Metadata`** — Tells AM: "this is a node, here's how to configure it, and here are its outcomes"
2. **`Config` interface** — Each method becomes a property in the tree designer. `@Attribute(order=N)` controls display order. Methods with `default` are optional; without `default` they're required.
3. **Constructor** — `@Inject` + `@Assisted Config` means Guice creates a new Config per node instance. Each node in a tree can have different config values.
4. **`process(TreeContext)`** — Called every time a user hits this node. Read state, make decisions, return an `Action`.
5. **`OutcomeProvider`** — Only needed if you implement `Node` directly. Defines the exit connections shown in the tree designer.

---

### Q5: What is `TreeContext` and what can you access from it?

**Answer**: `TreeContext` is the input to `process()`. It carries everything the node needs:

```java
context.sharedState      // JsonValue — non-sensitive data shared between nodes
                         //   e.g., username, realm, header values, riskScore
                         //   Survives across nodes in the tree

context.transientState   // JsonValue — sensitive data (passwords, OTPs)
                         //   Encrypted at rest, cleared after tree completes

context.request          // ExternalRequestContext — the HTTP request
  .headers               //   ListMultimap<String, String> — all HTTP headers
  .clientIp              //   String — client IP address
  .cookies               //   Map<String, String> — request cookies
  .parameters            //   Map<String, List<String>> — query parameters
  .hostName              //   String — server hostname
  .serverUrl             //   String — AM's base URL

context.getCallback(TextOutputCallback.class)   // Get user-submitted callbacks
context.hasCallbacks()                           // Check if callbacks were returned
context.universalId      // Optional<String> — user's universal ID (if identified)
```

**Key rule**: shared state is readable by ALL nodes in the tree. Don't put secrets there — use transient state for passwords and tokens.

**Interview answer**: "TreeContext gives me everything: shared state for passing data between nodes, transient state for secrets like passwords, and the ExternalRequestContext for HTTP headers, client IP, and cookies. In our header enrichment node, I read `context.request.headers` to capture X-Forwarded-For and User-Agent, then wrote them into shared state so the downstream risk scoring node could use them."

---

### Q6: What is `Action` and how do you control tree flow?

**Answer**: `Action` is the return type of `process()`. It tells the tree engine:
1. **Where to go next** (which outcome)
2. **What state changes to make** (shared state, session properties)

```java
// Boolean routing (AbstractDecisionNode)
return goTo(true).build();       // → "true" outcome
return goTo(false).build();      // → "false" outcome

// Single outcome (SingleOutcomeNode)
return goToNext().build();       // → only outcome

// Named outcome (custom OutcomeProvider)
return Action.goTo("low").build();     // → "low" outcome
return Action.goTo("error").build();   // → "error" outcome

// Modify shared state
JsonValue newState = context.sharedState.copy();
newState.put("riskScore", 75);
return Action.goTo("high").replaceSharedState(newState).build();

// Set session properties (available to policies and apps after auth)
return Action.goTo("low")
    .putSessionProperty("riskLevel", "low")
    .putSessionProperty("riskScore", "25")
    .build();

// Send callbacks (request user input)
return Action.send(new TextOutputCallback(TextOutputCallback.INFORMATION, "Hello"))
    .build();
```

**Key methods on ActionBuilder**:
- `replaceSharedState(JsonValue)` — overwrites shared state
- `replaceTransientState(JsonValue)` — overwrites transient state
- `putSessionProperty(key, value)` — writes into the AM session
- `withErrorMessage(msg)` — sets error message
- `build()` — creates the final Action

**Interview answer**: "`Action` controls the flow. `goTo(true/false)` for decision nodes, `Action.goTo('outcomeId')` for multi-outcome. The builder pattern lets me chain state modifications — I can replace shared state, set session properties, and route in a single fluent call. Session properties are especially useful because downstream policies can read them for authorization decisions."

---

## Config Interface

### Q7: How does the Config interface work and what types are supported?

**Answer**: The `Config` interface defines admin-configurable properties. Each method becomes a field in the tree designer UI.

```java
public interface Config {

    // Required field (no default) — admin MUST set this
    @Attribute(order = 100)
    String apiEndpoint();

    // Optional field with default — admin can override
    @Attribute(order = 200)
    default int timeout() { return 30; }

    // Boolean toggle
    @Attribute(order = 300)
    default boolean enableLogging() { return true; }

    // Enum dropdown — AM renders as a select box
    @Attribute(order = 400)
    default Timezone timezone() { return Timezone.UTC; }

    // Set<String> — multi-value input (admin adds multiple values)
    @Attribute(order = 500)
    Set<String> allowedHeaders();

    // Map<String, String> — key-value pairs
    @Attribute(order = 600)
    Map<String, String> headerMappings();
}
```

**Supported types**: `String`, `int`, `boolean`, `enum`, `Set<String>`, `Map<String, String>`, `char[]` (passwords with `@Password`).

**`@Attribute(order = N)`** — Required. Controls the display order in the UI. Use increments of 100 for easy insertion later.

**How to access config**: Via the injected reference:
```java
int hours = config.startHour();        // reads the configured value
Timezone tz = config.timezone();       // reads the enum selection
Set<String> headers = config.headersToCapture();  // reads the multi-value set
```

**Interview answer**: "The Config interface is elegant — each method becomes a UI field. `@Attribute(order)` controls layout. Enums render as dropdowns, Sets as multi-value inputs, booleans as toggles. The admin configures everything in the tree designer — no code changes needed. Each node instance in a tree gets its own Config, so the same node type can have different settings in different positions."

---

## OutcomeProvider

### Q8: When and how do you create a custom OutcomeProvider?

**Answer**: You need a custom OutcomeProvider when your node has more than 2 outcomes (or different outcomes than true/false).

```java
// Define as a static inner class
public static class RiskOutcomeProvider implements OutcomeProvider {
    @Override
    public List<Outcome> getOutcomes(PreferredLocales locales, JsonValue nodeAttributes) {
        return List.of(
            new Outcome("low", "Low Risk"),       // id used in Action.goTo(), displayName in UI
            new Outcome("medium", "Medium Risk"),
            new Outcome("high", "High Risk"),
            new Outcome("error", "Error")
        );
    }
}
```

**Wire it in `@Node.Metadata`**:
```java
@Node.Metadata(
    outcomeProvider = RiskOutcomeProvider.class,  // <-- your custom provider
    configClass = MyNode.Config.class
)
```

**Use in `process()`**:
```java
return Action.goTo("low").build();    // must match an Outcome id
return Action.goTo("error").build();  // must match an Outcome id
```

**Built-in providers you get for free**:
- `AbstractDecisionNode.OutcomeProvider` → `"true"`, `"false"`
- `SingleOutcomeNode.OutcomeProvider` → single outcome (auto-wired)

**Advanced**: `getOutcomes()` receives `nodeAttributes` (the node's config). You could dynamically generate outcomes based on config — e.g., a country router that reads a list of countries from config and creates one outcome per country.

**Interview answer**: "OutcomeProvider defines the exit connections in the tree designer. For our risk router, I created a provider with 4 outcomes — low, medium, high, error. Each `Outcome` has an ID that matches what `Action.goTo()` uses and a display name for the UI. For simple cases, `AbstractDecisionNode` gives you true/false automatically."

---

## Plugin Class and Registration

### Q9: How does AM discover and load custom nodes?

**Answer**: Three pieces work together:

**1. Plugin class** — extends `AbstractNodeAmPlugin`:
```java
public class TechCorpNodesPlugin extends AbstractNodeAmPlugin {

    private static final String CURRENT_VERSION = "1.0.0";

    @Override
    protected Map<String, Iterable<? extends Class<? extends Node>>> getNodesByVersion() {
        return Collections.singletonMap(
            CURRENT_VERSION,
            List.of(BusinessHoursNode.class, HeaderCheckNode.class)
        );
    }

    @Override
    public String getPluginVersion() {
        return CURRENT_VERSION;
    }
}
```

**2. Service registration file** — `META-INF/services/org.forgerock.openam.plugins.AmPlugin`:
```
com.techcorp.nodes.TechCorpNodesPlugin
```

**3. JAR deployed to** `WEB-INF/lib/` + AM restart.

**How AM loads it**:
1. On startup, AM scans `META-INF/services/org.forgerock.openam.plugins.AmPlugin` in all JARs
2. Finds `TechCorpNodesPlugin`, instantiates it
3. Calls `getNodesByVersion()` to discover node classes
4. Registers each class as an available node type
5. Nodes appear in the tree designer Components panel

**Version management**: `getNodesByVersion()` maps versions to node classes. When you add nodes in v1.1.0, add a new map entry. AM calls `upgrade(fromVersion)` for existing installs so you can migrate config.

**Interview answer**: "AM uses Java's ServiceLoader pattern. The plugin class extends `AbstractNodeAmPlugin` and is listed in `META-INF/services`. On startup, AM discovers it, calls `getNodesByVersion()` to register all node classes, and they appear in the tree designer. The version mapping supports upgrade paths — when we added new nodes in v1.1, existing installs got the `upgrade()` callback."

---

## Properties File (i18n)

### Q10: What is the properties file for and what happens without it?

**Answer**: The `.properties` file provides UI labels, descriptions, and help text for the tree designer.

**File location**: Must match the `i18nFile` path in `@Node.Metadata`:
```java
@Node.Metadata(i18nFile = "com/techcorp/nodes/BusinessHoursNode")
// → src/main/resources/com/techcorp/nodes/BusinessHoursNode.properties
```

**File contents**:
```properties
nodeDescription=Business Hours Check
nodeHelp=Checks if current time is within configurable business hours.

startHour=Start Hour
startHour.help=The hour (0-23) when business hours begin.

endHour=End Hour
endHour.help=The hour (0-23) when business hours end.

timezone=Timezone
timezone.help=The timezone to evaluate business hours in.
```

**Key convention**: Property name = Config method name. `nodeDescription` and `nodeHelp` are special — they set the node's title and tooltip in the Components panel.

**Without it**: Nodes still work, but the UI shows raw Java method names (`startHour` instead of "Start Hour") and no help text. Functional but unprofessional.

**Localization**: Create `BusinessHoursNode_fr.properties` for French, `BusinessHoursNode_ja.properties` for Japanese, etc. AM picks the right one based on browser locale.

---

## Project Structure and Dependencies

### Q11: What is the Maven project structure and what are the key dependencies?

**Answer**:

```
custom-nodes/
├── pom.xml
├── src/main/java/com/techcorp/nodes/
│   ├── BusinessHoursNode.java        ← Node class
│   ├── TechCorpNodesPlugin.java      ← Plugin registration
│   └── ...
├── src/main/resources/
│   ├── com/techcorp/nodes/
│   │   └── BusinessHoursNode.properties   ← UI labels
│   └── META-INF/services/
│       └── org.forgerock.openam.plugins.AmPlugin   ← Plugin discovery
└── lib/                               ← Extracted JARs (local repo)
```

**Key dependencies** (all `<scope>provided</scope>` — AM supplies them at runtime):

| Dependency | What it provides |
|-----------|-----------------|
| `auth-node-api` | Node, AbstractDecisionNode, SingleOutcomeNode, TreeContext, Action, OutcomeProvider |
| `openam-annotations` | `@Attribute` annotation |
| `openam-plugin-framework` | `AmPlugin` interface, `PluginTools`, `PluginException` |
| `forgerock-util` | `JsonValue` (used in shared state) |
| `openam-i18n` | `PreferredLocales` (used in OutcomeProvider) |
| `jakarta.inject-api` | `@Inject` annotation |
| `guice` + `guice-assistedinject` | `@Assisted` annotation, DI framework |
| `openam-core` | `CoreWrapper`, `AMIdentity` (optional — for identity operations) |

**Why `provided` scope**: These JARs already exist in AM's `WEB-INF/lib/`. Your JAR only contains YOUR code. If you bundled them, you'd get class conflicts.

**Without ForgeRock Maven repo**: Extract JARs from a running AM container (`WEB-INF/lib/`) and install to local Maven repo with `mvn install:install-file`.

---

## Build, Deploy, and Lifecycle

### Q12: What is the build → deploy → test cycle?

**Answer**:

```
1. Write code       →  Java classes + Config + properties
2. Build            →  mvn clean package  →  target/my-nodes-1.0.0.jar
3. Deploy           →  Copy JAR to AM's WEB-INF/lib/
4. Restart AM       →  AM discovers and registers nodes
5. Test in tree     →  Drag nodes into tree designer, configure, test auth flow
6. Iterate          →  Edit code → mvn package → redeploy → restart → test
```

**Important**:
- Delete old versions of your JAR before deploying new ones (avoid classpath conflicts)
- AM restart is required — nodes are loaded at startup
- For development, use `getPluginVersion()` returning `"0.0.0"` — AM treats this as dev mode and reinstalls SMS schema on every restart
- Config changes in the tree designer are stored in DS (am-config) — they survive restarts
- If your node fails to load (bad code), AM still starts — the node just won't appear in Components

---

## Practical Patterns

### Q13: How do you read HTTP headers and enrich shared state?

**Answer**: This is the "header enrichment" pattern — capture request context for downstream nodes.

```java
@Override
public Action process(TreeContext context) throws NodeProcessException {
    // Copy shared state (it's immutable — never modify in place)
    JsonValue newState = context.sharedState.copy();

    // Read client IP (always available)
    newState.put("clientIp", context.request.clientIp);

    // Read specific headers
    List<String> forwardedFor = context.request.headers.get("X-Forwarded-For");
    if (forwardedFor != null && !forwardedFor.isEmpty()) {
        newState.put("header_X-Forwarded-For", forwardedFor.get(0));
    }

    // Return with updated state
    return Action.goTo("found").replaceSharedState(newState).build();
}
```

**Key points**:
- `context.request.headers` is a `ListMultimap` (one header name can have multiple values)
- Always `.copy()` shared state before modifying — it's immutable
- Use `replaceSharedState()` to pass the modified state forward
- Downstream nodes read these values from `context.sharedState.get("clientIp")`

---

### Q14: How do you set session properties and why?

**Answer**: Session properties are written into the AM session token. After authentication completes, they're available to:
- Authorization policies (conditions can check session properties)
- Applications via session validation / token introspection
- Web Agents and PingGateway for per-request decisions

```java
return Action.goTo("low")
    .putSessionProperty("riskLevel", "low")
    .putSessionProperty("riskScore", "25")
    .build();
```

**Use case**: Risk router sets `riskLevel=high` → authorization policy denies access to sensitive resources when `riskLevel=high` → no code change needed to enforce, just policy config.

---

## Common Mistakes

### Q15: What are common mistakes when building custom nodes?

**Answer**:

1. **`javax.inject` vs `jakarta.inject`** — PingAM 8 runs on Tomcat 10 (Jakarta EE). Use `jakarta.inject.Inject`, not `javax.inject.Inject`. Wrong namespace → compilation failure.

2. **Modifying shared state in place** — `context.sharedState` is immutable. Always `.copy()` first, then `replaceSharedState()`.

3. **Missing `META-INF/services` file** — Without this, AM never discovers your plugin. Nodes won't appear. No error.

4. **Forgetting `@Assisted` on Config** — Each node instance gets its own Config. Without `@Assisted`, Guice can't inject per-instance config → startup failure.

5. **Outcome ID mismatch** — `Action.goTo("Low")` won't match `new Outcome("low", "Low Risk")`. IDs are case-sensitive.

6. **Bundling provided dependencies** — Including AM's own JARs in your JAR causes classloader conflicts. Always use `<scope>provided</scope>`.

7. **No AM restart after deploy** — Nodes are loaded at startup only. Copying a new JAR without restart does nothing.

8. **Old JAR still in WEB-INF/lib** — Two versions of the same plugin → unpredictable behavior. Always delete old versions.

---

*Created: 2026-01-31 — Lab 12: Custom Authentication Nodes (Hands-On Development)*
*Based on building and deploying BusinessHoursNode, HeaderCheckNode, and RiskLevelRouterNode to PingAM 8.0.2*
