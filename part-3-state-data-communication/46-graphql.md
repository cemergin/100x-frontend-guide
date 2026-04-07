<!--
  CHAPTER: 46
  TITLE: GraphQL — When, Why & How
  PART: III — State, Data & Communication
  PREREQS: Chapters 11, 34
  KEY_TOPICS: GraphQL, Apollo Client, urql, Relay, schema design, queries, mutations, subscriptions, fragments, codegen, normalized cache, pagination, optimistic updates, persisted queries, GraphQL vs REST vs tRPC
  DIFFICULTY: Intermediate → Advanced
  UPDATED: 2026-04-07
-->

# Chapter 46: GraphQL — When, Why & How

> **Part III — State, Data & Communication** | Prerequisites: Chapters 11, 34 | Difficulty: Intermediate to Advanced

<details>
<summary><strong>TL;DR</strong></summary>

- GraphQL shines when you have complex relational data, multiple client platforms needing different data shapes, and large teams with separate frontend/backend ownership; it is overkill for simple CRUD, small teams (use tRPC), or public APIs (use REST)
- Apollo Client remains the most feature-rich GraphQL client for React Native and Next.js, but urql is a lighter, more composable alternative when you do not need Apollo's normalized cache complexity
- GraphQL Code Generator eliminates runtime type ambiguity by generating TypeScript types, typed hooks, and document nodes from your schema and operations -- skip it and you lose the primary advantage of GraphQL's type system
- Relay-style cursor connections are the only pagination pattern that works reliably at scale; offset pagination in GraphQL is a trap that breaks under concurrent writes
- The "GraphQL tax" is real: N+1 server queries, schema maintenance overhead, tooling complexity, and a normalized cache that becomes a source of bugs around year two -- know the costs before you commit

</details>

Chapter 10 covered data fetching broadly and gave you a decision matrix: REST for simplicity, tRPC when you own both ends, GraphQL for complex data graphs with multiple clients. This chapter is the deep dive into GraphQL. Not the "here's what a query looks like" tutorial -- you can get that from the docs. This is the chapter about what happens after you choose GraphQL: the schema design decisions that haunt you, the client setup that actually works in production, the caching model that will either save or sabotage you, and the honest trade-offs that nobody mentions in conference talks.

Here is the thing about GraphQL that the marketing doesn't tell you: **it moves complexity, it doesn't eliminate it.** REST puts complexity on the client (multiple endpoints, over-fetching, orchestrating requests). GraphQL moves that complexity to the schema layer, the resolver layer, and the client cache. The total complexity is roughly the same. The question is whether the place GraphQL puts it is a better fit for your team, your data model, and your product.

This chapter will help you answer that question with clarity, and if the answer is yes, it will show you how to implement GraphQL properly in a React Native + Next.js monorepo with typed operations, normalized caching, real-time subscriptions, and pagination that doesn't fall apart.

### In This Chapter
- When GraphQL Makes Sense (and When It Doesn't)
- GraphQL Fundamentals: Schema, Types, Resolvers
- Schema Design for Mobile Apps
- Apollo Client: Complete Setup and Advanced Patterns
- urql: The Lighter Alternative
- GraphQL Code Generator: End-to-End Type Safety
- Fragment Colocation and Composition
- Cursor-Based Pagination with Relay Connections
- Real-Time Subscriptions
- Performance: Persisted Queries, Batching, Caching
- The Honest Trade-Offs
- Complete Monorepo Setup

### Related Chapters
- [Ch 10: Data Fetching & Server Communication] -- the broader data fetching context and protocol decision matrix
- [Ch 11: Caching Strategies] -- how GraphQL's normalized cache fits into the larger caching architecture
- [Ch 12: Offline-First & Real-Time] -- offline patterns that compose with GraphQL subscriptions
- [Ch 15: Monorepo Architecture] -- the monorepo setup that makes shared GraphQL code possible

---

## 1. WHEN GRAPHQL MAKES SENSE

Let's start with the decision that matters most: should you even use GraphQL?

The answer is not "always" and it is not "never." It depends on three things: your data model, your team structure, and your client diversity. Get this wrong and you will spend six months building a GraphQL layer that makes everything slower and more complicated.

### 1.1 GraphQL Thrives When...

**You have complex relational data.** Social feeds where a post has an author, comments, likes, shared media, and each comment has its own author and replies. Marketplaces where a listing has a seller, reviews, pricing tiers, shipping options, and related listings. CMS-backed apps where content has nested blocks, references, variants, and localized fields. If your data looks like a graph -- entities with relationships to other entities -- GraphQL is a natural fit because it lets clients traverse that graph in a single request.

```
// REST: 3-4 requests to render a social feed post
GET /posts/123
GET /users/456          (author)
GET /posts/123/comments?limit=3
GET /posts/123/likes/count

// GraphQL: 1 request, exactly the data you need
query PostDetail($id: ID!) {
  post(id: $id) {
    id
    body
    createdAt
    author {
      id
      name
      avatarUrl
    }
    comments(first: 3) {
      edges {
        node {
          id
          body
          author { name avatarUrl }
        }
      }
    }
    likesCount
    viewerHasLiked
  }
}
```

**You have multiple clients needing different data shapes.** A mobile app needs a compact post with a 140-character body preview and a thumbnail. A web app needs the full body with rich text and a high-res image. An admin dashboard needs the post plus moderation history and analytics. With REST, you either build separate endpoints for each client (BFF pattern), or you over-fetch for everyone and let clients throw away what they don't need. With GraphQL, each client queries exactly what it needs from the same schema.

**You have large teams with separate frontend/backend ownership.** When the frontend team and the backend team are different groups (or different companies), GraphQL's schema serves as a typed contract. The backend team publishes the schema. The frontend team writes queries against it. Neither team blocks the other. The schema is the API documentation, the type system, and the integration test surface all in one artifact.

**Your product evolves rapidly.** Adding a new field to a GraphQL schema is non-breaking. Clients that don't query the field don't know it exists. Clients that need it start querying it. No versioning. No new endpoint. No coordinated deploy. For a mobile app where you can't force-update every user, this forward-compatibility is gold.

### 1.2 GraphQL Hurts When...

**You have simple CRUD.** If your app is a todo list, a settings page, or a form-heavy admin tool where every screen maps 1:1 to a database table, GraphQL adds ceremony with no benefit. You'll write a schema, resolvers, queries, and codegen config to achieve what REST gives you with a URL and a JSON body. Use REST. Or better yet, use tRPC if you own both ends.

**You're a small team in a TypeScript monorepo.** If the same three developers write the frontend and the backend, and everything is TypeScript, tRPC gives you end-to-end type safety without a schema layer, without codegen, and without a client library. The types flow directly from your backend functions to your frontend calls. GraphQL's contract benefits only matter when the people on each side of the contract are different.

**You're building a public API.** REST with OpenAPI is the standard for public APIs. Every developer knows how to call a REST endpoint. GraphQL requires clients to understand the query language, handle the always-200 response model, and deal with the tooling. GitHub's GraphQL API is the exception that proves the rule -- and even GitHub still maintains a REST API alongside it.

**Your data is not relational.** If you're fetching time-series metrics, streaming binary data, or doing simple key-value lookups, GraphQL's graph traversal model adds nothing. Use the protocol that fits: REST for resources, SSE for streams, WebSocket for bidirectional, gRPC for service-to-service.

### 1.3 The Decision Matrix

```
┌─────────────────────────────────┬──────────┬──────────┬──────────┐
│ Factor                          │ REST     │ GraphQL  │ tRPC     │
├─────────────────────────────────┼──────────┼──────────┼──────────┤
│ Simple CRUD                     │ ★★★      │ ★        │ ★★★      │
│ Complex relational data         │ ★        │ ★★★      │ ★★       │
│ Multiple client platforms       │ ★★       │ ★★★      │ ★        │
│ Public API                      │ ★★★      │ ★        │ ✗        │
│ Small team, shared codebase     │ ★★       │ ★        │ ★★★      │
│ Large team, separate ownership  │ ★★       │ ★★★      │ ★        │
│ Rapid product iteration         │ ★★       │ ★★★      │ ★★★      │
│ Offline-first mobile            │ ★★       │ ★★       │ ★★       │
│ Real-time features              │ ★ (SSE)  │ ★★★      │ ★★       │
│ Caching simplicity              │ ★★★      │ ★        │ ★★★      │
│ Learning curve                  │ Low      │ High     │ Medium   │
│ Tooling maturity                │ Highest  │ High     │ Growing  │
└─────────────────────────────────┴──────────┴──────────┴──────────┘
```

The honest summary: **GraphQL earns its keep when your data is relational, your clients are diverse, and your team is large enough to benefit from the schema contract.** In a typical startup with a React Native app and a Next.js web app hitting the same backend, GraphQL becomes compelling once you have 15+ entities with relationships. Below that threshold, tRPC in a monorepo is faster to build and easier to maintain.

---

## 2. GRAPHQL FUNDAMENTALS

If you've decided GraphQL is the right choice, you need to understand the model deeply -- not just "it's like REST but with one endpoint." The mental model is fundamentally different.

### 2.1 The Schema Is the Contract

In REST, the API is defined by URLs and HTTP verbs. In GraphQL, the API is defined by a **schema** written in SDL (Schema Definition Language). The schema declares every type, every field, every argument, and every relationship in your API. It is the single source of truth.

```graphql
# schema.graphql

# --- Scalars ---
scalar DateTime
scalar URL

# --- Enums ---
enum PostStatus {
  DRAFT
  PUBLISHED
  ARCHIVED
}

enum SortOrder {
  NEWEST
  OLDEST
  MOST_LIKED
}

# --- Core Types ---
type User {
  id: ID!
  username: String!
  displayName: String!
  avatarUrl: URL
  bio: String
  createdAt: DateTime!
  posts(first: Int, after: String): PostConnection!
  followers(first: Int, after: String): UserConnection!
  following(first: Int, after: String): UserConnection!
  followersCount: Int!
  followingCount: Int!
}

type Post {
  id: ID!
  body: String!
  status: PostStatus!
  author: User!
  media: [Media!]!
  comments(first: Int, after: String): CommentConnection!
  likesCount: Int!
  commentsCount: Int!
  viewerHasLiked: Boolean!
  createdAt: DateTime!
  updatedAt: DateTime!
}

type Comment {
  id: ID!
  body: String!
  author: User!
  post: Post!
  parentComment: Comment
  replies(first: Int, after: String): CommentConnection!
  likesCount: Int!
  createdAt: DateTime!
}

union Media = Image | Video

type Image {
  id: ID!
  url: URL!
  width: Int!
  height: Int!
  altText: String
  blurhash: String
}

type Video {
  id: ID!
  url: URL!
  thumbnailUrl: URL!
  duration: Int!
  width: Int!
  height: Int!
}

# --- Connections (Relay-style pagination) ---
type PostConnection {
  edges: [PostEdge!]!
  pageInfo: PageInfo!
  totalCount: Int!
}

type PostEdge {
  node: Post!
  cursor: String!
}

type CommentConnection {
  edges: [CommentEdge!]!
  pageInfo: PageInfo!
  totalCount: Int!
}

type CommentEdge {
  node: Comment!
  cursor: String!
}

type UserConnection {
  edges: [UserEdge!]!
  pageInfo: PageInfo!
  totalCount: Int!
}

type UserEdge {
  node: User!
  cursor: String!
}

type PageInfo {
  hasNextPage: Boolean!
  hasPreviousPage: Boolean!
  startCursor: String
  endCursor: String
}

# --- Root Types ---
type Query {
  # Single entity lookups
  viewer: User!
  user(id: ID!): User
  post(id: ID!): Post

  # Feeds
  feed(first: Int!, after: String, sort: SortOrder): PostConnection!
  explore(first: Int!, after: String): PostConnection!
  search(query: String!, first: Int!, after: String): SearchResultConnection!
}

type Mutation {
  # Posts
  createPost(input: CreatePostInput!): CreatePostPayload!
  updatePost(input: UpdatePostInput!): UpdatePostPayload!
  deletePost(id: ID!): DeletePostPayload!

  # Interactions
  likePost(postId: ID!): LikePostPayload!
  unlikePost(postId: ID!): UnlikePostPayload!
  createComment(input: CreateCommentInput!): CreateCommentPayload!
  deleteComment(id: ID!): DeleteCommentPayload!

  # User
  updateProfile(input: UpdateProfileInput!): UpdateProfilePayload!
  followUser(userId: ID!): FollowUserPayload!
  unfollowUser(userId: ID!): UnfollowUserPayload!
}

type Subscription {
  # Real-time updates
  postLiked(postId: ID!): Post!
  commentAdded(postId: ID!): Comment!
  newFeedPost: Post!
}
```

Notice what this schema gives you that a REST API doesn't: **the entire data model is visible in one file.** Every relationship, every argument, every nullable field. A frontend developer can read this and know exactly what data is available without hitting a single endpoint or reading a separate documentation site.

### 2.2 The Type System Advantage

GraphQL's type system is not just documentation -- it's a machine-readable contract that powers tooling. This is the thing that makes GraphQL worthwhile despite the complexity.

The schema tells your toolchain:
- **What types exist** -- so codegen can generate TypeScript interfaces
- **What fields are nullable** -- so your code handles nulls correctly at compile time
- **What arguments are required** -- so your queries fail at build time, not runtime
- **What relationships exist** -- so your cache can normalize data across queries

```typescript
// WITHOUT codegen -- the GraphQL "dark ages"
const { data } = useQuery(GET_POST);
// data is `any`. You're guessing. Every field access is a runtime prayer.
const authorName = data?.post?.author?.name; // string? null? undefined? Who knows.

// WITH codegen -- the GraphQL advantage
const { data } = usePostDetailQuery({ variables: { id: '123' } });
// data is PostDetailQuery -- fully typed.
// data.post.author.name is `string` -- guaranteed by the schema.
// data.post.media is `(Image | Video)[]` -- union types work.
// If you query a field that doesn't exist, TypeScript catches it at compile time.
```

This is why Section 6 on codegen is not optional. Without codegen, GraphQL is worse than REST because you have all the complexity with none of the type safety.

### 2.3 Resolvers: Where the Data Comes From

On the server side, every field in the schema has a **resolver** -- a function that returns the data for that field. Understanding resolvers matters for frontend developers because the performance of your queries depends on how resolvers are implemented.

```typescript
// Server-side resolver map (simplified)
const resolvers = {
  Query: {
    viewer: (_, __, { user }) => user,
    post: (_, { id }, { dataSources }) => dataSources.posts.getById(id),
    feed: (_, { first, after, sort }, { user, dataSources }) =>
      dataSources.posts.getFeed({ userId: user.id, first, after, sort }),
  },

  Post: {
    // This resolver runs for every Post in a list.
    // If your feed returns 20 posts, this runs 20 times.
    // Without DataLoader, that's 20 database queries. (The N+1 problem.)
    author: (post, _, { dataSources }) =>
      dataSources.users.getById(post.authorId),

    comments: (post, { first, after }, { dataSources }) =>
      dataSources.comments.getByPostId({ postId: post.id, first, after }),

    viewerHasLiked: (post, _, { user, dataSources }) =>
      dataSources.likes.hasLiked({ userId: user.id, postId: post.id }),
  },

  Media: {
    // Union type resolver -- tells GraphQL which concrete type to use
    __resolveType: (media) => {
      if (media.duration !== undefined) return 'Video';
      return 'Image';
    },
  },
};
```

The key insight for frontend developers: **every field you add to a query potentially triggers a resolver on the server.** Querying `post.author.posts.edges.node.author` is legal but means the server resolves `author` -> `posts` -> `author` in a chain. Be thoughtful about query depth. We'll come back to this in the performance section.

### 2.4 Queries, Mutations, and Subscriptions

GraphQL has three operation types:

**Queries** are reads. They are expected to be side-effect-free. Multiple fields in a query can resolve in parallel on the server.

```graphql
query FeedScreen($first: Int!, $after: String) {
  feed(first: $first, after: $after) {
    edges {
      node {
        id
        body
        createdAt
        author {
          id
          displayName
          avatarUrl
        }
        likesCount
        commentsCount
        viewerHasLiked
      }
    }
    pageInfo {
      hasNextPage
      endCursor
    }
  }
}
```

**Mutations** are writes. They execute sequentially -- if you send two mutations in one request, the second waits for the first. They return the mutated data so the client can update its cache.

```graphql
mutation LikePost($postId: ID!) {
  likePost(postId: $postId) {
    post {
      id
      likesCount
      viewerHasLiked
    }
  }
}
```

**Subscriptions** are real-time. The client opens a persistent connection (usually WebSocket) and receives updates when the subscribed event occurs.

```graphql
subscription OnCommentAdded($postId: ID!) {
  commentAdded(postId: $postId) {
    id
    body
    author {
      id
      displayName
      avatarUrl
    }
    createdAt
  }
}
```

---

## 3. SCHEMA DESIGN FOR MOBILE APPS

Schema design is where most GraphQL projects go wrong. A bad schema creates a bad developer experience that no client library can fix. Here are the patterns that work in production.

### 3.1 Input Types for Mutations

Never use individual arguments for mutations. Always use a single `input` argument with a dedicated input type. This makes mutations extensible without breaking changes.

```graphql
# BAD: Individual arguments -- adding a field is a breaking change
type Mutation {
  createPost(body: String!, mediaIds: [ID!]): Post!
}

# GOOD: Input type -- adding a field is non-breaking
input CreatePostInput {
  body: String!
  mediaIds: [ID!]
  status: PostStatus = DRAFT
  scheduledAt: DateTime
  replyToId: ID
}

type Mutation {
  createPost(input: CreatePostInput!): CreatePostPayload!
}
```

### 3.2 Mutation Payloads with Error Types

Never return the raw entity from a mutation. Return a payload type that can express both success and domain-level errors.

```graphql
# The payload pattern: every mutation returns a union of success + error types

type CreatePostPayload {
  post: Post
  errors: [CreatePostError!]!
}

type CreatePostError {
  field: String
  message: String!
  code: CreatePostErrorCode!
}

enum CreatePostErrorCode {
  BODY_TOO_LONG
  BODY_EMPTY
  MEDIA_NOT_FOUND
  RATE_LIMITED
  FORBIDDEN
}

# Usage in the client:
# mutation CreatePost($input: CreatePostInput!) {
#   createPost(input: $input) {
#     post {
#       id
#       body
#     }
#     errors {
#       field
#       message
#       code
#     }
#   }
# }
```

This pattern lets you handle domain errors in the GraphQL response instead of relying on HTTP status codes (which are always 200 in GraphQL) or throwing GraphQL errors (which are for infrastructure failures, not business logic).

### 3.3 Union Types for Polymorphic Data

Real-world feeds contain mixed content. Use union types to model this.

```graphql
union FeedItem = Post | SharedPost | Poll | Announcement

type SharedPost {
  id: ID!
  sharedBy: User!
  originalPost: Post!
  comment: String
  createdAt: DateTime!
}

type Poll {
  id: ID!
  question: String!
  options: [PollOption!]!
  author: User!
  expiresAt: DateTime
  viewerVotedOption: ID
  createdAt: DateTime!
}

type PollOption {
  id: ID!
  text: String!
  voteCount: Int!
}

type Announcement {
  id: ID!
  title: String!
  body: String!
  priority: AnnouncementPriority!
  createdAt: DateTime!
}

# Querying union types uses inline fragments
# query Feed {
#   feed(first: 20) {
#     edges {
#       node {
#         __typename
#         ... on Post {
#           id
#           body
#           author { displayName }
#         }
#         ... on SharedPost {
#           id
#           sharedBy { displayName }
#           originalPost { id body }
#         }
#         ... on Poll {
#           id
#           question
#           options { id text voteCount }
#         }
#         ... on Announcement {
#           id
#           title
#           body
#         }
#       }
#     }
#   }
# }
```

### 3.4 Relay-Style Connections for Pagination

The Relay connection spec is not just for Relay users. It is the standard pattern for paginated data in GraphQL because it solves real problems:

1. **Cursor-based pagination is stable under concurrent writes.** Offset pagination breaks when items are inserted or deleted during pagination.
2. **`PageInfo` gives the client everything it needs** to render "load more" buttons and infinite scroll without guessing.
3. **`totalCount` is separate from the page** so the client can show "237 comments" without loading all of them.
4. **Edges can carry edge-specific data** (like the relationship metadata between two entities).

```graphql
# The Relay connection pattern:
type PostConnection {
  edges: [PostEdge!]!       # The items, wrapped in edges
  pageInfo: PageInfo!        # Pagination metadata
  totalCount: Int!           # Total items (optional but useful)
}

type PostEdge {
  node: Post!               # The actual item
  cursor: String!           # Opaque cursor for this position
}

type PageInfo {
  hasNextPage: Boolean!     # Are there more items after the last edge?
  hasPreviousPage: Boolean! # Are there more items before the first edge?
  startCursor: String       # Cursor of the first edge
  endCursor: String         # Cursor of the last edge
}
```

The `first`/`after` pattern for forward pagination:
- `first: 20` -- give me the first 20 items
- `after: "cursor_xyz"` -- starting after this cursor
- When `pageInfo.hasNextPage` is false, you've reached the end

We'll build the full client-side pagination implementation in Section 8.

### 3.5 Viewer Pattern

For authenticated data, use a `viewer` field that represents the currently logged-in user. This avoids passing user IDs around and makes authorization implicit.

```graphql
type Query {
  # The authenticated user's view of the world
  viewer: User!

  # Fields on the viewer that represent "my" data
  # (You can also put these directly on the User type
  # and access them via viewer.notifications, etc.)
}

type User {
  # Public fields
  id: ID!
  username: String!
  displayName: String!

  # Viewer-specific fields (only meaningful for the authenticated user)
  email: String              # Only visible to the viewer themselves
  notifications(first: Int, after: String): NotificationConnection!
  savedPosts(first: Int, after: String): PostConnection!
  blockedUsers: [User!]!
}
```

### 3.6 Schema Design Checklist

Before shipping a schema to production, run through this checklist:

```
□ Every mutation uses an input type, not individual arguments
□ Every mutation returns a payload type with optional errors
□ Paginated fields use Relay-style connections (edges/node/pageInfo)
□ Nullable fields are intentionally nullable (not "I forgot the !")
□ Union types have __resolveType on the server
□ Custom scalars (DateTime, URL, etc.) are defined and validated
□ The viewer pattern is used for authenticated data
□ No field returns an unbounded list (always paginated)
□ Enum values use SCREAMING_SNAKE_CASE
□ Field names use camelCase
□ Type names use PascalCase
□ Deprecated fields have @deprecated(reason: "...") directives
```

---

## 4. APOLLO CLIENT

Apollo Client is the most widely used GraphQL client in the React ecosystem. It's feature-rich, well-documented, and battle-tested. It is also complex, opinionated about caching, and can be frustrating when its normalized cache does something you don't expect. This section is the guide to using it well.

### 4.1 Installation and Setup

```bash
# Core packages
npm install @apollo/client graphql

# For React Native WebSocket subscriptions
npm install graphql-ws

# For codegen (covered in Section 6)
npm install -D @graphql-codegen/cli @graphql-codegen/client-preset
```

### 4.2 Apollo Client Configuration

Here is a production-ready Apollo Client setup for a React Native + Next.js monorepo. This goes in your shared packages.

```typescript
// packages/graphql/src/client.ts

import {
  ApolloClient,
  InMemoryCache,
  ApolloLink,
  HttpLink,
  split,
  from,
  NormalizedCacheObject,
} from '@apollo/client';
import { GraphQLWsLink } from '@apollo/client/link/subscriptions';
import { getMainDefinition } from '@apollo/client/utilities';
import { onError } from '@apollo/client/link/error';
import { RetryLink } from '@apollo/client/link/retry';
import { createClient as createWsClient } from 'graphql-ws';
import { typePolicies } from './cache/type-policies';

// --- Types ---

interface ClientConfig {
  httpUrl: string;
  wsUrl: string;
  getToken: () => Promise<string | null>;
  onAuthError: () => void;
  platform: 'web' | 'mobile';
}

// --- Error Link ---

function createErrorLink(onAuthError: () => void): ApolloLink {
  return onError(({ graphQLErrors, networkError, operation }) => {
    if (graphQLErrors) {
      for (const error of graphQLErrors) {
        console.error(
          `[GraphQL Error] ${error.message}`,
          `Operation: ${operation.operationName}`,
          `Path: ${error.path?.join('.')}`,
        );

        // Handle authentication errors
        if (
          error.extensions?.code === 'UNAUTHENTICATED' ||
          error.extensions?.code === 'FORBIDDEN'
        ) {
          onAuthError();
        }
      }
    }

    if (networkError) {
      console.error(`[Network Error] ${networkError.message}`);
    }
  });
}

// --- Auth Link ---

function createAuthLink(getToken: () => Promise<string | null>): ApolloLink {
  return new ApolloLink((operation, forward) => {
    return forward(operation).map((response) => response);
  });
}

// Use setContext for async token retrieval
import { setContext } from '@apollo/client/link/context';

function createAuthContextLink(
  getToken: () => Promise<string | null>,
): ApolloLink {
  return setContext(async (_, { headers }) => {
    const token = await getToken();
    return {
      headers: {
        ...headers,
        ...(token ? { Authorization: `Bearer ${token}` } : {}),
      },
    };
  });
}

// --- Retry Link ---

const retryLink = new RetryLink({
  delay: {
    initial: 300,
    max: 10_000,
    jitter: true,
  },
  attempts: {
    max: 3,
    retryIf: (error) => {
      // Only retry network errors, not GraphQL errors
      return !!error && !error.statusCode;
    },
  },
});

// --- Create Client ---

export function createApolloClient(config: ClientConfig): ApolloClient<NormalizedCacheObject> {
  const { httpUrl, wsUrl, getToken, onAuthError, platform } = config;

  // HTTP link for queries and mutations
  const httpLink = new HttpLink({
    uri: httpUrl,
    // Use GET for queries (enables CDN caching with persisted queries)
    useGETForQueries: false, // Enable when using persisted queries
  });

  // WebSocket link for subscriptions
  const wsLink = new GraphQLWsLink(
    createWsClient({
      url: wsUrl,
      connectionParams: async () => {
        const token = await getToken();
        return { authorization: token ? `Bearer ${token}` : '' };
      },
      // Reconnection configuration
      retryAttempts: Infinity,
      shouldRetry: () => true,
      retryWait: async (retryCount) => {
        // Exponential backoff with jitter
        const delay = Math.min(1000 * Math.pow(2, retryCount), 30_000);
        const jitter = delay * 0.2 * Math.random();
        await new Promise((resolve) => setTimeout(resolve, delay + jitter));
      },
      // Keep-alive
      keepAlive: 10_000,
    }),
  );

  // Split traffic: subscriptions go to WebSocket, everything else to HTTP
  const transportLink = split(
    ({ query }) => {
      const definition = getMainDefinition(query);
      return (
        definition.kind === 'OperationDefinition' &&
        definition.operation === 'subscription'
      );
    },
    wsLink,
    httpLink,
  );

  // Compose the link chain
  const link = from([
    retryLink,
    createErrorLink(onAuthError),
    createAuthContextLink(getToken),
    transportLink,
  ]);

  // Create the cache
  const cache = new InMemoryCache({
    typePolicies,
    // Possible types for union/interface types (generated by codegen)
    possibleTypes: {
      Media: ['Image', 'Video'],
      FeedItem: ['Post', 'SharedPost', 'Poll', 'Announcement'],
      SearchResult: ['User', 'Post', 'Tag'],
    },
  });

  return new ApolloClient({
    link,
    cache,
    defaultOptions: {
      watchQuery: {
        // 'cache-and-network' shows cached data immediately, then updates from network
        fetchPolicy: platform === 'mobile' ? 'cache-and-network' : 'cache-first',
        // Return partial data from cache while missing fields are loading
        returnPartialData: false,
        // How long to keep data "fresh" before next fetch
        nextFetchPolicy: 'cache-first',
      },
      query: {
        fetchPolicy: 'cache-first',
        errorPolicy: 'all', // Return both data and errors
      },
      mutate: {
        errorPolicy: 'all',
      },
    },
    // Only enable devtools in development
    connectToDevTools: __DEV__,
  });
}
```

### 4.3 Type Policies and Cache Normalization

Apollo's normalized cache is both its greatest strength and its most common source of bugs. Understanding it is non-negotiable.

**How normalization works:** When Apollo receives a query response, it doesn't store it as a blob. It splits the response into individual objects, identifies them by `__typename:id`, and stores them in a flat lookup table. If two queries return the same `Post` with `id: "123"`, they share the same cache entry. Update one and the other updates automatically.

```
// Query response:
{
  feed: {
    edges: [
      {
        node: {
          __typename: "Post",
          id: "post_1",
          body: "Hello world",
          author: {
            __typename: "User",
            id: "user_1",
            displayName: "Alice"
          },
          likesCount: 5
        }
      }
    ]
  }
}

// Apollo stores this as:
{
  "Post:post_1": {
    __typename: "Post",
    id: "post_1",
    body: "Hello world",
    author: { __ref: "User:user_1" },
    likesCount: 5
  },
  "User:user_1": {
    __typename: "User",
    id: "user_1",
    displayName: "Alice"
  },
  ROOT_QUERY: {
    "feed({\"first\":20})": {
      edges: [
        { node: { __ref: "Post:post_1" } }
      ]
    }
  }
}
```

This normalization means that if you `likePost` and the mutation returns `{ post: { id: "post_1", likesCount: 6 } }`, every query showing `post_1` automatically updates to show 6 likes. No cache invalidation needed. No refetching. This is the magic.

But the magic breaks if:
- Objects don't have `id` fields (Apollo can't normalize them)
- You have custom cache keys that don't match
- Paginated lists need manual merge logic
- Two fields return the same type with different filters

Type policies solve these problems:

```typescript
// packages/graphql/src/cache/type-policies.ts

import { TypePolicies, FieldPolicy, Reference } from '@apollo/client';

// --- Helper: Relay-style connection merge ---

type KeyArgs = FieldPolicy['keyArgs'];

function relayConnectionMerge(keyArgs: KeyArgs = false): FieldPolicy {
  return {
    keyArgs,
    merge(existing, incoming, { args, readField }) {
      if (!incoming) return existing;
      if (!existing || !args?.after) {
        // First page or fresh fetch -- replace entirely
        return incoming;
      }

      // Subsequent pages -- merge edges, update pageInfo
      const existingEdges = existing.edges ?? [];
      const incomingEdges = incoming.edges ?? [];

      // Deduplicate by cursor to avoid duplicates on re-fetch
      const existingCursors = new Set(
        existingEdges.map((edge: any) => readField('cursor', edge)),
      );
      const newEdges = incomingEdges.filter(
        (edge: any) => !existingCursors.has(readField('cursor', edge)),
      );

      return {
        ...incoming,
        edges: [...existingEdges, ...newEdges],
        // Always use the latest pageInfo from the server
        pageInfo: incoming.pageInfo,
      };
    },
  };
}

// --- Type Policies ---

export const typePolicies: TypePolicies = {
  Query: {
    fields: {
      // Feed pagination: merge pages, key by sort order
      feed: relayConnectionMerge(['sort']),

      // Explore pagination: merge pages, no key args
      explore: relayConnectionMerge(),

      // Search: key by query string, merge pages
      search: relayConnectionMerge(['query']),

      // Single entity lookups: use incoming data
      post: {
        read(_, { args, toReference }) {
          // Allow reading a post from cache by ID without a network request
          return toReference({ __typename: 'Post', id: args?.id });
        },
      },

      user: {
        read(_, { args, toReference }) {
          return toReference({ __typename: 'User', id: args?.id });
        },
      },
    },
  },

  Post: {
    fields: {
      comments: relayConnectionMerge(),
      // Ensure media union types are handled
      media: {
        merge: false, // Always replace, don't merge arrays
      },
    },
  },

  User: {
    fields: {
      posts: relayConnectionMerge(),
      followers: relayConnectionMerge(),
      following: relayConnectionMerge(),
    },
  },

  // Types without ID fields need keyFields
  PageInfo: {
    keyFields: false, // Embedded object, not a standalone entity
  },

  // Edge types are identified by their cursor, not by an ID
  PostEdge: {
    keyFields: false,
  },
  CommentEdge: {
    keyFields: false,
  },
  UserEdge: {
    keyFields: false,
  },
};
```

### 4.4 React Hooks: useQuery, useMutation, useSubscription

With the client configured, here's how you use it in components. These examples assume codegen is set up (Section 6), so hooks and types are generated.

**useQuery: Fetching Data**

```typescript
// screens/FeedScreen.tsx

import { useCallback } from 'react';
import { FlatList, RefreshControl } from 'react-native';
import { useFeedQuery } from '@acme/graphql/generated';
import { PostCard } from '../components/PostCard';

const PAGE_SIZE = 20;

export function FeedScreen() {
  const { data, loading, error, fetchMore, refetch, networkStatus } = useFeedQuery({
    variables: { first: PAGE_SIZE },
    // Show cached data while fetching fresh data from network
    fetchPolicy: 'cache-and-network',
    // networkStatus 4 = refetch, 3 = fetchMore
    notifyOnNetworkStatusChange: true,
  });

  const isRefreshing = networkStatus === 4;
  const isLoadingMore = networkStatus === 3;

  const handleLoadMore = useCallback(() => {
    const pageInfo = data?.feed.pageInfo;
    if (!pageInfo?.hasNextPage || isLoadingMore) return;

    fetchMore({
      variables: {
        after: pageInfo.endCursor,
      },
      // No updateQuery needed -- type policies handle the merge
    });
  }, [data?.feed.pageInfo, fetchMore, isLoadingMore]);

  if (loading && !data) {
    return <FeedSkeleton />;
  }

  if (error && !data) {
    return <ErrorScreen error={error} onRetry={refetch} />;
  }

  const posts = data?.feed.edges.map((edge) => edge.node) ?? [];

  return (
    <FlatList
      data={posts}
      keyExtractor={(post) => post.id}
      renderItem={({ item }) => <PostCard post={item} />}
      onEndReached={handleLoadMore}
      onEndReachedThreshold={0.5}
      refreshControl={
        <RefreshControl refreshing={isRefreshing} onRefresh={refetch} />
      }
      ListFooterComponent={isLoadingMore ? <LoadingMore /> : null}
    />
  );
}
```

**useMutation: Writing Data with Optimistic Updates**

```typescript
// hooks/useLikePost.ts

import { useLikePostMutation, useUnlikePostMutation } from '@acme/graphql/generated';

export function useLikePost() {
  const [likePost] = useLikePostMutation();
  const [unlikePost] = useUnlikePostMutation();

  const toggleLike = useCallback(
    async (postId: string, isCurrentlyLiked: boolean, currentLikesCount: number) => {
      const mutation = isCurrentlyLiked ? unlikePost : likePost;
      const newLikesCount = isCurrentlyLiked
        ? currentLikesCount - 1
        : currentLikesCount + 1;

      await mutation({
        variables: { postId },
        // Optimistic update: update the UI immediately
        optimisticResponse: {
          __typename: 'Mutation',
          [isCurrentlyLiked ? 'unlikePost' : 'likePost']: {
            __typename: isCurrentlyLiked ? 'UnlikePostPayload' : 'LikePostPayload',
            post: {
              __typename: 'Post',
              id: postId,
              likesCount: newLikesCount,
              viewerHasLiked: !isCurrentlyLiked,
            },
          },
        },
        // The mutation response includes the post fields,
        // so Apollo automatically updates the normalized cache.
        // No manual cache.writeQuery needed.
      });
    },
    [likePost, unlikePost],
  );

  return { toggleLike };
}
```

**useMutation: Creating Entities and Updating Lists**

When you create an entity, you need to manually add it to cached lists because Apollo doesn't know which lists should include the new item.

```typescript
// hooks/useCreatePost.ts

import { useCreatePostMutation, FeedDocument, FeedQuery } from '@acme/graphql/generated';

export function useCreatePost() {
  const [createPost, { loading }] = useCreatePostMutation({
    // Update the feed cache after successful creation
    update(cache, { data }) {
      if (!data?.createPost.post) return;

      const newPost = data.createPost.post;

      // Read the current feed from cache
      const existingFeed = cache.readQuery<FeedQuery>({
        query: FeedDocument,
        variables: { first: 20 },
      });

      if (!existingFeed) return;

      // Prepend the new post to the feed
      cache.writeQuery<FeedQuery>({
        query: FeedDocument,
        variables: { first: 20 },
        data: {
          ...existingFeed,
          feed: {
            ...existingFeed.feed,
            totalCount: existingFeed.feed.totalCount + 1,
            edges: [
              {
                __typename: 'PostEdge',
                cursor: newPost.id, // Temporary cursor
                node: newPost,
              },
              ...existingFeed.feed.edges,
            ],
          },
        },
      });
    },
    // After the mutation, also refetch the feed to get the real cursor
    // and ensure consistency
    refetchQueries: ['Feed'],
    awaitRefetchQueries: false, // Don't wait -- user sees instant update
  });

  return { createPost, loading };
}
```

### 4.5 Fetch Policies Explained

Apollo's fetch policies determine where data comes from. Choosing the right one for each query is crucial for perceived performance.

```
┌─────────────────────┬────────────────────────────────────────────────┐
│ Policy              │ Behavior                                       │
├─────────────────────┼────────────────────────────────────────────────┤
│ cache-first         │ Read from cache. Only hit network on cache     │
│ (default)           │ miss. Best for data that rarely changes.       │
├─────────────────────┼────────────────────────────────────────────────┤
│ cache-and-network   │ Return cached data immediately, THEN fetch     │
│                     │ from network and update. Best for feeds and    │
│                     │ lists where freshness matters but instant      │
│                     │ display matters more.                          │
├─────────────────────┼────────────────────────────────────────────────┤
│ network-only        │ Always fetch from network. Ignore cache on     │
│                     │ read, but write response to cache. Use for     │
│                     │ data that MUST be fresh (payment status).      │
├─────────────────────┼────────────────────────────────────────────────┤
│ cache-only          │ Only read from cache. Never hit network.       │
│                     │ Throws if data not in cache. Rare -- used      │
│                     │ for truly offline scenarios.                   │
├─────────────────────┼────────────────────────────────────────────────┤
│ no-cache            │ Always fetch from network. Don't write to      │
│                     │ cache. Use for one-off queries where caching   │
│                     │ would be wasteful (search autocomplete).       │
├─────────────────────┼────────────────────────────────────────────────┤
│ standby             │ Like cache-first, but doesn't automatically    │
│                     │ update when the cache changes. Manual control. │
└─────────────────────┴────────────────────────────────────────────────┘
```

**The recommended pattern for mobile apps:**

```typescript
// Data that changes frequently: show cached, then update
const { data } = useFeedQuery({
  fetchPolicy: 'cache-and-network',
});

// Data that rarely changes: trust the cache
const { data } = useUserProfileQuery({
  fetchPolicy: 'cache-first',
  variables: { id: userId },
});

// Data that must be fresh: always fetch
const { data } = usePaymentStatusQuery({
  fetchPolicy: 'network-only',
  variables: { orderId },
});

// Data for prefetching (e.g., on hover/focus)
client.query({
  query: PostDetailDocument,
  variables: { id: postId },
  fetchPolicy: 'cache-first', // Don't refetch if we already have it
});
```

### 4.6 The ApolloProvider Setup

```typescript
// apps/mobile/src/providers/ApolloProvider.tsx

import React, { useMemo } from 'react';
import { ApolloProvider as BaseApolloProvider } from '@apollo/client';
import { createApolloClient } from '@acme/graphql';
import { useAuth } from '../hooks/useAuth';

const API_URL = process.env.EXPO_PUBLIC_API_URL ?? 'http://localhost:4000';
const WS_URL = API_URL.replace(/^http/, 'ws') + '/graphql';

export function ApolloProvider({ children }: { children: React.ReactNode }) {
  const { getToken, signOut } = useAuth();

  const client = useMemo(
    () =>
      createApolloClient({
        httpUrl: `${API_URL}/graphql`,
        wsUrl: WS_URL,
        getToken,
        onAuthError: signOut,
        platform: 'mobile',
      }),
    [getToken, signOut],
  );

  return <BaseApolloProvider client={client}>{children}</BaseApolloProvider>;
}
```

```typescript
// apps/web/src/app/providers.tsx (Next.js App Router)

'use client';

import React, { useMemo } from 'react';
import { ApolloProvider as BaseApolloProvider } from '@apollo/client';
import { createApolloClient } from '@acme/graphql';
import { useAuth } from '@clerk/nextjs';

export function ApolloProvider({ children }: { children: React.ReactNode }) {
  const { getToken, signOut } = useAuth();

  const client = useMemo(
    () =>
      createApolloClient({
        httpUrl: process.env.NEXT_PUBLIC_API_URL + '/graphql',
        wsUrl: process.env.NEXT_PUBLIC_WS_URL + '/graphql',
        getToken: () => getToken({ template: 'graphql' }),
        onAuthError: signOut,
        platform: 'web',
      }),
    [getToken, signOut],
  );

  return <BaseApolloProvider client={client}>{children}</BaseApolloProvider>;
}
```

---

## 5. URQL: THE LIGHTER ALTERNATIVE

Apollo Client is comprehensive but heavy. If you don't need normalized caching, Apollo's ~45KB gzipped bundle and complex type policies might be more than you need. Enter urql.

### 5.1 When to Pick urql Over Apollo

```
Choose urql when:
├── You want a smaller bundle (~14KB gzipped vs ~45KB for Apollo)
├── Document caching (by query + variables) is sufficient
├── You don't need fine-grained cache normalization
├── You value composability over built-in features
├── You want clearer, more predictable caching behavior
└── Your app is mostly read-heavy with few cross-query cache updates

Choose Apollo when:
├── You need normalized caching (updates to one entity propagate everywhere)
├── You have complex optimistic update requirements
├── You need mature devtools (Apollo DevTools is best-in-class)
├── Your schema has deep relationships traversed across many queries
└── You're building a large app where cache consistency is critical
```

### 5.2 urql Setup

```typescript
// packages/graphql/src/urql-client.ts

import {
  Client,
  cacheExchange,
  fetchExchange,
  subscriptionExchange,
  mapExchange,
} from 'urql';
import { createClient as createWsClient } from 'graphql-ws';
import { authExchange } from '@urql/exchange-auth';
import { retryExchange } from '@urql/exchange-retry';

interface UrqlClientConfig {
  httpUrl: string;
  wsUrl: string;
  getToken: () => Promise<string | null>;
  onAuthError: () => void;
}

export function createUrqlClient(config: UrqlClientConfig): Client {
  const { httpUrl, wsUrl, getToken, onAuthError } = config;

  // WebSocket client for subscriptions
  const wsClient = createWsClient({
    url: wsUrl,
    connectionParams: async () => {
      const token = await getToken();
      return { authorization: token ? `Bearer ${token}` : '' };
    },
  });

  return new Client({
    url: httpUrl,
    exchanges: [
      // Error handling
      mapExchange({
        onError(error) {
          const isAuthError = error.graphQLErrors.some(
            (e) =>
              e.extensions?.code === 'UNAUTHENTICATED' ||
              e.extensions?.code === 'FORBIDDEN',
          );
          if (isAuthError) onAuthError();
        },
      }),

      // Cache: document-level (by query + variables hash)
      cacheExchange,

      // Auth: add token to every request
      authExchange(async (utils) => {
        let token = await getToken();
        return {
          addAuthToOperation(operation) {
            if (!token) return operation;
            return utils.appendHeaders(operation, {
              Authorization: `Bearer ${token}`,
            });
          },
          didAuthError(error) {
            return error.graphQLErrors.some(
              (e) => e.extensions?.code === 'UNAUTHENTICATED',
            );
          },
          async refreshAuth() {
            token = await getToken();
          },
        };
      }),

      // Retry
      retryExchange({
        initialDelayMs: 1000,
        maxDelayMs: 15_000,
        randomDelay: true,
        maxNumberAttempts: 3,
        retryIf: (error) => !!(error && error.networkError),
      }),

      // Subscriptions
      subscriptionExchange({
        forwardSubscription(request) {
          const input = { ...request, query: request.query || '' };
          return {
            subscribe(sink) {
              const unsubscribe = wsClient.subscribe(input, sink);
              return { unsubscribe };
            },
          };
        },
      }),

      // HTTP fetch (must be last)
      fetchExchange,
    ],
  });
}
```

### 5.3 urql's Exchange Architecture

urql's power comes from its **exchange** architecture. Exchanges are middleware for GraphQL operations -- they can intercept, modify, cache, retry, or batch operations. They compose like Express middleware.

```
Operation Flow:

  Component                                             Network
     │                                                     ▲
     ▼                                                     │
  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
  │ dedupe   │→ │  cache   │→ │   auth   │→ │  fetch   │──┘
  │ Exchange │  │ Exchange │  │ Exchange │  │ Exchange │
  └──────────┘  └──────────┘  └──────────┘  └──────────┘
                     │
                     │ (cache hit: return immediately,
                     │  skip remaining exchanges)
                     ▼
                  Component
```

You can write custom exchanges:

```typescript
// A custom exchange that logs slow queries

import { pipe, tap } from 'wonka';
import type { Exchange } from 'urql';

export const slowQueryExchange: Exchange = ({ forward }) => (ops$) => {
  const timers = new Map<number, number>();

  return pipe(
    ops$,
    tap((op) => {
      if (op.kind === 'query' || op.kind === 'mutation') {
        timers.set(op.key, Date.now());
      }
    }),
    forward,
    tap((result) => {
      const start = timers.get(result.operation.key);
      if (start) {
        const duration = Date.now() - start;
        timers.delete(result.operation.key);
        if (duration > 2000) {
          console.warn(
            `[Slow GraphQL] ${result.operation.context.url} ` +
            `${result.operation.kind} took ${duration}ms`,
            result.operation.query.definitions[0],
          );
        }
      }
    }),
  );
};
```

### 5.4 urql with Normalized Cache (@urql/exchange-graphcache)

If you need normalized caching with urql, install `@urql/exchange-graphcache`. It's opt-in, unlike Apollo where it's the default.

```typescript
import { cacheExchange } from '@urql/exchange-graphcache';

const normalizedCache = cacheExchange({
  keys: {
    PageInfo: () => null, // Embedded type, no key
    PostEdge: () => null,
  },
  resolvers: {
    Query: {
      post: (_, args) => ({ __typename: 'Post', id: args.id }),
    },
  },
  updates: {
    Mutation: {
      createPost(result, _args, cache) {
        // Invalidate the feed so it refetches
        cache.invalidate('Query', 'feed');
      },
      likePost(result, _args, cache) {
        // Normalized cache auto-updates by ID -- nothing to do here
      },
    },
  },
});
```

---

## 6. GRAPHQL CODE GENERATOR

This is the section that transforms GraphQL from "fancy REST with extra steps" into "end-to-end type-safe data layer." Without codegen, you are writing GraphQL queries as strings, parsing responses as `any`, and hoping your queries match the schema. With codegen, your queries are validated at build time, your responses are fully typed, and your IDE autocompletes field names.

### 6.1 Setup

```bash
npm install -D @graphql-codegen/cli @graphql-codegen/client-preset
```

### 6.2 Configuration

```typescript
// codegen.ts

import type { CodegenConfig } from '@graphql-codegen/cli';

const config: CodegenConfig = {
  // Where to find the schema
  schema: [
    {
      'http://localhost:4000/graphql': {
        headers: {
          // Dev token for schema introspection
          Authorization: 'Bearer dev-introspection-token',
        },
      },
    },
  ],

  // Where to find your operations (queries, mutations, subscriptions)
  documents: [
    'packages/graphql/src/operations/**/*.graphql',
    'apps/mobile/src/**/*.graphql',
    'apps/web/src/**/*.graphql',
  ],

  generates: {
    // Single output file with all generated types and hooks
    'packages/graphql/src/generated/': {
      preset: 'client',
      presetConfig: {
        // Generate fragment masking helpers (Relay-style)
        fragmentMasking: { unmaskFunctionName: 'useFragment' },
      },
      config: {
        // Use exact optional properties (no `undefined`)
        exactOptionalPropertyTypes: true,
        // Scalar type mappings
        scalars: {
          DateTime: 'string',
          URL: 'string',
          JSON: 'Record<string, unknown>',
        },
        // Generate enum types as TypeScript unions (not enums)
        enumsAsTypes: true,
        // Don't add __typename to every type (add it where needed)
        skipTypename: false,
        // Deduplicate fragment types
        dedupeFragments: true,
      },
      plugins: [],
    },
  },

  // Watch mode for development
  watch: process.env.NODE_ENV === 'development',

  // Ignore type validation errors from the schema (useful during development)
  ignoreNoDocuments: true,
};

export default config;
```

### 6.3 Operation Files

Organize your GraphQL operations in `.graphql` files next to the features that use them. Codegen reads these files and generates typed hooks.

```graphql
# packages/graphql/src/operations/feed.graphql

query Feed($first: Int!, $after: String, $sort: SortOrder) {
  feed(first: $first, after: $after, sort: $sort) {
    edges {
      node {
        ...PostCard
      }
    }
    pageInfo {
      hasNextPage
      endCursor
    }
    totalCount
  }
}

query PostDetail($id: ID!) {
  post(id: $id) {
    ...PostDetail
  }
}

mutation CreatePost($input: CreatePostInput!) {
  createPost(input: $input) {
    post {
      ...PostCard
    }
    errors {
      field
      message
      code
    }
  }
}

mutation LikePost($postId: ID!) {
  likePost(postId: $postId) {
    post {
      id
      likesCount
      viewerHasLiked
    }
  }
}

mutation UnlikePost($postId: ID!) {
  unlikePost(postId: $postId) {
    post {
      id
      likesCount
      viewerHasLiked
    }
  }
}
```

```graphql
# packages/graphql/src/operations/fragments.graphql

fragment PostCard on Post {
  id
  body
  status
  createdAt
  author {
    ...UserAvatar
  }
  media {
    ... on Image {
      id
      url
      width
      height
      blurhash
    }
    ... on Video {
      id
      thumbnailUrl
      duration
    }
  }
  likesCount
  commentsCount
  viewerHasLiked
}

fragment PostDetail on Post {
  ...PostCard
  updatedAt
  comments(first: 10) {
    edges {
      node {
        ...CommentItem
      }
    }
    pageInfo {
      hasNextPage
      endCursor
    }
    totalCount
  }
}

fragment CommentItem on Comment {
  id
  body
  createdAt
  likesCount
  author {
    ...UserAvatar
  }
}

fragment UserAvatar on User {
  id
  username
  displayName
  avatarUrl
}
```

### 6.4 Running Codegen

```json
// package.json (root)
{
  "scripts": {
    "codegen": "graphql-codegen --config codegen.ts",
    "codegen:watch": "graphql-codegen --config codegen.ts --watch"
  }
}
```

```bash
# Generate types once
npm run codegen

# Watch mode (regenerates on file change)
npm run codegen:watch
```

### 6.5 What Codegen Generates

After running codegen, you get a `generated` directory with:

```
packages/graphql/src/generated/
├── fragment-masking.ts     # useFragment helper for fragment masking
├── gql.ts                  # The gql() tagged template function
├── graphql.ts              # All TypeScript types and document nodes
└── index.ts                # Re-exports
```

The generated types look like this:

```typescript
// generated/graphql.ts (simplified)

// --- Enum types ---
export type PostStatus = 'DRAFT' | 'PUBLISHED' | 'ARCHIVED';
export type SortOrder = 'NEWEST' | 'OLDEST' | 'MOST_LIKED';

// --- Fragment types ---
export type UserAvatarFragment = {
  __typename: 'User';
  id: string;
  username: string;
  displayName: string;
  avatarUrl: string | null;
};

export type PostCardFragment = {
  __typename: 'Post';
  id: string;
  body: string;
  status: PostStatus;
  createdAt: string;
  author: UserAvatarFragment;
  media: Array<
    | { __typename: 'Image'; id: string; url: string; width: number; height: number; blurhash: string | null }
    | { __typename: 'Video'; id: string; thumbnailUrl: string; duration: number }
  >;
  likesCount: number;
  commentsCount: number;
  viewerHasLiked: boolean;
};

// --- Query types ---
export type FeedQueryVariables = {
  first: number;
  after?: string | null;
  sort?: SortOrder | null;
};

export type FeedQuery = {
  __typename: 'Query';
  feed: {
    __typename: 'PostConnection';
    totalCount: number;
    edges: Array<{
      __typename: 'PostEdge';
      node: PostCardFragment;
    }>;
    pageInfo: {
      __typename: 'PageInfo';
      hasNextPage: boolean;
      endCursor: string | null;
    };
  };
};

// --- Document nodes ---
export const FeedDocument = /* GraphQL */ `...`;   // The query string
export const PostCardFragmentDoc = /* GraphQL */ `...`;

// --- Typed hooks (when using the React plugin) ---
export function useFeedQuery(
  baseOptions: Apollo.QueryHookOptions<FeedQuery, FeedQueryVariables>,
) { ... }

export function useCreatePostMutation(
  baseOptions?: Apollo.MutationHookOptions<CreatePostMutation, CreatePostMutationVariables>,
) { ... }
```

Now every component that uses `useFeedQuery` gets full type safety:

```typescript
const { data } = useFeedQuery({ variables: { first: 20 } });

// data.feed.edges[0].node.author.displayName -> string
// data.feed.edges[0].node.likesCount -> number
// data.feed.edges[0].node.nonExistent -> TypeScript ERROR
// data.feed.pageInfo.hasNextPage -> boolean
```

### 6.6 Integrating Codegen into CI

```yaml
# .github/workflows/ci.yml (relevant job)

codegen-check:
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v4
    - uses: actions/setup-node@v4
      with:
        node-version: 20
    - run: npm ci

    # Start the GraphQL server for schema introspection
    - run: npm run dev:server &
    - run: npx wait-on http://localhost:4000/graphql

    # Run codegen
    - run: npm run codegen

    # Fail if generated files have changed (dev forgot to run codegen)
    - run: git diff --exit-code packages/graphql/src/generated/
```

This catches the most common codegen mistake: someone changes a query or the schema, forgets to run codegen, and commits stale generated types that compile but don't match reality.

---

## 7. FRAGMENTS: COLOCATING DATA REQUIREMENTS WITH COMPONENTS

Fragments are GraphQL's component model. They let you colocate data requirements with the components that render the data, then compose fragments into larger queries. This is the pattern that makes GraphQL scale in large codebases.

### 7.1 The Problem Fragments Solve

Without fragments, every query is a monolithic blob that duplicates field selections across screens:

```graphql
# BAD: The feed query selects fields for PostCard
query Feed {
  feed(first: 20) {
    edges {
      node {
        id
        body
        createdAt
        author { id displayName avatarUrl }
        likesCount
        commentsCount
        viewerHasLiked
      }
    }
  }
}

# BAD: The profile query duplicates the same fields
query UserProfile($id: ID!) {
  user(id: $id) {
    posts(first: 20) {
      edges {
        node {
          id
          body
          createdAt
          author { id displayName avatarUrl }
          likesCount
          commentsCount
          viewerHasLiked
        }
      }
    }
  }
}

# Now PostCard needs a new field (media). You update it in... both queries?
# And the search results query? And the bookmarks query?
# This doesn't scale.
```

With fragments, each component declares its own data requirements:

```graphql
# PostCard declares what it needs
fragment PostCard on Post {
  id
  body
  createdAt
  author { ...UserAvatar }
  likesCount
  commentsCount
  viewerHasLiked
  media {
    ... on Image { id url width height blurhash }
    ... on Video { id thumbnailUrl duration }
  }
}

# UserAvatar declares what it needs
fragment UserAvatar on User {
  id
  displayName
  avatarUrl
}

# Queries compose fragments -- they don't duplicate fields
query Feed { feed(first: 20) { edges { node { ...PostCard } } } }
query UserProfile($id: ID!) { user(id: $id) { posts(first: 20) { edges { node { ...PostCard } } } } }
```

Now when PostCard needs a new field, you add it to the `PostCard` fragment. Every query that uses `...PostCard` automatically includes the new field. One change, one place.

### 7.2 Fragment Masking (Relay-Style)

Fragment masking is a stricter version of fragment colocation. With regular fragments, any parent component can access the fragment's fields. With fragment masking, only the component that owns the fragment can read its data. This enforces encapsulation.

The codegen config we set up in Section 6 generates a `useFragment` helper for this:

```typescript
// components/PostCard.tsx

import { useFragment, type FragmentType } from '@acme/graphql/generated/fragment-masking';
import { PostCardFragmentDoc, type PostCardFragment } from '@acme/graphql/generated';
import { UserAvatar } from './UserAvatar';

interface PostCardProps {
  // FragmentType is an opaque type -- the parent can't read PostCard fields
  post: FragmentType<typeof PostCardFragmentDoc>;
}

export function PostCard({ post: postRef }: PostCardProps) {
  // useFragment "unmasks" the data -- only this component can read it
  const post = useFragment(PostCardFragmentDoc, postRef);

  return (
    <View style={styles.card}>
      <UserAvatar user={post.author} />
      <Text style={styles.body}>{post.body}</Text>
      <View style={styles.meta}>
        <LikeButton
          postId={post.id}
          isLiked={post.viewerHasLiked}
          count={post.likesCount}
        />
        <Text>{post.commentsCount} comments</Text>
      </View>
      <MediaGallery media={post.media} />
    </View>
  );
}
```

```typescript
// screens/FeedScreen.tsx

import { useFeedQuery } from '@acme/graphql/generated';
import { PostCard } from '../components/PostCard';

export function FeedScreen() {
  const { data } = useFeedQuery({ variables: { first: 20 } });

  // data.feed.edges[0].node is FragmentType<PostCardFragment>
  // You CAN'T access data.feed.edges[0].node.body here.
  // The data is "masked" -- only PostCard can read PostCard fields.
  // This prevents parent components from depending on child data requirements.

  return (
    <FlatList
      data={data?.feed.edges ?? []}
      renderItem={({ item }) => <PostCard post={item.node} />}
    />
  );
}
```

**Why this matters:** In a large codebase, fragment masking prevents the "data dependency spaghetti" problem where parent components reach into child fragments and break when those fragments change. It's the GraphQL equivalent of component encapsulation.

### 7.3 Fragment Composition Patterns

Fragments compose naturally. Build small fragments for leaf components, then compose them into larger fragments for container components.

```graphql
# Leaf fragments (small, specific)
fragment UserAvatar on User {
  id
  displayName
  avatarUrl
}

fragment UserBio on User {
  id
  bio
  createdAt
  followersCount
  followingCount
}

# Composed fragment (combines leaf fragments)
fragment UserProfileHeader on User {
  ...UserAvatar
  ...UserBio
  username
  viewerIsFollowing
}

# Page-level query (composes all needed fragments)
query UserProfile($id: ID!) {
  user(id: $id) {
    ...UserProfileHeader
    posts(first: 20) {
      edges {
        node {
          ...PostCard
        }
      }
      pageInfo {
        hasNextPage
        endCursor
      }
    }
  }
}
```

The composition hierarchy mirrors the component hierarchy:

```
UserProfileScreen
├── UserProfileHeader
│   ├── UserAvatar (fragment: UserAvatar)
│   └── UserBio (fragment: UserBio)
└── PostList
    └── PostCard (fragment: PostCard)
        ├── UserAvatar (fragment: UserAvatar)
        └── MediaGallery
```

---

## 8. CURSOR-BASED PAGINATION WITH RELAY CONNECTIONS

Pagination in GraphQL is not just "pass a page number." Cursor-based pagination is the standard because it works correctly under concurrent writes, scales to millions of items, and has a well-defined protocol (the Relay connection spec).

### 8.1 Why Cursor-Based, Not Offset-Based

```
Offset pagination breaks under concurrent writes:

  User loads page 1 (items 1-20)
  Another user deletes item 5
  User loads page 2 (items 21-40)
  BUT items shifted: user sees item 20 again (it moved from position 20 to 19)
  AND misses what was at position 21 (it moved to position 20)

Cursor pagination is stable:

  User loads first 20 items, gets cursor for position after item 20
  Another user deletes item 5
  User loads next 20 items AFTER cursor
  Gets items 21-40 from the ORIGINAL sequence
  No duplicates, no missed items
```

### 8.2 Full Pagination Implementation

```typescript
// hooks/usePaginatedFeed.ts

import { useCallback, useMemo } from 'react';
import { useFeedQuery, type SortOrder } from '@acme/graphql/generated';

const PAGE_SIZE = 20;

interface UsePaginatedFeedOptions {
  sort?: SortOrder;
}

export function usePaginatedFeed(options: UsePaginatedFeedOptions = {}) {
  const { sort = 'NEWEST' } = options;

  const { data, loading, error, fetchMore, refetch, networkStatus } = useFeedQuery({
    variables: { first: PAGE_SIZE, sort },
    fetchPolicy: 'cache-and-network',
    notifyOnNetworkStatusChange: true,
  });

  const isRefreshing = networkStatus === 4;
  const isLoadingMore = networkStatus === 3;
  const isInitialLoad = loading && !data;

  const posts = useMemo(
    () => data?.feed.edges.map((edge) => edge.node) ?? [],
    [data?.feed.edges],
  );

  const pageInfo = data?.feed.pageInfo;
  const totalCount = data?.feed.totalCount ?? 0;

  const loadMore = useCallback(async () => {
    if (!pageInfo?.hasNextPage || isLoadingMore) return;

    try {
      await fetchMore({
        variables: {
          after: pageInfo.endCursor,
          first: PAGE_SIZE,
          sort,
        },
        // Type policies handle the merge -- no updateQuery needed
      });
    } catch (error) {
      // fetchMore errors are usually network issues.
      // Don't crash -- the user can try again.
      console.warn('[Feed] Failed to load more:', error);
    }
  }, [pageInfo, isLoadingMore, fetchMore, sort]);

  const refresh = useCallback(async () => {
    try {
      await refetch({ first: PAGE_SIZE, sort });
    } catch (error) {
      console.warn('[Feed] Failed to refresh:', error);
    }
  }, [refetch, sort]);

  return {
    posts,
    totalCount,
    loading: isInitialLoad,
    refreshing: isRefreshing,
    loadingMore: isLoadingMore,
    hasNextPage: pageInfo?.hasNextPage ?? false,
    error,
    loadMore,
    refresh,
  };
}
```

### 8.3 Infinite Scroll Component

```typescript
// screens/FeedScreen.tsx

import React, { useCallback } from 'react';
import { FlatList, RefreshControl, ActivityIndicator, View, Text } from 'react-native';
import { usePaginatedFeed } from '../hooks/usePaginatedFeed';
import { PostCard } from '../components/PostCard';

export function FeedScreen() {
  const {
    posts,
    totalCount,
    loading,
    refreshing,
    loadingMore,
    hasNextPage,
    error,
    loadMore,
    refresh,
  } = usePaginatedFeed({ sort: 'NEWEST' });

  const renderItem = useCallback(
    ({ item }: { item: (typeof posts)[0] }) => <PostCard post={item} />,
    [],
  );

  const keyExtractor = useCallback(
    (item: (typeof posts)[0]) => item.id,
    [],
  );

  const ListFooterComponent = useCallback(() => {
    if (loadingMore) {
      return (
        <View style={{ paddingVertical: 20 }}>
          <ActivityIndicator />
        </View>
      );
    }
    if (!hasNextPage && posts.length > 0) {
      return (
        <View style={{ paddingVertical: 20, alignItems: 'center' }}>
          <Text style={{ color: '#999' }}>You've reached the end</Text>
        </View>
      );
    }
    return null;
  }, [loadingMore, hasNextPage, posts.length]);

  if (loading) {
    return <FeedSkeleton />;
  }

  if (error && posts.length === 0) {
    return <ErrorScreen error={error} onRetry={refresh} />;
  }

  return (
    <FlatList
      data={posts}
      keyExtractor={keyExtractor}
      renderItem={renderItem}
      onEndReached={loadMore}
      onEndReachedThreshold={0.5}
      refreshControl={
        <RefreshControl refreshing={refreshing} onRefresh={refresh} />
      }
      ListFooterComponent={ListFooterComponent}
      // Performance optimizations for large lists
      removeClippedSubviews
      maxToRenderPerBatch={10}
      windowSize={5}
      initialNumToRender={10}
      getItemLayout={undefined} // Dynamic heights -- can't use getItemLayout
    />
  );
}
```

### 8.4 Bidirectional Pagination

Some use cases need backward pagination (loading older messages in a chat, scrolling up to see previous items). The Relay spec supports this with `last`/`before` arguments.

```graphql
query ChatMessages($channelId: ID!, $last: Int!, $before: String) {
  channel(id: $channelId) {
    messages(last: $last, before: $before) {
      edges {
        node {
          id
          body
          author { id displayName avatarUrl }
          createdAt
        }
      }
      pageInfo {
        hasPreviousPage
        startCursor
      }
    }
  }
}
```

```typescript
// hooks/useChatMessages.ts

export function useChatMessages(channelId: string) {
  const { data, fetchMore } = useChatMessagesQuery({
    variables: { channelId, last: 30 },
  });

  const loadOlder = useCallback(async () => {
    const pageInfo = data?.channel.messages.pageInfo;
    if (!pageInfo?.hasPreviousPage) return;

    await fetchMore({
      variables: {
        before: pageInfo.startCursor,
        last: 30,
      },
    });
  }, [data, fetchMore]);

  // Messages come newest-last in backward pagination
  const messages = useMemo(
    () => data?.channel.messages.edges.map((e) => e.node) ?? [],
    [data],
  );

  return { messages, loadOlder, hasPreviousPage: data?.channel.messages.pageInfo.hasPreviousPage };
}
```

---

## 9. REAL-TIME SUBSCRIPTIONS

GraphQL subscriptions provide real-time data over a persistent connection. They are the most elegant way to add live features to a GraphQL app -- but they are also the most operationally complex.

### 9.1 When Subscriptions vs Polling

```
Use subscriptions when:
├── Low latency matters (< 1 second): chat messages, typing indicators
├── Updates are frequent and unpredictable: live sports scores, stock tickers
├── The user is actively watching for changes: real-time collaboration
└── You need server-push semantics: notifications, alerts

Use polling when:
├── Updates are infrequent (every 30s+): dashboard metrics, order status
├── You don't want WebSocket infrastructure: simpler to deploy and scale
├── Data staleness of a few seconds is acceptable
├── The client is not actively watching (background refresh)
└── You're behind a CDN that doesn't support WebSocket
```

The practical rule: **start with polling, switch to subscriptions when polling frequency exceeds once every 5 seconds.** Polling is simpler to implement, debug, and scale. Subscriptions are a backend infrastructure commitment (WebSocket servers, connection state management, load balancer configuration).

### 9.2 Subscription Setup

The Apollo Client setup from Section 4.2 already includes WebSocket transport. Here's how to use subscriptions in components.

```typescript
// hooks/useRealtimeComments.ts

import { useEffect } from 'react';
import {
  usePostDetailQuery,
  useCommentAddedSubscription,
  PostDetailDocument,
  type PostDetailQuery,
} from '@acme/graphql/generated';
import { useApolloClient } from '@apollo/client';

export function useRealtimeComments(postId: string) {
  const client = useApolloClient();

  // Base query for the post and initial comments
  const { data, loading, error } = usePostDetailQuery({
    variables: { id: postId },
  });

  // Subscribe to new comments
  const { data: subscriptionData } = useCommentAddedSubscription({
    variables: { postId },
    // Only subscribe when the query has loaded
    skip: !data,
  });

  // When a new comment arrives via subscription, add it to the cache
  useEffect(() => {
    if (!subscriptionData?.commentAdded) return;

    const newComment = subscriptionData.commentAdded;

    // Update the post's comments in the cache
    client.cache.updateQuery<PostDetailQuery>(
      {
        query: PostDetailDocument,
        variables: { id: postId },
      },
      (existing) => {
        if (!existing?.post) return existing;

        // Check if the comment already exists (deduplication)
        const alreadyExists = existing.post.comments.edges.some(
          (edge) => edge.node.id === newComment.id,
        );
        if (alreadyExists) return existing;

        return {
          ...existing,
          post: {
            ...existing.post,
            commentsCount: existing.post.commentsCount + 1,
            comments: {
              ...existing.post.comments,
              totalCount: existing.post.comments.totalCount + 1,
              edges: [
                ...existing.post.comments.edges,
                {
                  __typename: 'CommentEdge' as const,
                  cursor: newComment.id,
                  node: newComment,
                },
              ],
            },
          },
        };
      },
    );
  }, [subscriptionData, client, postId]);

  return {
    post: data?.post,
    loading,
    error,
  };
}
```

### 9.3 Subscription with Polling Fallback

For production robustness, combine subscriptions with polling as a fallback. If the WebSocket disconnects, the polling keeps data fresh until the connection recovers.

```typescript
// hooks/useLiveFeed.ts

import { useEffect, useRef, useState } from 'react';
import { useFeedQuery, useNewFeedPostSubscription } from '@acme/graphql/generated';

const POLL_INTERVAL = 30_000; // 30 seconds fallback

export function useLiveFeed() {
  const [wsConnected, setWsConnected] = useState(true);

  const { data, loading, refetch, startPolling, stopPolling } = useFeedQuery({
    variables: { first: 20 },
    fetchPolicy: 'cache-and-network',
  });

  // Subscribe to new posts
  const { data: newPostData, error: subError } = useNewFeedPostSubscription({
    onError: () => {
      // WebSocket disconnected -- fall back to polling
      setWsConnected(false);
    },
  });

  // Manage polling based on WebSocket connection state
  useEffect(() => {
    if (wsConnected) {
      stopPolling();
    } else {
      startPolling(POLL_INTERVAL);
    }

    return () => stopPolling();
  }, [wsConnected, startPolling, stopPolling]);

  // When subscription reconnects, stop polling
  useEffect(() => {
    if (newPostData && !wsConnected) {
      setWsConnected(true);
    }
  }, [newPostData, wsConnected]);

  return {
    data,
    loading,
    wsConnected,
    refetch,
  };
}
```

### 9.4 The graphql-ws Protocol

The modern WebSocket transport for GraphQL is the `graphql-ws` library and protocol. It replaces the older `subscriptions-transport-ws` (which is unmaintained). Here's what you need to know:

```
graphql-ws protocol lifecycle:

  Client                              Server
    │                                    │
    │──── ConnectionInit ───────────────>│  (send auth params)
    │<─── ConnectionAck ────────────────│  (server accepts)
    │                                    │
    │──── Subscribe { id, query } ──────>│  (start subscription)
    │<─── Next { id, payload } ─────────│  (data event)
    │<─── Next { id, payload } ─────────│  (another event)
    │<─── Next { id, payload } ─────────│  (and another)
    │                                    │
    │──── Complete { id } ──────────────>│  (client unsubscribes)
    │                                    │
    │──── Ping ─────────────────────────>│  (keep-alive)
    │<─── Pong ─────────────────────────│
    │                                    │
```

**Key configuration for mobile apps:**

```typescript
const wsClient = createWsClient({
  url: wsUrl,
  connectionParams: async () => ({
    authorization: `Bearer ${await getToken()}`,
  }),

  // Critical for mobile: handle app backgrounding
  lazy: true, // Only connect when there's an active subscription
  lazyCloseTimeout: 5_000, // Keep connection alive 5s after last subscription

  // Reconnection for flaky mobile networks
  retryAttempts: Infinity,
  shouldRetry: () => true,
  retryWait: async (retryCount) => {
    const delay = Math.min(1000 * Math.pow(2, retryCount), 30_000);
    await new Promise((r) => setTimeout(r, delay + delay * 0.2 * Math.random()));
  },

  // Keep-alive to detect dead connections
  keepAlive: 10_000,

  // Handle connection events for UI feedback
  on: {
    connected: () => console.log('[WS] Connected'),
    closed: (event) => console.log('[WS] Closed', event),
    error: (error) => console.error('[WS] Error', error),
  },
});
```

---

## 10. PERFORMANCE

GraphQL's flexibility comes with performance pitfalls that don't exist in REST. This section covers the techniques that keep GraphQL fast in production.

### 10.1 Persisted Queries

Every GraphQL request includes the full query string in the body. For a complex query with fragments, that string can be several kilobytes. Persisted queries replace the query string with a hash, reducing request size by 90%+ and enabling CDN caching.

**How it works:**

```
Without persisted queries:
  POST /graphql
  Body: { "query": "query Feed($first: Int!) { feed(first: $first) { edges { node { id body ... } } } }", "variables": { "first": 20 } }
  Size: ~2KB

With persisted queries:
  POST /graphql (or GET /graphql?extensions=...)
  Body: { "extensions": { "persistedQuery": { "sha256Hash": "abc123...", "version": 1 } }, "variables": { "first": 20 } }
  Size: ~200 bytes
```

**Apollo Client supports Automatic Persisted Queries (APQ) out of the box:**

```typescript
// packages/graphql/src/client.ts

import { createPersistedQueryLink } from '@apollo/client/link/persisted-queries';
import { sha256 } from 'crypto-hash';

const persistedQueryLink = createPersistedQueryLink({
  sha256,
  useGETForHashedQueries: true, // Enables CDN caching for queries
});

// Add to the link chain (before httpLink)
const link = from([
  retryLink,
  createErrorLink(onAuthError),
  createAuthContextLink(getToken),
  persistedQueryLink,
  transportLink,
]);
```

**The APQ flow:**

```
1. Client sends request with only the hash (no query string)
2. Server checks if it has seen this hash before
   ├── YES: Execute the query
   └── NO: Return "PersistedQueryNotFound" error
3. Client retries with BOTH the hash AND the full query string
4. Server stores the hash -> query mapping and executes
5. All subsequent requests use only the hash
```

This is the simplest approach. For tighter security, you can use a **query allowlist** generated at build time:

```typescript
// codegen.ts -- add persisted operations plugin

const config: CodegenConfig = {
  // ... existing config
  generates: {
    // ... existing generates
    'packages/graphql/src/generated/persisted-operations.json': {
      plugins: ['graphql-codegen-persisted-query-ids'],
      config: {
        output: 'client',
        algorithm: 'sha256',
      },
    },
  },
};
```

With a query allowlist, the server only executes queries whose hashes appear in the allowlist. This prevents query abuse -- attackers can't send arbitrary queries.

### 10.2 Query Complexity Analysis

GraphQL lets clients request arbitrarily nested data. Without limits, a malicious client can send a query that fetches the entire database:

```graphql
# Denial of service query:
query Evil {
  feed(first: 100) {
    edges {
      node {
        author {
          posts(first: 100) {
            edges {
              node {
                comments(first: 100) {
                  edges {
                    node {
                      author {
                        posts(first: 100) {
                          # ... and so on, nesting infinitely
                        }
                      }
                    }
                  }
                }
              }
            }
          }
        }
      }
    }
  }
}
```

**Server-side protections (the backend team needs to implement these):**

```typescript
// Server-side: Query depth and complexity limits

import depthLimit from 'graphql-depth-limit';
import { createComplexityLimitRule } from 'graphql-validation-complexity';

const server = new ApolloServer({
  schema,
  validationRules: [
    // Limit query depth to 10 levels
    depthLimit(10),

    // Limit query complexity (each field has a cost)
    createComplexityLimitRule(1000, {
      scalarCost: 1,
      objectCost: 2,
      listFactor: 10, // Lists multiply the cost of their children
    }),
  ],
});
```

**Frontend impact:** If your queries are rejected for complexity, you need to restructure them. Break deep queries into multiple shallower queries, or reduce the number of items in paginated fields.

### 10.3 DataLoader and the N+1 Problem

This is a server-side concern, but frontend developers need to understand it because it directly affects query performance.

```
The N+1 problem:

  Query: feed(first: 20) { edges { node { author { name } } } }

  Without DataLoader:
    1 query: SELECT * FROM posts LIMIT 20           (the feed)
    20 queries: SELECT * FROM users WHERE id = ?     (each post's author)
    Total: 21 queries for 20 posts

  With DataLoader:
    1 query: SELECT * FROM posts LIMIT 20           (the feed)
    1 query: SELECT * FROM users WHERE id IN (?, ?, ..., ?)  (all authors, batched)
    Total: 2 queries for 20 posts
```

**If your GraphQL API is slow, check the server-side resolvers for N+1 issues before blaming the client.** DataLoader is the standard solution -- it batches and deduplicates resolver calls within a single request.

### 10.4 Request Batching

Apollo Client can batch multiple queries into a single HTTP request. This is useful when a screen renders multiple components that each make their own query.

```typescript
// packages/graphql/src/client.ts

import { BatchHttpLink } from '@apollo/client/link/batch-http';

// Replace HttpLink with BatchHttpLink
const httpLink = new BatchHttpLink({
  uri: httpUrl,
  batchMax: 5, // Max 5 queries per batch
  batchInterval: 20, // Wait 20ms to collect queries before sending
});
```

**When to use batching:**
- Multiple independent queries fire on the same screen render
- Your server supports batched queries (most GraphQL servers do)
- You're not using HTTP/2 (which already multiplexes requests)

**When NOT to use batching:**
- You're using persisted queries with GET (GET queries can't be batched)
- Your CDN caches individual query responses (batched queries bypass the CDN)
- One slow resolver blocks the entire batch response

### 10.5 CDN Caching for GraphQL

REST APIs get CDN caching for free because each URL is a unique cache key. GraphQL uses a single endpoint, so CDN caching requires extra work.

**Option 1: GET requests with persisted queries**

```
GET /graphql?extensions={"persistedQuery":{"sha256Hash":"abc123","version":1}}&variables={"first":20}

This URL is unique per query + variables, so the CDN can cache it.
Set Cache-Control headers on the server per-query.
```

**Option 2: Edge caching with Stellate (formerly GraphCDN)**

Stellate is a CDN purpose-built for GraphQL. It understands your schema, caches per-type, and automatically invalidates cached queries when mutations change the underlying data.

```typescript
// Stellate configuration
export default {
  name: 'my-app',
  originUrl: 'https://api.myapp.com/graphql',
  rules: [
    {
      types: ['Post'],
      maxAge: 60, // Cache posts for 60 seconds
      staleWhileRevalidate: 300,
    },
    {
      types: ['User'],
      maxAge: 300, // Cache user profiles for 5 minutes
      staleWhileRevalidate: 600,
    },
    {
      types: ['Query'],
      fields: ['viewer'],
      maxAge: 0, // Never cache viewer-specific data
    },
  ],
};
```

### 10.6 Client-Side Performance Checklist

```
□ Codegen is configured and running in CI
□ Queries are colocated with components via fragments
□ No query fetches more than 50 items at once (use pagination)
□ Persisted queries are enabled in production
□ Fetch policies are intentional per-query (not all cache-and-network)
□ Large lists use FlatList with virtualization, not map()
□ Images use blurhash placeholders (returned in the query)
□ Optimistic updates are implemented for like/follow/bookmark actions
□ Subscriptions have polling fallback
□ Apollo DevTools or urql devtools are configured for development
□ Query depth does not exceed 5 levels in any operation
□ Fragment masking prevents over-fetching in parent components
```

---

## 11. THE HONEST TRADE-OFFS

Every conference talk about GraphQL covers the benefits. This section covers what happens after the honeymoon period -- the problems that emerge at year two, when the schema has 200 types, three teams are committing to it, and the normalized cache has become a source of intermittent bugs that nobody can reproduce.

### 11.1 The N+1 Problem Is Your Problem Now

In REST, the N+1 problem is solved by the backend team with SQL joins and eager loading. In GraphQL, the resolver architecture splits data fetching across individual field resolvers, which naturally creates N+1 patterns. DataLoader mitigates this, but it requires discipline: every resolver that loads related data needs to go through DataLoader. Miss one and you've got a slow query that only shows up under load.

**The frontend impact:** You will file performance bugs against the backend that turn out to be N+1 issues in resolvers. Learn to read server-side traces (Datadog, New Relic) to diagnose these. When your "simple" query takes 2 seconds, it's usually 200 sequential database queries caused by nested resolvers.

### 11.2 Schema Maintenance Overhead

REST endpoints are independent -- changing one doesn't affect others. A GraphQL schema is a single interconnected type system. Changing a type that's used in 15 queries requires careful coordination. Deprecating a field means checking which clients still query it (which requires query analytics). Adding a required field to an input type is a breaking change.

**What this looks like in practice:**

```
Week 1: "The schema is a beautiful, self-documenting contract!"
Month 6: "Who added this field and why? There's no comment."
Year 1: "We can't rename this field because the iOS app from 6 months ago still queries it."
Year 2: "40% of our schema is deprecated fields we can't remove."
```

**Mitigations:**
- Use schema linting (graphql-eslint) to enforce naming conventions and deprecation policies
- Use query analytics to track which fields are actually queried by which clients
- Set a deprecation-to-removal timeline (e.g., 6 months after deprecation for mobile, 3 months for web)
- Run schema reviews like code reviews -- every schema change gets a PR

### 11.3 The Normalized Cache Is a Complexity Multiplier

Apollo's normalized cache is powerful but creates a class of bugs that don't exist with simple document caching:

**Stale references:** An entity is deleted on the server but remains in the cache as a dangling reference. Other queries that reference it show stale data or crash.

```typescript
// The fix: cache eviction on delete mutations
const [deletePost] = useDeletePostMutation({
  update(cache, { data }) {
    if (!data?.deletePost.success) return;
    // Evict the post from the normalized cache
    cache.evict({ id: cache.identify({ __typename: 'Post', id: postId }) });
    // Garbage collect dangling references
    cache.gc();
  },
});
```

**Partial data:** Different queries fetch different fields for the same entity. If Query A fetches `Post.body` and Query B fetches `Post.media`, reading the cache after Query A might give you a post without media, even if Query B has loaded it.

**Type policy bugs:** Merge functions in type policies are tricky to get right. A pagination merge that deduplicates incorrectly can drop items. A key fields configuration that's wrong can cause entities to overwrite each other.

**The mitigation:** If your app's data model is simple (few relationships, few shared entities across queries), use urql's document cache or even TanStack Query with a plain fetch. You only need normalized caching when the same entities genuinely appear across multiple screens and you need updates to propagate automatically.

### 11.4 Tooling Complexity

A fully set-up GraphQL stack requires:

```
1. Schema definition (SDL or code-first)
2. Server framework (Apollo Server, Mercurius, Yoga)
3. DataLoader for N+1 prevention
4. Client library (Apollo Client, urql)
5. Codegen for type safety
6. Fragment colocation
7. Type policies for cache configuration
8. Persisted queries for production
9. Schema linting
10. Query analytics
11. Subscription infrastructure (WebSocket server)
12. Schema registry (for breaking change detection)
```

Compare to REST + TanStack Query:

```
1. Server framework (Express, Fastify)
2. Client: fetch + TanStack Query
3. Type sharing: tRPC or shared TypeScript types
```

The difference is 12 moving parts vs 3. Each moving part is a configuration file, a dependency, a potential version conflict, and a thing someone on the team needs to understand. This is the "GraphQL tax."

### 11.5 When Teams Abandon GraphQL

The most common pattern for GraphQL abandonment:

1. Team adopts GraphQL for a new project. Excitement is high.
2. Initial setup works great. Queries are elegant. Types are beautiful.
3. Month 6: codegen breaks after a dependency update. Fix takes a day.
4. Month 9: normalized cache has mysterious stale data bugs. Fix takes a week.
5. Year 1: schema has grown organically without design reviews. Naming is inconsistent. Deprecated fields accumulate.
6. Year 1.5: the mobile app needs a query that's awkward to express in the schema. Team writes a REST endpoint for it.
7. Year 2: new hires struggle with the tooling complexity. "Why can't I just fetch data?" becomes a common question.
8. Year 2.5: team evaluates tRPC for the next project.

**The lesson:** GraphQL is not a "set it and forget it" choice. It requires ongoing schema governance, cache maintenance, and tooling investment. If your team isn't prepared for that maintenance, you'll pay the tax without getting the benefits.

### 11.6 The Honest Recommendation

```
If you're reading this and haven't chosen yet:

  Team size < 5, TypeScript monorepo    → Use tRPC. Not even close.
  Team size < 5, no shared types        → Use REST + TanStack Query.
  Team size 5-15, mobile + web          → GraphQL is a strong choice IF you invest
                                           in schema design, codegen, and governance.
  Team size 15+, multiple platforms      → GraphQL with a schema registry and
                                           dedicated platform team.
  Public API                            → REST with OpenAPI. Always.
```

---

## 12. COMPLETE MONOREPO SETUP

Let's put everything together. Here's the full setup for a React Native + Next.js monorepo with Apollo Client, codegen, fragments, pagination, and subscriptions.

### 12.1 Directory Structure

```
monorepo/
├── apps/
│   ├── mobile/                    # React Native (Expo)
│   │   ├── src/
│   │   │   ├── providers/
│   │   │   │   └── ApolloProvider.tsx
│   │   │   ├── screens/
│   │   │   │   ├── FeedScreen.tsx
│   │   │   │   ├── PostDetailScreen.tsx
│   │   │   │   └── ProfileScreen.tsx
│   │   │   └── components/
│   │   │       ├── PostCard.tsx
│   │   │       └── UserAvatar.tsx
│   │   └── app.tsx
│   │
│   └── web/                       # Next.js
│       ├── src/
│       │   ├── app/
│       │   │   ├── layout.tsx
│       │   │   ├── providers.tsx
│       │   │   ├── page.tsx
│       │   │   └── post/[id]/page.tsx
│       │   └── components/
│       │       ├── PostCard.tsx
│       │       └── UserAvatar.tsx
│       └── next.config.ts
│
├── packages/
│   └── graphql/                   # Shared GraphQL package
│       ├── src/
│       │   ├── client.ts          # Apollo Client factory
│       │   ├── cache/
│       │   │   └── type-policies.ts
│       │   ├── operations/
│       │   │   ├── feed.graphql
│       │   │   ├── post.graphql
│       │   │   ├── user.graphql
│       │   │   ├── comments.graphql
│       │   │   └── fragments.graphql
│       │   ├── generated/          # Auto-generated by codegen
│       │   │   ├── fragment-masking.ts
│       │   │   ├── gql.ts
│       │   │   ├── graphql.ts
│       │   │   └── index.ts
│       │   └── index.ts           # Public API
│       ├── codegen.ts
│       └── package.json
│
├── codegen.ts                     # Root codegen config
└── package.json
```

### 12.2 Package Configuration

```json
// packages/graphql/package.json
{
  "name": "@acme/graphql",
  "version": "0.1.0",
  "main": "./src/index.ts",
  "types": "./src/index.ts",
  "dependencies": {
    "@apollo/client": "^3.11.0",
    "graphql": "^16.9.0",
    "graphql-ws": "^5.16.0"
  },
  "devDependencies": {
    "@graphql-codegen/cli": "^5.0.0",
    "@graphql-codegen/client-preset": "^4.3.0",
    "typescript": "^5.5.0"
  },
  "scripts": {
    "codegen": "graphql-codegen --config codegen.ts",
    "codegen:watch": "graphql-codegen --config codegen.ts --watch"
  }
}
```

### 12.3 The Shared GraphQL Package Entry Point

```typescript
// packages/graphql/src/index.ts

// Client factory
export { createApolloClient } from './client';

// Type policies
export { typePolicies } from './cache/type-policies';

// Generated types and hooks
export * from './generated';

// Re-export commonly used Apollo utilities
export {
  ApolloProvider,
  useApolloClient,
  gql,
} from '@apollo/client';
```

### 12.4 Operations by Feature

```graphql
# packages/graphql/src/operations/post.graphql

query PostDetail($id: ID!) {
  post(id: $id) {
    ...PostDetail
  }
}

mutation CreatePost($input: CreatePostInput!) {
  createPost(input: $input) {
    post {
      ...PostCard
    }
    errors {
      field
      message
      code
    }
  }
}

mutation UpdatePost($input: UpdatePostInput!) {
  updatePost(input: $input) {
    post {
      ...PostDetail
    }
    errors {
      field
      message
      code
    }
  }
}

mutation DeletePost($id: ID!) {
  deletePost(id: $id) {
    success
    errors {
      message
      code
    }
  }
}

mutation LikePost($postId: ID!) {
  likePost(postId: $postId) {
    post {
      id
      likesCount
      viewerHasLiked
    }
  }
}

mutation UnlikePost($postId: ID!) {
  unlikePost(postId: $postId) {
    post {
      id
      likesCount
      viewerHasLiked
    }
  }
}

subscription PostLiked($postId: ID!) {
  postLiked(postId: $postId) {
    id
    likesCount
  }
}
```

```graphql
# packages/graphql/src/operations/comments.graphql

query PostComments($postId: ID!, $first: Int!, $after: String) {
  post(id: $postId) {
    id
    comments(first: $first, after: $after) {
      edges {
        node {
          ...CommentItem
        }
      }
      pageInfo {
        hasNextPage
        endCursor
      }
      totalCount
    }
  }
}

mutation CreateComment($input: CreateCommentInput!) {
  createComment(input: $input) {
    comment {
      ...CommentItem
    }
    errors {
      field
      message
      code
    }
  }
}

mutation DeleteComment($id: ID!) {
  deleteComment(id: $id) {
    success
    errors {
      message
      code
    }
  }
}

subscription CommentAdded($postId: ID!) {
  commentAdded(postId: $postId) {
    ...CommentItem
  }
}
```

```graphql
# packages/graphql/src/operations/user.graphql

query Viewer {
  viewer {
    ...UserProfileHeader
    email
    notifications(first: 10) {
      edges {
        node {
          id
          type
          message
          read
          createdAt
        }
      }
      pageInfo {
        hasNextPage
        endCursor
      }
    }
  }
}

query UserProfile($id: ID!) {
  user(id: $id) {
    ...UserProfileHeader
    posts(first: 20) {
      edges {
        node {
          ...PostCard
        }
      }
      pageInfo {
        hasNextPage
        endCursor
      }
      totalCount
    }
  }
}

mutation UpdateProfile($input: UpdateProfileInput!) {
  updateProfile(input: $input) {
    user {
      ...UserProfileHeader
      email
    }
    errors {
      field
      message
      code
    }
  }
}

mutation FollowUser($userId: ID!) {
  followUser(userId: $userId) {
    user {
      id
      followersCount
      viewerIsFollowing
    }
  }
}

mutation UnfollowUser($userId: ID!) {
  unfollowUser(userId: $userId) {
    user {
      id
      followersCount
      viewerIsFollowing
    }
  }
}

fragment UserProfileHeader on User {
  ...UserAvatar
  ...UserBio
  username
  viewerIsFollowing
}

fragment UserBio on User {
  id
  bio
  createdAt
  followersCount
  followingCount
}
```

### 12.5 Complete PostCard Component (React Native)

```typescript
// apps/mobile/src/components/PostCard.tsx

import React, { memo, useCallback } from 'react';
import { View, Text, Pressable, StyleSheet, Image } from 'react-native';
import { useNavigation } from '@react-navigation/native';
import {
  useFragment,
  type FragmentType,
} from '@acme/graphql/generated/fragment-masking';
import {
  PostCardFragmentDoc,
  useLikePostMutation,
  useUnlikePostMutation,
} from '@acme/graphql/generated';
import { UserAvatar } from './UserAvatar';
import { formatRelativeTime } from '@acme/shared/utils/time';

interface PostCardProps {
  post: FragmentType<typeof PostCardFragmentDoc>;
}

export const PostCard = memo(function PostCard({ post: postRef }: PostCardProps) {
  const post = useFragment(PostCardFragmentDoc, postRef);
  const navigation = useNavigation();

  const [likePost] = useLikePostMutation();
  const [unlikePost] = useUnlikePostMutation();

  const handlePress = useCallback(() => {
    navigation.navigate('PostDetail', { id: post.id });
  }, [navigation, post.id]);

  const handleLike = useCallback(async () => {
    const mutation = post.viewerHasLiked ? unlikePost : likePost;
    const newCount = post.viewerHasLiked
      ? post.likesCount - 1
      : post.likesCount + 1;

    await mutation({
      variables: { postId: post.id },
      optimisticResponse: {
        __typename: 'Mutation',
        [post.viewerHasLiked ? 'unlikePost' : 'likePost']: {
          __typename: post.viewerHasLiked
            ? 'UnlikePostPayload'
            : 'LikePostPayload',
          post: {
            __typename: 'Post',
            id: post.id,
            likesCount: newCount,
            viewerHasLiked: !post.viewerHasLiked,
          },
        },
      },
    });
  }, [post, likePost, unlikePost]);

  const handleAuthorPress = useCallback(() => {
    navigation.navigate('UserProfile', { id: post.author.id });
  }, [navigation, post.author.id]);

  // Render the first image if present
  const firstImage = post.media.find((m) => m.__typename === 'Image');

  return (
    <Pressable onPress={handlePress} style={styles.container}>
      <View style={styles.header}>
        <Pressable onPress={handleAuthorPress}>
          <UserAvatar user={post.author} size={40} />
        </Pressable>
        <View style={styles.headerText}>
          <Text style={styles.displayName}>{post.author.displayName}</Text>
          <Text style={styles.timestamp}>
            {formatRelativeTime(post.createdAt)}
          </Text>
        </View>
      </View>

      <Text style={styles.body} numberOfLines={5}>
        {post.body}
      </Text>

      {firstImage && firstImage.__typename === 'Image' && (
        <Image
          source={{ uri: firstImage.url }}
          style={[
            styles.image,
            { aspectRatio: firstImage.width / firstImage.height },
          ]}
          // Use blurhash as placeholder
          blurRadius={firstImage.blurhash ? 0 : 2}
        />
      )}

      <View style={styles.actions}>
        <Pressable onPress={handleLike} style={styles.actionButton}>
          <Text style={post.viewerHasLiked ? styles.liked : styles.actionText}>
            {post.viewerHasLiked ? '❤' : '♡'} {post.likesCount}
          </Text>
        </Pressable>
        <Pressable onPress={handlePress} style={styles.actionButton}>
          <Text style={styles.actionText}>
            💬 {post.commentsCount}
          </Text>
        </Pressable>
      </View>
    </Pressable>
  );
});

const styles = StyleSheet.create({
  container: {
    backgroundColor: '#fff',
    paddingVertical: 12,
    paddingHorizontal: 16,
    borderBottomWidth: StyleSheet.hairlineWidth,
    borderBottomColor: '#e0e0e0',
  },
  header: {
    flexDirection: 'row',
    alignItems: 'center',
    marginBottom: 8,
  },
  headerText: {
    marginLeft: 10,
  },
  displayName: {
    fontSize: 15,
    fontWeight: '600',
    color: '#1a1a1a',
  },
  timestamp: {
    fontSize: 13,
    color: '#999',
    marginTop: 1,
  },
  body: {
    fontSize: 15,
    lineHeight: 22,
    color: '#1a1a1a',
    marginBottom: 8,
  },
  image: {
    width: '100%',
    borderRadius: 12,
    marginBottom: 8,
    backgroundColor: '#f0f0f0',
  },
  actions: {
    flexDirection: 'row',
    gap: 24,
    marginTop: 4,
  },
  actionButton: {
    flexDirection: 'row',
    alignItems: 'center',
  },
  actionText: {
    fontSize: 14,
    color: '#666',
  },
  liked: {
    fontSize: 14,
    color: '#e74c3c',
  },
});
```

### 12.6 Complete PostDetailScreen with Comments and Subscriptions

```typescript
// apps/mobile/src/screens/PostDetailScreen.tsx

import React, { useCallback, useState } from 'react';
import {
  View,
  Text,
  FlatList,
  TextInput,
  Pressable,
  KeyboardAvoidingView,
  Platform,
  StyleSheet,
  ActivityIndicator,
} from 'react-native';
import {
  usePostDetailQuery,
  usePostCommentsQuery,
  useCreateCommentMutation,
  useCommentAddedSubscription,
  PostCommentsDocument,
  type PostCommentsQuery,
} from '@acme/graphql/generated';
import { useApolloClient } from '@apollo/client';
import { PostCard } from '../components/PostCard';
import { CommentItem } from '../components/CommentItem';

const COMMENTS_PAGE_SIZE = 20;

interface Props {
  route: { params: { id: string } };
}

export function PostDetailScreen({ route }: Props) {
  const { id: postId } = route.params;
  const client = useApolloClient();
  const [commentText, setCommentText] = useState('');

  // Fetch post detail
  const { data: postData, loading: postLoading } = usePostDetailQuery({
    variables: { id: postId },
  });

  // Fetch comments with pagination
  const {
    data: commentsData,
    loading: commentsLoading,
    fetchMore,
    networkStatus,
  } = usePostCommentsQuery({
    variables: { postId, first: COMMENTS_PAGE_SIZE },
    notifyOnNetworkStatusChange: true,
  });

  const isLoadingMore = networkStatus === 3;

  // Subscribe to new comments
  useCommentAddedSubscription({
    variables: { postId },
    onData: ({ data: { data } }) => {
      if (!data?.commentAdded) return;
      const newComment = data.commentAdded;

      // Add the new comment to the cache
      client.cache.updateQuery<PostCommentsQuery>(
        {
          query: PostCommentsDocument,
          variables: { postId, first: COMMENTS_PAGE_SIZE },
        },
        (existing) => {
          if (!existing?.post) return existing;

          const alreadyExists = existing.post.comments.edges.some(
            (edge) => edge.node.id === newComment.id,
          );
          if (alreadyExists) return existing;

          return {
            ...existing,
            post: {
              ...existing.post,
              comments: {
                ...existing.post.comments,
                totalCount: existing.post.comments.totalCount + 1,
                edges: [
                  ...existing.post.comments.edges,
                  {
                    __typename: 'CommentEdge' as const,
                    cursor: newComment.id,
                    node: newComment,
                  },
                ],
              },
            },
          };
        },
      );
    },
  });

  // Create comment mutation
  const [createComment, { loading: submitting }] = useCreateCommentMutation();

  const handleSubmitComment = useCallback(async () => {
    if (!commentText.trim() || submitting) return;

    const { data } = await createComment({
      variables: {
        input: {
          postId,
          body: commentText.trim(),
        },
      },
    });

    if (data?.createComment.errors.length) {
      // Handle errors
      const error = data.createComment.errors[0];
      alert(error.message);
      return;
    }

    setCommentText('');
  }, [commentText, submitting, createComment, postId]);

  const handleLoadMoreComments = useCallback(() => {
    const pageInfo = commentsData?.post?.comments.pageInfo;
    if (!pageInfo?.hasNextPage || isLoadingMore) return;

    fetchMore({
      variables: { after: pageInfo.endCursor },
    });
  }, [commentsData, isLoadingMore, fetchMore]);

  if (postLoading && !postData) {
    return <ActivityIndicator style={{ flex: 1 }} />;
  }

  const post = postData?.post;
  if (!post) {
    return (
      <View style={styles.center}>
        <Text>Post not found</Text>
      </View>
    );
  }

  const comments =
    commentsData?.post?.comments.edges.map((edge) => edge.node) ?? [];
  const totalComments = commentsData?.post?.comments.totalCount ?? 0;

  return (
    <KeyboardAvoidingView
      style={styles.container}
      behavior={Platform.OS === 'ios' ? 'padding' : 'height'}
      keyboardVerticalOffset={90}
    >
      <FlatList
        data={comments}
        keyExtractor={(comment) => comment.id}
        ListHeaderComponent={
          <View>
            <PostCard post={post} />
            <View style={styles.commentsHeader}>
              <Text style={styles.commentsTitle}>
                Comments ({totalComments})
              </Text>
            </View>
          </View>
        }
        renderItem={({ item }) => <CommentItem comment={item} />}
        onEndReached={handleLoadMoreComments}
        onEndReachedThreshold={0.5}
        ListFooterComponent={
          isLoadingMore ? <ActivityIndicator style={{ padding: 16 }} /> : null
        }
        ListEmptyComponent={
          !commentsLoading ? (
            <Text style={styles.emptyText}>No comments yet</Text>
          ) : null
        }
      />

      {/* Comment input */}
      <View style={styles.inputContainer}>
        <TextInput
          style={styles.input}
          placeholder="Write a comment..."
          value={commentText}
          onChangeText={setCommentText}
          multiline
          maxLength={500}
        />
        <Pressable
          onPress={handleSubmitComment}
          disabled={!commentText.trim() || submitting}
          style={[
            styles.sendButton,
            (!commentText.trim() || submitting) && styles.sendButtonDisabled,
          ]}
        >
          <Text style={styles.sendButtonText}>
            {submitting ? '...' : 'Send'}
          </Text>
        </Pressable>
      </View>
    </KeyboardAvoidingView>
  );
}

const styles = StyleSheet.create({
  container: {
    flex: 1,
    backgroundColor: '#fff',
  },
  center: {
    flex: 1,
    justifyContent: 'center',
    alignItems: 'center',
  },
  commentsHeader: {
    padding: 16,
    borderBottomWidth: StyleSheet.hairlineWidth,
    borderBottomColor: '#e0e0e0',
  },
  commentsTitle: {
    fontSize: 16,
    fontWeight: '600',
    color: '#1a1a1a',
  },
  emptyText: {
    padding: 24,
    textAlign: 'center',
    color: '#999',
    fontSize: 15,
  },
  inputContainer: {
    flexDirection: 'row',
    alignItems: 'flex-end',
    padding: 12,
    borderTopWidth: StyleSheet.hairlineWidth,
    borderTopColor: '#e0e0e0',
    backgroundColor: '#fff',
  },
  input: {
    flex: 1,
    minHeight: 36,
    maxHeight: 100,
    borderWidth: 1,
    borderColor: '#ddd',
    borderRadius: 18,
    paddingHorizontal: 16,
    paddingVertical: 8,
    fontSize: 15,
    backgroundColor: '#f8f8f8',
  },
  sendButton: {
    marginLeft: 8,
    paddingHorizontal: 16,
    paddingVertical: 8,
    backgroundColor: '#007AFF',
    borderRadius: 18,
  },
  sendButtonDisabled: {
    opacity: 0.5,
  },
  sendButtonText: {
    color: '#fff',
    fontWeight: '600',
    fontSize: 15,
  },
});
```

### 12.7 Next.js App Router Integration

Apollo Client in Next.js App Router requires special handling because Server Components cannot use hooks, and the client needs to work across server and client boundaries.

```typescript
// apps/web/src/lib/apollo-server.ts
// For Server Components: use Apollo Client directly (no hooks)

import { ApolloClient, InMemoryCache, HttpLink } from '@apollo/client';
import { registerApolloClient } from '@apollo/experimental-nextjs-app-support';
import { typePolicies } from '@acme/graphql';

export const { getClient, query, PreloadQuery } = registerApolloClient(() => {
  return new ApolloClient({
    cache: new InMemoryCache({ typePolicies }),
    link: new HttpLink({
      uri: process.env.GRAPHQL_ENDPOINT,
      // Server-to-server: use internal URL, no auth needed if behind VPC
      headers: {
        'x-internal-token': process.env.INTERNAL_API_TOKEN ?? '',
      },
    }),
  });
});
```

```typescript
// apps/web/src/app/layout.tsx

import { ApolloWrapper } from './providers';

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        <ApolloWrapper>{children}</ApolloWrapper>
      </body>
    </html>
  );
}
```

```typescript
// apps/web/src/app/providers.tsx

'use client';

import React from 'react';
import { ApolloNextAppProvider } from '@apollo/experimental-nextjs-app-support';
import { createApolloClient } from '@acme/graphql';
import { useAuth } from '@clerk/nextjs';

function makeClient() {
  // For SSR: create a new client per request
  // (no auth token available during SSR)
  return createApolloClient({
    httpUrl: process.env.NEXT_PUBLIC_API_URL + '/graphql',
    wsUrl: process.env.NEXT_PUBLIC_WS_URL + '/graphql',
    getToken: async () => null,
    onAuthError: () => {},
    platform: 'web',
  });
}

export function ApolloWrapper({ children }: { children: React.ReactNode }) {
  return (
    <ApolloNextAppProvider makeClient={makeClient}>
      {children}
    </ApolloNextAppProvider>
  );
}
```

```typescript
// apps/web/src/app/post/[id]/page.tsx
// Server Component with GraphQL data fetching

import { Suspense } from 'react';
import { PreloadQuery } from '@/lib/apollo-server';
import { PostDetailDocument } from '@acme/graphql/generated';
import { PostDetailClient } from './PostDetailClient';

interface Props {
  params: Promise<{ id: string }>;
}

export default async function PostPage({ params }: Props) {
  const { id } = await params;

  return (
    <PreloadQuery
      query={PostDetailDocument}
      variables={{ id }}
    >
      <Suspense fallback={<PostDetailSkeleton />}>
        <PostDetailClient id={id} />
      </Suspense>
    </PreloadQuery>
  );
}

function PostDetailSkeleton() {
  return (
    <div className="animate-pulse space-y-4 p-6">
      <div className="h-8 bg-gray-200 rounded w-3/4" />
      <div className="h-4 bg-gray-200 rounded w-full" />
      <div className="h-4 bg-gray-200 rounded w-5/6" />
    </div>
  );
}
```

```typescript
// apps/web/src/app/post/[id]/PostDetailClient.tsx

'use client';

import { usePostDetailQuery } from '@acme/graphql/generated';

export function PostDetailClient({ id }: { id: string }) {
  // This reads from the cache that was pre-populated by PreloadQuery
  const { data, error } = usePostDetailQuery({
    variables: { id },
  });

  if (error) return <div>Error loading post</div>;
  if (!data?.post) return <div>Post not found</div>;

  const post = data.post;

  return (
    <article className="max-w-2xl mx-auto p-6">
      <header className="flex items-center gap-3 mb-6">
        <img
          src={post.author.avatarUrl ?? '/default-avatar.png'}
          alt={post.author.displayName}
          className="w-10 h-10 rounded-full"
        />
        <div>
          <p className="font-semibold">{post.author.displayName}</p>
          <time className="text-sm text-gray-500">
            {new Date(post.createdAt).toLocaleDateString()}
          </time>
        </div>
      </header>

      <div className="prose prose-lg">{post.body}</div>

      <footer className="mt-6 flex gap-6 text-gray-600">
        <span>{post.likesCount} likes</span>
        <span>{post.commentsCount} comments</span>
      </footer>
    </article>
  );
}
```

### 12.8 ESLint Configuration for GraphQL

```bash
npm install -D @graphql-eslint/eslint-plugin
```

```javascript
// .eslintrc.js (GraphQL files)

module.exports = {
  overrides: [
    {
      files: ['*.graphql'],
      parser: '@graphql-eslint/eslint-plugin',
      plugins: ['@graphql-eslint'],
      parserOptions: {
        operations: ['packages/graphql/src/operations/**/*.graphql'],
        schema: 'http://localhost:4000/graphql',
      },
      rules: {
        // Enforce naming conventions
        '@graphql-eslint/naming-convention': [
          'error',
          {
            types: 'PascalCase',
            FieldDefinition: 'camelCase',
            EnumValueDefinition: 'UPPER_CASE',
            InputValueDefinition: 'camelCase',
            FragmentDefinition: 'PascalCase',
            OperationDefinition: {
              style: 'PascalCase',
              forbiddenPrefixes: ['Query', 'Mutation', 'Subscription', 'Get'],
            },
          },
        ],
        // Require unique operation names
        '@graphql-eslint/unique-operation-name': 'error',
        // Require unique fragment names
        '@graphql-eslint/unique-fragment-name': 'error',
        // Prevent unused fragments
        '@graphql-eslint/no-unused-fragments': 'error',
        // Require selections on object types (prevent over-fetching)
        '@graphql-eslint/require-selections': ['error', { fieldName: 'id' }],
        // Limit query depth
        '@graphql-eslint/selection-set-depth': ['error', { maxDepth: 7 }],
        // Require descriptions on types (schema files only)
        '@graphql-eslint/require-description': 'off',
      },
    },
  ],
};
```

### 12.9 Testing GraphQL Components

```typescript
// __tests__/PostCard.test.tsx

import React from 'react';
import { render, fireEvent, waitFor } from '@testing-library/react-native';
import { MockedProvider, type MockedResponse } from '@apollo/client/testing';
import {
  LikePostDocument,
  type LikePostMutation,
  type LikePostMutationVariables,
} from '@acme/graphql/generated';
import { PostCard } from '../components/PostCard';

// Mock post data matching the PostCard fragment
const mockPost = {
  __typename: 'Post' as const,
  id: 'post_1',
  body: 'Hello, world!',
  status: 'PUBLISHED' as const,
  createdAt: '2026-04-07T12:00:00Z',
  author: {
    __typename: 'User' as const,
    id: 'user_1',
    username: 'alice',
    displayName: 'Alice',
    avatarUrl: 'https://example.com/avatar.jpg',
  },
  media: [],
  likesCount: 5,
  commentsCount: 3,
  viewerHasLiked: false,
};

// Mock the like mutation
const likeMock: MockedResponse<LikePostMutation, LikePostMutationVariables> = {
  request: {
    query: LikePostDocument,
    variables: { postId: 'post_1' },
  },
  result: {
    data: {
      __typename: 'Mutation',
      likePost: {
        __typename: 'LikePostPayload',
        post: {
          __typename: 'Post',
          id: 'post_1',
          likesCount: 6,
          viewerHasLiked: true,
        },
      },
    },
  },
};

function renderWithApollo(
  component: React.ReactElement,
  mocks: MockedResponse[] = [],
) {
  return render(
    <MockedProvider mocks={mocks} addTypename>
      {component}
    </MockedProvider>,
  );
}

describe('PostCard', () => {
  it('renders post content', () => {
    const { getByText } = renderWithApollo(<PostCard post={mockPost} />);

    expect(getByText('Hello, world!')).toBeTruthy();
    expect(getByText('Alice')).toBeTruthy();
    expect(getByText(/5/)).toBeTruthy(); // likes count
    expect(getByText(/3/)).toBeTruthy(); // comments count
  });

  it('handles like mutation', async () => {
    const { getByText } = renderWithApollo(
      <PostCard post={mockPost} />,
      [likeMock],
    );

    const likeButton = getByText(/♡ 5/);
    fireEvent.press(likeButton);

    // Optimistic update shows immediately
    await waitFor(() => {
      expect(getByText(/❤ 6/)).toBeTruthy();
    });
  });
});
```

### 12.10 Debugging Checklist

When things go wrong with GraphQL (and they will), here's where to look:

```
Query returns no data:
├── Check Apollo DevTools → Cache tab → is the data there?
├── Check the Network tab → did the request return data?
├── Check fetchPolicy → is it 'cache-only' when cache is empty?
├── Check the operation name → does it match the generated hook?
└── Check variables → is an ID null/undefined?

Cache shows stale data:
├── Check type policies → is the merge function correct?
├── Check keyFields → does every type have a unique identifier?
├── Check possibleTypes → are union types configured?
├── Try cache.evict() → manually remove the stale entity
└── Try refetch() → force a network request

Subscription doesn't fire:
├── Check WebSocket connection → Network tab → WS tab
├── Check connectionParams → is the auth token present?
├── Check server logs → is the subscription resolver running?
├── Check the variable → does postId match an existing entity?
└── Check skip condition → is the subscription being skipped?

TypeScript errors after schema change:
├── Run codegen → npm run codegen
├── Check schema.graphql → did the type change?
├── Check the operation → does it query a removed field?
├── Restart TypeScript server → Cmd+Shift+P → Restart TS Server
└── Delete generated folder and regenerate from scratch

Performance issues:
├── Check query depth → is it too deep?
├── Check list sizes → are you fetching 100+ items?
├── Check network tab → how large is the response?
├── Check server traces → is there an N+1 issue?
└── Enable persisted queries → reduce request size
```

---

## Summary

This chapter covered GraphQL end-to-end for mobile and web applications:

1. **When to use GraphQL:** Complex relational data, multiple client platforms, large teams. Don't use it for simple CRUD, small teams (use tRPC), or public APIs (use REST).

2. **Schema design:** Input types for mutations, payload types with errors, Relay connections for pagination, union types for polymorphic data. The schema is the contract -- treat it with the same rigor as your component API.

3. **Apollo Client:** Production setup with auth, error handling, retry, and WebSocket subscriptions. Cache normalization with type policies. Fetch policies tailored per-query. Optimistic updates for instant UI feedback.

4. **urql:** Lighter alternative when you don't need normalized caching. Exchange architecture for composable middleware. Use `@urql/exchange-graphcache` when you do need normalization.

5. **Codegen:** Non-negotiable. Without it, GraphQL is worse than REST. The `client-preset` generates typed hooks, document nodes, and fragment masking from your schema and operations.

6. **Fragments:** Colocate data requirements with components. Fragment masking enforces encapsulation. Compose small fragments into larger ones mirroring the component hierarchy.

7. **Pagination:** Relay-style cursor connections. Apollo's type policies handle merge logic. `fetchMore` with `after` cursor for infinite scroll.

8. **Subscriptions:** Use for low-latency real-time features. Fall back to polling for reliability. The `graphql-ws` protocol with automatic reconnection.

9. **Performance:** Persisted queries reduce request size and enable CDN caching. Query depth limits prevent abuse. DataLoader on the server solves N+1.

10. **Trade-offs:** The GraphQL tax is real -- schema maintenance, normalized cache complexity, tooling overhead, and the N+1 problem. Know the costs before committing. Start with tRPC in a monorepo if you can.

The patterns in this chapter compose into a complete data layer: typed queries with codegen, colocated fragments, cursor pagination, optimistic mutations, and real-time subscriptions. When set up properly, GraphQL provides a developer experience that's hard to match -- your components declare their data requirements, your cache keeps everything consistent, and your types flow from schema to screen. But "properly" is the operative word, and it requires the discipline and tooling investment this chapter describes.

Next, review Chapter 10 for the broader data fetching context, and Chapter 11 for how GraphQL's cache layer fits into the larger caching architecture of your application.
