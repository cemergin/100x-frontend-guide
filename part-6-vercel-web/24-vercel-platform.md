<!--
  CHAPTER: 27
  TITLE: Vercel Platform Mastery
  PART: VI — Vercel & the Web
  PREREQS: None
  KEY_TOPICS: Fluid Compute, Vercel Functions, Blob, Edge Config, AI Gateway, Queues, CLI, vercel.ts, Rolling Releases, Analytics, Speed Insights, BotID, DDoS, Sandbox
  DIFFICULTY: Intermediate
  UPDATED: 2026-04-07
-->

# Chapter 24: Vercel Platform Mastery

---

> "Vercel is where frontend developers go to deploy, and where senior engineers go to build production systems."

---

<details>
<summary><strong>TL;DR</strong></summary>

- Vercel in 2026 is a full compute platform, not just a Next.js host; Fluid Compute reuses function instances across concurrent requests, Node.js 24 is the default, and the function timeout is 300 seconds
- Active CPU pricing means you pay for computation, not wall-clock time; idle time waiting on DB queries or external APIs is free
- Vercel Storage includes Blob (file storage), Edge Config (sub-millisecond key-value reads), and Marketplace integrations for Postgres (Neon) and Redis (Upstash)
- AI Gateway provides unified multi-provider routing with automatic failover, cost tracking, and rate limiting across OpenAI, Anthropic, Google, and others
- Preview deployments on every PR, rolling releases for production, and Speed Insights + Analytics give you a deployment pipeline that most teams never outgrow

</details>

## The Pitch You Need to Forget

If your mental model of Vercel is still "the company that hosts Next.js apps," you are working with outdated information. That was 2020. Maybe 2021. It is now 2026, and Vercel is a full compute platform that happens to have best-in-class frontend deployment baked in.

This chapter is going to walk through the entire Vercel platform as it exists today — not the marketing version, not the "getting started" tutorial version, but the version you need to understand when you are responsible for production systems serving real users and real money.

We will cover Fluid Compute, Functions, Storage, AI Gateway, Queues, the deployment pipeline, the CLI, configuration, analytics, security, and — because I believe in honest trade-off analysis — the self-hosted alternative on AWS for teams that want zero vendor lock-in.

Let us begin.

---

## 1. Vercel Is Not Just a Hosting Platform

### The Old Mental Model

In the early days, the pitch was simple:

1. Push to GitHub.
2. Vercel builds your Next.js app.
3. Static pages go to the CDN, API routes become Lambda functions.
4. Done.

This was magical in 2019. It was also limited. You could not run a long-lived process. You could not reuse connections across requests. You could not run anything that was not JavaScript or TypeScript. The serverless functions were cold, isolated, and ephemeral in the most frustrating sense of those words.

### The New Reality

Today, Vercel is a compute platform. Here is what that means:

- **Backend frameworks run natively.** Express, Fastify, Hono, Flask, Django — they all deploy to Vercel without a Next.js wrapper. You do not need to contort your backend into API routes.
- **Fluid Compute reuses function instances across concurrent requests.** This is not traditional serverless. A single function instance can handle multiple requests simultaneously, just like a real server.
- **Node.js 24 LTS is the default runtime.** You also have Bun, Rust, and Python as first-class options.
- **The default function timeout is 300 seconds.** Five minutes. Not the 10-second ceiling you remember.
- **Active CPU pricing** means you pay for computation, not wall-clock time. If your function is waiting on a database query, you are not being billed for that idle time.

This is a fundamentally different platform than what existed three years ago.

### Who Actually Uses It This Way?

Real companies running real backends on Vercel:

- **Perplexity** runs their AI-powered search backend on Vercel Functions with streaming responses.
- **Loom** uses Vercel for their web application layer, with Fluid Compute handling API traffic.
- **HashiCorp** migrated their documentation and marketing infrastructure to Vercel, including server-side logic.

The point is not that every company should move their entire backend to Vercel. The point is that Vercel is no longer a "frontend only" platform, and if you are building a modern web application with a moderate backend, you no longer need a separate infrastructure provider for the server-side pieces.

### When Vercel Is the Right Call

| Scenario | Vercel? | Why |
|----------|---------|-----|
| Web app with API routes | Yes | This is the sweet spot |
| AI-powered product with streaming | Yes | Fluid Compute + AI Gateway is excellent |
| Marketing site with CMS | Yes | ISR + Edge Config for feature flags |
| High-throughput REST API (100k+ RPS) | Maybe | Works, but evaluate cost at scale |
| Long-running batch jobs (30+ min) | No | Use dedicated compute (ECS, Cloud Run) |
| WebSocket-heavy real-time app | No | Vercel does not support persistent WebSockets at the platform level |
| Regulatory requirement for single-tenant | No | Use your own cloud account |

---

## 2. Fluid Compute

This is the single most important thing Vercel has shipped for backend workloads, and most developers do not fully understand it. Let me fix that.

### The Problem with Traditional Serverless

Traditional serverless (AWS Lambda, the old Vercel model) works like this:

1. A request comes in.
2. The platform spins up an instance of your function.
3. Your function handles the request.
4. The instance sits there, idle, waiting for the next request.
5. If another request comes in while the first is still processing, a *new* instance spins up.
6. After some idle time, the instance is destroyed.

The critical problem is step 5. One instance, one concurrent request. This means:

- **Cold starts multiply.** Under load, you are constantly spinning up new instances.
- **Connection pools are useless.** Each instance has its own database connection. 100 concurrent requests means 100 database connections. Your Postgres instance weeps.
- **Memory is wasted.** Each instance loads the same code, the same dependencies, the same in-memory caches. None of it is shared.

### How Fluid Compute Works

Fluid Compute flips this model:

1. A request comes in.
2. The platform routes it to an existing function instance (or creates one if none exist).
3. While that function is waiting on I/O (database query, API call, file read), another request can be handled by **the same instance**.
4. The instance is reused across concurrent requests, just like a traditional server.

```
Traditional Serverless:          Fluid Compute:
                                 
Request A --> Instance 1         Request A ─┐
Request B --> Instance 2         Request B ─┼──> Instance 1
Request C --> Instance 3         Request C ─┘
Request D --> Instance 4         Request D ─┐
                                 Request E ─┼──> Instance 2
                                 Request F ─┘
```

This changes everything:

- **Cold starts are rare.** Instances are reused, so new instances only spin up when existing ones are actually CPU-saturated.
- **Connection pools work.** A single instance maintains a pool of database connections and shares them across requests.
- **In-memory caches are effective.** A parsed config file, a compiled regex, a loaded ML model — they persist across requests within the same instance.
- **Active CPU pricing is fair.** You are billed for the CPU time your code actually uses, not the wall-clock time an instance exists.

### Active CPU Pricing

This is worth understanding in detail because it changes how you think about cost optimization.

**Old model (wall-clock billing):**
```
Your function runs for 2 seconds.
1.8 seconds of that is waiting for a database query.
0.2 seconds is actual CPU work.
You are billed for 2 seconds.
```

**Fluid Compute (active CPU billing):**
```
Your function runs for 2 seconds.
1.8 seconds of that is waiting for a database query.
0.2 seconds is actual CPU work.
You are billed for 0.2 seconds.
```

This is a massive cost reduction for I/O-heavy workloads, which is... most web applications.

The practical implication: **do not optimize for speed of I/O waits.** Optimize for reducing CPU work. Caching parsed JSON, avoiding unnecessary serialization, using efficient algorithms — these are the things that save money under active CPU pricing.

### Fluid Compute vs. Traditional Approaches

| Dimension | Traditional Serverless | Fluid Compute | Traditional Server (EC2/ECS) |
|-----------|----------------------|---------------|------------------------------|
| Cold starts | Frequent under load | Rare | None (always running) |
| Connection pooling | Per-instance only | Shared across requests | Shared across requests |
| Scaling | Instance per request | Instances scale with CPU need | Manual or auto-scaling |
| Cost model | Pay per invocation + duration | Pay per active CPU time | Pay for uptime |
| Memory sharing | No | Yes, within instance | Yes |
| Idle cost | Low (instances terminate) | Low (instances terminate) | High (always running) |
| Deployment | Push and done | Push and done | Container build + deploy |
| Max concurrency | Limited by instance count | High, fewer instances needed | Limited by instance resources |

### How to Verify Fluid Compute Is Working

You do not need to do anything to enable Fluid Compute. It is the default for all Vercel Functions. But you can verify it is working by logging instance reuse:

```typescript
// api/health.ts
let requestCount = 0;
const instanceId = crypto.randomUUID();

export default function handler(req: Request) {
  requestCount++;
  return Response.json({
    instanceId,
    requestCount,
    message: `This instance has handled ${requestCount} requests`,
  });
}
```

Deploy this, hit it with concurrent requests, and you will see the same `instanceId` with an incrementing `requestCount`. That is Fluid Compute in action.

### Things to Watch Out For

Fluid Compute is excellent, but it introduces patterns you need to respect:

1. **Global state is shared across requests.** If you mutate a global variable in one request, the next request on that instance sees the mutation. This is both a feature (caching) and a footgun (state leaking between users).

```typescript
// DANGEROUS: Leaking state between requests
let currentUser: User | null = null;

export default function handler(req: Request) {
  currentUser = await getUser(req); // This persists!
  // If the next request arrives before this one finishes,
  // it might see the wrong user.
}

// SAFE: Request-scoped state
export default function handler(req: Request) {
  const currentUser = await getUser(req); // Local variable, no leaking
}
```

2. **Connection pools need limits.** Because instances are reused, a connection pool can accumulate connections over time. Set explicit maximums.

```typescript
import { Pool } from 'pg';

// Good: explicit pool limits
const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  max: 10, // Never exceed 10 connections per instance
  idleTimeoutMillis: 30000,
});
```

3. **Memory leaks are amplified.** In traditional serverless, a memory leak is hidden because instances are short-lived. In Fluid Compute, instances live longer, so a leak will eventually cause issues. Profile your memory usage.

---

## 3. Vercel Functions

Vercel Functions are the compute primitive. Every API route, every server-side rendered page, every server action — they all run as Vercel Functions under the hood.

### Runtime Options

As of 2026, you have four runtime options:

| Runtime | Use Case | Cold Start | Ecosystem |
|---------|----------|------------|-----------|
| **Node.js 24 LTS** (default) | General purpose | Fast (Fluid Compute) | Full npm ecosystem |
| **Bun** | Performance-sensitive | Very fast | Growing, npm-compatible |
| **Python** | ML inference, data processing | Moderate | Full pip ecosystem |
| **Rust** | CPU-intensive computation | Minimal | Cargo ecosystem |

#### Node.js 24 (Default)

This is what you should use unless you have a specific reason not to. Node.js 24 brings:

- Native `fetch` and Web APIs
- Stable `AsyncLocalStorage` for request-scoped context
- Performance improvements across the board
- Full compatibility with the npm ecosystem

```typescript
// api/users.ts — runs on Node.js 24 by default
export const config = {
  runtime: 'nodejs', // optional, this is the default
  maxDuration: 60,   // seconds, up to 300
};

export default async function handler(req: Request) {
  const users = await db.query('SELECT * FROM users LIMIT 100');
  return Response.json(users);
}
```

#### Bun Runtime

Bun is fast. Really fast. If you have a CPU-bound function (JSON parsing, template rendering, cryptographic operations), Bun can be 2-3x faster than Node.js for those specific workloads.

```typescript
// api/transform.ts
export const config = {
  runtime: 'bun',
};

export default async function handler(req: Request) {
  const data = await req.json();
  // CPU-intensive transformation benefits from Bun's speed
  const transformed = heavyTransform(data);
  return Response.json(transformed);
}
```

Be aware: the Bun ecosystem is maturing but not as battle-tested as Node.js. If you depend on native Node.js modules (like `bcrypt` with C++ bindings), test thoroughly.

#### Python Runtime

Python on Vercel is not a gimmick. It is a real runtime with access to pip packages, including heavy hitters like numpy, pandas, and scikit-learn.

```python
# api/predict.py
from http.server import BaseHTTPRequestHandler
import json
import joblib

model = joblib.load('model.pkl')  # Loaded once, reused via Fluid Compute

class handler(BaseHTTPRequestHandler):
    def do_POST(self):
        content_length = int(self.headers['Content-Length'])
        body = json.loads(self.rfile.read(content_length))
        
        prediction = model.predict([body['features']])
        
        self.send_response(200)
        self.send_header('Content-Type', 'application/json')
        self.end_headers()
        self.wfile.write(json.dumps({'prediction': prediction.tolist()}).encode())
```

#### Rust Runtime

For the ultimate in performance and low-level control:

```rust
// api/compute/src/main.rs
use vercel_runtime::{run, Body, Error, Request, Response, StatusCode};

#[tokio::main]
async fn main() -> Result<(), Error> {
    run(handler).await
}

pub async fn handler(_req: Request) -> Result<Response<Body>, Error> {
    // CPU-intensive work that benefits from Rust's performance
    let result = expensive_computation();
    
    Ok(Response::builder()
        .status(StatusCode::OK)
        .header("Content-Type", "application/json")
        .body(Body::Text(serde_json::to_string(&result)?))?)
}
```

### Edge Functions: The Deprecated Path

I need to address this directly because there is a lot of outdated content on the internet.

**Edge Functions are deprecated in favor of Fluid Compute.** Do not use them for new projects.

The original pitch for Edge Functions was compelling: run code at the CDN edge, close to users, with sub-millisecond cold starts. The reality was painful:

- No Node.js APIs (no `fs`, no `crypto`, no `Buffer` without polyfills)
- Limited to the Web API surface (like a Service Worker)
- 128MB memory limit
- No native modules
- A completely different mental model for error handling and debugging

Fluid Compute solves the problem Edge Functions were trying to solve (fast cold starts, geographic distribution) without the restrictions. Full Node.js, full npm ecosystem, connection pooling, and instances reused across requests.

If you have existing Edge Functions, they still work. But migrate them to standard Vercel Functions (which use Fluid Compute by default) when you get the chance.

### Streaming Responses

Streaming is first-class in Vercel Functions. This is critical for AI applications where you want to stream tokens to the client as they are generated:

```typescript
// api/chat.ts
export const config = {
  maxDuration: 300, // AI responses can take a while
};

export default async function handler(req: Request) {
  const { messages } = await req.json();

  const stream = new ReadableStream({
    async start(controller) {
      const aiStream = await openai.chat.completions.create({
        model: 'gpt-5.4',
        messages,
        stream: true,
      });

      for await (const chunk of aiStream) {
        const text = chunk.choices[0]?.delta?.content || '';
        controller.enqueue(new TextEncoder().encode(`data: ${text}\n\n`));
      }
      
      controller.enqueue(new TextEncoder().encode('data: [DONE]\n\n'));
      controller.close();
    },
  });

  return new Response(stream, {
    headers: {
      'Content-Type': 'text/event-stream',
      'Cache-Control': 'no-cache',
      'Connection': 'keep-alive',
    },
  });
}
```

### Timeouts and Configuration

The default timeout is 300 seconds (5 minutes) on paid plans, but you should set it explicitly per function:

```typescript
export const config = {
  maxDuration: 30,  // 30 seconds for a typical API route
  memory: 1024,     // 1GB RAM (default is 1024)
  regions: ['iad1'], // US East, or let Vercel choose
};
```

**Region configuration matters.** If your database is in `us-east-1`, your functions should be in `iad1` (Vercel's US East region). Latency between regions is real and measurable.

| Vercel Region | AWS Equivalent | Location |
|---------------|---------------|----------|
| `iad1` | `us-east-1` | Washington, D.C. |
| `sfo1` | `us-west-1` | San Francisco |
| `lhr1` | `eu-west-2` | London |
| `hnd1` | `ap-northeast-1` | Tokyo |
| `cdg1` | `eu-west-3` | Paris |
| `sin1` | `ap-southeast-1` | Singapore |
| `syd1` | `ap-southeast-2` | Sydney |

### Middleware: Full Node.js Now

One of the most significant changes in recent Vercel history: **Middleware now supports full Node.js.** It is no longer restricted to the Edge runtime.

This means you can do things in Middleware that were previously impossible:

```typescript
// middleware.ts
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';
import { verifyJWT } from './lib/auth'; // Uses Node.js crypto
import { getFeatureFlags } from './lib/flags'; // Hits a database

export async function middleware(request: NextRequest) {
  // This was impossible when Middleware was edge-only
  const token = request.headers.get('authorization')?.split('Bearer ')[1];
  
  if (token) {
    const user = await verifyJWT(token); // Full Node.js crypto
    const flags = await getFeatureFlags(user.id); // Database query
    
    const response = NextResponse.next();
    response.headers.set('x-user-id', user.id);
    response.headers.set('x-feature-flags', JSON.stringify(flags));
    return response;
  }

  return NextResponse.redirect(new URL('/login', request.url));
}

export const config = {
  matcher: ['/dashboard/:path*', '/api/:path*'],
};
```

You can choose which runtime Middleware uses:

```typescript
// middleware.ts
export const config = {
  runtime: 'nodejs', // Full Node.js (default now)
  // runtime: 'edge', // Legacy, not recommended
};
```

---

## 4. Vercel Storage

Vercel's storage story has evolved significantly. Here is the current state.

### What Happened to Vercel Postgres and Vercel KV?

They are gone. Vercel no longer offers first-party Postgres or Redis. Instead, they have a Marketplace model where you provision databases from third-party providers (Neon, Upstash, PlanetScale, etc.) directly through the Vercel dashboard.

This is actually a better model. Vercel was never going to out-database companies whose entire business is databases. The Marketplace approach gives you:

- **Best-in-class databases** from specialist providers
- **Automatic environment variable injection** (connection strings are set automatically)
- **Unified billing** through Vercel
- **One-click provisioning** from the Vercel dashboard

### Blob Storage

Vercel Blob is file storage. Think of it as S3 with a developer-friendly API and automatic CDN distribution.

#### When to Use Blob

- User-uploaded files (avatars, documents, images)
- Generated assets (PDFs, reports, exports)
- Static assets that change frequently
- AI-generated content (images, audio)

#### Public vs. Private Blobs

```typescript
import { put, del, list } from '@vercel/blob';

// Public blob — accessible via CDN URL, no authentication needed
export async function uploadAvatar(file: File, userId: string) {
  const blob = await put(`avatars/${userId}.jpg`, file, {
    access: 'public',
    contentType: 'image/jpeg',
  });
  
  // blob.url is a public CDN URL like:
  // https://abc123.public.blob.vercel-storage.com/avatars/user-123.jpg
  return blob.url;
}

// Private blob — requires a token to access
export async function uploadDocument(file: File, userId: string) {
  const blob = await put(`documents/${userId}/${file.name}`, file, {
    access: 'private',
    contentType: file.type,
  });
  
  // blob.url requires authentication to access
  // Use blob.downloadUrl for time-limited access
  return blob;
}
```

#### Blob Operations

```typescript
import { put, del, list, head, copy } from '@vercel/blob';

// List blobs with prefix
const { blobs, cursor } = await list({
  prefix: 'avatars/',
  limit: 100,
});

// Check if a blob exists and get metadata
const metadata = await head('avatars/user-123.jpg');
// { size: 45231, contentType: 'image/jpeg', uploadedAt: '2026-...' }

// Copy a blob
await copy('avatars/user-123.jpg', 'backups/avatars/user-123.jpg');

// Delete a blob
await del('avatars/user-123.jpg');

// Delete multiple blobs
await del([
  'avatars/user-123.jpg',
  'avatars/user-456.jpg',
]);
```

#### Blob with Next.js Server Actions

```typescript
// app/upload/actions.ts
'use server';

import { put } from '@vercel/blob';
import { revalidatePath } from 'next/cache';

export async function uploadFile(formData: FormData) {
  const file = formData.get('file') as File;
  
  if (!file || file.size === 0) {
    return { error: 'No file provided' };
  }

  if (file.size > 10 * 1024 * 1024) {
    return { error: 'File must be under 10MB' };
  }

  const blob = await put(file.name, file, {
    access: 'public',
  });

  // Save blob.url to your database
  await db.files.create({
    url: blob.url,
    name: file.name,
    size: file.size,
  });

  revalidatePath('/files');
  return { url: blob.url };
}
```

### Edge Config

Edge Config is Vercel's ultra-fast read store. It is designed for data that is read extremely frequently and written infrequently.

#### What Makes Edge Config Special

- **Sub-millisecond reads.** The data is embedded in the runtime, not fetched over the network.
- **Global distribution.** Available at every Vercel edge location.
- **Eventually consistent writes.** When you update Edge Config, it propagates globally in seconds.

#### When to Use Edge Config

| Use Case | Edge Config? | Why |
|----------|-------------|-----|
| Feature flags | Yes | Read on every request, change occasionally |
| A/B test configuration | Yes | Read often, update periodically |
| Maintenance mode toggle | Yes | Perfect fit |
| Redirect maps | Yes | Read on every request |
| User data | No | Too dynamic, use a database |
| Session storage | No | Writes on every request |
| Rate limiting counters | No | Requires atomic writes |

#### Edge Config in Practice

```typescript
import { get, getAll } from '@vercel/edge-config';

// Read a single key
const isMaintenanceMode = await get<boolean>('maintenanceMode');

// Read multiple keys
const config = await getAll<{
  maintenanceMode: boolean;
  featureFlags: Record<string, boolean>;
  redirects: Array<{ from: string; to: string }>;
}>();

// Use in Middleware for feature flags
// middleware.ts
import { get } from '@vercel/edge-config';
import { NextResponse } from 'next/server';

export async function middleware(request: NextRequest) {
  const flags = await get<Record<string, boolean>>('featureFlags');
  
  if (flags?.newCheckout && request.nextUrl.pathname === '/checkout') {
    return NextResponse.rewrite(new URL('/checkout-v2', request.url));
  }
  
  return NextResponse.next();
}
```

#### Updating Edge Config

Edge Config is not a database. You update it through the API or the Vercel dashboard:

```typescript
import { createClient } from '@vercel/edge-config';

const edgeConfig = createClient(process.env.EDGE_CONFIG);

// Read (fast, sub-millisecond)
const value = await edgeConfig.get('featureFlags');

// Write (slower, goes through the API)
// Typically done from a CI/CD pipeline or admin panel
await fetch(
  `https://api.vercel.com/v1/edge-config/${edgeConfigId}/items`,
  {
    method: 'PATCH',
    headers: {
      Authorization: `Bearer ${process.env.VERCEL_API_TOKEN}`,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({
      items: [
        {
          operation: 'upsert',
          key: 'featureFlags',
          value: { newCheckout: true, darkMode: false },
        },
      ],
    }),
  }
);
```

### Marketplace Databases

Instead of first-party database offerings, Vercel now integrates with specialist providers through the Marketplace.

#### Neon Postgres

Neon is the recommended Postgres provider on Vercel. It has serverless-friendly connection pooling and scales to zero.

```typescript
// Using Neon with Drizzle ORM (recommended)
import { drizzle } from 'drizzle-orm/neon-http';
import { neon } from '@neondatabase/serverless';

const sql = neon(process.env.DATABASE_URL!);
const db = drizzle(sql);

// Queries work like any Postgres client
const users = await db.select().from(usersTable).limit(10);
```

Provisioning through the Vercel Marketplace:

```bash
# Through the CLI
vercel integration add neon

# Or through the dashboard: Settings > Integrations > Browse Marketplace
```

After provisioning, the `DATABASE_URL` environment variable is automatically set in your project.

#### Upstash Redis

Upstash is the recommended Redis provider. It offers HTTP-based Redis (no persistent connections needed) and a generous free tier.

```typescript
import { Redis } from '@upstash/redis';

const redis = new Redis({
  url: process.env.KV_REST_API_URL!,
  token: process.env.KV_REST_API_TOKEN!,
});

// Standard Redis operations
await redis.set('user:123:session', JSON.stringify(session), { ex: 3600 });
const session = await redis.get<Session>('user:123:session');

// Rate limiting with Upstash
import { Ratelimit } from '@upstash/ratelimit';

const ratelimit = new Ratelimit({
  redis,
  limiter: Ratelimit.slidingWindow(10, '10s'), // 10 requests per 10 seconds
});

export async function middleware(request: NextRequest) {
  const ip = request.ip ?? '127.0.0.1';
  const { success, limit, remaining } = await ratelimit.limit(ip);
  
  if (!success) {
    return new NextResponse('Too Many Requests', {
      status: 429,
      headers: {
        'X-RateLimit-Limit': limit.toString(),
        'X-RateLimit-Remaining': remaining.toString(),
      },
    });
  }
  
  return NextResponse.next();
}
```

### Storage Decision Matrix

| Need | Solution | Why |
|------|----------|-----|
| File uploads | Blob | Purpose-built, CDN-distributed |
| Feature flags | Edge Config | Sub-millisecond reads |
| Relational data | Neon Postgres (Marketplace) | Serverless-friendly, scales to zero |
| Caching / sessions | Upstash Redis (Marketplace) | HTTP-based, no connection overhead |
| Full-text search | Typesense / Meilisearch | Marketplace or self-hosted |
| Vector embeddings | Pinecone / Weaviate (Marketplace) | Specialized vector stores |
| File storage (large) | AWS S3 / Cloudflare R2 | More cost-effective at scale |

---

## 5. Vercel AI Gateway

If you are building AI-powered applications in 2026 (and statistically, you probably are), the AI Gateway deserves your attention.

### What It Does

The Vercel AI Gateway sits between your application and AI providers (OpenAI, Anthropic, Google, Mistral, Cohere, and others). It provides:

1. **Multi-provider routing** — Send requests to different providers based on model, cost, or availability.
2. **Automatic failover** — If OpenAI is down, automatically route to Anthropic.
3. **Observability** — See every AI request, its latency, token count, and cost.
4. **Cost tracking** — Real-time dashboards showing spend per model, per user, per feature.
5. **Rate limiting** — Protect against runaway AI costs.
6. **Caching** — Cache identical prompts to avoid redundant API calls.

### Why Not Just Call the API Directly?

You can. But here is what happens at scale:

1. **Provider outages become your outages.** OpenAI has had multiple significant outages. Without failover, your product goes down when they do.
2. **Cost surprises.** An AI feature that seems cheap in development can generate massive bills when real users start hammering it. Without observability, you find out from your credit card statement.
3. **Debugging is blind.** When an AI response is wrong, you need to see the exact prompt, the exact response, the token count, and the latency. Without centralized logging, this is painful.

### Using the AI Gateway with the Vercel AI SDK

```typescript
// app/api/chat/route.ts
import { streamText } from 'ai';
import { gateway } from '@ai-sdk/gateway';

export async function POST(req: Request) {
  const { messages } = await req.json();

  // The AI Gateway routes through Vercel's unified proxy.
  // Failover, logging, and cost tracking are handled at the
  // platform level — no direct provider imports needed.
  const result = streamText({
    model: gateway('openai/gpt-5.4'),  // Primary model via AI Gateway
    messages,
    // AI Gateway handles failover to anthropic/claude-sonnet-4.5
    // if OpenAI is unavailable (configured in dashboard)
  });

  return result.toUIMessageStreamResponse();
}
```

### Configuring Provider Failover

In the Vercel dashboard (or via API), you configure failover chains:

```
Primary:  openai/gpt-5.4
Fallback: anthropic/claude-sonnet-4.5
Fallback: google/gemini-2.5-pro

Routing rules:
- If primary returns 5xx: try next provider
- If primary latency > 10s: try next provider
- If primary rate limited (429): try next provider
```

### Cost Controls

```typescript
// You can set cost alerts and limits per-project
// This is configured in the Vercel dashboard, but here is the mental model:

// Per-user limits
const AI_COST_LIMITS = {
  free: { daily: 0.50, monthly: 10 },
  pro: { daily: 5.00, monthly: 100 },
  enterprise: { daily: 50.00, monthly: 1000 },
};
```

The AI Gateway tracks costs in real-time and can block requests when limits are exceeded, preventing the "I left my API key in production and got a $50,000 bill" scenario.

### Observability Dashboard

The AI Gateway dashboard shows:

- **Request volume** by model, endpoint, and user
- **Token usage** (input vs. output, per model)
- **Cost breakdown** (per model, per endpoint, daily/weekly/monthly)
- **Latency percentiles** (p50, p95, p99 per model)
- **Error rates** by provider
- **Cache hit rates** for cached prompts

This is the kind of observability that teams usually build custom solutions for. Having it built into the platform is a significant time saver.

---

## 6. Vercel Queues

Queues are the newest major primitive on the Vercel platform, and they solve a real problem: durable, asynchronous event processing without managing infrastructure.

### The Problem Queues Solve

Not every operation should happen synchronously in a request/response cycle:

- Sending a welcome email after signup
- Processing an uploaded image (resize, optimize, generate thumbnails)
- Syncing data to a third-party CRM
- Generating a PDF report
- Running an AI pipeline that takes 30 seconds

Without queues, you end up with one of these bad patterns:

1. **Doing it in the request handler** — Slow response, risk of timeout.
2. **Fire-and-forget fetch calls** — If the function shuts down, the work is lost.
3. **External queue service (SQS, RabbitMQ)** — Works, but now you are managing infrastructure.

### How Vercel Queues Work

Vercel Queues provide durable event streaming with at-least-once delivery:

```typescript
// lib/queues.ts
import { Queue } from '@vercel/queue';

// Define a queue with a handler
export const emailQueue = new Queue('send-email', {
  handler: async (event: { to: string; template: string; data: Record<string, unknown> }) => {
    await sendEmail(event.to, event.template, event.data);
  },
  retries: 3,
  retryDelay: '1m', // Wait 1 minute between retries
});

export const imageProcessingQueue = new Queue('process-image', {
  handler: async (event: { blobUrl: string; userId: string }) => {
    const image = await fetch(event.blobUrl).then(r => r.arrayBuffer());
    const resized = await sharp(image).resize(800, 600).toBuffer();
    await put(`processed/${event.userId}/thumb.jpg`, resized, { access: 'public' });
  },
  retries: 5,
  retryDelay: '30s',
  maxDuration: 120, // 2 minutes max for image processing
});
```

#### Publishing to a Queue

```typescript
// app/api/signup/route.ts
import { emailQueue } from '@/lib/queues';

export async function POST(req: Request) {
  const { email, name } = await req.json();
  
  // Create the user synchronously
  const user = await db.users.create({ email, name });
  
  // Queue the welcome email — returns immediately
  await emailQueue.publish({
    to: email,
    template: 'welcome',
    data: { name },
  });
  
  // Response is fast, email is sent asynchronously
  return Response.json({ user });
}
```

#### Queue Guarantees

| Property | Value |
|----------|-------|
| Delivery | At-least-once |
| Ordering | Best-effort (not strict FIFO) |
| Retention | 7 days |
| Max message size | 256 KB |
| Max retries | Configurable, up to 10 |
| Retry delay | Configurable, exponential backoff available |
| Concurrency | Configurable per queue |
| Visibility timeout | Configurable |

**At-least-once delivery means your handlers must be idempotent.** If your handler sends an email, make sure sending the same email twice is not a disaster. Use idempotency keys:

```typescript
export const paymentQueue = new Queue('process-payment', {
  handler: async (event: { orderId: string; amount: number; idempotencyKey: string }) => {
    // Check if we already processed this payment
    const existing = await db.payments.findByIdempotencyKey(event.idempotencyKey);
    if (existing) {
      console.log(`Payment ${event.idempotencyKey} already processed, skipping`);
      return;
    }
    
    await processPayment(event.orderId, event.amount);
    await db.payments.create({
      orderId: event.orderId,
      amount: event.amount,
      idempotencyKey: event.idempotencyKey,
    });
  },
});
```

### Queue Use Cases

| Use Case | Why a Queue? |
|----------|-------------|
| Email sending | Do not block the request on SMTP |
| Image processing | CPU-intensive, might timeout |
| Webhook delivery | Retry on failure, do not block the original action |
| Data sync (CRM, analytics) | Non-critical, can be eventual |
| PDF generation | Slow, resource-intensive |
| AI batch processing | Process multiple items without holding a connection |
| Audit logging | Fire-and-forget, with durability |

---

## 7. Deployment Pipeline

Vercel's deployment pipeline is its original superpower, and it has only gotten better.

### Preview Deployments

Every push to a non-production branch creates a preview deployment. This is not a staging environment — it is a full, isolated deployment with its own URL.

```
Push to feature/new-checkout
→ Vercel builds and deploys
→ https://your-project-abc123-team.vercel.app
→ Unique URL, runs against preview environment variables
→ GitHub/GitLab comment with the URL and build status
```

Preview deployments are powerful because:

1. **Every PR is reviewable.** Designers, PMs, and QA can click a link and see the change.
2. **Preview environment variables** are separate from production. You can point previews at a staging database.
3. **Preview deployments expire.** They do not run forever, so you do not pay for unused deployments.

### Production Deployments

Merging to your production branch (usually `main`) triggers a production deployment:

```
Merge PR to main
→ Vercel builds and deploys
→ Atomic deployment — new version goes live only when fully built
→ Zero-downtime switch — the old version serves requests until the new one is ready
→ Instant rollback available
```

### Rolling Releases (Canary Deployments)

This is one of the most operationally important features for production systems. Rolling Releases let you gradually shift traffic from the old deployment to the new one.

```
Deploy new version
→ 1% of traffic goes to the new version
→ Monitor error rates and latency
→ If healthy, ramp to 10%, then 50%, then 100%
→ If unhealthy, automatically roll back to 0%
```

You configure this in the Vercel dashboard under Project Settings > Rolling Releases.

The key insight: **this is not just about reducing blast radius.** It is about confidence. When you know that a bad deploy will only affect 1% of users and will automatically roll back, you deploy more frequently. More frequent deploys means smaller changes, which means easier debugging when something goes wrong.

### Instant Rollback

Every deployment on Vercel is immutable and retained. Rolling back is not "redeploy the old code" — it is "point traffic at the old deployment." This is instant because no build is required.

```bash
# Rollback to a specific deployment
vercel rollback dpl_abc123

# Rollback to the previous production deployment
vercel rollback
```

### Environment Variables

Vercel has a mature environment variable system with scoping:

```bash
# Add an environment variable
vercel env add DATABASE_URL production
# Prompts for the value

# Add to all environments at once
vercel env add API_KEY production preview development

# List environment variables
vercel env ls

# Pull environment variables to .env.local
vercel env pull .env.local

# Remove an environment variable
vercel env rm DATABASE_URL production
```

Environment variable scoping:

| Scope | When Used |
|-------|-----------|
| `production` | Only on production deployments |
| `preview` | Only on preview deployments |
| `development` | Only when pulled locally via `vercel env pull` |

**Best practice:** Use different database connections for each scope. Production hits your production database. Preview hits a staging database (or a Neon branch). Development hits a local database.

### Domain Configuration

```bash
# Add a custom domain
vercel domains add yourdomain.com

# Add a subdomain
vercel domains add api.yourdomain.com

# List domains
vercel domains ls

# Verify DNS configuration
vercel domains verify yourdomain.com
```

Vercel automatically provisions and renews TLS certificates for all domains.

---

## 8. vercel.ts Configuration

The `vercel.json` file you know is still supported, but `vercel.ts` is the modern approach. TypeScript configuration means autocomplete, type checking, and the ability to use logic in your configuration.

### Basic vercel.ts

```typescript
// vercel.ts
import { defineConfig } from '@vercel/config';

export default defineConfig({
  // Build configuration
  buildCommand: 'pnpm build',
  outputDirectory: '.next',
  installCommand: 'pnpm install',
  framework: 'nextjs',

  // Function defaults
  functions: {
    'api/**': {
      memory: 1024,
      maxDuration: 60,
    },
    'api/ai/**': {
      memory: 3008,
      maxDuration: 300,
    },
  },

  // Region configuration
  regions: ['iad1'],
});
```

### Rewrites

```typescript
import { defineConfig } from '@vercel/config';

export default defineConfig({
  rewrites: [
    // Proxy API requests to a different backend
    {
      source: '/api/v1/:path*',
      destination: 'https://api.backend.com/:path*',
    },
    // SPA fallback — serve index.html for all unmatched routes
    {
      source: '/((?!api/).*)',
      destination: '/index.html',
    },
  ],
});
```

### Redirects

```typescript
import { defineConfig } from '@vercel/config';

export default defineConfig({
  redirects: [
    // Permanent redirect (301)
    {
      source: '/blog/:slug',
      destination: '/posts/:slug',
      permanent: true,
    },
    // Temporary redirect (307)
    {
      source: '/docs',
      destination: 'https://docs.example.com',
      permanent: false,
    },
    // Regex-based redirect
    {
      source: '/old-product/:id(\\d+)',
      destination: '/products/:id',
      permanent: true,
    },
  ],
});
```

### Headers

```typescript
import { defineConfig } from '@vercel/config';

export default defineConfig({
  headers: [
    {
      source: '/api/:path*',
      headers: [
        { key: 'Access-Control-Allow-Origin', value: 'https://yourdomain.com' },
        { key: 'Access-Control-Allow-Methods', value: 'GET, POST, PUT, DELETE' },
        { key: 'Access-Control-Allow-Headers', value: 'Content-Type, Authorization' },
      ],
    },
    {
      source: '/:path*',
      headers: [
        { key: 'X-Frame-Options', value: 'DENY' },
        { key: 'X-Content-Type-Options', value: 'nosniff' },
        { key: 'Referrer-Policy', value: 'strict-origin-when-cross-origin' },
        { key: 'Permissions-Policy', value: 'camera=(), microphone=(), geolocation=()' },
      ],
    },
    // Cache static assets aggressively
    {
      source: '/static/:path*',
      headers: [
        { key: 'Cache-Control', value: 'public, max-age=31536000, immutable' },
      ],
    },
  ],
});
```

### Cron Jobs

```typescript
import { defineConfig } from '@vercel/config';

export default defineConfig({
  crons: [
    {
      path: '/api/cron/daily-report',
      schedule: '0 9 * * *', // Every day at 9 AM UTC
    },
    {
      path: '/api/cron/cleanup',
      schedule: '0 */6 * * *', // Every 6 hours
    },
    {
      path: '/api/cron/sync-data',
      schedule: '*/15 * * * *', // Every 15 minutes
    },
  ],
});
```

The cron endpoint should verify the request is from Vercel:

```typescript
// api/cron/daily-report.ts
export async function GET(req: Request) {
  // Verify the request is from Vercel Cron
  const authHeader = req.headers.get('authorization');
  if (authHeader !== `Bearer ${process.env.CRON_SECRET}`) {
    return new Response('Unauthorized', { status: 401 });
  }

  // Do the work
  await generateDailyReport();
  
  return Response.json({ success: true });
}
```

### A Complete vercel.ts Example

Here is a realistic, production-grade configuration:

```typescript
// vercel.ts
import { defineConfig } from '@vercel/config';

const isProduction = process.env.VERCEL_ENV === 'production';

export default defineConfig({
  // Build
  buildCommand: 'pnpm turbo build',
  outputDirectory: 'apps/web/.next',
  installCommand: 'pnpm install --frozen-lockfile',
  framework: 'nextjs',

  // Regions — colocate with database
  regions: ['iad1'],

  // Functions
  functions: {
    'api/**': {
      memory: 1024,
      maxDuration: 30,
    },
    'api/ai/**': {
      memory: 3008,
      maxDuration: 300,
    },
    'api/webhooks/**': {
      memory: 512,
      maxDuration: 60,
    },
  },

  // Rewrites
  rewrites: [
    // Proxy analytics to avoid ad blockers
    {
      source: '/data/:path*',
      destination: 'https://plausible.io/:path*',
    },
  ],

  // Redirects
  redirects: [
    { source: '/blog', destination: '/posts', permanent: true },
    { source: '/pricing', destination: '/plans', permanent: false },
  ],

  // Security headers
  headers: [
    {
      source: '/:path*',
      headers: [
        { key: 'X-Frame-Options', value: 'DENY' },
        { key: 'X-Content-Type-Options', value: 'nosniff' },
        { key: 'Referrer-Policy', value: 'strict-origin-when-cross-origin' },
        {
          key: 'Content-Security-Policy',
          value: isProduction
            ? "default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline';"
            : '', // Relaxed CSP in development
        },
        {
          key: 'Strict-Transport-Security',
          value: 'max-age=63072000; includeSubDomains; preload',
        },
      ],
    },
  ],

  // Cron jobs
  crons: [
    { path: '/api/cron/daily-report', schedule: '0 9 * * *' },
    { path: '/api/cron/cleanup-expired', schedule: '0 3 * * *' },
    { path: '/api/cron/sync-analytics', schedule: '*/30 * * * *' },
  ],
});
```

### vercel.ts vs. vercel.json

| Feature | vercel.ts | vercel.json |
|---------|-----------|-------------|
| Type checking | Yes | No |
| Autocomplete | Yes | No (unless using JSON schema) |
| Conditional logic | Yes | No |
| Environment-aware | Yes (`process.env`) | No |
| Comments | Yes | No |
| Import other files | Yes | No |
| Backward compatible | N/A | Still supported |

My recommendation: use `vercel.ts` for new projects. Migrate existing `vercel.json` when you next need to change configuration.

---

## 9. Analytics & Observability

### Web Analytics

Vercel Web Analytics gives you privacy-friendly, cookie-free analytics. No consent banners needed.

```typescript
// app/layout.tsx
import { Analytics } from '@vercel/analytics/react';

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html>
      <body>
        {children}
        <Analytics />
      </body>
    </html>
  );
}
```

What you get:

- Page views and unique visitors
- Top pages
- Referrer tracking
- Country and device breakdown
- Custom events

#### Custom Events

```typescript
import { track } from '@vercel/analytics';

// Track a signup
track('signup', {
  plan: 'pro',
  source: 'pricing-page',
});

// Track a purchase
track('purchase', {
  amount: 99,
  currency: 'USD',
  product: 'annual-plan',
});

// Track a feature usage
track('feature-used', {
  feature: 'ai-chat',
  messageCount: 5,
});
```

### Speed Insights

Speed Insights measures real user performance (Core Web Vitals) from actual visitors:

```typescript
// app/layout.tsx
import { SpeedInsights } from '@vercel/speed-insights/next';

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html>
      <body>
        {children}
        <SpeedInsights />
      </body>
    </html>
  );
}
```

This reports:

- **LCP** (Largest Contentful Paint) — How fast the main content loads
- **FID/INP** (Interaction to Next Paint) — How responsive the page is
- **CLS** (Cumulative Layout Shift) — How stable the visual layout is
- **TTFB** (Time to First Byte) — Server response time
- **FCP** (First Contentful Paint) — When the first pixel renders

Each metric is broken down by page, device type, and country. You can see which pages are slow and why.

### Feature Flags

Vercel has a native feature flags system that integrates with Edge Config:

```typescript
import { unstable_flag as flag } from '@vercel/flags/next';

export const showNewCheckout = flag({
  key: 'new-checkout',
  decide: async () => {
    // Could read from Edge Config, a database, or any source
    const flags = await get('featureFlags');
    return flags?.newCheckout ?? false;
  },
});

// Usage in a Server Component
import { showNewCheckout } from '@/flags';

export default async function CheckoutPage() {
  const useNewCheckout = await showNewCheckout();

  if (useNewCheckout) {
    return <NewCheckoutFlow />;
  }

  return <LegacyCheckoutFlow />;
}
```

Feature flags can also be combined with Rolling Releases for percentage-based rollouts:

```typescript
export const showNewCheckout = flag({
  key: 'new-checkout',
  decide: async (params) => {
    // Roll out to 10% of users
    const hash = hashUserId(params.visitor.id);
    return hash % 100 < 10;
  },
});
```

### Vercel Agent

Vercel Agent is an AI-powered code review and incident investigation tool. When connected to your repository, it:

1. **Reviews PRs automatically** — Catches performance regressions, security issues, and best practice violations.
2. **Investigates incidents** — When an anomaly is detected (error rate spike, latency increase), the Agent correlates it with recent deployments and code changes.
3. **Suggests fixes** — Not just "this is wrong" but "here is how to fix it."

To enable it, install the Vercel Agent GitHub App and configure it in your project settings.

---

## 10. Security

### BotID

BotID is Vercel's bot detection and classification system. It identifies whether incoming traffic is from a human, a search engine crawler, an AI scraper, or a malicious bot.

```typescript
// middleware.ts
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';

export function middleware(request: NextRequest) {
  const botInfo = request.headers.get('x-vercel-bot-info');
  
  if (botInfo) {
    const bot = JSON.parse(botInfo);
    
    // Block known malicious bots
    if (bot.category === 'malicious') {
      return new NextResponse('Forbidden', { status: 403 });
    }
    
    // Rate limit AI scrapers
    if (bot.category === 'ai-scraper') {
      // Serve a simplified version or rate limit
      return NextResponse.rewrite(new URL('/robots-limited', request.url));
    }
    
    // Let search engines through
    if (bot.category === 'search-engine') {
      return NextResponse.next();
    }
  }
  
  return NextResponse.next();
}
```

BotID categories:

| Category | Examples | Typical Action |
|----------|----------|----------------|
| `search-engine` | Googlebot, Bingbot | Allow, serve pre-rendered |
| `social` | Twitterbot, FacebookBot | Allow, serve OG previews |
| `ai-scraper` | GPTBot, ClaudeBot, CommonCrawl | Rate limit or block |
| `monitoring` | Pingdom, UptimeRobot | Allow |
| `malicious` | Known bad actors, DDoS | Block |
| `unclassified` | Unknown bots | Apply default policy |

### DDoS Protection

Vercel includes DDoS protection at every tier. This is not something you configure — it is built into the infrastructure. The Vercel edge network absorbs volumetric attacks before they reach your functions.

For application-layer protection, combine with rate limiting:

```typescript
// middleware.ts
import { Ratelimit } from '@upstash/ratelimit';
import { Redis } from '@upstash/redis';

const ratelimit = new Ratelimit({
  redis: Redis.fromEnv(),
  limiter: Ratelimit.fixedWindow(100, '60s'),
  analytics: true,
});

export async function middleware(request: NextRequest) {
  // DDoS protection is automatic at the infrastructure level
  // Application-level rate limiting adds another layer
  
  const ip = request.ip ?? request.headers.get('x-forwarded-for') ?? '127.0.0.1';
  const { success } = await ratelimit.limit(ip);
  
  if (!success) {
    return new NextResponse('Rate Limited', { status: 429 });
  }
  
  return NextResponse.next();
}
```

### Firewall & URL Whitelisting

Vercel Firewall lets you create rules at the platform level:

```
Rule 1: Block all traffic from country X
Rule 2: Allow only /api/webhooks from IP range 203.0.113.0/24
Rule 3: Rate limit /api/auth/* to 10 requests per minute per IP
Rule 4: Block requests with suspicious User-Agent patterns
```

These rules execute before your code runs, at the edge. This means blocked requests never consume your function compute.

For webhook endpoints, whitelist the source IPs:

```typescript
// vercel.ts
import { defineConfig } from '@vercel/config';

export default defineConfig({
  // Firewall rules can also be defined in config
  firewall: {
    rules: [
      {
        name: 'stripe-webhooks-only',
        path: '/api/webhooks/stripe',
        action: 'allow',
        conditions: {
          sourceIp: [
            '3.18.12.63',
            '3.130.192.231',
            '13.235.14.237',
            '13.235.122.149',
            // ... Stripe's IP ranges
          ],
        },
      },
    ],
  },
});
```

### Vercel Sandbox

Vercel Sandbox provides ephemeral Firecracker microVMs for running untrusted code safely. This is critical for:

- **AI code generation** — Let users run AI-generated code without risking your infrastructure.
- **Code playgrounds** — Offer interactive code examples that execute server-side.
- **Agent workflows** — AI agents that need to execute code as part of their reasoning.

```typescript
import { Sandbox } from '@vercel/sandbox';

export async function POST(req: Request) {
  const { code } = await req.json();

  const sandbox = await Sandbox.create({
    runtime: 'nodejs',
    timeout: 30000, // 30 seconds max
    memory: 512,    // 512MB
  });

  try {
    const result = await sandbox.execute(code);
    return Response.json({
      output: result.stdout,
      error: result.stderr,
      exitCode: result.exitCode,
    });
  } finally {
    await sandbox.destroy();
  }
}
```

Each sandbox is:

- **Isolated** — Firecracker microVM, not just a container. Full hardware-level isolation.
- **Ephemeral** — Created and destroyed per execution. No state persists.
- **Resource-limited** — Configurable memory, CPU, and timeout limits.
- **Network-restricted** — You control what the sandbox can access.

---

## 11. The Vercel CLI

The Vercel CLI is how you interact with the platform from your terminal. Here are the commands you will actually use.

### Setup and Linking

```bash
# Install the CLI
npm i -g vercel

# Login
vercel login

# Link a local project to a Vercel project
vercel link

# This creates .vercel/project.json with your project and org IDs
```

### Deployment

```bash
# Deploy to preview
vercel

# Deploy to production
vercel --prod

# Deploy a specific directory
vercel ./dist --prod

# Deploy with a specific name
vercel --name my-project

# Deploy and skip build (use prebuilt output)
vercel deploy --prebuilt
```

### Environment Variables

```bash
# List all environment variables
vercel env ls

# Add a variable (interactive)
vercel env add SECRET_KEY production

# Pull variables to .env.local
vercel env pull .env.local

# Pull variables for a specific environment
vercel env pull .env.production --environment production

# Remove a variable
vercel env rm SECRET_KEY production
```

### Logs and Debugging

```bash
# View recent logs
vercel logs

# Stream logs in real-time
vercel logs --follow

# View logs for a specific deployment
vercel logs https://your-deployment-url.vercel.app

# Inspect a deployment
vercel inspect dpl_abc123
```

### Domain Management

```bash
# List domains
vercel domains ls

# Add a domain
vercel domains add yourdomain.com

# Remove a domain
vercel domains rm yourdomain.com

# Check DNS configuration
vercel domains verify yourdomain.com
```

### Development

```bash
# Start local development server (with Vercel environment)
vercel dev

# This runs your framework's dev server but with:
# - Vercel environment variables injected
# - Serverless functions emulated
# - Edge Config available locally
```

### Project Management

```bash
# List your projects
vercel projects ls

# List deployments
vercel ls

# Promote a deployment to production
vercel promote dpl_abc123

# Rollback to previous deployment
vercel rollback

# Remove a deployment
vercel rm dpl_abc123
```

### Pull and Build (CI/CD Integration)

```bash
# Pull project settings and env vars (useful in CI)
vercel pull --yes --environment production

# Build locally using Vercel's build system
vercel build

# Deploy prebuilt output (skip build on Vercel)
vercel deploy --prebuilt
```

This is the pattern for custom CI/CD pipelines:

```yaml
# .github/workflows/deploy.yml
name: Deploy to Vercel
on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Install Vercel CLI
        run: npm install -g vercel
      
      - name: Pull Vercel Configuration
        run: vercel pull --yes --environment=production --token=${{ secrets.VERCEL_TOKEN }}
      
      - name: Build
        run: vercel build --prod --token=${{ secrets.VERCEL_TOKEN }}
      
      - name: Deploy
        run: vercel deploy --prebuilt --prod --token=${{ secrets.VERCEL_TOKEN }}
```

---

## 12. The Self-Hosted Alternative on AWS

I promised honest trade-off analysis, so here it is. Vercel is excellent, but it is a vendor. Some teams need to self-host. Some teams want to self-host. And some teams should consider it even if they do not want to.

### When to Consider Self-Hosting

| Reason | Legitimacy | Notes |
|--------|-----------|-------|
| Regulatory compliance (data sovereignty) | High | Some industries require specific cloud accounts |
| Cost at scale (>$10k/mo on Vercel) | High | AWS is usually cheaper at high volume |
| Existing AWS infrastructure | Medium | If you already have the expertise and infra |
| Team familiarity with AWS | Medium | Vercel is faster to learn, but AWS skills exist |
| "We might need to leave Vercel someday" | Low | This is YAGNI unless you have a specific reason |
| "I want full control" | Low-Medium | Legitimate, but comes with a maintenance cost |

### The Self-Hosted Stack

If you are going to self-host a Next.js (or any modern framework) application, here are your options from least to most control.

#### Option 1: AWS Amplify (Managed, Least Control)

AWS Amplify is Amazon's answer to Vercel. It deploys Next.js apps with SSR support, preview deployments, and environment variables.

```bash
# Install Amplify CLI
npm install -g @aws-amplify/cli

# Initialize
amplify init

# Add hosting
amplify add hosting

# Deploy
amplify publish
```

**Pros:** Closest to Vercel's DX on AWS. Managed, minimal infrastructure knowledge needed.

**Cons:** Feature lag behind Vercel. Limited customization. The SSR implementation is not always up to date with the latest Next.js features. Smaller community.

#### Option 2: OpenNext + SST (Serverless, Moderate Control)

OpenNext is an open-source adapter that converts Next.js's build output into a format compatible with AWS Lambda, CloudFront, and S3. SST (Serverless Stack) is a framework for deploying serverless applications on AWS.

```typescript
// sst.config.ts
import { SSTConfig } from 'sst';
import { NextjsSite } from 'sst/constructs';

export default {
  config() {
    return {
      name: 'my-app',
      region: 'us-east-1',
    };
  },
  stacks(app) {
    app.stack(function Site({ stack }) {
      const site = new NextjsSite(stack, 'site', {
        // OpenNext handles the conversion automatically
        customDomain: {
          domainName: 'yourdomain.com',
          domainAlias: 'www.yourdomain.com',
        },
        environment: {
          DATABASE_URL: process.env.DATABASE_URL!,
          NEXT_PUBLIC_API_URL: 'https://api.yourdomain.com',
        },
      });

      stack.addOutputs({
        URL: site.url,
      });
    });
  },
} satisfies SSTConfig;
```

```bash
# Deploy with SST
npx sst deploy --stage production
```

**What this creates on AWS:**

```
CloudFront (CDN)
├── S3 Bucket (static assets)
├── Lambda@Edge (middleware, SSR)
├── Lambda (API routes, ISR revalidation)
├── SQS (ISR queue)
└── DynamoDB (ISR cache)
```

**Pros:** Serverless, scales to zero, uses standard AWS services, good cost efficiency, open source.

**Cons:** More complex than Vercel deploy. Cold starts (no Fluid Compute equivalent). OpenNext can lag behind Next.js releases by weeks or months. Debugging distributed AWS services is harder than checking the Vercel dashboard.

#### Option 3: ECS Fargate (Containerized, Full Control)

Run Next.js as a Docker container on ECS Fargate. This is the "treat it like a real server" approach.

```dockerfile
# Dockerfile
FROM node:24-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:24-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production
COPY --from=builder /app/.next/standalone ./
COPY --from=builder /app/.next/static ./.next/static
COPY --from=builder /app/public ./public

EXPOSE 3000
CMD ["node", "server.js"]
```

```typescript
// infra/main.ts (AWS CDK)
import * as cdk from 'aws-cdk-lib';
import * as ecs from 'aws-cdk-lib/aws-ecs';
import * as ec2 from 'aws-cdk-lib/aws-ec2';
import * as elbv2 from 'aws-cdk-lib/aws-elasticloadbalancingv2';
import * as cloudfront from 'aws-cdk-lib/aws-cloudfront';

export class NextjsStack extends cdk.Stack {
  constructor(scope: cdk.App, id: string) {
    super(scope, id);

    const vpc = new ec2.Vpc(this, 'Vpc', { maxAzs: 2 });

    const cluster = new ecs.Cluster(this, 'Cluster', { vpc });

    const taskDefinition = new ecs.FargateTaskDefinition(this, 'Task', {
      memoryLimitMiB: 1024,
      cpu: 512,
    });

    taskDefinition.addContainer('NextJs', {
      image: ecs.ContainerImage.fromAsset('.'),
      portMappings: [{ containerPort: 3000 }],
      environment: {
        NODE_ENV: 'production',
        DATABASE_URL: 'your-database-url',
      },
      logging: ecs.LogDrivers.awsLogs({ streamPrefix: 'nextjs' }),
    });

    const service = new ecs.FargateService(this, 'Service', {
      cluster,
      taskDefinition,
      desiredCount: 2,
      assignPublicIp: false,
    });

    const alb = new elbv2.ApplicationLoadBalancer(this, 'ALB', {
      vpc,
      internetFacing: true,
    });

    const listener = alb.addListener('Listener', { port: 443 });
    listener.addTargets('Target', {
      port: 3000,
      targets: [service],
      healthCheck: {
        path: '/api/health',
        interval: cdk.Duration.seconds(30),
      },
    });

    // CloudFront in front of the ALB for caching
    new cloudfront.Distribution(this, 'CDN', {
      defaultBehavior: {
        origin: new origins.LoadBalancerV2Origin(alb),
        viewerProtocolPolicy: cloudfront.ViewerProtocolPolicy.REDIRECT_TO_HTTPS,
        cachePolicy: cloudfront.CachePolicy.CACHING_OPTIMIZED,
      },
      additionalBehaviors: {
        '/_next/static/*': {
          origin: new origins.S3Origin(staticBucket),
          cachePolicy: cloudfront.CachePolicy.CACHING_OPTIMIZED,
        },
      },
    });
  }
}
```

**Pros:** Full control over the runtime, networking, and scaling. Standard Docker containers, no vendor lock-in. You understand every piece of the stack. Consistent performance (no cold starts).

**Cons:** You are now responsible for VPC configuration, load balancer setup, auto-scaling policies, container health checks, log aggregation, certificate management, and deployment pipelines. This is a full-time job for at least one person.

#### Option 4: CloudFront + S3 (Static Only)

If your application is fully static (no SSR, no API routes), you can skip all the compute and just serve files from S3 through CloudFront:

```typescript
// infra/static-site.ts (AWS CDK)
import * as cdk from 'aws-cdk-lib';
import * as s3 from 'aws-cdk-lib/aws-s3';
import * as cloudfront from 'aws-cdk-lib/aws-cloudfront';
import * as s3deploy from 'aws-cdk-lib/aws-s3-deployment';

export class StaticSiteStack extends cdk.Stack {
  constructor(scope: cdk.App, id: string) {
    super(scope, id);

    const bucket = new s3.Bucket(this, 'SiteBucket', {
      blockPublicAccess: s3.BlockPublicAccess.BLOCK_ALL,
      removalPolicy: cdk.RemovalPolicy.DESTROY,
    });

    const distribution = new cloudfront.Distribution(this, 'Distribution', {
      defaultBehavior: {
        origin: new origins.S3BucketOrigin(bucket),
        viewerProtocolPolicy: cloudfront.ViewerProtocolPolicy.REDIRECT_TO_HTTPS,
      },
      defaultRootObject: 'index.html',
      errorResponses: [
        {
          httpStatus: 404,
          responseHttpStatus: 200,
          responsePagePath: '/index.html', // SPA fallback
        },
      ],
    });

    new s3deploy.BucketDeployment(this, 'Deploy', {
      sources: [s3deploy.Source.asset('./out')], // next export output
      destinationBucket: bucket,
      distribution,
      distributionPaths: ['/*'],
    });
  }
}
```

### Self-Hosted Analytics

If you self-host your application, you probably want self-hosted analytics too:

#### PostHog (Full Product Analytics)

```typescript
// PostHog is open-source product analytics
// Self-host with Docker or use their cloud
import posthog from 'posthog-js';

posthog.init('your-project-key', {
  api_host: 'https://analytics.yourdomain.com', // Self-hosted instance
  capture_pageview: true,
  capture_pageleave: true,
});

// Track events
posthog.capture('purchase', { amount: 99, plan: 'pro' });

// Feature flags
if (posthog.isFeatureEnabled('new-checkout')) {
  // Show new checkout
}
```

#### Plausible (Privacy-Friendly Web Analytics)

```typescript
// Plausible is lightweight, privacy-friendly analytics
// Self-host with Docker
// Add to your HTML:
<script
  defer
  data-domain="yourdomain.com"
  src="https://analytics.yourdomain.com/js/script.js"
/>
```

### Self-Hosted Monitoring

| Tool | Purpose | Self-Hosted? |
|------|---------|-------------|
| Grafana + Prometheus | Metrics and dashboards | Yes |
| Loki | Log aggregation | Yes |
| Sentry | Error tracking | Yes (open source) |
| Jaeger | Distributed tracing | Yes |
| Uptime Kuma | Uptime monitoring | Yes |

### Self-Hosted Turborepo Cache

If you use Turborepo in a monorepo, the remote cache can be self-hosted instead of using Vercel's hosted cache:

```bash
# Using turborepo-remote-cache (open source)
# Deploy to your infrastructure
docker run -p 3000:3000 \
  -e STORAGE_PROVIDER=s3 \
  -e S3_ACCESS_KEY=your-key \
  -e S3_SECRET_KEY=your-secret \
  -e S3_REGION=us-east-1 \
  -e S3_BUCKET=turborepo-cache \
  ducktors/turborepo-remote-cache
```

Then configure Turborepo to use your self-hosted cache:

```json
// .turbo/config.json
{
  "teamId": "team_custom",
  "apiUrl": "https://turbo-cache.yourdomain.com"
}
```

### The Honest Comparison

| Dimension | Vercel | Self-Hosted (AWS) |
|-----------|--------|-------------------|
| **Time to deploy** | 5 minutes | Days to weeks |
| **Ongoing maintenance** | Zero | Significant |
| **Cost (small scale)** | $20-100/mo | Higher (minimum AWS costs) |
| **Cost (large scale)** | Can be expensive | Usually cheaper |
| **DX (developer experience)** | Excellent | Depends on your platform team |
| **Preview deployments** | Built-in | Build it yourself or use Amplify |
| **Edge caching** | Automatic | Configure CloudFront yourself |
| **SSL certificates** | Automatic | ACM or Let's Encrypt |
| **Monitoring** | Built-in | Build it yourself |
| **Scaling** | Automatic | Auto-scaling groups, but you configure them |
| **Cold starts** | Minimal (Fluid Compute) | Lambda has them, ECS does not |
| **Vendor lock-in** | Medium (Next.js is portable) | Low (standard AWS services) |
| **Team required** | 0 (platform team) | 1-2 people minimum |
| **Compliance (SOC2, HIPAA)** | Vercel has SOC2 | You control everything |
| **Geographic control** | Limited to Vercel regions | Any AWS region |

### My Recommendation

For most teams:

1. **Start with Vercel.** The speed of iteration is worth more than the cost savings of self-hosting.
2. **Monitor your Vercel bill.** If it exceeds $5,000/month and is growing, evaluate self-hosting.
3. **If you self-host, use SST + OpenNext** unless you have a strong platform engineering team. The ECS approach requires more expertise but gives you more control.
4. **Do not self-host for hypothetical reasons.** "We might need to leave Vercel" is not a reason. "We have a regulatory requirement for a dedicated VPC" is a reason.

The 100x architect does not choose technology based on what is trendy or what is "most pure." They choose based on what lets their team ship the best product with the resources they have.

---

## Summary

Vercel in 2026 is a full compute platform:

- **Fluid Compute** eliminates the cold start problem and makes serverless behave like a real server.
- **Active CPU pricing** means you pay for actual computation, not wall-clock time.
- **Full Node.js Middleware** removes the constraints that made Edge Functions painful.
- **Blob, Edge Config, and Marketplace databases** cover your storage needs.
- **AI Gateway** gives you multi-provider routing, failover, and cost control.
- **Queues** handle durable asynchronous processing.
- **Rolling Releases** give you canary deployments.
- **vercel.ts** gives you typed, programmable configuration.
- **BotID and DDoS protection** secure your application at the edge.
- **Sandbox** lets you run untrusted code safely.

And if none of this is right for your team, OpenNext + SST on AWS is a legitimate alternative. Just be honest about the trade-offs.

The platform is a tool. Use it when it helps. Switch when it does not. That is the engineering mindset.

---

> **Next up:** Chapter 25 dives into Vercel-specific performance optimization patterns, including ISR strategies, streaming SSR, and partial prerendering.
