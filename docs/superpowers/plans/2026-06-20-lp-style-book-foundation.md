# LP's Style Book — Foundation (Plan 1 of 3)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Stand up a fully browsable LP's Style Book — project scaffolding, database schema, auth, collection pages, item detail pages, home dashboard, and seed data — so LP and friends can view the wardrobe immediately after this plan is complete.

**Architecture:** Next.js App Router with server components doing all data fetching from Supabase. The site is publicly readable (anyone with the URL can browse). Only the `/admin` route requires LP to be logged in via Supabase email magic link. RLS on all tables enforces this: reads are public, writes require `auth.uid()`.

**Tech Stack:** Next.js 14 (App Router), TypeScript (strict), Tailwind CSS, Supabase (PostgreSQL + Auth + Storage), Vercel, npm, Jest + React Testing Library, tsx (seed script runner).

## Global Constraints

- TypeScript strict mode — `"strict": true` in tsconfig. No `any`.
- Package manager: npm only. Never yarn or pnpm.
- All source files under `src/`. No files at root except config.
- Tailwind palette — use only these tokens: `cream` (`#faf7f2`), `espresso` (`#3d3025`), `tan` (`#b8976a`), `warm` (`#8b6f5e`).
- Fonts: EB Garamond (serif, headings), Inter (sans-serif, body) via `next/font/google`.
- Supabase project name: `lp-style-book`. Storage bucket: `item-photos` (public read).
- Project folder: `projects/lp-style-book/`. GitHub repo: `lp-style-book`.
- No `console.log` left in committed code.
- All user-visible text uses real copy from the spec — no "Lorem ipsum" or placeholder text.

---

## File Map

Files created in this plan:

```
lp-style-book/
├── src/
│   ├── app/
│   │   ├── layout.tsx                     — root layout: fonts, nav, body styles
│   │   ├── globals.css                    — Tailwind directives + base heading font
│   │   ├── page.tsx                       — Home: hero carousel + dashboard stats
│   │   ├── collection/
│   │   │   └── page.tsx                   — My Collection: sidebar + item grid
│   │   ├── items/
│   │   │   └── [id]/
│   │   │       └── page.tsx               — Item detail: photos, score, wear history
│   │   ├── brands/
│   │   │   └── [id]/
│   │   │       └── page.tsx               — Brand detail: profile + owned items
│   │   ├── recommendations/
│   │   │   └── page.tsx                   — Recommendations stub (populated in Plan 3)
│   │   ├── admin/
│   │   │   └── page.tsx                   — Admin home stub (populated in Plan 2)
│   │   ├── login/
│   │   │   └── page.tsx                   — LP email magic-link login
│   │   └── api/
│   │       └── auth/
│   │           └── callback/
│   │               └── route.ts           — Supabase auth code exchange
│   ├── components/
│   │   ├── Nav.tsx                        — top nav: logo + three links
│   │   ├── ItemCard.tsx                   — grid card: photo, name, brand, LP score
│   │   ├── CategorySidebar.tsx            — filter sidebar: categories + brands
│   │   ├── LPScore.tsx                    — score badge and breakdown display
│   │   ├── HeroCarousel.tsx               — cycling hero images (client component)
│   │   └── DashboardStats.tsx             — stat cards for home page
│   ├── lib/
│   │   └── supabase/
│   │       ├── client.ts                  — browser Supabase client
│   │       └── server.ts                  — server Supabase client (cookie-based)
│   └── types/
│       └── index.ts                       — all TS types + computeLPScore + getItemLPScore
├── supabase/
│   ├── schema.sql                         — full DDL: tables, RLS, seed categories
│   └── seed.ts                            — seed script: brands, items, reviews, wear records
├── __tests__/
│   ├── types/
│   │   └── lp-score.test.ts               — unit tests for computeLPScore + getItemLPScore
│   └── components/
│       ├── ItemCard.test.tsx
│       ├── LPScore.test.tsx
│       └── CategorySidebar.test.tsx
├── middleware.ts                           — admin auth guard + auth session refresh
├── jest.config.ts
├── jest.setup.ts
├── tailwind.config.ts
├── next.config.ts
├── tsconfig.json
├── .env.local                             — never committed
├── .env.local.example                     — committed, values redacted
├── .gitignore
└── README.md
```

---

## Task 1: Project Setup

**Files:**
- Create: `lp-style-book/` (all root config files)
- Create: `.env.local.example`
- Create: `.gitignore`
- Create: `README.md`

**Interfaces:**
- Produces: A running Next.js 14 dev server at `http://localhost:3000` with warm minimal colours applied.

- [ ] **Step 1: Create the GitHub repo**

  Go to github.com, click New Repository, name it `lp-style-book`, set it to Public, do NOT add a README or .gitignore (we'll add them ourselves). Copy the repo URL (e.g. `https://github.com/lpwickham-ship-it/lp-style-book.git`).

- [ ] **Step 2: Scaffold the Next.js app**

  Run from inside `projects/`:
  ```bash
  npx create-next-app@14 lp-style-book \
    --typescript \
    --tailwind \
    --eslint \
    --app \
    --src-dir \
    --import-alias "@/*" \
    --no-git
  cd lp-style-book
  ```
  Expected: project created, `npm run dev` starts without errors.

- [ ] **Step 3: Replace `tailwind.config.ts` with the warm minimal palette**

  ```typescript
  // tailwind.config.ts
  import type { Config } from 'tailwindcss'

  const config: Config = {
    content: ['./src/**/*.{js,ts,jsx,tsx,mdx}'],
    theme: {
      extend: {
        colors: {
          cream: '#faf7f2',
          espresso: '#3d3025',
          tan: '#b8976a',
          warm: '#8b6f5e',
        },
        fontFamily: {
          serif: ['var(--font-garamond)', 'Georgia', 'serif'],
          sans: ['var(--font-inter)', 'system-ui', 'sans-serif'],
        },
      },
    },
    plugins: [],
  }

  export default config
  ```

- [ ] **Step 4: Replace `src/app/globals.css`**

  ```css
  @tailwind base;
  @tailwind components;
  @tailwind utilities;

  @layer base {
    h1, h2, h3, h4, h5, h6 {
      font-family: var(--font-garamond), Georgia, serif;
    }
  }
  ```

- [ ] **Step 5: Install Supabase packages and testing dependencies**

  ```bash
  npm install @supabase/supabase-js @supabase/ssr
  npm install --save-dev jest @testing-library/react @testing-library/jest-dom \
    jest-environment-jsdom @types/jest tsx
  ```

- [ ] **Step 6: Create `jest.config.ts`**

  ```typescript
  // jest.config.ts
  import type { Config } from 'jest'
  import nextJest from 'next/jest.js'

  const createJestConfig = nextJest({ dir: './' })

  const config: Config = {
    coverageProvider: 'v8',
    testEnvironment: 'jsdom',
    setupFilesAfterFramework: ['<rootDir>/jest.setup.ts'],
  }

  export default createJestConfig(config)
  ```

- [ ] **Step 7: Create `jest.setup.ts`**

  ```typescript
  // jest.setup.ts
  import '@testing-library/jest-dom'
  ```

- [ ] **Step 8: Add test script to `package.json`**

  In `package.json`, add to the `"scripts"` block:
  ```json
  "test": "jest",
  "test:watch": "jest --watch"
  ```

- [ ] **Step 9: Create `.env.local.example`**

  ```
  NEXT_PUBLIC_SUPABASE_URL=your-supabase-project-url
  NEXT_PUBLIC_SUPABASE_ANON_KEY=your-supabase-anon-key
  ```

- [ ] **Step 10: Update `.gitignore`**

  Add these lines to the bottom of the existing `.gitignore`:
  ```
  .env.local
  .env*.local
  ```

- [ ] **Step 11: Create `README.md`**

  ```markdown
  # LP's Style Book

  Personal wardrobe intelligence platform. Track owned clothing, improve purchase decisions, and learn style preferences.

  ## Tech Stack

  Next.js 14 · TypeScript · Tailwind CSS · Supabase · Vercel

  ## Running Locally

  1. Copy `.env.local.example` to `.env.local` and fill in your Supabase credentials.
  2. Run `npm install`
  3. Run `npm run dev`
  4. Open [http://localhost:3000](http://localhost:3000)

  ## Seeding the Database

  After setting up the schema in Supabase:
  ```bash
  npx tsx supabase/seed.ts
  ```
  ```

- [ ] **Step 12: Initialise git and connect to GitHub**

  ```bash
  git init
  git add .
  git commit -m "Scaffold Next.js project with warm minimal config"
  git branch -M main
  git remote add origin https://github.com/lpwickham-ship-it/lp-style-book.git
  git push -u origin main
  ```

- [ ] **Step 13: Verify dev server starts**

  ```bash
  npm run dev
  ```
  Expected: `ready - started server on 0.0.0.0:3000`.

---

## Task 2: Supabase Project + Database Schema

**Files:**
- Create: `supabase/schema.sql`

**Interfaces:**
- Produces: All database tables created in Supabase with RLS enabled. Categories and subcategories seeded.

- [ ] **Step 1: Create a Supabase project**

  Go to supabase.com → New Project → name it `lp-style-book` → pick a strong database password → choose a region close to you → Create Project.

  Wait ~2 minutes for the project to spin up.

- [ ] **Step 2: Get your credentials**

  In the Supabase dashboard → Settings → API. Copy:
  - **Project URL** → paste as `NEXT_PUBLIC_SUPABASE_URL` in `.env.local`
  - **anon public key** → paste as `NEXT_PUBLIC_SUPABASE_ANON_KEY` in `.env.local`

- [ ] **Step 3: Create a Storage bucket**

  In Supabase dashboard → Storage → New Bucket → name: `item-photos` → check "Public bucket" → Create.

- [ ] **Step 4: Save the schema SQL to `supabase/schema.sql`**

  Create the directory and file:
  ```bash
  mkdir supabase
  ```

  Write to `supabase/schema.sql`:
  ```sql
  -- items belong to brands and subcategories
  -- subcategories belong to categories
  -- reviews, wear_records, and photos belong to items

  create table brands (
    id uuid primary key default gen_random_uuid(),
    name text not null unique,
    country text,
    notes text,
    created_at timestamptz default now()
  );

  create table categories (
    id uuid primary key default gen_random_uuid(),
    name text not null unique,
    slug text not null unique
  );

  create table subcategories (
    id uuid primary key default gen_random_uuid(),
    category_id uuid not null references categories(id),
    name text not null,
    slug text not null unique
  );

  create table items (
    id uuid primary key default gen_random_uuid(),
    name text not null,
    brand_id uuid references brands(id),
    subcategory_id uuid references subcategories(id),
    description text,
    material text,
    purchase_price numeric(10, 2),
    purchase_date date,
    purchase_location text,
    source_url text,
    status text not null default 'owned' check (status in ('owned', 'archived')),
    in_collection boolean not null default false,
    in_wishlist boolean not null default false,
    is_recommendation boolean not null default false,
    created_at timestamptz default now()
  );

  create table item_photos (
    id uuid primary key default gen_random_uuid(),
    item_id uuid not null references items(id) on delete cascade,
    storage_path text not null,
    is_primary boolean not null default false,
    created_at timestamptz default now()
  );

  create table lp_reviews (
    id uuid primary key default gen_random_uuid(),
    item_id uuid not null references items(id) on delete cascade,
    fit smallint not null check (fit between 1 and 10),
    comfort smallint not null check (comfort between 1 and 10),
    quality smallint not null check (quality between 1 and 10),
    versatility smallint not null check (versatility between 1 and 10),
    value smallint not null check (value between 1 and 10),
    notes text,
    reviewed_at timestamptz default now()
  );

  create table wear_records (
    id uuid primary key default gen_random_uuid(),
    item_id uuid not null references items(id) on delete cascade,
    worn_on date not null,
    season text check (season in ('Spring', 'Summer', 'Autumn', 'Winter')),
    occasion text check (occasion in ('Casual', 'Smart Casual', 'Work', 'Formal', 'Sport')),
    created_at timestamptz default now()
  );

  create table wishlist_items (
    id uuid primary key default gen_random_uuid(),
    item_id uuid not null references items(id) on delete cascade,
    status text not null default 'Wishlist'
      check (status in ('Wishlist', 'Researching', 'Considering', 'Purchased', 'Archived')),
    notes text,
    alternatives text,
    interest_score smallint check (interest_score between 1 and 10),
    created_at timestamptz default now()
  );

  create table failed_purchases (
    id uuid primary key default gen_random_uuid(),
    item_id uuid not null references items(id) on delete cascade,
    reason text not null check (reason in (
      'poor fit', 'poor quality', 'poor value', 'rarely worn',
      'doesn''t suit style', 'uncomfortable', 'impulse purchase', 'other'
    )),
    notes text,
    created_at timestamptz default now()
  );

  create table recommendation_entries (
    id uuid primary key default gen_random_uuid(),
    item_id uuid references items(id) on delete set null,
    written_take text not null,
    published_at timestamptz default now()
  );

  create table item_pairings (
    id uuid primary key default gen_random_uuid(),
    item_id uuid not null references items(id) on delete cascade,
    paired_item_id uuid not null references items(id) on delete cascade,
    note text,
    unique(item_id, paired_item_id),
    check (item_id != paired_item_id)
  );

  create table hero_images (
    id uuid primary key default gen_random_uuid(),
    storage_path text not null,
    display_order integer not null default 0,
    created_at timestamptz default now()
  );

  -- RLS: all tables readable by anyone, writable only by authenticated users
  alter table brands enable row level security;
  alter table categories enable row level security;
  alter table subcategories enable row level security;
  alter table items enable row level security;
  alter table item_photos enable row level security;
  alter table lp_reviews enable row level security;
  alter table wear_records enable row level security;
  alter table wishlist_items enable row level security;
  alter table failed_purchases enable row level security;
  alter table recommendation_entries enable row level security;
  alter table item_pairings enable row level security;
  alter table hero_images enable row level security;

  create policy "public read" on brands for select using (true);
  create policy "public read" on categories for select using (true);
  create policy "public read" on subcategories for select using (true);
  create policy "public read" on items for select using (true);
  create policy "public read" on item_photos for select using (true);
  create policy "public read" on lp_reviews for select using (true);
  create policy "public read" on wear_records for select using (true);
  create policy "public read" on wishlist_items for select using (true);
  create policy "public read" on failed_purchases for select using (true);
  create policy "public read" on recommendation_entries for select using (true);
  create policy "public read" on item_pairings for select using (true);
  create policy "public read" on hero_images for select using (true);

  create policy "auth write" on brands for all using (auth.uid() is not null);
  create policy "auth write" on categories for all using (auth.uid() is not null);
  create policy "auth write" on subcategories for all using (auth.uid() is not null);
  create policy "auth write" on items for all using (auth.uid() is not null);
  create policy "auth write" on item_photos for all using (auth.uid() is not null);
  create policy "auth write" on lp_reviews for all using (auth.uid() is not null);
  create policy "auth write" on wear_records for all using (auth.uid() is not null);
  create policy "auth write" on wishlist_items for all using (auth.uid() is not null);
  create policy "auth write" on failed_purchases for all using (auth.uid() is not null);
  create policy "auth write" on recommendation_entries for all using (auth.uid() is not null);
  create policy "auth write" on item_pairings for all using (auth.uid() is not null);
  create policy "auth write" on hero_images for all using (auth.uid() is not null);

  -- Seed static reference data
  insert into categories (name, slug) values
    ('Clothing', 'clothing'),
    ('Accessories', 'accessories'),
    ('Shoes', 'shoes');

  insert into subcategories (category_id, name, slug) values
    ((select id from categories where slug = 'clothing'), 'Tops', 'tops'),
    ((select id from categories where slug = 'clothing'), 'Shirts', 'shirts'),
    ((select id from categories where slug = 'clothing'), 'Bottoms', 'bottoms'),
    ((select id from categories where slug = 'clothing'), 'Outerwear', 'outerwear'),
    ((select id from categories where slug = 'clothing'), 'Knitwear', 'knitwear'),
    ((select id from categories where slug = 'accessories'), 'Watches', 'watches'),
    ((select id from categories where slug = 'accessories'), 'Belts', 'belts'),
    ((select id from categories where slug = 'accessories'), 'Bags', 'bags'),
    ((select id from categories where slug = 'accessories'), 'Scarves', 'scarves'),
    ((select id from categories where slug = 'accessories'), 'Eyewear', 'eyewear'),
    ((select id from categories where slug = 'accessories'), 'Ties', 'ties'),
    ((select id from categories where slug = 'shoes'), 'Trainers', 'trainers'),
    ((select id from categories where slug = 'shoes'), 'Boots', 'boots'),
    ((select id from categories where slug = 'shoes'), 'Loafers', 'loafers'),
    ((select id from categories where slug = 'shoes'), 'Derby', 'derby'),
    ((select id from categories where slug = 'shoes'), 'Oxford', 'oxford');
  ```

- [ ] **Step 5: Run the schema in Supabase**

  In the Supabase dashboard → SQL Editor → New Query → paste the entire contents of `supabase/schema.sql` → Run.

  Expected: "Success. No rows returned." No error messages.

- [ ] **Step 6: Verify tables exist**

  In Supabase dashboard → Table Editor. You should see: brands, categories, subcategories, items, item_photos, lp_reviews, wear_records, wishlist_items, failed_purchases, recommendation_entries, item_pairings, hero_images.

  Click `categories` — it should show 3 rows (Clothing, Accessories, Shoes).

- [ ] **Step 7: Commit**

  ```bash
  git add supabase/schema.sql .env.local.example
  git commit -m "Add database schema and Supabase project setup"
  ```

---

## Task 3: TypeScript Types + Supabase Clients

**Files:**
- Create: `src/types/index.ts`
- Create: `src/lib/supabase/client.ts`
- Create: `src/lib/supabase/server.ts`
- Create: `__tests__/types/lp-score.test.ts`

**Interfaces:**
- Produces:
  - `computeLPScore(review)` → `number` (1.0–10.0, one decimal place)
  - `getItemLPScore(reviews)` → `number | null`
  - `createClient()` (browser) → Supabase browser client
  - `createClient()` (server) → Supabase server client (async, cookie-based)
  - All shared types exported from `@/types`

- [ ] **Step 1: Write the failing tests for `computeLPScore` and `getItemLPScore`**

  ```typescript
  // __tests__/types/lp-score.test.ts
  import { computeLPScore, getItemLPScore } from '@/types'
  import type { LPReview } from '@/types'

  const makeReview = (overrides: Partial<LPReview> = {}): LPReview => ({
    id: 'r1',
    item_id: 'i1',
    fit: 8,
    comfort: 7,
    quality: 9,
    versatility: 6,
    value: 7,
    notes: null,
    reviewed_at: '2026-06-20T00:00:00Z',
    ...overrides,
  })

  describe('computeLPScore', () => {
    it('averages five dimensions to one decimal place', () => {
      // (8 + 7 + 9 + 6 + 7) / 5 = 37 / 5 = 7.4
      expect(computeLPScore(makeReview())).toBe(7.4)
    })

    it('returns 10.0 when all dimensions are 10', () => {
      expect(computeLPScore(makeReview({ fit: 10, comfort: 10, quality: 10, versatility: 10, value: 10 }))).toBe(10)
    })

    it('returns 1.0 when all dimensions are 1', () => {
      expect(computeLPScore(makeReview({ fit: 1, comfort: 1, quality: 1, versatility: 1, value: 1 }))).toBe(1)
    })

    it('rounds to one decimal place', () => {
      // (7 + 7 + 7 + 7 + 8) / 5 = 36 / 5 = 7.2
      expect(computeLPScore(makeReview({ fit: 7, comfort: 7, quality: 7, versatility: 7, value: 8 }))).toBe(7.2)
    })
  })

  describe('getItemLPScore', () => {
    it('returns null for an empty reviews array', () => {
      expect(getItemLPScore([])).toBeNull()
    })

    it('returns the score of the only review', () => {
      expect(getItemLPScore([makeReview()])).toBe(7.4)
    })

    it('returns the score of the most recent review when multiple exist', () => {
      const older = makeReview({ fit: 5, comfort: 5, quality: 5, versatility: 5, value: 5, reviewed_at: '2025-01-01T00:00:00Z' })
      const newer = makeReview({ fit: 9, comfort: 9, quality: 9, versatility: 9, value: 9, reviewed_at: '2026-06-01T00:00:00Z' })
      expect(getItemLPScore([older, newer])).toBe(9)
    })
  })
  ```

- [ ] **Step 2: Run the tests to confirm they fail**

  ```bash
  npm test -- --testPathPattern="lp-score"
  ```
  Expected: FAIL — `Cannot find module '@/types'`

- [ ] **Step 3: Create `src/types/index.ts`**

  ```typescript
  // src/types/index.ts

  export type Brand = {
    id: string
    name: string
    country: string | null
    notes: string | null
    created_at: string
  }

  export type Category = {
    id: string
    name: string
    slug: string
  }

  export type Subcategory = {
    id: string
    category_id: string
    name: string
    slug: string
    category?: Category
  }

  export type Item = {
    id: string
    name: string
    brand_id: string | null
    subcategory_id: string | null
    description: string | null
    material: string | null
    purchase_price: number | null
    purchase_date: string | null
    purchase_location: string | null
    source_url: string | null
    status: 'owned' | 'archived'
    in_collection: boolean
    in_wishlist: boolean
    is_recommendation: boolean
    created_at: string
  }

  export type ItemPhoto = {
    id: string
    item_id: string
    storage_path: string
    is_primary: boolean
    created_at: string
  }

  export type LPReview = {
    id: string
    item_id: string
    fit: number
    comfort: number
    quality: number
    versatility: number
    value: number
    notes: string | null
    reviewed_at: string
  }

  export type WearRecord = {
    id: string
    item_id: string
    worn_on: string
    season: 'Spring' | 'Summer' | 'Autumn' | 'Winter' | null
    occasion: 'Casual' | 'Smart Casual' | 'Work' | 'Formal' | 'Sport' | null
    created_at: string
  }

  export type WishlistItem = {
    id: string
    item_id: string
    status: 'Wishlist' | 'Researching' | 'Considering' | 'Purchased' | 'Archived'
    notes: string | null
    alternatives: string | null
    interest_score: number | null
    created_at: string
  }

  export type FailedPurchase = {
    id: string
    item_id: string
    reason: 'poor fit' | 'poor quality' | 'poor value' | 'rarely worn' | "doesn't suit style" | 'uncomfortable' | 'impulse purchase' | 'other'
    notes: string | null
    created_at: string
  }

  export type RecommendationEntry = {
    id: string
    item_id: string | null
    written_take: string
    published_at: string
  }

  export type ItemPairing = {
    id: string
    item_id: string
    paired_item_id: string
    note: string | null
  }

  export type HeroImage = {
    id: string
    storage_path: string
    display_order: number
    created_at: string
  }

  // Enriched types — used by list views and detail pages
  export type ItemWithRelations = Item & {
    brand: Brand | null
    subcategory: (Subcategory & { category: Category }) | null
    photos: ItemPhoto[]
    lp_score: number | null
    wear_count: number
  }

  export type ItemDetail = ItemWithRelations & {
    reviews: LPReview[]
    wear_records: WearRecord[]
    pairings: Array<ItemPairing & { paired_item: ItemWithRelations }>
  }

  export type BrandWithStats = Brand & {
    items: ItemWithRelations[]
    average_lp_score: number | null
    total_wear_count: number
  }

  // Averages five review dimensions, rounded to one decimal place.
  export function computeLPScore(
    review: Pick<LPReview, 'fit' | 'comfort' | 'quality' | 'versatility' | 'value'>
  ): number {
    const sum = review.fit + review.comfort + review.quality + review.versatility + review.value
    return Math.round((sum / 5) * 10) / 10
  }

  // Returns the LP score of the most recent review, or null if none exist.
  export function getItemLPScore(reviews: LPReview[]): number | null {
    if (reviews.length === 0) return null
    const latest = [...reviews].sort(
      (a, b) => new Date(b.reviewed_at).getTime() - new Date(a.reviewed_at).getTime()
    )[0]
    return computeLPScore(latest)
  }
  ```

- [ ] **Step 4: Run the tests to confirm they pass**

  ```bash
  npm test -- --testPathPattern="lp-score"
  ```
  Expected: PASS — 7 tests passing.

- [ ] **Step 5: Create `src/lib/supabase/client.ts`**

  ```typescript
  // src/lib/supabase/client.ts
  import { createBrowserClient } from '@supabase/ssr'

  export function createClient() {
    return createBrowserClient(
      process.env.NEXT_PUBLIC_SUPABASE_URL!,
      process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
    )
  }
  ```

- [ ] **Step 6: Create `src/lib/supabase/server.ts`**

  ```typescript
  // src/lib/supabase/server.ts
  import { createServerClient } from '@supabase/ssr'
  import { cookies } from 'next/headers'

  export async function createClient() {
    const cookieStore = await cookies()
    return createServerClient(
      process.env.NEXT_PUBLIC_SUPABASE_URL!,
      process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
      {
        cookies: {
          getAll() {
            return cookieStore.getAll()
          },
          setAll(cookiesToSet) {
            try {
              cookiesToSet.forEach(({ name, value, options }) =>
                cookieStore.set(name, value, options)
              )
            } catch {
              // Called from a server component — cookie setting is a no-op
            }
          },
        },
      }
    )
  }
  ```

- [ ] **Step 7: Commit**

  ```bash
  git add src/types/ src/lib/ __tests__/ jest.config.ts jest.setup.ts
  git commit -m "Add TypeScript types, Supabase clients, and LP score unit tests"
  ```

---

## Task 4: Auth — Login Page + Admin Guard Middleware

**Files:**
- Create: `middleware.ts`
- Create: `src/app/login/page.tsx`
- Create: `src/app/api/auth/callback/route.ts`
- Create: `src/app/admin/page.tsx`

**Interfaces:**
- Produces:
  - `/login` — email magic-link form; on submit sends OTP via Supabase Auth
  - `/api/auth/callback` — exchanges Supabase code for session, redirects to `/admin`
  - `/admin` — only reachable when LP is authenticated; redirects to `/login` otherwise
  - All other routes — publicly accessible

- [ ] **Step 1: Create `middleware.ts`**

  ```typescript
  // middleware.ts
  import { NextResponse } from 'next/server'
  import type { NextRequest } from 'next/server'
  import { createServerClient } from '@supabase/ssr'

  export async function middleware(request: NextRequest) {
    const response = NextResponse.next({ request })

    const supabase = createServerClient(
      process.env.NEXT_PUBLIC_SUPABASE_URL!,
      process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
      {
        cookies: {
          getAll() {
            return request.cookies.getAll()
          },
          setAll(cookiesToSet) {
            cookiesToSet.forEach(({ name, value, options }) =>
              response.cookies.set(name, value, options)
            )
          },
        },
      }
    )

    // Refresh the session on every request (keeps it alive)
    const { data: { user } } = await supabase.auth.getUser()

    const { pathname } = request.nextUrl

    // Guard: /admin requires an authenticated session
    if (pathname.startsWith('/admin') && !user) {
      return NextResponse.redirect(new URL('/login', request.url))
    }

    // Guard: authenticated users don't need the login page
    if (pathname === '/login' && user) {
      return NextResponse.redirect(new URL('/admin', request.url))
    }

    return response
  }

  export const config = {
    matcher: [
      '/((?!_next/static|_next/image|favicon.ico|.*\\.(?:svg|png|jpg|jpeg|gif|webp)$).*)',
    ],
  }
  ```

- [ ] **Step 2: Create `src/app/api/auth/callback/route.ts`**

  ```typescript
  // src/app/api/auth/callback/route.ts
  import { NextResponse } from 'next/server'
  import { createClient } from '@/lib/supabase/server'

  export async function GET(request: Request) {
    const { searchParams, origin } = new URL(request.url)
    const code = searchParams.get('code')

    if (code) {
      const supabase = await createClient()
      await supabase.auth.exchangeCodeForSession(code)
    }

    return NextResponse.redirect(`${origin}/admin`)
  }
  ```

- [ ] **Step 3: Create `src/app/login/page.tsx`**

  ```tsx
  // src/app/login/page.tsx
  'use client'
  import { useState } from 'react'
  import { createClient } from '@/lib/supabase/client'

  export default function LoginPage() {
    const [email, setEmail] = useState('')
    const [sent, setSent] = useState(false)
    const [loading, setLoading] = useState(false)

    async function handleSubmit(e: React.FormEvent) {
      e.preventDefault()
      setLoading(true)
      const supabase = createClient()
      await supabase.auth.signInWithOtp({
        email,
        options: {
          emailRedirectTo: `${window.location.origin}/api/auth/callback`,
        },
      })
      setSent(true)
      setLoading(false)
    }

    return (
      <div className="min-h-screen bg-cream flex items-center justify-center px-6">
        <div className="max-w-sm w-full">
          <h1 className="font-serif text-3xl text-espresso mb-1 tracking-tight">
            LP's Style Book
          </h1>
          <p className="text-warm text-xs tracking-widest uppercase mb-10">Admin Access</p>

          {sent ? (
            <p className="text-espresso text-sm leading-relaxed">
              Check your inbox — a login link is on its way to{' '}
              <strong>{email}</strong>.
            </p>
          ) : (
            <form onSubmit={handleSubmit} className="space-y-4">
              <input
                type="email"
                value={email}
                onChange={e => setEmail(e.target.value)}
                placeholder="your@email.com"
                required
                className="w-full border border-warm/30 bg-white px-4 py-3 text-espresso text-sm focus:outline-none focus:border-tan placeholder:text-warm/40 transition-colors"
              />
              <button
                type="submit"
                disabled={loading}
                className="w-full bg-espresso text-cream py-3 text-xs tracking-widest uppercase hover:bg-espresso/90 disabled:opacity-50 transition-colors"
              >
                {loading ? 'Sending…' : 'Send Login Link'}
              </button>
            </form>
          )}
        </div>
      </div>
    )
  }
  ```

- [ ] **Step 4: Create the admin stub `src/app/admin/page.tsx`**

  ```tsx
  // src/app/admin/page.tsx
  import { createClient } from '@/lib/supabase/server'
  import { redirect } from 'next/navigation'

  export default async function AdminPage() {
    const supabase = await createClient()
    const { data: { user } } = await supabase.auth.getUser()
    if (!user) redirect('/login')

    return (
      <div className="max-w-4xl mx-auto px-6 py-10">
        <h1 className="font-serif text-3xl text-espresso mb-2">Admin</h1>
        <p className="text-warm text-sm mb-8">Logged in as {user.email}</p>
        <p className="text-warm text-sm">
          Product import and item management arrive in Plan 2.
        </p>
      </div>
    )
  }
  ```

- [ ] **Step 5: Manually test the login flow**

  1. Start the dev server: `npm run dev`
  2. Visit `http://localhost:3000/admin` — it should redirect to `/login`.
  3. Enter your email and submit the form — you should see "Check your inbox…".
  4. Open the email and click the login link — it should redirect to `/admin` and show your email address.
  5. Visit `http://localhost:3000` — it should load without redirecting (publicly accessible).

- [ ] **Step 6: Commit**

  ```bash
  git add middleware.ts src/app/login/ src/app/api/ src/app/admin/
  git commit -m "Add Supabase auth: email magic-link login and admin route guard"
  ```

---

## Task 5: Global Layout + Navigation

**Files:**
- Modify: `src/app/layout.tsx`
- Create: `src/components/Nav.tsx`
- Create: `__tests__/components/Nav.test.tsx`

**Interfaces:**
- Produces: `<Nav />` — renders site name + three navigation links. Consumed by `layout.tsx`.

- [ ] **Step 1: Write the failing test for `Nav`**

  ```tsx
  // __tests__/components/Nav.test.tsx
  import { render, screen } from '@testing-library/react'
  import Nav from '@/components/Nav'

  jest.mock('next/link', () => {
    return function MockLink({ href, children }: { href: string; children: React.ReactNode }) {
      return <a href={href}>{children}</a>
    }
  })

  describe('Nav', () => {
    it('renders the site name', () => {
      render(<Nav />)
      expect(screen.getByText("LP's Style Book")).toBeInTheDocument()
    })

    it('renders all three navigation links', () => {
      render(<Nav />)
      expect(screen.getByRole('link', { name: /home/i })).toHaveAttribute('href', '/')
      expect(screen.getByRole('link', { name: /recommendations/i })).toHaveAttribute('href', '/recommendations')
      expect(screen.getByRole('link', { name: /my collection/i })).toHaveAttribute('href', '/collection')
    })
  })
  ```

- [ ] **Step 2: Run the test to confirm it fails**

  ```bash
  npm test -- --testPathPattern="Nav"
  ```
  Expected: FAIL — `Cannot find module '@/components/Nav'`

- [ ] **Step 3: Create `src/components/Nav.tsx`**

  ```tsx
  // src/components/Nav.tsx
  import Link from 'next/link'

  export default function Nav() {
    return (
      <nav className="border-b border-espresso/10 bg-cream sticky top-0 z-50">
        <div className="max-w-6xl mx-auto px-6 py-4 flex items-center justify-between">
          <Link
            href="/"
            className="font-serif text-espresso tracking-widest uppercase text-sm"
          >
            LP's Style Book
          </Link>
          <div className="flex gap-8">
            <Link
              href="/"
              className="text-warm hover:text-espresso text-xs tracking-widest uppercase transition-colors"
            >
              Home
            </Link>
            <Link
              href="/recommendations"
              className="text-warm hover:text-espresso text-xs tracking-widest uppercase transition-colors"
            >
              Recommendations
            </Link>
            <Link
              href="/collection"
              className="text-warm hover:text-espresso text-xs tracking-widest uppercase transition-colors"
            >
              My Collection
            </Link>
          </div>
        </div>
      </nav>
    )
  }
  ```

- [ ] **Step 4: Run the test to confirm it passes**

  ```bash
  npm test -- --testPathPattern="Nav"
  ```
  Expected: PASS — 2 tests passing.

- [ ] **Step 5: Replace `src/app/layout.tsx`**

  ```tsx
  // src/app/layout.tsx
  import type { Metadata } from 'next'
  import { EB_Garamond, Inter } from 'next/font/google'
  import Nav from '@/components/Nav'
  import './globals.css'

  const garamond = EB_Garamond({
    subsets: ['latin'],
    variable: '--font-garamond',
    weight: ['400', '500', '600'],
  })

  const inter = Inter({
    subsets: ['latin'],
    variable: '--font-inter',
  })

  export const metadata: Metadata = {
    title: "LP's Style Book",
    description: 'Personal wardrobe intelligence',
  }

  export default function RootLayout({ children }: { children: React.ReactNode }) {
    return (
      <html lang="en" className={`${garamond.variable} ${inter.variable}`}>
        <body className="bg-cream text-espresso font-sans antialiased min-h-screen">
          <Nav />
          <main>{children}</main>
        </body>
      </html>
    )
  }
  ```

- [ ] **Step 6: Verify in browser**

  Visit `http://localhost:3000`. You should see the nav bar with "LP's Style Book" on the left and three links on the right. Background should be warm off-white (`#faf7f2`).

- [ ] **Step 7: Commit**

  ```bash
  git add src/app/layout.tsx src/components/Nav.tsx __tests__/components/Nav.test.tsx
  git commit -m "Add global layout and navigation with warm minimal design"
  ```

---

## Task 6: ItemCard + CategorySidebar Components

**Files:**
- Create: `src/components/ItemCard.tsx`
- Create: `src/components/CategorySidebar.tsx`
- Create: `__tests__/components/ItemCard.test.tsx`
- Create: `__tests__/components/CategorySidebar.test.tsx`

**Interfaces:**
- Consumes:
  - `ItemCard` receives `item: ItemWithRelations` from `@/types`
  - `CategorySidebar` receives `categories: Array<Category & { subcategories: Subcategory[] }>`, `brands: Brand[]`, `selectedCategory?: string`, `selectedSubcategory?: string`, `selectedBrand?: string`
- Produces:
  - `<ItemCard item={item} />` — card with photo, item name, brand name, LP score badge
  - `<CategorySidebar ... />` — collapsible category/brand filter links

- [ ] **Step 1: Write the failing tests for `ItemCard`**

  ```tsx
  // __tests__/components/ItemCard.test.tsx
  import { render, screen } from '@testing-library/react'
  import ItemCard from '@/components/ItemCard'
  import type { ItemWithRelations } from '@/types'

  jest.mock('next/link', () => {
    return function MockLink({ href, children }: { href: string; children: React.ReactNode }) {
      return <a href={href}>{children}</a>
    }
  })
  jest.mock('next/image', () => {
    return function MockImage({ alt }: { alt: string }) {
      return <img alt={alt} />
    }
  })

  const mockItem: ItemWithRelations = {
    id: 'item-1',
    name: 'Navy Wool Tie',
    brand_id: 'brand-1',
    subcategory_id: 'sub-1',
    description: null,
    material: 'Wool',
    purchase_price: 120,
    purchase_date: null,
    purchase_location: null,
    source_url: null,
    status: 'owned',
    in_collection: true,
    in_wishlist: false,
    is_recommendation: false,
    created_at: '2026-06-01T00:00:00Z',
    brand: { id: 'brand-1', name: "Drake's", country: 'UK', notes: null, created_at: '2026-01-01T00:00:00Z' },
    subcategory: { id: 'sub-1', category_id: 'cat-1', name: 'Ties', slug: 'ties', category: { id: 'cat-1', name: 'Accessories', slug: 'accessories' } },
    photos: [],
    lp_score: 8.2,
    wear_count: 14,
  }

  describe('ItemCard', () => {
    it('renders the item name', () => {
      render(<ItemCard item={mockItem} />)
      expect(screen.getByText('Navy Wool Tie')).toBeInTheDocument()
    })

    it('renders the brand name', () => {
      render(<ItemCard item={mockItem} />)
      expect(screen.getByText("Drake's")).toBeInTheDocument()
    })

    it('renders the LP score', () => {
      render(<ItemCard item={mockItem} />)
      expect(screen.getByText('8.2')).toBeInTheDocument()
    })

    it('links to the item detail page', () => {
      render(<ItemCard item={mockItem} />)
      expect(screen.getByRole('link')).toHaveAttribute('href', '/items/item-1')
    })

    it('shows a placeholder when there are no photos', () => {
      render(<ItemCard item={mockItem} />)
      expect(screen.getByText('No photo')).toBeInTheDocument()
    })
  })
  ```

- [ ] **Step 2: Run tests to confirm they fail**

  ```bash
  npm test -- --testPathPattern="ItemCard"
  ```
  Expected: FAIL — `Cannot find module '@/components/ItemCard'`

- [ ] **Step 3: Create `src/components/ItemCard.tsx`**

  ```tsx
  // src/components/ItemCard.tsx
  import Link from 'next/link'
  import Image from 'next/image'
  import type { ItemWithRelations } from '@/types'

  export default function ItemCard({ item }: { item: ItemWithRelations }) {
    const primaryPhoto = item.photos.find(p => p.is_primary) ?? item.photos[0]
    const photoUrl = primaryPhoto
      ? `${process.env.NEXT_PUBLIC_SUPABASE_URL}/storage/v1/object/public/item-photos/${primaryPhoto.storage_path}`
      : null

    return (
      <Link href={`/items/${item.id}`} className="group block">
        <div className="aspect-[3/4] bg-warm/10 relative overflow-hidden mb-3">
          {photoUrl ? (
            <Image
              src={photoUrl}
              alt={item.name}
              fill
              className="object-cover group-hover:scale-105 transition-transform duration-500"
            />
          ) : (
            <div className="absolute inset-0 flex items-center justify-center">
              <span className="text-warm/40 text-xs tracking-widest uppercase">No photo</span>
            </div>
          )}
          {item.lp_score !== null && (
            <div className="absolute top-3 right-3 bg-espresso text-cream text-xs px-2 py-1 tracking-wide">
              {item.lp_score}
            </div>
          )}
        </div>
        <div>
          <p className="text-espresso text-sm font-medium leading-snug">{item.name}</p>
          {item.brand && (
            <p className="text-warm text-xs tracking-wide mt-0.5">{item.brand.name}</p>
          )}
        </div>
      </Link>
    )
  }
  ```

- [ ] **Step 4: Run the `ItemCard` tests to confirm they pass**

  ```bash
  npm test -- --testPathPattern="ItemCard"
  ```
  Expected: PASS — 5 tests passing.

- [ ] **Step 5: Write the failing tests for `CategorySidebar`**

  ```tsx
  // __tests__/components/CategorySidebar.test.tsx
  import { render, screen } from '@testing-library/react'
  import CategorySidebar from '@/components/CategorySidebar'
  import type { Category, Subcategory, Brand } from '@/types'

  jest.mock('next/link', () => {
    return function MockLink({ href, children }: { href: string; children: React.ReactNode }) {
      return <a href={href}>{children}</a>
    }
  })

  type CategoryWithSubs = Category & { subcategories: Subcategory[] }

  const categories: CategoryWithSubs[] = [
    {
      id: 'cat-1', name: 'Clothing', slug: 'clothing',
      subcategories: [
        { id: 's1', category_id: 'cat-1', name: 'Tops', slug: 'tops' },
        { id: 's2', category_id: 'cat-1', name: 'Shirts', slug: 'shirts' },
      ],
    },
    {
      id: 'cat-2', name: 'Shoes', slug: 'shoes',
      subcategories: [
        { id: 's3', category_id: 'cat-2', name: 'Boots', slug: 'boots' },
      ],
    },
  ]

  const brands: Brand[] = [
    { id: 'b1', name: "Drake's", country: 'UK', notes: null, created_at: '2026-01-01T00:00:00Z' },
    { id: 'b2', name: 'Buck Mason', country: 'US', notes: null, created_at: '2026-01-01T00:00:00Z' },
  ]

  describe('CategorySidebar', () => {
    it('renders all category names', () => {
      render(<CategorySidebar categories={categories} brands={brands} />)
      expect(screen.getByText('Clothing')).toBeInTheDocument()
      expect(screen.getByText('Shoes')).toBeInTheDocument()
    })

    it('renders subcategory names', () => {
      render(<CategorySidebar categories={categories} brands={brands} />)
      expect(screen.getByText('Tops')).toBeInTheDocument()
      expect(screen.getByText('Shirts')).toBeInTheDocument()
      expect(screen.getByText('Boots')).toBeInTheDocument()
    })

    it('renders brand names', () => {
      render(<CategorySidebar categories={categories} brands={brands} />)
      expect(screen.getByText("Drake's")).toBeInTheDocument()
      expect(screen.getByText('Buck Mason')).toBeInTheDocument()
    })
  })
  ```

- [ ] **Step 6: Run tests to confirm they fail**

  ```bash
  npm test -- --testPathPattern="CategorySidebar"
  ```
  Expected: FAIL — `Cannot find module '@/components/CategorySidebar'`

- [ ] **Step 7: Create `src/components/CategorySidebar.tsx`**

  ```tsx
  // src/components/CategorySidebar.tsx
  import Link from 'next/link'
  import type { Category, Subcategory, Brand } from '@/types'

  type CategoryWithSubs = Category & { subcategories: Subcategory[] }

  type Props = {
    categories: CategoryWithSubs[]
    brands: Brand[]
    selectedCategory?: string
    selectedSubcategory?: string
    selectedBrand?: string
  }

  export default function CategorySidebar({
    categories,
    brands,
    selectedCategory,
    selectedSubcategory,
    selectedBrand,
  }: Props) {
    return (
      <aside className="w-48 flex-shrink-0">
        <div className="mb-8">
          <p className="text-xs tracking-widest uppercase text-warm mb-3">Categories</p>
          <ul className="space-y-1">
            <li>
              <Link
                href="/collection"
                className={`text-sm transition-colors block py-0.5 ${!selectedCategory ? 'text-espresso font-medium' : 'text-warm hover:text-espresso'}`}
              >
                All items
              </Link>
            </li>
            {categories.map(cat => (
              <li key={cat.id}>
                <Link
                  href={`/collection?category=${cat.slug}`}
                  className={`text-sm transition-colors block py-0.5 ${selectedCategory === cat.slug && !selectedSubcategory ? 'text-espresso font-medium' : 'text-warm hover:text-espresso'}`}
                >
                  {cat.name}
                </Link>
                {(selectedCategory === cat.slug || selectedSubcategory) && (
                  <ul className="ml-3 mt-1 space-y-1">
                    {cat.subcategories.map(sub => (
                      <li key={sub.id}>
                        <Link
                          href={`/collection?subcategory=${sub.slug}`}
                          className={`text-xs transition-colors block py-0.5 ${selectedSubcategory === sub.slug ? 'text-espresso font-medium' : 'text-warm hover:text-espresso'}`}
                        >
                          {sub.name}
                        </Link>
                      </li>
                    ))}
                  </ul>
                )}
              </li>
            ))}
          </ul>
        </div>

        <div>
          <p className="text-xs tracking-widest uppercase text-warm mb-3">Brands</p>
          <ul className="space-y-1">
            {brands.map(brand => (
              <li key={brand.id}>
                <Link
                  href={`/collection?brand=${encodeURIComponent(brand.name)}`}
                  className={`text-sm transition-colors block py-0.5 ${selectedBrand === brand.name ? 'text-espresso font-medium' : 'text-warm hover:text-espresso'}`}
                >
                  {brand.name}
                </Link>
              </li>
            ))}
          </ul>
        </div>
      </aside>
    )
  }
  ```

- [ ] **Step 8: Run the `CategorySidebar` tests to confirm they pass**

  ```bash
  npm test -- --testPathPattern="CategorySidebar"
  ```
  Expected: PASS — 3 tests passing.

- [ ] **Step 9: Commit**

  ```bash
  git add src/components/ItemCard.tsx src/components/CategorySidebar.tsx __tests__/components/
  git commit -m "Add ItemCard and CategorySidebar components with tests"
  ```

---

## Task 7: My Collection Page

**Files:**
- Create: `src/app/collection/page.tsx`

**Interfaces:**
- Consumes:
  - `createClient()` from `@/lib/supabase/server`
  - `<CategorySidebar />` from `@/components/CategorySidebar`
  - `<ItemCard />` from `@/components/ItemCard`
  - `getItemLPScore()` from `@/types`
- Produces: `/collection` page with sidebar filters and item grid

- [ ] **Step 1: Create `src/app/collection/page.tsx`**

  ```tsx
  // src/app/collection/page.tsx
  import { createClient } from '@/lib/supabase/server'
  import CategorySidebar from '@/components/CategorySidebar'
  import ItemCard from '@/components/ItemCard'
  import { getItemLPScore } from '@/types'
  import type { Category, Subcategory, Brand, LPReview } from '@/types'

  type PageProps = {
    searchParams: { category?: string; subcategory?: string; brand?: string }
  }

  export default async function CollectionPage({ searchParams }: PageProps) {
    const supabase = await createClient()

    const [{ data: categories }, { data: brands }] = await Promise.all([
      supabase.from('categories').select('*, subcategories(*)').order('name'),
      supabase.from('brands').select('id, name').order('name'),
    ])

    let itemQuery = supabase
      .from('items')
      .select(`
        *,
        brand:brands(*),
        subcategory:subcategories(*, category:categories(*)),
        photos:item_photos(*),
        reviews:lp_reviews(*),
        wear_records:wear_records(id)
      `)
      .eq('in_collection', true)
      .eq('status', 'owned')
      .order('created_at', { ascending: false })

    if (searchParams.subcategory) {
      const { data: sub } = await supabase
        .from('subcategories')
        .select('id')
        .eq('slug', searchParams.subcategory)
        .single()
      if (sub) itemQuery = itemQuery.eq('subcategory_id', sub.id)
    } else if (searchParams.category) {
      const { data: cat } = await supabase
        .from('categories')
        .select('id, subcategories(id)')
        .eq('slug', searchParams.category)
        .single()
      if (cat && cat.subcategories.length > 0) {
        const subIds = (cat.subcategories as { id: string }[]).map(s => s.id)
        itemQuery = itemQuery.in('subcategory_id', subIds)
      }
    }

    if (searchParams.brand) {
      const { data: brand } = await supabase
        .from('brands')
        .select('id')
        .eq('name', searchParams.brand)
        .single()
      if (brand) itemQuery = itemQuery.eq('brand_id', brand.id)
    }

    const { data: rawItems } = await itemQuery

    const items = (rawItems ?? []).map(item => ({
      ...item,
      lp_score: getItemLPScore((item.reviews as LPReview[]) ?? []),
      wear_count: (item.wear_records as { id: string }[])?.length ?? 0,
    }))

    return (
      <div className="max-w-6xl mx-auto px-6 py-10 flex gap-10">
        <CategorySidebar
          categories={(categories as (Category & { subcategories: Subcategory[] })[]) ?? []}
          brands={(brands as Brand[]) ?? []}
          selectedCategory={searchParams.category}
          selectedSubcategory={searchParams.subcategory}
          selectedBrand={searchParams.brand}
        />
        <div className="flex-1 min-w-0">
          <div className="flex items-baseline justify-between mb-8">
            <h1 className="font-serif text-3xl text-espresso">My Collection</h1>
            <span className="text-warm text-sm">{items.length} items</span>
          </div>
          {items.length === 0 ? (
            <p className="text-warm text-sm">No items found.</p>
          ) : (
            <div className="grid grid-cols-2 md:grid-cols-3 gap-x-6 gap-y-10">
              {items.map(item => (
                <ItemCard key={item.id} item={item} />
              ))}
            </div>
          )}
        </div>
      </div>
    )
  }
  ```

- [ ] **Step 2: Verify in browser (after seed data is added in Task 9)**

  Visit `http://localhost:3000/collection`. Should show the sidebar and item grid. If no seed data yet, shows "No items found." — that is correct behaviour.

- [ ] **Step 3: Commit**

  ```bash
  git add src/app/collection/
  git commit -m "Add My Collection page with category sidebar and filtered item grid"
  ```

---

## Task 8: LPScore Component + Item Detail Page

**Files:**
- Create: `src/components/LPScore.tsx`
- Create: `src/app/items/[id]/page.tsx`
- Create: `__tests__/components/LPScore.test.tsx`

**Interfaces:**
- Consumes:
  - `LPScore` receives `score: number | null`, `review?: LPReview`
  - Item detail page uses `createClient()` and all item-related types
- Produces:
  - `<LPScore score={8.2} />` — displays the score badge
  - `<LPScore score={8.2} review={review} />` — displays badge + five dimension breakdown
  - `/items/[id]` — full item detail page

- [ ] **Step 1: Write failing tests for `LPScore`**

  ```tsx
  // __tests__/components/LPScore.test.tsx
  import { render, screen } from '@testing-library/react'
  import LPScore from '@/components/LPScore'
  import type { LPReview } from '@/types'

  const review: LPReview = {
    id: 'r1', item_id: 'i1',
    fit: 8, comfort: 7, quality: 9, versatility: 6, value: 7,
    notes: 'Great tie', reviewed_at: '2026-06-20T00:00:00Z',
  }

  describe('LPScore', () => {
    it('renders the score', () => {
      render(<LPScore score={8.2} />)
      expect(screen.getByText('8.2')).toBeInTheDocument()
      expect(screen.getByText('/ 10')).toBeInTheDocument()
    })

    it('renders "Not yet reviewed" when score is null', () => {
      render(<LPScore score={null} />)
      expect(screen.getByText('Not yet reviewed')).toBeInTheDocument()
    })

    it('renders dimension breakdown when review is provided', () => {
      render(<LPScore score={7.4} review={review} />)
      expect(screen.getByText('Fit')).toBeInTheDocument()
      expect(screen.getByText('Comfort')).toBeInTheDocument()
      expect(screen.getByText('Quality')).toBeInTheDocument()
      expect(screen.getByText('Versatility')).toBeInTheDocument()
      expect(screen.getByText('Value')).toBeInTheDocument()
    })
  })
  ```

- [ ] **Step 2: Run tests to confirm they fail**

  ```bash
  npm test -- --testPathPattern="LPScore"
  ```
  Expected: FAIL — `Cannot find module '@/components/LPScore'`

- [ ] **Step 3: Create `src/components/LPScore.tsx`**

  ```tsx
  // src/components/LPScore.tsx
  import type { LPReview } from '@/types'

  type Props = {
    score: number | null
    review?: LPReview
  }

  const dimensions: { key: keyof Pick<LPReview, 'fit' | 'comfort' | 'quality' | 'versatility' | 'value'>; label: string }[] = [
    { key: 'fit', label: 'Fit' },
    { key: 'comfort', label: 'Comfort' },
    { key: 'quality', label: 'Quality' },
    { key: 'versatility', label: 'Versatility' },
    { key: 'value', label: 'Value' },
  ]

  export default function LPScore({ score, review }: Props) {
    if (score === null) {
      return <p className="text-warm text-sm">Not yet reviewed</p>
    }

    return (
      <div>
        <div className="flex items-baseline gap-1 mb-4">
          <span className="font-serif text-4xl text-espresso">{score}</span>
          <span className="text-warm text-sm">/ 10</span>
        </div>

        {review && (
          <div className="space-y-2">
            {dimensions.map(({ key, label }) => (
              <div key={key} className="flex items-center gap-3">
                <span className="text-xs tracking-wide text-warm w-20 flex-shrink-0">{label}</span>
                <div className="flex-1 h-1 bg-warm/20 rounded-full overflow-hidden">
                  <div
                    className="h-full bg-tan rounded-full"
                    style={{ width: `${review[key] * 10}%` }}
                  />
                </div>
                <span className="text-xs text-espresso w-4 text-right">{review[key]}</span>
              </div>
            ))}
          </div>
        )}
      </div>
    )
  }
  ```

- [ ] **Step 4: Run the `LPScore` tests to confirm they pass**

  ```bash
  npm test -- --testPathPattern="LPScore"
  ```
  Expected: PASS — 3 tests passing.

- [ ] **Step 5: Create `src/app/items/[id]/page.tsx`**

  ```tsx
  // src/app/items/[id]/page.tsx
  import { notFound } from 'next/navigation'
  import Image from 'next/image'
  import Link from 'next/link'
  import { createClient } from '@/lib/supabase/server'
  import LPScore from '@/components/LPScore'
  import { getItemLPScore } from '@/types'
  import type { LPReview, WearRecord } from '@/types'

  export default async function ItemPage({ params }: { params: { id: string } }) {
    const supabase = await createClient()

    const { data: item } = await supabase
      .from('items')
      .select(`
        *,
        brand:brands(*),
        subcategory:subcategories(*, category:categories(*)),
        photos:item_photos(*),
        reviews:lp_reviews(*),
        wear_records:wear_records(*),
        pairings:item_pairings!item_id(
          *,
          paired_item:items!paired_item_id(
            *, brand:brands(*), photos:item_photos(*)
          )
        )
      `)
      .eq('id', params.id)
      .single()

    if (!item) notFound()

    const reviews = (item.reviews as LPReview[]) ?? []
    const wearRecords = (item.wear_records as WearRecord[]) ?? []
    const lpScore = getItemLPScore(reviews)
    const latestReview = reviews.sort(
      (a, b) => new Date(b.reviewed_at).getTime() - new Date(a.reviewed_at).getTime()
    )[0]

    const primaryPhoto = item.photos.find((p: { is_primary: boolean }) => p.is_primary) ?? item.photos[0]
    const photoUrl = primaryPhoto
      ? `${process.env.NEXT_PUBLIC_SUPABASE_URL}/storage/v1/object/public/item-photos/${primaryPhoto.storage_path}`
      : null

    return (
      <div className="max-w-6xl mx-auto px-6 py-10">
        <Link
          href="/collection"
          className="text-xs tracking-widest uppercase text-warm hover:text-espresso transition-colors mb-8 inline-block"
        >
          ← Back
        </Link>

        <div className="grid md:grid-cols-2 gap-12">
          {/* Left: primary photo */}
          <div className="aspect-[3/4] bg-warm/10 relative overflow-hidden">
            {photoUrl ? (
              <Image src={photoUrl} alt={item.name} fill className="object-cover" />
            ) : (
              <div className="absolute inset-0 flex items-center justify-center">
                <span className="text-warm/40 text-xs tracking-widest uppercase">No photo</span>
              </div>
            )}
          </div>

          {/* Right: details */}
          <div>
            <p className="text-xs tracking-widest uppercase text-warm mb-2">
              {item.brand?.name ?? ''}
            </p>
            <h1 className="font-serif text-3xl text-espresso mb-6 leading-tight">{item.name}</h1>

            {item.description && (
              <p className="text-warm text-sm leading-relaxed mb-6">{item.description}</p>
            )}

            {item.material && (
              <p className="text-xs tracking-wide text-warm mb-6">
                <span className="uppercase mr-2">Material</span>
                {item.material}
              </p>
            )}

            <div className="border-t border-espresso/10 pt-6 mb-6">
              <p className="text-xs tracking-widest uppercase text-warm mb-4">LP Score</p>
              <LPScore score={lpScore} review={latestReview} />
            </div>

            <div className="border-t border-espresso/10 pt-6 mb-6">
              <p className="text-xs tracking-widest uppercase text-warm mb-3">Wear History</p>
              <p className="text-sm text-espresso">
                {wearRecords.length === 0
                  ? 'Not yet worn'
                  : `Worn ${wearRecords.length} time${wearRecords.length === 1 ? '' : 's'}`}
              </p>
              {wearRecords.length > 0 && (
                <p className="text-xs text-warm mt-1">
                  Last worn:{' '}
                  {new Date(
                    wearRecords.sort(
                      (a, b) => new Date(b.worn_on).getTime() - new Date(a.worn_on).getTime()
                    )[0].worn_on
                  ).toLocaleDateString('en-GB', { day: 'numeric', month: 'long', year: 'numeric' })}
                </p>
              )}
            </div>

            {item.purchase_price && (
              <div className="border-t border-espresso/10 pt-6">
                <p className="text-xs tracking-widest uppercase text-warm mb-1">Purchase Price</p>
                <p className="text-sm text-espresso">
                  ${item.purchase_price.toFixed(2)}
                  {wearRecords.length > 0 && (
                    <span className="text-warm ml-2">
                      (${(item.purchase_price / wearRecords.length).toFixed(2)} per wear)
                    </span>
                  )}
                </p>
              </div>
            )}
          </div>
        </div>
      </div>
    )
  }
  ```

- [ ] **Step 6: Commit**

  ```bash
  git add src/components/LPScore.tsx src/app/items/ __tests__/components/LPScore.test.tsx
  git commit -m "Add LPScore component and item detail page with tests"
  ```

---

## Task 9: Home Page + Hero Carousel

**Files:**
- Create: `src/components/HeroCarousel.tsx`
- Create: `src/components/DashboardStats.tsx`
- Modify: `src/app/page.tsx`

**Interfaces:**
- Consumes:
  - `HeroCarousel` receives `images: HeroImage[]` from `@/types`
  - `DashboardStats` receives stat values as props
  - Home page uses `createClient()` to fetch stats
- Produces: `/` — hero + dashboard stats

- [ ] **Step 1: Create `src/components/HeroCarousel.tsx`**

  This is a client component — it cycles through images automatically.

  ```tsx
  // src/components/HeroCarousel.tsx
  'use client'
  import { useState, useEffect } from 'react'
  import Image from 'next/image'
  import type { HeroImage } from '@/types'

  export default function HeroCarousel({ images }: { images: HeroImage[] }) {
    const [current, setCurrent] = useState(0)

    useEffect(() => {
      if (images.length <= 1) return
      const interval = setInterval(() => {
        setCurrent(i => (i + 1) % images.length)
      }, 5000)
      return () => clearInterval(interval)
    }, [images.length])

    if (images.length === 0) {
      return (
        <div className="w-full aspect-[16/7] bg-warm/10 flex items-center justify-center">
          <p className="text-warm/50 text-xs tracking-widest uppercase">No hero images yet</p>
        </div>
      )
    }

    const active = images[current]
    const photoUrl = `${process.env.NEXT_PUBLIC_SUPABASE_URL}/storage/v1/object/public/item-photos/${active.storage_path}`

    return (
      <div className="w-full aspect-[16/7] relative overflow-hidden">
        <Image
          key={active.id}
          src={photoUrl}
          alt="Style inspiration"
          fill
          className="object-cover transition-opacity duration-1000"
          priority
        />
        <div className="absolute inset-0 bg-espresso/20" />
      </div>
    )
  }
  ```

- [ ] **Step 2: Create `src/components/DashboardStats.tsx`**

  ```tsx
  // src/components/DashboardStats.tsx
  type Stat = { label: string; value: string | number }

  export default function DashboardStats({ stats }: { stats: Stat[] }) {
    return (
      <div className="grid grid-cols-2 md:grid-cols-4 gap-px bg-espresso/10">
        {stats.map(stat => (
          <div key={stat.label} className="bg-cream px-6 py-6">
            <p className="font-serif text-3xl text-espresso mb-1">{stat.value}</p>
            <p className="text-xs tracking-widest uppercase text-warm">{stat.label}</p>
          </div>
        ))}
      </div>
    )
  }
  ```

- [ ] **Step 3: Replace `src/app/page.tsx`**

  ```tsx
  // src/app/page.tsx
  import { createClient } from '@/lib/supabase/server'
  import HeroCarousel from '@/components/HeroCarousel'
  import DashboardStats from '@/components/DashboardStats'
  import ItemCard from '@/components/ItemCard'
  import { getItemLPScore } from '@/types'
  import type { HeroImage, LPReview } from '@/types'

  export default async function HomePage() {
    const supabase = await createClient()

    const [
      { data: heroImages },
      { data: allItems },
      { count: itemCount },
    ] = await Promise.all([
      supabase.from('hero_images').select('*').order('display_order'),
      supabase
        .from('items')
        .select('*, brand:brands(*), subcategory:subcategories(*, category:categories(*)), photos:item_photos(*), reviews:lp_reviews(*), wear_records:wear_records(id)')
        .eq('in_collection', true)
        .eq('status', 'owned'),
      supabase.from('items').select('*', { count: 'exact', head: true }).eq('in_collection', true).eq('status', 'owned'),
    ])

    const items = (allItems ?? []).map(item => ({
      ...item,
      lp_score: getItemLPScore((item.reviews as LPReview[]) ?? []),
      wear_count: (item.wear_records as { id: string }[])?.length ?? 0,
    }))

    const scoredItems = items.filter(i => i.lp_score !== null)
    const avgScore =
      scoredItems.length > 0
        ? (scoredItems.reduce((sum, i) => sum + (i.lp_score ?? 0), 0) / scoredItems.length).toFixed(1)
        : '—'

    const totalValue = items.reduce((sum, i) => sum + (i.purchase_price ?? 0), 0)
    const mostWorn = [...items].sort((a, b) => b.wear_count - a.wear_count).slice(0, 3)
    const recentItems = [...items]
      .sort((a, b) => new Date(b.created_at).getTime() - new Date(a.created_at).getTime())
      .slice(0, 6)

    const stats = [
      { label: 'Items', value: itemCount ?? 0 },
      { label: 'Collection Value', value: totalValue > 0 ? `$${totalValue.toLocaleString()}` : '—' },
      { label: 'Avg LP Score', value: avgScore },
      { label: 'Most Worn', value: mostWorn[0]?.name ?? '—' },
    ]

    return (
      <div>
        <HeroCarousel images={(heroImages as HeroImage[]) ?? []} />

        <div className="max-w-6xl mx-auto px-6 py-10">
          <div className="mb-12">
            <h1 className="font-serif text-4xl text-espresso mb-2">LP's Style Book</h1>
            <p className="text-warm text-sm tracking-wide">Personal wardrobe intelligence</p>
          </div>

          <DashboardStats stats={stats} />

          {recentItems.length > 0 && (
            <div className="mt-12">
              <h2 className="font-serif text-2xl text-espresso mb-6">Recently Added</h2>
              <div className="grid grid-cols-2 md:grid-cols-3 lg:grid-cols-6 gap-x-4 gap-y-8">
                {recentItems.map(item => (
                  <ItemCard key={item.id} item={item} />
                ))}
              </div>
            </div>
          )}
        </div>
      </div>
    )
  }
  ```

- [ ] **Step 4: Add stub Recommendations page**

  ```tsx
  // src/app/recommendations/page.tsx
  export default function RecommendationsPage() {
    return (
      <div className="max-w-6xl mx-auto px-6 py-10">
        <h1 className="font-serif text-3xl text-espresso mb-2">Recommendations</h1>
        <p className="text-warm text-sm">LP's editorial takes on clothing items — coming in Plan 3.</p>
      </div>
    )
  }
  ```

- [ ] **Step 5: Add Brand detail page**

  ```tsx
  // src/app/brands/[id]/page.tsx
  import { notFound } from 'next/navigation'
  import { createClient } from '@/lib/supabase/server'
  import ItemCard from '@/components/ItemCard'
  import { getItemLPScore } from '@/types'
  import type { LPReview } from '@/types'

  export default async function BrandPage({ params }: { params: { id: string } }) {
    const supabase = await createClient()

    const { data: brand } = await supabase
      .from('brands')
      .select('*')
      .eq('id', params.id)
      .single()

    if (!brand) notFound()

    const { data: rawItems } = await supabase
      .from('items')
      .select('*, brand:brands(*), subcategory:subcategories(*, category:categories(*)), photos:item_photos(*), reviews:lp_reviews(*), wear_records:wear_records(id)')
      .eq('brand_id', params.id)
      .eq('in_collection', true)
      .eq('status', 'owned')
      .order('created_at', { ascending: false })

    const items = (rawItems ?? []).map(item => ({
      ...item,
      lp_score: getItemLPScore((item.reviews as LPReview[]) ?? []),
      wear_count: (item.wear_records as { id: string }[])?.length ?? 0,
    }))

    const scoredItems = items.filter(i => i.lp_score !== null)
    const avgScore =
      scoredItems.length > 0
        ? (scoredItems.reduce((sum, i) => sum + (i.lp_score ?? 0), 0) / scoredItems.length).toFixed(1)
        : null
    const totalWears = items.reduce((sum, i) => sum + i.wear_count, 0)

    return (
      <div className="max-w-6xl mx-auto px-6 py-10">
        <div className="mb-10">
          <p className="text-xs tracking-widest uppercase text-warm mb-2">{brand.country ?? 'Brand'}</p>
          <h1 className="font-serif text-4xl text-espresso mb-4">{brand.name}</h1>
          <div className="flex gap-8 text-sm text-warm">
            <span>{items.length} items owned</span>
            {avgScore && <span>Avg LP Score: {avgScore}</span>}
            <span>{totalWears} total wears</span>
          </div>
          {brand.notes && (
            <p className="text-warm text-sm leading-relaxed mt-4 max-w-prose">{brand.notes}</p>
          )}
        </div>

        {items.length > 0 && (
          <div className="grid grid-cols-2 md:grid-cols-4 gap-x-6 gap-y-10">
            {items.map(item => (
              <ItemCard key={item.id} item={item} />
            ))}
          </div>
        )}
      </div>
    )
  }
  ```

- [ ] **Step 6: Commit**

  ```bash
  git add src/app/page.tsx src/app/recommendations/ src/app/brands/ src/components/HeroCarousel.tsx src/components/DashboardStats.tsx
  git commit -m "Add home page with hero carousel, dashboard stats, brand pages, and recommendations stub"
  ```

---

## Task 10: Seed Data

**Files:**
- Create: `supabase/seed.ts`

**Interfaces:**
- Produces: 17 brands, 38 items (all `in_collection: true`), one review per item, wear records per item, 16 wishlist items in Supabase. Run with: `npx tsx supabase/seed.ts`

- [ ] **Step 1: Create `supabase/seed.ts`**

  ```typescript
  // supabase/seed.ts
  import { createClient } from '@supabase/supabase-js'

  const supabase = createClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.SUPABASE_SERVICE_ROLE_KEY!, // service role key — bypasses RLS for seeding
  )

  // ──────────────────────────────────────────────────────────────
  // Reference data
  // ──────────────────────────────────────────────────────────────

  const BRANDS = [
    { name: "Drake's", country: 'UK', notes: 'Heritage British accessories and tailoring. Exceptional tie and pocket square selection.' },
    { name: 'Buck Mason', country: 'US', notes: 'American basics done properly. Best slub tees and chinos at their price point.' },
    { name: 'Sunspel', country: 'UK', notes: 'The gold standard for underwear and casual basics. Long-staple cotton only.' },
    { name: 'Loro Piana', country: 'Italy', notes: 'Unmatched cashmere and superfine wool. Investment pieces that last decades.' },
    { name: 'Reiss', country: 'UK', notes: 'Reliable smart-casual staples. Good tailoring for the price.' },
    { name: 'Orlebar Brown', country: 'UK', notes: 'Premium resort and swimwear. Worth the price for the quality and fit.' },
    { name: 'Officine Générale', country: 'France', notes: 'Relaxed Parisian tailoring. Linen and cotton blends that breathe.' },
    { name: 'Albam', country: 'UK', notes: 'Understated British quality. Great knitwear and workwear-inspired pieces.' },
    { name: 'Corridor NYC', country: 'US', notes: 'Neo-trad American. Beautiful fabrics sourced from Japanese mills.' },
    { name: 'Alex Mill', country: 'US', notes: 'NYC basics with a relaxed fit. Excellent denim and oxford shirts.' },
    { name: 'The Real McCoy\'s', country: 'Japan', notes: 'Japanese heritage reproductions. Made to last a lifetime.' },
    { name: 'Naked & Famous', country: 'Canada', notes: 'Best raw denim at their price point. Fades beautifully over time.' },
    { name: 'Polo Ralph Lauren', country: 'US', notes: 'Reliable for basics. Oxford shirts and chinos are almost always good.' },
    { name: 'Hamilton Shirts', country: 'US', notes: 'Heritage American shirtmaker. Solid construction, classic fits.' },
    { name: 'Turnbull & Asser', country: 'UK', notes: 'Jermyn Street institution. Dress shirts and ties at their finest.' },
    { name: 'New & Lingwood', country: 'UK', notes: 'Eton and Cambridge outfitter. Socks and underwear are exceptional.' },
    { name: 'Uniqlo', country: 'Japan', notes: 'The benchmark for affordable basics. Linen shirts and merino wool are standouts.' },
  ]

  const ITEMS = [
    // Drake's
    { brand: "Drake's", name: 'Navy Grenadine Tie', sub: 'Ties', price: 145, mat: 'Silk grenadine', desc: 'The essential navy tie. Open-weave grenadine catches the light without being flashy. Pairs with everything.', wears: 22, fit: 9, comfort: 9, quality: 10, vers: 9, val: 8, failed: false },
    { brand: "Drake's", name: 'Wool Challis Pocket Square', sub: 'Accessories', price: 65, mat: 'Wool challis', desc: 'Burgundy and navy paisley. Soft enough to fold without creasing, drapes beautifully in a chest pocket.', wears: 18, fit: 10, comfort: 10, quality: 10, vers: 7, val: 7, failed: false },
    // Buck Mason
    { brand: 'Buck Mason', name: 'Slub Cotton Tee', sub: 'Tops', price: 58, mat: '100% slub cotton', desc: 'The best casual tee at this price. The texture is subtle and improves with washing.', wears: 41, fit: 8, comfort: 10, quality: 8, vers: 9, val: 9, failed: false },
    { brand: 'Buck Mason', name: 'Stretch Chino', sub: 'Bottoms', price: 118, mat: 'Cotton-stretch blend', desc: 'Comfortable enough for a long day but looks smart. The stretch is barely noticeable to anyone but you.', wears: 28, fit: 7, comfort: 9, quality: 7, vers: 8, val: 8, failed: false },
    // Sunspel
    { brand: 'Sunspel', name: 'Riviera Polo Shirt', sub: 'Tops', price: 165, mat: 'Long-staple cotton', desc: 'The polo shirt by which all others are judged. Minimal branding, perfect weight, flattering fit.', wears: 19, fit: 9, comfort: 10, quality: 10, vers: 8, val: 7, failed: false },
    { brand: 'Sunspel', name: 'Classic Boxer Short', sub: 'Accessories', price: 55, mat: 'Long-staple cotton', desc: 'Expensive for underwear and worth every cent. Last significantly longer than cheaper alternatives.', wears: 60, fit: 9, comfort: 10, quality: 10, vers: 10, val: 8, failed: false },
    // Loro Piana
    { brand: 'Loro Piana', name: 'Cashmere Crewneck Sweater', sub: 'Knitwear', price: 1200, mat: 'Baby cashmere', desc: 'Softer than anything else I own. An investment that pays off every winter.', wears: 14, fit: 9, comfort: 10, quality: 10, vers: 9, val: 7, failed: false },
    { brand: 'Loro Piana', name: 'Wish Scarf', sub: 'Scarves', price: 450, mat: 'Baby cashmere', desc: 'Impossibly light for how warm it keeps you. One of the best purchases I\'ve made.', wears: 12, fit: 10, comfort: 10, quality: 10, vers: 8, val: 7, failed: false },
    // Reiss
    { brand: 'Reiss', name: 'Slim Wool Trousers', sub: 'Bottoms', price: 195, mat: 'Wool blend', desc: 'Good everyday trousers for smart-casual occasions. Fit is reliable, nothing special but nothing bad.', wears: 24, fit: 7, comfort: 7, quality: 7, vers: 8, val: 7, failed: false },
    { brand: 'Reiss', name: 'Merino Turtleneck', sub: 'Knitwear', price: 145, mat: 'Merino wool', desc: 'Bought in dark navy. Pilled after six washes — disappointing for the price.', wears: 4, fit: 6, comfort: 5, quality: 3, vers: 5, val: 2, failed: true, failReason: 'poor quality' },
    // Orlebar Brown
    { brand: 'Orlebar Brown', name: 'Bulldog Swim Short', sub: 'Bottoms', price: 195, mat: 'Recycled polyester', desc: 'The swim short that looks as good at the bar as in the water. Worth the price.', wears: 9, fit: 9, comfort: 9, quality: 9, vers: 7, val: 7, failed: false },
    // Officine Générale
    { brand: 'Officine Générale', name: 'Linen Overshirt', sub: 'Shirts', price: 295, mat: 'Irish linen', desc: 'Perfect summer layer. Worn open over a tee or buttoned up with chinos.', wears: 16, fit: 9, comfort: 10, quality: 9, vers: 9, val: 8, failed: false },
    // Albam
    { brand: 'Albam', name: 'Shetland Crewneck', sub: 'Knitwear', price: 175, mat: 'Shetland wool', desc: 'Proper scratchy Shetland, as it should be. Rustic texture and warm. A true workhorse.', wears: 20, fit: 8, comfort: 7, quality: 9, vers: 8, val: 9, failed: false },
    { brand: 'Albam', name: 'Chore Coat', sub: 'Outerwear', price: 225, mat: 'Cotton canvas', desc: 'Versatile work jacket. The pockets are genuinely useful and it breaks in beautifully.', wears: 17, fit: 8, comfort: 8, quality: 9, vers: 9, val: 9, failed: false },
    // Corridor NYC
    { brand: 'Corridor NYC', name: 'Seersucker Blazer', sub: 'Outerwear', price: 495, mat: 'Cotton seersucker', desc: 'Summer tailoring that doesn\'t take itself too seriously. The texture does the work.', wears: 7, fit: 8, comfort: 9, quality: 9, vers: 7, val: 7, failed: false },
    // Alex Mill
    { brand: 'Alex Mill', name: 'Standard Oxford Shirt', sub: 'Shirts', price: 115, mat: 'Cotton oxford', desc: 'The benchmark oxford shirt. Relaxed fit, proper weight, no branding. Fades beautifully.', wears: 33, fit: 9, comfort: 9, quality: 9, vers: 10, val: 9, failed: false },
    { brand: 'Alex Mill', name: 'Mill Jean', sub: 'Bottoms', price: 168, mat: '100% cotton selvedge denim', desc: 'Classic five-pocket jean in a straight fit. Honest denim at a fair price.', wears: 27, fit: 8, comfort: 8, quality: 9, vers: 9, val: 9, failed: false },
    // The Real McCoy's
    { brand: 'The Real McCoy\'s', name: 'Joe McCoy Denim Jacket', sub: 'Outerwear', price: 595, mat: 'Japanese selvedge denim', desc: 'Built to last fifty years. The indigo is already fading in all the right places.', wears: 11, fit: 8, comfort: 8, quality: 10, vers: 8, val: 8, failed: false },
    // Naked & Famous
    { brand: 'Naked & Famous', name: 'Weird Guy Raw Denim', sub: 'Bottoms', price: 195, mat: 'Japanese raw selvedge denim', desc: 'Six months of wear and the fades are extraordinary. Worth the patience.', wears: 45, fit: 8, comfort: 7, quality: 10, vers: 8, val: 9, failed: false },
    // Polo Ralph Lauren
    { brand: 'Polo Ralph Lauren', name: 'Custom Fit Oxford Shirt', sub: 'Shirts', price: 98, mat: 'Cotton oxford cloth', desc: 'Reliable and inoffensive. Not exciting but always correct.', wears: 19, fit: 7, comfort: 8, quality: 7, vers: 9, val: 8, failed: false },
    { brand: 'Polo Ralph Lauren', name: 'Merino Wool V-Neck', sub: 'Knitwear', price: 145, mat: 'Merino wool', desc: 'Bought this as a placeholder. The colour was off and it felt cheap. Donated after three wears.', wears: 3, fit: 5, comfort: 5, quality: 4, vers: 5, val: 3, failed: true, failReason: "doesn't suit style" },
    // Hamilton Shirts
    { brand: 'Hamilton Shirts', name: 'Broadcloth Dress Shirt', sub: 'Shirts', price: 225, mat: '2-ply cotton broadcloth', desc: 'American shirtmaking at its best. The collar roll is perfect and it keeps its shape all day.', wears: 13, fit: 9, comfort: 9, quality: 10, vers: 8, val: 8, failed: false },
    // Turnbull & Asser
    { brand: 'Turnbull & Asser', name: 'Jermyn Street Dress Shirt', sub: 'Shirts', price: 295, mat: '2-ply Sea Island cotton', desc: 'The finest cotton I\'ve worn against skin. The shirt equivalent of a Loro Piana sweater.', wears: 8, fit: 9, comfort: 10, quality: 10, vers: 7, val: 7, failed: false },
    { brand: 'Turnbull & Asser', name: 'Repp Stripe Tie', sub: 'Ties', price: 165, mat: 'Silk', desc: 'Navy and gold repp stripe. A Jermyn Street classic. Goes with everything in the wardrobe.', wears: 11, fit: 9, comfort: 9, quality: 10, vers: 8, val: 7, failed: false },
    // New & Lingwood
    { brand: 'New & Lingwood', name: 'Merino Ankle Socks', sub: 'Accessories', price: 28, mat: 'Merino wool', desc: 'Exceptional socks. Lasted three years without holes or thinning. Worth the premium.', wears: 90, fit: 9, comfort: 10, quality: 10, vers: 10, val: 9, failed: false },
    // Uniqlo
    { brand: 'Uniqlo', name: 'Premium Linen Shirt', sub: 'Shirts', price: 49, mat: 'French linen', desc: 'The best value linen shirt available. The fabric is good and it washes well.', wears: 22, fit: 7, comfort: 9, quality: 7, vers: 8, val: 10, failed: false },
    { brand: 'Uniqlo', name: 'Extra Fine Merino Crewneck', sub: 'Knitwear', price: 49, mat: 'Extra fine merino', desc: 'Impressive for the price. Pilled slightly after a year but remained wearable. Good value.', wears: 30, fit: 7, comfort: 8, quality: 6, vers: 9, val: 9, failed: false },
    // Additional items to reach 38
    { brand: "Drake's", name: 'Madder Silk Tie', sub: 'Ties', price: 165, mat: 'Madder silk', desc: 'Brick red with a subtle paisley. The matte finish of madder silk is unlike anything else.', wears: 9, fit: 9, comfort: 9, quality: 10, vers: 7, val: 7, failed: false },
    { brand: 'Sunspel', name: 'Sea Island Cotton Tee', sub: 'Tops', price: 85, mat: 'Sea Island cotton', desc: 'The ultimate white tee. So fine it feels like a second skin. Worth the indulgence.', wears: 15, fit: 9, comfort: 10, quality: 10, vers: 10, val: 7, failed: false },
    { brand: 'Albam', name: 'Ripstop Work Trouser', sub: 'Bottoms', price: 145, mat: 'Cotton ripstop', desc: 'Weekend workhorse. Comfortable, durable, and looks better with wear.', wears: 21, fit: 8, comfort: 9, quality: 9, vers: 7, val: 9, failed: false },
    { brand: 'Officine Générale', name: 'Cotton Twill Chino', sub: 'Bottoms', price: 265, mat: 'Japanese cotton twill', desc: 'Impeccable French chino. The fabric has a slight sheen and drapes beautifully.', wears: 18, fit: 9, comfort: 8, quality: 9, vers: 9, val: 7, failed: false },
    { brand: 'Corridor NYC', name: 'Japanese Cotton Poplin Shirt', sub: 'Shirts', price: 295, mat: 'Japanese cotton poplin', desc: 'The fabric feels almost handmade. A shirt to treasure.', wears: 10, fit: 8, comfort: 9, quality: 10, vers: 8, val: 7, failed: false },
    { brand: 'Polo Ralph Lauren', name: 'Relaxed Fit Chino', sub: 'Bottoms', price: 98, mat: 'Chino cotton', desc: 'Bought impulsively in the wrong size. Too large in the seat. Classic impulse buy mistake.', wears: 2, fit: 3, comfort: 5, quality: 6, vers: 5, val: 3, failed: true, failReason: 'poor fit' },
    { brand: "Drake's", name: 'Cashmere Scarf', sub: 'Scarves', price: 195, mat: 'Cashmere', desc: 'Camel-coloured cashmere. Light and warm. A winter essential.', wears: 16, fit: 10, comfort: 10, quality: 9, vers: 9, val: 8, failed: false },
    { brand: 'Reiss', name: 'Harrington Jacket', sub: 'Outerwear', price: 245, mat: 'Cotton', desc: 'Clean and versatile transitional jacket. Smart without being stiff.', wears: 12, fit: 8, comfort: 8, quality: 7, vers: 9, val: 7, failed: false },
    { brand: 'Loro Piana', name: 'Windmate Jacket', sub: 'Outerwear', price: 1850, mat: 'Storm System® fabric', desc: 'The most functional garment I own. Windproof, rainproof, and packs to nothing.', wears: 22, fit: 9, comfort: 10, quality: 10, vers: 8, val: 7, failed: false },
    { brand: 'Uniqlo', name: 'Wide-Leg Chino', sub: 'Bottoms', price: 49, mat: 'Cotton', desc: 'Wrong cut for my body type. Looked shapeless and was never worn after the first try-on.', wears: 1, fit: 2, comfort: 6, quality: 6, vers: 4, val: 2, failed: true, failReason: "doesn't suit style" },
    { brand: 'The Real McCoy\'s', name: 'Buco J-100 Leather Jacket', sub: 'Outerwear', price: 1200, mat: 'Horsehide leather', desc: 'A jacket to pass down. The horsehide is stiff now and will break in over years.', wears: 6, fit: 8, comfort: 7, quality: 10, vers: 7, val: 8, failed: false },
  ] as const

  const WISHLIST = [
    { brand: "Drake's", name: 'Grenadine Tie in Forest Green', price: 145, status: 'Considering', interest: 8, notes: 'Would fill a gap in the tie rotation. Seen it worn by several people I respect.' },
    { brand: 'Sunspel', name: 'Q82 Swim Short', price: 110, status: 'Researching', interest: 7, notes: 'Waiting for the navy colourway to come back in stock.' },
    { brand: 'Loro Piana', name: 'Open Walk Loafers', price: 890, status: 'Wishlist', interest: 9, notes: 'The ideal warm-weather loafer. Watching for a sale.' },
    { brand: 'Officine Générale', name: 'Batiste Shirt', price: 255, status: 'Researching', interest: 7, notes: 'Very lightweight summer option. Need to try the fit in person.' },
    { brand: 'Albam', name: 'Indigo Work Shirt', price: 145, status: 'Considering', interest: 8, notes: 'The indigo version of the chore coat would pair well with raw denim.' },
    { brand: 'Alex Mill', name: 'Washed Twill Trouser', price: 148, status: 'Wishlist', interest: 6, notes: 'A more relaxed alternative to chinos for weekends.' },
    { brand: 'Corridor NYC', name: 'Floral Jacquard Tie', price: 185, status: 'Considering', interest: 7, notes: 'Something different. The pattern is bold but the colours are muted enough.' },
    { brand: 'Hamilton Shirts', name: 'End-on-End Shirt', price: 225, status: 'Researching', interest: 8, notes: 'The end-on-end texture adds interest without pattern. On the shortlist.' },
    { brand: 'Loro Piana', name: 'Baby Cashmere Polo', price: 1050, status: 'Wishlist', interest: 8, notes: 'An indulgence. The summer equivalent of the crewneck.' },
    { brand: 'New & Lingwood', name: 'Striped Cotton Pyjamas', price: 195, status: 'Wishlist', interest: 6, notes: 'A considered luxury. Long-staple cotton for sleeping in.' },
    { brand: 'Turnbull & Asser', name: 'Poplin Dress Shirt in Pale Blue', price: 295, status: 'Researching', interest: 9, notes: 'The blue equivalent of the white. Would replace two lesser shirts.' },
    { brand: 'Naked & Famous', name: 'Easy Guy in Heavyweight Denim', price: 215, status: 'Considering', interest: 7, notes: 'A more relaxed cut for weekends. The heavyweight fabric will fade slowly.' },
    { brand: 'Reiss', name: 'Slim Cotton Suit', price: 495, status: 'Archived', interest: 4, notes: 'Decided against — the wool-blend suits I already own are better for formal occasions.' },
    { brand: 'Orlebar Brown', name: 'Setter Shirt', price: 195, status: 'Purchased', interest: 9, notes: 'Ordered. Linen-cotton blend in navy. Should arrive next week.' },
    { brand: 'Buck Mason', name: 'Raw Denim Jacket', price: 198, status: 'Wishlist', interest: 7, notes: 'American alternative to the Real McCoy\'s for casual wear.' },
    { brand: 'Albam', name: 'Merino Turtle Neck', price: 165, status: 'Considering', interest: 8, notes: 'A cleaner alternative to the Shetland for smart-casual occasions.' },
  ]

  async function main() {
    process.stdout.write('Seeding brands...\n')
    const { data: insertedBrands, error: brandErr } = await supabase
      .from('brands')
      .upsert(BRANDS, { onConflict: 'name' })
      .select()
    if (brandErr) throw brandErr
    const brandMap = Object.fromEntries((insertedBrands ?? []).map(b => [b.name, b.id]))

    process.stdout.write('Fetching subcategories...\n')
    const { data: subs } = await supabase.from('subcategories').select('id, slug, name')
    const subMap = Object.fromEntries((subs ?? []).map(s => [s.name, s.id]))

    process.stdout.write('Seeding items...\n')
    for (const item of ITEMS) {
      const { data: inserted, error: itemErr } = await supabase
        .from('items')
        .insert({
          name: item.name,
          brand_id: brandMap[item.brand],
          subcategory_id: subMap[item.sub],
          description: item.desc,
          material: item.mat,
          purchase_price: item.price,
          in_collection: true,
          is_recommendation: false,
          in_wishlist: false,
          status: 'owned',
        })
        .select()
        .single()
      if (itemErr) throw itemErr

      // Add review
      const { error: reviewErr } = await supabase.from('lp_reviews').insert({
        item_id: inserted.id,
        fit: item.fit,
        comfort: item.comfort,
        quality: item.quality,
        versatility: item.vers,
        value: item.val,
        notes: null,
      })
      if (reviewErr) throw reviewErr

      // Add wear records
      const wearCount = item.wears
      const baseDate = new Date('2024-01-01')
      const seasons = ['Spring', 'Summer', 'Autumn', 'Winter'] as const
      const occasions = ['Casual', 'Smart Casual', 'Work', 'Formal', 'Sport'] as const
      for (let i = 0; i < wearCount; i++) {
        const date = new Date(baseDate)
        date.setDate(baseDate.getDate() + Math.floor((i / wearCount) * 540))
        await supabase.from('wear_records').insert({
          item_id: inserted.id,
          worn_on: date.toISOString().split('T')[0],
          season: seasons[i % 4],
          occasion: occasions[i % 5],
        })
      }

      // Add failed purchase record if applicable
      if (item.failed) {
        await supabase.from('failed_purchases').insert({
          item_id: inserted.id,
          reason: (item as { failReason?: string }).failReason ?? 'other',
          notes: null,
        })
      }
    }

    process.stdout.write('Seeding wishlist...\n')
    for (const wi of WISHLIST) {
      const { data: draftItem } = await supabase
        .from('items')
        .insert({
          name: wi.name,
          brand_id: brandMap[wi.brand],
          in_collection: false,
          in_wishlist: true,
          is_recommendation: false,
          status: 'owned',
          purchase_price: wi.price,
        })
        .select()
        .single()
      if (!draftItem) continue

      await supabase.from('wishlist_items').insert({
        item_id: draftItem.id,
        status: wi.status,
        interest_score: wi.interest,
        notes: wi.notes,
      })
    }

    process.stdout.write('Done.\n')
  }

  main().catch(err => {
    process.stderr.write(String(err) + '\n')
    process.exit(1)
  })
  ```

- [ ] **Step 2: Get the service role key**

  In the Supabase dashboard → Settings → API → copy the **service_role** key (not the anon key). Add it to `.env.local`:
  ```
  SUPABASE_SERVICE_ROLE_KEY=your-service-role-key-here
  ```

  > **Important:** The service role key bypasses RLS. Never commit it, never expose it client-side.

- [ ] **Step 3: Run the seed script**

  ```bash
  npx tsx supabase/seed.ts
  ```
  Expected output:
  ```
  Seeding brands...
  Fetching subcategories...
  Seeding items...
  Seeding wishlist...
  Done.
  ```

- [ ] **Step 4: Verify in Supabase dashboard**

  In Supabase → Table Editor → `brands`: should show 17 rows. `items`: should show 54 rows (38 collection + 16 wishlist). `lp_reviews`: 38 rows. `wear_records`: many rows.

- [ ] **Step 5: Verify in the browser**

  Visit `http://localhost:3000/collection`. Should now display a full grid of items with names and brands. Visit `http://localhost:3000` — dashboard stats should show real numbers.

- [ ] **Step 6: Commit**

  ```bash
  git add supabase/seed.ts
  git commit -m "Add seed data: 17 brands, 38 collection items, 16 wishlist items with reviews and wear records"
  ```

---

## Task 11: Deploy to Vercel

**Files:** No code changes — Vercel configuration only.

- [ ] **Step 1: Install Vercel CLI and log in**

  ```bash
  npm install -g vercel
  vercel login
  ```

- [ ] **Step 2: Deploy**

  Run from inside `lp-style-book/`:
  ```bash
  vercel
  ```
  Follow the prompts: link to your Vercel account, create a new project named `lp-style-book`, accept all defaults.

- [ ] **Step 3: Add environment variables in Vercel**

  In the Vercel dashboard → your project → Settings → Environment Variables. Add:
  - `NEXT_PUBLIC_SUPABASE_URL` — your Supabase project URL
  - `NEXT_PUBLIC_SUPABASE_ANON_KEY` — your Supabase anon key
  - `SUPABASE_SERVICE_ROLE_KEY` — your service role key (mark as secret)

- [ ] **Step 4: Redeploy**

  ```bash
  vercel --prod
  ```
  Expected: a live URL like `lp-style-book.vercel.app`.

- [ ] **Step 5: Verify the live site**

  Visit the Vercel URL. Collection page should load with seed data. Admin login should work.

- [ ] **Step 6: Update `README.md` with the live URL**

  Add a line at the top of `README.md`:
  ```markdown
  **Live:** https://lp-style-book.vercel.app
  ```

- [ ] **Step 7: Commit and push**

  ```bash
  git add README.md
  git commit -m "Add live Vercel URL to README"
  git push
  ```

---

## Self-Review Checklist

After writing this plan, checking it against the spec:

**Spec coverage:**
- [x] Database schema — Task 2
- [x] Categories and subcategories seeded — Task 2
- [x] TypeScript strict types for all entities — Task 3
- [x] Supabase Auth (email magic link for LP) — Task 4
- [x] Admin route guard — Task 4
- [x] Public read access (anyone can browse) — Task 2 (RLS policies)
- [x] Warm minimal visual design (palette, fonts) — Task 1, Task 5
- [x] Nav: Home, Recommendations, My Collection — Task 5
- [x] Home page with hero carousel and dashboard stats — Task 9
- [x] My Collection page with sidebar and grid — Task 7
- [x] Item detail page with LP score, wear history, cost-per-wear — Task 8
- [x] Brand detail page with stats — Task 9
- [x] Recommendations stub — Task 9
- [x] Seed data: 17 brands, 38 items, reviews, wear records, 16 wishlist items — Task 10
- [x] Failed purchase records in seed data — Task 10
- [x] computeLPScore unit tests — Task 3
- [x] ItemCard, CategorySidebar, LPScore component tests — Tasks 6, 8
- [x] Deployed to Vercel — Task 11

**What is NOT in this plan (covered in Plans 2 and 3):**
- Product import workflow (URL scraping, fallback search, photo download)
- Admin item add/edit UI
- Wear tracking UI (log a wear)
- LP Review UI (write a review)
- Wishlist Pipeline management UI
- Recommendations page with labelled figure
- Visual polish (animations, transitions)

---

## Execution Handoff

Plan complete and saved to `docs/superpowers/plans/2026-06-20-lp-style-book-foundation.md`.

**Two execution options:**

**1. Subagent-Driven (recommended)** — A fresh subagent handles each task and I review between tasks. Fastest iteration, catches mistakes early.

**2. Inline Execution** — Execute tasks in this session using the executing-plans skill, with checkpoints for review.

Which approach?
