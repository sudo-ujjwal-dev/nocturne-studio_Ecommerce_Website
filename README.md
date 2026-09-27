# Nocturne Studio

A full-stack e-commerce site built for a CodeAlpha Web Development internship -
vanilla HTML/CSS/JS on the frontend, Express + Prisma + MySQL on the backend.

Brand concept: objects designed for the hours after work ends — lighting,
audio, travel, and everyday goods shot and written as one coherent product
line. Tagline: *"Made for the quiet hours."*

---

## 1. Project structure

```
nocturne-studio/
├── public/                  # Frontend — plain HTML/CSS/JS, no framework, no build step
│   ├── index.html            shop.html          product.html
│   ├── cart.html             checkout.html      login.html
│   ├── register.html         account.html       order-success.html
│   ├── css/
│   │   ├── base.css          design tokens, reset, typography
│   │   ├── components.css    nav, buttons, cards, drawer, toasts, forms
│   │   ├── pages.css         hero, homepage sections, shop/PDP/checkout layouts
│   │   └── responsive.css    cross-cutting breakpoint refinements
│   └── js/
│       ├── api.js            the only file that calls fetch()
│       ├── ui.js             header, mobile menu, toasts, search overlay
│       ├── auth.js           register/login forms, session state
│       ├── cart.js           localStorage cart, drawer, cart page
│       ├── products.js       homepage rails, shop grid, product detail page
│       ├── checkout.js       checkout form → POST /api/orders
│       └── account.js        account page + order history + order confirmation
│
└── server/                  # Backend — Express + Prisma + MySQL
    ├── prisma/
    │   ├── schema.prisma      User, Product, Order, OrderItem models
    │   └── seed.js            18 products + 1 demo account
    ├── src/
    │   ├── routes/            auth, products, orders, users
    │   ├── controllers/       thin HTTP layer
    │   ├── services/          business logic (auth, products, orders/checkout)
    │   ├── middleware/        JWT auth guard, central error handler
    │   ├── validators/        request body validation
    │   └── utils/             Prisma client, JWT helpers, serializers, ApiError
    ├── server.js
    ├── package.json
    └── .env.example
```

---

## 2. Requirements

- Node.js 18+
- A running MySQL server (local install, Docker, or a hosted instance)

---

## 3. Backend + frontend setup (one server runs both)

The Express server now serves the `/public` frontend directly, so there's
only one process to run — no separate static server, no CORS to configure
between frontend and backend.

```bash
cd server
npm install
cp .env.example .env
```

Edit `.env`:

```
DATABASE_URL="mysql://USER:PASSWORD@localhost:3306/nocturne_studio"
JWT_SECRET="replace-with-a-long-random-string"
```

Create the database, run migrations, and seed it:

```bash
# Create an empty schema first, e.g.:
#   mysql -u root -p -e "CREATE DATABASE nocturne_studio"

npx prisma migrate dev --name init
npm run prisma:seed
```

Start the server:

```bash
npm run dev        # nodemon, restarts on change
# or
npm start
```

Open **http://localhost:4000** in your browser — that's the whole site.
Confirm the API itself is up at `http://localhost:4000/api/health`.

---

## 4. Deploying online

See the deployment walkthrough (Render + a free Aiven MySQL database) in the
chat where this project was built, or follow these steps:

1. **Database:** create a free MySQL instance on [aiven.io](https://aiven.io)
   (always-free tier, no card required). Copy its connection details and
   download its CA certificate.
2. **Backend + frontend:** push this repo to GitHub, create a Render Web
   Service pointed at the `server/` directory (build: `npm install && npx
   prisma generate`, start: `npx prisma migrate deploy && node server.js`).
   Set `DATABASE_URL`, `JWT_SECRET`, and `NODE_ENV=production` as environment
   variables on Render.
3. Run `npm run prisma:seed` once against the production database (Render
   Shell, or locally with `DATABASE_URL` pointed at production) to load the
   catalog and demo account.
4. Visit the Render URL — frontend and API are served from the same
   deployment, so there's nothing else to configure.

---

## 5. Test account

A demo account is created by the seed script:

```
email:    demo@nocturne.studio
password: Password123!
```

Or register a new one from `register.html` — registration is fully live
against the database.

---

## 6. API summary

All responses are JSON: `{ success, data?, message?, details?, meta? }`.

| Method | Route                    | Auth | Description                                   |
|--------|--------------------------|------|------------------------------------------------|
| POST   | `/api/auth/register`     | –    | Create account, returns user + JWT             |
| POST   | `/api/auth/login`        | –    | Log in, returns user + JWT                     |
| GET    | `/api/auth/me`           | ✓    | Current user                                   |
| POST   | `/api/auth/logout`       | –    | Clears the auth cookie                         |
| GET    | `/api/products`          | –    | List products — search/category/price/sort/pagination |
| GET    | `/api/products/:id`      | –    | Product detail by numeric id                   |
| GET    | `/api/products/slug/:slug` | – | Product detail by slug, includes related items |
| POST   | `/api/orders`            | ✓    | Place an order (server validates & prices everything) |
| GET    | `/api/orders`            | ✓    | Current user's order history                   |
| GET    | `/api/orders/:id`        | ✓    | One order — 403 if it isn't yours              |
| GET    | `/api/users/me`          | ✓    | Current user (alias of `/api/auth/me`)         |

Auth: send `Authorization: Bearer <token>` (the frontend stores the token in
`localStorage` and attaches it automatically via `api.js`).

**Query params for `GET /api/products`:** `page`, `limit`, `sort`
(`newest` | `price_asc` | `price_desc` | `rating` | `name`), `category`,
`search`, `minPrice`, `maxPrice`, `featured=true`, `isNew=true`.

---

## 7. How checkout integrity works

`POST /api/orders` never trusts anything the browser sends about price or
stock. Inside a single Prisma transaction it:

1. Re-reads every product from the database by id.
2. Decrements stock with the availability check built into the same atomic
   `UPDATE … WHERE stock >= quantity` — so two simultaneous checkouts can't
   both succeed on the last unit; the loser gets a 409 and the whole order
   rolls back.
3. Computes subtotal/shipping/total from the database prices it just read.
4. Creates the order and its line items.

If any step fails, nothing is written — no partial orders, no stock drift.

---

## 8. Known limitations

- Product photography is seeded from placeholder images (picsum.photos),
  color-graded via CSS to look like one consistent shoot. Swap `image` /
  `gallery` URLs in `prisma/seed.js` for real photography.
- Payment is a demo UI only (per the assignment brief) — no card data is
  collected or transmitted; only the shipping/contact fields are sent to the
  server.
- Order status (`PENDING` → `PROCESSING` → `COMPLETED` → `CANCELLED`) is set
  once at creation; there's no admin panel to transition it. Extending this
  would mean adding an admin-only route guarded by a `role` field on `User`.
- The `.env` shipping/free-shipping values are duplicated as constants in
  `cart.js` purely to render an *estimate* client-side; the order actually
  charged always comes from the server's own calculation.
