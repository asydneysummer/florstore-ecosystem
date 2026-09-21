# FlorStore — multi-tenant flower shop SaaS

**FlorStore** is a multi-tenant platform for flower shops: a per-shop API and PostgreSQL footprint, staff web cabinet with florist PWA, public marketing and storefront sites, a Telegram Personal relay, and marketplace bouquet feeds (Flowwow, VK, Yandex). Product modules live in **separate private repositories** for deploy boundaries; **this repo is the public ecosystem index** (no application source here).

Built and owned by Michael ([asydneysummer](https://github.com/asydneysummer)).

**Production:** [florstore.store](https://florstore.store) (marketing) · example tenant [kupibuket63.ru](https://kupibuket63.ru) / [store.kupibuket63.ru](https://store.kupibuket63.ru) · staff UI at `store.{tenant-domain}` and `app.{tenant-domain}` per shop.

Ecosystem index: [florstore-ecosystem](https://github.com/asydneysummer/florstore-ecosystem) (this repository).

## Features

- Per-shop **Shop API** with **three PostgreSQL databases** per tenant (`fsusers`, `fsdeals`, `fsnomenclature`) — users/RBAC, deals/POS, catalog and pricing
- **Admin API** control plane: tenant registry, subscriptions, HMAC-verified **entitlements**, platform support tooling
- **Web cabinet** for shop staff plus **florist PWA** for production-floor workflows
- **Public site** module: platform marketing at `florstore.store` and tenant-branded storefronts
- **Telegram gateway**: MTProto relay so staff can use Telegram Personal from the cabinet without exposing shop servers directly to Telegram
- **Butonika bridge**: XLSX ETL from legacy exports into nomenclature and **public bouquet feeds** for Yandex, VK, and Flowwow

## Roles & capabilities

| Role | Scope | Can do |
|------|--------|--------|
| **Platform admin** | Ecosystem (`florstore-api-admin`) | Manage tenants and subscriptions; issue and rotate shop credentials; view/support cross-tenant operations; drive entitlement payloads consumed by shop servers |
| **OWNER** | Single shop | Full shop configuration, staff and role assignment, billing/onboarding hooks with the platform, destructive or structural settings |
| **MANAGER** | Single shop | Day-to-day operations: deals pipeline, catalog and pricing (within policy), reports, staff coordination |
| **FLORIST** | Single shop | Florist PWA: assembly tasks, bouquet status, production-floor updates tied to deals |
| **CASHIER** | Single shop | POS checkout, payments and deal completion at the counter |
| **COURIER** | Single shop | Delivery assignments and status updates on outbound orders |
| **READONLY** | Single shop | Read-only access to deals, catalog, and reports (no mutations) |
| **CLIENT** | Storefront | Browse public catalog, place orders, account/history on the tenant storefront (unauthenticated or registered client, per shop config) |

Shop roles are enforced in **florstore-api-shop** / **florstore-web**; the platform admin role lives in **florstore-api-admin**.

## Architecture / tech map

```
                         Platform (single deploy)
                         ┌──────────────────────────┐
  florstore-admin        │   florstore-api-admin    │
  (desktop, optional)    │   admin PostgreSQL       │
                         └────────────┬─────────────┘
                                      │ HMAC-signed entitlements
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
                    └───────┬───────────┘
                            │
              florstore-site ── billing / onboarding ──► admin API
              (florstore.store + tenant vitrines)

              EU relay VPS: florstore-telegram-gateway ◄──MTProto──► Telegram
                                    ▲
                                    │ HTTPS proxy + inbox webhooks
                              florstore-web

              florstore-x-butonika: XLSX ETL ──► nomenclature / feed builders
```

| Module | Responsibility |
|--------|----------------|
| [florstore-api-admin](https://github.com/asydneysummer/florstore-api-admin) | Control plane, subscriptions, entitlements, support |
| [florstore-api-shop](https://github.com/asydneysummer/florstore-api-shop) | Tenant API: users, deals/POS, nomenclature (three DBs) |
| [florstore-web](https://github.com/asydneysummer/florstore-web) | Staff cabinet + florist PWA; Telegram proxy client |
| [florstore-site](https://github.com/asydneysummer/florstore-site) | Marketing site and tenant storefronts |
| [florstore-telegram-gateway](https://github.com/asydneysummer/florstore-telegram-gateway) | Python MTProto gateway on EU VPS |
| [florstore-x-butonika](https://github.com/asydneysummer/florstore-x-butonika) | Butonika XLSX import and marketplace feed export |

## Key engineering work

- **Multi-tenant isolation by deployment**: one shop API instance and **three PostgreSQL databases** per tenant instead of a shared noisy-neighbor schema — clear blast radius and native backups per store.
- **HMAC entitlements**: shop servers periodically refresh signed capability flags from the admin API so subscription state and feature gates stay authoritative without embedding billing in every request path.
- **Split data domains**: users/RBAC, transactional deals, and nomenclature/pricing separated at the database layer to scale IO and migrations independently.
- **Public bouquet feeds**: generated catalog snapshots for **Yandex**, **VK**, and **Flowwow** from shop nomenclature (via **florstore-x-butonika** ETL and feed builders).
- **Telegram MTProto gateway**: Personal-account relay on a **EU VPS** so the web cabinet talks HTTPS to the gateway while Telegram sees a stable regional endpoint — not shop-origin MTProto.
- **Cross-module contracts**: admin ↔ shop entitlement sync; site ↔ admin billing/onboarding; web ↔ gateway inbox webhooks — each boundary documented in its module repo.
- **Native per-shop deploy**: shop API runs on dedicated VPS per tenant (no Docker requirement on the shop tier), matching one-server-one-shop operations.

## Tech stack

- **Backend:** Node.js, Express, Prisma, PostgreSQL
- **Frontend:** React, Vite, Tailwind CSS, PWA (staff + florist)
- **Telegram:** Python MTProto gateway
- **Integrations:** XLSX ETL, Yandex/VK/Flowwow feed formats
- **Deploy:** Platform admin cluster; per-tenant shop VPS; EU relay for Telegram

## Repository layout

| Path | Role |
|------|------|
| `README.md` | Public FlorStore ecosystem index (modules, architecture, roles) |
| Module repos (linked below) | Application source, env examples, and deploy scripts |

There is **no runnable app** in this repository — clone module repos for local development.

## Local setup

```bash
git clone https://github.com/asydneysummer/florstore-ecosystem.git
cd florstore-ecosystem
# Read this index, then clone the module you need (private — request access):
# florstore-api-admin | florstore-api-shop | florstore-web | florstore-site
# florstore-telegram-gateway | florstore-x-butonika
```

Each module README and `.env.example` define ports, database URLs, and HMAC keys for that service. Do not commit real credentials.

## Related repositories

| Module | Repository | Production surface |
|--------|------------|-------------------|
| Admin API | [florstore-api-admin](https://github.com/asydneysummer/florstore-api-admin) *(private — request access)* | Platform admin server |
| Shop API | [florstore-api-shop](https://github.com/asydneysummer/florstore-api-shop) *(private — request access)* | Per-tenant VPS |
| Web + florist | [florstore-web](https://github.com/asydneysummer/florstore-web) *(private — request access)* | `store.{domain}`, `app.{domain}` |
| Marketing / storefront | [florstore-site](https://github.com/asydneysummer/florstore-site) *(private — request access)* | [florstore.store](https://florstore.store) |
| Telegram gateway | [florstore-telegram-gateway](https://github.com/asydneysummer/florstore-telegram-gateway) *(private — request access)* | EU MTProto relay |
| Butonika / feeds | [florstore-x-butonika](https://github.com/asydneysummer/florstore-x-butonika) *(private — request access)* | Internal ETL + feeds |

### Other portfolio indexes

| Ecosystem | Index |
|-----------|-------|
| **HopeClass** (EdTech) | [hopeclass-ecosystem](https://github.com/asydneysummer/hopeclass-ecosystem) |
| **Buketov** (beauty salon CRM) | [buketov-app](https://github.com/asydneysummer/buketov-app) *(private — request access)* |

*Public portfolio index · module source in private repos · request access for demos and code review.*

---

# FlorStore — экосистема SaaS для цветочных магазинов

**FlorStore** — мультитenant-платформа для цветочных магазинов: API и три PostgreSQL на магазин, веб-кабинет с PWA фlorist, маркетинг и витрины, Telegram-шлюз MTProto и фиды букетов для маркетплейсов. **Этот репозиторий — публичный индекс экосистемы**; исходники приложений — в приватных модулях.

Автор: Michael ([asydneysummer](https://github.com/asydneysummer)).

**Продакшен:** [florstore.store](https://florstore.store) · пример tenant [kupibuket63.ru](https://kupibuket63.ru) / [store.kupibuket63.ru](https://store.kupibuket63.ru).

## Роли (кратко)

Платформенный admin; в магазине — OWNER, MANAGER, FLORIST, CASHIER, COURIER, READONLY; на витрине — CLIENT. Подробная таблица — в английской части README выше.

## Модули

См. таблицы **Related repositories** и **Architecture** в английской секции — те же шесть репозиториев: `florstore-api-admin`, `florstore-api-shop`, `florstore-web`, `florstore-site`, `florstore-telegram-gateway`, `florstore-x-butonika`.

*Публичный индекс портфолио · исходники по запросу.*
