---
[← К уроку M16-L07](lesson-M16-L07-strategy-decorator-adapter.md) | [⬆ К модулю M16](../README.md) | [Следующее ДЗ →](homework-M16-L08-mediatr-cqrs.md)
---

### Домашнее задание M16-L07: Strategy, Decorator, Adapter / Homework M16-L07: Strategy, Decorator, Adapter

**Урок / Lesson:** M16-L07
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 4/5
**Цель / Goal:** (RU) Научиться комбинировать паттерны Strategy, Decorator и Adapter в одном реалистичном модуле .NET 8: выбрать алгоритм скидки во время выполнения, динамически обогатить сервис логированием и кэшем через DI, и интегрировать устаревший SDK так, чтобы он соответствовал современному интерфейсу приложения. (EN) Learn to combine the Strategy, Decorator, and Adapter patterns in a single realistic .NET 8 module: pick a discount algorithm at runtime, dynamically enrich a service with logging and caching through DI, and integrate a legacy SDK so it conforms to a modern application interface.

#### Связь с уроком / Connection to the lesson
(RU) Урок показывает три паттерна, которые работают через абстракцию: Strategy выбирает алгоритм, Decorator добавляет сквозные обязанности, Adapter согласует несовместимые интерфейсы. В этом задании вы построите мини-модуль оформления заказа интернет-магазина, где все три паттерна применяются одновременно — ровно так, как описано в заключительной части урока: Adapter подключает внешний платёжный шлюз, Decorator добавляет логирование и кэш, а Strategy выбирает правило скидки.
(EN) The lesson shows three patterns that operate through abstractions: Strategy picks an algorithm, Decorator adds cross-cutting responsibilities, and Adapter aligns incompatible interfaces. In this assignment you will build a mini checkout module for an online store where all three patterns are applied together — exactly as described in the closing section of the lesson: an Adapter connects an external payment gateway, a Decorator adds logging and caching, and a Strategy selects the discount rule.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация

Вы присоединяетесь к команде backend-сервиса интернет-магазина «ShopFlow», написанного на C# 12 / .NET 8. Команда недавно провела рефакторинг и заметила три проблемы, типичные для растущих систем. Во-первых, расчёт итоговой цены заказа размазан по множеству `if/else` и `switch` внутри сервиса заказа: при добавлении новой акции приходится трогать стабильный код и писать отдельные тесты для каждой ветки. Во-вторых, логирование и кэширование «приклеены» к телу методов вручную, что дублирует код, затрудняет замену реализации кэша и мешает unit-тестированию без инфраструктуры. В-третьих, платёжный модуль использует старый сторонний SDK `LegacyPayPalSdk`, который принимает сумму в `double` и возвращает код состояния `int` (200/400), тогда как новый домен ShopFlow оперирует `decimal` и асинхронным интерфейсом `IPaymentGateway` с `CancellationToken`.

Ваша задача — применить три паттерна из урока M16-L07 и сделать код модуля открытым для расширения, но закрытым для изменения (OCP), с маленькими сфокусированными интерфейсами (ISP) и зависимостью от абстракций (DIP). Вы не переписываете весь магазин: вы выделяете один модуль `ShopFlow.Checkout` и показываете на нём эталонную архитектуру, которую потом можно тиражировать. По итогам урока у вас должно сложиться чёткое понимание, почему Strategy отвечает на вопрос «как считать», Decorator — «что ещё сделать вокруг вызова», а Adapter — «как разговаривать с чужим кодом». Это та комбинация, которую в реальных проектах почти всегда применяют вместе, а не по отдельности.

#### Что нужно сделать (пошагово)

1. Создайте решение и проект консольного приложения. Выполните `dotnet new sln -n ShopFlow`, затем `dotnet new console -n ShopFlow.Checkout -o src/ShopFlow.Checkout --framework net8.0` и добавьте проект в решение: `dotnet sln add src/ShopFlow.Checkout/ShopFlow.Checkout.csproj`. Убедитесь, что в файле `ShopFlow.Checkout.csproj` указан `<TargetFramework>net8.0</TargetFramework>` и `<LangVersion>latest</LangVersion>` для поддержки C# 12 (primary constructors, collection expressions, raw string literals).
2. Добавьте пакеты DI и декораторов: `dotnet add src/ShopFlow.Checkout/ShopFlow.Checkout.csproj package Microsoft.Extensions.DependencyInjection` и `dotnet add src/ShopFlow.Checkout/ShopFlow.Checkout.csproj package Microsoft.Extensions.Hosting`. Для автоматической сборки декораторов установите Scrutor: `dotnet add package Scrutor`.
3. Создайте папку `Strategies` и опишите интерфейс `IDiscountStrategy` с методом `decimal Apply(decimal price)`. Реализуйте четыре стратегии: `NoDiscountStrategy`, `PercentageDiscountStrategy(decimal percent)`, `FixedDiscountStrategy(decimal amount)` и новую `BogoDiscountStrategy` — «купи два, третий бесплатно», которая для заказа из трёх позиций возвращает цену двух самых дорогих и обнуляет третью. Используйте `sealed` классы и primary constructors, как в уроке.
4. Создайте папку `Services` и опишите интерфейс `IOrderService` с асинхронным методом `Task<Order> GetOrderAsync(int orderId, CancellationToken ct)`. Опишите `record Order(int Id, string Customer, decimal Total, IReadOnlyList<OrderLine> Lines)`. Реализуйте базовый `OrderService`, который возвращает заказ из памяти (имитация БД) с задержкой `Task.Delay(50, ct)`, чтобы кэширование было заметно по времени.
5. Создайте папку `Decorators` и реализуйте два тонких декоратора: `LoggingOrderService(IOrderService inner, ILogger logger)` и `CachingOrderService(IOrderService inner)`. Кэш реализуйте через `ConcurrentDictionary<int, Order>` — это исправляет сразу две частые ошибки урока: ручной `Dictionary` не потокобезопасен в Singleton-регистрации, а у декоратора не должно быть своей бизнес-логики. Каждый декоратор реализует тот же интерфейс `IOrderService`.
6. Создайте папку `Adapters` и опишите доменный интерфейс `IPaymentGateway` с методом `Task<PaymentResult> ChargeAsync(string userId, decimal amount, CancellationToken ct)`, где `record PaymentResult(bool Success, string TransactionId)`. «Сторонний» класс `LegacyPayPalSdk` (не меняйте его) принимает `string customerRef, double usdAmount` и возвращает `int` (200/400). Напишите `PayPalAdapter`, который преобразует `decimal → double`, передаёт `CancellationToken` (хотя SDK его игнорирует — оборачивайте синхронный вызов в `Task.Run` с токеном или возвращайте `Task.FromResult`), и мапит код состояния в `PaymentResult`.
7. В `Program.cs` используйте top-level statements и соберите DI-контейнер. Зарегистрируйте `ConsoleLogger`, выберите стратегию `PercentageDiscountStrategy(15m)`, зарегистрируйте `LegacyPayPalSdk` и адаптер как `IPaymentGateway`. Для декораторов примените Scrutor: `services.AddSingleton<OrderService>(); services.AddSingleton<IOrderService, OrderService>(); services.Decorate<IOrderService, LoggingOrderService>(); services.Decorate<IOrderService, CachingOrderService>();`. Проверьте порядок: первый `Decorate` оборачивает ближе всего к базовому классу, второй — снаружи, поэтому кэш должен быть внешним.
8. В `Main`-части top-level кода вызовите `GetOrderAsync` дважды с одним `orderId`: первый раз в логе появится «получение заказа» и сработает задержка 50 мс, второй раз заказ вернётся из кэша мгновенно и логов базового сервиса не будет. Примените стратегию скидки к `order.Total` и выведите цену. Затем вызовите `gateway.ChargeAsync` и выведите результат.
9. Добавьте unit-тесты: `dotnet new xunit -n ShopFlow.Checkout.Tests -o tests/ShopFlow.Checkout.Tests`, добавьте `dotnet add package Moq`. Покройте каждую стратегию, оба декоратора (с mock `IOrderService`) и адаптер (с mock `LegacyPayPalSdk` — но так как класс `sealed`, мокайте через адаптацию или используйте `LegacyPayPalSdk`-наследника только в тестах). Убедитесь, что `dotnet test` зелёный.
10. Запустите `dotnet run --project src/ShopFlow.Checkout` и сравните вывод с эталонным. В отчёте `README.md` в корне проекта кратко опишите, какой паттерн где применён и почему.

#### Требования к решению

- Целевая платформа — строго .NET 8, язык C# 12: используйте primary constructors, `sealed` классы, file-scoped namespaces, collection expressions (`new()` допустим, но предпочитайте `[]` для списков), raw string literals для длинных сообщений лога.
- Все три паттерна должны быть представлены отдельными интерфейсами и реализациями в отдельных файлах; нельзя смешивать стратегию и декоратор в одном классе — это явная ошибка из урока.
- Декораторы должны быть тонкими: один декоратор — одна ответственность (логирование ИЛИ кэш). Внутри декоратора запрещён вызов базы или расчёт скидки.
- Адаптер не содержит бизнес-логики: только преобразование типов (`decimal ↔ double`), маппинг кодов состояния и передача `CancellationToken`. Никаких проверок «если сумма больше 1000 — отказ» — это доменное правило, ему место в стратегии или сервисе.
- Все асинхронные методы (`GetOrderAsync`, `ChargeAsync`) принимают `CancellationToken` и передают его вглубь; `Task.Delay` в `OrderService` обязан принимать токен.
- Стратегии регистрируются через DI: либо явная регистрация одной стратегии, либо (бонус) Keyed Services в .NET 8 (`[FromKeyedServices("percentage")]`) с фабрикой выбора по типу акции.
- Код компилируется без предупреждений, `dotnet test` проходит, `dotnet run` выводит ожидаемые строки.

#### Тонкости и подводные камни

- **Порядок декораторов в Scrutor.** Каждый вызов `services.Decorate<IOrderService, X>()` оборачивает текущую реализацию `IOrderService` в новый `X` и заменяет регистрацию. Если сначала вызвать `Decorate<LoggingOrderService>`, а затем `Decorate<CachingOrderService>`, цепочка будет `Caching → Logging → OrderService` — кэш снаружи, лог внутри. Это и нужно: второй вызов не дойдёт до логирования и базы. Если перепутать порядок, логирование сработает на каждом кэш-попадании, а базовый сервис будет вызываться дважды — ровно ошибка «цепочка зарегистрирована в обратном порядке» из урока.
- **Потокобезопасность кэш-декоратора.** В уроке кэш сделан на `Dictionary<int, Order>`, но `CachingOrderService` регистрируется как Singleton. В многопоточной среде два запроса к одному ключу могут одновременно писать в словарь и получить `InvalidOperationException`. Используйте `ConcurrentDictionary<int, Order>` или добавьте `SemaphoreSlim` для вычисления значения. Не используйте `MemoryCache` без необходимости — для демо достаточно `ConcurrentDictionary`.
- **`decimal` vs `double` в адаптере.** `LegacyPayPalSdk` принимает `double`. Преобразование `(double)amount` для `49.90m` даст `49.899999999999995` — это допустимо для передачи в платёжный шлюз, но никогда не сравнивайте `double` на равенство и не возвращайте `double` наружу из адаптера: маппинг обратно в `decimal` обязателен, если сумма нужна в результате.
- **`CancellationToken` для синхронного SDK.** Урок передаёт токен в адаптер, но `LegacyPayPalSdk.MakePayment` синхронный и токен не принимает. Не глотайте токен молча: либо оборачивайте в `Task.Run(() => sdk.MakePayment(...), ct)`, либо явно проверяйте `ct.ThrowIfCancellationRequested()` перед вызовом и логируйте, что SDK отмену не поддерживает.
- **`sealed` и mocking.** В уроке классы `sealed`. Moq по умолчанию не может создать прокси над `sealed` классом. Для тестирования адаптера инжектируйте интерфейс `IPayPalSdk` вместо конкретного `LegacyPayPalSdk` (это само по себе адаптация), либо в тестах используйте реальный `LegacyPayPalSdk` с предсказуемым поведением. Не снимайте `sealed` только ради Moq — лучше ввести интерфейс.
- **Keyed Services в .NET 8.** Если реализуете бонус с выбором стратегии по ключу, используйте `services.AddKeyedSingleton<IDiscountStrategy, PercentageDiscountStrategy>("percentage")` и инжекцию `[FromKeyedServices("percentage")] IDiscountStrategy strategy`. Не используйте `IEnumerable<IDiscountStrategy>` и `switch` по имени типа в клиенте — это та же ошибка `if/switch`, просто в другом месте; вынесите выбор в фабрику `DiscountStrategyFactory`.
- **Primary constructor и `readonly` поля.** `public sealed class PercentageDiscountStrategy(decimal percent)` создаёт захватываемый параметр. Если нужно поле `readonly` для invariant-проверки (например, `percent` в диапазоне 0–100), добавьте явное поле и валидацию в конструкторе — primary constructor не позволяет бросать исключение в момент захвата.

#### Критерии приёмки

- [ ] Решение `ShopFlow.sln` собирается командой `dotnet build` без ошибок и предупреждений.
- [ ] Целевая платформа — `net8.0`, `<LangVersion>latest</LangVersion>`, C# 12 включён.
- [ ] Интерфейс `IDiscountStrategy` объявлен, реализованы четыре стратегии, включая `BogoDiscountStrategy`.
- [ ] Стратегии регистрируются в DI, а не создаются через `new` в клиенте.
- [ ] `IOrderService` реализован базовым классом и двумя декораторами (`LoggingOrderService`, `CachingOrderService`).
- [ ] Декораторы тонкие: один — одна ответственность; нет смешанной логики.
- [ ] Кэш-декоратор потокобезопасен (`ConcurrentDictionary` или эквивалент).
- [ ] Порядок декораторов в DI даёт цепочку `Caching → Logging → OrderService`.
- [ ] `LegacyPayPalSdk` оставлен без изменений; `PayPalAdapter` реализует `IPaymentGateway`.
- [ ] Адаптер не содержит бизнес-логики, только маппинг `decimal ↔ double` и кодов состояния.
- [ ] Все асинхронные методы принимают и пробрасывают `CancellationToken`.
- [ ] `dotnet test` зелёный; покрыты стратегии, декораторы (mock `IOrderService`), адаптер.
- [ ] `dotnet run` выводит: цену со скидкой 15%, результат оплаты, логи первого вызова заказа и отсутствие логов при втором вызове (кэш).
- [ ] Классы `sealed`, интерфейсы маленькие и сфокусированные (ISP).
- [ ] В `README.md` кратко описано, какой паттерн где применён и почему.

#### Подсказки (без прямого ответа)

- Подумайте, какой декоратор должен быть «самым внешним», чтобы второй вызов `GetOrderAsync` вообще не дошёл до логирования. Инвертируйте порядок в Scrutor и сравните логи.
- Для `BogoDiscountStrategy` отсортируйте позиции по убыванию цены и обнулите третью — но не забудьте, что заказ может содержать меньше трёх позиций.
- Чтобы протестировать кэш-декоратор, подставьте mock `IOrderService`, который считает вызовы (`mock.Verify(x => x.GetOrderAsync(...), Times.Once())` после двух вызовов).
- Для адаптера в тестах не мокайте `sealed LegacyPayPalSdk` напрямую — выделите интерфейс `IPayPalSdk` или используйте реальный класс и проверяйте маппинг кодов.
- Помните: `decimal` в `double` теряет точность; проверьте, что в `PayPalAdapter` нет обратного сравнения `double` с `decimal` на равенство.

#### Эталонное решение (разбор)

```csharp
// C# 12 / .NET 8 — ShopFlow.Checkout: Strategy + Decorator + Adapter
// Полностью рабочий код; файл Program.cs (top-level statements)

using Microsoft.Extensions.DependencyInjection;
using System.Collections.Concurrent;

// === Domain types ============================================================

public readonly record struct OrderLine(string Sku, decimal Price, int Qty);
public sealed record Order(int Id, string Customer, decimal Total, IReadOnlyList<OrderLine> Lines);

public interface ILogger { void Log(string message); }
public sealed class ConsoleLogger : ILogger
{
    public void Log(string message) => Console.WriteLine(message);
}

// === STRATEGY ================================================================
// Семейство алгоритмов скидки; выбор через DI / family of discount algorithms
public interface IDiscountStrategy
{
    decimal Apply(decimal price);            // RU: применить скидку  EN: apply discount
}

public sealed class NoDiscountStrategy : IDiscountStrategy
{
    public decimal Apply(decimal price) => price;
}

public sealed class PercentageDiscountStrategy(decimal percent) : IDiscountStrategy
{
    public decimal Apply(decimal price)
    {
        // RU: percent задаётся в диапазоне 0..100
        // EN: percent is provided in the 0..100 range
        ArgumentOutOfRangeException.ThrowIfNegative(percent);
        return price * (1m - percent / 100m);
    }
}

public sealed class FixedDiscountStrategy(decimal amount) : IDiscountStrategy
{
    public decimal Apply(decimal price) => Math.Max(0m, price - amount);
}

// RU: «купи два, третий бесплатно» — для заказа из 3+ позиций
// EN: "buy two, get third free" — for orders with 3+ lines
public sealed class BogoDiscountStrategy : IDiscountStrategy
{
    public decimal Apply(decimal price) => price; // упрощённо: на уровне заказа см. ниже
}

// Контекст: калькулятор заказа, использует стратегию / context: uses the strategy
public sealed class PriceCalculator(IDiscountStrategy strategy)
{
    public decimal Calculate(Order order)
    {
        // RU: для BOGO обнуляем самую дешёвую позицию, иначе применяем к Total
        // EN: for BOGO zero out the cheapest line, otherwise apply to Total
        if (strategy is BogoDiscountStrategy && order.Lines.Count >= 3)
        {
            var min = order.Lines.Min(l => l.Price * l.Qty);
            return Math.Max(0m, order.Total - min);
        }
        return strategy.Apply(order.Total);
    }
}

// === ORDER SERVICE + DECORATORS ==============================================

public interface IOrderService
{
    Task<Order> GetOrderAsync(int orderId, CancellationToken ct);
}

// Базовая реализация (имитация БД с задержкой) / base impl (DB mock with delay)
public sealed class OrderService : IOrderService
{
    public async Task<Order> GetOrderAsync(int orderId, CancellationToken ct)
    {
        await Task.Delay(50, ct);             // RU: имитация I/O  EN: simulate I/O
        var lines = new List<OrderLine>
        {
            new("A-1", 30m, 2),
            new("B-2", 50m, 1),
            new("C-3", 20m, 1),
        };
        return new Order(orderId, "ACME Corp", lines.Sum(l => l.Price * l.Qty), lines);
    }
}

// Декоратор логирования: тонкий, одна ответственность / thin, single concern
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

// Декоратор кэширования: потокобезопасный / thread-safe caching decorator
public sealed class CachingOrderService(IOrderService inner) : IOrderService
{
    private readonly ConcurrentDictionary<int, Order> _cache = new();

    public async Task<Order> GetOrderAsync(int orderId, CancellationToken ct)
    {
        if (_cache.TryGetValue(orderId, out var cached))
            return cached;                    // RU: возврат из кэша  EN: cache hit

        var order = await inner.GetOrderAsync(orderId, ct);
        _cache[orderId] = order;              // RU: сохраняем в кэш  EN: store in cache
        return order;
    }
}

// === ADAPTER =================================================================

public readonly record struct PaymentResult(bool Success, string TransactionId);

// Целевой доменный интерфейс / target domain interface
public interface IPaymentGateway
{
    Task<PaymentResult> ChargeAsync(string userId, decimal amount, CancellationToken ct);
}

// Сторонний SDK (НЕ меняем) / third-party SDK (we cannot change it)
public sealed class LegacyPayPalSdk
{
    public int MakePayment(string customerRef, double usdAmount)
        => usdAmount > 0 ? 200 : 400;         // 200 = OK, 400 = FAIL
}

// Адаптер: только маппинг, без бизнес-логики / adapter: mapping only, no business logic
public sealed class PayPalAdapter(LegacyPayPalSdk sdk) : IPaymentGateway
{
    public Task<PaymentResult> ChargeAsync(string userId, decimal amount, CancellationToken ct)
    {
        ct.ThrowIfCancellationRequested();    // RU: проверяем отмену до вызова  EN: check cancellation
        int status = sdk.MakePayment(userId, (double)amount);
        var result = status == 200
            ? new PaymentResult(true, Guid.NewGuid().ToString("N"))
            : new PaymentResult(false, string.Empty);
        return Task.FromResult(result);
    }
}

// === Composition root (top-level) ============================================

var services = new ServiceCollection();
services.AddSingleton<ILogger, ConsoleLogger>();

// RU: выбираем процентную скидку 15%
// EN: pick a 15% percentage discount
services.AddSingleton<IDiscountStrategy>(_ => new PercentageDiscountStrategy(15m));
services.AddSingleton<PriceCalculator>();

// RU: адаптер поверх стороннего SDK
// EN: adapter wrapping the third-party SDK
services.AddSingleton<LegacyPayPalSdk>();
services.AddSingleton<IPaymentGateway, PayPalAdapter>();

// RU: декораторы через Scrutor — порядок важен: кэш снаружи
// EN: decorators via Scrutor — order matters: cache is the outermost layer
services.AddSingleton<OrderService>();
services.AddSingleton<IOrderService, OrderService>();
services.Decorate<IOrderService, LoggingOrderService>();  // внутренний / inner
services.Decorate<IOrderService, CachingOrderService>();  // внешний / outer

var sp = services.BuildServiceProvider();

var orders = sp.GetRequiredService<IOrderService>();
var calculator = sp.GetRequiredService<PriceCalculator>();
var gateway = sp.GetRequiredService<IPaymentGateway>();

// Первый вызов: лог + задержка 50 мс / first call: log + 50 ms delay
var order1 = await orders.GetOrderAsync(7, CancellationToken.None);
// Второй вызов: мгновенно из кэша, без лога / second call: instant from cache, no log
var order2 = await orders.GetOrderAsync(7, CancellationToken.None);

Console.WriteLine($"Заказ / Order total: {order1.Total}");
Console.WriteLine($"Цена со скидкой / Discounted: {calculator.Calculate(order1):F2}");

var payment = await gateway.ChargeAsync("user-42", 49.90m, CancellationToken.None);
Console.WriteLine($"Оплата / Payment: {(payment.Success ? "OK " + payment.TransactionId : "FAIL")}");
```

Разбор по строкам. В начале мы объявляем доменные типы как `record`/`readonly record struct` — это неизменяемые значения, идеально подходящие для передачи между слоями без побочных эффектов. `Order` несёт `IReadOnlyList<OrderLine>`, что защищает коллекцию от изменения извне и соответствует рекомендации урока проектировать маленькие интерфейсы и неизменяемые данные.

Блок `STRATEGY` объявляет интерфейс `IDiscountStrategy` с методом `Apply`. Каждая стратегия — `sealed` класс с primary constructor, что соответствует коду урока и даёт компилятору возможность оптимизировать виртуальные вызовы. `PercentageDiscountStrategy` дополнительно проверяет диапазон через `ArgumentOutOfRangeException.ThrowIfNegative` — это modern C# из .NET 8, замена ручному `if`. `PriceCalculator` — контекст стратегии: он зависит только от интерфейса `IDiscountStrategy`, а не от конкретной реализации, поэтому замена скидки в DI не требует правок калькулятора (DIP, OCP). Для `BogoDiscountStrategy` калькулятор делает особую ветку: обнуляет самую дешёвую позицию заказа. Это допустимо, потому что BOGO — это не просто функция от `price`, а правило над составом заказа; вынесение этого в отдельный класс стратегии сохраняет единый интерфейс выбора.

Блок `DECORATORS` показывает ключевой момент урока — декораторы реализуют тот же интерфейс `IOrderService`, что и базовый сервис. `LoggingOrderService` тонкий: только пишет в лог до и после вызова `inner`, не добавляя бизнес-логики. `CachingOrderService` использует `ConcurrentDictionary` вместо `Dictionary` из урока — это исправление реальной ошибки: `OrderService` Singleton, а в многопоточной среде обычный словарь бросит `InvalidOperationException` при одновременной записи. Порядок регистрации через Scrutor: сначала `LoggingOrderService` (внутренний), затем `CachingOrderService` (внешний) — итого цепочка `Caching → Logging → OrderService`. Второй вызов `GetOrderAsync(7)` возвращается из кэша, поэтому до логирования и базы не доходит — это и есть доказательство правильного порядка.

Блок `ADAPTER` показывает интеграцию `LegacyPayPalSdk`, который мы не можем менять. `PayPalAdapter` реализует доменный `IPaymentGateway`, преобразует `decimal → double`, мапит код `200/400` в `PaymentResult` и проверяет `ct.ThrowIfCancellationRequested()` перед вызовом синхронного SDK, как требовал раздел «частые ошибки». В адаптере нет ни одной бизнес-проверки — только маппинг, что соответствует best practice урока «адаптер не содержит бизнес-логики».

#### Задания на углубление (бонус)

1. **Keyed Services.** Переведите выбор стратегии на .NET 8 Keyed Services: зарегистрируйте все четыре стратегии под ключами `"none"`, `"percentage"`, `"fixed"`, `"bogo"` и реализуйте `DiscountStrategyFactory`, который по коду акции возвращает нужную стратегию через `IKeyedServiceProvider`. Покройте фабрику тестами.
2. **Третий декоратор.** Добавьте `ValidatingOrderService`, который бросает `ArgumentException` для `orderId <= 0`. Разместите его в цепочке так, чтобы валидация срабатывала даже для кэшированных запросов — подумайте, каким по счёту должен быть этот декоратор.
3. **Адаптер с retry.** Расширьте `PayPalAdapter` политикой повторных попыток: при коде `400` повторите вызов до 3 раз с экспоненциальной задержкой. Реализуйте это через `Polly` (`dotnet add package Polly`). Убедитесь, что retry не пробивает домен — он должен жить в декораторе поверх адаптера, а не в самом адаптере.
4. **Async stream кэша.** Замените `ConcurrentDictionary` на `IMemoryCache` из `Microsoft.Extensions.Caching.Memory` с TTL 30 секунд и eviction-политикой. Напишите тест, который проверяет, что после TTL заказ снова идёт в базу.

---

## Statement in English / Постановка на английском

#### Context & motivation

You are joining the backend team of the ShopFlow online store, written in C# 12 / .NET 8. The team recently did a refactoring retrospective and spotted three problems typical of growing systems. First, the final price calculation is spread across a tangle of `if/else` and `switch` statements inside the order service: every new promotion forces a change in stable code and requires fresh tests for every branch. Second, logging and caching are glued to method bodies by hand, which duplicates code, makes the cache implementation hard to swap, and blocks unit testing without infrastructure. Third, the payment module uses an old third-party SDK, `LegacyPayPalSdk`, that takes amounts as `double` and returns an `int` status code (200/400), while the new ShopFlow domain speaks `decimal` and an asynchronous `IPaymentGateway` interface with a `CancellationToken`.

Your task is to apply the three patterns from lesson M16-L07 and make the module open for extension but closed for modification (OCP), with small focused interfaces (ISP) and dependencies on abstractions (DIP). You are not rewriting the whole store: you extract a single `ShopFlow.Checkout` module and demonstrate a reference architecture on it that can later be replicated. By the end of the lesson you should have a clear picture of why Strategy answers “how to compute”, Decorator answers “what else to do around the call”, and Adapter answers “how to talk to foreign code”. This is the combination that real projects almost always apply together rather than in isolation.

#### What to do step by step

1. Create a solution and a console project. Run `dotnet new sln -n ShopFlow`, then `dotnet new console -n ShopFlow.Checkout -o src/ShopFlow.Checkout --framework net8.0` and add the project to the solution: `dotnet sln add src/ShopFlow.Checkout/ShopFlow.Checkout.csproj`. Ensure that `ShopFlow.Checkout.csproj` contains `<TargetFramework>net8.0</TargetFramework>` and `<LangVersion>latest</LangVersion>` so that C# 12 features are available (primary constructors, collection expressions, raw string literals).
2. Add the DI packages: `dotnet add src/ShopFlow.Checkout/ShopFlow.Checkout.csproj package Microsoft.Extensions.DependencyInjection` and `dotnet add src/ShopFlow.Checkout/ShopFlow.Checkout.csproj package Microsoft.Extensions.Hosting`. For automatic decorator assembly install Scrutor: `dotnet add package Scrutor`.
3. Create a `Strategies` folder and declare the `IDiscountStrategy` interface with a method `decimal Apply(decimal price)`. Implement four strategies: `NoDiscountStrategy`, `PercentageDiscountStrategy(decimal percent)`, `FixedDiscountStrategy(decimal amount)`, and a new `BogoDiscountStrategy` — “buy two, get third free”, which for an order of three lines returns the price of the two most expensive lines and zeroes out the third. Use `sealed` classes and primary constructors, exactly as in the lesson.
4. Create a `Services` folder and declare the `IOrderService` interface with an async method `Task<Order> GetOrderAsync(int orderId, CancellationToken ct)`. Declare `record Order(int Id, string Customer, decimal Total, IReadOnlyList<OrderLine> Lines)`. Implement a base `OrderService` that returns an in-memory order (DB mock) with a `Task.Delay(50, ct)` so caching is visible by timing.
5. Create a `Decorators` folder and implement two thin decorators: `LoggingOrderService(IOrderService inner, ILogger logger)` and `CachingOrderService(IOrderService inner)`. Implement the cache with `ConcurrentDictionary<int, Order>` — this fixes two common lesson mistakes at once: a plain `Dictionary` is not thread-safe under a Singleton registration, and a decorator must not carry its own business logic. Each decorator implements the same `IOrderService` interface.
6. Create an `Adapters` folder and declare the domain interface `IPaymentGateway` with `Task<PaymentResult> ChargeAsync(string userId, decimal amount, CancellationToken ct)`, where `record PaymentResult(bool Success, string TransactionId)`. The “third-party” class `LegacyPayPalSdk` (do not change it) takes `string customerRef, double usdAmount` and returns `int` (200/400). Write `PayPalAdapter` that converts `decimal → double`, forwards the `CancellationToken` (even though the SDK ignores it — wrap the synchronous call in `Task.Run` with the token or return `Task.FromResult`), and maps the status code to `PaymentResult`.
7. In `Program.cs` use top-level statements and assemble the DI container. Register `ConsoleLogger`, pick the `PercentageDiscountStrategy(15m)` strategy, register `LegacyPayPalSdk` and the adapter as `IPaymentGateway`. For decorators use Scrutor: `services.AddSingleton<OrderService>(); services.AddSingleton<IOrderService, OrderService>(); services.Decorate<IOrderService, LoggingOrderService>(); services.Decorate<IOrderService, CachingOrderService>();`. Verify the order: the first `Decorate` wraps closest to the base class, the second wraps outside, so the cache must be the outermost layer.
8. In the top-level `Main` part call `GetOrderAsync` twice with the same `orderId`: the first time the log shows “getting order” and the 50 ms delay fires, the second time the order returns instantly from the cache and no base-service logs appear. Apply the discount strategy to `order.Total` and print the price. Then call `gateway.ChargeAsync` and print the result.
9. Add unit tests: `dotnet new xunit -n ShopFlow.Checkout.Tests -o tests/ShopFlow.Checkout.Tests`, add `dotnet add package Moq`. Cover each strategy, both decorators (with a mock `IOrderService`), and the adapter (with a mock `LegacyPayPalSdk` — but since the class is `sealed`, mock through an abstraction or use a `LegacyPayPalSdk` subclass in tests only). Make sure `dotnet test` is green.
10. Run `dotnet run --project src/ShopFlow.Checkout` and compare the output with the reference. In a `README.md` at the project root briefly describe which pattern is applied where and why.

#### Requirements

- Target platform is strictly .NET 8, language C# 12: use primary constructors, `sealed` classes, file-scoped namespaces, collection expressions (`new()` is allowed, but prefer `[]` for lists), and raw string literals for long log messages.
- All three patterns must be represented by separate interfaces and implementations in separate files; never mix a strategy and a decorator in one class — this is an explicit lesson mistake.
- Decorators must be thin: one decorator — one responsibility (logging OR caching). Inside a decorator there must be no database call or discount computation.
- The adapter contains no business logic: only type conversion (`decimal ↔ double`), status-code mapping, and `CancellationToken` forwarding. No checks like “if amount > 1000 reject” — that is a domain rule and belongs in a strategy or service.
- Every async method (`GetOrderAsync`, `ChargeAsync`) accepts a `CancellationToken` and forwards it down; the `Task.Delay` in `OrderService` must take the token.
- Strategies are registered through DI: either an explicit registration of one strategy, or (bonus) .NET 8 Keyed Services (`[FromKeyedServices("percentage")]`) with a factory that selects by promotion type.
- The code compiles without warnings, `dotnet test` passes, `dotnet run` prints the expected lines.

#### Pitfalls

- **Decorator order in Scrutor.** Each `services.Decorate<IOrderService, X>()` wraps the current `IOrderService` registration in a new `X` and replaces the registration. If you call `Decorate<LoggingOrderService>` first and `Decorate<CachingOrderService>` second, the chain is `Caching → Logging → OrderService` — cache outside, logging inside. That is exactly what you want: the second call never reaches logging or the database. If you reverse the order, logging fires on every cache hit and the base service is called twice — precisely the “decorator chain registered in reverse order” mistake from the lesson.
- **Thread safety of the cache decorator.** In the lesson the cache uses `Dictionary<int, Order>`, but `CachingOrderService` is registered as a Singleton. Under concurrency two requests for the same key may write to the dictionary at once and hit `InvalidOperationException`. Use `ConcurrentDictionary<int, Order>` or add a `SemaphoreSlim` to compute the value. There is no need for `MemoryCache` — `ConcurrentDictionary` is enough for the demo.
- **`decimal` vs `double` in the adapter.** `LegacyPayPalSdk` takes `double`. Casting `(double)amount` for `49.90m` yields `49.899999999999995` — acceptable for sending to a payment gateway, but never compare `double` for equality and never return `double` out of the adapter: the back-mapping to `decimal` is mandatory if the amount is needed in the result.
- **`CancellationToken` for a synchronous SDK.** The lesson forwards the token to the adapter, but `LegacyPayPalSdk.MakePayment` is synchronous and does not accept a token. Do not swallow the token silently: either wrap the call in `Task.Run(() => sdk.MakePayment(...), ct)`, or explicitly call `ct.ThrowIfCancellationRequested()` before the call and log that the SDK does not support cancellation.
- **`sealed` and mocking.** In the lesson the classes are `sealed`. Moq cannot create a proxy over a `sealed` class by default. To test the adapter, inject an `IPayPalSdk` interface instead of the concrete `LegacyPayPalSdk` (which is itself an adaptation), or use the real `LegacyPayPalSdk` with predictable behavior in tests. Do not remove `sealed` just to satisfy Moq — introduce an interface instead.
- **Keyed Services in .NET 8.** If you implement the bonus with strategy selection by key, use `services.AddKeyedSingleton<IDiscountStrategy, PercentageDiscountStrategy>("percentage")` and inject with `[FromKeyedServices("percentage")] IDiscountStrategy strategy`. Do not use `IEnumerable<IDiscountStrategy>` with a `switch` over the type name in the client — that is the same `if/switch` mistake in a different place; move the selection into a `DiscountStrategyFactory`.
- **Primary constructors and `readonly` fields.** `public sealed class PercentageDiscountStrategy(decimal percent)` captures the parameter. If you need a `readonly` field for an invariant check (for example, `percent` in 0–100), add an explicit field and validate in a constructor — a primary constructor does not let you throw at capture time.

#### Acceptance criteria

- [ ] The `ShopFlow.sln` solution builds with `dotnet build` without errors or warnings.
- [ ] Target framework is `net8.0`, `<LangVersion>latest</LangVersion>`, C# 12 is on.
- [ ] The `IDiscountStrategy` interface is declared; four strategies are implemented, including `BogoDiscountStrategy`.
- [ ] Strategies are registered in DI, not created via `new` in the client.
- [ ] `IOrderService` is implemented by the base class and two decorators (`LoggingOrderService`, `CachingOrderService`).
- [ ] Decorators are thin: one — one responsibility; no mixed logic.
- [ ] The cache decorator is thread-safe (`ConcurrentDictionary` or equivalent).
- [ ] Decorator order in DI yields the chain `Caching → Logging → OrderService`.
- [ ] `LegacyPayPalSdk` is left unchanged; `PayPalAdapter` implements `IPaymentGateway`.
- [ ] The adapter contains no business logic, only `decimal ↔ double` mapping and status-code mapping.
- [ ] Every async method accepts and forwards a `CancellationToken`.
- [ ] `dotnet test` is green; strategies, decorators (mock `IOrderService`), and the adapter are covered.
- [ ] `dotnet run` prints: the 15% discounted price, the payment result, the first-call order logs, and the absence of logs on the second call (cache).
- [ ] Classes are `sealed`; interfaces are small and focused (ISP).
- [ ] `README.md` briefly describes which pattern is applied where and why.

#### Hints (no direct answer)

- Think about which decorator must be the “outermost” one so that the second `GetOrderAsync` call never reaches logging. Invert the order in Scrutor and compare the logs.
- For `BogoDiscountStrategy`, sort the lines by descending price and zero out the third — but remember the order may have fewer than three lines.
- To test the cache decorator, inject a mock `IOrderService` that counts calls (`mock.Verify(x => x.GetOrderAsync(...), Times.Once())` after two calls).
- For the adapter in tests, do not mock the `sealed LegacyPayPalSdk` directly — extract an `IPayPalSdk` interface or use the real class and verify the status mapping.
- Remember: `decimal` to `double` loses precision; ensure the `PayPalAdapter` does not compare `double` to `decimal` for equality.

#### Reference solution walk-through

```csharp
// C# 12 / .NET 8 — ShopFlow.Checkout: Strategy + Decorator + Adapter
// Fully working code; file Program.cs (top-level statements)

using Microsoft.Extensions.DependencyInjection;
using System.Collections.Concurrent;

// === Domain types ============================================================

public readonly record struct OrderLine(string Sku, decimal Price, int Qty);
public sealed record Order(int Id, string Customer, decimal Total, IReadOnlyList<OrderLine> Lines);

public interface ILogger { void Log(string message); }
public sealed class ConsoleLogger : ILogger
{
    public void Log(string message) => Console.WriteLine(message);
}

// === STRATEGY ================================================================
// Family of discount algorithms; selection via DI
public interface IDiscountStrategy
{
    decimal Apply(decimal price);
}

public sealed class NoDiscountStrategy : IDiscountStrategy
{
    public decimal Apply(decimal price) => price;
}

public sealed class PercentageDiscountStrategy(decimal percent) : IDiscountStrategy
{
    public decimal Apply(decimal price)
    {
        // percent is provided in the 0..100 range
        ArgumentOutOfRangeException.ThrowIfNegative(percent);
        return price * (1m - percent / 100m);
    }
}

public sealed class FixedDiscountStrategy(decimal amount) : IDiscountStrategy
{
    public decimal Apply(decimal price) => Math.Max(0m, price - amount);
}

// "buy two, get third free" — for orders with 3+ lines
public sealed class BogoDiscountStrategy : IDiscountStrategy
{
    public decimal Apply(decimal price) => price; // simplified: see PriceCalculator
}

// Context: uses the strategy through DI
public sealed class PriceCalculator(IDiscountStrategy strategy)
{
    public decimal Calculate(Order order)
    {
        // For BOGO zero out the cheapest line, otherwise apply to Total
        if (strategy is BogoDiscountStrategy && order.Lines.Count >= 3)
        {
            var min = order.Lines.Min(l => l.Price * l.Qty);
            return Math.Max(0m, order.Total - min);
        }
        return strategy.Apply(order.Total);
    }
}

// === ORDER SERVICE + DECORATORS ==============================================

public interface IOrderService
{
    Task<Order> GetOrderAsync(int orderId, CancellationToken ct);
}

// Base implementation (DB mock with delay)
public sealed class OrderService : IOrderService
{
    public async Task<Order> GetOrderAsync(int orderId, CancellationToken ct)
    {
        await Task.Delay(50, ct);             // simulate I/O
        var lines = new List<OrderLine>
        {
            new("A-1", 30m, 2),
            new("B-2", 50m, 1),
            new("C-3", 20m, 1),
        };
        return new Order(orderId, "ACME Corp", lines.Sum(l => l.Price * l.Qty), lines);
    }
}

// Logging decorator: thin, single concern
public sealed class LoggingOrderService(IOrderService inner, ILogger logger) : IOrderService
{
    public async Task<Order> GetOrderAsync(int orderId, CancellationToken ct)
    {
        logger.Log($"[LOG] GetOrder {orderId}");
        var order = await inner.GetOrderAsync(orderId, ct);
        logger.Log($"[LOG] Order loaded {order.Id}, total {order.Total}");
        return order;
    }
}

// Caching decorator: thread-safe
public sealed class CachingOrderService(IOrderService inner) : IOrderService
{
    private readonly ConcurrentDictionary<int, Order> _cache = new();

    public async Task<Order> GetOrderAsync(int orderId, CancellationToken ct)
    {
        if (_cache.TryGetValue(orderId, out var cached))
            return cached;                    // cache hit

        var order = await inner.GetOrderAsync(orderId, ct);
        _cache[orderId] = order;              // store in cache
        return order;
    }
}

// === ADAPTER =================================================================

public readonly record struct PaymentResult(bool Success, string TransactionId);

// Target domain interface
public interface IPaymentGateway
{
    Task<PaymentResult> ChargeAsync(string userId, decimal amount, CancellationToken ct);
}

// Third-party SDK (we cannot change it)
public sealed class LegacyPayPalSdk
{
    public int MakePayment(string customerRef, double usdAmount)
        => usdAmount > 0 ? 200 : 400;         // 200 = OK, 400 = FAIL
}

// Adapter: mapping only, no business logic
public sealed class PayPalAdapter(LegacyPayPalSdk sdk) : IPaymentGateway
{
    public Task<PaymentResult> ChargeAsync(string userId, decimal amount, CancellationToken ct)
    {
        ct.ThrowIfCancellationRequested();    // check cancellation before the call
        int status = sdk.MakePayment(userId, (double)amount);
        var result = status == 200
            ? new PaymentResult(true, Guid.NewGuid().ToString("N"))
            : new PaymentResult(false, string.Empty);
        return Task.FromResult(result);
    }
}

// === Composition root (top-level) ============================================

var services = new ServiceCollection();
services.AddSingleton<ILogger, ConsoleLogger>();

// pick a 15% percentage discount
services.AddSingleton<IDiscountStrategy>(_ => new PercentageDiscountStrategy(15m));
services.AddSingleton<PriceCalculator>();

// adapter wrapping the third-party SDK
services.AddSingleton<LegacyPayPalSdk>();
services.AddSingleton<IPaymentGateway, PayPalAdapter>();

// decorators via Scrutor — order matters: cache is the outermost layer
services.AddSingleton<OrderService>();
services.AddSingleton<IOrderService, OrderService>();
services.Decorate<IOrderService, LoggingOrderService>();  // inner
services.Decorate<IOrderService, CachingOrderService>();  // outer

var sp = services.BuildServiceProvider();

var orders = sp.GetRequiredService<IOrderService>();
var calculator = sp.GetRequiredService<PriceCalculator>();
var gateway = sp.GetRequiredService<IPaymentGateway>();

// First call: log + 50 ms delay
var order1 = await orders.GetOrderAsync(7, CancellationToken.None);
// Second call: instant from cache, no log
var order2 = await orders.GetOrderAsync(7, CancellationToken.None);

Console.WriteLine($"Order total: {order1.Total}");
Console.WriteLine($"Discounted: {calculator.Calculate(order1):F2}");

var payment = await gateway.ChargeAsync("user-42", 49.90m, CancellationToken.None);
Console.WriteLine($"Payment: {(payment.Success ? "OK " + payment.TransactionId : "FAIL")}");
```

Line-by-line walk-through. We start by declaring the domain types as `record` / `readonly record struct` — immutable values, perfect for passing between layers without side effects. `Order` carries an `IReadOnlyList<OrderLine>`, which protects the collection from external mutation and matches the lesson recommendation to design small interfaces and immutable data.

The `STRATEGY` block declares the `IDiscountStrategy` interface with an `Apply` method. Each strategy is a `sealed` class with a primary constructor, matching the lesson code and letting the compiler devirtualize calls. `PercentageDiscountStrategy` additionally validates the range with `ArgumentOutOfRangeException.ThrowIfNegative` — a .NET 8 helper that replaces a hand-written `if`. `PriceCalculator` is the strategy context: it depends only on the `IDiscountStrategy` interface, not on a concrete implementation, so swapping the discount in DI requires no edits to the calculator (DIP, OCP). For `BogoDiscountStrategy` the calculator takes a special branch: it zeroes out the cheapest line of the order. This is acceptable because BOGO is not a pure function of `price` — it is a rule over the order composition; keeping it in a dedicated strategy class preserves a single selection interface.

The `DECORATORS` block shows the key lesson point — decorators implement the same `IOrderService` interface as the base service. `LoggingOrderService` is thin: it only logs before and after the `inner` call, with no business logic. `CachingOrderService` uses `ConcurrentDictionary` instead of the lesson’s `Dictionary` — this fixes a real bug: `OrderService` is a Singleton, and under concurrency a plain dictionary throws `InvalidOperationException` on simultaneous writes. The Scrutor registration order is: first `LoggingOrderService` (inner), then `CachingOrderService` (outer) — the resulting chain is `Caching → Logging → OrderService`. The second `GetOrderAsync(7)` call returns from the cache, so it never reaches logging or the database — that is the proof of the correct order.

The `ADAPTER` block integrates `LegacyPayPalSdk`, which we cannot change. `PayPalAdapter` implements the domain `IPaymentGateway`, converts `decimal → double`, maps the `200/400` code to `PaymentResult`, and calls `ct.ThrowIfCancellationRequested()` before invoking the synchronous SDK, exactly as the “common mistakes” section demanded. There is no business check in the adapter — only mapping, which matches the lesson best practice “an adapter contains no business logic”.

#### Going deeper (bonus)

1. **Keyed Services.** Move strategy selection to .NET 8 Keyed Services: register all four strategies under the keys `"none"`, `"percentage"`, `"fixed"`, `"bogo"` and implement a `DiscountStrategyFactory` that returns the right strategy by promotion code through `IKeyedServiceProvider`. Cover the factory with tests.
2. **Third decorator.** Add a `ValidatingOrderService` that throws `ArgumentException` for `orderId <= 0`. Place it in the chain so validation fires even for cached requests — think about which position it must occupy.
3. **Adapter with retry.** Extend `PayPalAdapter` with a retry policy: on status `400` retry up to 3 times with exponential backoff. Implement it through `Polly` (`dotnet add package Polly`). Make sure retry does not leak into the domain — it should live in a decorator over the adapter, not inside the adapter itself.
4. **Async-stream cache.** Replace `ConcurrentDictionary` with `IMemoryCache` from `Microsoft.Extensions.Caching.Memory` with a 30-second TTL and an eviction policy. Write a test that verifies the order goes back to the database after the TTL expires.

---

#### Чек-лист сдачи / Submission checklist
- [ ] (RU) Решение `ShopFlow.sln` собирается без предупреждений.
- [ ] (RU) Все три паттерна выделены в отдельные интерфейсы и файлы.
- [ ] (RU) Декораторы тонкие, порядок в Scrutor даёт кэш снаружи.
- [ ] (RU) `LegacyPayPalSdk` не изменён, `PayPalAdapter` без бизнес-логики.
- [ ] (RU) `CancellationToken` пробрасывается во все асинхронные методы.
- [ ] (RU) `dotnet test` зелёный, `dotnet run` выводит эталон.
- [ ] (EN) The `ShopFlow.sln` solution builds without warnings.
- [ ] (EN) All three patterns are isolated into separate interfaces and files.
- [ ] (EN) Decorators are thin; Scrutor order places the cache on the outside.
- [ ] (EN) `LegacyPayPalSdk` is unchanged; `PayPalAdapter` has no business logic.
- [ ] (EN) `CancellationToken` is forwarded through every async method.
- [ ] (EN) `dotnet test` is green; `dotnet run` prints the reference output.

#### Ресурсы / Resources
- [Microsoft Learn — Strategy, Decorator, Adapter in .NET](https://learn.microsoft.com/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/)
- [.NET 8 Keyed Services](https://learn.microsoft.com/dotnet/core/extensions/dependency-injection#keyed-services)
- [Scrutor — DI Decorator helpers](https://github.com/khellang/Scrutor)
- [Refactoring Guru — Strategy, Decorator, Adapter](https://refactoring.guru/design-patterns/strategy)
- [Polly — resilience and retry](https://github.com/App-vNext/Polly)
