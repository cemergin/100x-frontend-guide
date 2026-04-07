<!--
  CHAPTER: 12
  TITLE: Offline-First & Real-Time Patterns
  PART: III — State, Data & Communication
  PREREQS: Chapters 9-11
  KEY_TOPICS: Legend State, CRDT, WebSocket, offline sync, conflict resolution, NetInfo, background sync, queue-based mutations, optimistic UI
  DIFFICULTY: Advanced
  UPDATED: 2026-04-07
-->

# Chapter 12: Offline-First & Real-Time Patterns

> **Part III — State, Data & Communication** | Prerequisites: Chapters 9-11 | Difficulty: Advanced

Here's a scenario that happens more often than you'd think: your user is on a subway, editing a document, adding items to a cart, or composing a message. The cell signal drops. What happens next determines whether your app is a tool they trust or a toy they tolerate.

Most apps handle this poorly. The request fails, an error toast appears, the user's work is lost or stuck in limbo, and they learn to never use the app without a strong signal. That's not a technology failure — it's a design failure. The technology to handle this well has existed for years. Most teams just never invest the time to implement it.

On the other end of the spectrum, you have real-time: multiple users editing the same document, live dashboards updating as data changes, chat messages appearing instantly across devices. Real-time feels magical when it works. When it doesn't — when messages arrive out of order, when edits conflict, when the UI flickers as it reconciles competing updates — it feels broken in a way that's worse than no real-time at all.

This chapter covers both ends of the connectivity spectrum. Offline-first and real-time are not separate concerns — they're the same problem viewed from different angles. Both require you to answer the same fundamental question: **when multiple sources of truth exist (the client, the server, other clients), how do you reconcile them?**

### In This Chapter
- The Offline-First Mindset — why network should be a progressive enhancement
- Connectivity Detection with NetInfo
- Legend State: Reactive State with Built-in Sync
- Queue-Based Offline Mutations
- Conflict Resolution Strategies
- CRDTs: Conflict-Free Replicated Data Types
- WebSocket Architecture for Real-Time
- Optimistic UI with Rollback
- Background Sync Patterns
- Building an Offline-First Architecture End-to-End

### Related Chapters
- [Ch 9: State Management at Scale] — state management foundations
- [Ch 10: Data Fetching & Server Communication] — API patterns that feed into sync
- [Ch 11: Caching Strategies] — cache-first patterns as the foundation for offline
- [Ch 13: Performance Optimization] — performance impact of sync strategies

---

## 1. THE OFFLINE-FIRST MINDSET

Most developers build apps with an implicit assumption: **the network is always available.** They fetch data on mount, submit forms with POST requests, and show error messages when things fail. The network is the happy path, and offline is an error state.

Offline-first inverts this. **The local device is the source of truth. The network is a sync mechanism.** The app works fully on the device, and whenever the network is available, it syncs changes to and from the server.

```
TRADITIONAL (NETWORK-FIRST):

  User Action → Network Request → Wait... → Success → Update UI
                                           → Failure → Show Error 😡

OFFLINE-FIRST:

  User Action → Update Local State → Update UI (instant ✓)
                     │
                     └──→ Queue Sync → When Online → Sync to Server
                                                        │
                                                        └──→ Resolve Conflicts if needed
```

### Why This Matters More Than You Think

It's not just about subways and airplane mode. Consider these real-world scenarios:

- **Flaky connections**: Most mobile connections aren't "online" or "offline" — they're somewhere in between. A user might have one bar of LTE that drops packets and has 2-second latency. If your app treats this as "online" and tries to fetch, the user experience is worse than if you'd just shown cached data.

- **Low-bandwidth scenarios**: A user in a developing country on 2G can technically reach your server, but the 800ms round trip makes your app feel broken. If the data is local, it feels fast.

- **Battery life**: Every network request costs battery. An app that minimizes network requests by working locally and batching syncs uses less power.

- **Server outages**: If your app depends on your server for every operation, a server outage means a dead app. An offline-first app keeps working.

The companies that build the best mobile experiences — Google Maps, Spotify, Notion — all embrace offline-first to varying degrees. It's not a niche concern; it's a quality bar.

---

## 2. CONNECTIVITY DETECTION WITH NETINFO

Before you can handle offline, you need to know when you're offline. This sounds simple. It isn't.

### The Problem with Simple Connectivity Checks

```typescript
// NAIVE: This tells you if the radio is on, not if the internet works
const isOnline = navigator.onLine; // Web
// This is essentially useless for real offline detection
```

A device can have Wi-Fi connected but no internet (captive portal, router without WAN). It can have cellular bars but packet loss so high that requests time out. "Connected" and "online" are not the same thing.

### NetInfo: The Right Way

```typescript
// hooks/useConnectivity.ts
import NetInfo, {
  NetInfoState,
  NetInfoStateType,
} from '@react-native-community/netinfo';
import { useEffect, useState, useCallback, useRef } from 'react';

interface ConnectivityState {
  isConnected: boolean;
  isInternetReachable: boolean;
  connectionType: NetInfoStateType;
  connectionQuality: 'good' | 'poor' | 'offline';
  details: {
    isConnectionExpensive: boolean;
    cellularGeneration: string | null;
    strength: number | null;
  };
}

export function useConnectivity(): ConnectivityState {
  const [state, setState] = useState<ConnectivityState>({
    isConnected: true,
    isInternetReachable: true,
    connectionType: NetInfoStateType.unknown,
    connectionQuality: 'good',
    details: {
      isConnectionExpensive: false,
      cellularGeneration: null,
      strength: null,
    },
  });

  useEffect(() => {
    const unsubscribe = NetInfo.addEventListener((netState: NetInfoState) => {
      const isConnected = netState.isConnected ?? false;
      const isInternetReachable = netState.isInternetReachable ?? false;

      // Determine connection quality
      let connectionQuality: 'good' | 'poor' | 'offline' = 'offline';
      if (isConnected && isInternetReachable) {
        if (
          netState.type === NetInfoStateType.cellular &&
          netState.details?.cellularGeneration === '2g'
        ) {
          connectionQuality = 'poor';
        } else if (
          netState.type === NetInfoStateType.cellular &&
          netState.details?.cellularGeneration === '3g'
        ) {
          connectionQuality = 'poor';
        } else {
          connectionQuality = 'good';
        }
      }

      setState({
        isConnected,
        isInternetReachable,
        connectionType: netState.type,
        connectionQuality,
        details: {
          isConnectionExpensive: netState.details?.isConnectionExpensive ?? false,
          cellularGeneration:
            netState.type === NetInfoStateType.cellular
              ? netState.details?.cellularGeneration ?? null
              : null,
          strength: null,
        },
      });
    });

    return unsubscribe;
  }, []);

  return state;
}
```

### Integrating NetInfo with TanStack Query

This is crucial. TanStack Query needs to know about connectivity to pause/resume queries and mutations:

```typescript
// lib/query-online-manager.ts
import NetInfo from '@react-native-community/netinfo';
import { onlineManager, focusManager } from '@tanstack/react-query';
import { AppState, Platform } from 'react-native';

// Tell TanStack Query about network state
export function setupQueryNetworkIntegration() {
  // Online/offline detection
  onlineManager.setEventListener((setOnline) => {
    return NetInfo.addEventListener((state) => {
      const online = Boolean(state.isConnected && state.isInternetReachable);
      setOnline(online);
    });
  });

  // App focus detection (refetch on app foreground)
  focusManager.setEventListener((setFocused) => {
    const subscription = AppState.addEventListener('change', (status) => {
      if (Platform.OS !== 'web') {
        setFocused(status === 'active');
      }
    });

    return () => subscription.remove();
  });
}
```

When TanStack Query knows it's offline:
- **Queries**: Don't attempt network requests. Serve from cache only.
- **Mutations**: Queue them. When connectivity returns, replay in order.

### The Connectivity Banner Pattern

Users should know when they're offline — but not in an alarming way:

```typescript
// components/ConnectivityBanner.tsx
import { useConnectivity } from '../hooks/useConnectivity';
import Animated, {
  useAnimatedStyle,
  withTiming,
  useSharedValue,
} from 'react-native-reanimated';
import { useEffect } from 'react';

export function ConnectivityBanner() {
  const { isConnected, isInternetReachable, connectionQuality } = useConnectivity();
  const translateY = useSharedValue(-50);

  const isOffline = !isConnected || !isInternetReachable;

  useEffect(() => {
    translateY.value = withTiming(isOffline ? 0 : -50, { duration: 300 });
  }, [isOffline]);

  const animatedStyle = useAnimatedStyle(() => ({
    transform: [{ translateY: translateY.value }],
  }));

  return (
    <Animated.View
      style={[
        {
          position: 'absolute',
          top: 0,
          left: 0,
          right: 0,
          backgroundColor: connectionQuality === 'poor' ? '#F59E0B' : '#EF4444',
          padding: 8,
          alignItems: 'center',
          zIndex: 1000,
        },
        animatedStyle,
      ]}
    >
      <Text style={{ color: 'white', fontSize: 13, fontWeight: '600' }}>
        {connectionQuality === 'poor'
          ? 'Slow connection — showing cached data'
          : 'No connection — changes will sync when online'}
      </Text>
    </Animated.View>
  );
}
```

---

## 3. LEGEND STATE: REACTIVE STATE WITH BUILT-IN SYNC

Legend State is a state management library that was designed from the ground up for the kind of reactive, sync-capable state that offline-first apps need. While you can build offline-first with Zustand + TanStack Query (and we showed how in Chapter 11), Legend State provides sync primitives as first-class features.

### Why Legend State for Sync-Heavy Apps

```
┌──────────────────────────────────────────────────┐
│  Zustand + TanStack Query                         │
│  Good for: Apps with clear separation between     │
│  client state and server state                    │
│  You build: Your own sync layer                   │
│  Tradeoff: More control, more work                │
│                                                    │
│  Legend State                                      │
│  Good for: Apps where local state and server      │
│  state are deeply intertwined                     │
│  Built-in: Persistence, sync, conflict resolution │
│  Tradeoff: Less flexibility, more magic           │
└──────────────────────────────────────────────────┘
```

### Legend State Basics

```typescript
// store/todos.ts
import { observable, computed } from '@legendapp/state';
import { ObservablePersistMMKV } from '@legendapp/state/persist-plugins/mmkv';
import { synced } from '@legendapp/state/sync';
import { syncedCrud } from '@legendapp/state/sync-plugins/crud';

// Observable state with automatic persistence and sync
export const todos$ = observable(
  syncedCrud({
    // CRUD operations that sync with your backend
    list: async () => {
      const response = await fetch('/api/todos');
      return response.json();
    },
    create: async (todo) => {
      const response = await fetch('/api/todos', {
        method: 'POST',
        body: JSON.stringify(todo),
      });
      return response.json();
    },
    update: async (todo) => {
      const response = await fetch(`/api/todos/${todo.id}`, {
        method: 'PUT',
        body: JSON.stringify(todo),
      });
      return response.json();
    },
    delete: async (id) => {
      await fetch(`/api/todos/${id}`, { method: 'DELETE' });
    },
    // Persistence configuration
    persist: {
      name: 'todos',
      plugin: ObservablePersistMMKV,
    },
    // Retry failed operations
    retry: {
      infinite: true,
      backoff: 'exponential',
      maxDelay: 30000,
    },
    // Handle being offline
    offlineBehavior: 'retry',
  })
);
```

### Using Legend State in Components

```typescript
// components/TodoList.tsx
import { observer } from '@legendapp/state/react';
import { todos$ } from '../store/todos';
import { For } from '@legendapp/state/react';

const TodoList = observer(function TodoList() {
  return (
    <FlatList
      data={todos$.get()}
      renderItem={({ item }) => (
        <TodoItem todo={item} />
      )}
    />
  );
});

const TodoItem = observer(function TodoItem({ todo }: { todo: Todo }) {
  const handleToggle = () => {
    // This immediately updates the local state
    // AND queues a sync to the server
    todos$[todo.id].completed.set(!todo.completed);
  };

  const handleDelete = () => {
    todos$[todo.id].delete();
  };

  return (
    <Pressable onPress={handleToggle}>
      <View style={styles.todoItem}>
        <Checkbox checked={todo.completed} />
        <Text style={todo.completed ? styles.completed : styles.active}>
          {todo.title}
        </Text>
        <Pressable onPress={handleDelete}>
          <TrashIcon />
        </Pressable>
      </View>
    </Pressable>
  );
});
```

### Legend State's Fine-Grained Reactivity

One of Legend State's key advantages is its proxy-based reactivity system. Unlike Zustand or Redux where a state change re-renders every subscriber, Legend State only re-renders components that access the specific property that changed:

```typescript
// This component ONLY re-renders when todo.title changes
// Not when todo.completed changes, not when other todos change
const TodoTitle = observer(function TodoTitle({ todoId }: { todoId: string }) {
  const title = todos$[todoId].title.get();
  return <Text>{title}</Text>;
});

// Compare with Zustand, where you'd need careful selector memoization:
// const title = useTodoStore(state => state.todos[todoId].title);
// Works, but the store still diffs the entire selector output on every state change
```

### Legend State Sync with Timestamps

For conflict resolution, Legend State can use timestamps to determine which change wins:

```typescript
import { syncedCrud } from '@legendapp/state/sync-plugins/crud';

export const documents$ = observable(
  syncedCrud({
    list: () => api.listDocuments(),
    create: (doc) => api.createDocument(doc),
    update: (doc) => api.updateDocument(doc),
    delete: (id) => api.deleteDocument(id),

    // Timestamp-based conflict resolution
    fieldUpdatedAt: 'updatedAt', // Server returns this field
    fieldCreatedAt: 'createdAt',

    // When there's a conflict, the most recently updated version wins
    // This is "Last Write Wins" — the simplest strategy
    changesSince: 'last-sync',

    persist: {
      name: 'documents',
      plugin: ObservablePersistMMKV,
    },
  })
);
```

---

## 4. QUEUE-BASED OFFLINE MUTATIONS

Whether you use Legend State's built-in sync or build your own, the fundamental pattern for offline mutations is a **persistent queue**. Here's a production-grade implementation:

### The Mutation Queue

```typescript
// lib/mutation-queue.ts
import { MMKV } from 'react-native-mmkv';
import NetInfo from '@react-native-community/netinfo';

const storage = new MMKV({ id: 'mutation-queue' });
const QUEUE_KEY = 'pending_mutations';

export interface PendingMutation {
  id: string;
  type: string;              // e.g., 'CREATE_TODO', 'UPDATE_PROFILE'
  payload: unknown;
  createdAt: number;
  retryCount: number;
  maxRetries: number;
  priority: 'high' | 'normal' | 'low';
  // For optimistic UI rollback
  optimisticUpdate?: {
    queryKey: unknown[];
    previousData: unknown;
  };
}

class MutationQueue {
  private isProcessing = false;
  private listeners: Set<(queue: PendingMutation[]) => void> = new Set();

  getQueue(): PendingMutation[] {
    const raw = storage.getString(QUEUE_KEY);
    return raw ? JSON.parse(raw) : [];
  }

  private saveQueue(queue: PendingMutation[]): void {
    storage.set(QUEUE_KEY, JSON.stringify(queue));
    this.listeners.forEach(listener => listener(queue));
  }

  enqueue(mutation: Omit<PendingMutation, 'id' | 'createdAt' | 'retryCount'>): string {
    const id = `mut_${Date.now()}_${Math.random().toString(36).slice(2)}`;
    const queue = this.getQueue();

    queue.push({
      ...mutation,
      id,
      createdAt: Date.now(),
      retryCount: 0,
    });

    // Sort by priority
    queue.sort((a, b) => {
      const priorityOrder = { high: 0, normal: 1, low: 2 };
      return priorityOrder[a.priority] - priorityOrder[b.priority];
    });

    this.saveQueue(queue);
    this.processQueue(); // Try to process immediately

    return id;
  }

  dequeue(id: string): void {
    const queue = this.getQueue().filter(m => m.id !== id);
    this.saveQueue(queue);
  }

  subscribe(listener: (queue: PendingMutation[]) => void): () => void {
    this.listeners.add(listener);
    return () => this.listeners.delete(listener);
  }

  async processQueue(): Promise<void> {
    if (this.isProcessing) return;

    const netState = await NetInfo.fetch();
    if (!netState.isConnected || !netState.isInternetReachable) return;

    this.isProcessing = true;

    try {
      const queue = this.getQueue();

      for (const mutation of queue) {
        try {
          await this.executeMutation(mutation);
          this.dequeue(mutation.id);
        } catch (error) {
          if (mutation.retryCount >= mutation.maxRetries) {
            console.error(`Mutation ${mutation.id} exceeded max retries, removing`);
            this.dequeue(mutation.id);
            // Rollback optimistic update if present
            if (mutation.optimisticUpdate) {
              this.rollback(mutation);
            }
          } else {
            // Increment retry count
            const updated = this.getQueue().map(m =>
              m.id === mutation.id
                ? { ...m, retryCount: m.retryCount + 1 }
                : m
            );
            this.saveQueue(updated);
          }
          break; // Stop processing on failure (preserve order)
        }
      }
    } finally {
      this.isProcessing = false;
    }
  }

  private async executeMutation(mutation: PendingMutation): Promise<void> {
    // Map mutation types to API calls
    const handlers: Record<string, (payload: any) => Promise<void>> = {
      CREATE_TODO: (p) => api.createTodo(p),
      UPDATE_TODO: (p) => api.updateTodo(p.id, p.data),
      DELETE_TODO: (p) => api.deleteTodo(p.id),
      UPDATE_PROFILE: (p) => api.updateProfile(p),
      // Add more mutation types as needed
    };

    const handler = handlers[mutation.type];
    if (!handler) {
      throw new Error(`Unknown mutation type: ${mutation.type}`);
    }

    await handler(mutation.payload);
  }

  private rollback(mutation: PendingMutation): void {
    if (!mutation.optimisticUpdate) return;
    // This would integrate with your query client to rollback
    // the optimistic update
    console.warn(`Rolling back optimistic update for ${mutation.id}`);
  }
}

export const mutationQueue = new MutationQueue();
```

### Connecting the Queue to Network Changes

```typescript
// lib/setup-offline.ts
import NetInfo from '@react-native-community/netinfo';
import { mutationQueue } from './mutation-queue';

export function setupOfflineSupport() {
  // Process queue when connectivity changes
  NetInfo.addEventListener((state) => {
    if (state.isConnected && state.isInternetReachable) {
      console.log('Connectivity restored, processing mutation queue');
      mutationQueue.processQueue();
    }
  });

  // Also try processing on app foreground
  AppState.addEventListener('change', (status) => {
    if (status === 'active') {
      mutationQueue.processQueue();
    }
  });
}
```

### Using the Queue in Components

```typescript
// hooks/useOfflineCreateTodo.ts
import { useMutation, useQueryClient } from '@tanstack/react-query';
import { onlineManager } from '@tanstack/react-query';
import { mutationQueue } from '../lib/mutation-queue';

export function useCreateTodo() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: async (input: CreateTodoInput) => {
      if (!onlineManager.isOnline()) {
        // Queue for later processing
        const mutationId = mutationQueue.enqueue({
          type: 'CREATE_TODO',
          payload: input,
          maxRetries: 5,
          priority: 'normal',
          optimisticUpdate: {
            queryKey: ['todos'],
            previousData: queryClient.getQueryData(['todos']),
          },
        });

        // Return an optimistic result
        return {
          ...input,
          id: mutationId, // Use mutation ID as temp ID
          createdAt: new Date().toISOString(),
          _pending: true,
        } as Todo;
      }

      // Online: execute directly
      return api.createTodo(input);
    },

    onMutate: async (input) => {
      await queryClient.cancelQueries({ queryKey: ['todos'] });
      const previous = queryClient.getQueryData<Todo[]>(['todos']);

      // Optimistic update
      queryClient.setQueryData<Todo[]>(['todos'], (old = []) => [
        {
          ...input,
          id: `optimistic-${Date.now()}`,
          createdAt: new Date().toISOString(),
          _pending: !onlineManager.isOnline(),
        } as Todo,
        ...old,
      ]);

      return { previous };
    },

    onError: (err, input, context) => {
      // Only rollback if we're online (offline mutations are queued)
      if (onlineManager.isOnline() && context?.previous) {
        queryClient.setQueryData(['todos'], context.previous);
      }
    },

    onSettled: () => {
      if (onlineManager.isOnline()) {
        queryClient.invalidateQueries({ queryKey: ['todos'] });
      }
    },
  });
}
```

### Visual Indicators for Pending Mutations

Users should know which changes haven't synced yet:

```typescript
// components/TodoItem.tsx
function TodoItem({ todo }: { todo: Todo & { _pending?: boolean } }) {
  return (
    <View style={[styles.todoItem, todo._pending && styles.pending]}>
      <Text style={styles.title}>{todo.title}</Text>
      {todo._pending && (
        <View style={styles.pendingBadge}>
          <CloudOffIcon size={14} color="#9CA3AF" />
          <Text style={styles.pendingText}>Waiting to sync</Text>
        </View>
      )}
    </View>
  );
}
```

---

## 5. CONFLICT RESOLUTION STRATEGIES

When a user modifies data offline and the same data was modified on the server (by another user, an admin, or a background process), you have a **conflict**. How you resolve it depends on your domain.

### Strategy 1: Last Write Wins (LWW)

The simplest strategy. Whoever wrote last (by timestamp) wins.

```typescript
// Server-side conflict resolution
async function resolveConflict(
  clientData: Record<string, any>,
  serverData: Record<string, any>
): Promise<Record<string, any>> {
  if (clientData.updatedAt > serverData.updatedAt) {
    return clientData; // Client wins
  }
  return serverData; // Server wins
}
```

**When to use:** Simple data where losing an edit is acceptable. User preferences, settings, status updates.

**When NOT to use:** Collaborative editing, financial data, anything where losing an edit is unacceptable.

**The timestamp problem:** Clocks on mobile devices aren't reliable. A user can change their system clock. A device's clock might be minutes off from the server. If you use LWW, use server-assigned timestamps:

```typescript
// Client sends the mutation with its local timestamp
// Server compares and assigns the authoritative timestamp
app.put('/api/documents/:id', async (req, res) => {
  const { data, clientTimestamp } = req.body;
  const serverDoc = await db.documents.findUnique({ where: { id: req.params.id } });

  // If server version is newer than what the client saw, it's a conflict
  if (serverDoc.updatedAt > clientTimestamp) {
    // Last Write Wins: overwrite anyway, but log the conflict
    await logConflict(serverDoc, data);
  }

  const updated = await db.documents.update({
    where: { id: req.params.id },
    data: { ...data, updatedAt: new Date() }, // Server timestamp
  });

  res.json(updated);
});
```

### Strategy 2: Field-Level Merge

Instead of replacing the entire document, merge at the field level:

```typescript
// Field-level merge: each field is resolved independently
function fieldLevelMerge(
  base: Record<string, any>,    // Last known common version
  client: Record<string, any>,  // Client's version
  server: Record<string, any>   // Server's current version
): Record<string, any> {
  const result: Record<string, any> = { ...server };

  for (const key of Object.keys(client)) {
    const baseValue = base[key];
    const clientValue = client[key];
    const serverValue = server[key];

    // Client changed this field, server didn't
    if (clientValue !== baseValue && serverValue === baseValue) {
      result[key] = clientValue;
    }
    // Server changed this field, client didn't
    else if (serverValue !== baseValue && clientValue === baseValue) {
      result[key] = serverValue;
    }
    // Both changed this field (true conflict)
    else if (clientValue !== baseValue && serverValue !== baseValue) {
      if (clientValue === serverValue) {
        result[key] = clientValue; // Same change, no conflict
      } else {
        // True conflict: pick a strategy per field
        result[key] = serverValue; // Default: server wins for conflicting fields
        // Could also: prompt user, merge strings, etc.
      }
    }
  }

  return result;
}
```

**When to use:** Forms, profiles, documents with distinct fields that different users might edit.

### Strategy 3: Server Wins / Client Wins

Simple policies that avoid complexity:

```typescript
// Server-always-wins: client changes are overwritten on next sync
// Good for: data controlled by admins (pricing, feature flags)
async function syncWithServerWins(localData: Data[], serverData: Data[]): Promise<Data[]> {
  // Server data is the truth, local data is discarded
  return serverData;
}

// Client-always-wins: client changes overwrite server on sync
// Good for: personal data (notes, preferences) on a single device
async function syncWithClientWins(localData: Data[]): Promise<Data[]> {
  await api.bulkUpdate(localData);
  return localData;
}
```

### Strategy 4: Operational Transform (OT)

Used by Google Docs. Instead of syncing state, sync **operations** (insert character at position 5, delete characters 10-15):

```typescript
// Operational Transform example (simplified)
type Operation =
  | { type: 'insert'; position: number; text: string }
  | { type: 'delete'; position: number; length: number };

function transformOperation(
  op1: Operation, // Operation that was applied first
  op2: Operation  // Operation that needs to be transformed
): Operation {
  if (op1.type === 'insert' && op2.type === 'insert') {
    if (op2.position >= op1.position) {
      return { ...op2, position: op2.position + op1.text.length };
    }
    return op2;
  }

  if (op1.type === 'insert' && op2.type === 'delete') {
    if (op2.position >= op1.position) {
      return { ...op2, position: op2.position + op1.text.length };
    }
    return op2;
  }

  // ... more cases for delete+insert, delete+delete
  return op2;
}
```

OT is powerful but complex. Unless you're building a collaborative text editor, you probably don't need it. And if you are building a collaborative text editor, use a library (Yjs, Automerge) instead of implementing OT from scratch.

### Strategy 5: User-Prompted Resolution

Sometimes the right answer is to ask the user:

```typescript
// components/ConflictResolver.tsx
function ConflictResolver({
  localVersion,
  serverVersion,
  onResolve,
}: ConflictResolverProps) {
  return (
    <Modal visible={true}>
      <View style={styles.container}>
        <Text style={styles.title}>This item was modified while you were offline</Text>

        <View style={styles.comparison}>
          <View style={styles.version}>
            <Text style={styles.versionLabel}>Your version</Text>
            <Text style={styles.versionDate}>
              {formatDate(localVersion.updatedAt)}
            </Text>
            <DiffView data={localVersion} />
          </View>

          <View style={styles.version}>
            <Text style={styles.versionLabel}>Server version</Text>
            <Text style={styles.versionDate}>
              {formatDate(serverVersion.updatedAt)}
            </Text>
            <DiffView data={serverVersion} />
          </View>
        </View>

        <View style={styles.actions}>
          <Button
            title="Keep mine"
            onPress={() => onResolve(localVersion)}
          />
          <Button
            title="Keep server"
            onPress={() => onResolve(serverVersion)}
          />
          <Button
            title="Merge both"
            onPress={() => onResolve(fieldLevelMerge(localVersion, serverVersion))}
          />
        </View>
      </View>
    </Modal>
  );
}
```

### Choosing a Strategy

```
┌─────────────────────────────────────────────────────────────┐
│  CONFLICT RESOLUTION DECISION MATRIX                         │
│                                                              │
│  Single user, single device?                                 │
│  → Client Wins (no conflicts possible)                       │
│                                                              │
│  Multiple users, but edits don't overlap?                    │
│  → Last Write Wins (simple, usually fine)                    │
│                                                              │
│  Multiple users editing same records, different fields?      │
│  → Field-Level Merge                                         │
│                                                              │
│  Multiple users editing same text/content?                   │
│  → CRDTs or Operational Transform                            │
│                                                              │
│  Business-critical data where loss is unacceptable?          │
│  → User-Prompted Resolution                                  │
│                                                              │
│  Admin-controlled data + user-created data?                  │
│  → Server Wins for admin data, Client Wins for user data     │
└─────────────────────────────────────────────────────────────┘
```

---

## 6. CRDTs: CONFLICT-FREE REPLICATED DATA TYPES

CRDTs are data structures that can be modified independently on different replicas and merged automatically without conflicts. They're the holy grail of distributed state — if you pick the right CRDT for your use case.

### The Core Idea

A CRDT guarantees that if two replicas have seen the same set of operations (in any order), they will converge to the same state. No conflict resolution needed. No coordination between replicas. It's mathematically guaranteed.

```
Replica A:                    Replica B:
  counter = 0                   counter = 0
  increment()                   increment()
  increment()                   increment()
  counter = 2                   increment()
                                counter = 3

  After merge:                  After merge:
  counter = 5                   counter = 5
  ✓ Same result!                ✓ Same result!
```

### Common CRDT Types

```
┌─────────────────────────────────────────────────────────────┐
│  G-Counter (Grow-Only Counter)                               │
│  Each replica tracks its own increments                      │
│  Merge: sum of all replica counts                            │
│  Use for: Like counts, view counts, event counters           │
│                                                              │
│  PN-Counter (Positive-Negative Counter)                      │
│  Two G-Counters: one for increments, one for decrements      │
│  Merge: sum of increments - sum of decrements                │
│  Use for: Cart quantity, upvotes/downvotes                   │
│                                                              │
│  G-Set (Grow-Only Set)                                       │
│  Elements can be added but never removed                     │
│  Merge: union of all elements                                │
│  Use for: Tag lists, seen-message IDs                        │
│                                                              │
│  OR-Set (Observed-Remove Set)                                │
│  Elements can be added and removed                           │
│  Merge: add wins over concurrent remove                      │
│  Use for: Shopping carts, favorite lists, todo lists         │
│                                                              │
│  LWW-Register (Last-Writer-Wins Register)                    │
│  Single value with timestamp                                 │
│  Merge: highest timestamp wins                               │
│  Use for: Single-value fields (name, email, status)          │
│                                                              │
│  RGA (Replicated Growable Array) / Y.Array                   │
│  Ordered list with insert/delete operations                  │
│  Merge: preserves all inserts in consistent order            │
│  Use for: Collaborative lists, document text                 │
└─────────────────────────────────────────────────────────────┘
```

### Implementing a Simple G-Counter

```typescript
// lib/crdt/g-counter.ts

interface GCounter {
  // Map of replicaId → count
  counts: Record<string, number>;
}

function createGCounter(replicaId: string): GCounter {
  return { counts: { [replicaId]: 0 } };
}

function increment(counter: GCounter, replicaId: string): GCounter {
  return {
    counts: {
      ...counter.counts,
      [replicaId]: (counter.counts[replicaId] || 0) + 1,
    },
  };
}

function value(counter: GCounter): number {
  return Object.values(counter.counts).reduce((sum, n) => sum + n, 0);
}

function merge(a: GCounter, b: GCounter): GCounter {
  const allReplicas = new Set([
    ...Object.keys(a.counts),
    ...Object.keys(b.counts),
  ]);

  const counts: Record<string, number> = {};
  for (const replica of allReplicas) {
    counts[replica] = Math.max(
      a.counts[replica] || 0,
      b.counts[replica] || 0
    );
  }

  return { counts };
}

// Usage
const deviceA = createGCounter('device-a');
const deviceB = createGCounter('device-b');

const a1 = increment(deviceA, 'device-a'); // A: 1, total: 1
const a2 = increment(a1, 'device-a');       // A: 2, total: 2

const b1 = increment(deviceB, 'device-b'); // B: 1, total: 1
const b2 = increment(b1, 'device-b');       // B: 2, total: 2
const b3 = increment(b2, 'device-b');       // B: 3, total: 3

const merged = merge(a2, b3);
console.log(value(merged)); // 5 — correct!
```

### Using Yjs for Real-World CRDTs

For production use, don't implement CRDTs from scratch. Use Yjs:

```typescript
// lib/collaborative-doc.ts
import * as Y from 'yjs';
import { WebsocketProvider } from 'y-websocket';

// Create a Yjs document
const ydoc = new Y.Doc();

// Get a shared type (like a collaborative text field)
const ytext = ydoc.getText('content');
const ymap = ydoc.getMap('metadata');
const yarray = ydoc.getArray('items');

// Connect to a WebSocket server for real-time sync
const provider = new WebsocketProvider(
  'wss://your-server.com',
  'document-123',
  ydoc
);

// Local changes are automatically synced
ytext.insert(0, 'Hello ');
ytext.insert(6, 'World');
// ytext.toString() === 'Hello World'

// Observe changes from other clients
ytext.observe((event) => {
  console.log('Text changed:', ytext.toString());
  // Update your React state / re-render
});

// For offline support: persist Yjs state to MMKV
import { MMKV } from 'react-native-mmkv';
const storage = new MMKV({ id: 'yjs-docs' });

// Save state periodically
function persistYDoc(docId: string, ydoc: Y.Doc) {
  const state = Y.encodeStateAsUpdate(ydoc);
  storage.set(`yjs:${docId}`, Buffer.from(state).toString('base64'));
}

// Restore state on app start
function restoreYDoc(docId: string, ydoc: Y.Doc) {
  const encoded = storage.getString(`yjs:${docId}`);
  if (encoded) {
    const state = Uint8Array.from(Buffer.from(encoded, 'base64'));
    Y.applyUpdate(ydoc, state);
  }
}
```

### When to Use CRDTs vs Simpler Approaches

CRDTs add complexity. Use them when you need them, not because they're cool:

```
Use CRDTs when:
  - Multiple users edit the same content simultaneously
  - Offline edits must never be lost
  - You need real-time collaboration (Google Docs-like)
  - You need decentralized sync (no server authority)

Use simpler approaches when:
  - Single user per device (most mobile apps)
  - Edits are at the record level, not field/character level
  - You can afford to lose an offline edit occasionally
  - Server is the clear authority (client-server, not peer-to-peer)
```

---

## 7. WEBSOCKET ARCHITECTURE FOR REAL-TIME

When you need data to update in real-time — chat messages, live scores, collaborative editing, notifications — you need a persistent connection between client and server. WebSockets are the standard solution.

### WebSocket vs Polling vs SSE

```
┌──────────────────────────────────────────────────────────┐
│  POLLING                                                  │
│  Client → Server (every N seconds)                       │
│                                                           │
│  Pros: Simple, works everywhere, stateless server        │
│  Cons: Latency (up to N seconds), wasted requests,      │
│        battery drain on mobile                           │
│  Use for: Low-frequency updates (every 30s+)             │
│                                                           │
│  LONG POLLING                                             │
│  Client → Server (holds connection until data available) │
│                                                           │
│  Pros: Lower latency than polling, works everywhere      │
│  Cons: Complex error handling, still one-directional     │
│  Use for: Medium-frequency updates, fallback for WS      │
│                                                           │
│  SERVER-SENT EVENTS (SSE)                                 │
│  Server → Client (persistent, one-directional)           │
│                                                           │
│  Pros: Simple, auto-reconnect, works with HTTP/2         │
│  Cons: One direction only (server → client)              │
│  Use for: Notifications, live feeds, stock tickers       │
│                                                           │
│  WEBSOCKET                                                │
│  Client ↔ Server (persistent, bi-directional)            │
│                                                           │
│  Pros: Low latency, bi-directional, efficient            │
│  Cons: More complex server, connection management        │
│  Use for: Chat, collaboration, gaming, live dashboards   │
└──────────────────────────────────────────────────────────┘
```

### Building a WebSocket Client for React Native

```typescript
// lib/websocket.ts
import NetInfo from '@react-native-community/netinfo';

type MessageHandler = (message: any) => void;

interface WebSocketConfig {
  url: string;
  protocols?: string[];
  reconnectInterval?: number;
  maxReconnectInterval?: number;
  reconnectBackoffMultiplier?: number;
  heartbeatInterval?: number;
  onConnect?: () => void;
  onDisconnect?: (reason: string) => void;
  onError?: (error: Event) => void;
}

export class ReconnectingWebSocket {
  private ws: WebSocket | null = null;
  private config: Required<WebSocketConfig>;
  private currentReconnectInterval: number;
  private reconnectTimer: ReturnType<typeof setTimeout> | null = null;
  private heartbeatTimer: ReturnType<typeof setInterval> | null = null;
  private messageHandlers: Map<string, Set<MessageHandler>> = new Map();
  private isManualClose = false;
  private messageQueue: string[] = []; // Queue messages while disconnected

  constructor(config: WebSocketConfig) {
    this.config = {
      protocols: [],
      reconnectInterval: 1000,
      maxReconnectInterval: 30000,
      reconnectBackoffMultiplier: 2,
      heartbeatInterval: 30000,
      onConnect: () => {},
      onDisconnect: () => {},
      onError: () => {},
      ...config,
    };
    this.currentReconnectInterval = this.config.reconnectInterval;
  }

  connect(): void {
    if (this.ws?.readyState === WebSocket.OPEN) return;

    this.isManualClose = false;

    try {
      this.ws = new WebSocket(this.config.url, this.config.protocols);

      this.ws.onopen = () => {
        console.log('WebSocket connected');
        this.currentReconnectInterval = this.config.reconnectInterval;
        this.config.onConnect();
        this.startHeartbeat();
        this.flushMessageQueue();
      };

      this.ws.onmessage = (event: MessageEvent) => {
        try {
          const message = JSON.parse(event.data);

          // Handle pong (heartbeat response)
          if (message.type === 'pong') return;

          // Route to registered handlers
          const handlers = this.messageHandlers.get(message.type);
          if (handlers) {
            handlers.forEach(handler => handler(message.payload));
          }

          // Also notify wildcard handlers
          const wildcardHandlers = this.messageHandlers.get('*');
          if (wildcardHandlers) {
            wildcardHandlers.forEach(handler => handler(message));
          }
        } catch (error) {
          console.error('Failed to parse WebSocket message:', error);
        }
      };

      this.ws.onclose = (event: CloseEvent) => {
        this.stopHeartbeat();
        this.config.onDisconnect(event.reason || 'Connection closed');

        if (!this.isManualClose) {
          this.scheduleReconnect();
        }
      };

      this.ws.onerror = (error: Event) => {
        this.config.onError(error);
      };
    } catch (error) {
      console.error('WebSocket connection failed:', error);
      this.scheduleReconnect();
    }
  }

  disconnect(): void {
    this.isManualClose = true;
    this.stopHeartbeat();
    if (this.reconnectTimer) {
      clearTimeout(this.reconnectTimer);
    }
    if (this.ws) {
      this.ws.close(1000, 'Client disconnect');
      this.ws = null;
    }
  }

  send(type: string, payload: unknown): void {
    const message = JSON.stringify({ type, payload });

    if (this.ws?.readyState === WebSocket.OPEN) {
      this.ws.send(message);
    } else {
      // Queue message for when connection is restored
      this.messageQueue.push(message);
    }
  }

  on(type: string, handler: MessageHandler): () => void {
    if (!this.messageHandlers.has(type)) {
      this.messageHandlers.set(type, new Set());
    }
    this.messageHandlers.get(type)!.add(handler);

    // Return unsubscribe function
    return () => {
      this.messageHandlers.get(type)?.delete(handler);
    };
  }

  private scheduleReconnect(): void {
    this.reconnectTimer = setTimeout(() => {
      console.log(`Reconnecting in ${this.currentReconnectInterval}ms...`);
      this.connect();
      this.currentReconnectInterval = Math.min(
        this.currentReconnectInterval * this.config.reconnectBackoffMultiplier,
        this.config.maxReconnectInterval
      );
    }, this.currentReconnectInterval);
  }

  private startHeartbeat(): void {
    this.heartbeatTimer = setInterval(() => {
      if (this.ws?.readyState === WebSocket.OPEN) {
        this.ws.send(JSON.stringify({ type: 'ping' }));
      }
    }, this.config.heartbeatInterval);
  }

  private stopHeartbeat(): void {
    if (this.heartbeatTimer) {
      clearInterval(this.heartbeatTimer);
      this.heartbeatTimer = null;
    }
  }

  private flushMessageQueue(): void {
    while (this.messageQueue.length > 0) {
      const message = this.messageQueue.shift()!;
      if (this.ws?.readyState === WebSocket.OPEN) {
        this.ws.send(message);
      } else {
        this.messageQueue.unshift(message);
        break;
      }
    }
  }
}
```

### Using WebSocket with React

```typescript
// hooks/useWebSocket.ts
import { useEffect, useRef, useCallback } from 'react';
import { ReconnectingWebSocket } from '../lib/websocket';
import { useAuthStore } from '../store/auth';

const wsInstance = new ReconnectingWebSocket({
  url: `wss://api.example.com/ws`,
  heartbeatInterval: 30000,
});

export function useWebSocket(type: string, handler: (payload: any) => void) {
  const handlerRef = useRef(handler);
  handlerRef.current = handler;

  useEffect(() => {
    return wsInstance.on(type, (payload) => {
      handlerRef.current(payload);
    });
  }, [type]);
}

export function useWebSocketConnect() {
  const token = useAuthStore(s => s.token);

  useEffect(() => {
    if (token) {
      wsInstance.connect();
      // Authenticate after connecting
      wsInstance.send('auth', { token });
    }

    return () => {
      wsInstance.disconnect();
    };
  }, [token]);
}

export function sendWebSocketMessage(type: string, payload: unknown) {
  wsInstance.send(type, payload);
}
```

### Integrating WebSocket with TanStack Query

The WebSocket updates your query cache in real-time:

```typescript
// hooks/useRealtimeTodos.ts
import { useQuery, useQueryClient } from '@tanstack/react-query';
import { useWebSocket } from './useWebSocket';

export function useRealtimeTodos() {
  const queryClient = useQueryClient();

  // Initial fetch and cache
  const query = useQuery({
    queryKey: ['todos'],
    queryFn: fetchTodos,
    staleTime: Infinity, // Don't refetch — WebSocket handles updates
  });

  // Real-time updates via WebSocket
  useWebSocket('todo:created', (newTodo: Todo) => {
    queryClient.setQueryData<Todo[]>(['todos'], (old = []) => [newTodo, ...old]);
  });

  useWebSocket('todo:updated', (updatedTodo: Todo) => {
    queryClient.setQueryData<Todo[]>(['todos'], (old = []) =>
      old.map(t => t.id === updatedTodo.id ? updatedTodo : t)
    );
  });

  useWebSocket('todo:deleted', ({ id }: { id: string }) => {
    queryClient.setQueryData<Todo[]>(['todos'], (old = []) =>
      old.filter(t => t.id !== id)
    );
  });

  return query;
}
```

### WebSocket Architecture Patterns

**Room-Based Pattern (Chat, Collaboration):**

```typescript
// Join/leave rooms for scoped real-time updates
function useChatRoom(roomId: string) {
  const queryClient = useQueryClient();

  useEffect(() => {
    sendWebSocketMessage('room:join', { roomId });

    return () => {
      sendWebSocketMessage('room:leave', { roomId });
    };
  }, [roomId]);

  useWebSocket('message:new', (message: Message) => {
    if (message.roomId === roomId) {
      queryClient.setQueryData<Message[]>(['messages', roomId], (old = []) => [
        ...old,
        message,
      ]);
    }
  });
}
```

**Fan-Out Pattern (Notifications, Live Updates):**

```typescript
// Server broadcasts to all connected clients
// Client filters by relevance

useWebSocket('notification', (notification: Notification) => {
  // Add to notification list
  queryClient.setQueryData<Notification[]>(['notifications'], (old = []) => [
    notification,
    ...old,
  ]);

  // Show a toast/alert
  showToast(notification.message);

  // If the notification affects data we have cached, invalidate it
  if (notification.type === 'product_updated') {
    queryClient.invalidateQueries({
      queryKey: ['product', notification.payload.productId],
    });
  }
});
```

---

## 8. OPTIMISTIC UI WITH ROLLBACK

Optimistic UI is the practice of immediately reflecting the user's action in the UI before the server confirms it. Combined with offline support, it creates an experience where the app feels instant regardless of network conditions.

### The Full Pattern

```typescript
// hooks/useOptimisticMutation.ts
import {
  useMutation,
  useQueryClient,
  type QueryKey,
} from '@tanstack/react-query';

interface OptimisticMutationOptions<TData, TVariables> {
  mutationFn: (variables: TVariables) => Promise<TData>;
  queryKey: QueryKey;
  // How to optimistically update the cache
  optimisticUpdate: (old: TData | undefined, variables: TVariables) => TData;
  // Optional: transform server response before caching
  transformResponse?: (response: TData, variables: TVariables) => TData;
}

export function useOptimisticMutation<TData, TVariables>({
  mutationFn,
  queryKey,
  optimisticUpdate,
  transformResponse,
}: OptimisticMutationOptions<TData, TVariables>) {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn,

    onMutate: async (variables: TVariables) => {
      // 1. Cancel any outgoing refetches (they would overwrite our optimistic update)
      await queryClient.cancelQueries({ queryKey });

      // 2. Snapshot the previous value (for rollback)
      const previousData = queryClient.getQueryData<TData>(queryKey);

      // 3. Optimistically update the cache
      queryClient.setQueryData<TData>(queryKey, (old) =>
        optimisticUpdate(old, variables)
      );

      // 4. Return context with the snapshotted value
      return { previousData };
    },

    onError: (err, variables, context) => {
      // Rollback to the previous value on error
      if (context?.previousData !== undefined) {
        queryClient.setQueryData(queryKey, context.previousData);
      }

      // Show error feedback to user
      console.error('Mutation failed, rolled back:', err);
    },

    onSuccess: (data, variables) => {
      // Optionally update cache with server's response
      if (transformResponse) {
        queryClient.setQueryData(queryKey, transformResponse(data, variables));
      }
    },

    onSettled: () => {
      // Always refetch after mutation to ensure we're in sync
      queryClient.invalidateQueries({ queryKey });
    },
  });
}
```

### Practical Example: Like Button

The like button is the canonical optimistic UI example. Nobody wants to wait 200ms for the server to confirm a like:

```typescript
// hooks/useLikePost.ts
export function useLikePost(postId: string) {
  return useOptimisticMutation<Post[], { postId: string; liked: boolean }>({
    mutationFn: ({ postId, liked }) =>
      liked ? api.likePost(postId) : api.unlikePost(postId),

    queryKey: ['posts'],

    optimisticUpdate: (posts = [], { postId, liked }) =>
      posts.map(post =>
        post.id === postId
          ? {
              ...post,
              liked,
              likeCount: post.likeCount + (liked ? 1 : -1),
            }
          : post
      ),
  });
}

// components/LikeButton.tsx
function LikeButton({ post }: { post: Post }) {
  const { mutate: toggleLike, isPending } = useLikePost(post.id);

  return (
    <Pressable
      onPress={() => toggleLike({ postId: post.id, liked: !post.liked })}
      style={{ opacity: isPending ? 0.7 : 1 }}
    >
      <HeartIcon filled={post.liked} />
      <Text>{post.likeCount}</Text>
    </Pressable>
  );
}
```

### Optimistic UI for Complex Operations

For operations more complex than toggles, track the pending state visually:

```typescript
// Drag-and-drop reorder with optimistic update
function useReorderItems() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: ({ itemId, newIndex }: { itemId: string; newIndex: number }) =>
      api.reorderItem(itemId, newIndex),

    onMutate: async ({ itemId, newIndex }) => {
      await queryClient.cancelQueries({ queryKey: ['items'] });
      const previous = queryClient.getQueryData<Item[]>(['items']);

      queryClient.setQueryData<Item[]>(['items'], (items = []) => {
        const result = [...items];
        const currentIndex = result.findIndex(i => i.id === itemId);
        if (currentIndex === -1) return result;

        const [item] = result.splice(currentIndex, 1);
        result.splice(newIndex, 0, { ...item, _reordering: true });
        return result;
      });

      return { previous };
    },

    onError: (err, vars, context) => {
      queryClient.setQueryData(['items'], context?.previous);
      // Haptic feedback for error
      Haptics.notificationAsync(Haptics.NotificationFeedbackType.Error);
    },

    onSuccess: () => {
      // Clear the _reordering flag
      queryClient.setQueryData<Item[]>(['items'], (items = []) =>
        items.map(i => ({ ...i, _reordering: false }))
      );
    },
  });
}
```

---

## 9. BACKGROUND SYNC PATTERNS

Sometimes you need to sync data even when the app isn't in the foreground. React Native gives you several options, each with different tradeoffs.

### App State Transitions

```typescript
// lib/background-sync.ts
import { AppState, type AppStateStatus } from 'react-native';

let appState: AppStateStatus = AppState.currentState;

export function setupBackgroundSync() {
  AppState.addEventListener('change', async (nextAppState) => {
    // App coming to foreground: sync any pending changes
    if (appState.match(/inactive|background/) && nextAppState === 'active') {
      console.log('App foregrounded — syncing');
      await syncPendingChanges();
      await refreshStaleData();
    }

    // App going to background: flush any pending writes
    if (nextAppState === 'background') {
      console.log('App backgrounded — flushing pending writes');
      await flushPendingMutations();
    }

    appState = nextAppState;
  });
}

async function syncPendingChanges() {
  const queue = mutationQueue.getQueue();
  if (queue.length > 0) {
    await mutationQueue.processQueue();
  }
}

async function refreshStaleData() {
  // Invalidate queries that are likely stale after being backgrounded
  queryClient.invalidateQueries({
    predicate: (query) => {
      const staleTime = query.options.staleTime ?? 0;
      const lastUpdated = query.state.dataUpdatedAt;
      return Date.now() - lastUpdated > staleTime;
    },
  });
}

async function flushPendingMutations() {
  // Ensure any in-flight mutations complete before the app is suspended
  const mutationCache = queryClient.getMutationCache();
  const pendingMutations = mutationCache.getAll().filter(
    m => m.state.status === 'pending'
  );

  if (pendingMutations.length > 0) {
    // Give mutations a few seconds to complete
    await Promise.race([
      Promise.all(pendingMutations.map(m => m.state.submittedAt)),
      new Promise(resolve => setTimeout(resolve, 3000)),
    ]);
  }
}
```

### Expo Background Fetch

For periodic sync even when the app is closed:

```typescript
// lib/background-fetch.ts
import * as BackgroundFetch from 'expo-background-fetch';
import * as TaskManager from 'expo-task-manager';

const BACKGROUND_SYNC_TASK = 'BACKGROUND_SYNC';

// Define the background task
TaskManager.defineTask(BACKGROUND_SYNC_TASK, async () => {
  try {
    console.log('Background sync task executing');

    // Process queued mutations
    const queue = mutationQueue.getQueue();
    if (queue.length > 0) {
      await mutationQueue.processQueue();
    }

    // Prefetch critical data
    await prefetchCriticalData();

    return BackgroundFetch.BackgroundFetchResult.NewData;
  } catch (error) {
    console.error('Background sync failed:', error);
    return BackgroundFetch.BackgroundFetchResult.Failed;
  }
});

// Register the background task
export async function registerBackgroundSync() {
  const status = await BackgroundFetch.getStatusAsync();

  if (status === BackgroundFetch.BackgroundFetchStatus.Available) {
    await BackgroundFetch.registerTaskAsync(BACKGROUND_SYNC_TASK, {
      minimumInterval: 15 * 60, // Minimum 15 minutes (OS may delay further)
      stopOnTerminate: false,
      startOnBoot: true,
    });
    console.log('Background sync registered');
  } else {
    console.log('Background fetch not available:', status);
  }
}
```

**Important caveats about background fetch:**

- iOS is extremely aggressive about limiting background execution. Your task might not run for hours.
- The minimum interval is a suggestion, not a guarantee. The OS optimizes for battery.
- You have approximately 30 seconds of execution time.
- Don't rely on background fetch for time-critical sync. It's for improving the experience when the user returns, not for real-time updates.

### Push-Triggered Sync

For more reliable background sync, use push notifications to wake the app:

```typescript
// When a silent push notification arrives
import * as Notifications from 'expo-notifications';

Notifications.setNotificationHandler({
  handleNotification: async (notification) => {
    const data = notification.request.content.data;

    if (data.type === 'sync_required') {
      // Perform sync in response to push
      await syncPendingChanges();
      await refreshData(data.queryKeys);
    }

    // Don't show the notification to the user (silent push)
    return {
      shouldShowAlert: false,
      shouldPlaySound: false,
      shouldSetBadge: false,
    };
  },
});
```

---

## 10. BUILDING AN OFFLINE-FIRST ARCHITECTURE END-TO-END

Let's put it all together with a concrete example: a task management app that works offline, syncs when online, and handles conflicts.

### Architecture Overview

```
┌──────────────────────────────────────────────────────────────┐
│  MOBILE APP                                                   │
│                                                               │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │  UI Layer (React Native components)                      │ │
│  │  - Shows local data immediately                          │ │
│  │  - Indicates sync status (pending, synced, conflict)     │ │
│  ├─────────────────────────────────────────────────────────┤ │
│  │  State Layer (TanStack Query + Zustand)                  │ │
│  │  - TanStack Query: server state cache                    │ │
│  │  - Zustand: UI state, sync status                        │ │
│  ├─────────────────────────────────────────────────────────┤ │
│  │  Sync Layer                                              │ │
│  │  - Mutation queue (MMKV-persisted)                       │ │
│  │  - Conflict resolver                                     │ │
│  │  - WebSocket for real-time updates                       │ │
│  │  - Background sync scheduler                             │ │
│  ├─────────────────────────────────────────────────────────┤ │
│  │  Persistence Layer (MMKV)                                │ │
│  │  - TanStack Query cache persistence                      │ │
│  │  - Mutation queue persistence                            │ │
│  │  - Sync metadata (last sync time, versions)              │ │
│  └─────────────────────────────────────────────────────────┘ │
│                                                               │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │  Network Layer                                           │ │
│  │  - NetInfo for connectivity detection                    │ │
│  │  - WebSocket (real-time) + REST (mutations)              │ │
│  │  - Automatic retry with exponential backoff              │ │
│  └─────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────┐
│  SERVER                                                       │
│                                                               │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │  API Layer (REST or GraphQL)                             │ │
│  │  - CRUD endpoints                                        │ │
│  │  - Sync endpoint (accepts batch mutations)               │ │
│  │  - Conflict detection (version checking)                 │ │
│  ├─────────────────────────────────────────────────────────┤ │
│  │  WebSocket Server                                        │ │
│  │  - Broadcasts changes to connected clients               │ │
│  │  - Room-based subscriptions                              │ │
│  ├─────────────────────────────────────────────────────────┤ │
│  │  Database                                                │ │
│  │  - Version column on every synced table                  │ │
│  │  - updatedAt timestamps                                  │ │
│  │  - Soft deletes (deleted_at instead of DELETE)           │ │
│  └─────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────┘
```

### The Sync Protocol

```typescript
// Sync endpoint: client sends local changes, receives remote changes

// Client request
interface SyncRequest {
  lastSyncTimestamp: number;
  mutations: Array<{
    type: 'create' | 'update' | 'delete';
    entity: string;
    id: string;
    data: Record<string, any>;
    clientTimestamp: number;
    version: number; // Client's version of the entity
  }>;
}

// Server response
interface SyncResponse {
  serverTimestamp: number;
  // Changes from server that client doesn't have
  changes: Array<{
    type: 'create' | 'update' | 'delete';
    entity: string;
    id: string;
    data: Record<string, any>;
    version: number;
  }>;
  // Conflicts that need client resolution
  conflicts: Array<{
    entity: string;
    id: string;
    clientVersion: Record<string, any>;
    serverVersion: Record<string, any>;
  }>;
}
```

### Server-Side Sync Endpoint

```typescript
// api/sync/route.ts
export async function POST(request: Request) {
  const body: SyncRequest = await request.json();
  const userId = await getUserFromToken(request);

  const changes: SyncResponse['changes'] = [];
  const conflicts: SyncResponse['conflicts'] = [];

  // Process client mutations
  for (const mutation of body.mutations) {
    const serverEntity = await db.findById(mutation.entity, mutation.id);

    if (mutation.type === 'create') {
      // No conflict possible for creates
      await db.create(mutation.entity, {
        ...mutation.data,
        userId,
        version: 1,
      });
    } else if (mutation.type === 'update') {
      if (!serverEntity) {
        // Entity was deleted on server
        conflicts.push({
          entity: mutation.entity,
          id: mutation.id,
          clientVersion: mutation.data,
          serverVersion: null,
        });
      } else if (serverEntity.version !== mutation.version) {
        // Version mismatch: conflict
        conflicts.push({
          entity: mutation.entity,
          id: mutation.id,
          clientVersion: mutation.data,
          serverVersion: serverEntity,
        });
      } else {
        // No conflict: apply update
        await db.update(mutation.entity, mutation.id, {
          ...mutation.data,
          version: serverEntity.version + 1,
        });
      }
    } else if (mutation.type === 'delete') {
      if (serverEntity && serverEntity.version === mutation.version) {
        await db.softDelete(mutation.entity, mutation.id);
      }
      // If version mismatch on delete, the server version wins
    }
  }

  // Get changes from server since last sync
  const serverChanges = await db.getChangesSince(
    userId,
    body.lastSyncTimestamp
  );

  return Response.json({
    serverTimestamp: Date.now(),
    changes: serverChanges,
    conflicts,
  } satisfies SyncResponse);
}
```

### Client-Side Sync Manager

```typescript
// lib/sync-manager.ts
import { queryClient } from './query-client';
import { storage } from './mmkv';
import { mutationQueue } from './mutation-queue';

const LAST_SYNC_KEY = 'last_sync_timestamp';

class SyncManager {
  private isSyncing = false;

  async sync(): Promise<void> {
    if (this.isSyncing) return;
    this.isSyncing = true;

    try {
      const lastSync = storage.getNumber(LAST_SYNC_KEY) ?? 0;
      const pendingMutations = mutationQueue.getQueue();

      const response = await fetch('/api/sync', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          lastSyncTimestamp: lastSync,
          mutations: pendingMutations.map(m => ({
            type: m.type,
            ...m.payload,
          })),
        }),
      });

      const syncResult: SyncResponse = await response.json();

      // 1. Apply server changes to local cache
      for (const change of syncResult.changes) {
        queryClient.setQueryData(
          [change.entity, change.id],
          change.data
        );
        // Also update list queries
        this.updateListCache(change);
      }

      // 2. Handle conflicts
      if (syncResult.conflicts.length > 0) {
        await this.resolveConflicts(syncResult.conflicts);
      }

      // 3. Clear synced mutations from queue
      for (const mutation of pendingMutations) {
        const hasConflict = syncResult.conflicts.some(
          c => c.id === (mutation.payload as any).id
        );
        if (!hasConflict) {
          mutationQueue.dequeue(mutation.id);
        }
      }

      // 4. Update sync timestamp
      storage.set(LAST_SYNC_KEY, syncResult.serverTimestamp);

    } catch (error) {
      console.error('Sync failed:', error);
    } finally {
      this.isSyncing = false;
    }
  }

  private async resolveConflicts(
    conflicts: SyncResponse['conflicts']
  ): Promise<void> {
    for (const conflict of conflicts) {
      // For now, use field-level merge
      // In a real app, you might prompt the user for important conflicts
      const resolved = fieldLevelMerge(
        {}, // We'd need the base version for proper 3-way merge
        conflict.clientVersion,
        conflict.serverVersion
      );

      // Push resolved version to server
      await fetch(`/api/${conflict.entity}/${conflict.id}`, {
        method: 'PUT',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(resolved),
      });

      // Update local cache
      queryClient.setQueryData(
        [conflict.entity, conflict.id],
        resolved
      );
    }
  }

  private updateListCache(change: SyncResponse['changes'][0]): void {
    queryClient.setQueryData<any[]>(
      [change.entity + 's'],
      (old = []) => {
        if (change.type === 'create') {
          return [change.data, ...old];
        }
        if (change.type === 'update') {
          return old.map(item =>
            item.id === change.id ? change.data : item
          );
        }
        if (change.type === 'delete') {
          return old.filter(item => item.id !== change.id);
        }
        return old;
      }
    );
  }
}

export const syncManager = new SyncManager();
```

---

## 11. TESTING OFFLINE SCENARIOS

You can't ship offline-first features without testing them. Here's how:

### In Development

```typescript
// Toggle offline mode in development
if (__DEV__) {
  // Add a dev menu option to simulate offline
  DevSettings.addMenuItem('Toggle Offline Mode', () => {
    const current = onlineManager.isOnline();
    onlineManager.setOnline(!current);
    Alert.alert(
      'Network Mode',
      current ? 'Now OFFLINE (simulated)' : 'Now ONLINE'
    );
  });
}
```

### Integration Tests

```typescript
// __tests__/offline-sync.test.ts
import { renderHook, act } from '@testing-library/react-hooks';
import { onlineManager, QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { useCreateTodo } from '../hooks/useCreateTodo';

describe('Offline mutations', () => {
  beforeEach(() => {
    onlineManager.setOnline(true);
  });

  it('queues mutations when offline and replays when online', async () => {
    // Go offline
    act(() => {
      onlineManager.setOnline(false);
    });

    const { result } = renderHook(() => useCreateTodo(), {
      wrapper: createWrapper(),
    });

    // Create a todo while offline
    await act(async () => {
      result.current.mutate({ title: 'Buy groceries' });
    });

    // Verify it's in the queue
    expect(mutationQueue.getQueue()).toHaveLength(1);

    // Come back online
    act(() => {
      onlineManager.setOnline(true);
    });

    // Queue should be processed
    await waitFor(() => {
      expect(mutationQueue.getQueue()).toHaveLength(0);
    });
  });

  it('shows optimistic data while offline', async () => {
    act(() => {
      onlineManager.setOnline(false);
    });

    const { result } = renderHook(
      () => ({
        create: useCreateTodo(),
        todos: useTodos(),
      }),
      { wrapper: createWrapper() }
    );

    await act(async () => {
      result.current.create.mutate({ title: 'Offline todo' });
    });

    // Todo should appear in the list with pending indicator
    expect(result.current.todos.data).toContainEqual(
      expect.objectContaining({
        title: 'Offline todo',
        _pending: true,
      })
    );
  });
});
```

---

## CHAPTER SUMMARY

Offline-first and real-time are two sides of the same coin: managing state across multiple sources of truth that may not always be connected.

The key patterns:

- **Connectivity detection** with NetInfo, integrated with TanStack Query's online manager
- **Legend State** for apps that need sync as a first-class primitive
- **Mutation queues** persisted to MMKV for offline write support
- **Conflict resolution** ranging from simple Last Write Wins to CRDTs, chosen based on your domain
- **WebSocket architecture** for real-time updates, integrated with TanStack Query's cache
- **Optimistic UI** that makes the app feel instant regardless of network conditions
- **Background sync** for keeping data fresh even when the app isn't in the foreground

The most important takeaway: **offline-first is not a feature, it's a mindset.** Start with the assumption that the network is unreliable, design your data flow around local-first operations, and treat the network as a sync mechanism rather than a dependency. The result is an app that's faster, more resilient, and more trustworthy.

---

*Next: [Chapter 13: Performance Optimization]*
*Previous: [Chapter 11: Caching Strategies — Mobile & Web]*