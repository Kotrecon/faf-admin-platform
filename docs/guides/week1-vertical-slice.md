# Неделя 1: Vertical Slice

**Цель:** Доказать основной архитектурный контракт платформы.

```bash
Service → PageModel (schema + data) → Portal BFF → Admin Shell → Dashboard
```

## 🎯 Результат недели

К концу недели должно работать:

- [ ] Admin Shell загружает страницу через Portal BFF
- [ ] Portal BFF проксирует PageModel от Cache Service Admin API
- [ ] Cache Service возвращает один ответ: schema + current data
- [ ] Shell строит экран через общие renderer-компоненты
- [ ] На экране есть: status, 4 KPI, один график
- [ ] Данные обновляются раз в 10 секунд (polling)
- [ ] При ошибке polling старые данные остаются как stale
- [ ] Есть ручное обновление (refresh button)
- [ ] Correlation ID проходит через всю цепочку и виден в логах
- [ ] Смена route останавливает polling
- [ ] Нет прямой зависимости shell от Cache Service-specific HTML/CSS/JS

## 📋 Scope

### Делаем

| Компонент            | Минимум                                                                    |
| -------------------- | -------------------------------------------------------------------------- |
| **Admin Shell**      | Одно SPA, main layout, упрощённый sidebar, query-param router              |
| **Navigation**       | Два route: `?page=system-overview` и `?service=cache&page=overview`        |
| **Portal BFF**       | Один .NET service, static route map, proxy к Cache Admin API               |
| **Service Registry** | Жёсткая конфигурация: System + Cache                                       |
| **Авторизация**      | Mock user / существующая dev auth (не строить JWT инфраструктуру с нуля)   |
| **Page Endpoint**    | Один: Cache Overview, schema + data вместе                                 |
| **Schema Renderer**  | Только 3 widget type: `status`, `kpi`, `timeSeries`                        |
| **Admin UI Kit**     | 4-6 базовых компонентов (layout, card, status badge, KPI, chart container) |
| **Polling**          | 10 секунд для Cache Overview                                               |
| **States**           | Loading spinner, error block, stale indicator                              |
| **Telemetry**        | Correlation ID + structured logs на BFF и API                              |
| **Cache Operations** | Только read-only (никаких flush, write, delete)                            |

### Не делаем

- [ ] JWT + refresh flow с нуля
- [ ] Реальный RBAC engine
- [ ] Dynamic service discovery
- [ ] Plugin registry / manifests
- [ ] Remote microfrontend / Module Federation
- [ ] OpenAPI UI generator
- [ ] Generic CRUD renderer
- [ ] Key Explorer / cache browser
- [ ] Cache write / delete / flush operations
- [ ] Audit viewer
- [ ] CSV export
- [ ] i18n / theme switcher / full a11y suite
- [ ] CSP strict-dynamic
- [ ] OTel RUM
- [ ] WebSocket / SSE

## 📐 Минимальный контракт

### PageModel v1

```json
{
  "schemaVersion": "1",
  "service": {
    "slug": "cache",
    "name": "Cache Service",
    "status": "online"
  },
  "page": {
    "slug": "overview",
    "title": "Cache Overview",
    "refreshIntervalSeconds": 10
  },
  "layout": {
    "widgets": [
      { "type": "status", "title": "Service Status" },
      {
        "type": "kpi",
        "title": "Hit Ratio",
        "dataPath": "metrics.hitRatio",
        "unit": "%"
      },
      {
        "type": "kpi",
        "title": "Hits",
        "dataPath": "metrics.hits",
        "unit": ""
      },
      {
        "type": "kpi",
        "title": "Misses",
        "dataPath": "metrics.misses",
        "unit": ""
      },
      {
        "type": "kpi",
        "title": "Active Keys",
        "dataPath": "metrics.activeKeys",
        "unit": ""
      },
      {
        "type": "timeSeries",
        "title": "Hits/Misses over time",
        "dataPath": "series.hitsMisses"
      }
    ]
  },
  "data": {
    "health": {
      "status": "online",
      "lastCheck": "2026-09-20T18:00:00Z"
    },
    "metrics": {
      "hitRatio": 94.5,
      "hits": 12453,
      "misses": 721,
      "activeKeys": 3847
    },
    "series": {
      "hitsMisses": {
        "labels": ["18:00", "18:05", "18:10", "18:15", "18:20"],
        "datasets": [
          { "label": "Hits", "data": [1200, 1350, 1100, 1450, 1300] },
          { "label": "Misses", "data": [80, 65, 95, 70, 85] }
        ]
      }
    }
  },
  "generatedAt": "2026-09-20T18:20:00Z",
  "correlationId": "abc123-def456"
}
```

### Widget Types (разрешённые)

| Type         | Поля                                                    | Описание         |
| ------------ | ------------------------------------------------------- | ---------------- |
| `status`     | `title`, `status` (online/degraded/offline)             | Статус сервиса   |
| `kpi`        | `title`, `dataPath`, `value`, `unit`, `threshold` (opt) | Числовая метрика |
| `timeSeries` | `title`, `dataPath`, `labels`, `datasets`               | Линейный график  |

## 🗓️ План по дням

### День 1 — Контракт и скелет

**Задачи:**

- [ ] Создать solution/repository boundaries: `admin-shell`, `admin-portal-bff`, `cache-service`
- [ ] Описать `PageModel v1` DTO/TypeScript interface
- [ ] Создать Cache Admin endpoint: `GET /admin/cache/dashboard`
  - Возвращает фиксированную schema + mock metrics
- [ ] Создать Portal BFF endpoint: `GET /admin-api/pages/cache/overview`
  - Проксирует ответ от Cache Admin API
- [ ] Добавить correlation ID:
  - Принять из header или сгенерировать
  - Передать дальше в сервис
  - Записать в structured logs

**Результат дня:**  
Запрос через BFF возвращает один корректный `PageModel`.

**Проверка:**

```bash
curl http://localhost:5000/admin-api/pages/cache/overview
```

Видим валидный JSON с schema + mock data.

---

### День 2 — Shell и routing

**Задачи:**

- [ ] Создать один admin frontend (Vanilla JS)
- [ ] Реализовать shell layout:
  - Header (placeholder)
  - Primary sidebar
  - Main content area
- [ ] Добавить упрощённый sidebar:
  - `System Overview`
  - `Cache Service`
- [ ] Реализовать query-param routing:
  - `?page=system-overview`
  - `?service=cache&page=overview`
- [ ] Рендерить страницу Cache Overview по URL
- [ ] Сделать loading/error/empty state компоненты

**Результат дня:**  
Пользователь открывает `?service=cache&page=overview` и получает page model через BFF.

**Проверка:**

- Открыть браузер, перейти на `/admin?service=cache&page=overview`
- Увидеть sidebar с двумя пунктами
- Увидеть loading state → PageModel загружается → placeholder content

---

### День 3 — Schema renderer

**Задачи:**

- [ ] Создать renderer layout (проход по `layout.widgets[]`)
- [ ] Реализовать `status` widget:
  - Online / Degraded / Offline badge
- [ ] Реализовать `kpi` widget:
  - Title, value, unit
- [ ] Реализовать `timeSeries` widget:
  - Простой линейный график (Chart.js / lightweight chart / SVG)
- [ ] Подключить 4 KPI: hit ratio, hits, misses, active keys
- [ ] Подключить один график hits/misses

**Результат дня:**  
UI строится из schema, а не из hardcoded Cache-specific JSX/HTML.

**Проверка:**

- Изменить mock data в сервисе → UI обновляется
- Изменить schema (добавить 5-й KPI) → UI рендерит новый KPI без изменений кода Shell

---

### День 4 — Polling и деградация

**Задачи:**

- [ ] Polling page endpoint каждые 10 секунд
- [ ] Manual refresh button
- [ ] Timestamp "Updated X sec ago"
- [ ] Stale state при ошибке polling (старые данные + warning)
- [ ] Abort текущего fetch при смене route
- [ ] Остановка таймера при уходе со страницы (cleanup)
- [ ] Проверить поведение при:
  - Временной ошибке API
  - Долгой ошибке API
  - Восстановлении API

**Результат дня:**  
Dashboard живой и корректно деградирует.

**Проверка:**

- Открыть DevTools Network
- Остановить сервис → увидеть stale state
- Запустить сервис → данные обновляются
- Перейти на другой route → polling останавливается

---

### День 5 — Демонстрация и фиксация

**Задачи:**

- [ ] Добавить второй статический route: System Overview (placeholder)
- [ ] Проверить deep link и browser Back/Forward
- [ ] Сделать минимальные integration tests:
  - BFF endpoint возвращает PageModel
  - Cache endpoint возвращает PageModel
- [ ] Один frontend smoke test:
  - Cache route → status/KPI/chart рендерится
- [ ] Описать "как подключить новый сервис" в `docs/guides/connect-new-service.md`
- [ ] Сделать короткое demo:
  - GIF / скриншоты / 2-минутное видео
- [ ] Написать этот документ до конца

**Результат дня:**  
Есть повторяемый вертикальный срез, а не одноразовый экран.

---

## ✅ Definition of Done

Прототип готов, если одновременно выполняется:

- [ ] Открывается URL: `/admin?service=cache&page=overview`
- [ ] Admin Shell загружает страницу только через Portal BFF
- [ ] Portal BFF проксирует PageModel от Cache Service Admin API
- [ ] Cache Service возвращает **один** ответ: schema + current data
- [ ] Shell строит экран через общие renderer-компоненты
- [ ] На экране есть: status, 4 KPI, 1 график
- [ ] Данные обновляются раз в 10 секунд
- [ ] При временной ошибке старые данные остаются видимыми как stale
- [ ] Есть ручное обновление
- [ ] Correlation ID можно найти в логах BFF и Cache Service
- [ ] Смена route останавливает polling
- [ ] Нет прямой зависимости shell от Cache Service-specific HTML/CSS/JS
- [ ] Добавление второго сервиса потребует нового route/config/page model, а не переписывания shell

## 📊 Метрики успеха

| Метрика                   | Целевое значение      |
| ------------------------- | --------------------- |
| Время от BFF до Shell     | < 200ms (p95)         |
| Время от Cache API до BFF | < 100ms (p95)         |
| Размер PageModel response | < 50KB                |
| First contentful paint    | < 1.5s                |
| Polling interval          | 10 секунд (стабильно) |
| Error rate (week 1)       | < 1% запросов         |

## 🚀 Следующие шаги

После успешного завершения недели 1:

1. Добавить второй сервис (AI Router или Prompting)
2. Вынести schema отдельно от data (если нужно)
3. Добавить generic CRUD renderer
4. Реализовать базовый RBAC/permissions
5. Добавить Cache Key Explorer (как custom module)
6. Реализовать Cache Maintenance workflow

---

**Статус:** В работе  
**Начало:** 2026-09-21  
**Конец:** 2026-09-25  
**Ответственный:** Platform Team
