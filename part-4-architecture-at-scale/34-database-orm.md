<!--
  CHAPTER: 34
  TITLE: Database & ORM Patterns — The Server-Side Data Layer
  PART: IV — Architecture at Scale
  PREREQS: Chapters 11, 27, 28
  KEY_TOPICS: Drizzle ORM, Prisma, Neon Postgres, PlanetScale, Turso, Supabase, database schema design, migrations, relations, type-safe queries, connection pooling, serverless databases, seeding, database branching
  DIFFICULTY: Intermediate → Advanced
  UPDATED: 2026-04-07
-->

# Chapter 34: Database & ORM Patterns — The Server-Side Data Layer

> **Part IV — Architecture at Scale** | Prerequisites: Chapters 11, 27, 28 | Difficulty: Intermediate to Advanced

<details>
<summary><strong>TL;DR</strong></summary>

- When you own the BFF layer with Next.js Server Components and Server Actions, you ARE the backend team -- the database is your responsibility
- Drizzle ORM is the recommended choice for new projects: SQL-like syntax, zero runtime overhead, perfect TypeScript inference, and a tiny bundle
- Prisma remains strong for larger teams that value schema-first design, powerful migration tooling, and a mature ecosystem
- Neon Postgres is the default serverless database: branching for preview environments, autoscaling to zero, built-in connection pooling, and first-class Vercel integration
- Turso (libSQL/SQLite) is the answer for edge-native, read-heavy workloads where single-digit-millisecond latency matters
- Supabase is a full platform play: Postgres + Auth + Realtime + Storage that can replace your entire backend for certain product shapes
- Connection pooling is not optional in serverless -- every cold start opens a new connection, and Postgres defaults max out at 100
- Database migrations are a production safety exercise: never drop columns, always add nullable first, backfill, then constrain

</details>

Let me tell you a story about the moment I realized frontend architects need to understand databases.

It was 2023. Our team had just shipped a Next.js app with the App Router. Server Components were querying our API, which called another API, which called the database. Three network hops for a list of blog posts. The p95 latency was 800ms. A junior engineer asked, "Why don't we just... query the database from the Server Component?" The room went quiet.

That question -- and the architecture we built after answering it -- cut our latency to 40ms and eliminated an entire service from our infrastructure. But it also meant that I, the "frontend architect," was now responsible for schema design, migrations, connection pooling, and query optimization. The boundary between frontend and backend didn't blur. It evaporated.

If you're building with Next.js App Router, Remix, or any framework that gives you a server-side execution context, this chapter is not optional. You don't get to say "that's a backend concern" when your Server Components are executing `SELECT * FROM posts WHERE published = true` on every page render.

### In This Chapter
- Why the frontend architect needs database knowledge
- Choosing a database (Postgres, MySQL, SQLite) for your use case
- Drizzle ORM: the recommended ORM for TypeScript-first teams
- Prisma: when schema-first and mature tooling matter more
- Schema design patterns for real applications
- Migrations: the discipline that keeps production alive
- Type-safe queries that catch errors at compile time
- Relations and joins without N+1 disasters
- Connection pooling for serverless environments
- Neon Postgres deep dive: branching, autoscaling, Vercel integration
- Seeding strategies for development and testing
- Server Components + Database: the direct-query pattern
- Supabase as a full platform
- The complete data architecture: everything wired together

### Related Chapters
- [Ch 11: Caching Strategies] -- caching database queries at the framework level
- [Ch 25: Next.js App Router] -- Server Components and Server Actions
- [Ch 27: Developer Experience & Tooling] -- the tooling ecosystem around databases
- [Ch 28: Beast Mode Full-Stack] -- advanced patterns that build on this foundation

---

## 1. WHY THE FRONTEND ARCHITECT NEEDS DATABASE KNOWLEDGE

This is not a section you can skip. If you're thinking "I'll just let the backend team handle it," you need to understand what has changed.

### The BFF Revolution

The Backend for Frontend (BFF) pattern has been around since Sam Newman coined it in 2015. The idea was simple: instead of one generic API serving web, mobile, iOS, and Android, each client gets its own lightweight backend that shapes data exactly how that client needs it.

For years, the BFF was a separate service. A Node.js Express server. A Go service. Something the "backend team" owned.

Then React Server Components happened.

```
┌───────────────────────────────────────────────────────────────────┐
│                   THE OLD WORLD (2019-2022)                       │
│                                                                   │
│  Browser → React SPA → REST/GraphQL API → Backend Service → DB   │
│                                                                   │
│  Frontend team owns: Browser + SPA                               │
│  Backend team owns: API + Service + DB                           │
│                                                                   │
├───────────────────────────────────────────────────────────────────┤
│                   THE NEW WORLD (2023+)                           │
│                                                                   │
│  Browser → Next.js Server Component → DB                         │
│                                                                   │
│  Frontend team owns: Browser + Server Component + DB             │
│  Backend team owns: ...shared services? ML pipelines?            │
│                                                                   │
└───────────────────────────────────────────────────────────────────┘
```

In a Next.js App Router application:

- **Server Components** execute on the server. They can import database clients directly and run queries. No API call. No network hop. Direct database access.
- **Server Actions** are form handlers and mutation endpoints that execute on the server. They write to the database. They are defined in the same codebase as your components.
- **Route Handlers** (`app/api/*/route.ts`) are your escape hatch for when you need a traditional API (mobile clients, webhooks, third-party integrations).

This means:

```typescript
// app/posts/page.tsx -- This is a SERVER COMPONENT
// It runs on the server. It never ships to the browser.
import { db } from '@/db';
import { posts } from '@/db/schema';
import { desc, eq } from 'drizzle-orm';

export default async function PostsPage() {
  // This query runs on the server at request time.
  // No API. No fetch. No network hop. Just a database query.
  const publishedPosts = await db
    .select()
    .from(posts)
    .where(eq(posts.published, true))
    .orderBy(desc(posts.createdAt));

  return (
    <div>
      {publishedPosts.map(post => (
        <article key={post.id}>
          <h2>{post.title}</h2>
          <p>{post.excerpt}</p>
        </article>
      ))}
    </div>
  );
}
```

**You cannot hand this off to "the backend team" when you ARE the backend.**

### What You Need to Know (And What You Don't)

You do not need to become a DBA. You do not need to tune `shared_buffers` or optimize vacuum schedules. Here's the split:

**You need to know:**
- Schema design (tables, columns, types, relations, indexes)
- ORM usage (Drizzle or Prisma -- how to query, insert, update, delete)
- Migrations (how to evolve your schema safely in production)
- Connection pooling (why it matters in serverless, how to set it up)
- Basic query optimization (select only what you need, avoid N+1, add indexes for slow queries)
- Seeding (how to populate development and test databases)

**You can defer to specialists:**
- Database server tuning (memory, CPU, disk I/O)
- Replication topology (read replicas, failover)
- Backup and disaster recovery strategies
- Advanced query plans and optimizer hints
- Sharding and partitioning at massive scale

The first list is this chapter. The second list is your DevOps/Platform team -- or managed services like Neon that handle it for you.

---

## 2. CHOOSING A DATABASE

This is the first decision, and it's less complicated than the internet makes it sound. Here is the truth: **Postgres is the right default for most applications.** Start there. Deviate only when you have a specific, measurable reason.

### The Three Contenders

#### PostgreSQL — The Default

Postgres is the most capable open-source relational database. It supports JSON columns, full-text search, arrays, enums, generated columns, row-level security, and roughly every feature you'll ever need. The ecosystem is enormous. Every ORM supports it. Every cloud provider offers it.

**Managed Postgres providers for serverless:**

| Provider | Key Feature | Serverless | Branching | Free Tier | Vercel Integration |
|----------|-------------|------------|-----------|-----------|-------------------|
| **Neon** | Autoscale to zero, branching | Yes (native) | Yes | 0.5 GB | First-class |
| **Supabase** | Full platform (Auth + Realtime + Storage) | Yes | Coming | 500 MB | Marketplace |
| **Railway** | Simple deployment, good DX | Yes | No | $5/mo credits | Community |
| **Vercel Postgres** | Powered by Neon under the hood | Yes | Via Neon | 256 MB | Native |

**Choose Postgres when:** You're building any application. This is the default. You need a specific reason NOT to use it.

#### MySQL (PlanetScale) — Horizontal Scaling

PlanetScale brought MySQL into the serverless era with Vitess-powered horizontal scaling and a unique "schema-less migrations" approach where you branch your schema like Git branches, make changes, and merge them. No migration files. No locking DDL.

**Choose MySQL/PlanetScale when:** You need horizontal sharding at massive scale, your team has MySQL expertise, or you're working with an existing MySQL database.

> **Note:** PlanetScale removed their free tier in 2024. Evaluate the pricing against Neon's generous free tier before committing.

#### SQLite (Turso/libSQL) — Edge-Native

Turso runs libSQL (a fork of SQLite) at the edge. Your database is replicated to data centers near your users. Reads are local -- single-digit millisecond latency. Writes go to a primary and replicate out.

```
┌─────────────────────────────────────────────────────────────────┐
│                    TURSO ARCHITECTURE                            │
│                                                                  │
│   User (Tokyo) ──→ Edge Replica (Tokyo) ──→ 1ms read            │
│   User (London) ──→ Edge Replica (London) ──→ 1ms read          │
│   User (NYC) ──→ Edge Replica (Virginia) ──→ 2ms read           │
│                                                                  │
│   Write from any location ──→ Primary (your chosen region)      │
│                            ──→ Replicates to all edges           │
│                                                                  │
│   Best for: Read-heavy workloads, content sites, config stores  │
│   Not ideal: Write-heavy, strong consistency requirements       │
└─────────────────────────────────────────────────────────────────┘
```

Turso also supports **embedded replicas** -- a SQLite file embedded in your application that syncs with the cloud database. This means zero-latency reads even on the first request.

**Choose Turso when:** Read-heavy workloads, edge-first architecture, content-heavy sites, or when you need embedded database capabilities (Electron apps, React Native with local-first sync).

### The Decision Matrix

```
┌──────────────────┬────────────┬──────────────┬──────────────────┐
│                  │  POSTGRES  │    MySQL     │  SQLite (Turso)  │
│                  │  (Neon)    │ (PlanetScale)│                  │
├──────────────────┼────────────┼──────────────┼──────────────────┤
│ Default choice   │     ✓      │              │                  │
│ JSON support     │  Excellent │    Good      │    Good          │
│ Full-text search │  Built-in  │  Built-in    │    FTS5          │
│ Edge latency     │  ~30-50ms  │   ~30-50ms   │    ~1-5ms        │
│ Scale to zero    │     ✓      │      ✓       │       ✓          │
│ Branching        │     ✓      │      ✓       │       ✓          │
│ Free tier        │  Generous  │    None      │    Generous      │
│ Drizzle support  │     ✓      │      ✓       │       ✓          │
│ Prisma support   │     ✓      │      ✓       │       ✓          │
│ Max connections  │ Pool-based │    ∞ (HTTP)  │  ∞ (HTTP/embed)  │
│ Complexity       │    Low     │   Medium     │      Low         │
└──────────────────┴────────────┴──────────────┴──────────────────┘
```

**The 100x recommendation:** Start with Neon Postgres. If query latency at the edge becomes a bottleneck, add Turso for read-heavy paths. Don't split your data across databases unless you have evidence that you need to.

---

## 3. DRIZZLE ORM — THE RECOMMENDED ORM

Drizzle has emerged as the ORM of choice for TypeScript-first teams. Here's why, and how to use it end to end.

### Why Drizzle Is Winning

1. **SQL-like syntax.** If you know SQL, you know Drizzle. There is no "Drizzle query language" to learn. The API mirrors SQL.
2. **Zero runtime overhead.** Drizzle generates SQL strings. There is no query engine, no WASM runtime, no heavyweight client. It's a thin TypeScript layer over your database driver.
3. **Perfect TypeScript inference.** Select a subset of columns? TypeScript knows the return type. Join two tables? TypeScript infers the combined type. No codegen step. No manual type definitions.
4. **Tiny bundle.** Drizzle core is ~50KB. Compare to Prisma Client which can be several MB.
5. **Relational queries.** Drizzle's relational query API gives you Prisma-like `include` syntax when you want it, without sacrificing the SQL-like API when you need control.

### Installation

```bash
# Core ORM + Postgres driver + migration toolkit
pnpm add drizzle-orm @neondatabase/serverless
pnpm add -D drizzle-kit
```

For other databases:

```bash
# MySQL
pnpm add drizzle-orm mysql2
pnpm add -D drizzle-kit

# SQLite (Turso)
pnpm add drizzle-orm @libsql/client
pnpm add -D drizzle-kit

# SQLite (better-sqlite3, for local dev)
pnpm add drizzle-orm better-sqlite3
pnpm add -D drizzle-kit @types/better-sqlite3
```

### Drizzle Config

Create `drizzle.config.ts` at your project root:

```typescript
// drizzle.config.ts
import { defineConfig } from 'drizzle-kit';

export default defineConfig({
  dialect: 'postgresql',
  schema: './src/db/schema/index.ts',
  out: './drizzle',
  dbCredentials: {
    url: process.env.DATABASE_URL!,
  },
  // Enable verbose logging during development
  verbose: true,
  // Strict mode warns about destructive changes
  strict: true,
});
```

### Schema Definition

This is where Drizzle shines. Your schema IS your TypeScript code. No separate schema file. No codegen. Your types come directly from your table definitions.

```typescript
// src/db/schema/users.ts
import {
  pgTable,
  uuid,
  varchar,
  text,
  timestamp,
  boolean,
  pgEnum,
  index,
  uniqueIndex,
} from 'drizzle-orm/pg-core';

// Enums are first-class
export const userRoleEnum = pgEnum('user_role', [
  'owner',
  'admin',
  'member',
  'viewer',
]);

export const users = pgTable(
  'users',
  {
    id: uuid('id').primaryKey().defaultRandom(),
    email: varchar('email', { length: 255 }).notNull(),
    name: varchar('name', { length: 255 }),
    avatarUrl: text('avatar_url'),
    role: userRoleEnum('role').default('member').notNull(),
    emailVerified: boolean('email_verified').default(false).notNull(),
    createdAt: timestamp('created_at', { withTimezone: true })
      .defaultNow()
      .notNull(),
    updatedAt: timestamp('updated_at', { withTimezone: true })
      .defaultNow()
      .notNull()
      .$onUpdate(() => new Date()),
    deletedAt: timestamp('deleted_at', { withTimezone: true }),
  },
  (table) => [
    uniqueIndex('users_email_idx').on(table.email),
    index('users_created_at_idx').on(table.createdAt),
  ]
);

// TypeScript types are inferred AUTOMATICALLY
// No codegen. No manual type definitions.
export type User = typeof users.$inferSelect;
export type NewUser = typeof users.$inferInsert;
```

```typescript
// src/db/schema/organizations.ts
import { pgTable, uuid, varchar, text, timestamp, index } from 'drizzle-orm/pg-core';

export const organizations = pgTable(
  'organizations',
  {
    id: uuid('id').primaryKey().defaultRandom(),
    name: varchar('name', { length: 255 }).notNull(),
    slug: varchar('slug', { length: 100 }).notNull(),
    logoUrl: text('logo_url'),
    createdAt: timestamp('created_at', { withTimezone: true })
      .defaultNow()
      .notNull(),
    updatedAt: timestamp('updated_at', { withTimezone: true })
      .defaultNow()
      .notNull()
      .$onUpdate(() => new Date()),
  },
  (table) => [
    index('organizations_slug_idx').on(table.slug),
  ]
);

export type Organization = typeof organizations.$inferSelect;
export type NewOrganization = typeof organizations.$inferInsert;
```

```typescript
// src/db/schema/memberships.ts
import { pgTable, uuid, timestamp, pgEnum, primaryKey } from 'drizzle-orm/pg-core';
import { users } from './users';
import { organizations } from './organizations';

export const membershipRoleEnum = pgEnum('membership_role', [
  'owner',
  'admin',
  'member',
  'viewer',
]);

export const memberships = pgTable(
  'memberships',
  {
    userId: uuid('user_id')
      .notNull()
      .references(() => users.id, { onDelete: 'cascade' }),
    organizationId: uuid('organization_id')
      .notNull()
      .references(() => organizations.id, { onDelete: 'cascade' }),
    role: membershipRoleEnum('role').default('member').notNull(),
    joinedAt: timestamp('joined_at', { withTimezone: true })
      .defaultNow()
      .notNull(),
  },
  (table) => [
    primaryKey({ columns: [table.userId, table.organizationId] }),
  ]
);

export type Membership = typeof memberships.$inferSelect;
export type NewMembership = typeof memberships.$inferInsert;
```

```typescript
// src/db/schema/posts.ts
import {
  pgTable,
  uuid,
  varchar,
  text,
  timestamp,
  boolean,
  jsonb,
  index,
} from 'drizzle-orm/pg-core';
import { users } from './users';
import { organizations } from './organizations';

export const posts = pgTable(
  'posts',
  {
    id: uuid('id').primaryKey().defaultRandom(),
    title: varchar('title', { length: 500 }).notNull(),
    slug: varchar('slug', { length: 500 }).notNull(),
    content: text('content'),
    excerpt: text('excerpt'),
    // JSON columns for flexible metadata
    metadata: jsonb('metadata').$type<{
      readingTime?: number;
      coverImage?: string;
      tags?: string[];
      seoTitle?: string;
      seoDescription?: string;
    }>(),
    published: boolean('published').default(false).notNull(),
    publishedAt: timestamp('published_at', { withTimezone: true }),
    authorId: uuid('author_id')
      .notNull()
      .references(() => users.id),
    organizationId: uuid('organization_id')
      .notNull()
      .references(() => organizations.id, { onDelete: 'cascade' }),
    createdAt: timestamp('created_at', { withTimezone: true })
      .defaultNow()
      .notNull(),
    updatedAt: timestamp('updated_at', { withTimezone: true })
      .defaultNow()
      .notNull()
      .$onUpdate(() => new Date()),
    deletedAt: timestamp('deleted_at', { withTimezone: true }),
  },
  (table) => [
    index('posts_author_id_idx').on(table.authorId),
    index('posts_organization_id_idx').on(table.organizationId),
    index('posts_published_idx').on(table.published),
    index('posts_slug_org_idx').on(table.slug, table.organizationId),
  ]
);

export type Post = typeof posts.$inferSelect;
export type NewPost = typeof posts.$inferInsert;
```

```typescript
// src/db/schema/audit-logs.ts
import { pgTable, uuid, varchar, text, timestamp, jsonb, index } from 'drizzle-orm/pg-core';
import { users } from './users';
import { organizations } from './organizations';

export const auditLogs = pgTable(
  'audit_logs',
  {
    id: uuid('id').primaryKey().defaultRandom(),
    action: varchar('action', { length: 100 }).notNull(), // e.g., 'post.created', 'user.invited'
    entityType: varchar('entity_type', { length: 100 }).notNull(), // e.g., 'post', 'user'
    entityId: uuid('entity_id').notNull(),
    actorId: uuid('actor_id').references(() => users.id),
    organizationId: uuid('organization_id')
      .notNull()
      .references(() => organizations.id, { onDelete: 'cascade' }),
    metadata: jsonb('metadata').$type<Record<string, unknown>>(),
    ipAddress: varchar('ip_address', { length: 45 }),
    userAgent: text('user_agent'),
    createdAt: timestamp('created_at', { withTimezone: true })
      .defaultNow()
      .notNull(),
  },
  (table) => [
    index('audit_logs_org_idx').on(table.organizationId),
    index('audit_logs_actor_idx').on(table.actorId),
    index('audit_logs_entity_idx').on(table.entityType, table.entityId),
    index('audit_logs_created_at_idx').on(table.createdAt),
  ]
);

export type AuditLog = typeof auditLogs.$inferSelect;
export type NewAuditLog = typeof auditLogs.$inferInsert;
```

```typescript
// src/db/schema/index.ts — Barrel export
export * from './users';
export * from './organizations';
export * from './memberships';
export * from './posts';
export * from './audit-logs';
export * from './relations';
```

### Defining Relations

Drizzle has two query APIs:

1. **SQL-like API** (`db.select().from(...).where(...)`) -- explicit joins, full control
2. **Relational API** (`db.query.users.findMany({ with: { posts: true } })`) -- Prisma-like, convenient

The relational API requires you to define relations:

```typescript
// src/db/schema/relations.ts
import { relations } from 'drizzle-orm';
import { users } from './users';
import { organizations } from './organizations';
import { memberships } from './memberships';
import { posts } from './posts';
import { auditLogs } from './audit-logs';

export const usersRelations = relations(users, ({ many }) => ({
  memberships: many(memberships),
  posts: many(posts),
  auditLogs: many(auditLogs),
}));

export const organizationsRelations = relations(organizations, ({ many }) => ({
  memberships: many(memberships),
  posts: many(posts),
  auditLogs: many(auditLogs),
}));

export const membershipsRelations = relations(memberships, ({ one }) => ({
  user: one(users, {
    fields: [memberships.userId],
    references: [users.id],
  }),
  organization: one(organizations, {
    fields: [memberships.organizationId],
    references: [organizations.id],
  }),
}));

export const postsRelations = relations(posts, ({ one }) => ({
  author: one(users, {
    fields: [posts.authorId],
    references: [users.id],
  }),
  organization: one(organizations, {
    fields: [posts.organizationId],
    references: [organizations.id],
  }),
}));

export const auditLogsRelations = relations(auditLogs, ({ one }) => ({
  actor: one(users, {
    fields: [auditLogs.actorId],
    references: [users.id],
  }),
  organization: one(organizations, {
    fields: [auditLogs.organizationId],
    references: [organizations.id],
  }),
}));
```

### Database Client Setup

```typescript
// src/db/index.ts
import { drizzle } from 'drizzle-orm/neon-http';
import { neon } from '@neondatabase/serverless';
import * as schema from './schema';

// Create the Neon SQL client
const sql = neon(process.env.DATABASE_URL!);

// Create the Drizzle client with schema for relational queries
export const db = drizzle(sql, { schema });

// Export the type for use in other modules
export type Database = typeof db;
```

For environments that need WebSocket connections (transactions, interactive queries):

```typescript
// src/db/index.ts — WebSocket variant for transactions
import { drizzle } from 'drizzle-orm/neon-serverless';
import { Pool } from '@neondatabase/serverless';
import * as schema from './schema';

// Use Pool for connection pooling in serverless
const pool = new Pool({ connectionString: process.env.DATABASE_URL! });

export const db = drizzle(pool, { schema });
export type Database = typeof db;
```

### CRUD Operations with Drizzle

#### SELECT — Reading Data

```typescript
import { db } from '@/db';
import { posts, users } from '@/db/schema';
import { eq, and, desc, gt, like, count, sql, ilike, isNull } from 'drizzle-orm';

// ── Basic select ──────────────────────────────────────────────
const allPosts = await db.select().from(posts);

// ── Select specific columns ──────────────────────────────────
// TypeScript knows the return type is { id: string; title: string }[]
const postTitles = await db
  .select({
    id: posts.id,
    title: posts.title,
  })
  .from(posts);

// ── WHERE clause ─────────────────────────────────────────────
const publishedPosts = await db
  .select()
  .from(posts)
  .where(eq(posts.published, true));

// ── Multiple conditions (AND) ────────────────────────────────
const recentPublished = await db
  .select()
  .from(posts)
  .where(
    and(
      eq(posts.published, true),
      gt(posts.publishedAt, new Date('2026-01-01')),
      isNull(posts.deletedAt)
    )
  );

// ── ORDER BY + LIMIT + OFFSET ────────────────────────────────
const paginatedPosts = await db
  .select()
  .from(posts)
  .where(eq(posts.published, true))
  .orderBy(desc(posts.publishedAt))
  .limit(20)
  .offset(40); // page 3

// ── LIKE / ILIKE search ──────────────────────────────────────
const searchResults = await db
  .select()
  .from(posts)
  .where(ilike(posts.title, `%${searchTerm}%`));

// ── COUNT ────────────────────────────────────────────────────
const [{ total }] = await db
  .select({ total: count() })
  .from(posts)
  .where(eq(posts.published, true));

// ── Aggregation with GROUP BY ────────────────────────────────
const postsByAuthor = await db
  .select({
    authorId: posts.authorId,
    postCount: count(),
  })
  .from(posts)
  .groupBy(posts.authorId);
```

#### JOIN — Combining Tables

```typescript
// ── INNER JOIN ───────────────────────────────────────────────
const postsWithAuthors = await db
  .select({
    postId: posts.id,
    postTitle: posts.title,
    authorName: users.name,
    authorEmail: users.email,
  })
  .from(posts)
  .innerJoin(users, eq(posts.authorId, users.id))
  .where(eq(posts.published, true));

// ── LEFT JOIN ────────────────────────────────────────────────
// Users and their post count (includes users with 0 posts)
const usersWithPostCount = await db
  .select({
    userId: users.id,
    userName: users.name,
    postCount: count(posts.id),
  })
  .from(users)
  .leftJoin(posts, eq(users.id, posts.authorId))
  .groupBy(users.id, users.name);

// ── Multiple JOINs ──────────────────────────────────────────
import { memberships, organizations } from '@/db/schema';

const orgMembers = await db
  .select({
    userName: users.name,
    userEmail: users.email,
    orgName: organizations.name,
    role: memberships.role,
    joinedAt: memberships.joinedAt,
  })
  .from(memberships)
  .innerJoin(users, eq(memberships.userId, users.id))
  .innerJoin(organizations, eq(memberships.organizationId, organizations.id))
  .where(eq(organizations.slug, 'acme-corp'));
```

#### INSERT — Creating Data

```typescript
// ── Single insert ────────────────────────────────────────────
const [newUser] = await db
  .insert(users)
  .values({
    email: 'alice@example.com',
    name: 'Alice Johnson',
    role: 'member',
  })
  .returning(); // Returns the full inserted row

// ── Bulk insert ──────────────────────────────────────────────
const newPosts = await db
  .insert(posts)
  .values([
    {
      title: 'First Post',
      slug: 'first-post',
      content: 'Hello world',
      authorId: newUser.id,
      organizationId: orgId,
    },
    {
      title: 'Second Post',
      slug: 'second-post',
      content: 'Another post',
      authorId: newUser.id,
      organizationId: orgId,
    },
  ])
  .returning();

// ── Insert with conflict handling (upsert) ───────────────────
const upsertedUser = await db
  .insert(users)
  .values({
    email: 'alice@example.com',
    name: 'Alice Updated',
  })
  .onConflictDoUpdate({
    target: users.email,
    set: {
      name: sql`excluded.name`,
      updatedAt: new Date(),
    },
  })
  .returning();
```

#### UPDATE — Modifying Data

```typescript
// ── Simple update ────────────────────────────────────────────
const [updatedPost] = await db
  .update(posts)
  .set({
    title: 'Updated Title',
    updatedAt: new Date(),
  })
  .where(eq(posts.id, postId))
  .returning();

// ── Conditional update ───────────────────────────────────────
// Publish all draft posts for an author
const publishedPosts = await db
  .update(posts)
  .set({
    published: true,
    publishedAt: new Date(),
  })
  .where(
    and(
      eq(posts.authorId, authorId),
      eq(posts.published, false)
    )
  )
  .returning();

// ── Soft delete ──────────────────────────────────────────────
await db
  .update(posts)
  .set({ deletedAt: new Date() })
  .where(eq(posts.id, postId));
```

#### DELETE — Removing Data

```typescript
// ── Hard delete ──────────────────────────────────────────────
await db.delete(posts).where(eq(posts.id, postId));

// ── Delete with returning ────────────────────────────────────
const [deletedPost] = await db
  .delete(posts)
  .where(eq(posts.id, postId))
  .returning();
```

### Relational Queries (The Prisma-Like API)

When you've defined relations, Drizzle gives you a powerful relational query API:

```typescript
// ── Find many with relations ─────────────────────────────────
const postsWithAuthors = await db.query.posts.findMany({
  where: eq(posts.published, true),
  orderBy: [desc(posts.publishedAt)],
  limit: 20,
  with: {
    author: true, // includes the full user object
  },
});
// Type: { id: string; title: string; ...; author: User }[]

// ── Nested relations ─────────────────────────────────────────
const orgsWithMembers = await db.query.organizations.findMany({
  with: {
    memberships: {
      with: {
        user: true,
      },
    },
  },
});
// Type: { id: string; name: string; memberships: { user: User; role: string }[] }[]

// ── Select specific columns from relations ───────────────────
const postsLite = await db.query.posts.findMany({
  columns: {
    id: true,
    title: true,
    slug: true,
    publishedAt: true,
  },
  with: {
    author: {
      columns: {
        name: true,
        avatarUrl: true,
      },
    },
  },
  where: eq(posts.published, true),
  limit: 10,
});
// Type: { id: string; title: string; slug: string; publishedAt: Date | null;
//         author: { name: string | null; avatarUrl: string | null } }[]

// ── Find one ─────────────────────────────────────────────────
const post = await db.query.posts.findFirst({
  where: and(
    eq(posts.slug, 'my-post'),
    eq(posts.organizationId, orgId)
  ),
  with: {
    author: {
      columns: { name: true, avatarUrl: true },
    },
  },
});
```

### Transactions

```typescript
// ── Basic transaction ────────────────────────────────────────
const result = await db.transaction(async (tx) => {
  // Create the organization
  const [org] = await tx
    .insert(organizations)
    .values({
      name: 'Acme Corp',
      slug: 'acme-corp',
    })
    .returning();

  // Add the creator as owner
  await tx.insert(memberships).values({
    userId: currentUserId,
    organizationId: org.id,
    role: 'owner',
  });

  // Create an audit log entry
  await tx.insert(auditLogs).values({
    action: 'organization.created',
    entityType: 'organization',
    entityId: org.id,
    actorId: currentUserId,
    organizationId: org.id,
  });

  return org;
});

// ── Transaction with rollback ────────────────────────────────
try {
  await db.transaction(async (tx) => {
    await tx.insert(posts).values({ /* ... */ });

    // If this fails, the entire transaction rolls back
    await tx.insert(auditLogs).values({ /* ... */ });

    // You can also manually roll back
    if (someCondition) {
      tx.rollback();
    }
  });
} catch (error) {
  // Transaction was rolled back
  console.error('Transaction failed:', error);
}
```

### Subqueries and Raw SQL

```typescript
// ── Subquery ─────────────────────────────────────────────────
import { sql, eq, inArray } from 'drizzle-orm';

// Posts by users who are admins
const adminPosts = await db
  .select()
  .from(posts)
  .where(
    inArray(
      posts.authorId,
      db
        .select({ id: users.id })
        .from(users)
        .where(eq(users.role, 'admin'))
    )
  );

// ── Raw SQL escape hatch ─────────────────────────────────────
// When the query builder doesn't support what you need
const result = await db.execute(sql`
  SELECT
    p.id,
    p.title,
    u.name as author_name,
    ts_rank(to_tsvector('english', p.content), plainto_tsquery('english', ${searchTerm})) as rank
  FROM posts p
  JOIN users u ON p.author_id = u.id
  WHERE to_tsvector('english', p.content) @@ plainto_tsquery('english', ${searchTerm})
  ORDER BY rank DESC
  LIMIT 20
`);

// ── Raw SQL in select ────────────────────────────────────────
const postsWithWordCount = await db
  .select({
    id: posts.id,
    title: posts.title,
    wordCount: sql<number>`array_length(string_to_array(${posts.content}, ' '), 1)`,
  })
  .from(posts);
```

### Prepared Statements

For queries that run frequently, prepared statements skip the query planning step on subsequent executions:

```typescript
// ── Prepare a parameterized query ────────────────────────────
import { placeholder } from 'drizzle-orm';

const getPostBySlug = db
  .select()
  .from(posts)
  .where(
    and(
      eq(posts.slug, placeholder('slug')),
      eq(posts.organizationId, placeholder('orgId'))
    )
  )
  .prepare('get_post_by_slug');

// Execute with different parameters -- reuses the prepared plan
const post1 = await getPostBySlug.execute({ slug: 'hello-world', orgId: 'abc-123' });
const post2 = await getPostBySlug.execute({ slug: 'second-post', orgId: 'abc-123' });
```

### Drizzle Studio

Drizzle Kit includes a built-in data browser:

```bash
# Launch Drizzle Studio -- opens in your browser
pnpm drizzle-kit studio
```

This gives you a visual interface to browse tables, view data, run queries, and edit rows directly. It's like pgAdmin or TablePlus, but built into your toolchain and aware of your schema.

---

## 4. PRISMA — WHEN SCHEMA-FIRST WINS

Drizzle is my recommendation, but Prisma is not wrong. Here is when Prisma is the better choice, and how to use it.

### When to Choose Prisma Over Drizzle

- **Larger teams (10+ engineers):** Prisma's schema file is a single source of truth that non-TypeScript developers can read. The `.prisma` schema language is simpler than TypeScript table definitions.
- **Schema-first thinking:** If you want to design your schema in a dedicated DSL and generate everything from it (types, client, migrations), Prisma's workflow is more natural.
- **Stronger migration tooling:** `prisma migrate` generates migration files with full SQL, tracks migration history in the database, handles migration conflicts, and supports baseline migrations for existing databases.
- **Prisma Studio:** A polished GUI for browsing and editing data, more mature than Drizzle Studio.
- **Ecosystem integrations:** Prisma has mature integrations with NextAuth.js, tRPC adapters, Zod schema generators (`zod-prisma-types`), and more.

### The Schema

```prisma
// prisma/schema.prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
  // For Neon serverless, use the pooled connection string
  directUrl = env("DIRECT_DATABASE_URL")
}

enum UserRole {
  OWNER
  ADMIN
  MEMBER
  VIEWER
}

enum MembershipRole {
  OWNER
  ADMIN
  MEMBER
  VIEWER
}

model User {
  id            String    @id @default(uuid())
  email         String    @unique
  name          String?
  avatarUrl     String?   @map("avatar_url")
  role          UserRole  @default(MEMBER)
  emailVerified Boolean   @default(false) @map("email_verified")
  createdAt     DateTime  @default(now()) @map("created_at")
  updatedAt     DateTime  @updatedAt @map("updated_at")
  deletedAt     DateTime? @map("deleted_at")

  memberships Membership[]
  posts       Post[]
  auditLogs   AuditLog[]

  @@index([createdAt])
  @@map("users")
}

model Organization {
  id        String   @id @default(uuid())
  name      String
  slug      String
  logoUrl   String?  @map("logo_url")
  createdAt DateTime @default(now()) @map("created_at")
  updatedAt DateTime @updatedAt @map("updated_at")

  memberships Membership[]
  posts       Post[]
  auditLogs   AuditLog[]

  @@index([slug])
  @@map("organizations")
}

model Membership {
  userId         String         @map("user_id")
  organizationId String         @map("organization_id")
  role           MembershipRole @default(MEMBER)
  joinedAt       DateTime       @default(now()) @map("joined_at")

  user         User         @relation(fields: [userId], references: [id], onDelete: Cascade)
  organization Organization @relation(fields: [organizationId], references: [id], onDelete: Cascade)

  @@id([userId, organizationId])
  @@map("memberships")
}

model Post {
  id             String    @id @default(uuid())
  title          String
  slug           String
  content        String?
  excerpt        String?
  metadata       Json?
  published      Boolean   @default(false)
  publishedAt    DateTime? @map("published_at")
  authorId       String    @map("author_id")
  organizationId String    @map("organization_id")
  createdAt      DateTime  @default(now()) @map("created_at")
  updatedAt      DateTime  @updatedAt @map("updated_at")
  deletedAt      DateTime? @map("deleted_at")

  author       User         @relation(fields: [authorId], references: [id])
  organization Organization @relation(fields: [organizationId], references: [id], onDelete: Cascade)

  @@index([authorId])
  @@index([organizationId])
  @@index([published])
  @@index([slug, organizationId])
  @@map("posts")
}

model AuditLog {
  id             String   @id @default(uuid())
  action         String
  entityType     String   @map("entity_type")
  entityId       String   @map("entity_id")
  actorId        String?  @map("actor_id")
  organizationId String   @map("organization_id")
  metadata       Json?
  ipAddress      String?  @map("ip_address")
  userAgent      String?  @map("user_agent")
  createdAt      DateTime @default(now()) @map("created_at")

  actor        User?        @relation(fields: [actorId], references: [id])
  organization Organization @relation(fields: [organizationId], references: [id], onDelete: Cascade)

  @@index([organizationId])
  @@index([actorId])
  @@index([entityType, entityId])
  @@index([createdAt])
  @@map("audit_logs")
}
```

### Prisma Client Setup

```bash
# Install
pnpm add prisma @prisma/client
pnpm add -D prisma

# Generate the client (run after every schema change)
pnpm prisma generate

# Push schema to database (development -- no migration file)
pnpm prisma db push

# Create a migration (production-safe)
pnpm prisma migrate dev --name init

# Apply migrations in production
pnpm prisma migrate deploy
```

```typescript
// src/db/prisma.ts
import { PrismaClient } from '@prisma/client';

const globalForPrisma = globalThis as unknown as {
  prisma: PrismaClient | undefined;
};

// Prevent multiple instances in development (Next.js hot reload)
export const prisma = globalForPrisma.prisma ?? new PrismaClient({
  log: process.env.NODE_ENV === 'development'
    ? ['query', 'error', 'warn']
    : ['error'],
});

if (process.env.NODE_ENV !== 'production') {
  globalForPrisma.prisma = prisma;
}
```

### Prisma Queries

```typescript
import { prisma } from '@/db/prisma';

// ── Find many with relations ─────────────────────────────────
const postsWithAuthors = await prisma.post.findMany({
  where: {
    published: true,
    deletedAt: null,
  },
  include: {
    author: {
      select: { name: true, avatarUrl: true },
    },
  },
  orderBy: { publishedAt: 'desc' },
  take: 20,
  skip: 40, // pagination
});

// ── Find unique ──────────────────────────────────────────────
const user = await prisma.user.findUnique({
  where: { email: 'alice@example.com' },
  include: {
    memberships: {
      include: { organization: true },
    },
  },
});

// ── Create with nested relations ─────────────────────────────
const org = await prisma.organization.create({
  data: {
    name: 'Acme Corp',
    slug: 'acme-corp',
    memberships: {
      create: {
        userId: currentUserId,
        role: 'OWNER',
      },
    },
  },
  include: {
    memberships: { include: { user: true } },
  },
});

// ── Upsert ───────────────────────────────────────────────────
const user = await prisma.user.upsert({
  where: { email: 'alice@example.com' },
  update: { name: 'Alice Updated' },
  create: {
    email: 'alice@example.com',
    name: 'Alice',
  },
});

// ── Transaction ──────────────────────────────────────────────
const [post, auditLog] = await prisma.$transaction([
  prisma.post.create({
    data: {
      title: 'New Post',
      slug: 'new-post',
      authorId: userId,
      organizationId: orgId,
    },
  }),
  prisma.auditLog.create({
    data: {
      action: 'post.created',
      entityType: 'post',
      entityId: 'will-be-set', // See interactive transaction below
      actorId: userId,
      organizationId: orgId,
    },
  }),
]);

// ── Interactive transaction (when you need the result of one query in another)
const result = await prisma.$transaction(async (tx) => {
  const post = await tx.post.create({
    data: {
      title: 'New Post',
      slug: 'new-post',
      authorId: userId,
      organizationId: orgId,
    },
  });

  await tx.auditLog.create({
    data: {
      action: 'post.created',
      entityType: 'post',
      entityId: post.id, // Now we can reference the created post
      actorId: userId,
      organizationId: orgId,
    },
  });

  return post;
});
```

### Generating Zod Schemas from Prisma

One of Prisma's ecosystem strengths is automatic Zod schema generation:

```bash
pnpm add -D zod-prisma-types
```

Add to your Prisma schema:

```prisma
generator zod {
  provider                         = "zod-prisma-types"
  output                           = "../src/db/zod"
  createModelTypes                 = true
  createInputTypes                 = true
  addInputTypeValidation           = true
}
```

```bash
pnpm prisma generate
# Now you have Zod schemas in src/db/zod/
```

```typescript
import { PostCreateInputSchema, PostUpdateInputSchema } from '@/db/zod';

// Use in Server Actions for validation
export async function createPost(formData: FormData) {
  'use server';
  const data = PostCreateInputSchema.parse({
    title: formData.get('title'),
    slug: formData.get('slug'),
    content: formData.get('content'),
  });
  // data is fully validated and typed
}
```

### Drizzle vs Prisma: The Honest Comparison

```
┌───────────────────────────┬──────────────┬──────────────┐
│                           │   DRIZZLE    │    PRISMA    │
├───────────────────────────┼──────────────┼──────────────┤
│ Schema definition         │  TypeScript  │  .prisma DSL │
│ Type inference            │  Automatic   │  Generated   │
│ Codegen required          │  No          │  Yes         │
│ Bundle size               │  ~50KB       │  ~2-4MB      │
│ Cold start impact         │  Minimal     │  Noticeable  │
│ SQL familiarity           │  High        │  Low         │
│ Learning curve            │  Low (SQL)   │  Medium      │
│ Migration tooling         │  Good        │  Excellent   │
│ Data browser              │  Good        │  Excellent   │
│ Serverless optimized      │  Yes         │  Yes*        │
│ Edge runtime              │  Yes         │  Partial     │
│ Raw SQL escape hatch      │  Elegant     │  Available   │
│ Ecosystem maturity        │  Growing     │  Mature      │
│ Team onboarding           │  Fast        │  Fast        │
│ Complex queries           │  Natural     │  Can be hard │
│ N+1 prevention            │  Manual      │  Automatic   │
└───────────────────────────┴──────────────┴──────────────┘

* Prisma Accelerate or Data Proxy needed for optimal serverless perf
```

**The verdict:** Use Drizzle for new projects, especially if your team knows SQL. Use Prisma if you have an existing Prisma codebase, a large team that benefits from the schema DSL, or you need the ecosystem integrations (NextAuth adapter, etc.).

---

## 5. SCHEMA DESIGN PATTERNS

Good schema design prevents 90% of future problems. Bad schema design causes 90% of future rewrites. Here are the patterns that matter for real applications.

### UUID vs Auto-Increment

```typescript
// ── UUID (recommended for most cases) ────────────────────────
id: uuid('id').primaryKey().defaultRandom(),

// ── Auto-increment (when you need ordered IDs) ──────────────
id: serial('id').primaryKey(),

// ── CUID2 (human-friendly, sortable, collision-resistant) ────
import { createId } from '@paralleldrive/cuid2';

id: varchar('id', { length: 128 }).primaryKey().$defaultFn(() => createId()),
```

**When to use UUID:**
- Distributed systems (no coordination needed to generate IDs)
- When IDs are exposed in URLs (harder to enumerate)
- When merging data from multiple sources

**When to use auto-increment:**
- When you need ordered IDs (sorting by creation order)
- When storage size matters (4 bytes vs 16 bytes)
- Internal-only IDs not exposed to users

**The 100x recommendation:** Use UUID with `defaultRandom()` for primary keys. Use CUID2 when you need sortable, URL-friendly IDs (e.g., public slugs, API keys).

### Timestamps Pattern

Every table should have these:

```typescript
// The timestamp trio -- add to EVERY table
createdAt: timestamp('created_at', { withTimezone: true })
  .defaultNow()
  .notNull(),
updatedAt: timestamp('updated_at', { withTimezone: true })
  .defaultNow()
  .notNull()
  .$onUpdate(() => new Date()),
```

For tables that support soft delete:

```typescript
deletedAt: timestamp('deleted_at', { withTimezone: true }),
```

> **Always use `withTimezone: true`.** Storing timestamps without timezone information is a bug waiting to happen when your users span multiple timezones. Postgres stores everything as UTC internally; `timestamptz` just ensures the conversion happens correctly.

### Soft Deletes

Soft deletes keep data recoverable and maintain referential integrity for audit trails:

```typescript
// ── Schema ───────────────────────────────────────────────────
export const posts = pgTable('posts', {
  // ... other columns
  deletedAt: timestamp('deleted_at', { withTimezone: true }),
});

// ── Helper function ──────────────────────────────────────────
export function withSoftDelete<T extends { deletedAt: unknown }>(
  table: T
) {
  return isNull(table.deletedAt as any);
}

// ── Usage ────────────────────────────────────────────────────
// Always filter out soft-deleted records
const activePosts = await db
  .select()
  .from(posts)
  .where(and(
    eq(posts.published, true),
    isNull(posts.deletedAt)
  ));

// Soft delete
await db
  .update(posts)
  .set({ deletedAt: new Date() })
  .where(eq(posts.id, postId));

// Restore
await db
  .update(posts)
  .set({ deletedAt: null })
  .where(eq(posts.id, postId));

// Hard delete (scheduled cleanup job, runs weekly)
await db
  .delete(posts)
  .where(
    and(
      isNotNull(posts.deletedAt),
      lt(posts.deletedAt, new Date(Date.now() - 30 * 24 * 60 * 60 * 1000)) // 30 days ago
    )
  );
```

### The Multi-Tenant Organization Pattern

This is the most common pattern for SaaS applications:

```
┌─────────────────────────────────────────────────────────────────┐
│                   MULTI-TENANT DATA MODEL                       │
│                                                                  │
│   Users ────────< Memberships >──────── Organizations            │
│     │                  │                      │                  │
│     │                  │                      │                  │
│     └── authored ──< Posts >── belongs to ────┘                  │
│     │                                         │                  │
│     └── acted ────< AuditLogs >── belongs to ─┘                  │
│                                                                  │
│   Every data row belongs to an organization.                     │
│   Users access data through their memberships.                   │
│   Roles are per-membership, not per-user.                        │
└─────────────────────────────────────────────────────────────────┘
```

Key principles:
1. **Every data table has an `organizationId` foreign key.** This is your tenant isolation boundary.
2. **User roles are per-organization.** Alice can be an admin at Acme Corp and a viewer at Beta Inc.
3. **Every query filters by `organizationId`.** This is not optional. Miss it once and you have a data leak.

```typescript
// ── Data access layer that enforces tenant isolation ─────────
// src/db/queries/posts.ts
import { db } from '@/db';
import { posts } from '@/db/schema';
import { and, eq, isNull, desc } from 'drizzle-orm';

export async function getPostsForOrg(organizationId: string) {
  return db
    .select()
    .from(posts)
    .where(
      and(
        eq(posts.organizationId, organizationId),
        isNull(posts.deletedAt)
      )
    )
    .orderBy(desc(posts.createdAt));
}

export async function getPublishedPostBySlug(
  organizationId: string,
  slug: string
) {
  return db.query.posts.findFirst({
    where: and(
      eq(posts.organizationId, organizationId),
      eq(posts.slug, slug),
      eq(posts.published, true),
      isNull(posts.deletedAt)
    ),
    with: {
      author: {
        columns: { name: true, avatarUrl: true },
      },
    },
  });
}
```

### Permissions Pattern

For fine-grained permissions beyond simple roles:

```typescript
// src/db/schema/permissions.ts
import { pgTable, uuid, varchar, timestamp, boolean, primaryKey } from 'drizzle-orm/pg-core';

export const permissions = pgTable(
  'permissions',
  {
    id: uuid('id').primaryKey().defaultRandom(),
    name: varchar('name', { length: 100 }).notNull(), // e.g., 'posts.create', 'posts.delete'
    description: text('description'),
  }
);

export const rolePermissions = pgTable(
  'role_permissions',
  {
    role: membershipRoleEnum('role').notNull(),
    permissionId: uuid('permission_id')
      .notNull()
      .references(() => permissions.id, { onDelete: 'cascade' }),
  },
  (table) => [
    primaryKey({ columns: [table.role, table.permissionId] }),
  ]
);

// ── Permission check helper ──────────────────────────────────
export async function hasPermission(
  userId: string,
  organizationId: string,
  permissionName: string
): Promise<boolean> {
  const result = await db
    .select({ count: count() })
    .from(memberships)
    .innerJoin(rolePermissions, eq(memberships.role, rolePermissions.role))
    .innerJoin(permissions, eq(rolePermissions.permissionId, permissions.id))
    .where(
      and(
        eq(memberships.userId, userId),
        eq(memberships.organizationId, organizationId),
        eq(permissions.name, permissionName)
      )
    );

  return result[0].count > 0;
}
```

### Polymorphic Relations

When different entity types share a common pattern (like comments that can be on posts, images, or documents):

```typescript
// ── Approach 1: Polymorphic table (entityType + entityId) ────
export const comments = pgTable('comments', {
  id: uuid('id').primaryKey().defaultRandom(),
  content: text('content').notNull(),
  entityType: varchar('entity_type', { length: 50 }).notNull(), // 'post', 'image', 'document'
  entityId: uuid('entity_id').notNull(),
  authorId: uuid('author_id').notNull().references(() => users.id),
  createdAt: timestamp('created_at', { withTimezone: true }).defaultNow().notNull(),
}, (table) => [
  index('comments_entity_idx').on(table.entityType, table.entityId),
]);

// ── Approach 2: Separate foreign keys (more type-safe) ───────
export const comments = pgTable('comments', {
  id: uuid('id').primaryKey().defaultRandom(),
  content: text('content').notNull(),
  // Only one of these is set per row
  postId: uuid('post_id').references(() => posts.id),
  imageId: uuid('image_id').references(() => images.id),
  documentId: uuid('document_id').references(() => documents.id),
  authorId: uuid('author_id').notNull().references(() => users.id),
  createdAt: timestamp('created_at', { withTimezone: true }).defaultNow().notNull(),
});
```

Approach 1 is more flexible (add new entity types without schema changes) but loses referential integrity at the database level. Approach 2 is stricter and type-safe but requires a schema migration for each new entity type.

**The 100x recommendation:** Use Approach 1 (polymorphic) for audit logs, comments, and notifications where flexibility matters. Use Approach 2 (separate FKs) for core domain models where integrity is critical.

---

## 6. MIGRATIONS

Migrations are the discipline that keeps your production database alive. A bad migration can take down your entire application. A missed migration can leave your schema out of sync. This section is about not destroying production data.

### Drizzle Kit Migrations

```bash
# ── Generate a migration from schema changes ──────────────────
pnpm drizzle-kit generate

# This creates a SQL file in ./drizzle/ like:
# drizzle/0001_add_posts_table.sql

# ── Push schema directly (development only, no migration file) ─
pnpm drizzle-kit push

# ── Apply migrations to the database ─────────────────────────
pnpm drizzle-kit migrate

# ── Drop all tables (development only!) ───────────────────────
pnpm drizzle-kit drop

# ── View migration status ─────────────────────────────────────
pnpm drizzle-kit check
```

A generated migration looks like this:

```sql
-- drizzle/0001_add_posts_table.sql
CREATE TABLE IF NOT EXISTS "posts" (
  "id" uuid PRIMARY KEY DEFAULT gen_random_uuid() NOT NULL,
  "title" varchar(500) NOT NULL,
  "slug" varchar(500) NOT NULL,
  "content" text,
  "excerpt" text,
  "metadata" jsonb,
  "published" boolean DEFAULT false NOT NULL,
  "published_at" timestamp with time zone,
  "author_id" uuid NOT NULL,
  "organization_id" uuid NOT NULL,
  "created_at" timestamp with time zone DEFAULT now() NOT NULL,
  "updated_at" timestamp with time zone DEFAULT now() NOT NULL,
  "deleted_at" timestamp with time zone
);

CREATE INDEX IF NOT EXISTS "posts_author_id_idx" ON "posts" ("author_id");
CREATE INDEX IF NOT EXISTS "posts_organization_id_idx" ON "posts" ("organization_id");
CREATE INDEX IF NOT EXISTS "posts_published_idx" ON "posts" ("published");
CREATE INDEX IF NOT EXISTS "posts_slug_org_idx" ON "posts" ("slug", "organization_id");

ALTER TABLE "posts"
  ADD CONSTRAINT "posts_author_id_users_id_fk"
  FOREIGN KEY ("author_id") REFERENCES "users"("id") ON DELETE NO ACTION ON UPDATE NO ACTION;

ALTER TABLE "posts"
  ADD CONSTRAINT "posts_organization_id_organizations_id_fk"
  FOREIGN KEY ("organization_id") REFERENCES "organizations"("id") ON DELETE CASCADE ON UPDATE NO ACTION;
```

### Prisma Migrations

```bash
# ── Create a migration (development) ─────────────────────────
pnpm prisma migrate dev --name add_posts_table

# ── Apply migrations in production ────────────────────────────
pnpm prisma migrate deploy

# ── Reset database (development only!) ────────────────────────
pnpm prisma migrate reset

# ── View migration status ─────────────────────────────────────
pnpm prisma migrate status
```

### The Golden Rules of Production Migrations

These rules exist because I've seen every one of them violated, and the result was always an outage:

#### Rule 1: Never Delete a Column in One Step

```
BAD:
  Migration 1: DROP COLUMN email_old

GOOD:
  Migration 1: Stop reading from email_old in code, deploy code
  Migration 2: (next week) DROP COLUMN email_old
```

Why: If you drop a column while old code is still running (during a rolling deploy), the old code will crash trying to read the dropped column.

#### Rule 2: Always Add Columns as Nullable First

```
BAD:
  ALTER TABLE users ADD COLUMN phone VARCHAR(20) NOT NULL;
  -- This fails if the table has existing rows

GOOD:
  Migration 1: ALTER TABLE users ADD COLUMN phone VARCHAR(20);  -- nullable
  Migration 2: UPDATE users SET phone = '' WHERE phone IS NULL;  -- backfill
  Migration 3: ALTER TABLE users ALTER COLUMN phone SET NOT NULL; -- constrain
```

#### Rule 3: Add Indexes Concurrently

```sql
-- BAD: Locks the entire table while building the index
CREATE INDEX posts_title_idx ON posts (title);

-- GOOD: Does not lock the table
CREATE INDEX CONCURRENTLY posts_title_idx ON posts (title);
```

> **Note:** Drizzle Kit does not generate `CONCURRENTLY` by default. For large tables in production, edit the generated migration SQL to add it.

#### Rule 4: Rename in Three Steps

```
BAD:
  ALTER TABLE users RENAME COLUMN name TO full_name;
  -- All running code that references "name" will crash

GOOD:
  Migration 1: Add "full_name" column, backfill from "name"
  Migration 2: Deploy code that reads from "full_name" (and writes to both)
  Migration 3: Deploy code that only reads/writes "full_name"
  Migration 4: Drop "name" column
```

#### Rule 5: Test Migrations on a Copy of Production

```bash
# Use Neon branching to create a copy of production
neon branches create --name migration-test --parent main

# Run migrations against the branch
DATABASE_URL=$BRANCH_URL pnpm drizzle-kit migrate

# Verify everything works
# Delete the branch when done
neon branches delete migration-test
```

### CI/CD Migration Strategy

```yaml
# .github/workflows/deploy.yml
name: Deploy
on:
  push:
    branches: [main]

jobs:
  migrate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run migrations
        env:
          DATABASE_URL: ${{ secrets.DATABASE_URL }}
        run: |
          pnpm drizzle-kit migrate

  deploy:
    needs: migrate  # Deploy AFTER migrations succeed
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Deploy to Vercel
        run: vercel deploy --prod --token=${{ secrets.VERCEL_TOKEN }}
```

**Critical:** Migrations run BEFORE deployment. The new code expects the new schema. If the migration fails, the deploy does not happen.

---

## 7. TYPE-SAFE QUERIES

Type safety is the entire reason we use TypeScript ORMs instead of writing raw SQL strings. Here is what "type-safe" actually means in practice, and how Drizzle delivers it.

### Inference from Schema

```typescript
import { posts } from '@/db/schema';

// The type of `posts.$inferSelect` is AUTOMATICALLY:
type Post = {
  id: string;
  title: string;
  slug: string;
  content: string | null;
  excerpt: string | null;
  metadata: {
    readingTime?: number;
    coverImage?: string;
    tags?: string[];
    seoTitle?: string;
    seoDescription?: string;
  } | null;
  published: boolean;
  publishedAt: Date | null;
  authorId: string;
  organizationId: string;
  createdAt: Date;
  updatedAt: Date;
  deletedAt: Date | null;
};

// The type of `posts.$inferInsert` knows which fields are optional:
type NewPost = {
  id?: string;           // optional (has defaultRandom())
  title: string;          // required
  slug: string;           // required
  content?: string | null;
  excerpt?: string | null;
  metadata?: { ... } | null;
  published?: boolean;    // optional (has default)
  publishedAt?: Date | null;
  authorId: string;       // required
  organizationId: string; // required
  createdAt?: Date;       // optional (has defaultNow())
  updatedAt?: Date;       // optional (has defaultNow())
  deletedAt?: Date | null;
};
```

### Select Narrows the Type

This is where Drizzle's inference is exceptional:

```typescript
// Full select -- returns Post[]
const full = await db.select().from(posts);
//    ^ Post[]

// Partial select -- TypeScript narrows the type automatically
const partial = await db
  .select({
    id: posts.id,
    title: posts.title,
    published: posts.published,
  })
  .from(posts);
//    ^ { id: string; title: string; published: boolean }[]

// Join select -- TypeScript merges the types
const joined = await db
  .select({
    postTitle: posts.title,
    authorName: users.name,
  })
  .from(posts)
  .innerJoin(users, eq(posts.authorId, users.id));
//    ^ { postTitle: string; authorName: string | null }[]
//      Note: authorName is `string | null` because users.name is nullable
```

### Type-Safe Where Clauses

```typescript
// This WORKS -- published is a boolean column
db.select().from(posts).where(eq(posts.published, true));

// This is a TYPE ERROR -- cannot compare boolean column to string
db.select().from(posts).where(eq(posts.published, 'yes'));
//                                                 ^^^^
// Type error: Argument of type 'string' is not assignable

// This is a TYPE ERROR -- misspelled column name
db.select().from(posts).where(eq(posts.publised, true));
//                                     ^^^^^^^^^
// Property 'publised' does not exist on type
```

### Building Dynamic Queries

Real applications need dynamic query building (filters, sorting, pagination):

```typescript
import { and, eq, ilike, desc, asc, isNull, SQL } from 'drizzle-orm';

interface PostFilters {
  search?: string;
  published?: boolean;
  authorId?: string;
  organizationId: string; // always required for tenant isolation
  sortBy?: 'createdAt' | 'title' | 'publishedAt';
  sortOrder?: 'asc' | 'desc';
  limit?: number;
  offset?: number;
}

export async function getPosts(filters: PostFilters) {
  const conditions: SQL[] = [
    eq(posts.organizationId, filters.organizationId),
    isNull(posts.deletedAt),
  ];

  if (filters.search) {
    conditions.push(ilike(posts.title, `%${filters.search}%`));
  }

  if (filters.published !== undefined) {
    conditions.push(eq(posts.published, filters.published));
  }

  if (filters.authorId) {
    conditions.push(eq(posts.authorId, filters.authorId));
  }

  const sortColumn = filters.sortBy
    ? posts[filters.sortBy]
    : posts.createdAt;

  const sortDirection = filters.sortOrder === 'asc' ? asc : desc;

  return db
    .select()
    .from(posts)
    .where(and(...conditions))
    .orderBy(sortDirection(sortColumn))
    .limit(filters.limit ?? 20)
    .offset(filters.offset ?? 0);
}
```

### Avoiding Common Type Pitfalls

```typescript
// ── PITFALL 1: Assuming non-null on nullable columns ─────────
const post = await db.query.posts.findFirst({
  where: eq(posts.id, postId),
});

// post is `Post | undefined` -- not `Post`!
// Always check for undefined
if (!post) {
  throw new Error('Post not found');
}

// ── PITFALL 2: Using `any` to silence type errors ────────────
// BAD
const results: any = await db.select().from(posts);

// GOOD -- let Drizzle infer the type
const results = await db.select().from(posts);

// ── PITFALL 3: Manual type assertions that lie ───────────────
// BAD -- if you add a column later, this type is wrong
interface PostRow {
  id: string;
  title: string;
}
const results = await db.select().from(posts) as PostRow[];

// GOOD -- use the inferred types
type PostRow = typeof posts.$inferSelect;
// Or just let the query infer it
```

---

## 8. RELATIONS AND JOINS

### Understanding N+1 Queries

The N+1 problem is the single most common performance issue in database-backed applications. Here it is:

```typescript
// ── THE N+1 PROBLEM ──────────────────────────────────────────
// 1 query to get all posts
const allPosts = await db.select().from(posts).limit(20);

// Then N queries to get each author (BAD!)
const postsWithAuthors = await Promise.all(
  allPosts.map(async (post) => {
    const [author] = await db
      .select()
      .from(users)
      .where(eq(users.id, post.authorId));
    return { ...post, author };
  })
);
// Total: 21 queries for 20 posts. With 100 posts, 101 queries.

// ── THE FIX: Use a JOIN ──────────────────────────────────────
const postsWithAuthors = await db
  .select({
    post: posts,
    author: {
      name: users.name,
      avatarUrl: users.avatarUrl,
    },
  })
  .from(posts)
  .innerJoin(users, eq(posts.authorId, users.id))
  .limit(20);
// Total: 1 query.

// ── OR: Use the relational API ───────────────────────────────
const postsWithAuthors = await db.query.posts.findMany({
  limit: 20,
  with: {
    author: {
      columns: { name: true, avatarUrl: true },
    },
  },
});
// Drizzle generates an efficient query (typically 1-2 queries)
```

### One-to-One Relations

```typescript
// Schema: User has one Profile
export const profiles = pgTable('profiles', {
  id: uuid('id').primaryKey().defaultRandom(),
  userId: uuid('user_id').notNull().references(() => users.id).unique(),
  bio: text('bio'),
  website: varchar('website', { length: 500 }),
  location: varchar('location', { length: 200 }),
});

// Relations
export const profilesRelations = relations(profiles, ({ one }) => ({
  user: one(users, {
    fields: [profiles.userId],
    references: [users.id],
  }),
}));

export const usersRelations = relations(users, ({ one, many }) => ({
  profile: one(profiles),
  // ... other relations
}));

// Query
const userWithProfile = await db.query.users.findFirst({
  where: eq(users.id, userId),
  with: { profile: true },
});
// Type: { id: string; name: string; ...; profile: Profile | null }
```

### One-to-Many Relations

```typescript
// Already defined in our schema: User has many Posts
const authorWithPosts = await db.query.users.findFirst({
  where: eq(users.id, userId),
  with: {
    posts: {
      where: eq(posts.published, true),
      orderBy: [desc(posts.publishedAt)],
      limit: 10,
    },
  },
});
// Type: { id: string; name: string; ...; posts: Post[] }
```

### Many-to-Many Relations

Our `memberships` table is the classic many-to-many join table:

```typescript
// User belongs to many Organizations (through Memberships)
// Organization has many Users (through Memberships)

// Get all organizations a user belongs to
const userOrgs = await db.query.users.findFirst({
  where: eq(users.id, userId),
  with: {
    memberships: {
      with: {
        organization: true,
      },
    },
  },
});

// Result shape:
// {
//   id: '...',
//   name: 'Alice',
//   memberships: [
//     {
//       role: 'owner',
//       joinedAt: Date,
//       organization: { id: '...', name: 'Acme Corp', slug: 'acme-corp' }
//     },
//     {
//       role: 'member',
//       joinedAt: Date,
//       organization: { id: '...', name: 'Beta Inc', slug: 'beta-inc' }
//     }
//   ]
// }
```

### Complex Joins with the SQL-like API

When the relational API is not enough (aggregations, subqueries, complex conditions):

```typescript
// ── Organization dashboard: members with their post counts ───
const orgDashboard = await db
  .select({
    userId: users.id,
    userName: users.name,
    userEmail: users.email,
    role: memberships.role,
    joinedAt: memberships.joinedAt,
    postCount: count(posts.id),
    lastPostDate: sql<Date | null>`max(${posts.createdAt})`,
  })
  .from(memberships)
  .innerJoin(users, eq(memberships.userId, users.id))
  .leftJoin(
    posts,
    and(
      eq(posts.authorId, users.id),
      eq(posts.organizationId, memberships.organizationId)
    )
  )
  .where(eq(memberships.organizationId, orgId))
  .groupBy(users.id, users.name, users.email, memberships.role, memberships.joinedAt)
  .orderBy(desc(count(posts.id)));
```

### Eager vs Lazy Loading

Drizzle does not have lazy loading. Every query is explicit. This is a feature, not a limitation.

```typescript
// ── Eager loading (Drizzle's only mode -- and the right one) ─
const post = await db.query.posts.findFirst({
  where: eq(posts.id, postId),
  with: { author: true }, // You explicitly ask for the author
});

// ── Prisma's lazy loading (available but discouraged) ────────
// In Prisma, if you access post.author without including it,
// Prisma can make an additional query (with Prisma Client extensions).
// This is implicit and can cause N+1 problems silently.
// Always use `include` or `select` explicitly in Prisma too.
```

**The 100x principle:** Explicit is better than implicit. Always declare what data you need upfront. Never rely on lazy loading.

---

## 9. CONNECTION POOLING FOR SERVERLESS

This section exists because serverless + databases is genuinely hard, and getting it wrong means your application goes down at the worst possible time -- when traffic spikes.

### The Problem

Traditional applications maintain a pool of database connections that persist across requests. A Node.js Express server might open 10 connections at startup and reuse them for thousands of requests.

Serverless functions are different:

```
┌─────────────────────────────────────────────────────────────────┐
│              THE SERVERLESS CONNECTION PROBLEM                   │
│                                                                  │
│  Traditional server:                                            │
│    Server starts → opens 10 connections → handles 10,000 reqs   │
│                                                                  │
│  Serverless:                                                     │
│    Request 1  → cold start → opens connection 1                 │
│    Request 2  → cold start → opens connection 2                 │
│    Request 3  → cold start → opens connection 3                 │
│    ...                                                           │
│    Request 100 → cold start → opens connection 100              │
│    Request 101 → ERROR: too many connections!                    │
│                                                                  │
│  Postgres default max_connections: 100                           │
│  Neon free tier: 100 connections                                │
│  A traffic spike of 100+ concurrent users = connection exhaustion│
└─────────────────────────────────────────────────────────────────┘
```

Each serverless function invocation may:
1. Start a new runtime (cold start)
2. Open a new database connection
3. Run the query
4. The function freezes (but the connection stays open)

With enough concurrent requests, you exhaust the database's connection limit.

### The Solutions

#### Solution 1: Neon's Built-In Connection Pooler

Neon provides a built-in PgBouncer-based connection pooler. You just use a different connection string:

```bash
# Direct connection (for migrations, admin tasks)
DIRECT_DATABASE_URL="postgresql://user:pass@ep-xyz-123.us-east-2.aws.neon.tech/mydb"

# Pooled connection (for application queries)
DATABASE_URL="postgresql://user:pass@ep-xyz-123-pooler.us-east-2.aws.neon.tech/mydb"
#                                      ^^^^^^^
#                          Note: "-pooler" in the hostname
```

```typescript
// src/db/index.ts -- Neon with connection pooling
import { drizzle } from 'drizzle-orm/neon-http';
import { neon } from '@neondatabase/serverless';
import * as schema from './schema';

// The HTTP driver is ideal for serverless -- no persistent connections
const sql = neon(process.env.DATABASE_URL!);
export const db = drizzle(sql, { schema });
```

The Neon HTTP driver (`@neondatabase/serverless` with `neon()`) is the best option for Vercel Functions. It uses HTTP to communicate with Neon, so there are no persistent TCP connections to pool. Each query is a stateless HTTP request.

For transactions (which require a persistent connection):

```typescript
import { drizzle } from 'drizzle-orm/neon-http';
import { neon } from '@neondatabase/serverless';
import { Pool, neonConfig } from '@neondatabase/serverless';
import * as schema from './schema';

// HTTP client for simple queries (stateless, no connection management)
const sql = neon(process.env.DATABASE_URL!);
export const db = drizzle(sql, { schema });

// Pool client for transactions (connection-based)
import { drizzle as drizzlePool } from 'drizzle-orm/neon-serverless';

const pool = new Pool({ connectionString: process.env.DATABASE_URL! });
export const dbPool = drizzlePool(pool, { schema });

// Use dbPool when you need transactions
await dbPool.transaction(async (tx) => {
  // ...
});
```

#### Solution 2: Prisma with Connection Pooling

Prisma recommends using their Accelerate service or configuring the connection pool in the Prisma Client:

```typescript
// prisma/schema.prisma
datasource db {
  provider  = "postgresql"
  url       = env("DATABASE_URL")       // pooled URL
  directUrl = env("DIRECT_DATABASE_URL") // direct URL for migrations
}

// src/db/prisma.ts
import { PrismaClient } from '@prisma/client';

const globalForPrisma = globalThis as unknown as {
  prisma: PrismaClient | undefined;
};

export const prisma = globalForPrisma.prisma ?? new PrismaClient({
  datasources: {
    db: {
      url: process.env.DATABASE_URL, // Use pooled connection string
    },
  },
});

if (process.env.NODE_ENV !== 'production') {
  globalForPrisma.prisma = prisma;
}
```

#### Solution 3: Turso (No Pooling Needed)

Turso uses HTTP for all communication. There are no persistent database connections. Connection pooling is not relevant.

```typescript
import { drizzle } from 'drizzle-orm/libsql';
import { createClient } from '@libsql/client';
import * as schema from './schema';

const client = createClient({
  url: process.env.TURSO_DATABASE_URL!,
  authToken: process.env.TURSO_AUTH_TOKEN!,
});

export const db = drizzle(client, { schema });
```

### Connection Pooling Decision Tree

```
Q: Are you using Vercel Functions (or any serverless)?
├── No → Standard connection pool in your ORM is fine
└── Yes →
    Q: What database?
    ├── Neon → Use the HTTP driver (neon() from @neondatabase/serverless)
    │         No pooler needed for simple queries.
    │         Use the pooled connection string for transactions.
    ├── Supabase → Use the Supabase JS client (uses HTTP)
    │              Or use the pooled connection string (port 6543)
    ├── Turso → HTTP by default. No pooling needed.
    └── Other Postgres → You MUST set up PgBouncer or equivalent.
                          Or use Prisma Accelerate.
```

---

## 10. NEON POSTGRES DEEP DIVE

Neon is the recommended database for Next.js applications deployed to Vercel. Here is why, and how to use it fully.

### Why Neon

1. **Autoscale to zero.** Your database costs $0 when nobody is using it. It wakes up in ~500ms on the first query.
2. **Branching.** Create a full copy of your database in milliseconds. Preview deployments get their own database branch.
3. **Built-in connection pooling.** PgBouncer is included. Use the pooled connection string.
4. **Vercel integration.** One click to connect. Environment variables are set automatically.
5. **Generous free tier.** 0.5 GB storage, 190 compute hours/month, branching included.

### Setup with Vercel

```bash
# Via Vercel Dashboard:
# 1. Go to your project → Storage → Connect Store → Neon
# 2. Follow the prompts
# 3. Environment variables are auto-populated:
#    - DATABASE_URL (pooled)
#    - DATABASE_URL_UNPOOLED (direct)
#    - PGHOST, PGDATABASE, PGUSER, PGPASSWORD

# Or via CLI:
vercel integration add neon
```

### Database Branching

This is Neon's killer feature. Every preview deployment gets its own database branch -- a copy-on-write clone of your production data.

```
┌─────────────────────────────────────────────────────────────────┐
│                    NEON BRANCHING MODEL                          │
│                                                                  │
│   main (production)                                             │
│   ├── pr-123-feature-auth (preview deployment)                  │
│   │   └── Has a copy of production data                         │
│   │   └── Migrations from the PR are applied                    │
│   │   └── Changes do NOT affect production                      │
│   │   └── Deleted when the PR is merged/closed                  │
│   ├── pr-456-redesign (preview deployment)                      │
│   │   └── Independent copy of production data                   │
│   └── staging                                                    │
│       └── Long-lived branch for staging environment             │
│                                                                  │
│   Branches are created in ~1 second (copy-on-write).            │
│   Storage is shared -- you only pay for the diff.               │
└─────────────────────────────────────────────────────────────────┘
```

#### Setting Up Branch-per-Preview

```yaml
# .github/workflows/preview.yml
name: Preview Deployment
on:
  pull_request:
    types: [opened, synchronize, reopened]

jobs:
  preview:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Create Neon Branch
        id: neon
        uses: neondatabase/create-branch-action@v5
        with:
          project_id: ${{ secrets.NEON_PROJECT_ID }}
          branch_name: pr-${{ github.event.pull_request.number }}
          api_key: ${{ secrets.NEON_API_KEY }}
          # Parent branch -- clone from production
          parent: main

      - name: Run Migrations on Branch
        env:
          DATABASE_URL: ${{ steps.neon.outputs.db_url_with_pooler }}
        run: pnpm drizzle-kit migrate

      - name: Deploy Preview
        env:
          DATABASE_URL: ${{ steps.neon.outputs.db_url_with_pooler }}
        run: vercel deploy --token=${{ secrets.VERCEL_TOKEN }}
```

```yaml
# .github/workflows/cleanup.yml
name: Cleanup Preview
on:
  pull_request:
    types: [closed]

jobs:
  cleanup:
    runs-on: ubuntu-latest
    steps:
      - name: Delete Neon Branch
        uses: neondatabase/delete-branch-action@v3
        with:
          project_id: ${{ secrets.NEON_PROJECT_ID }}
          branch: pr-${{ github.event.pull_request.number }}
          api_key: ${{ secrets.NEON_API_KEY }}
```

### Neon CLI

```bash
# Install
brew install neonctl
# or
npm i -g neonctl

# Auth
neonctl auth

# List projects
neonctl projects list

# Create a branch
neonctl branches create --name feature-xyz --project-id your-project-id

# Get connection string
neonctl connection-string --branch feature-xyz --project-id your-project-id

# Delete a branch
neonctl branches delete feature-xyz --project-id your-project-id
```

### Point-in-Time Restore

Neon keeps a history of your data. You can restore to any point in time within your retention window:

```bash
# Restore to a specific timestamp
neonctl branches create \
  --name restore-point \
  --project-id your-project-id \
  --parent main \
  --restore-point "2026-04-01T12:00:00Z"
```

This creates a new branch with your data as it was at that timestamp. Useful for recovering from bad migrations or data corruption.

---

## 11. SEEDING

Seed scripts populate your database with initial or test data. Good seeding saves hours of manual data entry and makes development reproducible.

### The Seed Script

```typescript
// src/db/seed.ts
import { db } from './index';
import {
  users,
  organizations,
  memberships,
  posts,
} from './schema';
import { faker } from '@faker-js/faker';
import { eq } from 'drizzle-orm';

// Seed configuration
const SEED = 42; // Deterministic randomness
faker.seed(SEED);

async function seed() {
  console.log('🌱 Seeding database...');

  // Clear existing data (development only!)
  console.log('Clearing existing data...');
  await db.delete(posts);
  await db.delete(memberships);
  await db.delete(organizations);
  await db.delete(users);

  // ── Create users ─────────────────────────────────────────
  console.log('Creating users...');
  const createdUsers = await db
    .insert(users)
    .values(
      Array.from({ length: 20 }, () => ({
        email: faker.internet.email().toLowerCase(),
        name: faker.person.fullName(),
        avatarUrl: faker.image.avatar(),
        role: 'member' as const,
        emailVerified: faker.datatype.boolean(),
      }))
    )
    .returning();

  // ── Create organizations ─────────────────────────────────
  console.log('Creating organizations...');
  const orgNames = [
    'Acme Corporation',
    'Beta Industries',
    'Gamma Labs',
    'Delta Systems',
    'Epsilon Technologies',
  ];

  const createdOrgs = await db
    .insert(organizations)
    .values(
      orgNames.map((name) => ({
        name,
        slug: name.toLowerCase().replace(/\s+/g, '-'),
        logoUrl: faker.image.urlLoremFlickr({ category: 'business' }),
      }))
    )
    .returning();

  // ── Create memberships ───────────────────────────────────
  console.log('Creating memberships...');
  const membershipValues = [];

  for (const org of createdOrgs) {
    // First user is always the owner
    membershipValues.push({
      userId: createdUsers[0].id,
      organizationId: org.id,
      role: 'owner' as const,
    });

    // Add 3-8 random members
    const memberCount = faker.number.int({ min: 3, max: 8 });
    const shuffled = faker.helpers.shuffle(createdUsers.slice(1));

    for (let i = 0; i < memberCount; i++) {
      membershipValues.push({
        userId: shuffled[i].id,
        organizationId: org.id,
        role: faker.helpers.arrayElement(['admin', 'member', 'viewer'] as const),
      });
    }
  }

  await db.insert(memberships).values(membershipValues);

  // ── Create posts ─────────────────────────────────────────
  console.log('Creating posts...');
  const postValues = [];

  for (const org of createdOrgs) {
    // Get members of this org
    const orgMemberships = membershipValues.filter(
      (m) => m.organizationId === org.id
    );

    for (let i = 0; i < 15; i++) {
      const isPublished = faker.datatype.boolean({ probability: 0.7 });
      const author = faker.helpers.arrayElement(orgMemberships);

      postValues.push({
        title: faker.lorem.sentence({ min: 3, max: 10 }),
        slug: faker.lorem.slug({ min: 2, max: 5 }),
        content: faker.lorem.paragraphs({ min: 3, max: 10 }),
        excerpt: faker.lorem.paragraph(),
        metadata: {
          readingTime: faker.number.int({ min: 2, max: 15 }),
          tags: faker.helpers.arrayElements(
            ['engineering', 'design', 'product', 'culture', 'tutorial'],
            { min: 1, max: 3 }
          ),
        },
        published: isPublished,
        publishedAt: isPublished
          ? faker.date.past({ years: 1 })
          : null,
        authorId: author.userId,
        organizationId: org.id,
      });
    }
  }

  await db.insert(posts).values(postValues);

  console.log('✅ Seeding complete!');
  console.log(`   Created ${createdUsers.length} users`);
  console.log(`   Created ${createdOrgs.length} organizations`);
  console.log(`   Created ${membershipValues.length} memberships`);
  console.log(`   Created ${postValues.length} posts`);
}

seed()
  .catch((error) => {
    console.error('Seed failed:', error);
    process.exit(1);
  })
  .finally(async () => {
    process.exit(0);
  });
```

### Running the Seed Script

```json
// package.json
{
  "scripts": {
    "db:generate": "drizzle-kit generate",
    "db:migrate": "drizzle-kit migrate",
    "db:push": "drizzle-kit push",
    "db:seed": "tsx src/db/seed.ts",
    "db:studio": "drizzle-kit studio",
    "db:reset": "drizzle-kit drop && drizzle-kit push && tsx src/db/seed.ts"
  }
}
```

```bash
# Run the seed
pnpm db:seed

# Full reset: drop all tables, recreate schema, seed
pnpm db:reset
```

### Idempotent Seeds

For production seeding (initial data like roles, permissions, default settings), seeds must be idempotent:

```typescript
// src/db/seed-production.ts
import { db } from './index';
import { permissions } from './schema';
import { sql } from 'drizzle-orm';

const DEFAULT_PERMISSIONS = [
  { name: 'posts.create', description: 'Create new posts' },
  { name: 'posts.edit', description: 'Edit any post' },
  { name: 'posts.delete', description: 'Delete any post' },
  { name: 'posts.publish', description: 'Publish or unpublish posts' },
  { name: 'members.invite', description: 'Invite new members' },
  { name: 'members.remove', description: 'Remove members' },
  { name: 'members.change-role', description: 'Change member roles' },
  { name: 'org.settings', description: 'Modify organization settings' },
  { name: 'org.billing', description: 'Access billing information' },
  { name: 'org.delete', description: 'Delete the organization' },
];

async function seedProduction() {
  console.log('Seeding production defaults...');

  // Upsert: insert if not exists, skip if already present
  for (const perm of DEFAULT_PERMISSIONS) {
    await db
      .insert(permissions)
      .values(perm)
      .onConflictDoNothing({ target: permissions.name });
  }

  console.log('Production seed complete.');
}

seedProduction()
  .catch(console.error)
  .finally(() => process.exit(0));
```

### Per-Environment Seeding

```typescript
// src/db/seed.ts
const ENVIRONMENT = process.env.NODE_ENV || 'development';

async function seed() {
  // Always run production seeds (permissions, roles, etc.)
  await seedProduction();

  if (ENVIRONMENT === 'development' || ENVIRONMENT === 'test') {
    // Only in dev/test: create fake users, posts, etc.
    await seedDevelopment();
  }

  if (ENVIRONMENT === 'test') {
    // Only in test: create specific fixture data for tests
    await seedTestFixtures();
  }
}
```

---

## 12. SERVER COMPONENTS + DATABASE

This is where everything comes together. Server Components querying the database directly is the defining pattern of the modern Next.js data architecture.

### Direct Database Queries in Server Components

```typescript
// app/dashboard/page.tsx
import { db } from '@/db';
import { posts, users, organizations, memberships } from '@/db/schema';
import { eq, and, desc, count, isNull } from 'drizzle-orm';
import { auth } from '@/lib/auth';
import { redirect } from 'next/navigation';

export default async function DashboardPage() {
  const session = await auth();
  if (!session?.user) redirect('/login');

  const userId = session.user.id;
  const orgId = session.user.organizationId;

  // These queries run in parallel on the server
  const [recentPosts, memberCount, orgDetails] = await Promise.all([
    // Recent posts for this organization
    db.query.posts.findMany({
      where: and(
        eq(posts.organizationId, orgId),
        isNull(posts.deletedAt)
      ),
      orderBy: [desc(posts.updatedAt)],
      limit: 5,
      with: {
        author: {
          columns: { name: true, avatarUrl: true },
        },
      },
    }),

    // Total member count
    db
      .select({ count: count() })
      .from(memberships)
      .where(eq(memberships.organizationId, orgId))
      .then(([r]) => r.count),

    // Organization details
    db.query.organizations.findFirst({
      where: eq(organizations.id, orgId),
    }),
  ]);

  return (
    <div>
      <h1>Welcome back to {orgDetails?.name}</h1>
      <p>{memberCount} team members</p>

      <h2>Recent Posts</h2>
      {recentPosts.map((post) => (
        <article key={post.id}>
          <h3>{post.title}</h3>
          <p>By {post.author.name}</p>
          <time>{post.updatedAt.toLocaleDateString()}</time>
        </article>
      ))}
    </div>
  );
}
```

### Server Actions for Mutations

```typescript
// app/posts/actions.ts
'use server';

import { db } from '@/db';
import { posts, auditLogs } from '@/db/schema';
import { eq, and } from 'drizzle-orm';
import { auth } from '@/lib/auth';
import { revalidatePath } from 'next/cache';
import { redirect } from 'next/navigation';
import { z } from 'zod';

const CreatePostSchema = z.object({
  title: z.string().min(1).max(500),
  slug: z.string().min(1).max(500).regex(/^[a-z0-9-]+$/),
  content: z.string().optional(),
  excerpt: z.string().optional(),
});

export async function createPost(formData: FormData) {
  const session = await auth();
  if (!session?.user) throw new Error('Unauthorized');

  const validated = CreatePostSchema.parse({
    title: formData.get('title'),
    slug: formData.get('slug'),
    content: formData.get('content'),
    excerpt: formData.get('excerpt'),
  });

  const [newPost] = await db
    .insert(posts)
    .values({
      ...validated,
      authorId: session.user.id,
      organizationId: session.user.organizationId,
    })
    .returning();

  // Create audit log
  await db.insert(auditLogs).values({
    action: 'post.created',
    entityType: 'post',
    entityId: newPost.id,
    actorId: session.user.id,
    organizationId: session.user.organizationId,
  });

  revalidatePath('/dashboard');
  redirect(`/posts/${newPost.slug}`);
}

export async function publishPost(postId: string) {
  const session = await auth();
  if (!session?.user) throw new Error('Unauthorized');

  const [updatedPost] = await db
    .update(posts)
    .set({
      published: true,
      publishedAt: new Date(),
    })
    .where(
      and(
        eq(posts.id, postId),
        eq(posts.organizationId, session.user.organizationId)
      )
    )
    .returning();

  if (!updatedPost) throw new Error('Post not found');

  await db.insert(auditLogs).values({
    action: 'post.published',
    entityType: 'post',
    entityId: postId,
    actorId: session.user.id,
    organizationId: session.user.organizationId,
  });

  revalidatePath('/dashboard');
  revalidatePath(`/posts/${updatedPost.slug}`);
}

export async function deletePost(postId: string) {
  const session = await auth();
  if (!session?.user) throw new Error('Unauthorized');

  // Soft delete
  await db
    .update(posts)
    .set({ deletedAt: new Date() })
    .where(
      and(
        eq(posts.id, postId),
        eq(posts.organizationId, session.user.organizationId)
      )
    );

  await db.insert(auditLogs).values({
    action: 'post.deleted',
    entityType: 'post',
    entityId: postId,
    actorId: session.user.id,
    organizationId: session.user.organizationId,
  });

  revalidatePath('/dashboard');
}
```

### Using Server Actions in Client Components

```typescript
// app/posts/new/post-form.tsx
'use client';

import { createPost } from '../actions';
import { useActionState } from 'react';

export function PostForm() {
  const [state, formAction, isPending] = useActionState(createPost, null);

  return (
    <form action={formAction}>
      <div>
        <label htmlFor="title">Title</label>
        <input
          id="title"
          name="title"
          type="text"
          required
          disabled={isPending}
        />
      </div>

      <div>
        <label htmlFor="slug">Slug</label>
        <input
          id="slug"
          name="slug"
          type="text"
          required
          pattern="[a-z0-9-]+"
          disabled={isPending}
        />
      </div>

      <div>
        <label htmlFor="content">Content</label>
        <textarea
          id="content"
          name="content"
          rows={10}
          disabled={isPending}
        />
      </div>

      <button type="submit" disabled={isPending}>
        {isPending ? 'Creating...' : 'Create Post'}
      </button>
    </form>
  );
}
```

### Caching Database Queries

Next.js provides caching at the framework level. Use `unstable_cache` (or the newer `use cache` directive) to cache database query results:

```typescript
// src/db/queries/posts.ts
import { unstable_cache } from 'next/cache';
import { db } from '@/db';
import { posts } from '@/db/schema';
import { eq, and, desc, isNull } from 'drizzle-orm';

// Cached query -- result is cached and revalidated by tag
export const getPublishedPosts = unstable_cache(
  async (organizationId: string) => {
    return db.query.posts.findMany({
      where: and(
        eq(posts.organizationId, organizationId),
        eq(posts.published, true),
        isNull(posts.deletedAt)
      ),
      orderBy: [desc(posts.publishedAt)],
      with: {
        author: {
          columns: { name: true, avatarUrl: true },
        },
      },
    });
  },
  ['published-posts'], // cache key prefix
  {
    tags: ['posts'], // revalidation tag
    revalidate: 60,  // seconds
  }
);

// When a post is created/updated/deleted, revalidate the cache
import { revalidateTag } from 'next/cache';
revalidateTag('posts');
```

With the newer `use cache` directive (Next.js 15+):

```typescript
// app/posts/page.tsx
import { db } from '@/db';
import { posts } from '@/db/schema';
import { eq, desc, isNull, and } from 'drizzle-orm';

async function getPublishedPosts(orgId: string) {
  'use cache';
  // This function's result is cached automatically
  return db.query.posts.findMany({
    where: and(
      eq(posts.organizationId, orgId),
      eq(posts.published, true),
      isNull(posts.deletedAt)
    ),
    orderBy: [desc(posts.publishedAt)],
    with: {
      author: { columns: { name: true, avatarUrl: true } },
    },
  });
}

export default async function PostsPage() {
  const session = await auth();
  const posts = await getPublishedPosts(session.user.organizationId);

  return (
    <div>
      {posts.map((post) => (
        <PostCard key={post.id} post={post} />
      ))}
    </div>
  );
}
```

### When to Add an API Layer

Direct database access from Server Components is the default. Add an API layer when:

1. **Mobile clients need the same data.** React Native apps cannot call your Server Actions. Expose Route Handlers (`app/api/`) or a tRPC router for mobile.
2. **Third-party integrations.** Webhooks, partner APIs, and external services need HTTP endpoints.
3. **Real-time data.** Server Components are request-response. For WebSocket/SSE-based real-time updates, you need an API layer (or Supabase Realtime).
4. **Cross-platform shared business logic.** When the same mutation must be callable from web, mobile, and API consumers.

```typescript
// app/api/posts/route.ts -- REST API for mobile clients
import { db } from '@/db';
import { posts } from '@/db/schema';
import { eq, and, desc, isNull } from 'drizzle-orm';
import { NextRequest, NextResponse } from 'next/server';
import { verifyApiToken } from '@/lib/auth';

export async function GET(request: NextRequest) {
  const token = await verifyApiToken(request);
  if (!token) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
  }

  const orgId = request.nextUrl.searchParams.get('organizationId');
  if (!orgId) {
    return NextResponse.json({ error: 'organizationId required' }, { status: 400 });
  }

  const result = await db.query.posts.findMany({
    where: and(
      eq(posts.organizationId, orgId),
      eq(posts.published, true),
      isNull(posts.deletedAt)
    ),
    orderBy: [desc(posts.publishedAt)],
    with: {
      author: { columns: { name: true, avatarUrl: true } },
    },
  });

  return NextResponse.json({ posts: result });
}
```

---

## 13. SUPABASE AS A FULL PLATFORM

Supabase is not just a database. It is Postgres + Auth + Realtime + Storage + Edge Functions. For certain product shapes, Supabase replaces your entire backend.

### When Supabase Makes Sense

Supabase is the right choice when:

- You want Postgres but also need Auth, file storage, and real-time out of the box
- Your team is small and you want to minimize the number of services
- You need Row Level Security (RLS) for multi-tenant isolation at the database level
- You need real-time subscriptions (live cursors, live chat, collaborative editing)
- You are building a cross-platform product (web + mobile) and want a unified backend

Supabase is NOT the right choice when:

- You need fine-grained control over your server-side logic (Server Components + Drizzle is better)
- You are already using Neon and just need a database
- You want to avoid vendor lock-in on the auth and storage layers

### Setup

```bash
pnpm add @supabase/supabase-js @supabase/ssr
```

```typescript
// src/lib/supabase/server.ts
import { createServerClient } from '@supabase/ssr';
import { cookies } from 'next/headers';

export async function createClient() {
  const cookieStore = await cookies();

  return createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll() {
          return cookieStore.getAll();
        },
        setAll(cookiesToSet) {
          try {
            cookiesToSet.forEach(({ name, value, options }) =>
              cookieStore.set(name, value, options)
            );
          } catch {
            // The `setAll` method was called from a Server Component.
            // This can be ignored if you have middleware refreshing sessions.
          }
        },
      },
    }
  );
}
```

```typescript
// src/lib/supabase/client.ts
import { createBrowserClient } from '@supabase/ssr';

export function createClient() {
  return createBrowserClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
  );
}
```

### Row Level Security (RLS)

RLS is Supabase's approach to authorization at the database level. Instead of checking permissions in your application code, you define policies on the database that Postgres enforces on every query.

```sql
-- Enable RLS on the posts table
ALTER TABLE posts ENABLE ROW LEVEL SECURITY;

-- Policy: Users can only read published posts in their organization
CREATE POLICY "Users can read published posts in their org"
ON posts FOR SELECT
USING (
  published = true
  AND organization_id IN (
    SELECT organization_id
    FROM memberships
    WHERE user_id = auth.uid()
  )
);

-- Policy: Users can insert posts in their organization
CREATE POLICY "Users can create posts in their org"
ON posts FOR INSERT
WITH CHECK (
  organization_id IN (
    SELECT organization_id
    FROM memberships
    WHERE user_id = auth.uid()
  )
  AND author_id = auth.uid()
);

-- Policy: Users can update their own posts
CREATE POLICY "Users can update own posts"
ON posts FOR UPDATE
USING (author_id = auth.uid())
WITH CHECK (author_id = auth.uid());

-- Policy: Admins/owners can update any post in their org
CREATE POLICY "Admins can update any post in their org"
ON posts FOR UPDATE
USING (
  organization_id IN (
    SELECT organization_id
    FROM memberships
    WHERE user_id = auth.uid()
    AND role IN ('admin', 'owner')
  )
);
```

Now every query through the Supabase client automatically respects these policies:

```typescript
// This query ONLY returns posts the user is allowed to see
// RLS handles the filtering -- no application-level auth check needed
const { data: posts, error } = await supabase
  .from('posts')
  .select('*, author:users(name, avatar_url)')
  .eq('published', true)
  .order('published_at', { ascending: false })
  .limit(20);
```

### Real-Time Subscriptions

```typescript
// Subscribe to new posts in real time
'use client';

import { useEffect, useState } from 'react';
import { createClient } from '@/lib/supabase/client';
import type { Post } from '@/db/schema';

export function useRealtimePosts(organizationId: string) {
  const [posts, setPosts] = useState<Post[]>([]);
  const supabase = createClient();

  useEffect(() => {
    // Subscribe to changes on the posts table
    const channel = supabase
      .channel('posts-changes')
      .on(
        'postgres_changes',
        {
          event: '*', // INSERT, UPDATE, DELETE
          schema: 'public',
          table: 'posts',
          filter: `organization_id=eq.${organizationId}`,
        },
        (payload) => {
          if (payload.eventType === 'INSERT') {
            setPosts((prev) => [payload.new as Post, ...prev]);
          } else if (payload.eventType === 'UPDATE') {
            setPosts((prev) =>
              prev.map((p) =>
                p.id === (payload.new as Post).id ? (payload.new as Post) : p
              )
            );
          } else if (payload.eventType === 'DELETE') {
            setPosts((prev) =>
              prev.filter((p) => p.id !== (payload.old as Post).id)
            );
          }
        }
      )
      .subscribe();

    return () => {
      supabase.removeChannel(channel);
    };
  }, [organizationId, supabase]);

  return posts;
}
```

### Supabase + Drizzle

You can use Supabase as the database provider and Drizzle as the ORM. This gives you the best of both worlds: Supabase's Auth/Realtime/Storage plus Drizzle's type-safe queries.

```typescript
// src/db/index.ts -- Drizzle with Supabase Postgres
import { drizzle } from 'drizzle-orm/postgres-js';
import postgres from 'postgres';
import * as schema from './schema';

// Use the Supabase Postgres connection string (pooled)
const connectionString = process.env.DATABASE_URL!;
const client = postgres(connectionString);

export const db = drizzle(client, { schema });
```

Use the Supabase client for Auth, Realtime, and Storage. Use Drizzle for all database queries and mutations.

### Supabase for React Native

Supabase works well as a unified backend for cross-platform apps:

```typescript
// In your React Native app
import { createClient } from '@supabase/supabase-js';
import AsyncStorage from '@react-native-async-storage/async-storage';

const supabase = createClient(
  process.env.EXPO_PUBLIC_SUPABASE_URL!,
  process.env.EXPO_PUBLIC_SUPABASE_ANON_KEY!,
  {
    auth: {
      storage: AsyncStorage,
      autoRefreshToken: true,
      persistSession: true,
      detectSessionInUrl: false,
    },
  }
);

// Same APIs work across web and mobile
const { data: posts } = await supabase
  .from('posts')
  .select('*')
  .eq('published', true);
```

---

## 14. THE COMPLETE DATA ARCHITECTURE

Let's wire everything together. Here is the complete data architecture for a production application: Drizzle + Neon + Next.js Server Components, with an API layer for React Native.

### Project Structure

```
src/
├── db/
│   ├── index.ts              # Database client (Drizzle + Neon)
│   ├── schema/
│   │   ├── index.ts          # Barrel export
│   │   ├── users.ts          # Users table
│   │   ├── organizations.ts  # Organizations table
│   │   ├── memberships.ts    # Memberships (many-to-many)
│   │   ├── posts.ts          # Posts table
│   │   ├── audit-logs.ts     # Audit logs
│   │   └── relations.ts      # All relation definitions
│   ├── queries/
│   │   ├── posts.ts          # Post query functions
│   │   ├── users.ts          # User query functions
│   │   └── organizations.ts  # Organization query functions
│   ├── mutations/
│   │   ├── posts.ts          # Post mutation functions
│   │   ├── users.ts          # User mutation functions
│   │   └── organizations.ts  # Organization mutation functions
│   └── seed.ts               # Seed script
├── app/
│   ├── dashboard/
│   │   └── page.tsx          # Server Component (direct DB queries)
│   ├── posts/
│   │   ├── page.tsx          # Posts list (Server Component)
│   │   ├── [slug]/
│   │   │   └── page.tsx      # Single post (Server Component)
│   │   └── actions.ts        # Server Actions for mutations
│   └── api/
│       ├── posts/
│       │   └── route.ts      # REST API for mobile
│       └── trpc/
│           └── [trpc]/
│               └── route.ts  # tRPC for mobile
├── lib/
│   └── auth.ts               # Authentication
├── drizzle.config.ts         # Drizzle Kit config
└── drizzle/
    ├── 0000_init.sql         # Initial migration
    ├── 0001_add_posts.sql    # Subsequent migration
    └── meta/                 # Migration metadata
```

### The Data Access Layer

Separate your queries and mutations into a clean data access layer:

```typescript
// src/db/queries/posts.ts
import { db } from '@/db';
import { posts, users } from '@/db/schema';
import { and, eq, desc, isNull, ilike, count, SQL } from 'drizzle-orm';

export interface PostListParams {
  organizationId: string;
  published?: boolean;
  authorId?: string;
  search?: string;
  limit?: number;
  offset?: number;
}

export async function listPosts(params: PostListParams) {
  const {
    organizationId,
    published,
    authorId,
    search,
    limit = 20,
    offset = 0,
  } = params;

  const conditions: SQL[] = [
    eq(posts.organizationId, organizationId),
    isNull(posts.deletedAt),
  ];

  if (published !== undefined) {
    conditions.push(eq(posts.published, published));
  }

  if (authorId) {
    conditions.push(eq(posts.authorId, authorId));
  }

  if (search) {
    conditions.push(ilike(posts.title, `%${search}%`));
  }

  const [items, [{ total }]] = await Promise.all([
    db.query.posts.findMany({
      where: and(...conditions),
      orderBy: [desc(posts.createdAt)],
      limit,
      offset,
      with: {
        author: {
          columns: { id: true, name: true, avatarUrl: true },
        },
      },
    }),
    db
      .select({ total: count() })
      .from(posts)
      .where(and(...conditions)),
  ]);

  return {
    items,
    total,
    hasMore: offset + limit < total,
  };
}

export async function getPostBySlug(organizationId: string, slug: string) {
  return db.query.posts.findFirst({
    where: and(
      eq(posts.organizationId, organizationId),
      eq(posts.slug, slug),
      isNull(posts.deletedAt)
    ),
    with: {
      author: {
        columns: { id: true, name: true, avatarUrl: true },
      },
    },
  });
}
```

```typescript
// src/db/mutations/posts.ts
import { db } from '@/db';
import { posts, auditLogs } from '@/db/schema';
import { eq, and } from 'drizzle-orm';
import type { NewPost } from '@/db/schema';

export async function createPost(
  data: Omit<NewPost, 'id' | 'createdAt' | 'updatedAt'>,
  actorId: string
) {
  return db.transaction(async (tx) => {
    const [post] = await tx
      .insert(posts)
      .values(data)
      .returning();

    await tx.insert(auditLogs).values({
      action: 'post.created',
      entityType: 'post',
      entityId: post.id,
      actorId,
      organizationId: data.organizationId,
    });

    return post;
  });
}

export async function updatePost(
  postId: string,
  organizationId: string,
  data: Partial<Pick<NewPost, 'title' | 'slug' | 'content' | 'excerpt' | 'metadata'>>,
  actorId: string
) {
  return db.transaction(async (tx) => {
    const [post] = await tx
      .update(posts)
      .set(data)
      .where(
        and(
          eq(posts.id, postId),
          eq(posts.organizationId, organizationId)
        )
      )
      .returning();

    if (!post) throw new Error('Post not found');

    await tx.insert(auditLogs).values({
      action: 'post.updated',
      entityType: 'post',
      entityId: postId,
      actorId,
      organizationId,
      metadata: { updatedFields: Object.keys(data) },
    });

    return post;
  });
}

export async function softDeletePost(
  postId: string,
  organizationId: string,
  actorId: string
) {
  return db.transaction(async (tx) => {
    const [post] = await tx
      .update(posts)
      .set({ deletedAt: new Date() })
      .where(
        and(
          eq(posts.id, postId),
          eq(posts.organizationId, organizationId)
        )
      )
      .returning();

    if (!post) throw new Error('Post not found');

    await tx.insert(auditLogs).values({
      action: 'post.deleted',
      entityType: 'post',
      entityId: postId,
      actorId,
      organizationId,
    });

    return post;
  });
}
```

### The Server Component Layer

```typescript
// app/posts/page.tsx
import { listPosts } from '@/db/queries/posts';
import { auth } from '@/lib/auth';
import { redirect } from 'next/navigation';
import { PostList } from './post-list';
import { Pagination } from '@/components/pagination';

interface Props {
  searchParams: Promise<{
    page?: string;
    search?: string;
    published?: string;
  }>;
}

export default async function PostsPage({ searchParams }: Props) {
  const session = await auth();
  if (!session?.user) redirect('/login');

  const params = await searchParams;
  const page = parseInt(params.page || '1', 10);
  const limit = 20;

  const { items, total, hasMore } = await listPosts({
    organizationId: session.user.organizationId,
    search: params.search,
    published: params.published === 'true' ? true : params.published === 'false' ? false : undefined,
    limit,
    offset: (page - 1) * limit,
  });

  return (
    <div>
      <h1>Posts</h1>
      <PostList posts={items} />
      <Pagination
        currentPage={page}
        totalItems={total}
        itemsPerPage={limit}
        hasMore={hasMore}
      />
    </div>
  );
}
```

### The API Layer (for Mobile)

```typescript
// app/api/trpc/[trpc]/route.ts
// tRPC router that mobile clients call
import { fetchRequestHandler } from '@trpc/server/adapters/fetch';
import { appRouter } from '@/server/routers';
import { createContext } from '@/server/context';

const handler = (req: Request) =>
  fetchRequestHandler({
    endpoint: '/api/trpc',
    req,
    router: appRouter,
    createContext,
  });

export { handler as GET, handler as POST };
```

```typescript
// src/server/routers/posts.ts
import { router, protectedProcedure } from '../trpc';
import { z } from 'zod';
import { listPosts, getPostBySlug } from '@/db/queries/posts';
import { createPost, updatePost, softDeletePost } from '@/db/mutations/posts';

export const postsRouter = router({
  list: protectedProcedure
    .input(
      z.object({
        search: z.string().optional(),
        published: z.boolean().optional(),
        limit: z.number().min(1).max(100).default(20),
        offset: z.number().min(0).default(0),
      })
    )
    .query(async ({ input, ctx }) => {
      return listPosts({
        organizationId: ctx.session.user.organizationId,
        ...input,
      });
    }),

  bySlug: protectedProcedure
    .input(z.object({ slug: z.string() }))
    .query(async ({ input, ctx }) => {
      return getPostBySlug(ctx.session.user.organizationId, input.slug);
    }),

  create: protectedProcedure
    .input(
      z.object({
        title: z.string().min(1).max(500),
        slug: z.string().min(1).max(500),
        content: z.string().optional(),
        excerpt: z.string().optional(),
      })
    )
    .mutation(async ({ input, ctx }) => {
      return createPost(
        {
          ...input,
          authorId: ctx.session.user.id,
          organizationId: ctx.session.user.organizationId,
        },
        ctx.session.user.id
      );
    }),

  update: protectedProcedure
    .input(
      z.object({
        id: z.string().uuid(),
        title: z.string().min(1).max(500).optional(),
        slug: z.string().min(1).max(500).optional(),
        content: z.string().optional(),
        excerpt: z.string().optional(),
      })
    )
    .mutation(async ({ input, ctx }) => {
      const { id, ...data } = input;
      return updatePost(
        id,
        ctx.session.user.organizationId,
        data,
        ctx.session.user.id
      );
    }),

  delete: protectedProcedure
    .input(z.object({ id: z.string().uuid() }))
    .mutation(async ({ input, ctx }) => {
      return softDeletePost(
        input.id,
        ctx.session.user.organizationId,
        ctx.session.user.id
      );
    }),
});
```

### The Complete Flow

```
┌─────────────────────────────────────────────────────────────────┐
│              THE COMPLETE DATA ARCHITECTURE                      │
│                                                                  │
│  NEXT.JS WEB APP                                                │
│  ┌──────────────────────────────────────────────────────┐       │
│  │  Server Components ──→ db/queries/* ──→ Drizzle ──→ │       │
│  │  Server Actions    ──→ db/mutations/* ──→ Drizzle ──→│──→ Neon│
│  │  Route Handlers    ──→ db/queries/* ──→ Drizzle ──→ │  Postgres
│  │  tRPC Router       ──→ db/mutations/* ──→ Drizzle ──→│       │
│  └──────────────────────────────────────────────────────┘       │
│       ▲                          ▲                               │
│       │                          │                               │
│  Browser (SSR)              Mobile App                           │
│  Client Components       React Native                            │
│  (forms, interactions)    (via tRPC or REST API)                 │
│                                                                  │
│  SHARED:                                                         │
│  - db/schema/* (TypeScript source of truth)                     │
│  - db/queries/* (read operations)                               │
│  - db/mutations/* (write operations with audit logs)            │
│  - Zod schemas (validation, shared between web + mobile)        │
└─────────────────────────────────────────────────────────────────┘
```

---

## 15. INDEXING AND QUERY PERFORMANCE

You don't need to be a DBA, but you do need to know when queries are slow and how to fix them.

### When to Add an Index

```
RULE OF THUMB:
- Any column in a WHERE clause → probably needs an index
- Any column in a JOIN condition → probably needs an index
- Any column in an ORDER BY → consider an index
- Foreign key columns → always need an index (Drizzle doesn't auto-create these)
- Composite queries → consider a multi-column index
```

### Common Index Patterns

```typescript
// ── Single column index ──────────────────────────────────────
index('posts_author_id_idx').on(table.authorId),

// ── Composite index (for queries that filter on both columns) ─
index('posts_org_published_idx').on(table.organizationId, table.published),

// ── Unique index ─────────────────────────────────────────────
uniqueIndex('users_email_idx').on(table.email),

// ── Partial index (only indexes rows matching a condition) ────
// Great for soft deletes -- only index non-deleted rows
import { sql } from 'drizzle-orm';

// In raw migration SQL:
// CREATE INDEX posts_active_idx ON posts (organization_id, created_at)
// WHERE deleted_at IS NULL;
```

### Finding Slow Queries

In development, enable query logging:

```typescript
// Drizzle -- enable logging
export const db = drizzle(sql, {
  schema,
  logger: true, // Logs all SQL queries to console
});

// Or use a custom logger
export const db = drizzle(sql, {
  schema,
  logger: {
    logQuery(query: string, params: unknown[]) {
      console.log('SQL:', query);
      console.log('Params:', params);
    },
  },
});
```

In production, use Neon's query insights dashboard to find slow queries, then add indexes for the most common patterns.

### The Explain Escape Hatch

When a query is slow and you need to understand why:

```typescript
// Run EXPLAIN ANALYZE to see the query plan
const plan = await db.execute(sql`
  EXPLAIN ANALYZE
  SELECT p.*, u.name as author_name
  FROM posts p
  JOIN users u ON p.author_id = u.id
  WHERE p.organization_id = ${orgId}
  AND p.published = true
  ORDER BY p.published_at DESC
  LIMIT 20
`);

console.log(plan);
// Look for:
// - "Seq Scan" (full table scan -- probably needs an index)
// - "Index Scan" (good -- using an index)
// - High "actual time" values (bottleneck)
```

---

## 16. DATABASE TESTING

### Test Setup with a Separate Database

```typescript
// vitest.config.ts
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    env: {
      DATABASE_URL: process.env.TEST_DATABASE_URL || 'postgresql://localhost:5432/myapp_test',
    },
    globalSetup: './tests/global-setup.ts',
  },
});
```

```typescript
// tests/global-setup.ts
import { migrate } from 'drizzle-orm/neon-http/migrator';
import { drizzle } from 'drizzle-orm/neon-http';
import { neon } from '@neondatabase/serverless';

export async function setup() {
  const sql = neon(process.env.DATABASE_URL!);
  const db = drizzle(sql);

  // Run migrations on the test database
  await migrate(db, { migrationsFolder: './drizzle' });
}

export async function teardown() {
  // Optional: clean up
}
```

```typescript
// tests/helpers/db.ts
import { db } from '@/db';
import { users, organizations, memberships, posts, auditLogs } from '@/db/schema';

export async function cleanDatabase() {
  // Delete in reverse dependency order
  await db.delete(auditLogs);
  await db.delete(posts);
  await db.delete(memberships);
  await db.delete(organizations);
  await db.delete(users);
}

export async function createTestUser(overrides?: Partial<typeof users.$inferInsert>) {
  const [user] = await db
    .insert(users)
    .values({
      email: `test-${Date.now()}@example.com`,
      name: 'Test User',
      role: 'member',
      emailVerified: true,
      ...overrides,
    })
    .returning();
  return user;
}

export async function createTestOrg(ownerUserId: string) {
  const [org] = await db
    .insert(organizations)
    .values({
      name: 'Test Org',
      slug: `test-org-${Date.now()}`,
    })
    .returning();

  await db.insert(memberships).values({
    userId: ownerUserId,
    organizationId: org.id,
    role: 'owner',
  });

  return org;
}
```

```typescript
// tests/db/queries/posts.test.ts
import { describe, it, expect, beforeEach } from 'vitest';
import { listPosts, getPostBySlug } from '@/db/queries/posts';
import { createPost } from '@/db/mutations/posts';
import { cleanDatabase, createTestUser, createTestOrg } from '../helpers/db';

describe('Post queries', () => {
  let user: Awaited<ReturnType<typeof createTestUser>>;
  let org: Awaited<ReturnType<typeof createTestOrg>>;

  beforeEach(async () => {
    await cleanDatabase();
    user = await createTestUser();
    org = await createTestOrg(user.id);
  });

  it('lists published posts for an organization', async () => {
    // Create a published post
    await createPost(
      {
        title: 'Published Post',
        slug: 'published-post',
        content: 'Hello world',
        published: true,
        publishedAt: new Date(),
        authorId: user.id,
        organizationId: org.id,
      },
      user.id
    );

    // Create a draft post
    await createPost(
      {
        title: 'Draft Post',
        slug: 'draft-post',
        content: 'Not yet',
        published: false,
        authorId: user.id,
        organizationId: org.id,
      },
      user.id
    );

    const result = await listPosts({
      organizationId: org.id,
      published: true,
    });

    expect(result.items).toHaveLength(1);
    expect(result.items[0].title).toBe('Published Post');
    expect(result.total).toBe(1);
  });

  it('finds a post by slug', async () => {
    await createPost(
      {
        title: 'My Post',
        slug: 'my-post',
        content: 'Content here',
        published: true,
        publishedAt: new Date(),
        authorId: user.id,
        organizationId: org.id,
      },
      user.id
    );

    const post = await getPostBySlug(org.id, 'my-post');

    expect(post).not.toBeNull();
    expect(post!.title).toBe('My Post');
    expect(post!.author.name).toBe('Test User');
  });
});
```

---

## 17. EDGE CASES AND PRODUCTION GOTCHAS

### Gotcha 1: The `updatedAt` Trap

Drizzle's `$onUpdate` only fires when you use Drizzle to update. Direct SQL updates, migrations, and other tools bypass it. For reliable `updated_at` timestamps, use a database trigger:

```sql
-- Add this to a migration
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = NOW();
  RETURN NEW;
END;
$$ language 'plpgsql';

CREATE TRIGGER update_posts_updated_at
  BEFORE UPDATE ON posts
  FOR EACH ROW
  EXECUTE FUNCTION update_updated_at_column();

-- Repeat for every table with updated_at
CREATE TRIGGER update_users_updated_at
  BEFORE UPDATE ON users
  FOR EACH ROW
  EXECUTE FUNCTION update_updated_at_column();
```

### Gotcha 2: UUID Performance

UUIDs are random, which means inserts into B-tree indexes are random too. This can cause index fragmentation on very large tables. Solutions:

1. **UUIDv7:** Time-ordered UUIDs that are sortable and index-friendly. Use `uuid_generate_v7()` if your Postgres version supports it, or generate them in application code.
2. **CUID2:** Sortable, collision-resistant, and URL-friendly. Good for application-generated IDs.

```typescript
// Using UUIDv7 (requires uuid package or Postgres extension)
import { v7 as uuidv7 } from 'uuid';

id: uuid('id').primaryKey().$defaultFn(() => uuidv7()),
```

### Gotcha 3: Timezone Nightmares

```typescript
// ALWAYS use withTimezone: true
createdAt: timestamp('created_at', { withTimezone: true }).defaultNow().notNull(),

// NEVER do this
createdAt: timestamp('created_at').defaultNow().notNull(),
// Without timezone, you get "timestamp without timezone"
// which silently loses timezone information and causes bugs
// when your servers are in different timezones
```

### Gotcha 4: Forgetting to Filter by Organization

The most common security bug in multi-tenant applications:

```typescript
// BAD -- Returns posts from ALL organizations
const post = await db.query.posts.findFirst({
  where: eq(posts.id, postId),
});

// GOOD -- Scoped to the user's organization
const post = await db.query.posts.findFirst({
  where: and(
    eq(posts.id, postId),
    eq(posts.organizationId, session.user.organizationId)
  ),
});
```

Consider creating a wrapper that always includes the organization filter:

```typescript
// src/db/scoped.ts
import { db } from '@/db';
import { eq, and, SQL } from 'drizzle-orm';

export function scopedQuery(organizationId: string) {
  return {
    posts: {
      findMany: (options: Parameters<typeof db.query.posts.findMany>[0] = {}) =>
        db.query.posts.findMany({
          ...options,
          where: options.where
            ? and(eq(posts.organizationId, organizationId), options.where)
            : eq(posts.organizationId, organizationId),
        }),
      findFirst: (options: Parameters<typeof db.query.posts.findFirst>[0] = {}) =>
        db.query.posts.findFirst({
          ...options,
          where: options.where
            ? and(eq(posts.organizationId, organizationId), options.where)
            : eq(posts.organizationId, organizationId),
        }),
    },
  };
}

// Usage
const scoped = scopedQuery(session.user.organizationId);
const posts = await scoped.posts.findMany({
  where: eq(posts.published, true), // org filter is automatically added
});
```

### Gotcha 5: Cold Start + Connection Establishment

On Vercel, the first request to a serverless function may take:
- ~200ms for the function cold start
- ~100-300ms for TLS handshake to the database
- ~50ms for authentication
- ~30ms for the query itself

Total: ~400-600ms for the first request, ~30-50ms for subsequent requests.

Mitigations:
1. Use Neon's HTTP driver (skips TLS handshake for the connection)
2. Enable Vercel's Fluid Compute (keeps functions warm longer)
3. Use `use cache` to cache database query results
4. Use ISR (Incremental Static Regeneration) for pages that can tolerate stale data

---

## 18. THE DATABASE TOOLKIT CHECKLIST

Before shipping to production, verify:

```
┌─────────────────────────────────────────────────────────────────┐
│              PRODUCTION DATABASE CHECKLIST                        │
│                                                                  │
│  □ Connection pooling configured (pooled connection string)     │
│  □ Migrations tested on a branch/copy of production             │
│  □ All foreign key columns have indexes                         │
│  □ Soft deletes used for user-facing data                       │
│  □ All timestamps use withTimezone: true                        │
│  □ Organization ID filter on every query (multi-tenant)         │
│  □ Audit logs for sensitive operations                          │
│  □ Seed script for development and test environments            │
│  □ Database branching set up for preview deployments            │
│  □ Backup/restore strategy documented                           │
│  □ Query logging enabled in development                         │
│  □ No secrets in migration files                                │
│  □ Migration CI runs BEFORE deployment                          │
│  □ Database URL stored in environment variables (not code)      │
│  □ updatedAt triggers at database level (not just ORM)          │
│  □ Test suite has its own database (not shared with dev)        │
└─────────────────────────────────────────────────────────────────┘
```

---

## CHAPTER SUMMARY

The frontend architect's relationship with the database has fundamentally changed. In the Server Component era, you are not calling an API that someone else wrote. You are writing the queries. You are designing the schema. You are running the migrations.

Here is what matters:

1. **Start with Neon Postgres.** It is the right default. Autoscale to zero. Branching for preview environments. Built-in connection pooling. Generous free tier.

2. **Use Drizzle ORM.** SQL-like syntax that feels natural. Zero runtime overhead. Perfect TypeScript inference without codegen. If your team prefers schema-first design, Prisma is a strong second choice.

3. **Design your schema deliberately.** UUIDs for primary keys. Timestamps with timezone. Soft deletes for recoverable data. Organization ID on every tenant-scoped table.

4. **Treat migrations as a discipline.** Never delete columns in one step. Always add nullable first. Test on a database branch before production. Run migrations before deployment in CI.

5. **Pool your connections.** In serverless, every cold start opens a new connection. Use the Neon HTTP driver or pooled connection strings. Do not skip this.

6. **Query directly from Server Components.** No API layer needed for web. Add tRPC or REST routes only for mobile clients and external integrations.

7. **Cache at the framework level.** Use `use cache` or `unstable_cache` to cache database query results. Revalidate with tags when data changes.

8. **Test your data layer.** Separate test database. Clean state before each test. Test queries and mutations independently from UI components.

The database is no longer someone else's problem. It's yours. Own it.

---

### Further Reading

- [Drizzle ORM Documentation](https://orm.drizzle.team/docs/overview)
- [Prisma Documentation](https://www.prisma.io/docs)
- [Neon Documentation](https://neon.tech/docs)
- [Turso Documentation](https://docs.turso.tech)
- [Supabase Database Guide](https://supabase.com/docs/guides/database)
- [Next.js Data Fetching](https://nextjs.org/docs/app/building-your-application/data-fetching)
- [Vercel Postgres (Neon) Integration](https://vercel.com/docs/storage/vercel-postgres)
