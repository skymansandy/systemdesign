# Feature Flags

Designing a feature flag and experimentation platform (LaunchDarkly, Firebase Remote Config, GrowthBook, Spotify FIDO) is deceptively deep. On the surface it looks like "if/else with a remote config." In practice it touches local evaluation engines, consistent hashing for experiment bucketing, real-time flag propagation, offline resilience, metrics pipelines, and SDK design that must never regress app startup time. Interviewers use this topic to test whether you can design a platform that other engineers consume -- an SDK is an API with very different constraints than a backend service.

What makes the mobile SDK uniquely hard: it runs inside someone else's app, so it cannot crash, block the main thread, or consume excessive memory. Flags must evaluate in microseconds, not milliseconds. The device goes offline and the app still needs flag values. Experiment assignment must be deterministic and sticky across app restarts, reinstalls, and even devices. And you must collect exposure events without becoming a battery-draining telemetry hog.

---

## Scoping the Problem

The first thing I'd nail down is **client-side vs. server-side evaluation** -- this is the single biggest architectural decision. Client-side means the SDK has all targeting rules locally and evaluates flags without network calls. Server-side means every evaluation is an API call. For mobile, client-side wins decisively.

Next, I'd ask about flag volume. 50 flags vs. 5,000 flags per project changes the payload size, memory strategy, and sync approach. Then: do we need experimentation (A/B testing) or just feature flags? Experimentation adds sticky bucketing, exposure tracking, and statistical rigor.

Other questions that meaningfully shape the design:

- **Targeting complexity?** Simple on/off per user vs. percentage rollouts vs. multi-variate targeting rules with segments.
- **Real-time updates or polling?** Does a flag change need to propagate in seconds (SSE) or are periodic polls (minutes) acceptable?
- **Offline requirement?** Must the SDK return meaningful values when the device has no connectivity?
- **Multi-platform?** Android-only or KMP shared SDK with iOS? This drives the shared-code boundary.
- **Host app's startup budget?** If the app must be interactive in 800ms, the SDK cannot block for 200ms fetching flags.
- **Kill switch latency target?** How fast must a bad flag be disabled globally?
- **Privacy constraints?** Can user attributes be sent to the flag service, or must evaluation happen entirely on-device?

**Core scope for this design:** evaluate typed flags (Boolean, String, Int, JSON), percentage rollouts, user targeting, A/B test assignment with sticky bucketing, real-time updates via SSE, exposure tracking, offline evaluation, debug overlay, and local developer overrides.

**Key non-functional priorities:**

- **Evaluation latency** -- sub-1ms per flag. Flag checks happen in render paths; blocking means jank.
- **SDK init time** -- under 50ms added to cold start. Host apps have strict startup budgets.
- **Memory overhead** -- under 2MB for 500 flags. The SDK shares memory with the host app.
- **Update propagation** -- under 30s for real-time, under 5min for polling. Kill switches must be fast.
- **Offline resilience** -- 100% evaluation success with cached data. Never throw or return null when offline.
- **Exposure delivery** -- 99.9% eventual delivery. Missing events corrupt experiment analysis.
- **Battery impact** -- negligible. Real-time connections and telemetry can drain battery if careless.
- **Binary size** -- under 500KB added to APK. Large SDKs get rejected by app teams.

!!! note "Mobile vs. Backend SDK Constraints"
    The constraints above are what separate mobile SDK design from backend SDK design. A backend SDK can call the server per evaluation (low datacenter latency), keep state in memory (long-lived process), flush telemetry immediately, and use generous thread pools. A mobile SDK must evaluate locally (network round-trip is unacceptable), persist to disk (process is killed frequently), batch and compress telemetry (battery), and never block the main thread. Every design decision in this document flows from these constraints.

---

## API Design

### SDK Public API Surface

The SDK API is what thousands of developers consume daily. It must be minimal, type-safe, and impossible to misuse.

=== "Kotlin (KMP)"

    ```kotlin
    // Initialization -- non-blocking, returns immediately
    val flagClient = FeatureFlagClient.init(
        config = FlagConfig(
            sdkKey = "sdk-mobile-prod-abc123",
            pollingIntervalMs = 300_000, // 5 min fallback
            streamingEnabled = true,
            offlineMode = false,
        ),
        context = userContext {
            key("usr_abc123")
            attribute("country", "US")
            attribute("plan", "premium")
            attribute("app_version", "4.2.0")
        },
    )

    // Typed evaluation -- synchronous, never throws
    val showNewCheckout: Boolean = flagClient.boolVariation("new_checkout", defaultValue = false)
    val checkoutVersion: String = flagClient.stringVariation("checkout_version", defaultValue = "v1")
    val maxRetries: Int = flagClient.intVariation("max_retries", defaultValue = 3)
    val cardLayout: JsonObject = flagClient.jsonVariation("card_layout", defaultValue = defaultLayout)

    // Detailed evaluation (for debugging / analytics)
    val detail: EvaluationDetail<Boolean> = flagClient.boolVariationDetail("new_checkout", false)
    // detail.value = true
    // detail.reason = EvalReason.TargetMatch(ruleIndex = 2)
    // detail.variationIndex = 1

    // Listen for flag changes (reactive)
    flagClient.observe("new_checkout").collect { newValue: Boolean ->
        // Re-render UI when flag changes
    }

    // Update user context (e.g., after login)
    flagClient.identify(updatedContext)

    // Cleanup
    flagClient.close()
    ```

=== "Compose Integration"

    ```kotlin
    @Composable
    fun CheckoutScreen() {
        val showNewCheckout by rememberFlag("new_checkout", defaultValue = false)

        if (showNewCheckout) {
            NewCheckoutFlow()
        } else {
            LegacyCheckoutFlow()
        }
    }

    @Composable
    fun <T> rememberFlag(key: String, defaultValue: T): State<T> {
        val client = LocalFlagClient.current
        return client.observe(key)
            .collectAsState(initial = client.variation(key, defaultValue))
    }
    ```

### Evaluation Model: Local vs. Server vs. Hybrid

| Approach | Evaluation | Latency | Offline | Used By |
|----------|-----------|---------|---------|---------|
| **Local evaluation (rules on device)** | SDK evaluates targeting rules locally | ~0.1ms | Full support | LaunchDarkly, GrowthBook |
| **Server evaluation (API per flag)** | Server evaluates, returns value | 50-200ms | No support | Firebase Remote Config (fetch model) |
| **Hybrid (server eval + local cache)** | Server evaluates, SDK caches results | 0.1ms cached, 50ms miss | Cached values only | Optimizely |

I'd go with **local evaluation with server-synced rules**. Sub-millisecond evaluation is non-negotiable -- flag checks happen in Compose recomposition and RecyclerView binding. Even 5ms per flag is unacceptable if you check 20 flags during screen load. Full offline evaluation works because the SDK has all targeting rules cached locally. User attributes never leave the device for evaluation (privacy win). And there's no dependency on server availability -- if the flag service goes down, the app works fine with cached rules.

Server evaluation kills offline support entirely and makes every flag check a potential failure point. Hybrid gives you the worst of both worlds: you still need network for fresh evaluations, and cached values go stale without the SDK knowing rules changed.

!!! tip "Pro Tip"
    LaunchDarkly's mobile SDK downloads the entire flag configuration for a specific user context (pre-evaluated for the user but with rules for local re-evaluation). GrowthBook downloads the raw targeting rules and evaluates entirely on-device. Both are valid -- the tradeoff is payload size vs. evaluation complexity.

### Flag Configuration Endpoints

=== "Initial Fetch"

    ```http
    GET /api/v1/sdk/flags
    Authorization: Bearer sdk-mobile-prod-abc123
    X-Context-Hash: sha256(user_context)
    If-None-Match: "etag-abc123"

    Response 200:
    {
      "flags": {
        "new_checkout": {
          "key": "new_checkout",
          "version": 42,
          "type": "boolean",
          "defaultValue": false,
          "targeting": {
            "rules": [
              {
                "conditions": [
                  { "attribute": "country", "op": "in", "values": ["US", "CA"] },
                  { "attribute": "plan", "op": "eq", "value": "premium" }
                ],
                "variation": 1
              }
            ],
            "fallthrough": { "rollout": { "variations": [0, 1], "weights": [70, 30] } }
          },
          "variations": [false, true],
          "salt": "flag-salt-xyz"
        }
      },
      "segments": {
        "beta_users": {
          "included": ["usr_abc", "usr_def"],
          "rules": [{ "attribute": "created_at", "op": "after", "value": "2025-01-01" }]
        }
      },
      "etag": "etag-def456"
    }
    ```

=== "Delta Update (SSE)"

    ```http
    GET /api/v1/sdk/flags/stream
    Authorization: Bearer sdk-mobile-prod-abc123
    Accept: text/event-stream

    event: flag-update
    data: {"key":"new_checkout","version":43,"targeting":{...}}

    event: flag-delete
    data: {"key":"old_experiment","version":44}

    event: heartbeat
    data: {"timestamp":1715100000}
    ```

=== "Event Tracking"

    ```http
    POST /api/v1/sdk/events
    Authorization: Bearer sdk-mobile-prod-abc123
    Content-Encoding: gzip

    {
      "events": [
        {
          "type": "exposure",
          "flagKey": "new_checkout",
          "variation": 1,
          "value": true,
          "reason": "RULE_MATCH",
          "userKey": "usr_abc123",
          "timestamp": 1715100000000
        },
        {
          "type": "custom",
          "eventKey": "checkout_completed",
          "userKey": "usr_abc123",
          "value": 49.99,
          "timestamp": 1715100005000
        }
      ]
    }
    ```

### Polling vs. Streaming

I'd use **SSE primary + polling fallback**. SSE gives near-real-time updates with simpler reconnection than WebSocket -- the client handles reconnect and `Last-Event-ID` natively. We don't need bidirectional communication; the SDK only receives flag updates. Polling kicks in when SSE is unavailable (corporate proxies, restrictive networks).

For efficiency: initial fetch downloads all flags in a single request (~50-200KB gzipped for 500 flags). SSE sends only changed flags. Event batching collects exposures in memory, flushing every 30 seconds or at 100 events, compressed with gzip. `If-None-Match` on polling returns `304 Not Modified` when flags haven't changed.

!!! warning "Edge Case"
    Some corporate networks and mobile carriers strip SSE connections after 30-60 seconds. The SDK must detect a stale SSE connection (no heartbeat received within expected interval) and fall back to polling automatically. LaunchDarkly's mobile SDK implements exactly this pattern.

---

## SDK Architecture

### Component Architecture

```mermaid
graph TB
    subgraph Host["Host Application"]
        UI["UI Layer<br/>(Compose/Views)"]
        DI["DI Container<br/>(Hilt/Koin)"]
    end

    subgraph SDK["Feature Flag SDK"]
        API["Public API<br/>FlagClient"]
        EVAL["Evaluation<br/>Engine"]
        CACHE["In-Memory<br/>Cache"]
        STORE["Persistent<br/>Store"]
        SYNC["Sync Manager<br/>(SSE + Polling)"]
        EVENTS["Event<br/>Processor"]
        CONTEXT["Context<br/>Manager"]
        OVERRIDE["Override<br/>Manager"]
    end

    subgraph Remote["Remote Services"]
        FLAGS["Flag Config<br/>CDN/API"]
        STREAM["SSE<br/>Endpoint"]
        INGEST["Event<br/>Ingestion"]
    end

    UI -->|"variation(key)"| API
    DI -->|provides| API
    API --> EVAL
    EVAL --> CACHE
    EVAL --> OVERRIDE
    EVAL --> CONTEXT
    CACHE --> STORE
    SYNC --> CACHE
    SYNC --> FLAGS
    SYNC --> STREAM
    EVENTS --> INGEST
    API -->|"track exposure"| EVENTS
```

**FlagClient** is the single entry point -- thread-safe, callable from any thread. **Evaluation Engine** evaluates targeting rules against user context as a pure function with no I/O (~0.1ms). **In-Memory Cache** holds flag configs in a `ConcurrentHashMap` with lock-free reads. **Persistent Store** writes configs to disk (DataStore/SQLite) on `Dispatchers.IO` and reads on cold start. **Sync Manager** manages the SSE connection + polling fallback and pushes updates into the in-memory cache. **Event Processor** batches exposure events, compresses, and flushes on a background coroutine. **Context Manager** holds the user context and notifies sync on `identify()`. **Override Manager** stores developer-forced flag values for debug builds.

### KMP Alignment

The evaluation engine is the crown jewel of the shared layer -- pure computation with zero platform dependencies (rules, context, math). It lives entirely in `commonMain` alongside the in-memory cache logic, SSE parsing, polling/retry logic, event batching, and the full `FlagClient` API. Platform-specific code is limited to: persistence (`DataStore` on Android, `NSUserDefaults` on iOS), network transport (Ktor with OkHttp/Darwin engines), lifecycle-aware flush triggers, and Compose integration (`rememberFlag`).

!!! tip "Pro Tip"
    LaunchDarkly open-sources their evaluation engine, and it is completely platform-agnostic. This is the ideal candidate for 100% shared KMP code.

### SDK States

The SDK transitions through: **Initializing** (loading from disk, evaluations use cached/default values), **Ready** (initial fetch complete, in-memory cache populated), **Streaming** (SSE active, changes reflected in seconds), **Stale** (cached data older than threshold -- still evaluates but logs warning and triggers background refresh), **Offline** (no connectivity -- evaluates from disk cache, events queued), and **Error** (server returned bad data -- falls back to cache, never crashes).

---

## Data Flow for Core Scenarios

### Flag Evaluation (Hot Path)

```mermaid
sequenceDiagram
    participant UI as Compose UI
    participant API as FlagClient
    participant OVR as Override Manager
    participant EVAL as Eval Engine
    participant CACHE as In-Memory Cache
    participant EVT as Event Processor

    UI->>API: boolVariation("new_checkout", false)
    API->>OVR: Check override for "new_checkout"
    OVR-->>API: No override
    API->>CACHE: Get flag config "new_checkout"
    CACHE-->>API: FlagConfig(rules, variations, salt)
    API->>EVAL: evaluate(flagConfig, userContext)
    Note over EVAL: Match rules top-to-bottom<br/>First match wins<br/>Fallthrough → percentage rollout
    EVAL-->>API: EvalResult(value=true, variationIndex=1, reason=RULE_MATCH)
    API->>EVT: trackExposure("new_checkout", 1, RULE_MATCH)
    Note over EVT: Deduplicate: only track<br/>first exposure per flag per session
    API-->>UI: true
```

### A/B Test Assignment (Sticky Bucketing)

```mermaid
sequenceDiagram
    participant APP as Application
    participant API as FlagClient
    participant EVAL as Eval Engine
    participant BUCKET as Bucket Store
    participant EVT as Event Processor

    APP->>API: stringVariation("exp_pricing", "control")
    API->>BUCKET: Check sticky assignment for "exp_pricing" + user
    BUCKET-->>API: No previous assignment
    API->>EVAL: evaluate with rollout
    Note over EVAL: hash(flag.salt + user.key) → 0.7234<br/>Bucket ranges: control [0, 0.5), variant_a [0.5, 0.75), variant_b [0.75, 1.0)<br/>0.7234 → variant_a
    EVAL-->>API: EvalResult(value="variant_a", variationIndex=1)
    API->>BUCKET: Persist assignment: exp_pricing → variant_a
    API->>EVT: trackExposure("exp_pricing", 1, EXPERIMENT)
    API-->>APP: "variant_a"

    Note over APP: Later, flag rollout percentages change...
    APP->>API: stringVariation("exp_pricing", "control")
    API->>BUCKET: Check sticky assignment
    BUCKET-->>API: Previously assigned: variant_a
    API-->>APP: "variant_a" (sticky -- ignores new percentages)
```

### Flag Update Propagation (Real-Time)

```mermaid
sequenceDiagram
    participant DASH as Dashboard
    participant SERVER as Flag Service
    participant SSE as SSE Stream
    participant SYNC as Sync Manager
    participant CACHE as In-Memory Cache
    participant STORE as Persistent Store
    participant UI as Compose UI

    DASH->>SERVER: Update "new_checkout" targeting
    SERVER->>SSE: Publish flag-update event
    SSE->>SYNC: event: flag-update, data: {new config}
    SYNC->>SYNC: Validate version > current
    SYNC->>CACHE: Update in-memory cache
    SYNC->>STORE: Persist to disk (async)
    CACHE-->>UI: Flow emission triggers recomposition
    Note over UI: rememberFlag("new_checkout")<br/>automatically recomposes
```

### Offline Evaluation

```mermaid
sequenceDiagram
    participant APP as Application
    participant API as FlagClient
    participant CACHE as In-Memory Cache
    participant STORE as Persistent Store

    Note over API: Network unavailable
    APP->>API: boolVariation("new_checkout", false)
    API->>CACHE: Get flag config
    alt Cache populated (warm start)
        CACHE-->>API: FlagConfig from last sync
        API-->>APP: true (from cached rules)
    else Cache empty (cold start, no prior sync)
        CACHE-->>API: Miss
        API->>STORE: Read from disk
        alt Disk cache exists
            STORE-->>API: FlagConfig from disk
            API-->>APP: true (from persisted rules)
        else No disk cache (first launch ever)
            STORE-->>API: Miss
            API-->>APP: false (default value)
            Note over API: Log: "Flag evaluated with default,<br/>no cached configuration available"
        end
    end
```

---

## Design Deep Dives

### Client-Side Evaluation Engine

The evaluation engine is the core of the SDK -- a pure function with no I/O or side effects:

```kotlin
fun evaluate(flag: FlagConfig, context: UserContext): EvalResult {
    // 1. Check if flag is globally off
    if (!flag.on) return EvalResult(flag.offVariation, Reason.OFF)

    // 2. Check individual targeting (specific user keys)
    flag.targets.forEach { target ->
        if (context.key in target.values) {
            return EvalResult(flag.variations[target.variation], Reason.TARGET_MATCH)
        }
    }

    // 3. Evaluate rules top-to-bottom (first match wins)
    flag.rules.forEachIndexed { index, rule ->
        if (rule.conditions.all { it.matches(context) }) {
            val variation = rule.resolveVariation(context, flag.salt)
            return EvalResult(flag.variations[variation], Reason.RULE_MATCH(index))
        }
    }

    // 4. Fallthrough (percentage rollout or fixed variation)
    val variation = flag.fallthrough.resolveVariation(context, flag.salt)
    return EvalResult(flag.variations[variation], Reason.FALLTHROUGH)
}
```

Condition matching supports: `eq` (exact match), `in` (set membership), `gt/lt/gte/lte` (semantic version comparison), `contains` (substring), `matches` (regex), `segment` (segment membership lookup), and `before/after` (date comparison).

!!! warning "Edge Case"
    Semantic version comparison is tricky. `"4.10.0"` must be greater than `"4.9.0"`, but naive string comparison says otherwise. The evaluation engine must parse versions into `(major, minor, patch)` tuples. LaunchDarkly has a dedicated `semver` operator for this reason.

The rules engine uses `AND` logic within a rule and `OR` across rules (first match wins). **Segments** are reusable groups of users referenced across multiple flags, containing explicit include/exclude lists and dynamic rules:

```kotlin
data class TargetingRule(
    val conditions: List<Condition>,  // AND logic within a rule
    val variation: Int?,               // Fixed variation
    val rollout: Rollout?,            // Percentage-based variation
)

data class Segment(
    val key: String,
    val included: Set<String>,    // Explicitly included user keys
    val excluded: Set<String>,    // Explicitly excluded user keys
    val rules: List<SegmentRule>, // Dynamic rules (same condition format)
)
```

!!! note "How Uber Targets"
    Uber's feature flag platform supports "geo-fenced" targeting where flags evaluate based on the user's current city. This is just a custom attribute (`city = "San Francisco"`) updated in the context when location changes. The targeting engine itself doesn't know about geography -- it just matches attribute values.

### Two-Tier Caching

```mermaid
graph LR
    EVAL["Evaluation<br/>Engine"] -->|"~0.01ms"| MEM["In-Memory Cache<br/>(ConcurrentHashMap)"]
    MEM -->|"~1ms"| DISK["Disk Cache<br/>(DataStore/Files)"]
    DISK -->|"~50ms"| NET["Network<br/>(SSE/Polling)"]
    
    style MEM fill:#2d5,stroke:#333,color:#000
    style DISK fill:#da2,stroke:#333,color:#000
    style NET fill:#d55,stroke:#333,color:#000
```

On SDK init, the in-memory cache is empty but disk may have data -- load disk to memory asynchronously (non-blocking). On first fetch, both caches are populated. SSE updates hit in-memory immediately and write to disk async. On app restart, disk rehydrates memory -- stale but functional. Disk cache expires after 30 days of no update.

For first-launch scenarios where no cache exists, the SDK supports embedding a **bootstrap payload** in the app binary:

```kotlin
val flagClient = FeatureFlagClient.init(
    config = FlagConfig(
        sdkKey = "sdk-mobile-prod-abc123",
        bootstrap = BootstrapSource.Resource("flags_bootstrap.json"), // bundled in APK
    ),
    context = userContext { key("usr_abc123") },
)
```

The bootstrap file is generated at build time and bundled into the APK. Never fresher than the last build, but guarantees the SDK always has values on first launch.

!!! tip "Pro Tip"
    Netflix bundles a default flag configuration in their app binary and calls it the "genesis config." This ensures the app can render a complete experience even if the flag service is unreachable on first launch. The config is updated on every app release build.

### Experiment Assignment & Consistent Hashing

Experiment assignment must be deterministic and sticky. The same user must always get the same variant, even across app restarts, reinstalls, and devices.

```kotlin
fun assignBucket(flagSalt: String, userKey: String): Double {
    val input = "$flagSalt.$userKey"
    val hash = murmurHash3(input.encodeToByteArray())
    return (hash.toUInt().toDouble()) / UInt.MAX_VALUE.toDouble()
}

fun resolveVariation(context: UserContext, flagSalt: String, rollout: Rollout): Int {
    val bucket = assignBucket(flagSalt, context.key)
    var cumulative = 0.0
    for ((variation, weight) in rollout.weightedVariations) {
        cumulative += weight / 100_000.0  // Weights in millipercents for precision
        if (bucket < cumulative) return variation
    }
    return rollout.weightedVariations.last().variation
}
```

I'd use **MurmurHash3** -- it's the industry standard for bucketing (LaunchDarkly, GrowthBook). Very fast with excellent uniformity. SHA-256 is cryptographic overkill (10-50x slower). CRC32 has poor uniformity that creates assignment bias. xxHash is a good alternative but MurmurHash has wider ecosystem support.

When a user is assigned to an experiment variant, the assignment is persisted in a **sticky bucket store** (DataStore-backed). On subsequent evaluations, the stored assignment is returned regardless of whether rollout percentages changed. This prevents the re-bucketing problem: changing percentages from 50/50 to 70/30 without sticky assignments corrupts experiment data because users switch variants mid-experiment.

!!! warning "Edge Case"
    This is why LaunchDarkly separates "feature flags" (re-bucketing OK) from "experiments" (sticky required). For cross-device stickiness, local bucketing isn't enough -- the assignment must be stored server-side and included in the flag configuration response.

### Metrics Collection

Exposure tracking must be accurate without draining battery. The key is **deduplication** -- only track the first exposure per flag per session:

```kotlin
class ExposureTracker {
    private val seen = ConcurrentHashMap<String, Boolean>()

    fun trackIfNew(flagKey: String, variation: Int, reason: EvalReason): Boolean {
        val key = "$flagKey:$variation"
        return seen.putIfAbsent(key, true) == null
    }
}
```

The event batching pipeline: evaluations and custom events flow into an in-memory buffer, which flushes every 30 seconds or at 100 events. Events are gzip-compressed before sending. Failed flushes move to a disk-backed retry queue. On app background, an immediate flush is triggered via lifecycle callback. The disk queue caps at 10MB with FIFO eviction.

```mermaid
graph LR
    EVAL["Flag<br/>Evaluation"] -->|exposure| BUFFER["In-Memory<br/>Buffer"]
    APP["App Code"] -->|custom event| BUFFER
    BUFFER -->|"30s timer or<br/>100 events"| COMPRESS["Gzip<br/>Compress"]
    COMPRESS --> SEND["HTTP POST<br/>/events"]
    SEND -->|"failure"| RETRY["Retry Queue<br/>(disk-backed)"]
    RETRY -->|"next flush"| SEND
```

!!! tip "Pro Tip"
    GrowthBook only tracks the **first exposure** per user per experiment *lifetime*, not per session. This dramatically reduces event volume. The tradeoff: you cannot detect users who saw a different variant due to a config error. LaunchDarkly tracks per-session for more granular debugging.

### Gradual Rollout & Kill Switch

Rollout percentage is a property of the flag configuration. When the operator changes it from 5% to 25%, the new config flows through SSE to the SDK. The evaluation engine uses the same consistent hash, so bucket ranges **expand, never shuffle** -- users in the 5% bucket are always in the 25% bucket too. This guarantees monotonic rollout: no user who had the feature loses it as the percentage increases.

```
Rollout progression (same hash, expanding ranges):
 1%: [0.00, 0.01) → users with hash < 0.01 get the feature
 5%: [0.00, 0.05) → same users + more
25%: [0.00, 0.25) → same users + more
```

The **kill switch** is the most critical feature. When a feature causes crashes, it must be disabled instantly:

```mermaid
sequenceDiagram
    participant ENG as On-Call Engineer
    participant DASH as Dashboard
    participant API as Flag API
    participant SSE as SSE Hub
    participant SDK as Mobile SDK (millions)
    participant CACHE as Local Cache

    ENG->>DASH: Toggle "new_checkout" OFF
    DASH->>API: PATCH /flags/new_checkout {on: false}
    API->>SSE: Broadcast flag-update
    SSE->>SDK: event: flag-update {on: false}
    SDK->>CACHE: Update in-memory + disk
    Note over SDK: Next evaluation returns<br/>offVariation (default)
    Note over SDK: Total propagation: 1-5 seconds
```

Even with SSE, some devices won't receive the update immediately (offline, backgrounded, SSE disconnected). Mitigation layers: SSE push (1-5s for connected devices), aggressive polling fallback, push notification (10-30s), force fetch on app foreground, and as a last resort, a circuit breaker in the backend API that rejects requests from clients using the bad feature regardless of their local flag state.

!!! warning "Edge Case"
    A kill switch only works as fast as the **slowest propagation path**. If a device is offline, it keeps evaluating the old flag value until it connects. For truly critical kill switches (data loss, security vulnerability), the backend circuit breaker is essential.

### Flag Lifecycle & Stale Detection

Flags accumulate over time and become tech debt. Release flags live days to weeks (remove after 100% rollout). Experiment flags live weeks to months (remove after conclusion). Ops flags like `maintenance_mode` are permanent. The SDK helps detect stale flags by tracking which flags are actually evaluated and reporting unused flags to the backend periodically.

!!! tip "Pro Tip"
    Spotify's FIDO platform enforces flag expiration dates. If a flag isn't cleaned up by its expiry, it shows warnings and can auto-disable. This prevents the "5,000 flags and nobody knows which ones are still needed" problem. Integrate a lint rule that detects flag keys in code and cross-references with the flag service to find references to deleted flags.

### Performance Budget

**Startup impact:** SDK init adds ~10ms (create objects, register lifecycle). Disk cache read is async and non-blocking. First evaluation adds ~0.1ms from bootstrap or default. Total: ~10ms on an 800ms cold start budget.

**Evaluation latency:** Override check (0.005ms) + cache lookup (0.005ms) + rule evaluation for 5 rules (0.05ms) + MurmurHash3 (0.01ms) + exposure tracking (0.01ms) = **~0.08ms total**, well under the 1ms budget.

**Memory:** Flag configs for 500 flags (~1.2MB) + evaluation engine (~50KB) + event buffer (~100KB) + SSE buffer (~50KB) = **~1.4MB total**, under the 2MB budget.

!!! warning "Edge Case"
    Watch out for flag configurations with large JSON variation values (e.g., a flag that returns a 50KB UI configuration object). 100 such flags balloons memory to 5MB just for variations. Set a max variation size limit in the SDK and log warnings when flags exceed it.

### Integration Patterns

**Compose** integration uses `CompositionLocal` for DI and `rememberFlag` for reactive rendering -- flag changes trigger automatic recomposition.

**Hilt/Koin** integration evaluates flags at DI graph construction time, but there's a timing trap: if the flag changes later via SSE, the DI-provided singleton still uses the old value. Use `Provider<T>` for re-evaluation, or only use DI-level flags for toggles that don't change during a session. Compose integration handles this naturally since `rememberFlag` is reactive.

**Testing** uses a `FakeFlagClient` with predetermined values injected into tests:

```kotlin
class FakeFlagClient(
    private val overrides: Map<String, Any> = emptyMap()
) : FlagClient {
    override fun boolVariation(key: String, default: Boolean): Boolean {
        return overrides[key] as? Boolean ?: default
    }
}
```

---

## Scalability, Reliability & Edge Cases

| Scenario | Decision | Reasoning |
|----------|----------|-----------|
| **Flag evaluated before SDK init completes** | Return default value, track "evaluation before ready" event | Never block the caller. The app must render something immediately. |
| **SSE disconnects** | Exponential backoff (1s, 2s, 4s... max 5min), fall back to polling | Aggressive reconnection wastes battery. Backoff + polling covers all cases. |
| **Circular flag dependencies** | Disallow in rule validation on server | Evaluation engine is not re-entrant. Circular rules cause infinite loops. |
| **User context changes mid-session (login)** | Call `identify()`, re-fetch flags, re-evaluate all observed flags | Emit new values on all observed Flows so Compose recomposes. |
| **Flag key typo** | Return default value, log "unknown flag" warning | Never crash on a missing flag, but log loudly for developer debugging. |
| **Extremely large payload (>5MB)** | Stream and parse incrementally, cap per-flag size server-side | 1MB max per flag variation, 10MB max total payload. Protect against OOM. |
| **Clock skew** | Use server timestamps for event ordering, client timestamps for display | Future-dated exposure events confuse analytics pipelines. |
| **Multiple SDK instances** | Enforce singleton per SDK key, throw on duplicate init | Multiple instances duplicate connections, caches, and events. |
| **Variation type mismatch** | Return default value, log type error | `boolVariation` on a string flag must not crash -- degrade gracefully. |
| **App update changes context** | Re-evaluate all flags on app version change | A flag targeting `app_version >= 5.0` must fire after update from 4.x. |
| **Concurrent multi-thread evaluation** | Lock-free reads via `ConcurrentHashMap`, atomic swap for writes | Flags are read-heavy, write-rare. Lock-free reads prevent contention. |
| **GDPR opt-out** | Disable exposure events, still evaluate flags with anonymous context | Evaluation is functional, exposure events are tracking. Separate the concerns. |

---

## Wrap Up

- **Local evaluation with server-synced rules** -- sub-millisecond latency, full offline support, no server dependency for flag checks.
- **Two-tier caching (memory + disk) with bootstrap** -- fast evaluation, offline resilience, and first-launch coverage.
- **SSE with polling fallback** -- near-instant propagation without WebSocket complexity.
- **MurmurHash3 deterministic bucketing + sticky bucket store** -- consistent experiment assignment without re-bucketing.
- **Batched, compressed, disk-backed event delivery** -- battery-efficient with 99.9% eventual delivery.

**What I'd improve with more time:** mutual TLS for SDK auth, local evaluation audit log (circular buffer, last 1,000 evaluations), flag dependency graph to detect conflicts, predictive pre-fetch based on navigation patterns, edge evaluation via CDN for sub-50ms first load, typed flag code generation from the flag schema at build time (Spotify FIDO does this), and holdout groups for measuring cumulative experiment impact.

---

## References

- [LaunchDarkly SDK Architecture](https://docs.launchdarkly.com/sdk/concepts/client-side-server-side) -- Client-side vs server-side evaluation model
- [LaunchDarkly Evaluation Engine (Open Source)](https://github.com/launchdarkly/go-server-sdk-evaluation) -- Reference implementation of flag evaluation logic
- [GrowthBook SDK Architecture](https://docs.growthbook.io/lib/architecture) -- Client-side evaluation with targeting rules
- [Firebase Remote Config](https://firebase.google.com/docs/remote-config) -- Google's server-evaluated config with local caching
- [Spotify FIDO](https://engineering.atspotify.com/2020/10/29/how-we-improved-developer-productivity-for-our-devops-teams/) -- Flag expiration and code generation
- [Uber Experimentation Platform (Morpheus)](https://www.uber.com/en-US/blog/xp/) -- Large-scale experimentation with mobile SDKs
- [Netflix Feature Flagging](https://netflixtechblog.com/its-all-a-about-testing-the-netflix-experimentation-platform-4e1ca458c15f) -- Genesis config and experimentation culture
- [Martin Fowler -- Feature Toggles](https://martinfowler.com/articles/feature-toggles.html) -- Canonical taxonomy of toggle types
- [MurmurHash3 Specification](https://github.com/aappleby/smhasher/wiki/MurmurHash3) -- Hash function used for experiment bucketing
