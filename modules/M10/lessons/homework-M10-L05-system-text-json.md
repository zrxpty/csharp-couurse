---
[← К уроку M10-L05](lesson-M10-L05-system-text-json.md) | [⬆ К модулю M10](../README.md) | [Следующее ДЗ →](homework-M10-L06-memorystream-pipes.md)
---

### Домашнее задание M10-L05: System.Text.Json: сериализация/десериализация, опции, JsonSerializerContext / Homework M10-L05: System.Text.Json: serialization, options, JsonSerializerContext

**Урок / Lesson:** M10-L05
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 4/5
**Цель / Goal:** (RU) Научиться сериализовать и десериализовать объекты через `System.Text.Json`, грамотно настраивать `JsonSerializerOptions` как синглтон, управлять именами свойств через политики и атрибуты, игнорировать `null`, читать «грязный» JSON с комментариями и регистром ключей, а также реализовать source-generated контекст `JsonSerializerContext` для trim/AOT-совместимости и сравнить оба пути. (EN) Learn to serialize and deserialize objects with `System.Text.Json`, configure `JsonSerializerOptions` as a singleton, control property names via policies and attributes, ignore `null`, read "dirty" JSON with comments and mixed key casing, and implement a source-generated `JsonSerializerContext` for trim/AOT compatibility, then compare both paths.

#### Связь с уроком / Connection to the lesson

(RU) Домашнее задание напрямую опирается на примеры и best practices урока M10-L05: кэширование `JsonSerializerOptions`, использование `JsonSerializerDefaults.Web`, атрибуты `[JsonPropertyName]` и `[JsonSourceGenerationOptions]`, регистрация типов через `[JsonSerializable]`, а также типичные ошибки — пересоздание опций, забытый case-insensitive режим, циклы ссылок и комментарии в JSON. Вы пройдёте обе ветки сериализатора — рефлексионную и source-gen — чтобы увидеть разницу в производительности и совместимости с Native AOT.

(EN) This homework builds directly on the examples and best practices from lesson M10-L05: caching `JsonSerializerOptions`, using `JsonSerializerDefaults.Web`, the `[JsonPropertyName]` and `[JsonSourceGenerationOptions]` attributes, registering types with `[JsonSerializable]`, and the common mistakes — recreating options, forgetting case-insensitive mode, reference cycles, and comments in JSON. You will exercise both serializer branches — reflection and source-gen — to observe the difference in performance and Native AOT compatibility.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация

Вы присоединились к команде, разрабатывающей внутренний сервис «TaskTracker» для небольшой компании. Сервис хранит задачи, пользователей и комментарии, а общается с внешними системами (мобильным приложением, парой legacy-скриптов на Python и браузерным админ-фронтендом) исключительно через JSON-файлы и HTTP-like обмен строками. Команда уже набила шишки: фронтенд ломался, потому что C# отдавал `DueDate` в PascalCase, а JS ждал `dueDate`; мобильное приложение падало, потому что сервер иногда присылал `"due_date"`, а иногда `"DueDate"`; один из разработчиков создал `new JsonSerializerOptions()` прямо в цикле обработки запросов, и сервис начал тормозить так, что отчёты строились минутами вместо секунд. Кроме того, компания планирует выпустить утилиту командной строки для офлайн-импорта задач, и её хотят публиковать как Native AOT-бинарник — а значит, рефлексия в сериализаторе должна быть заменена на source generation.

Ваша задача — привести сериализацию в порядок: написать чистую модель данных, настроить опции один раз, реализовать чтение «грязного» JSON из внешних источников, добавить source-generated контекст для AOT и показать, что оба пути дают одинаковый результат, но source-gen быстрее и безопаснее для trimming. Это типичная работа middle/Senior C#-разработчика, который отвечает за надёжность интеграционного слоя. Вы не пишете веб-сервер — вы пишете библиотеку + консольную утилиту, которые сериализуют и десериализуют JSON разными способами и печатают сравнение. Это позволяет сфокусироваться именно на `System.Text.Json`, а не на инфраструктуре ASP.NET Core.

#### Что нужно сделать (пошагово)

1. **Создайте проект.** В пустой папке выполните `dotnet new console -n TaskTracker.Json -o TaskTracker.Json --framework net8.0`, затем перейдите в неё: `cd TaskTracker.Json`. Откройте `.csproj` и убедитесь, что включены `<Nullable>enable</Nullable>` и `<ImplicitUsings>enable</ImplicitUsings>`. Добавьте `<PublishTrimmed>true</PublishTrimmed>` и `<PublishAot>true</PublishAot>` в отдельный `PropertyGroup`-комментарием «для проверки AOT позже» — пока оставьте выключенным, чтобы не мешал обычной отладке, но держите конфигурацию готовой.

2. **Опишите модель данных.** Создайте файл `Models.cs` с записями (records) `User`, `TaskItem`, `Comment`. Поля: `User(int Id, string FirstName, string LastName, string? Email)`, `TaskItem(int Id, string Title, string? Description, int AssigneeId, DateTime DueDate, TaskStatus Status, List<Comment> Comments)`, `Comment(int Id, int AuthorId, string Text, DateTime CreatedAt)`. Используйте C# 12: record-типы, коллекционные выражения в инициализаторах, `required` там, где это уместно. Примените `[JsonPropertyName]` к ключевым полям так, чтобы внешнее имя было `snake_case` (`"id"`, `"first_name"`, `"due_date"`, `"task_status"`). Для `Status` используйте `enum TaskStatus { Todo, InProgress, Done, Cancelled }` и конвертер строк через `JsonStringEnumConverter`.

3. **Настройте опции как синглтон.** В `JsonConfig.cs` объявите `static readonly JsonSerializerOptions Options = new(JsonSerializerDefaults.Web) { WriteIndented = true, DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull, ReadCommentHandling = JsonCommentHandling.Skip, AllowTrailingCommas = true, Converters = { new JsonStringEnumConverter(JsonNamingPolicy.CamelCase) } };`. Никогда не пересоздавайте этот экземпляр в коде. В комментарии прямо в файле напишите, почему это синглтон (со ссылкой на кэш метаданных).

4. **Реализуйте рефлексионный путь.** В `ReflectionPipeline.cs` создайте класс с методами `string Serialize(TaskItem task)` и `TaskItem? Deserialize(string json)`, которые вызывают `JsonSerializer.Serialize`/`Deserialize` с `Options`. Добавьте метод `SerializeStreamAsync(TaskItem task, Stream stream)`, использующий `JsonSerializer.SerializeAsync`, чтобы продемонстрировать потоковую запись.

5. **Реализуйте source-gen путь.** В `AppJsonContext.cs` объявите `internal partial class AppJsonContext : JsonSerializerContext { }` с атрибутами `[JsonSourceGenerationOptions(...)]` (повторите ключевые настройки: camelCase, case-insensitive, `WriteIndented`, `WhenWritingNull`, `JsonStringEnumConverter`) и `[JsonSerializable(typeof(TaskItem))]`, `[JsonSerializable(typeof(List<TaskItem>))]`, `[JsonSerializable(typeof(User))]`. Создайте класс `SourceGenPipeline.cs` с методами, аналогичными рефлексионным, но вызывающими `JsonSerializer.Serialize(task, AppJsonContext.Default.TaskItem)` и т.д.

6. **Подготовьте «грязный» тестовый JSON.** В `sample.json` положите JSON с комментариями (`// ...`), trailing-запятыми, ключами в разном регистре (`"ID"`, `"FIRST_NAME"`, `"task_status": "in_progress"`) и одним полем `"due_date"` в ISO-8601. Убедитесь, что без `ReadCommentHandling = Skip` и `PropertyNameCaseInsensitive` десериализация падает с `JsonException`.

7. **Напишите Program.cs (top-level statements).** Создайте образец `TaskItem` с двумя комментариями, сериализуйте его обоими путями, распечатайте JSON, десериализуйте «грязный» `sample.json` обоими путями и распечатайте поля. Замерьте время 10 000 сериализаций каждым путём через `Stopwatch` и выведите сравнение. Используйте raw-string literals `"""..."""` для встраивания JSON в код.

8. **Запустите и проверьте.** `dotnet build`, затем `dotnet run`. Убедитесь, что оба пути дают одинаковые объекты, source-gen не выбрасывает `NotSupportedException`, а замеры показывают, что source-gen быстрее (особенно первый вызов). В выводе должны быть: JSON-представление задачи, распарсенный объект из «грязного» JSON и таблица замеров.

9. **(Бонус, опционально)** Выполните `dotnet publish -r win-x64 -c Release` с включённым `PublishAot` (временно раскомментируйте свойства) и убедитесь, что сборка проходит без warning IL3050/IL2026, связанных с reflection. Если warnings есть — найдите, какой `[JsonSerializable]` пропущен.

#### Требования к решению

- Целевой фреймворк — `net8.0`; C# 12 включён по умолчанию. Используйте top-level statements в `Program.cs`, record-типы для моделей, коллекционные выражения (`[]`, `[a, b]`) при инициализации списков, raw-string literals для JSON.
- `JsonSerializerOptions` создаётся ровно один раз как `static readonly` поле и переиспользуется всеми методами. Никаких `new JsonSerializerOptions()` в циклах или методах.
- Включены: `JsonNamingPolicy.CamelCase` (через `JsonSerializerDefaults.Web`), `PropertyNameCaseInsensitive = true`, `DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull`, `ReadCommentHandling = JsonCommentHandling.Skip`, `AllowTrailingCommas = true`. Перечисления сериализуются строками через `JsonStringEnumConverter`.
- Source-gen контекст `AppJsonContext` оформлен как `internal partial class : JsonSerializerContext` с `[JsonSourceGenerationOptions]` и `[JsonSerializable]` для всех используемых типов (`TaskItem`, `List<TaskItem>`, `User`). Имена свойств в source-gen совпадают с рефлексионным путём.
- Оба пути (reflection и source-gen) демонстрируются на одном объекте; результаты выводятся и сверяются. «Грязный» JSON (комментарии, trailing commas, смешанный регистр) успешно десериализуется обоими путями.
- Код компилируется без warning-ов, nullable-аннотации корректны (`string?` для nullable, `!` только там, где это безопасно). `dotnet build` чистый.
- Программа выводит человекочитаемый отчёт: сериализованный JSON, поля распарсенного объекта, таблицу замеров `Stopwatch` (минимум 10 000 итераций на путь).
- Никаких внешних NuGet-пакетов: только встроенный `System.Text.Json`. `Newtonsoft.Json` не использовать.

#### Тонкости и подводные камни

- **Пересоздание опций убивает производительность.** Урок подчёркивает: сериализатор кэширует метаданные внутри `JsonSerializerOptions`. Если вы пишете `new JsonSerializerOptions()` на каждый вызов, кэш каждый раз строится заново, и на больших payload-ах замедление может достигать десятков раз. В этом ДЗ опции — синглтон, и в комментарии должно быть объяснение.
- **PascalCase vs camelCase.** По умолчанию `System.Text.Json` отдаёт имена свойств как в C# (`FirstName`). Внешние клиенты (JS, Python) ждут `firstName`. Решение — `JsonNamingPolicy.CamelCase` (через `JsonSerializerDefaults.Web`) или явные `[JsonPropertyName]`. В этом задании вы используете оба: политики на уровне опций + `[JsonPropertyName]` для переопределения конкретных полей в `snake_case`. Внимание: `[JsonPropertyName]` имеет приоритет над политикой именования.
- **Case-insensitive — критично для внешних источников.** Если внешнее API присылает `"id"` вместо `"Id"`, без `PropertyNameCaseInsensitive = true` поле молча останется `0`/`null`. В «грязном» `sample.json` ключи намеренно в разном регистре — это проверка.
- **Комментарии и trailing commas.** Дефолтные опции бросают `JsonException` на `// comment` и на `,]`. Включайте `ReadCommentHandling = JsonCommentHandling.Skip` и `AllowTrailingCommas = true` для чтения человеко-редактируемого JSON.
- **Source-gen: забытый `[JsonSerializable]`.** Если вызвать `AppJsonContext.Default.SomeType` для типа, не отмеченного `[JsonSerializable]`, получите `NotSupportedException` в рантайме. Регистрируйте ВСЕ типы, включая `List<TaskItem>` (а не только `TaskItem`).
- **Перечисления.** По умолчанию enum сериализуется числом. Для человекочитаемого JSON и совместимости с внешними клиентами нужен `JsonStringEnumConverter`. В source-gen его надо указать в `[JsonSourceGenerationOptions(UseStringEnumConverter = true)]` или передать `JsonStringEnumConverter` через `Converters` — проверьте, что оба пути выдают `"in_progress"`, а не `1`.
- **AOT/trim warnings.** Рефлексионный путь в `PublishAot`-сборке порождает IL3050/IL2026 и может падать в publish. Source-gen этих warnings не имеет, потому что код генерируется на этапе компиляции и не требует метаданных рефлексии. Если у вас остаются warnings — значит, где-то используется reflection-over-JSON.

#### Критерии приёмки

- [ ] Проект `TaskTracker.Json` создан, `dotnet build` проходит без errors и без warnings, `net8.0` + C# 12.
- [ ] Модели `User`, `TaskItem`, `Comment`, `TaskStatus` — records с корректными nullable-аннотациями.
- [ ] `[JsonPropertyName]` применён к полям так, что внешние имена — `snake_case` (`id`, `first_name`, `due_date`), а политика опций делает camelCase для остальных.
- [ ] `JsonSerializerOptions` создан один раз как `static readonly`, в комментарии объяснён синглтон-паттерн.
- [ ] Включены `PropertyNameCaseInsensitive`, `DefaultIgnoreCondition = WhenWritingNull`, `ReadCommentHandling = Skip`, `AllowTrailingCommas = true`.
- [ ] `JsonStringEnumConverter` зарегистрирован; `TaskStatus` сериализуется строкой (`"in_progress"`).
- [ ] Рефлексионный путь (`ReflectionPipeline`) реализован, включая `SerializeAsync` в `Stream`.
- [ ] `AppJsonContext` объявлен как `partial class : JsonSerializerContext` с `[JsonSerializable]` для `TaskItem`, `List<TaskItem>`, `User`.
- [ ] `SourceGenPipeline` использует `AppJsonContext.Default.TaskItem` и т.п.; `NotSupportedException` не возникает.
- [ ] «Грязный» `sample.json` (комментарии, trailing commas, смешанный регистр) десериализуется обоими путями без исключений.
- [ ] `Program.cs` использует top-level statements, raw-string literals для JSON, коллекционные выражения для списков.
- [ ] Вывод содержит: сериализованный JSON задачи, поля распарсенного объекта, таблицу замеров `Stopwatch` (≥10 000 итераций на путь).
- [ ] Source-gen путь в замерах не медленнее рефлексионного; первый вызов source-gen заметно быстрее первого вызова рефлексии (если замерять «холодный старт»).
- [ ] Никаких внешних NuGet-пакетов; `Newtonsoft.Json` отсутствует.
- [ ] (Бонус) `dotnet publish -r win-x64 -c Release` с `PublishAot` проходит без IL3050/IL2026 по сериализации.

#### Подсказки (без прямого ответа)

- Вспомните из урока: `JsonSerializerDefaults.Web` уже включает camelCase + case-insensitive + trailing commas — вам останется добавить только `ReadCommentHandling` и `DefaultIgnoreCondition`.
- Для `JsonStringEnumConverter` в source-gen используйте `UseStringEnumConverter = true` в `[JsonSourceGenerationOptions]`, иначе enum уйдёт числом.
- Чтобы замерить «холодный старт» отдельно, вызовите сериализацию один раз до `Stopwatch`, а потом замеряйте пачку — так вы увидите разницу в первом вызове.
- Если `AppJsonContext.Default.ListTaskItem` ругается — проверьте, что `List<TaskItem>` зарегистрирован отдельно от `TaskItem`. Дженерик-коллекция — это отдельный тип для генератора.
- Raw-string literal `"""{ "id": 1 }"""` удобен для встраивания JSON с двойными кавычками без эскейпинга.

#### Эталонное решение (разбор)

```csharp
// C# 12 / .NET 8 — TaskTracker.Json: reflection + source-gen
// Эталонное решение ДЗ M10-L05 / Reference solution for HW M10-L05

using System.Diagnostics;
using System.Text.Json;
using System.Text.Json.Serialization;

namespace TaskTracker.Json;

// --- Модели данных / Data models ---
// Внешние имена зафиксированы как snake_case через [JsonPropertyName].
// External names pinned to snake_case via [JsonPropertyName].
public enum TaskStatus { Todo, InProgress, Done, Cancelled }

public record User(
    [property: JsonPropertyName("id")] int Id,
    [property: JsonPropertyName("first_name")] string FirstName,
    [property: JsonPropertyName("last_name")] string LastName,
    [property: JsonPropertyName("email")] string? Email);

public record Comment(
    [property: JsonPropertyName("id")] int Id,
    [property: JsonPropertyName("author_id")] int AuthorId,
    [property: JsonPropertyName("text")] string Text,
    [property: JsonPropertyName("created_at")] DateTime CreatedAt);

public record TaskItem(
    [property: JsonPropertyName("id")] int Id,
    [property: JsonPropertyName("title")] string Title,
    [property: JsonPropertyName("description")] string? Description,
    [property: JsonPropertyName("assignee_id")] int AssigneeId,
    [property: JsonPropertyName("due_date")] DateTime DueDate,
    [property: JsonPropertyName("task_status")] TaskStatus Status,
    [property: JsonPropertyName("comments")] List<Comment> Comments);

// --- Опции (синглтон!) / Options (singleton!) ---
// Сериализатор кэширует метаданные типов внутри JsonSerializerOptions.
// Пересоздание опций = перестройка кэша = кратное падение производительности.
// The serializer caches type metadata inside JsonSerializerOptions.
// Recreating options = rebuilding the cache = order-of-magnitude slowdown.
public static class JsonConfig
{
    public static readonly JsonSerializerOptions Options = new(JsonSerializerDefaults.Web)
    {
        WriteIndented = true,
        DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull,
        ReadCommentHandling = JsonCommentHandling.Skip,
        AllowTrailingCommas = true,
        Converters = { new JsonStringEnumConverter(JsonNamingPolicy.CamelCase) }
    };
}

// --- Source-generated контекст / Source-generated context ---
// Генератор создаёт код сериализации на этапе компиляции: без рефлексии,
// безопасно для trimming и Native AOT.
// The generator emits serialization code at compile time: no reflection,
// safe for trimming and Native AOT.
[JsonSourceGenerationOptions(
    PropertyNamingPolicy = JsonSourceGenerationPropertyNamingPolicy.CamelCase,
    PropertyNameCaseInsensitive = true,
    WriteIndented = true,
    DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull,
    ReadCommentHandling = JsonCommentHandling.Skip,
    AllowTrailingCommas = true,
    UseStringEnumConverter = true)]
[JsonSerializable(typeof(TaskItem))]
[JsonSerializable(typeof(List<TaskItem>))]
[JsonSerializable(typeof(User))]
internal partial class AppJsonContext : JsonSerializerContext { }

// --- Рефлексионный путь / Reflection pipeline ---
public static class ReflectionPipeline
{
    public static string Serialize(TaskItem task) =>
        JsonSerializer.Serialize(task, JsonConfig.Options);

    public static TaskItem? Deserialize(string json) =>
        JsonSerializer.Deserialize<TaskItem>(json, JsonConfig.Options);

    public static Task SerializeStreamAsync(TaskItem task, Stream stream) =>
        JsonSerializer.SerializeAsync(stream, task, JsonConfig.Options);
}

// --- Source-gen путь / Source-gen pipeline ---
public static class SourceGenPipeline
{
    public static string Serialize(TaskItem task) =>
        JsonSerializer.Serialize(task, AppJsonContext.Default.TaskItem);

    public static TaskItem? Deserialize(string json) =>
        JsonSerializer.Deserialize(json, AppJsonContext.Default.TaskItem);
}

// --- Program.cs (top-level statements) ---
var sample = new TaskItem(
    Id: 42,
    Title: "Refactor JSON layer",
    Description: null,
    AssigneeId: 7,
    DueDate: new DateTime(2025, 1, 31, 18, 0, 0, DateTimeKind.Utc),
    Status: TaskStatus.InProgress,
    Comments:
    [
        new(1, 7, "Started", DateTime.UtcNow),
        new(2, 7, "Need review", DateTime.UtcNow)
    ]);

string dirtyJson = """
    // внешний JSON с комментариями и разным регистром
    {
      "ID": 99,
      "Title": "External task",
      "DESCRIPTION": null,
      "assignee_id": 5,
      "due_date": "2025-02-14T12:00:00Z",
      "task_status": "in_progress",   // trailing comma + comment
      "comments": [
        { "id": 10, "author_id": 5, "text": "Imported", "created_at": "2025-01-01T00:00:00Z" }
      ]
    }
    """;

Console.WriteLine("=== Reflection serialize ===");
Console.WriteLine(ReflectionPipeline.Serialize(sample));

Console.WriteLine("=== Source-gen serialize ===");
Console.WriteLine(SourceGenPipeline.Serialize(sample));

Console.WriteLine("=== Deserialize dirty JSON (reflection) ===");
var r1 = ReflectionPipeline.Deserialize(dirtyJson);
Console.WriteLine($"{r1!.Id} {r1.Title} {r1.Status} comments={r1.Comments.Count}");

Console.WriteLine("=== Deserialize dirty JSON (source-gen) ===");
var r2 = SourceGenPipeline.Deserialize(dirtyJson);
Console.WriteLine($"{r2!.Id} {r2.Title} {r2.Status} comments={r2.Comments.Count}");

// --- Замеры / Benchmarks ---
const int N = 10_000;
_ = ReflectionPipeline.Serialize(sample);   // прогрев / warmup
_ = SourceGenPipeline.Serialize(sample);

var sw = Stopwatch.StartNew();
for (int i = 0; i < N; i++) _ = ReflectionPipeline.Serialize(sample);
sw.Stop();
long reflectionMs = sw.ElapsedMilliseconds;

sw.Restart();
for (int i = 0; i < N; i++) _ = SourceGenPipeline.Serialize(sample);
sw.Stop();
long sourceGenMs = sw.ElapsedMilliseconds;

Console.WriteLine($"Benchmark ({N} iterations):");
Console.WriteLine($"  reflection : {reflectionMs} ms");
Console.WriteLine($"  source-gen : {sourceGenMs} ms");
```

Разбор по строкам: модели оформлены `record`-ами — это даёт value-семантику и лаконичный синтаксис конструктора; ` [property: JsonPropertyName(...)]` применяет атрибут к свойству автосгенерированного рекорда (без `property:` атрибут попал бы на параметр конструктора, что для `System.Text.Json` недостаточно). Внешние имена зафиксированы как `snake_case` — это моделирует интеграцию с Python/legacy API, где snake_case — норма. Опции создаются один раз через `new(JsonSerializerDefaults.Web)`: этот конструктор уже включает camelCase + case-insensitive + trailing commas, а мы добавляем `WriteIndented`, `WhenWritingNull`, `ReadCommentHandling = Skip` и `JsonStringEnumConverter`. `JsonConfig.Options` — `static readonly`, то есть синглтон; сериализатор кэширует внутри него метаданные типов, и повторное использование даёт кратный прирост скорости (прямой best practice из урока).

`AppJsonContext` — `internal partial class : JsonSerializerContext`. `partial` обязателен: генератор дописывает вторую часть класса с реализацией `JsonTypeInfo<T>` для каждого `[JsonSerializable]`. В `[JsonSourceGenerationOptions]` мы повторяем настройки рефлексионных опций: обратите внимание, что здесь нет `JsonSerializerDefaults.Web` — все настройки задаются явно, и вместо `JsonNamingPolicy.CamelCase` используется `JsonSourceGenerationPropertyNamingPolicy.CamelCase` (source-gen-специфичный enum). `UseStringEnumConverter = true` заменяет `JsonStringEnumConverter` из рефлексионного пути. `ReflectionPipeline` и `SourceGenPipeline` симметричны: одни и те же методы, разные перегрузки `JsonSerializer.Serialize`/`Deserialize`. Для source-gen передаётся `AppJsonContext.Default.TaskItem` (это `JsonTypeInfo<TaskItem>`), и сериализатор использует заранее сгенерированный код без reflection.

`Program.cs` — top-level statements. Коллекционное выражение `[ new(...), new(...) ]` инициализирует `List<Comment>` без явного `new List<Comment> { ... }`. Raw-string literal `"""..."""` позволяет встроить JSON с двойными кавычками и даже комментариями без эскейпинга — это и есть «грязный» payload. Прогрев перед замерами важен: первый вызов рефлексии строит кэш метаданных, и без прогрева замер будет искажён. `Stopwatch` показывает, что source-gen на холодном старте быстрее (нет рефлексии), а на разогретом — обычно сопоставим или чуть быстрее. Бонус-проверка: `dotnet publish -r win-x64 -c Release` с `PublishAot` проходит без IL3050/IL2026 именно потому, что весь JSON-путь идёт через `AppJsonContext`, и trimmer ничего не вырезает.

#### Задания на углубление (бонус)

1. **Reference cycles.** Добавьте в `Comment` nullable-поле `ReplyToComment? Parent`. Создайте два комментария, ссылающихся друг на друга, и попробуйте сериализовать. Получите `JsonException` "cycle detected". Исправьте через `ReferenceHandler = ReferenceHandler.Preserve` (в обоих путях) и объясните, как `$id`/`$ref` меняют payload. Затем попробуйте `ReferenceHandler.IgnoreCycles` и сравните результат.
2. **Streaming больших данных.** Сгенерируйте `List<TaskItem>` из 100 000 элементов и сериализуйте его в файл через `SerializeAsync` в `FileStream`. Замерьте пиковую память (через `GC.GetTotalMemory`) и сравните с синхронной `Serialize` в строку. Объясните, почему стриминг экономит память.
3. **Custom converter.** Напишите `JsonConverter<DateTime>` (или `JsonConverter<DateOnly>`), который пишет дату в формате `dd.MM.yyyy` для русского UI и читает несколько форматов. Зарегистрируйте его в `JsonConfig.Options` и в `[JsonSourceGenerationOptions]` (через `Converters = typeof(...)` в source-gen-варианте).
4. **Полный AOT-сборка.** Включите `PublishAot=true`, выполните `dotnet publish -r win-x64 -c Release`, измерьте размер бинарника с source-gen и без (закомментировав `AppJsonContext` и используя reflection) — последний должен либо падать с warning-ами, либо ломаться в рантайме. Зафиксируйте вывод.

---

## Statement in English / Постановка на английском

#### Context & motivation

You have joined a team building an in-house "TaskTracker" service for a small company. The service stores tasks, users, and comments, and it talks to external systems (a mobile app, a couple of legacy Python scripts, and a browser-based admin frontend) exclusively through JSON files and HTTP-like string exchange. The team has already collected its share of bruises: the frontend broke because C# emitted `DueDate` in PascalCase while the JS code expected `dueDate`; the mobile app crashed because the server sometimes sent `"due_date"` and sometimes `"DueDate"`; one developer created `new JsonSerializerOptions()` right inside the request-handling loop, and the service slowed down so much that reports took minutes instead of seconds. On top of that, the company plans to ship a command-line utility for offline task import as a Native AOT binary — which means reflection in the serializer must be replaced with source generation.

Your job is to bring serialization into shape: write a clean data model, configure options once, implement reading of "dirty" JSON coming from external sources, add a source-generated context for AOT, and demonstrate that both paths produce the same result while source-gen is faster and safer for trimming. This is a typical middle/Senior C# developer task where you own the reliability of the integration layer. You are not building a web server — you are building a library plus a console utility that serialize and deserialize JSON in different ways and print a comparison. This keeps the focus on `System.Text.Json` itself rather than on ASP.NET Core infrastructure.

#### What to do step by step

1. **Create the project.** In an empty folder run `dotnet new console -n TaskTracker.Json -o TaskTracker.Json --framework net8.0`, then `cd TaskTracker.Json`. Open the `.csproj` and confirm `<Nullable>enable</Nullable>` and `<ImplicitUsings>enable</ImplicitUsings>` are present. Add `<PublishTrimmed>true</PublishTrimmed>` and `<PublishAot>true</PublishAot>` inside a commented-out `PropertyGroup` labelled "for AOT check later" — keep them disabled during normal debugging, but keep the configuration ready.

2. **Describe the data model.** Create `Models.cs` with records `User`, `TaskItem`, `Comment`. Fields: `User(int Id, string FirstName, string LastName, string? Email)`, `TaskItem(int Id, string Title, string? Description, int AssigneeId, DateTime DueDate, TaskStatus Status, List<Comment> Comments)`, `Comment(int Id, int AuthorId, string Text, DateTime CreatedAt)`. Use C# 12: record types, collection expressions in initializers, `required` where it fits. Apply `[JsonPropertyName]` to key fields so the external name is `snake_case` (`"id"`, `"first_name"`, `"due_date"`, `"task_status"`). For `Status` use `enum TaskStatus { Todo, InProgress, Done, Cancelled }` and convert it to string via `JsonStringEnumConverter`.

3. **Configure options as a singleton.** In `JsonConfig.cs` declare `static readonly JsonSerializerOptions Options = new(JsonSerializerDefaults.Web) { WriteIndented = true, DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull, ReadCommentHandling = JsonCommentHandling.Skip, AllowTrailingCommas = true, Converters = { new JsonStringEnumConverter(JsonNamingPolicy.CamelCase) } };`. Never recreate this instance anywhere in the code. Add an inline comment explaining why it must be a singleton (referencing the metadata cache).

4. **Implement the reflection path.** In `ReflectionPipeline.cs` create a class with methods `string Serialize(TaskItem task)` and `TaskItem? Deserialize(string json)` that call `JsonSerializer.Serialize`/`Deserialize` with `Options`. Add `SerializeStreamAsync(TaskItem task, Stream stream)` using `JsonSerializer.SerializeAsync` to demonstrate streaming writes.

5. **Implement the source-gen path.** In `AppJsonContext.cs` declare `internal partial class AppJsonContext : JsonSerializerContext { }` with `[JsonSourceGenerationOptions(...)]` (repeat the key settings: camelCase, case-insensitive, `WriteIndented`, `WhenWritingNull`, string enums) and `[JsonSerializable(typeof(TaskItem))]`, `[JsonSerializable(typeof(List<TaskItem>))]`, `[JsonSerializable(typeof(User))]`. Create `SourceGenPipeline.cs` with methods mirroring the reflection ones, but calling `JsonSerializer.Serialize(task, AppJsonContext.Default.TaskItem)` and so on.

6. **Prepare a "dirty" test JSON.** Put into `sample.json` a JSON document with comments (`// ...`), trailing commas, keys in mixed case (`"ID"`, `"FIRST_NAME"`, `"task_status": "in_progress"`), and one `"due_date"` field in ISO-8601. Confirm that without `ReadCommentHandling = Skip` and `PropertyNameCaseInsensitive` the deserialization throws `JsonException`.

7. **Write Program.cs (top-level statements).** Build a sample `TaskItem` with two comments, serialize it both ways, print the JSON, deserialize the dirty `sample.json` both ways, and print the fields. Measure the time of 10,000 serializations for each path using `Stopwatch` and print a comparison. Use raw-string literals `"""..."""` to embed JSON in code.

8. **Run and verify.** `dotnet build`, then `dotnet run`. Confirm that both paths yield identical objects, that source-gen does not throw `NotSupportedException`, and that benchmarks show source-gen is faster (especially the first call). The output must include the JSON representation of the task, the parsed object from the dirty JSON, and the benchmark table.

9. **(Bonus, optional)** Run `dotnet publish -r win-x64 -c Release` with `PublishAot` enabled (temporarily uncomment the properties) and confirm the build passes without IL3050/IL2026 warnings related to reflection. If warnings appear, find which `[JsonSerializable]` is missing.

#### Requirements

- Target framework is `net8.0`; C# 12 is on by default. Use top-level statements in `Program.cs`, record types for models, collection expressions (`[]`, `[a, b]`) for list initialization, and raw-string literals for JSON.
- `JsonSerializerOptions` is created exactly once as a `static readonly` field and reused by every method. No `new JsonSerializerOptions()` inside loops or methods.
- Enabled: `JsonNamingPolicy.CamelCase` (via `JsonSerializerDefaults.Web`), `PropertyNameCaseInsensitive = true`, `DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull`, `ReadCommentHandling = JsonCommentHandling.Skip`, `AllowTrailingCommas = true`. Enums are serialized as strings via `JsonStringEnumConverter`.
- The source-gen context `AppJsonContext` is declared as `internal partial class : JsonSerializerContext` with `[JsonSourceGenerationOptions]` and `[JsonSerializable]` for every type in use (`TaskItem`, `List<TaskItem>`, `User`). Property names in source-gen match the reflection path.
- Both paths (reflection and source-gen) are demonstrated on the same object; results are printed and cross-checked. The "dirty" JSON (comments, trailing commas, mixed casing) deserializes successfully through both paths.
- The code compiles without warnings, nullable annotations are correct (`string?` for nullable, `!` only where safe). `dotnet build` is clean.
- The program prints a human-readable report: serialized JSON, fields of the parsed object, and a `Stopwatch` benchmark table (at least 10,000 iterations per path).
- No external NuGet packages: only the built-in `System.Text.Json`. Do not use `Newtonsoft.Json`.

#### Pitfalls

- **Recreating options kills performance.** The lesson stresses that the serializer caches type metadata inside `JsonSerializerOptions`. If you write `new JsonSerializerOptions()` per call, the cache is rebuilt every time, and on large payloads the slowdown can reach orders of magnitude. In this homework options are a singleton, and the comment must explain why.
- **PascalCase vs camelCase.** By default `System.Text.Json` emits property names as in C# (`FirstName`). External clients (JS, Python) expect `firstName`. The fix is `JsonNamingPolicy.CamelCase` (via `JsonSerializerDefaults.Web`) or explicit `[JsonPropertyName]`. Here you use both: a naming policy at the options level plus `[JsonPropertyName]` to override specific fields into `snake_case`. Note: `[JsonPropertyName]` takes precedence over the naming policy.
- **Case-insensitive is critical for external sources.** If an external API sends `"id"` instead of `"Id"`, without `PropertyNameCaseInsensitive = true` the field silently stays `0`/`null`. The dirty `sample.json` deliberately mixes casing — this is the test.
- **Comments and trailing commas.** Default options throw `JsonException` on `// comment` and on `,]`. Enable `ReadCommentHandling = JsonCommentHandling.Skip` and `AllowTrailingCommas = true` when reading human-edited JSON.
- **Source-gen: a forgotten `[JsonSerializable]`.** If you call `AppJsonContext.Default.SomeType` for a type not marked with `[JsonSerializable]`, you get a runtime `NotSupportedException`. Register ALL types, including `List<TaskItem>` (not just `TaskItem`).
- **Enums.** By default an enum is serialized as a number. For human-readable JSON and external-client compatibility you need `JsonStringEnumConverter`. In source-gen it must be set via `[JsonSourceGenerationOptions(UseStringEnumConverter = true)]` or by passing a converter — verify that both paths emit `"in_progress"` instead of `1`.
- **AOT/trim warnings.** The reflection path in a `PublishAot` build produces IL3050/IL2026 and may fail at publish. Source-gen has none of these warnings because the code is generated at compile time and does not need reflection metadata. If you still see warnings, reflection-over-JSON is being used somewhere.

#### Acceptance criteria

- [ ] The `TaskTracker.Json` project is created; `dotnet build` succeeds with no errors and no warnings; `net8.0` + C# 12.
- [ ] Models `User`, `TaskItem`, `Comment`, `TaskStatus` are records with correct nullable annotations.
- [ ] `[JsonPropertyName]` is applied so external names are `snake_case` (`id`, `first_name`, `due_date`), while the naming policy camelCases the rest.
- [ ] `JsonSerializerOptions` is created once as `static readonly`; a comment explains the singleton pattern.
- [ ] `PropertyNameCaseInsensitive`, `DefaultIgnoreCondition = WhenWritingNull`, `ReadCommentHandling = Skip`, `AllowTrailingCommas = true` are enabled.
- [ ] `JsonStringEnumConverter` is registered; `TaskStatus` serializes as a string (`"in_progress"`).
- [ ] The reflection path (`ReflectionPipeline`) is implemented, including `SerializeAsync` into a `Stream`.
- [ ] `AppJsonContext` is declared as `partial class : JsonSerializerContext` with `[JsonSerializable]` for `TaskItem`, `List<TaskItem>`, `User`.
- [ ] `SourceGenPipeline` uses `AppJsonContext.Default.TaskItem` etc.; no `NotSupportedException`.
- [ ] The dirty `sample.json` (comments, trailing commas, mixed casing) deserializes through both paths without exceptions.
- [ ] `Program.cs` uses top-level statements, raw-string literals for JSON, and collection expressions for lists.
- [ ] The output contains: serialized JSON of the task, fields of the parsed object, and a `Stopwatch` table (≥10,000 iterations per path).
- [ ] Source-gen is not slower than reflection in benchmarks; the first source-gen call is noticeably faster than the first reflection call (if you measure cold start).
- [ ] No external NuGet packages; `Newtonsoft.Json` is absent.
- [ ] (Bonus) `dotnet publish -r win-x64 -c Release` with `PublishAot` passes without IL3050/IL2026 from serialization.

#### Hints (no direct answer)

- Recall from the lesson: `JsonSerializerDefaults.Web` already turns on camelCase + case-insensitive + trailing commas — you only add `ReadCommentHandling` and `DefaultIgnoreCondition`.
- For `JsonStringEnumConverter` in source-gen use `UseStringEnumConverter = true` inside `[JsonSourceGenerationOptions]`, otherwise the enum goes out as a number.
- To measure cold start separately, call serialization once before `Stopwatch`, then benchmark a batch — you will see the first-call difference.
- If `AppJsonContext.Default.ListTaskItem` complains — check that `List<TaskItem>` is registered separately from `TaskItem`. A generic collection is a distinct type for the generator.
- A raw-string literal `"""{ "id": 1 }"""` is handy for embedding JSON with double quotes without escaping.

#### Reference solution walk-through

```csharp
// C# 12 / .NET 8 — TaskTracker.Json: reflection + source-gen
// Reference solution for HW M10-L05

using System.Diagnostics;
using System.Text.Json;
using System.Text.Json.Serialization;

namespace TaskTracker.Json;

// --- Data models ---
// External names pinned to snake_case via [JsonPropertyName].
public enum TaskStatus { Todo, InProgress, Done, Cancelled }

public record User(
    [property: JsonPropertyName("id")] int Id,
    [property: JsonPropertyName("first_name")] string FirstName,
    [property: JsonPropertyName("last_name")] string LastName,
    [property: JsonPropertyName("email")] string? Email);

public record Comment(
    [property: JsonPropertyName("id")] int Id,
    [property: JsonPropertyName("author_id")] int AuthorId,
    [property: JsonPropertyName("text")] string Text,
    [property: JsonPropertyName("created_at")] DateTime CreatedAt);

public record TaskItem(
    [property: JsonPropertyName("id")] int Id,
    [property: JsonPropertyName("title")] string Title,
    [property: JsonPropertyName("description")] string? Description,
    [property: JsonPropertyName("assignee_id")] int AssigneeId,
    [property: JsonPropertyName("due_date")] DateTime DueDate,
    [property: JsonPropertyName("task_status")] TaskStatus Status,
    [property: JsonPropertyName("comments")] List<Comment> Comments);

// --- Options (singleton!) ---
// The serializer caches type metadata inside JsonSerializerOptions.
// Recreating options = rebuilding the cache = order-of-magnitude slowdown.
public static class JsonConfig
{
    public static readonly JsonSerializerOptions Options = new(JsonSerializerDefaults.Web)
    {
        WriteIndented = true,
        DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull,
        ReadCommentHandling = JsonCommentHandling.Skip,
        AllowTrailingCommas = true,
        Converters = { new JsonStringEnumConverter(JsonNamingPolicy.CamelCase) }
    };
}

// --- Source-generated context ---
// The generator emits serialization code at compile time: no reflection,
// safe for trimming and Native AOT.
[JsonSourceGenerationOptions(
    PropertyNamingPolicy = JsonSourceGenerationPropertyNamingPolicy.CamelCase,
    PropertyNameCaseInsensitive = true,
    WriteIndented = true,
    DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull,
    ReadCommentHandling = JsonCommentHandling.Skip,
    AllowTrailingCommas = true,
    UseStringEnumConverter = true)]
[JsonSerializable(typeof(TaskItem))]
[JsonSerializable(typeof(List<TaskItem>))]
[JsonSerializable(typeof(User))]
internal partial class AppJsonContext : JsonSerializerContext { }

// --- Reflection pipeline ---
public static class ReflectionPipeline
{
    public static string Serialize(TaskItem task) =>
        JsonSerializer.Serialize(task, JsonConfig.Options);

    public static TaskItem? Deserialize(string json) =>
        JsonSerializer.Deserialize<TaskItem>(json, JsonConfig.Options);

    public static Task SerializeStreamAsync(TaskItem task, Stream stream) =>
        JsonSerializer.SerializeAsync(stream, task, JsonConfig.Options);
}

// --- Source-gen pipeline ---
public static class SourceGenPipeline
{
    public static string Serialize(TaskItem task) =>
        JsonSerializer.Serialize(task, AppJsonContext.Default.TaskItem);

    public static TaskItem? Deserialize(string json) =>
        JsonSerializer.Deserialize(json, AppJsonContext.Default.TaskItem);
}

// --- Program.cs (top-level statements) ---
var sample = new TaskItem(
    Id: 42,
    Title: "Refactor JSON layer",
    Description: null,
    AssigneeId: 7,
    DueDate: new DateTime(2025, 1, 31, 18, 0, 0, DateTimeKind.Utc),
    Status: TaskStatus.InProgress,
    Comments:
    [
        new(1, 7, "Started", DateTime.UtcNow),
        new(2, 7, "Need review", DateTime.UtcNow)
    ]);

string dirtyJson = """
    // external JSON with comments and mixed casing
    {
      "ID": 99,
      "Title": "External task",
      "DESCRIPTION": null,
      "assignee_id": 5,
      "due_date": "2025-02-14T12:00:00Z",
      "task_status": "in_progress",   // trailing comma + comment
      "comments": [
        { "id": 10, "author_id": 5, "text": "Imported", "created_at": "2025-01-01T00:00:00Z" }
      ]
    }
    """;

Console.WriteLine("=== Reflection serialize ===");
Console.WriteLine(ReflectionPipeline.Serialize(sample));

Console.WriteLine("=== Source-gen serialize ===");
Console.WriteLine(SourceGenPipeline.Serialize(sample));

Console.WriteLine("=== Deserialize dirty JSON (reflection) ===");
var r1 = ReflectionPipeline.Deserialize(dirtyJson);
Console.WriteLine($"{r1!.Id} {r1.Title} {r1.Status} comments={r1.Comments.Count}");

Console.WriteLine("=== Deserialize dirty JSON (source-gen) ===");
var r2 = SourceGenPipeline.Deserialize(dirtyJson);
Console.WriteLine($"{r2!.Id} {r2.Title} {r2.Status} comments={r2.Comments.Count}");

// --- Benchmarks ---
const int N = 10_000;
_ = ReflectionPipeline.Serialize(sample);   // warmup
_ = SourceGenPipeline.Serialize(sample);

var sw = Stopwatch.StartNew();
for (int i = 0; i < N; i++) _ = ReflectionPipeline.Serialize(sample);
sw.Stop();
long reflectionMs = sw.ElapsedMilliseconds;

sw.Restart();
for (int i = 0; i < N; i++) _ = SourceGenPipeline.Serialize(sample);
sw.Stop();
long sourceGenMs = sw.ElapsedMilliseconds;

Console.WriteLine($"Benchmark ({N} iterations):");
Console.WriteLine($"  reflection : {reflectionMs} ms");
Console.WriteLine($"  source-gen : {sourceGenMs} ms");
```

Line-by-line walk-through: the models are `record`s — this gives value semantics and a concise constructor syntax; `[property: JsonPropertyName(...)]` applies the attribute to the auto-generated property of the record (without `property:` the attribute would land on the constructor parameter, which is insufficient for `System.Text.Json`). External names are pinned to `snake_case`, modelling integration with Python/legacy APIs where snake_case is the norm. Options are created once via `new(JsonSerializerDefaults.Web)`: that constructor already turns on camelCase + case-insensitive + trailing commas, and we add `WriteIndented`, `WhenWritingNull`, `ReadCommentHandling = Skip`, and `JsonStringEnumConverter`. `JsonConfig.Options` is `static readonly`, i.e. a singleton; the serializer caches type metadata inside it, and reuse yields an order-of-magnitude speedup (a direct best practice from the lesson).

`AppJsonContext` is an `internal partial class : JsonSerializerContext`. `partial` is mandatory: the generator writes the other half of the class with the `JsonTypeInfo<T>` implementation for each `[JsonSerializable]`. In `[JsonSourceGenerationOptions]` we mirror the reflection settings: note there is no `JsonSerializerDefaults.Web` here — every setting is explicit, and instead of `JsonNamingPolicy.CamelCase` we use `JsonSourceGenerationPropertyNamingPolicy.CamelCase` (a source-gen-specific enum). `UseStringEnumConverter = true` replaces the `JsonStringEnumConverter` from the reflection path. `ReflectionPipeline` and `SourceGenPipeline` are symmetric: the same method names, different overloads of `JsonSerializer.Serialize`/`Deserialize`. For source-gen we pass `AppJsonContext.Default.TaskItem` (which is a `JsonTypeInfo<TaskItem>`), and the serializer uses pre-generated code with no reflection.

`Program.cs` uses top-level statements. The collection expression `[ new(...), new(...) ]` initializes a `List<Comment>` without an explicit `new List<Comment> { ... }`. The raw-string literal `"""..."""` lets us embed JSON with double quotes and even comments without escaping — this is the "dirty" payload. Warmup before benchmarks matters: the first reflection call builds the metadata cache, and without warmup the measurement is skewed. `Stopwatch` shows that source-gen is faster on a cold start (no reflection) and roughly on par or slightly faster when warm. Bonus check: `dotnet publish -r win-x64 -c Release` with `PublishAot` passes without IL3050/IL2026 precisely because the whole JSON path goes through `AppJsonContext`, and the trimmer strips nothing needed.

#### Going deeper (bonus)

1. **Reference cycles.** Add a nullable `ReplyToComment? Parent` field to `Comment`. Create two comments referencing each other and try to serialize. You will get `JsonException` "cycle detected". Fix it with `ReferenceHandler = ReferenceHandler.Preserve` (in both paths) and explain how `$id`/`$ref` change the payload. Then try `ReferenceHandler.IgnoreCycles` and compare.
2. **Streaming big data.** Generate a `List<TaskItem>` of 100,000 items and serialize it to a file via `SerializeAsync` into a `FileStream`. Measure peak memory (via `GC.GetTotalMemory`) and compare with synchronous `Serialize` to a string. Explain why streaming saves memory.
3. **Custom converter.** Write a `JsonConverter<DateTime>` (or `JsonConverter<DateOnly>`) that writes dates as `dd.MM.yyyy` for a Russian UI and reads several formats. Register it in `JsonConfig.Options` and in `[JsonSourceGenerationOptions]` (via `Converters = typeof(...)` in the source-gen variant).
4. **Full AOT build.** Turn on `PublishAot=true`, run `dotnet publish -r win-x64 -c Release`, measure the binary size with source-gen and without (by commenting out `AppJsonContext` and using reflection) — the latter should either fail with warnings or break at runtime. Record the output.

---

#### Чек-лист сдачи / Submission checklist

- [ ] Проект `TaskTracker.Json` собирается без warnings, `net8.0` + C# 12.
- [ ] `JsonSerializerOptions` — синглтон (`static readonly`), с комментарием-обоснованием.
- [ ] Включены camelCase, case-insensitive, `WhenWritingNull`, `ReadCommentHandling = Skip`, `AllowTrailingCommas`, `JsonStringEnumConverter`.
- [ ] Рефлексионный путь реализован, включая `SerializeAsync`.
- [ ] `AppJsonContext` объявлен, все типы зарегистрированы через `[JsonSerializable]`.
- [ ] Source-gen путь не выбрасывает `NotSupportedException`.
- [ ] «Грязный» JSON десериализуется обоими путями.
- [ ] `Program.cs` использует top-level statements, raw strings, collection expressions.
- [ ] Выводит JSON, поля распарсенного объекта и таблицу замеров `Stopwatch`.
- [ ] (Бонус) AOT-публикация проходит без IL3050/IL2026.
- [ ] Project `TaskTracker.Json` builds with no warnings, `net8.0` + C# 12.
- [ ] `JsonSerializerOptions` is a singleton (`static readonly`) with an explanatory comment.
- [ ] camelCase, case-insensitive, `WhenWritingNull`, `ReadCommentHandling = Skip`, `AllowTrailingCommas`, `JsonStringEnumConverter` are enabled.
- [ ] Reflection path is implemented, including `SerializeAsync`.
- [ ] `AppJsonContext` is declared; all types are registered with `[JsonSerializable]`.
- [ ] Source-gen path does not throw `NotSupportedException`.
- [ ] Dirty JSON deserializes through both paths.
- [ ] `Program.cs` uses top-level statements, raw strings, collection expressions.
- [ ] Prints JSON, parsed-object fields, and a `Stopwatch` benchmark table.
- [ ] (Bonus) AOT publish passes without IL3050/IL2026.

#### Ресурсы / Resources

- [Microsoft Learn — System.Text.Json overview — https://learn.microsoft.com/dotnet/standard/serialization/system-text-json/overview](https://learn.microsoft.com/dotnet/standard/serialization/system-text-json/overview)
- [Microsoft Learn — How to serialize and deserialize JSON — https://learn.microsoft.com/dotnet/standard/serialization/system-text-json/how-to](https://learn.microsoft.com/dotnet/standard/serialization/system-text-json/how-to)
- [Microsoft Learn — Source generation in System.Text.Json — https://learn.microsoft.com/dotnet/standard/serialization/system-text-json/source-generation](https://learn.microsoft.com/dotnet/standard/serialization/system-text-json/source-generation)
- [Microsoft Learn — JsonSerializerOptions — https://learn.microsoft.com/dotnet/api/system.text.json.jsonserializeroptions](https://learn.microsoft.com/dotnet/api/system.text.json.jsonserializeroptions)
- [Microsoft Learn — JsonSerializerContext — https://learn.microsoft.com/dotnet/api/system.text.json.serialization.jsonserializercontext](https://learn.microsoft.com/dotnet/api/system.text.json.serialization.jsonserializercontext)
- [Microsoft Learn — Prepare libraries for trimming — https://learn.microsoft.com/dotnet/core/deploying/trimming/prepare-libraries-for-trimming](https://learn.microsoft.com/dotnet/core/deploying/trimming/prepare-libraries-for-trimming)
- [Microsoft Learn — Native AOT deployment — https://learn.microsoft.com/dotnet/core/deploying/native-aot](https://learn.microsoft.com/dotnet/core/deploying/native-aot)
