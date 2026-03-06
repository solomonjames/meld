# API Rate Limiting - Acceptance Criteria

## Happy Path

**AC-1: Standard user receives rate limit headers on every response**
- Given a standard API key with fewer than 100 requests in the current sliding window
- When the user makes an API request
- Then the response includes the following headers:
  - `X-RateLimit-Limit: 100`
  - `X-RateLimit-Remaining: {remaining requests}`
  - `X-RateLimit-Reset: {unix timestamp when the window resets}`

**AC-2: Standard user request succeeds within rate limit**
- Given a standard API key that has made fewer than 100 requests in the last 60 seconds
- When the user makes an API request
- Then the request is processed normally with a 2xx response
- And `X-RateLimit-Remaining` decrements by 1

**AC-3: Premium user receives correct rate limit headers**
- Given a premium API key with fewer than 1000 requests in the current sliding window
- When the user makes an API request
- Then the response includes `X-RateLimit-Limit: 1000`
- And `X-RateLimit-Remaining` reflects the remaining count out of 1000

**AC-4: Premium user request succeeds within rate limit**
- Given a premium API key that has made fewer than 1000 requests in the last 60 seconds
- When the user makes an API request
- Then the request is processed normally with a 2xx response

## Error Handling

**AC-5: Standard user exceeds rate limit**
- Given a standard API key that has made 100 requests in the last 60 seconds
- When the user makes an additional API request
- Then the response status is `429 Too Many Requests`
- And the response includes a `Retry-After` header with the number of seconds until the user can make another request
- And the response includes `X-RateLimit-Remaining: 0`

**AC-6: Premium user exceeds rate limit**
- Given a premium API key that has made 1000 requests in the last 60 seconds
- When the user makes an additional API request
- Then the response status is `429 Too Many Requests`
- And the response includes a `Retry-After` header with the number of seconds until the user can make another request
- And the response includes `X-RateLimit-Remaining: 0`

**AC-7: Invalid or missing API key**
- Given a request with an invalid or missing API key
- When the user makes an API request
- Then the request is rejected with the appropriate authentication error (401/403)
- And no rate limit headers are included (rate limiting does not apply to unauthenticated requests)

## Edge Cases

**AC-8: Sliding window allows requests after oldest requests expire**
- Given a standard API key that hit 100 requests and received a 429 response
- When the oldest request in the sliding window ages past 60 seconds
- Then subsequent requests are allowed again
- And `X-RateLimit-Remaining` reflects the newly available capacity

**AC-9: Burst at window boundary**
- Given a standard API key that made 99 requests starting 59 seconds ago
- When the user makes 1 more request immediately
- Then the 100th request succeeds
- And the 101st request within the same sliding window returns `429 Too Many Requests`

**AC-10: Retry-After accuracy**
- Given a standard API key that has exhausted its rate limit
- When the user receives a 429 response with a `Retry-After` value of N seconds
- Then waiting N seconds and retrying results in a successful request

**AC-11: Concurrent requests at the limit boundary**
- Given a standard API key with exactly 1 request remaining in the sliding window
- When 2 requests arrive simultaneously
- Then exactly 1 request succeeds and 1 returns `429 Too Many Requests`
- And the rate counter does not allow negative remaining counts

**AC-12: X-RateLimit-Reset reflects sliding window**
- Given a standard API key with active requests in the sliding window
- When the user makes a request
- Then `X-RateLimit-Reset` contains a Unix timestamp indicating when the earliest request in the window expires, restoring capacity

## Authorization Checks

**AC-13: Tier-appropriate limits are enforced per API key**
- Given two API keys, one standard and one premium
- When both keys make requests
- Then the standard key is limited to 100 requests per minute
- And the premium key is limited to 1000 requests per minute
- And each key's counters are tracked independently

**AC-14: Tier change takes effect on rate limits**
- Given a standard API key that is upgraded to premium
- When the user makes requests after the upgrade
- Then the new limit of 1000 requests per minute applies
- And `X-RateLimit-Limit` reflects the updated tier
