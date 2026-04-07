<!--
  CHAPTER: 41
  TITLE: Real-Time Collaboration & Multiplayer Features
  PART: III — State, Data & Communication
  PREREQS: Chapters 12, 13
  KEY_TOPICS: WebSockets, Socket.io, Ably, Pusher, Liveblocks, Yjs, CRDTs, presence, live cursors, collaborative editing, conflict resolution, operational transformation, real-time sync, Supabase Realtime, Firebase Realtime Database
  DIFFICULTY: Advanced
  UPDATED: 2026-04-07
-->

# Chapter 41: Real-Time Collaboration & Multiplayer Features

> **Part III — State, Data & Communication** | Prerequisites: Chapters 12, 13 | Difficulty: Advanced

<details>
<summary><strong>TL;DR</strong></summary>

- Real-time collaboration requires fundamentally different architecture than request-response: you need persistent connections, conflict resolution, and awareness of other users' actions as they happen
- WebSockets are the foundation; choose between self-hosted (Socket.io) and managed services (Ably, Pusher, Supabase Realtime) based on your scale, budget, and how much infrastructure you want to own
- Presence (who's online, typing indicators) is the easiest real-time feature to ship and delivers outsized user impact; start here before attempting collaborative editing
- For collaborative editing, CRDTs (specifically Yjs) have won over Operational Transformation for most new projects; they're simpler to reason about, work offline, and don't require a central server
- Liveblocks provides a managed platform that handles presence, cursors, and collaborative data structures so you don't have to build the hard parts yourself
- Supabase Realtime turns your Postgres database into a real-time data source with minimal code; it's the fastest path to real-time if you're already on Supabase
- Scaling WebSocket connections is a genuinely hard problem; managed services exist because most teams shouldn't solve it themselves

</details>

You've built an app where users can create and edit things. It works great for one person. Now your product manager says: "What if two people could edit the same document at the same time? Like Google Docs." And suddenly you're staring into the abyss of distributed systems.

Real-time collaboration is one of those features that sounds simple, seems almost trivial when you see it working in other products, and then humbles you completely when you try to build it. The gap between "I can send a WebSocket message" and "multiple users can simultaneously edit a shared document without losing each other's changes" is wider than the gap between "I can write a SQL query" and "I can build a database."

But here's the good news: the ecosystem has matured enormously. You don't need to understand the mathematical proofs behind CRDTs or implement the Operational Transformation algorithm from scratch. Libraries like Yjs, managed services like Liveblocks and Supabase Realtime, and battle-tested tools like Socket.io have made real-time collaboration accessible to teams that don't have Google's engineering resources.

This chapter will take you from zero to building real multiplayer features -- presence indicators, live cursors, collaborative editing, and real-time data sync -- with practical code you can ship.

### In This Chapter
- Real-Time Architecture: push vs poll, transport protocols, connection lifecycle
- WebSocket Providers: Socket.io, Ably, Pusher, Supabase Realtime, Firebase RTDB
- Presence: online indicators, typing indicators, active user lists
- Live Cursors & Awareness: showing other users' positions in real-time
- Collaborative Editing: OT vs CRDTs, Yjs, Liveblocks
- Conflict Resolution: strategies from simple to sophisticated
- Real-Time in React Native: background handling, battery, cache integration
- Supabase Realtime: Postgres changes broadcast via WebSocket
- Scaling WebSocket Connections: the hard problems and how to solve them
- Complete Examples: chat, collaborative task board, live activity feed

### Related Chapters
- [Ch 12: Offline-First & Real-Time Patterns] — offline-first mindset, connectivity detection, queue-based mutations
- [Ch 13: Performance Optimization] — rendering performance with frequent real-time updates
- [Ch 9: State Management at Scale] — integrating real-time data with your state architecture
- [Ch 10: Data Fetching & Server Communication] — TanStack Query integration with real-time

---

## 1. REAL-TIME ARCHITECTURE

Before you write a single line of WebSocket code, you need to understand the fundamental architectural choices. Getting these wrong means rebuilding everything later.

### Push vs Poll: The Core Decision

There are three ways to get updated data to the client:

```
POLLING (Pull):
  Client ──── GET /messages ────→ Server    (every 5 seconds)
  Client ←──── [messages] ───── Server
  Client ──── GET /messages ────→ Server    (5 seconds later, nothing new)
  Client ←──── [messages] ───── Server      (wasted request)
  Client ──── GET /messages ────→ Server    (5 seconds later)
  Client ←──── [messages] ───── Server

  Problem: Wasteful. 90%+ of requests return nothing new.
           Delay = up to poll interval (5s).

LONG POLLING (Pull, but smarter):
  Client ──── GET /messages ────→ Server    (server holds connection open)
  ....... (server waits until new data exists) .......
  Client ←──── [new message] ── Server      (responds immediately when data available)
  Client ──── GET /messages ────→ Server    (immediately reconnects)
  ....... (waits again) .......

  Better: No wasted requests. Low latency.
  Problem: Still HTTP overhead per message. Connection management is awkward.

PUSH (WebSocket / SSE):
  Client ←───── persistent connection ─────→ Server
  Server pushes [new message] whenever it has one
  Server pushes [user joined] whenever that happens
  Server pushes [typing indicator] in real-time

  Best: Minimal overhead. True real-time. Bidirectional (WebSocket).
```

### When Real-Time Is Worth the Complexity

Real-time is not free. It adds WebSocket server infrastructure, connection state management, conflict resolution logic, and operational complexity. Use this decision framework:

```
┌─────────────────────────────────────────────────────────────────┐
│  USE REAL-TIME (WebSockets / SSE) WHEN:                        │
│                                                                  │
│  ✓ Users need to see changes within < 1 second                  │
│  ✓ Multiple users interact with the same data simultaneously    │
│  ✓ The feature is inherently "live" (chat, cursors, feeds)      │
│  ✓ You need server-initiated communication (notifications)      │
│  ✓ High-frequency updates (stock prices, game state, IoT)       │
│                                                                  │
├─────────────────────────────────────────────────────────────────┤
│  USE POLLING WHEN:                                               │
│                                                                  │
│  ✓ Updates are infrequent (< once per minute)                   │
│  ✓ Slight delay is acceptable (dashboard that refreshes)        │
│  ✓ Infrastructure simplicity matters more than latency           │
│  ✓ You're behind restrictive firewalls/proxies                  │
│  ✓ Your backend doesn't support persistent connections          │
│                                                                  │
├─────────────────────────────────────────────────────────────────┤
│  USE SSE (Server-Sent Events) WHEN:                             │
│                                                                  │
│  ✓ Server pushes TO client only (no client→server needed)       │
│  ✓ You want auto-reconnection built into the protocol           │
│  ✓ You need HTTP/2 multiplexing benefits                        │
│  ✓ Simple notification feeds, live logs, progress updates       │
│                                                                  │
│  Note: SSE is unidirectional. If you need client→server,        │
│  combine SSE (for push) + normal HTTP (for mutations).          │
└─────────────────────────────────────────────────────────────────┘
```

### WebSocket vs SSE: Protocol Comparison

```
┌────────────────────┬──────────────────────┬──────────────────────┐
│                    │  WebSocket           │  SSE                 │
├────────────────────┼──────────────────────┼──────────────────────┤
│ Direction          │ Bidirectional        │ Server → Client only │
│ Protocol           │ ws:// / wss://       │ HTTP (text/event-    │
│                    │                      │ stream)              │
│ Binary data        │ Yes                  │ No (text only)       │
│ Auto-reconnect     │ Manual               │ Built-in             │
│ HTTP/2 compat      │ Separate connection  │ Multiplexed          │
│ Proxy-friendly     │ Can be blocked       │ Always works         │
│ Max connections    │ ~6 per domain        │ ~6 per domain (HTTP  │
│  (browser)         │ (separate from HTTP) │ 1.1), unlimited H2   │
│ Overhead per msg   │ 2-6 bytes            │ ~50+ bytes (HTTP     │
│                    │                      │ framing)             │
│ Best for           │ Chat, collab edit,   │ Notifications, live  │
│                    │ gaming, presence     │ feeds, log streaming │
└────────────────────┴──────────────────────┴──────────────────────┘
```

### The WebSocket Connection Lifecycle

Every real-time feature starts with a WebSocket connection. Understanding its lifecycle is critical:

```typescript
// lib/websocket-manager.ts
// A production-grade WebSocket manager with reconnection, heartbeat, and message queuing

type ConnectionState = 'connecting' | 'connected' | 'reconnecting' | 'disconnected';
type MessageHandler = (data: unknown) => void;

interface WebSocketManagerOptions {
  url: string;
  protocols?: string[];
  reconnect?: {
    enabled: boolean;
    maxAttempts: number;
    baseDelay: number;      // milliseconds
    maxDelay: number;        // milliseconds
    backoffMultiplier: number;
  };
  heartbeat?: {
    enabled: boolean;
    interval: number;        // milliseconds
    timeout: number;         // milliseconds
    message: string | (() => string);
  };
  auth?: {
    token: string;
  };
}

const DEFAULT_OPTIONS: Required<WebSocketManagerOptions> = {
  url: '',
  protocols: [],
  reconnect: {
    enabled: true,
    maxAttempts: 10,
    baseDelay: 1000,
    maxDelay: 30000,
    backoffMultiplier: 2,
  },
  heartbeat: {
    enabled: true,
    interval: 30000,
    timeout: 5000,
    message: 'ping',
  },
  auth: {
    token: '',
  },
};

export class WebSocketManager {
  private ws: WebSocket | null = null;
  private options: Required<WebSocketManagerOptions>;
  private state: ConnectionState = 'disconnected';
  private reconnectAttempts = 0;
  private reconnectTimer: ReturnType<typeof setTimeout> | null = null;
  private heartbeatTimer: ReturnType<typeof setInterval> | null = null;
  private heartbeatTimeoutTimer: ReturnType<typeof setTimeout> | null = null;
  private messageQueue: string[] = [];
  private handlers = new Map<string, Set<MessageHandler>>();
  private stateListeners = new Set<(state: ConnectionState) => void>();

  constructor(options: WebSocketManagerOptions) {
    this.options = { ...DEFAULT_OPTIONS, ...options } as Required<WebSocketManagerOptions>;
  }

  // --- Connection ---

  connect(): void {
    if (this.state === 'connected' || this.state === 'connecting') return;

    this.setState('connecting');

    const url = new URL(this.options.url);
    if (this.options.auth.token) {
      url.searchParams.set('token', this.options.auth.token);
    }

    this.ws = new WebSocket(url.toString(), this.options.protocols);

    this.ws.onopen = () => {
      this.setState('connected');
      this.reconnectAttempts = 0;
      this.flushMessageQueue();
      this.startHeartbeat();
    };

    this.ws.onmessage = (event: MessageEvent) => {
      // Reset heartbeat timeout on any message
      this.resetHeartbeatTimeout();

      try {
        const parsed = JSON.parse(event.data);
        const { type, payload } = parsed;

        if (type === 'pong') return; // Heartbeat response

        const handlers = this.handlers.get(type);
        if (handlers) {
          handlers.forEach((handler) => handler(payload));
        }

        // Also notify wildcard handlers
        const wildcardHandlers = this.handlers.get('*');
        if (wildcardHandlers) {
          wildcardHandlers.forEach((handler) => handler(parsed));
        }
      } catch {
        // Non-JSON message, pass raw data to wildcard handlers
        const wildcardHandlers = this.handlers.get('*');
        if (wildcardHandlers) {
          wildcardHandlers.forEach((handler) => handler(event.data));
        }
      }
    };

    this.ws.onclose = (event: CloseEvent) => {
      this.stopHeartbeat();

      // 1000 = normal closure, 1001 = going away (page navigation)
      if (event.code === 1000 || event.code === 1001) {
        this.setState('disconnected');
        return;
      }

      // Abnormal closure — attempt reconnection
      if (this.options.reconnect.enabled) {
        this.attemptReconnect();
      } else {
        this.setState('disconnected');
      }
    };

    this.ws.onerror = () => {
      // The error event is always followed by a close event.
      // We handle reconnection in onclose.
    };
  }

  disconnect(): void {
    this.options.reconnect.enabled = false; // Prevent reconnection
    this.stopHeartbeat();
    if (this.reconnectTimer) clearTimeout(this.reconnectTimer);
    this.ws?.close(1000, 'Client disconnect');
    this.setState('disconnected');
  }

  // --- Reconnection with Exponential Backoff ---

  private attemptReconnect(): void {
    if (this.reconnectAttempts >= this.options.reconnect.maxAttempts) {
      this.setState('disconnected');
      this.emit('max_reconnect_attempts', {});
      return;
    }

    this.setState('reconnecting');

    // Exponential backoff with jitter
    const delay = Math.min(
      this.options.reconnect.baseDelay *
        Math.pow(this.options.reconnect.backoffMultiplier, this.reconnectAttempts) +
        Math.random() * 1000, // Jitter prevents thundering herd
      this.options.reconnect.maxDelay
    );

    this.reconnectTimer = setTimeout(() => {
      this.reconnectAttempts++;
      this.connect();
    }, delay);
  }

  // --- Heartbeat ---

  private startHeartbeat(): void {
    if (!this.options.heartbeat.enabled) return;

    this.heartbeatTimer = setInterval(() => {
      const message =
        typeof this.options.heartbeat.message === 'function'
          ? this.options.heartbeat.message()
          : this.options.heartbeat.message;

      this.send('ping', { message });

      // Set timeout — if we don't hear back, the connection is dead
      this.heartbeatTimeoutTimer = setTimeout(() => {
        // Force close to trigger reconnection
        this.ws?.close(4000, 'Heartbeat timeout');
      }, this.options.heartbeat.timeout);
    }, this.options.heartbeat.interval);
  }

  private stopHeartbeat(): void {
    if (this.heartbeatTimer) clearInterval(this.heartbeatTimer);
    if (this.heartbeatTimeoutTimer) clearTimeout(this.heartbeatTimeoutTimer);
  }

  private resetHeartbeatTimeout(): void {
    if (this.heartbeatTimeoutTimer) clearTimeout(this.heartbeatTimeoutTimer);
  }

  // --- Messaging ---

  send(type: string, payload: unknown): void {
    const message = JSON.stringify({ type, payload });

    if (this.state === 'connected' && this.ws?.readyState === WebSocket.OPEN) {
      this.ws.send(message);
    } else {
      // Queue messages while disconnected
      this.messageQueue.push(message);
    }
  }

  private flushMessageQueue(): void {
    while (this.messageQueue.length > 0) {
      const message = this.messageQueue.shift()!;
      this.ws?.send(message);
    }
  }

  // --- Event Handling ---

  on(type: string, handler: MessageHandler): () => void {
    if (!this.handlers.has(type)) {
      this.handlers.set(type, new Set());
    }
    this.handlers.get(type)!.add(handler);

    // Return unsubscribe function
    return () => {
      this.handlers.get(type)?.delete(handler);
    };
  }

  private emit(type: string, payload: unknown): void {
    const handlers = this.handlers.get(type);
    if (handlers) {
      handlers.forEach((handler) => handler(payload));
    }
  }

  // --- State ---

  private setState(newState: ConnectionState): void {
    this.state = newState;
    this.stateListeners.forEach((listener) => listener(newState));
  }

  getState(): ConnectionState {
    return this.state;
  }

  onStateChange(listener: (state: ConnectionState) => void): () => void {
    this.stateListeners.add(listener);
    return () => this.stateListeners.delete(listener);
  }
}
```

### Using the WebSocket Manager in React

```typescript
// hooks/useWebSocket.ts
import { useEffect, useRef, useState, useCallback } from 'react';
import { WebSocketManager } from '../lib/websocket-manager';

type ConnectionState = 'connecting' | 'connected' | 'reconnecting' | 'disconnected';

interface UseWebSocketOptions {
  url: string;
  token?: string;
  enabled?: boolean;
}

interface UseWebSocketReturn {
  state: ConnectionState;
  send: (type: string, payload: unknown) => void;
  on: (type: string, handler: (data: unknown) => void) => () => void;
}

export function useWebSocket({
  url,
  token,
  enabled = true,
}: UseWebSocketOptions): UseWebSocketReturn {
  const managerRef = useRef<WebSocketManager | null>(null);
  const [state, setState] = useState<ConnectionState>('disconnected');

  useEffect(() => {
    if (!enabled) return;

    const manager = new WebSocketManager({
      url,
      auth: token ? { token } : undefined,
      reconnect: {
        enabled: true,
        maxAttempts: 10,
        baseDelay: 1000,
        maxDelay: 30000,
        backoffMultiplier: 2,
      },
      heartbeat: {
        enabled: true,
        interval: 30000,
        timeout: 5000,
        message: 'ping',
      },
    });

    managerRef.current = manager;

    const unsubState = manager.onStateChange(setState);
    manager.connect();

    return () => {
      unsubState();
      manager.disconnect();
      managerRef.current = null;
    };
  }, [url, token, enabled]);

  const send = useCallback((type: string, payload: unknown) => {
    managerRef.current?.send(type, payload);
  }, []);

  const on = useCallback((type: string, handler: (data: unknown) => void) => {
    return managerRef.current?.on(type, handler) ?? (() => {});
  }, []);

  return { state, send, on };
}
```

### Connection State UI

Always show users the connection state. Real-time features that silently break are worse than no real-time at all:

```typescript
// components/ConnectionStatus.tsx
import { useWebSocket } from '../hooks/useWebSocket';

function ConnectionStatus() {
  const { state } = useWebSocket({
    url: process.env.NEXT_PUBLIC_WS_URL!,
  });

  if (state === 'connected') return null; // Don't show anything when connected

  const statusConfig = {
    connecting: {
      label: 'Connecting...',
      color: 'bg-yellow-500',
      animate: true,
    },
    reconnecting: {
      label: 'Reconnecting...',
      color: 'bg-orange-500',
      animate: true,
    },
    disconnected: {
      label: 'Disconnected. Changes may not be saved.',
      color: 'bg-red-500',
      animate: false,
    },
  } as const;

  const config = statusConfig[state as keyof typeof statusConfig];
  if (!config) return null;

  return (
    <div className={`fixed top-0 inset-x-0 z-50 ${config.color} text-white text-center py-2 text-sm`}>
      {config.animate && (
        <span className="inline-block w-2 h-2 rounded-full bg-white animate-pulse mr-2" />
      )}
      {config.label}
    </div>
  );
}
```

---

## 2. WEBSOCKET PROVIDERS

You have two broad choices: self-host your WebSocket infrastructure, or use a managed service. This decision has enormous implications for cost, reliability, and development speed.

### Socket.io: The Self-Hosted Standard

Socket.io is by far the most popular WebSocket library. It wraps the raw WebSocket API with rooms, namespaces, automatic reconnection, and a fallback to long-polling when WebSockets aren't available.

**Server setup:**

```typescript
// server/socket-server.ts
import { createServer } from 'http';
import { Server, Socket } from 'socket.io';
import { verifyToken } from './auth';

const httpServer = createServer();
const io = new Server(httpServer, {
  cors: {
    origin: process.env.CLIENT_URL,
    methods: ['GET', 'POST'],
    credentials: true,
  },
  // Transport configuration
  transports: ['websocket', 'polling'], // Prefer WebSocket, fall back to polling
  pingInterval: 25000,  // How often to send ping
  pingTimeout: 20000,   // How long to wait for pong before disconnecting
  maxHttpBufferSize: 1e6, // 1MB max message size
});

// --- Authentication Middleware ---
io.use(async (socket: Socket, next) => {
  const token = socket.handshake.auth.token;
  if (!token) {
    return next(new Error('Authentication required'));
  }

  try {
    const user = await verifyToken(token);
    socket.data.user = user;
    next();
  } catch {
    next(new Error('Invalid token'));
  }
});

// --- Namespaces ---
// Namespaces let you split logic into separate "apps" on the same connection
const chatNamespace = io.of('/chat');
const notificationsNamespace = io.of('/notifications');

// --- Connection Handler ---
io.on('connection', (socket: Socket) => {
  const userId = socket.data.user.id;
  const userName = socket.data.user.name;

  console.log(`User connected: ${userName} (${socket.id})`);

  // --- Rooms ---
  // Rooms are arbitrary groups of sockets. Perfect for:
  // - Project teams
  // - Document collaborators
  // - Chat channels

  socket.on('join-room', (roomId: string) => {
    socket.join(roomId);

    // Notify others in the room
    socket.to(roomId).emit('user-joined', {
      userId,
      userName,
      timestamp: Date.now(),
    });

    // Send current room members to the joining user
    const room = io.sockets.adapter.rooms.get(roomId);
    const memberCount = room?.size ?? 0;
    socket.emit('room-joined', { roomId, memberCount });
  });

  socket.on('leave-room', (roomId: string) => {
    socket.leave(roomId);
    socket.to(roomId).emit('user-left', {
      userId,
      userName,
      timestamp: Date.now(),
    });
  });

  // --- Messaging ---
  socket.on('message', (data: { roomId: string; content: string }) => {
    const message = {
      id: crypto.randomUUID(),
      userId,
      userName,
      content: data.content,
      roomId: data.roomId,
      timestamp: Date.now(),
    };

    // Send to everyone in the room INCLUDING the sender
    io.to(data.roomId).emit('message', message);

    // Persist to database (fire-and-forget or await if you need confirmation)
    saveMessage(message).catch(console.error);
  });

  // --- Typing Indicators ---
  socket.on('typing-start', (roomId: string) => {
    socket.to(roomId).emit('user-typing', { userId, userName });
  });

  socket.on('typing-stop', (roomId: string) => {
    socket.to(roomId).emit('user-stopped-typing', { userId });
  });

  // --- Disconnect ---
  socket.on('disconnect', (reason: string) => {
    console.log(`User disconnected: ${userName} (${reason})`);
    // Socket.io automatically removes the socket from all rooms on disconnect
  });
});

// --- Broadcasting Patterns ---

// To everyone connected:
// io.emit('announcement', { message: 'Server maintenance at midnight' });

// To everyone in a room:
// io.to('room-123').emit('update', data);

// To everyone in a room EXCEPT the sender:
// socket.to('room-123').emit('update', data);

// To a specific socket:
// io.to(socketId).emit('private-message', data);

// To everyone in multiple rooms:
// io.to('room-1').to('room-2').emit('update', data);

httpServer.listen(3001, () => {
  console.log('Socket.io server running on port 3001');
});
```

**Client setup:**

```typescript
// lib/socket-client.ts
import { io, Socket } from 'socket.io-client';

let socket: Socket | null = null;

export function getSocket(token: string): Socket {
  if (socket?.connected) return socket;

  socket = io(process.env.NEXT_PUBLIC_WS_URL!, {
    auth: { token },
    transports: ['websocket', 'polling'],
    reconnection: true,
    reconnectionAttempts: 10,
    reconnectionDelay: 1000,
    reconnectionDelayMax: 30000,
    timeout: 20000,
  });

  socket.on('connect', () => {
    console.log('Connected to Socket.io server');
  });

  socket.on('connect_error', (error) => {
    console.error('Connection error:', error.message);
    if (error.message === 'Invalid token') {
      // Token expired — refresh and reconnect
      refreshTokenAndReconnect();
    }
  });

  socket.on('disconnect', (reason) => {
    console.log('Disconnected:', reason);
    // 'io server disconnect' = server kicked us
    // 'io client disconnect' = we called socket.disconnect()
    // 'transport close' = connection lost
    // 'ping timeout' = server stopped responding
  });

  return socket;
}

// React hook
export function useSocketIO(token: string) {
  const [socket, setSocket] = useState<Socket | null>(null);
  const [connected, setConnected] = useState(false);

  useEffect(() => {
    const s = getSocket(token);
    setSocket(s);

    const onConnect = () => setConnected(true);
    const onDisconnect = () => setConnected(false);

    s.on('connect', onConnect);
    s.on('disconnect', onDisconnect);
    setConnected(s.connected);

    return () => {
      s.off('connect', onConnect);
      s.off('disconnect', onDisconnect);
    };
  }, [token]);

  return { socket, connected };
}
```

### Ably: Managed Global Scale

Ably is a managed real-time platform with guaranteed message delivery, global edge infrastructure, and enterprise features. You pay per message and per connection, but you get reliability guarantees that are extremely expensive to build yourself.

```typescript
// lib/ably-client.ts
import Ably from 'ably';
import { useEffect, useState } from 'react';

// Initialize Ably with token authentication (never expose API keys to client)
const ably = new Ably.Realtime({
  authUrl: '/api/ably-token', // Your server endpoint that creates Ably tokens
  authMethod: 'POST',
  // Ably handles reconnection automatically with built-in strategies
});

// --- Channel Patterns ---

// Basic pub/sub channel
const updatesChannel = ably.channels.get('project:updates');

// Subscribe to messages
updatesChannel.subscribe('task-created', (message) => {
  console.log('New task:', message.data);
});

// Publish a message
updatesChannel.publish('task-created', {
  id: '123',
  title: 'Implement real-time sync',
  createdBy: 'user-456',
});

// --- Presence ---
const projectChannel = ably.channels.get('project:abc');

// Enter presence
await projectChannel.presence.enter({
  userId: 'user-456',
  name: 'Alice',
  avatar: 'https://...',
});

// Subscribe to presence changes
projectChannel.presence.subscribe('enter', (member) => {
  console.log(`${member.data.name} joined`);
});

projectChannel.presence.subscribe('leave', (member) => {
  console.log(`${member.data.name} left`);
});

// Get current members
const members = await projectChannel.presence.get();

// --- React Hook ---
export function useAblyChannel(channelName: string) {
  const [messages, setMessages] = useState<Ably.Message[]>([]);
  const [channel, setChannel] = useState<Ably.RealtimeChannel | null>(null);

  useEffect(() => {
    const ch = ably.channels.get(channelName);
    setChannel(ch);

    ch.subscribe((message) => {
      setMessages((prev) => [...prev, message]);
    });

    return () => {
      ch.unsubscribe();
      ch.detach();
    };
  }, [channelName]);

  const publish = (event: string, data: unknown) => {
    channel?.publish(event, data);
  };

  return { messages, publish, channel };
}
```

### Pusher: Simple Pub/Sub

Pusher is the simplest managed option. It does pub/sub channels extremely well and not much else. If you need basic real-time features (notifications, live updates, typing indicators) without collaborative editing, Pusher is the fastest to integrate.

```typescript
// lib/pusher-client.ts
import Pusher from 'pusher-js';

const pusher = new Pusher(process.env.NEXT_PUBLIC_PUSHER_KEY!, {
  cluster: process.env.NEXT_PUBLIC_PUSHER_CLUSTER!,
  // Use auth endpoint for private/presence channels
  channelAuthorization: {
    endpoint: '/api/pusher/auth',
    transport: 'ajax',
  },
});

// --- Channel Types ---

// Public channel (no auth needed, anyone can subscribe)
const publicChannel = pusher.subscribe('updates');
publicChannel.bind('new-notification', (data: { message: string }) => {
  showNotification(data.message);
});

// Private channel (requires auth, only authorized users can subscribe)
const privateChannel = pusher.subscribe('private-user-123');
privateChannel.bind('direct-message', (data: unknown) => {
  handleDirectMessage(data);
});

// Presence channel (like private, but with awareness of who's subscribed)
const presenceChannel = pusher.subscribe('presence-project-abc');

presenceChannel.bind('pusher:subscription_succeeded', (members: any) => {
  // members.count, members.each(), members.me
  console.log(`${members.count} users online`);
  members.each((member: any) => {
    console.log(`- ${member.info.name}`);
  });
});

presenceChannel.bind('pusher:member_added', (member: any) => {
  console.log(`${member.info.name} joined`);
});

presenceChannel.bind('pusher:member_removed', (member: any) => {
  console.log(`${member.info.name} left`);
});

// --- Server-Side Trigger (Node.js) ---
// Messages are triggered FROM your server, not from the client
// (Pusher is primarily a server-to-client push mechanism)

// server/pusher.ts
import PusherServer from 'pusher';

const pusherServer = new PusherServer({
  appId: process.env.PUSHER_APP_ID!,
  key: process.env.PUSHER_KEY!,
  secret: process.env.PUSHER_SECRET!,
  cluster: process.env.PUSHER_CLUSTER!,
  useTLS: true,
});

// Trigger an event
await pusherServer.trigger('updates', 'new-notification', {
  message: 'New comment on your post',
});

// Trigger to multiple channels
await pusherServer.trigger(
  ['private-user-123', 'private-user-456'],
  'project-update',
  { projectId: 'abc', action: 'task-completed' }
);
```

### Decision Matrix: Choosing a WebSocket Provider

```
┌──────────────────┬────────────┬──────────┬──────────┬───────────────┬────────────┐
│                  │ Socket.io  │ Ably     │ Pusher   │ Supabase RT   │ Firebase   │
│                  │            │          │          │               │ RTDB       │
├──────────────────┼────────────┼──────────┼──────────┼───────────────┼────────────┤
│ Hosting          │ Self-host  │ Managed  │ Managed  │ Managed /     │ Managed    │
│                  │            │          │          │ Self-host     │            │
├──────────────────┼────────────┼──────────┼──────────┼───────────────┼────────────┤
│ Complexity       │ Medium     │ Low      │ Low      │ Low           │ Low        │
├──────────────────┼────────────┼──────────┼──────────┼───────────────┼────────────┤
│ Scalability      │ Manual     │ Global,  │ Good,    │ Good,         │ Excellent, │
│                  │ (Redis)    │ auto     │ limits   │ Postgres-     │ auto       │
│                  │            │          │          │ bound         │            │
├──────────────────┼────────────┼──────────┼──────────┼───────────────┼────────────┤
│ Delivery         │ At-most-   │ At-least │ At-most- │ At-most-once  │ At-least-  │
│ guarantee        │ once       │ once     │ once     │               │ once       │
├──────────────────┼────────────┼──────────┼──────────┼───────────────┼────────────┤
│ Presence         │ Manual     │ Built-in │ Built-in │ Built-in      │ Built-in   │
├──────────────────┼────────────┼──────────┼──────────┼───────────────┼────────────┤
│ DB integration   │ None       │ None     │ None     │ Postgres      │ Own DB     │
│                  │            │          │          │ native        │            │
├──────────────────┼────────────┼──────────┼──────────┼───────────────┼────────────┤
│ Cost at scale    │ Server     │ $$-$$$   │ $$       │ $-$$          │ $$         │
│                  │ costs only │          │          │               │            │
├──────────────────┼────────────┼──────────┼──────────┼───────────────┼────────────┤
│ Best for         │ Full       │ Enter-   │ Simple   │ Apps already  │ Mobile     │
│                  │ control,   │ prise,   │ pub/sub, │ on Supabase,  │ apps,      │
│                  │ custom     │ global,  │ notif-   │ DB-driven     │ offline-   │
│                  │ protocols  │ mission- │ ications │ real-time     │ first      │
│                  │            │ critical │          │               │            │
├──────────────────┼────────────┼──────────┼──────────┼───────────────┼────────────┤
│ React Native     │ Works      │ Works    │ Works    │ Works         │ SDK        │
│                  │ (WebSocket │ (SDK)    │ (SDK)    │ (JS SDK)      │ available  │
│                  │ polyfill)  │          │          │               │            │
└──────────────────┴────────────┴──────────┴──────────┴───────────────┴────────────┘
```

**The 100x recommendation:** Start with Supabase Realtime if you're already on Supabase (you get it for free). Use Socket.io if you need full control and are comfortable operating WebSocket infrastructure. Use Ably if you need guaranteed delivery at global scale and have the budget. Use Pusher for simple notification-style real-time where you just need server-to-client push.

---

## 3. PRESENCE

Presence is the most impactful real-time feature relative to its implementation complexity. Showing who's online, who's typing, and who's viewing the same page transforms a single-player app into something that feels alive.

### What Presence Means

```
┌─────────────────────────────────────────────────────────────┐
│  PRESENCE encompasses:                                       │
│                                                              │
│  1. ONLINE STATUS                                            │
│     "Sarah is online" (green dot)                            │
│     "Mike was last seen 5 minutes ago"                       │
│                                                              │
│  2. ACTIVITY AWARENESS                                       │
│     "Sarah is viewing this document"                         │
│     "3 people are on this page right now"                    │
│                                                              │
│  3. ACTION INDICATORS                                        │
│     "Sarah is typing..."                                     │
│     "Mike is editing the title"                              │
│     "Alex is dragging a task"                                │
│                                                              │
│  4. CURSOR/SELECTION POSITION                                │
│     "Sarah's cursor is on line 42" (covered in Section 4)   │
│     "Mike has selected rows 3-7"                             │
└─────────────────────────────────────────────────────────────┘
```

### Presence with Liveblocks

Liveblocks makes presence trivially easy. It's purpose-built for this:

```typescript
// liveblocks.config.ts
import { createClient } from '@liveblocks/client';
import { createRoomContext } from '@liveblocks/react';

const client = createClient({
  authEndpoint: '/api/liveblocks-auth',
});

// Define your presence data shape
type Presence = {
  cursor: { x: number; y: number } | null;
  selectedId: string | null;
  isTyping: boolean;
  name: string;
  color: string;
  avatar: string;
};

// Define the storage types (for collaborative data, covered in Section 5)
type Storage = {
  tasks: LiveList<LiveObject<Task>>;
};

// Define user metadata
type UserMeta = {
  id: string;
  info: {
    name: string;
    email: string;
    avatar: string;
    color: string;
  };
};

export const {
  RoomProvider,
  useMyPresence,
  useOthers,
  useOthersMapped,
  useSelf,
  useUpdateMyPresence,
  useBroadcastEvent,
  useEventListener,
  useStorage,
  useMutation,
} = createRoomContext<Presence, Storage, UserMeta>(client);
```

```typescript
// components/CollaborativeRoom.tsx
import { RoomProvider } from '../liveblocks.config';

export function CollaborativeRoom({ roomId, children }: {
  roomId: string;
  children: React.ReactNode;
}) {
  return (
    <RoomProvider
      id={roomId}
      initialPresence={{
        cursor: null,
        selectedId: null,
        isTyping: false,
        name: '',
        color: '',
        avatar: '',
      }}
    >
      {children}
    </RoomProvider>
  );
}
```

```typescript
// components/ActiveUsers.tsx
import { useOthers, useSelf } from '../liveblocks.config';

const COLORS = [
  '#E57373', '#81C784', '#64B5F6', '#FFD54F',
  '#BA68C8', '#4DB6AC', '#FF8A65', '#A1887F',
];

export function ActiveUsers() {
  const others = useOthers();
  const self = useSelf();

  const MAX_SHOWN = 5;
  const overflow = others.length - MAX_SHOWN;

  return (
    <div className="flex items-center -space-x-2">
      {/* Current user */}
      {self && (
        <div
          className="w-8 h-8 rounded-full border-2 border-white flex items-center
                     justify-center text-white text-xs font-medium ring-2 ring-blue-500"
          style={{ backgroundColor: self.info.color }}
          title="You"
        >
          {self.info.name.charAt(0).toUpperCase()}
        </div>
      )}

      {/* Other users */}
      {others.slice(0, MAX_SHOWN).map((other) => (
        <div
          key={other.connectionId}
          className="w-8 h-8 rounded-full border-2 border-white flex items-center
                     justify-center text-white text-xs font-medium"
          style={{ backgroundColor: other.info.color }}
          title={other.info.name}
        >
          {other.info.name.charAt(0).toUpperCase()}
        </div>
      ))}

      {/* Overflow indicator */}
      {overflow > 0 && (
        <div className="w-8 h-8 rounded-full border-2 border-white bg-gray-500
                       flex items-center justify-center text-white text-xs font-medium">
          +{overflow}
        </div>
      )}
    </div>
  );
}
```

### Typing Indicators

Typing indicators are a specific presence pattern with one critical detail: **debouncing**. You don't want to flood the WebSocket with "is typing" events on every keystroke.

```typescript
// hooks/useTypingIndicator.ts
import { useCallback, useRef, useEffect } from 'react';
import { useUpdateMyPresence, useOthers } from '../liveblocks.config';

interface UseTypingIndicatorReturn {
  onKeyDown: () => void;
  typingUsers: { name: string; color: string }[];
}

export function useTypingIndicator(): UseTypingIndicatorReturn {
  const updatePresence = useUpdateMyPresence();
  const others = useOthers();
  const timeoutRef = useRef<ReturnType<typeof setTimeout> | null>(null);

  const onKeyDown = useCallback(() => {
    // Set typing to true
    updatePresence({ isTyping: true });

    // Clear previous timeout
    if (timeoutRef.current) {
      clearTimeout(timeoutRef.current);
    }

    // Set typing to false after 2 seconds of no keystrokes
    timeoutRef.current = setTimeout(() => {
      updatePresence({ isTyping: false });
    }, 2000);
  }, [updatePresence]);

  // Clean up on unmount
  useEffect(() => {
    return () => {
      if (timeoutRef.current) clearTimeout(timeoutRef.current);
      updatePresence({ isTyping: false });
    };
  }, [updatePresence]);

  // Get list of currently typing users
  const typingUsers = others
    .filter((other) => other.presence.isTyping)
    .map((other) => ({
      name: other.info.name,
      color: other.info.color,
    }));

  return { onKeyDown, typingUsers };
}

// components/TypingIndicator.tsx
export function TypingIndicator() {
  const { typingUsers } = useTypingIndicator();

  if (typingUsers.length === 0) return null;

  const text =
    typingUsers.length === 1
      ? `${typingUsers[0].name} is typing...`
      : typingUsers.length === 2
        ? `${typingUsers[0].name} and ${typingUsers[1].name} are typing...`
        : `${typingUsers[0].name} and ${typingUsers.length - 1} others are typing...`;

  return (
    <div className="flex items-center gap-2 text-sm text-gray-500 h-6">
      <TypingDots />
      <span>{text}</span>
    </div>
  );
}

function TypingDots() {
  return (
    <span className="flex gap-0.5">
      {[0, 1, 2].map((i) => (
        <span
          key={i}
          className="w-1.5 h-1.5 rounded-full bg-gray-400 animate-bounce"
          style={{ animationDelay: `${i * 150}ms` }}
        />
      ))}
    </span>
  );
}
```

### Custom Presence with Socket.io

If you're not using Liveblocks, you can build presence with Socket.io:

```typescript
// server/presence.ts
import { Server, Socket } from 'socket.io';

interface UserPresence {
  userId: string;
  name: string;
  avatar: string;
  color: string;
  status: 'active' | 'idle' | 'away';
  lastSeen: number;
  cursor?: { x: number; y: number } | null;
  currentPage?: string;
}

// In-memory presence store (use Redis for multi-server)
const presenceByRoom = new Map<string, Map<string, UserPresence>>();

export function setupPresence(io: Server) {
  io.on('connection', (socket: Socket) => {
    const user = socket.data.user;

    socket.on('join-room', (roomId: string) => {
      socket.join(roomId);

      // Add to presence
      if (!presenceByRoom.has(roomId)) {
        presenceByRoom.set(roomId, new Map());
      }

      const roomPresence = presenceByRoom.get(roomId)!;
      const presence: UserPresence = {
        userId: user.id,
        name: user.name,
        avatar: user.avatar,
        color: assignColor(roomPresence.size),
        status: 'active',
        lastSeen: Date.now(),
      };

      roomPresence.set(user.id, presence);

      // Send current presence list to the new user
      socket.emit('presence-sync', Array.from(roomPresence.values()));

      // Notify others
      socket.to(roomId).emit('presence-join', presence);
    });

    socket.on('presence-update', (data: { roomId: string; update: Partial<UserPresence> }) => {
      const roomPresence = presenceByRoom.get(data.roomId);
      if (!roomPresence) return;

      const current = roomPresence.get(user.id);
      if (!current) return;

      const updated = { ...current, ...data.update, lastSeen: Date.now() };
      roomPresence.set(user.id, updated);

      socket.to(data.roomId).emit('presence-update', {
        userId: user.id,
        ...data.update,
      });
    });

    socket.on('disconnect', () => {
      // Remove from all rooms
      for (const [roomId, roomPresence] of presenceByRoom) {
        if (roomPresence.has(user.id)) {
          roomPresence.delete(user.id);
          io.to(roomId).emit('presence-leave', { userId: user.id });

          // Clean up empty rooms
          if (roomPresence.size === 0) {
            presenceByRoom.delete(roomId);
          }
        }
      }
    });
  });
}

function assignColor(index: number): string {
  const colors = [
    '#E57373', '#81C784', '#64B5F6', '#FFD54F',
    '#BA68C8', '#4DB6AC', '#FF8A65', '#A1887F',
  ];
  return colors[index % colors.length];
}
```

### Presence in React Native

React Native introduces specific challenges for presence: the app can go to the background at any time, and you need to handle that gracefully.

```typescript
// hooks/usePresenceRN.ts (React Native)
import { useEffect, useRef } from 'react';
import { AppState, AppStateStatus } from 'react-native';
import { useSocketIO } from './useSocketIO';

export function usePresenceRN(roomId: string, userId: string) {
  const { socket } = useSocketIO(token);
  const appStateRef = useRef(AppState.currentState);

  useEffect(() => {
    if (!socket) return;

    // Join room and set active
    socket.emit('join-room', roomId);
    socket.emit('presence-update', {
      roomId,
      update: { status: 'active' },
    });

    // Handle app state changes
    const subscription = AppState.addEventListener(
      'change',
      (nextAppState: AppStateStatus) => {
        if (
          appStateRef.current === 'active' &&
          nextAppState.match(/inactive|background/)
        ) {
          // App going to background — mark as away
          socket.emit('presence-update', {
            roomId,
            update: { status: 'away' },
          });
        } else if (
          appStateRef.current.match(/inactive|background/) &&
          nextAppState === 'active'
        ) {
          // App coming to foreground — mark as active
          socket.emit('presence-update', {
            roomId,
            update: { status: 'active', lastSeen: Date.now() },
          });
        }

        appStateRef.current = nextAppState;
      }
    );

    return () => {
      subscription.remove();
      socket.emit('leave-room', roomId);
    };
  }, [socket, roomId, userId]);
}
```

---

## 4. LIVE CURSORS & AWARENESS

Live cursors -- showing where other users' mouse pointers or selections are -- is the feature that makes an app feel truly "multiplayer." It's also the feature where naive implementations create the worst performance problems.

### The Core Challenge: Frequency vs Smoothness

A mouse generates hundreds of move events per second. Broadcasting all of them would:
1. Flood the WebSocket with messages
2. Cause other clients to render hundreds of updates per second
3. Waste bandwidth (most intermediate positions don't matter)

But if you throttle too aggressively, remote cursors appear to teleport between positions instead of moving smoothly. The solution: **throttle sending, interpolate on receiving**.

### Throttled Cursor Broadcasting

```typescript
// hooks/useLiveCursors.ts
import { useCallback, useEffect, useRef } from 'react';
import { useUpdateMyPresence, useOthersMapped } from '../liveblocks.config';

const CURSOR_THROTTLE_MS = 50; // 20 updates per second max

export function useLiveCursors() {
  const updatePresence = useUpdateMyPresence();
  const lastUpdateRef = useRef(0);
  const rafRef = useRef<number | null>(null);
  const pendingCursorRef = useRef<{ x: number; y: number } | null>(null);

  const handlePointerMove = useCallback(
    (e: React.PointerEvent) => {
      const now = Date.now();

      // Store the latest position
      pendingCursorRef.current = { x: e.clientX, y: e.clientY };

      // Throttle: only send if enough time has passed
      if (now - lastUpdateRef.current >= CURSOR_THROTTLE_MS) {
        updatePresence({ cursor: pendingCursorRef.current });
        lastUpdateRef.current = now;
        pendingCursorRef.current = null;
      } else if (!rafRef.current) {
        // Schedule a final update after the throttle window
        rafRef.current = requestAnimationFrame(() => {
          if (pendingCursorRef.current) {
            updatePresence({ cursor: pendingCursorRef.current });
            lastUpdateRef.current = Date.now();
            pendingCursorRef.current = null;
          }
          rafRef.current = null;
        });
      }
    },
    [updatePresence]
  );

  const handlePointerLeave = useCallback(() => {
    updatePresence({ cursor: null });
  }, [updatePresence]);

  // Get other users' cursors
  const cursors = useOthersMapped((other) => ({
    cursor: other.presence.cursor,
    info: other.info,
  }));

  useEffect(() => {
    return () => {
      if (rafRef.current) cancelAnimationFrame(rafRef.current);
    };
  }, []);

  return {
    handlePointerMove,
    handlePointerLeave,
    cursors,
  };
}
```

### Rendering Remote Cursors with Smooth Interpolation

```typescript
// components/LiveCursors.tsx
import { useLiveCursors } from '../hooks/useLiveCursors';

export function LiveCursorsProvider({ children }: { children: React.ReactNode }) {
  const { handlePointerMove, handlePointerLeave, cursors } = useLiveCursors();

  return (
    <div
      onPointerMove={handlePointerMove}
      onPointerLeave={handlePointerLeave}
      className="relative"
      style={{ touchAction: 'none' }} // Prevent scrolling on touch
    >
      {children}

      {/* Render other users' cursors */}
      {cursors.map(([connectionId, { cursor, info }]) => {
        if (!cursor) return null;

        return (
          <RemoteCursor
            key={connectionId}
            x={cursor.x}
            y={cursor.y}
            name={info.name}
            color={info.color}
          />
        );
      })}
    </div>
  );
}

// Individual remote cursor with CSS transition smoothing
function RemoteCursor({
  x,
  y,
  name,
  color,
}: {
  x: number;
  y: number;
  name: string;
  color: string;
}) {
  return (
    <div
      className="pointer-events-none fixed top-0 left-0 z-50"
      style={{
        transform: `translate(${x}px, ${y}px)`,
        // CSS transition provides smooth interpolation between position updates
        // At 20 updates/sec (50ms throttle), 75ms transition creates overlap for smoothness
        transition: 'transform 75ms linear',
      }}
    >
      {/* Cursor SVG */}
      <svg
        width="24"
        height="36"
        viewBox="0 0 24 36"
        fill="none"
        className="drop-shadow-md"
      >
        <path
          d="M5.65376 12.3673H5.46026L5.31717 12.4976L0.500002 16.8829L0.500002 1.19841L11.7841 12.3673H5.65376Z"
          fill={color}
          stroke="white"
        />
      </svg>

      {/* Name label */}
      <div
        className="absolute left-5 top-5 px-2 py-0.5 rounded-full text-xs text-white whitespace-nowrap shadow-sm"
        style={{ backgroundColor: color }}
      >
        {name}
      </div>
    </div>
  );
}
```

### Selection Awareness

Beyond cursors, showing what other users have selected prevents conflicting edits:

```typescript
// hooks/useSelectionAwareness.ts
import { useCallback } from 'react';
import { useUpdateMyPresence, useOthers } from '../liveblocks.config';

export function useSelectionAwareness() {
  const updatePresence = useUpdateMyPresence();
  const others = useOthers();

  const selectItem = useCallback(
    (itemId: string | null) => {
      updatePresence({ selectedId: itemId });
    },
    [updatePresence]
  );

  // Get items that other users have selected
  const selectedByOthers = new Map<string, { name: string; color: string }>();
  for (const other of others) {
    if (other.presence.selectedId) {
      selectedByOthers.set(other.presence.selectedId, {
        name: other.info.name,
        color: other.info.color,
      });
    }
  }

  return { selectItem, selectedByOthers };
}

// components/SelectableCard.tsx
function SelectableCard({
  id,
  children,
}: {
  id: string;
  children: React.ReactNode;
}) {
  const { selectItem, selectedByOthers } = useSelectionAwareness();
  const otherUser = selectedByOthers.get(id);

  return (
    <div
      onClick={() => selectItem(id)}
      className="relative border rounded-lg p-4 cursor-pointer"
      style={{
        borderColor: otherUser ? otherUser.color : undefined,
        borderWidth: otherUser ? 2 : 1,
      }}
    >
      {children}

      {/* Show who's editing this */}
      {otherUser && (
        <div
          className="absolute -top-3 left-2 px-2 py-0.5 rounded-full text-xs text-white"
          style={{ backgroundColor: otherUser.color }}
        >
          {otherUser.name} is editing
        </div>
      )}
    </div>
  );
}
```

### Performance Considerations for Live Cursors

```
┌──────────────────────────────────────────────────────────────────┐
│  CURSOR PERFORMANCE CHECKLIST                                     │
│                                                                   │
│  1. THROTTLE outgoing updates (16-50ms, depending on needs)       │
│     - 50ms = 20 updates/sec (good for most cases)                │
│     - 16ms = 60 updates/sec (for pixel-perfect apps like Figma)  │
│                                                                   │
│  2. USE CSS TRANSITIONS for interpolation, not JS animation       │
│     - transform + transition is GPU-accelerated                   │
│     - Avoid animating top/left (triggers layout)                  │
│                                                                   │
│  3. RENDER CURSORS in a SEPARATE LAYER                            │
│     - Use position: fixed + pointer-events: none                  │
│     - This prevents cursor rendering from triggering              │
│       layout/paint on the main content                            │
│                                                                   │
│  4. BATCH cursor updates with requestAnimationFrame               │
│     - Don't update state on every mouse event                    │
│     - Collect and flush once per frame                            │
│                                                                   │
│  5. HIDE cursors of inactive users after ~10 seconds              │
│     - Stale cursors are confusing, not helpful                    │
│                                                                   │
│  6. LIMIT visible cursors                                         │
│     - 5-10 cursors is manageable; 50 cursors is chaos             │
│     - Show cursors of users on the same "page" or section only   │
└──────────────────────────────────────────────────────────────────┘
```

---

## 5. COLLABORATIVE EDITING

This is the hardest real-time problem. Two users editing the same text simultaneously, without losing either person's changes, without the text becoming garbled, without needing a central coordinator, and ideally working offline too. There are two proven approaches.

### Operational Transformation (OT)

OT is the approach Google Docs uses. It was invented in the late 1980s and has been refined for decades.

**How it works:**

```
The core idea: transform operations against each other so they
produce the same result regardless of the order they're applied.

User A types "Hello" at position 0
User B types "World" at position 0
Both happen simultaneously.

Without OT:
  Server receives A's op: insert "Hello" at 0
  Server receives B's op: insert "World" at 0
  Result depends on order: "HelloWorld" or "WorldHello"

With OT:
  Server receives A's op: insert "Hello" at 0 → applies it
  Server receives B's op: insert "World" at 0
    → TRANSFORMS it against A's op
    → "A inserted 5 chars before position 0, so B's position shifts to 5"
    → Transformed op: insert "World" at 5
  Result: "HelloWorld" (deterministic)
```

**Characteristics of OT:**
- Requires a central server to coordinate transformations
- Complex to implement correctly (the transformation functions for all operation combinations are tricky)
- Well-understood, battle-tested (Google Docs has used it for 15+ years)
- Does NOT work offline (needs the server to transform operations)

### CRDTs (Conflict-free Replicated Data Types)

CRDTs are the newer approach. They use mathematical structures that guarantee convergence without a central coordinator.

**How they work:**

```
The core idea: design data structures where all operations commute --
meaning the order doesn't matter. Any order of applying operations
produces the same final result.

There are two kinds:
- Operation-based (CmRDT): broadcast operations, apply them in any order
- State-based (CvRDT): broadcast full state, merge with a function
  that is commutative, associative, and idempotent

For text editing, Yjs uses a sequence CRDT (YATA algorithm):
- Each character has a unique ID (lamport timestamp + client ID)
- Each character knows which character it was inserted after
- Deletions mark characters as "tombstones" (never truly deleted)
- Any peer can reconstruct the correct document by sorting
  characters by their insertion relationships
```

### OT vs CRDTs: The Decision

```
┌─────────────────────┬──────────────────────┬─────────────────────┐
│                     │  OT                  │  CRDTs              │
├─────────────────────┼──────────────────────┼─────────────────────┤
│ Central server      │ Required             │ Not required        │
│ Offline editing     │ Not possible         │ Fully supported     │
│ P2P possible        │ No                   │ Yes                 │
│ Implementation      │ Very complex         │ Complex (but libs   │
│ complexity          │                      │ handle it)          │
│ Memory overhead     │ Low                  │ Higher (tombstones) │
│ Proven at scale     │ Google Docs (15 yr+) │ Figma, VS Code      │
│ Library ecosystem   │ Sparse               │ Yjs, Automerge      │
│ Conflict-free       │ By coordination      │ By mathematical     │
│                     │                      │ guarantee           │
├─────────────────────┼──────────────────────┼─────────────────────┤
│ RECOMMENDATION      │ Only if you're       │ Default choice for  │
│                     │ using Google's        │ new projects.       │
│                     │ infrastructure or     │ Use Yjs.            │
│                     │ have an existing      │                     │
│                     │ OT implementation     │                     │
└─────────────────────┴──────────────────────┴─────────────────────┘
```

### Yjs: The Practical CRDT Library

Yjs is the dominant CRDT library for JavaScript. It provides shared data types (text, arrays, maps) that can be edited concurrently and will always converge to the same state.

```typescript
// lib/yjs-setup.ts
import * as Y from 'yjs';
import { WebsocketProvider } from 'y-websocket';

// Create a Yjs document
const ydoc = new Y.Doc();

// Connect to a WebSocket server for syncing
// y-websocket provides both the client provider and a ready-to-use server
const provider = new WebsocketProvider(
  'wss://your-yjs-server.example.com',
  'my-document-room',  // Room name — all clients with the same name share the doc
  ydoc,
  {
    connect: true,
    // Authentication
    params: { token: 'your-auth-token' },
    // Awareness (presence) is built into the provider
  }
);

// Connection status
provider.on('status', (event: { status: string }) => {
  console.log('Connection status:', event.status); // 'connecting', 'connected', 'disconnected'
});

// --- Shared Data Types ---

// Shared Text (for collaborative text editing)
const ytext = ydoc.getText('document-content');

// Shared Map (for key-value data like document metadata)
const ymeta = ydoc.getMap('document-meta');

// Shared Array (for ordered collections like task lists)
const ytasks = ydoc.getArray('tasks');

// --- Observing Changes ---

ytext.observe((event) => {
  // event.delta contains the changes in Quill delta format
  // [{ retain: 5 }, { insert: 'hello' }, { delete: 3 }]
  console.log('Text changed:', event.delta);
});

ymeta.observe((event) => {
  event.changes.keys.forEach((change, key) => {
    if (change.action === 'add') {
      console.log(`Key "${key}" added:`, ymeta.get(key));
    } else if (change.action === 'update') {
      console.log(`Key "${key}" updated:`, ymeta.get(key));
    } else if (change.action === 'delete') {
      console.log(`Key "${key}" deleted`);
    }
  });
});

// --- Making Changes ---

// All changes within a transaction are atomic
ydoc.transact(() => {
  ytext.insert(0, 'Hello, world!');
  ymeta.set('title', 'My Document');
  ymeta.set('lastModified', Date.now());
});

// --- Awareness (Presence) ---
const awareness = provider.awareness;

// Set local user state
awareness.setLocalStateField('user', {
  name: 'Alice',
  color: '#E57373',
  cursor: null,
});

// Listen for awareness changes
awareness.on('change', () => {
  const states = awareness.getStates();
  // Map of clientID → { user: { name, color, cursor } }
  states.forEach((state, clientId) => {
    if (clientId !== ydoc.clientID) {
      console.log('Remote user:', state.user?.name);
    }
  });
});
```

### Building a Collaborative Text Editor with Yjs + TipTap

TipTap is a headless rich text editor built on ProseMirror that has first-class Yjs integration:

```typescript
// components/CollaborativeEditor.tsx
import { useEditor, EditorContent } from '@tiptap/react';
import StarterKit from '@tiptap/starter-kit';
import Collaboration from '@tiptap/extension-collaboration';
import CollaborationCursor from '@tiptap/extension-collaboration-cursor';
import * as Y from 'yjs';
import { WebsocketProvider } from 'y-websocket';
import { useEffect, useMemo } from 'react';

interface CollaborativeEditorProps {
  documentId: string;
  userName: string;
  userColor: string;
}

export function CollaborativeEditor({
  documentId,
  userName,
  userColor,
}: CollaborativeEditorProps) {
  // Create Yjs document and provider (memoized to persist across renders)
  const { ydoc, provider } = useMemo(() => {
    const doc = new Y.Doc();
    const wsProvider = new WebsocketProvider(
      process.env.NEXT_PUBLIC_YJS_WS_URL!,
      documentId,
      doc
    );
    return { ydoc: doc, provider: wsProvider };
  }, [documentId]);

  // Clean up on unmount
  useEffect(() => {
    return () => {
      provider.destroy();
      ydoc.destroy();
    };
  }, [provider, ydoc]);

  const editor = useEditor({
    extensions: [
      StarterKit.configure({
        // The Collaboration extension handles history (undo/redo),
        // so we disable the default history
        history: false,
      }),
      // Collaborative editing via Yjs
      Collaboration.configure({
        document: ydoc,
        field: 'content', // Name of the Y.XmlFragment
      }),
      // Collaborative cursors via Yjs awareness
      CollaborationCursor.configure({
        provider,
        user: {
          name: userName,
          color: userColor,
        },
      }),
    ],
  });

  return (
    <div className="collaborative-editor">
      {/* Connection status */}
      <ConnectionBadge provider={provider} />

      {/* Editor toolbar */}
      {editor && <EditorToolbar editor={editor} />}

      {/* The editor itself */}
      <EditorContent
        editor={editor}
        className="prose prose-lg max-w-none min-h-[400px] p-4 border rounded-lg
                   focus-within:ring-2 focus-within:ring-blue-500"
      />
    </div>
  );
}

function ConnectionBadge({ provider }: { provider: WebsocketProvider }) {
  const [status, setStatus] = useState('connecting');

  useEffect(() => {
    const handler = ({ status: s }: { status: string }) => setStatus(s);
    provider.on('status', handler);
    return () => provider.off('status', handler);
  }, [provider]);

  return (
    <div className="flex items-center gap-2 text-sm mb-2">
      <span
        className={`w-2 h-2 rounded-full ${
          status === 'connected' ? 'bg-green-500' : 'bg-yellow-500 animate-pulse'
        }`}
      />
      <span className="text-gray-500">
        {status === 'connected' ? 'Synced' : 'Syncing...'}
      </span>
    </div>
  );
}

function EditorToolbar({ editor }: { editor: any }) {
  return (
    <div className="flex gap-1 mb-2 p-2 border rounded-lg bg-gray-50">
      <button
        onClick={() => editor.chain().focus().toggleBold().run()}
        className={`px-3 py-1 rounded ${
          editor.isActive('bold') ? 'bg-gray-200 font-bold' : ''
        }`}
      >
        B
      </button>
      <button
        onClick={() => editor.chain().focus().toggleItalic().run()}
        className={`px-3 py-1 rounded ${
          editor.isActive('italic') ? 'bg-gray-200 italic' : ''
        }`}
      >
        I
      </button>
      <button
        onClick={() => editor.chain().focus().toggleHeading({ level: 2 }).run()}
        className={`px-3 py-1 rounded ${
          editor.isActive('heading', { level: 2 }) ? 'bg-gray-200' : ''
        }`}
      >
        H2
      </button>
      <button
        onClick={() => editor.chain().focus().toggleBulletList().run()}
        className={`px-3 py-1 rounded ${
          editor.isActive('bulletList') ? 'bg-gray-200' : ''
        }`}
      >
        List
      </button>
    </div>
  );
}
```

### Collaborative Data with Liveblocks Storage

Liveblocks provides a managed alternative to Yjs with a simpler API. Instead of learning Yjs data types, you work with familiar-feeling structures:

```typescript
// liveblocks.config.ts (extended from Section 3)
import { createClient } from '@liveblocks/client';
import { createRoomContext } from '@liveblocks/react';
import { LiveList, LiveMap, LiveObject } from '@liveblocks/client';

type Task = {
  id: string;
  title: string;
  status: 'todo' | 'in-progress' | 'done';
  assignee: string | null;
  createdAt: number;
};

type Storage = {
  tasks: LiveList<LiveObject<Task>>;
  boardConfig: LiveObject<{
    name: string;
    columns: LiveList<string>;
  }>;
};

// ... (rest of config as before)
```

```typescript
// components/CollaborativeTaskBoard.tsx
import { useStorage, useMutation } from '../liveblocks.config';
import { LiveObject } from '@liveblocks/client';

export function CollaborativeTaskBoard() {
  // useStorage subscribes to storage updates in real-time
  const tasks = useStorage((root) => root.tasks);

  // useMutation provides a function that can modify shared storage
  const addTask = useMutation(({ storage }, title: string) => {
    const tasks = storage.get('tasks');
    tasks.push(
      new LiveObject({
        id: crypto.randomUUID(),
        title,
        status: 'todo',
        assignee: null,
        createdAt: Date.now(),
      })
    );
  }, []);

  const updateTaskStatus = useMutation(
    ({ storage }, taskId: string, status: Task['status']) => {
      const tasks = storage.get('tasks');
      const taskIndex = tasks.findIndex((t) => t.get('id') === taskId);
      if (taskIndex !== -1) {
        tasks.get(taskIndex)?.set('status', status);
      }
    },
    []
  );

  const deleteTask = useMutation(({ storage }, taskId: string) => {
    const tasks = storage.get('tasks');
    const taskIndex = tasks.findIndex((t) => t.get('id') === taskId);
    if (taskIndex !== -1) {
      tasks.delete(taskIndex);
    }
  }, []);

  if (tasks === null) {
    return <div>Loading board...</div>;
  }

  const columns = ['todo', 'in-progress', 'done'] as const;

  return (
    <div className="grid grid-cols-3 gap-4">
      {columns.map((status) => (
        <div key={status} className="bg-gray-50 rounded-lg p-4">
          <h3 className="font-semibold mb-3 capitalize">
            {status.replace('-', ' ')}
          </h3>

          <div className="space-y-2">
            {tasks
              .filter((task) => task.status === status)
              .map((task) => (
                <TaskCard
                  key={task.id}
                  task={task}
                  onStatusChange={(newStatus) =>
                    updateTaskStatus(task.id, newStatus)
                  }
                  onDelete={() => deleteTask(task.id)}
                />
              ))}
          </div>

          {status === 'todo' && (
            <AddTaskButton onAdd={(title) => addTask(title)} />
          )}
        </div>
      ))}
    </div>
  );
}
```

### Yjs Persistence: Saving to a Database

Yjs documents exist in memory. You need persistence to survive server restarts:

```typescript
// server/yjs-persistence.ts
import * as Y from 'yjs';
import { MongoClient } from 'mongodb';

// Yjs documents can be serialized to a compact binary format
// and restored from it later

interface DocumentRecord {
  docName: string;
  state: Buffer;      // Y.encodeStateAsUpdate output
  updatedAt: Date;
}

export class YjsPersistence {
  private db: MongoClient;

  constructor(mongoUri: string) {
    this.db = new MongoClient(mongoUri);
  }

  async loadDocument(docName: string): Promise<Y.Doc> {
    const ydoc = new Y.Doc();

    const collection = this.db.db('collab').collection<DocumentRecord>('documents');
    const record = await collection.findOne({ docName });

    if (record) {
      // Apply the saved state to the new document
      Y.applyUpdate(ydoc, new Uint8Array(record.state));
    }

    return ydoc;
  }

  async saveDocument(docName: string, ydoc: Y.Doc): Promise<void> {
    const state = Y.encodeStateAsUpdate(ydoc);

    const collection = this.db.db('collab').collection<DocumentRecord>('documents');
    await collection.updateOne(
      { docName },
      {
        $set: {
          state: Buffer.from(state),
          updatedAt: new Date(),
        },
      },
      { upsert: true }
    );
  }

  // Incremental persistence: save only the new updates
  // This is more efficient for frequent saves
  async saveUpdate(docName: string, update: Uint8Array): Promise<void> {
    const collection = this.db.db('collab').collection('document_updates');
    await collection.insertOne({
      docName,
      update: Buffer.from(update),
      createdAt: new Date(),
    });
  }

  // Periodically compact updates into a single snapshot
  async compactDocument(docName: string): Promise<void> {
    const ydoc = await this.loadUpdates(docName);
    await this.saveDocument(docName, ydoc);

    // Delete individual updates
    const collection = this.db.db('collab').collection('document_updates');
    await collection.deleteMany({ docName });
  }

  private async loadUpdates(docName: string): Promise<Y.Doc> {
    const ydoc = new Y.Doc();

    // Load base snapshot
    const base = await this.db
      .db('collab')
      .collection<DocumentRecord>('documents')
      .findOne({ docName });

    if (base) {
      Y.applyUpdate(ydoc, new Uint8Array(base.state));
    }

    // Apply incremental updates
    const updates = await this.db
      .db('collab')
      .collection('document_updates')
      .find({ docName })
      .sort({ createdAt: 1 })
      .toArray();

    for (const record of updates) {
      Y.applyUpdate(ydoc, new Uint8Array(record.update));
    }

    return ydoc;
  }
}
```

---

## 6. CONFLICT RESOLUTION

When multiple users edit the same data simultaneously, conflicts are inevitable. Your strategy for resolving them determines whether your app feels reliable or chaotic.

### Strategy 1: Last-Write-Wins (LWW)

The simplest strategy. Each update has a timestamp; the latest timestamp wins.

```typescript
// Last-Write-Wins implementation
interface LWWValue<T> {
  value: T;
  timestamp: number;
  clientId: string;
}

function merge<T>(local: LWWValue<T>, remote: LWWValue<T>): LWWValue<T> {
  if (remote.timestamp > local.timestamp) {
    return remote;
  }
  if (remote.timestamp === local.timestamp) {
    // Tiebreaker: higher client ID wins (deterministic)
    return remote.clientId > local.clientId ? remote : local;
  }
  return local;
}
```

```
WHEN TO USE LWW:
  ✓ Simple key-value data (user settings, feature flags)
  ✓ Data where "last edit is correct" is true (status fields)
  ✓ Low conflict probability (few concurrent editors)

WHEN NOT TO USE:
  ✗ Text editing (you'd lose entire paragraphs)
  ✗ Collaborative lists (items would appear/disappear)
  ✗ Any data where partial merging makes sense
```

### Strategy 2: Field-Level Merge

Instead of LWW on the entire object, apply LWW per field. This lets two users edit different fields of the same object without conflict.

```typescript
// Field-level merge
interface FieldTimestamps {
  [field: string]: number;
}

interface VersionedObject<T> {
  data: T;
  fieldTimestamps: FieldTimestamps;
}

function mergeObjects<T extends Record<string, unknown>>(
  local: VersionedObject<T>,
  remote: VersionedObject<T>
): VersionedObject<T> {
  const merged: Record<string, unknown> = { ...local.data };
  const timestamps: FieldTimestamps = { ...local.fieldTimestamps };

  for (const field of Object.keys(remote.data)) {
    const remoteTime = remote.fieldTimestamps[field] ?? 0;
    const localTime = local.fieldTimestamps[field] ?? 0;

    if (remoteTime > localTime) {
      merged[field] = remote.data[field];
      timestamps[field] = remoteTime;
    }
  }

  return {
    data: merged as T,
    fieldTimestamps: timestamps,
  };
}

// Example: Two users edit the same task
// User A changes title at T=100
// User B changes status at T=101
// Result: Both changes preserved (title from A, status from B)
```

### Strategy 3: Vector Clocks

Vector clocks track causal ordering -- they tell you whether event A happened before event B, after it, or concurrently (which indicates a true conflict).

```typescript
// Vector clock implementation
type VectorClock = Record<string, number>;

function increment(clock: VectorClock, nodeId: string): VectorClock {
  return {
    ...clock,
    [nodeId]: (clock[nodeId] ?? 0) + 1,
  };
}

function merge(a: VectorClock, b: VectorClock): VectorClock {
  const merged: VectorClock = { ...a };
  for (const [nodeId, time] of Object.entries(b)) {
    merged[nodeId] = Math.max(merged[nodeId] ?? 0, time);
  }
  return merged;
}

type Ordering = 'before' | 'after' | 'concurrent' | 'equal';

function compare(a: VectorClock, b: VectorClock): Ordering {
  let aBeforeB = false;
  let bBeforeA = false;

  const allNodes = new Set([...Object.keys(a), ...Object.keys(b)]);

  for (const node of allNodes) {
    const aTime = a[node] ?? 0;
    const bTime = b[node] ?? 0;

    if (aTime < bTime) aBeforeB = true;
    if (aTime > bTime) bBeforeA = true;
  }

  if (!aBeforeB && !bBeforeA) return 'equal';
  if (aBeforeB && !bBeforeA) return 'before';
  if (!aBeforeB && bBeforeA) return 'after';
  return 'concurrent'; // True conflict — both edited independently
}

// Usage:
// const ordering = compare(localClock, remoteClock);
// if (ordering === 'before') → remote is newer, accept it
// if (ordering === 'after') → local is newer, keep it
// if (ordering === 'concurrent') → TRUE CONFLICT, need manual resolution
// if (ordering === 'equal') → same state, no-op
```

### Strategy 4: CRDTs (Automatic Resolution)

CRDTs eliminate conflicts by design. Covered in detail in Section 5 with Yjs. The key insight: if you can express your data as a CRDT, you never need manual conflict resolution.

**Common CRDT types and their use cases:**

```
┌──────────────────────┬──────────────────────────────────────────┐
│  CRDT Type           │  Use Case                                │
├──────────────────────┼──────────────────────────────────────────┤
│  G-Counter           │  View counts, like counts (only grows)   │
│  PN-Counter          │  Vote counts (can increment & decrement) │
│  LWW-Register        │  Single value with last-write-wins       │
│  MV-Register         │  Single value preserving all concurrent  │
│                      │  writes (user resolves)                  │
│  G-Set               │  Tags, labels (add-only)                 │
│  OR-Set (AWSet)      │  Membership, shopping carts              │
│                      │  (add and remove)                        │
│  LWW-Map             │  Key-value store (per-field LWW)         │
│  RGA / YATA          │  Ordered text / arrays                   │
│  (Yjs, Automerge)    │  (collaborative editing)                 │
└──────────────────────┴──────────────────────────────────────────┘
```

### Conflict Resolution Decision Framework

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                  │
│  What kind of data are you syncing?                              │
│                                                                  │
│  ├── Simple key-value (settings, preferences)                    │
│  │   └──→ Last-Write-Wins. Done.                                │
│  │                                                               │
│  ├── Structured objects (user profiles, project config)          │
│  │   └──→ Field-Level Merge (LWW per field)                     │
│  │                                                               │
│  ├── Ordered collections (task lists, playlists)                 │
│  │   ├── Low concurrency? → Field-level merge + position index  │
│  │   └── High concurrency? → CRDT (Yjs Array)                  │
│  │                                                               │
│  ├── Rich text / code                                            │
│  │   └──→ CRDT (Yjs) or OT. No other option works.             │
│  │                                                               │
│  └── Data where conflicts MUST be shown to the user             │
│      └──→ Vector Clocks to detect conflicts +                   │
│           UI for manual resolution (like git merge conflicts)   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 7. REAL-TIME IN REACT NATIVE

React Native adds layers of complexity to real-time features. The app lifecycle, battery constraints, and mobile network conditions all need special handling.

### WebSocket Connection Management

```typescript
// lib/realtime-rn.ts
import { AppState, AppStateStatus, Platform } from 'react-native';
import NetInfo, { NetInfoState } from '@react-native-community/netinfo';
import { WebSocketManager } from './websocket-manager';

export class ReactNativeRealtimeManager {
  private wsManager: WebSocketManager;
  private appState: AppStateStatus = 'active';
  private isConnected = true;
  private shouldBeConnected = false;

  constructor(url: string, token: string) {
    this.wsManager = new WebSocketManager({
      url,
      auth: { token },
      reconnect: {
        enabled: true,
        maxAttempts: 15,
        baseDelay: 1000,
        maxDelay: 60000, // Longer max delay for mobile
        backoffMultiplier: 2,
      },
      heartbeat: {
        enabled: true,
        interval: 30000,
        timeout: 10000, // Longer timeout for mobile networks
        message: 'ping',
      },
    });
  }

  start(): void {
    this.shouldBeConnected = true;
    this.setupAppStateListener();
    this.setupNetInfoListener();
    this.connectIfShould();
  }

  stop(): void {
    this.shouldBeConnected = false;
    this.wsManager.disconnect();
  }

  private setupAppStateListener(): void {
    AppState.addEventListener('change', (nextAppState: AppStateStatus) => {
      const wasActive = this.appState === 'active';
      const isNowActive = nextAppState === 'active';
      const isNowBackground = nextAppState === 'background';

      this.appState = nextAppState;

      if (wasActive && isNowBackground) {
        // App going to background
        // On iOS, the WebSocket will be killed ~30 seconds after backgrounding
        // On Android, it depends on battery optimization settings
        //
        // Strategy: disconnect cleanly rather than letting the OS kill it
        // This prevents the server from holding a dead connection
        this.wsManager.disconnect();
      }

      if (!wasActive && isNowActive) {
        // App coming to foreground
        // Reconnect and fetch any missed updates
        this.connectIfShould();
      }
    });
  }

  private setupNetInfoListener(): void {
    NetInfo.addEventListener((state: NetInfoState) => {
      const wasConnected = this.isConnected;
      this.isConnected = Boolean(state.isConnected && state.isInternetReachable);

      if (!wasConnected && this.isConnected) {
        // Network restored
        this.connectIfShould();
      }

      if (wasConnected && !this.isConnected) {
        // Network lost — let the WebSocket manager handle reconnection
        // Don't force disconnect; it'll detect the failure via heartbeat
      }
    });
  }

  private connectIfShould(): void {
    if (
      this.shouldBeConnected &&
      this.isConnected &&
      this.appState === 'active'
    ) {
      this.wsManager.connect();
    }
  }

  // Expose the underlying manager for event handling
  get manager(): WebSocketManager {
    return this.wsManager;
  }
}
```

### Integrating Real-Time Updates with TanStack Query

Real-time data should flow through your existing cache layer, not bypass it:

```typescript
// hooks/useRealtimeQuery.ts
import { useEffect } from 'react';
import { useQueryClient, useQuery, UseQueryOptions } from '@tanstack/react-query';

/**
 * Combines TanStack Query with WebSocket updates.
 *
 * Initial data comes from the API (TanStack Query handles loading/caching).
 * Subsequent updates come via WebSocket and are merged into the cache.
 *
 * This gives you:
 * - Offline support (TanStack Query's cache persists)
 * - Loading states, error handling (TanStack Query)
 * - Real-time updates (WebSocket)
 * - Background refetch to catch missed messages (TanStack Query)
 */
export function useRealtimeQuery<T>(
  queryKey: readonly unknown[],
  queryFn: () => Promise<T>,
  wsEvent: string,
  updateFn: (oldData: T, wsPayload: unknown) => T,
  options?: Omit<UseQueryOptions<T>, 'queryKey' | 'queryFn'>
) {
  const queryClient = useQueryClient();
  const query = useQuery({ queryKey, queryFn, ...options });

  useEffect(() => {
    // Subscribe to WebSocket event
    const unsubscribe = wsManager.on(wsEvent, (payload) => {
      // Update the TanStack Query cache directly
      queryClient.setQueryData<T>(queryKey, (oldData) => {
        if (!oldData) return oldData;
        return updateFn(oldData, payload);
      });
    });

    return unsubscribe;
  }, [queryKey, wsEvent, queryClient, updateFn]);

  return query;
}

// Usage: Real-time task list
function useRealtimeTasks(projectId: string) {
  return useRealtimeQuery(
    ['tasks', projectId],
    () => api.getTasks(projectId),
    `tasks:${projectId}:update`,
    (tasks, payload) => {
      const event = payload as {
        action: 'created' | 'updated' | 'deleted';
        task: Task;
      };

      switch (event.action) {
        case 'created':
          return [...tasks, event.task];
        case 'updated':
          return tasks.map((t) => (t.id === event.task.id ? event.task : t));
        case 'deleted':
          return tasks.filter((t) => t.id !== event.task.id);
        default:
          return tasks;
      }
    }
  );
}
```

### Battery-Conscious Real-Time

Mobile devices have limited battery. Real-time features can be a significant drain if not managed carefully:

```typescript
// hooks/useBatteryAwareRealtime.ts
import { useEffect, useRef, useState } from 'react';
import { AppState } from 'react-native';

interface BatteryAwareConfig {
  /** Full real-time: WebSocket + presence + cursors */
  full: {
    heartbeatInterval: number;
    cursorThrottle: number;
    presenceEnabled: boolean;
  };
  /** Reduced: WebSocket for data only, no presence/cursors */
  reduced: {
    heartbeatInterval: number;
    cursorThrottle: number;
    presenceEnabled: boolean;
  };
  /** Minimal: Long-polling only */
  minimal: {
    pollInterval: number;
  };
}

const DEFAULT_CONFIG: BatteryAwareConfig = {
  full: {
    heartbeatInterval: 25000,
    cursorThrottle: 50,
    presenceEnabled: true,
  },
  reduced: {
    heartbeatInterval: 60000,
    cursorThrottle: 200,
    presenceEnabled: false,
  },
  minimal: {
    pollInterval: 30000,
  },
};

type RealtimeMode = 'full' | 'reduced' | 'minimal';

export function useRealtimeMode(): RealtimeMode {
  const [mode, setMode] = useState<RealtimeMode>('full');

  useEffect(() => {
    // Check battery level if available (Web Battery API or RN bridge)
    // Many RN apps use react-native-device-info for battery level
    checkBatteryLevel().then((level) => {
      if (level !== null) {
        if (level < 0.1) {
          setMode('minimal');  // < 10% battery: minimal real-time
        } else if (level < 0.25) {
          setMode('reduced');  // < 25% battery: reduced features
        } else {
          setMode('full');     // Good battery: full features
        }
      }
    });

    // Also reduce when data saver mode is on
    checkDataSaverMode().then((isOn) => {
      if (isOn) setMode('reduced');
    });
  }, []);

  return mode;
}

async function checkBatteryLevel(): Promise<number | null> {
  try {
    // For React Native, use react-native-device-info
    // For web, use navigator.getBattery()
    if ('getBattery' in navigator) {
      const battery = await (navigator as any).getBattery();
      return battery.level;
    }
    return null;
  } catch {
    return null;
  }
}

async function checkDataSaverMode(): Promise<boolean> {
  // Web: navigator.connection?.saveData
  // RN: Check via native module
  if ('connection' in navigator) {
    return (navigator as any).connection?.saveData ?? false;
  }
  return false;
}
```

---

## 8. SUPABASE REALTIME

Supabase Realtime is uniquely powerful because it integrates directly with your Postgres database. Database changes automatically broadcast to connected clients, with row-level security applied. This means you get real-time features without building a separate event system.

### How Supabase Realtime Works

```
┌─────────────────────────────────────────────────────────────┐
│                                                              │
│  1. Postgres Change →  Supabase intercepts via logical       │
│                        replication (WAL)                     │
│                                                              │
│  2. Broadcast     →  Sends change to all subscribed          │
│                      WebSocket clients                       │
│                                                              │
│  3. Authorization →  Row Level Security (RLS) policies       │
│                      filter what each user sees              │
│                                                              │
│  Three Channel Types:                                        │
│                                                              │
│  • Postgres Changes: Listen to INSERT/UPDATE/DELETE on       │
│    specific tables. The database IS the event source.        │
│                                                              │
│  • Broadcast: Ephemeral pub/sub messages between clients.    │
│    Not stored. Think cursor positions, typing indicators.    │
│                                                              │
│  • Presence: Track and sync shared state between clients.    │
│    Who's online, user status.                                │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Listening to Database Changes

```typescript
// hooks/useSupabaseRealtime.ts
import { useEffect } from 'react';
import { useQueryClient } from '@tanstack/react-query';
import { supabase } from '../lib/supabase';
import type { RealtimePostgresChangesPayload } from '@supabase/supabase-js';

interface Task {
  id: string;
  title: string;
  status: string;
  project_id: string;
  assignee_id: string;
  created_at: string;
}

/**
 * Subscribe to real-time changes on the tasks table.
 * Updates are filtered by project_id and respect RLS policies.
 */
export function useRealtimeTasks(projectId: string) {
  const queryClient = useQueryClient();

  useEffect(() => {
    const channel = supabase
      .channel(`tasks:${projectId}`)
      // Listen to INSERT events
      .on<Task>(
        'postgres_changes',
        {
          event: 'INSERT',
          schema: 'public',
          table: 'tasks',
          filter: `project_id=eq.${projectId}`,
        },
        (payload: RealtimePostgresChangesPayload<Task>) => {
          const newTask = payload.new;
          queryClient.setQueryData<Task[]>(
            ['tasks', projectId],
            (old = []) => [...old, newTask]
          );
        }
      )
      // Listen to UPDATE events
      .on<Task>(
        'postgres_changes',
        {
          event: 'UPDATE',
          schema: 'public',
          table: 'tasks',
          filter: `project_id=eq.${projectId}`,
        },
        (payload: RealtimePostgresChangesPayload<Task>) => {
          const updated = payload.new;
          queryClient.setQueryData<Task[]>(
            ['tasks', projectId],
            (old = []) =>
              old.map((task) => (task.id === updated.id ? updated : task))
          );
        }
      )
      // Listen to DELETE events
      .on<Task>(
        'postgres_changes',
        {
          event: 'DELETE',
          schema: 'public',
          table: 'tasks',
          filter: `project_id=eq.${projectId}`,
        },
        (payload: RealtimePostgresChangesPayload<Task>) => {
          const deleted = payload.old;
          queryClient.setQueryData<Task[]>(
            ['tasks', projectId],
            (old = []) => old.filter((task) => task.id !== deleted.id)
          );
        }
      )
      .subscribe((status) => {
        console.log('Realtime subscription status:', status);
      });

    return () => {
      supabase.removeChannel(channel);
    };
  }, [projectId, queryClient]);
}
```

### Supabase Broadcast (Ephemeral Messages)

Broadcast channels are for messages that don't need persistence -- cursor positions, typing indicators, ephemeral UI state:

```typescript
// hooks/useSupabaseBroadcast.ts
import { useEffect, useCallback, useRef } from 'react';
import { supabase } from '../lib/supabase';
import type { RealtimeChannel } from '@supabase/supabase-js';

export function useSupabaseBroadcast<T>(channelName: string, eventName: string) {
  const channelRef = useRef<RealtimeChannel | null>(null);
  const handlersRef = useRef<((payload: T) => void)[]>([]);

  useEffect(() => {
    const channel = supabase.channel(channelName, {
      config: {
        broadcast: {
          // self: true means the sender also receives the message
          self: false,
        },
      },
    });

    channel
      .on('broadcast', { event: eventName }, (payload) => {
        handlersRef.current.forEach((handler) =>
          handler(payload.payload as T)
        );
      })
      .subscribe();

    channelRef.current = channel;

    return () => {
      supabase.removeChannel(channel);
    };
  }, [channelName, eventName]);

  const broadcast = useCallback(
    (payload: T) => {
      channelRef.current?.send({
        type: 'broadcast',
        event: eventName,
        payload,
      });
    },
    [eventName]
  );

  const onMessage = useCallback((handler: (payload: T) => void) => {
    handlersRef.current.push(handler);
    return () => {
      handlersRef.current = handlersRef.current.filter((h) => h !== handler);
    };
  }, []);

  return { broadcast, onMessage };
}

// Usage: cursor broadcasting
function useCursorBroadcast(documentId: string) {
  const { broadcast, onMessage } = useSupabaseBroadcast<{
    userId: string;
    x: number;
    y: number;
  }>(`cursors:${documentId}`, 'cursor-move');

  return { broadcastCursor: broadcast, onCursorMove: onMessage };
}
```

### Supabase Presence

```typescript
// hooks/useSupabasePresence.ts
import { useEffect, useState, useCallback } from 'react';
import { supabase } from '../lib/supabase';
import type { RealtimeChannel, RealtimePresenceState } from '@supabase/supabase-js';

interface PresenceUser {
  userId: string;
  name: string;
  avatar: string;
  onlineAt: string;
}

export function useSupabasePresence(
  channelName: string,
  currentUser: PresenceUser
) {
  const [onlineUsers, setOnlineUsers] = useState<PresenceUser[]>([]);
  const [channel, setChannel] = useState<RealtimeChannel | null>(null);

  useEffect(() => {
    const ch = supabase.channel(channelName);

    ch.on('presence', { event: 'sync' }, () => {
      // presenceState returns a map of presence key → array of presences
      const state = ch.presenceState<PresenceUser>();
      const users: PresenceUser[] = [];

      Object.values(state).forEach((presences) => {
        presences.forEach((p) => {
          users.push({
            userId: p.userId,
            name: p.name,
            avatar: p.avatar,
            onlineAt: p.onlineAt,
          });
        });
      });

      setOnlineUsers(users);
    })
    .on('presence', { event: 'join' }, ({ newPresences }) => {
      console.log('User joined:', newPresences);
    })
    .on('presence', { event: 'leave' }, ({ leftPresences }) => {
      console.log('User left:', leftPresences);
    })
    .subscribe(async (status) => {
      if (status === 'SUBSCRIBED') {
        // Track our own presence
        await ch.track({
          userId: currentUser.userId,
          name: currentUser.name,
          avatar: currentUser.avatar,
          onlineAt: new Date().toISOString(),
        });
      }
    });

    setChannel(ch);

    return () => {
      ch.untrack();
      supabase.removeChannel(ch);
    };
  }, [channelName, currentUser.userId]);

  return { onlineUsers, channel };
}

// Usage
function ProjectRoom({ projectId }: { projectId: string }) {
  const { user } = useAuth();
  const { onlineUsers } = useSupabasePresence(`project:${projectId}`, {
    userId: user.id,
    name: user.name,
    avatar: user.avatarUrl,
    onlineAt: new Date().toISOString(),
  });

  return (
    <div>
      <h3>Online now ({onlineUsers.length})</h3>
      {onlineUsers.map((u) => (
        <div key={u.userId} className="flex items-center gap-2">
          <img src={u.avatar} className="w-6 h-6 rounded-full" alt={u.name} />
          <span>{u.name}</span>
          <span className="text-xs text-gray-400">
            since {new Date(u.onlineAt).toLocaleTimeString()}
          </span>
        </div>
      ))}
    </div>
  );
}
```

### Row-Level Security with Realtime

This is what makes Supabase Realtime production-ready: your existing RLS policies automatically apply to real-time subscriptions. A user only receives changes they're authorized to see.

```sql
-- Example: RLS policy for tasks
-- Users can only see tasks in projects they're members of

CREATE POLICY "Users can view tasks in their projects"
  ON tasks FOR SELECT
  USING (
    project_id IN (
      SELECT project_id FROM project_members
      WHERE user_id = auth.uid()
    )
  );

-- This SAME policy applies to real-time subscriptions.
-- If User A subscribes to tasks changes, they'll only receive
-- changes for tasks in projects they're a member of.
-- No additional filtering code needed on the client.
```

```typescript
// The filter in the subscription ADDS to RLS, it doesn't replace it
// Even without a filter, RLS would prevent unauthorized data from leaking
const channel = supabase
  .channel('my-tasks')
  .on(
    'postgres_changes',
    {
      event: '*',
      schema: 'public',
      table: 'tasks',
      // This filter is for PERFORMANCE (reduces messages sent over the wire)
      // RLS handles SECURITY (prevents unauthorized access even without this filter)
      filter: `project_id=eq.${projectId}`,
    },
    handleChange
  )
  .subscribe();
```

---

## 9. SCALING WEBSOCKET CONNECTIONS

WebSocket scaling is fundamentally different from HTTP scaling. With HTTP, each request is independent -- you can load-balance across servers with no coordination. With WebSockets, connections are persistent and stateful. A message sent to "room-123" needs to reach every client in that room, even if they're connected to different servers.

### The Core Scaling Challenge

```
THE PROBLEM:

  Server A holds connections for:       Server B holds connections for:
    - Alice (room-123)                    - Bob (room-123)
    - Charlie (room-456)                  - Diana (room-123)

  Alice sends a message to room-123.
  Server A has Alice's connection, so it can send to Alice.
  But Bob and Diana are on Server B.
  Server A doesn't know about them.

THE SOLUTION: A shared message bus

  Server A  ←───→  Redis Pub/Sub  ←───→  Server B
                   (or similar)

  1. Alice sends message to Server A
  2. Server A publishes to Redis channel "room-123"
  3. Server B subscribes to "room-123", receives the message
  4. Server B sends to Bob and Diana
```

### Socket.io with Redis Adapter

Socket.io has a first-party Redis adapter that handles this automatically:

```typescript
// server/scaled-socket-server.ts
import { createServer } from 'http';
import { Server } from 'socket.io';
import { createAdapter } from '@socket.io/redis-adapter';
import { createClient } from 'redis';

const httpServer = createServer();
const io = new Server(httpServer, {
  cors: { origin: process.env.CLIENT_URL },
  // Important for scaling:
  transports: ['websocket'], // Disable polling — it doesn't work well with load balancers
                              // unless you configure sticky sessions
});

// Create Redis clients (pub/sub requires separate connections)
const pubClient = createClient({ url: process.env.REDIS_URL });
const subClient = pubClient.duplicate();

async function start() {
  await Promise.all([pubClient.connect(), subClient.connect()]);

  // Attach the Redis adapter
  io.adapter(createAdapter(pubClient, subClient));

  io.on('connection', (socket) => {
    // Everything works exactly the same as single-server Socket.io.
    // Rooms, broadcasts, etc. all transparently work across servers.

    socket.on('message', (data) => {
      // This broadcast reaches ALL clients in the room,
      // across ALL servers connected to the same Redis
      io.to(data.roomId).emit('message', data);
    });
  });

  httpServer.listen(3001);
  console.log('Socket.io server with Redis adapter running on port 3001');
}

start();
```

### Connection Limits

```
BROWSER LIMITS:
  - HTTP/1.1: ~6 connections per domain
  - WebSocket: separate from HTTP limit, but browsers
    still limit to ~250-300 per domain
  - In practice: 1 WebSocket connection per app is typical

SINGLE SERVER LIMITS:
  - File descriptors: ulimit -n (default: 1024, set to 65535+)
  - Memory: ~10-50KB per WebSocket connection
  - CPU: message serialization/deserialization
  - Practical limit: 10,000 - 100,000 connections per server
    (depending on message frequency)

SCALING MATH:
  100,000 concurrent users
  ÷ 50,000 connections per server
  = 2 WebSocket servers minimum
  + 1 Redis instance for pub/sub
  + load balancer with sticky sessions (or WebSocket-aware routing)

  At 1 million concurrent users:
  = 20+ WebSocket servers
  = Redis Cluster or dedicated message broker
  = Significant operational complexity
```

### Managed Services vs Self-Hosted

```
┌─────────────────────────┬───────────────────────┬──────────────────────┐
│                         │ Self-Hosted            │ Managed (Ably,       │
│                         │ (Socket.io + Redis)    │ Pusher, Supabase)    │
├─────────────────────────┼───────────────────────┼──────────────────────┤
│ 1K concurrent users     │ 1 server ($20/mo)     │ Free tier            │
│ 10K concurrent          │ 1-2 servers ($50/mo)  │ $50-100/mo           │
│ 100K concurrent         │ 3-5 servers + Redis   │ $500-2,000/mo        │
│                         │ ($300-500/mo)         │                      │
│ 1M concurrent           │ 20+ servers + Redis   │ $5,000-20,000/mo     │
│                         │ Cluster + DevOps team │                      │
│                         │ ($2,000-5,000/mo)     │                      │
├─────────────────────────┼───────────────────────┼──────────────────────┤
│ Time to implement       │ Days to weeks         │ Hours                │
│ Operational burden      │ High (you manage it)  │ Zero                 │
│ Delivery guarantees     │ DIY                   │ Built-in (Ably)      │
│ Global edge             │ DIY                   │ Built-in             │
│ Customization           │ Unlimited             │ Limited to their API │
└─────────────────────────┴───────────────────────┴──────────────────────┘

RECOMMENDATION BY TEAM SIZE:
  - Solo / small startup  → Managed (Supabase Realtime or Pusher)
  - Mid-size team         → Managed (Ably) or self-hosted if you have DevOps
  - Large team / big tech → Self-hosted for cost control and customization
```

### Load Balancer Configuration

If you're self-hosting, your load balancer needs WebSocket support:

```nginx
# nginx.conf for WebSocket load balancing
upstream websocket_servers {
    # ip_hash ensures the same client always reaches the same server
    # This is crucial for Socket.io's HTTP long-polling fallback
    ip_hash;

    server ws-server-1:3001;
    server ws-server-2:3001;
    server ws-server-3:3001;
}

server {
    listen 443 ssl;
    server_name ws.example.com;

    ssl_certificate /etc/ssl/certs/example.com.pem;
    ssl_certificate_key /etc/ssl/private/example.com.key;

    location / {
        proxy_pass http://websocket_servers;

        # These headers are REQUIRED for WebSocket upgrade
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        # Forward client IP
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

        # Timeouts — WebSocket connections are long-lived
        proxy_read_timeout 86400s;   # 24 hours
        proxy_send_timeout 86400s;
    }
}
```

### Monitoring WebSocket Infrastructure

You can't scale what you can't measure:

```typescript
// server/ws-metrics.ts
import { Server } from 'socket.io';

interface WSMetrics {
  totalConnections: number;
  connectionsPerRoom: Map<string, number>;
  messagesPerSecond: number;
  averageLatency: number;
  memoryUsage: number;
}

export function setupMetrics(io: Server) {
  let messageCount = 0;
  let lastMessageCountReset = Date.now();

  // Track message throughput
  io.engine.on('message', () => {
    messageCount++;
  });

  // Collect metrics every 10 seconds
  setInterval(() => {
    const now = Date.now();
    const elapsed = (now - lastMessageCountReset) / 1000;

    const metrics: WSMetrics = {
      totalConnections: io.engine.clientsCount,
      connectionsPerRoom: new Map(),
      messagesPerSecond: messageCount / elapsed,
      averageLatency: 0, // Calculated from ping/pong
      memoryUsage: process.memoryUsage().heapUsed,
    };

    // Count connections per room
    const rooms = io.sockets.adapter.rooms;
    rooms.forEach((sockets, roomId) => {
      // Skip socket IDs (Socket.io creates a "room" for each socket ID)
      if (!sockets.has(roomId)) {
        metrics.connectionsPerRoom.set(roomId, sockets.size);
      }
    });

    // Emit to monitoring dashboard or send to your metrics service
    console.log('WS Metrics:', {
      connections: metrics.totalConnections,
      mps: metrics.messagesPerSecond.toFixed(1),
      rooms: metrics.connectionsPerRoom.size,
      memory: `${(metrics.memoryUsage / 1024 / 1024).toFixed(1)}MB`,
    });

    // Reset counter
    messageCount = 0;
    lastMessageCountReset = now;
  }, 10000);
}
```

---

## 10. COMPLETE EXAMPLES

Let's bring everything together with four production-ready examples.

### Example 1: Real-Time Chat with Socket.io

```typescript
// --- SERVER ---
// server/chat-server.ts

import { Server, Socket } from 'socket.io';

interface ChatMessage {
  id: string;
  roomId: string;
  userId: string;
  userName: string;
  content: string;
  timestamp: number;
  type: 'text' | 'system';
}

interface ChatRoom {
  id: string;
  name: string;
  members: Set<string>;
  messageHistory: ChatMessage[]; // Last N messages in memory
}

const rooms = new Map<string, ChatRoom>();
const MAX_HISTORY = 100;

export function setupChat(io: Server) {
  const chatNs = io.of('/chat');

  chatNs.on('connection', (socket: Socket) => {
    const user = socket.data.user;

    // Join a chat room
    socket.on('chat:join', async (roomId: string) => {
      socket.join(roomId);

      // Initialize room if needed
      if (!rooms.has(roomId)) {
        rooms.set(roomId, {
          id: roomId,
          name: roomId,
          members: new Set(),
          messageHistory: [],
        });
      }

      const room = rooms.get(roomId)!;
      room.members.add(user.id);

      // Send message history to the joining user
      socket.emit('chat:history', room.messageHistory);

      // Announce join
      const systemMessage: ChatMessage = {
        id: crypto.randomUUID(),
        roomId,
        userId: 'system',
        userName: 'System',
        content: `${user.name} joined the chat`,
        timestamp: Date.now(),
        type: 'system',
      };
      chatNs.to(roomId).emit('chat:message', systemMessage);
      room.messageHistory.push(systemMessage);

      // Send updated member count
      chatNs.to(roomId).emit('chat:members', {
        count: room.members.size,
        members: Array.from(room.members),
      });
    });

    // Send a message
    socket.on(
      'chat:send',
      (data: { roomId: string; content: string }, ack?: (response: { id: string }) => void) => {
        const message: ChatMessage = {
          id: crypto.randomUUID(),
          roomId: data.roomId,
          userId: user.id,
          userName: user.name,
          content: data.content,
          timestamp: Date.now(),
          type: 'text',
        };

        // Broadcast to room
        chatNs.to(data.roomId).emit('chat:message', message);

        // Store in history
        const room = rooms.get(data.roomId);
        if (room) {
          room.messageHistory.push(message);
          if (room.messageHistory.length > MAX_HISTORY) {
            room.messageHistory.shift();
          }
        }

        // Acknowledge receipt (for optimistic UI)
        if (ack) ack({ id: message.id });

        // Persist to database
        saveMessageToDB(message).catch(console.error);
      }
    );

    // Typing indicator
    socket.on('chat:typing', (roomId: string) => {
      socket.to(roomId).emit('chat:user-typing', {
        userId: user.id,
        userName: user.name,
      });
    });

    socket.on('chat:stop-typing', (roomId: string) => {
      socket.to(roomId).emit('chat:user-stop-typing', {
        userId: user.id,
      });
    });

    // Leave room
    socket.on('chat:leave', (roomId: string) => {
      socket.leave(roomId);
      const room = rooms.get(roomId);
      if (room) {
        room.members.delete(user.id);
      }

      chatNs.to(roomId).emit('chat:message', {
        id: crypto.randomUUID(),
        roomId,
        userId: 'system',
        userName: 'System',
        content: `${user.name} left the chat`,
        timestamp: Date.now(),
        type: 'system',
      });
    });

    // Disconnect
    socket.on('disconnect', () => {
      for (const [roomId, room] of rooms) {
        if (room.members.has(user.id)) {
          room.members.delete(user.id);
          chatNs.to(roomId).emit('chat:message', {
            id: crypto.randomUUID(),
            roomId,
            userId: 'system',
            userName: 'System',
            content: `${user.name} disconnected`,
            timestamp: Date.now(),
            type: 'system',
          });
        }
      }
    });
  });
}
```

```typescript
// --- CLIENT ---
// components/ChatRoom.tsx

import { useState, useEffect, useRef, useCallback, useMemo } from 'react';
import { io, Socket } from 'socket.io-client';

interface ChatMessage {
  id: string;
  roomId: string;
  userId: string;
  userName: string;
  content: string;
  timestamp: number;
  type: 'text' | 'system';
}

export function ChatRoom({
  roomId,
  userId,
  userName,
  token,
}: {
  roomId: string;
  userId: string;
  userName: string;
  token: string;
}) {
  const [messages, setMessages] = useState<ChatMessage[]>([]);
  const [input, setInput] = useState('');
  const [typingUsers, setTypingUsers] = useState<Map<string, string>>(new Map());
  const [connected, setConnected] = useState(false);
  const socketRef = useRef<Socket | null>(null);
  const messagesEndRef = useRef<HTMLDivElement>(null);
  const typingTimeoutRef = useRef<ReturnType<typeof setTimeout> | null>(null);

  // Connect to chat namespace
  useEffect(() => {
    const socket = io(`${process.env.NEXT_PUBLIC_WS_URL}/chat`, {
      auth: { token },
      transports: ['websocket'],
    });

    socketRef.current = socket;

    socket.on('connect', () => {
      setConnected(true);
      socket.emit('chat:join', roomId);
    });

    socket.on('disconnect', () => setConnected(false));

    socket.on('chat:history', (history: ChatMessage[]) => {
      setMessages(history);
    });

    socket.on('chat:message', (message: ChatMessage) => {
      setMessages((prev) => [...prev, message]);
    });

    socket.on('chat:user-typing', ({ userId: uid, userName: name }: { userId: string; userName: string }) => {
      setTypingUsers((prev) => new Map(prev).set(uid, name));
    });

    socket.on('chat:user-stop-typing', ({ userId: uid }: { userId: string }) => {
      setTypingUsers((prev) => {
        const next = new Map(prev);
        next.delete(uid);
        return next;
      });
    });

    return () => {
      socket.emit('chat:leave', roomId);
      socket.disconnect();
    };
  }, [roomId, token]);

  // Auto-scroll to bottom on new messages
  useEffect(() => {
    messagesEndRef.current?.scrollIntoView({ behavior: 'smooth' });
  }, [messages]);

  // Send message
  const sendMessage = useCallback(() => {
    if (!input.trim() || !socketRef.current) return;

    // Optimistic UI: add message immediately
    const optimisticMessage: ChatMessage = {
      id: `temp-${Date.now()}`,
      roomId,
      userId,
      userName,
      content: input.trim(),
      timestamp: Date.now(),
      type: 'text',
    };
    setMessages((prev) => [...prev, optimisticMessage]);

    // Send to server
    socketRef.current.emit(
      'chat:send',
      { roomId, content: input.trim() },
      (response: { id: string }) => {
        // Replace optimistic ID with server ID
        setMessages((prev) =>
          prev.map((m) =>
            m.id === optimisticMessage.id ? { ...m, id: response.id } : m
          )
        );
      }
    );

    setInput('');
    socketRef.current.emit('chat:stop-typing', roomId);
  }, [input, roomId, userId, userName]);

  // Typing indicator with debounce
  const handleInputChange = useCallback(
    (value: string) => {
      setInput(value);

      if (!socketRef.current) return;

      socketRef.current.emit('chat:typing', roomId);

      if (typingTimeoutRef.current) {
        clearTimeout(typingTimeoutRef.current);
      }

      typingTimeoutRef.current = setTimeout(() => {
        socketRef.current?.emit('chat:stop-typing', roomId);
      }, 2000);
    },
    [roomId]
  );

  // Format typing indicator text
  const typingText = useMemo(() => {
    const names = Array.from(typingUsers.values()).filter(
      (name) => name !== userName
    );
    if (names.length === 0) return null;
    if (names.length === 1) return `${names[0]} is typing...`;
    if (names.length === 2) return `${names[0]} and ${names[1]} are typing...`;
    return `${names[0]} and ${names.length - 1} others are typing...`;
  }, [typingUsers, userName]);

  return (
    <div className="flex flex-col h-[600px] border rounded-lg">
      {/* Connection status */}
      {!connected && (
        <div className="bg-yellow-100 text-yellow-800 text-sm px-4 py-2 text-center">
          Reconnecting...
        </div>
      )}

      {/* Messages */}
      <div className="flex-1 overflow-y-auto p-4 space-y-3">
        {messages.map((message) => (
          <div
            key={message.id}
            className={`${
              message.type === 'system'
                ? 'text-center text-sm text-gray-400 italic'
                : message.userId === userId
                  ? 'flex justify-end'
                  : 'flex justify-start'
            }`}
          >
            {message.type === 'text' && (
              <div
                className={`max-w-[70%] rounded-lg px-4 py-2 ${
                  message.userId === userId
                    ? 'bg-blue-500 text-white'
                    : 'bg-gray-100 text-gray-900'
                }`}
              >
                {message.userId !== userId && (
                  <div className="text-xs font-medium mb-1 opacity-75">
                    {message.userName}
                  </div>
                )}
                <div>{message.content}</div>
                <div className="text-xs opacity-50 mt-1">
                  {new Date(message.timestamp).toLocaleTimeString()}
                </div>
              </div>
            )}
            {message.type === 'system' && <span>{message.content}</span>}
          </div>
        ))}
        <div ref={messagesEndRef} />
      </div>

      {/* Typing indicator */}
      {typingText && (
        <div className="px-4 py-1 text-sm text-gray-500">{typingText}</div>
      )}

      {/* Input */}
      <div className="border-t p-4 flex gap-2">
        <input
          type="text"
          value={input}
          onChange={(e) => handleInputChange(e.target.value)}
          onKeyDown={(e) => e.key === 'Enter' && sendMessage()}
          placeholder="Type a message..."
          className="flex-1 border rounded-lg px-4 py-2 focus:outline-none focus:ring-2
                     focus:ring-blue-500"
        />
        <button
          onClick={sendMessage}
          disabled={!input.trim() || !connected}
          className="bg-blue-500 text-white px-6 py-2 rounded-lg disabled:opacity-50
                     hover:bg-blue-600 transition-colors"
        >
          Send
        </button>
      </div>
    </div>
  );
}
```

### Example 2: Collaborative Task Board with Supabase Realtime

```typescript
// components/RealtimeTaskBoard.tsx
import { useState, useEffect } from 'react';
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { supabase } from '../lib/supabase';

interface Task {
  id: string;
  title: string;
  description: string;
  status: 'todo' | 'in_progress' | 'review' | 'done';
  assignee_id: string | null;
  project_id: string;
  position: number;
  created_at: string;
  updated_at: string;
}

const COLUMNS: { id: Task['status']; label: string }[] = [
  { id: 'todo', label: 'To Do' },
  { id: 'in_progress', label: 'In Progress' },
  { id: 'review', label: 'Review' },
  { id: 'done', label: 'Done' },
];

export function RealtimeTaskBoard({ projectId }: { projectId: string }) {
  const queryClient = useQueryClient();

  // Fetch tasks via TanStack Query
  const { data: tasks = [], isLoading } = useQuery({
    queryKey: ['tasks', projectId],
    queryFn: async () => {
      const { data, error } = await supabase
        .from('tasks')
        .select('*')
        .eq('project_id', projectId)
        .order('position');

      if (error) throw error;
      return data as Task[];
    },
  });

  // Subscribe to real-time changes
  useEffect(() => {
    const channel = supabase
      .channel(`board:${projectId}`)
      .on<Task>(
        'postgres_changes',
        {
          event: '*',
          schema: 'public',
          table: 'tasks',
          filter: `project_id=eq.${projectId}`,
        },
        (payload) => {
          queryClient.setQueryData<Task[]>(
            ['tasks', projectId],
            (oldTasks = []) => {
              switch (payload.eventType) {
                case 'INSERT':
                  // Avoid duplicates (from optimistic updates)
                  if (oldTasks.some((t) => t.id === payload.new.id)) {
                    return oldTasks;
                  }
                  return [...oldTasks, payload.new];

                case 'UPDATE':
                  return oldTasks.map((t) =>
                    t.id === payload.new.id ? payload.new : t
                  );

                case 'DELETE':
                  return oldTasks.filter((t) => t.id !== payload.old.id);

                default:
                  return oldTasks;
              }
            }
          );
        }
      )
      .subscribe();

    return () => {
      supabase.removeChannel(channel);
    };
  }, [projectId, queryClient]);

  // Mutation: Move task to new column
  const moveTask = useMutation({
    mutationFn: async ({
      taskId,
      newStatus,
    }: {
      taskId: string;
      newStatus: Task['status'];
    }) => {
      const { error } = await supabase
        .from('tasks')
        .update({ status: newStatus, updated_at: new Date().toISOString() })
        .eq('id', taskId);

      if (error) throw error;
    },
    // Optimistic update
    onMutate: async ({ taskId, newStatus }) => {
      await queryClient.cancelQueries({ queryKey: ['tasks', projectId] });

      const previousTasks = queryClient.getQueryData<Task[]>(['tasks', projectId]);

      queryClient.setQueryData<Task[]>(
        ['tasks', projectId],
        (old = []) =>
          old.map((t) =>
            t.id === taskId ? { ...t, status: newStatus } : t
          )
      );

      return { previousTasks };
    },
    onError: (_err, _vars, context) => {
      // Rollback on error
      if (context?.previousTasks) {
        queryClient.setQueryData(
          ['tasks', projectId],
          context.previousTasks
        );
      }
    },
  });

  // Mutation: Create task
  const createTask = useMutation({
    mutationFn: async (title: string) => {
      const { error } = await supabase.from('tasks').insert({
        id: crypto.randomUUID(),
        title,
        description: '',
        status: 'todo' as const,
        project_id: projectId,
        position: tasks.filter((t) => t.status === 'todo').length,
      });

      if (error) throw error;
    },
  });

  if (isLoading) return <BoardSkeleton />;

  return (
    <div className="grid grid-cols-4 gap-4 p-4">
      {COLUMNS.map((column) => (
        <div
          key={column.id}
          className="bg-gray-50 rounded-xl p-4 min-h-[400px]"
          onDragOver={(e) => e.preventDefault()}
          onDrop={(e) => {
            const taskId = e.dataTransfer.getData('taskId');
            if (taskId) {
              moveTask.mutate({ taskId, newStatus: column.id });
            }
          }}
        >
          <div className="flex items-center justify-between mb-4">
            <h3 className="font-semibold text-gray-700">{column.label}</h3>
            <span className="text-sm text-gray-400 bg-gray-200 rounded-full px-2">
              {tasks.filter((t) => t.status === column.id).length}
            </span>
          </div>

          <div className="space-y-2">
            {tasks
              .filter((t) => t.status === column.id)
              .sort((a, b) => a.position - b.position)
              .map((task) => (
                <div
                  key={task.id}
                  draggable
                  onDragStart={(e) => {
                    e.dataTransfer.setData('taskId', task.id);
                  }}
                  className="bg-white rounded-lg p-3 shadow-sm border border-gray-200
                             cursor-grab active:cursor-grabbing hover:shadow-md
                             transition-shadow"
                >
                  <div className="font-medium text-sm">{task.title}</div>
                  {task.description && (
                    <div className="text-xs text-gray-500 mt-1 line-clamp-2">
                      {task.description}
                    </div>
                  )}
                </div>
              ))}
          </div>

          {column.id === 'todo' && (
            <QuickAddTask onAdd={(title) => createTask.mutate(title)} />
          )}
        </div>
      ))}
    </div>
  );
}

function QuickAddTask({ onAdd }: { onAdd: (title: string) => void }) {
  const [title, setTitle] = useState('');

  return (
    <form
      onSubmit={(e) => {
        e.preventDefault();
        if (title.trim()) {
          onAdd(title.trim());
          setTitle('');
        }
      }}
      className="mt-3"
    >
      <input
        type="text"
        value={title}
        onChange={(e) => setTitle(e.target.value)}
        placeholder="Add a task..."
        className="w-full border border-dashed border-gray-300 rounded-lg px-3 py-2
                   text-sm focus:outline-none focus:border-blue-400
                   placeholder:text-gray-400"
      />
    </form>
  );
}

function BoardSkeleton() {
  return (
    <div className="grid grid-cols-4 gap-4 p-4">
      {[1, 2, 3, 4].map((col) => (
        <div key={col} className="bg-gray-50 rounded-xl p-4 min-h-[400px]">
          <div className="h-6 w-24 bg-gray-200 rounded animate-pulse mb-4" />
          {[1, 2, 3].map((card) => (
            <div
              key={card}
              className="h-20 bg-white rounded-lg mb-2 animate-pulse"
            />
          ))}
        </div>
      ))}
    </div>
  );
}
```

### Example 3: Live Activity Feed with Supabase

```typescript
// components/LiveActivityFeed.tsx
import { useState, useEffect, useRef } from 'react';
import { supabase } from '../lib/supabase';

interface ActivityEvent {
  id: string;
  user_id: string;
  user_name: string;
  user_avatar: string;
  action: string;
  target_type: string;
  target_name: string;
  metadata: Record<string, unknown>;
  created_at: string;
}

export function LiveActivityFeed({ projectId }: { projectId: string }) {
  const [activities, setActivities] = useState<ActivityEvent[]>([]);
  const [newCount, setNewCount] = useState(0);
  const isScrolledToBottom = useRef(true);
  const containerRef = useRef<HTMLDivElement>(null);

  // Initial fetch
  useEffect(() => {
    const fetchActivities = async () => {
      const { data, error } = await supabase
        .from('activity_events')
        .select('*')
        .eq('project_id', projectId)
        .order('created_at', { ascending: false })
        .limit(50);

      if (!error && data) {
        setActivities(data.reverse()); // Show oldest first
      }
    };

    fetchActivities();
  }, [projectId]);

  // Real-time subscription
  useEffect(() => {
    const channel = supabase
      .channel(`activity:${projectId}`)
      .on<ActivityEvent>(
        'postgres_changes',
        {
          event: 'INSERT',
          schema: 'public',
          table: 'activity_events',
          filter: `project_id=eq.${projectId}`,
        },
        (payload) => {
          setActivities((prev) => [...prev.slice(-99), payload.new]);

          if (!isScrolledToBottom.current) {
            setNewCount((prev) => prev + 1);
          }
        }
      )
      .subscribe();

    return () => {
      supabase.removeChannel(channel);
    };
  }, [projectId]);

  // Track scroll position
  const handleScroll = () => {
    const el = containerRef.current;
    if (!el) return;
    const atBottom = el.scrollHeight - el.scrollTop - el.clientHeight < 50;
    isScrolledToBottom.current = atBottom;
    if (atBottom) setNewCount(0);
  };

  // Auto-scroll when at bottom
  useEffect(() => {
    if (isScrolledToBottom.current) {
      containerRef.current?.scrollTo({
        top: containerRef.current.scrollHeight,
        behavior: 'smooth',
      });
    }
  }, [activities]);

  const scrollToBottom = () => {
    containerRef.current?.scrollTo({
      top: containerRef.current.scrollHeight,
      behavior: 'smooth',
    });
    setNewCount(0);
  };

  return (
    <div className="relative border rounded-lg bg-white">
      <div className="px-4 py-3 border-b font-semibold text-sm text-gray-700">
        Activity
        <span className="ml-2 inline-block w-2 h-2 rounded-full bg-green-500 animate-pulse" />
      </div>

      <div
        ref={containerRef}
        onScroll={handleScroll}
        className="h-[400px] overflow-y-auto p-4 space-y-3"
      >
        {activities.map((activity) => (
          <div key={activity.id} className="flex items-start gap-3 animate-fade-in">
            <img
              src={activity.user_avatar}
              alt={activity.user_name}
              className="w-8 h-8 rounded-full mt-0.5"
            />
            <div>
              <div className="text-sm">
                <span className="font-medium">{activity.user_name}</span>{' '}
                <span className="text-gray-600">{activity.action}</span>{' '}
                <span className="font-medium text-blue-600">
                  {activity.target_name}
                </span>
              </div>
              <div className="text-xs text-gray-400 mt-0.5">
                {formatRelativeTime(activity.created_at)}
              </div>
            </div>
          </div>
        ))}
      </div>

      {/* New activity indicator */}
      {newCount > 0 && (
        <button
          onClick={scrollToBottom}
          className="absolute bottom-16 left-1/2 -translate-x-1/2
                     bg-blue-500 text-white text-sm px-4 py-1.5 rounded-full
                     shadow-lg hover:bg-blue-600 transition-colors"
        >
          {newCount} new {newCount === 1 ? 'activity' : 'activities'}
        </button>
      )}
    </div>
  );
}

function formatRelativeTime(dateString: string): string {
  const date = new Date(dateString);
  const now = new Date();
  const diffMs = now.getTime() - date.getTime();
  const diffSeconds = Math.floor(diffMs / 1000);

  if (diffSeconds < 60) return 'just now';
  if (diffSeconds < 3600) return `${Math.floor(diffSeconds / 60)}m ago`;
  if (diffSeconds < 86400) return `${Math.floor(diffSeconds / 3600)}h ago`;
  return date.toLocaleDateString();
}
```

### Example 4: Full Liveblocks Integration with Presence, Cursors, and Collaborative Data

```typescript
// app/board/[boardId]/page.tsx
// Complete Liveblocks integration combining presence, cursors, and storage

import { RoomProvider } from '../../../liveblocks.config';
import { ClientSideSuspense } from '@liveblocks/react';
import { LiveList, LiveObject } from '@liveblocks/client';
import { ActiveUsers } from '../../../components/ActiveUsers';
import { LiveCursorsProvider } from '../../../components/LiveCursors';
import { TypingIndicator } from '../../../components/TypingIndicator';

// The complete page component
export default function BoardPage({ params }: { params: { boardId: string } }) {
  const { boardId } = params;

  return (
    <RoomProvider
      id={`board-${boardId}`}
      initialPresence={{
        cursor: null,
        selectedId: null,
        isTyping: false,
        name: '',
        color: '',
        avatar: '',
      }}
      initialStorage={{
        tasks: new LiveList([]),
        boardConfig: new LiveObject({
          name: 'My Board',
          columns: new LiveList(['To Do', 'In Progress', 'Done']),
        }),
      }}
    >
      <ClientSideSuspense fallback={<BoardSkeleton />}>
        {() => <BoardContent boardId={boardId} />}
      </ClientSideSuspense>
    </RoomProvider>
  );
}

function BoardContent({ boardId }: { boardId: string }) {
  return (
    <LiveCursorsProvider>
      <div className="min-h-screen bg-gray-100">
        {/* Header with active users */}
        <header className="bg-white border-b px-6 py-3 flex items-center justify-between">
          <h1 className="text-xl font-bold">Project Board</h1>
          <div className="flex items-center gap-4">
            <ActiveUsers />
            <ConnectionIndicator />
          </div>
        </header>

        {/* Board */}
        <main className="p-6">
          <CollaborativeBoard />
        </main>

        {/* Typing indicator */}
        <footer className="fixed bottom-0 inset-x-0 px-6 py-2 bg-white border-t">
          <TypingIndicator />
        </footer>
      </div>
    </LiveCursorsProvider>
  );
}

// Collaborative board using Liveblocks Storage
function CollaborativeBoard() {
  const tasks = useStorage((root) => root.tasks);
  const columns = useStorage((root) => root.boardConfig.columns);
  const { selectItem, selectedByOthers } = useSelectionAwareness();

  const addTask = useMutation(({ storage }, columnIndex: number) => {
    storage.get('tasks').push(
      new LiveObject({
        id: crypto.randomUUID(),
        title: 'New Task',
        status: columns![columnIndex],
        assignee: null,
        createdAt: Date.now(),
      })
    );
  }, [columns]);

  const moveTask = useMutation(
    ({ storage }, taskId: string, newColumn: string) => {
      const tasks = storage.get('tasks');
      const index = tasks.findIndex((t) => t.get('id') === taskId);
      if (index !== -1) {
        tasks.get(index)?.set('status', newColumn);
      }
    },
    []
  );

  if (!tasks || !columns) return null;

  return (
    <div className="flex gap-4 overflow-x-auto pb-4">
      {columns.map((column, colIndex) => (
        <div
          key={column}
          className="bg-white rounded-xl shadow-sm w-80 flex-shrink-0"
          onDragOver={(e) => e.preventDefault()}
          onDrop={(e) => {
            const taskId = e.dataTransfer.getData('taskId');
            if (taskId) moveTask(taskId, column);
          }}
        >
          <div className="p-4 border-b font-semibold text-gray-700">
            {column}
            <span className="ml-2 text-gray-400 font-normal">
              {tasks.filter((t) => t.status === column).length}
            </span>
          </div>

          <div className="p-2 space-y-2 min-h-[200px]">
            {tasks
              .filter((t) => t.status === column)
              .map((task) => {
                const otherUser = selectedByOthers.get(task.id);

                return (
                  <div
                    key={task.id}
                    draggable
                    onDragStart={(e) =>
                      e.dataTransfer.setData('taskId', task.id)
                    }
                    onClick={() => selectItem(task.id)}
                    className="bg-gray-50 rounded-lg p-3 cursor-grab
                               active:cursor-grabbing hover:bg-gray-100
                               transition-colors relative"
                    style={{
                      borderLeft: otherUser
                        ? `3px solid ${otherUser.color}`
                        : '3px solid transparent',
                    }}
                  >
                    {otherUser && (
                      <div
                        className="absolute -top-2 -right-2 w-6 h-6 rounded-full
                                   text-white text-xs flex items-center justify-center"
                        style={{ backgroundColor: otherUser.color }}
                        title={`${otherUser.name} is viewing`}
                      >
                        {otherUser.name[0]}
                      </div>
                    )}
                    <EditableTitle taskId={task.id} initialTitle={task.title} />
                  </div>
                );
              })}
          </div>

          <button
            onClick={() => addTask(colIndex)}
            className="w-full p-3 text-sm text-gray-400 hover:text-gray-600
                       hover:bg-gray-50 transition-colors rounded-b-xl"
          >
            + Add task
          </button>
        </div>
      ))}
    </div>
  );
}

// Inline editable title with typing indicators
function EditableTitle({
  taskId,
  initialTitle,
}: {
  taskId: string;
  initialTitle: string;
}) {
  const [isEditing, setIsEditing] = useState(false);
  const [value, setValue] = useState(initialTitle);
  const { onKeyDown } = useTypingIndicator();

  const updateTitle = useMutation(
    ({ storage }, newTitle: string) => {
      const tasks = storage.get('tasks');
      const index = tasks.findIndex((t) => t.get('id') === taskId);
      if (index !== -1) {
        tasks.get(index)?.set('title', newTitle);
      }
    },
    [taskId]
  );

  if (!isEditing) {
    return (
      <div
        onDoubleClick={() => setIsEditing(true)}
        className="text-sm font-medium cursor-text"
      >
        {initialTitle}
      </div>
    );
  }

  return (
    <input
      autoFocus
      value={value}
      onChange={(e) => {
        setValue(e.target.value);
        onKeyDown();
      }}
      onBlur={() => {
        setIsEditing(false);
        if (value.trim() !== initialTitle) {
          updateTitle(value.trim());
        }
      }}
      onKeyDown={(e) => {
        if (e.key === 'Enter') {
          (e.target as HTMLInputElement).blur();
        }
        if (e.key === 'Escape') {
          setValue(initialTitle);
          setIsEditing(false);
        }
      }}
      className="text-sm font-medium w-full bg-white border rounded px-2 py-1
                 focus:outline-none focus:ring-2 focus:ring-blue-500"
    />
  );
}

function ConnectionIndicator() {
  // Liveblocks provides connection status via useStatus() hook
  // Available in @liveblocks/react
  return (
    <div className="flex items-center gap-1.5 text-xs text-gray-500">
      <span className="w-2 h-2 rounded-full bg-green-500" />
      Connected
    </div>
  );
}
```

---

## ARCHITECTURE DECISION RECORDS

### ADR-41-1: Default to CRDTs over OT for collaborative editing

**Decision:** Use Yjs (CRDT) as the default for collaborative editing in new projects.

**Rationale:** CRDTs work offline, don't require a central server for conflict resolution, and have a mature library ecosystem (Yjs). OT is only recommended when integrating with existing OT infrastructure (e.g., Google's collaboration stack).

**Consequences:** Higher memory usage due to tombstones. Need to implement periodic compaction for long-lived documents. But the tradeoff is worth it for the architectural simplicity and offline support.

### ADR-41-2: Use managed WebSocket services for teams without dedicated infrastructure engineers

**Decision:** Default to managed real-time services (Supabase Realtime for Supabase users, Ably for others) unless the team has dedicated infrastructure engineers.

**Rationale:** WebSocket scaling is an operational burden that is poorly understood by most frontend-focused teams. Managed services eliminate this entirely. The cost premium is offset by engineering time saved.

**Consequences:** Vendor lock-in risk. Reduced customization. But for most teams, the reliability and reduced operational burden are the right tradeoff.

### ADR-41-3: Always integrate real-time updates through the existing cache layer

**Decision:** Real-time data should flow through TanStack Query (or whatever cache layer you use), not bypass it.

**Rationale:** Bypassing the cache creates two sources of truth (the cache and the WebSocket state). This leads to stale UI after reconnection, inconsistent state between components, and duplicate data fetching.

**Consequences:** Slightly more complex integration code. But a single source of truth for all server data eliminates an entire category of bugs.

---

## KEY TAKEAWAYS

1. **Start with presence, not collaborative editing.** Online indicators and typing indicators are 10% of the complexity and 80% of the user impact. Ship presence first, then iterate.

2. **WebSocket connections are stateful infrastructure.** You can't scale them the same way you scale HTTP. Either use a managed service or invest in Redis-backed multi-server architecture from the start.

3. **CRDTs have won.** For new collaborative editing features, use Yjs. The ecosystem is mature, the libraries handle the complexity, and you get offline support for free.

4. **Real-time data should flow through your cache.** Whether you use TanStack Query, SWR, or another cache layer, real-time updates should update the cache, not bypass it. One source of truth.

5. **Throttle everything.** Cursor positions, typing indicators, selection changes -- any high-frequency event needs throttling before it hits the WebSocket. 50ms is a good default.

6. **Show connection state.** Users need to know when their changes aren't syncing. A subtle connection indicator is non-negotiable for any real-time feature.

7. **Mobile is different.** React Native apps go to the background, lose connections, and need to reconcile state when they return. Handle AppState changes explicitly.

8. **Measure before you scale.** Most apps don't need to handle 100K concurrent WebSocket connections. Start simple, instrument everything, and scale when the metrics demand it.

---

## FURTHER READING

- **Yjs documentation** — https://docs.yjs.dev — the definitive guide to Yjs shared types, providers, and persistence
- **Liveblocks documentation** — https://liveblocks.io/docs — managed real-time collaboration platform
- **Supabase Realtime guide** — https://supabase.com/docs/guides/realtime — Postgres-native real-time
- **Martin Kleppmann, "Designing Data-Intensive Applications"** — Chapter 5 (Replication) and Chapter 12 (The Future of Data Systems) cover the distributed systems theory behind CRDTs and conflict resolution
- **"A Comprehensive Study of CRDTs"** (Shapiro et al., 2011) — the foundational academic paper on CRDTs
- **Socket.io documentation** — https://socket.io/docs/v4/ — rooms, namespaces, adapters, scaling
- **"Building real-time collaborative editing"** — Figma's engineering blog post on their CRDT implementation
- **Ably architecture whitepaper** — deep dive into managed real-time infrastructure at global scale
