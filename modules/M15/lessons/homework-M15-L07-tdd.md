---
[← К уроку M15-L07](lesson-M15-L07-tdd.md) | [⬆ К модулю M15](../README.md) | [Следующее ДЗ →](homework-M15-L08-coverage-coverlet.md)
---

### Домашнее задание M15-L07: TDD, red-green-refactor / Homework M15-L07: TDD, red-green-refactor

**Урок / Lesson:** M15-L07
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 4/5
**Цель / Goal:** (RU) Научиться вести разработку строгим циклом red-green-refactor на реальной доменной задаче: спроектировать через тесты класс `InvoiceCalculator`, который считает сумму счёта со скидками и порогами, и провести рефакторинг, не ломая зелёные тесты. (EN) Learn to drive development with a strict red-green-refactor cycle on a realistic domain task: design an `InvoiceCalculator` class through tests that computes invoice totals with discounts and thresholds, and refactor it without breaking green tests.

#### Связь с уроком / Connection to the lesson
(RU) Урок вводит цикл red-green-refactor, подчёркивает, что красным должен быть Assert, а не ошибка компиляции, и что green должен быть минимальным и намеренно «грязным», а красоту наводят только на refactor при зелёных тестах. В этом ДЗ вы примените все эти принципы на доменной задаче расчёта счёта: пройдёте через несколько циклов, поймаете дизайн-запах «трудно тестировать» и уберёте дублирование стратегией ценообразования, как в примере `Basket`/`IPricing` из урока.
(EN) The lesson introduces the red-green-refactor loop, stresses that the red must be an assertion failure rather than a compile error, and that green must be minimal and intentionally ugly, with beauty added only during refactor while tests stay green. In this homework you will apply all of these principles to a domain invoice-calculation task: you will go through several cycles, encounter the "hard to test" design smell, and remove duplication with a pricing strategy, mirroring the `Basket`/`IPricing` example from the lesson.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация

Вы — backend-разработчик биллингового модуля интернет-магазина. Продакт-менеджер принёс требования к новому правилу расчёта итоговой суммы счёта: в счёт можно добавить несколько строк с товаром (SKU, количество, цена за единицу), к строкам применяются пороговые скидки за объём, а к итогу счёта — прогрессивная скидка за общую сумму. Требования сформулированы как набор бизнес-правил с конкретными числами, то есть поведение детерминированное и хорошо специфицируемое — идеальный кандидат для TDD, как прямо сказано в уроке: «бизнес-логика, денежные расчёты, доменные правила».

Ваша команда недавно перешла на TDD и столкнулась с двумя типичными проблемами: иногда разработчики пишут сразу несколько падающих тестов и потом путаются, какой именно код делает тест зелёным; иногда шаг green растягивается на час, потому что вместо минимального «грязного» решения сразу пишут «красивое». В этом задании вы должны строго соблюсти дисциплину из урока: один красный тест за шаг, минимальный green, refactor только при зелёных тестах, цикл короче пяти-десяти минут. Денежные расчёты — область, где ошибка на копейку критична, поэтому красные тесты должны быть точными, а зеленые решения — детерминированными, без работы с реальной базой или сетью.

Дополнительно, по ходу развития спецификации вы заметите, что три правила скидки выглядят похоже и порождают дублирование. Урок прямо указывает, что именно здесь рождается обобщение через параметризацию или стратегию: на этапе refactor, при зелёных тестах, вы должны «выдавить» интерфейс `IPriceRule` из реального использования, как в примере `IPricing` из урока. Так вы не только реализуете функциональность, но и покажете, как TDD заставляет дизайн расти органически, а не из прогнозов.

#### Что нужно сделать (пошагово)

1. **Создайте solution и три проекта** в пустой папке `InvoiceTdd`. Используйте .NET 8 и C# 12. Команды:
   ```
   dotnet new sln -n InvoiceTdd
   dotnet new classlib -n Invoice.Domain -f net8.0
   dotnet new xunit -n Invoice.Tests -f net8.0
   dotnet sln add Invoice.Domain/Invoice.Domain.csproj
   dotnet sln add Invoice.Tests/Invoice.Tests.csproj
   dotnet add Invoice.Tests/Invoice.Domain.csproj reference Invoice.Domain/Invoice.Domain.csproj
   ```
   В `Invoice.Domain.csproj` установите `<LangVersion>latest</LangVersion>` и `<Nullable>enable</Nullable>`. Убедитесь, что `dotnet build` проходит без ошибок.

2. **Цикл RED 1.** В проекте тестов создайте файл `InvoiceCalculatorTests.cs`. Напишите первый тест: для пустого счёта (без строк) итог равен нулю. Запустите `dotnet test` — он должен упасть с ошибкой компиляции, потому что `InvoiceCalculator` ещё не существует. Урок предупреждает: ошибка компиляции — это не настоящий red. Поэтому немедленно создайте класс-заглушку `InvoiceCalculator` в `Invoice.Domain`, чтобы тест компилировался, и перезапустите `dotnet test`. Теперь красным должен быть именно `Assert.Equal(0m, calc.Total)` — утверждение, а не компиляция.

3. **Цикл GREEN 1.** Сделайте минимальный, намеренно «грязный» green: верните `0m` из свойства `Total`. Урок прямо разрешает возвращать константу на первом шаге. Запустите `dotnet test` — зелёный. Не обобщайте пока на список строк: обобщение прийдёт на следующем red.

4. **Цикл RED 2.** Напишите второй тест: добавьте одну строку с SKU `"A1"`, количеством 2 и ценой 10m, итог должен быть 20m. Запустите — красный (Assertion, не компиляция). GREEN 2: добавьте метод `AddLine(string sku, int qty, decimal unitPrice)` и простейшую логику `Total` через сумму по одной строке. Можно пока хранить ровно одну строку — тест зелёный, и это допустимо.

5. **Цикл RED 3.** Третий тест: две разные строки `"A1"` и `"B2"`, итог должен быть суммой обеих. Теперь однострочного хранения недостаточно — green заставит вас завести `List<InvoiceLine>`. Сделайте это минимально. Тест зелёный.

6. **Цикл RED 4.** Тест: тот же SKU добавляется дважды, строки должны объединяться (как `Same_sku_merges_into_one_line` из урока). GREEN 4: найдите существующую строку по SKU и прибавьте количество. Рефакторить не нужно — просто зелейте тест.

7. **Цикл RED 5 — пороговая скидка за объём.** Бизнес-правило: если количество строки ≥ 10, к этой строке применяется скидка 5%. Тест: `AddLine("A1", 10, 10m)` → итог 95m (100 − 5%). RED 5 зелёного решения нет — вам нужно условие в расчёте строки. GREEN 5: добавьте `if (qty >= 10) subtotal *= 0.95m`. Грязно, но зелёно.

8. **Цикл RED 6 — второй порог.** Правило: при количестве ≥ 50 скидка 10%. Тест: `AddLine("A1", 50, 10m)` → 450m. GREEN 6: ещё один `if`. Теперь у вас дублирование логики скидок — это сигнал к refactor.

9. **REFACTOR 6.** При зелёных тестах выделите интерфейс `IPriceRule { decimal LineTotal(int qty, decimal unitPrice); }` и две реализации: `StandardRule` (без скидки) и `VolumeDiscountRule(int threshold, decimal rate)`. Зарегистрируйте правила как список в `InvoiceCalculator` и выбирайте первое подходящее по порогу. Запустите `dotnet test` — все зелёные. Это шаг рефакторинга из урока: нового поведения нет, только устранение дублирования, тесты зелёные всё время.

10. **Цикл RED 7 — скидка за итог счёта.** Правило: если `Subtotal` (сумма после строковых скидок) ≥ 1000m, применяется дополнительная скидка 3% ко всему счёту. Тест: 100 строк по 1 шт. по 10m → subtotal 1000m → итог 970m. GREEN 7: простой `if (subtotal >= 1000m) subtotal *= 0.97m`. REFACTOR 7: выделите `IInvoiceDiscount` стратегию аналогично.

11. **Финальная проверка.** Запустите `dotnet test -v n` и убедитесь, что все тесты зелёные, а в выводе видно их имена. Запустите `dotnet build -warnaserror` — никаких предупреждений. Сдайте `.csproj` и `.cs` файлы, а также короткий `WALKTHROUGH.md` с описанием циклов red-green-refactor в формате «RED N: что проверял → GREEN N: что добавил → REFACTOR N: что убрал».

#### Требования к решению

- Целевая платформа строго .NET 8, язык C# 12: используйте top-level statements в `Program.cs` (если он нужен для демо), collection expressions (`[]`), pattern matching (`is`, switch expressions), file-scoped namespaces, `sealed` классы, `init`-свойства там, где уместно. Для денежных значений — только `decimal`, никаких `double` или `float`.
- Структура решения: библиотека классов `Invoice.Domain` (продакшн-код) и xUnit-проект `Invoice.Tests` (тесты). Тесты не должны обращаться к базе данных, сети, файловой системе или `DateTime.Now` — все зависимости изолируйте за интерфейсами и заменяйте fakes или детерминированными значениями, как требует урок.
- Строгое соблюдение цикла: один падающий тест за шаг, минимальный green без лишнего поведения, refactor только при зелёных тестах. Запрещено писать несколько красных тестов одновременно — это частая ошибка из урока. Запрещено добавлять новое поведение на этапе refactor.
- Все скидки и правила должны быть параметризованы через конструкторы или интерфейсы, чтобы их можно было подменять в тестах. Должны быть реализованы: пороговая скидка за объём (два порога: 10 шт. → 5%, 50 шт. → 10%) и скидка за итог счёта (1000m → 3%). Класс `InvoiceCalculator` должен иметь методы `AddLine` и свойство `Total`.
- Тесты должны покрывать: пустой счёт, одну строку, несколько разных строк, объединение одинаковых SKU, оба порога объёмной скидки, скидку за итог, комбинацию строковой и итоговой скидки, граничные значения (ровно 10 шт., ровно 50 шт., ровно 1000m). Минимум 10 тестов, каждый с говорящим именем вида `Total_is_X_when_Y` или русским эквивалентом — но имя метода на английском для совместимости с runner-ами.
- Код должен собираться с `-warnaserror` без предупреждений. В `csproj` включите `<TreatWarningsAsErrors>true</TreatWarningsAsErrors>`. Используйте `sealed` на классах, чтобы явно запретить наследование, если оно не планируется — это best practice для доменных моделей.

#### Тонкости и подводные камни

- **Красный — это Assert, а не компиляция.** Самая частая ошибка новичков: создать тест на несуществующий класс и считать ошибку компиляции «красным». Урок прямо говорит, что такой шаг растягивается и ломает ритм. Сначала создайте класс-заглушку, чтобы тест компилировался, и только тогда настоящий красный — упавшее утверждение.
- **Минимальный green может быть константой.** На первом цикле возвращение `0m` из `Total` — это корректный green, а не «обман». Урок это разрешает. Ошибка — оставить константу навсегда; обобщение приходит на следующем red, когда тест требует другого значения.
- **Не пишите несколько красных тестов сразу.** Если упало два теста, непонятно, какой код относился к какому. Один red → один green → refactor — это ритм из урока. «Несколько падающих тестов одновременно» — явная частая ошибка.
- **Refactor только на зелёном.** Никогда не меняйте структуру кода, если хоть один тест красный. Сначала верните зелёный, потом наводите красоту. Рефакторинг на красном — частая ошибка, ведущая к потере поведения.
- **Денежные расчёты — только `decimal`.** Использование `double` приведёт к ошибкам округления (0.1 + 0.2 ≠ 0.3). Все цены, количества (если они целые — `int`, если дробные — тоже `decimal`), скидки — `decimal`. Сравнения делайте через `Assert.Equal(expected, actual)` без точности, потому что `decimal` точен.
- **Граничные значения — отдельные тесты.** «Ровно 10 шт.» — это граница порога, поведение переключается. Тест на 9 шт. (без скидки) и на 10 шт. (со скидкой) — это два разных теста, не один параметризованный «в районе порога». Урок поощряет маленькие шаги, и границы — классическое место, где «минимальный green» ловит off-by-one.
- **Дублирование — сигнал к стратегии.** Когда второй `if` скидки дублирует первый, урок говорит: «появление второго-третьего теста часто вскрывает дублирование, которое убирается рефакторингом». Не оставляйте два `if` — выделите `IPriceRule`. Это и есть «выдавливание интерфейса из реального использования».
- **Не обобщайте раньше времени.** Не заводите `IPriceRule` на первом цикле, когда нужно только вернуть `0m`. Стратегия рождается на refactor после красного, вскрывшего дублирование, а не из прогноза «когда-нибудь будут скидки». Урок явно критикует проектирование «из прогнозов».
- **Имена тестов — документация.** `Total_is_970_when_subtotal_is_1000_and_threshold_discount_applies` длинное, но читается как спецификация. Избегайте `Test1`, `TestAdd`. Урок рассматривает тесты как «леса», которые остаются навсегда, — они должны быть читаемы.
- **Вне TDD — не притворяйтесь.** Если вы делаете spike по исследованию новой библиотеки, не пишите тесты «для галочки». Урок разделяет: спайк без тестов и выбрасывается, потом переписывается через TDD. В этом задании вы внутри TDD, но в `WALKTHROUGH.md` отметьте, где вы сознательно могли бы быть «вне TDD».

#### Критерии приёмки

- [ ] Создан solution `InvoiceTdd` с проектами `Invoice.Domain` (classlib, net8.0) и `Invoice.Tests` (xunit, net8.0), ссылка тестов на домен настроена.
- [ ] `dotnet build` и `dotnet test` проходят без ошибок и предупреждений (`-warnaserror` / `TreatWarningsAsErrors=true`).
- [ ] Целевая платформа net8.0, C# 12 (`LangVersion=latest`), `Nullable=enable`, file-scoped namespaces, `sealed` классы.
- [ ] Денежные значения — только `decimal`; нигде нет `double`/`float` для денег.
- [ ] Реализован `InvoiceCalculator` с методами `AddLine(string sku, int qty, decimal unitPrice)` и свойством `Total`.
- [ ] Поддержка пустого счёта (Total = 0), одной строки, нескольких строк, объединения одинаковых SKU.
- [ ] Пороговая объёмная скидка: 10 шт. → 5%, 50 шт. → 10%; граничные значения 9/10/49/50 покрыты тестами.
- [ ] Скидка за итог счёта: subtotal ≥ 1000m → 3%; граничное значение ровно 1000m покрыто тестом.
- [ ] Выделен интерфейс `IPriceRule` с минимум двумя реализациями; `InvoiceCalculator` использует список правил.
- [ ] Выделен интерфейс `IInvoiceDiscount` (или эквивалент) для скидки за итог; стратегия подменяема.
- [ ] Минимум 10 xUnit-тестов с говорящими именами, каждый тестирует одно поведение.
- [ ] Тесты не обращаются к БД, сети, файлам, `DateTime.Now`; все зависимости изолированы.
- [ ] В `WALKTHROUGH.md` описаны циклы red-green-refactor по шагам (RED/GREEN/REFACTOR N с пояснением).
- [ ] В коде нет «мёртвых» веток, неиспользуемых параметров, закомментированного кода; `sealed` на доменных классах.
- [ ] Соблюдён ритм: шаги короткие (≤ 5–10 минут на цикл), один красный тест за шаг, green минимальный.

#### Подсказки (без прямого ответа)

- Если на RED 1 вы получаете ошибку компиляции вместо Assertion — это не настоящий red. Создайте класс-заглушку с пустым телом свойства и перезапустите тесты.
- Для GREEN 1 возвращать `0m` буквально — нормально. Не пишите список строк, пока второй тест не заставит.
- Когда появляется второй `if` для скидки — остановитесь и спросите: «какой интерфейс спрятан за этим дублированием?».
- Граница порога — это `qty >= threshold`, не `qty > threshold`. Подумайте, какой тест поймает off-by-one.
- Для скидки за итог: сначала считаете subtotal по строкам (со строковыми скидками), потом применяете скидку ко всему subtotal. Не путайте порядок.
- Используйте collection expressions для инициализации списка правил: `new List<IPriceRule> { ... }` или `[ ]`-синтаксис C# 12, где уместно.
- В тестах применяйте `[Theory]` и `[InlineData]` только для действительно однотипных случаев (например, несколько порогов объёмной скидки), но граничные значения держите отдельными `[Fact]` для читаемости.

#### Эталонное решение (разбор)

```csharp
// C# 12 / .NET 8 — Invoice.Domain/InvoiceCalculator.cs
// C# 12 / .NET 8 — Invoice.Domain/InvoiceCalculator.cs
namespace Invoice.Domain;

// IPriceRule — стратегия расчёта суммы строки со скидкой
// IPriceRule — strategy for computing a discounted line total
public interface IPriceRule
{
    // Возвращает true, если правило применимо к данному количеству
    // Returns true when the rule applies to the given quantity
    bool AppliesTo(int qty);
    decimal LineTotal(int qty, decimal unitPrice);
}

// Стандартное правило без скидки — базовый случай
// Standard rule with no discount — the base case
public sealed class StandardRule : IPriceRule
{
    public bool AppliesTo(int qty) => true;               // всегда применимо как fallback
                                                          // always applies as a fallback
    public decimal LineTotal(int qty, decimal unitPrice) => qty * unitPrice;
}

// Пороговая скидка за объём: при qty >= threshold применяется rate
// Volume discount: when qty >= threshold a rate is applied
public sealed class VolumeDiscountRule(int threshold, decimal rate) : IPriceRule
{
    public bool AppliesTo(int qty) => qty >= threshold;
    public decimal LineTotal(int qty, decimal unitPrice)
    {
        var baseTotal = qty * unitPrice;
        return baseTotal * (1m - rate);                  // скидка применяется ко всей строке
                                                        // discount applied to the whole line
    }
}

// IInvoiceDiscount — стратегия скидки на итог счёта
// IInvoiceDiscount — strategy for a discount on the invoice subtotal
public interface IInvoiceDiscount
{
    bool AppliesTo(decimal subtotal);
    decimal Apply(decimal subtotal);
}

public sealed class TotalThresholdDiscount(decimal threshold, decimal rate) : IInvoiceDiscount
{
    public bool AppliesTo(decimal subtotal) => subtotal >= threshold;
    public decimal Apply(decimal subtotal) => subtotal * (1m - rate);
}

public sealed class InvoiceLine
{
    public string Sku { get; }
    public int Quantity { get; private set; }
    public decimal UnitPrice { get; }

    public InvoiceLine(string sku, int qty, decimal unitPrice)
    {
        Sku = sku;
        Quantity = qty;
        UnitPrice = unitPrice;
    }

    public void AddQuantity(int qty) => Quantity += qty;
}

// InvoiceCalculator — агрегирует строки и применяет правила
// InvoiceCalculator — aggregates lines and applies rules
public sealed class InvoiceCalculator
{
    private readonly List<InvoiceLine> _lines = [];
    private readonly IPriceRule[] _lineRules;
    private readonly IInvoiceDiscount[] _invoiceDiscounts;

    // Правила передаются через конструктор — тестируемость и подмена
    // Rules are injected through the constructor — testability and substitution
    public InvoiceCalculator(
        IPriceRule[]? lineRules = null,
        IInvoiceDiscount[]? invoiceDiscounts = null)
    {
        _lineRules = lineRules ?? [new VolumeDiscountRule(50, 0.10m),
                                   new VolumeDiscountRule(10, 0.05m),
                                   new StandardRule()];
        _invoiceDiscounts = invoiceDiscounts ?? [new TotalThresholdDiscount(1000m, 0.03m)];
    }

    public void AddLine(string sku, int qty, decimal unitPrice)
    {
        // RED 4: одинаковый SKU объединяется в одну строку
        // RED 4: same SKU is merged into a single line
        var existing = _lines.FirstOrDefault(l => l.Sku == sku);
        if (existing is not null)
        {
            existing.AddQuantity(qty);
            return;
        }
        _lines.Add(new InvoiceLine(sku, qty, unitPrice));
    }

    public decimal Total
    {
        get
        {
            // Сначала считаем строки с применением первого подходящего правила
            // First compute lines applying the first matching rule
            decimal subtotal = 0m;
            foreach (var line in _lines)
            {
                var rule = _lineRules.First(r => r.AppliesTo(line.Quantity));
                subtotal += rule.LineTotal(line.Quantity, line.UnitPrice);
            }

            // Затем применяем скидку на итог счёта
            // Then apply the invoice-level discount
            foreach (var d in _invoiceDiscounts)
            {
                if (d.AppliesTo(subtotal))
                {
                    subtotal = d.Apply(subtotal);
                }
            }
            return subtotal;
        }
    }
}
```

```csharp
// C# 12 / .NET 8 — Invoice.Tests/InvoiceCalculatorTests.cs
namespace Invoice.Tests;

using Invoice.Domain;
using Xunit;

public class InvoiceCalculatorTests
{
    private static InvoiceCalculator NewCalc() => new();

    [Fact]
    public void Total_is_zero_for_empty_invoice() // RED 1
        => Assert.Equal(0m, NewCalc().Total);

    [Fact]
    public void Total_is_price_times_quantity_for_single_line() // RED 2
    {
        var calc = NewCalc();
        calc.AddLine("A1", 2, 10m);
        Assert.Equal(20m, calc.Total);
    }

    [Fact]
    public void Total_sums_two_different_lines() // RED 3
    {
        var calc = NewCalc();
        calc.AddLine("A1", 1, 10m);
        calc.AddLine("B2", 1, 5m);
        Assert.Equal(15m, calc.Total);
    }

    [Fact]
    public void Same_sku_merges_into_one_line() // RED 4
    {
        var calc = NewCalc();
        calc.AddLine("A1", 1, 10m);
        calc.AddLine("A1", 1, 10m);
        Assert.Equal(20m, calc.Total);
    }

    [Fact]
    public void Volume_discount_5pct_applies_at_qty_10() // RED 5
    {
        var calc = NewCalc();
        calc.AddLine("A1", 10, 10m);
        Assert.Equal(95m, calc.Total); // 100 - 5%
    }

    [Fact]
    public void No_volume_discount_below_threshold_qty_9() // граница
    {
        var calc = NewCalc();
        calc.AddLine("A1", 9, 10m);
        Assert.Equal(90m, calc.Total);
    }

    [Fact]
    public void Volume_discount_10pct_applies_at_qty_50() // RED 6
    {
        var calc = NewCalc();
        calc.AddLine("A1", 50, 10m);
        Assert.Equal(450m, calc.Total); // 500 - 10%
    }

    [Fact]
    public void Invoice_discount_3pct_applies_at_subtotal_1000() // RED 7
    {
        var calc = NewCalc();
        for (int i = 0; i < 100; i++)
            calc.AddLine($"SKU-{i}", 1, 10m);
        Assert.Equal(970m, calc.Total); // 1000 - 3%
    }

    [Fact]
    public void No_invoice_discount_below_1000_subtotal() // граница итога
    {
        var calc = NewCalc();
        for (int i = 0; i < 99; i++)
            calc.AddLine($"SKU-{i}", 1, 10m);
        Assert.Equal(990m, calc.Total);
    }

    [Fact]
    public void Volume_and_invoice_discounts_combine() // интеграционный
    {
        var calc = NewCalc();
        calc.AddLine("A1", 10, 100m); // 1000 - 5% = 950
        Assert.Equal(950m, calc.Total); // 950 < 1000 → no invoice discount
    }

    [Fact]
    public void Custom_rules_can_be_injected() // подмена стратегий
    {
        var calc = new InvoiceCalculator(
            lineRules: [new StandardRule()],
            invoiceDiscounts: [new TotalThresholdDiscount(0m, 0.50m)]); // 50% ко всему
        calc.AddLine("A1", 1, 10m);
        Assert.Equal(5m, calc.Total);
    }
}
```

**Разбор по строкам.** Первая группа циклов (RED 1 → GREEN 1 → RED 2 → GREEN 2) воспроизводит классический TDD-ритм из урока: пустой счёт даёт ноль, потом одна строка даёт цену на количество. Возврат `0m` на GREEN 1 — это намеренно «грязный» green, который урок прямо разрешает; обобщение приходит на RED 2, когда тест требует другого значения. RED 3 заставляет перейти от одной строки к `List<InvoiceLine>` — это шаг, на котором дизайн «выдавливается» из реального использования, а не из прогноза: мы не заводили список заранее, он появился, когда этого потребовал тест.

RED 4 (объединение SKU) повторяет пример `Same_sku_merges_into_one_line` из урока почти один-в-один: поиск существующей строки через `FirstOrDefault` и `AddQuantity`. RED 5 и RED 6 вводят два порога скидки — и вот здесь возникает дублирование, о котором предупреждает урок: два `if (qty >= ...)` в расчёте строки. Вместо того чтобы оставить их, мы на REFACTOR 6 выделяем `IPriceRule` с `AppliesTo` и `LineTotal`, ровно как `IPricing` из примера `Basket`. Порядок правил в массиве важен: сначала более специфичное (50 шт. → 10%), потом менее (10 шт. → 5%), потом `StandardRule` как fallback с `AppliesTo => true` — это паттерн "first match wins", хорошо читаемый в `Total`.

RED 7 вводит скидку на итог счёта — это уже другой уровень абстракции, поэтому для него заводится отдельный интерфейс `IInvoiceDiscount`, а не переиспользуется `IPriceRule`: разные правила работают с разными агрегатами (строка против итог). Урок учит: не обобщайте раньше времени, но когда дублирование есть — убирайте. Здесь дублирования между уровнями нет, поэтому два интерфейса оправданы. Конструктор `InvoiceCalculator` принимает массивы правил с дефолтами через `??` и collection expressions `[]` — это C# 12, и одновременно это точка подмены для тестов: последний тест `Custom_rules_can_be_injected` доказывает, что стратегии подменяемы, а значит дизайн тестопригоден, как требует урок. `sealed` на всех классах — best practice для доменных моделей: запрет наследования без явной необходимости. `decimal` везде — деньги нельзя считать в `double`. Граничные тесты (9 шт., 99 строк) ловят off-by-one, который «минимальный green» часто пропускает.

#### Задания на углубление (бонус)

1. **Параметризуйте пороги через конфигурацию.** Вместо хардкода `10/50/1000` загружайте правила из `appsettings.json` или из словаря, передаваемого в конструктор. Напишите тест, который проверяет, что конфигурация корректно материализуется в `IPriceRule[]`.
2. **Добавьте прогрессивную скидку по кумулятивному количеству SKU.** Если суммарное количество одного SKU по всем добавлениям (даже в разных строках-счёта, если вы введёте сессии) ≥ 100, применяется дополнительная скидка 2%. Напишите красный тест, потом green, потом refactor, чтобы не дублировать логику с `VolumeDiscountRule`.
3. **Внедрите `IClock` для скидок «по времени».** Добавьте правило: скидка 1% действует только в выходные. Введите интерфейс `IClock { DateTime UtcNow { get; } }`, в тестах передавайте fake с фиксированной датой. Урок требует изолировать время за интерфейсом — это и есть применение.
4. **Mutation testing.** Запустите `dotnet stryker` (если доступно) на проекте тестов и убедитесь, что мутации вроде `>=` → `>`, `1m - rate` → `1m + rate` убиваются тестами. Если какой-то мутант выживает — добавьте красный тест, который его поймает.

---

## Statement in English / Постановка на английском

#### Context & motivation

You are a backend engineer on the billing module of an online shop. The product manager has brought requirements for a new invoice-total calculation rule: an invoice may contain several line items (SKU, quantity, unit price), the lines are subject to volume-threshold discounts, and the invoice total is subject to a progressive discount based on the overall amount. The requirements are stated as a set of business rules with concrete numbers — deterministic behavior with crisp rules — which is exactly the kind of place the lesson says TDD fits: "business logic, money math, domain rules."

Your team recently adopted TDD and has run into two recurring problems: sometimes developers write several failing tests at once and then lose track of which code made which test green; sometimes the green step stretches to an hour because, instead of a minimal "ugly" solution, they immediately write the "beautiful" one. In this assignment you must strictly observe the discipline taught in the lesson: one red test per step, minimal green, refactor only while tests are green, and a loop shorter than five to ten minutes. Money math is an area where a one-cent error is critical, so red tests must be precise and green solutions must be deterministic, with no real database or network access.

Additionally, as the specification evolves you will notice that the three discount rules look alike and create duplication. The lesson explicitly says this is where generalization is born through parameterization or a strategy: during the refactor step, with tests green, you must "squeeze out" an `IPriceRule` interface from real usage, exactly like the `IPricing` example in the lesson. That way you not only implement the feature but also demonstrate how TDD makes design grow organically rather than from forecasts.

#### What to do step by step

1. **Create a solution and three projects** in an empty folder `InvoiceTdd`. Use .NET 8 and C# 12. Commands:
   ```
   dotnet new sln -n InvoiceTdd
   dotnet new classlib -n Invoice.Domain -f net8.0
   dotnet new xunit -n Invoice.Tests -f net8.0
   dotnet sln add Invoice.Domain/Invoice.Domain.csproj
   dotnet sln add Invoice.Tests/Invoice.Tests.csproj
   dotnet add Invoice.Tests/Invoice.Domain.csproj reference Invoice.Domain/Invoice.Domain.csproj
   ```
   In `Invoice.Domain.csproj` set `<LangVersion>latest</LangVersion>` and `<Nullable>enable</Nullable>`. Verify that `dotnet build` succeeds with no errors.

2. **RED 1 cycle.** In the test project create `InvoiceCalculatorTests.cs`. Write the first test: for an empty invoice (no lines) the total equals zero. Run `dotnet test` — it should fail to compile because `InvoiceCalculator` does not exist yet. The lesson warns: a compile error is not a real red. So immediately create a stub class `InvoiceCalculator` in `Invoice.Domain` so the test compiles, and rerun `dotnet test`. Now the red must be exactly `Assert.Equal(0m, calc.Total)` — the assertion, not the compilation.

3. **GREEN 1 cycle.** Make the minimal, intentionally "ugly" green: return `0m` from the `Total` property. The lesson explicitly permits returning a constant on the first step. Run `dotnet test` — green. Do not generalize to a list of lines yet: generalization comes on the next red.

4. **RED 2 cycle.** Write a second test: add a single line with SKU `"A1"`, quantity 2 and unit price 10m; the total must be 20m. Run — red (assertion, not compilation). GREEN 2: add an `AddLine(string sku, int qty, decimal unitPrice)` method and a minimal `Total` over a single line. You may store just one line for now — the test is green, and that is acceptable.

5. **RED 3 cycle.** A third test: two different lines `"A1"` and `"B2"`, the total must be the sum of both. Now single-line storage is no longer enough — green will force you to introduce a `List<InvoiceLine>`. Do it minimally. The test is green.

6. **RED 4 cycle.** Test: the same SKU is added twice, lines must be merged (mirroring `Same_sku_merges_into_one_line` from the lesson). GREEN 4: find the existing line by SKU and add the quantity. No need to refactor yet — just make the test green.

7. **RED 5 cycle — volume threshold discount.** Business rule: if a line quantity is ≥ 10, a 5% discount applies to that line. Test: `AddLine("A1", 10, 10m)` → total 95m (100 − 5%). There is no green yet — you need a condition in the line calculation. GREEN 5: add `if (qty >= 10) subtotal *= 0.95m`. Ugly but green.

8. **RED 6 cycle — second threshold.** Rule: at quantity ≥ 50 the discount is 10%. Test: `AddLine("A1", 50, 10m)` → 450m. GREEN 6: another `if`. Now you have duplicated discount logic — a signal to refactor.

9. **REFACTOR 6.** With tests green, extract an interface `IPriceRule { decimal LineTotal(int qty, decimal unitPrice); }` and two implementations: `StandardRule` (no discount) and `VolumeDiscountRule(int threshold, decimal rate)`. Register the rules as a list in `InvoiceCalculator` and pick the first matching one by threshold. Run `dotnet test` — all green. This is the refactor step from the lesson: no new behavior, only duplication removed, tests green throughout.

10. **RED 7 cycle — invoice-total discount.** Rule: if the subtotal (the sum after line discounts) is ≥ 1000m, an additional 3% discount applies to the whole invoice. Test: 100 lines of 1 unit at 10m → subtotal 1000m → total 970m. GREEN 7: a plain `if (subtotal >= 1000m) subtotal *= 0.97m`. REFACTOR 7: extract an `IInvoiceDiscount` strategy analogously.

11. **Final check.** Run `dotnet test -v n` and confirm all tests are green and their names are visible in the output. Run `dotnet build -warnaserror` — no warnings. Submit the `.csproj` and `.cs` files, plus a short `WALKTHROUGH.md` describing the red-green-refactor cycles in the form "RED N: what was checked → GREEN N: what was added → REFACTOR N: what was removed."

#### Requirements

- The target platform is strictly .NET 8, language C# 12: use top-level statements in `Program.cs` (if you need it for a demo), collection expressions (`[]`), pattern matching (`is`, switch expressions), file-scoped namespaces, `sealed` classes, and `init` properties where appropriate. For monetary values use only `decimal` — never `double` or `float`.
- Solution structure: a class library `Invoice.Domain` (production code) and an xUnit project `Invoice.Tests` (tests). Tests must not touch a database, network, file system, or `DateTime.Now` — isolate all dependencies behind interfaces and replace them with fakes or deterministic values, as the lesson requires.
- Strict cycle observance: one failing test per step, minimal green with no extra behavior, refactor only while tests are green. Writing several red tests at once is forbidden — it is a common mistake called out in the lesson. Adding new behavior during refactor is forbidden.
- All discounts and rules must be parameterized through constructors or interfaces so they can be substituted in tests. Required: a volume-threshold discount (two thresholds: 10 units → 5%, 50 units → 10%) and an invoice-total discount (1000m → 3%). The `InvoiceCalculator` class must have `AddLine` and a `Total` property.
- Tests must cover: empty invoice, single line, several different lines, merging identical SKUs, both volume-threshold cases, the invoice-total discount, the combination of a line and an invoice discount, and boundary values (exactly 10 units, exactly 50 units, exactly 1000m). At least 10 tests, each with a speaking name of the form `Total_is_X_when_Y`. Method names must be in English for runner compatibility.
- The code must build with `-warnaserror` and no warnings. Enable `<TreatWarningsAsErrors>true</TreatWarningsAsErrors>` in the `csproj`. Use `sealed` on classes to explicitly forbid inheritance where it is not planned — a best practice for domain models.

#### Pitfalls

- **Red is an assertion, not a compilation.** The most common beginner mistake is to write a test against a non-existent class and treat the compile error as "red." The lesson explicitly says such a step drags on and breaks the rhythm. First create a stub class so the test compiles, and only then have a real red — a failing assertion.
- **Minimal green may be a constant.** On the first cycle returning `0m` from `Total` is a legitimate green, not "cheating." The lesson permits it. The mistake is to leave the constant forever; generalization arrives on the next red, when a test demands a different value.
- **Do not write several red tests at once.** If two tests fail, it is unclear which code belongs to which. One red → one green → refactor is the rhythm from the lesson. "Multiple failing tests at once" is an explicit common mistake.
- **Refactor only on green.** Never change the structure of the code while any test is red. Restore green first, then improve beauty. Refactoring on red is a frequent mistake that loses behavior.
- **Money math is `decimal` only.** Using `double` leads to rounding errors (0.1 + 0.2 ≠ 0.3). All prices, quantities (integers as `int`, fractions as `decimal`), and discounts must be `decimal`. Compare with `Assert.Equal(expected, actual)` without a precision argument, because `decimal` is exact.
- **Boundary values are separate tests.** "Exactly 10 units" is a threshold boundary where behavior switches. A test for 9 units (no discount) and a test for 10 units (with discount) are two distinct tests, not one parameterized "around the threshold" test. The lesson encourages small steps, and boundaries are the classic place where a "minimal green" catches an off-by-one.
- **Duplication is a signal for a strategy.** When a second discount `if` duplicates the first, the lesson says: "the second or third test often exposes duplication, which the refactor step removes." Do not leave two `if`s — extract an `IPriceRule`. That is exactly "squeezing an interface out of real usage."
- **Do not generalize prematurely.** Do not introduce `IPriceRule` on the first cycle when you only need to return `0m`. A strategy is born during refactor after a red exposes duplication, not from a forecast "someday there will be discounts." The lesson explicitly criticizes designing "from forecasts."
- **Test names are documentation.** `Total_is_970_when_subtotal_is_1000_and_threshold_discount_applies` is long but reads like a specification. Avoid `Test1`, `TestAdd`. The lesson treats tests as "scaffolding" that stays forever — they must be readable.
- **Outside TDD — do not pretend.** If you are running a spike to explore a new library, do not write tests "for the checkbox." The lesson draws a line: a spike has no tests and is thrown away, then the feature is rewritten "properly" with TDD. In this assignment you are inside TDD, but in `WALKTHROUGH.md` note where you might consciously be "outside TDD."

#### Acceptance criteria

- [ ] A solution `InvoiceTdd` exists with projects `Invoice.Domain` (classlib, net8.0) and `Invoice.Tests` (xunit, net8.0); the test project references the domain project.
- [ ] `dotnet build` and `dotnet test` pass with no errors and no warnings (`-warnaserror` / `TreatWarningsAsErrors=true`).
- [ ] Target framework is net8.0, language is C# 12 (`LangVersion=latest`), `Nullable=enable`, file-scoped namespaces, `sealed` classes.
- [ ] Monetary values are `decimal` only; no `double`/`float` for money anywhere.
- [ ] `InvoiceCalculator` is implemented with `AddLine(string sku, int qty, decimal unitPrice)` and a `Total` property.
- [ ] Empty invoice (Total = 0), single line, multiple lines, and merging of identical SKUs are supported.
- [ ] Volume discount: 10 units → 5%, 50 units → 10%; boundaries 9/10/49/50 are covered by tests.
- [ ] Invoice-total discount: subtotal ≥ 1000m → 3%; the boundary of exactly 1000m is covered by a test.
- [ ] An `IPriceRule` interface is extracted with at least two implementations; `InvoiceCalculator` uses a list of rules.
- [ ] An `IInvoiceDiscount` interface (or equivalent) is extracted for the invoice-total discount; the strategy is substitutable.
- [ ] At least 10 xUnit tests with speaking names, each testing a single behavior.
- [ ] Tests do not touch DB, network, files, or `DateTime.Now`; all dependencies are isolated.
- [ ] `WALKTHROUGH.md` describes the red-green-refactor cycles step by step (RED/GREEN/REFACTOR N with explanation).
- [ ] No dead branches, unused parameters, or commented-out code in the solution; `sealed` on domain classes.
- [ ] Rhythm is observed: short steps (≤ 5–10 minutes per cycle), one red test per step, minimal green.

#### Hints

- If on RED 1 you get a compile error instead of an assertion failure — that is not a real red. Create a stub class with an empty property body and rerun the tests.
- For GREEN 1 returning `0m` literally is fine. Do not write a list of lines until the second test forces it.
- When a second discount `if` appears — stop and ask: "what interface is hidden behind this duplication?"
- The threshold boundary is `qty >= threshold`, not `qty > threshold`. Think about which test will catch an off-by-one.
- For the invoice-total discount: first compute the subtotal per line (with line discounts), then apply the discount to the whole subtotal. Do not confuse the order.
- Use collection expressions to initialize the rule list: `new List<IPriceRule> { ... }` or the `[]` syntax of C# 12 where appropriate.
- In tests, use `[Theory]` and `[InlineData]` only for genuinely homogeneous cases (for example, several volume thresholds), but keep boundary values as separate `[Fact]` tests for readability.

#### Reference solution walk-through

```csharp
// C# 12 / .NET 8 — Invoice.Domain/InvoiceCalculator.cs
namespace Invoice.Domain;

// IPriceRule — strategy for computing a discounted line total
public interface IPriceRule
{
    bool AppliesTo(int qty);                          // true when the rule matches the quantity
    decimal LineTotal(int qty, decimal unitPrice);
}

// Standard rule with no discount — the base case / fallback
public sealed class StandardRule : IPriceRule
{
    public bool AppliesTo(int qty) => true;
    public decimal LineTotal(int qty, decimal unitPrice) => qty * unitPrice;
}

// Volume discount: when qty >= threshold a rate is applied
public sealed class VolumeDiscountRule(int threshold, decimal rate) : IPriceRule
{
    public bool AppliesTo(int qty) => qty >= threshold;
    public decimal LineTotal(int qty, decimal unitPrice)
    {
        var baseTotal = qty * unitPrice;
        return baseTotal * (1m - rate);               // discount applied to the whole line
    }
}

// IInvoiceDiscount — strategy for a discount on the invoice subtotal
public interface IInvoiceDiscount
{
    bool AppliesTo(decimal subtotal);
    decimal Apply(decimal subtotal);
}

public sealed class TotalThresholdDiscount(decimal threshold, decimal rate) : IInvoiceDiscount
{
    public bool AppliesTo(decimal subtotal) => subtotal >= threshold;
    public decimal Apply(decimal subtotal) => subtotal * (1m - rate);
}

public sealed class InvoiceLine
{
    public string Sku { get; }
    public int Quantity { get; private set; }
    public decimal UnitPrice { get; }

    public InvoiceLine(string sku, int qty, decimal unitPrice)
    {
        Sku = sku;
        Quantity = qty;
        UnitPrice = unitPrice;
    }

    public void AddQuantity(int qty) => Quantity += qty;
}

// InvoiceCalculator — aggregates lines and applies rules
public sealed class InvoiceCalculator
{
    private readonly List<InvoiceLine> _lines = [];
    private readonly IPriceRule[] _lineRules;
    private readonly IInvoiceDiscount[] _invoiceDiscounts;

    // Rules are injected through the constructor — testability and substitution
    public InvoiceCalculator(
        IPriceRule[]? lineRules = null,
        IInvoiceDiscount[]? invoiceDiscounts = null)
    {
        _lineRules = lineRules ?? [new VolumeDiscountRule(50, 0.10m),
                                   new VolumeDiscountRule(10, 0.05m),
                                   new StandardRule()];
        _invoiceDiscounts = invoiceDiscounts ?? [new TotalThresholdDiscount(1000m, 0.03m)];
    }

    public void AddLine(string sku, int qty, decimal unitPrice)
    {
        var existing = _lines.FirstOrDefault(l => l.Sku == sku);
        if (existing is not null)
        {
            existing.AddQuantity(qty);
            return;
        }
        _lines.Add(new InvoiceLine(sku, qty, unitPrice));
    }

    public decimal Total
    {
        get
        {
            // First compute each line applying the first matching rule
            decimal subtotal = 0m;
            foreach (var line in _lines)
            {
                var rule = _lineRules.First(r => r.AppliesTo(line.Quantity));
                subtotal += rule.LineTotal(line.Quantity, line.UnitPrice);
            }

            // Then apply invoice-level discounts to the subtotal
            foreach (var d in _invoiceDiscounts)
            {
                if (d.AppliesTo(subtotal))
                {
                    subtotal = d.Apply(subtotal);
                }
            }
            return subtotal;
        }
    }
}
```

```csharp
// C# 12 / .NET 8 — Invoice.Tests/InvoiceCalculatorTests.cs
namespace Invoice.Tests;

using Invoice.Domain;
using Xunit;

public class InvoiceCalculatorTests
{
    private static InvoiceCalculator NewCalc() => new();

    [Fact]
    public void Total_is_zero_for_empty_invoice() // RED 1
        => Assert.Equal(0m, NewCalc().Total);

    [Fact]
    public void Total_is_price_times_quantity_for_single_line() // RED 2
    {
        var calc = NewCalc();
        calc.AddLine("A1", 2, 10m);
        Assert.Equal(20m, calc.Total);
    }

    [Fact]
    public void Total_sums_two_different_lines() // RED 3
    {
        var calc = NewCalc();
        calc.AddLine("A1", 1, 10m);
        calc.AddLine("B2", 1, 5m);
        Assert.Equal(15m, calc.Total);
    }

    [Fact]
    public void Same_sku_merges_into_one_line() // RED 4
    {
        var calc = NewCalc();
        calc.AddLine("A1", 1, 10m);
        calc.AddLine("A1", 1, 10m);
        Assert.Equal(20m, calc.Total);
    }

    [Fact]
    public void Volume_discount_5pct_applies_at_qty_10() // RED 5
    {
        var calc = NewCalc();
        calc.AddLine("A1", 10, 10m);
        Assert.Equal(95m, calc.Total); // 100 - 5%
    }

    [Fact]
    public void No_volume_discount_below_threshold_qty_9() // boundary
    {
        var calc = NewCalc();
        calc.AddLine("A1", 9, 10m);
        Assert.Equal(90m, calc.Total);
    }

    [Fact]
    public void Volume_discount_10pct_applies_at_qty_50() // RED 6
    {
        var calc = NewCalc();
        calc.AddLine("A1", 50, 10m);
        Assert.Equal(450m, calc.Total); // 500 - 10%
    }

    [Fact]
    public void Invoice_discount_3pct_applies_at_subtotal_1000() // RED 7
    {
        var calc = NewCalc();
        for (int i = 0; i < 100; i++)
            calc.AddLine($"SKU-{i}", 1, 10m);
        Assert.Equal(970m, calc.Total); // 1000 - 3%
    }

    [Fact]
    public void No_invoice_discount_below_1000_subtotal() // total boundary
    {
        var calc = NewCalc();
        for (int i = 0; i < 99; i++)
            calc.AddLine($"SKU-{i}", 1, 10m);
        Assert.Equal(990m, calc.Total);
    }

    [Fact]
    public void Volume_and_invoice_discounts_combine() // integration
    {
        var calc = NewCalc();
        calc.AddLine("A1", 10, 100m); // 1000 - 5% = 950
        Assert.Equal(950m, calc.Total); // 950 < 1000 → no invoice discount
    }

    [Fact]
    public void Custom_rules_can_be_injected() // strategy substitution
    {
        var calc = new InvoiceCalculator(
            lineRules: [new StandardRule()],
            invoiceDiscounts: [new TotalThresholdDiscount(0m, 0.50m)]); // 50% off everything
        calc.AddLine("A1", 1, 10m);
        Assert.Equal(5m, calc.Total);
    }
}
```

**Walk-through.** The first group of cycles (RED 1 → GREEN 1 → RED 2 → GREEN 2) reproduces the classic TDD rhythm from the lesson: an empty invoice yields zero, then a single line yields price times quantity. Returning `0m` on GREEN 1 is an intentionally "ugly" green that the lesson explicitly permits; generalization arrives on RED 2 when a test demands a different value. RED 3 forces the move from a single line to a `List<InvoiceLine>` — a step where design is "squeezed out" of real usage rather than forecast: we did not introduce the list in advance, it appeared when the test required it.

RED 4 (merging SKUs) mirrors the `Same_sku_merges_into_one_line` example from the lesson almost one-to-one: finding an existing line with `FirstOrDefault` and calling `AddQuantity`. RED 5 and RED 6 introduce two discount thresholds — and here the duplication the lesson warns about appears: two `if (qty >= ...)` branches in the line calculation. Instead of leaving them, on REFACTOR 6 we extract `IPriceRule` with `AppliesTo` and `LineTotal`, exactly like `IPricing` in the `Basket` example. The order of rules in the array matters: the more specific rule first (50 units → 10%), then the less specific (10 units → 5%), then `StandardRule` as a fallback with `AppliesTo => true` — this is a "first match wins" pattern that reads well inside `Total`.

RED 7 introduces a discount on the invoice total — a different level of abstraction, so it gets its own interface `IInvoiceDiscount` rather than reusing `IPriceRule`: different rules operate on different aggregates (a line versus the total). The lesson teaches: do not generalize prematurely, but once duplication exists, remove it. Here there is no duplication between the levels, so two interfaces are justified. The `InvoiceCalculator` constructor accepts arrays of rules with defaults via `??` and collection expressions `[]` — that is C# 12, and at the same time it is the substitution point for tests: the last test `Custom_rules_can_be_injected` proves the strategies are substitutable, which means the design is testable as the lesson requires. `sealed` on all classes is a best practice for domain models: forbidding inheritance unless it is explicitly needed. `decimal` everywhere — money cannot be computed in `double`. Boundary tests (9 units, 99 lines) catch off-by-one errors that a "minimal green" often lets through.

#### Going deeper

1. **Parameterize thresholds via configuration.** Instead of hard-coding `10/50/1000`, load the rules from `appsettings.json` or from a dictionary passed to the constructor. Write a test that verifies the configuration is correctly materialized into an `IPriceRule[]`.
2. **Add a cumulative-SKU progressive discount.** If the total quantity of a single SKU across all additions (even across invoice sessions, if you introduce them) reaches 100, an additional 2% discount applies. Write a red test, then green, then refactor so you do not duplicate the `VolumeDiscountRule` logic.
3. **Introduce an `IClock` for time-based discounts.** Add a rule: a 1% discount applies only on weekends. Introduce an `IClock { DateTime UtcNow { get; } }` interface; in tests pass a fake with a fixed date. The lesson requires isolating time behind an interface — this is the application.
4. **Mutation testing.** Run `dotnet stryker` (if available) on the test project and verify that mutations such as `>=` → `>`, `1m - rate` → `1m + rate` are killed by the tests. If a mutant survives — add a red test that catches it.

---

#### Чек-лист сдачи / Submission checklist

- [ ] (RU) Solution `InvoiceTdd` с проектами `Invoice.Domain` и `Invoice.Tests` создан.
- [ ] (RU) `dotnet build` и `dotnet test` проходят без ошибок и предупреждений.
- [ ] (RU) Целевая платформа net8.0, C# 12, `Nullable=enable`, `sealed` классы, file-scoped namespaces.
- [ ] (RU) Все денежные значения в `decimal`; нет `double`/`float`.
- [ ] (RU) `InvoiceCalculator.AddLine` и `Total` реализованы; поддержка пустого счёта, одной строки, нескольких строк, объединения SKU.
- [ ] (RU) Скидки за объём (10→5%, 50→10%) и за итог (1000→3%) реализованы и покрыты тестами с границами.
- [ ] (RU) Интерфейсы `IPriceRule` и `IInvoiceDiscount` выделены; стратегии подменяемы в тестах.
- [ ] (RU) Минимум 10 xUnit-тестов с говорящими именами; нет обращений к БД/сети/файлам/`DateTime.Now`.
- [ ] (RU) `WALKTHROUGH.md` с описанием циклов red-green-refactor приложен.
- [ ] (RU) Циклы короткие (≤ 5–10 мин), один красный тест за шаг, green минимальный, refactor только на зелёном.
- [ ] (EN) Solution `InvoiceTdd` with `Invoice.Domain` and `Invoice.Tests` projects created.
- [ ] (EN) `dotnet build` and `dotnet test` pass with no errors and no warnings.
- [ ] (EN) Target net8.0, C# 12, `Nullable=enable`, `sealed` classes, file-scoped namespaces.
- [ ] (EN) All monetary values are `decimal`; no `double`/`float`.
- [ ] (EN) `InvoiceCalculator.AddLine` and `Total` implemented; empty, single-line, multi-line and SKU-merge cases supported.
- [ ] (EN) Volume discounts (10→5%, 50→10%) and invoice-total discount (1000→3%) implemented and tested with boundaries.
- [ ] (EN) `IPriceRule` and `IInvoiceDiscount` interfaces extracted; strategies substitutable in tests.
- [ ] (EN) At least 10 xUnit tests with speaking names; no DB/network/file/`DateTime.Now` access.
- [ ] (EN) `WALKTHROUGH.md` describing the red-green-refactor cycles attached.
- [ ] (EN) Short cycles (≤ 5–10 min), one red test per step, minimal green, refactor only on green.

#### Ресурсы / Resources

- [Microsoft Learn — Unit testing in .NET — https://learn.microsoft.com/dotnet/core/testing/](https://learn.microsoft.com/dotnet/core/testing/)
- [Microsoft Learn — xUnit overview — https://learn.microsoft.com/dotnet/core/testing/unit-testing-with-dotnet-test](https://learn.microsoft.com/dotnet/core/testing/unit-testing-with-dotnet-test)
- [Martin Fowler — Test-Driven Development — https://martinfowler.com/bliki/TestDrivenDevelopment.html](https://martinfowler.com/bliki/TestDrivenDevelopment.html)
- [Robert C. Martin — The Three Laws of TDD — https://butunclebob.com/ArticleS.UncleBob.TheThreeRulesOfTdd](https://butunclebob.com/ArticleS.UncleBob.TheThreeRulesOfTdd)
- [Kent Beck — TDD by Example (book reference)](https://www.oreilly.com/library/view/test-driven-development/0321146530/)
- [Microsoft Learn — Testing ASP.NET Core MVC apps — https://learn.microsoft.com/dotnet/architecture/modern-web-apps-azure/test-asp-net-core-mvc-apps](https://learn.microsoft.com/dotnet/architecture/modern-web-apps-azure/test-asp-net-core-mvc-apps)

---

[← К уроку M15-L07](lesson-M15-L07-tdd.md) | [⬆ К модулю M15](../README.md) | [Следующее ДЗ →](homework-M15-L08-coverage-coverlet.md)
