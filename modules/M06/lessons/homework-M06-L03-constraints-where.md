---
[← К уроку M06-L03](lesson-M06-L03-constraints-where.md) | [⬆ К модулю M06](../README.md) | [Следующее ДЗ →](homework-M06-L04-dictionary-hashset.md)
---

### Домашнее задание M06-L03: Ограничения where (class/struct/new/interface) / Homework M06-L03: Constraints where (class/struct/new/interface)

**Урок / Lesson:** M06-L03
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 4/5
**Цель / Goal:** (RU) Научиться осознанно применять ограничения `where` к обобщённым типам и методам: выбирать минимально достаточное ограничение, корректно комбинировать несколько ограничений для одного и разных параметров типа, учитывать nullable-контекст и избегать типичных ошибок (порядок `new()`, несовместимость `struct`+`class`, лишние ограничения «на всякий случай»). (EN) Learn to deliberately apply `where` constraints to generic types and methods: choose the minimal sufficient constraint, correctly combine several constraints for the same and for different type parameters, account for the nullable context, and avoid typical mistakes (`new()` ordering, `struct`+`class` incompatibility, redundant "just-in-case" constraints).

#### Связь с уроком / Connection to the lesson
(RU — 2–3 предложения)
Это задание напрямую закрепляет все восемь видов ограничений из урока M06-L03: `class`, `struct`, `new()`, интерфейсные и базовые ограничения, `notnull`, `unmanaged`, а также комбинации `where` и правило «минимально достаточного» ограничения. Вы будете переписывать обобщённый код, который без ограничений не компилируется или работает неоптимально (boxing, `object.Equals`), и доводить его до состояния, в котором компилятор обеспечивает нужные гарантии. Все best practices и частые ошибки из урока являются чек-листом приёмки.
(EN — same)
This homework directly reinforces all eight constraint kinds from lesson M06-L03: `class`, `struct`, `new()`, interface and base-class constraints, `notnull`, `unmanaged`, as well as `where` combinations and the "minimal sufficient" principle. You will rewrite generic code that either does not compile or works suboptimally (boxing, `object.Equals`) without constraints, and bring it to a state where the compiler enforces the required guarantees. Every best practice and common mistake from the lesson is part of the acceptance checklist.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация
Представьте, что вы присоединяетесь к команде, которая разрабатывает библиотеку «универсальных утилит» `GenericToolkit` для внутреннего использования в нескольких проектах компании. Библиотека уже существует, но написана «в лоб»: автор обобщённых классов не накладывал ограничения `where` вообще, поэтому код либо не компилируется, либо содержит уязвимости к `null`, либо неоптимален из-за boxing при сравнениях через `object.Equals`. Ваша задача — отрефакторить ключевые компоненты библиотеки так, чтобы компилятор гарантировал нужные свойства типов, а код стал быстрее и безопаснее.

Библиотека состоит из нескольких независимых компонентов, каждый из которых демонстрирует один или несколько видов ограничений из урока. Конкретно: `Repository<TEntity>` хранит сущности и создаёт их по запросу — здесь нужны `class`, интерфейсное ограничение `IEntity` и `new()`. `Range<T>` представляет диапазон значений (для чисел, дат, перечислений) — здесь нужен `struct` вместе с `IComparable<T>`. `Factory<T>` порождает объекты с предустановкой — здесь `class, new()`. `Cache<TKey, TValue>` запрещает nullable-ключи — `notnull`. `Vector2<T>` — векторная математика без аллокаций — `struct, INumber<T>` и опционально `unmanaged`. Дополнительно вы реализуете статический `Sorter` с `IComparable<T>` и一个小-й бонус с `enum`/`delegate` ограничениями.

Мотивация к заданию двойная. Во-первых, на практике ограничения `where` — это не теоретическое украшение, а инструмент, который смещает проверки с времени выполнения на время компиляции: вместо `if (entity is null)` компилятор сам доказывает, что `null` невозможен, вместо `as IEntity` — что интерфейс реализован. Во-вторых, избыток ограничений убивает переиспользование: если на `Repository` навесить `where TEntity : class, IEntity, new(), IComparable<IEntity>`, то половина сущностей не подойдёт, хотя сравнение в репозитории не используется. Урок формулирует главное правило — «минимально достаточно» — и это задание заставляет вас применять его осознанно на каждом компоненте.

#### Что нужно сделать (пошагово)
1. Создайте новый проект консольного приложения .NET 8: выполните в терминале команду `dotnet new console -n GenericToolkit -o GenericToolkit --framework net8.0`, перейдите в папку `cd GenericToolkit` и откройте проект в редакторе. Убедитесь, что в `GenericToolkit.csproj` присутствует `<Nullable>enable</Nullable>` и `<LangVersion>latest</LangVersion>` — если их нет, добавьте вручную, чтобы были доступны все возможности C# 12 и nullable-аннотации.
2. Включите treat-warnings-as-errors для серьёзности: добавьте в `.csproj` `<TreatWarningsAsErrors>true</TreatWarningsAsErrors>` и `<Nullable>enable</Nullable>`. Это заставит вас устранить все nullable-предупреждения — без них невозможно пройти критерии приёмки.
3. Создайте файл `GenericToolkit.cs` с пространством имён `GenericToolkit`. В нём разместите все классы задания. Используйте top-level statements в `Program.cs` для демонстрации работы — точка входа должна вызывать методы классов и выводить результаты.
4. Реализуйте интерфейс `IEntity` со свойством `int Id { get; init; }`. Реализуйте sealed-класс `Product : IEntity` со свойствами `Id` и `Name` (строка, init, дефолт `""`). Реализуйте sealed-класс `Order : IEntity` со свойствами `Id` и `Total` (decimal). Оба класса должны иметь публичный конструктор без параметров, чтобы проходить ограничение `new()`.
5. Реализуйте `Repository<TEntity> where TEntity : class, IEntity, new()` с приватным `Dictionary<int, TEntity>` и методами: `Add(int id)` — создаёт `new TEntity { Id = id }` и кладёт в словарь; `Find(int id)` — возвращает `TEntity?` через `TryGetValue`; `GetOrAdd(int id)` — если элемент есть, возвращает его, иначе создаёт, кладёт и возвращает. Обратите внимание: благодаря `class` вы можете вернуть `null`, а благодаря `new()` — создать экземпляр, а благодаря `IEntity` — записать `Id`.
6. Реализуйте обобщённый `Range<T> where T : struct, IComparable<T>` с полями `Min` и `Max` (readonly), конструктором, который выбрасывает `ArgumentException`, если `min.CompareTo(max) > 0`, методом `Contains(T value)`, возвращающим `min.CompareTo(value) <= 0 && value.CompareTo(max) <= 0`, и методом `Clamp(T value)`. Проверьте работу на `int`, `double` и на пользовательской `struct Point2D : IComparable<Point2D>` (сравнение по сумме координат). Убедитесь, что `string` и `Nullable<int>` НЕ компилируются как аргументы `T` — это доказывает, что ограничение `struct` работает.
7. Реализуйте `Factory<T> where T : class, new()` с методом `Create(Action<T>? configure = null)`, который создаёт `new T()`, опционально применяет конфигурацию и возвращает экземпляр. В демонстрации используйте `Factory<Product>` и `Factory<Order>`. Подумайте, почему `Factory<string>` не скомпилируется — у `string` нет публичного конструктора без параметров.
8. Реализуйте `Cache<TKey, TValue> where TKey : notnull` с приватным `Dictionary<TKey, TValue>`, методами `Set` и `Get` (с `TryGetValue`). Попробуйте в клиентском коде передать `string?` ключ, в котором может быть `null` — компилятор должен выдать предупреждение/ошибку. Это демонстрирует ограничение `notnull` в действии.
9. Реализуйте `Vector2<T> where T : struct, INumber<T>` (используйте `System.Numerics.INumber<T>`), реализующий `IEquatable<Vector2<T>>`, с методами `Add`, `Subtract`, `Scale(T factor)`, операторами `+`, `-`, `==`, `!=` и переопределёнными `Equals`/`GetHashCode` через `HashCode.Combine`. Перегрузите `ToString` для удобной отладки. Сделайте структуру `readonly`. Продемонстрируйте на `Vector2<int>` и `Vector2<double>`.
10. Реализуйте статический класс `Sorter` с методами `Max<T>(T a, T b) where T : IComparable<T>` и `Min<T>` аналогично, а также `IsSorted<T>(IEnumerable<T> items) where T : IComparable<T>`. Сравните производительность с версией без ограничения (через `object.Equals`) на массиве из миллиона `int` — зафиксируйте разницу в `Stopwatch` и выведите на консоль.
11. Бонусно: реализуйте `EnumHelper<T> where T : struct, Enum` с методом `GetValues()`, возвращающим `T[]` через `Enum.GetValues<T>()`, и методом `ParseInvariant(string name)`. Реализуйте `DelegateInvoker<T> where T : Delegate` — обёртку, которая хранит делегат и подсчитывает количество вызовов. Это покажет специальные ограничения `enum` и `delegate`.
12. В `Program.cs` через top-level statements вызовите все компоненты: репозиторий, диапазон, фабрику, кэш, вектор, сортировщик, бонусные утилиты. Выводите результаты с `Console.WriteLine`. Запустите проект командой `dotnet run` и убедитесь, что нет ошибок и предупреждений. Ожидаемый вывод должен содержать, например: `Product id=1`, `Range contains 5: True`, `Factory created Order`, `Cache[answer]=42`, `Vector (4,6)`, `Max(3,7)=7`, `IsSorted: True`.

#### Требования к решению
Код должен компилироваться под C# 12 / .NET 8 без единого предупреждения уровня nullable (учитывая `TreatWarningsAsErrors=true`). Все ограничения `where` должны быть минимально достаточными: нельзя добавлять `class` или `new()` «на всякий случай» — только если код реально использует соответствующие операции (`null`-сравнение, `new T()`, доступ к членам интерфейса). Порядок ограничений в комбинированных секциях должен соответствовать уроку: первыми `class`/`struct`, затем базовый класс или интерфейсы, последним — `new()`. Для разных параметров типа должны быть отдельные секции `where`, расположенные по одному на параметр.

Все обобщённые классы должны быть документированы XML-комментариями `///`, объясняющими **почему** наложено каждое ограничение (например: «`class` — чтобы возвращать `null` из `Find`; `IEntity` — чтобы записывать `Id`; `new()` — чтобы создавать экземпляры в `Add`»). Структуры должны быть `readonly`, где это уместно. Для `Vector2<T>` обязательно переопределение `Equals`/`GetHashCode` и реализация `IEquatable<T>`, чтобы избежать boxing. В `Repository.GetOrAdd` необходимо гарантировать, что один и тот же `id` не создаёт дубликат.

Нельзя использовать рефлексию (`Activator.CreateInstance`) там, где достаточно `new()` — это убивает смысл ограничения. Нельзя использовать `object.Equals` или `object.CompareTo` там, где можно ограничиться `IEquatable<T>`/`IComparable<T>`. Все публичные методы с ссылочными параметрами должны проверять аргументы через `ArgumentNullException.ThrowIfNull`. Проект должен запускаться командой `dotnet run` и выводить ожидаемые результаты.

#### Тонкости и подводные камни
Главная тонкость — порядок ограничений. Если вы поставите `new()` перед интерфейсами (`where T : new(), IEntity, class`), компилятор выдаст ошибку CS0449 или аналогичную: `new()` обязан быть последним. Аналогично `class` и `struct` должны идти первыми и не могут сочетаться друг с другом — попытка `where T : struct, class` некорректна по определению. Запомните мнемонику: «reference/value, then base/interfaces, then constructor».

Вторая тонкость — `where T : struct` исключает `Nullable<T>`. Это значит, что `Range<int?>` не скомпилируется, что часто удивляет новичков, ожидавших «struct — значит, все значимые типы, в том числе nullable». На самом деле `Nullable<T>` — это特殊ная обёртка, которая намеренно запрещена в `struct`-ограничении, чтобы избежать двойной nullable-семантики. Если вам нужно принимать и nullable — используйте `where T : struct` для самого `T`, а nullable делайте на уровне параметров метода, а не типа.

Третья тонкость — `new()` не работает с типами без публичного конструктора без параметров: `string`, все `enum`, абстрактные классы, структуры с явно объявленным конструктором, принимающим аргументы (но без явного дефолтного). Для таких случаев используйте `static abstract` фабричный член интерфейса (C# 11+) или передавайте `Func<T>` явно.

Четвёртая тонкость — `notnull` действует только в nullable-контексте (`<Nullable>enable</Nullable>`). Вне этого контекста ограничение `notnull` фактически ничего не проверяет. Поэтому важно убедиться, что nullable-аннотации включены в проекте. С другой стороны, `class` без вопросительного знака в nullable-контексте означает «non-null reference type», а `class?` — допускает nullable ссылочный тип. Не путайте эти две формы.

Пятая тонкость — производительность. Использование `object.Equals` для значимых типов приводит к boxing и аллокации; `IComparable<object>` аналогично медленнее `IComparable<T>`. Ограничение по интерфейсу `IEquatable<T>`/`IComparable<T>` не только повышает читаемость, но и даёт реальный прирост на горячих путях. Зафиксируйте это в бенчмарке со `Stopwatch`.

Шестая тонкость — `unmanaged` автоматически подразумевает `struct`, но не наоборот: `struct` может содержать ссылочные поля (например, строку), а `unmanaged` — нет. Используйте `unmanaged` осознанно, только когда действительно нужен interop/Span/P/Invoke, иначе вы лишний раз сужаете множество допустимых типов.

#### Критерии приёмки
- [ ] Проект `GenericToolkit` создаётся командой `dotnet new console` под .NET 8 и собирается без ошибок.
- [ ] В `.csproj` включены `<Nullable>enable</Nullable>`, `<LangVersion>latest</LangVersion>`, `<TreatWarningsAsErrors>true</TreatWarningsAsErrors>`.
- [ ] `Repository<TEntity>` имеет ограничения `class, IEntity, new()` в корректном порядке, и использует все три возможности (`null`, `Id`, `new TEntity()`).
- [ ] `Range<T>` имеет ограничения `struct, IComparable<T>`; конструктор выбрасывает `ArgumentException` при `min > max`.
- [ ] `Range<T>` НЕ компилируется с `string` или `int?` как аргументом — продемонстрировано закомментированной попыткой.
- [ ] `Factory<T>` имеет ограничения `class, new()` и НЕ компилируется с `string` (нет дефолтного конструктора).
- [ ] `Cache<TKey, TValue>` использует `notnull` для `TKey` и компилятор выдаёт ошибку/предупреждение при попытке передать `null`-ключ.
- [ ] `Vector2<T>` имеет `struct, INumber<T>`, реализует `IEquatable<Vector2<T>>`, переопределяет `Equals`/`GetHashCode`, поддерживает операторы.
- [ ] `Vector2<T>` — `readonly struct`, без boxing при сравнениях.
- [ ] `Sorter.Max/Min/IsSorted` используют ограничение `IComparable<T>` и не обращаются к `object.Equals`.
- [ ] Бенчмарк `Stopwatch` показывает разницу во времени между `IComparable<T>` и `object.Equals` версией — разница зафиксирована в выводе.
- [ ] Бонус `EnumHelper<T> where T : struct, Enum` компилируется и возвращает значения перечисления.
- [ ] Бонус `DelegateInvoker<T> where T : Delegate` компилируется и подсчитывает вызовы.
- [ ] Все обобщённые классы имеют XML-комментарии, объясняющие причину каждого ограничения.
- [ ] Все ограничения минимально достаточны — нет лишних `class`/`new()` там, где не используются соответствующие операции.
- [ ] Команда `dotnet run` выводит ожидаемые результаты без предупреждений.

#### Подсказки (без прямого ответа)
- Для проверки `Range<string>` просто попробуйте создать `Range<string>` и закомментируйте — компилятор подскажет, какой именно запрет сработал.
- `INumber<T>` находится в пространстве имён `System.Numerics`; добавьте `using System.Numerics;`.
- Для `Enum.GetValues<T>()` (генерик-версия) нужен .NET 5+, в .NET 8 доступна.
- Чтобы вызвать делегат у `T` где `T : Delegate`, приведите его через `((Delegate)(object)del).DynamicInvoke(...)`, либо используйте ограничение по конкретному типу делегата. Подумайте, какой способ безопаснее.
- Для `GetHashCode` нескольких полей используйте `HashCode.Combine(a, b)`.
- `ArgumentNullException.ThrowIfNull` доступен в .NET 6+ и экономит boilerplate.
- Если компилятор ругается на `new TEntity { Id = id }` — проверьте, что `IEntity.Id` имеет `init`, а не только `get`.

#### Эталонное решение (разбор)
```csharp
// C# 12 / .NET 8 — Эталон GenericToolkit / Reference GenericToolkit
using System;
using System.Collections.Generic;
using System.Numerics;

namespace GenericToolkit;

// Интерфейс сущности / Entity interface
public interface IEntity
{
    int Id { get; init; }   // init — доступен для записи при создании / init-settable on construction
}

public sealed class Product : IEntity
{
    public int Id { get; init; }
    public string Name { get; init; } = "";
}

public sealed class Order : IEntity
{
    public int Id { get; init; }
    public decimal Total { get; init; }
}

/// <summary>
/// Репозиторий ссылочных сущностей с идентификатором.
/// class — позволяет возвращать null из Find; IEntity — даёт доступ к Id; new() — позволяет new TEntity().
/// </summary>
public class Repository<TEntity> where TEntity : class, IEntity, new()
{
    private readonly Dictionary<int, TEntity> _store = new();

    public void Add(int id)
    {
        // new() + IEntity.id (init) — оба ограничения использованы / both constraints used
        var entity = new TEntity { Id = id };
        _store[id] = entity;
    }

    public TEntity? Find(int id)                      // class → можно вернуть null / class → nullable return
        => _store.TryGetValue(id, out var e) ? e : null;

    public TEntity GetOrAdd(int id)
    {
        if (_store.TryGetValue(id, out var existing))
            return existing;
        var created = new TEntity { Id = id };        // new() used
        _store[id] = created;
        return created;
    }
}

/// <summary>
/// Диапазон значимых сравнимых значений.
/// struct — только значимые типы (исключает string, Nullable<T>); IComparable<T> — сравнение без boxing.
/// </summary>
public readonly struct Range<T> where T : struct, IComparable<T>
{
    public T Min { get; }
    public T Max { get; }

    public Range(T min, T max)
    {
        if (min.CompareTo(max) > 0)
            throw new ArgumentException($"min {min} > max {max}", nameof(min));
        Min = min; Max = max;
    }

    public bool Contains(T value)
        => Min.CompareTo(value) <= 0 && value.CompareTo(Max) <= 0;

    public T Clamp(T value)
        => value.CompareTo(Min) < 0 ? Min
         : value.CompareTo(Max) > 0 ? Max
         : value;
}

/// <summary>
/// Фабрика объектов с публичным конструктором без параметров.
/// class, new() — единственные нужные ограничения; никаких лишних интерфейсов.
/// </summary>
public class Factory<T> where T : class, new()
{
    public T Create(Action<T>? configure = null)
    {
        var item = new T();                  // new() used
        configure?.Invoke(item);
        return item;
    }
}

/// <summary>
/// Кэш с непустым ключом в nullable-контексте.
/// notnull — запрещает nullable ссылочные ключи; работает только при <Nullable>enable</Nullable>.
/// </summary>
public class Cache<TKey, TValue> where TKey : notnull
{
    private readonly Dictionary<TKey, TValue> _items = new();
    public void Set(TKey key, TValue value) => _items[key] = value;
    public TValue? Get(TKey key) => _items.TryGetValue(key, out var v) ? v : default;
}

/// <summary>
/// Двумерный вектор для произвольных чисел.
/// struct — значимый, без аллокаций; INumber<T> — арифметика; IEquatable<Vector2<T>> — сравнение без boxing.
/// </summary>
public readonly struct Vector2<T> : IEquatable<Vector2<T>>
    where T : struct, INumber<T>
{
    public T X { get; }
    public T Y { get; }
    public Vector2(T x, T y) { X = x; Y = y; }

    public Vector2<T> Add(Vector2<T> o) => new(X + o.X, Y + o.Y);
    public Vector2<T> Subtract(Vector2<T> o) => new(X - o.X, Y - o.Y);
    public Vector2<T> Scale(T f) => new(X * f, Y * f);

    public bool Equals(Vector2<T> o) => X == o.X && Y == o.Y;
    public override bool Equals(object? obj) => obj is Vector2<T> v && Equals(v);
    public override int GetHashCode() => HashCode.Combine(X, Y);
    public override string ToString() => $"({X}, {Y})";

    public static Vector2<T> operator +(Vector2<T> a, Vector2<T> b) => a.Add(b);
    public static Vector2<T> operator -(Vector2<T> a, Vector2<T> b) => a.Subtract(b);
    public static bool operator ==(Vector2<T> a, Vector2<T> b) => a.Equals(b);
    public static bool operator !=(Vector2<T> a, Vector2<T> b) => !a.Equals(b);
}

/// <summary>Сравнения только через IComparable<T> — без boxing.</summary>
public static class Sorter
{
    public static T Max<T>(T a, T b) where T : IComparable<T>
        => a.CompareTo(b) >= 0 ? a : b;
    public static T Min<T>(T a, T b) where T : IComparable<T>
        => a.CompareTo(b) <= 0 ? a : b;
    public static bool IsSorted<T>(IEnumerable<T> items) where T : IComparable<T>
    {
        var first = true;
        T prev = default!;
        foreach (var cur in items)
        {
            if (!first && prev.CompareTo(cur) > 0) return false;
            prev = cur; first = false;
        }
        return true;
    }
}

// Бонус: enum и delegate ограничения / Bonus: enum and delegate constraints
public static class EnumHelper<T> where T : struct, Enum
{
    public static T[] GetValues() => Enum.GetValues<T>();
    public static T ParseInvariant(string name) => Enum.Parse<T>(name, ignoreCase: false);
}

public sealed class DelegateInvoker<T> where T : Delegate
{
    private readonly T _delegate;
    public int CallCount { get; private set; }
    public DelegateInvoker(T del)
    {
        ArgumentNullException.ThrowIfNull(del);
        _delegate = del;
    }
    public object? Invoke(params object?[] args)
    {
        CallCount++;
        return _delegate.DynamicInvoke(args);
    }
}
```

**Разбор по строкам.** В `Repository<TEntity>` ограничение `class` стоит первым (это обязательно по правилу порядка), затем `IEntity` (интерфейс), затем `new()` (обязательно последним). Эти три ограничения **все используются** в коде: `class` — чтобы вернуть `null` из `Find` (значимый тип не мог бы быть `null`); `IEntity` — чтобы записать `Id` в `new TEntity { Id = id }`; `new()` — чтобы создать `new TEntity()`. Если бы мы убрали любое из них, код не скомпилировался бы — значит, ограничение действительно минимально достаточное, а не «на всякий случай». Это ключевая концепция урока.

В `Range<T>` ограничение `struct` гарантирует, что `T` — значимый тип, а `IComparable<T>` даёт доступ к `CompareTo` без boxing. Комбинация `struct, IComparable<T>` разрешает `int`, `double`, `DateTime`, пользовательские `struct`, но запрещает `string` (ссылочный) и `int?` (`Nullable<T>` исключён из `struct`). Конструктор проверяет `min.CompareTo(max) > 0` и выбрасывает `ArgumentException` — это защита инварианта на этапе построения, что соответствует best practice «документировать инварианты типа».

В `Factory<T>` только `class, new()` — это намеренно минимальный набор. Если бы мы добавили `IEntity`, фабрика перестала бы работать с произвольными классами вроде `StringBuilder`. В `Cache<TKey, TValue>` используется `notnull`, который в nullable-контексте запрещает nullable ссылочные ключи — это защищает словарь от `null`-ключа в рантайме, смещая проверку на компиляцию.

В `Vector2<T>` ограничение `struct, INumber<T>` из `System.Numerics` даёт доступ к операторам `+`, `-`, `*` через статические абстрактные члены интерфейса (C# 11+). Это современная альтернатива «числовому» ограничению, которое раньше приходилось эмулировать делегатами. Реализация `IEquatable<Vector2<T>>` и переопределение `Equals(object?)` с делегированием в `Equals(Vector2<T>)` гарантирует, что сравнения через `==` и `Equals` не упаковывают структуру — это и есть тот прирост производительности, ради которого интерфейсные ограничения существуют. `HashCode.Combine` — recommended способ построения хеша без ручной математики.

В `Sorter` ограничение `IComparable<T>` одно, без `class` или `new()` — потому что методу нужно только сравнивать, ничего больше. Это иллюстрация главного правила урока: «минимально достаточно». Если бы мы добавили `where T : class, IComparable<T>`, то `int` перестал бы подходить, хотя сравнение чисел — основной сценарий. В `IsSorted` используется `prev = default!` — `!` гасит nullable-предупреждение, поскольку `prev` гарантированно инициализируется на первой итерации.

В бонусных `EnumHelper<T> where T : struct, Enum` и `DelegateInvoker<T> where T : Delegate` показаны специальные ограничения C# 7.3: они сужают тип до перечислений или делегатов соответственно. `Enum.GetValues<T>()` — генерик-версия из .NET 5+, избавляющая от приведения типа. В `DelegateInvoker` `DynamicInvoke` медленный, но безопасный способ вызвать произвольный делегат; на практике для конкретного типа делегата лучше ограничиваться именно им (`Func<int,int>` и т.п.).

#### Задания на углубление (бонус)
1. Замените `new()` в `Repository` и `Factory` на интерфейс `IInitializable<T>` со `static abstract T Create();` (C# 11+). Сравните два подхода: что лучше для типов без дефолтного конструктора? Какие ограничения это снимает/добавляет?
2. Реализуйте `SpanFriendlyBuffer<T> where T : unmanaged` с фиксированным буфером `Span<T>` и методами чтения/записи по индексу. Сравните производительность с `T[]` через `BenchmarkDotNet`. Зафиксируйте разницу в наносекундах.
3. Добавьте `where T : U` (conversion constraint) в обобщённый метод `Converter.Convert<TSource, TTarget>(TSource source) where TSource : TTarget` и продемонстрируйте безопасное сужающее преобразование без `as`/`is`.
4. Реализуйте `WeakCache<TKey, TValue> where TKey : class` (используя `WeakReference`) и сравните с обычным `Cache`. Подумайте, почему `class` здесь обязателен, а `notnull` недостаточен.

---

## Statement in English / Постановка на английском

#### Context & motivation
Imagine you are joining a team that maintains an in-house "generic utilities" library called `GenericToolkit`, shared across several company projects. The library already exists, but it was written naively: the author of the generic classes never applied any `where` constraints at all, so the code either fails to compile, or is vulnerable to `null`, or runs suboptimally because comparisons fall back to `object.Equals` and trigger boxing. Your job is to refactor the key components so that the compiler itself enforces the required type properties, and the code becomes both faster and safer.

The library consists of several independent components, each demonstrating one or several constraint kinds from the lesson. Specifically: `Repository<TEntity>` stores entities and creates them on demand — here you need `class`, the interface constraint `IEntity`, and `new()`. `Range<T>` represents a range of values (for numbers, dates, enums) — here you need `struct` together with `IComparable<T>`. `Factory<T>` produces objects with optional configuration — here `class, new()`. `Cache<TKey, TValue>` forbids nullable keys — `notnull`. `Vector2<T>` is allocation-free vector math — `struct, INumber<T>` and optionally `unmanaged`. On top of that, you implement a static `Sorter` with `IComparable<T>` and a small bonus using `enum`/`delegate` constraints.

The motivation for this assignment is twofold. First, in real-world code `where` constraints are not a theoretical ornament but a tool that shifts checks from run time to compile time: instead of `if (entity is null)` the compiler itself proves that `null` is impossible; instead of `as IEntity`, it proves the interface is implemented. Second, an excess of constraints kills reuse: if you slap `where TEntity : class, IEntity, new(), IComparable<IEntity>` onto `Repository`, half of your entities will no longer fit, even though comparison is never used inside the repository. The lesson formulates the guiding rule — "minimal sufficiency" — and this homework forces you to apply it deliberately on every component.

#### What to do step by step
1. Create a fresh .NET 8 console application: in a terminal run `dotnet new console -n GenericToolkit -o GenericToolkit --framework net8.0`, then `cd GenericToolkit` and open the project in your editor. Confirm that `GenericToolkit.csproj` contains `<Nullable>enable</Nullable>` and `<LangVersion>latest</LangVersion>` — if not, add them manually so that every C# 12 feature and nullable annotations are available.
2. Turn on treat-warnings-as-errors to make it serious: add `<TreatWarningsAsErrors>true</TreatWarningsAsErrors>` and `<Nullable>enable</Nullable>` to the `.csproj`. This forces you to clear every nullable warning — without that the acceptance criteria cannot be satisfied.
3. Create a file `GenericToolkit.cs` with namespace `GenericToolkit`. Put all of the assignment's classes there. Use top-level statements in `Program.cs` for the demonstration — the entry point should call methods of the classes and print results.
4. Implement an interface `IEntity` with a property `int Id { get; init; }`. Implement a sealed class `Product : IEntity` with properties `Id` and `Name` (string, init, default `""`). Implement a sealed class `Order : IEntity` with properties `Id` and `Total` (decimal). Both classes must have a public parameterless constructor so they satisfy the `new()` constraint.
5. Implement `Repository<TEntity> where TEntity : class, IEntity, new()` with a private `Dictionary<int, TEntity>` and methods: `Add(int id)` — creates `new TEntity { Id = id }` and stores it; `Find(int id)` — returns `TEntity?` via `TryGetValue`; `GetOrAdd(int id)` — returns the existing entry or creates, stores and returns a new one. Notice: thanks to `class` you may return `null`, thanks to `new()` you may instantiate, and thanks to `IEntity` you may assign `Id`.
6. Implement a generic `Range<T> where T : struct, IComparable<T>` with `Min` and `Max` readonly fields, a constructor that throws `ArgumentException` when `min.CompareTo(max) > 0`, a method `Contains(T value)` returning `min.CompareTo(value) <= 0 && value.CompareTo(max) <= 0`, and a `Clamp(T value)` method. Test it with `int`, `double`, and a custom `struct Point2D : IComparable<Point2D>` (compare by sum of coordinates). Confirm that `string` and `Nullable<int>` do NOT compile as `T` — this proves the `struct` constraint works.
7. Implement `Factory<T> where T : class, new()` with a method `Create(Action<T>? configure = null)` that creates `new T()`, optionally applies configuration and returns the instance. In the demo use `Factory<Product>` and `Factory<Order>`. Think about why `Factory<string>` fails to compile — `string` has no public parameterless constructor.
8. Implement `Cache<TKey, TValue> where TKey : notnull` with a private `Dictionary<TKey, TValue>`, methods `Set` and `Get` (via `TryGetValue`). Try in client code to pass a `string?` key that may be `null` — the compiler must emit a warning/error. This demonstrates the `notnull` constraint in action.
9. Implement `Vector2<T> where T : struct, INumber<T>` (use `System.Numerics.INumber<T>`) implementing `IEquatable<Vector2<T>>`, with methods `Add`, `Subtract`, `Scale(T factor)`, operators `+`, `-`, `==`, `!=` and overridden `Equals`/`GetHashCode` via `HashCode.Combine`. Override `ToString` for convenient debugging. Make the struct `readonly`. Demonstrate on `Vector2<int>` and `Vector2<double>`.
10. Implement a static class `Sorter` with methods `Max<T>(T a, T b) where T : IComparable<T>` and `Min<T>` analogously, plus `IsSorted<T>(IEnumerable<T> items) where T : IComparable<T>`. Compare its performance with a version that uses no constraint (via `object.Equals`) on an array of one million `int` — capture the difference with `Stopwatch` and print it to the console.
11. Bonus: implement `EnumHelper<T> where T : struct, Enum` with a method `GetValues()` returning `T[]` via `Enum.GetValues<T>()`, and a method `ParseInvariant(string name)`. Implement `DelegateInvoker<T> where T : Delegate` — a wrapper that stores a delegate and counts invocations. This showcases the special `enum` and `delegate` constraints.
12. In `Program.cs` via top-level statements call every component: repository, range, factory, cache, vector, sorter, bonus utilities. Print results with `Console.WriteLine`. Run the project with `dotnet run` and make sure there are no errors or warnings. Expected output should include, for instance: `Product id=1`, `Range contains 5: True`, `Factory created Order`, `Cache[answer]=42`, `Vector (4,6)`, `Max(3,7)=7`, `IsSorted: True`.

#### Requirements
The code must compile under C# 12 / .NET 8 with zero nullable-level warnings (given `TreatWarningsAsErrors=true`). Every `where` constraint must be minimally sufficient: do not add `class` or `new()` "just in case" — only when the code actually uses the corresponding operations (`null` comparison, `new T()`, access to interface members). The ordering of constraints in combined clauses must follow the lesson: first `class`/`struct`, then the base class or interfaces, and `new()` last. Different type parameters must get their own `where` clauses, one per parameter.

Every generic class must be documented with `///` XML comments explaining **why** each constraint exists (for example: "`class` — so `Find` may return `null`; `IEntity` — so `Id` can be assigned; `new()` — so instances can be created in `Add`"). Structs should be `readonly` where appropriate. For `Vector2<T>`, overriding `Equals`/`GetHashCode` and implementing `IEquatable<T>` is mandatory to avoid boxing. In `Repository.GetOrAdd` you must guarantee that the same `id` does not create a duplicate.

You must not use reflection (`Activator.CreateInstance`) where `new()` suffices — that defeats the purpose of the constraint. You must not use `object.Equals` or `object.CompareTo` where `IEquatable<T>`/`IComparable<T>` would do. All public methods accepting reference parameters must validate arguments via `ArgumentNullException.ThrowIfNull`. The project must run with `dotnet run` and print the expected results.

#### Pitfalls
The main pitfall is constraint ordering. If you place `new()` before the interfaces (`where T : new(), IEntity, class`), the compiler emits error CS0449 or similar: `new()` must come last. Likewise `class` and `struct` must come first and cannot be combined with each other — `where T : struct, class` is illegal by definition. Memorize the mnemonic: "reference/value, then base/interfaces, then constructor".

The second pitfall is that `where T : struct` excludes `Nullable<T>`. This means `Range<int?>` will not compile, which often surprises beginners who expect "struct means all value types, including nullable". In reality `Nullable<T>` is a special wrapper deliberately excluded from the `struct` constraint to avoid double-nullable semantics. If you need to accept nullable values, apply `where T : struct` to `T` itself and make the nullability live on method parameters rather than on the type.

The third pitfall is that `new()` does not work with types lacking a public parameterless constructor: `string`, all `enum`s, abstract classes, structs whose only constructors take arguments (and which have no explicit default). For these cases use a `static abstract` factory member on an interface (C# 11+) or accept a `Func<T>` explicitly.

The fourth pitfall is that `notnull` only takes effect under the nullable context (`<Nullable>enable</Nullable>`). Outside that context the `notnull` constraint effectively checks nothing. So make sure nullable annotations are on in the project. Note also that `class` without a question mark in a nullable context means "non-null reference type", while `class?` permits a nullable reference type. Do not confuse the two forms.

The fifth pitfall is performance. Using `object.Equals` on value types causes boxing and allocations; `IComparable<object>` is similarly slower than `IComparable<T>`. An interface constraint on `IEquatable<T>`/`IComparable<T>` is not just clearer — it produces a real speedup on hot paths. Capture that in a `Stopwatch` benchmark.

The sixth pitfall is that `unmanaged` implicitly implies `struct`, but not vice versa: a `struct` may hold reference fields (a string, for instance), while `unmanaged` may not. Use `unmanaged` deliberately, only when interop/Span/P/Invoke is genuinely needed, otherwise you narrow the set of acceptable types for nothing.

#### Acceptance criteria
- [ ] The `GenericToolkit` project is created with `dotnet new console` on .NET 8 and builds without errors.
- [ ] `.csproj` enables `<Nullable>enable</Nullable>`, `<LangVersion>latest</LangVersion>`, `<TreatWarningsAsErrors>true</TreatWarningsAsErrors>`.
- [ ] `Repository<TEntity>` has `class, IEntity, new()` in the correct order and uses all three (`null`, `Id`, `new TEntity()`).
- [ ] `Range<T>` has `struct, IComparable<T>`; the constructor throws `ArgumentException` when `min > max`.
- [ ] `Range<T>` does NOT compile with `string` or `int?` as the argument — demonstrated by a commented-out attempt.
- [ ] `Factory<T>` has `class, new()` and does NOT compile with `string` (no default ctor).
- [ ] `Cache<TKey, TValue>` uses `notnull` for `TKey` and the compiler emits an error/warning when a `null` key is passed.
- [ ] `Vector2<T>` has `struct, INumber<T>`, implements `IEquatable<Vector2<T>>`, overrides `Equals`/`GetHashCode`, supports operators.
- [ ] `Vector2<T>` is a `readonly struct` with no boxing on comparisons.
- [ ] `Sorter.Max/Min/IsSorted` use the `IComparable<T>` constraint and never call `object.Equals`.
- [ ] The `Stopwatch` benchmark shows a time difference between the `IComparable<T>` and `object.Equals` versions — the difference is captured in the output.
- [ ] The bonus `EnumHelper<T> where T : struct, Enum` compiles and returns enum values.
- [ ] The bonus `DelegateInvoker<T> where T : Delegate` compiles and counts invocations.
- [ ] Every generic class has XML comments explaining the reason for each constraint.
- [ ] All constraints are minimally sufficient — no redundant `class`/`new()` where the corresponding operations are unused.
- [ ] `dotnet run` prints the expected results with no warnings.

#### Hints
- To test `Range<string>`, simply try to construct `Range<string>` and comment it out — the compiler will tell you which restriction fired.
- `INumber<T>` lives in the `System.Numerics` namespace; add `using System.Numerics;`.
- `Enum.GetValues<T>()` (the generic overload) requires .NET 5+; it is available in .NET 8.
- To invoke a delegate of type `T` where `T : Delegate`, cast via `((Delegate)(object)del).DynamicInvoke(...)`, or constrain by a concrete delegate type. Think about which is safer.
- For `GetHashCode` over several fields, use `HashCode.Combine(a, b)`.
- `ArgumentNullException.ThrowIfNull` is available in .NET 6+ and removes boilerplate.
- If the compiler complains about `new TEntity { Id = id }`, check that `IEntity.Id` has `init`, not just `get`.

#### Reference solution walk-through
```csharp
// C# 12 / .NET 8 — Reference GenericToolkit
using System;
using System.Collections.Generic;
using System.Numerics;

namespace GenericToolkit;

// Entity interface
public interface IEntity
{
    int Id { get; init; }   // init-settable on construction
}

public sealed class Product : IEntity
{
    public int Id { get; init; }
    public string Name { get; init; } = "";
}

public sealed class Order : IEntity
{
    public int Id { get; init; }
    public decimal Total { get; init; }
}

/// <summary>
/// Repository of reference entities with an Id.
/// class — lets Find return null; IEntity — exposes Id; new() — allows new TEntity().
/// </summary>
public class Repository<TEntity> where TEntity : class, IEntity, new()
{
    private readonly Dictionary<int, TEntity> _store = new();

    public void Add(int id)
    {
        // new() + IEntity.Id (init) — both constraints exercised
        var entity = new TEntity { Id = id };
        _store[id] = entity;
    }

    public TEntity? Find(int id)                      // class → nullable return allowed
        => _store.TryGetValue(id, out var e) ? e : null;

    public TEntity GetOrAdd(int id)
    {
        if (_store.TryGetValue(id, out var existing))
            return existing;
        var created = new TEntity { Id = id };        // new() used
        _store[id] = created;
        return created;
    }
}

/// <summary>
/// Range of comparable value types.
/// struct — value types only (excludes string, Nullable<T>); IComparable<T> — boxing-free comparison.
/// </summary>
public readonly struct Range<T> where T : struct, IComparable<T>
{
    public T Min { get; }
    public T Max { get; }

    public Range(T min, T max)
    {
        if (min.CompareTo(max) > 0)
            throw new ArgumentException($"min {min} > max {max}", nameof(min));
        Min = min; Max = max;
    }

    public bool Contains(T value)
        => Min.CompareTo(value) <= 0 && value.CompareTo(Max) <= 0;

    public T Clamp(T value)
        => value.CompareTo(Min) < 0 ? Min
         : value.CompareTo(Max) > 0 ? Max
         : value;
}

/// <summary>
/// Factory for objects with a public parameterless ctor.
/// class, new() — the only constraints needed; no extra interfaces.
/// </summary>
public class Factory<T> where T : class, new()
{
    public T Create(Action<T>? configure = null)
    {
        var item = new T();                  // new() used
        configure?.Invoke(item);
        return item;
    }
}

/// <summary>
/// Cache with a non-null key under the nullable context.
/// notnull — forbids nullable reference keys; only meaningful with <Nullable>enable</Nullable>.
/// </summary>
public class Cache<TKey, TValue> where TKey : notnull
{
    private readonly Dictionary<TKey, TValue> _items = new();
    public void Set(TKey key, TValue value) => _items[key] = value;
    public TValue? Get(TKey key) => _items.TryGetValue(key, out var v) ? v : default;
}

/// <summary>
/// 2D vector over arbitrary numbers.
/// struct — value type, allocation-free; INumber<T> — arithmetic; IEquatable<Vector2<T>> — boxing-free equality.
/// </summary>
public readonly struct Vector2<T> : IEquatable<Vector2<T>>
    where T : struct, INumber<T>
{
    public T X { get; }
    public T Y { get; }
    public Vector2(T x, T y) { X = x; Y = y; }

    public Vector2<T> Add(Vector2<T> o) => new(X + o.X, Y + o.Y);
    public Vector2<T> Subtract(Vector2<T> o) => new(X - o.X, Y - o.Y);
    public Vector2<T> Scale(T f) => new(X * f, Y * f);

    public bool Equals(Vector2<T> o) => X == o.X && Y == o.Y;
    public override bool Equals(object? obj) => obj is Vector2<T> v && Equals(v);
    public override int GetHashCode() => HashCode.Combine(X, Y);
    public override string ToString() => $"({X}, {Y})";

    public static Vector2<T> operator +(Vector2<T> a, Vector2<T> b) => a.Add(b);
    public static Vector2<T> operator -(Vector2<T> a, Vector2<T> b) => a.Subtract(b);
    public static bool operator ==(Vector2<T> a, Vector2<T> b) => a.Equals(b);
    public static bool operator !=(Vector2<T> a, Vector2<T> b) => !a.Equals(b);
}

/// <summary>Comparisons via IComparable<T> only — no boxing.</summary>
public static class Sorter
{
    public static T Max<T>(T a, T b) where T : IComparable<T>
        => a.CompareTo(b) >= 0 ? a : b;
    public static T Min<T>(T a, T b) where T : IComparable<T>
        => a.CompareTo(b) <= 0 ? a : b;
    public static bool IsSorted<T>(IEnumerable<T> items) where T : IComparable<T>
    {
        var first = true;
        T prev = default!;
        foreach (var cur in items)
        {
            if (!first && prev.CompareTo(cur) > 0) return false;
            prev = cur; first = false;
        }
        return true;
    }
}

// Bonus: enum and delegate constraints
public static class EnumHelper<T> where T : struct, Enum
{
    public static T[] GetValues() => Enum.GetValues<T>();
    public static T ParseInvariant(string name) => Enum.Parse<T>(name, ignoreCase: false);
}

public sealed class DelegateInvoker<T> where T : Delegate
{
    private readonly T _delegate;
    public int CallCount { get; private set; }
    public DelegateInvoker(T del)
    {
        ArgumentNullException.ThrowIfNull(del);
        _delegate = del;
    }
    public object? Invoke(params object?[] args)
    {
        CallCount++;
        return _delegate.DynamicInvoke(args);
    }
}
```

**Line-by-line walk-through.** In `Repository<TEntity>` the constraint `class` comes first (mandatory by the ordering rule), then `IEntity` (an interface), then `new()` (mandatory last). All three constraints are **actually used** in the code: `class` — so `Find` may return `null` (a value type could never be `null`); `IEntity` — so `Id` can be assigned in `new TEntity { Id = id }`; `new()` — so `new TEntity()` compiles. If you removed any of them, the code would no longer compile — which means the constraints are genuinely minimally sufficient rather than "just in case". That is the central lesson concept.

In `Range<T>` the `struct` constraint guarantees that `T` is a value type, and `IComparable<T>` exposes `CompareTo` without boxing. The combination `struct, IComparable<T>` admits `int`, `double`, `DateTime`, custom structs, but rejects `string` (reference type) and `int?` (`Nullable<T>` excluded from `struct`). The constructor checks `min.CompareTo(max) > 0` and throws `ArgumentException` — this guards the type invariant at construction time, matching the best practice of "documenting type invariants".

In `Factory<T>` only `class, new()` is used — deliberately minimal. If we had added `IEntity`, the factory would stop working with arbitrary classes such as `StringBuilder`. In `Cache<TKey, TValue>` the `notnull` constraint, under the nullable context, forbids nullable reference keys — this protects the dictionary from a runtime `null` key by moving the check to compile time.

In `Vector2<T>` the `struct, INumber<T>` constraint from `System.Numerics` exposes the `+`, `-`, `*` operators via static abstract interface members (C# 11+). This is the modern replacement for the old "numeric constraint" that previously had to be emulated with delegates. Implementing `IEquatable<Vector2<T>>` and overriding `Equals(object?)` to delegate to `Equals(Vector2<T>)` guarantees that `==` and `Equals` comparisons do not box the struct — that is exactly the performance win for which interface constraints exist. `HashCode.Combine` is the recommended way to build a hash without manual arithmetic.

In `Sorter` the single constraint `IComparable<T>` carries no `class` or `new()` — because the method only needs to compare, nothing more. This is the illustration of the lesson's main rule: "minimally sufficient". If we had written `where T : class, IComparable<T>`, `int` would no longer fit, even though comparing integers is the primary use case. In `IsSorted` the `prev = default!` uses `!` to silence the nullable warning, since `prev` is provably assigned on the first iteration.

In the bonus `EnumHelper<T> where T : struct, Enum` and `DelegateInvoker<T> where T : Delegate` we show the special C# 7.3 constraints: they narrow the type to enums or delegates respectively. `Enum.GetValues<T>()` is the generic overload from .NET 5+ that removes the cast. In `DelegateInvoker`, `DynamicInvoke` is slow but safe for invoking an arbitrary delegate; in production, for a specific delegate type, it is better to constrain by that exact type (`Func<int,int>` and so on).

#### Going deeper (bonus)
1. Replace `new()` in `Repository` and `Factory` with an interface `IInitializable<T>` carrying `static abstract T Create();` (C# 11+). Compare the two approaches: which is better for types without a default constructor? Which constraints does this remove or add?
2. Implement a `SpanFriendlyBuffer<T> where T : unmanaged` with a fixed `Span<T>` buffer and indexed read/write methods. Compare its performance to `T[]` with `BenchmarkDotNet`. Capture the difference in nanoseconds.
3. Add a `where T : U` (conversion constraint) to a generic method `Converter.Convert<TSource, TTarget>(TSource source) where TSource : TTarget` and demonstrate a safe narrowing conversion without `as`/`is`.
4. Implement `WeakCache<TKey, TValue> where TKey : class` (using `WeakReference`) and compare it with the regular `Cache`. Think about why `class` is mandatory here and why `notnull` is not enough.

---

#### Чек-лист сдачи / Submission checklist
- [ ] Проект `GenericToolkit` собирается под .NET 8 без предупреждений (RU).
- [ ] Включены `<Nullable>enable</Nullable>`, `<LangVersion>latest</LangVersion>`, `<TreatWarningsAsErrors>`.
- [ ] Все семь компонентов реализованы: `Repository`, `Range`, `Factory`, `Cache`, `Vector2`, `Sorter`, бонус `EnumHelper`/`DelegateInvoker`.
- [ ] Ограничения минимально достаточны и упорядочены по правилу урока.
- [ ] XML-комментарии объясняют каждое ограничение.
- [ ] `dotnet run` выводит ожидаемые строки.
- [ ] Закомментированы попытки `Range<string>`, `Range<int?>`, `Factory<string>`, доказывающие действие ограничений.
- [ ] (EN) The `GenericToolkit` project builds on .NET 8 with no warnings.
- [ ] (EN) `<Nullable>enable</Nullable>`, `<LangVersion>latest</LangVersion>`, `<TreatWarningsAsErrors>` are set.
- [ ] (EN) All seven components are implemented: `Repository`, `Range`, `Factory`, `Cache`, `Vector2`, `Sorter`, plus bonus `EnumHelper`/`DelegateInvoker`.
- [ ] (EN) Constraints are minimally sufficient and ordered per the lesson rule.
- [ ] (EN) XML comments explain each constraint.
- [ ] (EN) `dotnet run` prints the expected lines.
- [ ] (EN) Commented-out attempts `Range<string>`, `Range<int?>`, `Factory<string>` are present to prove the constraints.

#### Ресурсы / Resources
- [Microsoft Learn — Constraints on type parameters — https://learn.microsoft.com/dotnet/csharp/language-reference/keywords/where-generic-type-constraint](https://learn.microsoft.com/dotnet/csharp/language-reference/keywords/where-generic-type-constraint)
- [Generics (C# Programming Guide) — https://learn.microsoft.com/dotnet/csharp/fundamentals/types/generics](https://learn.microsoft.com/dotnet/csharp/fundamentals/types/generics)
- [System.Numerics.INumber<T> — https://learn.microsoft.com/dotnet/api/system.numerics.inumber-1](https://learn.microsoft.com/dotnet/api/system.numerics.inumber-1)
- [Static abstract members in interfaces (C# 11) — https://learn.microsoft.com/dotnet/csharp/whats-new/csharp-11#static-abstract-members-in-interfaces](https://learn.microsoft.com/dotnet/csharp/whats-new/csharp-11#static-abstract-members-in-interfaces)
- [Enum.GetValues<T> — https://learn.microsoft.com/dotnet/api/system.enum.getvalues](https://learn.microsoft.com/dotnet/api/system.enum.getvalues)

---
[← К уроку M06-L03](lesson-M06-L03-constraints-where.md) | [⬆ К модулю M06](../README.md) | [Следующее ДЗ →](homework-M06-L04-dictionary-hashset.md)
