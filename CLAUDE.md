# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Dev server

Start with the launch config at `.claude/launch.json` — it spins up a Node HTTP server on port 3456 serving `rescuebite-app.html` as `/`:

```bash
/Users/shahbaz/local/bin/node -e "$(cat .claude/launch.json | python3 -c "import sys,json; print(json.load(sys.stdin)['configurations'][0]['runtimeArgs'][1])")"
```

Or just open `http://localhost:3456` if the server is already running from the Claude Code launch config.

**After every edit to `rescuebite-app.html`, copy it to `index.html`** (Vercel serves `index.html`):
```bash
cp rescuebite-app.html index.html
```

Then commit and push — Vercel auto-deploys on push to `main`.

## Architecture

Single-file vanilla app — all HTML, CSS, and JS lives in **`rescuebite-app.html`**. `index.html` is always a verbatim copy for Vercel. Never edit `index.html` directly.

### State & router

```js
const S = { view, cart, listingId, qty, bizMode, user, orders, ... }
function go(view, data)   // router — merges data into S, calls render()
function render()         // calls renderHeader() + renderContent() + renderTabBar()
```

All views are plain functions (`vHome`, `vDetail`, `vCart`, `vCheckout`, ...) that return HTML strings. `renderContent()` picks the right one via a `map` object. No framework, no virtual DOM.

### Data layer

```js
let DB = { listings: [], orders: [] }          // in-memory cache, populated on load
const supa = supabase.createClient(SUPA_URL, SUPA_KEY)  // CDN-loaded Supabase JS v2
```

`initDB()` runs on `DOMContentLoaded` — parallel-fetches listings, orders, consumer profile, and partner profile from Supabase, then merges into `S` and `localStorage`.

All writes go to Supabase first, then update `DB` on success (write-through). `getL()` and `getO()` are the only read accessors — they filter `DB` by city / consumer ID and run through `mapL()` / `mapO()` (DB row → app object).

### Two modes

`S.bizMode` (boolean) separates partner and consumer flows entirely:
- `CONSUMER_VIEWS` array — `go()` redirects any of these to `bizdash` when `S.bizMode` is true
- Partner profile stored in `localStorage.rb_biz`; consumer profile in `localStorage.rb_user`
- `getBizProfile()` reads `rb_biz`; `CID` (device UUID in `rb_cid`) identifies consumers

### Cart & checkout

```
Detail (qty ±) → addToCart() → Cart view → Checkout (user details) → OTP verify → Success
```

`S.cart = [{ listingId, qty }]` persisted to `rb_cart`. `confirmOrder()` builds `S.pendingOrders[]` (one per cart item), generates a 6-digit OTP stored in `S.otpCode`, and navigates to `otpverify`. `verifyAndCompleteOrder()` batch-inserts all orders and clears the cart.

### Key constants

- `COUNTRY_CODES` — 24-country array for phone dropdowns
- `BIZ_CAT_MAP` — business type → allowed listing categories (used in `vNewListing`)
- `fmtTime(t)` / `parse12to24(str)` — convert between `"HH:MM"` and `"7:00 PM"` for pickup windows

### Supabase tables

| Table | Purpose |
|-------|---------|
| `listings` | Partner bag listings (`rem` = remaining stock) |
| `orders` | Consumer reservations |
| `consumers` | Consumer profiles (keyed by `CID`) |
| `partners` | Partner/business profiles |
| `otp_tokens` | Short-lived OTP codes (email → code, expires_at) |

RLS is enabled on all tables with anon full-access policies (MVP).

### localStorage keys

| Key | Value |
|-----|-------|
| `rb_cid` | Device UUID (consumer identity) |
| `rb_user` | `{ name, email, phone, countryCode }` |
| `rb_biz` | `{ id, name, type, area, city, contact, phone }` |
| `rb_cart` | `[{ listingId, qty }]` |
| `rb_city` | Selected city string |
| `rb_signed_out` | `"1"` when consumer has explicitly signed out (blocks DB re-sync) |
