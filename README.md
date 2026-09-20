# Faf Admin Platform

**Internal Administration Platform as a Product**.

Единая платформа для внутренних административных интерфейсов экосистемы Firefly AI Flow.

## 🎯 Цель

Создать переиспользуемую платформу, которая позволяет сервисам быстро подключать административные интерфейсы без дублирования sidebar, auth, routing, дизайн-системы и базовых dashboard-компонентов.

## 🏗️ Архитектура

```bash
┌─────────────────────────────────────────────────────────────┐
│                      Admin Shell                             │
│  • Единый frontend                                           │
│  • Vanilla JavaScript + Web Components                       │
│  • Schema renderer                                           │
│  • Custom module host                                        │
└────────────────────────────┬──────────────────────────────────┘
                             │
                             │ Internal Admin API
                             ▼
┌─────────────────────────────────────────────────────────────┐
│                   Portal BFF                                 │
│  • Administrative entry point                                │
│  • Service registry / manifests                              │
│  • API composition / context forwarding                      │
│  • Thin proxy to service admin APIs                          │
└────────────────────────────┬──────────────────────────────────┘
                             │
      ┌──────────────────────┼───────────────────────────────┐
      ▼                      ▼                               ▼
┌───────────────┐    ┌───────────────┐              ┌────────────────┐
│ Cache Service │    │ AI Router     │              │ Prompting      │
│ Admin API     │    │ Admin API     │              │ Admin API      │
│               │    │               │              │                │
│ Page model:   │    │ Page model:   │              │ Page model:    │
│ schema + data │    │ schema + data │              │ schema + data  │
└───────────────┘    └───────────────┘              └────────────────┘
```

## 📦 Структура репозитория

```bash
faf-admin-platform/
├── README.md                       # Архитектура (.NET 10 + Vanilla JS)
├── QUICKSTART.md                   # План на 5 дней
├── .gitignore
├── docs/
│   ├── README.md                   # Общая документация
│   ├── QUICKSTART.md               # Быстрый старт
│   ├── TECH_STACK.md               # .NET 10 + Vanilla JS
│   ├── NAMING.md                   # Faf Admin Platform naming
│   ├── ADMIN_UI_KIT_ARCH.md        # Архитектура UI Kit
│   ├── contracts/
│   │   └── pagemodel-v1.md
│   ├── guides/
│   │   ├── week1-vertical-slice.md
│   │   └── connect-new-service.md
│   └── adr/
│       └── 001-schema-data-together.md
├── src/
│   ├── admin-shell/                # Фронтенд (Vanilla JS)
│   ├── admin-portal-bff/           # Backend (.NET 10)
│   └── admin-ui-kit/               # ← НОВЫЙ! Вместо design-system
│       ├── README.md
│       ├── components/
│       │   ├── layout/             # admin-page, sidebar, header
│       │   ├── widgets/            # status, kpi, timeseries
│       │   ├── states/             # loading, error, stale
│       │   └── actions/            # button, modal, confirm
│       ├── styles/                 # @faf/foundations import
│       ├── utils/
│       └── tests/
├── samples/
│   └── cache-service-adapter/
├── tests/
│   ├── integration/
│   └── e2e/
└── tools/
    └── contract-tests/
```

## 🚀 Быстрый старт

### Неделя 1: Vertical Slice

Цель первой недели — доказать основной контракт:

```bash
Service → PageModel (schema + data) → BFF → Shell → Dashboard
```

**Результат:** один работающий Cache Service Overview dashboard.

См. [docs/guides/week1-vertical-slice.md](docs/guides/week1-vertical-slice.md)

### Подключение нового сервиса

1. Реализовать Admin API endpoint для page model
2. Добавить service manifest в Portal BFF
3. Создать page schema (или использовать generic)
4. Протестировать через Admin Shell

См. [docs/guides/connect-new-service.md](docs/guides/connect-new-service.md)

## 📄 Контракты

| Контракт         | Описание                                      | Статус        |
| ---------------- | --------------------------------------------- | ------------- |
| PageModel v1     | Декларативная модель страницы (schema + data) | В разработке  |
| Service Manifest | Metadata сервиса для navigation и discovery   | Запланировано |
| Widget Types     | Allowlisted типы виджетов для schema          | В разработке  |

См. [docs/contracts/](docs/contracts/)

## 🛠️ Технологии

| Компонент    | Стек                                         |
| ------------ | -------------------------------------------- |
| Admin Shell  | Vanilla JavaScript (ES2022+), Web Components |
| Portal BFF   | .NET 10, ASP.NET Core                        |
| Admin UI Kit | Web Components на основе @faf/foundations    |
| Contracts    | JSON Schema, OpenAPI                         |
| Build        | Vite (frontend), dotnet publish (backend)    |

## 📈 Roadmap

- [ ] **Week 1:** Vertical slice (Cache Overview)
- [ ] **Week 2-3:** Generic CRUD, RBAC baseline
- [ ] **Week 4:** Second service (AI Router or Prompting)
- [ ] **Month 2:** System dashboards, audit viewer
- [ ] **Month 3:** Custom modules, plugin model

## 🤝 Вклад

Этот репозиторий — продукт platform team. Сервисы подключаются через стабильные контракты.

Для предложения изменений:

1. Создать issue с описанием use case
2. Обсудить на architecture review
3. Реализовать в отдельной ветке
4. Добавить contract tests

## 📄 Лицензия

Internal use only.

---

**Статус:** Active development  
**Версия:** 0.1.0-alpha  
**Последнее обновление:** 2026-09-20  
**Продукт:** Faf Admin Platform (Firefly AI Flow)  
**Стек:** .NET 10 + Vanilla JavaScript
