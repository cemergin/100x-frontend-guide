# Part III — State, Data & Communication

> Managing state without losing your mind, fetching data without losing your users.

## Chapters

| Ch | Title | Lines | Difficulty | Key Topics |
|----|-------|-------|-----------|------------|
| [10](./09-state-management.md) | State Management at Scale | 737 | Intermediate to Advanced | Zustand, Jotai, Legend State, TanStack Query, Redux, React Hook Form, Zod, MMKV, finite states, type states, XState Store |
| [11](./10-data-fetching.md) | Data Fetching & Server Communication | 3,446 | Intermediate to Advanced | TanStack Query, tRPC, REST, GraphQL, WebSockets, SSE, API layer, retry, backoff, pagination, optimistic updates |
| [12](./11-caching.md) | Caching Strategies — Mobile & Web | 1,874 | Intermediate | MMKV, AsyncStorage, TanStack Query cache, HTTP caching, Cache-Control, ETag, stale-while-revalidate, CDN, edge caching, image caching |
| [13](./12-offline-realtime.md) | Offline-First & Real-Time Patterns | 2,198 | Advanced | Legend State, CRDT, WebSocket, offline sync, conflict resolution, NetInfo, background sync, queue-based mutations, optimistic UI |

## Reading Order

1. **Chapter 10** (State Management) first -- it introduces the three-category model that the rest of the part builds on.
2. **Chapter 11** (Data Fetching) next -- deep dive into server communication and the API layer.
3. **Chapter 12** (Caching) builds on both 10 and 11 -- caching touches state and data fetching equally.
4. **Chapter 13** (Offline & Real-Time) last -- it synthesizes everything into the hardest connectivity scenarios.

## Prerequisites

- Chapter 1 (React Native Internals) and Chapter 3 (Rendering Pipeline) for Chapter 10.
- Chapters 10-11 for Chapter 12.
- Chapters 10-12 for Chapter 13.

## What You'll Be Able to Do After This Part

- Separate server state, client state, and form state using the right tool for each category.
- Build a typed API layer with TanStack Query, tRPC, or GraphQL that handles retries, pagination, and optimistic updates.
- Design multi-layer cache architectures from device storage to CDN edge.
- Implement offline-first patterns with mutation queues, conflict resolution, and background sync.
- Add real-time features using WebSockets and CRDTs without breaking consistency.
