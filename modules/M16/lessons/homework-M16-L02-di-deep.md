---
[← К уроку M16-L02](lesson-M16-L02-di-deep.md) | [⬆ К модулю M16](../README.md) | [Следующее ДЗ →](homework-M16-L03-scrutor.md)
---

### Домашнее задание M16-L02: DI-контейнер вглубь, lifetimes, антипаттерны (captive dependency) / Homework M16-L02: DI container deep dive, lifetimes, anti-patterns (captive dependency)

**Урок / Lesson:** M16-L02
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 4/5
**Цель / Goal:** (RU) Научиться осознанно выбирать время жизни сервисов, выявлять и устранять captive dependency, правильно получать scoped-зависимости из долгоживущих объектов через `IServiceScopeFactory`, включать валидацию контейнера и писать детерминированные тесты на DI. (EN) Learn to choose service lifetimes deliberately, detect and eliminate captive dependencies, correctly obtain scoped dependencies from long-lived objects via `IServiceScopeFactory`, enable container validation, and write deterministic DI tests.

#### Связь с уроком / Connection to the lesson
(RU) Урок объясняет три времени жизни (`Transient`, `Scoped`, `Singleton`) через «кафе-метафору» и центральную ловушку — captive dependency, когда долгоживущий сервис захватывает короткоживущий. В ДЗ вы воспроизведёте все три варианта антипаттерна (прямой захват, скрытый через фабрику, захват через `IServiceProvider`), увидите их последствия в тестах и исправите через `IServiceScopeFactory` и валидацию `ValidateScopes`/`ValidateOnBuild`.
(EN) The lesson explains the three lifetimes through a coffee-shop metaphor and the central trap — the captive dependency, where a longer-lived service captures a shorter-lived one. In this homework you will reproduce all three flavours of the anti-pattern (direct capture, hidden factory capture, capture via `IServiceProvider`), observe their consequences in tests, and fix them with `IServiceScopeFactory` plus `ValidateScopes`/`ValidateOnBuild`.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация

Вы поддерживаете внутренний сервис уведомлений `Notifier` для платформы курсов. Сервис запускается как `IHostedService`, раз в несколько секунд читает «очередь» ожидающих уведомлений из репозитория (имитация БД) и отправляет их через канал. Репозиторий имеет состояние — он держит `Guid CorrelationId`, который генерируется на каждое подключение к БД, чтобы в логах можно было связать все запросы одной «сессии». В проде вы заметили странный баг: `CorrelationId` остаётся одним и тем же часами, хотя по логике должен меняться на каждое новое подключение. Логи разных запусков «смешиваются», в тестах случайные падения, в метриках — паразитный рост потребления памяти.

При разборе вы понимаете, что корень проблемы — не бизнес-логика, а DI. Фоновый сервис зарегистрирован как hosted service (он singleton-подобен по времени жизни: один экземпляр на всё приложение), а репозиторий — как `Scoped`. Сервис захватил репозиторий в конструкторе, и `CorrelationId` навсегда «залип» в одном экземпляре. Это и есть captive dependency — захваченная зависимость из урока. Контейнер молчаливо пропустил такую регистрацию, потому что в проде `ValidateScopes=false`. В dev-окружении с валидацией та же регистрация упала бы ещё на старте.

Вам нужно: воспроизвести все три варианта captive dependency из урока (прямой захват через конструктор, скрытый через `provider.GetRequiredService` из root, захват через сохранённый `IServiceProvider`), написать детерминированные тесты, которые ловят баг по поведению (а не по структуре), и затем переписать решение правильно — через `IServiceScopeFactory`, с включённой валидацией и легкими конструкторами. Параллельно вы разберёте вторую тему урока — resolve из root в `Program.cs` — и третью — async-инициализацию в конструкторе, реализовав корректный `IHostedService` с тяжёлым стартом.

#### Что нужно сделать (пошагово)

1. Создайте решение и три проекта. Выполните:
   ```
   dotnet new sln -n NotifierLab
   dotnet new console -n NotifierLab.App -o src/NotifierLab.App --framework net8.0
   dotnet new xunit -n NotifierLab.Tests -o tests/NotifierLab.Tests --framework net8.0
   dotnet new classlib -n NotifierLab.Core -o src/NotifierLab.Core --framework net8.0
   dotnet sln add src/NotifierLab.Core src/NotifierLab.App tests/NotifierLab.Tests
   dotnet add src/NotifierLab.App reference src/NotifierLab.Core
   dotnet add tests/NotifierLab.Tests reference src/NotifierLab.Core
   dotnet add src/NotifierLab.App package Microsoft.Extensions.Hosting
   dotnet add tests/NotifierLab.Tests package Microsoft.Extensions.Hosting
   ```
   Включите `<Nullable>enable</Nullable>` и `<TreatWarningsAsErrors>true</TreatWarningsAsErrors>` в каждом `.csproj`.

2. В `NotifierLab.Core` смоделируйте состояние. Создайте класс `PendingNotification` (record с `Guid Id`, `string Payload`). Создайте `INotificationRepository` с методом `Task<IReadOnlyList<PendingNotification>> FetchBatchAsync(CancellationToken ct)`. Сделайте реализацию `InMemoryNotificationRepository`: она хранит список `PendingNotification`, генерирует `Guid CorrelationId = Guid.NewGuid()` в конструкторе (это и есть «подключение к БД»), а `FetchBatchAsync` возвращает до 5 элементов, удаляя их из внутреннего списка, и логирует (через `Console.WriteLine`) текущий `CorrelationId`. В логе должно быть видно, какой `CorrelationId` обслужил запрос.

3. Создайте `INotificationChannel` с `Task SendAsync(PendingNotification n, CancellationToken ct)` и тривиальную реализацию `ConsoleChannel`, которая пишет в консоль `"[sent] {Id} {Payload}"`.

4. **Воспроизведите антипаттерн №1 — прямой захват.** Создайте `CaptiveNotifier : BackgroundService`, который в конструкторе принимает `INotificationRepository` и сохраняет его в поле. Реализация `ExecuteAsync`: цикл с `Task.Delay(1_000, ct)`, в каждой итерации `_repo.FetchBatchAsync(ct)` и `SendAsync` по каждому. В `Program.cs` зарегистрируйте `services.AddScoped<INotificationRepository, InMemoryNotificationRepository>()`, `services.AddSingleton<INotificationChannel, ConsoleChannel>()`, `services.AddHostedService<CaptiveNotifier>()`. Запустите `dotnet run --project src/NotifierLab.App`. Убедитесь по логам, что `CorrelationId` ОДИН И ТОТ ЖЕ во всех итерациях — это и есть captive dependency.

5. **Воспроизведите антипаттерн №2 — скрытый захват через фабрику.** Создайте `FactoryCaptiveNotifier : BackgroundService`, который принимает `IServiceProvider` и в `ExecuteAsync` вызывает `_provider.GetRequiredService<INotificationRepository>()` без создания scope. Зарегистрируйте его вместо предыдущего. Запустите, убедитесь, что `CorrelationId` снова «залип» — потому что `_provider` это root, а root не имеет scope.

6. **Воспроизведите антипаттерн №3 — захват через сохранённый `IServiceProvider`.** Создайте `ProviderCaptiveNotifier : BackgroundService`, который принимает `IServiceProvider`, но хранит `_provider` и резолвит репозиторий «лениво» один раз в поле через `Lazy<INotificationRepository>`. Поведение должно быть тем же: один `CorrelationId` навсегда.

7. **Напишите тесты, которые ловят баг по поведению.** В `tests/NotifierLab.Tests` создайте `CaptiveDependencyTests`. Не тестируйте структуру регистраций напрямую — тестируйте наблюдаемое поведение: после двух «итераций» цикла `CorrelationId` в логах репозитория должен быть РАЗНЫМ, если репозиторий scoped. Используйте подмену: создайте `RecordingRepository`, который в список записывает каждый использованный `CorrelationId`, и проверьте, что за две итерации в списке два разных значения. Чтобы можно было «прокрутить» цикл детерминированно, вынесите рабочую функцию `DoOneCycleAsync(IServiceScope scope, CancellationToken ct)` из `BackgroundService` в отдельный тестируемый класс `NotifierEngine` — это и есть правильная декомпозиция (hosted service тонок, логика в engine).

8. **Исправьте решение.** Создайте `ScopedNotifierEngine`, который принимает `IServiceScopeFactory` (singleton-safe) и метод `RunOneCycleAsync(CancellationToken ct)`, создающий `using var scope = _scopeFactory.CreateScope()`, резолвящий репозиторий и канал из `scope.ServiceProvider` и выполняющий один цикл. Создайте `CorrectNotifier : BackgroundService`, который просто вызывает `RunOneCycleAsync` в цикле с задержкой. Зарегистрируйте `services.AddScoped<INotificationRepository, ...>()`, `services.AddSingleton<INotificationChannel, ...>()`, `services.AddSingleton<NotifierEngine>()` (или hosted service с инъекцией `IServiceScopeFactory`), `services.AddHostedService<CorrectNotifier>()`.

9. **Включите валидацию контейнера.** В `Program.cs` используйте `Host.CreateApplicationBuilder(args)` (он в dev включает `ValidateScopes` и `ValidateOnBuild`). Дополнительно покажите явно: `var host = builder.Build();` и проверьте, что при попытке зарегистрировать антипаттерн №1 и старте в dev-окружении контейнер кидает `InvalidOperationException` про captive dependency. Зафиксируйте это в тесте: соберите `ServiceCollection`, добавьте антипаттерн, вызовите `BuildServiceProvider(new ServiceProviderOptions { ValidateScopes = true, ValidateOnBuild = true })` и `Assert.Throws<InvalidOperationException>(...)`.

10. **Воспроизведите тему «resolve из root».** В `Program.cs` добавьте комментированный опасный фрагмент: `// var repo = app.Services.GetRequiredService<INotificationRepository>();` — объясните в комментарии русским языком, почему это превращает scoped в фактический singleton. Покажите правильную альтернативу: `using var scope = app.Services.CreateScope(); var repo = scope.ServiceProvider.GetRequiredService<INotificationRepository>();`.

11. **Воспроизведите тему «async resolution».** Создайте `HeavyInitService` с конструктором, который НЕ делает I/O, и метод `InitializeAsync` (тяжёлая асинхронная инициализация). Создайте `InitHostedService : IHostedService`, который в `StartAsync` вызывает `await _heavy.InitializeAsync(ct)`. Покажите в комментарии, почему вызывать `InitializeAsync().GetAwaiter().GetResult()` в конструкторе — это антипаттерн (тупик в синхронном контексте, race condition).

12. Запустите `dotnet test` — все тесты зелёные. Запустите `dotnet run --project src/NotifierLab.App` — в логах `CorrelationId` меняется каждую секунду (новый scope → новый репозиторий → новый `CorrelationId`). Зафиксируйте вывод в `README.md` в корне решения (2–3 абзаца, что вы наблюдали).

#### Требования к решению

- Целевой фреймворк `net8.0`, язык C# 12. Используйте top-level statements в `Program.cs`, collection expressions (`[]`, `..`), pattern matching, `record` для DTO, `required`-члены или `init`-сеттеры там, где это уместно.
- Проект должен собираться без предупреждений: `<TreatWarningsAsErrors>true</TreatWarningsAsErrors>`, `<Nullable>enable</Nullable>`.
- Запрещено: хранить `Scoped`-зависимость в поле `Singleton`/hosted-сервиса; резолвить scoped из root без scope; вызывать `.Result`/`.Wait()`/`async void` в конструкторе; регистрировать все сервисы как `Singleton` «для скорости».
- Обязательно: правильное решение использует `IServiceScopeFactory` и создаёт scope на каждый цикл/использование; валидация `ValidateScopes` и `ValidateOnBuild` включена в dev- и тест-окружении; конструкторы лёгкие, тяжёлая инициализация — в `IHostedService.StartAsync`.
- Структура: `NotifierLab.Core` (интерфейсы и реализации сервисов), `NotifierLab.App` (`Program.cs`, `BackgroundService`-классы), `NotifierLab.Tests` (xUnit).
- Все три варианта captive dependency должны присутствовать в коде (можно в отдельных файлах `Captive*`) и быть покрыты тестами, которые демонстрируют баг и/или падают при попытке собрать контейнер с `ValidateScopes=true`.
- README в корне решения должен кратко объяснять наблюдаемый эффект и содержать пример вывода лога.

#### Тонкости и подводные камни

- `IServiceScopeFactory` — сама по себе singleton-safe: её можно безопасно инжектить в синглтоны и hosted-сервисы. А вот сам `IServiceProvider`, который вы получаете как root, — НЕ singleton-safe для scoped-резолва: из него scoped «прилипает».
- `Host.CreateApplicationBuilder` автоматически включает `ValidateScopes` в dev-окружении (`Environment == "Development"`). В проде валидация по умолчанию выключена — поэтому captive dependency может «прорваться» и проявиться только в рантайме. В тестах всегда собирайте провайдер с `new ServiceProviderOptions { ValidateScopes = true, ValidateOnBuild = true }`.
- `BackgroundService` — это `IHostedService`, и его экземпляр создаётся один раз на всё приложение. Любая зависимость, инжектированная в его конструктор, захватывается на всё время жизни приложения. Поэтому единственный правильный способ получить scoped-сервис из `BackgroundService` — `IServiceScopeFactory.CreateScope()` внутри `ExecuteAsync`.
- Если вы резолвите scoped-сервис из root в `Program.cs` (например, чтобы «прогреть» кэш), вы фактически делаете его singletonом: экземпляр живёт, пока живёт root. Правильно — `using var scope = app.Services.CreateScope();` и работа внутри scope.
- `async void` в конструкторе и `.Result`/`.Wait()` — это тупики в синхронном контексте (особенно в старых ASP.NET, но и в тестах с `SynchronizationContext`). Тяжёлую инициализацию выносите в `IHostedService.StartAsync` и `await` её там.
- `ValidateOnBuild=true` ловит не только captive dependency, но и любые неразрешимые регистрации (отсутствующая регистрация, циклические зависимости в некоторых случаях). Включайте и то, и другое.
- Не путайте «lifetime сервиса» и «lifetime зависимости»: сервис может быть `Scoped`, но если он зависит от `Singleton` — это нормально (короткий держит долгий). Ловушка только в обратном направлении: долгий держит короткий.
- `AddHostedService<T>()` регистрирует `T` с временем жизни, которое делает экземпляр доступным через `IHostedService`, но сам экземпляр создаётся один раз. Не кладите состояние запроса в поля hosted-сервиса.
- В тестах не полагайтесь на `Guid.NewGuid()` как на «гарантированно уникальный» — для детерминизма подставляйте свой генератор (`Func<Guid>`) или мокайте репозиторий целиком, проверяя именно `CorrelationId` по поведению.

#### Критерии приёмки

- [ ] Решение `NotifierLab.sln` собирается командой `dotnet build` без ошибок и без предупреждений (warnings as errors).
- [ ] Целевой фреймворк `net8.0`, C# 12, top-level statements в `Program.cs`.
- [ ] Присутствуют все три варианта captive dependency: `CaptiveNotifier` (прямой захват), `FactoryCaptiveNotifier` (через `GetRequiredService` из root), `ProviderCaptiveNotifier` (через сохранённый `IServiceProvider`).
- [ ] При запуске антипаттернов №1–№3 в логах `CorrelationId` остаётся одним и тем же во всех итерациях (зафиксировано в README).
- [ ] Правильное решение `CorrectNotifier` + `ScopedNotifierEngine` использует `IServiceScopeFactory.CreateScope()` на каждый цикл; `CorrelationId` меняется каждую итерацию.
- [ ] В `Program.cs` явно включена валидация через `Host.CreateApplicationBuilder` (dev) и продемонстрирован явный `ServiceProviderOptions { ValidateScopes = true, ValidateOnBuild = true }` в тестах.
- [ ] Тест `Assert.Throws<InvalidOperationException>` падает на сборке контейнера с антипаттерном №1 при `ValidateScopes=true`.
- [ ] Тест по поведению подтверждает, что за две итерации правильного engine в репозитории появилось два разных `CorrelationId`.
- [ ] Тема «resolve из root» отражена: опасный фрагмент закомментирован с пояснением, правильная альтернатива через `CreateScope()` показана.
- [ ] Тема «async resolution» отражена: `InitHostedService` вызывает `await _heavy.InitializeAsync(ct)` в `StartAsync`; в комментарии объяснён антипаттерн `.Result` в конструкторе.
- [ ] Никакой конструктор не делает блокирующих вызовов I/O и не вызывает `.Result`/`.Wait()`/`async void`.
- [ ] Никакой `Singleton`/`BackgroundService` не принимает `Scoped`-зависимость в конструктор.
- [ ] Все тесты `dotnet test` зелёные.
- [ ] README в корне решения содержит 2–3 абзаца наблюдений и пример вывода лога.
- [ ] Код использует collection expressions, pattern matching, `record` для DTO (минимум в двух местах).
- [ ] В коде нет заглушек `TODO`/`FIXME` и закомментированного мусора, кроме целенаправленно показанных антипаттернов с пояснениями.

#### Подсказки (без прямого ответа)

- Подумайте, почему `IServiceScopeFactory` «безопасна» для синглтона, а `IServiceProvider` (root) — нет. Разница в том, кто создаёт экземпляры scoped-сервисов и в каком scope они живут.
- Чтобы протестировать цикл `BackgroundService` детерминированно, вынесите тело одной итерации в отдельный метод, который принимает `IServiceScope` (или `CancellationToken`) и не зависит от таймера. Тогда в тесте можно вызвать метод дважды и проверить поведение.
- Для проверки captive dependency по структуре используйте не «парсинг регистраций», а попытку построить провайдер с `ValidateScopes=true` — контейнер сам скажет, что не так.
- `CorrelationId` — это удобный «маркер» состояния scoped-сервиса. Если он одинаковый между итерациями — вы захватили scoped в долгоживущем объекте.
- Помните, что `AddHostedService<T>` не делает `T` scoped — экземпляр один на приложение.
- Для тяжёлой инициализации спросите себя: «должен ли этот код выполниться до первого HTTP-запроса?» Если да — `IHostedService.StartAsync`, а не конструктор.

#### Эталонное решение (разбор)

```csharp
// C# 12 / .NET 8 — Эталон решения ДЗ M16-L02
// Reference solution for homework M16-L02
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;

namespace NotifierLab.Core;

// --- DTO: одно ожидающее уведомление / a single pending notification ---
public sealed record PendingNotification(Guid Id, string Payload);

// --- Репозиторий: scoped, имеет состояние (CorrelationId) ---
// Repository: scoped, holds state (CorrelationId)
public interface INotificationRepository
{
    Guid CorrelationId { get; } // маркер «подключения» / "connection" marker
    Task<IReadOnlyList<PendingNotification>> FetchBatchAsync(CancellationToken ct);
}

public sealed class InMemoryNotificationRepository : INotificationRepository
{
    private readonly List<PendingNotification> _queue;
    public Guid CorrelationId { get; } = Guid.NewGuid();

    public InMemoryNotificationRepository()
    {
        // стартовый набор / seed queue
        _queue = [
            new(Guid.NewGuid(), "hello-1"),
            new(Guid.NewGuid(), "hello-2"),
            new(Guid.NewGuid(), "hello-3"),
            new(Guid.NewGuid(), "hello-4"),
            new(Guid.NewGuid(), "hello-5"),
            new(Guid.NewGuid(), "hello-6"),
            new(Guid.NewGuid(), "hello-7"),
        ];
    }

    public Task<IReadOnlyList<PendingNotification>> FetchBatchAsync(CancellationToken ct)
    {
        ct.ThrowIfCancellationRequested();
        var batch = _queue.Count > 0
            ? _queue[..Math.Min(5, _queue.Count)] // collection slicing
            : [];
        _queue.RemoveRange(0, batch.Count);
        Console.WriteLine($"[repo] CorrelationId={CorrelationId}, fetched={batch.Count}");
        return Task.FromResult<IReadOnlyList<PendingNotification>>(batch);
    }
}

// --- Канал отправки / send channel ---
public interface INotificationChannel
{
    Task SendAsync(PendingNotification n, CancellationToken ct);
}

public sealed class ConsoleChannel : INotificationChannel
{
    public Task SendAsync(PendingNotification n, CancellationToken ct)
    {
        Console.WriteLine($"[sent] {n.Id} {n.Payload}");
        return Task.CompletedTask;
    }
}

// --- Антипаттерн №1: прямой захват scoped в синглтоне (Captive dependency) ---
// Anti-pattern #1: direct capture of scoped in a singleton
public sealed class CaptiveNotifier : BackgroundService
{
    private readonly INotificationRepository _repo; // ЗАХВАТ! / captured!
    public CaptiveNotifier(INotificationRepository repo) => _repo = repo;

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            var batch = await _repo.FetchBatchAsync(stoppingToken);
            // канал здесь для простоты захардкожен / channel hardcoded for brevity
            foreach (var n in batch) Console.WriteLine($"[captive-sent] {n.Id}");
            await Task.Delay(TimeSpan.FromSeconds(1), stoppingToken);
        }
    }
}

// --- Антипаттерн №2: скрытый захват через root-провайдер ---
// Anti-pattern #2: hidden capture via root provider
public sealed class FactoryCaptiveNotifier : BackgroundService
{
    private readonly IServiceProvider _provider;
    public FactoryCaptiveNotifier(IServiceProvider provider) => _provider = provider;

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            // резолв из root без scope → scoped «прилипает» / resolving from root
            var repo = _provider.GetRequiredService<INotificationRepository>();
            var batch = await repo.FetchBatchAsync(stoppingToken);
            await Task.Delay(TimeSpan.FromSeconds(1), stoppingToken);
        }
    }
}

// --- Правильный engine: scoped через IServiceScopeFactory ---
// Correct engine: scoped via IServiceScopeFactory
public sealed class ScopedNotifierEngine
{
    private readonly IServiceScopeFactory _scopeFactory; // singleton-safe
    private readonly INotificationChannel _channel;       // singleton — можно в поле

    public ScopedNotifierEngine(IServiceScopeFactory scopeFactory, INotificationChannel channel)
    {
        _scopeFactory = scopeFactory;
        _channel = channel; // singleton: безопасно держать / safe to hold
    }

    // Один цикл — один scope. Тело метода тестируется без таймера.
    // One cycle — one scope. The body is testable without a timer.
    public async Task RunOneCycleAsync(CancellationToken ct)
    {
        await using var scope = _scopeFactory.CreateAsyncScope(); // .NET 8: async scope
        var repo = scope.ServiceProvider.GetRequiredService<INotificationRepository>();
        var batch = await repo.FetchBatchAsync(ct);
        foreach (var n in batch)
            await _channel.SendAsync(n, ct);
    }
}

// --- Правильный hosted service: тонкая обёртка над engine ---
// Correct hosted service: thin wrapper over the engine
public sealed class CorrectNotifier : BackgroundService
{
    private readonly ScopedNotifierEngine _engine;
    public CorrectNotifier(ScopedNotifierEngine engine) => _engine = engine;

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            await _engine.RunOneCycleAsync(stoppingToken);
            await Task.Delay(TimeSpan.FromSeconds(1), stoppingToken);
        }
    }
}

// --- Тема «async resolution»: тяжёлая инициализация в StartAsync ---
// "Async resolution": heavy init in StartAsync
public sealed class HeavyInitService
{
    public int State { get; private set; }
    // конструктор лёгкий — никакой I/O / cheap constructor, no I/O
    public Task InitializeAsync(CancellationToken ct)
    {
        State = 42; // имитация тяжёлой работы / imitates heavy work
        return Task.CompletedTask;
    }
}

public sealed class InitHostedService : IHostedService
{
    private readonly HeavyInitService _heavy;
    public InitHostedService(HeavyInitService heavy) => _heavy = heavy;

    public async Task StartAsync(CancellationToken cancellationToken)
    {
        // правильно: await в StartAsync / correct: await in StartAsync
        await _heavy.InitializeAsync(cancellationToken);
    }
    public Task StopAsync(CancellationToken cancellationToken) => Task.CompletedTask;
}
```

Разбор по строкам. `PendingNotification` — `record`, что даёт value-семантику и структурное равенство для DTO, ровно как в best practices урока. `InMemoryNotificationRepository` помечен как scoped-кандидат: у него есть состояние — список `_queue` и `CorrelationId`, который генерируется в конструкторе. Это и есть «подключение к БД»: каждый новый экземпляр = новое подключение = новый `CorrelationId`. `FetchBatchAsync` использует collection slicing `_queue[..n]` (C# 12) и collection expression `[]` для пустого результата — это новые идиомы языка. `CaptiveNotifier` — первый антипаттерн: он принимает `INotificationRepository` в конструкторе. Поскольку `BackgroundService` создаётся один раз, репозиторий тоже создаётся один раз и «залипает»: все итерации видят один `CorrelationId`. Это ровно «прямой захват» из урока, и контейнер с `ValidateScopes=true` упадёт ещё на `Build`. `FactoryCaptiveNotifier` — второй антипаттерн: он хранит `IServiceProvider` (root) и резолвит репозиторий «на месте». Кажется, что мы создаём репозиторий каждый раз, но root не имеет scope, поэтому DI отдаёт тот же singleton-like экземпляр — `CorrelationId` снова не меняется. Это «скрытый захват через фабрику». Правильное решение — `ScopedNotifierEngine`, который принимает `IServiceScopeFactory`. Эта фабрика singleton-safe (как подчёркнуто в уроке): её можно безопасно держать в долгоживущем объекте. На каждый цикл вызывается `CreateAsyncScope()` (новый в .NET 8 метод для `IAsyncDisposable` scope), из нового scope резолвится репозиторий — а значит, каждый цикл получает свежий `CorrelationId`. Канал отправки — настоящий singleton, поэтому его безопасно держать в поле engine. Ключевая декомпозиция: тело цикла вынесено в `RunOneCycleAsync`, который принимает только `CancellationToken` — это делает логику тестируемой без таймера и без `BackgroundService`. `CorrectNotifier` — тонкая hosted-обёртка, просто вызывающая engine в цикле с задержкой. `HeavyInitService` + `InitHostedService` закрывают третью тему урока — async resolution: конструктор лёгкий, тяжёлая инициализация в `StartAsync` с `await`, никакого `.Result`. Если бы мы попытались инициализировать в конструкторе через `_heavy.InitializeAsync().GetAwaiter().GetResult()`, мы бы получили тупик в синхронном контексте и race condition — ровно то, против чего предостерегает урок. Применённые концепции урока: правило «самое короткое время жизни, которое работает» (repo scoped, channel singleton, engine singleton, scope per use), `IServiceScopeFactory` как единственный правильный мост между долгоживущим и scoped, валидация `ValidateScopes`/`ValidateOnBuild`, лёгкие конструкторы и тяжёлая инициализация в hosted service, наблюдение captive dependency по поведению (через `CorrelationId`), а не по структуре регистраций.

#### Задания на углубление (бонус)

1. **Keyed services (.NET 8).** Добавьте два канала: `ConsoleChannel` (keyed `"console"`) и `CountingChannel` (keyed `"count"`). Настройте `ScopedNotifierEngine` так, чтобы канал выбирался через `[FromKeyedServices("console")]` или ключ из конфигурации. Покройте тестами переключение ключа. Сравните с классическим множественным регистрациям через `IEnumerable<INotificationChannel>`.
2. **Детектор captive dependency по сборке.** Напишите xUnit-теорию, которая перебирает несколько «плохих» комбинаций регистраций (`Singleton→Scoped`, `Singleton→Scoped` через `IServiceProvider`, `Singleton→Transient` с состоянием) и проверяет, что каждая из них падает при `ValidateScopes=true`. Документируйте, какие варианты НЕ падают и почему (например, `Transient` без состояния).
3. **Свой `IServiceScopeFactory` для тестов.** Реализуйте `TestScopeFactory`, который возвращает scope с предзаполненным `ServiceProvider` (через `ServiceCollection` + моки). Покажите, как это упрощает детерминированное тестирование engine без реального `Host`. Обсудите, почему прямое использование `BuildServiceProvider()` в тестах без `ValidateScopes` скрывает баги.
4. **Benchmarkscope.** С помощью `BenchmarkDotNet` измерьте накладные расходы `CreateScope()` + resolve vs прямого resolve из root. Объясните, почему экономия на scope «не стоит» бага captive dependency, и приведите цифры.

---

## Statement in English / Постановка на английском

#### Context & motivation

You maintain an internal notification service called `Notifier` for a course platform. The service runs as an `IHostedService`: every few seconds it reads a “queue” of pending notifications from a repository (a database stand-in) and sends them through a channel. The repository has state — it holds a `Guid CorrelationId` generated on every new “database connection”, so that all requests of a single session can be correlated in the logs. In production you notice a strange defect: the `CorrelationId` stays the same for hours, although by design it should change on every new connection. Logs from different runs get mixed up, tests fail non-deterministically, and metrics show parasitic memory growth.

When you investigate, you realise the root cause is not the business logic — it is DI. The background service is registered as a hosted service (effectively singleton by lifetime: one instance for the whole application), while the repository is `Scoped`. The service captured the repository in its constructor, and the `CorrelationId` got “frozen” inside that single instance forever. This is exactly the captive dependency from the lesson: a longer-lived service holding a shorter-lived one. The container silently accepted the registration because production runs with `ValidateScopes=false`. In a dev environment with validation the same registration would have failed at startup.

Your task is to reproduce all three flavours of the captive dependency described in the lesson (direct capture via constructor, hidden capture through `provider.GetRequiredService` from the root, and capture through a stored `IServiceProvider`), write deterministic tests that catch the bug by behaviour rather than by structure, and then rewrite the solution correctly — using `IServiceScopeFactory`, with container validation enabled and constructors kept cheap. Along the way you will also exercise the lesson’s second theme — resolving from the root in `Program.cs` — and the third theme — async initialisation in a constructor — by implementing a correct `IHostedService` with heavy startup.

#### What to do step by step

1. Create a solution and three projects. Run:
   ```
   dotnet new sln -n NotifierLab
   dotnet new console -n NotifierLab.App -o src/NotifierLab.App --framework net8.0
   dotnet new xunit -n NotifierLab.Tests -o tests/NotifierLab.Tests --framework net8.0
   dotnet new classlib -n NotifierLab.Core -o src/NotifierLab.Core --framework net8.0
   dotnet sln add src/NotifierLab.Core src/NotifierLab.App tests/NotifierLab.Tests
   dotnet add src/NotifierLab.App reference src/NotifierLab.Core
   dotnet add tests/NotifierLab.Tests reference src/NotifierLab.Core
   dotnet add src/NotifierLab.App package Microsoft.Extensions.Hosting
   dotnet add tests/NotifierLab.Tests package Microsoft.Extensions.Hosting
   ```
   Enable `<Nullable>enable</Nullable>` and `<TreatWarningsAsErrors>true</TreatWarningsAsErrors>` in every `.csproj`.

2. In `NotifierLab.Core` model the state. Create a class `PendingNotification` (a record with `Guid Id`, `string Payload`). Create `INotificationRepository` with a method `Task<IReadOnlyList<PendingNotification>> FetchBatchAsync(CancellationToken ct)`. Implement `InMemoryNotificationRepository`: it keeps a list of `PendingNotification`, generates `Guid CorrelationId = Guid.NewGuid()` in the constructor (this stands for a database connection), and `FetchBatchAsync` returns up to 5 items, removing them from the internal list, logging (via `Console.WriteLine`) the current `CorrelationId`. The log must show which `CorrelationId` served the request.

3. Create `INotificationChannel` with `Task SendAsync(PendingNotification n, CancellationToken ct)` and a trivial implementation `ConsoleChannel` that writes `[sent] {Id} {Payload}` to the console.

4. **Reproduce anti-pattern #1 — direct capture.** Create `CaptiveNotifier : BackgroundService` that takes `INotificationRepository` in the constructor and stores it in a field. `ExecuteAsync`: a loop with `Task.Delay(1_000, ct)`, in each iteration calling `_repo.FetchBatchAsync(ct)` and `SendAsync` per item. In `Program.cs` register `services.AddScoped<INotificationRepository, InMemoryNotificationRepository>()`, `services.AddSingleton<INotificationChannel, ConsoleChannel>()`, `services.AddHostedService<CaptiveNotifier>()`. Run `dotnet run --project src/NotifierLab.App`. Confirm from the logs that `CorrelationId` is THE SAME across all iterations — that is the captive dependency.

5. **Reproduce anti-pattern #2 — hidden capture via factory.** Create `FactoryCaptiveNotifier : BackgroundService` that accepts `IServiceProvider` and in `ExecuteAsync` calls `_provider.GetRequiredService<INotificationRepository>()` without creating a scope. Register it instead of the previous one. Run and confirm `CorrelationId` is again stuck — because `_provider` is the root, and the root has no scope.

6. **Reproduce anti-pattern #3 — capture through a stored `IServiceProvider`.** Create `ProviderCaptiveNotifier : BackgroundService` that accepts `IServiceProvider`, stores it, and resolves the repository lazily once into a field via `Lazy<INotificationRepository>`. The behaviour must be the same: one `CorrelationId` forever.

7. **Write tests that catch the bug by behaviour.** In `tests/NotifierLab.Tests` create `CaptiveDependencyTests`. Do not test registration structure directly — test observable behaviour: after two “iterations” of the cycle, the `CorrelationId` recorded by the repository must be DIFFERENT, provided the repository is scoped. Use a stand-in: create a `RecordingRepository` that appends every used `CorrelationId` to a list and assert that after two iterations the list has two distinct values. To “drive” the cycle deterministically, extract the worker function `DoOneCycleAsync(IServiceScope scope, CancellationToken ct)` from the `BackgroundService` into a separate testable class `NotifierEngine` — this is the correct decomposition (hosted service stays thin, logic lives in the engine).

8. **Fix the solution.** Create `ScopedNotifierEngine` that accepts `IServiceScopeFactory` (singleton-safe) and a method `RunOneCycleAsync(CancellationToken ct)` that creates `using var scope = _scopeFactory.CreateScope()`, resolves the repository and the channel from `scope.ServiceProvider`, and runs one cycle. Create `CorrectNotifier : BackgroundService` that simply calls `RunOneCycleAsync` in a loop with a delay. Register `services.AddScoped<INotificationRepository, ...>()`, `services.AddSingleton<INotificationChannel, ...>()`, `services.AddSingleton<NotifierEngine>()` (or a hosted service injecting `IServiceScopeFactory`), `services.AddHostedService<CorrectNotifier>()`.

9. **Enable container validation.** In `Program.cs` use `Host.CreateApplicationBuilder(args)` (it enables `ValidateScopes` and `ValidateOnBuild` in dev). Additionally show explicitly: `var host = builder.Build();` and verify that attempting to register anti-pattern #1 and starting in dev makes the container throw `InvalidOperationException` about a captive dependency. Capture this in a test: build a `ServiceCollection`, add the anti-pattern, call `BuildServiceProvider(new ServiceProviderOptions { ValidateScopes = true, ValidateOnBuild = true })` and `Assert.Throws<InvalidOperationException>(...)`.

10. **Reproduce the “resolve from root” theme.** In `Program.cs` add a commented-out dangerous fragment: `// var repo = app.Services.GetRequiredService<INotificationRepository>();` — explain in an English comment why this turns a scoped service into an effective singleton. Show the correct alternative: `using var scope = app.Services.CreateScope(); var repo = scope.ServiceProvider.GetRequiredService<INotificationRepository>();`.

11. **Reproduce the “async resolution” theme.** Create `HeavyInitService` whose constructor does NO I/O, plus a method `InitializeAsync` (heavy asynchronous initialisation). Create `InitHostedService : IHostedService` that calls `await _heavy.InitializeAsync(ct)` in `StartAsync`. Show in a comment why calling `InitializeAsync().GetAwaiter().GetResult()` in a constructor is an anti-pattern (deadlock in a synchronisation context, race conditions).

12. Run `dotnet test` — all tests green. Run `dotnet run --project src/NotifierLab.App` — in the logs `CorrelationId` changes every second (a new scope → a new repository → a new `CorrelationId`). Capture the output in a `README.md` at the solution root (2–3 paragraphs of what you observed).

#### Requirements

- Target framework `net8.0`, language C# 12. Use top-level statements in `Program.cs`, collection expressions (`[]`, `..`), pattern matching, `record` for DTOs, `required` members or `init` setters where appropriate.
- The project must compile without warnings: `<TreatWarningsAsErrors>true</TreatWarningsAsErrors>`, `<Nullable>enable</Nullable>`.
- Forbidden: storing a `Scoped` dependency in a field of a `Singleton`/hosted service; resolving scoped from root without a scope; calling `.Result`/`.Wait()`/`async void` in a constructor; registering every service as `Singleton` “for speed”.
- Mandatory: the correct solution uses `IServiceScopeFactory` and creates a scope per cycle/use; validation `ValidateScopes` and `ValidateOnBuild` is enabled in dev and test environments; constructors are cheap and heavy initialisation lives in `IHostedService.StartAsync`.
- Structure: `NotifierLab.Core` (service interfaces and implementations), `NotifierLab.App` (`Program.cs`, `BackgroundService` classes), `NotifierLab.Tests` (xUnit).
- All three flavours of captive dependency must be present in the code (each in its own `Captive*` file is fine) and covered by tests that either demonstrate the bug or fail to build the container with `ValidateScopes=true`.
- A README at the solution root must briefly explain the observed effect and include a sample log output.

#### Pitfalls

- `IServiceScopeFactory` is itself singleton-safe: you can inject it into singletons and hosted services without trouble. But the `IServiceProvider` you get as the root is NOT singleton-safe for scoped resolution: scoped services “stick” to it.
- `Host.CreateApplicationBuilder` automatically enables `ValidateScopes` in dev (`Environment == "Development"`). In production validation is off by default — so a captive dependency can slip through and only surface at runtime. In tests always build the provider with `new ServiceProviderOptions { ValidateScopes = true, ValidateOnBuild = true }`.
- `BackgroundService` is an `IHostedService`, and its instance is created exactly once for the whole application. Any dependency injected into its constructor is captured for the lifetime of the app. Therefore the only correct way to obtain a scoped service from a `BackgroundService` is `IServiceScopeFactory.CreateScope()` inside `ExecuteAsync`.
- If you resolve a scoped service from the root in `Program.cs` (for example to “warm up” a cache), you effectively make it a singleton: the instance lives as long as the root. The right way is `using var scope = app.Services.CreateScope();` and work inside that scope.
- `async void` in a constructor and `.Result`/`.Wait()` are deadlocks waiting to happen in a synchronisation context (especially in legacy ASP.NET, but also in tests with a `SynchronizationContext`). Move heavy initialisation into `IHostedService.StartAsync` and `await` it there.
- `ValidateOnBuild=true` catches not only captive dependencies but also any unresolvable registration (missing registration, circular dependencies in some cases). Enable both.
- Do not confuse “lifetime of the service” with “lifetime of the dependency”: a `Scoped` service that depends on a `Singleton` is fine (shorter holds longer). The trap is only the reverse: longer holds shorter.
- `AddHostedService<T>()` registers `T` with a lifetime that makes the instance available through `IHostedService`, but the instance itself is created once. Do not put per-request state in fields of a hosted service.
- In tests, do not rely on `Guid.NewGuid()` as “guaranteed unique” — for determinism inject a generator (`Func<Guid>`) or mock the repository entirely, asserting on `CorrelationId` by behaviour.

#### Acceptance criteria

- [ ] The `NotifierLab.sln` solution builds with `dotnet build` with no errors and no warnings (warnings as errors).
- [ ] Target framework `net8.0`, C# 12, top-level statements in `Program.cs`.
- [ ] All three flavours of captive dependency are present: `CaptiveNotifier` (direct capture), `FactoryCaptiveNotifier` (via `GetRequiredService` from root), `ProviderCaptiveNotifier` (via a stored `IServiceProvider`).
- [ ] When anti-patterns #1–#3 run, the logs show the same `CorrelationId` across all iterations (captured in README).
- [ ] The correct solution `CorrectNotifier` + `ScopedNotifierEngine` uses `IServiceScopeFactory.CreateScope()` per cycle; `CorrelationId` changes every iteration.
- [ ] `Program.cs` explicitly enables validation via `Host.CreateApplicationBuilder` (dev) and demonstrates explicit `ServiceProviderOptions { ValidateScopes = true, ValidateOnBuild = true }` in tests.
- [ ] A test `Assert.Throws<InvalidOperationException>` fails when building the container with anti-pattern #1 under `ValidateScopes=true`.
- [ ] A behaviour test confirms that across two iterations of the correct engine, the repository recorded two distinct `CorrelationId` values.
- [ ] The “resolve from root” theme is reflected: the dangerous fragment is commented out with an explanation, the correct alternative via `CreateScope()` is shown.
- [ ] The “async resolution” theme is reflected: `InitHostedService` calls `await _heavy.InitializeAsync(ct)` in `StartAsync`; a comment explains the `.Result`-in-constructor anti-pattern.
- [ ] No constructor performs blocking I/O or calls `.Result`/`.Wait()`/`async void`.
- [ ] No `Singleton`/`BackgroundService` accepts a `Scoped` dependency in its constructor.
- [ ] All `dotnet test` tests are green.
- [ ] A README at the solution root contains 2–3 paragraphs of observations and a sample log output.
- [ ] The code uses collection expressions, pattern matching, and `record` for DTOs (in at least two places).
- [ ] The code has no `TODO`/`FIXME` stubs and no commented-out junk, except the deliberately shown anti-patterns with explanations.

#### Hints (no direct answer)

- Think about why `IServiceScopeFactory` is “safe” for a singleton while `IServiceProvider` (root) is not. The difference is who creates scoped instances and in which scope they live.
- To test a `BackgroundService` cycle deterministically, extract the body of a single iteration into a separate method that takes an `IServiceScope` (or `CancellationToken`) and does not depend on a timer. Then in the test you can call the method twice and assert on behaviour.
- To check a captive dependency structurally, do not parse registrations — try to build the provider with `ValidateScopes=true` and let the container tell you what is wrong.
- `CorrelationId` is a convenient “marker” of a scoped service’s state. If it is identical across iterations, you have captured a scoped service in a long-lived object.
- Remember that `AddHostedService<T>` does not make `T` scoped — the instance is one per application.
- For heavy initialisation ask yourself: “must this code run before the first HTTP request?” If yes — `IHostedService.StartAsync`, not a constructor.

#### Reference solution walk-through

```csharp
// C# 12 / .NET 8 — Reference solution for homework M16-L02
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;

namespace NotifierLab.Core;

// --- DTO: a single pending notification ---
public sealed record PendingNotification(Guid Id, string Payload);

// --- Repository: scoped, holds state (CorrelationId) ---
public interface INotificationRepository
{
    Guid CorrelationId { get; } // "connection" marker
    Task<IReadOnlyList<PendingNotification>> FetchBatchAsync(CancellationToken ct);
}

public sealed class InMemoryNotificationRepository : INotificationRepository
{
    private readonly List<PendingNotification> _queue;
    public Guid CorrelationId { get; } = Guid.NewGuid();

    public InMemoryNotificationRepository()
    {
        _queue = [
            new(Guid.NewGuid(), "hello-1"),
            new(Guid.NewGuid(), "hello-2"),
            new(Guid.NewGuid(), "hello-3"),
            new(Guid.NewGuid(), "hello-4"),
            new(Guid.NewGuid(), "hello-5"),
            new(Guid.NewGuid(), "hello-6"),
            new(Guid.NewGuid(), "hello-7"),
        ];
    }

    public Task<IReadOnlyList<PendingNotification>> FetchBatchAsync(CancellationToken ct)
    {
        ct.ThrowIfCancellationRequested();
        var batch = _queue.Count > 0
            ? _queue[..Math.Min(5, _queue.Count)] // collection slicing
            : [];
        _queue.RemoveRange(0, batch.Count);
        Console.WriteLine($"[repo] CorrelationId={CorrelationId}, fetched={batch.Count}");
        return Task.FromResult<IReadOnlyList<PendingNotification>>(batch);
    }
}

// --- Send channel ---
public interface INotificationChannel
{
    Task SendAsync(PendingNotification n, CancellationToken ct);
}

public sealed class ConsoleChannel : INotificationChannel
{
    public Task SendAsync(PendingNotification n, CancellationToken ct)
    {
        Console.WriteLine($"[sent] {n.Id} {n.Payload}");
        return Task.CompletedTask;
    }
}

// --- Anti-pattern #1: direct capture of scoped in a singleton (captive dependency) ---
public sealed class CaptiveNotifier : BackgroundService
{
    private readonly INotificationRepository _repo; // captured!
    public CaptiveNotifier(INotificationRepository repo) => _repo = repo;

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            var batch = await _repo.FetchBatchAsync(stoppingToken);
            foreach (var n in batch) Console.WriteLine($"[captive-sent] {n.Id}");
            await Task.Delay(TimeSpan.FromSeconds(1), stoppingToken);
        }
    }
}

// --- Anti-pattern #2: hidden capture via the root provider ---
public sealed class FactoryCaptiveNotifier : BackgroundService
{
    private readonly IServiceProvider _provider;
    public FactoryCaptiveNotifier(IServiceProvider provider) => _provider = provider;

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            // resolving from root, no scope → scoped sticks
            var repo = _provider.GetRequiredService<INotificationRepository>();
            var batch = await repo.FetchBatchAsync(stoppingToken);
            await Task.Delay(TimeSpan.FromSeconds(1), stoppingToken);
        }
    }
}

// --- Correct engine: scoped via IServiceScopeFactory ---
public sealed class ScopedNotifierEngine
{
    private readonly IServiceScopeFactory _scopeFactory; // singleton-safe
    private readonly INotificationChannel _channel;       // singleton — safe to hold

    public ScopedNotifierEngine(IServiceScopeFactory scopeFactory, INotificationChannel channel)
    {
        _scopeFactory = scopeFactory;
        _channel = channel;
    }

    // One cycle — one scope. The body is testable without a timer.
    public async Task RunOneCycleAsync(CancellationToken ct)
    {
        await using var scope = _scopeFactory.CreateAsyncScope(); // .NET 8: async scope
        var repo = scope.ServiceProvider.GetRequiredService<INotificationRepository>();
        var batch = await repo.FetchBatchAsync(ct);
        foreach (var n in batch)
            await _channel.SendAsync(n, ct);
    }
}

// --- Correct hosted service: thin wrapper over the engine ---
public sealed class CorrectNotifier : BackgroundService
{
    private readonly ScopedNotifierEngine _engine;
    public CorrectNotifier(ScopedNotifierEngine engine) => _engine = engine;

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            await _engine.RunOneCycleAsync(stoppingToken);
            await Task.Delay(TimeSpan.FromSeconds(1), stoppingToken);
        }
    }
}

// --- "Async resolution" theme: heavy init in StartAsync ---
public sealed class HeavyInitService
{
    public int State { get; private set; }
    // cheap constructor — no I/O
    public Task InitializeAsync(CancellationToken ct)
    {
        State = 42; // imitates heavy work
        return Task.CompletedTask;
    }
}

public sealed class InitHostedService : IHostedService
{
    private readonly HeavyInitService _heavy;
    public InitHostedService(HeavyInitService heavy) => _heavy = heavy;

    public async Task StartAsync(CancellationToken cancellationToken)
    {
        // correct: await in StartAsync
        await _heavy.InitializeAsync(cancellationToken);
    }
    public Task StopAsync(CancellationToken cancellationToken) => Task.CompletedTask;
}
```

Walk-through. `PendingNotification` is a `record`, which gives value semantics and structural equality for DTOs, exactly as the lesson’s best practices recommend. `InMemoryNotificationRepository` is a scoped candidate: it has state — the `_queue` list and a `CorrelationId` generated in the constructor. That is the “database connection”: every new instance is a new connection with a new `CorrelationId`. `FetchBatchAsync` uses collection slicing `_queue[..n]` (C# 12) and the collection expression `[]` for an empty result — modern language idioms. `CaptiveNotifier` is the first anti-pattern: it takes `INotificationRepository` in the constructor. Because a `BackgroundService` is created once, the repository is also created once and “frozen”: every iteration sees the same `CorrelationId`. This is exactly the “direct capture” from the lesson, and a container with `ValidateScopes=true` will throw at `Build`. `FactoryCaptiveNotifier` is the second anti-pattern: it stores the `IServiceProvider` (root) and resolves the repository “on the spot”. It looks as if a new repository is built every time, but the root has no scope, so DI hands back the same singleton-like instance — the `CorrelationId` does not change. That is the “hidden capture via factory”. The correct solution is `ScopedNotifierEngine`, which accepts `IServiceScopeFactory`. This factory is singleton-safe (as the lesson stresses): it is safe to hold in a long-lived object. On every cycle `CreateAsyncScope()` is called (a new .NET 8 method for an `IAsyncDisposable` scope), and the repository is resolved from the fresh scope — so every cycle gets a fresh `CorrelationId`. The send channel is a genuine singleton, so it is safe to keep it in a field of the engine. The key decomposition: the cycle body is moved into `RunOneCycleAsync`, which takes only a `CancellationToken` — this makes the logic testable without a timer and without `BackgroundService`. `CorrectNotifier` is a thin hosted wrapper that simply calls the engine in a loop with a delay. `HeavyInitService` plus `InitHostedService` cover the third lesson theme — async resolution: the constructor is cheap, the heavy initialisation lives in `StartAsync` with `await`, never `.Result`. If we tried to initialise in the constructor via `_heavy.InitializeAsync().GetAwaiter().GetResult()`, we would get a deadlock in a synchronisation context and a race condition — exactly what the lesson warns against. Lesson concepts applied: the rule “shortest lifetime that actually works” (repo scoped, channel singleton, engine singleton, scope per use), `IServiceScopeFactory` as the only correct bridge between long-lived and scoped, `ValidateScopes`/`ValidateOnBuild` validation, cheap constructors with heavy init in a hosted service, and observing the captive dependency by behaviour (via `CorrelationId`) rather than by registration structure.

#### Going deeper (bonus)

1. **Keyed services (.NET 8).** Add two channels: `ConsoleChannel` (keyed `"console"`) and `CountingChannel` (keyed `"count"`). Configure `ScopedNotifierEngine` so the channel is selected through `[FromKeyedServices("console")]` or a key from configuration. Cover the key switch with tests. Compare with the classic multi-registration approach through `IEnumerable<INotificationChannel>`.
2. **A build-time captive-dependency detector.** Write an xUnit theory that iterates several “bad” registration combinations (`Singleton→Scoped`, `Singleton→Scoped` via `IServiceProvider`, `Singleton→Transient` with state) and asserts each one throws under `ValidateScopes=true`. Document which variants do NOT throw and why (for example `Transient` without state).
3. **Your own `IServiceScopeFactory` for tests.** Implement a `TestScopeFactory` that returns a scope with a pre-filled `ServiceProvider` (via `ServiceCollection` and mocks). Show how this simplifies deterministic engine testing without a real `Host`. Discuss why using `BuildServiceProvider()` in tests without `ValidateScopes` hides bugs.
4. **Scope benchmark.** With `BenchmarkDotNet` measure the overhead of `CreateScope()` plus resolve versus resolving directly from the root. Explain why saving on a scope is “not worth” the captive-dependency bug, and include the numbers.

---

#### Чек-лист сдачи / Submission checklist
- [ ] Решение `NotifierLab.sln` собрано, `dotnet build` без предупреждений (RU).
- [ ] Все три варианта captive dependency присутствуют и покрыты тестами (RU).
- [ ] Правильное решение использует `IServiceScopeFactory` и `CreateScope()` на цикл (RU).
- [ ] В dev/test включены `ValidateScopes` и `ValidateOnBuild` (RU).
- [ ] Тест `Assert.Throws<InvalidOperationException>` на антипаттерне №1 (RU).
- [ ] README содержит наблюдения и пример лога (RU).
- [ ] The `NotifierLab.sln` solution builds, `dotnet build` with no warnings (EN).
- [ ] All three captive-dependency flavours are present and covered by tests (EN).
- [ ] The correct solution uses `IServiceScopeFactory` and `CreateScope()` per cycle (EN).
- [ ] `ValidateScopes` and `ValidateOnBuild` are enabled in dev/test (EN).
- [ ] An `Assert.Throws<InvalidOperationException>` test for anti-pattern #1 (EN).
- [ ] README contains observations and a log sample (EN).

#### Ресурсы / Resources
- [Microsoft Learn — Dependency injection guidelines](https://learn.microsoft.com/dotnet/core/extensions/dependency-injection-guidelines)
- [Microsoft Learn — DI in .NET](https://learn.microsoft.com/dotnet/core/extensions/dependency-injection)
- [Microsoft Learn — Keyed services](https://learn.microsoft.com/dotnet/core/extensions/dependency-injection#keyed-services)
- [Microsoft Learn — Hosted service with scoped service](https://learn.microsoft.com/dotnet/core/extensions/work-with-scope)
- [Mark Seemann — Captive Dependency](https://blog.ploeh.dk/2014/06/02/captive-dependency/)
