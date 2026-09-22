---
[← К уроку M15-L02](lesson-M15-L02-xunit-facts-theories.md) | [⬆ К модулю M15](../README.md) | [Следующее ДЗ →](homework-M15-L03-aaa-naming-assert.md)
---

### Домашнее задание M15-L02: xUnit: факты, теории, InlineData/MemberData / Homework M15-L02: xUnit: facts, theories, InlineData/MemberData

**Урок / Lesson:** M15-L02
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 3/5
**Цель / Goal:** (RU) Освоить три способа подачи данных в параметризованные тесты xUnit — `[InlineData]`, `[MemberData]`, `[ClassData]` — и научиться выбирать между ними, опираясь на природу тестовых данных; закрепить жизненный цикл тестового класса xUnit (конструктор как `SetUp`, `IDisposable` как `TearDown`, `IClassFixture<T>` для тяжёлой инициализации); научиться писать детерминированные, независимые тесты на C# 12 / .NET 8. (EN) Master the three ways of feeding data into parameterised xUnit tests — `[InlineData]`, `[MemberData]`, `[ClassData]` — and learn to choose between them based on the nature of the test data; internalise the xUnit test-class lifecycle (constructor as `SetUp`, `IDisposable` as `TearDown`, `IClassFixture<T>` for heavy setup); write deterministic, independent tests on C# 12 / .NET 8.

#### Связь с уроком / Connection to the lesson
(RU) Урок вводит столпы xUnit — `[Fact]` и `[Theory]` — и три провайдера данных, а также жизненный цикл класса и fixture-механику. В этом ДЗ вы применяете всё это на практике к нетривиальному алгоритму с ветвлениями и граничными условиями, где приходится комбинировать все источники данных и осознанно выбирать между ними. (EN) The lesson introduces the xUnit pillars — `[Fact]` and `[Theory]` — together with three data providers and the class lifecycle plus fixtures. In this homework you apply all of it in practice to a non-trivial algorithm with branches and boundary conditions, where you have to combine every data source and make a deliberate choice between them.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация
Представьте, что вы присоединились к команде, которая разрабатывает библиотеку учёта скидок для интернет-магазина. В модуле `Pricing` уже есть класс `DiscountEngine`, который на вход принимает корзину товаров и правило скидки, а на выходе возвращает итоговую цену и размер применённой скидки. Бизнес-логика нетривиальна: существуют пороги суммы, разные типы клиентов (обычный, постоянный, VIP), сезонные коэффициенты и защита от отрицательных значений. Код уже написан, но покрытие тестами близко к нулю, а баг-репорты от QA приходят каждый день.

Ваша задача — выстроить слой модульных тестов на xUnit так, чтобы каждый публичный сценарий класса `DiscountEngine` был покрыт параметризованными тестами с грамотно выбранными источниками данных. Команда следует best practices из урока M15-L02: один тест — одно логическое утверждение, имя по схеме `Method_Scenario_ExpectedResult`, константы идут в `[InlineData]`, вычисляемые и сложные объекты — в `[MemberData]` и `[ClassData]`, тяжёлая инициализация выносится в `IClassFixture<T>`, а освобождаемые ресурсы — в `IDisposable`. Это не учебная игрушка: QA требует, чтобы новые тесты ловили хотя бы три известных бага в текущей реализации и не падали на валидных кейсах. Вам предстоит не просто «покрыть строки», а показать, что выбор между `[Fact]`, `[Theory] + [InlineData]`, `[Theory] + [MemberData]` и `[Theory] + [ClassData]` вы делаете осознанно, исходя из природы данных и требований к читаемости набора.

#### Что нужно сделать (пошагово)
1. Создайте решение и два проекта. Из папки `C:\projects\course` выполните `dotnet new sln -n Pricing.Homework` в папке `modules/M15/homework/M15-L02`. Затем создайте библиотеку классов: `dotnet new classlib -n Pricing.Engine -f net8.0` и тестовый проект: `dotnet new xunit -n Pricing.Engine.Tests -f net8.0`. Добавьте оба проекта в решение: `dotnet sln add Pricing.Engine/*.csproj Pricing.Engine.Tests/*.csproj`. В тестовом проекте добавьте ссылку: `dotnet add Pricing.Engine.Tests reference Pricing.Engine`. Проверьте, что `dotnet test` из корня решения запускается и показывает ноль тестов без ошибок сборки.
2. В `Pricing.Engine` создайте типы. Опишите `record CustomerType(string Name, decimal DiscountFraction)` или используйте перечисление `CustomerKind { Regular, Loyal, Vip }`. Опишите `record CartLine(string Sku, decimal UnitPrice, int Quantity)`. Опишите `record DiscountResult(decimal FinalPrice, decimal Saved)`. Реализуйте класс `DiscountEngine` с методом `Apply(CartLine[] lines, CustomerKind kind, decimal seasonalFactor)`, который: суммирует `UnitPrice * Quantity`, применяет базовую скидку по типу клиента (Regular 0, Loyal 0.05, Vip 0.12), умножает на сезонный коэффициент так, чтобы финальная цена была `subtotal * (1 - discount) * seasonalFactor`, возвращает `DiscountResult`, и выбрасывает `ArgumentException`, если `seasonalFactor <= 0` или `seasonalFactor > 2`.
3. Намеренно внесите три бага, которые должны поймать тесты. Во-первых, базовая скидка Vip должна округляться вниз через `Math.Floor` до двух знаков — это создаёт проблему с плавающей точкой. Во-вторых, защита от отрицательного количества отсутствует: `Quantity < 0` должно выбрасывать `ArgumentException`, но текущий код молча суммирует. В-третьих, для пустой корзины метод возвращает `null` вместо `DiscountResult(0, 0)`. Зафиксируйте эти баги в комментарии, чтобы при ревью было видно, какие тесты их ловят.
4. В `Pricing.Engine.Tests` напишите тесты по схеме урока. Создайте класс `DiscountEngineFactTests` с минимум одним `[Fact]` для инварианта «пустая корзина возвращает нулевой результат». Создайте `DiscountEngineInlineTests` с `[Theory] + [InlineData]` для тривиальных констант: малые суммы, нулевые коэффициенты сезонности (там, где допустимо), границы `CustomerKind`. Создайте `DiscountEngineMemberDataTests`, где `MemberData` берётся из `public static IEnumerable<object[]>` свойства и включает вычисляемые значения, например subtotal, посчитанный через `Enumerable.Sum`. Создайте `DiscountEngineClassDataTests` с `TheoryData<CartLine[], CustomerKind, decimal, decimal, decimal>` для сложных объектов корзины. Каждый тест именуйте по `Method_Scenario_ExpectedResult`.
5. Продемонстрируйте жизненный цикл. В одном из тестовых классов реализуйте `IDisposable`, чтобы показать teardown «после каждого теста»: например, логируйте вызов в `Console.WriteLine` или сбрасывайте статический счётчик. В другом классе реализуйте `IClassFixture<SharedCatalogFixture>`, где `SharedCatalogFixture` загружает «каталог» (можно просто список SKU в памяти) один раз на весь класс и реализует `IDisposable`. Покажите через `[Fact]`, что fixture действительно один и тот же экземпляр для всех тестов класса, но экземпляр тестового класса — новый.
6. Запустите и проанализируйте. Выполните `dotnet test --logger "console;verbosity=normal"`. Убедитесь, что параметризованные тесты разворачиваются в отдельные строки в отчёте — каждый `InlineData` и каждая строка `MemberData`/`ClassData` дают свой результат. Зафиксируйте в файле `NOTES.md`, какие тесты упали на трёх багах и почему. Затем «почините» баги в `DiscountEngine` и убедитесь, что все тесты зелёные. В `NOTES.md` отметьте, какие именно тестовые сценарии поймали каждый из трёх багов.
7. Проверьте детерминизм и независимость. Убедитесь, что порядок выполнения не влияет на результат: запустите `dotnet test -- xunit.parallelizeTestCollections=true` и убедитесь, что ничего не падает из-за гонки состояний. Если в каком-то тесте есть общее статическое поле — refactorите его в fixture или локальную переменную, чтобы тесты стали независимыми.

#### Требования к решению
Решение должно компилироваться на .NET 8 с языковыми возможностями C# 12: top-level statements в `Program.cs` тестового проекта не обязательны, но используйте `record` для DTO, collection expressions (`[]`, `..`) для инициализации массивов, `nameof` для ссылок на члены в `MemberData`, file-scoped namespaces, nullable reference types включены. Каждый параметризованный тест обязан иметь ровно один аргумент-провайдер на каждый сценарий: нельзя смешивать `InlineData` и `MemberData` на одном методе. `MemberData` должен ссылаться строго на `public static` свойство или поле, возвращающее `IEnumerable<object[]>` или `TheoryData<>`. `ClassData`-класс должен иметь публичный конструктор без параметров и наследоваться от `TheoryData<...>` либо реализовывать `IEnumerable<object[]>`.

Имена тестов строго по схеме `Method_Scenario_ExpectedResult`, например `Apply_EmptyCart_ReturnsZeroResult`, `Apply_VipCustomerWithSeasonalFactor_AppliesCombinedDiscount`, `Apply_NegativeQuantity_ThrowsArgumentException`. Каждый тест должен содержать одну логическую проверку: либо `Assert.Equal` для значения, либо `Assert.Throws<T>` для исключения, либо `Assert.NotNull` для непустого объекта — не объединяйте несколько разных проверок в одном теле. Для чисел с плавающей точкой используйте перегрузку `Assert.Equal(expected, actual, precision: 2)`, чтобы избежать ложных срабатываний из-за погрешности `double`/`decimal`. Тесты не должны обращаться к реальной файловой системе или сети — если нужен «каталог», поднимайте его в памяти через `IClassFixture`. Запрещено использовать `Thread.Sleep`, `DateTime.Now` и другие источники недетерминированности; если нужно время — инъектируйте `IClock`-заглушку.

#### Тонкости и подводные камни
Главная ловушка урока — попытка запихнуть всё в `[InlineData]`. Этот атрибут принимает только константы времени компиляции: примитивы, строки, перечисления. Попытка передать `new CartLine(...)` в `InlineData` не скомпилируется. Для ссылочных типов и вычисляемых значений используйте `MemberData` с `TheoryData<CartLine[], CustomerKind, decimal, decimal, decimal>` — это даёт типобезопасность и читаемые имена параметров. Вторая ловушка — `MemberData` на нестатическом члене. Свойство или поле обязано быть `static`, иначе xUnit не сможет получить данные без экземпляра, и тест упадёт с загадочным сообщением. Третья — забытое `nameof`: пишите `[MemberData(nameof(Cases))]`, а не `[MemberData("Cases")]`, чтобы переименование свойства не сломало тест молча.

Четвёртая тонкость — жизненный цикл. xUnit создаёт новый экземпляр тестового класса на каждый тест, поэтому любой `readonly` поле в классе изолировано между тестами автоматически. Это удобно, но это же означает, что тяжёлая инициализация в конструкторе повторится N раз — выносите её в `IClassFixture<T>`. Пятая — `IDisposable`. Если тест открывает `HttpClient` или `FileStream`, реализуйте `IDisposable` на тестовом классе; xUnit вызовет `Dispose` после теста. Шестая — параллелизм. xUnit по умолчанию параллелит коллекции тестов; общее изменяемое статическое состояние между тестами приведёт к недетерминированным падениям. Седьмая — пустая корзина. Не полагайтесь, что `Sum` вернёт `0` для пустого массива — это так, но тест должен явно проверять результат, а не молчаливо полагаться. Восьмая — плавающая точка. `decimal` точнее `double`, но `Math.Floor` всё равно создаёт surprising результаты; всегда сравнивайте с указанием precision. Девятая — `Assert.Throws` возвращает экземпляр исключения, и его можно дополнительно проверить через `Assert.Equal("expected message", ex.Message)` или `Assert.IsType<ArgumentException>(ex)`.

#### Критерии приёмки
- [ ] Создано решение `Pricing.Homework.sln` с проектами `Pricing.Engine` и `Pricing.Engine.Tests` на .NET 8.
- [ ] `dotnet build` проходит без предупреждений уровня error и без ошибок.
- [ ] `dotnet test` до починки багов падает минимум на трёх тестах, соответствующих трём багам.
- [ ] После починки все тесты зелёные; параметризованные тесты разворачиваются в отдельные строки отчёта.
- [ ] Есть минимум один `[Fact]` для инварианта пустой корзины.
- [ ] Есть минимум один `[Theory] + [InlineData]` с тремя и более случаями для констант.
- [ ] Есть минимум один `[Theory] + [MemberData]` со `static` свойством, возвращающим `TheoryData<>` или `IEnumerable<object[]>`.
- [ ] Есть минимум один `[Theory] + [ClassData]` с классом-наследником `TheoryData<...>` и публичным конструктором без параметров.
- [ ] Минимум один тестовый класс реализует `IDisposable` и показывает teardown после каждого теста.
- [ ] Минимум один тестовый класс реализует `IClassFixture<T>` с fixture, реализующей `IDisposable`.
- [ ] Все тесты именованы по схеме `Method_Scenario_ExpectedResult`.
- [ ] Используется C# 12: `record`, file-scoped namespaces, collection expressions, `nameof`.
- [ ] Числовые сравнения с плавающей точкой используют перегрузку с precision.
- [ ] `NOTES.md` фиксирует, какие тесты поймали какие баги.
- [ ] Тесты детерминированы: нет обращений к `DateTime.Now`, файловой системе, сети, `Thread.Sleep`.

#### Подсказки (без прямого ответа)
- Чтобы `InlineData` с перечислением работал, передавайте значение как целое или через приведение: `[InlineData(CustomerKind.Vip)]` допускается, потому что перечисления — константы времени компиляции.
- Для `MemberData` с вычисляемым subtotal используйте выражение `Enumerable.Range(1, 3).Select(i => new CartLine(...))` внутри свойства — это и есть «вычисляемые данные», ради которых `MemberData` существует.
- Если `Assert.Throws<T>` не ловит исключение, проверьте, что тип исключения совпадает или является базовым для реально выброшенного.
- Для проверки «fixture один на класс» заведите в fixture счётчик конструкторов и сравнивайте его значение в разных `[Fact]` — все увидят одно и то же число.
- Чтобы избежать гонки при параллелизме, статическое состояние переносите в `IClassFixture` или в локальную переменную тестового метода.

#### Эталонное решение (разбор)
```csharp
// C# 12 / .NET 8 — xUnit 2.6+
// Эталонное решение ДЗ M15-L02 / Reference solution for homework M15-L02
// Двуязычные комментарии / Bilingual comments

using System;
using System.Collections.Generic;
using System.Linq;
using Xunit;

namespace Pricing.Engine;

// Перечисление типа клиента — константа времени компиляции для InlineData.
// Customer kind enum — a compile-time constant suitable for InlineData.
public enum CustomerKind { Regular, Loyal, Vip }

// Record-DTO для строки корзины. Используем record ради value-семантики.
// Cart-line DTO as a record for value semantics.
public record CartLine(string Sku, decimal UnitPrice, int Quantity);

// Результат применения скидки.
// Result of applying a discount.
public record DiscountResult(decimal FinalPrice, decimal Saved);

// Тестируемая система / System under test.
public sealed class DiscountEngine
{
    public DiscountResult Apply(CartLine[] lines, CustomerKind kind, decimal seasonalFactor)
    {
        // Баг №2: отсутствует проверка отрицательного количества.
        // Bug #2: missing guard for negative quantity.
        if (lines is null) throw new ArgumentNullException(nameof(lines));
        if (seasonalFactor <= 0 || seasonalFactor > 2)
            throw new ArgumentException("seasonalFactor must be in (0; 2]", nameof(seasonalFactor));

        var subtotal = lines.Sum(l => l.UnitPrice * l.Quantity);
        var discount = kind switch
        {
            CustomerKind.Regular => 0m,
            CustomerKind.Loyal   => 0.05m,
            CustomerKind.Vip     => 0.12m,
            _ => throw new ArgumentOutOfRangeException(nameof(kind))
        };

        // Баг №1: Floor до двух знаков создаёт проблему с плавающей точкой.
        // Bug #1: flooring to two decimals causes floating-point surprises.
        var discounted = subtotal * (1m - discount);
        var final = Math.Floor(discounted * seasonalFactor * 100m) / 100m;
        var saved = subtotal - final;

        // Баг №3: пустая корзина возвращает null (ДО починки возвращалось null).
        // Bug #3: empty cart returns null (before the fix it returned null).
        return lines.Length == 0 ? new DiscountResult(0m, 0m) : new DiscountResult(final, saved);
    }
}

namespace Pricing.Engine.Tests;

// --- [Fact]: инвариант / invariant ---
public class DiscountEngineFactTests
{
    private readonly DiscountEngine _engine = new(); // новый экземпляр на каждый тест

    [Fact]
    public void Apply_EmptyCart_ReturnsZeroResult()
    {
        var result = _engine.Apply([], CustomerKind.Regular, 1m);
        Assert.NotNull(result);
        Assert.Equal(0m, result.FinalPrice);
        Assert.Equal(0m, result.Saved);
    }
}

// --- [Theory] + [InlineData]: тривиальные константы / trivial constants ---
public class DiscountEngineInlineTests
{
    [Theory]
    [InlineData(100, 1, CustomerKind.Regular, 1.0, 100)]   // без скидки
    [InlineData(100, 1, CustomerKind.Loyal,   1.0, 95)]    // 5% скидки
    [InlineData(100, 1, CustomerKind.Vip,     1.0, 88)]    // 12% скидки
    [InlineData(10,  10, CustomerKind.Regular, 1.5, 150)]  // сезонный коэффициент 1.5
    public void Apply_InlineConstants_ReturnsExpected(
        decimal price, int qty, CustomerKind kind, decimal seasonal, decimal expectedFinal)
    {
        var engine = new DiscountEngine();
        var line = new CartLine("SKU-1", price, qty);
        var result = engine.Apply([line], kind, seasonal);
        Assert.Equal(expectedFinal, result.FinalPrice, precision: 2);
    }
}

// --- [Theory] + [MemberData]: вычисляемые данные / computed data ---
public class DiscountEngineMemberDataTests
{
    // Статическое свойство — обязательно static, обязательно IEnumerable<object[]> или TheoryData.
    // Must be static; must return IEnumerable<object[]> or TheoryData.
    public static TheoryData<CartLine[], CustomerKind, decimal, decimal> ComputedCases => new()
    {
        { Enumerable.Range(1, 3).Select(i => new CartLine($"SKU-{i}", 10m * i, i)).ToArray(),
          CustomerKind.Vip, 1.1m, 84.70m },
        { [new CartLine("X", 50m, 2)], CustomerKind.Loyal, 1.0m, 95.00m },
    };

    [Theory]
    [MemberData(nameof(ComputedCases))]
    public void Apply_ComputedMemberData_ReturnsExpected(
        CartLine[] lines, CustomerKind kind, decimal seasonal, decimal expectedFinal)
    {
        var result = new DiscountEngine().Apply(lines, kind, seasonal);
        Assert.Equal(expectedFinal, result.FinalPrice, precision: 2);
    }
}

// --- [Theory] + [ClassData]: переиспользуемый источник / reusable source ---
public sealed class DiscountSeasonalData : TheoryData<CartLine[], CustomerKind, decimal, decimal>
{
    public DiscountSeasonalData()
    {
        Add([new CartLine("A", 200m, 1)], CustomerKind.Vip, 0.9m, 158.40m);
        Add([new CartLine("B", 20m, 5)], CustomerKind.Regular, 1.2m, 120.00m);
    }
}

public class DiscountEngineClassDataTests
{
    [Theory]
    [ClassData(typeof(DiscountSeasonalData))]
    public void Apply_ClassData_ReturnsExpected(
        CartLine[] lines, CustomerKind kind, decimal seasonal, decimal expectedFinal)
    {
        Assert.Equal(expectedFinal, new DiscountEngine().Apply(lines, kind, seasonal).FinalPrice, precision: 2);
    }
}

// --- IDisposable: teardown после каждого теста / per-test teardown ---
public sealed class LifecycleTests : IDisposable
{
    private int _counter; // изолировано между тестами, потому что экземпляр новый

    public LifecycleTests() => _counter = 42; // SetUp

    [Fact]
    public void Counter_StartsAtSetupValue() => Assert.Equal(42, _counter);

    public void Dispose() => _counter = 0; // TearDown, вызывается xUnit автоматически
}

// --- IClassFixture: тяжёлая инициализация на класс / heavy setup per class ---
public sealed class SharedCatalogFixture : IDisposable
{
    public int InstanceId { get; } = Random.Shared.Next(1, 1_000_000);
    public IReadOnlyList<string> Catalog => ["SKU-1", "SKU-2", "SKU-3"];
    public void Dispose() { /* освобождение ресурсов каталога / release catalog resources */ }
}

public class FixtureTests : IClassFixture<SharedCatalogFixture>
{
    private readonly SharedCatalogFixture _fixture;
    public FixtureTests(SharedCatalogFixture fixture) => _fixture = fixture;

    [Fact] public void Fixture_HasCatalog() => Assert.NotEmpty(_fixture.Catalog);
    [Fact] public void Fixture_HasStableInstanceId() => Assert.InRange(_fixture.InstanceId, 1, 1_000_000);
}
```

Разбор по строкам. Класс `DiscountEngine` инкапсулирует три намеренных бага, чтобы продемонстрировать, как разные источники данных ловят разные классы ошибок. Метод `Apply` использует pattern matching `switch` по `CustomerKind` — это идиома C# 12, а компилятор через `_ =>` заставляет покрыть все случаи, что само по себе защита от забытого типа клиента. Проверка `seasonalFactor` выбрасывает `ArgumentException` — для этого сценария пишется отдельный `[Theory]` с `Assert.Throws<ArgumentException>`. Коллекция `[]` в `Apply_EmptyCart_ReturnsZeroResult` — это collection expression C# 12, создающий пустой массив; тест явно проверяет `NotNull` и нули, чтобы поймать баг №3 (раньше возвращался `null`).

В `DiscountEngineInlineTests` атрибут `InlineData` передаёт только константы: `decimal` через литералы, `CustomerKind` через значения перечисления (они допустимы, потому что константы времени компиляции). Каждый случай — отдельная строка в отчёте `dotnet test`, что упрощает диагностику. В `DiscountEngineMemberDataTests` используется `TheoryData<...>` вместо сырого `IEnumerable<object[]>` — это даёт типобезопасность и читаемые имена параметров в сообщениях об ошибках; `nameof(ComputedCases)` защищает от переименования. Внутри свойства `Enumerable.Range(...).Select(...)` — это и есть «вычисляемые данные», ради которых `MemberData` существует: такие значения невозможно описать в `InlineData`.

`DiscountSeasonalData` наследует `TheoryData<...>` и имеет публичный конструктор без параметров (требование урока для `ClassData`); метод `Add` типизирован, поэтому опечатка в типе аргумента ловится на компиляции, а не в рантайме. `LifecycleTests` реализует `IDisposable` — xUnit вызывает `Dispose` после каждого теста, что демонстрирует teardown «после каждого теста». Поле `_counter` изолировано между тестами автоматически, потому что xUnit создаёт новый экземпляр класса на каждый тест — это центральная идея жизненного цикла урока. `FixtureTests` реализует `IClassFixture<SharedCatalogFixture>`: fixture создаётся один раз на весь класс и инъектируется через конструктор, тогда как сам тестовый класс по-прежнему инстанцируется заново на каждый тест — это демонстрирует различие между «общей инициализацией» и «изолированным состоянием теста». Все сравнения `decimal` используют `precision: 2`, чтобы избежать ложных падений из-за `Math.Floor` в баге №1; после починки бага тесты остаются зелёными и при повышении precision.

#### Задания на углубление (бонус)
1. Добавьте `[Theory]` с `MemberData`, который генерирует данные из внешнего JSON-файла, загружаемого один раз через `IClassFixture`. Сравните читаемость и скорость против `ClassData`.
2. Реализуйте `IClassFixture<JsonCatalogFixture>` и измерьте через `Stopwatch`, насколько быстрее «общая» инициализация против повторной в конструкторе при 100 тестах.
3. Добавьте тест, который проверяет, что `Apply` выбрасывает `ArgumentException` с конкретным сообщением через `Assert.Throws<ArgumentException>` и последующий `Assert.Contains("seasonalFactor", ex.Message)`.
4. Перепишите `DiscountEngine` с использованием `required`-свойств на `record` и сравните, как изменится форма `InlineData` и `MemberData` — можно ли вообще использовать `InlineData` для `required`-членов?

---

## Statement in English / Постановка на английском

#### Context & motivation
Imagine you have just joined a team building a discount-accounting library for an online store. The `Pricing` module already contains a `DiscountEngine` class that takes a shopping cart and a discount rule as input and returns the final price together with the amount saved. The business logic is non-trivial: there are subtotal thresholds, several customer kinds (regular, loyal, VIP), seasonal factors, and guards against negative values. The code is written, but test coverage is close to zero, while QA keeps filing bug reports every single day.

Your job is to build a unit-test layer in xUnit so that every public scenario of `DiscountEngine` is covered by parameterised tests with thoughtfully chosen data sources. The team follows the best practices from lesson M15-L02: one test, one logical assertion; the `Method_Scenario_ExpectedResult` naming scheme; constants go into `[InlineData]`; computed and complex objects go into `[MemberData]` and `[ClassData]`; heavy setup moves into `IClassFixture<T>`; disposable resources go through `IDisposable`. This is not a toy exercise: QA requires that the new tests catch at least three known bugs in the current implementation and do not fail on valid cases. You are not merely “covering lines”; you must demonstrate that the choice between `[Fact]`, `[Theory] + [InlineData]`, `[Theory] + [MemberData]` and `[Theory] + [ClassData]` is deliberate, driven by the nature of the data and the readability of the set.

#### What to do step by step
1. Create a solution and two projects. From the `C:\projects\course` folder run `dotnet new sln -n Pricing.Homework` inside `modules/M15/homework/M15-L02`. Then create a class library: `dotnet new classlib -n Pricing.Engine -f net8.0` and a test project: `dotnet new xunit -n Pricing.Engine.Tests -f net8.0`. Add both projects to the solution: `dotnet sln add Pricing.Engine/*.csproj Pricing.Engine.Tests/*.csproj`. From the test project add a reference: `dotnet add Pricing.Engine.Tests reference Pricing.Engine`. Verify that `dotnet test` from the solution root runs and reports zero tests without build errors.
2. In `Pricing.Engine`, define the types. Describe `record CustomerType(string Name, decimal DiscountFraction)` or use an enum `CustomerKind { Regular, Loyal, Vip }`. Define `record CartLine(string Sku, decimal UnitPrice, int Quantity)`. Define `record DiscountResult(decimal FinalPrice, decimal Saved)`. Implement `DiscountEngine` with a method `Apply(CartLine[] lines, CustomerKind kind, decimal seasonalFactor)` that sums `UnitPrice * Quantity`, applies the base discount per customer kind (Regular 0, Loyal 0.05, Vip 0.12), multiplies by the seasonal factor so that the final price is `subtotal * (1 - discount) * seasonalFactor`, returns a `DiscountResult`, and throws `ArgumentException` when `seasonalFactor <= 0` or `seasonalFactor > 2`.
3. Deliberately introduce three bugs that the tests must catch. First, the Vip base discount must be rounded down via `Math.Floor` to two decimals — this creates a floating-point pitfall. Second, the guard for negative quantity is missing: `Quantity < 0` should throw `ArgumentException`, but the current code silently sums it. Third, for an empty cart the method returns `null` instead of `DiscountResult(0, 0)`. Record these bugs in a comment so the reviewer can see which tests catch them.
4. In `Pricing.Engine.Tests`, write tests following the lesson pattern. Create `DiscountEngineFactTests` with at least one `[Fact]` for the invariant “an empty cart returns a zero result”. Create `DiscountEngineInlineTests` with `[Theory] + [InlineData]` for trivial constants: small subtotals, valid seasonal factors, `CustomerKind` boundaries. Create `DiscountEngineMemberDataTests` where `MemberData` comes from a `public static IEnumerable<object[]>` property and includes computed values, for example a subtotal calculated via `Enumerable.Sum`. Create `DiscountEngineClassDataTests` with `TheoryData<CartLine[], CustomerKind, decimal, decimal, decimal>` for complex cart objects. Name every test `Method_Scenario_ExpectedResult`.
5. Demonstrate the lifecycle. In one of the test classes implement `IDisposable` to show a per-test teardown: log the call to `Console.WriteLine` or reset a static counter. In another class implement `IClassFixture<SharedCatalogFixture>` where `SharedCatalogFixture` loads a “catalog” (a simple in-memory list of SKUs is enough) once per class and implements `IDisposable`. Use a `[Fact]` to show that the fixture is indeed the same instance for every test in the class, while the test-class instance is fresh per test.
6. Run and analyse. Execute `dotnet test --logger "console;verbosity=normal"`. Verify that parameterised tests expand into individual rows in the report — each `InlineData` row and each `MemberData`/`ClassData` row produces its own result. Record in a `NOTES.md` file which tests fail on the three bugs and why. Then “fix” the bugs in `DiscountEngine` and confirm that every test is green. In `NOTES.md` note which test scenarios caught each of the three bugs.
7. Verify determinism and independence. Confirm that execution order does not affect the outcome: run `dotnet test -- xunit.parallelizeTestCollections=true` and make sure nothing fails because of a state race. If any test relies on a shared static field, refactor it into a fixture or a local variable so that the tests become independent.

#### Requirements
The solution must compile on .NET 8 with C# 12 language features: top-level statements in the test project’s `Program.cs` are optional, but use `record` for DTOs, collection expressions (`[]`, `..`) for array initialisation, `nameof` for member references in `MemberData`, file-scoped namespaces, and nullable reference types enabled. Every parameterised test must use exactly one provider argument per scenario: do not mix `InlineData` and `MemberData` on the same method. `MemberData` must refer strictly to a `public static` property or field returning `IEnumerable<object[]>` or `TheoryData<>`. The `ClassData` class must have a public parameterless constructor and derive from `TheoryData<...>` or implement `IEnumerable<object[]>`.

Test names must follow the `Method_Scenario_ExpectedResult` scheme, e.g. `Apply_EmptyCart_ReturnsZeroResult`, `Apply_VipCustomerWithSeasonalFactor_AppliesCombinedDiscount`, `Apply_NegativeQuantity_ThrowsArgumentException`. Each test must contain a single logical assertion: either `Assert.Equal` for a value, `Assert.Throws<T>` for an exception, or `Assert.NotNull` for a non-null object — do not combine several unrelated checks in one body. For floating-point numbers use the `Assert.Equal(expected, actual, precision: 2)` overload to avoid false positives caused by `double`/`decimal` precision. Tests must not touch the real file system or the network; if a “catalog” is needed, build it in memory through `IClassFixture`. `Thread.Sleep`, `DateTime.Now` and other sources of non-determinism are forbidden; if time is needed, inject an `IClock` stub.

#### Pitfalls
The main trap of the lesson is trying to shove everything into `[InlineData]`. This attribute accepts only compile-time constants: primitives, strings, enums. Trying to pass `new CartLine(...)` into `InlineData` will not compile. For reference types and computed values use `MemberData` with `TheoryData<CartLine[], CustomerKind, decimal, decimal, decimal>` — this gives type safety and readable parameter names. The second trap is `MemberData` on a non-static member. The property or field must be `static`, otherwise xUnit cannot obtain the data without an instance and the test fails with a cryptic message. The third trap is forgetting `nameof`: write `[MemberData(nameof(Cases))]`, not `[MemberData("Cases")]`, so that renaming the property does not silently break the test.

The fourth pitfall is the lifecycle. xUnit creates a new test-class instance for every test, so any `readonly` field is isolated between tests automatically. That is convenient, but it also means that heavy setup in the constructor runs N times — move it into `IClassFixture<T>`. The fifth pitfall is `IDisposable`. If a test opens an `HttpClient` or a `FileStream`, implement `IDisposable` on the test class; xUnit will call `Dispose` after the test. The sixth pitfall is parallelism. xUnit parallelises test collections by default; shared mutable static state across tests will cause non-deterministic failures. The seventh pitfall is the empty cart. Do not rely on `Sum` returning `0` for an empty array — it does, but the test should assert the result explicitly rather than silently assume it. The eighth pitfall is floating point. `decimal` is more precise than `double`, but `Math.Floor` still produces surprising results; always compare with an explicit precision. The ninth is `Assert.Throws`: it returns the exception instance, so you can additionally check `Assert.Equal("expected message", ex.Message)` or `Assert.IsType<ArgumentException>(ex)`.

#### Acceptance criteria
- [ ] A solution `Pricing.Homework.sln` exists with `Pricing.Engine` and `Pricing.Engine.Tests` projects on .NET 8.
- [ ] `dotnet build` succeeds with no error-level warnings and no errors.
- [ ] Before the bug fix `dotnet test` fails on at least three tests corresponding to the three bugs.
- [ ] After the fix all tests are green; parameterised tests expand into individual report rows.
- [ ] At least one `[Fact]` exists for the empty-cart invariant.
- [ ] At least one `[Theory] + [InlineData]` exists with three or more cases for constants.
- [ ] At least one `[Theory] + [MemberData]` exists with a `static` property returning `TheoryData<>` or `IEnumerable<object[]>`.
- [ ] At least one `[Theory] + [ClassData]` exists with a class deriving from `TheoryData<...>` and a public parameterless constructor.
- [ ] At least one test class implements `IDisposable` and shows a per-test teardown.
- [ ] At least one test class implements `IClassFixture<T>` with a fixture that implements `IDisposable`.
- [ ] All tests are named `Method_Scenario_ExpectedResult`.
- [ ] C# 12 is used: `record`, file-scoped namespaces, collection expressions, `nameof`.
- [ ] Floating-point comparisons use the precision overload.
- [ ] `NOTES.md` records which tests caught which bugs.
- [ ] Tests are deterministic: no `DateTime.Now`, file system, network, or `Thread.Sleep`.

#### Hints (no direct answer)
- To make `InlineData` work with an enum, pass the value directly or via a cast: `[InlineData(CustomerKind.Vip)]` is allowed because enums are compile-time constants.
- For `MemberData` with a computed subtotal, use `Enumerable.Range(1, 3).Select(i => new CartLine(...))` inside the property — that is exactly the “computed data” `MemberData` exists for.
- If `Assert.Throws<T>` does not catch the exception, verify that the type matches or is a base type of the actually thrown exception.
- To check that the fixture is shared per class, keep a constructor counter in the fixture and compare its value across different `[Fact]` methods — they all see the same number.
- To avoid races under parallelism, move static state into an `IClassFixture` or into a local variable of the test method.

#### Reference solution walk-through
```csharp
// C# 12 / .NET 8 — xUnit 2.6+
// Reference solution for homework M15-L02
// Bilingual comments

using System;
using System.Collections.Generic;
using System.Linq;
using Xunit;

namespace Pricing.Engine;

// Customer kind enum — a compile-time constant suitable for InlineData.
public enum CustomerKind { Regular, Loyal, Vip }

// Cart-line DTO as a record for value semantics.
public record CartLine(string Sku, decimal UnitPrice, int Quantity);

// Result of applying a discount.
public record DiscountResult(decimal FinalPrice, decimal Saved);

// System under test.
public sealed class DiscountEngine
{
    public DiscountResult Apply(CartLine[] lines, CustomerKind kind, decimal seasonalFactor)
    {
        // Bug #2: missing guard for negative quantity.
        if (lines is null) throw new ArgumentNullException(nameof(lines));
        if (seasonalFactor <= 0 || seasonalFactor > 2)
            throw new ArgumentException("seasonalFactor must be in (0; 2]", nameof(seasonalFactor));

        var subtotal = lines.Sum(l => l.UnitPrice * l.Quantity);
        var discount = kind switch
        {
            CustomerKind.Regular => 0m,
            CustomerKind.Loyal   => 0.05m,
            CustomerKind.Vip     => 0.12m,
            _ => throw new ArgumentOutOfRangeException(nameof(kind))
        };

        // Bug #1: flooring to two decimals causes floating-point surprises.
        var discounted = subtotal * (1m - discount);
        var final = Math.Floor(discounted * seasonalFactor * 100m) / 100m;
        var saved = subtotal - final;

        // Bug #3: empty cart returns null (before the fix it returned null).
        return lines.Length == 0 ? new DiscountResult(0m, 0m) : new DiscountResult(final, saved);
    }
}

namespace Pricing.Engine.Tests;

// --- [Fact]: invariant ---
public class DiscountEngineFactTests
{
    private readonly DiscountEngine _engine = new(); // fresh instance per test

    [Fact]
    public void Apply_EmptyCart_ReturnsZeroResult()
    {
        var result = _engine.Apply([], CustomerKind.Regular, 1m);
        Assert.NotNull(result);
        Assert.Equal(0m, result.FinalPrice);
        Assert.Equal(0m, result.Saved);
    }
}

// --- [Theory] + [InlineData]: trivial constants ---
public class DiscountEngineInlineTests
{
    [Theory]
    [InlineData(100, 1, CustomerKind.Regular, 1.0, 100)]   // no discount
    [InlineData(100, 1, CustomerKind.Loyal,   1.0, 95)]    // 5% discount
    [InlineData(100, 1, CustomerKind.Vip,     1.0, 88)]    // 12% discount
    [InlineData(10,  10, CustomerKind.Regular, 1.5, 150)]  // seasonal factor 1.5
    public void Apply_InlineConstants_ReturnsExpected(
        decimal price, int qty, CustomerKind kind, decimal seasonal, decimal expectedFinal)
    {
        var engine = new DiscountEngine();
        var line = new CartLine("SKU-1", price, qty);
        var result = engine.Apply([line], kind, seasonal);
        Assert.Equal(expectedFinal, result.FinalPrice, precision: 2);
    }
}

// --- [Theory] + [MemberData]: computed data ---
public class DiscountEngineMemberDataTests
{
    // Must be static; must return IEnumerable<object[]> or TheoryData.
    public static TheoryData<CartLine[], CustomerKind, decimal, decimal> ComputedCases => new()
    {
        { Enumerable.Range(1, 3).Select(i => new CartLine($"SKU-{i}", 10m * i, i)).ToArray(),
          CustomerKind.Vip, 1.1m, 84.70m },
        { [new CartLine("X", 50m, 2)], CustomerKind.Loyal, 1.0m, 95.00m },
    };

    [Theory]
    [MemberData(nameof(ComputedCases))]
    public void Apply_ComputedMemberData_ReturnsExpected(
        CartLine[] lines, CustomerKind kind, decimal seasonal, decimal expectedFinal)
    {
        var result = new DiscountEngine().Apply(lines, kind, seasonal);
        Assert.Equal(expectedFinal, result.FinalPrice, precision: 2);
    }
}

// --- [Theory] + [ClassData]: reusable source ---
public sealed class DiscountSeasonalData : TheoryData<CartLine[], CustomerKind, decimal, decimal>
{
    public DiscountSeasonalData()
    {
        Add([new CartLine("A", 200m, 1)], CustomerKind.Vip, 0.9m, 158.40m);
        Add([new CartLine("B", 20m, 5)], CustomerKind.Regular, 1.2m, 120.00m);
    }
}

public class DiscountEngineClassDataTests
{
    [Theory]
    [ClassData(typeof(DiscountSeasonalData))]
    public void Apply_ClassData_ReturnsExpected(
        CartLine[] lines, CustomerKind kind, decimal seasonal, decimal expectedFinal)
    {
        Assert.Equal(expectedFinal, new DiscountEngine().Apply(lines, kind, seasonal).FinalPrice, precision: 2);
    }
}

// --- IDisposable: per-test teardown ---
public sealed class LifecycleTests : IDisposable
{
    private int _counter; // isolated between tests because the instance is fresh

    public LifecycleTests() => _counter = 42; // SetUp

    [Fact]
    public void Counter_StartsAtSetupValue() => Assert.Equal(42, _counter);

    public void Dispose() => _counter = 0; // TearDown, called by xUnit automatically
}

// --- IClassFixture: heavy setup per class ---
public sealed class SharedCatalogFixture : IDisposable
{
    public int InstanceId { get; } = Random.Shared.Next(1, 1_000_000);
    public IReadOnlyList<string> Catalog => ["SKU-1", "SKU-2", "SKU-3"];
    public void Dispose() { /* release catalog resources */ }
}

public class FixtureTests : IClassFixture<SharedCatalogFixture>
{
    private readonly SharedCatalogFixture _fixture;
    public FixtureTests(SharedCatalogFixture fixture) => _fixture = fixture;

    [Fact] public void Fixture_HasCatalog() => Assert.NotEmpty(_fixture.Catalog);
    [Fact] public void Fixture_HasStableInstanceId() => Assert.InRange(_fixture.InstanceId, 1, 1_000_000);
}
```

Line-by-line walk-through. The `DiscountEngine` class encapsulates three deliberate bugs to show how different data sources catch different classes of errors. The `Apply` method uses a pattern-matching `switch` on `CustomerKind` — a C# 12 idiom — and the compiler forces every case to be covered through the `_ =>` arm, which is itself a safeguard against a forgotten customer kind. The `seasonalFactor` guard throws `ArgumentException`, for which a separate `[Theory]` with `Assert.Throws<ArgumentException>` is written. The `[]` collection in `Apply_EmptyCart_ReturnsZeroResult` is a C# 12 collection expression that creates an empty array; the test explicitly checks `NotNull` and zeros to catch bug #3 (which previously returned `null`).

In `DiscountEngineInlineTests`, `InlineData` carries only constants: `decimal` via literals and `CustomerKind` via enum values (allowed because enums are compile-time constants). Each case becomes a separate row in the `dotnet test` report, which simplifies diagnostics. In `DiscountEngineMemberDataTests`, `TheoryData<...>` is used instead of raw `IEnumerable<object[]>` for type safety and readable parameter names in error messages; `nameof(ComputedCases)` protects against silent breakage on rename. Inside the property, `Enumerable.Range(...).Select(...)` is exactly the “computed data” that justifies `MemberData`: such values cannot be expressed in `InlineData`.

`DiscountSeasonalData` derives from `TheoryData<...>` and has a public parameterless constructor (a lesson requirement for `ClassData`); the `Add` method is typed, so a typo in an argument type is caught at compile time rather than at runtime. `LifecycleTests` implements `IDisposable` — xUnit calls `Dispose` after each test, demonstrating a per-test teardown. The `_counter` field is isolated between tests automatically because xUnit creates a new class instance per test — the central lifecycle idea of the lesson. `FixtureTests` implements `IClassFixture<SharedCatalogFixture>`: the fixture is created once per class and injected through the constructor, while the test class itself is still instantiated fresh per test — this shows the distinction between “shared setup” and “isolated test state”. All `decimal` comparisons use `precision: 2` to avoid false failures caused by `Math.Floor` in bug #1; after the fix the tests stay green even when the precision is raised.

#### Going deeper (bonus)
1. Add a `[Theory]` with `MemberData` that generates data from an external JSON file loaded once through `IClassFixture`. Compare readability and speed against `ClassData`.
2. Implement `IClassFixture<JsonCatalogFixture>` and measure with `Stopwatch` how much faster the shared setup is compared to repeating it in the constructor across 100 tests.
3. Add a test that verifies `Apply` throws `ArgumentException` with a specific message via `Assert.Throws<ArgumentException>` followed by `Assert.Contains("seasonalFactor", ex.Message)`.
4. Rewrite `DiscountEngine` using `required` properties on the `record` and compare how the shape of `InlineData` and `MemberData` changes — can `InlineData` be used at all for `required` members?

---

#### Чек-лист сдачи / Submission checklist
- [ ] (RU) Решение собирается через `dotnet build` без ошибок на .NET 8 / C# 12.
- [ ] (RU) Все четыре формы тестов присутствуют: `[Fact]`, `[InlineData]`, `[MemberData]`, `[ClassData]`.
- [ ] (RU) `MemberData` ссылается на `static` член через `nameof`.
- [ ] (RU) `ClassData`-класс имеет публичный конструктор без параметров.
- [ ] (RU) Реализованы `IDisposable` и `IClassFixture<T>` в разных тестовых классах.
- [ ] (RU) Тесты ловят три заведомых бага до починки и зелёные после.
- [ ] (RU) `NOTES.md` фиксирует соответствие тестов и багов.
- [ ] (EN) Solution builds via `dotnet build` without errors on .NET 8 / C# 12.
- [ ] (EN) All four test forms are present: `[Fact]`, `[InlineData]`, `[MemberData]`, `[ClassData]`.
- [ ] (EN) `MemberData` references a `static` member via `nameof`.
- [ ] (EN) The `ClassData` class has a public parameterless constructor.
- [ ] (EN) `IDisposable` and `IClassFixture<T>` are implemented in separate test classes.
- [ ] (EN) Tests catch the three planted bugs before the fix and are green after.
- [ ] (EN) `NOTES.md` records the mapping between tests and bugs.

#### Ресурсы / Resources
- [xUnit — Getting started with .NET CLI — https://xunit.net/docs/getting-started/netcore/cmdline](https://xunit.net/docs/getting-started/netcore/cmdline)
- [xUnit — Live Documentation — https://xunit.net/docs/comparisons](https://xunit.net/docs/comparisons)
- [Microsoft Learn — Unit testing C# in .NET using dotnet test and xUnit — https://learn.microsoft.com/dotnet/core/testing/unit-testing-with-dotnet-test](https://learn.microsoft.com/dotnet/core/testing/unit-testing-with-dotnet-test)
- [Microsoft Learn — TheoryData and Theory — https://learn.microsoft.com/dotnet/core/testing/unit-testing-best-practices](https://learn.microsoft.com/dotnet/core/testing/unit-testing-best-practices)
- [C# 12 — What’s new — https://learn.microsoft.com/dotnet/csharp/whats-new/csharp-12](https://learn.microsoft.com/dotnet/csharp/whats-new/csharp-12)
