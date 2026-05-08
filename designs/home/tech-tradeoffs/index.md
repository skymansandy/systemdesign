# Technology Trade-offs Reference

A principal engineer's quick-reference for the technology choices that come up in every mobile system design interview. Each section gives you the **comparison table**, the **decision framework**, and the **one-liner** you'd say in an interview.

---

## 1. Network Protocol

| Protocol | Payload | Direction | Latency | Mobile Fit | Use When |
|---|---|---|---|---|---|
| REST | JSON (text) | Request-Response | Medium | Excellent | CRUD, simple reads/writes |
| GraphQL | JSON (text) | Request-Response | Medium | Good | Complex data needs, reducing over-fetch |
| gRPC | Protobuf (binary) | Bi-streaming | Low | Good (mobile-to-server) | Internal services, streaming, low bandwidth |
| WebSocket | Any | Bidirectional | Very Low | Medium (battery cost) | Chat, real-time collaboration |
| SSE | Text | Server → Client | Low | Good | Notifications, feed updates, live scores |

### Decision Framework

```
Is the data mostly read-only with infrequent updates?
  → REST with caching

Does the client need different fields in different screens?
  → GraphQL to avoid over-fetching

Is bidirectional real-time needed?
  → WebSocket (foreground only, push + sync for background)

Is it server-to-client only real-time?
  → SSE (simpler, auto-reconnect, HTTP/2 compatible)

Is bandwidth critical (emerging markets, large payloads)?
  → gRPC with Protobuf (50-80% smaller than JSON)
```

!!! tip "Pro Tip"
    "I'd use REST for the CRUD operations and layer WebSocket on top for real-time delivery while the app is foregrounded. When backgrounded, I'd tear down the WebSocket and rely on FCM for urgent updates plus delta sync on next foreground."

---

## 2. Local Database

| Database | Language | KMP Support | Reactive | Migrations | Best For |
|---|---|---|---|---|---|
| Room | Kotlin/Java | No (Android only) | Flow built-in | Auto + manual | Android-only apps |
| SQLDelight | Kotlin | Yes (full KMP) | Flow via extensions | Verified migrations | KMP shared data layer |
| Realm | Kotlin/Swift | Partial | Live objects | Auto schema sync | Rapid prototyping, simple models |
| DataStore | Kotlin | Yes (KMP) | Flow built-in | N/A (key-value) | Preferences, small config |

### Decision Framework

| Constraint | Choose |
|---|---|
| Android-only, team knows Room | **Room** |
| KMP / shared business logic | **SQLDelight** |
| Simple key-value, <1MB | **DataStore (Proto or Preferences)** |
| Complex object graphs, real-time sync | **Realm** (evaluate lock-in risk) |

### Performance Characteristics

| Operation | Room | SQLDelight | Realm |
|---|---|---|---|
| Bulk insert (10K rows) | ~200ms | ~180ms | ~150ms |
| Indexed query | <1ms | <1ms | <1ms |
| Complex JOIN | Good (SQL) | Good (SQL) | Poor (no JOINs) |
| Full-text search | FTS4/5 support | FTS4/5 support | Built-in text search |
| DB size on device | Standard SQLite | Standard SQLite | Larger (object store) |

!!! warning "Edge Case"
    Room's `@Relation` annotation loads all related entities eagerly — on a large dataset, this causes OOM. Use explicit JOINs with `@RawQuery` or `@RewriteQueriesToDropUnusedColumns` for complex queries.

---

## 3. Dependency Injection

| Framework | KMP Support | Compile-time Safe | Learning Curve | Performance |
|---|---|---|---|---|
| Hilt | No (Android only) | Yes (Dagger under the hood) | Steep | Excellent (no reflection) |
| Koin | Yes (full KMP) | No (runtime resolution) | Low | Good (slight startup cost) |
| kotlin-inject | Yes (full KMP) | Yes (KSP-based) | Medium | Excellent |
| Manual DI | Yes | Yes | Low | Best |

### Decision Framework

| Constraint | Choose |
|---|---|
| Android-only, large team | **Hilt** (industry standard, enforces structure) |
| KMP, moderate app size | **Koin** (pragmatic, fast setup) |
| KMP, want compile safety | **kotlin-inject** (Dagger-like, KMP-native) |
| SDK / library development | **Manual DI** (no framework dependency on consumers) |

!!! tip "Pro Tip"
    In interviews, don't spend time on DI framework choice unless asked. Mention it once: "I'd use Koin for KMP compatibility — it gives us runtime DI across platforms." Then move on to the interesting parts.

---

## 4. Networking Library

| Library | KMP Support | HTTP/2 | WebSocket | Serialization | Notes |
|---|---|---|---|---|---|
| Ktor | Yes (full KMP) | Yes | Yes | kotlinx.serialization | First choice for KMP |
| Retrofit | No (Android/JVM) | Via OkHttp | Via OkHttp | Gson, Moshi, kotlinx | Android industry standard |
| OkHttp | No (Android/JVM) | Yes | Yes | N/A (raw) | Low-level, interceptors |
| URLSession | iOS only | Yes | Yes (iOS 13+) | Codable | Native iOS |

### Decision Framework

| Constraint | Choose |
|---|---|
| KMP shared networking | **Ktor** |
| Android-only, existing codebase | **Retrofit + OkHttp** |
| Need low-level control | **OkHttp** directly |
| iOS-specific optimizations | **URLSession** |

---

## 5. Image Loading

| Library | KMP Support | Compose | Disk Cache | Memory Cache | Transformations |
|---|---|---|---|---|---|
| Coil | Yes (KMP 3.x) | Native | Yes | Yes (LRU) | Built-in + custom |
| Glide | No (Android) | Via integration | Yes | Yes (LRU + active) | Built-in + custom |
| Kingfisher | No (iOS/Swift) | SwiftUI support | Yes | Yes | Built-in |

### Decision Framework

| Constraint | Choose |
|---|---|
| KMP + Compose Multiplatform | **Coil 3.x** |
| Android-only, legacy Views | **Glide** (better lifecycle handling) |
| Android-only, Compose | **Coil** |
| iOS/Swift | **Kingfisher** |

!!! tip "Pro Tip"
    The library choice is less interesting than the **caching policy**. In interviews, say "I'd use Coil, but the design decision is whether to cache aggressively (better offline, more disk usage) or conservatively (less storage, more network). For a feed app, I'd cache thumbnails aggressively and full-res images with an LRU eviction at 100 MB."

---

## 6. State Management

| Pattern | Unidirectional | Testability | Complexity | Best For |
|---|---|---|---|---|
| MVVM + StateFlow | Yes (with discipline) | Good | Low-Medium | Most apps |
| MVI (Redux-like) | Strictly yes | Excellent | Medium | Complex UI state, time-travel debug |
| Compose State + ViewModel | Yes | Good | Low | Simple screens |
| Decompose (KMP) | Yes | Excellent | Medium | KMP navigation + state |

### MVI vs MVVM Decision

| Factor | Prefer MVVM | Prefer MVI |
|---|---|---|
| Team size | Small, experienced | Large, needs consistency |
| UI complexity | Simple forms, lists | Complex interactions, multi-step flows |
| State predictability | Less critical | Must reproduce exact states |
| Debugging needs | Standard | Need state replay / time-travel |

```kotlin
// MVI — Single state, single event stream
data class FeedState(
    val items: List<FeedItem> = emptyList(),
    val isLoading: Boolean = false,
    val error: AppError? = null
)

sealed class FeedIntent {
    object LoadFeed : FeedIntent()
    object Refresh : FeedIntent()
    data class LikePost(val postId: String) : FeedIntent()
}
```

!!! note "Interview Context"
    Most interviewers don't care whether you say MVVM or MVI — they care that you have a **unidirectional data flow** and can explain how state changes propagate. Pick one, name it, and move on.

---

## 7. Serialization

| Library | KMP Support | Speed | Code Gen | Schema Evolution |
|---|---|---|---|---|
| kotlinx.serialization | Yes (full KMP) | Fast | KSP plugin | Manual migration |
| Moshi | No (Android/JVM) | Fast | KSP or reflection | Lenient by default |
| Gson | No (Android/JVM) | Medium | Reflection | Lenient |
| Protobuf | Yes (full KMP) | Fastest | protoc compiler | Built-in versioning |

### Decision Framework

| Constraint | Choose |
|---|---|
| KMP project | **kotlinx.serialization** |
| Android-only, existing Moshi | Keep **Moshi** |
| Need schema versioning + compact payloads | **Protobuf** |
| Quick and dirty | **Gson** (but migrate away) |

---

## 8. Background Work

| API | Platform | Min Interval | Survives Reboot | Constraints | Best For |
|---|---|---|---|---|---|
| WorkManager | Android | 15 min | Yes | Network, battery, storage | Sync, upload, cleanup |
| BGTaskScheduler | iOS | System-decided | Yes | Limited execution time | Sync, maintenance |
| AlarmManager | Android | Exact | Yes (with permission) | None | Alarms, reminders |
| Foreground Service | Android | N/A (runs now) | No | Must show notification | Music, navigation, uploads |

### Decision Matrix

```
Need guaranteed execution? → WorkManager / BGTaskScheduler
Need exact timing? → AlarmManager (Android), UNNotification (iOS)
Need long-running now? → Foreground Service (Android)
Need periodic sync? → WorkManager 15min + FCM for urgent
Can it wait for good conditions? → WorkManager with constraints
```

!!! warning "Edge Case"
    Android OEMs (Samsung, Xiaomi, Huawei) aggressively kill background work beyond AOSP limits. In interviews, mention this: "WorkManager handles most OEM quirks, but on some devices we'd need FCM high-priority messages as a fallback trigger for critical syncs."

---

## 9. Push Notification Services

| Service | Platform | Delivery Guarantee | Silent Push | Priority Levels |
|---|---|---|---|---|
| FCM | Android (+ iOS, Web) | At-least-once | Yes (`data` message) | Normal, High |
| APNs | iOS | At-least-once | Yes (content-available) | 1-10 (mapped to delivery) |
| Unified Push | Android (FOSS) | Varies | Yes | Single |

### Key Design Decisions

| Decision | Options | Recommendation |
|---|---|---|
| Notification vs. Data message | `notification` (system tray) vs. `data` (app handles) | **Data messages** for full control |
| Token refresh | On each app start vs. periodic | On start + FCM `onNewToken` callback |
| Topic vs. device targeting | Subscribe to topics vs. server stores tokens | Topics for broadcast, tokens for targeted |
| Deduplication | Client-side idempotency key | Always — push is at-least-once |

---

## 10. Testing Strategy

| Test Type | Scope | Speed | Reliability | Use For |
|---|---|---|---|---|
| Unit (JUnit/Kotest) | Function/class | ~ms | Very high | Business logic, ViewModels, UseCases |
| Integration | Module boundary | ~100ms | High | Repository + DB, API contract |
| UI (Compose Test) | Screen | ~1s | Medium | Critical user journeys |
| E2E (Maestro/Appium) | Full app | ~10s | Low-Medium | Smoke tests, release validation |
| Screenshot (Paparazzi) | Visual | ~100ms | High | Design system, regression |

### The Testing Diamond (Mobile)

```
         ┌──────┐
         │  E2E │  ← Few: smoke tests for critical paths
        ┌┴──────┴┐
        │   UI   │  ← Some: key screens, navigation
       ┌┴────────┴┐
       │Integration│  ← More: DB, API, Repository
      ┌┴──────────┴┐
      │    Unit     │  ← Most: ViewModels, UseCases, Mappers
      └─────────────┘
```

!!! tip "Pro Tip"
    In interviews, mention testing briefly: "I'd unit test the ViewModel and UseCase, integration test the Repository with an in-memory database, and have a few Compose UI tests for the critical path. For this design, the most important test is verifying the offline queue processes operations in order after reconnection."
