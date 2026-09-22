---
[← К уроку M13-L04](lesson-M13-L04-di-lifetimes.md) | [⬆ К модулю M13](../README.md) | [Следующее ДЗ →](homework-M13-L05-configuration-ioptions.md)
---

### Домашнее задание M13-L04: Встроенный DI: AddTransient/Scoped/Singleton, lifetimes / Homework M13-L04: Built-in DI: AddTransient/Scoped/Singleton, lifetimes

**Урок / Lesson:** M13-L04
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 4/5
**Цель / Goal:** (RU) Научиться регистрировать сервисы с тремя временами жизни (Transient/Scoped/Singleton), наблюдать их поведение через Guid-идентификаторы экземпляров, безопасно разрешать Scoped-зависимости из долгоживущих потребителей (BackgroundService/Singleton) через `IServiceScopeFactory`, включать `ValidateScopes`/`ValidateOnBuild` для раннего обнаружения captive dependency и применять современные возможности C# 12 / .NET 8 (top-level statements, collection expressions, raw string literals, pattern matching). (EN) Learn to register services with the three built-in lifetimes (Transient/Scoped/Singleton), observe their behaviour through instance Guid identifiers, safely resolve Scoped dependencies from long-lived consumers (BackgroundService/Singleton) through `IServiceScopeFactory`, enable `ValidateScopes`/`ValidateOnBuild` to detect captive dependencies early, and apply modern C# 12 / .NET 8 features (top-level statements, collection expressions, raw string literals, pattern matching).

#### Связь с уроком / Connection to the lesson
(RU) Урок описывает контейнер `Microsoft.Extensions.DependencyInjection` как «бухгалтера», который создаёт объекты и управляет их жизненным циклом через три lifetime: Transient (новый экземпляр каждый раз), Scoped (один на scope/HTTP-запрос) и Singleton (один на всё приложение). Домашнее задание закрепляет эти концепции на практике: вы зарегистрируете сервисы всеми тремя способами, воспроизведёте captive dependency и научитесь его избегать через `IServiceScopeFactory`, а также включите валидацию контейнера, чтобы ловить ошибки lifetimes ещё на старте. (EN) The lesson describes the `Microsoft.Extensions.DependencyInjection` container as a "bookkeeper" that creates objects and manages their lifecycles through three lifetimes: Transient (a new instance every time), Scoped (one per scope / HTTP request) and Singleton (one for the whole application). This homework cements those concepts in practice: you will register services all three ways, reproduce a captive dependency and learn to avoid it through `IServiceScopeFactory`, and enable container validation to catch lifetime errors at startup.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация

Вы поддерживаете внутреннее консольное приложение `LifetimeLab` для образовательного отдела компании. Приложение моделирует типичный backend: есть лёгкий stateless-форматтер (превращает доменные события в строки для лога), есть репозиторий, привязанный к «транзакции» (имитация `DbContext`), и есть глобальный кэш конфигурации, разделяемый всеми потоками. Кроме того, в приложении работает фоновый `BackgroundService`, который раз в несколько секунд формирует отчёт на основе свежих данных репозитория — типичный сценарий, где долгоживущий потребитель должен безопасно обращаться к Scoped-сервису.

Исторически приложение писали разные люди, и в нём успели накопиться классические ошибки DI: кто-то внедрил `IRepository` напрямую в singleton-воркер (captive dependency), кто-то забыл `using` на `CreateScope()`, кто-то попробовал сделать конструктор `async Task`. В Dev-окружении всё «работало», потому что валидация контейнера была выключена, а в Production стали всплывать странные баги: один и тот же `IRepository` жил сутками, отдавая всем потокам один и тот же Guid, и отчёты фонового воркера «зависали» на устаревших данных. Ваша задача — привести DI в порядок: правильно подобрать lifetime для каждого сервиса, доказать корректность через Guid-эксперименты, устранить captive dependency с помощью `IServiceScopeFactory`, включить `ValidateScopes`/`ValidateOnBuild` и убедиться, что контейнер сам откажется стартовать при ошибке lifetimes. Это даёт вам интуицию, которую невозможно получить из теории: вы собственными глазами увидите, как один и тот же класс ведёт себя по-разному в зависимости от регистрации, и научитесь читать диагностические исключения контейнера.

#### Что нужно сделать (пошагово)

1. Создайте новый проект: `dotnet new console -n LifetimeLab -o LifetimeLab --framework net8.0`. Перейдите в каталог `LifetimeLab` и откройте проект. Убедитесь, что в `LifetimeLab.csproj` указаны `<TargetFramework>net8.0</TargetFramework>` и `<ImplicitUsings>enable</ImplicitUsings>`, `<Nullable>enable</Nullable>`.
2. Добавьте пакет хостинга и логирования: `dotnet add package Microsoft.Extensions.Hosting`. Этого достаточно — `Microsoft.Extensions.DependencyInjection` и `Microsoft.Extensions.Logging` подтянутся транзитивно. Выполните `dotnet build` и убедитесь, что сборка успешна (`Build succeeded`, 0 Warning, 0 Error).
3. В `Program.cs` используйте top-level statements. Создайте `Host.CreateDefaultBuilder(args)`, в `ConfigureServices` вызовите расширение `AddAppServices()` (его вы напишете отдельным статическим классом `ServiceRegistration`). Включите валидацию: `.UseDefaultServiceProvider((_, options) => { options.ValidateScopes = true; options.ValidateOnBuild = true; })`. Соберите хост через `.Build()`.
4. Определите три интерфейса: `IEventFormatter` (метод `string Format(string evt, Guid correlationId)`), `IRepository` (свойство `Guid InstanceId`, метод `string Snapshot()`), `IConfigCache` (свойство `Guid InstanceId`, метод `string Get(string key)`). Все реализации — `sealed` классы C# 12, каждый хранит `private readonly Guid _id = Guid.NewGuid();` и выводит его в `Snapshot()`/`Get()`, чтобы вы могли наблюдать lifetime глазами.
5. Реализации: `TransientFormatter : IEventFormatter`, `ScopedRepository : IRepository`, `SingletonConfigCache : IConfigCache`. Зарегистрируйте их в `AddAppServices` соответственно `AddTransient`, `AddScoped`, `AddSingleton` — по интерфейсу, а не по конкретному классу (best practice из урока).
6. Напишите `ReportWorker : BackgroundService`. В конструктор внедрите **только** `IServiceScopeFactory` (НЕ `IRepository` — иначе captive dependency). В `ExecuteAsync` организуйте цикл `while (!stoppingToken.IsCancellationRequested)`, внутри которого через `await using AsyncServiceScope scope = _scopeFactory.CreateAsyncScope();` получайте свежий `IRepository` и печатайте отчёт. Делайте 2–3 итерации с `Task.Delay(TimeSpan.FromMilliseconds(800), stoppingToken)`, после чего токен отмены завершит цикл (используйте `CancellationTokenSource` с `CancelAfter` в `Program.cs`).
7. В `Program.cs` после `host.StartAsync()` запустите демонстрационную функцию `Diagnostics.RunAsync(host.Services)`, которая: (а) дважды запросит `IEventFormatter` и покажет два разных Guid (Transient); (б) дважды запросит `IConfigCache` и покажет одинаковый Guid (Singleton); (в) создаст два scope через `CreateAsyncScope()` и внутри каждого дважды запросит `IRepository`, продемонстрировав «один экземпляр внутри scope, разные — между scope» (Scoped). Все выводы оформите через `ILogger<T>` с категориями — это приближает задание к реальному backend.
8. В отдельном методе воспроизведите captive dependency «в негатив»: временно зарегистрируйте `IBadSingleton`, в конструктор которого注入ён `IRepository`, и попробуйте разрешить его из корневого провайдера. Из-за `ValidateScopes = true` контейнер выбросит `InvalidOperationException` с сообщением про `Cannot consume scoped service from singleton`. Перехватите исключение в `try/catch` и залогируйте его как «ожидаемая ошибка, демонстрирующая captive dependency». После проверки уберите эту регистрацию, чтобы приложение оставалось корректным.
9. Запустите приложение: `dotnet run --project LifetimeLab`. Ожидаемый вывод: логи запуска хоста, три блока диагностики (Transient/Singleton/Scoped с правильными равенствами/неравенствами Guid), 2–3 строки отчёта от `ReportWorker` с **разными** `IRepository.InstanceId` (доказательство, что scope создаётся на каждой итерации), и сообщение о перехваченной ошибке captive dependency.
10. Добавьте unit-тест `LifetimeLab.Tests` (xUnit): `dotnet new xunit -o LifetimeLab.Tests`, `dotnet add LifetimeLab.Tests reference LifetimeLab`. Напишите тест `Scoped_Service_Is_Same_Inside_Scope` и `Scoped_Service_Differs_Across_Scopes`, используя `ServiceCollection` напрямую. Убедитесь `dotnet test` зелёный.

#### Требования к решению

- Целевая платформа — .NET 8, язык C# 12. Включите `<Nullable>enable</Nullable>`, `<ImplicitUsings>enable</ImplicitUsings>`, по желанию `<TreatWarningsAsErrors>true</TreatWarningsAsErrors>`.
- Используйте top-level statements в `Program.cs`; никаких `static void Main`.
- Все публичные классы-сервисы — `sealed`, с `readonly` полями, без изменяемого состояния (кроме thread-safe singleton, если потребуется). Конструкторы синхронные — `async Task`-конструктор запрещён языком и уроком.
- Регистрация сервисов — строго через интерфейсы: `AddTransient<IEventFormatter, TransientFormatter>()` и т.п. Конкретные классы в потребителях не упоминаются.
- `BackgroundService` принимает только `IServiceScopeFactory`. Внутри `ExecuteAsync` каждый цикл создаёт `AsyncServiceScope` через `await using`. Никакого ручного `Dispose()` без `using`.
- В `Host.CreateDefaultBuilder` включены `ValidateScopes = true` и `ValidateOnBuild = true`. Должно быть видно, что в Dev контейнер реально проверяет граф зависимостей.
- Все три lifetime задействованы и **наблюдаемы** через Guid-идентификаторы экземпляров. Логи используют `ILogger` с категориями, а не «голый» `Console.WriteLine` (допускается `Console.WriteLine` только в стартовом выводе для наглядности).
- Код компилируется без warning и проходит `dotnet test`. В решении нет TODO, заглушек, закомментированного кода «на потом».
- Соблюдается правило урока «зависимость не живёт дольше потребителя»: Singleton не зависит от Scoped; если singleton-воркеру нужен Scoped — только через фабрику scope.

#### Тонкости и подводные камни

- **Captive dependency — главная ловушка.** Если `Singleton` (или `BackgroundService`, который тоже singleton в терминах DI) примет в конструктор `IRepository` (Scoped), то этот `IRepository` «захватится» навсегда: один и тот же экземпляр будет жить столько же, сколько воркер, и все итерации цикла будут видеть один Guid. Контейнер не «замечает» это сам по себе — только `ValidateScopes = true` в Development выбрасывает `InvalidOperationException`. Поэтому: либо переделайте зависимость на Singleton, либо внедряйте `IServiceScopeFactory` и создавайте scope на каждое использование. Это прямой перенос best practice из урока.
- **`using`/`await using` на scope — обязательно.** Забытый `Dispose` у `CreateScope()` утечёт scoped-ресурсы: не диспозятся `DbContext`, не закроются соединения, не отработают `IAsyncDisposable`. В `async`-контексте (`ExecuteAsync`) используйте именно `CreateAsyncScope()` и `await using AsyncServiceScope` — это даёт `IAsyncDisposable`-путь диспоза. Синхронный `CreateScope()` оставляйте только для коротких синхронных блоков.
- **Конструктор не может быть `async`.** C# не разрешает `public async Foo(...)`. Если сервису нужна async-инициализация (открыть соединение, прогреть кэш), выносите её в метод `InitAsync()` или используйте фабрику `services.AddSingleton<IFoo>(sp => { var f = new Foo(); f.InitAsync(sp).GetAwaiter().GetResult(); return f; })` с осторожностью. В этом ДЗ async-инициализация не нужна — держите конструкторы простыми.
- **`ValidateOnBuild` ловит незарегистрированные зависимости** на старте: если потребитель требует `IBar`, а `IBar` не зарегистрирован, хост упадёт при `Build()`, а не при первом разрешении. Это значительно сокращает время дебага.
- **Singleton с mutable state должен быть потокобезопасным.** Если в `IConfigCache` появится кэш-словарь, используйте `ConcurrentDictionary<string, string>` или `lock`. Урок явно предупреждает: singleton живёт вечно и к нему обращаются несколько потоков.
- **Не разрешайте Scoped из корневого провайдера.** `host.Services.GetRequiredService<IRepository>()` в Dev выбросит исключение при `ValidateScopes`. Scoped-сервисы берутся только внутри scope. Если вам нужен Scoped на старте приложения — создайте scope явно.
- **Регистрация по интерфейсу упрощает тесты.** `AddScoped<IRepository, ScopedRepository>()` позволяет в тестах подменить реализацию на mock/fake, не трогая потребителей. Регистрация по конкретному классу (`AddScoped<ScopedRepository>()`) ломает эту гибкость.
- **Порядок диспоза обратный порядку создания.** Контейнер диспозит сервисы в обратном порядке — это гарантирует, что зависимости умерли после потребителей. Если ваш сервис держит другой `IDisposable` напрямую (не через DI), диспозьте его вручную.

#### Критерии приёмки

- [ ] Проект `LifetimeLab` собирается под .NET 8 / C# 12 без warning и error.
- [ ] В `Program.cs` используются top-level statements, `Host.CreateDefaultBuilder` и `Build()`.
- [ ] Включены `ValidateScopes = true` и `ValidateOnBuild = true` через `UseDefaultServiceProvider`.
- [ ] Зарегистрированы `IEventFormatter` (Transient), `IRepository` (Scoped), `IConfigCache` (Singleton) — по интерфейсам.
- [ ] Демонстрация Transient показывает **два разных** Guid при двух разрешениях.
- [ ] Демонстрация Singleton показывает **одинаковый** Guid при двух разрешениях.
- [ ] Демонстрация Scoped показывает одинаковый Guid внутри одного scope и **разные** Guid между двумя scope.
- [ ] `ReportWorker : BackgroundService` принимает только `IServiceScopeFactory`, без `IRepository` в конструкторе.
- [ ] В `ExecuteAsync` каждая итерация создаёт `await using AsyncServiceScope` и получает свежий `IRepository` с новым Guid.
- [ ] Воспроизведён captive dependency: временная регистрация `IBadSingleton(IRepository)` выбрасывает `InvalidOperationException` из-за `ValidateScopes`; ошибка перехвачена и залогирована.
- [ ] Все классы-сервисы `sealed`, поля `readonly`, конструкторы синхронные.
- [ ] Логи используют `ILogger` с категориями; нет «голого» `Console.WriteLine` в основном потоке диагностики.
- [ ] Проект `LifetimeLab.Tests` (xUnit) содержит минимум два зелёных теста про Scoped.
- [ ] `dotnet test` проходит, `dotnet run` выводит все ожидаемые блоки.
- [ ] В решении нет TODO, закомментированного кода, заглушек.

#### Подсказки (без прямого ответа)

- Вспомните аналогии урока: одноразовая бумажная чашка (Transient), столовый поднос (Scoped), сейф в банке (Singleton). Какая из них подходит сервису, который не хранит состояние и дешев в создании?
- Если воркер «живёт вечно», а репозиторий — «на одну транзакцию», кто кого должен переживать? Какое правило из урока нарушается при прямой инъекции?
- Чтобы увидеть два разных Guid у Transient, достаточно вызвать `GetRequiredService` дважды из **одного** провайдера. Для Scoped два раза из **одного** scope дадут один Guid — нужен второй scope.
- Для перехвата captive dependency используйте обычный `try/catch (InvalidOperationException ex)` и залогируйте `ex.Message`. Сообщение контейнера содержит фразу «scoped service» — это и есть диагностический признак.
- Включить валидацию можно и через `appsettings.json` (`"ValidationOptions"`), но в ДЗ проще сделать программно в `UseDefaultServiceProvider`.

#### Эталонное решение (разбор)

```csharp
// LifetimeLab/Program.cs — C# 12 / .NET 8, top-level statements
using LifetimeLab;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;

// Включаем ValidateScopes и ValidateOnBuild — ловим ошибки lifetimes на старте.
// Enable ValidateScopes and ValidateOnBuild — catch lifetime errors at startup.
var host = Host.CreateDefaultBuilder(args)
    .UseDefaultServiceProvider((_, options) =>
    {
        options.ValidateScopes = true;   // запрещает singleton -> scoped / forbids singleton -> scoped
        options.ValidateOnBuild = true;  // проверяет, что все зависимости зарегистрированы / checks all deps registered
    })
    .ConfigureServices((_, s) => s.AddAppServices())
    .Build();

await host.StartAsync();

// Диагностика lifetimes глазами / Observe lifetimes with your own eyes.
await Diagnostics.RunAsync(host.Services);

// Фоновый воркер уже запущен хостом; даём ему поработать 2.5 c, потом гасим.
// The hosted worker is already started by the host; let it run ~2.5s then stop.
using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(2.5));
try { await host.WaitForShutdownAsync(cts.Token); }
catch (OperationCanceledException) { /* ожидаемо / expected */ }

await host.StopAsync();
```

```csharp
// LifetimeLab/Services.cs
using Microsoft.Extensions.Logging;

namespace LifetimeLab;

// --- Интерфейсы / Interfaces ---
public interface IEventFormatter
{
    string Format(string evt, Guid correlationId); // Transient-кандидат
}

public interface IRepository
{
    Guid InstanceId { get; }      // Scoped-кандидат: Guid наблюдаем
    string Snapshot();
}

public interface IConfigCache
{
    Guid InstanceId { get; }      // Singleton-кандидат: один Guid на всё приложение
    string Get(string key);
}

// --- Реализации: sealed + readonly поля, синхронные конструкторы ---
public sealed class TransientFormatter : IEventFormatter
{
    private readonly Guid _id = Guid.NewGuid();
    public string Format(string evt, Guid correlationId) =>
        $"[{correlationId:B}] {_id:B} evt={evt}";
}

public sealed class ScopedRepository : IRepository
{
    private readonly ILogger<ScopedRepository> _logger;
    public Guid InstanceId { get; } = Guid.NewGuid();

    public ScopedRepository(ILogger<ScopedRepository> logger) => _logger = logger;

    public string Snapshot()
    {
        _logger.LogDebug("Snapshot requested for {InstanceId}", InstanceId);
        return $"repo#{InstanceId:B}";
    }
}

public sealed class SingletonConfigCache : IConfigCache
{
    // Singleton с immutable состоянием — потокобезопасен по построению.
    // A singleton with immutable state is thread-safe by construction.
    private readonly Dictionary<string, string> _values = new()
    {
        ["env"] = "dev",
        ["region"] = "eu",
    };

    public Guid InstanceId { get; } = Guid.NewGuid();

    public string Get(string key) => _values.TryGetValue(key, out var v) ? v : "<missing>";
}

// --- Долгоживущий потребитель, безопасно работающий со Scoped ---
public sealed class ReportWorker : BackgroundService
{
    // НЕЛЬЗЯ注入ить IRepository напрямую — это captive dependency.
    // Do NOT inject IRepository directly — that is a captive dependency.
    private readonly IServiceScopeFactory _scopeFactory;
    private readonly ILogger<ReportWorker> _logger;

    public ReportWorker(IServiceScopeFactory scopeFactory, ILogger<ReportWorker> logger)
    {
        _scopeFactory = scopeFactory;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            // Каждая итерация — свой scope, свой ScopedRepository, свой Guid.
            // Each iteration: its own scope, its own ScopedRepository, its own Guid.
            await using AsyncServiceScope scope = _scopeFactory.CreateAsyncScope();
            var repo = scope.ServiceProvider.GetRequiredService<IRepository>();
            _logger.LogInformation("Worker report from {Repo}", repo.Snapshot());

            try { await Task.Delay(TimeSpan.FromMilliseconds(800), stoppingToken); }
            catch (OperationCanceledException) { break; }
        }
    }
}

// --- Регистрация / Registration ---
public static class ServiceRegistration
{
    public static IServiceCollection AddAppServices(this IServiceCollection services)
    {
        services.AddHostedService<ReportWorker>();                          // hosted == singleton
        services.AddTransient<IEventFormatter, TransientFormatter>();
        services.AddScoped<IRepository, ScopedRepository>();
        services.AddSingleton<IConfigCache, SingletonConfigCache>();
        return services;
    }
}
```

```csharp
// LifetimeLab/Diagnostics.cs
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Logging;

namespace LifetimeLab;

public static class Diagnostics
{
    public static async Task RunAsync(IServiceProvider root)
    {
        var log = root.GetRequiredService<ILogger<Diagnostics>>();

        // Transient: два разрешения — два разных Guid.
        var f1 = root.GetRequiredService<IEventFormatter>();
        var f2 = root.GetRequiredService<IEventFormatter>();
        log.LogInformation("Transient formatter A: {A}", f1.Format("evtA", Guid.NewGuid()));
        log.LogInformation("Transient formatter B: {B}", f2.Format("evtB", Guid.NewGuid()));

        // Singleton: всегда один Guid.
        var c1 = root.GetRequiredService<IConfigCache>();
        var c2 = root.GetRequiredService<IConfigCache>();
        log.LogInformation("Singleton cache same? {Same}", c1.InstanceId == c2.InstanceId);

        // Scoped: один Guid внутри scope, разные — между scope.
        Guid s1a, s1b, s2;
        await using (var scope1 = root.CreateAsyncScope())
        {
            s1a = scope1.ServiceProvider.GetRequiredService<IRepository>().InstanceId;
            s1b = scope1.ServiceProvider.GetRequiredService<IRepository>().InstanceId;
        }
        await using (var scope2 = root.CreateAsyncScope())
        {
            s2 = scope2.ServiceProvider.GetRequiredService<IRepository>().InstanceId;
        }
        log.LogInformation("Scoped same in scope? {Same}", s1a == s1b);   // True
        log.LogInformation("Scoped diff across scopes? {Diff}", s1a != s2); // True

        // Демонстрация captive dependency: намеренно неверная регистрация.
        // Captive dependency demo: an intentionally wrong registration.
        try
        {
            var bad = new ServiceCollection()
                .AddScoped<IRepository, ScopedRepository>()
                .AddSingleton<BadSingleton>() // конструктор принимает IRepository / ctor takes IRepository
                .BuildServiceProvider(new ServiceProviderOptions { ValidateScopes = true });
            bad.GetRequiredService<BadSingleton>();
        }
        catch (InvalidOperationException ex)
        {
            log.LogWarning("Captive dependency detected (expected): {Message}", ex.Message);
        }
    }
}

// Намеренно неверный класс — только для негативной демонстрации.
// Intentionally wrong class — only for the negative demo.
file sealed class BadSingleton
{
    public BadSingleton(IRepository repo) { } // singleton -> scoped = captive dependency
}
```

**Разбор по строкам.** `Host.CreateDefaultBuilder` поднимает хост с логированием и DI; `UseDefaultServiceProvider` включает `ValidateScopes` и `ValidateOnBuild` — это прямое применение best practice из урока, чтобы контейнер сам ловил ошибки lifetimes и незарегистрированные зависимости на старте, а не в рантайме. Регистрации в `AddAppServices` идут строго по интерфейсам (`AddTransient<IEventFormatter, TransientFormatter>` и т.д.) — это даёт тестируемость и заменяемость, о чём прямо говорит урок. `TransientFormatter` не имеет состояния кроме `Guid`, создаётся дёшево и выбрасывается — идеальный Transient; два разрешения из корневого провайдера дают разные Guid, что и наблюдает `Diagnostics.RunAsync`. `ScopedRepository` получает `ILogger` через конструктор — DI сам подставит нужную категорию логгера, и в рамках одного scope мы получаем один экземпляр с одним Guid. `SingletonConfigCache` хранит immutable-словарь, инициализированный коллекционным выражением C# 12 (`new() { ["env"] = "dev" }`), и потому потокобезопасен по построению — урок подчёркивает, что Singleton обязан быть потокобезопасным. Главный обучающий момент — `ReportWorker`: это `BackgroundService`, который в терминах DI является singleton и живёт столько же, сколько приложение. Если бы он принял `IRepository` в конструктор, мы получили бы captive dependency: один репозиторий навсегда, и все итерации цикла видели бы один Guid, «зависнув» на устаревших данных. Вместо этого мы注入им `IServiceScopeFactory` — саму фабрику можно безопасно держать в singleton — и в каждой итерации `ExecuteAsync` создаём `await using AsyncServiceScope`. `CreateAsyncScope()` (а не синхронный `CreateScope()`) даёт `IAsyncDisposable`-путь диспоза, что важно для асинхронных ресурсов; `await using` гарантирует, что scope диспозится даже при исключении, и контейнер автоматически диспозит `ScopedRepository` и его логгер в обратном порядке. В результате каждая итерация видит **новый** Guid — прямое доказательство, что lifetime выбран верно. Блок `Diagnostics.RunAsync` с двумя scope демонстрирует ключевое свойство Scoped: `s1a == s1b` (один экземпляр внутри scope) и `s1a != s2` (разные между scope). Наконец, негативный блок с `BadSingleton` собирает отдельный `ServiceCollection` с `ValidateScopes = true` и регистрирует singleton, чей конструктор требует Scoped-зависимость; при разрешении контейнер выбрасывает `InvalidOperationException`, и мы перехватываем его как «ожидаемую ошибку» — это teaches вас читать диагностические сообщения и понимать, что именно запретило контейнер. `file sealed class BadSingleton` использует модификатор `file` из C# 11+, чтобы класс существовал только в этом файле и не засорял пространство имён. Паттерн-матчинг `try/catch (InvalidOperationException ex)` и `try { await Task.Delay(..., stoppingToken); } catch (OperationCanceledException) { break; }` показывает, как аккуратно обрабатывать отмену без `IsCancellationRequested`-шума. Всё вместе решение покрывает все lifetime, captive dependency, scope factory, валидацию контейнера и современные возможности C# 12.

#### Задания на углубление (бонус)

1. **Кастомная фабрика с открытым scope.** Реализуйте `IReportFactory`, который внутри себя держит `IServiceScopeFactory` и возвращает `IReport` вместе с `IDisposable`-токеном, освобождающим scope. Покажите, что несколько отчётов, созданных через фабрику, получают разные `IRepository.InstanceId`.
2. **Keyed services (нововведение .NET 8).** Зарегистрируйте две реализации `IRepository` — `LiveRepository` и `ArchiveRepository` — через `AddKeyedScoped<IRepository>("live")`/`AddKeyedScoped<IRepository>("archive")` и разрешите их через `[FromKeyedServices("live")]` или `provider.GetKeyedService<IRepository>("live")`. Сравните с классическим подходом.
3. **Тест на captive dependency.** Напишите xUnit-тест, который утверждает, что `BuildServiceProvider(new ServiceProviderOptions { ValidateScopes = true })` выбрасывает именно `InvalidOperationException` для регистрации singleton→scoped. Используйте `Assert.Throws<InvalidOperationException>`.
4. **Benchmark Transient vs Singleton.** С помощью `dotnet add package BenchmarkDotNet` измерьте стоимость разрешения `IEventFormatter` (Transient) vs `IConfigCache` (Singleton) в цикле 1 000 000 раз. Объясните, почему Transient дороже, и когда это всё же оправдано (лекарь-объект, отсутствие состояния).

---

## Statement in English / Постановка на английском

#### Context & motivation

You maintain an internal console application `LifetimeLab` for the company's education department. The application models a typical backend: there is a lightweight stateless formatter (turns domain events into log strings), there is a repository tied to a "transaction" (an imitation of `DbContext`), and there is a global configuration cache shared by every thread. On top of that, a `BackgroundService` runs in the background and produces a report every few seconds based on fresh repository data — a textbook scenario where a long-lived consumer must safely reach into a Scoped service.

Historically the app was written by different people, and over time it accumulated the classic DI mistakes: someone injected `IRepository` straight into a singleton worker (a captive dependency), someone forgot `using` on `CreateScope()`, someone tried to write an `async Task` constructor. In Dev everything "worked" because container validation was off, but in Production strange bugs surfaced: the same `IRepository` instance lived for days, returning the same Guid to every thread, and the background worker's reports got stuck on stale data. Your job is to clean up the DI: pick the right lifetime for each service, prove correctness with Guid experiments, eliminate the captive dependency with `IServiceScopeFactory`, turn on `ValidateScopes`/`ValidateOnBuild`, and confirm that the container itself refuses to start when a lifetime rule is violated. This gives you the kind of intuition theory cannot deliver: you will see with your own eyes how the very same class behaves differently depending on registration, and you will learn to read the container's diagnostic exceptions.

#### What to do step by step

1. Create a new project: `dotnet new console -n LifetimeLab -o LifetimeLab --framework net8.0`. Enter the `LifetimeLab` directory and open the project. Make sure `LifetimeLab.csproj` declares `<TargetFramework>net8.0</TargetFramework>`, `<ImplicitUsings>enable</ImplicitUsings>`, `<Nullable>enable</Nullable>`.
2. Add the hosting and logging package: `dotnet add package Microsoft.Extensions.Hosting`. That is enough — `Microsoft.Extensions.DependencyInjection` and `Microsoft.Extensions.Logging` come transitively. Run `dotnet build` and confirm a clean build (`Build succeeded`, 0 Warning, 0 Error).
3. In `Program.cs` use top-level statements. Build a host with `Host.CreateDefaultBuilder(args)`, call an extension method `AddAppServices()` (you will write it in a static class `ServiceRegistration`) inside `ConfigureServices`, and enable validation: `.UseDefaultServiceProvider((_, options) => { options.ValidateScopes = true; options.ValidateOnBuild = true; })`. Build the host with `.Build()`.
4. Define three interfaces: `IEventFormatter` (method `string Format(string evt, Guid correlationId)`), `IRepository` (property `Guid InstanceId`, method `string Snapshot()`), `IConfigCache` (property `Guid InstanceId`, method `string Get(string key)`). All implementations are `sealed` C# 12 classes; each holds `private readonly Guid _id = Guid.NewGuid();` and prints it in `Snapshot()`/`Get()` so you can observe the lifetime with your eyes.
5. Implementations: `TransientFormatter : IEventFormatter`, `ScopedRepository : IRepository`, `SingletonConfigCache : IConfigCache`. Register them in `AddAppServices` as `AddTransient`, `AddScoped`, `AddSingleton` respectively — against the interface, not the concrete class (a best practice from the lesson).
6. Write `ReportWorker : BackgroundService`. In its constructor inject **only** `IServiceScopeFactory` (NOT `IRepository` — that would be a captive dependency). In `ExecuteAsync` run a `while (!stoppingToken.IsCancellationRequested)` loop; inside it, use `await using AsyncServiceScope scope = _scopeFactory.CreateAsyncScope();` to get a fresh `IRepository` and print a report. Run 2–3 iterations with `Task.Delay(TimeSpan.FromMilliseconds(800), stoppingToken)`, then let a `CancellationTokenSource` with `CancelAfter` terminate the loop from `Program.cs`.
7. After `host.StartAsync()` in `Program.cs`, run a diagnostics function `Diagnostics.RunAsync(host.Services)` that: (a) requests `IEventFormatter` twice and shows two different Guids (Transient); (b) requests `IConfigCache` twice and shows the same Guid (Singleton); (c) creates two scopes via `CreateAsyncScope()` and inside each requests `IRepository` twice, demonstrating "one instance inside a scope, different across scopes" (Scoped). Route every line through `ILogger<T>` with categories — that brings the task closer to a real backend.
8. In a separate method reproduce the captive dependency "negatively": temporarily register an `IBadSingleton` whose constructor注入s `IRepository`, and try to resolve it from the root provider. Because of `ValidateScopes = true`, the container throws `InvalidOperationException` with a message about "Cannot consume scoped service from singleton". Catch the exception in a `try/catch` and log it as "expected error, demonstrating captive dependency". After the check remove the registration so the application stays correct.
9. Run the app: `dotnet run --project LifetimeLab`. Expected output: host startup logs, three diagnostics blocks (Transient/Singleton/Scoped with the right Guid equalities/inequalities), 2–3 report lines from `ReportWorker` with **different** `IRepository.InstanceId` (proof that a scope is created per iteration), and a message about the captured captive-dependency error.
10. Add a unit test project `LifetimeLab.Tests` (xUnit): `dotnet new xunit -o LifetimeLab.Tests`, `dotnet add LifetimeLab.Tests reference LifetimeLab`. Write a test `Scoped_Service_Is_Same_Inside_Scope` and `Scoped_Service_Differs_Across_Scopes`, using `ServiceCollection` directly. Confirm `dotnet test` is green.

#### Requirements

- Target platform is .NET 8, language C# 12. Turn on `<Nullable>enable</Nullable>`, `<ImplicitUsings>enable</ImplicitUsings>`, optionally `<TreatWarningsAsErrors>true</TreatWarningsAsErrors>`.
- Use top-level statements in `Program.cs`; no `static void Main`.
- All public service classes are `sealed`, with `readonly` fields, no mutable state (except a thread-safe singleton if needed). Constructors are synchronous — an `async Task` constructor is forbidden both by the language and by the lesson.
- Service registration is strictly through interfaces: `AddTransient<IEventFormatter, TransientFormatter>()` and so on. Concrete classes are never referenced by consumers.
- The `BackgroundService` accepts only `IServiceScopeFactory`. Inside `ExecuteAsync` every iteration creates an `AsyncServiceScope` via `await using`. No manual `Dispose()` without `using`.
- `Host.CreateDefaultBuilder` has `ValidateScopes = true` and `ValidateOnBuild = true`. It must be visible that in Dev the container really validates the dependency graph.
- All three lifetimes are used and **observable** through instance Guid identifiers. Logs use `ILogger` with categories, not bare `Console.WriteLine` (a `Console.WriteLine` in the very first startup banner is acceptable for clarity).
- The code compiles without warnings and passes `dotnet test`. No TODOs, no stubs, no commented-out "for later" code.
- The lesson's rule "a dependency cannot outlive its consumer" is respected: a Singleton does not depend on a Scoped; if a singleton worker needs a Scoped, it goes only through the scope factory.

#### Pitfalls

- **Captive dependency is the main trap.** If a `Singleton` (or a `BackgroundService`, which is also a singleton in DI terms) takes `IRepository` (Scoped) in its constructor, that `IRepository` is "captured" forever: the same instance lives as long as the worker, and every loop iteration sees the same Guid. The container does not "notice" this on its own — only `ValidateScopes = true` in Development throws `InvalidOperationException`. So: either rework the dependency to Singleton, or inject `IServiceScopeFactory` and create a scope per use. This is a direct application of the lesson's best practice.
- **`using`/`await using` on a scope is mandatory.** A forgotten `Dispose` on `CreateScope()` leaks scoped resources: `DbContext` is not disposed, connections stay open, `IAsyncDisposable` is not honored. In an `async` context (`ExecuteAsync`) use `CreateAsyncScope()` and `await using AsyncServiceScope` — that gives the `IAsyncDisposable` dispose path. Leave the synchronous `CreateScope()` only for short synchronous blocks.
- **A constructor cannot be `async`.** C# forbids `public async Foo(...)`. If a service needs async initialization (open a connection, warm a cache), move it to an `InitAsync()` method or use a factory `services.AddSingleton<IFoo>(sp => { var f = new Foo(); f.InitAsync(sp).GetAwaiter().GetResult(); return f; })` with care. This homework needs no async init — keep constructors simple.
- **`ValidateOnBuild` catches unregistered dependencies** at startup: if a consumer requires `IBar` and `IBar` is not registered, the host dies at `Build()`, not at first resolution. That dramatically shortens debug time.
- **A singleton with mutable state must be thread-safe.** If `IConfigCache` gains a cache dictionary, use `ConcurrentDictionary<string, string>` or a `lock`. The lesson warns explicitly: a singleton lives forever and several threads reach for it.
- **Do not resolve Scoped from the root provider.** `host.Services.GetRequiredService<IRepository>()` throws in Dev under `ValidateScopes`. Scoped services are taken only inside a scope. If you need a Scoped service at startup — create a scope explicitly.
- **Registering against an interface makes tests easy.** `AddScoped<IRepository, ScopedRepository>()` lets tests swap in a mock/fake without touching consumers. Registering against a concrete class (`AddScoped<ScopedRepository>()`) breaks that flexibility.
- **Disposal order is the reverse of creation order.** The container disposes services in reverse order — guaranteeing that dependencies die after consumers. If your service holds another `IDisposable` directly (not through DI), dispose of it manually.

#### Acceptance criteria

- [ ] The `LifetimeLab` project builds under .NET 8 / C# 12 with no warnings or errors.
- [ ] `Program.cs` uses top-level statements, `Host.CreateDefaultBuilder` and `Build()`.
- [ ] `ValidateScopes = true` and `ValidateOnBuild = true` are enabled via `UseDefaultServiceProvider`.
- [ ] `IEventFormatter` (Transient), `IRepository` (Scoped), `IConfigCache` (Singleton) are registered against interfaces.
- [ ] The Transient demo shows **two different** Guids across two resolutions.
- [ ] The Singleton demo shows the **same** Guid across two resolutions.
- [ ] The Scoped demo shows the same Guid inside one scope and **different** Guids across two scopes.
- [ ] `ReportWorker : BackgroundService` accepts only `IServiceScopeFactory`; no `IRepository` in the constructor.
- [ ] In `ExecuteAsync` every iteration creates an `await using AsyncServiceScope` and gets a fresh `IRepository` with a new Guid.
- [ ] The captive dependency is reproduced: a temporary `IBadSingleton(IRepository)` registration throws `InvalidOperationException` thanks to `ValidateScopes`; the error is caught and logged.
- [ ] All service classes are `sealed`, fields are `readonly`, constructors are synchronous.
- [ ] Logs use `ILogger` with categories; no bare `Console.WriteLine` in the main diagnostics flow.
- [ ] The `LifetimeLab.Tests` (xUnit) project has at least two green tests about Scoped.
- [ ] `dotnet test` passes, `dotnet run` prints all expected blocks.
- [ ] No TODOs, no commented-out code, no stubs.

#### Hints (no direct answer)

- Recall the lesson's analogies: a paper cup (Transient), a cafeteria tray (Scoped), a bank safe (Singleton). Which one fits a service that holds no state and is cheap to create?
- If the worker "lives forever" and the repository is "for one transaction", who should outlive whom? Which rule from the lesson does a direct injection break?
- To see two different Transient Guids, call `GetRequiredService` twice from the **same** provider. For Scoped, two calls from the **same** scope yield one Guid — you need a second scope.
- To catch the captive dependency use a plain `try/catch (InvalidOperationException ex)` and log `ex.Message`. The container's message contains the phrase "scoped service" — that is the diagnostic signature.
- Validation can also be turned on through `appsettings.json` (`"ValidationOptions"`), but for this homework it is simpler to do it programmatically in `UseDefaultServiceProvider`.

#### Reference solution walk-through

```csharp
// LifetimeLab/Program.cs — C# 12 / .NET 8, top-level statements
using LifetimeLab;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;

// Turn on ValidateScopes and ValidateOnBuild — catch lifetime errors at startup.
var host = Host.CreateDefaultBuilder(args)
    .UseDefaultServiceProvider((_, options) =>
    {
        options.ValidateScopes = true;   // forbids singleton -> scoped
        options.ValidateOnBuild = true;  // checks all deps are registered
    })
    .ConfigureServices((_, s) => s.AddAppServices())
    .Build();

await host.StartAsync();

// Observe lifetimes with your own eyes.
await Diagnostics.RunAsync(host.Services);

// The hosted worker is already started by the host; let it run ~2.5s then stop.
using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(2.5));
try { await host.WaitForShutdownAsync(cts.Token); }
catch (OperationCanceledException) { /* expected */ }

await host.StopAsync();
```

```csharp
// LifetimeLab/Services.cs
using Microsoft.Extensions.Logging;

namespace LifetimeLab;

public interface IEventFormatter
{
    string Format(string evt, Guid correlationId); // a Transient candidate
}

public interface IRepository
{
    Guid InstanceId { get; }      // a Scoped candidate: Guid is observable
    string Snapshot();
}

public interface IConfigCache
{
    Guid InstanceId { get; }      // a Singleton candidate: one Guid for the whole app
    string Get(string key);
}

// Implementations: sealed + readonly fields, synchronous constructors.
public sealed class TransientFormatter : IEventFormatter
{
    private readonly Guid _id = Guid.NewGuid();
    public string Format(string evt, Guid correlationId) =>
        $"[{correlationId:B}] {_id:B} evt={evt}";
}

public sealed class ScopedRepository : IRepository
{
    private readonly ILogger<ScopedRepository> _logger;
    public Guid InstanceId { get; } = Guid.NewGuid();

    public ScopedRepository(ILogger<ScopedRepository> logger) => _logger = logger;

    public string Snapshot()
    {
        _logger.LogDebug("Snapshot requested for {InstanceId}", InstanceId);
        return $"repo#{InstanceId:B}";
    }
}

public sealed class SingletonConfigCache : IConfigCache
{
    // A singleton with immutable state is thread-safe by construction.
    private readonly Dictionary<string, string> _values = new()
    {
        ["env"] = "dev",
        ["region"] = "eu",
    };

    public Guid InstanceId { get; } = Guid.NewGuid();

    public string Get(string key) => _values.TryGetValue(key, out var v) ? v : "<missing>";
}

// A long-lived consumer that reaches into Scoped safely.
public sealed class ReportWorker : BackgroundService
{
    // Do NOT inject IRepository directly — that is a captive dependency.
    private readonly IServiceScopeFactory _scopeFactory;
    private readonly ILogger<ReportWorker> _logger;

    public ReportWorker(IServiceScopeFactory scopeFactory, ILogger<ReportWorker> logger)
    {
        _scopeFactory = scopeFactory;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            // Each iteration: its own scope, its own ScopedRepository, its own Guid.
            await using AsyncServiceScope scope = _scopeFactory.CreateAsyncScope();
            var repo = scope.ServiceProvider.GetRequiredService<IRepository>();
            _logger.LogInformation("Worker report from {Repo}", repo.Snapshot());

            try { await Task.Delay(TimeSpan.FromMilliseconds(800), stoppingToken); }
            catch (OperationCanceledException) { break; }
        }
    }
}

// Registration.
public static class ServiceRegistration
{
    public static IServiceCollection AddAppServices(this IServiceCollection services)
    {
        services.AddHostedService<ReportWorker>();                          // hosted == singleton
        services.AddTransient<IEventFormatter, TransientFormatter>();
        services.AddScoped<IRepository, ScopedRepository>();
        services.AddSingleton<IConfigCache, SingletonConfigCache>();
        return services;
    }
}
```

```csharp
// LifetimeLab/Diagnostics.cs
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Logging;

namespace LifetimeLab;

public static class Diagnostics
{
    public static async Task RunAsync(IServiceProvider root)
    {
        var log = root.GetRequiredService<ILogger<Diagnostics>>();

        // Transient: two resolutions — two different Guids.
        var f1 = root.GetRequiredService<IEventFormatter>();
        var f2 = root.GetRequiredService<IEventFormatter>();
        log.LogInformation("Transient formatter A: {A}", f1.Format("evtA", Guid.NewGuid()));
        log.LogInformation("Transient formatter B: {B}", f2.Format("evtB", Guid.NewGuid()));

        // Singleton: always the same Guid.
        var c1 = root.GetRequiredService<IConfigCache>();
        var c2 = root.GetRequiredService<IConfigCache>();
        log.LogInformation("Singleton cache same? {Same}", c1.InstanceId == c2.InstanceId);

        // Scoped: one Guid inside a scope, different across scopes.
        Guid s1a, s1b, s2;
        await using (var scope1 = root.CreateAsyncScope())
        {
            s1a = scope1.ServiceProvider.GetRequiredService<IRepository>().InstanceId;
            s1b = scope1.ServiceProvider.GetRequiredService<IRepository>().InstanceId;
        }
        await using (var scope2 = root.CreateAsyncScope())
        {
            s2 = scope2.ServiceProvider.GetRequiredService<IRepository>().InstanceId;
        }
        log.LogInformation("Scoped same in scope? {Same}", s1a == s1b);   // True
        log.LogInformation("Scoped diff across scopes? {Diff}", s1a != s2); // True

        // Captive dependency demo: an intentionally wrong registration.
        try
        {
            var bad = new ServiceCollection()
                .AddScoped<IRepository, ScopedRepository>()
                .AddSingleton<BadSingleton>() // ctor takes IRepository
                .BuildServiceProvider(new ServiceProviderOptions { ValidateScopes = true });
            bad.GetRequiredService<BadSingleton>();
        }
        catch (InvalidOperationException ex)
        {
            log.LogWarning("Captive dependency detected (expected): {Message}", ex.Message);
        }
    }
}

// Intentionally wrong class — only for the negative demo.
file sealed class BadSingleton
{
    public BadSingleton(IRepository repo) { } // singleton -> scoped = captive dependency
}
```

**Line-by-line walk-through.** `Host.CreateDefaultBuilder` brings up a host with logging and DI; `UseDefaultServiceProvider` turns on `ValidateScopes` and `ValidateOnBuild` — a direct application of the lesson's best practice so the container itself catches lifetime mistakes and unregistered dependencies at startup instead of in the runtime. Registrations in `AddAppServices` go strictly against interfaces (`AddTransient<IEventFormatter, TransientFormatter>` etc.), which gives testability and swap-ability, as the lesson explicitly states. `TransientFormatter` has no state beyond a Guid, is cheap to create and thrown away — an ideal Transient; two resolutions from the root provider yield different Guids, exactly what `Diagnostics.RunAsync` observes. `ScopedRepository` receives an `ILogger` through its constructor — DI itself injects the right logger category, and within a single scope we get one instance with one Guid. `SingletonConfigCache` keeps an immutable dictionary initialised with a C# 12 collection expression (`new() { ["env"] = "dev" }`) and is therefore thread-safe by construction — the lesson stresses that a Singleton must be thread-safe. The central teaching moment is `ReportWorker`: it is a `BackgroundService`, which in DI terms is a singleton and lives as long as the application. If it took `IRepository` in its constructor we would get a captive dependency: one repository forever, and every loop iteration would see the same Guid, stuck on stale data. Instead we inject `IServiceScopeFactory` — the factory itself is safe to keep in a singleton — and in every iteration of `ExecuteAsync` we create an `await using AsyncServiceScope`. `CreateAsyncScope()` (not the synchronous `CreateScope()`) gives the `IAsyncDisposable` dispose path, important for asynchronous resources; `await using` guarantees the scope is disposed even on exception, and the container automatically disposes `ScopedRepository` and its logger in reverse order. As a result every iteration sees a **new** Guid — direct proof the lifetime is correct. The `Diagnostics.RunAsync` block with two scopes demonstrates the key property of Scoped: `s1a == s1b` (one instance inside a scope) and `s1a != s2` (different across scopes). Finally, the negative block with `BadSingleton` builds a separate `ServiceCollection` with `ValidateScopes = true` and registers a singleton whose constructor demands a Scoped dependency; on resolution the container throws `InvalidOperationException`, which we catch as an "expected error" — this teaches you to read diagnostic messages and understand exactly what the container forbade. `file sealed class BadSingleton` uses the `file` modifier (C# 11+) so the class exists only in this file and does not pollute the namespace. The pattern `try/catch (InvalidOperationException ex)` together with `try { await Task.Delay(..., stoppingToken); } catch (OperationCanceledException) { break; }` shows how to handle cancellation cleanly without `IsCancellationRequested` noise. Taken together, the solution covers all three lifetimes, the captive dependency, the scope factory, container validation and modern C# 12 features.

#### Going deeper (bonus)

1. **Custom factory with an open scope.** Implement an `IReportFactory` that holds an `IServiceScopeFactory` internally and returns an `IReport` together with an `IDisposable` token that releases the scope. Show that several reports produced through the factory get different `IRepository.InstanceId` values.
2. **Keyed services (a .NET 8 feature).** Register two `IRepository` implementations — `LiveRepository` and `ArchiveRepository` — via `AddKeyedScoped<IRepository>("live")`/`AddKeyedScoped<IRepository>("archive")` and resolve them through `[FromKeyedServices("live")]` or `provider.GetKeyedService<IRepository>("live")`. Compare with the classic approach.
3. **A test for the captive dependency.** Write an xUnit test asserting that `BuildServiceProvider(new ServiceProviderOptions { ValidateScopes = true })` throws exactly `InvalidOperationException` for a singleton→scoped registration. Use `Assert.Throws<InvalidOperationException>`.
4. **Benchmark Transient vs Singleton.** With `dotnet add package BenchmarkDotNet` measure the cost of resolving `IEventFormatter` (Transient) vs `IConfigCache` (Singleton) in a loop of 1,000,000 iterations. Explain why Transient is more expensive and when it is still justified (a cheap object, no state).

---

#### Чек-лист сдачи / Submission checklist
- [ ] Проект `LifetimeLab` собирается под .NET 8 / C# 12 без warning.
- [ ] Включены `ValidateScopes` и `ValidateOnBuild`.
- [ ] Регистрации трёх lifetime по интерфейсам.
- [ ] Демонстрация Transient/Singleton/Scoped через Guid.
- [ ] `ReportWorker` принимает только `IServiceScopeFactory`, создаёт `AsyncServiceScope` на итерацию.
- [ ] Captive dependency воспроизведён и перехвачен.
- [ ] Тесты xUnit зелёные.
- [ ] `dotnet run` выводит все ожидаемые блоки.
- [ ] The `LifetimeLab` project builds under .NET 8 / C# 12 with no warnings.
- [ ] `ValidateScopes` and `ValidateOnBuild` are enabled.
- [ ] All three lifetimes are registered against interfaces.
- [ ] Transient/Singleton/Scoped are demonstrated through Guids.
- [ ] `ReportWorker` takes only `IServiceScopeFactory` and creates an `AsyncServiceScope` per iteration.
- [ ] The captive dependency is reproduced and caught.
- [ ] xUnit tests are green.
- [ ] `dotnet run` prints all expected blocks.

#### Ресурсы / Resources
- [Microsoft Learn — Dependency injection in .NET — https://learn.microsoft.com/dotnet/core/extensions/dependency-injection]
- [Dependency injection guidelines — https://learn.microsoft.com/dotnet/core/extensions/dependency-injection-guidelines]
- [DI in ASP.NET Core — https://learn.microsoft.com/aspnet/core/fundamentals/dependency-injection]
- [Keyed services in .NET 8 — https://learn.microsoft.com/dotnet/core/extensions/dependency-injection#keyed-services]
- [BackgroundService guidance — https://learn.microsoft.com/dotnet/core/extensions/workers]
