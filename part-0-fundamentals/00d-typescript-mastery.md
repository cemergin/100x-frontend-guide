<!--
  CHAPTER: 0d
  TITLE: TypeScript — From Fundamentals to Enterprise Mastery
  PART: 0 — JavaScript & React Fundamentals
  PREREQS: Chapter 0
  KEY_TOPICS: type system, inference, unions, intersections, discriminated unions, generics, type guards, utility types, conditional types, template literal types, module augmentation, declaration files, structural typing, branded types, Zod, type-safe APIs, tsconfig, project references, monorepo configs, strictness, type challenges
  DIFFICULTY: Beginner → Advanced
  UPDATED: 2026-04-07
-->

# Chapter 0d: TypeScript — From Fundamentals to Enterprise Mastery

> **Part 0 — JavaScript & React Fundamentals** | Prerequisites: Chapter 0 | Difficulty: Beginner to Advanced

<details>
<summary><strong>TL;DR</strong></summary>

- TypeScript eliminates ~70% of production JavaScript bugs at compile time; treat it as a type system, not "JS with annotations"
- Structural typing means shape matters, not name; discriminated unions make impossible states unrepresentable and are the most important pattern in frontend TS
- Generics with constraints let you build APIs that guide callers toward correct usage through types alone
- Conditional types, mapped types, and template literals enable type-level programming -- powerful for library authors, use sparingly in application code
- Zod bridges runtime validation and compile-time types; use it at every system boundary (API responses, form inputs, environment variables)
- Strict mode (`strict: true` in tsconfig) is non-negotiable; start with it or migrate to it early

</details>

Here's a number for you: **70% of JavaScript bugs in production are type errors.** Not logic errors, not race conditions, not off-by-one mistakes — type errors. Calling a method on `undefined`. Passing a string where a number was expected. Accessing a property that doesn't exist because someone renamed it three weeks ago and the codebase has 400 files.

TypeScript eliminates this entire category of bugs. Not reduces. *Eliminates.* And it does so before your code ever runs — at the moment you write it, in your editor, with a red squiggly line that says "hey, this is going to break."

But here's what most TypeScript tutorials get wrong: they teach it as "JavaScript with type annotations." Sprinkle some `: string` and `: number` around, add an interface, call it a day. That's like learning to drive by memorizing where the pedals are. You can technically move the car, but you don't understand what's happening when you turn the wheel.

TypeScript is a *type system*. It's a language for describing the shapes and behaviors of data. It's a way of encoding contracts between different parts of your codebase so that the compiler can verify those contracts automatically. When you understand it deeply — really deeply — you can express constraints that catch entire categories of bugs, make impossible states unrepresentable, and build APIs that guide users toward correct usage through the types alone.

This chapter is the one you'll keep coming back to. We're going from the absolute fundamentals through the kind of advanced type-level programming that makes library authors cry tears of joy. By the end, you'll be the person on your team who reads the TypeScript error messages, understands what they mean, and fixes them in minutes instead of hours.

### In This Chapter
- Why TypeScript Exists — JavaScript's type coercion problem and the billion-dollar mistake
- The Type System Fundamentals — primitives, objects, arrays, tuples, type annotations vs inference
- Structural vs Nominal Typing — why TypeScript is different from Java/C#
- Unions, Intersections, and Discriminated Unions — the most important patterns in frontend TS
- Interfaces vs Type Aliases — when to use each and why it matters
- Generics — parameterized types from basics to advanced constraints
- Type Guards and Narrowing — runtime checks that inform the compiler
- Utility Types — every built-in type transformer with real-world examples
- Conditional Types, Mapped Types, and Template Literals — type-level programming
- Advanced Patterns for Frontend — branded types, Zod, type-safe APIs, module augmentation
- tsconfig Mastery — every important compiler option explained
- Enterprise TypeScript — migration, strictness, tooling, performance
- Type Challenges — hands-on exercises to make it all stick

### Related Chapters
- [Ch 0: JavaScript Fundamentals] — the runtime language TypeScript compiles to
- [Ch 3: State Management] — type-safe state with Zustand, Redux Toolkit
- [Ch 7: API Layer] — type-safe API clients with tRPC, React Query
- [Ch 12: Monorepo Architecture] — tsconfig project references, workspace configs

---

## 1. WHY TYPESCRIPT EXISTS

### JavaScript's Type Coercion Problem

JavaScript is dynamically typed, which means variables don't have types — values do. A variable can hold a string one moment and a number the next. This flexibility is part of what made JavaScript approachable for beginners, but it's also the source of an enormous class of bugs.

Consider JavaScript's type coercion rules:

```javascript
// JavaScript: "This is fine."
"5" + 3        // "53"  (string concatenation)
"5" - 3        // 2     (numeric subtraction)
true + true    // 2     (booleans coerced to numbers)
[] + []        // ""    (arrays coerced to strings)
[] + {}        // "[object Object]"
{} + []        // 0     (block statement + unary plus)
"" == false    // true  (abstract equality coercion)
null == undefined  // true
NaN === NaN    // false (NaN is not equal to itself)
```

These aren't edge cases — they're the language specification. Every one of these has bitten someone in production. The `"5" + 3` pattern alone has caused thousands of bugs in forms that concatenate user input with numeric values instead of adding them.

### The Billion-Dollar Mistake

Tony Hoare, who invented null references in 1965, called it his "billion-dollar mistake." JavaScript doubled down on this mistake by having *two* null-like values: `null` and `undefined`. And both of them are silent killers:

```javascript
function getUser(id) {
  return users.find(u => u.id === id);
  // Returns undefined if not found — silently
}

const user = getUser(42);
console.log(user.name);  // Runtime error: Cannot read properties of undefined
```

This is the most common JavaScript error in the world. And it's entirely preventable. TypeScript's `strictNullChecks` flag makes `null` and `undefined` explicit parts of the type system. You can't access `.name` on something that might be `undefined` without first checking:

```typescript
function getUser(id: number): User | undefined {
  return users.find(u => u.id === id);
}

const user = getUser(42);
console.log(user.name);  // TS Error: 'user' is possibly 'undefined'

// You must narrow first:
if (user) {
  console.log(user.name);  // OK — TypeScript knows user is User here
}
```

### TypeScript as "Sets of Allowed Values"

Here's the mental model that makes everything else click: **every type in TypeScript is a set of allowed values.**

- `string` is the set of all possible strings
- `number` is the set of all possible numbers
- `"hello"` is a set with exactly one member: the string "hello"
- `string | number` is the union of both sets
- `never` is the empty set — no values
- `unknown` is the universal set — all values

When you annotate `x: string`, you're saying "x can hold any value from the set of all strings." When TypeScript checks your code, it's doing set theory — verifying that the set of values that *could* flow into a position is a subset of the set of values that position *accepts*.

This mental model explains everything from union types (set union) to intersection types (set intersection) to `never` (empty set) to `unknown` (universal set). Keep it in mind as we go deeper.

### Catching Errors at Compile Time

Here's the real value proposition: **every error TypeScript catches at compile time is an error you don't have to debug in production.** And compile-time errors come with exact file locations, exact line numbers, and descriptions of what went wrong.

Compare:
- **Runtime error:** "TypeError: Cannot read properties of undefined (reading 'map')" — somewhere in your app, at some point, under some conditions, for some users
- **Compile-time error:** "Property 'map' does not exist on type 'string'. Did you mean to use 'match'?" — right here, line 47, in this file, right now

The second one takes seconds to fix. The first one might take hours — reproducing the conditions, tracing through the call stack, figuring out which code path leads to the value being `undefined` instead of an array.

TypeScript isn't about writing more code. It's about writing *less debugging code* — fewer console.logs, fewer runtime checks, fewer "wait, what type is this variable at this point?"

---

## 2. THE TYPE SYSTEM FUNDAMENTALS

### Primitive Types

TypeScript has the same primitive types as JavaScript, plus a few extras:

```typescript
// The basics
let name: string = "Alice";
let age: number = 30;
let isActive: boolean = true;
let nothing: null = null;
let notDefined: undefined = undefined;

// BigInt (for numbers larger than Number.MAX_SAFE_INTEGER)
let bigNumber: bigint = 9007199254740991n;

// Symbol (unique identifiers)
let id: symbol = Symbol("id");
```

### Variable Typing and Inference: `let` vs `const`

TypeScript's inference is context-aware, and the difference between `let` and `const` matters:

```typescript
let name = "Alice";      // Type: string (can be reassigned to any string)
const name = "Alice";    // Type: "Alice" (literal type — can never change)

let count = 42;          // Type: number
const count = 42;        // Type: 42

let active = true;       // Type: boolean
const active = true;     // Type: true
```

Why does this matter? Because `const` declarations get **literal types** — the narrowest possible type. TypeScript knows a `const` can never be reassigned, so it infers the exact value as the type. This is incredibly useful for discriminated unions, configuration objects, and anywhere you need exact string/number matching.

```typescript
// This matters in practice:
const config = {
  mode: "production",   // Type: string (not "production"!)
  port: 3000,           // Type: number (not 3000!)
};

// Wait, why? Because object properties CAN be reassigned.
// config.mode = "development" is legal.

// To get literal types in objects, use 'as const':
const config = {
  mode: "production",
  port: 3000,
} as const;
// Type: { readonly mode: "production"; readonly port: 3000 }
```

The `as const` assertion is one of the most underused features in TypeScript. It makes the entire object deeply readonly with literal types. Use it for configuration objects, action types, route definitions — anywhere you have constant data.

### Object Types

Objects can be typed inline or through interfaces/type aliases (we'll cover the difference later):

```typescript
// Inline object type
let user: { name: string; age: number; email?: string } = {
  name: "Alice",
  age: 30,
};

// Optional properties (?) can be undefined
user.email;  // Type: string | undefined

// Readonly properties
let point: { readonly x: number; readonly y: number } = { x: 10, y: 20 };
point.x = 5;  // Error: Cannot assign to 'x' because it is a read-only property

// Index signatures (dynamic keys)
let scores: { [key: string]: number } = {};
scores["math"] = 95;
scores["english"] = 88;

// Record shorthand (equivalent to index signature)
let scores: Record<string, number> = {};
```

### Array Types

Two equivalent syntaxes:

```typescript
let numbers: number[] = [1, 2, 3];
let names: Array<string> = ["Alice", "Bob"];  // Generic syntax

// Readonly arrays (can't push, pop, or modify)
let frozen: readonly number[] = [1, 2, 3];
frozen.push(4);  // Error: Property 'push' does not exist on type 'readonly number[]'

// Or using ReadonlyArray<T>
let frozen: ReadonlyArray<number> = [1, 2, 3];
```

Prefer `number[]` over `Array<number>` — it's more concise and the community convention. Use `readonly` arrays for data that shouldn't be mutated (function parameters, Redux state, constants).

### Tuples

Tuples are fixed-length arrays where each position has a specific type:

```typescript
// A tuple: exactly 2 elements, string then number
let pair: [string, number] = ["Alice", 30];

// Destructuring works naturally
const [name, age] = pair;  // name: string, age: number

// Named tuples (documentation only, no runtime effect)
type UserEntry = [name: string, age: number, active: boolean];

// Tuples with optional elements
type FlexiblePair = [string, number?];
const a: FlexiblePair = ["hello"];       // OK
const b: FlexiblePair = ["hello", 42];   // OK

// Rest elements in tuples
type AtLeastOne = [string, ...string[]];
const c: AtLeastOne = ["hello"];                    // OK
const d: AtLeastOne = ["hello", "world", "!"];      // OK
const e: AtLeastOne = [];                           // Error: needs at least 1 element

// Readonly tuples
type Point = readonly [number, number];
const origin: Point = [0, 0];
origin[0] = 5;  // Error: Cannot assign to '0' because it is a read-only property
```

Tuples are common in React — `useState` returns `[state, setter]`, and custom hooks often return tuples.

### Type Annotations vs Inference

Here's the rule of thumb that experienced TypeScript developers follow:

**Let TypeScript infer when it can. Annotate when you should.**

```typescript
// DON'T annotate what TS can infer:
const name: string = "Alice";        // Redundant — TS knows it's a string
const numbers: number[] = [1, 2, 3]; // Redundant — TS knows it's number[]

// DO annotate:

// 1. Function parameters (TS can't infer these)
function greet(name: string): void {
  console.log(`Hello, ${name}`);
}

// 2. Function return types (makes the contract explicit)
function getUser(id: number): User | undefined {
  return users.find(u => u.id === id);
}

// 3. Empty initializations
const items: string[] = [];  // Without annotation, TS infers never[]
const map: Map<string, User> = new Map();

// 4. Complex expressions where the inferred type is unclear
const result: ApiResponse<User> = await fetchUser(id);
```

There's a philosophical debate about whether you should *always* annotate function return types. Here's my take: **annotate return types for exported functions, let inference work for internal ones.** Explicit return types on exports create a contract boundary — if you accidentally change the implementation in a way that changes the return type, the compiler catches it at the function definition rather than at every call site. This is especially important for library code and shared modules.

### `any` vs `unknown` vs `never`: The Danger Hierarchy

These three types are the edges of TypeScript's type system, and understanding them is essential.

#### `any`: The Escape Hatch (and the Trap)

`any` disables type checking entirely. A value of type `any` can be assigned to anything, and anything can be assigned to it:

```typescript
let x: any = "hello";
x = 42;
x = { foo: "bar" };
x.nonExistentMethod();  // No error — TS doesn't check
x.a.b.c.d;              // No error — TS doesn't check

// any is contagious:
const y: string = x;  // No error — any bypasses the type system
y.toUpperCase();       // Might crash at runtime if x wasn't a string
```

**`any` is the virus of the type system.** Once it enters your codebase, it spreads. A function that returns `any` infects every variable that stores its result. A parameter typed as `any` means the function body has no type safety.

Use `any` only as a last resort during migration from JavaScript. In a mature codebase, every `any` should be a TODO.

#### `unknown`: The Safe Catch-All

`unknown` is the type-safe counterpart of `any`. It can hold any value, but you can't do anything with it until you narrow it:

```typescript
let x: unknown = "hello";

// Can't use it directly:
x.toUpperCase();  // Error: 'x' is of type 'unknown'
x + 1;            // Error: 'x' is of type 'unknown'

// Must narrow first:
if (typeof x === "string") {
  x.toUpperCase();  // OK — TypeScript knows x is string here
}

if (x instanceof Date) {
  x.getTime();  // OK — TypeScript knows x is Date here
}
```

**Use `unknown` instead of `any` for values whose type you don't know yet.** This includes:
- API responses before validation
- `catch` clause errors
- User input
- Third-party library returns with bad typings

```typescript
// Bad:
try {
  await fetchData();
} catch (error: any) {
  console.log(error.message);  // Might crash if error isn't an Error
}

// Good:
try {
  await fetchData();
} catch (error: unknown) {
  if (error instanceof Error) {
    console.log(error.message);  // Safe
  } else {
    console.log("Unknown error:", String(error));
  }
}
```

#### `never`: The Impossible Type

`never` is the empty set — no value can ever be of type `never`. It represents things that should never happen:

```typescript
// Functions that never return:
function throwError(message: string): never {
  throw new Error(message);
}

function infiniteLoop(): never {
  while (true) {}
}

// Exhaustive checks:
type Shape = "circle" | "square" | "triangle";

function getArea(shape: Shape): number {
  switch (shape) {
    case "circle": return Math.PI * 10 * 10;
    case "square": return 10 * 10;
    case "triangle": return (10 * 10) / 2;
    default:
      // If we handled all cases, shape is 'never' here.
      // If someone adds a new shape and forgets to handle it,
      // this line will show a compile error.
      const exhaustiveCheck: never = shape;
      return exhaustiveCheck;
  }
}
```

The exhaustive check pattern is one of the most powerful TypeScript patterns. It turns a runtime bug (forgetting to handle a new case) into a compile-time error.

### Type Casting with `as`

Sometimes you know more about a value's type than TypeScript does. The `as` keyword tells TypeScript to treat a value as a specific type:

```typescript
// DOM elements:
const input = document.getElementById("email") as HTMLInputElement;
input.value = "test@example.com";  // OK — we told TS this is an input element

// API responses after validation:
const data = await response.json() as User;
```

**Important:** `as` is a lie you tell the compiler. It performs no runtime check. If you're wrong, you get runtime errors:

```typescript
const data = await response.json() as User;
// If the API returns something else, this will crash later
// when you try to access User properties
```

Prefer type guards (runtime checks that narrow types) over type assertions. We'll cover these in section 7.

#### Double Assertion through `unknown`

TypeScript prevents obviously wrong assertions:

```typescript
const x = "hello" as number;  // Error: can't assert string to number
```

But sometimes you need to force it (usually during migration or with poorly typed libraries):

```typescript
const x = "hello" as unknown as number;  // Works — but you're on your own
```

This is called a "double assertion" — you first widen to `unknown` (the universal set, which any type is assignable to) and then narrow to your target type. **This should be exceptionally rare in your codebase.** If you find yourself doing it often, something is wrong with your types.

---

## 3. STRUCTURAL VS NOMINAL TYPING

This section explains something that trips up every developer coming from Java, C#, or Swift.

### Structural Typing: Shape Matters, Not Name

TypeScript uses **structural typing** (also called "duck typing" at the type level). Two types are compatible if they have the same shape — the same properties with the same types. The name of the type is irrelevant.

```typescript
interface Dog {
  name: string;
  breed: string;
}

interface Pet {
  name: string;
  breed: string;
}

const myDog: Dog = { name: "Rex", breed: "Labrador" };
const myPet: Pet = myDog;  // OK! Same shape = compatible

// This function accepts anything with a name property:
function greet(thing: { name: string }) {
  console.log(`Hello, ${thing.name}`);
}

greet(myDog);  // OK
greet(myPet);  // OK
greet({ name: "Alice", age: 30 });  // OK — has 'name', extra props are fine here
```

This is fundamentally different from Java or C#, where `Dog` and `Pet` would be incompatible types even with identical fields, unless one explicitly extends or implements the other.

### Why Structural Typing Makes Sense for JavaScript

JavaScript is a structural language at runtime. There's no way to check "is this object an instance of the Dog interface?" at runtime — interfaces don't exist at runtime. What you *can* check is "does this object have a `name` property that's a string?" Structural typing aligns TypeScript's type system with JavaScript's actual behavior.

### When You Want Nominal Typing: The Branded Types Pattern

Sometimes structural typing is too permissive. Consider:

```typescript
type UserId = string;
type OrderId = string;

function getUser(id: UserId): User { ... }
function getOrder(id: OrderId): Order { ... }

const userId: UserId = "user_123";
const orderId: OrderId = "order_456";

getUser(orderId);  // No error! Both are just 'string'
```

This is a real bug waiting to happen. You passed an order ID to a function that expects a user ID, and TypeScript didn't catch it because both are structurally identical (`string`).

**Branded types** solve this by adding a phantom property that makes types structurally distinct:

```typescript
// The brand pattern:
type Brand<T, B extends string> = T & { readonly __brand: B };

type UserId = Brand<string, "UserId">;
type OrderId = Brand<string, "OrderId">;

// Constructor functions:
function createUserId(id: string): UserId {
  return id as UserId;
}

function createOrderId(id: string): OrderId {
  return id as OrderId;
}

// Now the compiler catches mistakes:
const userId = createUserId("user_123");
const orderId = createOrderId("order_456");

function getUser(id: UserId): User { ... }

getUser(userId);   // OK
getUser(orderId);  // Error! OrderId is not assignable to UserId
getUser("raw_string");  // Error! string is not assignable to UserId
```

The `__brand` property never actually exists at runtime — it's purely a compile-time fiction. But it makes `UserId` and `OrderId` structurally different, so TypeScript treats them as incompatible types.

**Where to use branded types:**
- IDs (UserId, OrderId, ProductId — never mix them up)
- Validated strings (Email, PhoneNumber, URL — can only be created through validation)
- Units (Pixels, Rem, Percentage — don't add pixels to percentages)
- Money (USD, EUR — don't add different currencies)

```typescript
// Validated email type:
type Email = Brand<string, "Email">;

function validateEmail(input: string): Email | null {
  const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
  return emailRegex.test(input) ? (input as Email) : null;
}

function sendEmail(to: Email, subject: string): void {
  // We know 'to' has been validated — the type guarantees it
}

// Can't pass an unvalidated string:
sendEmail("not-validated", "Hello");  // Error!

// Must validate first:
const email = validateEmail(userInput);
if (email) {
  sendEmail(email, "Hello");  // OK
}
```

This pattern eliminates an entire class of "forgot to validate" bugs. The type system forces validation to happen.

---

## 4. UNIONS AND INTERSECTIONS

Unions and intersections are the core of TypeScript's type algebra. If types are sets of values, unions are set unions and intersections are set intersections.

### Union Types: `A | B`

A union type `A | B` means "a value that belongs to set A *or* set B (or both)":

```typescript
type StringOrNumber = string | number;

let x: StringOrNumber = "hello";  // OK
x = 42;                           // OK
x = true;                         // Error: boolean is not in the union

// You can only use properties common to ALL members:
function printLength(x: string | number): void {
  x.toString();    // OK — both string and number have toString
  x.toUpperCase(); // Error — number doesn't have toUpperCase
  x.toFixed();     // Error — string doesn't have toFixed
}
```

### Intersection Types: `A & B`

An intersection type `A & B` means "a value that belongs to set A *and* set B simultaneously":

```typescript
type HasName = { name: string };
type HasAge = { age: number };

type Person = HasName & HasAge;
// Equivalent to: { name: string; age: number }

const person: Person = { name: "Alice", age: 30 };  // Must have BOTH

// Intersections combine all properties:
type Admin = Person & { role: "admin"; permissions: string[] };
```

A common misconception: intersection of *primitives* that don't overlap produces `never`:

```typescript
type Impossible = string & number;  // never — no value is both string and number
```

### Discriminated Unions: The Most Important Pattern in Frontend TypeScript

Discriminated unions are, without exaggeration, the single most useful TypeScript pattern for frontend development. They let you model states that have different shapes depending on a "tag" field.

```typescript
// The classic example: async request state
type RequestState<T> =
  | { status: "idle" }
  | { status: "loading" }
  | { status: "success"; data: T }
  | { status: "error"; error: Error };

// Each variant has a 'status' field (the discriminant).
// TypeScript can narrow based on this field.

function renderUserProfile(state: RequestState<User>) {
  switch (state.status) {
    case "idle":
      return <Text>Press button to load</Text>;
    case "loading":
      return <ActivityIndicator />;
    case "success":
      // TypeScript knows 'data' exists here
      return <Text>{state.data.name}</Text>;
    case "error":
      // TypeScript knows 'error' exists here
      return <Text>Error: {state.error.message}</Text>;
  }
}
```

**Why this is so powerful:** In plain JavaScript, you'd model this as a single object with optional fields:

```javascript
// The old way — error-prone
const state = {
  loading: false,
  error: null,
  data: null,
};

// Nothing prevents impossible states:
const broken = {
  loading: true,
  error: new Error("fail"),
  data: { name: "Alice" },
  // Loading, errored, AND has data? What do you render?
};
```

With discriminated unions, **impossible states are unrepresentable.** You literally cannot create a value that is both "loading" and "success" — the type system forbids it.

More real-world discriminated unions:

```typescript
// Navigation routes
type Route =
  | { name: "Home" }
  | { name: "Profile"; params: { userId: string } }
  | { name: "Settings"; params: { section?: string } }
  | { name: "Chat"; params: { roomId: string; messageId?: string } };

// Form field types
type FormField =
  | { type: "text"; value: string; maxLength?: number }
  | { type: "number"; value: number; min?: number; max?: number }
  | { type: "select"; value: string; options: string[] }
  | { type: "checkbox"; value: boolean }
  | { type: "date"; value: Date; minDate?: Date; maxDate?: Date };

// Payment methods
type PaymentMethod =
  | { type: "card"; last4: string; brand: "visa" | "mastercard" | "amex" }
  | { type: "paypal"; email: string }
  | { type: "apple_pay" }
  | { type: "bank_transfer"; bankName: string; accountLast4: string };

// Animation states
type AnimationState =
  | { phase: "idle" }
  | { phase: "entering"; progress: number }
  | { phase: "active"; startedAt: number }
  | { phase: "exiting"; progress: number }
  | { phase: "exited" };
```

### Control Flow Narrowing

TypeScript's control flow analysis is remarkably smart. It tracks how conditions narrow types through your code:

```typescript
function processValue(x: string | number | null) {
  // Here: x is string | number | null

  if (x === null) {
    // Here: x is null
    return;
  }
  // Here: x is string | number (null was eliminated)

  if (typeof x === "string") {
    // Here: x is string
    x.toUpperCase();  // OK
  } else {
    // Here: x is number
    x.toFixed(2);  // OK
  }
}
```

TypeScript narrows based on:
- `typeof` checks: `typeof x === "string"` narrows to `string`
- `instanceof` checks: `x instanceof Date` narrows to `Date`
- `in` operator: `"name" in x` narrows to types that have `name`
- Equality checks: `x === null`, `x === "hello"`
- Truthiness: `if (x)` eliminates `null`, `undefined`, `0`, `""`, `false`
- Discriminant property: `if (x.status === "success")`

```typescript
// Truthiness narrowing:
function printName(name: string | null | undefined) {
  if (name) {
    // name is string (null and undefined are falsy)
    console.log(name.toUpperCase());
  }
}

// in narrowing:
type Fish = { swim: () => void };
type Bird = { fly: () => void };

function move(animal: Fish | Bird) {
  if ("swim" in animal) {
    animal.swim();  // TypeScript knows it's Fish
  } else {
    animal.fly();   // TypeScript knows it's Bird
  }
}
```

### Exhaustive Checks with `never`

The exhaustive check pattern uses `never` to ensure you handle every variant of a union:

```typescript
type Shape =
  | { kind: "circle"; radius: number }
  | { kind: "square"; side: number }
  | { kind: "triangle"; base: number; height: number };

function area(shape: Shape): number {
  switch (shape.kind) {
    case "circle":
      return Math.PI * shape.radius ** 2;
    case "square":
      return shape.side ** 2;
    case "triangle":
      return (shape.base * shape.height) / 2;
    default:
      // shape is 'never' here — all cases handled
      const _exhaustive: never = shape;
      return _exhaustive;
  }
}
```

Now if someone adds `{ kind: "rectangle"; width: number; height: number }` to the `Shape` union, the `default` case will get a compile error because `shape` would be `{ kind: "rectangle"; ... }` instead of `never`. The developer is *forced* to add the new case.

A cleaner utility for exhaustive checks:

```typescript
function assertNever(value: never, message?: string): never {
  throw new Error(message ?? `Unexpected value: ${JSON.stringify(value)}`);
}

// Usage:
function area(shape: Shape): number {
  switch (shape.kind) {
    case "circle":    return Math.PI * shape.radius ** 2;
    case "square":    return shape.side ** 2;
    case "triangle":  return (shape.base * shape.height) / 2;
    default:          return assertNever(shape);
  }
}
```

---

## 5. INTERFACES VS TYPE ALIASES

This is one of the most common questions in TypeScript, and the answer is more nuanced than "they're the same thing."

### Interface

```typescript
interface User {
  name: string;
  age: number;
  email?: string;
}

// Extends other interfaces:
interface Admin extends User {
  role: "admin";
  permissions: string[];
}

// Can extend multiple interfaces:
interface SuperAdmin extends Admin, Auditable {
  superPowers: string[];
}
```

### Type Alias

```typescript
type User = {
  name: string;
  age: number;
  email?: string;
};

// Uses intersection for extension:
type Admin = User & {
  role: "admin";
  permissions: string[];
};
```

### The Real Differences That Matter

**1. Declaration Merging — Interfaces can be reopened, types cannot:**

```typescript
// Interface: declaration merging
interface Window {
  myCustomProperty: string;
}
// This MERGES with the existing Window interface — doesn't replace it

// Type: no merging
type Window = {
  myCustomProperty: string;
};
// Error: Duplicate identifier 'Window'
```

Declaration merging is essential for:
- Augmenting third-party types (adding properties to `Window`, `Request`, etc.)
- Library authors who want consumers to extend their types
- Module augmentation (we'll cover this in section 10)

**2. Union and Intersection Types — Only type aliases can do this:**

```typescript
// Type aliases can be unions:
type StringOrNumber = string | number;
type RequestStatus = "idle" | "loading" | "success" | "error";

// Interfaces cannot be unions:
interface StringOrNumber = string | number;  // Syntax error

// Type aliases can be mapped types, conditional types, etc:
type Nullable<T> = T | null;
type ReadonlyDeep<T> = { readonly [K in keyof T]: ReadonlyDeep<T[K]> };
```

**3. Performance — Interfaces are slightly more efficient for the compiler:**

TypeScript's compiler caches interface checks more effectively than type aliases. For large codebases, using interfaces for object shapes can marginally improve compile times. In practice, this difference only matters at extreme scale (thousands of types).

**4. Error Messages — Interfaces show the name, type aliases sometimes inline:**

```typescript
interface UserInterface {
  name: string;
  age: number;
}

type UserType = {
  name: string;
  age: number;
};

// Error messages with interfaces tend to show 'UserInterface'
// Error messages with type aliases sometimes inline the full type
```

### My Recommendation

Here's the rule I follow and recommend for teams:

- **Use `interface` for object shapes** — things that describe the structure of a value. Components props, API responses, domain models.
- **Use `type` for everything else** — unions, intersections, utility types, conditional types, mapped types, tuple types.
- **Use `interface` when you need declaration merging** — augmenting third-party types, library extension points.
- **Use `type` for discriminated unions** — they require union syntax.

```typescript
// interface: object shapes
interface User {
  id: string;
  name: string;
  email: string;
}

interface UserListProps {
  users: User[];
  onSelect: (user: User) => void;
}

// type: everything else
type RequestStatus = "idle" | "loading" | "success" | "error";
type RequestState<T> =
  | { status: "idle" }
  | { status: "loading" }
  | { status: "success"; data: T }
  | { status: "error"; error: Error };
type Nullable<T> = T | null;
type UserOrAdmin = User | Admin;
```

---

## 6. GENERICS

Generics are parameterized types — types that take other types as arguments. They're the mechanism for writing reusable, type-safe abstractions.

### What Generics Solve

Without generics, you'd have to choose between type safety and reusability:

```typescript
// Type-safe but not reusable:
function firstString(arr: string[]): string | undefined {
  return arr[0];
}
function firstNumber(arr: number[]): number | undefined {
  return arr[0];
}

// Reusable but not type-safe:
function first(arr: any[]): any {
  return arr[0];
}

// With generics: both reusable AND type-safe:
function first<T>(arr: T[]): T | undefined {
  return arr[0];
}

first(["hello", "world"]);  // Type: string | undefined
first([1, 2, 3]);           // Type: number | undefined
```

`T` is a **type parameter** — a placeholder that gets filled in when the function is called. TypeScript usually infers `T` from the arguments, so you rarely need to specify it explicitly.

### Generic Functions

```typescript
// Identity function — the "hello world" of generics:
function identity<T>(value: T): T {
  return value;
}
identity("hello");  // T inferred as string, returns string
identity(42);       // T inferred as number, returns number

// Multiple type parameters:
function pair<A, B>(first: A, second: B): [A, B] {
  return [first, second];
}
pair("hello", 42);  // Type: [string, number]

// Generic arrow functions (note the trailing comma in TSX files):
const identity = <T,>(value: T): T => value;
// The comma after T is needed in .tsx files to distinguish from JSX

// Generic async functions:
async function fetchData<T>(url: string): Promise<T> {
  const response = await fetch(url);
  return response.json() as T;
}
const users = await fetchData<User[]>("/api/users");  // Type: User[]
```

### Generic Interfaces and Type Aliases

```typescript
// Generic interface:
interface ApiResponse<T> {
  data: T;
  status: number;
  message: string;
  timestamp: Date;
}

// Usage:
type UserResponse = ApiResponse<User>;
type UserListResponse = ApiResponse<User[]>;

// Generic type alias:
type Result<T, E = Error> = 
  | { ok: true; value: T }
  | { ok: false; error: E };

// The E = Error is a default type parameter
const result: Result<User> = { ok: true, value: user };
const error: Result<User> = { ok: false, error: new Error("Not found") };
const customError: Result<User, ValidationError> = { 
  ok: false, 
  error: new ValidationError("Invalid email") 
};
```

### Constraints with `extends`

You can constrain what types are allowed for a generic parameter:

```typescript
// T must have a 'length' property:
function longest<T extends { length: number }>(a: T, b: T): T {
  return a.length >= b.length ? a : b;
}

longest("hello", "hi");       // OK — strings have length
longest([1, 2, 3], [1, 2]);   // OK — arrays have length
longest(10, 20);               // Error — numbers don't have length

// T must be a key of the object:
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}

const user = { name: "Alice", age: 30 };
getProperty(user, "name");   // Type: string
getProperty(user, "age");    // Type: number
getProperty(user, "email");  // Error: "email" is not in keyof User

// Multiple constraints using intersection:
function merge<T extends object, U extends object>(a: T, b: U): T & U {
  return { ...a, ...b };
}
```

The `keyof` operator is essential with generics — it produces a union of an object's keys:

```typescript
interface User {
  name: string;
  age: number;
  email: string;
}

type UserKeys = keyof User;  // "name" | "age" | "email"
```

### Default Type Parameters

Like function default arguments, generics can have defaults:

```typescript
interface PaginatedResponse<T, M = { page: number; total: number }> {
  data: T[];
  meta: M;
}

// Uses default meta type:
type UserList = PaginatedResponse<User>;
// Equivalent to PaginatedResponse<User, { page: number; total: number }>

// Custom meta type:
type CursorUserList = PaginatedResponse<User, { cursor: string; hasMore: boolean }>;
```

### Inference with Generics

TypeScript's generic inference is powerful — most of the time, you don't need to specify type arguments explicitly:

```typescript
// TS infers T from the argument:
function wrap<T>(value: T): { wrapped: T } {
  return { wrapped: value };
}
wrap("hello");  // T = string, inferred
wrap(42);       // T = number, inferred

// TS infers from multiple arguments:
function combine<T>(arr: T[], value: T): T[] {
  return [...arr, value];
}
combine([1, 2, 3], 4);    // T = number
combine(["a", "b"], "c"); // T = string
combine([1, 2, 3], "4");  // Error: string is not number

// When inference fails, specify explicitly:
const emptyList = useState<User[]>([]);  // Can't infer User[] from []
```

### Best Practices for Generics

**1. Single-letter names are fine for simple generics:**

```typescript
// These are conventional and understood:
function map<T, U>(arr: T[], fn: (item: T) => U): U[] { ... }
// T = the input type, U = the output type

// Use descriptive names for complex generics:
type EventHandler<TEvent extends BaseEvent, TResult = void> = 
  (event: TEvent) => TResult;
```

Common conventions: `T` for a single type, `K` for keys, `V` for values, `E` for elements, `R` for return types.

**2. Don't use generics when you don't need them:**

```typescript
// Unnecessary generic — string works fine:
function greet<T extends string>(name: T): string {
  return `Hello, ${name}`;
}

// Just use string:
function greet(name: string): string {
  return `Hello, ${name}`;
}
```

A generic parameter should appear in at least two places (input and output, or in a constraint). If it only appears once, you probably don't need it.

**3. Prefer constraints over manual narrowing:**

```typescript
// Bad: accepts anything, then narrows
function processItems<T>(items: T): void {
  if (Array.isArray(items)) { ... }
}

// Good: constrains the type upfront
function processItems<T>(items: T[]): void { ... }
```

---

## 7. TYPE GUARDS AND NARROWING

Type guards are runtime checks that TypeScript understands. They bridge the gap between TypeScript's compile-time type system and JavaScript's runtime behavior.

### Built-in Type Guards

#### `typeof` — for primitives

```typescript
function padLeft(value: string, padding: string | number): string {
  if (typeof padding === "number") {
    return " ".repeat(padding) + value;  // padding is number
  }
  return padding + value;  // padding is string
}
```

`typeof` works for: `"string"`, `"number"`, `"boolean"`, `"symbol"`, `"bigint"`, `"undefined"`, `"function"`, `"object"`.

Note: `typeof null === "object"` (JavaScript's oldest bug). TypeScript handles this correctly in narrowing, but be aware.

#### `instanceof` — for class instances

```typescript
function formatError(error: Error | string): string {
  if (error instanceof Error) {
    return `${error.name}: ${error.message}`;  // error is Error
  }
  return error;  // error is string
}

// Works with custom classes:
class ApiError extends Error {
  constructor(public statusCode: number, message: string) {
    super(message);
  }
}

function handleError(error: unknown) {
  if (error instanceof ApiError) {
    console.log(`API Error ${error.statusCode}: ${error.message}`);
  } else if (error instanceof Error) {
    console.log(`Error: ${error.message}`);
  } else {
    console.log(`Unknown error: ${String(error)}`);
  }
}
```

#### `in` operator — for property existence

```typescript
type Fish = { swim: () => void; name: string };
type Bird = { fly: () => void; name: string };

function move(animal: Fish | Bird): string {
  if ("swim" in animal) {
    animal.swim();   // animal is Fish
    return `${animal.name} swims`;
  }
  animal.fly();      // animal is Bird
  return `${animal.name} flies`;
}
```

#### Truthiness narrowing

```typescript
function printName(name: string | null | undefined) {
  if (name) {
    // name is string — null and undefined are falsy
    console.log(name.toUpperCase());
  }
}

// Be careful with values that are legitimately falsy:
function printCount(count: number | null) {
  if (count) {
    console.log(count);  // Misses count === 0!
  }
  // Better:
  if (count !== null) {
    console.log(count);  // Correctly handles 0
  }
}
```

### User-Defined Type Guards: The `is` Keyword

When built-in checks aren't enough, you can write custom type guard functions:

```typescript
interface Cat {
  meow(): void;
  purr(): void;
}

interface Dog {
  bark(): void;
  fetch(): void;
}

// Type guard function — the return type `animal is Cat` is the magic
function isCat(animal: Cat | Dog): animal is Cat {
  return "meow" in animal;
}

function interact(animal: Cat | Dog) {
  if (isCat(animal)) {
    animal.purr();  // TypeScript knows it's Cat
  } else {
    animal.fetch(); // TypeScript knows it's Dog
  }
}
```

The `animal is Cat` return type annotation tells TypeScript: "if this function returns true, the argument is of type Cat." This is a contract you're responsible for upholding — TypeScript trusts your implementation.

**Real-world type guard examples:**

```typescript
// Validate API response shape:
interface User {
  id: string;
  name: string;
  email: string;
}

function isUser(value: unknown): value is User {
  return (
    typeof value === "object" &&
    value !== null &&
    "id" in value &&
    "name" in value &&
    "email" in value &&
    typeof (value as User).id === "string" &&
    typeof (value as User).name === "string" &&
    typeof (value as User).email === "string"
  );
}

// Filter arrays by type:
type Item = TextItem | ImageItem | VideoItem;

function isImageItem(item: Item): item is ImageItem {
  return item.type === "image";
}

const images = items.filter(isImageItem);
// Type: ImageItem[] — TypeScript narrows the array!

// Check for non-null values:
function isNonNull<T>(value: T): value is NonNullable<T> {
  return value !== null && value !== undefined;
}

const values: (string | null)[] = ["hello", null, "world", null];
const strings = values.filter(isNonNull);
// Type: string[]
```

### Assertion Functions: The `asserts` Keyword

Assertion functions are like type guards, but instead of returning a boolean, they throw if the condition isn't met:

```typescript
function assertIsString(value: unknown): asserts value is string {
  if (typeof value !== "string") {
    throw new Error(`Expected string, got ${typeof value}`);
  }
}

function processInput(input: unknown) {
  assertIsString(input);
  // After the assertion, TypeScript knows input is string
  console.log(input.toUpperCase());  // OK
}

// Assert non-null:
function assertDefined<T>(value: T): asserts value is NonNullable<T> {
  if (value === null || value === undefined) {
    throw new Error("Value is null or undefined");
  }
}

function processUser(user: User | null) {
  assertDefined(user);
  // user is User here
  console.log(user.name);
}
```

Assertion functions are useful in test code and at boundary layers where you want to fail fast.

---

## 8. UTILITY TYPES

TypeScript ships with a collection of built-in utility types that transform existing types. These are the power tools of everyday TypeScript.

### `Partial<T>` — Make all properties optional

```typescript
interface User {
  name: string;
  age: number;
  email: string;
}

type PartialUser = Partial<User>;
// { name?: string; age?: number; email?: string }

// Real-world: update functions that accept partial objects
function updateUser(id: string, updates: Partial<User>): void {
  // Can pass any subset of User properties
}

updateUser("123", { name: "Alice" });          // OK
updateUser("123", { age: 31, email: "a@b.c" }); // OK
updateUser("123", {});                          // OK
```

### `Required<T>` — Make all properties required

```typescript
interface Config {
  host?: string;
  port?: number;
  debug?: boolean;
}

type FullConfig = Required<Config>;
// { host: string; port: number; debug: boolean }

// Real-world: resolved config after applying defaults
function resolveConfig(partial: Config): Required<Config> {
  return {
    host: partial.host ?? "localhost",
    port: partial.port ?? 3000,
    debug: partial.debug ?? false,
  };
}
```

### `Pick<T, K>` — Select specific properties

```typescript
interface User {
  id: string;
  name: string;
  email: string;
  password: string;
  createdAt: Date;
}

type PublicUser = Pick<User, "id" | "name" | "email">;
// { id: string; name: string; email: string }

// Real-world: API response that only includes certain fields
type UserListItem = Pick<User, "id" | "name">;
```

### `Omit<T, K>` — Remove specific properties

```typescript
type UserWithoutPassword = Omit<User, "password">;
// { id: string; name: string; email: string; createdAt: Date }

// Real-world: creating a new user (no id yet)
type CreateUserInput = Omit<User, "id" | "createdAt">;

// Combining with Partial:
type UpdateUserInput = Partial<Omit<User, "id" | "createdAt">>;
```

### `Record<K, V>` — Object type with specific key and value types

```typescript
// Simple key-value maps:
type Scores = Record<string, number>;
const scores: Scores = { math: 95, english: 88 };

// With string literal keys:
type RequestStatus = "idle" | "loading" | "success" | "error";
type StatusMessages = Record<RequestStatus, string>;
const messages: StatusMessages = {
  idle: "Press start",
  loading: "Loading...",
  success: "Done!",
  error: "Something went wrong",
};
// Every key must be present — no missing cases!

// Real-world: grouping items by category
type GroupedUsers = Record<string, User[]>;
```

### `Exclude<T, U>` — Remove types from a union

```typescript
type Status = "active" | "inactive" | "deleted" | "banned";

type VisibleStatus = Exclude<Status, "deleted">;
// "active" | "inactive" | "banned"

type LiveStatus = Exclude<Status, "deleted" | "banned">;
// "active" | "inactive"
```

### `Extract<T, U>` — Keep only matching types from a union

```typescript
type AllEvents = "click" | "scroll" | "keydown" | "keyup" | "resize";

type KeyEvents = Extract<AllEvents, "keydown" | "keyup">;
// "keydown" | "keyup"

// More commonly used with complex unions:
type StringEvent = Extract<AllEvents, `key${string}`>;
// "keydown" | "keyup" — matches the pattern
```

### `ReturnType<T>` — Extract the return type of a function

```typescript
function createUser(name: string, age: number) {
  return { id: crypto.randomUUID(), name, age, createdAt: new Date() };
}

type User = ReturnType<typeof createUser>;
// { id: string; name: string; age: number; createdAt: Date }

// Incredibly useful for inferring types from factory functions
// and hooks without manually defining them.
```

### `Parameters<T>` — Extract function parameter types as a tuple

```typescript
function search(query: string, page: number, limit: number): SearchResult {
  // ...
}

type SearchParams = Parameters<typeof search>;
// [query: string, page: number, limit: number]

// Useful for wrapper functions:
function loggedSearch(...args: Parameters<typeof search>): SearchResult {
  console.log("Searching:", args);
  return search(...args);
}
```

### `Awaited<T>` — Unwrap Promise types

```typescript
type A = Awaited<Promise<string>>;           // string
type B = Awaited<Promise<Promise<number>>>;  // number (deeply unwraps)

// Real-world: getting the resolved type of an async function
async function fetchUsers(): Promise<User[]> { ... }

type Users = Awaited<ReturnType<typeof fetchUsers>>;
// User[]
```

### `NonNullable<T>` — Remove null and undefined

```typescript
type MaybeString = string | null | undefined;
type DefinitelyString = NonNullable<MaybeString>;
// string

// Useful in generic contexts:
function process<T>(value: T): NonNullable<T> {
  if (value === null || value === undefined) {
    throw new Error("Value required");
  }
  return value as NonNullable<T>;
}
```

### Combining Utility Types: Real React Native Examples

```typescript
// Props for a form component:
interface UserFormData {
  name: string;
  email: string;
  phone: string;
  avatar: string;
  bio: string;
}

// Edit form only needs partial data, no avatar upload inline
type EditFormProps = {
  initialData: Partial<Omit<UserFormData, "avatar">>;
  onSave: (data: Partial<UserFormData>) => Promise<void>;
};

// Display component picks specific fields:
type UserCardProps = Pick<UserFormData, "name" | "email" | "avatar"> & {
  onPress: () => void;
};

// Screen params — some screens need IDs, some don't:
type RootStackParams = {
  Home: undefined;
  UserList: undefined;
  UserDetail: Pick<UserFormData, "name"> & { userId: string };
  EditUser: { userId: string; initialData: Partial<UserFormData> };
};

// API response wrapper:
interface ApiResponse<T> {
  data: T;
  meta: {
    page: number;
    total: number;
    limit: number;
  };
}

type UserListResponse = ApiResponse<Pick<UserFormData, "name" | "email">[]>;

// Hook return type extraction:
function useUserForm(userId: string) {
  const [data, setData] = useState<Partial<UserFormData>>({});
  const [errors, setErrors] = useState<Partial<Record<keyof UserFormData, string>>>({});
  const [isSubmitting, setIsSubmitting] = useState(false);
  
  return { data, setData, errors, isSubmitting } as const;
}

type FormState = ReturnType<typeof useUserForm>;
```

---

## 9. CONDITIONAL TYPES, MAPPED TYPES, AND TEMPLATE LITERALS

This is where TypeScript goes from "useful type annotations" to "type-level programming." These features let you write types that compute other types — transforming, filtering, and generating type information at compile time.

### Conditional Types: `T extends U ? X : Y`

Conditional types are ternary expressions at the type level:

```typescript
type IsString<T> = T extends string ? true : false;

type A = IsString<string>;   // true
type B = IsString<number>;   // false
type C = IsString<"hello">;  // true ("hello" extends string)

// Practical: extract the element type from an array
type ElementOf<T> = T extends (infer E)[] ? E : never;

type X = ElementOf<string[]>;    // string
type Y = ElementOf<number[]>;    // number
type Z = ElementOf<string>;      // never (not an array)

// Practical: unwrap Promise
type UnwrapPromise<T> = T extends Promise<infer R> ? R : T;

type A = UnwrapPromise<Promise<string>>;  // string
type B = UnwrapPromise<number>;           // number (not a promise, returns as-is)
```

#### The `infer` Keyword

`infer` lets you introduce a type variable within a conditional type — it "captures" part of the type being tested:

```typescript
// Extract function return type (this is how ReturnType is implemented):
type MyReturnType<T> = T extends (...args: any[]) => infer R ? R : never;

// Extract first argument type:
type FirstArg<T> = T extends (first: infer F, ...rest: any[]) => any ? F : never;

type A = FirstArg<(name: string, age: number) => void>;  // string

// Extract props type from a React component:
type PropsOf<T> = T extends React.ComponentType<infer P> ? P : never;

type ButtonProps = PropsOf<typeof Button>;  // Whatever props Button accepts
```

#### Distributive Conditional Types

When a conditional type acts on a union, it distributes over each member:

```typescript
type ToArray<T> = T extends any ? T[] : never;

type A = ToArray<string | number>;
// string[] | number[]  (NOT (string | number)[])

// Each member is processed individually:
// ToArray<string> | ToArray<number> = string[] | number[]
```

This is usually what you want, but you can prevent distribution by wrapping in a tuple:

```typescript
type ToArrayNoDistribute<T> = [T] extends [any] ? T[] : never;

type B = ToArrayNoDistribute<string | number>;
// (string | number)[]
```

### Mapped Types: Transforming Object Types

Mapped types iterate over the keys of a type and produce a new type:

```typescript
// Make all properties optional (this is how Partial is implemented):
type MyPartial<T> = {
  [K in keyof T]?: T[K];
};

// Make all properties readonly:
type MyReadonly<T> = {
  readonly [K in keyof T]: T[K];
};

// Make all properties nullable:
type Nullable<T> = {
  [K in keyof T]: T[K] | null;
};

// Transform property types:
type Stringify<T> = {
  [K in keyof T]: string;
};

interface User {
  name: string;
  age: number;
  active: boolean;
}

type StringifiedUser = Stringify<User>;
// { name: string; age: string; active: string }
```

#### Removing Modifiers with `-`

```typescript
// Remove readonly:
type Mutable<T> = {
  -readonly [K in keyof T]: T[K];
};

// Remove optional:
type Concrete<T> = {
  [K in keyof T]-?: T[K];
};
```

#### Key Remapping with `as`

TypeScript 4.1 introduced key remapping — changing the keys while mapping:

```typescript
// Prefix all keys with 'get':
type Getters<T> = {
  [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K];
};

interface User {
  name: string;
  age: number;
}

type UserGetters = Getters<User>;
// {
//   getName: () => string;
//   getAge: () => number;
// }

// Filter keys by remapping to never:
type OnlyStrings<T> = {
  [K in keyof T as T[K] extends string ? K : never]: T[K];
};

type StringProps = OnlyStrings<User>;
// { name: string }  — age (number) was filtered out

// Create event handler types:
type EventHandlers<T> = {
  [K in keyof T as `on${Capitalize<string & K>}Change`]: (value: T[K]) => void;
};

type UserHandlers = EventHandlers<User>;
// {
//   onNameChange: (value: string) => void;
//   onAgeChange: (value: number) => void;
// }
```

### Template Literal Types

Template literal types apply template literal syntax to the type level:

```typescript
type Greeting = `Hello, ${string}`;

const a: Greeting = "Hello, Alice";  // OK
const b: Greeting = "Hello, Bob";    // OK
const c: Greeting = "Hi, Alice";     // Error: doesn't match pattern

// Combining with unions:
type Color = "red" | "green" | "blue";
type Size = "small" | "medium" | "large";

type ColorSize = `${Color}-${Size}`;
// "red-small" | "red-medium" | "red-large" | "green-small" | ...
// All 9 combinations!

// Built-in string manipulation types:
type Upper = Uppercase<"hello">;       // "HELLO"
type Lower = Lowercase<"HELLO">;       // "hello"
type Cap = Capitalize<"hello">;        // "Hello"
type Uncap = Uncapitalize<"Hello">;    // "hello"

// Real-world: CSS property types
type CSSUnit = "px" | "rem" | "em" | "%";
type CSSValue = `${number}${CSSUnit}`;

const width: CSSValue = "100px";   // OK
const height: CSSValue = "2.5rem"; // OK
const bad: CSSValue = "100";       // Error: missing unit

// Real-world: route pattern matching
type ApiRoute = `/api/${string}`;
type UserRoute = `/api/users/${string}`;

// Real-world: event names
type DOMEvent = `on${Capitalize<string>}`;
const handler: DOMEvent = "onClick";   // OK
const bad2: DOMEvent = "click";        // Error
```

### Putting It All Together: A Complex Real-World Example

Let's build a type-safe event emitter:

```typescript
type EventMap = {
  userCreated: { userId: string; name: string };
  userDeleted: { userId: string };
  orderPlaced: { orderId: string; items: string[]; total: number };
  orderCancelled: { orderId: string; reason: string };
};

// Generate listener method types:
type EventListeners<T extends Record<string, any>> = {
  [K in keyof T as `on${Capitalize<string & K>}`]: (
    callback: (data: T[K]) => void
  ) => void;
};

type Listeners = EventListeners<EventMap>;
// {
//   onUserCreated: (callback: (data: { userId: string; name: string }) => void) => void;
//   onUserDeleted: (callback: (data: { userId: string }) => void) => void;
//   onOrderPlaced: (callback: (data: { orderId: string; items: string[]; total: number }) => void) => void;
//   onOrderCancelled: (callback: (data: { orderId: string; reason: string }) => void) => void;
// }

// Or a more traditional emit pattern:
interface TypedEventEmitter<T extends Record<string, any>> {
  on<K extends keyof T>(event: K, listener: (data: T[K]) => void): void;
  off<K extends keyof T>(event: K, listener: (data: T[K]) => void): void;
  emit<K extends keyof T>(event: K, data: T[K]): void;
}

// Usage is fully type-safe:
declare const emitter: TypedEventEmitter<EventMap>;

emitter.on("userCreated", (data) => {
  // data is { userId: string; name: string } — fully inferred
  console.log(data.name);
});

emitter.emit("orderPlaced", {
  orderId: "123",
  items: ["widget"],
  total: 29.99,
});

emitter.emit("userCreated", { orderId: "123" });
// Error: missing userId and name, extra orderId
```

---

## 10. ADVANCED PATTERNS FOR FRONTEND

### Discriminated Unions for State Management

We introduced discriminated unions earlier. Let's see how they transform real-world state management:

```typescript
// A comprehensive async state type:
type AsyncState<T, E = Error> =
  | { status: "idle" }
  | { status: "loading"; startedAt: number }
  | { status: "success"; data: T; loadedAt: number }
  | { status: "error"; error: E; failedAt: number }
  | { status: "refreshing"; data: T; startedAt: number };

// Notice: 'refreshing' has both data (stale) AND is loading.
// This is a real state in apps — showing stale data while refreshing.

// Type-safe reducer:
type AsyncAction<T, E = Error> =
  | { type: "FETCH_START" }
  | { type: "FETCH_SUCCESS"; data: T }
  | { type: "FETCH_ERROR"; error: E }
  | { type: "REFRESH_START" }
  | { type: "RESET" };

function asyncReducer<T, E = Error>(
  state: AsyncState<T, E>,
  action: AsyncAction<T, E>
): AsyncState<T, E> {
  switch (action.type) {
    case "FETCH_START":
      return { status: "loading", startedAt: Date.now() };
    case "FETCH_SUCCESS":
      return { status: "success", data: action.data, loadedAt: Date.now() };
    case "FETCH_ERROR":
      return { status: "error", error: action.error, failedAt: Date.now() };
    case "REFRESH_START":
      if (state.status === "success") {
        return { status: "refreshing", data: state.data, startedAt: Date.now() };
      }
      return { status: "loading", startedAt: Date.now() };
    case "RESET":
      return { status: "idle" };
  }
}

// Component usage:
function UserProfile({ state }: { state: AsyncState<User> }) {
  switch (state.status) {
    case "idle":
      return null;
    case "loading":
      return <LoadingSpinner />;
    case "error":
      return <ErrorMessage error={state.error} />;
    case "success":
      return <ProfileCard user={state.data} />;
    case "refreshing":
      return (
        <>
          <ProfileCard user={state.data} />
          <RefreshIndicator />
        </>
      );
  }
}
```

### Branded Types for IDs (Revisited with Full Pattern)

Here's a production-ready branded types implementation:

```typescript
// Brand utility:
declare const __brand: unique symbol;
type Brand<T, B> = T & { readonly [__brand]: B };

// Define your branded ID types:
type UserId = Brand<string, "UserId">;
type OrderId = Brand<string, "OrderId">;
type ProductId = Brand<string, "ProductId">;
type SessionId = Brand<string, "SessionId">;

// Smart constructors (the only way to create branded values):
const UserId = (id: string): UserId => id as UserId;
const OrderId = (id: string): OrderId => id as OrderId;
const ProductId = (id: string): ProductId => id as ProductId;

// Domain models use branded types:
interface User {
  id: UserId;
  name: string;
  email: string;
}

interface Order {
  id: OrderId;
  userId: UserId;
  products: ProductId[];
  total: number;
}

// API functions are type-safe:
async function getUser(id: UserId): Promise<User> { ... }
async function getOrder(id: OrderId): Promise<Order> { ... }
async function getUserOrders(userId: UserId): Promise<Order[]> { ... }

// Compiler catches mistakes:
const userId = UserId("user_123");
const orderId = OrderId("order_456");

getUser(userId);     // OK
getUser(orderId);    // Error! OrderId is not assignable to UserId
getUser("raw_id");   // Error! string is not assignable to UserId

// Branded numeric types for units:
type Pixels = Brand<number, "Pixels">;
type Rem = Brand<number, "Rem">;
type Percentage = Brand<number, "Percentage">;

function setWidth(value: Pixels): void { ... }
setWidth(100 as Pixels);  // OK
setWidth(Pixels(100));     // OK (with smart constructor)

// Validated types:
type NonEmptyString = Brand<string, "NonEmptyString">;
type PositiveNumber = Brand<number, "PositiveNumber">;
type EmailAddress = Brand<string, "EmailAddress">;

function validateEmail(input: string): EmailAddress | null {
  const regex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
  return regex.test(input) ? (input as EmailAddress) : null;
}
```

### Zod Schema Inference: Single Source of Truth

Zod is a runtime validation library that integrates beautifully with TypeScript. Instead of defining types and validation separately (and keeping them in sync), you define a Zod schema and infer the type from it:

```typescript
import { z } from "zod";

// Define the schema (single source of truth):
const UserSchema = z.object({
  id: z.string().uuid(),
  name: z.string().min(1).max(100),
  email: z.string().email(),
  age: z.number().int().min(0).max(150),
  role: z.enum(["user", "admin", "moderator"]),
  preferences: z.object({
    theme: z.enum(["light", "dark", "system"]),
    notifications: z.boolean(),
    language: z.string().default("en"),
  }),
  createdAt: z.coerce.date(),
});

// Infer the TypeScript type from the schema:
type User = z.infer<typeof UserSchema>;
// {
//   id: string;
//   name: string;
//   email: string;
//   age: number;
//   role: "user" | "admin" | "moderator";
//   preferences: {
//     theme: "light" | "dark" | "system";
//     notifications: boolean;
//     language: string;
//   };
//   createdAt: Date;
// }

// Validate API responses:
async function fetchUser(id: string): Promise<User> {
  const response = await fetch(`/api/users/${id}`);
  const data = await response.json();
  return UserSchema.parse(data);
  // Throws ZodError if data doesn't match schema
  // Returns typed User if valid
}

// Partial schemas for updates:
const UpdateUserSchema = UserSchema.partial().omit({ id: true, createdAt: true });
type UpdateUser = z.infer<typeof UpdateUserSchema>;

// Schemas for forms:
const CreateUserSchema = UserSchema.omit({ id: true, createdAt: true });
type CreateUserInput = z.infer<typeof CreateUserSchema>;

// Compose schemas:
const PaginatedResponseSchema = <T extends z.ZodType>(itemSchema: T) =>
  z.object({
    data: z.array(itemSchema),
    meta: z.object({
      page: z.number(),
      total: z.number(),
      pageSize: z.number(),
    }),
  });

const UserListSchema = PaginatedResponseSchema(UserSchema);
type UserListResponse = z.infer<typeof UserListSchema>;
```

**Why Zod + TypeScript is the winning pattern:**
- **Single source of truth** — the schema defines both the type AND the validation rules
- **Runtime safety** — unlike TypeScript types (which are erased at runtime), Zod actually validates data
- **Composable** — schemas can be combined, extended, and transformed
- **Error messages** — Zod provides detailed, structured validation errors
- **Form integration** — works seamlessly with React Hook Form, Formik, etc.

### Type-Safe API Clients

#### The tRPC Pattern

tRPC gives you end-to-end type safety between your API server and client without code generation:

```typescript
// Server (simplified):
import { initTRPC } from "@trpc/server";

const t = initTRPC.create();

const appRouter = t.router({
  user: t.router({
    getById: t.procedure
      .input(z.object({ id: z.string() }))
      .query(async ({ input }) => {
        return await db.user.findUnique({ where: { id: input.id } });
      }),
    create: t.procedure
      .input(CreateUserSchema)
      .mutation(async ({ input }) => {
        return await db.user.create({ data: input });
      }),
  }),
});

export type AppRouter = typeof appRouter;

// Client — types flow automatically:
import type { AppRouter } from "./server";
import { createTRPCClient } from "@trpc/client";

const client = createTRPCClient<AppRouter>({ ... });

// Fully typed — input and output:
const user = await client.user.getById.query({ id: "123" });
// user is typed as User | null

const newUser = await client.user.create.mutate({
  name: "Alice",
  email: "alice@example.com",
  // TypeScript knows exactly what fields are required
});
```

#### Typed Fetch Wrapper

For REST APIs without tRPC, create a typed fetch utility:

```typescript
// Define your API contract:
interface ApiEndpoints {
  "GET /api/users": {
    response: User[];
    query?: { page?: number; limit?: number };
  };
  "GET /api/users/:id": {
    response: User;
    params: { id: string };
  };
  "POST /api/users": {
    response: User;
    body: CreateUserInput;
  };
  "PUT /api/users/:id": {
    response: User;
    params: { id: string };
    body: Partial<User>;
  };
  "DELETE /api/users/:id": {
    response: void;
    params: { id: string };
  };
}

// Type-safe fetch function:
type Method = "GET" | "POST" | "PUT" | "DELETE";

type EndpointKey = keyof ApiEndpoints;

type ExtractMethod<K extends string> = K extends `${infer M} ${string}` ? M : never;

async function api<K extends EndpointKey>(
  endpoint: K,
  options?: Omit<ApiEndpoints[K], "response">
): Promise<ApiEndpoints[K]["response"]> {
  // Implementation builds URL from params, sends body, etc.
  // The important thing is the TYPE SIGNATURE
}

// Usage is fully type-safe:
const users = await api("GET /api/users", { query: { page: 1 } });
// users is User[]

const user = await api("POST /api/users", {
  body: { name: "Alice", email: "alice@example.com", ... },
});
// user is User

await api("DELETE /api/users/:id", { params: { id: "123" } });
```

### Module Augmentation

Module augmentation lets you add properties to existing types from third-party libraries:

```typescript
// Adding custom properties to Express Request:
declare module "express" {
  interface Request {
    user?: User;
    sessionId?: string;
  }
}

// Now you can use req.user in Express handlers without casting

// Adding custom theme properties to styled-components:
declare module "styled-components/native" {
  export interface DefaultTheme {
    colors: {
      primary: string;
      secondary: string;
      background: string;
      text: string;
      error: string;
    };
    spacing: {
      xs: number;
      sm: number;
      md: number;
      lg: number;
      xl: number;
    };
    typography: {
      heading: TextStyle;
      body: TextStyle;
      caption: TextStyle;
    };
  }
}

// Extending React Native's global types:
declare module "react-native" {
  interface ViewStyle {
    // Add custom style properties from a UI library
    gap?: number;
  }
}

// Extending environment variables:
declare module "@env" {
  export const API_URL: string;
  export const API_KEY: string;
  export const ENVIRONMENT: "development" | "staging" | "production";
}

// Adding custom matchers to Jest:
declare module "expect" {
  interface Matchers<R> {
    toBeWithinRange(floor: number, ceiling: number): R;
    toBeValidEmail(): R;
  }
}
```

**Key rule:** module augmentation uses `interface` (which supports declaration merging), not `type` (which doesn't). This is one of the few cases where the interface vs type choice is not a matter of preference.

### Declaration Files (`.d.ts`) and DefinitelyTyped

Declaration files contain only type information — no runtime code. They're how TypeScript understands JavaScript libraries.

```typescript
// custom-module.d.ts
declare module "some-untyped-library" {
  export function doSomething(input: string): number;
  export interface Config {
    verbose: boolean;
    timeout: number;
  }
  export default function init(config: Config): void;
}

// Declaring global variables:
// globals.d.ts
declare const __DEV__: boolean;
declare const __APP_VERSION__: string;

// Declaring modules for non-code imports:
declare module "*.png" {
  const value: number;  // React Native image source
  export default value;
}

declare module "*.svg" {
  import React from "react";
  import { SvgProps } from "react-native-svg";
  const content: React.FC<SvgProps>;
  export default content;
}

declare module "*.json" {
  const value: Record<string, unknown>;
  export default value;
}
```

**DefinitelyTyped** is the community repository of type declarations for JavaScript libraries. When you install `@types/lodash`, you're getting types from DefinitelyTyped. The convention:

```bash
# Library without built-in types:
npm install lodash
npm install --save-dev @types/lodash

# Library WITH built-in types (no @types needed):
npm install zod           # Ships its own types
npm install @tanstack/react-query  # Ships its own types
```

Modern libraries increasingly ship their own types. Check the library's `package.json` for a `types` or `typings` field.

### The Type Registry Pattern

The type registry pattern provides a centralized way to map string keys to types, enabling type-safe lookups across your application:

```typescript
// Define a registry interface:
interface EventRegistry {
  "user:login": { userId: string; timestamp: number };
  "user:logout": { userId: string };
  "cart:add": { productId: string; quantity: number };
  "cart:remove": { productId: string };
  "order:placed": { orderId: string; total: number };
  "order:cancelled": { orderId: string; reason: string };
}

// Type-safe event system built on the registry:
class EventBus {
  private listeners = new Map<string, Set<Function>>();

  on<K extends keyof EventRegistry>(
    event: K,
    handler: (data: EventRegistry[K]) => void
  ): () => void {
    if (!this.listeners.has(event as string)) {
      this.listeners.set(event as string, new Set());
    }
    this.listeners.get(event as string)!.add(handler);
    return () => this.listeners.get(event as string)?.delete(handler);
  }

  emit<K extends keyof EventRegistry>(event: K, data: EventRegistry[K]): void {
    this.listeners.get(event as string)?.forEach(fn => fn(data));
  }
}

const bus = new EventBus();

// Fully type-safe:
bus.on("user:login", (data) => {
  console.log(data.userId);     // OK
  console.log(data.timestamp);  // OK
  console.log(data.email);      // Error: 'email' doesn't exist
});

bus.emit("cart:add", { productId: "123", quantity: 2 });  // OK
bus.emit("cart:add", { productId: "123" });  // Error: missing quantity

// The registry can be extended via declaration merging:
interface EventRegistry {
  "notification:received": { title: string; body: string };
}
// Now "notification:received" is a valid event key
```

---

## 11. TSCONFIG MASTERY

The `tsconfig.json` file controls how TypeScript compiles your code. Most developers copy a config from a blog post and never touch it again. Let's actually understand what the important options do.

### Strictness Options

These are the options that determine how thoroughly TypeScript checks your code. **You want all of these enabled in a mature project.**

#### `strict: true`

This is a shorthand that enables a collection of strict checks. As of TypeScript 5.x, `strict: true` enables:

```jsonc
{
  "compilerOptions": {
    "strict": true
    // Equivalent to enabling all of these:
    // "noImplicitAny": true,
    // "strictNullChecks": true,
    // "strictFunctionTypes": true,
    // "strictBindCallApply": true,
    // "strictPropertyInitialization": true,
    // "noImplicitThis": true,
    // "useUnknownInCatchVariables": true,
    // "alwaysStrict": true
  }
}
```

#### `noImplicitAny`

Errors when TypeScript can't infer a type and would fall back to `any`:

```typescript
// With noImplicitAny: true
function greet(name) {      // Error: Parameter 'name' implicitly has an 'any' type
  console.log(name);
}

function greet(name: string) {  // OK
  console.log(name);
}
```

#### `strictNullChecks`

Makes `null` and `undefined` distinct types that must be handled explicitly:

```typescript
// With strictNullChecks: true
const users: User[] = [];
const first = users[0];  // Type: User (without noUncheckedIndexedAccess)

function find(id: string): User | undefined {
  return users.find(u => u.id === id);
}
const user = find("123");
user.name;  // Error: 'user' is possibly 'undefined'
```

**This is the single most impactful strictness option.** Without it, TypeScript doesn't help with null/undefined bugs at all — which are the most common JavaScript bugs.

#### `noUncheckedIndexedAccess`

This one isn't part of `strict`, but it probably should be. It makes array/object index access return `T | undefined` instead of `T`:

```typescript
// With noUncheckedIndexedAccess: true
const arr = [1, 2, 3];
const first = arr[0];  // Type: number | undefined (not number!)

const obj: Record<string, number> = { a: 1 };
const value = obj["b"];  // Type: number | undefined

// Forces you to handle the undefined case:
if (first !== undefined) {
  console.log(first + 1);  // OK
}
```

**Enable this.** Array out-of-bounds access is a real source of runtime errors.

#### `exactOptionalPropertyTypes`

Makes a distinction between "optional" and "can be undefined":

```typescript
// With exactOptionalPropertyTypes: true
interface User {
  name: string;
  nickname?: string;  // Can be omitted, but NOT set to undefined
}

const a: User = { name: "Alice" };                    // OK
const b: User = { name: "Bob", nickname: "Bobby" };   // OK
const c: User = { name: "Carol", nickname: undefined }; // Error!
```

This is subtle but important — it means "the property may not exist" vs "the property exists but is undefined" are different things, which reflects how `hasOwnProperty` and `in` checks work at runtime.

### Target and Module Options

```jsonc
{
  "compilerOptions": {
    // What JavaScript version to compile to:
    "target": "ES2022",
    // For React Native with Hermes: ES2022 is well-supported
    // For web: depends on your browser support requirements

    // Module system:
    "module": "ESNext",
    // "ESNext" for bundled apps (Webpack, Metro, Vite)
    // "NodeNext" for Node.js apps

    // Module resolution strategy:
    "moduleResolution": "bundler",
    // "bundler" for apps using a bundler (most frontend apps)
    // "NodeNext" for Node.js
    // "node" (legacy, avoid for new projects)

    // Where to find .d.ts files:
    "typeRoots": ["./node_modules/@types", "./types"],

    // What libraries to include:
    "lib": ["ES2022"],
    // Add "DOM" for web projects
    // Omit "DOM" for React Native
  }
}
```

### Path Aliases

Path aliases replace long relative imports with short absolute ones:

```jsonc
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"],
      "@components/*": ["src/components/*"],
      "@hooks/*": ["src/hooks/*"],
      "@utils/*": ["src/utils/*"],
      "@types/*": ["src/types/*"],
      "@api/*": ["src/api/*"],
      "@assets/*": ["src/assets/*"]
    }
  }
}
```

```typescript
// Before:
import { Button } from "../../../components/ui/Button";
import { useAuth } from "../../../../hooks/useAuth";

// After:
import { Button } from "@components/ui/Button";
import { useAuth } from "@hooks/useAuth";
```

**Important:** TypeScript path aliases are for type checking only. Your bundler (Metro for React Native, Webpack/Vite for web) needs matching configuration to resolve these paths at runtime. For React Native, use `babel-plugin-module-resolver`. For Next.js, it reads `tsconfig.json` paths automatically.

### Project References and Composite Builds

Project references split a large TypeScript project into smaller, independently compilable units. This is essential for monorepos.

```jsonc
// packages/shared/tsconfig.json
{
  "compilerOptions": {
    "composite": true,   // Required for project references
    "declaration": true, // Required for composite
    "declarationMap": true,
    "outDir": "dist",
    "rootDir": "src"
  },
  "include": ["src"]
}

// packages/mobile-app/tsconfig.json
{
  "compilerOptions": {
    "outDir": "dist",
    "rootDir": "src"
  },
  "references": [
    { "path": "../shared" }
  ],
  "include": ["src"]
}

// Root tsconfig.json
{
  "files": [],
  "references": [
    { "path": "packages/shared" },
    { "path": "packages/mobile-app" },
    { "path": "packages/web-app" },
    { "path": "packages/api" }
  ]
}
```

Build with `tsc --build` (or `tsc -b`), which:
- Builds projects in dependency order
- Only rebuilds what changed
- Uses declaration files (.d.ts) for cross-package type checking

### Monorepo `tsconfig` Patterns

A typical monorepo has a hierarchy of configs:

```jsonc
// tsconfig.base.json — shared settings for ALL packages
{
  "compilerOptions": {
    "strict": true,
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true
  }
}

// packages/shared/tsconfig.json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "composite": true,
    "outDir": "dist",
    "rootDir": "src"
  },
  "include": ["src"]
}

// packages/mobile-app/tsconfig.json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "jsx": "react-native",
    "types": ["react-native", "jest"]
  },
  "references": [
    { "path": "../shared" }
  ],
  "include": ["src", "app"],
  "exclude": ["node_modules"]
}

// packages/web-app/tsconfig.json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "jsx": "react-jsx",
    "lib": ["ES2022", "DOM", "DOM.Iterable"],
    "types": ["vite/client"]
  },
  "references": [
    { "path": "../shared" }
  ],
  "include": ["src"],
  "exclude": ["node_modules"]
}
```

---

## 12. ENTERPRISE TYPESCRIPT

### Migration Strategies: JavaScript to TypeScript

Migrating a JavaScript codebase to TypeScript is a journey, not a switch flip. Here's the proven approach:

#### Phase 1: Setup (Day 1)

```jsonc
// tsconfig.json — start permissive
{
  "compilerOptions": {
    "allowJs": true,          // Allow .js files alongside .ts
    "checkJs": false,         // Don't check .js files yet
    "strict": false,          // Start without strict mode
    "noImplicitAny": false,   // Allow implicit any
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "esModuleInterop": true,
    "jsx": "react-native",    // or "react-jsx" for web
    "outDir": "dist",
    "skipLibCheck": true
  },
  "include": ["src"],
  "exclude": ["node_modules"]
}
```

Rename your entry file to `.ts`/`.tsx`. Everything else stays as `.js`. Your project should still build.

#### Phase 2: Incremental Conversion (Weeks 1-4)

Convert files one at a time, starting from the leaves (files with no local imports):

1. Utility functions and constants
2. Type definitions (create new `.ts` files for shared types)
3. API clients and data layers
4. Hooks and services
5. Components (starting with simple ones)

**Rule:** Every new file is `.ts`/`.tsx`. Never write new `.js`.

```typescript
// For files you can't convert yet, use JSDoc for gradual typing:
/** @type {import('./types').User} */
const user = getUser();

/** 
 * @param {string} name 
 * @param {number} age 
 * @returns {import('./types').User} 
 */
function createUser(name, age) {
  return { name, age, id: generateId() };
}
```

#### Phase 3: Enable Strictness (Weeks 4-8)

Turn on strict checks one at a time:

```jsonc
// Week 4: Start checking JS files
{ "checkJs": true }

// Week 5: No implicit any
{ "noImplicitAny": true }

// Week 6: Strict null checks (the big one)
{ "strictNullChecks": true }

// Week 7: Remaining strict checks
{ "strict": true }

// Week 8: Extra strictness
{ "noUncheckedIndexedAccess": true }
```

Each step will reveal errors. Fix them. The `strictNullChecks` step is always the biggest — it typically surfaces hundreds of potential null reference bugs that were hiding in your codebase. That's the whole point.

#### Phase 4: Zero JavaScript (Month 3+)

```jsonc
{
  "compilerOptions": {
    "allowJs": false,  // No more .js files allowed
    "strict": true,
    "noUncheckedIndexedAccess": true
  }
}
```

### Progressive Strictness

Not every team can go straight to maximum strictness. Here's a tiered approach:

```
Level 0: TypeScript with "strict": false
  - Basic type annotations
  - Catches obvious type mismatches
  - Lots of implicit 'any'

Level 1: "strict": true
  - No implicit 'any'
  - Strict null checks
  - Proper function types
  - This is where most teams should aim to be

Level 2: "strict": true + extras
  - noUncheckedIndexedAccess
  - exactOptionalPropertyTypes
  - noPropertyAccessFromIndexSignature
  - Very thorough, catches subtle bugs

Level 3: Maximum safety
  - All of Level 2
  - Branded types for IDs
  - Zod for all external data
  - Type tests for public APIs
  - Explicit return types on all exports
  - This is for libraries and critical infrastructure
```

### API Reporting with api-extractor

Microsoft's `api-extractor` analyzes your library's public API surface and generates reports:

```bash
npm install --save-dev @microsoft/api-extractor
```

```jsonc
// api-extractor.json
{
  "mainEntryPointFilePath": "<projectFolder>/dist/index.d.ts",
  "apiReport": {
    "enabled": true,
    "reportFileName": "<unscopedPackageName>.api.md"
  },
  "docModel": {
    "enabled": true
  },
  "dtsRollup": {
    "enabled": true,
    "untrimmedFilePath": "<projectFolder>/dist/<unscopedPackageName>.d.ts"
  }
}
```

**Why this matters:** api-extractor generates a `.api.md` file that shows your package's complete public API. When this file changes in a PR, reviewers can see exactly what API surface changed. It catches accidental API breakage before it ships.

### Dead Code Detection with knip

knip finds unused files, dependencies, and exports in your TypeScript project:

```bash
npx knip
```

```
Unused files (2)
 src/utils/legacyHelper.ts
 src/components/OldButton.tsx

Unused dependencies (1)
 moment

Unused exports (5)
 src/utils/format.ts: formatLegacy
 src/api/client.ts: oldFetch, deprecatedEndpoint
 src/types/index.ts: LegacyUser, OldConfig
```

**Configure in `package.json`:**

```jsonc
{
  "knip": {
    "entry": ["src/index.ts", "src/App.tsx"],
    "project": ["src/**/*.{ts,tsx}"],
    "ignore": ["src/**/*.test.ts", "src/**/*.stories.tsx"],
    "ignoreDependencies": ["@types/jest"]
  }
}
```

Run knip in CI to prevent dead code from accumulating. In a large codebase, dead code is not just clutter — it's confusion. New developers don't know if unused exports are intentionally preserved or accidentally abandoned.

### Type Testing with `tsd`

For library authors, type tests verify that your types work correctly:

```bash
npm install --save-dev tsd
```

```typescript
// index.test-d.ts
import { expectType, expectError, expectAssignable } from "tsd";
import { createUser, UserId, type User } from "./index";

// Test that return type is correct:
expectType<User>(createUser("Alice", 30));

// Test that UserId is not assignable from plain string:
expectError(createUser(123, 30));  // Should error: number not string

// Test that types are assignable:
declare const user: User;
expectAssignable<{ name: string }>(user);

// Test that specific fields exist:
expectType<string>(user.name);
expectType<number>(user.age);
```

Add to your test script:

```jsonc
{
  "scripts": {
    "test:types": "tsd"
  }
}
```

### Performance Optimization

TypeScript compilation can become slow in large projects. Here's how to diagnose and fix it.

#### Avoiding Barrel Files

Barrel files (`index.ts` that re-exports everything) are convenient but terrible for performance:

```typescript
// src/components/index.ts — THE PROBLEM
export { Button } from "./Button";
export { Card } from "./Card";
export { Modal } from "./Modal";
export { Input } from "./Input";
// ... 200 more components

// Importing ONE component:
import { Button } from "@components";
// TypeScript must process ALL 200+ component files to resolve this import
```

**The fix:** Import directly from the source file:

```typescript
import { Button } from "@components/Button";
// TypeScript only processes Button.tsx
```

For IDE auto-imports, configure your editor to prefer direct imports over barrel imports.

#### Explicit Return Types on Exported Functions

When a function doesn't have an explicit return type, TypeScript must compute it from the implementation — and this computation can be expensive if the function body is complex.

```typescript
// Slow: TypeScript must analyze the entire function body
export function buildConfig(env: string) {
  // 100 lines of conditional logic...
  return { /* complex inferred type */ };
}

// Fast: TypeScript uses the declared type directly
export function buildConfig(env: string): AppConfig {
  // TypeScript checks the body against AppConfig
  // but consumers only see 'AppConfig', no inference needed
  return { /* ... */ };
}
```

This is especially impactful for functions that return complex objects or use lots of conditional types.

#### The `--generateTrace` Flag

When compilation is slow and you don't know why:

```bash
tsc --generateTrace ./trace-output
```

This generates trace files you can visualize in Chrome's `chrome://tracing` tool. Look for:
- Types that take a long time to check
- Files that are processed unnecessarily
- Deep type instantiation chains

#### `skipLibCheck: true`

This skips type checking of `.d.ts` files (including `node_modules/@types`). It's almost always worth enabling because:
- It significantly speeds up compilation
- Errors in third-party `.d.ts` files are not your problem to fix
- Your own `.d.ts` files are already checked when their `.ts` source is compiled

#### `isolatedModules: true`

This ensures each file can be compiled independently (required by most build tools like esbuild, SWC, Babel):

```jsonc
{
  "compilerOptions": {
    "isolatedModules": true
  }
}
```

It disallows a few TypeScript patterns that require cross-file analysis:
- `const enum` (use regular `enum` or string unions instead)
- `export =` and `import =` (use ES modules)
- Re-exporting types without `export type` (use `export type { Foo }`)

### Workspace Configurations (pnpm + TypeScript)

A production monorepo setup with pnpm:

```yaml
# pnpm-workspace.yaml
packages:
  - "packages/*"
  - "apps/*"
```

```jsonc
// packages/ui/package.json
{
  "name": "@myapp/ui",
  "version": "1.0.0",
  "main": "src/index.ts",      // Point to source for internal packages
  "types": "src/index.ts",     // TypeScript resolves types from source
  "scripts": {
    "typecheck": "tsc --noEmit",
    "build": "tsup src/index.ts --format esm,cjs --dts"
  }
}

// apps/mobile/package.json
{
  "name": "@myapp/mobile",
  "dependencies": {
    "@myapp/ui": "workspace:*"  // pnpm workspace protocol
  }
}
```

**The `workspace:*` protocol** tells pnpm to link to the local package instead of downloading from npm. Combined with TypeScript project references, this gives you:
- Instant cross-package type checking
- No build step needed during development
- Proper build order in CI

---

## 13. TYPE CHALLENGES

Time to test your understanding. These challenges progress from warm-up to genuinely difficult. Try to solve each one before looking at the solution.

### Challenge 1: `MyPick<T, K>` (Easy)

Implement a type that picks specific properties from an object type (like the built-in `Pick`).

```typescript
// Your implementation:
type MyPick<T, K extends keyof T> = ???

// Tests:
interface Todo {
  title: string;
  description: string;
  completed: boolean;
}

type Expected = { title: string; completed: boolean };
type Result = MyPick<Todo, "title" | "completed">;  // Should equal Expected
```

<details>
<summary>Solution</summary>

```typescript
type MyPick<T, K extends keyof T> = {
  [P in K]: T[P];
};
```

**Why it works:** `K extends keyof T` constrains K to valid keys. The mapped type `[P in K]` iterates over the selected keys, and `T[P]` looks up the value type.

</details>

### Challenge 2: `MyReadonly<T>` (Easy)

Make all properties of a type readonly.

```typescript
type MyReadonly<T> = ???

// Test:
type ReadonlyTodo = MyReadonly<Todo>;
// { readonly title: string; readonly description: string; readonly completed: boolean }
```

<details>
<summary>Solution</summary>

```typescript
type MyReadonly<T> = {
  readonly [P in keyof T]: T[P];
};
```

</details>

### Challenge 3: `MyReturnType<T>` (Medium)

Extract the return type of a function type.

```typescript
type MyReturnType<T> = ???

// Tests:
type A = MyReturnType<() => string>;           // string
type B = MyReturnType<(x: number) => boolean>; // boolean
type C = MyReturnType<() => Promise<User>>;    // Promise<User>
```

<details>
<summary>Solution</summary>

```typescript
type MyReturnType<T extends (...args: any[]) => any> = 
  T extends (...args: any[]) => infer R ? R : never;
```

**Why it works:** The constraint `extends (...args: any[]) => any` ensures T is a function. The conditional type with `infer R` captures the return type.

</details>

### Challenge 4: `DeepReadonly<T>` (Medium)

Make all properties of a type readonly, recursively.

```typescript
type DeepReadonly<T> = ???

// Test:
type Nested = {
  a: string;
  b: {
    c: number;
    d: {
      e: boolean;
    };
  };
  f: string[];
};

type ReadonlyNested = DeepReadonly<Nested>;
// All properties at all levels are readonly
// readonlyNested.b.d.e = false; // Error!
```

<details>
<summary>Solution</summary>

```typescript
type DeepReadonly<T> = T extends Function
  ? T
  : T extends object
    ? { readonly [K in keyof T]: DeepReadonly<T[K]> }
    : T;
```

**Why it works:** We check if T is a function (don't recurse into functions). If it's an object, we make all properties readonly and recurse. Primitives are returned as-is.

</details>

### Challenge 5: `Flatten<T>` (Medium)

Flatten a nested array type by one level.

```typescript
type Flatten<T> = ???

// Tests:
type A = Flatten<[1, 2, [3, 4]]>;           // [1, 2, 3, 4]
type B = Flatten<[[1], [2], [3]]>;           // [1, 2, 3]
type C = Flatten<[1, [2, [3]]]>;             // [1, 2, [3]] (only one level!)
```

<details>
<summary>Solution</summary>

```typescript
type Flatten<T extends any[]> = T extends [infer First, ...infer Rest]
  ? First extends any[]
    ? [...First, ...Flatten<Rest>]
    : [First, ...Flatten<Rest>]
  : [];
```

**Why it works:** We recursively process the tuple. For each element, if it's an array, we spread it. Otherwise, we keep it as-is. The recursion processes the rest of the tuple.

</details>

### Challenge 6: `TupleToUnion<T>` (Medium)

Convert a tuple type to a union of its element types.

```typescript
type TupleToUnion<T> = ???

// Tests:
type A = TupleToUnion<[string, number, boolean]>;  // string | number | boolean
type B = TupleToUnion<["a", "b", "c"]>;             // "a" | "b" | "c"
type C = TupleToUnion<[]>;                           // never
```

<details>
<summary>Solution</summary>

```typescript
type TupleToUnion<T extends any[]> = T[number];
```

**Why it works:** Indexing a tuple type with `number` produces a union of all element types. This is one of those TypeScript tricks that feels like magic once you see it.

</details>

### Challenge 7: `Last<T>` (Medium)

Get the last element type from a tuple.

```typescript
type Last<T extends any[]> = ???

// Tests:
type A = Last<[1, 2, 3]>;     // 3
type B = Last<["a"]>;          // "a"
type C = Last<[true, false]>;  // false
```

<details>
<summary>Solution</summary>

```typescript
type Last<T extends any[]> = T extends [...infer _, infer L] ? L : never;
```

**Why it works:** The rest element `...infer _` captures everything except the last element, and `infer L` captures the last one.

</details>

### Challenge 8: `StringToUnion<T>` (Hard)

Convert a string literal type to a union of its characters.

```typescript
type StringToUnion<T extends string> = ???

// Tests:
type A = StringToUnion<"hello">;  // "h" | "e" | "l" | "o"
type B = StringToUnion<"">;       // never
type C = StringToUnion<"abc">;    // "a" | "b" | "c"
```

<details>
<summary>Solution</summary>

```typescript
type StringToUnion<T extends string> = 
  T extends `${infer First}${infer Rest}`
    ? First | StringToUnion<Rest>
    : never;
```

**Why it works:** Template literal types with `infer` can destructure strings. `First` captures the first character, `Rest` captures the remainder. We union the first character with the recursive result on the rest.

</details>

### Challenge 9: `IsEqual<A, B>` (Hard)

Check if two types are exactly equal (not just assignable).

```typescript
type IsEqual<A, B> = ???

// Tests:
type A = IsEqual<string, string>;        // true
type B = IsEqual<string, number>;        // false
type C = IsEqual<{ a: 1 }, { a: 1 }>;   // true
type D = IsEqual<any, string>;           // false (any is not string!)
type E = IsEqual<never, never>;          // true
type F = IsEqual<1, 1 | 2>;             // false
```

<details>
<summary>Solution</summary>

```typescript
type IsEqual<A, B> = 
  (<T>() => T extends A ? 1 : 2) extends (<T>() => T extends B ? 1 : 2)
    ? true
    : false;
```

**Why it works:** This is a well-known trick. Two generic conditional types are only compatible if their conditions are identical. This correctly handles edge cases with `any`, `never`, and union types that simpler approaches (like mutual assignability) get wrong.

</details>

### Challenge 10: `ParseQueryString<T>` (Advanced)

Parse a URL query string into an object type.

```typescript
type ParseQueryString<T extends string> = ???

// Tests:
type A = ParseQueryString<"name=Alice&age=30">;
// { name: "Alice"; age: "30" }

type B = ParseQueryString<"page=1&limit=10&sort=desc">;
// { page: "1"; limit: "10"; sort: "desc" }

type C = ParseQueryString<"">;
// {}
```

<details>
<summary>Solution</summary>

```typescript
type ParsePair<T extends string> = 
  T extends `${infer Key}=${infer Value}`
    ? { [K in Key]: Value }
    : {};

type MergeIntersection<T> = { [K in keyof T]: T[K] };

type ParseQueryString<T extends string> = 
  T extends ""
    ? {}
    : T extends `${infer Pair}&${infer Rest}`
      ? MergeIntersection<ParsePair<Pair> & ParseQueryString<Rest>>
      : MergeIntersection<ParsePair<T>>;
```

**Why it works:** We recursively split on `&` to get individual key-value pairs. Each pair is split on `=` to extract the key and value. The results are intersected together, and `MergeIntersection` flattens the intersection into a single object type for readability.

**This is type-level string parsing.** The same technique can parse route paths, CSS values, configuration strings, or any structured string format.

</details>

---

## CLOSING THOUGHTS

TypeScript is not a tax on your productivity — it's an investment that pays compound interest. Every type you write is a bug you prevent. Every generic is a pattern you encoded once instead of debugging in five places. Every discriminated union is an impossible state that will never make it to production.

The journey from "TypeScript beginner who annotates everything with `: any`" to "TypeScript architect who designs type-safe systems" is not about memorizing syntax. It's about building mental models:

- **Types are sets of values.** Union is set union. Intersection is set intersection. `never` is the empty set.
- **The compiler is your pair programmer.** Don't fight it — give it enough information to help you. When you get an error you don't understand, the error is almost always right. Read it carefully.
- **Make impossible states unrepresentable.** If your types allow a state that your app should never be in, tighten the types until they don't.
- **The best type is the one you didn't write.** Let inference do its job. Annotate at boundaries, not everywhere.
- **Types are documentation that never goes stale.** A function signature with good types tells you more than any JSDoc comment.

When you find yourself spending time on type gymnastics, ask: "Am I encoding a real constraint, or am I showing off?" The best TypeScript code is not the cleverest — it's the one that makes the most common mistakes impossible while staying readable to the rest of your team.

The tools in this chapter — discriminated unions, branded types, Zod schemas, conditional types, mapped types — are your arsenal. Use them wisely, and you'll build systems that are not just correct today, but that *stay* correct as they evolve.

---

> **Next up:** [Chapter 0e] — where we take these TypeScript foundations and apply them to React component patterns, hooks, and the rendering model. The type system meets the UI layer.
