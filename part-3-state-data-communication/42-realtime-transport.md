<!--
  CHAPTER: 42
  TITLE: Real-Time Transport — WebSockets, SSE & Live Data
  PART: III — State, Data & Communication
  PREREQS: Chapters 11, 12
  KEY_TOPICS: WebSocket, Socket.io, SSE, EventSource, real-time feeds, chat, live updates, connection management, reconnection, heartbeat, background handling, battery optimization, TanStack Query integration
  DIFFICULTY: Intermediate → Advanced
  UPDATED: 2026-04-07
-->

# Chapter 42: Real-Time Transport — WebSockets, SSE & Live Data

> **Part III — State, Data & Communication** | Prerequisites: Chapters 11, 12 | Difficulty: Intermediate to Advanced

<details>
<summary><strong>TL;DR</strong></summary>

- Real-time transport is a spectrum: polling for slow-changing data, SSE for server-to-client push, WebSockets for bidirectional communication; pick the lightest transport that meets your requirements
- A production WebSocket client needs exponential backoff with jitter, heartbeat monitoring, message queuing during disconnects, and typed message envelopes -- the connection itself is 10% of the work
- Socket.io adds rooms, namespaces, acknowledgements, and automatic fallback at the cost of a larger dependency and opinionated protocol; use it when you need those features, not as a default
- Server-Sent Events are underrated for mobile: one-way push with automatic reconnection, perfect for notifications, live feeds, and AI streaming responses
- The two patterns for real-time + TanStack Query (invalidation signals vs direct cache updates) have fundamentally different consistency and performance characteristics; most apps should use both, depending on the data type
- Mobile-specific concerns dominate: AppState-aware disconnect/reconnect, catch-up on missed messages via last-seen cursors, and battery-conscious keep-alive intervals

</details>

Chapter 12 introduced the offline-first mindset and showed how WebSockets fit into a sync architecture. This chapter goes deeper on the **transport layer itself** -- the mechanics of maintaining persistent connections on mobile devices, the patterns that make real-time features reliable, and the complete implementations you need for the features your users expect.

This is not about collaborative editing or CRDTs (see Chapter 41). This is about the plumbing: how bytes move between your server and your user's phone in real time, and what happens when that phone goes into a tunnel, switches from Wi-Fi to cellular, gets backgrounded for twenty minutes, or runs low on battery. Every app with a chat screen, a notification feed, a live score, or a stock ticker needs this chapter.

### In This Chapter
- When You Actually Need Real-Time (and when polling is fine)
- WebSocket Protocol Fundamentals
- Building a Production WebSocket Client with TypeScript
- Socket.io in React Native
- Server-Sent Events and the react-native-sse Polyfill
- Real-Time + TanStack Query Integration Patterns
- Complete Chat Implementation
- Live Feed / Activity Stream
- Background Handling and Catch-Up
- Battery and Data Optimization

### Related Chapters
- [Ch 10: Data Fetching & Server Communication] -- API layer, TanStack Query foundations
- [Ch 11: Caching Strategies] -- cache invalidation patterns that real-time builds on
- [Ch 12: Offline-First & Real-Time] -- offline sync, Legend State, basic WebSocket usage
- [Ch 13: Performance Optimization] -- measuring impact of real-time on performance
- [Ch 41: Real-Time Collaboration] -- CRDTs, Yjs, collaborative editing (different problem)

---

## 1. WHEN YOU ACTUALLY NEED REAL-TIME

The first question is not "which real-time technology?" -- it is "do I need real-time at all?" Every persistent connection costs battery, memory, server resources, and complexity. If polling works, polling is the right answer.

### The Decision Matrix

```
┌──────────────────────────────────────────────────────────────────────┐
│  DATA TYPE              LATENCY NEED    DIRECTION     RECOMMENDATION │
│─────────────────────────────────────────────────────────────────────│
│  User profile           Minutes         Pull          Polling / SWR  │
│  Dashboard metrics      10-30 seconds   Pull          Polling        │
│  Notification badge     1-5 seconds     Push          SSE            │
│  Social feed (new post) 5-15 seconds    Push          SSE / Polling  │
│  Chat messages          < 500ms         Bidirectional WebSocket      │
│  Typing indicators      < 200ms         Bidirectional WebSocket      │
│  Live sports scores     < 1 second      Push          SSE / WS       │
│  Stock prices           < 100ms         Push          WebSocket      │
│  Collaborative editing  < 100ms         Bidirectional WebSocket      │
│  AI streaming response  Streaming       Push          SSE            │
│  File upload progress   1-2 seconds     Push          SSE / Polling  │
│  Order status updates   5-30 seconds    Push          Push + Polling │
└──────────────────────────────────────────────────────────────────────┘
```

### When Polling Is Good Enough

Polling gets a bad reputation, but it is simple, stateless, and works through every proxy and firewall on earth. Here is the rule of thumb:

**If your acceptable latency is greater than your polling interval, and your user count is manageable, poll.**

```typescript
// Polling with TanStack Query -- this is often all you need
function useDashboardMetrics() {
  return useQuery({
    queryKey: ['dashboard', 'metrics'],
    queryFn: fetchDashboardMetrics,
    refetchInterval: 15_000, // Every 15 seconds
    refetchIntervalInBackground: false, // Stop when app is backgrounded
  });
}

// Smart polling: increase interval when data hasn't changed
function useAdaptivePolling<T>(
  queryKey: string[],
  queryFn: () => Promise<T>,
  options: {
    baseInterval: number;
    maxInterval: number;
    backoffMultiplier: number;
  }
) {
  const intervalRef = useRef(options.baseInterval);
  const lastDataHashRef = useRef<string>('');

  return useQuery({
    queryKey,
    queryFn: async () => {
      const data = await queryFn();
      const hash = JSON.stringify(data);

      if (hash === lastDataHashRef.current) {
        // Data hasn't changed -- back off
        intervalRef.current = Math.min(
          intervalRef.current * options.backoffMultiplier,
          options.maxInterval
        );
      } else {
        // Data changed -- reset to base interval
        intervalRef.current = options.baseInterval;
        lastDataHashRef.current = hash;
      }

      return data;
    },
    refetchInterval: () => intervalRef.current,
    refetchIntervalInBackground: false,
  });
}

// Usage: starts at 5s, backs off to 60s when data is stable
const { data } = useAdaptivePolling(
  ['orderStatus', orderId],
  () => api.getOrderStatus(orderId),
  { baseInterval: 5_000, maxInterval: 60_000, backoffMultiplier: 1.5 }
);
```

### When Polling Breaks Down

Polling fails when:
1. **Latency matters** -- users notice the delay between the event and the UI update
2. **Scale matters** -- 100,000 clients polling every 5 seconds is 20,000 requests per second, most returning "no change"
3. **Bidirectional communication is needed** -- polling is inherently pull-based
4. **Battery matters** -- each poll wakes the radio, and the cellular radio consumes significant power for 10-15 seconds per wake cycle even for a tiny request

When any of these are true, you need a persistent connection. The question becomes: SSE or WebSocket?

### SSE vs WebSocket: The Real Trade-Off

```
┌────────────────────────────────────────────────────────────────┐
│                                                                │
│  USE SSE WHEN:                                                 │
│  ✓ Data flows one direction (server → client)                  │
│  ✓ You want automatic reconnection built into the protocol     │
│  ✓ You need to work with HTTP/2 multiplexing                   │
│  ✓ Your infrastructure (load balancers, CDN) is HTTP-native    │
│  ✓ You are streaming AI responses (OpenAI, Anthropic, etc.)    │
│                                                                │
│  USE WEBSOCKET WHEN:                                           │
│  ✓ Communication is bidirectional (chat, collaboration)        │
│  ✓ You need lowest possible latency (< 100ms)                 │
│  ✓ You send frequent small messages from client to server      │
│  ✓ You need binary data (audio streams, file chunks)           │
│  ✓ You need fine-grained connection control (ping/pong)        │
│                                                                │
│  USE BOTH WHEN:                                                │
│  ✓ SSE for notifications/feeds + WebSocket for chat            │
│  ✓ SSE for AI streaming + WebSocket for user actions           │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

---

## 2. WEBSOCKET PROTOCOL FUNDAMENTALS

Before writing a client, you need to understand what is happening at the protocol level. This matters because the bugs you will encounter -- stale connections, half-open sockets, proxy timeouts -- are protocol-level problems.

### The Upgrade Handshake

WebSocket starts as an HTTP request and "upgrades" to a persistent TCP connection:

```
CLIENT → SERVER (HTTP):
GET /ws/chat HTTP/1.1
Host: api.yourapp.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
Sec-WebSocket-Protocol: chat
Authorization: Bearer eyJhbG...

SERVER → CLIENT (HTTP):
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
Sec-WebSocket-Protocol: chat

--- TCP connection now upgraded, HTTP is done ---
```

Key things to understand:

1. **Authentication happens during the handshake.** You cannot send custom headers after the upgrade. In React Native, the `WebSocket` constructor accepts `protocols` but not custom headers directly. You have two options: pass the token as a query parameter (`wss://api.yourapp.com/ws?token=xxx`) or send an authentication message as the first WebSocket frame after connecting.

2. **The connection is now raw TCP.** HTTP headers, cookies, and middleware no longer apply. Your load balancer needs to support WebSocket upgrades (most do, but some older ones do not).

3. **Proxies and CDNs can interfere.** Some corporate proxies strip the `Upgrade` header. Some CDNs have timeout limits on idle WebSocket connections (Cloudflare defaults to 100 seconds of inactivity). This is why heartbeats are not optional.

### Frame Types

WebSocket communication happens in frames. You will not deal with these directly in JavaScript, but understanding them explains behavior:

```
┌─────────────────────────────────────────────────┐
│  FRAME TYPE    OPCODE   PURPOSE                  │
│─────────────────────────────────────────────────│
│  Text          0x1      UTF-8 data (JSON)        │
│  Binary        0x2      Binary data (images, etc)│
│  Close         0x8      Graceful shutdown         │
│  Ping          0x9      Keep-alive (server → cl)  │
│  Pong          0xA      Keep-alive response        │
│  Continuation  0x0      Large message fragment     │
└─────────────────────────────────────────────────┘
```

### Close Codes That Matter

When a WebSocket closes, the close code tells you why. Your reconnection logic should behave differently based on the code:

```typescript
// Understanding close codes changes your reconnection strategy
const CLOSE_CODES = {
  1000: 'Normal closure',           // Do NOT reconnect (intentional)
  1001: 'Going away',               // Server shutting down -- reconnect
  1002: 'Protocol error',           // Bug in your code -- do NOT reconnect
  1003: 'Unsupported data',         // Bug in your code -- do NOT reconnect
  1005: 'No close code',            // Browser quirk -- reconnect
  1006: 'Abnormal closure',         // Network issue -- reconnect
  1007: 'Invalid payload data',     // Bug -- do NOT reconnect
  1008: 'Policy violation',         // Auth issue -- re-authenticate first
  1009: 'Message too big',          // Reduce message size -- do NOT reconnect
  1011: 'Internal server error',    // Server bug -- reconnect with backoff
  1012: 'Service restart',          // Server restarting -- reconnect
  1013: 'Try again later',          // Server overloaded -- reconnect with long backoff
  1014: 'Bad gateway',              // Proxy issue -- reconnect
  4000: 'Auth expired',             // Custom: refresh token then reconnect
  4001: 'Rate limited',             // Custom: back off significantly
  4002: 'Session replaced',         // Custom: another device took over
} as const;
```

### Why WebSocket Works Natively in React Native

React Native includes a WebSocket implementation in its core runtime. You do not need a polyfill or a native module. The `WebSocket` global works the same as in browsers:

```typescript
// This works in React Native out of the box
const ws = new WebSocket('wss://api.yourapp.com/ws');
ws.onopen = () => console.log('connected');
ws.onmessage = (event) => console.log(event.data);
ws.onclose = (event) => console.log('closed', event.code);
ws.onerror = (event) => console.log('error', event);
ws.send('hello');
```

The underlying implementation uses OkHttp on Android and NSURLSession on iOS, which means it handles TLS, certificate pinning (if configured), and proxy settings from the OS automatically. It does **not** implement per-message deflate compression by default -- that requires a native module or server-side configuration.

---

## 3. BUILDING A PRODUCTION WEBSOCKET CLIENT

Chapter 12 introduced a `ReconnectingWebSocket` class. This chapter builds a production-grade version with proper TypeScript typing, jitter in the backoff, connection state management, and mobile-specific handling.

### The Message Envelope

Every message over the wire should follow a consistent envelope format. This is the single most important design decision for your WebSocket layer:

```typescript
// types/realtime.ts

/**
 * Every WebSocket message follows this envelope.
 * The `type` field is used for routing to handlers.
 * The `id` field enables deduplication and acknowledgement.
 * The `timestamp` enables ordering and staleness detection.
 */
interface WebSocketEnvelope<T extends string = string, P = unknown> {
  type: T;
  payload: P;
  id: string;
  timestamp: number;
  /** Server includes this for messages that are responses to client messages */
  replyTo?: string;
}

// Define your message types as a union
type ServerMessage =
  | WebSocketEnvelope<'chat:message', ChatMessage>
  | WebSocketEnvelope<'chat:typing', TypingIndicator>
  | WebSocketEnvelope<'chat:read', ReadReceipt>
  | WebSocketEnvelope<'notification:new', Notification>
  | WebSocketEnvelope<'feed:update', FeedItem>
  | WebSocketEnvelope<'presence:update', PresenceUpdate>
  | WebSocketEnvelope<'system:error', SystemError>
  | WebSocketEnvelope<'system:pong', null>
  | WebSocketEnvelope<'system:ack', { messageId: string }>;

type ClientMessage =
  | WebSocketEnvelope<'chat:send', { conversationId: string; content: string; tempId: string }>
  | WebSocketEnvelope<'chat:typing:start', { conversationId: string }>
  | WebSocketEnvelope<'chat:typing:stop', { conversationId: string }>
  | WebSocketEnvelope<'chat:read', { conversationId: string; messageId: string }>
  | WebSocketEnvelope<'presence:update', { status: 'online' | 'away' | 'offline' }>
  | WebSocketEnvelope<'system:ping', null>
  | WebSocketEnvelope<'subscribe', { channel: string }>
  | WebSocketEnvelope<'unsubscribe', { channel: string }>;

// Extract payload type by message type
type PayloadOf<T extends string, M extends WebSocketEnvelope> =
  M extends WebSocketEnvelope<T, infer P> ? P : never;

// Type-safe handler map
type MessageHandlerMap = {
  [K in ServerMessage['type']]?: (
    payload: PayloadOf<K, ServerMessage>,
    envelope: WebSocketEnvelope
  ) => void;
};

// Supporting types
interface ChatMessage {
  id: string;
  conversationId: string;
  senderId: string;
  content: string;
  createdAt: string;
  tempId?: string; // Client-side temp ID for optimistic matching
}

interface TypingIndicator {
  conversationId: string;
  userId: string;
  isTyping: boolean;
}

interface ReadReceipt {
  conversationId: string;
  userId: string;
  lastReadMessageId: string;
}

interface Notification {
  id: string;
  type: 'message' | 'follow' | 'like' | 'system';
  title: string;
  body: string;
  data: Record<string, unknown>;
  createdAt: string;
}

interface FeedItem {
  id: string;
  type: 'post' | 'comment' | 'reaction';
  data: Record<string, unknown>;
  createdAt: string;
}

interface PresenceUpdate {
  userId: string;
  status: 'online' | 'away' | 'offline';
  lastSeen?: string;
}

interface SystemError {
  code: string;
  message: string;
}
```

### The Complete WebSocket Client

```typescript
// lib/realtime-client.ts
import { AppState, AppStateStatus } from 'react-native';
import NetInfo, { NetInfoState } from '@react-native-community/netinfo';
import { nanoid } from 'nanoid/non-secure';

// --- Configuration ---

interface RealtimeClientConfig {
  /** WebSocket server URL (wss://) */
  url: string;
  /** Function that returns the current auth token */
  getToken: () => Promise<string | null>;
  /** Sub-protocols for the WebSocket handshake */
  protocols?: string[];
  /** Minimum reconnect delay in ms (default: 1000) */
  reconnectBaseDelay?: number;
  /** Maximum reconnect delay in ms (default: 30000) */
  reconnectMaxDelay?: number;
  /** Heartbeat interval in ms (default: 25000) */
  heartbeatInterval?: number;
  /** How long to wait for a pong before considering connection dead (default: 10000) */
  heartbeatTimeout?: number;
  /** Maximum number of messages to queue while disconnected (default: 100) */
  maxQueueSize?: number;
  /** Whether to automatically manage connection based on AppState (default: true) */
  autoManageBackground?: boolean;
}

// --- Connection State ---

type ConnectionState =
  | 'disconnected'
  | 'connecting'
  | 'authenticating'
  | 'connected'
  | 'reconnecting'
  | 'suspended'; // backgrounded intentionally

interface ConnectionInfo {
  state: ConnectionState;
  reconnectAttempt: number;
  lastConnected: number | null;
  lastDisconnected: number | null;
  lastMessageId: string | null; // For catch-up on reconnection
}

// --- Event System ---

type ConnectionEventMap = {
  'connection:state': ConnectionState;
  'connection:error': { code: string; message: string };
  'connection:latency': { roundTripMs: number };
};

type EventMap = MessageHandlerMap & {
  [K in keyof ConnectionEventMap]?: (data: ConnectionEventMap[K]) => void;
};

// --- The Client ---

export class RealtimeClient {
  private ws: WebSocket | null = null;
  private config: Required<RealtimeClientConfig>;
  private connectionInfo: ConnectionInfo = {
    state: 'disconnected',
    reconnectAttempt: 0,
    lastConnected: null,
    lastDisconnected: null,
    lastMessageId: null,
  };

  // Handler registry
  private handlers: Map<string, Set<Function>> = new Map();

  // Message queue for offline buffering
  private messageQueue: Array<{ message: string; addedAt: number }> = [];

  // Timers
  private reconnectTimer: ReturnType<typeof setTimeout> | null = null;
  private heartbeatTimer: ReturnType<typeof setInterval> | null = null;
  private pongTimer: ReturnType<typeof setTimeout> | null = null;
  private pingTimestamp: number = 0;

  // Deduplication: track recently seen message IDs
  private seenMessageIds: Set<string> = new Set();
  private seenMessageCleanupTimer: ReturnType<typeof setInterval> | null = null;

  // System subscriptions
  private netInfoUnsubscribe: (() => void) | null = null;
  private appStateSubscription: { remove: () => void } | null = null;

  constructor(config: RealtimeClientConfig) {
    this.config = {
      protocols: [],
      reconnectBaseDelay: 1000,
      reconnectMaxDelay: 30000,
      heartbeatInterval: 25000,
      heartbeatTimeout: 10000,
      maxQueueSize: 100,
      autoManageBackground: true,
      ...config,
    };

    // Clean up seen message IDs every 5 minutes
    this.seenMessageCleanupTimer = setInterval(() => {
      this.seenMessageIds.clear();
    }, 5 * 60 * 1000);
  }

  // --- Public API ---

  async connect(): Promise<void> {
    if (this.connectionInfo.state === 'connected' ||
        this.connectionInfo.state === 'connecting') {
      return;
    }

    this.setupSystemListeners();
    await this.establishConnection();
  }

  disconnect(): void {
    this.setState('disconnected');
    this.cleanup();
    if (this.ws) {
      this.ws.close(1000, 'Client disconnect');
      this.ws = null;
    }
  }

  /**
   * Send a typed message. If disconnected, the message is queued
   * and will be sent when the connection is restored.
   */
  send<T extends ClientMessage['type']>(
    type: T,
    payload: PayloadOf<T, ClientMessage>
  ): string {
    const id = nanoid();
    const envelope: WebSocketEnvelope = {
      type,
      payload,
      id,
      timestamp: Date.now(),
    };

    const serialized = JSON.stringify(envelope);

    if (this.ws?.readyState === WebSocket.OPEN) {
      this.ws.send(serialized);
    } else {
      this.enqueueMessage(serialized);
    }

    return id; // Return message ID for tracking
  }

  /**
   * Register a handler for a specific message type.
   * Returns an unsubscribe function.
   */
  on<T extends ServerMessage['type']>(
    type: T,
    handler: (payload: PayloadOf<T, ServerMessage>, envelope: WebSocketEnvelope) => void
  ): () => void {
    if (!this.handlers.has(type)) {
      this.handlers.set(type, new Set());
    }
    this.handlers.get(type)!.add(handler);

    return () => {
      this.handlers.get(type)?.delete(handler);
      if (this.handlers.get(type)?.size === 0) {
        this.handlers.delete(type);
      }
    };
  }

  /**
   * Register a handler for connection state changes.
   */
  onConnectionState(handler: (state: ConnectionState) => void): () => void {
    if (!this.handlers.has('connection:state')) {
      this.handlers.set('connection:state', new Set());
    }
    this.handlers.get('connection:state')!.add(handler);

    return () => {
      this.handlers.get('connection:state')?.delete(handler);
    };
  }

  /**
   * Get current connection info for UI display.
   */
  getConnectionInfo(): Readonly<ConnectionInfo> {
    return { ...this.connectionInfo };
  }

  /**
   * Get the last message ID for catch-up on reconnection.
   */
  getLastMessageId(): string | null {
    return this.connectionInfo.lastMessageId;
  }

  /**
   * Destroy the client entirely. Call this when unmounting.
   */
  destroy(): void {
    this.disconnect();
    this.handlers.clear();
    this.messageQueue = [];
    this.seenMessageIds.clear();
    if (this.seenMessageCleanupTimer) {
      clearInterval(this.seenMessageCleanupTimer);
    }
    this.teardownSystemListeners();
  }

  // --- Connection Management ---

  private async establishConnection(): Promise<void> {
    this.setState('connecting');

    const token = await this.config.getToken();
    if (!token) {
      this.setState('disconnected');
      this.emit('connection:error', {
        code: 'AUTH_MISSING',
        message: 'No auth token available',
      });
      return;
    }

    // Pass token as query parameter since WebSocket doesn't support custom headers
    const separator = this.config.url.includes('?') ? '&' : '?';
    const url = `${this.config.url}${separator}token=${encodeURIComponent(token)}`;

    // Include last message ID for server-side catch-up
    const lastId = this.connectionInfo.lastMessageId;
    const catchupUrl = lastId ? `${url}&lastMessageId=${encodeURIComponent(lastId)}` : url;

    try {
      this.ws = new WebSocket(catchupUrl, this.config.protocols);

      this.ws.onopen = () => {
        this.setState('connected');
        this.connectionInfo.reconnectAttempt = 0;
        this.connectionInfo.lastConnected = Date.now();
        this.startHeartbeat();
        this.flushMessageQueue();
      };

      this.ws.onmessage = (event: MessageEvent) => {
        this.handleMessage(event.data);
      };

      this.ws.onclose = (event: CloseEvent) => {
        this.stopHeartbeat();
        this.connectionInfo.lastDisconnected = Date.now();

        if (this.connectionInfo.state === 'disconnected' ||
            this.connectionInfo.state === 'suspended') {
          // Intentional close -- do not reconnect
          return;
        }

        // Decide whether to reconnect based on close code
        if (this.shouldReconnect(event.code)) {
          this.scheduleReconnect();
        } else {
          this.setState('disconnected');
          this.emit('connection:error', {
            code: `CLOSE_${event.code}`,
            message: event.reason || 'Connection closed permanently',
          });
        }
      };

      this.ws.onerror = () => {
        // onerror is always followed by onclose, so we handle reconnection there
        this.emit('connection:error', {
          code: 'WS_ERROR',
          message: 'WebSocket error occurred',
        });
      };
    } catch (error) {
      this.setState('reconnecting');
      this.scheduleReconnect();
    }
  }

  private shouldReconnect(closeCode: number): boolean {
    // Do not reconnect for intentional closes or unrecoverable errors
    const noReconnectCodes = new Set([
      1000, // Normal closure
      1002, // Protocol error (bug)
      1003, // Unsupported data (bug)
      1007, // Invalid payload (bug)
      1009, // Message too big
      4002, // Session replaced (another device)
    ]);

    return !noReconnectCodes.has(closeCode);
  }

  private scheduleReconnect(): void {
    if (this.reconnectTimer) {
      clearTimeout(this.reconnectTimer);
    }

    this.setState('reconnecting');
    const attempt = this.connectionInfo.reconnectAttempt;

    // Exponential backoff with full jitter (AWS-recommended algorithm)
    // See: https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/
    const exponentialDelay = Math.min(
      this.config.reconnectMaxDelay,
      this.config.reconnectBaseDelay * Math.pow(2, attempt)
    );
    // Full jitter: random value between 0 and the exponential delay
    const jitteredDelay = Math.random() * exponentialDelay;

    this.reconnectTimer = setTimeout(async () => {
      this.connectionInfo.reconnectAttempt++;
      await this.establishConnection();
    }, jitteredDelay);
  }

  // --- Heartbeat ---

  private startHeartbeat(): void {
    this.stopHeartbeat();

    this.heartbeatTimer = setInterval(() => {
      if (this.ws?.readyState !== WebSocket.OPEN) return;

      this.pingTimestamp = Date.now();
      this.ws.send(JSON.stringify({
        type: 'system:ping',
        payload: null,
        id: nanoid(),
        timestamp: this.pingTimestamp,
      }));

      // Set a timeout -- if no pong arrives, the connection is dead
      this.pongTimer = setTimeout(() => {
        console.warn('[RealtimeClient] Pong timeout -- connection presumed dead');
        // Force close so onclose triggers reconnection
        this.ws?.close(4000, 'Pong timeout');
      }, this.config.heartbeatTimeout);

    }, this.config.heartbeatInterval);
  }

  private stopHeartbeat(): void {
    if (this.heartbeatTimer) {
      clearInterval(this.heartbeatTimer);
      this.heartbeatTimer = null;
    }
    if (this.pongTimer) {
      clearTimeout(this.pongTimer);
      this.pongTimer = null;
    }
  }

  // --- Message Handling ---

  private handleMessage(raw: string): void {
    let envelope: WebSocketEnvelope;

    try {
      envelope = JSON.parse(raw);
    } catch {
      console.warn('[RealtimeClient] Failed to parse message:', raw.slice(0, 100));
      return;
    }

    // Deduplication
    if (envelope.id && this.seenMessageIds.has(envelope.id)) {
      return;
    }
    if (envelope.id) {
      this.seenMessageIds.add(envelope.id);
    }

    // Track last message ID for catch-up
    if (envelope.id && envelope.type !== 'system:pong') {
      this.connectionInfo.lastMessageId = envelope.id;
    }

    // Handle system messages internally
    if (envelope.type === 'system:pong') {
      if (this.pongTimer) {
        clearTimeout(this.pongTimer);
        this.pongTimer = null;
      }
      const latency = Date.now() - this.pingTimestamp;
      this.emit('connection:latency', { roundTripMs: latency });
      return;
    }

    // Route to registered handlers
    const handlers = this.handlers.get(envelope.type);
    if (handlers) {
      handlers.forEach(handler => {
        try {
          handler(envelope.payload, envelope);
        } catch (error) {
          console.error(
            `[RealtimeClient] Handler error for ${envelope.type}:`,
            error
          );
        }
      });
    }
  }

  // --- Message Queue ---

  private enqueueMessage(serialized: string): void {
    if (this.messageQueue.length >= this.config.maxQueueSize) {
      // Drop oldest messages when queue is full
      this.messageQueue.shift();
    }
    this.messageQueue.push({ message: serialized, addedAt: Date.now() });
  }

  private flushMessageQueue(): void {
    if (this.ws?.readyState !== WebSocket.OPEN || this.messageQueue.length === 0) {
      return;
    }

    const now = Date.now();
    const MAX_AGE = 5 * 60 * 1000; // Drop messages older than 5 minutes

    const validMessages = this.messageQueue.filter(
      (item) => now - item.addedAt < MAX_AGE
    );

    for (const item of validMessages) {
      try {
        this.ws.send(item.message);
      } catch {
        // If send fails, stop flushing -- connection may be dead
        break;
      }
    }

    this.messageQueue = [];
  }

  // --- State Management ---

  private setState(state: ConnectionState): void {
    if (this.connectionInfo.state === state) return;
    this.connectionInfo.state = state;
    this.emit('connection:state', state);
  }

  private emit(event: string, data: unknown): void {
    const handlers = this.handlers.get(event);
    if (handlers) {
      handlers.forEach(handler => {
        try {
          handler(data);
        } catch (error) {
          console.error(`[RealtimeClient] Event handler error for ${event}:`, error);
        }
      });
    }
  }

  // --- Background Handling ---

  private setupSystemListeners(): void {
    if (!this.config.autoManageBackground) return;

    // Monitor app state
    this.appStateSubscription = AppState.addEventListener(
      'change',
      this.handleAppStateChange
    );

    // Monitor network state
    this.netInfoUnsubscribe = NetInfo.addEventListener(
      this.handleNetInfoChange
    );
  }

  private teardownSystemListeners(): void {
    this.appStateSubscription?.remove();
    this.appStateSubscription = null;

    this.netInfoUnsubscribe?.();
    this.netInfoUnsubscribe = null;
  }

  private handleAppStateChange = (state: AppStateStatus): void => {
    if (state === 'background' || state === 'inactive') {
      // Suspend: disconnect gracefully, but remember we want to reconnect
      if (this.connectionInfo.state === 'connected' ||
          this.connectionInfo.state === 'reconnecting') {
        this.setState('suspended');
        this.stopHeartbeat();
        if (this.reconnectTimer) {
          clearTimeout(this.reconnectTimer);
          this.reconnectTimer = null;
        }
        if (this.ws) {
          this.ws.close(1000, 'App backgrounded');
          this.ws = null;
        }
      }
    } else if (state === 'active') {
      // Resume: reconnect if we were suspended
      if (this.connectionInfo.state === 'suspended') {
        this.establishConnection();
      }
    }
  };

  private handleNetInfoChange = (state: NetInfoState): void => {
    const isOnline = Boolean(state.isConnected && state.isInternetReachable);

    if (!isOnline) {
      // Going offline -- stop trying to reconnect
      if (this.reconnectTimer) {
        clearTimeout(this.reconnectTimer);
        this.reconnectTimer = null;
      }
    } else if (
      this.connectionInfo.state === 'reconnecting' &&
      !this.reconnectTimer
    ) {
      // Coming back online -- reconnect immediately
      this.connectionInfo.reconnectAttempt = 0;
      this.establishConnection();
    }
  };

  private cleanup(): void {
    this.stopHeartbeat();
    if (this.reconnectTimer) {
      clearTimeout(this.reconnectTimer);
      this.reconnectTimer = null;
    }
  }
}
```

### React Hooks for the Client

```typescript
// hooks/useRealtime.ts
import { useEffect, useRef, useCallback, useSyncExternalStore } from 'react';
import { RealtimeClient, type ConnectionState } from '../lib/realtime-client';

// Singleton instance -- one connection per app
let clientInstance: RealtimeClient | null = null;

export function getRealtimeClient(): RealtimeClient {
  if (!clientInstance) {
    throw new Error(
      'RealtimeClient not initialized. Call initRealtimeClient() first.'
    );
  }
  return clientInstance;
}

export function initRealtimeClient(config: ConstructorParameters<typeof RealtimeClient>[0]): RealtimeClient {
  if (clientInstance) {
    clientInstance.destroy();
  }
  clientInstance = new RealtimeClient(config);
  return clientInstance;
}

/**
 * Subscribe to a specific message type from the WebSocket.
 * The handler is stable across re-renders (uses a ref internally).
 */
export function useRealtimeMessage<T extends ServerMessage['type']>(
  type: T,
  handler: (payload: PayloadOf<T, ServerMessage>, envelope: WebSocketEnvelope) => void
): void {
  const handlerRef = useRef(handler);
  handlerRef.current = handler;

  useEffect(() => {
    const client = getRealtimeClient();
    const unsubscribe = client.on(type, (payload, envelope) => {
      handlerRef.current(payload, envelope);
    });

    return unsubscribe;
  }, [type]);
}

/**
 * Get the current connection state reactively.
 * Uses useSyncExternalStore for tear-safe reads.
 */
export function useConnectionState(): ConnectionState {
  const stateRef = useRef<ConnectionState>('disconnected');

  const subscribe = useCallback((onStoreChange: () => void) => {
    const client = getRealtimeClient();
    const unsub = client.onConnectionState((state) => {
      stateRef.current = state;
      onStoreChange();
    });

    // Initialize with current state
    stateRef.current = client.getConnectionInfo().state;

    return unsub;
  }, []);

  const getSnapshot = useCallback(() => stateRef.current, []);

  return useSyncExternalStore(subscribe, getSnapshot, getSnapshot);
}

/**
 * Send a typed message. Returns the message ID for tracking.
 */
export function useRealtimeSend() {
  return useCallback(
    <T extends ClientMessage['type']>(
      type: T,
      payload: PayloadOf<T, ClientMessage>
    ): string => {
      return getRealtimeClient().send(type, payload);
    },
    []
  );
}

/**
 * Connection state indicator component helper.
 */
export function useConnectionBanner() {
  const state = useConnectionState();

  return {
    isConnected: state === 'connected',
    isReconnecting: state === 'reconnecting',
    isSuspended: state === 'suspended',
    showBanner: state === 'reconnecting' || state === 'disconnected',
    message:
      state === 'reconnecting'
        ? 'Reconnecting...'
        : state === 'disconnected'
          ? 'Connection lost'
          : state === 'suspended'
            ? 'Paused (app in background)'
            : null,
  };
}
```

### Initializing at App Startup

```typescript
// app/_layout.tsx (Expo Router) or App.tsx
import { useEffect } from 'react';
import { initRealtimeClient, getRealtimeClient } from '../hooks/useRealtime';
import { useAuth } from '../hooks/useAuth';

function RealtimeProvider({ children }: { children: React.ReactNode }) {
  const { getAccessToken, isAuthenticated } = useAuth();

  useEffect(() => {
    if (!isAuthenticated) return;

    const client = initRealtimeClient({
      url: 'wss://api.yourapp.com/ws',
      getToken: getAccessToken,
      heartbeatInterval: 25000,
      reconnectBaseDelay: 1000,
      reconnectMaxDelay: 30000,
    });

    client.connect();

    return () => {
      client.destroy();
    };
  }, [isAuthenticated]);

  return <>{children}</>;
}

export default function RootLayout() {
  return (
    <AuthProvider>
      <QueryProvider>
        <RealtimeProvider>
          {/* Your app */}
        </RealtimeProvider>
      </QueryProvider>
    </AuthProvider>
  );
}
```

---

## 4. SOCKET.IO IN REACT NATIVE

Socket.io is to WebSockets what Express is to Node's HTTP module: a higher-level abstraction that adds features you would otherwise build yourself. The question is whether those features are worth the trade-offs.

### What Socket.io Adds Over Raw WebSocket

```
┌────────────────────────────────────────────────────────────────────┐
│  FEATURE                RAW WEBSOCKET        SOCKET.IO             │
│──────────────────────────────────────────────────────────────────│
│  Auto reconnection      Build it yourself     Built-in             │
│  Heartbeat              Build it yourself     Built-in             │
│  Message acknowledgement Build it yourself    Built-in (callbacks)  │
│  Rooms                  Build it yourself     Built-in              │
│  Namespaces             Not applicable        Built-in              │
│  Binary data            Manual encoding       Automatic             │
│  Fallback transport     Not possible          HTTP long-polling     │
│  Middleware             Not applicable        Server-side hooks     │
│  Multiplexing           One connection        Multiple namespaces   │
│  Bundle size            0 KB (native)         ~40 KB (client)       │
│  Protocol               Standard RFC 6455    Custom (on top of WS) │
│  Server requirement     Any WS server        Socket.io server      │
└────────────────────────────────────────────────────────────────────┘
```

### When to Use Socket.io

Use Socket.io when:
- Your backend team already uses it (and migrating is not justified)
- You need rooms for multi-channel subscriptions (e.g., chat with many conversations open)
- You need acknowledgements for guaranteed delivery without building your own
- You need to support extremely old clients that cannot use WebSocket (rare on mobile)

Use raw WebSocket when:
- You want to keep your client small and dependency-free
- Your server is not Node.js (Socket.io's protocol is proprietary)
- You need to connect to third-party WebSocket APIs (they will not speak Socket.io)
- You want full control over the wire protocol

### Setup in React Native

```bash
npx expo install socket.io-client
```

```typescript
// lib/socket.ts
import { io, Socket } from 'socket.io-client';
import { getAccessToken } from '../auth/tokens';

// Type your events
interface ServerToClientEvents {
  'chat:message': (message: ChatMessage) => void;
  'chat:typing': (data: { userId: string; conversationId: string }) => void;
  'notification:new': (notification: Notification) => void;
  'presence:update': (data: { userId: string; status: string }) => void;
  'error': (data: { code: string; message: string }) => void;
}

interface ClientToServerEvents {
  'chat:send': (
    data: { conversationId: string; content: string },
    callback: (response: { success: boolean; messageId?: string; error?: string }) => void
  ) => void;
  'chat:typing:start': (data: { conversationId: string }) => void;
  'chat:typing:stop': (data: { conversationId: string }) => void;
  'chat:read': (data: { conversationId: string; messageId: string }) => void;
  'room:join': (roomId: string) => void;
  'room:leave': (roomId: string) => void;
}

let socket: Socket<ServerToClientEvents, ClientToServerEvents> | null = null;

export async function connectSocket(): Promise<Socket<ServerToClientEvents, ClientToServerEvents>> {
  if (socket?.connected) return socket;

  const token = await getAccessToken();

  socket = io('https://api.yourapp.com', {
    // Authentication
    auth: { token },

    // Transport configuration
    transports: ['websocket'], // Skip polling on mobile -- go straight to WS
    upgrade: false,            // Don't start with polling then upgrade

    // Reconnection
    reconnection: true,
    reconnectionAttempts: Infinity,
    reconnectionDelay: 1000,
    reconnectionDelayMax: 30000,
    randomizationFactor: 0.5, // Built-in jitter

    // Timeouts
    timeout: 20000,

    // Compression (if server supports)
    // perMessageDeflate: true, // Requires native module in RN
  });

  // Handle auth expiry mid-session
  socket.on('error', async (data) => {
    if (data.code === 'AUTH_EXPIRED') {
      const newToken = await getAccessToken(); // This should refresh
      socket!.auth = { token: newToken };
      socket!.connect(); // Reconnect with new token
    }
  });

  return socket;
}

export function getSocket(): Socket<ServerToClientEvents, ClientToServerEvents> {
  if (!socket) {
    throw new Error('Socket not initialized. Call connectSocket() first.');
  }
  return socket;
}

export function disconnectSocket(): void {
  socket?.disconnect();
  socket = null;
}
```

### Using Rooms and Namespaces

Rooms are the killer feature of Socket.io. When a user opens a conversation, they join a room. When they navigate away, they leave it. The server only sends messages for rooms the client has joined.

```typescript
// hooks/useSocketRoom.ts
import { useEffect } from 'react';
import { getSocket } from '../lib/socket';

/**
 * Join a Socket.io room when the component mounts,
 * leave when it unmounts.
 */
export function useSocketRoom(roomId: string | null) {
  useEffect(() => {
    if (!roomId) return;

    const socket = getSocket();
    socket.emit('room:join', roomId);

    return () => {
      socket.emit('room:leave', roomId);
    };
  }, [roomId]);
}

// Usage in a chat screen:
function ChatScreen({ conversationId }: { conversationId: string }) {
  // Join the room for this conversation
  useSocketRoom(`conversation:${conversationId}`);

  // Now you'll receive messages only for this conversation
  useEffect(() => {
    const socket = getSocket();

    const handler = (message: ChatMessage) => {
      // Only messages for this conversation arrive here
      // (server filters by room)
    };

    socket.on('chat:message', handler);
    return () => { socket.off('chat:message', handler); };
  }, [conversationId]);
}
```

### Acknowledgements for Guaranteed Delivery

This is where Socket.io shines over raw WebSocket. Acknowledgements let you confirm that the server received and processed your message:

```typescript
// Sending a message with acknowledgement
function useSendMessage() {
  const [sending, setSending] = useState(false);

  const sendMessage = useCallback(async (
    conversationId: string,
    content: string
  ): Promise<{ success: boolean; messageId?: string }> => {
    setSending(true);

    return new Promise((resolve) => {
      const socket = getSocket();

      // The third argument is the acknowledgement callback
      socket.emit(
        'chat:send',
        { conversationId, content },
        (response) => {
          setSending(false);

          if (response.success) {
            // Server confirmed receipt and returned the real message ID
            resolve({ success: true, messageId: response.messageId });
          } else {
            resolve({ success: false });
          }
        }
      );

      // Timeout if server doesn't acknowledge within 10 seconds
      setTimeout(() => {
        setSending(false);
        resolve({ success: false });
      }, 10000);
    });
  }, []);

  return { sendMessage, sending };
}
```

### Namespaces for Feature Separation

Namespaces let you multiplex multiple logical connections over a single TCP connection:

```typescript
// Multiple namespaces, single TCP connection
const chatSocket = io('https://api.yourapp.com/chat', {
  auth: { token },
  transports: ['websocket'],
});

const notificationsSocket = io('https://api.yourapp.com/notifications', {
  auth: { token },
  transports: ['websocket'],
});

// These share the underlying TCP connection but have
// independent event handlers, rooms, and middleware
```

### Handling Binary Data

Socket.io automatically handles binary serialization, which is useful for sending images or audio:

```typescript
// Send an image via Socket.io
import * as FileSystem from 'expo-file-system';

async function sendImageMessage(conversationId: string, imageUri: string) {
  const base64 = await FileSystem.readAsStringAsync(imageUri, {
    encoding: FileSystem.EncodingType.Base64,
  });

  const buffer = Uint8Array.from(atob(base64), c => c.charCodeAt(0));

  const socket = getSocket();
  socket.emit('chat:send', {
    conversationId,
    content: '', // Text content empty
    attachment: buffer, // Socket.io handles binary serialization
    attachmentType: 'image/jpeg',
  } as any, (response) => {
    console.log('Image sent:', response);
  });
}
```

---

## 5. SERVER-SENT EVENTS (SSE)

SSE is the most underappreciated real-time technology for mobile apps. It is simpler than WebSocket, works over plain HTTP (so every proxy, CDN, and load balancer handles it), includes automatic reconnection at the protocol level, and is perfect for the many use cases where data only flows from server to client.

### How SSE Works

```
CLIENT → SERVER:
GET /events HTTP/1.1
Accept: text/event-stream
Authorization: Bearer eyJhbG...

SERVER → CLIENT:
HTTP/1.1 200 OK
Content-Type: text/event-stream
Cache-Control: no-cache
Connection: keep-alive

data: {"type":"notification","title":"New message"}

event: feedUpdate
data: {"postId":"123","action":"created"}

event: feedUpdate
data: {"postId":"456","action":"updated"}
id: msg-789
retry: 5000

--- Connection stays open, server sends events as they occur ---
```

Key protocol features:
- **`id` field**: The browser/client sends `Last-Event-ID` header on reconnection, so the server can resume from where it left off. This is built into the protocol -- no custom logic needed.
- **`retry` field**: Server tells client how long to wait before reconnecting. The client follows this automatically.
- **`event` field**: Lets you categorize events so the client can listen for specific types.

### SSE in React Native

React Native does not include a native `EventSource` implementation. You need a polyfill:

```bash
npx expo install react-native-sse
```

```typescript
// lib/sse-client.ts
import EventSource from 'react-native-sse';

interface SSEConfig {
  url: string;
  getToken: () => Promise<string | null>;
  onEvent?: (event: string, data: unknown) => void;
  onError?: (error: unknown) => void;
}

interface SSEConnection {
  close: () => void;
  addEventListener: (event: string, handler: (e: MessageEvent) => void) => void;
}

export async function createSSEConnection(config: SSEConfig): Promise<SSEConnection> {
  const token = await config.getToken();

  const es = new EventSource(config.url, {
    headers: {
      Authorization: `Bearer ${token}`,
    },
    // react-native-sse supports custom headers (unlike browser EventSource)
  });

  es.addEventListener('open', () => {
    console.log('[SSE] Connection established');
  });

  es.addEventListener('error', (event: any) => {
    console.warn('[SSE] Error:', event);
    config.onError?.(event);
  });

  // Default message handler (events without an `event:` field)
  es.addEventListener('message', (event: MessageEvent) => {
    try {
      const data = JSON.parse(event.data);
      config.onEvent?.('message', data);
    } catch {
      config.onEvent?.('message', event.data);
    }
  });

  return es;
}
```

### When SSE Beats WebSocket

**1. Notifications**

Notifications are inherently server-to-client. You never need to send data back over the notification stream. SSE is perfect:

```typescript
// hooks/useNotificationStream.ts
import { useEffect, useRef } from 'react';
import { useQueryClient } from '@tanstack/react-query';
import EventSource from 'react-native-sse';
import { useAuth } from './useAuth';

export function useNotificationStream() {
  const queryClient = useQueryClient();
  const { getAccessToken } = useAuth();
  const esRef = useRef<EventSource | null>(null);

  useEffect(() => {
    let isCancelled = false;

    async function connect() {
      const token = await getAccessToken();
      if (isCancelled) return;

      const es = new EventSource(
        'https://api.yourapp.com/events/notifications',
        { headers: { Authorization: `Bearer ${token}` } }
      );

      esRef.current = es;

      es.addEventListener('notification', (event: MessageEvent) => {
        const notification = JSON.parse(event.data);

        // Update the notifications cache directly
        queryClient.setQueryData<Notification[]>(
          ['notifications'],
          (old = []) => [notification, ...old]
        );

        // Also invalidate the unread count
        queryClient.invalidateQueries({ queryKey: ['notifications', 'unreadCount'] });
      });

      es.addEventListener('badge', (event: MessageEvent) => {
        const { count } = JSON.parse(event.data);
        queryClient.setQueryData(['notifications', 'unreadCount'], count);
      });

      es.addEventListener('error', async () => {
        // react-native-sse handles reconnection automatically
        // But we may need to refresh the token
        if (!isCancelled) {
          es.close();
          // Small delay before reconnecting with fresh token
          setTimeout(() => connect(), 2000);
        }
      });
    }

    connect();

    return () => {
      isCancelled = true;
      esRef.current?.close();
    };
  }, []);
}
```

**2. AI Streaming Responses**

This is the use case that has made SSE mainstream. When you call an LLM API and stream the response token by token, SSE is the standard:

```typescript
// hooks/useAIStream.ts
import { useState, useCallback, useRef } from 'react';
import EventSource from 'react-native-sse';
import { useAuth } from './useAuth';

interface StreamOptions {
  prompt: string;
  model?: string;
  onToken?: (token: string) => void;
  onComplete?: (fullResponse: string) => void;
  onError?: (error: string) => void;
}

export function useAIStream() {
  const [isStreaming, setIsStreaming] = useState(false);
  const [streamedText, setStreamedText] = useState('');
  const { getAccessToken } = useAuth();
  const esRef = useRef<EventSource | null>(null);
  const fullTextRef = useRef('');

  const startStream = useCallback(async (options: StreamOptions) => {
    // Close any existing stream
    esRef.current?.close();

    setIsStreaming(true);
    setStreamedText('');
    fullTextRef.current = '';

    const token = await getAccessToken();

    // SSE with POST body -- requires a custom SSE implementation
    // or a GET endpoint with query params
    const params = new URLSearchParams({
      prompt: options.prompt,
      model: options.model ?? 'default',
    });

    const es = new EventSource(
      `https://api.yourapp.com/ai/stream?${params}`,
      { headers: { Authorization: `Bearer ${token}` } }
    );

    esRef.current = es;

    es.addEventListener('token', (event: MessageEvent) => {
      const { text } = JSON.parse(event.data);
      fullTextRef.current += text;
      setStreamedText(fullTextRef.current);
      options.onToken?.(text);
    });

    es.addEventListener('done', (event: MessageEvent) => {
      setIsStreaming(false);
      options.onComplete?.(fullTextRef.current);
      es.close();
    });

    es.addEventListener('error', (event: any) => {
      setIsStreaming(false);

      // Distinguish between SSE reconnection and actual errors
      if (event.data) {
        const { message } = JSON.parse(event.data);
        options.onError?.(message);
      }
      es.close();
    });
  }, []);

  const stopStream = useCallback(() => {
    esRef.current?.close();
    setIsStreaming(false);
  }, []);

  return { startStream, stopStream, isStreaming, streamedText };
}

// Usage:
function AIChat() {
  const { startStream, isStreaming, streamedText } = useAIStream();

  return (
    <View>
      <Text>{streamedText}</Text>
      {isStreaming && <ActivityIndicator />}
      <Button
        title="Generate"
        onPress={() =>
          startStream({
            prompt: 'Explain WebSockets in React Native',
            onComplete: (text) => {
              // Save to history, etc.
            },
          })
        }
      />
    </View>
  );
}
```

**3. Live Feeds and Score Updates**

```typescript
// hooks/useLiveScores.ts
import { useEffect } from 'react';
import { useQueryClient } from '@tanstack/react-query';
import EventSource from 'react-native-sse';

export function useLiveScores(matchIds: string[]) {
  const queryClient = useQueryClient();

  useEffect(() => {
    if (matchIds.length === 0) return;

    const es = new EventSource(
      `https://api.yourapp.com/events/scores?ids=${matchIds.join(',')}`
    );

    es.addEventListener('score', (event: MessageEvent) => {
      const update = JSON.parse(event.data);

      // Update the specific match in the cache
      queryClient.setQueryData(
        ['match', update.matchId],
        (old: Match | undefined) =>
          old ? { ...old, ...update } : undefined
      );

      // Also update the match in any list queries
      queryClient.setQueriesData<Match[]>(
        { queryKey: ['matches'] },
        (old = []) =>
          old.map((match) =>
            match.id === update.matchId ? { ...match, ...update } : match
          )
      );
    });

    es.addEventListener('status', (event: MessageEvent) => {
      const { matchId, status } = JSON.parse(event.data);
      // Match started, halftime, ended, etc.
      queryClient.setQueryData(
        ['match', matchId],
        (old: Match | undefined) =>
          old ? { ...old, status } : undefined
      );
    });

    return () => es.close();
  }, [matchIds.join(',')]);
}
```

---

## 6. REAL-TIME + TANSTACK QUERY INTEGRATION

This is where most teams get confused. You have TanStack Query managing your server state cache, and you have a WebSocket (or SSE) pushing updates. How do these two systems interact?

There are two patterns, and they solve different problems.

### Pattern 1: Invalidation Signals

The WebSocket tells TanStack Query that data has changed. TanStack Query refetches from the REST API.

```
WebSocket: "hey, the order changed"
  → queryClient.invalidateQueries(['order', '123'])
    → TanStack Query refetches GET /api/orders/123
      → UI updates with fresh data
```

```typescript
// hooks/useRealtimeInvalidation.ts
import { useEffect } from 'react';
import { useQueryClient } from '@tanstack/react-query';
import { useRealtimeMessage } from './useRealtime';

/**
 * Pattern 1: WebSocket sends invalidation signals.
 * TanStack Query refetches from the REST API.
 */
export function useRealtimeInvalidation() {
  const queryClient = useQueryClient();

  // Server sends: { type: 'invalidate', payload: { queryKey: ['order', '123'] } }
  useRealtimeMessage('invalidate', (payload: { queryKey: unknown[] }) => {
    queryClient.invalidateQueries({ queryKey: payload.queryKey });
  });

  // Server sends: { type: 'invalidate:many', payload: { queryKeys: [...] } }
  useRealtimeMessage('invalidate:many', (payload: { queryKeys: unknown[][] }) => {
    payload.queryKeys.forEach((key) => {
      queryClient.invalidateQueries({ queryKey: key });
    });
  });
}
```

**When to use Pattern 1:**
- The data is complex and the server is the source of truth
- You already have REST endpoints that return the correct shape
- You want to reuse your existing TanStack Query cache logic
- The data changes infrequently (a few times per minute)
- Consistency matters more than latency (you want the full server-validated response)

**Trade-offs:**
- Extra network request for every update (the refetch)
- Slightly higher latency (WS signal + HTTP round trip)
- Simple to implement and reason about
- Works even if the WebSocket payload format differs from the REST response

### Pattern 2: Direct Cache Updates

The WebSocket pushes the actual data, and you write it directly into the TanStack Query cache.

```
WebSocket: { type: 'chat:message', payload: { id: '456', text: 'hello', ... } }
  → queryClient.setQueryData(['messages', conversationId], ...)
    → UI updates immediately (no refetch)
```

```typescript
// hooks/useRealtimeChatSync.ts
import { useEffect } from 'react';
import { useQueryClient, InfiniteData } from '@tanstack/react-query';
import { useRealtimeMessage } from './useRealtime';

interface MessagePage {
  messages: ChatMessage[];
  nextCursor: string | null;
}

/**
 * Pattern 2: WebSocket pushes data directly into TanStack Query cache.
 * No refetch needed -- the cache is updated in real time.
 */
export function useRealtimeChatSync(conversationId: string) {
  const queryClient = useQueryClient();

  // New message: prepend to the first page of the infinite query
  useRealtimeMessage('chat:message', (message: ChatMessage) => {
    if (message.conversationId !== conversationId) return;

    queryClient.setQueryData<InfiniteData<MessagePage>>(
      ['messages', conversationId],
      (old) => {
        if (!old) return old;

        // Check for duplicate (optimistic update already added it)
        const firstPage = old.pages[0];
        if (firstPage?.messages.some((m) => m.id === message.id || m.tempId === message.tempId)) {
          // Replace the optimistic message with the server-confirmed one
          return {
            ...old,
            pages: old.pages.map((page, index) => {
              if (index !== 0) return page;
              return {
                ...page,
                messages: page.messages.map((m) =>
                  m.tempId === message.tempId ? message : m
                ),
              };
            }),
          };
        }

        // New message from someone else -- prepend to first page
        return {
          ...old,
          pages: [
            {
              ...firstPage,
              messages: [message, ...firstPage.messages],
            },
            ...old.pages.slice(1),
          ],
        };
      }
    );
  });

  // Typing indicators: separate query for transient data
  useRealtimeMessage('chat:typing', (data: TypingIndicator) => {
    if (data.conversationId !== conversationId) return;

    queryClient.setQueryData<Record<string, boolean>>(
      ['typing', conversationId],
      (old = {}) => ({
        ...old,
        [data.userId]: data.isTyping,
      })
    );
  });

  // Read receipts: update the message status
  useRealtimeMessage('chat:read', (receipt: ReadReceipt) => {
    if (receipt.conversationId !== conversationId) return;

    queryClient.setQueryData<InfiniteData<MessagePage>>(
      ['messages', conversationId],
      (old) => {
        if (!old) return old;

        return {
          ...old,
          pages: old.pages.map((page) => ({
            ...page,
            messages: page.messages.map((msg) => {
              // Mark all messages up to lastReadMessageId as read by this user
              if (msg.createdAt <= receipt.lastReadMessageId) {
                return {
                  ...msg,
                  readBy: [...(msg.readBy ?? []), receipt.userId],
                };
              }
              return msg;
            }),
          })),
        };
      }
    );
  });
}
```

**When to use Pattern 2:**
- Low latency is critical (chat, live scores, stock prices)
- The WebSocket payload matches or is a subset of your query data shape
- Updates are frequent (multiple times per second)
- You need optimistic UI matching (replace temp IDs with server IDs)

**Trade-offs:**
- More complex cache update logic (you are now a cache writer, not just a cache reader)
- Must handle deduplication (optimistic updates + real-time updates can collide)
- The WebSocket payload must be trustworthy -- you are skipping server validation on the read path
- Infinite query cache updates are tricky to get right (page boundaries, cursor management)

### The Hybrid Approach (Recommended for Most Apps)

Most production apps use both patterns depending on the data type:

```typescript
// hooks/useRealtimeSync.ts

export function useRealtimeSync() {
  const queryClient = useQueryClient();

  // PATTERN 2: Direct updates for high-frequency, latency-sensitive data
  useRealtimeMessage('chat:message', (message) => {
    queryClient.setQueryData(/* ... direct cache update ... */);
  });

  useRealtimeMessage('presence:update', (update) => {
    queryClient.setQueryData(['presence', update.userId], update);
  });

  // PATTERN 1: Invalidation for complex, low-frequency data
  useRealtimeMessage('order:updated', (data) => {
    queryClient.invalidateQueries({ queryKey: ['order', data.orderId] });
    queryClient.invalidateQueries({ queryKey: ['orders'] }); // List too
  });

  useRealtimeMessage('profile:updated', (data) => {
    queryClient.invalidateQueries({ queryKey: ['profile', data.userId] });
  });

  // PATTERN 1.5: Optimistic invalidation with stale data override
  // Show the WS data immediately, but also refetch for the full picture
  useRealtimeMessage('feed:update', (item) => {
    // Write what we have now (partial data)
    queryClient.setQueryData(['feed'], (old: FeedItem[] = []) => {
      return [item, ...old.slice(0, 49)]; // Cap at 50
    });

    // Also refetch to get the full data (e.g., user avatars, computed fields)
    queryClient.invalidateQueries({ queryKey: ['feed'] });
  });
}
```

### Combining SSE with TanStack Query's Streaming

TanStack Query v5 does not have native SSE support, but you can bridge them cleanly:

```typescript
// hooks/useSSEQuery.ts
import { useQuery, useQueryClient } from '@tanstack/react-query';
import { useEffect, useRef } from 'react';
import EventSource from 'react-native-sse';

/**
 * A query that initially fetches via HTTP, then receives
 * real-time updates via SSE.
 */
export function useSSEQuery<T>(
  queryKey: string[],
  fetchUrl: string,
  sseUrl: string,
  sseEvent: string,
  options?: { transform?: (sseData: unknown, current: T) => T }
) {
  const queryClient = useQueryClient();
  const esRef = useRef<EventSource | null>(null);

  // Initial fetch via normal HTTP
  const query = useQuery<T>({
    queryKey,
    queryFn: async () => {
      const res = await fetch(fetchUrl);
      return res.json();
    },
    staleTime: Infinity, // SSE keeps it fresh
  });

  // SSE for real-time updates
  useEffect(() => {
    const es = new EventSource(sseUrl);
    esRef.current = es;

    es.addEventListener(sseEvent, (event: MessageEvent) => {
      const data = JSON.parse(event.data);

      if (options?.transform) {
        queryClient.setQueryData<T>(queryKey, (current) =>
          current ? options.transform!(data, current) : current
        );
      } else {
        queryClient.setQueryData(queryKey, data);
      }
    });

    return () => {
      es.close();
      esRef.current = null;
    };
  }, [sseUrl, sseEvent]);

  return query;
}

// Usage:
function MatchDetail({ matchId }: { matchId: string }) {
  const { data: match } = useSSEQuery<Match>(
    ['match', matchId],
    `https://api.yourapp.com/matches/${matchId}`,
    `https://api.yourapp.com/events/matches/${matchId}`,
    'update',
    {
      transform: (update, current) => ({ ...current, ...(update as Partial<Match>) }),
    }
  );

  if (!match) return <MatchSkeleton />;
  return <MatchScore match={match} />;
}
```

---

## 7. CHAT IMPLEMENTATION

Chat is the canonical real-time feature, and building it well requires combining nearly every pattern in this chapter. Here is a complete implementation.

### Data Model

```typescript
// types/chat.ts

interface Conversation {
  id: string;
  name: string | null; // null for 1:1, name for groups
  participants: Participant[];
  lastMessage: ChatMessage | null;
  unreadCount: number;
  updatedAt: string;
}

interface Participant {
  userId: string;
  name: string;
  avatarUrl: string | null;
  role: 'admin' | 'member';
  joinedAt: string;
}

interface ChatMessage {
  id: string;
  tempId?: string;
  conversationId: string;
  senderId: string;
  content: string;
  type: 'text' | 'image' | 'system';
  status: 'sending' | 'sent' | 'delivered' | 'read' | 'failed';
  createdAt: string;
  readBy?: string[];
}

interface MessagesPage {
  messages: ChatMessage[];
  nextCursor: string | null;
  hasMore: boolean;
}
```

### API Layer

```typescript
// api/chat.ts

export const chatApi = {
  getConversations: async (): Promise<Conversation[]> => {
    const res = await apiClient.get('/conversations');
    return res.data;
  },

  getMessages: async (
    conversationId: string,
    cursor?: string
  ): Promise<MessagesPage> => {
    const params = cursor ? `?cursor=${cursor}&limit=30` : '?limit=30';
    const res = await apiClient.get(`/conversations/${conversationId}/messages${params}`);
    return res.data;
  },

  sendMessage: async (
    conversationId: string,
    content: string,
    tempId: string
  ): Promise<ChatMessage> => {
    const res = await apiClient.post(
      `/conversations/${conversationId}/messages`,
      { content, tempId }
    );
    return res.data;
  },

  markRead: async (conversationId: string, messageId: string): Promise<void> => {
    await apiClient.post(`/conversations/${conversationId}/read`, { messageId });
  },
};
```

### Chat Hooks

```typescript
// hooks/useMessages.ts
import {
  useInfiniteQuery,
  useMutation,
  useQueryClient,
  InfiniteData,
} from '@tanstack/react-query';
import { useCallback, useRef, useEffect } from 'react';
import { nanoid } from 'nanoid/non-secure';
import { chatApi } from '../api/chat';
import { useRealtimeMessage, useRealtimeSend } from './useRealtime';
import { useAuth } from './useAuth';

export function useMessages(conversationId: string) {
  const queryClient = useQueryClient();
  const { user } = useAuth();
  const queryKey = ['messages', conversationId];

  // --- Paginated message history (newest first, load older on scroll up) ---

  const messagesQuery = useInfiniteQuery({
    queryKey,
    queryFn: ({ pageParam }) => chatApi.getMessages(conversationId, pageParam),
    initialPageParam: undefined as string | undefined,
    getNextPageParam: (lastPage) =>
      lastPage.hasMore ? lastPage.nextCursor : undefined,
    staleTime: Infinity, // WebSocket keeps this fresh
  });

  // Flatten all pages into a single array, newest first
  const messages = messagesQuery.data?.pages.flatMap((p) => p.messages) ?? [];

  // --- Real-time message reception ---

  useRealtimeMessage('chat:message', (message: ChatMessage) => {
    if (message.conversationId !== conversationId) return;

    queryClient.setQueryData<InfiniteData<MessagesPage>>(
      queryKey,
      (old) => {
        if (!old) return old;

        const firstPage = old.pages[0];

        // Dedup: check if this message (or its optimistic version) already exists
        const exists = firstPage?.messages.some(
          (m) => m.id === message.id || (m.tempId && m.tempId === message.tempId)
        );

        if (exists) {
          // Replace optimistic message with server-confirmed version
          return {
            ...old,
            pages: old.pages.map((page, i) => ({
              ...page,
              messages: page.messages.map((m) =>
                m.tempId === message.tempId
                  ? { ...message, status: 'delivered' as const }
                  : m
              ),
            })),
          };
        }

        // New message from another user -- prepend
        return {
          ...old,
          pages: [
            { ...firstPage, messages: [message, ...firstPage.messages] },
            ...old.pages.slice(1),
          ],
        };
      }
    );

    // Update conversation list with new last message
    queryClient.setQueryData<Conversation[]>(['conversations'], (old = []) =>
      old.map((conv) =>
        conv.id === conversationId
          ? { ...conv, lastMessage: message, updatedAt: message.createdAt }
          : conv
      )
    );
  });

  // --- Send message with optimistic update ---

  const sendMutation = useMutation({
    mutationFn: ({
      content,
      tempId,
    }: {
      content: string;
      tempId: string;
    }) => chatApi.sendMessage(conversationId, content, tempId),

    onMutate: async ({ content, tempId }) => {
      // Cancel any outgoing refetches
      await queryClient.cancelQueries({ queryKey });

      const previousMessages =
        queryClient.getQueryData<InfiniteData<MessagesPage>>(queryKey);

      // Optimistic message
      const optimisticMessage: ChatMessage = {
        id: `temp-${tempId}`,
        tempId,
        conversationId,
        senderId: user!.id,
        content,
        type: 'text',
        status: 'sending',
        createdAt: new Date().toISOString(),
      };

      queryClient.setQueryData<InfiniteData<MessagesPage>>(
        queryKey,
        (old) => {
          if (!old) return old;
          const firstPage = old.pages[0];
          return {
            ...old,
            pages: [
              {
                ...firstPage,
                messages: [optimisticMessage, ...firstPage.messages],
              },
              ...old.pages.slice(1),
            ],
          };
        }
      );

      return { previousMessages };
    },

    onError: (_err, { tempId }, context) => {
      // Rollback optimistic update
      if (context?.previousMessages) {
        queryClient.setQueryData(queryKey, context.previousMessages);
      }

      // Or mark the message as failed (better UX -- lets user retry)
      queryClient.setQueryData<InfiniteData<MessagesPage>>(
        queryKey,
        (old) => {
          if (!old) return old;
          return {
            ...old,
            pages: old.pages.map((page) => ({
              ...page,
              messages: page.messages.map((m) =>
                m.tempId === tempId ? { ...m, status: 'failed' as const } : m
              ),
            })),
          };
        }
      );
    },

    onSuccess: (serverMessage, { tempId }) => {
      // Replace temp message with server-confirmed message
      queryClient.setQueryData<InfiniteData<MessagesPage>>(
        queryKey,
        (old) => {
          if (!old) return old;
          return {
            ...old,
            pages: old.pages.map((page) => ({
              ...page,
              messages: page.messages.map((m) =>
                m.tempId === tempId
                  ? { ...serverMessage, status: 'sent' as const }
                  : m
              ),
            })),
          };
        }
      );
    },
  });

  const sendMessage = useCallback(
    (content: string) => {
      const tempId = nanoid();
      sendMutation.mutate({ content, tempId });
    },
    [sendMutation]
  );

  // --- Retry failed message ---

  const retryMessage = useCallback(
    (tempId: string, content: string) => {
      // Remove the failed message
      queryClient.setQueryData<InfiniteData<MessagesPage>>(
        queryKey,
        (old) => {
          if (!old) return old;
          return {
            ...old,
            pages: old.pages.map((page) => ({
              ...page,
              messages: page.messages.filter((m) => m.tempId !== tempId),
            })),
          };
        }
      );

      // Resend
      sendMutation.mutate({ content, tempId: nanoid() });
    },
    [sendMutation]
  );

  return {
    messages,
    sendMessage,
    retryMessage,
    isLoading: messagesQuery.isLoading,
    isFetchingMore: messagesQuery.isFetchingNextPage,
    hasMore: messagesQuery.hasNextPage,
    fetchMore: messagesQuery.fetchNextPage,
  };
}
```

### Typing Indicators

```typescript
// hooks/useTypingIndicator.ts
import { useState, useEffect, useCallback, useRef } from 'react';
import { useRealtimeMessage, useRealtimeSend } from './useRealtime';

const TYPING_TIMEOUT = 3000; // Consider "stopped typing" after 3 seconds

export function useTypingIndicator(conversationId: string) {
  const [typingUsers, setTypingUsers] = useState<Map<string, NodeJS.Timeout>>(new Map());
  const send = useRealtimeSend();
  const isTypingRef = useRef(false);
  const typingTimerRef = useRef<NodeJS.Timeout | null>(null);

  // --- Receive typing indicators from others ---

  useRealtimeMessage('chat:typing', (data: TypingIndicator) => {
    if (data.conversationId !== conversationId) return;

    setTypingUsers((prev) => {
      const next = new Map(prev);

      if (data.isTyping) {
        // Clear existing timeout for this user
        const existingTimer = next.get(data.userId);
        if (existingTimer) clearTimeout(existingTimer);

        // Set a timeout to auto-remove after TYPING_TIMEOUT
        const timer = setTimeout(() => {
          setTypingUsers((p) => {
            const n = new Map(p);
            n.delete(data.userId);
            return n;
          });
        }, TYPING_TIMEOUT);

        next.set(data.userId, timer);
      } else {
        const timer = next.get(data.userId);
        if (timer) clearTimeout(timer);
        next.delete(data.userId);
      }

      return next;
    });
  });

  // --- Send typing indicator (debounced) ---

  const onTextChange = useCallback(() => {
    if (!isTypingRef.current) {
      isTypingRef.current = true;
      send('chat:typing:start', { conversationId });
    }

    // Reset the "stopped typing" timer
    if (typingTimerRef.current) {
      clearTimeout(typingTimerRef.current);
    }

    typingTimerRef.current = setTimeout(() => {
      isTypingRef.current = false;
      send('chat:typing:stop', { conversationId });
    }, TYPING_TIMEOUT);
  }, [conversationId, send]);

  // Cleanup on unmount
  useEffect(() => {
    return () => {
      if (isTypingRef.current) {
        send('chat:typing:stop', { conversationId });
      }
      if (typingTimerRef.current) {
        clearTimeout(typingTimerRef.current);
      }
      // Clear all typing timers
      typingUsers.forEach((timer) => clearTimeout(timer));
    };
  }, []);

  return {
    typingUserIds: Array.from(typingUsers.keys()),
    onTextChange,
  };
}
```

### The Complete Chat Screen

```typescript
// screens/ChatScreen.tsx
import React, { useCallback, useRef, useState, useEffect } from 'react';
import {
  View,
  Text,
  TextInput,
  FlatList,
  TouchableOpacity,
  KeyboardAvoidingView,
  Platform,
  StyleSheet,
  ActivityIndicator,
  Pressable,
} from 'react-native';
import { useSafeAreaInsets } from 'react-native-safe-area-context';
import { useMessages } from '../hooks/useMessages';
import { useTypingIndicator } from '../hooks/useTypingIndicator';
import { useConnectionBanner } from '../hooks/useRealtime';
import { useAuth } from '../hooks/useAuth';
import { formatMessageTime } from '../utils/time';

interface ChatScreenProps {
  conversationId: string;
  conversationName: string;
}

export function ChatScreen({ conversationId, conversationName }: ChatScreenProps) {
  const insets = useSafeAreaInsets();
  const { user } = useAuth();
  const flatListRef = useRef<FlatList>(null);
  const [inputText, setInputText] = useState('');

  const {
    messages,
    sendMessage,
    retryMessage,
    isLoading,
    isFetchingMore,
    hasMore,
    fetchMore,
  } = useMessages(conversationId);

  const { typingUserIds, onTextChange } = useTypingIndicator(conversationId);
  const { showBanner, message: bannerMessage } = useConnectionBanner();

  // --- Send handler ---

  const handleSend = useCallback(() => {
    const text = inputText.trim();
    if (!text) return;

    sendMessage(text);
    setInputText('');

    // Scroll to bottom
    setTimeout(() => {
      flatListRef.current?.scrollToOffset({ offset: 0, animated: true });
    }, 100);
  }, [inputText, sendMessage]);

  // --- Text input change handler (for typing indicators) ---

  const handleTextChange = useCallback(
    (text: string) => {
      setInputText(text);
      if (text.length > 0) {
        onTextChange();
      }
    },
    [onTextChange]
  );

  // --- Message rendering ---

  const renderMessage = useCallback(
    ({ item: message }: { item: ChatMessage }) => {
      const isOwn = message.senderId === user?.id;
      const isFailed = message.status === 'failed';
      const isSending = message.status === 'sending';

      return (
        <View
          style={[
            styles.messageBubble,
            isOwn ? styles.ownMessage : styles.otherMessage,
            isFailed && styles.failedMessage,
          ]}
        >
          {!isOwn && (
            <Text style={styles.senderName}>
              {message.senderName ?? 'Unknown'}
            </Text>
          )}

          <Text
            style={[
              styles.messageText,
              isOwn ? styles.ownMessageText : styles.otherMessageText,
            ]}
          >
            {message.content}
          </Text>

          <View style={styles.messageFooter}>
            <Text style={styles.messageTime}>
              {formatMessageTime(message.createdAt)}
            </Text>

            {isOwn && (
              <Text style={styles.messageStatus}>
                {isSending ? 'Sending...' : isFailed ? 'Failed' : ''}
                {message.status === 'delivered' ? '✓✓' : ''}
                {message.status === 'read' ? '✓✓' : ''}
                {message.status === 'sent' ? '✓' : ''}
              </Text>
            )}
          </View>

          {isFailed && (
            <Pressable
              onPress={() => retryMessage(message.tempId!, message.content)}
              style={styles.retryButton}
            >
              <Text style={styles.retryText}>Tap to retry</Text>
            </Pressable>
          )}
        </View>
      );
    },
    [user?.id, retryMessage]
  );

  // --- Typing indicator ---

  const renderTypingIndicator = () => {
    if (typingUserIds.length === 0) return null;

    return (
      <View style={styles.typingContainer}>
        <View style={styles.typingBubble}>
          <View style={styles.typingDots}>
            <TypingDot delay={0} />
            <TypingDot delay={200} />
            <TypingDot delay={400} />
          </View>
        </View>
      </View>
    );
  };

  // --- Load more (scroll to top = older messages) ---

  const handleLoadMore = useCallback(() => {
    if (hasMore && !isFetchingMore) {
      fetchMore();
    }
  }, [hasMore, isFetchingMore, fetchMore]);

  return (
    <KeyboardAvoidingView
      style={styles.container}
      behavior={Platform.OS === 'ios' ? 'padding' : undefined}
      keyboardVerticalOffset={Platform.OS === 'ios' ? 90 : 0}
    >
      {/* Connection status banner */}
      {showBanner && (
        <View style={styles.connectionBanner}>
          <Text style={styles.connectionBannerText}>{bannerMessage}</Text>
        </View>
      )}

      {/* Messages list (inverted -- newest at bottom) */}
      {isLoading ? (
        <View style={styles.loadingContainer}>
          <ActivityIndicator size="large" />
        </View>
      ) : (
        <FlatList
          ref={flatListRef}
          data={messages}
          renderItem={renderMessage}
          keyExtractor={(item) => item.id}
          inverted // Newest messages at the bottom
          contentContainerStyle={styles.messagesList}
          onEndReached={handleLoadMore}
          onEndReachedThreshold={0.5}
          ListHeaderComponent={renderTypingIndicator} // "Header" is at bottom when inverted
          ListFooterComponent={
            isFetchingMore ? (
              <View style={styles.loadingMore}>
                <ActivityIndicator size="small" />
              </View>
            ) : null
          }
          // Performance optimizations
          removeClippedSubviews={true}
          maxToRenderPerBatch={15}
          windowSize={10}
          initialNumToRender={20}
          getItemLayout={undefined} // Dynamic height messages
        />
      )}

      {/* Input bar */}
      <View style={[styles.inputContainer, { paddingBottom: insets.bottom || 8 }]}>
        <TextInput
          style={styles.textInput}
          value={inputText}
          onChangeText={handleTextChange}
          placeholder="Message..."
          placeholderTextColor="#999"
          multiline
          maxLength={4000}
          returnKeyType="default"
        />

        <TouchableOpacity
          onPress={handleSend}
          disabled={!inputText.trim()}
          style={[
            styles.sendButton,
            !inputText.trim() && styles.sendButtonDisabled,
          ]}
        >
          <Text style={styles.sendButtonText}>Send</Text>
        </TouchableOpacity>
      </View>
    </KeyboardAvoidingView>
  );
}

// --- Typing animation dot ---

function TypingDot({ delay }: { delay: number }) {
  // In a real app, use Reanimated for this
  return <View style={[styles.typingDot, { opacity: 0.4 }]} />;
}

// --- Styles ---

const styles = StyleSheet.create({
  container: {
    flex: 1,
    backgroundColor: '#F5F5F5',
  },
  connectionBanner: {
    backgroundColor: '#EF4444',
    paddingVertical: 6,
    paddingHorizontal: 16,
    alignItems: 'center',
  },
  connectionBannerText: {
    color: 'white',
    fontSize: 13,
    fontWeight: '600',
  },
  loadingContainer: {
    flex: 1,
    justifyContent: 'center',
    alignItems: 'center',
  },
  messagesList: {
    paddingHorizontal: 12,
    paddingVertical: 8,
  },
  messageBubble: {
    maxWidth: '78%',
    paddingHorizontal: 14,
    paddingVertical: 8,
    borderRadius: 18,
    marginVertical: 2,
  },
  ownMessage: {
    alignSelf: 'flex-end',
    backgroundColor: '#007AFF',
    borderBottomRightRadius: 4,
  },
  otherMessage: {
    alignSelf: 'flex-start',
    backgroundColor: '#FFFFFF',
    borderBottomLeftRadius: 4,
  },
  failedMessage: {
    backgroundColor: '#FEE2E2',
  },
  senderName: {
    fontSize: 12,
    fontWeight: '600',
    color: '#007AFF',
    marginBottom: 2,
  },
  messageText: {
    fontSize: 16,
    lineHeight: 21,
  },
  ownMessageText: {
    color: '#FFFFFF',
  },
  otherMessageText: {
    color: '#1A1A1A',
  },
  messageFooter: {
    flexDirection: 'row',
    justifyContent: 'flex-end',
    alignItems: 'center',
    marginTop: 2,
    gap: 4,
  },
  messageTime: {
    fontSize: 11,
    color: 'rgba(255,255,255,0.7)',
  },
  messageStatus: {
    fontSize: 11,
    color: 'rgba(255,255,255,0.7)',
  },
  retryButton: {
    marginTop: 4,
    paddingVertical: 4,
  },
  retryText: {
    fontSize: 12,
    color: '#EF4444',
    fontWeight: '600',
  },
  typingContainer: {
    paddingHorizontal: 4,
    paddingVertical: 4,
  },
  typingBubble: {
    alignSelf: 'flex-start',
    backgroundColor: '#E5E5E5',
    borderRadius: 18,
    paddingHorizontal: 14,
    paddingVertical: 10,
  },
  typingDots: {
    flexDirection: 'row',
    gap: 4,
  },
  typingDot: {
    width: 8,
    height: 8,
    borderRadius: 4,
    backgroundColor: '#999',
  },
  loadingMore: {
    paddingVertical: 16,
    alignItems: 'center',
  },
  inputContainer: {
    flexDirection: 'row',
    alignItems: 'flex-end',
    paddingHorizontal: 12,
    paddingTop: 8,
    backgroundColor: '#FFFFFF',
    borderTopWidth: StyleSheet.hairlineWidth,
    borderTopColor: '#E0E0E0',
  },
  textInput: {
    flex: 1,
    minHeight: 36,
    maxHeight: 120,
    backgroundColor: '#F0F0F0',
    borderRadius: 20,
    paddingHorizontal: 16,
    paddingTop: Platform.OS === 'ios' ? 9 : 8,
    paddingBottom: Platform.OS === 'ios' ? 9 : 8,
    fontSize: 16,
    color: '#1A1A1A',
  },
  sendButton: {
    marginLeft: 8,
    marginBottom: 4,
    paddingHorizontal: 14,
    paddingVertical: 8,
    backgroundColor: '#007AFF',
    borderRadius: 18,
  },
  sendButtonDisabled: {
    backgroundColor: '#B0B0B0',
  },
  sendButtonText: {
    color: '#FFFFFF',
    fontSize: 15,
    fontWeight: '600',
  },
});
```

---

## 8. LIVE FEED / ACTIVITY STREAM

A live feed has different requirements than chat. New items appear at the top, but you do not auto-scroll -- that would be jarring. Instead, you show a "new items" badge and let the user pull to see them.

### The Feed Hook

```typescript
// hooks/useLiveFeed.ts
import { useState, useCallback, useRef, useEffect } from 'react';
import {
  useInfiniteQuery,
  useQueryClient,
  InfiniteData,
} from '@tanstack/react-query';
import { useRealtimeMessage } from './useRealtime';

interface FeedPage {
  items: FeedItem[];
  nextCursor: string | null;
  hasMore: boolean;
}

interface FeedItem {
  id: string;
  type: string;
  content: Record<string, unknown>;
  createdAt: string;
  author: {
    id: string;
    name: string;
    avatarUrl: string;
  };
}

export function useLiveFeed() {
  const queryClient = useQueryClient();
  const queryKey = ['feed'];
  const [newItemCount, setNewItemCount] = useState(0);
  const newItemsRef = useRef<FeedItem[]>([]);
  const isScrolledToTopRef = useRef(true);

  // --- Fetch feed with pagination ---

  const feedQuery = useInfiniteQuery({
    queryKey,
    queryFn: async ({ pageParam }): Promise<FeedPage> => {
      const params = pageParam ? `?cursor=${pageParam}` : '';
      const res = await fetch(`/api/feed${params}`);
      return res.json();
    },
    initialPageParam: undefined as string | undefined,
    getNextPageParam: (lastPage) =>
      lastPage.hasMore ? lastPage.nextCursor : undefined,
    staleTime: 60_000, // SSE/WS keeps it fresh between fetches
  });

  const items = feedQuery.data?.pages.flatMap((p) => p.items) ?? [];

  // --- Real-time new items ---

  useRealtimeMessage('feed:update', (newItem: FeedItem) => {
    if (isScrolledToTopRef.current) {
      // User is at the top -- insert directly
      queryClient.setQueryData<InfiniteData<FeedPage>>(queryKey, (old) => {
        if (!old) return old;
        const firstPage = old.pages[0];

        // Dedup
        if (firstPage.items.some((i) => i.id === newItem.id)) return old;

        return {
          ...old,
          pages: [
            { ...firstPage, items: [newItem, ...firstPage.items] },
            ...old.pages.slice(1),
          ],
        };
      });
    } else {
      // User has scrolled down -- buffer and show badge
      newItemsRef.current = [newItem, ...newItemsRef.current];
      setNewItemCount((c) => c + 1);
    }
  });

  // --- Flush buffered items when user scrolls to top ---

  const flushNewItems = useCallback(() => {
    if (newItemsRef.current.length === 0) return;

    queryClient.setQueryData<InfiniteData<FeedPage>>(queryKey, (old) => {
      if (!old) return old;
      const firstPage = old.pages[0];

      // Dedup
      const existingIds = new Set(firstPage.items.map((i) => i.id));
      const uniqueNew = newItemsRef.current.filter(
        (item) => !existingIds.has(item.id)
      );

      return {
        ...old,
        pages: [
          { ...firstPage, items: [...uniqueNew, ...firstPage.items] },
          ...old.pages.slice(1),
        ],
      };
    });

    newItemsRef.current = [];
    setNewItemCount(0);
  }, [queryClient]);

  // --- Track scroll position ---

  const onScroll = useCallback((event: { nativeEvent: { contentOffset: { y: number } } }) => {
    const y = event.nativeEvent.contentOffset.y;
    const wasAtTop = isScrolledToTopRef.current;
    isScrolledToTopRef.current = y < 50;

    // If user just scrolled back to top, flush buffered items
    if (!wasAtTop && isScrolledToTopRef.current) {
      flushNewItems();
    }
  }, [flushNewItems]);

  return {
    items,
    newItemCount,
    flushNewItems,
    onScroll,
    isLoading: feedQuery.isLoading,
    isFetchingMore: feedQuery.isFetchingNextPage,
    hasMore: feedQuery.hasNextPage,
    fetchMore: feedQuery.fetchNextPage,
    refresh: feedQuery.refetch,
  };
}
```

### The Feed Screen

```typescript
// screens/FeedScreen.tsx
import React, { useRef, useCallback } from 'react';
import {
  View,
  Text,
  FlatList,
  RefreshControl,
  TouchableOpacity,
  StyleSheet,
  Animated,
} from 'react-native';
import { useLiveFeed } from '../hooks/useLiveFeed';

export function FeedScreen() {
  const {
    items,
    newItemCount,
    flushNewItems,
    onScroll,
    isLoading,
    isFetchingMore,
    hasMore,
    fetchMore,
    refresh,
  } = useLiveFeed();

  const flatListRef = useRef<FlatList>(null);
  const [refreshing, setRefreshing] = React.useState(false);

  const handleRefresh = useCallback(async () => {
    setRefreshing(true);
    await refresh();
    setRefreshing(false);
  }, [refresh]);

  const handleNewItemsBadge = useCallback(() => {
    flushNewItems();
    flatListRef.current?.scrollToOffset({ offset: 0, animated: true });
  }, [flushNewItems]);

  return (
    <View style={styles.container}>
      {/* New items badge */}
      {newItemCount > 0 && (
        <TouchableOpacity
          style={styles.newItemsBadge}
          onPress={handleNewItemsBadge}
          activeOpacity={0.8}
        >
          <Text style={styles.newItemsBadgeText}>
            {newItemCount} new {newItemCount === 1 ? 'post' : 'posts'}
          </Text>
        </TouchableOpacity>
      )}

      <FlatList
        ref={flatListRef}
        data={items}
        renderItem={({ item }) => <FeedItemCard item={item} />}
        keyExtractor={(item) => item.id}
        onScroll={onScroll}
        scrollEventThrottle={16}
        refreshControl={
          <RefreshControl refreshing={refreshing} onRefresh={handleRefresh} />
        }
        onEndReached={() => {
          if (hasMore && !isFetchingMore) fetchMore();
        }}
        onEndReachedThreshold={0.5}
      />
    </View>
  );
}

function FeedItemCard({ item }: { item: FeedItem }) {
  return (
    <View style={styles.feedItem}>
      <Text style={styles.feedAuthor}>{item.author.name}</Text>
      <Text style={styles.feedContent}>{String(item.content.text ?? '')}</Text>
      <Text style={styles.feedTime}>
        {new Date(item.createdAt).toLocaleTimeString()}
      </Text>
    </View>
  );
}

const styles = StyleSheet.create({
  container: {
    flex: 1,
    backgroundColor: '#F5F5F5',
  },
  newItemsBadge: {
    position: 'absolute',
    top: 12,
    alignSelf: 'center',
    zIndex: 100,
    backgroundColor: '#007AFF',
    paddingHorizontal: 16,
    paddingVertical: 8,
    borderRadius: 20,
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 2 },
    shadowOpacity: 0.15,
    shadowRadius: 4,
    elevation: 4,
  },
  newItemsBadgeText: {
    color: '#FFFFFF',
    fontSize: 14,
    fontWeight: '600',
  },
  feedItem: {
    backgroundColor: '#FFFFFF',
    marginHorizontal: 12,
    marginVertical: 4,
    padding: 16,
    borderRadius: 12,
  },
  feedAuthor: {
    fontSize: 15,
    fontWeight: '700',
    color: '#1A1A1A',
    marginBottom: 4,
  },
  feedContent: {
    fontSize: 15,
    color: '#333',
    lineHeight: 21,
  },
  feedTime: {
    fontSize: 12,
    color: '#999',
    marginTop: 8,
  },
});
```

---

## 9. BACKGROUND HANDLING AND CATCH-UP

This is the section that separates mobile real-time from web real-time. On the web, the tab stays open and the WebSocket stays connected (mostly). On mobile, the OS will background your app, suspend it, or kill it entirely. When the user returns, you have a gap.

### The Problem

```
Timeline:
  10:00  User opens app, WebSocket connects
  10:05  User gets a phone call → app backgrounds
  10:05  iOS suspends the app (WebSocket is dead)
  10:06  3 new messages arrive on the server
  10:07  A notification is sent (push notification)
  10:15  User returns to app → app foregrounds
  10:15  WebSocket reconnects
  10:15  ??? How do we get the 3 missed messages?
```

### The Catch-Up Protocol

The solution is a **cursor-based catch-up**. Your client tracks the ID (or timestamp) of the last message it received. On reconnection, it sends this to the server, and the server sends everything that happened after that point.

This is already built into our `RealtimeClient` above (the `lastMessageId` query parameter). Here is the server-side contract:

```
CLIENT CONNECTS:
  wss://api.yourapp.com/ws?token=xxx&lastMessageId=msg-456

SERVER RESPONSE (catch-up):
  1. Send all messages with ID > msg-456, in order
  2. Then switch to live mode (push new messages as they arrive)

IF lastMessageId IS TOO OLD (> 24 hours of messages):
  Server sends: { type: "system:catchup_overflow", payload: { reason: "too_many_missed" } }
  Client responds by: refetching from REST API (full refresh)
```

### Implementation: AppState-Aware Connection Manager

```typescript
// hooks/useRealtimeLifecycle.ts
import { useEffect, useRef } from 'react';
import { AppState, AppStateStatus } from 'react-native';
import { useQueryClient } from '@tanstack/react-query';
import { getRealtimeClient } from './useRealtime';
import { useRealtimeMessage } from './useRealtime';

/**
 * Manages the real-time connection lifecycle:
 * - Disconnects on background
 * - Reconnects on foreground
 * - Handles catch-up overflow
 * - Refetches critical queries on resume
 */
export function useRealtimeLifecycle() {
  const queryClient = useQueryClient();
  const backgroundTimestampRef = useRef<number | null>(null);

  // Handle catch-up overflow (too many missed messages)
  useRealtimeMessage('system:catchup_overflow' as any, () => {
    // The server cannot send all missed messages.
    // Refetch all critical queries from REST.
    queryClient.invalidateQueries({ queryKey: ['conversations'] });
    queryClient.invalidateQueries({ queryKey: ['messages'] });
    queryClient.invalidateQueries({ queryKey: ['notifications'] });
    queryClient.invalidateQueries({ queryKey: ['feed'] });
  });

  useEffect(() => {
    const subscription = AppState.addEventListener('change', handleStateChange);
    return () => subscription.remove();
  }, []);

  function handleStateChange(state: AppStateStatus) {
    if (state === 'background' || state === 'inactive') {
      backgroundTimestampRef.current = Date.now();
      // The RealtimeClient handles disconnect automatically
      // (see autoManageBackground in the client)
    }

    if (state === 'active' && backgroundTimestampRef.current) {
      const backgroundDuration = Date.now() - backgroundTimestampRef.current;
      backgroundTimestampRef.current = null;

      // The RealtimeClient reconnects and sends lastMessageId automatically.
      // But if we were backgrounded for a long time, proactively refetch
      // critical data in case the WS catch-up is not sufficient.

      const LONG_BACKGROUND_THRESHOLD = 5 * 60 * 1000; // 5 minutes

      if (backgroundDuration > LONG_BACKGROUND_THRESHOLD) {
        // Invalidate everything -- we've been gone too long to trust the cache
        queryClient.invalidateQueries();
      } else {
        // Short background -- just refetch the most time-sensitive data
        queryClient.invalidateQueries({ queryKey: ['conversations'] });
        queryClient.invalidateQueries({
          queryKey: ['notifications', 'unreadCount'],
        });
      }
    }
  }
}
```

### Bridging Push Notifications and Real-Time

When the app is backgrounded, push notifications handle alerting the user. When the user taps a notification and opens the app, the real-time system needs to work with the push notification payload:

```typescript
// hooks/usePushNotificationBridge.ts
import { useEffect } from 'react';
import * as Notifications from 'expo-notifications';
import { useQueryClient } from '@tanstack/react-query';
import { router } from 'expo-router';

/**
 * When a push notification is tapped, pre-seed the cache
 * with the notification data so the target screen renders instantly
 * (before the REST fetch or WS catch-up completes).
 */
export function usePushNotificationBridge() {
  const queryClient = useQueryClient();

  useEffect(() => {
    const subscription = Notifications.addNotificationResponseReceivedListener(
      (response) => {
        const data = response.notification.request.content.data;

        switch (data.type) {
          case 'new_message': {
            // Pre-seed the message into the cache
            const message = data.message as ChatMessage;

            queryClient.setQueryData<InfiniteData<MessagesPage>>(
              ['messages', message.conversationId],
              (old) => {
                if (!old) return old;
                const firstPage = old.pages[0];
                if (firstPage.messages.some((m) => m.id === message.id)) {
                  return old; // Already have it
                }
                return {
                  ...old,
                  pages: [
                    {
                      ...firstPage,
                      messages: [message, ...firstPage.messages],
                    },
                    ...old.pages.slice(1),
                  ],
                };
              }
            );

            // Navigate to the conversation
            router.push(`/chat/${message.conversationId}`);
            break;
          }

          case 'new_notification': {
            queryClient.invalidateQueries({ queryKey: ['notifications'] });
            router.push('/notifications');
            break;
          }
        }
      }
    );

    return () => subscription.remove();
  }, []);
}
```

---

## 10. BATTERY AND DATA OPTIMIZATION

A WebSocket connection that is always on, pinging every 15 seconds, receiving every event from every channel, will drain your user's battery and data plan. Production apps need to be smarter.

### Keep-Alive Interval Tuning

The heartbeat interval is a trade-off between detecting dead connections quickly and consuming battery:

```typescript
// Adjust heartbeat based on connection type and app state
import NetInfo from '@react-native-community/netinfo';

interface HeartbeatConfig {
  wifi: number;          // Lower -- Wi-Fi is cheap
  cellular: number;      // Higher -- cellular radio wake is expensive
  background: number;    // Highest -- or disable entirely
}

const HEARTBEAT_INTERVALS: HeartbeatConfig = {
  wifi: 20_000,          // 20 seconds on Wi-Fi
  cellular: 45_000,      // 45 seconds on cellular
  background: 0,         // Disabled when backgrounded (disconnect instead)
};

function getOptimalHeartbeatInterval(
  connectionType: string,
  appState: string
): number {
  if (appState !== 'active') return HEARTBEAT_INTERVALS.background;
  if (connectionType === 'wifi') return HEARTBEAT_INTERVALS.wifi;
  return HEARTBEAT_INTERVALS.cellular;
}
```

**Why cellular is more expensive:** When a phone sends data over cellular, the radio transitions from an idle state to an active state. This transition takes power, and the radio stays active for 10-15 seconds after the last transmission (the "tail time"). A ping every 15 seconds means the radio never enters idle, consuming battery continuously. A ping every 45 seconds allows about 30 seconds of idle time between pings.

### Payload Size Reduction

Every byte matters on cellular. Here are practical strategies:

```typescript
// 1. Use short keys in WebSocket messages
// BAD: 247 bytes
{
  "type": "chat:message",
  "payload": {
    "messageId": "abc123",
    "conversationId": "conv456",
    "senderId": "user789",
    "content": "Hello!",
    "createdAt": "2026-04-07T10:30:00.000Z",
    "messageType": "text"
  }
}

// GOOD: 134 bytes (45% smaller)
{
  "t": "cm",
  "p": {
    "i": "abc123",
    "c": "conv456",
    "s": "user789",
    "x": "Hello!",
    "a": 1743937800,
    "y": 0
  }
}

// The client maps short keys back to readable names
const MESSAGE_KEY_MAP = {
  i: 'id',
  c: 'conversationId',
  s: 'senderId',
  x: 'content',
  a: 'createdAt',
  y: 'type',
} as const;

const MESSAGE_TYPE_MAP = {
  0: 'text',
  1: 'image',
  2: 'system',
} as const;

function expandMessage(compact: Record<string, unknown>): ChatMessage {
  return {
    id: compact.i as string,
    conversationId: compact.c as string,
    senderId: compact.s as string,
    content: compact.x as string,
    createdAt: new Date((compact.a as number) * 1000).toISOString(),
    type: MESSAGE_TYPE_MAP[compact.y as keyof typeof MESSAGE_TYPE_MAP],
  } as ChatMessage;
}
```

```typescript
// 2. Use timestamps as epoch seconds, not ISO strings
// "2026-04-07T10:30:00.000Z" = 24 bytes
// 1743937800                  = 10 bytes

// 3. Use numeric enums instead of strings
// "delivered" = 11 bytes
// 3           = 1 byte
```

### Message Batching

When many small messages arrive in rapid succession (e.g., a busy chat room), batching prevents excessive re-renders:

```typescript
// lib/message-batcher.ts

type BatchCallback<T> = (items: T[]) => void;

/**
 * Batches incoming items and flushes them in batches.
 * Reduces re-renders when many messages arrive in quick succession.
 */
export class MessageBatcher<T> {
  private buffer: T[] = [];
  private flushTimer: ReturnType<typeof setTimeout> | null = null;
  private callback: BatchCallback<T>;
  private maxBatchSize: number;
  private flushIntervalMs: number;

  constructor(
    callback: BatchCallback<T>,
    options: { maxBatchSize?: number; flushIntervalMs?: number } = {}
  ) {
    this.callback = callback;
    this.maxBatchSize = options.maxBatchSize ?? 10;
    this.flushIntervalMs = options.flushIntervalMs ?? 100;
  }

  add(item: T): void {
    this.buffer.push(item);

    // Flush immediately if batch is full
    if (this.buffer.length >= this.maxBatchSize) {
      this.flush();
      return;
    }

    // Otherwise, schedule a flush
    if (!this.flushTimer) {
      this.flushTimer = setTimeout(() => this.flush(), this.flushIntervalMs);
    }
  }

  flush(): void {
    if (this.flushTimer) {
      clearTimeout(this.flushTimer);
      this.flushTimer = null;
    }

    if (this.buffer.length > 0) {
      const batch = this.buffer;
      this.buffer = [];
      this.callback(batch);
    }
  }

  destroy(): void {
    if (this.flushTimer) {
      clearTimeout(this.flushTimer);
    }
    this.buffer = [];
  }
}

// Usage with the realtime client:
const messageBatcher = new MessageBatcher<ChatMessage>(
  (messages) => {
    // Single cache update for all messages in the batch
    queryClient.setQueryData<InfiniteData<MessagesPage>>(
      ['messages', conversationId],
      (old) => {
        if (!old) return old;
        const firstPage = old.pages[0];
        const existingIds = new Set(firstPage.messages.map((m) => m.id));
        const newMessages = messages.filter((m) => !existingIds.has(m.id));

        if (newMessages.length === 0) return old;

        return {
          ...old,
          pages: [
            {
              ...firstPage,
              messages: [...newMessages.reverse(), ...firstPage.messages],
            },
            ...old.pages.slice(1),
          ],
        };
      }
    );
  },
  { maxBatchSize: 10, flushIntervalMs: 100 }
);
```

### Selective Subscriptions

Do not subscribe to everything. Only listen for events relevant to the current screen:

```typescript
// hooks/useSelectiveSubscription.ts
import { useEffect } from 'react';
import { getRealtimeClient, useRealtimeSend } from './useRealtime';

/**
 * Subscribe to a channel when the component mounts,
 * unsubscribe when it unmounts.
 *
 * The server only sends events for subscribed channels,
 * reducing bandwidth and processing on both sides.
 */
export function useSubscription(channel: string) {
  const send = useRealtimeSend();

  useEffect(() => {
    send('subscribe', { channel });

    return () => {
      send('unsubscribe', { channel });
    };
  }, [channel, send]);
}

// Usage:
function ConversationScreen({ conversationId }: { conversationId: string }) {
  // Only receive messages for this conversation while on this screen
  useSubscription(`conversation:${conversationId}`);

  // Messages for other conversations will not be sent to this client
  // (the server handles filtering based on subscriptions)
}

function DashboardScreen() {
  // Subscribe to multiple channels
  useSubscription('dashboard:metrics');
  useSubscription('dashboard:alerts');
}
```

### Per-Message Compression

WebSocket supports per-message deflate compression at the protocol level. In React Native, this requires server-side configuration since the built-in WebSocket implementation handles it transparently if the server negotiates it:

```
Server configuration (Node.js with ws):

const wss = new WebSocket.Server({
  perMessageDeflate: {
    zlibDeflateOptions: {
      chunkSize: 1024,
      memLevel: 7,
      level: 3,  // Balance between compression ratio and CPU
    },
    threshold: 128,  // Only compress messages larger than 128 bytes
  },
});

// Typical compression ratios for JSON:
// - Small messages (< 128 bytes): not worth compressing
// - Chat messages (~200 bytes): 30-40% reduction
// - Large payloads (~2KB+): 60-80% reduction
```

---

## SUMMARY

Real-time transport is not a single technology decision -- it is a layered set of patterns that must work together:

```
┌───────────────────────────────────────────────────────────────┐
│                    REAL-TIME ARCHITECTURE                      │
│                                                               │
│  ┌─────────────┐     ┌──────────────┐     ┌──────────────┐  │
│  │   POLLING    │     │     SSE      │     │  WEBSOCKET   │  │
│  │  Dashboard   │     │ Notifications│     │    Chat      │  │
│  │  Order status│     │ AI streaming │     │  Typing      │  │
│  │  Profile     │     │ Live scores  │     │  Presence    │  │
│  └──────┬───────┘     └──────┬───────┘     └──────┬───────┘  │
│         │                    │                    │           │
│         └────────────┬───────┴────────────────────┘           │
│                      │                                        │
│              ┌───────▼───────────┐                            │
│              │  TanStack Query   │                            │
│              │  (Cache Layer)    │                            │
│              │  - Invalidation   │                            │
│              │  - Direct updates │                            │
│              │  - Optimistic UI  │                            │
│              └───────┬───────────┘                            │
│                      │                                        │
│              ┌───────▼───────────┐                            │
│              │   React State     │                            │
│              │   (UI Layer)      │                            │
│              └───────────────────┘                            │
│                                                               │
│  MOBILE-SPECIFIC:                                             │
│  - AppState-aware connect/disconnect                          │
│  - Cursor-based catch-up on reconnect                         │
│  - Push notification bridging for background events           │
│  - Adaptive heartbeat based on connection type                │
│  - Selective subscriptions to reduce bandwidth                │
│  - Message batching to reduce re-renders                      │
└───────────────────────────────────────────────────────────────┘
```

### The Rules

1. **Start with polling.** If it meets your latency requirements and your server can handle the load, stop there.

2. **SSE for one-way push.** Notifications, feeds, AI streaming, live scores -- if the client only receives, SSE is simpler and more robust than WebSocket.

3. **WebSocket for bidirectional.** Chat, typing indicators, collaborative features, and anything where the client sends frequent small messages.

4. **One connection per app.** Do not open multiple WebSocket connections. Use channels, rooms, or message routing to multiplex.

5. **Always implement catch-up.** Assume the connection will drop. Track the last received message ID and request everything you missed on reconnection.

6. **Respect the battery.** Disconnect on background. Adjust heartbeat intervals by connection type. Batch messages. Compress payloads.

7. **Let TanStack Query own the cache.** Whether you use invalidation signals or direct cache updates, TanStack Query should be the single source of truth for your UI. Do not maintain a separate real-time state.

8. **Type everything.** The message envelope, the handler map, the payload types -- type them all. A mistyped WebSocket message is the hardest bug to debug because there is no compiler error and no HTTP status code to help you.

---

### What's Next

- [Ch 43: Push Notifications] -- when the app is backgrounded, push notifications take over from real-time transport
- [Ch 41: Real-Time Collaboration] -- CRDTs and conflict resolution for collaborative editing, which builds on the transport patterns from this chapter
