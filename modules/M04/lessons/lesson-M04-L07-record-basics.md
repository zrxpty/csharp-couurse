[⬆ К модулю M04](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M04-L07: record (кратко, для value-семантики) / record (brief, value semantics)

**Модуль / Module:** M04
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

`record` — это специальный вид типа в C#, появившийся в C# 9 (как `record class`) и расширенный в C# 10 (как `record struct`). Его главная задача — дать вам **value-семантику** (сравнение по содержимому) с минимумом кода. Представьте себе класс, у которого компилятор сам написал `Equals`, `GetHashCode`, `==`, `!=`, `ToString`, копирование через `with` и деконструкцию — это и есть record.

**Зачем нужен record?** В обычном `class` сравнение по умолчанию идёт по ссылке: два разных объекта с одинаковыми полями не равны. Для «объекта-значения» (координаты, деньги, дата-диапазон, DTO с неизменяемыми полями) это неудобно — хочется, чтобы две точки `(1, 2)` были равны независимо от того, где созданы. Можно переопределить `Equals`/`GetHashCode` вручную, но это рутинно и чревато ошибками. `record` делает это автоматически.

**record class vs record struct.** `record class` (или просто `record`) — это ссылочный тип с value-равенством: равенство проверяется по полям, но объект живёт в куче и передаётся по ссылке. `record struct` — значимый тип: живёт на стеке (или inline в массиве), копируется при присваивании, тоже имеет value-равенство. Выбирайте `record struct` для маленьких компактных значений (точка, цвет, деньги), `record class` — когда нужна ссылочная семантика хранения, но содержательное равенство (например, большой DTO, который дорого копировать).

**Value equality.** Компилятор генерирует метод `Equals(T other)`, который сравнивает все поля (для `record class` — все, включая базовые). Также генерируются `==` и `!=`, которые вызывают `Equals`. Важно: равенство учитывает типы — `record Person` и `record Animal` с одинаковыми полями не равны, даже если данные совпадают.

**with-expressions.** Главное удобство неизменяемых record — создание копии с изменёнными полями: `var moved = point with { X = 5 };`. Старый объект не трогается, новый получает изменённое поле. Это работает благодаря тому, что record генерирует защищённый конструктор копирования и метод `Clone()`. Для `record struct` `with` тоже работает (C# 10+).

**init-only свойства.** Сравните:
```csharp
public string Name { get; init; } // можно задать только при создании
```
`init` позволяет присвоить значение только в конструкторе или инициализаторе объекта, но не позже. Records по умолчанию используют позиционные параметры `record Point(int X, int Y);`, которые компилятор разворачивает в `init`-свойства — поэтому record естественным образом неизменяем. Если нужна изменяемость — добавьте `set` вручную, но это ломает чистую value-семантику (осторожно с `with` и хэш-таблицами).

**Деконструкция.** Позиционный record автоматически получает метод `Deconstruct`, поэтому работает синтаксис `var (x, y) = point;`. Удобно для pattern matching и распаковки.

**Когда record, а когда class?**
- Record — если объект — это **значение**: его идентичность = его данные, он неизменяем (или почти), равенство определяется полями. Примеры: `Money`, `DateRange`, `Coordinate`, `Point`, `Money`, DTO ответа API.
- Class — если объект — это **сущность** с идентичностью независимо от данных, изменяемая, с поведением и инкапсуляцией состояния. Примеры: `BankAccount` (идентичность по номеру счёта), `UserSession`, `Order` в доменной модели.
- Struct (обычный, не record) — для маленьких高性能 значений без value-равенства по умолчанию, когда не нужна вся автоматика record.
- Record struct — когда нужно и значимое размещение, и value-равенство с `with`/`Deconstruct`.

**Короткая аналогия:** обычный `class` — это «коробка с биркой»: две коробки с одинаковым содержимым всё равно разные коробки. `record` — это «этикетка на содержимом»: важен сам состав, а не экземпляр бумаги. `record struct` — «содержимое без бирки вообще»: данные лежат прямо в переменной, без ссылки на коробку.

#### Theory (EN)

`record` is a special kind of type in C#, introduced in C# 9 (as `record class`) and extended in C# 10 (as `record struct`). Its main job is to give you **value semantics** (comparison by content) with a minimum of code. Imagine a class for which the compiler itself writes `Equals`, `GetHashCode`, `==`, `!=`, `ToString`, copy-through-`with`, and deconstruction — that is a record.

**Why does record exist?** In an ordinary `class`, equality defaults to reference comparison: two distinct objects with identical fields are not equal. For a "value object" (a coordinate, an amount of money, a date range, an immutable DTO) this is inconvenient — you want two points `(1, 2)` to be equal no matter where they were created. You could override `Equals`/`GetHashCode` by hand, but that is tedious and error-prone. `record` does it automatically.

**record class vs record struct.** A `record class` (or just `record`) is a reference type with value equality: equality is checked field-by-field, but the object lives on the heap and is passed by reference. A `record struct` is a value type: it lives on the stack (or inline in an array), is copied on assignment, and also has value equality. Pick `record struct` for small compact values (point, color, money), and `record class` when you want reference-style storage but meaningful equality (for example a large DTO that is expensive to copy around).

**Value equality.** The compiler synthesizes an `Equals(T other)` method that compares every field (for `record class` — all of them, including inherited ones). It also synthesizes `==` and `!=` operators that call `Equals`. Note that equality honors runtime types: a `record Person` and a `record Animal` with identical fields are not equal, even if the data matches.

**with-expressions.** The headline convenience of immutable records is creating a copy with some fields changed: `var moved = point with { X = 5 };`. The original object is untouched, the new one gets the modified field. This works because a record generates a protected copy constructor and a `Clone()` method. For `record struct`, `with` works too (C# 10+).

**init-only properties.** Compare:
```csharp
public string Name { get; init; } // settable only during construction
```
`init` allows assignment only in a constructor or object initializer, never afterwards. Records by default use positional parameters `record Point(int X, int Y);`, which the compiler expands into `init`-only properties — so a record is naturally immutable. If you need mutability, add `set` manually, but that undermines clean value semantics (be careful with `with` and hash tables).

**Deconstruction.** A positional record automatically gets a `Deconstruct` method, so the syntax `var (x, y) = point;` works out of the box. Handy for pattern matching and unpacking.

**When record, when class?**
- Record — when the object *is a value*: its identity equals its data, it is immutable (or nearly so), equality is defined by fields. Examples: `Money`, `DateRange`, `Coordinate`, `Point`, API response DTOs.
- Class — when the object is an *entity* with an identity independent of its data, is mutable, carries behavior and encapsulated state. Examples: `BankAccount` (identity by account number), `UserSession`, `Order` in a domain model.
- A plain struct (not record) — for small high-performance values that do not need automatic value equality.
- Record struct — when you want value-type storage *and* value equality with `with`/`Deconstruct`.

**Short analogy:** an ordinary `class` is a "labeled box": two boxes with the same contents are still different boxes. A `record` is a "label on contents": what matters is the contents, not the piece of paper. A `record struct` is "contents with no box at all": the data lives directly in the variable, with no reference to a box.

#### Пример кода / Code Example

```csharp
// C# 12 / .NET 8 — рабочий пример: record class, record struct, with, init, деконструкция
using System;

// Positional record class (C# 9+) — ссылочный тип с value-равенством / reference type with value equality
// Позиционный record автоматически создаёт init-only свойства, Equals, GetHashCode, ==, !=, ToString, Deconstruct
public record class Point(int X, int Y); // координата — это значение / a coordinate is a value

// Positional record struct (C# 10+) — значимый тип с value-равенством / value type with value equality
// Живёт на стеке/inline, копируется при присваивании / lives on stack/inline, copied on assignment
public record struct Money(decimal Amount, string Currency);

// Record с расширенным телом: добавляем валидацию в конструкторе / record with body: add validation
public record Temperature(double Celsius)
{
    public double Celsius { get; init; } = Celsius >= -273.15 ? Celsius : throw new ArgumentOutOfRangeException(nameof(Celsius));

    // Вычисляемое свойство — не участвует в равенстве (только поля/свойства init/get) / computed: not part of equality
    public double Fahrenheit => Celsius * 9.0 / 5.0 + 32.0;
}

class Program
{
    static void Main()
    {
        // Value equality: два разных экземпляра с одинаковыми данными равны / two distinct instances with same data are equal
        var p1 = new Point(1, 2);
        var p2 = new Point(1, 2);
        Console.WriteLine(p1 == p2);          // True (ссылки разные, но данные одинаковые / different refs, same data)
        Console.WriteLine(ReferenceEquals(p1, p2)); // False
        Console.WriteLine(p1);                // Point { X = 1, Y = 2 } — автогенерированный ToString

        // with-выражение: копия с изменённым полем / with-expression: copy with a changed field
        var moved = p1 with { X = 5 };
        Console.WriteLine(moved);             // Point { X = 5, Y = 2 }
        Console.WriteLine(p1);                // Point { X = 1, Y = 2 } — оригинал не изменён / original untouched

        // record struct: копируется при присваивании / copied on assignment
        var price = new Money(19.95m, "EUR");
        var discounted = price with { Amount = price.Amount * 0.9m };
        Console.WriteLine(discounted);        // Money { Amount = 17.955, Currency = EUR }

        // Деконструкция позиционного record / deconstruction of a positional record
        var (x, y) = p1;
        Console.WriteLine($"Decomposed: x={x}, y={y}"); // Decomposed: x=1, y=2

        // init-only: можно задать только при создании / settable only at construction
        var temp = new Temperature(25.0) { }; // ok
        // temp.Celsius = 30.0; // ОШИБКА компиляции: init-only / COMPILE ERROR: init-only

        // with на record с телом — создаёт копию через конструктор копирования / with on record with body
        var hotter = temp with { };           // копия с тем же значением / copy with same value
        Console.WriteLine(hotter.Fahrenheit); // 77

        // record struct vs record class в коллекциях: value equality работает для поиска/уникальности
        var set = new HashSet<Point> { p1, p2, moved };
        Console.WriteLine(set.Count);         // 2 — p1 и p2 схлопнулись (равны), moved отличается
    }
}
```

#### Best Practices
- Делайте позиционные record неизменяемыми: используйте `init`-свойства по умолчанию, не добавляйте `set` без необходимости. (RU)
- Предпочитайте `record struct` для маленьких (≤ 16 байт) значений с value-равенством; `record class` — для крупных DTO и моделей с равенством по содержимому. (RU)
- Не вводите изменяемые ссылочные поля в record — они ломают `GetHashCode` и `with`. Если нужно — храните `ImmutableArray`/`ReadOnlyCollection`. (RU)
- Используйте `with` для «модификации» вместо ручного клонирования: короче, меньше ошибок. (RU)
- Помните, что `Equals` у record учитывает runtime-тип: наследование record может привести к неожиданностям — старайтесь делать record `sealed`. (RU)
- Keep positional records immutable: rely on `init` properties by default, do not add `set` without a reason. (EN)
- Prefer `record struct` for small (≤ 16 bytes) values with value equality; use `record class` for large DTOs and content-equal models. (EN)
- Avoid mutable reference fields inside a record — they break `GetHashCode` and `with`. If needed, store `ImmutableArray`/`ReadOnlyCollection`. (EN)
- Use `with` for "modification" instead of manual cloning: shorter, fewer errors. (EN)
- Remember that record `Equals` honors the runtime type: record inheritance can surprise you — prefer making records `sealed`. (EN)

#### Частые ошибки / Common Mistakes
- [Добавили `set` в позиционный record и положили экземпляр в `HashSet`/словарь] → [Не делайте record изменяемым; для коллекций-по-значению оставляйте `init`, либо пересоздавайте через `with` и убирайте/перевставляйте.] (RU)
- [Сравниваете record через `ReferenceEquals` и удивляетесь `False`] → [Record сравнивайте через `==` или `Equals` — там value-семантика; `ReferenceEquals` проверяет только ссылку.] (RU)
- [Наследуете один record от другого и ждёте равенства при разных типах] → [Равенство учитывает runtime-тип — два record с одинаковыми полями, но разными типами не равны. Делайте record `sealed` или проектируйте иерархию осознанно.] (RU)
- [Используете `record struct` с большим количеством полей] → [Копирование при каждом присваивании станет дорогим; для крупных значений берите `record class`.] (RU)
- [Забыли, что вычисляемые свойства не входят в равенство] → [Это нормально, но помните: `Equals`/`GetHashCode` учитывают только init/get поля позиций, не вычисляемые.] (RU)
- [Added `set` to a positional record and put an instance in a `HashSet`/dictionary] → [Do not make records mutable; for value-keyed collections keep `init`, or recreate via `with` and re-insert.] (EN)
- [Comparing records via `ReferenceEquals` and surprised by `False`] → [Compare records with `==` or `Equals` — that is where value semantics live; `ReferenceEquals` only checks the reference.] (EN)
- [Deriving one record from another and expecting equality across different types] → [Equality honors the runtime type — two records with identical fields but different types are not equal. Make records `sealed` or design the hierarchy deliberately.] (EN)
- [Using `record struct` with many fields] → [Copying on every assignment gets expensive; for large values use `record class`.] (EN)
- [Forgot that computed properties are not part of equality] → [That is by design, but remember: `Equals`/`GetHashCode` only consider positional init/get fields, not computed ones.] (EN)

#### Чек-лист самопроверки / Self-check Checklist
- [ ] Я могу объяснить, чем `record class` отличается от `record struct` (ссылочный vs значимый тип, размещение и копирование). (RU)
- [ ] Я знаю, что value equality генерируется компилятором и учитывает runtime-тип. (RU)
- [ ] Я умею использовать `with` для создания копии с изменёнными полями. (RU)
- [ ] Я понимаю, что `init` делает свойство неизменяемым после создания. (RU)
- [ ] Я могу деконструировать позиционный record через `var (a, b) = r;`. (RU)
- [ ] Я могу выбрать между record, class и обычным struct для конкретной задачи. (RU)
- [ ] Я понимаю, почему изменяемые ссылочные поля в record опасны для хэш-таблиц. (RU)
- [ ] I can explain the difference between `record class` and `record struct` (reference vs value type, storage and copying). (EN)
- [ ] I know that value equality is compiler-generated and honors the runtime type. (EN)
- [ ] I can use `with` to create a copy with changed fields. (EN)
- [ ] I understand that `init` makes a property immutable after construction. (EN)
- [ ] I can deconstruct a positional record via `var (a, b) = r;`. (EN)
- [ ] I can choose between record, class, and a plain struct for a given task. (EN)
- [ ] I understand why mutable reference fields in a record are dangerous for hash tables. (EN)

#### Ресурсы / Resources
- [Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/fundamentals/types/records](https://learn.microsoft.com/dotnet/csharp/fundamentals/types/records)

---

[⬆ К модулю M04](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
