<!--
  CHAPTER: 38
  TITLE: Search Implementation — From Input to Results
  PART: IV — Architecture at Scale
  PREREQS: Chapters 11, 34
  KEY_TOPICS: Algolia, Typesense, Meilisearch, full-text search, search UX, debounce, autocomplete, faceted search, recent searches, search suggestions, SQLite FTS, Postgres full-text search
  DIFFICULTY: Intermediate
  UPDATED: 2026-04-07
-->

# Chapter 38: Search Implementation — From Input to Results

> "Search is the most honest UI in your app. Users type exactly what they want, and you either give it to them or you don't." — Every search engineer
>
> "If your search doesn't return what I'm looking for in the first three results, I'm gone." — Every user, whether they know it or not

---

<details>
<summary><strong>TL;DR</strong></summary>

- For small datasets (< 10K records), client-side search with a library like Fuse.js is fine; for medium datasets, Postgres full-text search (tsvector + GIN indexes) is surprisingly good; for large datasets or complex search needs (typo tolerance, faceting, geo), use a dedicated engine like Algolia or Typesense
- Algolia is the enterprise standard: instant, globally distributed, incredible DX, but expensive at scale; Typesense is the best open-source alternative with excellent typo tolerance and self-hosting; Meilisearch is simpler with great defaults for smaller teams
- Search UX is at least as important as search infrastructure: debounce input at 250-300ms, show recent searches, highlight matches, handle empty and no-results states gracefully, and store recent searches locally with MMKV
- Keep your search index in sync with your database using webhooks or change-data-capture patterns; stale search results are worse than slow search results
- Track what users search for, especially zero-result queries — they're telling you exactly what's missing from your product

</details>

---

## 1. THE SEARCH STACK

### Choosing the Right Tool for the Job

Search is one of those features where the right architectural choice saves you months of work, and the wrong one haunts you for years. I've seen teams build full Elasticsearch clusters for an app with 500 products. I've also seen teams try to make Postgres `LIKE '%query%'` work for a marketplace with 2 million listings. Both are painful.

Here's the decision framework:

```
┌──────────────────────────────────────────────────────────────────────┐
│                   THE SEARCH DECISION TREE                           │
│                                                                      │
│  How many searchable records do you have?                            │
│                                                                      │
│  < 1,000 records                                                     │
│  └─→ Client-side search (Fuse.js, loaded in memory)                │
│      Pros: Zero infrastructure, instant, works offline               │
│      Cons: Doesn't scale, no server-side analytics                  │
│                                                                      │
│  1,000 - 100,000 records                                            │
│  └─→ Do you need typo tolerance, facets, or relevance tuning?       │
│      ├── No  → Postgres full-text search (tsvector + GIN)           │
│      │         Pros: No new infra, transactionally consistent       │
│      │         Cons: Limited relevance tuning, no typo tolerance    │
│      └── Yes → Dedicated search engine                               │
│                                                                      │
│  100,000+ records                                                    │
│  └─→ Dedicated search engine (always)                               │
│      ├── Budget for managed service? → Algolia                      │
│      ├── Want to self-host?          → Typesense or Meilisearch     │
│      └── Already have Elastic?       → Keep using it (but consider  │
│                                         switching for simplicity)    │
│                                                                      │
│  Need on-device search (offline)?                                    │
│  └─→ SQLite FTS5 (via expo-sqlite or op-sqlite)                    │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

### Client-Side Search

For small datasets, client-side search is the simplest and fastest option. You load the data into memory and search it with a fuzzy-matching library. No server round trips. Works offline. Zero infrastructure.

```typescript
// Client-side search with Fuse.js
// Best for: product catalogs < 1000 items, settings pages, help articles

import Fuse from 'fuse.js';

interface Product {
  id: string;
  name: string;
  description: string;
  category: string;
  tags: string[];
}

// Configure Fuse with your searchable fields
const fuse = new Fuse<Product>(products, {
  // Which fields to search
  keys: [
    { name: 'name', weight: 2 },         // Name matches are twice as important
    { name: 'description', weight: 1 },
    { name: 'category', weight: 1.5 },
    { name: 'tags', weight: 1 },
  ],
  
  // Fuzzy matching config
  threshold: 0.3,          // 0 = exact match, 1 = match anything
  distance: 100,           // How far to search from the expected position
  minMatchCharLength: 2,   // Don't match single characters
  
  // Performance
  shouldSort: true,        // Sort by relevance score
  includeScore: true,      // Include relevance score in results
  includeMatches: true,    // Include match positions (for highlighting)
});

// Search
const results = fuse.search('runnig shoes'); // Note the typo — Fuse handles it
// Returns: [{ item: { name: 'Running Shoes Pro', ... }, score: 0.12, matches: [...] }]
```

**When client-side search breaks down:**
- More than a few thousand records (memory usage, initial load time)
- You need server-side search analytics
- You need faceted search (filter by category AND price range AND rating)
- You need search across multiple content types with different relevance rules

### Database Search (Postgres Full-Text Search)

Postgres has a surprisingly capable full-text search engine built in. For many apps, it's all you need, and you don't need to add any new infrastructure.

```sql
-- Setting up Postgres full-text search

-- 1. Add a tsvector column to your table
ALTER TABLE products ADD COLUMN search_vector tsvector;

-- 2. Create a GIN index for fast lookups
CREATE INDEX idx_products_search ON products USING GIN(search_vector);

-- 3. Populate the search vector
UPDATE products SET search_vector = 
  setweight(to_tsvector('english', coalesce(name, '')), 'A') ||        -- Name: highest weight
  setweight(to_tsvector('english', coalesce(category, '')), 'B') ||    -- Category: medium weight
  setweight(to_tsvector('english', coalesce(description, '')), 'C');   -- Description: lower weight

-- 4. Create a trigger to keep it in sync
CREATE OR REPLACE FUNCTION update_search_vector() RETURNS trigger AS $$
BEGIN
  NEW.search_vector :=
    setweight(to_tsvector('english', coalesce(NEW.name, '')), 'A') ||
    setweight(to_tsvector('english', coalesce(NEW.category, '')), 'B') ||
    setweight(to_tsvector('english', coalesce(NEW.description, '')), 'C');
  RETURN NEW;
END
$$ LANGUAGE plpgsql;

CREATE TRIGGER products_search_update
  BEFORE INSERT OR UPDATE ON products
  FOR EACH ROW EXECUTE FUNCTION update_search_vector();

-- 5. Query with ranking
SELECT 
  id, 
  name, 
  description,
  ts_rank(search_vector, query) AS rank,
  ts_headline('english', description, query, 'StartSel=<mark>, StopSel=</mark>') AS highlighted_description
FROM products, plainto_tsquery('english', 'running shoes') AS query
WHERE search_vector @@ query
ORDER BY rank DESC
LIMIT 20;
```

```typescript
// Using Postgres FTS with Drizzle ORM

import { sql } from 'drizzle-orm';
import { pgTable, text, index } from 'drizzle-orm/pg-core';
import { db } from './database';

// Schema definition
export const products = pgTable('products', {
  id: text('id').primaryKey(),
  name: text('name').notNull(),
  description: text('description'),
  category: text('category'),
  searchVector: text('search_vector'), // tsvector column
}, (table) => ({
  searchIdx: index('idx_products_search').using('gin', table.searchVector),
}));

// Search function
export async function searchProducts(query: string, limit = 20) {
  if (!query.trim()) return [];

  const results = await db.execute(sql`
    SELECT 
      id,
      name,
      description,
      category,
      ts_rank(search_vector, plainto_tsquery('english', ${query})) as rank,
      ts_headline(
        'english', 
        description, 
        plainto_tsquery('english', ${query}),
        'StartSel=<mark>, StopSel=</mark>, MaxWords=35, MinWords=15'
      ) as highlighted_description
    FROM products
    WHERE search_vector @@ plainto_tsquery('english', ${query})
    ORDER BY rank DESC
    LIMIT ${limit}
  `);

  return results.rows;
}
```

**Postgres FTS strengths:**
- Zero additional infrastructure
- Transactionally consistent (search is always in sync with data)
- Supports weighted search, phrase matching, and highlighting
- Handles stemming (searching "running" matches "run", "runs", "runner")
- GIN indexes make it fast even for large tables

**Postgres FTS weaknesses:**
- No built-in typo tolerance (searching "runnig" won't match "running")
- Limited relevance tuning compared to dedicated engines
- No faceted search without additional queries
- Performance degrades with complex queries on very large datasets (10M+ rows)

### SQLite FTS5 for On-Device Search

For offline-first mobile apps, SQLite FTS5 lets you search data that's stored locally on the device. Combined with Expo SQLite or OP-SQLite, this gives you a powerful on-device search engine.

```typescript
// On-device search with SQLite FTS5 in React Native

import * as SQLite from 'expo-sqlite';

const db = await SQLite.openDatabaseAsync('app.db');

// Create an FTS5 virtual table
await db.execAsync(`
  CREATE VIRTUAL TABLE IF NOT EXISTS products_fts USING fts5(
    name,
    description,
    category,
    content='products',
    content_rowid='rowid',
    tokenize='porter unicode61'  -- Stemming + Unicode support
  );
`);

// Populate the FTS index from your products table
await db.execAsync(`
  INSERT INTO products_fts(rowid, name, description, category)
  SELECT rowid, name, description, category FROM products;
`);

// Search with ranking and highlighting
async function searchProductsOffline(query: string): Promise<SearchResult[]> {
  if (!query.trim()) return [];

  const results = await db.getAllAsync<{
    name: string;
    description: string;
    category: string;
    rank: number;
    highlighted_name: string;
  }>(`
    SELECT 
      name,
      description,
      category,
      rank,
      highlight(products_fts, 0, '<mark>', '</mark>') as highlighted_name
    FROM products_fts
    WHERE products_fts MATCH ?
    ORDER BY rank
    LIMIT 20
  `, [query]);

  return results;
}

// Keep FTS index in sync when products change
async function onProductInserted(product: Product) {
  await db.runAsync(
    'INSERT INTO products_fts(rowid, name, description, category) VALUES (?, ?, ?, ?)',
    [product.rowid, product.name, product.description, product.category]
  );
}

async function onProductUpdated(product: Product) {
  await db.runAsync(`
    UPDATE products_fts SET name = ?, description = ?, category = ?
    WHERE rowid = ?
  `, [product.name, product.description, product.category, product.rowid]);
}

async function onProductDeleted(rowid: number) {
  await db.runAsync('DELETE FROM products_fts WHERE rowid = ?', [rowid]);
}
```

---

## 2. SEARCH UX PATTERNS

### The UX Matters More Than the Engine

Here's an uncomfortable truth: a mediocre search engine with excellent UX will feel better to users than a world-class search engine with bad UX. I've seen apps with Algolia under the hood that felt terrible because the search screen was poorly designed. And I've seen apps with basic Postgres search that felt snappy and delightful because the UX was thoughtful.

The search experience is defined by six things:

1. **Input behavior** — How does the search box respond to typing?
2. **Speed** — How fast do results appear?
3. **Empty state** — What does the user see before they've typed anything?
4. **Results presentation** — How are results displayed?
5. **No-results state** — What happens when there are no matches?
6. **History** — Can users quickly re-do a previous search?

### Debounced Input

Never make an API call on every keystroke. Users type faster than your server can respond, and you'll flood your search provider with requests (and your bill will reflect it). Debounce at 250-300ms — this is the sweet spot between feeling responsive and not making unnecessary requests.

```typescript
// src/hooks/use-debounced-search.ts

import { useState, useEffect, useRef } from 'react';

interface UseDebouncedSearchOptions<T> {
  searchFn: (query: string) => Promise<T[]>;
  debounceMs?: number;
  minQueryLength?: number;
}

interface UseDebouncedSearchResult<T> {
  query: string;
  setQuery: (q: string) => void;
  results: T[];
  isLoading: boolean;
  error: Error | null;
  hasSearched: boolean;
}

export function useDebouncedSearch<T>({
  searchFn,
  debounceMs = 300,
  minQueryLength = 2,
}: UseDebouncedSearchOptions<T>): UseDebouncedSearchResult<T> {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState<T[]>([]);
  const [isLoading, setIsLoading] = useState(false);
  const [error, setError] = useState<Error | null>(null);
  const [hasSearched, setHasSearched] = useState(false);

  // Track the latest request to avoid race conditions
  const latestRequestRef = useRef(0);

  useEffect(() => {
    // Don't search if query is too short
    if (query.length < minQueryLength) {
      setResults([]);
      setIsLoading(false);
      setHasSearched(false);
      return;
    }

    setIsLoading(true);
    setError(null);

    const requestId = ++latestRequestRef.current;

    const timer = setTimeout(async () => {
      try {
        const searchResults = await searchFn(query);

        // Only update if this is still the latest request
        // (prevents race conditions when user types fast)
        if (requestId === latestRequestRef.current) {
          setResults(searchResults);
          setHasSearched(true);
          setIsLoading(false);
        }
      } catch (err) {
        if (requestId === latestRequestRef.current) {
          setError(err instanceof Error ? err : new Error('Search failed'));
          setResults([]);
          setIsLoading(false);
        }
      }
    }, debounceMs);

    return () => clearTimeout(timer);
  }, [query, searchFn, debounceMs, minQueryLength]);

  return { query, setQuery, results, isLoading, error, hasSearched };
}
```

### Recent Searches with MMKV

Users search for the same things repeatedly. Storing recent searches locally gives them instant access to their common queries without any network calls.

```typescript
// src/search/recent-searches.ts

import { MMKV } from 'react-native-mmkv';

const storage = new MMKV({ id: 'search-history' });
const RECENT_SEARCHES_KEY = 'recent_searches';
const MAX_RECENT_SEARCHES = 15;

export interface RecentSearch {
  query: string;
  timestamp: number;
  resultCount: number; // How many results were returned
}

/**
 * Get the user's recent searches, most recent first.
 */
export function getRecentSearches(): RecentSearch[] {
  const raw = storage.getString(RECENT_SEARCHES_KEY);
  if (!raw) return [];

  try {
    return JSON.parse(raw) as RecentSearch[];
  } catch {
    return [];
  }
}

/**
 * Add a search to the recent searches list.
 * Deduplicates by query (case-insensitive) and maintains max size.
 */
export function addRecentSearch(query: string, resultCount: number): void {
  const normalized = query.trim().toLowerCase();
  if (!normalized) return;

  const current = getRecentSearches();

  // Remove duplicate if it exists
  const filtered = current.filter(
    (s) => s.query.toLowerCase() !== normalized
  );

  // Add new search at the beginning
  const updated: RecentSearch[] = [
    { query: query.trim(), timestamp: Date.now(), resultCount },
    ...filtered,
  ].slice(0, MAX_RECENT_SEARCHES);

  storage.set(RECENT_SEARCHES_KEY, JSON.stringify(updated));
}

/**
 * Remove a specific recent search.
 */
export function removeRecentSearch(query: string): void {
  const current = getRecentSearches();
  const filtered = current.filter(
    (s) => s.query.toLowerCase() !== query.toLowerCase()
  );
  storage.set(RECENT_SEARCHES_KEY, JSON.stringify(filtered));
}

/**
 * Clear all recent searches.
 */
export function clearRecentSearches(): void {
  storage.delete(RECENT_SEARCHES_KEY);
}
```

### Search Suggestions / Autocomplete

Autocomplete reduces typing effort and guides users toward queries that will actually return results. It's the difference between a search that feels helpful and one that feels like a guessing game.

```typescript
// src/search/suggestions.ts

/**
 * Generate search suggestions from multiple sources:
 * 1. Recent searches that match the current input
 * 2. Popular searches (fetched periodically from the server)
 * 3. Autocomplete from your search provider
 */

interface SearchSuggestion {
  text: string;
  type: 'recent' | 'popular' | 'autocomplete';
  icon: 'clock' | 'trending' | 'search';
}

export function getLocalSuggestions(
  partialQuery: string,
  recentSearches: RecentSearch[],
  popularSearches: string[]
): SearchSuggestion[] {
  const normalized = partialQuery.toLowerCase().trim();
  if (!normalized) return [];

  const suggestions: SearchSuggestion[] = [];

  // Recent searches that start with the query
  const matchingRecent = recentSearches
    .filter((s) => s.query.toLowerCase().startsWith(normalized))
    .slice(0, 3)
    .map((s) => ({
      text: s.query,
      type: 'recent' as const,
      icon: 'clock' as const,
    }));
  suggestions.push(...matchingRecent);

  // Popular searches that start with the query
  const matchingPopular = popularSearches
    .filter((s) => s.toLowerCase().startsWith(normalized))
    .filter((s) => !matchingRecent.some((r) => r.text.toLowerCase() === s.toLowerCase()))
    .slice(0, 3)
    .map((s) => ({
      text: s,
      type: 'popular' as const,
      icon: 'trending' as const,
    }));
  suggestions.push(...matchingPopular);

  return suggestions.slice(0, 6); // Max 6 suggestions
}
```

### Highlighting Matches

When users search, they need to see *why* a result matched. Highlighting the matching text in the result makes the connection obvious and helps users scan results quickly.

```typescript
// src/search/highlight.tsx

import React from 'react';
import { Text, type TextStyle } from 'react-native';

interface HighlightedTextProps {
  text: string;
  highlight: string;
  style?: TextStyle;
  highlightStyle?: TextStyle;
}

/**
 * Renders text with highlighted matching portions.
 * 
 * Example:
 * <HighlightedText text="Running Shoes Pro" highlight="run" />
 * Renders: [Run]ning Shoes Pro  (with Run highlighted)
 */
export function HighlightedText({
  text,
  highlight,
  style,
  highlightStyle,
}: HighlightedTextProps) {
  if (!highlight.trim()) {
    return <Text style={style}>{text}</Text>;
  }

  // Escape regex special characters in the highlight string
  const escaped = highlight.replace(/[.*+?^${}()|[\]\\]/g, '\\$&');
  const regex = new RegExp(`(${escaped})`, 'gi');
  const parts = text.split(regex);

  return (
    <Text style={style}>
      {parts.map((part, index) =>
        regex.test(part) ? (
          <Text
            key={index}
            style={[
              { backgroundColor: '#FEF08A', fontWeight: '600' },
              highlightStyle,
            ]}
          >
            {part}
          </Text>
        ) : (
          <Text key={index}>{part}</Text>
        )
      )}
    </Text>
  );
}

/**
 * For HTML-based highlighting (from Algolia/Typesense that return
 * pre-highlighted HTML), use this helper to parse the HTML marks.
 */
export function parseHighlightedHTML(html: string): Array<{
  text: string;
  highlighted: boolean;
}> {
  const parts: Array<{ text: string; highlighted: boolean }> = [];
  const regex = /<mark>(.*?)<\/mark>/gi;
  let lastIndex = 0;
  let match;

  while ((match = regex.exec(html)) !== null) {
    // Text before the match
    if (match.index > lastIndex) {
      parts.push({
        text: html.slice(lastIndex, match.index),
        highlighted: false,
      });
    }

    // The highlighted match
    parts.push({
      text: match[1],
      highlighted: true,
    });

    lastIndex = match.index + match[0].length;
  }

  // Remaining text after the last match
  if (lastIndex < html.length) {
    parts.push({
      text: html.slice(lastIndex),
      highlighted: false,
    });
  }

  return parts;
}
```

---

## 3. ALGOLIA

### The Enterprise Standard

Algolia is the search engine you compare everything else against. It's fast (single-digit millisecond queries, globally), it has incredible relevance out of the box, and the developer experience is genuinely delightful. It's also expensive, especially at scale.

**Why teams pick Algolia:**
- Speed: < 5ms query time globally (they have data centers everywhere)
- Typo tolerance: Searching "iphne" matches "iPhone." Out of the box, zero config.
- Relevance: The ranking formula is incredibly well-tuned by default
- InstantSearch: Pre-built UI components for React (and React Native)
- Analytics: Built-in search analytics, click tracking, A/B testing
- Faceting: Filter by any attribute, get counts, range filters
- Geo search: Search within a geographic area

**Why teams leave Algolia:**
- Cost: $1+ per 1,000 search requests on the paid plans, which adds up fast
- Vendor lock-in: The SDK and query syntax are Algolia-specific
- Data residency: Limited control over where your data lives

### Setting Up Algolia

```bash
# Install Algolia packages
npm install algoliasearch react-instantsearch
# For React Native:
npx expo install algoliasearch react-instantsearch-core
```

```typescript
// src/lib/algolia.ts — Client setup

import algoliasearch from 'algoliasearch';

// Use the search-only API key (safe for client-side)
// NEVER expose the admin API key in client code
export const searchClient = algoliasearch(
  process.env.EXPO_PUBLIC_ALGOLIA_APP_ID!,
  process.env.EXPO_PUBLIC_ALGOLIA_SEARCH_KEY!
);

// Index names — keep them consistent across environments
export const ALGOLIA_INDEXES = {
  products: `products_${process.env.EXPO_PUBLIC_ENV}`,          // products_production
  products_price_asc: `products_${process.env.EXPO_PUBLIC_ENV}_price_asc`,
  products_price_desc: `products_${process.env.EXPO_PUBLIC_ENV}_price_desc`,
  articles: `articles_${process.env.EXPO_PUBLIC_ENV}`,
  users: `users_${process.env.EXPO_PUBLIC_ENV}`,
} as const;
```

### Indexing Data with Algolia

```typescript
// scripts/sync-algolia-index.ts — Server-side indexing script

import algoliasearch from 'algoliasearch';
import { db } from '../src/lib/database';

const client = algoliasearch(
  process.env.ALGOLIA_APP_ID!,
  process.env.ALGOLIA_ADMIN_KEY! // Admin key — server-side only!
);

const productsIndex = client.initIndex('products_production');

async function syncProductsToAlgolia() {
  console.log('Fetching products from database...');

  const products = await db.query.products.findMany({
    with: { category: true, brand: true },
  });

  // Transform database records to Algolia records
  const algoliaRecords = products.map((product) => ({
    // objectID is required by Algolia — use your database ID
    objectID: product.id,

    // Searchable attributes
    name: product.name,
    description: product.description,
    category: product.category.name,
    brand: product.brand.name,
    tags: product.tags,

    // Attributes for filtering (facets)
    price: product.priceInCents / 100,
    rating: product.averageRating,
    inStock: product.inventory > 0,
    color: product.color,
    size: product.size,

    // Attributes for display (not searchable)
    imageUrl: product.imageUrl,
    slug: product.slug,

    // Geo search (if applicable)
    _geoloc: product.latitude && product.longitude
      ? { lat: product.latitude, lng: product.longitude }
      : undefined,
  }));

  console.log(`Syncing ${algoliaRecords.length} products to Algolia...`);

  // Use saveObjects for full sync — it creates or updates records
  await productsIndex.saveObjects(algoliaRecords);

  console.log('Sync complete!');
}

// Configure index settings (run once, or when settings change)
async function configureIndex() {
  await productsIndex.setSettings({
    // Which attributes to search in, and in what order of importance
    searchableAttributes: [
      'name',           // Most important — searched first
      'brand',
      'category',
      'tags',
      'description',    // Least important — searched last
    ],

    // Attributes that can be used as filters
    attributesForFaceting: [
      'category',
      'brand',
      'color',
      'size',
      'filterOnly(inStock)',    // Can filter on, but not facet
      'searchable(tags)',        // Can both filter and search within facet values
    ],

    // Custom ranking (after Algolia's built-in relevance)
    customRanking: [
      'desc(rating)',       // Higher rated products first
      'desc(popularity)',   // More popular products first
    ],

    // Highlighting
    attributesToHighlight: ['name', 'description'],
    highlightPreTag: '<mark>',
    highlightPostTag: '</mark>',

    // Snippeting (truncated description with highlights)
    attributesToSnippet: ['description:30'],

    // Pagination
    hitsPerPage: 20,

    // Typo tolerance
    typoTolerance: true,
    minWordSizefor1Typo: 3,  // Allow 1 typo for words >= 3 chars
    minWordSizefor2Typos: 7, // Allow 2 typos for words >= 7 chars
  });

  console.log('Index settings configured!');
}
```

### Algolia with React Native (InstantSearch)

```typescript
// src/features/search/algolia-search-screen.tsx

import React, { useRef, useState } from 'react';
import {
  View,
  TextInput,
  FlatList,
  Text,
  Pressable,
  ActivityIndicator,
  StyleSheet,
  Keyboard,
} from 'react-native';
import {
  InstantSearch,
  useSearchBox,
  useInfiniteHits,
} from 'react-instantsearch-core';
import { searchClient, ALGOLIA_INDEXES } from '../../lib/algolia';
import { HighlightedText } from '../../search/highlight';
import {
  getRecentSearches,
  addRecentSearch,
  removeRecentSearch,
  clearRecentSearches,
} from '../../search/recent-searches';

/**
 * Complete Algolia-powered search screen for React Native.
 */
export function AlgoliaSearchScreen() {
  return (
    <InstantSearch
      searchClient={searchClient}
      indexName={ALGOLIA_INDEXES.products}
    >
      <SearchScreenContent />
    </InstantSearch>
  );
}

function SearchScreenContent() {
  const { query, refine: setQuery } = useSearchBox();
  const [isFocused, setIsFocused] = useState(false);
  const inputRef = useRef<TextInput>(null);

  const showRecentSearches = isFocused && !query;
  const showResults = query.length >= 2;

  return (
    <View style={styles.container}>
      {/* Search Input */}
      <View style={styles.inputContainer}>
        <TextInput
          ref={inputRef}
          style={styles.input}
          value={query}
          onChangeText={setQuery}
          onFocus={() => setIsFocused(true)}
          onBlur={() => setIsFocused(false)}
          placeholder="Search products..."
          placeholderTextColor="#9CA3AF"
          returnKeyType="search"
          autoCapitalize="none"
          autoCorrect={false}
          clearButtonMode="while-editing"
          accessibilityLabel="Search products"
          accessibilityRole="search"
        />
        {query.length > 0 && (
          <Pressable
            onPress={() => {
              setQuery('');
              inputRef.current?.focus();
            }}
            style={styles.clearButton}
            accessibilityLabel="Clear search"
          >
            <Text style={styles.clearButtonText}>Clear</Text>
          </Pressable>
        )}
      </View>

      {/* Recent Searches (when input is focused but empty) */}
      {showRecentSearches && (
        <RecentSearchesList
          onSelect={(q) => {
            setQuery(q);
            Keyboard.dismiss();
          }}
        />
      )}

      {/* Search Results */}
      {showResults && <SearchResults query={query} />}

      {/* Empty State (not focused, no query) */}
      {!isFocused && !query && <SearchEmptyState />}
    </View>
  );
}

function SearchResults({ query }: { query: string }) {
  const { hits, isLastPage, showMore } = useInfiniteHits<{
    objectID: string;
    name: string;
    description: string;
    category: string;
    price: number;
    rating: number;
    imageUrl: string;
    _highlightResult: {
      name: { value: string };
      description: { value: string };
    };
  }>();

  // Track the search for recent searches
  React.useEffect(() => {
    if (hits.length > 0) {
      addRecentSearch(query, hits.length);
    }
  }, [query, hits.length]);

  if (hits.length === 0) {
    return <NoResultsState query={query} />;
  }

  return (
    <FlatList
      data={hits}
      keyExtractor={(item) => item.objectID}
      keyboardShouldPersistTaps="handled"
      keyboardDismissMode="on-drag"
      onEndReached={() => {
        if (!isLastPage) showMore();
      }}
      onEndReachedThreshold={0.5}
      renderItem={({ item }) => (
        <SearchResultItem
          name={item.name}
          description={item.description}
          category={item.category}
          price={item.price}
          rating={item.rating}
          imageUrl={item.imageUrl}
          highlightedName={item._highlightResult?.name?.value}
          highlightedDescription={item._highlightResult?.description?.value}
          query={query}
        />
      )}
      ListFooterComponent={
        !isLastPage ? (
          <ActivityIndicator style={styles.loadingMore} color="#6366F1" />
        ) : null
      }
    />
  );
}

function SearchResultItem({
  name,
  description,
  category,
  price,
  rating,
  imageUrl,
  highlightedName,
  highlightedDescription,
  query,
}: {
  name: string;
  description: string;
  category: string;
  price: number;
  rating: number;
  imageUrl: string;
  highlightedName?: string;
  highlightedDescription?: string;
  query: string;
}) {
  return (
    <Pressable style={styles.resultItem} accessibilityRole="button">
      {/* Product Image */}
      <View style={styles.resultImage}>
        <View style={styles.thumbnailPlaceholder} />
      </View>

      {/* Product Info */}
      <View style={styles.resultInfo}>
        {/* Name with highlights */}
        <HighlightedText
          text={name}
          highlight={query}
          style={styles.resultName}
          highlightStyle={styles.highlight}
        />

        {/* Category */}
        <Text style={styles.resultCategory}>{category}</Text>

        {/* Description snippet */}
        <Text style={styles.resultDescription} numberOfLines={2}>
          {description}
        </Text>

        {/* Price and Rating */}
        <View style={styles.resultMeta}>
          <Text style={styles.resultPrice}>${price.toFixed(2)}</Text>
          <Text style={styles.resultRating}>
            {rating.toFixed(1)}
          </Text>
        </View>
      </View>
    </Pressable>
  );
}

function RecentSearchesList({
  onSelect,
}: {
  onSelect: (query: string) => void;
}) {
  const [searches, setSearches] = useState(getRecentSearches());

  if (searches.length === 0) return null;

  return (
    <View style={styles.recentContainer}>
      <View style={styles.recentHeader}>
        <Text style={styles.recentTitle}>Recent Searches</Text>
        <Pressable
          onPress={() => {
            clearRecentSearches();
            setSearches([]);
          }}
          accessibilityLabel="Clear recent searches"
        >
          <Text style={styles.clearAllText}>Clear All</Text>
        </Pressable>
      </View>

      {searches.map((search) => (
        <Pressable
          key={search.query}
          style={styles.recentItem}
          onPress={() => onSelect(search.query)}
          accessibilityRole="button"
          accessibilityLabel={`Search for ${search.query}`}
        >
          <Text style={styles.recentQuery}>{search.query}</Text>
          <Pressable
            onPress={() => {
              removeRecentSearch(search.query);
              setSearches(getRecentSearches());
            }}
            hitSlop={8}
            accessibilityLabel={`Remove ${search.query} from recent searches`}
          >
            <Text style={styles.removeIcon}>x</Text>
          </Pressable>
        </Pressable>
      ))}
    </View>
  );
}

function NoResultsState({ query }: { query: string }) {
  return (
    <View style={styles.emptyState}>
      <Text style={styles.emptyStateTitle}>No results for "{query}"</Text>
      <Text style={styles.emptyStateSubtitle}>
        Try checking your spelling or using different keywords.
      </Text>
    </View>
  );
}

function SearchEmptyState() {
  return (
    <View style={styles.emptyState}>
      <Text style={styles.emptyStateTitle}>Search for products</Text>
      <Text style={styles.emptyStateSubtitle}>
        Find what you're looking for by name, category, or brand.
      </Text>
    </View>
  );
}

const styles = StyleSheet.create({
  container: {
    flex: 1,
    backgroundColor: '#FFFFFF',
  },
  inputContainer: {
    flexDirection: 'row',
    alignItems: 'center',
    paddingHorizontal: 16,
    paddingVertical: 8,
    borderBottomWidth: 1,
    borderBottomColor: '#E5E7EB',
  },
  input: {
    flex: 1,
    height: 44,
    backgroundColor: '#F3F4F6',
    borderRadius: 10,
    paddingHorizontal: 16,
    fontSize: 16,
    color: '#1F2937',
  },
  clearButton: {
    marginLeft: 12,
    paddingVertical: 8,
  },
  clearButtonText: {
    color: '#6366F1',
    fontSize: 16,
    fontWeight: '500',
  },
  resultItem: {
    flexDirection: 'row',
    padding: 16,
    borderBottomWidth: 1,
    borderBottomColor: '#F3F4F6',
  },
  resultImage: {
    marginRight: 12,
  },
  thumbnailPlaceholder: {
    width: 80,
    height: 80,
    backgroundColor: '#F3F4F6',
    borderRadius: 8,
  },
  resultInfo: {
    flex: 1,
  },
  resultName: {
    fontSize: 16,
    fontWeight: '600',
    color: '#1F2937',
    marginBottom: 2,
  },
  highlight: {
    backgroundColor: '#FEF08A',
    fontWeight: '700',
  },
  resultCategory: {
    fontSize: 13,
    color: '#6B7280',
    marginBottom: 4,
  },
  resultDescription: {
    fontSize: 14,
    color: '#4B5563',
    lineHeight: 20,
    marginBottom: 6,
  },
  resultMeta: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    alignItems: 'center',
  },
  resultPrice: {
    fontSize: 16,
    fontWeight: '700',
    color: '#1F2937',
  },
  resultRating: {
    fontSize: 13,
    color: '#F59E0B',
  },
  recentContainer: {
    padding: 16,
  },
  recentHeader: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    alignItems: 'center',
    marginBottom: 12,
  },
  recentTitle: {
    fontSize: 15,
    fontWeight: '600',
    color: '#374151',
  },
  clearAllText: {
    fontSize: 14,
    color: '#6366F1',
  },
  recentItem: {
    flexDirection: 'row',
    alignItems: 'center',
    paddingVertical: 10,
  },
  recentQuery: {
    flex: 1,
    fontSize: 15,
    color: '#1F2937',
  },
  removeIcon: {
    fontSize: 14,
    color: '#9CA3AF',
    padding: 4,
  },
  emptyState: {
    flex: 1,
    justifyContent: 'center',
    alignItems: 'center',
    paddingHorizontal: 32,
  },
  emptyStateTitle: {
    fontSize: 18,
    fontWeight: '600',
    color: '#1F2937',
    textAlign: 'center',
    marginBottom: 8,
  },
  emptyStateSubtitle: {
    fontSize: 14,
    color: '#6B7280',
    textAlign: 'center',
    lineHeight: 20,
  },
  loadingMore: {
    padding: 16,
  },
});
```

### Algolia Query Rules

Query rules let you customize search results without writing code. Want to boost certain products when someone searches "sale"? Want to show a banner when someone searches "return policy"? Query rules.

```typescript
// scripts/configure-algolia-rules.ts — Server-side

import algoliasearch from 'algoliasearch';

const client = algoliasearch(
  process.env.ALGOLIA_APP_ID!,
  process.env.ALGOLIA_ADMIN_KEY!
);
const index = client.initIndex('products_production');

async function configureQueryRules() {
  await index.saveRules([
    // Rule 1: When someone searches "sale", boost items that are on sale
    {
      objectID: 'boost-sale-items',
      conditions: [
        {
          anchoring: 'contains',
          pattern: 'sale',
        },
      ],
      consequence: {
        params: {
          filters: 'onSale:true',
          optionalFilters: ['onSale:true<score=10>'],
        },
      },
    },

    // Rule 2: When someone searches "free shipping", filter to eligible items
    {
      objectID: 'free-shipping-filter',
      conditions: [
        {
          anchoring: 'contains',
          pattern: 'free shipping',
        },
      ],
      consequence: {
        params: {
          filters: 'freeShipping:true',
        },
        filterPromotes: true,
      },
    },

    // Rule 3: Pin a specific product for a brand search
    {
      objectID: 'pin-featured-nike',
      conditions: [
        {
          anchoring: 'is',
          pattern: 'nike',
        },
      ],
      consequence: {
        promote: [
          {
            objectID: 'nike-air-max-2026',  // Always show this first
            position: 0,
          },
        ],
      },
    },
  ]);

  console.log('Query rules configured!');
}
```

### Algolia Pricing Considerations

Let me be straight with you about Algolia pricing, because it catches a lot of teams off guard:

```
┌──────────────────────────────────────────────────────────────────────┐
│                   ALGOLIA PRICING REALITY CHECK                      │
│                                                                      │
│  Free tier:    10K search requests/month, 10K records               │
│  Build plan:   Starts around $0.50/1K requests + record storage     │
│  Grow plan:    Starts around $1/1K requests + more features         │
│  Premium:      Custom pricing (enterprise features)                  │
│                                                                      │
│  WHAT COUNTS AS A "SEARCH REQUEST":                                 │
│  - Every keystroke in search-as-you-type = 1 request                │
│  - A user typing "running shoes" = roughly 14 requests              │
│  - With debounce (300ms): roughly 3-5 requests                      │
│  - Facet queries count as additional requests                        │
│                                                                      │
│  EXAMPLE: 10K DAU, each user searches 3x/day, 5 keystrokes avg     │
│  Without debounce: 10,000 x 3 x 14 = 420,000 requests/day         │
│  With debounce:    10,000 x 3 x 4  = 120,000 requests/day          │
│  Monthly:          3.6M requests (without) / 1M (with debounce)     │
│                                                                      │
│  At $1/1K requests: $1,000-$3,600/month                             │
│                                                                      │
│  TIPS TO REDUCE COST:                                                │
│  1. Always debounce (250-300ms)                                      │
│  2. Set a minimum query length (2-3 chars)                           │
│  3. Cache frequent queries client-side                               │
│  4. Use Algolia Recommend instead of extra search calls              │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 4. TYPESENSE

### The Open-Source Algolia Alternative

Typesense is what happens when someone looks at Algolia and says "this is great, but I want to self-host it and not pay $3,000/month." It's an open-source search engine built in C++, designed specifically for instant search. It's fast, has excellent typo tolerance, and the self-hosted version is genuinely production-ready.

**Why Typesense:**
- Open-source (GPLv3) with a managed cloud option
- Written in C++ — extremely fast, low memory footprint
- Typo tolerance that rivals Algolia
- Simpler API than Elasticsearch (which is both a strength and limitation)
- Self-hostable on a single server (no cluster required for reasonable scales)
- Built-in geo search, faceting, and curation

**Compared to Algolia:**
- 10-100x cheaper (self-hosted is just server costs; Typesense Cloud starts much lower)
- Slightly less sophisticated ranking algorithm
- Fewer UI components (but getting better)
- No built-in analytics (pair with your own analytics)
- Community smaller but growing fast

### Setting Up Typesense

```bash
# Self-hosted with Docker
docker run -p 8108:8108 \
  -v /tmp/typesense-data:/data \
  typesense/typesense:27.1 \
  --data-dir /data \
  --api-key=your-api-key-here \
  --enable-cors
```

```bash
# Install the JavaScript client
npm install typesense
# For React InstantSearch compatibility:
npm install typesense-instantsearch-adapter
```

### Typesense Collection Schema

```typescript
// scripts/setup-typesense.ts

import Typesense from 'typesense';

const typesenseClient = new Typesense.Client({
  nodes: [
    {
      host: process.env.TYPESENSE_HOST!,
      port: 8108,
      protocol: 'https',
    },
  ],
  apiKey: process.env.TYPESENSE_ADMIN_KEY!,
  connectionTimeoutSeconds: 5,
});

// Define the collection schema
async function createProductsCollection() {
  const schema = {
    name: 'products',
    fields: [
      // Searchable string fields
      { name: 'name', type: 'string' as const },
      { name: 'description', type: 'string' as const },
      { name: 'category', type: 'string' as const, facet: true },
      { name: 'brand', type: 'string' as const, facet: true },
      { name: 'tags', type: 'string[]' as const, facet: true },

      // Numeric fields (for filtering and sorting)
      { name: 'price', type: 'float' as const },
      { name: 'rating', type: 'float' as const },
      { name: 'popularity', type: 'int32' as const },

      // Boolean fields
      { name: 'in_stock', type: 'bool' as const, facet: true },

      // Geo field
      { name: 'location', type: 'geopoint' as const, optional: true },

      // Display-only fields (not searchable by default)
      { name: 'image_url', type: 'string' as const, index: false },
      { name: 'slug', type: 'string' as const, index: false },

      // Sort field
      { name: 'created_at', type: 'int64' as const },
    ],

    // Default sorting
    default_sorting_field: 'popularity',

    // Token separators (for searching "t-shirt" when user types "tshirt")
    token_separators: ['-', '/'],

    // Symbols to index (for searching "$99" or "C++")
    symbols_to_index: ['$', '+'],
  };

  try {
    await typesenseClient.collections().create(schema);
    console.log('Collection created!');
  } catch (error: any) {
    if (error.httpStatus === 409) {
      console.log('Collection already exists.');
    } else {
      throw error;
    }
  }
}
```

### Indexing Data to Typesense

```typescript
// scripts/sync-typesense.ts

async function syncProductsToTypesense() {
  const products = await db.query.products.findMany({
    with: { category: true, brand: true },
  });

  const documents = products.map((product) => ({
    id: product.id,
    name: product.name,
    description: product.description ?? '',
    category: product.category.name,
    brand: product.brand.name,
    tags: product.tags ?? [],
    price: product.priceInCents / 100,
    rating: product.averageRating,
    popularity: product.viewCount,
    in_stock: product.inventory > 0,
    location: product.latitude && product.longitude
      ? [product.latitude, product.longitude]
      : undefined,
    image_url: product.imageUrl,
    slug: product.slug,
    created_at: Math.floor(new Date(product.createdAt).getTime() / 1000),
  }));

  console.log(`Importing ${documents.length} documents...`);

  // Batch import — much faster than individual inserts
  const results = await typesenseClient
    .collections('products')
    .documents()
    .import(documents, { action: 'upsert' });

  // Check for errors
  const errors = results.filter((r) => !r.success);
  if (errors.length > 0) {
    console.error(`${errors.length} documents failed to import:`, errors.slice(0, 5));
  } else {
    console.log('All documents imported successfully!');
  }
}
```

### Searching with Typesense

```typescript
// src/lib/typesense-search.ts

import Typesense from 'typesense';

// Search-only client (safe for client-side)
const searchClient = new Typesense.Client({
  nodes: [
    {
      host: process.env.EXPO_PUBLIC_TYPESENSE_HOST!,
      port: 443,
      protocol: 'https',
    },
  ],
  // Use a search-only API key — NOT the admin key
  apiKey: process.env.EXPO_PUBLIC_TYPESENSE_SEARCH_KEY!,
  connectionTimeoutSeconds: 3,
});

interface SearchOptions {
  query: string;
  page?: number;
  perPage?: number;
  filters?: string;
  sortBy?: string;
  facetBy?: string[];
  geoPoint?: { lat: number; lng: number; radiusKm: number };
}

export async function searchProducts(options: SearchOptions) {
  const {
    query,
    page = 1,
    perPage = 20,
    filters,
    sortBy,
    facetBy = ['category', 'brand', 'in_stock'],
    geoPoint,
  } = options;

  const searchParameters: Record<string, any> = {
    q: query,
    query_by: 'name,brand,category,tags,description',
    query_by_weights: '5,3,3,2,1',  // Name matches weighted highest

    // Highlighting
    highlight_full_fields: 'name',
    highlight_start_tag: '<mark>',
    highlight_end_tag: '</mark>',

    // Pagination
    page,
    per_page: perPage,

    // Facets
    facet_by: facetBy.join(','),
    max_facet_values: 20,

    // Typo tolerance
    num_typos: 2,
    typo_tokens_threshold: 3,

    // Snippets for description
    snippet_threshold: 30,
  };

  // Optional filters
  if (filters) {
    searchParameters.filter_by = filters;
  }

  // Optional sorting
  if (sortBy) {
    searchParameters.sort_by = sortBy;
  }

  // Geo search
  if (geoPoint) {
    searchParameters.filter_by = [
      searchParameters.filter_by,
      `location:(${geoPoint.lat}, ${geoPoint.lng}, ${geoPoint.radiusKm} km)`,
    ].filter(Boolean).join(' && ');
    searchParameters.sort_by = `location(${geoPoint.lat}, ${geoPoint.lng}):asc`;
  }

  const result = await searchClient
    .collections('products')
    .documents()
    .search(searchParameters);

  return {
    hits: result.hits?.map((hit) => ({
      ...hit.document,
      highlights: hit.highlights,
      textMatch: hit.text_match,
    })) ?? [],
    totalFound: result.found,
    facets: result.facet_counts?.reduce((acc, facet) => {
      acc[facet.field_name] = facet.counts.map((c) => ({
        value: c.value,
        count: c.count,
      }));
      return acc;
    }, {} as Record<string, Array<{ value: string; count: number }>>) ?? {},
    page,
    totalPages: Math.ceil((result.found ?? 0) / perPage),
  };
}

// Example usage:
//
// Basic search:
// searchProducts({ query: 'running shoes' })
//
// With filters:
// searchProducts({
//   query: 'shoes',
//   filters: 'category:=Running && price:<100 && in_stock:true',
// })
//
// Sorted by price:
// searchProducts({
//   query: 'shoes',
//   sortBy: 'price:asc',
// })
//
// Geo search (within 10km):
// searchProducts({
//   query: 'coffee shop',
//   geoPoint: { lat: 40.7128, lng: -74.0060, radiusKm: 10 },
// })
```

### Using Typesense with React InstantSearch

Typesense provides an adapter that makes it compatible with Algolia's InstantSearch UI components. This means you can use the same InstantSearch React components with either Algolia or Typesense, and switch between them with minimal code changes.

```typescript
// src/lib/typesense-instantsearch.ts

import TypesenseInstantSearchAdapter from 'typesense-instantsearch-adapter';

const typesenseAdapter = new TypesenseInstantSearchAdapter({
  server: {
    apiKey: process.env.EXPO_PUBLIC_TYPESENSE_SEARCH_KEY!,
    nodes: [
      {
        host: process.env.EXPO_PUBLIC_TYPESENSE_HOST!,
        port: 443,
        protocol: 'https',
      },
    ],
  },
  additionalSearchParameters: {
    query_by: 'name,brand,category,tags,description',
    query_by_weights: '5,3,3,2,1',
    num_typos: 2,
  },
});

export const typesenseSearchClient = typesenseAdapter.searchClient;

// Now use it the same way as Algolia:
//
// <InstantSearch
//   searchClient={typesenseSearchClient}
//   indexName="products"
// >
//   ... same components as the Algolia example ...
// </InstantSearch>
```

---

## 5. MEILISEARCH

### The Simplicity-First Search Engine

Meilisearch is the search engine for teams that want good search without becoming search engineers. It's opinionated about defaults (which means less configuration), easy to self-host, and has a clean REST API. If Typesense is "Algolia but open-source," Meilisearch is "Algolia but simpler."

**Why Meilisearch:**
- Extremely easy to get started (literally one Docker command and you're searching)
- Great defaults — typo tolerance, relevance ranking, and highlighting work out of the box
- Clean REST API (no SDK required, though SDKs exist)
- Auto-batching of index operations (you don't need to manually batch)
- Built-in multi-tenant support (tenant tokens)
- Generous open-source license (MIT)

**Compared to Typesense:**
- Simpler API (fewer configuration options, which is both pro and con)
- Written in Rust (Typesense is C++) — both are fast
- Better multi-tenancy support
- Less fine-grained control over ranking and relevance
- Slightly higher memory usage for large datasets
- Growing ecosystem, but smaller than Typesense

```bash
# Get Meilisearch running in 30 seconds
docker run -p 7700:7700 \
  -v /tmp/meili-data:/meili_data \
  getmeili/meilisearch:v1.12 \
  --master-key=your-master-key-here
```

```typescript
// src/lib/meilisearch.ts

import { MeiliSearch } from 'meilisearch';

// Admin client (server-side only)
const adminClient = new MeiliSearch({
  host: process.env.MEILISEARCH_HOST!,
  apiKey: process.env.MEILISEARCH_ADMIN_KEY!,
});

// Search client (safe for client-side — uses search-only key)
export const searchClient = new MeiliSearch({
  host: process.env.EXPO_PUBLIC_MEILISEARCH_HOST!,
  apiKey: process.env.EXPO_PUBLIC_MEILISEARCH_SEARCH_KEY!,
});

// Index configuration
async function configureIndex() {
  const index = adminClient.index('products');

  // Set searchable attributes (order matters — first = highest relevance)
  await index.updateSearchableAttributes([
    'name',
    'brand',
    'category',
    'tags',
    'description',
  ]);

  // Set filterable attributes (for faceted search)
  await index.updateFilterableAttributes([
    'category',
    'brand',
    'price',
    'rating',
    'in_stock',
    'color',
    '_geo', // For geo search
  ]);

  // Set sortable attributes
  await index.updateSortableAttributes([
    'price',
    'rating',
    'created_at',
  ]);

  // Set displayed attributes (what's returned in results)
  await index.updateDisplayedAttributes([
    'id', 'name', 'description', 'category', 'brand',
    'price', 'rating', 'image_url', 'slug', 'in_stock',
  ]);

  // Configure typo tolerance
  await index.updateTypoTolerance({
    enabled: true,
    minWordSizeForTypos: {
      oneTypo: 4,
      twoTypos: 8,
    },
  });

  console.log('Meilisearch index configured!');
}

// Search function
export async function searchProducts(query: string, options?: {
  filters?: string;
  sort?: string[];
  page?: number;
  hitsPerPage?: number;
  facets?: string[];
}) {
  const index = searchClient.index('products');

  const result = await index.search(query, {
    filter: options?.filters,
    sort: options?.sort,
    page: options?.page ?? 1,
    hitsPerPage: options?.hitsPerPage ?? 20,
    facets: options?.facets ?? ['category', 'brand', 'in_stock'],
    attributesToHighlight: ['name', 'description'],
    highlightPreTag: '<mark>',
    highlightPostTag: '</mark>',
    attributesToCrop: ['description'],
    cropLength: 30,
  });

  return {
    hits: result.hits,
    totalHits: result.totalHits,
    facetDistribution: result.facetDistribution,
    page: result.page,
    totalPages: result.totalPages,
    processingTimeMs: result.processingTimeMs,
  };
}

// Usage:
// searchProducts('running shoes', {
//   filters: 'category = "Footwear" AND price < 100 AND in_stock = true',
//   sort: ['price:asc'],
//   facets: ['category', 'brand', 'color'],
// })
```

### Typesense vs Meilisearch — The Honest Comparison

```
┌──────────────────────────────────────────────────────────────────────┐
│               TYPESENSE vs MEILISEARCH                               │
│                                                                      │
│  Factor                Typesense          Meilisearch               │
│  ──────                ─────────          ───────────               │
│  Language              C++                Rust                       │
│  License               GPLv3              MIT                        │
│  Setup complexity      Low                Very low                   │
│  API complexity        Moderate           Simple                     │
│  Typo tolerance        Excellent          Excellent                  │
│  Relevance tuning      More control       Fewer knobs               │
│  Faceted search        Yes                Yes                        │
│  Geo search            Yes                Yes                        │
│  Multi-tenancy         Basic              Built-in (tenant tokens)  │
│  InstantSearch compat  Via adapter        Via adapter               │
│  Managed cloud         Yes (paid)         Yes (paid)                │
│  Memory usage          Lower              Higher                     │
│  Max dataset size      Larger (tested)    Smaller (improving)       │
│  Community size        Larger             Growing fast               │
│  Documentation         Good               Excellent                  │
│                                                                      │
│  MY TAKE:                                                            │
│  - Need fine-grained control? Typesense                              │
│  - Want simplest setup?       Meilisearch                            │
│  - Multi-tenant SaaS?         Meilisearch (tenant tokens)           │
│  - Large dataset (1M+)?       Typesense (more battle-tested)        │
│  - Either one is a huge upgrade from Postgres LIKE queries.          │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 6. DATABASE SEARCH

### When Postgres Is Enough

I said it in the decision tree and I'll say it again: **Postgres full-text search is good enough for many apps.** If you don't need typo tolerance, if your dataset is under 100K records, and if you're already running Postgres, you might not need Algolia or Typesense.

The advantage of Postgres FTS is zero additional infrastructure. Your search is always in sync with your data (it's the same database!). There's no indexing pipeline, no webhook to keep in sync, no separate service to monitor. This simplicity has real value.

### Advanced Postgres FTS with Drizzle

```typescript
// src/lib/postgres-search.ts

import { sql, type SQL } from 'drizzle-orm';
import { db } from './database';

/**
 * Full-text search with Postgres, supporting:
 * - Weighted search across multiple fields
 * - Prefix matching (search-as-you-type)
 * - Highlighted snippets
 * - Relevance ranking
 */
export async function fullTextSearch(params: {
  query: string;
  table: string;
  limit?: number;
  offset?: number;
  filters?: SQL;
}) {
  const { query, table, limit = 20, offset = 0, filters } = params;

  if (!query.trim()) return { results: [], total: 0 };

  // Convert user query to tsquery format
  // "running shoes" becomes "running:* & shoes:*" (prefix matching)
  const tsqueryTerms = query
    .trim()
    .split(/\s+/)
    .map((term) => `${term}:*`)
    .join(' & ');

  const results = await db.execute(sql`
    WITH search_results AS (
      SELECT
        *,
        ts_rank(search_vector, to_tsquery('english', ${tsqueryTerms})) AS relevance,
        ts_headline(
          'english',
          name,
          to_tsquery('english', ${tsqueryTerms}),
          'StartSel=<mark>, StopSel=</mark>, MaxWords=50, MinWords=10'
        ) AS highlighted_name,
        ts_headline(
          'english',
          description,
          to_tsquery('english', ${tsqueryTerms}),
          'StartSel=<mark>, StopSel=</mark>, MaxWords=35, MinWords=15'
        ) AS highlighted_description,
        COUNT(*) OVER() AS total_count
      FROM ${sql.raw(table)}
      WHERE search_vector @@ to_tsquery('english', ${tsqueryTerms})
      ${filters ? sql`AND ${filters}` : sql``}
      ORDER BY relevance DESC
      LIMIT ${limit}
      OFFSET ${offset}
    )
    SELECT * FROM search_results
  `);

  return {
    results: results.rows,
    total: results.rows[0]?.total_count ?? 0,
  };
}

/**
 * Search suggestions based on existing data.
 * Returns distinct values that match the prefix.
 */
export async function getSearchSuggestions(
  partialQuery: string,
  limit = 5
): Promise<string[]> {
  if (partialQuery.length < 2) return [];

  const results = await db.execute(sql`
    SELECT DISTINCT name
    FROM products
    WHERE name ILIKE ${partialQuery + '%'}
    ORDER BY name
    LIMIT ${limit}
  `);

  return results.rows.map((r: any) => r.name);
}
```

### Adding Trigram Similarity (Poor Man's Typo Tolerance)

Postgres doesn't have built-in typo tolerance, but the `pg_trgm` extension gets you partway there:

```sql
-- Enable the trigram extension
CREATE EXTENSION IF NOT EXISTS pg_trgm;

-- Create a trigram index
CREATE INDEX idx_products_name_trgm ON products USING GIN (name gin_trgm_ops);

-- Now you can do similarity searches
-- "runnig" will match "running" because they share many trigrams
SELECT name, similarity(name, 'runnig') AS sim
FROM products
WHERE name % 'runnig'  -- % operator uses a default threshold of 0.3
ORDER BY sim DESC
LIMIT 10;

-- Or combine with full-text search for the best of both:
SELECT 
  name,
  ts_rank(search_vector, query) AS fts_rank,
  similarity(name, 'runnig shoes') AS trgm_sim
FROM products, plainto_tsquery('english', 'runnig shoes') AS query
WHERE 
  search_vector @@ query  -- Full-text match
  OR name % 'runnig shoes' -- OR trigram similarity match
ORDER BY 
  (ts_rank(search_vector, query) * 2 + similarity(name, 'runnig shoes')) DESC
LIMIT 20;
```

```typescript
// Using pg_trgm with Drizzle for fuzzy search

export async function fuzzySearch(query: string, limit = 20) {
  const results = await db.execute(sql`
    SELECT 
      id,
      name,
      description,
      category,
      price,
      similarity(name, ${query}) as name_similarity,
      CASE 
        WHEN search_vector @@ plainto_tsquery('english', ${query}) THEN
          ts_rank(search_vector, plainto_tsquery('english', ${query}))
        ELSE 0
      END as fts_rank
    FROM products
    WHERE 
      search_vector @@ plainto_tsquery('english', ${query})
      OR similarity(name, ${query}) > 0.2
      OR name ILIKE ${'%' + query + '%'}
    ORDER BY 
      (COALESCE(fts_rank, 0) * 3 + COALESCE(name_similarity, 0) * 2) DESC
    LIMIT ${limit}
  `);

  return results.rows;
}
```

---

## 7. FACETED SEARCH

### What Is Faceted Search?

Faceted search is the set of filters you see on the left side of Amazon, the category pills on Airbnb, or the price slider on any e-commerce site. Facets let users narrow down results by attributes: category, price range, rating, color, size, location.

The key insight about faceted search is that **facet counts must update dynamically.** When a user filters to "Running Shoes," the brand facet should show only brands that have running shoes, with accurate counts. This is surprisingly hard to do with raw database queries but trivial with Algolia or Typesense.

### Faceted Search with Algolia

Algolia handles facets natively. You define which attributes are facetable in your index settings, and they're returned with every search.

```typescript
// src/features/search/faceted-search.tsx

import React, { useState } from 'react';
import {
  View,
  Text,
  Pressable,
  ScrollView,
  StyleSheet,
} from 'react-native';
import {
  InstantSearch,
  useRefinementList,
  useRange,
  useClearRefinements,
  useCurrentRefinements,
} from 'react-instantsearch-core';
import { searchClient, ALGOLIA_INDEXES } from '../../lib/algolia';

/**
 * Category facet — select one or more categories.
 */
function CategoryFacet() {
  const { items, refine } = useRefinementList({
    attribute: 'category',
    sortBy: ['count:desc', 'name:asc'],
    limit: 10,
  });

  return (
    <View style={styles.facetSection}>
      <Text style={styles.facetTitle}>Category</Text>
      {items.map((item) => (
        <Pressable
          key={item.value}
          style={styles.facetItem}
          onPress={() => refine(item.value)}
          accessibilityRole="checkbox"
          accessibilityState={{ checked: item.isRefined }}
        >
          <View
            style={[
              styles.checkbox,
              item.isRefined && styles.checkboxChecked,
            ]}
          >
            {item.isRefined && <Text style={styles.checkmark}>check</Text>}
          </View>
          <Text
            style={[
              styles.facetLabel,
              item.isRefined && styles.facetLabelActive,
            ]}
          >
            {item.label}
          </Text>
          <Text style={styles.facetCount}>({item.count})</Text>
        </Pressable>
      ))}
    </View>
  );
}

/**
 * Brand facet — same pattern, different attribute.
 */
function BrandFacet() {
  const { items, refine } = useRefinementList({
    attribute: 'brand',
    sortBy: ['count:desc'],
    limit: 8,
  });

  return (
    <View style={styles.facetSection}>
      <Text style={styles.facetTitle}>Brand</Text>
      {items.map((item) => (
        <Pressable
          key={item.value}
          style={styles.facetItem}
          onPress={() => refine(item.value)}
          accessibilityRole="checkbox"
          accessibilityState={{ checked: item.isRefined }}
        >
          <View
            style={[styles.checkbox, item.isRefined && styles.checkboxChecked]}
          >
            {item.isRefined && <Text style={styles.checkmark}>check</Text>}
          </View>
          <Text
            style={[styles.facetLabel, item.isRefined && styles.facetLabelActive]}
          >
            {item.label}
          </Text>
          <Text style={styles.facetCount}>({item.count})</Text>
        </Pressable>
      ))}
    </View>
  );
}

/**
 * Price range slider facet.
 */
function PriceRangeFacet() {
  const { range, start, refine } = useRange({
    attribute: 'price',
  });

  const [localMin, setLocalMin] = useState(start[0] ?? range.min ?? 0);
  const [localMax, setLocalMax] = useState(start[1] ?? range.max ?? 1000);

  const handleApply = () => {
    refine([localMin, localMax]);
  };

  return (
    <View style={styles.facetSection}>
      <Text style={styles.facetTitle}>Price Range</Text>
      <View style={styles.priceInputs}>
        <View style={styles.priceField}>
          <Text style={styles.priceLabel}>Min</Text>
          <Text style={styles.priceValue}>${localMin}</Text>
        </View>
        <Text style={styles.priceSeparator}>-</Text>
        <View style={styles.priceField}>
          <Text style={styles.priceLabel}>Max</Text>
          <Text style={styles.priceValue}>${localMax}</Text>
        </View>
      </View>
      {/* In a real app, use a slider component here */}
      <Pressable onPress={handleApply} style={styles.applyButton}>
        <Text style={styles.applyButtonText}>Apply</Text>
      </Pressable>
    </View>
  );
}

/**
 * Active filters display + clear button.
 */
function ActiveFilters() {
  const { items } = useCurrentRefinements();
  const { refine: clearAll } = useClearRefinements();

  if (items.length === 0) return null;

  return (
    <View style={styles.activeFilters}>
      <ScrollView horizontal showsHorizontalScrollIndicator={false}>
        {items.map((item) =>
          item.refinements.map((refinement) => (
            <Pressable
              key={`${item.attribute}-${refinement.label}`}
              style={styles.filterChip}
              onPress={() => item.refine(refinement)}
              accessibilityLabel={`Remove filter: ${refinement.label}`}
            >
              <Text style={styles.filterChipText}>
                {refinement.label} x
              </Text>
            </Pressable>
          ))
        )}
        <Pressable
          style={styles.clearAllChip}
          onPress={clearAll}
          accessibilityLabel="Clear all filters"
        >
          <Text style={styles.clearAllText}>Clear All</Text>
        </Pressable>
      </ScrollView>
    </View>
  );
}

/**
 * Complete faceted search screen.
 */
export function FacetedSearchScreen() {
  const [showFilters, setShowFilters] = useState(false);

  return (
    <InstantSearch
      searchClient={searchClient}
      indexName={ALGOLIA_INDEXES.products}
    >
      <View style={styles.container}>
        {/* Active Filters */}
        <ActiveFilters />

        {/* Filter Toggle */}
        <Pressable
          style={styles.filterToggle}
          onPress={() => setShowFilters(!showFilters)}
        >
          <Text style={styles.filterToggleText}>
            {showFilters ? 'Hide Filters' : 'Show Filters'}
          </Text>
        </Pressable>

        <View style={styles.content}>
          {/* Filter Panel */}
          {showFilters && (
            <ScrollView style={styles.filterPanel}>
              <CategoryFacet />
              <BrandFacet />
              <PriceRangeFacet />
            </ScrollView>
          )}

          {/* Results */}
          <View style={styles.resultsPanel}>
            {/* Reuse SearchResults component from Section 3 */}
          </View>
        </View>
      </View>
    </InstantSearch>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1, backgroundColor: '#FFF' },
  content: { flex: 1, flexDirection: 'row' },
  filterPanel: { width: 280, borderRightWidth: 1, borderRightColor: '#E5E7EB', padding: 16 },
  resultsPanel: { flex: 1 },
  facetSection: { marginBottom: 24 },
  facetTitle: { fontSize: 15, fontWeight: '700', color: '#1F2937', marginBottom: 10 },
  facetItem: { flexDirection: 'row', alignItems: 'center', paddingVertical: 6 },
  checkbox: { width: 20, height: 20, borderRadius: 4, borderWidth: 1.5, borderColor: '#D1D5DB', marginRight: 10, justifyContent: 'center', alignItems: 'center' },
  checkboxChecked: { backgroundColor: '#6366F1', borderColor: '#6366F1' },
  checkmark: { color: '#FFF', fontSize: 12, fontWeight: '700' },
  facetLabel: { flex: 1, fontSize: 14, color: '#374151' },
  facetLabelActive: { fontWeight: '600', color: '#6366F1' },
  facetCount: { fontSize: 12, color: '#9CA3AF' },
  priceInputs: { flexDirection: 'row', alignItems: 'center', marginBottom: 12 },
  priceField: { flex: 1, backgroundColor: '#F3F4F6', borderRadius: 8, padding: 10 },
  priceLabel: { fontSize: 11, color: '#6B7280', marginBottom: 2 },
  priceValue: { fontSize: 16, fontWeight: '600', color: '#1F2937' },
  priceSeparator: { marginHorizontal: 8, fontSize: 16, color: '#9CA3AF' },
  applyButton: { backgroundColor: '#6366F1', borderRadius: 8, padding: 10, alignItems: 'center' },
  applyButtonText: { color: '#FFF', fontWeight: '600', fontSize: 14 },
  activeFilters: { paddingHorizontal: 16, paddingVertical: 8, borderBottomWidth: 1, borderBottomColor: '#E5E7EB' },
  filterChip: { backgroundColor: '#EEF2FF', borderRadius: 16, paddingHorizontal: 12, paddingVertical: 6, marginRight: 8 },
  filterChipText: { fontSize: 13, color: '#6366F1', fontWeight: '500' },
  clearAllChip: { paddingHorizontal: 12, paddingVertical: 6 },
  clearAllText: { fontSize: 13, color: '#EF4444', fontWeight: '500' },
  filterToggle: { padding: 12, borderBottomWidth: 1, borderBottomColor: '#E5E7EB' },
  filterToggleText: { fontSize: 14, color: '#6366F1', fontWeight: '600', textAlign: 'center' },
});
```

### Faceted Search with Database Queries

If you're not using a dedicated search engine, you can implement facets with database queries. It's more work, but it's possible:

```typescript
// src/lib/db-faceted-search.ts

import { sql, type SQL } from 'drizzle-orm';
import { db } from './database';

interface FacetedSearchParams {
  query: string;
  filters: {
    categories?: string[];
    brands?: string[];
    minPrice?: number;
    maxPrice?: number;
    inStock?: boolean;
  };
  limit?: number;
  offset?: number;
}

interface FacetedSearchResult {
  results: any[];
  total: number;
  facets: {
    categories: Array<{ value: string; count: number }>;
    brands: Array<{ value: string; count: number }>;
    priceRanges: Array<{ range: string; count: number }>;
  };
}

export async function facetedSearch(params: FacetedSearchParams): Promise<FacetedSearchResult> {
  const { query, filters, limit = 20, offset = 0 } = params;

  // Build WHERE clause
  const conditions: SQL[] = [];

  if (query) {
    conditions.push(
      sql`search_vector @@ plainto_tsquery('english', ${query})`
    );
  }

  if (filters.categories?.length) {
    conditions.push(
      sql`category = ANY(${filters.categories})`
    );
  }

  if (filters.brands?.length) {
    conditions.push(
      sql`brand = ANY(${filters.brands})`
    );
  }

  if (filters.minPrice !== undefined) {
    conditions.push(sql`price >= ${filters.minPrice}`);
  }

  if (filters.maxPrice !== undefined) {
    conditions.push(sql`price <= ${filters.maxPrice}`);
  }

  if (filters.inStock !== undefined) {
    conditions.push(sql`in_stock = ${filters.inStock}`);
  }

  const whereClause = conditions.length > 0
    ? sql`WHERE ${sql.join(conditions, sql` AND `)}`
    : sql``;

  // Run results query and facet queries in parallel
  const [resultsQuery, categoryFacets, brandFacets, priceRanges] = await Promise.all([
    // Main results
    db.execute(sql`
      SELECT *, COUNT(*) OVER() as total_count
      FROM products
      ${whereClause}
      ORDER BY ${query
        ? sql`ts_rank(search_vector, plainto_tsquery('english', ${query})) DESC`
        : sql`created_at DESC`
      }
      LIMIT ${limit} OFFSET ${offset}
    `),

    // Category facets (counts per category, respecting other filters)
    db.execute(sql`
      SELECT category as value, COUNT(*) as count
      FROM products
      ${whereClause}
      GROUP BY category
      ORDER BY count DESC
      LIMIT 20
    `),

    // Brand facets
    db.execute(sql`
      SELECT brand as value, COUNT(*) as count
      FROM products
      ${whereClause}
      GROUP BY brand
      ORDER BY count DESC
      LIMIT 20
    `),

    // Price range facets
    db.execute(sql`
      SELECT
        CASE
          WHEN price < 25 THEN 'Under $25'
          WHEN price < 50 THEN '$25 - $50'
          WHEN price < 100 THEN '$50 - $100'
          WHEN price < 200 THEN '$100 - $200'
          ELSE '$200+'
        END as range,
        COUNT(*) as count
      FROM products
      ${whereClause}
      GROUP BY range
      ORDER BY MIN(price)
    `),
  ]);

  return {
    results: resultsQuery.rows,
    total: resultsQuery.rows[0]?.total_count ?? 0,
    facets: {
      categories: categoryFacets.rows as Array<{ value: string; count: number }>,
      brands: brandFacets.rows as Array<{ value: string; count: number }>,
      priceRanges: priceRanges.rows as Array<{ range: string; count: number }>,
    },
  };
}
```

---

## 8. BUILDING A COMPLETE SEARCH SCREEN

### Putting It All Together

Let's build a complete, production-ready search screen for React Native. This combines everything we've discussed: debounced input, recent searches, suggestions, results with highlighting, filters, and pagination. I'll use a provider-agnostic approach so you can plug in Algolia, Typesense, Meilisearch, or even Postgres.

```typescript
// src/features/search/search-provider.tsx

import React, { createContext, useContext, type ReactNode } from 'react';

/**
 * Provider-agnostic search interface.
 * Implement this for Algolia, Typesense, Meilisearch, or Postgres.
 */
export interface SearchProvider<T = any> {
  search(params: {
    query: string;
    page?: number;
    perPage?: number;
    filters?: Record<string, string[]>;
    sortBy?: string;
  }): Promise<{
    hits: T[];
    totalHits: number;
    facets: Record<string, Array<{ value: string; count: number }>>;
    page: number;
    totalPages: number;
  }>;

  getSuggestions(query: string): Promise<string[]>;
}

const SearchProviderContext = createContext<SearchProvider | null>(null);

export function SearchProviderWrapper({
  provider,
  children,
}: {
  provider: SearchProvider;
  children: ReactNode;
}) {
  return (
    <SearchProviderContext.Provider value={provider}>
      {children}
    </SearchProviderContext.Provider>
  );
}

export function useSearchProvider(): SearchProvider {
  const provider = useContext(SearchProviderContext);
  if (!provider) {
    throw new Error('useSearchProvider must be used within SearchProviderWrapper');
  }
  return provider;
}
```

```typescript
// src/features/search/search-state.ts

import { useState, useCallback, useRef, useEffect } from 'react';
import { useSearchProvider } from './search-provider';
import {
  getRecentSearches,
  addRecentSearch,
  clearRecentSearches as clearStoredRecentSearches,
  type RecentSearch,
} from '../../search/recent-searches';

export interface SearchState {
  // Input
  query: string;
  setQuery: (q: string) => void;

  // Results
  results: any[];
  totalHits: number;
  isLoading: boolean;
  error: Error | null;

  // Pagination
  page: number;
  totalPages: number;
  loadNextPage: () => void;
  isLoadingMore: boolean;

  // Facets
  facets: Record<string, Array<{ value: string; count: number }>>;
  activeFilters: Record<string, string[]>;
  toggleFilter: (attribute: string, value: string) => void;
  clearFilters: () => void;

  // Suggestions and History
  suggestions: string[];
  recentSearches: RecentSearch[];
  clearRecentSearches: () => void;

  // UI State
  hasSearched: boolean;
  isFocused: boolean;
  setIsFocused: (f: boolean) => void;
}

export function useSearchState(): SearchState {
  const provider = useSearchProvider();

  const [query, setQuery] = useState('');
  const [results, setResults] = useState<any[]>([]);
  const [totalHits, setTotalHits] = useState(0);
  const [isLoading, setIsLoading] = useState(false);
  const [isLoadingMore, setIsLoadingMore] = useState(false);
  const [error, setError] = useState<Error | null>(null);
  const [page, setPage] = useState(1);
  const [totalPages, setTotalPages] = useState(0);
  const [facets, setFacets] = useState<Record<string, Array<{ value: string; count: number }>>>({});
  const [activeFilters, setActiveFilters] = useState<Record<string, string[]>>({});
  const [suggestions, setSuggestions] = useState<string[]>([]);
  const [recentSearches, setRecentSearches] = useState<RecentSearch[]>(getRecentSearches());
  const [hasSearched, setHasSearched] = useState(false);
  const [isFocused, setIsFocused] = useState(false);

  const latestRequestRef = useRef(0);

  // Debounced search
  useEffect(() => {
    if (query.length < 2) {
      setResults([]);
      setHasSearched(false);
      return;
    }

    setIsLoading(true);
    const requestId = ++latestRequestRef.current;

    const timer = setTimeout(async () => {
      try {
        const response = await provider.search({
          query,
          page: 1,
          filters: activeFilters,
        });

        if (requestId === latestRequestRef.current) {
          setResults(response.hits);
          setTotalHits(response.totalHits);
          setFacets(response.facets);
          setPage(response.page);
          setTotalPages(response.totalPages);
          setHasSearched(true);
          setIsLoading(false);

          addRecentSearch(query, response.totalHits);
          setRecentSearches(getRecentSearches());
        }
      } catch (err) {
        if (requestId === latestRequestRef.current) {
          setError(err instanceof Error ? err : new Error('Search failed'));
          setIsLoading(false);
        }
      }
    }, 300);

    return () => clearTimeout(timer);
  }, [query, activeFilters, provider]);

  // Get suggestions as user types
  useEffect(() => {
    if (query.length < 2) {
      setSuggestions([]);
      return;
    }

    const timer = setTimeout(async () => {
      try {
        const results = await provider.getSuggestions(query);
        setSuggestions(results);
      } catch {
        // Suggestions are non-critical; fail silently
      }
    }, 150); // Shorter debounce for suggestions

    return () => clearTimeout(timer);
  }, [query, provider]);

  const loadNextPage = useCallback(async () => {
    if (isLoadingMore || page >= totalPages) return;

    setIsLoadingMore(true);
    try {
      const response = await provider.search({
        query,
        page: page + 1,
        filters: activeFilters,
      });
      setResults((prev) => [...prev, ...response.hits]);
      setPage(response.page);
      setTotalPages(response.totalPages);
    } catch {
      // Pagination errors are non-fatal
    } finally {
      setIsLoadingMore(false);
    }
  }, [query, page, totalPages, activeFilters, isLoadingMore, provider]);

  const toggleFilter = useCallback((attribute: string, value: string) => {
    setActiveFilters((prev) => {
      const current = prev[attribute] ?? [];
      const isActive = current.includes(value);

      return {
        ...prev,
        [attribute]: isActive
          ? current.filter((v) => v !== value)
          : [...current, value],
      };
    });
    setPage(1); // Reset to first page when filters change
  }, []);

  const clearFilters = useCallback(() => {
    setActiveFilters({});
    setPage(1);
  }, []);

  const clearRecentSearchesFn = useCallback(() => {
    clearStoredRecentSearches();
    setRecentSearches([]);
  }, []);

  return {
    query,
    setQuery,
    results,
    totalHits,
    isLoading,
    error,
    page,
    totalPages,
    loadNextPage,
    isLoadingMore,
    facets,
    activeFilters,
    toggleFilter,
    clearFilters,
    suggestions,
    recentSearches,
    clearRecentSearches: clearRecentSearchesFn,
    hasSearched,
    isFocused,
    setIsFocused,
  };
}
```

```typescript
// src/features/search/complete-search-screen.tsx

import React, { useRef } from 'react';
import {
  View,
  TextInput,
  FlatList,
  Text,
  Pressable,
  ActivityIndicator,
  Keyboard,
  StyleSheet,
} from 'react-native';
import { useSearchState, type SearchState } from './search-state';
import { HighlightedText } from '../../search/highlight';

export function CompleteSearchScreen() {
  const search = useSearchState();
  const inputRef = useRef<TextInput>(null);

  const showSuggestions = search.isFocused && search.query.length >= 2 && !search.hasSearched;
  const showRecentSearches = search.isFocused && search.query.length === 0;
  const showResults = search.hasSearched;

  return (
    <View style={styles.container}>
      {/* Header with search input */}
      <View style={styles.header}>
        <View style={styles.inputWrapper}>
          <TextInput
            ref={inputRef}
            style={styles.input}
            value={search.query}
            onChangeText={search.setQuery}
            onFocus={() => search.setIsFocused(true)}
            onBlur={() => search.setIsFocused(false)}
            placeholder="Search products..."
            placeholderTextColor="#9CA3AF"
            returnKeyType="search"
            autoCapitalize="none"
            autoCorrect={false}
            accessibilityLabel="Search products"
            accessibilityRole="search"
          />
          {search.query.length > 0 && (
            <Pressable
              onPress={() => {
                search.setQuery('');
                inputRef.current?.focus();
              }}
              style={styles.clearInput}
              accessibilityLabel="Clear search"
            >
              <Text style={styles.clearInputText}>x</Text>
            </Pressable>
          )}
        </View>
      </View>

      {/* Loading indicator */}
      {search.isLoading && !search.isLoadingMore && (
        <View style={styles.loadingBar}>
          <ActivityIndicator size="small" color="#6366F1" />
          <Text style={styles.loadingText}>Searching...</Text>
        </View>
      )}

      {/* Active filters */}
      <ActiveFilterChips search={search} />

      {/* Content area */}
      {showRecentSearches && (
        <RecentSearches
          searches={search.recentSearches}
          onSelect={(q) => {
            search.setQuery(q);
            Keyboard.dismiss();
          }}
          onClear={search.clearRecentSearches}
        />
      )}

      {showSuggestions && (
        <Suggestions
          suggestions={search.suggestions}
          onSelect={(q) => {
            search.setQuery(q);
            Keyboard.dismiss();
          }}
        />
      )}

      {showResults && search.results.length > 0 && (
        <ResultsList search={search} />
      )}

      {showResults && search.results.length === 0 && !search.isLoading && (
        <NoResults query={search.query} />
      )}

      {!search.isFocused && !search.hasSearched && (
        <EmptyState />
      )}
    </View>
  );
}

function ActiveFilterChips({ search }: { search: SearchState }) {
  const hasFilters = Object.values(search.activeFilters).some((v) => v.length > 0);
  if (!hasFilters) return null;

  return (
    <View style={styles.chipContainer}>
      {Object.entries(search.activeFilters).map(([attr, values]) =>
        values.map((value) => (
          <Pressable
            key={`${attr}-${value}`}
            style={styles.chip}
            onPress={() => search.toggleFilter(attr, value)}
            accessibilityLabel={`Remove filter: ${value}`}
          >
            <Text style={styles.chipText}>{value} x</Text>
          </Pressable>
        ))
      )}
      <Pressable onPress={search.clearFilters}>
        <Text style={styles.clearFiltersText}>Clear all</Text>
      </Pressable>
    </View>
  );
}

function ResultsList({ search }: { search: SearchState }) {
  return (
    <FlatList
      data={search.results}
      keyExtractor={(item) => item.id || item.objectID}
      keyboardShouldPersistTaps="handled"
      keyboardDismissMode="on-drag"
      onEndReached={search.loadNextPage}
      onEndReachedThreshold={0.5}
      ListHeaderComponent={
        <View style={styles.resultsHeader}>
          <Text style={styles.resultsCount}>
            {search.totalHits.toLocaleString()} results
          </Text>
        </View>
      }
      renderItem={({ item }) => (
        <ResultCard item={item} query={search.query} />
      )}
      ListFooterComponent={
        search.isLoadingMore ? (
          <ActivityIndicator style={styles.loadingMore} color="#6366F1" />
        ) : null
      }
    />
  );
}

function ResultCard({ item, query }: { item: any; query: string }) {
  return (
    <Pressable style={styles.resultCard} accessibilityRole="button">
      <View style={styles.resultImageContainer}>
        <View style={styles.resultImagePlaceholder} />
      </View>
      <View style={styles.resultContent}>
        <HighlightedText
          text={item.name}
          highlight={query}
          style={styles.resultName}
          highlightStyle={styles.highlightMark}
        />
        <Text style={styles.resultCategory}>{item.category}</Text>
        <Text style={styles.resultDescription} numberOfLines={2}>
          {item.description}
        </Text>
        <View style={styles.resultFooter}>
          <Text style={styles.resultPrice}>
            ${(item.price ?? 0).toFixed(2)}
          </Text>
          {item.rating && (
            <Text style={styles.resultRating}>
              {item.rating.toFixed(1)}
            </Text>
          )}
        </View>
      </View>
    </Pressable>
  );
}

function RecentSearches({
  searches,
  onSelect,
  onClear,
}: {
  searches: Array<{ query: string; timestamp: number; resultCount: number }>;
  onSelect: (query: string) => void;
  onClear: () => void;
}) {
  if (searches.length === 0) return null;

  return (
    <View style={styles.sectionContainer}>
      <View style={styles.sectionHeader}>
        <Text style={styles.sectionTitle}>Recent Searches</Text>
        <Pressable onPress={onClear}>
          <Text style={styles.sectionAction}>Clear All</Text>
        </Pressable>
      </View>
      {searches.slice(0, 8).map((s) => (
        <Pressable
          key={s.query}
          style={styles.suggestionItem}
          onPress={() => onSelect(s.query)}
          accessibilityRole="button"
          accessibilityLabel={`Search for ${s.query}`}
        >
          <Text style={styles.suggestionText}>{s.query}</Text>
        </Pressable>
      ))}
    </View>
  );
}

function Suggestions({
  suggestions,
  onSelect,
}: {
  suggestions: string[];
  onSelect: (query: string) => void;
}) {
  if (suggestions.length === 0) return null;

  return (
    <View style={styles.sectionContainer}>
      <Text style={styles.sectionTitle}>Suggestions</Text>
      {suggestions.map((s) => (
        <Pressable
          key={s}
          style={styles.suggestionItem}
          onPress={() => onSelect(s)}
          accessibilityRole="button"
          accessibilityLabel={`Search for ${s}`}
        >
          <Text style={styles.suggestionText}>{s}</Text>
        </Pressable>
      ))}
    </View>
  );
}

function NoResults({ query }: { query: string }) {
  return (
    <View style={styles.centerState}>
      <Text style={styles.centerTitle}>No results for "{query}"</Text>
      <Text style={styles.centerSubtitle}>
        Try a different search term or check your spelling.
      </Text>
    </View>
  );
}

function EmptyState() {
  return (
    <View style={styles.centerState}>
      <Text style={styles.centerTitle}>Search for products</Text>
      <Text style={styles.centerSubtitle}>
        Find what you need by name, category, or brand.
      </Text>
    </View>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1, backgroundColor: '#FFFFFF' },
  header: { paddingHorizontal: 16, paddingVertical: 10, borderBottomWidth: 1, borderBottomColor: '#E5E7EB' },
  inputWrapper: { flexDirection: 'row', alignItems: 'center', backgroundColor: '#F3F4F6', borderRadius: 12, paddingHorizontal: 12 },
  input: { flex: 1, height: 44, fontSize: 16, color: '#1F2937' },
  clearInput: { padding: 4 },
  clearInputText: { fontSize: 16, color: '#9CA3AF' },
  loadingBar: { flexDirection: 'row', alignItems: 'center', justifyContent: 'center', padding: 8, backgroundColor: '#F0F0FF' },
  loadingText: { marginLeft: 8, fontSize: 13, color: '#6366F1' },
  chipContainer: { flexDirection: 'row', flexWrap: 'wrap', padding: 12, gap: 8, borderBottomWidth: 1, borderBottomColor: '#E5E7EB' },
  chip: { backgroundColor: '#EEF2FF', borderRadius: 16, paddingHorizontal: 12, paddingVertical: 6 },
  chipText: { fontSize: 13, color: '#6366F1', fontWeight: '500' },
  clearFiltersText: { fontSize: 13, color: '#EF4444', fontWeight: '500', alignSelf: 'center' },
  resultsHeader: { flexDirection: 'row', justifyContent: 'space-between', padding: 16, borderBottomWidth: 1, borderBottomColor: '#F3F4F6' },
  resultsCount: { fontSize: 14, color: '#6B7280', fontWeight: '500' },
  resultCard: { flexDirection: 'row', padding: 16, borderBottomWidth: 1, borderBottomColor: '#F3F4F6' },
  resultImageContainer: { marginRight: 12 },
  resultImagePlaceholder: { width: 80, height: 80, backgroundColor: '#F3F4F6', borderRadius: 8 },
  resultContent: { flex: 1 },
  resultName: { fontSize: 16, fontWeight: '600', color: '#1F2937', marginBottom: 2 },
  highlightMark: { backgroundColor: '#FEF08A', fontWeight: '700' },
  resultCategory: { fontSize: 12, color: '#6B7280', marginBottom: 4 },
  resultDescription: { fontSize: 14, color: '#4B5563', lineHeight: 20, marginBottom: 8 },
  resultFooter: { flexDirection: 'row', justifyContent: 'space-between', alignItems: 'center' },
  resultPrice: { fontSize: 16, fontWeight: '700', color: '#1F2937' },
  resultRating: { fontSize: 13, color: '#F59E0B' },
  loadingMore: { padding: 16 },
  sectionContainer: { padding: 16 },
  sectionHeader: { flexDirection: 'row', justifyContent: 'space-between', alignItems: 'center', marginBottom: 12 },
  sectionTitle: { fontSize: 15, fontWeight: '700', color: '#374151' },
  sectionAction: { fontSize: 14, color: '#6366F1' },
  suggestionItem: { flexDirection: 'row', alignItems: 'center', paddingVertical: 10 },
  suggestionText: { fontSize: 15, color: '#1F2937' },
  centerState: { flex: 1, justifyContent: 'center', alignItems: 'center', paddingHorizontal: 32 },
  centerTitle: { fontSize: 18, fontWeight: '600', color: '#1F2937', textAlign: 'center', marginBottom: 8 },
  centerSubtitle: { fontSize: 14, color: '#6B7280', textAlign: 'center', lineHeight: 20 },
});
```

---

## 9. SEARCH INDEXING

### Keeping Your Search Index in Sync

The hardest problem in search isn't relevance or speed — it's keeping your search index in sync with your database. When a product's price changes, the search index needs to reflect that change. When a product is deleted, it needs to disappear from search results. When a new product is created, it needs to be searchable.

There are three approaches, and each has trade-offs:

```
┌──────────────────────────────────────────────────────────────────────┐
│                   INDEX SYNC STRATEGIES                               │
│                                                                      │
│  Strategy 1: Webhook on Change                                       │
│  - How: Your API sends an update to the index on each               │
│    create/update/delete                                              │
│  - Latency: < 1 second                                              │
│  - Complexity: Low                                                   │
│  - Reliability: Medium (updates can fail, need retry logic)         │
│  - Best for: Simple apps with low write volume                      │
│                                                                      │
│  Strategy 2: Queue-Based Sync                                        │
│  - How: Changes are pushed to a queue (SQS, Redis, BullMQ),        │
│    a worker processes them and updates the index                    │
│  - Latency: 1-10 seconds                                            │
│  - Complexity: Medium                                                │
│  - Reliability: High (queues handle retries, dead letters)          │
│  - Best for: Most production apps                                    │
│                                                                      │
│  Strategy 3: Periodic Full Reindex                                   │
│  - How: A cron job does a full reindex every N minutes/hours        │
│  - Latency: Minutes to hours                                         │
│  - Complexity: Low                                                   │
│  - Reliability: High (idempotent, self-healing)                     │
│  - Best for: Data that changes infrequently, or as a backup        │
│                                                                      │
│  RECOMMENDED: Strategy 2 (queue) + Strategy 3 (periodic) as backup  │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

### Webhook-Based Indexing

```typescript
// src/api/search-index.ts
// Trigger this from your CRUD operations

import { searchClient } from '../../lib/algolia';

type IndexAction = 'create' | 'update' | 'delete';

interface IndexPayload {
  action: IndexAction;
  collection: string;
  documentId: string;
  document?: Record<string, unknown>;
}

/**
 * Update the search index when data changes.
 * Call this from your API routes after successful mutations.
 */
export async function updateSearchIndex(payload: IndexPayload) {
  const { action, collection, documentId, document } = payload;

  const index = searchClient.initIndex(collection);

  try {
    switch (action) {
      case 'create':
      case 'update':
        if (!document) throw new Error('Document required for create/update');
        await index.saveObject({
          objectID: documentId,
          ...document,
        });
        break;

      case 'delete':
        await index.deleteObject(documentId);
        break;
    }

    console.log(`[SearchIndex] ${action} ${collection}/${documentId} successful`);
  } catch (error) {
    console.error(`[SearchIndex] ${action} ${collection}/${documentId} failed:`, error);
    // Don't throw — search index sync failures shouldn't break the main operation
    // Instead, queue for retry
    await queueForRetry(payload);
  }
}

// Usage in an API route:
//
// export async function POST(request: Request) {
//   const product = await db.insert(products).values(data).returning();
//
//   // Fire-and-forget: don't await, don't block the response
//   updateSearchIndex({
//     action: 'create',
//     collection: 'products',
//     documentId: product.id,
//     document: transformProductForSearch(product),
//   });
//
//   return Response.json(product);
// }
```

### Queue-Based Indexing

For production apps with significant write volume, a queue-based approach is more reliable:

```typescript
// src/jobs/search-index-worker.ts

import { Queue, Worker } from 'bullmq';
import { Redis } from 'ioredis';

const connection = new Redis(process.env.REDIS_URL!);

// Create the queue
export const searchIndexQueue = new Queue('search-index', {
  connection,
  defaultJobOptions: {
    attempts: 3,
    backoff: {
      type: 'exponential',
      delay: 1000,
    },
    removeOnComplete: { count: 1000 },
    removeOnFail: { count: 5000 },
  },
});

// Create the worker
const worker = new Worker(
  'search-index',
  async (job) => {
    const { action, collection, documentId, document } = job.data;

    switch (action) {
      case 'create':
      case 'update':
        await searchProvider.upsertDocument(collection, documentId, document);
        break;
      case 'delete':
        await searchProvider.deleteDocument(collection, documentId);
        break;
    }
  },
  {
    connection,
    concurrency: 10, // Process 10 jobs at a time
  }
);

worker.on('completed', (job) => {
  console.log(`[SearchIndex] Job ${job.id} completed`);
});

worker.on('failed', (job, error) => {
  console.error(`[SearchIndex] Job ${job?.id} failed:`, error.message);
});

// Enqueue function — call this from your API routes
export async function enqueueIndexUpdate(payload: {
  action: 'create' | 'update' | 'delete';
  collection: string;
  documentId: string;
  document?: Record<string, unknown>;
}) {
  await searchIndexQueue.add('index-update', payload, {
    // Deduplicate: if the same document is updated multiple times quickly,
    // only process the latest update
    jobId: `${payload.collection}-${payload.documentId}-${payload.action}`,
  });
}
```

### Periodic Full Reindex

As a safety net, run a full reindex periodically to catch any documents that might have slipped through:

```typescript
// scripts/full-reindex.ts

import { db } from '../src/lib/database';
import { sql } from 'drizzle-orm';

const BATCH_SIZE = 500;

async function fullReindex() {
  console.log('Starting full reindex...');
  const startTime = Date.now();

  let offset = 0;
  let totalIndexed = 0;

  while (true) {
    // Fetch a batch of products
    const batch = await db.query.products.findMany({
      limit: BATCH_SIZE,
      offset,
      with: { category: true, brand: true },
      orderBy: (products, { asc }) => [asc(products.id)],
    });

    if (batch.length === 0) break;

    // Transform to search documents
    const documents = batch.map(transformProductForSearch);

    // Batch upsert to search provider
    await searchProvider.batchUpsert('products', documents);

    totalIndexed += batch.length;
    offset += BATCH_SIZE;

    console.log(`Indexed ${totalIndexed} products...`);
  }

  // Clean up: remove documents from the index that no longer exist in DB
  const allDbIds = await db.execute(sql`SELECT id FROM products`);
  const dbIdSet = new Set(allDbIds.rows.map((r: any) => r.id));

  // Get all IDs from the search index
  const indexedIds = await searchProvider.getAllDocumentIds('products');
  const orphanedIds = indexedIds.filter((id: string) => !dbIdSet.has(id));

  if (orphanedIds.length > 0) {
    console.log(`Removing ${orphanedIds.length} orphaned documents...`);
    await searchProvider.batchDelete('products', orphanedIds);
  }

  const duration = ((Date.now() - startTime) / 1000).toFixed(1);
  console.log(
    `Full reindex complete! ${totalIndexed} documents indexed, ` +
    `${orphanedIds.length} orphans removed. Took ${duration}s.`
  );
}

function transformProductForSearch(product: any) {
  return {
    id: product.id,
    objectID: product.id, // Algolia compatibility
    name: product.name,
    description: product.description ?? '',
    category: product.category?.name ?? '',
    brand: product.brand?.name ?? '',
    tags: product.tags ?? [],
    price: product.priceInCents / 100,
    rating: product.averageRating ?? 0,
    inStock: (product.inventory ?? 0) > 0,
    imageUrl: product.imageUrl,
    slug: product.slug,
    createdAt: product.createdAt,
  };
}

// Run with: npx tsx scripts/full-reindex.ts
// Schedule with cron: every 6 hours as a safety net
fullReindex().catch(console.error);
```

---

## 10. SEARCH ANALYTICS

### Your Users Are Telling You What's Missing

Search analytics is the most underused product intelligence tool I've ever seen. Your users are literally typing what they want into a box. If you're not analyzing what they type, you're ignoring the most direct feedback channel you have.

### What to Track

```typescript
// src/analytics/search-analytics.ts

interface SearchEvent {
  // What the user searched for
  query: string;
  
  // What they saw
  resultCount: number;
  
  // What they did
  action: 'search' | 'click' | 'convert' | 'refine' | 'no_result';
  
  // Context
  timestamp: number;
  userId?: string;
  sessionId: string;
  platform: 'mobile' | 'web';
  
  // Result interaction (if action is 'click')
  clickedResultId?: string;
  clickedResultPosition?: number;
  
  // Filters applied
  activeFilters?: Record<string, string[]>;
  
  // Performance
  responseTimeMs?: number;
}

/**
 * Track a search event.
 * Send to your analytics provider (Amplitude, Mixpanel, PostHog, etc.)
 * AND to your search provider's analytics (Algolia Insights, etc.)
 */
export function trackSearchEvent(event: SearchEvent): void {
  // Send to your general analytics
  analytics.track('search_event', {
    query: event.query,
    result_count: event.resultCount,
    action: event.action,
    clicked_position: event.clickedResultPosition,
    response_time_ms: event.responseTimeMs,
    has_filters: Object.keys(event.activeFilters ?? {}).length > 0,
  });

  // Send to Algolia Insights (if using Algolia)
  if (event.action === 'click' && event.clickedResultId) {
    algoliaInsights.clickedObjectIDsAfterSearch({
      eventName: 'Product Clicked',
      index: 'products',
      queryID: event.query,  // Use the actual queryID from Algolia
      objectIDs: [event.clickedResultId],
      positions: [event.clickedResultPosition ?? 1],
    });
  }

  // Track zero-result queries separately — they're gold
  if (event.resultCount === 0) {
    analytics.track('search_zero_results', {
      query: event.query,
      filters: event.activeFilters,
    });
  }
}
```

### Key Metrics to Monitor

```
┌──────────────────────────────────────────────────────────────────────┐
│                   SEARCH ANALYTICS DASHBOARD                         │
│                                                                      │
│  USAGE METRICS                                                       │
│  - Search volume: How many searches per day/week?                   │
│  - Unique searchers: What % of users use search?                    │
│  - Searches per session: How often do users search?                 │
│  - Search to browse ratio: Are users searching or browsing?         │
│                                                                      │
│  QUALITY METRICS                                                     │
│  - Zero-result rate: % of searches that return nothing              │
│    (Target: < 5%. Above 10% = serious problem.)                    │
│  - Click-through rate: % of searches that lead to a click           │
│    (Target: > 30%. Below 15% = relevance problem.)                 │
│  - Mean reciprocal rank: Average position of first clicked result   │
│    (Target: < 3. Above 5 = results ordering problem.)              │
│  - Refinement rate: % of searches followed by a filter change       │
│    (High rate = users can't find what they need initially.)         │
│                                                                      │
│  BUSINESS METRICS                                                    │
│  - Search-to-conversion rate: % of searches leading to action       │
│  - Revenue from search: $ generated from search-initiated flows     │
│  - Time to result: How long from first keystroke to click           │
│                                                                      │
│  DEBUGGING METRICS                                                   │
│  - Top searches: Most common queries                                │
│  - Top zero-result queries: What users want but can't find          │
│  - Slowest queries: Which queries take longest?                     │
│  - Search exit rate: % of users who leave after searching           │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

### Zero-Result Queries: Your Product Roadmap

Zero-result queries are the most actionable metric in your search analytics. Users are telling you exactly what they want, and you don't have it (or your search can't find it).

```typescript
// scripts/analyze-zero-results.ts

/**
 * Analyze zero-result queries to find opportunities.
 * Run weekly and share with product team.
 */
async function analyzeZeroResultQueries() {
  const zeroResults = await analytics.query({
    event: 'search_zero_results',
    timeRange: 'last_7_days',
    groupBy: 'query',
    orderBy: 'count_desc',
    limit: 50,
  });

  console.log('=== ZERO RESULT QUERIES (Last 7 Days) ===\n');
  console.log('Top queries with no results:');

  zeroResults.forEach((item: any, index: number) => {
    console.log(`  ${index + 1}. "${item.query}" (${item.count} searches)`);
  });

  console.log('\n--- ACTION ITEMS ---');
  console.log('1. Review top 10 queries — do we sell these products?');
  console.log('2. Check if any are synonym issues (e.g., "sneakers" vs "trainers")');
  console.log('3. Check if any are typo issues that our search engine should handle');
  console.log('4. Share with product team for roadmap input');
}
```

### Using Analytics to Improve Relevance

The feedback loop: track what users click on, feed it back into your search ranking:

```typescript
// src/search/personalization.ts

/**
 * Use click data to improve search relevance over time.
 * 
 * With Algolia: Use Click Analytics + Personalization features
 * With Typesense: Implement a custom boosting layer
 * With Postgres: Boost results that have higher click-through rates
 */

// For Algolia: send click events to improve relevance
import aa from 'search-insights';

aa('init', {
  appId: process.env.EXPO_PUBLIC_ALGOLIA_APP_ID!,
  apiKey: process.env.EXPO_PUBLIC_ALGOLIA_SEARCH_KEY!,
});

export function trackSearchClick(params: {
  queryId: string;
  objectId: string;
  position: number;
}) {
  aa('clickedObjectIDsAfterSearch', {
    eventName: 'Search Result Clicked',
    index: 'products',
    queryID: params.queryId,
    objectIDs: [params.objectId],
    positions: [params.position],
  });
}

export function trackSearchConversion(params: {
  queryId: string;
  objectId: string;
}) {
  aa('convertedObjectIDsAfterSearch', {
    eventName: 'Search Result Converted',
    index: 'products',
    queryID: params.queryId,
    objectIDs: [params.objectId],
  });
}

// For Postgres: maintain a click count and use it in ranking
// UPDATE products SET search_click_count = search_click_count + 1 WHERE id = $1;
//
// Then in your search query:
// ORDER BY (ts_rank(search_vector, query) * 3 + log(search_click_count + 1)) DESC
```

### The Search Improvement Cycle

Building great search isn't a one-time project. It's a continuous improvement cycle:

```
┌──────────────────────────────────────────────────────────────────────┐
│             THE SEARCH IMPROVEMENT CYCLE                             │
│                                                                      │
│        ┌──────────┐                                                  │
│        │  MEASURE  │  -- Track search analytics                      │
│        └────┬─────┘                                                  │
│             │                                                        │
│        ┌────v─────┐                                                  │
│        │ IDENTIFY  │  -- Find zero-result queries, low CTR queries  │
│        └────┬─────┘                                                  │
│             │                                                        │
│        ┌────v─────┐                                                  │
│        │  IMPROVE  │  -- Add synonyms, boost rules, new content     │
│        └────┬─────┘                                                  │
│             │                                                        │
│        ┌────v─────┐                                                  │
│        │ VALIDATE  │  -- A/B test the changes (see Chapter 37!)     │
│        └────┬─────┘                                                  │
│             │                                                        │
│             └────────── repeat                                       │
│                                                                      │
│  WEEKLY: Review zero-result queries, top queries, CTR               │
│  MONTHLY: Review overall search quality metrics                      │
│  QUARTERLY: Evaluate if your search provider still fits             │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

The best search experiences are built by teams that look at the data every week and make small, continuous improvements. Add a synonym here, boost a product category there, fix a ranking issue here. Each improvement is small, but they compound over time into a search experience that feels magical.

---

Search is one of those features that seems simple until you build it. But with the right tools, the right UX patterns, and a commitment to measuring and improving, you can build a search experience that makes users feel like your app can read their mind. That's the goal. Not just returning results — returning *the right results, instantly, even when users can't spell.*

---

**Next:** [Chapter 39: Analytics & Event Tracking](../part-4-architecture-at-scale/39-analytics.md)

**Previous:** [Chapter 37: Feature Flags, A/B Testing & Experimentation](../part-5-deployment-operations/37-feature-flags.md)
