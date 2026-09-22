[⬆ К модулю M15](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M15-L03: AAA, именование, Assert / AAA, naming, Assert

**Модуль / Module:** M15
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

Юнит-тест — это не просто «код, который что-то проверяет». Это короткий, читаемый и самодокументируемый сценарий, который объясняет, как система должна себя вести в конкретной ситуации. Чтобы тесты были понятными и стабильными, индустрия выработала простой, но мощный паттерн — **Arrange-Act-Assert (AAA)**. Он делит тело теста на три зоны, разделённые пустой строкой, каждая из которых отвечает на свой вопрос.

**Arrange (Подготовка).** Здесь мы создаём объекты, настраиваем зависимости, готовим входные данные. Аналогия: повар перед готовкой раскладывает ингредиенты на столе — мясо, специи, сковорода. Пока ничего не жарится, всё просто лежит наготове. В этой зоне не должно быть никаких вызовов проверяемой логики.

**Act (Действие).** Один вызов тестируемого метода. Именно здесь происходит «жарка». Аналогия: повар кладёт мясо на сковороду. Зона Act должна быть максимально короткой — обычно одна строка. Если в Act появляется несколько вызовов или ветвлений, значит, тест проверяет сразу несколько поведений и его нужно разбить.

**Assert (Проверка).** Сравнение фактического результата с ожидаемым. Аналогия: повар пробует блюдо — солёное ли, готово ли. Здесь мы задаём вопрос «что должно было получиться?» и сверяем с тем, что получилось. На один тест — одно логическое утверждение в идеале, хотя технически assert-ов может быть несколько, если они проверяют одну и ту же мысль.

Паттерн AAA даёт два главных выигрыша: читаемость (любой разработчик за 5 секунд видит, что тестируется и что ожидается) и стабильность (чёткие зоны мешают «размазать» логику и случайно проверить побочный эффект вместо главного).

**Именование тестов.** Имя теста — это его спецификация. Соглашение `MethodName_Scenario_ExpectedResult` стало де-факто стандартом в .NET-сообществе. Пример: `Discount_AppliesForVipCustomer_Returns20PercentOff`. Три части отвечают на три вопроса: что вызываем, при каком условии, какой результат. Преимущества: имя читается как предложение, в отчёте о падении сразу видно, что сломалось, и не нужны комментарии. Альтернативы вроде `ShouldApplyDiscountForVip` тоже допустимы, но хуже тем, что не указывают метод и ожидаемый результат явно. Избегайте имён `Test1`, `TestMethod_Discount` — они не несут смысла без открытия кода.

**Класс утверждений Assert.** xUnit предоставляет богатый набор: `Assert.Equal(expected, actual)` для равенства, `Assert.True(condition)` / `Assert.False(condition)` для булевых проверок, `Assert.Null` / `Assert.NotNull` для nullability, `Assert.Contains` для наличия элемента, `Assert.Throws<TException>(action)` для проверки выброса исключений. Главное правило: выбирайте самый специфичный assert. Не пишите `Assert.True(result == 42)`, когда можно `Assert.Equal(42, result)` — второе даёт понятное сообщение об ошибке «expected 42, got 41», а первое лишь «Assert.True failed».

**Fluent Asserts.** Библиотеки вроде FluentAssertions (или `Shouldly`) превращают проверки в читаемые предложения: `result.Should().Be(42).And.BePositive()`. Это особенно удобно для сложных объектов и коллекций. Однако FluentAssertions с версии 8 перешла на коммерческую лицензию — для open-source проектов рассмотрите альтернативы (`Shouldly`, встроенные `Assert.Collection`). В этом курсе мы остаёмся на встроенном xUnit `Assert`, чтобы избежать лишних зависимостей.

**Частые ловушки.** Смешивание зон (логика в Assert), несколько действий в Act, проверка реализации вместо поведения, магические числа без имени. Каждая из них снижает читаемость и стабильность теста. Помните: тест — это документация, которая никогда не врёт, потому что компилятор и раннер заставляют её быть актуальной.

#### Theory (EN)

A unit test is not just «code that checks something». It is a short, readable, self-documenting scenario that explains how the system should behave in a specific situation. To keep tests clear and stable, the industry has adopted a simple but powerful pattern — **Arrange-Act-Assert (AAA)**. It splits the test body into three zones separated by blank lines, each answering its own question.

**Arrange.** Here we create objects, configure dependencies, prepare inputs. Analogy: a chef before cooking lays ingredients on the table — meat, spices, a pan. Nothing is being fried yet, everything is just ready. No invocation of the logic under test belongs here.

**Act.** A single call to the method under test. This is where the «frying» happens. Analogy: the chef puts meat in the pan. The Act zone should be as short as possible — usually one line. If multiple calls or branches appear in Act, the test is verifying several behaviors at once and must be split.

**Assert.** Comparison of the actual result against the expected one. Analogy: the chef tastes the dish — is it salty, is it done. Here we ask «what should have happened?» and compare with what did happen. Ideally one logical assertion per test, although technically several asserts are fine if they verify the same idea.

The AAA pattern gives two main wins: readability (any developer sees in 5 seconds what is tested and what is expected) and stability (clear zones prevent smearing logic and accidentally testing a side effect instead of the main behavior).

**Test naming.** A test name is its specification. The `MethodName_Scenario_ExpectedResult` convention has become the de-facto standard in the .NET community. Example: `Discount_AppliesForVipCustomer_Returns20PercentOff`. The three parts answer three questions: what we call, under what condition, with what result. Benefits: the name reads like a sentence, a failure report immediately shows what broke, and no comments are needed. Alternatives like `ShouldApplyDiscountForVip` are acceptable but weaker because they do not name the method or the expected result explicitly. Avoid names like `Test1` or `TestMethod_Discount` — they carry no meaning without opening the code.

**The Assert class.** xUnit ships a rich set: `Assert.Equal(expected, actual)` for equality, `Assert.True(condition)` / `Assert.False(condition)` for booleans, `Assert.Null` / `Assert.NotNull` for nullability, `Assert.Contains` for membership, `Assert.Throws<TException>(action)` for exception checks. The key rule: pick the most specific assert. Do not write `Assert.True(result == 42)` when you can write `Assert.Equal(42, result)` — the second gives a clear message «expected 42, got 41», while the first only says «Assert.True failed».

**Fluent Asserts.** Libraries like FluentAssertions (or `Shouldly`) turn checks into readable sentences: `result.Should().Be(42).And.BePositive()`. This is especially handy for complex objects and collections. Note, however, that FluentAssertions from version 8 moved to a commercial license — for open-source projects consider alternatives (`Shouldly`, built-in `Assert.Collection`). In this course we stick with the built-in xUnit `Assert` to avoid extra dependencies.

**Common traps.** Mixing zones (logic in Assert), several actions in Act, testing implementation instead of behavior, magic numbers without a name. Each lowers readability and stability. Remember: a test is documentation that never lies, because the compiler and the runner force it to stay current.

#### Пример кода / Code Example

```csharp
// C# 12+ / .NET 8+, xUnit
// Примеры паттерна AAA, именования и утверждений / AAA, naming and assert examples

namespace Shop.Tests;

public class DiscountCalculatorTests
{
    // AAA: три зоны, разделённые пустой строкой / Three zones separated by blank lines
    [Fact]
    public void Discount_AppliesForVipCustomer_Returns20PercentOff()
    {
        // Arrange — подготовка данных / prepare data
        var calculator = new DiscountCalculator();
        var customer = new Customer(IsVip: true, YearsRegistered: 3);

        // Act — один вызов тестируемого метода / single call to the method under test
        decimal discount = calculator.Calculate(customer, orderTotal: 1000m);

        // Assert — специфичное утверждение / specific assertion
        Assert.Equal(200m, discount);
    }

    // Имя теста = спецификация: Method_Scenario_ExpectedResult / Name = specification
    [Theory]
    [InlineData(0, 0)]      // 0 лет — без скидки / 0 years → no discount
    [InlineData(1, 50)]     // 1 год — 5% / 1 year → 5%
    [InlineData(5, 100)]    // 5 лет — 10% / 5 years → 10%
    public void LoyaltyDiscount_ByYearsRegistered_ReturnsExpectedAmount(
        int years, int expectedDiscount)
    {
        // Arrange
        var calculator = new DiscountCalculator();
        var customer = new Customer(IsVip: false, YearsRegistered: years);

        // Act
        decimal discount = calculator.Calculate(customer, orderTotal: 1000m);

        // Assert — используем самый специфичный assert / use the most specific assert
        Assert.Equal(expectedDiscount, discount);
    }

    // Проверка исключения через Assert.Throws / Exception check via Assert.Throws
    [Fact]
    public void Calculate_NegativeTotal_ThrowsArgumentException()
    {
        // Arrange
        var calculator = new DiscountCalculator();
        var customer = new Customer(IsVip: false, YearsRegistered: 0);

        // Act + Assert объединены для Throws / Act + Assert merged for Throws
        var ex = Assert.Throws<ArgumentException>(
            () => calculator.Calculate(customer, orderTotal: -1m));

        // Дополнительная проверка сообщения / additional message check
        Assert.Contains("orderTotal", ex.Message, StringComparison.OrdinalIgnoreCase);
    }

    // Проверка коллекции через Assert.Collection / Collection check via Assert.Collection
    [Fact]
    public void GetAvailableDiscounts_ForNewCustomer_ReturnsOnlyWelcomeDiscount()
    {
        // Arrange
        var calculator = new DiscountCalculator();
        var customer = new Customer(IsVip: false, YearsRegistered: 0);

        // Act
        IReadOnlyList<string> discounts = calculator.GetAvailableDiscounts(customer);

        // Assert — точная структура коллекции / exact collection structure
        Assert.Single(discounts);
        Assert.Equal("Welcome10", discounts[0]);
        Assert.Contains("Welcome10", discounts);
    }
}

// Минимальный SUT для компилируемости примера / Minimal SUT so the example compiles
public sealed record Customer(bool IsVip, int YearsRegistered);

public sealed class DiscountCalculator
{
    public decimal Calculate(Customer customer, decimal orderTotal)
    {
        ArgumentOutOfRangeException.ThrowIfNegativeOrZero(orderTotal);

        decimal discount = 0m;

        if (customer.IsVip)
            discount += orderTotal * 0.20m;     // 20% VIP / 20% VIP

        discount += orderTotal * Math.Min(customer.YearsRegistered * 0.01m, 0.10m);

        return discount;
    }

    public IReadOnlyList<string> GetAvailableDiscounts(Customer customer)
    {
        var list = new List<string> { "Welcome10" };

        if (customer.IsVip)
            list.Add("Vip20");

        if (customer.YearsRegistered >= 1)
            list.Add($"Loyalty{Math.Min(customer.YearsRegistered * 5, 50)}");

        return list;
    }
}
```

#### Best Practices

- Один тест — одно поведение. Если в имени теста появляется «и», скорее всего, тест нужно разбить на два.
- Разделяйте зоны AAA пустой строкой — это визуальный сигнал читателю и самому себе.
- Используйте самый специфичный `Assert.*`: `Equal` вместо `True(x == y)`, `Throws<T>` вместо `try/catch + Assert.Fail`.
- Имя теста должно читаться как предложение: `Method_Scenario_ExpectedResult`. Избегайте `Test1`, `TestMethod`.
- В `Assert.Equal` первым аргументом передавайте ожидаемое значение, вторым — фактическое — так сообщение об ошибке будет корректным.
- Магические числа выносите в именованные константы или параметры `[InlineData]`.
- One test — one behavior. If the word «and» sneaks into the test name, you probably need two tests.
- Separate AAA zones with a blank line — a visual signal to the reader and to yourself.
- Use the most specific `Assert.*`: `Equal` over `True(x == y)`, `Throws<T>` over `try/catch + Assert.Fail`.
- A test name should read as a sentence: `Method_Scenario_ExpectedResult`. Avoid `Test1`, `TestMethod`.
- In `Assert.Equal` pass the expected value first and the actual second — so the failure message is correct.
- Extract magic numbers into named constants or `[InlineData]` parameters.

#### Частые ошибки / Common Mistakes

- Несколько действий в зоне Act → разбейте тест, чтобы каждый проверял ровно одно поведение. (RU)
- Логика в зоне Assert (вычисления, ветвления) → оставьте в Assert только сравнения, всю подготовку переносите в Arrange. (RU)
- `Assert.True(result == 42)` вместо `Assert.Equal(42, result)` → выбирайте специфичный assert для понятного сообщения об ошибке. (RU)
- Имена `Test1`, `TestDiscount` → используйте `Method_Scenario_ExpectedResult`, имя должно само объяснять, что проверяется. (RU)
- Проверка внутренней реализации вместо поведения → тестируйте публичный контракт, иначе рефакторинг ломает тесты без изменения поведения. (RU)
- Several actions in the Act zone → split the test so each verifies exactly one behavior. (EN)
- Logic inside Assert (computations, branches) → keep only comparisons in Assert, move all setup to Arrange. (EN)
- `Assert.True(result == 42)` instead of `Assert.Equal(42, result)` → pick the specific assert for a clear failure message. (EN)
- Names like `Test1`, `TestDiscount` → use `Method_Scenario_ExpectedResult`, the name should explain what is checked. (EN)
- Testing internal implementation instead of behavior → test the public contract, otherwise refactoring breaks tests without behavior changes. (EN)

#### Чек-лист самопроверки / Self-check Checklist

- [ ] Тест разделён на три зоны AAA пустыми строками. (RU)
- [ ] В зоне Act ровно один вызов тестируемого метода. (RU)
- [ ] Имя теста следует шаблону `Method_Scenario_ExpectedResult` и читается как предложение. (RU)
- [ ] Использован самый специфичный `Assert.*` для каждой проверки. (RU)
- [ ] В `Assert.Equal` ожидаемое значение идёт первым аргументом. (RU)
- [ ] Проверяется поведение, а не детали реализации. (RU)
- [ ] Нет магических чисел без имени — вынесены в константы или параметры. (RU)
- [ ] The test is split into three AAA zones with blank lines. (EN)
- [ ] The Act zone contains exactly one call to the method under test. (EN)
- [ ] The test name follows `Method_Scenario_ExpectedResult` and reads like a sentence. (EN)
- [ ] The most specific `Assert.*` is used for each check. (EN)
- [ ] In `Assert.Equal` the expected value is the first argument. (EN)
- [ ] Behavior is tested, not implementation details. (EN)
- [ ] No unnamed magic numbers — they are extracted into constants or parameters. (EN)

#### Ресурсы / Resources

- Microsoft Learn — https://learn.microsoft.com/dotnet/core/testing/unit-testing-best-practices

---

[⬆ К модулю M15](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
