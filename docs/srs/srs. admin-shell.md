# Документ 4. Архитектурные принципы, Admin Shell и Portal BFF

**Версия:** 0.2  
**Статус:** Зафиксировано для прототипа  
**Назначение:** определяет единый вход во внутреннюю административную платформу, правила подключения сервисов, генерации сервисных страниц и границы ответственности между Shell, BFF и service admin APIs.

---

## 1. Архитектурная цель

Создать единую внутреннюю административную платформу для системы «помощник режиссёра мероприятий».

Платформа строится вокруг:

- **Admin Shell** — единого frontend-приложения;
- **Administrative Portal BFF** — тонкой серверной точки входа для admin frontend;
- **Service Admin APIs** — административных API конкретных сервисов;
- **schema-rendered pages** — типовых сервисных dashboard-страниц, которые Shell строит из декларативных контрактов;
- **custom service modules** — модулей для сложных сервисных workflows;
- **shared design system** — единого набора Web Components, CSS tokens и общих UX-правил.

Ключевой результат:

> Новый сервис может подключить manifest, административный API и декларативную schema страницы — и появиться в общей админке без копирования sidebar, auth, роутинга, таблиц, тостов, RBAC и базовых dashboard-компонентов.

---

## 2. Базовая формула

```text
Admin Shell = UX composition + design system + schema rendering
Portal BFF = administrative entry point + API composition + context forwarding
Service Admin APIs = domain truth + commands + metrics + audit writing
Schemas = стандартные service pages
Custom modules = сложные доменные workflows
Shared contracts = compatibility boundary
Backend policies = security enforcement
```

Или короче:

```text
Shell owns experience.
BFF owns composition.
Services own business truth.
Schemas render standard pages.
Modules implement complex workflows.
Contracts own compatibility.
Backend enforces security.
```

---

## 3. Общая схема

```text
┌────────────────────────────────────────────────────────────────┐
│                   Admin Shell / Frontend                        │
│                                                                │
│ • Primary sidebar: общие dashboards + сервисы                 │
│ • Context sidebar: страницы выбранного сервиса                │
│ • Routing, deep links, focus mode                             │
│ • JWT session, UI-level RBAC visibility                       │
│ • Design system / Web Components                              │
│ • Schema renderer                                             │
│ • Generic CRUD renderer                                       │
│ • Custom module host                                          │
│ • Toasts, modals, loading/error/empty states                  │
└────────────────────────────┬───────────────────────────────────┘
                             │
                             │ Internal Admin API
                             ▼
┌────────────────────────────────────────────────────────────────┐
│                   Administrative Portal BFF                     │
│                                                                │
│ • Единая administrative entry point                            │
│ • Service registry / manifests / navigation metadata           │
│ • Effective permissions                                       │
│ • Routing к service admin APIs                                 │
│ • JWT / user / environment / correlation context forwarding    │
│ • System dashboards aggregation                               │
│ • System audit read model                                     │
│ • Thin proxy / adapters                                       │
│ • Stateless by default                                        │
└────────────────────────────┬───────────────────────────────────┘
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

---

## 4. Границы ответственности

| Область                  |   Admin Shell   |                  Portal BFF                   |      Service Admin API       |       Service Module        |
| ------------------------ | :-------------: | :-------------------------------------------: | :--------------------------: | :-------------------------: |
| Общий layout             |       Да        |                      Нет                      |             Нет              |             Нет             |
| Primary/context sidebar  |       Да        |                 Даёт metadata                 |             Нет              |             Нет             |
| Browser routing          |       Да        |     Валидирует metadata при необходимости     |             Нет              |  Использует route context   |
| JWT session              |       Да        |             Может выдавать `/me`              |        Валидирует JWT        |  Использует Shell context   |
| UI visibility по RBAC    |       Да        |       Агрегирует effective permissions        |             Нет              |  Декларирует requirements   |
| Авторизация команд       |       Нет       | Дополнительно на BFF routes при необходимости |       Да, обязательно        |             Нет             |
| Общесистемные dashboards |    Рендерит     |                  Агрегирует                   |       Публикует данные       |             Нет             |
| Системный audit viewer   |    Рендерит     |             Агрегирует read model             |     Пишет service audit      | Может открыть audit context |
| Generic CRUD             |    Рендерит     |             Проксирует/адаптирует             | Отдаёт resource contract/API |        Не обязателен        |
| Service dashboard        | Рендерит schema |             Проксирует page model             |     Отдаёт schema + data     |  Рендерит сложную страницу  |
| Dangerous workflow       |    Хостит UI    |          Может быть policy boundary           |     Выполняет и аудирует     |    Реализует UX workflow    |
| Telemetry context        |   RUM/errors    |        Correlation/context forwarding         |     Metrics/traces/logs      |    Page/module telemetry    |

---

## 5. Admin Shell

### 5.1. Назначение

Admin Shell — единое frontend-приложение и единый административный UX.

Он не содержит бизнес-логику Cache Service, AI Router, Prompting или Ingestion. Он предоставляет платформенные механизмы, в которых эти сервисы отображаются и управляются.

### 5.2. Ответственности Shell

| Функция         | Требование                                                    |
| --------------- | ------------------------------------------------------------- |
| Layout          | Единый header, primary sidebar, context sidebar, main content |
| Navigation      | Общие dashboards, список сервисов, service pages              |
| Routing         | Deep links, back/forward, query params для state таблиц       |
| Authentication  | Использование JWT + refresh flow по общему ТЗ                 |
| RBAC UI         | Скрытие/отключение недоступных страниц и actions              |
| Design system   | Общие компоненты, tokens, themes, accessibility               |
| Schema renderer | Рендеринг dashboard/resource pages из декларативной schema    |
| Generic CRUD    | Таблицы, формы, pagination, filters, bulk actions, export     |
| Module host     | Lazy loading и lifecycle custom service modules               |
| UX states       | Loading, empty, error, stale, retry, permission denied        |
| Notifications   | Интеграция с существующими toast                              |
| Modals          | Создание, редактирование, подтверждение опасных операций      |
| Polling         | Управление lifecycle polling, default 10 секунд               |
| Telemetry       | Ошибки UI, page events, correlation ID support                |

---

## 6. Portal BFF

### 6.1. Назначение

Portal BFF — лёгкий внутренний backend для административного frontend. Он является единой точкой входа для Admin Shell, но не заменяет API Gateway и не становится центральным бизнес-сервисом.

### 6.2. Что делает

| Функция             | Требование                                                      |
| ------------------- | --------------------------------------------------------------- |
| Service registry    | Список сервисов, navigation metadata, status, manifest version  |
| `/me`               | Текущий пользователь, роли, effective permissions               |
| System dashboards   | Агрегация данных для кросс-сервисных страниц                    |
| System audit        | Агрегированное read model представление audit                   |
| Routing             | Разрешение service slug → service admin endpoint                |
| Context forwarding  | User, JWT, correlation ID, environment, locale                  |
| Proxy/adapter       | Тонкий прокси к service admin APIs                              |
| Error normalization | Приведение ошибок сервисов к общему admin error contract        |
| Observability       | Structured logs, traces, correlation                            |
| Stateless behavior  | Без хранения бизнес-данных и без собственных business workflows |

### 6.3. Что BFF не делает

| Не делает                                         | Причина                                               |
| ------------------------------------------------- | ----------------------------------------------------- |
| Не хранит доменные данные сервисов                | Источник истины — service domain                      |
| Не реализует cache/AI/prompting business logic    | Это ответственность сервисов                          |
| Не заменяет audit writer сервисов                 | Сервис пишет свой audit при выполнении команды        |
| Не авторизует сервис вместо него                  | Service Admin API всегда выполняет финальную проверку |
| Не выполняет тяжёлую трансформацию каждого ответа | Чтобы не стать bottleneck                             |
| Не хранит schema registry в MVP                   | Сервис отдаёт schema + data одним page endpoint       |
| Не реализует сложную оркестрацию                  | Пока нет подтверждённой потребности                   |
| Не является общим API Gateway                     | У него административная, а не ingress-роль            |

### 6.4. Прототипная модель маршрута

Для упрощения topology и исключения CORS-зоопарка:

```text
Admin Shell → Portal BFF → Service Admin APIs
```

Portal BFF скрывает внутренние адреса сервисов от браузера и обеспечивает единую точку доступа, но остаётся тонким.

---

## 7. Навигация

### 7.1. Двухуровневая модель

```text
┌──────────────────────┬──────────────────────┬────────────────────────────┐
│ Primary sidebar      │ Context sidebar      │ Main content               │
│                      │                      │                            │
│ Общие dashboards     │ Cache Service        │ Cache Overview             │
│ Сервисы              │ ├── Overview         │                            │
│ Инструменты          │ ├── Keys             │ KPI, charts, tables        │
│                      │ ├── TTL              │                            │
│                      │ ├── Maintenance      │                            │
│                      │ ├── Activity         │                            │
│                      │ └── Audit            │                            │
└──────────────────────┴──────────────────────┴────────────────────────────┘
```

### 7.2. Primary sidebar

Содержит:

```text
Общие dashboards
├── Системный обзор
├── Пользователи
├── Использование AI
├── Общие алерты
├── Проблемы
├── Производительность
├── Аптайм
└── Общий Audit

Сервисы
├── AI Router
├── Prompting
├── Ingestion
├── Cache Service
└── Другие сервисы

Инструменты
├── Документация
├── Настройки
├── Профиль
└── Выход
```

### 7.3. Service navigation

| Действие               | Поведение                                                            |
| ---------------------- | -------------------------------------------------------------------- |
| Hover/focus по сервису | Открывает временный flyout preview; main content и URL не меняются   |
| Click по сервису       | Закрепляет service context, открывает context sidebar и default page |
| Click по service page  | Открывает страницу, меняет URL                                       |
| Direct URL             | Восстанавливает service/page/filters/pagination                      |
| Escape                 | Закрывает flyout                                                     |
| Focus mode             | Скрывает context sidebar и расширяет main content                    |

### 7.4. URL

```text
/admin?page=system-overview
/admin?page=users
/admin?page=alerts

/admin?service=cache&page=overview
/admin?service=cache&page=keys&search=user%3A&ttl=lt-1h&pageNumber=1&limit=50
/admin?service=ai-router&page=rate-limits
```

---

## 8. Дизайн-система и Web Components

### 8.1. Принцип

Admin Shell владеет визуальным языком. Сервисы не передают HTML, CSS или JavaScript для рендеринга своих стандартных страниц.

Сервис предоставляет **декларативные данные и metadata**, Shell отображает их своими Web Components.

```text
Service:
  "Вот метрика, unit, threshold, data и разрешённые действия."

Shell:
  "Я отрисую её стандартной карточкой, графиком или таблицей
   в едином стиле, с едиными a11y и state правилами."
```

### 8.2. Минимальный набор компонентов

| Категория     | Web Components                                                                                                                    |
| ------------- | --------------------------------------------------------------------------------------------------------------------------------- |
| Layout        | `<admin-page>`, `<admin-section>`, `<admin-dashboard-grid>`, `<admin-panel>`                                                      |
| Navigation    | `<admin-primary-sidebar>`, `<admin-context-sidebar>`, `<admin-page-header>`                                                       |
| Data          | `<admin-kpi-card>`, `<admin-status-badge>`, `<admin-chart>`, `<admin-data-grid>`, `<admin-activity-feed>`, `<admin-audit-viewer>` |
| Input/actions | `<admin-button>`, `<admin-modal>`, `<admin-confirm-dialog>`, `<admin-filter-toolbar>`, `<admin-inline-editor>`                    |
| States        | `<admin-loading-state>`, `<admin-error-state>`, `<admin-empty-state>`, `<admin-permission-denied>`                                |

Web Components позволяют создавать изолированные custom elements; Shadow DOM изолирует структуру, стили и поведение компонента от окружающего документа. [developer.mozilla](https://developer.mozilla.org/en-US/docs/Web/API/Web_components/Using_custom_elements)

### 8.3. Практическое правило

- Web Components и Shadow DOM использовать для shell, shared primitives и сложных независимых widgets.
- Не применять Shadow DOM механически к каждому маленькому элементу.
- Единый style vocabulary строится через CSS tokens и CSS Cascade Layers.
- Сервисные модули должны использовать shared components, а не создавать конкурирующие карточки, таблицы и диалоги.

---

## 9. Schema-rendered service pages

### 9.1. Принцип

Стандартные service-specific страницы, особенно dashboards, генерируются Admin Shell из versioned declarative page model.

```text
One route → one page endpoint → one declarative page model.
Shell renders.
Service owns data and page metadata.
BFF stays thin.
```

### 9.2. Модель page response для прототипа

В прототипе применяется **вариант A**: service admin API возвращает schema и актуальные data одним endpoint.

```text
GET /admin-api/services/{service}/pages/{page}
```

Логически ответ включает:

| Раздел          | Содержимое                                            |
| --------------- | ----------------------------------------------------- |
| `schemaVersion` | Версия page model                                     |
| `service`       | Service id, slug, name, status, environment           |
| `page`          | Page id, title, refresh policy, required permissions  |
| `capabilities`  | Поддерживаемые возможности                            |
| `layout`        | Sections, grid hints, widget descriptors              |
| `data`          | Актуальные KPI, time series, таблицы, activity        |
| `state`         | `generatedAt`, stale status, warnings, partial errors |
| `links`         | Допустимые drill-down/navigation targets              |
| `correlationId` | Связь с логами и trace                                |

### 9.3. Почему schema + data вместе

| Причина                        | Решение                                                   |
| ------------------------------ | --------------------------------------------------------- |
| Прототип должен быть простым   | Один endpoint вместо нескольких                           |
| BFF должен быть маленьким      | Нет registry/cache schema инфраструктуры                  |
| Отладка должна быть прозрачной | Один page request в DevTools и logs                       |
| Service deployment             | Schema и data меняются согласованно                       |
| Dashboard polling              | Раз в 10 секунд повторяется один запрос                   |
| Изменения потом                | Контракт допускает разделение schema/data без ломки Shell |

### 9.4. Polling

| Правило               | Значение                                                        |
| --------------------- | --------------------------------------------------------------- |
| Default interval      | 10 секунд                                                       |
| Настройка             | `refreshIntervalSeconds` в page model                           |
| При уходе со страницы | Polling останавливается                                         |
| In-flight request     | Отменяется через `AbortController`                              |
| Failure               | Показывается stale/error state, предыдущие данные не затираются |
| Manual refresh        | Доступен на странице                                            |
| Hidden tab            | Интервал может замедляться или polling может останавливаться    |

### 9.5. Когда разделять schema и data

Разделение выполняется только при наличии измеримых триггеров:

- schema занимает существенную долю каждого polling response;
- dashboard используют много одновременных пользователей;
- schema стабильна, а data часто обновляются;
- отдельные widgets имеют разные периоды, filters или источники;
- нужны независимые schema caching и version distribution;
- появляется registry/versioning инфраструктура.

До этих условий page endpoint остаётся единым.

---

## 10. Schema rules

### 10.1. Разрешённые widget types MVP

| Widget type     | Назначение                           |
| --------------- | ------------------------------------ |
| `status`        | Статус сервиса/health                |
| `kpi`           | Числовая метрика                     |
| `timeSeries`    | Временной ряд                        |
| `distribution`  | Histogram/distribution, например TTL |
| `table`         | Read-only агрегированная таблица     |
| `resourceTable` | Generic CRUD table                   |
| `activityFeed`  | Последние события                    |
| `alertList`     | Активные alerts                      |
| `markdown`      | Ограниченный информационный контент  |
| `emptyState`    | Пустое состояние                     |

### 10.2. Security schema rules

Schema не может содержать:

- executable JavaScript;
- inline event handlers;
- произвольный HTML;
- CSS от сервиса;
- произвольные external script URLs;
- arbitrary fetch URLs;
- выражения, которые исполняются как код.

Schema содержит только allowlisted декларативные описания:

- widget type;
- title/description;
- data path;
- unit/format;
- thresholds;
- page links;
- permission/capability requirements;
- standard action descriptors.

### 10.3. Поведение неизвестных типов

| Ситуация               | Поведение                                                                         |
| ---------------------- | --------------------------------------------------------------------------------- |
| Unknown schema version | Safe error state с указанием compatibility problem                                |
| Unknown widget type    | Safe placeholder + telemetry event                                                |
| Missing data path      | Виджет показывает no-data/error state                                             |
| Missing capability     | Виджет/страница не отображается                                                   |
| Missing permission     | UI скрывает элемент; direct request получает 403                                  |
| Invalid schema         | Страница не рендерится частично небезопасно; показывается schema validation error |

---

## 11. Generic CRUD rendering

### 11.1. Generic rendering применяется для

- таблиц типовых сущностей;
- поиска;
- status/date filters;
- серверной пагинации 25/50/100;
- total count;
- inline editing простых полей;
- create/edit modal forms;
- bulk status change;
- soft delete;
- audit history;
- CSV export.

### 11.2. Resource metadata

Service предоставляет metadata:

| Поле             | Назначение                    |
| ---------------- | ----------------------------- |
| `resource`       | Название сущности             |
| `endpoint`       | Base endpoint                 |
| `columns`        | Колонки                       |
| `searchField`    | Одно основное поле поиска     |
| `filters`        | Status/date filters           |
| `editableFields` | Допустимые inline fields      |
| `formFields`     | Поля modal form               |
| `bulkActions`    | Поддерживаемые действия       |
| `softDelete`     | Поддержка soft delete         |
| `audit`          | Поддержка audit history       |
| `export`         | CSV export                    |
| `permissions`    | Read/write/delete permissions |

### 11.3. Граница generic CRUD

Если страница начинает требовать сложный workflow, многошаговую форму, специфичную state machine, domain diagram, complex diff/merge или опасную процедуру, generic CRUD прекращает применяться. Страница становится custom service module.

---

## 12. Custom service modules

### 12.1. Когда нужен модуль

| Сценарий                          | Причина                                     |
| --------------------------------- | ------------------------------------------- |
| Maintenance и flush cache         | Typed confirmation, reason, progress, audit |
| Prompt version diff/merge         | Специальная визуализация и workflow         |
| Ingestion reprocess               | Очереди, стадии, retries, partial failures  |
| AI Router fallback chain          | Доменная визуализация и интерактивность     |
| Сложные зависимости полей         | Нужна доменная state machine                |
| Schema становится сложнее UI-кода | Признак, что нужен модуль                   |

### 12.2. MVP plugin model

```text
Plugin = first-party lazy-loaded service module
       + module manifest
       + mount/unmount lifecycle
       + declared permissions/capabilities
       + stable Shell host API
```

Не требуется на этапе прототипа:

- remote modules;
- Module Federation;
- iframe sandbox;
- third-party code;
- независимые deployments модулей;
- plugin marketplace.

---

## 13. Действия и команды

### 13.1. Standard action descriptor

Schema может содержать стандартные действия:

| Поле                 | Назначение                                       |
| -------------------- | ------------------------------------------------ |
| `id`                 | Идентификатор                                    |
| `label`              | Текст                                            |
| `icon`               | Иконка                                           |
| `kind`               | `navigate`, `refresh`, `apiCommand`, `openModal` |
| `operationRef`       | Ссылка на известную операцию API contract        |
| `requiredPermission` | Permission                                       |
| `confirmationPolicy` | None / standard / typed confirmation             |
| `dangerLevel`        | Info / warning / destructive / critical          |
| `reasonRequired`     | Нужна ли причина                                 |
| `successMessage`     | Текст toast                                      |
| `invalidationKeys`   | Какие page/table data нужно обновить             |

### 13.2. Command pipeline

```text
User action
→ UI permission check
→ Confirmation dialog, если требуется
→ Command request через Portal BFF
→ Service Admin API authorization
→ Business validation
→ Command execution
→ Service audit write
→ Structured result
→ Toast + data refresh + activity update
```

Portal BFF может быть дополнительной policy boundary, но финальная авторизация и audit command принадлежат сервису.

---

## 14. Security

### Базовая модель

```text
Frontend visibility ≠ authorization
BFF routing ≠ domain authorization
Service policy = final enforcement
```

### Security baseline

| Контроль            | Требование                                            |
| ------------------- | ----------------------------------------------------- |
| Auth                | ASP.NET Identity + JWT + refresh flow                 |
| Roles               | `Admin`, `SecurityAdmin`, `ContentManager`            |
| Permissions         | Fine-grained policy checks на backend                 |
| Token               | `Authorization: Bearer {token}`                       |
| Frontend storage    | `localStorage` на этапе прототипа                     |
| Validation          | Frontend — формат/UX; backend — бизнес-валидация      |
| Output              | Нет raw unsafe HTML rendering                         |
| Schema              | Allowlist, validation, no executable code             |
| CSP                 | Поэтапное внедрение strict CSP                        |
| Sensitive data      | Masking/redaction в UI, export, audit, logs           |
| Destructive actions | Permission + confirmation + audit + reason, где нужно |
| Errors              | Не показывать internal traces/secrets пользователю    |

---

## 15. Observability, audit и деградация

### 15.1. Observability

| Уровень | Ответственность                                         |
| ------- | ------------------------------------------------------- |
| Shell   | Frontend errors, page performance, route/page events    |
| BFF     | Correlation context, routing logs, proxy latency/errors |
| Service | Metrics, traces, command logs, health, business errors  |
| Module  | Page-specific telemetry                                 |

### 15.2. Audit

| Требование       | Детали                                                     |
| ---------------- | ---------------------------------------------------------- |
| Пишет            | Service, который выполняет command                         |
| Читает           | Unified audit viewer через Shell/BFF                       |
| Формат           | Diff: field, old, new                                      |
| Context          | Actor, service, environment, scope, result, correlation ID |
| Доступ           | Admin и SecurityAdmin                                      |
| Sensitive values | Redacted                                                   |
| Хранение         | Бессрочно на текущем этапе                                 |
| Export           | Позже                                                      |

### 15.3. Graceful degradation

| Ситуация           | Поведение                                                           |
| ------------------ | ------------------------------------------------------------------- |
| Нет dashboard data | Error/stale state, shell не падает                                  |
| Один widget failed | Падает только widget, не вся page                                   |
| Service offline    | Health/overview/audit доступны; commands отключены                  |
| Нет capability     | Page/widget не показывается                                         |
| Нет permission     | Page/action скрыт; deep link обрабатывает 403                       |
| Polling error      | Сохранить последние данные, показать stale indicator                |
| Browser offline    | Read-only/degraded mode; destructive commands не ставятся в очередь |

---

## 16. Реализационный порядок

| Шаг | Результат                                                                                           |
| --- | --------------------------------------------------------------------------------------------------- |
| 1   | Portal BFF skeleton: health, `/me`, service registry, routing, correlation forwarding               |
| 2   | Admin Shell skeleton: login/session, primary sidebar, context sidebar, routing, toasts, base states |
| 3   | Design system MVP: page/layout, cards, status, chart container, table, modal, confirm dialog        |
| 4   | Shared API client: JWT, refresh, errors, timeout, abort, query params                               |
| 5   | Schema page model v1: schema + data in one endpoint                                                 |
| 6   | Schema renderer MVP: status, KPI, timeSeries, distribution, table, activity feed                    |
| 7   | Generic CRUD renderer: tables, filters, pagination, modal forms, audit, CSV                         |
| 8   | Cache Service Overview как первый schema-rendered page                                              |
| 9   | Cache Maintenance как первый custom module                                                          |
| 10  | Contract tests, RBAC tests, critical E2E, a11y checks                                               |
| 11  | Следующие schema/service modules: AI Router, Prompting, Ingestion                                   |

---

## 17. Критерии готовности прототипа

Прототип считается архитектурно подтверждённым, когда:

- существует единый Admin Shell;
- Shell работает через Portal BFF как единую internal entry point;
- sidebar содержит global pages и services;
- сервис открывается с pinned context sidebar;
- service page может быть рендерена из declarative page model;
- page model возвращается одним endpoint вместе со schema и data;
- polling работает с default interval 10 секунд;
- Shell использует единые компоненты дизайн-системы;
- generic table поддерживает search, filters, pagination и CSV export;
- RBAC отображается в UI и финально проверяется service backend;
- Cache Service Overview построен schema renderer-ом;
- Cache Maintenance реализован как custom module;
- destructive operation проходит confirmation, permission check и audit;
- прямые URL восстанавливают service/page context;
- недоступность одного сервиса или widget не ломает весь Shell.

---

## 18. Итоговая формула прототипа

```text
One Admin Shell.
One administrative entry point.
One shared design system.
One contract model.
One page endpoint per schema-rendered page.
One thin BFF.
Many services.
Generic UI where repeated.
Custom modules where complex.
```

Это и фиксируем как **документ №4: «Архитектурные принципы, Admin Shell и Portal BFF»**.
