---
[← К уроку M16-L01](lesson-M16-L01-solid.md) | [⬆ К модулю M16](../README.md) | [Следующее ДЗ →](homework-M16-L02-di-deep.md)
---

### Домашнее задание M16-L01: SOLID: SRP, OCP, LSP, ISP, DIP с примерами / Homework M16-L01: SOLID: SRP, OCP, LSP, ISP, DIP with examples

**Урок / Lesson:** M16-L01
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 4/5
**Цель / Goal:** (RU) Спроектировать и реализовать модуль обработки заказов магазина, в котором каждый из пяти принципов SOLID применён осознанно и проверим: классы разделены по причинам изменения (SRP), новые скидки и способы оплаты добавляются без правок существующего кода (OCP), подтипы скидок соблюдают контракт базового класса (LSP), интерфейсы уведомлений разделены по ролям (ISP), а высокоуровневый сервис зависит только от абстракций, собираемых в Composition Root (DIP). (EN) Design and implement a shop order-processing module in which every SOLID principle is applied deliberately and verifiably: classes are split by reason for change (SRP), new discounts and payment methods are added without editing existing code (OCP), discount subtypes honor the base contract (LSP), notification interfaces are split by role (ISP), and the high-level service depends only on abstractions wired in the Composition Root (DIP).

#### Связь с уроком / Connection to the lesson
(RU) Урок вводит пять принципов SOLID с аналогиями и единым кодовым примером: `OrderService`, принимающий `IOrderRepository`, `IPaymentGateway` и `INotificationService` через конструктор, абстрактный `Discount` с контрактом валидации, и Strategy для способов оплаты. Это ДЗ превращает теорию в практику: вы строите аналогичную систему с нуля, добавляете юнит-тесты с фейками и сами убеждаетесь, что каждый принцип работает — а не просто декларируется.
(EN) The lesson introduces the five SOLID principles with analogies and a single code example: an `OrderService` that takes `IOrderRepository`, `IPaymentGateway`, and `INotificationService` through its constructor, an abstract `Discount` with a validation contract, and a Strategy for payment methods. This homework turns theory into practice: you build a similar system from scratch, add unit tests with fakes, and convince yourself that each principle actually holds — not just is declared.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация
Представьте, что вы пришли в стартап «Шардик-Маркет», который продаёт цифровые курсы. Текущий код обработки заказов — это один класс `OrderManager` на 600 строк: он читает заказ из SQL-базы, считает скидку через гигантский `switch` по типам клиентов, списывает деньги через шлюз конкретного банка, шлёт email через SmtpClient, печатает PDF-инвойс и ещё пишет в лог-файл. Каждое изменение — будь то новая скидка «чёрная пятница», новый способ оплаты криптой или переход с email на Telegram — заставляет править этот класс, ломать existing-тесты и нервировать релиз-инженеров. Менеджер просит вас переписать модуль так, чтобы добавление нового варианта не требовало правки уже работающего кода, тесты не зависели от реальной базы и SMTP, а классы оставались компактными и читаемыми.

Ваша задача — спроектировать модуль `M16.SOLID.Homework` на C# 12 / .NET 8, в котором сознательно применены все пять принципов SOLID. Вы не переписываете старый код вслепую, а строите новую архитектуру рядом, показывая, как каждый принцип решает конкретную боль стартапа. Параллельно вы пишете юнит-тесты с фейковыми реализациями репозитория и нотификатора — это и есть проверка DIP: если тестам нужна реальная база или SMTP, принцип нарушен. В финале вы собираете всё в Composition Root через `Microsoft.Extensions.DependencyInjection` и демонстрируете, что смена способа оплаты или скидки — это одна строка регистрации.

#### Что нужно сделать (пошагово)
1. Создайте решение и проект консольного приложения: `dotnet new sln -n ShardikMarket`, затем `dotnet new console -n M16.SOLID.Homework -o src/M16.SOLID.Homework --framework net8.0`, добавьте его в решение `dotnet sln add src/M16.SOLID.Homework`. Включите nullable-контекст и LangVersion 12 в `.csproj`: `<Nullable>enable</Nullable>` и `<LangVersion>12</LangVersion>`.
2. Добавьте пакет DI: `dotnet add src/M16.SOLID.Homework package Microsoft.Extensions.DependencyInjection` и `dotnet add src/M16.SOLID.Homework package Microsoft.Extensions.Hosting`. Создайте проект тестов: `dotnet new xunit -n M16.SOLID.Homework.Tests -o tests/M16.SOLID.Homework.Tests`, `dotnet sln add tests/M16.SOLID.Homework.Tests`, `dotnet add tests/M16.SOLID.Homework.Tests reference src/M16.SOLID.Homework`.
3. Определите доменную модель в файле `Domain.cs`: `public enum OrderStatus { New, Paid, Shipped, Cancelled }` и `public sealed record Order(Guid Id, Guid CustomerId, decimal Total, OrderStatus Status);`. Используйте `record` и `sealed` — это best practice из урока для компактных иммутабельных значений.
4. Реализуйте DIP-репозиторий: интерфейс `IOrderRepository { Task<Order?> GetByIdAsync(Guid, CancellationToken); Task SaveAsync(Order, CancellationToken); }` и класс `InMemoryOrderRepository` со словарём `Dictionary<Guid, Order>`. Репозиторий должен быть `sealed`.
5. Реализуйте OCP + LSP для скидок: интерфейс `IDiscountPolicy { decimal Apply(Order); }`, абстрактный `DiscountBase` с защищённым методом `Enforce(Order, decimal)`, который бросает `InvalidOperationException`, если скидка отрицательна или превышает `order.Total` — это и есть LSP-контракт. Сделайте три подтипа: `PercentDiscount(decimal percent)`, `FixedDiscount(decimal amount)`, `NoDiscount`. Все — `sealed` и используют первичные конструкторы C# 12.
6. Реализуйте OCP для оплаты: `IPaymentGateway { Task ChargeAsync(decimal, Guid, CancellationToken); }` и две реализации — `CreditCardPayment` и `CryptoPayment`. Пусть `CreditCardPayment` принимает `HttpClient` через первичный конструктор, а `CryptoPayment` — `ICryptoClient`.
7. Реализуйте ISP: разделите уведомления на `INotifier { Task NotifyAsync(Guid, string, CancellationToken); }` и `IInvoiceGenerator { Task<byte[]> GenerateAsync(Guid orderId, CancellationToken); }`. Не делайте один «толстый» `INotificationService` с обоими методами. Реализуйте `EmailNotifier : INotifier` и `PdfInvoiceGenerator : IInvoiceGenerator`.
8. Реализуйте SRP-сервис `OrderProcessor` с первичным конструктором, принимающим `IOrderRepository`, `IDiscountPolicy`, `IPaymentGateway`, `INotifier`. Метод `PlaceAsync` считает итог через скидку, списывает деньги, сохраняет заказ, шлёт уведомление. Никакой логики логирования, PDF или SQL внутри.
9. Соберите Composition Root в `Program.cs` через `Host.CreateDefaultBuilder` и `IServiceCollection`: зарегистрируйте `InMemoryOrderRepository` как `Scoped`, скидку — `Singleton<IDiscountPolicy, PercentDiscount>` (параметр передайте через фабрику), оплату, нотификатор и `OrderProcessor`. Выведите в консоль ID оформленного заказа.
10. Напишите юнит-тесты в `OrderProcessorTests.cs`: используйте фейки `FakeOrderRepository` и `FakeNotifier`, проверьте, что после `PlaceAsync` заказ сохранён со статусом `Paid`, вызван `ChargeAsync`, вызвана нотификация. Напишите `DiscountTests.cs`: для `PercentDiscount(0.1m)` и заказа 100 итог 90; для `NoDiscount` итог равен `Total`. Напишите LSP-контрактный тест: проверьте, что все подтипы скидок для `Order` с `Total = 100` возвращают значение в диапазоне `[0, 100]`.
11. Запустите `dotnet build` и `dotnet test` — всё должно компилироваться и проходить. Запустите `dotnet run --project src/M16.SOLID.Homework` — в консоли должен появиться GUID нового заказа.
12. Сделайте эксперимент OCP: добавьте новый класс `LoyalCustomerDiscount` в отдельный файл, не трогая `OrderProcessor` и существующие скидки. Зарегистрируйте его одной строкой в Composition Root и убедитесь, что тесты остаются зелёными.

#### Требования к решению
- Целевой фреймворк `net8.0`, язык C# 12, файловые пространства имён, nullable-контекст включён. Используйте первичные конструкторы, `sealed`, `record`, collection expressions и pattern matching там, где они уместны.
- Все классы реализации помечены `sealed`, кроме абстрактного `DiscountBase`, который служит контрактом. Интерфейсы маленькие и ролевые — не более 2–3 методов каждый.
- `OrderProcessor` не создаёт `new` для репозитория, оплаты, скидки или нотификатора — все зависимости приходят через конструктор. Внутри `OrderProcessor` нет ни слова о SQL, SMTP, PDF, HttpClient или лог-файлах.
- Скидки реализованы через интерфейс `IDiscountPolicy` и абстрактный `DiscountBase` с защищённым контрактом `Enforce`. Любой подтип, нарушающий инвариант `[0, order.Total]`, выбрасывает исключение — это и есть LSP-защита.
- Composition Root — единственное место, где знают о конкретных реализациях. Смена скидки с `PercentDiscount` на `LoyalCustomerDiscount` — одна строка регистрации.
- Юнит-тесты не обращаются к реальной базе, сети или файловой системе. Все внешние зависимости подменены фейками. Если для теста нужен `HttpClient`, используйте `IHttpClientFactory` или фейковый шлюз.
- Код компилируется без warning-ов уровня error, `dotnet test` зелёный, `dotnet run` выводит GUID заказа.

#### Тонкости и подводные камни
- Не путайте SRP «одна причина для изменения» с «один метод на класс». Класс может содержать несколько методов, если они все относятся к одной ответственности. `OrderProcessor.PlaceAsync` оркестрирует, но не реализует детали — это нормально.
- Главная ловушка OCP — соблазн добавить `switch` по типу клиента внутри `OrderProcessor`. Если вы видите `switch (order.CustomerType)` — это сигнал, что нужен полиморфизм или Strategy. Новая скидка должна быть новым классом, а не новой веткой `switch`.
- LSP-ловушка: подтип, который бросает `NotSupportedException` для `Order` с `Total = 0`, выглядит безобидно, но ломает контракт, если базовый класс обещает «всегда возвращает число». Используйте контрактный тест, перебирающий все реализации `IDiscountPolicy` через рефлексию или явный список.
- ISP часто нарушают «на вырост»: разработчик кладёт в `INotifier` метод `GenerateInvoiceAsync`, потому что «потом пригодится». Не кладите. Если `OrderProcessor` не печатает инвойс, ему не нужен этот метод. Сделайте отдельный `IInvoiceGenerator`, который использует другой сервис.
- DIP-ловушка: регистрация `OrderProcessor` как `Singleton`, а `IOrderRepository` как `Scoped` приведёт к captured dependency — синглтон захватит первый scoped-объект на всё время жизни. Регистрируйте `OrderProcessor` как `Scoped` или `Transient`, если его зависимости scoped.
- Не переусложняйте: не создавайте `IOrderProcessorFactory` + `OrderProcessorFactoryProvider` для тривиальной логики. SOLID — не догма, как напоминает урок. Применяйте, где есть реальная сложность и риск изменений.
- Проверьте, что `CancellationToken` пробрасывается во все асинхронные методы — это часть контракта и best practice, который упрощает тестирование отмены.
- Фейки в тестах — это классы-заглушки, а не обязательно Mock-библиотека. Простой `FakeNotifier : INotifier` со списком вызовов часто читаемее, чем `Mock<INotifier>`.

#### Критерии приёмки
- [ ] Решение собирается командой `dotnet build` без ошибок на .NET 8 / C# 12.
- [ ] `dotnet test` проходит все тесты без пропусков.
- [ ] `dotnet run` выводит GUID оформленного заказа в консоль.
- [ ] `OrderProcessor` принимает все зависимости через первичный конструктор и не содержит `new` для внешних ресурсов.
- [ ] Скидки реализованы через `IDiscountPolicy` + `DiscountBase` с контрактом `Enforce`.
- [ ] Добавление `LoyalCustomerDiscount` не требует правки `OrderProcessor` и существующих скидок.
- [ ] `INotifier` и `IInvoiceGenerator` — раздельные интерфейсы, ни один клиент не зависит от лишних методов.
- [ ] Все классы-реализации помечены `sealed`, кроме `DiscountBase`.
- [ ] `IOrderRepository` реализован через `InMemoryOrderRepository`, тесты используют фейк или этот класс без реальной БД.
- [ ] Composition Root — единственное место со знанием о конкретных реализациях.
- [ ] Существует контрактный тест, проверяющий инвариант `[0, order.Total]` для всех скидок.
- [ ] `CancellationToken` пробрасывается во все асинхронные методы с дефолтным значением.
- [ ] Доменная модель — `sealed record`, иммутабельная.
- [ ] Nullable-контекст включён, нет warning-ов о nullability.
- [ ] Нет «бог-объекта» и «божественного» интерфейса; ни один интерфейс не содержит больше трёх методов.
- [ ] В README или комментарии указано, какой принцип где применён (SRP/OCP/LSP/ISP/DIP).

#### Подсказки (без прямого ответа)
- Если не знаете, где граница SRP — спросите: «Сколько причин у этого класса измениться?» Если больше одной — разделите.
- Для OCP спросите: «Что я правлю, когда появляется новый способ оплаты?» Если правите `OrderProcessor` — нарушен OCP.
- Для LSP напишите параметризованный тест, который принимает `Func<IDiscountPolicy>` и проверяет контракт для всех фабрик.
- Для ISP спросите: «Какой метод из этого интерфейса не вызывает клиент?» Если хоть один — разделите.
- Для DIP спросите: «Могу ли я протестировать `OrderProcessor` без БД и сети?» Если нет — нарушен DIP.
- Используйте `ActivatorUtilities.CreateInstance` или фабрику лямбду в `AddSingleton<IDiscountPolicy>(_ => new PercentDiscount(0.1m))`, чтобы передать параметр конструктора.

#### Эталонное решение (разбор)

```csharp
// C# 12 / .NET 8 — M16.SOLID.Homework
// Комментарии RU + EN
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;

namespace M16.SOLID.Homework;

// === Доменная модель: иммутабельный record + enum ===
// === Domain model: immutable record + enum ===
public enum OrderStatus { New, Paid, Shipped, Cancelled }
public sealed record Order(Guid Id, Guid CustomerId, decimal Total, OrderStatus Status);

// === DIP: репозиторий как абстракция, InMemory — деталь ===
// === DIP: repository as abstraction, InMemory is a detail ===
public interface IOrderRepository
{
    Task<Order?> GetByIdAsync(Guid id, CancellationToken ct = default);
    Task SaveAsync(Order order, CancellationToken ct = default);
}

public sealed class InMemoryOrderRepository : IOrderRepository
{
    private readonly Dictionary<Guid, Order> _db = [];
    public Task<Order?> GetByIdAsync(Guid id, CancellationToken ct = default)
        => Task.FromResult<Order?>(_db.TryGetValue(id, out var o) ? o : null);
    public Task SaveAsync(Order order, CancellationToken ct = default)
    {
        _db[order.Id] = order; // иммутабельный record — просто перезаписываем / overwrite is fine
        return Task.CompletedTask;
    }
}

// === OCP + LSP: политика скидок с защищённым контрактом ===
// === OCP + LSP: discount policy with a guarded contract ===
public interface IDiscountPolicy
{
    decimal Apply(Order order);
}

public abstract class DiscountBase : IDiscountPolicy
{
    public abstract decimal Apply(Order order);

    // LSP-контракт: результат всегда в [0, order.Total]
    // LSP contract: result is always within [0, order.Total]
    protected static decimal Enforce(Order order, decimal discounted) =>
        discounted < 0 || discounted > order.Total
            ? throw new InvalidOperationException(
                $"Скидка нарушает инвариант / Discount breaks invariant: {discounted}")
            : discounted;
}

public sealed class PercentDiscount(decimal percent) : DiscountBase
{
    public override decimal Apply(Order order) =>
        Enforce(order, order.Total * (1 - percent)); // 10% скидка / 10% off
}

public sealed class FixedDiscount(decimal amount) : DiscountBase
{
    public override decimal Apply(Order order) =>
        Enforce(order, Math.Max(0m, order.Total - amount));
}

public sealed class NoDiscount : DiscountBase
{
    public override decimal Apply(Order order) => Enforce(order, order.Total);
}

// OCP: новая скидка — новый класс, OrderProcessor не трогаем
// OCP: a new discount is a new class, OrderProcessor stays untouched
public sealed class LoyalCustomerDiscount(decimal extraPercent) : DiscountBase
{
    public override decimal Apply(Order order) =>
        Enforce(order, order.Total * (1 - extraPercent));
}

// === OCP: способы оплаты как Strategy ===
// === OCP: payment methods as Strategy ===
public interface IPaymentGateway
{
    Task ChargeAsync(decimal amount, Guid customerId, CancellationToken ct = default);
}

public sealed class CreditCardPayment(HttpClient http) : IPaymentGateway
{
    public Task ChargeAsync(decimal amount, Guid customerId, CancellationToken ct = default) =>
        http.PostAsJsonAsync("/charge", new { amount, customerId }, ct);
}

public interface ICryptoClient
{
    Task TransferAsync(decimal amount, Guid customerId, CancellationToken ct = default);
}

public sealed class CryptoPayment(ICryptoClient crypto) : IPaymentGateway
{
    public Task ChargeAsync(decimal amount, Guid customerId, CancellationToken ct = default) =>
        crypto.TransferAsync(amount, customerId, ct);
}

// === ISP: уведомление и инвойс — раздельные ролевые интерфейсы ===
// === ISP: notification and invoice are separate role interfaces ===
public interface INotifier
{
    Task NotifyAsync(Guid userId, string message, CancellationToken ct = default);
}

public interface IInvoiceGenerator
{
    Task<byte[]> GenerateAsync(Guid orderId, CancellationToken ct = default);
}

public sealed class EmailNotifier : INotifier
{
    public Task NotifyAsync(Guid userId, string message, CancellationToken ct = default) =>
        Task.CompletedTask; // заглушка для демо / demo stub
}

public sealed class PdfInvoiceGenerator : IInvoiceGenerator
{
    public Task<byte[]> GenerateAsync(Guid orderId, CancellationToken ct = default) =>
        Task.FromResult<byte[]>([]); // collection expression для пустого массива
}

// === SRP: оркестратор оформления заказа, ничего лишнего ===
// === SRP: order placement orchestrator, nothing else ===
public sealed class OrderProcessor(
    IOrderRepository repository,
    IDiscountPolicy discount,
    IPaymentGateway payment,
    INotifier notifier)
{
    public async Task<Guid> PlaceAsync(Order order, CancellationToken ct = default)
    {
        var final = discount.Apply(order);          // OCP: новая скидка = новый класс
        await payment.ChargeAsync(final, order.CustomerId, ct);   // DIP: абстракция
        var paid = order with { Status = OrderStatus.Paid, Total = final }; // record `with`
        await repository.SaveAsync(paid, ct);
        await notifier.NotifyAsync(order.CustomerId, $"Заказ {order.Id} оплачен / Order {order.Id} paid", ct);
        return paid.Id;
    }
}

// === Composition Root: единственное место, знающее о деталях ===
// === Composition Root: the only place aware of details ===
public static class CompositionRoot
{
    public static IServiceProvider Build()
    {
        var services = new ServiceCollection();
        services.AddHttpClient<CreditCardPayment>();
        services.AddSingleton<ICryptoClient, DummyCryptoClient>();
        services.AddScoped<IOrderRepository, InMemoryOrderRepository>();
        services.AddSingleton<IDiscountPolicy>(_ => new PercentDiscount(0.10m)); // смена — одной строкой
        services.AddScoped<IPaymentGateway, CreditCardPayment>();
        services.AddScoped<INotifier, EmailNotifier>();
        services.AddScoped<IInvoiceGenerator, PdfInvoiceGenerator>();
        services.AddScoped<OrderProcessor>();
        return services.BuildServiceProvider();
    }
}

internal sealed class DummyCryptoClient : ICryptoClient
{
    public Task TransferAsync(decimal amount, Guid customerId, CancellationToken ct = default) => Task.CompletedTask;
}

// Точка входа (top-level statements)
var sp = CompositionRoot.Build();
var processor = sp.GetRequiredService<OrderProcessor>();
var order = new Order(Guid.NewGuid(), Guid.NewGuid(), 100m, OrderStatus.New);
var id = await processor.PlaceAsync(order);
Console.WriteLine($"Оформлен заказ / Order placed: {id}");
```

Разбор по строкам. `Order` — `sealed record`, иммутабельный; свойство `with` позволяет создать копию с новым статусом без мутации — это best practice из урока. `IOrderRepository` — абстракция DIP: `OrderProcessor` не знает, SQL это или InMemory. `InMemoryOrderRepository` помечен `sealed` и использует collection expression `[]` для словаря — современный синтаксис C# 12. `IDiscountPolicy` + `DiscountBase` — связка OCP и LSP: новый способ скидки добавляется новым классом (`LoyalCustomerDiscount`), а контракт `Enforce` гарантирует, что любой подтип вернёт значение в `[0, order.Total]`. Если бы кто-то написал `ZeroDiscount`, бросающий для `Total > 0`, контрактный тест поймал бы нарушение LSP. `IPaymentGateway` с `CreditCardPayment` и `CryptoPayment` — классическая Strategy: смена способа оплаты — одна строка в Composition Root, `OrderProcessor` не правится. `INotifier` и `IInvoiceGenerator` разделены — это ISP: `OrderProcessor` использует только нотификатор и не зависит от метода генерации инвойса, который нужен другому сервису. `OrderProcessor` через первичный конструктор принимает четыре зависимости и не содержит ни одного `new` для внешних ресурсов — чистый SRP: он только оркестрирует оформление. Composition Root — единственное место, знающее о конкретных реализациях; `AddSingleton<IDiscountPolicy>(_ => new PercentDiscount(0.10m))` показывает, как передать параметр конструктора через фабрику. Тесты подставляют `FakeOrderRepository` и `FakeNotifier`, что доказывает DIP: `OrderProcessor` тестируется без БД и SMTP.

#### Задания на углубление (бонус)
1. Добавьте декоратор `LoggingOrderProcessor`, оборачивающий `OrderProcessor` и пишущий в лог до/после вызова `PlaceAsync`, не меняя сам `OrderProcessor`. Зарегистрируйте декоратор в Composition Root через `AddDecorator` или ручную фабрику.
2. Напишите «анти-тест»: намеренно нарушающий LSP класс `BrokenDiscount`, который для `Total > 100` бросает исключение, и покажите, что контрактный тест падает. Объясните, почему композиция здесь лучше наследования.
3. Реализуйте цепочку скидок (Chain of Responsibility или Composite `CompositeDiscount`), применяющую несколько скидок подряд, и проверьте, что итог всё ещё в `[0, Total]`.
4. Переведите регистрации Composition Root на Source Generator или Scrutor-сканирование сборки, чтобы новые реализации регистрировались автоматически по конвенции.

---

## Statement in English / Постановка на английском

#### Context & motivation
Imagine you joined the "Shardik-Market" startup, which sells digital courses. The current order-processing code is a single 600-line `OrderManager` class: it reads an order from a SQL database, computes a discount through a giant `switch` over customer types, charges money through a specific bank gateway, sends email through an `SmtpClient`, prints a PDF invoice, and also writes to a log file. Every change — a new Black Friday discount, a new crypto payment method, or a switch from email to Telegram — forces you to edit this class, break existing tests, and stress out the release engineers. The manager asks you to rewrite the module so that adding a new variant never requires editing working code, tests do not depend on a real database or SMTP, and classes stay compact and readable.

Your task is to design an `M16.SOLID.Homework` module on C# 12 / .NET 8 in which all five SOLID principles are applied deliberately. You do not rewrite the old code blindly; you build a new architecture next to it and show how each principle solves a specific pain of the startup. In parallel you write unit tests with fake repository and notifier implementations — this is the very check for DIP: if tests need a real database or SMTP, the principle is broken. In the finale you wire everything in a Composition Root through `Microsoft.Extensions.DependencyInjection` and demonstrate that switching the payment method or the discount is a single registration line.

#### What to do step by step
1. Create a solution and a console project: `dotnet new sln -n ShardikMarket`, then `dotnet new console -n M16.SOLID.Homework -o src/M16.SOLID.Homework --framework net8.0`, add it to the solution with `dotnet sln add src/M16.SOLID.Homework`. Enable nullable context and LangVersion 12 in the `.csproj`: `<Nullable>enable</Nullable>` and `<LangVersion>12</LangVersion>`.
2. Add the DI package: `dotnet add src/M16.SOLID.Homework package Microsoft.Extensions.DependencyInjection` and `dotnet add src/M16.SOLID.Homework package Microsoft.Extensions.Hosting`. Create a test project: `dotnet new xunit -n M16.SOLID.Homework.Tests -o tests/M16.SOLID.Homework.Tests`, `dotnet sln add tests/M16.SOLID.Homework.Tests`, `dotnet add tests/M16.SOLID.Homework.Tests reference src/M16.SOLID.Homework`.
3. Define the domain model in `Domain.cs`: `public enum OrderStatus { New, Paid, Shipped, Cancelled }` and `public sealed record Order(Guid Id, Guid CustomerId, decimal Total, OrderStatus Status);`. Using `record` and `sealed` is the lesson's best practice for compact immutable values.
4. Implement the DIP repository: interface `IOrderRepository { Task<Order?> GetByIdAsync(Guid, CancellationToken); Task SaveAsync(Order, CancellationToken); }` and an `InMemoryOrderRepository` class backed by a `Dictionary<Guid, Order>`. The repository must be `sealed`.
5. Implement OCP + LSP for discounts: interface `IDiscountPolicy { decimal Apply(Order); }`, an abstract `DiscountBase` with a protected `Enforce(Order, decimal)` method that throws `InvalidOperationException` when the discount is negative or exceeds `order.Total` — this is the LSP contract. Add three subtypes: `PercentDiscount(decimal percent)`, `FixedDiscount(decimal amount)`, `NoDiscount`. All are `sealed` and use C# 12 primary constructors.
6. Implement OCP for payments: `IPaymentGateway { Task ChargeAsync(decimal, Guid, CancellationToken); }` with two implementations — `CreditCardPayment` and `CryptoPayment`. Let `CreditCardPayment` take an `HttpClient` through a primary constructor, and `CryptoPayment` take an `ICryptoClient`.
7. Implement ISP: split notifications into `INotifier { Task NotifyAsync(Guid, string, CancellationToken); }` and `IInvoiceGenerator { Task<byte[]> GenerateAsync(Guid orderId, CancellationToken); }`. Do not create a single fat `INotificationService` with both methods. Implement `EmailNotifier : INotifier` and `PdfInvoiceGenerator : IInvoiceGenerator`.
8. Implement the SRP service `OrderProcessor` with a primary constructor taking `IOrderRepository`, `IDiscountPolicy`, `IPaymentGateway`, `INotifier`. The `PlaceAsync` method computes the discounted total, charges the payment, saves the order, and sends a notification. No logging, PDF, or SQL logic inside.
9. Wire the Composition Root in `Program.cs` through `Host.CreateDefaultBuilder` and `IServiceCollection`: register `InMemoryOrderRepository` as `Scoped`, the discount as `Singleton<IDiscountPolicy, PercentDiscount>` (pass the parameter through a factory), the payment, the notifier, and `OrderProcessor`. Print the placed order's ID to the console.
10. Write unit tests in `OrderProcessorTests.cs`: use fakes `FakeOrderRepository` and `FakeNotifier`, assert that after `PlaceAsync` the order is saved with `Status == Paid`, that `ChargeAsync` was called, and that the notification was called. Write `DiscountTests.cs`: for `PercentDiscount(0.1m)` and an order of 100 the total is 90; for `NoDiscount` the total equals `Total`. Write an LSP contract test: assert that all discount subtypes for an `Order` with `Total = 100` return a value within `[0, 100]`.
11. Run `dotnet build` and `dotnet test` — everything must compile and pass. Run `dotnet run --project src/M16.SOLID.Homework` — the console must print the GUID of a new order.
12. Run the OCP experiment: add a new `LoyalCustomerDiscount` class in a separate file without touching `OrderProcessor` or existing discounts. Register it with a single line in the Composition Root and confirm that all tests stay green.

#### Requirements
- Target framework `net8.0`, language C# 12, file-scoped namespaces, nullable context enabled. Use primary constructors, `sealed`, `record`, collection expressions, and pattern matching where they fit.
- All implementation classes are `sealed`, except the abstract `DiscountBase`, which serves as the contract. Interfaces are small and role-based — no more than two or three methods each.
- `OrderProcessor` creates no `new` for the repository, payment, discount, or notifier — all dependencies arrive through the constructor. Inside `OrderProcessor` there is no mention of SQL, SMTP, PDF, `HttpClient`, or log files.
- Discounts are implemented through the `IDiscountPolicy` interface and the abstract `DiscountBase` with the guarded `Enforce` contract. Any subtype that breaks the `[0, order.Total]` invariant throws — this is the LSP guard.
- The Composition Root is the only place aware of concrete implementations. Switching the discount from `PercentDiscount` to `LoyalCustomerDiscount` is a single registration line.
- Unit tests never touch a real database, network, or file system. All external dependencies are replaced by fakes. If a test needs an `HttpClient`, use `IHttpClientFactory` or a fake gateway.
- The code compiles without error-level warnings, `dotnet test` is green, and `dotnet run` prints the order GUID.

#### Pitfalls
- Do not confuse SRP's "one reason to change" with "one method per class." A class may hold several methods if they all belong to one responsibility. `OrderProcessor.PlaceAsync` orchestrates but does not implement the details — that is fine.
- The main OCP trap is the temptation to add a `switch` over customer type inside `OrderProcessor`. If you see `switch (order.CustomerType)`, that is a signal for polymorphism or Strategy. A new discount must be a new class, not a new `switch` branch.
- LSP trap: a subtype that throws `NotSupportedException` for an `Order` with `Total = 0` looks harmless but breaks the contract if the base class promises "always returns a number." Write a contract test that enumerates all `IDiscountPolicy` implementations through reflection or an explicit list.
- ISP is often violated "for the future": a developer drops `GenerateInvoiceAsync` into `INotifier` because "we may need it later." Do not. If `OrderProcessor` does not print invoices, it does not need that method. Make a separate `IInvoiceGenerator` used by another service.
- DIP trap: registering `OrderProcessor` as `Singleton` while `IOrderRepository` is `Scoped` produces a captured dependency — the singleton captures the first scoped instance for its whole lifetime. Register `OrderProcessor` as `Scoped` or `Transient` when its dependencies are scoped.
- Do not over-engineer: do not build an `IOrderProcessorFactory` plus an `OrderProcessorFactoryProvider` for trivial logic. SOLID is not dogma, as the lesson reminds us. Apply it where there is genuine complexity and a real risk of change.
- Make sure `CancellationToken` is threaded through every async method — this is part of the contract and a best practice that simplifies cancellation testing.
- Test fakes are stub classes, not necessarily a mocking library. A simple `FakeNotifier : INotifier` with a list of calls is often more readable than `Mock<INotifier>`.

#### Acceptance criteria
- [ ] The solution builds with `dotnet build` without errors on .NET 8 / C# 12.
- [ ] `dotnet test` passes all tests with no skips.
- [ ] `dotnet run` prints the placed order's GUID to the console.
- [ ] `OrderProcessor` takes all dependencies through a primary constructor and contains no `new` for external resources.
- [ ] Discounts are implemented through `IDiscountPolicy` + `DiscountBase` with the `Enforce` contract.
- [ ] Adding `LoyalCustomerDiscount` requires no edits to `OrderProcessor` or existing discounts.
- [ ] `INotifier` and `IInvoiceGenerator` are separate interfaces; no client depends on extra methods.
- [ ] All implementation classes are `sealed`, except `DiscountBase`.
- [ ] `IOrderRepository` is implemented by `InMemoryOrderRepository`; tests use a fake or this class without a real DB.
- [ ] The Composition Root is the only place aware of concrete implementations.
- [ ] A contract test checks the `[0, order.Total]` invariant for every discount.
- [ ] `CancellationToken` is threaded through every async method with a default value.
- [ ] The domain model is a `sealed record`, immutable.
- [ ] Nullable context is enabled; there are no nullability warnings.
- [ ] There is no "God object" and no "god" interface; no interface has more than three methods.
- [ ] A README or comment states which principle is applied where (SRP/OCP/LSP/ISP/DIP).

#### Hints (no direct answer)
- For SRP, ask: "How many reasons does this class have to change?" More than one means split it.
- For OCP, ask: "What do I edit when a new payment method appears?" If you edit `OrderProcessor`, OCP is violated.
- For LSP, write a parameterized test taking `Func<IDiscountPolicy>` and checking the contract for every factory.
- For ISP, ask: "Which method of this interface does the client not call?" If any, split it.
- For DIP, ask: "Can I test `OrderProcessor` without a database and network?" If not, DIP is violated.
- Use `AddSingleton<IDiscountPolicy>(_ => new PercentDiscount(0.1m))` or `ActivatorUtilities.CreateInstance` to pass a constructor parameter.

#### Reference solution walk-through

```csharp
// C# 12 / .NET 8 — M16.SOLID.Homework
// Comments: EN (+ RU)
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;

namespace M16.SOLID.Homework;

// === Domain model: immutable record + enum ===
public enum OrderStatus { New, Paid, Shipped, Cancelled }
public sealed record Order(Guid Id, Guid CustomerId, decimal Total, OrderStatus Status);

// === DIP: repository as an abstraction, InMemory is a detail ===
public interface IOrderRepository
{
    Task<Order?> GetByIdAsync(Guid id, CancellationToken ct = default);
    Task SaveAsync(Order order, CancellationToken ct = default);
}

public sealed class InMemoryOrderRepository : IOrderRepository
{
    private readonly Dictionary<Guid, Order> _db = [];
    public Task<Order?> GetByIdAsync(Guid id, CancellationToken ct = default)
        => Task.FromResult<Order?>(_db.TryGetValue(id, out var o) ? o : null);
    public Task SaveAsync(Order order, CancellationToken ct = default)
    {
        _db[order.Id] = order; // immutable record — overwrite is safe
        return Task.CompletedTask;
    }
}

// === OCP + LSP: discount policy with a guarded contract ===
public interface IDiscountPolicy
{
    decimal Apply(Order order);
}

public abstract class DiscountBase : IDiscountPolicy
{
    public abstract decimal Apply(Order order);

    // LSP contract: result is always within [0, order.Total]
    protected static decimal Enforce(Order order, decimal discounted) =>
        discounted < 0 || discounted > order.Total
            ? throw new InvalidOperationException(
                $"Discount breaks invariant: {discounted}")
            : discounted;
}

public sealed class PercentDiscount(decimal percent) : DiscountBase
{
    public override decimal Apply(Order order) =>
        Enforce(order, order.Total * (1 - percent)); // 10% off
}

public sealed class FixedDiscount(decimal amount) : DiscountBase
{
    public override decimal Apply(Order order) =>
        Enforce(order, Math.Max(0m, order.Total - amount));
}

public sealed class NoDiscount : DiscountBase
{
    public override decimal Apply(Order order) => Enforce(order, order.Total);
}

// OCP: a new discount is a new class, OrderProcessor stays untouched
public sealed class LoyalCustomerDiscount(decimal extraPercent) : DiscountBase
{
    public override decimal Apply(Order order) =>
        Enforce(order, order.Total * (1 - extraPercent));
}

// === OCP: payment methods as Strategy ===
public interface IPaymentGateway
{
    Task ChargeAsync(decimal amount, Guid customerId, CancellationToken ct = default);
}

public sealed class CreditCardPayment(HttpClient http) : IPaymentGateway
{
    public Task ChargeAsync(decimal amount, Guid customerId, CancellationToken ct = default) =>
        http.PostAsJsonAsync("/charge", new { amount, customerId }, ct);
}

public interface ICryptoClient
{
    Task TransferAsync(decimal amount, Guid customerId, CancellationToken ct = default);
}

public sealed class CryptoPayment(ICryptoClient crypto) : IPaymentGateway
{
    public Task ChargeAsync(decimal amount, Guid customerId, CancellationToken ct = default) =>
        crypto.TransferAsync(amount, customerId, ct);
}

// === ISP: notification and invoice are separate role interfaces ===
public interface INotifier
{
    Task NotifyAsync(Guid userId, string message, CancellationToken ct = default);
}

public interface IInvoiceGenerator
{
    Task<byte[]> GenerateAsync(Guid orderId, CancellationToken ct = default);
}

public sealed class EmailNotifier : INotifier
{
    public Task NotifyAsync(Guid userId, string message, CancellationToken ct = default) =>
        Task.CompletedTask; // demo stub
}

public sealed class PdfInvoiceGenerator : IInvoiceGenerator
{
    public Task<byte[]> GenerateAsync(Guid orderId, CancellationToken ct = default) =>
        Task.FromResult<byte[]>([]); // collection expression for an empty array
}

// === SRP: order placement orchestrator, nothing else ===
public sealed class OrderProcessor(
    IOrderRepository repository,
    IDiscountPolicy discount,
    IPaymentGateway payment,
    INotifier notifier)
{
    public async Task<Guid> PlaceAsync(Order order, CancellationToken ct = default)
    {
        var final = discount.Apply(order);          // OCP: new discount = new class
        await payment.ChargeAsync(final, order.CustomerId, ct);   // DIP: abstraction
        var paid = order with { Status = OrderStatus.Paid, Total = final }; // record `with`
        await repository.SaveAsync(paid, ct);
        await notifier.NotifyAsync(order.CustomerId, $"Order {order.Id} paid", ct);
        return paid.Id;
    }
}

// === Composition Root: the only place aware of details ===
public static class CompositionRoot
{
    public static IServiceProvider Build()
    {
        var services = new ServiceCollection();
        services.AddHttpClient<CreditCardPayment>();
        services.AddSingleton<ICryptoClient, DummyCryptoClient>();
        services.AddScoped<IOrderRepository, InMemoryOrderRepository>();
        services.AddSingleton<IDiscountPolicy>(_ => new PercentDiscount(0.10m)); // swap in one line
        services.AddScoped<IPaymentGateway, CreditCardPayment>();
        services.AddScoped<INotifier, EmailNotifier>();
        services.AddScoped<IInvoiceGenerator, PdfInvoiceGenerator>();
        services.AddScoped<OrderProcessor>();
        return services.BuildServiceProvider();
    }
}

internal sealed class DummyCryptoClient : ICryptoClient
{
    public Task TransferAsync(decimal amount, Guid customerId, CancellationToken ct = default) => Task.CompletedTask;
}

// Entry point (top-level statements)
var sp = CompositionRoot.Build();
var processor = sp.GetRequiredService<OrderProcessor>();
var order = new Order(Guid.NewGuid(), Guid.NewGuid(), 100m, OrderStatus.New);
var id = await processor.PlaceAsync(order);
Console.WriteLine($"Order placed: {id}");
```

Line-by-line walk-through. `Order` is a `sealed record`, immutable; the `with` expression lets us produce a copy with a new status without mutation — a lesson best practice. `IOrderRepository` is the DIP abstraction: `OrderProcessor` does not know whether it is SQL or in-memory. `InMemoryOrderRepository` is `sealed` and uses the `[]` collection expression for the dictionary — modern C# 12 syntax. `IDiscountPolicy` plus `DiscountBase` is the OCP + LSP pair: a new discount is a new class (`LoyalCustomerDiscount`), and the `Enforce` contract guarantees every subtype returns a value within `[0, order.Total]`. If someone wrote a `ZeroDiscount` that threw for `Total > 0`, the contract test would catch the LSP violation. `IPaymentGateway` with `CreditCardPayment` and `CryptoPayment` is a classic Strategy: swapping the payment method is one line in the Composition Root, and `OrderProcessor` is never edited. `INotifier` and `IInvoiceGenerator` are split — that is ISP: `OrderProcessor` uses only the notifier and does not depend on the invoice method that another service needs. `OrderProcessor` takes four dependencies through a primary constructor and contains no `new` for external resources — pure SRP: it only orchestrates order placement. The Composition Root is the only place aware of concrete implementations; `AddSingleton<IDiscountPolicy>(_ => new PercentDiscount(0.10m))` shows how to pass a constructor parameter through a factory. Tests inject `FakeOrderRepository` and `FakeNotifier`, which proves DIP: `OrderProcessor` is testable without a database or SMTP.

#### Going deeper (bonus)
1. Add a `LoggingOrderProcessor` decorator wrapping `OrderProcessor` and logging before/after `PlaceAsync` without modifying `OrderProcessor`. Register the decorator in the Composition Root via `AddDecorator` or a manual factory.
2. Write an "anti-test": an intentionally LSP-violating `BrokenDiscount` that throws for `Total > 100`, and show that the contract test fails. Explain why composition is better than inheritance here.
3. Implement a discount chain (Chain of Responsibility or a `CompositeDiscount`) applying several discounts in sequence, and verify the final value still lies within `[0, Total]`.
4. Move Composition Root registrations to a Source Generator or Scrutor assembly scan so new implementations are registered automatically by convention.

---

#### Чек-лист сдачи / Submission checklist
- [ ] Решение собирается `dotnet build` и проходит `dotnet test` (RU).
- [ ] `dotnet run` выводит GUID заказа (RU).
- [ ] Все пять принципов применены осознанно и помечены в коде или README (RU).
- [ ] Юнит-тесты используют фейки, а не реальные БД/SMTP (RU).
- [ ] Добавлена бонусная скидка `LoyalCustomerDiscount` без правок `OrderProcessor` (RU).
- [ ] The solution builds with `dotnet build` and passes `dotnet test` (EN).
- [ ] `dotnet run` prints the order GUID (EN).
- [ ] All five principles are applied deliberately and marked in code or README (EN).
- [ ] Unit tests use fakes rather than real DB/SMTP (EN).
- [ ] The bonus `LoyalCustomerDiscount` is added without editing `OrderProcessor` (EN).

#### Ресурсы / Resources
- Microsoft Learn — Dependency injection in .NET: https://learn.microsoft.com/dotnet/core/extensions/dependency-injection
- Microsoft Learn — Microservices DDD/CQRS patterns: https://learn.microsoft.com/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/
- Robert C. Martin, «Clean Architecture» (SOLID fundamentals).
- Source-making — SOLID principles: https://wiki.c2.com/?PrinciplesOfObjectOrientedDesign
- C# 12 features: https://learn.microsoft.com/dotnet/csharp/whats-new/csharp-12
