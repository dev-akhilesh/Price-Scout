# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

PriceScout — a Next.js app that tracks e-commerce product prices. Users paste a product URL, Firecrawl scrapes the name/price/currency/image, and the data is stored in Supabase. A daily cron job (triggered by Supabase `pg_cron`) re-scrapes all tracked products, records price history, and emails price-drop alerts via Resend.

## Commands

```bash
npm run dev      # start dev server (localhost:3000)
npm run build    # production build
npm run start    # run production build
npm run lint     # eslint
```

There is no test suite configured in this repo.

Generate a `CRON_SECRET` for `.env.local`:

```bash
node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"
```

## Environment

Required vars in `.env.local` (see README for full setup steps): `NEXT_PUBLIC_SUPABASE_URL`, `NEXT_PUBLIC_SUPABASE_ANON_KEY`, `SUPABASE_SERVICE_ROLE_KEY`, `FIRECRAWL_API_KEY`, `RESEND_API_KEY`, `RESEND_FROM_EMAIL`, `CRON_SECRET`, `NEXT_PUBLIC_APP_URL`.

## Architecture

**Data flow for adding a product**: `AddProductForm` (client) → `addProduct` server action in [app/actions.js](app/actions.js) → `scrapeProduct` in [lib/firecrawl.js](lib/firecrawl.js) (Firecrawl `extract` with a fixed JSON schema for name/price/currency/image) → upsert into Supabase `products` (unique on `user_id, url`) → insert into `price_history` only if the price actually changed. All product reads/writes go through server actions in `app/actions.js`, not client-side Supabase calls.

**Automated price checks**: Supabase `pg_cron` calls `POST /api/cron/check-prices` ([app/api/cron/check-prices/route.js](app/api/cron/check-prices/route.js)) daily via `net.http_post`, authenticated with a `Bearer CRON_SECRET` header checked against `process.env.CRON_SECRET`. This route uses the Supabase **service role** client (bypasses RLS) to read every user's products, re-scrape each via Firecrawl, update `current_price`, append to `price_history` on change, and call `sendPriceDropAlert` ([lib/email.js](lib/email.js)) via Resend when the new price is lower. The cron endpoint and the schema/schedule live in Supabase SQL migrations described in the README (not present as files in this repo — apply them by hand in the Supabase SQL editor).

**Auth**: Google OAuth via Supabase Auth. `AuthModal`/`AuthButton` kick off sign-in; `app/auth/callback/route.js` exchanges the OAuth `code` for a session. Session refresh on every request happens in `proxy.js` (Next's proxy/middleware entry point) delegating to `utils/supabase/middleware.js`'s `updateSession`. There are three separate Supabase client constructors, each for a distinct context — don't cross them:
- `utils/supabase/client.js` — browser client (anon key)
- `utils/supabase/server.js` — server-component/action client, cookie-bound (anon key)
- inline `createClient` with `SUPABASE_SERVICE_ROLE_KEY` in the cron route — the only place RLS is bypassed

**RLS**: `products` and `price_history` are row-level-secured by `user_id` (see README migration SQL). Any new query path for these tables from a user-facing context must go through the cookie-bound server/browser clients so RLS applies; only the cron route legitimately uses the service-role client.

**UI**: shadcn/ui (`new-york` style, components in [components/ui/](components/ui/)) + Tailwind v4 + Recharts (`PriceChart.js` renders price history) + lucide-react icons. Path alias `@/*` maps to repo root (see `jsconfig.json` / `components.json`).

**Server actions vs API routes**: mutations reachable from the UI (`addProduct`, `deleteProduct`, `getProducts`, `getPriceHistory`, `signOut`) are Next.js server actions in `app/actions.js`, called directly from components — there is no corresponding REST API for these. The only API route is the cron price-check endpoint, which is a genuine HTTP endpoint because it's invoked externally by Supabase.
