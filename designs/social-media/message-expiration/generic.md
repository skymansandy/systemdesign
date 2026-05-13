# Backend Architecture

How do apps like Telegram, Signal, and Snapchat delete messages after a TTL expires -- in real time, at scale, without melting their databases? This is a nuanced infrastructure problem that tests your understanding of storage engines, distributed scheduling, consistency trade-offs, and the gap between "deleted from the UI" and "deleted from disk." Walk through it methodically: clarify what "expiration" means, design the storage layer, then drill into the hard parts -- clock skew, deletion throughput at scale, and cryptographic erasure.

!!! note "Mobile Perspective"
    For client-side expiration -- local DB cleanup, UI countdown timers, process-death resilience, and screenshot prevention -- see [Mobile Client Architecture](mobile.md) *(not yet written)*.

---

## Problem & Design Scope

### Clarifying Questions

| # | Question | Why It Matters |
|---|----------|---------------|
| 1 | **Who sets the TTL?** | Per-message (Telegram), per-chat (Signal), or global policy (Snapchat) -- each has different storage implications |
| 2 | **What does "deleted" mean?** | Removed from UI? Tombstoned? Cryptographically erased? Purged from backups? Compliance requirements change the answer |
| 3 | **What's the TTL range?** | 5 seconds (Snapchat) vs. 7 days (Telegram) drives whether you need sub-second precision or batch daily cleanup |
| 4 | **How many messages per second need expiration checks?** | 10K/sec is a different problem than 10M/sec |
| 5 | **Must deletion be exact-time or best-effort?** | "Delete at exactly T+24h" requires a scheduler. "Delete within a few minutes of T+24h" allows batch processing |
| 6 | **Multi-device -- must deletion propagate?** | If Alice's message expires, it must disappear from Bob's phone, Alice's phone, and Alice's tablet simultaneously |
| 7 | **Is the message content or just the reference deleted?** | Media (images, video) stored in object storage needs separate cleanup from the DB row |
| 8 | **Audit trail requirements?** | Some jurisdictions require proof of deletion. Metadata about the deletion event may need to persist even after the content is gone |
| 9 | **Does deletion affect read replicas and caches?** | A message "deleted" in the primary but still served from a Redis cache or CDN edge is not really deleted |
| 10 | **Encryption model?** | With E2E encryption, the server may not even know the message content -- but it still stores the ciphertext blob |

### Functional Requirements

| Requirement | Details |
|-------------|---------|
| **Per-message TTL** | Sender sets a self-destruct timer (e.g., 5s, 1h, 24h, 7d) when composing |
| **Automatic deletion** | Messages are removed after TTL expires, no user action required |
| **Multi-device propagation** | Deletion is pushed to all participants' devices |
| **Media cleanup** | Associated images, video, files are purged from object storage |
| **Deletion confirmation** | Sender is notified that the message was successfully deleted across all devices |

### Non-Functional Requirements

| Requirement | Target | Rationale |
|-------------|--------|-----------|
| **Deletion accuracy** | Within 30 seconds of TTL expiry | Users trust the timer. A message lingering 5 minutes past expiry is a privacy violation |
| **Throughput** | Handle 1M message expirations/minute | At Telegram's scale (700M+ MAU), expired messages accumulate fast |
| **No data leakage** | Content not recoverable post-deletion | This is often a legal/compliance requirement, not just a UX one |
| **Availability** | Deletion must not depend on the recipient being online | Server-side deletion happens regardless of device state |
| **Consistency** | All replicas/caches purged within 60 seconds | Stale reads of deleted messages erode user trust |
| **Scalability** | Linear cost scaling with message volume | Deletion infra shouldn't become the bottleneck as traffic grows |

### Capacity Estimation

| Parameter | Value |
|-----------|-------|
| DAU | 200M |
| Messages with TTL (% of total) | 15% |
| Total messages/day | 10B |
| Expiring messages/day | 1.5B |
| Expirations/second (avg) | ~17K/sec |
| Peak expirations/second (3x) | ~50K/sec |
| Avg message size | 300 bytes |
| Storage reclaimed/day | 1.5B x 300B ≈ 450 GB/day |

---

## API Design

### Setting TTL on a Message

```
POST /api/v1/messages
{
  "conversation_id": "conv_abc",
  "body": "This message will self-destruct",
  "media_ids": ["media_123"],
  "ttl_seconds": 86400,           // 24 hours
  "ttl_start": "on_read"          // "on_send" or "on_read"
}
```

!!! tip "Pro Tip"
    The `ttl_start` field is a critical design decision. **"on_send"** means the timer starts when the server receives the message -- simpler, deterministic, server-controlled. **"on_read"** means the timer starts when the recipient first opens it -- requires client-side reporting of read events. Telegram uses "on_read" for secret chats. Snapchat uses "on_open." Signal uses "on_read." Each has different server-side implications.

### Deletion Signal to Clients

```
// WebSocket push to all devices in the conversation
{
  "type": "message_expired",
  "message_id": "msg_xyz",
  "conversation_id": "conv_abc",
  "expired_at": "2025-03-16T10:30:00Z"
}

// For offline devices: processed on next sync
GET /api/v1/conversations/{conv_id}/sync?since={last_sync_cursor}
→ includes deletion tombstones in the sync response
```

---

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Message Flow                              │
│                                                                  │
│  ┌──────────┐    ┌──────────────┐    ┌──────────────────────┐   │
│  │  Client   │───▶│  API Gateway  │───▶│  Message Service     │   │
│  │ (send msg │    │              │    │  - stores message    │   │
│  │  w/ TTL)  │    └──────────────┘    │  - publishes to      │   │
│  └──────────┘                        │    expiration queue   │   │
│                                       └──────────┬───────────┘   │
│                                                  │               │
│                    ┌─────────────────────────────┼───────────┐   │
│                    │                             ▼           │   │
│                    │   ┌───────────────────────────────────┐ │   │
│                    │   │      Expiration Scheduler         │ │   │
│                    │   │  (the core of this design)        │ │   │
│                    │   │                                   │ │   │
│                    │   │  Option A: DB TTL Index           │ │   │
│                    │   │  Option B: Delay Queue            │ │   │
│                    │   │  Option C: Time-Bucketed Scanner  │ │   │
│                    │   └──────────────┬────────────────────┘ │   │
│                    │                  │                      │   │
│                    │     ┌────────────▼────────────┐         │   │
│                    │     │   Deletion Executor     │         │   │
│                    │     │  - delete from DB       │         │   │
│                    │     │  - delete from S3/CDN   │         │   │
│                    │     │  - invalidate caches    │         │   │
│                    │     │  - push to clients      │         │   │
│                    │     └────────────┬────────────┘         │   │
│                    │                  │                      │   │
│                    │     ┌────────────▼────────────┐         │   │
│                    │     │    Notification Fan-out │         │   │
│                    │     │  - WebSocket push       │         │   │
│                    │     │  - Push notification    │         │   │
│                    │     │  - Sync tombstone       │         │   │
│                    │     └─────────────────────────┘         │   │
│                    │         Expiration Pipeline             │   │
│                    └────────────────────────────────────────┘    │
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────────────────┐  │
│  │  Message DB   │  │  Object Store │  │  Cache (Redis)       │  │
│  │  (Cassandra)  │  │  (S3)        │  │  - recent messages   │  │
│  │              │  │  - media     │  │  - conversation list │  │
│  └──────────────┘  └──────────────┘  └───────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Design Deep Dive

### The Core Problem: How to Find Expired Messages Efficiently

This is where the interview gets interesting. There are three fundamentally different approaches, each with distinct trade-offs.

#### Option A: Database TTL (Cassandra TTL)

Cassandra and ScyllaDB natively support per-row TTL. When you write a message, you set the TTL, and Cassandra deletes it automatically during compaction.

```cql
INSERT INTO messages (conversation_id, message_id, body, sender_id, created_at)
VALUES ('conv_abc', 'msg_xyz', 'Secret message', 'user_1', toTimestamp(now()))
USING TTL 86400;   -- auto-deleted after 24 hours
```

| Pros | Cons |
|------|------|
| Zero application-level scheduling code | Deletion happens during **compaction**, not at exact TTL expiry. Can lag minutes to hours |
| Battle-tested at scale (Cassandra handles this natively) | No hook to trigger side effects (media cleanup, cache invalidation, client notification) |
| Storage reclaimed automatically | "On-read" TTL start is impossible -- Cassandra TTL is set at write time |
| Simple operational model | No deletion confirmation or audit trail |

**Verdict:** Good for "fire-and-forget" deletion where exact timing and side effects don't matter. Not sufficient alone for a Telegram-like feature where you need client notification and media cleanup.

#### Option B: Delay Queue (Kafka + Scheduled Delivery)

When a message with TTL is stored, enqueue a delayed deletion event. The event fires at the exact expiry time.

```
Message stored at T=0 with TTL=24h
  → Enqueue: { action: "delete", message_id: "msg_xyz", fire_at: T+24h }

At T+24h, the delay queue delivers the event to the Deletion Executor
```

**Implementation options for the delay queue:**

| Technology | How It Works | Trade-offs |
|------------|-------------|------------|
| **Kafka + consumer with time check** | Produce to a topic partitioned by expiry hour. Consumers poll and check `fire_at <= now()` | Simple, but consumers must handle out-of-order arrivals and rebalancing |
| **Amazon SQS Delay Queue** | Set `DelaySeconds` (max 15 min) or use `MessageTimer` | Max delay is 15 minutes -- need chaining for longer TTLs |
| **Redis sorted set** | `ZADD expiry_queue <expiry_timestamp> <message_id>`. Poll with `ZRANGEBYSCORE` | Fast, but Redis is in-memory -- 1.5B entries/day needs significant RAM |
| **Custom wheel timer** | Hierarchical timing wheel (Netty-style) with disk-backed overflow | Most precise, but complex to build and operate |
| **Database-backed scheduler** | `expiration_queue` table with `fire_at` index, polled by workers | Simple, durable, but polling adds DB load |

!!! tip "Pro Tip"
    **The database-backed scheduler is underrated** for this use case. A simple table with a `fire_at` index, polled every 5 seconds by N workers using `SELECT ... WHERE fire_at <= now() ORDER BY fire_at LIMIT 1000 FOR UPDATE SKIP LOCKED`, handles 50K/sec easily on a modern Postgres instance. It's durable, debuggable, and doesn't require a separate queue infrastructure. This is what most teams should start with before reaching for Kafka.

#### Option C: Time-Bucketed Scanner (Recommended for Scale)

Partition expiring messages into **time buckets** (e.g., 1-minute windows). A fleet of scanner workers processes each bucket when its time arrives.

```
┌──────────────────────────────────────────────────────────┐
│                   Expiry Index Table                      │
│                                                          │
│  Partition Key: expiry_bucket (minute granularity)       │
│  Clustering Key: message_id                              │
│                                                          │
│  ┌─────────────────┬──────────────────────────────────┐  │
│  │ expiry_bucket    │ message_ids                      │  │
│  ├─────────────────┼──────────────────────────────────┤  │
│  │ 2025-03-16T10:30 │ msg_001, msg_002, ..., msg_847  │  │
│  │ 2025-03-16T10:31 │ msg_848, msg_849, ..., msg_1203 │  │
│  │ 2025-03-16T10:32 │ msg_1204, ...                   │  │
│  └─────────────────┴──────────────────────────────────┘  │
└──────────────────────────────────────────────────────────┘

Scanner workers:
  Every minute, pick the current bucket
  Process all message_ids in that bucket
  Execute deletion pipeline for each
  Mark bucket as processed
```

```sql
-- Expiry index table (Cassandra)
CREATE TABLE message_expiry_index (
    expiry_bucket   TIMESTAMP,        -- rounded to minute
    message_id      TEXT,
    conversation_id TEXT,
    has_media       BOOLEAN,
    PRIMARY KEY (expiry_bucket, message_id)
) WITH default_time_to_live = 604800;  -- auto-cleanup after 7 days

-- Scanner query (executed every minute by worker fleet)
SELECT * FROM message_expiry_index
WHERE expiry_bucket = '2025-03-16T10:30:00';
```

| Pros | Cons |
|------|------|
| Predictable, bounded work per cycle | Granularity is limited to bucket size (1 min) |
| Horizontally scalable (shard buckets across workers) | Requires coordination to avoid double-processing |
| No long-lived delayed messages in a queue | Hot buckets (many expirations in one minute) need sub-partitioning |
| Works natively with Cassandra's partition model | Two writes per message (message + expiry index) |

!!! warning "Edge Case"
    **Hot buckets**: If a viral group chat sets 24h TTL and gets 100K messages in one minute, the bucket for T+24h will have 100K entries. Solution: sub-partition by conversation_id or add a random shard key (`expiry_bucket, shard_id, message_id`) and distribute scanner workers across shards.

**Verdict: Use Option C (time-bucketed scanner) as the primary mechanism, with Option A (Cassandra TTL) as a safety net.** The scanner handles side effects (media cleanup, cache invalidation, client notification). Cassandra TTL ensures that even if the scanner misses a message, the data is eventually purged from storage.

### The Deletion Pipeline

Once the scanner identifies expired messages, each deletion triggers a pipeline:

```
┌─────────────┐     ┌───────────────┐     ┌──────────────┐
│ 1. DB Delete │────▶│ 2. Media      │────▶│ 3. Cache     │
│ (tombstone   │     │    Cleanup    │     │    Invalidate│
│  in Cassandra│     │ (S3 delete)  │     │ (Redis DEL)  │
│  + write     │     │              │     │              │
│  deletion    │     │              │     │              │
│  event)      │     │              │     │              │
└─────────────┘     └───────────────┘     └──────────────┘
       │                                         │
       ▼                                         ▼
┌──────────────┐                        ┌──────────────┐
│ 4. Client    │                        │ 5. Audit Log │
│    Push      │                        │ (if required)│
│ (WebSocket + │                        │              │
│  push notif) │                        │              │
└──────────────┘                        └──────────────┘
```

**Key design decisions in the pipeline:**

1. **Tombstones, not hard deletes**: Write a tombstone record (`{ message_id, deleted_at, reason: "ttl_expired" }`) before deleting the content. This allows offline devices to sync the deletion and is essential for consistency.

2. **Media deletion is async and batched**: S3 `DeleteObjects` supports batch deletion of up to 1,000 keys. Accumulate media keys and batch-delete every few seconds.

3. **Cache invalidation is best-effort**: Redis keys may have already expired via their own TTL. Use `DEL` if the key exists, but don't fail the pipeline if the cache miss.

4. **Client push is fire-and-forget**: Send the WebSocket event. If the client is offline, the tombstone will be picked up during next sync. Don't block on delivery confirmation.

### "On-Read" TTL: The Hard Variant

When TTL starts on first read (Telegram secret chats, Snapchat), the server doesn't know when to start the timer until the client reports a read event.

```
Sequence:
1. Alice sends message with ttl=60s, ttl_start=on_read
2. Server stores message with expiry_at=NULL
3. Bob opens the chat → client sends read receipt
4. Server sets expiry_at = now() + 60s
5. Server enqueues expiration event for T+60s
6. At T+60s, deletion pipeline fires
```

!!! warning "Edge Case"
    **Multi-device read race**: Bob opens the message on his phone and tablet simultaneously. Both devices send read receipts. The server must ensure `expiry_at` is set only once (idempotent write). Use `UPDATE messages SET expiry_at = :time WHERE id = :id IF expiry_at IS NULL` (Cassandra lightweight transaction) or a compare-and-swap in Redis.

### Clock Skew and Distributed Timing

In a distributed system, "now" is not a single value. Different servers have slightly different clocks.

| Problem | Impact | Mitigation |
|---------|--------|-----------|
| **Server clock skew** | Message expires 30s early or late on different nodes | Use NTP with tight sync (< 1s). Accept ±30s accuracy for TTLs > 1 minute |
| **Client clock manipulation** | Malicious client reports fake read time to delay/accelerate TTL | Server always uses its own clock for `expiry_at`, never trusts client timestamps |
| **Timezone confusion** | TTL stored as absolute time in different zones | Always use UTC epoch milliseconds internally. Never store local time |
| **Leap seconds** | Rare but can cause 1s discontinuity | Use TAI or monotonic clocks for timer comparisons. In practice, ±1s is fine for TTL |

### Cryptographic Erasure: True Deletion

Simply deleting a database row doesn't guarantee the data is gone. It may exist in:
- Replication logs (Cassandra commit log, Kafka topics)
- Backup snapshots
- OS file system (deleted but not overwritten)
- Read replicas with replication lag

**Cryptographic erasure** solves this: encrypt each message with a unique key. To "delete" the message, delete the key. The ciphertext becomes unrecoverable.

```
Write path:
  1. Generate per-message key K
  2. Encrypt message: ciphertext = AES-GCM(K, plaintext)
  3. Store ciphertext in message DB
  4. Store K in a separate key store (Redis, Vault) with TTL

Deletion path:
  1. Delete K from key store
  2. Ciphertext in DB, replicas, backups is now irrecoverable
  3. Background job eventually purges the ciphertext rows
```

!!! tip "Pro Tip"
    This is an **interview power move**. If you mention cryptographic erasure, you demonstrate awareness of the gap between "application-level delete" and "data is truly gone." Most candidates stop at `DELETE FROM messages`. Bringing up crypto erasure shows you think about compliance (GDPR right to erasure), backup recovery scenarios, and defense in depth.

---

## Data Model & Storage

### Primary Message Store (Cassandra)

```cql
CREATE TABLE messages (
    conversation_id TEXT,
    message_id      TIMEUUID,
    sender_id       TEXT,
    body_encrypted  BLOB,          -- ciphertext if using crypto erasure
    media_refs      LIST<TEXT>,     -- S3 keys for attached media
    ttl_seconds     INT,           -- original TTL value (0 = no expiry)
    ttl_start       TEXT,          -- 'on_send' or 'on_read'
    expiry_at       TIMESTAMP,     -- NULL until TTL is activated
    created_at      TIMESTAMP,
    PRIMARY KEY (conversation_id, message_id)
) WITH CLUSTERING ORDER BY (message_id DESC);
```

### Deletion Tombstone Store

```cql
CREATE TABLE deletion_events (
    conversation_id TEXT,
    message_id      TIMEUUID,
    deleted_at      TIMESTAMP,
    reason          TEXT,           -- 'ttl_expired', 'user_deleted', 'moderation'
    PRIMARY KEY (conversation_id, deleted_at)
) WITH CLUSTERING ORDER BY (deleted_at DESC)
  AND default_time_to_live = 2592000;  -- tombstones kept for 30 days for sync
```

### Key Store (for Cryptographic Erasure)

```
Redis / Vault:
  Key:   msg_key:{message_id}
  Value: AES-256 key (32 bytes)
  TTL:   matches message TTL
```

---

## Scalability & Reliability

### Horizontal Scaling of the Scanner

```
                    ┌─────────────────────┐
                    │   Coordinator       │
                    │   (assigns buckets  │
                    │    to workers)      │
                    └────────┬────────────┘
                             │
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
        ┌──────────┐  ┌──────────┐  ┌──────────┐
        │ Worker 1 │  │ Worker 2 │  │ Worker 3 │
        │ bucket   │  │ bucket   │  │ bucket   │
        │ :00-:19  │  │ :20-:39  │  │ :40-:59  │
        └──────────┘  └──────────┘  └──────────┘
```

- **Partition assignment**: Use consistent hashing or a simple modulo on `bucket_minute % num_workers`. Workers claim ownership via a distributed lock (ZooKeeper, etcd).
- **Idempotent deletion**: If a worker crashes mid-bucket, the next worker can re-process safely. Deletion of an already-deleted message is a no-op.
- **Backpressure**: If deletions are falling behind (bucket queue growing), auto-scale workers. Monitor `current_time - oldest_unprocessed_bucket` as the key lag metric.

### Failure Modes

| Failure | Impact | Mitigation |
|---------|--------|-----------|
| Scanner worker dies | One minute's bucket is delayed | Watchdog detects unprocessed buckets, reassigns to healthy worker |
| Cassandra node down | Writes to expiry index fail | Retry with exponential backoff. Cassandra replication (RF=3) provides read availability |
| Redis key store outage | Can't delete encryption keys | Queue deletion events in Kafka, replay when Redis recovers |
| S3 delete fails | Orphaned media files | Dead letter queue for failed media deletions, reconciliation job runs daily |
| Network partition | Client doesn't receive deletion push | Tombstone-based sync catches up on reconnect |

### Monitoring & Alerting

| Metric | Alert Threshold | Why |
|--------|----------------|-----|
| `expiration_lag_seconds` | > 120s | Messages lingering past TTL is a privacy incident |
| `deletion_pipeline_errors_per_min` | > 100 | Systematic failure in deletion infra |
| `orphaned_media_keys_count` | > 10,000 | S3 storage leak, costs growing |
| `tombstone_sync_backlog` | > 1M | Offline devices accumulating un-synced deletions |

---

## Edge Cases & Decisions

| Scenario | Decision | Reasoning |
|----------|----------|-----------|
| **Message forwarded before expiry** | Forwarded copy gets its own TTL (or none if chat isn't TTL-enabled) | Original TTL only applies to the original conversation |
| **User goes offline before reading TTL message** | "On-read" TTL stays dormant. "On-send" TTL expires regardless | On-send is server-controlled; on-read waits for client event |
| **Backup restoration after expiry** | Cryptographic erasure ensures restored ciphertext is unreadable | Key was deleted before backup restore. This is the whole point |
| **Group chat TTL with 500 members** | Server deletes once; sends 500 deletion events | Don't delete 500 copies of the message -- there's only one row. Fan out the notification |
| **TTL change mid-flight** | Disallow. TTL is immutable once set | Allowing changes adds complexity (re-scheduling) with minimal user value |
| **Regulatory hold / legal preservation** | Override TTL for flagged accounts, store in separate compliance store | Must be invisible to the user to avoid evidence destruction |

---

## Wrap Up

The key design decisions for message expiration at scale:

1. **Time-bucketed scanner** as the primary expiration mechanism. Minute-granularity buckets, horizontally scalable workers, idempotent deletion.
2. **Cassandra native TTL** as a safety net -- ensures data is eventually purged even if the scanner misses it.
3. **Tombstone-based deletion propagation** so offline devices can sync deletions on reconnect.
4. **Cryptographic erasure** for true deletion -- delete the key, and the ciphertext becomes irrecoverable across all replicas and backups.
5. **"On-read" TTL** requires a two-phase write: store with NULL expiry, set expiry on read receipt with idempotent CAS.
6. **Monitor expiration lag** as the critical metric. Messages lingering past TTL is a privacy incident, not just a bug.

**With more time**, I'd design: configurable TTL policies per organization (enterprise compliance), integration with GDPR right-to-erasure workflows, multi-region deletion coordination with causal ordering, and a TTL visualization dashboard for end users showing exactly when each message will expire.

---

## References

- [Telegram's MTProto Protocol](https://core.telegram.org/mtproto) -- Secret chat encryption and self-destruct implementation
- [Cassandra TTL and Tombstones](https://cassandra.apache.org/doc/latest/cassandra/operating/compaction/) -- How Cassandra handles TTL-based deletion during compaction
- [Signal Protocol: Disappearing Messages](https://signal.org/blog/disappearing-messages/) -- Client-side timer-based deletion design
- [Cryptographic Erasure in Practice](https://www.usenix.org/conference/usenixsecurity19) -- Academic treatment of key-based data destruction
- [Designing Data-Intensive Applications, Ch. 8](https://dataintensive.net/) -- Martin Kleppmann on distributed clocks and timing
- [Building Reliable Reprocessing and Dead Letter Queues with Kafka](https://www.uber.com/blog/reliable-reprocessing/) -- Uber's approach to failure handling in event pipelines
