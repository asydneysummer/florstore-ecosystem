# FlorStore — multi-tenant flower shop SaaS

**FlorStore** is a multi-tenant platform for flower shops: a per-shop API and PostgreSQL footprint, staff web cabinet with florist PWA, public marketing and tenant storefront sites, a Telegram Personal relay, and marketplace bouquet feeds (Flowwow, VK, Yandex). Product modules live in **separate private repositories** for deploy boundaries; **this repo is the public ecosystem index** (no application source here).

Built and owned by Alexander ([asydneysummer](https://github.com/asydneysummer)).

**Production:** [florstore.store](https://florstore.store) (platform marketing and merchant cabinet) · example tenant storefront [kupibuket63.ru](https://kupibuket63.ru) with staff at [store.kupibuket63.ru](https://store.kupibuket63.ru) · per shop: `store.{tenant-domain}` and `app.{tenant-domain}`.

> **Access:** module repositories are private — [open an issue](https://github.com/asydneysummer/florstore-ecosystem/issues) or contact Alexander for interview / demo access.

## At a glance

| Layer | What it does end-to-end | Repository |
|-------|-------------------------|------------|
| **Control plane** | Tenant registry, subscriptions, feature flags, entitlement JWT issuance, cross-shop support, shop discovery for desktop clients, optional billing sync from the public site stack | [florstore-api-admin](https://github.com/asydneysummer/florstore-api-admin) *(private — request access)* |
| **Shop backend** | One flower business per VPS: staff JWT apps, deals/POS, warehouse and pricing, finance, public bouquet feeds, storefront CLIENT auth/orders/payments, entitlement refresh to admin | [florstore-api-shop](https://github.com/asydneysummer/florstore-api-shop) *(private — request access)* |
| **Staff UI** | Browser cabinet at `store.*` plus florist PWA at `app.*`: sales, CRM, messenger, warehouse, catalog feeds, finance, analytics, users, settings; offline-capable PWA | [florstore-web](https://github.com/asydneysummer/florstore-web) *(private — request access)* |
| **Platform public web** | `florstore.store`: landing, SEO blog, registration, merchant cabinet, subscription payments, platform admin — connects merchants to admin API and tenant shop APIs | [florstore-site](https://github.com/asydneysummer/florstore-site) *(private — request access)* |
| **Messenger relay** | EU-hosted MTProto gateway: Telegram Personal sessions off the shop VPS; HTTPS proxy and webhooks for the staff messenger inbox | [florstore-telegram-gateway](https://github.com/asydneysummer/florstore-telegram-gateway) *(private — request access)* |
| **Legacy ETL / feeds** | Butonika XLSX import into nomenclature and feed-oriented export helpers for marketplace XML/JSON alongside shop-native feed builders | [florstore-x-butonika](https://github.com/asydneysummer/florstore-x-butonika) *(private — request access)* |

**Tenant storefront example (separate repo):** [kupibuket63](https://github.com/asydneysummer/kupibuket63) — public customer site «Магия Букета» (Samara) on the shop public feed and `/public/*` CLIENT API, with staff on **florstore-web** + **florstore-api-shop** (not a fork of **florstore-site**).

## Screenshots

Production-style UI samples for portfolio review. **No credentials, tokens, or private host hints** in this repo. Full-size PNGs: `docs/screenshots/`.

### Platform site (`florstore-site` → florstore.store)

| | |
|---|---|
| ![FlorStore marketing home](docs/screenshots/florstore-site/home.png) | **Home** — positioning, onboarding modals, lead capture |
| ![FlorStore login](docs/screenshots/florstore-site/login.png) | **Merchant / user login** — entry to cabinet flows |
| ![FlorStore admin login](docs/screenshots/florstore-site/admin-login.png) | **Platform admin login** — separate operator surface |

### Tenant storefront example — KupiBuket63 / «Магия Букета» (Samara)

Customer-facing tenant site (public [kupibuket63](https://github.com/asydneysummer/kupibuket63) repo): catalog from shop XML feed, cart, checkout, CLIENT account modals.

| | |
|---|---|
| ![Tenant home](docs/screenshots/kupibuket63/home.png) | **Home** — brand hero, delivery promises, CTAs |
| ![Catalog](docs/screenshots/kupibuket63/catalog.png) | **Catalog** — categories and product grid |
| ![Bouquet product](docs/screenshots/kupibuket63/bouquet.png) | **Product detail** — bouquet PDP, add to cart |
| ![Cart](docs/screenshots/kupibuket63/cart.png) | **Cart drawer** — line items toward checkout |
| ![Auth or account](docs/screenshots/kupibuket63/auth-or-home.png) | **Client auth / account** — registered customer flows |

---

## Module portfolio

Each module below is a **private** repository; this section summarizes product functionality for employers (no secrets, no production config).

### florstore-api-admin — platform control plane

**Repository:** [florstore-api-admin](https://github.com/asydneysummer/florstore-api-admin) *(private — request access)* · **Deploy:** central platform server · **Stack:** Node.js 22+, TypeScript, **Fastify 5**, Prisma 7, PostgreSQL (`admin` DB only)

**Purpose.** Single admin PostgreSQL and HTTP API for everything that is not tied to one flower shop: tenant records, subscription state, feature entitlements, cross-shop support, shop discovery for desktop clients, and billing events that shop servers must honor.

**Architecture highlights.**

- **Two-API SaaS split:** this service touches only the `admin` database; shop transactional data stays on each tenant’s **florstore-api-shop** server.
- **Three auth zones** with explicit Fastify decorators: admin JWT (`requireAdmin`), shop request HMAC (`requireShopHmac`), and carefully scoped public discovery routes.
- **HMAC request signing** with timestamp window and nonce-style headers; per-shop secrets generated on provisioning, encrypted at rest (AES-256-GCM), shown once on create/rotate.
- **Entitlement JWT** issued to shop API instances; claims encode subscription + feature flags for coordinated rollout.
- **Cross-server support:** tickets/messages in `admin` DB; `shopUserId` is an opaque string from shop `fsusers`, not a foreign key into tenant databases.
- **Optional billing sync** from **florstore-site** when configured (server-to-server, idempotent subscription updates).
- **Operations:** health and DB probes, OpenAPI at `/docs`, rate limiting, helmet, CORS, zod validation in handlers.

**Key engineering work.**

- Stable **EntitlementClaims** contract consumed by every shop API instance.
- Admin session hygiene: bcrypt passwords, opaque refresh tokens stored hashed, rotation on refresh.
- Secure inter-service auth between admin and each shop instance (HMAC verify + anti-replay).
- Billing integration hook from the marketing/storefront stack without embedding payment logic in shop tier.
- Production-oriented surface documented in module `AGENTS.md` (route tables, deploy notes).

**Does not:** touch tenant `fsusers` / `fsdeals` / `fsnomenclature` data.

---

### florstore-api-shop — per-tenant shop backend

**Repository:** [florstore-api-shop](https://github.com/asydneysummer/florstore-api-shop) *(private — request access)* · **Deploy:** dedicated VPS per shop (native install, systemd; example public API host `store.kupibuket63.ru`) · **Stack:** Fastify 5, Prisma 7, **three PostgreSQL databases** per instance

**Purpose.** Authoritative HTTP API for one flower business: staff and app JWT auth, users/RBAC, deals and POS, warehouse and pricing, finance, public bouquet feeds, and storefront **CLIENT** APIs for consumer sites.

**Architecture highlights.**

- **Single-tenant-by-deployment:** one shop per server; three PostgreSQL schemas/clients — `fsusers`, `fsdeals`, `fsnomenclature` — without a `shopId` column (the process *is* the shop).
- **Mandatory entitlement gate** on staff login/refresh: outbound HMAC to **florstore-api-admin**, cached entitlement JWT, documented offline grace and background refresh.
- **Cross-database references** use plain UUID fields (no Prisma cross-DB relations).
- **Public surface** split into unauthenticated feeds (`public-feed`, merchant XML) and authenticated storefront routes (`public-storefront`, payments) with role **`CLIENT`** and storefront app id.
- **Native provisioning** via `install.sh` + systemd (PostgreSQL + Node on shop server).

**Key engineering work.**

- **Resilient licensing:** background entitlement refresh, verified admin JWT cache, heartbeat and support relay to admin.
- **Stock-aware public feeds** merging nomenclature bundles with deal stock for showcase availability rules.
- **Storefront CLIENT domain:** phone auth, order pipeline into deals DB, delivery validation, favorites and bonus programs, optional online pay webhooks.
- **Supply economics:** proportional discount/expense allocation, reconciliation cost rules, settlement and auto-invoice linkage.
- **Pricing engine:** cascade (item → group → category → default), rounding modes, supply-triggered recalc, scheduled fixed-period cleanup.
- **Role-aware route handlers** for pricing, bundles, finance, and deals (see module `AGENTS.md`).

**Product surface (summary).** Nomenclature and bundles; supplies/stock/write-offs; pricing rules; deals/customers/payments/finance; public JSON/XML/Yandex/VK/Flowwow feeds; CLIENT register/login/orders/favorites/bonus; `/public/track/*`; optional Telegram relay and web push when enabled.

---

### florstore-web — staff cabinet and florist PWA

**Repository:** [florstore-web](https://github.com/asydneysummer/florstore-web) *(private — request access)* · **Deploy:** `store.{tenant-domain}` (cabinet) and `app.{tenant-domain}` (`florstore-florist/` PWA) · **Example:** [store.kupibuket63.ru](https://store.kupibuket63.ru)

**Purpose.** Browser-based operations app for flower-shop staff: sales, warehouse, catalog, finance, analytics, users, and settings at `store.{domain}`; companion **florist PWA** at `app.{domain}` for production-floor workflows.

**Architecture highlights.**

- React 19 + Vite + Tailwind; **same-origin `/api`** proxy to tenant **florstore-api-shop** (dev proxy via `VITE_API_PROXY` or local shop port).
- TanStack Query + typed `api.ts` facade (~full shop domain); optional **offline** IndexedDB cache, mutation outbox, sync engine, conflict UI.
- **PWA:** `vite-plugin-pwa`, Workbox strategies, install/update prompts, optional push helpers.
- **Messenger** features call **florstore-telegram-gateway** over HTTPS (no MTProto in the browser bundle).
- **`florstore-florist/`:** separate Vite package sharing auth/entitlement patterns (`VITE_APP_CODE=florstore-florist`).

**Key engineering work.**

- **Multi-tenant packaging:** one codebase deployed per shop at `store.{domain}` (see operator docs in module repo).
- **Offline-first staff UX:** prefetch deals/stock/uploads, background sync, `SyncStatusBar` / `ConflictPanel`.
- **Large single SPA** covering deals, inventory, nomenclature bundles, finance, feeds, and analytics tabs with Playwright E2E coverage.
- **Role-aware admin UX:** `filterNavSections()` hides Settings and Subscription unless **OWNER** or **MANAGER**; messenger channel admin limited in UI.
- **Companion florist app** isolated for assembly-focused workflows without forking shop API contracts.

**Product surface (summary).** Deals kanban and payments; CRM; messenger; supplies/stock/write-offs; showcase and feed panels (site, Flowwow); nomenclature tabs; finance and analytics; users/settings/subscription; florist PWA sub-app.

---

### florstore-site — platform marketing, merchant cabinet, platform admin

**Repository:** [florstore-site](https://github.com/asydneysummer/florstore-site) *(private — request access)* · **Production:** [florstore.store](https://florstore.store) · **Stack:** React 19, Vite, Express API, Prisma 7, PostgreSQL

**Purpose.** Public face of the FlorStore SaaS platform: marketing and SEO blog, shop registration, merchant **cabinet** (catalog artifacts, billing, analytics), and internal **platform admin**. End-customer tenant storefronts (e.g. **kupibuket63**) are separate shop deployments linked via `shop_code`, `api_url`, and cabinet settings — not a fork of this repository.

**Architecture highlights.**

- React SPA (`src/`) + Express API (`server/index.ts`) with JWT in HTTP-only cookies; middleware enforces **USER** vs **ADMIN** on API routes.
- Merchant data modeled per `user_id` in PostgreSQL (Prisma 7); cabinet REST in `server/cabinet-routes.ts`.
- **Billing:** T‑Bank / YooKassa checkout and webhooks; optional push of rent/subscription state to **florstore-api-admin** when `shop_code` and sync configuration are set.
- **Optional staff sync:** cabinet employees mapped to shop roles; proxied CRUD to tenant shop API when `api_url` and sync configuration are set.
- **SEO pipeline:** markdown blog corpus, dynamic sitemap/robots, RSS, server-side article prerender injection, optional indexing workers.
- **Demo mode:** isolated demo database behind unlock gate; Lead Scout optional worker for lead geography.

**Key engineering work.**

- Subscription lifecycle in cabinet: trial, `paid_until`, payment history, internal shop-billing endpoints for platform integration.
- Catalog XML export and bouquet/collection CRUD with uploads for downstream storefront consumers.
- Rate-limited registration/login, production HTML shell caching rules for admin vs public routes.
- Deploy automation (`deploy/deploy.sh`) with Postgres bootstrap documented in module repo.
- Clear boundary vs **florstore-website** legacy JSON store and vs per-tenant customer sites.

**Product surface (summary).** Landing/pricing/onboarding; `/cabinet` settings and analytics; subscription payments; `/admin` clients/leads/content/rent; SEO blog; demo; agent-articles billing helper where enabled.

---

### florstore-telegram-gateway — Telegram Personal MTProto relay

**Repository:** [florstore-telegram-gateway](https://github.com/asydneysummer/florstore-telegram-gateway) *(private — request access)* · **Deploy:** EU VPS (Finland relay — stable regional endpoint for Telegram)

**Purpose.** Keep **Telegram Personal (MTProto)** sessions and connectivity off tenant shop VPS hosts while still enabling messenger workflows in **florstore-web**.

**Architecture highlights.**

- Python MTProto service on EU VPS; staff browser talks **HTTPS** only to the gateway.
- Webhooks / inbox events forwarded toward **florstore-api-shop** for CRM/deals correlation.
- Session material stays on gateway host; shop API bundles do not embed MTProto or long-lived Telegram keys.
- Complements optional env-gated relay hooks in shop API.

**Key engineering work.**

- Stable regional endpoint for Telegram vs shop-origin MTProto.
- Proxy semantics aligned with **florstore-web** `MessengerPanel` (connect/disconnect for OWNER/MANAGER).
- Inbox integration path documented alongside shop API messenger routes.

---

### florstore-x-butonika — Butonika XLSX bridge and feed helpers

**Repository:** [florstore-x-butonika](https://github.com/asydneysummer/florstore-x-butonika) *(private — request access)* · **Deploy:** internal tooling against tenant nomenclature / feed pipelines

**Purpose.** Bridge legacy **Butonika** spreadsheet exports into FlorStore nomenclature and feed-oriented marketplace outputs during migration or hybrid operations.

**Architecture highlights.**

- XLSX ingest pipelines mapped to shop nomenclature shapes (items, bundles, pricing-related fields).
- Batch or API-oriented paths into the three-database shop model (documented in module repo).
- Runs alongside native **florstore-api-shop** public feed builders rather than replacing them.

**Key engineering work.**

- Validation/mapping from legacy tables to FlorStore schemas.
- Feed helper outputs for **Yandex**, **VK**, and **Flowwow** where operations still rely on Butonika exports.
- Operational tooling for shops transitioning to full **florstore-web** catalog management.

---

### Tenant example — KupiBuket63 («Магия Букета», Samara)

**Public storefront repo:** [kupibuket63](https://github.com/asydneysummer/kupibuket63) · **Domains:** [kupibuket63.ru](https://kupibuket63.ru) (customers) · [store.kupibuket63.ru](https://store.kupibuket63.ru) (shop API + **florstore-web** staff cabinet)

**How modules connect**

```
Customer browser → kupibuket63.ru (React SPA)
        │  /feed/bouquets.xml ──► shop API public feed
        │  /api/public/*      ──► florstore-api-shop (CLIENT auth, orders, delivery)
Staff browser    → store.kupibuket63.ru → florstore-web → same shop API (staff JWT)
Platform billing → florstore-site / florstore-api-admin (shop_code, entitlements) — not on the marketing HTML origin
```

**Customer site capabilities:** XML/YML-like catalog poll; bouquet PDP; cart/checkout modals; CLIENT register/login; orders, favorites, bonus; delivery/pickup options from live API; optional online pay; SEO landings, Stories hub, Yandex merchant/services feeds at build time; legal/cookie UX.

**Staff capabilities:** same **florstore-web** navigation as any tenant (deals, customers, messenger, warehouse, catalog feeds including site-feed panel pointing at this vitrine, finance, analytics, users, settings/subscription for OWNER/MANAGER).

Screenshots for this tenant are under `docs/screenshots/kupibuket63/` in **this index repo**.

---

## Features (ecosystem)

- **Per-tenant shop API** with **three PostgreSQL databases** (`fsusers`, `fsdeals`, `fsnomenclature`) — RBAC, deals/POS, catalog/pricing isolated at backup/migration boundaries
- **Admin control plane** — registry, subscriptions, feature flags, HMAC entitlements, support inbox, shop discovery
- **Staff web + florist PWA** — full operational UI with optional offline sync and Telegram messenger via EU gateway
- **Platform site (`florstore.store`)** — marketing, merchant cabinet, payments, platform admin, SEO blog
- **Tenant customer storefronts** — separate repos/deploys (example **kupibuket63**) consuming shop public feeds and CLIENT APIs
- **Public bouquet feeds** — showcase XML/JSON plus Yandex, VK, Flowwow merchant formats (shop-native builders + optional **florstore-x-butonika** ETL)
- **Telegram Personal relay** — **florstore-telegram-gateway** on EU VPS
- **Native per-shop deploy** — one shop ≈ one VPS for **florstore-api-shop** (no Docker requirement on shop tier)
- **Cross-module contracts** — admin ↔ shop entitlements; site ↔ admin billing/onboarding; web ↔ gateway ↔ shop inbox; documented per module repo

## Roles & capabilities

Roles are enforced in **florstore-api-shop** (authoritative) and mirrored in **florstore-web** / tenant storefront UIs. Platform operators use **florstore-api-admin** (and optional desktop **florstore-admin**).

### Platform admin (`florstore-api-admin`)

| Actor | Authentication | Capabilities |
|-------|----------------|--------------|
| **Platform owner / operator** | Admin JWT (`requireAdmin`) | Login/refresh/logout; create/update/delete shops; rotate shop HMAC secrets; read/update/extend subscriptions; manage global and per-shop feature flags; list/reply/update support tickets |
| **Shop API instance (×N tenants)** | Request HMAC (`x-shop-*` headers) | Refresh entitlement JWT; send heartbeat; forward support messages from shop staff |
| **Discovery client** | None (public route) | Resolve shop code → public shop fields + `shopApiUrl` |
| **Billing sync caller** | Server-to-server shared secret (when configured) | Idempotent subscription/rent sync after payment or manual grant from storefront stack |
| **Monitoring** | None | Health and database readiness probes |

### Shop staff (`florstore-api-shop` + `florstore-web`)

Staff roles use enum **`OWNER` · `MANAGER` · `FLORIST` · `CASHIER` · `COURIER` · `READONLY`**. **`CLIENT`** is storefront-only (see below) and excluded from staff user lists.

| Role | Typical use | Capabilities (summary) |
|------|-------------|----------------------|
| **OWNER** | Shop owner | Full staff access; manage all users including other **OWNER** (with safeguards against removing last active owner); pricing mutations; finance account/invoice admin; shop settings and subscription screens in web UI; Telegram messenger setup; customer create/edit; owner-only deal refunds and sensitive supply edits |
| **MANAGER** | Store manager | Same as owner for day-to-day modules; **cannot** assign **OWNER** role; settings + subscription nav visible; users admin without promoting to owner |
| **FLORIST** | Production / sales floor | Deals, supplies, stock reads, operational finance deposits/withdrawals/pay where allowed; nomenclature read/write except pricing mutations blocked at API; **no** settings/subscription nav; customers/users admin read-only in UI; messenger channel admin hidden |
| **CASHIER** | POS counter | Same API pattern as **FLORIST** for payments/deposits; **no** pricing mutations or finance account/invoice CRUD |
| **COURIER** | Delivery | Delivery-focused deal updates; courier/manager assignment on orders; **403** on bundle/pricing/supply mutations and finance writes |
| **READONLY** | Audit / reporting | Read APIs across modules; mutating routes return **403** |

**Web UI navigation gates (`florstore-web`):** **Settings** and **Subscription** sections visible only to **OWNER** and **MANAGER** (`filterNavSections`).

### Storefront CLIENT — guest vs authenticated (`florstore-api-shop` `/public/*`)

Applies to tenant customer sites (e.g. **kupibuket63**) and any vitrine using the storefront app id (e.g. `kupibuket-storefront`).

| Actor | Authentication | Capabilities |
|-------|----------------|--------------|
| **Guest (anonymous)** | None | Browse feed-driven catalog, marketing/stories/legal pages; read public delivery/settings endpoints; add items to **browser-local cart**; no Bearer token |
| **CLIENT (authenticated)** | Storefront JWT after register/login | Profile; place orders; order history; favorites sync; bonus ledger; delivery defaults; online payment flows when enabled for tenant |
| **Staff on customer domain** | — | **Not supported** on tenant marketing origin — staff use **florstore-web** at `store.{domain}` |

API gate for CLIENT routes: authenticated user must have role **`CLIENT`** and correct storefront **app** id (not staff JWT).

### Public flows — platform site vs tenant vitrine

| Surface | Actor | Flow |
|---------|-------|------|
| **florstore.store** marketing | Anonymous visitor | Landing, blog, legal, sitemap; consult/lead forms; demo unlock when demo mode enabled |
| **florstore.store** `/cabinet` | Registered merchant (**USER**) | JWT cookie session; edit shop connection settings; manage merchant-side catalog artifacts; subscription payments; analytics; optional employee records synced to shop API |
| **florstore.store** `/admin` | Platform **ADMIN** | Operator dashboard: clients, shop codes, rent, leads, content tooling, link analytics |
| **Tenant site** (e.g. kupibuket63.ru) | Guest | Catalog/checkout UX without account |
| **Tenant site** | **CLIENT** | Account modals: orders, bonus, favorites |

Cabinet employee roles (`director`, `marketer`, `manager`, `florist`, `hybrid`) map to shop staff roles when shop API staff sync is enabled (**florstore-site**).

## Architecture (high level)

```
                         Platform (single deploy)
                         ┌──────────────────────────┐
  florstore-admin        │   florstore-api-admin    │
  (desktop, optional)    │   admin PostgreSQL       │
                         └────────────┬─────────────┘
                                      │ HMAC-signed entitlements + heartbeat
                                      ▼
              Per tenant (1 VPS ≈ 1 shop, e.g. kupibuket63)
              ┌─────────────────────────────────────────────┐
              │ florstore-api-shop                          │
              │   fsusers │ fsdeals │ fsnomenclature (PG×3) │
              └─────┬───────────────────┬───────────────────┘
                    │                   │
         florstore-web (+ florist PWA) │ public bouquet feeds
         store.* / app.*               │ (Yandex / VK / Flowwow)
                    │                   │
     Tenant customer site (kupibuket63)─┘  XML/JSON feed + /public/*
                    │
              florstore-site (florstore.store) ── billing / onboarding ──► admin API

              EU relay VPS: florstore-telegram-gateway ◄──MTProto──► Telegram
                                    ▲
                                    │ HTTPS + inbox webhooks
                              florstore-web / shop API

              florstore-x-butonika: XLSX ETL ──► nomenclature / feed builders
```

## Cross-module integrations

| From | To | Mechanism |
|------|-----|-----------|
| **florstore-api-shop** | **florstore-api-admin** | Outbound HTTPS + **HMAC** — entitlement refresh, heartbeat, support relay |
| **florstore-api-admin** | **florstore-api-shop** | Entitlement JWT claims consumed on shop login and feature gates |
| **florstore-site** | **florstore-api-admin** | Authenticated **billing / onboarding sync** when merchant `shop_code` is configured |
| **florstore-site** | **florstore-api-shop** | Optional **staff sync** proxy when merchant `api_url` and sync configuration are set |
| **florstore-web** | **florstore-telegram-gateway** | HTTPS messenger proxy; no MTProto in browser |
| **florstore-telegram-gateway** | **florstore-api-shop** | Inbox / webhook events for messenger panel |
| **florstore-x-butonika** | **florstore-api-shop** nomenclature | XLSX ETL and feed-oriented exports (Yandex / VK / Flowwow pipelines) |
| **kupibuket63** (tenant site) | **florstore-api-shop** | nginx-proxied `/feed/*` and `/api/public/*` to tenant shop host |

## Tech stack

| Area | Technologies |
|------|----------------|
| **Shop & admin APIs** | Node.js 22+, TypeScript (ESM), **Fastify 5**, Prisma 7, PostgreSQL, Zod, JWT/HMAC |
| **Platform site** | React 19, React Router, Vite, Tailwind, Express, Prisma, payment provider integrations |
| **Staff & tenant SPAs** | React 19, Vite, Tailwind 4, TanStack Query, PWA (Workbox) |
| **Telegram gateway** | Python MTProto service on EU VPS, HTTPS façade |
| **Integrations** | XLSX ETL (**florstore-x-butonika**), Yandex/VK/Flowwow feed XML/JSON |
| **Deploy** | Platform admin cluster; **one VPS per tenant** shop API; EU relay for Telegram; static/nginx for tenant storefronts |

## Related portfolio

| Ecosystem | Index |
|-----------|-------|
| **HopeClass** (EdTech) | [hopeclass-ecosystem](https://github.com/asydneysummer/hopeclass-ecosystem) |
| **Buketov** (beauty salon CRM) | [buketov-app](https://github.com/asydneysummer/buketov-app) *(private — request access)* |

### Module repositories (all private — request access)

| Module | Repository | Production surface |
|--------|------------|-------------------|
| Admin API | [florstore-api-admin](https://github.com/asydneysummer/florstore-api-admin) | Platform server |
| Shop API | [florstore-api-shop](https://github.com/asydneysummer/florstore-api-shop) | Per-tenant VPS |
| Web + florist | [florstore-web](https://github.com/asydneysummer/florstore-web) | `store.{domain}`, `app.{domain}` |
| Platform site | [florstore-site](https://github.com/asydneysummer/florstore-site) | [florstore.store](https://florstore.store) |
| Telegram gateway | [florstore-telegram-gateway](https://github.com/asydneysummer/florstore-telegram-gateway) | EU MTProto relay |
| Butonika / feeds | [florstore-x-butonika](https://github.com/asydneysummer/florstore-x-butonika) | Internal ETL + feeds |
| Tenant storefront example | [kupibuket63](https://github.com/asydneysummer/kupibuket63) | [kupibuket63.ru](https://kupibuket63.ru) |

*Public portfolio index only — no application source, secrets, or production configuration in this repository.*

---

# FlorStore — экосистема SaaS для цветочных магазинов

**FlorStore** — мультитенантная платформа для цветочных магазинов: API и три PostgreSQL на магазин, веб-кабинет с PWA флориста, маркетинг платформы и отдельные витрины tenant-ов, Telegram-реле MTProto и фиды букетов для маркетплейсов (Flowwow, VK, Яндекс). Модули продукта — в **отдельных приватных репозиториях**; **этот репозиторий — публичный индекс экосистемы** (без исходников приложений).

Автор: Александр ([asydneysummer](https://github.com/asydneysummer)).

**Продакшен:** [florstore.store](https://florstore.store) (маркетинг и кабинет арендатора) · пример витрины [kupibuket63.ru](https://kupibuket63.ru), учётка [store.kupibuket63.ru](https://store.kupibuket63.ru) · на каждый магазин: `store.{домен}` и `app.{домен}`.

> **Доступ:** репозитории модулей приватные — [issue](https://github.com/asydneysummer/florstore-ecosystem/issues) или контакт с Александром для доступа к коду / демо.

## Обзор (At a glance)

| Слой | Что делает end-to-end | Репозиторий |
|------|------------------------|-------------|
| **Control plane** | Реестр магазинов, подписки, feature flags, выдача entitlement JWT, сквозная поддержка, discovery для desktop, опциональная синхронизация биллинга с публичного сайта | [florstore-api-admin](https://github.com/asydneysummer/florstore-api-admin) *(приватный — запросите доступ)* |
| **Backend магазина** | Один бизнес на VPS: JWT приложений персонала, сделки/POS, склад и цены, финансы, публичные фиды букетов, CLIENT API витрины, refresh entitlements в admin | [florstore-api-shop](https://github.com/asydneysummer/florstore-api-shop) *(приватный — запросите доступ)* |
| **UI персонала** | Кабинет `store.*` и PWA флориста `app.*`: продажи, CRM, мессенджер, склад, фиды, финансы, аналитика, пользователи; офлайн-PWA | [florstore-web](https://github.com/asydneysummer/florstore-web) *(приватный — запросите доступ)* |
| **Публичный web платформы** | `florstore.store`: лендинг, SEO-блог, регистрация, кабинет мерчанта, оплата подписки, админка платформы | [florstore-site](https://github.com/asydneysummer/florstore-site) *(приватный — запросите доступ)* |
| **Реле мессенджера** | MTProto на EU VPS: Telegram Personal вне shop-сервера; HTTPS и webhooks для inbox в кабинете | [florstore-telegram-gateway](https://github.com/asydneysummer/florstore-telegram-gateway) *(приватный — запросите доступ)* |
| **ETL / фиды** | Импорт XLSX Butonika в номенклатуру и вспомогательные экспорты для маркетплейсов | [florstore-x-butonika](https://github.com/asydneysummer/florstore-x-butonika) *(приватный — запросите доступ)* |

**Пример витрины tenant (отдельный репозиторий):** [kupibuket63](https://github.com/asydneysummer/kupibuket63) — сайт «Магия Букета» (Самара) на public feed и `/public/*` shop API, персонал в **florstore-web** + **florstore-api-shop** (это не форк **florstore-site**).

## Скриншоты

Примеры UI для портфолио. **Без учётных данных, токенов и подсказок с приватными хостами.** PNG: `docs/screenshots/`.

### Сайт платформы (`florstore-site` → florstore.store)

| | |
|---|---|
| ![Главная FlorStore](docs/screenshots/florstore-site/home.png) | **Главная** — позиционирование, онбординг, лиды |
| ![Логин](docs/screenshots/florstore-site/login.png) | **Вход мерчанта / пользователя** |
| ![Admin login](docs/screenshots/florstore-site/admin-login.png) | **Вход оператора платформы** |

### Пример витрины tenant — KupiBuket63 / «Магия Букета» (Самара)

| | |
|---|---|
| ![Главная tenant](docs/screenshots/kupibuket63/home.png) | **Главная** — бренд, доставка, CTA |
| ![Каталог](docs/screenshots/kupibuket63/catalog.png) | **Каталог** |
| ![Букет](docs/screenshots/kupibuket63/bouquet.png) | **Карточка букета** |
| ![Корзина](docs/screenshots/kupibuket63/cart.png) | **Корзина** |
| ![Авторизация](docs/screenshots/kupibuket63/auth-or-home.png) | **Клиент: вход / ЛК** |

---

## Портфолио модулей

Краткое описание продукта по каждому **приватному** модулю (без секретов и prod-конфигурации).

### florstore-api-admin — control plane платформы

**Репозиторий:** [florstore-api-admin](https://github.com/asydneysummer/florstore-api-admin) *(приватный — запросите доступ)* · **Деплой:** сервер платформы · **Стек:** Node.js 22+, TypeScript, **Fastify 5**, Prisma 7, PostgreSQL (только БД `admin`)

**Назначение.** Единая admin PostgreSQL и HTTP API для всего, что не привязано к одному цветочному магазину: записи tenant-ов, состояние подписки, feature entitlements, сквозная поддержка, discovery магазинов для desktop-клиентов и события биллинга, которые обязаны учитывать shop-серверы.

**Архитектура.**

- **Два API SaaS:** сервис работает только с БД `admin`; транзакционные данные магазина остаются на **florstore-api-shop** каждого tenant-а.
- **Три зоны auth** с декораторами Fastify: admin JWT (`requireAdmin`), HMAC запросов shop (`requireShopHmac`), ограниченные публичные discovery-маршруты.
- **Подпись HMAC** с окном timestamp и anti-replay; секреты на магазин при провижининге, шифрование at rest (AES-256-GCM), однократный показ при создании/ротации.
- **Entitlement JWT** для экземпляров shop API; claims кодируют подписку и feature flags.
- **Сквозная поддержка:** тикеты/сообщения в БД admin; `shopUserId` — opaque string из `fsusers` tenant-а, не FK в tenant БД.
- **Опциональный billing sync** от **florstore-site** при настройке (server-to-server, идемпотентно).
- **Эксплуатация:** health/db probes, OpenAPI `/docs`, rate limit, helmet, CORS, zod в handlers.

**Ключевая инженерная работа.**

- Стабильный контракт **EntitlementClaims** для всех shop API.
- Гигиена admin-сессий: bcrypt, refresh-токены hashed, ротация при refresh.
- Межсервисная безопасность admin ↔ shop (HMAC + anti-replay).
- Hook интеграции биллинга с маркетингового стека без payment-логики на shop tier.
- Production surface и таблицы маршрутов в `AGENTS.md` модуля.

**Не делает:** не читает и не пишет tenant БД `fsusers` / `fsdeals` / `fsnomenclature`.

---

### florstore-api-shop — backend одного магазина

**Репозиторий:** [florstore-api-shop](https://github.com/asydneysummer/florstore-api-shop) *(приватный — запросите доступ)* · **Деплой:** отдельный VPS на магазин (native install, systemd; пример public API: `store.kupibuket63.ru`) · **Стек:** Fastify 5, Prisma 7, **три PostgreSQL** на инстанс

**Назначение.** Авторитетный HTTP API одного цветочного бизнеса: JWT приложений и персонала, пользователи/RBAC, сделки и POS, склад и цены, финансы, публичные фиды букетов и **CLIENT** API витрины для сайтов покупателей.

**Архитектура.**

- **Один tenant = один деплой:** три клиента Prisma/PostgreSQL — `fsusers`, `fsdeals`, `fsnomenclature` — без колонки `shopId` (процесс *есть* магазин).
- **Обязательный entitlement gate** при login/refresh staff: исходящий HMAC к **florstore-api-admin**, кэш entitlement JWT, документированный offline grace и фоновый refresh.
- **Ссылки между БД** — plain UUID (без cross-DB relations в Prisma).
- **Публичный контур:** фиды без auth и storefront-маршруты с ролью **`CLIENT`** и app id витрины.
- **Native provisioning:** `install.sh` + systemd на shop-сервере.

**Ключевая инженерная работа.**

- **Устойчивое лицензирование:** refresh entitlement, кэш JWT admin, heartbeat и relay в поддержку.
- **Stock-aware public feeds:** комплекты номенклатуры + остатки сделок для правил витрины.
- **Домен CLIENT витрины:** phone auth, заказы в deals DB, доставка, избранное, бонусы, опционально online pay webhooks.
- **Экономика поставок:** аллокация скидок/расходов, акты сверки, settlements, auto-invoice.
- **Движок цен:** каскад item → group → category → default, округления, пересчёт от поставок, scheduler fixed-period.
- **RBAC в handlers** pricing/bundles/finance/deals (см. `AGENTS.md` модуля).

**Продуктовый контур (сводка).** Номенклатура и комплекты; поставки/остатки/списания; pricing; сделки/клиенты/оплаты/финансы; public JSON/XML/Yandex/VK/Flowwow; CLIENT register/login/orders/favorites/bonus; `/public/track/*`; опционально Telegram relay и web push.

---

### florstore-web — кабинет персонала и PWA флориста

**Репозиторий:** [florstore-web](https://github.com/asydneysummer/florstore-web) *(приватный — запросите доступ)* · **Деплой:** `store.{домен}` (кабинет) и `app.{домен}` (PWA `florstore-florist/`) · **Пример:** [store.kupibuket63.ru](https://store.kupibuket63.ru)

**Назначение.** Операционное веб-приложение персонала цветочного магазина: продажи, склад, каталог, финансы, аналитика, пользователи и настройки на `store.{домен}`; companion **PWA флориста** на `app.{домен}` для цеха сборки.

**Архитектура.**

- React 19 + Vite + Tailwind; **same-origin `/api`** → **florstore-api-shop** tenant-а (dev proxy через `VITE_API_PROXY` или локальный shop port).
- TanStack Query + типизированный `api.ts`; опциональный **offline** IndexedDB, outbox мутаций, sync engine, UI конфликтов.
- **PWA:** `vite-plugin-pwa`, Workbox, install/update prompts, опциональные push helpers.
- **Мессенджер** через **florstore-telegram-gateway** по HTTPS (без MTProto в браузере).
- **`florstore-florist/`:** отдельный Vite-пакет с теми же auth/entitlement паттернами (`VITE_APP_CODE=florstore-florist`).

**Ключевая инженерная работа.**

- **Multi-tenant packaging:** одна кодовая база, деплой на `store.{домен}` каждого клиента (см. эксплуатационные docs в репозитории).
- **Offline-first UX:** prefetch deals/stock/uploads, фоновая синхронизация, `SyncStatusBar` / `ConflictPanel`.
- **Большой SPA** с deals, складом, номенклатурой, финансами, фидами и аналитикой; Playwright E2E.
- **Role-aware UI:** `filterNavSections()` скрывает Настройки и Подписку без **OWNER**/**MANAGER**; admin мессенджера ограничен в UI.
- **Companion florist app** без дублирования контрактов shop API.

**Продуктовый контур (сводка).** Kanban сделок и оплаты; CRM; мессенджер; supplies/stock/write-offs; витрина и фиды (site, Flowwow); вкладки номенклатуры; финансы и аналитика; users/settings/subscription; sub-app флориста.

---

### florstore-site — маркетинг, кабинет мерчанта, админка платформы

**Репозиторий:** [florstore-site](https://github.com/asydneysummer/florstore-site) *(приватный — запросите доступ)* · **Прод:** [florstore.store](https://florstore.store) · **Стек:** React 19, Vite, Express API, Prisma 7, PostgreSQL

**Назначение.** Публичное лицо SaaS FlorStore: маркетинг и SEO-блог, регистрация магазинов, **кабинет** мерчанта (артефакты каталога, биллинг, аналитика) и **админка** платформы. Клиентские витрины tenant-ов (например **kupibuket63**) — **отдельные деплои**, связанные через `shop_code`, `api_url` и настройки кабинета; это не форк данного репозитория.

**Архитектура.**

- React SPA (`src/`) + Express API (`server/index.ts`); JWT в HTTP-only cookies; middleware разделяет **USER** и **ADMIN** на API.
- Данные мерчанта per `user_id` в PostgreSQL (Prisma 7); cabinet REST в `server/cabinet-routes.ts`.
- **Биллинг:** checkout T‑Bank / YooKassa, webhooks; опциональный push аренды/подписки в **florstore-api-admin** при `shop_code` и настроенной синхронизации.
- **Опциональный staff sync:** сотрудники кабинета → роли shop API; proxy CRUD к tenant shop API при `api_url` и конфигурации sync.
- **SEO:** markdown-блог, sitemap/robots, RSS, prerender статей, опциональные indexing workers.
- **Demo:** изолированная demo-БД за unlock gate; Lead Scout worker для лидов (опционально).

**Ключевая инженерная работа.**

- Жизненный цикл подписки в кабинете: trial, `paid_until`, история платежей, internal shop-billing endpoints.
- XML-экспорт каталога и CRUD букетов/подборок с uploads для downstream витрин.
- Rate limit register/login, правила кэша HTML shell для admin vs public.
- Deploy automation (`deploy/deploy.sh`) и bootstrap Postgres в документации модуля.
- Явная граница vs legacy **florstore-website** (JSON store) и vs per-tenant customer sites.

**Продуктовый контур (сводка).** Landing/pricing/onboarding; `/cabinet` settings и analytics; оплаты подписки; `/admin` clients/leads/content/rent; SEO blog; demo; agent-articles billing helper при включении.

**Граница:** витрины покупателей (например [kupibuket63.ru](https://kupibuket63.ru)) потребляют **florstore-api-shop** public feeds и CLIENT API; этот модуль подключает мерчанта к платформе.

---

### florstore-telegram-gateway — MTProto-реле Telegram Personal

**Репозиторий:** [florstore-telegram-gateway](https://github.com/asydneysummer/florstore-telegram-gateway) *(приватный — запросите доступ)* · **Деплой:** EU VPS (Финляндия — стабильная региональная точка для Telegram)

**Назначение.** Держать сессии и connectivity **Telegram Personal (MTProto)** вне VPS shop-tenant-ов, сохраняя мессенджер в **florstore-web**.

**Архитектура.**

- Python MTProto на EU VPS; браузер staff обращается только по **HTTPS** к gateway.
- Webhooks / inbox → **florstore-api-shop** для связи переписки с CRM/сделками.
- Session material на хосте gateway; shop API и SPA не содержат MTProto и долгоживущие ключи Telegram.
- Дополняет опциональные env-gated relay hooks в shop API.

**Ключевая инженерная работа.**

- Стабильная региональная точка для Telegram vs MTProto с origin shop-сервера.
- Proxy-семантика под **MessengerPanel** в **florstore-web** (connect/disconnect для OWNER/MANAGER).
- Inbox path документирован вместе с messenger routes shop API.

---

### florstore-x-butonika — мост Butonika XLSX и фиды

**Репозиторий:** [florstore-x-butonika](https://github.com/asydneysummer/florstore-x-butonika) *(приватный — запросите доступ)* · **Деплой:** внутренние ETL-задачи к номенклатуре tenant-а

**Назначение.** Мост legacy-экспортов **Butonika** (XLSX) в номенклатуру FlorStore и feed-oriented выходы маркетплейсов при миграции или гибридной эксплуатации.

**Архитектура.**

- Ingest XLSX → shapes номенклатуры FlorStore (items, bundles, pricing-related fields).
- Batch или API-пути в три shop-БД (документированы в репозитории модуля).
- Работает **рядом** с native public feed builders **florstore-api-shop**, не заменяя их.

**Ключевая инженерная работа.**

- **Валидация и маппинг** legacy-таблиц в три shop-БД (через shop API или batch-пути, документированные в репозитории модуля).
- Feed helper outputs для **Yandex**, **VK**, **Flowwow** при операциях на Butonika-экспортах.
- Ops-инструменты для магазинов в переходе на полный каталог в **florstore-web**.

---

### Пример tenant — KupiBuket63 («Магия Букета», Самара)

**Репозиторий витрины:** [kupibuket63](https://github.com/asydneysummer/kupibuket63) · **Домены:** [kupibuket63.ru](https://kupibuket63.ru) · [store.kupibuket63.ru](https://store.kupibuket63.ru)

**Связь модулей**

```
Покупатель → kupibuket63.ru (React SPA)
        │  /feed/bouquets.xml → public feed shop API
        │  /api/public/*      → florstore-api-shop (CLIENT, заказы, доставка)
Персонал   → store.kupibuket63.ru → florstore-web → тот же shop API (staff JWT)
Биллинг платформы → florstore-site / florstore-api-admin (shop_code, entitlements)
```

**Сайт для покупателя:** опрос XML-каталога; PDP; корзина/checkout; CLIENT auth; заказы, избранное, бонусы; доставка/самовывоз; SEO, Stories, merchant feeds; legal/cookies.

**Учётка:** полная навигация **florstore-web** (сделки, клиенты, мессенджер, склад, фиды «Сайт», финансы, аналитика, пользователи, настройки/подписка для OWNER/MANAGER).

Скриншоты: `docs/screenshots/kupibuket63/` в этом индекс-репозитории.

---

## Возможности (экосистема)

- **Shop API на tenant** и **три PostgreSQL** (`fsusers`, `fsdeals`, `fsnomenclature`) — RBAC, сделки/POS, каталог/цены
- **Admin control plane** — реестр, подписки, flags, HMAC entitlements, поддержка, discovery
- **Кабинет + PWA флориста** — операционный UI, офлайн, Telegram через EU gateway
- **Сайт платформы** — маркетинг, кабинет мерчанта, оплаты, админка, SEO-блог
- **Витрины tenant-ов** — отдельные репозитории (пример **kupibuket63**) на public feed и CLIENT API
- **Публичные фиды букетов** — showcase XML/JSON, Yandex, VK, Flowwow (+ optional **florstore-x-butonika**)
- **Telegram Personal** — **florstore-telegram-gateway** на EU VPS
- **Деплой shop tier** — один VPS ≈ один магазин для **florstore-api-shop**
- **Контракты между модулями** — admin ↔ shop; site ↔ admin; web ↔ gateway ↔ shop

## Роли и возможности

Роли авторитетно задаются в **florstore-api-shop**; UI **florstore-web** и витрины следуют им. Операторы платформы — **florstore-api-admin** (+ опционально desktop **florstore-admin**).

### Платформенный admin (`florstore-api-admin`)

| Актор | Аутентификация | Возможности |
|-------|----------------|-------------|
| **Владелец / оператор платформы** | Admin JWT (`requireAdmin`) | Login/refresh/logout; создание/изменение/удаление магазинов; ротация HMAC-секретов shop; чтение/изменение/продление подписок; глобальные и per-shop feature flags; список/ответ/обновление тикетов поддержки |
| **Экземпляр shop API (×N tenant-ов)** | HMAC запросов (`x-shop-*` headers) | Refresh entitlement JWT; heartbeat; пересылка сообщений поддержки от staff магазина |
| **Discovery-клиент** | Нет (публичный route) | Код магазина → публичные поля + `shopApiUrl` |
| **Billing sync caller** | Server-to-server shared secret (если настроено) | Идемпотентная синхронизация аренды/подписки после оплаты или ручного grant со стека storefront |
| **Мониторинг** | Нет | Health и готовность БД |

### Персонал магазина (`florstore-api-shop` + `florstore-web`)

Роли staff: enum **`OWNER` · `MANAGER` · `FLORIST` · `CASHIER` · `COURIER` · `READONLY`**. **`CLIENT`** — только витрина (ниже) и исключён из staff user lists.

| Роль | Типичная роль | Возможности (сводка) |
|------|---------------|----------------------|
| **OWNER** | Владелец магазина | Полный staff-доступ; управление всеми пользователями включая **OWNER** (с защитой последнего активного owner); мутации pricing; админ фин. счетов/инвойсов; экраны настроек и подписки в web UI; настройка Telegram-мессенджера; create/edit клиентов; owner-only возвраты по сделкам и чувствительные правки поставок |
| **MANAGER** | Управляющий | Как owner в операционных модулях; **не** может назначить роль **OWNER**; nav settings + subscription; users admin без повышения до owner |
| **FLORIST** | Цех / зал | Сделки, поставки, чтение остатков, операционные deposit/withdraw/pay где разрешено; номенклатура без мутаций pricing на API; **нет** nav settings/subscription; customers/users admin read-only в UI; скрыто admin каналов мессенджера |
| **CASHIER** | Касса | Тот же API-паттерн, что у **FLORIST** для оплат/deposits; **нет** мутаций pricing и CRUD фин. счетов/инвойсов |
| **COURIER** | Доставка | Обновления сделок по доставке; назначение courier/manager на заказы; **403** на мутации bundle/pricing/supply и фин. записи |
| **READONLY** | Аудит / отчёты | Read API по модулям; mutating routes → **403** |

**Навигация web UI (`florstore-web`):** разделы **Настройки** и **Подписка** видны только **OWNER** и **MANAGER** (`filterNavSections`).

### CLIENT витрины — гость и авторизованный клиент (`florstore-api-shop` `/public/*`)

Для tenant customer sites (например **kupibuket63**) и любой витрины с app id storefront (например `kupibuket-storefront`).

| Актор | Аутентификация | Возможности |
|-------|----------------|-------------|
| **Гость (anonymous)** | Нет | Каталог по фиду, marketing/stories/legal; публичные delivery/settings endpoints; корзина в **local storage браузера**; без Bearer token |
| **CLIENT (authenticated)** | Storefront JWT после register/login | Профиль; оформление заказов; история заказов; sync избранного; бонусный ledger; defaults доставки; online pay при включении для tenant |
| **Персонал на customer-домене** | — | **Не поддерживается** на marketing origin tenant — staff используют **florstore-web** на `store.{домен}` |

Gate CLIENT-маршрутов: у authenticated user роль **`CLIENT`** и корректный storefront **app** id (не staff JWT).

### Публичные сценарии — сайт платформы vs витрина tenant

| Поверхность | Актор | Сценарий |
|-------------|-------|----------|
| **florstore.store** marketing | Anonymous visitor | Landing, blog, legal, sitemap; consult/lead forms; demo unlock при включённом demo mode |
| **florstore.store** `/cabinet` | Зарегистрированный мерчант (**USER**) | JWT cookie session; настройки подключения к shop API; merchant-side catalog artifacts; оплаты подписки; analytics; опционально sync записей сотрудников в shop API |
| **florstore.store** `/admin` | **ADMIN** платформы | Operator dashboard: clients, shop codes, rent, leads, content tooling, link analytics |
| **Сайт tenant** (например kupibuket63.ru) | Guest | Catalog/checkout UX без аккаунта |
| **Сайт tenant** | **CLIENT** | Account modals: orders, bonus, favorites |

Роли сотрудников кабинета (`director`, `marketer`, `manager`, `florist`, `hybrid`) мапятся на shop staff roles при включённом shop API staff sync (**florstore-site**).

## Архитектура (обзор)

```
                         Сервер платформы
                         ┌──────────────────────────┐
  florstore-admin        │   florstore-api-admin    │
  (desktop)              │   admin PostgreSQL       │
                         └────────────┬─────────────┘
                                      │ HMAC entitlements + heartbeat
                                      ▼
              На tenant (1 VPS ≈ 1 магазин, напр. kupibuket63)
              ┌─────────────────────────────────────────────┐
              │ florstore-api-shop                          │
              │   fsusers │ fsdeals │ fsnomenclature (PG×3) │
              └─────┬───────────────────┬───────────────────┘
                    │                   │
         florstore-web (+ PWA флорист)  │ public feeds
         store.* / app.*                │ (Yandex / VK / Flowwow)
                    │                   │
     Сайт покупателя (kupibuket63)──────┘  feed + /public/*
                    │
              florstore-site (florstore.store) ── billing / onboarding ──► admin API

              EU VPS: florstore-telegram-gateway ◄──MTProto──► Telegram
                                    ▲
                                    │ HTTPS + webhooks
                              florstore-web / shop API

              florstore-x-butonika: XLSX ETL ──► номенклатура / фиды
```

## Межмодульные интеграции

| Откуда | Куда | Механизм |
|--------|------|----------|
| **florstore-api-shop** | **florstore-api-admin** | HTTPS + **HMAC** — entitlement, heartbeat, поддержка |
| **florstore-api-admin** | **florstore-api-shop** | Entitlement JWT на login и feature gates |
| **florstore-site** | **florstore-api-admin** | **Billing / onboarding sync** при настроенном `shop_code` |
| **florstore-site** | **florstore-api-shop** | Опциональный **staff sync** при `api_url` и конфигурации |
| **florstore-web** | **florstore-telegram-gateway** | HTTPS messenger proxy |
| **florstore-telegram-gateway** | **florstore-api-shop** | Inbox / webhooks |
| **florstore-x-butonika** | номенклатура shop API | XLSX ETL и экспорт фидов |
| **kupibuket63** | **florstore-api-shop** | nginx: `/feed/*`, `/api/public/*` на shop host |

## Стек технологий

| Область | Технологии |
|---------|------------|
| **Shop & admin API** | Node.js 22+, TypeScript, **Fastify 5**, Prisma 7, PostgreSQL, Zod, JWT/HMAC |
| **Сайт платформы** | React 19, Vite, Tailwind, Express, Prisma, платёжные провайдеры |
| **Staff & tenant SPA** | React 19, Vite, Tailwind 4, TanStack Query, PWA (Workbox) |
| **Telegram gateway** | Python MTProto на EU VPS, HTTPS façade |
| **Интеграции** | XLSX (**florstore-x-butonika**), Yandex/VK/Flowwow |
| **Деплой** | Кластер admin; **VPS на tenant**; EU relay; nginx для витрин |

## Связанное портфолио

| Экосистема | Индекс |
|------------|--------|
| **HopeClass** (EdTech) | [hopeclass-ecosystem](https://github.com/asydneysummer/hopeclass-ecosystem) |
| **Buketov** (CRM салона) | [buketov-app](https://github.com/asydneysummer/buketov-app) *(приватный — запросите доступ)* |

### Репозитории модулей (приватные — запросите доступ)

| Модуль | Репозиторий | Прод |
|--------|-------------|------|
| Admin API | [florstore-api-admin](https://github.com/asydneysummer/florstore-api-admin) | Сервер платформы |
| Shop API | [florstore-api-shop](https://github.com/asydneysummer/florstore-api-shop) | VPS tenant |
| Web + флорист | [florstore-web](https://github.com/asydneysummer/florstore-web) | `store.*`, `app.*` |
| Сайт платформы | [florstore-site](https://github.com/asydneysummer/florstore-site) | [florstore.store](https://florstore.store) |
| Telegram gateway | [florstore-telegram-gateway](https://github.com/asydneysummer/florstore-telegram-gateway) | EU relay |
| Butonika / фиды | [florstore-x-butonika](https://github.com/asydneysummer/florstore-x-butonika) | ETL |
| Пример витрины | [kupibuket63](https://github.com/asydneysummer/kupibuket63) | [kupibuket63.ru](https://kupibuket63.ru) |

*Публичный индекс — без исходников приложений, секретов и production-конфигурации.*
