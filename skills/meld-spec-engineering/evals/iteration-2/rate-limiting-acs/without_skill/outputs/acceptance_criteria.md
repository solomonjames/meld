# Acceptance Criteria: API Rate Limiting

## Feature Summary

Rate limit API requests using a sliding window algorithm, with tiered limits based on user plan, informative response headers, and proper HTTP 429 handling when limits are exceeded.

---

## AC-1: Standard Rate Limit Enforcement

**Given** a standard (non-premium) API key
**When** the key has made fewer than 100 requests in the current sliding window (1 minute)
**Then** the request is processed normally and returns the expected response

**Given** a standard API key
**When** the key has made 100 or more requests within the current 1-minute sliding window
**Then** the request is rejected with HTTP status `429 Too Many Requests`

---

## AC-2: Premium Rate Limit Enforcement

**Given** a premium API key
**When** the key has made fewer than 1000 requests in the current 1-minute sliding window
**Then** the request is processed normally and returns the expected response

**Given** a premium API key
**When** the key has made 1000 or more requests within the current 1-minute sliding window
**Then** the request is rejected with HTTP status `429 Too Many Requests`

---

## AC-3: Sliding Window Behavior

**Given** a standard API key that made 100 requests between T+0s and T+10s
**When** a new request arrives at T+61s
**Then** the request is allowed because the requests at T+0s through T+10s have fallen outside the 1-minute sliding window

**Given** a standard API key that made 50 requests at T+0s and 50 requests at T+30s
**When** a new request arrives at T+45s
**Then** the request is rejected because all 100 previous requests are still within the sliding window

**Given** the same key from the above scenario
**When** a new request arrives at T+61s
**Then** the request is allowed because only the 50 requests from T+30s remain within the window (50 < 100)

---

## AC-4: Rate Limit Response Headers on Successful Requests

**Given** any valid API request that is processed successfully
**When** the response is returned
**Then** the following headers are included:

| Header                  | Value                                                              |
|-------------------------|--------------------------------------------------------------------|
| `X-RateLimit-Limit`     | The maximum number of requests allowed per minute (100 or 1000)    |
| `X-RateLimit-Remaining` | The number of requests remaining in the current window             |
| `X-RateLimit-Reset`     | Unix timestamp (in seconds) when the oldest request in the current window expires, opening capacity |

---

## AC-5: Rate Limit Response Headers on Rejected Requests

**Given** an API request that is rejected due to rate limiting
**When** the 429 response is returned
**Then** the response includes:

| Header                  | Value                                                              |
|-------------------------|--------------------------------------------------------------------|
| `X-RateLimit-Limit`     | The maximum number of requests allowed per minute (100 or 1000)    |
| `X-RateLimit-Remaining` | `0`                                                                |
| `X-RateLimit-Reset`     | Unix timestamp when the oldest request in the window expires       |
| `Retry-After`           | Number of seconds the client should wait before retrying           |

---

## AC-6: Retry-After Header Accuracy

**Given** a rate-limited request receiving a 429 response
**When** the client waits the number of seconds specified in the `Retry-After` header and retries
**Then** the retried request is accepted (assuming no additional requests consumed the freed capacity)

---

## AC-7: Per-Key Isolation

**Given** API key A has exhausted its rate limit
**When** API key B (with available capacity) makes a request
**Then** key B's request is processed normally, unaffected by key A's usage

---

## AC-8: Rate Limit Header Values Are Accurate

**Given** a standard API key that has made exactly 37 requests in the current window
**When** a new request is made
**Then** `X-RateLimit-Remaining` returns `62` (100 - 38, accounting for the current request)
**And** `X-RateLimit-Limit` returns `100`

---

## AC-9: Plan Tier Reflected in Limit Header

**Given** a premium API key
**When** any request is made
**Then** `X-RateLimit-Limit` returns `1000`

**Given** a standard API key
**When** any request is made
**Then** `X-RateLimit-Limit` returns `100`

---

## AC-10: 429 Response Body

**Given** a rate-limited request
**When** the 429 response is returned
**Then** the response body contains a JSON object with:
- A human-readable error message indicating the rate limit has been exceeded
- The `retry_after` value in seconds matching the `Retry-After` header

---

## AC-11: Invalid or Missing API Key

**Given** a request with an invalid or missing API key
**When** the request is received
**Then** the request is rejected with the appropriate authentication error (401/403) before rate limiting is evaluated
**And** no rate limit headers are included in the response

---

## Out of Scope

- Distributed rate limiting across multiple server instances (implementation detail, not acceptance criteria)
- Rate limit configuration changes at runtime
- Burst allowances beyond the stated per-minute limits
- Rate limiting by IP address (only API key-based limiting is in scope)
