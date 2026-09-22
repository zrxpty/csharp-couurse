[⬆ К модулю M16](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M16-L02: DI-контейнер вглубь, lifetimes, антипаттерны (captive dependency) / DI container deep dive, lifetimes, anti-patterns (captive dependency)

**Модуль / Module:** M16
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

DI-контейнер в .NET — это не «магия», а реестр описаний сервисов (`ServiceDescriptor`), который умеет строить граф объектов по запросу. Контейнер `Microsoft.Extensions.DependencyInjection` знает три времени жизни: `Singleton`, `Scoped` и `Transient`. Чтобы понимать, когда контейнер ломается, нужно представлять себе, что каждое время жизни — это «правило владения» экземпляром, а не просто пометка.

Представь кафе. `Transient` — бумажный стаканчик: каждый заказывает свой, после использования выбрасывают. `Scoped` — поднос: один на визит посетителя, его можно передавать по цепочке заказов внутри визита, но после ухода клиента его моют. `Singleton` — кухонный комбайн: один на всё кафе, живёт, пока живёт сам контейнер.

Главная тема урока — **captive dependency** (захваченная зависимость). Это ситуация, когда сервис с более долгим временем жизни держит сервис с более коротким. Классический пример: `Singleton` зависит от `Scoped`. Контейнер не запрещает такую регистрацию, и это — ловушка. Синглтон создаётся один раз, конструктор вызывается один раз, и `Scoped`-зависимость в реальности тоже создаётся один раз и навсегда «запаздывает» в синглтоне. Каждый новый scope получает тот же экземпляр, который синглтон удержал. Результат: данные «перетекают» между запросами, `DbContext` течёт, кэш растёт без очистки, тесты становятся недетерминированными.

Антипаттерн проявляется в трёх вариантах. **Прямой захват**: `Singleton → Scoped` через конструктор — контейнер даже кидает исключение в режиме `ValidateScopes=true`, но в проде валидация часто выключена. **Скрытый захват через фабрику**: синглтон вызывает `provider.GetRequiredService<ScopedService>()` прямо из root-провайдера, минуя scope. **Захват через `IServiceProvider`**: синглтон хранит сам `IServiceProvider` и резолвит из него по мере надобности — если это root, scoped снова «прилипает».

Правильный способ получить scoped-сервис из долгоживущего объекта — `IServiceScopeFactory`. Фабрика сама синглтон-совместима: ты создаёшь `_scopeFactory.CreateScope()`, берёшь `scope.ServiceProvider`, резолвишь, работаешь,.dispose(). Для периодических задач это стандарт: фоновый `IHostedService` держит `IServiceScopeFactory` и в каждом цикле создаёт новый scope.

Вторая тема — **resolve из root**. Вызов `app.Services.GetRequiredService<ScopedService>()` в `Program.cs` создаёт scope автоматически для scoped-сервиса, но это обманка: экземпляр живёт пока живёт root, то есть по сути становится синглтоном. То же самое в `IHostedService.StartAsync` без явного scope. Правило: никогда не резолв scoped/transient с состоянием прямо из root; всегда открывай scope.

Третья тема — **async resolution**. Методы построения графа синхронны (`GetRequiredService`). Если в конструкторе ты запускаешь асинхронную работу (I/O, БД) и не дожидаешься её правильно, ты получаешь `async void`-эффекты и race condition. Решение: тяжёлую инициализацию выносить в `IHostedService` или в `async factory method`, а конструктор оставить лёгким. Не делай `async`-свойства и не вызывай `.Result`/`.Wait()` — это тупик в синхронном контексте.

Контейнер проверяет корректность регистраций через `BuildServiceProvider(new ServiceProviderOptions { ValidateOnBuild = true, ValidateScopes = true })`. Включай это в dev-окружении и в тестах: он поймает captive dependency ещё на старте.

#### Theory (EN)

The .NET DI container is not magic — it is a registry of `ServiceDescriptor` entries that knows how to build object graphs on demand. `Microsoft.Extensions.DependencyInjection` recognises three lifetimes: `Singleton`, `Scoped`, and `Transient`. To understand when the container breaks, think of each lifetime as an ownership rule rather than a label.

Imagine a coffee shop. `Transient` is a paper cup: every customer gets their own, it is thrown away after use. `Scoped` is a tray: one per visit, passed along the chain of orders within that visit, then washed when the customer leaves. `Singleton` is the kitchen blender: one for the whole café, alive as long as the café (the root container) is alive.

The central topic of this lesson is the **captive dependency**. It occurs when a longer-lived service holds a shorter-lived one. The classic case: a `Singleton` depends on a `Scoped` service. The container does not forbid this registration — that is the trap. The singleton is constructed once, its constructor runs once, and the `Scoped` dependency is in practice also constructed once and forever “frozen” inside the singleton. Every new scope gets the same instance that the singleton captured. The result: state leaks between requests, a `DbContext` grows and throws concurrency errors, caches never reset, tests become non-deterministic.

The anti-pattern shows up in three flavours. **Direct capture**: `Singleton → Scoped` through the constructor — the container even throws when `ValidateScopes=true`, but validation is often disabled in production. **Hidden capture through a factory**: a singleton calls `provider.GetRequiredService<ScopedService>()` directly from the root provider, bypassing any scope. **Capture through `IServiceProvider`**: a singleton stores the `IServiceProvider` itself and resolves from it on demand — if that provider is the root, scoped services again stick forever.

The correct way to obtain a scoped service from a long-lived object is `IServiceScopeFactory`. The factory itself is singleton-safe: you call `_scopeFactory.CreateScope()`, take `scope.ServiceProvider`, resolve, do the work, and `Dispose()`. For recurring background work this is the standard pattern: a hosted service holds `IServiceScopeFactory` and opens a fresh scope on every cycle.

The second theme is **resolving from the root**. Calling `app.Services.GetRequiredService<ScopedService>()` in `Program.cs` does create a scope for a scoped service, but it is misleading: that instance lives as long as the root, so it is effectively a singleton. The same happens inside `IHostedService.StartAsync` without an explicit scope. The rule: never resolve stateful scoped/transient services directly from the root; always open a scope.

The third theme is **async resolution**. Graph construction methods are synchronous (`GetRequiredService`). If a constructor starts asynchronous work (I/O, database) and you do not await it correctly, you get `async void` effects and race conditions. The fix: move heavy initialisation into an `IHostedService` or an `async factory method`, and keep the constructor cheap. Avoid `async` properties and never call `.Result` or `.Wait()` — that is a deadlock waiting to happen in a synchronisation context.

The container can check registration correctness with `BuildServiceProvider(new ServiceProviderOptions { ValidateOnBuild = true, ValidateScopes = true })`. Turn this on in development and in tests: it catches captive dependencies before the app even serves a request.

#### Пример кода / Code Example

```csharp
// C# 12 / .NET 8 — DI lifetimes, captive dependency, IServiceScopeFactory
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;

// --- Сервис с состоянием (scoped) / Stateful scoped service ---
public sealed class UserSession // scoped: один экземпляр на scope
{
    public Guid SessionId { get; } = Guid.NewGuid();
}

// --- Правильный фоновой сервис: держит фабрику, а не сам scoped ---
public sealed class SessionReporter : BackgroundService
{
    private readonly IServiceScopeFactory _scopeFactory; // синглтон-совместима / singleton-safe

    public SessionReporter(IServiceScopeFactory scopeFactory) =>
        _scopeFactory = scopeFactory;

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            // Создаём новый scope на каждой итерации / new scope per iteration
            using var scope = _scopeFactory.CreateScope();
            var session = scope.ServiceProvider.GetRequiredService<UserSession>();

            Console.WriteLine(
                $"[RU] Сессия в scope: {session.SessionId}  " +
                $"[EN] Session in scope: {session.SessionId}");

            await Task.Delay(TimeSpan.FromSeconds(1), stoppingToken);
        }
    }
}

// --- НЕправильно: синглтон, который захватывает scoped (captive dependency) ---
// НЕ делайте так / DO NOT do this:
//
// public sealed class CaptiveSingleton
// {
//     private readonly UserSession _session; // захват! / captured!
//     public CaptiveSingleton(UserSession session) => _session = session;
// }
//
// services.AddSingleton<CaptiveSingleton>();   // сервис — singleton
// services.AddScoped<UserSession>();           // зависимость — scoped
// → _session создастся ОДИН раз и «залипнет» в синглтоне навсегда.

// --- Регистрация с проверкой scopes / Registration with scope validation ---
var builder = Host.CreateApplicationBuilder(args);

builder.Services.AddScoped<UserSession>();
builder.Services.AddHostedService<SessionReporter>();

// В dev включаем валидацию, чтобы captive dependency упал на старте.
// In dev we enable validation so a captive dependency fails at startup.
var host = builder.Build(); // CreateApplicationBuilder уже включает ValidateScopes в dev.
await host.RunAsync();
```

#### Best Practices
- Регистрируй сервисы с самым коротким временем жизни, которое реально нужно: начинай с `Transient`, переходи к `Scoped`, и только потом к `Singleton`.
- Из долгоживущих объектов (синглтон, фоновый сервис) получай scoped-зависимости только через `IServiceScopeFactory.CreateScope()`.
- Включай `ValidateScopes=true` и `ValidateOnBuild=true` в dev- и тест-окружении, чтобы ловить captive dependency на старте.
- Держи конструкторы лёгкими; тяжёлую и асинхронную инициализацию выноси в `IHostedService` или асинхронную фабрику.
- Не резолви stateful scoped/transient-сервисы напрямую из root-провайдера.
- Prefer the shortest lifetime that actually works: start with `Transient`, move to `Scoped`, only then to `Singleton`.
- From long-lived objects obtain scoped dependencies solely through `IServiceScopeFactory.CreateScope()`.
- Enable `ValidateScopes=true` and `ValidateOnBuild=true` in development and test environments to catch captive dependencies at startup.
- Keep constructors cheap; move heavy or asynchronous initialisation into an `IHostedService` or an async factory.
- Never resolve stateful scoped/transient services directly from the root provider.

#### Частые ошибки / Common Mistakes
- `Singleton` держит `Scoped` через конструктор → используй `IServiceScopeFactory` и создавай scope на каждое использование (RU).
- Резолв scoped-сервиса из root в `Program.cs` «на старте» → открывай явный `IServiceScope` или используй hosted service (RU).
- `async void` или `.Result` в конструкторе → вынеси инициализацию в `IHostedService.StartAsync` и используй `await` (RU).
- Регистрация всех сервисов как `Singleton` «чтобы было быстрее» → оценивай реальное время жизни по состоянию, не по скорости (RU).
- Same instance shared between requests after registering `Scoped` as `Singleton` → register with the lifetime that matches the service’s state (EN).
- A `Singleton` holds a `Scoped` dependency via the constructor → resolve scoped services through `IServiceScopeFactory` per use (EN).
- Resolving a scoped service from the root provider at startup → open an explicit `IServiceScope` or use a hosted service (EN).
- Calling `.Result` or using `async void` in a constructor → move initialisation into `IHostedService.StartAsync` and `await` it (EN).
- Registering everything as `Singleton` for “performance” → choose lifetime by state, not by speed (EN).

#### Чек-лист самопроверки / Self-check Checklist
- [ ] Ни один `Singleton` не принимает `Scoped`-зависимость в конструктор (RU).
- [ ] Все долгоживущие объекты используют `IServiceScopeFactory` для scoped-сервисов (RU).
- [ ] В dev/test включены `ValidateScopes` и `ValidateOnBuild` (RU).
- [ ] Ни один конструктор не делает блокирующих вызовов I/O или `.Result` (RU).
- [ ] Stateful scoped-сервисы не резолвятся напрямую из root-провайдера (RU).
- [ ] No `Singleton` accepts a `Scoped` dependency in its constructor (EN).
- [ ] All long-lived objects use `IServiceScopeFactory` for scoped services (EN).
- [ ] `ValidateScopes` and `ValidateOnBuild` are enabled in dev/test (EN).
- [ ] No constructor performs blocking I/O or `.Result` calls (EN).
- [ ] Stateful scoped services are never resolved directly from the root provider (EN).

#### Ресурсы / Resources
- [Microsoft Learn — https://learn.microsoft.com/dotnet/core/extensions/dependency-injection-guidelines](https://learn.microsoft.com/dotnet/core/extensions/dependency-injection-guidelines)

---

[⬆ К модулю M16](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
