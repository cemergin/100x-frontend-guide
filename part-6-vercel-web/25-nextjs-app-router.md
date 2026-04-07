<!--
  CHAPTER: 28
  TITLE: Next.js App Router — The Complete Picture
  PART: VI — Vercel & the Web
  PREREQS: Chapter 24
  KEY_TOPICS: Server Components, Client Components, Server Actions, Route Handlers, Streaming, ISR, PPR, caching, middleware, layouts, templates, parallel routes
  DIFFICULTY: Intermediate → Advanced
  UPDATED: 2026-04-07
-->

# Chapter 25: Next.js App Router — The Complete Picture

---

<details>
<summary><strong>TL;DR</strong></summary>

- The App Router is not "Pages Router with a different folder structure"; it is an architecture change where components render on the server by default and ship zero JS to the client
- Server Components can directly access databases, file systems, and environment variables; use `'use client'` only for components that need interactivity (state, effects, event handlers)
- Data fetching happens directly in Server Components with `await` -- no `useEffect`, no loading states for initial data, no API routes to wire up
- The caching model uses `use cache`, `cacheLife`, and `cacheTag` directives for fine-grained control; Partial Prerendering (PPR) combines static shells with streaming dynamic content
- Layouts persist across navigations (no remount), templates remount on every navigation; parallel routes and intercepting routes enable advanced patterns like modals and split views

</details>

Let me be blunt with you: if you're still thinking about the App Router as "Pages Router but with a different folder structure," you're going to have a bad time. I've watched teams migrate to the App Router and fail — not because the API is hard, but because they never updated their mental model. They kept trying to write Pages Router code in `app/` and wondered why everything felt wrong.

The App Router is a fundamentally different way to build web applications. It's not a routing change. It's an architecture change. Server Components rewrite the contract between your React code and the browser. The caching model is completely different. Data fetching doesn't look anything like `getServerSideProps`. And once you internalize these shifts, you'll wonder how you ever built apps the old way.

This chapter is the complete picture. We're going deep on every major concept, with real code, real trade-offs, and real opinions. Buckle up.

---

## 1. The Mental Model Shift

### The Old World: Everything Ships to the Client

In the Pages Router — and in most React frameworks before it — every component you wrote ended up in a JavaScript bundle that got shipped to the browser. Sure, you had SSR, but that was just "render HTML on the server, then hydrate the same components on the client." Every `useState`, every `useEffect`, every utility function, every date library — all of it went to the browser.

The mental model was simple: **you write React, it runs in the browser, and SSR is an optimization for initial load.**

### The New World: Server by Default

The App Router inverts this. Components render on the server by default. They never ship JavaScript to the client. They can directly access your database, your file system, your environment variables. They can `await` async operations right in the component body.

```tsx
// app/dashboard/page.tsx
// This is a Server Component by default. Zero JS shipped to the client.
import { db } from '@/lib/db'

export default async function DashboardPage() {
  const metrics = await db.query('SELECT * FROM metrics WHERE date = CURRENT_DATE')

  return (
    <div>
      <h1>Dashboard</h1>
      <MetricsGrid data={metrics} />
    </div>
  )
}
```

No `useEffect`. No `useState`. No loading spinners for initial data. No API routes to wire up. The component fetches data and renders HTML. That HTML goes to the browser. Done.

This isn't SSR in the traditional sense. There's no hydration of this component. The browser never sees the code for `DashboardPage`. It sees the HTML output and nothing else.

### Why This Matters

Think about what this means at scale:

| Aspect | Pages Router | App Router (Server Components) |
|--------|-------------|-------------------------------|
| Bundle size | Every component adds to JS bundle | Server Components add zero JS |
| Data fetching | API routes + client fetch, or getServerSideProps | Direct DB/API access in components |
| Secrets | Must be in API routes only | Accessible in any Server Component |
| Third-party libraries | All ship to client | Server-only libs never reach client |
| Hydration cost | Full page hydration | Only Client Components hydrate |

A real example: Shopify's Hydrogen framework (built on React Server Components) reported **45% smaller JavaScript bundles** after migrating to this model. That's not a micro-optimization. That's a different class of performance.

### The Opt-In Model

You don't opt out of server rendering — you opt INTO client-side interactivity. This is the critical shift.

Need `useState`? Add `"use client"` at the top of the file.
Need `useEffect`? `"use client"`.
Need `onClick` handlers? `"use client"`.
Need an event listener? `"use client"`.

But here's the thing: **most of your components don't need any of that**. Most components in a typical app are just taking data and rendering UI. They're display components. And in the App Router, those stay on the server where they belong.

The rule of thumb I give teams: **Start every component as a Server Component. Only add `"use client"` when the compiler tells you that you need it — or when you genuinely need browser APIs.** Don't preemptively mark things as Client Components because you "might need state later."

---

## 2. Server Components vs. Client Components

### The Boundary

Here's the most important diagram you'll see in this chapter:

```
Server Component Tree
+-- ServerComponent (no JS shipped)
|   +-- ServerComponent (no JS shipped)
|   |   +-- ClientComponent ("use client" -- JS shipped)
|   |       +-- ClientComponent (inherits client boundary)
|   |       +-- ClientComponent (inherits client boundary)
|   +-- ServerComponent (no JS shipped)
|       +-- ClientComponent ("use client" -- JS shipped)
+-- ServerComponent (no JS shipped)
```

The rules:

1. **Server Components can render Client Components.** This is the normal pattern.
2. **Client Components cannot import Server Components.** This is the hard rule.
3. **Client Components can render Server Components passed as `children` or other props.** This is the escape hatch.

Let me show you each pattern.

### Pattern 1: Server Component Renders Client Component

```tsx
// app/products/page.tsx (Server Component)
import { db } from '@/lib/db'
import { ProductFilter } from './product-filter' // Client Component
import { ProductGrid } from './product-grid'     // Server Component

export default async function ProductsPage() {
  const products = await db.products.findMany()
  const categories = await db.categories.findMany()

  return (
    <div>
      <h1>Products</h1>
      <ProductFilter categories={categories} />
      <ProductGrid products={products} />
    </div>
  )
}
```

```tsx
// app/products/product-filter.tsx (Client Component)
'use client'

import { useState } from 'react'

interface Props {
  categories: { id: string; name: string }[]
}

export function ProductFilter({ categories }: Props) {
  const [selected, setSelected] = useState<string | null>(null)

  return (
    <div className="flex gap-2">
      {categories.map((cat) => (
        <button
          key={cat.id}
          onClick={() => setSelected(cat.id)}
          className={selected === cat.id ? 'bg-blue-500 text-white' : 'bg-gray-100'}
        >
          {cat.name}
        </button>
      ))}
    </div>
  )
}
```

```tsx
// app/products/product-grid.tsx (Server Component -- no "use client")
interface Props {
  products: { id: string; name: string; price: number }[]
}

export function ProductGrid({ products }: Props) {
  return (
    <div className="grid grid-cols-3 gap-4">
      {products.map((product) => (
        <div key={product.id} className="border p-4 rounded">
          <h3>{product.name}</h3>
          <p>${product.price}</p>
        </div>
      ))}
    </div>
  )
}
```

Notice: `ProductGrid` is a Server Component. It receives data as props, renders HTML, and ships zero JavaScript. `ProductFilter` needs `useState` for interactivity, so it's a Client Component. The data (`categories`) is serialized from the server and passed as props.

### Pattern 2: The Composition Pattern (Children Trick)

What if you have a Client Component that needs to wrap a Server Component? You can't import a Server Component inside a Client Component. But you can pass it as `children`:

```tsx
// app/layout.tsx (Server Component)
import { AuthProvider } from './auth-provider'   // Client Component
import { Sidebar } from './sidebar'               // Server Component

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        <AuthProvider>
          <div className="flex">
            <Sidebar />
            <main>{children}</main>
          </div>
        </AuthProvider>
      </body>
    </html>
  )
}
```

```tsx
// app/auth-provider.tsx (Client Component)
'use client'

import { createContext, useContext, useState } from 'react'

const AuthContext = createContext<{ user: any } | null>(null)

export function AuthProvider({ children }: { children: React.ReactNode }) {
  const [user, setUser] = useState(null)

  return (
    <AuthContext.Provider value={{ user }}>
      {children}
    </AuthContext.Provider>
  )
}

export function useAuth() {
  const context = useContext(AuthContext)
  if (!context) throw new Error('useAuth must be used within AuthProvider')
  return context
}
```

Here, `AuthProvider` is a Client Component, but `Sidebar` (a Server Component) is passed through it as children. This works because the Server Component is rendered on the server first, and the Client Component receives its output as a "slot" — it doesn't need to import or render the Server Component itself.

### The Serialization Boundary

When data crosses from Server Components to Client Components (via props), it must be serializable. This means:

**Can cross the boundary:**
- Strings, numbers, booleans, null, undefined
- Arrays and plain objects (containing serializable values)
- Date objects (serialized as strings)
- FormData
- TypedArrays, ArrayBuffer
- Map, Set
- Server Actions (functions marked with `"use server"`)

**Cannot cross the boundary:**
- Functions (except Server Actions)
- Classes/class instances
- Symbols
- DOM nodes
- Closures

```tsx
// THIS WILL BREAK
// app/page.tsx (Server Component)
export default function Page() {
  const handleClick = () => console.log('clicked') // Not serializable!

  return <ClientButton onClick={handleClick} /> // Error!
}
```

```tsx
// THIS WORKS
// app/page.tsx (Server Component)
import { trackClick } from './actions' // Server Action

export default function Page() {
  return <ClientButton onClick={trackClick} /> // Server Actions are serializable
}
```

### When to Use Each

Here's the decision framework I use:

| Need | Component Type | Why |
|------|---------------|-----|
| Fetch data from DB/API | Server Component | Direct access, no client bundle |
| Read filesystem / env vars | Server Component | Server-only APIs |
| Use heavy libraries (markdown, syntax highlighting) | Server Component | Don't ship to client |
| `useState`, `useReducer` | Client Component | State requires client JS |
| `useEffect`, `useRef` | Client Component | Lifecycle/DOM requires client JS |
| `onClick`, `onChange`, event handlers | Client Component | Browser events require client JS |
| Browser APIs (localStorage, geolocation) | Client Component | Only exist in browser |
| Third-party components that use hooks | Client Component | They internally need client |
| Display-only UI (cards, lists, grids) | Server Component | No interactivity needed |
| Forms with progressive enhancement | Server Component + Server Actions | Works without JS |

The ratio in a well-architected App Router app is typically **70-80% Server Components, 20-30% Client Components.** If your ratio is inverted, you're probably marking too many things as `"use client"`.

### The "use client" Infection Problem

Here's a pitfall I see constantly. A developer adds `"use client"` to one component, and it starts spreading:

```
ComponentA ("use client")
  +-- imports ComponentB --> ComponentB becomes client too
       +-- imports ComponentC --> ComponentC becomes client too
            +-- imports utility.ts --> utility.ts is now in client bundle
```

Once you cross the client boundary, everything below is client. This is why you should **push the `"use client"` boundary as far down the tree as possible.**

Bad:
```tsx
// app/dashboard/page.tsx
'use client' // DON'T DO THIS -- entire page is now client

import { useState } from 'react'
import { HeavyChart } from '@/components/heavy-chart'
import { DataTable } from '@/components/data-table'
// ... everything here ships to client
```

Good:
```tsx
// app/dashboard/page.tsx (Server Component)
import { db } from '@/lib/db'
import { InteractiveChart } from './interactive-chart' // only this is client
import { DataTable } from './data-table' // this stays server

export default async function DashboardPage() {
  const data = await db.metrics.findMany()

  return (
    <div>
      <InteractiveChart data={data} />
      <DataTable data={data} />
    </div>
  )
}
```

```tsx
// app/dashboard/interactive-chart.tsx
'use client' // Only this small component is client

import { useState } from 'react'

export function InteractiveChart({ data }: { data: Metric[] }) {
  const [range, setRange] = useState('7d')
  // chart rendering with interactivity
}
```

The Client Component should be as small and as leaf-level as possible. Extract the interactive bits into their own component. Keep the data-fetching and layout in Server Components.

---

## 3. Server Actions

### The End of "Create an API Route for That"

Remember the old workflow?

1. Create a form component
2. Create an API route at `/api/something`
3. Write fetch logic in the component
4. Handle loading/error states
5. Revalidate data somehow

Server Actions collapse all of this into a single pattern:

```tsx
// app/todos/page.tsx
import { db } from '@/lib/db'
import { revalidatePath } from 'next/cache'

export default async function TodosPage() {
  const todos = await db.todos.findMany({ orderBy: { createdAt: 'desc' } })

  async function addTodo(formData: FormData) {
    'use server'

    const title = formData.get('title') as string
    await db.todos.create({ data: { title } })
    revalidatePath('/todos')
  }

  return (
    <div>
      <h1>Todos</h1>

      <form action={addTodo}>
        <input name="title" placeholder="What needs to be done?" required />
        <button type="submit">Add</button>
      </form>

      <ul>
        {todos.map((todo) => (
          <li key={todo.id}>{todo.title}</li>
        ))}
      </ul>
    </div>
  )
}
```

That `addTodo` function runs on the server. The `"use server"` directive tells Next.js to create an HTTP endpoint for it automatically. The form submits to that endpoint. The page revalidates. No client JavaScript required for the basic flow.

### How Server Actions Work Under the Hood

When you write a Server Action, Next.js:

1. Generates a unique ID for the action
2. Creates an HTTP POST endpoint mapped to that ID
3. On the client, the form's `action` attribute points to this endpoint
4. When submitted, the form data is sent as `multipart/form-data`
5. The server executes the function, then returns the result
6. The client re-renders with fresh server data

This means **Server Actions work even with JavaScript disabled.** The form submits as a standard HTML form. This is progressive enhancement done right.

### Separate Action Files

For larger applications, you'll want to organize actions in separate files:

```tsx
// app/todos/actions.ts
'use server'

import { db } from '@/lib/db'
import { revalidatePath } from 'next/cache'
import { z } from 'zod'

const TodoSchema = z.object({
  title: z.string().min(1).max(200),
})

export async function addTodo(formData: FormData) {
  const parsed = TodoSchema.safeParse({
    title: formData.get('title'),
  })

  if (!parsed.success) {
    return { error: parsed.error.flatten().fieldErrors }
  }

  await db.todos.create({
    data: { title: parsed.data.title },
  })

  revalidatePath('/todos')
  return { success: true }
}

export async function deleteTodo(id: string) {
  await db.todos.delete({ where: { id } })
  revalidatePath('/todos')
}

export async function toggleTodo(id: string) {
  const todo = await db.todos.findUnique({ where: { id } })
  if (!todo) return { error: 'Todo not found' }

  await db.todos.update({
    where: { id },
    data: { completed: !todo.completed },
  })

  revalidatePath('/todos')
}
```

### Using Server Actions from Client Components

Server Actions aren't just for forms. You can call them from Client Components like any async function:

```tsx
// app/todos/todo-item.tsx
'use client'

import { useTransition } from 'react'
import { toggleTodo, deleteTodo } from './actions'

interface Props {
  todo: { id: string; title: string; completed: boolean }
}

export function TodoItem({ todo }: Props) {
  const [isPending, startTransition] = useTransition()

  return (
    <li className={isPending ? 'opacity-50' : ''}>
      <input
        type="checkbox"
        checked={todo.completed}
        onChange={() => startTransition(() => toggleTodo(todo.id))}
      />
      <span className={todo.completed ? 'line-through' : ''}>{todo.title}</span>
      <button
        onClick={() => startTransition(() => deleteTodo(todo.id))}
        disabled={isPending}
      >
        Delete
      </button>
    </li>
  )
}
```

`useTransition` gives you a pending state while the action executes. The UI stays responsive, and the update is non-blocking.

### The `useActionState` Pattern

For forms that need validation feedback, `useActionState` is the go-to:

```tsx
// app/auth/login/page.tsx
'use client'

import { useActionState } from 'react'
import { login } from './actions'

export default function LoginPage() {
  const [state, formAction, isPending] = useActionState(login, {
    error: null,
  })

  return (
    <form action={formAction}>
      <div>
        <label htmlFor="email">Email</label>
        <input id="email" name="email" type="email" required />
      </div>

      <div>
        <label htmlFor="password">Password</label>
        <input id="password" name="password" type="password" required />
      </div>

      {state.error && (
        <p className="text-red-500">{state.error}</p>
      )}

      <button type="submit" disabled={isPending}>
        {isPending ? 'Logging in...' : 'Log in'}
      </button>
    </form>
  )
}
```

```tsx
// app/auth/login/actions.ts
'use server'

import { redirect } from 'next/navigation'
import { createSession, verifyCredentials } from '@/lib/auth'

export async function login(prevState: any, formData: FormData) {
  const email = formData.get('email') as string
  const password = formData.get('password') as string

  const user = await verifyCredentials(email, password)

  if (!user) {
    return { error: 'Invalid email or password' }
  }

  await createSession(user.id)
  redirect('/dashboard')
}
```

### Revalidation Patterns

Server Actions pair with two revalidation strategies:

**`revalidatePath`** — Revalidate everything associated with a URL path:

```tsx
'use server'
import { revalidatePath } from 'next/cache'

export async function updateProfile(formData: FormData) {
  await db.users.update(/* ... */)
  revalidatePath('/profile')           // Revalidate the profile page
  revalidatePath('/profile', 'layout') // Revalidate the layout too
  revalidatePath('/', 'layout')        // Revalidate everything (nuclear option)
}
```

**`revalidateTag`** — Revalidate all cache entries tagged with a specific string:

```tsx
'use server'
import { revalidateTag } from 'next/cache'

export async function publishPost(id: string) {
  await db.posts.update({ where: { id }, data: { published: true } })
  revalidateTag('posts')       // All cache entries tagged 'posts' are invalidated
  revalidateTag(`post-${id}`)  // This specific post's cache entry too
}
```

Tags are the more surgical approach. We'll cover how to tag cache entries in section 7.

### Error Handling in Server Actions

Always validate input. Always handle errors. Server Actions are HTTP endpoints — treat them like it:

```tsx
// app/actions/create-order.ts
'use server'

import { z } from 'zod'
import { auth } from '@/lib/auth'
import { db } from '@/lib/db'
import { revalidateTag } from 'next/cache'

const OrderSchema = z.object({
  productId: z.string().uuid(),
  quantity: z.number().int().min(1).max(99),
})

export async function createOrder(rawData: { productId: string; quantity: number }) {
  // 1. Auth check
  const session = await auth()
  if (!session) {
    throw new Error('Unauthorized')
  }

  // 2. Input validation
  const parsed = OrderSchema.safeParse(rawData)
  if (!parsed.success) {
    return {
      success: false as const,
      errors: parsed.error.flatten().fieldErrors,
    }
  }

  // 3. Business logic
  const product = await db.products.findUnique({
    where: { id: parsed.data.productId },
  })

  if (!product) {
    return { success: false as const, errors: { productId: ['Product not found'] } }
  }

  if (product.stock < parsed.data.quantity) {
    return { success: false as const, errors: { quantity: ['Insufficient stock'] } }
  }

  // 4. Mutation
  try {
    const order = await db.orders.create({
      data: {
        userId: session.user.id,
        productId: parsed.data.productId,
        quantity: parsed.data.quantity,
        total: product.price * parsed.data.quantity,
      },
    })

    revalidateTag('orders')
    revalidateTag(`product-${product.id}`)

    return { success: true as const, orderId: order.id }
  } catch (error) {
    console.error('Order creation failed:', error)
    return {
      success: false as const,
      errors: { _form: ['Something went wrong. Please try again.'] },
    }
  }
}
```

Notice: we return error objects instead of throwing (except for auth, where we want to hard-fail). This lets the client handle validation errors gracefully.

---

## 4. Route Handlers

### When You Still Need API Endpoints

Server Actions handle mutations beautifully, but sometimes you need a traditional API endpoint:

- **Webhooks** from third-party services (Stripe, GitHub, etc.)
- **Public APIs** consumed by mobile apps or external services
- **File downloads** or streaming responses
- **Integration with non-React clients**
- **Long-polling** or server-sent events

Route Handlers give you the Web Standard `Request`/`Response` API:

```tsx
// app/api/webhooks/stripe/route.ts
import { headers } from 'next/headers'
import Stripe from 'stripe'

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!)

export async function POST(request: Request) {
  const body = await request.text()
  const headersList = await headers()
  const signature = headersList.get('stripe-signature')!

  let event: Stripe.Event

  try {
    event = stripe.webhooks.constructEvent(
      body,
      signature,
      process.env.STRIPE_WEBHOOK_SECRET!
    )
  } catch (err) {
    return new Response('Invalid signature', { status: 400 })
  }

  switch (event.type) {
    case 'checkout.session.completed':
      await handleCheckoutCompleted(event.data.object)
      break
    case 'customer.subscription.deleted':
      await handleSubscriptionCanceled(event.data.object)
      break
  }

  return new Response('OK', { status: 200 })
}
```

### HTTP Methods

Route Handlers support all standard HTTP methods:

```tsx
// app/api/posts/route.ts
import { db } from '@/lib/db'
import { NextResponse } from 'next/server'
import { auth } from '@/lib/auth'

// GET /api/posts
export async function GET(request: Request) {
  const { searchParams } = new URL(request.url)
  const page = parseInt(searchParams.get('page') ?? '1')
  const limit = 20

  const posts = await db.posts.findMany({
    skip: (page - 1) * limit,
    take: limit,
    orderBy: { createdAt: 'desc' },
  })

  return NextResponse.json({ posts, page })
}

// POST /api/posts
export async function POST(request: Request) {
  const session = await auth()
  if (!session) {
    return new Response('Unauthorized', { status: 401 })
  }

  const body = await request.json()
  const post = await db.posts.create({
    data: {
      title: body.title,
      content: body.content,
      authorId: session.user.id,
    },
  })

  return NextResponse.json(post, { status: 201 })
}
```

### Dynamic Route Handlers

For routes with parameters, use the standard `[param]` syntax:

```tsx
// app/api/posts/[id]/route.ts
import { db } from '@/lib/db'
import { NextResponse } from 'next/server'

export async function GET(
  request: Request,
  { params }: { params: Promise<{ id: string }> }
) {
  const { id } = await params

  const post = await db.posts.findUnique({ where: { id } })

  if (!post) {
    return new Response('Not found', { status: 404 })
  }

  return NextResponse.json(post)
}

export async function PATCH(
  request: Request,
  { params }: { params: Promise<{ id: string }> }
) {
  const { id } = await params
  const body = await request.json()

  const post = await db.posts.update({
    where: { id },
    data: body,
  })

  return NextResponse.json(post)
}

export async function DELETE(
  request: Request,
  { params }: { params: Promise<{ id: string }> }
) {
  const { id } = await params
  await db.posts.delete({ where: { id } })
  return new Response(null, { status: 204 })
}
```

### Static vs. Dynamic Route Handlers

By default, a `GET` Route Handler with no dynamic input is **statically evaluated at build time**:

```tsx
// This is STATIC -- built once, served from cache
export async function GET() {
  const data = await fetchSomething()
  return NextResponse.json(data)
}
```

It becomes **dynamic** when you:
- Read `request` object (URL, headers, cookies)
- Use dynamic functions (`headers()`, `cookies()`)
- Use a dynamic route segment (`[id]`)
- Export `const dynamic = 'force-dynamic'`

```tsx
// This is DYNAMIC -- runs on every request
export async function GET(request: Request) {
  const { searchParams } = new URL(request.url)
  // Reading request makes it dynamic
  return NextResponse.json({ query: searchParams.get('q') })
}
```

### Streaming Responses

Route Handlers support streaming, which is great for AI/LLM responses or large datasets:

```tsx
// app/api/chat/route.ts
import { openai } from '@/lib/openai'

export async function POST(request: Request) {
  const { messages } = await request.json()

  const stream = await openai.chat.completions.create({
    model: 'gpt-4o',
    messages,
    stream: true,
  })

  const encoder = new TextEncoder()

  const readableStream = new ReadableStream({
    async start(controller) {
      for await (const chunk of stream) {
        const text = chunk.choices[0]?.delta?.content ?? ''
        controller.enqueue(encoder.encode(text))
      }
      controller.close()
    },
  })

  return new Response(readableStream, {
    headers: { 'Content-Type': 'text/plain; charset=utf-8' },
  })
}
```

---

## 5. Layouts and Templates

### The Layout System

Layouts are one of the App Router's best features. A layout wraps all pages in its directory and below, and it **persists across navigations.** State is preserved. The component doesn't unmount.

```
app/
+-- layout.tsx          <-- Root layout (required)
+-- page.tsx            <-- Home page
+-- dashboard/
    +-- layout.tsx      <-- Dashboard layout (persists across dashboard pages)
    +-- page.tsx        <-- /dashboard
    +-- analytics/
    |   +-- page.tsx    <-- /dashboard/analytics (wrapped by dashboard layout)
    +-- settings/
        +-- page.tsx    <-- /dashboard/settings (wrapped by dashboard layout)
```

```tsx
// app/layout.tsx -- Root Layout
import { Inter } from 'next/font/google'
import './globals.css'

const inter = Inter({ subsets: ['latin'] })

export const metadata = {
  title: 'My Application',
  description: 'Built with Next.js App Router',
}

export default function RootLayout({
  children,
}: {
  children: React.ReactNode
}) {
  return (
    <html lang="en">
      <body className={inter.className}>
        <header>
          <nav>{/* Global navigation */}</nav>
        </header>
        <main>{children}</main>
        <footer>{/* Global footer */}</footer>
      </body>
    </html>
  )
}
```

```tsx
// app/dashboard/layout.tsx -- Dashboard Layout
import { auth } from '@/lib/auth'
import { redirect } from 'next/navigation'
import { DashboardSidebar } from './sidebar'

export default async function DashboardLayout({
  children,
}: {
  children: React.ReactNode
}) {
  const session = await auth()
  if (!session) redirect('/login')

  return (
    <div className="flex">
      <DashboardSidebar user={session.user} />
      <div className="flex-1 p-6">{children}</div>
    </div>
  )
}
```

Key behaviors:
- The root layout **must** contain `<html>` and `<body>` tags
- Layouts are **Server Components** by default (they can be Client Components, but usually shouldn't be)
- When navigating between `/dashboard/analytics` and `/dashboard/settings`, the `DashboardLayout` doesn't re-render or re-mount
- This means sidebar state, scroll position, and any other local state in the layout is preserved
- Layouts **cannot** access the current pathname (use `usePathname()` in a Client Component if needed)

### Nested Layouts

Layouts compose naturally. Each segment can have its own layout:

```tsx
// app/dashboard/settings/layout.tsx
import { SettingsTabs } from './tabs'

export default function SettingsLayout({
  children,
}: {
  children: React.ReactNode
}) {
  return (
    <div>
      <h2 className="text-2xl font-bold mb-4">Settings</h2>
      <SettingsTabs />
      <div className="mt-4">{children}</div>
    </div>
  )
}
```

Now `/dashboard/settings/profile` renders inside:
1. Root Layout
2. Dashboard Layout
3. Settings Layout
4. Profile Page

### Templates: When You Want Re-mounting

Templates are like layouts but they **create a new instance for each navigation.** State is reset. Effects re-fire. The DOM is re-created.

```tsx
// app/dashboard/template.tsx
export default function DashboardTemplate({
  children,
}: {
  children: React.ReactNode
}) {
  return (
    <div>
      {children}
    </div>
  )
}
```

Use templates when you need:
- **Page transition animations** (each page mounts/unmounts)
- **Per-page analytics tracking** via `useEffect`
- **Reset state** on each navigation
- **Fresh Suspense boundaries** for each page

| Feature | Layout | Template |
|---------|--------|----------|
| Persists across navigations | Yes | No |
| State preserved | Yes | No |
| Effects re-run | No | Yes |
| Re-renders on navigation | No (only children change) | Yes (full re-mount) |
| Use case | Shared UI (nav, sidebar) | Animations, per-page effects |

### Parallel Routes

Parallel routes let you render multiple pages in the same layout simultaneously. This is powerful for dashboards, split views, or modal patterns:

```
app/
+-- dashboard/
    +-- layout.tsx
    +-- page.tsx
    +-- @analytics/
    |   +-- page.tsx
    |   +-- loading.tsx
    +-- @activity/
    |   +-- page.tsx
    |   +-- loading.tsx
    +-- @notifications/
        +-- page.tsx
        +-- default.tsx
```

```tsx
// app/dashboard/layout.tsx
export default function DashboardLayout({
  children,
  analytics,
  activity,
  notifications,
}: {
  children: React.ReactNode
  analytics: React.ReactNode
  activity: React.ReactNode
  notifications: React.ReactNode
}) {
  return (
    <div>
      <div>{children}</div>
      <div className="grid grid-cols-2 gap-4">
        <div>{analytics}</div>
        <div>{activity}</div>
      </div>
      <aside>{notifications}</aside>
    </div>
  )
}
```

Each `@slot` loads independently and can have its own loading state, error boundary, and data fetching. They stream in as they're ready. This is how you build dashboards that feel fast even when individual widgets are slow.

### Intercepting Routes

Intercepting routes let you "catch" a navigation in the current layout instead of doing a full page navigation. The classic example: clicking a photo opens it in a modal (intercepted), but refreshing the page or sharing the URL loads the full photo page.

```
app/
+-- feed/
|   +-- page.tsx
|   +-- @modal/
|   |   +-- (..)photo/[id]/
|   |   |   +-- page.tsx    <-- Intercepted: renders in modal
|   |   +-- default.tsx
|   +-- layout.tsx
+-- photo/
    +-- [id]/
        +-- page.tsx         <-- Direct navigation: full page
```

```tsx
// app/feed/@modal/(..)/photo/[id]/page.tsx
import { Modal } from '@/components/modal'
import { getPhoto } from '@/lib/photos'

export default async function PhotoModal({
  params,
}: {
  params: Promise<{ id: string }>
}) {
  const { id } = await params
  const photo = await getPhoto(id)

  return (
    <Modal>
      <img src={photo.url} alt={photo.alt} />
      <p>{photo.description}</p>
    </Modal>
  )
}
```

Instagram uses exactly this pattern. Click a post in your feed, it opens in a modal overlay. Refresh the page or share the URL, and you get the full post page. Same URL, different presentation based on navigation context.

---

## 6. Streaming and Suspense

### Why Streaming Matters

Traditional SSR works like this:

1. Server receives request
2. Server fetches ALL data
3. Server renders ALL HTML
4. Server sends complete HTML to client
5. Client displays page

The problem: step 2 blocks everything. If you have one slow database query, the entire page waits. TTFB (Time to First Byte) suffers.

Streaming SSR works like this:

1. Server receives request
2. Server starts rendering immediately
3. Server sends HTML for parts that are ready (shell, static content)
4. Slow parts stream in as they complete
5. Client progressively displays content

The user sees the page shell almost instantly, and dynamic content fills in as it's ready.

### `loading.tsx` -- The Easy Way

The simplest way to add streaming is with `loading.tsx`:

```
app/
+-- dashboard/
    +-- page.tsx
    +-- loading.tsx
```

```tsx
// app/dashboard/loading.tsx
export default function DashboardLoading() {
  return (
    <div className="animate-pulse">
      <div className="h-8 w-48 bg-gray-200 rounded mb-4" />
      <div className="grid grid-cols-3 gap-4">
        <div className="h-32 bg-gray-200 rounded" />
        <div className="h-32 bg-gray-200 rounded" />
        <div className="h-32 bg-gray-200 rounded" />
      </div>
    </div>
  )
}
```

Next.js automatically wraps the page in a `<Suspense>` boundary with your loading component as the fallback. When the page's async data resolves, the loading skeleton is replaced with the real content.

### Manual `<Suspense>` -- The Precise Way

For more control, use `<Suspense>` boundaries directly:

```tsx
// app/dashboard/page.tsx
import { Suspense } from 'react'
import { RevenueChart } from './revenue-chart'
import { RecentOrders } from './recent-orders'
import { TopProducts } from './top-products'

export default function DashboardPage() {
  return (
    <div>
      <h1>Dashboard</h1>

      {/* These three sections load independently */}
      <div className="grid grid-cols-2 gap-6">
        <Suspense fallback={<ChartSkeleton />}>
          <RevenueChart />
        </Suspense>

        <Suspense fallback={<OrdersSkeleton />}>
          <RecentOrders />
        </Suspense>
      </div>

      <Suspense fallback={<ProductsSkeleton />}>
        <TopProducts />
      </Suspense>
    </div>
  )
}
```

```tsx
// app/dashboard/revenue-chart.tsx (Server Component)
import { db } from '@/lib/db'

export async function RevenueChart() {
  // This might take 2 seconds -- but it doesn't block the rest of the page
  const revenue = await db.revenue.findMany({
    where: { date: { gte: subDays(new Date(), 30) } },
    orderBy: { date: 'asc' },
  })

  return (
    <div className="border rounded p-4">
      <h2>Revenue (Last 30 Days)</h2>
      {/* Render chart */}
    </div>
  )
}
```

Each `<Suspense>` boundary streams independently. If `RecentOrders` resolves in 200ms and `RevenueChart` takes 2 seconds, the orders appear first. The chart streams in later.

### Streaming Architecture

Here's how a well-designed streaming page works:

```
Request arrives
    |
    v
+-------------------------------------+
| Static Shell (immediate)             |
| +----------------------------------+ |
| | Navigation / Header              | |  <-- Sent immediately
| +----------------------------------+ |
| +-------------+ +-----------------+  |
| | Skeleton    | |  Skeleton       |  |  <-- Sent immediately (fallbacks)
| | (Chart)     | |  (Orders)       |  |
| +-------------+ +-----------------+  |
| +----------------------------------+ |
| | Skeleton (Products)              | |  <-- Sent immediately (fallback)
| +----------------------------------+ |
+-------------------------------------+

...200ms later...
  Orders resolve --> Real orders streamed in, replacing skeleton

...800ms later...
  Products resolve --> Real products streamed in, replacing skeleton

...2000ms later...
  Chart resolves --> Real chart streamed in, replacing skeleton
```

The user sees a complete (skeleton) page in ~50ms. Content fills in progressively. Compare that to waiting 2 seconds for everything.

### Nested Suspense Boundaries

You can nest Suspense boundaries for fine-grained control:

```tsx
<Suspense fallback={<PageSkeleton />}>
  <UserProfile userId={id}>
    <Suspense fallback={<PostsSkeleton />}>
      <UserPosts userId={id} />
    </Suspense>
    <Suspense fallback={<FriendsSkeleton />}>
      <UserFriends userId={id} />
    </Suspense>
  </UserProfile>
</Suspense>
```

If `UserProfile` is still loading, you see `PageSkeleton`. Once it resolves, you see the profile with `PostsSkeleton` and `FriendsSkeleton` inside. Then those resolve independently.

### Error Boundaries with `error.tsx`

Just like `loading.tsx` provides automatic Suspense, `error.tsx` provides automatic error boundaries:

```tsx
// app/dashboard/error.tsx
'use client' // Error components must be Client Components

import { useEffect } from 'react'

export default function DashboardError({
  error,
  reset,
}: {
  error: Error & { digest?: string }
  reset: () => void
}) {
  useEffect(() => {
    // Log to your error reporting service
    console.error(error)
  }, [error])

  return (
    <div className="p-6 text-center">
      <h2 className="text-xl font-bold">Something went wrong</h2>
      <p className="text-gray-600 mt-2">
        We couldn't load the dashboard. Please try again.
      </p>
      <button
        onClick={reset}
        className="mt-4 px-4 py-2 bg-blue-500 text-white rounded"
      >
        Try again
      </button>
    </div>
  )
}
```

The `reset` function re-renders the error boundary's content, giving the user a retry mechanism without a full page reload.

### `not-found.tsx`

Handle 404 states gracefully with `not-found.tsx`:

```tsx
// app/posts/[slug]/page.tsx
import { notFound } from 'next/navigation'
import { db } from '@/lib/db'

export default async function PostPage({
  params,
}: {
  params: Promise<{ slug: string }>
}) {
  const { slug } = await params
  const post = await db.posts.findUnique({ where: { slug } })

  if (!post) {
    notFound() // Renders the nearest not-found.tsx
  }

  return <article>{/* render post */}</article>
}
```

```tsx
// app/posts/[slug]/not-found.tsx
import Link from 'next/link'

export default function PostNotFound() {
  return (
    <div className="text-center py-12">
      <h2 className="text-2xl font-bold">Post Not Found</h2>
      <p className="text-gray-600 mt-2">
        The post you're looking for doesn't exist or has been removed.
      </p>
      <Link href="/posts" className="text-blue-500 mt-4 inline-block">
        Back to all posts
      </Link>
    </div>
  )
}
```

---

## 7. Data Fetching and Caching

### The Cache Model: A Fresh Start

Next.js has gone through several iterations of its caching model. The current approach (as of Next.js 15+) centers on the `use cache` directive, `cacheLife`, `cacheTag`, and `revalidateTag`. If you learned the older `unstable_cache` API or the `fetch` cache options, the new model is simpler and more predictable.

### The `use cache` Directive

The `use cache` directive marks a function or component as cacheable:

```tsx
// app/products/page.tsx
import { cacheLife, cacheTag } from 'next/cache'

async function getProducts() {
  'use cache'
  cacheLife('hours')
  cacheTag('products')

  const products = await db.products.findMany({
    where: { published: true },
    orderBy: { createdAt: 'desc' },
  })

  return products
}

export default async function ProductsPage() {
  const products = await getProducts()

  return (
    <div>
      <h1>Products</h1>
      {products.map((p) => (
        <ProductCard key={p.id} product={p} />
      ))}
    </div>
  )
}
```

Key points about `use cache`:
- It can be placed at the top of an async function or an async component
- The function's arguments become the cache key (they must be serializable)
- The cached result is stored and reused according to the `cacheLife` profile
- Tags allow targeted invalidation via `revalidateTag`

### `cacheLife` -- Controlling TTL

`cacheLife` sets how long cached data lives:

```tsx
import { cacheLife } from 'next/cache'

async function getPopularPosts() {
  'use cache'
  cacheLife('minutes') // Revalidate every few minutes

  return db.posts.findMany({
    orderBy: { views: 'desc' },
    take: 10,
  })
}

async function getStaticContent() {
  'use cache'
  cacheLife('max') // Cache as long as possible

  return db.content.findFirst({ where: { slug: 'about' } })
}

async function getUserNotifications(userId: string) {
  'use cache'
  cacheLife('seconds') // Very short cache for frequently changing data

  return db.notifications.findMany({
    where: { userId, read: false },
  })
}
```

Built-in profiles:

| Profile | Stale | Revalidate | Expire |
|---------|-------|------------|--------|
| `'seconds'` | 0 | 1s | 60s |
| `'minutes'` | 5min | 1min | 1h |
| `'hours'` | 5min | 1h | 1d |
| `'days'` | 5min | 1d | 1w |
| `'weeks'` | 5min | 1w | 30d |
| `'max'` | 5min | 30d | ~forever |

You can also define custom profiles in `next.config.ts`:

```tsx
// next.config.ts
import type { NextConfig } from 'next'

const nextConfig: NextConfig = {
  experimental: {
    dynamicIO: true,
    cacheLife: {
      'product-listing': {
        stale: 300,     // 5 minutes
        revalidate: 900, // 15 minutes
        expire: 86400,  // 1 day
      },
      'user-profile': {
        stale: 0,
        revalidate: 60,
        expire: 3600,
      },
    },
  },
}

export default nextConfig
```

```tsx
async function getProductListing(categoryId: string) {
  'use cache'
  cacheLife('product-listing') // Uses custom profile

  return db.products.findMany({ where: { categoryId } })
}
```

### `cacheTag` and `revalidateTag` -- Surgical Invalidation

Tags let you invalidate specific cache entries without path-based revalidation:

```tsx
// Tagging cached data
async function getProduct(id: string) {
  'use cache'
  cacheTag(`product-${id}`, 'products')
  cacheLife('hours')

  return db.products.findUnique({ where: { id } })
}

async function getProductReviews(productId: string) {
  'use cache'
  cacheTag(`product-${productId}-reviews`, 'reviews')
  cacheLife('minutes')

  return db.reviews.findMany({ where: { productId } })
}
```

```tsx
// Invalidating cached data (in a Server Action)
'use server'
import { revalidateTag } from 'next/cache'

export async function updateProduct(id: string, data: ProductUpdate) {
  await db.products.update({ where: { id }, data })

  // Invalidate this specific product's cache
  revalidateTag(`product-${id}`)
  // Also invalidate the product listing cache
  revalidateTag('products')
}

export async function addReview(productId: string, review: ReviewInput) {
  await db.reviews.create({ data: { ...review, productId } })

  // Invalidate reviews for this product
  revalidateTag(`product-${productId}-reviews`)
}
```

The pattern is clear: **tag your cached data with descriptive strings, then invalidate those tags when the data changes.**

### Migration from `unstable_cache`

If you're coming from the older API:

```tsx
// OLD: unstable_cache
import { unstable_cache } from 'next/cache'

const getCachedProducts = unstable_cache(
  async () => {
    return db.products.findMany()
  },
  ['products'],
  { revalidate: 3600, tags: ['products'] }
)

// NEW: use cache
async function getProducts() {
  'use cache'
  cacheLife('hours')
  cacheTag('products')

  return db.products.findMany()
}
```

The new API is more intuitive -- the caching configuration lives inside the function, not wrapped around it.

### When NOT to Cache

Not everything should be cached. Here's my rule:

- **Cache:** Product listings, blog posts, static pages, aggregated analytics, configuration
- **Don't cache:** User-specific data with real-time requirements, shopping cart contents, live notifications, data that changes on every request
- **Short cache:** Search results, user dashboards, social feeds

```tsx
// Don't cache -- user-specific, frequently changing
async function getCurrentUserCart(userId: string) {
  // No 'use cache' directive -- always fresh
  return db.carts.findUnique({
    where: { userId },
    include: { items: true },
  })
}

// Cache with tag -- changes on known events
async function getProductCatalog() {
  'use cache'
  cacheTag('catalog')
  cacheLife('hours')

  return db.products.findMany({ where: { active: true } })
}
```

---

## 8. ISR and PPR

### Incremental Static Regeneration (ISR)

ISR lets you build static pages at build time and update them incrementally without a full rebuild. This is the sweet spot between fully static (fast but stale) and fully dynamic (fresh but slow).

```tsx
// app/blog/[slug]/page.tsx

// Generate static pages for the most popular posts at build time
export async function generateStaticParams() {
  const posts = await db.posts.findMany({
    where: { published: true },
    orderBy: { views: 'desc' },
    take: 100,
  })

  return posts.map((post) => ({ slug: post.slug }))
}

export default async function BlogPost({
  params,
}: {
  params: Promise<{ slug: string }>
}) {
  const { slug } = await params
  const post = await getPost(slug) // Uses 'use cache' with cacheLife('hours')

  if (!post) notFound()

  return (
    <article>
      <h1>{post.title}</h1>
      <time>{post.publishedAt.toLocaleDateString()}</time>
      <div>{post.htmlContent}</div>
    </article>
  )
}

// The cached getPost function
async function getPost(slug: string) {
  'use cache'
  cacheLife('hours')
  cacheTag(`post-${slug}`)

  return db.posts.findUnique({ where: { slug, published: true } })
}
```

With this setup:
- The 100 most popular posts are built at deploy time (fast for users, no cold start)
- Other posts are generated on-demand and cached
- The cache revalidates on the schedule set by `cacheLife`
- You can force-revalidate with `revalidateTag('post-my-slug')` when content is edited

### On-Demand Revalidation with Webhooks

The real power of ISR comes from on-demand revalidation. When a CMS editor publishes content, you don't wait for the TTL -- you invalidate immediately:

```tsx
// app/api/revalidate/route.ts
import { revalidateTag } from 'next/cache'

export async function POST(request: Request) {
  const secret = request.headers.get('x-revalidation-secret')

  if (secret !== process.env.REVALIDATION_SECRET) {
    return new Response('Unauthorized', { status: 401 })
  }

  const body = await request.json()

  // CMS sends: { type: 'post', slug: 'my-new-post' }
  switch (body.type) {
    case 'post':
      revalidateTag(`post-${body.slug}`)
      revalidateTag('posts') // Also revalidate the listing
      break
    case 'product':
      revalidateTag(`product-${body.id}`)
      revalidateTag('products')
      break
  }

  return new Response('Revalidated', { status: 200 })
}
```

### Partial Prerendering (PPR)

PPR is where things get really interesting. It's the evolution of how Next.js handles the static/dynamic spectrum, and it solves a fundamental problem: **most pages are mostly static with a few dynamic parts.**

Think about a product page:
- **Static:** Product name, description, images, specifications
- **Dynamic:** Price (may vary by region), inventory status, user-specific recommendations, reviews count

Before PPR, you had two choices:
1. Make it fully static (stale dynamic data)
2. Make it fully dynamic (slow initial load)

PPR gives you a third option: **prerender the static shell, stream in the dynamic parts.**

```tsx
// next.config.ts
import type { NextConfig } from 'next'

const nextConfig: NextConfig = {
  experimental: {
    ppr: true,
  },
}

export default nextConfig
```

```tsx
// app/products/[id]/page.tsx
import { Suspense } from 'react'
import { getProduct } from '@/lib/products'
import { ProductPrice } from './product-price'
import { InventoryStatus } from './inventory-status'
import { Recommendations } from './recommendations'

export default async function ProductPage({
  params,
}: {
  params: Promise<{ id: string }>
}) {
  const { id } = await params
  const product = await getProduct(id) // Cached -- part of static shell

  return (
    <div>
      {/* STATIC SHELL -- prerendered at build time */}
      <h1>{product.name}</h1>
      <img src={product.image} alt={product.name} />
      <p>{product.description}</p>

      {/* DYNAMIC -- streamed at request time */}
      <Suspense fallback={<PriceSkeleton />}>
        <ProductPrice productId={id} />
      </Suspense>

      <Suspense fallback={<StatusSkeleton />}>
        <InventoryStatus productId={id} />
      </Suspense>

      <Suspense fallback={<RecommendationsSkeleton />}>
        <Recommendations productId={id} />
      </Suspense>
    </div>
  )
}
```

```tsx
// app/products/[id]/product-price.tsx
import { cookies } from 'next/headers'

export async function ProductPrice({ productId }: { productId: string }) {
  // Reading cookies makes this dynamic -- it can't be prerendered
  const cookieStore = await cookies()
  const region = cookieStore.get('region')?.value ?? 'us'

  const price = await getPriceForRegion(productId, region)

  return (
    <div className="text-2xl font-bold">
      {formatCurrency(price, region)}
    </div>
  )
}
```

What happens at request time:

1. User requests `/products/abc123`
2. Server instantly returns the **prerendered static shell** (product name, description, image)
3. TTFB is near-zero because the shell was built at deploy time
4. Dynamic parts (`ProductPrice`, `InventoryStatus`, `Recommendations`) **stream in** as they resolve
5. User sees the page almost instantly with placeholders, then dynamic content fills in

This is the best of both worlds: **static-site speed with dynamic-site freshness.**

### PPR vs ISR vs SSR: The Trade-off Table

| Strategy | TTFB | Data Freshness | Compute Cost | When to Use |
|----------|------|---------------|--------------|-------------|
| Full Static (SSG) | ~0ms | Stale until rebuild | Build-time only | Marketing pages, docs |
| ISR | ~0ms (cached), ~200ms (revalidating) | Minutes to hours old | Per-revalidation | Blog posts, product catalogs |
| PPR | ~0ms (shell), streaming (dynamic) | Shell: stale, Dynamic: fresh | Per-request (dynamic parts only) | Product pages, dashboards |
| Full SSR | ~200-2000ms | Always fresh | Per-request (everything) | User-specific pages |

My recommendation: **PPR for most pages in 2026.** It gives you the performance of static with the flexibility of dynamic. ISR for content-heavy pages where everything can be cached. Full SSR only when the entire page is user-specific.

---

## 9. Middleware

### What Middleware Is

Middleware runs **before** every request hits your application. Before route matching, before rendering, before caching. It intercepts the request at the edge and can:

- Redirect or rewrite the request
- Set or modify headers
- Read and set cookies
- Return a response directly (blocking the request)

```tsx
// middleware.ts (must be at the root of your project)
import { NextResponse } from 'next/server'
import type { NextRequest } from 'next/server'

export function middleware(request: NextRequest) {
  // Example: redirect non-www to www
  const hostname = request.headers.get('host')
  if (hostname === 'example.com') {
    return NextResponse.redirect(
      new URL(request.url.replace('example.com', 'www.example.com'))
    )
  }

  return NextResponse.next()
}

export const config = {
  // Only run on specific paths
  matcher: ['/((?!_next/static|_next/image|favicon.ico).*)'],
}
```

### Auth Checks in Middleware

The most common middleware pattern -- protect routes before they render:

```tsx
// middleware.ts
import { NextResponse } from 'next/server'
import type { NextRequest } from 'next/server'
import { verifyToken } from '@/lib/auth'

const protectedRoutes = ['/dashboard', '/settings', '/api/protected']
const authRoutes = ['/login', '/register']

export async function middleware(request: NextRequest) {
  const token = request.cookies.get('session')?.value
  const { pathname } = request.nextUrl

  // Check if route is protected
  const isProtected = protectedRoutes.some((route) =>
    pathname.startsWith(route)
  )
  const isAuthRoute = authRoutes.some((route) =>
    pathname.startsWith(route)
  )

  if (isProtected) {
    if (!token) {
      const loginUrl = new URL('/login', request.url)
      loginUrl.searchParams.set('redirect', pathname)
      return NextResponse.redirect(loginUrl)
    }

    const session = await verifyToken(token)
    if (!session) {
      return NextResponse.redirect(new URL('/login', request.url))
    }

    // Add user info to headers for downstream use
    const response = NextResponse.next()
    response.headers.set('x-user-id', session.userId)
    return response
  }

  // Redirect logged-in users away from auth pages
  if (isAuthRoute && token) {
    const session = await verifyToken(token)
    if (session) {
      return NextResponse.redirect(new URL('/dashboard', request.url))
    }
  }

  return NextResponse.next()
}

export const config = {
  matcher: ['/((?!_next/static|_next/image|favicon.ico).*)'],
}
```

### Geolocation-Based Routing

Middleware has access to geo data on Vercel (or you can use a geo-IP service):

```tsx
// middleware.ts
import { NextResponse } from 'next/server'
import type { NextRequest } from 'next/server'

export function middleware(request: NextRequest) {
  const country = request.geo?.country ?? 'US'
  const { pathname } = request.nextUrl

  // Redirect EU users to GDPR-compliant version
  const euCountries = ['DE', 'FR', 'IT', 'ES', 'NL', 'BE', 'AT', 'PL']
  if (euCountries.includes(country) && !pathname.startsWith('/eu')) {
    return NextResponse.rewrite(new URL(`/eu${pathname}`, request.url))
  }

  // Set region cookie for downstream use
  const response = NextResponse.next()
  response.cookies.set('region', country, {
    httpOnly: true,
    sameSite: 'lax',
    maxAge: 60 * 60 * 24, // 1 day
  })

  return response
}
```

### A/B Testing with Middleware

```tsx
// middleware.ts
import { NextResponse } from 'next/server'
import type { NextRequest } from 'next/server'

export function middleware(request: NextRequest) {
  const { pathname } = request.nextUrl

  // Only apply to specific pages
  if (pathname !== '/pricing') return NextResponse.next()

  // Check if user already has a variant assigned
  let variant = request.cookies.get('pricing-variant')?.value

  if (!variant) {
    // Assign variant randomly
    variant = Math.random() < 0.5 ? 'control' : 'new-pricing'
  }

  // Rewrite to the variant page
  const response = NextResponse.rewrite(
    new URL(`/pricing/${variant}`, request.url)
  )

  // Persist the variant assignment
  response.cookies.set('pricing-variant', variant, {
    maxAge: 60 * 60 * 24 * 30, // 30 days
  })

  return response
}
```

### Rate Limiting in Middleware

```tsx
// middleware.ts
import { NextResponse } from 'next/server'
import type { NextRequest } from 'next/server'

// Simple in-memory rate limiter (use Redis in production)
const rateLimit = new Map<string, { count: number; timestamp: number }>()

function isRateLimited(ip: string, limit: number, windowMs: number): boolean {
  const now = Date.now()
  const entry = rateLimit.get(ip)

  if (!entry || now - entry.timestamp > windowMs) {
    rateLimit.set(ip, { count: 1, timestamp: now })
    return false
  }

  entry.count++
  return entry.count > limit
}

export function middleware(request: NextRequest) {
  if (request.nextUrl.pathname.startsWith('/api/')) {
    const ip = request.headers.get('x-forwarded-for') ?? '127.0.0.1'

    if (isRateLimited(ip, 100, 60 * 1000)) {
      return new Response('Too many requests', {
        status: 429,
        headers: { 'Retry-After': '60' },
      })
    }
  }

  return NextResponse.next()
}
```

### Middleware Limitations

Keep in mind:

- Middleware should be **fast.** It runs on every matched request. Don't put heavy computation here.
- On Vercel, middleware runs at the Edge by default. This means limited Node.js API access.
- You cannot use `fs`, heavy npm packages, or Node.js-specific APIs in Edge middleware.
- Middleware cannot directly render HTML or return React components.
- There is **one** middleware file per project (`middleware.ts` at the root). Use route matching to scope behavior.

---

## 10. Next.js as a BFF (Backend for Frontend)

### The Monorepo Pattern Where Your Web App IS Your API

Here's an architectural pattern that I've seen work extremely well for teams building both a web app and a mobile app: **use your Next.js application as the Backend for Frontend.**

The idea: your Next.js Route Handlers and Server Actions aren't just serving your web app -- they're the API layer for your React Native mobile app too. One codebase, one deployment, shared types.

```
monorepo/
+-- apps/
|   +-- web/                    <-- Next.js app (web + API)
|   |   +-- app/
|   |   |   +-- api/
|   |   |   |   +-- users/
|   |   |   |   |   +-- route.ts
|   |   |   |   +-- posts/
|   |   |   |   |   +-- route.ts
|   |   |   |   +-- auth/
|   |   |   |       +-- route.ts
|   |   |   +-- page.tsx
|   |   |   +-- dashboard/
|   |   |       +-- page.tsx
|   |   +-- package.json
|   +-- mobile/                 <-- React Native app
|       +-- src/
|       |   +-- screens/
|       |   +-- api/
|       |       +-- client.ts   <-- Typed API client
|       +-- package.json
+-- packages/
|   +-- shared-types/           <-- Shared TypeScript types
|   |   +-- src/
|   |   |   +-- user.ts
|   |   |   +-- post.ts
|   |   |   +-- index.ts
|   |   +-- package.json
|   +-- validators/             <-- Shared validation schemas
|       +-- src/
|       |   +-- user.ts
|       |   +-- post.ts
|       +-- package.json
+-- package.json
```

### Shared Types

The key to making this work is sharing types between the API and the mobile client:

```tsx
// packages/shared-types/src/user.ts
export interface User {
  id: string
  email: string
  name: string
  avatarUrl: string | null
  createdAt: string
}

export interface CreateUserInput {
  email: string
  name: string
  password: string
}

export interface LoginInput {
  email: string
  password: string
}

export interface AuthResponse {
  user: User
  token: string
}

export interface ApiError {
  message: string
  code: string
  details?: Record<string, string[]>
}

export type ApiResponse<T> =
  | { success: true; data: T }
  | { success: false; error: ApiError }
```

### The API Routes

```tsx
// apps/web/app/api/auth/login/route.ts
import { NextResponse } from 'next/server'
import { LoginSchema } from '@repo/validators'
import { verifyPassword, createToken } from '@/lib/auth'
import { db } from '@/lib/db'
import type { AuthResponse, ApiResponse } from '@repo/shared-types'

export async function POST(request: Request) {
  const body = await request.json()
  const parsed = LoginSchema.safeParse(body)

  if (!parsed.success) {
    return NextResponse.json<ApiResponse<never>>({
      success: false,
      error: {
        message: 'Validation failed',
        code: 'VALIDATION_ERROR',
        details: parsed.error.flatten().fieldErrors,
      },
    }, { status: 400 })
  }

  const user = await db.users.findUnique({
    where: { email: parsed.data.email },
  })

  if (!user || !await verifyPassword(parsed.data.password, user.passwordHash)) {
    return NextResponse.json<ApiResponse<never>>({
      success: false,
      error: {
        message: 'Invalid credentials',
        code: 'INVALID_CREDENTIALS',
      },
    }, { status: 401 })
  }

  const token = await createToken(user.id)

  return NextResponse.json<ApiResponse<AuthResponse>>({
    success: true,
    data: {
      user: {
        id: user.id,
        email: user.email,
        name: user.name,
        avatarUrl: user.avatarUrl,
        createdAt: user.createdAt.toISOString(),
      },
      token,
    },
  })
}
```

### Typed Fetch Client for Mobile

```tsx
// apps/mobile/src/api/client.ts
import type {
  ApiResponse,
  User,
  AuthResponse,
  CreateUserInput,
  LoginInput,
} from '@repo/shared-types'

const API_BASE = process.env.EXPO_PUBLIC_API_URL ?? 'https://myapp.com'

class ApiClient {
  private token: string | null = null

  setToken(token: string) {
    this.token = token
  }

  clearToken() {
    this.token = null
  }

  private async fetch<T>(
    path: string,
    options: RequestInit = {}
  ): Promise<ApiResponse<T>> {
    const headers: Record<string, string> = {
      'Content-Type': 'application/json',
      ...(options.headers as Record<string, string>),
    }

    if (this.token) {
      headers['Authorization'] = `Bearer ${this.token}`
    }

    const response = await fetch(`${API_BASE}/api${path}`, {
      ...options,
      headers,
    })

    return response.json()
  }

  // Auth
  async login(input: LoginInput): Promise<ApiResponse<AuthResponse>> {
    return this.fetch('/auth/login', {
      method: 'POST',
      body: JSON.stringify(input),
    })
  }

  async register(input: CreateUserInput): Promise<ApiResponse<AuthResponse>> {
    return this.fetch('/auth/register', {
      method: 'POST',
      body: JSON.stringify(input),
    })
  }

  // Users
  async getProfile(): Promise<ApiResponse<User>> {
    return this.fetch('/users/me')
  }

  async updateProfile(data: Partial<User>): Promise<ApiResponse<User>> {
    return this.fetch('/users/me', {
      method: 'PATCH',
      body: JSON.stringify(data),
    })
  }
}

export const api = new ApiClient()
```

### Using tRPC for Type Safety

For even tighter type safety, tRPC eliminates the need for a separate API client. The types flow from server to client automatically:

```tsx
// packages/trpc/src/router.ts
import { initTRPC, TRPCError } from '@trpc/server'
import { z } from 'zod'
import { db } from '@repo/database'

const t = initTRPC.context<{ userId: string | null }>().create()

const publicProcedure = t.procedure
const protectedProcedure = t.procedure.use(({ ctx, next }) => {
  if (!ctx.userId) {
    throw new TRPCError({ code: 'UNAUTHORIZED' })
  }
  return next({ ctx: { userId: ctx.userId } })
})

export const appRouter = t.router({
  posts: t.router({
    list: publicProcedure
      .input(z.object({
        page: z.number().min(1).default(1),
        limit: z.number().min(1).max(50).default(20),
      }))
      .query(async ({ input }) => {
        return db.posts.findMany({
          skip: (input.page - 1) * input.limit,
          take: input.limit,
          orderBy: { createdAt: 'desc' },
        })
      }),

    create: protectedProcedure
      .input(z.object({
        title: z.string().min(1).max(200),
        content: z.string().min(1),
      }))
      .mutation(async ({ ctx, input }) => {
        return db.posts.create({
          data: {
            ...input,
            authorId: ctx.userId,
          },
        })
      }),
  }),

  users: t.router({
    me: protectedProcedure.query(async ({ ctx }) => {
      return db.users.findUniqueOrThrow({
        where: { id: ctx.userId },
      })
    }),
  }),
})

export type AppRouter = typeof appRouter
```

```tsx
// apps/web/app/api/trpc/[trpc]/route.ts
import { fetchRequestHandler } from '@trpc/server/adapters/fetch'
import { appRouter } from '@repo/trpc'
import { getSessionFromRequest } from '@/lib/auth'

const handler = async (request: Request) => {
  return fetchRequestHandler({
    endpoint: '/api/trpc',
    req: request,
    router: appRouter,
    createContext: async () => {
      const session = await getSessionFromRequest(request)
      return { userId: session?.userId ?? null }
    },
  })
}

export { handler as GET, handler as POST }
```

```tsx
// apps/mobile/src/api/trpc.ts
import { createTRPCClient, httpBatchLink } from '@trpc/client'
import type { AppRouter } from '@repo/trpc'

export const trpc = createTRPCClient<AppRouter>({
  links: [
    httpBatchLink({
      url: `${process.env.EXPO_PUBLIC_API_URL}/api/trpc`,
      headers() {
        const token = getStoredToken() // from secure storage
        return token ? { Authorization: `Bearer ${token}` } : {}
      },
    }),
  ],
})

// Usage in React Native -- fully typed!
const posts = await trpc.posts.list.query({ page: 1, limit: 20 })
// posts is typed as Post[] -- no manual type assertions

const newPost = await trpc.posts.create.mutate({
  title: 'Hello from mobile',
  content: 'This was posted from the React Native app',
})
```

### Why This Pattern Works

| Benefit | Description |
|---------|-------------|
| **Single source of truth** | One API, one set of business logic, one database layer |
| **Type safety** | Types flow from server to all clients via shared packages or tRPC |
| **Single deployment** | Your API scales with your web app on Vercel |
| **Shared validation** | Zod schemas used on server and client |
| **No API drift** | Web and mobile always use the same API version |
| **Incremental adoption** | Start with typed fetch, migrate to tRPC later |

### When This Pattern Doesn't Work

Be honest with yourself about the trade-offs:

- **If your API serves many different consumers** (not just your apps), a dedicated API service might be better
- **If your API has very different scaling needs** than your web frontend, coupling them hurts
- **If your team is large** and you want API and frontend to deploy independently, a BFF adds coupling
- **If you need GraphQL** or another API paradigm, this becomes awkward

For teams of 2-20 building a web app + mobile app? This pattern is a cheat code.

---

## 11. Performance Patterns

### Image Optimization

Next.js `<Image>` is not optional -- it's a performance requirement. Here's why it matters and how to use it properly:

```tsx
import Image from 'next/image'

// BAD: Raw img tag
// <img src="/hero.jpg" alt="Hero" />
// Result: 4MB unoptimized JPEG, no responsive sizes, layout shift

// GOOD: Next.js Image
<Image
  src="/hero.jpg"
  alt="Hero image showing our product"
  width={1200}
  height={600}
  priority  // Above-the-fold images should be priority
  placeholder="blur"
  blurDataURL="data:image/jpeg;base64,/9j/4AAQ..."
/>
```

What `<Image>` does for you:
- **Automatic format conversion** -- serves WebP/AVIF to browsers that support it
- **Responsive sizes** -- generates multiple resolutions, serves the right one
- **Lazy loading** -- images below the fold load on scroll (unless `priority` is set)
- **Layout shift prevention** -- reserves space based on width/height
- **CDN caching** -- optimized images are cached at the edge

For responsive images with different sizes at different breakpoints:

```tsx
<Image
  src="/product.jpg"
  alt="Product photo"
  width={800}
  height={600}
  sizes="(max-width: 640px) 100vw, (max-width: 1024px) 50vw, 33vw"
/>
```

The `sizes` attribute tells the browser which size image to request based on viewport width. Without it, the browser might download a 2x image on a small screen.

For images with unknown dimensions (user-uploaded content), use `fill`:

```tsx
<div className="relative w-full aspect-video">
  <Image
    src={user.avatarUrl}
    alt={user.name}
    fill
    className="object-cover rounded"
    sizes="(max-width: 768px) 100vw, 50vw"
  />
</div>
```

### Font Optimization

`next/font` eliminates layout shift from font loading and self-hosts fonts (no Google Fonts network requests):

```tsx
// app/layout.tsx
import { Inter, JetBrains_Mono } from 'next/font/google'

const inter = Inter({
  subsets: ['latin'],
  display: 'swap',
  variable: '--font-inter',
})

const jetbrainsMono = JetBrains_Mono({
  subsets: ['latin'],
  display: 'swap',
  variable: '--font-mono',
})

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en" className={`${inter.variable} ${jetbrainsMono.variable}`}>
      <body className="font-sans">{children}</body>
    </html>
  )
}
```

```ts
// tailwind.config.ts -- Use CSS variables for font families
// fontFamily: {
//   sans: ['var(--font-inter)', 'system-ui', 'sans-serif'],
//   mono: ['var(--font-mono)', 'monospace'],
// }
```

For local/custom fonts:

```tsx
import localFont from 'next/font/local'

const customFont = localFont({
  src: [
    { path: './fonts/custom-regular.woff2', weight: '400', style: 'normal' },
    { path: './fonts/custom-bold.woff2', weight: '700', style: 'normal' },
    { path: './fonts/custom-italic.woff2', weight: '400', style: 'italic' },
  ],
  display: 'swap',
  variable: '--font-custom',
})
```

### Script Loading

Third-party scripts are a performance killer. `next/script` gives you fine-grained control:

```tsx
import Script from 'next/script'

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        {children}

        {/* Load analytics after page is interactive */}
        <Script
          src="https://www.googletagmanager.com/gtag/js?id=G-XXXXX"
          strategy="afterInteractive"
        />
        <Script id="gtag-init" strategy="afterInteractive">
          {`
            window.dataLayer = window.dataLayer || [];
            function gtag(){dataLayer.push(arguments);}
            gtag('js', new Date());
            gtag('config', 'G-XXXXX');
          `}
        </Script>

        {/* Load non-critical scripts when browser is idle */}
        <Script
          src="https://cdn.intercom.io/widget.js"
          strategy="lazyOnload"
        />

        {/* Critical scripts that must load before page (rare) */}
        <Script
          src="/scripts/polyfill.js"
          strategy="beforeInteractive"
        />
      </body>
    </html>
  )
}
```

| Strategy | When it loads | Use for |
|----------|-------------|---------|
| `beforeInteractive` | Before page hydration | Polyfills, critical detection scripts |
| `afterInteractive` | After page hydration | Analytics, tag managers |
| `lazyOnload` | During browser idle time | Chat widgets, social embeds |
| `worker` | In a web worker (experimental) | Heavy non-UI scripts |

### Bundle Analysis

You need to see what's in your bundles. Install the analyzer:

```bash
npm install @next/bundle-analyzer
```

```tsx
// next.config.ts
import type { NextConfig } from 'next'
import withBundleAnalyzer from '@next/bundle-analyzer'

const nextConfig: NextConfig = {
  // your config
}

export default process.env.ANALYZE === 'true'
  ? withBundleAnalyzer({ enabled: true })(nextConfig)
  : nextConfig
```

```bash
ANALYZE=true npm run build
```

This opens an interactive treemap showing every module in your bundles. Look for:
- **Large dependencies** you didn't know about (moment.js, lodash full import)
- **Duplicate dependencies** (two versions of the same library)
- **Client Components** that include server-only code
- **Unused exports** that tree-shaking didn't catch

Common wins:
```tsx
// BAD: imports entire lodash
import _ from 'lodash'
_.debounce(fn, 300)

// GOOD: tree-shakeable import
import debounce from 'lodash/debounce'
debounce(fn, 300)

// BETTER: use a tiny alternative
import { debounce } from '@/lib/utils' // your own 10-line implementation
```

### Core Web Vitals Optimization

Here's a checklist I use for every Next.js project:

**LCP (Largest Contentful Paint) -- Target: < 2.5s**

```tsx
// 1. Priority load above-the-fold images
<Image src="/hero.jpg" alt="Hero" priority />

// 2. Use streaming to get content to the browser faster
// (Suspense boundaries as shown in section 6)

// 3. Avoid client-side data fetching for above-the-fold content
// Use Server Components instead
```

**CLS (Cumulative Layout Shift) -- Target: < 0.1**

```tsx
// 1. Always set width/height on images
<Image src="/photo.jpg" width={800} height={600} alt="..." />

// 2. Use font-display: swap with next/font (automatic)
const inter = Inter({ subsets: ['latin'], display: 'swap' })

// 3. Reserve space for dynamic content
<div className="min-h-[200px]"> {/* Reserve space */}
  <Suspense fallback={<Skeleton className="h-[200px]" />}>
    <DynamicContent />
  </Suspense>
</div>

// 4. Avoid injecting content above existing content
// Don't dynamically add banners above the main content after load
```

**INP (Interaction to Next Paint) -- Target: < 200ms**

```tsx
// 1. Keep Client Components small and focused
// Don't wrap entire pages in "use client"

// 2. Use useTransition for non-urgent updates
'use client'
import { useTransition } from 'react'

function SearchFilter({ onFilter }: { onFilter: (q: string) => void }) {
  const [isPending, startTransition] = useTransition()

  return (
    <input
      onChange={(e) => {
        startTransition(() => {
          onFilter(e.target.value) // Non-blocking update
        })
      }}
    />
  )
}

// 3. Lazy load heavy Client Components
import dynamic from 'next/dynamic'

const HeavyEditor = dynamic(() => import('./editor'), {
  loading: () => <EditorSkeleton />,
  ssr: false, // Don't SSR if it's purely interactive
})

// 4. Avoid large synchronous operations in event handlers
// Debounce, virtualize long lists, use web workers for heavy computation
```

### Dynamic Imports for Code Splitting

Beyond `next/dynamic`, you can use standard dynamic imports to split your code:

```tsx
// Load a heavy library only when needed
async function handleExport() {
  const { exportToPDF } = await import('@/lib/pdf-export')
  await exportToPDF(data)
}

// Conditionally load components
export default function EditorPage() {
  const [showPreview, setShowPreview] = useState(false)

  return (
    <div>
      <Editor />
      <button onClick={() => setShowPreview(true)}>Preview</button>
      {showPreview && <MarkdownPreview />}
    </div>
  )
}

// With next/dynamic for SSR control
const MarkdownPreview = dynamic(
  () => import('./markdown-preview'),
  { ssr: false } // Only loads in browser
)
```

### The Performance Optimization Decision Tree

When optimizing a Next.js App Router application, follow this order:

1. **Architecture first:** Are you using Server Components where possible? Is the `"use client"` boundary pushed down?
2. **Data fetching:** Are you using streaming? Are slow queries behind Suspense boundaries?
3. **Caching:** Is frequently-accessed data cached with `use cache`? Are tags set up for invalidation?
4. **Assets:** Are images using `<Image>`? Are fonts using `next/font`? Are scripts deferred?
5. **Bundle:** Run the analyzer. Remove unnecessary dependencies. Dynamic import heavy libraries.
6. **Measure:** Use Lighthouse, Web Vitals, and real-user monitoring. Don't optimize blindly.

---

## Putting It All Together

Let me show you a complete file structure for a well-architected App Router application:

```
app/
+-- layout.tsx                        # Root layout: html, body, fonts, global providers
+-- page.tsx                          # Home page (Server Component)
+-- loading.tsx                       # Global loading state
+-- error.tsx                         # Global error boundary
+-- not-found.tsx                     # Global 404
+-- globals.css                       # Global styles
|
+-- (marketing)/                      # Route group: no /marketing in URL
|   +-- layout.tsx                    # Marketing layout (different nav)
|   +-- page.tsx                      # / (home, if you override)
|   +-- about/page.tsx                # /about
|   +-- pricing/page.tsx              # /pricing
|   +-- blog/
|       +-- page.tsx                  # /blog (list)
|       +-- [slug]/page.tsx           # /blog/my-post
|
+-- (app)/                            # Route group: app pages
|   +-- layout.tsx                    # App layout (sidebar, auth check)
|   +-- dashboard/
|   |   +-- page.tsx                  # /dashboard
|   |   +-- loading.tsx               # Dashboard loading skeleton
|   |   +-- @analytics/              # Parallel route
|   |   |   +-- page.tsx
|   |   +-- @activity/               # Parallel route
|   |       +-- page.tsx
|   +-- settings/
|   |   +-- layout.tsx                # Settings sub-layout (tabs)
|   |   +-- page.tsx                  # /settings (redirects to /settings/profile)
|   |   +-- profile/page.tsx          # /settings/profile
|   |   +-- billing/page.tsx          # /settings/billing
|   +-- projects/
|       +-- page.tsx                  # /projects (list)
|       +-- new/page.tsx              # /projects/new (creation form)
|       +-- [id]/
|           +-- page.tsx              # /projects/:id
|           +-- loading.tsx
|           +-- settings/page.tsx     # /projects/:id/settings
|
+-- api/
|   +-- webhooks/
|   |   +-- stripe/route.ts           # POST /api/webhooks/stripe
|   +-- trpc/
|   |   +-- [trpc]/route.ts           # tRPC handler
|   +-- health/route.ts               # GET /api/health
|
+-- actions/                           # Shared Server Actions
    +-- auth.ts
    +-- projects.ts
    +-- billing.ts

middleware.ts                          # Auth, geo, rate limiting
```

Each piece has its place. Server Components for data. Client Components for interactivity. Server Actions for mutations. Route Handlers for webhooks and external APIs. Layouts for persistent UI. Suspense for progressive loading. Middleware for request-level logic.

---

## Key Takeaways

1. **Server Components are the default.** You opt INTO client-side interactivity, not out of server rendering. Push `"use client"` as far down the tree as possible.

2. **Server Actions replace most API routes.** For mutations triggered by your own UI, Server Actions are simpler, type-safe, and progressively enhanced.

3. **Streaming is free performance.** Wrap slow data fetches in `<Suspense>` boundaries. The user sees the page shell instantly.

4. **The caching model is declarative.** Use `'use cache'` with `cacheLife` and `cacheTag` for fine-grained caching. Use `revalidateTag` for surgical invalidation.

5. **PPR is the future.** Static shells with dynamic holes. Near-zero TTFB with fresh data. Use it.

6. **Middleware is your bouncer.** Auth checks, redirects, geo-routing, A/B tests -- all before the request hits your app.

7. **Next.js makes an excellent BFF.** Share types between web and mobile via a monorepo. Use tRPC for end-to-end type safety.

8. **Performance is architectural.** The biggest wins come from using the right rendering strategy (Server vs. Client, Static vs. Dynamic, ISR vs. PPR), not from micro-optimizations.

---

*Next up in Chapter 26: we'll dive into Vercel's platform features -- deployment, preview environments, edge functions, and the infrastructure that makes all of this fast in production.*
