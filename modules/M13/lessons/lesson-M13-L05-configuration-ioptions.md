[⬆ К модулю M13](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M13-L05: Конфигурация: appsettings.json, env vars, IOptions / Configuration: appsettings.json, env vars, IOptions

**Модуль / Module:** M13
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

Конфигурация в ASP.NET Core — это единый абстрактный слой, который собирает значения из множества источников и представляет их как один плоский словарь ключей. Главный вход — интерфейс `IConfiguration`. Под капотом работает построитель `ConfigurationBuilder`, к которому хост по умолчанию подключает несколько провайдеров в строгом порядке: `appsettings.json`, `appsettings.{Environment}.json`, переменные окружения (env vars), и аргументы командной строки. Порядок важен: каждый следующий провайдер **перекрывает** значения предыдущих. Это значит, что локальные настройки лежат в JSON-файле, а «секреты продакшена» приходят через переменные окружения и легко переопределяют файловые значения без правки кода.

Полезная аналогия — стеклянные слои. Каждый провайдер — прозрачная плёнка с надписями. Когда вы накладываете плёнки друг на друга и смотрите сверху, видна самая верхняя надпись для каждой клетки. Нижние плёнки всё ещё там, но их перекрыли. Так переменные окружения «заклеивают» значения из `appsettings.json`, не удаляя сам файл.

Ключи в конфигурации используют двоеточие как разделитель уровней: `"ConnectionStrings:Default"`, `"Jwt:Issuer"`. В переменных окружения двоеточие недопустимо, поэтому применяется двойное подчёркивание: `Jwt__Issuer`. Конфигурация умеет читать иерархию: секция `Jwt` с дочерними `Issuer`, `Audience`, `ExpiresMinutes` превращается в дерево. Метод `GetSection("Jwt")` возвращает подсекцию, а `["Jwt:Issuer"]` — конкретное значение.

Читать сырые строки через `IConfiguration` напрямую удобно, но не типобезопасно и рассыпает магические строки по коду. Современный подход — **строго типизированные опции** через `IOptions<T>`. Вы создаёте класс `JwtOptions` со свойствами, и конфигурация автоматически маппится в него. Регистрируется это так: `builder.Services.Configure<JwtOptions>(builder.Configuration.GetSection("Jwt"))`. После внедрения зависимостей вы получаете `IOptions<JwtOptions>` в любом сервисе и обращаетесь к `options.Value.Issuer` — с IntelliSense, проверкой типов и рефакторингом.

Семейство `IOptions` имеет три варианта со своими контрактами времени жизни. `IOptions<T>` — синглтон, значение фиксируется при запуске и не меняется; подходит для большинства случаев. `IOptionsSnapshot<T>` — per-request в scoped-сервисах: пересчитывается при каждом запросе, что позволяет видеть изменения в `appsettings.json` без перезапуска (полезно в разработке). `IOptionsMonitor<T>` — синглтон с пуш-обновлениями и событием `OnChange`: используется, когда долгоживущий сервис должен реагировать на изменения конфигурации в реальном времени. `IMonitor` — не отдельный интерфейс, а разговорное сокращение для `IOptionsMonitor<T>`.

Наконец, **валидация опций** защищает приложение от запуска с некорректной конфигурацией. Без валидации опечатка в имени ключа молча оставит свойство в `null`. Есть два механизма: атрибуты из `System.ComponentModel.DataAnnotations` (`[Required]`, `[Range]`) на свойствах класса и `Validate`/`ValidateDataAnnotations` при регистрации. Ключевое — `ValidateOnStart()`: оно запускает проверку при старте хоста и бросает исключение, если конфигурация невалидна. Приложение «не поднимется» с неправильными настройками — это лучше, чем падать посреди запроса в продакшене.

#### Theory (EN)

Configuration in ASP.NET Core is a single abstract layer that gathers values from many sources and exposes them as one flat dictionary of keys. The main entry point is the `IConfiguration` interface. Under the hood a `ConfigurationBuilder` runs, and the default host wires up several providers in a strict order: `appsettings.json`, `appsettings.{Environment}.json`, environment variables, and command-line arguments. Order matters: each later provider **overrides** values from earlier ones. That means local settings live in a JSON file, while production secrets arrive through environment variables and cleanly override file values without touching code.

A useful analogy is stacked transparent layers. Each provider is a clear film with writing on it. When you stack the films and look from above, the topmost writing shows for every cell. The lower films are still there, just covered. Environment variables “sticker over” values from `appsettings.json` without deleting the file.

Keys use a colon as the hierarchy separator: `"ConnectionStrings:Default"`, `"Jwt:Issuer"`. In environment variables colons are not allowed, so a double underscore is used instead: `Jwt__Issuer`. Configuration is hierarchical: a `Jwt` section with children `Issuer`, `Audience`, `ExpiresMinutes` forms a tree. `GetSection("Jwt")` returns a subsection; `["Jwt:Issuer"]` returns a single value.

Reading raw strings via `IConfiguration` directly works, but it is not type-safe and scatters magic strings through the codebase. The modern approach is **strongly typed options** via `IOptions<T>`. You declare a class `JwtOptions` with properties, and configuration maps into it automatically. Registration is a one-liner: `builder.Services.Configure<JwtOptions>(builder.Configuration.GetSection("Jwt"))`. Through dependency injection you then receive `IOptions<JwtOptions>` in any service and read `options.Value.Issuer` — with IntelliSense, type checking, and safe refactoring.

The `IOptions` family has three flavours with distinct lifetime contracts. `IOptions<T>` is a singleton: the value is fixed at startup and never changes; it fits the majority of cases. `IOptionsSnapshot<T>` is per-request inside scoped services: it is recomputed on every request, which lets you see `appsettings.json` edits without a restart (handy in development). `IOptionsMonitor<T>` is a singleton with push updates and an `OnChange` event: use it when a long-lived service must react to configuration changes in real time. “IMonitor” is not a separate interface — it is shorthand people use for `IOptionsMonitor<T>`.

Finally, **options validation** protects the app from starting with bad configuration. Without it, a typo in a key name silently leaves a property at `null`. Two mechanisms exist: attributes from `System.ComponentModel.DataAnnotations` (`[Required]`, `[Range]`) on the option class properties, and `Validate`/`ValidateDataAnnotations` calls at registration. The critical piece is `ValidateOnStart()`: it runs validation when the host starts and throws if configuration is invalid. The app refuses to boot with wrong settings — far better than crashing mid-request in production.

#### Пример кода / Code Example

```csharp
// Файл: appsettings.json / File: appsettings.json
// {
//   "Jwt": {
//     "Issuer": "https://api.example.com",
//     "Audience": "web-client",
//     "ExpiresMinutes": 60,
//     "SigningKey": ""
//   }
// }

using System.ComponentModel.DataAnnotations;
using Microsoft.Extensions.Options;

// Класс опций с валидацией через атрибуты / Options class with attribute validation
public sealed class JwtOptions
{
    [Required(AllowEmptyStrings = false, ErrorMessage = "Issuer обязателен / Issuer is required")]
    public string Issuer { get; init; } = string.Empty;

    [Required(AllowEmptyStrings = false)]
    public string Audience { get; init; } = string.Empty;

    [Range(1, 1440, ErrorMessage = "ExpiresMinutes должен быть 1..1440 / Must be 1..1440")]
    public int ExpiresMinutes { get; init; }

    [Required(AllowEmptyStrings = false)]
    [MinLength(16, ErrorMessage = "SigningKey слишком короткий / SigningKey too short")]
    public string SigningKey { get; init; } = string.Empty;
}

// Контракт сервиса, зависящего от опций / Service contract depending on options
public interface ITokenService
{
    string IssueToken(string subject);
}

// Сервис использует IOptionsMonitor, чтобы реагировать на изменения / Service uses IOptionsMonitor for live updates
public sealed class TokenService : ITokenService
{
    private readonly IOptionsMonitor<JwtOptions> _options;
    public TokenService(IOptionsMonitor<JwtOptions> options) => _options = options;

    public string IssueToken(string subject)
    {
        // Текущее значение всегда свежее при push-обновлении / Current value stays fresh via push updates
        JwtOptions current = _options.CurrentValue;
        return $"{current.Issuer}|{current.Audience}|{current.ExpiresMinutes}m|{subject}";
    }
}

// Регистрация в Program.cs / Registration in Program.cs
public static class Program
{
    public static void Main(string[] args)
    {
        WebApplicationBuilder builder = WebApplication.CreateBuilder(args);

        // Configure + ValidateDataAnnotations + ValidateOnStart: проверяем при старте
        // Configure + validation attributes + fail-fast on startup
        builder.Services
            .AddOptions<JwtOptions>()
            .Bind(builder.Configuration.GetSection("Jwt"))
            .ValidateDataAnnotations()
            .Validate(o => o.Issuer.StartsWith("https://"), "Issuer должен быть https / Issuer must be https")
            .ValidateOnStart();

        // Для сценария с обновлениями регистрируем IOptionsMonitor автоматически через ITokenService
        // IOptionsMonitor<T> is resolvable automatically; register the service
        builder.Services.AddSingleton<ITokenService, TokenService>();

        WebApplication app = builder.Build();

        // Маршрут для демонстрации / Demo endpoint
        app.MapGet("/token/{subject}", (string subject, ITokenService svc) => svc.IssueToken(subject));

        app.Run();
    }
}

// Использование IOptionsSnapshot в scoped-сервисе (пересчёт на каждый запрос)
// IOptionsSnapshot in a scoped service (recomputed per request)
public sealed class ReportService
{
    private readonly JwtOptions _options;
    public ReportService(IOptionsSnapshot<JwtOptions> snapshot) => _options = snapshot.Value;
    public string Describe() => $"Issuer={_options.Issuer}, Expires={_options.ExpiresMinutes}m";
}

// Чтение сырой конфигурации через IConfiguration (когда нет класса опций)
// Reading raw config via IConfiguration (when no options class exists)
public sealed class RawConfigReader
{
    private readonly IConfiguration _config;
    public RawConfigReader(IConfiguration config) => _config = config;

    public string? Connection =>
        _config.GetConnectionString("Default"); // = Configuration.GetSection("ConnectionStrings")["Default"]
}
```

#### Best Practices

- Выносите каждую логическую секцию в отдельный класс опций (`JwtOptions`, `SmtpOptions`, `CorsOptions`) и регистрируйте через `AddOptions<T>().Bind(...)` — это даёт типобезопасность и поддержку валидации.
- Use `ValidateOnStart()` for every critical options class so the app fails fast at boot instead of breaking mid-request.
- Не храните секреты в `appsettings.json`. Секреты — в env vars, Azure Key Vault, User Secrets (Development) или Docker/K8s secrets.
- Keep secrets out of `appsettings.json`. Use env vars, Azure Key Vault, User Secrets in Development, or Docker/Kubernetes secrets.
- Используйте `IOptionsSnapshot<T>` в scoped-сервисах, `IOptionsMonitor<T>` в синглтонах, `IOptions<T>` — когда значение гарантированно неизменно.
- Name option classes with the `Options` suffix and keep them `sealed` with `init`-only setters for immutability.
- Применяйте `GetSection` вместо длинных двоеточий в строках: читаемость выше и меньше опечаток.

#### Частые ошибки / Common Mistakes

- Опечатка в имени секции при `Bind` → класс опций остаётся пустым; включите `ValidateOnStart()` и обязательные атрибуты, чтобы это вскрылось сразу.
- A typo in the section name during `Bind` → the options object stays empty; enable `ValidateOnStart()` and required attributes to surface it immediately.
- Использование `IOptionsSnapshot<T>` в синглтоне → исключение о несоответствии времени жизни; используйте `IOptionsMonitor<T>` для синглтонов.
- Using `IOptionsSnapshot<T>` inside a singleton → lifetime mismatch exception; use `IOptionsMonitor<T>` for singletons.
- Забыли `ValidateOnStart()` → валидация запускается только при первом обращении к `Value`, и ошибка всплывёт в рантайме.
- Forgot `ValidateOnStart()` → validation runs only on first `.Value` access, so errors surface at runtime instead of startup.
- Передача `Jwt__Issuer` через env var с одинарным подчёркиванием `Jwt_Issuer` → не маппится в иерархию; используйте строго двойное подчёркивание `Jwt__Issuer`.
- Relying on `IConfiguration[...]` everywhere instead of typed options → magic strings, no refactoring support, no validation.

#### Чек-лист самопроверки / Self-check Checklist

- [ ] Классы опций созданы для каждой секции с `init`-сеттерами и помечены `sealed`.
- [ ] Option classes created per section with `init` setters and marked `sealed`.
- [ ] Опции зарегистрированы через `AddOptions<T>().Bind(section).ValidateDataAnnotations().ValidateOnStart()`.
- [ ] Options registered via `AddOptions<T>().Bind(section).ValidateDataAnnotations().ValidateOnStart()`.
- [ ] Секреты подаются через env vars / User Secrets, а не лежат в `appsettings.json`.
- [ ] Secrets come from env vars / User Secrets, not from `appsettings.json`.
- [ ] Выбран правильный вариант: `IOptions` для неизменных, `IOptionsSnapshot` для scoped, `IOptionsMonitor` для синглтонов с обновлениями.
- [ ] The right variant is chosen: `IOptions` for immutable, `IOptionsSnapshot` for scoped, `IOptionsMonitor` for singletons with updates.
- [ ] Приложение падает при старте, если конфигурация невалидна (проверено вручную).
- [ ] The app fails at startup when configuration is invalid (verified manually).
- [ ] Двойное подчёркивание `__` используется в env vars для вложенных ключей.
- [ ] Double underscore `__` is used in env vars for nested keys.

#### Ресурсы / Resources
- [Microsoft Learn — https://learn.microsoft.com/aspnet/core/fundamentals/configuration/](https://learn.microsoft.com/aspnet/core/fundamentals/configuration/)
- [Options pattern in .NET — https://learn.microsoft.com/dotnet/core/extensions/options](https://learn.microsoft.com/dotnet/core/extensions/options)
- [Options validation — https://learn.microsoft.com/aspnet/core/fundamentals/configuration/options](https://learn.microsoft.com/aspnet/core/fundamentals/configuration/options)

---

[⬆ К модулю M13](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
