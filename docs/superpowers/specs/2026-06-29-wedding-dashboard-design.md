# Wedding Dashboard — Design Spec
**Date:** 2026-06-29  
**Project:** `wedding-dashboard`  
**Location:** `projects/wedding-dashboard/`

---

## Overview

A local Next.js web app that serves as Louis-Philippe and Anna's wedding planning hub. Runs with `npm run dev` on their laptop — no hosting, no login. Pre-loaded with all their data. Checkboxes and edits persist in `localStorage` so progress survives page refreshes.

**Wedding:** October 17, 2026 · Ottawa, ON · #wickhamwedding2026

---

## Tech Stack

- Next.js + TypeScript + Tailwind CSS (project standard)
- `localStorage` for persistence — no database, no backend
- Package manager: npm

---

## Sections (top to bottom on one scrollable page)

### 1. Hero

- Names: "Louis-Philippe & Anna"
- Date: October 17, 2026
- Live countdown: days / hours / minutes (updates every second via `useEffect` + `setInterval`)
- Hashtag: #wickhamwedding2026
- Visual: cream background, dark warm text, thin decorative rule

### 2. To-Do Checklist

All items from the notebook pre-loaded. Each item has a checkbox. Checking an item:
- Strikes through the text and greys it out
- Saves its state to `localStorage` keyed by item ID

Items starred in the notebook get a small "Priority" badge.

State shape in localStorage:
```json
{ "todo": { "item-id": true } }
```

**Categories and items:**

**Ceremony**
- [ ] Book officiant (Steve?) ⭐
- [ ] Write vows
- [ ] Book musician for ceremony
- [ ] Decide: record player for cocktail hour?
- [ ] Finalize order of service / readings
- [ ] Ring bearer & flower girl — confirm + outfits
- [ ] How to explain kids-only ceremony to guests
- [ ] Welcome sign

**Reception**
- [ ] Seating chart ⭐
- [ ] Book menu tasting ⭐
- [ ] Figure out meals / contact Kim ⭐
- [ ] Plan reception entrance (bride/groom + parties)
- [ ] Where to put gifts & cards table
- [ ] Photo booth — DIY or rent? ⭐
- [ ] Champagne plan
- [ ] Bathroom bins
- [ ] Coat check
- [ ] Bud vases — who fills & where
- [ ] Dinner kissing game
- [ ] Wedding favours?
- [ ] Update Zola website with food info

**People & Roles**
- [ ] Confirm MCs
- [ ] Confirm speeches (parents, best man, maid of honour)
- [ ] 2 mics + schedule of dinner speeches
- [ ] Assign bridesmaid roles
- [ ] Assign groomsmen roles
- [ ] Speech requirements — send to speakers
- [ ] Thank you speech (LP + Anna)
- [ ] Thank you gifts — boys (Casio A158WA-1 × 4)
- [ ] Thank you gifts — girls (× 4)
- [ ] Decide who picks outfits for wedding party
- [ ] Gifts for wedding party

**Logistics**
- [ ] Build day-of timeline (getting ready + photography) ⭐
- [ ] Send timeline to all vendors ⭐
- [ ] Rehearsal dinner — plan & book
- [ ] Send invitations ⭐
- [ ] Bridal party transportation
- [ ] Set-up + tear-down team — confirm
- [ ] When does venue open for vendors?
- [ ] DJ songs list
- [ ] First dance — decide song + book classes?
- [ ] Parent dances — decide songs
- [ ] Where do guests sign (guestbook)?
- [ ] Hashtag + photo sharing app for guests
- [ ] Sunday brunch (day after) — book?

**Attire & Beauty**
- [ ] Confirm hair & makeup
- [ ] Hair & makeup trial
- [ ] Dress alterations
- [ ] Heels
- [ ] Wedding jewellery
- [ ] Eyelashes & nails
- [ ] Bridesmaid colour swatches
- [ ] Photography shot list + locations ⭐

### 3. Budget Tracker

Read-only table. Pre-loaded from Excel data. Columns:

| Service | Status | Paid So Far | Still Owing | Total Cost | Notes |
|---|---|---|---|---|---|

Status shown as a coloured dot:
- 🟢 Green = Paid
- 🟡 Yellow = In Progress  
- ⚪ Grey = TBD

**Pre-loaded rows:**

| Service | Status | Paid | Owing | Total | Notes |
|---|---|---|---|---|---|
| Venue + Food + Drinks | In Progress | $2,500 | TBD | TBD | Tax + gratuity included |
| Ceremony Setup | In Progress | $0 | TBD | TBD | |
| Photographer | In Progress | $500 | TBD | ~$3,165 | Southern Ontario Lady |
| Florals | Paid | $994 | $0 | $994 | Tax + gratuity included |
| Decor | In Progress | $0 | TBD | TBD | |
| Music | In Progress | $450 | TBD | ~$1,920 | Tax included |
| Bride's attire | Paid | $2,017 | $0 | $2,017 | |
| Bridesmaid colour swatches | In Progress | $0 | TBD | TBD | |
| Groom's attire | Paid | $835 | $0 | $835 | Suit Supply |
| Fittings / alterations | In Progress | $0 | TBD | ~$200 | Fayez |
| Stationery | Paid | $350 | $0 | $350 | Naomi |
| Stamps | Paid | $56 | $0 | $56 | |
| Rings | Paid | $3,265.70 | $0 | $3,265.70 | Goldform |
| Transportation | In Progress | $0 | TBD | TBD | |
| Hair / Makeup | In Progress | $0 | TBD | ~$327 | Tax + gratuity included |
| Officiant | In Progress | $0 | TBD | $200 | Tax + gratuity included |
| Wedding night hotel | In Progress | $0 | TBD | TBD | |
| Day-after brunch | In Progress | $0 | TBD | ~$100 | |
| Thank you gifts — boys | In Progress | $0 | TBD | ~$181 | Casio A158WA-1 × 4 |
| Thank you gifts — girls | In Progress | $0 | TBD | ~$181 | |
| Honeymoon | TBD | — | — | — | |

Totals row at the bottom: sum of Paid So Far and sum of Still Owing (skips TBD cells).

### 4. Vendor Contacts

Card grid — one card per vendor. Each card shows:
- Vendor name (pre-filled)
- Category label (e.g. "Photography")
- Phone field (editable, saves to localStorage)
- Email field (editable, saves to localStorage)
- Notes field (editable, saves to localStorage)

Pre-filled vendor cards: Venue, Photographer, Florals, Music/DJ, Officiant, Hair & Makeup, Alterations (Fayez), Rings (Goldform), Stationery (Naomi), Transportation.

Edits save on blur.

State shape in localStorage:
```json
{ "vendors": { "venue": { "phone": "...", "email": "...", "notes": "..." } } }
```

---

## Visual Design

- **Palette:** Cream (`#FAF8F4`) background, dark warm brown (`#2C1A0E`) text, dusty rose (`#C9847A`) accents for badges and dots, sage green (`#7A9E7E`) for "Paid" status
- **Typography:** serif for headings (Playfair Display via `next/font/google` — self-hosted at build time, works offline), sans-serif for body
- **Layout:** max-width container, generous padding, section dividers
- **Checkboxes:** custom-styled, animated checkmark on completion

---

## Persistence

All state lives in `localStorage` under the key `"wickham-wedding"`. On load, the app reads this and hydrates the UI. No server, no sync — works fully offline.

---

## File Structure

```
projects/wedding-dashboard/
├── src/
│   ├── app/
│   │   ├── page.tsx          ← main page, composes all sections
│   │   ├── layout.tsx
│   │   └── globals.css
│   ├── components/
│   │   ├── Hero.tsx
│   │   ├── TodoSection.tsx
│   │   ├── BudgetTable.tsx
│   │   └── VendorContacts.tsx
│   └── data/
│       ├── todos.ts          ← all todo items and categories
│       └── budget.ts         ← all budget rows
├── package.json
└── README.md
```

---

## Out of Scope

- No authentication
- No hosting / deployment (local only)
- No multi-device sync
- No editing of budget numbers in the UI (read-only from the data file)
