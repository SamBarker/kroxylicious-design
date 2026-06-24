# 117 - Reconfiguration Trigger SPI

**Builds on:** [Proposal 083 — Changing Active Proxy Configuration](https://github.com/kroxylicious/design/blob/main/proposals/083-hot-reload-feature.md)

This proposal defines a pluggable Service Provider Interface (SPI) for triggering `KafkaProxy.reconfigure()`. Trigger implementations are discovered via `ServiceLoader`, configured in the proxy's YAML configuration, and are responsible for sourcing new configuration and driving the reconfiguration lifecycle. The SPI formalises the trigger responsibilities established during the design of Proposal 083 and provides the extension point that allows different deployment models — standalone, Kubernetes, embedded — to use different reconfiguration strategies without proxy changes.

## Current situation

Proposal 083 delivered `KafkaProxy.reconfigure(Configuration)` — the core mechanism for applying configuration changes to a running proxy without a full restart. The method accepts a complete `Configuration`, detects what changed, and converges the running state to match.

However, nothing calls `reconfigure()` today. The standalone binary (`kroxylicious-app`) has no way to apply configuration changes at runtime. Operators who embed the proxy can call `reconfigure()` directly from their own code, but the project-shipped binary needs a trigger mechanism to make hot reload usable.

During the Proposal 083 review, several trigger mechanisms were discussed — file watchers, HTTP endpoints, and operator callbacks — but all were explicitly deferred to keep that proposal focused on the reconfiguration machinery itself. The discussion also established that triggers carry significant responsibility: configuration sourcing, static validation, failure policy, rollback, concurrency handling, debouncing, and configuration persistence. These responsibilities need a formal contract.

## Motivation

- **The shipped binary needs hot reload.** Without a trigger mechanism, `kroxylicious-app` cannot use the reconfiguration capability that Proposal 083 introduced. Configuration changes still require a full process restart.

- **Different deployments need different triggers.** A bare-metal deployment watching a config file has different requirements from a Kubernetes operator reconciling a CRD, which has different requirements from a custom control plane using an HTTP API. The trigger mechanism must be pluggable.

- **Trigger authors need a contract.** Proposal 083 pushed substantial responsibility onto triggers — failure policy, rollback, concurrency handling — but that responsibility is currently documented only in PR comments. A formal SPI with documented responsibilities makes it possible for third parties to write correct trigger implementations.

## Proposal

### SPI overview

The trigger SPI consists of three interfaces:

- **`ReconfigurationTrigger`** — the trigger implementation itself, created by the factory, responsible for watching for configuration changes and calling `reconfigure()`.
- **`ReconfigurationTriggerFactory`** — discovered via `ServiceLoader`, responsible for creating a `ReconfigurationTrigger` from its typed configuration.
- **`ReconfigurationTriggerContext`** — provided by the runtime, gives the trigger access to `reconfigure()`, `shutdown()`, source-agnostic configuration parsing, pre-flight validation, and the proxy's startup configuration path.

A proxy has at most one active trigger. Triggers are not composable (unlike filters in a chain). When no trigger is configured, the proxy operates as it does today — hot reload is not available.

### `ReconfigurationTrigger`

```java
/**
 * A reconfiguration trigger watches for configuration changes and drives
 * {@link ReconfigurationTriggerContext#reconfigure(Configuration)} when a
 * change is detected.
 *
 * <h2>Lifecycle</h2>
 * <p>A trigger instance is created by its {@link ReconfigurationTriggerFactory}
 * and has proxy-level lifecycle: one instance exists per proxy, and it lives
 * for the lifetime of the proxy process.
 *
 * <ul>
 *   <li>{@link #start()} is called after the proxy has completed startup and
 *       is serving traffic. The trigger should begin watching for changes.</li>
 *   <li>{@link #close()} is called before proxy shutdown begins. The trigger
 *       should stop watching, release resources, and return promptly. Any
 *       in-flight {@code reconfigure()} call will complete independently.</li>
 * </ul>
 *
 * <h2>Threading</h2>
 * <p>{@code start()} and {@code close()} are called on the proxy's main thread.
 * The trigger is free to create its own threads (e.g. a file watcher thread, an
 * HTTP server thread) but must manage their lifecycle. {@code reconfigure()} is
 * thread-safe and may be called from any thread.
 *
 * <h2>Trigger responsibilities</h2>
 * <p>See the "Trigger responsibilities" section of this proposal for the full
 * contract that trigger implementations must follow.
 */
public interface ReconfigurationTrigger extends Closeable {

    /**
     * Start watching for configuration changes.
     *
     * <p>Called once, after the proxy has completed startup. The trigger should
     * begin watching for changes and call
     * {@link ReconfigurationTriggerContext#reconfigure(Configuration)} when a
     * change is detected. This method should return promptly — long-running
     * work (file watching, HTTP listening) should happen on background threads
     * managed by the trigger.
     *
     * @throws Exception if the trigger cannot start (e.g. cannot open a watch
     *         on the configuration file, cannot bind an HTTP port)
     */
    void start() throws Exception;

    /**
     * Stop watching and release resources. Called before proxy shutdown.
     */
    @Override
    void close();
}
```

### `ReconfigurationTriggerFactory`

```java
/**
 * Factory for creating {@link ReconfigurationTrigger} instances. Discovered
 * via {@link java.util.ServiceLoader}.
 *
 * <p>Each factory declares the type of its configuration object via
 * {@link #configType()}. The runtime deserialises the trigger-specific
 * configuration from the proxy's YAML configuration and passes it to
 * {@link #create(ReconfigurationTriggerContext, Object)}.
 *
 * @param <C> the trigger-specific configuration type. Must be deserializable
 *            from YAML by Jackson.
 */
public interface ReconfigurationTriggerFactory<C> {

    /**
     * Creates a new trigger instance.
     *
     * <p>The trigger is not yet active — the caller will invoke
     * {@link ReconfigurationTrigger#start()} after this method returns.
     *
     * @param context provides access to reconfigure() and configuration parsing
     * @param config  the trigger-specific configuration, deserialized from YAML
     * @return a new trigger instance, ready to be started
     * @throws Exception if the trigger cannot be constructed (e.g. invalid
     *         configuration values)
     */
    ReconfigurationTrigger create(ReconfigurationTriggerContext context, C config) throws Exception;

    /**
     * Returns the type of the trigger-specific configuration object.
     * The runtime uses this to deserialize the {@code config:} section of the
     * trigger's YAML configuration block.
     *
     * @return the configuration class
     */
    Class<C> configType();
}
```

### `ReconfigurationTriggerContext`

```java
/**
 * Runtime context provided to a {@link ReconfigurationTrigger}. This is the
 * trigger's view of the proxy — triggers never interact with
 * {@code KafkaProxy} directly.
 *
 * <p>The context provides four categories of capability:
 * <ul>
 *   <li><b>Reconfiguration</b> — {@link #reconfigure(Configuration)} drives
 *       the proxy to converge to a new configuration.</li>
 *   <li><b>Proxy lifecycle</b> — {@link #shutdown()} initiates an orderly
 *       proxy shutdown, enabling failure policies that terminate the proxy
 *       on unrecoverable errors.</li>
 *   <li><b>Configuration handling</b> — {@link #parseConfiguration(InputStream)}
 *       and {@link #validateConfiguration(Configuration)} allow triggers to
 *       parse and pre-validate configuration from any source before applying
 *       it.</li>
 *   <li><b>Startup context</b> — {@link #configFilePath()} provides the path
 *       the proxy was originally started with.</li>
 * </ul>
 *
 * <p>The context is thread-safe. All methods may be called from any thread.
 */
public interface ReconfigurationTriggerContext {

    /**
     * Apply a new configuration to the running proxy. Delegates to
     * {@link KafkaProxy#reconfigure(Configuration)} — see that method's
     * Javadoc (Proposal 083) for the full contract including error reporting,
     * concurrency control, and scope limitations.
     *
     * @param newConfig the desired end-state configuration; must be non-null
     *                  and statically valid
     * @return a future that completes with a {@link ReconfigureResult}
     *         describing any per-component failures, or completes
     *         exceptionally on catastrophic failure or input rejection
     */
    CompletableFuture<ReconfigureResult> reconfigure(Configuration newConfig);

    /**
     * Initiate an orderly shutdown of the proxy.
     *
     * <p>Triggers use this to implement failure policies that terminate the
     * proxy on unrecoverable errors — for example, shutting down when
     * {@code reconfigure()} returns non-empty {@code errors()}, or as a
     * last resort when a rollback attempt itself fails.
     *
     * <p>This method returns immediately; the actual shutdown proceeds
     * asynchronously. The trigger's {@link ReconfigurationTrigger#close()}
     * method will be called as part of the shutdown sequence.
     */
    void shutdown();

    /**
     * Parse a YAML configuration stream into a {@link Configuration} object.
     *
     * <p>Uses the same parser and static validation rules that the proxy
     * applies at startup, ensuring consistency regardless of how the trigger
     * sources configuration. Triggers may obtain the stream from any source:
     * a file ({@link java.nio.file.Files#newInputStream}), an HTTP request
     * body, an in-memory buffer, etc.
     *
     * @param configurationYaml the YAML configuration as a stream
     * @return the parsed and statically validated configuration
     * @throws ConfigurationException if the stream cannot be read or contains
     *         invalid configuration
     */
    Configuration parseConfiguration(InputStream configurationYaml);

    /**
     * Validate a {@link Configuration} against the proxy's current running
     * state without applying it. Performs the same pre-flight checks that
     * {@link #reconfigure(Configuration)} would perform before beginning
     * any state-changing work — in particular, detecting out-of-scope
     * changes that would cause {@code reconfigure()} to reject the
     * configuration.
     *
     * <p>A trigger can use this to implement a two-phase workflow:
     * validate first, then apply only if validation passes. This catches
     * problems before any virtual cluster experiences downtime.
     *
     * <p>A successful validation does not guarantee that a subsequent
     * {@code reconfigure()} call will succeed — runtime conditions (port
     * availability, upstream reachability) may change between validation
     * and application. But it does guarantee that the configuration will
     * not be rejected for structural or scope reasons.
     *
     * @param config the configuration to validate
     * @throws OutOfScopeChangeException if the configuration differs from
     *         the running configuration in an out-of-scope section
     * @throws ConfigurationException if the configuration fails validation
     */
    void validateConfiguration(Configuration config);

    /**
     * Returns the path to the configuration file the proxy was started with.
     *
     * <p>This is the file path that was passed to the proxy at startup. It
     * does not change during the proxy's lifetime. Triggers may use it as a
     * default watch target, as a baseline for change detection, or to locate
     * configuration relative to the proxy's working directory. Triggers that
     * source configuration from non-file origins may ignore it.
     *
     * @return the startup configuration file path
     */
    Path configFilePath();
}
```

### Configuration model

Triggers are selected and configured via a top-level `reconfigurationTrigger` section in the proxy's YAML configuration:

```yaml
reconfigurationTrigger:
  type: FileWatcher                   # ServiceLoader type name
  config:                             # trigger-specific configuration
    debounceInterval: PT1S            # implementation-specific settings
```

This follows the same `type` + `config` pattern used by filters, routers, and other Kroxylicious plugins.

When the `reconfigurationTrigger` section is absent, no trigger is created and the proxy operates as today — configuration changes require a restart.

The `reconfigurationTrigger` section is **static configuration**: it is not hot-reloadable. Changing the trigger type or its configuration requires a proxy restart. This is consistent with Proposal 083's scope limitations — `reconfigure()` applies only virtual-cluster and filter configuration; other sections (including the trigger section) are out of scope and will cause `reconfigure()` to reject the configuration with `OutOfScopeChangeException` if they differ.

### Trigger responsibilities

Proposal 083 defined `KafkaProxy.reconfigure()` as a minimal operation that reports outcomes without taking policy action. This was a deliberate design choice — it pushes failure handling, rollback, and operational policy onto the caller. For triggers, "the caller" is the trigger implementation. The following responsibilities form the contract that trigger implementations must satisfy.

#### Configuration sourcing

The trigger is responsible for obtaining and delivering a new `Configuration` to `reconfigure()`. How the configuration is sourced — watching a file, receiving an HTTP request, responding to a CRD reconciliation — is the trigger's concern. The `ReconfigurationTriggerContext` provides `parseConfiguration(InputStream)` so triggers can parse YAML from any source (file, HTTP body, in-memory buffer) using the same parser and validation rules the proxy applies at startup. Triggers may also construct `Configuration` objects programmatically if their source is not YAML.

#### Static validation

Static validation (schema conformance, required fields, field-value ranges, internal consistency) must be performed on the new configuration **before** calling `reconfigure()`. This is the caller's responsibility per Proposal 083's contract. `parseConfiguration(InputStream)` performs static validation as part of parsing. Triggers that construct `Configuration` objects directly must ensure equivalent validation. Additionally, `validateConfiguration(Configuration)` allows triggers to check for out-of-scope changes and other pre-flight failures before committing to a reconfiguration that would disrupt traffic — enabling a validate-then-apply workflow.

#### Failure policy

The proxy does not act on `ReconfigureResult.errors()`. The trigger expresses its failure policy via `whenComplete()` on the returned future. Three canonical patterns are defined in Proposal 083:

- **Shut down on any failure** — call `proxy.shutdown()` if `errors()` is non-empty
- **Best-effort** — log failures, take no proxy-level action; surviving VCs continue serving
- **Rollback on failure** — call `reconfigure(oldConfig)` when `errors()` is non-empty

The choice between these (or a custom policy) is the trigger's decision, typically determined at deployment time by the trigger's configuration or hardcoded by the trigger implementation.

#### Previous configuration tracking

Triggers that support rollback must maintain their own record of the previous known-good configuration. The proxy does not expose a getter for its running configuration. Triggers typically have a natural source-of-truth for this: a previous file snapshot, a ConfigMap revision, an HTTP request history.

#### Concurrency handling

`reconfigure()` rejects concurrent calls with `ConcurrentReconfigureException` (the future completes exceptionally). The trigger **must not** treat this as a real failure:

- **Do not shut down** — the proxy is healthy; another reconfiguration is in flight.
- **Do not roll back** — rolling back would undo the other reconfiguration's changes.
- **Do retry** — typically after a short delay, with the most recent desired configuration.

The recommended discrimination is `ex instanceof ConcurrentReconfigureException` — respond with retry or no-op rather than a destructive policy.

#### Out-of-scope change handling

`reconfigure()` rejects configurations that differ in out-of-scope sections with `OutOfScopeChangeException` (the future completes exceptionally). Like `ConcurrentReconfigureException`, this means the proxy did not change state. The trigger should log the rejection and **not** apply destructive policies (shutdown, rollback).

#### Debouncing

Since concurrent `reconfigure()` calls are rejected rather than queued, triggers that may receive rapid configuration changes (e.g. a file watcher receiving multiple filesystem events during an atomic file replacement) must debounce internally. The pattern is: absorb events for a short window, then call `reconfigure()` with the latest configuration. The debounce interval is a trigger-specific configuration concern.

#### Configuration persistence

Whether to persist the applied configuration to disk is a trigger concern. A Kubernetes operator owns configuration state via CRD and does not want the proxy overwriting files. A bare-metal file watcher may not need persistence because the file is already the source of truth. A custom trigger may persist to a database. The proxy takes no action on configuration persistence.

#### Change detection (optimisation)

Triggers may perform their own change detection to avoid unnecessary `reconfigure()` calls. For example, a Kubernetes operator might compare ConfigMap checksums to skip no-op reconciliation loops. The proxy performs its own change detection internally (it will not restart unaffected virtual clusters), so trigger-level detection is an optimisation, not a correctness requirement.

### Trigger lifecycle

The trigger lifecycle is tied to the proxy's lifecycle:

```
Proxy startup
    │
    ├── Parse proxy configuration (including reconfigurationTrigger section)
    ├── Discover ReconfigurationTriggerFactory via ServiceLoader
    ├── Deserialize trigger-specific config
    ├── Call factory.create(context, config) → ReconfigurationTrigger
    │
    ├── Proxy completes startup (VCs serving)
    │
    ├── Call trigger.start()
    │     │
    │     ├── Success: trigger is active, watching for changes
    │     └── Failure: log warning, proxy continues without hot reload
    │
    ├── ... proxy running, trigger calling reconfigure() as needed ...
    │
    ├── Proxy shutdown initiated
    │
    ├── Call trigger.close()
    │     (trigger stops watching, releases resources)
    │
    └── Proxy completes shutdown
```

**Startup ordering.** The trigger is started *after* the proxy has completed startup and all virtual clusters are serving. This is why the `ReconfigurationTrigger` interface separates construction (`create()`) from activation (`start()`): the factory creates the trigger during proxy initialisation, but the trigger must not begin watching for changes — or call `reconfigure()` — until the proxy is ready. Without this separation, a file watcher trigger could detect the existing configuration file immediately on construction and attempt a `reconfigure()` before the proxy has loaded its initial configuration, which would throw `IllegalStateException` per Proposal 083.

**Failure to start.** If the trigger's `start()` method throws, the proxy logs a warning and continues running without hot reload capability. This is a pragmatic choice: the proxy is functional and serving traffic; the operator can diagnose the trigger failure and restart the proxy if hot reload is required. Failing the entire proxy startup because a trigger couldn't start would be disproportionate.

**Shutdown ordering.** The trigger is closed *before* the proxy begins its shutdown sequence. This prevents the trigger from attempting a `reconfigure()` call while the proxy is shutting down. Any `reconfigure()` call already in flight will complete independently — the proxy handles the `IllegalStateException` case per Proposal 083.

**In-flight reconfiguration at shutdown.** If a trigger-initiated `reconfigure()` is in progress when the proxy receives a shutdown signal, the proxy waits for the reconfiguration to complete before proceeding with shutdown. The trigger's `close()` is called after the reconfiguration completes.

### Example: File watcher trigger

To illustrate the SPI in use, here is a sketch of how a file watcher trigger would be structured. This is not a specification for a file watcher — that is an implementation concern — but demonstrates that the SPI is sufficient for the most common trigger pattern.

```java
public class FileWatcherTriggerFactory
        implements ReconfigurationTriggerFactory<FileWatcherConfig> {

    @Override
    public ReconfigurationTrigger create(
            ReconfigurationTriggerContext context,
            FileWatcherConfig config) {
        return new FileWatcherTrigger(context, config);
    }

    @Override
    public Class<FileWatcherConfig> configType() {
        return FileWatcherConfig.class;
    }
}

// Registered in META-INF/services/...ReconfigurationTriggerFactory
```

```java
class FileWatcherTrigger implements ReconfigurationTrigger {

    private final ReconfigurationTriggerContext context;
    private final FileWatcherConfig config;
    private WatchService watchService;
    private Thread watchThread;

    // ... constructor ...

    @Override
    public void start() throws Exception {
        Path watchPath = context.configFilePath();
        watchService = FileSystems.getDefault().newWatchService();
        // register watch on parent directory (handles K8s ConfigMap symlinks)
        watchPath.getParent().register(watchService, ENTRY_MODIFY, ENTRY_CREATE);

        watchThread = new Thread(() -> {
            while (!Thread.currentThread().isInterrupted()) {
                // wait for events, debounce, then:
                try (InputStream in = Files.newInputStream(watchPath)) {
                    Configuration newConfig = context.parseConfiguration(in);
                    context.validateConfiguration(newConfig);
                    context.reconfigure(newConfig)
                           .whenComplete((result, ex) -> {
                               if (ex instanceof ConcurrentReconfigureException) {
                                   // retry later
                                   return;
                               }
                               if (ex != null) {
                                   LOG.error("Reconfigure failed", ex);
                                   return;
                               }
                               for (var error : result.errors()) {
                                   LOG.error("Component failed: {}",
                                             error.humanReadableIdentifier(),
                                             error.cause());
                               }
                           });
                } catch (ConfigurationException e) {
                    LOG.error("Failed to parse or validate configuration", e);
                }
            }
        });
        watchThread.setDaemon(true);
        watchThread.start();
    }

    @Override
    public void close() {
        watchThread.interrupt();
        watchService.close();
    }
}
```

This example demonstrates:
- The trigger manages its own threads
- `parseConfiguration(InputStream)` parses from any source — here a file, but equally an HTTP body or in-memory buffer
- `validateConfiguration()` catches out-of-scope changes before any VC experiences downtime
- `whenComplete()` implements failure policy (best-effort in this case)
- `ConcurrentReconfigureException` is handled with retry semantics
- The trigger uses `configFilePath()` as its default watch target

## Affected projects

- **kroxylicious-runtime** (`kroxylicious-api` module) — the SPI interfaces (`ReconfigurationTrigger`, `ReconfigurationTriggerFactory`, `ReconfigurationTriggerContext`) are added as public API.
- **kroxylicious-runtime** (runtime module) — implements `ReconfigurationTriggerContext`, integrates trigger lifecycle with `KafkaProxy` startup and shutdown, and performs ServiceLoader discovery.
- **kroxylicious-app** — configures a trigger (initially a file watcher, shipped as a separate module) for the standalone binary.
- **kroxylicious-operator** — not directly affected. The operator embeds the proxy and calls `reconfigure()` directly; it does not use the trigger SPI. If a future operator design prefers to delegate to an in-proxy trigger, it can configure one via the SPI.

## Compatibility

- **Additive.** No existing behaviour changes. A proxy with no `reconfigurationTrigger` configuration operates identically to today.
- **Proposal 083 unchanged.** The `KafkaProxy.reconfigure()` contract, `ReconfigureResult`, `ReconfigureError`, concurrency control, and scope limitations are unchanged.
- **Configuration format.** The `reconfigurationTrigger` section is new; its absence is a no-op. Because it is an out-of-scope section for `reconfigure()`, any change to it in a new configuration will be rejected with `OutOfScopeChangeException` — which is the correct behaviour (trigger changes require a restart).
- **Plugin convention.** The `type` + `config` pattern and `ServiceLoader` discovery follow established Kroxylicious conventions and do not introduce new mechanisms.

## Rejected alternatives

- **Per-call `ReloadOptions`**: An earlier Proposal 083 iteration proposed a `ReloadOptions` parameter on each `reconfigure()` call carrying failure policy (rollback/terminate) and persistence settings. Rejected because failure policy is a deployment-time decision that should not vary between invocations. A trigger that hardcodes "best-effort" and another that hardcodes "rollback" should not be able to vary their behaviour per call — that creates inconsistency. The `whenComplete()` pattern achieves the same expressiveness without a per-call parameter.

- **`VirtualClusterLifecycleObserver`**: An earlier Proposal 083 iteration proposed a push-based observer injected at `KafkaProxy` construction time, notified of every lifecycle transition. While valuable for a future control-plane integration, it is a broader mechanism than triggers need and was deferred to avoid coupling it with the trigger SPI. The `whenComplete()` pattern on `reconfigure()` is sufficient for the failure-handling use case.

- **Multiple simultaneous triggers**: Considered allowing multiple triggers to be active (e.g. both a file watcher and an HTTP endpoint). Rejected because `reconfigure()` only allows one reconfiguration at a time (`ConcurrentReconfigureException`), and multiple triggers racing to reconfigure would create unpredictable behaviour. If a deployment needs both file-based and HTTP-based triggering, a single trigger implementation can support both input mechanisms internally.

- **Trigger signals "config changed" without providing `Configuration`**: An alternative where the trigger simply signals "reload" and the runtime re-reads and parses the configuration file. Simpler for file-based triggers but does not support non-file configuration sources (HTTP request bodies, CRD specs, programmatically generated configuration). The `parseConfiguration(InputStream)` method on `ReconfigurationTriggerContext` gives file-based triggers the same simplicity (open a stream, call parse) while preserving flexibility for other sources.

- **Hardcoded trigger in `kroxylicious-app`**: Instead of an SPI, wire a file watcher directly into the standalone binary. Rejected because it forces users who need a different trigger mechanism (HTTP, custom control plane) to embed the proxy rather than just providing a different trigger on the classpath. The SPI cost is small and the extensibility value is high.

- **Proxy-managed configuration persistence**: An earlier design had the proxy persist the applied configuration to disk after a successful `reconfigure()`. Rejected because persistence requirements vary by deployment: a Kubernetes operator owns state via CRD and does not want the proxy overwriting files; a bare-metal deployment may want file persistence; a custom control plane may persist to a database. This is a trigger concern, not a proxy concern.

- **Trigger configuration in `updateStrategy` or `configurationReload` YAML block**: Earlier iterations of Proposal 083 proposed YAML-level configuration for failure policy and rollback behaviour. Rejected in favour of caller-side policy via `whenComplete()` — the proxy reports outcomes and takes no policy action. The only YAML configuration for triggers is the `reconfigurationTrigger` section that selects and configures the trigger implementation.
