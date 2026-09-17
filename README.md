# FlorStore — Flower Shop SaaS Ecosystem

**FlorStore** is a multi-tenant flower-shop platform: per-shop APIs, web cabinets, public storefronts, Telegram messenger relay, and integrations with marketplaces (Flowwow, VK, Yandex). Modules share one product story but live in **separate private repos** for deploy boundaries.

> Portfolio index · private source repos · author: Michael / asydneysummer

## Features

- Per-shop backend API with three PostgreSQL databases (users, deals, nomenclature)
- Central admin API for subscriptions, entitlements, and support
- Web cabinet for shop staff + florist PWA
- Public marketing site and tenant storefronts
- Telegram Personal messenger relay via Finland VPS (MTProto)
- Butonika bridge: XLSX ETL and marketplace feed export

## Modules

| Module | Repository | Production |
|--------|------------|------------|
| **Admin API** — control plane, subscriptions, entitlements | [florstore-api-admin](https://github.com/asydneysummer/florstore-api-admin) *(private — request access)* | Platform server |
| **Shop API** — POS, deals, nomenclature (1 server = 1 shop) | [florstore-api-shop](https://github.com/asydneysummer/florstore-api-shop) *(private — request access)* | Per-tenant VPS |
| **Web cabinet** — store staff UI + florist PWA | [florstore-web](https://github.com/asydneysummer/florstore-web) *(private — request access)* | `store.{domain}`, `app.{domain}` |
| **Public site** — marketing + tenant storefronts | [florstore-site](https://github.com/asydneysummer/florstore-site) *(private — request access)* | [florstore.store](https://florstore.store) |
| **Telegram gateway** — MTProto relay (Finland VPS) | [florstore-telegram-gateway](https://github.com/asydneysummer/florstore-telegram-gateway) *(private — request access)* | Finland proxy |
| **Butonika bridge** — XLSX ETL, marketplace feeds | [florstore-x-butonika](https://github.com/asydneysummer/florstore-x-butonika) *(private — request access)* | Internal tooling |

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

## Cross-module integrations

| From | To | Mechanism |
|------|-----|-----------|
| florstore-api-shop | florstore-api-admin | HMAC-signed entitlement refresh |
| florstore-web | florstore-telegram-gateway | HTTPS proxy + webhook inbox |
| florstore-site | florstore-api-admin | Billing sync (`BILLING_SYNC_SECRET`) |
| florstore-x-butonika | shop nomenclature | XLSX ETL → feeds (Flowwow/VK/Yandex) |

## Tech stack

- **Backend:** Node.js, Express, Prisma, PostgreSQL
- **Frontend:** React, Vite, Tailwind, PWA
- **Telegram:** Python MTProto gateway on EU VPS
- **Deploy:** Native per-shop servers (no Docker for shop API)

## Related portfolio

| Ecosystem | Index |
|-----------|-------|
| **HopeClass** (EdTech) | [hopeclass-ecosystem](https://github.com/asydneysummer/hopeclass-ecosystem) |
| **Buketov** (beauty salon CRM) | [buketov-app](https://github.com/asydneysummer/buketov-app) *(private — request access)* |

*Public index only — no application source in this repo.*

---

# FlorStore — экосистема SaaS для цветочных магазинов

**FlorStore** — мультитenant-платформа для цветочных магазинов: API на каждый магазин, веб-кабинеты, публичные витрины, Telegram-шлюз мессенджера и интеграции с маркетплейсами (Flowwow, VK, Яндекс). Модули образуют единый продукт, но вынесены в **отдельные приватные репозитории** для границ деплоя.

> Портфолио-индекс · приватные исходники · автор: Michael / asydneysummer

## Возможности

- Backend API на каждый магазин с тремя БД PostgreSQL (пользователи, сделки, номенклатура)
- Центральный admin API: подписки, entitlements, поддержка
- Веб-кабинет для персонала + PWA фlorist
- Публичный маркетинговый сайт и витрины tenant-ов
- Telegram Personal через VPS в Финляндии (MTProto)
- Мост Butonika: ETL из XLSX и экспорт фидов на маркетплейсы

## Модули

| Модуль | Репозиторий | Продакшен |
|--------|-------------|-----------|
| **Admin API** — control plane, подписки, entitlements | [florstore-api-admin](https://github.com/asydneysummer/florstore-api-admin) *(приватный — запросите доступ)* | Сервер платформы |
| **Shop API** — POS, сделки, номенклатура (1 сервер = 1 магазин) | [florstore-api-shop](https://github.com/asydneysummer/florstore-api-shop) *(приватный — запросите доступ)* | VPS каждого tenant |
| **Web cabinet** — UI персонала + PWA фlorist | [florstore-web](https://github.com/asydneysummer/florstore-web) *(приватный — запросите доступ)* | `store.{domain}`, `app.{domain}` |
| **Public site** — маркетинг + витрины tenant-ов | [florstore-site](https://github.com/asydneysummer/florstore-site) *(приватный — запросите доступ)* | [florstore.store](https://florstore.store) |
| **Telegram gateway** — MTProto-шлюз (VPS Финляндия) | [florstore-telegram-gateway](https://github.com/asydneysummer/florstore-telegram-gateway) *(приватный — запросите доступ)* | Прокси в FI |
| **Butonika bridge** — ETL XLSX, фиды маркетплейсов | [florstore-x-butonika](https://github.com/asydneysummer/florstore-x-butonika) *(приватный — запросите доступ)* | Внутренние инструменты |

## Архитектура (обзор)

```
                    Сервер платформы
                    ┌─────────────────────┐
   florstore-admin  │ florstore-api-admin │◄── HMAC ── florstore-api-shop (×N)
   (desktop)       │  admin PostgreSQL   │              fsusers / fsdeals / fsnomenclature
                    └──────────┬──────────┘                      │
                               │                                 │
                    florstore-site (публичный)           florstore-web + florist PWA
                               │                                 │
                               └──────── billing / onboarding ───┘

   VPS Финляндия: florstore-telegram-gateway ◄──► shop API ◄──► Telegram MTProto
```

## Межмодульные интеграции

| Откуда | Куда | Механизм |
|--------|------|----------|
| florstore-api-shop | florstore-api-admin | HMAC-подписанное обновление entitlements |
| florstore-web | florstore-telegram-gateway | HTTPS-прокси + webhook inbox |
| florstore-site | florstore-api-admin | Синхронизация биллинга (`BILLING_SYNC_SECRET`) |
| florstore-x-butonika | номенклатура магазина | ETL XLSX → фиды (Flowwow/VK/Яндекс) |

## Стек

- **Backend:** Node.js, Express, Prisma, PostgreSQL
- **Frontend:** React, Vite, Tailwind, PWA
- **Telegram:** Python MTProto-шлюз на EU VPS
- **Деплой:** Нативные серверы на магазин (без Docker для shop API)

## Связанное портфолио

| Экосистема | Индекс |
|------------|--------|
| **HopeClass** (EdTech) | [hopeclass-ecosystem](https://github.com/asydneysummer/hopeclass-ecosystem) |
| **Buketov** (CRM салона красоты) | [buketov-app](https://github.com/asydneysummer/buketov-app) *(приватный — запросите доступ)* |

*Публичный индекс — исходников приложений в этом репозитории нет.*
