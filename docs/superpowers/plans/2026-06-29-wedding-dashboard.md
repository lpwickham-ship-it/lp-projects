# Wedding Dashboard Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a local Next.js wedding planning dashboard for Louis-Philippe & Anna, pre-loaded with their to-do list, budget, and vendor contacts, with checkbox state persisted in localStorage.

**Architecture:** Single scrollable Next.js page with four sections — Hero (countdown), Todo checklist, Budget table, Vendor contacts. All mutable state (checked todos, vendor fields) lives in localStorage under one key. Data is hardcoded in TypeScript files; no backend.

**Tech Stack:** Next.js 15 (App Router), TypeScript strict, Tailwind CSS, next/font/google, localStorage, npm, jest + @testing-library/jest-dom for data tests.

## Global Constraints

- TypeScript strict mode always on
- npm only — never yarn or pnpm
- App Router (`src/app/`) — no Pages Router
- Import alias `@/` maps to `src/`
- Tailwind for all styling — no CSS modules, no inline styles
- No external state library — only React state + localStorage
- localStorage key: `"wickham-wedding"`
- Run `npm run dev` to start — no deployment needed

---

## File Map

```
projects/wedding-dashboard/
├── src/
│   ├── app/
│   │   ├── layout.tsx          ← root layout: fonts, metadata, body class
│   │   ├── page.tsx            ← assembles Hero + TodoSection + BudgetTable + VendorContacts
│   │   └── globals.css         ← @tailwind directives + font CSS vars
│   ├── components/
│   │   ├── Hero.tsx            ← countdown + names + date + hashtag
│   │   ├── TodoItem.tsx        ← single checkbox row with priority badge
│   │   ├── TodoSection.tsx     ← all categories + progress counter, uses useTodoState
│   │   ├── BudgetTable.tsx     ← read-only table with status dots and totals row
│   │   └── VendorContacts.tsx  ← editable contact cards, uses useVendorState
│   ├── data/
│   │   ├── todos.ts            ← all categories and items (exported constants)
│   │   └── budget.ts           ← all budget rows and vendor list (exported constants)
│   └── hooks/
│       └── useLocalStorage.ts  ← useTodoState + useVendorState (SSR-safe)
├── tailwind.config.ts          ← extends fontFamily for serif + sans
├── package.json
└── README.md
```

---

### Task 1: Scaffold the project

**Files:**
- Create: `projects/wedding-dashboard/` (all base files via create-next-app)
- Modify: `tailwind.config.ts`
- Modify: `src/app/globals.css` (replace boilerplate)

**Interfaces:**
- Produces: running Next.js dev server, Tailwind with custom font families, `@/` alias working

- [ ] **Step 1: Run create-next-app**

Run from `projects/`:
```bash
npx create-next-app@latest wedding-dashboard \
  --typescript \
  --tailwind \
  --eslint \
  --app \
  --src-dir \
  --import-alias "@/*" \
  --no-git
```
When prompted about `--turbopack`, answer yes.

Expected output: "Success! Created wedding-dashboard at ..."

- [ ] **Step 2: Install jest and testing dependencies**

```bash
cd wedding-dashboard
npm install --save-dev jest jest-environment-jsdom @types/jest
```

- [ ] **Step 3: Create jest.config.ts**

`next/jest` is included with Next.js — no separate install needed. It handles TypeScript and the `@/` alias automatically.

```typescript
// jest.config.ts
import type { Config } from 'jest'
import nextJest from 'next/jest.js'

const createJestConfig = nextJest({ dir: './' })

const config: Config = {
  testEnvironment: 'node',
  testMatch: ['**/__tests__/**/*.test.ts'],
}

export default createJestConfig(config)
```

- [ ] **Step 4: Update tailwind.config.ts**

Replace the generated `tailwind.config.ts` with:
```typescript
import type { Config } from 'tailwindcss'

const config: Config = {
  content: [
    './src/pages/**/*.{js,ts,jsx,tsx,mdx}',
    './src/components/**/*.{js,ts,jsx,tsx,mdx}',
    './src/app/**/*.{js,ts,jsx,tsx,mdx}',
  ],
  theme: {
    extend: {
      fontFamily: {
        serif: ['var(--font-playfair)', 'Georgia', 'serif'],
        sans: ['var(--font-inter)', 'system-ui', 'sans-serif'],
      },
    },
  },
  plugins: [],
}

export default config
```

- [ ] **Step 5: Replace globals.css**

Replace all contents of `src/app/globals.css` with:
```css
@tailwind base;
@tailwind components;
@tailwind utilities;
```

- [ ] **Step 6: Verify dev server starts**

```bash
npm run dev
```
Expected: "Ready on http://localhost:3000" (no errors). Then Ctrl+C.

- [ ] **Step 7: Commit**

```bash
git init && git add -A && git commit -m "Scaffold wedding-dashboard Next.js project"
```

---

### Task 2: Data files

**Files:**
- Create: `src/data/todos.ts`
- Create: `src/data/budget.ts`
- Create: `src/data/__tests__/todos.test.ts`
- Create: `src/data/__tests__/budget.test.ts`

**Interfaces:**
- Produces:
  - `TODO_CATEGORIES: TodoCategory[]` from `@/data/todos`
  - `TodoItem` type: `{ id: string; label: string; priority?: 'high' | 'normal'; note?: string }`
  - `TodoCategory` type: `{ id: string; title: string; items: TodoItem[] }`
  - `BUDGET_ROWS: BudgetRow[]` from `@/data/budget`
  - `BudgetRow` type: `{ id: string; service: string; status: BudgetStatus; paid: number | null; owing: number | null; total: number | null; notes?: string }`
  - `BudgetStatus` type: `'Paid' | 'In Progress' | 'TBD'`
  - `VENDORS: Vendor[]` from `@/data/budget`
  - `Vendor` type: `{ id: string; name: string; category: string }`

- [ ] **Step 1: Write the failing data integrity tests**

Create `src/data/__tests__/todos.test.ts`:
```typescript
import { TODO_CATEGORIES } from '../todos'

it('all todo IDs are unique', () => {
  const ids = TODO_CATEGORIES.flatMap(c => c.items.map(i => i.id))
  expect(new Set(ids).size).toBe(ids.length)
})

it('all items have a non-empty label', () => {
  const items = TODO_CATEGORIES.flatMap(c => c.items)
  items.forEach(item => expect(item.label.trim().length).toBeGreaterThan(0))
})

it('all category IDs are unique', () => {
  const ids = TODO_CATEGORIES.map(c => c.id)
  expect(new Set(ids).size).toBe(ids.length)
})
```

Create `src/data/__tests__/budget.test.ts`:
```typescript
import { BUDGET_ROWS, VENDORS } from '../budget'

it('all budget row IDs are unique', () => {
  const ids = BUDGET_ROWS.map(r => r.id)
  expect(new Set(ids).size).toBe(ids.length)
})

it('all vendor IDs are unique', () => {
  const ids = VENDORS.map(v => v.id)
  expect(new Set(ids).size).toBe(ids.length)
})

it('status is one of the allowed values', () => {
  const allowed = ['Paid', 'In Progress', 'TBD']
  BUDGET_ROWS.forEach(r => expect(allowed).toContain(r.status))
})
```

- [ ] **Step 2: Run tests to confirm they fail**

```bash
npx jest src/data
```
Expected: FAIL — "Cannot find module '../todos'"

- [ ] **Step 3: Write src/data/todos.ts**

```typescript
export type Priority = 'high' | 'normal'

export interface TodoItem {
  id: string
  label: string
  priority?: Priority
}

export interface TodoCategory {
  id: string
  title: string
  items: TodoItem[]
}

export const TODO_CATEGORIES: TodoCategory[] = [
  {
    id: 'ceremony',
    title: 'Ceremony',
    items: [
      { id: 'officiant', label: 'Book officiant (Steve?)', priority: 'high' },
      { id: 'vows', label: 'Write vows' },
      { id: 'ceremony-musician', label: 'Book musician for ceremony' },
      { id: 'cocktail-record-player', label: 'Decide: record player for cocktail hour?' },
      { id: 'order-of-service', label: 'Finalize order of service / readings' },
      { id: 'ring-bearer-flower-girl', label: 'Ring bearer & flower girl — confirm + outfits' },
      { id: 'kids-only-ceremony', label: 'How to explain kids-only ceremony to guests' },
      { id: 'welcome-sign', label: 'Welcome sign' },
    ],
  },
  {
    id: 'reception',
    title: 'Reception',
    items: [
      { id: 'seating-chart', label: 'Seating chart', priority: 'high' },
      { id: 'menu-tasting', label: 'Book menu tasting', priority: 'high' },
      { id: 'meals-contact-kim', label: 'Figure out meals / contact Kim', priority: 'high' },
      { id: 'reception-entrance', label: 'Plan reception entrance (bride/groom + parties)' },
      { id: 'gifts-cards-table', label: 'Where to put gifts & cards table' },
      { id: 'photo-booth', label: 'Photo booth — DIY or rent?', priority: 'high' },
      { id: 'champagne', label: 'Champagne plan' },
      { id: 'bathroom-bins', label: 'Bathroom bins' },
      { id: 'coat-check', label: 'Coat check' },
      { id: 'bud-vases', label: 'Bud vases — who fills & where' },
      { id: 'dinner-kissing-game', label: 'Dinner kissing game' },
      { id: 'wedding-favours', label: 'Wedding favours?' },
      { id: 'update-zola-food', label: 'Update Zola website with food info' },
    ],
  },
  {
    id: 'people-roles',
    title: 'People & Roles',
    items: [
      { id: 'mcs', label: 'Confirm MCs' },
      { id: 'speeches', label: 'Confirm speeches (parents, best man, maid of honour)' },
      { id: 'mics-schedule', label: '2 mics + schedule of dinner speeches' },
      { id: 'bridesmaid-roles', label: 'Assign bridesmaid roles' },
      { id: 'groomsmen-roles', label: 'Assign groomsmen roles' },
      { id: 'speech-requirements', label: 'Speech requirements — send to speakers' },
      { id: 'thank-you-speech', label: 'Thank you speech (LP + Anna)' },
      { id: 'thank-you-gifts-boys', label: 'Thank you gifts — boys (Casio A158WA-1 × 4)' },
      { id: 'thank-you-gifts-girls', label: 'Thank you gifts — girls (× 4)' },
      { id: 'who-picks-outfits', label: 'Decide who picks outfits for wedding party' },
      { id: 'gifts-wedding-party', label: 'Gifts for wedding party' },
    ],
  },
  {
    id: 'logistics',
    title: 'Logistics',
    items: [
      { id: 'day-of-timeline', label: 'Build day-of timeline (getting ready + photography)', priority: 'high' },
      { id: 'send-timeline-vendors', label: 'Send timeline to all vendors', priority: 'high' },
      { id: 'rehearsal-dinner', label: 'Rehearsal dinner — plan & book' },
      { id: 'send-invitations', label: 'Send invitations', priority: 'high' },
      { id: 'bridal-party-transport', label: 'Bridal party transportation' },
      { id: 'setup-teardown-team', label: 'Set-up + tear-down team — confirm' },
      { id: 'venue-open-time', label: 'When does venue open for vendors?' },
      { id: 'dj-songs', label: 'DJ songs list' },
      { id: 'first-dance', label: 'First dance — decide song + book classes?' },
      { id: 'parent-dances', label: 'Parent dances — decide songs' },
      { id: 'guestbook', label: 'Where do guests sign (guestbook)?' },
      { id: 'hashtag-photo-app', label: 'Hashtag + photo sharing app for guests' },
      { id: 'sunday-brunch', label: 'Sunday brunch (day after) — book?' },
    ],
  },
  {
    id: 'attire-beauty',
    title: 'Attire & Beauty',
    items: [
      { id: 'confirm-hair-makeup', label: 'Confirm hair & makeup' },
      { id: 'hair-makeup-trial', label: 'Hair & makeup trial' },
      { id: 'dress-alterations', label: 'Dress alterations' },
      { id: 'heels', label: 'Heels' },
      { id: 'wedding-jewellery', label: 'Wedding jewellery' },
      { id: 'eyelashes-nails', label: 'Eyelashes & nails' },
      { id: 'bridesmaid-swatches', label: 'Bridesmaid colour swatches' },
      { id: 'photo-shot-list', label: 'Photography shot list + locations', priority: 'high' },
    ],
  },
]
```

- [ ] **Step 4: Write src/data/budget.ts**

```typescript
export type BudgetStatus = 'Paid' | 'In Progress' | 'TBD'

export interface BudgetRow {
  id: string
  service: string
  status: BudgetStatus
  paid: number | null
  owing: number | null
  total: number | null
  notes?: string
}

export const BUDGET_ROWS: BudgetRow[] = [
  { id: 'venue', service: 'Venue + Food + Drinks', status: 'In Progress', paid: 2500, owing: null, total: null, notes: 'Tax + gratuity included' },
  { id: 'ceremony-setup', service: 'Ceremony Setup', status: 'In Progress', paid: 0, owing: null, total: null },
  { id: 'photographer', service: 'Photographer', status: 'In Progress', paid: 500, owing: null, total: 3165, notes: 'Southern Ontario Lady' },
  { id: 'florals', service: 'Florals', status: 'Paid', paid: 994, owing: 0, total: 994, notes: 'Tax + gratuity included' },
  { id: 'decor', service: 'Decor', status: 'In Progress', paid: 0, owing: null, total: null },
  { id: 'music', service: 'Music', status: 'In Progress', paid: 450, owing: null, total: 1920, notes: 'Tax included' },
  { id: 'brides-attire', service: "Bride's Attire", status: 'Paid', paid: 2017, owing: 0, total: 2017 },
  { id: 'bridesmaid-swatches', service: 'Bridesmaid Colour Swatches', status: 'In Progress', paid: 0, owing: null, total: null },
  { id: 'grooms-attire', service: "Groom's Attire", status: 'Paid', paid: 835, owing: 0, total: 835, notes: 'Suit Supply' },
  { id: 'alterations', service: 'Fittings / Alterations', status: 'In Progress', paid: 0, owing: null, total: 200, notes: 'Fayez' },
  { id: 'stationery', service: 'Stationery', status: 'Paid', paid: 350, owing: 0, total: 350, notes: 'Naomi' },
  { id: 'stamps', service: 'Stamps', status: 'Paid', paid: 56, owing: 0, total: 56 },
  { id: 'rings', service: 'Rings', status: 'Paid', paid: 3265.70, owing: 0, total: 3265.70, notes: 'Goldform' },
  { id: 'transportation', service: 'Transportation', status: 'In Progress', paid: 0, owing: null, total: null },
  { id: 'hair-makeup', service: 'Hair / Makeup', status: 'In Progress', paid: 0, owing: null, total: 327, notes: 'Tax + gratuity included' },
  { id: 'officiant', service: 'Officiant', status: 'In Progress', paid: 0, owing: null, total: 200, notes: 'Tax + gratuity included' },
  { id: 'wedding-hotel', service: 'Wedding Night Hotel', status: 'In Progress', paid: 0, owing: null, total: null },
  { id: 'day-after-brunch', service: 'Day-After Brunch', status: 'In Progress', paid: 0, owing: null, total: 100 },
  { id: 'gifts-boys', service: 'Thank You Gifts — Boys', status: 'In Progress', paid: 0, owing: null, total: 181, notes: 'Casio A158WA-1 × 4' },
  { id: 'gifts-girls', service: 'Thank You Gifts — Girls', status: 'In Progress', paid: 0, owing: null, total: 181 },
  { id: 'honeymoon', service: 'Honeymoon', status: 'TBD', paid: null, owing: null, total: null },
]

export interface Vendor {
  id: string
  name: string
  category: string
}

export const VENDORS: Vendor[] = [
  { id: 'venue', name: 'Venue', category: 'Venue' },
  { id: 'photographer', name: 'Photographer', category: 'Photography' },
  { id: 'florals', name: 'Florals', category: 'Florals' },
  { id: 'music-dj', name: 'Music / DJ', category: 'Music' },
  { id: 'officiant', name: 'Officiant', category: 'Ceremony' },
  { id: 'hair-makeup', name: 'Hair & Makeup', category: 'Beauty' },
  { id: 'alterations', name: 'Alterations (Fayez)', category: 'Attire' },
  { id: 'rings', name: 'Rings (Goldform)', category: 'Jewellery' },
  { id: 'stationery', name: 'Stationery (Naomi)', category: 'Stationery' },
  { id: 'transportation', name: 'Transportation', category: 'Logistics' },
]
```

- [ ] **Step 5: Run tests to confirm they pass**

```bash
npx jest src/data
```
Expected: PASS — 5 tests passing

- [ ] **Step 6: Commit**

```bash
git add src/data/ jest.config.ts && git commit -m "Add todo and budget data with integrity tests"
```

---

### Task 3: useLocalStorage hook

**Files:**
- Create: `src/hooks/useLocalStorage.ts`
- Create: `src/hooks/__tests__/useLocalStorage.test.ts`

**Interfaces:**
- Consumes: nothing
- Produces:
  - `useTodoState(): { checked: Record<string, boolean>; toggle: (id: string) => void; mounted: boolean }`
  - `useVendorState(): { vendors: Record<string, VendorContact>; update: (id: string, field: 'phone' | 'email' | 'notes', value: string) => void; mounted: boolean }`
  - `VendorContact` type: `{ phone: string; email: string; notes: string }`

- [ ] **Step 1: Write failing test**

Create `src/hooks/__tests__/useLocalStorage.test.ts`:
```typescript
// Use node environment (no DOM needed — we test the pure logic)

const STORAGE_KEY = 'wickham-wedding'

beforeEach(() => {
  // Simple in-memory localStorage mock
  const store: Record<string, string> = {}
  global.localStorage = {
    getItem: (k: string) => store[k] ?? null,
    setItem: (k: string, v: string) => { store[k] = v },
    removeItem: (k: string) => { delete store[k] },
    clear: () => Object.keys(store).forEach(k => delete store[k]),
    length: 0,
    key: () => null,
  }
})

it('reads empty state when localStorage is empty', () => {
  localStorage.clear()
  const raw = localStorage.getItem(STORAGE_KEY)
  expect(raw).toBeNull()
})

it('writes and reads back todo state', () => {
  const data = { todo: { 'seating-chart': true }, vendors: {} }
  localStorage.setItem(STORAGE_KEY, JSON.stringify(data))
  const back = JSON.parse(localStorage.getItem(STORAGE_KEY)!)
  expect(back.todo['seating-chart']).toBe(true)
})

it('merges todo state without clobbering vendor state', () => {
  const initial = {
    todo: {},
    vendors: { venue: { phone: '613-555-0000', email: '', notes: '' } },
  }
  localStorage.setItem(STORAGE_KEY, JSON.stringify(initial))
  const data = JSON.parse(localStorage.getItem(STORAGE_KEY)!)
  const next = { ...data, todo: { 'seating-chart': true } }
  localStorage.setItem(STORAGE_KEY, JSON.stringify(next))
  const back = JSON.parse(localStorage.getItem(STORAGE_KEY)!)
  expect(back.vendors.venue.phone).toBe('613-555-0000')
  expect(back.todo['seating-chart']).toBe(true)
})
```

- [ ] **Step 2: Run test to confirm it fails**

```bash
npx jest src/hooks
```
Expected: PASS (these tests don't import the hook yet — they test localStorage directly). All 3 should pass since they only use the mock.

- [ ] **Step 3: Write src/hooks/useLocalStorage.ts**

```typescript
'use client'

import { useState, useEffect } from 'react'

const STORAGE_KEY = 'wickham-wedding'

export interface VendorContact {
  phone: string
  email: string
  notes: string
}

type StorageData = {
  todo: Record<string, boolean>
  vendors: Record<string, VendorContact>
}

const DEFAULT_DATA: StorageData = { todo: {}, vendors: {} }

function readStorage(): StorageData {
  if (typeof window === 'undefined') return DEFAULT_DATA
  try {
    const raw = localStorage.getItem(STORAGE_KEY)
    return raw ? (JSON.parse(raw) as StorageData) : DEFAULT_DATA
  } catch {
    return DEFAULT_DATA
  }
}

function writeStorage(data: StorageData): void {
  if (typeof window === 'undefined') return
  localStorage.setItem(STORAGE_KEY, JSON.stringify(data))
}

export function useTodoState() {
  const [checked, setChecked] = useState<Record<string, boolean>>({})
  const [mounted, setMounted] = useState(false)

  useEffect(() => {
    setChecked(readStorage().todo)
    setMounted(true)
  }, [])

  function toggle(id: string) {
    const next = { ...checked, [id]: !checked[id] }
    setChecked(next)
    writeStorage({ ...readStorage(), todo: next })
  }

  return { checked, toggle, mounted }
}

export function useVendorState() {
  const [vendors, setVendors] = useState<Record<string, VendorContact>>({})
  const [mounted, setMounted] = useState(false)

  useEffect(() => {
    setVendors(readStorage().vendors)
    setMounted(true)
  }, [])

  function update(id: string, field: 'phone' | 'email' | 'notes', value: string) {
    const current = vendors[id] ?? { phone: '', email: '', notes: '' }
    const next = { ...vendors, [id]: { ...current, [field]: value } }
    setVendors(next)
    writeStorage({ ...readStorage(), vendors: next })
  }

  return { vendors, update, mounted }
}
```

- [ ] **Step 4: Run all tests**

```bash
npx jest
```
Expected: PASS — all 8 tests passing

- [ ] **Step 5: Commit**

```bash
git add src/hooks/ && git commit -m "Add useLocalStorage hook for todo and vendor state"
```

---

### Task 4: Hero component

**Files:**
- Create: `src/components/Hero.tsx`

**Interfaces:**
- Consumes: nothing (hardcoded wedding date)
- Produces: `<Hero />` — no props

- [ ] **Step 1: Create src/components/Hero.tsx**

```tsx
'use client'

import { useState, useEffect } from 'react'

const WEDDING_DATE = new Date('2026-10-17T00:00:00')

function getTimeLeft() {
  const diff = WEDDING_DATE.getTime() - Date.now()
  if (diff <= 0) return { days: 0, hours: 0, minutes: 0, seconds: 0 }
  return {
    days: Math.floor(diff / (1000 * 60 * 60 * 24)),
    hours: Math.floor((diff % (1000 * 60 * 60 * 24)) / (1000 * 60 * 60)),
    minutes: Math.floor((diff % (1000 * 60 * 60)) / (1000 * 60)),
    seconds: Math.floor((diff % (1000 * 60)) / 1000),
  }
}

export default function Hero() {
  const [time, setTime] = useState(getTimeLeft())

  useEffect(() => {
    const id = setInterval(() => setTime(getTimeLeft()), 1000)
    return () => clearInterval(id)
  }, [])

  const units = [
    { value: time.days, label: 'Days' },
    { value: time.hours, label: 'Hours' },
    { value: time.minutes, label: 'Minutes' },
    { value: time.seconds, label: 'Seconds' },
  ]

  return (
    <header className="py-16 text-center border-b border-stone-200">
      <p className="text-xs tracking-widest uppercase text-rose-400 mb-4">Ottawa, Ontario</p>
      <h1 className="font-serif text-5xl md:text-6xl text-stone-800 mb-3">
        Louis-Philippe &amp; Anna
      </h1>
      <p className="text-stone-400 text-lg mb-12">October 17, 2026</p>

      <div className="flex justify-center gap-6 md:gap-10 mb-10">
        {units.map(({ value, label }) => (
          <div key={label} className="flex flex-col items-center">
            <span className="font-serif text-4xl md:text-5xl text-stone-800 tabular-nums w-16 md:w-20 text-center">
              {String(value).padStart(2, '0')}
            </span>
            <span className="text-xs uppercase tracking-widest text-stone-400 mt-1">{label}</span>
          </div>
        ))}
      </div>

      <p className="text-stone-400 text-sm tracking-wide">#wickhamwedding2026</p>
    </header>
  )
}
```

- [ ] **Step 2: Commit**

```bash
git add src/components/Hero.tsx && git commit -m "Add Hero component with live countdown"
```

---

### Task 5: TodoItem and TodoSection

**Files:**
- Create: `src/components/TodoItem.tsx`
- Create: `src/components/TodoSection.tsx`

**Interfaces:**
- Consumes:
  - `TODO_CATEGORIES` from `@/data/todos`
  - `useTodoState()` from `@/hooks/useLocalStorage`
- Produces:
  - `<TodoItem id checked onToggle label priority? />` — no default export props type
  - `<TodoSection />` — no props

- [ ] **Step 1: Create src/components/TodoItem.tsx**

```tsx
interface Props {
  id: string
  label: string
  priority?: 'high' | 'normal'
  checked: boolean
  onToggle: (id: string) => void
}

export default function TodoItem({ id, label, priority, checked, onToggle }: Props) {
  return (
    <li className="flex items-start gap-3 py-2.5">
      <button
        onClick={() => onToggle(id)}
        aria-label={checked ? 'Mark incomplete' : 'Mark complete'}
        className={`mt-0.5 flex-shrink-0 w-5 h-5 rounded border-2 flex items-center justify-center transition-colors ${
          checked
            ? 'bg-rose-400 border-rose-400'
            : 'border-stone-300 hover:border-rose-300'
        }`}
      >
        {checked && (
          <svg
            className="w-3 h-3 text-white"
            fill="none"
            viewBox="0 0 24 24"
            stroke="currentColor"
            strokeWidth={3}
          >
            <path strokeLinecap="round" strokeLinejoin="round" d="M5 13l4 4L19 7" />
          </svg>
        )}
      </button>

      <span
        className={`text-sm leading-relaxed flex-1 ${
          checked ? 'line-through text-stone-400' : 'text-stone-700'
        }`}
      >
        {label}
      </span>

      {priority === 'high' && !checked && (
        <span className="flex-shrink-0 text-xs px-1.5 py-0.5 rounded bg-rose-100 text-rose-500 font-medium">
          Priority
        </span>
      )}
    </li>
  )
}
```

- [ ] **Step 2: Create src/components/TodoSection.tsx**

```tsx
'use client'

import { TODO_CATEGORIES } from '@/data/todos'
import { useTodoState } from '@/hooks/useLocalStorage'
import TodoItem from './TodoItem'

export default function TodoSection() {
  const { checked, toggle, mounted } = useTodoState()

  const allItems = TODO_CATEGORIES.flatMap(c => c.items)
  const doneCount = mounted ? allItems.filter(i => checked[i.id]).length : 0

  return (
    <section className="py-12">
      <div className="flex items-baseline justify-between mb-8">
        <h2 className="font-serif text-3xl text-stone-800">To-Do</h2>
        {mounted && (
          <span className="text-sm text-stone-400">
            {doneCount} / {allItems.length} done
          </span>
        )}
      </div>

      <div className="grid grid-cols-1 md:grid-cols-2 gap-6">
        {TODO_CATEGORIES.map(category => (
          <div key={category.id} className="bg-stone-50 rounded-2xl p-6 border border-stone-100">
            <h3 className="text-xs font-semibold uppercase tracking-widest text-stone-400 mb-3">
              {category.title}
            </h3>
            <ul className="divide-y divide-stone-100">
              {category.items.map(item => (
                <TodoItem
                  key={item.id}
                  id={item.id}
                  label={item.label}
                  priority={item.priority}
                  checked={mounted ? !!checked[item.id] : false}
                  onToggle={toggle}
                />
              ))}
            </ul>
          </div>
        ))}
      </div>
    </section>
  )
}
```

- [ ] **Step 3: Commit**

```bash
git add src/components/TodoItem.tsx src/components/TodoSection.tsx && git commit -m "Add TodoItem and TodoSection components"
```

---

### Task 6: BudgetTable

**Files:**
- Create: `src/components/BudgetTable.tsx`

**Interfaces:**
- Consumes: `BUDGET_ROWS: BudgetRow[]`, `BudgetStatus` from `@/data/budget`
- Produces: `<BudgetTable />` — no props

- [ ] **Step 1: Create src/components/BudgetTable.tsx**

```tsx
import { BUDGET_ROWS, BudgetStatus } from '@/data/budget'

const STATUS_DOT: Record<BudgetStatus, string> = {
  Paid: 'bg-green-400',
  'In Progress': 'bg-amber-400',
  TBD: 'bg-stone-300',
}

function StatusDot({ status }: { status: BudgetStatus }) {
  return (
    <span className="flex items-center gap-2">
      <span className={`inline-block w-2 h-2 rounded-full ${STATUS_DOT[status]}`} />
      <span className="text-stone-600">{status}</span>
    </span>
  )
}

function fmt(val: number | null): string {
  if (val === null) return '—'
  return `$${val.toLocaleString('en-CA', { minimumFractionDigits: 2, maximumFractionDigits: 2 })}`
}

export default function BudgetTable() {
  const totalPaid = BUDGET_ROWS.reduce((s, r) => s + (r.paid ?? 0), 0)
  const totalOwing = BUDGET_ROWS.reduce((s, r) => s + (r.owing ?? 0), 0)
  const totalCost = BUDGET_ROWS.reduce((s, r) => s + (r.total ?? 0), 0)

  return (
    <section className="py-12">
      <h2 className="font-serif text-3xl text-stone-800 mb-8">Budget</h2>
      <div className="overflow-x-auto rounded-2xl border border-stone-200">
        <table className="w-full text-sm">
          <thead className="bg-stone-50">
            <tr>
              {['Service', 'Status', 'Paid', 'Still Owing', 'Total', 'Notes'].map(h => (
                <th
                  key={h}
                  className={`px-4 py-3 text-xs font-semibold uppercase tracking-widest text-stone-400 ${
                    ['Paid', 'Still Owing', 'Total'].includes(h) ? 'text-right' : 'text-left'
                  }`}
                >
                  {h}
                </th>
              ))}
            </tr>
          </thead>
          <tbody className="divide-y divide-stone-100">
            {BUDGET_ROWS.map(row => (
              <tr key={row.id} className="hover:bg-stone-50 transition-colors">
                <td className="px-4 py-3 text-stone-700 font-medium">{row.service}</td>
                <td className="px-4 py-3"><StatusDot status={row.status} /></td>
                <td className="px-4 py-3 text-right text-stone-600">{fmt(row.paid)}</td>
                <td className="px-4 py-3 text-right text-stone-600">{fmt(row.owing)}</td>
                <td className="px-4 py-3 text-right text-stone-600">{fmt(row.total)}</td>
                <td className="px-4 py-3 text-stone-400 text-xs">{row.notes ?? ''}</td>
              </tr>
            ))}
          </tbody>
          <tfoot className="bg-stone-50">
            <tr className="font-semibold text-stone-700">
              <td className="px-4 py-3" colSpan={2}>Totals</td>
              <td className="px-4 py-3 text-right">{fmt(totalPaid)}</td>
              <td className="px-4 py-3 text-right">{fmt(totalOwing)}</td>
              <td className="px-4 py-3 text-right">{fmt(totalCost)}</td>
              <td className="px-4 py-3" />
            </tr>
          </tfoot>
        </table>
      </div>
    </section>
  )
}
```

- [ ] **Step 2: Commit**

```bash
git add src/components/BudgetTable.tsx && git commit -m "Add BudgetTable component"
```

---

### Task 7: VendorContacts

**Files:**
- Create: `src/components/VendorContacts.tsx`

**Interfaces:**
- Consumes:
  - `VENDORS: Vendor[]` from `@/data/budget`
  - `useVendorState()` from `@/hooks/useLocalStorage` — returns `{ vendors, update, mounted }`
  - `update(id: string, field: 'phone' | 'email' | 'notes', value: string) => void`
- Produces: `<VendorContacts />` — no props

- [ ] **Step 1: Create src/components/VendorContacts.tsx**

```tsx
'use client'

import { VENDORS } from '@/data/budget'
import { useVendorState } from '@/hooks/useLocalStorage'

const FIELD_PLACEHOLDERS = {
  phone: '613-000-0000',
  email: 'name@example.com',
  notes: 'Notes...',
}

export default function VendorContacts() {
  const { vendors, update, mounted } = useVendorState()

  return (
    <section className="py-12">
      <h2 className="font-serif text-3xl text-stone-800 mb-8">Vendors</h2>
      <div className="grid grid-cols-1 sm:grid-cols-2 md:grid-cols-3 gap-4">
        {VENDORS.map(vendor => {
          const contact = mounted
            ? (vendors[vendor.id] ?? { phone: '', email: '', notes: '' })
            : { phone: '', email: '', notes: '' }

          return (
            <div
              key={vendor.id}
              className="bg-stone-50 rounded-2xl p-5 border border-stone-100"
            >
              <p className="text-xs uppercase tracking-widest text-rose-400 mb-1">
                {vendor.category}
              </p>
              <h3 className="font-serif text-lg text-stone-800 mb-4">{vendor.name}</h3>

              <div className="space-y-3">
                {(['phone', 'email', 'notes'] as const).map(field => (
                  <div key={field}>
                    <label className="block text-xs text-stone-400 uppercase tracking-wide mb-1 capitalize">
                      {field}
                    </label>
                    <input
                      type={field === 'email' ? 'email' : 'text'}
                      value={contact[field]}
                      onChange={e => update(vendor.id, field, e.target.value)}
                      placeholder={FIELD_PLACEHOLDERS[field]}
                      className="w-full text-sm bg-white border border-stone-200 rounded-lg px-3 py-2 text-stone-700 placeholder-stone-300 focus:outline-none focus:border-rose-300 focus:ring-1 focus:ring-rose-200 transition-colors"
                    />
                  </div>
                ))}
              </div>
            </div>
          )
        })}
      </div>
    </section>
  )
}
```

- [ ] **Step 2: Commit**

```bash
git add src/components/VendorContacts.tsx && git commit -m "Add VendorContacts component with localStorage persistence"
```

---

### Task 8: Page assembly, layout, and README

**Files:**
- Modify: `src/app/layout.tsx`
- Modify: `src/app/page.tsx`
- Create: `README.md`

**Interfaces:**
- Consumes: `Hero`, `TodoSection`, `BudgetTable`, `VendorContacts`
- Produces: complete running app

- [ ] **Step 1: Replace src/app/layout.tsx**

```tsx
import type { Metadata } from 'next'
import { Inter, Playfair_Display } from 'next/font/google'
import './globals.css'

const inter = Inter({
  subsets: ['latin'],
  variable: '--font-inter',
})

const playfair = Playfair_Display({
  subsets: ['latin'],
  variable: '--font-playfair',
})

export const metadata: Metadata = {
  title: 'LP & Anna — Wedding Dashboard',
}

export default function RootLayout({
  children,
}: {
  children: React.ReactNode
}) {
  return (
    <html lang="en" className={`${inter.variable} ${playfair.variable}`}>
      <body className="bg-[#FAF8F4] font-sans antialiased">{children}</body>
    </html>
  )
}
```

- [ ] **Step 2: Replace src/app/page.tsx**

```tsx
import Hero from '@/components/Hero'
import TodoSection from '@/components/TodoSection'
import BudgetTable from '@/components/BudgetTable'
import VendorContacts from '@/components/VendorContacts'

export default function Home() {
  return (
    <main className="max-w-5xl mx-auto px-4 md:px-8 pb-24">
      <Hero />
      <TodoSection />
      <div className="border-t border-stone-200" />
      <BudgetTable />
      <div className="border-t border-stone-200" />
      <VendorContacts />
    </main>
  )
}
```

- [ ] **Step 3: Write README.md**

```markdown
# Wedding Dashboard

LP & Anna's private wedding planning hub. October 17, 2026.

## How to run

```bash
npm install
npm run dev
```

Then open http://localhost:3000 in your browser.

## What's in it

- **Countdown** to October 17, 2026
- **To-Do checklist** — check things off as you go (saves automatically)
- **Budget tracker** — all your vendors and what's been paid
- **Vendor contacts** — phone, email, notes for each vendor (saves automatically)

## Adding or changing items

- To-do items: edit `src/data/todos.ts`
- Budget rows: edit `src/data/budget.ts`
- Vendor cards: edit `src/data/budget.ts` (the `VENDORS` list)
```

- [ ] **Step 4: Run the app and verify it looks correct**

```bash
npm run dev
```

Open http://localhost:3000. Verify:
- Countdown is ticking
- All 5 to-do categories appear with their items
- Priority badges show on starred items
- Budget table has all 21 rows and a totals row
- Vendor contact cards appear and fields are editable
- Check a few todo boxes, refresh the page — they stay checked

- [ ] **Step 5: Run all tests one final time**

```bash
npx jest
```
Expected: PASS — all 8 tests passing, no errors.

- [ ] **Step 6: Final commit**

```bash
git add -A && git commit -m "Assemble full wedding dashboard — ready to use"
```
