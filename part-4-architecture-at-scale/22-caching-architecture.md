<!--
  CHAPTER: 22
  TITLE: Full-Stack Caching Architecture — From Database to Pixels
  PART: IV — Architecture at Scale
  PREREQS: Chapters 11, 16, 27
  KEY_TOPICS: caching pyramid, database query cache, Redis/Upstash, CDN edge cache, HTTP caching, TanStack Query, MMKV persistence, image CDN, CMS integration, headless CMS, content pipeline, cache warming, thundering herd, cache stampede, write-through, write-behind, cache-aside, distributed cache, ISR, stale-while-revalidate
  DIFFICULTY: Advanced
  UPDATED: 2026-04-07
-->

# Chapter 22: Full-Stack Caching Architecture — From Database to Pixels

> **Part IV — Architecture at Scale** | Prerequisites: Chapters 11, 16, 27 | Difficulty: Advanced

<details>
<summary><strong>TL;DR</strong></summary>

- A production caching system is a seven-layer pyramid from database to in-memory React state; understanding each layer's speed, persistence, and invalidation model is the difference between an app that feels instant and one that costs you $80k/month in compute
- Cache-aside (lazy loading) is the default pattern for 90% of use cases; write-through for data that must survive crashes; write-behind for high-throughput writes where you can tolerate data loss risk; read-through when the cache library supports it natively
- Redis/Upstash is your application-layer workhorse: session storage, rate limiting, cache-aside TTL objects, pub/sub invalidation, and thundering herd protection via mutex locks
- CDN edge caching with correct Cache-Control headers is the single highest-leverage optimization in your entire stack; one header change can eliminate 95% of origin traffic
- TanStack Query is not just a fetching library -- it is a distributed cache with staleTime as your freshness budget, gcTime as your memory budget, and MMKV persistence as your cold-start accelerator
- CMS content flows through a pipeline: editor publishes in headless CMS, webhook fires, ISR regenerates, CDN caches, mobile app's query layer background-refetches, device persists -- six hops, each with its own TTL and invalidation strategy
- Cache invalidation is genuinely the hardest problem; use time-based TTL as your default, event-driven webhooks for content you control, tag-based revalidation for grouped invalidation, and accept eventual consistency as a design choice rather than a bug

</details>

Chapter 11 taught you the individual caching tools -- MMKV, TanStack Query, Cache-Control headers, Redis. That chapter was about *what each tool does and how to use it.* This chapter is fundamentally different. This chapter is about the **architectural pattern** of layered caching -- how data flows from a Postgres row through seven distinct cache layers until it becomes pixels on a user's screen, and how you design, invalidate, and debug that entire pipeline as a single coherent system.

The distinction matters. Understanding Redis is like understanding how a brake pad works. Understanding full-stack caching architecture is like understanding how the entire braking system works -- from the driver's foot, through the hydraulic lines, to the calipers, to the pads, to the rotors, to the tires' contact patch with the road. When something goes wrong, you need to know which layer failed, not just how each layer works in isolation.

I've watched caching problems consume entire engineering teams. One team spent three weeks debugging why users saw stale event listings -- the fix was a single `revalidateTag` call that nobody knew was needed because nobody had documented the full cache flow. Another team over-cached everything with aggressive TTLs and their Redis bill hit $12,000/month because they were caching user-specific data that had zero cache-hit ratio. Both problems came from the same root cause: no one had drawn the complete picture of how data flows from source of truth to user's eyeballs.

This chapter draws that picture.

### In This Chapter
- The Caching Pyramid -- seven layers from database to React state
- Caching Patterns (the computer science: cache-aside, write-through, write-behind, read-through)
- The Database Layer (Postgres, Neon, PlanetScale, Drizzle ORM)
- Application Cache (Redis/Upstash: data structures, thundering herd, stampede prevention)
- CDN Edge Layer (Cache-Control masterclass, ISR, edge config)
- HTTP Cache on the Client (mobile-specific behaviors, ETags, conditional requests)
- TanStack Query as a Cache Layer (freshness budgets, memory budgets, persistence)
- Device Storage as the Final Cache (MMKV hydration, image disk cache)
- CMS & Content Management for Mobile (headless CMS, content pipelines, webhooks)
- Image & Media CDN (Cloudinary, Imgix, responsive serving, video patterns)
- Cache Invalidation -- The Hard Part (time, event, tag, version strategies)
- Putting It All Together -- a real architecture traced end-to-end

### Related Chapters
- [Ch 11: Caching Strategies] -- individual caching tools and their APIs
- [Ch 10: Data Fetching & Server Communication] -- the fetching layer that feeds these caches
- [Ch 12: Offline-First & Real-Time] -- extending cache patterns to offline and live data
- [Ch 13: Performance Optimization] -- measuring the impact of caching decisions
- [Ch 16: Design Systems at Scale] -- shared components that consume cached data
- [Ch 27: Vercel Platform & Edge] -- the deployment target for CDN and ISR strategies

---

## 1. THE CACHING PYRAMID

Every piece of data in your application lives at one or more layers of what I call the Caching Pyramid. The pyramid has seven layers, ordered from slowest-but-most-authoritative (bottom) to fastest-but-most-volatile (top). Understanding this pyramid is the single most important mental model in this chapter.

```
+---------------------------------------------------------------------------+
|                        THE CACHING PYRAMID                                |
|                                                                           |
|              +-------------------------------------+                      |
|              |  L7: In-Memory Cache                 |  ~0ms               |
|              |  React state, TanStack Query cache   |  volatile / session |
|              |  Size: 50-200MB                       |                     |
|              +-------------------------------------+                      |
|              |  L6: Device Storage                   |  ~1ms              |
|              |  MMKV, SQLite, SecureStore            |  persistent / dev  |
|              |  Size: unlimited (within reason)      |                     |
|              +-------------------------------------+                      |
|              |  L5: HTTP Cache                       |  ~1-5ms            |
|              |  Browser cache, NSURLCache, OkHttp    |  header-controlled |
|              |  Size: OS-managed                      |                    |
|              +-------------------------------------+                      |
|         ========================================== (network boundary)     |
|              |  L4: CDN Edge Cache                   |  ~10-50ms          |
|              |  Vercel Edge, CloudFront, Fastly      |  globally distrib  |
|              |  Size: effectively unlimited           |                    |
|              +-------------------------------------+                      |
|              |  L3: Application Cache                |  ~1-5ms            |
|              |  Redis, Upstash, Memcached            |  server-side       |
|              |  Size: plan-dependent (1-100GB)        |                    |
|              +-------------------------------------+                      |
|              |  L2: Database Query Cache              |  ~1-10ms           |
|              |  pg shared_buffers, MySQL query cache |  query-level       |
|              |  Size: configured buffer pool          |                    |
|              +-------------------------------------+                      |
|              |  L1: Database (Source of Truth)        |  ~5-50ms+          |
|              |  Postgres, MySQL, PlanetScale, Neon   |  authoritative     |
|              |  Size: disk-bound                      |                    |
|              +-------------------------------------+                      |
|                                                                           |
|   Rule: Data flows UP when read, DOWN when written.                       |
|   Each layer can serve a response WITHOUT hitting the layer below.        |
|   Each layer has its own TTL, invalidation strategy, and failure mode.    |
|                                                                           |
+---------------------------------------------------------------------------+
```

### 1.1 The Golden Rule of the Pyramid

**Data flows up when read, down when written.** When a user opens the TicketFlow app and views an event listing:

1. TanStack Query checks its in-memory cache (L7) -- cache hit? Return instantly.
2. Cache miss? Check MMKV persistence (L6) -- have we seen this data before? Hydrate from disk.
3. Still need fresh data? The HTTP layer checks its cache (L5) -- is there a valid cached response?
4. No valid HTTP cache? The request hits the CDN edge (L4) -- does the PoP have a cached response?
5. CDN miss? The request reaches the origin server, which checks Redis (L3) -- is the computed response cached?
6. Redis miss? The database query cache (L2) might have the query result buffered.
7. Full miss? The database executes the query (L1) and the result propagates back up through every layer.

The beauty of this system is that **most requests never reach the bottom.** In a well-architected system, 95%+ of reads are served from L4-L7 and never touch your origin server. Your database handles writes and the small percentage of cache misses. This is how you serve millions of users from a $50/month database.

### 1.2 Each Layer in Detail

#### Layer 1: Database (Source of Truth)

**What lives here:** The canonical, authoritative version of every piece of data. User accounts, event listings, transactions, content.

**Speed:** 5-50ms for simple queries, 100ms+ for complex joins or aggregations. Add network latency if the database is in a different region than your server.

**TTL:** Infinite. This is your source of truth. Data lives here until explicitly deleted.

**Invalidation:** N/A -- this layer *is* the truth. Writes go here first (or eventually, if using write-behind).

**Failure mode:** If the database goes down, your writes fail. Reads can be served from upper layers if cached, but eventually all caches expire and the whole system degrades. This is why you need read replicas and connection pooling.

#### Layer 2: Database Query Cache

**What lives here:** Recently executed query results, buffered in the database engine's memory.

**Speed:** 1-10ms. The query is recognized as previously executed and the result is returned from shared memory rather than re-executing.

**TTL:** Automatic. The database invalidates cached queries when the underlying tables are modified (in Postgres, `shared_buffers` manages this automatically).

**Invalidation:** Automatic on write. When you `UPDATE events SET title = 'New Title' WHERE id = 42`, Postgres invalidates any buffered query results that touched that row.

**Failure mode:** Transparent. A cache miss at this layer simply means the query executes normally. You won't even notice unless you're watching `pg_stat_statements`.

#### Layer 3: Application Cache (Redis/Upstash)

**What lives here:** Computed responses, session data, rate limiting counters, frequently accessed objects that are expensive to compute from the database.

**Speed:** 1-5ms from the same region. Upstash global replication can serve from the nearest region.

**TTL:** Explicit. You set TTL per key. Common patterns: 5 minutes for API responses, 24 hours for user profiles, 1 hour for search results.

**Invalidation:** Time-based (TTL expiry), explicit (delete the key on write), or pattern-based (delete all keys matching a prefix).

**Failure mode:** If Redis goes down, your app falls through to the database. This should be graceful -- never make Redis a hard dependency for reads. Treat it as an accelerator, not a requirement.

```typescript
// The fundamental pattern: graceful Redis fallback
async function getEvent(eventId: string): Promise<Event> {
  // Try L3 (Redis)
  try {
    const cached = await redis.get(`event:${eventId}`);
    if (cached) return JSON.parse(cached);
  } catch (error) {
    // Redis is down -- log it, don't crash
    logger.warn('Redis unavailable, falling through to DB', { error });
  }

  // Fall through to L1 (Database)
  const event = await db.query.events.findFirst({
    where: eq(events.id, eventId),
  });

  // Try to populate L3 for next time
  try {
    await redis.set(`event:${eventId}`, JSON.stringify(event), { ex: 300 }); // 5 min TTL
  } catch {
    // Best-effort cache population
  }

  return event;
}
```

#### Layer 4: CDN Edge Cache

**What lives here:** Full HTTP responses cached at Points of Presence (PoPs) worldwide. Static assets (JS bundles, images, fonts) and dynamic responses that are cacheable (ISR pages, API responses with appropriate headers).

**Speed:** 10-50ms from the nearest PoP. Sub-10ms if the user is physically close to a PoP.

**TTL:** Header-controlled. `Cache-Control: s-maxage=3600` means the CDN caches for 1 hour. `stale-while-revalidate=86400` means the CDN serves stale content for up to 24 hours while revalidating in the background.

**Invalidation:** Path-based purge, tag-based purge (Surrogate-Key), or global purge. ISR combines time-based revalidation with on-demand revalidation via `revalidateTag` or `revalidatePath`.

**Failure mode:** CDN goes down = requests go directly to origin. This is catastrophic for traffic because your origin now takes the full load. This is why CDN uptime matters more than almost any other infrastructure component.

#### Layer 5: HTTP Cache (Client-Side)

**What lives here:** HTTP responses cached by the operating system's HTTP stack. On iOS, `NSURLCache`. On Android, OkHttp's disk cache. In browsers, the HTTP cache.

**Speed:** 1-5ms. Reading from local disk, no network involved.

**TTL:** Controlled by the `Cache-Control` and `Expires` headers from the server's response. The client obeys these headers (mostly -- mobile HTTP clients have quirks we'll cover).

**Invalidation:** Automatic based on headers. `ETag` and `If-None-Match` enable conditional requests -- the client asks "has this changed?" and the server responds with 304 Not Modified (zero body, saving bandwidth) or 200 with the new body.

**Failure mode:** The OS may evict cached responses under memory pressure. Your app should never *depend* on the HTTP cache -- treat it as an optimization.

#### Layer 6: Device Storage (MMKV)

**What lives here:** Serialized application data persisted to the device's filesystem. TanStack Query cache snapshots, user preferences, auth tokens (in SecureStore), recently viewed items.

**Speed:** ~1ms for MMKV. ~15ms for AsyncStorage (which is why we don't use AsyncStorage).

**TTL:** Persistent until explicitly cleared. Survives app restarts, phone reboots, and app updates.

**Invalidation:** Application-controlled. You clear stale data on app launch, after schema migrations, or when the user logs out.

**Failure mode:** Disk full = writes fail silently. Corrupted storage = app should detect and rebuild from network. Always handle MMKV read failures gracefully.

#### Layer 7: In-Memory Cache (React State / TanStack Query)

**What lives here:** The working set of data currently needed by the UI. Query results, computed/derived state, UI state.

**Speed:** ~0ms. It's already in JavaScript memory.

**TTL:** `staleTime` in TanStack Query (how long before data is considered stale and eligible for background refetch). `gcTime` (how long unused data stays in memory before garbage collection).

**Invalidation:** Automatic refetch when staleTime expires. Manual invalidation via `queryClient.invalidateQueries()`. Optimistic updates for immediate UI feedback on mutations.

**Failure mode:** App killed = all in-memory cache lost. This is why L6 (MMKV persistence) exists -- to restore the in-memory cache on cold start.

### 1.3 The Cost Equation

Every layer in the pyramid has a cost:

```
+------------------+---------------+-----------------+------------------+
| Layer            | Speed         | $ Cost          | Staleness Risk   |
+------------------+---------------+-----------------+------------------+
| L7: In-Memory    | ~0ms          | Device RAM       | Session-length   |
| L6: Device Store | ~1ms          | Device storage   | App-controlled   |
| L5: HTTP Cache   | ~1-5ms        | Device storage   | Header-driven    |
| L4: CDN Edge     | ~10-50ms      | CDN bandwidth $$ | TTL-driven       |
| L3: Redis        | ~1-5ms        | Redis hosting $  | TTL-driven       |
| L2: DB Query     | ~1-10ms       | DB memory        | Auto-invalidated |
| L1: Database     | ~5-50ms+      | DB compute $$$   | None (truth)     |
+------------------+---------------+-----------------+------------------+
```

The fundamental trade-off: **higher layers are faster and cheaper but more likely to serve stale data.** Lower layers are slower and more expensive but more accurate. Your job as an architect is to decide, *for each type of data*, which layers it should live at and how stale it's allowed to be.

---

## 2. CACHING PATTERNS -- THE COMPUTER SCIENCE

Before we dive into implementation, you need the vocabulary. These five patterns are the building blocks of every caching system ever built.

### 2.1 Cache-Aside (Lazy Loading)

The most common pattern. The application is responsible for checking the cache and populating it on miss.

```
                Read Path                          Write Path

   App --check--> Cache                    App --write--> Database
    |               |                       |
    | miss          | hit                   |--invalidate--> Cache
    |               |
    v               v
   Database     Return data
    |
    |--populate--> Cache
    |
    v
   Return data
```

```typescript
// Cache-aside with Upstash Redis -- the bread and butter

type CacheOptions = {
  ttl?: number;      // seconds
  prefix?: string;
};

async function cacheAside<T>(
  key: string,
  fetcher: () => Promise<T>,
  options: CacheOptions = {}
): Promise<T> {
  const { ttl = 300, prefix = 'app' } = options;
  const cacheKey = `${prefix}:${key}`;

  // Step 1: Check cache
  const cached = await redis.get<T>(cacheKey);
  if (cached !== null) {
    return cached; // Cache hit
  }

  // Step 2: Cache miss -- fetch from source
  const data = await fetcher();

  // Step 3: Populate cache for next time
  if (data !== null && data !== undefined) {
    await redis.set(cacheKey, data, { ex: ttl });
  }

  return data;
}

// Usage in an API route
export async function GET(request: Request) {
  const eventId = new URL(request.url).searchParams.get('id');

  const event = await cacheAside(
    `event:${eventId}`,
    () => db.query.events.findFirst({
      where: eq(events.id, eventId!),
      with: { venue: true, artists: true },
    }),
    { ttl: 300 } // 5 minutes
  );

  return Response.json(event);
}
```

**When to use:** This is your default. Use cache-aside for any read-heavy data where staleness of a few minutes is acceptable. Event listings, user profiles, search results, catalog data.

**Advantage:** Simple to implement. Cache only gets populated with data that's actually requested (no waste). Cache failure is graceful -- you just hit the database.

**Disadvantage:** First request for any key always hits the database (cold start penalty). Three round trips on a miss (check cache, fetch DB, write cache).

### 2.2 Write-Through

The application writes to the cache AND the database simultaneously. Reads always check the cache first.

```
                Write Path                         Read Path

   App --write--> Cache --write--> Database    App --read--> Cache
                                                |              |
                                                | miss         | hit
                                                |              |
                                                v              v
                                               Database    Return data
```

```typescript
// Write-through pattern
async function writeThrough<T extends Record<string, unknown>>(
  entity: string,
  id: string,
  data: T,
  dbWriter: (data: T) => Promise<void>
): Promise<void> {
  const cacheKey = `${entity}:${id}`;

  // Write to BOTH cache and database
  await Promise.all([
    redis.set(cacheKey, JSON.stringify(data), { ex: 3600 }),
    dbWriter(data),
  ]);
}

// Usage: updating a user profile
async function updateProfile(userId: string, updates: ProfileUpdate) {
  const updatedProfile = { ...currentProfile, ...updates, updatedAt: new Date() };

  await writeThrough(
    'profile',
    userId,
    updatedProfile,
    (data) => db.update(users).set(data).where(eq(users.id, userId))
  );

  return updatedProfile;
}
```

**When to use:** Data that is read frequently and updated occasionally, where you need the cache to always be consistent with the database. User profiles, app configuration, feature flags.

**Advantage:** Cache is always up-to-date. No cache miss after a write.

**Disadvantage:** Write latency increases (waiting for both cache and DB). If the cache write succeeds but the DB write fails, you have inconsistency. You're writing to cache even if the data is never read again.

### 2.3 Write-Behind (Write-Back)

The application writes to the cache only. The cache asynchronously writes to the database in the background.

```
                Write Path (sync)              Write Path (async)

   App --write--> Cache                Cache --batch write--> Database
                   |                            (background)
                   v
              Return immediately
```

```typescript
// Write-behind with a queue (conceptual -- use a real queue in production)
import { Queue } from 'bullmq';

const writeQueue = new Queue('db-writes', {
  connection: { host: 'redis-host', port: 6379 },
});

async function writeBehind<T>(
  entity: string,
  id: string,
  data: T
): Promise<void> {
  const cacheKey = `${entity}:${id}`;

  // Step 1: Write to cache immediately (fast)
  await redis.set(cacheKey, JSON.stringify(data), { ex: 3600 });

  // Step 2: Queue the database write (async)
  await writeQueue.add('db-write', {
    entity,
    id,
    data,
    timestamp: Date.now(),
  });

  // Returns immediately -- DB write happens in background
}

// Worker processes the queue
const worker = new Worker('db-writes', async (job) => {
  const { entity, id, data } = job.data;

  await db.insert(getTable(entity)).values(data).onConflictDoUpdate({
    target: [getTable(entity).id],
    set: data,
  });
}, {
  connection: { host: 'redis-host', port: 6379 },
});
```

**When to use:** High-throughput writes where you need low latency and can tolerate the risk of data loss if the cache crashes before the background write completes. Analytics events, view counts, activity logs. **Never use for financial transactions or critical user data.**

**Advantage:** Extremely fast writes. Can batch multiple writes into a single database operation. Great for write-heavy workloads.

**Disadvantage:** Data loss risk if cache crashes before background write. Complexity of managing the write queue. Debugging is harder because writes are async.

### 2.4 Read-Through

The cache itself is responsible for fetching data from the database on a miss. The application only talks to the cache.

```
                Read Path

   App --read--> Cache --(on miss)--> Database
                   |                      |
                   | hit                  | fetch
                   |                      |
                   v                      v
              Return data            Cache stores it
                                    then returns
```

```typescript
// Read-through pattern -- the cache handles its own misses
// This is conceptually how TanStack Query works!
class ReadThroughCache<T> {
  constructor(
    private redis: Redis,
    private fetcher: (key: string) => Promise<T | null>,
    private ttl: number = 300
  ) {}

  async get(key: string): Promise<T | null> {
    // Cache checks itself
    const cached = await this.redis.get<T>(key);
    if (cached !== null) return cached;

    // Cache fetches on miss
    const data = await this.fetcher(key);
    if (data !== null) {
      await this.redis.set(key, data, { ex: this.ttl });
    }

    return data;
  }
}

// Usage
const eventCache = new ReadThroughCache<Event>(
  redis,
  async (key) => {
    const id = key.replace('event:', '');
    return db.query.events.findFirst({ where: eq(events.id, id) });
  },
  300
);

// Application code is clean -- just reads from cache
const event = await eventCache.get(`event:${eventId}`);
```

**When to use:** When you want to abstract the caching logic away from application code. Libraries like TanStack Query implement this pattern -- your component just calls `useQuery` and the library handles cache checking, fetching, and storage.

**Advantage:** Clean application code. Caching logic is centralized. Easy to change caching strategy without modifying business logic.

**Disadvantage:** Less control over when and how fetches happen. Harder to implement custom invalidation logic.

### 2.5 Cache Warming (Pre-Population)

Proactively populate the cache before traffic arrives, rather than waiting for cache misses.

```typescript
// Cache warming -- run on deployment or on a schedule
async function warmEventCache(): Promise<void> {
  console.log('Warming event cache...');

  // Fetch the most popular events (the ones most likely to be requested)
  const popularEvents = await db.query.events.findMany({
    where: and(
      eq(events.status, 'published'),
      gte(events.startDate, new Date()),
    ),
    orderBy: desc(events.viewCount),
    limit: 500,
    with: { venue: true, artists: true },
  });

  // Populate Redis in a pipeline (batch operation)
  const pipeline = redis.pipeline();
  for (const event of popularEvents) {
    pipeline.set(`event:${event.id}`, JSON.stringify(event), { ex: 300 });
  }
  await pipeline.exec();

  console.log(`Warmed ${popularEvents.length} events in cache`);
}

// Run after deployment
// In a Next.js API route triggered by CI/CD:
export async function POST(request: Request) {
  const authHeader = request.headers.get('authorization');
  if (authHeader !== `Bearer ${process.env.DEPLOY_SECRET}`) {
    return new Response('Unauthorized', { status: 401 });
  }

  await warmEventCache();
  return Response.json({ warmed: true });
}
```

**When to use:** After deployments (to avoid cold-start cache misses on the new instance). For predictable traffic patterns (warm the cache at 8am before the morning traffic spike). For launch events where you know traffic will surge.

### 2.6 The Decision Matrix

```
+--------------------+-------------+--------------+--------------+--------------+
| Pattern            | Read Perf   | Write Perf   | Consistency  | Complexity   |
+--------------------+-------------+--------------+--------------+--------------+
| Cache-Aside        | Good*       | Good         | Eventual     | Low          |
| Write-Through      | Excellent   | Slower       | Strong       | Medium       |
| Write-Behind       | Excellent   | Excellent    | Weak         | High         |
| Read-Through       | Good*       | N/A          | Eventual     | Medium       |
| Cache Warming      | Excellent   | N/A          | Depends      | Low          |
+--------------------+-------------+--------------+--------------+--------------+
| * First request hits DB (cold miss)                                           |
+--------------------+-------------+--------------+--------------+--------------+
```

**For the TicketFlow app, here's what we'd use:**

| Data Type | Pattern | Why |
|-----------|---------|-----|
| Event listings | Cache-aside + warming | Read-heavy, staleness acceptable, warm popular events |
| User profiles | Write-through | Updated occasionally, read on every request |
| View counts | Write-behind | High frequency, eventual consistency fine |
| Ticket purchases | No caching | Financial transactions, always hit the database |
| CMS content | Read-through (ISR) | Next.js handles the read-through via ISR |
| Feature flags | Cache warming + Edge Config | Must be instant, rarely changes |

---

## 3. THE DATABASE LAYER

The bottom of the pyramid. Where truth lives.

### 3.1 Postgres Query Cache: shared_buffers

Postgres has a built-in caching layer that most developers never think about. When you execute a query, Postgres stores the result pages in `shared_buffers` -- a region of shared memory that acts as a page cache between the database files on disk and the query executor.

```sql
-- Check your shared_buffers setting
SHOW shared_buffers;
-- Default is typically 128MB. For a production server, set to 25% of available RAM.
-- On a 16GB server: shared_buffers = 4GB

-- Monitor cache hit ratio (should be > 99% for a well-tuned database)
SELECT
  sum(heap_blks_read) as heap_read,
  sum(heap_blks_hit) as heap_hit,
  sum(heap_blks_hit) / (sum(heap_blks_hit) + sum(heap_blks_read))::float as hit_ratio
FROM pg_statio_user_tables;

-- Per-table cache hit ratio (find which tables are cold)
SELECT
  relname,
  heap_blks_read as disk_reads,
  heap_blks_hit as cache_hits,
  CASE WHEN heap_blks_hit + heap_blks_read > 0
    THEN round(heap_blks_hit::numeric / (heap_blks_hit + heap_blks_read), 4)
    ELSE 0
  END as hit_ratio
FROM pg_statio_user_tables
ORDER BY disk_reads DESC;
```

**Target: 99%+ cache hit ratio.** If you're below 95%, either your shared_buffers is too small or your queries are scanning too many rows (missing indexes).

### 3.2 pg_stat_statements for Query Profiling

Before you cache application data, make sure you're not wasting database time on unoptimized queries. `pg_stat_statements` tracks execution statistics for every query.

```sql
-- Enable the extension
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

-- Find your slowest queries (by total time spent)
SELECT
  query,
  calls,
  total_exec_time::numeric(12,2) as total_ms,
  mean_exec_time::numeric(12,2) as mean_ms,
  rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;

-- Find queries with the most calls (cache candidates!)
SELECT
  query,
  calls,
  mean_exec_time::numeric(12,2) as mean_ms,
  rows
FROM pg_stat_statements
ORDER BY calls DESC
LIMIT 20;
```

The queries with the highest call count AND acceptable staleness tolerance are your prime candidates for Redis caching.

### 3.3 Neon Serverless Postgres

Neon is the database that makes sense for Vercel deployments. It auto-scales to zero, supports branching (one database per preview deployment), and has built-in connection pooling.

```typescript
// drizzle.config.ts -- Neon with connection pooling
import { neon } from '@neondatabase/serverless';
import { drizzle } from 'drizzle-orm/neon-http';

// Use the pooled connection string for serverless functions
const sql = neon(process.env.DATABASE_URL!);
export const db = drizzle(sql);

// For long-running processes (not serverless), use the direct connection
// import { Pool } from '@neondatabase/serverless';
// const pool = new Pool({ connectionString: process.env.DATABASE_URL_DIRECT });
```

**Key Neon features for caching architecture:**

1. **Branching for preview environments.** Each PR gets its own database branch. Cache warming scripts can run against the branch without affecting production data.

2. **Auto-suspend.** Neon scales to zero when idle. This means your database query cache (shared_buffers) is cold after a suspend. Design your caching strategy so that L3 (Redis) absorbs the cold-start penalty.

3. **Built-in connection pooler.** Serverless functions create and destroy connections rapidly. Neon's pooler (PgBouncer-based) sits in front of Postgres and reuses connections. Without it, each Vercel Function invocation would open and close a connection, which is catastrophically slow.

```
                          Vercel Functions
                     +---------+ +---------+
                     | fn-001  | | fn-002  |  ...100s of concurrent functions
                     +----+----+ +----+----+
                          |           |
                          v           v
                    +---------------------+
                    |  Neon Connection     |
                    |  Pooler (PgBouncer)  |  Maintains ~20 real connections
                    +----------+----------+
                               |
                               v
                    +---------------------+
                    |  Neon Postgres       |
                    |  (shared_buffers)    |
                    +---------------------+
```

### 3.4 PlanetScale (MySQL, Vitess-based)

If you're in the MySQL ecosystem, PlanetScale provides horizontal sharding via Vitess, schema-level caching, and a safe migration workflow.

```typescript
// PlanetScale with Drizzle ORM
import { drizzle } from 'drizzle-orm/planetscale-serverless';
import { Client } from '@planetscale/database';

const client = new Client({
  host: process.env.DATABASE_HOST,
  username: process.env.DATABASE_USERNAME,
  password: process.env.DATABASE_PASSWORD,
});

export const db = drizzle(client);
```

PlanetScale's Boost feature provides automatic query-level caching -- similar to a managed Redis layer, but integrated directly into the database connection. Queries that are frequently executed with the same parameters are served from an in-memory cache without hitting the MySQL query executor.

### 3.5 Drizzle ORM Query Optimization

Your ORM is the gateway to L1 and L2. Poorly written queries negate every caching layer above them. Here are the patterns that matter:

```typescript
// BAD: N+1 query -- fetches 100 events, then 100 separate venue queries
const events = await db.query.events.findMany({ limit: 100 });
for (const event of events) {
  event.venue = await db.query.venues.findFirst({
    where: eq(venues.id, event.venueId),
  });
}
// Result: 101 database queries. 101 cache misses on cold start.

// GOOD: Single query with join
const events = await db.query.events.findMany({
  limit: 100,
  with: {
    venue: true,          // Drizzle generates a JOIN
    artists: true,        // Another JOIN
  },
});
// Result: 1 database query. 1 cache entry in Redis.

// BETTER: Select only what you need
const events = await db.query.events.findMany({
  limit: 100,
  columns: {
    id: true,
    title: true,
    startDate: true,
    thumbnailUrl: true,
    // NOT selecting: description, longDescription, internalNotes, etc.
  },
  with: {
    venue: {
      columns: { name: true, city: true },  // Only venue name and city
    },
  },
});
// Result: Smaller payload = faster DB query + smaller Redis cache entry + less bandwidth
```

**Rule of thumb:** Fetch wide at L1, cache the wide result at L3, and let upper layers (TanStack Query) derive what they need. Or fetch narrow at L1, cache narrow at L3, and accept that different views might have different cache entries. The first approach wastes memory but simplifies invalidation. The second approach saves memory but complicates invalidation. For most mobile apps, fetch wide and cache wide.

### 3.6 Read Replicas for Read-Heavy Mobile Workloads

When your mobile app has 100,000+ DAU, reads vastly outnumber writes (typically 100:1 or higher). Read replicas let you scale reads horizontally while keeping writes on the primary.

```
                          Mobile App
                             |
                    +--------+--------+
                    |                 |
                Reads               Writes
                    |                 |
                    v                 v
         +------------------+  +--------------+
         |  Read Replica(s) |  |   Primary    |
         |  (multiple)      |<-|   (single)   |
         +------------------+  +--------------+
                              replication lag: ~10-100ms
```

```typescript
// Drizzle with read/write splitting
import { drizzle as drizzlePrimary } from 'drizzle-orm/neon-http';
import { neon } from '@neondatabase/serverless';

const primaryDb = drizzlePrimary(neon(process.env.DATABASE_URL_PRIMARY!));
const replicaDb = drizzlePrimary(neon(process.env.DATABASE_URL_REPLICA!));

// Read operations use replica
export async function getEvents() {
  return replicaDb.query.events.findMany({
    where: eq(events.status, 'published'),
    orderBy: desc(events.startDate),
  });
}

// Write operations use primary
export async function createEvent(data: NewEvent) {
  return primaryDb.insert(events).values(data).returning();
}
```

**Important caveat:** Replication lag means a user who just created an event might not see it in the event list for a few hundred milliseconds if the list is served from a replica. This is where **read-your-own-writes** consistency matters -- after a write, serve that user's subsequent reads from the primary (or from the Redis cache you just populated via write-through).

---

## 4. APPLICATION CACHE -- REDIS AND UPSTASH

Redis is the Swiss Army knife of your caching architecture. It's not just a key-value store -- it's a data structure server that supports strings, hashes, sorted sets, lists, streams, and more. Each data structure enables different caching patterns.

### 4.1 Redis Data Structures for Caching

```typescript
// STRING: The basic cache entry
// Use for: API response caching, serialized objects
await redis.set('event:42', JSON.stringify(eventData), { ex: 300 });
const event = await redis.get<EventData>('event:42');

// HASH: When you need partial reads/updates
// Use for: User profiles (read only the fields you need)
await redis.hset('user:99', {
  name: 'Alice',
  email: 'alice@example.com',
  plan: 'pro',
  lastSeen: Date.now().toString(),
});
// Read just the plan -- no need to deserialize the entire object
const plan = await redis.hget<string>('user:99', 'plan');
// Update just lastSeen without touching other fields
await redis.hset('user:99', { lastSeen: Date.now().toString() });

// SORTED SET: Leaderboards, rankings, time-ordered data
// Use for: Trending events, recently viewed items
await redis.zadd('trending:events', {
  score: viewCount,
  member: `event:${eventId}`,
});
// Get top 10 trending events
const trending = await redis.zrange('trending:events', 0, 9, { rev: true });

// LIST: Queues, recent items
// Use for: Activity feeds, recent searches
await redis.lpush('user:99:recent-searches', searchQuery);
await redis.ltrim('user:99:recent-searches', 0, 19); // Keep only last 20

// SET: Unique membership
// Use for: Online users, feature flag audiences
await redis.sadd('online-users', userId);
await redis.sismember('online-users', userId); // O(1) lookup
```

### 4.2 Upstash Serverless Redis

Upstash is Redis built for serverless. Two critical features:

1. **REST API:** Works in Vercel Edge Functions, which don't support TCP connections. The `@upstash/redis` package uses HTTP under the hood.
2. **Global replication:** Data is replicated to multiple regions. Reads are served from the nearest region. Writes go to the primary.

```typescript
// Works in Vercel Edge Functions (no TCP required)
import { Redis } from '@upstash/redis';

const redis = new Redis({
  url: process.env.UPSTASH_REDIS_REST_URL!,
  token: process.env.UPSTASH_REDIS_REST_TOKEN!,
});

// This runs at the edge, <5ms to read from the nearest replica
export const config = { runtime: 'edge' };

export default async function handler(request: Request) {
  const cached = await redis.get('homepage-data');
  if (cached) {
    return new Response(JSON.stringify(cached), {
      headers: { 'Content-Type': 'application/json' },
    });
  }
  // ... fetch from origin
}
```

### 4.3 Session Storage in Redis

Sessions are a perfect Redis use case: short-lived, frequently accessed, one entry per user.

```typescript
// Session management with Upstash Redis
import { nanoid } from 'nanoid';

const SESSION_TTL = 60 * 60 * 24 * 7; // 7 days

interface Session {
  userId: string;
  email: string;
  role: 'user' | 'admin' | 'event-manager';
  createdAt: number;
}

export async function createSession(user: User): Promise<string> {
  const sessionId = nanoid(32);
  const session: Session = {
    userId: user.id,
    email: user.email,
    role: user.role,
    createdAt: Date.now(),
  };

  await redis.set(`session:${sessionId}`, session, { ex: SESSION_TTL });
  return sessionId;
}

export async function getSession(sessionId: string): Promise<Session | null> {
  return redis.get<Session>(`session:${sessionId}`);
}

export async function destroySession(sessionId: string): Promise<void> {
  await redis.del(`session:${sessionId}`);
}

// Middleware: validate session on every API request
export async function withSession(request: Request): Promise<Session> {
  const token = request.headers.get('authorization')?.replace('Bearer ', '');
  if (!token) throw new Error('No session token');

  const session = await getSession(token);
  if (!session) throw new Error('Invalid or expired session');

  // Extend session TTL on activity (sliding expiration)
  await redis.expire(`session:${token}`, SESSION_TTL);

  return session;
}
```

### 4.4 Rate Limiting with Redis

Essential for protecting your mobile app's API. Two patterns:

```typescript
// Sliding window rate limiter
import { Ratelimit } from '@upstash/ratelimit';

const ratelimit = new Ratelimit({
  redis: Redis.fromEnv(),
  limiter: Ratelimit.slidingWindow(100, '60 s'), // 100 requests per 60 seconds
  analytics: true,
  prefix: 'ratelimit:api',
});

// In your API middleware
export async function rateLimitMiddleware(request: Request): Promise<Response | null> {
  const ip = request.headers.get('x-forwarded-for') ?? 'anonymous';
  const { success, limit, remaining, reset } = await ratelimit.limit(ip);

  if (!success) {
    return new Response('Too Many Requests', {
      status: 429,
      headers: {
        'X-RateLimit-Limit': limit.toString(),
        'X-RateLimit-Remaining': remaining.toString(),
        'X-RateLimit-Reset': reset.toString(),
        'Retry-After': Math.ceil((reset - Date.now()) / 1000).toString(),
      },
    });
  }

  return null; // Continue to the handler
}

// Token bucket for burst-tolerant rate limiting
const burstLimiter = new Ratelimit({
  redis: Redis.fromEnv(),
  limiter: Ratelimit.tokenBucket(10, '10 s', 30), // 10 tokens per 10s, max 30 burst
  prefix: 'ratelimit:burst',
});
```

### 4.5 The Thundering Herd Problem

This is the scenario that takes down production systems. It happens when a popular cache entry expires and hundreds of concurrent requests all miss the cache simultaneously, all hitting the database at once.

```
Normal operation:
  Request 1 --> Cache HIT --> Return (fast)
  Request 2 --> Cache HIT --> Return (fast)
  Request 3 --> Cache HIT --> Return (fast)

After cache expires (thundering herd):
  Request 1 --> Cache MISS --> Database --> (slow)
  Request 2 --> Cache MISS --> Database --> (slow)    <-- 100 concurrent
  Request 3 --> Cache MISS --> Database --> (slow)       requests all
  Request 4 --> Cache MISS --> Database --> (slow)       hitting the DB
  ...                                                      at once
  Request 100 -> Cache MISS --> Database --> (slow)
```

**Solution 1: Mutex lock (single-flight)**

Only one request fetches from the database. All other requests wait for it to finish and then read from the cache.

```typescript
async function cacheAsideWithMutex<T>(
  key: string,
  fetcher: () => Promise<T>,
  ttl: number = 300
): Promise<T> {
  // Try cache first
  const cached = await redis.get<T>(key);
  if (cached !== null) return cached;

  // Try to acquire a mutex lock
  const lockKey = `lock:${key}`;
  const acquired = await redis.set(lockKey, '1', { nx: true, ex: 10 }); // 10s lock timeout

  if (acquired) {
    // We got the lock -- we're responsible for fetching
    try {
      const data = await fetcher();
      await redis.set(key, JSON.stringify(data), { ex: ttl });
      return data;
    } finally {
      await redis.del(lockKey); // Release the lock
    }
  } else {
    // Someone else is fetching -- wait and retry
    await new Promise((resolve) => setTimeout(resolve, 100));
    const result = await redis.get<T>(key);
    if (result !== null) return result;

    // Still no data -- retry the whole thing (with a limit)
    return cacheAsideWithMutex(key, fetcher, ttl);
  }
}
```

**Solution 2: Stale-while-revalidate at the Redis level**

Serve the stale cached value immediately while refreshing in the background.

```typescript
interface CacheEntry<T> {
  data: T;
  expiresAt: number;   // When the data is considered stale
  hardExpiresAt: number; // When the data should absolutely not be served
}

async function staleWhileRevalidate<T>(
  key: string,
  fetcher: () => Promise<T>,
  options: { staleTime: number; maxStaleTime: number } = {
    staleTime: 300,
    maxStaleTime: 3600,
  }
): Promise<T> {
  const entry = await redis.get<CacheEntry<T>>(key);
  const now = Date.now() / 1000;

  if (entry) {
    if (now < entry.expiresAt) {
      // Fresh -- return immediately
      return entry.data;
    }

    if (now < entry.hardExpiresAt) {
      // Stale but within grace period -- return stale data AND revalidate in background
      revalidateInBackground(key, fetcher, options); // fire-and-forget
      return entry.data;
    }
  }

  // No cached data or hard-expired -- must fetch synchronously
  return fetchAndCache(key, fetcher, options);
}

async function revalidateInBackground<T>(
  key: string,
  fetcher: () => Promise<T>,
  options: { staleTime: number; maxStaleTime: number }
): Promise<void> {
  const lockKey = `revalidate:${key}`;
  const acquired = await redis.set(lockKey, '1', { nx: true, ex: 30 });
  if (!acquired) return; // Another instance is already revalidating

  try {
    await fetchAndCache(key, fetcher, options);
  } finally {
    await redis.del(lockKey);
  }
}

async function fetchAndCache<T>(
  key: string,
  fetcher: () => Promise<T>,
  options: { staleTime: number; maxStaleTime: number }
): Promise<T> {
  const data = await fetcher();
  const now = Date.now() / 1000;

  const entry: CacheEntry<T> = {
    data,
    expiresAt: now + options.staleTime,
    hardExpiresAt: now + options.maxStaleTime,
  };

  await redis.set(key, JSON.stringify(entry), { ex: options.maxStaleTime });
  return data;
}
```

### 4.6 Cache Stampede Prevention

Related to the thundering herd, a cache stampede happens when many keys expire at the same time (for example, if you warmed them all at the same time with the same TTL). The database gets slammed with cache misses for all keys simultaneously.

**Solution: Jittered TTL**

```typescript
function jitteredTtl(baseTtl: number, jitterPercent: number = 0.1): number {
  const jitter = baseTtl * jitterPercent;
  return Math.floor(baseTtl + Math.random() * jitter * 2 - jitter);
}

// Instead of all keys expiring at exactly 300 seconds:
await redis.set('event:1', data, { ex: jitteredTtl(300) }); // 270-330s
await redis.set('event:2', data, { ex: jitteredTtl(300) }); // 270-330s
await redis.set('event:3', data, { ex: jitteredTtl(300) }); // 270-330s
// Now they expire at slightly different times, spreading the load
```

**Solution: Probabilistic early recomputation (XFetch)**

Before a key actually expires, randomly decide to refresh it early. The probability increases as the key gets closer to expiration.

```typescript
async function xfetch<T>(
  key: string,
  fetcher: () => Promise<T>,
  ttl: number,
  beta: number = 1.0 // Higher beta = more eager early recomputation
): Promise<T> {
  const entry = await redis.get<{ data: T; delta: number; expiry: number }>(key);

  if (entry) {
    const now = Date.now() / 1000;
    const timeToExpiry = entry.expiry - now;
    const earlyExpiry = entry.delta * beta * Math.log(Math.random());

    if (timeToExpiry + earlyExpiry > 0) {
      // Not time to recompute yet
      return entry.data;
    }
    // Probabilistically decided to recompute early
  }

  const start = Date.now();
  const data = await fetcher();
  const delta = (Date.now() - start) / 1000; // How long the fetch took

  await redis.set(key, JSON.stringify({
    data,
    delta,
    expiry: Date.now() / 1000 + ttl,
  }), { ex: ttl });

  return data;
}
```

### 4.7 Pub/Sub for Cache Invalidation Across Instances

When you have multiple server instances (which you do in serverless -- every function invocation is its own instance), you need a way to signal "this cache key is invalid" to all of them.

```typescript
// Publisher: when data changes, broadcast invalidation
async function invalidateCache(tags: string[]): Promise<void> {
  // Delete from Redis
  const keys = tags.map((tag) => `cache:${tag}`);
  if (keys.length > 0) {
    await redis.del(...keys);
  }

  // Publish invalidation event (for any long-running subscribers)
  await redis.publish('cache-invalidation', JSON.stringify({
    tags,
    timestamp: Date.now(),
  }));
}

// When an event is updated:
async function updateEvent(eventId: string, data: Partial<Event>): Promise<void> {
  await db.update(events).set(data).where(eq(events.id, eventId));

  // Invalidate all caches related to this event
  await invalidateCache([
    `event:${eventId}`,
    'events:list',
    'events:trending',
    `venue:${data.venueId}:events`, // Invalidate the venue's event list too
  ]);
}
```

---

## 5. CDN EDGE LAYER

The CDN edge is your single highest-leverage caching layer. One correctly set `Cache-Control` header can eliminate 95% of requests from ever reaching your origin server. One incorrectly set header can serve stale private data to the wrong user or make every request bypass the cache entirely.

### 5.1 How CDNs Work

```
                   User in Tokyo          User in NYC          User in London
                        |                      |                      |
                        v                      v                      v
                  +-----------+          +-----------+          +-----------+
                  | Tokyo PoP |          |  NYC PoP  |          |London PoP |
                  |  (cache)  |          |  (cache)  |          |  (cache)  |
                  +-----+-----+          +-----+-----+          +-----+-----+
                        |                      |                      |
                        | cache miss           | cache miss           | cache miss
                        |                      |                      |
                        +----------+-----------+----------------------+
                                   |
                                   v
                          +-----------------+
                          |  Origin Shield  |  (optional intermediate cache)
                          +--------+--------+
                                   |
                                   v
                          +-----------------+
                          |  Origin Server  |  (your Vercel/AWS deployment)
                          +-----------------+
```

**Key concepts:**

- **PoP (Point of Presence):** A CDN server location. Vercel has 18+ edge regions. CloudFront has 400+ edge locations.
- **Origin Shield:** An intermediate cache layer between the PoPs and your origin. Instead of 18 PoPs each hitting your origin on a cache miss, they all go through the shield, which serves as a single aggregated cache.
- **Cache Key:** What the CDN uses to identify a cached response. Typically: URL + specified headers + specified cookies.

### 5.2 Cache-Control Header Masterclass

This is the section that pays for the entire book. Get these headers right and your infrastructure costs drop by an order of magnitude.

```
Cache-Control: public, max-age=0, s-maxage=3600, stale-while-revalidate=86400, stale-if-error=604800
```

Let's break down every directive:

```
+-----------------------------+----------------------------------------------+
| Directive                   | Meaning                                      |
+-----------------------------+----------------------------------------------+
| public                      | Any cache (CDN, proxy, browser) can cache    |
| private                     | Only the end user's browser can cache        |
| no-cache                    | Cache it, but MUST revalidate before use     |
| no-store                    | Do NOT cache at all. Not in memory, not disk |
| max-age=N                   | Browser: treat as fresh for N seconds        |
| s-maxage=N                  | CDN/proxy: treat as fresh for N seconds      |
|                             | (overrides max-age for shared caches)        |
| stale-while-revalidate=N    | Serve stale for N seconds while revalidating |
|                             | in the background. The CDN magic directive.   |
| stale-if-error=N            | If origin is down, serve stale for N seconds |
| immutable                   | Never revalidate. Used for hashed filenames  |
| must-revalidate             | Once stale, MUST revalidate. No stale-serve  |
| no-transform                | Don't modify the response (no compression)   |
+-----------------------------+----------------------------------------------+
```

### 5.3 Cache-Control Recipes for Mobile Apps

Here are the exact headers for different data types in a production mobile app:

```typescript
// Next.js API routes with precise Cache-Control

// RECIPE 1: Static assets (JS bundles, CSS, fonts)
// These use content-hashed filenames (_next/static/abc123.js)
// Cache forever -- the filename changes when the content changes
export function getStaticHeaders(): HeadersInit {
  return {
    'Cache-Control': 'public, max-age=31536000, immutable',
    // 1 year. Never revalidate. The URL is the cache key.
  };
}

// RECIPE 2: Event listings API (read-heavy, staleness acceptable)
export async function GET() {
  const events = await getEvents();
  return Response.json(events, {
    headers: {
      'Cache-Control': 'public, s-maxage=60, stale-while-revalidate=3600',
      // CDN: fresh for 60 seconds.
      // After 60s: serve stale for up to 1 hour while revalidating in background.
      // User sees instant response. CDN asynchronously fetches fresh data.
    },
  });
}

// RECIPE 3: User-specific data (must NOT be cached on CDN)
export async function GET(request: Request) {
  const session = await getSession(request);
  const profile = await getProfile(session.userId);
  return Response.json(profile, {
    headers: {
      'Cache-Control': 'private, max-age=60, stale-while-revalidate=300',
      // private: CDN must NOT cache this
      // Browser can cache for 60 seconds
      // Browser can serve stale for 5 minutes while revalidating
    },
  });
}

// RECIPE 4: Financial data (never cache)
export async function GET(request: Request) {
  const balance = await getWalletBalance(request);
  return Response.json(balance, {
    headers: {
      'Cache-Control': 'no-store',
      // Not cached anywhere. Ever.
    },
  });
}

// RECIPE 5: CMS content (ISR-managed)
// In Next.js, ISR handles this automatically via revalidate:
export const revalidate = 3600; // Re-generate every hour
// This sets s-maxage=3600 and stale-while-revalidate automatically

// RECIPE 6: Images from your CDN (immutable with content hash)
export function getImageHeaders(etag: string): HeadersInit {
  return {
    'Cache-Control': 'public, max-age=31536000, immutable',
    'ETag': etag,
    // Images served from /images/abc123/event-poster.webp
    // The abc123 is a content hash -- changes when the image changes
  };
}

// RECIPE 7: API responses that need conditional caching
export async function GET(request: Request) {
  const data = await getEventDetails(eventId);
  const etag = generateETag(data);

  // Check if the client already has this version
  if (request.headers.get('if-none-match') === etag) {
    return new Response(null, { status: 304 }); // Not modified -- zero bytes transferred
  }

  return Response.json(data, {
    headers: {
      'Cache-Control': 'public, max-age=0, must-revalidate',
      'ETag': etag,
    },
  });
}
```

### 5.4 The Vary Header

The `Vary` header tells the CDN to cache different versions of the same URL based on request headers. This is critical for serving different content to different users from the same URL.

```typescript
// Vary on Accept-Language: cache one version per language
return Response.json(localizedContent, {
  headers: {
    'Cache-Control': 'public, s-maxage=3600',
    'Vary': 'Accept-Language',
    // CDN stores: /events?vary=en, /events?vary=fr, /events?vary=ja
  },
});

// Vary on Accept: serve JSON to mobile, HTML to browsers
return new Response(body, {
  headers: {
    'Cache-Control': 'public, s-maxage=3600',
    'Vary': 'Accept',
    // CDN stores: /events?vary=application/json, /events?vary=text/html
  },
});
```

**Warning:** `Vary: Authorization` or `Vary: Cookie` effectively disables CDN caching because every user has a different value. If you need per-user caching, use `Cache-Control: private` instead (caches only in the user's browser, not at the CDN).

### 5.5 Cache Keys and Key Normalization

CDNs use the full URL (including query parameters) as the cache key by default. This means `/events?page=1&sort=date` and `/events?sort=date&page=1` are two different cache entries even though they return the same data.

```typescript
// Normalize query parameters in your API to improve cache hit ratio
export async function GET(request: Request) {
  const url = new URL(request.url);

  // Sort parameters alphabetically
  const params = new URLSearchParams([...url.searchParams.entries()].sort());

  // Strip tracking parameters that don't affect the response
  params.delete('utm_source');
  params.delete('utm_medium');
  params.delete('utm_campaign');
  params.delete('fbclid');
  params.delete('gclid');

  // Redirect to canonical URL if parameters were reordered
  const canonical = `${url.pathname}?${params.toString()}`;
  if (url.search !== `?${params.toString()}`) {
    return Response.redirect(new URL(canonical, url.origin), 301);
  }

  // ... serve response
}
```

### 5.6 Purging and Invalidation

When data changes, you need to tell the CDN to drop its cached copy.

**Path-based purge:**
```typescript
// Purge a specific path via the Vercel API
await fetch('https://api.vercel.com/v1/projects/your-project/purge', {
  method: 'POST',
  headers: { Authorization: `Bearer ${process.env.VERCEL_TOKEN}` },
  body: JSON.stringify({ paths: ['/api/events/42', '/events/42'] }),
});
```

**Tag-based purge (Surrogate-Key pattern):**

This is the most powerful invalidation mechanism. Tag your responses with semantic labels, then purge by tag.

```typescript
// When serving an event response, tag it
export async function GET(
  request: Request,
  { params }: { params: { id: string } }
) {
  const event = await getEvent(params.id);
  return Response.json(event, {
    headers: {
      'Cache-Control': 'public, s-maxage=3600',
      // Tag this response for later invalidation
      'Cache-Tag': `event-${params.id}, venue-${event.venueId}, events-all`,
    },
  });
}

// When an event is updated, purge by tag
// In Next.js App Router:
import { revalidateTag } from 'next/cache';

export async function POST(request: Request) {
  const { eventId, venueId } = await request.json();

  // This purges ALL cached responses tagged with 'event-42'
  revalidateTag(`event-${eventId}`);

  // Also purge the event list
  revalidateTag('events-all');

  return Response.json({ revalidated: true });
}
```

### 5.7 ISR (Incremental Static Regeneration) as a CDN Pattern

ISR is the best of both worlds: the performance of static sites with the freshness of server-rendered pages. Under the hood, it's a CDN caching pattern with on-demand revalidation.

```typescript
// app/events/[id]/page.tsx -- Next.js App Router with ISR

// Revalidate this page every 60 seconds
export const revalidate = 60;

// Pre-generate pages for popular events at build time
export async function generateStaticParams() {
  const popularEvents = await db.query.events.findMany({
    where: eq(events.status, 'published'),
    orderBy: desc(events.viewCount),
    limit: 100,
    columns: { id: true },
  });
  return popularEvents.map((e) => ({ id: e.id }));
}

export default async function EventPage({
  params,
}: {
  params: { id: string };
}) {
  const event = await getEvent(params.id);
  if (!event) notFound();
  return <EventDetails event={event} />;
}
```

**How ISR works at the CDN level:**

```
Time 0:00 -- First request
  CDN: Cache MISS -> Origin renders page -> CDN caches with s-maxage=60
  User gets: fresh page (~200ms)

Time 0:30 -- Second request (within revalidate window)
  CDN: Cache HIT
  User gets: cached page (~10ms)

Time 1:01 -- Request after revalidate window
  CDN: Cache STALE -> Serves stale to user -> Triggers background revalidation
  User gets: stale page instantly (~10ms)
  Background: Origin re-renders -> CDN updates cache

Time 1:02 -- Next request
  CDN: Cache HIT (fresh from background revalidation)
  User gets: fresh page (~10ms)
```

**On-demand ISR via webhooks (the CMS pattern):**

```typescript
// app/api/revalidate/route.ts
import { revalidateTag, revalidatePath } from 'next/cache';

export async function POST(request: Request) {
  // Verify the webhook signature
  const body = await request.text();
  const signature = request.headers.get('x-sanity-webhook-signature');
  if (!verifyWebhookSignature(signature, body)) {
    return new Response('Invalid signature', { status: 401 });
  }

  const payload = JSON.parse(body);

  // Sanity webhook payload tells us what changed
  if (payload._type === 'event') {
    revalidateTag(`event-${payload._id}`);
    revalidateTag('events-list');
    revalidatePath('/events');
    revalidatePath(`/events/${payload.slug?.current}`);
  }

  if (payload._type === 'venue') {
    revalidateTag(`venue-${payload._id}`);
    revalidateTag('events-list'); // Events show venue info
  }

  return Response.json({ revalidated: true, timestamp: Date.now() });
}
```

### 5.8 Edge Config for Ultra-Fast Reads

Vercel Edge Config is a globally replicated key-value store optimized for reads that complete in under 1ms. Perfect for data that is read on every request but changes rarely.

```typescript
import { get } from '@vercel/edge-config';

// These reads are sub-millisecond -- faster than Redis
export async function middleware(request: NextRequest) {
  // Feature flags
  const newCheckoutEnabled = await get<boolean>('feature-new-checkout');

  // A/B test assignments
  const experimentConfig = await get<ExperimentConfig>('experiment-homepage-v2');

  // Redirect rules
  const redirects = await get<RedirectRule[]>('redirects');
  const redirect = redirects?.find(
    (r) => request.nextUrl.pathname === r.source
  );
  if (redirect) {
    return NextResponse.redirect(new URL(redirect.destination, request.url));
  }

  // Maintenance mode
  const maintenance = await get<boolean>('maintenance-mode');
  if (maintenance) {
    return NextResponse.rewrite(new URL('/maintenance', request.url));
  }

  return NextResponse.next();
}
```

**Use Edge Config for:** Feature flags, A/B test configuration, redirect maps, maintenance mode, blocked IP lists. Anything that is read on every request and updated via dashboard or API.

**Do NOT use for:** User-specific data, high-cardinality data (more than ~10MB), data that changes more than once per second.

---

## 6. HTTP CACHE -- THE CLIENT SIDE

The HTTP cache is the invisible cache layer that most mobile developers don't think about. It sits between your app's networking code and the actual network request, and it can serve responses without any network activity.

### 6.1 How HTTP Caching Works on Mobile

```
         Your React Native App
                  |
                  v
         +------------------+
         |  fetch() / axios |
         +--------+---------+
                  |
                  v
         +------------------+
         |   HTTP Cache     |  <-- NSURLCache (iOS) / OkHttp cache (Android)
         |   (on device)    |
         +--------+---------+
                  |
           Is the cached response
           still fresh?
                  |
          +-------+-------+
          |               |
         YES             NO
          |               |
          v               v
    Return cached    Send request to
    response         server (possibly
    (~1ms)           with If-None-Match
                     for conditional request)
```

**iOS (NSURLCache):** React Native uses NSURLSession under the hood. iOS maintains a shared URL cache that respects Cache-Control headers. The default cache size is 20MB in memory and 200MB on disk.

**Android (OkHttp):** React Native on Android uses OkHttp, which has a built-in HTTP cache. By default, it may not be configured. You need to set it up.

### 6.2 ETag and Conditional Requests

ETags are the most bandwidth-efficient caching mechanism for mobile apps on cellular connections. Instead of downloading the full response, the client asks "has this changed since the last time I asked?"

```typescript
// Server-side: generate and check ETags
import { createHash } from 'crypto';

function generateETag(data: unknown): string {
  const hash = createHash('md5').update(JSON.stringify(data)).digest('hex');
  return `"${hash}"`;
}

export async function GET(request: Request) {
  const events = await getEvents();
  const etag = generateETag(events);

  // Check if client already has this exact data
  const clientEtag = request.headers.get('if-none-match');
  if (clientEtag === etag) {
    // Client has the latest version -- return 304 with no body
    return new Response(null, {
      status: 304,
      headers: { 'ETag': etag },
    });
    // Bandwidth saved: ~0 bytes instead of ~50KB
  }

  // Client needs the data -- send full response
  return Response.json(events, {
    headers: {
      'ETag': etag,
      'Cache-Control': 'public, max-age=0, must-revalidate',
    },
  });
}
```

**Bandwidth savings on cellular:** A typical event listing API response is 30-50KB. A 304 response is ~200 bytes (just headers). If the data hasn't changed (which is 90%+ of the time for a mobile app polling for updates), you've reduced bandwidth by 99.5%. On a metered cellular connection, this adds up fast.

### 6.3 React Native Fetch and HTTP Caching Behavior

React Native's `fetch` implementation varies by platform:

```typescript
// iOS: NSURLSession respects Cache-Control headers by default
// Android: OkHttp respects Cache-Control headers IF the cache is configured

// To explicitly control caching behavior:
const response = await fetch(url, {
  // 'default' -- standard HTTP caching behavior
  // 'no-cache' -- always revalidate with server (conditional request)
  // 'no-store' -- bypass cache entirely
  // 'force-cache' -- use cached response even if stale
  // 'only-if-cached' -- return cached response or fail
  cache: 'default',
});

// For TanStack Query, the HTTP cache is transparent:
// TanStack Query calls fetch -> fetch checks HTTP cache -> maybe hits network
// This means you get TWO layers of caching: TanStack Query (L7) + HTTP cache (L5)
```

**Gotcha:** When using TanStack Query with a short `staleTime`, the query will refetch when it becomes stale. But if the HTTP cache has a valid cached response, the refetch returns instantly from the HTTP cache without hitting the network. This is *desirable* behavior -- you get the freshness checks of TanStack Query with the bandwidth savings of HTTP caching.

---

## 7. TANSTACK QUERY AS A CACHE LAYER

Chapter 11 covered TanStack Query's API. This section covers TanStack Query as an *architectural caching layer* -- how to think about it in the context of the full pyramid.

### 7.1 staleTime as Your "Freshness Budget"

`staleTime` is the most important configuration value in your entire caching architecture. It answers the question: **how stale is this data allowed to be before we should check for updates?**

```typescript
// Configure staleTime per data type -- this is your freshness policy
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 1000 * 60 * 5, // 5 minutes default -- good for most data
      gcTime: 1000 * 60 * 30,   // 30 minutes in memory after last use
    },
  },
});

// Override per query for different freshness requirements
function useEventList() {
  return useQuery({
    queryKey: ['events', 'list'],
    queryFn: fetchEventList,
    staleTime: 1000 * 60 * 5, // 5 minutes -- event lists don't change often
  });
}

function useTicketAvailability(eventId: string) {
  return useQuery({
    queryKey: ['events', eventId, 'availability'],
    queryFn: () => fetchAvailability(eventId),
    staleTime: 1000 * 30, // 30 seconds -- ticket availability changes fast
  });
}

function useUserProfile() {
  return useQuery({
    queryKey: ['user', 'profile'],
    queryFn: fetchProfile,
    staleTime: 1000 * 60 * 60, // 1 hour -- user's own profile rarely changes
  });
}

function useWalletBalance() {
  return useQuery({
    queryKey: ['user', 'wallet'],
    queryFn: fetchWalletBalance,
    staleTime: 0, // ALWAYS refetch -- financial data must be fresh
  });
}
```

### 7.2 gcTime as Your "Memory Budget"

`gcTime` (garbage collection time) controls how long TanStack Query keeps data in memory *after the last component using it unmounts.* This is your memory budget.

```typescript
// Memory-conscious configuration for mobile
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      // Default: keep in memory for 30 minutes after last use
      gcTime: 1000 * 60 * 30,
    },
  },
});

// For large data (like search results with images)
function useSearchResults(query: string) {
  return useQuery({
    queryKey: ['search', query],
    queryFn: () => search(query),
    gcTime: 1000 * 60 * 5, // Only 5 minutes -- search results are large
  });
}

// For data you want to keep around (user's own profile)
function useProfile() {
  return useQuery({
    queryKey: ['user', 'profile'],
    queryFn: fetchProfile,
    gcTime: Infinity, // Never garbage collect -- always in memory while app is open
  });
}
```

**Mobile memory math:** The average TanStack Query cache entry is 1-10KB (serialized JSON). At 5KB per entry and 200 entries, that's 1MB -- negligible. But if you're caching image URLs that trigger in-memory image caches, or caching paginated lists with thousands of items, the memory footprint grows fast. Profile your app's memory on a low-end device.

### 7.3 Background Refetch as "Eventual Consistency"

TanStack Query's background refetching is an implementation of the stale-while-revalidate pattern at the application level.

```
User opens Events screen (data is stale):

  Step 1: Return stale data instantly (from L7 in-memory cache)
  Step 2: Show stale data to user (no loading spinner!)
  Step 3: Fetch fresh data in background
  Step 4: When fresh data arrives, update the UI seamlessly

  User's experience: instant load, data might update slightly after 1-2 seconds
```

```typescript
function EventList() {
  const { data: events, isStale, isFetching } = useQuery({
    queryKey: ['events', 'list'],
    queryFn: fetchEventList,
    staleTime: 1000 * 60 * 5,
  });

  return (
    <View>
      {/* Optional: subtle indicator that data is refreshing */}
      {isFetching && isStale && (
        <View style={styles.refreshIndicator}>
          <ActivityIndicator size="small" />
        </View>
      )}

      {/* Data is shown immediately -- never a loading spinner on revisit */}
      <FlashList
        data={events}
        renderItem={({ item }) => <EventCard event={item} />}
        estimatedItemSize={120}
      />
    </View>
  );
}
```

### 7.4 Query Key Hierarchy as "Cache Addressing"

Query keys are not just identifiers -- they're a hierarchical addressing scheme for your cache.

```typescript
// Query key hierarchy for events:
// ['events']                           All events (namespace root)
// ['events', 'list']                   Event list
// ['events', 'list', { page: 1 }]     Event list, page 1
// ['events', 'list', { page: 2 }]     Event list, page 2
// ['events', '42']                     Single event #42
// ['events', '42', 'availability']     Event #42 ticket availability
// ['events', '42', 'reviews']          Event #42 reviews

// Invalidation uses this hierarchy:
// Invalidate everything under 'events':
queryClient.invalidateQueries({ queryKey: ['events'] });
// This invalidates ALL of the above -- list, every event, availability, reviews

// Invalidate just one event and its sub-queries:
queryClient.invalidateQueries({ queryKey: ['events', '42'] });
// Invalidates: ['events', '42'], availability, reviews
// Does NOT invalidate: ['events', 'list'], ['events', '43']

// Invalidate just the list (not individual events):
queryClient.invalidateQueries({ queryKey: ['events', 'list'], exact: true });
```

This hierarchy is your in-memory cache's addressing scheme. Design it intentionally:

```typescript
// The TicketFlow query key factory
export const queryKeys = {
  events: {
    all: ['events'] as const,
    lists: () => [...queryKeys.events.all, 'list'] as const,
    list: (filters: EventFilters) =>
      [...queryKeys.events.lists(), filters] as const,
    details: () => [...queryKeys.events.all, 'detail'] as const,
    detail: (id: string) => [...queryKeys.events.details(), id] as const,
    availability: (id: string) =>
      [...queryKeys.events.detail(id), 'availability'] as const,
  },
  user: {
    all: ['user'] as const,
    profile: () => [...queryKeys.user.all, 'profile'] as const,
    wallet: () => [...queryKeys.user.all, 'wallet'] as const,
    tickets: () => [...queryKeys.user.all, 'tickets'] as const,
    ticket: (id: string) => [...queryKeys.user.tickets(), id] as const,
  },
  venues: {
    all: ['venues'] as const,
    detail: (id: string) => [...queryKeys.venues.all, id] as const,
    events: (id: string) =>
      [...queryKeys.venues.detail(id), 'events'] as const,
  },
} as const;

// Usage
useQuery({
  queryKey: queryKeys.events.detail('42'),
  queryFn: () => fetchEvent('42'),
});

// Invalidation after a ticket purchase
queryClient.invalidateQueries({ queryKey: queryKeys.events.availability('42') });
queryClient.invalidateQueries({ queryKey: queryKeys.user.wallet() });
queryClient.invalidateQueries({ queryKey: queryKeys.user.tickets() });
```

### 7.5 Optimistic Updates as "Write-Through to the UI Layer"

Optimistic updates are the write-through pattern applied to L7 (in-memory cache). Instead of waiting for the server to confirm a mutation, you update the cache immediately and roll back if the server rejects it.

```typescript
function useLikeEvent() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (eventId: string) =>
      fetch(`/api/events/${eventId}/like`, { method: 'POST' }),

    // Optimistic update: modify cache immediately
    onMutate: async (eventId) => {
      // Cancel in-flight refetches (so they don't overwrite our optimistic update)
      await queryClient.cancelQueries({
        queryKey: queryKeys.events.detail(eventId),
      });

      // Snapshot the current state (for rollback)
      const previous = queryClient.getQueryData<Event>(
        queryKeys.events.detail(eventId)
      );

      // Optimistically update the cache
      queryClient.setQueryData<Event>(
        queryKeys.events.detail(eventId),
        (old) => {
          if (!old) return old;
          return { ...old, isLiked: true, likeCount: old.likeCount + 1 };
        }
      );

      return { previous }; // Return context for rollback
    },

    // If the mutation fails, roll back to the previous state
    onError: (_error, eventId, context) => {
      if (context?.previous) {
        queryClient.setQueryData(
          queryKeys.events.detail(eventId),
          context.previous
        );
      }
    },

    // After success or error, always refetch to ensure consistency
    onSettled: (_data, _error, eventId) => {
      queryClient.invalidateQueries({
        queryKey: queryKeys.events.detail(eventId),
      });
    },
  });
}
```

### 7.6 Persistence to MMKV as "L2 Cache" for TanStack Query

The most powerful pattern in the mobile caching pyramid: persist TanStack Query's in-memory cache to MMKV. On cold start, hydrate the query client from MMKV before any network request. The user sees their data immediately.

```typescript
// lib/query-persistence.ts
import { createSyncStoragePersister } from '@tanstack/query-sync-storage-persister';
import { MMKV } from 'react-native-mmkv';

const storage = new MMKV({ id: 'query-cache' });

// MMKV adapter for TanStack Query persistence
const mmkvStorage = {
  getItem: (key: string): string | null => {
    return storage.getString(key) ?? null;
  },
  setItem: (key: string, value: string): void => {
    storage.set(key, value);
  },
  removeItem: (key: string): void => {
    storage.delete(key);
  },
};

export const queryPersister = createSyncStoragePersister({
  storage: mmkvStorage,
  // Only persist queries with gcTime > 0 and that succeeded
  // Reduce serialization overhead by throttling
  throttleTime: 2000, // Write to MMKV at most every 2 seconds
});

// App.tsx -- wrap with PersistQueryClientProvider
import { PersistQueryClientProvider } from '@tanstack/react-query-persist-client';

export default function App() {
  return (
    <PersistQueryClientProvider
      client={queryClient}
      persistOptions={{
        persister: queryPersister,
        maxAge: 1000 * 60 * 60 * 24, // 24 hours -- don't restore data older than this
        dehydrateOptions: {
          shouldDehydrateQuery: (query) => {
            // Only persist successful queries
            if (query.state.status !== 'success') return false;
            // Don't persist sensitive data
            if (
              query.queryKey[0] === 'user' &&
              query.queryKey[1] === 'wallet'
            )
              return false;
            // Don't persist very large responses
            const dataSize = JSON.stringify(query.state.data).length;
            if (dataSize > 100_000) return false; // Skip entries > 100KB
            return true;
          },
        },
      }}
    >
      <Navigation />
    </PersistQueryClientProvider>
  );
}
```

**The cold start experience:**

```
Without MMKV persistence:
  App opens -> loading spinner (2-3 seconds) -> data appears

With MMKV persistence:
  App opens -> cached data appears instantly (~100ms) -> background refetch -> data updates silently
```

This is why users perceive some apps as "instant" -- they're showing cached data from the last session while fetching fresh data in the background.

---

## 8. DEVICE STORAGE (MMKV) AS THE FINAL CACHE

### 8.1 Hydrating the Query Client on Cold Start

The MMKV persistence from Section 7.6 handles this automatically, but understanding the flow is important:

```
Cold Start Timeline:

  0ms     App process created, JS bundle loads
  ~100ms  React tree mounts, PersistQueryClientProvider initializes
  ~120ms  MMKV reads cached query data from disk (~1ms for up to 1MB)
  ~130ms  QueryClient is hydrated -- all persisted queries are in memory
  ~150ms  Components render with cached data (no loading spinners!)
  ~200ms  App is visually ready with data from last session
  ~300ms  Background refetches begin for stale queries
  ~500ms  First refetch responses arrive -- UI updates silently
  ~1000ms All critical data is fresh

  Total time to useful data: ~150ms (vs ~2000ms without persistence)
```

### 8.2 Why This Makes Your App Feel Instant

The human perception of speed is not about how fast your data arrives -- it's about how fast the user sees *something useful.* Showing cached data from the last session is vastly better than showing a loading spinner, even if the cached data is slightly stale.

```typescript
// Example: A user opens the TicketFlow app at 9am
// They last used it at 11pm the night before (10 hours ago)

// The events list they see at 9am:
// - Shows the events from last night's cache (10 hours stale)
// - Most events haven't changed (concerts are still scheduled)
// - Background refetch updates the list in ~500ms
// - Any new events appear, any cancelled events disappear
// - The user barely notices the update

// This is DRAMATICALLY better than:
// - Empty screen -> loading spinner -> 2 seconds -> events appear
```

### 8.3 Image Disk Cache (expo-image)

Images are the heaviest assets in a mobile app. expo-image provides a disk cache with automatic management.

```typescript
import { Image } from 'expo-image';

// expo-image handles disk caching automatically
// Default cache budget: ~200MB on disk
function EventPoster({ event }: { event: Event }) {
  return (
    <Image
      source={{ uri: event.posterUrl }}
      style={styles.poster}
      contentFit="cover"
      // Transition effect when loading (no flash)
      transition={200}
      // Memory cache for recently viewed images
      cachePolicy="memory-disk"
      // Placeholder while loading (from a tiny base64 preview)
      placeholder={event.posterBlurhash}
    />
  );
}
```

**How expo-image caching works:**

```
First view of event poster:
  1. Check memory cache -> MISS
  2. Check disk cache -> MISS
  3. Download from CDN -> ~500ms (depending on image size and network)
  4. Store in disk cache
  5. Store in memory cache
  6. Display image

Second view (same session):
  1. Check memory cache -> HIT -> display instantly (~0ms)

Next session (cold start):
  1. Check memory cache -> MISS (app was killed)
  2. Check disk cache -> HIT -> display from disk (~5ms)
```

### 8.4 Clearing Cache: When and How

Cache clearing should be deliberate, not a panicked reaction to bugs.

```typescript
// User-triggered cache clear (Settings screen)
async function clearAllCaches(): Promise<void> {
  // 1. Clear TanStack Query in-memory cache
  queryClient.clear();

  // 2. Clear MMKV persisted cache
  const queryStorage = new MMKV({ id: 'query-cache' });
  queryStorage.clearAll();

  // 3. Clear image cache
  await Image.clearDiskCache();
  await Image.clearMemoryCache();

  // 4. Do NOT clear auth tokens or user preferences
  // Those are in separate MMKV instances

  console.log('All caches cleared');
}

// Automatic cache clear on major app update
async function handleAppUpdate(
  previousVersion: string,
  currentVersion: string
): Promise<void> {
  const previousMajor = parseInt(previousVersion.split('.')[0]);
  const currentMajor = parseInt(currentVersion.split('.')[0]);

  if (currentMajor > previousMajor) {
    // Major version bump -- cache format might have changed
    await clearAllCaches();
  }
}

// Automatic cache clear on logout
async function logout(): Promise<void> {
  await clearAllCaches();

  // Also clear auth-specific storage
  const authStorage = new MMKV({ id: 'auth' });
  authStorage.clearAll();
  await SecureStore.deleteItemAsync('refreshToken');
}
```

---

## 9. CMS AND CONTENT MANAGEMENT FOR MOBILE

Most mobile apps have content that isn't purely dynamic user data -- marketing pages, help articles, terms of service, event descriptions with rich formatting, localized strings. This content lives in a CMS and flows through a specific pipeline to reach the user's device.

### 9.1 Headless CMS Options

A headless CMS provides content via API (no rendering opinion). Your React Native app and Next.js web app consume the same content API but render differently.

```
+-------------------+--------------+--------------+--------------+-------------+
| CMS               | Best For     | Pricing      | Content API  | Mobile Fit  |
+-------------------+--------------+--------------+--------------+-------------+
| Sanity            | Structured   | Free tier +  | GROQ + REST  | Excellent   |
|                   | content,     | per-request  | + GraphQL    | (JS client  |
|                   | real-time    |              |              | works in RN)|
|                   | collaboration|              |              |             |
+-------------------+--------------+--------------+--------------+-------------+
| Contentful        | Enterprise,  | Per-space +  | REST +       | JS SDK      |
|                   | localization | per-user     | GraphQL      |             |
+-------------------+--------------+--------------+--------------+-------------+
| Payload CMS       | Self-hosted, | Open source  | REST +       | JS client   |
|                   | full control | (free)       | GraphQL      |             |
+-------------------+--------------+--------------+--------------+-------------+
| Strapi            | Self-hosted, | Open source  | REST +       | JS SDK      |
|                   | customizable | (free)       | GraphQL      |             |
+-------------------+--------------+--------------+--------------+-------------+
| Hygraph           | GraphQL-     | Per-seat +   | GraphQL      | Any GQL     |
|                   | native       | per-request  | native       | client      |
+-------------------+--------------+--------------+--------------+-------------+
```

**For the TicketFlow app, I'd choose Sanity.** Here's why:
- GROQ (Graph-Relational Object Queries) is perfect for fetching exactly the content shape you need
- Real-time collaboration for event managers editing event descriptions
- Portable Text for rich content that renders differently on web vs mobile
- Generous free tier for startups
- Webhook support for ISR invalidation

### 9.2 Content Pipeline: CMS to Mobile App

```
+--------------+      +--------------+      +--------------+      +---------------+
|   Editor     |      |   Sanity     |      |  Webhook     |      |  Next.js API  |
|   publishes  |----->|   Content    |----->|  fires       |----->|  revalidate   |
|   in CMS     |      |   Lake       |      |              |      |  handler      |
+--------------+      +--------------+      +--------------+      +-------+-------+
                                                                          |
                                                                          v
                                                               +------------------+
                                                               |  revalidateTag() |
                                                               |  ISR regenerates |
                                                               +--------+---------+
                                                                        |
                                                                        v
                                                               +------------------+
                                                               |  CDN caches new  |
                                                               |  version         |
                                                               +--------+---------+
                                                                        |
                                              +-------------------------+
                                              |                         |
                                              v                         v
                                    +------------------+     +------------------+
                                    |  Web user visits |     |  Mobile app      |
                                    |  Gets fresh ISR  |     |  TanStack Query  |
                                    |  page from CDN   |     |  refetches       |
                                    +------------------+     +--------+---------+
                                                                      |
                                                                      v
                                                             +------------------+
                                                             |  MMKV persists   |
                                                             |  updated cache   |
                                                             +------------------+
```

### 9.3 Fetching CMS Content for Mobile

```typescript
// lib/sanity.ts -- Sanity client shared between web and mobile
import { createClient } from '@sanity/client';

export const sanityClient = createClient({
  projectId: process.env.EXPO_PUBLIC_SANITY_PROJECT_ID!,
  dataset: 'production',
  apiVersion: '2026-04-01',
  useCdn: true, // Use Sanity's CDN for reads (faster, eventually consistent)
});

// Fetching event content with GROQ
export async function getEventContent(slug: string) {
  return sanityClient.fetch(
    `*[_type == "event" && slug.current == $slug][0]{
      title,
      description,
      "posterUrl": poster.asset->url,
      "posterBlurhash": poster.asset->metadata.blurhash,
      body[]{
        ...,
        _type == "image" => {
          ...,
          "url": asset->url,
          "dimensions": asset->metadata.dimensions,
        }
      },
      artists[]->{
        name,
        "photoUrl": photo.asset->url,
      },
      venue->{
        name,
        address,
        "mapUrl": map.asset->url,
      },
      schedule[]{
        time,
        stage,
        artist->{name},
      },
    }`,
    { slug }
  );
}

// In the mobile app -- cache CMS content with TanStack Query
function useEventContent(slug: string) {
  return useQuery({
    queryKey: ['cms', 'event', slug],
    queryFn: () => getEventContent(slug),
    staleTime: 1000 * 60 * 15, // 15 minutes -- CMS content changes infrequently
    gcTime: 1000 * 60 * 60,    // 1 hour in memory
  });
}
```

### 9.4 Rendering Rich Content in Mobile Apps

CMS content often includes rich text with embedded images, links, and formatted text. On web, this renders as HTML. On mobile, you need to render as native components.

```typescript
// Rendering Sanity Portable Text in React Native
import {
  PortableText,
  type PortableTextComponents,
} from '@portabletext/react-native';
import { Text, View, Linking } from 'react-native';
import { Image } from 'expo-image';

const portableTextComponents: PortableTextComponents = {
  block: {
    h1: ({ children }) => <Text style={styles.h1}>{children}</Text>,
    h2: ({ children }) => <Text style={styles.h2}>{children}</Text>,
    normal: ({ children }) => <Text style={styles.body}>{children}</Text>,
    blockquote: ({ children }) => (
      <View style={styles.blockquote}>
        <Text style={styles.blockquoteText}>{children}</Text>
      </View>
    ),
  },
  marks: {
    strong: ({ children }) => <Text style={styles.bold}>{children}</Text>,
    em: ({ children }) => <Text style={styles.italic}>{children}</Text>,
    link: ({ value, children }) => (
      <Text
        style={styles.link}
        onPress={() => Linking.openURL(value?.href)}
      >
        {children}
      </Text>
    ),
  },
  types: {
    image: ({ value }) => (
      <Image
        source={{ uri: `${value.url}?w=800&auto=format` }}
        style={[
          styles.inlineImage,
          {
            aspectRatio:
              value.dimensions.width / value.dimensions.height,
          },
        ]}
        contentFit="contain"
        placeholder={value.blurhash}
        transition={200}
      />
    ),
  },
};

function EventDescription({ body }: { body: PortableTextBlock[] }) {
  return (
    <View style={styles.content}>
      <PortableText value={body} components={portableTextComponents} />
    </View>
  );
}
```

### 9.5 Content Preview for Editors

Editors need to see their changes before publishing. This requires a "draft mode" that bypasses the cache.

```typescript
// Next.js: Draft mode for CMS previews
// app/api/preview/route.ts
import { draftMode } from 'next/headers';

export async function GET(request: Request) {
  const { searchParams } = new URL(request.url);
  const secret = searchParams.get('secret');
  const slug = searchParams.get('slug');

  // Validate the secret
  if (secret !== process.env.SANITY_PREVIEW_SECRET) {
    return new Response('Invalid token', { status: 401 });
  }

  // Enable draft mode
  const draft = await draftMode();
  draft.enable();

  // Redirect to the page being previewed
  return Response.redirect(new URL(`/events/${slug}`, request.url));
}

// In the page component, detect draft mode
export default async function EventPage({
  params,
}: {
  params: { slug: string };
}) {
  const { isEnabled: isDraft } = await draftMode();

  // In draft mode: fetch directly from Sanity (no CDN cache, includes drafts)
  // In production: fetch from CDN (cached, published only)
  const event = isDraft
    ? await sanityClient.fetch(query, { slug: params.slug }, {
        perspective: 'previewDrafts',
      })
    : await sanityClient.fetch(query, { slug: params.slug });

  return (
    <>
      {isDraft && <DraftBanner />}
      <EventDetails event={event} />
    </>
  );
}
```

**For mobile preview:** Use a deep link that enables draft mode in the app.

```typescript
// React Native: deep link handler for CMS preview
// ticketflow://preview?slug=summer-festival-2026&token=abc123
function usePreviewMode() {
  const [isPreview, setIsPreview] = useState(false);

  useEffect(() => {
    const subscription = Linking.addEventListener('url', ({ url }) => {
      const parsed = new URL(url);
      if (parsed.pathname === '/preview') {
        const token = parsed.searchParams.get('token');
        if (token === PREVIEW_TOKEN) {
          setIsPreview(true);
          // Navigate to the content being previewed
          const slug = parsed.searchParams.get('slug');
          router.push(`/events/${slug}`);
        }
      }
    });
    return () => subscription.remove();
  }, []);

  return isPreview;
}
```

### 9.6 Multi-Language Content (i18n from CMS)

For apps serving multiple markets, the CMS handles content translation and your caching strategy caches per locale.

```typescript
// Fetch content for a specific locale
export async function getLocalizedEvent(slug: string, locale: string) {
  return sanityClient.fetch(
    `*[_type == "event" && slug.current == $slug][0]{
      "title": title[$locale],
      "description": description[$locale],
      "body": body[$locale],
      // Non-localized fields
      "posterUrl": poster.asset->url,
      startDate,
      venue->{name, address},
    }`,
    { slug, locale }
  );
}

// Cache key includes locale
function useLocalizedEvent(slug: string) {
  const locale = useLocale(); // from your i18n setup

  return useQuery({
    queryKey: ['cms', 'event', slug, locale],
    queryFn: () => getLocalizedEvent(slug, locale),
    staleTime: 1000 * 60 * 15,
  });
}

// CDN caching: Vary on Accept-Language
// Your API route should include: Vary: Accept-Language
// This tells the CDN to cache separate versions per language
```

### 9.7 CMS for Both Web and Mobile -- Same Content, Different Rendering

The power of a headless CMS is that the same content serves both platforms:

```typescript
// Shared: content fetching (used by both web and mobile)
// packages/shared/src/cms/queries.ts
export async function getEventContent(
  slug: string,
  locale: string = 'en'
) {
  return sanityClient.fetch(eventQuery, { slug, locale });
}

// Web: render as Next.js page with ISR
// apps/web/app/events/[slug]/page.tsx
export default async function EventPage({ params }: Props) {
  const event = await getEventContent(params.slug);
  return (
    <article>
      <h1>{event.title}</h1>
      <PortableText value={event.body} components={webComponents} />
    </article>
  );
}
export const revalidate = 3600; // ISR: 1 hour

// Mobile: render as React Native screen with TanStack Query
// apps/mobile/src/screens/EventScreen.tsx
function EventScreen({ route }: Props) {
  const { data: event } = useQuery({
    queryKey: ['cms', 'event', route.params.slug],
    queryFn: () => getEventContent(route.params.slug),
    staleTime: 1000 * 60 * 15,
  });

  return (
    <ScrollView>
      <Text style={styles.title}>{event?.title}</Text>
      <PortableText value={event?.body} components={mobileComponents} />
    </ScrollView>
  );
}
```

### 9.8 When NOT to Use a CMS

A CMS is for **editorial content** -- content created and curated by humans (editors, marketers, content managers). Do not use a CMS for:

- **User-generated content:** Comments, reviews, user profiles. These live in your database.
- **Real-time data:** Chat messages, live scores, stock prices. These use WebSocket/SSE, not CMS.
- **Transactional data:** Orders, payments, tickets. These live in your database with ACID guarantees.
- **Personalized content:** Recommendations, activity feeds. These are computed per-user.
- **Configuration data:** Feature flags, A/B tests. Use Edge Config or LaunchDarkly.

**Use a CMS for:** Event descriptions, venue information, help articles, legal pages, marketing content, announcements, push notification templates, onboarding flows.

---

## 10. IMAGE AND MEDIA CDN

Images account for 50-80% of a mobile app's bandwidth. An image CDN transforms, optimizes, and serves images from the edge -- the single biggest performance win for image-heavy apps.

### 10.1 Image CDN Services

```
+------------------+-----------------+--------------------+------------------+
| Service          | Strengths       | Pricing Model      | Mobile Fit       |
+------------------+-----------------+--------------------+------------------+
| Cloudinary       | Most transforms | Per-transform +    | Excellent        |
|                  | AI-based crops  | bandwidth + storage| (has RN SDK)     |
+------------------+-----------------+--------------------+------------------+
| Imgix            | URL-based API   | Per-origin image   | Great            |
|                  | Real-time       | + bandwidth        | (URL transforms) |
+------------------+-----------------+--------------------+------------------+
| Cloudflare       | Cheapest at     | Per-image/month +  | Good             |
| Images           | scale, Workers  | unique transforms  |                  |
|                  | integration     |                    |                  |
+------------------+-----------------+--------------------+------------------+
| Vercel Image     | Built into      | Included in Vercel | Good (for web)   |
| Optimization     | Next.js <Image> | plan, per-optimize | N/A for RN       |
+------------------+-----------------+--------------------+------------------+
| Sanity Image     | Built into      | Included in Sanity | Great (for CMS   |
| Pipeline         | Sanity CMS      | plan               | content images)  |
+------------------+-----------------+--------------------+------------------+
```

### 10.2 On-The-Fly Transformations

Image CDNs transform images via URL parameters. No pre-processing needed.

```typescript
// Cloudinary URL-based transforms
function cloudinaryUrl(
  publicId: string,
  transforms: CloudinaryTransform
): string {
  const {
    width,
    height,
    quality = 'auto',
    format = 'auto',
    crop = 'fill',
  } = transforms;
  return [
    `https://res.cloudinary.com/${CLOUD_NAME}/image/upload`,
    `/c_${crop},w_${width},h_${height},q_${quality},f_${format}`,
    `/${publicId}`,
  ].join('');
}

// Sanity image URL builder
import imageUrlBuilder from '@sanity/image-url';

const builder = imageUrlBuilder(sanityClient);

function sanityImageUrl(
  source: SanityImageSource,
  width: number
): string {
  return builder
    .image(source)
    .width(width)
    .auto('format') // WebP on Chrome, AVIF where supported
    .quality(80)
    .url();
}

// Imgix URL-based transforms
function imgixUrl(
  path: string,
  params: Record<string, string | number>
): string {
  const searchParams = new URLSearchParams(
    Object.entries(params).map(([k, v]) => [k, String(v)])
  );
  return `https://${IMGIX_DOMAIN}/${path}?${searchParams.toString()}`;
}

// Usage:
// imgixUrl('events/poster.jpg', {
//   w: 400, h: 300, fit: 'crop', auto: 'format,compress',
// })
```

### 10.3 Responsive Images: Serving Different Sizes Per Device Density

Mobile devices have different screen densities (1x, 2x, 3x). Serving a 3x image to a 1x device wastes bandwidth. Serving a 1x image to a 3x device looks blurry.

```typescript
import { PixelRatio, Dimensions } from 'react-native';
import { Image } from 'expo-image';

const PIXEL_RATIO = PixelRatio.get(); // 1, 2, or 3
const SCREEN_WIDTH = Dimensions.get('window').width;

function OptimizedImage({
  source,
  width,
  height,
  style,
}: {
  source: SanityImageSource;
  width: number;
  height: number;
  style?: ViewStyle;
}) {
  // Request the right size for this device's density
  const pixelWidth = Math.round(width * PIXEL_RATIO);
  const pixelHeight = Math.round(height * PIXEL_RATIO);

  const uri = builder
    .image(source)
    .width(pixelWidth)
    .height(pixelHeight)
    .auto('format')
    .quality(PIXEL_RATIO >= 3 ? 65 : 80) // Lower quality at 3x (user won't notice)
    .url();

  return (
    <Image
      source={{ uri }}
      style={[{ width, height }, style]}
      contentFit="cover"
      cachePolicy="memory-disk"
      transition={200}
    />
  );
}

// Usage
// <OptimizedImage source={event.poster} width={SCREEN_WIDTH} height={200} />
// On a 3x iPhone: requests 1170x600 image
// On a 2x Android: requests 720x400 image
// On a 1x device: requests 360x200 image
// All from the same source image, transformed on-the-fly by the CDN
```

### 10.4 Video CDN Patterns

For apps with video content (event trailers, how-to videos):

```typescript
// HLS adaptive streaming -- the video serves different quality levels
// based on the user's network speed. The CDN handles this.

// Upload to Cloudflare Stream, Mux, or Cloudinary
// They automatically generate HLS manifests with multiple quality levels

function VideoPlayer({ videoId }: { videoId: string }) {
  // Mux example: HLS streaming URL
  const hlsUrl = `https://stream.mux.com/${videoId}.m3u8`;

  // Thumbnail/poster image (also from the CDN)
  const thumbnailUrl = `https://image.mux.com/${videoId}/thumbnail.webp?time=5&width=800`;

  return (
    <Video
      source={{ uri: hlsUrl }}
      posterSource={{ uri: thumbnailUrl }}
      style={styles.video}
      useNativeControls
      resizeMode="contain"
      // expo-av handles HLS adaptive bitrate automatically
    />
  );
}
```

### 10.5 Caching Strategy: Immutable URLs with Content Hashing

The best caching strategy for images: make the URL change when the content changes, then cache forever.

```typescript
// Pattern: content-addressed image URLs
// Instead of: /images/event-poster.jpg (mutable -- how do you invalidate?)
// Use: /images/abc123def456/event-poster.webp (content hash in URL)

// When the event manager uploads a new poster:
// 1. Image CDN generates a new content hash
// 2. New URL: /images/xyz789ghi012/event-poster.webp
// 3. Old URL is still cached (but nothing references it anymore)
// 4. Cache-Control: immutable, max-age=31536000

// Sanity does this automatically -- each image asset has a unique ID:
// https://cdn.sanity.io/images/project/dataset/abc123-800x600.webp

// The URL only changes when the image content changes.
// You NEVER need to invalidate image cache entries.
```

---

## 11. CACHE INVALIDATION -- THE HARD PART

Phil Karlton's famous quote: "There are only two hard things in Computer Science: cache invalidation and naming things." He was right. Let's tackle the hard one.

### 11.1 Time-Based Invalidation (TTL)

The simplest approach. Data expires after a fixed duration.

```typescript
// TTL guidelines per data type
const TTL_CONFIG = {
  // Rapidly changing data -- short TTL
  ticketAvailability: 30,        // 30 seconds
  trendingEvents: 60,            // 1 minute
  searchResults: 120,            // 2 minutes

  // Moderately changing data -- medium TTL
  eventList: 300,                // 5 minutes
  eventDetails: 600,             // 10 minutes
  venueInfo: 1800,               // 30 minutes

  // Rarely changing data -- long TTL
  userProfile: 3600,             // 1 hour
  cmsContent: 3600,              // 1 hour
  helpArticles: 86400,           // 24 hours

  // Static data -- very long TTL
  categoryList: 86400,           // 24 hours
  countryList: 604800,           // 7 days
  staticAssets: 31536000,        // 1 year (with content hash in URL)
} as const;
```

**Advantage:** Dead simple. No coordination needed. Predictable behavior.

**Disadvantage:** Data can be stale for up to the full TTL duration. If you set TTL to 5 minutes, a user might see data that's 4 minutes and 59 seconds old.

### 11.2 Event-Based Invalidation (Webhooks)

Data is invalidated when the source of truth changes, triggered by a webhook or database event.

```typescript
// The complete webhook pipeline for CMS content

// Step 1: CMS sends webhook when content changes
// Sanity webhook payload:
// {
//   "_type": "event",
//   "_id": "event-42",
//   "slug": { "current": "summer-festival-2026" },
//   "operation": "update"
// }

// Step 2: Next.js API route receives webhook
// app/api/webhooks/sanity/route.ts
import { revalidateTag, revalidatePath } from 'next/cache';
import { isValidSignature, SIGNATURE_HEADER_NAME } from '@sanity/webhook';

export async function POST(request: Request) {
  const body = await request.text();
  const signature = request.headers.get(SIGNATURE_HEADER_NAME) ?? '';

  if (!isValidSignature(body, signature, process.env.SANITY_WEBHOOK_SECRET!)) {
    return new Response('Invalid signature', { status: 401 });
  }

  const payload = JSON.parse(body);

  // Step 3: Invalidate relevant caches
  switch (payload._type) {
    case 'event':
      // Invalidate Next.js cache (ISR pages + fetch cache)
      revalidateTag(`event-${payload._id}`);
      revalidateTag('events-list');
      revalidatePath('/events');
      revalidatePath(`/events/${payload.slug?.current}`);

      // Invalidate Redis cache
      await redis.del(`event:${payload._id}`);
      await redis.del('events:list');
      await redis.del('events:trending');

      // Publish invalidation event for any subscribers
      await redis.publish('cache-invalidation', JSON.stringify({
        type: 'event',
        id: payload._id,
        operation: payload.operation,
      }));
      break;

    case 'venue':
      revalidateTag(`venue-${payload._id}`);
      revalidateTag('events-list'); // Events show venue info
      await redis.del(`venue:${payload._id}`);
      break;

    case 'artist':
      revalidateTag(`artist-${payload._id}`);
      await redis.del(`artist:${payload._id}`);
      break;
  }

  return Response.json({
    revalidated: true,
    type: payload._type,
    id: payload._id,
    timestamp: Date.now(),
  });
}
```

**Advantage:** Data is fresh within seconds of a change. No unnecessary stale data.

**Disadvantage:** Requires infrastructure (webhook endpoints, signature validation, error handling). If the webhook fails, the cache stays stale until TTL expires. You need a fallback.

### 11.3 Tag-Based Invalidation

The most flexible approach. Tag cached entries with semantic labels, then invalidate by tag.

```typescript
// Tag-based cache with Redis
class TaggedCache {
  constructor(private redis: Redis) {}

  async set(
    key: string,
    data: unknown,
    tags: string[],
    ttl: number
  ): Promise<void> {
    const pipeline = this.redis.pipeline();

    // Store the data
    pipeline.set(key, JSON.stringify(data), { ex: ttl });

    // Associate tags with this key
    for (const tag of tags) {
      pipeline.sadd(`tag:${tag}`, key);
      pipeline.expire(`tag:${tag}`, ttl + 60); // Tag set expires slightly after data
    }

    // Store reverse mapping (key -> tags) for cleanup
    pipeline.sadd(`keys:${key}:tags`, ...tags);
    pipeline.expire(`keys:${key}:tags`, ttl + 60);

    await pipeline.exec();
  }

  async get<T>(key: string): Promise<T | null> {
    const data = await this.redis.get<string>(key);
    return data ? JSON.parse(data) : null;
  }

  async invalidateByTag(tag: string): Promise<number> {
    // Get all keys associated with this tag
    const keys = await this.redis.smembers(`tag:${tag}`);
    if (keys.length === 0) return 0;

    // Delete all those keys
    const pipeline = this.redis.pipeline();
    for (const key of keys) {
      pipeline.del(key);
      pipeline.del(`keys:${key}:tags`);
    }
    pipeline.del(`tag:${tag}`);
    await pipeline.exec();

    return keys.length;
  }
}

// Usage
const cache = new TaggedCache(redis);

// Cache an event with multiple tags
await cache.set(
  'event:42',
  eventData,
  ['event-42', 'venue-7', 'events-all', 'artist-15', 'category-music'],
  300
);

// When venue 7 updates, invalidate all cached data related to it
const invalidated = await cache.invalidateByTag('venue-7');
// This removes event:42 (and any other events at venue 7) from cache
```

### 11.4 Version-Based Invalidation

Append a version identifier to cache keys. When data changes, bump the version.

```typescript
// Version-based cache invalidation
class VersionedCache {
  constructor(private redis: Redis) {}

  private async getVersion(namespace: string): Promise<number> {
    const version = await this.redis.get<number>(`version:${namespace}`);
    return version ?? 1;
  }

  async set(
    namespace: string,
    key: string,
    data: unknown,
    ttl: number
  ): Promise<void> {
    const version = await this.getVersion(namespace);
    await this.redis.set(
      `${namespace}:v${version}:${key}`,
      JSON.stringify(data),
      { ex: ttl }
    );
  }

  async get<T>(namespace: string, key: string): Promise<T | null> {
    const version = await this.getVersion(namespace);
    const data = await this.redis.get<string>(
      `${namespace}:v${version}:${key}`
    );
    return data ? JSON.parse(data) : null;
  }

  async bumpVersion(namespace: string): Promise<number> {
    // Increment the version -- all old cache entries are now "invisible"
    // (they still exist in Redis but nothing requests them; TTL cleans up)
    return this.redis.incr(`version:${namespace}`);
  }
}

// Usage
const cache = new VersionedCache(redis);

// Normal reads/writes
await cache.set('events', '42', eventData, 300);
const event = await cache.get('events', '42');

// When the events schema changes (e.g., after a deployment):
await cache.bumpVersion('events');
// All existing event cache entries are now inaccessible
// They'll be lazily repopulated with the new schema
```

### 11.5 The Write-Invalidate vs Write-Update Trade-Off

When data changes, you have two choices for the cache:

**Write-invalidate:** Delete the cache entry. Next read will miss and repopulate.
```typescript
// Write-invalidate
async function updateEvent(id: string, data: Partial<Event>) {
  await db.update(events).set(data).where(eq(events.id, id));
  await redis.del(`event:${id}`); // Invalidate -- next read will cache the new version
}
```

**Write-update:** Update the cache entry directly with the new data.
```typescript
// Write-update
async function updateEvent(id: string, data: Partial<Event>) {
  const updated = await db
    .update(events)
    .set(data)
    .where(eq(events.id, id))
    .returning();
  await redis.set(`event:${id}`, JSON.stringify(updated[0]), { ex: 300 });
}
```

```
+-----------------+---------------------------+---------------------------+
|                 | Write-Invalidate          | Write-Update              |
+-----------------+---------------------------+---------------------------+
| Simplicity      | Simple -- just delete     | Must construct full entry |
| Consistency     | Eventually (on next read) | Immediately consistent    |
| Next read speed | Slow (cache miss)         | Fast (cache hit)          |
| Write speed     | Fast (just a DELETE)      | Slower (SET with data)    |
| Race conditions | Safe (worst case = miss)  | Risk of stale overwrites  |
| Complexity      | Low                       | Medium (need full object) |
+-----------------+---------------------------+---------------------------+
```

**My recommendation:** Use write-invalidate as your default. It's simpler and safer. Use write-update only when you can guarantee the update is atomic and you need the cache to be immediately consistent (like after a user profile update where you want the next page load to show the new data without a cache miss).

### 11.6 Dealing With Eventual Consistency

In a layered caching system, there is an unavoidable period where different layers have different versions of the data. Accept this and design for it.

```
Timeline of a content update:

  T+0s    Editor publishes in Sanity CMS
  T+0.5s  Webhook fires to your API
  T+1s    revalidateTag invalidates Next.js cache
  T+2s    ISR regenerates the page in the background
  T+3s    CDN serves new version to web users

  T+0s to T+300s  Mobile app's TanStack Query still has old data (staleTime=5min)
  T+300s          TanStack Query marks data as stale, background refetch begins
  T+301s          Fresh data arrives from API (which is now serving new version)
  T+301s          MMKV persistence updates with new data

  Gap: Up to 5 minutes where mobile users see old data.
  Is this acceptable? For event listings, absolutely.
  For ticket availability? No -- use staleTime=30s for that.
```

**Design principle:** Accept eventual consistency as a feature, not a bug. Communicate it clearly in your architecture docs so the team knows what staleness to expect for each data type.

---

## 12. PUTTING IT ALL TOGETHER -- A REAL ARCHITECTURE

Let's trace a single content item through all seven layers. The scenario: an event manager updates the "Summer Festival 2026" event listing in the TicketFlow app.

### 12.1 The Complete Flow

```
+---------------------------------------------------------------------------+
|                    COMPLETE CACHE FLOW: EVENT UPDATE                       |
|                                                                           |
|  STEP 1: Content Creation (T+0s)                                         |
|  ----------------------------------------                                |
|  Event manager opens Sanity Studio, updates the event description,        |
|  changes the lineup, and clicks "Publish"                                 |
|                                                                           |
|  STEP 2: Webhook Fires (T+0.3s)                                         |
|  ----------------------------------------                                |
|  Sanity sends a webhook to:                                               |
|  POST https://ticketflow.app/api/webhooks/sanity                         |
|  Body: { _type: "event", _id: "evt_summer2026", operation: "update" }    |
|                                                                           |
|  STEP 3: Server-Side Invalidation (T+0.5s)                               |
|  ----------------------------------------                                |
|  The webhook handler:                                                     |
|  a) revalidateTag('event-evt_summer2026')  -> Next.js ISR cache          |
|  b) revalidateTag('events-list')           -> Event listing ISR cache    |
|  c) redis.del('event:evt_summer2026')      -> Redis application cache    |
|  d) redis.del('events:list')               -> Redis list cache           |
|                                                                           |
|  STEP 4: ISR Regeneration (T+1-3s)                                       |
|  ----------------------------------------                                |
|  Next.js regenerates /events/summer-festival-2026 in the background       |
|  The new HTML is generated, including fresh CMS content                   |
|  CDN caches the new version with s-maxage=3600                           |
|                                                                           |
|  STEP 5: Web User Visits (T+any time after step 4)                       |
|  ----------------------------------------                                |
|  Browser requests /events/summer-festival-2026                            |
|  CDN serves the freshly regenerated page (~10ms)                         |
|                                                                           |
|  STEP 6: Mobile App -- Background Refetch (T+staleTime after last fetch) |
|  ----------------------------------------                                |
|  User opens the TicketFlow app or navigates to the event screen           |
|  TanStack Query checks: is ['events', 'evt_summer2026'] stale?           |
|  If staleTime (5 min) has passed since last fetch:                        |
|    a) Return cached data instantly from L7 (in-memory)                   |
|    b) Trigger background refetch                                          |
|    c) Fetch hits CDN -> fresh ISR page data -> returns new content       |
|    d) UI updates silently with new data                                   |
|                                                                           |
|  If app was killed and restarted (cold start):                            |
|    a) MMKV restores cached data from L6 (device storage)                 |
|    b) User sees last-known data instantly (~100ms)                        |
|    c) Background refetch pulls fresh data                                 |
|    d) MMKV updates with fresh data for next cold start                   |
|                                                                           |
|  STEP 7: Image Cache (unchanged)                                         |
|  ----------------------------------------                                |
|  The event poster URL didn't change (unless manager uploaded a new one)   |
|  expo-image's disk cache serves the poster instantly                      |
|  If a new poster was uploaded, the URL is different (content-addressed),  |
|  so expo-image fetches the new one and caches it                          |
|                                                                           |
+---------------------------------------------------------------------------+
```

### 12.2 Timing Breakdown

```
+-----------------------------+-----------+----------------------------------+
| Step                        | Duration  | Layer(s) Involved                |
+-----------------------------+-----------+----------------------------------+
| Editor publishes content    | 0s        | CMS (external)                   |
| Webhook delivery            | ~300ms    | CMS -> L4 (CDN edge -> origin)   |
| Redis invalidation          | ~5ms      | L3 (Redis)                       |
| ISR tag invalidation        | ~1ms      | L4 (CDN internal)                |
| ISR background regeneration | ~1-3s     | L1->L3->L4 (DB->Redis->CDN)     |
| Web user gets fresh page    | ~10ms     | L4->L5 (CDN->browser)            |
| Mobile: stale data served   | ~0ms      | L7 (in-memory)                   |
| Mobile: background refetch  | ~50-200ms | L4->L5->L7 (CDN->HTTP->memory)  |
| Mobile: MMKV persistence    | ~2ms      | L6 (device storage)              |
| Mobile cold start hydration | ~1ms      | L6->L7 (MMKV->memory)            |
+-----------------------------+-----------+----------------------------------+
| TOTAL: content change to    | ~0.5-3s   | All layers on the write path     |
|   web user seeing fresh     |           |                                  |
| TOTAL: content change to    | ~0-5min   | Depends on staleTime config      |
|   mobile user seeing fresh  |           |                                  |
| TOTAL: cold start to data   | ~100ms    | L6->L7 only (no network!)        |
|   visible on mobile         |           |                                  |
+-----------------------------+-----------+----------------------------------+
```

### 12.3 The Complete Architecture Diagram

```
+---------------------------------------------------------------------------+
|                                                                           |
|                          +---------------+                                |
|                          |  Sanity CMS   | <- Event managers, editors     |
|                          |  (content)    |                                |
|                          +-------+-------+                                |
|                     webhook ---- |                                        |
|                                  v                                        |
|  +---------------+     +--------------------+     +------------------+   |
|  |   Neon        |     |   Next.js on       |     |  Upstash Redis   |   |
|  |   Postgres    |<----|   Vercel           |---->|  (cache + rate   |   |
|  |   (L1)        |     |   (origin server)  |     |   limit + pubsub)|   |
|  |               |     |                    |     |  (L3)            |   |
|  +---------------+     +--------+-----------+     +------------------+   |
|                                  |                                        |
|                         ISR pages + API                                   |
|                         responses                                         |
|                                  |                                        |
|                                  v                                        |
|                        +------------------+                               |
|                        |  Vercel Edge     |                               |
|                        |  Network (CDN)   |     +------------------+     |
|                        |  (L4)            |     |  Cloudinary      |     |
|                        +--------+---------+     |  (image CDN)     |     |
|                                 |               +--------+---------+     |
|                    +------------+------------+           |               |
|                    |                         |           |               |
|                    v                         v           |               |
|           +--------------+          +--------------+     |               |
|           |  Web Browser |          |  Mobile App  |     |               |
|           |              |          |              |     |               |
|           |  HTTP Cache  |          |  HTTP Cache  |     |               |
|           |  (L5)        |          |  (L5)        |<----+               |
|           |              |          |              |                      |
|           |  Next.js     |          |  TanStack    |                      |
|           |  (SWR/RSC)   |          |  Query (L7)  |                      |
|           |              |          |              |                      |
|           +--------------+          |  MMKV (L6)   |                      |
|                                     |              |                      |
|                                     |  expo-image  |                      |
|                                     |  disk cache  |                      |
|                                     +--------------+                      |
|                                                                           |
+---------------------------------------------------------------------------+
```

### 12.4 The Decision Matrix: What to Cache Where

This is the reference table. Bookmark it, print it, stick it on your wall.

```
+------------------------+-------+-------+-------+-------+-------+-------+-------+
| Data Type              |  L1   |  L2   |  L3   |  L4   |  L5   |  L6   |  L7   |
|                        |  DB   | QCache| Redis |  CDN  | HTTP  | MMKV  | Query |
+------------------------+-------+-------+-------+-------+-------+-------+-------+
| Event listings         |  SoT  | Auto  | 5min  | 60s+  | 60s   | Yes   | 5min  |
| Event details          |  SoT  | Auto  | 10min | 60s+  | 60s   | Yes   | 5min  |
| Ticket availability    |  SoT  | Auto  | 30s   | No*   | No    | No    | 30s   |
| User profile (own)     |  SoT  | Auto  | 1hr   | No**  | 60s   | Yes   | 1hr   |
| User profile (others)  |  SoT  | Auto  | 30min | 5min  | 5min  | Yes   | 30min |
| CMS content            |  SoT  | Auto  | 1hr   | ISR   | 5min  | Yes   | 15min |
| Search results         |  SoT  | Auto  | 2min  | No**  | No    | No    | 2min  |
| Financial data         |  SoT  | Auto  | No    | No    | No    | No    | 0s    |
| Feature flags          |  -    | -     | 5min  | Edge  | No    | Yes   | 5min  |
| Static assets          |  -    | -     | -     | 1yr   | 1yr   | N/A   | N/A   |
| Images                 |  -    | -     | -     | 1yr   | 1yr   | Disk  | N/A   |
| Help articles          |  SoT  | Auto  | 24hr  | ISR   | 1hr   | Yes   | 1hr   |
| Push notification tmpl |  SoT  | Auto  | 1hr   | -     | -     | No    | -     |
+------------------------+-------+-------+-------+-------+-------+-------+-------+
| Legend:                                                                         |
| SoT = Source of Truth    Auto = DB handles it    ISR = on-demand revalidation  |
| No* = too dynamic         No** = user-specific    - = not applicable           |
| Edge = Vercel Edge Config  Disk = expo-image disk cache                        |
+------------------------+-------+-------+-------+-------+-------+-------+-------+
```

### 12.5 The Implementation Checklist

Use this checklist when setting up the caching architecture for a new feature or a new project:

```
DATABASE LAYER
  [ ] Postgres shared_buffers set to 25% of available RAM
  [ ] pg_stat_statements enabled for query profiling
  [ ] Connection pooling configured (Neon built-in or PgBouncer)
  [ ] Read replica set up for read-heavy workloads (if needed)
  [ ] Drizzle queries use JOINs, not N+1 patterns
  [ ] Drizzle queries select only needed columns

APPLICATION CACHE (REDIS)
  [ ] Upstash Redis provisioned and connected
  [ ] Cache-aside pattern implemented for hot data paths
  [ ] TTLs configured per data type (see decision matrix)
  [ ] Jittered TTL to prevent cache stampede
  [ ] Mutex lock for thundering herd protection on popular keys
  [ ] Rate limiting configured (sliding window)
  [ ] Session storage in Redis (if using sessions)
  [ ] Graceful fallback: Redis down = fall through to DB

CDN EDGE
  [ ] Cache-Control headers set correctly per route
  [ ] Static assets: public, max-age=31536000, immutable
  [ ] API responses: s-maxage + stale-while-revalidate
  [ ] User-specific data: private or no-store
  [ ] ISR configured for CMS-driven pages
  [ ] Webhook handler for on-demand revalidation
  [ ] Image CDN configured (Cloudinary/Imgix/Sanity)
  [ ] Edge Config for feature flags and A/B tests

CLIENT SIDE
  [ ] TanStack Query configured with per-query staleTime
  [ ] Query key factory with hierarchical keys
  [ ] MMKV persistence for cold start hydration
  [ ] shouldDehydrateQuery excludes sensitive and large data
  [ ] Optimistic updates for user-initiated mutations
  [ ] expo-image cachePolicy set to memory-disk
  [ ] ETag/If-None-Match for bandwidth savings

CACHE INVALIDATION
  [ ] Webhook pipeline: CMS -> API -> revalidateTag -> Redis del
  [ ] Webhook signature validation
  [ ] Tag-based invalidation for grouped content
  [ ] Fallback: TTL expiry if webhook fails
  [ ] Monitoring: log cache hit/miss ratios
  [ ] Alerting: high cache miss rate = something is wrong
```

---

## 13. MONITORING AND DEBUGGING THE CACHE STACK

A caching architecture is only as good as your ability to observe it. When something goes wrong (and it will), you need to know which layer is misbehaving.

### 13.1 Cache Hit Rate Monitoring

```typescript
// Wrap your cache-aside function with monitoring
async function cacheAsideWithMetrics<T>(
  key: string,
  fetcher: () => Promise<T>,
  options: { ttl: number; layer: string }
): Promise<T> {
  const start = Date.now();

  const cached = await redis.get<T>(key);
  const duration = Date.now() - start;

  if (cached !== null) {
    // Cache HIT
    metrics.increment('cache.hit', {
      layer: options.layer,
      key_prefix: key.split(':')[0],
    });
    metrics.histogram('cache.hit.latency', duration, {
      layer: options.layer,
    });
    return cached;
  }

  // Cache MISS
  metrics.increment('cache.miss', {
    layer: options.layer,
    key_prefix: key.split(':')[0],
  });

  const fetchStart = Date.now();
  const data = await fetcher();
  const fetchDuration = Date.now() - fetchStart;
  metrics.histogram('cache.miss.fetch_latency', fetchDuration, {
    layer: options.layer,
  });

  await redis.set(key, JSON.stringify(data), { ex: options.ttl });
  return data;
}
```

### 13.2 Debugging Cache Issues

The most common cache bugs and how to diagnose them:

**Problem: Users see stale data after an update.**
```
Diagnosis checklist:
1. Did the webhook fire? Check CMS webhook logs.
2. Did the webhook handler succeed? Check your API route logs.
3. Was revalidateTag called with the correct tag? Check the tag string.
4. Is the CDN serving the new version? curl -I the URL and check Age header.
5. Is Redis cleared? Check redis.get for the key.
6. Is TanStack Query refetching? The client's staleTime may not have passed yet.
7. Is MMKV serving the old version? Only on cold start -- check persistence config.
```

**Problem: Cache miss rate is too high (>20% for warmed data).**
```
Diagnosis checklist:
1. Are TTLs too short? Check your TTL config.
2. Is Redis evicting keys? Check maxmemory-policy and memory usage.
3. Are cache keys inconsistent? e.g., 'event:42' vs 'event:042' vs 'events:42'
4. Are you caching per-user data with low reuse? Check key cardinality.
5. Did a deployment clear the cache? Check if you have cache warming post-deploy.
```

**Problem: Redis memory usage is growing unboundedly.**
```
Diagnosis checklist:
1. Are you setting TTL on every key? Check for keys without expiry.
2. Are you caching user-specific data? Check key count vs unique user count.
3. Are you caching large payloads? Use redis-cli --bigkeys to find large keys.
4. Is the eviction policy correct? Use allkeys-lru for caches.
```

### 13.3 The Cache Debug Header

Add a custom header to API responses that shows which cache layer served the response. Invaluable for debugging.

```typescript
// Middleware that adds cache debug headers
function addCacheDebugHeaders(
  response: Response,
  cacheInfo: CacheInfo
): Response {
  const headers = new Headers(response.headers);
  headers.set('X-Cache-Layer', cacheInfo.layer);       // 'redis' | 'cdn' | 'origin'
  headers.set('X-Cache-Status', cacheInfo.status);     // 'hit' | 'miss' | 'stale'
  headers.set('X-Cache-TTL', cacheInfo.remainingTtl.toString());
  headers.set('X-Cache-Key', cacheInfo.key);
  headers.set('X-Cache-Age', cacheInfo.age.toString()); // seconds since cached
  return new Response(response.body, { ...response, headers });
}

// Usage in your API
export async function GET(request: Request) {
  const cacheInfo: CacheInfo = {
    layer: 'origin',
    status: 'miss',
    key: '',
    remainingTtl: 0,
    age: 0,
  };

  // Check Redis
  const cached = await redis.get(cacheKey);
  if (cached) {
    cacheInfo.layer = 'redis';
    cacheInfo.status = 'hit';
    const ttl = await redis.ttl(cacheKey);
    cacheInfo.remainingTtl = ttl;
    return addCacheDebugHeaders(Response.json(cached), cacheInfo);
  }

  // Fetch from DB
  const data = await fetchFromDB();
  await redis.set(cacheKey, data, { ex: 300 });

  return addCacheDebugHeaders(Response.json(data), cacheInfo);
}
```

---

## 14. ANTI-PATTERNS AND COMMON MISTAKES

After years of building and debugging caching systems, these are the mistakes I see most often.

### 14.1 Caching User-Specific Data on the CDN

```typescript
// WRONG: This caches the response for ALL users on the CDN
export async function GET(request: Request) {
  const session = await getSession(request);
  const userData = await getUserData(session.userId);
  return Response.json(userData, {
    headers: {
      // BUG! User A sees User B's data!
      'Cache-Control': 'public, s-maxage=3600',
    },
  });
}

// RIGHT: Use 'private' or 'no-store' for user-specific data
return Response.json(userData, {
  headers: { 'Cache-Control': 'private, max-age=60' },
});
```

### 14.2 Not Setting TTL on Redis Keys

```typescript
// WRONG: No TTL -- this key lives forever, even if the data is deleted from the DB
await redis.set('event:42', JSON.stringify(event));

// RIGHT: Always set a TTL
await redis.set('event:42', JSON.stringify(event), { ex: 300 });

// EVEN BETTER: Use a default TTL wrapper
async function redisSetWithTTL(
  key: string,
  data: unknown,
  ttl: number = 300
): Promise<void> {
  await redis.set(key, JSON.stringify(data), { ex: ttl });
}
```

### 14.3 Cache Key Inconsistency

```typescript
// WRONG: Inconsistent key formats across the codebase
await redis.set('event:42', data);        // Some code uses this
await redis.set('events:42', data);       // Other code uses this
await redis.set('event_42', data);        // A third pattern
await redis.del('event:42');              // Deletes one, but the other two stay stale

// RIGHT: Centralize key generation
const cacheKeys = {
  event: (id: string) => `event:${id}`,
  eventList: (filters?: string) =>
    `events:list${filters ? `:${filters}` : ''}`,
  venue: (id: string) => `venue:${id}`,
  user: (id: string) => `user:${id}`,
  session: (id: string) => `session:${id}`,
} as const;
```

### 14.4 Caching Errors

```typescript
// WRONG: Caching null/error responses
const data = await fetcher(); // Returns null due to a transient error
await redis.set(key, JSON.stringify(data), { ex: 300 });
// Now null is cached for 5 minutes!

// RIGHT: Only cache successful, non-null responses
const data = await fetcher();
if (data !== null && data !== undefined) {
  await redis.set(key, JSON.stringify(data), { ex: 300 });
}

// OR: Cache errors with a very short TTL (to prevent hammering the DB)
if (data === null) {
  await redis.set(`${key}:null`, '1', { ex: 10 }); // "Negative cache" for 10 seconds
}
```

### 14.5 Ignoring Cache Warming After Deployments

```
Problem: After deploying new server instances, all caches are cold.
Every request hits the database. If you get a traffic spike, the database dies.

Solution: Add cache warming to your CI/CD pipeline.
After the deployment step, call your cache warming endpoint.
The warming endpoint pre-populates Redis with the top N most-accessed items.
```

---

## 15. THE CACHING ARCHITECT'S CHEAT SHEET

If you remember nothing else from this chapter, remember this:

```
+---------------------------------------------------------------------------+
|                                                                           |
|  1. THE DEFAULT IS CACHE-ASIDE WITH TTL                                  |
|     It covers 90% of cases. Start here.                                  |
|                                                                           |
|  2. SET Cache-Control HEADERS ON EVERY RESPONSE                          |
|     This is the highest-leverage optimization in your stack.             |
|     Static assets: immutable, 1 year.                                    |
|     API responses: s-maxage + stale-while-revalidate.                    |
|     User data: private.                                                   |
|     Financial data: no-store.                                             |
|                                                                           |
|  3. PERSIST TANSTACK QUERY TO MMKV                                       |
|     This single pattern makes your app feel instant on cold start.       |
|                                                                           |
|  4. USE WEBHOOKS + revalidateTag FOR CMS CONTENT                         |
|     CMS publishes -> webhook -> invalidate -> ISR regenerates.           |
|     The mobile app catches up via TanStack Query staleTime.              |
|                                                                           |
|  5. PROTECT AGAINST THUNDERING HERDS                                     |
|     Use mutex locks on popular cache keys.                               |
|     Use jittered TTLs to prevent cache stampedes.                        |
|                                                                           |
|  6. NEVER CACHE USER-SPECIFIC DATA ON THE CDN                            |
|     public = CDN can cache = everyone sees the same response.            |
|     private = only the user's browser/device can cache.                  |
|                                                                           |
|  7. MONITOR HIT RATES                                                     |
|     Redis: >95% hit rate for warmed data.                                |
|     CDN: >90% hit rate for public content.                               |
|     Postgres: >99% buffer hit rate.                                      |
|     If any of these are lower, investigate.                              |
|                                                                           |
|  8. ACCEPT EVENTUAL CONSISTENCY                                           |
|     Design your TTLs per data type.                                      |
|     Document the maximum staleness for each type.                        |
|     Communicate this to your team and stakeholders.                      |
|                                                                           |
+---------------------------------------------------------------------------+
```

---

## Summary

Full-stack caching architecture is not about any single tool. It's about understanding how data flows through seven distinct layers -- from the database (L1) through the query cache (L2), application cache (L3), CDN edge (L4), HTTP cache (L5), device storage (L6), and in-memory cache (L7) -- and making deliberate decisions about what lives at each layer, how long it lives there, and how it gets invalidated when the source of truth changes.

The five caching patterns (cache-aside, write-through, write-behind, read-through, and cache warming) are your building blocks. Cache-aside is your default for 90% of cases. Write-through for data that must be immediately consistent after writes. Write-behind for high-throughput writes. Cache warming for predictable traffic spikes.

The CMS content pipeline (editor publishes, webhook fires, ISR regenerates, CDN caches, mobile refetches, device persists) is the canonical example of full-stack caching. Each hop has a TTL, each TTL represents a consistency trade-off, and the mobile app's cold-start experience depends on the MMKV persistence layer at the bottom of the client-side stack.

Cache invalidation remains the hardest problem. Use time-based (TTL) as your default, event-based (webhooks) for content you control, tag-based for grouped invalidation, and version-based when you need clean cache rotation.

The single highest-leverage optimization in this entire chapter is correct `Cache-Control` headers. One header change can eliminate 95% of origin traffic. The second highest-leverage optimization is MMKV persistence of TanStack Query -- it makes your app feel instant on cold start, which is the most important moment in your user's experience.

Build the pyramid. Set the headers. Persist the cache. Accept eventual consistency. Monitor hit rates. Sleep well at night.

---

**Next:** [Chapter 23: Security Architecture] -- authentication, authorization, and the security patterns that protect the data flowing through all these cache layers.
