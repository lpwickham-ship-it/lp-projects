# LP's Style Book — Plan 3A: Item Tracking Design

**Date:** 2026-06-21
**Status:** Approved

---

## Goal

Add wear logging, LP reviews, and failed purchase tracking directly to the item detail page, visible only when logged in. This turns the wardrobe into an active record — every wear counts, every review is scored, and failed purchases are honestly archived.

---

## What Gets Built

Three server actions and one client component. The item detail page gains an action panel at the bottom of the right column, visible only to authenticated users. The panel toggles between three inline forms without any page navigation.

### New Files

| File | Role |
|------|-------|
| `src/components/ItemActionPanel.tsx` | Client component — manages idle/wear-form/review-form/failed-form state |
| `src/app/actions/logWear.ts` | Server action — inserts `wear_records` row, revalidates item page |
| `src/app/actions/saveReview.ts` | Server action — upserts `lp_reviews` row, revalidates item page |
| `src/app/actions/archiveItem.ts` | Server action — inserts `failed_purchases` row, sets `item.status = 'archived'`, revalidates |

### Modified Files

| File | Change |
|------|--------|
| `src/app/items/[id]/page.tsx` | Check auth, pass `isAdmin` + current `reviews` + `item.id` to `<ItemActionPanel>` |
| `src/components/ItemCard.tsx` | Show "Archived" badge when `item.status === 'archived'` |

---

## Item Detail Page Changes

The server component at `src/app/items/[id]/page.tsx` already fetches the full item with reviews and wear records. It will additionally:

1. Call `supabase.auth.getUser()` to determine if LP is logged in
2. Pass `isAdmin: boolean`, `itemId: string`, and `existingReview: LPReview | null` as props to `<ItemActionPanel>`

`<ItemActionPanel>` is placed at the bottom of the right column, after the purchase price section, inside its own `border-t border-espresso/10 pt-6` section.

---

## ItemActionPanel Behaviour

The panel has four states, controlled by local React state:

### Idle state
Three buttons displayed in a row:
- `Log Wear`
- `Write Review`
- `Mark as Failed`

### Wear form state
Clicking "Log Wear" replaces the button row with:
- **Date** — `<input type="date">` pre-filled with today's date (YYYY-MM-DD)
- **Season** — `<select>`: Spring / Summer / Autumn / Winter
- **Occasion** — `<select>`: Casual / Smart Casual / Work / Formal / Sport
- **"Log Wear"** submit button + **"Cancel"** link (returns to idle, no save)

On submit: calls `logWear({ itemId, wornOn, season, occasion })` → revalidates → returns to idle.

### Review form state
Clicking "Write Review" replaces the button row with:
- Five number inputs (1–10): **Fit**, **Comfort**, **Quality**, **Versatility**, **Value**
- Pre-filled with existing review scores if a review already exists for this item
- **Notes** — `<textarea>` (optional), pre-filled if existing review has notes
- **"Save Review"** submit button + **"Cancel"** link

On submit: calls `saveReview({ itemId, fit, comfort, quality, versatility, value, notes })` → revalidates → returns to idle.

One review per item — if a review already exists, `saveReview` updates it (upsert by `item_id`).

### Failed form state
Clicking "Mark as Failed" replaces the button row with:
- **Reason** — `<select>`: Poor Fit / Poor Quality / Poor Value / Rarely Worn / Doesn't Suit Style / Uncomfortable / Impulse Purchase / Other
- **Notes** — `<textarea>` (optional)
- **"Archive Item"** submit button + **"Cancel"** link

On submit: calls `archiveItem({ itemId, reason, notes })` → revalidates → returns to idle. The item's `status` becomes `'archived'` in the database.

---

## Server Actions

### `logWear`

```
Input:  { itemId: string, wornOn: string, season: WearRecord['season'], occasion: WearRecord['occasion'] }
Effect: INSERT into wear_records; revalidatePath('/items/[itemId]')
Auth:   redirects to /login if no session
```

### `saveReview`

```
Input:  { itemId: string, fit: number, comfort: number, quality: number, versatility: number, value: number, notes: string }
Effect: UPSERT lp_reviews (match on item_id — one review per item); revalidatePath('/items/[itemId]')
Auth:   redirects to /login if no session
```

Upsert strategy: query for existing review by `item_id`. If found, UPDATE. If not, INSERT.

### `archiveItem`

```
Input:  { itemId: string, reason: FailedPurchase['reason'], notes: string }
Effect: INSERT into failed_purchases; UPDATE items SET status = 'archived' WHERE id = itemId; revalidatePath('/items/[itemId]')
Auth:   redirects to /login if no session
```

---

## Archived Items in the Collection

`src/components/ItemCard.tsx` receives the full item including `status`. When `status === 'archived'`, a small badge is displayed in the top-left corner of the card image area:

- Text: `Archived`
- Style: muted warm tone — `bg-warm/20 text-warm text-[10px] tracking-widest uppercase px-2 py-0.5`
- Position: `absolute top-2 left-2`

Archived items are not filtered out of the collection — they remain visible and clickable.

---

## Auth Pattern

All three server actions follow the same auth check used throughout the codebase:

```typescript
const supabase = await createClient()
const { data: { user } } = await supabase.auth.getUser()
if (!user) redirect('/login')
```

The `ItemActionPanel` itself is only rendered on the item detail page when `isAdmin === true`, so the buttons never appear to logged-out visitors.

---

## Data Types (already in `src/types/index.ts`)

All types used by this feature are already defined — no changes to `types/index.ts` needed:

- `WearRecord` — `worn_on`, `season`, `occasion`
- `LPReview` — `fit`, `comfort`, `quality`, `versatility`, `value`, `notes`
- `FailedPurchase` — `reason`, `notes`

---

## Out of Scope

- Bulk wear logging (logging multiple past wears at once)
- Deleting or editing a logged wear
- Deleting a review entirely (editing is supported via upsert)
- Filtering archived items out of the collection (future Plan 3B)
- Any analytics on wear data (future Plan 3B)
