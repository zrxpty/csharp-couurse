[⬆ К модулю M15](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M15-L08: Покрытие кода (coverlet), отчёты / Code coverage (coverlet), reports

**Модуль / Module:** M15
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

Покрытие кода (code coverage) — это метрика, показывающая, какая доля вашего кода была выполнена во время прогона автоматических тестов. В экосистеме .NET де-факто стандартом для измерения покрытия является **coverlet** — кроссплатформенный коллектор, который работает без полноформатных профайлеров и интегрируется напрямую в `dotnet test`.

coverlet измеряет три основных вида покрытия: **line coverage** (процент выполненных строк), **branch coverage** (процент выполненных ветвей условных переходов `if/switch`), и **method coverage** (процент вызванных методов). Line coverage — самая «честная» и понятная метрика: строка либо выполнилась, либо нет. Branch coverage глубже: даже если обе строки `if` и `else` выполнились в разных тестах, ветвление считается покрытым только тогда, когда проверены все пути решения. Именно поэтому branch coverage обычно ниже line coverage и лучше отражает реальное качество тестов.

Аналогия: представьте дорожную сеть города. Line coverage — это «по какой улице хотя бы раз проехала машина». Branch coverage — «на каждом перекрёстке мы свернули и налево, и направо». Можно проехать по всем улицам, но ни разу не повернуть направо — и ветвления останутся непроверенными.

Установка coverlet в тестовый проект выполняется через NuGet-пакет `coverlet.collector`. После этого команда `dotnet test --collect:"XPlat Code Coverage"` собирает результаты в формате OpenCover и складывает их в папку `TestResults/...`. Чтобы получить читаемые HTML-отчёты, используют **ReportGenerator** — инструмент, который превращает XML-файлы coverlet в наглядные HTML-страницы с подсветкой непокрытых строк прямо в исходниках. Запуск: `dotnet reportgenerator -reports:"**/coverage.cobertura.xml" -targetdir:"coverage-report" -reporttypes:Html`.

Здесь важно упомянуть **миф о 100% покрытии**. Достичь 100% line coverage технически возможно, но это почти всегда плохая идея. Во-первых, покрытие считает *выполнение* строки, а не её *корректность*. Можно написать тест, который просто вызывает метод и игнорирует результат — строка «выполнится», но баг останется. Во-вторых, часть кода по природе плохо тестируется: генераторы, автогенерированный код, защитные ветки `throw new NotImplementedException()` для путей, которые никогда не произойдут. Гоняться за 100% — значит писать пустые тесты, увеличивать время прогона и maintenance-нагрузку без реальной пользы.

Правильное отношение к coverage: **это сигнал, а не цель**. Цель тестов — уверенность, что система ведёт себя правильно. Покрытие показывает, *где тестов нет совсем* — непокрытый код это «слепая зона», в которой могут прятаться баги. Сигнал «модуль X покрыт на 35%» означает «мы почти не знаем, как он работает под нагрузкой». Сигнал «модуль Y покрыт на 95%, но branch только 60%» означает «есть сложные условия, которые мы не проверили со всех сторон». Используйте coverage как компас: выделяйте критичные пути, ставьте пороги (например, 80% для core-логики, без жёсткого требования для UI-адаптеров), отслеживайте деградацию в CI. Хорошая практика — блокировать merge, если покрытие *упало*, а не требовать его повышения ради цифры.

В CI coverage обычно настраивается так: прогон тестов → генерация отчёта → публикация артефакта → комментарий в PR с дельтой по изменению покрытия. Так команда видит, не добавил ли автор новую функциональность без тестов.

#### Theory (EN)

Code coverage is a metric that shows which portion of your code was actually executed during an automated test run. In the .NET ecosystem, the de facto standard for measuring coverage is **coverlet**, a cross-platform collector that works without heavyweight profilers and integrates directly into `dotnet test`.

coverlet measures three main kinds of coverage: **line coverage** (the percentage of executed source lines), **branch coverage** (the percentage of executed conditional branches in `if`/`switch` statements), and **method coverage** (the percentage of methods that were invoked at least once). Line coverage is the most intuitive metric: a line either ran or it did not. Branch coverage goes deeper: even if both the `if` and `else` lines executed across different tests, a branch is only considered covered when every decision path has been taken. That is why branch coverage is usually lower than line coverage and reflects the real quality of your tests far better.

Analogy: imagine a city's road network. Line coverage is "has a car driven on this street at least once". Branch coverage is "at every intersection, did we turn both left and right?". You can drive through every street yet never turn right at a single junction — and the branches remain untested.

Installing coverlet into a test project is done via the `coverlet.collector` NuGet package. After that, the command `dotnet test --collect:"XPlat Code Coverage"` collects results in OpenCover format and drops them into a `TestResults/...` folder. To turn those XML files into readable HTML reports, you use **ReportGenerator**, a tool that transforms coverlet's output into visual HTML pages with uncovered lines highlighted directly in the source code. Invocation: `dotnet reportgenerator -reports:"**/coverage.cobertura.xml" -targetdir:"coverage-report" -reporttypes:Html`.

This is where the **myth of 100% coverage** comes in. Achieving 100% line coverage is technically possible, but it is almost always a bad idea. First, coverage counts the *execution* of a line, not its *correctness*. You can write a test that simply calls a method and ignores the result — the line "executes", but the bug stays. Second, some code is inherently hard to test meaningfully: source generators, auto-generated code, defensive `throw new NotImplementedException()` branches for paths that can never occur. Chasing 100% means writing hollow tests, inflating run time, and increasing maintenance load for no real benefit.

The right attitude toward coverage: **it is a signal, not a goal**. The goal of tests is confidence that the system behaves correctly. Coverage shows *where tests are completely absent* — uncovered code is a "blind spot" where bugs can hide. A signal "module X is 35% covered" means "we barely know how it behaves under load". A signal "module Y is 95% covered, but branches only 60%" means "there are complex conditions we never checked from every side". Use coverage as a compass: identify critical paths, set thresholds (say, 80% for core logic, no hard requirement for UI adapters), and track regressions in CI. A good practice is to block a merge if coverage *dropped*, rather than demanding it increase for the number's sake.

In CI, coverage is typically wired as: run tests → generate report → publish artifact → post a PR comment with the coverage delta. That way the team can immediately see whether an author added new functionality without tests.

#### Пример кода / Code Example

```csharp
// C# 12 / .NET 8 — пример класса с бизнес-логикой и тестами с покрытием
// Example class with business logic and tests with coverage tracking

namespace Shop.Pricing;

// Рассчитывает итоговую цену со скидкой и налогом
// Computes the final price including discount and tax
public sealed class PriceCalculator
{
    private const decimal TaxRate = 0.20m; // 20% НДС / 20% VAT

    public decimal Compute(decimal unitPrice, int quantity, decimal discountPercent)
    {
        // Защитные проверки — эти ветки НЕ нужно гнать ради 100% coverage
        // Guard checks — do not chase these branches for 100% coverage
        if (unitPrice < 0)
            throw new ArgumentOutOfRangeException(nameof(unitPrice), "Цена не может быть отрицательной / Price cannot be negative");
        if (quantity <= 0)
            throw new ArgumentOutOfRangeException(nameof(quantity), "Количество должно быть положительным / Quantity must be positive");
        if (discountPercent is < 0 or > 100)
            throw new ArgumentOutOfRangeException(nameof(discountPercent), "Скидка вне диапазона / Discount out of range");

        decimal subtotal = unitPrice * quantity;                 // Подытог / Subtotal
        decimal discount = subtotal * (discountPercent / 100m);  // Скидка / Discount
        decimal taxable = subtotal - discount;                   // База для налога / Taxable base
        decimal tax = taxable * TaxRate;                         // Налог / Tax

        // Итоговая цена — ветвление важно покрыть ОБА пути тестами
        // Final price — both branches must be covered by tests
        return discountPercent > 0
            ? Math.Round(taxable + tax, 2)
            : Math.Round(subtotal + tax, 2);
    }
}

// Тесты для xUnit — coverlet соберёт метрики при `dotnet test --collect:"XPlat Code Coverage"`
// xUnit tests — coverlet collects metrics when running `dotnet test --collect:"XPlat Code Coverage"`
public sealed class PriceCalculatorTests
{
    private readonly PriceCalculator _calc = new();

    [Fact]
    public void Compute_WithDiscount_ReturnsTaxedRoundedTotal()
    {
        // Arrange: цена 100, кол-во 3, скидка 10%
        // Arrange: price 100, qty 3, discount 10%
        decimal result = _calc.Compute(unitPrice: 100m, quantity: 3, discountPercent: 10m);

        // subtotal = 300, discount = 30, taxable = 270, tax = 54, total = 324
        Assert.Equal(324m, result);
    }

    [Fact]
    public void Compute_WithoutDiscount_ReturnsTaxedRoundedTotal()
    {
        // Второй путь ветвления — без скидки, иначе branch coverage будет неполным
        // Second branch path — without discount, otherwise branch coverage is incomplete
        decimal result = _calc.Compute(unitPrice: 50m, quantity: 2, discountPercent: 0m);

        // subtotal = 100, taxable = 100, tax = 20, total = 120
        Assert.Equal(120m, result);
    }

    [Theory]
    [InlineData(-1, 1, 0)]     // отрицательная цена / negative price
    [InlineData(10, 0, 0)]     // нулевое количество / zero quantity
    [InlineData(10, 1, 150)]   // скидка > 100 / discount over 100
    public void Compute_InvalidInput_Throws(decimal price, int qty, decimal discount)
    {
        // Эти проверки важны для безопасности API, но не для «красоты» процента
        // These checks matter for API safety, not for the coverage percentage
        Assert.Throws<ArgumentOutOfRangeException>(() => _calc.Compute(price, qty, discount));
    }
}
```

#### Best Practices

- Используйте coverage как **сигнал для ревью**, а не как KPI, который нужно «выполнить». Непокрытый код — повод задать вопрос автору, а не причина автоматически блокировать PR.
- В CI отслеживайте **дельту покрытия**, а не абсолют. Падение на 5% в новом PR — тревожнее, чем общий уровень 70%.
- Ставьте разные пороги для разных слоёв: core-доменная логика — 80–90%, инфраструктура и адаптеры — мягче, автогенерированный код — исключайте через `ExcludeFromCodeCoverage`.
- Смотрите на **branch coverage** чаще, чем на line coverage — он честнее отражает качество тестов для сложных условий.
- Исключайте из отчёта сгенерированный код, миграции и стартап-бойлерплейт, иначе цифры будут вводить в заблуждение.

- Use coverage as a **review signal**, not a KPI to "hit". Uncovered code is a reason to ask the author a question, not an automatic PR block.
- Track the **coverage delta** in CI rather than the absolute number. A 5% drop in a new PR is more alarming than a steady 70% baseline.
- Set different thresholds per layer: core domain logic at 80–90%, infrastructure and adapters more lenient, autogenerated code excluded via `ExcludeFromCodeCoverage`.
- Look at **branch coverage** more often than line coverage — it reflects real test quality for complex conditions far more honestly.
- Exclude generated code, migrations, and startup boilerplate from reports, or the numbers will mislead you.

#### Частые ошибки / Common Mistakes

- Гнаться за 100% coverage, писать пустые тесты ради процента → Покрытие не равно корректность. Тестируйте поведение и assert'ы, а не факт вызова строки.
- Игнорировать branch coverage и смотреть только на line coverage → Сложные `if` могут иметь 100% строк, но 50% веток. Включайте branch-метрику в отчёты по умолчанию.
- Включать в отчёт автогенерированный код и миграции → Это занижает осмысленные цифры и отвлекает внимание. Исключайте через фильтры coverlet.
- Блокировать merge по абсолютному порогу без учёта контекста → Одна непокрытая строка в UI-адаптере не должна ломать CI для core-логики. Применяйте пороги по слоям.
- Доверять coverage как гаранту отсутствия багов → coverage показывает «где тестов нет», а не «где тесты правильные». Дополняйте мутационное тестирование и code review.

- Chasing 100% coverage with hollow tests written for the percentage → Coverage is not correctness. Test behavior and assertions, not the fact that a line was called.
- Ignoring branch coverage and looking only at line coverage → A complex `if` can have 100% lines but 50% branches. Enable the branch metric in reports by default.
- Including autogenerated code and migrations in the report → This drags down meaningful numbers and distracts attention. Exclude via coverlet filters.
- Blocking merges on an absolute threshold without context → One uncovered line in a UI adapter should not break CI for core logic. Apply thresholds per layer.
- Trusting coverage as a guarantee of bug-free code → Coverage shows "where tests are missing", not "where tests are correct". Supplement with mutation testing and code review.

#### Чек-лист самопроверки / Self-check Checklist

- [ ] В тестовый проект добавлен NuGet-пакет `coverlet.collector`.
- [ ] Команда `dotnet test --collect:"XPlat Code Coverage"` создаёт XML-файл с метриками.
- [ ] ReportGenerator установлен и генерирует HTML-отчёт с подсветкой непокрытых строк.
- [ ] В CI настроена публикация отчёта и комментарий с дельтой покрытия в PR.
- [ ] Пороги coverage заданы по слоям (core / инфраструктура / UI), а не единым числом.
- [ ] Автогенерированный код исключён из отчёта через `ExcludeFromCodeCoverage` или фильтры.
- [ ] Команда понимает разницу между line coverage и branch coverage и смотрит обе метрики.
- [ ] Coverage используется как сигнал для ревью, а не как цель «догнать до 100%».

- [ ] The test project references the `coverlet.collector` NuGet package.
- [ ] The command `dotnet test --collect:"XPlat Code Coverage"` produces an XML file with metrics.
- [ ] ReportGenerator is installed and produces an HTML report highlighting uncovered lines.
- [ ] CI is configured to publish the report and post a coverage-delta comment on the PR.
- [ ] Coverage thresholds are defined per layer (core / infrastructure / UI), not as a single number.
- [ ] Autogenerated code is excluded from the report via `ExcludeFromCodeCoverage` or filters.
- [ ] The team understands the difference between line coverage and branch coverage and checks both.
- [ ] Coverage is used as a review signal, not as a "reach 100%" goal.

#### Ресурсы / Resources

- [coverlet-coverage/coverlet — GitHub — https://github.com/coverlet-coverage/coverlet]
- [ReportGenerator — https://github.com/danielpalme/ReportGenerator]
- [Microsoft Learn — Unit test coverage — https://learn.microsoft.com/dotnet/core/testing/unit-testing-code-coverage]

---

[⬆ К модулю M15](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
