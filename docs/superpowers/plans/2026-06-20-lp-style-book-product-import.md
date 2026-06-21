# LP's Style Book — Plan 2: Product Import Workflow

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Let LP add items to the wardrobe by pasting a product URL — the system scrapes the product details and photo automatically, falling back to alternative sites and finally a manual form if scraping fails.

**Architecture:** A server action receives a URL, attempts to scrape product name/brand/price/description/photo from the page HTML, downloads the photo to Supabase Storage, and creates a draft item record. A form lets LP review and complete the draft before saving.

**Tech Stack:** Next.js 14 App Router, Supabase Storage, cheerio (HTML scraping), TypeScript strict

## Global Constraints

- TypeScript strict mode — no `any`
- Tailwind classes only, warm minimal palette (cream/espresso/tan/warm)
- Next.js 14 App Router patterns — server actions for mutations
- All scraped photos downloaded to Supabase Storage (`item-photos` bucket), never stored as external URLs
- Admin-only feature — all import routes under `/admin`
- npm only

---

## Plan 2 Scope (to be written out in full before execution)

### What gets built

1. **URL scraper** (`src/lib/scraper.ts`)
   - Fetch page HTML, extract: product name, brand, price, description, material, primary image URL
   - Fallback chain: try original URL → try alternative retailer sites for same product → return partial data if all fail
   - Download primary image to Supabase Storage, return `storage_path`

2. **Import form** (`src/app/admin/import/page.tsx`)
   - Text input: paste a product URL
   - On submit: call scrape server action → populate a draft form with scraped data
   - Draft form fields: name, brand (searchable select from existing brands), subcategory, description, material, purchase_price, source_url, photo preview
   - Presence flags: `in_collection`, `in_wishlist`, `is_recommendation` (checkboxes)
   - Save button: creates the item record, links the photo, redirects to item detail page

3. **Admin nav link** — add "Import Item" link to the admin page

4. **Brand creation inline** — if the scraped brand doesn't exist in the DB, offer to create it on the fly from the import form

### Alternative site fallback logic

When scraping fails or returns incomplete data:
- If original URL is a brand's own site, try searching major retailers (Mr Porter, END, SSENSE) for the same product name + brand
- If no alternative found, prefill what was scraped and mark remaining fields as manual-entry required

### Out of scope for Plan 2

- Bulk import
- Barcode scanning
- Automatic wear tracking
- Price history

---

## Status

**Not started.** Paused after Plan 1 (foundation) shipped on 2026-06-20.

To resume: flesh out the full task breakdown (Tasks 1–N with complete code), then run subagent-driven-development.

---

# LP's Style Book — Plan 3: Tracking, Reviews, Wishlist, Analytics & Recommendations

**Goal:** Turn the wardrobe into an active intelligence system — log wears, write reviews, manage the wishlist pipeline, surface analytics, and publish LP's editorial recommendations.

**Architecture:** Admin forms for data entry (reviews, wear logs, wishlist management), server-rendered analytics pages pulling aggregated data from Supabase, and a public-facing Recommendations page with LP's written takes.

**Tech Stack:** Next.js 14 App Router, Supabase, TypeScript strict

## Global Constraints

- TypeScript strict mode — no `any`
- Tailwind classes only, warm minimal palette
- Admin forms under `/admin`, public pages at `/recommendations`
- npm only

---

## Plan 3 Scope (to be written out in full before execution)

### What gets built

1. **Wear logging** (`/admin/items/[id]/log-wear`)
   - Quick form: date worn, season (Spring/Summer/Autumn/Winter), occasion (Casual/Smart Casual/Work/Formal/Sport)
   - Updates wear count displayed on item cards and detail pages
   - Cost-per-wear calculation: `purchase_price / wear_count`

2. **LP Review form** (`/admin/items/[id]/review`)
   - Sliders or number inputs for: Fit, Comfort, Quality, Versatility, Value (each 1–10)
   - Notes text area
   - Saves to `lp_reviews` table, immediately updates LP Score on item detail page

3. **Failed purchase logging** (`/admin/items/[id]/failed`)
   - Reason dropdown: poor fit, poor quality, poor value, rarely worn, doesn't suit style, uncomfortable, impulse purchase, other
   - Notes field
   - Sets item status to `archived`

4. **Wishlist management** (`/admin/wishlist`)
   - List all `wishlist_items` with their pipeline status
   - Status pipeline: Wishlist → Researching → Considering → Purchased → Archived
   - Move items through pipeline with a button
   - Add notes and alternatives per wishlist item
   - Interest score (1–10) per item

5. **Analytics dashboard** (`/admin/analytics`)
   - Total wardrobe value
   - Average LP Score across all reviewed items
   - Most worn items (top 10)
   - Least worn items (candidates for removal)
   - Cost per wear by item
   - Spend by brand
   - Spend by category
   - Failed purchase rate and reasons breakdown

6. **Item pairings** (`/admin/items/[id]/pairings`)
   - Search and link items that pair well together
   - Note field per pairing
   - Displayed on item detail page

7. **Recommendations page** (`/recommendations`)
   - Public-facing page listing items where `is_recommendation = true`
   - LP's written take per item (from `recommendation_entries` table)
   - Admin form to write/edit the take: `/admin/recommendations`

8. **Hero image management** (`/admin/hero`)
   - Upload images to Supabase Storage (`hero_images` table)
   - Set display order
   - Delete images

### Out of scope for Plan 3

- Social sharing per item
- Price drop alerts
- Outfit builder

---

## Status

**Not started.** Depends on Plan 2 (product import) being complete first, so items can be added efficiently before tracking begins.

To resume: flesh out the full task breakdown (Tasks 1–N with complete code), then run subagent-driven-development.
