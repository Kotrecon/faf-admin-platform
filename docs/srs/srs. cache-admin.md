# ТЗ: Cache Service Control Panel

**Версия:** 0.1  
**Статус:** Draft / Prototype specification  
**Контекст:** внутренняя универсальная админ-панель системы «помощник режиссёра мероприятий»  
**Цель:** специализированная панель наблюдения и безопасного управления cache layer конкретного сервиса.

---

## 1. Назначение

Cache Service Control Panel — это сервис-специфичная административная страница для наблюдения за состоянием и эффективностью кэширования, исследования ключей, выполнения контролируемых операций записи и обслуживания cache storage.

Панель **не заменяет** общий дашборд сервиса. Она является специализированным control plane для cache layer и встраивается в общий frontend-shell:

- общая аутентификация и RBAC;
- единый sidebar и top tabs;
- общий стиль уведомлений;
- общая инфраструктура audit log;
- общая политика polling;
- единый механизм обработки API-ошибок.

Панель предназначена для внутренних пользователей: `Admin` и `SecurityAdmin`. `ContentManager` по умолчанию доступа к ней не получает.

---

## 2. Цели и границы

### Цели

- Быстро оценивать здоровье cache layer.
- Контролировать эффективность кэша: hits, misses, hit ratio, latency, errors, evictions.
- Видеть давление на память и риск деградации.
- Исследовать ключи по namespace/prefix без опасных операций над всем keyspace.
- Ручно создавать, обновлять и удалять записи при диагностике.
- Выполнять maintenance-операции с подтверждением, RBAC и аудитом.
- Просматривать последние технические и административные события.
- Предоставлять экспорт данных в CSV в рамках общего правила админки.

### Не входит в MVP

- Полноценная замена Grafana, Prometheus или централизованной observability-платформы.
- Автоматический remediation без явного действия оператора.
- Массовый импорт ключей.
- Редактирование Redis-конфигурации из UI.
- Управление topology/cluster/failover Redis.
- Полноценный редактор сериализованных payload без ограничений.
- Поиск по содержимому values.

---

## 3. Контекст и место в навигации

Панель открывается внутри универсальной административной оболочки.

```text
Universal Admin
│
├── System Dashboard
├── System Users
├── Security / Audit
│
└── Services
    ├── AI Router
    ├── Prompting
    ├── Ingestion
    ├── Cache Service
    │   ├── Dashboard
    │   ├── Cache Control
    │   ├── Metrics
    │   ├── Audit
    │   └── Settings
    └── ...
```

### Предлагаемый URL

```text
/admin?service=cache&page=control
/admin?service=cache&page=keys
/admin?service=cache&page=maintenance
/admin?service=cache&page=audit
```

Допускается начать с одной страницы `Cache Control`, а затем вынести перегруженные области в отдельные tabs:

| Tab           | Назначение                                                    |
| ------------- | ------------------------------------------------------------- |
| `Overview`    | Состояние, KPI, графики, последние события                    |
| `Keys`        | Key explorer: поиск, фильтры, пагинация, операции над ключами |
| `Maintenance` | Очистка expired, flush namespace или cache, история задач     |
| `Audit`       | Неизменяемая история административных действий                |

---

## 4. Макет панели

Начальный визуальный прототип уже задаёт удачную композицию:

```text
┌─────────────────────────────────────────────────────────────────────┐
│ SYSTEM / CACHE CONTROL                 [ENV: PRODUCTION] [ONLINE]   │
│ Cache Layer Admin              [15m] [1h] [24h] [7d] Updated: 6s   │
├─────────────────────────────────────────────────────────────────────┤
│ Hit ratio │ Hits/Misses │ Active keys │ Memory │ Errors │ Evictions │
├───────────────────────────────────────┬─────────────────────────────┤
│ Cache efficiency / TTL distribution   │ Service health / alerts      │
│ Graphs                                │ Critical / Warning feed      │
├───────────────────────────────────────┴─────────────────────────────┤
│ Search by prefix / namespace / TTL / state             [Export CSV] │
│ Stored keys table with server pagination                              │
├──────────────────────────────┬──────────────────────────────────────┤
│ Write key / diagnostic tools │ Latest activity                      │
│ Maintenance                  │ Recent technical/admin events        │
└──────────────────────────────┴──────────────────────────────────────┘
```

### Принцип приоритета

1. Состояние и риск — наверху.
2. Анализ и поиск ключей — центральная рабочая область.
3. Ручная запись — вторичная диагностическая операция.
4. Maintenance — явно изолированный блок.
5. История событий — рядом, но не заменяет audit trail.

---

## 5. Верхняя панель

### Обязательные элементы

| Элемент             | Требование                                                                         |
| ------------------- | ---------------------------------------------------------------------------------- |
| Название            | `Cache Layer Admin` или согласованное имя cache service                            |
| Service status      | `Online`, `Degraded`, `Offline`, `Unknown`                                         |
| Environment badge   | Явно: `LOCAL`, `DEV`, `STAGING`, `PRODUCTION`                                      |
| Period selector     | `15m`, `1h`, `24h`, `7d`; default — `15m` или `1h`                                 |
| Freshness indicator | Время последнего успешного обновления, например `Updated 6s ago`                   |
| Manual refresh      | Кнопка ручного обновления                                                          |
| Loading state       | Ненавязчивый индикатор фонового polling                                            |
| Error state         | Сообщение о частичной деградации данных, если один из источников метрик недоступен |

### Правило для production

Environment должен быть заметен текстом, а не только цветом. Нельзя допускать ситуацию, в которой оператор может перепутать `Staging` и `Production` при выполнении `Flush all`.

---

## 6. Health и KPI

### 6.1. Статус cache service

| Статус     | Критерий                                                                                         |
| ---------- | ------------------------------------------------------------------------------------------------ |
| `Online`   | Health check успешен, latency и error rate в допустимых пределах                                 |
| `Degraded` | Сервис доступен, но есть высокий latency, ошибки, evictions, memory pressure или иная деградация |
| `Offline`  | Cache backend недоступен                                                                         |
| `Unknown`  | Данные health check отсутствуют или устарели                                                     |

При статусе `Degraded` UI обязан показать краткую причину, например:

> Cache backend доступен, но memory usage 94% и обнаружено 137 evictions за последние 15 минут.

### 6.2. Верхний набор KPI

| Метрика                   | Приоритет | Описание                                             |
| ------------------------- | --------: | ---------------------------------------------------- |
| **Cache hit ratio**       |        P0 | Доля успешных cache lookup среди hits + misses       |
| **Hits / Misses**         |        P0 | Число попаданий и промахов за выбранный период       |
| **Active keys**           |        P1 | Количество активных ключей                           |
| **Memory usage**          |        P0 | Использованная память, лимит и процент использования |
| **Evictions**             |        P0 | Количество вытесненных ключей за выбранный период    |
| **Error rate**            |        P0 | Ошибки cache read/write/connection                   |
| **Latency p95**           |        P1 | p95 latency операций cache layer                     |
| **Expired keys**          |        P2 | Число истёкших ключей за период                      |
| **Average remaining TTL** |        P2 | Средний оставшийся TTL активных ключей               |

Для Redis hit ratio определяется как:

\[
\text{Hit Ratio} =
\frac{\text{Keyspace Hits}}
{\text{Keyspace Hits} + \text{Keyspace Misses}}
\times 100\%
\]

Redis предоставляет `keyspace_hits`, `keyspace_misses`, `evicted_keys`, `expired_keys`, данные по memory и keyspace через `INFO`; Redis также прямо рекомендует использовать hits и misses для расчёта cache hit ratio. [redis](https://redis.io/docs/latest/develop/reference/eviction/)

### 6.3. Цветовые состояния KPI

| Состояние     | UI-значение                                                         |
| ------------- | ------------------------------------------------------------------- |
| Normal        | Нейтральный или зелёный                                             |
| Warning       | Жёлтый/янтарный: approaching limit или деградация                   |
| Critical      | Красный: высокий error rate, memory exhaustion, service unavailable |
| Unknown       | Серый: нет данных                                                   |
| Informational | Голубой/бирюзовый: без оценки риска                                 |

Цвет не должен быть единственным носителем смысла: рядом нужен текст, иконка или label.

---

## 7. Графики

### Обязательные графики MVP

| График              | Ось X       | Ось Y                  | Цель                                                 |
| ------------------- | ----------- | ---------------------- | ---------------------------------------------------- |
| Hit / miss rate     | Время       | requests/sec или count | Оценка эффективности кэша                            |
| Hit ratio           | Время       | %                      | Быстрый анализ тренда                                |
| Memory usage        | Время       | MB/GB и % лимита       | Выявление давления на память                         |
| Evictions / expired | Время       | count                  | Выявление неконтролируемого eviction и TTL-поведения |
| Cache latency       | Время       | ms                     | Выявление деградации                                 |
| TTL distribution    | TTL buckets | Количество ключей      | Анализ профиля времени жизни ключей                  |

### TTL distribution

Рекомендуемые интервалы:

```text
Expired / < 1m / 1–5m / 5–15m / 15–60m / 1–6h / 6–24h / > 24h / persistent
```

График должен показывать:

- число ключей в bucket;
- выбранный namespace, если включён фильтр;
- timestamp снимка;
- состояние «нет данных»;
- предупреждение, если распределение строится на выборке, а не на полном сканировании.

### Источник временных рядов

Графики не должны строиться из разового обхода keyspace. Источник — observability/metrics layer, совместимый с вашей общей telemetry-архитектурой.

Для наименований, attributes и корреляции telemetry желательно опираться на OpenTelemetry Semantic Conventions, которые стандартизируют смысл атрибутов, metric instruments, units и имён для traces, metrics и logs. [opentelemetry](https://opentelemetry.io/docs/concepts/semantic-conventions/)

---

## 8. Key Explorer

### Назначение

Key Explorer — центральная рабочая область для просмотра и поиска ключей. Он не должен быть построен на полном вызове Redis `KEYS *` для регулярного UI-запроса.

Redis отмечает, что `KEYS` имеет сложность \(O(N)\), способен ухудшить производительность на больших базах и не должен использоваться в обычном production application code; для выборки подмножества keyspace рекомендуется `SCAN` либо специализированные структуры. [redis](https://redis.io/docs/latest/commands/keys/)

### Поиск и фильтры

| Функция           | Требование                                                                 |
| ----------------- | -------------------------------------------------------------------------- |
| Search            | Поиск по prefix / pattern имени ключа                                      |
| Минимальная длина | 2–3 символа, кроме явно выбранного namespace                               |
| Case sensitivity  | Зависит от соглашения именования ключей; default — literal/prefix matching |
| Namespace filter  | Поддержка namespace/prefix, например `user:`, `prompt:`, `ingestion:`      |
| TTL filter        | `expired`, `<1m`, `<1h`, `<24h`, `persistent`, `all`                       |
| Type filter       | Если backend позволяет: string/hash/list/set и т. п.                       |
| Size filter       | Опционально, отложить после MVP                                            |
| Sort              | Key, TTL, size, created/updated — если данные доступны                     |
| URL state         | Все фильтры синхронизируются с query params                                |
| Auto apply        | Применение автоматически; debounce для text search                         |

### Колонки таблицы

| Колонка       |                 MVP | Комментарий                                       |
| ------------- | ------------------: | ------------------------------------------------- |
| Key           |                  Да | Полное имя или безопасно сокращённое с copy       |
| Namespace     |                  Да | Выделяемый prefix                                 |
| TTL remaining |                  Да | `persistent`, `expired`, значение в readable form |
| Size          |   Да, если доступно | Размер payload                                    |
| Type          |      По возможности | Тип Redis value или абстрактный тип записи        |
| Created at    |  Если есть metadata | Не всегда доступно в native cache storage         |
| Last accessed | Если есть telemetry | Не пытаться эмулировать без фактических данных    |
| Hits          |      По возможности | Может требовать отдельного instrumentation        |
| Actions       |                  Да | Details, delete, возможно copy key                |

### Пагинация

| Требование                   | Значение                                                           |
| ---------------------------- | ------------------------------------------------------------------ |
| Тип                          | Серверная                                                          |
| Размер страницы              | 25 / 50 / 100                                                      |
| Total count                  | Показывать, только если стоимость вычисления приемлема             |
| При отсутствии точного total | Допустим режим cursor pagination с `estimated total` или без total |
| URL                          | `?page=1&limit=25&prefix=user%3A`                                  |
| Сортировка                   | Только по безопасным и поддерживаемым полям                        |

Для Redis/SCAN следует учитывать, что «точный total count» по произвольному pattern может быть дорогим. В таком случае API должен явно возвращать `total: null` либо `totalIsEstimated: true`, а UI не должен выдавать приблизительное число за точное.

### Просмотр значений

Просмотр value не должен быть включён по умолчанию.

| Требование      | Детали                                                            |
| --------------- | ----------------------------------------------------------------- |
| Доступ          | Отдельное permission `cache:view-values`                          |
| Редакция данных | Маскирование секретов, токенов, PII, credentials                  |
| Размер          | Ограничение preview, например первые N KB                         |
| Формат          | Автоопределение JSON/text/binary + raw download при необходимости |
| Логирование     | Каждое раскрытие sensitive value может попадать в audit           |
| Поиск по value  | Не входит в MVP                                                   |

---

## 9. Ручная запись и обновление ключа

### Назначение

Функция предназначена для диагностики, тестирования и точечных оперативных вмешательств. Она не является главным пользовательским сценарием на dashboard.

### UX

В MVP допустима левая форма, как на макете. После MVP предпочтительно перенести её в:

- `Write key` modal;
- side drawer;
- отдельную вкладку `Tools`.

### Поля

| Поле           | Требование                                                  |
| -------------- | ----------------------------------------------------------- |
| Key            | Обязательное, валидируется по naming convention             |
| Namespace      | Может формироваться отдельно или быть частью ключа          |
| Value          | Текст/JSON, с ограничением размера                          |
| Content type   | `text`, `json`, `serialized` — если применимо               |
| TTL            | Обязателен или явный режим `persistent`, в секундах         |
| Reason         | Обязателен для production manual write или хотя бы доступен |
| Overwrite mode | Явное подтверждение, если ключ уже существует               |

### Проверки

| Проверка                                 | Где                  |
| ---------------------------------------- | -------------------- |
| Required, допустимые символы, формат TTL | Frontend             |
| Максимальный размер value                | Backend              |
| Допустимый namespace                     | Backend              |
| Разрешённый TTL range                    | Backend              |
| Разрешение overwrite                     | Backend              |
| Permission                               | Backend              |
| Audit entry                              | Backend, обязательно |

### Безопасность

- В production нельзя молча перезаписывать существующий ключ.
- При overwrite UI показывает предыдущие metadata: TTL, размер, тип, last update.
- Для опасных namespaces может требоваться `SecurityAdmin`.
- Возможность записывать raw serialized payload должна быть ограничена.

---

## 10. Maintenance

### Операции MVP

| Операция          | Назначение                                                     |        Риск |
| ----------------- | -------------------------------------------------------------- | ----------: |
| `Clear expired`   | Очистка истёкших записей, если backend требует явного удаления |     Средний |
| `Delete key`      | Удаление одного выбранного ключа                               |     Средний |
| `Flush namespace` | Очистка ключей в конкретном namespace                          |     Высокий |
| `Flush all`       | Полная очистка выбранного cache scope                          | Критический |

### `Clear expired`

- Требует подтверждения, если операция затрагивает много ключей.
- Показывает scope и предполагаемое количество, если оно известно.
- Создаёт audit event.
- Показывает job/result в toast и activity feed.

### `Flush all`

`Flush all` — критическая деструктивная операция.

| Требование         | Детали                                                                                |
| ------------------ | ------------------------------------------------------------------------------------- |
| Permission         | Отдельное `cache:flush`                                                               |
| Роль               | По умолчанию только `SecurityAdmin`; `Admin` — только при явном назначении permission |
| Confirmation       | Обязательное модальное окно                                                           |
| Typed confirmation | Ввод строки вида `FLUSH <environment> <scope>`                                        |
| Scope              | Явно показать service, backend, environment, namespace/all keys                       |
| Reason             | Обязательное текстовое поле причины                                                   |
| Dry-run/count      | Показать ориентировочное число затрагиваемых ключей, если доступно                    |
| Audit              | Обязательная неизменяемая запись                                                      |
| Result             | Успех, ошибка, duration, affected count                                               |
| Cooldown           | Защита от повторного исполнения по двойному клику                                     |

`Flush all` никогда не должен выполняться только из-за простого подтверждения кнопкой `OK`.

---

## 11. Live Activity и Audit

### Live Activity

Это короткая оперативная лента последних технических и административных событий.

| Тип события                     |                   Показывать |
| ------------------------------- | ---------------------------: |
| Key created/updated вручную     |                           Да |
| Key deleted вручную             |                           Да |
| Clear expired                   |                           Да |
| Flush namespace/all             |                           Да |
| Service connected/disconnected  |                           Да |
| Cache backend error             |                           Да |
| Health state changed            |                           Да |
| TTL expiration отдельных ключей | Опционально, иначе будет шум |
| Обычные cache hits              |                          Нет |

Для события отображаются:

- timestamp с timezone;
- severity;
- тип;
- краткое описание;
- actor, если действие выполнено человеком;
- correlation/request ID;
- ссылка на key details, audit entry или технические логи.

### Audit Log

Audit log — отдельное долговременное хранилище административных действий.

| Требование        | Детали                                                                          |
| ----------------- | ------------------------------------------------------------------------------- |
| События           | Create/update/delete key, clear expired, flush, просмотр защищённого value      |
| Формат изменений  | Diff: field, old value, new value; secret values redacted                       |
| Кто читает        | `Admin`, `SecurityAdmin`                                                        |
| Хранение          | Бессрочно на текущем этапе                                                      |
| Экспорт           | Позже, CSV                                                                      |
| Неизменяемость    | Записи не редактируются через UI                                                |
| Обязательные поля | Actor, time, action, target, scope, environment, result, reason, correlation ID |

---

## 12. Роли и permissions

### Роли

| Роль             | Назначение                                               |
| ---------------- | -------------------------------------------------------- |
| `Admin`          | Операционное управление, в пределах выданных permissions |
| `SecurityAdmin`  | Полный доступ к security-sensitive операциям и audit     |
| `ContentManager` | Нет доступа по умолчанию                                 |

### Permissions

| Permission           | SecurityAdmin |        Admin         | ContentManager |
| -------------------- | :-----------: | :------------------: | :------------: |
| `cache:read`         |      Да       |          Да          |      Нет       |
| `cache:read-metrics` |      Да       |          Да          |      Нет       |
| `cache:read-keys`    |      Да       |          Да          |      Нет       |
| `cache:view-values`  |      Да       | По явному назначению |      Нет       |
| `cache:write`        |      Да       |          Да          |      Нет       |
| `cache:delete`       |      Да       |          Да          |      Нет       |
| `cache:maintenance`  |      Да       |          Да          |      Нет       |
| `cache:flush`        |      Да       | По явному назначению |      Нет       |
| `cache:view-audit`   |      Да       |          Да          |      Нет       |

Проверка permissions выполняется **на backend**, а не только скрытием кнопок на frontend.

---

## 13. Источники данных и архитектура

### Логические компоненты

```text
┌─────────────────────────────────────────────────┐
│               Admin Frontend Shell               │
│  Cache Overview / Keys / Maintenance / Audit     │
└───────────────────────┬─────────────────────────┘
                        │ HTTPS + JWT
                        ▼
┌─────────────────────────────────────────────────┐
│               Cache Admin API                    │
│                                                 │
│ Read API              Command API                │
│ - health              - write/update key         │
│ - metrics             - delete key               │
│ - charts              - clear expired            │
│ - keys index          - flush namespace/all      │
│ - activity            - audit command            │
└───────┬───────────────────┬─────────────────────┘
        │                   │
        ▼                   ▼
┌─────────────────┐  ┌───────────────────────────┐
│ Cache abstraction│  │ Observability / telemetry  │
│ layer            │  │ metrics, logs, traces      │
└────────┬────────┘  └──────────────┬────────────┘
         │                           │
         ▼                           ▼
┌─────────────────┐       ┌────────────────────────┐
│ Redis / other   │       │ Metrics store / audit DB│
│ cache backend   │       │ / event source          │
└─────────────────┘       └────────────────────────┘
```

### Правило абстракции

Admin API не должен напрямую смешивать UI-логику с конкретным Redis client API. Он работает через общую cache abstraction layer, которая:

- скрывает конкретный backend;
- централизует serialization;
- контролирует naming conventions;
- применяет TTL policies;
- публикует telemetry и audit events;
- позволяет позже поддержать несколько cache backends.

### Read path и command path

| Путь         | Характер                      | Примеры                                      |
| ------------ | ----------------------------- | -------------------------------------------- |
| Read path    | Частый, дешёвый, read-only    | health, metrics, charts, key index, activity |
| Command path | Редкий, защищённый, auditable | write, update, delete, clear expired, flush  |

Команды должны проходить отдельный pipeline:

```text
Authenticate
→ Authorize
→ Validate request
→ Confirm destructive intent where required
→ Execute command
→ Emit telemetry
→ Persist audit event
→ Return structured result
```

---

## 14. API-контракты уровня требований

Это не окончательные endpoint’ы, а контрактные группы.

### Read API

| Endpoint                                  | Назначение                                     |
| ----------------------------------------- | ---------------------------------------------- |
| `GET /api/admin/cache/overview`           | Агрегированные KPI, health, период, freshness  |
| `GET /api/admin/cache/metrics/timeseries` | Временные ряды для графиков                    |
| `GET /api/admin/cache/ttl-distribution`   | TTL histogram                                  |
| `GET /api/admin/cache/keys`               | Key explorer с поиском, фильтрами и пагинацией |
| `GET /api/admin/cache/keys/{keyId}`       | Metadata ключа                                 |
| `GET /api/admin/cache/keys/{keyId}/value` | Защищённый просмотр value                      |
| `GET /api/admin/cache/activity`           | Последние события                              |
| `GET /api/admin/cache/audit`              | Audit log с фильтрами                          |

### Command API

| Endpoint                                            | Назначение                    |
| --------------------------------------------------- | ----------------------------- |
| `POST /api/admin/cache/keys`                        | Создать запись                |
| `PATCH /api/admin/cache/keys/{keyId}`               | Обновить value/TTL            |
| `DELETE /api/admin/cache/keys/{keyId}`              | Удалить ключ                  |
| `POST /api/admin/cache/maintenance/clear-expired`   | Запустить очистку expired     |
| `POST /api/admin/cache/maintenance/flush-namespace` | Очистить namespace            |
| `POST /api/admin/cache/maintenance/flush-all`       | Полный flush с подтверждением |

### Общие требования API

- JWT передаётся в `Authorization: Bearer {token}`.
- Ошибки бизнес-валидации имеют единый формат.
- Все list endpoint’ы используют query params.
- Search и filters сериализуются в URL.
- List responses возвращают `items`, `page`, `limit`, а `total` — если он точен и его вычисление не несёт опасной стоимости.
- Каждая command operation возвращает structured result: `status`, `message`, `affectedCount`, `operationId`, `correlationId`.
- Для длительных операций возможен async job model со статусом выполнения.

---

## 15. Polling и live updates

### MVP

| Объект           |                                                                Интервал |
| ---------------- | ----------------------------------------------------------------------: |
| Health/KPI       |                                                 10 секунд, configurable |
| Latest activity  |                                                 10 секунд, configurable |
| Key table        | Только по пользовательскому действию, смене фильтра или ручному refresh |
| TTL distribution |                                    30–60 секунд либо по ручному refresh |
| Audit            |                                         Только по действию пользователя |

### После MVP

- SSE или WebSocket для live activity и смены health state.
- Polling остаётся fallback-механизмом.
- При скрытой вкладке браузера polling замедляется или приостанавливается.
- Ошибка polling не должна обнулять ранее корректно отображённые значения; UI показывает stale state и время последнего успеха.

---

## 16. Производительность

| Требование            | Решение                                                           |
| --------------------- | ----------------------------------------------------------------- |
| KPI                   | Агрегированный endpoint, кэшируемый на короткий период            |
| Временные ряды        | Из observability store, а не из ключей cache backend              |
| Список ключей         | Cursor/scan-based retrieval с лимитами                            |
| Полный обход keyspace | Запрещён в интерактивном запросе                                  |
| `KEYS *`              | Не использовать в production UI                                   |
| Pagination            | 25 / 50 / 100                                                     |
| Поиск                 | Prefix/pattern search с минимальной длиной запроса                |
| Export                | Асинхронный job при большом объёме                                |
| CSV export            | Только доступные пользователю поля; values исключены по умолчанию |
| Rate limit            | Отдельный rate limit для admin read/command endpoints             |

Redis отдельно предупреждает, что `KEYS` может ухудшить производительность на больших production-базах; для штатной выборки ключей следует использовать `SCAN` или другой безопасный механизм. [redis](https://redis.io/docs/latest/commands/keys/)

---

## 17. Нефункциональные требования

### UX

- Empty states обязательны для таблиц, графиков и activity feed.
- Все опасные операции требуют подтверждения.
- Все действия возвращают существующие toast-notifications.
- Состояния loading, error, stale data и empty data должны визуально различаться.
- Фильтры применяются автоматически.
- Search использует debounce.
- Интерфейс должен быть usable на ширине от 1280 px; мобильная оптимизация не приоритетна для MVP.

### Безопасность

- Значения ключей считаются потенциально чувствительными.
- Любое command action требует server-side authorization.
- Audit обязателен для write/delete/maintenance.
- Production environment явно маркируется.
- Tokens, passwords, credentials и PII редактируются/маскируются при выводе.
- Payload не логируется целиком в технических логах без явной политики redaction.

### Observability

- Все command actions получают correlation ID.
- Ошибки admin API логируются структурированно.
- Health, errors, latency и команды трассируются через общую observability layer.
- Используются унифицированные семантические имена и attributes telemetry там, где они определены OpenTelemetry. [opentelemetry](https://opentelemetry.io/docs/concepts/semantic-conventions/)

---

## 18. Критерии готовности MVP

MVP панели считается готовым, если:

- Пользователь с `cache:read` видит status, environment и freshness.
- Отображаются: hit ratio, hits/misses, active keys, memory usage, errors, evictions.
- Доступны минимум графики hit/miss, hit ratio, memory, TTL distribution.
- Таблица ключей поддерживает search по prefix, TTL filter и server-side pagination.
- `KEYS *` не используется для интерактивного UI.
- Значение ключа защищено отдельным permission и маскированием.
- Авторизованный пользователь может создать, обновить и удалить ключ в пределах своих прав.
- `Clear expired` и `Flush all` имеют разные уровни защиты.
- `Flush all` требует typed confirmation, reason и создаёт audit entry.
- Live activity показывает последние значимые технические и ручные события.
- Audit log хранит diff и административные действия.
- Метрики и health обновляются polling’ом раз в 10 секунд; интервал конфигурируем.
- Ошибки и успешные действия отображаются через существующие toast notifications.
- Доступ к панели, UI-кнопки и backend operations покрыты RBAC/permissions.

---

## 19. Роадмап

| Блок         | MVP                                   | Затем                        | На будущее                                |
| ------------ | ------------------------------------- | ---------------------------- | ----------------------------------------- |
| Dashboard    | Health, KPI, основные графики         | Custom period, drill-down    | Аномалии и SLO                            |
| Key Explorer | Prefix search, TTL filter, pagination | Size/type filters, sorting   | Hot keys, big keys, semantic key analysis |
| Values       | Metadata-first, controlled preview    | Safe structured JSON view    | Version history и compare                 |
| Write tools  | Create/update/delete с audit          | Namespace templates          | Controlled bulk operations                |
| Maintenance  | Clear expired, protected flush all    | Flush namespace, job history | Approval workflow                         |
| Activity     | Polling каждые 10 сек                 | SSE/WebSocket                | Correlated incident timeline              |
| Audit        | Diff, actor, scope, result            | CSV export                   | Compliance retention/reporting            |
| Metrics      | Redis/cache baseline                  | p95/p99, error breakdown     | Cross-service correlation                 |
| Security     | RBAC + permissions                    | Step-up verification         | MFA/approval for critical commands        |

---
