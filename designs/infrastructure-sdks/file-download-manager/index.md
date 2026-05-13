# File Download Manager

Designing a mobile file download manager -- the kind you see in Chrome, Telegram, or podcast apps -- is a deceptively hard problem. A single dropped connection at 95% of a 500 MB file must not restart the download. The OS aggressively kills background processes, so your download must survive process death. Concurrent downloads compete for bandwidth, memory, and I/O. And the user expects accurate progress, pause/resume, and the ability to queue dozens of files without the app freezing. Every design decision here is driven by those constraints.

---

## Scoping the Problem

The first thing I'd want to clarify is file sizes -- PDFs at 5 MB vs. podcast episodes at 200 MB vs. app bundles at 2 GB -- because this drives chunking strategy and storage pre-allocation. Next, does the server support HTTP Range requests? Without `Accept-Ranges: bytes`, resumable downloads are impossible, and that changes our entire fallback strategy.

I'd also ask about concurrency (1 download? 3? Unlimited?), whether background download is required (which means surviving Doze mode and process death), and whether WiFi-only mode is needed for metered connections. Integrity verification (checksums after download), priority support (user-initiated vs. prefetch), and multi-source CDN downloads all meaningfully change the architecture.

**Core scope:** Enqueue downloads by URL, pause/resume from byte offset using Range headers, cancel with cleanup, background execution that survives process death, real-time progress with speed and ETA, priority queue with configurable concurrency, automatic retry with exponential backoff, WiFi-only mode, persistent notification, and optional checksum validation.

**Key non-functional priorities:**

- **Resume reliability** -- a 500 MB file at 90% must resume from byte offset, never restart. All progress persisted to DB, not just in-memory.
- **Battery efficiency** -- < 2% battery/hour during background download. Excessive wake locks get the app killed or uninstalled.
- **Memory footprint** -- < 20 MB heap for the download engine. Buffer sizes must be bounded; no loading entire files into memory.
- **Process death resilience** -- DB query for pending downloads on startup in < 200ms, not a full re-scan.
- **Progress update rate** -- every 500ms to UI (conflated Flow), every 2s to notification (Android rate-limits rebuilds).

The mobile-specific constraints are what make this hard. Android Doze mode defers network access, foreground services require visible notifications, and WorkManager has minimum 15-minute intervals. WiFi-to-cellular handoff drops TCP connections. Storage may be ejected. And Android kills background processes aggressively -- any in-memory state (buffer position, speed calculation) is lost.

---

## UI Sketch

```
┌──────────────────────┐  ┌──────────────────────┐  ┌──────────────────────┐
│   Download Manager    │  │   Download Detail     │  │   Settings           │
├──────────────────────┤  ├──────────────────────┤  ├──────────────────────┤
│ Active (2)            │  │ ← podcast-ep-142.mp3  │  │                      │
│                       │  │──────────────────────│  │ Max concurrent: [3]   │
│ podcast-ep-142.mp3    │  │ URL: https://cdn...   │  │                      │
│ ████████░░░░ 67%      │  │ Size: 84.2 MB         │  │ WiFi-only:    [ON]   │
│ 12.4 MB/s  ETA 2:31   │  │ Downloaded: 56.4 MB   │  │                      │
│       [⏸ Pause]       │  │ Speed: 12.4 MB/s      │  │ Auto-retry:   [ON]   │
│                       │  │ ETA: 2:31             │  │ Max retries:  [5]    │
│ design-spec.pdf       │  │ Status: Downloading   │  │                      │
│ ██████████░░ 89%      │  │ Started: 10:32 AM     │  │ Download path:       │
│ 3.1 MB/s   ETA 0:04   │  │ Chunks: 4/4 active    │  │ /Downloads/          │
│       [⏸ Pause]       │  │                       │  │                      │
│───────────────────────│  │ ████████░░░░ 67%      │  │ Bandwidth limit:     │
│ Queued (3)            │  │                       │  │ [Unlimited ▼]        │
│                       │  │ [⏸ Pause] [✕ Cancel]  │  │                      │
│ video-tutorial.mp4    │  │                       │  │ Chunk size:          │
│ 1.2 GB  ⏳ Waiting     │  │ Integrity: SHA-256    │  │ [5 MB ▼]             │
│                       │  │ ✓ Will verify         │  │                      │
│ firmware-v2.1.bin     │  └──────────────────────┘  └──────────────────────┘
│ 340 MB  ⏳ Waiting     │
│                       │
│───────────────────────│
│ Completed (12)        │
│                       │
│ report-q4.pdf     ✓   │
│ 2.4 MB   Completed    │
│                       │
│ [+ Add URL]           │
└──────────────────────┘

┌──────────────────────────────────────────────┐
│          Notification (Foreground Service)     │
├──────────────────────────────────────────────┤
│ ▼ Downloads (2 active)                        │
│                                               │
│   podcast-ep-142.mp3  ████████░░ 67%  ⏸      │
│   design-spec.pdf     █████████░ 89%  ⏸      │
│                                               │
│   [Pause All]                                 │
└──────────────────────────────────────────────┘
```

```mermaid
flowchart LR
    LIST[Download List] -->|Tap item| DETAIL[Download Detail]
    LIST -->|FAB / Add URL| ADD[Add Download]
    ADD -->|Submit| LIST
    DETAIL -->|Back| LIST
    LIST -->|Gear icon| SETTINGS[Settings]
    SETTINGS -->|Back| LIST
    DETAIL -->|Completed + Tap| OPEN[Open File / Share]
```

---

## API Design

### Protocol Choice: HTTP with Range Requests

A download manager's primary interaction with the server is fetching bytes. I'd use **HTTP/1.1 or HTTP/2 with Range requests** -- every CDN, every cloud storage provider, and every file server supports this. The Range header is the foundation of resumable downloads.

| Protocol | Fit | Why / Why Not |
|----------|-----|---------------|
| **HTTP/1.1 + Range** | Best fit | Universal server support, resumable via `Range` header, well-understood caching (ETag, Last-Modified) |
| **HTTP/2** | Excellent | Multiplexing allows multiple chunk streams over one connection, header compression reduces overhead |
| **gRPC streaming** | Overkill | Adds protobuf overhead for raw byte transfer. No CDN ecosystem support for file serving |
| **WebSocket** | Wrong tool | Designed for bidirectional messaging, not bulk data transfer. No built-in resume semantics |

!!! tip "Pro Tip"
    In an interview, explicitly mention that you'd send a `HEAD` request first to check `Accept-Ranges: bytes` and `Content-Length`. If the server doesn't support ranges, you fall back to non-resumable download -- and you should tell the user upfront.

### Key HTTP Headers

| Header | Direction | Purpose |
|--------|-----------|---------|
| `Range: bytes=1048576-` | Request | Resume from byte 1 MB |
| `Accept-Ranges: bytes` | Response (HEAD) | Server confirms range support |
| `Content-Range: bytes 1048576-5242879/5242880` | Response | Server confirms the range served |
| `Content-Length` | Response | Total file size (HEAD) or chunk size (ranged GET) |
| `ETag` | Response | File version identifier -- detect if file changed since pause |
| `If-Range` | Request | "Give me the range only if ETag still matches; otherwise send the whole file" |

### Chunked vs. Single-Stream Download

I'd default to **single stream** and switch to **parallel chunks for files > 50 MB** when the server supports ranges. Single stream is simple with no merge step. Multi-chunk parallel (2-4 connections) can be 2-4x faster on high-latency links but adds merge complexity. This matches Chrome and Telegram's behavior.

### Client-Side API (Download Engine Interface)

This is the internal API exposed by the download engine to the UI layer -- the server is just an HTTP file host.

```kotlin
interface DownloadManager {
    suspend fun enqueue(request: DownloadRequest): DownloadId
    suspend fun pause(id: DownloadId)
    suspend fun resume(id: DownloadId)
    suspend fun cancel(id: DownloadId)
    fun observe(id: DownloadId): Flow<DownloadState>
    fun observeAll(): Flow<List<DownloadState>>
    suspend fun retry(id: DownloadId)
    suspend fun setPriority(id: DownloadId, priority: Priority)
}

data class DownloadRequest(
    val url: String,
    val fileName: String,
    val destinationDir: String,
    val headers: Map<String, String> = emptyMap(),
    val priority: Priority = Priority.NORMAL,
    val wifiOnly: Boolean = false,
    val expectedSize: Long? = null,
    val checksumSha256: String? = null,
)

data class DownloadState(
    val id: DownloadId,
    val url: String,
    val fileName: String,
    val status: Status,
    val totalBytes: Long,
    val downloadedBytes: Long,
    val speedBytesPerSec: Long,
    val etaSeconds: Long,
    val error: DownloadError?,
    val createdAt: Instant,
    val completedAt: Instant?,
)

enum class Status {
    QUEUED, DOWNLOADING, PAUSED, COMPLETED, FAILED, VERIFYING, CANCELLED
}

enum class Priority { LOW, NORMAL, HIGH, URGENT }
```

### Server Interaction Sequence

```
1. HEAD {url}
   → 200 OK
   → Accept-Ranges: bytes
   → Content-Length: 52428800
   → ETag: "abc123"

2. GET {url}
   Range: bytes=0-13107199
   If-Range: "abc123"
   → 206 Partial Content
   → Content-Range: bytes 0-13107199/52428800

3. (On resume after pause at byte 13107200)
   GET {url}
   Range: bytes=13107200-
   If-Range: "abc123"
   → 206 Partial Content  (ETag matches, continue)
   or
   → 200 OK  (ETag changed, file modified -- restart download)
```

!!! warning "Edge Case"
    If the server responds with `200 OK` instead of `206 Partial Content` to a Range request, the server does not support ranges or the file changed. Detect this by checking the status code and restart the download, informing the user that progress was lost.

---

## Architecture

### Component Diagram

```mermaid
graph TB
    subgraph UI Layer
        DL_LIST[Download List Screen]
        DL_DETAIL[Detail Screen]
        NOTIF[Notification Controller]
    end

    subgraph Domain Layer
        DM[DownloadManager]
        SCHED[Download Scheduler]
        SM[State Machine]
    end

    subgraph Data Layer
        ENGINE[Download Engine]
        REPO[Download Repository]
        DB[(SQLite / Room)]
        FS[File System]
        NET[HTTP Client]
    end

    subgraph Platform Layer
        WM[WorkManager]
        FGS[Foreground Service]
        CM[ConnectivityMonitor]
    end

    DL_LIST --> DM
    DL_DETAIL --> DM
    DM --> SCHED
    DM --> SM
    SCHED --> ENGINE
    DM --> REPO
    REPO --> DB
    ENGINE --> NET
    ENGINE --> FS
    ENGINE --> REPO
    FGS --> ENGINE
    WM --> FGS
    CM --> SCHED
    NOTIF --> DM
```

**DownloadManager** is the public API facade -- all operations flow through it. The **Scheduler** manages concurrency limits, priority queue, and WiFi-only constraints, decoupled from the engine so policy changes don't touch download logic. The **State Machine** enforces valid transitions. The **Download Engine** performs HTTP requests and writes bytes to disk -- stateless per-download, with all persistent state in DB. The **Repository** is the single source of truth, exposing Flows for reactive UI.

On the platform side: **Foreground Service** keeps the process alive during active downloads. **WorkManager** guarantees resumption after process death or reboot. **ConnectivityMonitor** uses `NetworkCallback` for real-time network changes.

!!! tip "Pro Tip: KMP Alignment"
    The download engine (Ktor + byte writing) is the most shareable KMP component -- Ktor supports Range headers on all platforms. Domain layer, data layer interfaces, and DB schema (SQLDelight) all live in `commonMain`. The platform boundary is just background scheduling (WorkManager vs. BGProcessingTask), file system paths, and notification APIs.

---

## Data Flow for Key Scenarios

### Starting a New Download

```mermaid
sequenceDiagram
    participant UI as UI Layer
    participant DM as DownloadManager
    participant SCH as Scheduler
    participant SM as StateMachine
    participant DB as Repository/DB
    participant ENG as Engine
    participant NET as HTTP Client
    participant FS as File System

    UI->>DM: enqueue(request)
    DM->>DB: insert(download, status=QUEUED)
    DM->>SM: transition(QUEUED)
    DM->>SCH: schedule(downloadId)

    SCH->>SCH: Check concurrency limit
    alt Slot available
        SCH->>SM: transition(DOWNLOADING)
        SCH->>DB: update(status=DOWNLOADING)
        SCH->>ENG: start(downloadId)
        ENG->>NET: HEAD url
        NET-->>ENG: 200 (size, ETag, Accept-Ranges)
        ENG->>DB: update(totalBytes, etag)
        ENG->>FS: allocate temp file
        ENG->>NET: GET url (Range: bytes=0-)
        loop Every buffer read (8KB)
            NET-->>ENG: bytes
            ENG->>FS: append to temp file
            ENG->>DB: update(downloadedBytes) [batched]
            ENG-->>UI: progress via Flow
        end
        ENG->>FS: rename temp → final
        ENG->>SM: transition(COMPLETED)
        ENG->>DB: update(status=COMPLETED)
    else Queue full
        SCH-->>UI: status remains QUEUED
    end
```

### Resuming After Network Interruption

```mermaid
sequenceDiagram
    participant CM as ConnectivityMonitor
    participant SCH as Scheduler
    participant ENG as Engine
    participant DB as Repository/DB
    participant NET as HTTP Client
    participant FS as File System

    Note over ENG: Network drops during download
    ENG->>ENG: IOException caught
    ENG->>DB: persist(downloadedBytes=currentOffset)
    ENG->>SCH: reportFailure(downloadId, NETWORK_ERROR)
    SCH->>DB: update(status=PAUSED, retryCount++)

    Note over CM: Network restored
    CM->>SCH: onNetworkAvailable(capabilities)
    SCH->>DB: query paused downloads
    SCH->>ENG: resume(downloadId)

    ENG->>DB: read(downloadedBytes, etag)
    ENG->>NET: GET url, Range: bytes={offset}-, If-Range: {etag}
    alt 206 Partial Content
        Note over ENG: ETag matches, resume from offset
        ENG->>FS: open temp file, seek to offset
        ENG->>FS: append remaining bytes
    else 200 OK (ETag changed)
        Note over ENG: File changed on server, restart
        ENG->>FS: delete temp file
        ENG->>DB: reset(downloadedBytes=0)
        ENG->>FS: create new temp file
        ENG->>FS: write from beginning
    end
```

### Parallel Chunk Download

```mermaid
sequenceDiagram
    participant ENG as Engine
    participant NET as HTTP Client
    participant FS as File System
    participant DB as Repository/DB

    ENG->>NET: HEAD url
    NET-->>ENG: Content-Length: 200MB, Accept-Ranges: bytes

    Note over ENG: Split into 4 chunks of 50MB each

    par Chunk 1 (bytes 0-49999999)
        ENG->>NET: GET Range: bytes=0-49999999
        NET-->>ENG: stream bytes
        ENG->>FS: write to chunk_0.tmp
    and Chunk 2 (bytes 50000000-99999999)
        ENG->>NET: GET Range: bytes=50000000-99999999
        NET-->>ENG: stream bytes
        ENG->>FS: write to chunk_1.tmp
    and Chunk 3 (bytes 100000000-149999999)
        ENG->>NET: GET Range: bytes=100000000-149999999
        NET-->>ENG: stream bytes
        ENG->>FS: write to chunk_2.tmp
    and Chunk 4 (bytes 150000000-199999999)
        ENG->>NET: GET Range: bytes=150000000-199999999
        NET-->>ENG: stream bytes
        ENG->>FS: write to chunk_3.tmp
    end

    ENG->>FS: merge chunk_0..3.tmp → final.tmp
    ENG->>FS: rename final.tmp → output file
    ENG->>FS: delete chunk files
    ENG->>DB: update(status=COMPLETED)
```

### User-Initiated Pause and Resume

```mermaid
sequenceDiagram
    participant UI as UI
    participant DM as DownloadManager
    participant SM as StateMachine
    participant ENG as Engine
    participant DB as Repository/DB

    UI->>DM: pause(downloadId)
    DM->>SM: transition(DOWNLOADING → PAUSED)
    DM->>ENG: cancel coroutine job
    ENG->>ENG: flush buffer to disk
    ENG->>DB: persist(downloadedBytes, status=PAUSED)
    DM-->>UI: state = PAUSED

    Note over UI: User taps Resume later

    UI->>DM: resume(downloadId)
    DM->>SM: transition(PAUSED → DOWNLOADING)
    DM->>DB: read(downloadedBytes, etag)
    DM->>ENG: start(downloadId, fromOffset)
    ENG->>ENG: Range request from saved offset
    DM-->>UI: state = DOWNLOADING
```

---

## Design Deep Dive

### Resumable Downloads (HTTP Range Headers)

The core mechanism that makes a download manager useful. HEAD to get metadata, persist it, record byte offset on interruption, resume with `Range: bytes={offset}-` and `If-Range: {etag}`, then validate: `206` means continue, `200` means file changed so restart.

```kotlin
suspend fun downloadWithResume(
    client: HttpClient,
    url: String,
    outputFile: File,
    startOffset: Long,
    etag: String?,
    onProgress: (bytesRead: Long, totalBytes: Long) -> Unit,
): DownloadResult {
    val response = client.get(url) {
        if (startOffset > 0) {
            header(HttpHeaders.Range, "bytes=$startOffset-")
            etag?.let { header(HttpHeaders.IfRange, it) }
        }
    }

    val isResumed = response.status == HttpStatusCode.PartialContent
    val actualOffset = if (isResumed) startOffset else 0L
    val totalBytes = if (isResumed) {
        parseContentRange(response)?.totalSize ?: response.contentLength()
    } else {
        response.contentLength()
    } ?: -1L

    val mode = if (isResumed) FileWriteMode.APPEND else FileWriteMode.OVERWRITE
    val channel = response.bodyAsChannel()
    val buffer = ByteArray(BUFFER_SIZE) // 8 KB

    outputFile.outputStream(mode).use { out ->
        var bytesWritten = actualOffset
        while (!channel.isClosedForRead) {
            val read = channel.readAvailable(buffer)
            if (read > 0) {
                out.write(buffer, 0, read)
                bytesWritten += read
                onProgress(bytesWritten, totalBytes)
            }
        }
    }

    return DownloadResult.Success(actualOffset, totalBytes)
}
```

!!! warning "Edge Case"
    Some servers return `Accept-Ranges: bytes` in the HEAD response but silently ignore the `Range` header in GET. Always verify: if you send `Range: bytes=1000-` but get a `200` with `Content-Length` equal to the full file size, the server did not honor the range. Restart the download.

### Download State Machine

A formal state machine prevents illegal transitions and makes the system predictable.

```mermaid
stateDiagram-v2
    [*] --> QUEUED : enqueue()
    QUEUED --> DOWNLOADING : slot available
    QUEUED --> CANCELLED : cancel()
    DOWNLOADING --> PAUSED : pause() / network lost
    DOWNLOADING --> COMPLETED : all bytes received
    DOWNLOADING --> FAILED : error + max retries exceeded
    DOWNLOADING --> VERIFYING : checksum configured
    DOWNLOADING --> CANCELLED : cancel()
    PAUSED --> DOWNLOADING : resume() / network restored
    PAUSED --> CANCELLED : cancel()
    FAILED --> QUEUED : retry()
    FAILED --> CANCELLED : cancel()
    VERIFYING --> COMPLETED : checksum matches
    VERIFYING --> FAILED : checksum mismatch
    COMPLETED --> [*]
    CANCELLED --> [*]
```

```kotlin
class DownloadStateMachine {
    private val validTransitions: Map<Status, Set<Status>> = mapOf(
        Status.QUEUED to setOf(Status.DOWNLOADING, Status.CANCELLED),
        Status.DOWNLOADING to setOf(
            Status.PAUSED, Status.COMPLETED, Status.FAILED,
            Status.VERIFYING, Status.CANCELLED
        ),
        Status.PAUSED to setOf(Status.DOWNLOADING, Status.CANCELLED),
        Status.FAILED to setOf(Status.QUEUED, Status.CANCELLED),
        Status.VERIFYING to setOf(Status.COMPLETED, Status.FAILED),
    )

    fun transition(current: Status, target: Status): Result<Status> {
        val allowed = validTransitions[current] ?: return Result.failure(
            IllegalStateException("No transitions from terminal state $current")
        )
        return if (target in allowed) {
            Result.success(target)
        } else {
            Result.failure(
                IllegalStateException("Invalid transition: $current → $target")
            )
        }
    }
}
```

### Priority Queue & Scheduling

The scheduler decides which downloads run and in what order. I'd use **priority-based with FIFO tiebreaker** -- user-initiated downloads get `HIGH`, prefetch gets `LOW`, and the user can manually promote items to `URGENT`. Chrome uses a similar system where user-clicked downloads are high priority while prefetch is low.

```kotlin
class DownloadScheduler(
    private val maxConcurrent: Int = 3,
    private val repository: DownloadRepository,
    private val engine: DownloadEngine,
    private val connectivityMonitor: ConnectivityMonitor,
) {
    private val activeDownloads = ConcurrentHashMap<DownloadId, Job>()
    private val mutex = Mutex()

    suspend fun schedule(downloadId: DownloadId) = mutex.withLock {
        if (activeDownloads.size < maxConcurrent) {
            startDownload(downloadId)
        }
        // else: stays QUEUED, will start when a slot opens
    }

    suspend fun onDownloadFinished(downloadId: DownloadId) = mutex.withLock {
        activeDownloads.remove(downloadId)
        promoteNextFromQueue()
    }

    private suspend fun promoteNextFromQueue() {
        val next = repository.getHighestPriorityQueued() ?: return
        if (shouldDownload(next)) {
            startDownload(next.id)
        }
    }

    private fun shouldDownload(download: DownloadRecord): Boolean {
        if (download.wifiOnly && !connectivityMonitor.isUnmetered()) return false
        return true
    }
}
```

### Parallel Chunk Downloading

A single TCP connection's throughput is limited by the congestion window and RTT -- multiple connections multiply effective bandwidth, especially on high-latency links. Skip parallel chunks when: file < 50 MB, server doesn't support Range, metered network, or server rate-limits per-IP.

```kotlin
data class ChunkPlan(
    val chunkIndex: Int,
    val startByte: Long,
    val endByte: Long,
    val tempFile: File,
    var downloadedBytes: Long = 0L,
)

suspend fun downloadParallel(
    url: String,
    totalSize: Long,
    chunkCount: Int,
    outputFile: File,
): DownloadResult = coroutineScope {
    val chunkSize = totalSize / chunkCount
    val chunks = (0 until chunkCount).map { i ->
        ChunkPlan(
            chunkIndex = i,
            startByte = i * chunkSize,
            endByte = if (i == chunkCount - 1) totalSize - 1 else (i + 1) * chunkSize - 1,
            tempFile = File(outputFile.parent, "${outputFile.name}.chunk$i"),
        )
    }

    val results = chunks.map { chunk ->
        async(Dispatchers.IO) { downloadChunk(url, chunk) }
    }.awaitAll()

    outputFile.outputStream().use { out ->
        chunks.sortedBy { it.chunkIndex }.forEach { chunk ->
            chunk.tempFile.inputStream().use { input -> input.copyTo(out) }
            chunk.tempFile.delete()
        }
    }

    DownloadResult.Success(0, totalSize)
}
```

!!! note
    Telegram uses parallel chunk downloads for large files, splitting them into 512 KB - 1 MB pieces and downloading up to 4 concurrently. They also use multiple data center connections for geographic distribution.

### Background Download Service

The hardest part of a mobile download manager -- the OS actively fights you. I'd use **Foreground Service for active downloads + WorkManager for recovery after process death.**

| Option | Max Duration | Survives Death | Network in Doze | Use Case |
|--------|-------------|----------------|-----------------|----------|
| **Foreground Service** | Unlimited (with notification) | Yes (restarts) | Yes | Active downloads user is aware of |
| **WorkManager** | 10 min (configurable) | Yes (rescheduled) | Expedited only | Recovery after process death |
| **System DownloadManager** | Unlimited | Yes (system-managed) | Yes | Simple downloads without custom UI |
| **Coroutine in ViewModel** | While UI is alive | No | No | Never for downloads |

```kotlin
class DownloadForegroundService : Service() {
    private val downloadManager: DownloadManager by inject()
    private val notificationController: DownloadNotificationController by inject()

    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        val downloadId = intent?.getStringExtra(EXTRA_DOWNLOAD_ID)
            ?: return START_NOT_STICKY

        startForeground(NOTIFICATION_ID, notificationController.buildGroupNotification())

        serviceScope.launch {
            downloadManager.observe(DownloadId(downloadId))
                .collect { state ->
                    notificationController.updateProgress(state)
                    if (state.status.isTerminal()) {
                        checkAndStopSelf()
                    }
                }
        }

        return START_REDELIVER_INTENT // Re-deliver intent if killed
    }

    private fun checkAndStopSelf() {
        if (downloadManager.activeCount() == 0) {
            stopForeground(STOP_FOREGROUND_REMOVE)
            stopSelf()
        }
    }
}
```

**WorkManager for recovery:**

```kotlin
class DownloadRecoveryWorker(
    context: Context,
    params: WorkerParameters,
    private val downloadManager: DownloadManager,
) : CoroutineWorker(context, params) {

    override suspend fun doWork(): Result {
        val interrupted = downloadManager.getInterruptedDownloads()
        interrupted.forEach { download ->
            downloadManager.resume(download.id)
        }
        return Result.success()
    }

    companion object {
        fun scheduleRecovery(context: Context) {
            val constraints = Constraints.Builder()
                .setRequiredNetworkType(NetworkType.CONNECTED)
                .build()

            val request = OneTimeWorkRequestBuilder<DownloadRecoveryWorker>()
                .setConstraints(constraints)
                .setBackoffCriteria(BackoffPolicy.EXPONENTIAL, 30, TimeUnit.SECONDS)
                .build()

            WorkManager.getInstance(context)
                .enqueueUniqueWork("download_recovery", ExistingWorkPolicy.KEEP, request)
        }
    }
}
```

!!! warning "Edge Case"
    On Android 12+, foreground service launch from background is restricted. You must either start the service while the app is in foreground, use `WorkManager.setExpedited()`, or declare a foreground service type (`dataSync`) in the manifest. Missing this causes `ForegroundServiceStartNotAllowedException`.

### Progress Tracking

The key insight is multi-tier update rates -- in-memory for UI (every 500ms via conflated Flow), batched DB writes (every 1 MB or 5s), and rate-limited notifications (every 2s):

```kotlin
class ProgressTracker(
    private val repository: DownloadRepository,
    private val notificationController: DownloadNotificationController,
) {
    private val _progress = MutableStateFlow(ProgressSnapshot.EMPTY)
    val progress: StateFlow<ProgressSnapshot> = _progress.asStateFlow()

    private var lastDbWrite = 0L
    private var lastNotificationUpdate = 0L
    private val speedSamples = ArrayDeque<SpeedSample>(maxSize = 6) // 3s window

    fun onBytesRead(bytesRead: Long, totalBytes: Long) {
        val now = SystemClock.elapsedRealtime()

        // Always update in-memory (UI gets this via conflated Flow)
        speedSamples.addLast(SpeedSample(now, bytesRead))
        val speed = calculateRollingSpeed()
        val eta = if (speed > 0) (totalBytes - bytesRead) / speed else -1

        _progress.value = ProgressSnapshot(bytesRead, totalBytes, speed, eta)

        // Batch DB writes: every 1 MB or 5 seconds
        if (bytesRead - lastDbWrite > 1_048_576 || now - lastDbWrite > 5_000) {
            repository.updateProgress(bytesRead)
            lastDbWrite = now
        }

        // Rate-limit notifications: every 2 seconds
        if (now - lastNotificationUpdate > 2_000) {
            notificationController.updateProgress(bytesRead, totalBytes, speed)
            lastNotificationUpdate = now
        }
    }
}
```

!!! tip "Pro Tip"
    Use `SystemClock.elapsedRealtime()` instead of `System.currentTimeMillis()` for speed calculations. Wall clock time can jump (NTP sync, user changes), but elapsed realtime is monotonic.

### Storage Management

Often overlooked but critical. Temp files are named `{filename}.download.tmp`, pre-allocated to full size to catch disk-full early, atomically renamed only after completion + checksum validation, and deleted immediately on cancel. On startup, orphaned `.download.tmp` files without matching DB records are cleaned up.

```kotlin
class StorageManager(private val context: Context) {

    fun allocateTempFile(fileName: String, expectedSize: Long): Result<File> {
        val downloadDir = context.getExternalFilesDir(Environment.DIRECTORY_DOWNLOADS)
            ?: return Result.failure(StorageError.NoExternalStorage)

        val available = StatFs(downloadDir.path).availableBytes
        if (expectedSize > 0 && expectedSize > available - SAFETY_MARGIN) {
            return Result.failure(StorageError.InsufficientSpace(expectedSize, available))
        }

        val tempFile = File(downloadDir, "$fileName.download.tmp")
        return Result.success(tempFile)
    }

    fun commitDownload(tempFile: File, finalName: String): Result<File> {
        val finalFile = File(tempFile.parentFile, finalName)
        return if (tempFile.renameTo(finalFile)) {
            Result.success(finalFile)
        } else {
            // renameTo fails across mount points; fall back to copy + delete
            tempFile.copyTo(finalFile, overwrite = true)
            tempFile.delete()
            Result.success(finalFile)
        }
    }

    companion object {
        private const val SAFETY_MARGIN = 50 * 1024 * 1024L // 50 MB buffer
    }
}
```

!!! warning "Edge Case"
    `File.renameTo()` fails silently (returns `false`) when source and destination are on different mount points -- e.g., internal storage to SD card. Always check the return value and fall back to copy + delete.

### Network Constraints & Bandwidth Throttling

The `ConnectivityMonitor` uses `NetworkCallback` to track connectivity, metered status, and downstream bandwidth in real-time. WiFi-only downloads pause automatically when the network becomes metered. For bandwidth throttling, I'd use a **token bucket via an OkHttp Interceptor**:

```kotlin
class ThrottledSource(
    private val delegate: Source,
    private val maxBytesPerSecond: Long,
) : Source {
    private var bytesReadThisSecond = 0L
    private var windowStart = System.nanoTime()

    override fun read(sink: Buffer, byteCount: Long): Long {
        throttleIfNeeded()
        val bytesRead = delegate.read(sink, minOf(byteCount, CHUNK_SIZE))
        if (bytesRead > 0) bytesReadThisSecond += bytesRead
        return bytesRead
    }

    private fun throttleIfNeeded() {
        val elapsed = System.nanoTime() - windowStart
        if (elapsed >= 1_000_000_000) {
            bytesReadThisSecond = 0
            windowStart = System.nanoTime()
            return
        }
        if (bytesReadThisSecond >= maxBytesPerSecond) {
            val sleepMs = (1_000_000_000 - elapsed) / 1_000_000
            Thread.sleep(sleepMs)
            bytesReadThisSecond = 0
            windowStart = System.nanoTime()
        }
    }
}
```

### Integrity Verification

```kotlin
suspend fun verifyChecksum(
    file: File,
    expectedSha256: String,
    onProgress: (bytesProcessed: Long, total: Long) -> Unit,
): Boolean = withContext(Dispatchers.IO) {
    val digest = MessageDigest.getInstance("SHA-256")
    val buffer = ByteArray(8192)
    val total = file.length()
    var processed = 0L

    file.inputStream().use { input ->
        var read: Int
        while (input.read(buffer).also { read = it } != -1) {
            digest.update(buffer, 0, read)
            processed += read
            onProgress(processed, total)
        }
    }

    val actual = digest.digest().joinToString("") { "%02x".format(it) }
    actual.equals(expectedSha256, ignoreCase = true)
}
```

!!! tip "Pro Tip"
    Run checksum verification on `Dispatchers.IO` and show a "Verifying..." state in the UI. For a 1 GB file, SHA-256 takes 2-5 seconds on modern devices. Don't block the completion callback on this -- show completion immediately and verify in the background if speed matters more than safety.

### Retry Strategy

Transient errors (network timeouts, HTTP 429/5xx, DNS failures) retry with exponential backoff (1s, 2s, 4s, 8s, 16s, max 5). Permanent errors (404, 403, SSL, disk-full) surface to the user immediately.

```kotlin
class RetryPolicy(
    private val maxRetries: Int = 5,
    private val baseDelayMs: Long = 1_000,
    private val maxDelayMs: Long = 60_000,
) {
    fun shouldRetry(error: DownloadError, attempt: Int): RetryDecision {
        if (attempt >= maxRetries) return RetryDecision.GiveUp
        if (!error.isTransient) return RetryDecision.GiveUp

        val delay = minOf(
            baseDelayMs * (1L shl attempt) + Random.nextLong(0, 1000), // jitter
            maxDelayMs,
        )
        return RetryDecision.RetryAfter(delay)
    }
}

sealed class RetryDecision {
    data class RetryAfter(val delayMs: Long) : RetryDecision()
    data object GiveUp : RetryDecision()
}
```

!!! tip "Pro Tip"
    Always add jitter to exponential backoff. Without jitter, all clients that failed at the same time retry simultaneously, creating a "thundering herd" that overwhelms the server again. This is especially relevant for CDN outages.

---

## Scalability, Reliability & Edge Cases

| Scenario | Decision | Reasoning |
|----------|----------|-----------|
| **Server removes file mid-download** | Transition to FAILED, keep partial file for 24h | User might find an alternative source; auto-delete after grace period |
| **Disk full during download** | Pause all downloads, notify user, check periodically | Pause is more recoverable than fail -- user might free space |
| **Redirect chains** | Follow redirects, store final URL for resume | Some CDNs use signed redirect URLs that expire; store original URL too and re-resolve |
| **File already exists at destination** | Append `(1)`, `(2)` suffix like Chrome | Never silently overwrite |
| **WiFi -> cellular mid-download** | Pause WiFi-only downloads, continue others | Respect data preferences via `ConnectivityManager.NetworkCallback` |
| **ETag changes between pause and resume** | Restart from byte 0 | File content changed; partial data is invalid. Inform user why progress was lost |
| **Server returns wrong Content-Length** | Detect mismatch on completion, re-download if checksum fails | Some servers lie (gzip misconfiguration); checksum is the final arbiter |
| **Process killed during write** | On restart, verify temp file size matches DB offset | If temp file is smaller (OS didn't flush), adjust offset downward |
| **Concurrent resume of same URL** | Deduplicate by URL, reject second enqueue | Return existing download ID; don't waste bandwidth on duplicates |
| **VPN connects/disconnects** | Treat as network change, verify connectivity to host | VPN changes can make hosts unreachable or newly reachable |
| **Device reboots mid-download** | WorkManager `BOOT_COMPLETED` triggers recovery worker | All state is in DB; recovery worker queries interrupted downloads |
| **Signed URL expires before completion** | Detect 403, request fresh URL from app's backend | Large files on S3/GCS use signed URLs with TTL; the app needs a "refresh URL" API |

---

## Database Schema

```sql
CREATE TABLE downloads (
    id              TEXT PRIMARY KEY,
    url             TEXT NOT NULL,
    final_url       TEXT,               -- After redirect resolution
    file_name       TEXT NOT NULL,
    destination_dir TEXT NOT NULL,
    temp_file_path  TEXT,
    status          TEXT NOT NULL DEFAULT 'QUEUED',
    priority        INTEGER NOT NULL DEFAULT 1,
    total_bytes     INTEGER NOT NULL DEFAULT -1,
    downloaded_bytes INTEGER NOT NULL DEFAULT 0,
    etag            TEXT,
    checksum_sha256 TEXT,               -- Expected checksum (nullable)
    wifi_only       INTEGER NOT NULL DEFAULT 0,
    retry_count     INTEGER NOT NULL DEFAULT 0,
    error_message   TEXT,
    headers_json    TEXT,               -- Custom request headers
    created_at      INTEGER NOT NULL,
    updated_at      INTEGER NOT NULL,
    completed_at    INTEGER
);

CREATE INDEX idx_downloads_status ON downloads(status);
CREATE INDEX idx_downloads_priority_created
    ON downloads(priority DESC, created_at ASC);

CREATE TABLE chunk_progress (
    download_id     TEXT NOT NULL REFERENCES downloads(id) ON DELETE CASCADE,
    chunk_index     INTEGER NOT NULL,
    start_byte      INTEGER NOT NULL,
    end_byte        INTEGER NOT NULL,
    downloaded_bytes INTEGER NOT NULL DEFAULT 0,
    temp_file_path  TEXT NOT NULL,
    PRIMARY KEY (download_id, chunk_index)
);
```

!!! note
    The `chunk_progress` table only has rows for downloads using parallel chunking. Single-stream downloads track progress entirely in the `downloads` table, avoiding unnecessary complexity for small files.

---

## Wrap Up

- **HTTP Range requests as the foundation** -- universal CDN support, resumable by design, no custom protocol needed. ETag validation on resume detects server-side file changes.
- **Foreground Service + WorkManager** -- foreground service keeps the process alive for active downloads; WorkManager guarantees recovery after process death or reboot.
- **Priority queue with concurrency limit** -- prevents bandwidth starvation while giving users control. Parallel chunks only for files > 50 MB.
- **Batched progress persistence** -- writing every byte offset to DB would thrash I/O; 1 MB / 5s batching balances durability and performance. Multi-tier updates: in-memory for UI, batched for DB, rate-limited for notifications.
- **Atomic temp-file-then-rename** -- prevents corrupted partial files from appearing as "completed"; downstream consumers never see incomplete data.

**What I'd improve with more time:** Multi-source downloads from different CDN edges for maximum throughput, adaptive chunk sizing based on measured bandwidth, predictive scheduling using historical speed data, compression-aware downloading (`Content-Encoding: gzip` breaks Range math), download groups with group-level pause/resume, and peer-to-peer assist for app-specific content.

---

## References

- [Android DownloadManager](https://developer.android.com/reference/android/app/DownloadManager) -- System-provided download manager; useful for comparison, limited for custom UIs
- [WorkManager Guide](https://developer.android.com/topic/libraries/architecture/workmanager) -- Background task scheduling that survives process death
- [Foreground Services (Android 14+)](https://developer.android.com/develop/background-work/services/foreground-services) -- Updated restrictions and required types
- [HTTP Range Requests (MDN)](https://developer.mozilla.org/en-US/docs/Web/HTTP/Range_requests) -- Complete reference for Range, Content-Range, If-Range headers
- [OkHttp](https://square.github.io/okhttp/) -- HTTP client with interceptor chain, connection pooling, transparent compression
- [Ktor Client](https://ktor.io/docs/client.html) -- KMP-compatible HTTP client with streaming support
- [PRDownloader](https://github.com/MindorksOpenSource/PRDownloader) -- Open-source Android download manager library; good reference implementation
- [Telegram Source (TDLib)](https://github.com/tdlib/td) -- Telegram's download engine handles chunked parallel downloads across data centers
- [Chrome Download Architecture](https://chromium.googlesource.com/chromium/src/+/main/components/download/) -- Chromium's download component; multi-process, resumable, parallel
