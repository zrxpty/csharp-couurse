---
[← К уроку M05-L06](lesson-M05-L06-sealed-internal.md) | [⬆ К модулю M05](../README.md) | [Следующее ДЗ →](homework-M05-L07-is-as-patterns.md)
---

### Домашнее задание M05-L06: sealed, internal / Homework M05-L06: sealed, internal

**Урок / Lesson:** M05-L06
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 4/5
**Цель / Goal:** (RU) Научиться осознанно применять модификаторы `sealed` (для классов и методов) и `internal` (для типов и членов), грамотно проектировать периметр сборки, комбинировать `internal sealed`, открывать внутренние типы тест-проекту через `InternalsVisibleTo` и фиксировать критичное поведение через `sealed override`. (EN) Learn to deliberately apply `sealed` (on classes and methods) and `internal` (on types and members), design the assembly perimeter correctly, combine `internal sealed`, expose internal types to a test project via `InternalsVisibleTo`, and freeze critical behavior with `sealed override`.

#### Связь с уроком / Connection to the lesson
(RU — 2–3 предложения)
Урок вводит два модификатора, формирующих «периметр безопасности» кода: `sealed` запрещает наследование (а `sealed override` — дальнейшие переопределения метода), а `internal` ограничивает видимость текущей сборкой. Домашнее задание заставит вас пройти весь цикл: запечатать класс-значение, зафиксировать поведение виртуального метода на одном уровне, спрятать сервисную инфраструктуру за `internal sealed`, открыть её тест-проекту через `InternalsVisibleTo` и проверить, что компилятор реально запрещает нарушать эти границы.
(EN — same)
The lesson introduces two modifiers that form the code's "security perimeter": `sealed` forbids inheritance (and `sealed override` forbids further overrides of a method), while `internal` restricts visibility to the current assembly. This homework walks you through the full cycle: seal a value class, freeze a virtual method's behavior at one level, hide service infrastructure behind `internal sealed`, expose it to a test project via `InternalsVisibleTo`, and verify that the compiler actually enforces these boundaries.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация

Представьте, что вы присоединились к команде, разрабатывающей небольшую библиотеку учета платежей `Payables.Core`. Библиотека публикует узкий публичный API — несколько неизменяемых типов-значений (например, `Money`, `PaymentId`) и статический фасад `PayablesService`. Всё остальное (калькуляторы комиссий, провайдеры курсов валют, кэши, фабрики транзакций) должно оставаться деталями реализации: внешние потребители не должны о них знать, чтобы вы могли свободно рефакторить их между минорными версиями без нарушения контракта.

Недавно на ревью обнаружились три проблемы. Во-первых, кто-то отнаследовался от `Money` и добавил изменяемое поле `Tip`, тем самым сломав инвариант неизменяемости — теперь `Money` перестал быть честным значением и начал «протекать» в тестах на равенство. Во-вторых, во внутренней иерархии фигур отчётов (`ReportShape → PieChart → DonutChart`) кто-то переопределил метод рендера на уровне `DonutChart`, хотя команда договорилась, что стратегия отрисовки «кольца» окончательна и её нельзя менять дальше. В-третьих, вспомогательный класс `FeeCalculator` оказался `public`, хотя он нужен только внутри сборки, и теперь любое его переименование формально становится breaking change.

Ваша задача — навести порядок, опираясь на модификаторы `sealed` и `internal`, изученные в уроке M05-L06. Вы спроектируете периметр сборки так, чтобы публичная поверхность была минимальной и осознанной, инварианты типов-значений защищены от наследников, критичное поведение зафиксировано на нужном уровне иерархии, а тесты при этом могли бы проверять внутренности без раздувания API. Заодно вы убедитесь, что компилятор реально работает: попытки нарушить границы должны заканчиваться ошибками CS0501/CS0239/CS0122.

#### Что нужно сделать (пошагово)

1. **Создайте решение и проекты.** Выполните в пустой папке:
   ```bash
   dotnet new sln -n Payables
   dotnet new classlib -n Payables.Core -o src/Payables.Core -f net8.0
   dotnet new xunit -n Payables.Core.Tests -o tests/Payables.Core.Tests -f net8.0
   dotnet sln add src/Payables.Core/Payables.Core.csproj
   dotnet sln add tests/Payables.Core.Tests/Payables.Core.Tests.csproj
   dotnet add tests/Payables.Core.Tests/Payables.Core.Tests.csproj reference src/Payables.Core/Payables.Core.csproj
   ```
   В `src/Payables.Core/Payables.Core.csproj` включите `<TreatWarningsAsErrors>true</TreatWarningsAsErrors>` и `<Nullable>enable</Nullable>`, чтобы невыставленные границы сразу превращались в ошибки.

2. **Запечатайте тип-значение.** В файле `Money.cs` объявите `public sealed class Money` с неизменяемыми свойствами `Amount` (decimal) и `Currency` (строка из трёх букв). Реализуйте конструктор с валидацией (`amount < 0` → `ArgumentOutOfRangeException`, пустая валюта → `ArgumentException`), метод `Add(Money other)` с проверкой совпадения валют и переопределённый `ToString()`. Добавьте XML-комментарий, объясняющий, почему класс запечатан (защита инварианта неизменяемости + производительность через devirtualization).

3. **Покажите, что наследование запрещено.** В отдельном файле `Forbidden.cs` (или в комментарии) попробуйте написать `class FakeMoney : Money { }`. Вы должны увидеть ошибку компилятора **CS0509** — «…cannot derive from sealed type». Зафиксируйте номер ошибки в файле `NOTES.md` с пояснением.

4. **Иерархия с `sealed override`.** В `ReportShapes.cs` создайте иерархию:
   - `public class ReportShape` с `public virtual void Render() => Console.WriteLine("shape");`
   - `public class PieChart : ReportShape` с `public override void Render() => Console.WriteLine("pie");`
   - `public class DonutChart : PieChart` с `public sealed override void Render() => Console.WriteLine("donut");`
   Попытка добавить `class FancyDonut : DonutChart { public override void Render() {} }` должна дать ошибку **CS0239**. Запишите это в `NOTES.md`.

5. **Internal-инфраструктура.** Создайте `FeeCalculator.cs`: `internal sealed class FeeCalculator` с методом `decimal Apply(decimal amount, decimal rate)`. Создайте `ExchangeRateProvider.cs`: `internal sealed class ExchangeRateProvider`, конструктор принимает `decimal rate`, метод `decimal Convert(Money from, string target)` проверяет валюту и умножает. Эти классы не должны компилироваться, если их сделать `public` по ошибке — но в нашем задании они остаются `internal`.

6. **Public-фасад.** В `PayablesService.cs` сделайте `public static class PayablesService`, который внутри использует `ExchangeRateProvider` и `FeeCalculator`. Метод `public static Money ConvertUsdToEur(Money usd, decimal feeRate)` должен: проверить, что валюта — `USD`; применить комиссию через `FeeCalculator`; конвертировать через `ExchangeRateProvider` с курсом `0.92m`; округлить до двух знаков. Фасад — единственный публичный вход.

7. **`InternalsVisibleTo`.** В `Payables.Core.csproj` (или в `AssemblyInfo.cs`) добавьте:
   ```xml
   <ItemGroup>
     <InternalsVisibleTo Include="Payables.Core.Tests" />
   </ItemGroup>
   ```
   Это позволит тест-проекту видеть `FeeCalculator` и `ExchangeRateProvider`.

8. **Тесты.** В `tests/Payables.Core.Tests` напишите тесты xUnit: `Money_Add_SameCurrency_Works`, `Money_Add_DifferentCurrency_Throws`, `Money_Negative_Throws`, `Money_IsSealed` (через рефлексию: `typeof(Money).IsSealed` должно быть `true`), `FeeCalculator_Internal_IsAccessible` (создаёт `new FeeCalculator()` напрямую — это работает только благодаря `InternalsVisibleTo`), `PayablesService_ConvertUsdToEur_RoundsToTwoDecimals`. Убедитесь, что после удаления `InternalsVisibleTo` тест `FeeCalculator_Internal_IsAccessible` перестаёт компилироваться с ошибкой **CS0122**.

9. **Запуск.** Выполните `dotnet build` и `dotnet test`. Ожидаемый вывод: `Passed! - Failed: 0, Passed: N`. Все тесты зелёные.

10. **Замечание о производительности.** Напишите в `NOTES.md` короткое эссе (5–8 предложений): почему запечатанные классы помогают JIT в devirtualization, в каких случаях выигрыш реален (горячие циклы, value-типы-обёртки), и почему «запечатывать ради оптимизации без измерений» — антипаттерн из урока.

#### Требования к решению

- Решение собирается под .NET 8 / C# 12 без предупреждений (`TreatWarningsAsErrors=true`). Используйте top-level statements только в тестовом `Program.cs` (или стандартный xUnit-шаблон без `Main`).
- Все публичные типы библиотеки либо `public sealed`, либо `public abstract` с осознанным расширением; никаких «случайно публичных» классов. Внутренние сервисы помечены `internal sealed`.
- Класс `Money` неизменяем: свойства только на чтение, конструктор валидирует входные данные, никаких публичных сеттеров. Класс запечатан, в XML-документации указана причина.
- В иерархии `ReportShape → PieChart → DonutChart` метод `Render` на `DonutChart` помечен `sealed override`. Классы иерархии не запечатаны целиком (демонстрируется разница: фиксируется метод, а не весь класс).
- Файл `NOTES.md` содержит зафиксированные номера ошибок компилятора (CS0509, CS0239, CS0122) с пояснением по одной-две строки каждый.
- Тесты подтверждают, что `typeof(Money).IsSealed == true`, что `FeeCalculator` доступен из теста (значит `InternalsVisibleTo` работает), и что конвертация корректно округляет результат.
- Код снабжён двуязычными комментариями (RU + EN), как в примере урока.
- Все `internal` члены задокументированы: краткий комментарий объясняет, почему они внутренние.

#### Тонкости и подводные камни

- **`sealed` допустим только поверх `override`.** Если вы напишете `public sealed void Render()` в `ReportShape`, где метод не объявлен `virtual`/`override`, компилятор выдаст ошибку CS0549. Сначала должна быть цепочка `virtual → override → sealed override`. Это самая частая ошибка новичков — запечатать метод, который ещё не был виртуальным.
- **Граница `internal` — это сборка, а не пространство имён.** Не ждите, что тип `internal` в `Payables.Core` будет виден другому проекту, даже если он в том же `namespace Payables`. Два проекта = две сборки = две разные границы. Многие путают `internal` с «внутри namespace».
- **`InternalsVisibleTo` требует точного имени сборки.** Если тест-проект называется `Payables.Core.Tests`, то и в `Include` должно быть именно это имя. Подписанные сборки (strong naming) потребуют ещё и публичный ключ — для учебного проекта это неактуально, но помните про это в боевых библиотеках.
- **Запечатывание ради «оптимизации» без измерений — антипаттерн.** Урок явно предупреждает: сначала измерьте бенчмарком (BenchmarkDotNet), потом запечатывайте, только если выигрыш реален. Иначе вы лишаете себя расширяемости без выгоды.
- **`private` ≠ `internal`.** `private` ограничивает одним типом, `internal` — одной сборкой. Не путайте: `internal`-поле доступно всем классам сборки, что шире, чем кажется.
- **Не делайте члены `public` ради тестов.** Правильный путь — `InternalsVisibleTo`. Раздувание API ради тестируемости — частая ошибка из урока.
- **`sealed class` не запрещает доступ к `protected`-членам базового класса через `base`**, но запрещает создание новых наследников. Не путайте «запрет наследования» с «запрет доступа».
- **Иммутабельные value-типы стоит запечатывать по умолчанию**, иначе наследник может добавить изменяемое состояние и сломать семантику равенства (как в кейсе `Money` с `Tip`).
- **Девиртуализация работает не всегда.** JIT .NET 8 умеет devirtualize, но не для всех вызовов. Запечатывание даёт компилятору информацию, но не гарантирует инлайн — это лишь возможность, а не обещание.

#### Критерии приёмки

- [ ] Решение `dotnet build` собирается без ошибок и предупреждений (warnings as errors).
- [ ] `dotnet test` проходит все тесты (`Passed: N, Failed: 0`).
- [ ] `Money` объявлен как `public sealed class`, неизменяем, валидирует аргументы в конструкторе.
- [ ] Попытка унаследоваться от `Money` даёт ошибку **CS0509**, зафиксированную в `NOTES.md`.
- [ ] Иерархия `ReportShape → PieChart → DonutChart` содержит `public sealed override void Render()` на `DonutChart`.
- [ ] Попытка переопределить `Render` в наследнике `DonutChart` даёт ошибку **CS0239`, зафиксированную в `NOTES.md`.
- [ ] `FeeCalculator` и `ExchangeRateProvider` объявлены `internal sealed`.
- [ ] `PayablesService` — единственный публичный фасад, использующий внутренние сервисы.
- [ ] В `.csproj` настроен `InternalsVisibleTo` для `Payables.Core.Tests`.
- [ ] Тест `Money_IsSealed` проверяет `typeof(Money).IsSealed == true` и проходит.
- [ ] Тест `FeeCalculator_Internal_IsAccessible` создаёт `new FeeCalculator()` напрямую и проходит.
- [ ] Тест `PayablesService_ConvertUsdToEur_RoundsToTwoDecimals` проверяет округление (например, `123.456 USD + fee → 113.59 EUR` при курсе 0.92).
- [ ] `NOTES.md` содержит эссе о devirtualization (5–8 предложений).
- [ ] Код снабжён двуязычными комментариями RU+EN.
- [ ] Все `internal` члены имеют поясняющий комментарий «почему internal».
- [ ] В `Program.cs` тестов (если есть) используются top-level statements либо стандартный xUnit-шаблон без `Main`.

#### Подсказки (без прямого ответа)

- Подумайте, какой модификатор ставит «пломбу» на весь класс, а какой — только на одно звено цепочки переопределений. Это два разных инструмента.
- Чтобы проверить, что класс действительно запечатан, в тестах используйте `typeof(T).IsSealed` — это булево свойство рефлексии, не требующее создания экземпляра.
- Вспомните, что `internal` по умолчанию присваивается классам без модификатора. Но хорошим тоном считается писать его явно — для читаемости и самодокументируемости.
- Если тест не видит `internal`-класс, проверьте две вещи: (1) добавлен ли `InternalsVisibleTo`, (2) совпадает ли имя тест-сборки в `Include` с реальным именем.
- Для округления денежных сумм используйте `Math.Round(value, 2, MidpointRounding.AwayFromZero)` или `ToEven` — но задайтесь вопросом, какой режим «бухгалтерский», а какой «банковский».
- Ошибка CS0122 означает «недоступен из-за уровня защиты» — это маркер того, что вы попытались обратиться к `internal` типу снаружи его сборки (или без `InternalsVisibleTo`).

#### Эталонное решение (разбор)

```csharp
// C# 12+ / .NET 8. Модификаторы sealed и internal в действии.
// sealed & internal modifiers in action.

using System;

namespace Payables.Core;

/// <summary>
/// Неизменяемая сумма в заданной валюте. Запечатан, чтобы защитить
/// инвариант неизменяемости и помочь JIT в devirtualization.
/// Immutable amount in a currency. Sealed to protect the immutability
/// invariant and to help the JIT with devirtualization.
/// </summary>
public sealed class Money
{
    public decimal Amount { get; }       // только для чтения / read-only.
    public string Currency { get; }

    public Money(decimal amount, string currency)
    {
        if (amount < 0)
            throw new ArgumentOutOfRangeException(nameof(amount),
                "Amount must be non-negative / сумма должна быть неотрицательной.");
        if (string.IsNullOrWhiteSpace(currency) || currency.Length != 3)
            throw new ArgumentException("Currency must be a 3-letter code / валюта — 3 буквы.",
                nameof(currency));

        Amount = amount;
        Currency = currency.ToUpperInvariant();
    }

    public Money Add(Money other)
    {
        if (Currency != other.Currency)
            throw new InvalidOperationException(
                "Currency mismatch / несовпадение валют.");

        return new Money(Amount + other.Amount, Currency);
    }

    public override string ToString() => $"{Amount:0.00} {Currency}";
}

// Иерархия отчётных фигур: метод Render запечатан на уровне DonutChart.
// Report shape hierarchy: Render is sealed at the DonutChart level.
public class ReportShape
{
    public virtual void Render() => Console.WriteLine("shape / фигура");
}

public class PieChart : ReportShape
{
    public override void Render() => Console.WriteLine("pie / круговая диаграмма");
}

public class DonutChart : PieChart
{
    // sealed override: последний раз переопределён — дальше нельзя.
    // sealed override: overridden for the last time — no further overrides.
    public sealed override void Render() => Console.WriteLine("donut / кольцевая диаграмма");
}

// internal sealed — внутренняя инфраструктура: не видна снаружу, не наследуется.
// internal sealed — internal infrastructure: invisible outside, cannot be inherited.
internal sealed class FeeCalculator
{
    public decimal Apply(decimal amount, decimal feeRate)
    {
        if (feeRate < 0 || feeRate > 1)
            throw new ArgumentOutOfRangeException(nameof(feeRate));
        return amount * (1m - feeRate);
    }
}

internal sealed class ExchangeRateProvider
{
    private readonly decimal _rate;
    public ExchangeRateProvider(decimal rate) => _rate = rate;

    public decimal Convert(Money money, string targetCurrency)
    {
        if (money.Currency == targetCurrency) return money.Amount;
        return money.Amount * _rate;
    }
}

// Публичный фасад — единственный вход для внешних потребителей.
// Public façade — the only entry point for external consumers.
public static class PayablesService
{
    // Внутренние сервисы-синглтоны (internal, но фасад в той же сборке).
    // Internal service singletons (internal, but the façade is in the same assembly).
    private static readonly ExchangeRateProvider UsdToEur = new(0.92m);
    private static readonly FeeCalculator Fees = new();

    public static Money ConvertUsdToEur(Money usd, decimal feeRate)
    {
        if (usd.Currency != "USD")
            throw new ArgumentException("Only USD supported / поддерживается только USD.", nameof(usd));

        decimal afterFee = Fees.Apply(usd.Amount, feeRate);
        decimal eur = UsdToEur.Convert(new Money(afterFee, "USD"), "EUR");
        decimal rounded = Math.Round(eur, 2, MidpointRounding.AwayFromZero);

        return new Money(rounded, "EUR");
    }
}
```

Разбор по строкам. Класс `Money` помечен `public sealed` — это решение сразу убивает два зайца: защищает инвариант неизменяемости (никто не добавит изменяемое поле `Tip`) и даёт JIT сигнал для devirtualization. Свойства `Amount` и `Currency` имеют только геттеры — состояние действительно неизменяемо. Конструктор валидирует вход: отрицательная сумма и некорректная валюта отсекаются на входе, что соответствует best practice из урока («защищай контракт»). Метод `Add` проверяет совпадение валют — классическая защита от смешивания EUR и USD. XML-комментарий на классе объясняет, почему он запечатан: это требование урока «документируй намерение».

Иерархия `ReportShape → PieChart → DonutChart` демонстрирует тонкую разницу между `sealed class` и `sealed override`. Классы иерархии НЕ запечатаны целиком — от `PieChart` теоретически можно наследоваться. Но метод `Render` на `DonutChart` помечен `public sealed override`, что фиксирует поведение кольцевой диаграммы окончательно: команда решила, что стратегия рендера «donut» не должна меняться в подклассах, но при этом расширять иерархию новыми типами диаграмм можно. Это иллюстрирует best practice из урока: «применяй `sealed override`, чтобы зафиксировать поведение метода на конкретном уровне иерархии, не запечатывая весь класс». Любая попытка написать `class FancyDonut : DonutChart { public override void Render() {} }` упадёт с CS0239.

Классы `FeeCalculator` и `ExchangeRateProvider` объявлены `internal sealed` — самая строгая и одновременно самая гибкая для рефакторинга комбинация, рекомендованная в уроке. Они невидимы снаружи `Payables.Core` (никаких breaking changes при переименовании), не наследуются (никаких сюрпризов с инвариантами), и при этом полностью доступны внутри сборки. Фасад `PayablesService` — единственный `public` вход: он инкапсулирует внутренние сервисы и предоставляет осознанный публичный API. Это реализует принцип «минимизируй публичную поверхность».

Наконец, `InternalsVisibleTo` (настроенный в `.csproj`) открывает тест-проекту доступ к `FeeCalculator` и `ExchangeRateProvider` — это правильный путь тестирования внутренностей без раздувания API, как требует урок. Тест `FeeCalculator_Internal_IsAccessible` напрямую создаёт `new FeeCalculator()`, что было бы невозможно без `InternalsVisibleTo` (ошибка CS0122). Концепции урока применены все: запечатывание классов, `sealed override`, `internal`, `internal sealed`, `InternalsVisibleTo`, документирование намерения, минимизация публичной поверхности.

#### Задания на углубление (бонус)

1. **Бенчмарк devirtualization.** С помощью BenchmarkDotNet сравните производительность горячего цикла суммирования `Money` в sealed-версии и в незапечатанной (`public class MoneyUnsealed`). Напишите два бенчмарка, запустите, запишите разницу в наносекундах. Объясните, в каких случаях выигрыш реален, а в каких — не заметен.
2. **`InternalsVisibleTo` с подписью.** Разберитесь, как работает `InternalsVisibleTo` для strongly-named сборок: сгенерируйте `.snk`, подпишите обе сборки, настройте `Include` с публичным ключом. Зафиксируйте шаги в `NOTES.md`.
3. **`sealed` + records.** Перепишите `Money` как `public sealed record Money`. Сравните семантику: что даёт record-синтаксис (value equality, `with`), и нужно ли здесь вообще `sealed`. Объясните, наследуются ли records по умолчанию.
4. **Запрет наследования через `init`-только свойства.** Исследуйте, можно ли защитить инвариант неизменяемости без `sealed`, используя только `init`-сеттеры и private-конструкторы. Обсудите, чем такой подход слабее `sealed`.

---

## Statement in English / Постановка на английском

#### Context & motivation

Imagine you have joined a team building a small payments accounting library called `Payables.Core`. The library exposes a narrow public API — a few immutable value types (such as `Money`, `PaymentId`) and a static façade `PayablesService`. Everything else (fee calculators, exchange-rate providers, caches, transaction factories) must remain an implementation detail: external consumers must not know about them, so that you can refactor them freely between minor versions without breaking the contract.

A recent review surfaced three problems. First, someone derived from `Money` and added a mutable `Tip` field, thereby breaking the immutability invariant — `Money` stopped being an honest value and started leaking in equality tests. Second, in an internal hierarchy of report shapes (`ReportShape → PieChart → DonutChart`) someone overrode the render method at the `DonutChart` level, even though the team had agreed that the "donut" rendering strategy was final and could not be changed further down. Third, the helper class `FeeCalculator` turned out to be `public`, although it is only needed inside the assembly, so now any rename is formally a breaking change.

Your task is to restore order using the `sealed` and `internal` modifiers studied in lesson M05-L06. You will design the assembly perimeter so that the public surface is minimal and deliberate, value-type invariants are protected from inheritors, critical behavior is locked at the right level of the hierarchy, and tests can still inspect the internals without bloating the API. Along the way, you will confirm that the compiler actually enforces the rules: attempts to cross the boundaries should end with errors CS0501/CS0239/CS0122.

#### What to do step by step

1. **Create the solution and projects.** In an empty folder run:
   ```bash
   dotnet new sln -n Payables
   dotnet new classlib -n Payables.Core -o src/Payables.Core -f net8.0
   dotnet new xunit -n Payables.Core.Tests -o tests/Payables.Core.Tests -f net8.0
   dotnet sln add src/Payables.Core/Payables.Core.csproj
   dotnet sln add tests/Payables.Core.Tests/Payables.Core.Tests.csproj
   dotnet add tests/Payables.Core.Tests/Payables.Core.Tests.csproj reference src/Payables.Core/Payables.Core.csproj
   ```
   In `src/Payables.Core/Payables.Core.csproj` enable `<TreatWarningsAsErrors>true</TreatWarningsAsErrors>` and `<Nullable>enable</Nullable>` so that any missed boundary turns immediately into an error.

2. **Seal the value type.** In `Money.cs` declare `public sealed class Money` with immutable properties `Amount` (decimal) and `Currency` (a three-letter string). Implement a constructor with validation (`amount < 0` → `ArgumentOutOfRangeException`, empty currency → `ArgumentException`), a method `Add(Money other)` that checks currency equality, and an overridden `ToString()`. Add an XML comment explaining why the class is sealed (protecting the immutability invariant + devirtualization performance).

3. **Show that inheritance is forbidden.** In a separate file `Forbidden.cs` (or in a comment) try to write `class FakeMoney : Money { }`. You should see compiler error **CS0509** — "...cannot derive from sealed type". Record the error code in `NOTES.md` with a short explanation.

4. **Hierarchy with `sealed override`.** In `ReportShapes.cs` create a hierarchy:
   - `public class ReportShape` with `public virtual void Render() => Console.WriteLine("shape");`
   - `public class PieChart : ReportShape` with `public override void Render() => Console.WriteLine("pie");`
   - `public class DonutChart : PieChart` with `public sealed override void Render() => Console.WriteLine("donut");`
   An attempt to add `class FancyDonut : DonutChart { public override void Render() {} }` must fail with **CS0239**. Record this in `NOTES.md`.

5. **Internal infrastructure.** Create `FeeCalculator.cs`: `internal sealed class FeeCalculator` with a method `decimal Apply(decimal amount, decimal rate)`. Create `ExchangeRateProvider.cs`: `internal sealed class ExchangeRateProvider`, constructor takes `decimal rate`, method `decimal Convert(Money from, string target)` checks the currency and multiplies. These classes must not accidentally become `public` — in this task they stay `internal`.

6. **Public façade.** In `PayablesService.cs` make a `public static class PayablesService` that internally uses `ExchangeRateProvider` and `FeeCalculator`. The method `public static Money ConvertUsdToEur(Money usd, decimal feeRate)` must: check that the currency is `USD`; apply the fee via `FeeCalculator`; convert via `ExchangeRateProvider` at rate `0.92m`; round to two decimals. The façade is the only public entry point.

7. **`InternalsVisibleTo`.** In `Payables.Core.csproj` (or in `AssemblyInfo.cs`) add:
   ```xml
   <ItemGroup>
     <InternalsVisibleTo Include="Payables.Core.Tests" />
   </ItemGroup>
   ```
   This lets the test project see `FeeCalculator` and `ExchangeRateProvider`.

8. **Tests.** In `tests/Payables.Core.Tests` write xUnit tests: `Money_Add_SameCurrency_Works`, `Money_Add_DifferentCurrency_Throws`, `Money_Negative_Throws`, `Money_IsSealed` (via reflection: `typeof(Money).IsSealed` should be `true`), `FeeCalculator_Internal_IsAccessible` (creates `new FeeCalculator()` directly — works only thanks to `InternalsVisibleTo`), `PayablesService_ConvertUsdToEur_RoundsToTwoDecimals`. Verify that after removing `InternalsVisibleTo` the test `FeeCalculator_Internal_IsAccessible` stops compiling with error **CS0122**.

9. **Run.** Execute `dotnet build` and `dotnet test`. Expected output: `Passed! - Failed: 0, Passed: N`. All tests green.

10. **Performance note.** Write a short essay (5–8 sentences) in `NOTES.md`: why sealed classes help the JIT with devirtualization, when the gain is real (hot loops, value-type wrappers), and why "sealing for optimization without measuring" is the antipattern named in the lesson.

#### Requirements

- The solution builds under .NET 8 / C# 12 with no warnings (`TreatWarningsAsErrors=true`). Use top-level statements only in the test `Program.cs` (or the standard xUnit template without `Main`).
- All public types in the library are either `public sealed` or `public abstract` with deliberate extensibility; no "accidentally public" classes. Internal services are marked `internal sealed`.
- The `Money` class is immutable: properties are read-only, the constructor validates input, no public setters. The class is sealed and the reason is stated in the XML documentation.
- In the `ReportShape → PieChart → DonutChart` hierarchy the `Render` method on `DonutChart` is marked `sealed override`. The hierarchy classes are not sealed as a whole (this demonstrates the difference: we lock the method, not the entire class).
- The file `NOTES.md` records the compiler error codes (CS0509, CS0239, CS0122) with a one- or two-line explanation each.
- Tests confirm that `typeof(Money).IsSealed == true`, that `FeeCalculator` is reachable from the test (meaning `InternalsVisibleTo` works), and that conversion rounds the result correctly.
- The code is annotated with bilingual comments (RU + EN), as in the lesson example.
- All `internal` members are documented: a short comment explains why they are internal.

#### Pitfalls

- **`sealed` is valid only on `override`.** If you write `public sealed void Render()` in `ReportShape` where the method is not declared `virtual`/`override`, the compiler emits CS0549. There must be a `virtual → override → sealed override` chain first. This is the most common beginner mistake — sealing a method that was never virtual.
- **The `internal` boundary is the assembly, not the namespace.** Do not expect an `internal` type in `Payables.Core` to be visible to another project, even if it lives in the same `namespace Payables`. Two projects = two assemblies = two different boundaries. Many people confuse `internal` with "inside the namespace".
- **`InternalsVisibleTo` requires the exact assembly name.** If the test project is called `Payables.Core.Tests`, the `Include` value must be exactly that. Strong-named (signed) assemblies also require the public key — irrelevant for a study project, but remember it for production libraries.
- **Sealing "for optimization" without measurement is an antipattern.** The lesson warns explicitly: measure with a benchmark (BenchmarkDotNet) first, then seal only if the gain is real. Otherwise you lose extensibility for nothing.
- **`private` ≠ `internal`.** `private` restricts to a single type, `internal` restricts to a single assembly. Do not confuse them: an `internal` field is reachable by every class in the assembly, which is broader than it looks.
- **Do not make members `public` just for tests.** The correct path is `InternalsVisibleTo`. API bloat for testability is a frequent mistake from the lesson.
- **`sealed class` does not forbid access to `protected` members of the base via `base`**, but it forbids new inheritors. Do not confuse "no inheritance" with "no access".
- **Immutable value types should be sealed by default**, otherwise a subclass may add mutable state and break equality semantics (as in the `Money`/`Tip` case).
- **Devirtualization is not guaranteed.** The .NET 8 JIT can devirtualize, but not for every call. Sealing gives the compiler information, but it does not promise inlining — it is an opportunity, not a contract.

#### Acceptance criteria

- [ ] `dotnet build` succeeds with no errors and no warnings (warnings as errors).
- [ ] `dotnet test` passes all tests (`Passed: N, Failed: 0`).
- [ ] `Money` is declared as `public sealed class`, is immutable, and validates arguments in the constructor.
- [ ] An attempt to inherit from `Money` produces **CS0509**, recorded in `NOTES.md`.
- [ ] The `ReportShape → PieChart → DonutChart` hierarchy has `public sealed override void Render()` on `DonutChart`.
- [ ] An attempt to override `Render` in a `DonutChart` descendant produces **CS0239**, recorded in `NOTES.md`.
- [ ] `FeeCalculator` and `ExchangeRateProvider` are declared `internal sealed`.
- [ ] `PayablesService` is the only public façade and uses the internal services.
- [ ] `InternalsVisibleTo` is configured in `.csproj` for `Payables.Core.Tests`.
- [ ] The test `Money_IsSealed` asserts `typeof(Money).IsSealed == true` and passes.
- [ ] The test `FeeCalculator_Internal_IsAccessible` creates `new FeeCalculator()` directly and passes.
- [ ] The test `PayablesService_ConvertUsdToEur_RoundsToTwoDecimals` checks rounding (e.g. `123.456 USD + fee → 113.59 EUR` at rate 0.92).
- [ ] `NOTES.md` contains a devirtualization essay (5–8 sentences).
- [ ] The code has bilingual RU+EN comments.
- [ ] All `internal` members carry a "why internal" comment.
- [ ] The test `Program.cs` (if any) uses top-level statements or the standard xUnit template without `Main`.

#### Hints

- Think about which modifier puts a seal on the whole class and which one seals only a single link in the override chain. These are two different tools.
- To verify a class is actually sealed, in tests use `typeof(T).IsSealed` — a boolean reflection property that does not require an instance.
- Recall that `internal` is the default for classes without a modifier. But writing it explicitly is good style — for readability and self-documentation.
- If a test cannot see an `internal` class, check two things: (1) is `InternalsVisibleTo` added, (2) does the assembly name in `Include` match the real name.
- For rounding money use `Math.Round(value, 2, MidpointRounding.AwayFromZero)` or `ToEven` — but ask yourself which mode is "accountant" and which is "banker".
- Error CS0122 means "inaccessible due to its protection level" — it is the marker that you tried to reach an `internal` type from outside its assembly (or without `InternalsVisibleTo`).

#### Reference solution walk-through

```csharp
// C# 12+ / .NET 8. sealed and internal modifiers in action.

using System;

namespace Payables.Core;

/// <summary>
/// Immutable amount in a currency. Sealed to protect the immutability
/// invariant and to help the JIT with devirtualization.
/// </summary>
public sealed class Money
{
    public decimal Amount { get; }       // read-only.
    public string Currency { get; }

    public Money(decimal amount, string currency)
    {
        if (amount < 0)
            throw new ArgumentOutOfRangeException(nameof(amount),
                "Amount must be non-negative.");
        if (string.IsNullOrWhiteSpace(currency) || currency.Length != 3)
            throw new ArgumentException("Currency must be a 3-letter code.",
                nameof(currency));

        Amount = amount;
        Currency = currency.ToUpperInvariant();
    }

    public Money Add(Money other)
    {
        if (Currency != other.Currency)
            throw new InvalidOperationException("Currency mismatch.");

        return new Money(Amount + other.Amount, Currency);
    }

    public override string ToString() => $"{Amount:0.00} {Currency}";
}

// Report shape hierarchy: Render is sealed at the DonutChart level.
public class ReportShape
{
    public virtual void Render() => Console.WriteLine("shape");
}

public class PieChart : ReportShape
{
    public override void Render() => Console.WriteLine("pie");
}

public class DonutChart : PieChart
{
    // sealed override: overridden for the last time — no further overrides.
    public sealed override void Render() => Console.WriteLine("donut");
}

// internal sealed — internal infrastructure: invisible outside, cannot be inherited.
internal sealed class FeeCalculator
{
    public decimal Apply(decimal amount, decimal feeRate)
    {
        if (feeRate < 0 || feeRate > 1)
            throw new ArgumentOutOfRangeException(nameof(feeRate));
        return amount * (1m - feeRate);
    }
}

internal sealed class ExchangeRateProvider
{
    private readonly decimal _rate;
    public ExchangeRateProvider(decimal rate) => _rate = rate;

    public decimal Convert(Money money, string targetCurrency)
    {
        if (money.Currency == targetCurrency) return money.Amount;
        return money.Amount * _rate;
    }
}

// Public façade — the only entry point for external consumers.
public static class PayablesService
{
    // Internal service singletons (internal, but the façade is in the same assembly).
    private static readonly ExchangeRateProvider UsdToEur = new(0.92m);
    private static readonly FeeCalculator Fees = new();

    public static Money ConvertUsdToEur(Money usd, decimal feeRate)
    {
        if (usd.Currency != "USD")
            throw new ArgumentException("Only USD supported.", nameof(usd));

        decimal afterFee = Fees.Apply(usd.Amount, feeRate);
        decimal eur = UsdToEur.Convert(new Money(afterFee, "USD"), "EUR");
        decimal rounded = Math.Round(eur, 2, MidpointRounding.AwayFromZero);

        return new Money(rounded, "EUR");
    }
}
```

Line-by-line walk-through. The `Money` class is marked `public sealed` — this single decision kills two birds: it protects the immutability invariant (no one can add a mutable `Tip` field) and gives the JIT a signal for devirtualization. The `Amount` and `Currency` properties have only getters — the state is truly immutable. The constructor validates input: negative amounts and bad currencies are rejected at the door, matching the lesson's best practice "protect the contract". The `Add` method checks currency equality — a classic guard against mixing EUR and USD. The XML comment on the class explains why it is sealed: the lesson requires "document the intent".

The `ReportShape → PieChart → DonutChart` hierarchy demonstrates the subtle difference between `sealed class` and `sealed override`. The hierarchy classes are NOT sealed as a whole — you can still derive from `PieChart` in principle. But the `Render` method on `DonutChart` is marked `public sealed override`, which freezes the donut rendering behavior for good: the team decided that the "donut" render strategy must not change in subclasses, while still allowing the hierarchy to grow with new diagram types. This illustrates the lesson's best practice: "apply `sealed override` to lock a method's behavior at a specific hierarchy level without sealing the whole class". Any attempt to write `class FancyDonut : DonutChart { public override void Render() {} }` fails with CS0239.

The classes `FeeCalculator` and `ExchangeRateProvider` are declared `internal sealed` — the strictest and at the same time the most refactor-friendly combination recommended in the lesson. They are invisible outside `Payables.Core` (no breaking changes on rename), they cannot be inherited (no surprises with invariants), and yet they are fully reachable inside the assembly. The façade `PayablesService` is the only `public` entry point: it encapsulates the internal services and exposes a deliberate public API. This realizes the principle "minimize the public surface".

Finally, `InternalsVisibleTo` (configured in `.csproj`) opens the test project's access to `FeeCalculator` and `ExchangeRateProvider` — the correct way to test internals without bloating the API, as the lesson demands. The test `FeeCalculator_Internal_IsAccessible` creates `new FeeCalculator()` directly, which would be impossible without `InternalsVisibleTo` (error CS0122). Every concept from the lesson is applied: sealing classes, `sealed override`, `internal`, `internal sealed`, `InternalsVisibleTo`, documenting intent, minimizing the public surface.

#### Going deeper (bonus)

1. **Devirtualization benchmark.** Using BenchmarkDotNet, compare a hot loop summing `Money` in the sealed version against an unsealed one (`public class MoneyUnsealed`). Write two benchmarks, run them, record the difference in nanoseconds. Explain when the win is real and when it is unnoticeable.
2. **`InternalsVisibleTo` with signing.** Figure out how `InternalsVisibleTo` works for strong-named assemblies: generate a `.snk`, sign both assemblies, configure `Include` with the public key. Record the steps in `NOTES.md`.
3. **`sealed` + records.** Rewrite `Money` as a `public sealed record Money`. Compare the semantics: what the record syntax gives you (value equality, `with`), and whether `sealed` is even needed here. Discuss whether records are inheritable by default.
4. **Locking invariants without `sealed`.** Investigate whether the immutability invariant can be protected without `sealed`, using only `init`-only setters and private constructors. Discuss why such an approach is weaker than `sealed`.

---

#### Чек-лист сдачи / Submission checklist
- [ ] Создано решение `Payables.sln` с проектами `Payables.Core` и `Payables.Core.Tests`.
- [ ] `Money` — `public sealed class`, неизменяем, с валидацией и XML-комментарием.
- [ ] В `NOTES.md` зафиксирована ошибка CS0509 при попытке наследования.
- [ ] Иерархия фигур содержит `sealed override` на `DonutChart`; в `NOTES.md` зафиксирована CS0239.
- [ ] `FeeCalculator` и `ExchangeRateProvider` — `internal sealed`.
- [ ] `PayablesService` — единственный публичный фасад.
- [ ] Настроен `InternalsVisibleTo`; в `NOTES.md` отмечена CS0122 без него.
- [ ] Все xUnit-тесты зелёные; есть `Money_IsSealed` и `FeeCalculator_Internal_IsAccessible`.
- [ ] Код снабжён двуязычными комментариями RU+EN.
- [ ] Эссе о devirtualization в `NOTES.md` (5–8 предложений).
- [ ] Solution `Payables.sln` created with `Payables.Core` and `Payables.Core.Tests` projects.
- [ ] `Money` is a `public sealed class`, immutable, with validation and an XML comment.
- [ ] `NOTES.md` records error CS0509 on attempted inheritance.
- [ ] The shape hierarchy has `sealed override` on `DonutChart`; `NOTES.md` records CS0239.
- [ ] `FeeCalculator` and `ExchangeRateProvider` are `internal sealed`.
- [ ] `PayablesService` is the only public façade.
- [ ] `InternalsVisibleTo` is configured; `NOTES.md` notes CS0122 without it.
- [ ] All xUnit tests are green; `Money_IsSealed` and `FeeCalculator_Internal_IsAccessible` are present.
- [ ] Code has bilingual RU+EN comments.
- [ ] Devirtualization essay in `NOTES.md` (5–8 sentences).

#### Ресурсы / Resources
- [Microsoft Learn — sealed](https://learn.microsoft.com/dotnet/csharp/language-reference/keywords/sealed)
- [Microsoft Learn — internal](https://learn.microsoft.com/dotnet/csharp/language-reference/keywords/internal)
- [Microsoft Learn — InternalsVisibleToAttribute](https://learn.microsoft.com/dotnet/api/system.runtime.compilerservices.internalsvisibletoattribute)
- [Compiler errors CS0122 / CS0239 / CS0509](https://learn.microsoft.com/dotnet/csharp/language-reference/compiler-messages/)
- [BenchmarkDotNet — devirtualization](https://benchmarkdotnet.org/)
