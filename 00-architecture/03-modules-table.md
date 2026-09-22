# Таблица модулей M01–M20 / Modules Table

> Полный набор карточек модулей курса «Полный курс по C#».
> Complete set of module cards for the «Complete C# Course».
>
> Соглашения / Conventions:
> - **Уровень / Level** — первичный трек модуля (Foundation / Professional / Enterprise).
> - **Статус / Status** — Core (ядро, обязательно) / Optional (опционально) / Pro (продвинутый) / Deprecated (устаревший, справочно).
> - **Домен / Domain** — Core / Data / Concurrency / Web / Testing / Architecture / Performance / Security / DevOps.
> - **Prerequisites** — ID модулей, которые должны быть пройдены раньше.
> - **Следующие модули / Next** — модули, для которых данный модуль является прямым пререквизитом.
> - Часы: **теория / практика / проект**. Сумма по всем модулям ≈ 340 ч (120 теория + 140 практика + 80 проекты), согласована с паспортом.

---

## Модуль M01: Введение в C# и .NET / Intro to C# & .NET

**ID:** M01
**Уровень / Level:** Foundation
**Цель / Goal:** Понять, что такое C#, .NET, CLR, SDK, CLI; установить окружение, создать и запустить первое приложение, понять цикл компиляции и структуру проекта.
**Время / Time:** теория 3ч / практика 3ч / проект 0ч (итого 6ч)
**Сложность / Difficulty:** 1/5
**Prerequisites:** нет / none
**Следующие модули / Next:** M02
**Статус / Status:** Core
**Домен / Domain:** Core
**Уроки / Lessons:** 6 уроков — (1) Что такое C# и .NET (Framework vs Core vs 5/6/7/8); (2) CLR, IL, JIT, сборки; (3) Установка SDK, IDE, `dotnet` CLI; (4) `dotnet new`, структура проекта, `Program.cs`, top-level statements; (5) Компиляция, запуск, `dotnet run`/`build`; (6) Первый Hello World, отладка в IDE, NuGet-базис.

---

## Модуль M02: Типы, переменные, операторы / Types, Variables & Operators

**ID:** M02
**Уровень / Level:** Foundation
**Цель / Goal:** Освоить систему типов C# (значимые/ссылочные), переменные, константы, операторы, неявную типизацию (`var`), базовые литералы и преобразования типов.
**Время / Time:** теория 5ч / практика 5ч / проект 0ч (итого 10ч)
**Сложность / Difficulty:** 2/5
**Prerequisites:** M01
**Следующие модули / Next:** M03, M04
**Статус / Status:** Core
**Домен / Domain:** Core
**Уроки / Lessons:** 8 уроков — (1) Значимые и ссылочные типы, stack vs heap; (2) Примитивы: `int`, `double`, `decimal`, `bool`, `char`, `string`; (3) Объявление переменных, `var`, константы; (4) Операторы: арифметика, сравнение, логика, побитовые; (5) `null` и nullable-типы (`int?`); (6) Преобразования типов: явные/неявные, `Convert`, `Parse`, `TryParse`; (7) `string` основы, интерполяция, `StringBuilder`; (8) `enum` и кортежи (кратко).

---

## Модуль M03: Управление потоком, методы / Control Flow & Methods

**ID:** M03
**Уровень / Level:** Foundation
**Цель / Goal:** Владеть управляющими конструкциями (`if`/`switch`/pattern matching, циклы), объявлять и вызывать методы, понимать параметры, перегрузку и область видимости.
**Время / Time:** теория 5ч / практика 6ч / проект 2ч (итого 13ч)
**Сложность / Difficulty:** 2/5
**Prerequisites:** M02
**Следующие модули / Next:** M04, M07
**Статус / Status:** Core
**Домен / Domain:** Core
**Уроки / Lessons:** 8 уроков — (1) `if`/`else`, тернарный оператор; (2) `switch`, `switch` expressions, pattern matching (основы); (3) Циклы `for`, `while`, `do-while`, `foreach`; (4) `break`/`continue`/`return`; (5) Объявление методов, сигнатура, возвращаемые значения; (6) Параметры: `ref`, `out`, `in`, `params`, значения по умолчанию; (7) Перегрузка методов, область видимости; (8) Рекурсия и Tail-оптимизация (обзор).
*Проект:* мини-калькулятор с меню в консоли (объединяет поток управления и методы).

---

## Модуль M04: ООП: классы, объекты / OOP Basics

**ID:** M04
**Уровень / Level:** Foundation
**Цель / Goal:** Моделировать предметную область через классы и объекты: поля, свойства, конструкторы, методы, инкапсуляция, `static`, `readonly`, `const`.
**Время / Time:** теория 6ч / практика 7ч / проект 3ч (итого 16ч)
**Сложность / Difficulty:** 3/5
**Prerequisites:** M02, M03
**Следующие модули / Next:** M05
**Статус / Status:** Core
**Домен / Domain:** Core
**Уроки / Lessons:** 9 уроков — (1) Класс vs объект, объявление класса; (2) Поля, свойства (auto-properties, init-only, `required`); (3) Конструкторы и инициализаторы; (4) Инкапсуляция: модификаторы доступа; (5) `static` vs instance, `const` vs `readonly`; (6) `this`, индексаторы; (7) `record` (кратко, для value-семантики); (8) `struct` vs `class`; (9) Пространства имён, `using`, `namespace`.
*Проект:* модель «Библиотека» (классы `Book`, `Reader`, инкапсуляция состояния).

---

## Модуль M05: ООП: наследование, полиморфизм / OOP Advanced

**ID:** M05
**Уровень / Level:** Foundation
**Цель / Goal:** Применять наследование, полиморфизм, абстракцию: базовые/производные классы, `virtual`/`override`/`abstract`, интерфейсы, `sealed`, `base`, covariance/contravariance (обзор).
**Время / Time:** теория 6ч / практика 7ч / проект 3ч (итого 16ч)
**Сложность / Difficulty:** 3/5
**Prerequisites:** M04
**Следующие модули / Next:** M06, M16
**Статус / Status:** Core
**Домен / Domain:** Core
**Уроки / Lessons:** 9 уроков — (1) Наследование, `base`, конструкторы базового класса; (2) `virtual`/`override`/`new`; (3) Абстрактные классы и методы; (4) Интерфейсы, default interface methods, множественная реализация; (5) Полиморфизм на практике; (6) `sealed`, `internal`; (7) `is`/`as`, pattern matching типов; (8) `record` и inheritance, `with`; (9) Composition vs inheritance (вступление к SOLID).
*Проект:* иерархия «Фигуры» с вычислением площади (полиморфизм).

---

## Модуль M06: Коллекции и дженерики / Collections & Generics

**ID:** M06
**Уровень / Level:** Foundation
**Цель / Goal:** Использовать дженерики (классы, методы, ограничения `where`) и стандартные коллекции (`List<T>`, `Dictionary<K,V>`, `HashSet<T>`, `Queue/Stack`); понимать `IEnumerable<T>`, `IEnumerator`, yield.
**Время / Time:** теория 5ч / практика 6ч / проект 3ч (итого 14ч)
**Сложность / Difficulty:** 3/5
**Prerequisites:** M05
**Следующие модули / Next:** M07, M08
**Статус / Status:** Core
**Домен / Domain:** Core
**Уроки / Lessons:** 8 уроков — (1) Зачем дженерики, `List<T>`; (2) Обобщённые классы и методы; (3) Ограничения `where` (class/struct/new/interface); (4) `Dictionary<K,V>`, `HashSet<T>`, равенство и `GetHashCode`; (5) `Queue<T>`, `Stack<T>`, `SortedList`, `LinkedList`; (6) `IEnumerable<T>` / `IEnumerator<T>`, итерация; (7) `yield return`, ленивые последовательности; (8) `Span<T>` / `Memory<T>` (обзор, Optional).
*Проект:* обобщённый репозиторий в памяти с поиском.

---

## Модуль M07: Обработка исключений / Exception Handling

**ID:** M07
**Уровень / Level:** Foundation
**Цель / Goal:** Корректно обрабатывать ошибки: `try`/`catch`/`finally`, типы исключений, кастомные исключения, `when`, `throw`/`rethrow`, `exception filters`, связь с `Result`-паттерном (обзор).
**Время / Time:** теория 4ч / практика 5ч / проект 2ч (итого 11ч)
**Сложность / Difficulty:** 2/5
**Prerequisites:** M03, M06
**Следующие модули / Next:** M08, M09
**Статус / Status:** Core
**Домен / Domain:** Core
**Уроки / Lessons:** 7 уроков — (1) Исключения vs коды возврата, иерархия `Exception`; (2) `try`/`catch`/`finally`, порядок catch; (3) `throw` и `throw;` (rethrow) — разница стеков; (4) Кастомные исключения, свойства; (5) `when`-фильтры; (6) `InnerException`, цепочки; (7) `Result<T>`/`OneOf` (Optional), когда не использовать исключения.
*Проект:* устойчивый парсер входных данных с логированием ошибок.

---

## Модуль M08: LINQ

**ID:** M08
**Уровень / Level:** Foundation
**Цель / Goal:** Свободно писать запросы LINQ (method/query syntax): `Where`, `Select`, `OrderBy`, `GroupBy`, `Join`, агрегации, `First`/`Single`/`Any`; понимать отложенное выполнение и `IQueryable` vs `IEnumerable`.
**Время / Time:** теория 7ч / практика 7ч / проект 3ч (итого 17ч)
**Сложность / Difficulty:** 3/5
**Prerequisites:** M06, M07
**Следующие модули / Next:** M10, M12
**Статус / Status:** Core
**Домен / Domain:** Data
**Уроки / Lessons:** 9 уроков — (1) Основы LINQ, method vs query syntax; (2) `Where`, `Select`, проекции; (3) `OrderBy`/`ThenBy`, `Reverse`; (4) `GroupBy`, `ToLookup`; (5) `Join`, `GroupJoin`, `Zip`; (6) Агрегаты: `Sum`/`Min`/`Max`/`Average`/`Count`; (7) `First`/`SingleOrDefault`/`ElementAt` и исключения; (8) Отложенное vs немедленное выполнение, `ToList`/`ToArray`; (9) `IEnumerable` vs `IQueryable`, провайдеры (вступление к EF).
*Проект:* аналитика по набору данных (топ-N, группировки, join).

---

## Модуль M09: Асинхронное программирование / Async Programming (async/await)

**ID:** M09
**Уровень / Level:** Foundation
**Цель / Goal:** Понимать модель `async`/`await`, `Task`/`Task<T>`/`ValueTask`, контекст синхронизации, `ConfigureAwait`, cancellation (`CancellationToken`), типичные ошибки (deadlocks, fire-and-forget).
**Время / Time:** теория 7ч / практика 7ч / проект 3ч (итого 17ч)
**Сложность / Difficulty:** 4/5
**Prerequisites:** M07
**Следующие модули / Next:** M10, M11, M17
**Статус / Status:** Core
**Домен / Domain:** Concurrency
**Уроки / Lessons:** 10 уроков — (1) Зачем async, потоки vs async I/O; (2) `Task`, `Task<T>`, создание и ожидание; (3) `async`/`await`, компиляция в state machine; (4) `Task.Run`, CPU-bound vs I/O-bound; (5) `WhenAll`/`WhenAny`; (6) `CancellationToken`, cooperative cancellation; (7) `ConfigureAwait(false)`, `SynchronizationContext`; (8) `ValueTask`, кэшированные результаты; (9) Антипаттерны: fire-and-forget, `.Result`, deadlocks; (10) Async streams `IAsyncEnumerable`.
*Проект:* параллельный загрузчик нескольких ресурсов с cancellation и прогрессом.

---

## Модуль M10: Файлы, потоки, сериализация / I/O & Serialization

**ID:** M10
**Уровень / Level:** Foundation
**Цель / Goal:** Работать с файловой системой, потоками (`Stream`, `FileStream`, `StreamReader/Writer`), async I/O, сериализацией (JSON — `System.Text.Json`, XML — обзор), путями и кодировками.
**Время / Time:** теория 5ч / практика 5ч / проект 8ч (итого 18ч)
**Сложность / Difficulty:** 3/5
**Prerequisites:** M08, M09
**Следующие модули / Next:** M12
**Статус / Status:** Core
**Домен / Domain:** Data
**Уроки / Lessons:** 8 уроков — (1) `File`/`FileInfo`/`Directory`/`Path`; (2) `Stream`, `FileStream`, буферизация; (3) `StreamReader`/`StreamWriter`, кодировки; (4) Async файловые операции; (5) `System.Text.Json`: сериализация/десериализация, опции, `JsonSerializerContext` (source-gen); (6) `MemoryStream`, `PipeReader`/`PipeWriter` (обзор, Optional); (7) XML-сериализация (Optional/Deprecated-вектор для legacy); (8) Binary-форматы (Protocol Buffers/MemoryPack — обзор, Optional).
*Проект (Foundation capstone):* консольное CRUD-приложение с хранением данных в JSON-файле, async I/O, LINQ-запросами и обработкой исключений. Объединяет M01–M10.

---

## Модуль M11: Многопоточное программирование / Multithreading

**ID:** M11
**Уровень / Level:** Professional
**Цель / Goal:** Понимать потоки, пул потоков, синхронизацию (`lock`, `Monitor`, `SemaphoreSlim`, `Interlocked`), `ConcurrentDictionary` и другие concurrent-коллекции, `Channel<T>`, различия потоков и задач, тонкости `volatile` и барьеров памяти.
**Время / Time:** теория 7ч / практика 8ч / проект 5ч (итого 20ч)
**Сложность / Difficulty:** 4/5
**Prerequisites:** M09
**Следующие модули / Next:** M13, M17
**Статус / Status:** Core
**Домен / Domain:** Concurrency
**Уроки / Lessons:** 10 уроков — (1) `Thread` (исторически), пул потоков, почему `Task`; (2) `lock`/`Monitor`, критические секции; (3) `Interlocked`, атомарные операции; (4) `SemaphoreSlim`, `Mutex`, `ReaderWriterLockSlim`; (5) `volatile`, барьеры памяти, модель памяти .NET (обзор); (6) `ConcurrentDictionary`, `ConcurrentQueue`, `ConcurrentBag`; (7) `Channel<T>`, producer/consumer; (8) `Parallel.For`/`ForEach`/`Invoke`, `Partitioner`; (9) Deadlocks/race conditions, диагностика; (10) `ThreadPool` настройки, `TaskCreationOptions.LongRunning` (Optional).
*Проект:* потокобезопасная очередь обработки задач с producer/consumer и метриками.

---

## Модуль M12: Работа с базами данных (EF Core) / Databases & EF Core

**ID:** M12
**Уровень / Level:** Professional
**Цель / Goal:** Моделировать предметную область, работать с БД через Entity Framework Core 8: code-first, миграции, отношения, LINQ-providers, tracking/no-tracking, транзакции, raw SQL, производительность запросов.
**Время / Time:** теория 7ч / практика 9ч / проект 5ч (итого 21ч)
**Сложность / Difficulty:** 3/5
**Prerequisites:** M08, M10
**Следующие модули / Next:** M13
**Статус / Status:** Core
**Домен / Domain:** Data
**Уроки / Lessons:** 10 уроков — (1) EF Core vs ADO.NET, провайдеры (SQL Server/SQLite/PostgreSQL); (2) Code-first, сущности, `DbContext`, подключение; (3) Миграции, `dotnet ef`, seeding; (4) Отношения: 1:N, N:N, 1:1, навигационные свойства; (5) LINQ to Entities, переводы в SQL; (6) `AsNoTracking`, проекции, `Include`/`ThenInclude`, N+1; (7) Транзакции, `SaveChanges`, `IDbContextTransaction`; (8) Raw SQL, `FromSqlRaw`, хранимые процедуры; (9) Concurrency tokens, optimistic concurrency; (10) Производительность: индексы, `AsSplitQuery`, batched updates.
*Проект:* доменная модель «Интернет-магазин» (клиенты, заказы, товары) с миграциями и LINQ-запросами.

---

## Модуль M13: Веб: ASP.NET Core основы / Web Basics

**ID:** M13
**Уровень / Level:** Professional
**Цель / Goal:** Понять устройство ASP.NET Core 8: хостинг, `Program.cs`, Middleware pipeline, маршрутизация, DI (встроенный контейнер), конфигурация, logging, статические файлы, основы MVC и Razor (обзор).
**Время / Time:** теория 6ч / практика 7ч / проект 4ч (итого 17ч)
**Сложность / Difficulty:** 3/5
**Prerequisites:** M11, M12
**Следующие модули / Next:** M14, M15, M16
**Статус / Status:** Core
**Домен / Domain:** Web
**Уроки / Lessons:** 9 уроков — (1) Архитектура ASP.NET Core, хост, `WebApplication`; (2) Middleware pipeline, `Use`/`Run`/`Map`, порядок; (3) Маршрутизация, endpoint routing; (4) Встроенный DI: `AddTransient/Scoped/Singleton`, lifetimes; (5) Конфигурация: `appsettings.json`, env vars, `IOptions`; (6) Logging, `ILogger`, провайдеры, structured logging; (7) Статические файлы, `wwwroot`; (8) MVC/Controllers (обзор) и Razor Pages (обзор); (9) Error handling middleware, environment-specific startup.
*Проект:* базовое веб-приложение с DI, конфигурацией по средам и логированием.

---

## Модуль M14: Веб: Web API, REST, Minimal API / Web API

**ID:** M14
**Уровень / Level:** Professional
**Цель / Goal:** Проектировать и реализовывать REST Web API на ASP.NET Core 8: Minimal API, контроллеры, валидация, версионирование, Swagger/OpenAPI, обработка ошибок, `HttpClient`-клиент.
**Время / Time:** теория 6ч / практика 8ч / проект 5ч (итого 19ч)
**Сложность / Difficulty:** 3/5
**Prerequisites:** M13
**Следующие модули / Next:** M15, M18, M19
**Статус / Status:** Core
**Домен / Domain:** Web
**Уроки / Lessons:** 10 уроков — (1) REST-принципы, ресурсы, методы, статус-коды; (2) Minimal API, `MapGet`/`MapPost`, groups; (3) Контроллеры, `[ApiController]`, attribute routing; (4) Model binding, `[FromBody]`/`[FromQuery]`/`[FromRoute]`; (5) Валидация, `DataAnnotations`, `FluentValidation` (Optional); (6) ProblemDetails, обработка ошибок API; (7) Версионирование API; (8) Swagger/OpenAPI, генерация документации; (9) CORS, rate limiting (обзор); (10) `HttpClient`, `IHttpClientFactory`, resilient HTTP-вызовы (вступление к Polly).
*Проект:* REST API для «Интернет-магазина» (CRUD, валидация, Swagger, EF Core).

---

## Модуль M15: Тестирование (xUnit, Moq) / Testing

**ID:** M15
**Уровень / Level:** Professional
**Цель / Goal:** Покрывать код тестами: unit-тесты (xUnit), моки (Moq), `Shouldly`/`FluentAssertions` (Optional), integration-тесты (`WebApplicationFactory`), TDD-практика, покрытие кода, тестируемость архитектуры.
**Время / Time:** теория 5ч / практика 7ч / проект 5ч (итого 17ч)
**Сложность / Difficulty:** 3/5
**Prerequisites:** M13, M14
**Следующие модули / Next:** M16, M19
**Статус / Status:** Core
**Домен / Domain:** Testing
**Уроки / Lessons:** 9 уроков — (1) Виды тестов (unit/integration/e2e), пирамида; (2) xUnit: факты, теории, `InlineData`/`MemberData`; (3) AAA, именование, `Assert`; (4) Moq: `Mock<T>`, `Setup`, `Verify`, `ItExpr`; (5) Тестирование DI, замена `DbContext` (in-memory, SQLite); (6) `WebApplicationFactory`, integration-тесты API; (7) TDD, red-green-refactor; (8) Покрытие кода (`coverlet`), отчёты; (9) Тестируемость как архитектурное свойство (вступление к M16).
*Проект:* полное покрытие (unit + integration) для API из M14.

---

## Модуль M16: Архитектура: паттерны, SOLID, DI / Architecture & Patterns

**ID:** M16
**Уровень / Level:** Professional
**Цель / Goal:** Проектировать тестируемую архитектуру: SOLID, DI-контейнер вглубь (Scrutor, декораторы), слои и clean architecture, паттерны (Repository, Unit of Work, Factory, Strategy, Decorator, Mediator/MediatorR, CQRS-обзор).
**Время / Time:** теория 7ч / практика 8ч / проект 5ч (итого 20ч)
**Сложность / Difficulty:** 4/5
**Prerequisites:** M05, M13, M15
**Следующие модули / Next:** M17, M18, M20
**Статус / Status:** Core
**Домен / Domain:** Architecture
**Уроки / Lessons:** 10 уроков — (1) SOLID: SRP, OCP, LSP, ISP, DIP с примерами; (2) DI-контейнер вглубь, lifetimes, антипаттерны (captive dependency); (3) Scrutor: assembly scan, decorators, adapters; (4) Слоистая архитектура, clean architecture, ports & adapters; (5) Repository + Unit of Work (и критика); (6) Factory, Abstract Factory, Builder; (7) Strategy, Decorator, Adapter; (8) MediatorR, CQRS (обзор); (9) Domain-Driven Design (обзор, Optional); (10) Anti-patterns: god object, anemic domain, service-locator.
*Проект (Professional capstone):* многослойное Web API (API → Application → Domain → Infrastructure) с DI, Repository, MediatR, тестами. Объединяет M11–M16.

---

## Модуль M17: Производительность и профилирование / Performance

**ID:** M17
**Уровень / Level:** Enterprise
**Цель / Goal:** Измерять и оптимизировать производительность .NET: профилирование CPU/memory, аллокации, GC (поколения, LOH, pinned), `Span<T>`/`Memory<T>`, benchmarking (`BenchmarkDotNet`), строки и pooled objects, контейнерная производительность.
**Время / Time:** теория 7ч / практика 8ч / проект 5ч (итого 20ч)
**Сложность / Difficulty:** 4/5
**Prerequisites:** M09, M11, M16
**Следующие модули / Next:** M20
**Статус / Status:** Pro
**Домен / Domain:** Performance
**Уроки / Lessons:** 9 уроков — (1) Методология: measure → optimize → measure, метрики (throughput, latency, p99); (2) `BenchmarkDotNet`, микро-бенчмарки; (3) Аллокации, boxing, `struct` vs `class`, `record struct`; (4) GC: поколения, LOH/POH/Small-Object-Heap, рабочие станции vs сервер, настройки; (5) `Span<T>`/`Memory<T>`, `stackalloc`, zero-copy; (6) Строки, `string.Create`, `ObjectPool<T>`, `ArrayPool<T>`; (7) Профилировщики: `dotnet-trace`, `dotnet-counters`, `dotnet-dump`, PerfView; (8) Async-производительность, `ValueTask`, горячие пути; (9) AOT (`PublishAot`) и trim-совместимость (Optional).
*Проект:* оптимизация горячего пути API с замерами «до/после» и профилем аллокаций.

---

## Модуль M18: Безопасность / Security

**ID:** M18
**Уровень / Level:** Enterprise
**Цель / Goal:** Проектировать безопасные приложения: аутентификация/авторизация (ASP.NET Core Identity, JWT, OAuth2/OpenID Connect), хранение секретов (Data Protection, Azure Key Vault, User Secrets), шифрование, защита веб-уязвимостей (OWASP Top 10), аудит.
**Время / Time:** теория 7ч / практика 7ч / проект 5ч (итого 19ч)
**Сложность / Difficulty:** 4/5
**Prerequisites:** M14, M16
**Следующие модули / Next:** M19, M20
**Статус / Status:** Pro
**Домен / Domain:** Security
**Уроки / Lessons:** 9 уроков — (1) Модель угроз, STRIDE, принципы (least privilege, defense in depth); (2) Аутентификация: ASP.NET Core Identity, cookies; (3) JWT, claims, `Authorization` policies/handlers; (4) OAuth2/OpenID Connect, внешние провайдеры; (5) Управление секретами: User Secrets, Data Protection API, Key Vault; (6) Симметричное/асимметричное шифрование, хэширование (`BCrypt`/`Argon2`), `RandomNumberGenerator`; (7) OWASP Top 10 в контексте ASP.NET Core (XSS, CSRF, SQLi, injection, insecure deserialization); (8) HTTPS, HSTS, security headers, CORS-политика; (9) Аудит, логирование security-событий, compliance (GDPR/обзор).
*Проект:* API с JWT-аутентификацией, ролевой авторизацией, защитой от CSRF/XSS и хранением секретов вне кода.

---

## Модуль M19: DevOps: CI/CD, Docker, деплой / DevOps & Deployment

**ID:** M19
**Уровень / Level:** Enterprise
**Цель / Goal:** Построить delivery-конвейер: контейнеризация (Docker, multi-stage builds), CI/CD (GitHub Actions/Azure DevOps), конфигурация окружений, стратегии деплоя, observability-базис (логи/метрики/health checks).
**Время / Time:** теория 6ч / практика 8ч / проект 6ч (итого 20ч)
**Сложность / Difficulty:** 3/5
**Prerequisites:** M14, M15, M18
**Следующие модули / Next:** M20
**Статус / Status:** Pro
**Домен / Domain:** DevOps
**Уроки / Lessons:** 9 уроков — (1) Образ жизни кода: Git-модель, trunk-based vs GitFlow, semver; (2) Docker-основы, `Dockerfile`, multi-stage build, образы .NET; (3) `docker compose`, локальная разработка с зависимостями (БД); (4) CI: build → test → pack → publish artifacts; (5) CD: deploy в тест/прод, окружения, секреты CI; (6) Стратегии деплоя: rolling, blue/green, canary (обзор); (7) Конфигурация: env vars, `appsettings.{env}.json`, feature flags; (8) Health checks, readiness/liveness, graceful shutdown; (9) Observability-базис: structured logging, метрики, distributed tracing (вступление к M20).
*Проект:* полностью автоматизированный конвейер (CI + CD) для API из M14/M16 с Docker-образом и health checks.

---

## Модуль M20: Микросервисы и enterprise-архитектура / Microservices & Enterprise

**ID:** M20
**Уровень / Level:** Enterprise
**Цель / Goal:** Проектировать распределённые системы: микросервисы, межсервисное взаимодействие (HTTP/gRPC, message brokers), resilience (Polly), observability (OpenTelemetry), распределённые транзакции (Saga, Outbox), trade-offs архитектуры.
**Время / Time:** теория 7ч / практика 8ч / проект 12ч (итого 27ч)
**Сложность / Difficulty:** 5/5
**Prerequisites:** M16, M17, M18, M19
**Следующие модули / Next:** — (финальный модуль / final module)
**Статус / Status:** Pro
**Домен / Domain:** Architecture
**Уроки / Lessons:** 10 уроков — (1) Монолит vs микросервисы, когда и зачем, trade-offs; (2) Декомпозиция по бизнес-возможностям, bounded contexts; (3) Межсервисное взаимодействие: HTTP, gRPC, message brokers (RabbitMQ/Kafka — обзор); (4) Resilience: Polly (retry, circuit breaker, timeout, bulkhead); (5) API Gateway, BFF, service discovery (обзор); (6) Observability: OpenTelemetry, tracing, metrics, logs в распределённой системе; (7) Распределённые данные: Saga, Outbox/CDC, eventual consistency; (8) Idempotency, messaging patterns; (9) Security в микросервисах (mTLS, token propagation); (10) Архитектурные решения и документирование (ADR), команда и эксплуатация.
*Проект (Enterprise capstone):* решение из 2–3 микросервисов с API Gateway, gRPC/HTTP, Polly-resilience, OpenTelemetry-observability, контейнерами и CI/CD. Объединяет M16–M20.

---

## Сводка по графу зависимостей / Dependency Graph Summary

Иллюстративный эскиз графа зависимостей (иллюстрация потока; полный список дуг — ниже в «Свойствах графа» и в сводной таблице):
Illustrative dependency sketch (flow illustration; the full edge list is in «Graph properties» and the summary table below):

```
M01 → M02 → M03 → M04 → M05 → M06 → M07 → M08 → M09 → M10
                     ↓                       ↓     ↓
                    M07                     M09   M10
                                                     ↓
                M09 → M11 → M13 → M14 → M15 → M16
                  ↑      ↑              ↓       ↓
                 M09    M12            M18     M17
                        ↑
                       M08,M10
                M16 → M17, M18 → M19 → M20
```

Топологический порядок (один из допустимых):

```
M01 → M02 → M03 → M04 → M05 → M06 → M07 → M08 → M09 → M10
→ M11 → M12 → M13 → M14 → M15 → M16 → M17 → M18 → M19 → M20
```

**Свойства графа / Graph properties:**
- Без циклов / acyclic (проверено; все prereq-дуги идут от меньшего ID к большему).
- Зеркальность / mirror: поле `Next` каждого модуля точно соответствует обратным дугам поля `Prerequisites` остальных (проверено попарно).
- Полный список prereq-дуг / full edge list:
  M01→M02; M02→M03,M04; M03→M04,M07; M04→M05; M05→M06,M16; M06→M07,M08;
  M07→M08,M09; M08→M10,M12; M09→M10,M11,M17; M10→M12; M11→M13,M17;
  M12→M13; M13→M14,M15,M16; M14→M15,M18,M19; M15→M16,M19; M16→M17,M18,M20;
  M17→M20; M18→M19,M20; M19→M20; M20→—.
- M01–M03 — линейное ядро Foundation (строго последовательно).
- Мосты: M06–M10 (Foundation → Professional), M15–M16 (Professional → Enterprise).
- M20 — единственный «сток» (финальный модуль, без Next); M01 — единственный «источник» (без пререквизитов).

> ASCII-эскиз выше носит иллюстративный характер; authoritative-граф — список дуг и сводная таблица ниже.
> The ASCII sketch above is illustrative; the authoritative graph is the edge list and the summary table below.

---

## Сводная таблица / Summary Table

| ID | Название / Title | Уровень | Домен | Статус | Сложн. | Часы | Prereq | Next |
|---|---|---|---|---|---|---|---|---|
| M01 | Intro | Foundation | Core | Core | 1 | 6 | — | M02 |
| M02 | Types & Operators | Foundation | Core | Core | 2 | 10 | M01 | M03, M04 |
| M03 | Control Flow & Methods | Foundation | Core | Core | 2 | 13 | M02 | M04, M07 |
| M04 | OOP Basics | Foundation | Core | Core | 3 | 16 | M02, M03 | M05 |
| M05 | OOP Advanced | Foundation | Core | Core | 3 | 16 | M04 | M06, M16 |
| M06 | Collections & Generics | Foundation | Core | Core | 3 | 14 | M05 | M07, M08 |
| M07 | Exception Handling | Foundation | Core | Core | 2 | 11 | M03, M06 | M08, M09 |
| M08 | LINQ | Foundation | Data | Core | 3 | 17 | M06, M07 | M10, M12 |
| M09 | Async Programming | Foundation | Concurrency | Core | 4 | 17 | M07 | M10, M11, M17 |
| M10 | I/O & Serialization | Foundation | Data | Core | 3 | 18 | M08, M09 | M12 |
| M11 | Multithreading | Professional | Concurrency | Core | 4 | 20 | M09 | M13, M17 |
| M12 | Databases & EF Core | Professional | Data | Core | 3 | 21 | M08, M10 | M13 |
| M13 | Web Basics | Professional | Web | Core | 3 | 17 | M11, M12 | M14, M15, M16 |
| M14 | Web API | Professional | Web | Core | 3 | 19 | M13 | M15, M18, M19 |
| M15 | Testing | Professional | Testing | Core | 3 | 17 | M13, M14 | M16, M19 |
| M16 | Architecture & Patterns | Professional | Architecture | Core | 4 | 20 | M05, M13, M15 | M17, M18, M20 |
| M17 | Performance | Enterprise | Performance | Pro | 4 | 20 | M09, M11, M16 | M20 |
| M18 | Security | Enterprise | Security | Pro | 4 | 19 | M14, M16 | M19, M20 |
| M19 | DevOps & Deployment | Enterprise | DevOps | Pro | 3 | 20 | M14, M15, M18 | M20 |
| M20 | Microservices & Enterprise | Enterprise | Architecture | Pro | 5 | 27 | M16, M17, M18, M19 | — |
| **Σ** | | | | | | **338** | | |

> Итого 338 ч ≈ 340 ч (теория 118 / практика 136 / проекты 84), согласовано с `01-course-passport.md`.

---

*Таблица модулей согласована с треками (`02-tracks.md`) и дорожной картой (`04-roadmap.md`).*
*The modules table is consistent with tracks (`02-tracks.md`) and the roadmap (`04-roadmap.md`).*
