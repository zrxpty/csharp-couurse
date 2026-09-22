[⬆ К модулю M16](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M16-L03: Scrutor: assembly scan, decorators, adapters / Scrutor: assembly scan, decorators, adapters

**Модуль / Module:** M16
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

**Scrutor** — это небольшая библиотека-расширение над стандартным DI-контейнером Microsoft.Extensions.DependencyInjection, которая закрывает две самые частые больные точки ручной регистрации:扫描 сборок (assembly scanning) и декорирование сервисов (decoration). Название — это игра слов: «scrutor» по-латыни означает «я исследую/обыскиваю», что намекает на способность библиотеки «обыскивать» сборки в поисках типов для регистрации.

**Зачем нужен assembly scanning?** В классическом DI вы регистрируете каждый сервис вручную: `services.AddSingleton<IOrderService, OrderService>();`. Когда сервисов десять — это терпимо. Когда их сто — `Program.cs` превращается в простыню на 300 строк, которую страшно трогать. Scrutor позволяет сказать: «зарегистрируй все классы, реализующие интерфейс `IService` в сборке `MyApp.Services`». Это аналог того, как EF Core «находит» сущности через `DbSet` или как контроллеры автоматически подхватываются ASP.NET. Вы описываете **конвенцию** (правило), а не каждый класс по отдельности.

**Регистрация по конвенции.** Метод `Scan` принимает делегат с тремя ключевыми блоками: `FromAssemblyOf<T>()` (откуда искать типы), `AddClasses(...)` (какие классы брать) и `UsingRegistrationStrategy(...)` / `As<...>` / `WithLifetime(...)` (как регистрировать). Например, можно сказать: «возьми все публичные классы из сборки Services, которые реализуют интерфейс с тем же именем минус префикс `I`, зарегистрируй их как соответствующий интерфейс со временем жизни Scoped». Это позволяет наложить единую политику на весь слой.

**Декораторы через DI.** Декоратор — это класс, который оборачивает другой сервис того же интерфейса, добавляя поведение (логирование, кэш, валидацию, retry) без изменения исходного класса. Руками декоратор собрать муторно: нужно разрешать «внутренний» экземпляр, следить за временем жизни, не зациклиться. Scrutor даёт метод `Decorate<TInterface, TDecorator>()` и `DecorateDecorator(...)`. Контейнер сам понимает, что `TDecorator` нужно «вложить» в предыдущую регистрацию `TInterface`. Декораторы можно цепочкой: логирование → кэш → реальный сервис.

**Адаптеры.** Scrutor умеет регистрировать адаптеры через `RegisterAdapter<TImplementation, TAdapter>()`: когда у вас есть сторонний тип (например, `ExternalBillingClient`), а ваш код ждёт интерфейс `IBillingGateway`, адаптер «переводит» один контракт в другой. Это удобно для изоляции внешних библиотек и предотвращает их прямое просачивание в доменную модель.

**Аналогия.** Сканирование — как таможня в аэропорту: вы задаёте правило («все пассажиры рейса SU1234 на регистрацию»), а не выписываете каждого по имени. Декоратор — как матрёшка: внутри `OrderService`, снаружи слой кэша, ещё снаружи слой логирования, но снаружи всё это выглядит как `IOrderService`.

**Когда использовать, а когда нет.** Scrutor идеален для толстых слоёв приложений (Services, Handlers, Validators) и для плагинных архитектур. Для двух-трёх инфраструктурных сервисов проще обойтись ручной регистрацией — сканирование тут лишь усложнит читаемость без выгоды.

#### Theory (EN)

**Scrutor** is a small extension library over the default Microsoft.Extensions.DependencyInjection container that fixes two of the most painful gaps in manual registration: assembly scanning and service decoration. The name is a Latin pun — *scrutor* means «I examine / I search through», which hints at the library's ability to «search through» assemblies for types to register.

**Why assembly scanning?** In classic DI you register every service by hand: `services.AddSingleton<IOrderService, OrderService>();`. With ten services that is tolerable; with a hundred, your `Program.cs` turns into a 300-line wall of text that nobody wants to touch. Scrutor lets you say: «register every class implementing `IService` in the `MyApp.Services` assembly». It is the same idea behind EF Core discovering entities through `DbSet`, or ASP.NET auto-discovering controllers. You describe a **convention** (a rule) instead of naming each class individually.

**Registration by convention.** The `Scan` method takes a delegate with three building blocks: `FromAssemblyOf<T>()` (where to look), `AddClasses(...)` (which classes to pick), and `UsingRegistrationStrategy(...)` / `As<...>` / `WithLifetime(...)` (how to register them). For example: «take all public classes from the Services assembly that implement an interface with the same name minus the `I` prefix, register them as that interface with Scoped lifetime». This applies one uniform policy to a whole layer.

**Decorators via DI.** A decorator is a class wrapping another service of the same interface, adding behavior (logging, caching, validation, retry) without modifying the original class. Building a decorator by hand is awkward: you have to resolve the «inner» instance, respect lifetimes, and avoid an infinite resolution loop. Scrutor gives you `Decorate<TInterface, TDecorator>()` and `DecorateDecorator(...)`. The container understands that `TDecorator` must wrap the previous registration of `TInterface`. Decorators chain cleanly: logging → cache → real service.

**Adapters.** Scrutor registers adapters through `RegisterAdapter<TImplementation, TAdapter>()`: when you have a third-party type (say `ExternalBillingClient`) but your code expects an `IBillingGateway`, the adapter «translates» one contract into the other. This isolates external libraries and keeps them from leaking into your domain model.

**Analogy.** Scanning is like airport passport control: you set a rule («all passengers of flight SU1234 to the gate») instead of naming each person. A decorator is a Russian matryoshka doll: `OrderService` inside, a cache layer around it, a logging layer outside, yet from the outside the whole stack still looks like `IOrderService`.

**When to use it — and when not to.** Scrutor shines for thick layers (Services, Handlers, Validators) and plugin-style architectures. For two or three infrastructure services, plain manual registration is clearer and scanning only adds indirection without payoff.

#### Пример кода / Code Example

```csharp
// C# 12+ / .NET 8+ — рабочий пример Scrutor: scan + decorator + adapter
// Install-Package Scrutor

using Microsoft.Extensions.DependencyInjection;
using System.Reflection;

// --- Доменный интерфейс и реализация / Domain interface and implementation
public interface IOrderService
{
    Task<Guid> CreateAsync(string customer); // Создать заказ / Create an order
}

public class OrderService : IOrderService
{
    public Task<Guid> CreateAsync(string customer)
    {
        Console.WriteLine($"[OrderService] Создан заказ для {customer} / Created order for {customer}");
        return Task.FromResult(Guid.NewGuid());
    }
}

// --- Декоратор: логирование / Decorator: logging
public class LoggingOrderDecorator : IOrderService
{
    private readonly IOrderService _inner;            // Внутренний сервис / Inner service
    private readonly ILogger _logger;

    public LoggingOrderDecorator(IOrderService inner, ILogger logger)
    {
        _inner = inner;
        _logger = logger;
    }

    public async Task<Guid> CreateAsync(string customer)
    {
        _logger.Log($"[Decorator] Перед вызовом / Before call: {customer}");
        var id = await _inner.CreateAsync(customer);  // Делегируем / Delegate
        _logger.Log($"[Decorator] После вызова / After call: {id}");
        return id;
    }
}

// --- Внешний тип и адаптер / External type and adapter
public sealed class ExternalBillingClient            // Сторонняя библиотека / Third-party lib
{
    public void Charge(decimal amount) => Console.WriteLine($"[External] {amount} списано / charged");
}

public interface IBillingGateway                     // Наш контракт / Our contract
{
    void Pay(decimal amount);
}

// Адаптер переводит IBillingGateway → ExternalBillingClient
// Adapter translates IBillingGateway → ExternalBillingClient
public class BillingAdapter : IBillingGateway
{
    private readonly ExternalBillingClient _client;
    public BillingAdapter(ExternalBillingClient client) => _client = client;
    public void Pay(decimal amount) => _client.Charge(amount);
}

public interface ILogger { void Log(string msg); }
public class ConsoleLogger : ILogger { public void Log(string msg) => Console.WriteLine(msg); }

// --- Регистрация через Scrutor / Scrutor registration
public static class ServiceConfigurator
{
    public static IServiceCollection AddAppServices(this IServiceCollection services)
    {
        // 1) Скан сборки: все реализации I*Service как их интерфейсы, Scoped
        // 1) Assembly scan: all I*Service implementations as their interfaces, Scoped
        services.Scan(scan => scan
            .FromAssemblyOf<OrderService>()
            .AddClasses(c => c.AssignableTo<IServiceMarker>())        // По маркеру / By marker
            .UsingRegistrationStrategy(RegistrationStrategy.Skip)     // Пропустить дубль / Skip duplicates
            .AsMatchingInterface()                                    // IFoo → Foo / IFoo → Foo
            .WithScopedLifetime());

        // 2) Явная регистрация базового сервиса / Explicit base registration
        services.AddScoped<IOrderService, OrderService>();

        // 3) Декорирование: логирование поверх OrderService
        // 3) Decoration: logging wraps OrderService
        services.Decorate<IOrderService, LoggingOrderDecorator>();

        // 4) Адаптер внешнего типа / Adapter for external type
        services.AddSingleton<ExternalBillingClient>();
        services.RegisterAdapter<ExternalBillingClient, IBillingGateway>(
            client => new BillingAdapter(client));

        services.AddSingleton<ILogger, ConsoleLogger>();
        return services;
    }
}

// Маркер для выборочного сканирования / Marker for selective scanning
public interface IServiceMarker { }

// --- Использование / Usage
public class App
{
    public static async Task Main()
    {
        var services = new ServiceCollection();
        services.AddAppServices();
        var sp = services.BuildServiceProvider();

        var orders = sp.GetRequiredService<IOrderService>();  // Это декорированный сервис
        await orders.CreateAsync("Alice");                    // Decorated service in action

        var billing = sp.GetRequiredService<IBillingGateway>();
        billing.Pay(9.99m);                                   // Через адаптер / Through adapter
    }
}
```

#### Best Practices
- Сканируйте сборки по явному маркерному интерфейсу (`IServiceMarker`) или атрибуту, а не «вообще все классы» — это делает правила предсказуемыми и устойчивыми к случайным добавлениям.
- Always scan by an explicit marker interface or attribute (`IServiceMarker`) rather than «every class in the assembly» — this keeps the rule predictable and resilient to accidental additions.
- Регистрируйте декораторы только через Scrutor; не создавайте «внутренний» ключ (`GetService<IOrderService>("inner")`) вручную — контейнер сам управляет цепочкой.
- Register decorators only through Scrutor; do not hand-roll an «inner» keyed lookup — the container manages the chain for you.
- Для адаптеров изолируйте сторонние типы в отдельной сборке `Adapters`, чтобы домен не зависел от внешних пакетов.
- Keep third-party types behind adapters in a dedicated `Adapters` assembly so the domain never depends on external packages.
- Явно указывайте `RegistrationStrategy.Skip` или `Append`, иначе повторная регистрация бросит исключение.
- Set `RegistrationStrategy.Skip` or `Append` explicitly, otherwise a duplicate registration throws.
- Декораторы, хранящие состояние (кэш, счётчик), должны совпадать по времени жизни с декорируемым сервисом.
- Stateful decorators (cache, counters) must share the lifetime of the decorated service.

#### Частые ошибки / Common Mistakes
- Сканирование «всех классов сборки» → раздувание контейнера мусорными типами (DTO, models). Решение: фильтруйте через маркерный интерфейс или `AssignableTo<T>()`.
- Scanning «all classes in the assembly» → container bloat with DTOs and models. Fix: filter with a marker interface or `AssignableTo<T>()`.
- Декоратор с временем жизни Singleton оборачивает Scoped-сервис → захватывает его надолго и нарушает DI-правила. Решение: выравнивайте lifetimes или используйте фабрику.
- Singleton decorator wrapping a Scoped service → captures it for too long, violating DI rules. Fix: align lifetimes or use a factory.
- `AsSelf()` вместо `AsMatchingInterface()` → тип зарегистрирован сам по себе, но не разрешается по интерфейсу. Решение: используйте `AsMatchingInterface()` или `As<IFoo>()`.
- `AsSelf()` instead of `AsMatchingInterface()` → type registered but not resolvable by interface. Fix: use `AsMatchingInterface()` or `As<IFoo>()`.
- Забыли `Decorate(...)` после `AddScoped<IInterface, Impl>()` → decorator не применён, сервис работает «голым». Решение: проверяйте регистрацию в unit-тесте.
- Forgot `Decorate(...)` after `AddScoped<IInterface, Impl>()` → decorator never applied, service runs bare. Fix: assert decoration in a unit test.
- Адаптер возвращает `null`-проверку стороннего клиента прямо в домене → утечка внешнего типа. Решение: создавайте адаптер в Composition Root, домен знает только интерфейс.
- Adapter exposes the third-party client to the domain → leak of external type. Fix: construct the adapter in the Composition Root; the domain sees only the interface.

#### Чек-лист самопроверки / Self-check Checklist
- [ ] Сборки для сканирования выбраны явно (`FromAssemblyOf<T>` или `FromAssemblies(...)`), а не «все подряд».
- [ ] Assembly scanning targets are explicit (`FromAssemblyOf<T>` or `FromAssemblies(...)`), not «everything».
- [ ] Используется маркерный интерфейс или атрибут для фильтрации типов.
- [ ] A marker interface or attribute is used to filter types.
- [ ] Стратегия дубликатов (`Skip` / `Append`) задана явно.
- [ ] Duplicate strategy (`Skip` / `Append`) is set explicitly.
- [ ] Время жизни декоратора совпадает с декорируемым сервисом.
- [ ] Decorator lifetime matches the decorated service's lifetime.
- [ ] Декораторы зарегистрированы через `Decorate<,>()`, а не вручную.
- [ ] Decorators registered via `Decorate<,>()`, not by hand.
- [ ] Внешние типы спрятаны за адаптерами в Composition Root.
- [ ] External types are hidden behind adapters in the Composition Root.
- [ ] Есть unit-тест, проверяющий, что декоратор действительно обёрнут.
- [ ] A unit test verifies the decorator is actually applied.

#### Ресурсы / Resources
- GitHub — Scrutor — https://github.com/khellang/Scrutor
- Microsoft Learn — Dependency injection in .NET — https://learn.microsoft.com/dotnet/core/extensions/dependency-injection
- Martin Fowler — Decorator pattern — https://martinfowler.com/articles/injection.html#DealingWithSimilarServicesOrANeedForDecoration

---

[⬆ К модулю M16](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
