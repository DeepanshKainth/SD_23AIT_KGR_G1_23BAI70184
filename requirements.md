# Requirements — Online Coding Judge (LeetCode / HackerRank Style)

---

## 1. Functional Requirements

### Problem Management
- Admins can create, update, delete, and publish coding problems.
- Each problem has: title, description, difficulty (Easy / Medium / Hard), tags/topics,
  constraints, sample input/output, hidden test cases, editorial, and hints.
- Problems support multiple programming languages (Python, Java, C++, JavaScript, Go, etc.).
- Problems can be organized into topic tags (Arrays, DP, Graphs, etc.) and curated lists.

### User Management
- Users can register, log in, and manage their profile (avatar, bio, social links).
- OAuth 2.0 login via Google and GitHub.
- Users have a public profile showing solved problems, submission history, rating, and badges.
- Role-based access: Guest (view only), User (submit), Admin (manage problems), Contest Admin.

### Code Submission & Judging
- Users can write code in an in-browser code editor with syntax highlighting.
- Submit code for a problem; the system executes it against all hidden test cases.
- Support "Run" (against custom or sample test cases) vs. "Submit" (all hidden test cases).
- Return verdict: Accepted (AC), Wrong Answer (WA), Time Limit Exceeded (TLE),
  Memory Limit Exceeded (MLE), Runtime Error (RE), Compilation Error (CE).
- Each problem defines per-language time limits and memory limits.
- Execution is fully sandboxed — user code cannot access the network, filesystem, or system calls.
- Execution results include: verdict, runtime (ms), memory used (MB), and failing test case
  (for WA; partial disclosure only).

### Submission History
- Users can view their past submissions for each problem (code, verdict, runtime, memory, timestamp).
- Filter submissions by language, verdict, and date range.
- Users can view their own code; cannot view others' accepted code unless editorial is published.

### Contest Module
- Admins can create timed contests (ICPC-style or LeetCode Weekly style).
- Contests have: start time, end time, duration, problem set, scoring rules, and registration.
- Real-time leaderboard updates during the contest (rank by score, then by penalty time).
- Users can register for upcoming contests and receive reminders.
- After the contest ends, problems are unlocked for practice.
- Rating system: ELO-based rating updates after each contest.

### Discussion & Editorial
- Each problem has a discussion forum with threaded comments and upvotes.
- Admins can publish an official editorial with solution walkthrough and code.
- Users can post solution approaches (after they have an accepted submission).

### Leaderboard & Stats
- Global leaderboard by rating and by number of problems solved.
- Per-problem statistics: acceptance rate, submission count, difficulty distribution of solvers.
- Per-user statistics: heatmap of daily activity, topics mastered, submission calendar.

### Search & Discovery
- Full-text search over problems by title, tags, and topic.
- Filter problems by difficulty, tag, status (solved / attempted / unsolved), and company tag.
- "Problem of the Day" — one featured problem per day.

### Notifications
- Email and in-app notifications for: contest reminders, rating updates, editorial publication,
  and reply to discussion posts.

---

## 2. Non-Functional Requirements

### Performance
- Code execution result returned in **< 5 seconds** for 95% of submissions (p95).
- Problem page load time **< 200 ms** (p95), excluding code execution.
- Leaderboard refresh latency **< 2 seconds** during active contests.
- Search results returned in **< 100 ms** (p95).

### Availability
- **99.9% uptime** for the web platform (problem browsing, submission history, profiles).
- **99.5% uptime** for the code execution pipeline (degraded gracefully — queue when overloaded).
- Graceful degradation: if the judge is overloaded, queue the submission and notify user.

### Scalability
- Support **5 million+ registered users** and **500,000+ Daily Active Users (DAU)**.
- Handle **50,000+ code submissions per hour** at peak (e.g., during a major contest).
- Support **10,000+ concurrent users** during a live contest leaderboard.
- Problem catalog of 3,000+ problems, each with 10–200 hidden test cases.

### Security
- All user code runs in an **isolated sandbox** (separate container/VM per submission).
  - No network access, no filesystem writes outside a temp dir, no fork bombs.
  - CPU and memory hard limits enforced at the kernel level (cgroups / seccomp).
- All traffic encrypted in transit via **TLS 1.3**.
- CSRF protection on all state-changing endpoints.
- Rate limiting: max 10 submissions/minute per user to prevent abuse.
- SQL injection and XSS prevention via parameterized queries and content security policies.

### Reliability
- Submissions must never be lost — persisted to the DB before dispatching to judge workers.
- Idempotent submission IDs so retries do not create duplicate submissions.
- Worker crashes do not lose submissions; message queue provides at-least-once delivery
  with exactly-once semantics via idempotency keys.

### Fairness & Correctness
- All executions for the same problem are run on identical hardware specs.
- Time measurements exclude container startup overhead (measured inside the sandbox).
- Test case outputs compared with configurable whitespace normalization and
  floating-point tolerance.

### Maintainability
- Microservices architecture; judge workers independently deployable and scalable.
- All services emit structured logs, metrics (Prometheus), and distributed traces (Jaeger).
- Judge workers are stateless; new language support added by deploying a new worker image.

### Compliance
- User code is access-controlled and not retained longer than necessary.
- GDPR: right to erasure removes personal data but preserves anonymized aggregate stats.

---

*Last updated: System Design Case Study v1.0*
