# The Mania (themania) — Fashion Store

Online store for **The Mania**, a premium women's fashion brand. Shoppers can browse the catalogue, filter and search it, add items to a bag, and design a custom dress in a colour and size studio that sends the order through WhatsApp. An admin studio at `/admin` manages products, stock, sale prices, signature designs, site text and customer messages.

The project has two parts in one repository: a **React + Vite** storefront (`client/`) and an **Express + Prisma + PostgreSQL** API (`server/`).

![Home page](https://mujtabaasif.vercel.app/assets/projects-screenshots/the-mania/home.webp)

**Live site:** https://the-mania-two.vercel.app/ · **Portfolio:** https://mujtabawd.vercel.app/

## Tech stack

| Area | Choice |
| --- | --- |
| Frontend | React 18 (JavaScript/JSX), Vite 5 |
| Styling | Tailwind CSS loaded from its CDN script in `client/index.html`, Font Awesome 6 icons |
| Fonts | Playfair Display (headings), Montserrat (body), Bodoni Moda (wordmark) |
| Backend | Node.js 20+, Express 5 |
| Database | PostgreSQL through Prisma 6 (Neon in production) |
| Email | Nodemailer over Gmail SMTP (optional) |
| Analytics | `@vercel/analytics` |
| Hosting | Two Vercel projects: `client/` as a static site, `server/` as a serverless function |

## Getting started

Requirements: Node.js 20 or newer and a PostgreSQL database (local or hosted).

```bash
# 1. Install (root installs both workspaces)
npm install

# 2. Configure the API
cp server/.env.example server/.env      # then set DATABASE_URL
cp client/.env.example client/.env      # leave VITE_API_URL empty for local work

# 3. Create the tables and load the sample catalogue (20 products, 4 designs)
cd server && npm run db:deploy && cd ..

# 4. Run API (port 5000) and storefront (port 5173) together
npm run dev
```

Open <http://localhost:5173>. The Vite dev server proxies `/api/*` to `http://localhost:5000`, so the client needs no API address locally.

### Scripts

| Where | Script | What it does |
| --- | --- | --- |
| root | `npm run dev` | Start API and storefront together (`concurrently`) |
| root | `npm run dev:client` / `dev:server` | Start one side only |
| root | `npm run build:client` | Production build of the storefront into `client/dist/` |
| root | `npm run db:push` | Sync the Prisma schema to the database |
| root | `npm run db:seed` | Load sample products and designs (skipped if tables already have rows) |
| server | `npm run db:deploy` | `db push` + both seed scripts, used once per new database |
| server | `npm start` | Run the API without auto-reload |

There are no automated tests or lint scripts yet.

## Environment variables

**`server/.env`**

| Variable | Required | Purpose |
| --- | --- | --- |
| `DATABASE_URL` | Yes | Postgres connection string. In production use Neon's **pooled** (`-pooler`) string. |
| `CLIENT_URL` | Yes | Allowed CORS origin(s), comma-separated. `*.vercel.app` previews are always allowed. |
| `PORT` | Local only | Defaults to `5000`. Not used on Vercel. |
| `NODE_ENV` | No | `development` adds stack traces to error responses. |
| `GMAIL_USER`, `GMAIL_APP_PASSWORD`, `ADMIN_NOTIFICATION_EMAIL` | No | Contact-form emails. Without them, messages are still saved and shown in the admin. |

**`client/.env`**

| Variable | Purpose |
| --- | --- |
| `VITE_API_URL` | API origin in production, e.g. `https://themania-api.vercel.app` (no trailing slash, no `/api`). Leave empty locally. Vite bakes it in at build time, so **redeploy after changing it**. |

## Project structure

```
client/
├── index.html               Tailwind CDN config, fonts, Font Awesome
├── vite.config.js           Dev server + /api proxy to :5000
├── vercel.json              SPA fallback: every path → index.html
└── src/
    ├── App.jsx              Hand-rolled router: URL → page, link interception, back/forward
    ├── pages/               One component per route (24 pages)
    ├── components/          Header, Footer, Hero, product/grid/gallery/tabs, CustomDesign,
    │                        ColorPicker, Toast, profile tabs, Admin* panels
    ├── data/                siteContentDefaults (hero/footer/SEO fallbacks), colorFamilies,
    │                        teamMembers
    └── utils/
        ├── api.js           fetch wrapper; prefixes VITE_API_URL
        ├── products.js      fetchAllProducts / fetchProductById (+ imageUrl → image)
        ├── cart.js          Cart in localStorage ("forren-cart") + "cart-updated" event
        ├── pricing.js       getEffectivePrice / isOnSale (honours sale window)
        ├── useSiteContent.js / useSeo.js   Editable content + <title>/meta from the API
        └── adminTable.js, csv.js, adminUi.js, toast.js
server/
├── api/index.js             Vercel serverless entry (lazy-loads the Express app)
├── vercel.json              Rewrites every path to /api
├── prisma/
│   ├── schema.prisma        Data model (below)
│   ├── seed.products.js     20-product catalogue
│   └── seed.designs.js      4 signature designs
└── src/
    ├── app.js               Express app, CORS, JSON, routes, 404 + error handlers
    ├── server.js            Local entry: app.listen(PORT)
    ├── config/              env.js, prisma.js (shared client cached on globalThis)
    ├── routes/ → controllers/ → services/    one trio per resource
    ├── middleware/          notFound, errorHandler
    └── utils/               ApiError, asyncHandler
render.yaml                  Optional alternative: run the API on Render instead of Vercel
```

## Pages and URLs

Routing is written by hand in [client/src/App.jsx](client/src/App.jsx) with no router library. A document-level click handler catches `<a href>` clicks to known paths, calls `history.pushState`, and swaps the page. Back/forward and deep links work. Unknown paths show the home page.

| URL | Page | Data source |
| --- | --- | --- |
| `/` | Home: hero, editorial collection tiles, featured tabs, services, Instagram grid, testimonials, newsletter | API (products, hero text) |
| `/collections/new-arrivals`, `/bestsellers`, `/summer-collection`, `/winter-essentials` | Home with that featured tab pre-selected | API |
| `/shop` | Catalogue. Query params: `?category=`, `?sort=newest\|bestselling\|price-low\|price-high`, `?search=` | API |
| `/product/:id` | Product detail: gallery, colours, sizes, stock bar, quantity, tabs, related items | API |
| `/cart` | Bag with quantity controls and order summary | localStorage |
| `/checkout` → `/order-confirmation` | Shipping and payment form, then a thank-you page | Static (see status) |
| `/custom-design` | Custom dress studio: reference design or photo, colour, style, size, measurements → WhatsApp | API (designs) |
| `/about`, `/careers`, `/journal`, `/lookbook`, `/sustainability` | Brand pages | Static |
| `/contact` | Contact form | API (saves message) |
| `/faq`, `/shipping`, `/return`, `/payment`, `/size-guide`, `/policy`, `/term`, `/accessibility` | Help and legal pages | Static |
| `/profile` (alias `/account`), `/wishlist` | Account area: profile, security, notifications, preferences, addresses, payment, orders | Static mock-up |
| `/admin` | Admin Studio | API |

**Adding a page:** create it in `client/src/pages/`, add its path to `pathToPage` in `App.jsx`, and add a `case` in `renderPage()`.

## API

Base path `/api`. Responses are `{ status: "ok", data, count? }`. Errors are `{ status: "fail" | "error", message }`.

| Method | Path | Purpose |
| --- | --- | --- |
| GET | `/health` | Health check |
| GET | `/products` (`?category=`) | List products |
| GET | `/products/featured`, `/products/low-stock` | Featured products; products at or below their low-stock threshold |
| GET / POST / PUT / DELETE | `/products`, `/products/:id` | Read and manage products (slug is generated automatically) |
| GET / POST | `/products/:id/stock-history`, `/products/:id/stock` | Stock audit trail; adjust stock by a signed `delta` with a `reason` |
| POST | `/products/bulk-price` | `{ category, percent }` sets a sale price for a category; `{ clear: true }` removes it |
| GET / POST / PUT / DELETE | `/designs`, `/designs/:id` | Signature designs (`?all=true` includes inactive, `?featured=true`) |
| POST / GET / DELETE | `/contact`, `/contact/:id` | Submit (and email) a message; admin list/delete |
| GET / POST | `/careers`, `/careers/apply` | Open roles (hard-coded); submit an application |
| GET / DELETE | `/careers/applications`, `/careers/applications/:id` | Admin list/delete |
| GET / PUT | `/content`, `/content/:key` | Editable site content (JSON per key) |
| GET | `/dashboard/stats` | Counts, low stock, inventory value, recent items |

## Data model

Defined in [server/prisma/schema.prisma](server/prisma/schema.prisma). The array and object fields (`colors`, `sizes`, `images`, `details`…) are stored as JSON **text** and parsed by the services, so API clients always receive real arrays and objects.

| Model | Holds |
| --- | --- |
| `Product` | Name, slug, category, price, image(s), stock + low-stock threshold, SKU, sale price with optional start/end dates, tag, rating, reviews, colours, sizes, key features, details, care/shipping/return info, related product ids |
| `StockAdjustment` | Each manual stock change (delta, reason, note). Written in the same transaction as the stock update. |
| `Design` | Signature designs for custom orders: base price, colours, sizes, featured, active |
| `SiteContent` | Key/value JSON: `homepage.hero`, `footer.brand`, `seo.home`, `seo.about`, `seo.contact`, `seo.faq`, `seo.terms` |
| `ContactMessage`, `CareerApplication` | Form submissions |

## Admin Studio (`/admin`)

| Tab | What it does |
| --- | --- |
| Dashboard | Product, design, message and application counts; low-stock list; inventory value; recent products and messages |
| Products | Add/edit form with every storefront field; searchable, sortable, paginated table; stock adjustments with history; CSV export |
| Pricing | Percentage discount for a whole category (or all), or clear sales |
| Designs | Create, edit, feature and hide signature designs for the custom studio |
| Content | Home hero text and buttons, footer text and social links, page titles and meta descriptions |
| Messages / Applications | Search, sort, delete, CSV export |

> **Warning:** `/admin` has no login, and none of the write endpoints (`POST`/`PUT`/`DELETE`) check who is calling. Anyone who finds the API address can change products, prices and content. Add authentication before relying on the live site.

## Current feature status

| Feature | Status |
| --- | --- |
| Catalogue, filters, sort, search, pagination | Working (API) |
| Product pages, sale pricing, stock display | Working (API) |
| Bag / cart | Working, saved in the browser (`localStorage`) |
| Checkout and order confirmation | **Not connected.** Form only. The summary shows `$0.00`, no order is stored, no payment is taken, the cart isn't cleared, and the confirmation shows placeholder text. |
| Custom design studio | Working. Sends the order as a WhatsApp message. |
| Contact form | Working. Saved to the database, plus email if Gmail is configured. |
| Careers "Apply" | Opens an email (`mailto:`). The `/api/careers/apply` endpoint exists but is not used by the page. |
| Newsletter, wishlist, account/profile, order history | UI only, nothing is saved |
| Admin Studio | Working (API), but unprotected |

### Known issues

- **Signature designs are empty in production.** [CustomDesign.jsx](client/src/components/CustomDesign.jsx) calls `fetch("/api/designs")` directly instead of `api.get(...)`, so it ignores `VITE_API_URL`.
- **Custom-order WhatsApp link has no number.** It opens `https://wa.me/?text=…`, so the customer has to pick a chat. The floating button uses `wa.me/923214574973`.
- **Free-shipping threshold is inconsistent.** The product page says "over $100". The cart and home page use $150.
- The shop's price slider and price sorting use the base price, not the sale price.
- Tailwind runs from the CDN script, which is meant for prototyping. For production, install Tailwind as a build step.
- `client/src/components/Header copy.jsx` is an unused leftover.
- `DEPLOYMENT.md` says the seed loads 17 products. It loads 20.

## Deployment

See [DEPLOYMENT.md](DEPLOYMENT.md). In short, one repository becomes two Vercel projects:

1. **Database.** Create a Neon Postgres database and copy its **pooled** connection string. Run `DATABASE_URL="…" npm run db:deploy` in `server/`.
2. **API.** Create a Vercel project with Root Directory `server` and Framework **Other**. Set `DATABASE_URL` and `CLIENT_URL`. Check `/api/health`.
3. **Storefront.** Create a Vercel project with Root Directory `client`. Set `VITE_API_URL` to the API origin, then **redeploy**.

`render.yaml` is kept as an alternative way to host the API as a long-running server.

## Brand

| Token | Value |
| --- | --- |
| Ink (header, footer, buttons) | `#09090B` (zinc-950) / `#141110` |
| Accent gold | `#F59E0B` (amber-500), `#B45309` (amber-700) |
| Cream backgrounds | `#F8F3ED`, `#F6EFE6`, `#FAF5EE` |
| Wordmark dot / admin accent | `#DA1E4C`, `#E11D48` (rose-600) |
| Currency | US dollars |
