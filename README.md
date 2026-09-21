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

**What it does end-to-end**

- Maintains the **shop (tenant) registry**: shop codes, public metadata, encrypted per-shop HMAC secrets (one-time reveal on create/rotate).
- **Subscription lifecycle** per shop: status, plan, period end, manual extend, alignment with storefront billing when sync is enabled.
- **Feature flags** globally and per shop; entitlement JWT claims combine subscription state + flags for shop API instances.
- **Cross-shop support inbox**: tickets and messages from shop staff (opaque `shopUserId` from tenant `fsusers`, not a cross-database FK); admin replies from platform tools.
- **Shop discovery** for internet/desktop clients: resolve shop code → public fields + `shopApiUrl` without exposing PII.
- **Shop API face** (HMAC): entitlement refresh, heartbeat, forward support messages from shop users.
- **Optional billing sync** from **florstore-site** when configured (server-to-server, idempotent subscription updates).
- **Operations:** health and DB probes, OpenAPI at `/docs`, rate limiting, helmet, CORS, zod validation.

**Does not:** touch tenant `fsusers` / `fsdeals` / `fsnomenclature` data — strict two-API SaaS split.

---

### florstore-api-shop — per-tenant shop backend

**Repository:** [florstore-api-shop](https://github.com/asydneysummer/florstore-api-shop) *(private — request access)* · **Deploy:** dedicated VPS per shop (native install, systemd; example public API host `store.kupibuket63.ru`) · **Stack:** Fastify 5, Prisma 7, **three PostgreSQL databases** per instance

**What it does end-to-end**

- **Identity & staff apps:** JWT auth for `florstore`, `florstore-community`, `florstore-web`, `florstore-florist`; user CRUD with roles; invitations and refresh-token rotation; **mandatory entitlement gate** on login/refresh (HMAC to admin, cached JWT, documented offline grace).
- **Nomenclature:** items, categories, suppliers, contractors, bundles (showcase/catalog), image uploads, collections.
- **Warehouse:** supplies, stock, write-offs, reconciliation acts, weighted-average cost, settlements, service supplies.
- **Pricing:** manual / markup / fixed-period rules, category markups, supply-triggered recalc, hourly scheduler for expired fixed prices.
- **Deals & CRM:** deal pipeline, customers, payments; courier assignment; finance module (accounts, transfers, supplier/contractor invoices) with role-gated mutations.
- **Public feeds (no auth):** JSON/XML showcase bouquet feed, Yandex and VK merchant XML, Flowwow feeds, collection feeds — **stock-aware** eligibility for showcase bundles.
- **Storefront CLIENT API** (`app: kupibuket-storefront`, role `CLIENT`): register/login, profile, delivery options, orders, favorites, bonus ledger; optional online payment webhooks when enabled.
- **Analytics & comms:** public site tracking endpoints; optional Telegram relay hooks and web push (feature-gated by env).
- **Licensing ops:** background entitlement refresh, heartbeat to admin, support message relay outbound.

**Data model:** `fsusers`, `fsdeals`, `fsnomenclature` — no `shopId` column; the process **is** one shop. Cross-DB references are plain UUIDs.

---

### florstore-web — staff cabinet and florist PWA

**Repository:** [florstore-web](https://github.com/asydneysummer/florstore-web) *(private — request access)* · **Deploy:** `store.{tenant-domain}` (cabinet) and `app.{tenant-domain}` (`florstore-florist/` PWA) · **Example:** [store.kupibuket63.ru](https://store.kupibuket63.ru)

**What it does end-to-end**

- **Sales:** deals list/kanban, order lifecycle (pickup / courier / post), payments; owner-gated online refunds in UI.
- **CRM:** customer directory and bonus-related flows.
- **Messenger:** staff inbox; Telegram channel connect/disconnect for owners and managers (via **florstore-telegram-gateway**).
- **Warehouse:** supplies, stock, write-offs panels.
- **Catalog ops:** showcase, site XML/JSON feed tuning, Flowwow feed panel, collections, full nomenclature tabs (items, service items, bundles, suppliers, contractors, pricing).
- **Finance & analytics:** cash accounts, product/ABC/XYZ reports, site/blog analytics, Flowwow analytics tabs.
- **Administration:** staff users and roles, shop settings (hours, delivery windows/fees), subscription panel (nav gated), bundle templates.
- **PWA:** installable staff app, Workbox shell caching, optional **offline** IndexedDB cache, mutation outbox, sync engine, conflict UI.
- **Florist companion app** in `florstore-florist/`: focused SPA for production floor (login, entitlement check, deals scaffold).

**Integration:** browser calls same-origin `/api` → tenant **florstore-api-shop** (Vite dev proxy to local or remote shop API).

---

### florstore-site — platform marketing, merchant cabinet, platform admin

**Repository:** [florstore-site](https://github.com/asydneysummer/florstore-site) *(private — request access)* · **Production:** [florstore.store](https://florstore.store) · **Stack:** React 19, Vite, Express API, Prisma 7, PostgreSQL

**What it does end-to-end**

- **Marketing:** landing, pricing, onboarding funnel, template gallery, legal pages, cookie consent, link/visit analytics.
- **Auth:** shop registration and login with HTTP-only JWT session cookies; trial/subscription state and tariff locking.
- **Merchant cabinet (`/cabinet`):** shop settings (`domain`, server hints, `api_url`, `shop_code`), bouquet components, collections, photo uploads, XML catalog export, employees, promotion views, revenue/bouquet/client/UTM analytics.
- **Billing:** subscription checkout via payment providers (e.g. T‑Bank, YooKassa), webhooks, payment history; **push rent state to florstore-api-admin** when `shop_code` and sync configuration are set.
- **Staff sync (optional):** cabinet employee records mapped to shop roles; proxied CRUD to tenant shop API when configured.
- **Platform admin (`/admin`):** clients, shop codes, rent/subscriptions, leads, consults, content plan/week, tracked links, blog stats, demo shop administration.
- **SEO blog:** markdown corpus, categories, RSS, sitemap/robots, article prerender, indexing workers (IndexNow/Yandex tooling).
- **Demo mode:** password-gated demo experience with isolated demo database.
- **Lead Scout:** optional background worker for lead geography (map provider keys via env).

**Important boundary:** **End-customer tenant storefronts** (e.g. [kupibuket63.ru](https://kupibuket63.ru)) are **separate deployments** (tenant repo + shop API). This module connects merchants to the platform; tenant vitrines consume **florstore-api-shop** public feeds and CLIENT APIs.

---

### florstore-telegram-gateway — Telegram Personal MTProto relay

**Repository:** [florstore-telegram-gateway](https://github.com/asydneysummer/florstore-telegram-gateway) *(private — request access)* · **Deploy:** EU VPS (Finland relay — stable regional endpoint for Telegram)

**What it does end-to-end**

- Hosts **Telegram Personal (MTProto)** connectivity **off the shop server** so tenant VPS IPs are not exposed directly to Telegram infrastructure.
- Exposes **HTTPS APIs** consumed by **florstore-web** messenger features (send/receive proxy semantics).
- Delivers **webhooks / inbox events** toward **florstore-api-shop** so conversations correlate with CRM/deals context in the cabinet.
- Keeps session material on the gateway host; shop API and browser clients do not embed MTProto stack or long-lived Telegram session keys in the staff SPA bundle.
- Complements optional shop-side relay hooks (env-gated) documented in the shop API module.

---

### florstore-x-butonika — Butonika XLSX bridge and feed helpers

**Repository:** [florstore-x-butonika](https://github.com/asydneysummer/florstore-x-butonika) *(private — request access)* · **Deploy:** internal tooling against tenant nomenclature / feed pipelines

**What it does end-to-end**

- **Imports Butonika-style XLSX exports** into FlorStore nomenclature shapes (items, bundles, pricing-related fields — per module scripts).
- **ETL validation and mapping** from legacy spreadsheet workflows into the three-database shop model (via shop API or batch paths documented in that repo).
- **Assists marketplace feed generation** alongside native **florstore-api-shop** public feed builders — Yandex, VK, Flowwow XML/JSON snapshots where the bridge is used in operations.
- Used when migrating or synchronizing shops that still produce Butonika spreadsheets before full native catalog management in **florstore-web**.

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

**Функциональность end-to-end**

- **Реестр магазинов (tenant):** коды, публичные поля, HMAC-секреты с шифрованием at rest и однократным показом при создании/ротации.
- **Жизненный цикл подписки:** статус, план, конец периода, продление, согласование с биллингом витрины при включённой синхронизации.
- **Feature flags** глобально и на магазин; claims entitlement JWT для shop API.
- **Inbox поддержки** между магазинами и владельцем платформы (идентификаторы staff — opaque string из `fsusers` tenant-а).
- **Discovery:** по коду магазина → публичные поля + `shopApiUrl`.
- **Интерфейс для shop API (HMAC):** refresh entitlement, heartbeat, relay сообщений поддержки.
- **Опциональный billing sync** от **florstore-site** (server-to-server, идемпотентно).
- **Эксплуатация:** health/db probes, OpenAPI `/docs`, rate limit, helmet, CORS, zod.

**Не делает:** не читает/не пишет tenant БД `fsusers` / `fsdeals` / `fsnomenclature`.

---

### florstore-api-shop — backend одного магазина

**Репозиторий:** [florstore-api-shop](https://github.com/asydneysummer/florstore-api-shop) *(приватный — запросите доступ)* · **Деплой:** отдельный VPS на магазин · **Пример API:** `store.kupibuket63.ru`

**Функциональность end-to-end**

- **Персонал:** JWT для приложений `florstore`, `florstore-web`, `florstore-florist` и др.; пользователи, роли, инвайты, refresh-токены; **обязательная проверка entitlement** при login/refresh (HMAC к admin, кэш, grace offline).
- **Номенклатура:** товары, категории, поставщики, контрагенты, комплекты (витрина/каталог), загрузка изображений, подборки.
- **Склад:** поставки, остатки, списания, акты сверки, средневзвешенная себестоимость, settlements.
- **Ценообразование:** правила manual/markup/fixed-period, наценки категорий, пересчёт от поставок, планировщик истечения fixed prices.
- **Сделки и CRM:** воронка, клиенты, оплаты, назначение курьера; финмодуль (счета, переводы, счета поставщиков/контрагентов) с RBAC на мутации.
- **Публичные фиды (без auth):** JSON/XML витрины, Yandex/VK merchant XML, Flowwow, подборки — с учётом **остатков** для showcase.
- **CLIENT API витрины** (роль `CLIENT`, app storefront): регистрация/логин, профиль, доставка, заказы, избранное, бонусы; опционально онлайн-оплата.
- **Аналитика и связь:** `/public/track/*`; опционально Telegram relay и web push.
- **Лицензирование:** фоновый refresh entitlement, heartbeat, relay в поддержку.

**Данные:** три PostgreSQL на инстанс — без колонки `shopId` (один процесс = один магазин).

---

### florstore-web — кабинет персонала и PWA флориста

**Репозиторий:** [florstore-web](https://github.com/asydneysummer/florstore-web) *(приватный — запросите доступ)* · **Пример:** [store.kupibuket63.ru](https://store.kupibuket63.ru)

**Функциональность end-to-end**

- **Продажи:** список/канбан сделок, pickup/courier/post, оплаты; возвраты онлайн-оплат в UI у OWNER.
- **CRM:** клиенты, бонусы.
- **Мессенджер:** inbox; подключение Telegram-канала (OWNER/MANAGER) через **florstore-telegram-gateway**.
- **Склад:** поставки, остатки, списания.
- **Каталог:** витрина, фиды сайта XML/JSON, Flowwow, подборки, номенклатура (вкладки items/service/bundles/suppliers/contractors/pricing).
- **Финансы и аналитика:** счета, отчёты, аналитика сайта/блога/Flowwow.
- **Администрирование:** пользователи и роли, настройки магазина, подписка (nav только OWNER/MANAGER), шаблоны комплектов.
- **PWA:** установка, Workbox, офлайн IndexedDB, outbox синхронизации, разрешение конфликтов.
- **`florstore-florist/`:** отдельное PWA для флориста на `app.{домен}`.

**Интеграция:** `/api` same-origin → **florstore-api-shop** tenant-а.

---

### florstore-site — маркетинг, кабинет мерчанта, админка платформы

**Репозиторий:** [florstore-site](https://github.com/asydneysummer/florstore-site) *(приватный — запросите доступ)* · **Прод:** [florstore.store](https://florstore.store)

**Функциональность end-to-end**

- **Маркетинг:** лендинг, тарифы, воронка, галерея шаблонов, legal, cookies, аналитика ссылок/визитов.
- **Auth:** регистрация/логин, trial и подписка, JWT в HTTP-only cookie.
- **Кабинет `/cabinet`:** настройки (`domain`, `api_url`, `shop_code`), компоненты букетов, подборки, фото, XML-экспорт, сотрудники, промо, аналитика.
- **Биллинг:** оплата аренды (T‑Bank / YooKassa), webhooks, история; **push состояния в florstore-api-admin** при настроенном `shop_code`.
- **Синхронизация staff (опционально):** сотрудники кабинета → роли shop API.
- **Админка `/admin`:** клиенты, коды магазинов, аренда, лиды, консультации, контент-план/неделя, ссылки, статистика блога, демо.
- **SEO-блог:** markdown, RSS, sitemap, prerender, indexing workers.
- **Demo:** изолированная demo-БД с кодом доступа.
- **Lead Scout:** опциональный worker лидов.

**Граница:** клиентские витрины (например **kupibuket63.ru**) — **отдельный деплой**; этот модуль — платформа для мерчанта, не HTML каждой витрины.

---

### florstore-telegram-gateway — MTProto-реле Telegram Personal

**Репозиторий:** [florstore-telegram-gateway](https://github.com/asydneysummer/florstore-telegram-gateway) *(приватный — запросите доступ)* · **Деплой:** EU VPS (Финляндия)

**Функциональность end-to-end**

- Держит **MTProto** и сессии **Telegram Personal** вне VPS магазина.
- **HTTPS API** для **florstore-web** (отправка/приём через прокси).
- **Webhooks / inbox** в **florstore-api-shop** для связи переписки с CRM/сделками.
- Браузерный кабинет не содержит MTProto и долгоживущие ключи Telegram.
- Дополняет env-gated hooks в shop API.

---

### florstore-x-butonika — мост Butonika XLSX и фиды

**Репозиторий:** [florstore-x-butonika](https://github.com/asydneysummer/florstore-x-butonika) *(приватный — запросите доступ)* · **Деплой:** внутренние ETL-задачи к номенклатуре tenant-а

**Функциональность end-to-end**

- **Импорт XLSX Butonika** в модели номенклатуры FlorStore.
- **Валидация и маппинг** legacy-таблиц в три shop-БД (через API/ batch — см. модуль).
- **Помощь генерации фидов** Yandex / VK / Flowwow рядом с native builders **florstore-api-shop**.
- Для магазинов, ещё ведущих учёт в Butonika, до полного перехода на **florstore-web**.

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
| **Владелец / оператор платформы** | Admin JWT | Login/refresh; CRUD магазинов; ротация HMAC; подписки и flags; поддержка |
| **Экземпляр shop API** | HMAC запросов | Refresh entitlement; heartbeat; relay поддержки |
| **Discovery-клиент** | Публичный route | Код магазина → поля + `shopApiUrl` |
| **Billing sync** | Server-to-server secret (если настроено) | Идемпотентная синхронизация аренды/подписки |
| **Мониторинг** | — | Health и готовность БД |

### Персонал магазина (`florstore-api-shop` + `florstore-web`)

Роли staff: **`OWNER` · `MANAGER` · `FLORIST` · `CASHIER` · `COURIER` · `READONLY`**. **`CLIENT`** — только витрина (ниже).

| Роль | Назначение | Возможности (сводка) |
|------|------------|----------------------|
| **OWNER** | Владелец | Полный доступ; управление OWNER с защитой последнего активного; pricing; фин. админ; настройки и подписка в UI; Telegram; CRM write; чувствительные операции по сделкам/поставкам |
| **MANAGER** | Менеджер | Как owner в операциях; **не** назначает **OWNER**; settings/subscription в nav; users без повышения до owner |
| **FLORIST** | Флорист / зал | Сделки, склад, операционные оплаты где разрешено; без мутаций pricing; без settings/subscription; customers/users read-only в UI |
| **CASHIER** | Кассир | Как FLORIST по оплатам; без pricing и фин. CRUD счетов/инвойсов |
| **COURIER** | Курьер | Доставка и статусы; без мутаций комплектов/pricing/склада и фин. записей |
| **READONLY** | Только чтение | Read API; мутации → **403** |

**UI (`florstore-web`):** **Настройки** и **Подписка** только у **OWNER** и **MANAGER**.

### CLIENT витрины — гость и авторизованный клиент (`/public/*` shop API)

Для сайтов вроде **kupibuket63** (app id storefront tenant-а).

| Актор | Auth | Возможности |
|-------|------|-------------|
| **Гость** | Нет | Каталог по фиду, stories/legal, публичные delivery/settings; корзина в браузере |
| **CLIENT** | JWT после register/login | Профиль, заказы, история, избранное, бонусы, доставка, онлайн-оплата если включена |
| **Персонал на домене витрины** | — | **Не поддерживается** — только **florstore-web** на `store.{домен}` |

Gate CLIENT: роль **`CLIENT`** и корректный **app** storefront (не staff JWT).

### Публичные сценарии — платформа vs витрина tenant

| Поверхность | Актор | Сценарий |
|-------------|-------|----------|
| **florstore.store** | Гость | Лендинг, блог, legal; лиды; demo по коду |
| **florstore.store** `/cabinet` | Мерчант (**USER**) | JWT cookie; настройки подключения к shop API; каталог в кабинете; оплаты; аналитика; sync сотрудников |
| **florstore.store** `/admin` | **ADMIN** платформы | Клиенты, коды, аренда, лиды, контент, аналитика |
| **Сайт tenant** (kupibuket63.ru) | Гость | Каталог и checkout без аккаунта |
| **Сайт tenant** | **CLIENT** | ЛК: заказы, бонусы, избранное |

Роли сотрудников в кабинете (`director`, `marketer`, …) мапятся на shop roles при включённом staff sync (**florstore-site**).

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
