# Faf.AdminPlatform.Bff

**Portal BFF (Backend for Frontend) для Faf Admin Platform**.

Тонкий слой для агрегации и маршрутизации запросов к сервисным Admin API.

## 📦 Структура

```bash
src/admin-portal-bff/
├── Faf.AdminPlatform.Bff.csproj
├── Program.cs                    # Minimal API entry point
├── appsettings.json              # Service registry config
├── Controllers/
│   └── PagesController.cs        # Proxy to service admin APIs
├── Models/
│   ├── PageModel.cs              # PageModel v1 DTO
│   └── ServiceConfig.cs          # Service manifest DTO
└── Services/
    └── ServiceRegistry.cs        # Service discovery (static config)
```

## 🚀 Быстрый старт

### Установка .NET 10 SDK

```bash
# Проверка версии
dotnet --version

# Должно быть: 10.0.x или новее
```

### Запуск

```bash
cd src/admin-portal-bff
dotnet restore
dotnet run
```

По умолчанию: `http://localhost:5000`

## 📚 Endpoints

### GET /admin-api/pages/{service}/{page}

Проксирует запрос к сервисному Admin API.

**Пример:**

```bash
curl http://localhost:5000/admin-api/pages/cache/overview
```

**Response:** PageModel v1 (schema + data)

### GET /admin-api/services

Возвращает список зарегистрированных сервисов.

**Response:**

```json
[
  {
    "slug": "cache",
    "name": "Cache Service",
    "status": "online",
    "environment": "dev",
    "pages": ["overview", "keys"]
  },
  {
    "slug": "ai-router",
    "name": "AI Router",
    "status": "online",
    "environment": "dev",
    "pages": ["overview"]
  }
]
```

## ⚙️ Конфигурация

### appsettings.json

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
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
          "refreshIntervalSeconds": 10
        },
        {
          "slug": "keys",
          "title": "Cache Keys",
          "route": "/pages/keys",
          "refreshIntervalSeconds": 30
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

## 🏗️ Архитектура

```bash
┌─────────────────────────────────────────────────────────────┐
│                      Admin Shell                             │
│  (Vanilla JS + Web Components)                               │
└────────────────────────────┬──────────────────────────────────┘
                             │
                             │ GET /admin-api/pages/{service}/{page}
                             ▼
┌─────────────────────────────────────────────────────────────┐
│                   Portal BFF                                 │
│  • Service registry                                          │
│  • Routing to service admin APIs                             │
│  • Correlation ID forwarding                                 │
│  • Error normalization                                       │
└────────────────────────────┬──────────────────────────────────┘
                             │
      ┌──────────────────────┼───────────────────────────────┐
      ▼                      ▼                               ▼
┌───────────────┐    ┌───────────────┐              ┌────────────────┐
│ Cache Service │    │ AI Router     │              │ Prompting      │
│ Admin API     │    │ Admin API     │              │ Admin API      │
└───────────────┘    └───────────────┘              └────────────────┘
```

## 🔧 Correlation ID

BFF автоматически генерирует и передаёт correlation ID:

```csharp
var correlationId = HttpContext.TraceIdentifier;
client.DefaultRequestHeaders.Add("X-Correlation-ID", correlationId);
```

Ищите `X-Correlation-ID` в логах всех сервисов цепочки.

## 🧪 Тестирование

### Проверка endpoint

```bash
# Список сервисов
curl http://localhost:5000/admin-api/services

# PageModel
curl http://localhost:5000/admin-api/pages/cache/overview
```

### Integration tests

```bash
cd src/admin-portal-bff
dotnet test
```

## 📄 Models

### PageModel v1

```csharp
public class PageModel
{
    public string SchemaVersion { get; set; } = "1";
    public ServiceInfo Service { get; set; }
    public PageInfo Page { get; set; }
    public PageLayout Layout { get; set; }
    public object Data { get; set; }
    public DateTime GeneratedAt { get; set; }
    public string CorrelationId { get; set; }
}
```

### ServiceConfig

```csharp
public class ServiceConfig
{
    public string Slug { get; set; }
    public string Name { get; set; }
    public string BaseUrl { get; set; }
    public string AdminPath { get; set; }
    public string Status { get; set; }
    public string Environment { get; set; }
    public List<PageConfig> Pages { get; set; }
}
```

## 🚀 Развёртывание

### Production build

```bash
cd src/admin-portal-bff
dotnet publish -c Release -o ./publish
```

### Docker (опционально)

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:10.0 AS base
WORKDIR /app
EXPOSE 80

FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
WORKDIR /src
COPY . .
RUN dotnet publish -c Release -o /app/publish

FROM base AS final
WORKDIR /app
COPY --from=build /app/publish .
ENTRYPOINT ["dotnet", "Faf.AdminPlatform.Bff.dll"]
```

## 📊 Метрики

| Метрика             | Значение    |
| ------------------- | ----------- |
| Response time (p95) | < 50ms      |
| Throughput          | 1000+ req/s |
| Error rate          | < 0.1%      |

## 📄 Лицензия

Internal use only.

---

**Статус:** 0.1.0-alpha  
**Дата:** 2026-09-20  
**Стек:** .NET 10 + Minimal APIs
