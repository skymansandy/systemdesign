# Feature Flag / Experimentation Platform -- Mobile SDK Architecture

This document covers the **mobile SDK design** of a feature flag and experimentation platform (LaunchDarkly / Firebase Remote Config / GrowthBook / Spotify FIDO). The focus is on the client-side engine: how flags are evaluated locally, how experiment assignments stay sticky, how the SDK bootstraps without blocking app startup, and how you handle the kill-switch scenario where a bad feature must be disabled instantly across millions of devices.

!!! note "Why This Topic for Staff-Level Interviews"
    Feature flagging is deceptively deep. On the surface it looks like "if/else with a remote config." In reality it touches **local evaluation engines, consistent hashing for experiment bucketing, real-time flag propagation, offline resilience, metrics pipelines, and SDK design that must not regress app startup time.** Interviewers use this topic to test whether you can design a platform that other engineers consume -- an SDK is an API with very different constraints than a backend service.

**What makes the mobile SDK uniquely hard:**

- The SDK runs inside someone else's app -- you cannot crash, block the main thread, or consume excessive memory.
- Flags must evaluate in **microseconds**, not milliseconds. Any latency directly impacts the host app's UI rendering.
- The device goes offline, and the app still needs flag values. You cannot fail open or closed blindly.
- Experiment assignment must be **deterministic and sticky** -- the user must see the same variant across app restarts, reinstalls, and even devices.
- You must collect exposure events for statistical analysis without becoming a battery-draining telemetry hog.

Every design decision in this document is driven by those constraints.

---

## Problem & Design Scope

### Clarifying Questions

Before drawing a single box, ask the interviewer these questions to bound the problem:

1. **Client-side or server-side evaluation?** Client-side means the SDK has all targeting rules locally and evaluates flags without network calls. Server-side means every evaluation is an API call. This is the single biggest architectural decision.
2. **What is the flag volume?** 50 flags vs. 5,000 flags per project changes the payload size, memory strategy, and sync approach.
3. **Do we need experimentation (A/B testing) or just feature flags?** Experimentation adds sticky bucketing, exposure tracking, and statistical rigor.
4. **What targeting complexity is required?** Simple on/off per user vs. percentage rollouts vs. multi-variate targeting rules with segments.
5. **Real-time updates or polling?** Does a flag change need to propagate in seconds (SSE/WebSocket) or are periodic polls (minutes) acceptable?
6. **Offline requirement?** Must the SDK return meaningful values when the device has no connectivity?
7. **Multi-platform?** Android-only or KMP shared SDK with iOS? This drives the shared-code boundary.
8. **What is the host app's startup budget?** If the app must be interactive in 800ms, the SDK cannot block for 200ms fetching flags.
9. **Kill switch latency target?** How fast must a bad flag be disabled globally?
10. **Privacy constraints?** Can user attributes (email, country, plan) be sent to the flag service, or must evaluation happen entirely on-device?

### Functional Requirements

| Requirement | Details |
|-------------|---------|
| **Evaluate feature flags** | Return typed values (Boolean, String, Int, JSON) for a given flag key |
| **Percentage rollouts** | Gradually roll out a feature to N% of users deterministically |
| **User targeting** | Evaluate flags based on user attributes (country, plan, app version, custom) |
| **A/B test assignment** | Assign users to experiment variants with sticky bucketing |
| **Real-time updates** | Receive flag changes without app restart |
| **Exposure tracking** | Record when a user is exposed to a flag/experiment for analytics |
| **Offline evaluation** | Return cached or default flag values when offline |
| **Debug overlay** | Developer-facing UI to inspect flag values, overrides, and targeting |
| **Local overrides** | Developers can force flag values during development/QA |

### Non-Functional Requirements

| Requirement | Target | Why It Matters |
|-------------|--------|----------------|
| **Evaluation latency** | < 1ms per flag | Flag checks happen in render paths; blocking = jank |
| **SDK init time** | < 50ms added to app startup | Host apps have strict cold-start budgets |
| **Memory overhead** | < 2 MB for 500 flags | SDK runs alongside the host app's own memory needs |
| **Update propagation** | < 30s for real-time, < 5min for polling | Kill switches must propagate fast |
| **Offline resilience** | 100% evaluation success with cached data | Never throw or return null when offline |
| **Exposure event delivery** | 99.9% eventual delivery | Missing events corrupt experiment analysis |
| **Battery impact** | Negligible (< 0.1% per hour) | Real-time connections and telemetry can drain battery |
| **Binary size** | < 500 KB added to APK | Large SDKs get rejected by app teams |

### Mobile vs Backend Constraints

| Concern | Backend SDK | Mobile SDK |
|---------|------------|------------|
| **Evaluation model** | Can call server per evaluation (low latency in datacenter) | Must evaluate locally -- network round-trip is unacceptable |
| **State persistence** | In-memory (process is long-lived) | Must persist to disk -- process is killed frequently |
| **Updates** | SSE/polling from same datacenter | SSE over unreliable mobile network with reconnection logic |
| **Memory** | Generous (server has GBs) | Constrained (SDK shares memory with host app) |
| **Startup** | Service boots once, stays up | App cold-starts often; SDK init must be near-zero |
| **Telemetry** | Flush immediately | Batch, compress, flush on schedule to save battery |
| **Threading** | Thread pools | Must never block main thread; use Dispatchers.IO |

---

## UI Sketch

### Flag Evaluation Flow (Developer Mental Model)

```
┌─────────────────────────────────────────────────────────────────┐
│                        Host Application                         │
│                                                                 │
│   if (flagClient.boolVariation("new_checkout", false)) {        │
│       // Show new checkout flow                                 │
│   } else {                                                      │
│       // Show legacy checkout flow                              │
│   }                                                             │
│                                                                 │
│   ┌───────────────────────────────────────────────────────┐     │
│   │              Feature Flag SDK                         │     │
│   │                                                       │     │
│   │  ┌─────────┐   ┌──────────┐   ┌──────────────────┐  │     │
│   │  │ In-Mem  │──▶│ Eval     │──▶│ Return variant + │  │     │
│   │  │ Cache   │   │ Engine   │   │ track exposure   │  │     │
│   │  └────▲────┘   └──────────┘   └──────────────────┘  │     │
│   │       │                                               │     │
│   │  ┌────┴────┐   ┌──────────┐   ┌──────────────────┐  │     │
│   │  │ Disk    │   │ SSE /    │   │ Event            │  │     │
│   │  │ Cache   │   │ Polling  │   │ Batcher          │  │     │
│   │  └─────────┘   └──────────┘   └──────────────────┘  │     │
│   └───────────────────────────────────────────────────────┘     │
└─────────────────────────────────────────────────────────────────┘
```

### Debug Overlay Screen

```
┌─────────────────────────┐
│  Feature Flag Debug     │
├─────────────────────────┤
│ User: usr_abc123        │
│ Env: production         │
│ Flags loaded: 127       │
│ Last sync: 12s ago      │
│ Connection: SSE active  │
│─────────────────────────│
│ 🔍 Filter flags...     │
│─────────────────────────│
│ ☑ new_checkout    true  │
│   Source: TARGETING     │
│   Rule: country = US    │
│   Override: none        │
│                         │
│ ☐ dark_mode       false │
│   Source: DEFAULT        │
│   Rule: rollout 30%     │
│   Override: none        │
│                         │
│ ☑ exp_pricing   variant │
│   Source: EXPERIMENT    │
│   Bucket: 0.7234        │
│   Override: [FORCED]    │
│─────────────────────────│
│ [Force Override] [Reset]│
│ [Copy State] [Export]   │
└─────────────────────────┘
```

### Key SDK States

| State | Behavior | Developer Experience |
|-------|----------|---------------------|
| **Initializing** | Loading from disk cache, hasn't fetched yet | Flags evaluate using cached or default values |
| **Ready** | Initial fetch complete, in-memory cache populated | Flags evaluate using fresh server values |
| **Streaming** | SSE connection active, receiving real-time updates | Flag changes reflected within seconds |
| **Stale** | Cached data is older than staleness threshold | SDK still evaluates, logs warning, triggers background refresh |
| **Offline** | No network connectivity | Flags evaluate from disk cache; events queued for later |
| **Error** | Server returned error or invalid payload | Falls back to cached values; never crashes host app |

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

### API Design Approach Comparison

| Approach | Evaluation | Latency | Offline | Complexity | Used By |
|----------|-----------|---------|---------|------------|---------|
| **Local evaluation (rules on device)** | SDK evaluates targeting rules locally | ~0.1ms | Full support | Higher SDK complexity | LaunchDarkly, GrowthBook |
| **Server evaluation (API per flag)** | Server evaluates, returns value | 50-200ms | No support | Simple SDK | Firebase Remote Config (fetch model) |
| **Hybrid (server eval + local cache)** | Server evaluates, SDK caches results | 0.1ms cached, 50ms miss | Cached values only | Medium | Optimizely |

### Decision: Local Evaluation with Server-Synced Rules

**Why local evaluation:**

- **Latency**: Sub-millisecond evaluation. Flag checks happen in render code paths (Compose recomposition, RecyclerView binding). Even 5ms per flag is unacceptable if you check 20 flags during screen load.
- **Offline**: Full evaluation works without network. The SDK has all targeting rules cached locally.
- **Privacy**: User attributes never leave the device for evaluation. Only exposure events are sent back.
- **Reliability**: No dependency on server availability for flag evaluation. If the flag service goes down, the app works fine with cached rules.

**Why not server evaluation:**

- Network dependency kills offline support entirely.
- Latency makes it unusable in UI rendering paths.
- Every flag check becomes a potential failure point.

**Why not hybrid:**

- Server evaluation + caching gives you the worst of both worlds: you still need network for fresh evaluations, and cached values can go stale without the SDK knowing the targeting rules changed.

!!! tip "Pro Tip"
    LaunchDarkly's mobile SDK downloads the entire flag configuration for a specific user context (pre-evaluated for the user but with rules for local re-evaluation). GrowthBook downloads the raw targeting rules and evaluates entirely on-device. Both are valid -- the tradeoff is payload size vs. evaluation complexity.

---

## API Endpoint Design & Additional Considerations

### Flag Configuration API

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

### Event Tracking API

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
      "type": "exposure",
      "flagKey": "exp_pricing",
      "variation": 2,
      "value": "variant_b",
      "reason": "EXPERIMENT",
      "userKey": "usr_abc123",
      "timestamp": 1715100001000
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

### Polling vs Streaming Tradeoffs

| Aspect | Polling | SSE (Server-Sent Events) | WebSocket |
|--------|---------|--------------------------|-----------|
| **Latency** | Minutes (poll interval) | Seconds | Seconds |
| **Battery** | Good (request + sleep) | Moderate (persistent connection) | Poor (bidirectional keep-alive) |
| **Reconnection** | Built-in (next poll) | Auto-reconnect with Last-Event-ID | Manual reconnect logic |
| **Server cost** | Stateless, CDN-cacheable | Stateful connection per client | Stateful + more complex |
| **Mobile-friendly** | Excellent | Good with backoff | Poor for this use case |

**Decision: SSE primary + polling fallback.**

SSE gives near-real-time updates with simpler reconnection than WebSocket (the browser/client handles reconnect and `Last-Event-ID` natively). We do not need bidirectional communication -- the SDK only receives flag updates, it does not send data over the stream. Polling kicks in when SSE is unavailable (corporate proxies, restrictive networks).

!!! warning "Edge Case"
    Some corporate networks and mobile carriers strip SSE connections after 30-60 seconds. The SDK must detect a stale SSE connection (no heartbeat received within expected interval) and fall back to polling automatically. LaunchDarkly's mobile SDK implements exactly this pattern.

### Pagination & Efficiency

- **Initial fetch**: Download all flags for the environment in a single request. For 500 flags with targeting rules, expect 50-200 KB gzipped. This is small enough for a single request.
- **Delta updates**: SSE sends only changed flags. No need for pagination on the stream.
- **Event batching**: Collect exposure events in memory, flush every 30 seconds or when the batch reaches 100 events. Compress with gzip before sending.
- **ETag caching**: Use `If-None-Match` on polling requests. If flags haven't changed, the server returns `304 Not Modified` with zero body.

---

## High-Level Architecture

### SDK Component Architecture

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

### Component Responsibilities

| Component | Responsibility | Threading |
|-----------|---------------|-----------|
| **FlagClient (Public API)** | Single entry point. Delegates to evaluation engine. Thread-safe. | Any thread (caller's) |
| **Evaluation Engine** | Evaluates targeting rules against user context. Pure function, no I/O. | Any thread (CPU-bound, ~0.1ms) |
| **In-Memory Cache** | Holds current flag configurations in a `ConcurrentHashMap`. | Any thread (lock-free reads) |
| **Persistent Store** | Writes flag configs to disk (DataStore / SQLite). Reads on cold start. | `Dispatchers.IO` |
| **Sync Manager** | Manages SSE connection + polling fallback. Updates in-memory cache. | `Dispatchers.IO` |
| **Event Processor** | Batches exposure events, compresses, flushes on schedule. | Background coroutine |
| **Context Manager** | Holds current user context. Notifies sync manager on `identify()`. | Any thread |
| **Override Manager** | Stores developer-forced flag values (debug only). | Any thread |

### KMP Alignment

| Layer | Shared (commonMain) | Platform-Specific |
|-------|--------------------|--------------------|
| **Evaluation Engine** | Full targeting rule engine, consistent hashing | None |
| **In-Memory Cache** | `ConcurrentHashMap` abstraction, cache logic | Atomics implementation |
| **Persistent Store** | Store interface, serialization | `DataStore` (Android), `NSUserDefaults` (iOS) |
| **Sync Manager** | SSE parsing, polling logic, retry/backoff | Ktor engine (OkHttp / Darwin) |
| **Event Processor** | Batching logic, event serialization | Flush triggers (lifecycle-aware) |
| **Public API** | Full `FlagClient` API | Compose integration (`rememberFlag`) |

!!! tip "Pro Tip"
    The evaluation engine is the crown jewel of the shared layer. It is pure computation with zero platform dependencies -- just rules, context, and math. This is the ideal candidate for 100% shared KMP code. LaunchDarkly open-sources their evaluation engine, and it is completely platform-agnostic.

---

## Data Flow for Basic Scenarios

### Evaluating a Flag (Hot Path)

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
    participant SYNC as Sync Manager

    Note over SYNC: Network unavailable
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

## Design Deep Dive

### Client-Side Flag Evaluation

The evaluation engine is the core of the SDK. It takes a flag configuration and a user context and returns a variation value.

```kotlin
// Pure function -- no I/O, no side effects
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

**Condition matching operators:**

| Operator | Example | Description |
|----------|---------|-------------|
| `eq` | `country eq "US"` | Exact string match |
| `in` | `country in ["US", "CA"]` | Set membership |
| `gt`, `lt`, `gte`, `lte` | `app_version gte "4.0"` | Semantic version comparison |
| `contains` | `email contains "@company.com"` | Substring match |
| `matches` | `userId matches "^test-.*"` | Regex match |
| `segment` | `segment in "beta_users"` | Segment membership lookup |
| `before`, `after` | `created_at after "2025-01-01"` | Date comparison |

!!! warning "Edge Case"
    Semantic version comparison is tricky. `"4.10.0"` must be greater than `"4.9.0"`, but naive string comparison says otherwise. The evaluation engine must parse versions into `(major, minor, patch)` tuples. LaunchDarkly has a dedicated `semver` operator for this reason.

### Flag Storage and Caching

The SDK maintains a **two-tier cache**: in-memory for evaluation speed, disk for offline resilience.

```mermaid
graph LR
    EVAL["Evaluation<br/>Engine"] -->|"~0.01ms"| MEM["In-Memory Cache<br/>(ConcurrentHashMap)"]
    MEM -->|"~1ms"| DISK["Disk Cache<br/>(DataStore/Files)"]
    DISK -->|"~50ms"| NET["Network<br/>(SSE/Polling)"]
    
    style MEM fill:#2d5,stroke:#333,color:#000
    style DISK fill:#da2,stroke:#333,color:#000
    style NET fill:#d55,stroke:#333,color:#000
```

**Cache lifecycle:**

| Event | In-Memory | Disk | Behavior |
|-------|-----------|------|----------|
| **SDK init** | Empty | May have data | Load disk → memory (async, non-blocking) |
| **First fetch** | Populated | Written | Both caches now fresh |
| **SSE update** | Updated immediately | Written async | In-memory always has latest |
| **App restart** | Empty | Has data | Disk → memory on init; stale but functional |
| **Cache eviction** | N/A (small enough to keep all) | TTL-based | Disk cache expires after 30 days of no update |

**Bootstrap values:**

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

The bootstrap file is generated at build time from the flag service and bundled into the APK. It is **never fresher than the last build** but guarantees the SDK always has values on first launch.

!!! tip "Pro Tip"
    Netflix bundles a default flag configuration in their app binary and calls it the "genesis config." This ensures the app can render a complete experience even if the flag service is unreachable on first launch. The config is updated on every app release build.

### Targeting Rules Engine

The rules engine evaluates flag configurations against user context. Rules are ordered and the **first matching rule wins**.

```kotlin
data class TargetingRule(
    val conditions: List<Condition>,  // AND logic within a rule
    val variation: Int?,               // Fixed variation
    val rollout: Rollout?,            // Percentage-based variation
)

data class Condition(
    val attribute: String,    // "country", "plan", "segment", etc.
    val operator: Operator,   // EQ, IN, GT, CONTAINS, etc.
    val values: List<String>, // Comparison values
    val negate: Boolean,      // Invert the result
)

// Rules are AND within a rule, OR across rules (first match wins)
// Rule 1: country IN [US, CA] AND plan EQ premium → variation 1
// Rule 2: segment IN beta_users → variation 1
// Fallthrough: 30% rollout → variation 1, else variation 0
```

**Segments** are reusable groups of users that can be referenced across multiple flags:

```kotlin
data class Segment(
    val key: String,
    val included: Set<String>,    // Explicitly included user keys
    val excluded: Set<String>,    // Explicitly excluded user keys
    val rules: List<SegmentRule>, // Dynamic rules (same condition format)
)
```

!!! note "How Uber Targets"
    Uber's feature flag platform supports "geo-fenced" targeting where flags can be evaluated based on the user's current city. This is implemented as a custom attribute (`city = "San Francisco"`) updated in the user context when location changes. The targeting engine itself doesn't know about geography -- it just matches attribute values.

### Experiment Assignment (Sticky Bucketing)

Experiment assignment must be **deterministic** and **sticky**. The same user must always get the same variant, even across app restarts, reinstalls, and devices.

**Deterministic assignment via consistent hashing:**

```kotlin
fun assignBucket(flagSalt: String, userKey: String): Double {
    // Hash produces a value in [0.0, 1.0)
    val input = "$flagSalt.$userKey"
    val hash = murmurHash3(input.encodeToByteArray())
    return (hash.toUInt().toDouble()) / UInt.MAX_VALUE.toDouble()
}

fun resolveVariation(
    context: UserContext,
    flagSalt: String,
    rollout: Rollout,
): Int {
    val bucket = assignBucket(flagSalt, context.key)
    var cumulative = 0.0
    for ((variation, weight) in rollout.weightedVariations) {
        cumulative += weight / 100_000.0  // Weights in millipercents for precision
        if (bucket < cumulative) return variation
    }
    return rollout.weightedVariations.last().variation
}
```

**Why MurmurHash3:**

| Hash Function | Speed | Distribution | Why/Why Not |
|---------------|-------|-------------|-------------|
| **MurmurHash3** | Very fast | Excellent uniformity | Industry standard for bucketing. LaunchDarkly, GrowthBook use it. |
| **SHA-256** | Slow | Perfect uniformity | Cryptographic -- overkill. 10-50x slower than MurmurHash. |
| **CRC32** | Fast | Poor uniformity | Collisions create assignment bias. Not suitable. |
| **xxHash** | Fastest | Excellent | Good alternative but MurmurHash has wider ecosystem support. |

**Sticky bucketing for experiments:**

When a user is assigned to an experiment variant, the assignment is persisted locally:

```kotlin
class StickyBucketStore(private val dataStore: DataStore<Preferences>) {
    suspend fun getAssignment(flagKey: String, userKey: String): Int? {
        val key = "$flagKey:$userKey"
        return dataStore.data.first()[stringPreferencesKey(key)]?.toIntOrNull()
    }

    suspend fun saveAssignment(flagKey: String, userKey: String, variation: Int) {
        val key = "$flagKey:$userKey"
        dataStore.edit { it[stringPreferencesKey(key)] = variation.toString() }
    }
}
```

!!! warning "Edge Case"
    **Re-bucketing problem**: If you change the rollout percentages of an experiment (e.g., from 50/50 to 70/30), existing users get re-bucketed if you don't have sticky assignments. This corrupts your experiment data because users switch variants mid-experiment. The sticky bucket store prevents this by always returning the first assignment. This is why LaunchDarkly separates "feature flags" (re-bucketing OK) from "experiments" (sticky required).

**Cross-device stickiness:**

Local sticky bucketing only works on one device. For cross-device consistency, the assignment must be stored server-side and fetched as part of the flag configuration. The server pre-evaluates experiment assignments for the user and includes them in the response.

### Metrics Collection

Experiment analysis requires accurate exposure tracking and conversion event collection.

**Exposure event deduplication:**

```kotlin
class ExposureTracker {
    // Only track first exposure per flag per session
    private val seen = ConcurrentHashMap<String, Boolean>()

    fun trackIfNew(flagKey: String, variation: Int, reason: EvalReason): Boolean {
        val key = "$flagKey:$variation"
        return seen.putIfAbsent(key, true) == null
    }
}
```

**Event batching pipeline:**

```mermaid
graph LR
    EVAL["Flag<br/>Evaluation"] -->|exposure| BUFFER["In-Memory<br/>Buffer"]
    APP["App Code"] -->|custom event| BUFFER
    BUFFER -->|"30s timer or<br/>100 events"| COMPRESS["Gzip<br/>Compress"]
    COMPRESS --> SEND["HTTP POST<br/>/events"]
    SEND -->|"failure"| RETRY["Retry Queue<br/>(disk-backed)"]
    RETRY -->|"next flush"| SEND
```

**Event delivery guarantees:**

| Scenario | Behavior |
|----------|----------|
| **Flush succeeds** | Events removed from buffer |
| **Flush fails (network)** | Events moved to disk-backed retry queue |
| **App killed before flush** | Events lost (acceptable -- exposure events are high volume) |
| **App backgrounded** | Trigger immediate flush via lifecycle callback |
| **Disk queue grows too large** | Drop oldest events (FIFO eviction at 10 MB cap) |

!!! tip "Pro Tip"
    GrowthBook's approach is worth noting: they only track the **first exposure** per user per experiment, ever. Not per session, per experiment lifetime. This dramatically reduces event volume. The tradeoff is you cannot detect users who were exposed but then saw a different variant due to a config error. LaunchDarkly tracks per-session for more granular debugging.

### Gradual Rollout Strategies

| Strategy | Description | When to Use |
|----------|-------------|-------------|
| **Canary** | Release to internal employees first (targeting rule: `email contains "@company.com"`) | Every feature, always start here |
| **Percentage ramp** | 1% → 5% → 25% → 50% → 100% over days | Standard rollout for user-facing features |
| **Ring-based** | Ring 0 (dogfood) → Ring 1 (beta) → Ring 2 (early adopters) → Ring 3 (all) | Microsoft/Spotify approach for high-risk changes |
| **Geo-based** | Roll out to one country/region first | Features with regulatory or localization risk |
| **Platform-based** | Android first, then iOS (or vice versa) | When one platform has better instrumentation |

**Implementing percentage ramp with the SDK:**

The rollout percentage is a property of the flag configuration. When the backend operator changes it from 5% to 25%, the new configuration flows through SSE to the SDK. The evaluation engine uses the same consistent hash for the user, so users in the 5% bucket are **always** in the 25% bucket too (bucket ranges expand, never shuffle).

```
Rollout progression (same hash, expanding ranges):
 1%: [0.00, 0.01) → users with hash < 0.01 get the feature
 5%: [0.00, 0.05) → same users + more
25%: [0.00, 0.25) → same users + more
```

This guarantees monotonic rollout: no user who had the feature loses it as the percentage increases.

### Kill Switch and Emergency Rollback

The kill switch is the most critical feature of any flag platform. When a feature causes crashes or degraded UX, it must be disabled instantly.

**Kill switch flow:**

```mermaid
sequenceDiagram
    participant ENG as On-Call Engineer
    participant DASH as Dashboard
    participant API as Flag API
    participant CDN as CDN/Edge
    participant SSE as SSE Hub
    participant SDK as Mobile SDK (millions)
    participant CACHE as Local Cache

    ENG->>DASH: Toggle "new_checkout" OFF
    DASH->>API: PATCH /flags/new_checkout {on: false}
    API->>CDN: Invalidate cached config
    API->>SSE: Broadcast flag-update
    SSE->>SDK: event: flag-update {on: false}
    SDK->>CACHE: Update in-memory + disk
    Note over SDK: Next evaluation returns<br/>offVariation (default)
    Note over SDK: Total propagation: 1-5 seconds
```

**Cached flag invalidation problem:**

Even with SSE, some devices won't receive the update immediately (offline, SSE disconnected, backgrounded). Mitigation strategies:

| Approach | Latency | Coverage |
|----------|---------|----------|
| **SSE push** | 1-5s | Online + connected devices |
| **Aggressive polling** | Next poll interval | Online devices with SSE failure |
| **Push notification** | 10-30s | Devices with push token registered |
| **App open check** | Next app foreground | All devices, eventually |
| **Force fetch on resume** | Immediate on foreground | All devices that open the app |

!!! warning "Edge Case"
    A kill switch only works as fast as the **slowest propagation path**. If a device is offline, it will keep evaluating the old flag value until it connects. For truly critical kill switches (feature causes data loss or security vulnerability), consider a **circuit breaker in the backend API** that rejects requests from clients using the bad feature, regardless of their local flag state.

### Flag Lifecycle Management

Flags accumulate over time and become tech debt. The SDK must support flag lifecycle management.

**Flag types:**

| Type | Lifespan | Example | Cleanup Strategy |
|------|----------|---------|-----------------|
| **Release flag** | Days to weeks | `new_checkout` | Remove after 100% rollout confirmed |
| **Experiment flag** | Weeks to months | `exp_pricing_v2` | Remove after experiment concludes |
| **Ops flag** | Permanent | `maintenance_mode` | Keep indefinitely, mark as permanent |
| **Permission flag** | Permanent | `enable_admin_panel` | Keep indefinitely, entitlement-based |

**Stale flag detection:**

The SDK can help detect stale flags by tracking which flags are actually evaluated:

```kotlin
class FlagUsageTracker {
    private val evaluatedFlags = ConcurrentHashMap<String, Long>()

    fun markEvaluated(flagKey: String) {
        evaluatedFlags[flagKey] = System.currentTimeMillis()
    }

    fun getUnusedFlags(allFlags: Set<String>): Set<String> {
        return allFlags - evaluatedFlags.keys
    }

    // Report to backend periodically
    fun generateUsageReport(): FlagUsageReport {
        return FlagUsageReport(
            evaluatedFlags = evaluatedFlags.toMap(),
            reportTimestamp = System.currentTimeMillis(),
        )
    }
}
```

!!! tip "Pro Tip"
    Spotify's FIDO platform enforces flag expiration dates. When a flag is created, the owner sets an expiry. If the flag isn't cleaned up by then, it starts showing warnings in the dashboard and can auto-disable. This prevents the "5,000 flags and nobody knows which ones are still needed" problem. Integrate a lint rule that detects flag keys in code and cross-references with the flag service to find references to deleted flags.

### Performance Impact

The SDK must be invisible from a performance perspective. Here is the budget:

**Startup impact:**

```
Cold start without SDK:     800ms
SDK init (sync):            +10ms   (create objects, register lifecycle)
Disk cache read (async):    +0ms    (non-blocking, happens in background)
First evaluation:           +0.1ms  (from bootstrap or default)
─────────────────────────────────
Total added:                ~10ms
```

**Evaluation latency breakdown:**

| Step | Time | Notes |
|------|------|-------|
| Override check | 0.005ms | HashMap lookup |
| Cache lookup | 0.005ms | ConcurrentHashMap.get() |
| Rule evaluation (5 rules) | 0.05ms | Condition matching, no I/O |
| Hash computation | 0.01ms | MurmurHash3 for rollout |
| Exposure tracking | 0.01ms | ConcurrentHashMap.putIfAbsent() |
| **Total** | **~0.08ms** | Well under 1ms budget |

**Memory overhead:**

| Component | Memory | Notes |
|-----------|--------|-------|
| Flag configs (500 flags) | ~1.2 MB | JSON-like structures in memory |
| Evaluation engine | ~50 KB | Code + constant data |
| Event buffer (100 events) | ~100 KB | Before flush |
| SSE connection buffer | ~50 KB | Read buffer |
| **Total** | **~1.4 MB** | Under 2 MB budget |

!!! warning "Edge Case"
    Watch out for flag configurations with large JSON variation values (e.g., a flag that returns a full UI configuration object of 50 KB). If you have 100 such flags, your memory footprint balloons to 5 MB just for variations. Set a max variation size limit in the SDK and log warnings when flags exceed it.

### Integration Patterns

#### Compose Conditional Rendering

```kotlin
// CompositionLocal for dependency injection
val LocalFlagClient = staticCompositionLocalOf<FlagClient> {
    error("FlagClient not provided")
}

// Provide at app root
@Composable
fun App() {
    CompositionLocalProvider(LocalFlagClient provides flagClient) {
        AppNavigation()
    }
}

// Flag-driven feature toggle
@Composable
fun HomeScreen() {
    val showPromo by rememberFlag("home_promo_banner", defaultValue = false)
    val layoutVariant by rememberFlag("home_layout", defaultValue = "classic")

    Column {
        if (showPromo) {
            PromoBanner()
        }
        when (layoutVariant) {
            "classic" -> ClassicHomeLayout()
            "modern" -> ModernHomeLayout()
            else -> ClassicHomeLayout() // default fallback
        }
    }
}
```

#### Dependency Injection with Flags

```kotlin
// Flag-aware module in Hilt
@Module
@InstallIn(SingletonComponent::class)
object FeatureModule {
    @Provides
    fun providePaymentProcessor(
        flagClient: FlagClient,
    ): PaymentProcessor {
        return if (flagClient.boolVariation("use_stripe_v2", false)) {
            StripeV2PaymentProcessor()
        } else {
            StripeLegacyPaymentProcessor()
        }
    }
}
```

!!! warning "Edge Case"
    **DI + flags timing problem**: If you evaluate a flag at DI graph construction time (app startup), the flag uses the bootstrap/cached value. If the flag changes later via SSE, the DI-provided singleton still uses the old value. Two options: (1) Use `Provider<PaymentProcessor>` so it re-evaluates each time, or (2) only use DI-level flag evaluation for flags that don't change during a session (like major feature toggles). Compose integration handles this naturally since `rememberFlag` is reactive.

#### Feature Flag Testing

```kotlin
// In tests, use a fake client with predetermined values
class FakeFlagClient(
    private val overrides: Map<String, Any> = emptyMap()
) : FlagClient {
    override fun boolVariation(key: String, default: Boolean): Boolean {
        return overrides[key] as? Boolean ?: default
    }
    // ... other variation methods
}

@Test
fun `new checkout flow shown when flag enabled`() {
    val flagClient = FakeFlagClient(mapOf("new_checkout" to true))
    // Inject into test, assert new checkout UI is rendered
}
```

---

## Edge Cases & Decisions

| Scenario | Decision | Reasoning |
|----------|----------|-----------|
| **Flag evaluated before SDK init completes** | Return default value, track "evaluation before ready" event | Never block the caller. The app must render something immediately. Firebase Remote Config does the same with `getDefaultConfig()`. |
| **SSE disconnects during flag update** | Retry with exponential backoff (1s, 2s, 4s, 8s... max 5min), fall back to polling | Aggressive reconnection wastes battery. Backoff + polling fallback covers all scenarios. |
| **Two flags reference each other in targeting** | Disallow circular dependencies in rule validation on server | The evaluation engine is not re-entrant. Circular rules cause infinite loops. Validate at flag creation time. |
| **User context changes mid-session (login)** | Call `identify()`, re-fetch flags, re-evaluate all observed flags | Flag values may change. Emit new values on all observed Flows so Compose recomposes. |
| **Flag key typo (key doesn't exist)** | Return default value, log warning with "unknown flag" | Never crash on a missing flag. But log loudly so the developer catches the typo in development. |
| **Extremely large flag payload (>5 MB)** | Stream and parse incrementally, cap individual flag size server-side | Protect against OOM. Set a 1 MB max per flag variation and 10 MB max total payload. |
| **Clock skew between client and server** | Use server-provided timestamps for event ordering, client timestamps for display only | Exposure events with future timestamps confuse analytics pipelines. |
| **Multiple SDK instances in same process** | Enforce singleton per SDK key, throw on duplicate init | Multiple instances duplicate connections, caches, and event buffers. Waste resources and cause race conditions. |
| **Flag variation type mismatch** | Return default value, log type error | Developer calls `boolVariation` on a string flag. Never crash; degrade gracefully. |
| **App update changes user context attributes** | Re-evaluate all flags on app version change | A flag targeting `app_version >= 5.0` must fire when the user updates from 4.x to 5.x. |
| **Concurrent flag evaluation from multiple threads** | Lock-free reads on in-memory cache using ConcurrentHashMap | Flags are read-heavy, write-rare. Lock-free reads prevent contention. Writes (from SSE updates) use atomic swap. |
| **User opts out of tracking (GDPR)** | Disable exposure event collection, still evaluate flags using anonymous context | Flag evaluation is functional (not tracking). Exposure events are tracking. Separate the two concerns. |

---

## Wrap Up

### Key Design Decisions

| Decision | Choice | Key Reason |
|----------|--------|------------|
| **Evaluation model** | Client-side, local evaluation | Sub-millisecond latency, full offline support |
| **Caching** | Two-tier (memory + disk) with bootstrap | Fast evaluation + offline resilience + first-launch coverage |
| **Real-time updates** | SSE with polling fallback | Near-instant propagation without WebSocket complexity |
| **Experiment assignment** | MurmurHash3 deterministic + sticky bucket store | Consistent assignment + no re-bucketing during experiments |
| **Event delivery** | Batched, compressed, disk-backed retry queue | Battery-efficient, eventual delivery guaranteed |
| **SDK init** | Non-blocking, async cache load | Zero impact on app startup time |
| **Compose integration** | `rememberFlag` with Flow-backed reactivity | Flag changes trigger automatic recomposition |
| **KMP strategy** | Evaluation engine + batching logic shared, persistence + network platform-specific | Maximize shared code where it matters most |

### Improvements with More Time

- **Mutual TLS** for SDK-to-server authentication instead of bearer tokens.
- **Local evaluation audit log** -- record every evaluation for on-device debugging (circular buffer, last 1,000).
- **Flag dependency graph** -- model dependencies between flags to detect conflicts (flag A enables a flow that flag B disables).
- **Predictive pre-fetch** -- use app navigation patterns to pre-warm flag evaluations for the next likely screen.
- **Edge evaluation** -- deploy flag configs to CDN edge nodes so the initial fetch comes from the nearest POP, reducing first-load latency to <50ms globally.
- **Typed flag generation** -- code-generate type-safe flag accessors from the flag service schema at build time, eliminating string key typos entirely (Spotify FIDO does this).
- **Holdout groups** -- exclude a percentage of users from all experiments as a global control group for measuring long-term cumulative impact.

---

## References

- [LaunchDarkly SDK Architecture](https://docs.launchdarkly.com/sdk/concepts/client-side-server-side) -- Client-side vs server-side evaluation model explained
- [LaunchDarkly Evaluation Engine (Open Source)](https://github.com/launchdarkly/go-server-sdk-evaluation) -- Reference implementation of flag evaluation logic
- [GrowthBook SDK Architecture](https://docs.growthbook.io/lib/architecture) -- Client-side evaluation with targeting rules
- [Firebase Remote Config](https://firebase.google.com/docs/remote-config) -- Google's server-evaluated config with local caching
- [Spotify FIDO (Feature Flag Platform)](https://engineering.atspotify.com/2020/10/29/how-we-improved-developer-productivity-for-our-devops-teams/) -- Internal platform with flag expiration and code generation
- [Uber Experimentation Platform (Morpheus)](https://www.uber.com/en-US/blog/xp/) -- Large-scale experimentation with mobile SDKs
- [Netflix Feature Flagging](https://netflixtechblog.com/its-all-a-about-testing-the-netflix-experimentation-platform-4e1ca458c15f) -- Genesis config and always-be-experimenting culture
- [Martin Fowler -- Feature Toggles](https://martinfowler.com/articles/feature-toggles.html) -- Canonical taxonomy of toggle types
- [Consistent Hashing for Experimentation](https://research.google/pubs/pub36356/) -- Google's paper on deterministic bucketing
- [MurmurHash3 Specification](https://github.com/aappleby/smhasher/wiki/MurmurHash3) -- Hash function used for experiment bucketing
