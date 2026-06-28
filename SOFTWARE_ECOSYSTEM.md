# SOFTWARE_ECOSYSTEM.md

## At a Glance

PJS Collectables is a small online business that sells popculture collectables — trading cards, merch, figures and the like. This ecosystem is the set of cooperating services that runs that business end to end: it ingests the wholesaler catalogue, keeps a single authoritative product dataset, tracks stock and orders, sells to customers through a public web shop, automates order-email ingestion and bookkeeping, and gives staff a mobile app for inventory work. The problem it solves is consistency at low operational cost — every surface (shop, mobile app, accounting) reads from the same trusted data and runs on one small VPS with shared monitoring, logging and a hardened public edge, so a one-person operation can run like a multi-service platform without duplicated logic or manual data shuffling.

What used to be six separate backend services — DBUpdater, InventoryManager, LexwareAPI, PJsShop `back-web`, MailBot and the InfrastructureStack `is-web` — is now **one Django modular monolith**, `pjscollectables-platform`: one image, one pipeline, one Postgres, one RabbitMQ, one Redis. Those services did not change what they do; their code was **moved into apps of the monolith** (`catalog`, `inventory`, `invoicing`, `shop`, `inbox`, `ops`), and the HTTP calls they used to make to each other became **in-process calls across import-linter-enforced module boundaries**. What remains are **three independently deployed projects**: the backend platform, the storefront SPA, and the internal Android app.

| Project | Role | Tech | Status |
| --- | --- | --- | --- |
| [pjscollectables-platform](#pjscollectables-platform-context) | Backend modular monolith: catalogue SSOT, stock/FIFO, storefront API + checkout, invoicing, mail ingest, B2B, ops/observability | Django · DRF · Celery · PostgreSQL · RabbitMQ · Redis · Docker · nginx | Active — Production |
| [PJsShopFront](#pjsshopfront-context) | Customer-facing storefront UI (browser entrypoint) | Vue 3 · Vite · Tailwind | Active — Production |
| [IMApp](#imapp-context) | Internal Android client for the stock (inventory) API | Kotlin · Jetpack Compose · OkHttp | Active — Internal sideload |

```mermaid
flowchart LR
    Customers(("Customers")) -->|HTTPS 443| CF["Cloudflare edge<br/>WAF · TLS · AOP mTLS"]
    Operators(("Staff / Operators")) -.->|WireGuard VPN| NGINX
    IMAPP["IMApp (Android)"] -->|X-App-Secret + Token| CF

    CF -->|pjscollectables.com| NGINX
    CF -->|im-api.pjscollectables.com/PREFIX| NGINX

    subgraph VPS["Single VPS — Docker (pjs-net + pjs-monitoring)"]
        NGINX["nginx 80/443<br/>public edge + maintenance door"]
        FRONT["PJsShopFront (Vue)<br/>pjs-shop-front:5173"]

        subgraph MONO["pjscollectables-platform — one image (web:8000 + worker + cron)"]
            SHOP["shop<br/>/api/ storefront"]
            CAT["catalog<br/>catalogue SSOT"]
            INV["inventory<br/>/api/inventory/ (mobile)"]
            INVOICE["invoicing"]
            INBOX["inbox<br/>(mail ingest)"]
            B2B["b2b<br/>/api/b2b/"]
            OPS["ops<br/>/health /status"]
            CORE["core<br/>(shared base)"]
        end

        DATA[("postgres · rabbit · redis")]
        OBS["Observability<br/>prometheus · loki · grafana · cadvisor"]
    end

    NGINX -->|/| FRONT
    NGINX -->|/api/ same-origin| SHOP
    NGINX -->|allow-listed /api/inventory/| INV
    FRONT -->|REST + session/CSRF| SHOP

    SHOP -.->|in-process services| CAT
    SHOP -.->|in-process services| INV
    SHOP -.->|in-process services| INVOICE
    INV -.->|in-process services| CAT
    INBOX -.->|in-process services| INV
    B2B -.->|in-process services| CAT
    MONO --> DATA

    SHOP -->|checkout capture| PayPal(("PayPal"))
    INVOICE -->|invoices/vouchers| Lexoffice(("Lexoffice"))
    INVOICE -->|EU VAT rates| Vatsense(("VATSense"))
    Mailbox(("GMX mailbox")) <-->|IMAP in · SMTP out| MONO
    MONO -.->|product media| R2(("Cloudflare R2"))
    OBS -.->|scrape /metrics · tail logs| MONO
```

### Contents
- [Infrastructure & Distribution](#infrastructure--distribution)
- [pjscollectables-platform](#pjscollectables-platform-context)
  - [catalog (former DBUpdater)](#catalog-former-dbupdater)
  - [inventory (former InventoryManager)](#inventory-former-inventorymanager)
  - [inbox (former MailBot)](#inbox-former-mailbot)
  - [shop (former PJsShop)](#shop-former-pjsshop)
  - [invoicing (former LexwareAPI)](#invoicing-former-lexwareapi)
  - [b2b](#b2b)
  - [ops (former InfrastructureStack is-web)](#ops-former-infrastructurestack-is-web)
  - [core](#core)
- [PJsShopFront](#pjsshopfront-context)
- [IMApp](#imapp-context)
- [Development Regulation Standards](#development-regulation-standards)

---

## Infrastructure & Distribution

This section describes the global, ecosystem-wide concerns that are not specific to any one project: where everything runs, how the public edge is structured, how this document is distributed, and which quality/security standards apply across all repositories.

### Hosting model

The entire ecosystem runs on a **single VPS** under **Docker Compose**. Each independently-deployed project ships its own Compose stack and joins the shared Docker bridge networks the **monolith** defines, rather than everything living in one stack. The consolidation collapsed the old per-service bridges to just two, segmented by concern:

- `pjs-net` — the main application bridge. The monolith's `web`, `worker`, `cron`, `postgres`, `rabbit`, `redis` and `nginx` join it, and the storefront SPA stack ([PJsShopFront](#pjsshopfront-context)) joins it as an *external* network so nginx can reach `pjs-shop-front:5173` by container DNS. All internal service-to-service traffic and the datastores live here.
- `pjs-monitoring` — the **observability spine**: Prometheus, Alertmanager, Loki, Promtail, Grafana and cAdvisor attach here, and the `web` and `nginx` containers join it too so their `/metrics` can be scraped and their logs collected without exposing public ports.

> **What changed:** the former per-service bridges (`is-net`, `dbu-net`, `im-net`, `pjs-backend-net`) are gone — with one application there are no cross-service hops left to segment. Container isolation is still the default: the datastores (`postgres`/`rabbit`/`redis`) are reachable only on `pjs-net` and are never published to the host, so they stay invisible outside the stack.

All services run on the VPS; there is **no Raspberry-Pi-hosted tier**. (See the monolith's `DEPLOY.md` for the authoritative VPS layout, deploy order and the one-command deploy.)

### Nginx & the public edge

The monolith's **nginx** is the only component that publishes ports `80`/`443` to the internet; everything else is reachable only over the `pjs-net` bridge or the admin VPN. The edge sits behind **Cloudflare** (Full-strict TLS, WAF, rate-limiting, origin-IP hiding, Authenticated Origin Pulls/mTLS so only Cloudflare can complete the origin handshake). TLS certificates are issued and auto-renewed **in-stack** by a `certbot` sidecar (ACME DNS-01 against the Cloudflare DNS API, into a shared `pjs-letsencrypt` volume nginx reads) — no manual host-side cert step. nginx exposes two public surfaces and keeps everything else VPN-only:

- **Storefront** — `pjscollectables.com`/`www` is served to customers: `/` proxies to the Vue container `pjs-shop-front:5173` (the [PJsShopFront](#pjsshopfront-context) stack) and `/api/` is same-origin to the monolith `web:8000`. A **maintenance door** marker (the `shop_open` file on the shared `pjs-control` volume, mounted read-only into nginx and toggled from the ops admin control panel) lets an operator serve a maintenance page for the whole shop by "closing the door"; it is also the fail-safe default, so a missing or down frontend degrades to the maintenance page instead of a 502.
- **Mobile Stock API** — `im-api.pjscollectables.com/<PREFIX>/api/inventory/…` exposes a deliberately-obscured, hardened subset of the monolith's `inventory` module for the [IMApp](#imapp-context) Android client (secret URL prefix + `X-App-Secret` header + endpoint allow-list + per-client rate limits + Cloudflare AOP mTLS; the edge rewrites the prefix away and proxies to `web:8000`). See `infra/nginx/MOBILE_API.md` in the monolith.
- **Protected (VPN-only)** — the unified Django admin (`admin.pjscollectables.com`), Grafana (`grafana.pjscollectables.com`) and the internal API namespaces (`/api/catalog/`, `/api/invoicing/`, `/api/b2b/`, the non-mobile `/api/inventory/` routes, plus `/admin`, `/metrics`, `/analysis`, `/status`) are **not** public — they return `404` on the public host and are reachable only over the WireGuard VPN (`10.8.0.0/24`). The five former per-service admin vhosts collapse to this one Django admin.

### Documentation distribution (this file)

`SOFTWARE_ECOSYSTEM.md` is the master technical reference, kept in sync across every repo by GitHub Actions with a public **single-source-of-truth (SSOT) repository** as the hub:

- The reusable workflow `reusable-sync-docs.yml` — now held centrally in the **monolith** (`MKRO-JWL/pjscollectables-platform`), alongside the ecosystem's other shared reusable workflows — is the one place the sync logic lives. On a push to a repo's `main`, its **Push to SSOT** step copies that repo's `SOFTWARE_ECOSYSTEM.md` into the SSOT repo **`MKRO-JWL/pjs-ecosystem-docs`** and commits it.
- On a `repository_dispatch` of type `sync_requested`, the **Fetch from SSOT** step pulls the canonical file from the SSOT raw URL and commits it into the receiving child repo.
- Each repo wires a thin caller (`sync_ecos_doc.yml`) that `uses:` the reusable workflow: the monolith references it locally (`./.github/workflows/reusable-sync-docs.yml`), and child repos reference it at `MKRO-JWL/pjscollectables-platform/.github/workflows/reusable-sync-docs.yml@main`.

In practice: edit this file once in the monolith → merge to `main` → the push propagates to the SSOT → a `sync_requested` dispatch fans the update out to every connected child repo, keeping all copies identical without manual editing.

> **Operational notes:** the reusable workflow must be committed on the monolith's `main` for children's `uses: …@main` references to resolve (and, if the monolith repo is private, its **Settings → Actions → Access** must allow the other `MKRO-JWL` repos). Because every repo's push-to-`main` also pushes *its* copy up, the SSOT is last-writer-wins; treat the monolith as the single canonical place this file is edited to avoid divergence. **InfrastructureStack — the former holder of these workflows — is retired; do not point any caller back at it.**

### Quality & security standards

Every repository is built and audited against shared standards rather than ad-hoc rules:

- **ISO/IEC 25010** — the product-quality model used as the scoring rubric for audits. The full rubric is kept embedded at the end of this document, under [Development Regulation Standards](#development-regulation-standards).
- **OWASP** — web-application security guidance (input validation, authn/session handling, transport security, secret management) informs the security posture across the Django app and the public edge.
- **Architectural boundaries** — module isolation in the monolith is enforced in CI by **import-linter** (`.importlinter`, the `contracts` gate): `core` may not depend on any domain app, and a domain app's internals are private (cross-app access only through `<app>.services`). This keeps the consolidated codebase from sliding back into a tangle of cross-imports.

These standards are referenced here, not duplicated per project; each project section's *Operations & Lifecycle* segment documents the concrete controls it implements.

---

# pjscollectables-platform Context

## Summary

`pjscollectables-platform` is the **backend modular monolith** for PJS Collectables: a single Django project, built into a single image and run as a small set of containers (`web`, `worker`, `cron` + Postgres/RabbitMQ/Redis + nginx + the observability stack). It consolidates the five former backend microservices — DBUpdater, InventoryManager, LexwareAPI, PJsShop `back-web` and the InfrastructureStack `is-web` — into one project with one Postgres, one RabbitMQ and one Redis, plus a new `inbox` mail-ingress app (the old MailBot) and a `b2b` business storefront. Each former service is now a Django **app** with its domain logic intact; cross-domain access goes only through each app's `services.py` seam, enforced by import-linter. It owns every piece of business data in the ecosystem and exposes exactly three external HTTP surfaces — the storefront API, the hardened mobile stock API, and the B2B API — behind a Cloudflare-fronted nginx edge. **Status: Active — Production.**

```mermaid
flowchart TB
    subgraph edge["Public edge (nginx, Cloudflare-fronted)"]
        pub["pjscollectables.com<br/>/ → SPA · /api/ → shop"]
        mob["im-api.pjscollectables.com<br/>/PREFIX/api/inventory/… (allow-list)"]
        adm["admin.pjscollectables.com<br/>(VPN only)"]
    end

    subgraph web["web (pjs-web:8000) — gunicorn, one Django app"]
        shop["shop · /api/…"]
        inventory["inventory · /api/inventory/…"]
        b2b["b2b · /api/b2b/…"]
        catalog["catalog · /api/catalog/ (internal)"]
        invoicing["invoicing · /api/invoicing/ (internal)"]
        ops["ops · /health /status /metrics + admin panel"]
        core["core (shared base)"]
        inbox["inbox (handlers)"]
    end

    worker["worker (Celery)"]
    cron["cron (supercronic)"]
    pg[("postgres")]
    rabbit[("rabbit (broker)")]
    redis[("redis (cache + results)")]

    pub --> shop
    mob --> inventory
    adm --> ops

    shop --> catalog
    shop --> inventory
    shop --> invoicing
    inventory --> catalog
    b2b --> catalog
    inbox --> inventory

    web --> pg
    web --> redis
    web -->|enqueue| rabbit
    rabbit --> worker
    worker --> pg
    cron -->|poll_inbox, exports| web

    shop --> PayPal(("PayPal"))
    invoicing --> Lexoffice(("Lexoffice"))
    invoicing --> Vatsense(("VATSense"))
    inbox <-->|IMAP| GMX(("GMX mailbox"))
    invoicing -->|SMTP| GMX
    web -.->|media| R2(("Cloudflare R2"))
```

## Purpose

The platform is the **single system of record** for the whole business: it is where product truth, stock, orders, customers, payments, invoices and ingested mail all live. Everything else in the ecosystem is a thin client of it — the [PJsShopFront](#pjsshopfront-context) SPA renders its storefront API, and [IMApp](#imapp-context) drives its stock API — so the platform is where consistency is enforced once and read everywhere.

Consolidating the former services into one deployable was the point of the migration: the integrations that used to be fragile network hops (DBUpdater ↔ InventoryManager ↔ PJsShop ↔ LexwareAPI ↔ MailBot) are now **in-process function calls** across module boundaries, removing inter-service auth, retries, partial-failure handling and version skew for those paths. What the platform still depends on are genuinely external systems: **PayPal** (payment capture), **Lexoffice** (bookkeeping), **VATSense** (EU VAT rates), the **GMX mailbox** (IMAP order-email ingress + SMTP transactional mail), **Cloudflare R2** (product media), and the **wholesaler** (catalogue spreadsheets + order-confirmation emails). Data ownership is therefore total and inward: business data flows out of the platform to its clients, and only operator inputs and third-party API results flow in.

**Boundary & non-goals.** The platform is not the UI — the browser storefront is [PJsShopFront](#pjsshopfront-context) and the mobile stock UI is [IMApp](#imapp-context); it serves them data and holds none of their client state. It is not a service mesh: the modules are isolated by import-linter contracts, not by network segmentation, so "integration between modules" means a typed Python call into `<app>.services`, never an HTTP round-trip. The only first-class internal seams other modules may cross are the `services.py` packages; a module's `models`, `views`, `serializers`, `admin`, `tasks` and `urls` are private.

### Module map

| App (module) | Domain | Migrated from | External HTTP surface |
| --- | --- | --- | --- |
| `core` | Shared models/utils/config, tracing, image helpers | — (new shared base) | none (internal only) |
| `catalog` | Catalogue SSOT (categories, products, licenses), wholesaler import | DBUpdater | `/api/catalog/` — internal/admin only |
| `inventory` | Stock, FIFO consumption, sales, mobile stock API, analysis | InventoryManager | `/api/inventory/…` (mobile via edge) + `/analysis/` (admin) |
| `invoicing` | Lexoffice invoices, VATSense tax, payout matching, PDF upload | LexwareAPI | `/api/invoicing/` — internal/admin only |
| `shop` | Storefront API, cart, orders, PayPal checkout, accounts | PJsShop `back-web` | `/api/…` (public storefront) |
| `inbox` | Mail ingress: IMAP poll, sender-rule routing, handlers | MailBot | none (producer; admin-managed) |
| `b2b` | Business-only storefront (gated on verified business accounts) | — (new) | `/api/b2b/…` |
| `ops` | Liveness/status, `/metrics`, maintenance-door control panel | InfrastructureStack `is-web` | `/health/`, `/status/`, `/metrics`, admin panel |
| `config` | Django project package (settings, urls, wsgi/asgi, celery) | — | — |

## Repository layout

- `config/` – Django project package: `settings.py`, `urls.py` (root URLconf that mounts every app), `wsgi.py`/`asgi.py`, `celery.py`.
- `core/` – shared base: cross-cutting `models`, `utils`, `services`, `middleware`, `trace.py` (correlation IDs), `images.py`. **Must not import any domain app** (import-linter rule #1).
- `catalog/` – catalogue SSOT (former DBUpdater): `models`, `serializers`, `views`, `signals`, `tasks`, `urls`, and `services/` (`products`, `product_query`, `catalog_update`, `category_utils`, `gpsr_normalizer`, `exports`, `close_out`).
- `inventory/` – stock ledger (former InventoryManager): `models`, `views`, `serializers`, `auth.py`, `urls`, `tasks`, `dbu_sync.py`, `sold_order_parser.py`, `analytics.py`, `analysis_views.py`/`analysis_urls.py`, and `services.py` (the cross-app seam: `receive_order`, `record_sale`, `receive_files`).
- `invoicing/` – bookkeeping bridge (former LexwareAPI): `models` (Job/JobLog, TaxRate, pending refs, withdrawal buckets), `views`, `urls`, and `services/` (`lexware`, `jobs`, `orders`, `upload`, `mailer`).
- `shop/` – storefront backend (former PJsShop): `models`, `views`, `serializers`, `urls`, `auth_backends`, `forms`, `tasks`, `signals`, and `services/` (`paypal`, `im_integration`, `lexware_integration`, `mail`).
- `inbox/` – mail ingress (former MailBot): `models` (`SenderRule`, `IngestedMessage`), `tasks`, and `services/` (`client` IMAP, `ingest`, `handlers` registry, `order_ingest` handler).
- `b2b/` – business storefront: `models` (`BusinessProfile`), `views`, `serializers`, `permissions`, `urls`.
- `ops/` – platform ops (former InfrastructureStack `is-web`): `status.py` (`/health`, `/status`), `maintenance.py`, `views`, `services`, `tasks`, `signals`.
- `infra/` – deployment assets (not Python): `nginx/` (dev + `nginx.prod.conf.template`, `cloudflare/`, `MOBILE_API.md`), `prometheus/`, `alertmanager/`, `loki`/`promtail/`, `grafana/`, `certbot/`, `cron/crontab`.
- `ops/` (scripts), `docker-compose.yml` (+ `.override.yml` dev, `.prod.yml` prod), `Dockerfile`, `entrypoint.sh`, `gunicorn.conf.py`, `.env.example`, `.importlinter`, `DEPLOY.md`, `requirements*.txt`.

## Integration with other Projects

The platform's external contract surface is small and deliberate. Internally, modules integrate **in-process** through `services.py` seams (documented per module under [Core services](#core-services)); externally, the platform exposes three application APIs plus the admin and ops endpoints, and consumes only third-party systems and operator inputs.

### Exposes

All app APIs are served by the single `web` container (`web:8000`, Docker DNS) and routed by nginx. Which surface is reachable from where is enforced at the edge: the public shop host serves only the storefront; the mobile host serves only the allow-listed stock endpoints; the admin and internal namespaces are VPN-only.

**1. Storefront REST API (`/api/…`)** — consumed by [PJsShopFront](#pjsshopfront-context). DRF with cookie session + CSRF; the SPA calls `/api/session/` once to obtain the CSRF/session cookies, then sends `X-CSRFToken` on mutating requests.

| Interface | Location | Auth | Notes | Errors |
| --- | --- | --- | --- | --- |
| Session/CSRF bootstrap | `GET/POST /api/session/` | None | Sets CSRF + session for browser clients. | 403 (CSRF) |
| Products | `GET/POST /api/products/`, `GET /api/products/facets/`, `GET/PUT/DELETE /api/products/{id}/` | Read: none; Write: session | Paginated list + `category__name` filter; faceted filters. | 400, 401/403, 404 |
| Categories | `GET /api/categories/`, `GET /api/categories/nav/` | None | Flat list (filter by `parent`/`slug`/`name`/`is_active`) + nested nav tree. | 404 |
| Cart | `GET/POST /api/cart/`, `POST /api/cart/add/`, `POST /api/cart/remove/`, `POST /api/cart/update/` | Session + CSRF | Cart with items + server-derived totals. | 400, 401/403 |
| Checkout | `GET /api/checkout/initiate/` | Session | Address defaults + cart state. | 401/403, 5xx |
| Orders | `POST /api/orders/place/` (manual/bank transfer), `GET/POST /api/orders/`, `GET /api/orders/{id}/` | Session (+ CSRF on place) | Order list/detail/placement. `POST /api/orders/create/` is **disabled (HTTP 410)**. | 400, 401/403, 410 |
| Accounts | `POST /api/login/`, `/api/logout/`, `/api/register/`, `/api/register-after-order/`, `/api/profile/delete/`, `/api/password-reset/`, `/api/password-reset/confirm/` | Session + CSRF | Credentials/profile actions; reset links use `FRONTEND_BASE_URL`. | 400, 401 |
| Profile | `GET/PATCH /api/profile/`, `GET /api/profile/wishlist/`, `GET /api/profile/orders/` | Session | Profile, addresses, wishlist, order history. | 401/403 |
| PayPal | `GET /api/paypal/client-id/`, `POST /api/paypal/create-order/`, `POST /api/paypal/capture-order/`, `POST /api/paypal/webhook/` | Session+CSRF (create/capture); signature-verified (webhook) | Server-authoritative amounts; capture re-verifies against the server cart; webhook is the idempotent backstop. | 400, 402, 502 |

**2. Mobile stock API (`/api/inventory/…`)** — consumed by [IMApp](#imapp-context) through the hardened `im-api.pjscollectables.com/<PREFIX>/…` gateway. Token auth: `POST /api/inventory/auth/token/` (username/password → `{token}`), sent as `Authorization: Token <token>`.

| Method | Path (under `/api/inventory/`) | Purpose | Auth | In mobile allow-list |
| --- | --- | --- | --- | --- |
| POST | `auth/token/` | Login → DRF token (throttled + django-axes). | username/password | yes |
| GET | `inventory/` | List visible inventory + summed total value. | Admin token | yes |
| POST | `add-stock/` | Increase quantity by `amount` (optional `order_id`). | Admin token | yes |
| POST | `reduce-stock/` | Decrease quantity by `amount`. | Admin token | yes |
| DELETE | `delete-item/` | Soft-delete by `item_no`. | Admin token | yes |
| POST | `query-dbu-item/` | Search the catalogue for products by title. | Admin token | yes |
| POST | `confirm-item/` | Import catalogue product details by `product_no`. | Admin token | yes |
| POST | `receive/` | Create order + items from posted data. | Authenticated | no (internal) |
| POST | `sales/` | Register sold lines for FIFO consumption + margin. | Authenticated | no (internal) |
| POST | `mark-sold/` | Mark item(s) sold. | Authenticated | no (internal) |

At the edge, a request reaches these only if it satisfies **all** of: the secret URL prefix `${IM_API_PREFIX}`, the `X-App-Secret: ${IM_APP_SECRET}` header, an allow-listed endpoint, the per-real-client rate limits (login `5/min`, ops `2/s` burst 10), and Cloudflare Authenticated Origin Pulls (mTLS) — else nginx returns `444`/`429`. Inside the app, a valid per-user DRF token + django-axes lockout apply.

**3. B2B API (`/api/b2b/…`)** — browser/SPA B2B customers. Session/Basic auth (not the platform-default token auth).

| Method | Path | Purpose | Auth |
| --- | --- | --- | --- |
| POST | `/api/b2b/register/` | A logged-in customer applies for B2B access (created PENDING approval). | Authenticated |
| GET | `/api/b2b/profile/` | The caller's own `BusinessProfile` + verification status. | Authenticated |
| GET | `/api/b2b/catalogue/` | B2B-only catalogue (id, blackfire_id, name, buy/sell price). | Verified business |

**4. Admin & ops** — `GET /admin/` (unified Django admin; the former im/dbu/is/back admin sites collapse to one) plus the admin utility routes `/admin/sync-dbu/` and `/admin/analytics/`, and the inventory `/analysis/` dashboard — all **VPN-only**. `GET /health/` (liveness), `GET /status/` (aggregated service health, see [ops](#ops-former-infrastructurestack-is-web)), `GET /metrics` (django-prometheus, whole app). Internal app namespaces `/api/catalog/` and `/api/invoicing/` exist for admin/diagnostic use and return **404 on the public host**.

**Service discovery / auth summary.** Internal Docker DNS `web:8000` on `pjs-net`; public hosts terminate TLS at nginx. Storefront = session+CSRF; mobile = pre-shared header + DRF token; B2B = session/basic + verified-business gate; admin/ops = VPN network placement.

### Consumes

The platform has **no in-ecosystem upstream** — PJsShopFront and IMApp consume it, not the other way around. It consumes only external systems and operator inputs:

- **PayPal** — OAuth + order create/capture + webhook (`api-m.sandbox.paypal.com` or live per `PAYPAL_ENV`). On outage, checkout returns `502/503` and the cart is preserved.
- **Lexoffice** (`api.lexware.io`) — invoice/contact/voucher/file-upload operations (invoicing). Empty key disables the feature (fail-soft); on outage, affected jobs fail or report per-file failures.
- **VATSense** (`api.vatsense.com`) — EU VAT-rate refresh (invoicing). Non-strict mode falls back to a default rate (`19`); strict mode fails the job.
- **GMX mailbox** — **IMAP ingress** (inbox polls `imap.gmx.net:993` for order-confirmation emails) and **SMTP egress** (invoicing/shop send invoice PDFs, password-reset and supplier-reorder mail). With no mail password the console email backend is used (safe in dev/CI); IMAP unreachable simply means the next poll finds nothing.
- **Cloudflare R2** — product/media object storage via django-storages when all four `R2_*` vars are set; otherwise local `FileSystemStorage` (valid for dev/CI/tests).
- **Wholesaler (Blackfire)** — catalogue/GPSR spreadsheets ingested by `catalog` management commands, and order-confirmation emails ingested by `inbox`. Operator/scheduled inputs; if absent, imports simply don't run.
- **Cloudflare DNS / edge** — the certbot sidecar issues+renews the apex+wildcard TLS cert via the ACME DNS-01 challenge against the Cloudflare DNS API; the public edge sits behind Cloudflare WAF + Authenticated Origin Pulls.

## Core services

This segment is the heart of the platform: it documents the modular-monolith contract and then each module — what it owns, the in-process seam other modules call, and the data-flow direction. Every former microservice integration is now one of these seams.

### Architecture: the modular-monolith contract & service seams

The platform is one Django project whose apps are kept honest by **import-linter** (`.importlinter`, run as the `contracts` CI gate via `lint-imports`). Two rules hold:

1. **`core` is the shared base** and must not depend on any domain app (directly or transitively).
2. **Each domain app's internals are private** — `models`, `views`, `serializers`, `admin`, `tasks`, `urls` may not be imported across apps. Cross-domain access goes **only** through `<app>.services` (deliberately absent from the forbidden list). The contracts set `allow_indirect_imports = True`, so a service legitimately importing its own app's models is fine; only a *direct* import of another app's internals is a violation.

This is what replaced the old network integrations. The canonical seams are:

- `catalog.services` — `query_products(...)`, catalogue read/update helpers. Called in-process by `shop`, `inventory` and `b2b` instead of HTTP to the old DBUpdater `/api/products/`.
- `inventory.services` — `receive_order(...)`, `record_sale(...)`, `receive_files(...)`. Called by `inbox` (parsed order emails), `shop` (captured sales), and `invoicing` (generated invoices / sold-order CSVs) instead of HTTP POSTs to the old InventoryManager `/api/receive/`, `/api/sales/`, `/api/receive/`.
- `shop.services` — `paypal`, `im_integration` (→ `inventory.services`), `lexware_integration` (→ `invoicing.services`), `mail`.
- `invoicing.services` — `lexware`, `jobs`, `orders`, `upload`, `mailer`.
- `inbox.services` — `client` (IMAP), `ingest`, `handlers` (registry), `order_ingest`.

Because these are typed function calls in one process, the failure modes that dominated the microservice era (inter-service auth, retry/backoff, partial delivery, version skew) are gone for those paths; what remains is ordinary exception handling and DB transactions.

### catalog (former DBUpdater)

The catalogue **single source of truth**: normalized categories, products and licenses imported from the wholesaler and stored indefinitely. It **owns** the canonical product/category/license data and the import pipeline (`catalog/services/catalog_update.py`, `category_utils`, `gpsr_normalizer`, `exports`, `close_out`; management commands for import/scrape/cleanup; `catalog.tasks` Celery jobs). Other modules **read** it in-process via `catalog.services.query_products(...)` — the shop renders products, inventory enriches stock with product metadata, b2b serves a business catalogue. Data flows **out** of catalog to every other module. Its `/api/catalog/` HTTP API still exists but, since all in-ecosystem consumers now call the service seam, it is internal/admin-facing only. **Non-goals:** catalog does not track stock or orders (that is inventory), does not sell (shop), and does not parse mail (inbox).

### inventory (former InventoryManager)

The **operational stock ledger**: the system of record for on-hand stock, FIFO lots, sold lines and margins, plus the stock-taking/analysis surface. It **owns** inventory state and the business mutations on it (add/reduce/delete/confirm), and exposes the mobile stock API consumed by [IMApp](#imapp-context). Its `services.py` is the only cross-app entry point and the in-process replacement for the old InventoryManager HTTP service:

- `receive_order(order_number, items, ...)` — create/reuse a supplier purchase `Order` and its `InventoryItem` lots from structured order data. **Idempotent on `order_number`** (re-ingest reuses the order and skips already-recorded lines), seeds `quantity_remaining` for FIFO, and runs in one transaction. Called by `inbox.order_ingest` (parsed wholesaler emails) and exposed as `POST /api/inventory/receive/`.
- `record_sale(lines, channel, order_id, ...)` — register outbound sale line(s) and FIFO-consume stock, matching each line to a product by cardmarket/blackfire id (else fuzzy title). Called in-process by `shop` on payment capture and exposed as `POST /api/inventory/sales/`.
- `receive_files(files)` — validate, spool and enqueue invoice PDFs / sold-order CSVs (10 MB/file cap) for Celery parsing. Called by `invoicing`.

inventory enriches items from the catalogue via `catalog.services` (in-process; the old `dbu_sync`/`utils` DBU-client HTTP path is retired) and serves the `/analysis/` dashboard (upcoming-payments charts from `ItemDetails`). **Non-goals:** it is not the catalogue authority (catalog) and not a storefront (shop/b2b); it serves IMApp over the hardened API, it does not render mobile UI. Treat write endpoints as **non-idempotent** except `receive_order` (idempotent by order).

### inbox (former MailBot)

The **mail-ingress** app — MailBot's relay, reincarnated in-process. A scheduled poll (`manage.py poll_inbox`, run by the `cron`/supercronic container; also `inbox.tasks.poll_inbox_task` for ad-hoc async triggering) fetches new mail from the GMX mailbox over IMAP (`inbox.services.client`), deduplicates and records it, then routes it:

- **`IngestedMessage`** — one row per fetched email; its unique `message_id` is the **idempotency key** (a re-polled email is never processed twice) and the row is the durable audit trail (`status`, `matched_handler`, `result`/`error`).
- **`SenderRule`** — admin-editable "filter by sender": the first active rule (lowest `priority`) whose pattern matches the From address selects a registered **handler**. Operators add/disable senders without code changes.
- **Handlers** — functions registered via `inbox.services.register_handler`. The built-in **`order_ingest`** parses a wholesaler order-confirmation email (BeautifulSoup over the line-item table, EN/DE column synonyms, order-number extraction) into `{order_number, items[]}` and forwards it **in-process** to `inventory.services.receive_order` — replacing MailBot's HTTP POST to InventoryManager. If the email carries no order number, a stable synthetic one is derived from the `message_id` so re-ingest stays idempotent.

inbox **owns** the ingest dedup/audit state and the sender-routing rules; it is a producer with **no inbound API**. Data flows from the inbox **into** inventory. **Parser note:** `order_ingest`'s extraction is built to the documented MailBot payload shape and must be calibrated against a real wholesaler sample (synonym lists + order-number patterns); unit tests pin the contract with a synthetic fixture.

### shop (former PJsShop)

The **storefront backend** and the only module that authenticates customers, persists carts and records orders for fulfilment. It **owns** the order lifecycle, cart state and the customer commerce profile (addresses, wishlist, language). It serves the public `/api/…` surface that [PJsShopFront](#pjsshopfront-context) renders. Core services:

- **Catalog/category/cart/order/profile APIs** — product listing/detail/facets over the catalogue (read via `catalog.services`), server-derived cart totals (`Cart`/`CartItem`), order orchestration (`Order`/`OrderItem`, statuses Pending/Completed/Canceled/Shipped), and `UserProfile`.
- **PayPal integration (`shop/services/paypal.py`)** — obtains an OAuth token server-side, creates/captures orders, serves the client-id to the SDK via `/api/paypal/client-id/`. **Capture is authoritative**: `POST /api/paypal/capture-order/ {orderId, terms_accepted}` captures, **re-verifies the amount against the server cart**, persists orders from PayPal's own response, and returns `{orders}` — amounts are never taken from the client. A signature-verified webhook (`PAYMENT.CAPTURE.COMPLETED`/`DENIED`) is the idempotent backstop. Legacy `POST /api/orders/create/` is **disabled (HTTP 410)**.
- **Sale push (`shop/services/im_integration.py`)** — on capture, pushes sold lines to `inventory.services.record_sale(...)` **in-process** (the old best-effort HTTP POST to `im-web:8000/api/sales/` is gone). FIFO consumption + margin happen in the same process; a failure there is logged and never breaks checkout.
- **Invoicing hook (`shop/services/lexware_integration.py`)** and **mail (`shop/services/mail.py`)** — invoice generation and transactional email (password reset, etc.) via the invoicing/mailer seams.

**Non-goals:** shop is not the catalogue authority (catalog) and not the inventory ledger (it notifies inventory of sales but does not track on-hand stock). Product images/media are served from `pjs-media` (R2 in prod).

### invoicing (former LexwareAPI)

The **bookkeeping automation bridge**: it turns sold-order/transaction CSV inputs into Lexoffice invoices, assigns payouts back to invoice references, refreshes EU tax rates from VATSense, and handles invoice-document upload. It **owns** `Job`/`JobLog` run history, the `TaxRate` cache, pending invoice/order matching entities, and withdrawal-match buckets. Core services (`invoicing/services/`):

- **`jobs`** — the control plane: `run_job(...)` persists a `Job`, runs the workflow, captures stdout/stderr into ordered `JobLog` rows, and writes a structured result (trigger-and-poll via `POST /api/invoicing/jobs/...` then `GET /api/invoicing/jobs/{id}`).
- **`lexware`** — invoice creation (sold-orders CSV → Lexoffice invoices, with country/tax/contact branching and OSS behavior), invoice assignment / payout matching (transaction-summary CSV ↔ Lexoffice vouchers), and tax refresh (`fetchAllTaxRates` → `TaxRate`, strict vs fail-soft default `19`).
- **`upload`** — local validation (type/size/duplicates/strict-PDF) + Lexoffice upload execution (per-file success/failure normalization).
- **`orders`**, **`mailer`** — sold-order handoff (forwards generated invoices / sold-order CSVs into `inventory.services.receive_files` in-process) and SMTP delivery of invoice/proforma PDFs.

invoicing **consumes** upstream CSV exports and the Lexoffice/VATSense APIs; its `/api/invoicing/` HTTP routes are internal/admin-facing (the shop calls it in-process). **Non-goals:** it is not a commerce backend (shop) or inventory ledger (inventory) — it consumes their data and turns it into bookkeeping outcomes. Job triggers are **non-idempotent**; check `GET …/jobs/{id}` before re-running.

### b2b

A new, gated **business storefront** under its own `/api/b2b/` namespace, kept separate from the public shop. A logged-in customer applies via `POST /api/b2b/register/`, which creates a `BusinessProfile` in a **PENDING** state for admin approval; `GET /api/b2b/profile/` returns their profile + verification status; `GET /api/b2b/catalogue/` is gated by the `IsVerifiedBusiness` permission and returns a B2B catalogue read **in-process** via `catalog.services.query_products` (the hook point for B2B-specific pricing/visibility later). It uses session/basic auth (not the platform-default token auth). It **owns** `BusinessProfile` (the business-account + verification state) and depends only on the catalogue seam.

### ops (former InfrastructureStack is-web)

Platform operations, migrated from the InfrastructureStack `is-web`. It owns:

- **Liveness** `GET /health/` → `{"status":"ok"}` (the container HEALTHCHECK and the nginx maintenance-door logic probe this).
- **Aggregated status** `GET /status/` (`ops/status.py`) — concurrently probes the dependent services (`web` self, `promtail`, `grafana`, `prometheus`) over Docker DNS with a 3 s per-probe timeout, caches for 60 s, and returns `{overall, services{...}}` where `overall ∈ {ok, degraded, down}`.
- **Maintenance door** (`ops/maintenance.py`, admin control panel) — toggling writes/removes the `shop_open` marker on the shared `pjs-control` volume (mounted read-only into nginx). With the marker absent (fail-safe default) nginx serves the maintenance page for the storefront; the check is per-request, so toggling is instant with no reload.

ops **owns** the control marker and the health/status surface; it holds no business data and implements no domain logic.

### core

The shared base every domain app builds on: cross-cutting models/utils, the `services` shared helpers, request `middleware`, correlation-ID tracing (`core/trace.py`), and image handling (`core/images.py`). Per the import-linter contract, **core may not import any domain app** — dependencies point inward to core, never outward.

## Operations & Lifecycle

### Runtime behavior

- **Startup dependencies & ordering.** The application containers `web`, `worker` and `cron` all `depends_on` `postgres`, `redis` **and** `rabbit` with `condition: service_healthy`. `web` runs the entrypoint bootstrap (`migrate` → `collectstatic` → gunicorn via `gunicorn.conf.py`, `GUNICORN_WORKERS` workers); `worker` runs `celery -A config worker`; `cron` runs `supercronic /app/infra/cron/crontab` as non-root (it skips the web bootstrap — migrate/collectstatic are `web`'s job). In prod, `nginx` `depends_on` `web` (started) **and** `certbot` (healthy — its healthcheck passes only once a live `fullchain.pem` exists), so nginx never crash-loops on a missing cert. All use `restart: unless-stopped`.
- **Shutdown & cleanup.** SIGTERM drains gunicorn in-flight requests and lets Celery finish in-flight tasks. Named volumes persist across recreation: `pjs-pgdata`, `pjs-rabbit`, `pjs-static`, `pjs-media`, `pjs-logs`, `pjs-control`, plus the observability volumes (`pjs-prometheus`, `pjs-loki`, `pjs-grafana`, …) and prod-only `pjs-letsencrypt`.
- **Healthy / readiness vs liveness.** Liveness = the process answers `GET /health/`. Readiness = dependencies reachable, surfaced by `GET /status/` (DB/cache/broker + observability). Compose healthchecks: `web` curls `/health/` (30 s, 30 s start); `worker` = `celery inspect ping`; `postgres` = `pg_isready`; `rabbit` = `rabbitmq-diagnostics ping`; `redis` = `redis-cli ping`; nginx probes its `:8080` local endpoint; certbot tests for the live cert file.
- **Isolated volume mounts.** `pjs-control` (the `shop_open` marker) is mounted **read-only** into nginx and read-write only into `web`; `pjs-pgdata` is dedicated and non-shared; `pjs-media`/`pjs-static` are shared read-only into nginx; Promtail/cAdvisor mount `/var/lib/docker/containers` and the Docker socket **read-only**. In prod the app containers run **non-root (`uid 1001`), read-only rootfs, `cap_drop: ALL`, `no-new-privileges`**, with tmpfs for `/tmp`, `/dev/shm`, `/home/app`.

### Failure behavior

- **If the platform is unavailable.** Because it is one app, an outage takes down the storefront API, the mobile stock API and the admin together; the storefront degrades to the nginx **maintenance page** (the `/` and `/api/` 502/503/504 → `maintenance.html`), and IMApp surfaces errors with no offline mode. The datastores (`postgres`/`rabbit`/`redis`) are the hard dependencies — a missing DB/broker credential **fails boot loudly** (`${VAR:?}`) rather than starting a degraded guest.
- **In-process seams remove cross-module partial failure.** The former DBU↔IM↔Shop↔Lexware↔MailBot network calls are now function calls in one transaction-scoped process, so there is no inter-service retry/backoff or partial-delivery to reason about for those paths. What remains: `inventory.services.receive_order` is **idempotent on `order_number`**; the shop→inventory sale push is in-process and any error is logged without breaking checkout; `inbox` dedups by `message_id`; invoicing job triggers are **non-idempotent** (poll the job before re-running).
- **External dependency degradation (fail-soft).** PayPal outage → checkout `502/503`, cart preserved. Lexoffice/VATSense outage → the affected invoicing job fails or falls back (VAT default `19`); empty keys disable the feature. GMX IMAP unreachable → the next poll simply finds nothing; no mail password → console email backend. R2 unset → local filesystem storage.
- **Local mocking / standalone.** `docker compose up` (auto-loads `docker-compose.override.yml`) runs the full stack locally with `DEBUG`, source-mounted, plain-HTTP nginx (maintenance door only) — no Cloudflare/TLS/secrets needed. PayPal defaults to sandbox; Lexoffice/VATSense/R2/mail are all fail-soft when blank, so the platform boots and is exercisable with an otherwise empty `.env` (only the DB/broker/Grafana credentials are required).

### Environment contract

One gitignored root `.env` (template `.env.example`). Datastore/broker/Grafana credentials are **required with no fallback** (`${VAR:?}` — a missing secret fails boot). Selected variables:

| Variable | Type | Default | Secret-managed |
| --- | --- | --- | --- |
| `DJANGO_MODE` | enum | `production` | no |
| `DEBUG` | bool | `False` | no |
| `SECRET_KEY` | string | — | **yes** |
| `ALLOWED_HOSTS` | csv | — (public apex + admin/im-api hosts) | no |
| `CORS_ALLOWED_ORIGINS` | csv | — (the SPA origin[s]) | no |
| `CSRF_TRUSTED_ORIGINS` | csv | — (admin host) | no |
| `SECURE_SSL_REDIRECT` / `SECURE_HSTS_*` | bool/int | `True` / `63072000` | no |
| `LOG_LEVEL` | enum | `INFO` | no |
| `DB_NAME` / `DB_USER` / `DB_PASSWORD` | string | — | **yes** (required) |
| `DB_HOST` / `DB_PORT` | string/int | `postgres` / `5432` | no |
| `RABBITMQ_DEFAULT_USER` / `RABBITMQ_DEFAULT_PASS` | string | — | **yes** (required) |
| `CELERY_BROKER_URL` | url | `amqp://USER:PASS@rabbit:5672//` | no |
| `CELERY_WORKER_CONCURRENCY` | int | `1` | no |
| `CELERY_RESULT_BACKEND` / `CACHE_URL` | url | `redis://redis:6379/0` · `…/1` | no |
| `CONTROL_DIR` | path | `/control` | no |
| `GRAFANA_USER` / `GRAFANA_PASSWORD` | string | — | **yes** (required) |
| `PAYPAL_ENV` | enum | `sandbox` (`live` in prod) | no |
| `PAYPAL_CURRENCY` | string | `EUR` | no |
| `PAYPAL_CLIENT_ID` / `PAYPAL_SECRET` / `PAYPAL_WEBHOOK_ID` | string | — (sandbox falls back to test creds) | **yes** (live) |
| `FRONTEND_BASE_URL` | url | — (password-reset links) | no |
| `ORDER_IMPORT_EMAIL` | email | — (supplier reorder CSV recipient) | no |
| `LEXOFFICE_API_KEY` / `VATSENSE_API_KEY` | string | — (empty disables) | **yes** |
| `GMX_MAIL` / `GMX_PASSWORD` | string | — (empty → console email backend) | **yes** |
| `DEFAULT_FROM_EMAIL` | email | `pjs-collectables@gmx.net` | no |
| `IMAP_HOST` / `IMAP_PORT` / `IMAP_USE_SSL` / `IMAP_MAILBOX` | string/int/bool | `imap.gmx.net` / `993` / `True` / `INBOX` | no |
| `IMAP_USER` / `IMAP_PASSWORD` | string | default to `GMX_MAIL`/`GMX_PASSWORD` | **yes** |
| `IS_FINAL` / `IS_ACTIVE_OSS` / `IS_NEW_TAX` / `IS_CREATING` / `IS_ASSIGNING` | bool | invoicing job toggles | no |
| `R2_BUCKET` / `R2_ACCESS_KEY_ID` / `R2_SECRET_ACCESS_KEY` / `R2_ENDPOINT_URL` | string | — (all four set → R2; else local FS) | **yes** |
| `R2_CUSTOM_DOMAIN` / `R2_REGION` | string | — / `auto` | no |
| `IM_API_PREFIX` / `IM_APP_SECRET` | string | — (prod overlay, **required**) | **yes** |
| `CLOUDFLARE_DNS_API_TOKEN` / `LETSENCRYPT_EMAIL` | string | — (prod overlay, **required**) | **yes** |
| `CERTBOT_DOMAIN` / `CERTBOT_STAGING` | string/bool | `pjscollectables.com` / `0` | no |
| `IMAGE_TAG` / `GUNICORN_WORKERS` | string/int | `latest` / `3` | no |

> Superusers are created manually, out of band (`docker compose exec web python manage.py createsuperuser`) — there is no deploy-time `DJANGO_SUPERUSER_*` bootstrap. Any secret that ever lived in the five former repos' git history must be rotated at the provider before go-live.

### Observability

- **Logs.** All containers use the Docker `json-file` driver (10 MB × 3); Promtail tails them into Loki. nginx writes a JSON-ish access log including the Cloudflare `CF-IPCountry` country code (`$http_cf_ipcountry`) — no GeoIP module needed. `core/trace.py` propagates correlation IDs so a request can be followed across modules.
- **Metrics.** `GET /metrics` (django-prometheus) for the whole app — this consolidates the per-service `/metrics` the former DBUpdater/InventoryManager/invoicing each exposed. cAdvisor adds per-container CPU/mem/net; nginx is scraped on `pjs-monitoring`. Prometheus scrapes every 15–30 s; **Alertmanager** routes fired rules to a webhook (the always-firing Watchdog rule exercises the pipeline).
- **Health endpoints.** `/health/` (liveness), `/status/` (aggregated readiness across web/promtail/grafana/prometheus); per-service Compose healthchecks for the datastores and observability containers.
- **Dashboards.** Grafana (VPN-only at `grafana.pjscollectables.com`) unifies Prometheus metrics + Loki logs for a whole-platform view.

---

# PJsShopFront Context

## Summary

PJsShopFront is the Vue 3 + Vite storefront UI for PJS Collectables. It renders the customer-facing catalog, cart, checkout and profile flows while delegating all authoritative commerce data and payment orchestration to the [pjscollectables-platform](#pjscollectables-platform-context) `shop` API. It is the ecosystem's browser entrypoint: it drives user sessions, fetches product/category data, and triggers order/payment flows through the backend. It ships its **own** Docker Compose stack and joins the platform's shared `pjs-net` bridge as an external network, so nginx can proxy `/` to it — deploying or redeploying the monolith never starts or stops it, and vice versa. **Status: Active — Production.**

```mermaid
flowchart LR
    Browser["Customers / Admins"] -->|HTTPS via nginx| Frontend["PJsShopFront (Vue)<br/>internal DNS: pjs-shop-front:5173"]
    Frontend -->|/api same-origin (REST + session cookies)| ShopAPI["platform shop API<br/>internal DNS: web:8000"]
    Frontend -->|PayPal SDK (client id from backend)| ShopAPI
```

## Purpose

PJsShopFront delivers the interactive customer experience. It translates catalog and checkout flows into UI actions while relying on the platform as the canonical record for products, carts, orders and payment capture. It **owns** only client-side UI state and the browser session bootstrap; it **depends entirely on** the platform `shop` API and holds no business data. This frontend is critical because it is the only project that turns the backend into an accessible storefront and provides the authenticated session flow bridging browser users to the transactional backend.

**Non-goals / boundary:** PJsShopFront performs no authoritative computation — it never holds product truth, cart totals or payment secrets (all owned by the platform `shop` module), and it never talks to the `catalog` or `inventory` modules directly. It is a pure presentation/consumer leaf.

## Repository layout
- `index.html` – Vite HTML entrypoint.
- `src/` – Vue application source.
  - `main.js` – app bootstrap (Vue + router); `App.vue` – root layout.
  - `router/` – route definitions; `views/` – page components (catalog, cart, profile, checkout); `components/` – reusable UI.
  - `services/` – API client (`api.js`), global session/cart state (`globals.js`), profile helper.
  - `assets/` – static images/styling.
- `public/` – static files served as-is.
- `package.json`, `vite.config.js`, `tailwind.config.js`, `postcss.config.js` – build/styling config.
- `SOFTWARE_ECOSYSTEM.md` – this document (the frontend repo is its current home).

## Integration with other Projects

### Exposes
- A browser SPA — it exposes **no inbound API** to other services. Its only "interface" is the rendered storefront for end users.

### Consumes
PJsShopFront consumes the [pjscollectables-platform](#pjscollectables-platform-context) `shop` REST API exclusively. In dev, Vite proxies `/api` to `VITE_API_PROXY_TARGET` (default `http://web:8000` — the monolith's web container, formerly `back-web:8000`); the Axios base (`src/services/api.js`) is `VITE_API_BASE_URL` or same-origin `/api/`. It assumes cookie-based sessions + CSRF and does not authenticate with the internal modules.

| Interface (platform `shop`) | Location | Auth | Notes | Client-facing errors |
| --- | --- | --- | --- | --- |
| Session bootstrap | `GET/POST /api/session/` | None | `sessionid`, `csrftoken`, `is_authenticated`. | 403 (CSRF), 5xx |
| Categories | `GET /api/categories/`, `/api/categories/nav/` | None | Filter by `parent`/`slug`/`name`/`is_active`; nav tree. | 404/5xx |
| Products | `GET /api/products/`, `/api/products/facets/` | None | Pagination + `category__name` filter; facets. | 404/5xx |
| Cart | `GET/POST /api/cart/`, `/api/cart/add/`, `/api/cart/remove/`, `/api/cart/update/` | Session + CSRF | Cart payload with items/totals. | 401/403, 5xx |
| Checkout | `GET /api/checkout/initiate/` | Session | Address defaults + cart state. | 401/403, 5xx |
| Orders | `POST /api/orders/place/` | Session + CSRF | Manual (bank-transfer) order placement. | 400, 401/403, 5xx |
| Auth | `POST /api/login/`, `/api/register/`, `/api/logout/`, `/api/profile/delete/`, `/api/password-reset/` | Session + CSRF | Credentials / profile ops. | 400, 401/403 |
| Profile | `GET/PATCH /api/profile/`, `/api/profile/wishlist/`, `/api/profile/orders/` | Session + CSRF | User details, addresses, wishlist, orders. | 401/403, 5xx |
| PayPal | `GET /api/paypal/client-id/`, `POST /api/paypal/create-order/`, `POST /api/paypal/capture-order/` | Server-side secrets | Init + capture proxied via the backend. | 400, 502/503 |

**Auth flow:** call `/api/session/` once per browser session and attach `X-CSRFToken` on mutating requests; cookies are stored by the browser and reused by Axios. **PayPal:** after approval the frontend makes the single `POST /api/paypal/capture-order/` call with `{ orderId, terms_accepted }`; the backend captures, re-verifies the amount, persists orders and returns `{ orders }` (amounts never come from the client). **If the platform is unavailable:** nginx serves the maintenance page, and the UI shows a maintenance/error state and disables cart/checkout until `/api/session/` and catalog endpoints respond.

## Core services

### API client and session bootstrap
`src/services/api.js` defines a shared Axios client with `withCredentials: true` and base `VITE_API_BASE_URL || '/api/'`, initialises the session via `/api/session/`, stores the CSRF/session identifiers, and injects `X-CSRFToken` on POST/PUT/PATCH/DELETE — the single integration point for authenticated browser flows.

### Catalog browsing
`fetchCategoryAndSubcategories`, `fetchProductsByCategoryNames`, `fetchUpcoming` orchestrate category/product queries (pagination, hierarchical categories), backing the catalog and upcoming-product views.

### Cart and checkout flows
Global cart state in `src/services/globals.js` (reactive Vue state) with `fetchCartDetails`/`addToCart`/`removeFromCart`; checkout via `fetchCheckoutInitiate`/order placement — the platform remains the source of truth for totals.

### Account, profile & PayPal
`loginUser`/`registerUser`/`logoutUser`/`deleteUser`/`fetchProfile`/`updateProfile` (`useProfile` wraps the profile API into a reactive object); PayPal integration requests the client ID from `/api/paypal/client-id/`, initialises the SDK, and relays approval to the backend capture endpoint — all secrets stay server-side.

## Operations & Lifecycle

### Runtime behavior
- **Startup dependencies & ordering:** needs the platform `shop` API reachable at the configured base/proxy before dynamic data loads; static assets/routing load without it but catalog/cart/profile data will fail. Ships its own compose stack on `pjs-net`; its image tag lives in this repo's `.env`, not the platform's.
- **Shutdown & cleanup:** stateless — stopping the Vite dev server or static host terminates it with no cleanup.
- **Healthy / readiness vs liveness:** liveness = the static bundle is served; readiness = the platform reachable and `/api/session/` returns 200. nginx shows the maintenance page when this stack is down/absent.
- **Isolated volume mounts:** none; avoid mounting `node_modules`/build output into shared volumes.

### Failure behavior
- **If the platform is down:** nginx serves the maintenance page; the SPA shows maintenance/error and disables cart and checkout until session + catalog endpoints recover.
- **Retry/backoff & idempotency:** minimal client retries with backoff on 5xx for catalog reloads; **never** repeat order/capture POSTs without user confirmation (order capture is not idempotent client-side — disable submit, single PayPal capture per order).
- **Partial failures:** PayPal capture may succeed while a follow-up fails; surface the failure and instruct the user to retry or contact support.
- **Local mocking/stubbing:** mock API responses for UI-only work, or run the platform locally and keep the proxy on `web:8000`.

### Environment contract
Build/runtime config via Vite env (`import.meta.env`).

| Variable | Type | Default | Secret-managed |
| --- | --- | --- | --- |
| `VITE_API_PROXY_TARGET` | url | `http://web:8000` | no |
| `VITE_API_BASE_URL` | url | `/api/` (same-origin) | no |
| `IMAGE_TAG` | string | per this repo's `.env` | no |

No secrets live in the frontend — all PayPal/credentials stay server-side in the platform.

### Observability
- **Logs:** browser console / Vue DevTools in development; production observability is via the served-asset host and the backend it calls (the frontend emits no server metrics).
- **Health/correlation:** readiness is inferred from successful `/api/session/` + catalog calls; backend correlation happens server-side in the platform.
- **Scaling & evolution:** stateless — scale via static hosting or multiple containers; the only constraint is backend capacity. Keep API routes, the session/CSRF flow and payload schemas stable; UI structure/styling can evolve freely.

---

# IMApp Context

## Summary

IMApp is the internal **Android client for the platform's `inventory` module** — a Kotlin + Jetpack Compose app (single-activity, Compose UI, OkHttp networking) that lets staff manage stock from a phone: log in, list inventory, add/reduce/delete stock, and query/confirm catalogue items. It is distributed as a **signed APK installed directly on the device** (no Play Store) and talks only to the hardened public mobile API. **Status: Active — Internal sideload.**

```mermaid
flowchart LR
    Phone["IMApp (Android)<br/>Kotlin/Compose"] -->|HTTPS + X-App-Secret + Token| CF["Cloudflare"]
    CF -->|im-api.pjscollectables.com/PREFIX/api/inventory| NGINX["nginx (allow-list + rate-limit + AOP)"]
    NGINX --> WEB["platform web:8000<br/>/api/inventory/…"]
    Phone -.->|debug build| Emu["http://10.0.2.2:8000/api/ (emulator → local platform)"]
```

## Purpose

IMApp gives operators a mobile control surface over the inventory ledger without exposing the full Django admin to the internet. It **depends entirely on** the [pjscollectables-platform](#pjscollectables-platform-context) `inventory` module (via the hardened `im-api` gateway in the platform's nginx) and **owns** nothing server-side — only an encrypted on-device DRF token and transient UI state. It is the human, mobile counterpart to the same `inventory.services` seam that the `shop` and `inbox` modules write to in-process.

**Non-goals / boundary:** IMApp is not a customer app (that is [PJsShopFront](#pjsshopfront-context)), does not talk to the `catalog` module directly (catalogue queries are proxied through inventory's `query-dbu-item`/`confirm-item`), has no offline database, and exposes no inbound interface. It is a pure leaf client on the operations boundary.

## Repository layout
- `app/src/main/java/com/InventoryManager/imapp/`
  - `core/network/` – `ApiClient.kt` (shared OkHttp), `AuthInterceptor.kt` (adds token + `X-App-Secret`).
  - `core/auth/SessionManager.kt` – encrypted token storage (EncryptedSharedPreferences).
  - `core/model/` – DTOs such as `InventoryItem.kt`; `core/navigation/`, `core/ui/`, `core/AppGraph.kt`, `core/IMApplication.kt`.
  - `feature/{login,register,inventory}/` – screens + ViewModels + state.
  - `theme/`, `MainActivity.kt`, `NavHost.kt` – Compose theme and host.
- `app/src/debug/.../network_security_config.xml` – cleartext allowed only for the emulator host in debug.
- `app/build.gradle.kts` – build types, signing, `BuildConfig` fields.
- `RELEASE.md` – signing keystore, Gradle-property secrets, build/sideload steps.

## Integration with other Projects

### Exposes
- Nothing. IMApp is a leaf client with **no inbound interface**; it only makes outbound calls.

### Consumes
The platform's inventory mobile API, via the hardened gateway:
- **Base URL:** release `https://im-api.pjscollectables.com/<PREFIX>/api/`; debug `http://10.0.2.2:8000/api/` (emulator → local platform), from `BuildConfig.API_BASE_URL`. In the monolith the stock endpoints live under **`/api/inventory/`** (namespaced in Phase 2); the edge rewrites `/<PREFIX>/api/inventory/…` → `/api/inventory/…` onto `web:8000`.
- **Headers:** `X-App-Secret: <APP_SECRET>` (release only) + `Authorization: Token <token>` (added by `AuthInterceptor`).
- **Endpoints used (allow-listed at nginx, under `/api/inventory/`):** `auth/token`, `inventory`, `add-stock`, `reduce-stock`, `delete-item`, `query-dbu-item`, `confirm-item`.
- **Auth lifecycle:** log in with username/password → receive DRF token → store encrypted on-device; on any `401` the app clears the token and returns to login. "Log out" clears it manually.
- **If unavailable:** if the platform or the gateway is unreachable (or the secret/prefix/allow-list/rate-limit rejects the request → `444`/`429`), the app surfaces an error; there is no offline mode and no local cache.

## Core services

### ApiClient (`core/network/ApiClient.kt`)
A single shared `OkHttpClient` initialised once from `IMApplication`, with connect/read/write timeouts of 15/30/30 s. The base URL and app secret come from `BuildConfig` so they differ per build type with no hardcoded values. `url(path)` resolves endpoint paths against the base URL.

### AuthInterceptor (`core/network/AuthInterceptor.kt`)
Attaches the stored DRF token and the `X-App-Secret` header to every outbound request.

### SessionManager (`core/auth/SessionManager.kt`)
Persists the token using `EncryptedSharedPreferences` (androidx.security-crypto) so the credential is encrypted at rest on the device.

### Feature ViewModels (`feature/login`, `feature/register`, `feature/inventory`)
Coroutine-based ViewModels (`viewModelScope`) drive login/register and the inventory list + stock mutations, exposing reactive state to the Compose screens.

## Operations & Lifecycle

### Runtime behavior
- **Build/install lifecycle (no container):** `./gradlew :app:assembleRelease` → signed `app-release.apk` → `adb install -r` or manual sideload ("install unknown apps"). Debug builds target the emulator loopback; release builds are HTTPS-only and carry `X-App-Secret`.
- **First run / "healthy":** healthy = the app can reach the gateway, obtain a token, and load `/inventory`. On launch it restores the encrypted token if present, else routes to login.
- **Readiness vs liveness:** liveness = the app process runs and renders; readiness = a valid token + reachable inventory mobile API.
- **State:** the only persisted state is the encrypted token; clearing app data or logging out removes it.

### Failure behavior
- **If the platform/gateway is unavailable or rejects:** errors are surfaced; `401` triggers token clear + relogin; edge rate limits (login `5/min`, ops `2/s` burst 10; Cloudflare WAF) throttle abuse.
- **Retry/backoff & idempotency:** no automatic retry storms; stock mutations are user-initiated single actions — the user re-issues on failure rather than the app auto-retrying.
- **Partial failures:** a failed mutation leaves server state unchanged (mutations are discrete API calls); the UI reflects the last successful read.
- **Local mocking/stubbing:** run a debug build against a local platform at `http://10.0.2.2:8000/api/` (cleartext allowed only for that host) to develop without the public gateway.

### Environment contract
Secrets are supplied at build time via **non-committed Gradle properties** (`~/.gradle/gradle.properties` or a local untracked file); when absent, debug still builds and release builds unsigned.

| Property | Type | Default | Secret-managed |
| --- | --- | --- | --- |
| `imApiBaseUrl` | url | `https://im-api.pjscollectables.com/CHANGE_ME_PREFIX/api/` | **yes** (contains secret prefix) |
| `imAppSecret` | string | `""` | **yes** (`X-App-Secret`) |
| `imKeystoreFile` / `imKeystorePassword` / `imKeyAlias` / `imKeyPassword` | string/path | unset (unsigned release if absent) | **yes** (signing) |
| `BuildConfig.API_BASE_URL` / `BuildConfig.APP_SECRET` | derived | from the above per build type | n/a |

The prefix and app secret must match the platform's `IM_API_PREFIX`/`IM_APP_SECRET` (rendered into `infra/nginx/nginx.prod.conf.template`; see `infra/nginx/MOBILE_API.md`).

### Observability
- **Logs:** OkHttp `HttpLoggingInterceptor` at `BASIC` (method + URL + status) in debug, `NONE` in release — bodies/headers are **never** logged, so tokens/secrets don't leak to Logcat.
- **Metrics/health:** none server-side; the app has no metrics endpoint. Operational health is observed from the platform side (the `im-api` edge logs `444`/`429`/auth failures).
- **Build/version:** `versionCode`/`versionName` in `app/build.gradle.kts`; `compileSdk`/`targetSdk` 35, `minSdk` 24.

---

# Development Regulation Standards
The software ecosystem is designed in alignment with **ISO/IEC 25010**, the _de facto international standard_ for defining software quality categories — ensuring that attributes are systematically applied across all projects.  Assessment becomes **python-specific, static read-only, standards-informed audit**.

---
## 🧮 Universal Scoring Rubric

### 🎚️ Scoring Scale (applies to all sub-characteristics)

|Score|Definition|
|---|---|
|**1**|Absent / violates basic principle (anti-pattern, no evidence)|
|**2**|Major gaps — barely partially implemented, or unstable|
|**3**|Minimal compliance — implemented ad-hoc or inconsistently|
|**4**|Partial — visible intent but lacks structure or completeness|
|**5**|Basic compliance — meets minimum industry norm, not automated|
|**6**|Reasonable — consistent and traceable but not enforced|
|**7**|Solid — good practice evident, minor improvement areas|
|**8**|Mature — standards clearly implemented and verified|
|**9**|Exemplary — fully automated enforcement and documentation|
|**10**|Benchmark — formalized, continuously validated, measurable quality gates|

> Sub-characteristics are **rounded to the nearest integer (1–10)** after evaluation. The mean of a category is a **decimal**.

---

### ⚖️ Category Weighting (in % of total quality score)

| Top-Level Characteristic  | Weight   | Rationale (industry convention)                                 |
| ------------------------- | -------- | --------------------------------------------------------------- |
| 🧠 Functional Suitability | **20 %** | Core correctness and completeness                               |
| ⚡ Performance Efficiency  | **15 %** | Speed + resource control are major but secondary to correctness |
| 🔗 Compatibility          | **5 %**  | Important for integration, lower standalone impact              |
| 🎨 Developer Usability    | **10 %** | Human interaction quality; varies by product type               |
| 🛡️ Reliability           | **10 %** | Stability and fault tolerance                                   |
| 🔒 Security               | **20 %** | Universal criticality (OWASP + ISO 27001 alignment)             |
| ⚙️ Maintainability        | **15 %** | Determines long-term cost and agility                           |
| 🚀 Portability            | **5 %**  | Valuable but rarely decisive for Python systems                 |
## 1. 🧠 Functional Suitability

Evaluate whether the software **fulfills its intended purpose** by providing all required functions, producing correct results, and supporting user goals effectively.

---

### ✅ **Functional Completeness**

Extent to which all necessary capabilities are implemented.

**Assessment Questions:**

- Verify that every intended user or system requirement has a corresponding implementation.
    
- Detect missing, redundant, or partially implemented features through static code analysis and test coverage mapping.
    
- Confirm that each functional area is represented by at least one automated or manual test case.
    

**Supporting standards & practices:**

- **ISO/IEC/IEEE 29148** – Requirements specification and traceability
    
- **ISO/IEC 25023** – Metrics for functional completeness and coverage
    
- **Presence of coverage reports** – Feature-to-test verification
    

---

### 🎯 **Functional Correctness**

Degree to which the software produces accurate and consistent results under defined conditions.

**Assessment Questions:**

- **Static presence of test cases** matching functions.
    
- Analyze logic paths and data validation for potential functional errors.
    
- Track and categorize functional defects to measure stability and reliability.
    

**Supporting standards & practices:**

- **ISO/IEC/IEEE 29119** – Software testing processes and reporting
    
- **Unit and Integration Testing Frameworks** – (PyTest)
    

---

### ⚙️ **Functional Appropriateness**

Degree to which functions support specific user tasks and achieve intended goals efficiently.

**Assessment Questions:**

- Evaluate alignment of implemented features with actual user workflows and goals.
    
- Identify redundant or overly complex functions that hinder usability or task efficiency.
    
- Measure user-task completion rates and performance within typical usage scenarios.
    

**Supporting standards & practices:**

- **ISO 9241-210** – Human-centred design for interactive systems
    
- **Design Thinking Frameworks** – Goal-oriented functional validation
    

---

## 2. ⚡ Performance Efficiency

Optimize response and resource use under defined conditions.

### ⏱️ Time Behaviour

Measure and control response time, throughput, and latency.  
**Supporting standards & practices:**

- **ACM SIGPLAN Performance Guidelines** – software timing optimization
    
- **ISO/IEC 29155** – performance efficiency measurement
    
- **Presence of load-test scripts/configs**
    

### 🧮 Resource Utilization

Assess how efficiently the software **uses CPU, memory, storage, and I/O resources** based on code structure, algorithm design, and configuration evidence.

**Assessment Questions:**

- Detect inefficient algorithms (e.g., nested loops, unbounded recursion).
    
- Identify excessive in-memory data or unbounded caching.
    
- Check for repeated database or file I/O inside loops.
    
- Review configuration limits (thread pools, connection pools, worker counts).
    
- Verify cleanup of resources (closing files, joining threads, releasing memory).
    
- **Static pattern detection** (e.g., nested loops, unbounded recursion, repeated I/O in loops)
    

**Supporting standards & practices:**

- **ISO/IEC 29155 / 25023** – define performance-efficiency and utilization measurement goals.
    
- **ACM SIGPLAN Efficiency Guidelines** – algorithmic and computational optimization principles.

### 📈 Capacity (Scalability)

Evaluate whether the system’s **architecture and implementation** can sustain performance as load, users, or data volumes increase.

**Assessment Questions:**

- Detect _stateless_ design patterns that enable horizontal scaling.
    
- Check for externalized session/data storage (e.g., Redis, database, S3) instead of in-memory state.
    
- Identify load-balancer or worker configuration parameters (thread pools, queue sizes, auto-scale settings).
    
- Verify that components use asynchronous or distributed processing when appropriate.
    
- Review database and cache design for sharding, pagination, or partitioning readiness.
    
- Ensure API endpoints and background jobs handle concurrency safely (no shared mutable state).
    

**Supporting standards & practices:**

- **ISO/IEC 29155 / 25023** – define scalability and capacity efficiency metrics.
    
- **CNCF Cloud-Native Best Practices** – stateless, containerized, horizontally scalable design principles.
    
- **Stateless Service Design** – promotes independent scale-out instances.
    
- **Infrastructure as Code Conventions** – Terraform, Kubernetes manifests defining replica/auto-scale logic.
## 3. 🔗 Compatibility

Ensure interoperability and coexistence with other systems.

### ⚖️ Co-existence

Assess whether the software can **operate correctly in shared environments**—such as multi-tenant systems, container clusters, or composite deployments—**without interfering** with other applications or services.

**Assessment Questions:**

- Verify adherence to **container and runtime standards** (e.g., OCI, Dockerfile compliance).
    
- Check for **hard-coded ports, paths, or environment assumptions** that may cause conflicts.
    
- Ensure **resource limits** (CPU, memory, network) are defined to prevent contention.
    
- Confirm that **logging, temporary files, and caches** use isolated directories or namespaces.
    
- Inspect configuration for **environment variable injection** instead of static credentials.
    
- Detect **shared resource dependencies** (e.g., same DB schema, file paths) that may break isolation.
    
- Evaluate service discovery and networking configs for **namespace or label separation**.
    

**Supporting standards & practices:**

- **ISO/IEC 25010 → Compatibility / Co-existence** – conceptual definition.
    
- **OCI / Docker Specification** – portable, isolated container execution.
    
- **POSIX Compliance** – predictable file and process behavior across systems.
    
- **CNCF Service Mesh Patterns (Istio, Linkerd)** – declarative inter-service coexistence (used as architectural reference, not runtime check).
    
- **Kubernetes Best Practices** – resource quotas, namespaces, and environment isolation.

### 🔄 Interoperability

Exchange and process information between systems effectively.  
**Supporting standards & practices:**

- **OpenAPI / REST / GraphQL** – interface definition standards
    
- **ISO/IEC 19510 (BPMN)** – process interoperability
    
- **JSON Schema / Protocol Buffers** – standardized data formats
---

## 4. 🎨 Usability (or Developer Usability)

If a **user interface (UI)** is present, evaluate traditional UX quality (learnability, operability, satisfaction).  
If **no UI is present**, evaluate **developer usability** — how easily the code can be understood, configured, and safely operated in an editor or terminal.

---

### 👁️ Appropriateness Recognizability

Users or developers can easily understand the **purpose and usage** of the software.

**Audit questions:**

- Are file, class, and function names self-descriptive and consistent?
    
- Does the project layout clearly expose its main entry points (`main.py`, CLI, module root)?
    
- Is there a `README.md` or equivalent that explains intent and usage?
    
- For UI projects: are labels, menus, and actions named consistently and clearly?
    

**Supporting standards & practices:**

- **ISO 9241-210** – human-centered design lifecycle
    
- **PEP 8 / PEP 257** – naming and docstring conventions
    
- **PyPA Layout Guidelines** – discoverable module structure
    

---

### 📚 Learnability

Ease with which a new user or developer can learn to use or contribute.

**Audit questions:**

- Is there a quick-start or setup guide that runs without guesswork?
    
- Are docstrings and type hints present to explain expected input/output?
    
- Are functions and modules short and conceptually focused?
    
- For UI projects: is the flow predictable and consistent between screens?
    

**Supporting standards & practices:**

- **Nielsen Heuristics** – simplicity and consistency
    
- **ISO 9241-110** – dialogue principles
    
- **PEP 484** – explicit type annotations
    

---

### 🕹️ Operability

Ease of controlling or executing the system.

**Audit questions:**

- Can the program be run with one clear command (`python -m`, `make run`, CLI help)?
    
- Are configurations centralized and environment-driven (`.env.example`, defaults)?
    
- Do logs and errors use readable, actionable messages?
    
- For UI: are controls responsive and consistent with expectations?
    

**Supporting standards & practices:**

- **ISO 9241-110** – feedback and operability
    
- **argparse / click** – Python CLI standards
    
- **Twelve-Factor App Principle III** – configuration via environment
    

---

### 🚫 User / Developer Error Protection

Prevent or reduce the impact of incorrect usage.

**Audit questions:**

- Does the code validate external input before processing?
    
- Are exceptions handled and reported with clear context (no silent fails)?
    
- Are destructive actions guarded by confirmation or dry-run options?
    
- For UI: are error messages specific and recoverable?
    

**Supporting standards & practices:**

- **ISO/TR 16982** – usability evaluation methods
    
- **PEP 3134** – clear exception chaining
    
- **Bandit / Ruff** – static safety enforcement
    

---

### 💅 Interface Aesthetics (UI or Code)

Maintain clarity and stylistic consistency.

**Audit questions:**

- Is formatting uniform (indentation, imports, naming)?
    
- Are linting/formatting configs present (`pyproject.toml`, `.ruff.toml`, etc.)?
    
- Are comments concise and aligned with code intent?
    
- For UI: is visual layout consistent and uncluttered?
    

**Supporting standards & practices:**

- **PEP 8** – Python style guide
    
- **Black / Ruff / isort** – automated enforcement
    
- **Material / Fluent Design** – if UI present
    

---

### ♿ Accessibility

Ensure usability for diverse users or accessibility of documentation.

**Audit questions:**

- Are docs provided in readable plain-text or markdown (not only images)?
    
- Are CLI tools keyboard-navigable with meaningful output for screen readers?
    
- Are accessibility or localization configs present when UI exists?
    

**Supporting standards & practices:**

- **EN 301 549 / WCAG 2.1** – accessibility compliance (UI)
    
- **PEP 12 / reStructuredText** – accessible text documentation
## 5. 🛡️ Reliability

Ensure consistent performance and fault tolerance under all conditions.

### 🧱 Maturity

Minimize failure frequency under normal conditions.  
**Supporting standards & practices:**

- **IEEE 1633** – reliability program standard
    
- **MTBF/MTTR Metrics** – reliability measurement
    

### 🌐 Availability

Ensure uptime and readiness of service.  

**Assessment Questions**:

- **Review of monitoring or alerting config files** (Prometheus rules, health endpoint docs)

- **Operational Health Signaling Requirement** (Every long-running service in the ecosystem **MUST expose** a `/health` endpoint that returns **HTTP 200 OK** when the service is operational.  )  

**Supporting standards & practices:**

- **SRE Principles(Google Site Reliability Engineering)** – availability engineering
    
- **ISO/IEC 20000** – IT service management
    
- **SLAs and Monitoring Alerts** – uptime enforcement
    

### 🧩 Fault Tolerance

Continue operation during faults or partial failures.  
**Supporting standards & practices:**

- **Redundancy & Failover Patterns** – resilience design
    
- **Chaos Engineering (Netflix Principles)** – fault injection testing simulation
    
- **Circuit Breaker / Retry Logic** – graceful degradation
    

### ♻️ Recoverability

Restore state and data after incidents.  
**Supporting standards & practices:**

- **ISO/IEC 27040** – backup and data recovery
    
- **NIST SP 800-34** – disaster recovery planning
    

---

## 6. 🔒 Security

Protect systems and data from unauthorized access and modification.

### 🔐 Confidentiality

Ensure that **information is accessible only to authorized entities** and protected against accidental or deliberate disclosure.

**Assessment Questions (AI-applicable):**

- Detect exposure of **secrets or credentials** in code, configs, or logs.
    
- Check for use of **secure transmission** (HTTPS/TLS, SSH) and encryption libraries.
    
- Verify presence of **access-control logic** (role checks, token validation, ACLs).
    
- Inspect API endpoints for **over-exposed data fields** or unfiltered responses.
    
- Confirm proper **logging and masking** of sensitive data (no passwords, tokens, PII).
    
- Scan of dependency files for pinned versions and presence of security scan config.
    
- Ensure configuration and environment variables isolate sensitive keys from source control.
    

**Supporting standards & practices:**

- **ISO/IEC 27001 / NIST SP 800-53** – formal information-security control frameworks.
    
- **OWASP Top 10** – open web-security baseline for identifying common exposure risks.
    
- **CWE / CVE Databases** – reference for known vulnerability classes and identifiers.
    
- **ISO/IEC 27002** – detailed security control catalogue (data protection, access management).
    
- **OWASP ASVS (Level 1–3)** – concrete application-security verification requirements.

### 🧾 Integrity

Protect against unauthorized data changes.  
**Supporting standards & practices:**

- **Hashing / Checksumming (SHA, CRC)** – data validation
    
- **Digital Signature Algorithms (FIPS 186-4)** – verification of integrity
    

### 🧭 Non-repudiation

Provide proof of actions or transactions.  
**Supporting standards & practices:**

- **Audit Logging / Blockchain Systems** – event traceability
    
- **PKI Standards (X.509)** – cryptographic signing
    

### 🧑‍💼 Accountability

Trace and attribute actions to responsible entities.  
**Supporting standards & practices:**

- **ISO/IEC 27002** – logging and audit control
    
- **SIEM Systems** – centralized accountability management
    

### 🪪 Authenticity

Verify identity of users and systems.  
**Supporting standards & practices:**

- **OAuth 2.0 / OpenID Connect** – authentication protocols
    
- **FIDO2 / WebAuthn** – hardware-backed authentication
---

## 7. ⚙️Maintainability

Keep modules loosely coupled and well-encapsulated. Avoid unnecessary complexity.

### 🧩 Modularity

Design systems with independent, interchangeable components.  
**Supporting standards & practices:**

- **SOLID (SRP, DIP, ISP)** – foundational OO design principles for module separation
    
- **Clean Architecture / Hexagonal Architecture** – enforces modular, layered systems
    
- **IEEE 1471 / ISO/IEC 42010** – _Architecture description standard_ for documenting system modules and interfaces
    
- **Python Packaging Authority (PyPA) Guidelines** – standardized module structure and isolation
    

---

### 🔁 Reusability

Promote reuse of components and logic through abstraction and interfaces.  
**Supporting standards & practices:**

- **DRY (Don’t Repeat Yourself)** – eliminate duplication for higher reuse
    
- **OCP (Open/Closed Principle)** – modules reusable without modification
    
- **Design Patterns (GoF)** – canonical reusable design templates
    
- **IEEE 1517** – _Software reuse processes standard_ (formal ISO/IEEE standard)
    

---

### 🔍 Analysability(Readability)

Ease of understanding, diagnosing, and assessing software.  

**Assessment Questions:**

- **Observability Metrics Requirement**(Every long-running service **MUST expose** a `/metrics` endpoint in **Prometheus text format** to provide internal runtime state.)  

**Supporting standards & practices:**

- **PEP 8** – Python style and readability conventions
    
- **Static analysis tools** (e.g., pylint, flake8, mypy) – enforce analyzable code
    
- **ISO/IEC 24765** – _Software engineering vocabulary_ (standardized terminology)
    
- **Code Review Guidelines (Google, NASA/JPL)** – structured human analysis frameworks

- **SRE Principles(Google Site Reliability Engineering)** – availability engineering

---

### 🔧 Modifiability

Ease of adapting software with minimal side effects. Favor straightforward, minimal solutions; remove unneeded abstraction.
**Supporting standards & practices:**

- **SOLID (OCP, DIP)** – enables safe extension and modification
    
- **Clean Architecture** – decouples layers to support evolution
    
- **Refactoring catalog (Fowler, 1999)** – structured modification patterns
    
- **Version Control Best Practices (GitFlow, trunk-based)** – controlled modification workflow
    

---

### 🧪 Testability

Ease of verifying code correctness and behavior.  
**Supporting standards & practices:**

- **ISO/IEC/IEEE 29119** – _Software testing_ international standard
    
- **ISTQB** – global framework for structured testing methodology
    
- **PyTest / unittest** – Python testing frameworks
    
- **Dependency Injection / Mocking** – improves unit isolation and test coverage
    

---

## 8. 🚀 Portability

Ensure software runs reliably across multiple platforms and environments.

### 🔄 Adaptability

Easily adjust to new or changing environments.  
**Supporting standards & practices:**

- **Twelve-Factor App** – environment-agnostic design
    
- **Cross-platform Frameworks (Qt, Flutter)** – portable UI systems
    
- **Environment Configuration via Env Vars** – runtime flexibility
    

### 📦 Installability

Simple deployment and configuration.  
**Supporting standards & practices:**

- **OCI / Docker Specification** – standardized container builds
    
- **Package Management (PyPI, npm)** – reproducible installation
    
- **Infrastructure as Code (Terraform, Ansible)** – automated setup
    

### 🔁 Replaceability

Ability to swap or upgrade components seamlessly.  
**Supporting standards & practices:**

- **API Versioning & Contracts** – backward-compatible replacement
    
- **Interface Abstraction Patterns** – decoupled component design
    
- **Microservice Architecture** – independent module replacement
