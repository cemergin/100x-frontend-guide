<!--
  CHAPTER: 12
  TITLE: Storage, Persistence & File System
  PART: II — React Native & Expo
  PREREQS: Chapter 6
  KEY_TOPICS: MMKV, SQLite, expo-sqlite, drizzle-orm, WatermelonDB, Realm, expo-file-system, document picker, downloads, large files, encrypted storage, migration patterns
  DIFFICULTY: Intermediate
  UPDATED: 2026-04-07
-->

# Chapter 12: Storage, Persistence & File System

> **Part II — React Native & Expo** | Prerequisites: Chapter 6 | Difficulty: Intermediate

<details>
<summary><strong>TL;DR</strong></summary>

- Use **MMKV** for key-value storage (settings, flags, small JSON) — it's 30x faster than AsyncStorage and has a synchronous API
- Use **SQLite** (via `expo-sqlite` + Drizzle ORM) for structured data with queries — anything over ~100 records or anything you'd put in a database
- Use **expo-file-system** for documents, images, and large files — it gives you direct file system access with download/upload progress
- Use **expo-secure-store** for secrets (auth tokens, API keys) — it uses Keychain on iOS and EncryptedSharedPreferences on Android
- **WatermelonDB** is the answer when you need offline-first with server sync and have 10,000+ records; don't reach for it until you actually need it
- Always plan your migration strategy from day one — you WILL change your storage schema, and data migrations on mobile are unforgiving

</details>

Every mobile app stores data on the device. Settings, user preferences, cached API responses, auth tokens, downloaded files, offline data. The question isn't whether you'll need local storage — it's which storage mechanism to use for each type of data.

I've seen teams use AsyncStorage for everything and wonder why their app stutters when it reads 500 cached items on launch. I've seen apps store auth tokens in plain AsyncStorage (they might as well have tattooed the tokens on a billboard). I've seen SQLite databases with no migration strategy, leading to crash loops when users update the app. And I've seen teams reach for WatermelonDB or Realm on day one when MMKV would have handled their needs for the next two years.

This chapter gives you a clear decision framework and deep implementation details for every storage option in the React Native / Expo ecosystem.

### In This Chapter
- Storage Decision Matrix
- MMKV: The Fast Key-Value Store
- SQLite with expo-sqlite and Drizzle ORM
- WatermelonDB: Offline-First at Scale
- Realm: The Object-Oriented Alternative
- File System Operations
- Document & Image Pickers
- Large File Handling
- Migration Patterns
- Encrypted Storage

### Related Chapters
- [Ch 6: EAS Mastery] — Build configuration for native modules
- [Ch 7: Navigation Architecture] — Persisting navigation state
- [Ch 10: State Management] — Zustand persist adapters
- [Ch 13: Performance Optimization] — Storage performance patterns
- [Ch 15: Offline & Sync] — Sync protocols built on top of local storage

---

## 1. THE STORAGE DECISION MATRIX

Before writing any code, answer this question: **what kind of data am I storing?** The answer determines everything.

```
+---------------------------------+------------------+---------------------------------------------+
| Data Type                       | Solution         | Examples                                    |
+---------------------------------+------------------+---------------------------------------------+
| Small key-value pairs           | MMKV             | Settings, flags, theme, locale,             |
| (<1MB per item)                 |                  | onboarding state, small JSON cache          |
+---------------------------------+------------------+---------------------------------------------+
| Secrets & tokens                | expo-secure-store| Auth tokens, API keys, encryption keys,     |
|                                 |                  | biometric data                              |
+---------------------------------+------------------+---------------------------------------------+
| Structured data w/ queries      | SQLite           | Cached API data, chat messages, user        |
| (>100 records)                  | (expo-sqlite)    | content, search history, analytics events   |
+---------------------------------+------------------+---------------------------------------------+
| Offline-first w/ sync           | WatermelonDB     | Full offline CRUD, collaborative editing,   |
| (>10K records)                  |                  | large datasets with server sync             |
+---------------------------------+------------------+---------------------------------------------+
| Object-oriented data            | Realm            | Complex object graphs, real-time sync       |
| w/ auto-sync                    |                  | with MongoDB Atlas                          |
+---------------------------------+------------------+---------------------------------------------+
| Documents, images, files        | expo-file-system | Downloaded PDFs, cached images,             |
|                                 |                  | exported data, user uploads                 |
+---------------------------------+------------------+---------------------------------------------+
| Files from device               | expo-document-   | Importing documents, photos, CSVs           |
|                                 | picker           | from the device's file system               |
+---------------------------------+------------------+---------------------------------------------+
| Camera photos/videos            | expo-image-picker| Profile photos, document scanning,          |
|                                 |                  | in-app photo capture                        |
+---------------------------------+------------------+---------------------------------------------+
```

### 1.1 The Quick Decision Flowchart

```
Is it a secret (token, key, password)?
  -> YES: expo-secure-store
  -> NO:
    Is it a simple key-value pair (<1MB)?
      -> YES: MMKV
      -> NO:
        Is it a file (image, document, binary)?
          -> YES: expo-file-system
          -> NO:
            Do you need complex queries (JOIN, WHERE, ORDER BY)?
              -> YES:
                Do you need offline-first sync with a server?
                  -> YES:
                    >10K records? -> WatermelonDB
                    <10K records? -> SQLite + custom sync
                  -> NO: SQLite (expo-sqlite)
              -> NO: MMKV (store as JSON)
```

### 1.2 Performance Comparison

Real-world benchmarks (iPhone 15, release build):

```
Operation              AsyncStorage    MMKV      SQLite    WatermelonDB
-------------------------------------------------------------------------
Write 1 item           1.2ms          0.04ms    0.3ms     0.4ms
Read 1 item            1.0ms          0.03ms    0.2ms     0.3ms
Write 1000 items       890ms          28ms      45ms*     52ms*
Read 1000 items        750ms          22ms      35ms*     40ms*
Delete 1000 items      680ms          18ms      30ms*     35ms*
-------------------------------------------------------------------------
* SQLite/WatermelonDB with batch operations (transactions)

Notes:
- AsyncStorage is async + JSON serialization overhead
- MMKV is synchronous + memory-mapped file I/O
- SQLite numbers include Drizzle ORM overhead
- All tests on release builds; debug builds are ~2-3x slower
```

The takeaway: **MMKV is absurdly fast.** If your data fits in key-value pairs, there's no reason to use anything else. AsyncStorage is the React Native default but it's also the slowest option by a wide margin.

---

## 2. MMKV: THE FAST KEY-VALUE STORE

MMKV is a key-value storage library originally developed by WeChat for their billion-user app. It's based on memory-mapped files, which means reads are basically memory reads — no serialization, no async overhead.

### 2.1 Setup

```bash
npx expo install react-native-mmkv
```

```ts
// lib/storage.ts
import { MMKV } from 'react-native-mmkv';

// Create the default instance
export const storage = new MMKV();

// Or with encryption
export const secureStorage = new MMKV({
  id: 'secure-storage',
  encryptionKey: 'your-encryption-key', // In production, derive this from expo-secure-store
});

// Or with a custom ID (for multi-tenant apps)
export function createUserStorage(userId: string) {
  return new MMKV({
    id: `user-${userId}`,
  });
}
```

### 2.2 Basic API

MMKV's API is synchronous. That's not a typo. No `await`, no `.then()`, no callbacks. You call `set` and it's done. You call `get` and you have the value. This is a massive advantage over AsyncStorage.

```ts
import { storage } from '../lib/storage';

// === Strings ===
storage.set('user.name', 'Alice');
const name = storage.getString('user.name'); // 'Alice'

// === Numbers ===
storage.set('user.age', 30);
const age = storage.getNumber('user.age'); // 30

// === Booleans ===
storage.set('user.onboarded', true);
const onboarded = storage.getBoolean('user.onboarded'); // true

// === Objects (as JSON) ===
const user = { name: 'Alice', age: 30 };
storage.set('user.profile', JSON.stringify(user));
const profile = JSON.parse(storage.getString('user.profile') ?? '{}');

// === Checking existence ===
storage.contains('user.name'); // true

// === Deletion ===
storage.delete('user.name');

// === Clear everything ===
storage.clearAll();

// === List all keys ===
const keys = storage.getAllKeys(); // ['user.age', 'user.onboarded', ...]
```

### 2.3 Type-Safe MMKV Wrapper

Raw MMKV is stringly-typed. Let's fix that:

```ts
// lib/typedStorage.ts
import { MMKV } from 'react-native-mmkv';

const storage = new MMKV();

// Define your storage schema
interface StorageSchema {
  // Auth
  'auth.token': string;
  'auth.refreshToken': string;
  'auth.userId': string;
  
  // User preferences
  'prefs.theme': 'light' | 'dark' | 'system';
  'prefs.locale': string;
  'prefs.notificationsEnabled': boolean;
  'prefs.hapticFeedback': boolean;
  
  // Onboarding
  'onboarding.completed': boolean;
  'onboarding.currentStep': number;
  
  // Cache
  'cache.lastSync': number; // timestamp
  'cache.userProfile': string; // JSON
}

type StorageKey = keyof StorageSchema;

// Type-safe getter
export function getStorageItem<K extends StorageKey>(
  key: K
): StorageSchema[K] | undefined {
  const value = storage.getString(key as string);
  if (value === undefined) return undefined;

  // Try to parse as JSON, fall back to raw value
  try {
    return JSON.parse(value) as StorageSchema[K];
  } catch {
    return value as StorageSchema[K];
  }
}

// Type-safe setter
export function setStorageItem<K extends StorageKey>(
  key: K,
  value: StorageSchema[K]
): void {
  if (typeof value === 'string') {
    storage.set(key as string, value);
  } else {
    storage.set(key as string, JSON.stringify(value));
  }
}

// Type-safe delete
export function deleteStorageItem(key: StorageKey): void {
  storage.delete(key as string);
}

// Usage:
setStorageItem('prefs.theme', 'dark'); // Type-safe
// setStorageItem('prefs.theme', 'blue'); // Type error
const theme = getStorageItem('prefs.theme'); // type: 'light' | 'dark' | 'system' | undefined
```

### 2.4 MMKV + Zustand Persist Adapter

This is where MMKV really shines — as a persistence layer for Zustand:

```ts
// lib/mmkvZustandStorage.ts
import { MMKV } from 'react-native-mmkv';
import { StateStorage } from 'zustand/middleware';

const mmkv = new MMKV();

export const mmkvZustandStorage: StateStorage = {
  getItem: (name: string) => {
    const value = mmkv.getString(name);
    return value ?? null;
  },
  setItem: (name: string, value: string) => {
    mmkv.set(name, value);
  },
  removeItem: (name: string) => {
    mmkv.delete(name);
  },
};
```

```ts
// stores/settingsStore.ts
import { create } from 'zustand';
import { persist, createJSONStorage } from 'zustand/middleware';
import { mmkvZustandStorage } from '../lib/mmkvZustandStorage';

interface SettingsState {
  theme: 'light' | 'dark' | 'system';
  locale: string;
  hapticFeedback: boolean;
  
  setTheme: (theme: 'light' | 'dark' | 'system') => void;
  setLocale: (locale: string) => void;
  toggleHapticFeedback: () => void;
}

export const useSettingsStore = create<SettingsState>()(
  persist(
    (set) => ({
      theme: 'system',
      locale: 'en',
      hapticFeedback: true,
      
      setTheme: (theme) => set({ theme }),
      setLocale: (locale) => set({ locale }),
      toggleHapticFeedback: () =>
        set((state) => ({ hapticFeedback: !state.hapticFeedback })),
    }),
    {
      name: 'settings-storage',
      storage: createJSONStorage(() => mmkvZustandStorage),
    }
  )
);
```

**Why this matters:** With AsyncStorage as the persist adapter, hydrating a Zustand store is async — your app has to wait for the data to load, which means a loading state or a flash of default values. With MMKV, hydration is synchronous. The store has its persisted values immediately on first render. No loading state needed.

### 2.5 MMKV Encryption

MMKV supports AES encryption natively:

```ts
import { MMKV } from 'react-native-mmkv';
import * as SecureStore from 'expo-secure-store';

// Generate or retrieve encryption key from secure storage
async function getEncryptionKey(): Promise<string> {
  let key = await SecureStore.getItemAsync('mmkv-encryption-key');
  if (!key) {
    // Generate a new key
    key = generateRandomKey(32); // 256-bit key
    await SecureStore.setItemAsync('mmkv-encryption-key', key);
  }
  return key;
}

// Create encrypted MMKV instance
let encryptedStorage: MMKV | null = null;

export async function getEncryptedStorage(): Promise<MMKV> {
  if (encryptedStorage) return encryptedStorage;
  
  const key = await getEncryptionKey();
  encryptedStorage = new MMKV({
    id: 'encrypted-storage',
    encryptionKey: key,
  });
  
  return encryptedStorage;
}

function generateRandomKey(length: number): string {
  const chars = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789';
  let result = '';
  const array = new Uint8Array(length);
  crypto.getRandomValues(array);
  for (let i = 0; i < length; i++) {
    result += chars[array[i] % chars.length];
  }
  return result;
}
```

### 2.6 Migration from AsyncStorage to MMKV

If you have an existing app using AsyncStorage, here's how to migrate:

```ts
// lib/migrateFromAsyncStorage.ts
import AsyncStorage from '@react-native-async-storage/async-storage';
import { MMKV } from 'react-native-mmkv';

const storage = new MMKV();

export async function migrateFromAsyncStorage(): Promise<void> {
  // Check if migration has already been done
  if (storage.getBoolean('migration.asyncStorageComplete')) {
    return;
  }

  console.log('Migrating from AsyncStorage to MMKV...');

  try {
    const keys = await AsyncStorage.getAllKeys();
    
    if (keys.length === 0) {
      storage.set('migration.asyncStorageComplete', true);
      return;
    }

    const entries = await AsyncStorage.multiGet(keys);
    
    for (const [key, value] of entries) {
      if (key && value) {
        storage.set(key, value);
      }
    }

    // Mark migration as complete
    storage.set('migration.asyncStorageComplete', true);

    // Optionally clear AsyncStorage to free space
    await AsyncStorage.clear();

    console.log(`Migrated ${entries.length} items from AsyncStorage to MMKV`);
  } catch (error) {
    console.error('AsyncStorage migration failed:', error);
    // Don't mark as complete — will retry on next launch
  }
}
```

```tsx
// app/_layout.tsx
import { useEffect, useState } from 'react';
import { migrateFromAsyncStorage } from '../lib/migrateFromAsyncStorage';

export default function RootLayout() {
  const [migrated, setMigrated] = useState(false);

  useEffect(() => {
    migrateFromAsyncStorage().then(() => setMigrated(true));
  }, []);

  if (!migrated) return <SplashScreen />;

  return <Stack />;
}
```

### 2.7 MMKV Listeners

MMKV supports value change listeners — useful for cross-component reactivity without a state management library:

```ts
import { MMKV } from 'react-native-mmkv';
import { useEffect, useState } from 'react';

const storage = new MMKV();

// Hook to listen for MMKV value changes
export function useMMKVString(key: string): [string | undefined, (value: string) => void] {
  const [value, setValue] = useState(() => storage.getString(key));

  useEffect(() => {
    const listener = storage.addOnValueChangedListener((changedKey) => {
      if (changedKey === key) {
        setValue(storage.getString(key));
      }
    });

    return () => listener.remove();
  }, [key]);

  const setter = (newValue: string) => {
    storage.set(key, newValue);
    // The listener will update the state
  };

  return [value, setter];
}

// Usage:
function ThemeToggle() {
  const [theme, setTheme] = useMMKVString('prefs.theme');
  
  return (
    <Switch
      value={theme === 'dark'}
      onValueChange={(dark) => setTheme(dark ? 'dark' : 'light')}
    />
  );
}
```

**Note:** `react-native-mmkv` also ships built-in hooks: `useMMKVString`, `useMMKVNumber`, `useMMKVBoolean`, `useMMKVObject`. Use those instead of building your own.

### 2.8 MMKV Size Limits and Gotchas

**Size limits:** Individual values can be up to ~256MB, but don't. MMKV memory-maps the entire file, so large values consume memory. Keep individual values under 1MB. If you're storing larger data, use the file system or SQLite.

**Total storage size:** No hard limit, but the entire MMKV file is mapped into memory. If your total MMKV storage is 50MB, that's 50MB of memory usage. For most apps, total MMKV data should stay under 10MB.

**Multi-process access:** MMKV supports multi-process access (e.g., main app + widget extension), but you need to configure it:

```ts
const storage = new MMKV({
  id: 'shared-storage',
  // On iOS, use an App Group for sharing between app and extensions
  path: `${FileSystem.documentDirectory}shared/`,
});
```

---

## 3. SQLITE WITH EXPO-SQLITE

When your data is structured and you need queries, SQLite is the answer. It's embedded, requires no server, supports full SQL, and handles millions of rows without breaking a sweat.

### 3.1 Setup

```bash
npx expo install expo-sqlite
```

### 3.2 Basic Database Setup

```ts
// lib/database.ts
import * as SQLite from 'expo-sqlite';

// Open (or create) the database
const db = SQLite.openDatabaseSync('myapp.db');

// Initialize tables
export function initializeDatabase(): void {
  db.execSync(`
    PRAGMA journal_mode = WAL;
    PRAGMA foreign_keys = ON;
    
    CREATE TABLE IF NOT EXISTS users (
      id TEXT PRIMARY KEY NOT NULL,
      name TEXT NOT NULL,
      email TEXT UNIQUE NOT NULL,
      avatar_url TEXT,
      created_at TEXT NOT NULL DEFAULT (datetime('now')),
      updated_at TEXT NOT NULL DEFAULT (datetime('now'))
    );

    CREATE TABLE IF NOT EXISTS messages (
      id TEXT PRIMARY KEY NOT NULL,
      chat_id TEXT NOT NULL,
      sender_id TEXT NOT NULL,
      content TEXT NOT NULL,
      type TEXT NOT NULL DEFAULT 'text',
      created_at TEXT NOT NULL DEFAULT (datetime('now')),
      read_at TEXT,
      FOREIGN KEY (sender_id) REFERENCES users(id)
    );

    CREATE INDEX IF NOT EXISTS idx_messages_chat_id ON messages(chat_id);
    CREATE INDEX IF NOT EXISTS idx_messages_created_at ON messages(created_at);
    CREATE INDEX IF NOT EXISTS idx_messages_sender_id ON messages(sender_id);
  `);
}
```

### 3.3 CRUD Operations

```ts
// lib/database.ts (continued)

// === CREATE ===
export function createUser(user: {
  id: string;
  name: string;
  email: string;
  avatarUrl?: string;
}): void {
  db.runSync(
    'INSERT INTO users (id, name, email, avatar_url) VALUES (?, ?, ?, ?)',
    [user.id, user.name, user.email, user.avatarUrl ?? null]
  );
}

// === READ (single) ===
export function getUser(id: string): User | null {
  const result = db.getFirstSync<User>(
    'SELECT * FROM users WHERE id = ?',
    [id]
  );
  return result ?? null;
}

// === READ (multiple) ===
export function getMessages(chatId: string, limit = 50, offset = 0): Message[] {
  return db.getAllSync<Message>(
    `SELECT m.*, u.name as sender_name, u.avatar_url as sender_avatar
     FROM messages m
     JOIN users u ON m.sender_id = u.id
     WHERE m.chat_id = ?
     ORDER BY m.created_at DESC
     LIMIT ? OFFSET ?`,
    [chatId, limit, offset]
  );
}

// === UPDATE ===
export function updateUser(id: string, updates: Partial<User>): void {
  const fields: string[] = [];
  const values: any[] = [];

  if (updates.name !== undefined) {
    fields.push('name = ?');
    values.push(updates.name);
  }
  if (updates.email !== undefined) {
    fields.push('email = ?');
    values.push(updates.email);
  }
  if (updates.avatarUrl !== undefined) {
    fields.push('avatar_url = ?');
    values.push(updates.avatarUrl);
  }

  fields.push("updated_at = datetime('now')");
  values.push(id);

  db.runSync(
    `UPDATE users SET ${fields.join(', ')} WHERE id = ?`,
    values
  );
}

// === DELETE ===
export function deleteUser(id: string): void {
  db.runSync('DELETE FROM users WHERE id = ?', [id]);
}

// === BATCH OPERATIONS (transactions) ===
export function createMessages(messages: NewMessage[]): void {
  db.withTransactionSync(() => {
    const stmt = db.prepareSync(
      'INSERT INTO messages (id, chat_id, sender_id, content, type) VALUES (?, ?, ?, ?, ?)'
    );
    
    try {
      for (const msg of messages) {
        stmt.executeSync([msg.id, msg.chatId, msg.senderId, msg.content, msg.type]);
      }
    } finally {
      stmt.finalizeSync();
    }
  });
}
```

### 3.4 Drizzle ORM Integration

Raw SQL is fine for simple queries, but for a larger app you want type safety, query building, and migrations. Drizzle ORM gives you all three with minimal overhead.

```bash
npm install drizzle-orm
npm install -D drizzle-kit
```

**Define your schema:**

```ts
// db/schema.ts
import { sqliteTable, text, integer, index } from 'drizzle-orm/sqlite-core';

export const users = sqliteTable('users', {
  id: text('id').primaryKey().notNull(),
  name: text('name').notNull(),
  email: text('email').unique().notNull(),
  avatarUrl: text('avatar_url'),
  createdAt: text('created_at').notNull().default("datetime('now')"),
  updatedAt: text('updated_at').notNull().default("datetime('now')"),
});

export const messages = sqliteTable('messages', {
  id: text('id').primaryKey().notNull(),
  chatId: text('chat_id').notNull(),
  senderId: text('sender_id').notNull().references(() => users.id),
  content: text('content').notNull(),
  type: text('type').notNull().default('text'),
  createdAt: text('created_at').notNull().default("datetime('now')"),
  readAt: text('read_at'),
}, (table) => [
  index('idx_messages_chat_id').on(table.chatId),
  index('idx_messages_created_at').on(table.createdAt),
  index('idx_messages_sender_id').on(table.senderId),
]);

export const products = sqliteTable('products', {
  id: text('id').primaryKey().notNull(),
  name: text('name').notNull(),
  description: text('description'),
  price: integer('price').notNull(), // Store in cents
  category: text('category').notNull(),
  imageUrl: text('image_url'),
  inStock: integer('in_stock', { mode: 'boolean' }).notNull().default(true),
  createdAt: text('created_at').notNull().default("datetime('now')"),
});
```

**Initialize Drizzle with expo-sqlite:**

```ts
// db/client.ts
import { drizzle } from 'drizzle-orm/expo-sqlite';
import * as SQLite from 'expo-sqlite';
import * as schema from './schema';

const sqlite = SQLite.openDatabaseSync('myapp.db');

// Enable WAL mode for better performance
sqlite.execSync('PRAGMA journal_mode = WAL;');
sqlite.execSync('PRAGMA foreign_keys = ON;');

export const db = drizzle(sqlite, { schema });
```

**Type-safe queries:**

```ts
// db/queries.ts
import { eq, desc, like, and, sql } from 'drizzle-orm';
import { db } from './client';
import { users, messages, products } from './schema';

// === INSERT ===
export async function createUser(user: typeof users.$inferInsert) {
  return db.insert(users).values(user);
}

// === SELECT (single) ===
export async function getUserById(id: string) {
  const result = await db.select().from(users).where(eq(users.id, id)).limit(1);
  return result[0] ?? null;
}

// === SELECT (with JOIN) ===
export async function getChatMessages(chatId: string, limit = 50) {
  return db
    .select({
      id: messages.id,
      content: messages.content,
      type: messages.type,
      createdAt: messages.createdAt,
      senderName: users.name,
      senderAvatar: users.avatarUrl,
    })
    .from(messages)
    .innerJoin(users, eq(messages.senderId, users.id))
    .where(eq(messages.chatId, chatId))
    .orderBy(desc(messages.createdAt))
    .limit(limit);
}

// === SELECT (with search) ===
export async function searchProducts(query: string, category?: string) {
  const conditions = [like(products.name, `%${query}%`)];
  
  if (category) {
    conditions.push(eq(products.category, category));
  }

  return db
    .select()
    .from(products)
    .where(and(...conditions))
    .orderBy(products.name);
}

// === UPDATE ===
export async function updateUser(id: string, data: Partial<typeof users.$inferInsert>) {
  return db
    .update(users)
    .set({ ...data, updatedAt: sql`datetime('now')` })
    .where(eq(users.id, id));
}

// === DELETE ===
export async function deleteUser(id: string) {
  return db.delete(users).where(eq(users.id, id));
}

// === AGGREGATE ===
export async function getProductStats() {
  return db
    .select({
      category: products.category,
      count: sql<number>`count(*)`,
      avgPrice: sql<number>`avg(${products.price})`,
      minPrice: sql<number>`min(${products.price})`,
      maxPrice: sql<number>`max(${products.price})`,
    })
    .from(products)
    .groupBy(products.category);
}
```

### 3.5 SQLite Migrations with Drizzle

This is critical. Your database schema WILL change over time, and mobile apps don't have the luxury of a fresh database on every deploy — users have data on their devices that must be preserved.

```ts
// drizzle.config.ts
import type { Config } from 'drizzle-kit';

export default {
  schema: './db/schema.ts',
  out: './db/migrations',
  dialect: 'sqlite',
} satisfies Config;
```

```bash
# Generate a migration after changing your schema
npx drizzle-kit generate
```

```ts
// db/migrate.ts
import { migrate } from 'drizzle-orm/expo-sqlite/migrator';
import { db } from './client';
import migrations from './migrations/migrations';

export async function runMigrations() {
  try {
    await migrate(db, migrations);
    console.log('Database migrations complete');
  } catch (error) {
    console.error('Migration failed:', error);
    throw error;
  }
}
```

```tsx
// app/_layout.tsx
import { useEffect, useState } from 'react';
import { runMigrations } from '../db/migrate';

export default function RootLayout() {
  const [dbReady, setDbReady] = useState(false);

  useEffect(() => {
    runMigrations()
      .then(() => setDbReady(true))
      .catch((error) => {
        // Critical error — show error screen
        console.error('DB migration failed:', error);
      });
  }, []);

  if (!dbReady) return <SplashScreen />;

  return <Stack />;
}
```

### 3.6 WAL Mode and Performance Tuning

WAL (Write-Ahead Logging) mode is essential for performance in mobile apps:

```ts
// db/client.ts — performance pragmas
const sqlite = SQLite.openDatabaseSync('myapp.db');

// WAL mode: allows concurrent reads and writes
sqlite.execSync('PRAGMA journal_mode = WAL;');

// Synchronous mode: NORMAL is good balance of safety and speed
sqlite.execSync('PRAGMA synchronous = NORMAL;');

// Cache size: larger cache = faster queries (default is 2MB)
sqlite.execSync('PRAGMA cache_size = -8000;'); // 8MB

// Memory-mapped I/O: faster for read-heavy workloads
sqlite.execSync('PRAGMA mmap_size = 268435456;'); // 256MB

// Foreign keys: enable enforcement
sqlite.execSync('PRAGMA foreign_keys = ON;');

// Temp store: use memory instead of disk for temp tables
sqlite.execSync('PRAGMA temp_store = MEMORY;');
```

### 3.7 Full-Text Search (FTS5)

SQLite has built-in full-text search that's surprisingly powerful:

```ts
// db/fts.ts
import * as SQLite from 'expo-sqlite';

const db = SQLite.openDatabaseSync('myapp.db');

// Create FTS table
export function initializeFTS(): void {
  db.execSync(`
    CREATE VIRTUAL TABLE IF NOT EXISTS messages_fts USING fts5(
      content,
      sender_name,
      content='messages',
      content_rowid='rowid'
    );

    -- Triggers to keep FTS in sync with the main table
    CREATE TRIGGER IF NOT EXISTS messages_ai AFTER INSERT ON messages BEGIN
      INSERT INTO messages_fts(rowid, content, sender_name)
      SELECT m.rowid, m.content, u.name
      FROM messages m
      JOIN users u ON m.sender_id = u.id
      WHERE m.id = NEW.id;
    END;

    CREATE TRIGGER IF NOT EXISTS messages_ad AFTER DELETE ON messages BEGIN
      INSERT INTO messages_fts(messages_fts, rowid, content, sender_name)
      VALUES('delete', OLD.rowid, OLD.content, '');
    END;

    CREATE TRIGGER IF NOT EXISTS messages_au AFTER UPDATE ON messages BEGIN
      INSERT INTO messages_fts(messages_fts, rowid, content, sender_name)
      VALUES('delete', OLD.rowid, OLD.content, '');
      INSERT INTO messages_fts(rowid, content, sender_name)
      SELECT m.rowid, m.content, u.name
      FROM messages m
      JOIN users u ON m.sender_id = u.id
      WHERE m.id = NEW.id;
    END;
  `);
}

// Search messages
export function searchMessages(query: string, limit = 20): SearchResult[] {
  return db.getAllSync<SearchResult>(
    `SELECT
       m.id,
       m.chat_id,
       m.content,
       m.created_at,
       u.name as sender_name,
       highlight(messages_fts, 0, '<mark>', '</mark>') as highlighted_content,
       rank
     FROM messages_fts
     JOIN messages m ON messages_fts.rowid = m.rowid
     JOIN users u ON m.sender_id = u.id
     WHERE messages_fts MATCH ?
     ORDER BY rank
     LIMIT ?`,
    [query, limit]
  );
}
```

### 3.8 React Hook for SQLite Queries

```tsx
// hooks/useSQLiteQuery.ts
import { useEffect, useState, useCallback } from 'react';

interface QueryState<T> {
  data: T | null;
  error: Error | null;
  isLoading: boolean;
  refetch: () => void;
}

export function useSQLiteQuery<T>(
  queryFn: () => Promise<T>,
  deps: any[] = []
): QueryState<T> {
  const [data, setData] = useState<T | null>(null);
  const [error, setError] = useState<Error | null>(null);
  const [isLoading, setIsLoading] = useState(true);

  const execute = useCallback(async () => {
    setIsLoading(true);
    setError(null);
    try {
      const result = await queryFn();
      setData(result);
    } catch (err) {
      setError(err instanceof Error ? err : new Error(String(err)));
    } finally {
      setIsLoading(false);
    }
  }, deps);

  useEffect(() => {
    execute();
  }, [execute]);

  return { data, error, isLoading, refetch: execute };
}

// Usage:
function ChatScreen({ chatId }: { chatId: string }) {
  const { data: messages, isLoading, refetch } = useSQLiteQuery(
    () => getChatMessages(chatId),
    [chatId]
  );

  if (isLoading) return <Loading />;

  return (
    <FlatList
      data={messages}
      renderItem={({ item }) => <MessageBubble message={item} />}
      onRefresh={refetch}
    />
  );
}
```

---

## 4. WATERMELONDB: OFFLINE-FIRST AT SCALE

WatermelonDB is built for apps that need to work offline with large datasets and sync to a server. It's built on top of SQLite but adds lazy loading, observable queries, and a sync protocol.

### 4.1 When to Use WatermelonDB

**Use it when:**
- You have 10,000+ records that need to be queried on-device
- You need offline-first with server synchronization
- You have lists that need to be performant with thousands of items (lazy loading)
- You need reactive/observable queries (UI updates when data changes)

**Don't use it when:**
- You have < 1,000 records (MMKV or plain SQLite is simpler)
- You don't need offline-first sync (plain SQLite is fine)
- You're building a simple CRUD app (the overhead isn't worth it)

### 4.2 Setup

```bash
npx expo install @nozbe/watermelondb
npx expo install @nozbe/with-observables
```

### 4.3 Schema Definition

```ts
// db/watermelon/schema.ts
import { appSchema, tableSchema } from '@nozbe/watermelondb';

export const schema = appSchema({
  version: 3, // Increment when you change the schema
  tables: [
    tableSchema({
      name: 'projects',
      columns: [
        { name: 'name', type: 'string' },
        { name: 'description', type: 'string', isOptional: true },
        { name: 'color', type: 'string' },
        { name: 'is_archived', type: 'boolean' },
        { name: 'owner_id', type: 'string', isIndexed: true },
        { name: 'created_at', type: 'number' },
        { name: 'updated_at', type: 'number' },
      ],
    }),
    tableSchema({
      name: 'tasks',
      columns: [
        { name: 'title', type: 'string' },
        { name: 'body', type: 'string', isOptional: true },
        { name: 'is_completed', type: 'boolean' },
        { name: 'priority', type: 'number' }, // 0=low, 1=medium, 2=high
        { name: 'project_id', type: 'string', isIndexed: true },
        { name: 'assignee_id', type: 'string', isIndexed: true },
        { name: 'due_at', type: 'number', isOptional: true },
        { name: 'completed_at', type: 'number', isOptional: true },
        { name: 'created_at', type: 'number' },
        { name: 'updated_at', type: 'number' },
      ],
    }),
    tableSchema({
      name: 'comments',
      columns: [
        { name: 'body', type: 'string' },
        { name: 'task_id', type: 'string', isIndexed: true },
        { name: 'author_id', type: 'string', isIndexed: true },
        { name: 'created_at', type: 'number' },
        { name: 'updated_at', type: 'number' },
      ],
    }),
  ],
});
```

### 4.4 Model Definitions

```ts
// db/watermelon/models/Project.ts
import { Model } from '@nozbe/watermelondb';
import { field, text, date, children, readonly } from '@nozbe/watermelondb/decorators';

export default class Project extends Model {
  static table = 'projects';
  static associations = {
    tasks: { type: 'has_many' as const, foreignKey: 'project_id' },
  };

  @text('name') name!: string;
  @text('description') description!: string;
  @text('color') color!: string;
  @field('is_archived') isArchived!: boolean;
  @text('owner_id') ownerId!: string;
  @readonly @date('created_at') createdAt!: Date;
  @readonly @date('updated_at') updatedAt!: Date;

  @children('tasks') tasks!: any; // Query<Task>
}
```

```ts
// db/watermelon/models/Task.ts
import { Model } from '@nozbe/watermelondb';
import { field, text, date, relation, readonly, writer } from '@nozbe/watermelondb/decorators';

export default class Task extends Model {
  static table = 'tasks';
  static associations = {
    projects: { type: 'belongs_to' as const, key: 'project_id' },
    comments: { type: 'has_many' as const, foreignKey: 'task_id' },
  };

  @text('title') title!: string;
  @text('body') body!: string;
  @field('is_completed') isCompleted!: boolean;
  @field('priority') priority!: number;
  @text('project_id') projectId!: string;
  @text('assignee_id') assigneeId!: string;
  @date('due_at') dueAt!: Date | null;
  @date('completed_at') completedAt!: Date | null;
  @readonly @date('created_at') createdAt!: Date;
  @readonly @date('updated_at') updatedAt!: Date;

  @relation('projects', 'project_id') project!: any; // Relation<Project>
  @children('comments') comments!: any; // Query<Comment>

  // Writer methods for mutations
  @writer async toggleCompleted() {
    await this.update((task) => {
      task.isCompleted = !task.isCompleted;
      task.completedAt = task.isCompleted ? new Date() : null;
    });
  }

  @writer async setPriority(priority: number) {
    await this.update((task) => {
      task.priority = priority;
    });
  }
}
```

### 4.5 Database Initialization

```ts
// db/watermelon/index.ts
import { Database } from '@nozbe/watermelondb';
import SQLiteAdapter from '@nozbe/watermelondb/adapters/sqlite';
import { schema } from './schema';
import { migrations } from './migrations';
import Project from './models/Project';
import Task from './models/Task';
import Comment from './models/Comment';

const adapter = new SQLiteAdapter({
  schema,
  migrations,
  jsi: true, // Use JSI for much better performance
  onSetUpError: (error) => {
    // Database setup failed — handle this gracefully
    console.error('WatermelonDB setup error:', error);
  },
});

export const database = new Database({
  adapter,
  modelClasses: [Project, Task, Comment],
});
```

### 4.6 Observable Queries (The Killer Feature)

WatermelonDB's observables automatically update your UI when the underlying data changes:

```tsx
// components/TaskList.tsx
import { withObservables } from '@nozbe/with-observables';
import { database } from '../db/watermelon';
import { Q } from '@nozbe/watermelondb';

// The "dumb" component — receives data as props
function TaskListComponent({ tasks }: { tasks: Task[] }) {
  return (
    <FlatList
      data={tasks}
      keyExtractor={(item) => item.id}
      renderItem={({ item }) => <TaskRow task={item} />}
    />
  );
}

// The "smart" wrapper — observes the query
const enhance = withObservables(['projectId'], ({ projectId }: { projectId: string }) => ({
  tasks: database
    .get<Task>('tasks')
    .query(
      Q.where('project_id', projectId),
      Q.where('is_completed', false),
      Q.sortBy('priority', Q.desc),
      Q.sortBy('created_at', Q.desc)
    )
    .observe(),
}));

export const TaskList = enhance(TaskListComponent);

// Usage — no manual refetching needed!
// When a task is created, updated, or deleted, the list updates automatically
// <TaskList projectId="proj-123" />
```

### 4.7 WatermelonDB Sync Protocol

WatermelonDB has a built-in sync protocol that handles push/pull synchronization:

```ts
// db/watermelon/sync.ts
import { synchronize } from '@nozbe/watermelondb/sync';
import { database } from './index';

export async function syncDatabase() {
  await synchronize({
    database,
    
    pullChanges: async ({ lastPulledAt, schemaVersion, migration }) => {
      const response = await fetch(
        `https://api.myapp.com/sync/pull?last_pulled_at=${lastPulledAt}&schema_version=${schemaVersion}`,
        {
          headers: { Authorization: `Bearer ${getAuthToken()}` },
        }
      );
      
      if (!response.ok) {
        throw new Error(`Sync pull failed: ${response.status}`);
      }

      const { changes, timestamp } = await response.json();
      
      return { changes, timestamp };
    },

    pushChanges: async ({ changes, lastPulledAt }) => {
      const response = await fetch('https://api.myapp.com/sync/push', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          Authorization: `Bearer ${getAuthToken()}`,
        },
        body: JSON.stringify({ changes, lastPulledAt }),
      });

      if (!response.ok) {
        throw new Error(`Sync push failed: ${response.status}`);
      }
    },

    migrationsEnabledAtVersion: 1,
  });
}
```

The server needs to return changes in this format:

```json
{
  "changes": {
    "projects": {
      "created": [
        { "id": "proj-1", "name": "New Project", "color": "#FF0000" }
      ],
      "updated": [
        { "id": "proj-2", "name": "Updated Name" }
      ],
      "deleted": ["proj-3"]
    },
    "tasks": {
      "created": [],
      "updated": [],
      "deleted": []
    }
  },
  "timestamp": 1700000000000
}
```

---

## 5. REALM: THE OBJECT-ORIENTED ALTERNATIVE

Realm is an object database that feels like working with native objects. It's particularly strong if you're already using MongoDB on the backend.

### 5.1 When to Use Realm

**Use it when:**
- You're using MongoDB Atlas as your backend and want automatic sync
- You prefer object-oriented data modeling over SQL
- You need real-time sync between devices
- Your team is more comfortable with ORM-style access than SQL

**Don't use it when:**
- You don't want a database vendor lock-in
- You need complex SQL queries (JOINs, subqueries, CTEs)
- You want to stay close to the SQL standard

### 5.2 Setup

```bash
npx expo install @realm/react realm
```

### 5.3 Schema Definition (Object Models)

```ts
// db/realm/models.ts
import Realm, { BSON, ObjectSchema } from 'realm';

export class Product extends Realm.Object<Product> {
  _id!: BSON.ObjectId;
  name!: string;
  description?: string;
  price!: number;
  category!: string;
  inStock!: boolean;
  imageUrl?: string;
  createdAt!: Date;

  static schema: ObjectSchema = {
    name: 'Product',
    primaryKey: '_id',
    properties: {
      _id: { type: 'objectId', default: () => new BSON.ObjectId() },
      name: 'string',
      description: 'string?',
      price: 'int',
      category: 'string',
      inStock: { type: 'bool', default: true },
      imageUrl: 'string?',
      createdAt: { type: 'date', default: () => new Date() },
    },
  };
}

export class CartItem extends Realm.Object<CartItem> {
  _id!: BSON.ObjectId;
  product!: Product;
  quantity!: number;

  static schema: ObjectSchema = {
    name: 'CartItem',
    primaryKey: '_id',
    properties: {
      _id: { type: 'objectId', default: () => new BSON.ObjectId() },
      product: 'Product',
      quantity: { type: 'int', default: 1 },
    },
  };
}
```

### 5.4 Realm Provider and Hooks

```tsx
// app/_layout.tsx
import { RealmProvider } from '@realm/react';
import { Product, CartItem } from '../db/realm/models';

export default function RootLayout() {
  return (
    <RealmProvider schema={[Product, CartItem]}>
      <Stack />
    </RealmProvider>
  );
}
```

```tsx
// screens/ProductList.tsx
import { useQuery, useRealm } from '@realm/react';
import { Product } from '../db/realm/models';

function ProductList() {
  const products = useQuery(Product, (collection) =>
    collection.filtered('inStock == true').sorted('name')
  );

  const realm = useRealm();

  const addProduct = () => {
    realm.write(() => {
      realm.create('Product', {
        name: 'New Product',
        price: 999,
        category: 'electronics',
      });
    });
  };

  return (
    <FlatList
      data={products}
      keyExtractor={(item) => item._id.toHexString()}
      renderItem={({ item }) => (
        <Text>{item.name} - ${(item.price / 100).toFixed(2)}</Text>
      )}
    />
  );
}
```

### 5.5 Realm with MongoDB Atlas Sync

```tsx
// app/_layout.tsx — with Atlas Device Sync
import { AppProvider, UserProvider, RealmProvider } from '@realm/react';
import { Product, CartItem } from '../db/realm/models';

export default function RootLayout() {
  return (
    <AppProvider id="your-atlas-app-id">
      <UserProvider fallback={<LoginScreen />}>
        <RealmProvider
          schema={[Product, CartItem]}
          sync={{
            flexible: true,
            onError: (session, error) => {
              console.error('Sync error:', error);
            },
            initialSubscriptions: {
              update: (subs, realm) => {
                subs.add(realm.objects(Product));
                subs.add(realm.objects(CartItem));
              },
            },
          }}
        >
          <Stack />
        </RealmProvider>
      </UserProvider>
    </AppProvider>
  );
}
```

---

## 6. FILE SYSTEM OPERATIONS

For anything that's a file — documents, images, downloads, exports — you need direct file system access.

### 6.1 Setup

```bash
npx expo install expo-file-system
```

### 6.2 Directory Structure

Every app gets two main directories:

```ts
import * as FileSystem from 'expo-file-system';

// Document Directory — persistent, backed up by iCloud/Google
// Use for: user-created content, important files
console.log(FileSystem.documentDirectory);
// iOS: /var/mobile/Containers/Data/Application/<UUID>/Documents/
// Android: /data/data/com.myapp/files/

// Cache Directory — can be cleared by OS when storage is low
// Use for: temporary files, downloaded images, cached data
console.log(FileSystem.cacheDirectory);
// iOS: /var/mobile/Containers/Data/Application/<UUID>/Library/Caches/
// Android: /data/data/com.myapp/cache/
```

**Rule of thumb:** If the user would be upset if the file disappeared, put it in `documentDirectory`. If it can be re-downloaded or regenerated, put it in `cacheDirectory`.

### 6.3 Basic File Operations

```ts
import * as FileSystem from 'expo-file-system';

// === READ a file ===
async function readFile(filename: string): Promise<string> {
  const path = `${FileSystem.documentDirectory}${filename}`;
  const content = await FileSystem.readAsStringAsync(path);
  return content;
}

// === READ a JSON file ===
async function readJSON<T>(filename: string): Promise<T> {
  const content = await readFile(filename);
  return JSON.parse(content);
}

// === WRITE a file ===
async function writeFile(filename: string, content: string): Promise<void> {
  const path = `${FileSystem.documentDirectory}${filename}`;
  await FileSystem.writeAsStringAsync(path, content);
}

// === WRITE a JSON file ===
async function writeJSON(filename: string, data: unknown): Promise<void> {
  await writeFile(filename, JSON.stringify(data, null, 2));
}

// === DELETE a file ===
async function deleteFile(filename: string): Promise<void> {
  const path = `${FileSystem.documentDirectory}${filename}`;
  await FileSystem.deleteAsync(path, { idempotent: true });
}

// === CHECK if file exists ===
async function fileExists(filename: string): Promise<boolean> {
  const path = `${FileSystem.documentDirectory}${filename}`;
  const info = await FileSystem.getInfoAsync(path);
  return info.exists;
}

// === GET file info ===
async function getFileInfo(filename: string) {
  const path = `${FileSystem.documentDirectory}${filename}`;
  const info = await FileSystem.getInfoAsync(path, { size: true });
  return info; // { exists, isDirectory, size, modificationTime, uri }
}

// === CREATE directory ===
async function createDirectory(dirname: string): Promise<void> {
  const path = `${FileSystem.documentDirectory}${dirname}`;
  await FileSystem.makeDirectoryAsync(path, { intermediates: true });
}

// === LIST directory contents ===
async function listFiles(dirname: string = ''): Promise<string[]> {
  const path = `${FileSystem.documentDirectory}${dirname}`;
  return FileSystem.readDirectoryAsync(path);
}

// === COPY a file ===
async function copyFile(from: string, to: string): Promise<void> {
  const fromPath = `${FileSystem.documentDirectory}${from}`;
  const toPath = `${FileSystem.documentDirectory}${to}`;
  await FileSystem.copyAsync({ from: fromPath, to: toPath });
}

// === MOVE a file ===
async function moveFile(from: string, to: string): Promise<void> {
  const fromPath = `${FileSystem.documentDirectory}${from}`;
  const toPath = `${FileSystem.documentDirectory}${to}`;
  await FileSystem.moveAsync({ from: fromPath, to: toPath });
}
```

### 6.4 Downloading Files with Progress

```tsx
// hooks/useFileDownload.ts
import { useState, useCallback, useRef } from 'react';
import * as FileSystem from 'expo-file-system';

interface DownloadState {
  progress: number; // 0-1
  isDownloading: boolean;
  error: string | null;
  localUri: string | null;
}

export function useFileDownload() {
  const [state, setState] = useState<DownloadState>({
    progress: 0,
    isDownloading: false,
    error: null,
    localUri: null,
  });
  const downloadResumable = useRef<FileSystem.DownloadResumable | null>(null);

  const download = useCallback(async (url: string, filename: string) => {
    const localUri = `${FileSystem.documentDirectory}downloads/${filename}`;

    // Ensure downloads directory exists
    await FileSystem.makeDirectoryAsync(
      `${FileSystem.documentDirectory}downloads/`,
      { intermediates: true }
    );

    setState({ progress: 0, isDownloading: true, error: null, localUri: null });

    const callback = (downloadProgress: FileSystem.DownloadProgressData) => {
      const progress =
        downloadProgress.totalBytesWritten /
        downloadProgress.totalBytesExpectedToWrite;
      setState((prev) => ({ ...prev, progress }));
    };

    downloadResumable.current = FileSystem.createDownloadResumable(
      url,
      localUri,
      {},
      callback
    );

    try {
      const result = await downloadResumable.current.downloadAsync();
      if (result) {
        setState({
          progress: 1,
          isDownloading: false,
          error: null,
          localUri: result.uri,
        });
      }
    } catch (error) {
      setState((prev) => ({
        ...prev,
        isDownloading: false,
        error: error instanceof Error ? error.message : 'Download failed',
      }));
    }
  }, []);

  const pause = useCallback(async () => {
    if (downloadResumable.current) {
      await downloadResumable.current.pauseAsync();
      setState((prev) => ({ ...prev, isDownloading: false }));
    }
  }, []);

  const resume = useCallback(async () => {
    if (downloadResumable.current) {
      setState((prev) => ({ ...prev, isDownloading: true }));
      const result = await downloadResumable.current.resumeAsync();
      if (result) {
        setState({
          progress: 1,
          isDownloading: false,
          error: null,
          localUri: result.uri,
        });
      }
    }
  }, []);

  return { ...state, download, pause, resume };
}

// Usage:
function DownloadButton({ url, filename }: { url: string; filename: string }) {
  const { progress, isDownloading, error, localUri, download, pause, resume } =
    useFileDownload();

  return (
    <View>
      {!isDownloading && !localUri && (
        <Button title="Download" onPress={() => download(url, filename)} />
      )}
      {isDownloading && (
        <>
          <ProgressBar progress={progress} />
          <Text>{Math.round(progress * 100)}%</Text>
          <Button title="Pause" onPress={pause} />
        </>
      )}
      {!isDownloading && progress > 0 && progress < 1 && (
        <Button title="Resume" onPress={resume} />
      )}
      {localUri && <Text>Downloaded to {localUri}</Text>}
      {error && <Text style={{ color: 'red' }}>{error}</Text>}
    </View>
  );
}
```

### 6.5 Uploading Files

```ts
// lib/fileUpload.ts
import * as FileSystem from 'expo-file-system';

interface UploadOptions {
  uri: string;
  uploadUrl: string;
  fieldName?: string;
  mimeType?: string;
  headers?: Record<string, string>;
}

export async function uploadFile({
  uri,
  uploadUrl,
  fieldName = 'file',
  mimeType = 'application/octet-stream',
  headers = {},
}: UploadOptions): Promise<FileSystem.FileSystemUploadResult> {
  const result = await FileSystem.uploadAsync(uploadUrl, uri, {
    fieldName,
    httpMethod: 'POST',
    uploadType: FileSystem.FileSystemUploadType.MULTIPART,
    mimeType,
    headers: {
      ...headers,
    },
  });

  return result;
}
```

### 6.6 Sharing Files

```ts
// lib/fileSharing.ts
import * as FileSystem from 'expo-file-system';
import * as Sharing from 'expo-sharing';

export async function shareFile(
  filename: string,
  mimeType?: string
): Promise<void> {
  const uri = `${FileSystem.documentDirectory}${filename}`;
  
  const isAvailable = await Sharing.isAvailableAsync();
  if (!isAvailable) {
    throw new Error('Sharing is not available on this device');
  }

  await Sharing.shareAsync(uri, {
    mimeType: mimeType ?? getMimeType(filename),
    dialogTitle: `Share ${filename}`,
  });
}

function getMimeType(filename: string): string {
  const ext = filename.split('.').pop()?.toLowerCase();
  const mimeTypes: Record<string, string> = {
    pdf: 'application/pdf',
    json: 'application/json',
    csv: 'text/csv',
    txt: 'text/plain',
    png: 'image/png',
    jpg: 'image/jpeg',
    jpeg: 'image/jpeg',
    gif: 'image/gif',
    mp4: 'video/mp4',
    mp3: 'audio/mpeg',
    zip: 'application/zip',
  };
  return mimeTypes[ext ?? ''] ?? 'application/octet-stream';
}
```

---

## 7. DOCUMENT PICKER

For letting users select files from their device.

### 7.1 Setup

```bash
npx expo install expo-document-picker
```

### 7.2 Picking Documents

```tsx
// components/DocumentPicker.tsx
import * as DocumentPicker from 'expo-document-picker';
import * as FileSystem from 'expo-file-system';

interface PickedDocument {
  name: string;
  uri: string;
  size: number;
  mimeType: string;
}

// Pick from device file system
export async function pickDocument(
  types?: string[] // e.g., ['application/pdf', 'text/csv']
): Promise<PickedDocument | null> {
  const result = await DocumentPicker.getDocumentAsync({
    type: types ?? ['*/*'], // All file types
    copyToCacheDirectory: true, // Copy to app's cache for reliable access
    multiple: false,
  });

  if (result.canceled) return null;

  const asset = result.assets[0];
  return {
    name: asset.name,
    uri: asset.uri,
    size: asset.size ?? 0,
    mimeType: asset.mimeType ?? 'application/octet-stream',
  };
}

export async function pickMultipleDocuments(
  types?: string[]
): Promise<PickedDocument[]> {
  const result = await DocumentPicker.getDocumentAsync({
    type: types ?? ['*/*'],
    copyToCacheDirectory: true,
    multiple: true,
  });

  if (result.canceled) return [];

  return result.assets.map((asset) => ({
    name: asset.name,
    uri: asset.uri,
    size: asset.size ?? 0,
    mimeType: asset.mimeType ?? 'application/octet-stream',
  }));
}

// === Save picked document to permanent storage ===
export async function saveDocument(
  pickedDoc: PickedDocument,
  directory = 'documents'
): Promise<string> {
  const destDir = `${FileSystem.documentDirectory}${directory}/`;
  await FileSystem.makeDirectoryAsync(destDir, { intermediates: true });

  const destUri = `${destDir}${pickedDoc.name}`;
  await FileSystem.copyAsync({ from: pickedDoc.uri, to: destUri });

  return destUri;
}
```

```tsx
// Usage in a component
function ImportScreen() {
  const [file, setFile] = useState<PickedDocument | null>(null);

  const handlePick = async () => {
    const doc = await pickDocument(['application/pdf', 'text/csv', 'application/json']);
    if (doc) {
      setFile(doc);
      // Save to permanent storage
      const savedUri = await saveDocument(doc);
      console.log('Saved to:', savedUri);
    }
  };

  return (
    <View>
      <Button title="Import File" onPress={handlePick} />
      {file && (
        <Text>
          Selected: {file.name} ({(file.size / 1024).toFixed(1)} KB)
        </Text>
      )}
    </View>
  );
}
```

---

## 8. IMAGE PICKER

For photos and videos — from the camera roll or camera capture.

### 8.1 Setup

```bash
npx expo install expo-image-picker
```

### 8.2 Picking Images

```tsx
// lib/imagePicker.ts
import * as ImagePicker from 'expo-image-picker';
import { Alert } from 'react-native';

interface PickedImage {
  uri: string;
  width: number;
  height: number;
  type: 'image' | 'video';
  fileSize?: number;
  base64?: string;
}

// Pick from camera roll
export async function pickImage(options?: {
  allowsEditing?: boolean;
  aspect?: [number, number];
  quality?: number; // 0-1
  base64?: boolean;
}): Promise<PickedImage | null> {
  // Request permission
  const { status } = await ImagePicker.requestMediaLibraryPermissionsAsync();
  if (status !== 'granted') {
    Alert.alert(
      'Permission Required',
      'Please grant photo library access to select images.',
      [{ text: 'OK' }]
    );
    return null;
  }

  const result = await ImagePicker.launchImageLibraryAsync({
    mediaTypes: ['images'],
    allowsEditing: options?.allowsEditing ?? true,
    aspect: options?.aspect ?? [1, 1],
    quality: options?.quality ?? 0.8,
    base64: options?.base64 ?? false,
    exif: false,
  });

  if (result.canceled) return null;

  const asset = result.assets[0];
  return {
    uri: asset.uri,
    width: asset.width,
    height: asset.height,
    type: asset.type ?? 'image',
    fileSize: asset.fileSize,
    base64: asset.base64 ?? undefined,
  };
}

// Capture from camera
export async function takePhoto(options?: {
  allowsEditing?: boolean;
  aspect?: [number, number];
  quality?: number;
}): Promise<PickedImage | null> {
  const { status } = await ImagePicker.requestCameraPermissionsAsync();
  if (status !== 'granted') {
    Alert.alert(
      'Permission Required',
      'Please grant camera access to take photos.',
      [{ text: 'OK' }]
    );
    return null;
  }

  const result = await ImagePicker.launchCameraAsync({
    mediaTypes: ['images'],
    allowsEditing: options?.allowsEditing ?? true,
    aspect: options?.aspect,
    quality: options?.quality ?? 0.8,
  });

  if (result.canceled) return null;

  const asset = result.assets[0];
  return {
    uri: asset.uri,
    width: asset.width,
    height: asset.height,
    type: asset.type ?? 'image',
    fileSize: asset.fileSize,
  };
}
```

### 8.3 Image Manipulation (Resizing, Compressing)

```bash
npx expo install expo-image-manipulator
```

```ts
// lib/imageProcessing.ts
import { manipulateAsync, SaveFormat } from 'expo-image-manipulator';

export async function resizeImage(
  uri: string,
  maxWidth: number,
  maxHeight: number,
  quality = 0.8
): Promise<{ uri: string; width: number; height: number }> {
  const result = await manipulateAsync(
    uri,
    [{ resize: { width: maxWidth, height: maxHeight } }],
    { compress: quality, format: SaveFormat.JPEG }
  );

  return result;
}

export async function createThumbnail(
  uri: string,
  size = 150
): Promise<string> {
  const result = await manipulateAsync(
    uri,
    [{ resize: { width: size, height: size } }],
    { compress: 0.6, format: SaveFormat.JPEG }
  );

  return result.uri;
}

export async function prepareForUpload(
  uri: string,
  maxDimension = 1920,
  quality = 0.85
): Promise<{ uri: string; width: number; height: number }> {
  return manipulateAsync(
    uri,
    [{ resize: { width: maxDimension } }], // Height auto-calculated
    { compress: quality, format: SaveFormat.JPEG }
  );
}
```

### 8.4 Avatar Upload Flow (Complete Example)

```tsx
// components/AvatarPicker.tsx
import React, { useState } from 'react';
import { View, Image, TouchableOpacity, ActivityIndicator, Text, StyleSheet } from 'react-native';
import { pickImage } from '../lib/imagePicker';
import { prepareForUpload } from '../lib/imageProcessing';
import { uploadFile } from '../lib/fileUpload';

interface AvatarPickerProps {
  currentAvatarUrl?: string;
  onUploadComplete: (url: string) => void;
}

export function AvatarPicker({ currentAvatarUrl, onUploadComplete }: AvatarPickerProps) {
  const [isUploading, setIsUploading] = useState(false);
  const [previewUri, setPreviewUri] = useState<string | null>(null);

  const handlePick = async () => {
    const image = await pickImage({
      allowsEditing: true,
      aspect: [1, 1],
      quality: 0.9,
    });

    if (!image) return;

    setPreviewUri(image.uri);
    setIsUploading(true);

    try {
      // Resize for upload (max 500x500 for avatars)
      const processed = await prepareForUpload(image.uri, 500, 0.85);

      // Upload
      const result = await uploadFile({
        uri: processed.uri,
        uploadUrl: 'https://api.myapp.com/upload/avatar',
        fieldName: 'avatar',
        mimeType: 'image/jpeg',
      });

      const { url } = JSON.parse(result.body);
      onUploadComplete(url);
    } catch (error) {
      console.error('Avatar upload failed:', error);
      setPreviewUri(null);
    } finally {
      setIsUploading(false);
    }
  };

  const displayUri = previewUri ?? currentAvatarUrl;

  return (
    <TouchableOpacity onPress={handlePick} disabled={isUploading} style={styles.container}>
      {displayUri ? (
        <Image source={{ uri: displayUri }} style={styles.avatar} />
      ) : (
        <View style={[styles.avatar, styles.placeholder]}>
          <Text style={styles.placeholderText}>+</Text>
        </View>
      )}
      {isUploading && (
        <View style={styles.overlay}>
          <ActivityIndicator color="white" />
        </View>
      )}
    </TouchableOpacity>
  );
}

const styles = StyleSheet.create({
  container: { position: 'relative' },
  avatar: { width: 100, height: 100, borderRadius: 50 },
  placeholder: { backgroundColor: '#E0E0E0', justifyContent: 'center', alignItems: 'center' },
  placeholderText: { fontSize: 32, color: '#999' },
  overlay: {
    ...StyleSheet.absoluteFillObject,
    borderRadius: 50,
    backgroundColor: 'rgba(0,0,0,0.5)',
    justifyContent: 'center',
    alignItems: 'center',
  },
});
```

---

## 9. LARGE FILE HANDLING

Mobile apps face unique challenges with large files: limited memory, unreliable networks, and the need for background processing.

### 9.1 Streaming Reads

For large files, don't read the entire file into memory:

```ts
// Reading large files in chunks
import * as FileSystem from 'expo-file-system';

async function processLargeFile(uri: string): Promise<void> {
  const info = await FileSystem.getInfoAsync(uri, { size: true });
  if (!info.exists || !('size' in info)) return;

  const chunkSize = 64 * 1024; // 64KB chunks
  const totalChunks = Math.ceil(info.size / chunkSize);

  for (let i = 0; i < totalChunks; i++) {
    const position = i * chunkSize;
    const length = Math.min(chunkSize, info.size - position);
    
    // Read chunk as base64
    const chunk = await FileSystem.readAsStringAsync(uri, {
      encoding: FileSystem.EncodingType.Base64,
      position,
      length,
    });

    // Process chunk...
    await processChunk(chunk, i, totalChunks);
  }
}
```

### 9.2 Resumable Downloads

```ts
// lib/resumableDownload.ts
import * as FileSystem from 'expo-file-system';
import { MMKV } from 'react-native-mmkv';

const storage = new MMKV({ id: 'downloads' });

interface DownloadState {
  url: string;
  localUri: string;
  resumeData?: string;
  totalBytes: number;
  downloadedBytes: number;
  status: 'pending' | 'downloading' | 'paused' | 'completed' | 'failed';
}

export class ResumableDownloadManager {
  private downloads = new Map<string, FileSystem.DownloadResumable>();

  async startDownload(
    id: string,
    url: string,
    filename: string,
    onProgress?: (progress: number) => void
  ): Promise<string> {
    const localUri = `${FileSystem.documentDirectory}downloads/${filename}`;

    // Check for existing resume data
    const savedState = this.getState(id);
    
    const callback = (progress: FileSystem.DownloadProgressData) => {
      const percent = progress.totalBytesWritten / progress.totalBytesExpectedToWrite;
      this.saveState(id, {
        url,
        localUri,
        totalBytes: progress.totalBytesExpectedToWrite,
        downloadedBytes: progress.totalBytesWritten,
        status: 'downloading',
      });
      onProgress?.(percent);
    };

    let downloadResumable: FileSystem.DownloadResumable;

    if (savedState?.resumeData) {
      // Resume from where we left off
      downloadResumable = new FileSystem.DownloadResumable(
        url,
        localUri,
        {},
        callback,
        savedState.resumeData
      );
    } else {
      downloadResumable = FileSystem.createDownloadResumable(
        url,
        localUri,
        {},
        callback
      );
    }

    this.downloads.set(id, downloadResumable);

    try {
      const result = savedState?.resumeData
        ? await downloadResumable.resumeAsync()
        : await downloadResumable.downloadAsync();

      if (result) {
        this.saveState(id, {
          url,
          localUri: result.uri,
          totalBytes: 0,
          downloadedBytes: 0,
          status: 'completed',
        });
        return result.uri;
      }
      throw new Error('Download returned no result');
    } catch (error) {
      this.saveState(id, {
        url,
        localUri,
        totalBytes: savedState?.totalBytes ?? 0,
        downloadedBytes: savedState?.downloadedBytes ?? 0,
        status: 'failed',
      });
      throw error;
    }
  }

  async pauseDownload(id: string): Promise<void> {
    const download = this.downloads.get(id);
    if (!download) return;

    const saveResult = await download.pauseAsync();
    const state = this.getState(id);
    if (state) {
      this.saveState(id, {
        ...state,
        resumeData: JSON.stringify(saveResult),
        status: 'paused',
      });
    }
  }

  private saveState(id: string, state: DownloadState): void {
    storage.set(`download-${id}`, JSON.stringify(state));
  }

  private getState(id: string): DownloadState | null {
    const data = storage.getString(`download-${id}`);
    return data ? JSON.parse(data) : null;
  }
}
```

### 9.3 Cache Management

```ts
// lib/cacheManager.ts
import * as FileSystem from 'expo-file-system';

export class CacheManager {
  private cacheDir: string;
  private maxCacheSize: number; // bytes

  constructor(subdirectory: string, maxSizeMB = 100) {
    this.cacheDir = `${FileSystem.cacheDirectory}${subdirectory}/`;
    this.maxCacheSize = maxSizeMB * 1024 * 1024;
  }

  async initialize(): Promise<void> {
    await FileSystem.makeDirectoryAsync(this.cacheDir, { intermediates: true });
  }

  async getCacheSize(): Promise<number> {
    const files = await FileSystem.readDirectoryAsync(this.cacheDir);
    let totalSize = 0;

    for (const file of files) {
      const info = await FileSystem.getInfoAsync(`${this.cacheDir}${file}`, {
        size: true,
      });
      if (info.exists && 'size' in info) {
        totalSize += info.size;
      }
    }

    return totalSize;
  }

  async evictOldFiles(): Promise<void> {
    const currentSize = await this.getCacheSize();
    if (currentSize <= this.maxCacheSize) return;

    const files = await FileSystem.readDirectoryAsync(this.cacheDir);
    const fileInfos = await Promise.all(
      files.map(async (file) => {
        const path = `${this.cacheDir}${file}`;
        const info = await FileSystem.getInfoAsync(path, { size: true });
        return {
          path,
          size: info.exists && 'size' in info ? info.size : 0,
          modTime: info.exists && 'modificationTime' in info
            ? info.modificationTime
            : 0,
        };
      })
    );

    // Sort by modification time (oldest first)
    fileInfos.sort((a, b) => a.modTime - b.modTime);

    let freed = 0;
    const target = currentSize - this.maxCacheSize * 0.8; // Free to 80%

    for (const file of fileInfos) {
      if (freed >= target) break;
      await FileSystem.deleteAsync(file.path, { idempotent: true });
      freed += file.size;
    }

    console.log(`Cache eviction: freed ${(freed / 1024 / 1024).toFixed(1)}MB`);
  }

  async clearCache(): Promise<void> {
    await FileSystem.deleteAsync(this.cacheDir, { idempotent: true });
    await this.initialize();
  }
}

// Usage:
const imageCache = new CacheManager('images', 200); // 200MB max
const downloadCache = new CacheManager('downloads', 500); // 500MB max

// Periodically check cache size
async function maintainCaches() {
  await imageCache.evictOldFiles();
  await downloadCache.evictOldFiles();
}
```

---

## 10. MIGRATION PATTERNS

Data migrations on mobile are unforgiving. On the web, you can run migrations on your server and every client sees the new schema immediately. On mobile, every user has their own local database, and they might skip multiple app versions between updates. Your migration strategy must handle jumping from v1 to v5 gracefully.

### 10.1 MMKV Migrations

MMKV is schemaless, so "migrations" really mean data format changes:

```ts
// lib/migrations/mmkvMigrations.ts
import { MMKV } from 'react-native-mmkv';

const storage = new MMKV();

const MIGRATION_VERSION_KEY = 'migration.version';
const CURRENT_VERSION = 3;

export function runMMKVMigrations(): void {
  const currentVersion = storage.getNumber(MIGRATION_VERSION_KEY) ?? 0;

  if (currentVersion >= CURRENT_VERSION) return;

  console.log(`Running MMKV migrations from v${currentVersion} to v${CURRENT_VERSION}`);

  // Run migrations in order
  if (currentVersion < 1) migrationV1();
  if (currentVersion < 2) migrationV2();
  if (currentVersion < 3) migrationV3();

  storage.set(MIGRATION_VERSION_KEY, CURRENT_VERSION);
}

// v1: Rename keys from old format
function migrationV1(): void {
  // Old: 'darkMode' (boolean) -> New: 'prefs.theme' ('light' | 'dark' | 'system')
  const darkMode = storage.getBoolean('darkMode');
  if (darkMode !== undefined) {
    storage.set('prefs.theme', darkMode ? 'dark' : 'light');
    storage.delete('darkMode');
  }

  // Old: 'language' -> New: 'prefs.locale'
  const language = storage.getString('language');
  if (language) {
    storage.set('prefs.locale', language);
    storage.delete('language');
  }
}

// v2: Restructure user data
function migrationV2(): void {
  const userJson = storage.getString('user');
  if (userJson) {
    try {
      const user = JSON.parse(userJson);
      // Split monolithic user object into separate keys
      if (user.token) storage.set('auth.token', user.token);
      if (user.id) storage.set('auth.userId', user.id);
      if (user.name) storage.set('cache.userProfile', JSON.stringify({
        name: user.name,
        email: user.email,
        avatar: user.avatar,
      }));
      storage.delete('user');
    } catch {
      // Corrupt data — just delete it
      storage.delete('user');
    }
  }
}

// v3: Add defaults for new features
function migrationV3(): void {
  if (!storage.contains('prefs.hapticFeedback')) {
    storage.set('prefs.hapticFeedback', JSON.stringify(true));
  }
  if (!storage.contains('prefs.notificationsEnabled')) {
    storage.set('prefs.notificationsEnabled', JSON.stringify(true));
  }
}
```

### 10.2 SQLite Manual Migrations

If you're not using Drizzle's migration system, here's a manual approach:

```ts
// db/migrations.ts
import * as SQLite from 'expo-sqlite';

const db = SQLite.openDatabaseSync('myapp.db');

const USER_VERSION_PRAGMA = 'PRAGMA user_version';

interface Migration {
  version: number;
  name: string;
  up: () => void;
}

const migrations: Migration[] = [
  {
    version: 1,
    name: 'initial_schema',
    up: () => {
      db.execSync(`
        CREATE TABLE IF NOT EXISTS users (
          id TEXT PRIMARY KEY,
          name TEXT NOT NULL,
          email TEXT UNIQUE NOT NULL,
          created_at TEXT DEFAULT (datetime('now'))
        );
        CREATE TABLE IF NOT EXISTS messages (
          id TEXT PRIMARY KEY,
          sender_id TEXT NOT NULL,
          content TEXT NOT NULL,
          created_at TEXT DEFAULT (datetime('now'))
        );
      `);
    },
  },
  {
    version: 2,
    name: 'add_avatar_and_chat_id',
    up: () => {
      db.execSync(`
        ALTER TABLE users ADD COLUMN avatar_url TEXT;
        ALTER TABLE messages ADD COLUMN chat_id TEXT;
        CREATE INDEX idx_messages_chat_id ON messages(chat_id);
      `);
    },
  },
  {
    version: 3,
    name: 'add_read_receipts',
    up: () => {
      db.execSync(`
        ALTER TABLE messages ADD COLUMN read_at TEXT;
        ALTER TABLE messages ADD COLUMN type TEXT DEFAULT 'text';
        CREATE INDEX idx_messages_read_at ON messages(read_at);
      `);
    },
  },
  {
    version: 4,
    name: 'add_products_table',
    up: () => {
      db.execSync(`
        CREATE TABLE products (
          id TEXT PRIMARY KEY,
          name TEXT NOT NULL,
          price INTEGER NOT NULL,
          category TEXT NOT NULL,
          in_stock INTEGER DEFAULT 1,
          created_at TEXT DEFAULT (datetime('now'))
        );
        CREATE INDEX idx_products_category ON products(category);
      `);
    },
  },
];

export function runMigrations(): void {
  // Get current database version
  const result = db.getFirstSync<{ user_version: number }>(USER_VERSION_PRAGMA);
  const currentVersion = result?.user_version ?? 0;

  const pendingMigrations = migrations.filter((m) => m.version > currentVersion);

  if (pendingMigrations.length === 0) {
    console.log(`Database is up to date (v${currentVersion})`);
    return;
  }

  console.log(
    `Running ${pendingMigrations.length} migration(s): v${currentVersion} -> v${migrations[migrations.length - 1].version}`
  );

  db.withTransactionSync(() => {
    for (const migration of pendingMigrations) {
      console.log(`  Running migration v${migration.version}: ${migration.name}`);
      migration.up();
    }

    // Update the version
    const newVersion = pendingMigrations[pendingMigrations.length - 1].version;
    db.execSync(`PRAGMA user_version = ${newVersion}`);
  });

  console.log('All migrations complete');
}
```

### 10.3 WatermelonDB Migrations

```ts
// db/watermelon/migrations.ts
import { schemaMigrations, addColumns, createTable } from '@nozbe/watermelondb/Schema/migrations';

export const migrations = schemaMigrations({
  migrations: [
    {
      toVersion: 2,
      steps: [
        addColumns({
          table: 'tasks',
          columns: [
            { name: 'due_at', type: 'number', isOptional: true },
            { name: 'completed_at', type: 'number', isOptional: true },
          ],
        }),
      ],
    },
    {
      toVersion: 3,
      steps: [
        createTable({
          name: 'comments',
          columns: [
            { name: 'body', type: 'string' },
            { name: 'task_id', type: 'string', isIndexed: true },
            { name: 'author_id', type: 'string', isIndexed: true },
            { name: 'created_at', type: 'number' },
            { name: 'updated_at', type: 'number' },
          ],
        }),
        addColumns({
          table: 'tasks',
          columns: [
            { name: 'assignee_id', type: 'string', isIndexed: true },
          ],
        }),
      ],
    },
  ],
});
```

### 10.4 Migration Safety Tips

**1. Never delete columns in SQLite.** SQLite doesn't support `DROP COLUMN` on older versions (pre-3.35.0). Instead, create a new table, copy data, drop old table, rename. Or just leave the old column — unused columns cost almost nothing.

**2. Always wrap migrations in transactions.** If a migration fails halfway through, you want the entire thing to roll back, not leave your database in a corrupt state.

**3. Test migration paths.** Don't just test v(N-1) to v(N). Test v1 to v(N), v2 to v(N), etc. Users who haven't updated in months will hit multiple migrations at once.

**4. Have a nuclear option.** If migrations fail catastrophically, you need a fallback:

```ts
function runMigrationsWithFallback(): void {
  try {
    runMigrations();
  } catch (error) {
    console.error('Migration failed, resetting database:', error);
    
    // Delete the database file
    FileSystem.deleteAsync(
      `${FileSystem.documentDirectory}SQLite/myapp.db`,
      { idempotent: true }
    );
    
    // Re-create with current schema
    initializeDatabase();
    
    // Flag that data was lost — show user a message
    storage.set('db.wasReset', true);
  }
}
```

**5. Version your migration runner.** Keep a log of which migrations have run:

```ts
// After each migration
db.runSync(
  'INSERT INTO _migrations (version, name, applied_at) VALUES (?, ?, datetime("now"))',
  [migration.version, migration.name]
);
```

---

## 11. ENCRYPTED STORAGE

Different data needs different levels of protection. Here's the full encryption landscape for mobile storage.

### 11.1 expo-secure-store (For Secrets)

```bash
npx expo install expo-secure-store
```

This is the right choice for tokens, API keys, and small secrets. It uses:
- **iOS:** Keychain Services (hardware-backed on devices with Secure Enclave)
- **Android:** EncryptedSharedPreferences (backed by Android Keystore)

```ts
// lib/secureStorage.ts
import * as SecureStore from 'expo-secure-store';

// === Store a secret ===
export async function storeSecret(key: string, value: string): Promise<void> {
  await SecureStore.setItemAsync(key, value, {
    keychainAccessible: SecureStore.WHEN_UNLOCKED_THIS_DEVICE_ONLY,
    // Options:
    // WHEN_UNLOCKED — accessible when device is unlocked
    // AFTER_FIRST_UNLOCK — accessible after first unlock (persists across locks)
    // WHEN_UNLOCKED_THIS_DEVICE_ONLY — same as WHEN_UNLOCKED, not backed up
    // AFTER_FIRST_UNLOCK_THIS_DEVICE_ONLY — same, not backed up
  });
}

// === Retrieve a secret ===
export async function getSecret(key: string): Promise<string | null> {
  return SecureStore.getItemAsync(key);
}

// === Delete a secret ===
export async function deleteSecret(key: string): Promise<void> {
  await SecureStore.deleteItemAsync(key);
}

// === Auth token management ===
export const authTokens = {
  async save(accessToken: string, refreshToken: string): Promise<void> {
    await Promise.all([
      storeSecret('auth.accessToken', accessToken),
      storeSecret('auth.refreshToken', refreshToken),
    ]);
  },

  async getAccessToken(): Promise<string | null> {
    return getSecret('auth.accessToken');
  },

  async getRefreshToken(): Promise<string | null> {
    return getSecret('auth.refreshToken');
  },

  async clear(): Promise<void> {
    await Promise.all([
      deleteSecret('auth.accessToken'),
      deleteSecret('auth.refreshToken'),
    ]);
  },
};
```

**Limitations of expo-secure-store:**
- Value size limit: ~2KB (varies by platform)
- Async only — no synchronous access
- Not suitable for large amounts of data
- Keys are limited to ~100 characters

### 11.2 MMKV Encryption (For Larger Encrypted Data)

For data that's larger than 2KB but still needs encryption:

```ts
// lib/encryptedMMKV.ts
import { MMKV } from 'react-native-mmkv';
import * as SecureStore from 'expo-secure-store';

let _encryptedStorage: MMKV | null = null;

export async function getEncryptedMMKV(): Promise<MMKV> {
  if (_encryptedStorage) return _encryptedStorage;

  // Get or create encryption key
  let encKey = await SecureStore.getItemAsync('mmkv.encKey');
  if (!encKey) {
    // Generate a random 16-char key
    const array = new Uint8Array(16);
    crypto.getRandomValues(array);
    encKey = Array.from(array, (b) => b.toString(16).padStart(2, '0')).join('');
    await SecureStore.setItemAsync('mmkv.encKey', encKey);
  }

  _encryptedStorage = new MMKV({
    id: 'encrypted',
    encryptionKey: encKey,
  });

  return _encryptedStorage;
}
```

### 11.3 SQLCipher (For Encrypted SQLite)

If you need full database encryption, SQLCipher is the standard. It encrypts the entire SQLite database file.

**Note:** As of 2026, `expo-sqlite` doesn't natively support SQLCipher. You'll need a community package or config plugin:

```bash
# Community SQLCipher wrapper
npx expo install react-native-quick-sqlite
# or
npx expo install @op-engineering/op-sqlite # Has built-in encryption
```

```ts
// With op-sqlite (encryption built-in)
import { open } from '@op-engineering/op-sqlite';

const db = open({
  name: 'encrypted.db',
  encryptionKey: 'your-encryption-key', // Derive from SecureStore
});

// Use like any SQLite database
db.execute('CREATE TABLE IF NOT EXISTS secrets (key TEXT, value TEXT)');
```

### 11.4 When to Encrypt What

```
+---------------------------------+--------------------+----------------------+
| Data Type                       | Encrypt?           | Method               |
+---------------------------------+--------------------+----------------------+
| Auth tokens, API keys           | Always             | expo-secure-store    |
| User PII (SSN, medical)         | Always             | MMKV encrypted or   |
|                                 |                    | SQLCipher            |
| Financial data                  | Always             | SQLCipher            |
| Chat messages                   | Depends on app     | SQLCipher if needed  |
| User preferences                | No                 | MMKV (plain)         |
| Cached API responses            | Usually no         | MMKV/SQLite (plain)  |
| Downloaded files                | Depends on content | File-level encryption|
| Analytics events                | No                 | SQLite (plain)       |
| Feature flags                   | No                 | MMKV (plain)         |
| Onboarding state                | No                 | MMKV (plain)         |
+---------------------------------+--------------------+----------------------+
```

**Rule of thumb:** Encrypt data that would cause harm if the device is compromised. Don't encrypt everything — it adds complexity and performance overhead. Most user preferences and cached data don't need encryption.

---

## 12. PUTTING IT ALL TOGETHER

Here's how a production app typically organizes its storage:

```
lib/
  storage/
    mmkv.ts              -> MMKV instance + typed wrapper
    secureStore.ts       -> expo-secure-store wrapper for secrets
    cacheManager.ts      -> File cache management
  
db/
  client.ts              -> Drizzle + expo-sqlite setup
  schema.ts              -> Drizzle table definitions
  queries.ts             -> Type-safe query functions
  migrations/            -> Drizzle migration files
  migrate.ts             -> Migration runner
  
hooks/
  useStorageValue.ts     -> React hook for MMKV values
  useSQLiteQuery.ts      -> React hook for SQLite queries
  useFileDownload.ts     -> Download with progress hook

migrations/
  mmkvMigrations.ts      -> MMKV data migrations
```

### Initialization Order

```tsx
// app/_layout.tsx
export default function RootLayout() {
  const [ready, setReady] = useState(false);

  useEffect(() => {
    async function initialize() {
      // 1. Run MMKV migrations (synchronous, fast)
      runMMKVMigrations();

      // 2. Run SQLite migrations (async, but usually fast)
      await runDatabaseMigrations();

      // 3. Initialize caches
      await imageCache.initialize();
      await downloadCache.initialize();

      // 4. Migrate from AsyncStorage if needed (one-time)
      await migrateFromAsyncStorage();

      setReady(true);
    }

    initialize().catch(console.error);
  }, []);

  if (!ready) return <SplashScreen />;

  return <Stack />;
}
```

---

## SUMMARY

Storage in mobile apps isn't one-size-fits-all. The right choice depends entirely on what you're storing:

- **MMKV** is your workhorse for key-value data. It's 30x faster than AsyncStorage, synchronous, and handles everything from user preferences to cached JSON responses. If you're building a new app, skip AsyncStorage entirely and go straight to MMKV.

- **SQLite** (via `expo-sqlite` + Drizzle ORM) is your structured data engine. When you need queries, relationships, indexes, and migrations — that's SQLite. Drizzle gives you type safety without the runtime overhead of a heavy ORM.

- **WatermelonDB** is for when you outgrow basic SQLite. If you need observable queries, lazy loading for huge lists, and a built-in sync protocol, WatermelonDB is the proven choice. But don't reach for it prematurely — it adds significant complexity.

- **expo-secure-store** is for secrets. Period. Auth tokens, API keys, encryption keys. Not for general data.

- **expo-file-system** is for files. Downloads, uploads, document management, cache control. Combined with expo-document-picker and expo-image-picker, it handles the entire file lifecycle.

The most important thing you can do is plan your migration strategy from day one. You WILL change your storage schema. You WILL need to rename keys, add columns, restructure data. The teams that plan for this from the start ship updates confidently. The teams that don't end up with crash loops and data loss.

---

**Next:** [Chapter 13: Performance Optimization](/part-2-react-native-expo/13-performance) — how to measure, diagnose, and fix performance problems in React Native apps.
