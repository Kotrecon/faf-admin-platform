# ТЗ: Двухуровневая навигация универсальной админ-панели

**Версия:** 0.1  
**Статус:** Draft  
**Контекст:** внутренняя универсальная административная панель для управления системой и набором backend-сервисов.  
**Цель:** реализовать масштабируемую, контекстную и доступную навигацию для общесистемных dashboard-страниц и service-specific панелей.

---

## 1. Назначение

Навигация должна поддерживать два разных уровня работы:

1. **Общесистемный уровень** — страницы, не привязанные к одному сервису: системный обзор, пользователи, использование AI-моделей, токены, общие алерты, общий audit.
2. **Сервисный уровень** — страницы конкретного сервиса: например, для Cache Service это `Overview`, `Keys`, `TTL`, `Maintenance`, `Activity`, `Audit`.

Основной паттерн — **двухуровневая навигация**:

```text
┌──────────────────────┬──────────────────────┬────────────────────────────┐
│ Primary navigation   │ Context navigation   │ Main content               │
│                      │                      │                            │
│ Общие дашборды       │ Cache Service        │ Cache Overview             │
│ Сервисы              │ ├── Overview         │                            │
│ Системные инструменты│ ├── Keys             │ KPI / charts / tables      │
│                      │ ├── TTL              │                            │
│                      │ ├── Maintenance      │                            │
│                      │ ├── Activity         │                            │
│                      │ └── Audit            │                            │
└──────────────────────┴──────────────────────┴────────────────────────────┘
```

Первый sidebar является глобальным и постоянным. Второй sidebar является контекстным и отображается при выборе сервиса или иной вложенной области.

---

## 2. Термины

| Термин              | Описание                                                                        |
| ------------------- | ------------------------------------------------------------------------------- |
| **Primary sidebar** | Первый, глобальный левый sidebar                                                |
| **Context sidebar** | Второй sidebar с локальной навигацией выбранного сервиса                        |
| **Global page**     | Страница, не зависящая от одного сервиса                                        |
| **Service page**    | Страница, относящаяся к конкретному сервису                                     |
| **Service context** | Текущий выбранный сервис                                                        |
| **Flyout preview**  | Временное меню страниц сервиса при hover/focus                                  |
| **Pinned context**  | Сервисный контекст, закреплённый после click или прямого URL                    |
| **Capability**      | Возможность, которую сервис поддерживает, например `keyEnumeration` или `audit` |
| **Permission**      | Разрешение текущего пользователя на просмотр страницы или выполнение команды    |

---

## 3. Компоновка

### Desktop layout

```text
┌───────────────┬────────────────────────┬────────────────────────────────┐
│ Primary       │ Context sidebar        │ Main content                   │
│ sidebar       │                        │                                │
│               │ Selected service       │ Header / breadcrumbs           │
│ Dashboards    │ Status + environment   │ Filters / actions              │
│ - Overview    │                        │                                │
│ - Users       │ Service pages          │ Current page content           │
│ - AI Usage    │ - Overview             │                                │
│ - Alerts      │ - Metrics              │                                │
│               │ - Keys                 │                                │
│ Services      │ - Maintenance          │                                │
│ - AI Router   │ - Activity             │                                │
│ - Prompting   │ - Audit                │                                │
│ - Ingestion   │                        │                                │
│ - Cache       │                        │                                │
│               │                        │                                │
│ Tools         │                        │                                │
│ - Documentation                              │                           │
│ - Settings                                   │                           │
└───────────────┴────────────────────────┴────────────────────────────────┘
```

### Размеры

| Элемент                    |                                 Размер | Требование                                           |
| -------------------------- | -------------------------------------: | ---------------------------------------------------- |
| Primary sidebar            | 64–88 px compact / 220–260 px expanded | Может сворачиваться                                  |
| Context sidebar            |                             220–280 px | Показывается при закреплённом service context        |
| Main content               |                      Остаточная ширина | Не менее 900 px для комфортной работы с таблицами    |
| Верхняя панель             |                               48–64 px | Может содержать breadcrumbs, environment, user menu  |
| Минимальная рабочая ширина |                                1280 px | MVP ориентирован на desktop                          |
| Focus mode                 |             Context sidebar скрывается | Основной контент занимает освобождённое пространство |

---

## 4. Primary sidebar

Primary sidebar делится на логические секции.

### 4.1. Секция «Общие дашборды»

Содержит только кросс-сервисные страницы.

| Пункт              | Назначение                                                  | URL                     |
| ------------------ | ----------------------------------------------------------- | ----------------------- |
| Системный обзор    | Общее состояние платформы                                   | `?page=system-overview` |
| Пользователи       | Пользователи всей системы, активность, предпочтения моделей | `?page=users`           |
| Использование AI   | Токены, usage моделей, распределение нагрузки               | `?page=ai-usage`        |
| Общие алерты       | Алерты по всем сервисам                                     | `?page=alerts`          |
| Проблемы           | Агрегированные ошибки и инциденты                           | `?page=problems`        |
| Производительность | Системная производительность                                | `?page=performance`     |
| Аптайм             | Доступность сервисов                                        | `?page=uptime`          |
| Общий Audit        | Кросс-сервисная история административных действий           | `?page=audit`           |

В MVP можно начать с ограниченного набора:

```text
Системный обзор
Пользователи
Общие алерты
Общий Audit
```

Остальные страницы добавляются по мере готовности observability и бизнес-функций.

### 4.2. Секция «Сервисы»

Содержит список зарегистрированных сервисов, например:

```text
AI Router
Prompting
Ingestion
Cache Service
Knowledge Base
Backend API
...
```

Для каждого сервиса отображаются:

| Поле            | Требование                                                      |
| --------------- | --------------------------------------------------------------- |
| Иконка          | Опционально, из конфигурации сервиса                            |
| Имя             | Обязательно                                                     |
| Статус          | Online / Degraded / Offline / Unknown                           |
| Severity marker | Если сервис имеет active critical alert                         |
| Environment     | Необязательно на уровне элемента, если общий environment единый |
| Tooltip         | В compact mode показывает полное название и статус              |

### 4.3. Секция «Инструменты»

Опциональная нижняя секция:

```text
Документация
Настройки
Справка
Профиль
Выход
```

Системные settings не должны смешиваться со страницами конкретного сервиса.

---

## 5. Context sidebar

### Назначение

Context sidebar показывает локальную навигацию выбранного сервиса и становится постоянным после того, как пользователь:

- кликнул по сервису;
- выбрал страницу сервиса в flyout;
- открыл прямую service URL;
- перешёл по ссылке из alert, audit, activity feed или dashboard.

### Структура

```text
┌─────────────────────────────┐
│ Cache Service               │
│ ● Online                    │
│ PRODUCTION                  │
├─────────────────────────────┤
│ Overview                    │
│ Keys                        │
│ TTL & Distribution          │
│ Maintenance                 │
│ Activity                    │
│ Audit                       │
└─────────────────────────────┘
```

### Header контекстного sidebar

| Элемент                | Требование                                                           |
| ---------------------- | -------------------------------------------------------------------- |
| Service name           | Обязателен                                                           |
| Service status         | Online / Degraded / Offline / Unknown                                |
| Environment            | Обязателен для operational services                                  |
| Collapse button        | Сворачивает context sidebar в focus mode                             |
| Back/close action      | Возврат к global navigation без pinned service context               |
| Optional quick actions | Только для безопасных действий; dangerous actions не размещать здесь |

### Страницы Cache Service

| Страница           | Назначение                                       |
| ------------------ | ------------------------------------------------ |
| Overview           | Health, KPI, базовые графики                     |
| Keys               | Key explorer, поиск, фильтры, пагинация          |
| TTL & Distribution | TTL histogram и аналитика TTL                    |
| Maintenance        | Clear expired, flush namespace/all с защитой     |
| Activity           | Последние технические и административные события |
| Audit              | История действий оператора                       |

### Пример страниц AI Router

| Страница    | Назначение                          |
| ----------- | ----------------------------------- |
| Overview    | Статус, KPI, latency, errors        |
| Requests    | Потоки запросов и результаты        |
| Models      | Используемые AI-модели и приоритеты |
| Rate limits | Использование лимитов               |
| Resilience  | Retries, circuit breaker, failures  |
| Alerts      | Alerts только этого сервиса         |
| Audit       | История административных изменений  |

### Формирование списка страниц

Набор пунктов context sidebar не является статическим. Он строится на основании:

1. конфигурации сервиса;
2. capabilities сервиса;
3. permissions текущего пользователя;
4. доступности сервиса.

Например, если Cache Provider не поддерживает key enumeration, пункт `Keys` не отображается. Если пользователь не имеет `cache:maintenance`, `Maintenance` не показывается.

---

## 6. Поведение hover, click и focus

### Основной принцип

**Hover — только preview. Click — изменение контекста.**

Навигация не должна быть доступна исключительно через hover.

### Hover / mouse enter

При наведении на сервис в primary sidebar:

1. Открывается flyout preview справа от primary sidebar.
2. Показывается название сервиса, status и список доступных страниц.
3. Main content не меняется.
4. URL не меняется.
5. Сервисный контекст не закрепляется.

```text
┌─────────────────────────────┐
│ Cache Service               │
│ ● Online                    │
├─────────────────────────────┤
│ Overview                    │
│ Keys                        │
│ TTL & Distribution          │
│ Maintenance                 │
│ Activity                    │
│ Audit                       │
└─────────────────────────────┘
```

### Click по имени сервиса

При клике по элементу сервиса:

1. Сервис становится текущим service context.
2. Открывается context sidebar.
3. Пользователь переходит на default service page.
4. URL обновляется.
5. Сервис сохраняется как pinned context.

Пример:

```text
/admin?service=cache&page=overview
```

### Click по пункту flyout

При клике по конкретной странице во flyout:

1. Сервис закрепляется.
2. Открывается context sidebar.
3. Открывается выбранная страница.
4. URL обновляется.

Пример:

```text
/admin?service=cache&page=maintenance
```

### Focus / keyboard navigation

При переходе Tab на элемент сервиса flyout должен открываться аналогично hover.

Клавиатурный сценарий:

| Клавиша                 | Поведение                                    |
| ----------------------- | -------------------------------------------- |
| `Tab`                   | Переход между элементами primary sidebar     |
| `Enter` / `Space`       | Закрепить сервис и открыть default page      |
| `ArrowRight`            | Открыть flyout или перейти в context sidebar |
| `ArrowDown` / `ArrowUp` | Навигация между service pages                |
| `Enter`                 | Открыть текущую service page                 |
| `Escape`                | Закрыть flyout, не меняя main content        |
| `Ctrl/Cmd + K`          | Опционально: глобальный command/search menu  |

### Touch

На touch-устройствах hover не используется:

- первый tap по сервису открывает или закрепляет service context;
- второй tap по странице открывает страницу;
- допускается отдельная кнопка раскрытия у service item.

---

## 7. Состояния навигации

| Состояние        | Primary sidebar                 | Context sidebar                           | Main content                  |
| ---------------- | ------------------------------- | ----------------------------------------- | ----------------------------- |
| Global page      | Показывает активный global item | Скрыт или показывает global context       | Общесистемная страница        |
| Hover preview    | Активен hovered service         | Временный flyout                          | Не меняется                   |
| Service overview | Активен выбранный сервис        | Открыт и закреплён                        | Default page сервиса          |
| Service page     | Активен выбранный сервис        | Открыт, активна текущая service page      | Страница сервиса              |
| Focus mode       | Может быть compact              | Скрыт                                     | Main content расширен         |
| Service offline  | Сервис выбран                   | Открыт, actions disabled по необходимости | Доступны данные/ошибки/health |
| Forbidden page   | Сервис выбран                   | Пункт скрыт или disabled по policy        | Страница 403/access denied    |
| Unknown route    | Контекст определяется из URL    | По возможности открыт                     | 404/not found state           |

---

## 8. URL и восстановление состояния

### Общесистемные страницы

```text
/admin?page=system-overview
/admin?page=users
/admin?page=ai-usage
/admin?page=alerts
/admin?page=problems
/admin?page=audit
```

### Сервисные страницы

```text
/admin?service=cache&page=overview
/admin?service=cache&page=keys
/admin?service=cache&page=ttl
/admin?service=cache&page=maintenance

/admin?service=ai-router&page=rate-limits
/admin?service=prompting&page=templates
```

### Состояние таблиц и фильтров

Состояние конкретной страницы должно быть serializable в query params:

```text
/admin?service=cache&page=keys
  &search=user%3A
  &ttl=lt-1h
  &pageNumber=1
  &limit=50
```

### Требования

| Требование      | Детали                                                                         |
| --------------- | ------------------------------------------------------------------------------ |
| Deep link       | Любой URL должен открывать корректный service/page context                     |
| Browser history | Back/Forward восстанавливает сервис, страницу и параметры                      |
| Bookmark        | URL пригоден для сохранения и передачи                                         |
| Refresh         | Состояние восстанавливается после перезагрузки                                 |
| Invalid service | Показать понятный 404/Service unavailable                                      |
| Forbidden page  | Показать 403 или скрыть пункт по выбранной политике                            |
| Migration       | При переименовании service slug старые ссылки могут поддерживаться redirect-ом |

---

## 9. Конфигурация сервисов

Frontend получает список сервисов и их navigation metadata через конфигурационный API или статический config, развиваемый в централизованном shared проекте.

### Минимальная модель сервиса

| Поле           | Назначение                                       |
| -------------- | ------------------------------------------------ |
| `id`           | Стабильный внутренний идентификатор              |
| `slug`         | Стабильный идентификатор в URL, например `cache` |
| `displayName`  | Отображаемое имя                                 |
| `description`  | Краткое описание в tooltip/flyout                |
| `icon`         | Иконка сервиса                                   |
| `order`        | Порядок сортировки                               |
| `status`       | Online / Degraded / Offline / Unknown            |
| `defaultPage`  | Страница по умолчанию                            |
| `pages`        | Массив service pages                             |
| `capabilities` | Поддерживаемые возможности                       |
| `permissions`  | Доступные текущему пользователю права            |
| `environment`  | Environment context при необходимости            |

### Минимальная модель service page

| Поле                   | Назначение                                          |
| ---------------------- | --------------------------------------------------- |
| `id`                   | Внутренний идентификатор страницы                   |
| `slug`                 | Идентификатор в URL                                 |
| `title`                | Название в sidebar                                  |
| `icon`                 | Иконка                                              |
| `order`                | Порядок отображения                                 |
| `requiredCapabilities` | Какие возможности нужны для отображения             |
| `requiredPermissions`  | Какие права нужны текущему пользователю             |
| `default`              | Является ли страницей по умолчанию                  |
| `badge`                | Опциональный счётчик/статус: alerts, errors и т. п. |
| `enabled`              | Сервис может временно отключить страницу            |

### Пример смысловой конфигурации Cache Service

```text
Cache Service
default page: overview

pages:
- overview
- keys             requires: keyEnumeration
- ttl              requires: ttlInspection
- maintenance      requires: maintenance permission
- activity         requires: activityFeed
- audit            requires: audit permission
```

### Правило capabilities

Недоступная capability — штатное состояние, а не ошибка.

| Capability отсутствует | Поведение UI                                                 |
| ---------------------- | ------------------------------------------------------------ |
| Нет key enumeration    | Не показывать `Keys`, оставить точечные операции при наличии |
| Нет TTL metadata       | Не показывать `TTL & Distribution`                           |
| Нет maintenance        | Не показывать `Maintenance`                                  |
| Нет activity feed      | Не показывать `Activity`                                     |
| Нет audit              | Не показывать `Audit`                                        |
| Нет memory metrics     | Скрыть соответствующую KPI-карточку или показать `N/A`       |

---

## 10. Permissions и видимость

### Базовая политика

Backend является источником истины для authorization. Frontend использует permissions только для отображения доступного интерфейса.

| Ситуация                               | Поведение                                                       |
| -------------------------------------- | --------------------------------------------------------------- |
| Нет права на page                      | Пункт скрывается по умолчанию                                   |
| Есть право чтения, нет права команды   | Страница доступна, command buttons скрыты/disabled              |
| Нет права на dangerous action          | Кнопка не рендерится                                            |
| Прямой URL к закрытой странице         | Backend/API возвращает 403; frontend показывает access denied   |
| Permission изменился в процессе сессии | Следующий API call возвращает актуальный статус; UI обновляется |

### Роли текущей платформы

| Роль             |      Общесистемные dashboard pages |              Сервисные панели |                Dangerous operations |
| ---------------- | ---------------------------------: | ----------------------------: | ----------------------------------: |
| `Admin`          |                                 Да | В рамках выданных permissions |                         Ограниченно |
| `SecurityAdmin`  |                                 Да |                            Да | Да, при соответствующих permissions |
| `ContentManager` | Только разрешённые бизнес-страницы |    Только разрешённые сервисы |                    Нет по умолчанию |

---

## 11. Focus mode и адаптивность

### Focus mode

Focus mode нужен для работы с широкими таблицами, сложными графиками и log/audit views.

Поведение:

1. Context sidebar скрывается.
2. Main content получает дополнительную ширину.
3. Primary sidebar остаётся в compact mode или также может быть скрыт.
4. Состояние сохраняется на время текущей сессии.
5. Пользователь может вернуть навигацию одной кнопкой или горячей клавишей.

```text
┌───────────┬───────────────────────────────────────────────────────┐
│ Compact   │ Main content                                           │
│ primary   │                                                       │
│ sidebar   │ Wide table / chart / audit log                         │
└───────────┴───────────────────────────────────────────────────────┘
```

### Breakpoints

| Ширина        | Поведение                                                                                  |
| ------------- | ------------------------------------------------------------------------------------------ |
| `>= 1440px`   | Оба sidebar раскрыты                                                                       |
| `1280–1439px` | Primary compact, context sidebar раскрыт                                                   |
| `< 1280px`    | Context sidebar в overlay/drawer; desktop MVP может предупреждать о рекомендованной ширине |
| Touch/tablet  | Primary и context navigation через drawer, без hover-зависимости                           |

---

## 12. Визуальные требования

### Активные состояния

| Элемент              | Требование                                          |
| -------------------- | --------------------------------------------------- |
| Active global page   | Контрастный background + indicator                  |
| Active service       | Выделение в primary sidebar                         |
| Active service page  | Выделение в context sidebar                         |
| Hover service        | Preview flyout без смены main content               |
| Service degraded     | Status badge + severity icon                        |
| Service offline      | Приглушённый item, но не обязательно скрытый        |
| Unread/active alerts | Badge с числом или severity marker                  |
| Disabled action      | Явно disabled, tooltip с причиной при необходимости |

### Состояния service item

```text
● Online       — зелёный + текст Online
▲ Degraded     — янтарный + текст Degraded
× Offline      — красный + текст Offline
? Unknown      — серый + текст Unknown
```

Цвет используется как вторичный сигнал. Основной смысл передаётся текстом и иконкой.

---

## 13. Accessibility

| Требование          | Детали                                                                                 |
| ------------------- | -------------------------------------------------------------------------------------- |
| Keyboard navigation | Полная навигация без мыши                                                              |
| Focus management    | При открытии flyout focus не должен теряться                                           |
| Escape              | Закрывает flyout/overlay                                                               |
| ARIA roles          | `nav`, `menu`, `menuitem`, `aria-expanded`, `aria-current` — по используемому паттерну |
| Tooltips            | Для иконок в compact sidebar                                                           |
| Contrast            | Соответствие WCAG AA минимум                                                           |
| Цвет                | Не единственный носитель статуса                                                       |
| Screen reader       | Название сервиса, статус и текущая страница доступны как текст                         |
| Reduced motion      | Учитывать системную настройку `prefers-reduced-motion`                                 |

---

## 14. Telemetry и аудит навигации

Навигация сама по себе не требует полноценного audit log, но полезно собирать технические и продуктовые события:

| Событие                                | Назначение                                      |
| -------------------------------------- | ----------------------------------------------- |
| `admin.navigation.global_page_opened`  | Использование системных страниц                 |
| `admin.navigation.service_opened`      | Переход к сервису                               |
| `admin.navigation.service_page_opened` | Использование service pages                     |
| `admin.navigation.focus_mode_changed`  | Проверка полезности focus mode                  |
| `admin.navigation.route_not_found`     | Выявление битых ссылок                          |
| `admin.navigation.access_denied`       | Контроль UX и permissions                       |
| `admin.navigation.flyout_opened`       | Опционально, с sampling; не обязательно для MVP |

Не логировать в telemetry чувствительные query params, например поисковые значения, идентификаторы пользователей или значения ключей cache.

---

## 15. Acceptance Criteria

Реализация считается готовой для MVP, когда выполняются все условия:

### Primary navigation

- Есть отдельные секции `Общие дашборды`, `Сервисы` и `Инструменты`.
- Общие страницы не требуют выбранного service context.
- В списке сервисов отображаются имя и состояние.
- Активные страницы и активный сервис визуально выделены.

### Service navigation

- Hover/focus по сервису открывает preview с доступными service pages.
- Hover не меняет URL и main content.
- Click по сервису закрепляет service context.
- Click по странице сервиса открывает нужную страницу и закрепляет контекст.
- Context sidebar отображает страницы только для текущего сервиса.
- Список страниц учитывает capabilities и permissions.

### URL и состояние

- Поддерживаются direct URL для global и service pages.
- Refresh восстанавливает service/page context.
- Browser Back/Forward работает корректно.
- Query params таблиц и фильтров сохраняются в URL.

### UX и accessibility

- Навигация работает с мышью, touch и клавиатурой.
- `Escape` закрывает flyout.
- Есть focus mode.
- У сервисов есть понятные states: Online, Degraded, Offline, Unknown.
- Layout не ломает широкие таблицы при стандартной desktop ширине.

### Security

- Frontend скрывает недоступные пункты на основе permissions.
- Backend независимо проверяет authorization на каждом API request.
- Прямой переход на запрещённую страницу корректно обрабатывается как access denied.
- Dangerous actions находятся внутри соответствующей service page и не доступны из hover preview.

---

## 16. Роадмап

| Блок            | MVP                                  | Следующий этап             | Позже                                      |
| --------------- | ------------------------------------ | -------------------------- | ------------------------------------------ |
| Primary sidebar | Общие дашборды, сервисы, инструменты | Badges и health markers    | Персональная настройка порядка             |
| Flyout preview  | Hover/focus preview                  | Быстрые service actions    | Персонализированные shortcuts              |
| Context sidebar | Pinned service + pages               | Группы/разделители страниц | Избранные страницы                         |
| URL routing     | Global/service routes + query state  | Redirect/migration routes  | Shareable workspace state                  |
| Capabilities    | Статический config + permissions     | Dynamic capabilities API   | Service self-registration                  |
| Focus mode      | Скрытие context sidebar              | Сохранение per user        | Полноценные layout presets                 |
| Responsive      | Desktop-first                        | Drawer для tablet          | Полная mobile navigation                   |
| Accessibility   | Keyboard, ARIA, Escape               | Automated a11y tests       | Пользовательские accessibility preferences |

---
