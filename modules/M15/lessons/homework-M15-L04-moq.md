---
[← К уроку M15-L04](lesson-M15-L04-moq.md) | [⬆ К модулю M15](../README.md) | [Следующее ДЗ →](homework-M15-L05-testing-di-dbcontext.md)
---

### Домашнее задание M15-L04: Moq: Mock<T>, Setup, Verify, ItExpr / Homework M15-L04: Moq: Mock<T>, Setup, Verify, ItExpr

**Урок / Lesson:** M15-L04
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 3/5
**Цель / Goal:** (RU) Научиться декларативно описывать поведение зависимостей через `Mock<T>`, `Setup`, `Returns`/`ReturnsAsync`/`Throws`/`ThrowsAsync`, проверять значимые взаимодействия через `Verify` + `Times`, применять `It.Is<T>` вместо `It.IsAny<T>` там, где аргумент важен, перехватывать аргументы через `Callback`, а также настраивать и верифицировать защищённые методы через `Moq.Protected` + `ItExpr`. Различать проверку состояния (state) и проверку поведения (behavior).
(EN) Learn to describe dependency behavior declaratively through `Mock<T>`, `Setup`, `Returns`/`ReturnsAsync`/`Throws`/`ThrowsAsync`, verify meaningful interactions through `Verify` + `Times`, prefer `It.Is<T>` over `It.IsAny<T>` when an argument matters, capture arguments with `Callback`, and set up / verify protected members through `Moq.Protected` + `ItExpr`. Distinguish state verification from behavior verification.

#### Связь с уроком / Connection to the lesson
(RU) Урок M15-L04 вводит базовый сценарий Moq из трёх шагов (создать мок — настроить `Setup` — проверить `Verify`) и разбирает `It.IsAny`, `It.Is`, `Callback`, `Raise`, `MockBehavior.Strict`/`Loose`, а также `ItExpr` + `Moq.Protected` для защищённых членов. В этом задании вы примените каждую из этих техник к осмысленному доменному сценарию «обработка заказа», где легко допустить типичные ошибки: забыть `Returns`, мокать не-виртуальный член, переборщить с `Verify` или использовать `It.IsAny` там, где важен конкретный аргумент.
(EN) Lesson M15-L04 introduces the three-step Moq flow (create the mock — configure `Setup` — verify `Verify`) and covers `It.IsAny`, `It.Is`, `Callback`, `Raise`, `MockBehavior.Strict`/`Loose`, and `ItExpr` + `Moq.Protected` for protected members. In this homework you will apply every one of these techniques to a meaningful "order processing" domain scenario, where it is easy to fall into the classic traps: forgetting `Returns`, mocking a non-virtual member, over-verifying, or using `It.IsAny` where a specific argument matters.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация
Представьте, что вы поддерживаете модуль оформления заказов интернет-магазина на C# 12 / .NET 8. Бизнес-логика обработки заказа (`OrderProcessor`) зависит от четырёх внешних сущностей: репозитория заказов `IOrderRepository`, платёжного шлюза `IPaymentGateway`, сервиса резервирования на складе `IInventoryService` и абстрактного класса-фрауд-чекера `FraudChecker` с защищённым методом `AssessRiskAsync`. Эти зависимости в реальной жизни ходят в базу данных, обращаются к внешнему HTTP-API банка и к стороннему антифрод-сервису — то есть их нельзя вызывать в юнит-тестах. Вы должны изолировать `OrderProcessor` от них с помощью Moq.

Мотивация именно такой конфигурации в том, что она естественным образом заставляет вас воспользоваться всем арсеналом урока: асинхронными `ReturnsAsync`/`ThrowsAsync` для `Task<T>`-методов, `It.Is<T>(predicate)` для проверки суммы и email при оплате, `Callback` для захвата сохранённого заказа и последующей state-проверки его статуса, `Times.Never` для негативных веток (оплата не должна происходить, если заказ не найден или фрауд-чекер заблокировал заказ), `Moq.Protected` + `ItExpr` для защищённого `AssessRiskAsync`, а также `MockBehavior.Strict` для одного из тестов, чтобы увидеть, как строгий режим ловит забытые настройки.

Важно понимать разницу между проверкой состояния и проверкой поведения. State-проверка отвечает «каков результат работы SUT?» — например, вернул ли `ProcessAsync` `true` и сохранил ли заказ со статусом `Completed`. Behavior-проверка отвечает «что SUT сделал с зависимостью?» — вызвал ли он `ReserveAsync` ровно один раз, не вызвал ли `ChargeAsync` при заблокированном фраудом заказе. Хороший тест комбинирует оба подхода, но не перегружает себя лишними `Verify`: каждый дополнительный верификат делает тест хрупким к безобидному рефакторингу.

#### Что нужно сделать (пошагово)
1. Создайте два проекта в одном решении. Выполните:
   - `dotnet new sln -n Shop`
   - `dotnet new classlib -n Shop -o src/Shop --framework net8.0`
   - `dotnet new xunit -n Shop.Tests -o tests/Shop.Tests --framework net8.0`
   - `dotnet sln add src/Shop tests/Shop.Tests`
   - `dotnet add tests/Shop.Tests reference src/Shop`
   - `dotnet add tests/Shop.Tests package Moq`
   - `dotnet add tests/Shop.Tests package FluentAssertions` (опционально)
2. В проекте `src/Shop` создайте файл `OrderProcessor.cs` и скопируйте туда production-код из раздела «Постановка: production-код» ниже. Это доменные типы (`Order`, `OrderStatus`, `PaymentResult`, `RiskScore`), интерфейсы зависимостей, абстрактный класс `FraudChecker` и сам `OrderProcessor`. Этот код дан вам готовым — его менять не нужно; ваша задача — тесты.
3. В проекте `tests/Shop.Tests` создайте файл `OrderProcessorTests.cs`. Реализуйте класс `OrderProcessorTests` с приватными полями-моками `Mock<IOrderRepository>`, `Mock<IPaymentGateway>`, `Mock<IInventoryService>` и `Mock<FraudChecker>`. Заметьте: для `FraudChecker` нужен `CallBase = true`, потому что публичный метод `CheckAsync` — конкретный (не виртуальный), и без `CallBase` он вернёт `default` вместо того, чтобы делегировать в `AssessRiskAsync`.
4. В конструкторе тест-класса создайте `OrderProcessor` (это SUT), передав в него `.Object` всех моков. Базовый режим — `MockBehavior.Loose`, как рекомендует урок для большинства сценариев.
5. Напишите тесты для следующих сценариев (минимум шесть). Каждый тест должен содержать явные секции Arrange / Act / Assert:
   - `Process_HappyPath_CompletesOrderAndReserves`: заказ найден, риск низкий, оплата успешна → метод возвращает `true`, заказ сохраняется со статусом `Completed`, `ReserveAsync` вызывается один раз, `ChargeAsync` вызывается один раз именно с нужными email и суммой.
   - `Process_MissingOrder_ThrowsAndSkipsPayment`: `GetByIdAsync` возвращает `null` → `InvalidOperationException`, и `ChargeAsync` / `ReserveAsync` не вызываются (`Times.Never`).
   - `Process_HighRisk_ThrowsBeforePayment`: `RiskScore.Level > 80` → `InvalidOperationException`, `ChargeAsync` не вызывается.
   - `Process_PaymentFails_MarksFailedAndSkipsInventory`: `PaymentResult.Success == false` → метод возвращает `false`, заказ сохраняется со статусом `Failed`, `ReserveAsync` не вызывается.
   - `Process_PaymentThrows_PropagatesAndSkipsInventory`: `ChargeAsync` выбрасывает `InvalidOperationException` через `ThrowsAsync` → исключение пробрасывается наружу, `ReserveAsync` и `SaveAsync` не вызываются.
   - `Process_StrictMock_RejectsUnconfiguredCalls`: тот же счастливый путь, но все моки созданы с `MockBehavior.Strict` — каждый вызов должен быть явно настроен, иначе Moq выбросит `MockException`.
6. Для защищённого метода `FraudChecker.AssessRiskAsync` используйте именно `Moq.Protected()` + `ItExpr.IsAny<Order>()` и `ReturnsAsync`/`Verify` по строковому имени `"AssessRiskAsync"`. Обычный `Setup` по выражению для `protected`-члена недоступен — это ключевая тонкость урока.
7. Для захвата сохранённого заказа в счастливом пути и в ветке отказа оплаты используйте `Callback<Order>(o => saved = o)` перед `Returns(Task.CompletedTask)`. Затем делайте state-проверку `saved.Status`.
8. Запустите тесты: `dotnet test`. Ожидаемый вывод — все тесты зелёные. Если Moq выбрасывает `Expression is not a method invocation` или `All invocations on the mock must have a corresponding setup` — вы забыли настройку или используете `It.IsAny` вместо конкретного аргумента в Strict-режиме.

#### Постановка: production-код (не менять)
```csharp
// src/Shop/OrderProcessor.cs — C# 12 / .NET 8
using System;
using System.Threading.Tasks;

namespace Shop;

public enum OrderStatus { Pending, Paid, Completed, Failed }

public sealed record Order(int Id, string CustomerEmail, decimal Total, OrderStatus Status);

public sealed record PaymentResult(bool Success, string TransactionId);

public sealed record RiskScore(int Level, string Reason);

public interface IOrderRepository
{
    Task<Order?> GetByIdAsync(int id);
    Task SaveAsync(Order order);
}

public interface IPaymentGateway
{
    Task<PaymentResult> ChargeAsync(string email, decimal amount);
}

public interface IInventoryService
{
    Task ReserveAsync(int orderId);
}

// Абстрактный класс с protected internal abstract методом.
// Для его настройки и верификации нужен Moq.Protected + ItExpr.
public abstract class FraudChecker
{
    protected internal abstract Task<RiskScore> AssessRiskAsync(Order order);

    public Task<RiskScore> CheckAsync(Order order) => AssessRiskAsync(order);
}

public sealed class OrderProcessor(
    IOrderRepository repo,
    IPaymentGateway payment,
    IInventoryService inventory,
    FraudChecker fraud)
{
    public async Task<bool> ProcessAsync(int orderId)
    {
        var order = await repo.GetByIdAsync(orderId)
            ?? throw new InvalidOperationException($"Order {orderId} not found");

        var risk = await fraud.CheckAsync(order);
        if (risk.Level > 80)
            throw new InvalidOperationException(
                $"Order {orderId} blocked by fraud checker: {risk.Reason}");

        var paymentResult = await payment.ChargeAsync(order.CustomerEmail, order.Total);
        if (!paymentResult.Success)
        {
            await repo.SaveAsync(order with { Status = OrderStatus.Failed });
            return false;
        }

        await inventory.ReserveAsync(order.Id);
        await repo.SaveAsync(order with { Status = OrderStatus.Completed });
        return true;
    }
}
```

#### Требования к решению
- Использовать C# 12 / .NET 8: primary-конструкторы, `record` с `with`-выражениями, файл-скопед namespace, top-level statements в `Program.cs` не обязательны (это библиотека).
- Каждый `Setup` должен явно указывать возвращаемое значение (`Returns`/`ReturnsAsync`) или исключение (`Throws`/`ThrowsAsync`). Забытый `Returns` — самая частая причина `NullReferenceException` в SUT, потому что Loose-мок вернёт `default`.
- Для методов, возвращающих `Task` без значения (`SaveAsync`, `ReserveAsync`), используйте `.Returns(Task.CompletedTask)`, а не `ReturnsAsync` — так точнее передаётся намерение (урок прямо рекомендует это).
- Различать state- и behavior-проверки: state — это `Assert.True(result)`, `Assert.Equal(OrderStatus.Completed, saved.Status)`; behavior — это `_repo.Verify(...)`, `_payment.Verify(...)`, `_fraud.Protected().Verify(...)`.
- Не более одного-двух `Verify` на тест для значимых взаимодействий. Не верифицируйте тривиальные вызовы вроде `GetByIdAsync` в каждом тесте — это делает тест хрупким.
- Использовать `It.Is<T>(predicate)` вместо `It.IsAny<T>()` там, где аргумент важен для бизнес-логики (email и сумма при оплате). `It.IsAny` оставлять только для действительно «любых» аргументов.
- Защищённый `AssessRiskAsync` настраивать и верифицировать только через `Moq.Protected()` + `ItExpr` по имени.
- Для `FraudChecker` установить `CallBase = true`, иначе конкретный `CheckAsync` не делегирует в перехваченный `AssessRiskAsync`.
- Один из тестов должен использовать `MockBehavior.Strict` для всех зависимостей, чтобы продемонстрировать явный контракт по каждому вызову.
- Имена тестов — по схеме `Method_Scenario_Expected` (урок M15-L03 по AAA и именованию).

#### Тонкости и подводные камни
- **Не-виртуальные методы.** `FraudChecker.CheckAsync` — конкретный метод, а не `virtual`. Moq не может его перехватить. Поэтому обязательно `CallBase = true`: тогда Moq вызывает реальную реализацию `CheckAsync`, которая внутри зовёт абстрактный `AssessRiskAsync` — а его Moq уже перехватывает. Без `CallBase` `CheckAsync` вернёт `default(RiskScore)`, и фрауд-логика молча отработает с `null`/`Level == 0`.
- **Забытый `Returns`.** Если вы напишете `_payment.Setup(p => p.ChargeAsync(...))` без `ReturnsAsync`, Loose-мок вернёт `default(PaymentResult)`, то есть `PaymentResult(false, null)`. Тест «успешная оплата» тогда поедет по ветке отказа, и вы потратите время на поиск причины. Всегда явно задавайте возвращаемое значение.
- **`ReturnsAsync` vs `Returns(Task.CompletedTask)`.** Для `Task<T>` используйте `ReturnsAsync(value)`, для `Task` без значения — `.Returns(Task.CompletedTask)`. `ReturnsAsync` для `Task` не скомпилируется.
- **`It.IsAny` там, где важен аргумент.** Если в счастливом пути вы напишете `It.IsAny<decimal>()` для суммы оплаты, тест пройдёт, даже если SUT по ошибке передаст `0m`. Замените на `It.Is<decimal>(d => d == order.Total)` — это поймает реальный баг.
- **`Verify` после раннего `return`.** В тесте `Process_MissingOrder_ThrowsAndSkipsPayment` SUT бросает исключение до вызова оплаты. Если вы случайно поставите `Verify(..., Times.Once)` для `ChargeAsync`, тест упадёт — но это правильно: вы проверяете, что оплата НЕ произошла (`Times.Never`). Главное — не путать `Once` и `Never` в негативных ветках.
- **Strict и забытые настройки.** В Strict-режиме любой ненастроенный вызов выбрасывает `MockException` с сообщением `All invocations on the mock must have a corresponding setup`. Это удобно для отладки: вы сразу видите, какой вызов забыли. Но Strict делает тест многословнее — используйте его точечно.
- **`ItExpr` для protected.** Обычный `Setup(r => r.AssessRiskAsync(...))` не скомпилируется, потому что метод `protected` — компилятор не даст обратиться к нему из тестового класса. Поэтому только `mock.Protected().Setup<Task<RiskScore>>("AssessRiskAsync", ItExpr.IsAny<Order>())`. Имя метода — строкой, аргументы — через `ItExpr`.
- **`Raise` и события.** В базовой постановке событие не используется; оно вынесено в углубление. Но помните: `mock.Raise(r => r.EventName += null, args)` поднимает событие со стороны мока, как будто зависимость сама его инициировала.
- **Перебор с `Verify`.** Если в каждом тесте верифицировать и `GetByIdAsync`, и `SaveAsync`, и `ReserveAsync`, и `ChargeAsync`, и `AssessRiskAsync` — тест станет ломаться при любом рефакторинге (например, добавлении логирования через отдельную зависимость). Верифицируйте только то, что несёт бизнес-смысл.
- **Статические члены.** Не пытайтесь замокать `DateTime.Now` или `Guid.NewGuid()` через Moq — это не работает. Если в будущем вам понадобится время, вводите `IClock`.

#### Критерии приёмки
- [ ] Решение компилируется под .NET 8 (`dotnet build` без ошибок и предупреждений).
- [ ] Созданы два проекта: `src/Shop` (classlib) и `tests/Shop.Tests` (xunit), соединённые `reference`.
- [ ] Установлен пакет `Moq` в тестовый проект.
- [ ] Production-код из постановки скопирован без изменений.
- [ ] В классе `OrderProcessorTests` приватные моки объявлены с `MockBehavior.Loose` (кроме Strict-теста).
- [ ] Для `Mock<FraudChecker>` установлено `CallBase = true`.
- [ ] SUT создаётся в конструкторе тест-класса через `.Object` всех моков.
- [ ] Каждый `Setup` явно задаёт `Returns`/`ReturnsAsync`/`Throws`/`ThrowsAsync` или `Returns(Task.CompletedTask)`.
- [ ] Реализованы все шесть сценариев: HappyPath, MissingOrder, HighRisk, PaymentFails, PaymentThrows, StrictMock.
- [ ] В HappyPath для `ChargeAsync` использован `It.Is<T>(predicate)` по email и сумме (не `It.IsAny`).
- [ ] В HappyPath и PaymentFails заказ захвачен через `Callback<Order>` и проверен `Assert.Equal(...Status)`.
- [ ] Защищённый `AssessRiskAsync` настроен и верифицирован через `Moq.Protected()` + `ItExpr` по имени.
- [ ] В негативных ветках использованы `Times.Never` для `ChargeAsync` и/или `ReserveAsync`.
- [ ] В Strict-тесте все вызовы явно настроены, тест зелёный.
- [ ] `dotnet test` показывает 6+ зелёных тестов, 0 красных.
- [ ] В тестах нет `It.IsAny` там, где бизнес-логика зависит от конкретного аргумента.
- [ ] Не более двух `Verify` на тест (кроме случаев, где behavior-проверка критична).

#### Подсказки (без прямого ответа)
- Если `Mock<FraudChecker>` возвращает странные значения риска — проверьте `CallBase`. Конкретный `CheckAsync` без `CallBase` не вызывает перехваченный `AssessRiskAsync`.
- Для `Task` без значения нельзя использовать `ReturnsAsync`. Нужен `.Returns(Task.CompletedTask)` — это явная рекомендация урока.
- Чтобы проверить, что оплата НЕ произошла, используйте `_payment.Verify(p => p.ChargeAsync(It.IsAny<string>(), It.IsAny<decimal>()), Times.Never)`. Здесь `It.IsAny` уместен: вам важно отсутствие вызова вообще, а не конкретные аргументы.
- В Strict-тесте начните с копии HappyPath и поочерёдно добавляйте `Setup` для каждого вызова, пока `MockException` не исчезнет. Сообщение об ошибке подскажет, какой именно вызов не настроен.
- `Callback` должен идти в цепочке `Setup(...).Callback(...).Returns(...)` — порядок вызовов не критичен, но `Returns` обязан присутствовать, иначе Loose-мок вернёт `default`.
- Для `It.Is<decimal>(d => d == order.Total)` помните: сравнение `decimal` через `==` корректно. Для строк email — тоже `==`.

#### Эталонное решение (разбор)
```csharp
// tests/Shop.Tests/OrderProcessorTests.cs — C# 12 / .NET 8, xUnit + Moq
using System;
using System.Threading.Tasks;
using Moq;
using Moq.Protected;
using Shop;
using Xunit;

public class OrderProcessorTests
{
    // Loose — широкое заглушивание по умолчанию / Loose: broad stubbing by default
    private readonly Mock<IOrderRepository> _repo = new(MockBehavior.Loose);
    private readonly Mock<IPaymentGateway> _payment = new(MockBehavior.Loose);
    private readonly Mock<IInventoryService> _inventory = new(MockBehavior.Loose);
    // CallBase обязателен: конкретный CheckAsync должен делегировать в перехваченный AssessRiskAsync
    // CallBase is mandatory: concrete CheckAsync must delegate to the intercepted AssessRiskAsync
    private readonly Mock<FraudChecker> _fraud = new(MockBehavior.Loose) { CallBase = true };
    private readonly OrderProcessor _sut;

    public OrderProcessorTests() =>
        _sut = new OrderProcessor(_repo.Object, _payment.Object, _inventory.Object, _fraud.Object);

    private static Order Sample(int id = 1, decimal total = 100m) =>
        new(id, "buyer@example.com", total, OrderStatus.Pending);

    // Вспомогательная настройка низкого риска через Moq.Protected + ItExpr
    // Helper: configure low risk via Moq.Protected + ItExpr
    private void SetupLowRisk() =>
        _fraud.Protected()
              .Setup<Task<RiskScore>>("AssessRiskAsync", ItExpr.IsAny<Order>())
              .ReturnsAsync(new RiskScore(10, "low"));

    [Fact]
    public async Task Process_HappyPath_CompletesOrderAndReserves()
    {
        // Arrange
        var order = Sample();
        _repo.Setup(r => r.GetByIdAsync(1)).ReturnsAsync(order);
        SetupLowRisk();

        // It.Is<T>(predicate): важны конкретные email и сумма / specific email and amount matter
        _payment.Setup(p => p.ChargeAsync(
                    It.Is<string>(s => s == order.CustomerEmail),
                    It.Is<decimal>(d => d == order.Total)))
                .ReturnsAsync(new PaymentResult(true, "tx-001"));

        // Callback для захвата сохранённого заказа / capture saved order
        Order? saved = null;
        _repo.Setup(r => r.SaveAsync(It.IsAny<Order>()))
             .Callback<Order>(o => saved = o)
             .Returns(Task.CompletedTask);

        // Act
        var result = await _sut.ProcessAsync(1);

        // Assert (state)
        Assert.True(result);
        Assert.NotNull(saved);
        Assert.Equal(OrderStatus.Completed, saved!.Status);

        // Assert (behavior) — только значимые вызовы / only meaningful calls
        _inventory.Verify(i => i.ReserveAsync(1), Times.Once);
        _payment.Verify(p => p.ChargeAsync(order.CustomerEmail, order.Total), Times.Once);
        _fraud.Protected().Verify("AssessRiskAsync", Times.Once(), ItExpr.IsAny<Order>());
    }

    [Fact]
    public async Task Process_MissingOrder_ThrowsAndSkipsPayment()
    {
        _repo.Setup(r => r.GetByIdAsync(42)).ReturnsAsync((Order?)null);

        await Assert.ThrowsAsync<InvalidOperationException>(() => _sut.ProcessAsync(42));

        // Негативная behavior-проверка: оплата и резерв не должны вызываться
        // Negative behavior verification: payment and reservation must not happen
        _payment.Verify(p => p.ChargeAsync(It.IsAny<string>(), It.IsAny<decimal>()), Times.Never);
        _inventory.Verify(i => i.ReserveAsync(It.IsAny<int>()), Times.Never);
    }

    [Fact]
    public async Task Process_HighRisk_ThrowsBeforePayment()
    {
        var order = Sample();
        _repo.Setup(r => r.GetByIdAsync(1)).ReturnsAsync(order);

        _fraud.Protected()
              .Setup<Task<RiskScore>>("AssessRiskAsync", ItExpr.IsAny<Order>())
              .ReturnsAsync(new RiskScore(95, "stolen card"));

        await Assert.ThrowsAsync<InvalidOperationException>(() => _sut.ProcessAsync(1));

        _payment.Verify(p => p.ChargeAsync(It.IsAny<string>(), It.IsAny<decimal>()), Times.Never);
    }

    [Fact]
    public async Task Process_PaymentFails_MarksFailedAndSkipsInventory()
    {
        var order = Sample();
        _repo.Setup(r => r.GetByIdAsync(1)).ReturnsAsync(order);
        SetupLowRisk();
        _payment.Setup(p => p.ChargeAsync(It.IsAny<string>(), It.IsAny<decimal>()))
                .ReturnsAsync(new PaymentResult(false, ""));

        Order? saved = null;
        _repo.Setup(r => r.SaveAsync(It.IsAny<Order>()))
             .Callback<Order>(o => saved = o)
             .Returns(Task.CompletedTask);

        var result = await _sut.ProcessAsync(1);

        Assert.False(result);
        Assert.NotNull(saved);
        Assert.Equal(OrderStatus.Failed, saved!.Status);
        _inventory.Verify(i => i.ReserveAsync(It.IsAny<int>()), Times.Never);
    }

    [Fact]
    public async Task Process_PaymentThrows_PropagatesAndSkipsInventory()
    {
        var order = Sample();
        _repo.Setup(r => r.GetByIdAsync(1)).ReturnsAsync(order);
        SetupLowRisk();
        // ThrowsAsync — эмуляция исключения из шлюза / simulate gateway exception
        _payment.Setup(p => p.ChargeAsync(It.IsAny<string>(), It.IsAny<decimal>()))
                .ThrowsAsync(new InvalidOperationException("gateway down"));

        await Assert.ThrowsAsync<InvalidOperationException>(() => _sut.ProcessAsync(1));

        _inventory.Verify(i => i.ReserveAsync(It.IsAny<int>()), Times.Never);
        _repo.Verify(r => r.SaveAsync(It.IsAny<Order>()), Times.Never);
    }

    [Fact]
    public async Task Process_StrictMock_RejectsUnconfiguredCalls()
    {
        // Strict: каждый вызов должен быть явно настроен / every call must be configured
        var strictRepo = new Mock<IOrderRepository>(MockBehavior.Strict);
        var strictPayment = new Mock<IPaymentGateway>(MockBehavior.Strict);
        var strictInventory = new Mock<IInventoryService>(MockBehavior.Strict);
        var strictFraud = new Mock<FraudChecker>(MockBehavior.Strict) { CallBase = true };

        var order = Sample();
        strictRepo.Setup(r => r.GetByIdAsync(1)).ReturnsAsync(order);
        strictFraud.Protected()
                   .Setup<Task<RiskScore>>("AssessRiskAsync", ItExpr.IsAny<Order>())
                   .ReturnsAsync(new RiskScore(0, "ok"));
        strictPayment.Setup(p => p.ChargeAsync(order.CustomerEmail, order.Total))
                     .ReturnsAsync(new PaymentResult(true, "tx-strict"));
        strictInventory.Setup(i => i.ReserveAsync(1)).Returns(Task.CompletedTask);
        strictRepo.Setup(r => r.SaveAsync(It.IsAny<Order>())).Returns(Task.CompletedTask);

        var sut = new OrderProcessor(strictRepo.Object, strictPayment.Object,
                                     strictInventory.Object, strictFraud.Object);

        var ok = await sut.ProcessAsync(1);

        Assert.True(ok);
    }
}
```

**Разбор по строкам.** Поле `_fraud` объявлено с `CallBase = true` — это критично: `FraudChecker.CheckAsync` конкретный (не виртуальный), Moq его не перехватывает, но с `CallBase` вызывает реальную реализацию, которая делегирует в перехваченный абстрактный `AssessRiskAsync`. Без этой строки `CheckAsync` вернул бы `default(RiskScore)`, и фрауд-ветка никогда бы не сработала. Метод `SetupLowRisk()` инкапсулирует настройку защищённого члена через `Moq.Protected().Setup<Task<RiskScore>>("AssessRiskAsync", ItExpr.IsAny<Order>()).ReturnsAsync(...)` — именно так урок предписывает работать с `protected`-членами: имя строкой, аргументы через `ItExpr`, тип возврата через generics. В `HappyPath` для `ChargeAsync` использован `It.Is<string>` и `It.Is<decimal>` с предикатами, а не `It.IsAny` — это та самая best practice из урока: когда аргумент важен для бизнес-логики, строгий матчер ловит реальные баги (например, передачу `0m`). `Callback<Order>(o => saved = o)` перехватывает сохраняемый заказ до того, как `Returns(Task.CompletedTask)` завершит `Setup`; затем state-проверка `Assert.Equal(OrderStatus.Completed, saved.Status)` подтверждает, что SUT действительно перевёл заказ в завершённое состояние. Behavior-проверки ограничены тремя значимыми вызовами (`ReserveAsync`, `ChargeAsync`, `AssessRiskAsync`) — ровно столько, сколько несёт бизнес-смысл; `GetByIdAsync` и `SaveAsync` здесь не верифицируются, чтобы тест не стал хрупким. В `MissingOrder` ключевая тонкость — `ReturnsAsync((Order?)null)` симулирует отсутствие заказа, а `Times.Never` на `ChargeAsync` и `ReserveAsync` гарантирует, что ранняя `InvalidOperationException` действительно оборвала поток. В `PaymentFails` возвращается `PaymentResult(false, "")` через `ReturnsAsync` — и проверяется, что SUT пошёл по ветке отказа: сохранил статус `Failed` и не резервировал товар. `PaymentThrows` использует `ThrowsAsync` для эмуляции сбоя шлюза и `Times.Never` на `SaveAsync`, доказывая, что исключение пробросилось без побочных сохранений. Наконец, `StrictMock` повторяет счастливый путь в `MockBehavior.Strict`: каждый из шести вызовов (`GetByIdAsync`, `AssessRiskAsync`, `ChargeAsync`, `ReserveAsync`, `SaveAsync`) явно настроен — стоит убрать любую настройку, и Moq выбросит `MockException`, что и демонстрирует ценность строгого режима для отладки забытых контрактов. Применены концепции урока: трёхшаговый сценарий Moq, `It.Is` vs `It.IsAny`, `Callback`, `ThrowsAsync`, `Returns(Task.CompletedTask)` для `Task`, `Moq.Protected` + `ItExpr`, `MockBehavior.Strict`, ограниченный набор `Verify`.

#### Задания на углубление (бонус)
1. **События и `Raise`.** Добавьте в `IOrderRepository` событие `event EventHandler<Order>? OrderCompleted;`. Сделайте так, чтобы реальная реализация репозитория поднимала его при `SaveAsync` со статусом `Completed` (это уже не мок, а тест-двойник). Затем в отдельном тесте используйте `_repo.Raise(r => r.OrderCompleted += null, order)` напрямую на моке, чтобы убедиться, что подписчик (например, `OrderProcessor`, который вы расширите логированием через `ILogger`) реагирует. Это закрепит тему `Raise` из урока.
2. **`It.Is` с составным предикатом.** В HappyPath замените простой предикат `It.Is<decimal>(d => d == order.Total)` на составной: `It.Is<decimal>(d => d > 0 && d == order.Total && d <= 10000m)`. Объясните, почему это безопаснее. Добавьте отдельный тест, где SUT по ошибке передаёт отрицательную сумму, и покажите, что строгий матчер ловит баг, а `It.IsAny` — нет.
3. **`MockBehavior.Strict` для всех тестов.** Перепишите весь класс тестов в Strict-режиме. Сравните читаемость и хрупкость. Напишите короткое эссе (5–7 предложений) на русском: когда Strict оправдан, а когда — избыточен.
4. **`VerifyAll` и `VerifyNoOtherCalls`.** Добавьте в один из тестов `_repo.VerifyAll()` и `_payment.VerifyNoOtherCalls()`. Объясните, чем эти методы отличаются от точечных `Verify` и почему урок рекомендует точечные. Покажите, как `VerifyNoOtherCalls` ловит «лишний» вызов, который вы случайно добавили в SUT.

---

## Statement in English / Постановка на английском

#### Context & motivation
Imagine you maintain the checkout module of an online store built on C# 12 / .NET 8. The order-processing business logic (`OrderProcessor`) depends on four external collaborators: an order repository `IOrderRepository`, a payment gateway `IPaymentGateway`, an inventory reservation service `IInventoryService`, and an abstract fraud checker class `FraudChecker` that exposes a protected method `AssessRiskAsync`. In real life these dependencies talk to a database, an external bank HTTP API, and a third-party anti-fraud service — none of which you can call from a unit test. Your job is to isolate `OrderProcessor` from them using Moq.

The motivation for this exact configuration is that it naturally forces you to reach for the full arsenal from the lesson: asynchronous `ReturnsAsync` / `ThrowsAsync` for `Task<T>`-returning methods, `It.Is<T>(predicate)` to assert the exact amount and email during payment, `Callback` to capture the saved order and then state-assert its status, `Times.Never` for the negative branches (payment must not happen when the order is missing or the fraud checker blocked it), `Moq.Protected` + `ItExpr` for the protected `AssessRiskAsync`, and `MockBehavior.Strict` for one of the tests so you can see how the strict mode catches forgotten setups.

It is essential to understand the difference between state verification and behavior verification. State verification answers "what is the result of the SUT's work?" — for example, did `ProcessAsync` return `true` and was the order saved with status `Completed`? Behavior verification answers "what did the SUT do with the dependency?" — did it call `ReserveAsync` exactly once, did it avoid calling `ChargeAsync` when the order was blocked by fraud? A good test combines both approaches, but does not overload itself with extra `Verify` calls: every additional verifier makes the test brittle under harmless refactoring.

#### What to do step by step
1. Create two projects in a single solution. Run:
   - `dotnet new sln -n Shop`
   - `dotnet new classlib -n Shop -o src/Shop --framework net8.0`
   - `dotnet new xunit -n Shop.Tests -o tests/Shop.Tests --framework net8.0`
   - `dotnet sln add src/Shop tests/Shop.Tests`
   - `dotnet add tests/Shop.Tests reference src/Shop`
   - `dotnet add tests/Shop.Tests package Moq`
   - `dotnet add tests/Shop.Tests package FluentAssertions` (optional)
2. In `src/Shop`, create `OrderProcessor.cs` and copy the production code from the "Production code" section below. It contains the domain types (`Order`, `OrderStatus`, `PaymentResult`, `RiskScore`), the dependency interfaces, the abstract class `FraudChecker`, and the `OrderProcessor` itself. This code is given to you ready-made — do not change it; your job is the tests.
3. In `tests/Shop.Tests`, create `OrderProcessorTests.cs`. Implement the `OrderProcessorTests` class with private mock fields `Mock<IOrderRepository>`, `Mock<IPaymentGateway>`, `Mock<IInventoryService>`, and `Mock<FraudChecker>`. Note: for `FraudChecker` you need `CallBase = true`, because the public method `CheckAsync` is concrete (non-virtual), and without `CallBase` it will return `default` instead of delegating to `AssessRiskAsync`.
4. In the test-class constructor, construct the `OrderProcessor` (the SUT), passing `.Object` of every mock. The default mode is `MockBehavior.Loose`, as the lesson recommends for the majority of scenarios.
5. Write tests for the following scenarios (at least six). Each test must have explicit Arrange / Act / Assert sections:
   - `Process_HappyPath_CompletesOrderAndReserves`: order found, low risk, payment succeeds → method returns `true`, order is saved with status `Completed`, `ReserveAsync` is called exactly once, `ChargeAsync` is called exactly once with the expected email and amount.
   - `Process_MissingOrder_ThrowsAndSkipsPayment`: `GetByIdAsync` returns `null` → `InvalidOperationException`, and `ChargeAsync` / `ReserveAsync` are never called (`Times.Never`).
   - `Process_HighRisk_ThrowsBeforePayment`: `RiskScore.Level > 80` → `InvalidOperationException`, `ChargeAsync` is never called.
   - `Process_PaymentFails_MarksFailedAndSkipsInventory`: `PaymentResult.Success == false` → method returns `false`, order is saved with status `Failed`, `ReserveAsync` is never called.
   - `Process_PaymentThrows_PropagatesAndSkipsInventory`: `ChargeAsync` throws `InvalidOperationException` via `ThrowsAsync` → the exception propagates outward, `ReserveAsync` and `SaveAsync` are never called.
   - `Process_StrictMock_RejectsUnconfiguredCalls`: the same happy path, but every mock is created with `MockBehavior.Strict` — every call must be explicitly configured, otherwise Moq throws `MockException`.
6. For the protected method `FraudChecker.AssessRiskAsync`, use `Moq.Protected()` + `ItExpr.IsAny<Order>()` with `ReturnsAsync` and `Verify` by the string name `"AssessRiskAsync"`. A normal expression-based `Setup` is unavailable for a `protected` member — this is a key lesson takeaway.
7. To capture the saved order in the happy path and in the payment-failure branch, use `Callback<Order>(o => saved = o)` before `Returns(Task.CompletedTask)`. Then state-assert `saved.Status`.
8. Run the tests: `dotnet test`. Expected output: every test green. If Moq throws `Expression is not a method invocation` or `All invocations on the mock must have a corresponding setup`, you forgot a setup, or you used `It.IsAny` instead of a concrete argument under Strict mode.

#### Production code (do not modify)
```csharp
// src/Shop/OrderProcessor.cs — C# 12 / .NET 8
using System;
using System.Threading.Tasks;

namespace Shop;

public enum OrderStatus { Pending, Paid, Completed, Failed }

public sealed record Order(int Id, string CustomerEmail, decimal Total, OrderStatus Status);

public sealed record PaymentResult(bool Success, string TransactionId);

public sealed record RiskScore(int Level, string Reason);

public interface IOrderRepository
{
    Task<Order?> GetByIdAsync(int id);
    Task SaveAsync(Order order);
}

public interface IPaymentGateway
{
    Task<PaymentResult> ChargeAsync(string email, decimal amount);
}

public interface IInventoryService
{
    Task ReserveAsync(int orderId);
}

// Abstract class with a protected internal abstract method.
// Setting it up and verifying it requires Moq.Protected + ItExpr.
public abstract class FraudChecker
{
    protected internal abstract Task<RiskScore> AssessRiskAsync(Order order);

    public Task<RiskScore> CheckAsync(Order order) => AssessRiskAsync(order);
}

public sealed class OrderProcessor(
    IOrderRepository repo,
    IPaymentGateway payment,
    IInventoryService inventory,
    FraudChecker fraud)
{
    public async Task<bool> ProcessAsync(int orderId)
    {
        var order = await repo.GetByIdAsync(orderId)
            ?? throw new InvalidOperationException($"Order {orderId} not found");

        var risk = await fraud.CheckAsync(order);
        if (risk.Level > 80)
            throw new InvalidOperationException(
                $"Order {orderId} blocked by fraud checker: {risk.Reason}");

        var paymentResult = await payment.ChargeAsync(order.CustomerEmail, order.Total);
        if (!paymentResult.Success)
        {
            await repo.SaveAsync(order with { Status = OrderStatus.Failed });
            return false;
        }

        await inventory.ReserveAsync(order.Id);
        await repo.SaveAsync(order with { Status = OrderStatus.Completed });
        return true;
    }
}
```

#### Requirements
- Target C# 12 / .NET 8: primary constructors, `record` with `with`-expressions, file-scoped namespaces. Top-level `Program.cs` is not required (this is a library).
- Every `Setup` must explicitly state a return value (`Returns` / `ReturnsAsync`) or an exception (`Throws` / `ThrowsAsync`). A forgotten `Returns` is the most common cause of `NullReferenceException` inside the SUT, because a Loose mock returns `default`.
- For methods returning `Task` without a value (`SaveAsync`, `ReserveAsync`), use `.Returns(Task.CompletedTask)` rather than `ReturnsAsync` — this conveys intent more precisely (the lesson explicitly recommends it).
- Distinguish state verification from behavior verification: state is `Assert.True(result)`, `Assert.Equal(OrderStatus.Completed, saved.Status)`; behavior is `_repo.Verify(...)`, `_payment.Verify(...)`, `_fraud.Protected().Verify(...)`.
- No more than one or two `Verify` calls per test for meaningful interactions. Do not verify trivial calls such as `GetByIdAsync` in every test — it makes the test brittle.
- Prefer `It.Is<T>(predicate)` over `It.IsAny<T>()` wherever the argument matters to business logic (email and amount during payment). Keep `It.IsAny` only for genuinely "any" arguments.
- Set up and verify the protected `AssessRiskAsync` exclusively through `Moq.Protected()` + `ItExpr` by name.
- Set `CallBase = true` on `Mock<FraudChecker>`, otherwise the concrete `CheckAsync` will not delegate to the intercepted `AssessRiskAsync`.
- One test must use `MockBehavior.Strict` for every dependency, to demonstrate an explicit per-call contract.
- Test names follow the `Method_Scenario_Expected` convention (lesson M15-L03 on AAA and naming).

#### Pitfalls
- **Non-virtual methods.** `FraudChecker.CheckAsync` is a concrete method, not `virtual`. Moq cannot intercept it. That is why `CallBase = true` is mandatory: Moq then calls the real `CheckAsync`, which internally invokes the abstract `AssessRiskAsync` — and that one Moq does intercept. Without `CallBase`, `CheckAsync` returns `default(RiskScore)`, so the fraud logic silently operates on `Level == 0`.
- **Forgotten `Returns`.** If you write `_payment.Setup(p => p.ChargeAsync(...))` without `ReturnsAsync`, the Loose mock returns `default(PaymentResult)`, i.e. `PaymentResult(false, null)`. The "successful payment" test then takes the failure branch, and you will waste time hunting the cause. Always set an explicit return.
- **`ReturnsAsync` vs `Returns(Task.CompletedTask)`.** Use `ReturnsAsync(value)` for `Task<T>`, and `.Returns(Task.CompletedTask)` for a value-less `Task`. `ReturnsAsync` on a plain `Task` will not compile.
- **`It.IsAny` where an argument matters.** If in the happy path you write `It.IsAny<decimal>()` for the payment amount, the test passes even if the SUT mistakenly passes `0m`. Replace it with `It.Is<decimal>(d => d == order.Total)` to catch a real bug.
- **`Verify` after an early return.** In `Process_MissingOrder_ThrowsAndSkipsPayment` the SUT throws before reaching payment. If you accidentally put `Verify(..., Times.Once)` on `ChargeAsync`, the test will fail — which is correct: you are asserting that payment did NOT happen (`Times.Never`). The trap is mixing up `Once` and `Never` in negative branches.
- **Strict and forgotten setups.** In Strict mode every unconfigured call throws `MockException` with the message `All invocations on the mock must have a corresponding setup`. This is great for debugging: you immediately see which call you forgot. But Strict makes tests more verbose — use it surgically.
- **`ItExpr` for protected members.** A normal `Setup(r => r.AssessRiskAsync(...))` will not compile, because the method is `protected` and the compiler forbids accessing it from the test class. The only path is `mock.Protected().Setup<Task<RiskScore>>("AssessRiskAsync", ItExpr.IsAny<Order>())`. The method name is a string; the arguments come through `ItExpr`.
- **`Raise` and events.** The base statement does not use events; that is moved to the "going deeper" section. Remember: `mock.Raise(r => r.EventName += null, args)` raises the event from the mock side, as if the dependency had fired it.
- **Over-verification.** If every test verifies `GetByIdAsync`, `SaveAsync`, `ReserveAsync`, `ChargeAsync`, and `AssessRiskAsync`, the test breaks under any refactor (say, adding logging through a new dependency). Verify only what carries business meaning.
- **Static members.** Do not try to mock `DateTime.Now` or `Guid.NewGuid()` with Moq — it does not work. If you need time in the future, introduce an `IClock` abstraction.

#### Acceptance criteria
- [ ] The solution compiles on .NET 8 (`dotnet build` with no errors or warnings).
- [ ] Two projects exist: `src/Shop` (classlib) and `tests/Shop.Tests` (xunit), linked by `reference`.
- [ ] The `Moq` package is installed in the test project.
- [ ] The production code from the statement is copied unchanged.
- [ ] In `OrderProcessorTests`, the private mocks are declared with `MockBehavior.Loose` (except the Strict test).
- [ ] `Mock<FraudChecker>` has `CallBase = true`.
- [ ] The SUT is constructed in the test-class constructor via `.Object` of every mock.
- [ ] Every `Setup` explicitly states `Returns` / `ReturnsAsync` / `Throws` / `ThrowsAsync` or `Returns(Task.CompletedTask)`.
- [ ] All six scenarios are implemented: HappyPath, MissingOrder, HighRisk, PaymentFails, PaymentThrows, StrictMock.
- [ ] In HappyPath, `It.Is<T>(predicate)` is used for email and amount on `ChargeAsync` (not `It.IsAny`).
- [ ] In HappyPath and PaymentFails, the order is captured via `Callback<Order>` and asserted with `Assert.Equal(...Status)`.
- [ ] The protected `AssessRiskAsync` is set up and verified through `Moq.Protected()` + `ItExpr` by name.
- [ ] Negative branches use `Times.Never` for `ChargeAsync` and/or `ReserveAsync`.
- [ ] In the Strict test, every call is explicitly configured and the test is green.
- [ ] `dotnet test` shows 6+ green tests, 0 red.
- [ ] No `It.IsAny` appears where business logic depends on a specific argument.
- [ ] At most two `Verify` calls per test (except where behavior verification is critical).

#### Hints (no direct answer)
- If `Mock<FraudChecker>` returns strange risk values — check `CallBase`. The concrete `CheckAsync` without `CallBase` does not invoke the intercepted `AssessRiskAsync`.
- For a value-less `Task`, you cannot use `ReturnsAsync`. Use `.Returns(Task.CompletedTask)` — an explicit lesson recommendation.
- To assert that payment did NOT happen, use `_payment.Verify(p => p.ChargeAsync(It.IsAny<string>(), It.IsAny<decimal>()), Times.Never)`. Here `It.IsAny` is appropriate: you care about the absence of any call, not the arguments.
- In the Strict test, start from a copy of HappyPath and add `Setup` for each call one by one until `MockException` disappears. The error message tells you which call is unconfigured.
- `Callback` goes in the chain `Setup(...).Callback(...).Returns(...)` — order is not critical, but `Returns` must be present, otherwise the Loose mock returns `default`.
- For `It.Is<decimal>(d => d == order.Total)`, remember that `==` on `decimal` is correct, and `==` on the email string works too.

#### Reference solution walk-through
```csharp
// tests/Shop.Tests/OrderProcessorTests.cs — C# 12 / .NET 8, xUnit + Moq
using System;
using System.Threading.Tasks;
using Moq;
using Moq.Protected;
using Shop;
using Xunit;

public class OrderProcessorTests
{
    // Loose: broad stubbing by default
    private readonly Mock<IOrderRepository> _repo = new(MockBehavior.Loose);
    private readonly Mock<IPaymentGateway> _payment = new(MockBehavior.Loose);
    private readonly Mock<IInventoryService> _inventory = new(MockBehavior.Loose);
    // CallBase is mandatory: the concrete CheckAsync must delegate to the intercepted AssessRiskAsync
    private readonly Mock<FraudChecker> _fraud = new(MockBehavior.Loose) { CallBase = true };
    private readonly OrderProcessor _sut;

    public OrderProcessorTests() =>
        _sut = new OrderProcessor(_repo.Object, _payment.Object, _inventory.Object, _fraud.Object);

    private static Order Sample(int id = 1, decimal total = 100m) =>
        new(id, "buyer@example.com", total, OrderStatus.Pending);

    // Helper: configure low risk via Moq.Protected + ItExpr
    private void SetupLowRisk() =>
        _fraud.Protected()
              .Setup<Task<RiskScore>>("AssessRiskAsync", ItExpr.IsAny<Order>())
              .ReturnsAsync(new RiskScore(10, "low"));

    [Fact]
    public async Task Process_HappyPath_CompletesOrderAndReserves()
    {
        // Arrange
        var order = Sample();
        _repo.Setup(r => r.GetByIdAsync(1)).ReturnsAsync(order);
        SetupLowRisk();

        // It.Is<T>(predicate): the exact email and amount matter
        _payment.Setup(p => p.ChargeAsync(
                    It.Is<string>(s => s == order.CustomerEmail),
                    It.Is<decimal>(d => d == order.Total)))
                .ReturnsAsync(new PaymentResult(true, "tx-001"));

        // Callback captures the saved order
        Order? saved = null;
        _repo.Setup(r => r.SaveAsync(It.IsAny<Order>()))
             .Callback<Order>(o => saved = o)
             .Returns(Task.CompletedTask);

        // Act
        var result = await _sut.ProcessAsync(1);

        // Assert (state)
        Assert.True(result);
        Assert.NotNull(saved);
        Assert.Equal(OrderStatus.Completed, saved!.Status);

        // Assert (behavior) — only meaningful calls
        _inventory.Verify(i => i.ReserveAsync(1), Times.Once);
        _payment.Verify(p => p.ChargeAsync(order.CustomerEmail, order.Total), Times.Once);
        _fraud.Protected().Verify("AssessRiskAsync", Times.Once(), ItExpr.IsAny<Order>());
    }

    [Fact]
    public async Task Process_MissingOrder_ThrowsAndSkipsPayment()
    {
        _repo.Setup(r => r.GetByIdAsync(42)).ReturnsAsync((Order?)null);

        await Assert.ThrowsAsync<InvalidOperationException>(() => _sut.ProcessAsync(42));

        // Negative behavior verification: payment and reservation must not happen
        _payment.Verify(p => p.ChargeAsync(It.IsAny<string>(), It.IsAny<decimal>()), Times.Never);
        _inventory.Verify(i => i.ReserveAsync(It.IsAny<int>()), Times.Never);
    }

    [Fact]
    public async Task Process_HighRisk_ThrowsBeforePayment()
    {
        var order = Sample();
        _repo.Setup(r => r.GetByIdAsync(1)).ReturnsAsync(order);

        _fraud.Protected()
              .Setup<Task<RiskScore>>("AssessRiskAsync", ItExpr.IsAny<Order>())
              .ReturnsAsync(new RiskScore(95, "stolen card"));

        await Assert.ThrowsAsync<InvalidOperationException>(() => _sut.ProcessAsync(1));

        _payment.Verify(p => p.ChargeAsync(It.IsAny<string>(), It.IsAny<decimal>()), Times.Never);
    }

    [Fact]
    public async Task Process_PaymentFails_MarksFailedAndSkipsInventory()
    {
        var order = Sample();
        _repo.Setup(r => r.GetByIdAsync(1)).ReturnsAsync(order);
        SetupLowRisk();
        _payment.Setup(p => p.ChargeAsync(It.IsAny<string>(), It.IsAny<decimal>()))
                .ReturnsAsync(new PaymentResult(false, ""));

        Order? saved = null;
        _repo.Setup(r => r.SaveAsync(It.IsAny<Order>()))
             .Callback<Order>(o => saved = o)
             .Returns(Task.CompletedTask);

        var result = await _sut.ProcessAsync(1);

        Assert.False(result);
        Assert.NotNull(saved);
        Assert.Equal(OrderStatus.Failed, saved!.Status);
        _inventory.Verify(i => i.ReserveAsync(It.IsAny<int>()), Times.Never);
    }

    [Fact]
    public async Task Process_PaymentThrows_PropagatesAndSkipsInventory()
    {
        var order = Sample();
        _repo.Setup(r => r.GetByIdAsync(1)).ReturnsAsync(order);
        SetupLowRisk();
        // ThrowsAsync — simulate a gateway exception
        _payment.Setup(p => p.ChargeAsync(It.IsAny<string>(), It.IsAny<decimal>()))
                .ThrowsAsync(new InvalidOperationException("gateway down"));

        await Assert.ThrowsAsync<InvalidOperationException>(() => _sut.ProcessAsync(1));

        _inventory.Verify(i => i.ReserveAsync(It.IsAny<int>()), Times.Never);
        _repo.Verify(r => r.SaveAsync(It.IsAny<Order>()), Times.Never);
    }

    [Fact]
    public async Task Process_StrictMock_RejectsUnconfiguredCalls()
    {
        // Strict: every call must be configured explicitly
        var strictRepo = new Mock<IOrderRepository>(MockBehavior.Strict);
        var strictPayment = new Mock<IPaymentGateway>(MockBehavior.Strict);
        var strictInventory = new Mock<IInventoryService>(MockBehavior.Strict);
        var strictFraud = new Mock<FraudChecker>(MockBehavior.Strict) { CallBase = true };

        var order = Sample();
        strictRepo.Setup(r => r.GetByIdAsync(1)).ReturnsAsync(order);
        strictFraud.Protected()
                   .Setup<Task<RiskScore>>("AssessRiskAsync", ItExpr.IsAny<Order>())
                   .ReturnsAsync(new RiskScore(0, "ok"));
        strictPayment.Setup(p => p.ChargeAsync(order.CustomerEmail, order.Total))
                     .ReturnsAsync(new PaymentResult(true, "tx-strict"));
        strictInventory.Setup(i => i.ReserveAsync(1)).Returns(Task.CompletedTask);
        strictRepo.Setup(r => r.SaveAsync(It.IsAny<Order>())).Returns(Task.CompletedTask);

        var sut = new OrderProcessor(strictRepo.Object, strictPayment.Object,
                                     strictInventory.Object, strictFraud.Object);

        var ok = await sut.ProcessAsync(1);

        Assert.True(ok);
    }
}
```

**Line-by-line walk-through.** The `_fraud` field is declared with `CallBase = true` — this is critical: `FraudChecker.CheckAsync` is concrete (non-virtual), so Moq cannot intercept it, but with `CallBase` it invokes the real implementation, which delegates to the intercepted abstract `AssessRiskAsync`. Without that line, `CheckAsync` would return `default(RiskScore)` and the fraud branch would never fire. The `SetupLowRisk()` helper encapsulates the protected-member setup via `Moq.Protected().Setup<Task<RiskScore>>("AssessRiskAsync", ItExpr.IsAny<Order>()).ReturnsAsync(...)` — exactly how the lesson prescribes working with `protected` members: name as a string, arguments through `ItExpr`, return type through generics. In `HappyPath`, `ChargeAsync` uses `It.Is<string>` and `It.Is<decimal>` with predicates rather than `It.IsAny` — this is the lesson's best practice: when an argument matters to business logic, a strict matcher catches real bugs (such as passing `0m`). `Callback<Order>(o => saved = o)` captures the saved order before `Returns(Task.CompletedTask)` finishes the `Setup`; the subsequent state assertion `Assert.Equal(OrderStatus.Completed, saved.Status)` confirms the SUT actually moved the order into the completed state. Behavior verifications are limited to three meaningful calls (`ReserveAsync`, `ChargeAsync`, `AssessRiskAsync`) — just enough to carry business meaning; `GetByIdAsync` and `SaveAsync` are not verified here, to keep the test non-brittle. In `MissingOrder`, the key subtlety is `ReturnsAsync((User?)null)` simulating the absence of the order, while `Times.Never` on `ChargeAsync` and `ReserveAsync` proves the early `InvalidOperationException` actually short-circuited the flow. In `PaymentFails`, `PaymentResult(false, "")` is returned via `ReturnsAsync`, and the test asserts that the SUT took the failure branch: saved status `Failed` and did not reserve inventory. `PaymentThrows` uses `ThrowsAsync` to simulate a gateway failure and `Times.Never` on `SaveAsync`, demonstrating that the exception propagated without side-effecting saves. Finally, `StrictMock` replays the happy path under `MockBehavior.Strict`: all six calls (`GetByIdAsync`, `AssessRiskAsync`, `ChargeAsync`, `ReserveAsync`, `SaveAsync`) are explicitly configured — remove any setup and Moq throws `MockException`, which is exactly the value of strict mode for debugging forgotten contracts. Lesson concepts applied: the three-step Moq flow, `It.Is` vs `It.IsAny`, `Callback`, `ThrowsAsync`, `Returns(Task.CompletedTask)` for `Task`, `Moq.Protected` + `ItExpr`, `MockBehavior.Strict`, and a bounded set of `Verify` calls.

#### Going deeper (bonus)
1. **Events and `Raise`.** Add an event `event EventHandler<Order>? OrderCompleted;` to `IOrderRepository`. Make the real repository implementation raise it on `SaveAsync` with status `Completed` (this is now a hand-written double, not a mock). Then, in a separate test, use `_repo.Raise(r => r.OrderCompleted += null, order)` directly on the mock to confirm that a subscriber (for example, an `OrderProcessor` you extend with `ILogger` logging) reacts. This cements the `Raise` topic from the lesson.
2. **`It.Is` with a compound predicate.** In `HappyPath`, replace the simple `It.Is<decimal>(d => d == order.Total)` with a compound one: `It.Is<decimal>(d => d > 0 && d == order.Total && d <= 10000m)`. Explain why this is safer. Add a separate test where the SUT mistakenly passes a negative amount and show that the strict matcher catches the bug while `It.IsAny` does not.
3. **`MockBehavior.Strict` across all tests.** Rewrite the entire test class in Strict mode. Compare readability and brittleness. Write a short essay (5–7 sentences) in English: when Strict is justified and when it is overkill.
4. **`VerifyAll` and `VerifyNoOtherCalls`.** Add `_repo.VerifyAll()` and `_payment.VerifyNoOtherCalls()` to one of the tests. Explain how these differ from targeted `Verify` calls and why the lesson recommends the targeted form. Show how `VerifyNoOtherCalls` catches an "extra" call that you accidentally introduced into the SUT.

---

#### Чек-лист сдачи / Submission checklist
- [ ] (RU) Решение компилируется под .NET 8 без предупреждений.
- [ ] (RU) Созданы проекты `src/Shop` и `tests/Shop.Tests`, установлен `Moq`.
- [ ] (RU) Production-код скопирован без изменений.
- [ ] (RU) Моки объявлены в Loose, `FraudChecker` с `CallBase = true`.
- [ ] (RU) Все шесть сценариев реализованы и зелёные.
- [ ] (RU) Защищённый метод настроен через `Moq.Protected` + `ItExpr`.
- [ ] (RU) В HappyPath использован `It.Is` для email и суммы, `Callback` для захвата.
- [ ] (RU) В негативных ветках — `Times.Never`.
- [ ] (RU) Strict-тест явно настраивает каждый вызов.
- [ ] (EN) Solution compiles on .NET 8 with no warnings.
- [ ] (EN) Projects `src/Shop` and `tests/Shop.Tests` created, `Moq` installed.
- [ ] (EN) Production code copied unchanged.
- [ ] (EN) Mocks declared in Loose, `FraudChecker` with `CallBase = true`.
- [ ] (EN) All six scenarios implemented and green.
- [ ] (EN) Protected member configured via `Moq.Protected` + `ItExpr`.
- [ ] (EN) HappyPath uses `It.Is` for email and amount, `Callback` for capture.
- [ ] (EN) Negative branches use `Times.Never`.
- [ ] (EN) Strict test explicitly configures every call.

#### Ресурсы / Resources
- [Moq Quickstart (GitHub Wiki) — https://github.com/Moq/moq/wiki/Quickstart](https://github.com/Moq/moq/wiki/Quickstart)
- [Moq API Reference — https://moq.github.io/moq4/](https://moq.github.io/moq4/)
- [Martin Fowler — Mocks Aren't Stubs — https://martinfowler.com/articles/mocksArentStubs.html](https://martinfowler.com/articles/mocksArentStubs.html)
- [Microsoft Learn — Unit testing best practices — https://learn.microsoft.com/dotnet/core/testing/](https://learn.microsoft.com/dotnet/core/testing/)
- [Microsoft Learn — Introduction to unit testing with Moq — https://learn.microsoft.com/dotnet/core/testing/unit-testing-with-moq](https://learn.microsoft.com/dotnet/core/testing/)
- [xUnit documentation — https://xunit.net/docs/getting-started/netcore](https://xunit.net/docs/getting-started/netcore)
