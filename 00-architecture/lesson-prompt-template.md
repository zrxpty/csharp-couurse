# Промт-шаблон для генерации уроков курса по C# / Lesson Generation Prompt Template

> Reusable промт для создания двуязычных уроков (RU+EN) курса по C# 12+ / .NET 8+.
> Используется оркестратором для генерации контента через субагентов.
> Акцент на concurrency-темы (async/await, многопоточность, делегаты, каналы, примитивы синхронизации),
> но применим ко всем темам C#.

---

## Как использовать / How to use

Подставь переменные `{MODULE}`, `{LESSON_ID}`, `{TITLE_RU}`, `{TITLE_EN}`, `{TOPICS}`, `{LINKS}`,
`{PREV_LINK}`, `{NEXT_LINK}`, `{FILE_PATH}` в шаблон ниже и передай субагенту.

---

## Шаблон промта / Prompt template

```
Ты — Агент 2 «Контент-мейкер», Senior C# разработчик и технический писатель.
Создай ПОЛНЫЙ двуязычный (RU+EN) контент ОДНОГО урока курса по C# и ЗАПИШИ его в файл через tool write.

## Урок
- ID: {LESSON_ID}
- Название RU: {TITLE_RU}
- Title EN: {TITLE_EN}
- Модуль: {MODULE}
- Темы для раскрытия: {TOPICS}
- Ресурсы-источники: {LINKS}

## Навигация (вставь в начало файла)
- Предыдущий: {PREV_LINK}
- Следующий: {NEXT_LINK}

## ДЕЙСТВИЕ
Используй tool write, чтобы записать в файл:
{FILE_PATH}

полное содержимое урока в формате ниже.

## Формат содержимого файла (markdown, двуязычно RU+EN)

[← Предыдущий / Previous]({PREV_LINK}) | [Следующий / Next →]({NEXT_LINK})

---

### Урок {LESSON_ID}: {TITLE_RU} / {TITLE_EN}

**Модуль / Module:** {MODULE}
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory
[300–700 слов на РУССКОМ. Понятно, с аналогиями. Покрой все темы выше.]

#### Theory (EN)
[300–700 words in ENGLISH covering the same topics. Clear, with analogies.]

#### Пример кода / Code Example
​```csharp
// рабочий код C# 12+ / .NET 8+, комментарии на RU и EN
​```

#### Best Practices
- [RU рекомендация]
- [EN recommendation]

#### Частые ошибки / Common Mistakes
- [ошибка] → [как избежать] (RU)
- [mistake] → [how to avoid] (EN)

#### Чек-лист самопроверки / Self-check Checklist
- [ ] пункт (RU)
- [ ] item (EN)

#### Ресурсы / Resources
- [Microsoft Learn — {LINKS} — название RU/EN]

---

[← Предыдущий / Previous]({PREV_LINK}) | [Следующий / Next →]({NEXT_LINK})

## Требования к качеству
- Код рабочий для C# 12 / .NET 8 (top-level statements, pattern matching, raw strings где уместно).
- Обязательны оба блока: Theory (RU) И Theory (EN) — оба полные (300–700 слов каждый).
- Никаких заглушек «TODO».
- Для concurrency-уроков (async, потоки, делегаты, каналы, синхронизация):
  показывай РАБОЧИЙ код, объясняй контекст синхронизации, предупреждай о дедлоках/race conditions.
- ЗАПИШИ файл через tool write, затем верни ОДНО предложение: путь к файлу и подтверждение записи.

Верни только подтверждение записи, не дублируй содержимое.
```

---

## Список всех тем курса (полный охват C#)

### Foundation (M01–M10)
- **M01** Введение в C# и .NET: CLR, IL, JIT, SDK, CLI, top-level statements, dotnet CLI
- **M02** Типы, переменные, операторы: value/ref types, примитивы, var/const, операторы, nullable, преобразования, string/StringBuilder, enum/кортежи
- **M03** Управление потоком, методы: if/switch/pattern matching, циклы, методы, ref/out/in/params, перегрузка, рекурсия
- **M04** ООП основы: классы, поля/свойства (init/required), конструкторы, инкапсуляция, static/readonly/const, this, индексаторы, record, struct vs class, namespace
- **M05** ООП наследование/полиморфизм: base, virtual/override/abstract, интерфейсы, sealed, is/as, record+inheritance, composition vs inheritance
- **M06** Коллекции и дженерики: List<T>, обобщённые классы/методы, where-ограничения, Dictionary/HashSet, Queue/Stack, IEnumerable/IEnumerator, yield, Span<T>
- **M07** Исключения: иерархия Exception, try/catch/finally, throw vs rethrow, кастомные исключения, when-фильтры, Result<T>
- **M08** LINQ: method/query syntax, Where/Select/OrderBy/GroupBy/Join, агрегаты, отложенное выполнение, IQueryable vs IEnumerable
- **M09** Асинхронное программирование ⭐CONCURRENCY: Task/Task<T>, async/await, state machine, Task.Run, WhenAll/WhenAny, CancellationToken, ConfigureAwait, ValueTask, антипаттерны, async streams
- **M10** Файлы/потоки/сериализация: File/Directory/Path, Stream/FileStream, StreamReader/Writer, async I/O, System.Text.Json, MemoryStream/Pipes, XML, binary

### Professional (M11–M16)
- **M11** Многопоточное программирование ⭐CONCURRENCY: Thread/ThreadPool, lock/Monitor, Interlocked, SemaphoreSlim/Mutex/ReaderWriterLockSlim, volatile/барьеры, ConcurrentDictionary/Queue/Bag, Channel<T>, Parallel.For, deadlocks, TaskCreationOptions
- **M12** БД (EF Core): code-first, миграции, отношения, LINQ to Entities, AsNoTracking, Include, транзакции, raw SQL, concurrency tokens, производительность
- **M13** ASP.NET Core основы: хост, middleware, маршрутизация, DI, конфигурация, logging, статические файлы, MVC/Razor
- **M14** Web API/REST: Minimal API, контроллеры, model binding, валидация, ProblemDetails, версионирование, Swagger, CORS, HttpClient/IHttpClientFactory
- **M15** Тестирование: xUnit, Moq, WebApplicationFactory, TDD, покрытие, тестируемость
- **M16** Архитектура: SOLID, DI вглубь, Scrutor, clean architecture, Repository/UoW, Factory/Strategy/Decorator, MediatR/CQRS, DDD, антипаттерны

### Enterprise (M17–M20)
- **M17** Производительность: BenchmarkDotNet, аллокации/GC, Span/Memory, ObjectPool/ArrayPool, профилировщики, async-перф, AOT
- **M18** Безопасность: Identity, JWT, OAuth2/OIDC, секреты, шифрование/хэширование, OWASP Top 10, security headers, аудит
- **M19** DevOps: Docker/multi-stage, CI/CD, окружения, стратегии деплоя, health checks, observability
- **M20** Микросервисы: декомпозиция, HTTP/gRPC/brokers, Polly-resilience, API Gateway, OpenTelemetry, Saga/Outbox, idempotency, mTLS

### ⭐ Concurrency-темы (особый акцент)
Делегаты/события, async/await, потоки, каналы (Channel<T>), примитивы синхронизации (lock, Monitor,
SemaphoreSlim, Mutex, ReaderWriterLockSlim, Interlocked, CountdownEvent, Barrier, ManualResetEventSlim),
volatile/модель памяти, concurrent-коллекции, producer/consumer, async streams.

---

## Формат имени файла
`lesson-{id}-{тема-slug}.md`

Примеры:
- `lesson-M09-L01-async-intro.md`
- `lesson-M11-L02-lock-monitor.md`
- `lesson-M11-L07-channels.md`
- `lesson-M04-L02-properties.md`

Тема-slug — короткое ключевое слово на латинице через дефис (async-await, lock-monitor, channels,
delegates, properties, linq-basics и т.д.).
