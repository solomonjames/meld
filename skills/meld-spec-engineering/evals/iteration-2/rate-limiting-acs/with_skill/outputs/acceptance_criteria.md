# Acceptance Criteria: API Rate Limiting

## Happy Path

**AC-1: Standard user receives rate limit headers on every response**
- Given an authenticated API request with a standard API key
- When the request is processed successfully
- Then the response includes `X-RateLimit-Limit: 100` header
- And the response includes `X-RateLimit-Remaining` header with the number of requests remaining in the current window
- And the response includes `X-RateLimit-Reset` header with the Unix timestamp when the window resets

**AC-2: Standard user makes requests within the limit**
- Given a standard API key that has made fewer than 100 requests in the current sliding window
- When a new API request is made with that key
- Then the request is processed normally (2xx response as appropriate)
- And `X-RateLimit-Remaining` decrements by 1 compared to the previous response

**AC-3: Premium user receives correct rate limit headers**
- Given an authenticated API request with a premium API key
- When the request is processed successfully
- Then the response includes `X-RateLimit-Limit: 1000` header
- And the response includes `X-RateLimit-Remaining` header reflecting the premium allowance
- And the response includes `X-RateLimit-Reset` header with the Unix timestamp when the window resets

**AC-4: Premium user makes requests within the elevated limit**
- Given a premium API key that has made fewer than 1000 requests in the current sliding window
- When a new API request is made with that key
- Then the request is processed normally (2xx response as appropriate)

## Error Handling

**AC-5: Standard user exceeds rate limit**
- Given a standard API key that has made 100 requests within the current 1-minute sliding window
- When a new API request is made with that key
- Then the response status is `429 Too Many Requests`
- And the response includes a `Retry-After` header with the number of seconds until the caller can retry
- And the response includes `X-RateLimit-Remaining: 0`
- And the response body contains a message indicating the rate limit has been exceeded

**AC-6: Premium user exceeds rate limit**
- Given a premium API key that has made 1000 requests within the current 1-minute sliding window
- When a new API request is made with that key
- Then the response status is `429 Too Many Requests`
- And the response includes a `Retry-After` header with the number of seconds until the caller can retry
- And the response includes `X-RateLimit-Remaining: 0`

**AC-7: Request with missing or invalid API key**
- Given a request with no API key or an unrecognized API key
- When the request is received
- Then the response status is `401 Unauthorized`
- And no rate limit headers are included (the request is rejected before rate limiting applies)

## Edge Cases

**AC-8: Sliding window allows requests after oldest ones expire**
- Given a standard API key that hit the 100-request limit
- When 30 seconds pass and 40 of the original requests fall outside the 1-minute sliding window
- Then the next request succeeds
- And `X-RateLimit-Remaining` reflects approximately 39 remaining requests (accounting for the newly made request)

**AC-9: Exactly at the boundary -- 100th request for standard user**
- Given a standard API key that has made 99 requests in the current sliding window
- When the 100th request is made
- Then the request is processed normally (2xx response as appropriate)
- And `X-RateLimit-Remaining: 0`

**AC-10: Request immediately after the 100th for standard user**
- Given a standard API key that has made exactly 100 requests in the current sliding window
- When the 101st request is made
- Then the response status is `429 Too Many Requests`
- And the `Retry-After` header indicates the number of seconds until the oldest request exits the window

**AC-11: Concurrent requests arriving simultaneously**
- Given a standard API key with 99 requests in the current window
- When two requests arrive at the same instant
- Then exactly one request succeeds and one returns `429 Too Many Requests`
- And the rate counter accurately reflects 100 total requests (no race condition allowing over-limit)

**AC-12: Rate limits are per API key, not global**
- Given two different standard API keys, Key A and Key B
- When Key A has exhausted its 100-request limit
- Then requests from Key B are still processed normally (assuming Key B is within its own limit)

**AC-13: Retry-After header accuracy**
- Given a standard API key that has exceeded its rate limit
- When the `Retry-After` value has elapsed
- Then the next request from that key succeeds

**AC-14: Idempotent requests still count toward rate limit**
- Given a standard API key with 98 requests remaining
- When the same idempotent request (e.g., identical GET) is made twice
- Then both requests count toward the rate limit
- And `X-RateLimit-Remaining` decrements by 2

## Authorization

**AC-15: Standard vs. premium tier detection**
- Given an API key associated with a standard-tier account
- When any API request is made
- Then the rate limit enforced is 100 requests per minute
- And `X-RateLimit-Limit` is 100

**AC-16: Premium tier grants elevated limit**
- Given an API key associated with a premium-tier account
- When any API request is made
- Then the rate limit enforced is 1000 requests per minute
- And `X-RateLimit-Limit` is 1000

**AC-17: Tier change takes effect on next window**
- Given a standard user whose account is upgraded to premium
- When the user's current sliding window expires and new requests are made
- Then the new rate limit of 1000 requests per minute applies
- And `X-RateLimit-Limit` updates to 1000

**AC-18: Revoked API key is rejected, not rate-limited**
- Given an API key that has been revoked or deactivated
- When a request is made with that key
- Then the response status is `401 Unauthorized`
- And the request does not consume any rate limit quota
