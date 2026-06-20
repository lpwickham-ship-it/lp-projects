# LP's Style Book — Design Spec

**Date:** 2026-06-20
**Status:** Approved

---

## Vision

A personal wardrobe intelligence platform for tracking owned clothing, improving purchase decisions, learning style preferences, and increasing wardrobe versatility. Not an e-commerce store. Not a recommendation engine. A tool for dressing better and buying smarter.

---

## Tech Stack

| Concern | Tool |
|---|---|
| Framework | Next.js (App Router) + TypeScript |
| Styling | Tailwind CSS |
| Database | Supabase (PostgreSQL) |
| Auth | Supabase Auth (email login) |
| File storage | Supabase Storage (item photos) |
| Deployment | Vercel |
| Package manager | npm |

---

## Visual Design

**Palette:** Warm Minimal — off-white background (`#faf7f2`), espresso type (`#3d3025`), tan/gold accents (`#b8976a`), warm brown secondary (`#8b6f5e`).

**Typography:** Serif headings (Georgia or equivalent), clean sans-serif body text. Uppercase tracking on labels and nav items.

**Illustrations:** Labelled figure on the Recommendations page — a stylized line-art figure with arrows pointing to each clothing item, clickable to navigate to that item's page. Individual item photography sits alongside the illustrations throughout the site.

**Photography:** LP uploads real photos of items. Inspirational imagery cycles on the Home page hero.

---

## Access & Auth

- **LP (admin):** Logs in with email via Supabase Auth. Full read/write access to all content and the admin area.
- **Friends (viewers):** Access via a single shareable link containing a secret token. Anyone with the link can browse the full site but cannot edit anything.
- No friend accounts required. The shareable link can be rotated by LP at any time.

---

## Pages

### Home (Dashboard)
The landing page for everyone. Shows a cycling hero image, LP's name and tagline, and a stats dashboard:
- Total collection value
- Number of items owned
- Average LP Score across the collection
- Most worn and least worn items
- Recently added items
- Recently reviewed items
- Wardrobe insights (e.g. cost-per-wear leaders)

### Recommendations
LP's editorial voice. A page where LP publishes takes on clothing items — owned or not. Layout features the labelled figure illustration in a sidebar with clickable labels that link to the relevant item page. Each recommendation entry includes:
- Item name and brand
- LP's written take
- Whether LP owns it
- LP's rating (if owned)
- Link to the item page or external source

### My Collection
The full wardrobe browser. Items are organised by category (Clothing, Accessories, Shoes) with subcategories. Filter sidebar allows filtering by brand, price range, category, and LP Score. Each item appears as a card with a photo, item name, brand, and LP Score. Clicking a card goes to the Item page.

### Item Page
Full detail view for a single item:
- Item name and brand
- Photos (multiple, uploaded by LP)
- General description and material
- LP Score (breakdown: Fit, Comfort, Quality, Versatility, Value → composite score out of 10)
- Wear history (count, last worn, season, occasion, cost-per-wear)
- Notes
- Products to pair with — LP manually selects other items from their collection and adds a short note explaining why they work together
- Back button returns to previous page

### Brand Pages
One page per brand. Shows:
- Brand name and country of origin
- LP's notes on the brand
- All items LP owns from this brand
- Average LP Score across brand items
- Total wear count across brand items

### Admin Area (LP only)
Accessible only when logged in. Includes:
- Add items via URL import (see Product Import Workflow below)
- Edit / archive items
- Upload additional photos for items
- Log a wear (date, season, occasion)
- Write or edit a recommendation
- Write or edit a review (LP Score)
- Mark a purchase as failed and record reason
- Manage the Wishlist Pipeline
- Upload and manage Home page hero images (the cycling inspirational images)
- Rotate the shareable friend link

---

## Product Import Workflow

The primary way LP logs items is by importing from a product URL. This avoids manual data entry for fields that already exist on the brand's website.

**Step 1 — Paste a URL or search by name**
LP either pastes a direct product URL (e.g. from Drake's, Buck Mason, Mr Porter) or types the product name into a search field in the admin area.

**Step 2 — Scrape the source**
The app fetches the page and extracts: product name, brand, description, materials, price, and images. Images are downloaded and saved to Supabase Storage so they are permanently owned by LP (not linked to an external CDN that could break).

**Step 3 — Auto-fallback to alternative sites**
If the original URL is blocked or returns insufficient data, the app automatically searches for the same product across alternative retail sources (e.g. Mr Porter, SSENSE, END Clothing, the brand's own site). It presents the best matches found and lets LP pick which source to import from. This happens without LP needing to do anything — the fallback is automatic.

**Step 4 — Manual entry (last resort)**
Only if all sources fail does the app fall back to a manual entry form, pre-filled with whatever partial data was found.

**Step 5 — Review and categorise the draft**
LP reviews the pre-filled draft and answers categorisation questions before saving:
- **Do you own this item?** If yes → added to My Collection. If no → not added to collection.
- **Add to Wishlist Pipeline?** If yes → added as a Wishlist item with a starting status of `Wishlist`.
- **Publish as a Recommendation?** If yes → LP writes their take and it appears on the Recommendations page.
- An item can be checked into more than one section simultaneously (e.g. owned and also a recommendation).

LP can also manually add or remove photos, edit any imported field, and fill in LP-specific fields (LP Score, wear data, notes) before or after saving.

---

## Data Model

### Item
- Name, brand (FK), category, subcategory
- Description, material
- Purchase price, purchase date, purchase location
- Source URL (where the item was imported from)
- Status: `owned` | `archived`
- Presence flags:
  - `in_collection` (boolean) — appears in My Collection
  - `in_wishlist` (boolean) — appears in Wishlist Pipeline
  - `is_recommendation` (boolean) — appears on Recommendations page
- Multiple photos (Supabase Storage — downloaded from source, not linked)
- LP Score (computed from reviews)
- Wear count (computed from wear records)

### Brand
- Name, country
- LP's notes

### Category / Subcategory
- Clothing → tops, bottoms, outerwear, knitwear, etc.
- Accessories → watches, belts, bags, scarves, eyewear, etc.
- Shoes → trainers, boots, loafers, dress shoes, etc.

### LP Review
- Item (FK)
- Fit, Comfort, Quality, Versatility, Value (each 1–10)
- LP Score (average of five dimensions, out of 10)
- Review notes
- Review date

### Wear Record
- Item (FK)
- Date worn
- Season (Spring / Summer / Autumn / Winter)
- Occasion (Casual / Smart Casual / Work / Formal / Sport)

### Wishlist Item
- Name, brand, external URL
- Status: `Wishlist` | `Researching` | `Considering` | `Purchased` | `Archived`
- Notes, alternatives considered
- Price, interest score (1–10)
- Can be converted to a Collection Item on purchase

### Failed Purchase Record
- Item (FK)
- Reason: poor fit / poor quality / poor value / rarely worn / doesn't suit style / uncomfortable / impulse purchase / other
- Notes

### Recommendation Entry
- Item (FK, optional — can recommend items LP doesn't own)
- Written take
- Date published

### Shareable Link Token
- Token string (secret)
- Created date
- Active: boolean

---

## MVP Scope

Prioritised build order per the project brief:

1. **Database schema** — all tables, relationships, and seed data
2. **Product import workflow** — URL scraping, auto-fallback to alternative sites, photo download to Supabase Storage, draft review form with presence flag checkboxes
3. **Collection workflows** — edit, archive items; browse and filter by category/brand/price
4. **Wear tracking and reviews** — log wears, write LP Reviews, compute LP Score
5. **Wishlist Pipeline** — manage statuses, convert to collection item, failed purchase tracking
6. **Dashboard analytics** — stats, insights, most/least worn
7. **Brand pages** — per-brand profile with aggregated stats
8. **Recommendations page** — editorial takes with labelled figure
9. **Visual polish** — animations, transitions, final typography pass

---

## Seed Data

Before building advanced features, the database is populated with realistic demo data:
- 15–20 brands (mix of price tiers, countries)
- 30–50 owned items (mix of categories, successful and failed purchases)
- Realistic LP Reviews and wear histories per item
- 15–20 wishlist items at various pipeline stages

---

## Future Features (Not MVP)

- Outfit generation
- Wardrobe gap analysis
- Purchase approval assistant
- Style recommendations (AI-driven)
- Travel packing assistant
- AI-driven wardrobe insights
