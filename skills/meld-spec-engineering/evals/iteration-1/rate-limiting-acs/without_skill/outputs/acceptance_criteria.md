# Acceptance Criteria: API Rate Limiting

## Feature Summary

Implement sliding window rate limiting on API requests, enforced per API key, with differentiated limits for standard and premium users. Rate limit metadata must be communicated via response headers, and exceeded limits must return an appropriate HTTP error response.

---

## AC-1: Standard Rate Limit Enforcement

**Given** a standard (non-premium) API key
**When** the key has made fewer than 100 requests within the current sliding 1-minute window
**Then** the request is processed normally and returns the expected response

---

## AC-2: Premium Rate Limit Enforcement

**Given** a premium API key
**When** the key has made fewer than 1000 requests within the current sliding 1-minute window
**Then** the request is processed normally and returns the expected response

---

## AC-3: Standard Limit Exceeded

**Given** a standard API key
**When** the key has already made 100 requests within the current sliding 1-minute window
**Then** the API returns HTTP `429 Too Many Requests`
**And** the response includes a `Retry-After` header indicating the number of seconds until the client can retry

---

## AC-4: Premium Limit Exceeded

**Given** a premium API key
**When** the key has already made 1000 requests within the current sliding 1-minute window
**Then** the API returns HTTP `429 Too Many Requests`
**And** the response includes a `Retry-After` header indicating the number of seconds until the client can retry

---

## AC-5: Rate Limit Headers on Successful Responses

**Given** any valid API key (standard or premium)
**When** a request is processed successfully
**Then** the response includes the following headers:
- `X-RateLimit-Limit` -- the maximum number of requests allowed in the window (100 for standard, 1000 for premium)
- `X-RateLimit-Remaining` -- the number of requests remaining in the current window
- `X-RateLimit-Reset` -- a Unix timestamp (in seconds) indicating when the current window resets

---

## AC-6: Rate Limit Headers on 429 Responses

**Given** any valid API key that has exceeded its rate limit
**When** the API returns a `429 Too Many Requests` response
**Then** the response includes the `X-RateLimit-Limit`, `X-RateLimit-Remaining` (value of 0), and `X-RateLimit-Reset` headers
**And** the `Retry-After` header value is consistent with the `X-RateLimit-Reset` timestamp

---

## AC-7: Sliding Window Behavior

**Given** a standard API key that made 100 requests between T+0s and T+30s
**When** 31 seconds have elapsed (at T+61s), so the earliest requests have fallen outside the 1-minute window
**Then** new requests are accepted again
**And** `X-RateLimit-Remaining` reflects only the requests still within the sliding window

---

## AC-8: Per-Key Isolation

**Given** two different API keys (A and B)
**When** key A has exhausted its rate limit
**Then** key B's requests are unaffected and continue to be processed normally, assuming key B is within its own limit

---

## AC-9: Invalid or Missing API Key

**Given** a request with no API key or an invalid API key
**When** the request is received
**Then** the system returns the appropriate authentication error (e.g., `401 Unauthorized`)
**And** rate limiting logic is not applied (no rate limit headers are returned)

---

## AC-10: Retry-After Header Accuracy

**Given** a rate-limited API key
**When** the API returns a `429` response with a `Retry-After` header
**Then** waiting for the number of seconds specified in `Retry-After` and retrying results in a successful request (assuming no additional requests consumed the allowance in the interim)

---

## AC-11: Concurrent Request Handling

**Given** a standard API key with 1 request remaining in its window
**When** two requests arrive simultaneously
**Then** exactly one request succeeds and one receives a `429` response
**And** the rate limit counter remains accurate (no race condition allows both to succeed)

---

## Non-Functional Requirements

- **Accuracy:** The sliding window calculation must be precise to the second; requests outside the window must not count toward the limit.
- **Performance:** Rate limit checks must add no more than 5ms of latency to request processing under normal load.
- **Consistency:** In a distributed deployment, rate limit counts must be consistent across all nodes (e.g., via a shared store such as Redis).
- **Observability:** Rate limit violations should be logged with the API key identifier, timestamp, and current count for monitoring and alerting purposes.
