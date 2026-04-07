<!--
  CHAPTER: 29
  TITLE: The AI-Powered Frontend
  PART: VI — Vercel & Web
  PREREQS: Chapter 24
  KEY_TOPICS: AI SDK v6, useChat, streaming, tool calling, AI Gateway, agents, structured output
  DIFFICULTY: Intermediate
  UPDATED: 2026-04-07
-->

# Chapter 26: The AI-Powered Frontend

> **Part VI — Vercel & Web** | Prerequisites: Chapter 24 | Difficulty: Intermediate

<details>
<summary><strong>TL;DR</strong></summary>

- The Vercel AI SDK is a three-layer architecture (Core, UI, RSC) that handles streaming, tool calling, structured output, and multi-provider routing so you focus on the product experience
- `useChat` manages conversation state, streaming, and error handling for chat interfaces; `useCompletion` handles single-turn generation; both work on web and React Native
- Tool calling lets LLMs execute functions mid-conversation using Zod-validated schemas; this is how you build AI features that take actions, not just generate text
- AI Gateway routes between providers (OpenAI, Anthropic, Google) with automatic failover, cost tracking, and rate limiting; never hardcode a single provider
- Structured output with Zod schemas forces the model to return typed JSON you can safely render; use it whenever you need the AI response to drive UI, not just display text

</details>

Here is the reality of frontend engineering in 2026: if you are not building AI features into your product, your competitors are. And they are shipping faster than you think.

But here is the other reality — one that most engineers discover the hard way: **building AI features that actually work in production is nothing like calling an API and printing the response.** It is streaming partial responses while maintaining UI responsiveness. It is handling tool calls where the model needs to execute functions mid-conversation. It is routing between providers when one goes down at 2 AM. It is structuring output so your frontend can actually parse and render it. It is managing conversation history without blowing up your token budget.

The Vercel AI SDK exists because the gap between "I called GPT and got text back" and "I have a production-ready AI feature" is enormous. It is a full-stack TypeScript toolkit that handles streaming, tool calling, structured output, multi-provider routing, and UI integration — so you can focus on the actual product experience instead of reinventing the plumbing.

This chapter teaches you to build AI-powered features the way production teams actually build them. Not toy demos. Real features — chat interfaces, content generation, AI agents, structured data extraction — with the patterns that survive contact with real users and real scale.

### In This Chapter
- The AI SDK v6 Architecture (Core, UI, RSC)
- Streaming Responses: Why and How
- useChat: Building Chat Interfaces
- useCompletion: Single-Turn Generation
- Tool Calling with Zod Schemas
- Structured Output: Beyond Free Text
- AI Gateway: Multi-Provider Routing
- Building Chat Interfaces That Don't Suck
- AI Features in Mobile Apps
- Building AI Agents with the AI SDK
- Production Patterns and Pitfalls

### Related Chapters
- [Ch 24: Next.js App Router] — server components, route handlers, server actions
- [Ch 25: Vercel Platform Deep Dive] — deployment, edge functions, environment variables
- [Ch 9: State Management at Scale] — managing chat state alongside app state
- [Ch 10: Data Fetching & Server Communication] — streaming patterns, caching

---

## 1. THE AI SDK V6 ARCHITECTURE

The Vercel AI SDK is not a thin wrapper around OpenAI's client library. It is a three-layer architecture designed to handle the full complexity of building AI features in modern web and mobile applications.

```
+-------------------------------------------------------------------+
|                        YOUR APPLICATION                           |
+-------------------------------------------------------------------+
|                                                                   |
|  +--------------------+  +-----------------+  +------------------+|
|  |    AI SDK UI        |  |  AI SDK RSC     |  |                  ||
|  |                     |  |                 |  |  Your Custom UI  ||
|  |  useChat            |  |  streamUI       |  |  (consuming      ||
|  |  useCompletion      |  |  createAI       |  |   core streams)  ||
|  |  useObject          |  |  AI state       |  |                  ||
|  |  useAssistant       |  |  UI state       |  |                  ||
|  +----------+----------+  +--------+--------+  +---------+-------+|
|             |                      |                      |       |
|  +----------+----------------------+----------------------+------+|
|  |                       AI SDK CORE                              ||
|  |                                                                ||
|  |  generateText    generateObject    streamText                  ||
|  |  streamObject    embedMany         tool()                      ||
|  |                                                                ||
|  |  Provider-agnostic — same API for every model                 ||
|  +----------------------------+-----------------------------------+|
|                               |                                   |
+-------------------------------+-----------------------------------+
                                |
+-------------------------------+-----------------------------------+
|                    AI GATEWAY / PROVIDERS                         |
|                               |                                   |
|  +---------+ +----------+ +--+------+ +----------+ +-----------+ |
|  | OpenAI  | |Anthropic | | Google  | | Mistral  | |  Local    | |
|  | GPT-4.1 | |Claude 4  | |Gemini 2 | | Large 3  | |  Ollama   | |
|  +---------+ +----------+ +---------+ +----------+ +-----------+ |
+-------------------------------------------------------------------+
```

### 1.1 Core Layer — The Engine

The Core layer is provider-agnostic. It defines the primitives for generating text, streaming text, generating structured objects, embedding text, and calling tools. It does not know or care whether you are using OpenAI, Anthropic, Google, or a local Ollama instance. You swap providers by changing one import.

```typescript
// The provider is the only thing that changes
import { openai } from '@ai-sdk/openai';
import { anthropic } from '@ai-sdk/anthropic';
import { google } from '@ai-sdk/google';

// Same API, different model
const result = await generateText({
  model: openai('gpt-4.1'),
  prompt: 'Explain React Server Components in one paragraph.',
});

// Swap to Anthropic — identical call shape
const result2 = await generateText({
  model: anthropic('claude-sonnet-4-20250514'),
  prompt: 'Explain React Server Components in one paragraph.',
});
```

This is not a trivial abstraction. Each provider has different API shapes, different streaming formats, different tool calling conventions, different token counting, and different error shapes. The Core layer normalizes all of it.

**Key Core functions:**

| Function | Purpose | Returns |
|----------|---------|---------|
| `generateText` | Single-shot text generation | `{ text, usage, toolCalls, toolResults }` |
| `streamText` | Streaming text generation | `ReadableStream` with helper methods |
| `generateObject` | Structured output (JSON) | Typed object matching your Zod schema |
| `streamObject` | Streaming structured output | Progressive object building |
| `embedMany` | Batch text embeddings | `{ embeddings, usage }` |
| `tool` | Define callable tools | Tool definition with Zod params |

### 1.2 UI Layer — React Hooks

The UI layer provides React hooks that handle the entire client-side lifecycle of AI interactions: sending messages, receiving streaming responses, managing conversation history, handling loading states, and dealing with errors. This is where `useChat`, `useCompletion`, and `useObject` live.

The critical insight: **these hooks manage a streaming protocol.** They are not just making API calls. They are consuming a custom stream format that carries text chunks, tool call requests, tool results, and metadata — all interleaved in real-time. Building this from scratch is where most teams burn weeks.

### 1.3 RSC Layer — Server Component Integration

The RSC (React Server Components) layer lets you stream AI-generated UI from the server. Instead of streaming text that the client renders, you stream actual React components. The model decides what UI to show, and the server renders it as a React tree that gets streamed to the client.

This is the most advanced layer, and honestly, you probably do not need it right away. Start with Core + UI. Move to RSC when you are building conversational interfaces where the AI generates complex, interactive UI — not just text.

---

## 2. STREAMING RESPONSES: WHY AND HOW

Let me explain why streaming matters with a simple math problem.

A model generating 500 tokens at 50 tokens per second takes 10 seconds. Without streaming, your user stares at a loading spinner for 10 seconds, then the entire response appears at once. With streaming, the first token appears after about 200 milliseconds, and the response builds word by word. The total time is identical. The perceived experience is completely different.

**Streaming is not a performance optimization. It is a UX optimization.** And for AI features, it is non-negotiable. Every major AI product streams. Users expect it. The absence of streaming feels broken.

### 2.1 How AI SDK Streaming Works

The AI SDK uses a custom streaming protocol built on top of HTTP streams. When you call `streamText` on the server and return the result, the SDK sends chunks over the wire in a format that carries more than just text:

```typescript
// app/api/chat/route.ts (Next.js Route Handler)
import { openai } from '@ai-sdk/openai';
import { streamText } from 'ai';

export async function POST(req: Request) {
  const { messages } = await req.json();

  const result = streamText({
    model: openai('gpt-4.1'),
    messages,
    system: 'You are a helpful assistant for a developer documentation site.',
  });

  // toDataStreamResponse() converts the stream to the AI SDK wire format
  return result.toDataStreamResponse();
}
```

On the client side, `useChat` consumes this stream and progressively updates the UI:

```tsx
// components/Chat.tsx
'use client';

import { useChat } from '@ai-sdk/react';

export function Chat() {
  const { messages, input, handleInputChange, handleSubmit, isLoading } =
    useChat();

  return (
    <div className="flex flex-col h-full">
      <div className="flex-1 overflow-y-auto p-4 space-y-4">
        {messages.map((m) => (
          <div
            key={m.id}
            className={m.role === 'user' ? 'text-right' : 'text-left'}
          >
            <span className="inline-block p-3 rounded-lg bg-gray-100">
              {m.content}
            </span>
          </div>
        ))}
      </div>

      <form onSubmit={handleSubmit} className="p-4 border-t">
        <input
          value={input}
          onChange={handleInputChange}
          placeholder="Ask something..."
          className="w-full p-2 border rounded"
          disabled={isLoading}
        />
      </form>
    </div>
  );
}
```

That is a complete streaming chat interface. The `useChat` hook handles:
- Sending messages to your API endpoint
- Parsing the streaming response
- Progressively appending text as it arrives
- Managing the full conversation history
- Loading state management
- Error handling

### 2.2 The Stream Wire Format

Under the hood, the AI SDK data stream protocol sends typed chunks. You do not usually need to know this, but understanding it helps when debugging:

```
0:"Hello"
0:", "
0:"how "
0:"can "
0:"I "
0:"help"
0:"?"
e:{"finishReason":"stop","usage":{"promptTokens":15,"completionTokens":7}}
d:{"finishReason":"stop","usage":{"promptTokens":15,"completionTokens":7}}
```

Each line is prefixed with a type identifier:
- `0` — text chunk
- `2` — data (custom metadata)
- `9` — tool call start
- `a` — tool call delta
- `b` — tool result
- `e` — step finish
- `d` — finish message

This is important because it means the stream carries much more than text. Tool calls, tool results, and metadata all flow through the same stream. The client-side hooks know how to parse all of these and update the UI accordingly.

### 2.3 Server-Sent Events vs. Data Stream

The AI SDK supports two response formats:

```typescript
// Option 1: AI SDK Data Stream (default, recommended)
return result.toDataStreamResponse();

// Option 2: Standard text stream (simpler, fewer features)
return result.toTextStreamResponse();
```

The Data Stream format carries tool calls, metadata, and multi-step interactions. Use it unless you have a specific reason to use plain text (like consuming the stream from a non-AI-SDK client).

---

## 3. USECHAT: BUILDING CHAT INTERFACES

`useChat` is the workhorse hook. It manages a multi-turn conversation with full streaming support, tool calling, and error handling. Let us go beyond the basic example and build something production-worthy.

### 3.1 The Full API Surface

```tsx
const {
  // State
  messages,          // Message[] — the full conversation
  input,             // string — current input value
  isLoading,         // boolean — whether a response is streaming
  error,             // Error | undefined
  data,              // JSONValue[] | undefined — custom server data

  // Actions
  handleInputChange, // ChangeEventHandler — controlled input
  handleSubmit,      // FormEventHandler — submit the current input
  append,            // (message) => void — programmatically add a message
  reload,            // () => void — regenerate the last response
  stop,              // () => void — stop the current stream
  setMessages,       // (messages) => void — replace the conversation
  setInput,          // (input) => void — set the input value

  // Metadata
  id,                // string — conversation ID
} = useChat({
  // Configuration
  api: '/api/chat',           // endpoint (default: '/api/chat')
  id: 'my-chat',              // unique ID for this conversation
  initialMessages: [],        // pre-populate the conversation
  body: {},                   // extra data sent with every request
  headers: {},                // extra headers
  maxSteps: 5,                // max tool-call round trips (critical for agents)

  // Callbacks
  onFinish: (message) => {},  // called when response is complete
  onError: (error) => {},     // called on error
  onResponse: (response) => {},  // called when HTTP response starts
});
```

### 3.2 Multi-Turn Conversations with Context

A chat is only as good as its context management. Here is how to build a chat that maintains context properly:

```tsx
// app/api/chat/route.ts
import { openai } from '@ai-sdk/openai';
import { streamText } from 'ai';
import { auth } from '@/lib/auth';
import { getChatHistory, saveChatMessage } from '@/lib/db';

export async function POST(req: Request) {
  const session = await auth();
  if (!session?.user) {
    return new Response('Unauthorized', { status: 401 });
  }

  const { messages, chatId } = await req.json();

  // Load persisted chat history if resuming
  const history = chatId
    ? await getChatHistory(chatId, session.user.id)
    : [];

  const allMessages = [...history, ...messages];

  const result = streamText({
    model: openai('gpt-4.1'),
    system: `You are a customer support agent for Acme Corp.
Current user: ${session.user.name} (${session.user.email})
Account type: ${session.user.plan}
Today's date: ${new Date().toISOString().split('T')[0]}

Rules:
- Be concise and helpful
- If the user asks about billing, use the getBillingInfo tool
- If you don't know something, say so — don't make things up
- Reference the user by name when appropriate`,
    messages: allMessages,
    maxTokens: 1000,
    temperature: 0.7,
    onFinish: async ({ text, usage }) => {
      // Persist the assistant's response
      if (chatId) {
        await saveChatMessage(chatId, {
          role: 'assistant',
          content: text,
          usage,
          timestamp: new Date(),
        });
      }
    },
  });

  return result.toDataStreamResponse();
}
```

### 3.3 Handling Loading, Errors, and Empty States

The difference between a demo and a product is how you handle the unhappy paths:

```tsx
'use client';

import { useChat } from '@ai-sdk/react';

export function ProductChat() {
  const {
    messages,
    input,
    handleInputChange,
    handleSubmit,
    isLoading,
    error,
    reload,
    stop,
  } = useChat({
    onError: (err) => {
      // Log to your error tracking service
      console.error('Chat error:', err);
    },
  });

  return (
    <div className="flex flex-col h-[600px]">
      {/* Messages */}
      <div className="flex-1 overflow-y-auto p-4 space-y-4">
        {messages.length === 0 && (
          <div className="text-center text-gray-500 mt-20">
            <p className="text-lg font-medium">How can I help you today?</p>
            <div className="mt-4 space-y-2">
              <SuggestionChip text="How do I reset my password?" />
              <SuggestionChip text="What's included in the Pro plan?" />
              <SuggestionChip text="I need to update my billing info" />
            </div>
          </div>
        )}

        {messages.map((m) => (
          <MessageBubble key={m.id} message={m} />
        ))}

        {isLoading && (
          <div className="flex items-center gap-2 text-gray-400">
            <TypingIndicator />
            <button
              onClick={stop}
              className="text-xs text-blue-500 hover:underline"
            >
              Stop generating
            </button>
          </div>
        )}

        {error && (
          <div className="p-3 bg-red-50 border border-red-200 rounded-lg">
            <p className="text-red-800 text-sm">
              Something went wrong. Please try again.
            </p>
            <button
              onClick={reload}
              className="mt-2 text-sm text-red-600 hover:underline"
            >
              Retry last message
            </button>
          </div>
        )}
      </div>

      {/* Input */}
      <form onSubmit={handleSubmit} className="p-4 border-t flex gap-2">
        <input
          value={input}
          onChange={handleInputChange}
          placeholder="Type a message..."
          className="flex-1 p-2 border rounded-lg"
          disabled={isLoading}
        />
        <button
          type="submit"
          disabled={isLoading || !input.trim()}
          className="px-4 py-2 bg-blue-600 text-white rounded-lg disabled:opacity-50"
        >
          Send
        </button>
      </form>
    </div>
  );
}
```

### 3.4 Sending Extra Data with Requests

Often you need to send context beyond just the messages — the current page, user preferences, selected documents:

```tsx
const { messages, handleSubmit } = useChat({
  // Static extra data sent with every request
  body: {
    projectId: currentProject.id,
    language: userPreferences.language,
  },
});

// Or dynamic data per-message
const handleSendWithContext = () => {
  handleSubmit(undefined, {
    body: {
      selectedDocuments: selectedDocs.map((d) => d.id),
      currentPage: window.location.pathname,
    },
  });
};
```

On the server, this arrives in the request body:

```typescript
export async function POST(req: Request) {
  const { messages, projectId, language, selectedDocuments } = await req.json();
  // Use the extra context in your system prompt or tool calls
}
```

---

## 4. USECOMPLETION: SINGLE-TURN GENERATION

Not everything is a conversation. Sometimes you need a single prompt-response interaction: summarize this text, generate a product description, rewrite this email. That is what `useCompletion` is for.

```tsx
'use client';

import { useCompletion } from '@ai-sdk/react';

export function EmailRewriter() {
  const { completion, input, handleInputChange, handleSubmit, isLoading } =
    useCompletion({
      api: '/api/rewrite',
    });

  return (
    <div className="grid grid-cols-2 gap-4">
      <div>
        <h3 className="font-medium mb-2">Original Email</h3>
        <form onSubmit={handleSubmit}>
          <textarea
            value={input}
            onChange={handleInputChange}
            rows={10}
            className="w-full p-3 border rounded-lg"
            placeholder="Paste the email you want to rewrite..."
          />
          <button
            type="submit"
            disabled={isLoading}
            className="mt-2 px-4 py-2 bg-blue-600 text-white rounded-lg"
          >
            {isLoading ? 'Rewriting...' : 'Rewrite'}
          </button>
        </form>
      </div>

      <div>
        <h3 className="font-medium mb-2">Rewritten Email</h3>
        <div className="p-3 border rounded-lg min-h-[200px] whitespace-pre-wrap">
          {completion || (
            <span className="text-gray-400">
              Rewritten email will appear here...
            </span>
          )}
        </div>
      </div>
    </div>
  );
}
```

The server side for completions:

```typescript
// app/api/rewrite/route.ts
import { openai } from '@ai-sdk/openai';
import { streamText } from 'ai';

export async function POST(req: Request) {
  const { prompt } = await req.json();

  const result = streamText({
    model: openai('gpt-4.1-mini'),
    system: `You are a professional email editor. Rewrite the given email to be:
- More concise (aim for 50% fewer words)
- More professional in tone
- Clearer in its ask or message
Preserve the original intent. Output only the rewritten email, no commentary.`,
    prompt,
  });

  return result.toDataStreamResponse();
}
```

### useCompletion vs. useChat: When to Use Which

Use `useCompletion` when:
- Single input, single output (no conversation history)
- The user provides content to transform (rewrite, summarize, translate)
- You do not need multi-turn back-and-forth

Use `useChat` when:
- Multi-turn conversation
- Follow-up questions are expected
- Tool calling is involved
- The AI needs to remember what was said earlier

---

## 5. TOOL CALLING WITH ZOD SCHEMAS

Tool calling is where AI features go from "cute demo" to "useful product feature." Instead of the model only generating text, it can request to call functions you define — check the weather, query a database, create a ticket, look up a user's order.

### 5.1 How Tool Calling Works

The flow is:

```
User: "What's the status of order #12345?"
  |
  v
Model: "I need to call the getOrderStatus tool with orderId: 12345"
  |
  v
Your Code: Executes getOrderStatus(12345) -> { status: 'shipped', eta: '2026-04-09' }
  |
  v
Model: "Your order #12345 has been shipped and is expected to arrive on April 9th."
```

The model does not execute the function itself. It returns a structured request saying "I want to call this tool with these arguments." Your code executes the function, returns the result, and the model uses that result to generate its final response.

### 5.2 Defining Tools with Zod

The AI SDK uses Zod schemas to define tool parameters. This gives you runtime validation and TypeScript type inference:

```typescript
import { tool } from 'ai';
import { z } from 'zod';

const getOrderStatus = tool({
  description: 'Get the current status of a customer order by order ID',
  parameters: z.object({
    orderId: z.string().describe('The order ID (e.g., ORD-12345)'),
  }),
  execute: async ({ orderId }) => {
    // This runs on your server — query your database, call your API
    const order = await db.orders.findUnique({ where: { id: orderId } });
    if (!order) {
      return { error: 'Order not found' };
    }
    return {
      orderId: order.id,
      status: order.status,
      estimatedDelivery: order.estimatedDelivery,
      trackingNumber: order.trackingNumber,
    };
  },
});

const getAccountBalance = tool({
  description: 'Get the current account balance and recent transactions',
  parameters: z.object({
    userId: z.string().describe('The user ID'),
    includeTransactions: z
      .boolean()
      .optional()
      .default(false)
      .describe('Whether to include recent transactions'),
  }),
  execute: async ({ userId, includeTransactions }) => {
    const account = await db.accounts.findUnique({ where: { userId } });
    const result: Record<string, unknown> = {
      balance: account.balance,
      currency: account.currency,
    };
    if (includeTransactions) {
      result.recentTransactions = await db.transactions.findMany({
        where: { userId },
        orderBy: { createdAt: 'desc' },
        take: 5,
      });
    }
    return result;
  },
});
```

### 5.3 Using Tools in streamText

```typescript
// app/api/chat/route.ts
import { openai } from '@ai-sdk/openai';
import { streamText } from 'ai';

export async function POST(req: Request) {
  const { messages } = await req.json();
  const session = await auth();

  const result = streamText({
    model: openai('gpt-4.1'),
    system: `You are a customer support agent. Use the available tools to look up
information when the customer asks about their orders or account.
Current user ID: ${session.user.id}`,
    messages,
    tools: {
      getOrderStatus,
      getAccountBalance,
    },
    maxSteps: 5, // Allow up to 5 tool call rounds
  });

  return result.toDataStreamResponse();
}
```

**`maxSteps` is critical.** Without it, the model will return a tool call request, but will not automatically continue the conversation after the tool executes. Setting `maxSteps: 5` means the SDK will:

1. Send messages to the model
2. If the model requests a tool call, execute the tool
3. Send the tool result back to the model
4. Let the model generate a response (or call another tool)
5. Repeat up to 5 times

This is what enables multi-step reasoning: "Let me check your order... it has been shipped. Let me also check your account to see if the charge has posted..."

### 5.4 Client-Side Tool Call Rendering

When the model calls tools, those tool calls appear in the message stream. You can render them:

```tsx
function MessageBubble({ message }: { message: Message }) {
  return (
    <div>
      {/* Render text parts */}
      {message.content && <p>{message.content}</p>}

      {/* Render tool invocations */}
      {message.toolInvocations?.map((toolInvocation) => (
        <ToolCallDisplay
          key={toolInvocation.toolCallId}
          tool={toolInvocation}
        />
      ))}
    </div>
  );
}

function ToolCallDisplay({ tool }: { tool: ToolInvocation }) {
  if (tool.state === 'call') {
    return (
      <div className="p-2 bg-gray-50 rounded text-sm">
        <span className="text-gray-500">Looking up {tool.toolName}...</span>
      </div>
    );
  }

  if (tool.state === 'result') {
    // Render based on which tool was called
    switch (tool.toolName) {
      case 'getOrderStatus':
        return <OrderStatusCard data={tool.result} />;
      case 'getAccountBalance':
        return <AccountBalanceCard data={tool.result} />;
      default:
        return <pre>{JSON.stringify(tool.result, null, 2)}</pre>;
    }
  }

  return null;
}
```

This is powerful. Instead of the model trying to describe the order status in text, you render a rich UI card. The model provides the data, your components provide the presentation.

### 5.5 Client-Side Tool Confirmation

Sometimes you want the user to confirm before a tool executes — especially for actions that modify data:

```typescript
// Server-side: define a tool without execute (human-in-the-loop)
const cancelOrder = tool({
  description: 'Cancel a customer order',
  parameters: z.object({
    orderId: z.string(),
    reason: z.string().optional(),
  }),
  // No execute function — this tool requires client-side confirmation
});
```

```tsx
// Client-side: handle the confirmation
function ToolCallDisplay({ tool, addToolResult }) {
  if (tool.toolName === 'cancelOrder' && tool.state === 'call') {
    return (
      <div className="p-4 border border-yellow-300 bg-yellow-50 rounded-lg">
        <p className="font-medium">Cancel order {tool.args.orderId}?</p>
        {tool.args.reason && (
          <p className="text-sm mt-1">{tool.args.reason}</p>
        )}
        <div className="mt-3 flex gap-2">
          <button
            onClick={async () => {
              const result = await cancelOrderAction(tool.args.orderId);
              addToolResult({
                toolCallId: tool.toolCallId,
                result,
              });
            }}
            className="px-3 py-1 bg-red-600 text-white rounded"
          >
            Yes, cancel it
          </button>
          <button
            onClick={() => {
              addToolResult({
                toolCallId: tool.toolCallId,
                result: { cancelled: false, reason: 'User declined' },
              });
            }}
            className="px-3 py-1 bg-gray-200 rounded"
          >
            No, keep it
          </button>
        </div>
      </div>
    );
  }
  // ... other tools
}
```

This is the human-in-the-loop pattern, and it is essential for any AI feature that has side effects.

---

## 6. STRUCTURED OUTPUT: BEYOND FREE TEXT

Here is a problem you will hit quickly: the model generates text, but your frontend needs structured data. You want a JSON object with specific fields, not a paragraph of prose. You want a list of extracted entities, not a sentence mentioning them.

### 6.1 generateObject — Type-Safe AI Output

`generateObject` constrains the model to output data matching a Zod schema. This is not string-parsing-and-hoping — the AI SDK uses the model's native structured output mode (JSON mode, function calling mode, or grammar-constrained generation depending on the provider):

```typescript
import { openai } from '@ai-sdk/openai';
import { generateObject } from 'ai';
import { z } from 'zod';

const ProductAnalysisSchema = z.object({
  sentiment: z.enum(['positive', 'negative', 'neutral', 'mixed']),
  keyThemes: z.array(
    z.object({
      theme: z.string(),
      sentiment: z.enum(['positive', 'negative', 'neutral']),
      frequency: z
        .number()
        .min(0)
        .max(1)
        .describe('How often this theme appears, 0-1'),
    })
  ),
  summary: z.string().max(200),
  actionItems: z.array(z.string()),
  overallScore: z.number().min(1).max(10),
});

type ProductAnalysis = z.infer<typeof ProductAnalysisSchema>;

export async function analyzeReviews(
  reviews: string[]
): Promise<ProductAnalysis> {
  const { object } = await generateObject({
    model: openai('gpt-4.1'),
    schema: ProductAnalysisSchema,
    prompt: `Analyze these product reviews and extract insights:

${reviews.map((r, i) => `Review ${i + 1}: ${r}`).join('\n\n')}`,
  });

  // object is fully typed as ProductAnalysis
  return object;
}
```

The return type is guaranteed to match your schema. No `JSON.parse`. No type assertions. No runtime validation on your end. The AI SDK handles all of it.

### 6.2 streamObject — Progressive Structured Output

For larger objects, you can stream the construction:

```tsx
'use client';

import { useObject } from '@ai-sdk/react';
import { ProductAnalysisSchema } from '@/lib/schemas';

export function ReviewAnalyzer({ reviews }: { reviews: string[] }) {
  const { object, submit, isLoading } = useObject({
    api: '/api/analyze-reviews',
    schema: ProductAnalysisSchema,
  });

  return (
    <div>
      <button onClick={() => submit({ reviews })} disabled={isLoading}>
        Analyze Reviews
      </button>

      {/* object is progressively built — fields appear as they're generated */}
      {object && (
        <div>
          {object.sentiment && (
            <Badge variant={object.sentiment}>{object.sentiment}</Badge>
          )}

          {object.overallScore !== undefined && (
            <ScoreDisplay score={object.overallScore} />
          )}

          {object.keyThemes?.map((theme, i) => (
            <ThemeCard key={i} theme={theme} />
          ))}

          {object.summary && <p>{object.summary}</p>}

          {object.actionItems && (
            <ul>
              {object.actionItems.map((item, i) => (
                <li key={i}>{item}</li>
              ))}
            </ul>
          )}
        </div>
      )}
    </div>
  );
}
```

The magic here: `object` is a partial version of your schema type. Fields appear one by one as the model generates them. Your UI progressively renders, giving the user immediate feedback instead of waiting for the entire analysis.

### 6.3 Enum and Union Types

Structured output really shines for classification and routing:

```typescript
const TicketClassificationSchema = z.object({
  category: z.enum([
    'billing',
    'technical',
    'account',
    'feature_request',
    'other',
  ]),
  priority: z.enum(['low', 'medium', 'high', 'urgent']),
  subCategory: z.string(),
  suggestedAssignee: z.enum([
    'billing_team',
    'engineering',
    'account_management',
    'product',
    'general_support',
  ]),
  requiresEscalation: z.boolean(),
  estimatedComplexity: z.number().min(1).max(5),
});

// Automatically route support tickets
export async function classifyTicket(ticketText: string) {
  const { object } = await generateObject({
    model: openai('gpt-4.1-mini'), // Mini is great for classification
    schema: TicketClassificationSchema,
    prompt: `Classify this support ticket:\n\n${ticketText}`,
  });

  // Route based on classification
  if (object.requiresEscalation || object.priority === 'urgent') {
    await notifyOnCall(object);
  }

  await assignTicket(object.suggestedAssignee, ticketText, object);
  return object;
}
```

### 6.4 When to Use Structured Output vs. Tool Calling

Both give you structured data. The difference:

- **Structured output** (`generateObject`): The entire response is a structured object. Use when you want the model to analyze, classify, or extract and return data.
- **Tool calling**: The model generates text AND can call functions during generation. Use when the model needs to take actions or retrieve information mid-conversation.

```
Need to extract data from text?           -> generateObject
Need to classify/categorize?              -> generateObject
Need to look up info during a chat?       -> tool calling
Need to perform actions (create, update)? -> tool calling
Need both structured data AND actions?    -> tool calling + generateObject
```

---

## 7. AI GATEWAY: MULTI-PROVIDER ROUTING

Here is a thing that will happen to you in production: your AI feature goes down because OpenAI has an outage. Or you are burning money on GPT-4.1 for tasks that GPT-4.1-mini handles just fine. Or your enterprise customers require Anthropic because their compliance team vetted it. Or you want to A/B test Claude vs. GPT to see which gives better results for your specific use case.

This is what AI Gateway solves.

### 7.1 What AI Gateway Does

Vercel AI Gateway is a proxy layer that sits between your application and AI providers. It handles:

- **Multi-provider routing**: Send requests to different providers based on rules
- **Automatic failover**: If OpenAI is down, fall back to Anthropic
- **Cost tracking**: See exactly how much you are spending per feature, per user, per model
- **Rate limiting**: Prevent runaway costs from bugs or abuse
- **Caching**: Cache identical requests to avoid redundant API calls
- **Logging and observability**: Every request is logged with latency, tokens, cost

### 7.2 Setting Up AI Gateway

AI Gateway is configured through your Vercel project. You configure provider credentials and routing rules in the Vercel dashboard, then reference models through the gateway in your code:

```typescript
// lib/ai.ts
import { gateway } from '@ai-sdk/gateway';

// The gateway wraps your configured providers
export const myGateway = gateway({
  baseURL: 'https://gateway.vercel.ai/v1',
  // API key is read from VERCEL_AI_GATEWAY_API_KEY env var
});
```

You can then use gateway model references in your AI SDK calls. The gateway handles the routing, failover, and provider credential management:

```typescript
import { streamText } from 'ai';
import { myGateway } from '@/lib/ai';

const result = streamText({
  model: myGateway('openai/gpt-4.1'), // provider/model format
  messages,
});
```

### 7.3 Provider-Specific Routing

Even without the managed gateway, you can route between providers at the application level:

```typescript
import { openai } from '@ai-sdk/openai';
import { anthropic } from '@ai-sdk/anthropic';
import { google } from '@ai-sdk/google';

// Route based on task type
function getModel(task: 'chat' | 'analysis' | 'code' | 'classification') {
  switch (task) {
    case 'chat':
      return anthropic('claude-sonnet-4-20250514'); // Great at conversation
    case 'analysis':
      return openai('gpt-4.1');       // Strong at structured analysis
    case 'code':
      return anthropic('claude-sonnet-4-20250514'); // Excellent at code gen
    case 'classification':
      return openai('gpt-4.1-mini');   // Fast and cheap for classification
  }
}

// Use it
const result = await streamText({
  model: getModel('chat'),
  messages,
});
```

### 7.4 Implementing Failover Manually

If you are not using Vercel's managed gateway, you can implement failover yourself:

```typescript
import { openai } from '@ai-sdk/openai';
import { anthropic } from '@ai-sdk/anthropic';
import { APICallError } from 'ai';

async function generateWithFallback(
  messages: CoreMessage[],
  systemPrompt: string,
) {
  const providers = [
    () => openai('gpt-4.1'),
    () => anthropic('claude-sonnet-4-20250514'),
  ];

  for (const getModel of providers) {
    try {
      return await streamText({
        model: getModel(),
        system: systemPrompt,
        messages,
      });
    } catch (error) {
      if (error instanceof APICallError && error.statusCode === 429) {
        // Rate limited — try next provider
        continue;
      }
      if (error instanceof APICallError && error.statusCode >= 500) {
        // Provider error — try next
        continue;
      }
      // Unknown error — throw
      throw error;
    }
  }

  throw new Error('All AI providers are unavailable');
}
```

### 7.5 Cost Optimization Strategies

This is where teams waste the most money. Here are the patterns that actually matter:

```typescript
// Pattern 1: Use the cheapest model that works
// Don't use GPT-4.1 for yes/no classification
const { object: classification } = await generateObject({
  model: openai('gpt-4.1-mini'), // 10x cheaper than gpt-4.1
  schema: z.object({ isSpam: z.boolean(), confidence: z.number() }),
  prompt: `Is this comment spam?\n\n${comment}`,
});

// Pattern 2: Cascade from cheap to expensive
async function answerQuestion(question: string, context: string) {
  // Try cheap model first
  const quickAnswer = await generateText({
    model: openai('gpt-4.1-mini'),
    prompt: `${context}\n\nQuestion: ${question}\n\nIf you can answer confidently, provide the answer. If you're not sure, respond with exactly "ESCALATE".`,
  });

  if (!quickAnswer.text.includes('ESCALATE')) {
    return quickAnswer.text;
  }

  // Escalate to expensive model
  const deepAnswer = await generateText({
    model: openai('gpt-4.1'),
    prompt: `${context}\n\nQuestion: ${question}`,
  });

  return deepAnswer.text;
}

// Pattern 3: Cache identical requests
import { unstable_cache } from 'next/cache';

const cachedClassify = unstable_cache(
  async (text: string) => {
    const { object } = await generateObject({
      model: openai('gpt-4.1-mini'),
      schema: ClassificationSchema,
      prompt: `Classify: ${text}`,
    });
    return object;
  },
  ['classify'],
  { revalidate: 3600 } // Cache for 1 hour
);
```

---

## 8. BUILDING CHAT INTERFACES THAT DON'T SUCK

I have seen dozens of AI chat interfaces in production apps. Most of them are terrible. Not because the AI is bad, but because the UI is an afterthought. Let me show you what a good one looks like.

### 8.1 The Anatomy of a Good Chat UI

```
+---------------------------------------------------------+
|  Chat Title / Context Indicator          [New Chat] [.]  |
+---------------------------------------------------------+
|                                                         |
|  System: Welcome! I can help you with orders,           |
|  billing, and account questions.                        |
|                                                         |
|  +----------------------------------+                   |
|  | User: What's my order status?    |                   |
|  +----------------------------------+                   |
|                                                         |
|  +-------------------------------------------+          |
|  | Assistant:                                 |          |
|  | Looking up your recent orders...           |          |
|  |                                            |          |
|  | +-------------------------------+          |          |
|  | | Order #ORD-12345              |          |          |
|  | | Status: Shipped               |          |          |
|  | | ETA: April 9, 2026            |          |          |
|  | | [Track Package]               |          |          |
|  | +-------------------------------+          |          |
|  |                                            |          |
|  | Your order was shipped yesterday and       |          |
|  | should arrive by Wednesday.                |          |
|  +-------------------------------------------+          |
|                                                         |
|  [Was this helpful? Yes / No]                           |
|                                                         |
+---------------------------------------------------------+
|  +----------------------------------+  [Attach] [Send]  |
|  | Type your message...             |                   |
|  +----------------------------------+                   |
|  Powered by AI - May not always be accurate             |
+---------------------------------------------------------+
```

### 8.2 Production Chat Component

Here is a complete, production-quality chat component:

```tsx
'use client';

import { useChat } from '@ai-sdk/react';
import { useRef, useEffect, useCallback } from 'react';
import { Message } from 'ai';

interface ChatProps {
  chatId?: string;
  systemMessage?: string;
  className?: string;
}

export function Chat({ chatId, systemMessage, className }: ChatProps) {
  const scrollRef = useRef<HTMLDivElement>(null);
  const inputRef = useRef<HTMLTextAreaElement>(null);

  const {
    messages,
    input,
    setInput,
    handleSubmit,
    isLoading,
    error,
    reload,
    stop,
    append,
  } = useChat({
    id: chatId,
    api: '/api/chat',
    body: { chatId },
    initialMessages: systemMessage
      ? [
          {
            id: 'system',
            role: 'system' as const,
            content: systemMessage,
          },
        ]
      : [],
    onFinish: () => {
      // Focus the input after response completes
      inputRef.current?.focus();
    },
  });

  // Auto-scroll to bottom on new messages
  useEffect(() => {
    if (scrollRef.current) {
      scrollRef.current.scrollTop = scrollRef.current.scrollHeight;
    }
  }, [messages]);

  // Handle textarea submit on Enter (Shift+Enter for newline)
  const handleKeyDown = useCallback(
    (e: React.KeyboardEvent<HTMLTextAreaElement>) => {
      if (e.key === 'Enter' && !e.shiftKey) {
        e.preventDefault();
        if (input.trim() && !isLoading) {
          handleSubmit(e as unknown as React.FormEvent);
        }
      }
    },
    [input, isLoading, handleSubmit]
  );

  // Handle suggestion chips
  const handleSuggestion = useCallback(
    (text: string) => {
      append({ role: 'user', content: text });
    },
    [append]
  );

  const visibleMessages = messages.filter((m) => m.role !== 'system');

  return (
    <div className={`flex flex-col h-full ${className}`}>
      {/* Message List */}
      <div ref={scrollRef} className="flex-1 overflow-y-auto px-4 py-6">
        {visibleMessages.length === 0 ? (
          <EmptyState onSuggestion={handleSuggestion} />
        ) : (
          <div className="max-w-3xl mx-auto space-y-6">
            {visibleMessages.map((message) => (
              <ChatMessage key={message.id} message={message} />
            ))}

            {isLoading && <StreamingIndicator onStop={stop} />}

            {error && <ErrorMessage onRetry={reload} />}
          </div>
        )}
      </div>

      {/* Input Area */}
      <div className="border-t bg-white px-4 py-3">
        <form
          onSubmit={handleSubmit}
          className="max-w-3xl mx-auto flex items-end gap-2"
        >
          <textarea
            ref={inputRef}
            value={input}
            onChange={(e) => setInput(e.target.value)}
            onKeyDown={handleKeyDown}
            placeholder="Message..."
            rows={1}
            className="flex-1 resize-none rounded-xl border p-3 focus:outline-none focus:ring-2 focus:ring-blue-500 max-h-36 overflow-y-auto"
            style={{ height: 'auto', minHeight: '44px' }}
            disabled={isLoading}
          />
          <button
            type="submit"
            disabled={isLoading || !input.trim()}
            className="rounded-xl bg-blue-600 px-4 py-3 text-white hover:bg-blue-700 disabled:opacity-50 disabled:cursor-not-allowed transition-colors"
          >
            Send
          </button>
        </form>
        <p className="text-center text-xs text-gray-400 mt-2">
          AI-generated responses may not always be accurate.
        </p>
      </div>
    </div>
  );
}

function ChatMessage({ message }: { message: Message }) {
  const isUser = message.role === 'user';

  return (
    <div className={`flex ${isUser ? 'justify-end' : 'justify-start'}`}>
      <div
        className={`max-w-[85%] rounded-2xl px-4 py-3 ${
          isUser ? 'bg-blue-600 text-white' : 'bg-gray-100 text-gray-900'
        }`}
      >
        {/* Text content */}
        {message.content && (
          <div className="prose prose-sm max-w-none">
            <MarkdownRenderer content={message.content} />
          </div>
        )}

        {/* Tool invocations */}
        {message.toolInvocations?.map((tool) => (
          <ToolInvocationCard key={tool.toolCallId} invocation={tool} />
        ))}
      </div>
    </div>
  );
}

function EmptyState({
  onSuggestion,
}: {
  onSuggestion: (text: string) => void;
}) {
  const suggestions = [
    "What's the status of my latest order?",
    'How do I update my billing information?',
    'Can you help me find a product?',
  ];

  return (
    <div className="flex flex-col items-center justify-center h-full text-center">
      <h2 className="text-xl font-semibold text-gray-900 mb-2">
        How can I help you today?
      </h2>
      <p className="text-gray-500 mb-6">
        Ask me anything about your account, orders, or our products.
      </p>
      <div className="flex flex-wrap gap-2 justify-center">
        {suggestions.map((s) => (
          <button
            key={s}
            onClick={() => onSuggestion(s)}
            className="px-4 py-2 bg-gray-100 hover:bg-gray-200 rounded-full text-sm text-gray-700 transition-colors"
          >
            {s}
          </button>
        ))}
      </div>
    </div>
  );
}

function StreamingIndicator({ onStop }: { onStop: () => void }) {
  return (
    <div className="flex items-center gap-3">
      <div className="flex gap-1">
        <span className="w-2 h-2 bg-gray-400 rounded-full animate-bounce" />
        <span className="w-2 h-2 bg-gray-400 rounded-full animate-bounce [animation-delay:0.1s]" />
        <span className="w-2 h-2 bg-gray-400 rounded-full animate-bounce [animation-delay:0.2s]" />
      </div>
      <button
        onClick={onStop}
        className="text-xs text-gray-500 hover:text-gray-700"
      >
        Stop generating
      </button>
    </div>
  );
}
```

### 8.3 Markdown Rendering in Chat

AI models output Markdown. You need to render it properly:

```tsx
import ReactMarkdown from 'react-markdown';
import { Prism as SyntaxHighlighter } from 'react-syntax-highlighter';
import { oneLight } from 'react-syntax-highlighter/dist/esm/styles/prism';

function MarkdownRenderer({ content }: { content: string }) {
  return (
    <ReactMarkdown
      components={{
        code({ inline, className, children, ...props }) {
          const match = /language-(\w+)/.exec(className || '');
          return !inline && match ? (
            <div className="relative group">
              <SyntaxHighlighter
                style={oneLight}
                language={match[1]}
                PreTag="div"
                {...props}
              >
                {String(children).replace(/\n$/, '')}
              </SyntaxHighlighter>
              <CopyButton text={String(children)} />
            </div>
          ) : (
            <code
              className="bg-gray-200 px-1 py-0.5 rounded text-sm"
              {...props}
            >
              {children}
            </code>
          );
        },
        a({ href, children }) {
          return (
            <a
              href={href}
              target="_blank"
              rel="noopener noreferrer"
              className="text-blue-600 underline"
            >
              {children}
            </a>
          );
        },
      }}
    >
      {content}
    </ReactMarkdown>
  );
}
```

### 8.4 Conversation Persistence

For any serious chat feature, you need to persist conversations:

```typescript
// lib/chat-store.ts
import { Message } from 'ai';

// Save conversation to your database
export async function saveConversation(
  chatId: string,
  userId: string,
  messages: Message[]
) {
  await db.conversations.upsert({
    where: { id: chatId },
    create: {
      id: chatId,
      userId,
      messages: JSON.stringify(messages),
      title: messages[0]?.content.slice(0, 100) || 'New Chat',
      createdAt: new Date(),
      updatedAt: new Date(),
    },
    update: {
      messages: JSON.stringify(messages),
      updatedAt: new Date(),
    },
  });
}

// Load conversation
export async function loadConversation(chatId: string, userId: string) {
  const conversation = await db.conversations.findUnique({
    where: { id: chatId, userId },
  });

  if (!conversation) return null;

  return {
    ...conversation,
    messages: JSON.parse(conversation.messages) as Message[],
  };
}
```

```tsx
// Use initialMessages to restore a conversation
'use client';

import { useChat } from '@ai-sdk/react';

export function ResumableChat({
  chatId,
  initialMessages,
}: {
  chatId: string;
  initialMessages: Message[];
}) {
  const chat = useChat({
    id: chatId,
    initialMessages,
    api: '/api/chat',
    body: { chatId },
    onFinish: (message) => {
      // Auto-save after each response
      saveConversation(chatId, userId, [...chat.messages, message]);
    },
  });

  // ... render chat UI
}
```

---

## 9. AI FEATURES IN MOBILE APPS

If you are building with React Native and Expo (which, if you have been following this guide, you are), you can use the AI SDK's Core layer directly. The UI hooks are React hooks — they work in React Native too, but you need to handle a few differences.

### 9.1 useChat in React Native

```tsx
// screens/ChatScreen.tsx
import { useChat } from '@ai-sdk/react';
import {
  View,
  TextInput,
  FlatList,
  Text,
  TouchableOpacity,
  KeyboardAvoidingView,
  Platform,
} from 'react-native';

export function ChatScreen() {
  const { messages, input, setInput, handleSubmit, isLoading } = useChat({
    api: `${API_URL}/api/chat`, // Full URL needed in RN
    headers: {
      Authorization: `Bearer ${authToken}`,
    },
  });

  const renderMessage = ({ item }: { item: Message }) => (
    <View
      style={[
        styles.messageBubble,
        item.role === 'user' ? styles.userMessage : styles.assistantMessage,
      ]}
    >
      <Text
        style={
          item.role === 'user' ? styles.userText : styles.assistantText
        }
      >
        {item.content}
      </Text>
    </View>
  );

  return (
    <KeyboardAvoidingView
      behavior={Platform.OS === 'ios' ? 'padding' : 'height'}
      style={styles.container}
      keyboardVerticalOffset={90}
    >
      <FlatList
        data={messages.filter((m) => m.role !== 'system')}
        renderItem={renderMessage}
        keyExtractor={(item) => item.id}
        contentContainerStyle={styles.messageList}
      />

      <View style={styles.inputContainer}>
        <TextInput
          value={input}
          onChangeText={setInput}
          placeholder="Message..."
          style={styles.textInput}
          multiline
          maxLength={2000}
          editable={!isLoading}
        />
        <TouchableOpacity
          onPress={() => handleSubmit()}
          disabled={isLoading || !input.trim()}
          style={[
            styles.sendButton,
            (!input.trim() || isLoading) && styles.sendButtonDisabled,
          ]}
        >
          <Text style={styles.sendButtonText}>Send</Text>
        </TouchableOpacity>
      </View>
    </KeyboardAvoidingView>
  );
}
```

### 9.2 Common AI Features for Mobile

Here are the AI features that actually work well in mobile apps:

**Content Summarization:**
```typescript
// Summarize long articles for mobile reading
export async function summarizeForMobile(content: string) {
  const { object } = await generateObject({
    model: openai('gpt-4.1-mini'),
    schema: z.object({
      headline: z.string().max(80),
      bulletPoints: z.array(z.string()).max(5),
      readTimeMinutes: z.number(),
      keyTakeaway: z.string().max(150),
    }),
    prompt: `Summarize this article for a mobile reader who wants the key points quickly:\n\n${content}`,
  });
  return object;
}
```

**Smart Search:**
```typescript
// Convert natural language queries to structured search
export async function smartSearch(query: string) {
  const { object } = await generateObject({
    model: openai('gpt-4.1-mini'),
    schema: z.object({
      searchTerms: z.array(z.string()),
      filters: z.object({
        category: z.string().optional(),
        priceRange: z
          .object({
            min: z.number().optional(),
            max: z.number().optional(),
          })
          .optional(),
        sortBy: z
          .enum([
            'relevance',
            'price_asc',
            'price_desc',
            'rating',
            'newest',
          ])
          .optional(),
      }),
      intent: z.enum(['browse', 'specific_item', 'compare', 'research']),
    }),
    prompt: `Parse this search query into structured search parameters: "${query}"`,
  });

  // Use structured output to build an actual database query
  return executeSearch(object);
}
```

**Content Generation:**
```typescript
// Generate product descriptions for sellers
export async function generateProductDescription(
  images: string[], // base64 or URLs
  category: string,
  attributes: Record<string, string>
) {
  const { object } = await generateObject({
    model: openai('gpt-4.1'),
    schema: z.object({
      title: z.string().max(100),
      description: z.string().max(500),
      highlights: z.array(z.string()).max(5),
      suggestedTags: z.array(z.string()).max(10),
      suggestedPrice: z
        .object({
          low: z.number(),
          mid: z.number(),
          high: z.number(),
          currency: z.string(),
        })
        .optional(),
    }),
    messages: [
      {
        role: 'user',
        content: [
          {
            type: 'text',
            text: `Generate a product listing for a ${category} item with these attributes: ${JSON.stringify(attributes)}`,
          },
          ...images.map((img) => ({
            type: 'image' as const,
            image: img,
          })),
        ],
      },
    ],
  });

  return object;
}
```

### 9.3 Offline Considerations for Mobile AI

Mobile apps need to handle offline states gracefully:

```tsx
import NetInfo from '@react-native-community/netinfo';

function useMobileChat() {
  const [isOnline, setIsOnline] = useState(true);
  const pendingMessages = useRef<Message[]>([]);

  useEffect(() => {
    const unsubscribe = NetInfo.addEventListener((state) => {
      setIsOnline(state.isConnected ?? false);
    });
    return unsubscribe;
  }, []);

  const chat = useChat({
    api: `${API_URL}/api/chat`,
    onError: () => {
      if (!isOnline) {
        // Queue message for later
        const lastUserMessage = chat.messages
          .filter((m) => m.role === 'user')
          .pop();
        if (lastUserMessage) {
          pendingMessages.current.push(lastUserMessage);
        }
      }
    },
  });

  // Retry pending messages when back online
  useEffect(() => {
    if (isOnline && pendingMessages.current.length > 0) {
      const pending = pendingMessages.current.shift();
      if (pending) {
        chat.append(pending);
      }
    }
  }, [isOnline]);

  return { ...chat, isOnline };
}
```

---

## 10. BUILDING AI AGENTS WITH THE AI SDK

An AI agent is different from a chatbot. A chatbot responds to messages. An agent takes actions autonomously to accomplish a goal. The difference is `maxSteps` and tool design.

### 10.1 What Makes an Agent

```
Chatbot:
  User says something -> Model responds -> Done

Agent:
  User states a goal -> Model plans steps -> Calls tools -> Evaluates results
  -> Calls more tools -> Evaluates again -> Repeats until goal is met -> Reports
```

The AI SDK supports agents through the `maxSteps` parameter combined with well-designed tools:

```typescript
const result = await streamText({
  model: anthropic('claude-sonnet-4-20250514'),
  system: `You are a research assistant. When given a question:
1. Search for relevant information using available tools
2. Cross-reference multiple sources
3. Synthesize the findings into a comprehensive answer
4. Cite your sources

Think step by step. Use tools as needed. Don't stop until you have a thorough answer.`,
  messages,
  tools: {
    webSearch,
    readDocument,
    queryDatabase,
    createNote,
  },
  maxSteps: 10, // Allow up to 10 tool-call rounds
});
```

### 10.2 Designing Agent Tools

The quality of your agent is 80% tool design. Bad tools make bad agents, regardless of the model.

**Good tool design principles:**

```typescript
// GOOD: Clear description, well-typed parameters, focused scope
const searchDocumentation = tool({
  description:
    'Search the product documentation. Returns relevant documentation sections ' +
    'with titles and content. Use this when the user asks how-to questions or ' +
    'needs to understand a feature.',
  parameters: z.object({
    query: z.string().describe('The search query in natural language'),
    section: z
      .enum(['getting-started', 'api-reference', 'guides', 'troubleshooting'])
      .optional()
      .describe('Narrow search to a specific documentation section'),
    limit: z
      .number()
      .min(1)
      .max(10)
      .default(5)
      .describe('Max number of results to return'),
  }),
  execute: async ({ query, section, limit }) => {
    const results = await docSearch.search(query, { section, limit });
    return results.map((r) => ({
      title: r.title,
      content: r.content.slice(0, 500), // Truncate to save tokens
      url: r.url,
      relevanceScore: r.score,
    }));
  },
});

// BAD: Vague description, untyped parameters, too broad
const doStuff = tool({
  description: 'Does things with the database',
  parameters: z.object({
    action: z.string(),
    data: z.any(),
  }),
  execute: async ({ action, data }) => {
    // This is a recipe for the model doing unpredictable things
    return await db.raw(action, data);
  },
});
```

**Key rules:**
1. **Descriptions are instructions.** The model reads them to decide when and how to use each tool. Be specific about when to use it and what it returns.
2. **Parameter descriptions matter.** They guide the model to pass correct values.
3. **Return only what is needed.** Large tool results eat tokens. Truncate, summarize, or paginate.
4. **Each tool should do one thing.** Do not make Swiss Army knife tools. Make specific tools.

### 10.3 A Complete Research Agent

```typescript
// app/api/research/route.ts
import { anthropic } from '@ai-sdk/anthropic';
import { streamText, tool } from 'ai';
import { z } from 'zod';

const searchWeb = tool({
  description: 'Search the web for current information about a topic',
  parameters: z.object({
    query: z.string().describe('The search query'),
  }),
  execute: async ({ query }) => {
    const results = await externalSearchAPI.search(query, { count: 5 });
    return results.map((r) => ({
      title: r.title,
      snippet: r.snippet,
      url: r.url,
    }));
  },
});

const readWebPage = tool({
  description:
    'Read the content of a web page. Use after search to get detailed info.',
  parameters: z.object({
    url: z.string().url().describe('The URL to read'),
  }),
  execute: async ({ url }) => {
    const content = await fetchAndParse(url);
    return {
      title: content.title,
      text: content.text.slice(0, 3000), // Limit token usage
      publishedDate: content.publishedDate,
    };
  },
});

const createReport = tool({
  description:
    'Create a structured research report. Use when you have gathered enough information.',
  parameters: z.object({
    title: z.string(),
    sections: z.array(
      z.object({
        heading: z.string(),
        content: z.string(),
        sources: z.array(z.string().url()),
      })
    ),
    summary: z.string(),
    confidence: z.enum(['high', 'medium', 'low']),
  }),
  execute: async (report) => {
    const saved = await db.reports.create({ data: report });
    return { reportId: saved.id, message: 'Report saved successfully' };
  },
});

export async function POST(req: Request) {
  const { messages } = await req.json();

  const result = streamText({
    model: anthropic('claude-sonnet-4-20250514'),
    system: `You are a thorough research agent. When given a research question:
1. Search the web for relevant, recent information
2. Read the most promising results for detailed information
3. Cross-reference claims across multiple sources
4. Create a structured report with your findings

Be thorough but efficient. Don't search for the same thing twice.
Prefer recent sources. Always cite your sources.`,
    messages,
    tools: {
      searchWeb,
      readWebPage,
      createReport,
    },
    maxSteps: 15, // Research may need many steps
    toolChoice: 'auto', // Let the model decide when to use tools
  });

  return result.toDataStreamResponse();
}
```

### 10.4 Agent Patterns

**Pattern 1: Plan-Execute-Verify**
```typescript
const planningTool = tool({
  description: 'Create a plan of steps before executing. Use this first.',
  parameters: z.object({
    goal: z.string(),
    steps: z.array(
      z.object({
        step: z.number(),
        action: z.string(),
        tool: z.string(),
        reasoning: z.string(),
      })
    ),
  }),
  execute: async (plan) => {
    // Just returns the plan — the model will execute each step
    return { plan, message: 'Plan created. Execute each step in order.' };
  },
});
```

**Pattern 2: Tool Chaining**
```typescript
// Each tool's output naturally feeds into the next
const tools = {
  // Step 1: Identify the problem
  analyzeError: tool({
    description: 'Analyze an error message and identify likely causes',
    parameters: z.object({
      errorMessage: z.string(),
      stackTrace: z.string().optional(),
    }),
    execute: async ({ errorMessage, stackTrace }) => {
      return {
        possibleCauses: ['...'],
        severity: 'high',
        category: 'database',
      };
    },
  }),

  // Step 2: Look up solutions based on analysis
  searchSolutions: tool({
    description: 'Search for solutions to a specific type of error',
    parameters: z.object({
      category: z.string(),
      cause: z.string(),
    }),
    execute: async ({ category, cause }) => {
      return { solutions: ['...'], documentation: ['...'] };
    },
  }),

  // Step 3: Generate a fix
  generateFix: tool({
    description: 'Generate a code fix based on the analysis and solution',
    parameters: z.object({
      solution: z.string(),
      context: z.string(),
    }),
    execute: async ({ solution, context }) => {
      return { fix: '...', explanation: '...' };
    },
  }),
};
```

**Pattern 3: Guardrails**
```typescript
const result = await streamText({
  model: anthropic('claude-sonnet-4-20250514'),
  system: `You are a helpful agent with these HARD CONSTRAINTS:
- NEVER modify production databases (only staging/dev)
- NEVER send emails to more than 10 recipients
- ALWAYS ask for confirmation before deleting anything
- If a tool fails, try at most 2 times before reporting the failure
- Maximum budget: 5 tool calls per request`,
  messages,
  tools,
  maxSteps: 7,
});
```

### 10.5 Rendering Agent Activity

Users need to see what the agent is doing:

```tsx
function AgentChat({ messages }: { messages: Message[] }) {
  return (
    <div>
      {messages.map((message) => (
        <div key={message.id}>
          {/* Regular content */}
          {message.content && <MessageContent content={message.content} />}

          {/* Show tool calls as agent "thinking" steps */}
          {message.toolInvocations?.map((invocation) => (
            <AgentStep key={invocation.toolCallId} invocation={invocation} />
          ))}
        </div>
      ))}
    </div>
  );
}

function AgentStep({ invocation }: { invocation: ToolInvocation }) {
  const [isExpanded, setIsExpanded] = useState(false);

  const stepLabels: Record<string, string> = {
    searchWeb: 'Searching the web',
    readWebPage: 'Reading page',
    queryDatabase: 'Querying data',
    createReport: 'Creating report',
  };

  const stepLabel = stepLabels[invocation.toolName] || invocation.toolName;

  return (
    <div className="ml-4 my-2 border-l-2 border-blue-200 pl-3">
      <button
        onClick={() => setIsExpanded(!isExpanded)}
        className="flex items-center gap-2 text-sm text-gray-600 hover:text-gray-900"
      >
        {invocation.state === 'call' ? (
          <Spinner className="w-4 h-4" />
        ) : (
          <CheckIcon className="w-4 h-4 text-green-500" />
        )}
        <span>{stepLabel}</span>
      </button>

      {isExpanded && invocation.state === 'result' && (
        <div className="mt-1 text-xs text-gray-500 bg-gray-50 p-2 rounded">
          <pre>{JSON.stringify(invocation.result, null, 2)}</pre>
        </div>
      )}
    </div>
  );
}
```

---

## 11. THE RSC LAYER: STREAMING UI

The RSC layer is the most advanced feature of the AI SDK. Instead of streaming text, you stream React components from the server. The model decides what UI to render, and your server creates actual React elements that are streamed to the client.

### 11.1 When You Need This

You need the RSC layer when:
- The AI generates different types of UI based on context (charts, cards, forms, tables)
- You want the AI to compose interactive components, not just text
- You are building a generative UI experience (like an AI dashboard builder)

You probably do not need it when:
- The AI just generates text with occasional tool call cards
- You are building a standard chat interface
- You are early in development (start with useChat, migrate to RSC later)

### 11.2 streamUI Example

```typescript
// app/api/chat-ui/route.ts
import { openai } from '@ai-sdk/openai';
import { streamUI } from 'ai/rsc';
import { z } from 'zod';

export async function submitMessage(userMessage: string) {
  'use server';

  const result = await streamUI({
    model: openai('gpt-4.1'),
    messages: [{ role: 'user', content: userMessage }],
    text: ({ content }) => <AssistantMessage>{content}</AssistantMessage>,
    tools: {
      showWeather: {
        description: 'Show weather information for a location',
        parameters: z.object({
          location: z.string(),
        }),
        generate: async function* ({ location }) {
          yield <WeatherSkeleton />;

          const weather = await getWeather(location);

          return <WeatherCard weather={weather} />;
        },
      },
      showStockChart: {
        description: 'Show a stock price chart',
        parameters: z.object({
          symbol: z.string(),
          period: z.enum(['1D', '1W', '1M', '3M', '1Y']),
        }),
        generate: async function* ({ symbol, period }) {
          yield <ChartSkeleton />;

          const data = await getStockData(symbol, period);

          return <StockChart data={data} symbol={symbol} />;
        },
      },
    },
  });

  return result.value;
}
```

Notice the `generate` functions use `async function*` (async generators). The `yield` keyword streams intermediate UI (like skeletons), and the `return` sends the final UI. This gives users instant visual feedback while data loads.

### 11.3 Client-Side RSC Integration

```tsx
'use client';

import { useState } from 'react';
import { useActions, useUIState } from 'ai/rsc';

export function GenerativeChat() {
  const [input, setInput] = useState('');
  const [messages, setMessages] = useUIState();
  const { submitMessage } = useActions();

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();

    // Add user message to UI immediately
    setMessages((prev) => [
      ...prev,
      {
        id: Date.now().toString(),
        role: 'user',
        display: <UserMessage>{input}</UserMessage>,
      },
    ]);

    // Get streamed UI from server
    const response = await submitMessage(input);

    setMessages((prev) => [
      ...prev,
      {
        id: Date.now().toString(),
        role: 'assistant',
        display: response,
      },
    ]);

    setInput('');
  };

  return (
    <div>
      {messages.map((m) => (
        <div key={m.id}>{m.display}</div>
      ))}
      <form onSubmit={handleSubmit}>
        <input value={input} onChange={(e) => setInput(e.target.value)} />
      </form>
    </div>
  );
}
```

---

## 12. PRODUCTION PATTERNS AND PITFALLS

### 12.1 Rate Limiting

AI API calls are expensive. Protect yourself:

```typescript
import { Ratelimit } from '@upstash/ratelimit';
import { Redis } from '@upstash/redis';

const ratelimit = new Ratelimit({
  redis: Redis.fromEnv(),
  limiter: Ratelimit.slidingWindow(20, '1 h'), // 20 requests per hour
  analytics: true,
});

export async function POST(req: Request) {
  const session = await auth();
  if (!session?.user) {
    return new Response('Unauthorized', { status: 401 });
  }

  const { success, limit, remaining, reset } = await ratelimit.limit(
    session.user.id
  );

  if (!success) {
    return new Response(
      JSON.stringify({
        error: 'Rate limit exceeded',
        limit,
        remaining,
        resetAt: new Date(reset).toISOString(),
      }),
      {
        status: 429,
        headers: {
          'X-RateLimit-Limit': limit.toString(),
          'X-RateLimit-Remaining': remaining.toString(),
          'X-RateLimit-Reset': reset.toString(),
        },
      }
    );
  }

  // ... handle the AI request
}
```

### 12.2 Token Management

Context windows are finite. Manage them:

```typescript
import { encodingForModel } from 'js-tiktoken';

function trimMessagesToFit(
  messages: CoreMessage[],
  maxTokens: number,
  model: string = 'gpt-4.1'
) {
  const encoder = encodingForModel(model);

  // Always keep the system message and the last user message
  const systemMessage = messages.find((m) => m.role === 'system');
  const lastUserMessage = messages.filter((m) => m.role === 'user').pop();

  let tokenCount = 0;
  if (systemMessage) {
    tokenCount += encoder.encode(systemMessage.content as string).length;
  }
  if (lastUserMessage) {
    tokenCount += encoder.encode(lastUserMessage.content as string).length;
  }

  // Add messages from newest to oldest until we hit the limit
  const middleMessages = messages.filter(
    (m) => m !== systemMessage && m !== lastUserMessage
  );

  const kept: CoreMessage[] = [];
  for (let i = middleMessages.length - 1; i >= 0; i--) {
    const content = middleMessages[i].content;
    const msgTokens = encoder.encode(
      typeof content === 'string' ? content : JSON.stringify(content)
    ).length;

    if (tokenCount + msgTokens > maxTokens) break;
    tokenCount += msgTokens;
    kept.unshift(middleMessages[i]);
  }

  return [
    ...(systemMessage ? [systemMessage] : []),
    ...kept,
    ...(lastUserMessage ? [lastUserMessage] : []),
  ];
}
```

### 12.3 Error Handling Patterns

```typescript
import { APICallError } from 'ai';

export async function POST(req: Request) {
  try {
    const { messages } = await req.json();

    const result = streamText({
      model: openai('gpt-4.1'),
      messages,
    });

    return result.toDataStreamResponse();
  } catch (error) {
    if (error instanceof APICallError) {
      // Provider API error (rate limit, server error, etc.)
      console.error('AI Provider Error:', {
        statusCode: error.statusCode,
        message: error.message,
        provider: 'openai',
      });

      if (error.statusCode === 429) {
        return new Response(
          'AI service is busy. Please try again in a moment.',
          { status: 429 }
        );
      }

      return new Response('AI service is temporarily unavailable.', {
        status: 503,
      });
    }

    // Unknown error
    console.error('Unexpected error:', error);
    return new Response('Internal server error', { status: 500 });
  }
}
```

### 12.4 Observability

You cannot improve what you cannot measure:

```typescript
const result = await streamText({
  model: openai('gpt-4.1'),
  messages,
  experimental_telemetry: {
    isEnabled: true,
    functionId: 'customer-support-chat',
    metadata: {
      userId: session.user.id,
      chatId,
      messageCount: messages.length,
    },
  },
  onFinish: async ({ text, usage, finishReason }) => {
    // Log to your analytics
    await analytics.track('ai_response', {
      userId: session.user.id,
      chatId,
      promptTokens: usage.promptTokens,
      completionTokens: usage.completionTokens,
      totalTokens: usage.totalTokens,
      finishReason,
      responseLength: text.length,
      model: 'gpt-4.1',
      estimatedCost:
        (usage.promptTokens * 0.01 + usage.completionTokens * 0.03) / 1000,
    });
  },
});
```

### 12.5 Security Considerations

This is where most teams get sloppy:

```typescript
// 1. NEVER expose API keys to the client
// AI calls must go through your backend route handlers

// 2. Sanitize user input
function sanitizeMessage(content: string): string {
  return content
    .slice(0, 10000) // Limit message length
    .replace(/\x00/g, ''); // Remove null bytes
}

// 3. Validate that tool results don't expose sensitive data
const getUserProfile = tool({
  description: 'Get basic user profile information',
  parameters: z.object({ userId: z.string() }),
  execute: async ({ userId }) => {
    const user = await db.users.findUnique({ where: { id: userId } });
    // Only return safe fields — never expose internal IDs, tokens, passwords
    return {
      name: user.name,
      plan: user.plan,
      memberSince: user.createdAt,
      // NOT: user.hashedPassword, user.apiKey, user.internalId
    };
  },
});

// 4. Limit what the model can do based on user permissions
function getToolsForUser(userRole: string) {
  const baseTools = { searchDocs, getOrderStatus };

  if (userRole === 'admin') {
    return { ...baseTools, updateOrder, issueRefund };
  }

  return baseTools;
}
```

### 12.6 Testing AI Features

AI is non-deterministic, which makes testing tricky. Here is what works:

```typescript
// 1. Mock the AI SDK in unit tests
import { vi } from 'vitest';

vi.mock('ai', () => ({
  streamText: vi.fn().mockReturnValue({
    toDataStreamResponse: () => new Response('mocked response'),
    text: 'mocked text',
    usage: { promptTokens: 10, completionTokens: 20 },
  }),
  generateObject: vi.fn().mockResolvedValue({
    object: {
      sentiment: 'positive',
      score: 8,
      summary: 'Test summary',
    },
  }),
}));

// 2. Test tool execution independently
describe('getOrderStatus tool', () => {
  it('returns order data for valid order ID', async () => {
    const result = await getOrderStatus.execute(
      { orderId: 'ORD-123' },
      { toolCallId: 'test', messages: [] }
    );
    expect(result).toHaveProperty('status');
    expect(result).toHaveProperty('estimatedDelivery');
  });

  it('returns error for invalid order ID', async () => {
    const result = await getOrderStatus.execute(
      { orderId: 'INVALID' },
      { toolCallId: 'test', messages: [] }
    );
    expect(result).toHaveProperty('error');
  });
});

// 3. Integration tests with low temperature for determinism
describe('classification pipeline', () => {
  it('classifies billing-related tickets correctly', async () => {
    const { object } = await generateObject({
      model: openai('gpt-4.1-mini'),
      schema: TicketClassificationSchema,
      prompt: 'I was charged twice for my subscription',
      temperature: 0, // More deterministic
    });

    expect(object.category).toBe('billing');
    expect(['medium', 'high']).toContain(object.priority);
  });
});
```

---

## 13. THE DECISION FRAMEWORK

When you are staring at a new AI feature request, use this framework:

```
Is it a multi-turn conversation?
  |
  +-- YES -> useChat
  |   |
  |   +-- Does it need tool calling?
  |   |   +-- YES -> Add tools + maxSteps
  |   |   +-- NO  -> Basic useChat
  |   |
  |   +-- Does the AI generate UI components?
  |       +-- YES -> RSC layer (streamUI)
  |       +-- NO  -> Standard useChat
  |
  +-- NO -> Single-turn generation
      |
      +-- Do you need structured data (JSON)?
      |   +-- YES -> generateObject / useObject
      |   +-- NO  -> useCompletion
      |
      +-- Is the output long (needs streaming)?
          +-- YES -> streamText / streamObject
          +-- NO  -> generateText / generateObject

Which model should I use?
  +-- Simple classification/extraction -> gpt-4.1-mini / claude-haiku
  +-- Complex reasoning/generation     -> gpt-4.1 / claude-sonnet
  +-- Long-form content / analysis     -> claude-sonnet / gpt-4.1
  +-- Code generation                  -> claude-sonnet / gpt-4.1

Do I need failover?
  +-- Production app with SLA    -> AI Gateway with multi-provider fallback
  +-- Internal tool              -> Single provider is fine
  +-- Cost-sensitive             -> Cascade from cheap to expensive models
```

---

## 14. REAL-WORLD ARCHITECTURES

### 14.1 E-Commerce: AI Shopping Assistant

```
User                    Next.js API Route          AI Provider
  |                           |                        |
  | "Find me a red dress      |                        |
  |  under $100"              |                        |
  | ------------------------->|                        |
  |                           |   streamText + tools   |
  |                           |----------------------->|
  |                           |                        |
  |                           |<-- tool call:          |
  |                           |    searchProducts({    |
  |                           |      query: "red dress"|
  |                           |      maxPrice: 100     |
  |                           |    })                  |
  |                           |                        |
  |                    [execute tool against DB]        |
  |                           |                        |
  |                           |-- tool result -------->|
  |                           |                        |
  |   <-- streaming response -|<-- "I found 3 red..." |
  |   + ProductCard components|                        |
  |                           |                        |
```

### 14.2 Developer Tool: AI Code Reviewer

```typescript
// A complete AI code review endpoint
export async function POST(req: Request) {
  const { diff, language, context } = await req.json();

  const { object } = await generateObject({
    model: anthropic('claude-sonnet-4-20250514'),
    schema: z.object({
      overallQuality: z.enum(['good', 'needs-work', 'critical-issues']),
      summary: z.string(),
      issues: z.array(
        z.object({
          severity: z.enum([
            'error',
            'warning',
            'suggestion',
            'nitpick',
          ]),
          line: z.number().optional(),
          file: z.string().optional(),
          description: z.string(),
          suggestedFix: z.string().optional(),
          category: z.enum([
            'bug',
            'security',
            'performance',
            'style',
            'logic',
            'testing',
          ]),
        })
      ),
      positives: z.array(z.string()),
      testingSuggestions: z.array(z.string()),
    }),
    prompt: `Review this ${language} code diff and provide detailed feedback:

Context: ${context}

Diff:
\`\`\`
${diff}
\`\`\`

Focus on bugs, security issues, and performance problems. Be specific about line numbers.`,
  });

  return Response.json(object);
}
```

### 14.3 SaaS Platform: AI Content Generator

```typescript
// Multi-step content generation pipeline
export async function generateBlogPost(topic: string, tone: string) {
  // Step 1: Generate outline (cheap model)
  const { object: outline } = await generateObject({
    model: openai('gpt-4.1-mini'),
    schema: z.object({
      title: z.string(),
      sections: z.array(
        z.object({
          heading: z.string(),
          keyPoints: z.array(z.string()),
        })
      ),
      estimatedWordCount: z.number(),
    }),
    prompt: `Create an outline for a blog post about "${topic}" in a ${tone} tone.`,
  });

  // Step 2: Generate full content (expensive model, streamed)
  const result = streamText({
    model: anthropic('claude-sonnet-4-20250514'),
    prompt: `Write a blog post based on this outline. Use a ${tone} tone.

Title: ${outline.title}

Outline:
${outline.sections
  .map(
    (s) =>
      `## ${s.heading}\n${s.keyPoints.map((p) => `- ${p}`).join('\n')}`
  )
  .join('\n\n')}

Write engaging, well-structured content. Use subheadings, bullet points, and examples where appropriate.`,
    maxTokens: 4000,
  });

  return result.toDataStreamResponse();
}
```

---

## 15. WHAT'S NEXT

The AI SDK is evolving fast. Here is where the ecosystem is heading:

**Multi-modal inputs**: Vision models let you send images alongside text. The AI SDK already supports this — you can send screenshots, photos, charts, and the model can reason about them. This is especially powerful in mobile apps where the camera is a first-class input.

**Voice**: Real-time voice interfaces are coming. The AI SDK is building toward streaming audio in and out, enabling conversational AI features that feel like talking to a person, not typing at a chatbot.

**Edge AI**: Running smaller models at the edge (Vercel Edge Functions, on-device with CoreML/ONNX) for latency-sensitive features like autocomplete and classification. The AI SDK's provider abstraction will make this a configuration change, not a rewrite.

**Agent orchestration**: Multi-agent systems where specialized agents collaborate — a planning agent, a research agent, a coding agent, a review agent — all coordinated through the AI SDK's tool and streaming primitives.

The key insight: **the AI SDK is not just a library. It is an abstraction layer that decouples your AI features from any specific provider, model, or deployment strategy.** Build on it, and you can swap models, add providers, change routing, and adopt new capabilities without touching your feature code.

That is the 100x move.

---

## CHAPTER SUMMARY

- The AI SDK has three layers: **Core** (provider-agnostic primitives), **UI** (React hooks), and **RSC** (streaming React components)
- **Streaming is non-negotiable** for AI features — use `streamText` and `toDataStreamResponse()`
- **useChat** manages multi-turn conversations; **useCompletion** handles single-turn generation
- **Tool calling** with Zod schemas lets models execute functions — this is what makes AI features useful, not just entertaining
- **Structured output** (`generateObject`) guarantees type-safe JSON responses matching your Zod schema
- **AI Gateway** provides multi-provider routing, failover, and cost tracking
- **Agents** are built with `maxSteps` + well-designed tools — tool design is 80% of agent quality
- **Rate limit, manage tokens, handle errors, and log everything** — production AI features need all of this
- The AI SDK abstracts away providers, so you can swap models without changing feature code

---

*Next chapter: [Chapter 27: Developer Experience & Tooling](/part-6-vercel-web/27-dx-tooling.md)*
