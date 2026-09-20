# PageModel v1

**Статус:** Draft  
**Версия:** 1.0.0-alpha  
**Назначение:** Декларативная модель сервисной страницы для Admin Platform.

## Обзор

PageModel — это ответ сервиса на запрос административной страницы. Он содержит:

- **schema** — декларативное описание layout и виджетов
- **data** — актуальные данные для рендеринга
- **metadata** — service info, page info, refresh policy, correlation

Сервис возвращает schema и data **одним ответом** на этапе прототипа.

## Endpoint

```bash
GET /admin-api/services/{service}/pages/{page}
```

Пример:

```bash
GET /admin-api/services/cache/pages/overview
```

## Response Model

```typescript
interface PageModel {
  // Версия контракта
  schemaVersion: "1";

  // Информация о сервисе
  service: {
    slug: string; // "cache", "ai-router", "prompting"
    name: string; // "Cache Service", "AI Router"
    status: "online" | "degraded" | "offline";
    environment?: string; // "dev", "staging", "prod"
  };

  // Информация о странице
  page: {
    slug: string; // "overview", "keys", "maintenance"
    title: string; // "Cache Overview"
    description?: string; // Опциональное описание
    refreshIntervalSeconds: number; // 10, 30, 60
    requiredPermissions?: string[]; // ["cache.read", "cache.admin"]
  };

  // Декларативный layout
  layout: {
    widgets: WidgetDescriptor[];
  };

  // Актуальные данные
  data: {
    health?: HealthData;
    metrics?: Record<string, MetricData>;
    series?: Record<string, TimeSeriesData>;
    tables?: Record<string, TableData>;
    [key: string]: unknown; // Service-specific data
  };

  // Metadata
  generatedAt: string; // ISO 8601 timestamp
  correlationId: string; // Для tracing
  warnings?: string[]; // Опциональные предупреждения
  partialErrors?: {
    // Ошибки по отдельным виджетам
    widgetId: string;
    error: string;
  }[];
}
```

## Widget Types

### Status Widget

Отображает статус сервиса.

```typescript
interface StatusWidget {
  type: "status";
  id?: string;
  title: string;
  dataPath: "health.status"; // Путь к данным
  status: "online" | "degraded" | "offline";
  lastCheck?: string; // ISO 8601
  message?: string; // Опциональное сообщение
}
```

**Пример данных:**

```json
{
  "type": "status",
  "title": "Service Status",
  "dataPath": "health.status",
  "status": "online",
  "lastCheck": "2026-09-20T18:00:00Z",
  "message": "All systems operational"
}
```

---

### KPI Widget

Числовая метрика с опциональным threshold.

```typescript
interface KpiWidget {
  type: "kpi";
  id?: string;
  title: string;
  dataPath: string; // "metrics.hitRatio"
  value: number;
  unit?: string; // "%", "ms", "req/s", ""
  format?: "number" | "percentage" | "duration" | "bytes";
  threshold?: {
    warning: number;
    critical: number;
    operator: "lt" | "gt" | "lte" | "gte";
  };
  trend?: {
    value: number;
    direction: "up" | "down" | "flat";
  };
}
```

**Пример данных:**

```json
{
  "type": "kpi",
  "title": "Hit Ratio",
  "dataPath": "metrics.hitRatio",
  "value": 94.5,
  "unit": "%",
  "format": "percentage",
  "threshold": {
    "warning": 90,
    "critical": 80,
    "operator": "lt"
  },
  "trend": {
    "value": 2.3,
    "direction": "up"
  }
}
```

---

### TimeSeries Widget

Линейный график временного ряда.

```typescript
interface TimeSeriesWidget {
  type: "timeSeries";
  id?: string;
  title: string;
  dataPath: string; // "series.hitsMisses"
  labels: string[]; // ["18:00", "18:05", ...]
  datasets: {
    label: string;
    data: number[];
    color?: string; // Hex color (опционально)
  }[];
  yAxis?: {
    label?: string;
    min?: number;
    max?: number;
  };
  xLabel?: string; // "Time"
  yLabel?: string; // "Requests"
}
```

**Пример данных:**

```json
{
  "type": "timeSeries",
  "title": "Hits/Misses over time",
  "dataPath": "series.hitsMisses",
  "labels": ["18:00", "18:05", "18:10", "18:15", "18:20"],
  "datasets": [
    {
      "label": "Hits",
      "data": [1200, 1350, 1100, 1450, 1300]
    },
    {
      "label": "Misses",
      "data": [80, 65, 95, 70, 85]
    }
  ],
  "xLabel": "Time",
  "yLabel": "Requests"
}
```

---

### Distribution Widget (будущее)

Histogram/distribution, например TTL distribution.

```typescript
interface DistributionWidget {
  type: "distribution";
  id?: string;
  title: string;
  dataPath: string;
  buckets: {
    label: string;
    value: number;
  }[];
}
```

---

### Table Widget (будущее)

Read-only агрегированная таблица.

```typescript
interface TableWidget {
  type: "table";
  id?: string;
  title: string;
  dataPath: string;
  columns: {
    key: string;
    label: string;
    format?: "text" | "number" | "percentage" | "duration" | "status";
  }[];
  rows: Record<string, unknown>[];
}
```

---

### Resource Table Widget (будущее)

Generic CRUD table с actions.

```typescript
interface ResourceTableWidget {
  type: "resourceTable";
  id?: string;
  title: string;
  resource: string;
  endpoint: string;
  columns: ColumnDescriptor[];
  actions: ActionDescriptor[];
  pagination: {
    defaultLimit: number;
    limits: number[];
  };
}
```

---

## Data Types

### HealthData

```typescript
interface HealthData {
  status: "online" | "degraded" | "offline";
  lastCheck: string; // ISO 8601
  version?: string;
  uptime?: number; // seconds
  checks?: {
    name: string;
    status: "pass" | "warn" | "fail";
    message?: string;
  }[];
}
```

### MetricData

```typescript
interface MetricData {
  value: number;
  unit?: string;
  timestamp?: string;
}
```

### TimeSeriesData

```typescript
interface TimeSeriesData {
  labels: string[];
  datasets: {
    label: string;
    data: number[];
    color?: string;
  }[];
}
```

### TableData

```typescript
interface TableData {
  columns: {
    key: string;
    label: string;
    format?: string;
  }[];
  rows: Record<string, unknown>[];
  totalCount?: number;
}
```

---

## Security Rules

Schema **не может** содержать:

- ❌ Executable JavaScript
- ❌ Inline event handlers
- ❌ Произвольный HTML
- ❌ CSS от сервиса
- ❌ Произвольные external script URLs
- ❌ Arbitrary fetch URLs
- ❌ Выражения, которые исполняются как код

Schema содержит **только** allowlisted декларативные описания.

---

## Поведение при ошибках

| Ситуация              | Поведение Shell                                                                   |
| --------------------- | --------------------------------------------------------------------------------- |
| Unknown schemaVersion | Safe error state с указанием compatibility problem                                |
| Unknown widget type   | Safe placeholder + telemetry event                                                |
| Missing data path     | Виджет показывает no-data/error state                                             |
| Missing capability    | Виджет/страница не отображается                                                   |
| Missing permission    | UI скрывает элемент; direct request получает 403                                  |
| Invalid schema        | Страница не рендерится частично небезопасно; показывается schema validation error |
| Partial data error    | `partialErrors` в response, affected widgets показывают error state               |

---

## Пример полного ответа

```json
{
  "schemaVersion": "1",
  "service": {
    "slug": "cache",
    "name": "Cache Service",
    "status": "online",
    "environment": "prod"
  },
  "page": {
    "slug": "overview",
    "title": "Cache Overview",
    "description": "Real-time cache performance metrics",
    "refreshIntervalSeconds": 10,
    "requiredPermissions": ["cache.read"]
  },
  "layout": {
    "widgets": [
      {
        "type": "status",
        "id": "w1",
        "title": "Service Status",
        "dataPath": "health.status",
        "status": "online",
        "lastCheck": "2026-09-20T18:00:00Z",
        "message": "All systems operational"
      },
      {
        "type": "kpi",
        "id": "w2",
        "title": "Hit Ratio",
        "dataPath": "metrics.hitRatio",
        "value": 94.5,
        "unit": "%",
        "format": "percentage",
        "threshold": {
          "warning": 90,
          "critical": 80,
          "operator": "lt"
        },
        "trend": {
          "value": 2.3,
          "direction": "up"
        }
      },
      {
        "type": "kpi",
        "id": "w3",
        "title": "Hits",
        "dataPath": "metrics.hits",
        "value": 12453,
        "unit": "",
        "format": "number"
      },
      {
        "type": "kpi",
        "id": "w4",
        "title": "Misses",
        "dataPath": "metrics.misses",
        "value": 721,
        "unit": "",
        "format": "number"
      },
      {
        "type": "kpi",
        "id": "w5",
        "title": "Active Keys",
        "dataPath": "metrics.activeKeys",
        "value": 3847,
        "unit": "",
        "format": "number"
      },
      {
        "type": "timeSeries",
        "id": "w6",
        "title": "Hits/Misses over time",
        "dataPath": "series.hitsMisses",
        "labels": ["18:00", "18:05", "18:10", "18:15", "18:20"],
        "datasets": [
          {
            "label": "Hits",
            "data": [1200, 1350, 1100, 1450, 1300]
          },
          {
            "label": "Misses",
            "data": [80, 65, 95, 70, 85]
          }
        ],
        "xLabel": "Time",
        "yLabel": "Requests"
      }
    ]
  },
  "data": {
    "health": {
      "status": "online",
      "lastCheck": "2026-09-20T18:00:00Z",
      "version": "1.2.3",
      "uptime": 864000
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

---

## Версионирование

| Версия      | Изменения                              |
| ----------- | -------------------------------------- |
| 1.0.0-alpha | Initial draft: status, kpi, timeSeries |
| 1.1.0       | Distribution widget (planned)          |
| 1.2.0       | Table widget (planned)                 |
| 2.0.0       | Breaking changes (TBD)                 |

---

**Контакт:** Platform Team  
**Последнее обновление:** 2026-09-20
