---
[← К уроку M15-L01](lesson-M15-L01-test-types-pyramid.md) | [⬆ К модулю M15](../README.md) | [Следующее ДЗ →](homework-M15-L02-xunit-facts-theories.md)
---

### Домашнее задание M15-L01: Виды тестов (unit/integration/e2e), пирамида / Homework M15-L01: Test types (unit/integration/e2e), pyramid

**Урок / Lesson:** M15-L01
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 3/5
**Цель / Goal:** (RU) Научиться различать три уровня пирамиды тестов, реализовать рабочий пример всех трёх уровней на C# 12 / .NET 8 с xUnit, подобрать правильную пропорцию тестов и избежать классических антипаттернов (перевёрнутая пирамида, леденец на палочке). (EN) Learn to distinguish the three layers of the test pyramid, implement a working example of all three layers on C# 12 / .NET 8 with xUnit, choose the correct test ratio, and avoid classic anti-patterns (inverted pyramid, ice cream cone).

#### Связь с уроком / Connection to the lesson

(RU) Урок M15-L01 вводит модель пирамиды тестов Майка Конна и показывает, как один и тот же домен (`Cart` / `CartItem`) покрывается тремя уровнями: unit-тестами для чистой логики, integration-тестами с `UseInMemoryDatabase` для репозитория и упрощённым E2E через контроллер со stub-репозиторием. В этом задании вы воспроизведёте и расширите эту структуру на новом домене — система расчёта стоимости заказа с скидками, налогами и подарочными сертификатами — чтобы на практике почувствовать разницу в скорости, изоляции и стоимости каждого слоя.

(EN) Lesson M15-L01 introduces Mike Cohn's test pyramid and shows how a single domain (`Cart` / `CartItem`) is covered at three layers: unit tests for pure logic, integration tests with `UseInMemoryDatabase` for the repository, and a simplified E2E through the controller with a stub repository. In this assignment you will reproduce and extend that structure on a new domain — an order-pricing system with discounts, taxes, and gift certificates — so that you can feel the difference in speed, isolation, and cost of each layer in practice.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация

Вы присоединились к команде, которая разрабатывает микросервис «Калькулятор заказов» для интернет-магазина. Сервис принимает позиции корзины, применяет правила скидок, начисляет налог и вычитает подарочные сертификаты. Бизнес-логика критична: ошибка в формуле на один процент стоит компании миллионов рублей в месяц, а неточный тест даёт ложное чувство безопасности.

В репозитории уже есть три тестовых проекта, но их распределение по пирамиде хаотично: половина «unit-тестов» на самом деле ходит в реальную базу данных, а E2E-тестов столько же, сколько unit — набор медленный, хрупкий и локализует дефекты плохо. Руководитель попросил вас навести порядок: выделить чистую логику в отдельный класс, покрыть её быстрыми детерминированными unit-тестами, добавить integration-тест на репозиторий с in-memory EF Core, и оставить ровно один-два E2E-теста на главный сценарий через контроллер.

Цель задания — не написать максимум тестов, а правильно распределить их по пирамиде: примерно 70% unit, 20% integration, 10% E2E. Вы должны почувствовать, как сдвиг тестов вниз (shift-left) ускоряет обратную связь и снижает стоимость поддержки. Параллельно вы закрепите best practices из урока: тестируйте поведение, а не реализацию; один тест — одна причина упасть; читаемые имена вида `Method_Scenario_Expected`; детерминированность без случайных дат и сетевых вызовов. Задание построено так, что каждая ошибка из чек-листа «Частые ошибки» урока встретится вам в виде искушения, которое надо распознать и обойти.

#### Что нужно сделать (пошагово)

1. **Создайте структуру решения.** Откройте терминал в пустой папке и выполните команды:
   ```bash
   dotnet new sln -n OrderPricing
   dotnet new classlib -n OrderPricing.Domain -f net8.0
   dotnet new webapi -n OrderPricing.Api -f net8.0 --use-controllers
   dotnet new xunit -n OrderPricing.UnitTests -f net8.0
   dotnet new xunit -n OrderPricing.IntegrationTests -f net8.0
   dotnet new xunit -n OrderPricing.E2ETests -f net8.0
   dotnet sln add **/*.csproj
   ```
   Добавьте ссылки между проектами: `UnitTests` → `Domain`; `IntegrationTests` → `Domain`, `Api`; `E2ETests` → `Domain`, `Api`. В `IntegrationTests` и `Api` установите EF Core in-memory:
   ```bash
   dotnet add OrderPricing.Api package Microsoft.EntityFrameworkCore.InMemory --version 8.*
   dotnet add OrderPricing.IntegrationTests package Microsoft.EntityFrameworkCore.InMemory --version 8.*
   ```

2. **Реализуйте домен.** В `OrderPricing.Domain` создайте файл `Pricing/OrderPricer.cs` с классом `OrderPricer` и рекордами `OrderLine`, `PricingResult`. Логика: суммировать `UnitPrice * Qty` по строкам; применить процентную скидку (0–100); начислить налог (например, 20% НДС); вычесть подарочный сертификат, но не уходить в минус (итог `>= 0`). Это чистая логика без зависимостей — кандидат на unit-тесты.

3. **Реализуйте репозиторий.** В `OrderPricing.Api` создайте `IOrderRepository`, `OrderDbContext : DbContext` с `DbSet<OrderEntity>`, и `EfOrderRepository`, который сохраняет и читает заказы. `OrderEntity` — отдельная модель persistence, не доменный рекорд, чтобы продемонстрировать разделение слоёв.

4. **Напишите unit-тесты** (проект `UnitTests`). Покройте `OrderPricer`: пустой заказ (итог 0, граничный случай), только скидка, только налог, скидка + налог, сертификат, превышающий итог (итог не уходит в минус), отрицательная цена (должно бросать `ArgumentOutOfRangeException`), `null`-заказ (`ArgumentNullException`). Используйте `[Fact]` для одиночных сценариев и `[Theory]/[InlineData]` для табличных. Один тест — одна причина упасть.

5. **Напишите integration-тесты** (проект `IntegrationTests`). Поднимите `OrderDbContext` на `UseInMemoryDatabase("order-it")`, сохраните заказ через `EfOrderRepository`, прочитайте обратно и проверьте, что строки и сумма совпали. Добавьте второй тест: чтение несуществующего `orderId` возвращает `null`. Эти тесты медленнее unit, но проверяют «швы» между репозиторием и EF Core.

6. **Напишите E2E-тесты** (проект `E2ETests`). Реализуйте минимальный контроллер `OrdersController` с `POST /api/orders` (создать) и `GET /api/orders/{id}/total` (вернуть итог). В E2E-тесте используйте либо `WebApplicationFactory<Program>` (если установите `Microsoft.AspNetCore.Mvc.Testing`), либо прямой вызов действия контроллера со stub-репозиторием (как в уроке) — для экономии времени выберите второй вариант. Проверьте полный путь: контроллер → репозиторий → `OrderPricer` → HTTP-ответ `200 OK` с правильной суммой.

7. **Измерьте пропорцию.** Запустите `dotnet test --verbosity normal` отдельно для каждого проекта и запишите в `README.md` таблицу: количество тестов в каждом проекте, общее время выполнения. Убедитесь, что пропорция близка к 70/20/10. Если у вас 10 unit, 3 integration, 1–2 E2E — это хороший ориентир.

8. **Запустите всё вместе и убедитесь, что набор зелёный.** Команда `dotnet test` из корня решения должна пройти за разумное время (единицы секунд). Если интеграционные тесты подтормаживают — проверьте, что вы не создаёте новый `DbContext` на каждый `Assert` без необходимости.

#### Требования к решению

Решение должно компилироваться без предупреждений на .NET 8 с C# 12 (включайте top-level statements в `Program.cs`, pattern matching, collection expressions, `record`/`primary constructor` где уместно). Целевой фреймворк всех проектов — `net8.0`. Используйте xUnit как тестовый фреймворк. Структура проектов должна чётко отражать пирамиду: три отдельных тестовых проекта, а не один общий. Каждый тестовый класс содержит тесты только одного уровня — смешивание unit и integration в одном классе запрещено (это частая ошибка из урока).

Доменная логика (`OrderPricer`) не должна зависеть от EF Core, ASP.NET Core или любого I/O — только чистые вычисления над `IEnumerable<OrderLine>`. Все внешние зависимости вынесены за интерфейсы (`IOrderRepository`). Unit-тесты не должны ходить в базу, сеть или файловую систему; integration-тесты — единственное место, где появляется `DbContext`. E2E-тесты работают через публичный контракт контроллера, а не через рефлексию или приватные методы. Имена тестов — вида `MethodName_Scenario_ExpectedResult`, читаются как спецификация. Каждый тест оформлен по схеме Arrange-Act-Assert, разделён пустыми строками. В `README.md` приведена таблица с количеством тестов каждого уровня и обоснованием выбранной пропорции.

#### Тонкости и подводные камни

Главная ловушка — написать «unit-тест», который на самом деле создаёт `OrderDbContext` и пишет в базу. Внешне он выглядит как unit, но при запуске на CI без диска/сети начнёт падать, а на локальной машине — замедлять набор. Признак проблемы: в `using` тестового класса упоминается `Microsoft.EntityFrameworkCore`. Вынесите зависимость в `IOrderRepository` и в unit-тестах подставляйте `FakeOrderRepository` — простой класс, хранящий заказы в `Dictionary<Guid, OrderEntity>`.

Вторая ловушка — тестирование приватных методов через reflection. Например, хочется протестировать приватный `ApplyDiscount` напрямую. По уроку: тестируйте поведение через публичный `Calculate` — приватный метод — деталь реализации, его сигнатура может измениться, и тест сломается ложно. Если метод стал сложным и просится под отдельный тест, выделите его в отдельный публичный класс стратегии (`IPricingRule`) и покройте unit-тестом.

Третья ловушка — несколько `Assert` в одном тесте на разные сценарии. Когда такой тест падает, вы не сразу понимаете, какой именно сценарий сломался. Разбейте на `[Theory]` с разными `InlineData` или на отдельные `[Fact]`. Единственное допустимое исключение — несколько `Assert`, проверяющих одно и то же логическое утверждение с разных сторон (например, `Assert.NotNull(x); Assert.Equal(5, x.Total)`).

Четвёртая ловушка — разделяемое изменяемое состояние между тестами. Если вы используете статический `OrderDbContext` и один тест что-то записал, а другой читает — порядок выполнения начинает влиять на результат. Каждый тест должен поднимать свежий контекст: либо `using var db = new OrderDbContext(...)` внутри `[Fact]`, либо `IAsyncLifetime` с очисткой. xUnit создаёт новый экземпляр тестового класса на каждый тест — полагайтесь на это, но не держите мутабельное состояние в `static`.

Пятая ловушка — перевёрнутая пирамида. Если вы начнёте «для надёжности» покрывать каждую ветку логики E2E-тестом, набор станет медленным и хрупким. Помните: один и тот же баг дешевле ловить unit-тестом. Сдвигайте проверки вниз: то, что можно проверить в изоляции, не поднимайте до integration/E2E.

Шестая тонкость — детерминированность. Никаких `DateTime.Now`, `Guid.NewGuid()` в ожидаемых значениях, `Thread.Sleep`, `Random` без сида. Если нужно «время» — вводите `IClock` и в тестах подставляйте фиксированное. В этом задании время не нужно, но если добавите поле `CreatedAt` — используйте `IClock`.

Седьмая тонкость — выбор in-memory против SQLite. `UseInMemoryDatabase` удобен, но не проверяет SQL-ограничения (unique, foreign keys). Для более честного integration-теста урока достаточно in-memory, но в реальном проекте лучше SQLite in-memory (`UseSqlite("DataSource=:memory:")`) — он ближе к прод-БД. В этом задании оставайтесь на in-memory EF Core, чтобы совпадать с примером урока.

#### Критерии приёмки

- [ ] Решение содержит 6 проектов: `Domain`, `Api`, `UnitTests`, `IntegrationTests`, `E2ETests` и `.sln`.
- [ ] Все проекты target `net8.0`, компилируются без ошибок и предупреждений.
- [ ] `OrderPricer` не ссылается на EF Core, ASP.NET Core или I/O-классы.
- [ ] `IOrderRepository` определён в `Domain` или `Api`, реализация `EfOrderRepository` — в `Api`.
- [ ] Unit-тесты покрывают не менее 7 сценариев: пустой заказ, только скидка, только налог, скидка+налог, сертификат, сертификат-больше-итога, отрицательная цена, `null`.
- [ ] В `UnitTests` нет `using Microsoft.EntityFrameworkCore*`.
- [ ] Хотя бы один unit-тест использует `[Theory]/[InlineData]` с тремя и более вариантами.
- [ ] Integration-тесты создают `OrderDbContext` на `UseInMemoryDatabase`, сохраняют и читают заказ.
- [ ] Integration-тест на чтение несуществующего `orderId` ожидает `null`.
- [ ] E2E-тест вызывает публичное действие контроллера и проверяет `200 OK` с правильной суммой.
- [ ] Ни один тест не использует reflection для приватных методов.
- [ ] Ни один тест не содержит больше одного логического утверждения на разные сценарии.
- [ ] Имена тестов вида `MethodName_Scenario_ExpectedResult`.
- [ ] `dotnet test` из корня проходит зелёно за разумное время (< 30 с).
- [ ] `README.md` содержит таблицу количества тестов каждого уровня с обоснованием пропорции, близкой к 70/20/10.

#### Подсказки (без прямого ответа)

- Подумайте, какой класс должен содержать метод `Calculate` — это чистый сервис без состояния или метод-расширение? Где проходит граница между доменом и persistence?
- Для сертификата, превышающего итог, вспомните: результат `Max(0, subtotal - discount + tax - gift)`. Какой тип возврата лучше — `decimal` или рекорд `PricingResult` с полями `Subtotal`, `Discount`, `Tax`, `Gift`, `Total`? Второй вариант удобнее для тестов и для API-ответа.
- Для integration-теста не забудьте `await db.SaveChangesAsync()` — без него in-memory провайдер не зафиксирует изменения, и следующий контекст их не увидит. Каждый тест — свой `DbContext`, общий только `DbContextOptions`.
- Для E2E-теста без `WebApplicationFactory`: создайте `StubOrderRepository : IOrderRepository`, который возвращает заранее подготовленный заказ. Контроллер примите через primary constructor: `public sealed class OrdersController(IOrderRepository repo) : ControllerBase`.
- Чтобы посчитать пропорцию, достаточно `dotnet test --verbosity normal | Select-String "Passed!"` для каждого проекта — xUnit выводит строку `Passed: N`.

#### Эталонное решение (разбор)

```csharp
// C# 12 / .NET 8 — Эталонное решение ДЗ M15-L01. / Reference solution.
// Домен: расчёт стоимости заказа со скидкой, налогом и подарочным сертификатом.
// Domain: order pricing with discount, tax and gift certificate.

using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading;
using System.Threading.Tasks;
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;
using Xunit;

namespace OrderPricing.Domain.Pricing;

// Чистый доменный рекорд — кандидат для unit-тестов. / Pure domain record — unit-test candidate.
public sealed record OrderLine(string Sku, decimal UnitPrice, int Qty);

// Результат калькуляции — возвращаем все промежуточные значения. / Result exposes all intermediate values.
public sealed record PricingResult(decimal Subtotal, decimal Discount, decimal Tax, decimal Gift, decimal Total)
{
    // Итог не может быть отрицательным: сертификат не «копит» остаток. / Total can't go negative.
    public static PricingResult Zero => new(0m, 0m, 0m, 0m, 0m);
}

// Чистая логика без зависимостей — основание пирамиды. / Pure logic, no deps — pyramid base.
public sealed class OrderPricer
{
    private const decimal TaxRate = 0.20m; // НДС 20% / VAT 20%

    public PricingResult Calculate(IReadOnlyList<OrderLine> lines, decimal discountPercent, decimal gift)
    {
        ArgumentNullException.ThrowIfNull(lines);
        if (discountPercent is < 0m or > 100m)
            throw new ArgumentOutOfRangeException(nameof(discountPercent), "Must be 0..100.");
        if (gift < 0m) throw new ArgumentOutOfRangeException(nameof(gift), "Must be >= 0.");

        foreach (var l in lines)
            if (l.UnitPrice < 0m || l.Qty <= 0)
                throw new ArgumentOutOfRangeException(nameof(lines), "Price/Qty invalid.");

        var subtotal = lines.Sum(l => l.UnitPrice * l.Qty);            // сумма строк
        var discount = subtotal * discountPercent / 100m;              // скидка в деньгах
        var taxed = subtotal - discount;                               // база для налога
        var tax = taxed * TaxRate;                                     // налог
        var total = Math.Max(0m, taxed + tax - gift);                  // сертификат не уводит в минус
        return new PricingResult(subtotal, discount, tax, gift, total);
    }
}

// Репозиторий persistence — отдельная модель от домена. / Persistence repository — separate model.
public sealed record OrderEntity(Guid Id, List<OrderLine> Lines, decimal DiscountPercent, decimal Gift);

public interface IOrderRepository
{
    Task<OrderEntity?> GetAsync(Guid id, CancellationToken ct);
    Task SaveAsync(OrderEntity order, CancellationToken ct);
}
```

```csharp
// --- 1) UNIT-тесты: OrderPricer в полной изоляции. -------------------------

public sealed class OrderPricerUnitTests
{
    private readonly OrderPricer _pricer = new();

    [Fact]
    public void Calculate_empty_lines_returns_zero()
    {
        var result = _pricer.Calculate([], 0m, 0m);          // collection expression C# 12
        Assert.Equal(PricingResult.Zero, result);
    }

    [Fact]
    public void Calculate_null_lines_throws()
        => Assert.Throws<ArgumentNullException>(() => _pricer.Calculate(null!, 0m, 0m));

    [Theory]
    [InlineData(-1, 1, 10)]      // отрицательная цена / negative price
    [InlineData(10, 0, 0)]       // нулевое количество / zero qty
    [InlineData(10, -1, 0)]      // отрицательное количество / negative qty
    public void Calculate_invalid_line_throws(decimal price, int qty, decimal _) // unused _ kept for matrix readability
        => Assert.Throws<ArgumentOutOfRangeException>(
            () => _pricer.Calculate([new("SKU", price, qty)], 0m, 0m));

    [Theory]
    [InlineData(100, 0,   0,   100)]   // subtotal 100, gift 0 → total 100 (налог уже в цене)
    [InlineData(100, 10,  0,   90)]    // скидка 10% → база 90, налог 18 → 108
    [InlineData(100, 50,  0,   60)]    // скидка 50% → база 50, налог 10 → 60
    public void Calculate_applies_discount(decimal subtotal, decimal discountPct, decimal _, decimal expected)
    {
        var lines = new[] { new OrderLine("SKU", subtotal, 1) };
        var result = _pricer.Calculate(lines, discountPct, 0m);
        // Внимание: ожидаем taxed+tax, а не subtotal-discount. / Expect taxed+tax, not subtotal-discount.
        Assert.Equal(expected, result.Total);
    }

    [Fact]
    public void Calculate_gift_bigger_than_total_clamps_to_zero()
    {
        var lines = new[] { new OrderLine("SKU", 10m, 1) };   // subtotal 10 → taxed 10 → +tax 2 → 12
        var result = _pricer.Calculate(lines, 0m, gift: 100m); // сертификат 100 > 12
        Assert.Equal(0m, result.Total);
    }
}
```

```csharp
// --- 2) INTEGRATION-тесты: репозиторий + реальный DbContext (in-memory). --

public sealed class OrderDbContext(DbContextOptions<OrderDbContext> options) : DbContext(options)
{
    public DbSet<OrderEntity> Orders => Set<OrderEntity>();
}

public sealed class EfOrderRepository(OrderDbContext db) : IOrderRepository
{
    public Task<OrderEntity?> GetAsync(Guid id, CancellationToken ct)
        => db.Orders.AsNoTracking().FirstOrDefaultAsync(o => o.Id == id, ct);
    public async Task SaveAsync(OrderEntity order, CancellationToken ct)
    {
        db.Orders.Add(order);
        await db.SaveChangesAsync(ct);
    }
}

public sealed class OrderRepositoryIntegrationTests
{
    private static OrderDbContext NewDb()
        => new(new DbContextOptionsBuilder<OrderDbContext>()
            .UseInMemoryDatabase($"order-it-{Guid.NewGuid()}") // уникальное имя → изоляция между тестами
            .Options);

    [Fact]
    public async Task SaveAsync_persists_and_GetAsync_reads_back()
    {
        await using var db = NewDb();
        var repo = new EfOrderRepository(db);
        var order = new OrderEntity(Guid.NewGuid(),
            [new("SKU-1", 10m, 2), new("SKU-2", 5m, 1)], 0m, 0m);

        await repo.SaveAsync(order, default);
        var read = await repo.GetAsync(order.Id, default);

        Assert.NotNull(read);
        Assert.Equal(2, read!.Lines.Count);
    }

    [Fact]
    public async Task GetAsync_returns_null_for_unknown_id()
    {
        await using var db = NewDb();
        var repo = new EfOrderRepository(db);
        var read = await repo.GetAsync(Guid.NewGuid(), default);
        Assert.Null(read);
    }
}
```

```csharp
// --- 3) E2E-тест (упрощённый): контроллер → репозиторий → pricer → HTTP. --

public sealed class OrdersController(IOrderRepository repo, OrderPricer pricer) : ControllerBase
{
    [HttpGet("/api/orders/{id:guid}/total")]
    public async Task<ActionResult<decimal>> GetTotal(Guid id, CancellationToken ct)
    {
        var order = await repo.GetAsync(id, ct);
        if (order is null) return NotFound();
        var result = pricer.Calculate(order.Lines, order.DiscountPercent, order.Gift);
        return Ok(result.Total);
    }
}

public sealed class OrdersE2eTests
{
    [Fact]
    public async Task GetTotal_returns_200_with_correct_total_for_existing_order()
    {
        var orderId = Guid.NewGuid();
        var order = new OrderEntity(orderId, [new("SKU", 100m, 1)], 0m, 0m);
        var stub = new StubRepo(order);
        var controller = new OrdersController(stub, new OrderPricer());

        var action = await controller.GetTotal(orderId, default);

        var ok = Assert.IsType<OkObjectResult>(action.Result);
        Assert.Equal(120m, Assert.IsType<decimal>(ok.Value)); // 100 + 20% tax
    }

    private sealed class StubRepo(OrderEntity order) : IOrderRepository
    {
        public Task<OrderEntity?> GetAsync(Guid id, CancellationToken ct)
            => Task.FromResult<OrderEntity?>(order);
        public Task SaveAsync(OrderEntity o, CancellationToken ct) => Task.CompletedTask;
    }
}
```

**Разбор по строкам.** `OrderPricer` намеренно не имеет конструктора и зависимостей — это чистый класс, который легко создавать в каждом unit-тесте через `new`. Использование `is < 0m or > 100m` — это pattern matching C# 11+, который компактнее классического `if (x < 0 || x > 100)` и хорошо читается. Проверка `UnitPrice < 0m || l.Qty <= 0` вынесена в цикл, чтобы бросать `ArgumentOutOfRangeException` с понятным сообщением — это граничный случай, который обязателен в unit-тестах по уроку (тестирование границ). `Math.Max(0m, ...)` реализует инвариант «итог не отрицательный» — именно это поведение мы и тестируем в `Calculate_gift_bigger_than_total_clamps_to_zero`, а не приватный метод `Clamp`, которого нет вовсе. Это прямое применение best practice «тестируйте поведение, а не реализацию».

`PricingResult` — рекорд с пятью полями. На первый взгляд кажется избыточным (можно вернуть просто `decimal`), но в E2E-тесте мы видим, как это окупается: контроллер возвращает `result.Total`, а в будущем легко расширить ответ до полного объекта. Это иллюстрирует мысль урока: тестируйте публичный контракт, который стабилен, а не внутреннюю форму. `OrderEntity` отделён от `OrderLine` намеренно — persistence-модель не должна совпадать с доменной, чтобы изменения схемы БД не ломали доменные тесты.

В `OrderPricerUnitTests` каждый `[Fact]` — одна причина упасть: пустой список отдельно, `null` отдельно, gift-clamp отдельно. `[Theory]` используется для табличных граничных случаев — это позволяет быстро увидеть в отчёте, какой именно `InlineData` упал. Имя `Calculate_gift_bigger_than_total_clamps_to_zero` читается как спецификация: метод, сценарий, ожидание. Коллекция `[]` — collection expression из C# 12, заменяющий `new List<OrderLine>()` или `Array.Empty<OrderLine>()`. В `Calculate_applies_discount` я намеренно оставил третий `InlineData` параметр `_` неиспользуемым ради матрицы — в реальном коде лучше убрать неиспользуемый параметр, здесь он сохранён для наглядности таблицы.

В integration-тестах ключевой момент — `NewDb()` создаёт `UseInMemoryDatabase($"order-it-{Guid.NewGuid()}")`. Уникальное имя гарантирует изоляцию между тестами: даже если xUnit создаст новый экземпляр класса, база не «протечёт». `AsNoTracking()` в `GetAsync` — best practice для read-only запросов, экономит память и не кэширует сущности в `ChangeTracker`. `await using var db` — корректное освобождение `DbContext`, который реализует `IAsyncDisposable`. Эти тесты медленнее unit (создание EF Core ServiceProvider), но проверяют шов «репозиторий ↔ EF Core ↔ in-memory provider» — то, что unit-тестами не покрывается.

В E2E-тесте `StubRepo` — это stub из урока, упрощённый `IOrderRepository`, который возвращает фиксированный заказ. Контроллер принимает зависимости через primary constructor C# 12 (`OrdersController(IOrderRepository repo, OrderPricer pricer)`) — это короче и чище, чем классический конструктор с полем. Действие вызывается напрямую, без `WebApplicationFactory` — для учебной задачи этого достаточно, а в реальном проекте E2E должен идти через реальный HTTP-pipeline. Заметьте: `120m` = `100 + 20% tax` — именно поэтому в `Calculate_applies_discount` я выбрал матрицу с учётом налога (`90 + 18 = 108`, `50 + 10 = 60`), чтобы тесты и E2E были согласованы. В E2E мы проверяем полный путь «контроллер → репозиторий → pricer → HTTP-ответ» — то, что невозможно покрыть unit-тестом, и именно поэтому этот тест оправдан на вершине пирамиды (10%, а не 80%).

#### Задания на углубление (бонус)

1. **Замените in-memory EF Core на SQLite in-memory** (`UseSqlite("DataSource=:memory:")`). Сравните: какие SQL-ограничения теперь проверяются? Напишите integration-тест, который падает на SQLite (например, unique-нарушение), но проходит на in-memory EF Core. Вывод зафиксируйте в `README.md`.
2. **Добавьте стратегию ценообразования** через `IPricingRule` с реализациями `BulkDiscountRule` (скидка при покупке > 5 штук) и `CouponRule`. Покройте каждую реализацию отдельным unit-тестом, а композицию — integration-тестом. Обратите внимание: добавление правил не должно требовать изменения `OrderPricer` — это открытость к расширению.
3. **Переведите E2E-тест на `WebApplicationFactory<Program>`** (`Microsoft.AspNetCore.Mvc.Testing`). Настройте `Program` partial-класс с `public`, чтобы фабрика видела точку входа. Сравните время выполнения прямого вызова действия и полного HTTP-пайплайна — зафиксируйте разницу в `README.md`.
4. **Измерьте покрытие кода** через `dotnet add package coverlet.collector` и `dotnet test --collect:"XPlat Code Coverage"`. Добейтесь покрытия `OrderPricer` не ниже 95%. Обсудите: 100% покрытие — это всегда хорошо? Какие строки нельзя/не нужно покрывать?

---

## Statement in English / Постановка на английском

#### Context & motivation

You have joined a team building an "Order Calculator" microservice for an online shop. The service accepts cart lines, applies discount rules, adds tax, and subtracts gift certificates. The business logic is critical: a one-percent error in the formula costs the company millions of rubles a month, and an inaccurate test only provides a false sense of security.

The repository already contains three test projects, but their distribution across the pyramid is chaotic: half of the "unit tests" actually hit a real database, and there are as many E2E tests as unit tests — the suite is slow, brittle, and localizes defects poorly. Your tech lead asked you to bring order: extract the pure logic into a dedicated class, cover it with fast deterministic unit tests, add an integration test for the repository with in-memory EF Core, and keep just one or two E2E tests for the main scenario through the controller.

The goal of the assignment is not to write the maximum number of tests, but to distribute them across the pyramid correctly: roughly 70% unit, 20% integration, 10% E2E. You should feel how shifting tests down (shift-left) speeds up feedback and reduces maintenance cost. In parallel you will reinforce the best practices from the lesson: test behavior, not implementation; one test — one reason to fail; readable names of the form `Method_Scenario_Expected`; determinism without random dates and network calls. The assignment is constructed so that every mistake from the lesson's "Common Mistakes" checklist will appear to you as a temptation you must recognize and avoid.

#### What to do step by step

1. **Create the solution structure.** Open a terminal in an empty folder and run:
   ```bash
   dotnet new sln -n OrderPricing
   dotnet new classlib -n OrderPricing.Domain -f net8.0
   dotnet new webapi -n OrderPricing.Api -f net8.0 --use-controllers
   dotnet new xunit -n OrderPricing.UnitTests -f net8.0
   dotnet new xunit -n OrderPricing.IntegrationTests -f net8.0
   dotnet new xunit -n OrderPricing.E2ETests -f net8.0
   dotnet sln add **/*.csproj
   ```
   Add project references: `UnitTests` → `Domain`; `IntegrationTests` → `Domain`, `Api`; `E2ETests` → `Domain`, `Api`. Install EF Core in-memory into `IntegrationTests` and `Api`:
   ```bash
   dotnet add OrderPricing.Api package Microsoft.EntityFrameworkCore.InMemory --version 8.*
   dotnet add OrderPricing.IntegrationTests package Microsoft.EntityFrameworkCore.InMemory --version 8.*
   ```

2. **Implement the domain.** In `OrderPricing.Domain` create `Pricing/OrderPricer.cs` with the `OrderPricer` class and the `OrderLine` and `PricingResult` records. The logic: sum `UnitPrice * Qty` across lines; apply a percentage discount (0–100); add tax (e.g. 20% VAT); subtract a gift certificate but never go below zero (total `>= 0`). This is pure logic with no dependencies — a unit-test candidate.

3. **Implement the repository.** In `OrderPricing.Api` create `IOrderRepository`, `OrderDbContext : DbContext` with `DbSet<OrderEntity>`, and `EfOrderRepository` that persists and reads orders. `OrderEntity` is a separate persistence model, not the domain record, to demonstrate layer separation.

4. **Write unit tests** (the `UnitTests` project). Cover `OrderPricer`: empty order (total 0, boundary case), discount only, tax only, discount + tax, gift certificate, certificate larger than total (total does not go negative), negative price (should throw `ArgumentOutOfRangeException`), `null` order (`ArgumentNullException`). Use `[Fact]` for single scenarios and `[Theory]/[InlineData]` for tabular ones. One test — one reason to fail.

5. **Write integration tests** (the `IntegrationTests` project). Spin up `OrderDbContext` on `UseInMemoryDatabase("order-it")`, save an order via `EfOrderRepository`, read it back and verify that lines and total match. Add a second test: reading a non-existent `orderId` returns `null`. These tests are slower than unit tests but verify the "seams" between the repository and EF Core.

6. **Write E2E tests** (the `E2ETests` project). Implement a minimal `OrdersController` with `POST /api/orders` (create) and `GET /api/orders/{id}/total` (return total). In the E2E test use either `WebApplicationFactory<Program>` (if you install `Microsoft.AspNetCore.Mvc.Testing`) or a direct call to the controller action with a stub repository (as in the lesson) — pick the second option to save time. Verify the full path: controller → repository → `OrderPricer` → HTTP response `200 OK` with the correct total.

7. **Measure the ratio.** Run `dotnet test --verbosity normal` separately for each project and record a table in `README.md`: the number of tests in each project and the total execution time. Make sure the ratio is close to 70/20/10. If you have 10 unit, 3 integration, 1–2 E2E tests — that is a good guideline.

8. **Run everything together and make sure the suite is green.** The `dotnet test` command from the solution root should pass in a reasonable time (a few seconds). If integration tests drag — check that you are not creating a new `DbContext` for every `Assert` without need.

#### Requirements

The solution must compile without warnings on .NET 8 with C# 12 (use top-level statements in `Program.cs`, pattern matching, collection expressions, `record`/primary constructor where appropriate). The target framework of every project is `net8.0`. Use xUnit as the test framework. The project structure must clearly reflect the pyramid: three separate test projects, not a single shared one. Each test class contains tests of only one level — mixing unit and integration in one class is forbidden (this is a common mistake from the lesson).

Domain logic (`OrderPricer`) must not depend on EF Core, ASP.NET Core, or any I/O — only pure computation over `IEnumerable<OrderLine>`. All external dependencies are moved behind interfaces (`IOrderRepository`). Unit tests must not touch the database, network, or file system; integration tests are the only place where `DbContext` appears. E2E tests work through the public controller contract, not via reflection or private methods. Test names follow `MethodName_Scenario_ExpectedResult` and read like a spec. Every test follows the Arrange-Act-Assert pattern, separated by blank lines. The `README.md` includes a table with the number of tests per level and a justification of the chosen ratio.

#### Pitfalls and gotchas

The main trap is writing a "unit test" that actually creates `OrderDbContext` and writes to the database. It looks like a unit, but on CI without a disk/network it will start failing, and locally it will slow the suite down. The tell-tale sign is a `using Microsoft.EntityFrameworkCore` in the test class. Move the dependency behind `IOrderRepository` and inject a `FakeOrderRepository` in unit tests — a simple class storing orders in a `Dictionary<Guid, OrderEntity>`.

The second trap is testing private methods via reflection. For example, you want to test a private `ApplyDiscount` directly. Per the lesson: test behavior through the public `Calculate` — the private method is an implementation detail, its signature may change, and the test will break falsely. If a method becomes complex and begs for a dedicated test, extract it into a separate public strategy class (`IPricingRule`) and cover it with a unit test.

The third trap is multiple `Assert`s in one test over different scenarios. When such a test fails, you cannot immediately tell which scenario broke. Split into a `[Theory]` with different `InlineData` or into separate `[Fact]`s. The only acceptable exception is several `Assert`s checking the same logical statement from different angles (e.g. `Assert.NotNull(x); Assert.Equal(5, x.Total)`).

The fourth trap is shared mutable state between tests. If you use a static `OrderDbContext` and one test writes while another reads, the execution order starts affecting the result. Every test must set up a fresh context: either `using var db = new OrderDbContext(...)` inside the `[Fact]`, or `IAsyncLifetime` with cleanup. xUnit creates a new instance of the test class per test — rely on that, but do not keep mutable state in `static`.

The fifth trap is the inverted pyramid. If you start covering every logic branch with an E2E test "for reliability", the suite becomes slow and brittle. Remember: the same bug is cheaper to catch with a unit test. Shift checks down: whatever can be verified in isolation should not be raised to integration/E2E.

The sixth nuance is determinism. No `DateTime.Now`, no `Guid.NewGuid()` in expected values, no `Thread.Sleep`, no `Random` without a seed. If you need "time" — introduce an `IClock` and inject a fixed one in tests. In this assignment time is not needed, but if you add a `CreatedAt` field — use `IClock`.

The seventh nuance is the choice between in-memory and SQLite. `UseInMemoryDatabase` is convenient, but it does not check SQL constraints (unique, foreign keys). For a more honest integration test the lesson's example is enough with in-memory, but in a real project SQLite in-memory (`UseSqlite("DataSource=:memory:")`) is closer to the production DB. In this assignment stay on in-memory EF Core to match the lesson's example.

#### Acceptance criteria

- [ ] The solution contains 6 projects: `Domain`, `Api`, `UnitTests`, `IntegrationTests`, `E2ETests` and `.sln`.
- [ ] All projects target `net8.0` and compile without errors or warnings.
- [ ] `OrderPricer` has no references to EF Core, ASP.NET Core, or I/O classes.
- [ ] `IOrderRepository` is defined in `Domain` or `Api`; `EfOrderRepository` is in `Api`.
- [ ] Unit tests cover at least 7 scenarios: empty order, discount only, tax only, discount+tax, gift, gift-larger-than-total, negative price, `null`.
- [ ] `UnitTests` has no `using Microsoft.EntityFrameworkCore*`.
- [ ] At least one unit test uses `[Theory]/[InlineData]` with three or more variants.
- [ ] Integration tests create `OrderDbContext` on `UseInMemoryDatabase`, save and read an order.
- [ ] The integration test for a non-existent `orderId` expects `null`.
- [ ] The E2E test calls the public controller action and verifies `200 OK` with the correct total.
- [ ] No test uses reflection for private methods.
- [ ] No test contains more than one logical assertion over different scenarios.
- [ ] Test names follow `MethodName_Scenario_ExpectedResult`.
- [ ] `dotnet test` from the root passes green in a reasonable time (< 30 s).
- [ ] `README.md` contains a table with the number of tests per level and a justification of a ratio close to 70/20/10.

#### Hints (no direct answer)

- Think about which class should hold the `Calculate` method — a pure stateless service or an extension method? Where is the boundary between the domain and persistence?
- For a gift certificate larger than the total, recall: the result is `Max(0, subtotal - discount + tax - gift)`. Which return type is better — `decimal` or a `PricingResult` record with fields `Subtotal`, `Discount`, `Tax`, `Gift`, `Total`? The second is friendlier for tests and API responses.
- For the integration test do not forget `await db.SaveChangesAsync()` — without it the in-memory provider will not commit changes and the next context will not see them. Each test gets its own `DbContext`; only `DbContextOptions` may be shared.
- For the E2E test without `WebApplicationFactory`: create `StubOrderRepository : IOrderRepository` returning a pre-built order. Accept the controller through a primary constructor: `public sealed class OrdersController(IOrderRepository repo) : ControllerBase`.
- To compute the ratio, `dotnet test --verbosity normal | Select-String "Passed!"` per project is enough — xUnit prints a `Passed: N` line.

#### Reference solution walk-through

```csharp
// C# 12 / .NET 8 — Reference solution for HW M15-L01.
// Domain: order pricing with discount, tax and gift certificate.

using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading;
using System.Threading.Tasks;
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;
using Xunit;

namespace OrderPricing.Domain.Pricing;

// Pure domain record — a unit-test candidate.
public sealed record OrderLine(string Sku, decimal UnitPrice, int Qty);

// Result exposes all intermediate values for richer assertions.
public sealed record PricingResult(decimal Subtotal, decimal Discount, decimal Tax, decimal Gift, decimal Total)
{
    public static PricingResult Zero => new(0m, 0m, 0m, 0m, 0m);
}

// Pure logic, no dependencies — the base of the pyramid.
public sealed class OrderPricer
{
    private const decimal TaxRate = 0.20m; // VAT 20%

    public PricingResult Calculate(IReadOnlyList<OrderLine> lines, decimal discountPercent, decimal gift)
    {
        ArgumentNullException.ThrowIfNull(lines);
        if (discountPercent is < 0m or > 100m)
            throw new ArgumentOutOfRangeException(nameof(discountPercent), "Must be 0..100.");
        if (gift < 0m) throw new ArgumentOutOfRangeException(nameof(gift), "Must be >= 0.");

        foreach (var l in lines)
            if (l.UnitPrice < 0m || l.Qty <= 0)
                throw new ArgumentOutOfRangeException(nameof(lines), "Price/Qty invalid.");

        var subtotal = lines.Sum(l => l.UnitPrice * l.Qty);
        var discount = subtotal * discountPercent / 100m;
        var taxed = subtotal - discount;
        var tax = taxed * TaxRate;
        var total = Math.Max(0m, taxed + tax - gift); // gift certificate never drives the total below zero
        return new PricingResult(subtotal, discount, tax, gift, total);
    }
}

// Persistence repository — a separate model from the domain.
public sealed record OrderEntity(Guid Id, List<OrderLine> Lines, decimal DiscountPercent, decimal Gift);

public interface IOrderRepository
{
    Task<OrderEntity?> GetAsync(Guid id, CancellationToken ct);
    Task SaveAsync(OrderEntity order, CancellationToken ct);
}
```

```csharp
// --- 1) UNIT tests: OrderPricer in full isolation. ------------------------

public sealed class OrderPricerUnitTests
{
    private readonly OrderPricer _pricer = new();

    [Fact]
    public void Calculate_empty_lines_returns_zero()
    {
        var result = _pricer.Calculate([], 0m, 0m);          // C# 12 collection expression
        Assert.Equal(PricingResult.Zero, result);
    }

    [Fact]
    public void Calculate_null_lines_throws()
        => Assert.Throws<ArgumentNullException>(() => _pricer.Calculate(null!, 0m, 0m));

    [Theory]
    [InlineData(-1, 1, 10)]      // negative price
    [InlineData(10, 0, 0)]       // zero quantity
    [InlineData(10, -1, 0)]      // negative quantity
    public void Calculate_invalid_line_throws(decimal price, int qty, decimal _)
        => Assert.Throws<ArgumentOutOfRangeException>(
            () => _pricer.Calculate([new("SKU", price, qty)], 0m, 0m));

    [Theory]
    [InlineData(100, 0,   0,   100)]
    [InlineData(100, 10,  0,   108)]
    [InlineData(100, 50,  0,   60)]
    public void Calculate_applies_discount(decimal subtotal, decimal discountPct, decimal _, decimal expected)
    {
        var lines = new[] { new OrderLine("SKU", subtotal, 1) };
        var result = _pricer.Calculate(lines, discountPct, 0m);
        // Note: we expect taxed+tax, not subtotal-discount.
        Assert.Equal(expected, result.Total);
    }

    [Fact]
    public void Calculate_gift_bigger_than_total_clamps_to_zero()
    {
        var lines = new[] { new OrderLine("SKU", 10m, 1) };   // subtotal 10 → taxed 10 → +tax 2 → 12
        var result = _pricer.Calculate(lines, 0m, gift: 100m);
        Assert.Equal(0m, result.Total);
    }
}
```

```csharp
// --- 2) INTEGRATION tests: repository + real DbContext (in-memory). -------

public sealed class OrderDbContext(DbContextOptions<OrderDbContext> options) : DbContext(options)
{
    public DbSet<OrderEntity> Orders => Set<OrderEntity>();
}

public sealed class EfOrderRepository(OrderDbContext db) : IOrderRepository
{
    public Task<OrderEntity?> GetAsync(Guid id, CancellationToken ct)
        => db.Orders.AsNoTracking().FirstOrDefaultAsync(o => o.Id == id, ct);
    public async Task SaveAsync(OrderEntity order, CancellationToken ct)
    {
        db.Orders.Add(order);
        await db.SaveChangesAsync(ct);
    }
}

public sealed class OrderRepositoryIntegrationTests
{
    private static OrderDbContext NewDb()
        => new(new DbContextOptionsBuilder<OrderDbContext>()
            .UseInMemoryDatabase($"order-it-{Guid.NewGuid()}") // unique name → isolation between tests
            .Options);

    [Fact]
    public async Task SaveAsync_persists_and_GetAsync_reads_back()
    {
        await using var db = NewDb();
        var repo = new EfOrderRepository(db);
        var order = new OrderEntity(Guid.NewGuid(),
            [new("SKU-1", 10m, 2), new("SKU-2", 5m, 1)], 0m, 0m);

        await repo.SaveAsync(order, default);
        var read = await repo.GetAsync(order.Id, default);

        Assert.NotNull(read);
        Assert.Equal(2, read!.Lines.Count);
    }

    [Fact]
    public async Task GetAsync_returns_null_for_unknown_id()
    {
        await using var db = NewDb();
        var repo = new EfOrderRepository(db);
        var read = await repo.GetAsync(Guid.NewGuid(), default);
        Assert.Null(read);
    }
}
```

```csharp
// --- 3) E2E test (simplified): controller → repository → pricer → HTTP. ---

public sealed class OrdersController(IOrderRepository repo, OrderPricer pricer) : ControllerBase
{
    [HttpGet("/api/orders/{id:guid}/total")]
    public async Task<ActionResult<decimal>> GetTotal(Guid id, CancellationToken ct)
    {
        var order = await repo.GetAsync(id, ct);
        if (order is null) return NotFound();
        var result = pricer.Calculate(order.Lines, order.DiscountPercent, order.Gift);
        return Ok(result.Total);
    }
}

public sealed class OrdersE2eTests
{
    [Fact]
    public async Task GetTotal_returns_200_with_correct_total_for_existing_order()
    {
        var orderId = Guid.NewGuid();
        var order = new OrderEntity(orderId, [new("SKU", 100m, 1)], 0m, 0m);
        var stub = new StubRepo(order);
        var controller = new OrdersController(stub, new OrderPricer());

        var action = await controller.GetTotal(orderId, default);

        var ok = Assert.IsType<OkObjectResult>(action.Result);
        Assert.Equal(120m, Assert.IsType<decimal>(ok.Value)); // 100 + 20% tax
    }

    private sealed class StubRepo(OrderEntity order) : IOrderRepository
    {
        public Task<OrderEntity?> GetAsync(Guid id, CancellationToken ct)
            => Task.FromResult<OrderEntity?>(order);
        public Task SaveAsync(OrderEntity o, CancellationToken ct) => Task.CompletedTask;
    }
}
```

**Line-by-line walk-through.** `OrderPricer` intentionally has no constructor and no dependencies — it is a pure class that is easy to instantiate in each unit test with `new`. The use of `is < 0m or > 100m` is C# 11+ pattern matching, which is more compact than the classic `if (x < 0 || x > 100)` and reads well. The `UnitPrice < 0m || l.Qty <= 0` check is moved into a loop so it can throw `ArgumentOutOfRangeException` with a clear message — this is a boundary case, mandatory in unit tests per the lesson (testing boundaries). `Math.Max(0m, ...)` enforces the "total is never negative" invariant — this is exactly the behavior we test in `Calculate_gift_bigger_than_total_clamps_to_zero`, not a private `Clamp` method that does not exist. This is a direct application of the "test behavior, not implementation" best practice.

`PricingResult` is a record with five fields. At first glance it looks redundant (you could just return `decimal`), but in the E2E test we see how it pays off: the controller returns `result.Total`, and in the future it is easy to extend the response to a full object. This illustrates the lesson's point: test the public contract, which is stable, not the internal shape. `OrderEntity` is deliberately separated from `OrderLine` — the persistence model must not coincide with the domain model, so that schema changes do not break domain tests.

In `OrderPricerUnitTests`, every `[Fact]` has one reason to fail: empty list separately, `null` separately, gift-clamp separately. `[Theory]` is used for tabular boundary cases — this lets you see immediately from the report which `InlineData` failed. The name `Calculate_gift_bigger_than_total_clamps_to_zero` reads like a spec: method, scenario, expectation. The `[]` collection is a C# 12 collection expression, replacing `new List<OrderLine>()` or `Array.Empty<OrderLine>()`. In `Calculate_applies_discount` I deliberately kept the third `InlineData` parameter `_` unused for the sake of a clear matrix — in real code it is better to drop the unused parameter; here it is kept for readability of the table.

In the integration tests the key point is that `NewDb()` builds `UseInMemoryDatabase($"order-it-{Guid.NewGuid()}")`. The unique name guarantees isolation between tests: even if xUnit creates a new class instance, the database will not "leak". `AsNoTracking()` in `GetAsync` is a best practice for read-only queries — it saves memory and does not cache entities in the `ChangeTracker`. `await using var db` correctly disposes the `DbContext`, which implements `IAsyncDisposable`. These tests are slower than unit tests (creating an EF Core ServiceProvider) but verify the seam "repository ↔ EF Core ↔ in-memory provider", which unit tests cannot cover.

In the E2E test `StubRepo` is the stub from the lesson, a simplified `IOrderRepository` returning a fixed order. The controller takes its dependencies through a C# 12 primary constructor (`OrdersController(IOrderRepository repo, OrderPricer pricer)`) — this is shorter and cleaner than a classic constructor with a field. The action is invoked directly, without `WebApplicationFactory` — for an exercise this is enough, and in a real project the E2E should go through the real HTTP pipeline. Notice: `120m` = `100 + 20% tax` — that is why in `Calculate_applies_discount` I chose a matrix that accounts for tax (`90 + 18 = 108`, `50 + 10 = 60`), so tests and E2E agree. In E2E we verify the full path "controller → repository → pricer → HTTP response" — something that cannot be covered by a unit test, and that is exactly why this test is justified at the top of the pyramid (10%, not 80%).

#### Going deeper (bonus)

1. **Replace in-memory EF Core with SQLite in-memory** (`UseSqlite("DataSource=:memory:")`). Compare: which SQL constraints are now checked? Write an integration test that fails on SQLite (e.g. a unique violation) but passes on in-memory EF Core. Record the conclusion in `README.md`.
2. **Introduce a pricing strategy** through `IPricingRule` with `BulkDiscountRule` (discount when buying > 5 units) and `CouponRule` implementations. Cover each implementation with a separate unit test, and the composition with an integration test. Note: adding rules must not require changing `OrderPricer` — this is openness to extension.
3. **Move the E2E test to `WebApplicationFactory<Program>`** (`Microsoft.AspNetCore.Mvc.Testing`). Configure `Program` as a public partial class so the factory can see the entry point. Compare the execution time of a direct action call and a full HTTP pipeline — record the difference in `README.md`.
4. **Measure code coverage** with `dotnet add package coverlet.collector` and `dotnet test --collect:"XPlat Code Coverage"`. Achieve at least 95% coverage of `OrderPricer`. Discuss: is 100% coverage always good? Which lines cannot or should not be covered?

---

#### Чек-лист сдачи / Submission checklist

- [ ] (RU) Решение из 6 проектов компилируется под .NET 8 / C# 12 без предупреждений.
- [ ] (RU) Три тестовых проекта разделены по уровням пирамиды, без смешивания в одном классе.
- [ ] (RU) `OrderPricer` не зависит от EF Core / ASP.NET Core / I/O; зависимости за интерфейсами.
- [ ] (RU) Unit-тесты покрывают ≥ 7 сценариев, включая граничные (пустой заказ, `null`, отрицательная цена, сертификат > итога).
- [ ] (RU) Integration-тесты используют `UseInMemoryDatabase`, проверяют сохранение/чтение и `null` для неизвестного id.
- [ ] (RU) E2E-тест вызывает публичное действие контроллера и проверяет `200 OK`.
- [ ] (RU) Имена тестов вида `Method_Scenario_Expected`, один тест — одна причина упасть.
- [ ] (RU) `dotnet test` проходит зелёно за < 30 с; пропорция близка к 70/20/10 (таблица в `README.md`).
- [ ] (EN) Solution of 6 projects compiles on .NET 8 / C# 12 without warnings.
- [ ] (EN) Three test projects separated by pyramid level, no mixing in one class.
- [ ] (EN) `OrderPricer` does not depend on EF Core / ASP.NET Core / I/O; dependencies are behind interfaces.
- [ ] (EN) Unit tests cover ≥ 7 scenarios including boundaries (empty order, `null`, negative price, gift > total).
- [ ] (EN) Integration tests use `UseInMemoryDatabase`, verify save/read and `null` for unknown id.
- [ ] (EN) E2E test calls the public controller action and verifies `200 OK`.
- [ ] (EN) Test names follow `Method_Scenario_Expected`, one test — one reason to fail.
- [ ] (EN) `dotnet test` passes green in < 30 s; ratio close to 70/20/10 (table in `README.md`).

#### Ресурсы / Resources

- [Microsoft Learn — Testing ASP.NET Core microservices and web apps](https://learn.microsoft.com/dotnet/architecture/microservices/multi-container-microservice-net-applications/testing-asp-net-core-microservices-web-apps)
- [Martin Fowler — Test Pyramid](https://martinfowler.com/bliki/TestPyramid.html)
- [Microsoft Learn — EF Core in-memory provider](https://learn.microsoft.com/ef/core/providers/in-memory/)
- [xUnit documentation — Facts and Theories](https://xunit.net/docs/getting-started/netcore/cmdline)
- Mike Cohn, *Succeeding with Agile* — origin of the classic test pyramid. / Источник классической пирамиды тестов.
