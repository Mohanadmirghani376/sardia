# Sardia — Jinan Sardia International Supply Chain

B2B website connecting commercial buyers with vetted manufacturers in China.
Four sourcing divisions: **heavy trucks**, **construction machinery**,
**agricultural machinery** and **medical supplies**.

Built with Next.js 16 (App Router), React 19 and Tailwind CSS 4. Bilingual
English / Arabic with full RTL support.

---

## Quick start

```bash
npm install
npm run dev          # http://localhost:3000
```

| Script | What it does |
| --- | --- |
| `npm run dev` | Dev server |
| `npm run build` | Production build |
| `npm start` | Serve the production build |
| `npm run typecheck` | `tsc --noEmit` |
| `npm run check:i18n` | Lints Arabic strings for corruption (see below) |
| `npm run check:unused-i18n` | Lists dictionary keys nothing reads; exits 1 only on the verified set |
| `npm run check:contrast` | Checks every declared colour pairing against WCAG AA |
| `npm run check:links` | Boots the built app and checks every route returns 200 |
| `npm run check:dom` | Renders every route and audits real computed colours (needs a dev server) |
| `npm run images:manifest` | Regenerates `public/products/manifest.json` |
| `npm run verify` | typecheck → i18n → unused keys → contrast → build |

---

## Structure

```
src/
  proxy.ts                  Locale detection + redirect (/ → /en or /ar)
  app/
    actions.ts              submitEnquiry() server action
    globals.css             Design tokens, component classes
    robots.ts, sitemap.ts   SEO
    icon.svg                Favicon
    [locale]/
      layout.tsx            Root layout: lang, dir, header, footer
      page.tsx              Home
      about/ products/ services/ rfq/ contact/
      products/[slug]/      Per-division catalogue page
      opengraph-image.tsx   1200×630 social card
  components/               UI (Icon, SiteHeader, SiteFooter, forms, …)
  data/
    catalog.ts              4 divisions, bilingual product lines + specs
    site.ts                 Company + contact details  ← edit this
  lib/
    dictionaries/en.ts      English copy (source of truth for shape)
    dictionaries/ar.ts      Arabic copy (type-checked against en)
    i18n.ts                 Locale helpers
    metadata.ts             Per-page metadata builder
scripts/
  check-i18n.mjs            Arabic content linter
  check-links.mjs           Internal link / route checker
```

---

## Internationalisation

Every route lives under `src/app/[locale]/`. `proxy.ts` detects the visitor's
`Accept-Language` (or the `NEXT_LOCALE` cookie set by the language switcher) and
redirects to `/en/...` or `/ar/...`.

- `src/lib/dictionaries/en.ts` is the source of truth. `ar.ts` is typed as
  `Dictionary`, so a missing or misspelled Arabic key is a **build error**.
- Layout uses CSS logical properties throughout (`ms-`, `pe-`, `start-`, `end-`)
  so RTL needs no duplicated styles. Arabic renders `<html dir="rtl">`.
- The language switcher keeps the current path (`/en/products/trucks` ⇄
  `/ar/products/trucks`) and remembers the choice in a cookie.

### The i18n linter

Long bilingual strings are easy to corrupt while editing, and a stray Latin
word inside an Arabic sentence ships silently. `npm run check:i18n` flags:

- `[latin]` — Latin words in Arabic copy that are not known-good technical
  terms (brand names, standards, units)
- `[repl]` — U+FFFD replacement characters, i.e. lost text
- `[shape]` — arrow glyphs in Arabic copy

Known-good terms are allow-listed at the top of `scripts/check-i18n.mjs`.

---

## Product photography

Photographs are **optional per line**. As shipped, every product renders a
branded placeholder that names the exact file the slot expects, so the site
never shows a broken or invented image.

**To add a photo:**

1. Put the file at `public/products/<division>-<NN>.jpg` — see
   `public/products/README.md`. There are 24 slots, 6 per division.
2. Add an `image` field to that line in `src/data/catalog.ts`:

```ts
image: {
  src: "trucks-01.jpg",
  alt: {
    en: "4×2 tractor unit with curtain-sider cab, three-axle",
    ar: "وحدة جر 4×2 بكبوت بستارة جانبية وثلاثة محاور",
  },
},
```

`next/image` handles optimisation, responsive `srcset` and lazy loading.
`alt` is required once `src` is set.

Run `npm run images:manifest` to regenerate `public/products/manifest.json`,
a checklist of every slot.

On placeholders rather than stock photography: on a sourcing site, a generic
photo of *an* excavator is worse than none, because a buyer cannot tell whether
it is the unit they would receive. Photograph the actual goods.

---

## Brand

Palette, logo and typography follow the Jinan Sardia brand guidelines:

| Token | Hex | Role |
| --- | --- | --- |
| `--color-brand-500` | `#D91E36` | Red — dominant (60%) |
| `--color-green-brand-500` | `#008752` | Green — supporting (20%) |
| `--color-navy-800` | `#001F5B` | Navy — structural (10%) |
| `--color-brand-grey-400` | `#C5C6C8` | Grey — neutral (10%) |

Base values are sampled from the guideline imagery and live in the `BRAND`
object in `src/data/site.ts`. Replace them there and re-run
`npm run check:contrast` if you have the official values.

**Contrast note:** brand red on navy is only 3.11:1, which fails WCAG AA for
body text. The palette therefore carries lightened tints
(`--color-brand-on-navy`, `--color-green-on-navy`), and each division carries
`accentOnDark` / `accentText` steps. Do not use raw `brand-500` or
`brand-grey-700` as text on a navy ground.

---

## Connecting the enquiry form

`submitEnquiry()` in `src/app/actions.ts` records every submission. It has
four layers, in order:

1. **Durable write (always runs)** — appends JSON to `data/enquiries.jsonl`.
   Needs no configuration, so a lead cannot be lost to a missing env var. The
   directory is git-ignored. On a read-only or serverless filesystem this
   fails, and the layer below takes over.
2. **Webhook** — set `SARDIA_WEBHOOK_URL` and the payload is POSTed as JSON.
3. **Email** — set `RESEND_API_KEY` and a formatted alert is sent via the
   Resend REST API (no extra dependency). Optional env vars:
   `SARDIA_ALERT_EMAIL`, `SARDIA_FROM_EMAIL`.
4. **Spam** — a honeypot field (`company_website`) is wired up; if filled, the
   action returns success without doing anything. Plus a rate limit of 5
   submissions per minute per IP.

Two deliberate behaviours worth knowing:

- **Failures are never reported as success.** If neither the durable write nor
  any transport succeeded, the action returns an error and the visitor sees a
  red banner with your email address — because their message genuinely was not
  received. Silently swallowing that is how lead-gen sites lose enquiries.
- **The visitor's input is preserved** on any failure, so retrying is one click.

### Rate limiting is a speed bump, not a control

It is in-memory and keyed on the client IP, so it resets on deploy and is not
shared across instances. When the IP cannot be determined it **fails open**
rather than bucketing everyone together — one spammer locking out all real
buyers is worse than an extra spam submission. Put a WAF (Cloudflare, etc.)
in front if you need a real limit.

### Checking enquiries

```bash
# most recent enquiry
tail -n 1 data/enquiries.jsonl | jq .

# all RFQs
jq -c 'select(.intent=="rfq")' data/enquiries.jsonl
```

---

## Before going live

Everything below is a placeholder and must be replaced in `src/data/site.ts`:

- [ ] `email`, `phone`, `whatsapp` — currently `+86 531 0000 0000`
- [ ] `address` / `addressLine2` — currently just "Jinan, Shandong Province, China"
- [ ] `founded`, and the homepage statistics (factories / markets / on-time %)
      — **these were invented to lay out the page and are not measured.** They
      live in `home.statsSection` in `src/lib/dictionaries/{en,ar}.ts`, not in
      `site.stats` (which is not rendered). Replace or delete them.
- [ ] Set `NEXT_PUBLIC_SITE_URL` (used for canonical URLs, hreflang, sitemap)
- [ ] Set `SARDIA_WEBHOOK_URL` and/or `RESEND_API_KEY` so enquiries are
      delivered. Without either, they are only written to `data/enquiries.jsonl`.
- [ ] Review all price ranges in `src/data/catalog.ts` — they are orientation
      ranges, not quotes. They also feed the per-division JSON-LD
      `lowPrice`/`highPrice`, so Google may surface them as prices.
- [ ] Drop the official logo vector into `src/components/Logo.tsx` (the current
      mark is a reconstruction from the guidelines)
- [ ] Add product photographs — see "Product photography" above

Product prices, MOQ and lead times in `src/data/catalog.ts` are realistic
market ranges, not quotations. The site labels them as indicative; keep it
that way unless you have firm data.

---

## Notes on a few decisions

- **Trading-company disclosure.** The footer states plainly that Sardia is a
  sourcing company, not the manufacturer. This is deliberate — it is both
  legally safer and a trust signal with serious buyers.
- **Scroll reveal is fail-safe.** `.reveal` is only hidden when
  `<html data-js="on">` is set by the component itself, plus a 2s failsafe
  timer. If JS fails, content is visible rather than invisible.
- **`metadataBase` is per page.** A page's `generateMetadata` return value is
  not merged with the layout's for `metadataBase`, so `lib/metadata.ts` sets it
  on every page; without it, OG image URLs resolve against `localhost`.