# Part III — State, Data & Communication

> Managing state without losing your mind, fetching data without losing your users.

## Chapters

| Ch | Title | Lines | Difficulty | Key Topics |
|----|-------|------:|-----------|------------|
| 9 | [State Management at Scale](./09-state-management.md) | 3,195 | Intermediate to Advanced | Zustand, Jotai, Legend State, TanStack Query, Redux, React Hook Form, Zod, MMKV, finite states, type states, XState Store |
| 10 | [Data Fetching & Server Communication](./10-data-fetching.md) | 3,458 | Intermediate to Advanced | TanStack Query, tRPC, REST, GraphQL, WebSockets, SSE, API layer, retry, backoff, pagination, optimistic updates |
| 11 | [Caching Strategies — Mobile & Web](./11-caching.md) | 1,886 | Intermediate | MMKV, AsyncStorage, TanStack Query cache, HTTP caching, Cache-Control, ETag, stale-while-revalidate, CDN, edge caching, image caching |
| 12 | [Offline-First & Real-Time Patterns](./12-offline-realtime.md) | 2,210 | Advanced | Legend State, CRDT, WebSocket, offline sync, conflict resolution, NetInfo, background sync, queue-based mutations, optimistic UI |
| 35 | [Forms at Scale — Multi-Step, Dynamic & Complex](./35-forms-at-scale.md) | 3,730 | Intermediate to Advanced | React Hook Form, Zod, multi-step wizards, dynamic forms, conditional fields, file uploads, form state machines, server-side validation, autosave |
| 41 | [Real-Time Collaboration & Multiplayer Features](./41-realtime-collaboration.md) | 4,344 | Advanced | WebSockets, Socket.io, Ably, Liveblocks, Yjs, CRDTs, presence, live cursors, collaborative editing, conflict resolution, Supabase Realtime |
| 42 | [Real-Time Transport — WebSockets, SSE & Live Data](./42-realtime-transport.md) | 3,585 | Intermediate to Advanced | WebSocket, Socket.io, SSE, EventSource, real-time feeds, chat, live updates, connection management, reconnection, heartbeat, TanStack Query integration |
| 46 | [GraphQL — When, Why & How](./46-graphql.md) | 4,082 | Intermediate to Advanced | GraphQL, Apollo Client, urql, Relay, schema design, queries, mutations, subscriptions, fragments, codegen, normalized cache, persisted queries, GraphQL vs REST vs tRPC |

## Reading Order

1. **Chapter 9** (State Management) first -- it introduces the three-category model that the rest of the part builds on.
2. **Chapter 10** (Data Fetching) next -- deep dive into server communication and the API layer.
3. **Chapter 11** (Caching) builds on both 9 and 10 -- caching touches state and data fetching equally.
4. **Chapter 12** (Offline & Real-Time) synthesizes 9-11 into the hardest connectivity scenarios.
5. **Chapter 35** (Forms at Scale) after Chapter 9 -- advanced form patterns built on state management foundations.
6. **Chapter 42** (Real-Time Transport) after Chapters 10-11 -- the transport layer for live data.
7. **Chapter 41** (Real-Time Collaboration) after Chapter 12 -- multiplayer features using CRDTs and presence.
8. **Chapter 46** (GraphQL) after Chapter 10 -- an alternative data layer with its own caching and codegen story.

## Prerequisites

- Chapters 1, 3 for Chapter 9 (State Management).
- Chapters 9-11 for Chapter 12 (Offline & Real-Time).
- Chapters 10, 0d for Chapter 35 (Forms at Scale).
- Chapters 11, 12 for Chapters 41 and 42 (Real-Time).
- Chapters 11, 34 for Chapter 46 (GraphQL).

## What You'll Be Able to Do After This Part

- Separate server state, client state, and form state using the right tool for each category.
- Build a typed API layer with TanStack Query, tRPC, or GraphQL that handles retries, pagination, and optimistic updates.
- Design multi-layer cache architectures from device storage to CDN edge.
- Implement offline-first patterns with mutation queues, conflict resolution, and background sync.
- Build complex multi-step forms with validation, autosave, and server-side error handling.
- Add real-time features using WebSockets, SSE, and CRDTs without breaking consistency.
- Implement collaborative editing with presence, live cursors, and conflict resolution.
- Choose between REST, tRPC, and GraphQL with clear reasoning for your use case.
