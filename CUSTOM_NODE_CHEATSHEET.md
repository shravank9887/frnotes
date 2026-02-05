# Custom Authentication Node — Cheat Sheet

Quick-reference templates for all 3 node types + plugin + properties + project setup.

---

## Choose Your Base Class

```
Need boolean decision (yes/no)?     → AbstractDecisionNode  (goTo(true/false))
Need pass-through (no branching)?   → SingleOutcomeNode     (goToNext())
Need 3+ outcomes (routing)?         → implements Node        (Action.goTo("id"))
```

---

## Template 1: AbstractDecisionNode (true/false)

```java
package com.example.nodes;

import jakarta.inject.Inject;
import org.forgerock.openam.annotations.sm.Attribute;
import org.forgerock.openam.auth.node.api.*;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import com.google.inject.assistedinject.Assisted;

@Node.Metadata(
    outcomeProvider = AbstractDecisionNode.OutcomeProvider.class,
    configClass = MyDecisionNode.Config.class,
    tags = {"risk"},
    i18nFile = "com/example/nodes/MyDecisionNode"
)
public class MyDecisionNode extends AbstractDecisionNode {

    private static final Logger logger = LoggerFactory.getLogger(MyDecisionNode.class);

    public interface Config {
        @Attribute(order = 100)
        default int threshold() { return 50; }           // optional — has default

        @Attribute(order = 200)
        String requiredField();                           // required — no default
    }

    private final Config config;

    @Inject
    public MyDecisionNode(@Assisted Config config) {
        this.config = config;
    }

    @Override
    public Action process(TreeContext context) throws NodeProcessException {
        // Your logic here
        boolean result = true;
        return goTo(result).build();
    }
}
```

**Outcomes**: `true` → one path, `false` → another path.
**Key method**: `goTo(boolean)` — inherited from AbstractDecisionNode.

---

## Template 2: SingleOutcomeNode (pass-through)

```java
package com.example.nodes;

import jakarta.inject.Inject;
import org.forgerock.json.JsonValue;
import org.forgerock.openam.annotations.sm.Attribute;
import org.forgerock.openam.auth.node.api.*;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import com.google.inject.assistedinject.Assisted;

@Node.Metadata(
    outcomeProvider = SingleOutcomeNode.OutcomeProvider.class,
    configClass = MyPassthroughNode.Config.class,
    tags = {"utilities"},
    i18nFile = "com/example/nodes/MyPassthroughNode"
)
public class MyPassthroughNode extends SingleOutcomeNode {

    private static final Logger logger = LoggerFactory.getLogger(MyPassthroughNode.class);

    public interface Config {
        @Attribute(order = 100)
        default String prefix() { return "DEFAULT"; }

        @Attribute(order = 200)
        default boolean enableFeature() { return true; }
    }

    private final Config config;

    @Inject
    public MyPassthroughNode(@Assisted Config config) {
        this.config = config;
    }

    @Override
    public Action process(TreeContext context) throws NodeProcessException {
        // Enrich shared state
        JsonValue newState = context.sharedState.copy();
        newState.put("myKey", "myValue");

        return goToNext().replaceSharedState(newState).build();
    }
}
```

**Outcomes**: Single exit — always continues.
**Key method**: `goToNext()` — inherited from SingleOutcomeNode.
**Common use**: Logging, state enrichment, external API calls (side effects).

---

## Template 3: Multi-Outcome Node (implements Node directly)

```java
package com.example.nodes;

import java.util.List;
import java.util.Set;

import jakarta.inject.Inject;
import org.forgerock.json.JsonValue;
import org.forgerock.openam.annotations.sm.Attribute;
import org.forgerock.openam.auth.node.api.*;
import org.forgerock.util.i18n.PreferredLocales;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import com.google.inject.assistedinject.Assisted;

@Node.Metadata(
    outcomeProvider = MyRouterNode.MyOutcomeProvider.class,   // YOUR custom provider
    configClass = MyRouterNode.Config.class,
    tags = {"risk"},
    i18nFile = "com/example/nodes/MyRouterNode"
)
public class MyRouterNode implements Node {                    // implements Node directly

    private static final Logger logger = LoggerFactory.getLogger(MyRouterNode.class);

    // Outcome ID constants — shared between process() and OutcomeProvider
    private static final String PATH_A = "pathA";
    private static final String PATH_B = "pathB";
    private static final String PATH_C = "pathC";
    private static final String ERROR  = "error";

    public interface Config {
        @Attribute(order = 100)
        default String stateKey() { return "routeValue"; }

        @Attribute(order = 200)
        Set<String> pathAValues();

        @Attribute(order = 300)
        Set<String> pathBValues();
    }

    private final Config config;

    @Inject
    public MyRouterNode(@Assisted Config config) {
        this.config = config;
    }

    @Override
    public Action process(TreeContext context) throws NodeProcessException {
        JsonValue value = context.sharedState.get(config.stateKey());

        if (value == null || value.isNull()) {
            return Action.goTo(ERROR).build();
        }

        String val = value.asString();

        if (config.pathAValues().contains(val)) {
            return Action.goTo(PATH_A)
                .putSessionProperty("route", "A")
                .build();
        }

        if (config.pathBValues().contains(val)) {
            return Action.goTo(PATH_B).build();
        }

        return Action.goTo(PATH_C).build();
    }

    // Custom OutcomeProvider — defines exit connections in tree designer
    public static class MyOutcomeProvider implements OutcomeProvider {
        @Override
        public List<Outcome> getOutcomes(PreferredLocales locales, JsonValue nodeAttributes) {
            return List.of(
                new Outcome(PATH_A, "Path A"),
                new Outcome(PATH_B, "Path B"),
                new Outcome(PATH_C, "Path C"),
                new Outcome(ERROR, "Error")
            );
        }
    }
}
```

**Outcomes**: As many as you define in OutcomeProvider.
**Key method**: `Action.goTo("outcomeId")` — no helper methods, you manage everything.
**Extra imports**: `PreferredLocales`, `OutcomeProvider` — needed for custom OutcomeProvider.
**Rule**: Outcome IDs in `Action.goTo()` MUST match IDs in `new Outcome()` exactly (case-sensitive).

---

## Plugin Class Template

```java
package com.example.nodes;

import java.util.List;
import java.util.Map;
import org.forgerock.openam.auth.node.api.AbstractNodeAmPlugin;
import org.forgerock.openam.auth.node.api.Node;

public class MyPlugin extends AbstractNodeAmPlugin {

    private static final String CURRENT_VERSION = "1.0.0";

    @Override
    protected Map<String, Iterable<? extends Class<? extends Node>>> getNodesByVersion() {
        return Map.of(
            "1.0.0", List.of(
                MyDecisionNode.class,
                MyPassthroughNode.class,
                MyRouterNode.class
            )
            // Adding nodes later:
            // "1.1.0", List.of(NewNode.class)
        );
    }

    @Override
    public String getPluginVersion() {
        return CURRENT_VERSION;
    }
}
```

**Remember**: Bump `CURRENT_VERSION` when adding new nodes, or AM won't register them.
**Dev mode**: Use `"0.0.0"` — AM reinstalls SMS schema every restart.

---

## Service Registration File

**Path**: `src/main/resources/META-INF/services/org.forgerock.openam.plugins.AmPlugin`

```
com.example.nodes.MyPlugin
```

One line. Fully qualified class name. This is how AM discovers your plugin at startup.

---

## Properties File Template

**Path**: `src/main/resources/com/example/nodes/MyDecisionNode.properties`

```properties
nodeDescription=My Decision Node
nodeHelp=Description shown as tooltip in the tree designer.

threshold=Threshold
threshold.help=Help text shown next to this field.

requiredField=Required Field
requiredField.help=This field must be set by the admin.
```

**Naming rules**:
- `nodeDescription` → node name in Components panel
- `nodeHelp` → tooltip
- `methodName` → label for that config field
- `methodName.help` → help icon text
- Key must exactly match Config interface method name

---

## Config Field Types

```java
@Attribute(order = 100)
String text();                                  // text input (required)

@Attribute(order = 200)
default String textWithDefault() { return "x"; } // text input (optional, pre-filled)

@Attribute(order = 300)
default int number() { return 42; }             // number input

@Attribute(order = 400)
default boolean toggle() { return true; }       // checkbox (true = ticked)

@Attribute(order = 500)
default MyEnum dropdown() { return MyEnum.A; }  // dropdown select

@Attribute(order = 600)
Set<String> multiValue();                       // multi-value text (add multiple entries)

@Attribute(order = 700)
Map<String, String> keyValuePairs();            // key-value pair editor
```

---

## Action Builder — Common Patterns

```java
// Route to boolean outcome
return goTo(true).build();
return goTo(false).build();

// Route to single outcome
return goToNext().build();

// Route to named outcome
return Action.goTo("myOutcome").build();

// Modify shared state
JsonValue newState = context.sharedState.copy();   // ALWAYS copy first
newState.put("key", "value");
return goTo(true).replaceSharedState(newState).build();

// Set session properties (available to policies after auth)
return Action.goTo("low")
    .putSessionProperty("riskLevel", "low")
    .build();

// Send callbacks (request user input)
return Action.send(new TextOutputCallback(0, "message")).build();

// Combine everything
return Action.goTo("success")
    .replaceSharedState(newState)
    .putSessionProperty("key", "value")
    .build();
```

---

## TreeContext — Reading Data

```java
// Shared state (non-sensitive, passed between nodes)
String username = context.sharedState.get("username").asString();
int score = context.sharedState.get("riskScore").asInteger();

// Transient state (sensitive — passwords, OTPs — cleared after tree)
String password = context.transientState.get("password").asString();

// HTTP request context
String clientIp = context.request.clientIp;
String host = context.request.hostName;
Map<String, String> cookies = context.request.cookies;

// HTTP headers (ListMultimap — .get() returns List<String>)
List<String> values = context.request.headers.get("X-Forwarded-For");
if (values != null && !values.isEmpty()) {
    String firstValue = values.get(0);
}

// User identity (if already identified by upstream node)
Optional<String> uid = context.universalId;
```

---

## Project Structure

```
custom-nodes/
├── pom.xml
├── src/main/java/com/example/nodes/
│   ├── MyDecisionNode.java
│   ├── MyPassthroughNode.java
│   ├── MyRouterNode.java
│   └── MyPlugin.java
├── src/main/resources/
│   ├── com/example/nodes/
│   │   ├── MyDecisionNode.properties
│   │   ├── MyPassthroughNode.properties
│   │   └── MyRouterNode.properties
│   └── META-INF/services/
│       └── org.forgerock.openam.plugins.AmPlugin
└── lib/                    ← extracted JARs (for local Maven repo)
```

---

## Key Dependencies (all `<scope>provided</scope>`)

| JAR | What it provides |
|-----|-----------------|
| `auth-node-api` | Node, AbstractDecisionNode, SingleOutcomeNode, TreeContext, Action, OutcomeProvider |
| `openam-annotations` | `@Attribute` |
| `openam-plugin-framework` | AmPlugin, PluginTools, PluginException |
| `forgerock-util` | JsonValue |
| `openam-i18n` | PreferredLocales |
| `jakarta.inject-api` | `@Inject` |
| `guice` + `guice-assistedinject` | `@Assisted` |
| `openam-core` | CoreWrapper, AMIdentity (optional) |

---

## Build → Deploy → Test Cycle

```bash
# 1. Build
mvn clean package

# 2. Deploy (copy JAR into AM's WEB-INF/lib/)
MSYS_NO_PATHCONV=1 docker.exe cp target/my-nodes-1.0.0.jar \
  pingam:/usr/local/tomcat/webapps/am/WEB-INF/lib/

# 3. Restart AM
docker.exe restart pingam

# 4. Verify in AM Console → Trees → Components panel
```

---

## Common Mistakes

| Mistake | Symptom | Fix |
|---------|---------|-----|
| `javax.inject` instead of `jakarta.inject` | Compilation error | PingAM 8 = Jakarta EE |
| Missing `@Node.Metadata` | Node doesn't appear in AM | Add annotation above class |
| Missing META-INF/services file | Plugin not discovered | Create the file with FQCN |
| Missing `@Assisted` on Config | All nodes share same config | Add `@Assisted` annotation |
| Outcome ID case mismatch | Tree flow breaks silently | `"low"` ≠ `"Low"` |
| Modifying sharedState directly | State changes lost | `.copy()` then `replaceSharedState()` |
| Same plugin version after adding node | New node not registered | Bump version or use `"0.0.0"` |
| Old JAR still in WEB-INF/lib | Duplicate class errors | Delete old JAR before deploy |
| No AM restart after deploy | Changes not picked up | Nodes load at startup only |
| Bundling provided JARs | Classloader conflicts | Use `<scope>provided</scope>` |

---

## Topics We Didn't Cover (Worth Knowing)

### 1. Callbacks — Collecting User Input
Nodes can prompt users for input mid-authentication using callbacks:
```java
if (!context.hasCallbacks()) {
    // First pass — send the callback to collect input
    return Action.send(new NameCallback("Enter code:")).build();
}
// Second pass — user submitted the callback
String code = context.getCallback(NameCallback.class).get().getName();
```
AM re-calls `process()` after the user responds. Check `hasCallbacks()` to know which pass you're on.

**Callback types**: `NameCallback`, `PasswordCallback`, `TextOutputCallback`, `ChoiceCallback`, `ConfirmationCallback`, `HiddenValueCallback`

### 2. InputState / OutputState Annotations
Declare what shared state keys your node reads and writes:
```java
@Override
public InputState[] getInputs() {
    return new InputState[] { new InputState("username", true) };
}

@Override
public OutputState[] getOutputs() {
    return new OutputState[] { new OutputState("riskScore") };
}
```
AM uses these to validate tree wiring — warns if a node reads a key that no upstream node writes.

### 3. Auxiliary Services — Shared Configuration
When multiple nodes need the same config (e.g., external API URL + credentials), create a shared service instead of duplicating config in each node. Registered as an AM service, configurable per-realm.

### 4. Localization
Create translated `.properties` files:
```
MyNode.properties       ← English (default)
MyNode_fr.properties    ← French
MyNode_ja.properties    ← Japanese
```
AM picks the right one based on admin's browser locale.

### 5. Unit Testing
Use Mockito to mock TreeContext and Config:
```java
TreeContext context = mock(TreeContext.class);
Config config = mock(Config.class);
when(config.threshold()).thenReturn(50);
when(context.sharedState).thenReturn(JsonValue.json(object(field("score", 75))));

MyNode node = new MyNode(config);
Action result = node.process(context);
assertEquals("true", result.outcome);
```

### 6. Secret Store Integration
For nodes that need API keys or passwords, use AM's Secret Store rather than storing in Config:
```java
@Attribute(order = 100)
@Password
char[] apiKey();
```
`@Password` masks the input and encrypts the stored value.

---

*Created: 2026-01-31 — Lab 12 Custom Authentication Nodes*
