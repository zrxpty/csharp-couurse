[⬆ К модулю M10](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M10-L05: System.Text.Json: сериализация/десериализация, опции, JsonSerializerContext / System.Text.Json: serialization, options, JsonSerializerContext

**Модуль / Module:** M10
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

`System.Text.Json` — это встроенный в .NET высокопроизводительный сериализатор JSON, появившийся в .NET Core 3.0 и ставший стандартом де-факто в .NET 8+. Он пришёл на смену `Newtonsoft.Json` в сценариях, где важны скорость, низкие аллокации и работа в обрезанных сборках (trimming) и Ahead-of-Time компиляции (Native AOT).

Представьте JSON как «упакованный чемодан» для поездки. Сериализация (`JsonSerializer.Serialize`) — это аккуратная укладка вещей в чемодан: каждый объект превращается в текст с полями, значениями и скобками. Десериализация (`JsonSerializer.Deserialize<T>`) — это распаковка: текст превращается обратно в живые объекты C#. Чемодан одинаков для всех (это формат JSON), а одежда — это ваши типы.

Ключевые операции:
- `JsonSerializer.Serialize(obj)` — объект → JSON-строка (или в `Stream`/`Utf8JsonWriter`).
- `JsonSerializer.Deserialize<T>(json)` — JSON-строка → объект типа `T`.

Однако «голые» вызовы используют настройки по умолчанию, которые часто не совпадают с тем, что ожидает веб-API. Например, по умолчанию имена свойств сериализуются как в C# (PascalCase: `FirstName`), а REST-API и JavaScript-клиенты обычно ждут `camelCase` (`firstName`). Чтобы это исправить, применяют `JsonSerializerOptions`:

```csharp
var options = new JsonSerializerOptions
{
    PropertyNamingPolicy = JsonNamingPolicy.CamelCase,
    PropertyNameCaseInsensitive = true,
    WriteIndented = true,
    DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull
};
```

- `PropertyNamingPolicy = JsonNamingPolicy.CamelCase` — имена свойств на выходе станут camelCase.
- `PropertyNameCaseInsensitive = true` — при десериализации регистр ключей игнорируется: `"firstname"` и `"FirstName"` попадут в одно свойство. Это критично для устойчивости к внешним API.
- `WriteIndented` — красивый отступ для человека (включайте только в отладке, это медленнее).
- `DefaultIgnoreCondition` — не писать `null`-свойства в JSON.

Создавать `JsonSerializerOptions` заново при каждом вызове **нельзя** — сериализатор кэширует метаданные внутри опций, и пересоздание убивает производительность. Используйте `JsonSerializerOptions` как синглтон: один экземпляр на всё приложение (в .NET 8 есть удобный `JsonSerializerOptions.Default`, доступный через `new JsonSerializerOptions(JsonSerializerDefaults.Web)` для типовых веб-настроек).

**Source generation и JsonSerializerContext (C# 12 / .NET 8).**

Классическая рефлексия, на которой работает `System.Text.Json`, «тяжёлая»: во время выполнения сериализатор строит кэш метаданных о типах через reflection. Это работает, но: (1) медленнее при первом вызове, (2) **несовместимо с trimming и Native AOT**, потому чтоtrimmer не видит, какие типы реально нужны, и может вырезать их метаданные.

Решение — **source generators**. Вы описываете типы, которые надо сериализовать, в partial-классе, унаследованном от `JsonSerializerContext`, и генератор кода **на этапе компиляции** создаёт готовый код сериализации без рефлексии:

```csharp
[JsonSourceGenerationOptions(
    PropertyNamingPolicy = JsonSourceGenerationPropertyNamingPolicy.CamelCase,
    PropertyNameCaseInsensitive = true)]
[JsonSerializable(typeof(User))]
[JsonSerializable(typeof(List<User>))]
internal partial class AppJsonContext : JsonSerializerContext { }
```

Теперь вы можете вызывать `JsonSerializer.Serialize(user, AppJsonContext.Default.User)` — это и быстрее, и безопасно для trim/AOT. Тот же контекст можно передавать как `JsonTypeInfo<T>` или просто `JsonSerializerContext`. Для ASP.NET Core достаточно зарегистрировать контекст в `JsonOptions` — и вся веб-сериализация переключится на source-gen.

Аналогия: рефлексия — это повар, который каждый раз читает рецепт в книге (медленно и книга занимает место). Source generation — это повар, который выучил рецепт наизусть заранее (быстро и книга не нужна, можно её выкинуть).

Практический вывод: для библиотек, консольных утилит с AOT и высоконагруженных сервисов — используйте `JsonSerializerContext`. Для прототипов и админ-интерфейсов достаточно рефлексии с кэшированным `JsonSerializerOptions`.

#### Theory (EN)

`System.Text.Json` is the built-in, high-performance JSON serializer shipped with .NET, introduced in .NET Core 3.0 and now the de facto standard in .NET 8+. It replaces `Newtonsoft.Json` in scenarios where speed, low allocations, trimming, and Native AOT compilation matter.

Think of JSON as a "packed suitcase" for a trip. Serialization (`JsonSerializer.Serialize`) is neatly folding clothes into the suitcase: every object becomes text with fields, values, and braces. Deserialization (`JsonSerializer.Deserialize<T>`) is unpacking: text becomes live C# objects again. The suitcase is the same for everyone (it is the JSON format); the clothes are your types.

Core operations:
- `JsonSerializer.Serialize(obj)` — object → JSON string (or into a `Stream` / `Utf8JsonWriter`).
- `JsonSerializer.Deserialize<T>(json)` — JSON string → object of type `T`.

Bare calls use default settings that often mismatch what a web API expects. By default, property names are serialized as in C# (PascalCase: `FirstName`), but REST APIs and JavaScript clients usually expect `camelCase` (`firstName`). To fix this, use `JsonSerializerOptions`:

```csharp
var options = new JsonSerializerOptions
{
    PropertyNamingPolicy = JsonNamingPolicy.CamelCase,
    PropertyNameCaseInsensitive = true,
    WriteIndented = true,
    DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull
};
```

- `PropertyNamingPolicy = JsonNamingPolicy.CamelCase` — output names become camelCase.
- `PropertyNameCaseInsensitive = true` — during deserialization the key case is ignored: `"firstname"` and `"FirstName"` map to the same property. This is critical for resilience against external APIs.
- `WriteIndented` — pretty indentation for humans (enable only while debugging; it is slower).
- `DefaultIgnoreCondition` — skip `null` properties in JSON output.

Never recreate `JsonSerializerOptions` per call — the serializer caches type metadata inside the options instance, and recreating it destroys performance. Treat `JsonSerializerOptions` as a singleton: one instance per application. In .NET 8 you can use `new JsonSerializerOptions(JsonSerializerDefaults.Web)` for typical web settings, or the global `JsonSerializerOptions.Default`.

**Source generation and JsonSerializerContext (C# 12 / .NET 8).**

The classic reflection that powers `System.Text.Json` is "heavy": at runtime the serializer builds a metadata cache about types via reflection. This works, but it is (1) slower on first call and (2) **incompatible with trimming and Native AOT**, because the trimmer cannot see which types are actually needed and may strip their metadata.

The answer is **source generators**. You declare the types to serialize in a partial class derived from `JsonSerializerContext`, and the code generator produces ready-to-use serialization code at compile time, with no reflection:

```csharp
[JsonSourceGenerationOptions(
    PropertyNamingPolicy = JsonSourceGenerationPropertyNamingPolicy.CamelCase,
    PropertyNameCaseInsensitive = true)]
[JsonSerializable(typeof(User))]
[JsonSerializable(typeof(List<User>))]
internal partial class AppJsonContext : JsonSerializerContext { }
```

Now you can call `JsonSerializer.Serialize(user, AppJsonContext.Default.User)` — both faster and trim/AOT-safe. The same context can be passed as `JsonTypeInfo<T>` or as a `JsonSerializerContext`. For ASP.NET Core, registering the context in `JsonOptions` switches the whole web pipeline to source-gen serialization.

Analogy: reflection is a cook who re-reads the recipe from a book every time (slow, and the book takes space). Source generation is a cook who memorized the recipe in advance (fast, and the book can be thrown away — which is exactly what trimming does).

Practical takeaway: for libraries, AOT console tools, and high-load services, use `JsonSerializerContext`. For prototypes and admin tools, plain reflection with a cached `JsonSerializerOptions` is enough.

#### Пример кода / Code Example

```csharp
// C# 12+ / .NET 8+ — System.Text.Json: reflection + source-gen demo
// Полный рабочий пример / Full working example

using System.Collections.Generic;
using System.Text.Json;
using System.Text.Json.Serialization;

namespace M10.L05.Demo;

// Модель данных / Data model
public record User(
    [property: JsonPropertyName("id")] int Id,
    [property: JsonPropertyName("first_name")] string FirstName,
    [property: JsonPropertyName("last_name")] string LastName,
    [property: JsonPropertyName("email")] string? Email);

public class JsonSerializerDemo
{
    // Кэшированные опции (синглтон!) / Cached options (singleton!)
    // JsonSerializerDefaults.Web = camelCase + case-insensitive + allow trailing commas
    private static readonly JsonSerializerOptions Options =
        new(JsonSerializerDefaults.Web)
        {
            WriteIndented = true,
            DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull
        };

    public static void Run()
    {
        var user = new User(42, "Иван", "Петров", null);
        var list = new List<User>
        {
            user,
            new User(7, "Anna", "Smith", "anna@example.com")
        };

        // --- Рефлексионная сериализация / Reflection-based serialization ---
        string json = JsonSerializer.Serialize(list, Options);
        // camelCase на выходе: [{"id":42,"first_name":"Иван",...}]
        // camelCase output: keys follow JsonNamingPolicy.CamelCase
        Console.WriteLine(json);

        // Десериализация (case-insensitive) / Deserialization (case-insensitive)
        string incoming =
            """{"ID":99,"FIRST_NAME":"Bob","LAST_NAME":"Doe","EMAIL":"bob@x.io"}""";
        var parsed = JsonSerializer.Deserialize<User>(incoming, Options);
        Console.WriteLine($"{parsed!.Id}: {parsed.FirstName} {parsed.LastName}");
        // 99: Bob Doe — регистр ключей не важен / key case does not matter

        // --- Source-gen сериализация / Source-gen serialization ---
        // Быстрее, без рефлексии, безопасно для trim/AOT
        // Faster, no reflection, trim/AOT-safe
        string genJson = JsonSerializer.Serialize(list, AppJsonContext.Default.ListUser);
        var genParsed = JsonSerializer.Deserialize(
            incoming, AppJsonContext.Default.User);
        Console.WriteLine(genJson);
        Console.WriteLine($"{genParsed!.Id}: {genParsed.FirstName}");
    }
}

// --- Source generator context (C# 12 / .NET 8) ---
// Генератор кода создаёт реализацию на этапе компиляции
// The code generator emits the implementation at compile time
[JsonSourceGenerationOptions(
    PropertyNamingPolicy = JsonSourceGenerationPropertyNamingPolicy.CamelCase,
    PropertyNameCaseInsensitive = true,
    WriteIndented = true,
    DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull)]
[JsonSerializable(typeof(User))]
[JsonSerializable(typeof(List<User>))]
internal partial class AppJsonContext : JsonSerializerContext { }
```

```xml
<!-- .csproj: включите source-generators для trim/AOT -->
<!-- .csproj: enable source generators for trim/AOT -->
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <PublishTrimmed>true</PublishTrimmed>
    <PublishAot>true</PublishAot>
  </PropertyGroup>
</Project>
```

#### Best Practices

- Кэшируйте `JsonSerializerOptions` как синглтон на всё приложение — никогда не создавайте новый экземпляр в горячем пути / Cache `JsonSerializerOptions` as a singleton across the app — never allocate a new instance on the hot path.
- Используйте `JsonSerializerDefaults.Web` для типовых веб-API: camelCase + case-insensitive + trailing commas / Use `JsonSerializerDefaults.Web` for typical web APIs.
- Для библиотек, AOT и trim — переключайтесь на `JsonSerializerContext` source-gen / For libraries, AOT and trim — switch to `JsonSerializerContext` source-gen.
- Применяйте `[JsonPropertyName]` или `JsonNamingPolicy` для явного управления именами, не полагайтесь на PascalCase по умолчанию / Use `[JsonPropertyName]` or `JsonNamingPolicy` to control names explicitly; do not rely on default PascalCase.
- Игнорируйте `null` через `JsonIgnoreCondition.WhenWritingNull`, чтобы не засорять payload / Ignore `null` via `JsonIgnoreCondition.WhenWritingNull` to keep payloads clean.
- Используйте `System.Text.Json` для потоков (`SerializeAsync`/`DeserializeAsync`) на больших данных, чтобы не загружать всё в память / Use streaming `SerializeAsync`/`DeserializeAsync` for large payloads to avoid loading everything into memory.

#### Частые ошибки / Common Mistakes

- Создание `new JsonSerializerOptions()` на каждый вызов → кэш метаданных не переиспользуется, производительность падает в десятки раз. Создайте один экземпляр и переиспользуйте / Creating `new JsonSerializerOptions()` per call → metadata cache is never reused, performance drops by orders of magnitude. Create one instance and reuse it.
- Ожидание, что PascalCase совпадёт с JavaScript-клиентом → ключи `FirstName` ломают фронтенд. Включайте `JsonNamingPolicy.CamelCase` / Expecting PascalCase to match a JavaScript client → `FirstName` keys break the frontend. Enable `JsonNamingPolicy.CamelCase`.
- Десериализация без `PropertyNameCaseInsensitive = true` → внешние API с `id` вместо `Id` молча отдают `null`. Включайте case-insensitive для внешних источников / Deserializing without `PropertyNameCaseInsensitive = true` → external APIs sending `id` instead of `Id` silently yield `null`. Enable case-insensitive for external sources.
- Забыли `[JsonSerializable(typeof(T))]` в `JsonSerializerContext` → runtime-исключение `NotSupportedException` при source-gen, потому что тип не зарегистрирован / Forgot `[JsonSerializable(typeof(T))]` on `JsonSerializerContext` → runtime `NotSupportedException` because the type is not registered.
- Использование рефлексионного пути в AOT-приложении без `JsonSerializerContext` → trimmer вырезает метаданные, сериализация падает в publish. Всегда используйте source-gen для AOT / Using the reflection path in an AOT app without `JsonSerializerContext` → trimmer strips metadata and serialization fails at publish. Always use source-gen for AOT.
- Сериализация циклических ссылок без `ReferenceHandler.IgnoreCycles` / `Preserve` → `JsonException` "cycle detected". Выбирайте стратегию обработки циклов явно / Serializing circular references without `ReferenceHandler.IgnoreCycles` / `Preserve` → `JsonException` "cycle detected". Pick a cycle strategy explicitly.
- Чтение JSON с комментариями при дефолтных опциях → `JsonException`. Включайте `ReadCommentHandling = JsonCommentHandling.Skip` / Reading JSON with comments under default options → `JsonException`. Enable `ReadCommentHandling = JsonCommentHandling.Skip`.

#### Чек-лист самопроверки / Self-check Checklist

- [ ] `JsonSerializerOptions` создан один раз и переиспользуется (синглтон) / `JsonSerializerOptions` is created once and reused (singleton).
- [ ] Включён `JsonNamingPolicy.CamelCase` для веб-API / `JsonNamingPolicy.CamelCase` is enabled for web APIs.
- [ ] Включён `PropertyNameCaseInsensitive = true` для внешних источников / `PropertyNameCaseInsensitive = true` is enabled for external sources.
- [ ] `null`-значения игнорируются через `DefaultIgnoreCondition` / `null` values are ignored via `DefaultIgnoreCondition`.
- [ ] Создан `JsonSerializerContext` с `[JsonSerializable]` для всех нужных типов / A `JsonSerializerContext` with `[JsonSerializable]` is created for all needed types.
- [ ] Source-gen путь проверен в `PublishAot` / `PublishTrimmed` сборке / The source-gen path is verified under `PublishAot` / `PublishTrimmed` build.
- [ ] Большие payload стримятся через `SerializeAsync` / `DeserializeAsync` / Large payloads stream via `SerializeAsync` / `DeserializeAsync`.
- [ ] Имена свойств зафиксированы `[JsonPropertyName]` или политикой, нет неявного PascalCase / Property names are pinned with `[JsonPropertyName]` or a policy; no implicit PascalCase.

#### Ресурсы / Resources

- [Microsoft Learn — System.Text.Json overview — https://learn.microsoft.com/dotnet/standard/serialization/system-text-json/overview](https://learn.microsoft.com/dotnet/standard/serialization/system-text-json/overview)
- [Microsoft Learn — How to serialize and deserialize JSON — https://learn.microsoft.com/dotnet/standard/serialization/system-text-json/how-to](https://learn.microsoft.com/dotnet/standard/serialization/system-text-json/how-to)
- [Microsoft Learn — Source generation in System.Text.Json — https://learn.microsoft.com/dotnet/standard/serialization/system-text-json/source-generation](https://learn.microsoft.com/dotnet/standard/serialization/system-text-json/source-generation)
- [Microsoft Learn — JsonSerializerOptions — https://learn.microsoft.com/dotnet/api/system.text.json.jsonserializeroptions](https://learn.microsoft.com/dotnet/api/system.text.json.jsonserializeroptions)
- [.NET trimming and serialization — https://learn.microsoft.com/dotnet/core/deploying/trimming/prepare-libraries-for-trimming](https://learn.microsoft.com/dotnet/core/deploying/trimming/prepare-libraries-for-trimming)

---

[⬆ К модулю M10](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
