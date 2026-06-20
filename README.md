# Digital Menu SaaS — Technical Backbone

This is the foundational scaffold: auth, multi-tenant data model, menu CRUD,
and public checkout. It is intentionally minimal — no Stripe payment capture
yet, no image upload pipeline yet. It's built so those slot in without
restructuring anything below.

## What's actually wired up end-to-end right now

- Register/login (NestJS + JWT + bcrypt) → creates a Business + empty Menu automatically
- Owner dashboard: add/delete categories, add/delete items, toggle item availability
- Public menu page at `/m/[slug]` — no login required
- Cart → checkout → order creation, with **server-side price calculation**
  (the frontend never sends a trusted total)
- Tenant isolation enforced via `TenantGuard` on every owner-facing route

## What's stubbed / not yet built (by design, per the roadmap)

- Stripe payment capture (orders are created as `PENDING`/`UNPAID` — see the
  comment block in `frontend/app/m/[slug]/page.tsx` and
  `backend/src/orders/orders.controller.ts` for exactly where this plugs in)
- Image upload (item `imageUrl` currently expects you to paste a URL)
- Owner order management dashboard UI (the API endpoints exist —
  `GET /businesses/:businessId/orders` — just no page consuming them yet)
- Real-time order push (polling/WebSockets)

## Project structure

```
digital-menu-saas/
├── backend/   NestJS API (Prisma + PostgreSQL + JWT auth)
└── frontend/  Next.js 14 (App Router) — dashboard + public menu + checkout
```

## Local setup

### Prerequisites
- Node.js 20+
- PostgreSQL running locally (or a connection string to a hosted instance —
  Supabase/Neon/Railway all work fine for development)

### 1. Backend

```bash
cd backend
npm install
cp .env.example .env
# edit .env: set DATABASE_URL to your actual Postgres connection string,
# and set JWT_SECRET to a long random string

npx prisma migrate dev --name init   # creates all tables from schema.prisma
npx prisma generate                   # generates the typed Prisma client

npm run start:dev                     # runs on http://localhost:3001
```

Useful commands while developing:
```bash
npx prisma studio       # GUI to browse/edit your database directly
npx prisma migrate dev  # run after any schema.prisma change
```

### 2. Frontend

```bash
cd frontend
npm install
cp .env.example .env.local
# NEXT_PUBLIC_API_URL should point at your backend, default is fine locally

npm run dev   # runs on http://localhost:3000
```

### 3. Try the full flow

1. Go to `http://localhost:3000/register`, create an account with a business
   name like "Joe's Pizza" → you land on `/dashboard`
2. Add a category ("Pizzas") and an item inside it ("Margherita", $12.50)
3. Open `http://localhost:3000/m/joes-pizza` (the slug is auto-generated from
   your business name — check the `businesses` table via `prisma studio` if
   you're not sure what slug you got)
4. Add the item to cart, fill in guest checkout, place the order
5. Confirm the order landed by hitting `GET /businesses/:businessId/orders`
   with your JWT (e.g. via Postman/Insomnia) — there's no dashboard UI for
   this yet, see "What's stubbed" above

## Where to plug in Stripe next

Two TODO comments mark the exact integration points:
- `backend/src/orders/orders.controller.ts` — after order creation, create a
  PaymentIntent for `order.totalCents` and return its `client_secret`
- `frontend/app/m/[slug]/page.tsx` (`CheckoutSheet` component) — mount Stripe
  Elements with that secret instead of calling `onSuccess` immediately

You'll also need a new webhook controller (raw body parsing + signature
verification) that flips `Order.status` to `CONFIRMED` and
`Order.paymentStatus` to `PAID` once Stripe confirms the charge — never trust
a "payment succeeded" message that comes directly from the browser.

## Security notes worth re-reading before you extend this

- `TenantGuard` (`backend/src/common/guards/tenant.guard.ts`) is the backstop
  against cross-tenant data access. Every new route nested under
  `/businesses/:businessId/...` needs `@UseGuards(JwtAuthGuard, TenantGuard)`.
- Every service method that touches Category/Item also re-verifies the
  resource belongs to the requesting business (see `assertItemBelongsToBusiness`
  in `items.service.ts`) — this is deliberate defense in depth, don't remove
  it even though the guard already checks the top-level `businessId`.
- Order totals are always computed server-side from the database, never
  accepted from the client (`orders.service.ts` → `createOrder`).
- Money is stored as integer cents everywhere in the database. The decimal
  ↔ cents conversion happens in exactly two places: `items.service.ts`
  (input) and `lib/menu.ts` `formatCents` (display). Don't add a third.
