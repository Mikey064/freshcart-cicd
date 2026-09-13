# FreshCart CI/CD (Week 6 Capstone)

This repo picks up where [`freshcart-tf`](https://github.com/Mikey064/freshcart-tf) (Week 5 — Terraform-provisioned GCP infra) left off. That capstone stood up the load balancer, private backend VM, and staging environment on GCP. This one automates getting `checkout-api` onto that infrastructure: a pull-request CI check, a build → scan → push pipeline authenticated with no long-lived keys, an automatic staging deploy, a manually-approved production deploy, and a real rollback.

> TODO (Mikey): confirm/replace — Artifact Registry repo name, staging VM instance name, and the pipeline diagram file path below before publishing.

## Architecture

![Pipeline diagram](docs/pipeline-diagram.png) <!-- TODO: confirm actual filename/path -->

Every job's trigger and target, PR to production:

| Job | Trigger | Produces / targets |
|---|---|---|
| CI | `pull_request` → `main` | pass/fail check on the PR |
| build-and-scan | push to `main` | Docker image tagged `:<commit-sha>`, scanned with Trivy, pushed to Artifact Registry |
| deploy-staging | after build-and-scan succeeds | image running on the staging VM (`environment: staging`) |
| deploy-production | after staging deploy succeeds | same image tag promoted to the prod/dev VM behind the load balancer (`environment: production`, required reviewer) |

Infra target: GCP project `freshcart-twotier`, region `us-central1`. Production backend is the private VM (`dev-checkout-api`, no public IP) sitting behind the HTTP(S) load balancer at `136.68.219.236`, provisioned in `freshcart-tf`. Staging is the separate e2-small environment from that same repo's stretch goal.

## Authentication: Workload Identity Federation

The pipeline never stores a GCP key. GitHub Actions exchanges its own OIDC token for short-lived GCP credentials via a Workload Identity Pool scoped to this repository, impersonating a dedicated `github-deployer` service account with only `artifactregistry.writer`, `iap.tunnelResourceAccessor`, and `osLogin`. Deploys reach the private VMs through an IAP SSH tunnel rather than a public IP or a static SSH key.

## CI (`.github/workflows/ci.yml`)

Runs on every PR into `main`: checks out `checkout-api`, installs dependencies with npm caching, builds/typechecks. Required check — nothing merges without a green run.

## Build, scan, deploy (`.github/workflows/build-scan-deploy.yml`)

On merge to `main`: builds the Docker image from the Week 4 Dockerfile, tags it with the commit SHA, scans it with Trivy (fails the run on HIGH/CRITICAL findings — see `trivy-checkout-api-before.txt` / `trivy-checkout-api-after.txt` for a real before/after scan comparison), authenticates via WIF, and pushes to Artifact Registry. The staging job then deploys that exact image over an IAP tunnel; the production job reuses the *same* image reference (no rebuild) and only runs after a required reviewer approves the `production` environment.

## Rollback exercise

Rather than a separate rollback workflow, the rollback path is the same pipeline everything else goes through — a revert commit runs the identical CI → build → scan → staging → approved-production sequence as any other change, so recovery gets the same review gate a bad change would have needed in the first place:

1. **Broke it on purpose:** changed the health check route in `checkout-api/src/routes/health.ts` from `/healthz` to `/api/healthz` — a one-line change that silently moves the endpoint the load balancer's health check depends on.
2. **Shipped it:** committed and pushed to `main`, let the pipeline build, scan, and deploy it through staging to production as normal.
3. **Confirmed the break:** the load balancer's health check against `/healthz` started failing once the broken image reached production. <!-- TODO: paste the actual failing health-check output/screenshot here -->
4. **Rolled back:** ran `git revert <commit-hash> --no-edit`, pushed the revert to `main`, and let the same pipeline redeploy the restored route through staging and an approved production deploy.
5. **Confirmed recovery:** `/healthz` returned healthy again once the reverted image was live. <!-- TODO: paste the actual healthy output/screenshot here -->

Screenshots for each step above are in [`/screenshots`](./screenshots).

## What's actually here

Two services and a database (unchanged from Week 4/5):

- **`checkout-api/`** — Express + TypeScript API backed by Postgres. Lists products, handles search, places orders. This is the service the Week 6 pipeline builds, scans, and deploys.
- **`storefront/`** — static site (Vite + TypeScript) that calls the checkout API.
- **Postgres** — runs as a second container on the backend VM, matching the Week 4 `docker-compose.yml` pattern (see `freshcart-tf` for how the VM itself is provisioned).

## Running it locally

```
cd checkout-api
cp .env.example .env      # edit DATABASE_URL to point at your Postgres
npm install
npm run dev
```

Load `db/init.sql` into your database once to create the schema and seed products. Once running: `curl localhost:3000/healthz` → `{"status":"ok"}`.

```
cd storefront
npm install
npm run dev
```

## API reference

| Method | Path                | Does                                                                                                                     |
| ------ | ------------------- | ------------------------------------------------------------------------------------------------------------------------ |
| GET    | `/healthz`          | Checks the API can reach the database.                                                                                   |
| GET    | `/api/products`     | Lists all products. Add `?search=term` to filter by name.                                                                |
| GET    | `/api/products/:id` | One product.                                                                                                             |
| POST   | `/api/orders`       | `{ customerName, customerEmail, items: [{ productId, quantity }] }`. Validates stock, records the order transactionally. |
| GET    | `/api/orders/:id`   | An order and its line items.                                                                                             |

## Related repos and posts

- Infra (Terraform on GCP): [`freshcart-tf`](https://github.com/Mikey064/freshcart-tf) — ["Codifying FreshCart: Building a Reproducible Two-Tier GCP Architecture with Terraform"](https://michaelokpu.medium.com)
- Containerization: [`freshcart-docker-mikey`](https://github.com/Mikey064/freshcart-docker-mikey) — ["From 'Works on My Machine' to Repeatable Builds: Containerizing FreshCart with Docker"](https://michaelokpu.medium.com/from-works-on-my-machine-to-repeatable-builds-containerizing-freshcart-with-docker-1154d8cd92f7)
- This pipeline: blog post coming soon.
