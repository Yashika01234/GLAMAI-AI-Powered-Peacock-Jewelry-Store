# Glam AI — Peacock Week 🦚

A full-stack e-commerce storefront for a fictional festive sale ("Peacock Week") on
peacock-motif earrings. Built as a portfolio project to demonstrate a complete
client/server architecture: a React SPA talking to a real REST API backed by a JSON-file data store.

**Live demo isn't hosted anywhere** — this is a self-contained project you run locally
in two commands. See [Getting Started](#getting-started) below.

---

## Tech stack

| Layer      | Choice                                             |
|------------|-----------------------------------------------------|
| Frontend   | React 18, React Router 6, Vite                     |
| Styling    | Hand-written CSS (design tokens, glassmorphism, no framework) |
| Backend    | Node.js, Express                                   |
| Database   | Plain JSON file (`server/data.json`), read/written via Node's built-in `fs` -- zero native dependencies |
| State      | React Context (cart, wishlist, toast notifications) |
| Dev tooling| `concurrently` to run both servers with one command |

No external/paid APIs are used anywhere — including the "AI" chatbot, which is a
server-driven decision tree (see [Design notes](#design-notes)).

## Features

- **Product catalogue** — filterable by category, sortable by price/discount, served from a real database
- **Product detail pages** — routed per product (`/product/:id`), with quantity selection
- **Server-persisted cart** — cart lives in `server/data.json`, keyed by a client-generated id (`X-Cart-Id` header), so it survives refreshes without requiring a login
- **Wishlist** — client-side, persisted to `localStorage`
- **Checkout flow** — client + server validated form → creates a real `orders` row → order confirmation page fetched by id
- **Chatbot widget** — fixed-answer decision tree served from `GET /api/chat/nodes`, walked entirely client-side after the first fetch (no network round-trip per message, no AI API cost)
- **Countdown timer**, glassmorphism UI, responsive layout, loading skeletons, empty/error states

## Architecture

```
┌─────────────────────┐        HTTP (JSON)        ┌──────────────────────┐
│   React (Vite)       │ ───────────────────────▶ │   Express API         │
│   localhost:5173      │ ◀─────────────────────── │   localhost:4000       │
│                       │                           │                       │
│  Pages: Home,         │   /api/products           │  Routes:              │
│  ProductDetail,       │   /api/cart                │   products.js         │
│  Checkout,             │   /api/orders               │   cart.js             │
│  OrderConfirmation     │   /api/chat                 │   orders.js           │
│                       │                           │   chat.js             │
│  Context: Cart,       │                           │                       │
│  Wishlist, Toast       │                           │  fs (built-in) ──▶   │
└─────────────────────┘                           │  data.json (file)     │
                                                    └──────────────────────┘
```

In dev, Vite proxies any request to `/api/*` through to the Express server
(see `client/vite.config.js`), so the browser only ever talks to one origin.

## Folder structure

```
glamai-fullstack/
├── package.json          # root: runs client + server together
├── server/
│   ├── index.js           # Express app + route mounting
│   ├── db.js               # loadData()/saveData() for the JSON store
│   ├── seed.js              # seeds the products table on first run
│   └── routes/
│       ├── products.js      # GET /api/products, GET /api/products/:id
│       ├── cart.js           # GET/POST/PATCH/DELETE /api/cart
│       ├── orders.js          # POST /api/orders, GET /api/orders/:id
│       └── chat.js             # GET /api/chat/nodes
└── client/
    ├── vite.config.js
    ├── index.html
    └── src/
        ├── main.jsx           # entry point, providers
        ├── App.jsx             # routes
        ├── api.js               # fetch wrapper for the backend
        ├── palettes.js           # colour tokens for the SVG art
        ├── context/               # CartContext, WishlistContext, ToastContext
        ├── components/             # Header, Hero, ProductCard, Chatbot, CartDrawer, ...
        ├── pages/                   # Home, ProductDetail, Checkout, OrderConfirmation
        └── styles/global.css         # all styling
```

## Getting started

**Prerequisites:** Node.js 18 or newer (`node -v` to check), npm.

```bash
# 1. Install dependencies for both the server and the client
npm run install:all

# 2. Start both the API (port 4000) and the frontend (port 5173)
npm run dev
```

Then open **http://localhost:5173** — the data file (`server/data.json`) is created
and seeded with products automatically on first launch.

To run each side individually:

```bash
npm run server   # Express API only, http://localhost:4000
npm run client   # Vite dev server only, http://localhost:5173
```

### Building for production

```bash
npm run build --prefix client   # outputs client/dist (static build)
npm run start --prefix server    # runs the API without file-watching
```

You'd serve `client/dist` from any static host and point it at a deployed
instance of `server/` (or have Express serve the built files directly —
not wired up here to keep the two concerns separate for the demo).

## Troubleshooting

- **`npm error Missing script` right after extracting the zip.** Some zip tools
  (notably Windows' built-in "Extract All") create an extra nested folder, so you
  end up with `glamai-fullstack/glamai-fullstack/...`. Run `dir` (or `ls`) in your
  terminal — if you see another `glamai-fullstack` folder instead of `server`/
  `client`/`package.json`, `cd` into it once more.
- **`ERR_MODULE_NOT_FOUND: Cannot find package 'express'`.** This means
  `npm install` didn't fully complete in that folder (often because you were
  actually one directory off, see above). Run `npm install` again from the exact
  folder that contains that route file's `package.json`.
- **This project has no native dependencies on purpose** — the "database" is a
  plain JSON file written with Node's built-in `fs` module, specifically so
  nobody running this needs a C++ compiler or Visual Studio Build Tools
  installed. If a fresh `npm install` ever tries to run `node-gyp`/`gyp`, that
  means a dependency was added that needs native compilation — worth reverting.

## API reference

| Method | Endpoint                | Body                                                   | Notes                          |
|--------|--------------------------|---------------------------------------------------------|---------------------------------|
| GET    | `/api/products`           | —                                                        | Query: `category`, `sort`        |
| GET    | `/api/products/:id`         | —                                                        |                                   |
| GET    | `/api/cart`                   | —                                                        | Requires `X-Cart-Id` header        |
| POST   | `/api/cart`                     | `{ productId, qty }`                                       | Adds / increments an item           |
| PATCH  | `/api/cart/:productId`             | `{ qty }`                                                     | `qty <= 0` removes the item           |
| DELETE | `/api/cart/:productId`                | —                                                              |                                          |
| DELETE | `/api/cart`                              | —                                                                | Clears the whole cart                     |
| POST   | `/api/orders`                              | `{ customerName, email, address, city, pincode }`                  | Converts the current cart into an order     |
| GET    | `/api/orders/:id`                             | —                                                                    |                                                |
| GET    | `/api/chat/nodes`                                | —                                                                      | Returns the entire canned chat tree              |

## Design notes

- **Product "photos" are generated SVG, not stock images.** Each earring is drawn
  procedurally (`components/PeacockArt.jsx`) from a small set of paths and a colour
  palette. This keeps the repo dependency-free (no image licensing, no broken
  hotlinks) while still looking intentional and on-theme. Swapping in real product
  photography later just means rendering an `<img>` instead of `<PeacockArt />`
  in `ProductCard` / `ProductDetail`.
- **The cart has no user accounts.** A random id is generated on first visit
  (`localStorage`) and sent as `X-Cart-Id` on every cart/order request, so the
  server can persist cart state per browser without building auth. This is a
  common pattern for guest checkout flows.
- **The chatbot is intentionally not AI.** It's a REST-served decision tree —
  real client/server architecture, zero hallucination risk, zero API cost,
  instant responses. Swapping in a real LLM later is a matter of replacing the
  `goTo()` lookup in `Chatbot.jsx` with a call to a completions endpoint.

## Possible extensions (good "what I'd do next" talking points)

- Admin dashboard for CRUD on products (schema already supports it)
- Real authentication (JWT) so carts/orders belong to a user, not just a browser
- Product image upload + storage instead of generated art
- Payment integration (Stripe test mode) on the checkout flow
- Unit tests (Vitest for the client, Supertest for the API routes)
- Deploy: Vercel/Netlify for the client, Render/Railway for the API. Swap the JSON store for Postgres/SQLite once you need concurrent writers or real persistence guarantees

## License

MIT — do whatever you like with it, including using it as a portfolio piece.
