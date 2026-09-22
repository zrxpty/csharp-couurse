---
[← К уроку M14-L02](lesson-M14-L02-minimal-api.md) | [⬆ К модулю M14](../README.md) | [Следующее ДЗ →](homework-M14-L03-controllers-apicontroller.md)
---

### Домашнее задание M14-L02: Minimal API, MapGet/MapPost, groups / Homework M14-L02: Minimal API, MapGet/MapPost, groups

**Урок / Lesson:** M14-L02
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 3/5
**Цель / Goal:** (RU) Научиться строить полноценный REST API на Minimal API .NET 8: использовать `MapGet`/`MapPost`/`MapPut`/`MapDelete`, группировать эндпоинты через `MapGroup`, возвращать типизированные результаты через `TypedResults` и union-тип `Results<>`, применять `AddEndpointFilter` для сквозной логики и выносить регистрацию маршрутов в статические методы расширения. (EN) Learn to build a complete REST API on .NET 8 Minimal API: use `MapGet`/`MapPost`/`MapPut`/`MapDelete`, group endpoints with `MapGroup`, return typed results via `TypedResults` and the `Results<>` union, apply `AddEndpointFilter` for cross-cutting logic, and extract route registration into static extension methods.

#### Связь с уроком / Connection to the lesson
(RU) Урок вводит Minimal API как современную альтернативу контроллерам MVC, показывает отличие `Results` от `TypedResults`, объясняет route groups и endpoint filters. Это ДЗ закрепляет все эти темы на сквозном примере «мини-Todo API»: вы пройдёте от пустого проекта до типизированных эндпоинтов с группой, фильтром валидации и чистой регистрацией через метод расширения.
(EN) The lesson introduces Minimal API as a modern alternative to MVC controllers, contrasts `Results` with `TypedResults`, and explains route groups and endpoint filters. This homework cements all of those topics on a single “mini-Todo API”: you go from an empty project to typed endpoints with a group, a validation filter, and clean registration via an extension method.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация
Представьте, что вы — единственный backend-разработчик стартапа «Кофейные задачи», который делает простой сервис управления задачами бариста в кофейне. Команды небольшие, бюджет ограничен, а бизнес хочет получать фичи быстро. Писать тяжёлый контроллер с `[ApiController]`, модель-биндингом и форматтерами — избыточно: вам нужны пять-шесть HTTP-эндпоинтов и максимум скорости разработки. Minimal API идеально подходит: вы описываете маршруты прямо на `WebApplication`, параметры обработчика связываются с запросом автоматически, а возвращать можно типизированные `TypedResults` для корректной OpenAPI-схемы.

В уроке вы увидели ключевые идеи: `MapGroup` для общего префикса и общих политик, `AddEndpointFilter` для валидации и логирования, констрейнты маршрута вроде `{id:int}`, принцип «сервисы через параметры, а не через замыкания», и best practice — выносить регистрацию группы в статический метод расширения. В этом задании вы соберёте всё вместе. Важно не просто запустить код, а понять, почему именно так: почему `TypedResults` лучше `Results`, почему union-тип `Results<Ok<T>, NotFound>` даёт компилятору и Swagger точную схему, почему фильтр лучше дублирования проверки в каждом обработчике, и почему `null` вместо `NotFound()` ломает контракт API.

К концу задания у вас будет работающий `dotnet run`, отвечающий на `GET /api/todos`, `GET /api/todos/{id}`, `POST /api/todos`, `PUT /api/todos/{id}`, `DELETE /api/todos/{id}`, с группой, тегом, фильтром валидации и типизированными ответами. Вы сможете объяснить каждую строку и защитить выбор архитектуры.

#### Что нужно сделать (пошагово)
1. Создайте новый пустой веб-проект через `dotnet new web -n CoffeeTodos.Api -o CoffeeTodos.Api`. Это даст минимальный `Program.cs` с `WebApplication.CreateBuilder`. Убедитесь, что `.csproj` ссылается на `net8.0` и что включён `<ImplicitUsings>enable</ImplicitUsings>` — иначе придётся добавлять `using Microsoft.AspNetCore.Builder;` вручную.
2. Перейдите в каталог проекта `cd CoffeeTodos.Api` и убедитесь, что `dotnet run` стартует: в консоли должен появиться адрес `http://localhost:5000` (или `5001`/`5002` — зависит от `launchSettings.json`). Остановите сервер `Ctrl+C`.
3. Добавьте модель `record Todo(int Id, string Title, bool Done, DateTime CreatedAt);`. Используйте именно `record` — он неизменяемый, что упрощает обновление через `with`. Поле `CreatedAt` будет заполняться сервером, а не клиентом.
4. Создайте класс `TodoStore` — простой потокобезопасный «репозиторий» в памяти на `ConcurrentDictionary<int, Todo>` с инкрементальным счётчиком через `Interlocked.Increment`. Реализуйте методы `GetAll()`, `FindAsync(int id)`, `Add(Todo)`, `Update(int id, Todo)`, `Remove(int id)`. Почему `ConcurrentDictionary`, а не `Dictionary`? В Minimal API обработчики запускаются concurrently, а `Dictionary` не потокобезопасен на запись — вы получите случайные гонки в проде.
5. Зарегистрируйте `TodoStore` в DI как синглтон: `builder.Services.AddSingleton<TodoStore>();`. Синглтон уместен, потому что хранилище in-memory и общее для всех запросов. Никогда не регистрируйте контекст EF как синглтон — это типичная ошибка, но для чистого словаря она безопасна.
6. Создайте статический класс `TodoEndpoints` с методом расширения `public static IEndpointRouteBuilder MapTodoEndpoints(this IEndpointRouteBuilder app)`. Внутри объявите `var group = app.MapGroup("/api/todos").WithTags("Todos");`. Вынесение в метод расширения — это best practice из урока: `Program.cs` остаётся чистым, а регистрацию можно переиспользовать в тестах.
7. Внутри группы опишите эндпоинты: `MapGet("/", ...)` вернёт `TypedResults.Ok(store.GetAll())`; `MapGet("/{id:int}", ...)` вернёт `Results<Ok<Todo>, NotFound>` через union-тип; `MapPost("/", ...)` создаст задачу и вернёт `TypedResults.Created(...)`; `MapPut("/{id:int}", ...)` вернёт `Results<NoContent, NotFound>`; `MapDelete("/{id:int}", ...)` вернёт `TypedResults.NoContent()`. Обратите внимание: `MapGet("/{id:int}")` обязателен констрейнт `:int`, иначе в обработчик может прийти строка.
8. На `MapPost` добавьте `AddEndpointFilter`, который проверяет, что `Title` не пустой, и при нарушении возвращает `TypedResults.BadRequest("Title is required / Заголовок обязателен")`. Фильтр должен брать `context.Arguments[0]` — первый параметр обработчика, приводить к `Todo` и проверять `string.IsNullOrWhiteSpace(arg.Title)`. Это пример сквозной логики из урока: валидация живёт в одном месте, а не дублируется в каждом обработчике.
9. В `Program.cs` вызовите `app.MapTodoEndpoints();` после `var app = builder.Build();`. Запустите `dotnet run` и проверьте эндпоинты через `curl` или REST-клиент: `curl http://localhost:5000/api/todos` должен вернуть `[]`; `curl -X POST -H "Content-Type: application/json" -d '{"title":"Latte","done":false}' http://localhost:5000/api/todos` должен вернуть `201 Created` с телом и заголовком `Location`.
10. (Опционально, но рекомендуется) Добавьте Swagger через `builder.Services.AddEndpointsApiExplorer();` и `builder.Services.AddSwaggerGen();`, а в pipeline — `app.UseSwagger(); app.UseSwaggerUI();`. Откройте `/swagger` и убедитесь, что эндпоинты сгруппированы под тегом «Todos», а схемы ответов для `GET /{id}` показывают и `200 OK` с `Todo`, и `404 Not Found` — это доказывает, что union-тип `Results<>` действительно даёт корректную OpenAPI-схему.
11. Проверьте негативные сценарии: `POST` с пустым `title` должен вернуть `400 BadRequest` от фильтра; `GET /api/todos/abc` должен вернуть `404` (или `400` — зависит от поведения маршрутизации с констрейнтом `:int`, ключевой момент в том, что обработчик не вызывается для не-`int`); `GET /api/todos/9999` должен вернуть `404 NotFound` с пустым телом.
12. Закоммитьте решение в git: `git init && git add . && git commit -m "M14-L02 minimal api todo"`.

#### Требования к решению
- Проект — `dotnet new web`, целевая сборка `net8.0`, C# 12 (можно использовать top-level statements, `record`, `with`, pattern matching, collection expressions, raw string literals там, где это улучшает читаемость).
- Минимум пять эндпоинтов на одной группе `/api/todos`: `GET /`, `GET /{id:int}`, `POST /`, `PUT /{id:int}`, `DELETE /{id:int}`. Все эндпоинты должны быть внутри группы, а не на корне `app`.
- Возвращаемые значения: минимум три эндпоинта используют `TypedResults` напрямую, минимум один — union-тип `Results<Ok<Todo>, NotFound>` (для `GET /{id}`) и минимум один — `Results<NoContent, NotFound>` (для `PUT /{id}`). Это демонстрирует понимание разницы между `Results` и `TypedResults` и умение строить union-типы.
- На `POST` должен быть `AddEndpointFilter`, проверяющий `Title` и возвращающий `TypedResults.BadRequest` при пустом заголовке. Фильтр должен корректно приводить `context.Arguments[0]` к `Todo` и обрабатывать `null` (через `!` или `?? new Todo(0,"",false,DateTime.UtcNow)`).
- `TodoStore` должен быть зарегистрирован в DI и получаться через параметры обработчика, а не через замыкание. Запрещено писать `app.MapGet("/", () => _store.GetAll())` — это антипаттерн из урока.
- Регистрация группы вынесена в статический метод расширения `MapTodoEndpoints(this IEndpointRouteBuilder)`, `Program.cs` содержит только `builder`/`app`/`app.MapTodoEndpoints()`/`app.Run()`.
- Констрейнты маршрута `{id:int}` на всех эндпоинтах с `id`. Никаких «голых» `{id}`.
- `CreatedAt` заполняется сервером в `Add`, а не берётся из тела клиента (защита от подмены).
- Код компилируется без warnings уровня `error`, `dotnet run` стартует, эндпоинты отвечают ожидаемыми статус-кодами.

#### Тонкости и подводные камни
- **`Results` vs `TypedResults`**: если объявили сигнатуру `Results<Ok<Todo>, NotFound>`, внутри обработчика используйте именно `TypedResults.Ok`/`TypedResults.NotFound`. Если смешать `Results.Ok` (нетипизированный) с типизированной сигнатурой — компилятор это пропустит (через неявное преобразование), но теряется смысл типизации и Swagger может показать `200 OK` без схемы. Это самая частая ошибка урока.
- **`null` вместо `NotFound`**: вернуть `null` из обработчика, который объявлен как `Todo` (а не `Results<...>`), даст клиенту `200 OK` с пустым телом. Бизнес будет думать, что объект существует, но пуст. Всегда возвращайте явный `IResult` со статус-кодом. Если обработчик теоретически может не найти объект — сигнатура должна быть `Results<Ok<T>, NotFound>`, а не просто `T`.
- **Констрейнты маршрута**: `{id}` без `:int` означает, что в обработчик `int id` при попытке связать строку `"abc"` маршрутизация не вызовет обработчик вообще — будет `404` или `400`. Но если сигнатура `string id`, то строка пройдёт и сломает бизнес-логику. Всегда указывайте `:int`, `:guid`, `:long` там, где это уместно — это первая линия защиты.
- **Захват сервиса в замыкание**: `app.MapGet("/", () => _store.GetAll())` работает, но (а) ломает тестирование (нельзя подменить `_store`), (б) захватывает конкретный экземпляр, а не сервис из текущего scope, (в) для scoped-сервисов вроде EF `DbContext` даёт captive-dependency баг. Передавайте сервис как параметр обработчика: `(TodoStore store) => store.GetAll()`.
- **Фильтр и `context.Arguments`**: индексы в `Arguments` соответствуют порядку параметров обработчика, **исключая `HttpContext` и DI-сервисы**? Точнее: `Arguments` содержит параметры в порядке объявления, включая сервисы. Для `MapPost("/", (Todo todo, TodoStore store) => ...)` `Arguments[0]` — `Todo`, `Arguments[1]` — `TodoStore`. Проверяйте `null` перед приведением — фильтр может запуститься до биндинга в некоторых конфигурациях.
- **`WithTags` на группе, а не на эндпоинте**: если навесить `WithTags("Todos")` только на один эндпоинт группы, остальные свалятся в Swagger без категории. Навешивайте тег на всю группу — он применится ко всем эндпоинтам.
- **`RequireAuthorization` на группе**: если вся группа защищена, не дублируйте `RequireAuthorization` на отдельном эндпоинте — это путаница в политиках. Но если один эндпоинт должен быть анонимным (`GET /public`), выносите его в отдельную группу или используйте `AllowAnonymous()`.
- **`MapGet("/")` vs `MapGet("")`**: на группе оба дадут маршрут с префиксом, но `/` каноничнее и даёт `Location`-заголовок без двойных слешей. Для `POST` с `Created($"/api/todos/{id}", ...)` убедитесь, что префикс совпадает с группой, иначе клиент получит несуществующий `Location`.

#### Критерии приёмки
- [ ] Проект создан через `dotnet new web`, целевая сборка `net8.0`, код компилируется без ошибок.
- [ ] Есть модель `record Todo` минимум с полями `Id`, `Title`, `Done`, `CreatedAt`.
- [ ] `TodoStore` реализован на `ConcurrentDictionary` (или эквивалент с блокировкой) и потокобезопасен.
- [ ] `TodoStore` зарегистрирован в DI как синглтон и нигде не создаётся через `new` в обработчиках.
- [ ] Минимум пять эндпоинтов (`GET /`, `GET /{id:int}`, `POST /`, `PUT /{id:int}`, `DELETE /{id:int}`) внутри одной группы `/api/todos`.
- [ ] Группа имеет `WithTags("Todos")` — в Swagger эндпоинты сгруппированы под одной категорией.
- [ ] `GET /{id:int}` возвращает `Results<Ok<Todo>, NotFound>` (union-тип) и использует `TypedResults.NotFound()` при отсутствии.
- [ ] `PUT /{id:int}` возвращает `Results<NoContent, NotFound>` и использует `TypedResults.NoContent()`/`TypedResults.NotFound()`.
- [ ] `POST /` возвращает `TypedResults.Created($"/api/todos/{id}", created)` с корректным заголовком `Location`.
- [ ] На `POST` навешен `AddEndpointFilter`, проверяющий `Title` и возвращающий `TypedResults.BadRequest` при пустом заголовке.
- [ ] Сервис `TodoStore` передаётся как параметр обработчика во всех эндпоинтах, нет захвата через замыкание.
- [ ] Регистрация группы вынесена в статический метод расширения `MapTodoEndpoints`, `Program.cs` чистый.
- [ ] Констрейнты `{id:int}` на всех эндпоинтах с `id`.
- [ ] `CreatedAt` заполняется сервером в `Add`, а не берётся из тела клиента.
- [ ] `curl`/Swagger подтверждают: пустой `POST` → `400`, `GET /{id:int}` несуществующего → `404`, успешный `POST` → `201` с `Location`.
- [ ] В коде нет `null`-возвратов вместо `NotFound` в эндпоинтах, где возможен «не найден».

#### Подсказки (без прямого ответа)
- Для счётчика используйте `Interlocked.Increment(ref _seq)` — обычный `_seq++` не потокобезопасен.
- В фильтре помните, что `Arguments` — это `IList<object?>`, и порядок совпадает с порядком параметров обработчика.
- Для union-типа `Results<Ok<T>, NotFound>` тип `T` должен быть ссылочным или `nullable` — компилятор подскажет, если `T` — значимый тип.
- Swagger требует `AddEndpointsApiExplorer()` именно для Minimal API — для контроллеров он не нужен, потому что MVC сам регистрирует explorer.
- Чтобы `Created` вернул корректный `Location`, передавайте абсолютный или корневой путь, а не относительный.

#### Эталонное решение (разбор)
```csharp
// C# 12 / .NET 8 — Эталонное решение / Reference solution
using System.Collections.Concurrent;
using Microsoft.AspNetCore.Http.HttpResults;

var builder = WebApplication.CreateBuilder(args);

// Регистрируем хранилище в DI как синглтон / Register the store as a singleton
builder.Services.AddSingleton<TodoStore>();

// Swagger для Minimal API — нужен EndpointsApiExplorer / Minimal API Swagger needs explorer
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

// Регистрируем группу эндпоинтов через метод расширения / Register group via extension method
app.MapTodoEndpoints();

app.Run();

// --- Статический класс регистрации эндпоинтов / Static endpoint registration ---
public static class TodoEndpoints
{
    // Метод расширения вынесен, чтобы Program.cs оставался чистым (best practice из урока)
    // Extension method keeps Program.cs clean (lesson best practice)
    public static IEndpointRouteBuilder MapTodoEndpoints(this IEndpointRouteBuilder app)
    {
        // Группа с общим префиксом и тегом для Swagger / Group with shared prefix and Swagger tag
        var group = app.MapGroup("/api/todos")
                       .WithTags("Todos");

        // GET /api/todos — все задачи / all todos
        group.MapGet("/", (TodoStore store) =>
        {
            return TypedResults.Ok(store.GetAll());
        });

        // GET /api/todos/{id} — union-тип Results<> даёт корректную OpenAPI-схему
        // Union type Results<> yields a correct OpenAPI schema
        group.MapGet("/{id:int}", async Task<Results<Ok<Todo>, NotFound>> (int id, TodoStore store) =>
        {
            var todo = await store.FindAsync(id);
            return todo is null
                ? TypedResults.NotFound()   // явно NotFound, НЕ null / explicit NotFound, NOT null
                : TypedResults.Ok(todo);
        });

        // POST /api/todos — создать задачу / create a todo
        // Сервис TodoStore передаётся параметром, не замыканием (антипаттерн урока)
        // TodoStore is passed as a parameter, not captured (lesson anti-pattern avoided)
        group.MapPost("/", (Todo todo, TodoStore store) =>
        {
            var created = store.Add(todo);
            return TypedResults.Created($"/api/todos/{created.Id}", created);
        })
        .AddEndpointFilter(async (context, next) =>
        {
            // Первый параметр обработчика — Todo / first handler argument is Todo
            var arg = (Todo)context.Arguments[0]!;
            if (string.IsNullOrWhiteSpace(arg.Title))
            {
                // Прерываем цепочку BadRequest-ом / short-circuit with BadRequest
                return TypedResults.BadRequest("Title is required / Заголовок обязателен");
            }
            return await next(context); // передаём управление обработчику / pass to handler
        });

        // PUT /api/todos/{id} — обновить / update
        group.MapPut("/{id:int}", (int id, Todo todo, TodoStore store) =>
        {
            return store.Update(id, todo)
                ? Results.NoContent()
                : Results.NotFound();
        });

        // DELETE /api/todos/{id} — удалить / delete
        group.MapDelete("/{id:int}", (int id, TodoStore store) =>
        {
            store.Remove(id);
            return TypedResults.NoContent();
        });

        return app;
    }
}

// --- Модель и хранилище / Model and store ---

public record Todo(int Id, string Title, bool Done, DateTime CreatedAt);

public sealed class TodoStore
{
    // ConcurrentDictionary для потокобезопасности / thread-safe storage
    private readonly ConcurrentDictionary<int, Todo> _items = new();
    private int _seq = 0;

    public IReadOnlyCollection<Todo> GetAll() => _items.Values.ToArray();

    public Task<Todo?> FindAsync(int id) =>
        Task.FromResult(_items.TryGetValue(id, out var t) ? t : null);

    public Todo Add(Todo todo)
    {
        // Interlocked.Increment — потокобезопасный счётчик / thread-safe counter
        var id = Interlocked.Increment(ref _seq);
        // CreatedAt задаёт сервер, а не клиент (защита от подмены)
        // CreatedAt is set by the server, not the client (tamper protection)
        var created = todo with { Id = id, CreatedAt = DateTime.UtcNow };
        _items[id] = created;
        return created;
    }

    public bool Update(int id, Todo todo)
    {
        // Сохраняем оригинальный Id и CreatedAt — клиент не должен их менять
        // Keep original Id and CreatedAt — client must not change them
        if (!_items.TryGetValue(id, out var existing)) return false;
        _items[id] = todo with { Id = id, CreatedAt = existing.CreatedAt };
        return true;
    }

    public void Remove(int id) => _items.TryRemove(id, out _);
}
```

Разбор по строкам. `WebApplication.CreateBuilder(args)` — стандартная точка входа Minimal API; `builder.Services.AddSingleton<TodoStore>()` регистрирует хранилище в DI как синглтон (уместно для in-memory словаря, но **не** для `DbContext` — тот scoped). `AddEndpointsApiExplorer()` нужен именно Minimal API, потому что для контроллеров MVC регистрирует explorer сам; без него `AddSwaggerGen()` не увидит ваши эндпоинты. `app.MapTodoEndpoints()` — вызов метода расширения; так `Program.cs` остаётся чистым, а регистрацию можно переиспользовать в интеграционных тестах, передавая `IEndpointRouteBuilder` из `WebApplicationFactory`.

Внутри `MapTodoEndpoints` создаётся `MapGroup("/api/todos").WithTags("Todos")` — это даёт общий префикс и тег для Swagger. Констрейнт `{id:int}` на всех эндпоинтах с `id` защищает обработчик от строк вроде `"abc"`. Сигнатура `Task<Results<Ok<Todo>, NotFound>>` — union-тип: компилятор гарантирует, что обработчик вернёт один из двух вариантов, а OpenAPI-генератор строит схему с обоими статусами (`200` и `404`). Внутри используется `TypedResults.NotFound()` (а не `Results.NotFound()`) — это сохраняет типизацию. Проверка `todo is null` через pattern matching — идиоматичный C# 12.

`MapPost` с `AddEndpointFilter` демонстрирует сквозную логику: фильтр берёт `context.Arguments[0]` (первый параметр `Todo todo`), проверяет `Title` и при пустом заголовке возвращает `TypedResults.BadRequest`, прерывая цепочку (`next` не вызывается). Если валидация проходит, `await next(context)` передаёт управление обработчику. Важно: сервис `TodoStore` — параметр обработчика, а не захваченное поле замыкания — это best practice из урока, позволяет тестировать и избегает captive-dependency для scoped-сервисов. `TypedResults.Created($"/api/todos/{created.Id}", created)` выставляет статус `201` и заголовок `Location`, чтобы клиент мог сразу `GET` новый ресурс.

В `TodoStore` `ConcurrentDictionary` гарантирует потокобезопасность, `Interlocked.Increment` — атомарный счётчик (обычный `++` в concurrent-сценарии даёт гонки). `Add` через `with` создаёт новый `record` с серверным `Id` и `CreatedAt` — клиент не может их подменить, потому что они перезаписываются. `Update` сохраняет оригинальный `Id` и `CreatedAt` — иначе клиент мог бы «переместить» задачу на другой `Id` или подменить дату создания. `Remove` через `TryRemove` безопасен, даже если ключа нет.

Какие концепции урока применены: `MapGroup` + `WithTags` (группировка), `TypedResults` + `Results<>` (типизация и OpenAPI), `AddEndpointFilter` (сквозная логика), констрейнты маршрута (`{id:int}`), сервисы через параметры (не замыкания), регистрация через статический метод расширения (чистый `Program.cs`), явные `NotFound`/`NoContent` (а не `null`). Все частые ошибки из урока в эталонном решении отсутствуют намеренно — на это и стоит опираться при самопроверке.

#### Задания на углубление (бонус)
1. Добавьте пагинацию: `GET /api/todos?page=1&pageSize=10` с параметрами через `AsParameters` или отдельные `int page, int pageSize` и возвратом `Results<Ok<PagedResult<Todo>>, BadRequest>`. Реализуйте `PagedResult<T>` record с `Items`, `Total`, `Page`, `PageSize`.
2. Добавьте endpoint filter для логирования: до `next` логируйте `context.HttpContext.Request.Method` и путь, после — статус-код и длительность через `Stopwatch`. Убедитесь, что фильтр применяется ко всей группе.
3. Реализуйте второй фильтр — exception handler: оборачивает `await next(context)` в `try/catch`, логирует исключение и возвращает `TypedResults.Problem(...)`. Проверьте, что `GET /{id}` при выбросе `InvalidOperationException` в `TodoStore` возвращает `500` с JSON-описанием проблемы, а не «крашит» процесс.
4. Разделите эндпоинты на две вложенные группы: `/api/todos` (чтение, без авторизации) и `/api/todos/admin` (запись, с `RequireAuthorization("admin")`). Продемонстрируйте, что `RequireAuthorization` на вложенной группе не наследуется на родительскую, и наоборот.

---

## Statement in English / Постановка на английском

#### Context & motivation
Imagine you are the only backend developer at “Coffee Tasks”, a startup building a simple todo service for café baristas. The team is small, the budget is tight, and the business wants features fast. Writing a heavy controller decorated with `[ApiController]`, full model binding, and custom formatters is overkill: you need five or six HTTP endpoints and maximum development speed. Minimal API fits perfectly: you describe routes directly on `WebApplication`, handler parameters bind from the request automatically, and you can return typed `TypedResults` for a correct OpenAPI schema.

In the lesson you saw the key ideas: `MapGroup` for a shared prefix and shared policies, `AddEndpointFilter` for validation and logging, route constraints like `{id:int}`, the “services as parameters, not as closures” principle, and the best practice of extracting group registration into a static extension method. In this assignment you will put all of it together. The point is not merely to run the code but to understand why: why `TypedResults` is better than `Results`, why the `Results<Ok<T>, NotFound>` union gives the compiler and Swagger a precise schema, why a filter beats duplicating the check in every handler, and why returning `null` instead of `NotFound()` breaks the API contract.

By the end you will have a working `dotnet run` answering `GET /api/todos`, `GET /api/todos/{id}`, `POST /api/todos`, `PUT /api/todos/{id}`, `DELETE /api/todos/{id}`, with a group, a tag, a validation filter, and typed responses. You will be able to explain every line and defend the architectural choices.

#### What to do step by step
1. Create a new empty web project with `dotnet new web -n CoffeeTodos.Api -o CoffeeTodos.Api`. This gives you a minimal `Program.cs` with `WebApplication.CreateBuilder`. Confirm the `.csproj` targets `net8.0` and has `<ImplicitUsings>enable</ImplicitUsings>` — otherwise you will have to add `using Microsoft.AspNetCore.Builder;` by hand.
2. `cd CoffeeTodos.Api` and verify `dotnet run` starts: the console should print `http://localhost:5000` (or `5001`/`5002` depending on `launchSettings.json`). Stop the server with `Ctrl+C`.
3. Add the model `record Todo(int Id, string Title, bool Done, DateTime CreatedAt);`. Use a `record` — it is immutable, which makes updates via `with` trivial. `CreatedAt` is filled by the server, not by the client.
4. Create a `TodoStore` class — a simple thread-safe in-memory “repository” on `ConcurrentDictionary<int, Todo>` with an incremental counter via `Interlocked.Increment`. Implement `GetAll()`, `FindAsync(int id)`, `Add(Todo)`, `Update(int id, Todo)`, `Remove(int id)`. Why `ConcurrentDictionary` instead of `Dictionary`? In Minimal API handlers run concurrently and `Dictionary` is not thread-safe for writes — you will get random races in production.
5. Register `TodoStore` in DI as a singleton: `builder.Services.AddSingleton<TodoStore>();`. A singleton is fine because the store is in-memory and shared across all requests. Never register an EF `DbContext` as a singleton — that is a classic mistake — but for a plain dictionary it is safe.
6. Create a static class `TodoEndpoints` with an extension method `public static IEndpointRouteBuilder MapTodoEndpoints(this IEndpointRouteBuilder app)`. Inside, declare `var group = app.MapGroup("/api/todos").WithTags("Todos");`. Extracting to an extension method is the lesson’s best practice: `Program.cs` stays clean and registration can be reused in tests.
7. Inside the group describe the endpoints: `MapGet("/", ...)` returns `TypedResults.Ok(store.GetAll())`; `MapGet("/{id:int}", ...)` returns `Results<Ok<Todo>, NotFound>` via the union type; `MapPost("/", ...)` creates a todo and returns `TypedResults.Created(...)`; `MapPut("/{id:int}", ...)` returns `Results<NoContent, NotFound>`; `MapDelete("/{id:int}", ...)` returns `TypedResults.NoContent()`. Note that `/{id:int}` requires the `:int` constraint — otherwise a string may reach the handler.
8. On `MapPost` add `AddEndpointFilter` that verifies `Title` is not empty and otherwise returns `TypedResults.BadRequest("Title is required / Заголовок обязателен")`. The filter should take `context.Arguments[0]` — the handler’s first argument — cast to `Todo`, and check `string.IsNullOrWhiteSpace(arg.Title)`. This is the cross-cutting logic example from the lesson: validation lives in one place, not duplicated in every handler.
9. In `Program.cs` call `app.MapTodoEndpoints();` after `var app = builder.Build();`. Run `dotnet run` and test the endpoints with `curl` or a REST client: `curl http://localhost:5000/api/todos` should return `[]`; `curl -X POST -H "Content-Type: application/json" -d '{"title":"Latte","done":false}' http://localhost:5000/api/todos` should return `201 Created` with a body and a `Location` header.
10. (Optional but recommended) Add Swagger via `builder.Services.AddEndpointsApiExplorer();` and `builder.Services.AddSwaggerGen();`, then in the pipeline `app.UseSwagger(); app.UseSwaggerUI();`. Open `/swagger` and confirm that the endpoints are grouped under the “Todos” tag and that the response schemas for `GET /{id}` show both `200 OK` with `Todo` and `404 Not Found` — this proves the `Results<>` union really yields a correct OpenAPI schema.
11. Test negative scenarios: `POST` with an empty `title` should return `400 BadRequest` from the filter; `GET /api/todos/abc` should return `404` (or `400` — depending on routing with the `:int` constraint; the key point is that the handler is not invoked for a non-`int`); `GET /api/todos/9999` should return `404 NotFound` with an empty body.
12. Commit the solution to git: `git init && git add . && git commit -m "M14-L02 minimal api todo"`.

#### Requirements
- The project is `dotnet new web`, targeting `net8.0`, C# 12 (you may use top-level statements, `record`, `with`, pattern matching, collection expressions, raw string literals where they improve readability).
- At least five endpoints on a single `/api/todos` group: `GET /`, `GET /{id:int}`, `POST /`, `PUT /{id:int}`, `DELETE /{id:int}`. All endpoints must be inside the group, not on the root `app`.
- Return values: at least three endpoints use `TypedResults` directly, at least one uses the union type `Results<Ok<Todo>, NotFound>` (for `GET /{id}`), and at least one uses `Results<NoContent, NotFound>` (for `PUT /{id}`). This demonstrates understanding of the `Results` vs `TypedResults` distinction and the ability to build union types.
- On `POST` there must be an `AddEndpointFilter` that checks `Title` and returns `TypedResults.BadRequest` for an empty title. The filter must correctly cast `context.Arguments[0]` to `Todo` and handle `null` (via `!` or a `?? new Todo(...)` fallback).
- `TodoStore` is registered in DI and obtained through handler parameters, not through closures. Writing `app.MapGet("/", () => _store.GetAll())` is forbidden — it is the lesson’s anti-pattern.
- Group registration is extracted into a static extension method `MapTodoEndpoints(this IEndpointRouteBuilder)`; `Program.cs` contains only `builder`/`app`/`app.MapTodoEndpoints()`/`app.Run()`.
- Route constraints `{id:int}` on every endpoint that has an `id`. No bare `{id}`.
- `CreatedAt` is set by the server in `Add`, not taken from the client body (tamper protection).
- The code compiles without `error`-level warnings, `dotnet run` starts, and endpoints return the expected status codes.

#### Pitfalls
- **`Results` vs `TypedResults`**: if you declared the signature `Results<Ok<Todo>, NotFound>`, use `TypedResults.Ok`/`TypedResults.NotFound` inside the handler. Mixing in the untyped `Results.Ok` loses the typing point and Swagger may show `200 OK` with no schema. This is the most common mistake in the lesson.
- **`null` instead of `NotFound`**: returning `null` from a handler declared as `Todo` (rather than `Results<...>`) gives the client `200 OK` with an empty body. The business will think the object exists but is empty. Always return an explicit `IResult` with a status code. If a handler might not find an object, its signature must be `Results<Ok<T>, NotFound>`, not just `T`.
- **Route constraints**: `{id}` without `:int` means that when routing tries to bind the string `"abc"` to `int id`, the handler is not invoked at all — you get `404` or `400`. But if the signature is `string id`, the string flows through and breaks business logic. Always specify `:int`, `:guid`, `:long` where appropriate — it is the first line of defense.
- **Capturing a service in a closure**: `app.MapGet("/", () => _store.GetAll())` works, but (a) breaks testing (you cannot substitute `_store`), (b) captures a concrete instance rather than the service from the current scope, and (c) for scoped services like EF `DbContext` causes a captive-dependency bug. Pass the service as a handler parameter: `(TodoStore store) => store.GetAll()`.
- **Filter and `context.Arguments`**: indexes in `Arguments` correspond to the order of handler parameters, **including services**. For `MapPost("/", (Todo todo, TodoStore store) => ...)` `Arguments[0]` is `Todo` and `Arguments[1]` is `TodoStore`. Always check `null` before casting — the filter may run before binding in some configurations.
- **`WithTags` on the group, not the endpoint**: if you put `WithTags("Todos")` only on one endpoint of the group, the rest will pile up in Swagger without a category. Attach the tag to the whole group and it applies to every endpoint.
- **`RequireAuthorization` on a group**: if the whole group is protected, do not duplicate `RequireAuthorization` on a single endpoint — it confuses policies. If one endpoint must be anonymous (`GET /public`), move it to a separate group or use `AllowAnonymous()`.
- **`MapGet("/")` vs `MapGet("")`**: on a group both resolve to the prefixed route, but `/` is canonical and avoids double slashes in the `Location` header. For `POST` with `Created($"/api/todos/{id}", ...)` make sure the prefix matches the group, otherwise the client gets a non-existent `Location`.

#### Acceptance criteria
- [ ] Project created with `dotnet new web`, targets `net8.0`, compiles without errors.
- [ ] There is a `record Todo` with at least `Id`, `Title`, `Done`, `CreatedAt`.
- [ ] `TodoStore` is implemented on `ConcurrentDictionary` (or an equivalent with locking) and is thread-safe.
- [ ] `TodoStore` is registered in DI as a singleton and is never `new`-ed inside handlers.
- [ ] At least five endpoints (`GET /`, `GET /{id:int}`, `POST /`, `PUT /{id:int}`, `DELETE /{id:int}`) inside a single `/api/todos` group.
- [ ] The group has `WithTags("Todos")` — endpoints are grouped under one category in Swagger.
- [ ] `GET /{id:int}` returns `Results<Ok<Todo>, NotFound>` (union type) and uses `TypedResults.NotFound()` when missing.
- [ ] `PUT /{id:int}` returns `Results<NoContent, NotFound>` and uses `TypedResults.NoContent()`/`TypedResults.NotFound()`.
- [ ] `POST /` returns `TypedResults.Created($"/api/todos/{id}", created)` with a correct `Location` header.
- [ ] `POST` has an `AddEndpointFilter` checking `Title` and returning `TypedResults.BadRequest` for an empty title.
- [ ] `TodoStore` is passed as a handler parameter in every endpoint; no closure capture.
- [ ] Group registration is extracted into a static extension method `MapTodoEndpoints`; `Program.cs` is clean.
- [ ] `{id:int}` constraints on every endpoint with an `id`.
- [ ] `CreatedAt` is set by the server in `Add`, not taken from the client body.
- [ ] `curl`/Swagger confirm: empty `POST` → `400`, `GET /{id:int}` of a missing item → `404`, successful `POST` → `201` with `Location`.
- [ ] No `null` returns instead of `NotFound` in endpoints where “not found” is possible.

#### Hints (no direct answer)
- For the counter use `Interlocked.Increment(ref _seq)` — plain `_seq++` is not thread-safe.
- In the filter, remember `Arguments` is `IList<object?>` and the order matches the handler parameter order.
- For the union `Results<Ok<T>, NotFound>` the type `T` must be a reference or nullable type — the compiler will tell you if `T` is a value type.
- Swagger needs `AddEndpointsApiExplorer()` specifically for Minimal API — controllers do not need it because MVC registers the explorer itself.
- For `Created` to produce a correct `Location`, pass an absolute or root-absolute path, not a relative one.

#### Reference solution walk-through
```csharp
// C# 12 / .NET 8 — Reference solution
using System.Collections.Concurrent;
using Microsoft.AspNetCore.Http.HttpResults;

var builder = WebApplication.CreateBuilder(args);

// Register the store as a singleton in DI
builder.Services.AddSingleton<TodoStore>();

// Minimal API Swagger needs the endpoints explorer
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

// Register the endpoint group via an extension method
app.MapTodoEndpoints();

app.Run();

// --- Static endpoint registration class ---
public static class TodoEndpoints
{
    // Extension method keeps Program.cs clean (lesson best practice)
    public static IEndpointRouteBuilder MapTodoEndpoints(this IEndpointRouteBuilder app)
    {
        // Group with shared prefix and Swagger tag
        var group = app.MapGroup("/api/todos")
                       .WithTags("Todos");

        // GET /api/todos — all todos
        group.MapGet("/", (TodoStore store) =>
        {
            return TypedResults.Ok(store.GetAll());
        });

        // GET /api/todos/{id} — the Results<> union yields a correct OpenAPI schema
        group.MapGet("/{id:int}", async Task<Results<Ok<Todo>, NotFound>> (int id, TodoStore store) =>
        {
            var todo = await store.FindAsync(id);
            return todo is null
                ? TypedResults.NotFound()   // explicit NotFound, NOT null
                : TypedResults.Ok(todo);
        });

        // POST /api/todos — create a todo
        // TodoStore is passed as a parameter, not captured (lesson anti-pattern avoided)
        group.MapPost("/", (Todo todo, TodoStore store) =>
        {
            var created = store.Add(todo);
            return TypedResults.Created($"/api/todos/{created.Id}", created);
        })
        .AddEndpointFilter(async (context, next) =>
        {
            // First handler argument is Todo
            var arg = (Todo)context.Arguments[0]!;
            if (string.IsNullOrWhiteSpace(arg.Title))
            {
                // Short-circuit with BadRequest
                return TypedResults.BadRequest("Title is required / Заголовок обязателен");
            }
            return await next(context); // pass control to the handler
        });

        // PUT /api/todos/{id} — update
        group.MapPut("/{id:int}", (int id, Todo todo, TodoStore store) =>
        {
            return store.Update(id, todo)
                ? Results.NoContent()
                : Results.NotFound();
        });

        // DELETE /api/todos/{id} — delete
        group.MapDelete("/{id:int}", (int id, TodoStore store) =>
        {
            store.Remove(id);
            return TypedResults.NoContent();
        });

        return app;
    }
}

// --- Model and store ---

public record Todo(int Id, string Title, bool Done, DateTime CreatedAt);

public sealed class TodoStore
{
    // ConcurrentDictionary for thread safety
    private readonly ConcurrentDictionary<int, Todo> _items = new();
    private int _seq = 0;

    public IReadOnlyCollection<Todo> GetAll() => _items.Values.ToArray();

    public Task<Todo?> FindAsync(int id) =>
        Task.FromResult(_items.TryGetValue(id, out var t) ? t : null);

    public Todo Add(Todo todo)
    {
        // Interlocked.Increment — thread-safe counter
        var id = Interlocked.Increment(ref _seq);
        // CreatedAt is set by the server, not the client (tamper protection)
        var created = todo with { Id = id, CreatedAt = DateTime.UtcNow };
        _items[id] = created;
        return created;
    }

    public bool Update(int id, Todo todo)
    {
        // Keep original Id and CreatedAt — the client must not change them
        if (!_items.TryGetValue(id, out var existing)) return false;
        _items[id] = todo with { Id = id, CreatedAt = existing.CreatedAt };
        return true;
    }

    public void Remove(int id) => _items.TryRemove(id, out _);
}
```

Line-by-line walk-through. `WebApplication.CreateBuilder(args)` is the standard Minimal API entry point; `builder.Services.AddSingleton<TodoStore>()` registers the store in DI as a singleton (appropriate for an in-memory dictionary, but **not** for a `DbContext` — that one is scoped). `AddEndpointsApiExplorer()` is needed specifically for Minimal API because MVC registers the explorer itself for controllers; without it `AddSwaggerGen()` will not see your endpoints. `app.MapTodoEndpoints()` is the extension-method call; this keeps `Program.cs` clean and lets you reuse registration in integration tests by feeding an `IEndpointRouteBuilder` from `WebApplicationFactory`.

Inside `MapTodoEndpoints`, `MapGroup("/api/todos").WithTags("Todos")` produces a shared prefix and a Swagger tag. The `{id:int}` constraint on every `id` endpoint protects the handler from strings like `"abc"`. The signature `Task<Results<Ok<Todo>, NotFound>>` is a union type: the compiler guarantees the handler returns one of two shapes, and the OpenAPI generator builds a schema with both statuses (`200` and `404`). Inside, `TypedResults.NotFound()` (not the untyped `Results.NotFound()`) preserves the typing. The `todo is null` check uses pattern matching — idiomatic C# 12.

`MapPost` with `AddEndpointFilter` shows cross-cutting logic: the filter takes `context.Arguments[0]` (the first parameter, `Todo todo`), checks `Title`, and on an empty title returns `TypedResults.BadRequest`, short-circuiting the chain (`next` is not called). When validation passes, `await next(context)` hands control to the handler. Crucially, `TodoStore` is a handler parameter, not a captured closure field — this is the lesson’s best practice, enabling testing and avoiding captive-dependency issues for scoped services. `TypedResults.Created($"/api/todos/{created.Id}", created)` sets status `201` and the `Location` header so the client can immediately `GET` the new resource.

In `TodoStore`, `ConcurrentDictionary` guarantees thread safety and `Interlocked.Increment` is an atomic counter (plain `++` races in concurrent scenarios). `Add` via `with` creates a new `record` with the server-assigned `Id` and `CreatedAt` — the client cannot tamper with them because they are overwritten. `Update` preserves the original `Id` and `CreatedAt` — otherwise a client could “move” a todo to a different `Id` or fake the creation date. `Remove` via `TryRemove` is safe even when the key is absent.

Lesson concepts applied: `MapGroup` + `WithTags` (grouping), `TypedResults` + `Results<>` (typing and OpenAPI), `AddEndpointFilter` (cross-cutting logic), route constraints (`{id:int}`), services as parameters (not closures), registration through a static extension method (clean `Program.cs`), explicit `NotFound`/`NoContent` (not `null`). All of the lesson’s common mistakes are deliberately absent from the reference solution — use that as a self-check baseline.

#### Going deeper (bonus)
1. Add pagination: `GET /api/todos?page=1&pageSize=10` with parameters via `AsParameters` or individual `int page, int pageSize`, returning `Results<Ok<PagedResult<Todo>>, BadRequest>`. Implement a `PagedResult<T>` record with `Items`, `Total`, `Page`, `PageSize`.
2. Add a logging endpoint filter: before `next` log `context.HttpContext.Request.Method` and the path, after — the status code and duration via `Stopwatch`. Confirm the filter applies to the whole group.
3. Implement a second filter — an exception handler: wrap `await next(context)` in `try/catch`, log the exception, and return `TypedResults.Problem(...)`. Verify that `GET /{id}` throwing `InvalidOperationException` inside `TodoStore` returns `500` with a JSON problem description instead of crashing the process.
4. Split endpoints into two nested groups: `/api/todos` (read, no auth) and `/api/todos/admin` (write, with `RequireAuthorization("admin")`). Demonstrate that `RequireAuthorization` on a nested group does not propagate to the parent and vice versa.

---

#### Чек-лист сдачи / Submission checklist
- [ ] (RU) Проект `CoffeeTodos.Api` создан и компилируется.
- [ ] (RU) Все пять эндпоинтов в группе `/api/todos` с тегом «Todos».
- [ ] (RU) `GET /{id}` и `PUT /{id}` используют union-тип `Results<>`.
- [ ] (RU) На `POST` есть `AddEndpointFilter` с проверкой `Title`.
- [ ] (RU) `TodoStore` получен через параметры, не замыканием.
- [ ] (RU) Регистрация вынесена в `MapTodoEndpoints`.
- [ ] (RU) `curl`-сценарии: `201`, `400`, `404` работают.
- [ ] (RU) Код закоммичен в git.
- [ ] (EN) Project `CoffeeTodos.Api` created and compiles.
- [ ] (EN) All five endpoints are in the `/api/todos` group with the “Todos” tag.
- [ ] (EN) `GET /{id}` and `PUT /{id}` use the `Results<>` union.
- [ ] (EN) `POST` has an `AddEndpointFilter` checking `Title`.
- [ ] (EN) `TodoStore` is obtained via parameters, not a closure.
- [ ] (EN) Registration is extracted into `MapTodoEndpoints`.
- [ ] (EN) `curl` scenarios: `201`, `400`, `404` all work.
- [ ] (EN) Code committed to git.

#### Ресурсы / Resources
- [Microsoft Learn — Minimal APIs](https://learn.microsoft.com/aspnet/core/fundamentals/minimal-apis/)
- [Minimal APIs overview](https://learn.microsoft.com/aspnet/core/fundamentals/minimal-apis/overview)
- [Route handlers and groups](https://learn.microsoft.com/aspnet/core/fundamentals/minimal-apis/route-handlers)
- [TypedResults API](https://learn.microsoft.com/dotnet/api/microsoft.aspnetcore.http.typedresults)
- [Endpoint filters](https://learn.microsoft.com/aspnet/core/fundamentals/minimal-apis/min-api-filters)
