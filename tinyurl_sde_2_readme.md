# SDE 2 System Design Notes — TinyURL

## Why this problem is asked
TinyURL is a classic SDE 2 system design question because it tests whether you can:
- clarify requirements,
- estimate scale,
- identify the dominant access pattern,
- design a simple but scalable read-heavy service,
- discuss tradeoffs around ID generation, storage, caching, and reliability.

At SDE 2 level, the interviewer is usually looking for a structured answer with solid reasoning, not an overly complex architecture.

---

## 1. Problem statement
Design a URL shortening service like TinyURL.

The service should:
- accept a long URL and return a short URL,
- redirect users from the short URL to the original long URL,
- scale to large read traffic,
- keep mappings durable and unique.

Optional follow-ups:
- custom alias,
- expiration time,
- analytics / click tracking,
- abuse detection,
- user accounts.

---

## 2. Clarify requirements in interview
A strong SDE 2 answer starts by scoping the problem.

### Functional requirements
- Create a short URL for a given long URL
- Redirect short URL to long URL

### Optional requirements
- Custom alias
- Expiry / TTL
- Analytics
- Delete / disable link
- Same long URL returns same short URL or new short URL every time

### Non-functional requirements
- High availability
- Low redirect latency
- Durable storage of mappings
- Unique short codes
- Scalability

### Good interview line
> I’ll first design the core system for shorten + redirect, then extend it for analytics, expiration, and custom aliases if time permits.

---

## 3. Key observation
This is usually a **read-heavy** system.

- URL creation traffic is much smaller
- Redirect traffic is much larger

That means:
- optimize the read path,
- use caching aggressively,
- keep redirect latency low.

---

## 4. Back-of-the-envelope estimation
You do not need perfect math. You need reasonable thinking.

Example assumptions:
- 100 million new short URLs per month
- 1 billion redirects per month
- average long URL length: ~500 bytes
- metadata + short code + timestamps: ~100–200 bytes more

Implications:
- storage grows steadily but is manageable with sharding
- reads dominate, so DB offload via cache matters a lot
- service is not compute-heavy; it is lookup-heavy

---

## 5. API design

### Create short URL
```http
POST /api/v1/urls
Content-Type: application/json

{
  "long_url": "https://www.example.com/some/very/long/path"
}
```

Response:
```json
{
  "short_url": "https://tiny.ly/ab12Cd"
}
```

### Redirect
```http
GET /ab12Cd
```

Response:
- HTTP 301 or 302 redirect to original URL

### Optional analytics
```http
GET /api/v1/urls/ab12Cd/stats
```

---

## 6. High-level design
```text
Client
  |
  v
Load Balancer
  |
  v
URL Service (stateless app servers)
  |                  \
  |                   \
  v                    v
Cache (Redis)       URL Mapping DB
                          |
                          v
                    Analytics/Event Queue (optional)
```

### Responsibilities
- **Load Balancer**: distribute traffic across app servers
- **URL Service**: handle shorten and redirect APIs
- **Cache**: speed up redirect lookups
- **DB**: store durable mapping of short code to long URL
- **Queue**: optional asynchronous click tracking / analytics

---

## 7. Core data model
Primary mapping:
```text
short_code -> long_url
```

Example schema:
```text
URLMapping
---------
short_code   (PK)
long_url
created_at
expires_at   nullable
user_id      nullable
status       optional
click_count  optional but usually not updated synchronously
```

Optional reverse lookup if product requires deduplication:
```text
long_url_hash -> short_code
```

---

## 8. The most important design choice: short code generation
This is the core of the problem.

### Recommended approach
Generate a unique numeric ID and encode it using Base62.

Example:
- unique ID = `123456789`
- Base62 encode -> `8M0kX`

### Why Base62?
Base62 uses:
- `a-z`
- `A-Z`
- `0-9`

That gives 62 characters per position.

Capacity examples:
- `62^6` ≈ 56.8 billion
- `62^7` ≈ 3.5 trillion

So even 6–7 characters are usually enough for very large scale.

### Why this approach is good
- clean uniqueness guarantee if ID source is unique
- simple lookup path
- no collision-management complexity from truncating hashes
- easy to shard by short code

---

## 9. ID generation options

### Option A: DB auto-increment
Pros:
- simple

Cons:
- central bottleneck at high scale
- single writer dependency

### Option B: Dedicated ID generator
Pros:
- scalable
- centralized uniqueness

Cons:
- extra service to manage

### Option C: Snowflake-style distributed ID generator
Pros:
- scalable
- no single DB bottleneck
- globally unique IDs

Cons:
- more implementation complexity

### SDE 2 recommendation
> I’d use a distributed unique ID generator or a Snowflake-like scheme, then Base62-encode the ID into the short code.

---

## 10. Why not hash the long URL directly?
Alternative idea:
- hash the long URL,
- take part of the hash,
- encode as short code.

### Problems
- collisions become a real issue if hash is truncated
- harder to support multiple short URLs for the same long URL
- less flexible for custom aliases and product changes

### Good tradeoff statement
> I prefer ID-based generation over direct URL hashing because it gives simpler uniqueness guarantees and avoids collision-handling complexity.

---

## 11. Write path: create short URL
```text
1. Client sends long URL
2. Service validates input
3. Generate unique ID
4. Base62-encode it into short code
5. Store mapping in DB
6. Return short URL
```

### Diagram
```text
Client
  |
POST /urls
  |
URL Service
  |
Generate ID -> Base62 encode
  |
Store (short_code, long_url) in DB
  |
Return short URL
```

### Important note
For the base design, generate a **new short URL per request**. If product later requires deduplication, add a reverse index on normalized long URL hash.

---

## 12. Read path: redirect
```text
1. Client requests short URL
2. Service extracts short code
3. Check cache
4. If miss, fetch from DB
5. Return HTTP redirect
6. Optionally emit async click event
```

### Diagram
```text
Client
  |
GET /ab12Cd
  |
URL Service
  |
Check Redis cache
  | hit -> return redirect
  |
  | miss
  v
DB lookup
  |
Populate cache
  |
Return redirect
```

### Why this matters
Redirect is the hottest path. It should stay:
- fast,
- simple,
- mostly read-only.

---

## 13. Caching strategy
Cache:
```text
short_code -> long_url
```

### Why caching is critical
- this is a read-heavy workload
- hot links will be requested repeatedly
- cache significantly reduces DB load
- user-facing latency improves

### Good interview sentence
> Since redirect traffic dominates, the cache is one of the biggest performance wins in the system.

---

## 14. Database choice
The primary access pattern is:
```text
lookup by short_code
```

That makes this problem a natural fit for a key-value oriented data store.

### Acceptable SDE 2 answer
- Start with a relational DB if scale is moderate and operational simplicity matters
- Move to or design for a distributed key-value / wide-column store as traffic grows

### Strong balanced statement
> Since the main access pattern is a simple key lookup from short code to long URL, a distributed key-value store is a natural fit. A relational DB is also acceptable initially, especially for simpler implementation and operational familiarity.

---

## 15. Sharding strategy
As scale grows, shard by:
```text
hash(short_code)
```

### Why short_code is a good shard key
- primary read path uses short_code
- primary write path also stores by short_code
- avoids cross-shard queries for the core use case
- spreads load more evenly

### Good interview statement
> Since both read and write paths are keyed by short code, sharding by short code keeps the primary workflow local to one shard.

---

## 16. 301 vs 302 redirect

### 301 Permanent Redirect
Pros:
- cacheable by browsers/CDNs
- lower repeat lookup load

Cons:
- less flexible if destination changes later

### 302 Temporary Redirect
Pros:
- easier to preserve control over routing behavior
- better if destination may change or analytics policy needs flexibility

### Good tradeoff sentence
> If short links are immutable and I want stronger client-side caching, I’d use 301. If I want flexibility to change destinations or keep routing more dynamic, I’d choose 302.

---

## 17. Click analytics
Do **not** update analytics synchronously in the redirect path at high scale.

### Better approach
- return redirect immediately
- publish click event asynchronously
- process analytics later

### Diagram
```text
Redirect Service
   |
   +--> return redirect immediately
   |
   +--> emit click event to queue/log
              |
              v
        Analytics processor
```

### Why this is good
- keeps critical path low latency
- avoids extra DB writes on every redirect
- scales analytics independently

---

## 18. Availability and reliability
A TinyURL-like service is user-facing and should be highly available.

### Design for HA
- multiple stateless app servers
- load balancer
- replicated cache
- replicated DB / failover
- backups
- monitoring and alerts

### Good interview sentence
> I’d keep app servers stateless so they can scale horizontally and fail independently, and I’d replicate the mapping store to improve availability of the read path.

---

## 19. Abuse and security considerations
In production, the service can be abused for:
- spam,
- phishing,
- malicious redirects,
- automated link creation.

### Mitigations
- rate limiting on link creation
- URL validation
- blacklist / allowlist checks
- abuse detection pipeline
- optional delayed activation for suspicious links

This is a strong optional point if time permits.

---

## 20. Expiration support
If product requires expiry:
- store `expires_at`
- check during redirect
- optionally clean up in background

This is a straightforward extension.

---

## 21. Duplicate long URLs: same or new short URL?
Two product choices:

### Option 1: New short URL every time
Pros:
- simpler
- no reverse lookup needed

### Option 2: Reuse same short URL for same long URL
Pros:
- avoids duplicates

Cons:
- need reverse index
- must normalize URL carefully
- more write-path complexity

### Recommended interview stance
> For the base design, I’d generate a new short URL per request for simplicity. If deduplication is a product requirement, I’d add a reverse index using normalized long URL hash.

---

## 22. Strong SDE 2 answer structure
In the interview, answer in this order:

1. Clarify requirements
2. Call out read-heavy nature
3. Define APIs
4. Draw high-level architecture
5. Explain short-code generation
6. Explain DB + cache
7. Explain sharding
8. Add reliability / analytics / abuse follow-ups

### Good opening line
> I’ll start with the core shorten-and-redirect service, then optimize for the dominant read path, and finally cover ID generation, storage, sharding, and reliability tradeoffs.

---

## 23. What the interviewer is really testing
They are not testing whether you memorized TinyURL.
They are testing whether you can:
- structure a design discussion,
- identify critical access patterns,
- choose reasonable storage,
- handle uniqueness,
- optimize the hottest path,
- discuss tradeoffs calmly and clearly.

---

## 24. Strong vs weak answer

### Weak answer
- jumps into tools immediately
- no requirements clarification
- no scale reasoning
- no explanation of why cache / DB / ID strategy is chosen
- no awareness of read-heavy workload

### Strong answer
- starts simple
- explains the workload shape
- chooses practical building blocks
- justifies tradeoffs
- grows the design naturally

---

## 25. Sample polished SDE 2 answer
> I’d scope the core requirements to shortening a long URL and redirecting short URLs. This is typically a read-heavy system, so I’d optimize the redirect path for low latency. I’d use stateless app servers behind a load balancer, a durable mapping store for `short_code -> long_url`, and a cache like Redis for hot lookups. For short code generation, I’d avoid direct URL hashing and instead generate a unique distributed ID and Base62-encode it, because that gives simpler uniqueness guarantees. On the write path, the service validates the URL, generates the short code, stores the mapping, and returns the result. On the read path, it checks cache first, falls back to the DB, and returns an HTTP redirect. As scale grows, I’d shard by short code since both reads and writes key on it. Analytics should be asynchronous so the redirect path stays fast.

---

## 26. Follow-up questions to practice
Be ready for:
- How would you generate unique IDs at scale?
- Why not hash the long URL?
- What DB would you choose and why?
- How would you shard the service?
- How would you track clicks?
- 301 or 302?
- How would you support custom aliases?
- How would you expire links?
- How would you handle abuse?

---

## 27. Revision notes
### One-line takeaway
TinyURL is a **read-heavy key-lookup system** where the most important design choices are:
- short-code generation,
- fast redirect path,
- durable mapping storage,
- caching,
- sharding by short code.

### Memory hooks
- **Read-heavy system**
- **Unique ID + Base62 is the cleanest default**
- **Cache the redirect path**
- **Shard by short_code**
- **Keep analytics async**

---

## 28. Interview cheat sheet
```text
Problem type: Read-heavy mapping service
Core entity: short_code -> long_url
Best default code gen: unique ID + Base62
Hot path: redirect
Optimization: cache
Storage: relational initially or distributed KV at scale
Shard key: short_code
Analytics: async
Follow-ups: custom alias, expiry, abuse, 301 vs 302
```

