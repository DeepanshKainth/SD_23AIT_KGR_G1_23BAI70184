# Low-Level Design — Online Coding Judge

---

## 1. Database Schema (PostgreSQL)

```sql
-- ─────────────────────────────────────────────
-- USERS & AUTH
-- ─────────────────────────────────────────────

CREATE TABLE users (
    user_id        UUID          PRIMARY KEY DEFAULT gen_random_uuid(),
    username       VARCHAR(32)   NOT NULL UNIQUE,
    email          VARCHAR(255)  NOT NULL UNIQUE,
    password_hash  VARCHAR(255),                      -- NULL for OAuth-only
    display_name   VARCHAR(64),
    avatar_url     TEXT,
    bio            TEXT,
    role           VARCHAR(16)   NOT NULL DEFAULT 'user',  -- 'user','admin','contest_admin'
    rating         INT           NOT NULL DEFAULT 1500,    -- ELO-based contest rating
    problems_solved INT          NOT NULL DEFAULT 0,
    country_code   CHAR(2),
    github_url     TEXT,
    linkedin_url   TEXT,
    is_verified    BOOLEAN       NOT NULL DEFAULT FALSE,
    is_banned      BOOLEAN       NOT NULL DEFAULT FALSE,
    created_at     TIMESTAMPTZ   NOT NULL DEFAULT NOW(),
    last_login_at  TIMESTAMPTZ
);

CREATE INDEX idx_users_rating ON users(rating DESC);
CREATE INDEX idx_users_solved ON users(problems_solved DESC);

CREATE TABLE oauth_accounts (
    id             BIGSERIAL     PRIMARY KEY,
    user_id        UUID          NOT NULL REFERENCES users(user_id) ON DELETE CASCADE,
    provider       VARCHAR(16)   NOT NULL,             -- 'google', 'github'
    provider_uid   VARCHAR(255)  NOT NULL,
    UNIQUE (provider, provider_uid)
);

CREATE TABLE refresh_tokens (
    token_hash     VARCHAR(64)   PRIMARY KEY,
    user_id        UUID          NOT NULL REFERENCES users(user_id) ON DELETE CASCADE,
    device_id      VARCHAR(128),
    expires_at     TIMESTAMPTZ   NOT NULL,
    revoked        BOOLEAN       NOT NULL DEFAULT FALSE,
    created_at     TIMESTAMPTZ   NOT NULL DEFAULT NOW()
);

-- ─────────────────────────────────────────────
-- PROBLEMS & TAGS
-- ─────────────────────────────────────────────

CREATE TABLE tags (
    tag_id         SMALLSERIAL   PRIMARY KEY,
    slug           VARCHAR(64)   NOT NULL UNIQUE,      -- 'dynamic-programming', 'graph'
    display_name   VARCHAR(64)   NOT NULL,
    category       VARCHAR(32)               -- 'algorithm', 'data-structure', 'company'
);

CREATE TABLE problems (
    problem_id     UUID          PRIMARY KEY DEFAULT gen_random_uuid(),
    slug           VARCHAR(128)  NOT NULL UNIQUE,      -- 'two-sum', 'longest-palindrome'
    title          VARCHAR(255)  NOT NULL,
    description    TEXT          NOT NULL,             -- Markdown/HTML
    difficulty     VARCHAR(8)    NOT NULL,             -- 'easy', 'medium', 'hard'
    constraints    TEXT,
    hints          TEXT[],                             -- Array of hint strings
    editorial_id   UUID,                               -- FK to editorials table
    is_published   BOOLEAN       NOT NULL DEFAULT FALSE,
    is_premium     BOOLEAN       NOT NULL DEFAULT FALSE,
    acceptance_rate NUMERIC(5,2) NOT NULL DEFAULT 0.00,
    total_submissions BIGINT     NOT NULL DEFAULT 0,
    total_accepted  BIGINT       NOT NULL DEFAULT 0,
    created_by     UUID          REFERENCES users(user_id),
    created_at     TIMESTAMPTZ   NOT NULL DEFAULT NOW(),
    updated_at     TIMESTAMPTZ   NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_problems_difficulty ON problems(difficulty) WHERE is_published = TRUE;
CREATE INDEX idx_problems_acceptance ON problems(acceptance_rate DESC) WHERE is_published = TRUE;

CREATE TABLE problem_tags (
    problem_id     UUID          NOT NULL REFERENCES problems(problem_id) ON DELETE CASCADE,
    tag_id         SMALLINT      NOT NULL REFERENCES tags(tag_id),
    PRIMARY KEY (problem_id, tag_id)
);

-- Per-language metadata: time limit, memory limit, starter code template
CREATE TABLE problem_languages (
    problem_id     UUID          NOT NULL REFERENCES problems(problem_id) ON DELETE CASCADE,
    language       VARCHAR(16)   NOT NULL,             -- 'python3', 'java', 'cpp', 'javascript', 'go'
    time_limit_ms  INT           NOT NULL DEFAULT 1000,
    memory_limit_mb INT          NOT NULL DEFAULT 256,
    starter_code   TEXT          NOT NULL,             -- Skeleton shown in editor
    solution_code  TEXT,                               -- Admin reference solution (hidden)
    PRIMARY KEY (problem_id, language)
);

-- Test cases: input/output stored as S3 keys (can be large)
CREATE TABLE test_cases (
    test_case_id   UUID          PRIMARY KEY DEFAULT gen_random_uuid(),
    problem_id     UUID          NOT NULL REFERENCES problems(problem_id) ON DELETE CASCADE,
    case_number    SMALLINT      NOT NULL,
    input_s3_key   TEXT          NOT NULL,             -- S3 object key for input file
    output_s3_key  TEXT          NOT NULL,             -- S3 object key for expected output
    is_sample      BOOLEAN       NOT NULL DEFAULT FALSE, -- Shown to user as example?
    score_weight   NUMERIC(5,2)  NOT NULL DEFAULT 1.00,  -- For partial scoring
    UNIQUE (problem_id, case_number)
);

CREATE TABLE editorials (
    editorial_id   UUID          PRIMARY KEY DEFAULT gen_random_uuid(),
    problem_id     UUID          NOT NULL UNIQUE REFERENCES problems(problem_id),
    content        TEXT          NOT NULL,             -- Markdown
    solution_code  JSONB         NOT NULL DEFAULT '{}', -- {language: code} map
    published_at   TIMESTAMPTZ,
    created_by     UUID          REFERENCES users(user_id)
);

-- ─────────────────────────────────────────────
-- SUBMISSIONS
-- ─────────────────────────────────────────────

CREATE TABLE submissions (
    submission_id  UUID          PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id        UUID          NOT NULL REFERENCES users(user_id),
    problem_id     UUID          NOT NULL REFERENCES problems(problem_id),
    contest_id     UUID          REFERENCES contests(contest_id),  -- NULL if practice
    language       VARCHAR(16)   NOT NULL,
    code_s3_key    TEXT          NOT NULL,             -- User's code stored in S3
    status         VARCHAR(16)   NOT NULL DEFAULT 'PENDING',
      -- PENDING, JUDGING, ACCEPTED, WRONG_ANSWER, TLE, MLE, RUNTIME_ERROR, COMPILE_ERROR
    verdict        VARCHAR(16),                        -- Final verdict after judging
    runtime_ms     INT,                                -- Best runtime across test cases
    memory_mb      NUMERIC(7,2),                       -- Peak memory across test cases
    failed_case    SMALLINT,                           -- Index of first failed test case (WA only)
    error_message  TEXT,                               -- Compile error or RE message
    judged_at      TIMESTAMPTZ,
    created_at     TIMESTAMPTZ   NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_submissions_user_problem ON submissions(user_id, problem_id);
CREATE INDEX idx_submissions_problem      ON submissions(problem_id, created_at DESC);
CREATE INDEX idx_submissions_contest      ON submissions(contest_id, user_id) WHERE contest_id IS NOT NULL;
CREATE INDEX idx_submissions_status       ON submissions(status) WHERE status = 'PENDING';

-- Aggregate per-test-case results for detailed verdict display
CREATE TABLE submission_test_results (
    submission_id  UUID          NOT NULL REFERENCES submissions(submission_id) ON DELETE CASCADE,
    case_number    SMALLINT      NOT NULL,
    verdict        VARCHAR(16)   NOT NULL,             -- per-case verdict
    runtime_ms     INT,
    memory_mb      NUMERIC(7,2),
    PRIMARY KEY (submission_id, case_number)
);

-- ─────────────────────────────────────────────
-- CONTESTS
-- ─────────────────────────────────────────────

CREATE TABLE contests (
    contest_id     UUID          PRIMARY KEY DEFAULT gen_random_uuid(),
    slug           VARCHAR(128)  NOT NULL UNIQUE,
    title          VARCHAR(255)  NOT NULL,
    description    TEXT,
    format         VARCHAR(16)   NOT NULL DEFAULT 'icpc',  -- 'icpc', 'weighted'
    start_time     TIMESTAMPTZ   NOT NULL,
    end_time       TIMESTAMPTZ   NOT NULL,
    is_rated       BOOLEAN       NOT NULL DEFAULT TRUE,
    is_public      BOOLEAN       NOT NULL DEFAULT TRUE,
    created_by     UUID          REFERENCES users(user_id),
    created_at     TIMESTAMPTZ   NOT NULL DEFAULT NOW()
);

CREATE TABLE contest_problems (
    contest_id     UUID          NOT NULL REFERENCES contests(contest_id),
    problem_id     UUID          NOT NULL REFERENCES problems(problem_id),
    position       SMALLINT      NOT NULL,             -- A, B, C, D, ... (display order)
    score          INT           NOT NULL DEFAULT 100,  -- Max score for this problem
    PRIMARY KEY (contest_id, problem_id)
);

CREATE TABLE contest_registrations (
    contest_id     UUID          NOT NULL REFERENCES contests(contest_id),
    user_id        UUID          NOT NULL REFERENCES users(user_id),
    registered_at  TIMESTAMPTZ   NOT NULL DEFAULT NOW(),
    final_rank     INT,                                -- Set after contest ends
    rating_change  INT,                                -- ELO delta
    PRIMARY KEY (contest_id, user_id)
);

-- ─────────────────────────────────────────────
-- DISCUSSIONS
-- ─────────────────────────────────────────────

CREATE TABLE discussion_posts (
    post_id        UUID          PRIMARY KEY DEFAULT gen_random_uuid(),
    problem_id     UUID          NOT NULL REFERENCES problems(problem_id) ON DELETE CASCADE,
    author_id      UUID          NOT NULL REFERENCES users(user_id),
    parent_post_id UUID          REFERENCES discussion_posts(post_id),  -- NULL = top-level
    title          VARCHAR(255),                       -- Only for top-level posts
    content        TEXT          NOT NULL,
    upvotes        INT           NOT NULL DEFAULT 0,
    is_accepted_solution BOOLEAN NOT NULL DEFAULT FALSE,
    created_at     TIMESTAMPTZ   NOT NULL DEFAULT NOW(),
    updated_at     TIMESTAMPTZ   NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_posts_problem ON discussion_posts(problem_id, upvotes DESC);

CREATE TABLE post_votes (
    user_id        UUID          NOT NULL REFERENCES users(user_id),
    post_id        UUID          NOT NULL REFERENCES discussion_posts(post_id),
    vote           SMALLINT      NOT NULL DEFAULT 1,   -- 1 = upvote, -1 = downvote
    PRIMARY KEY (user_id, post_id)
);

-- ─────────────────────────────────────────────
-- USER PROGRESS
-- ─────────────────────────────────────────────

-- Materialized view of each user's best submission per problem
CREATE TABLE user_problem_status (
    user_id        UUID          NOT NULL REFERENCES users(user_id),
    problem_id     UUID          NOT NULL REFERENCES problems(problem_id),
    status         VARCHAR(16)   NOT NULL,             -- 'solved', 'attempted'
    best_runtime_ms INT,
    best_memory_mb  NUMERIC(7,2),
    first_solved_at TIMESTAMPTZ,
    last_submitted_at TIMESTAMPTZ,
    PRIMARY KEY (user_id, problem_id)
);

-- Daily submission count for heatmap
CREATE TABLE user_activity (
    user_id        UUID          NOT NULL REFERENCES users(user_id),
    activity_date  DATE          NOT NULL,
    submission_count SMALLINT    NOT NULL DEFAULT 0,
    accepted_count SMALLINT      NOT NULL DEFAULT 0,
    PRIMARY KEY (user_id, activity_date)
);
```

---

## 2. Redis Key Patterns

```
# Session / Auth
session:{user_id}                    → JWT payload + plan        TTL 3600s
rate:submit:{user_id}                → INT (submission count)    TTL 60s
rate:api:{user_id}                   → INT (request count)       TTL 60s

# Submission verdict polling (fast path before DB write completes)
verdict:{submission_id}              → JSON verdict object        TTL 300s

# Judge deduplication (prevent re-processing if Kafka redelivers)
judge:dedup:{submission_id}          → "1"                        TTL 600s

# Contest leaderboard (sorted set)
leaderboard:{contest_id}            → ZSET: member=user_id, score=composite_score

# Contest leaderboard page cache (top N)
leaderboard:cache:{contest_id}:page:1 → JSON array of top 25    TTL 5s

# Problem of the Day
potd:today                          → problem_id                 TTL until midnight

# Problem metadata cache (avoid DB hit on every page load)
problem:{problem_id}                → JSON problem object         TTL 3600s
problem:list:page:{n}:{filters}     → JSON page of problems       TTL 300s
```

---

## 3. Class Design / Entities

### Core Domain Entities

```
┌───────────────────────────────────────────────────────────────────────────────────┐
│                              DOMAIN MODEL                                         │
└───────────────────────────────────────────────────────────────────────────────────┘

┌────────────────────────┐       ┌────────────────────────┐
│         User           │       │        Problem         │
├────────────────────────┤       ├────────────────────────┤
│ userId: UUID           │       │ problemId: UUID        │
│ username: String       │       │ slug: String           │
│ email: String          │       │ title: String          │
│ role: UserRole         │       │ difficulty: Difficulty │
│ rating: Int            │       │ description: String    │
│ problemsSolved: Int    │       │ constraints: String    │
│ createdAt: Instant     │       │ hints: List<String>    │
└──────────┬─────────────┘       │ isPublished: Boolean   │
           │ 1                   │ acceptanceRate: Double │
           │                     │ totalSubmissions: Long │
           │ submits             │ tags: List<Tag>        │
           │ N                   │ languages: Map<Lang,   │
┌──────────▼─────────────┐       │   LangConfig>          │
│       Submission       │       │ testCases: List<       │
├────────────────────────┤       │   TestCase>            │
│ submissionId: UUID     │ N  1  └────────────────────────┘
│ userId: UUID           ├───────►
│ problemId: UUID        │
│ contestId: UUID?       │       ┌────────────────────────┐
│ language: Language     │       │       TestCase         │
│ codeS3Key: String      │       ├────────────────────────┤
│ status: SubStatus      │       │ testCaseId: UUID       │
│ verdict: Verdict?      │       │ problemId: UUID        │
│ runtimeMs: Int?        │       │ caseNumber: Int        │
│ memoryMb: Double?      │       │ inputS3Key: String     │
│ failedCase: Int?       │       │ outputS3Key: String    │
│ errorMessage: String?  │       │ isSample: Boolean      │
│ createdAt: Instant     │       │ scoreWeight: Double    │
└────────────────────────┘       └────────────────────────┘

┌────────────────────────┐       ┌────────────────────────┐
│        Contest         │       │    ContestProblem      │
├────────────────────────┤       ├────────────────────────┤
│ contestId: UUID        │ 1   N │ contestId: UUID        │
│ slug: String           ├───────► problemId: UUID        │
│ title: String          │       │ position: Char (A-Z)   │
│ format: ContestFormat  │       │ maxScore: Int          │
│ startTime: Instant     │       └────────────────────────┘
│ endTime: Instant       │
│ isRated: Boolean       │       ┌────────────────────────┐
│ registrations: List<   │       │  ContestRegistration   │
│   Registration>        │       ├────────────────────────┤
└────────────────────────┘       │ contestId: UUID        │
                                  │ userId: UUID           │
                                  │ finalRank: Int?        │
                                  │ ratingChange: Int?     │
                                  └────────────────────────┘

┌────────────────────────┐       ┌────────────────────────┐
│     DiscussionPost     │       │       Editorial        │
├────────────────────────┤       ├────────────────────────┤
│ postId: UUID           │       │ editorialId: UUID      │
│ problemId: UUID        │       │ problemId: UUID        │
│ authorId: UUID         │       │ content: String (MD)   │
│ parentPostId: UUID?    │       │ solutionCode:          │
│ title: String?         │       │   Map<Lang, String>    │
│ content: String        │       │ publishedAt: Instant?  │
│ upvotes: Int           │       └────────────────────────┘
│ createdAt: Instant     │
└────────────────────────┘
```

---

### Enumerations

```
enum UserRole        { GUEST, USER, ADMIN, CONTEST_ADMIN }
enum Difficulty      { EASY, MEDIUM, HARD }
enum Language        { PYTHON3, JAVA, CPP, JAVASCRIPT, GO, RUST, KOTLIN, SWIFT }
enum SubStatus       { PENDING, JUDGING, ACCEPTED, WRONG_ANSWER, TLE, MLE,
                       RUNTIME_ERROR, COMPILE_ERROR, SYSTEM_ERROR }
enum ContestFormat   { ICPC, WEIGHTED, IOI }
enum Verdict         { AC, WA, TLE, MLE, RE, CE, SE }
```

---

### Service Interfaces (Java-style pseudocode)

```java
// ─── Submission Service ────────────────────────────────────
interface SubmissionService {
    // Accept submission, persist, enqueue to Kafka, return 202
    SubmissionResponse submitCode(UUID userId, SubmitRequest req);

    // Fast-path: check Redis first, then DB
    SubmissionStatus   getStatus(UUID submissionId);

    // Paginated history for a user + problem
    Page<Submission>   getUserSubmissions(UUID userId, UUID problemId, Pageable page);
}

// ─── Judge Worker (internal, runs in pod) ─────────────────
interface JudgeWorker {
    JudgeResult execute(JudgeJob job);
    // JudgeJob: { submissionId, language, codeS3Key, testCases[], limits }
    // JudgeResult: { verdict, runtimeMs, memoryMb, failedCase, errorMessage }
}

// ─── Problem Service ───────────────────────────────────────
interface ProblemService {
    Problem          getProblem(String slug, UUID requestingUserId);
    Page<Problem>    listProblems(ProblemFilters filters, Pageable page);
    Problem          createProblem(CreateProblemRequest req);
    void             publishProblem(UUID problemId);
    List<TestCase>   getTestCases(UUID problemId);      // Admin only
    StarterCode      getStarterCode(UUID problemId, Language lang);
}

// ─── Contest Service ───────────────────────────────────────
interface ContestService {
    Contest                   getContest(String slug);
    void                      register(UUID contestId, UUID userId);
    LeaderboardPage           getLeaderboard(UUID contestId, int page);
    void                      processVerdict(UUID contestId, UUID userId,
                                             UUID problemId, Verdict verdict);
    void                      finalizeContest(UUID contestId);  // Triggers rating update
}

// ─── Search Service ────────────────────────────────────────
interface SearchService {
    SearchResults    search(String query, SearchFilters filters, Pageable page);
    List<String>     autocomplete(String prefix, int limit);
}
```

---

### Judge Job Message Schema (Kafka, JSON)

```json
{
  "submission_id": "a1b2c3d4-...",
  "problem_id":    "p9q8r7s6-...",
  "language":      "python3",
  "code_s3_key":   "submissions/2025/07/24/a1b2c3d4.py",
  "time_limit_ms": 2000,
  "memory_limit_mb": 256,
  "test_cases": [
    { "case_number": 1, "input_s3_key": "tc/p9q8/1.in", "output_s3_key": "tc/p9q8/1.out" },
    { "case_number": 2, "input_s3_key": "tc/p9q8/2.in", "output_s3_key": "tc/p9q8/2.out" }
  ],
  "submitted_at": "2025-07-24T12:00:00Z",
  "idempotency_key": "sub-a1b2c3d4"
}
```

### Judge Result Message Schema (Kafka, JSON)

```json
{
  "submission_id": "a1b2c3d4-...",
  "verdict":       "WRONG_ANSWER",
  "runtime_ms":    185,
  "memory_mb":     42.3,
  "failed_case":   3,
  "error_message": null,
  "test_results": [
    { "case_number": 1, "verdict": "AC", "runtime_ms": 90,  "memory_mb": 40.1 },
    { "case_number": 2, "verdict": "AC", "runtime_ms": 95,  "memory_mb": 41.0 },
    { "case_number": 3, "verdict": "WA", "runtime_ms": 185, "memory_mb": 42.3 }
  ],
  "judged_at": "2025-07-24T12:00:03Z"
}
```

---

*Last updated: System Design Case Study v1.0*
