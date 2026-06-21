# LP's Style Book — Plan 3A: Item Tracking

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add wear logging, LP reviews, and failed-purchase archiving directly to the item detail page, visible only when logged in.

**Architecture:** Three server actions handle all mutations. A single client component (`ItemActionPanel`) manages a four-state UI (idle → wear-form | review-form | failed-form) inline on the item detail page — no page navigation. The existing server component checks auth and passes `isAdmin` + `existingReview` to the panel. The ItemCard gets a small "Archived" badge for archived items.

**Tech Stack:** Next.js 14 App Router, Supabase, React `useTransition`, `revalidatePath`, TypeScript strict, React Testing Library

## Global Constraints

- TypeScript strict mode — no `any`
- Tailwind classes only, warm minimal palette: `bg-cream` / `text-espresso` / `text-tan` / `text-warm`
- Fonts: `font-serif` (EB Garamond) for headings, `font-sans` (Inter) for body
- Server actions use `'use server'` directive, import `createClient` from `@/lib/supabase/server`
- All mutations: check `supabase.auth.getUser()` first, redirect to `/login` if no user
- All mutations: call `revalidatePath('/items/${itemId}')` after successful DB write
- npm only

## File Map

| File | Change |
|------|--------|
| `src/app/actions/logWear.ts` | Create — inserts `wear_records` row |
| `src/app/actions/saveReview.ts` | Create — upserts `lp_reviews` row |
| `src/app/actions/archiveItem.ts` | Create — inserts `failed_purchases` row, sets `item.status = 'archived'` |
| `src/components/ItemActionPanel.tsx` | Create — client component, 4-state inline form |
| `src/app/items/[id]/page.tsx` | Modify — add auth check, render `<ItemActionPanel>` |
| `src/components/ItemCard.tsx` | Modify — add Archived badge when `status === 'archived'` |
| `__tests__/actions/itemTracking.test.ts` | Create — auth redirect tests + DB call verification |
| `__tests__/components/ItemActionPanel.test.tsx` | Create — form toggle and pre-fill tests |

## Existing types (already in `src/types/index.ts` — do NOT re-define)

```typescript
type WearRecord = {
  id: string; item_id: string; worn_on: string
  season: 'Spring' | 'Summer' | 'Autumn' | 'Winter' | null
  occasion: 'Casual' | 'Smart Casual' | 'Work' | 'Formal' | 'Sport' | null
  created_at: string
}
type LPReview = {
  id: string; item_id: string
  fit: number; comfort: number; quality: number; versatility: number; value: number
  notes: string | null; reviewed_at: string
}
type FailedPurchase = {
  id: string; item_id: string
  reason: 'poor fit' | 'poor quality' | 'poor value' | 'rarely worn' | "doesn't suit style" | 'uncomfortable' | 'impulse purchase' | 'other'
  notes: string | null; created_at: string
}
```

---

### Task 1: Three server actions and tests

**Files:**
- Create: `src/app/actions/logWear.ts`
- Create: `src/app/actions/saveReview.ts`
- Create: `src/app/actions/archiveItem.ts`
- Create: `__tests__/actions/itemTracking.test.ts`

**Interfaces:**
- Consumes: `createClient` from `@/lib/supabase/server`, `revalidatePath` from `next/cache`, `redirect` from `next/navigation`, types from `@/types`
- Produces:
  ```typescript
  // src/app/actions/logWear.ts
  export type LogWearInput = {
    itemId: string
    wornOn: string           // YYYY-MM-DD
    season: WearRecord['season']
    occasion: WearRecord['occasion']
  }
  export async function logWear(input: LogWearInput): Promise<void>

  // src/app/actions/saveReview.ts
  export type SaveReviewInput = {
    itemId: string
    fit: number; comfort: number; quality: number; versatility: number; value: number
    notes: string
  }
  export async function saveReview(input: SaveReviewInput): Promise<void>

  // src/app/actions/archiveItem.ts
  export type ArchiveItemInput = {
    itemId: string
    reason: FailedPurchase['reason']
    notes: string
  }
  export async function archiveItem(input: ArchiveItemInput): Promise<void>
  ```

- [ ] **Step 1: Write failing tests**

  Create `__tests__/actions/itemTracking.test.ts`:

  ```typescript
  // __tests__/actions/itemTracking.test.ts
  jest.mock('next/cache', () => ({ revalidatePath: jest.fn() }))
  jest.mock('next/navigation', () => ({ redirect: jest.fn() }))
  jest.mock('@/lib/supabase/server')

  import { logWear } from '@/app/actions/logWear'
  import { saveReview } from '@/app/actions/saveReview'
  import { archiveItem } from '@/app/actions/archiveItem'
  import { revalidatePath } from 'next/cache'
  import { redirect } from 'next/navigation'
  import { createClient } from '@/lib/supabase/server'

  const mockedCreateClient = createClient as jest.MockedFunction<typeof createClient>
  const mockedRevalidatePath = revalidatePath as jest.MockedFunction<typeof revalidatePath>
  const mockedRedirect = redirect as jest.MockedFunction<typeof redirect>

  type MockClient = Awaited<ReturnType<typeof createClient>>

  function makeClient(authenticated: boolean, existingReview: { id: string } | null = null) {
    const mockMaybeSingle = jest.fn().mockResolvedValue({ data: existingReview, error: null })
    const mockEq = jest.fn(() => ({ maybeSingle: mockMaybeSingle }))
    const mockSelect = jest.fn(() => ({ eq: mockEq }))
    const mockInsert = jest.fn().mockResolvedValue({ error: null })
    const mockEqUpdate = jest.fn().mockResolvedValue({ error: null })
    const mockUpdate = jest.fn(() => ({ eq: mockEqUpdate }))
    const mockFrom = jest.fn(() => ({ insert: mockInsert, select: mockSelect, update: mockUpdate }))

    const client = {
      auth: { getUser: jest.fn().mockResolvedValue({ data: { user: authenticated ? { id: 'u1' } : null } }) },
      from: mockFrom,
    }
    mockedCreateClient.mockResolvedValue(client as unknown as MockClient)
    return { mockFrom, mockInsert, mockUpdate, mockEqUpdate }
  }

  beforeEach(() => jest.clearAllMocks())

  describe('logWear', () => {
    it('redirects to /login when unauthenticated', async () => {
      makeClient(false)
      await logWear({ itemId: 'i1', wornOn: '2026-06-21', season: 'Summer', occasion: 'Casual' })
      expect(mockedRedirect).toHaveBeenCalledWith('/login')
    })

    it('inserts into wear_records and revalidates the item page', async () => {
      const { mockFrom, mockInsert } = makeClient(true)
      await logWear({ itemId: 'i1', wornOn: '2026-06-21', season: 'Summer', occasion: 'Casual' })
      expect(mockFrom).toHaveBeenCalledWith('wear_records')
      expect(mockInsert).toHaveBeenCalledWith({
        item_id: 'i1', worn_on: '2026-06-21', season: 'Summer', occasion: 'Casual',
      })
      expect(mockedRevalidatePath).toHaveBeenCalledWith('/items/i1')
    })

    it('stores null season when none selected', async () => {
      const { mockInsert } = makeClient(true)
      await logWear({ itemId: 'i1', wornOn: '2026-06-21', season: null, occasion: null })
      expect(mockInsert).toHaveBeenCalledWith(expect.objectContaining({ season: null, occasion: null }))
    })
  })

  describe('saveReview', () => {
    const reviewInput: Parameters<typeof saveReview>[0] = {
      itemId: 'i1', fit: 8, comfort: 7, quality: 9, versatility: 6, value: 7, notes: 'Great',
    }

    it('redirects to /login when unauthenticated', async () => {
      makeClient(false)
      await saveReview(reviewInput)
      expect(mockedRedirect).toHaveBeenCalledWith('/login')
    })

    it('inserts a new review when none exists', async () => {
      const { mockFrom, mockInsert } = makeClient(true, null)
      await saveReview(reviewInput)
      expect(mockFrom).toHaveBeenCalledWith('lp_reviews')
      expect(mockInsert).toHaveBeenCalledWith(expect.objectContaining({ item_id: 'i1', fit: 8 }))
      expect(mockedRevalidatePath).toHaveBeenCalledWith('/items/i1')
    })

    it('updates the existing review when one exists', async () => {
      const { mockUpdate, mockEqUpdate } = makeClient(true, { id: 'r1' })
      await saveReview(reviewInput)
      expect(mockUpdate).toHaveBeenCalledWith(expect.objectContaining({ fit: 8, comfort: 7 }))
      expect(mockEqUpdate).toHaveBeenCalledWith('id', 'r1')
    })
  })

  describe('archiveItem', () => {
    it('redirects to /login when unauthenticated', async () => {
      makeClient(false)
      await archiveItem({ itemId: 'i1', reason: 'poor fit', notes: '' })
      expect(mockedRedirect).toHaveBeenCalledWith('/login')
    })

    it('inserts into failed_purchases, updates item status, revalidates', async () => {
      const { mockFrom, mockInsert, mockUpdate } = makeClient(true)
      await archiveItem({ itemId: 'i1', reason: 'poor fit', notes: 'Too small' })
      expect(mockFrom).toHaveBeenCalledWith('failed_purchases')
      expect(mockInsert).toHaveBeenCalledWith(expect.objectContaining({ item_id: 'i1', reason: 'poor fit', notes: 'Too small' }))
      expect(mockFrom).toHaveBeenCalledWith('items')
      expect(mockUpdate).toHaveBeenCalledWith({ status: 'archived' })
      expect(mockedRevalidatePath).toHaveBeenCalledWith('/items/i1')
    })
  })
  ```

- [ ] **Step 2: Run tests to confirm they fail**

  ```bash
  npx jest --testPathPatterns="itemTracking" 2>&1 | tail -5
  ```

  Expected: FAIL — `Cannot find module '@/app/actions/logWear'`

- [ ] **Step 3: Create `src/app/actions/logWear.ts`**

  ```typescript
  // src/app/actions/logWear.ts
  'use server'

  import { revalidatePath } from 'next/cache'
  import { redirect } from 'next/navigation'
  import { createClient } from '@/lib/supabase/server'
  import type { WearRecord } from '@/types'

  export type LogWearInput = {
    itemId: string
    wornOn: string
    season: WearRecord['season']
    occasion: WearRecord['occasion']
  }

  export async function logWear(input: LogWearInput): Promise<void> {
    const supabase = await createClient()
    const { data: { user } } = await supabase.auth.getUser()
    if (!user) redirect('/login')

    const { error } = await supabase.from('wear_records').insert({
      item_id: input.itemId,
      worn_on: input.wornOn,
      season: input.season,
      occasion: input.occasion,
    })
    if (error) throw new Error(`Failed to log wear: ${error.message}`)
    revalidatePath(`/items/${input.itemId}`)
  }
  ```

- [ ] **Step 4: Create `src/app/actions/saveReview.ts`**

  ```typescript
  // src/app/actions/saveReview.ts
  'use server'

  import { revalidatePath } from 'next/cache'
  import { redirect } from 'next/navigation'
  import { createClient } from '@/lib/supabase/server'

  export type SaveReviewInput = {
    itemId: string
    fit: number
    comfort: number
    quality: number
    versatility: number
    value: number
    notes: string
  }

  export async function saveReview(input: SaveReviewInput): Promise<void> {
    const supabase = await createClient()
    const { data: { user } } = await supabase.auth.getUser()
    if (!user) redirect('/login')

    const { data: existing } = await supabase
      .from('lp_reviews')
      .select('id')
      .eq('item_id', input.itemId)
      .maybeSingle()

    const scores = {
      fit: input.fit,
      comfort: input.comfort,
      quality: input.quality,
      versatility: input.versatility,
      value: input.value,
      notes: input.notes.trim() || null,
    }

    if (existing) {
      const { error } = await supabase
        .from('lp_reviews')
        .update({ ...scores, reviewed_at: new Date().toISOString() })
        .eq('id', (existing as { id: string }).id)
      if (error) throw new Error(`Failed to update review: ${error.message}`)
    } else {
      const { error } = await supabase
        .from('lp_reviews')
        .insert({ item_id: input.itemId, ...scores })
      if (error) throw new Error(`Failed to save review: ${error.message}`)
    }

    revalidatePath(`/items/${input.itemId}`)
  }
  ```

- [ ] **Step 5: Create `src/app/actions/archiveItem.ts`**

  ```typescript
  // src/app/actions/archiveItem.ts
  'use server'

  import { revalidatePath } from 'next/cache'
  import { redirect } from 'next/navigation'
  import { createClient } from '@/lib/supabase/server'
  import type { FailedPurchase } from '@/types'

  export type ArchiveItemInput = {
    itemId: string
    reason: FailedPurchase['reason']
    notes: string
  }

  export async function archiveItem(input: ArchiveItemInput): Promise<void> {
    const supabase = await createClient()
    const { data: { user } } = await supabase.auth.getUser()
    if (!user) redirect('/login')

    const { error: failedError } = await supabase.from('failed_purchases').insert({
      item_id: input.itemId,
      reason: input.reason,
      notes: input.notes.trim() || null,
    })
    if (failedError) throw new Error(`Failed to log failed purchase: ${failedError.message}`)

    const { error: updateError } = await supabase
      .from('items')
      .update({ status: 'archived' })
      .eq('id', input.itemId)
    if (updateError) throw new Error(`Failed to archive item: ${updateError.message}`)

    revalidatePath(`/items/${input.itemId}`)
  }
  ```

- [ ] **Step 6: Run the tests to confirm they pass**

  ```bash
  npx jest --testPathPatterns="itemTracking" 2>&1 | tail -5
  ```

  Expected: 7 tests pass.

- [ ] **Step 7: Type-check**

  ```bash
  npx tsc --noEmit 2>&1
  ```

  Expected: no output.

- [ ] **Step 8: Commit**

  ```bash
  git add src/app/actions/logWear.ts src/app/actions/saveReview.ts src/app/actions/archiveItem.ts __tests__/actions/itemTracking.test.ts
  git commit -m "Add logWear, saveReview, and archiveItem server actions"
  ```

---

### Task 2: ItemActionPanel client component

**Files:**
- Create: `src/components/ItemActionPanel.tsx`
- Create: `__tests__/components/ItemActionPanel.test.tsx`

**Interfaces:**
- Consumes:
  ```typescript
  import { logWear, type LogWearInput } from '@/app/actions/logWear'
  import { saveReview, type SaveReviewInput } from '@/app/actions/saveReview'
  import { archiveItem, type ArchiveItemInput } from '@/app/actions/archiveItem'
  import type { LPReview, WearRecord, FailedPurchase } from '@/types'
  ```
- Produces:
  ```typescript
  // default export, used in Task 3 like this:
  <ItemActionPanel itemId={item.id} existingReview={latestReview ?? null} />
  // Props:
  type Props = { itemId: string; existingReview: LPReview | null }
  ```

- [ ] **Step 1: Write failing tests**

  Create `__tests__/components/ItemActionPanel.test.tsx`:

  ```typescript
  // __tests__/components/ItemActionPanel.test.tsx
  import { render, screen, fireEvent } from '@testing-library/react'
  import ItemActionPanel from '@/components/ItemActionPanel'
  import type { LPReview } from '@/types'

  jest.mock('@/app/actions/logWear', () => ({ logWear: jest.fn().mockResolvedValue(undefined) }))
  jest.mock('@/app/actions/saveReview', () => ({ saveReview: jest.fn().mockResolvedValue(undefined) }))
  jest.mock('@/app/actions/archiveItem', () => ({ archiveItem: jest.fn().mockResolvedValue(undefined) }))

  const existingReview: LPReview = {
    id: 'r1', item_id: 'item-1',
    fit: 9, comfort: 8, quality: 7, versatility: 6, value: 8,
    notes: 'Love it', reviewed_at: '2026-06-21T00:00:00Z',
  }

  describe('ItemActionPanel — idle state', () => {
    it('shows three action buttons', () => {
      render(<ItemActionPanel itemId="item-1" existingReview={null} />)
      expect(screen.getByText('Log Wear')).toBeInTheDocument()
      expect(screen.getByText('Write Review')).toBeInTheDocument()
      expect(screen.getByText('Mark as Failed')).toBeInTheDocument()
    })

    it('shows "Update Review" instead of "Write Review" when a review exists', () => {
      render(<ItemActionPanel itemId="item-1" existingReview={existingReview} />)
      expect(screen.getByText('Update Review')).toBeInTheDocument()
      expect(screen.queryByText('Write Review')).not.toBeInTheDocument()
    })
  })

  describe('ItemActionPanel — wear form', () => {
    it('shows wear form fields when Log Wear is clicked', () => {
      render(<ItemActionPanel itemId="item-1" existingReview={null} />)
      fireEvent.click(screen.getByText('Log Wear'))
      expect(screen.getByLabelText('Date')).toBeInTheDocument()
      expect(screen.getByLabelText('Season')).toBeInTheDocument()
      expect(screen.getByLabelText('Occasion')).toBeInTheDocument()
    })

    it('hides the action buttons when wear form is active', () => {
      render(<ItemActionPanel itemId="item-1" existingReview={null} />)
      fireEvent.click(screen.getByText('Log Wear'))
      expect(screen.queryByText('Write Review')).not.toBeInTheDocument()
      expect(screen.queryByText('Mark as Failed')).not.toBeInTheDocument()
    })

    it('returns to idle state when Cancel is clicked', () => {
      render(<ItemActionPanel itemId="item-1" existingReview={null} />)
      fireEvent.click(screen.getByText('Log Wear'))
      fireEvent.click(screen.getByText('Cancel'))
      expect(screen.getByText('Write Review')).toBeInTheDocument()
    })
  })

  describe('ItemActionPanel — review form', () => {
    it('shows review form fields when Write Review is clicked', () => {
      render(<ItemActionPanel itemId="item-1" existingReview={null} />)
      fireEvent.click(screen.getByText('Write Review'))
      expect(screen.getByLabelText('Fit')).toBeInTheDocument()
      expect(screen.getByLabelText('Comfort')).toBeInTheDocument()
      expect(screen.getByLabelText('Quality')).toBeInTheDocument()
      expect(screen.getByLabelText('Versatility')).toBeInTheDocument()
      expect(screen.getByLabelText('Value')).toBeInTheDocument()
    })

    it('pre-fills scores from existingReview', () => {
      render(<ItemActionPanel itemId="item-1" existingReview={existingReview} />)
      fireEvent.click(screen.getByText('Update Review'))
      expect(screen.getByLabelText('Fit')).toHaveValue(9)
      expect(screen.getByLabelText('Comfort')).toHaveValue(8)
      expect(screen.getByDisplayValue('Love it')).toBeInTheDocument()
    })
  })

  describe('ItemActionPanel — failed form', () => {
    it('shows failed form fields when Mark as Failed is clicked', () => {
      render(<ItemActionPanel itemId="item-1" existingReview={null} />)
      fireEvent.click(screen.getByText('Mark as Failed'))
      expect(screen.getByLabelText('Reason')).toBeInTheDocument()
      expect(screen.getByText('Archive Item')).toBeInTheDocument()
    })

    it('returns to idle state when Cancel is clicked', () => {
      render(<ItemActionPanel itemId="item-1" existingReview={null} />)
      fireEvent.click(screen.getByText('Mark as Failed'))
      fireEvent.click(screen.getByText('Cancel'))
      expect(screen.getByText('Log Wear')).toBeInTheDocument()
    })
  })
  ```

- [ ] **Step 2: Run tests to confirm they fail**

  ```bash
  npx jest --testPathPatterns="ItemActionPanel" 2>&1 | tail -5
  ```

  Expected: FAIL — `Cannot find module '@/components/ItemActionPanel'`

- [ ] **Step 3: Create `src/components/ItemActionPanel.tsx`**

  ```typescript
  // src/components/ItemActionPanel.tsx
  'use client'

  import { useState, useTransition } from 'react'
  import { logWear, type LogWearInput } from '@/app/actions/logWear'
  import { saveReview, type SaveReviewInput } from '@/app/actions/saveReview'
  import { archiveItem, type ArchiveItemInput } from '@/app/actions/archiveItem'
  import type { LPReview, WearRecord, FailedPurchase } from '@/types'

  type ActiveForm = 'idle' | 'wear' | 'review' | 'failed'

  type Props = {
    itemId: string
    existingReview: LPReview | null
  }

  const labelClass = 'block text-xs text-warm tracking-widest uppercase mb-1'
  const inputClass = 'w-full border border-espresso/20 bg-cream text-espresso text-sm px-3 py-2 focus:outline-none focus:border-espresso/60'
  const primaryBtn = 'bg-espresso text-cream text-xs tracking-widest uppercase px-5 py-2 hover:bg-warm transition-colors disabled:opacity-40'
  const cancelLink = 'text-xs text-warm hover:text-espresso transition-colors cursor-pointer'

  export default function ItemActionPanel({ itemId, existingReview }: Props) {
    const [active, setActive] = useState<ActiveForm>('idle')
    const [pending, startTransition] = useTransition()
    const [error, setError] = useState<string | null>(null)

    // Wear form state
    const today = new Date().toISOString().split('T')[0]
    const [wornOn, setWornOn] = useState(today)
    const [season, setSeason] = useState<WearRecord['season']>(null)
    const [occasion, setOccasion] = useState<WearRecord['occasion']>(null)

    // Review form state
    const [fit, setFit] = useState(existingReview?.fit ?? 5)
    const [comfort, setComfort] = useState(existingReview?.comfort ?? 5)
    const [quality, setQuality] = useState(existingReview?.quality ?? 5)
    const [versatility, setVersatility] = useState(existingReview?.versatility ?? 5)
    const [value, setValue] = useState(existingReview?.value ?? 5)
    const [notes, setNotes] = useState(existingReview?.notes ?? '')

    // Failed form state
    const [reason, setReason] = useState<FailedPurchase['reason']>('poor fit')
    const [failedNotes, setFailedNotes] = useState('')

    function cancel() { setActive('idle'); setError(null) }

    function submitWear() {
      setError(null)
      const input: LogWearInput = { itemId, wornOn, season, occasion }
      startTransition(async () => {
        try { await logWear(input); setActive('idle') }
        catch (e) { setError(e instanceof Error ? e.message : 'Failed to log wear') }
      })
    }

    function submitReview() {
      setError(null)
      const input: SaveReviewInput = { itemId, fit, comfort, quality, versatility, value, notes }
      startTransition(async () => {
        try { await saveReview(input); setActive('idle') }
        catch (e) { setError(e instanceof Error ? e.message : 'Failed to save review') }
      })
    }

    function submitArchive() {
      setError(null)
      const input: ArchiveItemInput = { itemId, reason, notes: failedNotes }
      startTransition(async () => {
        try { await archiveItem(input); setActive('idle') }
        catch (e) { setError(e instanceof Error ? e.message : 'Failed to archive item') }
      })
    }

    if (active === 'idle') {
      return (
        <div className="border-t border-espresso/10 pt-6">
          <p className="text-xs tracking-widest uppercase text-warm mb-4">Actions</p>
          <div className="flex flex-wrap gap-3">
            <button onClick={() => setActive('wear')} className={primaryBtn}>Log Wear</button>
            <button onClick={() => setActive('review')} className={primaryBtn}>
              {existingReview ? 'Update Review' : 'Write Review'}
            </button>
            <button
              onClick={() => setActive('failed')}
              className="border border-espresso/20 text-espresso text-xs tracking-widest uppercase px-5 py-2 hover:bg-espresso/5 transition-colors"
            >
              Mark as Failed
            </button>
          </div>
        </div>
      )
    }

    if (active === 'wear') {
      return (
        <div className="border-t border-espresso/10 pt-6">
          <p className="text-xs tracking-widest uppercase text-warm mb-4">Log Wear</p>
          <div className="space-y-4">
            <div>
              <label htmlFor="worn-on" className={labelClass}>Date</label>
              <input
                id="worn-on"
                type="date"
                value={wornOn}
                onChange={e => setWornOn(e.target.value)}
                className={inputClass}
              />
            </div>
            <div>
              <label htmlFor="season" className={labelClass}>Season</label>
              <select
                id="season"
                value={season ?? ''}
                onChange={e => setSeason((e.target.value as WearRecord['season']) || null)}
                className={inputClass}
              >
                <option value="">— optional —</option>
                {(['Spring', 'Summer', 'Autumn', 'Winter'] as const).map(s => (
                  <option key={s} value={s}>{s}</option>
                ))}
              </select>
            </div>
            <div>
              <label htmlFor="occasion" className={labelClass}>Occasion</label>
              <select
                id="occasion"
                value={occasion ?? ''}
                onChange={e => setOccasion((e.target.value as WearRecord['occasion']) || null)}
                className={inputClass}
              >
                <option value="">— optional —</option>
                {(['Casual', 'Smart Casual', 'Work', 'Formal', 'Sport'] as const).map(o => (
                  <option key={o} value={o}>{o}</option>
                ))}
              </select>
            </div>
            {error && <p className="text-red-600 text-xs">{error}</p>}
            <div className="flex items-center gap-4">
              <button onClick={submitWear} disabled={pending} className={primaryBtn}>
                {pending ? 'Saving…' : 'Log Wear'}
              </button>
              <span role="button" onClick={cancel} className={cancelLink}>Cancel</span>
            </div>
          </div>
        </div>
      )
    }

    if (active === 'review') {
      const scoreFields: { label: string; id: string; val: number; set: (n: number) => void }[] = [
        { label: 'Fit', id: 'score-fit', val: fit, set: setFit },
        { label: 'Comfort', id: 'score-comfort', val: comfort, set: setComfort },
        { label: 'Quality', id: 'score-quality', val: quality, set: setQuality },
        { label: 'Versatility', id: 'score-versatility', val: versatility, set: setVersatility },
        { label: 'Value', id: 'score-value', val: value, set: setValue },
      ]
      return (
        <div className="border-t border-espresso/10 pt-6">
          <p className="text-xs tracking-widest uppercase text-warm mb-4">
            {existingReview ? 'Update Review' : 'Write Review'}
          </p>
          <div className="space-y-4">
            <div className="grid grid-cols-2 gap-4 sm:grid-cols-3">
              {scoreFields.map(({ label, id, val, set }) => (
                <div key={label}>
                  <label htmlFor={id} className={labelClass}>{label}</label>
                  <input
                    id={id}
                    type="number"
                    min={1}
                    max={10}
                    value={val}
                    onChange={e => set(Math.min(10, Math.max(1, Number(e.target.value))))}
                    className={inputClass}
                  />
                </div>
              ))}
            </div>
            <div>
              <label htmlFor="review-notes" className={labelClass}>Notes</label>
              <textarea
                id="review-notes"
                rows={3}
                value={notes}
                onChange={e => setNotes(e.target.value)}
                placeholder="Optional notes…"
                className={`${inputClass} resize-none`}
              />
            </div>
            {error && <p className="text-red-600 text-xs">{error}</p>}
            <div className="flex items-center gap-4">
              <button onClick={submitReview} disabled={pending} className={primaryBtn}>
                {pending ? 'Saving…' : 'Save Review'}
              </button>
              <span role="button" onClick={cancel} className={cancelLink}>Cancel</span>
            </div>
          </div>
        </div>
      )
    }

    // active === 'failed'
    return (
      <div className="border-t border-espresso/10 pt-6">
        <p className="text-xs tracking-widest uppercase text-warm mb-4">Mark as Failed</p>
        <div className="space-y-4">
          <div>
            <label htmlFor="fail-reason" className={labelClass}>Reason</label>
            <select
              id="fail-reason"
              value={reason}
              onChange={e => setReason(e.target.value as FailedPurchase['reason'])}
              className={inputClass}
            >
              {(
                ['poor fit', 'poor quality', 'poor value', 'rarely worn',
                  "doesn't suit style", 'uncomfortable', 'impulse purchase', 'other'] as const
              ).map(r => (
                <option key={r} value={r}>{r.charAt(0).toUpperCase() + r.slice(1)}</option>
              ))}
            </select>
          </div>
          <div>
            <label htmlFor="fail-notes" className={labelClass}>Notes</label>
            <textarea
              id="fail-notes"
              rows={3}
              value={failedNotes}
              onChange={e => setFailedNotes(e.target.value)}
              placeholder="What went wrong?"
              className={`${inputClass} resize-none`}
            />
          </div>
          {error && <p className="text-red-600 text-xs">{error}</p>}
          <div className="flex items-center gap-4">
            <button
              onClick={submitArchive}
              disabled={pending}
              className="bg-warm text-cream text-xs tracking-widest uppercase px-5 py-2 hover:opacity-80 transition-opacity disabled:opacity-40"
            >
              {pending ? 'Archiving…' : 'Archive Item'}
            </button>
            <span role="button" onClick={cancel} className={cancelLink}>Cancel</span>
          </div>
        </div>
      </div>
    )
  }
  ```

- [ ] **Step 4: Run tests to confirm they pass**

  ```bash
  npx jest --testPathPatterns="ItemActionPanel" 2>&1 | tail -5
  ```

  Expected: 10 tests pass.

- [ ] **Step 5: Type-check**

  ```bash
  npx tsc --noEmit 2>&1
  ```

  Expected: no output.

- [ ] **Step 6: Run full test suite**

  ```bash
  npx jest 2>&1 | tail -5
  ```

  Expected: all new tests pass. The 2 pre-existing ItemCard failures (`shows a placeholder when there are no photos`) are unrelated — ignore them.

- [ ] **Step 7: Commit**

  ```bash
  git add src/components/ItemActionPanel.tsx __tests__/components/ItemActionPanel.test.tsx
  git commit -m "Add ItemActionPanel client component with wear, review, and archive forms"
  ```

---

### Task 3: Wire item detail page and ItemCard Archived badge

**Files:**
- Modify: `src/app/items/[id]/page.tsx`
- Modify: `src/components/ItemCard.tsx`

**Interfaces:**
- Consumes:
  - `ItemActionPanel` from `@/components/ItemActionPanel` (Props: `{ itemId: string, existingReview: LPReview | null }`)
  - `latestReview` already computed in the page from existing `reviews` array
  - `item.status: 'owned' | 'archived'` already on the `Item` type

- [ ] **Step 1: Read the current item detail page**

  Read `src/app/items/[id]/page.tsx` in full before editing. Confirm:
  - `const supabase = await createClient()` is on line ~11
  - `const reviews = (item.reviews as LPReview[]) ?? []` is present
  - `const latestReview = [...reviews].sort(...)[0]` is present

- [ ] **Step 2: Modify `src/app/items/[id]/page.tsx`**

  Make three targeted edits to the existing file:

  **Edit 1** — Add the auth check after the supabase client is created. Insert this line immediately after `const supabase = await createClient()`:

  ```typescript
  const { data: { user } } = await supabase.auth.getUser()
  ```

  **Edit 2** — Add the import for `ItemActionPanel` at the top of the file alongside the other imports:

  ```typescript
  import ItemActionPanel from '@/components/ItemActionPanel'
  ```

  **Edit 3** — Add `<ItemActionPanel>` at the bottom of the right column `<div>` (after the purchase price section, before the closing `</div>`). The right column div closes at the end of the JSX — place it there:

  ```tsx
  {user && (
    <ItemActionPanel
      itemId={item.id as string}
      existingReview={latestReview ?? null}
    />
  )}
  ```

  `latestReview` is already computed above as `const latestReview = [...reviews].sort(...)[0]`. The `?? null` handles the case where `reviews` is empty (latestReview would be `undefined`).

- [ ] **Step 3: Read the current ItemCard**

  Read `src/components/ItemCard.tsx` in full before editing. Confirm the image container div has `className="aspect-[3/4] bg-warm/10 relative overflow-hidden mb-3"` — the badge goes inside this div.

- [ ] **Step 4: Modify `src/components/ItemCard.tsx`**

  Inside the image container div (the `<div className="aspect-[3/4]...">`) add the Archived badge just below the LP score badge. The LP score badge is:

  ```tsx
  <div className="absolute top-3 right-3 bg-espresso text-cream text-xs px-2 py-1 tracking-wide">
    {item.lp_score ?? '–'}
  </div>
  ```

  Add this immediately after it:

  ```tsx
  {item.status === 'archived' && (
    <div className="absolute top-2 left-2 bg-warm/20 text-warm text-[10px] tracking-widest uppercase px-2 py-0.5">
      Archived
    </div>
  )}
  ```

- [ ] **Step 5: Type-check**

  ```bash
  npx tsc --noEmit 2>&1
  ```

  Expected: no output.

- [ ] **Step 6: Run full test suite**

  ```bash
  npx jest 2>&1 | tail -5
  ```

  Expected: all tests pass (new + existing). The 2 pre-existing ItemCard placeholder failures are unrelated — ignore them.

- [ ] **Step 7: Smoke test locally**

  Run `npm run dev`. Log in at `/login`. Navigate to any item detail page. Confirm:
  - Three action buttons appear at the bottom of the right column when logged in
  - Clicking "Log Wear" replaces the buttons with the wear form
  - Filling out the form and clicking "Log Wear" updates the wear count on the page without navigation
  - Clicking "Write Review" shows the review form; submitting updates the LP Score
  - Clicking "Mark as Failed" shows the archive form; submitting sets the status to archived
  - After archiving, navigate to the collection — the item shows an "Archived" badge

  Log out and confirm the action panel is completely hidden.

- [ ] **Step 8: Commit**

  ```bash
  git add src/app/items/\[id\]/page.tsx src/components/ItemCard.tsx
  git commit -m "Wire ItemActionPanel into item detail page and add Archived badge to ItemCard"
  ```

- [ ] **Step 9: Deploy**

  ```bash
  vercel --prod
  ```

  Confirm `https://lp-style-book.vercel.app` is live.

---

## Self-Review

**Spec coverage:**
- ✅ Wear logging: short form (date pre-filled, season, occasion) → `logWear` action
- ✅ LP Review form: five 1–10 inputs + notes, pre-fills from existing review → `saveReview` upsert
- ✅ Failed purchase logging: reason dropdown + notes → `archiveItem` action
- ✅ All forms inline on item detail page — no page navigation
- ✅ Auth-gated: actions redirect to /login, panel only rendered when `user !== null`
- ✅ Archived items stay visible in collection with "Archived" badge
- ✅ `revalidatePath` called after every mutation so page data refreshes without navigation

**Placeholder scan:** None found.

**Type consistency:**
- `LogWearInput`, `SaveReviewInput`, `ArchiveItemInput` — defined in Task 1, consumed in Task 2 (component imports them)
- `LPReview` — defined in `@/types`, used as `existingReview` prop type in Task 2, passed from Task 3 page
- `ItemActionPanel` Props — defined in Task 2, consumed in Task 3
