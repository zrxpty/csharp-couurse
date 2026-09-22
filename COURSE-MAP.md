# Курс: C# Architect — Полный курс от нуля до Senior

> **Статус документа:** Шаг 1 — Полная карта курса (Course Map)
> **Соглашение об именовании:** `M<NN>` — модуль, `M<NN>-L<NN>` — урок, `M<NN>-PRJ` — мини-проект модуля, `CAP-PRJ` — capstone.

---

## Паспорт курса

| Параметр | Значение |
|----------|----------|
| **Версия курса** | 1.0 (MVP, Шаг 1) |
| **Последнее обновление** | 2026-09-22 |
| **Версия C#** | 12+ (с заметками о 13+) |
| **Версия .NET** | 8 LTS (с дорожками на .NET 10 / .NET 12) |
| **Язык** | Русский (структура готова к локализации EN/…) |
| **Формат** | Текст + рабочий код + практика + проекты |
| **Целевые аудитории** | Новички, Junior, Middle (переход к Senior) |
| **Треки** | Foundation / Professional / Enterprise |
| **Модулей** | 20 (+ capstone) |
| **Оценка времени** | 80–200 ч (Foundation ~40ч / Professional ~70ч / Enterprise ~100ч) |
| **Предварительные требования** | Базовое понимание программирования желательно; возможен старт с нуля |
| **Лицензия контента** | Внутренняя (определяется Tier: Free / Paid / Premium / Enterprise) |
| **Хранение** | Один модуль = одна папка; manifest-файл с метаданными |

***

## Карта курса (Course Map)

### Трек: Foundation (0 → Junior)

| Модуль | ID | Название | Сложность | Время | Prerequisites | Статус |
|--------|----|----------|-----------|-------|---------------|--------|
| Введение в C# и .NET | M01 | Экосистема .NET, окружение, первая программа | 1/5 | 4ч | — | Core |
| Синтаксис и типы данных | M02 | Переменные, типы, операторы, nullable | 2/5 | 6ч | M01 | Core |
| Управление потоком выполнения | M03 | Ветвления, циклы, pattern matching | 2/5 | 5ч | M02 | Core |
| Методы и основы ООП | M04 | Методы, классы, инкапсуляция | 3/5 | 8ч | M03 | Core |
| Коллекции и LINQ (базовый) | M05 | Массивы, List, Dictionary, базовый LINQ | 3/5 | 7ч | M04 | Core |
| Работа с исключениями | M06 | try/catch, кастомные исключения, валидация | 2/5 | 4ч | M04 | Core |
| **Итоговый проект Foundation** | M06-PRJ | Консольное приложение «Менеджер задач» | 3/5 | 8ч | M01–M06 | Core |

### Трек: Professional (Junior → Middle)

| Модуль | ID | Название | Сложность | Время | Prerequisites | Статус |
|--------|----|----------|-----------|-------|---------------|--------|
| Продвинутое ООП | M07 | Наследование, полиморфизм, интерфейсы, records | 4/5 | 8ч | M04 | Core |
| Делегаты, события, лямбды | M08 | Func/Action, events, closures | 4/5 | 7ч | M05 | Core |
| Асинхронность и Task | M09 | async/await, Task, CancellationToken, ValueTask | 4/5 | 10ч | M05 | Core |
| LINQ (продвинутый) | M10 | Deferred execution, expression trees, IQueryable | 4/5 | 8ч | M05 | Core |
| Работа с файлами и сериализация | M11 | Stream, JSON (System.Text.Json), XML | 3/5 | 6ч | M02 | Core |
| Генерики и коллекции (глубоко) | M12 | Constraints, covariance/contravariance, кастомные коллекции | 4/5 | 8ч | M05 | Core |
| Введение в базы данных (EF Core) | M13 | Code-first, миграции, запросы, трекинг | 4/5 | 10ч | M07 | Core |
| **Итоговый проект Professional** | M13-PRJ | REST API «Библиотека книг» с EF Core | 4/5 | 12ч | M07–M13 | Core |

### Трек: Enterprise (Middle → Senior)

| Модуль | ID | Название | Сложность | Время | Prerequisites | Статус |
|--------|----|----------|-----------|-------|---------------|--------|
| Архитектура приложений | M14 | Слои, DI, Clean / Hexagonal, модульность | 5/5 | 12ч | M13 | Core |
| Паттерны проектирования | M15 | GoF, SOLID на практике, refactoring | 5/5 | 14ч | M07 | Core |
| Производительность и оптимизация | M16 | Аллокации, GC, профилирование, Span/Memory | 5/5 | 12ч | M12 | Core |
| Тестирование (xUnit, моки) | M17 | xUnit, Moq/NSubstitute, TDD, integration tests | 4/5 | 10ч | M09 | Core |
| Микросервисы и распределённые системы | M18 | ASP.NET Core, REST/gRPC, resilience, observability | 5/5 | 16ч | M14 | Optional |
| Event Sourcing и CQRS | M19 | Marten/EventStore, projections, saga | 5/5 | 14ч | M14 | Optional |
| High-Performance C# | M20 | Span<T>, pooling, SIMD, low-level, NativeAOT | 5/5 | 12ч | M16 | Optional |
| **Capstone-проект** | CAP-PRJ | Распределённая система «Платёжный шлюз» | 5/5 | 20ч | M14–M20 | Core |

***

## Краткое описание модулей (M01–M20)

> Полный раскрой каждого модуля выполняется по команде `Раскрой модуль [ID]` (Шаг 2). Здесь — продуктовый концепт каждого модуля.

### Foundation Track

- **M01 · Введение в C# и .NET** — экосистема .NET (Framework → Core → 8+), установка SDK, выбор IDE (Rider / VS / VS Code), структура проекта, NuGet, первая программа. *Домен-нейтральный старт.*
- **M02 · Синтаксис и типы данных** — value vs reference, примитивы, `decimal` для денег, nullable, `var` vs `dynamic`, boxing. *База для всех доменов.*
- **M03 · Управление потоком выполнения** — `if`/`switch`/`match` (C# 7+), циклы, ранние выходы, `switch` expressions, DRY в ветвлениях.
- **M04 · Методы и основы ООП** — сигнатуры, `ref`/`out`/`in`, перегрузки, классы, инкапсуляция, `static`. Первый шаг к архитектуре.
- **M05 · Коллекции и LINQ (базовый)** — `Array`, `List<T>`, `Dictionary<K,V>`, `HashSet<T>`, базовый LINQ (`Where`, `Select`, `OrderBy`).
- **M06 · Работа с исключениями** — `try/catch/finally`, иерархия, кастомные исключения, исключения vs Result-паттерн, валидация ввода.
- **M06-PRJ · Менеджер задач (консоль)** — CRUD задач, отметка выполнения, базовая персистентность (JSON). База → pro: фильтры, сортировка, история.

### Professional Track

- **M07 · Продвинутое ООП** — наследование, полиморфизм, абстракции, интерфейсы, `record` (C# 9+), `record struct` (C# 10+), `with`-выражения.
- **M08 · Делегаты, события, лямбды** — `Action`/`Func`/`Predicate`, events, `event` vs `delegate`, замыкания, captured variables, `INotifyPropertyChanged`.
- **M09 · Асинхронность и Task** — `async/await`, `Task`/`Task<T>`/`ValueTask`, `CancellationToken`, синхронный контекст, `ConfigureAwait`, частые ошибки (deadlocks, fire-and-forget).
- **M10 · LINQ (продвинутый)** — отложенное выполнение, `IQueryable` vs `IEnumerable`, expression trees, `GroupBy`/`Join`/`Zip`, параллельный LINQ (PLINQ).
- **M11 · Работа с файлами и сериализация** — `Stream`, `FileStream`, `StreamReader/Writer`, `System.Text.Json`, опции сериализации, XML, `System.IO.Compression`.
- **M12 · Генерики и коллекции (глубоко)** — `where`-constraints, ковариация/контравариация, `IEnumerable`/`IReadOnly*`, кастомные коллекции, `Span<T>` как мост к M16/M20.
- **M13 · Введение в базы данных (EF Core)** — code-first, миграции, контекст, трекинг vs no-tracking, запросы, `Include`, транзакции, пул контекстов.
- **M13-PRJ · REST API «Библиотека книг»** — ASP.NET Core minimal API + EF Core (SQLite/Postgres). База → pro: DTO, пагинация, фильтрация, тесты.

### Enterprise Track

- **M14 · Архитектура приложений** — слои (Domain/Application/Infrastructure), DI-контейнеры, Clean Architecture, Hexagonal, модульность, boundaries.
- **M15 · Паттерны проектирования** — GoF (Creational/Structural/Behavioral), SOLID с примерами нарушений и исправлений, refactoring smells, Repository/Unit of Work/Strategy/Factory/Observer.
- **M16 · Производительность и оптимизация** — аллокации, поколения GC, профилирование (dotTrace/BenchmarkDotNet), `Span<T>`/`Memory<T>`, `ArrayPool<T>`, `stackalloc`, string pooling.
- **M17 · Тестирование (xUnit, моки)** — xUnit, Moq/NSubstitute, FluentAssertions, TDD-цикл, integration tests (WebApplicationFactory), test pyramid, мутационное тестирование.
- **M18 · Микросервисы и распределённые системы** — ASP.NET Core, REST/gRPC, HttpClientFactory, Polly, distributed tracing (OpenTelemetry), межсервисный обмен, идемпотентность.
- **M19 · Event Sourcing и CQRS** — Marten/EventStoreDB, события vs состояние, projections, read-models, saga/process manager, eventually consistent.
- **M20 · High-Performance C#** — `Span<T>`/`Memory<T>`/`ref struct`, `ArrayPool`/`ObjectPool`, SIMD (`Vector<T>`, `Vector128`), `NativeAOT`, `ref`/`in`/`out` в hot paths, P/Invoke.
- **CAP-PRJ · Платёжный шлюз** — распределённая система: API-шлюз, accounts, transactions, event log, idempotency, observability. Объединяет M14–M20.

***

## Система тегирования

Каждый элемент контента помечается тегами:

| Категория | Возможные значения |
|-----------|-------------------|
| **Уровень** | Beginner, Intermediate, Advanced |
| **Тип** | Theory, Practice, Project, Quiz, CheatSheet, Video (опционально) |
| **Домен** | Core, Web, Data, Architecture, Testing, Performance, Desktop, Mobile, GameDev |
| **Статус** | Active, Deprecated, Legacy, Upcoming |
| **Версия C#** | 12+, 13+ |
| **Версия .NET** | 8 LTS, 10, 12 |
| **Tier-доступ** | Free, Paid, Premium, Enterprise |

**Пример:** `M09-L03` → `[Intermediate] [Theory] [Core] [Active] [C#12+] [.NET8+] [Paid]`

### Доменная матрица (какие модули куда идут)

| Домен | Foundation | Professional | Enterprise |
|-------|-----------|--------------|------------|
| **Core (база для всех)** | M01–M06 | M07–M13 | M14, M15, M17 |
| **Web API** | — | M13 | M14, M18 |
| **Data-интенсивные** | M05 | M10, M13 | M16 |
| **Enterprise/микросервисы** | — | — | M18, M19 |
| **High-Performance** | — | M12 | M16, M20 |

***

## Дорожная карта развития курса

### Версия 1.0 (MVP) — Q4 2026

- Модули M01–M13 (Foundation + Professional)
- Базовые проекты для каждого модуля
- Free Tier (M01–M03)

### Версия 1.5 (Расширение) — Q2 2027

- Модули M14–M17 (Enterprise: архитектура, паттерны, тестирование)
- Capstone-проект (CAP-PRJ)
- Premium Tier (код-ревью, менторство)

### Версия 2.0 (Мажорное обновление) — Q4 2027

- Обновление под C# 13 и .NET 10
- Модули M18–M20 (микросервисы, Event Sourcing, High-Performance)
- Enterprise Tier для корпораций
- Миграция устаревших примеров → `[Deprecated]`

### Будущие треки (2028+)

- C# для микросервисов (углублённо)
- C# для Event-Driven архитектуры
- C# для High-Performance систем (Span<T>, SIMD, low-level)
- C# для миграции с .NET Framework на .NET 8+
- C# + AI/ML (ML.NET, интеграция с AI-сервисами)

***

## Продуктовая линейка

### Free Tier

- Модули M01–M03 (введение, синтаксис, поток выполнения)
- Базовые примеры кода
- Доступ к сообществу (Telegram, Discord)

### Paid Tier (Full Course)

- Все модули M01–M20 (Foundation + Professional + Enterprise)
- Все проекты (базовые + pro-версии)
- Чек-листы, шпаргалки, cheat sheets
- Сертификат о прохождении

### Premium Tier

- Всё из Paid Tier
- Персональное код-ревью проектов
- Менторство (1:1 сессии)
- Доступ к закрытым материалам (advanced-темы, кейсы)
- Помощь с резюме и подготовкой к собеседованиям

### Enterprise Tier

- Всё из Premium Tier
- Адаптация под стек компании (внутренние инструменты, домен)
- Приватные сессии для команды
- SLA и отчётность
- White-label (курс под брендом компании)

***

## Метрики и обратная связь

Для каждого модуля предусмотрены:

- **Опрос после модуля:**
  - NPS (0–10): «Насколько вероятно, что ты порекомендуешь этот модуль?»
  - Сложность (1–5): «Как ты оцениваешь сложность?»
  - Релевантность (1–5): «Насколько материал применим в работе?»
- **Трекер прогресса:**
  - Где студенты застревают
  - На каком уроке чаще всего бросают
  - Среднее время прохождения
- **A/B тесты:**
  - Разные форматы подачи (больше кода vs больше теории)
  - Разные проекты (веб vs консоль vs desktop)

***

## Глоссарий и приложения

### Глоссарий терминов

- **CLR** — Common Language Runtime, среда выполнения .NET
- **JIT** — Just-In-Time компилятор
- **GC** — Garbage Collector, сборщик мусора
- **LINQ** — Language Integrated Query
- **EF Core** — Entity Framework Core, ORM для .NET
- **CQRS** — Command Query Responsibility Segregation
- **Event Sourcing** — паттерн хранения состояния через события
- **DI** — Dependency Injection, внедрение зависимостей
- **AOT** — Ahead-Of-Time компиляция (NativeAOT в .NET 8+)
- **BCL** — Base Class Library, стандартная библиотека .NET

### Шпаргалки (план)

- [ ] Синтаксис C# (операторы, ключевые слова)
- [ ] Типы данных (когда что использовать)
- [ ] LINQ (методы, синтаксис)
- [ ] async/await (паттерны, частые ошибки)
- [ ] EF Core (миграции, запросы, производительность)

### Чек-листы по best practices

- [ ] Код-стайл C# (именование, отступы, регионы)
- [ ] SOLID принципы (примеры нарушений и исправлений)
- [ ] Обработка исключений (когда ловить, когда пробрасывать)
- [ ] Асинхронность (когда async/await уместен)
- [ ] Производительность (избегание аллокаций, pooling)

### Список инструментов

- **IDE:** JetBrains Rider, Visual Studio 2022+, VS Code + C# Dev Kit
- **Расширения:**
  - C# Extensions (VS Code)
  - ReSharper / Rider (анализ кода)
  - GitLens (работа с Git)
- **NuGet пакеты:**
  - xUnit / NUnit (тестирование)
  - Moq / NSubstitute (моки)
  - FluentAssertions (читаемые ассерты)
  - Serilog / NLog (логирование)
  - AutoMapper (маппинг объектов)
  - MediatR (CQRS, медиатор)
  - Marten (Event Sourcing + PostgreSQL)
  - Polly (resilience)
  - BenchmarkDotNet (бенчмарки)
- **Утилиты:**
  - dotnet CLI
  - dotnet-format (форматирование)
  - dotTrace / Rider Profiler (профилирование)

***

## Команды для следующих шагов

| Команда | Действие |
|---------|----------|
| `Раскрой модуль [ID]` | Детальный раскрой модуля по шаблону (цель, уроки, код, задания, проект) |
| `Добавь модуль по [тема]` | Новый модуль с уникальным ID, метками, зависимостями |
| `Создай версию 2.0` | Обновление под C# 13 / .NET 10, новые модули, `[Deprecated]`-метки |
| `Адаптируй под [домен]` | Выделение модулей/проектов под домен (Web, Data, Enterprise, High-Performance) |

---

**Карта курса готова.** Жду подтверждения структуры, после чего перехожу к детальному раскрытию модулей по команде `Раскрой модуль [ID]`.
