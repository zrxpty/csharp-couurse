---
[← К уроку M07-L01](lesson-M07-L01-exceptions-hierarchy.md) | [⬆ К модулю M07](../README.md) | [Следующее ДЗ →](homework-M07-L02-try-catch-finally.md)
---

### Домашнее задание M07-L01: Исключения vs коды возврата, иерархия Exception / Homework M07-L01: Exceptions vs return codes, Exception hierarchy

**Урок / Lesson:** M07-L01
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 3/5
**Цель / Goal:** (RU) Научиться различать аварийные ситуации и ожидаемые бизнес-исходы, грамотно проектировать собственную иерархию исключений, унаследованную напрямую от `System.Exception`, и выбирать между исключениями и кодами возврата / типом `Result<T>` в реалистичной доменной задаче. (EN) Learn to distinguish accidents from expected business outcomes, design a custom exception hierarchy derived directly from `System.Exception`, and choose between exceptions and return codes / a `Result<T>` type in a realistic domain task.

#### Связь с уроком / Connection to the lesson
(RU) Это ДЗ опирается на три фундаментальных правила из урока M07-L01: бросать исключения только для непредвиденных ошибок, наследовать свои исключения от `Exception` (а не от устаревшего `ApplicationException`), и не использовать исключения для обычного потока управления. Вы будете строить мини-сервис заказов, в котором одни ситуации моделируются `throw`, а другие — значением `Result<T>`. (EN) This homework builds on three foundational rules from lesson M07-L01: throw exceptions only for unexpected errors, derive custom exceptions from `Exception` (not from the deprecated `ApplicationException`), and never use exceptions for normal control flow. You will build a mini order service in which some situations are modeled with `throw` and others with a `Result<T>` value.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация
Вы — backend-разработчик небольшого интернет-магазина «КотоМаркет». Сейчас в кодовой базе царит хаос: половина методов возвращает `-1`, `null` или строку с кодом ошибки вроде `"ERR_42"`, а другая половина бросает базовое `Exception` с сообщением «что-то пошло не так». Из-за этого вызывающий код постоянно забывает проверять результат, ошибки молча проглатываются, а в production периодически всплывают «пустые коробки» — заказы, которые оформились, но без товаров, потому что валидация была проигнорирована.

Ваш техлид поручил вам навести порядок в одном доменном модуле — `OrderPricing`. Этот модль отвечает за расчёт итоговой цены заказа: скидки, купоны, налоги. Нужно переписать его так, чтобы: (1) нарушения контракта метода (null-аргументы, пустые `Guid`, отрицательные количества) обрабатывались через исключения с конкретными типами, потому что это аварии и программистские ошибки, которые нельзя игнорировать; (2) ожидаемые бизнес-ситуации вроде «купон истёк», «товар распродан», «клиент не найден в лояльности» возвращались через тип-результат `Result<T>`, чтобы вызывающий код мог ветвиться без `try/catch`; (3) все собственные исключения наследовались напрямую от `Exception`, имели осмысленное имя и конструктор с `innerException`.

Эта задача — не учебная абстракция. В реальных .NET-проектах именно так и проектируют границы слоёв: контрактные нарушения «кричат» через исключения, а ожидаемые ветки логики живут в `Result`. Пройдя это ДЗ, вы получите мышечную память для границы «авария vs нормальный исход» — главного навыка всего модуля M07. Вы также потрогаете цену исключений: измерите, насколько `throw` медленнее `return`, и убедитесь, что исключения — это инструмент для редких сбоев, а не для обычной логики.

#### Что нужно сделать (пошагово)
1. Создайте новый проект: `dotnet new console -n CatMarket.Pricing -o CatMarket.Pricing` в подходящей директории. Убедитесь, что используется .NET 8: `dotnet --version` должен показать `8.x`. Откройте `CatMarket.Pricing.csproj` и проверьте, что `<TargetFramework>net8.0</TargetFramework>` присутствует.
2. В папке проекта создайте файл `Result.cs` с типом `Result<T>`. Используйте `readonly record struct` (как в уроке): поля `T Value` и `string? Error`, свойство `bool IsSuccess => Error is null`. Добавьте два статических фабричных метода `Result<T>.Ok(T value)` и `Result<T>.Fail(string error)` для удобства.
3. Создайте файл `DomainExceptions.cs`. Определите в нём как минимум два собственных исключения: `PricingException` (базовое для всего модуля) и `InvalidOrderLineException : PricingException`. Каждое должно иметь три конструктора: без параметров, с `string message`, и с `string message, Exception inner`. Используйте `sealed` для конечных типов. В комментариях RU+EN явно укажите, что наследование идёт от `Exception`, а не от `ApplicationException`, и почему.
4. Создайте файл `PricingService.cs`. Реализуйте класс `PricingService` с методом `Result<decimal> CalculateTotal(Order order, Coupon? coupon, LoyaltyTier tier)`. В начале метода бросайте `ArgumentNullException.ThrowIfNull(order)` для `order == null`. Если `order.Lines` пуст — бросайте `InvalidOrderLineException` с сообщением о пустом заказе (это нарушение контракта, авария). Если в `order.Lines` есть строка с `Quantity <= 0` или `UnitPrice < 0` — тоже бросайте `InvalidOrderLineException` (fail fast).
5. Для ожидаемых бизнес-ситуаций возвращайте `Result<decimal>.Fail(...)` без throw: если купон `coupon` не null, но `coupon.ExpiresAt < DateTimeOffset.UtcNow` — верните `Fail("COUPON_EXPIRED")`; если `tier == LoyaltyTier.None` — примените скидку 0%, иначе 5% / 10% / 15% по уровням `Bronze`/`Silver`/`Gold`.
6. В `Program.cs` вызовите `CalculateTotal` для трёх сценариев: (а) валидный заказ с купоном и лояльностью — ожидайте `IsSuccess == true` и ненулевую цену; (б) просроченный купон — ожидайте `IsSuccess == false`, `Error == "COUPON_EXPIRED"`, без `try/catch`; (в) заказ с `Quantity = 0` — оберните вызов в `try/catch (InvalidOrderLineException ex)` и распечатайте `ex.Message`.
7. Запустите `dotnet build` — должно быть 0 ошибок и 0 предупреждений. Запустите `dotnet run`. Ожидаемый вывод в консоли:
   - `Total: <значение>` для первого сценария,
   - `Error: COUPON_EXPIRED` для второго,
   - `Caught: <сообщение InvalidOrderLineException>` для третьего.
8. В отдельном файле `BenchException.cs` напишите простую микро-бенчмарку: функция, которая в цикле 1 000 000 раз либо `return -1`, либо `throw new InvalidOperationException()` (с `try/catch` вокруг). Замерьте `Stopwatch` для обоих вариантов. В `Program.cs` распечатайте оба времени. Убедитесь, что исключения в десятки/сотни раз медленнее.
9. Сделайте коммит в git: `git add -A && git commit -m "M07-L01: exceptions vs return codes"`. При необходимости `git init` сначала.
10. Прогоните `dotnet format` (если установлен) или вручную проверьте стиль.

#### Требования к решению
- Код должен компилироваться под C# 12 / .NET 8 без ошибок и предупреждений (включая nullable-аннотации: `<Nullable>enable</Nullable>` в `.csproj`).
- Все собственные исключения наследуются напрямую от `System.Exception`, помечены `sealed` (кроме базового `PricingException`, который может быть не sealed, если от него наследуются), и имеют полный набор из трёх стандартных конструкторов.
- Категорически запрещено наследоваться от `ApplicationException` — за это снимаются баллы. В комментарии к классу должно быть объяснение, почему.
- Метод `CalculateTotal` должен чётко разделять: контрактные нарушения → `throw` конкретного типа; ожидаемые бизнес-исходы → `Result<decimal>.Fail(...)`. Смешивание недопустимо: нельзя возвращать `Fail` для `order == null` и нельзя бросать исключение для `COUPON_EXPIRED`.
- Запрещён пустой `catch (Exception) { }`. Если где-то ловите — логируйте в `Console.Error` или пробрасывайте `throw;`.
- Тип `Result<T>` должен быть immutable (`readonly record struct`) и не должен выбрасывать исключения из своих членов.
- Имена файлов и типов должны совпадать с шагами выше. Папка проекта — `CatMarket.Pricing`.
- Микро-бенчмарка должна честно измерять обе ветки и печатать абсолютные числа в миллисекундах. Не нужно использовать BenchmarkDotNet — достаточно `Stopwatch`.

#### Тонкости и подводные камни
- **`SystemException` vs `ApplicationException`**: из урока вы помните, что `ApplicationException` устарел и Microsoft прямо не рекомендует его использовать. `SystemException` тоже не стоит использовать как базовый класс для своих исключений — он предназначен для ошибок среды выполнения (`NullReferenceException`, `IndexOutOfRangeException`, `StackOverflowException`, `IOException`). Наследуйтесь напрямую от `Exception`. Это ключевая тонкость: многие учебники старой школы до сих показывают `: ApplicationException`, но в .NET 8 это анти-паттерн.
- **`IOException`, `InvalidOperationException`, `ArgumentException`**: для файловых ошибок используйте `IOException` и его производные (`FileNotFoundException`, `DirectoryNotFoundException`); для нарушений логического состояния объекта, не подходящих под контракт аргумента, — `InvalidOperationException`; для невалидных аргументов — `ArgumentException` и его производные `ArgumentNullException`, `ArgumentOutOfRangeException`. В этом ДЗ вы создаёте `InvalidOrderLineException`, потому что это доменная авария, а не стандартная категория.
- **Когда исключения оправданы vs коды возврата**: исключение оправдано, если (а) событие непредвиденное и текущий код не может обработать его локально, (б) игнорирование ошибки ведёт к повреждению данных, (в) это нарушение контракта. Коды возврата / `Result<T>` оправданы, если событие ожидаемое и является частью нормального потока: «пользователь не найден», «купон истёк», «товар распродан». Главное правило: если вызывающий код **ожидает** ветвиться по этому исходу, делайте его `Result`.
- **Цена исключений**: создание исключения захватывает стек-трейс, разматывает стек, и это в сотни раз медленнее обычного `return`. Ваша микро-бенчмарка должна это показать. Поэтому никогда не используйте `throw` в горячих путях (циклы, парсинг миллионов строк).
- **.NET 8 события и фильтры**: в этом ДЗ фильтры (`when`) пока не нужны (они будут в M07-L02), но помните, что .NET 8 поддерживает exception filters и `AppDomain.FirstChanceException` для мониторинга. События домена (`AppDomain.UnhandledException`, `TaskScheduler.UnobservedTaskException`) используются для логирования на верхнем уровне — не для управления потоком.
- **`throw;` vs `throw ex;`**: если где-то перехватываете и пробрасываете, всегда используйте `throw;` (без `ex`), чтобы не обнулять стек-трейс. Это тонкость, которая не раз спасёт вас в production-дебаге.
- **Конструктор с `innerException`**: обязателен для каждого кастомного исключения. Он позволяет сохранить причинно-следственную цепочку: например, если `IOException` произошёл при чтении прайс-листа, оберните его в `PricingException("Не удалось загрузить прайс", ioEx)`.

#### Критерии приёмки
- [ ] Проект `CatMarket.Pricing` создан, `.csproj` содержит `<TargetFramework>net8.0</TargetFramework>` и `<Nullable>enable</Nullable>`.
- [ ] `dotnet build` проходит без ошибок и предупреждений.
- [ ] `dotnet run` выводит три строки: `Total: ...`, `Error: COUPON_EXPIRED`, `Caught: ...` в указанном порядке.
- [ ] Тип `Result<T>` реализован как `readonly record struct` с `Ok` и `Fail` фабриками.
- [ ] `PricingException` наследуется от `Exception` и не sealed.
- [ ] `InvalidOrderLineException` наследуется от `PricingException`, помечен `sealed`.
- [ ] В обоих классах есть три стандартных конструктора (без параметров, message, message+inner).
- [ ] Нигде в коде нет наследования от `ApplicationException`.
- [ ] В комментарии к `PricingException` явно написано, почему не `ApplicationException` (RU+EN).
- [ ] `CalculateTotal` бросает `ArgumentNullException` для `order == null` через `ArgumentNullException.ThrowIfNull`.
- [ ] `CalculateTotal` бросает `InvalidOrderLineException` для пустого заказа и для `Quantity <= 0` / `UnitPrice < 0`.
- [ ] `CalculateTotal` возвращает `Result<decimal>.Fail("COUPON_EXPIRED")` для просроченного купона без throw.
- [ ] Лояльность `Bronze`/`Silver`/`Gold` даёт 5%/10%/15% скидку; `None` — 0%.
- [ ] В `Program.cs` нет пустого `catch (Exception) { }`; везде либо логирование, либо `throw;`.
- [ ] Микро-бенчмарка честно измеряет обе ветки и печатает время в миллисекундах; исключения显著 медленнее.
- [ ] Git-коммит сделан с сообщением `M07-L01: exceptions vs return codes`.

#### Подсказки (без прямого ответа)
- Вспомните пример `OrderNotFoundException` из урока: та же структура трёх конструкторов, `sealed`, наследование от `Exception`.
- Для `Result<T>.Ok` возвращайте `new Result<T>(value, null)`, для `Fail` — `new Result<T>(default, error)`.
- `DateTimeOffset.UtcNow` — для проверки срока купона; сравнивайте с `coupon.ExpiresAt`.
- Для бенчмарки используйте `System.Diagnostics.Stopwatch`, `var sw = Stopwatch.StartNew();`. Цикл 1 000 000 итераций достаточен, чтобы увидеть разницу.
- Помните: `throw;` сохраняет стек, `throw ex;` — нет.

#### Эталонное решение (разбор)
```csharp
// C# 12 / .NET 8 — Эталон решения ДЗ M07-L01
// Reference solution for homework M07-L01
// Exceptions vs return codes + custom exception hierarchy.

using System;
using System.Collections.Generic;
using System.Diagnostics;

// Result<T> — тип-результат для ожидаемых бизнес-исходов.
// Result<T> — a result type for expected business outcomes.
// Не выбрасывает исключения, immutable.
// Does not throw, immutable.
public readonly record struct Result<T>(T Value, string? Error)
{
    public bool IsSuccess => Error is null;

    public static Result<T> Ok(T value) => new(value, null);
    public static Result<T> Fail(string error) => new(default, error);
}

// Базовое доменное исключение модуля расчёта цен.
// Base domain exception for the pricing module.
// ВАЖНО: наследуемся напрямую от Exception, а НЕ от ApplicationException.
// IMPORTANT: derive directly from Exception, NOT from ApplicationException.
// ApplicationException устарел и не несёт семантики — Microsoft это запрещает.
// ApplicationException is deprecated and carries no semantics — Microsoft forbids it.
public class PricingException : Exception
{
    public PricingException() { }
    public PricingException(string message) : base(message) { }
    public PricingException(string message, Exception inner) : base(message, inner) { }
}

// Авария: невалидная строка заказа. sealed — от него не наследуются.
// Accident: invalid order line. sealed — no further derivation.
public sealed class InvalidOrderLineException : PricingException
{
    public InvalidOrderLineException() { }
    public InvalidOrderLineException(string message) : base(message) { }
    public InvalidOrderLineException(string message, Exception inner) : base(message, inner) { }
}

// Доменные типы.
// Domain types.
public enum LoyaltyTier { None, Bronze, Silver, Gold }

public sealed class OrderLine
{
    public string Sku { get; init; } = string.Empty;
    public int Quantity { get; init; }
    public decimal UnitPrice { get; init; }
}

public sealed class Order
{
    public Guid Id { get; init; }
    public IReadOnlyList<OrderLine> Lines { get; init; } = Array.Empty<OrderLine>();
}

public sealed class Coupon
{
    public string Code { get; init; } = string.Empty;
    public decimal DiscountPercent { get; init; }
    public DateTimeOffset ExpiresAt { get; init; }
}

public static class PricingService
{
    public static Result<decimal> CalculateTotal(Order order, Coupon? coupon, LoyaltyTier tier)
    {
        // Контрактное нарушение — авария, fail fast через исключение.
        // Contract violation — accident, fail fast via exception.
        ArgumentNullException.ThrowIfNull(order);

        if (order.Lines.Count == 0)
            throw new InvalidOrderLineException(
                "Заказ не содержит строк / Order has no lines");

        decimal subtotal = 0m;
        foreach (var line in order.Lines)
        {
            if (line.Quantity <= 0)
                throw new InvalidOrderLineException(
                    $"Quantity <= 0 для SKU {line.Sku} / Quantity <= 0 for SKU {line.Sku}");
            if (line.UnitPrice < 0)
                throw new InvalidOrderLineException(
                    $"UnitPrice < 0 для SKU {line.Sku} / UnitPrice < 0 for SKU {line.Sku}");
            subtotal += line.Quantity * line.UnitPrice;
        }

        // Ожидаемый бизнес-исход — просроченный купон.
        // Expected business outcome — expired coupon.
        // НЕ бросаем исключение: вызывающий ожидает ветвление.
        // Do NOT throw: the caller expects to branch.
        if (coupon is not null && coupon.ExpiresAt < DateTimeOffset.UtcNow)
            return Result<decimal>.Fail("COUPON_EXPIRED");

        decimal discount = tier switch
        {
            LoyaltyTier.Bronze => 0.05m,
            LoyaltyTier.Silver => 0.10m,
            LoyaltyTier.Gold   => 0.15m,
            _                  => 0.00m,
        };

        decimal couponDiscount = coupon is null ? 0m : coupon.DiscountPercent;
        decimal totalDiscount = discount + couponDiscount;
        decimal total = subtotal * (1m - totalDiscount);

        return Result<decimal>.Ok(Math.Round(total, 2));
    }
}

public static class Program
{
    public static void Main()
    {
        var order = new Order
        {
            Id = Guid.NewGuid(),
            Lines = new List<OrderLine>
            {
                new() { Sku = "CAT-FOOD-1", Quantity = 2, UnitPrice = 499.00m },
                new() { Sku = "TOY-MOUSE",  Quantity = 1, UnitPrice = 199.00m },
            }
        };

        // (а) Валидный заказ + купон + лояльность.
        var validCoupon = new Coupon
        {
            Code = "WELCOME10",
            DiscountPercent = 0.10m,
            ExpiresAt = DateTimeOffset.UtcNow.AddHours(1),
        };
        var ok = PricingService.CalculateTotal(order, validCoupon, LoyaltyTier.Gold);
        Console.WriteLine(ok.IsSuccess
            ? $"Total: {ok.Value}"
            : $"Error: {ok.Error}");

        // (б) Просроченный купон — без try/catch, через Result.
        var expiredCoupon = new Coupon
        {
            Code = "OLD",
            DiscountPercent = 0.10m,
            ExpiresAt = DateTimeOffset.UtcNow.AddHours(-1),
        };
        var expired = PricingService.CalculateTotal(order, expiredCoupon, LoyaltyTier.Bronze);
        Console.WriteLine(expired.IsSuccess
            ? $"Total: {expired.Value}"
            : $"Error: {expired.Error}");

        // (в) Авария — невалидная строка, ловим конкретный тип.
        var badOrder = new Order
        {
            Id = Guid.NewGuid(),
            Lines = new List<OrderLine> { new() { Sku = "BAD", Quantity = 0, UnitPrice = 10m } },
        };
        try
        {
            _ = PricingService.CalculateTotal(badOrder, null, LoyaltyTier.None);
        }
        catch (InvalidOrderLineException ex)
        {
            Console.WriteLine($"Caught: {ex.Message}");
        }

        // Микро-бенчмарка: цена исключений.
        // Micro-benchmark: the cost of exceptions.
        BenchException.Run();
    }
}

internal static class BenchException
{
    public static void Run()
    {
        const int N = 1_000_000;

        var swReturn = Stopwatch.StartNew();
        for (int i = 0; i < N; i++)
        {
            var code = ReturnCode(i);
            if (code == -1) { /* simulate check */ }
        }
        swReturn.Stop();

        var swThrow = Stopwatch.StartNew();
        for (int i = 0; i < N; i++)
        {
            try
            {
                ThrowIfNegative(i - 1);
            }
            catch (InvalidOperationException) { /* swallow in benchmark only */ }
        }
        swThrow.Stop();

        Console.WriteLine($"Return: {swReturn.ElapsedMilliseconds} ms, " +
                          $"Throw: {swThrow.ElapsedMilliseconds} ms");
    }

    private static int ReturnCode(int x) => x < 0 ? -1 : x;
    private static void ThrowIfNegative(int x)
    {
        if (x < 0) throw new InvalidOperationException("negative");
    }
}
```

Разбор по строкам. `Result<T>` построен как `readonly record struct` ровно по образцу из урока — это даёт value-семантику, иммутабельность и авто-генерируемый `Equals`/`GetHashCode`. Методы `Ok` и `Fail` — это удобные фабрики, которые делают код вызывающей стороны читаемым (`Result<decimal>.Ok(total)` вместо `new Result<decimal>(total, null)`). `IsSuccess => Error is null` — нулевой `Error` означает успех; паттерн-матчинг `is null` корректен даже для value-типов.

`PricingException` наследуется напрямую от `Exception` (строка `: Exception`), а не от `ApplicationException`. В комментарии явно зафиксировано почему: `ApplicationException` устарел, Microsoft его не рекомендует, и он не несёт семантики. Это применение правила (2) из урока. Три конструктора соответствуют стандартной идиоме .NET: без параметров (для сериализации и крайних случаев), только message, и message + inner. Третий конструктор критически важен — он позволяет передать оригинальную причину (например, `IOException`) и не потерять её при логировании на верхнем уровне.

`InvalidOrderLineException` помечен `sealed` и наследуется от `PricingException`. Это даёт двухуровневую иерархию: можно ловить либо конкретно `InvalidOrderLineException`, либо «всё, что связано с прайсингом» через `catch (PricingException)`. sealed предотвращает дальнейшее неконтролируемое наследование и слегка ускоряет виртуальную диспетчеризацию. Конструкторы дублируют три формы — это обязательно для корректной сериализации и общепринятый стиль.

В `CalculateTotal` ключевой момент — разделение аварий и ожидаемых исходов. `ArgumentNullException.ThrowIfNull(order)` — это .NET 7+ валидатор, короче и быстрее ручной проверки; он бросает `ArgumentNullException` с именем параметра автоматически. Пустой `order.Lines` и невалидные `Quantity`/`UnitPrice` — это нарушения контракта (вызывающий передал невалидные данные), поэтому здесь `throw new InvalidOrderLineException(...)`. Это правило (1) и паттерн «fail fast» из урока: валидация в начале метода, конкретный тип исключения.

А вот просроченный купон — это `return Result<decimal>.Fail("COUPON_EXPIRED")`, а не throw. Почему? Потому что «купон истёк» — ожидаемый бизнес-сценарий: маркетинг может проводить кампании с истекающими купонами, и вызывающий код должен уметь показать пользователю «ваш купон больше не действует». Это правило (3) — не использовать исключения для обычного потока управления. Строка `if (coupon is not null && coupon.ExpiresAt < DateTimeOffset.UtcNow)` проверяет срок, а `return` отдаёт результат без разматывания стека. Скидки по лояльности и купону суммируются через `switch`-выражение — современный C# 12.

В `Program.Main` три сценария демонстрируют разницу. Сценарий (а) — успех, читаем `ok.Value`. Сценарий (б) — ожидаемая неудача, читаем `expired.Error`, никакого `try/catch`. Сценарий (в) — авария, ловим конкретно `InvalidOrderLineException` (а не базовый `Exception`), что позволяет писать точные обработчики. В бенчмарке `BenchException.Run` видно, что `throw` в цикле на 1 000 000 итераций в десятки-сотни раз медленнее `return` — это практическое подтверждение тезиса из урока о цене исключений. Заметьте: пустой `catch (InvalidOperationException) { }` в бенчмарке — это исключение из правила «не глотать», оправданное тем, что цель именно измерить цену throw/catch, а не обработать ошибку; в продакшен-коде так делать нельзя.

#### Задания на углубление (бонус)
1. Добавьте третий собственный тип `CatalogUnavailableException : PricingException` с конструктором, принимающим `IOException inner`. Смоделируйте в `CalculateTotal` чтение прайс-листа из файла, и при `IOException` оборачивайте его в `CatalogUnavailableException("Не удалось загрузить прайс", ioEx)`. Поймайте на верхнем уровне и распечатайте `ex.InnerException?.Message`.
2. Реализуйте перегрузку `Result<T>` с тегом-enum: `enum ResultKind { Ok, Fail }` вместо проверки `Error is null`. Сравните читабельность обоих подходов.
3. Расширьте бенчмарку: измерьте стоимость `throw` при разной глубине стека (1, 5, 10 фреймов). Постройте вывод о том, насколько глубина влияет на цену.
4. Напишите метод `ParseSku(string input)`, который для невалидного формата возвращает `Result<string>.Fail("BAD_FORMAT")`, а для `null` бросает `ArgumentNullException`. Объясните в комментарии, почему это разные категории.

---

## Statement in English / Постановка на английском

#### Context & motivation
You are a backend developer at a small online store called "CatMarket". The current code base is chaotic: half of the methods return `-1`, `null`, or a string error code like `"ERR_42"`, while the other half throw the base `Exception` with a generic "something went wrong" message. As a result, callers constantly forget to check the return value, errors get silently swallowed, and in production you keep finding "empty boxes" — orders that were placed but have no items, because validation was skipped.

Your tech lead has tasked you with cleaning up one domain module, `OrderPricing`. This module is responsible for computing the final price of an order: discounts, coupons, taxes. You must rewrite it so that: (1) contract violations of a method (null arguments, empty `Guid`, non-positive quantities) are handled through exceptions with concrete types, because these are accidents and programmer errors that cannot be ignored; (2) expected business situations such as "coupon expired", "item sold out", "customer not found in loyalty" are returned through a `Result<T>` type, so the caller can branch without `try/catch`; (3) all custom exceptions derive directly from `Exception`, have a meaningful name, and provide a constructor that accepts `innerException`.

This task is not an academic abstraction. In real .NET projects this is exactly how layer boundaries are designed: contract violations "shout" through exceptions, while expected logic branches live in `Result`. By completing this homework you will build muscle memory for the "accident vs normal outcome" boundary — the core skill of the entire M07 module. You will also touch the cost of exceptions: you will measure how much slower `throw` is compared to `return`, and you will see for yourself that exceptions are a tool for rare failures, not for ordinary logic.

#### What to do step by step
1. Create a new project: `dotnet new console -n CatMarket.Pricing -o CatMarket.Pricing` in a suitable directory. Make sure you are on .NET 8: `dotnet --version` should print `8.x`. Open `CatMarket.Pricing.csproj` and confirm that `<TargetFramework>net8.0</TargetFramework>` is present.
2. Inside the project folder create a file `Result.cs` with a `Result<T>` type. Use a `readonly record struct` (as in the lesson): fields `T Value` and `string? Error`, property `bool IsSuccess => Error is null`. Add two static factory methods `Result<T>.Ok(T value)` and `Result<T>.Fail(string error)` for convenience.
3. Create a file `DomainExceptions.cs`. Define at least two custom exceptions in it: `PricingException` (the base for the whole module) and `InvalidOrderLineException : PricingException`. Each must have three constructors: parameterless, with `string message`, and with `string message, Exception inner`. Mark leaf types `sealed`. In RU+EN comments explicitly state that the derivation is from `Exception`, not from `ApplicationException`, and explain why.
4. Create a file `PricingService.cs`. Implement a `PricingService` class with a method `Result<decimal> CalculateTotal(Order order, Coupon? coupon, LoyaltyTier tier)`. At the start of the method throw `ArgumentNullException.ThrowIfNull(order)` for `order == null`. If `order.Lines` is empty, throw `InvalidOrderLineException` with a message about an empty order (this is a contract violation, an accident). If any line in `order.Lines` has `Quantity <= 0` or `UnitPrice < 0`, also throw `InvalidOrderLineException` (fail fast).
5. For expected business situations return `Result<decimal>.Fail(...)` without throwing: if `coupon` is not null but `coupon.ExpiresAt < DateTimeOffset.UtcNow`, return `Fail("COUPON_EXPIRED")`; if `tier == LoyaltyTier.None`, apply a 0% discount, otherwise 5% / 10% / 15% for `Bronze` / `Silver` / `Gold`.
6. In `Program.cs` call `CalculateTotal` for three scenarios: (a) a valid order with a coupon and loyalty — expect `IsSuccess == true` and a non-zero price; (b) an expired coupon — expect `IsSuccess == false`, `Error == "COUPON_EXPIRED"`, without `try/catch`; (c) an order with `Quantity = 0` — wrap the call in `try/catch (InvalidOrderLineException ex)` and print `ex.Message`.
7. Run `dotnet build` — there must be 0 errors and 0 warnings. Run `dotnet run`. The expected console output is:
   - `Total: <value>` for the first scenario,
   - `Error: COUPON_EXPIRED` for the second,
   - `Caught: <InvalidOrderLineException message>` for the third.
8. In a separate file `BenchException.cs` write a simple micro-benchmark: a function that, in a loop of 1,000,000 iterations, either `return -1` or `throw new InvalidOperationException()` (with a `try/catch` around it). Measure both variants with `Stopwatch`. In `Program.cs` print both timings. Confirm that exceptions are tens to hundreds of times slower.
9. Commit to git: `git add -A && git commit -m "M07-L01: exceptions vs return codes"`. If needed, run `git init` first.
10. Run `dotnet format` (if installed) or check the style manually.

#### Requirements
- The code must compile under C# 12 / .NET 8 with no errors and no warnings (including nullable annotations: `<Nullable>enable</Nullable>` in the `.csproj`).
- All custom exceptions derive directly from `System.Exception`; leaf types are marked `sealed` (the base `PricingException` may be non-sealed if other types derive from it), and all of them provide the full set of three standard constructors.
- Deriving from `ApplicationException` is strictly forbidden and will cost points. The class comment must explain why.
- The `CalculateTotal` method must clearly separate: contract violations → `throw` a concrete type; expected business outcomes → `Result<decimal>.Fail(...)`. Mixing is not allowed: you may not return `Fail` for `order == null`, and you may not throw for `COUPON_EXPIRED`.
- An empty `catch (Exception) { }` is forbidden. If you catch anywhere, log to `Console.Error` or rethrow with `throw;`.
- The `Result<T>` type must be immutable (`readonly record struct`) and must not throw from any of its members.
- File and type names must match the steps above. The project folder is `CatMarket.Pricing`.
- The micro-benchmark must honestly measure both branches and print absolute numbers in milliseconds. BenchmarkDotNet is not required — a `Stopwatch` is enough.

#### Pitfalls
- **`SystemException` vs `ApplicationException`**: as you remember from the lesson, `ApplicationException` is deprecated and Microsoft explicitly discourages it. `SystemException` should also not be used as a base for your own exceptions — it is meant for runtime errors (`NullReferenceException`, `IndexOutOfRangeException`, `StackOverflowException`, `IOException`). Derive directly from `Exception`. This is the key pitfall: many old textbooks still show `: ApplicationException`, but in .NET 8 this is an anti-pattern.
- **`IOException`, `InvalidOperationException`, `ArgumentException`**: for file errors use `IOException` and its derived types (`FileNotFoundException`, `DirectoryNotFoundException`); for violations of an object's logical state that do not fit the argument contract, use `InvalidOperationException`; for invalid arguments use `ArgumentException` and its derived types `ArgumentNullException`, `ArgumentOutOfRangeException`. In this homework you create `InvalidOrderLineException` because it is a domain accident, not a standard category.
- **When exceptions are justified vs return codes**: an exception is justified if (a) the event is unexpected and the current code cannot handle it locally, (b) ignoring the error leads to data corruption, (c) it is a contract violation. Return codes / `Result<T>` are justified if the event is expected and part of the normal flow: "user not found", "coupon expired", "item sold out". The main rule: if the caller **expects** to branch on this outcome, make it a `Result`.
- **The cost of exceptions**: creating an exception captures a stack trace, unwinds the stack, and is hundreds of times slower than a plain `return`. Your micro-benchmark should demonstrate this. Never use `throw` in hot paths (loops, parsing millions of rows).
- **.NET 8 events and filters**: this homework does not need filters (`when`) yet (they will appear in M07-L02), but remember that .NET 8 supports exception filters and `AppDomain.FirstChanceException` for monitoring. Domain events (`AppDomain.UnhandledException`, `TaskScheduler.UnobservedTaskException`) are used for top-level logging — not for flow control.
- **`throw;` vs `throw ex;`**: if you catch and rethrow anywhere, always use `throw;` (without `ex`) so the stack trace is not reset. This pitfall will save you more than once in production debugging.
- **The `innerException` constructor**: mandatory for every custom exception. It preserves the causal chain: for example, if an `IOException` occurred while reading the price list, wrap it as `PricingException("Failed to load price list", ioEx)`.

#### Acceptance criteria
- [ ] The `CatMarket.Pricing` project is created; the `.csproj` contains `<TargetFramework>net8.0</TargetFramework>` and `<Nullable>enable</Nullable>`.
- [ ] `dotnet build` succeeds with no errors and no warnings.
- [ ] `dotnet run` prints three lines: `Total: ...`, `Error: COUPON_EXPIRED`, `Caught: ...` in this order.
- [ ] The `Result<T>` type is implemented as a `readonly record struct` with `Ok` and `Fail` factories.
- [ ] `PricingException` derives from `Exception` and is not sealed.
- [ ] `InvalidOrderLineException` derives from `PricingException` and is marked `sealed`.
- [ ] Both classes have the three standard constructors (parameterless, message, message + inner).
- [ ] Nowhere in the code is there derivation from `ApplicationException`.
- [ ] The comment on `PricingException` explicitly states why not `ApplicationException` (RU+EN).
- [ ] `CalculateTotal` throws `ArgumentNullException` for `order == null` via `ArgumentNullException.ThrowIfNull`.
- [ ] `CalculateTotal` throws `InvalidOrderLineException` for an empty order and for `Quantity <= 0` / `UnitPrice < 0`.
- [ ] `CalculateTotal` returns `Result<decimal>.Fail("COUPON_EXPIRED")` for an expired coupon without throwing.
- [ ] Loyalty `Bronze` / `Silver` / `Gold` gives a 5% / 10% / 15% discount; `None` — 0%.
- [ ] In `Program.cs` there is no empty `catch (Exception) { }`; everywhere there is either logging or `throw;`.
- [ ] The micro-benchmark honestly measures both branches and prints the time in milliseconds; exceptions are significantly slower.
- [ ] A git commit was made with the message `M07-L01: exceptions vs return codes`.

#### Hints (without giving away the answer)
- Recall the `OrderNotFoundException` example from the lesson: the same structure of three constructors, `sealed`, derivation from `Exception`.
- For `Result<T>.Ok` return `new Result<T>(value, null)`; for `Fail` — `new Result<T>(default, error)`.
- `DateTimeOffset.UtcNow` is for checking the coupon's expiry; compare it with `coupon.ExpiresAt`.
- For the benchmark use `System.Diagnostics.Stopwatch`, `var sw = Stopwatch.StartNew();`. A loop of 1,000,000 iterations is enough to see the difference.
- Remember: `throw;` preserves the stack, `throw ex;` does not.

#### Reference solution walk-through
```csharp
// C# 12 / .NET 8 — Reference solution for homework M07-L01
// Exceptions vs return codes + custom exception hierarchy.

using System;
using System.Collections.Generic;
using System.Diagnostics;

// Result<T> — a result type for expected business outcomes.
// Does not throw, immutable.
public readonly record struct Result<T>(T Value, string? Error)
{
    public bool IsSuccess => Error is null;

    public static Result<T> Ok(T value) => new(value, null);
    public static Result<T> Fail(string error) => new(default, error);
}

// Base domain exception for the pricing module.
// IMPORTANT: derive directly from Exception, NOT from ApplicationException.
// ApplicationException is deprecated and carries no semantics — Microsoft forbids it.
public class PricingException : Exception
{
    public PricingException() { }
    public PricingException(string message) : base(message) { }
    public PricingException(string message, Exception inner) : base(message, inner) { }
}

// Accident: invalid order line. sealed — no further derivation.
public sealed class InvalidOrderLineException : PricingException
{
    public InvalidOrderLineException() { }
    public InvalidOrderLineException(string message) : base(message) { }
    public InvalidOrderLineException(string message, Exception inner) : base(message, inner) { }
}

// Domain types.
public enum LoyaltyTier { None, Bronze, Silver, Gold }

public sealed class OrderLine
{
    public string Sku { get; init; } = string.Empty;
    public int Quantity { get; init; }
    public decimal UnitPrice { get; init; }
}

public sealed class Order
{
    public Guid Id { get; init; }
    public IReadOnlyList<OrderLine> Lines { get; init; } = Array.Empty<OrderLine>();
}

public sealed class Coupon
{
    public string Code { get; init; } = string.Empty;
    public decimal DiscountPercent { get; init; }
    public DateTimeOffset ExpiresAt { get; init; }
}

public static class PricingService
{
    public static Result<decimal> CalculateTotal(Order order, Coupon? coupon, LoyaltyTier tier)
    {
        // Contract violation — accident, fail fast via exception.
        ArgumentNullException.ThrowIfNull(order);

        if (order.Lines.Count == 0)
            throw new InvalidOrderLineException(
                "Order has no lines");

        decimal subtotal = 0m;
        foreach (var line in order.Lines)
        {
            if (line.Quantity <= 0)
                throw new InvalidOrderLineException(
                    $"Quantity <= 0 for SKU {line.Sku}");
            if (line.UnitPrice < 0)
                throw new InvalidOrderLineException(
                    $"UnitPrice < 0 for SKU {line.Sku}");
            subtotal += line.Quantity * line.UnitPrice;
        }

        // Expected business outcome — expired coupon.
        // Do NOT throw: the caller expects to branch.
        if (coupon is not null && coupon.ExpiresAt < DateTimeOffset.UtcNow)
            return Result<decimal>.Fail("COUPON_EXPIRED");

        decimal discount = tier switch
        {
            LoyaltyTier.Bronze => 0.05m,
            LoyaltyTier.Silver => 0.10m,
            LoyaltyTier.Gold   => 0.15m,
            _                  => 0.00m,
        };

        decimal couponDiscount = coupon is null ? 0m : coupon.DiscountPercent;
        decimal totalDiscount = discount + couponDiscount;
        decimal total = subtotal * (1m - totalDiscount);

        return Result<decimal>.Ok(Math.Round(total, 2));
    }
}

public static class Program
{
    public static void Main()
    {
        var order = new Order
        {
            Id = Guid.NewGuid(),
            Lines = new List<OrderLine>
            {
                new() { Sku = "CAT-FOOD-1", Quantity = 2, UnitPrice = 499.00m },
                new() { Sku = "TOY-MOUSE",  Quantity = 1, UnitPrice = 199.00m },
            }
        };

        // (a) Valid order + coupon + loyalty.
        var validCoupon = new Coupon
        {
            Code = "WELCOME10",
            DiscountPercent = 0.10m,
            ExpiresAt = DateTimeOffset.UtcNow.AddHours(1),
        };
        var ok = PricingService.CalculateTotal(order, validCoupon, LoyaltyTier.Gold);
        Console.WriteLine(ok.IsSuccess
            ? $"Total: {ok.Value}"
            : $"Error: {ok.Error}");

        // (b) Expired coupon — no try/catch, via Result.
        var expiredCoupon = new Coupon
        {
            Code = "OLD",
            DiscountPercent = 0.10m,
            ExpiresAt = DateTimeOffset.UtcNow.AddHours(-1),
        };
        var expired = PricingService.CalculateTotal(order, expiredCoupon, LoyaltyTier.Bronze);
        Console.WriteLine(expired.IsSuccess
            ? $"Total: {expired.Value}"
            : $"Error: {expired.Error}");

        // (c) Accident — invalid line, catch the concrete type.
        var badOrder = new Order
        {
            Id = Guid.NewGuid(),
            Lines = new List<OrderLine> { new() { Sku = "BAD", Quantity = 0, UnitPrice = 10m } },
        };
        try
        {
            _ = PricingService.CalculateTotal(badOrder, null, LoyaltyTier.None);
        }
        catch (InvalidOrderLineException ex)
        {
            Console.WriteLine($"Caught: {ex.Message}");
        }

        // Micro-benchmark: the cost of exceptions.
        BenchException.Run();
    }
}

internal static class BenchException
{
    public static void Run()
    {
        const int N = 1_000_000;

        var swReturn = Stopwatch.StartNew();
        for (int i = 0; i < N; i++)
        {
            var code = ReturnCode(i);
            if (code == -1) { /* simulate check */ }
        }
        swReturn.Stop();

        var swThrow = Stopwatch.StartNew();
        for (int i = 0; i < N; i++)
        {
            try
            {
                ThrowIfNegative(i - 1);
            }
            catch (InvalidOperationException) { /* swallow in benchmark only */ }
        }
        swThrow.Stop();

        Console.WriteLine($"Return: {swReturn.ElapsedMilliseconds} ms, " +
                          $"Throw: {swThrow.ElapsedMilliseconds} ms");
    }

    private static int ReturnCode(int x) => x < 0 ? -1 : x;
    private static void ThrowIfNegative(int x)
    {
        if (x < 0) throw new InvalidOperationException("negative");
    }
}
```

Walk-through, line by line. `Result<T>` is built as a `readonly record struct` exactly following the lesson's pattern — this gives value semantics, immutability, and auto-generated `Equals`/`GetHashCode`. The `Ok` and `Fail` methods are convenience factories that make the caller's code readable (`Result<decimal>.Ok(total)` instead of `new Result<decimal>(total, null)`). `IsSuccess => Error is null` means a null `Error` indicates success; the `is null` pattern is correct even for value types.

`PricingException` derives directly from `Exception` (the `: Exception` clause), not from `ApplicationException`. The comment explicitly records why: `ApplicationException` is deprecated, Microsoft discourages it, and it carries no semantics. This applies rule (2) from the lesson. The three constructors follow the standard .NET idiom: parameterless (for serialization and edge cases), message-only, and message + inner. The third constructor is critical — it lets you pass the original cause (for example an `IOException`) and not lose it when logging at the top level.

`InvalidOrderLineException` is marked `sealed` and derives from `PricingException`. This gives a two-level hierarchy: you can catch either the specific `InvalidOrderLineException` or "everything pricing-related" via `catch (PricingException)`. The `sealed` keyword prevents further uncontrolled derivation and slightly speeds up virtual dispatch. The constructors duplicate the three forms — this is mandatory for correct serialization and a common style.

In `CalculateTotal` the key point is the separation of accidents from expected outcomes. `ArgumentNullException.ThrowIfNull(order)` is the .NET 7+ validator, shorter and faster than a manual check; it throws `ArgumentNullException` with the parameter name automatically. An empty `order.Lines` and invalid `Quantity`/`UnitPrice` are contract violations (the caller passed invalid data), so here we `throw new InvalidOrderLineException(...)`. This is rule (1) and the "fail fast" pattern from the lesson: validation at the start of the method, with a concrete exception type.

The expired coupon, however, is a `return Result<decimal>.Fail("COUPON_EXPIRED")`, not a throw. Why? Because "coupon expired" is an expected business scenario: marketing can run campaigns with expiring coupons, and the caller must be able to show the user "your coupon is no longer valid". This is rule (3) — do not use exceptions for normal control flow. The line `if (coupon is not null && coupon.ExpiresAt < DateTimeOffset.UtcNow)` checks the expiry, and `return` hands back the result without unwinding the stack. Loyalty and coupon discounts are summed via a `switch` expression — modern C# 12.

In `Program.Main` the three scenarios demonstrate the difference. Scenario (a) is a success — we read `ok.Value`. Scenario (b) is an expected failure — we read `expired.Error`, with no `try/catch`. Scenario (c) is an accident — we catch specifically `InvalidOrderLineException` (not the base `Exception`), which lets us write precise handlers. In the `BenchException.Run` benchmark you can see that `throw` in a loop of 1,000,000 iterations is tens to hundreds of times slower than `return` — a practical confirmation of the lesson's point about the cost of exceptions. Note: the empty `catch (InvalidOperationException) { }` in the benchmark is an exception to the "never swallow" rule, justified because the goal is to measure the cost of throw/catch, not to handle an error; production code must not do this.

#### Going deeper (bonus)
1. Add a third custom type `CatalogUnavailableException : PricingException` with a constructor that accepts `IOException inner`. Model reading the price list from a file inside `CalculateTotal`, and on `IOException` wrap it as `CatalogUnavailableException("Failed to load price list", ioEx)`. Catch it at the top level and print `ex.InnerException?.Message`.
2. Implement an overload of `Result<T>` with an enum tag: `enum ResultKind { Ok, Fail }` instead of checking `Error is null`. Compare the readability of both approaches.
3. Extend the benchmark: measure the cost of `throw` at different stack depths (1, 5, 10 frames). Draw a conclusion about how depth affects cost.
4. Write a method `ParseSku(string input)` that, for an invalid format, returns `Result<string>.Fail("BAD_FORMAT")`, and for `null` throws `ArgumentNullException`. Explain in a comment why these are different categories.

---

#### Чек-лист сдачи / Submission checklist
- [ ] (RU) Проект `CatMarket.Pricing` создан и собирается без предупреждений.
- [ ] (RU) Файлы `Result.cs`, `DomainExceptions.cs`, `PricingService.cs`, `BenchException.cs`, `Program.cs` на месте.
- [ ] (RU) `dotnet run` выводит три строки в порядке `Total / Error / Caught`.
- [ ] (RU) Никакого наследования от `ApplicationException`.
- [ ] (RU) Git-коммит `M07-L01: exceptions vs return codes` сделан.
- [ ] (EN) The `CatMarket.Pricing` project is created and builds without warnings.
- [ ] (EN) The files `Result.cs`, `DomainExceptions.cs`, `PricingService.cs`, `BenchException.cs`, `Program.cs` are in place.
- [ ] (EN) `dotnet run` prints three lines in the order `Total / Error / Caught`.
- [ ] (EN) No derivation from `ApplicationException` anywhere.
- [ ] (EN) A git commit `M07-L01: exceptions vs return codes` was made.

#### Ресурсы / Resources
- [Microsoft Learn — Обработка и создание исключений / Handling and throwing exceptions — https://learn.microsoft.com/dotnet/standard/exceptions/](https://learn.microsoft.com/dotnet/standard/exceptions/)
- [Microsoft Learn — Best practices for exceptions — https://learn.microsoft.com/dotnet/standard/exceptions/best-practices-for-exceptions](https://learn.microsoft.com/dotnet/standard/exceptions/best-practices-for-exceptions)
- [.NET design guidelines — Exception throwing — https://learn.microsoft.com/dotnet/standard/design-guidelines/exceptions](https://learn.microsoft.com/dotnet/standard/design-guidelines/exceptions)
- [Microsoft Learn — Exception hierarchy and properties — https://learn.microsoft.com/dotnet/standard/exceptions/exception-class-and-properties](https://learn.microsoft.com/dotnet/standard/exceptions/exception-class-and-properties)
