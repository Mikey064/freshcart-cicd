# FreshCart — Containerization architecture notes

This document covers what was built for Week 4 (Docker), the reasoning behind each decision, and the real issues hit along the way. README.md explains how to run the app; this explains why it's built the way it is.

What was added

Neither service shipped with a Dockerfile or compose setup. This project added:

checkout-api/Dockerfile — multi-stage build (Node build stage → minimal runtime stage), non-root user
checkout-api/.dockerignore
storefront/Dockerfile — multi-stage build (Node/Vite build stage → nginx runtime stage), non-root user
storefront/nginx.conf — serves the built static site and reverse-proxies /api/* to checkout-api
storefront/.dockerignore
docker-compose.yml (repo root) — runs storefront, checkout-api, and postgres together on a shared network, with a named volume for Postgres
Architecture overview
Browser → storefront:8080 (nginx) ──/api/*──→ checkout-api:3000 (Express)
                                                        │
                                                        ▼
                                              postgres:5432
                                                        │
                                                        ▼
                                          pg_data (named volume)

All three services sit on one Docker bridge network (freshcart-net). Only checkout-api and postgres talk to each other directly — the browser never reaches Postgres or checkout-api's raw port in the intended flow; everything goes through nginx on 8080, the same way it will once this sits behind a real load balancer in Week 5.

Key decisions and why

Multi-stage builds for both services. Neither service's production image needs its own build tooling. checkout-api's build stage runs tsc to compile TypeScript into dist/; only dist/ and production node_modules cross into the final stage — the TypeScript compiler, tsx, @types/*, and the .ts source itself are left behind. storefront's build stage runs tsc && vite build; only the static output in dist/ crosses into the nginx stage — Node, npm, and all source files are left behind. Smaller final images, smaller attack surface.

Layer ordering: manifests before source. Both Dockerfiles COPY package*.json ./ and RUN npm ci before copying application source. Docker caches each layer by its inputs — as long as package.json/package-lock.json don't change, npm ci is skipped entirely on rebuild, even if every line of application code changed. Reversing this order (copying all source first) would invalidate the dependency-install cache on every single code change.

checkout-api's final stage: node:20-alpine, not distroless. A fully distroless final image was considered — it would have eliminated an entire class of npm-internal CVEs by never installing npm in the runtime image at all. We chose the smaller, more surgical fix instead: RUN npm install -g npm@latest before npm ci --omit=dev in the final stage, which patches the CVEs bundled inside npm itself (see Trivy findings below) while keeping a shell available for debugging. Trade-off: the image still carries npm as unused runtime weight; distroless remains an option to revisit later.

storefront's final stage: nginx:1.29-alpine + apk upgrade. Same reasoning — small officially-maintained base, and its only vulnerabilities were outdated OS packages (openssl, curl, libxml2, etc.) fixable with one apk upgrade --no-cache layer rather than a base image change.

Non-root users in both final stages. checkout-api runs as a dedicated appuser/appgroup (created via addgroup -S / adduser -S). storefront runs as nginx's built-in nginx user, with explicit chown on the html, cache, and conf directories it needs to write to. Neither container can bind to port 80 as a non-root process, which is why both listen on non-privileged ports (3000, 8080) instead — this is a direct consequence of the non-root requirement, not an arbitrary choice.

Named volume only on Postgres. checkout-api and storefront are stateless — rebuilding or replacing either container loses nothing. Postgres is the only service holding data that must survive a container restart, so it's the only one with a volume (pg_data) attached.

init.sql is bind-mounted, not baked into the image. It's mounted read-only into Postgres's /docker-entrypoint-initdb.d/, which the official Postgres image auto-runs — but only the first time it initializes an empty data directory. This was also the source of the first real bug hit (below).

Issues hit while building this, and how they were actually resolved

init.sql silently not running. Early on, checkout-api returned relation "products" does not exist even after loading init.sql manually. Root cause: Postgres only runs files in /docker-entrypoint-initdb.d/ against a fresh, empty data directory — a container that had already initialized once (even from an earlier manual test) skips it silently on every subsequent start, with no error. Fixed by removing the stale container and volume and letting Postgres re-initialize clean with init.sql mounted from the start.

storefront returning 403 Forbidden. docker exec into the container showed /usr/share/nginx/html was completely empty — no index.html. Traced back through several dead ends (permissions, a Dockerfile stage-name typo that briefly broke --target build debugging) before confirming the actual build stage produced real output (vite build succeeded, files existed) and that docker compose up had been serving a stale cached image from before a fix. Resolved with docker compose build --no-cache to force a genuinely clean rebuild.

storefront "connection reset by peer". After the 403 was fixed, requests to localhost:8080 started resetting instead of connecting. docker port revealed the actual mapping was 8080:80 (host:container) while nginx was configured to listen 8080 inside the container — Docker was forwarding to a container port nothing was listening on. Fixed by correcting the compose ports: mapping to 8080:8080, matching EXPOSE 8080 and nginx.conf's listen 8080;.

checkout-api unreachable after a rebuild. curl localhost:3000 failed outright. docker compose ps showed the container healthy and its logs showed it was listening fine — but the port mapping had drifted to 3001:3000 in compose. Not a bug in the app or image, just a stale mapping from earlier editing; resolved by using the actual published port.

Lesson across all three networking issues: when a container is Up and its own logs show no errors, the fault is almost always in the host↔container port mapping, not the application — docker port <container> was the fastest way to confirm this every time.

Vulnerability scanning (Trivy)

checkout-api — before: 8 findings (7 HIGH, 1 CRITICAL), all inside npm's own bundled dependencies (tar, sigstore, picomatch, brace-expansion, ip-address) — none were application dependencies. Fix: upgrade npm itself in the final stage (npm install -g npm@latest) before installing production deps, which pulls in npm's own patched dependency versions.

storefront — before: 11 HIGH findings, all Alpine OS packages (curl, libcrypto3, libssl3, libexpat, libxml2, nghttp2-libs, c-ares) — outdated versions in the base image, all with fixed versions already published upstream. Fix: RUN apk update && apk upgrade --no-cache added as the first instruction in the final stage.

Raw before/after scan output is kept in trivy-checkout-api-before.txt, trivy-checkout-api-after.txt, trivy-storefront-before.txt, and trivy-storefront-after.txt at the repo root.