[⬆ К модулю M15](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M15-L09: Тестируемость как архитектурное свойство (вступление к M16) / Testability as architectural property (intro to M16)

**Модуль / Module:** M15
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

Тестируемость — это не «наличие unit-тестов», а архитектурное свойство системы: способность её компонентов проверяться изолированно, предсказуемо и быстро. Если код тяжело тестировать, почти всегда это симптом более глубокой проблемы — высокого зацепления (coupling) и слабой связности. Тесты лишь подсвечивают архитектурные дефекты, как рентген подсвечивает перелом: боль не в рентгене, а в кости.

Представьте автомобиль. Двигатель можно снять, поставить на стенд и проверить отдельно — потому что он соединён с остальной машиной через чёткие «швы»: крепления, шланги, электрические разъёмы. В коде такие швы (seams) создаются через интерфейсы и внедрение зависимостей (Dependency Injection, DI). Класс, который сам создаёт `new HttpClient()` или `new DateTimeProvider()`, не имеет шва — вы не сможете подменить время или сеть в тесте. Класс, принимающий `IHttpClient` и `IClock` через конструктор, имеет шов: в тесте вы передаёте фейк, в продакшене — реальную реализацию.

Принцип инверсии зависимостей (Dependency Inversion Principle, DIP) гласит: модули высокого уровня не должны зависеть от модулей низкого уровня; оба должны зависеть от абстракций. Это и есть фундамент тестируемости. Когда доменная логика зависит от интерфейсов, а не от конкретных классов БД или HTTP, вы можете проверить бизнес-правила вообще без инфраструктуры. Это и есть «чистая архитектура» в действии: зависимости направлены внутрь, к домену.

Чистые функции (pure functions) — самый тестируемый код из возможных. Функция `decimal CalculateDiscount(Order order)` без побочных эффектов и внешних вызовов тестируется одной строкой: `Assert.Equal(10m, CalculateDiscount(sampleOrder))`. Чем больше логики вынести в чистые функции, тем меньше нужно моков, тем быстрее и надёжнее тесты. Состояние и ввод-вывод — «грязная» зона; чистая доменная логика — «чистая» зона. Разделяйте их.

Цикломатическая сложность и количество коллабораторов (классов, с которыми работает тестируемый юнит) — количественные индикаторы тестируемости. Если для теста одного метода нужно создать 7 моков — это запах: класс делает слишком много, нарушает Single Responsibility. Вместо мокирования всего подряд лучше декомпозировать класс и ввести абстракции.

Практический вывод: тестируемость достигается не «больше тестов», а осознанной архитектурой — DI, интерфейсами, чистыми функциями, инверсией зависимостей и контролем зацепления. Это плавно подводит нас к модулю M16, где мы формализуем те же принципы через SOLID и шаблоны архитектуры приложений (Clean Architecture, DDD, CQRS). Тестируемость — не побочный эффект хорошей архитектуры, а её мера.

#### Theory (EN)

Testability is not “having unit tests”; it is an architectural property of a system: the ability of its components to be verified in isolation, predictably and quickly. When code is hard to test, the root cause is almost always deeper — high coupling and low cohesion. Tests merely expose architectural defects, the way an X-ray exposes a fracture: the pain is not in the X-ray, it is in the bone.

Think of a car engine. You can unbolt it, mount it on a test bench and measure its output separately — because it is connected to the rest of the vehicle through well-defined seams: mounts, hoses, electrical connectors. In code, such seams are created through interfaces and Dependency Injection (DI). A class that itself does `new HttpClient()` or `new DateTimeProvider()` has no seam — you cannot substitute time or the network in a test. A class that accepts `IHttpClient` and `IClock` through its constructor has a seam: in a test you pass a fake, in production you pass the real implementation.

The Dependency Inversion Principle (DIP) says that high-level modules should not depend on low-level modules; both should depend on abstractions. This is the very foundation of testability. When your domain logic depends on interfaces rather than concrete database or HTTP classes, you can verify business rules with zero infrastructure. That is Clean Architecture in action: dependencies point inward, toward the domain.

Pure functions are the most testable code possible. A function `decimal CalculateDiscount(Order order)` with no side effects and no external calls is tested in a single line: `Assert.Equal(10m, CalculateDiscount(sampleOrder))`. The more logic you push into pure functions, the fewer mocks you need and the faster and more reliable your tests become. State and I/O live in a “dirty” zone; pure domain logic lives in a “clean” zone. Keep them separated.

Cyclomatic complexity and the number of collaborators (classes the unit under test works with) are quantitative indicators of testability. If testing one method requires creating seven mocks, that is a smell: the class is doing too much and violating Single Responsibility. Rather than mocking everything in sight, decompose the class and introduce abstractions.

The practical takeaway: testability is achieved not by “more tests” but by deliberate architecture — DI, interfaces, pure functions, dependency inversion and conscious control of coupling. This sets the stage for Module M16, where we formalize the same ideas through SOLID and application-architecture patterns (Clean Architecture, DDD, CQRS). Testability is not a by-product of good architecture — it is its measure.

#### Пример кода / Code Example

```csharp
// C# 12 / .NET 8 — Тестируемость через швы (seams), DI и чистые функции
// Testability via seams, DI and pure functions

using Microsoft.Extensions.DependencyInjection;

// --- Шов: абстракции, от которых зависит домен / Seams: abstractions the domain depends on ---
public interface IClock
{
    DateTime UtcNow { get; } // позволяет подменить время в тесте / lets tests substitute time
}

public interface IPriceRepository
{
    decimal GetBasePrice(string sku);
}

// --- Чистая функция: нет состояния, нет I/O → тривиально тестируется ---
// Pure function: no state, no I/O → trivially testable
public static class DiscountPolicy
{
    public static decimal CalculateDiscount(Order order, DateTime utcNow)
    {
        // Корректность проверяется без моков / correctness verified without mocks
        var seasonal = utcNow.Month is 12 or 1 ? 0.10m : 0m;
        var bulk = order.Quantity >= 100 ? 0.05m : 0m;
        return seasonal + bulk;
    }
}

public sealed record Order(string Sku, int Quantity);

// --- Высокоуровневый доменный сервис зависит только от абстракций (DIP) ---
// High-level domain service depends only on abstractions (DIP)
public sealed class OrderPricer(IClock clock, IPriceRepository prices)
{
    public PricedOrder Price(Order order)
    {
        var basePrice = prices.GetBasePrice(order.Sku);
        var discount = DiscountPolicy.CalculateDiscount(order, clock.UtcNow);
        var total = basePrice * order.Quantity * (1m - discount);
        return new PricedOrder(order.Sku, order.Quantity, basePrice, discount, total);
    }
}

public sealed record PricedOrder(string Sku, int Quantity, decimal BasePrice, decimal Discount, decimal Total);

// --- Инфраструктура: реальная реализация, подключается в продакшене ---
// Infrastructure: real implementation wired up in production
internal sealed class SystemClock : IClock
{
    public DateTime UtcNow => DateTime.UtcNow;
}

internal sealed class DbPriceRepository : IPriceRepository
{
    public decimal GetBasePrice(string sku) =>
        // здесь был бы реальный запрос к БД / real DB call would live here
        sku switch { "A1" => 10m, "B2" => 25m, _ => 1m };
}

// --- Конфигурация DI: шов зашивается один раз на границе системы ---
// DI composition: the seam is stitched once at the system boundary
public static class CompositionRoot
{
    public static ServiceProvider Build() => new ServiceCollection()
        .AddSingleton<IClock, SystemClock>()
        .AddSingleton<IPriceRepository, DbPriceRepository>()
        .AddSingleton<OrderPricer>()
        .BuildServiceProvider();
}
```

```csharp
// Тесты: благодаря шовам не нужна БД и не нужно реальное время
// Tests: thanks to seams, no DB and no real time required
using Xunit;

internal sealed class FakeClock(DateTime fixedUtc) : IClock
{
    public DateTime UtcNow => fixedUtc;
}

internal sealed class StubPrices : IPriceRepository
{
    public decimal GetBasePrice(string sku) => 10m; // детерминированный ответ / deterministic
}

public class OrderPricerTests
{
    [Fact]
    public void Bulk_order_in_december_gets_combined_discount()
    {
        var pricer = new OrderPricer(
            clock: new FakeClock(new DateTime(2024, 12, 15, 0, 0, 0, DateTimeKind.Utc)),
            prices: new StubPrices());

        var priced = pricer.Price(new Order("A1", 100));

        Assert.Equal(10m, priced.BasePrice);
        Assert.Equal(0.15m, priced.Discount);          // 10% seasonal + 5% bulk
        Assert.Equal(850m, priced.Total);              // 10 * 100 * 0.85
    }
}
```

#### Best Practices
- Зависите от абстракций, а не от конкретных классов; внедряйте зависимости через конструктор (DIP).
- Выносите доменные правила в чистые функции без побочных эффектов — их не нужно мокать.
- Размещайте «швы» на границах слоёв (домен ↔ инфраструктура), внутри слоя старайтесь работать с конкретными типами.
- Один класс — одна ответственность: если для теста нужно 5+ моков, это сигнал к декомпозиции.
- Тестируйте поведение, а не реализацию; не привязывайтесь к внутренним полям и порядку вызовов.
- Depend on abstractions, not concrete classes; inject dependencies through the constructor (DIP).
- Move domain rules into pure functions with no side effects — they need no mocking.
- Place seams at layer boundaries (domain ↔ infrastructure); inside a layer, concrete types are fine.
- One class, one responsibility: needing 5+ mocks to test a unit is a signal to decompose.
- Test behaviour, not implementation; do not assert on private fields or call order.

#### Частые ошибки / Common Mistakes
- `new` внутри метода домена (`new HttpClient()`, `DateTime.Now`) → принимайте зависимость через конструктор, чтобы появился шов для теста.
- Мокирование всего подряд, включая мапперы и DTO → тестируйте через чистые функции и реальные объекты-значения.
- Тесты, проверяющие «вызвался ли метод 2 раза» → проверяйте итоговое состояние и результат, а не счётчик вызовов.
- Сетевые/БД-вызовы в unit-тестах → изолируйте их за интерфейсом и тестируйте инфраструктуру отдельно (integration tests).
- Тесты, ломающиеся при переименовании приватного поля → тестируйте публичный контракт, а не внутренности.
- `new` inside a domain method (`new HttpClient()`, `DateTime.Now`) → accept the dependency via the constructor so a test seam appears.
- Mocking everything including mappers and DTOs → test through pure functions and real value objects.
- Tests asserting “method was called twice” → assert final state and result, not call counts.
- Network/DB calls inside unit tests → isolate them behind an interface and test infra separately (integration tests).
- Tests that break when a private field is renamed → test the public contract, not internals.

#### Чек-лист самопроверки / Self-check Checklist
- [ ] Каждый доменный класс получает зависимости через интерфейсы конструктора, без `new` в методах.
- [ ] Чистые функции вынесены и покрыты тестами без единого мока.
- [ ] Время, случайность, сеть, БД скрыты за интерфейсами (IClock, IRandom, IRepository…).
- [ ] Тесты запускаются изолированно и параллельно, не ломаясь от состояния окружения.
- [ ] Ни один unit-тест не обращается к реальной БД, файловой системе или сети.
- [ ] Each domain class receives dependencies via constructor interfaces, with no `new` in methods.
- [ ] Pure functions are extracted and covered by tests without any mocks.
- [ ] Time, randomness, network and DB are hidden behind interfaces (IClock, IRandom, IRepository…).
- [ ] Tests run in isolation and in parallel, unaffected by environment state.
- [ ] No unit test touches a real database, file system or network.

#### Ресурсы / Resources
- [Microsoft Learn — .NET Microservices: DDD/CQRS patterns](https://learn.microsoft.com/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/)
- [Microsoft Learn — Dependency injection in .NET](https://learn.microsoft.com/dotnet/core/extensions/dependency-injection)
- [xunit.net — Writing tests](https://xunit.net/docs/getting-started/netcore/cmdline)

---

[⬆ К модулю M15](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
