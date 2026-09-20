# Faf Admin Shell

**Frontend-приложение Faf Admin Platform**.

Vanilla JavaScript + Web Components для рендеринга административных интерфейсов.

## 📦 Структура

```bash
src/admin-shell/
├── index.html                  # Точка входа
├── package.json
├── vite.config.js
├── src/
│   ├── main.js                 # Entry point
│   ├── router.js               # Query-param routing
│   ├── api.js                  # HTTP client к BFF
│   ├── components/
│   │   ├── layout/
│   │   │   ├── admin-page.js
│   │   │   ├── admin-sidebar.js
│   │   │   └── admin-header.js
│   │   ├── widgets/
│   │   │   ├── status-widget.js
│   │   │   ├── kpi-widget.js
│   │   │   └── timeseries-widget.js
│   │   └── states/
│   │       ├── loading-state.js
│   │       ├── error-state.js
│   │       └── stale-state.js
│   ├── styles/
│   │   ├── tokens.css          # @import "@faf/foundations/styles"
│   │   ├── base.css
│   │   └── components.css
│   └── utils/
│       ├── fetch.js            # HTTP client с correlation ID
│       └── polling.js          # Polling manager
└── tests/
```

## 🚀 Быстрый старт

### Установка

```bash
cd src/admin-shell
npm install
```

### Запуск (development)

```bash
npm run dev
```

Открыть: `http://localhost:3000`

### Production build

```bash
npm run build
```

## 📚 Компоненты

### Layout

- `<admin-page>` — Layout страницы
- `<admin-sidebar>` — Primary sidebar
- `<admin-header>` — Header

### Widgets

- `<status-widget>` — Статус сервиса
- `<kpi-widget>` — KPI метрика
- `<timeseries-widget>` — Временной ряд

### States

- `<loading-state>` — Состояние загрузки
- `<error-state>` — Состояние ошибки
- `<stale-state>` — Устаревшие данные

## 🎨 Использование

### Базовая страница

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Faf Admin Platform</title>
    <link rel="stylesheet" href="/src/styles/tokens.css" />
  </head>
  <body>
    <admin-page>
      <admin-sidebar slot="sidebar">
        <!-- navigation -->
      </admin-sidebar>

      <admin-dashboard-grid slot="content">
        <status-widget status="online" title="Service Status"></status-widget>
        <kpi-widget title="Hit Ratio" value="94.5" unit="%"></kpi-widget>
        <timeseries-widget
          title="Hits/Misses"
          data='{"labels":[...],"datasets":[...]}'
        ></timeseries-widget>
      </admin-dashboard-grid>
    </admin-page>

    <script type="module" src="/src/main.js"></script>
  </body>
</html>
```

### JavaScript

```javascript
// src/main.js
import "@faf/admin-ui-kit";
import { router } from "./router.js";
import { pollingManager } from "./utils/polling.js";

// Инициализация роутера
router.init();

// Обработка query params: ?service=cache&page=overview
const params = new URLSearchParams(window.location.search);
const service = params.get("service");
const page = params.get("page");

if (service && page) {
  // Загрузка PageModel через BFF
  const response = await fetch(`/admin-api/pages/${service}/${page}`);
  const pageModel = await response.json();

  // Рендеринг через schema renderer
  renderPage(pageModel);

  // Запуск polling
  pollingManager.start(pageModel.page.refreshIntervalSeconds);
}
```

## 🏗️ Архитектура

```bash
┌─────────────────────────────────────────────────────────────┐
│                      Admin Shell                             │
│  • Vanilla JavaScript (ES2022+)                              │
│  • Web Components (native)                                   │
│  • Query-param routing                                       │
│  • Schema renderer                                           │
│  • Polling manager (10 sec default)                          │
└────────────────────────────┬──────────────────────────────────┘
                             │
                             │ GET /admin-api/pages/{service}/{page}
                             ▼
┌─────────────────────────────────────────────────────────────┐
│                   Portal BFF                                 │
└─────────────────────────────────────────────────────────────┘
```

## 🔧 Конфигурация

### vite.config.js

```javascript
import { defineConfig } from "vite";

export default defineConfig({
  server: {
    port: 3000,
    proxy: {
      "/admin-api": {
        target: "http://localhost:5000",
        changeOrigin: true,
      },
    },
  },
  build: {
    outDir: "dist",
    rollupOptions: {
      input: {
        main: "index.html",
      },
    },
  },
});
```

## 🧪 Тестирование

### Unit-тесты

```bash
npm run test
```

### E2E-тесты

```bash
npm run test:e2e
```

## 📊 Метрики

| Метрика                | Значение |
| ---------------------- | -------- |
| Bundle size (gzip)     | < 50KB   |
| First contentful paint | < 1.5s   |
| Time to interactive    | < 2.5s   |
| Lighthouse score       | > 90     |

## 📄 Лицензия

Internal use only.

---

**Статус:** 0.1.0-alpha  
**Дата:** 2026-09-20  
**Стек:** Vanilla JavaScript (ES2022+) + Web Components
