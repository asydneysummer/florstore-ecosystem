# FlorStore — экосистема SaaS для цветочных магазинов

**FlorStore** is a multi-tenant flower-shop platform: per-shop APIs, web cabinets, public storefronts, Telegram messenger relay, and integrations with marketplaces (Flowwow, VK, Yandex). Modules share one product story but live in **separate private repos** for deploy boundaries.

> Portfolio index · private source repos · author: Michael / asydneysummer

---

## Modules

| Module | Repository | Production |
|--------|------------|------------|
| **Admin API** — control plane, subscriptions, entitlements | [florstore-api-admin](https://github.com/asydneysummer/florstore-api-admin) *(private — request access)* | Platform server |
| **Shop API** — POS, deals, nomenclature (1 server = 1 shop) | [florstore-api-shop](https://github.com/asydneysummer/florstore-api-shop) *(private — request access)* | Per-tenant VPS |
| **Web cabinet** — store staff UI + florist PWA | [florstore-web](https://github.com/asydneysummer/florstore-web) *(private — request access)* | `store.{domain}`, `app.{domain}` |
| **Public site** — marketing + tenant storefronts | [florstore-site](https://github.com/asydneysummer/florstore-site) *(private — request access)* | [florstore.store](https://florstore.store) |
| **Telegram gateway** — MTProto relay (Finland VPS) | [florstore-telegram-gateway](https://github.com/asydneysummer/florstore-telegram-gateway) *(private — request access)* | Finland proxy |
| **Butonika bridge** — XLSX ETL, marketplace feeds | [florstore-x-butonika](https://github.com/asydneysummer/florstore-x-butonika) *(private — request access)* | Internal tooling |

---

## Architecture (high level)

```
                    Platform server
                    ┌─────────────────────┐
   florstore-admin  │ florstore-api-admin │◄── HMAC ── florstore-api-shop (×N)
   (desktop)       │  admin PostgreSQL   │              fsusers / fsdeals / fsnomenclature
                    └──────────┬──────────┘                      │
                               │                                 │
                    florstore-site (public)              florstore-web + florist PWA
                               │                                 │
                               └──────── billing / onboarding ───┘

   Finland VPS: florstore-telegram-gateway ◄──► shop API ◄──► Telegram MTProto
```

---

## Cross-module integrations

| From | To | Mechanism |
|------|-----|-----------|
| florstore-api-shop | florstore-api-admin | HMAC-signed entitlement refresh |
| florstore-web | florstore-telegram-gateway | HTTPS proxy + webhook inbox |
| florstore-site | florstore-api-admin | Billing sync (`BILLING_SYNC_SECRET`) |
| florstore-x-butonika | shop nomenclature | XLSX ETL → feeds (Flowwow/VK/Yandex) |

---

## Tech stack

- **Backend:** Node.js, Express, Prisma, PostgreSQL
- **Frontend:** React, Vite, Tailwind, PWA
- **Telegram:** Python MTProto gateway on EU VPS
- **Deploy:** Native per-shop servers (no Docker for shop API)

---

## Related portfolio

| Ecosystem | Index |
|-----------|-------|
| **HopeClass** (EdTech) | [hopeclass-ecosystem](https://github.com/asydneysummer/hopeclass-ecosystem) |
| **Buketov** (beauty salon CRM) | [buketov-app](https://github.com/asydneysummer/buketov-app) *(private — request access)* |

---

*Public index only — no application source in this repo.*
