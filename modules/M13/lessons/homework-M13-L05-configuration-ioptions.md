---
[← К уроку M13-L05](lesson-M13-L05-configuration-ioptions.md) | [⬆ К модулю M13](../README.md) | [Предыдущее ДЗ ←](homework-M13-L04-di-lifetimes.md) | [Следующее ДЗ →](homework-M13-L06-logging-ilogger.md)
---

### Домашнее задание M13-L05: Конфигурация: appsettings.json, env vars, IOptions / Homework M13-L05: Configuration: appsettings.json, env vars, IOptions

**Урок / Lesson:** M13-L05
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 4/5
**Цель / Goal:** (RU) Научиться собирать конфигурацию из нескольких источников (appsettings.json, appsettings.{Environment}.json, переменные окружения, аргументы командной строки), маппить секции в строго типизированные классы опций через `IOptions<T>` / `IOptionsSnapshot<T>` / `IOptionsMonitor<T>`, валидировать опции атрибутами DataAnnotations и пользовательскими делегатами `Validate`, а также включать fail-fast через `ValidateOnStart()`. (EN) Learn to assemble configuration from multiple sources (appsettings.json, appsettings.{Environment}.json, environment variables, command-line arguments), bind sections into strongly typed options classes via `IOptions<T>` / `IOptionsSnapshot<T>` / `IOptionsMonitor<T>`, validate options with DataAnnotations attributes and custom `Validate` delegates, and enable fail-fast behaviour through `ValidateOnStart()`.

#### Связь с уроком / Connection to the lesson
(RU) Урок показывает, что `IConfiguration` — это плоский словарь, в который провайдеры складывают значения в строгом порядке, и что каждый следующий источник перекрывает предыдущий. ДЗ закрепляет именно этот механизм: вы наблюдаете переопределение значений из `appsettings.json` переменными окружения, превращаете сырые строки в типобезопасные классы опций и защищаете запуск валидацией. (EN) The lesson shows that `IConfiguration` is a flat dictionary populated by providers in a strict order, each later source overriding earlier ones. This homework cements exactly that mechanism: you observe `appsettings.json` values being overridden by environment variables, turn raw strings into type-safe options classes, and protect startup with validation.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация

Вы разрабатываете мини-сервис `ConfigLab`, который выдаёт JWT-подобные токены для демо-клиентов и формирует короткие отчёты о текущей конфигурации. Сервис должен работать одинаково корректно в трёх средах: локальная разработка (`Development`), staging-сервер (`Staging`) и «продакшен-подобный» контур (`Production`). В каждой среде значения внешних параметров (URL эмитента, срок жизни токена, ключ подписи, лимиты запросов) разные, но код менять нельзя — должна меняться только конфигурация. Это классическая ситуация реальных проектов: один и тот же бинарник ездит по средам за счёт переменных окружения и `appsettings.{Environment}.json`.

Главная боль, которую вы должны устранить, — магические строки `configuration["Jwt:Issuer"]`, размазанные по сервисам. Они не дают IntelliSense, ломаются при рефакторинге и молча возвращают `null` при опечатке. Современный подход — строго типизированные опции: класс `JwtOptions` со свойствами `init`, регистрация через `AddOptions<JwtOptions>().Bind(section)`, и инъекция `IOptionsMonitor<JwtOptions>` туда, где нужно живое обновление. Дополнительно вы должны гарантировать, что сервис не поднимется с битой конфигурацией: например, если забыли задать `SigningKey` в продакшене, приложение обязано упасть при старте, а не при первом запросе пользователя. За это отвечает `ValidateOnStart()` в связке с атрибутами `[Required]`, `[Range]`, `[MinLength]` и кастомными делегатами `Validate`. Наконец, вы научитесь выбирать правильный вариант из семейства `IOptions` в зависимости от времени жизни сервиса: синглтону нужен `IOptionsMonitor`, scoped-сервису — `IOptionsSnapshot`, неизменяемому значению — `IOptions`.

#### Что нужно сделать (пошагово)

1. Создайте решение и проект. В терминале выполните:
   ```bash
   dotnet new sln -n ConfigLab
   dotnet new web -n ConfigLab.Api -o ConfigLab.Api
   dotnet sln add ConfigLab.Api/ConfigLab.Api.csproj
   cd ConfigLab.Api
   ```
   Должны появиться файлы `ConfigLab.sln`, `ConfigLab.Api/Program.cs`, `ConfigLab.Api/ConfigLab.Api.csproj`, `ConfigLab.Api/appsettings.json` и `ConfigLab.Api/Properties/launchSettings.json`. Ожидаемая версия SDK — .NET 8 (`dotnet --version` должно показать `8.x.x`).

2. Подготовьте базовый `appsettings.json`. Откройте файл и приведите его к виду:
   ```json
   {
     "Logging": { "LogLevel": { "Default": "Information" } },
     "Jwt": {
       "Issuer": "https://localhost:5001",
       "Audience": "web-client",
       "ExpiresMinutes": 60,
       "SigningKey": ""
     },
     "RateLimiting": {
       "RequestsPerMinute": 100,
       "Enabled": true
     }
   }
   ```
   Обратите внимание: `SigningKey` пустой — это намеренно, чтобы показать работу валидации.

3. Добавьте файл `appsettings.Development.json` со значениями, перекрывающими базовые для локальной разработки:
   ```json
   {
     "Jwt": { "ExpiresMinutes": 15, "SigningKey": "dev-secret-key-0123456789" },
     "RateLimiting": { "RequestsPerMinute": 1000 }
   }
   ```
   И `appsettings.Production.json` с боевой конфигурацией без секрета (секрет придёт через env var):
   ```json
   {
     "Jwt": { "Issuer": "https://api.configlab.io", "ExpiresMinutes": 30 }
   }
   ```

4. Создайте классы опций `JwtOptions` и `RateLimitingOptions` в папке `Options/`. Они должны быть `sealed`, со свойствами `init` и атрибутами валидации. Для `JwtOptions` добавьте `[Required]`, `[Range(1,1440)]`, `[MinLength(16)]` и кастомную проверку `Issuer.StartsWith("https://")`. Для `RateLimitingOptions` — `[Range(0, 10000)]` и булевый флаг `Enabled`.

5. Реализуйте сервисы. `TokenService` (синглтон) должен зависеть от `IOptionsMonitor<JwtOptions>` и в методе `IssueToken(string subject)` возвращать строку вида `iss|aud|exp|subject`, читая `_options.CurrentValue`. `ReportService` (scoped) должен зависеть от `IOptionsSnapshot<RateLimitingOptions>` и возвращать описание лимитов. Добавьте минимальный `RawConfigReader`, зависящий от `IConfiguration`, который через `GetConnectionString("Default")` возвращает строку подключения — это демонстрирует чтение сырой конфигурации без класса опций.

6. Зарегистрируйте всё в `Program.cs` через `AddOptions<T>().Bind(section).ValidateDataAnnotations().Validate(...).ValidateOnStart()`. Подключите `AddSingleton<ITokenService, TokenService>()` и `AddScoped<ReportService>()`. Добавьте endpoint `GET /token/{subject}` и `GET /report`.

7. Прогоните локально:
   ```bash
   dotnet run --project ConfigLab.Api
   ```
   Ожидаемый вывод при `Development`: приложение стартует, потому что `appsettings.Development.json` даёт непустой `SigningKey`. Запрос `GET /token/alice` вернёт что-то вроде `https://localhost:5001|web-client|15m|alice`.

8. Симулируйте продакшен через переменные окружения (Windows PowerShell):
   ```powershell
   $env:ASPNETCORE_ENVIRONMENT="Production"
   $env:Jwt__SigningKey="prod-super-secret-key-0123456789"
   dotnet run --project ConfigLab.Api
   ```
   Ожидаемый вывод: запуск успешен, потому что env var `Jwt__SigningKey` перекрыла пустое значение файла. Запрос `GET /token/alice` вернёт `https://api.configlab.io|web-client|30m|alice`.

9. Проверьте fail-fast. Уберите env var и запустите без `SigningKey` в `Production`:
   ```powershell
   Remove-Item Env:Jwt__SigningKey
   $env:ASPNETCORE_ENVIRONMENT="Production"
   dotnet run --project ConfigLab.Api
   ```
   Ожидаемый результат: приложение падает при старте с исключением `OptionsValidationException` и сообщением про обязательный `SigningKey`. Это и есть цель валидации.

10. Добавьте unit-тест на валидацию: проект `ConfigLab.Api.Tests` (xUnit) с тестом, который строит `IConfiguration` из словаря, биндит его в `JwtOptions` через `Configure`, и проверяет, что пустой `SigningKey` приводит к ошибке валидации. Используйте `Microsoft.Extensions.Options.ConfigurationExtensions` и `Microsoft.Extensions.Options.DataAnnotations`.

#### Требования к решению

- Проект должен компилироваться без предупреждений на .NET 8, C# 12, с `<Nullable>enable</Nullable>` и `<ImplicitUsings>enable</ImplicitUsings>`.
- Все классы опций — `sealed`, со свойствами `init`, с `Options`-суффиксом в имени (`JwtOptions`, `RateLimitingOptions`).
- Регистрация опций — строго через `AddOptions<T>().Bind(builder.Configuration.GetSection("X"))` с цепочкой `.ValidateDataAnnotations().Validate(...).ValidateOnStart()`. Использование `Configure<T>(section)` допустимо, но в этом ДЗ требуется именно `AddOptions` для единого стиля и поддержки валидации.
- `TokenService` — синглтон, использует `IOptionsMonitor<JwtOptions>` и читает `CurrentValue` внутри метода, а не в конструкторе (иначе обновления не подхватятся).
- `ReportService` — scoped, использует `IOptionsSnapshot<RateLimitingOptions>`, значение фиксируется в конструкторе (таков контракт snapshot — оно стабильно в рамках запроса).
- `RawConfigReader` использует `IConfiguration.GetConnectionString(...)` и не имеет класса опций (демонстрация альтернативы).
- `Program.cs` — top-level statements, без `Main`, без `class Program`.
- Все секреты (`SigningKey`) подаются только через env var или User Secrets; в `appsettings.json` значение остаётся пустым.
- Endpoint `/token/{subject}` возвращает сгенерированную строку; endpoint `/report` возвращает описание лимитов.
- Должны быть видны три источника перекрытия: файловый → `appsettings.{Environment}.json` → env vars. Аргументы командной строки опциональны.

#### Тонкости и подводные камни

- **Разделитель в env vars.** Двоеточие `Jwt:Issuer` в переменных окружения запрещено на многих платформах, поэтому конфигурация ожидает двойное подчёркивание: `Jwt__Issuer`. Одинарное подчёркивание `Jwt_Issuer` не создаст иерархию — значение останется «висящим» и не попадёт в `JwtOptions`. Это одна из самых частых ошибок, и она молчаливая: класс опций просто остаётся с `null`/`default`, и лишь `ValidateOnStart()` спасает.
- **Порядок провайдеров фиксирован.** Хост добавляет `appsettings.json` → `appsettings.{Env}.json` → env vars → command line. Позже = важнее. Не пытайтесь «переиграть» порядок вручную через `ConfigurationBuilder` в minimal API — дефолтный `WebApplication.CreateBuilder` уже всё настроил.
- **IOptionsSnapshot нельзя в синглтон.** Если внедрить `IOptionsSnapshot<T>` в синглтон-сервис, DI бросит `InvalidOperationException` о несоответствии времени жизни. Для синглтонов, которым нужны живые обновления, — `IOptionsMonitor<T>`.
- **CurrentValue vs Value.** У `IOptionsMonitor` есть `CurrentValue` (обновляется) и `CurrentValue` всегда свежее. У `IOptionsSnapshot` — только `Value`, и оно фиксируется на момент инъекции в рамках запроса. У `IOptions` — `Value`, зафиксированное при старте. Если в синглтоне прочитать `IOptionsMonitor.CurrentValue` один раз в конструкторе и сохранить в поле — обновления работать не будут.
- **ValidateOnStart обязателен.** Без него валидация запускается только при первом обращении к `.Value`, то есть на первом запросе — это позднее падение. С `ValidateOnStart()` хост проверяет опции при `Build()`/`Run()` и валится сразу.
- **Bind vs Configure.** `AddOptions<T>().Bind(section)` и `Configure<T>(section)` внешне похожи, но первый возвращает `OptionsBuilder<T>`, на который можно навесить `.ValidateDataAnnotations()` и `.ValidateOnStart()`. Для валидации используйте `AddOptions`.
- **init-сеттеры и Bind.** `Bind` использует reflection и умеет писать в `init`-свойства через special handling. Не делайте свойства с приватным `set` — Bind их не заполнит. `init` — правильный выбор.
- **GetSection не бросает.** `GetSection("Missing")` возвращает пустую секцию, а не `null`. Поэтому опечатка в имени секции = пустой класс опций = скрытая ошибка. Спасает валидация.
- **User Secrets в Development.** Для локальной разработки секреты удобнее держать в User Secrets (`dotnet user-secrets init`), а не в env vars. Но формат ключей тот же — иерархия через двоеточие в JSON, и `__` в env-именах.

#### Критерии приёмки

- [ ] Решение `ConfigLab.sln` компилируется без ошибок и предупреждений на .NET 8 / C# 12.
- [ ] В `appsettings.json` `SigningKey` пустой; секреты подаются через env vars/User Secrets.
- [ ] Есть `appsettings.Development.json` и `appsettings.Production.json` с перекрывающими значениями.
- [ ] `JwtOptions` и `RateLimitingOptions` — `sealed`, `init`, с атрибутами DataAnnotations.
- [ ] Регистрация через `AddOptions<T>().Bind(...).ValidateDataAnnotations().Validate(...).ValidateOnStart()`.
- [ ] `TokenService` — синглтон с `IOptionsMonitor<JwtOptions>`, читает `CurrentValue` в методе.
- [ ] `ReportService` — scoped с `IOptionsSnapshot<RateLimitingOptions>`, фиксирует `Value` в конструкторе.
- [ ] `RawConfigReader` использует `IConfiguration.GetConnectionString(...)`.
- [ ] `Program.cs` — top-level statements, без `Main`.
- [ ] Endpoint `/token/{subject}` возвращает корректную строку с текущей конфигурацией.
- [ ] В `Development` запуск успешен; `GET /token/alice` отражает значение из `appsettings.Development.json`.
- [ ] В `Production` с env var `Jwt__SigningKey` запуск успешен и `Issuer` взят из `appsettings.Production.json`.
- [ ] В `Production` без `SigningKey` приложение падает при старте с `OptionsValidationException`.
- [ ] Двойное подчёркивание `__` используется в env var (проверено вручную).
- [ ] Есть unit-тест на валидацию `JwtOptions` с пустым `SigningKey`.

#### Подсказки (без прямого ответа)

- Вспомните аналогию из урока про «стеклянные плёнки»: какой источник лежит выше всех? Как это объясняет поведение env var?
- Для `IOptionsMonitor` ответ на вопрос «где читать `CurrentValue`» лежит в разнице между моментом инъекции и моментом использования.
- Если валидация «не срабатывает» при старте — проверьте, что вы вызываете `.ValidateOnStart()`, а не только `.ValidateDataAnnotations()`.
- Для unit-теста соберите `ConfigurationBuilder` вручную с `AddInMemoryCollection(dict)` и используйте `OptionsBuilder` через `services.AddOptions<T>().Bind(config.GetSection("Jwt"))`.
- Не путайте `GetConnectionString("Default")` с `GetSection("ConnectionStrings")["Default"]` — это одно и то же, но первый читаемее.

#### Эталонное решение (разбор)

```csharp
// Файл: Options/JwtOptions.cs
using System.ComponentModel.DataAnnotations;

namespace ConfigLab.Api.Options;

// Строго типизированные опции для секции "Jwt" / Typed options for "Jwt" section
public sealed class JwtOptions
{
    [Required(AllowEmptyStrings = false, ErrorMessage = "Issuer обязателен / Issuer is required")]
    public string Issuer { get; init; } = string.Empty;

    [Required(AllowEmptyStrings = false)]
    public string Audience { get; init; } = string.Empty;

    [Range(1, 1440, ErrorMessage = "ExpiresMinutes должен быть 1..1440 / Must be 1..1440")]
    public int ExpiresMinutes { get; init; }

    [Required(AllowEmptyStrings = false)]
    [MinLength(16, ErrorMessage = "SigningKey слишком короткий (минимум 16) / SigningKey too short (min 16)")]
    public string SigningKey { get; init; } = string.Empty;
}

// Файл: Options/RateLimitingOptions.cs
using System.ComponentModel.DataAnnotations;

namespace ConfigLab.Api.Options;

public sealed class RateLimitingOptions
{
    [Range(0, 10000, ErrorMessage = "RequestsPerMinute должен быть 0..10000 / Must be 0..10000")]
    public int RequestsPerMinute { get; init; }

    public bool Enabled { get; init; }
}

// Файл: Services/TokenService.cs
using ConfigLab.Api.Options;
using Microsoft.Extensions.Options;

namespace ConfigLab.Api.Services;

public interface ITokenService
{
    string IssueToken(string subject);
}

// Синглтон с живым обновлением через IOptionsMonitor / Singleton with live updates via IOptionsMonitor
public sealed class TokenService : ITokenService
{
    private readonly IOptionsMonitor<JwtOptions> _options;
    public TokenService(IOptionsMonitor<JwtOptions> options) => _options = options;

    public string IssueToken(string subject)
    {
        JwtOptions current = _options.CurrentValue; // читаем ВНУТРИ метода — иначе обновления не видны
        return $"{current.Issuer}|{current.Audience}|{current.ExpiresMinutes}m|{subject}";
    }
}

// Файл: Services/ReportService.cs
using ConfigLab.Api.Options;
using Microsoft.Extensions.Options;

namespace ConfigLab.Api.Services;

// Scoped-сервис: IOptionsSnapshot стабилен в рамках запроса / Scoped: snapshot is stable per request
public sealed class ReportService
{
    private readonly RateLimitingOptions _options;
    public ReportService(IOptionsSnapshot<RateLimitingOptions> snapshot) => _options = snapshot.Value;

    public string Describe() =>
        $"RateLimit Enabled={_options.Enabled}, PerMinute={_options.RequestsPerMinute}";
}

// Файл: Services/RawConfigReader.cs
using Microsoft.Extensions.Configuration;

namespace ConfigLab.Api.Services;

// Демонстрация чтения сырой конфигурации без класса опций / Raw config read without an options class
public sealed class RawConfigReader
{
    private readonly IConfiguration _config;
    public RawConfigReader(IConfiguration config) => _config = config;

    public string? Connection => _config.GetConnectionString("Default");
}

// Файл: Program.cs
using ConfigLab.Api.Options;
using ConfigLab.Api.Services;

WebApplicationBuilder builder = WebApplication.CreateBuilder(args);

// Регистрируем JwtOptions: Bind + DataAnnotations + кастомная проверка + fail-fast
builder.Services.AddOptions<JwtOptions>()
    .Bind(builder.Configuration.GetSection("Jwt"))
    .ValidateDataAnnotations()
    .Validate(o => o.Issuer.StartsWith("https://", StringComparison.OrdinalIgnoreCase),
               "Issuer должен начинаться с https:// / Issuer must start with https://")
    .ValidateOnStart();

builder.Services.AddOptions<RateLimitingOptions>()
    .Bind(builder.Configuration.GetSection("RateLimiting"))
    .ValidateDataAnnotations()
    .ValidateOnStart();

builder.Services.AddSingleton<ITokenService, TokenService>();
builder.Services.AddScoped<ReportService>();
builder.Services.AddSingleton<RawConfigReader>();

WebApplication app = builder.Build();

app.MapGet("/token/{subject}", (string subject, ITokenService svc) => svc.IssueToken(subject));
app.MapGet("/report", (ReportService rs) => rs.Describe());
app.MapGet("/conn", (RawConfigReader r) => r.Connection ?? "(no connection string)");

app.Run();

// Файл: tests/ConfigLab.Api.Tests/JwtOptionsValidationTests.cs
using System.Collections.Generic;
using ConfigLab.Api.Options;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Options;
using Xunit;

namespace ConfigLab.Api.Tests;

public class JwtOptionsValidationTests
{
    private static IServiceProvider BuildProvider(IDictionary<string, string?> data)
    {
        IConfiguration config = new ConfigurationBuilder()
            .AddInMemoryCollection(data)
            .Build();

        ServiceCollection services = new();
        services.AddOptions<JwtOptions>()
            .Bind(config.GetSection("Jwt"))
            .ValidateDataAnnotations()
            .Validate(o => o.Issuer.StartsWith("https://"), "https only")
            .ValidateOnStart();
        return services.BuildServiceProvider();
    }

    [Fact]
    public void Empty_signing_key_fails_validation()
    {
        IServiceProvider provider = BuildProvider(new Dictionary<string, string?>
        {
            ["Jwt:Issuer"] = "https://localhost",
            ["Jwt:Audience"] = "web",
            ["Jwt:ExpiresMinutes"] = "10",
            ["Jwt:SigningKey"] = "", // пустой — должно упасть
        });

        IOptions<JwtOptions> options = provider.GetRequiredService<IOptions<JwtOptions>>();
        Assert.Throws<OptionsValidationException>(() => options.Value);
    }
}
```

Разбор по строкам. Класс `JwtOptions` помечен `sealed` — это закрывает наследование и даёт компилятору дополнительные оптимизации; `init`-сеттеры гарантируют, что после создания объект неизменяем, но `Bind` всё ещё может его заполнить через reflection-путь для `init`. Атрибуты `[Required(AllowEmptyStrings = false)]` ловят пустые строки, `[Range(1,1440)]` ограничивает срок жизни токена сутками, `[MinLength(16)]` защищает от слишком короткого ключа. `ErrorMessage` двуязычный для удобства. `RateLimitingOptions` устроен проще — числовой лимит и булев флаг.

В `TokenService` инъектируется `IOptionsMonitor<JwtOptions>`, а не `IOptions<JwtOptions>`, потому что сервис — синглтон и должен реагировать на изменения конфигурации без перезапуска (например, ротация `SigningKey`). Чтение `_options.CurrentValue` происходит **внутри метода** `IssueToken` — это критично: если сохранить значение в поле в конструкторе, обновления никогда не подхватятся. `ReportService`, напротив, scoped, и здесь уместен `IOptionsSnapshot<T>` — значение фиксируется один раз на запрос и остаётся стабильным, что удобно для логики отчёта. `RawConfigReader` намеренно показывает «старый» путь через `IConfiguration.GetConnectionString(...)`, который эквивалентен `GetSection("ConnectionStrings")["Default"]`, но читаемее.

Регистрация в `Program.cs` использует `AddOptions<T>()` вместо `Configure<T>()`, потому что только `AddOptions` возвращает `OptionsBuilder<T>`, на который можно навесить `.ValidateDataAnnotations()`, `.Validate(...)` и `.ValidateOnStart()`. Цепочка выстроена именно так, как в уроке: сначала `Bind`, потом DataAnnotations, потом кастомная проверка `Issuer.StartsWith("https://")`, и в конце `.ValidateOnStart()` — он-то и превращает валидацию из «ленивой» в «fail-fast при старте хоста». Без `ValidateOnStart` приложение бы поднялось и упало только на первом запросе, что в продакшене означает падение посреди рабочего цикла. Top-level statements соответствуют C# 12 / .NET 8 — нет ни `Main`, ни `class Program`, точка входа синтезируется компилятором.

Тест `Empty_signing_key_fails_validation` собирает `IConfiguration` через `AddInMemoryCollection`, регистрирует опции с тем же набором валидаторов и проверяет, что обращение к `.Value` бросает `OptionsValidationException`, если `SigningKey` пуст. Обратите внимание: в тесте `ValidateOnStart()` вызывается явно, но в самом тесте проверяется ленивая валидация через `.Value` — это нормально, потому что в тесте нет хоста; в реальном приложении за fail-fast отвечает именно старт хоста.

#### Задания на углубление (бонус)

1. Подключите **User Secrets** для `Development`: `dotnet user-secrets init`, `dotnet user-secrets set "Jwt:SigningKey" "dev-secret-from-secrets"`. Убедитесь, что User Secrets перекрывают `appsettings.Development.json`, и объясните, почему это безопаснее, чем хранить ключ в файле.
2. Реализуйте **`IOptionsChangeTokenSource<T>`** или используйте `AddJsonFile(..., reloadOnChange: true)` (он включён по умолчанию в `WebApplication.CreateBuilder`) и проверьте, что изменение `appsettings.json` в рантайме подхватывается `IOptionsMonitor` без перезапуска. Добавьте логирование в `_options.OnChange(o => ...)`.
3. Добавьте **третий вариант валидации** — `Validate` с комплексным правилом: «если `RateLimiting.Enabled = true`, то `RequestsPerMinute` должен быть не меньше 1». Напишите отдельный unit-тест на это правило.
4. Перепишите `RawConfigReader` на класс опций `ConnectionStringsOptions` с привязкой всей секции `ConnectionStrings` и сравните подходы по типобезопасности.

---

## Statement in English / Постановка на английском

#### Context & motivation

You are building a small service called `ConfigLab` that issues JWT-like tokens for demo clients and produces short reports about the current configuration. The service must behave correctly across three environments: local development (`Development`), a staging server (`Staging`), and a production-like contour (`Production`). In each environment the external parameters differ — issuer URL, token lifetime, signing key, rate-limit thresholds — but the code must not change. Only configuration should change. This is the classic reality of production projects: the same binary travels across environments thanks to environment variables and `appsettings.{Environment}.json`.

The main pain you must eliminate is the spread of magic strings like `configuration["Jwt:Issuer"]` across services. They give no IntelliSense, break under refactoring, and silently return `null` on a typo. The modern approach is strongly typed options: a `JwtOptions` class with `init` properties, registered through `AddOptions<JwtOptions>().Bind(section)`, with `IOptionsMonitor<JwtOptions>` injected where live updates are required. On top of that you must guarantee that the service refuses to boot with broken configuration — if someone forgot to set `SigningKey` in production, the app must crash at startup, not on the first user request. That is the job of `ValidateOnStart()` together with `[Required]`, `[Range]`, `[MinLength]` attributes and custom `Validate` delegates. Finally, you will learn to pick the right member of the `IOptions` family depending on the service lifetime: a singleton needs `IOptionsMonitor`, a scoped service wants `IOptionsSnapshot`, and an immutable value is fine with `IOptions`.

#### What to do step by step

1. Create the solution and project. In the terminal run:
   ```bash
   dotnet new sln -n ConfigLab
   dotnet new web -n ConfigLab.Api -o ConfigLab.Api
   dotnet sln add ConfigLab.Api/ConfigLab.Api.csproj
   cd ConfigLab.Api
   ```
   You should see `ConfigLab.sln`, `ConfigLab.Api/Program.cs`, `ConfigLab.Api/ConfigLab.Api.csproj`, `ConfigLab.Api/appsettings.json`, and `ConfigLab.Api/Properties/launchSettings.json`. The expected SDK version is .NET 8 (`dotnet --version` should report `8.x.x`).

2. Prepare the base `appsettings.json`. Open the file and bring it to this shape:
   ```json
   {
     "Logging": { "LogLevel": { "Default": "Information" } },
     "Jwt": {
       "Issuer": "https://localhost:5001",
       "Audience": "web-client",
       "ExpiresMinutes": 60,
       "SigningKey": ""
     },
     "RateLimiting": {
       "RequestsPerMinute": 100,
       "Enabled": true
     }
   }
   ```
   Note that `SigningKey` is empty on purpose — to demonstrate validation behaviour.

3. Add `appsettings.Development.json` with values that override the base file for local development:
   ```json
   {
     "Jwt": { "ExpiresMinutes": 15, "SigningKey": "dev-secret-key-0123456789" },
     "RateLimiting": { "RequestsPerMinute": 1000 }
   }
   ```
   And `appsettings.Production.json` with production values but without the secret (the secret will arrive via env var):
   ```json
   {
     "Jwt": { "Issuer": "https://api.configlab.io", "ExpiresMinutes": 30 }
   }
   ```

4. Create the option classes `JwtOptions` and `RateLimitingOptions` under `Options/`. They must be `sealed`, with `init` properties and validation attributes. For `JwtOptions` add `[Required]`, `[Range(1,1440)]`, `[MinLength(16)]`, and a custom check `Issuer.StartsWith("https://")`. For `RateLimitingOptions` add `[Range(0, 10000)]` and a boolean `Enabled` flag.

5. Implement the services. `TokenService` (a singleton) must depend on `IOptionsMonitor<JwtOptions>` and, in `IssueToken(string subject)`, return a string like `iss|aud|exp|subject` reading `_options.CurrentValue`. `ReportService` (scoped) must depend on `IOptionsSnapshot<RateLimitingOptions>` and return a description of the limits. Add a minimal `RawConfigReader` that depends on `IConfiguration` and exposes the connection string through `GetConnectionString("Default")` — this demonstrates raw config reads without an options class.

6. Register everything in `Program.cs` via `AddOptions<T>().Bind(section).ValidateDataAnnotations().Validate(...).ValidateOnStart()`. Add `AddSingleton<ITokenService, TokenService>()` and `AddScoped<ReportService>()`. Add endpoints `GET /token/{subject}` and `GET /report`.

7. Run locally:
   ```bash
   dotnet run --project ConfigLab.Api
   ```
   Expected output under `Development`: the app starts because `appsettings.Development.json` supplies a non-empty `SigningKey`. A request `GET /token/alice` returns something like `https://localhost:5001|web-client|15m|alice`.

8. Simulate production with environment variables (Windows PowerShell):
   ```powershell
   $env:ASPNETCORE_ENVIRONMENT="Production"
   $env:Jwt__SigningKey="prod-super-secret-key-0123456789"
   dotnet run --project ConfigLab.Api
   ```
   Expected: the app starts because the env var `Jwt__SigningKey` overrides the empty file value. `GET /token/alice` returns `https://api.configlab.io|web-client|30m|alice`.

9. Verify fail-fast. Remove the env var and run without `SigningKey` in `Production`:
   ```powershell
   Remove-Item Env:Jwt__SigningKey
   $env:ASPNETCORE_ENVIRONMENT="Production"
   dotnet run --project ConfigLab.Api
   ```
   Expected: the app crashes at startup with `OptionsValidationException` and a message about the required `SigningKey`. That is the goal of validation.

10. Add a unit test for validation: a `ConfigLab.Api.Tests` project (xUnit) that builds `IConfiguration` from a dictionary, binds it into `JwtOptions` via `Configure`, and verifies that an empty `SigningKey` triggers a validation failure. Use `Microsoft.Extensions.Options.ConfigurationExtensions` and `Microsoft.Extensions.Options.DataAnnotations`.

#### Requirements

- The project must compile without warnings on .NET 8, C# 12, with `<Nullable>enable</Nullable>` and `<ImplicitUsings>enable</ImplicitUsings>`.
- All options classes are `sealed`, with `init` properties, and carry the `Options` suffix (`JwtOptions`, `RateLimitingOptions`).
- Options registration uses `AddOptions<T>().Bind(builder.Configuration.GetSection("X"))` with the chain `.ValidateDataAnnotations().Validate(...).ValidateOnStart()`. Using `Configure<T>(section)` is acceptable in general, but this homework requires `AddOptions` for a uniform style and validation support.
- `TokenService` is a singleton that uses `IOptionsMonitor<JwtOptions>` and reads `CurrentValue` inside the method, not in the constructor (otherwise updates are not picked up).
- `ReportService` is scoped and uses `IOptionsSnapshot<RateLimitingOptions>`; the value is fixed in the constructor (that is the snapshot contract — stable within a request).
- `RawConfigReader` uses `IConfiguration.GetConnectionString(...)` and has no options class (an alternative demonstration).
- `Program.cs` uses top-level statements, no `Main`, no `class Program`.
- All secrets (`SigningKey`) come only through env vars or User Secrets; in `appsettings.json` the value stays empty.
- Endpoint `/token/{subject}` returns the generated string; endpoint `/report` returns the limit description.
- Three override sources must be visible: file → `appsettings.{Environment}.json` → env vars. Command-line arguments are optional.

#### Pitfalls

- **Separator in env vars.** The colon `Jwt:Issuer` is forbidden in environment variables on many platforms, so configuration expects a double underscore: `Jwt__Issuer`. A single underscore `Jwt_Issuer` does not build a hierarchy — the value just hangs and never reaches `JwtOptions`. This is one of the most common and silent mistakes: the options class simply stays `null`/`default`, and only `ValidateOnStart()` saves the day.
- **Provider order is fixed.** The host adds `appsettings.json` → `appsettings.{Env}.json` → env vars → command line. Later means more important. Do not try to “reorder” providers manually with `ConfigurationBuilder` inside a minimal API — the default `WebApplication.CreateBuilder` already wired everything correctly.
- **IOptionsSnapshot is forbidden in singletons.** Injecting `IOptionsSnapshot<T>` into a singleton service makes DI throw `InvalidOperationException` about a lifetime mismatch. For singletons that need live updates use `IOptionsMonitor<T>`.
- **CurrentValue vs Value.** `IOptionsMonitor` exposes `CurrentValue` (live). `IOptionsSnapshot` exposes `Value`, fixed at injection time for the duration of the request. `IOptions` exposes `Value`, fixed at startup. If a singleton reads `IOptionsMonitor.CurrentValue` once in the constructor and stores it in a field, updates will never be seen.
- **ValidateOnStart is mandatory.** Without it validation runs only on the first `.Value` access — i.e., on the first request — which is a late failure. With `ValidateOnStart()` the host validates options at `Build()`/`Run()` and crashes immediately.
- **Bind vs Configure.** `AddOptions<T>().Bind(section)` and `Configure<T>(section)` look similar, but the former returns `OptionsBuilder<T>` on which you can chain `.ValidateDataAnnotations()` and `.ValidateOnStart()`. For validation use `AddOptions`.
- **init setters and Bind.** `Bind` uses reflection and can write to `init` properties through special handling. Do not use private `set` — `Bind` will not fill it. `init` is the right choice.
- **GetSection never throws.** `GetSection("Missing")` returns an empty section, not `null`. So a typo in the section name yields an empty options object — a hidden bug. Validation is the safety net.
- **User Secrets in Development.** For local development secrets are easier to keep in User Secrets (`dotnet user-secrets init`) than in env vars. The key format is the same — a hierarchy via colon in JSON and `__` in env names.

#### Acceptance criteria

- [ ] The `ConfigLab.sln` solution compiles without errors or warnings on .NET 8 / C# 12.
- [ ] In `appsettings.json` `SigningKey` is empty; secrets arrive via env vars/User Secrets.
- [ ] `appsettings.Development.json` and `appsettings.Production.json` exist with overriding values.
- [ ] `JwtOptions` and `RateLimitingOptions` are `sealed`, `init`, with DataAnnotations attributes.
- [ ] Registration uses `AddOptions<T>().Bind(...).ValidateDataAnnotations().Validate(...).ValidateOnStart()`.
- [ ] `TokenService` is a singleton with `IOptionsMonitor<JwtOptions>`, reading `CurrentValue` in the method.
- [ ] `ReportService` is scoped with `IOptionsSnapshot<RateLimitingOptions>`, fixing `Value` in the constructor.
- [ ] `RawConfigReader` uses `IConfiguration.GetConnectionString(...)`.
- [ ] `Program.cs` uses top-level statements, no `Main`.
- [ ] Endpoint `/token/{subject}` returns the correct string for the current configuration.
- [ ] Under `Development` startup succeeds; `GET /token/alice` reflects the value from `appsettings.Development.json`.
- [ ] Under `Production` with env var `Jwt__SigningKey` startup succeeds and `Issuer` comes from `appsettings.Production.json`.
- [ ] Under `Production` without `SigningKey` the app crashes at startup with `OptionsValidationException`.
- [ ] The double underscore `__` is used in the env var (verified manually).
- [ ] A unit test for `JwtOptions` validation with an empty `SigningKey` exists.

#### Hints

- Recall the lesson’s “transparent films” analogy: which source sits on top of all the others? How does that explain env var behaviour?
- For `IOptionsMonitor`, the answer to “where do I read `CurrentValue`” lives in the difference between injection time and use time.
- If validation does not fire at startup, check that you call `.ValidateOnStart()`, not only `.ValidateDataAnnotations()`.
- For the unit test, build a `ConfigurationBuilder` by hand with `AddInMemoryCollection(dict)` and register options via `services.AddOptions<T>().Bind(config.GetSection("Jwt"))`.
- Do not confuse `GetConnectionString("Default")` with `GetSection("ConnectionStrings")["Default"]` — they are the same, but the first is more readable.

#### Reference solution walk-through

```csharp
// File: Options/JwtOptions.cs
using System.ComponentModel.DataAnnotations;

namespace ConfigLab.Api.Options;

// Strongly typed options for the "Jwt" section
public sealed class JwtOptions
{
    [Required(AllowEmptyStrings = false, ErrorMessage = "Issuer is required")]
    public string Issuer { get; init; } = string.Empty;

    [Required(AllowEmptyStrings = false)]
    public string Audience { get; init; } = string.Empty;

    [Range(1, 1440, ErrorMessage = "ExpiresMinutes must be 1..1440")]
    public int ExpiresMinutes { get; init; }

    [Required(AllowEmptyStrings = false)]
    [MinLength(16, ErrorMessage = "SigningKey too short (min 16)")]
    public string SigningKey { get; init; } = string.Empty;
}

// File: Options/RateLimitingOptions.cs
using System.ComponentModel.DataAnnotations;

namespace ConfigLab.Api.Options;

public sealed class RateLimitingOptions
{
    [Range(0, 10000, ErrorMessage = "RequestsPerMinute must be 0..10000")]
    public int RequestsPerMinute { get; init; }

    public bool Enabled { get; init; }
}

// File: Services/TokenService.cs
using ConfigLab.Api.Options;
using Microsoft.Extensions.Options;

namespace ConfigLab.Api.Services;

public interface ITokenService
{
    string IssueToken(string subject);
}

// Singleton with live updates via IOptionsMonitor
public sealed class TokenService : ITokenService
{
    private readonly IOptionsMonitor<JwtOptions> _options;
    public TokenService(IOptionsMonitor<JwtOptions> options) => _options = options;

    public string IssueToken(string subject)
    {
        JwtOptions current = _options.CurrentValue; // read INSIDE the method — otherwise updates are lost
        return $"{current.Issuer}|{current.Audience}|{current.ExpiresMinutes}m|{subject}";
    }
}

// File: Services/ReportService.cs
using ConfigLab.Api.Options;
using Microsoft.Extensions.Options;

namespace ConfigLab.Api.Services;

// Scoped service: IOptionsSnapshot is stable within a request
public sealed class ReportService
{
    private readonly RateLimitingOptions _options;
    public ReportService(IOptionsSnapshot<RateLimitingOptions> snapshot) => _options = snapshot.Value;

    public string Describe() =>
        $"RateLimit Enabled={_options.Enabled}, PerMinute={_options.RequestsPerMinute}";
}

// File: Services/RawConfigReader.cs
using Microsoft.Extensions.Configuration;

namespace ConfigLab.Api.Services;

// Demonstrates raw config reads without an options class
public sealed class RawConfigReader
{
    private readonly IConfiguration _config;
    public RawConfigReader(IConfiguration config) => _config = config;

    public string? Connection => _config.GetConnectionString("Default");
}

// File: Program.cs
using ConfigLab.Api.Options;
using ConfigLab.Api.Services;

WebApplicationBuilder builder = WebApplication.CreateBuilder(args);

// Register JwtOptions: Bind + DataAnnotations + custom check + fail-fast
builder.Services.AddOptions<JwtOptions>()
    .Bind(builder.Configuration.GetSection("Jwt"))
    .ValidateDataAnnotations()
    .Validate(o => o.Issuer.StartsWith("https://", StringComparison.OrdinalIgnoreCase),
               "Issuer must start with https://")
    .ValidateOnStart();

builder.Services.AddOptions<RateLimitingOptions>()
    .Bind(builder.Configuration.GetSection("RateLimiting"))
    .ValidateDataAnnotations()
    .ValidateOnStart();

builder.Services.AddSingleton<ITokenService, TokenService>();
builder.Services.AddScoped<ReportService>();
builder.Services.AddSingleton<RawConfigReader>();

WebApplication app = builder.Build();

app.MapGet("/token/{subject}", (string subject, ITokenService svc) => svc.IssueToken(subject));
app.MapGet("/report", (ReportService rs) => rs.Describe());
app.MapGet("/conn", (RawConfigReader r) => r.Connection ?? "(no connection string)");

app.Run();

// File: tests/ConfigLab.Api.Tests/JwtOptionsValidationTests.cs
using System.Collections.Generic;
using ConfigLab.Api.Options;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Options;
using Xunit;

namespace ConfigLab.Api.Tests;

public class JwtOptionsValidationTests
{
    private static IServiceProvider BuildProvider(IDictionary<string, string?> data)
    {
        IConfiguration config = new ConfigurationBuilder()
            .AddInMemoryCollection(data)
            .Build();

        ServiceCollection services = new();
        services.AddOptions<JwtOptions>()
            .Bind(config.GetSection("Jwt"))
            .ValidateDataAnnotations()
            .Validate(o => o.Issuer.StartsWith("https://"), "https only")
            .ValidateOnStart();
        return services.BuildServiceProvider();
    }

    [Fact]
    public void Empty_signing_key_fails_validation()
    {
        IServiceProvider provider = BuildProvider(new Dictionary<string, string?>
        {
            ["Jwt:Issuer"] = "https://localhost",
            ["Jwt:Audience"] = "web",
            ["Jwt:ExpiresMinutes"] = "10",
            ["Jwt:SigningKey"] = "", // empty — must fail
        });

        IOptions<JwtOptions> options = provider.GetRequiredService<IOptions<JwtOptions>>();
        Assert.Throws<OptionsValidationException>(() => options.Value);
    }
}
```

The `JwtOptions` class is marked `sealed` to close inheritance and give the compiler extra optimisation opportunities; the `init` setters guarantee the object is immutable after creation, while `Bind` can still populate it through the reflection path that supports `init`. The attributes `[Required(AllowEmptyStrings = false)]` catch empty strings, `[Range(1,1440)]` caps the token lifetime at one day, and `[MinLength(16)]` guards against a too-short signing key. The `ErrorMessage` is bilingual for convenience. `RateLimitingOptions` is simpler: a numeric limit and a boolean flag.

`TokenService` receives `IOptionsMonitor<JwtOptions>` instead of `IOptions<JwtOptions>` because the service is a singleton and must react to configuration changes without a restart (for example, `SigningKey` rotation). Reading `_options.CurrentValue` happens **inside** the `IssueToken` method — this is critical: if you stored the value in a field inside the constructor, updates would never be picked up. `ReportService`, by contrast, is scoped, and here `IOptionsSnapshot<T>` is the right fit — the value is fixed once per request and stays stable, which is convenient for report logic. `RawConfigReader` intentionally shows the “old” path through `IConfiguration.GetConnectionString(...)`, which is equivalent to `GetSection("ConnectionStrings")["Default"]` but reads better.

Registration in `Program.cs` uses `AddOptions<T>()` rather than `Configure<T>()` because only `AddOptions` returns `OptionsBuilder<T>`, on which you can chain `.ValidateDataAnnotations()`, `.Validate(...)`, and `.ValidateOnStart()`. The chain mirrors the lesson: first `Bind`, then DataAnnotations, then a custom check `Issuer.StartsWith("https://")`, and finally `.ValidateOnStart()` — this is the call that turns validation from “lazy” into “fail-fast at host startup”. Without `ValidateOnStart` the app would boot and only fail on the first request, which in production means a crash in the middle of a working cycle. Top-level statements match C# 12 / .NET 8 — there is no `Main` and no `class Program`; the compiler synthesises the entry point.

The test `Empty_signing_key_fails_validation` builds an `IConfiguration` via `AddInMemoryCollection`, registers the options with the same set of validators, and checks that accessing `.Value` throws `OptionsValidationException` when `SigningKey` is empty. Note that in the test `ValidateOnStart()` is called explicitly, but the test itself checks lazy validation through `.Value` — that is fine because there is no host in the test; in the real application the host startup is what guarantees fail-fast behaviour.

#### Going deeper (bonus)

1. Enable **User Secrets** for `Development`: `dotnet user-secrets init`, `dotnet user-secrets set "Jwt:SigningKey" "dev-secret-from-secrets"`. Verify that User Secrets override `appsettings.Development.json` and explain why this is safer than keeping the key in a file.
2. Implement an **`IOptionsChangeTokenSource<T>`** or rely on `AddJsonFile(..., reloadOnChange: true)` (enabled by default in `WebApplication.CreateBuilder`) and confirm that editing `appsettings.json` at runtime is picked up by `IOptionsMonitor` without a restart. Add logging in `_options.OnChange(o => ...)`.
3. Add a **third validation rule** via `Validate`: “if `RateLimiting.Enabled = true`, then `RequestsPerMinute` must be at least 1.” Write a dedicated unit test for that rule.
4. Rewrite `RawConfigReader` as an options class `ConnectionStringsOptions` that binds the whole `ConnectionStrings` section, and compare the two approaches for type safety.

---

#### Чек-лист сдачи / Submission checklist
- [ ] (RU) Решение `ConfigLab.sln` компилируется на .NET 8 / C# 12 без предупреждений.
- [ ] (RU) Секреты подаются через env vars / User Secrets, а не лежат в `appsettings.json`.
- [ ] (RU) Есть `appsettings.Development.json` и `appsettings.Production.json`.
- [ ] (RU) Опции зарегистрированы через `AddOptions<T>().Bind(...).ValidateDataAnnotations().Validate(...).ValidateOnStart()`.
- [ ] (RU) Синглтон использует `IOptionsMonitor`, scoped — `IOptionsSnapshot`.
- [ ] (RU) Приложение падает при старте с `OptionsValidationException`, если конфигурация невалидна.
- [ ] (RU) Unit-тест на валидацию присутствует и зелёный.
- [ ] (EN) The `ConfigLab.sln` solution compiles on .NET 8 / C# 12 without warnings.
- [ ] (EN) Secrets come from env vars / User Secrets, not from `appsettings.json`.
- [ ] (EN) `appsettings.Development.json` and `appsettings.Production.json` exist.
- [ ] (EN) Options are registered via `AddOptions<T>().Bind(...).ValidateDataAnnotations().Validate(...).ValidateOnStart()`.
- [ ] (EN) The singleton uses `IOptionsMonitor`, the scoped service uses `IOptionsSnapshot`.
- [ ] (EN) The app crashes at startup with `OptionsValidationException` when configuration is invalid.
- [ ] (EN) A validation unit test exists and is green.

#### Ресурсы / Resources
- [Microsoft Learn — Configuration in ASP.NET Core](https://learn.microsoft.com/aspnet/core/fundamentals/configuration/)
- [Options pattern in .NET](https://learn.microsoft.com/dotnet/core/extensions/options)
- [Options validation](https://learn.microsoft.com/aspnet/core/fundamentals/configuration/options)
- [Safe storage of app secrets in development](https://learn.microsoft.com/aspnet/core/security/app-secrets)
- [Environment variables in .NET](https://learn.microsoft.com/dotnet/core/tools/dotnet-environment-variables)

---
[← К уроку M13-L05](lesson-M13-L05-configuration-ioptions.md) | [⬆ К модулю M13](../README.md) | [Предыдущее ДЗ ←](homework-M13-L04-di-lifetimes.md) | [Следующее ДЗ →](homework-M13-L06-logging-ilogger.md)
