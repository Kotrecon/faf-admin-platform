# Как подключить новый сервис

**Целевая аудитория:** Разработчики сервисов, которые хотят добавить административный интерфейс в Faf Admin Platform.

**Время на подключение:** 1-2 дня для базовой dashboard-страницы.

## Обзор

Для подключения сервиса нужно:

1. Реализовать Admin API endpoint для page model
2. Добавить service manifest в Portal BFF
3. Создать page schema (или использовать generic)
4. Протестировать через Admin Shell

```bash
┌─────────────────┐
│  Ваш сервис     │
│  + Admin API    │
└────────┬────────┘
         │ PageModel
         ▼
┌─────────────────┐
│   Portal BFF    │
│   (routing)     │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│   Admin Shell   │
│   (render)      │
└─────────────────┘
```

## Шаг 1: Реализовать Admin API

### Endpoint

Создайте endpoint в вашем сервисе:

```bash
GET /admin/{service}/pages/{page}
```

Пример для Cache Service:

```bash
GET /admin/cache/pages/overview
```

### Response

Верните PageModel v1 (schema + data одним ответом):

```csharp
// .NET example
[HttpGet("/admin/cache/pages/overview")]
public async Task<ActionResult<PageModel>> GetCacheOverview()
{
    var correlationId = HttpContext.TraceIdentifier;

    var pageModel = new PageModel
    {
        SchemaVersion = "1",
        Service = new ServiceInfo
        {
            Slug = "cache",
            Name = "Cache Service",
            Status = "online",
            Environment = "prod"
        },
        Page = new PageInfo
        {
            Slug = "overview",
            Title = "Cache Overview",
            RefreshIntervalSeconds = 10
        },
        Layout = new PageLayout
        {
            Widgets = new[]
            {
                new StatusWidget { ... },
                new KpiWidget { ... },
                new TimeSeriesWidget { ... }
            }
        },
        Data = new
        {
            health = GetHealthData(),
            metrics = GetMetrics(),
            series = GetTimeSeries()
        },
        GeneratedAt = DateTime.UtcNow,
        CorrelationId = correlationId
    };

    return Ok(pageModel);
}
```

### Требования к endpoint

| Требование     | Детали                                     |
| -------------- | ------------------------------------------ |
| Auth           | JWT validation (если есть auth middleware) |
| Permissions    | Проверка `cache.read` или аналогичной      |
| Correlation ID | Передать из header или сгенерировать       |
| Response time  | < 100ms (p95) для dashboard                |
| Size           | < 50KB для типичной страницы               |
| Errors         | Возвращать стандартные HTTP status codes   |
| Logging        | Structured logs с correlation ID           |

### Mock data для разработки

На этапе разработки можно вернуть статические данные:

```csharp
private object GetMockData()
{
    return new
    {
        health = new
        {
            status = "online",
            lastCheck = DateTime.UtcNow.ToString("O"),
            version = "1.0.0-dev",
            uptime = 3600
        },
        metrics = new
        {
            hitRatio = 94.5,
            hits = 12453,
            misses = 721,
            activeKeys = 3847
        },
        series = new
        {
            hitsMisses = new
            {
                labels = new[] { "18:00", "18:05", "18:10", "18:15", "18:20" },
                datasets = new[]
                {
                    new { label = "Hits", data = new[] { 1200, 1350, 1100, 1450, 1300 } },
                    new { label = "Misses", data = new[] { 80, 65, 95, 70, 85 } }
                }
            }
        }
    };
}
```

## Шаг 2: Добавить service manifest в BFF

### Конфигурация Portal BFF

В `appsettings.json` или аналогичном config:

```json
{
  "Services": [
    {
      "slug": "cache",
      "name": "Cache Service",
      "baseUrl": "http://localhost:5001",
      "adminPath": "/admin/cache",
      "status": "online",
      "environment": "dev",
      "pages": [
        {
          "slug": "overview",
          "title": "Cache Overview",
          "route": "/pages/overview",
          "refreshIntervalSeconds": 10,
          "requiredPermissions": ["cache.read"]
        },
        {
          "slug": "keys",
          "title": "Cache Keys",
          "route": "/pages/keys",
          "refreshIntervalSeconds": 30,
          "requiredPermissions": ["cache.read"]
        }
      ]
    },
    {
      "slug": "ai-router",
      "name": "AI Router",
      "baseUrl": "http://localhost:5002",
      "adminPath": "/admin/ai-router",
      "status": "online",
      "environment": "dev",
      "pages": [
        {
          "slug": "overview",
          "title": "AI Router Overview",
          "route": "/pages/overview",
          "refreshIntervalSeconds": 10
        }
      ]
    }
  ]
}
```

### Proxy endpoint в BFF

Пример .NET controller:

```csharp
[ApiController]
[Route("admin-api/pages/{service}/{page}")]
public class PagesController : ControllerBase
{
    private readonly IHttpClientFactory _httpClientFactory;
    private readonly IConfiguration _config;
    private readonly ILogger<PagesController> _logger;

    public PagesController(
        IHttpClientFactory httpClientFactory,
        IConfiguration config,
        ILogger<PagesController> logger)
    {
        _httpClientFactory = httpClientFactory;
        _config = config;
        _logger = logger;
    }

    [HttpGet]
    public async Task<IActionResult> GetPage(
        string service,
        string page,
        CancellationToken ct)
    {
        var correlationId = HttpContext.TraceIdentifier;

        // Найти сервис в конфигурации
        var serviceConfig = _config
            .GetSection("Services")
            .Get<List<ServiceConfig>>()
            .FirstOrDefault(s => s.Slug == service);

        if (serviceConfig == null)
        {
            return NotFound($"Service '{service}' not found");
        }

        // Найти страницу
        var pageConfig = serviceConfig.Pages
            .FirstOrDefault(p => p.Slug == page);

        if (pageConfig == null)
        {
            return NotFound($"Page '{page}' not found for service '{service}'");
        }

        // Создать клиент
        var client = _httpClientFactory.CreateClient();
        client.BaseAddress = new Uri(serviceConfig.BaseUrl);

        // Добавить correlation ID
        client.DefaultRequestHeaders.Add("X-Correlation-ID", correlationId);

        // Запросить page model
        var endpoint = $"{serviceConfig.AdminPath}{pageConfig.Route}";
        _logger.LogInformation("Proxying to {Endpoint}", endpoint);

        var response = await client.GetAsync(endpoint, ct);

        if (!response.IsSuccessStatusCode)
        {
            _logger.LogError("Service returned {Status}", response.StatusCode);
            return StatusCode((int)response.StatusCode, "Service error");
        }

        var content = await response.Content.ReadAsStringAsync(ct);
        return Content(content, "application/json");
    }
}
```

## Шаг 3: Создать page schema

### Минимальная schema для первой страницы

Для начала достаточно 3-6 виджетов:

```json
{
  "layout": {
    "widgets": [
      {
        "type": "status",
        "title": "Service Status",
        "dataPath": "health.status"
      },
      {
        "type": "kpi",
        "title": "Request Rate",
        "dataPath": "metrics.requestRate",
        "unit": "req/s"
      },
      {
        "type": "kpi",
        "title": "Error Rate",
        "dataPath": "metrics.errorRate",
        "unit": "%"
      },
      {
        "type": "kpi",
        "title": "Avg Latency",
        "dataPath": "metrics.avgLatency",
        "unit": "ms"
      },
      {
        "type": "timeSeries",
        "title": "Requests over time",
        "dataPath": "series.requests",
        "labels": ["18:00", "18:05", "18:10", "18:15", "18:20"],
        "datasets": [
          { "label": "Requests", "data": [1200, 1350, 1100, 1450, 1300] }
        ]
      }
    ]
  }
}
```

### Рекомендации

| Совет                    | Пояснение                                |
| ------------------------ | ---------------------------------------- |
| Начните с 1 страницы     | Overview/dashboard достаточно для начала |
| Используйте 3-6 виджетов | Не перегружайте первую версию            |
| Покажите KPI             | 3-4 ключевые метрики сервиса             |
| Добавьте 1 график        | Временной ряд для тренда                 |
| Избегайте таблиц         | Для первой страницы не нужно             |
| Тестируйте на mock data  | Не ждите реальных данных                 |

## Шаг 4: Протестировать

### Проверка endpoint

```bash
# Прямой вызов сервиса
curl http://localhost:5001/admin/cache/pages/overview

# Через BFF
curl http://localhost:5000/admin-api/pages/cache/overview
```

Ожидаемый результат: валидный PageModel JSON.

### Проверка в Admin Shell

1. Запустить Admin Shell
2. Открыть `/admin?service=cache&page=overview`
3. Увидеть:
   - Sidebar с сервисом
   - Status widget
   - KPI widgets
   - TimeSeries chart
   - Polling каждые 10 секунд

### Проверка correlation ID

```bash
# В логах BFF и сервиса должен быть один correlation ID
curl -v http://localhost:5000/admin-api/pages/cache/overview
```

Искать header `X-Correlation-ID` или аналогичный.

## Чек-лист готовности

- [ ] Admin API endpoint возвращает валидный PageModel
- [ ] Service manifest добавлен в BFF конфигурацию
- [ ] Proxy endpoint в BFF работает
- [ ] Admin Shell отображает страницу
- [ ] Polling работает (обновление каждые N секунд)
- [ ] Correlation ID проходит через всю цепочку
- [ ] Error handling работает (сервис недоступен → stale state)
- [ ] Mock data заменены на реальные (или готовы к замене)

## Следующие шаги

После базовой страницы можно добавить:

- [ ] Дополнительные pages (keys, settings, maintenance)
- [ ] Custom module для сложных workflows
- [ ] Generic CRUD table для ресурсов
- [ ] Actions и commands (flush, delete, update)
- [ ] RBAC permissions
- [ ] Audit logging

## Примеры

### Cache Service

См. `samples/cache-service-adapter/`

### AI Router (планируется)

- Страница: Overview
- KPI: request rate, error rate, avg latency, fallback rate
- График: requests over time
- Widget: model distribution

### Prompting (планируется)

- Страница: Overview
- KPI: prompt count, token usage, avg tokens, cache hit ratio
- График: tokens over time

## Поддержка

Вопросы и проблемы:

- Создать issue в `faf-admin-platform` репозитории
- Указать сервис и страницу
- Приложить PageModel JSON (если есть)
- Приложить скриншот ошибки (если есть)

---

**Статус:** Active  
**Версия:** 1.0  
**Последнее обновление:** 2026-09-20
