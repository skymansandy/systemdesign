# Message Expiration

How do apps like Telegram, Signal, and Snapchat delete messages after a TTL expires -- in real time, at scale, without melting their databases? This is a nuanced infrastructure problem that tests your understanding of storage engines, distributed scheduling, consistency trade-offs, and the gap between "deleted from the UI" and "deleted from disk." Walk through it methodically: clarify what "expiration" means, design the storage layer, then drill into the hard parts -- clock skew, deletion throughput at scale, and cryptographic erasure.

!!! note "Mobile Perspective"
    Client-side expiration (local DB cleanup, UI countdown timers, process-death resilience, screenshot prevention) is not yet covered in this article.

---

## Scoping the Problem

The first thing I'd want to clarify is **who sets the TTL** -- per-message (Telegram), per-chat (Signal), or global policy (Snapchat). Each has different storage implications. Per-message TTL is the most flexible and the hardest to implement, so I'd scope for that.

Next, **what does "deleted" actually mean?** Removed from the UI? Tombstoned in the database? Cryptographically erased? Purged from backups? Compliance requirements fundamentally change the answer. I'd design for content being non-recoverable post-deletion, which pushes us toward crypto erasure.

The **TTL range** matters a lot. 5 seconds (Snapchat) vs. 7 days (Telegram) drives whether you need sub-second precision or batch daily cleanup. I'd scope for the full range -- seconds to days -- which means the expiration scheduler needs to handle both.

A few other questions that meaningfully shape the design:

- **Exact-time or best-effort deletion?** "Delete at exactly T+24h" requires a real-time scheduler. "Delete within a couple minutes of T+24h" allows batch processing. I'd target within 30 seconds -- tight enough for user trust, loose enough for batching.
- **Multi-device propagation?** If Alice's message expires, it must disappear from Bob's phone, Alice's phone, and Alice's tablet simultaneously. So yes -- server-side deletion with fan-out.
- **Content vs. reference?** Media (images, video) in object storage needs separate cleanup from the DB row. Two deletion paths.
- **Does the TTL start on send or on read?** "On-send" is server-controlled and deterministic. "On-read" requires client-side reporting and introduces a two-phase lifecycle. I'd support both -- it's a critical differentiator.
- **Encryption model?** With E2E encryption, the server stores ciphertext it can't read -- but it still needs to purge it.
- **Audit trail?** Some jurisdictions require proof of deletion. Metadata about the deletion event may need to persist even after the content is gone.

**Core scope:** Per-message TTL set by the sender (5s to 7d), automatic server-side deletion, multi-device propagation, media cleanup, deletion confirmation to sender. Support both "on-send" and "on-read" TTL start modes.

**Key non-functional priorities:**

- **Deletion accuracy** -- within 30 seconds of TTL expiry. Messages lingering past expiry is a privacy violation, not just a bug.
- **Throughput** -- handle 1M expirations/minute. At Telegram's scale (700M+ MAU), expired messages accumulate fast.
- **No data leakage** -- content not recoverable post-deletion. Often a legal/compliance requirement.
- **Availability** -- deletion must not depend on the recipient being online. Server-side deletion happens regardless of device state.
- **Consistency** -- all replicas and caches purged within 60 seconds.

**Quick capacity check:** With 200M DAU, 10B messages/day, and 15% having TTL, that's ~1.5B expirations/day -- roughly 17K/sec average, 50K/sec at peak. Each deletion reclaims ~300 bytes on average, so we're reclaiming ~450 GB/day. These numbers tell me batch-per-minute processing is viable, and the storage reclamation is meaningful but not enormous.

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
// includes deletion tombstones in the sync response
```

---

## Backend Architecture

### System Overview

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

### The Core Problem: Finding Expired Messages Efficiently

This is where the interview gets interesting. There are three fundamentally different approaches, and the right answer combines two of them.

#### Option A: Database TTL (Cassandra TTL)

Cassandra and ScyllaDB natively support per-row TTL. Set it at write time and Cassandra deletes automatically during compaction.

```cql
INSERT INTO messages (conversation_id, message_id, body, sender_id, created_at)
VALUES ('conv_abc', 'msg_xyz', 'Secret message', 'user_1', toTimestamp(now()))
USING TTL 86400;   -- auto-deleted after 24 hours
```

The appeal is zero application-level scheduling code -- battle-tested, storage reclaimed automatically. But the problems are real: deletion happens during **compaction**, not at exact TTL expiry (can lag minutes to hours), there's no hook for side effects (media cleanup, cache invalidation, client notification), "on-read" TTL start is impossible since Cassandra TTL is set at write time, and there's no deletion confirmation or audit trail.

**Verdict:** Good as a safety net, but not sufficient alone. You need client notification and media cleanup, and Cassandra TTL can't trigger those.

#### Option B: Delay Queue

When a message with TTL is stored, enqueue a delayed deletion event that fires at the exact expiry time.

```
Message stored at T=0 with TTL=24h
  → Enqueue: { action: "delete", message_id: "msg_xyz", fire_at: T+24h }

At T+24h, the delay queue delivers the event to the Deletion Executor
```

Several technologies work here:

| Technology | How It Works | Trade-offs |
|------------|-------------|------------|
| **Kafka + consumer with time check** | Produce to a topic partitioned by expiry hour. Consumers poll and check `fire_at <= now()` | Simple, but consumers must handle out-of-order arrivals and rebalancing |
| **Amazon SQS Delay Queue** | Set `DelaySeconds` (max 15 min) or use `MessageTimer` | Max delay is 15 minutes -- need chaining for longer TTLs |
| **Redis sorted set** | `ZADD expiry_queue <expiry_timestamp> <message_id>`. Poll with `ZRANGEBYSCORE` | Fast, but Redis is in-memory -- 1.5B entries/day needs significant RAM |
| **Custom wheel timer** | Hierarchical timing wheel (Netty-style) with disk-backed overflow | Most precise, but complex to build and operate |
| **Database-backed scheduler** | `expiration_queue` table with `fire_at` index, polled by workers | Simple, durable, but polling adds DB load |

!!! tip "Pro Tip"
    **The database-backed scheduler is underrated** for this use case. A simple table with a `fire_at` index, polled every 5 seconds by N workers using `SELECT ... WHERE fire_at <= now() ORDER BY fire_at LIMIT 1000 FOR UPDATE SKIP LOCKED`, handles 50K/sec easily on a modern Postgres instance. It's durable, debuggable, and doesn't require separate queue infrastructure. This is what most teams should start with before reaching for Kafka.

#### Option C: Time-Bucketed Scanner (Recommended for Scale)

This is the approach I'd recommend. Partition expiring messages into **time buckets** (e.g., 1-minute windows). A fleet of scanner workers processes each bucket when its time arrives.

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

The strengths here are predictable, bounded work per cycle, horizontal scalability (shard buckets across workers), no long-lived delayed messages sitting in a queue, and it works natively with Cassandra's partition model. The downsides are granularity limited to bucket size (1 min), coordination needed to avoid double-processing, hot buckets needing sub-partitioning, and two writes per message (message + expiry index).

!!! warning "Edge Case"
    **Hot buckets**: If a viral group chat sets 24h TTL and gets 100K messages in one minute, the bucket for T+24h will have 100K entries. Solution: sub-partition by conversation_id or add a random shard key (`expiry_bucket, shard_id, message_id`) and distribute scanner workers across shards.

**Verdict: Use Option C (time-bucketed scanner) as the primary mechanism, with Option A (Cassandra TTL) as a safety net.** The scanner handles side effects (media cleanup, cache invalidation, client notification). Cassandra TTL ensures that even if the scanner misses a message, the data is eventually purged from storage.

### The Deletion Pipeline

Once the scanner identifies expired messages, each deletion triggers a multi-step pipeline:

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

The key design decisions in this pipeline:

- **Tombstones, not hard deletes.** Write a tombstone record (`{ message_id, deleted_at, reason: "ttl_expired" }`) before deleting the content. This allows offline devices to sync the deletion and is essential for consistency.
- **Media deletion is async and batched.** S3 `DeleteObjects` supports batch deletion of up to 1,000 keys. Accumulate media keys and batch-delete every few seconds rather than one-by-one.
- **Cache invalidation is best-effort.** Redis keys may have already expired via their own TTL. Use `DEL` if the key exists, but don't fail the pipeline on a cache miss.
- **Client push is fire-and-forget.** Send the WebSocket event. If the client is offline, the tombstone will be picked up during next sync. Don't block on delivery confirmation.

### "On-Read" TTL: The Hard Variant

When TTL starts on first read (Telegram secret chats, Snapchat), the server doesn't know when to start the timer until the client reports a read event. This creates a two-phase lifecycle:

```
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

In a distributed system, "now" is not a single value. The timing concerns I'd address:

- **Server clock skew** -- different nodes expire the same message at slightly different times. Mitigation: NTP with tight sync (< 1s). Accept +/-30s accuracy for TTLs > 1 minute.
- **Client clock manipulation** -- a malicious client reports a fake read time to delay or accelerate TTL. Mitigation: the server always uses its own clock for `expiry_at`, never trusts client timestamps.
- **Timezone confusion** -- always use UTC epoch milliseconds internally. Never store local time.
- **Leap seconds** -- use TAI or monotonic clocks for timer comparisons. In practice, +/-1s is fine for TTL.

### Cryptographic Erasure: True Deletion

Simply deleting a database row doesn't guarantee the data is gone. It may exist in replication logs (Cassandra commit log, Kafka topics), backup snapshots, the OS file system (deleted but not overwritten), or read replicas with replication lag.

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
    This is an **interview power move**. Mentioning cryptographic erasure demonstrates awareness of the gap between "application-level delete" and "data is truly gone." Most candidates stop at `DELETE FROM messages`. Bringing up crypto erasure shows you think about compliance (GDPR right to erasure), backup recovery scenarios, and defense in depth.

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

## Scalability, Reliability & Edge Cases

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

- **Partition assignment**: Use consistent hashing or simple modulo on `bucket_minute % num_workers`. Workers claim ownership via a distributed lock (ZooKeeper, etcd).
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

### Edge Cases & Decisions

| Scenario | Decision | Reasoning |
|----------|----------|-----------|
| **Message forwarded before expiry** | Forwarded copy gets its own TTL (or none if chat isn't TTL-enabled) | Original TTL only applies to the original conversation |
| **User goes offline before reading TTL message** | "On-read" TTL stays dormant. "On-send" TTL expires regardless | On-send is server-controlled; on-read waits for client event |
| **Backup restoration after expiry** | Cryptographic erasure ensures restored ciphertext is unreadable | Key was deleted before backup restore -- this is the whole point |
| **Group chat TTL with 500 members** | Server deletes once; sends 500 deletion events | Don't delete 500 copies -- there's only one row. Fan out the notification |
| **TTL change mid-flight** | Disallow. TTL is immutable once set | Allowing changes adds complexity (re-scheduling) with minimal user value |
| **Regulatory hold / legal preservation** | Override TTL for flagged accounts, store in separate compliance store | Must be invisible to the user to avoid evidence destruction |

---

## Wrap Up

- **Time-bucketed scanner** as the primary expiration mechanism -- minute-granularity buckets, horizontally scalable workers, idempotent deletion.
- **Cassandra native TTL** as a safety net -- ensures data is eventually purged even if the scanner misses it.
- **Tombstone-based deletion propagation** so offline devices can sync deletions on reconnect.
- **Cryptographic erasure** for true deletion -- delete the key, and the ciphertext becomes irrecoverable across all replicas and backups.
- **"On-read" TTL** requires a two-phase write: store with NULL expiry, set expiry on read receipt with idempotent CAS.
- **Monitor expiration lag** as the critical metric. Messages lingering past TTL is a privacy incident, not just a bug.

**What I'd improve with more time:** configurable TTL policies per organization (enterprise compliance), integration with GDPR right-to-erasure workflows, multi-region deletion coordination with causal ordering, and a TTL visualization dashboard for end users showing exactly when each message will expire.

---

## References

- [Telegram's MTProto Protocol](https://core.telegram.org/mtproto) -- Secret chat encryption and self-destruct implementation
- [Cassandra TTL and Tombstones](https://cassandra.apache.org/doc/latest/cassandra/operating/compaction/) -- How Cassandra handles TTL-based deletion during compaction
- [Signal Protocol: Disappearing Messages](https://signal.org/blog/disappearing-messages/) -- Client-side timer-based deletion design
- [Cryptographic Erasure in Practice](https://www.usenix.org/conference/usenixsecurity19) -- Academic treatment of key-based data destruction
- [Designing Data-Intensive Applications, Ch. 8](https://dataintensive.net/) -- Martin Kleppmann on distributed clocks and timing
- [Building Reliable Reprocessing and Dead Letter Queues with Kafka](https://www.uber.com/blog/reliable-reprocessing/) -- Uber's approach to failure handling in event pipelines
