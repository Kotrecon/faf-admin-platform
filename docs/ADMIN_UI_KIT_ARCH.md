# Admin UI Kit Architecture

**Статус:** Draft  
**Версия:** 0.1.0  
**Дата:** 2026-09-20

## Обзор

Admin UI Kit — набор admin-specific Web Components, построенных на основе @faf/foundations и @faf/z-index.

**Цель:** Предоставить готовые компоненты для Faf Admin Platform, которые:

- Используют токены из Faf Design System
- Реализуют admin-specific паттерны (sidebar, dashboard, widgets)
- Легко мигрируют на @faf/components когда они появятся

## Архитектура

```bash
┌─────────────────────────────────────────────────────────────┐
│           Faf Design System (отдельный репо)                 │
│  @faf/foundations + @faf/z-index + @faf/contrast            │
└────────────────────────────┬──────────────────────────────────┘
                             │
                             │ импортирует токены
                             ▼
┌─────────────────────────────────────────────────────────────┐
│              Admin UI Kit (в этом репо)                      │
│  • Layout: admin-page, admin-sidebar, admin-header          │
│  • Widgets: status, kpi, timeseries, table                  │
│  • States: loading, error, stale, empty                     │
│  • Actions: button, modal, confirm-dialog                   │
└────────────────────────────┬──────────────────────────────────┘
                             │
                             │ использует
                             ▼
┌─────────────────────────────────────────────────────────────┐
│           Faf Admin Platform (фронтенд)                      │
│  • Admin Shell (Vanilla JS)                                  │
│  • Роутинг, polling, API client                              │
│  • Schema renderer                                           │
└─────────────────────────────────────────────────────────────┘
```

## Почему не отдельное репо?

| Преимущество          | Пояснение                             |
| --------------------- | ------------------------------------- |
| **Один репозиторий**  | Не нужно синхронизировать версии      |
| **Проще разработка**  | UI Kit и Shell в одном месте          |
| **Быстрее старт**     | Нет need настраивать отдельный CI/CD  |
| **Легче рефакторинг** | Можно менять API без semver headaches |
| **Меньше оверхеда**   | Один package.json, один build process |

## Структура

```bash
src/admin-ui-kit/
├── components/
│   ├── layout/
│   │   ├── admin-page.ts
│   │   ├── admin-header.ts
│   │   ├── admin-sidebar.ts
│   │   ├── admin-context-sidebar.ts
│   │   └── admin-dashboard-grid.ts
│   ├── widgets/
│   │   ├── status-widget.ts
│   │   ├── kpi-widget.ts
│   │   ├── timeseries-widget.ts
│   │   ├── table-widget.ts
│   │   └── activity-feed-widget.ts
│   ├── states/
│   │   ├── loading-state.ts
│   │   ├── error-state.ts
│   │   ├── stale-state.ts
│   │   └── empty-state.ts
│   └── actions/
│       ├── admin-button.ts
│       ├── admin-modal.ts
│       └── admin-confirm-dialog.ts
├── styles/
│   ├── tokens.css           # @import "@faf/foundations/styles"
│   └── components.css       # Общие стили
├── utils/
│   ├── correlation.ts
│   └── polling.ts
├── index.ts                 # Точка входа
└── package.json
```

## Использование токенов

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

## Временные компоненты

Компоненты, которых нет в @faf/components, создаём здесь:

| Компонент             | Аналог в @faf/components  | Статус    |
| --------------------- | ------------------------- | --------- |
| `<admin-sidebar>`     | Нет                       | Временный |
| `<kpi-widget>`        | MetricCard (планируется)  | Временный |
| `<timeseries-widget>` | Нет                       | Временный |
| `<status-widget>`     | StatusBadge (планируется) | Временный |

### Миграция

Когда компонент появится в @faf/components:

```diff
// Было
import { KpiWidget } from "@faf/admin-ui-kit";

// Стало
import { MetricCard } from "@faf/components";
```

## План разработки

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

## Тестирование

```bash
cd src/admin-ui-kit
npm run test
```

---

**Статус:** Draft  
**Дата:** 2026-09-20
