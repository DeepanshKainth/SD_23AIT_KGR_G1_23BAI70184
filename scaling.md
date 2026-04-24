# Scaling Strategy — Online Coding Judge

The most unique scaling challenge for a coding judge is the **code execution pipeline** —
it involves running arbitrary untrusted code at high concurrency with strict resource
isolation. This document covers how every layer of the system scales.

---

## 1. Load Balancing

### Global Traffic — Latency-Based DNS Routing
Route 53 or Cloudflare resolves `api.codejudge.io` to the nearest active region
(US-East, EU-West, AP-Southeast). During a globally distributed contest with users from
50+ countries, this ensures each user connects to a region within ~50 ms RTT.

**Failover:** If one region becomes unhealthy (health check failure), DNS TTL of 30 seconds
ensures rapid cutover to the next-closest region.

### CDN — Static Content & Problem Statements
Problem statements (markdown/HTML), cover images, and JS bundles are served from CDN edge nodes:

- `Cache-Control: public, max-age=3600` for problem HTML (invalidated on admin edit).
- `Cache-Control: public, max-age=31536000, immutable` for versioned JS/CSS bundles.
- Expected CDN hit rate > 85% for problem page loads. This offloads the origin entirely
  for the most common read workload.

### Application Load Balancer (L7) — Per Region
An AWS ALB or GCP HTTPS Load Balancer in each region:

- Routes `/api/v1/ws*` (WebSocket) to the WebSocket gateway cluster (sticky by connection).
- Routes `/api/v1/*` (REST) to the API Gateway cluster (stateless, round-robin).
- Routes `/health` to health check endpoint; removes unhealthy pods from rotation within 10 s.
- TLS termination at ALB; internal traffic over mTLS via Istio service mesh.

### API Gateway — Rate Limiting as Load Shield
Kong enforces per-user sliding-window counters backed by Redis:

| Tier | API Requests/min | Submissions/min | Run (custom test)/min |
|---|---|---|---|
| Free user | 60 | 5 | 10 |
| Premium user | 300 | 10 | 30 |
| Admin | 1000 | unlimited | unlimited |

Rate limiting at the gateway level prevents any single user from saturating the judge queue
or the database. During contest spikes, this is the first line of defense.

### Kubernetes HPA — Per Microservice

| Service | Scale Trigger | Min Pods | Max Pods |
|---|---|---|---|
| Submission Service | CPU > 60% | 3 | 30 |
| Problem Service | CPU > 60% | 2 | 20 |
| Contest Service | Req/sec > 200/pod | 3 | 30 |
| Search Service | CPU > 70% | 2 | 15 |
| Notification Service | Kafka lag > 1000 | 2 | 10 |
| WebSocket Gateway | Open connections > 5000/pod | 3 | 50 |

### Judge Worker Autoscaling — KEDA
This is the most critical scaling component. KEDA (Kubernetes Event-Driven Autoscaler)
watches the Kafka consumer group lag on the `submission-jobs` topic:

```yaml
# KEDA ScaledObject (simplified)
triggers:
  - type: kafka
    metadata:
      topic: submission-jobs
      consumerGroup: judge-workers-python
      lagThreshold: "10"     # Scale up if lag > 10 per partition
      activationLagThreshold: "1"
minReplicaCount: 5
maxReplicaCount: 300
```

- During idle: 5 workers per language pool (Python, Java, C++, JS, Go = 25 total).
- During a contest with 50,000 participants: scales to 200–300 workers within 90 seconds.
- Scale-down is gradual (cooldown: 5 minutes) to prevent oscillation.

**Why per-language partitioning?**  
Kafka `submission-jobs` is partitioned by language. Each language has its own consumer group
and worker pool. A TLE storm in Python (everyone submitting exponential backtracking solutions)
won't delay Java submissions — the worker pools are independent.

---

## 2. Caching Strategy

### Layer 1 — CDN Cache (Static Content)
Problem descriptions, editorial pages, and user profile images are cached at CDN PoPs.

- Problem HTML: TTL 1 hour; cache-busted via admin invalidation API on edit.
- User avatars: TTL 7 days.
- JS/CSS: immutable with content-hash in filename; CDN caches forever.

### Layer 2 — Redis Application Cache

The platform has two distinct hot paths: **problem reads** (dominant — 90% of traffic)
and **contest leaderboard reads** (bursty — 10,000+ concurrent during contest).

**Problem Metadata Cache**

| Key | Content | TTL | Invalidation |
|---|---|---|---|
| `problem:{slug}` | Full problem object JSON | 1 hour | On admin edit/publish |
| `problem:list:{filters}:{page}` | Paginated problem list | 5 min | On new publish |
| `potd:today` | Problem of the Day ID | Until midnight | Daily cron |
| `editorial:{slug}` | Editorial markdown | 1 hour | On publish |

Cache-aside pattern: check Redis → miss → query PostgreSQL → write Redis with TTL → return.

**Submission Verdict Cache**

When a judge returns a verdict, the Submission Service writes it to both PostgreSQL and Redis
before notifying the client:

```
verdict:{submission_id}  →  JSON verdict object   TTL 300s
```

The client polling `GET /submissions/{id}` hits Redis first. This avoids repeated DB queries
during the ~2-second window clients poll while waiting for a verdict.

**Contest Leaderboard Cache (Critical)**

During a live contest with 10,000+ concurrent viewers refreshing the leaderboard,
querying PostgreSQL for ranked scores on every request would be catastrophic.

Architecture:
1. All accepted submissions emit to Kafka `contest-events`.
2. Contest Service consumer updates a Redis Sorted Set atomically:
   ```
   ZADD leaderboard:{contest_id} <composite_score> <user_id>
   ```
   Composite score = `total_points * 10^9 - penalty_seconds`
   (single ZSET field encodes both score and penalty; ZREVRANK gives correct ordering).
3. Leaderboard page responses are cached:
   ```
   leaderboard:cache:{contest_id}:page:1  →  JSON top-25   TTL 5s
   ```
4. On GET `/contests/{slug}/leaderboard?page=1`, serve from cache (< 1 ms) if hit.
   On miss, ZREVRANGE + user lookups + serialize → store in cache → return.
5. User's own rank served separately via ZREVRANK (not cached; personalized).

**Rate Limiting**

Sliding window counters for submission and API rate limits:

```
rate:submit:{user_id}    →  INT count    TTL 60s   (Lua script for atomic check-and-incr)
rate:api:{user_id}       →  INT count    TTL 60s
```

**Judge Deduplication**

If a Kafka message is redelivered (worker crash after consuming but before committing offset),
the idempotency key prevents re-execution:

```
judge:dedup:{submission_id}   →  "1"   TTL 600s
```

Worker checks this key before starting execution; if it exists, skip and ack the message.

### Layer 3 — In-Process Cache (Problem Service)

Problem Service pods maintain a local **Caffeine LRU cache** of the 1,000 most-accessed
problem objects (hot problems stay in-process memory):

- Cache size: 1,000 entries (~50 MB per pod).
- Hit rate for top-100 problems: ~99% (LeetCode-style platforms have very skewed access —
  "Two Sum" alone receives millions of views).
- This eliminates Redis round-trips for the hottest 0.1% of problems.

### Cache Stampede Prevention

During a contest, when the leaderboard cache TTL expires simultaneously for thousands of
concurrent requests, all miss and hit the DB together (thundering herd).

Mitigation — **Probabilistic Early Expiry (XFetch)**:
```python
# Recompute with probability that increases as expiry approaches
remaining_ttl = redis.ttl(key)
if remaining_ttl < random.uniform(0, MAX_TTL):
    recompute_and_cache()  # Some requests regenerate early
```

For the leaderboard, the mutex lock approach is also applied: first miss acquires a
`SET NX PX 2000 lock:leaderboard:{id}`, recomputes, releases; others spin-wait 2 s and
then read the freshly cached value.

---

## 3. Database Scaling

### PostgreSQL — Primary Transactional Store

**Read replicas:** 2 replicas per region. The Problem Service, Submission Service (history reads),
User Service (profile reads), and Contest Service (past contest reads) all route reads to replicas.
Only writes (new submissions, verdict updates, registrations) go to primary.

```
Primary   ←  All writes
Replica 1 ←  Problem reads, User profile reads
Replica 2 ←  Submission history reads, Contest reads
```

**Connection pooling:** PgBouncer in transaction pooling mode. Each service connects to PgBouncer,
which maintains a fixed pool of 200 connections to Postgres (vs. thousands of raw service connections).
Prevents connection exhaustion during contest spikes.

**Partitioning:** The `submissions` table is range-partitioned by `created_at` (monthly partitions).
Old partitions are detached and moved to cold storage after 12 months. This keeps the active
partition small and index scans fast.

```sql
CREATE TABLE submissions PARTITION BY RANGE (created_at);
CREATE TABLE submissions_2025_07 PARTITION OF submissions
    FOR VALUES FROM ('2025-07-01') TO ('2025-08-01');
```

**Indexing strategy for hot queries:**
- `(user_id, problem_id)` — "have I solved this?" used on every problem page load.
- `(problem_id, created_at DESC)` — recent submissions per problem.
- `(status)` partial index where `status = 'PENDING'` — judge coordinator polls this.
- `(contest_id, user_id)` partial index — contest submission lookups.

**Write amplification control:** Acceptance rate and submission counts on the `problems` table
are updated via a **batch aggregation job** (runs every 5 minutes), not on every submission.
This converts 50,000 individual UPDATE statements/hour to a single batch UPDATE.

**Multi-region:** Active-passive configuration. US-East is primary with synchronous replication
to standby. EU-West and AP-Southeast run read replicas from US-East with async replication
(~20 ms lag acceptable for reads). In case of US-East failure, standby promoted in < 60 s
(RPO < 60 s, RTO < 2 min).

---

### Elasticsearch — Search Index

**Cluster:** 9 nodes (3 master-eligible, 6 data nodes).

**Index design:**

```json
{
  "index": "problems",
  "settings": {
    "number_of_shards": 6,
    "number_of_replicas": 1,
    "refresh_interval": "30s"
  },
  "mappings": {
    "properties": {
      "title":       { "type": "text", "analyzer": "standard" },
      "description": { "type": "text", "analyzer": "standard" },
      "tags":        { "type": "keyword" },
      "difficulty":  { "type": "keyword" },
      "acceptance_rate": { "type": "float" }
    }
  }
}
```

- 6 primary shards (distributed across 6 data nodes for parallel search).
- 1 replica per shard (12 total; allows any data node to fail without data loss).
- `refresh_interval: 30s` — new/edited problems appear in search within 30 s (not real-time),
  reducing segment merge overhead significantly vs. the default 1 s.

**Scaling writes:** Problem catalog is small (< 10,000 problems). Updates happen infrequently
(admin edits). Bulk indexing via a Kafka consumer (`search-index-updates` topic) batches
changes and sends to Elasticsearch in 100-doc batches.

**Autocomplete:** A separate `problems_suggest` index with `completion` field type provides
< 10 ms autocomplete responses from Elasticsearch's FST-backed suggest engine.

---

### Redis Cluster — Cache & Real-Time State

Redis Cluster with 6 nodes (3 primary + 3 replicas), hash-slot sharding:

- Node 0: slots 0–5460 (leaderboard ZSETs, rate limits)
- Node 1: slots 5461–10922 (problem cache, verdict cache)
- Node 2: slots 10923–16383 (sessions, dedup keys)

**Memory sizing:** Each leaderboard ZSET for a 50,000-participant contest uses ~6 MB. With
up to 10 concurrent contests, leaderboard state = ~60 MB. Problem cache for 3,000 problems
≈ 150 MB. Total cluster memory: 32 GB across 3 primaries — well within capacity.

**Eviction policy:** `allkeys-lru`. When memory pressure is high, least-recently-used keys
are evicted. Leaderboard ZSETs are protected by keeping TTL reset on every update; old
problem cache entries evict first.

---

### Object Storage (S3) — Test Cases & User Code

- Test case input/output files stored in S3 (`codejudge-testcases/{problem_id}/{n}.in`).
- User-submitted code stored in S3 (`codejudge-submissions/{year}/{month}/{submission_id}`).
- Judge workers fetch test cases from S3 on first run; **EFS (Elastic File System) cache**
  mounts the test cases directory on each worker node. Popular problem test cases stay
  in the EFS cache, reducing S3 GET requests from 100% to ~15% for hot problems.
- S3 Transfer Acceleration for multi-region judge worker access.

---

## 4. Judge-Specific Scaling

The code execution pipeline is unlike any other web service component. Key scaling tactics:

### Sandbox Resource Isolation
Each judge container enforces hard limits at the kernel level:

```bash
docker run \
  --network none \
  --read-only \
  --tmpfs /tmp:size=64m \
  --memory="256m" --memory-swap="256m" \
  --cpus="1" \
  --pids-limit 64 \
  --security-opt seccomp=judge-seccomp.json \
  judge-runner-python3:latest \
  /runner/run.sh
```

The `seccomp` profile allowlists only the syscalls needed for execution (read, write, open,
close, mmap, brk, exit) and blocks: `fork`, `clone`, `execve` of non-whitelisted binaries,
`socket`, `bind`, `connect`, `ptrace`.

### Execution Node Sizing
Judge worker nodes run on **compute-optimized** instances (c6i.4xlarge: 16 vCPU, 32 GB RAM):

- Each node runs up to 8 concurrent sandbox containers (leaving 2 vCPUs for node overhead).
- With 300 pods at peak, theoretical throughput = 300 × 1 submission/2s = **150 submissions/sec**
  = **540,000 submissions/hour** — well above the 50,000/hour peak requirement.

### Warm Sandbox Pool
Container startup takes ~500 ms (pulling image, setting up cgroups). For popular languages,
each judge node pre-creates a pool of 4 warm but idle containers per language:

- New job arrives → claim a warm container → inject code → run → return container to pool.
- Reduces per-submission startup from 500 ms to < 10 ms.
- Container is sanitized (tmpfs wiped) before returning to pool.

---

## 5. Scaling Summary

| Component | Peak Load | Scaling Mechanism | Target Throughput |
|---|---|---|---|
| API Gateway | 50,000 req/min | HPA on CPU | 833 req/sec |
| Submission Service | 50,000 submissions/hr | HPA on CPU + rate limiting | 14 submissions/sec |
| Judge Workers | 50,000 submissions/hr | KEDA on Kafka lag | 150 submissions/sec (headroom 10×) |
| PostgreSQL | 5,000 writes/min | Primary + 2 replicas + PgBouncer | 83 writes/sec |
| Redis Cluster | 500,000 ops/min | 3-node cluster | 8,333 ops/sec |
| Elasticsearch | 5,000 search req/min | 6 shards across 6 nodes | 83 req/sec |
| Contest Leaderboard | 10,000 viewers/contest | Redis ZSET + 5s page cache | < 1 ms read |
| WebSocket (contest) | 50,000 connections | HPA (5,000 conns/pod × 10 pods) | 50,000 concurrent |

---

*Last updated: System Design Case Study v1.0*
