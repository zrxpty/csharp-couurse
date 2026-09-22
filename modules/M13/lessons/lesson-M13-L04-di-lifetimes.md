[⬆ К модулю M13](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M13-L04: Встроенный DI: AddTransient/Scoped/Singleton, lifetimes / Built-in DI: AddTransient/Scoped/Singleton, lifetimes

**Модуль / Module:** M13
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

Встроенный контейнер DI в .NET (`Microsoft.Extensions.DependencyInjection`) — это «бухгалтер», который знает, как создавать объекты и сколько они должны жить. Регистрация происходит через `IServiceCollection`, а разрешение — через `IServiceProvider`. Каждый сервис регистрируется с одним из трёх **времен жизни (lifetimes)**: `Transient`, `Scoped` или `Singleton`.

**Transient (мгновенный)** — новый экземпляр **каждый раз**, когда кто-то просит сервис. Аналогия: одноразовая бумажная чашка — выпил кофе и выбросил. Подходит для лёгких, не сохраняющих состояние сервисов: форматтеры, мапперы, helper-классы. Главный минус — давление на GC, если объект дорогой.

**Scoped (охваченный областью)** — один экземпляр **в пределах одного scope**. Аналогия: столовый поднос — пока ты в столовой, еда на одном подносе, вышел — поднос сдали. Самый частый lifetime в веб-приложениях: один scope создаётся на каждый HTTP-запрос. Идеально для `DbContext`, репозиториев, unit-of-work — всё, что привязано к одной транзакции. Вне веб-запроса scope создаётся вручную через `IServiceScopeFactory.CreateScope()`.

**Singleton (единственный)** — ровно один экземпляр **на всё приложение**. Аналогия: сейф в банке — один на всех клиентов, доступ синхронизирован. Подходит для кэшей, конфигураций, тяжёлых фабрик, `HttpClient` через `IHttpClientFactory`. Осторожно: хранит состояние всё время жизни процесса, поэтому должен быть **потокобезопасным**.

**`IServiceCollection`** — коллекция дескрипторов `ServiceDescriptor`. Методы `AddTransient<T>()`, `AddScoped<T>()`, `AddSingleton<T>()` добавляют дескрипторы. Можно регистрировать интерфейс к реализации: `services.AddScoped<IOrderRepo, OrderRepo>()`. Можно фабрику: `services.AddSingleton<IFoo>(sp => new Foo(...))`.

**Когда какой выбирать?** По умолчанию — `Scoped` для сервисов с данными, `Transient` для лёгких сервисов без состояния, `Singleton` для глобальных долгоживущих вещей. Если сомневаетесь — берите `Scoped`, он самый безопасный в смысле утечек.

**Captive dependency (захваченная зависимость)** — главная ловушка. Если `Singleton` зависит от `Scoped`-сервиса, то scoped-зависимость «захватывается» навсегда и превращается в де-факто singleton, разделяя состояние между всеми запросами. Это редко то, что нужно, и часто ведёт к багам с данными. Контейнер **не detects** это автоматически (только `ValidateScopes` в Development выбрасывает исключение, если singleton разрешает scoped из корневого контейнера). Правило: **зависимость не может жить дольше потребителя**. Singleton → только singleton-зависимости. Scoped → scoped или singleton. Transient → что угодно (но scoped-зависимость в transient даст новый экземпляр scoped на каждое создание transient, что редко желательно).

**Async service / async-исключения** — DI не запрещает async-методы в конструкторе, но **конструктор не может быть `async`** и не должен возвращать `Task`. Асинхронная инициализация выносится в отдельный метод или в паттерн `IAsyncInitializer`/фабрику. Важно: если сервис имеет `IDisposable`/`IAsyncDisposable`, контейнер **автоматически диспозит** его при освобождении scope/root в обратном порядке создания. Для `IServiceScopeFactory` всегда диспозите созданный scope через `using` или `await using`.

#### Theory (EN)

The built-in DI container in .NET (`Microsoft.Extensions.DependencyInjection`) is a "bookkeeper" that knows how to create objects and how long they should live. Registration goes through `IServiceCollection`, resolution through `IServiceProvider`. Every service is registered with one of three **lifetimes**: `Transient`, `Scoped`, or `Singleton`.

**Transient** — a new instance **every time** someone asks for the service. Analogy: a paper cup — drink your coffee, throw it away. Good for lightweight, stateless services: formatters, mappers, helper classes. The main downside is GC pressure if the object is expensive.

**Scoped** — one instance **per scope**. Analogy: a cafeteria tray — while you are inside the cafeteria your food sits on one tray; when you leave, the tray is returned. The most common lifetime in web apps: one scope is created per HTTP request. Perfect for `DbContext`, repositories, unit-of-work — anything tied to a single transaction. Outside a web request you create a scope manually via `IServiceScopeFactory.CreateScope()`.

**Singleton** — exactly one instance **for the whole application**. Analogy: a bank safe — one for all clients, access synchronized. Good for caches, configuration, heavy factories, `HttpClient` through `IHttpClientFactory`. Caution: it keeps state for the whole process lifetime, so it must be **thread-safe**.

**`IServiceCollection`** is a collection of `ServiceDescriptor` records. Methods `AddTransient<T>()`, `AddScoped<T>()`, `AddSingleton<T>()` add descriptors. You can bind an interface to an implementation: `services.AddScoped<IOrderRepo, OrderRepo>()`. You can pass a factory: `services.AddSingleton<IFoo>(sp => new Foo(...))`.

**When to choose which?** Default to `Scoped` for data services, `Transient` for lightweight stateless services, `Singleton` for global long-lived things. When in doubt pick `Scoped` — it is the safest regarding leaks.

**Captive dependency** is the main trap. If a `Singleton` depends on a `Scoped` service, the scoped dependency is "captured" forever and becomes a de-facto singleton, sharing state across all requests — rarely what you want and often the source of data bugs. The container does **not** detect this automatically (only `ValidateScopes` in Development throws if a singleton resolves a scoped service from the root container). The rule: **a dependency cannot live longer than its consumer**. Singleton → only singleton dependencies. Scoped → scoped or singleton. Transient → anything, but a scoped dependency inside a transient gives a fresh scoped instance on every transient creation, which is rarely desired.

**Async service / async caveats** — DI does not forbid async methods, but **a constructor cannot be `async`** and must not return a `Task`. Asynchronous initialization is moved to a separate method or to an `IAsyncInitializer` pattern / factory. Important: if a service implements `IDisposable`/`IAsyncDisposable`, the container **automatically disposes** it when the scope/root is released, in reverse creation order. For `IServiceScopeFactory` always dispose the created scope with `using` or `await using`.

#### Пример кода / Code Example

```csharp
// C# 12 / .NET 8 — встроенный DI: lifetimes, captive dependency, scope factory
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;

// Интерфейсы / Interfaces
public interface IGuidPrinter
{
    void Print(string label); // Вывести свой Guid / Print own Guid
}

public interface IRepository
{
    Guid Id { get; } // Идентификатор экземпляра / Instance id
}

public interface ICache
{
    Guid Id { get; }
}

// --- Реализации с разным lifetime / Implementations with different lifetimes ---

// Transient: новый экземпляр каждый раз / new instance every time
public sealed class TransientPrinter : IGuidPrinter
{
    private readonly Guid _id = Guid.NewGuid();
    public void Print(string label) =>
        Console.WriteLine($"{label}: Transient  id={_id}");
}

// Scoped: один экземпляр на scope / one instance per scope
public sealed class ScopedRepository : IRepository
{
    public Guid Id { get; } = Guid.NewGuid();
}

// Singleton: один на всё приложение / one for the whole app
public sealed class SingletonCache : ICache
{
    public Guid Id { get; } = Guid.NewGuid();
}

// --- Безопасное использование Scoped из фонового сервиса ---
// Safe usage of Scoped from a singleton/background service
public sealed class ReportWorker : BackgroundService
{
    // ВНИМАНИЕ: НЕЛЬЗЯ注入ить Scoped-сервис напрямую в Singleton —
    // это captive dependency. Внедряем фабрику scope.
    // Do NOT inject a Scoped service directly into a Singleton —
    // that is a captive dependency. Inject the scope factory instead.
    private readonly IServiceScopeFactory _scopeFactory;

    public ReportWorker(IServiceScopeFactory scopeFactory) =>
        _scopeFactory = scopeFactory;

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            // Каждый цикл — свой scope, свой ScopedRepository.
            // Each iteration gets its own scope and its own ScopedRepository.
            await using AsyncServiceScope scope = _scopeFactory.CreateAsyncScope();
            var repo = scope.ServiceProvider.GetRequiredService<IRepository>();
            Console.WriteLine($"Worker report, repo id={repo.Id}");

            await Task.Delay(TimeSpan.FromSeconds(5), stoppingToken);
        }
    }
}

// --- Регистрация / Registration ---
public static IServiceCollection AddAppServices(this IServiceCollection services)
{
    services.AddHostedService<ReportWorker>();          // hosted = singleton
    services.AddTransient<IGuidPrinter, TransientPrinter>();
    services.AddScoped<IRepository, ScopedRepository>();
    services.AddSingleton<ICache, SingletonCache>();
    return services;
}

// --- Демонстрация / Demo ---
public static class Demo
{
    public static async Task RunAsync()
    {
        var host = Host.CreateDefaultBuilder()
            .ConfigureServices((_, s) => s.AddAppServices())
            .Build();

        // ValidateScopes включён в Development: если бы ReportWorker
        // напрямую принял IRepository, здесь выбросило бы исключение.
        // ValidateScopes is on in Development: if ReportWorker took
        // IRepository directly, this would throw.

        var sp = host.Services;

        // Transient: два разрешения — два разных id
        // Transient: two resolutions — two different ids
        sp.GetRequiredService<IGuidPrinter>().Print("A");
        sp.GetRequiredService<IGuidPrinter>().Print("B");

        // Singleton: всегда один id
        // Singleton: always the same id
        var cache1 = sp.GetRequiredService<ICache>();
        var cache2 = sp.GetRequiredService<ICache>();
        Console.WriteLine($"Singleton same? {cache1.Id == cache2.Id}");

        // Scoped: два scope — два id, внутри scope — один
        // Scoped: two scopes — two ids, within a scope — one
        await using (var s1 = sp.CreateAsyncScope())
        await using (var s2 = sp.CreateAsyncScope())
        {
            var r1a = s1.ServiceProvider.GetRequiredService<IRepository>();
            var r1b = s1.ServiceProvider.GetRequiredService<IRepository>();
            var r2  = s2.ServiceProvider.GetRequiredService<IRepository>();
            Console.WriteLine($"Scoped same in scope? {r1a.Id == r1b.Id}");  // True
            Console.WriteLine($"Scoped diff across scopes? {r1a.Id == r2.Id}"); // False
        }

        await host.StopAsync();
    }
}
```

#### Best Practices
- По умолчанию используйте `Scoped` для сервисов, работающих с данными; `Transient` — для лёгких без состояния; `Singleton` — для глобальных и потокобезопасных.
- Никогда не внедряйте `Scoped`-сервис в `Singleton` напрямую — используйте `IServiceScopeFactory` и создавайте scope на каждую операцию.
- Включайте `ValidateScopes = true` и `ValidateOnBuild = true` в Development, чтобы ловить ошибки lifetimes на старте.
- Реализуйте `IAsyncDisposable` для сервисов с async-ресурсами (DB-соединения, streams) — контейнер вызовет его при диспозе scope.
- Регистрируйте по интерфейсу (`AddScoped<IFoo, Foo>()`), а не по конкретному классу — это упрощает замену и тестирование.

- Default to `Scoped` for data services, `Transient` for lightweight stateless ones, `Singleton` for global and thread-safe ones.
- Never inject a `Scoped` service into a `Singleton` directly — use `IServiceScopeFactory` and create a scope per operation.
- Enable `ValidateScopes = true` and `ValidateOnBuild = true` in Development to catch lifetime errors at startup.
- Implement `IAsyncDisposable` for services holding async resources (DB connections, streams) — the container calls it on scope disposal.
- Register against an interface (`AddScoped<IFoo, Foo>()`), not a concrete class — easier to swap and to test.

#### Частые ошибки / Common Mistakes
- [RU] Захват Scoped в Singleton (captive dependency) → Внедряйте `IServiceScopeFactory` и создавайте scope на каждое использование.
- [RU] Внедрение `DbContext` в `BackgroundService` напрямую → Тот же captive-баг; создавайте scope через фабрику внутри `ExecuteAsync`.
- [RU] Конструктор `async Task` в сервисе → Невозможно в C#; выносите инициализацию в метод `InitAsync()` или фабрику.
- [RU] Singleton с изменяемым состоянием без синхронизации → Делайте класс потокобезопасным (`lock`, `ConcurrentDictionary`) или используйте `Scoped`.
- [RU] Забыли `using` на `CreateScope()` → Утечка scoped-ресурсов (например, не диспозятся `DbContext`); всегда `using`/`await using`.
- [RU] Регистрация `AddSingleton` с реализацией, имеющей Scoped-зависимость → ValidateScopes выбросит исключение в Dev; пересмотрите lifetime.
- [mistake] Capturing Scoped into Singleton (captive dependency) → Inject `IServiceScopeFactory` and create a scope per use.
- [mistake] Injecting `DbContext` into a `BackgroundService` directly → Same captive bug; create a scope via the factory inside `ExecuteAsync`.
- [mistake] `async Task` constructor in a service → Impossible in C#; move init to an `InitAsync()` method or a factory.
- [mistake] Singleton with mutable state and no synchronization → Make it thread-safe (`lock`, `ConcurrentDictionary`) or use `Scoped`.
- [mistake] Forgot `using` on `CreateScope()` → Leaks scoped resources (e.g. `DbContext` not disposed); always `using`/`await using`.
- [mistake] Registering `AddSingleton` with an implementation that has a Scoped dependency → ValidateScopes throws in Dev; rethink the lifetime.

#### Чек-лист самопроверки / Self-check Checklist
- [ ] Я знаю разницу между Transient, Scoped и Singleton и могу привести аналог для каждого.
- [ ] Я понимаю, что такое captive dependency, и знаю правило «зависимость не живёт дольше потребителя».
- [ ] Я умею безопасно разрешать Scoped-сервис из BackgroundService через `IServiceScopeFactory`.
- [ ] Я знаю, что `ValidateScopes` и `ValidateOnBuild` помогают ловить ошибки lifetimes на старте.
- [ ] Я помню, что конструктор не может быть `async`, и знаю, как обрабатывать async-инициализацию.
- [ ] Я всегда диспожу созданные scope через `using`/`await using`.
- [ ] I know the difference between Transient, Scoped and Singleton and can give an analogy for each.
- [ ] I understand what a captive dependency is and know the rule "a dependency cannot outlive its consumer".
- [ ] I can safely resolve a Scoped service from a BackgroundService via `IServiceScopeFactory`.
- [ ] I know `ValidateScopes` and `ValidateOnBuild` help catch lifetime errors at startup.
- [ ] I remember a constructor cannot be `async` and know how to handle async initialization.
- [ ] I always dispose created scopes with `using`/`await using`.

#### Ресурсы / Resources
- [Microsoft Learn — https://learn.microsoft.com/dotnet/core/extensions/dependency-injection]
- [Dependency injection in .NET (guidelines) — https://learn.microsoft.com/dotnet/core/extensions/dependency-injection-guidelines]
- [DI in ASP.NET Core — https://learn.microsoft.com/aspnet/core/fundamentals/dependency-injection]

---

[⬆ К модулю M13](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
