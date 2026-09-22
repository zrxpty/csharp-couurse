---
[← К уроку M07-L07](lesson-M07-L07-result-pattern.md) | [⬆ К модулю M07](../README.md) | [Следующее ДЗ →]()
---

### Домашнее задание M07-L07: Result<T>/OneOf (Optional), когда не использовать исключения / Homework M07-L07: Result<T>/OneOf (Optional), when not to use exceptions

**Урок / Lesson:** M07-L07
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 4/5
**Цель / Goal:** (RU) Научиться заменять исключения на паттерн `Result<T, TError>` и тип `OneOf` для ожидаемых бизнес-исходов, освоить монадическую композицию через `Bind`/`Map`, исчерпывающую обработку через `switch`/`Match`, реализовать аллокационно-свободный `struct Result` для горячего пути и провести честное сравнение производительности с исключением. (EN) Learn to replace exceptions with the `Result<T, TError>` pattern and the `OneOf` type for expected business outcomes, master monadic composition via `Bind`/`Map`, exhaustive handling via `switch`/`Match`, implement an allocation-free `struct Result` for a hot path, and run an honest performance comparison against exceptions.

#### Связь с уроком / Connection to the lesson

(RU) Урок доказывает, что исключения — для аварий, а `Result<T>` / `OneOf` — для ожидаемых исходов: отказ выдать деньги в банкомате не авария, а нормальный результат. В этом ДЗ вы построите мини-сервис обработки заказов, в котором «не найдено», «невалидный ввод» и «нарушение бизнес-правила» возвращаются как данные через `Result` и `OneOf`, а исключение остаётся только для genuinely broken-ситуаций. Вы повторите все примеры урока: record с первичным конструктором, `Bind`-композицию, `switch`-паттерны, `OneOf.Match` и `readonly struct` для горячего пути.

(EN) The lesson argues that exceptions are for crashes while `Result<T>` / `OneOf` are for expected outcomes: an ATM refusing to dispense cash is not a crash but a normal result. In this homework you will build a mini order-processing service where "not found", "invalid input", and "business rule violation" come back as data through `Result` and `OneOf`, and an exception remains only for genuinely broken situations. You will reproduce every example from the lesson: a record with a primary constructor, `Bind` composition, `switch` patterns, `OneOf.Match`, and a `readonly struct` for the hot path.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация

Вы присоединяетесь к команде платформы электронной коммерции, которая страдает от классической болезни: бизнес-логика повсюду бросает исключения. Метод `PlaceOrder` кидает `InvalidOrderException`, `OutOfStockException`, `PaymentDeclinedException`, а слой контроллеров оборачивает всё в `try/catch`, теряя типизацию ошибок и платя 1–3 микросекунды и килобайты аллокаций на каждый «неуспех». Профайлер показывает, что в час пик 30 % времени процессора уходит на раскрутку стека исключений для совершенно штатных ситуаций: пустая корзина, недостаток товара на складе, отклонённая карта. Команда решила провести рефакторинг по модели урока M07-L07: ожидаемые исходы вернуть как данные через `Result<T, TError>` и `OneOf`, а исключения оставить только для genuinely broken-ситуаций — разрыва соединения с платёжным шлюзом, нарушения инварианта программы.

Ваша задача — построить ядро нового сервиса заказов с нуля, продемонстрировав честный контракт методов: из сигнатуры должно быть видно, что операция может завершиться неудачно, и какие именно виды неудачи возможны. Вы реализуете два варианта `Result`: ссылочный `record` для обычных путей и `readonly struct` для горячего пути парсинга миллионов строк чеков. Вы примените `Bind` для линейной композиции шагов «валидация → резерв → оплата», `OneOf` для операции поиска заказа с тремя исходами, и `switch`-паттерны C# 12 для исчерпывающей обработки. В финале вы проведёте бенчмарк, доказывающий ускорение в 10–100 раз на горячем пути.

#### Что нужно сделать (пошагово)

1. Создайте решение и проект консольного приложения с .NET 8:
   ```
   dotnet new console -n M07L07.Homework -o M07L07.Homework --framework net8.0
   cd M07L07.Homework
   dotnet add package OneOf --version 3.1.0
   dotnet add package BenchmarkDotNet --version 0.13.12
   ```
   Включите `#nullable enable` и установите `LangVersion` в `latest` в `.csproj`.

2. В файле `Result.cs` реализуйте ссылочный вариант на `record` с первичным конструктором C# 12:
   ```csharp
   public record Result<T, TError>(bool IsSuccess, T? Value, TError? Error)
   ```
   Добавьте фабрики `Ok(T value)` и `Fail(TError error)`, а также метод `Bind<TNext>(Func<T, Result<TNext, TError>> next)` для монадической композиции. Повторите поведение из урока: на `Fail` цепочка коротко замыкается и возвращает ту же ошибку.

3. В файле `Errors.cs` опишите доменные ошибки как типизированный `enum` или иерархию `record`-ов: `InvalidInput`, `NotFound`, `OutOfStock`, `PaymentDeclined`, `RuleViolation`. Дайте каждой осмысленные `Code` и `Message`. Не используйте `string` как тип ошибки — это требование best practices урока.

4. В файле `StructResult.cs` реализуйте второй вариант — `readonly struct StructResult<T, TError>` без аллокаций для горячего пути. Должны быть `Ok`/`Fail`/`Bind` с тем же контрактом. Этот вариант нужен для парсинга миллионов строк чеков, где каждая аллокация стоит дорого.

5. В `OrderService.cs` постройте домен: классы `Order`, `Customer`, `Product` с неизменяемыми полями. Метод `PlaceOrder(Customer c, Product p, int qty)` возвращает `Result<Order, OrderError>`. Внутри он последовательно через `Bind` прогоняет: валидацию количества (`qty > 0`), проверку наличия на складе, списание со счёта клиента. Каждая операция возвращает `Result`. Итог — линейный код без вложенных `if`.

6. В `OrderLookup.cs` реализуйте метод `FindOrder(int id)`, возвращающий `OneOf<Order, NotFound, InvalidInput>`, как в примере урока с `FindUser`. Используйте `Match` для исчерпывающей обработки в вызывающем коде.

7. В `ReceiptParser.cs` реализуйте горячий путь: парсинг строки чека формата `"SKU;Qty;Price"` через `StructResult`. Здесь намеренно НЕ используйте `int.Parse`, который бросает `FormatException` — напишите безопасный парсер `TryParseInt`-стиля, возвращающий `StructResult<int, ParseError>`. Обработайте 100 000 строк в цикле и убедитесь, что ни одного исключения не возникло.

8. В `Program.cs` (top-level statements) продемонстрируйте все кейсы: успешный заказ, заказ с `OutOfStock`, поиск заказа через `OneOf`, парсинг чеков. Вывод должен содержать русскоязычные сообщения с кодами ошибок, например: `Ошибка [OUT_OF_STOCK]: Недостаточно товара на складе`.

9. В `Benchmarks.cs` с помощью BenchmarkDotNet сравните три подхода к парсингу невалидной строки: `int.Parse` в `try/catch`, `int.TryParse` и ваш `StructResult`-парсер. Ожидаемый результат — `StructResult` быстрее `try/catch` в десятки раз и не аллоцирует память. Запустите бенчмарк:
   ```
   dotnet run -c Release --project M07L07.Homework -- --filter *ParseBenchmark*
   ```

10. Покажите, где исключение всё-таки уместно: добавьте метод `ChargeCard` для genuinely broken-ситуации (разрыв соединения с шлюзом), который кидает `PaymentGatewayUnavailableException`. Прокомментируйте в коде, почему здесь исключение, а не `Result`.

#### Требования к решению

Решение компилируется под .NET 8 с `LangVersion=latest` и `#nullable enable`. Используется C# 12: первичные конструкторы, collection expressions, pattern matching, raw string literals где уместно. Все ожидаемые ошибки (`InvalidInput`, `NotFound`, `OutOfStock`, `PaymentDeclined`, `RuleViolation`, `ParseError`) возвращаются через `Result` или `OneOf` — ни одного `throw new ...Exception` в бизнес-логике, кроме явно обоснованного `PaymentGatewayUnavailableException`. Тип ошибки явный (`OrderError`, `ParseError`), а не `string` и не `Exception`. Композиция шагов в `PlaceOrder` идёт через `Bind`, без вложенных `if (result.IsSuccess)`. Обработка результата — через `switch` или `Match`, исчерпывающая. Горячий путь `ReceiptParser` использует `readonly struct StructResult`, а не ссылочный `record`, и не аллоцирует. Бенчмарк запускается в Release-конфигурации и демонстрирует измеримое преимущество `StructResult` над `try/catch`. Контракт каждого метода честен: из сигнатуры видно, что операция может завершиться неудачно и какими способами. Код сопровождается русскими и английскими комментариями в ключевых местах. Имена файлов, классов и методов совпадают с описанием выше. Вывод программы воспроизводим и понятен.

#### Тонкости и подводные камни

Главная ошибка, которую повторяют новички — путают «авария» и «не получилось». Отказ оплаты из-за недостатка баланса — это нормальный бизнес-исход, его нужно вернуть как `Result.Fail(PaymentDeclined)`, а не кидать `PaymentDeclinedException`. Исключение уместно только когда что-то genuinely broken: разрыв соединения с платёжным шлюзом, нарушение инварианта программы, `OutOfMemoryException`. Чётко разделите эти два мира в комментарии.

Вторая ошибка — использование `string` как типа ошибки. Кажется удобным `Result<T, string>`, но потребитель не сможет исчерпывающе обработать ветки в `switch`, а опечатки в кодах не ловятся компилятором. Заведите типизированный `OrderError` (enum или иерархия record-ов), как требует урок. Третья ошибка — проглатывание ошибки: `if (!result.IsSuccess) return;` без передачи `Error` вверх. Либо обрабатывайте, либо пробрасывайте через `Bind`/`Match`. Четвёртая — глубокая вложенность `if (result.IsSuccess)`, которую урок явно называет антипаттерном: применяйте `Bind`, чтобы код оставался линейным.

Пятая тонкость — `null` вместо явного результата. Не возвращайте `null` из `FindOrder` — используйте `OneOf<Order, NotFound, InvalidInput>`, и контракт заговорит. Шестая — аллокации в горячем пути: ссылочный `record Result<T, TError>` аллоцирует на каждый вызов, что в парсинге миллионов строк убивает производительность. Перейдите на `readonly struct StructResult` — он не аллоцирует и укладывается в наносекунды, как утверждает урок. Седьмая — забытый `#nullable enable`: без него `T? Value` не защищает от `null`, и `result.Value!` становится скрытой бомбой. Включите nullable и обращайтесь с `Value!` осознанно. Восьмая — неполный `switch` без `_`: если вы перечислили не все варианты `OneOf`, компилятор OneOf не даст скомпилировать `Match` без всех веток, но в `switch` по record-ам нужен `_` или `nameof`-дефолт. Девятая — смешивание `OneOf` и `Result` в одном методе без причины: выберите один инструмент для операции. `Result` — для бинаправленного «успех/неудача с типизированной ошибкой», `OneOf` — когда несколько принципиально разных успешных исходов или несколько видов ошибок с разной структурой.

#### Критерии приёмки

- [ ] Проект `M07L07.Homework` собирается под .NET 8 с `#nullable enable` и C# 12.
- [ ] `Result<T, TError>` реализован как `record` с первичным конструктором, `Ok`/`Fail`/`Bind`.
- [ ] `Bind` корректно коротко замыкает на `Fail` и возвращает ту же ошибку.
- [ ] Тип ошибки типизированный (`OrderError`/`ParseError`), а не `string` или `Exception`.
- [ ] `PlaceOrder` использует `Bind` для композиции без вложенных `if`.
- [ ] `FindOrder` возвращает `OneOf<Order, NotFound, InvalidInput>`.
- [ ] Вызывающий код обрабатывает `OneOf` через `Match` исчерпывающе.
- [ ] `StructResult` реализован как `readonly struct` без аллокаций.
- [ ] `ReceiptParser` парсит 100 000 строк без единого исключения.
- [ ] Бенчмарк BenchmarkDotNet показывает `StructResult` быстрее `try/catch`.
- [ ] В бизнес-логике нет `throw new ...Exception`, кроме обоснованного `PaymentGatewayUnavailableException`.
- [ ] `PaymentGatewayUnavailableException` сопровождён комментарием, почему здесь исключение.
- [ ] Вывод `Program.cs` содержит коды ошибок `[OUT_OF_STOCK]` и т. п.
- [ ] Никакой ошибки не проглатывается: каждая либо обработана, либо пробрасывается.
- [ ] `switch` по `Result` исчерпывающий, без забытого `_` без необходимости.

#### Подсказки (без прямого ответа)

- Подумайте, какой тип выгоднее для ошибки: enum (быстрый, но без полей) или record-иерархия (с полями вроде `MissingQty`). Вспомните, что урок рекомендует «осмысленные `Code` и `Message`».
- Для `Bind` вспомните сигнатуру из урока: `IsSuccess ? next(Value!) : Result<TNext>.Fail(Error!)`. Обобщите её на `TError`.
- В `PlaceOrder` начните с `ValidateQty(...)` и `.Bind(_ => CheckStock(...)).Bind(_ => Charge(...))` — цепочка шагов.
- Для `OneOf.Match` вспомните пример `FindUser` из урока: три лямбды подряд.
- В `StructResult` будьте осторожны с `default!` для `Value` — он остаётся неинициализированным на `Fail`.
- Для бенчмарка используйте `[MemoryDiagnoser]`, чтобы увидеть аллокации `try/catch`.
- В `ChargeCard` задайте себе вопрос: «Если шлюз упал, могу ли я продолжить?» Если нет — это genuinely broken.

#### Эталонное решение (разбор)

```csharp
// C# 12 / .NET 8 — эталонное решение ДЗ M07-L07
// Reference solution for homework M07-L07

using OneOf;

namespace M07L07.Homework;

// 1) Типизированная доменная ошибка заказа / Typed domain order error
public record OrderError(string Code, string Message);
public static class OrderErrors
{
    public static OrderError InvalidInput(string msg) => new("INVALID_INPUT", msg);
    public static OrderError OutOfStock(int requested, int available) =>
        new("OUT_OF_STOCK", $"Запрошено {requested}, доступно {available} / Requested {requested}, available {available}");
    public static OrderError PaymentDeclined(string reason) => new("PAYMENT_DECLINED", reason);
    public static OrderError RuleViolation(string msg) => new("RULE_VIOLATION", msg);
    public static OrderError NotFound(int id) => new("NOT_FOUND", $"Заказ {id} не найден / Order {id} not found");
}

// 2) Ссылочный Result на record с первичным конструктором / Reference Result on record
public record Result<T, TError>(bool IsSuccess, T? Value, TError? Error)
{
    public static Result<T, TError> Ok(T value) => new(true, value, default);
    public static Result<T, TError> Fail(TError error) => new(false, default, error);

    // Монадическая композиция: короткое замыкание на ошибке / Monadic composition: short-circuit on error
    public Result<TNext, TError> Bind<TNext>(Func<T, Result<TNext, TError>> next) =>
        IsSuccess ? next(Value!) : Result<TNext, TError>.Fail(Error!);
}

// 3) Аллокационно-свободный struct Result для горячего пути / Allocation-free struct Result for hot path
public readonly struct StructResult<T, TError>
{
    public bool IsSuccess { get; }
    public T? Value { get; }
    public TError? Error { get; }

    private StructResult(bool ok, T? value, TError? error) => (IsSuccess, Value, Error) = (ok, value, error);
    public static StructResult<T, TError> Ok(T value) => new(true, value, default);
    public static StructResult<T, TError> Fail(TError error) => new(false, default, error);

    public StructResult<TNext, TError> Bind<TNext>(Func<T, StructResult<TNext, TError>> next) =>
        IsSuccess ? next(Value!) : StructResult<TNext, TError>.Fail(Error!);
}

// 4) Доменные сущности / Domain entities
public record Product(string Sku, decimal Price, int Stock);
public record Customer(int Id, string Name, decimal Balance);
public record Order(int Id, Customer Customer, Product Product, int Qty, decimal Total);

// 5) Сервис заказов: композиция через Bind / Order service: composition via Bind
public class OrderService
{
    private int _nextId = 1;

    public Result<Order, OrderError> PlaceOrder(Customer c, Product p, int qty) =>
        ValidateQty(qty)
            .Bind(_ => CheckStock(p, qty))
            .Bind(_ => Charge(c, p, qty))
            .Bind(_ => Result<Order, OrderError>.Ok(new Order(_nextId++, c, p, qty, p.Price * qty)));

    private static Result<Unit, OrderError> ValidateQty(int qty) =>
        qty > 0
            ? Result<Unit, OrderError>.Ok(default)
            : Result<Unit, OrderError>.Fail(OrderErrors.InvalidInput("Количество должно быть > 0 / Qty must be > 0"));

    private static Result<Unit, OrderError> CheckStock(Product p, int qty) =>
        qty <= p.Stock
            ? Result<Unit, OrderError>.Ok(default)
            : Result<Unit, OrderError>.Fail(OrderErrors.OutOfStock(qty, p.Stock));

    private static Result<Unit, OrderError> Charge(Customer c, Product p, int qty)
    {
        var total = p.Price * qty;
        return c.Balance >= total
            ? Result<Unit, OrderError>.Ok(default)
            : Result<Unit, OrderError>.Fail(OrderErrors.PaymentDeclined("Недостаточно средств / Insufficient funds"));
    }

    // Genuinely broken — здесь ИСКЛЮЧЕНИЕ, а не Result / Genuinely broken — here an EXCEPTION, not Result
    public void ChargeCard(Customer c, decimal amount, IGateway gateway)
    {
        // Разрыв соединения — авария, продолжать нельзя / Connection loss is a crash, cannot continue
        if (!gateway.IsAvailable)
            throw new PaymentGatewayUnavailableException("Шлюз недоступен / Gateway unavailable");
        // ... списание ...
    }
}

public readonly record struct Unit;
public class PaymentGatewayUnavailableException : Exception
{
    public PaymentGatewayUnavailableException(string msg) : base(msg) { }
}
public interface IGateway { bool IsAvailable { get; } }

// 6) OneOf для поиска заказа / OneOf for order lookup
public class NotFound;
public class InvalidInput;
public class OrderLookup
{
    private readonly Dictionary<int, Order> _orders;
    public OrderLookup(Dictionary<int, Order> orders) => _orders = orders;

    public OneOf<Order, NotFound, InvalidInput> FindOrder(int id) =>
        id switch
        {
            <= 0 => new InvalidInput(),
            _ when _orders.TryGetValue(id, out var o) => o,
            _ => new NotFound()
        };

    public string Describe(int id) =>
        FindOrder(id).Match(
            o => $"Найден / Found: Order #{o.Id} ({o.Qty} x {o.Product.Sku})",
            _ => $"Ошибка [NOT_FOUND]: Заказ {id} не найден / Order {id} not found",
            _ => $"Ошибка [INVALID_INPUT]: Id должен быть > 0 / Id must be > 0"
        );
}

// 7) Безопасный парсер для горячего пути / Safe parser for hot path
public enum ParseError { Empty, NotANumber, Overflow }

public static class ReceiptParser
{
    public static StructResult<int, ParseError> TryParseInt(string s)
    {
        if (string.IsNullOrEmpty(s)) return StructResult<int, ParseError>.Fail(ParseError.Empty);
        if (!int.TryParse(s, out var v)) return StructResult<int, ParseError>.Fail(ParseError.NotANumber);
        return StructResult<int, ParseError>.Ok(v);
    }

    public static StructResult<(string Sku, int Qty, decimal Price), ParseError> ParseLine(string line)
    {
        var parts = line.Split(';');
        if (parts.Length != 3) return StructResult<(string, int, decimal), ParseError>.Fail(ParseError.Empty);
        var qty = TryParseInt(parts[1]);
        if (!qty.IsSuccess) return StructResult<(string, int, decimal), ParseError>.Fail(qty.Error);
        var price = TryParseInt(parts[2]);
        if (!price.IsSuccess) return StructResult<(string, int, decimal), ParseError>.Fail(price.Error);
        return StructResult<(string, int, decimal), ParseError>.Ok((parts[0], qty.Value, price.Value));
    }
}
```

Разбор по строкам. Тип `OrderError` — типизированный `record` с `Code` и `Message`, а не `string`: урок требует «осмысленные `Code` и `Message`, иначе потребитель не сможет реагировать». Фабрики `InvalidInput`, `OutOfStock` и т. д. дают осмысленные сообщения. `Result<T, TError>` — record с первичным конструктором C# 12 (`public record Result<T, TError>(bool IsSuccess, T? Value, TError? Error)`), как в примере урока, плюс обобщение `TError` и `Bind` для монадической композиции. Строка `IsSuccess ? next(Value!) : Result<TNext, TError>.Fail(Error!)` в точности повторяет короткое замыкание из урока: на `Fail` цепочка не идёт дальше, а возвращает ту же ошибку. `StructResult` — `readonly struct`, чтобы в горячем пути `ReceiptParser` не было аллокаций: урок подчёркивает «в горячих путях реализуйте `Result` как `readonly struct`». `PlaceOrder` строится как `ValidateQty(...).Bind(_ => CheckStock(...)).Bind(_ => Charge(...)).Bind(_ => Ok(new Order(...)))` — это линейный код без вложенных `if`, как требует best practice урока. Каждая операция возвращает `Result<Unit, OrderError>`, где `Unit` — маркер «значения нет, но успех». `ChargeCard` намеренно кидает `PaymentGatewayUnavailableException`, потому что разрыв шлюза — это genuinely broken по уроку: «разрыв соединения с критическим сервисом». Это единственное место с `throw` в бизнес-логике, и комментарий объясняет почему. `OrderLookup.FindOrder` возвращает `OneOf<Order, NotFound, InvalidInput>` в духе примера `FindUser` из урока, а `Describe` обрабатывает все три ветки через `Match`, что заставляет исчерпывающе покрыть исходы. `ReceiptParser.TryParseInt` и `ParseLine` используют `StructResult`, не бросают `FormatException` и не аллоцируют — это демонстрация производственного преимущества паттерна на горячем пути. `int.TryParse` внутри — легальный мост к BCL, но наш публичный контракт остаётся `StructResult`, а не `bool`. Таким образом эталон покрывает все концепции урока: record + primary constructor, `Bind`, `switch`/`Match`, `OneOf`, `readonly struct`, типизированная ошибка, граница между исключением и результатом.

#### Задания на углубление (бонус)

1. Реализуйте `Map` (он же `Select`) рядом с `Bind` и перепишите `PlaceOrder` через комбинацию `Map`+`Bind`, чтобы вернуть `Order` только на финальном шаге. Сравните читаемость.
2. Добавьте накопление нескольких ошибок валидации (не короткое замыкание): тип `Validation<TError>` как список ошибок, как в F# Validation. Покажите, что в `PlaceOrder` можно вернуть сразу все нарушения.
3. Реализуйте `Result` через discriminated union на `OneOf<T, TError>` и сравните эргономику с собственным record: какой подход заставляет полнее обрабатывать ветки?
4. Подключите Source Generator или анализатор Roslyn, который запрещает `throw new *Exception` в методах, помеченных `[ResultOnly]`, и заставляет использовать `Result`.

---

## Statement in English / Постановка на английском

#### Context & motivation

You are joining an e-commerce platform team that suffers from a classic disease: business logic throws exceptions everywhere. The `PlaceOrder` method throws `InvalidOrderException`, `OutOfStockException`, `PaymentDeclinedException`, while the controller layer wraps everything in `try/catch`, losing the typing of errors and paying 1–3 microseconds plus kilobytes of allocations for every "non-success." The profiler shows that, at peak hour, 30 % of CPU time is spent unwinding exception stacks for completely ordinary situations: an empty cart, insufficient stock, a declined card. The team has decided to refactor following the model of lesson M07-L07: expected outcomes will come back as data through `Result<T, TError>` and `OneOf`, while exceptions will remain only for genuinely broken situations — a severed connection to the payment gateway, a violated program invariant.

Your task is to build the core of the new order service from scratch, demonstrating an honest method contract: the signature should reveal that the operation can fail and which kinds of failure are possible. You will implement two `Result` variants — a reference `record` for ordinary paths and a `readonly struct` for the hot path of parsing millions of receipt lines. You will apply `Bind` for the linear composition of the "validate → reserve → pay" steps, `OneOf` for an order lookup with three outcomes, and C# 12 `switch` patterns for exhaustive handling. In the finale you will run a benchmark proving a 10–100× speedup on the hot path.

#### What to do step by step

1. Create a solution and a console project targeting .NET 8:
   ```
   dotnet new console -n M07L07.Homework -o M07L07.Homework --framework net8.0
   cd M07L07.Homework
   dotnet add package OneOf --version 3.1.0
   dotnet add package BenchmarkDotNet --version 0.13.12
   ```
   Enable `#nullable enable` and set `LangVersion` to `latest` in the `.csproj`.

2. In `Result.cs` implement the reference variant on a `record` with a C# 12 primary constructor:
   ```csharp
   public record Result<T, TError>(bool IsSuccess, T? Value, TError? Error)
   ```
   Add `Ok(T value)` and `Fail(TError error)` factories, plus a `Bind<TNext>(Func<T, Result<TNext, TError>> next)` method for monadic composition. Reproduce the lesson's behavior: on `Fail` the chain short-circuits and returns the same error.

3. In `Errors.cs` describe domain errors as a typed `enum` or a hierarchy of `record`s: `InvalidInput`, `NotFound`, `OutOfStock`, `PaymentDeclined`, `RuleViolation`. Give each meaningful `Code` and `Message`. Do not use `string` as the error type — that is a lesson best practice.

4. In `StructResult.cs` implement the second variant — a `readonly struct StructResult<T, TError>` with zero allocations for the hot path. It must expose `Ok`/`Fail`/`Bind` with the same contract. This variant is needed for parsing millions of receipt lines, where every allocation is costly.

5. In `OrderService.cs` build the domain: classes `Order`, `Customer`, `Product` with immutable fields. The method `PlaceOrder(Customer c, Product p, int qty)` returns `Result<Order, OrderError>`. Inside, it sequentially pipes through `Bind`: quantity validation (`qty > 0`), stock check, customer balance charge. Each step returns a `Result`. The result is linear code with no nested `if`.

6. In `OrderLookup.cs` implement `FindOrder(int id)` returning `OneOf<Order, NotFound, InvalidInput>`, mirroring the lesson's `FindUser` example. Use `Match` for exhaustive handling in the caller.

7. In `ReceiptParser.cs` build the hot path: parse a receipt line of the form `"SKU;Qty;Price"` through `StructResult`. Here deliberately do not use `int.Parse`, which throws `FormatException` — write a safe `TryParseInt`-style parser that returns `StructResult<int, ParseError>`. Process 100 000 lines in a loop and confirm that no exception is raised.

8. In `Program.cs` (top-level statements) demonstrate every case: a successful order, an order with `OutOfStock`, an order lookup through `OneOf`, and receipt parsing. The output must contain localized messages with error codes, e.g. `Error [OUT_OF_STOCK]: Insufficient stock`.

9. In `Benchmarks.cs` use BenchmarkDotNet to compare three approaches to parsing an invalid line: `int.Parse` in `try/catch`, `int.TryParse`, and your `StructResult` parser. The expected outcome is that `StructResult` is dozens of times faster than `try/catch` and allocates no memory. Run the benchmark:
   ```
   dotnet run -c Release --project M07L07.Homework -- --filter *ParseBenchmark*
   ```

10. Show where an exception is still appropriate: add a `ChargeCard` method for a genuinely broken situation (severed gateway connection) that throws `PaymentGatewayUnavailableException`. Comment in code why this is an exception rather than a `Result`.

#### Requirements

The solution compiles on .NET 8 with `LangVersion=latest` and `#nullable enable`. C# 12 is used: primary constructors, collection expressions, pattern matching, raw string literals where appropriate. All expected errors (`InvalidInput`, `NotFound`, `OutOfStock`, `PaymentDeclined`, `RuleViolation`, `ParseError`) are returned through `Result` or `OneOf` — there is no `throw new ...Exception` in business logic except the explicitly justified `PaymentGatewayUnavailableException`. The error type is explicit (`OrderError`, `ParseError`), not `string` and not `Exception`. Step composition in `PlaceOrder` goes through `Bind`, with no nested `if (result.IsSuccess)`. Result handling is via `switch` or `Match`, exhaustive. The hot path `ReceiptParser` uses a `readonly struct StructResult`, not a reference `record`, and does not allocate. The benchmark runs in Release configuration and shows a measurable advantage of `StructResult` over `try/catch`. Every method's contract is honest: the signature reveals that the operation can fail and how. The code carries Russian and English comments in key spots. File, class, and method names match the description above. The program output is reproducible and clear.

#### Pitfalls

The main mistake newcomers repeat is confusing a "crash" with "did not succeed." A payment refusal due to insufficient balance is a normal business outcome — it should be returned as `Result.Fail(PaymentDeclined)`, not thrown as `PaymentDeclinedException`. An exception is appropriate only when something is genuinely broken: a severed gateway connection, a violated program invariant, an `OutOfMemoryException`. Clearly separate these two worlds in a comment.

The second mistake is using `string` as the error type. `Result<T, string>` looks convenient, but the caller cannot exhaustively handle branches in a `switch`, and typos in codes are not caught by the compiler. Introduce a typed `OrderError` (enum or record hierarchy) as the lesson requires. The third mistake is swallowing an error: `if (!result.IsSuccess) return;` without propagating `Error` upward. Either handle it or propagate it through `Bind`/`Match`. The fourth is deep nesting of `if (result.IsSuccess)`, which the lesson explicitly calls an anti-pattern: use `Bind` to keep the code linear.

The fifth pitfall is `null` instead of an explicit result. Do not return `null` from `FindOrder` — use `OneOf<Order, NotFound, InvalidInput>`, and the contract starts to speak. The sixth is allocations on the hot path: a reference `record Result<T, TError>` allocates on every call, which kills performance when parsing millions of lines. Switch to a `readonly struct StructResult` — it does not allocate and runs in nanoseconds, as the lesson asserts. The seventh is forgetting `#nullable enable`: without it, `T? Value` does not protect against `null`, and `result.Value!` becomes a hidden bomb. Enable nullable and treat `Value!` deliberately. The eighth is an incomplete `switch` without `_`: if you did not list all `OneOf` variants, the OneOf compiler will not let a `Match` compile without all branches, but in a `switch` over records you need a `_` or a `nameof` default. The ninth is mixing `OneOf` and `Result` in one method without reason: pick one tool for an operation. `Result` is for the bidirectional "success/failure with a typed error," `OneOf` is for several fundamentally different successful outcomes or several error shapes with different structure.

#### Acceptance criteria

- [ ] The `M07L07.Homework` project builds on .NET 8 with `#nullable enable` and C# 12.
- [ ] `Result<T, TError>` is implemented as a `record` with a primary constructor, `Ok`/`Fail`/`Bind`.
- [ ] `Bind` short-circuits on `Fail` and returns the same error.
- [ ] The error type is typed (`OrderError`/`ParseError`), not `string` or `Exception`.
- [ ] `PlaceOrder` uses `Bind` for composition with no nested `if`.
- [ ] `FindOrder` returns `OneOf<Order, NotFound, InvalidInput>`.
- [ ] The caller handles `OneOf` exhaustively through `Match`.
- [ ] `StructResult` is implemented as an allocation-free `readonly struct`.
- [ ] `ReceiptParser` parses 100 000 lines with zero exceptions.
- [ ] The BenchmarkDotNet benchmark shows `StructResult` beating `try/catch`.
- [ ] Business logic has no `throw new ...Exception` except the justified `PaymentGatewayUnavailableException`.
- [ ] `PaymentGatewayUnavailableException` carries a comment explaining why it is an exception.
- [ ] `Program.cs` output contains error codes like `[OUT_OF_STOCK]`.
- [ ] No error is swallowed: each is either handled or propagated.
- [ ] The `switch` over `Result` is exhaustive, with no careless missing `_`.

#### Hints (no direct answer)

- Consider which type is better for an error: an enum (fast but fieldless) or a record hierarchy (with fields like `MissingQty`). Recall the lesson's recommendation of "meaningful `Code` and `Message`."
- For `Bind` recall the lesson signature: `IsSuccess ? next(Value!) : Result<TNext>.Fail(Error!)`. Generalize it to `TError`.
- In `PlaceOrder`, start with `ValidateQty(...)` and `.Bind(_ => CheckStock(...)).Bind(_ => Charge(...))` — a step chain.
- For `OneOf.Match` recall the lesson `FindUser` example: three lambdas in a row.
- In `StructResult` be careful with `default!` for `Value` — it stays uninitialized on `Fail`.
- For the benchmark use `[MemoryDiagnoser]` to see `try/catch` allocations.
- In `ChargeCard` ask yourself: "If the gateway is down, can I continue?" If not — it is genuinely broken.

#### Reference solution walk-through (English)

```csharp
// C# 12 / .NET 8 — reference solution for homework M07-L07

using OneOf;

namespace M07L07.Homework;

// 1) Typed domain order error
public record OrderError(string Code, string Message);
public static class OrderErrors
{
    public static OrderError InvalidInput(string msg) => new("INVALID_INPUT", msg);
    public static OrderError OutOfStock(int requested, int available) =>
        new("OUT_OF_STOCK", $"Requested {requested}, available {available}");
    public static OrderError PaymentDeclined(string reason) => new("PAYMENT_DECLINED", reason);
    public static OrderError RuleViolation(string msg) => new("RULE_VIOLATION", msg);
    public static OrderError NotFound(int id) => new("NOT_FOUND", $"Order {id} not found");
}

// 2) Reference Result on a record with primary constructor
public record Result<T, TError>(bool IsSuccess, T? Value, TError? Error)
{
    public static Result<T, TError> Ok(T value) => new(true, value, default);
    public static Result<T, TError> Fail(TError error) => new(false, default, error);

    // Monadic composition: short-circuit on error
    public Result<TNext, TError> Bind<TNext>(Func<T, Result<TNext, TError>> next) =>
        IsSuccess ? next(Value!) : Result<TNext, TError>.Fail(Error!);
}

// 3) Allocation-free struct Result for the hot path
public readonly struct StructResult<T, TError>
{
    public bool IsSuccess { get; }
    public T? Value { get; }
    public TError? Error { get; }

    private StructResult(bool ok, T? value, TError? error) => (IsSuccess, Value, Error) = (ok, value, error);
    public static StructResult<T, TError> Ok(T value) => new(true, value, default);
    public static StructResult<T, TError> Fail(TError error) => new(false, default, error);

    public StructResult<TNext, TError> Bind<TNext>(Func<T, StructResult<TNext, TError>> next) =>
        IsSuccess ? next(Value!) : StructResult<TNext, TError>.Fail(Error!);
}

// 4) Domain entities
public record Product(string Sku, decimal Price, int Stock);
public record Customer(int Id, string Name, decimal Balance);
public record Order(int Id, Customer Customer, Product Product, int Qty, decimal Total);

// 5) Order service: composition via Bind
public class OrderService
{
    private int _nextId = 1;

    public Result<Order, OrderError> PlaceOrder(Customer c, Product p, int qty) =>
        ValidateQty(qty)
            .Bind(_ => CheckStock(p, qty))
            .Bind(_ => Charge(c, p, qty))
            .Bind(_ => Result<Order, OrderError>.Ok(new Order(_nextId++, c, p, qty, p.Price * qty)));

    private static Result<Unit, OrderError> ValidateQty(int qty) =>
        qty > 0
            ? Result<Unit, OrderError>.Ok(default)
            : Result<Unit, OrderError>.Fail(OrderErrors.InvalidInput("Qty must be > 0"));

    private static Result<Unit, OrderError> CheckStock(Product p, int qty) =>
        qty <= p.Stock
            ? Result<Unit, OrderError>.Ok(default)
            : Result<Unit, OrderError>.Fail(OrderErrors.OutOfStock(qty, p.Stock));

    private static Result<Unit, OrderError> Charge(Customer c, Product p, int qty)
    {
        var total = p.Price * qty;
        return c.Balance >= total
            ? Result<Unit, OrderError>.Ok(default)
            : Result<Unit, OrderError>.Fail(OrderErrors.PaymentDeclined("Insufficient funds"));
    }

    // Genuinely broken — here an EXCEPTION, not a Result
    public void ChargeCard(Customer c, decimal amount, IGateway gateway)
    {
        // A severed connection is a crash, cannot continue
        if (!gateway.IsAvailable)
            throw new PaymentGatewayUnavailableException("Gateway unavailable");
        // ... charge ...
    }
}

public readonly record struct Unit;
public class PaymentGatewayUnavailableException : Exception
{
    public PaymentGatewayUnavailableException(string msg) : base(msg) { }
}
public interface IGateway { bool IsAvailable { get; } }

// 6) OneOf for order lookup
public class NotFound;
public class InvalidInput;
public class OrderLookup
{
    private readonly Dictionary<int, Order> _orders;
    public OrderLookup(Dictionary<int, Order> orders) => _orders = orders;

    public OneOf<Order, NotFound, InvalidInput> FindOrder(int id) =>
        id switch
        {
            <= 0 => new InvalidInput(),
            _ when _orders.TryGetValue(id, out var o) => o,
            _ => new NotFound()
        };

    public string Describe(int id) =>
        FindOrder(id).Match(
            o => $"Found: Order #{o.Id} ({o.Qty} x {o.Product.Sku})",
            _ => $"Error [NOT_FOUND]: Order {id} not found",
            _ => $"Error [INVALID_INPUT]: Id must be > 0"
        );
}

// 7) Safe parser for the hot path
public enum ParseError { Empty, NotANumber, Overflow }

public static class ReceiptParser
{
    public static StructResult<int, ParseError> TryParseInt(string s)
    {
        if (string.IsNullOrEmpty(s)) return StructResult<int, ParseError>.Fail(ParseError.Empty);
        if (!int.TryParse(s, out var v)) return StructResult<int, ParseError>.Fail(ParseError.NotANumber);
        return StructResult<int, ParseError>.Ok(v);
    }

    public static StructResult<(string Sku, int Qty, decimal Price), ParseError> ParseLine(string line)
    {
        var parts = line.Split(';');
        if (parts.Length != 3) return StructResult<(string, int, decimal), ParseError>.Fail(ParseError.Empty);
        var qty = TryParseInt(parts[1]);
        if (!qty.IsSuccess) return StructResult<(string, int, decimal), ParseError>.Fail(qty.Error);
        var price = TryParseInt(parts[2]);
        if (!price.IsSuccess) return StructResult<(string, int, decimal), ParseError>.Fail(price.Error);
        return StructResult<(string, int, decimal), ParseError>.Ok((parts[0], qty.Value, price.Value));
    }
}
```

Line-by-line walk-through. The `OrderError` type is a typed `record` with `Code` and `Message`, not a `string`: the lesson demands "meaningful `Code` and `Message`, otherwise the caller cannot react." The factories `InvalidInput`, `OutOfStock`, and so on produce meaningful messages. `Result<T, TError>` is a record with a C# 12 primary constructor (`public record Result<T, TError>(bool IsSuccess, T? Value, TError? Error)`), matching the lesson example, plus the `TError` generalization and a `Bind` for monadic composition. The line `IsSuccess ? next(Value!) : Result<TNext, TError>.Fail(Error!)` exactly reproduces the short-circuit from the lesson: on `Fail` the chain goes no further and returns the same error. `StructResult` is a `readonly struct` so that the hot path `ReceiptParser` does not allocate: the lesson stresses "in hot paths implement `Result` as a `readonly struct`." `PlaceOrder` is built as `ValidateQty(...).Bind(_ => CheckStock(...)).Bind(_ => Charge(...)).Bind(_ => Ok(new Order(...)))` — linear code with no nested `if`, as the lesson's best practice requires. Each step returns `Result<Unit, OrderError>`, where `Unit` is a marker for "no value, but success." `ChargeCard` deliberately throws `PaymentGatewayUnavailableException`, because a severed gateway is genuinely broken per the lesson: "a severed connection to a critical service." It is the only `throw` in business logic, and the comment explains why. `OrderLookup.FindOrder` returns `OneOf<Order, NotFound, InvalidInput>` in the spirit of the lesson's `FindUser` example, and `Describe` handles all three branches through `Match`, which forces exhaustive coverage of outcomes. `ReceiptParser.TryParseInt` and `ParseLine` use `StructResult`, do not throw `FormatException`, and do not allocate — a demonstration of the pattern's production advantage on the hot path. `int.TryParse` inside is a legitimate bridge to BCL, but our public contract remains `StructResult`, not `bool`. Thus the reference covers every lesson concept: record + primary constructor, `Bind`, `switch`/`Match`, `OneOf`, `readonly struct`, typed error, and the boundary between an exception and a result.

#### Going deeper (bonus)

1. Implement `Map` (also known as `Select`) alongside `Bind` and rewrite `PlaceOrder` through a `Map`+`Bind` combination, returning an `Order` only at the final step. Compare readability.
2. Add accumulation of multiple validation errors (no short-circuit): a `Validation<TError>` type as a list of errors, like F# Validation. Show that `PlaceOrder` can return every violation at once.
3. Implement `Result` as a discriminated union on `OneOf<T, TError>` and compare ergonomics with a custom record: which approach forces fuller branch handling?
4. Wire up a Source Generator or a Roslyn analyzer that forbids `throw new *Exception` in methods marked `[ResultOnly]` and forces `Result`.

---

#### Чек-лист сдачи / Submission checklist

- [ ] Проект собирается под .NET 8 с C# 12 и `#nullable enable`.
- [ ] `Result<T, TError>` и `StructResult` реализованы и покрыты `Bind`.
- [ ] `PlaceOrder` использует `Bind`, без вложенных `if`.
- [ ] `FindOrder` возвращает `OneOf` и обрабатывается через `Match`.
- [ ] Бенчмарк BenchmarkDotNet запущен в Release, результат приложен.
- [ ] Единственное исключение — `PaymentGatewayUnavailableException` с комментарием.
- [ ] Вывод программы воспроизводим и содержит коды ошибок.
- [ ] Файл `README.md` в проекте описывает, как запустить бенчмарк.
- [ ] The project builds on .NET 8 with C# 12 and `#nullable enable`.
- [ ] `Result<T, TError>` and `StructResult` are implemented with `Bind`.
- [ ] `PlaceOrder` uses `Bind` with no nested `if`.
- [ ] `FindOrder` returns `OneOf` and is handled via `Match`.
- [ ] The BenchmarkDotNet benchmark is run in Release and the result is attached.
- [ ] The only exception is `PaymentGatewayUnavailableException` with a comment.
- [ ] The program output is reproducible and contains error codes.
- [ ] A `README.md` in the project describes how to run the benchmark.

#### Ресурсы / Resources

- [Microsoft Learn — C# 12 what's new — https://learn.microsoft.com/dotnet/csharp/whats-new/csharp-12](https://learn.microsoft.com/dotnet/csharp/whats-new/csharp-12)
- [OneOf library (NuGet) — https://github.com/mcintyre321/OneOf](https://github.com/mcintyre321/OneOf)
- [BenchmarkDotNet — https://benchmarkdotnet.org/](https://benchmarkdotnet.org/)
- [Microsoft Learn — Best practices for exceptions — https://learn.microsoft.com/dotnet/standard/exceptions/best-practices-for-exceptions](https://learn.microsoft.com/dotnet/standard/exceptions/best-practices-for-exceptions)
- [Microsoft Learn — Pattern matching — https://learn.microsoft.com/dotnet/csharp/fundamentals/functional/pattern-matching](https://learn.microsoft.com/dotnet/csharp/fundamentals/functional/pattern-matching)

---

[⬆ К модулю M07](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
