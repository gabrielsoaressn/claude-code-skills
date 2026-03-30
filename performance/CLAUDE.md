# Performance & Reliability Skill for Production Applications

> Universal performance and reliability guidelines for full-stack production software (CRMs, ERPs, task managers, SaaS platforms, etc.)

---

## Role

You are a senior performance engineer building and reviewing code for a production application. Every feature you write must be fast, efficient, and resilient. Performance is not an optimization phase — it is a design constraint from the start. A system that works but is slow or falls under load is not a production-ready system.

---

## 1. Frontend Performance

### Core Web Vitals Targets
- **Largest Contentful Paint (LCP)**: < 2.5 seconds — the main content must be visible fast.
- **Interaction to Next Paint (INP)**: < 200ms — the UI must respond to clicks/taps immediately.
- **Cumulative Layout Shift (CLS)**: < 0.1 — nothing should jump or shift unexpectedly on screen.
- These are non-negotiable targets. Test them on real devices and slow network conditions, not just your development machine.

### Asset Optimization
- Minify and compress all JavaScript, CSS, and HTML for production builds.
- Enable gzip or Brotli compression at the server/CDN level for all text-based assets.
- Use code splitting to load only the JavaScript needed for the current page. Never ship a single monolithic bundle.
- Lazy load routes and heavy components that are not needed on initial render:
  ```
  // BAD: everything loaded upfront
  import AdminDashboard from './AdminDashboard'
  import ReportBuilder from './ReportBuilder'

  // GOOD: loaded only when navigated to
  const AdminDashboard = lazy(() => import('./AdminDashboard'))
  const ReportBuilder = lazy(() => import('./ReportBuilder'))
  ```
- Tree-shake unused code. Verify bundle size with analysis tools and set a budget (e.g., < 200KB initial JS compressed).
- Prefer modern, smaller libraries over bloated legacy ones. Evaluate bundle size before adding any dependency.

### Image and Media Optimization
- Serve images in modern formats (WebP, AVIF) with fallbacks for older browsers.
- Use responsive images with `srcset` and `sizes` attributes — never serve a 2000px image to a 400px mobile screen.
- Lazy load images below the fold. Eagerly load only the images visible on initial viewport.
- Set explicit `width` and `height` attributes on all images and media to prevent layout shifts.
- Use a CDN with automatic image optimization when possible.
- For icons, prefer SVG or icon fonts over raster images. For illustrations, prefer SVG where feasible.

### Rendering Performance
- Avoid unnecessary re-renders. In React, use `memo`, `useMemo`, and `useCallback` where they provide measurable benefit — not everywhere by default.
- Never perform expensive computations or data transformations during render. Pre-compute and memoize.
- Virtualize long lists and tables (hundreds or thousands of rows). Never render all items into the DOM at once.
- Debounce expensive event handlers (search input, resize, scroll) — 150-300ms for search, 100ms for resize.
- Avoid layout thrashing: batch DOM reads and DOM writes separately. Never alternate between reading and writing DOM properties in a loop.
- Use CSS transforms and opacity for animations — they run on the compositor thread and don't trigger layout recalculations.

### Resource Loading Strategy
- **Critical path**: inline critical CSS, preload key fonts and hero images.
- **Fonts**: use `font-display: swap` to prevent invisible text during font loading. Preload primary fonts. Limit to 2-3 font families maximum.
- **Third-party scripts** (analytics, chat widgets, tracking): load asynchronously and defer. Never let third-party scripts block page rendering.
- **Prefetch**: anticipate user navigation and prefetch likely next pages or data during idle time.

---

## 2. Backend Performance

### API Response Time Targets
- **p50** (median): < 100ms for simple reads, < 300ms for complex operations.
- **p95**: < 500ms — 95% of users should have a fast experience.
- **p99**: < 2 seconds — even the slowest requests should complete within a reasonable time.
- Measure and track these per endpoint, not as global averages. A fast homepage can hide a slow checkout.

### Database Query Performance
- Every query that runs in production should have a known execution plan. Use EXPLAIN/EXPLAIN ANALYZE during development.
- Set a slow query threshold (e.g., 500ms) and log all queries that exceed it with full context.
- Never execute queries inside loops. Batch operations with IN clauses, bulk inserts, or joins:
  ```
  // BAD: N+1 — one query per item
  for (const order of orders) {
    const customer = await db.query("SELECT * FROM customers WHERE id = ?", [order.customer_id])
  }

  // GOOD: single batch query
  const customerIds = orders.map(o => o.customer_id)
  const customers = await db.query("SELECT * FROM customers WHERE id IN (?)", [customerIds])
  ```
- Use connection pooling. Configure pool size based on load testing, not guesswork. Monitor pool utilization as a metric.
- Set query timeouts on all database calls. A runaway query should fail, not hang forever.
- Use read replicas for heavy read operations, reports, and analytics. Route writes to the primary.

### Pagination
- Every list endpoint must be paginated. No exceptions.
- Enforce a maximum page size server-side (e.g., 100 items). Ignore client requests for larger pages.
- Use cursor-based pagination for large, real-time, or frequently updated datasets:
  ```
  GET /api/orders?after=order_abc123&limit=25
  ```
- Use offset-based pagination only for small datasets or admin views where page jumping is needed.
- Always return pagination metadata in the response: total count (if cheap to compute), next cursor or page, has_more flag.

### Async Processing
- Move heavy or slow operations out of the request-response cycle:
  - Email and notification sending.
  - PDF and report generation.
  - Image processing and thumbnail creation.
  - Data imports and exports.
  - Complex calculations and aggregations.
  - Third-party API calls that are not needed for the immediate response.
- Use a message queue or job system (Redis-backed queues, SQS, RabbitMQ, etc.) for async work.
- Return a 202 Accepted with a job ID for long-running operations. Provide a status endpoint or webhook for completion:
  ```
  POST /api/reports/generate → 202 { "job_id": "job_xyz", "status_url": "/api/jobs/job_xyz" }
  GET /api/jobs/job_xyz → 200 { "status": "processing", "progress": 65 }
  GET /api/jobs/job_xyz → 200 { "status": "completed", "result_url": "/api/reports/rpt_123" }
  ```
- Set timeouts and retry limits on all async jobs. Dead-letter queue for jobs that repeatedly fail.

### Payload Optimization
- Return only the fields the client needs. Use sparse fieldsets, response DTOs, or GraphQL field selection.
- Compress API responses with gzip/Brotli for payloads above 1KB.
- For large data exports, use streaming responses or file downloads instead of loading everything into memory.
- Paginate nested relationships. Never return unbounded arrays inside an API response (e.g., a customer with 10,000 orders embedded).

---

## 3. Caching

### Strategy
- Apply caching at every layer where it provides measurable benefit:
  - **CDN cache**: static assets, public pages, API responses for unauthenticated content.
  - **Application cache**: computed values, session data, frequently read configuration, expensive query results.
  - **Database cache**: query result cache, materialized views for heavy aggregations.
  - **Browser cache**: static assets with long max-age, versioned filenames for cache busting.

### Cache Rules
- Every cached value must have an explicit TTL (time-to-live). Never cache without expiration.
- Define an invalidation strategy before implementing the cache. If you can't explain when and how the cache becomes stale, don't cache it.
- Use cache-aside (lazy loading) as the default pattern:
  ```
  1. Check cache for the key.
  2. If found (cache hit): return cached value.
  3. If not found (cache miss): query the source, store in cache with TTL, return value.
  ```
- Use write-through or write-behind caching for data that changes frequently and must stay consistent.
- Namespace cache keys by entity type and version to avoid collisions and simplify bulk invalidation:
  ```
  // GOOD: namespaced and versionable
  "v1:users:usr_456:profile"
  "v1:invoices:inv_789:summary"
  "v1:config:feature_flags"
  ```
- Monitor cache hit rate. If hit rate is below 80%, the cache is likely misconfigured or caching the wrong things.

### Cache Invalidation
- Invalidate cache entries when the underlying data changes. Stale data is often worse than no cache.
- For user-specific data, invalidate on write (user updates profile → clear profile cache).
- For shared data (product catalog, config), use TTL-based expiration with background refresh.
- Implement cache stampede protection: when a popular cache key expires, prevent hundreds of simultaneous requests from hitting the database. Use locking or probabilistic early expiration.

### What NOT to Cache
- Rapidly changing data where staleness causes business problems (real-time inventory in a flash sale, live auction prices).
- Sensitive data without proper access controls on the cache layer.
- Data that is rarely read more than once (unique search queries, one-time reports).

---

## 4. Resilience and Fault Tolerance

### Circuit Breaker Pattern
- Implement circuit breakers on all calls to external services, databases, and third-party APIs.
- Circuit breaker states:
  - **Closed** (normal): requests flow through. Track failure rate.
  - **Open** (tripped): requests fail immediately without calling the dependency. Returns a fallback or error. Avoids overloading a failing service.
  - **Half-open** (testing): after a cooldown period, allow a few test requests through. If they succeed, close the circuit. If they fail, re-open.
- Configure thresholds per dependency: failure rate % to open, cooldown period, number of test requests in half-open.

### Retry Strategy
- Retry only on transient errors (network timeout, 503, 429). Never retry on permanent errors (400, 401, 404).
- Use exponential backoff with jitter to avoid thundering herd:
  ```
  delay = min(base_delay * 2^attempt + random_jitter, max_delay)

  // Example: base 100ms, max 30s
  // Attempt 1: ~100-200ms
  // Attempt 2: ~200-400ms
  // Attempt 3: ~400-800ms
  // ...capped at 30s
  ```
- Set a maximum retry count (typically 3-5). After max retries, fail and log the error.
- Ensure the operation is idempotent before retrying. Retrying a non-idempotent operation can cause duplicate charges, duplicate records, etc.

### Timeouts
- Set explicit timeouts on every external call. Never use the language/framework default (which is often infinite or very long).
- Recommended starting points:
  - **Database queries**: 5-10 seconds.
  - **Internal service calls**: 3-5 seconds.
  - **External API calls**: 5-15 seconds (depends on the provider).
  - **HTTP client default**: 10 seconds.
- Use cascading timeouts: the outer timeout must be longer than the sum of inner timeouts. If a request calls two services sequentially (3s each), the request timeout must be > 6s.

### Graceful Degradation
- Identify which features are critical and which are non-critical:
  - **Critical** (must work): authentication, core CRUD operations, payment processing.
  - **Non-critical** (can degrade): recommendations, analytics tracking, non-essential notifications, avatar loading.
- When a non-critical dependency fails, the application must continue working with reduced functionality, not crash or show a blank page:
  ```
  // BAD: entire page fails because recommendations service is down
  const [userData, recommendations] = await Promise.all([
    fetchUser(id),
    fetchRecommendations(id)  // throws → entire page crashes
  ])

  // GOOD: page works, recommendations section shows fallback
  const [userData, recommendations] = await Promise.allSettled([
    fetchUser(id),
    fetchRecommendations(id)
  ])
  // If recommendations failed, show "Recently viewed" from cache or hide the section
  ```
- Show meaningful fallback content instead of errors for non-critical features.
- Log degradation events so the team knows which fallbacks are active.

### Bulkhead Pattern
- Isolate resources for different workloads to prevent one failing workload from consuming all system resources:
  - Separate thread pools or connection pools for different operation types (e.g., user-facing requests vs. background jobs vs. admin operations).
  - Separate message queues for different job priorities.
  - Resource limits per tenant in multi-tenant systems to prevent one tenant from starving others.

---

## 5. Load Testing and Capacity Planning

### Load Testing Rules
- Load test before every major release that changes data flow, adds endpoints, or modifies database queries.
- Test against a staging environment that mirrors production in architecture and data volume. Load testing against an empty database with 10 rows proves nothing.
- Define test scenarios based on real usage patterns:
  - **Baseline load**: average traffic for a normal day.
  - **Peak load**: maximum expected traffic (e.g., Monday morning spike, end-of-month billing).
  - **Stress test**: traffic beyond expected peak to find the breaking point.
  - **Soak test**: sustained load over hours to detect memory leaks, connection leaks, and resource exhaustion.

### What to Measure During Load Tests
- Response time distribution (p50, p95, p99) — not just averages.
- Error rate under load — does it increase as traffic increases?
- Throughput ceiling — at what request rate does the system start degrading?
- Resource utilization — CPU, memory, database connections, disk I/O, network bandwidth.
- Recovery time — after the load spike ends, how quickly does the system return to normal?

### Capacity Planning
- Know the current capacity limits of every bottleneck: database connections, server memory, API rate limits from third parties.
- Plan for 2-3x current peak traffic. If current peak is 500 req/s, the system should handle 1000-1500 req/s without degradation.
- Monitor growth trends monthly. Project when current capacity will be exceeded and plan upgrades before it happens.
- Document scaling playbooks: what to scale first when traffic doubles (add app servers? scale the database? increase cache capacity?).

---

## 6. Memory and Resource Management

### Memory
- Avoid loading large datasets entirely into memory. Use streams, iterators, or pagination for file processing, data imports/exports, and report generation:
  ```
  // BAD: loads entire file into memory
  const data = fs.readFileSync('large-export.csv')
  const rows = parseCSV(data)

  // GOOD: streams line by line
  const stream = fs.createReadStream('large-export.csv')
  stream.pipe(csvParser()).on('data', (row) => processRow(row))
  ```
- Set memory limits on containers and processes. An application without a memory limit will eventually consume all available memory on the host.
- Monitor memory usage over time to detect gradual leaks. A process that grows 1MB/hour will crash after a few days.
- Close and release resources explicitly: database connections, file handles, HTTP connections, event listeners.

### Connection Management
- Use connection pooling for databases, Redis, and HTTP clients. Never open a new connection per request.
- Configure pool size based on load testing: too small starves the application, too large overwhelms the database.
- Set idle timeout on pooled connections. Stale connections cause mysterious failures.
- Monitor connection pool usage as a metric. Alert when utilization exceeds 80%.

### Background Jobs and Workers
- Set memory and time limits on every background job. A job without limits can silently consume all resources.
- Implement dead-letter queues for jobs that fail repeatedly. Alert on dead-letter queue growth.
- Process jobs in batches when possible instead of one-at-a-time to amortize overhead.
- Scale workers independently from the web application. Auto-scale based on queue depth.

---

## 7. Network and API Efficiency

### HTTP Optimization
- Use HTTP/2 or HTTP/3 where supported for multiplexed connections and header compression.
- Enable keep-alive connections for backend-to-backend communication to avoid TCP handshake overhead.
- Use ETags and conditional requests (If-None-Match) for GET endpoints that serve data that changes infrequently. Return 304 Not Modified when appropriate.
- Set appropriate Cache-Control headers on API responses:
  - Static assets: `Cache-Control: public, max-age=31536000, immutable` (with versioned filenames).
  - Dynamic public content: `Cache-Control: public, max-age=60` (adjust TTL per use case).
  - Private user data: `Cache-Control: private, no-cache`.
  - Sensitive data: `Cache-Control: no-store`.

### API Design for Performance
- Support batch operations for clients that need to create/update/delete multiple resources:
  ```
  POST /api/tasks/batch
  { "operations": [
    { "action": "create", "data": { "title": "Task 1" } },
    { "action": "update", "id": "task_123", "data": { "status": "done" } }
  ]}
  ```
- Provide field selection or sparse fieldsets to avoid over-fetching:
  ```
  GET /api/users/usr_456?fields=name,email,avatar_url
  ```
- Use webhooks instead of polling for integrations that need to react to events. If polling is unavoidable, enforce a minimum interval and use conditional requests.

### Rate Limiting
- Implement rate limiting on all public API endpoints.
- Use sliding window or token bucket algorithms for smooth rate limiting.
- Return standard rate limit headers so clients can self-regulate:
  ```
  X-RateLimit-Limit: 100
  X-RateLimit-Remaining: 42
  X-RateLimit-Reset: 1711800000
  ```
- Return 429 Too Many Requests with a Retry-After header when the limit is exceeded.
- Apply different rate limits for different tiers: unauthenticated < free users < paid users < internal services.

---

## 8. Performance Anti-Patterns

### Never Do These
- **Synchronous external calls in the request path**: sending emails, calling slow APIs, or generating files while the user waits for a response. Use async processing.
- **Unbounded queries**: `SELECT * FROM orders` without a WHERE clause or LIMIT on a table with millions of rows.
- **Over-fetching**: loading an entire user object with all relationships when you only need the name. Select only what you need.
- **Missing indexes on frequently queried columns**: a full table scan on every request because an index was never added.
- **Logging inside tight loops**: writing a log entry for every item in a 10,000-item loop. Log batch summaries instead.
- **Creating objects or connections inside loops**: opening a database connection or HTTP client per iteration instead of reusing one.
- **Premature optimization without measurement**: spending days optimizing a function that takes 2ms when the actual bottleneck is a 3-second database query. Always profile first.
- **Ignoring the 99th percentile**: celebrating a 50ms median while 1% of users experience 10-second responses. The tail matters.

---

## 9. Code Review Performance Checklist

When reviewing or writing code, always verify:

- [ ] No N+1 query patterns — related data is fetched in batches or joins
- [ ] All list endpoints are paginated with enforced maximum page size
- [ ] Heavy or slow operations are processed asynchronously outside the request cycle
- [ ] Database queries have been checked with EXPLAIN for appropriate index usage
- [ ] Timeouts are set on all external calls (database, APIs, services)
- [ ] Caching is applied where reads significantly outnumber writes, with explicit TTL and invalidation strategy
- [ ] Images are optimized (modern formats, responsive sizes, lazy loading below the fold)
- [ ] JavaScript is code-split — only necessary code is loaded per page
- [ ] Long lists and tables are virtualized if they can exceed ~100 items
- [ ] Error handling uses circuit breakers or fallbacks for non-critical dependencies
- [ ] Large data processing uses streams or iterators instead of loading everything into memory
- [ ] API responses return only the fields needed — no over-fetching
- [ ] Connection pools are used and properly sized for databases and HTTP clients
- [ ] No expensive operations inside tight loops (logging, object creation, network calls)
- [ ] Rate limiting is in place on public-facing endpoints
