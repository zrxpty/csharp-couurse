[⬆ К модулю M16](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M16-L07: Strategy, Decorator, Adapter / Strategy, Decorator, Adapter

**Модуль / Module:** M16
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

Шаблоны **Strategy**, **Decorator** и **Adapter** относятся к категории паттернов поведения и структуры. Они решают три разные, но одинаково важные проблемы: выбор алгоритма во время выполнения, добавление обязанностей без наследования и согласование несовместимых интерфейсов.

**Strategy (Стратегия)** определяет семейство алгоритмов, инкапсулирует каждый из них и делает их взаимозаменяемыми. Представьте навигатор в автомобиле: один и тот же маршрут можно построить пешком, на велосипеде, на машине или на общественном транспорте. Алгоритм маршрутизации меняется в зависимости от контекста, а клиентский код (экран навигатора) остаётся прежним. В .NET Strategy обычно реализуется как интерфейс с несколькими реализациями, которые регистрируются в DI-контейнере и выбираются по ключу, типу или фабрике. Это особенно полезно при расчётах цен с разными правилами скидок, при валидации документов по типам или при выборе способа оплаты.

**Decorator (Декоратор)** динамически добавляет объекту новые обязанности и является гибкой альтернативой наследованию. Аналогия — матрёшка или многослойная обёртка подарка: каждая обёртка добавляет своё свойство, не меняя внутреннее содержимое. В .NET Decorator идеально сочетается с Dependency Injection: один и тот же интерфейс регистрируется несколько раз с разными реализациями, каждая из которых оборачивает предыдущую. Классический пример — логирование и кэширование. Если у вас есть `IOrderService`, то `LoggingOrderService` оборачивает базовый `OrderService`, а `CachingOrderService` может оборачивать уже залогированный вариант. Контейнер сам собирает цепочку, и клиент получает единый интерфейс с обогащённым поведением. Главное преимущество — каждый декоратор решает одну задачу (SRP), а классы остаются открытыми для расширения, но закрытыми для изменения (OCP).

**Adapter (Адаптер)** преобразует интерфейс одного класса в другой, ожидаемый клиентом. Аналогия — переходник для розетки: вилка европейского стандарта не входит в британскую розетку, но переходник делает их совместимыми без изменения ни вилки, ни розетки. В .NET Adapter незаменим при интеграции с устаревшими системами, сторонними библиотеками или при оборачивании статических классов (например, `DateTime.Now` или `File.ReadAllText`) в тестируемый интерфейс. Адаптер реализует целевой интерфейс и делегирует вызовы адаптируемому объекту, при необходимости преобразуя типы данных и форматы.

Все три паттерна объединяет принцип работы через абстракцию: клиент зависит от интерфейса, а не от конкретной реализации. Это упрощает тестирование (можно подставить mock), соответствует SOLID и делает систему готовой к изменениям. Strategy выбирает «как делать», Decorator добавляет «что ещё делать», Adapter говорит «как разговаривать» с тем, кто говорит на другом языке. В современной разработке на C# их редко применяют изолированно — они комбинируются в слоях приложения: Adapter подключает внешний сервис, Decorator добавляет логирование и кэш, а Strategy выбирает бизнес-правило.

#### Theory (EN)

The **Strategy**, **Decorator**, and **Adapter** patterns belong to the behavioral and structural categories of design patterns. They solve three distinct but equally important problems: choosing an algorithm at runtime, adding responsibilities without inheritance, and making incompatible interfaces work together.

**Strategy** defines a family of algorithms, encapsulates each one, and makes them interchangeable. Think of a car navigation app: the same route can be computed for walking, cycling, driving, or public transit. The routing algorithm changes based on context, while the client code (the navigation screen) stays the same. In .NET, Strategy is usually implemented as an interface with multiple implementations registered in the DI container and selected by key, type, or a factory. This is especially useful for price calculations with different discount rules, document validation by type, or choosing a payment method.

**Decorator** dynamically adds responsibilities to an object and is a flexible alternative to inheritance. The analogy is a Russian nesting doll or a layered gift wrap: each layer adds a property without changing the inside. In .NET, Decorator pairs perfectly with Dependency Injection: the same interface is registered multiple times with different implementations, each wrapping the previous one. A classic example is logging and caching. If you have `IOrderService`, then `LoggingOrderService` wraps the base `OrderService`, and `CachingOrderService` can wrap the already-logged version. The container assembles the chain, and the client gets a single interface with enriched behavior. The key benefit is that each decorator solves one task (SRP), and classes stay open for extension but closed for modification (OCP).

**Adapter** converts the interface of one class into another interface that clients expect. The analogy is a power plug adapter: a European plug does not fit a British socket, but the adapter makes them compatible without changing either the plug or the socket. In .NET, Adapter is indispensable when integrating legacy systems, third-party libraries, or when wrapping static classes (such as `DateTime.Now` or `File.ReadAllText`) into a testable interface. An adapter implements the target interface and delegates calls to the adaptee, transforming data types and formats as needed.

All three patterns share a common principle: the client depends on an abstraction, not a concrete implementation. This simplifies testing (you can inject a mock), follows SOLID, and makes the system ready for change. Strategy chooses “how to do it”, Decorator adds “what else to do”, and Adapter teaches “how to talk” to someone speaking another language. In modern C# development these patterns are rarely used in isolation — they combine across application layers: an Adapter connects an external service, a Decorator adds logging and cache, and a Strategy selects the business rule.

#### Пример кода / Code Example

```csharp
// C# 12+ / .NET 8+ — Strategy, Decorator, Adapter в одном демо
// Полностью рабочий код: скопируйте в Program.cs консольного проекта

using Microsoft.Extensions.DependencyInjection;

// ============================================================
// STRATEGY — выбор алгоритма расчёта скидки / discount strategy
// ============================================================

// Общий интерфейс стратегии / common strategy interface
public interface IDiscountStrategy
{
    decimal Apply(decimal price);              // Применить скидку / apply discount
}

// Конкретные стратегии / concrete strategies
public sealed class NoDiscountStrategy : IDiscountStrategy
{
    public decimal Apply(decimal price) => price;
}

public sealed class PercentageDiscountStrategy(decimal percent) : IDiscountStrategy
{
    public decimal Apply(decimal price) => price * (1m - percent / 100m);
}

public sealed class FixedDiscountStrategy(decimal amount) : IDiscountStrategy
{
    public decimal Apply(decimal price) => Math.Max(0m, price - amount);
}

// Контекст использует стратегию через DI / context uses strategy via DI
public sealed class PriceCalculator(IDiscountStrategy strategy)
{
    public decimal Calculate(decimal basePrice) => strategy.Apply(basePrice);
}

// ============================================================
// ADAPTER — оборачиваем сторонний класс в наш интерфейс
// Adapter: wrap a third-party class into our interface
// ============================================================

// Наш интерфейс для оплаты / our payment interface
public interface IPaymentGateway
{
    Task<bool> ChargeAsync(string userId, decimal amount, CancellationToken ct);
}

// Сторонняя библиотека (не меняем) / third-party library (we cannot change it)
public sealed class LegacyPayPalSdk
{
    public int MakePayment(string customerRef, double usdAmount)
        => usdAmount > 0 ? 200 : 400;          // 200 = OK, 400 = FAIL
}

// Адаптер приводит LegacyPayPalSdk к IPaymentGateway
// Adapter converts LegacyPayPalSdk to IPaymentGateway
public sealed class PayPalAdapter(LegacyPayPalSdk sdk) : IPaymentGateway
{
    public Task<bool> ChargeAsync(string userId, decimal amount, CancellationToken ct)
    {
        // Преобразуем decimal → double, строку оставляем как есть
        // Convert decimal to double, keep the string as-is
        int status = sdk.MakePayment(userId, (double)amount);
        return Task.FromResult(status == 200);
    }
}

// ============================================================
// DECORATOR — добавляем логирование и кэш поверх сервиса
// Decorator: add logging and cache on top of a service
// ============================================================

public interface IOrderService
{
    Task<Order> GetOrderAsync(int orderId, CancellationToken ct);
}

public sealed record Order(int Id, string Customer, decimal Total);

// Базовая реализация (имитация БД) / base implementation (DB mock)
public sealed class OrderService : IOrderService
{
    public Task<Order> GetOrderAsync(int orderId, CancellationToken ct)
    {
        var order = new Order(orderId, "ACME Corp", 199.99m);
        return Task.FromResult(order);
    }
}

// Декоратор логирования / logging decorator
public sealed class LoggingOrderService(IOrderService inner, ILogger logger) : IOrderService
{
    public async Task<Order> GetOrderAsync(int orderId, CancellationToken ct)
    {
        logger.Log($"[LOG] Получение заказа / GetOrder {orderId}");
        var order = await inner.GetOrderAsync(orderId, ct);
        logger.Log($"[LOG] Заказ получен / Order loaded {order.Id}, total {order.Total}");
        return order;
    }
}

// Декоратор кэширования / caching decorator
public sealed class CachingOrderService(IOrderService inner) : IOrderService
{
    private readonly Dictionary<int, Order> _cache = new();

    public async Task<Order> GetOrderAsync(int orderId, CancellationToken ct)
    {
        if (_cache.TryGetValue(orderId, out var cached))
            return cached;                       // Возврат из кэша / return from cache

        var order = await inner.GetOrderAsync(orderId, ct);
        _cache[orderId] = order;                 // Сохраняем в кэш / store in cache
        return order;
    }
}

// Простой логгер для демо / simple logger for the demo
public interface ILogger { void Log(string message); }
public sealed class ConsoleLogger : ILogger
{
    public void Log(string message) => Console.WriteLine(message);
}

// ============================================================
// DI: собираем цепочку декораторов вручную (Scrutor упрощает это)
// DI: assemble the decorator chain manually (Scrutor makes it simpler)
// ============================================================

public static class Program
{
    public static async Task Main()
    {
        var services = new ServiceCollection();
        services.AddSingleton<ILogger, ConsoleLogger>();

        // Регистрация стратегии — выбираем процентную скидку 10%
        // Register strategy — pick a 10% percentage discount
        services.AddSingleton<IDiscountStrategy>(_ => new PercentageDiscountStrategy(10m));
        services.AddSingleton<PriceCalculator>();

        // Регистрация адаптера стороннего SDK / register third-party SDK adapter
        services.AddSingleton<LegacyPayPalSdk>();
        services.AddSingleton<IPaymentGateway, PayPalAdapter>();

        // Регистрация сервиса с декораторами / register service with decorators
        services.AddSingleton<OrderService>();
        services.AddSingleton<IOrderService>(sp =>
        {
            // Порядок: Caching → Logging → OrderService (внешний вызов идёт через кэш)
            // Order: Caching wraps Logging wraps OrderService
            var baseSvc = sp.GetRequiredService<OrderService>();
            var logger = sp.GetRequiredService<ILogger>();
            var logged = new LoggingOrderService(baseSvc, logger);
            return new CachingOrderService(logged);
        });

        var sp = services.BuildServiceProvider();

        // --- Strategy demo ---
        var calculator = sp.GetRequiredService<PriceCalculator>();
        Console.WriteLine($"Цена со скидкой / Discounted price: {calculator.Calculate(100m):F2}");

        // --- Adapter demo ---
        var gateway = sp.GetRequiredService<IPaymentGateway>();
        bool paid = await gateway.ChargeAsync("user-42", 49.90m, CancellationToken.None);
        Console.WriteLine($"Оплата / Payment: {(paid ? "OK" : "FAIL")}");

        // --- Decorator demo (первый вызов — лог + БД, второй — кэш) ---
        var orders = sp.GetRequiredService<IOrderService>();
        _ = await orders.GetOrderAsync(7, CancellationToken.None);
        _ = await orders.GetOrderAsync(7, CancellationToken.None);
    }
}
```

#### Best Practices

- Регистрируйте стратегии в DI по ключу (Keyed Services в .NET 8) или через фабрику, чтобы выбор алгоритма был явным и тестируемым.
- Делайте декораторы тонкими: один декоратор — одна сквозная забота (логирование ИЛИ кэш ИЛИ валидация), не смешивайте их.
- Используйте библиотеку Scrutor (`Decorate<...>`) для автоматической сборки цепочки декораторов вместо ручной регистрации.
- Адаптер должен преобразовывать типы и форматы, но не содержать бизнес-логики — оставьте её в домене.
- Все три паттерна зависят от интерфейсов: проектируйте маленькие, сфокусированные интерфейсы (ISP).
- Добавляйте CancellationToken во все асинхронные методы стратегий, сервисов и адаптеров.

- Register strategies in DI by key (Keyed Services in .NET 8) or via a factory so algorithm selection is explicit and testable.
- Keep decorators thin: one decorator — one cross-cutting concern (logging OR caching OR validation); do not mix them.
- Use the Scrutor library (`Decorate<...>`) to assemble decorator chains automatically instead of manual registration.
- An adapter should convert types and formats but contain no business logic — keep that in the domain.
- All three patterns rely on interfaces: design small, focused interfaces (ISP).
- Add CancellationToken to every async method of strategies, services, and adapters.

#### Частые ошибки / Common Mistakes

- Стратегия выбрана через `if/switch` прямо в клиентском коде вместо интерфейса → вынесите выбор в фабрику или Keyed Services, клиент должен знать только интерфейс.
- Декоратор дублирует логику других декораторов (логирование и кэш в одном классе) → разделите на два декоратора, каждый решает одну задачу.
- Цепочка декораторов зарегистрирована в обратном порядке → проверьте, что внешний декоратор оборачивает внутренний, а не наоборот.
- Адаптер содержит бизнес-правила вместо преобразования → перенесите логику в домен, оставьте в адаптере только маппинг.
- Регистрация нескольких `IOrderService` без явного порядка в DI → используйте фабрику или Scrutor, не полагайтесь на порядок `AddSingleton`.
- Забытый `CancellationToken` в асинхронных методах → передавайте токен через все слои до самого низа.

- Strategy chosen via `if/switch` directly in client code instead of an interface → move selection into a factory or Keyed Services; the client should only know the interface.
- Decorator duplicates logic of other decorators (logging and caching in one class) → split into two decorators, each handling one task.
- Decorator chain registered in reverse order → verify the outer decorator wraps the inner one, not the other way around.
- Adapter contains business rules instead of conversion → move logic to the domain, leave only mapping in the adapter.
- Multiple `IOrderService` registrations without explicit order in DI → use a factory or Scrutor, do not rely on `AddSingleton` order.
- Missing `CancellationToken` in async methods → pass the token through every layer down to the bottom.

#### Чек-лист самопроверки / Self-check Checklist

- [ ] Стратегия объявлена через интерфейс, а не через перечисление в клиенте.
- [ ] Каждая стратегия — отдельный класс с единственной ответственностью.
- [ ] Декораторы реализуют тот же интерфейс, что и оборачиваемый объект.
- [ ] Цепочка декораторов собрана в DI в правильном порядке (кэш снаружи, лог внутри).
- [ ] Адаптер не содержит бизнес-логики, только преобразование интерфейса и типов.
- [ ] Все асинхронные методы принимают `CancellationToken`.
- [ ] Классы открыты для расширения (новая стратегия/декоратор), но закрыты для изменения (OCP).
- [ ] Код компилируется на C# 12 / .NET 8 и покрыт юнит-тестами с mock-объектами.

- [ ] Strategy is declared through an interface, not via an enum in the client.
- [ ] Each strategy is a separate class with a single responsibility.
- [ ] Decorators implement the same interface as the wrapped object.
- [ ] Decorator chain is assembled in DI in the correct order (cache outside, logging inside).
- [ ] Adapter contains no business logic, only interface and type conversion.
- [ ] Every async method accepts a `CancellationToken`.
- [ ] Classes are open for extension (new strategy/decorator) but closed for modification (OCP).
- [ ] Code compiles on C# 12 / .NET 8 and is covered by unit tests with mocks.

#### Ресурсы / Resources

- [Microsoft Learn — https://learn.microsoft.com/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/](https://learn.microsoft.com/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/)
- [Scrutor — DI Decorator helpers](https://github.com/khellang/Scrutor)
- [Refactoring Guru — Strategy, Decorator, Adapter](https://refactoring.guru/design-patterns/strategy)

---

[⬆ К модулю M16](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
