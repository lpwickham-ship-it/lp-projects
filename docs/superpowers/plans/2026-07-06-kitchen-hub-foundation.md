# Kitchen Hub — Foundation (Phase 1 of 3)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Stand up a shared, phone-accessible web app for Louis-Philippe and his wife to track pantry/fridge/freezer/household inventory, snack & beverage inventory, and a grocery list that auto-suggests items from low/out inventory.

**Architecture:** Next.js App Router. Every page and mutation runs server-side and queries Neon Postgres directly using the `DATABASE_URL` connection string — the browser never talks to the database directly, so there's no way to bypass the passcode gate by reading a database URL out of the page source. A single shared passcode (not individual accounts) protects the whole app: entering it sets an HMAC-signed cookie, `proxy.ts` checks that cookie on every request.

**Tech Stack:** Next.js 16 (App Router), TypeScript (strict), Tailwind CSS v4, Neon Postgres (provisioned via Vercel Marketplace) using `@neondatabase/serverless`, Vercel, npm, Jest.

**Later phases (not built here):** Phase 2 adds an AI chat interface for weekly meal planning that reads/writes this data. Phase 3 adds meal ratings, leftover tracking, and expiration-date tracking. Do not build anything from those phases now.

## Global Constraints

- TypeScript strict mode — `"strict": true` in tsconfig. No `any`.
- Package manager: npm only. Never yarn or pnpm.
- All source files under `src/`.
- Inventory categories are exactly: `Produce`, `Meat & Seafood`, `Dairy`, `Frozen`, `Bakery`, `Pantry`, `Household`. Snack/beverage categories are exactly: `Snacks`, `Beverages`. No custom/user-defined categories.
- Inventory items use status only (`Stocked` / `Low` / `Out`) — never exact quantities.
- The `DATABASE_URL` env var must **never** be prefixed `NEXT_PUBLIC_` — it must never reach the browser bundle.
- Auth is a single shared passcode via a signed cookie — not any auth feature built into the database provider, and not individual accounts.
- Project folder: `projects/kitchen-hub/`. GitHub repo: `kitchen-hub`.
- Deploy target: Vercel Hobby tier + Neon free tier (via Vercel Marketplace). This phase must cost $0 to run.
- Environment variables (`DATABASE_URL`, `KITCHEN_HUB_PASSCODE`, `KITCHEN_HUB_SESSION_SECRET`) are provisioned through Vercel (Marketplace for `DATABASE_URL`, `vercel env add` for the app secrets) and synced locally with `vercel env pull .env.local` — they are already present in `.env.local` and in the Vercel project as of this plan; do not ask for them to be pasted manually.
- The Neon client must be lazily initialized (see Task 3) — calling `neon()` at module load time crashes `next build` before env vars are available. Never wrap the client in a JavaScript `Proxy`.
- No `console.log` left in committed code.

---

## File Map

Files created in this plan:

```
kitchen-hub/
├── src/
│   ├── app/
│   │   ├── layout.tsx                     — root layout: NavBar + body styles
│   │   ├── globals.css                    — Tailwind v4 import
│   │   ├── page.tsx                       — redirects to /inventory
│   │   ├── login/page.tsx                 — passcode entry form
│   │   ├── inventory/page.tsx             — pantry/fridge/freezer/household list
│   │   ├── snacks/page.tsx                — snack & beverage list
│   │   ├── grocery-list/page.tsx          — suggested + manual grocery list
│   │   ├── api/login/route.ts             — validates passcode, sets session cookie
│   │   └── actions/
│   │       ├── auth.ts                    — logout server action
│   │       ├── inventory.ts               — add/cycle-status/delete inventory items
│   │       ├── snacks.ts                  — add/adjust-quantity/toggle/delete snack items
│   │       └── groceryList.ts             — add/mark-purchased/clear/resolve-suggestion
│   ├── components/
│   │   ├── NavBar.tsx                     — top nav + log out (hidden on /login)
│   │   ├── InventoryItemRow.tsx           — one inventory row, tap to cycle status
│   │   ├── AddInventoryItemForm.tsx
│   │   ├── SnackItemRow.tsx               — quantity +/-, running-low toggle
│   │   ├── AddSnackItemForm.tsx
│   │   ├── GroceryListSection.tsx         — suggested + manual sections
│   │   └── AddGroceryItemForm.tsx
│   └── lib/
│       ├── types.ts                       — categories, statuses, row types
│       ├── auth.ts                        — session token compute/verify (pure, tested)
│       ├── inventoryStatus.ts             — status-cycling logic (pure, tested)
│       ├── quantity.ts                    — quantity clamping logic (pure, tested)
│       ├── groceryList.ts                 — suggestion-derivation logic (pure, tested)
│       └── db.ts                          — lazily-initialized Neon client
├── __tests__/
│   └── lib/
│       ├── auth.test.ts
│       ├── inventoryStatus.test.ts
│       ├── quantity.test.ts
│       └── groceryList.test.ts
├── db/
│   └── schema.sql
├── proxy.ts                                — passcode gate for every route
├── jest.config.ts
├── tailwind config via `@import "tailwindcss"` in globals.css (no tailwind.config.ts needed — v4)
├── next.config.ts
├── tsconfig.json
├── .env.local.example
├── .gitignore
└── README.md
```

---

## Task 1: Project Setup

**Status: COMPLETE.** Project scaffolded, committed (`0ff16a3`), reviewed clean.

**Note for later tasks:** Task 1 installed `@supabase/supabase-js`, which Task 2 below removes in favor of `@neondatabase/serverless`. This is expected — the project switched database providers after Task 1 was already built and reviewed.

---

## Task 2: Switch to Neon and Add the Database Schema

**Context:** Task 1 installed `@supabase/supabase-js` before the project switched to Neon Postgres (provisioned via Vercel Marketplace, already connected to this Vercel project with `DATABASE_URL` already present in `.env.local` and in Vercel's env store — no credentials need to be requested or pasted anywhere in this task). This task swaps the dependency and writes the schema file. **Applying the schema against the live database is handled outside this task** (by whoever is running this plan, using the connection string already in `.env.local`) — this task only needs to produce a correct, committed `db/schema.sql`.

**Files:**
- Modify: `package.json` (dependency swap)
- Modify: `.env.local.example`
- Modify: `README.md`
- Create: `db/schema.sql`

**Interfaces:**
- Produces: `db/schema.sql` defining three tables — `inventory_items`, `snack_beverage_items`, `grocery_list_items` — that Task 3's `getDb()` client and later tasks' queries depend on by name and column.

- [ ] **Step 1: Swap the database dependency**

  ```bash
  npm uninstall @supabase/supabase-js
  npm install @neondatabase/serverless
  ```
  Expected: `package.json` no longer lists `@supabase/supabase-js`; it lists `@neondatabase/serverless`.

- [ ] **Step 2: Update `.env.local.example`**

  ```
  DATABASE_URL=your-neon-connection-string
  KITCHEN_HUB_PASSCODE=choose-a-passcode
  KITCHEN_HUB_SESSION_SECRET=choose-a-long-random-string
  ```

  Do not modify the real `.env.local` — it already has working values and is gitignored.

- [ ] **Step 3: Update `README.md`**

  Replace the "Running Locally" and "Setting Up the Database" sections with:

  ```markdown
  ## Running Locally

  1. Run `vercel env pull .env.local` to sync your database connection string
     and app secrets from Vercel (or copy `.env.local.example` to `.env.local`
     and fill in values manually if you're not using the Vercel CLI).
  2. Run `npm install`
  3. Run `npm run dev`
  4. Open [http://localhost:3000](http://localhost:3000) and enter your passcode.

  ## Setting Up the Database

  The database is Neon Postgres, provisioned through the Vercel Marketplace
  integration for this project. Apply `db/schema.sql` once against your
  Neon database (via the Neon dashboard's SQL Editor, linked from the
  integration in the Vercel dashboard, or any Postgres client pointed at
  `DATABASE_URL`).
  ```

- [ ] **Step 4: Write `db/schema.sql`**

  ```sql
  -- inventory_items: pantry, fridge, freezer, and household items.
  -- Status only — no exact quantities.
  create table inventory_items (
    id uuid primary key default gen_random_uuid(),
    name text not null,
    category text not null check (category in (
      'Produce', 'Meat & Seafood', 'Dairy', 'Frozen', 'Bakery', 'Pantry', 'Household'
    )),
    status text not null default 'Stocked' check (status in ('Stocked', 'Low', 'Out')),
    updated_at timestamptz not null default now()
  );

  -- snack_beverage_items: count-based items with a manual running-low flag.
  create table snack_beverage_items (
    id uuid primary key default gen_random_uuid(),
    name text not null,
    category text not null check (category in ('Snacks', 'Beverages')),
    quantity integer not null default 0 check (quantity >= 0),
    running_low boolean not null default false,
    updated_at timestamptz not null default now()
  );

  -- grocery_list_items: manually added items only. Auto-suggested items are
  -- derived from the two tables above at read time, not stored here.
  create table grocery_list_items (
    id uuid primary key default gen_random_uuid(),
    name text not null,
    category text not null check (category in (
      'Produce', 'Meat & Seafood', 'Dairy', 'Frozen', 'Bakery', 'Pantry',
      'Household', 'Snacks', 'Beverages'
    )),
    purchased boolean not null default false,
    created_at timestamptz not null default now()
  );

  -- RLS enabled with no policies: deny-by-default for any role other than
  -- the table owner. The app connects using the Neon connection string's
  -- owner role from server-side code only, which bypasses RLS. This blocks
  -- any accidental access via a differently-scoped role in the future.
  alter table inventory_items enable row level security;
  alter table snack_beverage_items enable row level security;
  alter table grocery_list_items enable row level security;
  ```

- [ ] **Step 5: Verify the project still installs and type-checks**

  ```bash
  npm install
  npx tsc --noEmit
  ```
  Expected: no errors.

- [ ] **Step 6: Commit**

  ```bash
  git add package.json package-lock.json .env.local.example README.md db/schema.sql
  git commit -m "Switch database provider to Neon and add schema"
  ```

---

## Task 3: Shared Types, Category Constants, and Neon Database Client

**Files:**
- Create: `src/lib/types.ts`
- Create: `src/lib/db.ts`

**Interfaces:**
- Produces:
  - `INVENTORY_CATEGORIES: readonly string[]`, `InventoryCategory` type
  - `INVENTORY_STATUSES: readonly string[]`, `InventoryStatus` type
  - `SNACK_BEVERAGE_CATEGORIES: readonly string[]`, `SnackBeverageCategory` type
  - `ALL_GROCERY_CATEGORIES: readonly string[]`, `GroceryCategory` type
  - `InventoryItem`, `SnackBeverageItem`, `GroceryListItem` row types
  - `getDb()` — returns a Neon tagged-template SQL function. Every later task's database access goes through this: call as `` const sql = getDb(); const rows = await sql`SELECT * FROM inventory_items` ``.

- [ ] **Step 1: Create `src/lib/types.ts`**

  ```typescript
  // src/lib/types.ts
  export const INVENTORY_CATEGORIES = [
    'Produce',
    'Meat & Seafood',
    'Dairy',
    'Frozen',
    'Bakery',
    'Pantry',
    'Household',
  ] as const
  export type InventoryCategory = (typeof INVENTORY_CATEGORIES)[number]

  export const INVENTORY_STATUSES = ['Stocked', 'Low', 'Out'] as const
  export type InventoryStatus = (typeof INVENTORY_STATUSES)[number]

  export const SNACK_BEVERAGE_CATEGORIES = ['Snacks', 'Beverages'] as const
  export type SnackBeverageCategory = (typeof SNACK_BEVERAGE_CATEGORIES)[number]

  export const ALL_GROCERY_CATEGORIES = [
    ...INVENTORY_CATEGORIES,
    ...SNACK_BEVERAGE_CATEGORIES,
  ] as const
  export type GroceryCategory = (typeof ALL_GROCERY_CATEGORIES)[number]

  export interface InventoryItem {
    id: string
    name: string
    category: InventoryCategory
    status: InventoryStatus
  }

  export interface SnackBeverageItem {
    id: string
    name: string
    category: SnackBeverageCategory
    quantity: number
    running_low: boolean
  }

  export interface GroceryListItem {
    id: string
    name: string
    category: GroceryCategory
    purchased: boolean
  }
  ```

- [ ] **Step 2: Create `src/lib/db.ts`**

  ```typescript
  // src/lib/db.ts
  import { neon } from '@neondatabase/serverless'

  type Sql = ReturnType<typeof neon>

  let _sql: Sql | null = null

  export function getDb(): Sql {
    if (!_sql) _sql = neon(process.env.DATABASE_URL!)
    return _sql
  }
  ```

  This is lazily initialized on purpose: calling `neon()` at module load time
  would crash `next build` before `DATABASE_URL` is available. Do not wrap
  `_sql` in a JavaScript `Proxy` — later tasks call `getDb()` directly and
  use it as a tagged template.

- [ ] **Step 3: Verify it compiles**

  ```bash
  npx tsc --noEmit
  ```
  Expected: no errors.

- [ ] **Step 4: Commit**

  ```bash
  git add src/lib/types.ts src/lib/db.ts
  git commit -m "Add shared types and Neon database client"
  ```

---

## Task 4: Passcode Authentication

**Files:**
- Create: `src/lib/auth.ts`
- Test: `__tests__/lib/auth.test.ts`
- Create: `src/app/api/login/route.ts`
- Create: `src/app/login/page.tsx`
- Create: `proxy.ts`
- Create: `src/app/actions/auth.ts`

**Interfaces:**
- Consumes: none (no database access in this task)
- Produces:
  - `SESSION_COOKIE_NAME: string`
  - `computeSessionToken(secret: string): string`
  - `isValidSessionToken(token: string | undefined, secret: string): boolean`
  - `logout(): Promise<void>` (server action, used by Task 8's NavBar)

- [ ] **Step 1: Write the failing tests for `src/lib/auth.ts`**

  ```typescript
  // __tests__/lib/auth.test.ts
  import { computeSessionToken, isValidSessionToken } from '@/lib/auth'

  describe('computeSessionToken', () => {
    it('produces the same token for the same secret', () => {
      expect(computeSessionToken('test-secret')).toBe(computeSessionToken('test-secret'))
    })

    it('produces different tokens for different secrets', () => {
      expect(computeSessionToken('secret-a')).not.toBe(computeSessionToken('secret-b'))
    })
  })

  describe('isValidSessionToken', () => {
    it('accepts a token computed with the matching secret', () => {
      const token = computeSessionToken('test-secret')
      expect(isValidSessionToken(token, 'test-secret')).toBe(true)
    })

    it('rejects a token computed with a different secret', () => {
      const token = computeSessionToken('wrong-secret')
      expect(isValidSessionToken(token, 'test-secret')).toBe(false)
    })

    it('rejects an undefined token', () => {
      expect(isValidSessionToken(undefined, 'test-secret')).toBe(false)
    })

    it('rejects a malformed token', () => {
      expect(isValidSessionToken('not-a-real-token', 'test-secret')).toBe(false)
    })
  })
  ```

- [ ] **Step 2: Run tests to verify they fail**

  ```bash
  npm test -- auth.test.ts
  ```
  Expected: FAIL with "Cannot find module '@/lib/auth'".

- [ ] **Step 3: Write `src/lib/auth.ts`**

  ```typescript
  // src/lib/auth.ts
  import { createHmac, timingSafeEqual } from 'crypto'

  export const SESSION_COOKIE_NAME = 'kitchen_hub_session'

  export function computeSessionToken(secret: string): string {
    return createHmac('sha256', secret).update('kitchen-hub-authenticated').digest('hex')
  }

  export function isValidSessionToken(token: string | undefined, secret: string): boolean {
    if (!token) return false
    const expected = computeSessionToken(secret)
    const expectedBuffer = Buffer.from(expected, 'hex')
    const tokenBuffer = Buffer.from(token, 'hex')
    if (expectedBuffer.length !== tokenBuffer.length) return false
    return timingSafeEqual(expectedBuffer, tokenBuffer)
  }
  ```

- [ ] **Step 4: Run tests to verify they pass**

  ```bash
  npm test -- auth.test.ts
  ```
  Expected: PASS, 5 tests.

- [ ] **Step 5: Commit**

  ```bash
  git add src/lib/auth.ts __tests__/lib/auth.test.ts
  git commit -m "Add passcode session token logic"
  ```

- [ ] **Step 6: Create `src/app/api/login/route.ts`**

  ```typescript
  // src/app/api/login/route.ts
  import { NextResponse } from 'next/server'
  import { SESSION_COOKIE_NAME, computeSessionToken } from '@/lib/auth'

  export async function POST(request: Request) {
    const body = await request.json()
    const passcode = body?.passcode

    if (typeof passcode !== 'string' || passcode !== process.env.KITCHEN_HUB_PASSCODE) {
      return NextResponse.json({ error: 'Incorrect passcode' }, { status: 401 })
    }

    const token = computeSessionToken(process.env.KITCHEN_HUB_SESSION_SECRET!)
    const response = NextResponse.json({ success: true })
    response.cookies.set(SESSION_COOKIE_NAME, token, {
      httpOnly: true,
      secure: process.env.NODE_ENV === 'production',
      sameSite: 'lax',
      path: '/',
      maxAge: 60 * 60 * 24 * 365,
    })
    return response
  }
  ```

- [ ] **Step 7: Create `src/app/login/page.tsx`**

  ```tsx
  // src/app/login/page.tsx
  'use client'

  import { useState } from 'react'
  import { useRouter } from 'next/navigation'

  export default function LoginPage() {
    const router = useRouter()
    const [passcode, setPasscode] = useState('')
    const [error, setError] = useState<string | null>(null)
    const [loading, setLoading] = useState(false)

    async function handleSubmit(e: React.FormEvent) {
      e.preventDefault()
      setLoading(true)
      setError(null)

      const res = await fetch('/api/login', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ passcode }),
      })

      if (res.ok) {
        router.push('/inventory')
        router.refresh()
      } else {
        setError('Incorrect passcode')
        setLoading(false)
      }
    }

    return (
      <div className="min-h-screen flex items-center justify-center bg-neutral-50 px-6">
        <form onSubmit={handleSubmit} className="max-w-sm w-full space-y-4">
          <h1 className="text-2xl font-semibold text-center">Kitchen Hub</h1>
          <input
            type="password"
            value={passcode}
            onChange={e => setPasscode(e.target.value)}
            placeholder="Passcode"
            required
            className="w-full border border-neutral-300 rounded px-4 py-3 text-center"
          />
          {error && <p className="text-red-600 text-sm text-center">{error}</p>}
          <button
            type="submit"
            disabled={loading}
            className="w-full bg-neutral-900 text-white rounded py-3 disabled:opacity-50"
          >
            {loading ? 'Checking…' : 'Enter'}
          </button>
        </form>
      </div>
    )
  }
  ```

- [ ] **Step 8: Create `proxy.ts` at the project root (same level as `src/`)**

  ```typescript
  // proxy.ts
  import { NextResponse } from 'next/server'
  import type { NextRequest } from 'next/server'
  import { SESSION_COOKIE_NAME, isValidSessionToken } from '@/lib/auth'

  export function proxy(request: NextRequest) {
    const token = request.cookies.get(SESSION_COOKIE_NAME)?.value
    const authenticated = isValidSessionToken(token, process.env.KITCHEN_HUB_SESSION_SECRET!)

    if (!authenticated) {
      return NextResponse.redirect(new URL('/login', request.url))
    }

    return NextResponse.next()
  }

  export const config = {
    matcher: ['/((?!login|api/login|_next/static|_next/image|favicon.ico).*)'],
  }
  ```

- [ ] **Step 9: Create `src/app/actions/auth.ts`**

  ```typescript
  // src/app/actions/auth.ts
  'use server'

  import { cookies } from 'next/headers'
  import { SESSION_COOKIE_NAME } from '@/lib/auth'

  export async function logout() {
    const cookieStore = await cookies()
    cookieStore.delete(SESSION_COOKIE_NAME)
  }
  ```

- [ ] **Step 10: Manually verify the passcode gate**

  ```bash
  npm run dev
  ```
  In a browser, visit `http://localhost:3000/` — expect a redirect to `/login` (there's no `/inventory` page yet, so you'll get a 404 after logging in; that's expected until Task 5). Enter the wrong passcode — expect "Incorrect passcode". Enter the correct passcode — expect the redirect to fire (even though the destination page doesn't exist yet).

- [ ] **Step 11: Commit**

  ```bash
  git add src/app/api/login/route.ts src/app/login/page.tsx proxy.ts src/app/actions/auth.ts
  git commit -m "Add passcode login flow and route protection"
  ```

---

## Task 5: Inventory Page

**Files:**
- Create: `src/lib/inventoryStatus.ts`
- Test: `__tests__/lib/inventoryStatus.test.ts`
- Create: `src/app/actions/inventory.ts`
- Create: `src/components/InventoryItemRow.tsx`
- Create: `src/components/AddInventoryItemForm.tsx`
- Create: `src/app/inventory/page.tsx`

**Interfaces:**
- Consumes: `getDb` from `@/lib/db` (Task 3), `InventoryItem`, `InventoryCategory`, `InventoryStatus`, `INVENTORY_CATEGORIES` from `@/lib/types` (Task 3)
- Produces: `nextInventoryStatus(current: InventoryStatus): InventoryStatus`, and server actions `addInventoryItem(name, category)`, `cycleInventoryItemStatus(id, currentStatus)`, `deleteInventoryItem(id)` — Task 7 also queries the same underlying `inventory_items` table.

- [ ] **Step 1: Write the failing test for status cycling**

  ```typescript
  // __tests__/lib/inventoryStatus.test.ts
  import { nextInventoryStatus } from '@/lib/inventoryStatus'

  it('cycles Stocked to Low', () => {
    expect(nextInventoryStatus('Stocked')).toBe('Low')
  })

  it('cycles Low to Out', () => {
    expect(nextInventoryStatus('Low')).toBe('Out')
  })

  it('cycles Out back to Stocked', () => {
    expect(nextInventoryStatus('Out')).toBe('Stocked')
  })
  ```

- [ ] **Step 2: Run test to verify it fails**

  ```bash
  npm test -- inventoryStatus.test.ts
  ```
  Expected: FAIL with "Cannot find module '@/lib/inventoryStatus'".

- [ ] **Step 3: Write `src/lib/inventoryStatus.ts`**

  ```typescript
  // src/lib/inventoryStatus.ts
  import type { InventoryStatus } from './types'

  const STATUS_CYCLE: InventoryStatus[] = ['Stocked', 'Low', 'Out']

  export function nextInventoryStatus(current: InventoryStatus): InventoryStatus {
    const currentIndex = STATUS_CYCLE.indexOf(current)
    const nextIndex = (currentIndex + 1) % STATUS_CYCLE.length
    return STATUS_CYCLE[nextIndex]
  }
  ```

- [ ] **Step 4: Run test to verify it passes**

  ```bash
  npm test -- inventoryStatus.test.ts
  ```
  Expected: PASS, 3 tests.

- [ ] **Step 5: Commit**

  ```bash
  git add src/lib/inventoryStatus.ts __tests__/lib/inventoryStatus.test.ts
  git commit -m "Add inventory status-cycling logic"
  ```

- [ ] **Step 6: Create `src/app/actions/inventory.ts`**

  ```typescript
  // src/app/actions/inventory.ts
  'use server'

  import { revalidatePath } from 'next/cache'
  import { getDb } from '@/lib/db'
  import { nextInventoryStatus } from '@/lib/inventoryStatus'
  import type { InventoryCategory, InventoryStatus } from '@/lib/types'

  export async function addInventoryItem(name: string, category: InventoryCategory) {
    const sql = getDb()
    await sql`INSERT INTO inventory_items (name, category) VALUES (${name}, ${category})`
    revalidatePath('/inventory')
  }

  export async function cycleInventoryItemStatus(id: string, currentStatus: InventoryStatus) {
    const sql = getDb()
    const next = nextInventoryStatus(currentStatus)
    await sql`UPDATE inventory_items SET status = ${next}, updated_at = now() WHERE id = ${id}`
    revalidatePath('/inventory')
    revalidatePath('/grocery-list')
  }

  export async function deleteInventoryItem(id: string) {
    const sql = getDb()
    await sql`DELETE FROM inventory_items WHERE id = ${id}`
    revalidatePath('/inventory')
    revalidatePath('/grocery-list')
  }
  ```

- [ ] **Step 7: Create `src/components/InventoryItemRow.tsx`**

  ```tsx
  // src/components/InventoryItemRow.tsx
  'use client'

  import { useTransition } from 'react'
  import { cycleInventoryItemStatus, deleteInventoryItem } from '@/app/actions/inventory'
  import type { InventoryItem, InventoryStatus } from '@/lib/types'

  const STATUS_COLORS: Record<InventoryStatus, string> = {
    Stocked: 'bg-green-100 text-green-800',
    Low: 'bg-yellow-100 text-yellow-800',
    Out: 'bg-red-100 text-red-800',
  }

  export function InventoryItemRow({ item }: { item: InventoryItem }) {
    const [isPending, startTransition] = useTransition()

    return (
      <li className="flex items-center justify-between py-2">
        <span>{item.name}</span>
        <div className="flex items-center gap-2">
          <button
            type="button"
            disabled={isPending}
            onClick={() => startTransition(() => cycleInventoryItemStatus(item.id, item.status))}
            className={`px-3 py-1 rounded text-sm ${STATUS_COLORS[item.status]}`}
          >
            {item.status}
          </button>
          <button
            type="button"
            disabled={isPending}
            onClick={() => startTransition(() => deleteInventoryItem(item.id))}
            className="text-neutral-400 hover:text-red-600 text-sm"
          >
            Remove
          </button>
        </div>
      </li>
    )
  }
  ```

- [ ] **Step 8: Create `src/components/AddInventoryItemForm.tsx`**

  ```tsx
  // src/components/AddInventoryItemForm.tsx
  'use client'

  import { useState, useTransition } from 'react'
  import { addInventoryItem } from '@/app/actions/inventory'
  import { INVENTORY_CATEGORIES, type InventoryCategory } from '@/lib/types'

  export function AddInventoryItemForm() {
    const [name, setName] = useState('')
    const [category, setCategory] = useState<InventoryCategory>(INVENTORY_CATEGORIES[0])
    const [isPending, startTransition] = useTransition()

    function handleSubmit(e: React.FormEvent) {
      e.preventDefault()
      if (!name.trim()) return
      const itemName = name.trim()
      startTransition(async () => {
        await addInventoryItem(itemName, category)
        setName('')
      })
    }

    return (
      <form onSubmit={handleSubmit} className="flex gap-2 py-4">
        <input
          value={name}
          onChange={e => setName(e.target.value)}
          placeholder="Item name"
          className="flex-1 border border-neutral-300 rounded px-3 py-2"
        />
        <select
          value={category}
          onChange={e => setCategory(e.target.value as InventoryCategory)}
          className="border border-neutral-300 rounded px-3 py-2"
        >
          {INVENTORY_CATEGORIES.map(c => (
            <option key={c} value={c}>{c}</option>
          ))}
        </select>
        <button
          type="submit"
          disabled={isPending}
          className="bg-neutral-900 text-white rounded px-4 py-2 disabled:opacity-50"
        >
          Add
        </button>
      </form>
    )
  }
  ```

- [ ] **Step 9: Create `src/app/inventory/page.tsx`**

  ```tsx
  // src/app/inventory/page.tsx
  import { getDb } from '@/lib/db'
  import { INVENTORY_CATEGORIES, type InventoryItem } from '@/lib/types'
  import { InventoryItemRow } from '@/components/InventoryItemRow'
  import { AddInventoryItemForm } from '@/components/AddInventoryItemForm'

  export const dynamic = 'force-dynamic'

  export default async function InventoryPage() {
    const sql = getDb()
    let items: InventoryItem[]
    try {
      items = (await sql`SELECT * FROM inventory_items ORDER BY name`) as InventoryItem[]
    } catch {
      return <p className="p-6 text-red-600">Couldn&apos;t load inventory — try again.</p>
    }

    return (
      <main className="max-w-2xl mx-auto p-6">
        <h1 className="text-2xl font-semibold mb-4">Pantry, Fridge &amp; Freezer</h1>
        <AddInventoryItemForm />
        {INVENTORY_CATEGORIES.map(category => {
          const categoryItems = items.filter(item => item.category === category)
          if (categoryItems.length === 0) return null
          return (
            <section key={category} className="mb-6">
              <h2 className="text-sm uppercase tracking-wide text-neutral-500 mb-2">{category}</h2>
              <ul className="divide-y divide-neutral-200">
                {categoryItems.map(item => (
                  <InventoryItemRow key={item.id} item={item} />
                ))}
              </ul>
            </section>
          )
        })}
      </main>
    )
  }
  ```

- [ ] **Step 10: Manually verify**

  ```bash
  npm run dev
  ```
  Log in, visit `/inventory`, add an item (e.g. "Olive Oil" / Pantry), confirm it appears, tap its status badge to cycle Stocked → Low → Out → Stocked, then remove it.

- [ ] **Step 11: Commit**

  ```bash
  git add src/app/actions/inventory.ts src/components/InventoryItemRow.tsx \
    src/components/AddInventoryItemForm.tsx src/app/inventory/page.tsx
  git commit -m "Add inventory page"
  ```

---

## Task 6: Snacks & Beverages Page

**Files:**
- Create: `src/lib/quantity.ts`
- Test: `__tests__/lib/quantity.test.ts`
- Create: `src/app/actions/snacks.ts`
- Create: `src/components/SnackItemRow.tsx`
- Create: `src/components/AddSnackItemForm.tsx`
- Create: `src/app/snacks/page.tsx`

**Interfaces:**
- Consumes: `getDb` (Task 3), `SnackBeverageItem`, `SnackBeverageCategory`, `SNACK_BEVERAGE_CATEGORIES` from `@/lib/types` (Task 3)
- Produces: `clampQuantity(quantity: number): number`, and server actions `addSnackItem(name, category)`, `adjustSnackQuantity(id, currentQuantity, delta)`, `toggleRunningLow(id, currentValue)`, `deleteSnackItem(id)` — Task 7 queries the `snack_beverage_items` table these write to.

- [ ] **Step 1: Write the failing test for quantity clamping**

  ```typescript
  // __tests__/lib/quantity.test.ts
  import { clampQuantity } from '@/lib/quantity'

  it('keeps positive quantities unchanged', () => {
    expect(clampQuantity(5)).toBe(5)
  })

  it('clamps negative quantities to zero', () => {
    expect(clampQuantity(-3)).toBe(0)
  })

  it('keeps zero as zero', () => {
    expect(clampQuantity(0)).toBe(0)
  })
  ```

- [ ] **Step 2: Run test to verify it fails**

  ```bash
  npm test -- quantity.test.ts
  ```
  Expected: FAIL with "Cannot find module '@/lib/quantity'".

- [ ] **Step 3: Write `src/lib/quantity.ts`**

  ```typescript
  // src/lib/quantity.ts
  export function clampQuantity(quantity: number): number {
    return Math.max(0, quantity)
  }
  ```

- [ ] **Step 4: Run test to verify it passes**

  ```bash
  npm test -- quantity.test.ts
  ```
  Expected: PASS, 3 tests.

- [ ] **Step 5: Commit**

  ```bash
  git add src/lib/quantity.ts __tests__/lib/quantity.test.ts
  git commit -m "Add quantity clamping logic"
  ```

- [ ] **Step 6: Create `src/app/actions/snacks.ts`**

  ```typescript
  // src/app/actions/snacks.ts
  'use server'

  import { revalidatePath } from 'next/cache'
  import { getDb } from '@/lib/db'
  import { clampQuantity } from '@/lib/quantity'
  import type { SnackBeverageCategory } from '@/lib/types'

  export async function addSnackItem(name: string, category: SnackBeverageCategory) {
    const sql = getDb()
    await sql`INSERT INTO snack_beverage_items (name, category, quantity) VALUES (${name}, ${category}, 0)`
    revalidatePath('/snacks')
  }

  export async function adjustSnackQuantity(id: string, currentQuantity: number, delta: number) {
    const sql = getDb()
    const nextQuantity = clampQuantity(currentQuantity + delta)
    await sql`UPDATE snack_beverage_items SET quantity = ${nextQuantity}, updated_at = now() WHERE id = ${id}`
    revalidatePath('/snacks')
  }

  export async function toggleRunningLow(id: string, currentValue: boolean) {
    const sql = getDb()
    await sql`UPDATE snack_beverage_items SET running_low = ${!currentValue}, updated_at = now() WHERE id = ${id}`
    revalidatePath('/snacks')
    revalidatePath('/grocery-list')
  }

  export async function deleteSnackItem(id: string) {
    const sql = getDb()
    await sql`DELETE FROM snack_beverage_items WHERE id = ${id}`
    revalidatePath('/snacks')
    revalidatePath('/grocery-list')
  }
  ```

- [ ] **Step 7: Create `src/components/SnackItemRow.tsx`**

  ```tsx
  // src/components/SnackItemRow.tsx
  'use client'

  import { useTransition } from 'react'
  import { adjustSnackQuantity, toggleRunningLow, deleteSnackItem } from '@/app/actions/snacks'
  import type { SnackBeverageItem } from '@/lib/types'

  export function SnackItemRow({ item }: { item: SnackBeverageItem }) {
    const [isPending, startTransition] = useTransition()

    return (
      <li className="flex items-center justify-between py-2">
        <span>{item.name}</span>
        <div className="flex items-center gap-3">
          <button
            type="button"
            disabled={isPending}
            onClick={() => startTransition(() => adjustSnackQuantity(item.id, item.quantity, -1))}
            className="px-2 py-1 border rounded"
          >
            −
          </button>
          <span className="w-6 text-center">{item.quantity}</span>
          <button
            type="button"
            disabled={isPending}
            onClick={() => startTransition(() => adjustSnackQuantity(item.id, item.quantity, 1))}
            className="px-2 py-1 border rounded"
          >
            +
          </button>
          <label className="flex items-center gap-1 text-sm">
            <input
              type="checkbox"
              checked={item.running_low}
              disabled={isPending}
              onChange={() => startTransition(() => toggleRunningLow(item.id, item.running_low))}
            />
            Running low
          </label>
          <button
            type="button"
            disabled={isPending}
            onClick={() => startTransition(() => deleteSnackItem(item.id))}
            className="text-neutral-400 hover:text-red-600 text-sm"
          >
            Remove
          </button>
        </div>
      </li>
    )
  }
  ```

- [ ] **Step 8: Create `src/components/AddSnackItemForm.tsx`**

  ```tsx
  // src/components/AddSnackItemForm.tsx
  'use client'

  import { useState, useTransition } from 'react'
  import { addSnackItem } from '@/app/actions/snacks'
  import { SNACK_BEVERAGE_CATEGORIES, type SnackBeverageCategory } from '@/lib/types'

  export function AddSnackItemForm() {
    const [name, setName] = useState('')
    const [category, setCategory] = useState<SnackBeverageCategory>(SNACK_BEVERAGE_CATEGORIES[0])
    const [isPending, startTransition] = useTransition()

    function handleSubmit(e: React.FormEvent) {
      e.preventDefault()
      if (!name.trim()) return
      const itemName = name.trim()
      startTransition(async () => {
        await addSnackItem(itemName, category)
        setName('')
      })
    }

    return (
      <form onSubmit={handleSubmit} className="flex gap-2 py-4">
        <input
          value={name}
          onChange={e => setName(e.target.value)}
          placeholder="Item name"
          className="flex-1 border border-neutral-300 rounded px-3 py-2"
        />
        <select
          value={category}
          onChange={e => setCategory(e.target.value as SnackBeverageCategory)}
          className="border border-neutral-300 rounded px-3 py-2"
        >
          {SNACK_BEVERAGE_CATEGORIES.map(c => (
            <option key={c} value={c}>{c}</option>
          ))}
        </select>
        <button
          type="submit"
          disabled={isPending}
          className="bg-neutral-900 text-white rounded px-4 py-2 disabled:opacity-50"
        >
          Add
        </button>
      </form>
    )
  }
  ```

- [ ] **Step 9: Create `src/app/snacks/page.tsx`**

  ```tsx
  // src/app/snacks/page.tsx
  import { getDb } from '@/lib/db'
  import { SNACK_BEVERAGE_CATEGORIES, type SnackBeverageItem } from '@/lib/types'
  import { SnackItemRow } from '@/components/SnackItemRow'
  import { AddSnackItemForm } from '@/components/AddSnackItemForm'

  export const dynamic = 'force-dynamic'

  export default async function SnacksPage() {
    const sql = getDb()
    let items: SnackBeverageItem[]
    try {
      items = (await sql`SELECT * FROM snack_beverage_items ORDER BY name`) as SnackBeverageItem[]
    } catch {
      return <p className="p-6 text-red-600">Couldn&apos;t load snacks &amp; drinks — try again.</p>
    }

    return (
      <main className="max-w-2xl mx-auto p-6">
        <h1 className="text-2xl font-semibold mb-4">Snacks &amp; Drinks</h1>
        <AddSnackItemForm />
        {SNACK_BEVERAGE_CATEGORIES.map(category => {
          const categoryItems = items.filter(item => item.category === category)
          if (categoryItems.length === 0) return null
          return (
            <section key={category} className="mb-6">
              <h2 className="text-sm uppercase tracking-wide text-neutral-500 mb-2">{category}</h2>
              <ul className="divide-y divide-neutral-200">
                {categoryItems.map(item => (
                  <SnackItemRow key={item.id} item={item} />
                ))}
              </ul>
            </section>
          )
        })}
      </main>
    )
  }
  ```

- [ ] **Step 10: Manually verify**

  ```bash
  npm run dev
  ```
  Visit `/snacks`, add an item (e.g. "Protein Bars" / Snacks), use +/- to adjust quantity, confirm it never goes below 0, toggle "Running low", then remove it.

- [ ] **Step 11: Commit**

  ```bash
  git add src/app/actions/snacks.ts src/components/SnackItemRow.tsx \
    src/components/AddSnackItemForm.tsx src/app/snacks/page.tsx
  git commit -m "Add snacks & beverages page"
  ```

---

## Task 7: Grocery List Page

**Files:**
- Create: `src/lib/groceryList.ts`
- Test: `__tests__/lib/groceryList.test.ts`
- Create: `src/app/actions/groceryList.ts`
- Create: `src/components/GroceryListSection.tsx`
- Create: `src/components/AddGroceryItemForm.tsx`
- Create: `src/app/grocery-list/page.tsx`

**Interfaces:**
- Consumes: `InventoryItem`, `SnackBeverageItem`, `GroceryListItem`, `GroceryCategory`, `ALL_GROCERY_CATEGORIES` from `@/lib/types` (Task 3); `getDb` (Task 3); queries the same `inventory_items` and `snack_beverage_items` tables that Tasks 5 and 6 write to.
- Produces: `deriveSuggestedItems(inventoryItems, snackItems): SuggestedGroceryItem[]` where `SuggestedGroceryItem = { sourceType: 'inventory' | 'snack', sourceId: string, name: string, category: GroceryCategory }`

- [ ] **Step 1: Write the failing tests for suggestion derivation**

  ```typescript
  // __tests__/lib/groceryList.test.ts
  import { deriveSuggestedItems } from '@/lib/groceryList'
  import type { InventoryItem, SnackBeverageItem } from '@/lib/types'

  const stocked: InventoryItem = { id: '1', name: 'Rice', category: 'Pantry', status: 'Stocked' }
  const low: InventoryItem = { id: '2', name: 'Olive Oil', category: 'Pantry', status: 'Low' }
  const out: InventoryItem = { id: '3', name: 'Milk', category: 'Dairy', status: 'Out' }

  const notRunningLow: SnackBeverageItem = {
    id: '4', name: 'Chips', category: 'Snacks', quantity: 5, running_low: false,
  }
  const runningLow: SnackBeverageItem = {
    id: '5', name: 'Sparkling Water', category: 'Beverages', quantity: 1, running_low: true,
  }

  it('excludes stocked inventory items', () => {
    expect(deriveSuggestedItems([stocked], [])).toEqual([])
  })

  it('includes low and out inventory items', () => {
    const result = deriveSuggestedItems([low, out], [])
    expect(result.map(r => r.name)).toEqual(['Olive Oil', 'Milk'])
  })

  it('excludes snack items that are not running low', () => {
    expect(deriveSuggestedItems([], [notRunningLow])).toEqual([])
  })

  it('includes snack items marked running low', () => {
    const result = deriveSuggestedItems([], [runningLow])
    expect(result).toHaveLength(1)
    expect(result[0]).toEqual({
      sourceType: 'snack',
      sourceId: '5',
      name: 'Sparkling Water',
      category: 'Beverages',
    })
  })

  it('combines inventory and snack suggestions', () => {
    const result = deriveSuggestedItems([stocked, low], [runningLow])
    expect(result.map(r => r.name)).toEqual(['Olive Oil', 'Sparkling Water'])
  })
  ```

- [ ] **Step 2: Run tests to verify they fail**

  ```bash
  npm test -- groceryList.test.ts
  ```
  Expected: FAIL with "Cannot find module '@/lib/groceryList'".

- [ ] **Step 3: Write `src/lib/groceryList.ts`**

  ```typescript
  // src/lib/groceryList.ts
  import type { InventoryItem, SnackBeverageItem, GroceryCategory } from './types'

  export interface SuggestedGroceryItem {
    sourceType: 'inventory' | 'snack'
    sourceId: string
    name: string
    category: GroceryCategory
  }

  export function deriveSuggestedItems(
    inventoryItems: InventoryItem[],
    snackItems: SnackBeverageItem[]
  ): SuggestedGroceryItem[] {
    const fromInventory = inventoryItems
      .filter(item => item.status !== 'Stocked')
      .map(item => ({
        sourceType: 'inventory' as const,
        sourceId: item.id,
        name: item.name,
        category: item.category,
      }))

    const fromSnacks = snackItems
      .filter(item => item.running_low)
      .map(item => ({
        sourceType: 'snack' as const,
        sourceId: item.id,
        name: item.name,
        category: item.category,
      }))

    return [...fromInventory, ...fromSnacks]
  }
  ```

- [ ] **Step 4: Run tests to verify they pass**

  ```bash
  npm test -- groceryList.test.ts
  ```
  Expected: PASS, 5 tests.

- [ ] **Step 5: Commit**

  ```bash
  git add src/lib/groceryList.ts __tests__/lib/groceryList.test.ts
  git commit -m "Add grocery suggestion derivation logic"
  ```

- [ ] **Step 6: Create `src/app/actions/groceryList.ts`**

  ```typescript
  // src/app/actions/groceryList.ts
  'use server'

  import { revalidatePath } from 'next/cache'
  import { getDb } from '@/lib/db'
  import type { GroceryCategory } from '@/lib/types'

  export async function addGroceryItem(name: string, category: GroceryCategory) {
    const sql = getDb()
    await sql`INSERT INTO grocery_list_items (name, category) VALUES (${name}, ${category})`
    revalidatePath('/grocery-list')
  }

  export async function markGroceryItemPurchased(id: string) {
    const sql = getDb()
    await sql`UPDATE grocery_list_items SET purchased = true WHERE id = ${id}`
    revalidatePath('/grocery-list')
  }

  export async function clearPurchasedGroceryItems() {
    const sql = getDb()
    await sql`DELETE FROM grocery_list_items WHERE purchased = true`
    revalidatePath('/grocery-list')
  }

  export async function resolveSuggestedItem(sourceType: 'inventory' | 'snack', sourceId: string) {
    const sql = getDb()

    if (sourceType === 'inventory') {
      await sql`UPDATE inventory_items SET status = 'Stocked', updated_at = now() WHERE id = ${sourceId}`
    } else {
      await sql`UPDATE snack_beverage_items SET running_low = false, updated_at = now() WHERE id = ${sourceId}`
    }

    revalidatePath('/grocery-list')
    revalidatePath('/inventory')
    revalidatePath('/snacks')
  }
  ```

- [ ] **Step 7: Create `src/components/GroceryListSection.tsx`**

  ```tsx
  // src/components/GroceryListSection.tsx
  'use client'

  import { useTransition } from 'react'
  import {
    resolveSuggestedItem,
    markGroceryItemPurchased,
    clearPurchasedGroceryItems,
  } from '@/app/actions/groceryList'
  import type { SuggestedGroceryItem } from '@/lib/groceryList'
  import type { GroceryListItem } from '@/lib/types'

  export function GroceryListSection({
    suggested,
    manual,
  }: {
    suggested: SuggestedGroceryItem[]
    manual: GroceryListItem[]
  }) {
    const [isPending, startTransition] = useTransition()

    return (
      <div className="space-y-8">
        <section>
          <h2 className="text-sm uppercase tracking-wide text-neutral-500 mb-2">Suggested</h2>
          {suggested.length === 0 ? (
            <p className="text-neutral-400 text-sm">Nothing needs restocking.</p>
          ) : (
            <ul className="divide-y divide-neutral-200">
              {suggested.map(item => (
                <li key={`${item.sourceType}-${item.sourceId}`} className="flex items-center gap-3 py-2">
                  <input
                    type="checkbox"
                    disabled={isPending}
                    onChange={() => startTransition(() => resolveSuggestedItem(item.sourceType, item.sourceId))}
                  />
                  <span>{item.name}</span>
                  <span className="text-xs text-neutral-400">{item.category}</span>
                </li>
              ))}
            </ul>
          )}
        </section>

        <section>
          <div className="flex items-center justify-between mb-2">
            <h2 className="text-sm uppercase tracking-wide text-neutral-500">Added by you</h2>
            <button
              type="button"
              disabled={isPending}
              onClick={() => startTransition(() => clearPurchasedGroceryItems())}
              className="text-xs text-neutral-400 hover:text-neutral-900"
            >
              Clear purchased
            </button>
          </div>
          {manual.length === 0 ? (
            <p className="text-neutral-400 text-sm">No items added yet.</p>
          ) : (
            <ul className="divide-y divide-neutral-200">
              {manual.map(item => (
                <li key={item.id} className="flex items-center gap-3 py-2">
                  <input
                    type="checkbox"
                    disabled={isPending}
                    onChange={() => startTransition(() => markGroceryItemPurchased(item.id))}
                  />
                  <span>{item.name}</span>
                  <span className="text-xs text-neutral-400">{item.category}</span>
                </li>
              ))}
            </ul>
          )}
        </section>
      </div>
    )
  }
  ```

- [ ] **Step 8: Create `src/components/AddGroceryItemForm.tsx`**

  ```tsx
  // src/components/AddGroceryItemForm.tsx
  'use client'

  import { useState, useTransition } from 'react'
  import { addGroceryItem } from '@/app/actions/groceryList'
  import { ALL_GROCERY_CATEGORIES, type GroceryCategory } from '@/lib/types'

  export function AddGroceryItemForm() {
    const [name, setName] = useState('')
    const [category, setCategory] = useState<GroceryCategory>(ALL_GROCERY_CATEGORIES[0])
    const [isPending, startTransition] = useTransition()

    function handleSubmit(e: React.FormEvent) {
      e.preventDefault()
      if (!name.trim()) return
      const itemName = name.trim()
      startTransition(async () => {
        await addGroceryItem(itemName, category)
        setName('')
      })
    }

    return (
      <form onSubmit={handleSubmit} className="flex gap-2 py-4">
        <input
          value={name}
          onChange={e => setName(e.target.value)}
          placeholder="Item name"
          className="flex-1 border border-neutral-300 rounded px-3 py-2"
        />
        <select
          value={category}
          onChange={e => setCategory(e.target.value as GroceryCategory)}
          className="border border-neutral-300 rounded px-3 py-2"
        >
          {ALL_GROCERY_CATEGORIES.map(c => (
            <option key={c} value={c}>{c}</option>
          ))}
        </select>
        <button
          type="submit"
          disabled={isPending}
          className="bg-neutral-900 text-white rounded px-4 py-2 disabled:opacity-50"
        >
          Add
        </button>
      </form>
    )
  }
  ```

- [ ] **Step 9: Create `src/app/grocery-list/page.tsx`**

  ```tsx
  // src/app/grocery-list/page.tsx
  import { getDb } from '@/lib/db'
  import { deriveSuggestedItems } from '@/lib/groceryList'
  import { GroceryListSection } from '@/components/GroceryListSection'
  import { AddGroceryItemForm } from '@/components/AddGroceryItemForm'
  import type { InventoryItem, SnackBeverageItem, GroceryListItem } from '@/lib/types'

  export const dynamic = 'force-dynamic'

  export default async function GroceryListPage() {
    const sql = getDb()
    let inventoryItems: InventoryItem[]
    let snackItems: SnackBeverageItem[]
    let manual: GroceryListItem[]
    try {
      ;[inventoryItems, snackItems, manual] = await Promise.all([
        sql`SELECT * FROM inventory_items` as Promise<InventoryItem[]>,
        sql`SELECT * FROM snack_beverage_items` as Promise<SnackBeverageItem[]>,
        sql`SELECT * FROM grocery_list_items WHERE purchased = false ORDER BY created_at` as Promise<GroceryListItem[]>,
      ])
    } catch {
      return <p className="p-6 text-red-600">Couldn&apos;t load the grocery list — try again.</p>
    }

    const suggested = deriveSuggestedItems(inventoryItems, snackItems)

    return (
      <main className="max-w-2xl mx-auto p-6">
        <h1 className="text-2xl font-semibold mb-4">Grocery List</h1>
        <AddGroceryItemForm />
        <GroceryListSection suggested={suggested} manual={manual} />
      </main>
    )
  }
  ```

- [ ] **Step 10: Manually verify the full loop**

  ```bash
  npm run dev
  ```
  On `/inventory`, mark an item Low or Out. Visit `/grocery-list` — confirm it appears under "Suggested". Check it off — confirm it disappears from Suggested, and back on `/inventory` its status has reset to Stocked. Add a manual item on `/grocery-list`, confirm it appears under "Added by you", check it off, confirm it disappears from the list.

- [ ] **Step 11: Commit**

  ```bash
  git add src/app/actions/groceryList.ts src/components/GroceryListSection.tsx \
    src/components/AddGroceryItemForm.tsx src/app/grocery-list/page.tsx
  git commit -m "Add grocery list page"
  ```

---

## Task 8: Navigation and Root Redirect

**Files:**
- Create: `src/components/NavBar.tsx`
- Modify: `src/app/layout.tsx`
- Modify: `src/app/page.tsx`

**Interfaces:**
- Consumes: `logout` from `@/app/actions/auth` (Task 4)

- [ ] **Step 1: Create `src/components/NavBar.tsx`**

  ```tsx
  // src/components/NavBar.tsx
  'use client'

  import Link from 'next/link'
  import { usePathname, useRouter } from 'next/navigation'
  import { logout } from '@/app/actions/auth'

  const LINKS = [
    { href: '/inventory', label: 'Inventory' },
    { href: '/snacks', label: 'Snacks & Drinks' },
    { href: '/grocery-list', label: 'Grocery List' },
  ]

  export function NavBar() {
    const pathname = usePathname()
    const router = useRouter()

    if (pathname === '/login') return null

    function handleLogout() {
      logout().then(() => {
        router.push('/login')
        router.refresh()
      })
    }

    return (
      <nav className="border-b border-neutral-200">
        <div className="max-w-2xl mx-auto flex items-center justify-between px-6 py-3">
          <div className="flex gap-4">
            {LINKS.map(link => (
              <Link
                key={link.href}
                href={link.href}
                className={pathname === link.href ? 'font-semibold' : 'text-neutral-500'}
              >
                {link.label}
              </Link>
            ))}
          </div>
          <button
            type="button"
            onClick={handleLogout}
            className="text-sm text-neutral-400 hover:text-neutral-900"
          >
            Log out
          </button>
        </div>
      </nav>
    )
  }
  ```

- [ ] **Step 2: Replace `src/app/layout.tsx`**

  ```tsx
  // src/app/layout.tsx
  import type { Metadata } from 'next'
  import './globals.css'
  import { NavBar } from '@/components/NavBar'

  export const metadata: Metadata = {
    title: 'Kitchen Hub',
  }

  export default function RootLayout({ children }: { children: React.ReactNode }) {
    return (
      <html lang="en">
        <body className="bg-white text-neutral-900 antialiased">
          <NavBar />
          {children}
        </body>
      </html>
    )
  }
  ```

- [ ] **Step 3: Replace `src/app/page.tsx`**

  ```tsx
  // src/app/page.tsx
  import { redirect } from 'next/navigation'

  export default function RootPage() {
    redirect('/inventory')
  }
  ```

- [ ] **Step 4: Manually verify**

  ```bash
  npm run dev
  ```
  Visit `/` while logged in — confirm redirect to `/inventory` with the nav bar visible and the current page bolded. Click "Log out" — confirm redirect to `/login` with no nav bar. Log back in.

- [ ] **Step 5: Commit**

  ```bash
  git add src/components/NavBar.tsx src/app/layout.tsx src/app/page.tsx
  git commit -m "Add navigation, root redirect, and logout"
  ```

---

## Task 9: Deploy to Vercel

**Context:** The Vercel project (`kitchen-hub`, under scope `louis-philippe-wickham`) and its Neon database integration were already provisioned earlier in this plan, with `DATABASE_URL`, `KITCHEN_HUB_PASSCODE`, and `KITCHEN_HUB_SESSION_SECRET` already set for Production, Preview, and Development in Vercel's env store. This task only needs to ship the code.

**Files:** none (deployment only)

- [ ] **Step 1: Push the latest commits**

  ```bash
  git push
  ```

- [ ] **Step 2: Deploy to production**

  ```bash
  npx vercel --prod
  ```
  This uses the existing local project link (`.vercel/project.json`) and the environment variables already configured in Vercel. Expected: a production URL is printed on success.

  If you'd prefer automatic deploys on every `git push` instead of running this manually, connect GitHub under the Vercel project's Settings → Git (this may first require linking your GitHub account under your Vercel account's Login Connections).

- [ ] **Step 3: Verify in production**

  Open the deployed URL on your phone. Confirm you're redirected to `/login`, that the wrong passcode is rejected, and that the correct passcode gets you into `/inventory`. Add a test item, confirm it shows up, then remove it.

- [ ] **Step 4: Share the URL**

  Send the deployed URL and the passcode to your wife so she can add it to her phone's home screen.
