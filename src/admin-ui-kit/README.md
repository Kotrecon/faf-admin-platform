# Faf Admin UI Kit

**Admin-specific Web Components для Faf Admin Platform**.

Построен на основе [@faf/foundations](https://github.com/your-org/faf-design-system/tree/master/packages/foundations) и [@faf/z-index](https://github.com/your-org/faf-design-system/tree/master/packages/z-index).

## 📦 Структура

```bash
src/admin-ui-kit/
├── components/
│   ├── layout/             # admin-page, admin-header, admin-sidebar
│   ├── widgets/            # status-widget, kpi-widget, timeseries-widget
│   ├── states/             # loading-state, error-state, stale-state
│   └── actions/            # admin-button, admin-modal, admin-confirm-dialog
├── styles/
│   ├── tokens.css          # @import "@faf/foundations/styles"
│   └── components.css      # Стили компонентов
├── utils/
│   ├── correlation.ts
│   └── polling.ts
├── index.ts                # Точка входа
├── package.json
└── README.md
```

## 🚀 Быстрый старт

### Установка

```bash
# В рамках репозитория (workspace)
cd src/admin-ui-kit
npm install @faf/foundations @faf/z-index
```

### Использование

```typescript
// В admin-shell/src/main.ts
import "@faf/admin-ui-kit";

// Компоненты регистрируются автоматически
// <admin-page>, <admin-sidebar>, <kpi-widget>, etc.
```

## 📚 Компоненты

### Layout

| Компонент                 | Описание        | Статус           |
| ------------------------- | --------------- | ---------------- |
| `<admin-page>`            | Layout страницы | 🚧 В разработке  |
| `<admin-header>`          | Шапка           | 🚧 В разработке  |
| `<admin-sidebar>`         | Primary sidebar | 🚧 В разработке  |
| `<admin-context-sidebar>` | Context sidebar | 🚧 Запланировано |
| `<admin-dashboard-grid>`  | Dashboard grid  | 🚧 В разработке  |

### Widgets

| Компонент                | Описание       | Статус           |
| ------------------------ | -------------- | ---------------- |
| `<status-widget>`        | Статус сервиса | 🚧 В разработке  |
| `<kpi-widget>`           | KPI метрика    | 🚧 В разработке  |
| `<timeseries-widget>`    | Временной ряд  | 🚧 В разработке  |
| `<table-widget>`         | Таблица        | 🚧 Запланировано |
| `<activity-feed-widget>` | Лента событий  | 🚧 Запланировано |

### States

| Компонент         | Описание           | Статус           |
| ----------------- | ------------------ | ---------------- |
| `<loading-state>` | Состояние загрузки | 🚧 В разработке  |
| `<error-state>`   | Состояние ошибки   | 🚧 В разработке  |
| `<stale-state>`   | Устаревшие данные  | 🚧 Запланировано |
| `<empty-state>`   | Пустое состояние   | 🚧 Запланировано |

### Actions

| Компонент                | Описание      | Статус           |
| ------------------------ | ------------- | ---------------- |
| `<admin-button>`         | Кнопка        | 🚧 Запланировано |
| `<admin-modal>`          | Модалка       | 🚧 Запланировано |
| `<admin-confirm-dialog>` | Подтверждение | 🚧 Запланировано |

## 🎨 Использование токенов

```typescript
// src/admin-ui-kit/components/widgets/kpi-widget.ts
import { semanticSpacing, semanticShadows } from "@faf/foundations";

export class KpiWidget extends HTMLElement {
  render() {
    this.shadowRoot.innerHTML = `
      <style>
        @import "@faf/foundations/styles";
        
        .kpi-card {
          padding: var(--faf-spacing-card-padding);
          box-shadow: var(--faf-shadow-card);
          border-radius: var(--faf-radius-card);
        }
      </style>
      <div class="kpi-card">
        <!-- content -->
      </div>
    `;
  }
}
```

## 🔄 Миграция на @faf/components

Когда компоненты появятся в базовой дизайн-системе:

```diff
// Было
import { KpiWidget } from "@faf/admin-ui-kit";

// Стало
import { MetricCard } from "@faf/components";
```

**Время на миграцию:** 1-2 строки на компонент.

## 📋 План разработки

### Фаза 1: MVP (Неделя 1-2)

- [ ] `admin-page`
- [ ] `admin-sidebar`
- [ ] `status-widget`
- [ ] `kpi-widget`
- [ ] `timeseries-widget`
- [ ] `loading-state`
- [ ] `error-state`

### Фаза 2: Dashboard (Неделя 3-4)

- [ ] `admin-dashboard-grid`
- [ ] `table-widget`
- [ ] `activity-feed-widget`

### Фаза 3: Actions (Неделя 5-6)

- [ ] `admin-button`
- [ ] `admin-modal`
- [ ] `admin-confirm-dialog`

## 🧪 Разработка

```bash
# Сборка
npm run build

# Тесты
npm run test

# Dev server
npm run dev
```

## 📄 Зависимости

```json
{
  "name": "@faf/admin-ui-kit",
  "version": "0.1.0",
  "dependencies": {
    "@faf/foundations": "^0.1.0",
    "@faf/z-index": "^0.1.0"
  }
}
```

---

**Статус:** 0.1.0-alpha  
**Дата:** 2026-09-20  
**Стек:** Vanilla JavaScript + Web Components
