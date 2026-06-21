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
