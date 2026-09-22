[⬆ К модулю M16](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M16-L01: SOLID: SRP, OCP, LSP, ISP, DIP с примерами / SOLID: SRP, OCP, LSP, ISP, DIP with examples

**Модуль / Module:** M16
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

Принципы SOLID — это пять фундаментальных правил объектно-ориентированного проектирования, сформулированных Робертом Мартином («Дядюшкой Бобом»). Они помогают создавать код, который легко читать, тестировать, расширять и поддерживать. Давайте разберём каждый из них с аналогиями и примерами.

**S — Single Responsibility Principle (SRP): Принцип единственной ответственности.** У каждого класса должна быть одна и только одна причина для изменения. Представьте швейцарский нож: он режет, открывает бутылки, крутит винты — но делает это посредственно. В коде класс, который одновременно считает зарплату, пишет логи и отправляет email, превращается в «Бога-объект». Разделите его: `SalaryCalculator` только считает, `EmailSender` только отправляет, `Logger` только логирует. Если меняется формула расчёта — трогаем только калькулятор; если SMTP-сервер — только отправитель.

**O — Open/Closed Principle (OCP): Принцип открытости/закрытости.** Классы должны быть открыты для расширения, но закрыты для модификации. Аналогия: автомобиль. Чтобы поставить новую магнитолу, вы не переделываете двигатель — используете стандартный разъём. В коде это достигается через абстракции: вместо `switch` по типам оплаты, добавляющим новые ветки при каждом новом способе, мы вводим интерфейс `IPaymentStrategy` и добавляем новые классы `PayPalPayment`, `CryptoPayment`, не трогая существующий код. Это снижает риск сломать проверенную логику.

**L — Liskov Substitution Principle (LSP): Принцип подстановки Барбары Лисков.** Подкласс должен быть взаимозаменяем со своим базовым классом без сюрпризов. Классический пример нарушения: базовый класс `Rectangle` с методами `Width`/`Height`, а наследник `Square` ломает инвариант — установка ширины меняет и высоту. Если код, ожидающий `Rectangle`, получает `Square` и считает площадь через установку ширины и высоты отдельно, результат неверен. Правило простое: если наследник не может выполнить контракт родителя (выбрасывает `NotSupportedException`, меняет семантику, усиляет предусловия или ослабляет постусловия) — наследование неверно, нужна композиция.

**I — Interface Segregation Principle (ISP): Принцип разделения интерфейсов.** Клиенты не должны зависеть от методов, которые они не используют. Аналогия: розетка. Вам не нужна «универсальная розетка», совмещающая USB, Type-C, евро и британскую вилку — вы выбираете нужную. В коде «толстый» интерфейс `IMachine { Print(); Scan(); Fax(); }` заставляет простой принтер реализовывать пустые методы `Scan()` и `Fax()`. Разделите на `IPrinter`, `IScanner`, `IFaxMachine` — и каждый класс реализует только то, что умеет.

**D — Dependency Inversion Principle (DIP): Принцип инверсии зависимостей.** Модули верхнего уровня не должны зависеть от модулей нижнего уровня; оба должны зависеть от абстракций. Абстракции не должны зависеть от деталей; детали зависят от абстракций. Аналогия: электрическая сеть. Прибор (верхний уровень) не зависит от конкретной электростанции (нижний уровень) — есть стандарт напряжения (абстракция). В коде: `OrderService` не должен создавать `new SqlOrderRepository()` напрямую. Вместо этого он принимает `IOrderRepository` через конструктор, а конкретная реализация (SQL, Mongo, InMemory) подаётся извне через DI-контейнер. Это даёт тестируемость (подставляем mock) и гибкость (меняем источник данных).

**Как принципы работают вместе:** SRP даёт мелкие классы, OCP делает их расширяемыми, LSP гарантирует безопасную замену, ISP держит интерфейсы точечными, а DIP связывает всё через абстракции. SOLID — не догма: чрезмерное применение к простому CRUD-скрипту создаёт переусложнение. Применяйте осознанно, где есть реальная сложность и риск изменений.

#### Theory (EN)

The SOLID principles are five fundamental object-oriented design rules coined by Robert C. Martin ("Uncle Bob"). They guide you toward code that is readable, testable, extensible, and maintainable. Let's walk through each one with analogies and examples.

**S — Single Responsibility Principle (SRP).** A class should have one, and only one, reason to change. Think of a Swiss Army knife: it cuts, opens bottles, and turns screws — but none of these tasks especially well. In code, a class that simultaneously computes payroll, writes logs, and sends emails becomes a "God object." Split it: `SalaryCalculator` only computes, `EmailSender` only sends, `Logger` only logs. If the formula changes, you touch only the calculator; if the SMTP server changes, only the sender.

**O — Open/Closed Principle (OCP).** Classes should be open for extension but closed for modification. Analogy: a car. To install a new stereo, you don't rebuild the engine — you use a standard slot. In code this is achieved through abstractions: instead of a `switch` over payment types that grows a new branch for every new method, you introduce an `IPaymentStrategy` interface and add new classes (`PayPalPayment`, `CryptoPayment`) without modifying existing, proven code. This reduces the risk of breaking working logic.

**L — Liskov Substitution Principle (LSP), named after Barbara Liskov.** A subclass must be substitutable for its base class without surprises. The classic violation: a base `Rectangle` with `Width`/`Height` properties, and a `Square` subclass that breaks the invariant — setting width also changes height. Code expecting a `Rectangle` and computing area by setting width and height separately gets a wrong answer when handed a `Square`. The rule is simple: if a subtype cannot honor the parent's contract (throws `NotSupportedException`, changes semantics, strengthens preconditions, or weakens postconditions), inheritance is wrong — reach for composition instead.

**I — Interface Segregation Principle (ISP).** Clients should not be forced to depend on methods they do not use. Analogy: outlets. You don't need a "universal outlet" combining USB, Type-C, Euro, and British plugs — you pick the one you need. In code, a fat `IMachine { Print(); Scan(); Fax(); }` forces a simple printer to implement empty `Scan()` and `Fax()` methods. Split it into `IPrinter`, `IScanner`, `IFaxMachine`, and each class implements only what it actually does.

**D — Dependency Inversion Principle (DIP).** High-level modules should not depend on low-level modules; both should depend on abstractions. Abstractions should not depend on details; details depend on abstractions. Analogy: the power grid. An appliance (high level) doesn't depend on a specific power plant (low level) — there is a voltage standard (the abstraction). In code: `OrderService` should not `new` up a `SqlOrderRepository` directly. Instead it accepts an `IOrderRepository` through its constructor, and the concrete implementation (SQL, Mongo, InMemory) is supplied from outside via a DI container. This yields testability (inject a mock) and flexibility (swap the data source).

**How the principles work together:** SRP produces small classes, OCP makes them extensible, LSP guarantees safe substitution, ISP keeps interfaces pinpoint, and DIP ties everything together through abstractions. SOLID is not dogma: over-applying it to a trivial CRUD script creates needless complexity. Apply it deliberately where there is genuine complexity and a real risk of change.

#### Пример кода / Code Example

```csharp
// C# 12 / .NET 8 — SOLID principles in action
// Comments: RU + EN

using Microsoft.Extensions.DependencyInjection;

namespace M16.SOLID.Demo;

// === DIP: abstraction lives in the high-level layer ===
// === DIP: абстракция живёт на верхнем уровне ===
public interface IOrderRepository
{
    Task<Order> GetByIdAsync(Guid id, CancellationToken ct = default);
    Task SaveAsync(Order order, CancellationToken ct = default);
}

// === DIP: detail depends on abstraction, not the other way ===
// === DIP: деталь зависит от абстракции, а не наоборот ===
public sealed class SqlOrderRepository(IConnectionFactory factory) : IOrderRepository
{
    public async Task<Order> GetByIdAsync(Guid id, CancellationToken ct = default)
    {
        // Реализация SQL-чтения / SQL read implementation
        await using var conn = factory.Create();
        return await conn.QuerySingleAsync<Order>(
            "SELECT * FROM Orders WHERE Id = @id", new { id }, cancellationToken: ct);
    }

    public Task SaveAsync(Order order, CancellationToken ct = default) =>
        throw new NotImplementedException("Демонстрационный код / demo stub");
}

// === SRP: этот класс только оформляет заказ / this class only places an order ===
public sealed class OrderService(
    IOrderRepository repository,            // DIP: зависим от абстракции / depend on abstraction
    IPaymentGateway payment,                // DIP + ISP: точечный интерфейс / focused interface
    INotificationService notifier)          // DIP + ISP: ещё один точечный интерфейс
{
    public async Task<Guid> PlaceAsync(Order order, CancellationToken ct = default)
    {
        order.Status = OrderStatus.AwaitingPayment;
        await payment.ChargeAsync(order.Total, order.CustomerId, ct);
        await repository.SaveAsync(order, ct);
        await notifier.NotifyAsync(order.CustomerId, "Заказ оформлен / Order placed", ct);
        return order.Id;
    }
}

// === OCP + Strategy: новый способ оплаты = новый класс, старый код не трогаем ===
// === OCP + Strategy: a new payment method = a new class, existing code untouched ===
public interface IPaymentGateway
{
    Task ChargeAsync(decimal amount, Guid customerId, CancellationToken ct = default);
}

public sealed class CreditCardPayment(HttpClient http) : IPaymentGateway
{
    public Task ChargeAsync(decimal amount, Guid customerId, CancellationToken ct = default)
        => http.PostAsJsonAsync("/charge", new { amount, customerId }, ct);
}

public sealed class CryptoPayment(ICryptoClient crypto) : IPaymentGateway
{
    public Task ChargeAsync(decimal amount, Guid customerId, CancellationToken ct = default)
        => crypto.TransferAsync(amount, customerId, ct);
}

// === ISP: интерфейсы разделены по ролям / interfaces split by role ===
// === ISP: clients depend only on what they use ===
public interface INotificationService
{
    Task NotifyAsync(Guid userId, string message, CancellationToken ct = default);
}

public interface IReportService // отдельная роль, не втиснута в INotificationService
{
    Task<byte[]> GenerateInvoiceAsync(Guid orderId, CancellationToken ct = default);
}

// === LSP: подтипы не ломают контракт базового типа / subtypes honor the base contract ===
public abstract class Discount
{
    public abstract decimal Apply(decimal total);

    // Контракт: всегда возвращает неотрицательную сумму >= 0
    // Contract: always returns a non-negative value
    public virtual decimal Validate(decimal total, decimal discounted)
        => discounted < 0 ? throw new InvalidOperationException("Скидка отрицательна / negative discount")
                          : discounted;
}

public sealed class PercentDiscount(decimal percent) : Discount
{
    public override decimal Apply(decimal total)
    {
        var discounted = total - total * percent;
        return Validate(total, discounted); // LSP: выполняем контракт предка
    }
}

// sealed class FlatDiscount: Discount — также соблюдает контракт.
// Если бы мы создали "ZeroDiscount", который выбрасывал для total > 0,
// это нарушило бы LSP — его нельзя безопасно подставить вместо Discount.
// A "ZeroDiscount" that throws for total > 0 would violate LSP — unsafe substitution.

// === Composition Root: всё связывается здесь, классы ничего не знают о DI ===
public static class CompositionRoot
{
    public static ServiceProvider Build()
    {
        var services = new ServiceCollection();
        services.AddHttpClient<CreditCardPayment>();
        services.AddSingleton<IConnectionFactory, SqlConnectionFactory>();
        services.AddScoped<IOrderRepository, SqlOrderRepository>();
        services.AddScoped<IPaymentGateway, CreditCardPayment>(); // поменять на CryptoPayment — одной строкой
        services.AddScoped<INotificationService, EmailNotifier>();
        services.AddScoped<OrderService>();
        return services.BuildServiceProvider();
    }
}

public sealed record Order(Guid Id, Guid CustomerId, decimal Total, OrderStatus Status);
public enum OrderStatus { AwaitingPayment, Paid, Shipped }
public interface IConnectionFactory { IDbConnection Create(); }
public interface ICryptoClient { Task TransferAsync(decimal amount, Guid customerId, CancellationToken ct); }
public sealed class EmailNotifier : INotificationService
{
    public Task NotifyAsync(Guid userId, string message, CancellationToken ct = default) => Task.CompletedTask;
}
internal static class DapperExtensions // заглушки для демонстрации компиляции / demo stubs
{
    public static Task<T> QuerySingleAsync<T>(this IDbConnection _, string sql, object param, CancellationToken cancellationToken) => Task.FromResult<T>(default!);
    public static Task PostAsJsonAsync(this HttpClient _, string uri, object value, CancellationToken ct) => Task.CompletedTask;
}
```

#### Best Practices

- Разделяйте классы по причинам изменения: если класс трогают из-за двух разных бизнес-процессов — нарушен SRP. Используйте фичи C# 12 — первичные конструкторы, `sealed`, `record`, файловые пространства имён — чтобы классы оставались компактными.
- Проектируйте от интерфейсов, а не от реализаций: `OrderService` должен принимать `IOrderRepository`, а не `SqlOrderRepository`. Регистрируйте реализации в Composition Root, а не раскидывайте `new` по коду.
- Предпочитайте композицию наследованию, если подкласс не может выполнить контракт предка. Помечайте классы `sealed` по умолчанию — это явно запрещает некорректное наследование и помогает JIT-компилятору.
- Держите интерфейсы маленькими и ролевыми: `INotificationService` отдельно от `IReportService`. Не делайте «божественные» интерфейсы на десятки методов.
- Покрывайте принципы тестами: если тест OrderService требует реальной БД — нарушение DIP; подставьте `InMemoryOrderRepository`.

- Split classes by reason for change: if two business processes touch the same class, SRP is violated. Use C# 12 features — primary constructors, `sealed`, `record`, file-scoped namespaces — to keep classes compact.
- Design against interfaces, not implementations: `OrderService` should take `IOrderRepository`, not `SqlOrderRepository`. Register implementations in the Composition Root, not scattered `new` calls.
- Prefer composition over inheritance when a subtype cannot honor the parent's contract. Mark classes `sealed` by default — it explicitly forbids unsafe inheritance and helps the JIT compiler.
- Keep interfaces small and role-based: `INotificationService` separate from `IReportService`. Avoid "god" interfaces with dozens of methods.
- Cover the principles with tests: if an `OrderService` test needs a real DB, DIP is broken — inject an `InMemoryOrderRepository`.

#### Частые ошибки / Common Mistakes

- `switch` по типам сущностей, растущий при каждом новом типе → вводите полиморфизм или Strategy (OCP). / A `switch` over entity types that grows with every new type → introduce polymorphism or Strategy (OCP).
- «Бог-объект» с логикой, данными, доступом к БД и UI в одном классе → разбивайте на мелкие классы по SRP. / A "God object" mixing logic, data, DB access, and UI in one class → split into small SRP classes.
- Наследник выбрасывает `NotSupportedException` для метода предка → замените наследование композицией или адаптером (LSP). / A subtype throws `NotSupportedException` for a base method → replace inheritance with composition or an adapter (LSP).
- Толстый интерфейс, где большинство реализаций оставляют методы пустыми → разделите интерфейс на ролевые части (ISP). / A fat interface where most implementations leave methods empty → split it into role-based parts (ISP).
- `new SqlRepository()` внутри бизнес-класса → принимайте `IRepository` через конструктор и регистрируйте в DI (DIP). / `new SqlRepository()` inside a business class → take `IRepository` through the constructor and register in DI (DIP).
- Сверх-абстракции на каждый чих: интерфейс + фабрика + провайдер для тривиального значения → SOLID — не догма, применяйте осознанно. / Over-abstraction for everything: an interface + factory + provider for a trivial value → SOLID is not dogma; apply it deliberately.
- Тест требует реальной БД или SMTP → это симптом нарушения DIP; вводите абстракцию и подставляйте фейк. / A test needs a real DB or SMTP → a symptom of DIP violation; introduce an abstraction and inject a fake.
- Нарушение инварианта при наследовании (Square вместо Rectangle) → проверяйте контракт в тестах подстановки (LSP). / Broken invariant under inheritance (Square vs Rectangle) → guard the contract with substitution tests (LSP).

#### Чек-лист самопроверки / Self-check Checklist

- [ ] У класса одна причина для изменения (SRP).
- [ ] Новый функционал добавляется новым классом, а не правкой старого (OCP).
- [ ] Подкласс можно подставить вместо базового без нарушения контракта (LSP).
- [ ] Ни один клиент не зависит от методов, которые не использует (ISP).
- [ ] Высокоуровневый модуль зависит от абстракции, а не от детали (DIP).
- [ ] Все зависимости подаются через конструктор и регистрируются в Composition Root.
- [ ] Классы помечены `sealed`, где наследование не планируется.
- [ ] Юнит-тесты используют фейки/моки, а не реальные внешние ресурсы.
- [ ] Нет «божественных» интерфейсов и «бог-объектов».
- [ ] Код компилируется и работает на C# 12 / .NET 8.
- [ ] The class has a single reason to change (SRP).
- [ ] New functionality is added via a new class, not by editing existing code (OCP).
- [ ] A subtype can replace its base class without breaking the contract (LSP).
- [ ] No client depends on methods it does not use (ISP).
- [ ] The high-level module depends on an abstraction, not a detail (DIP).
- [ ] All dependencies are injected via the constructor and registered in the Composition Root.
- [ ] Classes are `sealed` where inheritance is not intended.
- [ ] Unit tests use fakes/mocks, not real external resources.
- [ ] No "god" interfaces or "God objects" exist.
- [ ] The code compiles and runs on C# 12 / .NET 8.

#### Ресурсы / Resources

- Microsoft Learn — https://learn.microsoft.com/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/
- Robert C. Martin, «Clean Architecture» (основы SOLID).
- Microsoft Docs — Dependency injection in .NET: https://learn.microsoft.com/dotnet/core/extensions/dependency-injection
- Source-making — SOLID principles: https://wiki.c2.com/?PrinciplesOfObjectOrientedDesign

---

[⬆ К модулю M16](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
