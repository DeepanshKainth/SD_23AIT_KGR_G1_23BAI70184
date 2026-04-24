# API Design — Online Coding Judge

Base URL: `https://api.codejudge.io/v1`  
Auth: `Authorization: Bearer <JWT>` on all protected endpoints.  
Format: All bodies are `application/json` unless noted.

---

## 1. Authentication

### POST `/auth/register`
Register a new user account.

**Request**
```json
{
  "username": "algo_master",
  "email": "user@example.com",
  "password": "Str0ng!Pass",
  "display_name": "Algo Master"
}
```

**Response `201 Created`**
```json
{
  "user_id": "a1b2c3d4-...",
  "username": "algo_master",
  "email": "user@example.com",
  "access_token": "eyJhbGci...",
  "refresh_token": "dGhpcyBp...",
  "token_type": "Bearer",
  "expires_in": 3600
}
```

**Error Responses**

| Status | Code | Description |
|---|---|---|
| `400` | `VALIDATION_ERROR` | Missing or invalid fields |
| `409` | `USERNAME_TAKEN` | Username already exists |
| `409` | `EMAIL_EXISTS` | Email already registered |

---

### POST `/auth/login`
Authenticate with email and password.

**Request**
```json
{
  "email": "user@example.com",
  "password": "Str0ng!Pass",
  "device_id": "browser-fingerprint-abc"
}
```

**Response `200 OK`**
```json
{
  "access_token": "eyJhbGci...",
  "refresh_token": "dGhpcyBp...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "user": {
    "user_id": "a1b2c3d4-...",
    "username": "algo_master",
    "role": "user",
    "rating": 1742
  }
}
```

**Error Responses**

| Status | Code | Description |
|---|---|---|
| `401` | `INVALID_CREDENTIALS` | Wrong email or password |
| `403` | `ACCOUNT_BANNED` | User is banned |
| `429` | `TOO_MANY_ATTEMPTS` | Rate limit exceeded |

---

### POST `/auth/refresh`
Get a new access token using a refresh token.

**Request**
```json
{ "refresh_token": "dGhpcyBp..." }
```

**Response `200 OK`**
```json
{ "access_token": "eyJhbGci...", "expires_in": 3600 }
```

---

### DELETE `/auth/logout`
Revoke the current refresh token.  
**Response `204 No Content`**

---

## 2. Users

### GET `/users/me`
Get authenticated user's full profile.

**Response `200 OK`**
```json
{
  "user_id": "a1b2c3d4-...",
  "username": "algo_master",
  "display_name": "Algo Master",
  "avatar_url": "https://cdn.codejudge.io/avatars/a1b2.png",
  "bio": "Competitive programmer from India.",
  "rating": 1742,
  "problems_solved": 312,
  "country_code": "IN",
  "github_url": "https://github.com/algomaster",
  "stats": {
    "easy_solved": 120,
    "medium_solved": 150,
    "hard_solved": 42,
    "total_submissions": 890,
    "acceptance_rate": 64.2
  },
  "badges": [
    { "badge_id": "streak-100", "name": "100-Day Streak", "icon_url": "..." }
  ],
  "created_at": "2023-03-01T10:00:00Z"
}
```

---

### GET `/users/{username}`
Get a public user profile.

**Response `200 OK`**
```json
{
  "user_id": "...",
  "username": "algo_master",
  "display_name": "Algo Master",
  "rating": 1742,
  "problems_solved": 312,
  "rank": 4201,
  "recent_activity": [
    { "date": "2025-07-24", "count": 5 }
  ]
}
```

**Error Responses**

| Status | Code | Description |
|---|---|---|
| `404` | `USER_NOT_FOUND` | Username does not exist |

---

### PATCH `/users/me`
Update profile fields (all optional).

**Request**
```json
{
  "display_name": "New Name",
  "bio": "Updated bio.",
  "github_url": "https://github.com/newhandle"
}
```

**Response `200 OK`** — returns updated user object.

---

## 3. Problems

### GET `/problems`
List problems with filters and pagination.

**Query Parameters**

| Param | Type | Default | Description |
|---|---|---|---|
| `difficulty` | string | all | `easy`, `medium`, `hard` |
| `tag` | string | all | Tag slug (e.g., `dynamic-programming`) |
| `status` | string | all | `solved`, `attempted`, `unsolved` (requires auth) |
| `search` | string | — | Full-text search on title and tags |
| `page` | int | 1 | Page number |
| `limit` | int | 20 | Results per page (max 100) |
| `sort_by` | string | `id` | `acceptance_rate`, `difficulty`, `title` |

**Response `200 OK`**
```json
{
  "total": 2847,
  "page": 1,
  "limit": 20,
  "problems": [
    {
      "problem_id": "p1q2r3-...",
      "slug": "two-sum",
      "title": "Two Sum",
      "difficulty": "easy",
      "acceptance_rate": 48.3,
      "total_submissions": 12400000,
      "tags": ["array", "hash-table"],
      "user_status": "solved"
    }
  ]
}
```

---

### GET `/problems/{slug}`
Get full problem details.

**Response `200 OK`**
```json
{
  "problem_id": "p1q2r3-...",
  "slug": "two-sum",
  "title": "Two Sum",
  "difficulty": "easy",
  "description": "Given an array of integers `nums`...",
  "constraints": "- 2 <= nums.length <= 10^4\n- -10^9 <= nums[i] <= 10^9",
  "hints": ["Try using a hash map.", "Think about what complement means."],
  "tags": [
    { "slug": "array", "display_name": "Array" },
    { "slug": "hash-table", "display_name": "Hash Table" }
  ],
  "sample_test_cases": [
    {
      "case_number": 1,
      "input": "nums = [2,7,11,15], target = 9",
      "output": "[0,1]",
      "explanation": "nums[0] + nums[1] == 9, return [0, 1]."
    }
  ],
  "supported_languages": ["python3", "java", "cpp", "javascript", "go"],
  "starter_code": {
    "python3": "class Solution:\n    def twoSum(self, nums, target):\n        ",
    "java": "class Solution {\n    public int[] twoSum(int[] nums, int target) {\n    }\n}"
  },
  "stats": {
    "acceptance_rate": 48.3,
    "total_submissions": 12400000,
    "total_accepted": 5989200
  },
  "user_status": "solved",
  "has_editorial": true
}
```

**Error Responses**

| Status | Code | Description |
|---|---|---|
| `404` | `PROBLEM_NOT_FOUND` | Slug does not exist or problem not published |
| `403` | `PREMIUM_REQUIRED` | Problem is premium; user is on free plan |

---

### GET `/problems/{slug}/editorial`
Get the editorial for a problem (only after it's published).

**Response `200 OK`**
```json
{
  "content": "## Approach\nUse a hash map to store...",
  "solution_code": {
    "python3": "class Solution:\n    def twoSum(self, nums, target):\n        seen = {}\n...",
    "java": "class Solution { ... }"
  },
  "published_at": "2024-01-15T09:00:00Z"
}
```

**Error Responses**

| Status | Code | Description |
|---|---|---|
| `404` | `EDITORIAL_NOT_FOUND` | No editorial published yet |

---

### GET `/problems/today`
Get the Problem of the Day.

**Response `200 OK`** — same shape as `GET /problems/{slug}` with an added `potd_date` field.

---

## 4. Code Execution & Submissions

### POST `/problems/{slug}/run`
Run code against custom or sample test cases only (not hidden cases). Does not count toward
submission history.

**Request**
```json
{
  "language": "python3",
  "code": "class Solution:\n    def twoSum(self, nums, target):\n        ...",
  "custom_input": "nums = [3,2,4], target = 6"
}
```

**Response `202 Accepted`**
```json
{
  "run_id": "run-uuid-...",
  "status": "PENDING",
  "poll_url": "/v1/runs/run-uuid-..."
}
```

---

### GET `/runs/{run_id}`
Poll run result.

**Response `200 OK`** (while pending)
```json
{ "run_id": "run-uuid-...", "status": "JUDGING" }
```

**Response `200 OK`** (completed)
```json
{
  "run_id": "run-uuid-...",
  "status": "COMPLETED",
  "verdict": "WRONG_ANSWER",
  "stdout": "[0, 2]",
  "expected_output": "[1, 2]",
  "runtime_ms": 48,
  "memory_mb": 14.2,
  "error_message": null
}
```

---

### POST `/problems/{slug}/submit`
Submit code for full judging against all hidden test cases.

**Request**
```json
{
  "language": "python3",
  "code": "class Solution:\n    def twoSum(self, nums, target):\n        seen = {}\n        for i, n in enumerate(nums):\n            if target-n in seen:\n                return [seen[target-n], i]\n            seen[n] = i",
  "contest_id": null
}
```

**Response `202 Accepted`**
```json
{
  "submission_id": "s1b2c3-...",
  "status": "PENDING",
  "poll_url": "/v1/submissions/s1b2c3-...",
  "queue_position": 3
}
```

**Error Responses**

| Status | Code | Description |
|---|---|---|
| `400` | `UNSUPPORTED_LANGUAGE` | Language not allowed for this problem |
| `400` | `CODE_TOO_LARGE` | Code exceeds 64 KB limit |
| `401` | `UNAUTHORIZED` | Must be logged in to submit |
| `429` | `SUBMISSION_RATE_LIMIT` | Max 10 submissions/min exceeded |

---

### GET `/submissions/{submission_id}`
Get submission details and verdict. Used for polling.

**Response `200 OK`** (pending/judging)
```json
{
  "submission_id": "s1b2c3-...",
  "status": "JUDGING",
  "created_at": "2025-07-24T12:00:00Z"
}
```

**Response `200 OK`** (completed — AC)
```json
{
  "submission_id": "s1b2c3-...",
  "problem_slug": "two-sum",
  "language": "python3",
  "status": "ACCEPTED",
  "verdict": "ACCEPTED",
  "runtime_ms": 48,
  "memory_mb": 14.2,
  "runtime_percentile": 92.4,
  "memory_percentile": 76.1,
  "test_results": [
    { "case_number": 1, "verdict": "AC", "runtime_ms": 22, "memory_mb": 13.8 },
    { "case_number": 2, "verdict": "AC", "runtime_ms": 26, "memory_mb": 14.2 }
  ],
  "created_at": "2025-07-24T12:00:00Z",
  "judged_at": "2025-07-24T12:00:02Z"
}
```

**Response `200 OK`** (completed — WA)
```json
{
  "submission_id": "s1b2c3-...",
  "status": "WRONG_ANSWER",
  "verdict": "WRONG_ANSWER",
  "runtime_ms": 85,
  "memory_mb": 18.0,
  "failed_case": 5,
  "test_results": [
    { "case_number": 1, "verdict": "AC" },
    { "case_number": 5, "verdict": "WA" }
  ],
  "judged_at": "2025-07-24T12:00:03Z"
}
```

---

### GET `/problems/{slug}/submissions`
List the authenticated user's submissions for a problem.

**Query Params:** `language`, `verdict`, `page` (default 1), `limit` (default 20)

**Response `200 OK`**
```json
{
  "total": 8,
  "page": 1,
  "submissions": [
    {
      "submission_id": "s1b2c3-...",
      "language": "python3",
      "verdict": "ACCEPTED",
      "runtime_ms": 48,
      "memory_mb": 14.2,
      "created_at": "2025-07-24T12:00:00Z"
    }
  ]
}
```

---

### GET `/submissions/{submission_id}/code`
Retrieve the source code for a submission (owner only, or admin).

**Response `200 OK`**
```json
{
  "submission_id": "s1b2c3-...",
  "language": "python3",
  "code": "class Solution:\n    def twoSum(self, nums, target):\n        ..."
}
```

**Error Responses**

| Status | Code | Description |
|---|---|---|
| `403` | `NOT_YOUR_SUBMISSION` | Can only view own submission code |

---

## 5. Contests

### GET `/contests`
List upcoming, ongoing, and past contests.

**Query Params:** `status` (`upcoming`, `active`, `past`), `page`, `limit`

**Response `200 OK`**
```json
{
  "contests": [
    {
      "contest_id": "c1d2e3-...",
      "slug": "weekly-contest-420",
      "title": "Weekly Contest 420",
      "format": "weighted",
      "start_time": "2025-07-27T02:30:00Z",
      "end_time": "2025-07-27T04:00:00Z",
      "duration_minutes": 90,
      "is_rated": true,
      "registered_count": 18420,
      "is_registered": false
    }
  ]
}
```

---

### GET `/contests/{slug}`
Get contest details and its problem list.

**Response `200 OK`**
```json
{
  "contest_id": "c1d2e3-...",
  "slug": "weekly-contest-420",
  "title": "Weekly Contest 420",
  "start_time": "2025-07-27T02:30:00Z",
  "end_time": "2025-07-27T04:00:00Z",
  "problems": [
    { "position": "A", "slug": "easy-warmup", "title": "Easy Warmup", "score": 500 },
    { "position": "B", "slug": "medium-dp",   "title": "Medium DP",   "score": 1000 },
    { "position": "C", "slug": "hard-graph",  "title": "Hard Graph",  "score": 1500 },
    { "position": "D", "slug": "nightmare",   "title": "Nightmare",   "score": 3000 }
  ],
  "registered_count": 18420,
  "is_registered": true
}
```

---

### POST `/contests/{slug}/register`
Register the authenticated user for a contest.

**Response `200 OK`**
```json
{ "message": "Successfully registered for Weekly Contest 420." }
```

**Error Responses**

| Status | Code | Description |
|---|---|---|
| `400` | `CONTEST_STARTED` | Registration closed; contest has begun |
| `409` | `ALREADY_REGISTERED` | User already registered |

---

### GET `/contests/{slug}/leaderboard`
Get paginated contest leaderboard.

**Query Params:** `page` (default 1), `limit` (default 25, max 100)

**Response `200 OK`**
```json
{
  "contest_id": "c1d2e3-...",
  "updated_at": "2025-07-27T03:15:30Z",
  "total_participants": 18420,
  "page": 1,
  "leaderboard": [
    {
      "rank": 1,
      "user": { "user_id": "...", "username": "tourist", "rating": 3742 },
      "total_score": 6000,
      "penalty_minutes": 32,
      "problems": {
        "A": { "solved": true,  "attempts": 1, "solve_time_min": 4  },
        "B": { "solved": true,  "attempts": 2, "solve_time_min": 18 },
        "C": { "solved": true,  "attempts": 1, "solve_time_min": 55 },
        "D": { "solved": false, "attempts": 3, "solve_time_min": null }
      }
    }
  ]
}
```

---

## 6. Discussion

### GET `/problems/{slug}/discussions`
Get top-level discussion posts for a problem.

**Query Params:** `sort_by` (`new`, `top`), `page`, `limit`

**Response `200 OK`**
```json
{
  "total": 340,
  "posts": [
    {
      "post_id": "d1e2f3-...",
      "title": "Clean O(n) Python solution with explanation",
      "author": { "username": "algo_master", "rating": 1742 },
      "upvotes": 412,
      "reply_count": 18,
      "created_at": "2024-03-10T14:22:00Z"
    }
  ]
}
```

---

### POST `/problems/{slug}/discussions`
Create a new discussion post (requires accepted submission for approach posts).

**Request**
```json
{
  "title": "Simple HashMap approach — O(n) time, O(n) space",
  "content": "## Approach\nStore each number in a hash map as we iterate..."
}
```

**Response `201 Created`**
```json
{ "post_id": "d1e2f3-...", "title": "...", "created_at": "2025-07-24T12:00:00Z" }
```

---

### POST `/discussions/{post_id}/upvote`
Upvote a discussion post.  
**Response `200 OK`** — `{ "upvotes": 413 }`

---

## 7. Search

### GET `/search/problems`
Full-text search over problems.

**Query Params:** `q` (required), `difficulty`, `tag`, `page`, `limit`

**Response `200 OK`**
```json
{
  "total": 14,
  "results": [
    {
      "problem_id": "...",
      "slug": "longest-palindromic-substring",
      "title": "Longest Palindromic Substring",
      "difficulty": "medium",
      "acceptance_rate": 32.1,
      "tags": ["string", "dynamic-programming"],
      "highlight": "Find the <em>longest palindromic</em> substring in s."
    }
  ]
}
```

---

### GET `/search/autocomplete`
Autocomplete suggestions for the search bar.

**Query Params:** `q` (required), `limit` (default 5, max 10)

**Response `200 OK`**
```json
{
  "suggestions": [
    { "text": "Longest Palindromic Substring", "type": "problem", "slug": "longest-palindromic-substring" },
    { "text": "Longest Common Subsequence",    "type": "problem", "slug": "longest-common-subsequence" }
  ]
}
```

---

## 8. Admin Endpoints

> All require `role: admin` or `role: contest_admin`.

### POST `/admin/problems`
Create a new problem (draft).

**Request** — full problem object (title, description, difficulty, constraints, hints, tags).

**Response `201 Created`** — returns new problem with `is_published: false`.

---

### PATCH `/admin/problems/{slug}`
Update problem content.  
**Response `200 OK`** — returns updated problem.

---

### POST `/admin/problems/{slug}/publish`
Publish a problem (makes it visible to users).  
**Response `200 OK`** — `{ "message": "Problem published successfully." }`

---

### POST `/admin/problems/{slug}/test-cases`
Upload a test case (multipart form: `input` file + `output` file).

**Response `201 Created`**
```json
{ "test_case_id": "...", "case_number": 7, "is_sample": false }
```

---

## 9. Standard Error Format

All API errors return this envelope:

```json
{
  "error": {
    "status": 429,
    "code": "SUBMISSION_RATE_LIMIT",
    "message": "You can submit at most 10 times per minute. Please wait 23 seconds.",
    "retry_after": 23,
    "request_id": "req-abc123xyz",
    "timestamp": "2025-07-24T12:00:00Z"
  }
}
```

### HTTP Status Code Reference

| Status | Meaning |
|---|---|
| `200 OK` | Successful GET / PATCH / action |
| `201 Created` | Resource created (POST) |
| `202 Accepted` | Async job accepted (submit/run) |
| `204 No Content` | Success with no response body (DELETE, logout) |
| `400 Bad Request` | Validation failure, malformed request |
| `401 Unauthorized` | Missing or invalid JWT |
| `403 Forbidden` | Authenticated but not permitted |
| `404 Not Found` | Resource does not exist |
| `409 Conflict` | Duplicate resource or state conflict |
| `422 Unprocessable Entity` | Semantically invalid (e.g., end_time < start_time) |
| `429 Too Many Requests` | Rate limit hit; `Retry-After` header set |
| `500 Internal Server Error` | Unexpected server-side error |
| `503 Service Unavailable` | Judge queue overloaded; retry later |

---

## 10. WebSocket — Real-Time Verdict

For real-time submission verdict delivery, clients can connect to:

```
wss://api.codejudge.io/v1/ws?token=<JWT>
```

**Client subscribes to a submission:**
```json
{ "action": "subscribe", "submission_id": "s1b2c3-..." }
```

**Server pushes verdict when ready:**
```json
{
  "type": "verdict",
  "submission_id": "s1b2c3-...",
  "verdict": "ACCEPTED",
  "runtime_ms": 48,
  "memory_mb": 14.2,
  "runtime_percentile": 92.4
}
```

**Contest leaderboard subscription:**
```json
{ "action": "subscribe_leaderboard", "contest_id": "c1d2e3-..." }
```

**Server pushes rank updates every 30 seconds:**
```json
{
  "type": "leaderboard_update",
  "contest_id": "c1d2e3-...",
  "top_5": [ { "rank": 1, "username": "tourist", "score": 6000 } ],
  "your_rank": 142
}
```

---

*Last updated: System Design Case Study v1.0*
