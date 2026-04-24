# High-Level Design — Online Coding Judge

---

## 1. System Architecture Diagram

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                               CLIENT LAYER                                       │
│         Web Browser (React + Monaco Editor)     Mobile (PWA / App)               │
└──────────────────────────────────┬───────────────────────────────────────────────┘
                                   │ HTTPS / WebSocket
┌──────────────────────────────────▼───────────────────────────────────────────────┐
│                            CDN / EDGE LAYER                                      │
│              CloudFront / Cloudflare — Static Assets, Problem Images             │
└───────────────────┬──────────────────────────────────────────────────────────────┘
                    │
┌───────────────────▼──────────────────────────────────────────────────────────────┐
│                     API GATEWAY (Kong / AWS API Gateway)                         │
│          TLS Termination · JWT Auth · Rate Limiting · Request Routing            │
└───────┬──────────────────────────────────────────────────────────────────────────┘
        │
┌───────▼───────────────────────────────────────────────────────────────────────────────────────┐
│                              MICROSERVICES LAYER                                              │
│                                                                                               │
│  ┌───────────────┐  ┌───────────────┐  ┌─────────────────┐  ┌──────────────────────────────┐ │
│  │  User Service │  │Problem Service│  │Submission Service│  │    Contest Service           │ │
│  │  Auth/OAuth   │  │CRUD, test     │  │Accept, validate, │  │  Create, register, score,   │ │
│  │  Profile      │  │cases, tags    │  │enqueue, poll     │  │  real-time leaderboard      │ │
│  └──────┬────────┘  └──────┬────────┘  └────────┬─────────┘  └─────────────┬────────────────┘ │
│         │                  │                     │                           │                 │
│  ┌───────────────┐  ┌───────────────┐  ┌─────────────────┐  ┌──────────────────────────────┐ │
│  │Search Service │  │Discussion Svc │  │ Notification Svc │  │   Analytics / Stats Service  │ │
│  │Elasticsearch  │  │Forum, threads,│  │  Email, in-app   │  │   Heatmaps, global stats     │ │
│  │Autocomplete   │  │upvotes        │  │  push            │  │                              │ │
│  └───────────────┘  └───────────────┘  └─────────────────┘  └──────────────────────────────┘ │
└───────────────────────────────────────────────────────────────────────────────────────────────┘
        │                                          │
        │ Enqueue Submission Job                   │ Result / Verdict
┌───────▼──────────────────────────────────────────▼──────────────────────────────────────────┐
│                           ASYNC JOB PIPELINE                                                 │
│                                                                                              │
│   ┌────────────────────────────────────────────────────────────────────────────────────┐    │
│   │                      Message Queue — Apache Kafka                                  │    │
│   │  Topics:                                                                           │    │
│   │    submission-jobs     (new code submissions, partitioned by language)             │    │
│   │    judge-results       (verdicts back from judge workers)                         │    │
│   │    contest-events      (score updates, registration events)                       │    │
│   │    notification-events (email/push triggers)                                      │    │
│   └─────────────────────────────────────────────┬──────────────────────────────────────┘    │
│                                                  │                                           │
│   ┌──────────────────────────────────────────────▼──────────────────────────────────────┐   │
│   │                         JUDGE WORKER POOL                                           │   │
│   │                                                                                     │   │
│   │  Worker (Python)  Worker (Java)  Worker (C++)  Worker (JS/Node)  Worker (Go) ...   │   │
│   │                                                                                     │   │
│   │  Each worker:                                                                       │   │
│   │    1. Pulls job from Kafka partition                                                │   │
│   │    2. Spins up isolated execution sandbox (Docker + seccomp + cgroups)             │   │
│   │    3. Compiles (if needed) and runs code against each test case                    │   │
│   │    4. Collects verdict, runtime, memory                                            │   │
│   │    5. Publishes result to judge-results topic                                      │   │
│   │    6. Destroys sandbox                                                             │   │
│   │                                                                                     │   │
│   │  Autoscaled via KEDA on Kafka consumer lag metric                                  │   │
│   └─────────────────────────────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────────────────────────────────┘
        │
┌───────▼───────────────────────────────────────────────────────────────────────────────────────┐
│                                DATA & STORAGE LAYER                                           │
│                                                                                               │
│  ┌────────────────────┐  ┌──────────────────────┐  ┌───────────────┐  ┌──────────────────┐   │
│  │   PostgreSQL        │  │   Redis Cluster      │  │Elasticsearch  │  │  Object Storage  │   │
│  │  (Primary store:    │  │  Sessions, verdict   │  │  Problem      │  │  AWS S3 / GCS    │   │
│  │   users, problems,  │  │  poll cache,         │  │  full-text    │  │  Test cases      │   │
│  │   submissions,      │  │  leaderboard ZSET,   │  │  search,      │  │  User code       │   │
│  │   contests,         │  │  rate limiting,      │  │  autocomplete │  │  (encrypted)     │   │
│  │   discussions)      │  │  job dedup keys      │  │               │  │                  │   │
│  │  [Read replicas]    │  │                      │  │               │  │                  │   │
│  └────────────────────┘  └──────────────────────┘  └───────────────┘  └──────────────────┘   │
└───────────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Components

### 2.1 Client Layer
- **Web app** built with React + Monaco Editor (same editor as VS Code) for in-browser code editing
  with syntax highlighting, auto-indent, and keybindings per language.
- Real-time submission result polling via **WebSocket** (or long-polling fallback) — client
  opens a channel when it submits and receives the verdict when the judge finishes.

### 2.2 CDN / Edge Layer
- Serves static assets (JS bundles, CSS, problem images) from edge PoPs.
- Reduces load on origin servers by ~80% for static content.
- Problem statement HTML is cached at the CDN with `Cache-Control: public, max-age=3600`,
  invalidated on admin publish/edit.

### 2.3 API Gateway
- **Kong Gateway** handles: TLS termination, JWT validation, per-user rate limiting
  (sliding window, 10 submissions/min, 300 API requests/min), and routing to the correct
  microservice based on URL prefix.
- WebSocket upgrade for real-time verdict delivery is proxied through the gateway.

### 2.4 Microservices

| Service | Responsibility | Primary Data Store |
|---|---|---|
| **User Service** | Registration, OAuth, profile, RBAC | PostgreSQL |
| **Auth Service** | JWT issuance, refresh, session | PostgreSQL + Redis |
| **Problem Service** | Problem CRUD, test case management, tags | PostgreSQL + S3 |
| **Submission Service** | Accept submissions, enqueue, poll verdict | PostgreSQL + Redis + Kafka |
| **Judge Coordinator** | Route jobs to correct worker pool, track in-flight jobs | Kafka + Redis |
| **Contest Service** | Create/manage contests, scoring, leaderboard | PostgreSQL + Redis |
| **Discussion Service** | Threaded forum, upvotes, editorial | PostgreSQL |
| **Search Service** | Full-text problem search, autocomplete | Elasticsearch |
| **Notification Service** | Email (SES/SendGrid), push notifications | Kafka consumer |
| **Analytics Service** | User heatmaps, acceptance stats, admin dashboards | PostgreSQL + Data Warehouse |

### 2.5 Async Job Pipeline — Apache Kafka

The most critical design decision: **code execution is fully asynchronous**. A synchronous HTTP
call to a judge would block server threads for up to 5+ seconds per submission.

Flow:
```
User submits code
    → Submission Service writes submission record (status=PENDING) to PostgreSQL
    → Enqueues job message to Kafka topic: submission-jobs
    → Returns submission_id to client immediately (HTTP 202 Accepted)

Client polls GET /submissions/{id} or listens on WebSocket

Judge Worker consumes job
    → Runs code in sandbox
    → Publishes verdict to judge-results topic

Submission Service consumes judge-results
    → Updates submission record (status=ACCEPTED/WA/TLE/...)
    → Notifies client via WebSocket push
```

Kafka partitioning: `submission-jobs` is partitioned by **language** (e.g., partition 0 = Python,
partition 1 = Java). This allows language-specific worker pools to scale independently —
Python submissions won't be delayed by a Java backlog.

### 2.6 Judge Worker Pool

Each worker is a **Kubernetes pod** that:
1. Consumes a job from Kafka (one at a time per pod to avoid resource contention).
2. Creates an isolated Docker container with:
   - `--network none` (no network)
   - `--read-only` filesystem with tmpfs for `/tmp` only
   - `--memory` and `--cpus` flags matching the problem's limits
   - `seccomp` profile blocking dangerous syscalls (fork, execve of system binaries, etc.)
3. Compiles the code (C++/Java) inside the container.
4. Runs the binary/interpreter against each test case sequentially; stops on first failure.
5. Reports results to Kafka.
6. Destroys the container (`docker rm -f`).

**Autoscaling:** KEDA (Kubernetes Event-Driven Autoscaler) monitors the Kafka consumer group lag
on `submission-jobs` and scales worker pods up/down to keep lag < 50 messages. During a contest
spike, this can grow from 10 to 200 workers in ~60 seconds.

### 2.7 Data Layer

| Store | Use Case | Justification |
|---|---|---|
| **PostgreSQL** | All primary transactional data | ACID, relational, foreign keys, strong consistency |
| **Redis Cluster** | Sessions, verdict cache, leaderboard, rate limits | Sub-ms reads, sorted sets for leaderboard, TTL |
| **Elasticsearch** | Problem full-text search, autocomplete | Inverted index, BM25 relevance ranking |
| **S3 / GCS** | Test case files, user code archives | Durable, cheap, decoupled from DB |
| **Data Warehouse** | Acceptance rates, usage analytics, reports | OLAP queries, columnar storage |

### 2.8 Real-Time Leaderboard (Contest)

During active contests, the leaderboard must update in near real-time for up to 50,000 participants.

- Verdict updates → Kafka `contest-events` topic → Contest Service consumer.
- Contest Service updates a **Redis Sorted Set** (`leaderboard:{contest_id}`) where:
  - Score = `total_points * 10^9 - penalty_seconds` (allows single-field sort for both score
    and penalty simultaneously).
- Client polls `GET /contests/{id}/leaderboard?page=1` every 30 seconds.
- Top 25 leaderboard is cached in Redis with 5-second TTL; served from cache without DB query.

---

*Last updated: System Design Case Study v1.0*
