# Tech Stack

**Faf Admin Platform**.

## Стек технологий

| Компонент            | Технология                                | Версия  |
| -------------------- | ----------------------------------------- | ------- |
| **Admin Shell**      | Vanilla JavaScript                        | ES2022+ |
| **Portal BFF**       | .NET                                      | 10      |
| **Admin UI Kit**     | Web Components на основе @faf/foundations | Native  |
| **Styling**          | CSS Tokens + Cascade Layers               | Native  |
| **Build (Frontend)** | Vite                                      | Latest  |
| **Package Manager**  | npm / pnpm                                | Latest  |

## Почему Vanilla JavaScript?

| Причина                       | Пояснение                                   |
| ----------------------------- | ------------------------------------------- |
| Нет зависимости от фреймворка | Меньше bundle size, нет vendor lock-in      |
| Web Components нативные       | Браузеры поддерживают из коробки            |
| Простота                      | Нет need учить React/Vue/Svelte для команды |
| Долгосрочная поддержка        | ES-модули и Web Components — стандарт       |
| Легкость интеграции           | Можно подключить к любому проекту           |
| Performance                   | Нет overhead фреймворка                     |

## Почему .NET 10?

| Причина      | Пояснение                  |
| ------------ | -------------------------- |
| Latest LTS   | Долгосрочная поддержка     |
| Performance  | Улучшения в ASP.NET Core   |
| Minimal APIs | Простота для BFF layer     |
| Native AOT   | Опционально для production |

## Минимальные требования

| Требование | Значение                                   |
| ---------- | ------------------------------------------ |
| Browser    | Chrome/Edge 100+, Firefox 100+, Safari 15+ |
| .NET SDK   | 10.0+                                      |
| Node.js    | 18+ (для build tools)                      |
| ES Version | ES2022+                                    |

## Структура Admin Shell

```bash
src/admin-shell/
├── index.html
├── package.json
├── vite.config.js
├── src/
│   ├── main.js
│   ├── router.js
│   ├── api.js
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
│   │   ├── tokens.css
│   │   ├── base.css
│   │   └── components.css
│   └── utils/
│       ├── fetch.js
│       └── correlation.js
└── tests/
```

## Структура Portal BFF

```bash
src/admin-portal-bff/
├── Faf.AdminPlatform.Bff.csproj
├── Program.cs
├── appsettings.json
├── Controllers/
│   └── PagesController.cs
├── Models/
│   └── PageModel.cs
└── Services/
    └── ServiceRegistry.cs
```

## Сборка

### Admin Shell

```bash
cd src/admin-shell
npm install
npm run dev      # Development server (Vite)
npm run build    # Production build
```

### Portal BFF

```bash
cd src/admin-portal-bff
dotnet restore
dotnet run       # Development
dotnet publish   # Production
```

## Пример кода

### Vanilla JS Component

```javascript
// components/widgets/kpi-widget.js
export class KpiWidget extends HTMLElement {
  constructor() {
    super();
    this.attachShadow({ mode: "open" });
  }

  static get observedAttributes() {
    return ["title", "value", "unit"];
  }

  attributeChangedCallback(name, oldValue, newValue) {
    this.render();
  }

  render() {
    const title = this.getAttribute("title") || "";
    const value = this.getAttribute("value") || "0";
    const unit = this.getAttribute("unit") || "";

    this.shadowRoot.innerHTML = `
      <style>
        .kpi-card {
          padding: 1rem;
          border-radius: 8px;
          background: var(--bg-card);
        }
        .kpi-title {
          font-size: 0.875rem;
          color: var(--text-muted);
        }
        .kpi-value {
          font-size: 2rem;
          font-weight: 600;
          color: var(--text-primary);
        }
        .kpi-unit {
          font-size: 1rem;
          color: var(--text-muted);
        }
      </style>
      <div class="kpi-card">
        <div class="kpi-title">${title}</div>
        <div class="kpi-value">${value}<span class="kpi-unit">${unit}</span></div>
      </div>
    `;
  }
}

customElements.define("kpi-widget", KpiWidget);
```

### .NET 10 Minimal API

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddHttpClient();

var app = builder.Build();

app.MapGet("/admin-api/pages/{service}/{page}", async (
    string service,
    string page,
    IHttpClientFactory httpClientFactory,
    IConfiguration config,
    ILogger<PagesController> logger,
    CancellationToken ct) =>
{
    var correlationId = HttpContext.TraceIdentifier;

    var serviceConfig = config.GetSection("Services")
        .Get<List<ServiceConfig>>()
        .FirstOrDefault(s => s.Slug == service);

    if (serviceConfig == null)
        return Results.NotFound($"Service '{service}' not found");

    var client = httpClientFactory.CreateClient();
    client.BaseAddress = new Uri(serviceConfig.BaseUrl);
    client.DefaultRequestHeaders.Add("X-Correlation-ID", correlationId);

    var response = await client.GetAsync($"/admin/{service}/pages/{page}", ct);

    if (!response.IsSuccessStatusCode)
        return Results.StatusCode((int)response.StatusCode);

    var content = await response.Content.ReadAsStringAsync(ct);
    return Results.Content(content, "application/json");
});

app.Run();
```

---

**Статус:** Approved  
**Дата:** 2026-09-20  
**Стек:** .NET 10 + Vanilla JavaScript (ES2022+)
