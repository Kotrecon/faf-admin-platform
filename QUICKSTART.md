# Быстрый старт

**Faf Admin Platform**.

**Стек:** .NET 10 + Vanilla JavaScript

## Что уже есть

✅ Структура репозитория создана  
✅ Название: Faf Admin Platform  
✅ README.md с архитектурой  
✅ Контракт PageModel v1  
✅ Руководство по подключению сервиса  
✅ План на неделю 1  
✅ ADR 001: Schema + Data вместе  
✅ Sample для Cache Service  
✅ Admin UI Kit структура

## Что делать дальше

### День 0 (сегодня)

1. **Создать реальный Git репозиторий**

   ```bash
   cd faf-admin-platform
   git init
   git add .
   git commit -m "Initial commit: Faf Admin Platform structure"
   git remote add origin <your-repo-url>
   git push -u origin main
   ```

2. **Создать три проекта**

   ```bash
   # Admin Shell (Vanilla JavaScript)
   cd src/admin-shell
   npm init -y
   npm install vite --save-dev

   # Portal BFF (.NET 10)
   cd ../admin-portal-bff
   dotnet new webapi -n Faf.AdminPlatform.Bff

   # Admin UI Kit (уже создан)
   cd ../admin-ui-kit
   npm install @faf/foundations @faf/z-index
   ```

3. **Открыть issue/tracker**

Создать milestone "Week 1: Vertical Slice" и задачи по дням из `docs/guides/week1-vertical-slice.md`.

### День 1

- Реализовать PageModel DTO в BFF
- Создать Cache Service endpoint с mock data
- Добавить proxy endpoint в BFF
- Проверить curl: `GET /admin-api/pages/cache/overview`

### День 2

- Создать Admin Shell (одно SPA на Vanilla JS)
- Добавить layout + sidebar
- Реализовать query-param routing
- Загрузить PageModel через fetch

### День 3

- Реализовать schema renderer
- Добавить status, kpi, timeSeries widgets
- Подключить график (Chart.js или lightweight chart)

### День 4

- Добавить polling (10 секунд)
- Реализовать loading/error/stale states
- Добавить manual refresh

### День 5

- Добавить System Overview placeholder
- Проверить deep links
- Написать integration tests
- Сделать demo (GIF/скриншоты)
- Задокументировать результаты

## Чек-лист готовности недели 1

- [ ] URL `/admin?service=cache&page=overview` работает
- [ ] Shell → BFF → Service цепочка работает
- [ ] PageModel возвращается с schema + data
- [ ] 3 widget type рендерятся (status, kpi, timeSeries)
- [ ] Polling каждые 10 секунд
- [ ] Stale state при ошибке
- [ ] Correlation ID в логах
- [ ] Смена route останавливает polling

## Ссылки

- [README.md](README.md) — общая архитектура
- [Week 1 Guide](docs/guides/week1-vertical-slice.md) — детальный план
- [PageModel Contract](docs/contracts/pagemodel-v1.md) — контракт
- [Connect Guide](docs/guides/connect-new-service.md) — как подключить сервис
- [Sample](samples/cache-service-adapter/README.md) — reference implementation
- [Tech Stack](docs/TECH_STACK.md) — детали стека
- [UI Kit Arch](docs/ADMIN_UI_KIT_ARCH.md) — архитектура UI Kit

## Контакты

Вопросы и предложения — создавать issue в репозитории.

---

**Статус:** Ready to start  
**Версия:** 0.1.0-alpha  
**Дата:** 2026-09-20  
**Продукт:** Faf Admin Platform  
**Стек:** .NET 10 + Vanilla JavaScript
