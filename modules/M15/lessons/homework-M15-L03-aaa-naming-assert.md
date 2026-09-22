---
[← К уроку M15-L03](lesson-M15-L03-aaa-naming-assert.md) | [⬆ К модулю M15](../README.md) | [Следующее ДЗ →](homework-M15-L04-moq.md)
---

### Домашнее задание M15-L03: AAA, именование, Assert / Homework M15-L03: AAA, naming, Assert

**Урок / Lesson:** M15-L03
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 3/5
**Цель / Goal:** (RU) Научиться писать модульные тесты по паттерну Arrange-Act-Assert, давать тестам имена-спецификации по схеме `Method_Scenario_ExpectedResult`, выбирать самый специфичный метод класса `Assert` и избегать типовых ловушек (несколько действий в Act, логика в Assert, проверка реализации вместо поведения, магические числа). (EN) Learn to write unit tests following the Arrange-Act-Assert pattern, give tests specification-style names following `Method_Scenario_ExpectedResult`, pick the most specific method of the `Assert` class, and avoid common traps (several actions in Act, logic in Assert, testing implementation instead of behavior, magic numbers).

#### Связь с уроком / Connection to the lesson
(RU) Это практическое закрепление урока M15-L03: вы примените три зоны AAA, разделённые пустыми строками, соглашение об именовании `Method_Scenario_ExpectedResult`, набор утверждений xUnit (`Equal`, `True`/`False`, `Null`/`NotNull`, `Contains`, `Throws<T>`, `Collection`, `Single`) и правила выбора самого специфичного assert-а. Тестируемый класс построен так, чтобы провоцировать типичные ошибки из раздела «Частые ошибки» урока — несколько действий в Act, логику в Assert и проверку внутренней реализации вместо поведения.
(EN) This is the hands-on reinforcement of lesson M15-L03: you will apply three AAA zones separated by blank lines, the `Method_Scenario_ExpectedResult` naming convention, the xUnit assert set (`Equal`, `True`/`False`, `Null`/`NotNull`, `Contains`, `Throws<T>`, `Collection`, `Single`) and the rules for picking the most specific assert. The class under test is designed to trigger the typical mistakes from the lesson's "Common Mistakes" section — several actions in Act, logic inside Assert, and testing internal implementation rather than behavior.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация

Вы — backend-разработчик платёжного модуля онлайн-магазина. Команда перешла с ручного тестирования на xUnit, но первые тесты получились неаккуратными: в одном тесте проверяется сразу три поведения, имена выглядят как `Test1` и `TestMethod_Discount`, а вместо `Assert.Equal` везде стоит `Assert.True(result == expected)`. На ревью лида такие тесты не проходят: при падении сообщение «Assert.True failed» не объясняет, что именно сломалось, а открыть код приходится даже для понимания, что именно проверяется.

Вам дают готовый класс `ShippingCalculator` (он будет частью SUT — System Under Test) и просят покрыть его поведение грамотными модульными тестами. Класс считает стоимость доставки заказа: базовый тариф плюс наценки за вес, за расстояние и за срочность, со скидкой для VIP-клиентов. Логика несложная, но богата краевыми случаями: отрицательный вес, нулевая дистанция, срочный заказ для VIP, пустой набор посылок. Каждый такой случай — это отдельный тест с собственным именем-спецификацией и ровно одним утверждением в зоне Act. Параллельно вы должны продемонстрировать, что умеете выбирать самый специфичный assert: `Equal` вместо `True`, `Throws<T>` вместо `try/catch`, `Assert.Single` вместо `Assert.Equal(1, list.Count)`.

Контекст подобран так, чтобы все ловушки урока были видны и преодолимы на одном реалистичном примере: нельзя «случайно» написать несколько действий в Act, потому что у калькулятора только один публичный метод-точка входа; нельзя «случайно» проверить реализацию, потому что приватные поля инкапсулированы; нельзя «случайно» втащить магические числа, потому что тарифы вынесены в именованные константы. Это и есть цель упражнения — выработать мышечную память правильных решений.

#### Что нужно сделать (пошагово)

1. Создайте тестовый проект через `dotnet new xunit -n Shop.Tests` в корне модуля. Убедитесь, что используется `net8.0` и C# 12: откройте `.csproj` и проверьте `<TargetFramework>net8.0</TargetFramework>`. Если версия ниже — обновите. Команда для запуска: `dotnet test` — должна вернуть зелёный.
2. Добавьте класс-потребитель `ShippingCalculator` в проект (он же SUT для тестов). Скопируйте его из раздела «Эталонное решение» ниже или напишите самостоятельно по спецификации: метод `Calculate(Parcel parcel, Customer customer)` возвращает `decimal` — стоимость доставки; выбрасывает `ArgumentException` при отрицательном весе или расстоянии; даёт 15% скидку для VIP. Тарифы — именованные константы `BaseFee`, `PerKiloRate`, `PerKilometerRate`, `ExpressSurcharge`.
3. Для каждого поведения напишите отдельный тест в классе `ShippingCalculatorTests`. Тесты должны следовать паттерну AAA: три зоны, разделённые пустой строкой, комментарий `// Arrange`, `// Act`, `// Assert` в начале каждой зоны (как в примере урока). В зоне Act — ровно один вызов `calculator.Calculate(...)`.
4. Имена тестов стройте строго по схеме `Method_Scenario_ExpectedResult`: например, `Calculate_NegativeWeight_ThrowsArgumentException`, `Calculate_ForVipCustomer_Applies15PercentDiscount`, `Calculate_ExpressOrder_AddsExpressSurcharge`. Каждое имя должно читаться как предложение и не требовать открытия тела теста.
5. Используйте самый специфичный assert для каждой проверки. Для равенства — `Assert.Equal(expected, actual)` (ожидаемое первым!). Для nullability — `Assert.Null`/`Assert.NotNull`. Для коллекций доступных скидок — `Assert.Single` и `Assert.Contains`. Для исключений — `Assert.Throws<T>` с последующей проверкой `Assert.Contains("weight", ex.Message, StringComparison.OrdinalIgnoreCase)`. Запрещено использовать `Assert.True(x == y)` там, где можно `Assert.Equal`.
6. Вынесите магические числа в константы или в параметры `[InlineData]`. Например, базовый тариф должен фигурировать в тесте как `ShippingCalculator.BaseFee`, а не как `50m`. Сетку значений веса и ожидаемой стоимости оформите как `[Theory]` с несколькими `[InlineData]`.
7. Запустите `dotnet test` и добейтесь 100% зелёных тестов. Затем намеренно «сломайте» калькулятор (поменяйте коэффициент скидки VIP с 0.15 на 0.20) и убедитесь, что сообщение об ошибке понятно: оно должно содержать «expected … got …», а не «Assert.True failed». Верните правильный коэффициент.
8. Сделайте коммит с сообщением `test(M15-L03): shipping calculator AAA tests`. Убедитесь, что в истории коммитов появился файл тестов, а `.csproj` ссылается на xUnit правильной версии.

Ожидаемый вывод `dotnet test` в конце: `Passed: 8-10  Failed: 0  Skipped: 0`. Если тестов меньше 6 — вероятно, вы объединили несколько поведений в один тест; вернитесь к шагу 3 и разбейте. Если сообщение об ошибке при падении не содержит чисел — вы используете неспецифичный assert; вернитесь к шагу 5.

#### Требования к решению

- Код должен компилироваться под C# 12 / .NET 8: file-scoped namespaces, top-level statements в `Program.cs` (если он нужен), records для DTO, `ArgumentOutOfRangeException.ThrowIfNegativeOrZero` для валидации аргументов.
- Тестовый проект — отдельная сборка, ссылающаяся на SUT через `<ProjectReference>` или через включение файла класса в оба проекта. Смешивать продакшен-код и тесты в одном проекте нельзя — это нарушает разделение ответственности и мешает релизной сборке.
- Все тесты — методы с атрибутами `[Fact]` или `[Theory]`, помеченные `[Trait]` для группировки (например, `[Trait("Category", "Shipping")]`). Это поможет фильтровать тесты в CI: `dotnet test --filter Category=Shipping`.
- Строгое соблюдение AAA: три зоны, разделённые пустой строкой, ровно один вызов тестируемого метода в Act. Если в Act появляется вторая строка с вызовом — это сигнал, что нужно разбить тест на два.
- Имена тестов — только по схеме `Method_Scenario_ExpectedResult`. Никаких `Test1`, `TestMethod`, `ShouldWork`. Имя должно само объяснять, что проверяется, без необходимости читать тело.
- Используется только встроенный `Assert` xUnit; FluentAssertions и Shouldly не подключаются (по причине лицензии и лишних зависимостей, как указано в уроке).
- Магические числа вынесены в константы или `[InlineData]`. В теле теста не должно быть «голых» `50m`, `0.15m`, `5m` без имени.
- Проверяется поведение, а не реализация: тесты обращаются только к публичному методу `Calculate` и публичным константам. Приватные поля калькулятора не читаются через рефлексию.

#### Тонкости и подводные камни

- **Порядок аргументов в `Assert.Equal`.** Первым идёт ожидаемое значение, вторым — фактическое. Если перепутать, сообщение при падении скажет «expected 200, got 100» наоборот, и коллега потратит лишние минуты на поиск истины. Это прямое правило из best practices урока.
- **`Assert.True(result == expected)` — антипаттерн.** Технически работает, но при падении выдаёт лишь «Assert.True failed» без чисел. Всегда заменяйте на `Assert.Equal(expected, actual)` — сообщение становится самодокументируемым.
- **`Assert.Throws<T>` объединяет Act и Assert.** Это единственный sanctioned случай, когда зоны сливаются: лямбда-выражение — это и есть действие, а сам `Assert.Throws` — проверка. После него можно добавить ещё один `Assert.Contains` для сообщения исключения, и это не нарушает «одно поведение на тест».
- **Несколько действий в Act — частая ловушка.** Если вы пишете `var a = calc.Calculate(p1, c); var b = calc.Calculate(p2, c); Assert.Equal(a, b);`, вы проверяете сразу два поведения и детерминированность, и одно из них. Разбейте на два теста с разными именами.
- **Логика в Assert — ещё одна ловушка.** Если в зоне Assert появляются вычисления (`Assert.Equal(baseFee + weight * rate, result)`), вы переносите часть Arrange в конец теста. Вынесите ожидаемое значение в константу Arrange и сравнивайте «как есть».
- **Проверка реализации вместо поведения.** Не тестируйте приватные поля и не полагайтесь на конкретный алгоритм. Если вы проверяете «внутри калькулятора вызывается такой-то метод», рефакторинг (без изменения поведения) сломает тест. Тестируйте только публичный контракт — входы и выходы.
- **Магические числа без имени.** `0.15m` в теле теста — загадка для читателя: это скидка? налог? коэффициент? Дайте числу имя: `VipDiscountRate` или параметр `[InlineData]`. Это правило прямо из best practices урока.
- **`Assert.Single` vs `Assert.Equal(1, list.Count)`.** Первый даёт понятное сообщение «Collection contained 3 elements instead of 1», второй — «expected 1, got 3». Второй формально работает, но `Single` точнее выражает намерение и даёт лучший контекст.
- **`StringComparison` в `Assert.Contains` для строк.** Указывайте `StringComparison.OrdinalIgnoreCase`, чтобы тест не зависел от регистра сообщения исключения. Иначе малейшее изменение формата («Weight» → «weight») ломает зелёный тест.

#### Критерии приёмки

- [ ] Создан тестовый проект `Shop.Tests` с `TargetFramework=net8.0` и xUnit.
- [ ] Класс `ShippingCalculator` покрыт минимум 6 тестами (рекомендуется 8–10).
- [ ] Каждый тест разбит на три зоны AAA, разделённые пустой строкой, с комментариями `// Arrange`/`// Act`/`// Assert`.
- [ ] В зоне Act каждого теста ровно один вызов тестируемого метода (или одна лямбда для `Assert.Throws`).
- [ ] Все имена тестов следуют схеме `Method_Scenario_ExpectedResult` и читаются как предложение.
- [ ] Использован самый специфичный assert для каждой проверки: `Equal` вместо `True`, `Throws<T>` вместо `try/catch`, `Single` вместо `Count == 1`.
- [ ] В `Assert.Equal` ожидаемое значение всегда идёт первым аргументом.
- [ ] Магические числа вынесены в константы SUT или в параметры `[InlineData]`.
- [ ] Есть хотя бы один `[Theory]` с несколькими `[InlineData]` для параметризованного поведения (например, сетка весов).
- [ ] Есть тест на выброс исключения через `Assert.Throws<ArgumentException>` с проверкой сообщения через `Assert.Contains`.
- [ ] Есть тест на коллекцию через `Assert.Single` или `Assert.Collection`.
- [ ] Тестируется поведение, а не приватные детали реализации.
- [ ] `dotnet test` показывает 100% зелёных; при намеренной поломке сообщения об ошибках содержат числовые значения.
- [ ] Сделан коммит с сообщением `test(M15-L03): shipping calculator AAA tests`.

#### Подсказки (без прямого ответа)

- Подумайте, какие краевые случаи у калькулятора: отрицательный вес, нулевая дистанция, срочность + VIP, пустой набор посылок. Каждый — отдельный тест.
- Для параметризации веса и расстояния используйте `[Theory]` и `[InlineData]` — так одна сигнатура покрывает несколько входов без дублирования Arrange.
- Помните: имя теста — это его спецификация. Если вы не можете сформулировать «при каком условии» и «какой результат» одной фразой, тест, вероятно, проверяет несколько поведений.
- Для проверки исключения сначала решите, что важнее — тип исключения или сообщение. Обычно проверяют оба: `Assert.Throws<T>` даёт тип, последующий `Assert.Contains` — контекст.
- Магические числа в SUT — тоже проблема. Вынесите тарифы в `public const decimal BaseFee = 50m;` — так тесты смогут ссылаться на них и не дублировать значения.

#### Эталонное решение (разбор)

```csharp
// C# 12 / .NET 8, xUnit
// Эталонное ДЗ M15-L03: AAA, именование, Assert / Reference homework

namespace Shop.Tests;

using System;
using System.Collections.Generic;
using Xunit;

// --- SUT: System Under Test / тестируемая система ---

public sealed record Parcel(decimal WeightKg, decimal DistanceKm, bool IsExpress);
public sealed record Customer(bool IsVip);

public sealed class ShippingCalculator
{
    public const decimal BaseFee = 50m;             // базовый тариф / base fee
    public const decimal PerKiloRate = 5m;          // наценка за килограмм / per-kilo charge
    public const decimal PerKilometerRate = 2m;     // наценка за километр / per-km charge
    public const decimal ExpressSurcharge = 30m;    // наценка за срочность / express surcharge
    public const decimal VipDiscountRate = 0.15m;   // 15% скидка VIP / VIP discount

    public decimal Calculate(Parcel parcel, Customer customer)
    {
        // Валидация через throw-хелпер .NET 8 / .NET 8 throw helper
        ArgumentOutOfRangeException.ThrowIfNegativeOrZero(parcel.WeightKg);
        ArgumentOutOfRangeException.ThrowIfNegativeOrZero(parcel.DistanceKm);

        decimal cost = BaseFee
                     + parcel.WeightKg * PerKiloRate
                     + parcel.DistanceKm * PerKilometerRate;

        if (parcel.IsExpress)
            cost += ExpressSurcharge;

        if (customer.IsVip)
            cost *= 1m - VipDiscountRate;           // применяем скидку / apply discount

        return cost;
    }

    public IReadOnlyList<string> GetAvailableSurcharges(Parcel parcel)
    {
        var list = new List<string> { "Base" };

        if (parcel.IsExpress)
            list.Add("Express");

        if (parcel.WeightKg > 10m)
            list.Add("Heavy");

        return list;
    }
}

// --- Тесты / Tests ---

public class ShippingCalculatorTests
{
    // 1) Проверка базового расчёта — Theory с сеткой весов / Theory with weight grid
    [Theory]
    [InlineData(1, 0, 55)]     // 50 + 1*5 + 0*2 = 55
    [InlineData(2, 0, 60)]     // 50 + 2*5 + 0*2 = 60
    [InlineData(0, 1, 52)]     // 50 + 0*5 + 1*2 = 52
    public void Calculate_StandardOrder_ReturnsBasePlusWeightAndDistance(
        decimal weight, decimal distance, decimal expected)
    {
        // Arrange
        var calculator = new ShippingCalculator();
        var parcel = new Parcel(weight, distance, IsExpress: false);
        var customer = new Customer(IsVip: false);

        // Act
        decimal cost = calculator.Calculate(parcel, customer);

        // Assert — самый специфичный assert, ожидаемое первым / specific assert, expected first
        Assert.Equal(expected, cost);
    }

    // 2) Срочный заказ добавляет наценку / Express order adds surcharge
    [Fact]
    public void Calculate_ExpressOrder_AddsExpressSurcharge()
    {
        // Arrange
        var calculator = new ShippingCalculator();
        var parcel = new Parcel(WeightKg: 1m, DistanceKm: 0m, IsExpress: true);
        var customer = new Customer(IsVip: false);
        var expected = ShippingCalculator.BaseFee
                     + 1m * ShippingCalculator.PerKiloRate
                     + ShippingCalculator.ExpressSurcharge;

        // Act
        decimal cost = calculator.Calculate(parcel, customer);

        // Assert
        Assert.Equal(expected, cost);
    }

    // 3) VIP-клиент получает 15% скидку / VIP customer gets 15% discount
    [Fact]
    public void Calculate_ForVipCustomer_Applies15PercentDiscount()
    {
        // Arrange
        var calculator = new ShippingCalculator();
        var parcel = new Parcel(WeightKg: 1m, DistanceKm: 0m, IsExpress: false);
        var customer = new Customer(IsVip: true);
        decimal fullCost = ShippingCalculator.BaseFee + 1m * ShippingCalculator.PerKiloRate;
        decimal expected = fullCost * (1m - ShippingCalculator.VipDiscountRate);

        // Act
        decimal cost = calculator.Calculate(parcel, customer);

        // Assert
        Assert.Equal(expected, cost);
    }

    // 4) Срочность и VIP суммируются: наценка до скидки / Express + VIP stack
    [Fact]
    public void Calculate_ExpressOrderForVip_AddsSurchargeThenAppliesDiscount()
    {
        // Arrange
        var calculator = new ShippingCalculator();
        var parcel = new Parcel(WeightKg: 1m, DistanceKm: 0m, IsExpress: true);
        var customer = new Customer(IsVip: true);
        decimal fullCost = ShippingCalculator.BaseFee
                         + 1m * ShippingCalculator.PerKiloRate
                         + ShippingCalculator.ExpressSurcharge;
        decimal expected = fullCost * (1m - ShippingCalculator.VipDiscountRate);

        // Act
        decimal cost = calculator.Calculate(parcel, customer);

        // Assert
        Assert.Equal(expected, cost);
    }

    // 5) Отрицательный вес выбрасывает исключение / Negative weight throws
    [Fact]
    public void Calculate_NegativeWeight_ThrowsArgumentOutOfRangeException()
    {
        // Arrange
        var calculator = new ShippingCalculator();
        var parcel = new Parcel(WeightKg: -1m, DistanceKm: 1m, IsExpress: false);
        var customer = new Customer(IsVip: false);

        // Act + Assert (Throws объединяет зоны для исключений / Throws merges zones for exceptions)
        var ex = Assert.Throws<ArgumentOutOfRangeException>(
            () => calculator.Calculate(parcel, customer));

        // Дополнительная проверка контекста сообщения / Additional message-context check
        Assert.Contains("weightKg", ex.Message, StringComparison.OrdinalIgnoreCase);
    }

    // 6) Нулевая дистанция тоже невалидна (ThrowIfNegativeOrZero) / Zero distance invalid
    [Fact]
    public void Calculate_ZeroDistance_ThrowsArgumentOutOfRangeException()
    {
        // Arrange
        var calculator = new ShippingCalculator();
        var parcel = new Parcel(WeightKg: 1m, DistanceKg: 0m, IsExpress: false);
        var customer = new Customer(IsVip: false);

        // Act + Assert
        var ex = Assert.Throws<ArgumentOutOfRangeException>(
            () => calculator.Calculate(parcel, customer));

        Assert.Contains("distanceKm", ex.Message, StringComparison.OrdinalIgnoreCase);
    }

    // 7) Коллекция наценок для лёгкого заказа — ровно одна / Single surcharge for light parcel
    [Fact]
    public void GetAvailableSurcharges_LightParcel_ReturnsOnlyBase()
    {
        // Arrange
        var calculator = new ShippingCalculator();
        var parcel = new Parcel(WeightKg: 1m, DistanceKm: 0m, IsExpress: false);

        // Act
        IReadOnlyList<string> surcharges = calculator.GetAvailableSurcharges(parcel);

        // Assert — Single вместо Count == 1, Contains вместо индексной проверки / Single over Count, Contains over index
        Assert.Single(surcharges);
        Assert.Contains("Base", surcharges);
    }

    // 8) Коллекция наценок для срочного тяжёлого заказа — три элемента / Three surcharges
    [Fact]
    public void GetAvailableSurcharges_ExpressHeavyParcel_ReturnsBaseExpressHeavy()
    {
        // Arrange
        var calculator = new ShippingCalculator();
        var parcel = new Parcel(WeightKg: 20m, DistanceKm: 0m, IsExpress: true);

        // Act
        IReadOnlyList<string> surcharges = calculator.GetAvailableSurcharges(parcel);

        // Assert — Assert.Collection проверяет порядок и наличие / Assert.Collection checks order and membership
        Assert.Collection(surcharges,
            s => Assert.Equal("Base", s),
            s => Assert.Equal("Express", s),
            s => Assert.Equal("Heavy", s));
    }
}
```

Разбор по строкам. Зона Arrange во всех тестах создаёт ровно те объекты, которые нужны для одного вызова, и ничего больше — это прямое воплощение правила «повар раскладывает ингредиенты на столе» из теории урока. В тесте 1 используется `[Theory]` с тремя `[InlineData]`: одна сигнатура покрывает три входа без дублирования Arrange, а магические числа (55, 60, 52) вынесены в параметры, а не торчат в теле — это best practice «магические числа в параметры `InlineData`». В тестах 2–4 ожидаемое значение собирается из именованных констант `ShippingCalculator.BaseFee`, `PerKiloRate`, `ExpressSurcharge`, `VipDiscountRate`, а не из голых `50m`, `5m`, `30m`, `0.15m` — так тест самодокументируем и не хрупок к изменению тарифа в одном месте. Зона Act везде ровно из одной строки: `decimal cost = calculator.Calculate(parcel, customer);` — это правило «один вызов в Act» из частых ошибок урока; если бы здесь появилась вторая строка с вызовом, тест нужно было бы разбить. Зона Assert во всех не-исключительных тестах — ровно одна инструкция `Assert.Equal(expected, cost)` с ожидаемым первым аргументом: это и есть «самый специфичный assert» и правильный порядок аргументов. В тестах 5 и 6 зоны Act и Assert объединены через `Assert.Throws<ArgumentOutOfRangeException>(() => ...)`, потому что для проверки исключения лямбда — это и действие, и объект проверки; это единственный sanctioned случай слияния зон, прямо описанный в примере урока с `Calculate_NegativeTotal_ThrowsArgumentException`. Последующий `Assert.Contains("weightKg", ex.Message, StringComparison.OrdinalIgnoreCase)` проверяет контекст сообщения с учётом регистра — best practice для стабильности к формату строки. В тесте 7 используется `Assert.Single` вместо `Assert.Equal(1, list.Count)` — это пример выбора самого специфичного assert-а для коллекций из теории урока, дающий понятное сообщение «Collection contained 3 elements instead of 1». В тесте 8 `Assert.Collection` с тремя лямбдами-инспекторами проверяет и наличие, и порядок элементов — это продвинутый приём из раздела про коллекции, который заменяет три отдельных `Assert.Equal` и сразу падает на первом несоответствии. Имена всех восьми тестов следуют схеме `Method_Scenario_ExpectedResult` и читаются как предложения: `Calculate_ExpressOrder_AddsExpressSurcharge`, `Calculate_ForVipCustomer_Applies15PercentDiscount` — это прямое применение соглашения об именовании из урока, где имя-спецификация заменяет комментарий и объясняет падение в отчёте без открытия кода. Никаких `Test1` или `TestMethod` — это антипаттерн из «Частых ошибок». Тесты обращаются только к публичному методу `Calculate` и публичным константам; приватных полей и внутренней структуры калькулятора тесты не касаются — это правило «тестируй поведение, а не реализацию» из best practices.

#### Задания на углубление (бонус)

1. Добавьте тест на детерминированность: `Calculate_CalledTwiceWithSameInput_ReturnsSameResult`. Подумайте, не нарушает ли он правило «одно поведение на тест» и как сформулировать имя, чтобы это было видно.
2. Расширьте SUT методом `GetAvailableSurcharges` так, чтобы он возвращал коллекцию с дубликатами при определённом входе, и напишите тест через `Assert.Distinct` или ручную проверку уникальности. Сравните читаемость с `Assert.Collection`.
3. Перепишите один тест в стиле FluentAssertions (`result.Should().Be(expected)`) и сравните сообщение об ошибке с xUnit-версией. Обсудите в комментарии, почему в этом курсе выбран встроенный `Assert`.
4. Покройте SUT тестами с `[Trait]`-категориями `Shipping` и `Validation`, запустите `dotnet test --filter Category=Validation` и убедитесь, что выполняются только нужные тесты. Опишите, где такая фильтрация полезна в CI.

---

## Statement in English / Постановка на английском

#### Context & motivation

You are a backend developer on the payment module of an online store. The team has moved from manual testing to xUnit, but the first tests came out messy: a single test verifies three behaviors at once, names look like `Test1` and `TestMethod_Discount`, and everywhere `Assert.Equal` is replaced with `Assert.True(result == expected)`. Such tests do not pass lead review: when they fail, the message "Assert.True failed" does not explain what exactly broke, and you have to open the code even to understand what is being checked.

You are given a ready-made class `ShippingCalculator` (it will be part of the SUT — System Under Test) and asked to cover its behavior with proper unit tests. The class computes the shipping cost of an order: a base fee plus surcharges for weight, for distance, and for express delivery, with a discount for VIP customers. The logic is simple but rich in edge cases: negative weight, zero distance, an express order for a VIP, an empty set of parcels. Each such case is a separate test with its own specification-style name and exactly one assertion in the Act zone. In parallel, you must demonstrate that you can pick the most specific assert: `Equal` over `True`, `Throws<T>` over `try/catch`, `Assert.Single` over `Assert.Equal(1, list.Count)`.

The context is chosen so that every trap from the lesson is visible and resolvable on a single realistic example: you cannot "accidentally" write several actions in Act, because the calculator has only one public entry method; you cannot "accidentally" test implementation, because private fields are encapsulated; you cannot "accidentally" sneak in magic numbers, because rates are extracted into named constants. This is precisely the goal of the exercise — to build muscle memory for the right decisions.

#### What to do step by step

1. Create a test project with `dotnet new xunit -n Shop.Tests` at the module root. Make sure `net8.0` and C# 12 are used: open the `.csproj` and check `<TargetFramework>net8.0</TargetFramework>`. If the version is lower, upgrade it. The run command is `dotnet test` and it should be green.
2. Add the consumer class `ShippingCalculator` to the project (it is also the SUT for the tests). Copy it from the "Reference solution" section below or write it yourself according to the spec: the method `Calculate(Parcel parcel, Customer customer)` returns a `decimal` — the shipping cost; it throws `ArgumentException` for negative weight or distance; it gives a 15% discount to VIP. The rates are named constants `BaseFee`, `PerKiloRate`, `PerKilometerRate`, `ExpressSurcharge`.
3. For each behavior, write a separate test in the `ShippingCalculatorTests` class. Tests must follow the AAA pattern: three zones separated by a blank line, a `// Arrange`, `// Act`, `// Assert` comment at the start of each zone (as in the lesson example). The Act zone contains exactly one call to `calculator.Calculate(...)`.
4. Build test names strictly along the `Method_Scenario_ExpectedResult` scheme: for example, `Calculate_NegativeWeight_ThrowsArgumentException`, `Calculate_ForVipCustomer_Applies15PercentDiscount`, `Calculate_ExpressOrder_AddsExpressSurcharge`. Each name should read as a sentence and not require opening the test body.
5. Use the most specific assert for each check. For equality — `Assert.Equal(expected, actual)` (expected first!). For nullability — `Assert.Null`/`Assert.NotNull`. For collections of available discounts — `Assert.Single` and `Assert.Contains`. For exceptions — `Assert.Throws<T>` followed by `Assert.Contains("weight", ex.Message, StringComparison.OrdinalIgnoreCase)`. It is forbidden to use `Assert.True(x == y)` where `Assert.Equal` is possible.
6. Extract magic numbers into constants or into `[InlineData]` parameters. For example, the base fee should appear in the test as `ShippingCalculator.BaseFee`, not as `50m`. The grid of weight values and expected costs should be a `[Theory]` with several `[InlineData]`.
7. Run `dotnet test` and reach 100% green. Then deliberately "break" the calculator (change the VIP discount coefficient from 0.15 to 0.20) and make sure the error message is understandable: it should contain "expected … got …", not "Assert.True failed". Revert the correct coefficient.
8. Commit with the message `test(M15-L03): shipping calculator AAA tests`. Make sure the test file appears in the commit history and the `.csproj` references the correct xUnit version.

Expected `dotnet test` output at the end: `Passed: 8-10  Failed: 0  Skipped: 0`. If you have fewer than 6 tests, you probably merged several behaviors into one; go back to step 3 and split. If the failure message contains no numbers, you are using a non-specific assert; go back to step 5.

#### Requirements

- The code must compile under C# 12 / .NET 8: file-scoped namespaces, top-level statements in `Program.cs` (if needed), records for DTOs, `ArgumentOutOfRangeException.ThrowIfNegativeOrZero` for argument validation.
- The test project is a separate assembly that references the SUT via `<ProjectReference>` or by including the class file in both projects. Mixing production code and tests in one project is not allowed — it breaks separation of concerns and gets in the way of release builds.
- All tests are methods with `[Fact]` or `[Theory]` attributes, tagged with `[Trait]` for grouping (for example, `[Trait("Category", "Shipping")]`). This helps filter tests in CI: `dotnet test --filter Category=Shipping`.
- Strict AAA compliance: three zones separated by a blank line, exactly one call to the method under test in Act. If a second call line appears in Act, that is a signal to split the test into two.
- Test names follow only the `Method_Scenario_ExpectedResult` scheme. No `Test1`, `TestMethod`, `ShouldWork`. The name must explain what is checked without reading the body.
- Only the built-in xUnit `Assert` is used; FluentAssertions and Shouldly are not added (because of the license and extra dependencies, as stated in the lesson).
- Magic numbers are extracted into constants or `[InlineData]`. The body of a test must not contain "bare" `50m`, `0.15m`, `5m` without a name.
- Behavior is tested, not implementation: tests talk only to the public `Calculate` method and public constants. Private fields of the calculator are not read through reflection.

#### Pitfalls

- **Argument order in `Assert.Equal`.** The expected value comes first, the actual second. If you swap them, the failure message says "expected 200, got 100" the wrong way around, and a teammate spends extra minutes finding the truth. This is a direct rule from the lesson's best practices.
- **`Assert.True(result == expected)` is an anti-pattern.** It technically works, but on failure it only says "Assert.True failed" with no numbers. Always replace it with `Assert.Equal(expected, actual)` — the message becomes self-documenting.
- **`Assert.Throws<T>` merges Act and Assert.** This is the only sanctioned case where the zones blend: the lambda is the action, and `Assert.Throws` itself is the check. After it, you can add another `Assert.Contains` for the exception message, and this does not break the "one behavior per test" rule.
- **Several actions in Act is a common trap.** If you write `var a = calc.Calculate(p1, c); var b = calc.Calculate(p2, c); Assert.Equal(a, b);`, you are verifying two behaviors at once — determinism and one of the behaviors. Split into two tests with different names.
- **Logic in Assert is another trap.** If computations appear in the Assert zone (`Assert.Equal(baseFee + weight * rate, result)`), you are moving part of Arrange to the end of the test. Extract the expected value into an Arrange constant and compare "as is".
- **Testing implementation instead of behavior.** Do not test private fields and do not rely on a specific algorithm. If you check "such-and-such method is called inside the calculator", a refactor (without a behavior change) breaks the test. Test only the public contract — inputs and outputs.
- **Magic numbers without a name.** `0.15m` in the test body is a riddle for the reader: is it a discount? a tax? a coefficient? Give the number a name: `VipDiscountRate` or an `[InlineData]` parameter. This rule comes straight from the lesson's best practices.
- **`Assert.Single` vs `Assert.Equal(1, list.Count)`.** The first gives the clear message "Collection contained 3 elements instead of 1", the second only "expected 1, got 3". The second formally works, but `Single` expresses the intent more precisely and gives better context.
- **`StringComparison` in `Assert.Contains` for strings.** Specify `StringComparison.OrdinalIgnoreCase` so the test does not depend on the case of the exception message. Otherwise, the slightest format change ("Weight" → "weight") breaks a green test.

#### Acceptance criteria

- [ ] A test project `Shop.Tests` is created with `TargetFramework=net8.0` and xUnit.
- [ ] The `ShippingCalculator` class is covered by at least 6 tests (8–10 recommended).
- [ ] Each test is split into three AAA zones separated by a blank line, with `// Arrange`/`// Act`/`// Assert` comments.
- [ ] The Act zone of each test contains exactly one call to the method under test (or one lambda for `Assert.Throws`).
- [ ] All test names follow the `Method_Scenario_ExpectedResult` scheme and read as a sentence.
- [ ] The most specific assert is used for each check: `Equal` over `True`, `Throws<T>` over `try/catch`, `Single` over `Count == 1`.
- [ ] In `Assert.Equal` the expected value is always the first argument.
- [ ] Magic numbers are extracted into SUT constants or `[InlineData]` parameters.
- [ ] There is at least one `[Theory]` with several `[InlineData]` for parameterized behavior (for example, a weight grid).
- [ ] There is a test for throwing an exception via `Assert.Throws<ArgumentException>` with a message check via `Assert.Contains`.
- [ ] There is a collection test via `Assert.Single` or `Assert.Collection`.
- [ ] Behavior is tested, not private implementation details.
- [ ] `dotnet test` shows 100% green; on a deliberate break, failure messages contain numeric values.
- [ ] A commit with the message `test(M15-L03): shipping calculator AAA tests` is made.

#### Hints (no direct answer)

- Think about the calculator's edge cases: negative weight, zero distance, express + VIP, empty parcel set. Each is a separate test.
- To parameterize weight and distance, use `[Theory]` and `[InlineData]` — one signature covers several inputs without Arrange duplication.
- Remember: the test name is its specification. If you cannot phrase "under what condition" and "with what result" in a single phrase, the test probably checks several behaviors.
- For exception checks, first decide what matters more — the exception type or the message. Usually both are checked: `Assert.Throws<T>` gives the type, the following `Assert.Contains` gives the context.
- Magic numbers in the SUT are also a problem. Extract rates into `public const decimal BaseFee = 50m;` — tests can then reference them and not duplicate values.

#### Reference solution walk-through

```csharp
// C# 12 / .NET 8, xUnit
// Reference homework M15-L03: AAA, naming, Assert

namespace Shop.Tests;

using System;
using System.Collections.Generic;
using Xunit;

// --- SUT: System Under Test ---

public sealed record Parcel(decimal WeightKg, decimal DistanceKm, bool IsExpress);
public sealed record Customer(bool IsVip);

public sealed class ShippingCalculator
{
    public const decimal BaseFee = 50m;             // base fee
    public const decimal PerKiloRate = 5m;          // per-kilo charge
    public const decimal PerKilometerRate = 2m;     // per-km charge
    public const decimal ExpressSurcharge = 30m;    // express surcharge
    public const decimal VipDiscountRate = 0.15m;   // 15% VIP discount

    public decimal Calculate(Parcel parcel, Customer customer)
    {
        // Validation via the .NET 8 throw helper
        ArgumentOutOfRangeException.ThrowIfNegativeOrZero(parcel.WeightKg);
        ArgumentOutOfRangeException.ThrowIfNegativeOrZero(parcel.DistanceKm);

        decimal cost = BaseFee
                     + parcel.WeightKg * PerKiloRate
                     + parcel.DistanceKm * PerKilometerRate;

        if (parcel.IsExpress)
            cost += ExpressSurcharge;

        if (customer.IsVip)
            cost *= 1m - VipDiscountRate;           // apply discount

        return cost;
    }

    public IReadOnlyList<string> GetAvailableSurcharges(Parcel parcel)
    {
        var list = new List<string> { "Base" };

        if (parcel.IsExpress)
            list.Add("Express");

        if (parcel.WeightKg > 10m)
            list.Add("Heavy");

        return list;
    }
}

// --- Tests ---

public class ShippingCalculatorTests
{
    // 1) Base computation — Theory with a weight grid
    [Theory]
    [InlineData(1, 0, 55)]     // 50 + 1*5 + 0*2 = 55
    [InlineData(2, 0, 60)]     // 50 + 2*5 + 0*2 = 60
    [InlineData(0, 1, 52)]     // 50 + 0*5 + 1*2 = 52
    public void Calculate_StandardOrder_ReturnsBasePlusWeightAndDistance(
        decimal weight, decimal distance, decimal expected)
    {
        // Arrange
        var calculator = new ShippingCalculator();
        var parcel = new Parcel(weight, distance, IsExpress: false);
        var customer = new Customer(IsVip: false);

        // Act
        decimal cost = calculator.Calculate(parcel, customer);

        // Assert — most specific assert, expected first
        Assert.Equal(expected, cost);
    }

    // 2) Express order adds the surcharge
    [Fact]
    public void Calculate_ExpressOrder_AddsExpressSurcharge()
    {
        // Arrange
        var calculator = new ShippingCalculator();
        var parcel = new Parcel(WeightKg: 1m, DistanceKm: 0m, IsExpress: true);
        var customer = new Customer(IsVip: false);
        var expected = ShippingCalculator.BaseFee
                     + 1m * ShippingCalculator.PerKiloRate
                     + ShippingCalculator.ExpressSurcharge;

        // Act
        decimal cost = calculator.Calculate(parcel, customer);

        // Assert
        Assert.Equal(expected, cost);
    }

    // 3) VIP customer gets a 15% discount
    [Fact]
    public void Calculate_ForVipCustomer_Applies15PercentDiscount()
    {
        // Arrange
        var calculator = new ShippingCalculator();
        var parcel = new Parcel(WeightKg: 1m, DistanceKm: 0m, IsExpress: false);
        var customer = new Customer(IsVip: true);
        decimal fullCost = ShippingCalculator.BaseFee + 1m * ShippingCalculator.PerKiloRate;
        decimal expected = fullCost * (1m - ShippingCalculator.VipDiscountRate);

        // Act
        decimal cost = calculator.Calculate(parcel, customer);

        // Assert
        Assert.Equal(expected, cost);
    }

    // 4) Express and VIP stack: surcharge first, then discount
    [Fact]
    public void Calculate_ExpressOrderForVip_AddsSurchargeThenAppliesDiscount()
    {
        // Arrange
        var calculator = new ShippingCalculator();
        var parcel = new Parcel(WeightKg: 1m, DistanceKm: 0m, IsExpress: true);
        var customer = new Customer(IsVip: true);
        decimal fullCost = ShippingCalculator.BaseFee
                         + 1m * ShippingCalculator.PerKiloRate
                         + ShippingCalculator.ExpressSurcharge;
        decimal expected = fullCost * (1m - ShippingCalculator.VipDiscountRate);

        // Act
        decimal cost = calculator.Calculate(parcel, customer);

        // Assert
        Assert.Equal(expected, cost);
    }

    // 5) Negative weight throws
    [Fact]
    public void Calculate_NegativeWeight_ThrowsArgumentOutOfRangeException()
    {
        // Arrange
        var calculator = new ShippingCalculator();
        var parcel = new Parcel(WeightKg: -1m, DistanceKm: 1m, IsExpress: false);
        var customer = new Customer(IsVip: false);

        // Act + Assert (Throws merges the zones for exceptions)
        var ex = Assert.Throws<ArgumentOutOfRangeException>(
            () => calculator.Calculate(parcel, customer));

        // Additional message-context check
        Assert.Contains("weightKg", ex.Message, StringComparison.OrdinalIgnoreCase);
    }

    // 6) Zero distance is also invalid (ThrowIfNegativeOrZero)
    [Fact]
    public void Calculate_ZeroDistance_ThrowsArgumentOutOfRangeException()
    {
        // Arrange
        var calculator = new ShippingCalculator();
        var parcel = new Parcel(WeightKg: 1m, DistanceKg: 0m, IsExpress: false);
        var customer = new Customer(IsVip: false);

        // Act + Assert
        var ex = Assert.Throws<ArgumentOutOfRangeException>(
            () => calculator.Calculate(parcel, customer));

        Assert.Contains("distanceKm", ex.Message, StringComparison.OrdinalIgnoreCase);
    }

    // 7) Surcharges for a light parcel — exactly one
    [Fact]
    public void GetAvailableSurcharges_LightParcel_ReturnsOnlyBase()
    {
        // Arrange
        var calculator = new ShippingCalculator();
        var parcel = new Parcel(WeightKg: 1m, DistanceKm: 0m, IsExpress: false);

        // Act
        IReadOnlyList<string> surcharges = calculator.GetAvailableSurcharges(parcel);

        // Assert — Single over Count == 1, Contains over an index check
        Assert.Single(surcharges);
        Assert.Contains("Base", surcharges);
    }

    // 8) Surcharges for an express heavy parcel — three items
    [Fact]
    public void GetAvailableSurcharges_ExpressHeavyParcel_ReturnsBaseExpressHeavy()
    {
        // Arrange
        var calculator = new ShippingCalculator();
        var parcel = new Parcel(WeightKg: 20m, DistanceKm: 0m, IsExpress: true);

        // Act
        IReadOnlyList<string> surcharges = calculator.GetAvailableSurcharges(parcel);

        // Assert — Assert.Collection checks order and membership
        Assert.Collection(surcharges,
            s => Assert.Equal("Base", s),
            s => Assert.Equal("Express", s),
            s => Assert.Equal("Heavy", s));
    }
}
```

Line-by-line walk-through. The Arrange zone in every test creates exactly the objects needed for one call and nothing more — a direct embodiment of the "chef lays ingredients on the table" rule from the lesson theory. Test 1 uses `[Theory]` with three `[InlineData]` rows: a single signature covers three inputs without Arrange duplication, and the magic numbers (55, 60, 52) live in parameters rather than in the body — the "magic numbers into InlineData" best practice. In tests 2–4 the expected value is assembled from named constants `ShippingCalculator.BaseFee`, `PerKiloRate`, `ExpressSurcharge`, `VipDiscountRate`, not from bare `50m`, `5m`, `30m`, `0.15m` — the test is self-documenting and not brittle when a rate changes in one place. The Act zone is always exactly one line: `decimal cost = calculator.Calculate(parcel, customer);` — the "one call in Act" rule from the lesson's common mistakes; if a second call line appeared here, the test would have to be split. The Assert zone in every non-exceptional test is exactly one `Assert.Equal(expected, cost)` instruction with the expected value first: this is both "the most specific assert" and the correct argument order. In tests 5 and 6 the Act and Assert zones are merged via `Assert.Throws<ArgumentOutOfRangeException>(() => ...)`, because for exception checks the lambda is both the action and the object under check; this is the only sanctioned zone merge, directly described in the lesson example with `Calculate_NegativeTotal_ThrowsArgumentException`. The follow-up `Assert.Contains("weightKg", ex.Message, StringComparison.OrdinalIgnoreCase)` checks the message context case-insensitively — a best practice for stability against string format changes. Test 7 uses `Assert.Single` instead of `Assert.Equal(1, list.Count)` — an example of picking the most specific assert for collections from the lesson theory, giving the clear message "Collection contained 3 elements instead of 1". Test 8 uses `Assert.Collection` with three inspector lambdas to verify both presence and order — an advanced technique from the collections section that replaces three separate `Assert.Equal` calls and fails fast on the first mismatch. The names of all eight tests follow the `Method_Scenario_ExpectedResult` scheme and read as sentences: `Calculate_ExpressOrder_AddsExpressSurcharge`, `Calculate_ForVipCustomer_Applies15PercentDiscount` — a direct application of the naming convention from the lesson, where a specification-style name replaces a comment and explains a failure in the report without opening the code. There are no `Test1` or `TestMethod` names — those are anti-patterns from "Common Mistakes". Tests touch only the public `Calculate` method and public constants; private fields and internal structure of the calculator are never accessed — the "test behavior, not implementation" rule from best practices.

#### Going deeper (bonus)

1. Add a determinism test: `Calculate_CalledTwiceWithSameInput_ReturnsSameResult`. Think about whether it breaks the "one behavior per test" rule and how to phrase the name so that is visible.
2. Extend the SUT with a `GetAvailableSurcharges` variant that returns a collection with duplicates on a certain input, and write a test using `Assert.Distinct` or a manual uniqueness check. Compare readability with `Assert.Collection`.
3. Rewrite one test in the FluentAssertions style (`result.Should().Be(expected)`) and compare the failure message with the xUnit version. Discuss in a comment why this course chose the built-in `Assert`.
4. Cover the SUT with tests tagged by `[Trait]` categories `Shipping` and `Validation`, run `dotnet test --filter Category=Validation`, and make sure only the right tests run. Describe where such filtering is useful in CI.

---

#### Чек-лист сдачи / Submission checklist

- [ ] (RU) Создан проект `Shop.Tests` (net8.0, xUnit).
- [ ] (RU) Класс `ShippingCalculator` покрыт минимум 6 тестами.
- [ ] (RU) Каждый тест разбит на три зоны AAA с комментариями.
- [ ] (RU) В Act ровно один вызов тестируемого метода.
- [ ] (RU) Имена тестов по схеме `Method_Scenario_ExpectedResult`.
- [ ] (RU) Использованы самые специфичные `Assert.*`.
- [ ] (RU) В `Assert.Equal` ожидаемое значение первым.
- [ ] (RU) Магические числа вынесены в константы или `[InlineData]`.
- [ ] (RU) Есть `[Theory]`, тест на `Assert.Throws`, тест на коллекцию.
- [ ] (RU) `dotnet test` зелёный; коммит `test(M15-L03): shipping calculator AAA tests`.
- [ ] (EN) Project `Shop.Tests` created (net8.0, xUnit).
- [ ] (EN) `ShippingCalculator` covered by at least 6 tests.
- [ ] (EN) Each test split into three AAA zones with comments.
- [ ] (EN) Act contains exactly one call to the method under test.
- [ ] (EN) Test names follow `Method_Scenario_ExpectedResult`.
- [ ] (EN) The most specific `Assert.*` is used.
- [ ] (EN) In `Assert.Equal` the expected value is first.
- [ ] (EN) Magic numbers extracted into constants or `[InlineData]`.
- [ ] (EN) There is a `[Theory]`, an `Assert.Throws` test, a collection test.
- [ ] (EN) `dotnet test` is green; commit `test(M15-L03): shipping calculator AAA tests`.

#### Ресурсы / Resources

- Microsoft Learn — Unit testing best practices — https://learn.microsoft.com/dotnet/core/testing/unit-testing-best-practices
- xUnit Documentation — Assert class — https://xunit.net/docs/comparing-asserts
- Microsoft Learn — `ArgumentOutOfRangeException.ThrowIfNegativeOrZero` — https://learn.microsoft.com/dotnet/api/system.argumentoutofrangeexception.throwifnegativeorzero
- Martin Fowler — Given-When-Then and AAA — https://martinfowler.com/bliki/GivenWhenThen.html
