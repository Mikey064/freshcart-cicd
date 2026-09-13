# FreshCart

FreshCart is a small grocery-delivery app you'll build on throughout this program. It's not a finished product — it's the same working codebase you'll containerize, provision infrastructure for, ship through a pipeline, orchestrate, and monitor as the course goes on. Same app, every week, so the work compounds instead of starting over each time.

## What's actually here

Two services and a database:

- **`checkout-api/`** — an Express + TypeScript API backed by Postgres. Lists products, handles search, and places orders.
- **`storefront/`** — a static site (Vite + TypeScript, no framework) that calls the checkout API. Builds to plain HTML/CSS/JS.
- **Postgres** — not included in this repo. You'll stand it up yourself, in whatever way each week's lesson asks for.

Neither service ships with a Dockerfile, a `docker-compose.yml`, or any infrastructure code. That's intentional — writing those *is* the assignment in the weeks that need them.

## Running it locally, before any of that

You'll need a Postgres database reachable from your machine, and Node.js 20+.

**checkout-api**
```
cd checkout-api
cp .env.example .env      # edit DATABASE_URL to point at your Postgres
npm install
npm run dev                # tsx watch, restarts on save
```
Load `db/init.sql` into your database once, however you'd normally run a `.sql` file against Postgres — it creates the schema and seeds about 15 products. (If you're loading it into a container's Postgres, note that this file is written to work automatically if placed in Postgres's official `/docker-entrypoint-initdb.d/` — worth knowing before Week 4.)

Once it's running: `curl localhost:3000/healthz` should return `{"status":"ok"}`, and `curl localhost:3000/api/products` should return the seeded list.

**storefront**
```
cd storefront
npm install
npm run dev
```
Vite's dev server proxies `/api` to `localhost:3000` automatically (see `vite.config.ts`) — open the URL it prints, and you should see a product grid you can actually buy from.

## API reference

| Method | Path | Does |
|---|---|---|
| GET | `/healthz` | Checks the API can reach the database. |
| GET | `/api/products` | Lists all products. Add `?search=term` to filter by name. |
| GET | `/api/products/:id` | One product. |
| POST | `/api/orders` | `{ customerName, customerEmail, items: [{ productId, quantity }] }`. Validates stock, records the order transactionally. |
| GET | `/api/orders/:id` | An order and its line items. |


## Environment variables

**checkout-api** (`.env.example`): `DATABASE_URL`, `PORT`.
**storefront** (`.env.example`): `VITE_API_BASE_URL` — leave unset for local dev (the Vite proxy handles it); set it for a production build where the storefront is deployed separately from the API.

## Activities
1. Screenshots of outcomes at different stages of this project can be found in `/screenshots` folder.

2. `Architecture.md` explains what has been done so far in this project and the reason why the decisions were made was also explained in it. Check `./docs/Architecture.md`

3. Inside the /docs/ folder, the multi-stage layer diagrams for `checkout-api` and `storefront` as well as container topology diagram can be found in:
 - `./docs/checkout-api-diagram.jpg`
 - `./docs/storefront-diagram.jpg`
 - `./docs/container-topology-diagram.jpg`

 ## Blog Post
 Medium: ["From “Works on My Machine” to Repeatable Builds: Containerizing FreshCart with Docker"](https://michaelokpu.medium.com/from-works-on-my-machine-to-repeatable-builds-containerizing-freshcart-with-docker-1154d8cd92f7?sharedUserId=michaelokpu)
