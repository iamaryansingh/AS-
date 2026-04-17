# ChatFlow — Performance Optimization
### CS6650 Assignment 4 ·

---

## 1. Architecture Selection

In Assignment 3 implementation sustained **11,505 msg/s** WebSocket throughput with **0% message loss** across all three test scenarios (500K baseline, 1M stress, 21M endurance). The 1M stress test exposed two structural bottlenecks: **RabbitMQ queue backlog reaching 277K messages** as the consumer could only drain at **1,500 msg/s** while the client sent at 9,800–18,955 msg/s, and **flat table B-tree index growth** slowing writes as row count climbed past 655K. Both bottlenecks map directly to the two Assignment 4 optimizations — replacing RabbitMQ with Redis Streams, and converting the flat table to a time-partitioned schema.

---

## 2. Where We Started — Assignment 3

**Pipeline (Assignment 3):**

![Assignment 3 Architecture](./images/architecture-v1.png)

```
Clients → WebSocket Server → RabbitMQ Queue → Consumer → PostgreSQL (flat table)
```

**Problems under load:**
- RabbitMQ AMQP overhead — queue backlog hit **277K messages** at peak; consumer drained at only **1,500 msg/s** while client sent at 11,505 msg/s
- Flat `messages` table — 4 composite B-tree indexes updated on every batch insert; write rate degraded as row count grew past 655K
- No fan-out primitive — broadcasting to rooms required custom exchange/binding; no built-in crash recovery
- Extra infrastructure — RabbitMQ as a third process alongside PostgreSQL, each consuming memory under load

**Assignment 3 Performance (from load test client):**

| Test Scenario | WS Throughput | DB Write Rate | Message Loss |
|---------------|--------------|---------------|--------------|
| Baseline — 500K messages | **11,505 msg/s** | **1,520 msg/s** | **0%** |
| Stress — 1M messages | **9,800 msg/s** | **1,380 msg/s** | **0%** |
| Endurance — 21M messages | **18,955 msg/s** | **1,640 msg/s** | **0.01%** |

> **Note:** A3 used a custom Java load test client. JMeter was introduced in Assignment 4. The two tools are complementary but not directly comparable on response time.

---

## 3. Optimized Architecture

![Optimized ChatFlow Architecture](./images/architecture-v4.png)

**Message flow — two parallel paths from Redis:**

| Path | Mechanism | Purpose |
|------|-----------|---------|
| Real-time fan-out | `PUBLISH room:{roomId}` | Instant broadcast to all server instances → local WS clients |
| Persistence | `XADD room-stream:{roomId}` | Durable queue → Consumer → PostgreSQL |

**System at a glance:**

| Component | Technology | Scale |
|-----------|-----------|-------|
| Load balancer | AWS ALB | Routes across 2 server instances |
| WebSocket server | Java-WebSocket | 2 × instances, port 8080 |
| Message buffer | LinkedBlockingQueue | 1,000,000 capacity · non-blocking |
| Pub/Sub fan-out | Redis 6+ | 4 subscriber threads · 5 rooms each |
| Persistence queue | Redis Streams (AOF) | 20 consumer threads · 1 per room |
| Batch writer | HikariCP + JDBC | 5 threads · batch=500 |
| Database | PostgreSQL 13+ | Time-partitioned · quarterly |
| Metrics cache | Redis | 10s TTL · shared across servers |

---

## 4. Optimization 1 — Redis Pub/Sub + Streams

**Replaced:** RabbitMQ → **Redis Pub/Sub + Streams**

**Why Redis wins here:**
- Already in the stack — zero new infrastructure
- `PUBLISH` gives instant fan-out to all server instances with no configuration
- `XADD` + consumer groups give at-least-once delivery with built-in crash recovery (`XACK` deferred until safe)
- Pipeline batching: **200 messages per Redis round-trip** vs 1 per RabbitMQ AMQP call
- AOF persistence on disk — no heap OOM risk under backpressure

**How it works:**
- `ChatServer.onMessage()` enqueues to `publishBuffer` (non-blocking, O(1)) — never touches Redis directly
- 4 `RedisPublisherWorker` threads drain buffer in batches of 200 using Jedis pipelines
- On Redis side: **PUBLISH** fans out instantly; **XADD** streams to consumer for DB persistence
- Circuit breaker: 5 consecutive failures → open 10s

**Tradeoffs:**

| Benefit | Cost |
|---------|------|
| No new infrastructure | AOF required for durability (everysec) |
| Per-room parallel consumption | Pub/Sub is fire-and-forget — no delivery guarantee |
| Batching cuts Redis round-trips by 200× | More complex consumer logic (XACK, pending recovery) |

**Impact:**

| Metric | Before (A3) | After (A4) | Improvement |
|--------|-------------|-----------|-------------|
| RabbitMQ queue backlog | **277K messages** at peak | **0** — stream uses AOF disk | Backpressure eliminated |
| Message Throughput (WS) | **11,505 msg/s** | **31,546 msg/s** confirmed round-trip | **+174%** |
| DB Write Rate | **1,520 msg/s** | **13,941 msg/s** | **+9.3×** |

---

## 5. Optimization 2 — PostgreSQL Time-Partitioned Table

**Replaced:** Single flat table → **Range-partitioned table (quarterly)**

**Why partitioning:**
- Flat table: index depth grows with every insert → inserts get slower as table grows
- Partitioned table: new inserts always land in the **current (small) partition** — index depth stays constant
- Query planner prunes irrelevant partitions — range scans only touch relevant quarterly slice
- Old partitions can be dropped instantly (no `DELETE` + reindex)

**What we built:**
- Parent table `messages` partitioned `BY RANGE (timestamp)`
- Quarterly partitions: 2026-Q1 → 2027-Q1 (auto-propagate indexes)
- **3 composite indexes** on parent (propagated to all partitions):
  - `(room_id, timestamp)` — room history queries
  - `(user_id, timestamp)` — user activity queries
  - `(timestamp)` — active user count window
- `ON CONFLICT (message_id, timestamp) DO NOTHING` — idempotent inserts
- LRU dedup cache (1,000 entries/room) in consumer — prevents most duplicates reaching DB

**Tradeoffs:**

| Benefit | Cost |
|---------|------|
| Index depth constant regardless of total rows | Partition key must be in primary key |
| Queries touch only relevant partition | Cross-partition queries scan multiple tables |
| Old data dropped in O(1) — drop partition | Manual partition creation per quarter |

**Impact:**

| Metric | Before (A3) | After (A4) | Improvement |
|--------|-------------|-----------|-------------|
| Avg Insert Throughput | **1,520 msg/s** (flat table, 5 writers) | **13,941 msg/s** | **+9.3×** |
| Index maintenance | 4 indexes updated per batch on growing flat table | Indexes stay shallow — current partition only | No write degradation under load |
| DB Write Error Rate | **0%** | **0%** | Maintained |

---

## 6. Load Test Results

### Before vs After — Baseline (500K messages)

| Metric | A3 | A4 | Improvement |
|--------|----|----|-------------|
| WS Throughput | **11,505 msg/s** | **23,345 msg/s** | **+103%** |
| DB Write Rate | **1,520 msg/s** | **11,605 msg/s** | **+7.6×** |

### Before vs After — Stress (1M messages)

| Metric | A3 | A4 | Improvement |
|--------|----|----|-------------|
| WS Throughput | **9,800 msg/s** | **31,546 msg/s** | **+222%** |
| DB Write Rate | **1,380 msg/s** | **13,941 msg/s** | **+10.1×** |

### Baseline (500K) vs Stress (1M) — A4

| Metric | Baseline — 500K | Stress — 1M |
|--------|----------------|-------------|
| WS Round-trip Throughput | **23,345 msg/s** | **31,546 msg/s** |
| DB Write Throughput | **11,605 msg/s** | **13,941 msg/s** |
| Failed Messages | **0 (0%)** | **0 (0%)** |
| p50 Round-trip | **9,800 ms** | **13,491 ms** |
| p99 Round-trip | **19,229 ms** | **28,234 ms** |
| DB Catchup Time | **40.14 s** | **104.58 s** |

### Baseline Results (A4 JMeter — 1,000 users, 100K calls)

| Transaction | Samples | Error Rate | Mean | p95 | p99 |
|-------------|---------|------------|------|-----|-----|
| WebSocket Open | 51,277 | **0%** | 507 ms | 810 ms | 838 ms |
| WebSocket Write | 51,288 | **0.58%** | 0.05 ms | 0 ms | 1 ms |
| GET /metrics | 14,643 | 9.2% | 459 ms | 798 ms | 1,046 ms |
| **Total** | **117,208** | **1.4%** | **279 ms** | **780 ms** | **835 ms** |

![Baseline JMeter Report](./images/baseline-summary-report.png)

![Baseline Response Time Graph](./images/baseline-response-time-graph.png)

### Stress Results (A4 JMeter — 500 users, 30 min)

| Transaction | Samples | Error Rate | Mean | p95 | p99 |
|-------------|---------|------------|------|-----|-----|
| WebSocket Open | 83,731 | **0%** | 393 ms | 694 ms | 865 ms |
| WebSocket Write | 83,731 | **0.18%** | 0.05 ms | 0 ms | 1 ms |
| GET /metrics | 244,786 | 19.7% | 313 ms | 809 ms | 920 ms |
| **Total** | **412,248** | **11.7%** | **266 ms** | **640 ms** | **913 ms** |

![Stress Test Report](./images/stress-summary-report.png)

![Stress Active Threads](./images/stress-active-threads.png)

> **Note on HTTP error rate:** The 9.2–19.7% errors are on the `/metrics` analytics endpoint, not the chat pipeline. WebSocket writes hold at **0.18–0.58%** throughout. The HTTP errors come from the PostgreSQL read pool (5 connections) saturating under concurrent analytics queries, addressed in Future Optimizations.

---

## 7. Future Optimizations

| # | Optimization | Impact | Complexity |
|---|-------------|--------|------------|
| 1 | **Multiple Consumer Instances** — A3's primary bottleneck was single consumer at 1,500 msg/s | Scales DB write rate linearly | Low |
| 2 | **PostgreSQL Read Replica** — route `/metrics` off primary | Fixes 9–19% HTTP error rate | Medium |
| 3 | **Materialized View** — pre-compute analytics, refresh every 30s | `/metrics` latency: 450ms → <10ms | Low |
| 4 | **Adaptive Batch Sizing** — scale `batchSize` with `writeBuffer` depth | +20–30% write throughput under burst | Low |
| 5 | **Automated Partition Management** (`pg_partman`) | Prevents outage when partitions run out | Low |

---

## Appendix — System Configuration

### Assignment 3 (Baseline)

| Parameter | Value |
|-----------|-------|
| Consumer threads (RabbitMQ pull) | 10–20 threads |
| DB writer threads | 5–10 threads |
| Batch size / flush interval | 1,000 messages / 500 ms |
| Message buffer | ConcurrentLinkedQueue (unbounded) |
| DB connection pool | HikariCP, max 50 |
| DB write rate (sustained) | 1,500 msg/s |

### Assignment 4 (Optimized)

| Parameter | Value |
|-----------|-------|
| WebSocket server instances | 2 (behind AWS ALB) |
| publishBuffer capacity | 1,000,000 messages |
| Publisher threads / batch size | 4 threads · 200 msg/batch |
| Redis Pub/Sub subscriber threads | 4 threads · 5 rooms each |
| Redis Stream consumer threads | 20 threads · 1 per room |
| writeBuffer capacity | 100,000 messages |
| DB batch writer threads | 5 threads · 500 msg/batch |
| DB flush interval | 500 ms |
| Redis pool size (publish) | 100 connections |
| DB connection pool | 5 connections (HikariCP) |
| Redis cache TTL | 10 seconds |
| Redis persistence | AOF · everysec (≤1s loss) |
| DB partitioning | Quarterly · 2026-Q1 → 2027-Q1 |
