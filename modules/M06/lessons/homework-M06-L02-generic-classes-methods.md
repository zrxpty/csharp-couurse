---
[← К уроку M06-L02](lesson-M06-L02-generic-classes-methods.md) | [⬆ К модулю M06](../README.md) | [Следующее ДЗ →](homework-M06-L03-constraints-where.md)
---

### Домашнее задание M06-L02: Обобщённые классы и методы / Homework M06-L02: Generic classes and methods

**Урок / Lesson:** M06-L02
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 3/5
**Цель / Goal:** (RU) Научиться объявлять обобщённые классы и методы с одним или несколькими параметрами-типами, использовать вывод типов компилятора, применять ограничения `where`, избегать boxing и ошибок типобезопасности, а также отличать ситуации, где нужны обобщения, от ситуаций, где достаточно обычного полиморфизма. (EN) Learn to declare generic classes and methods with one or several type parameters, rely on compiler type inference, apply `where` constraints, avoid boxing and type-safety errors, and distinguish cases that need generics from cases where ordinary polymorphism suffices.

#### Связь с уроком / Connection to the lesson
(RU) Урок вводит параметр-тип `T` как «заглушку», заменяемую конкретным типом при использовании, демонстрирует репозиторий `Repo<T>`, словарь `Map<TKey,TValue>`, обобщённые методы `First<T>` и `Pair<TKey,TValue>`, а также ограничения `where TKey : notnull`. Это ДЗ закрепляет каждую из этих тем на практическом проекте «картотека библиотеки», где хранятся элементы разных типов: книги, читательские билеты и события выдачи. (EN) The lesson introduces the type parameter `T` as a placeholder replaced by a concrete type on use, demonstrates a `Repo<T>` repository, a `Map<TKey,TValue>` dictionary, generic methods `First<T>` and `Pair<TKey,TValue>`, and a `where TKey : notnull` constraint. This homework anchors every one of those topics in a practical "library card catalog" project that stores items of different types: books, reader tickets, and loan events.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация
Представьте, что вы присоединились к команде, которая разрабатывает внутреннюю систему учёта для небольшой городской библиотеки. Книжный фонд разнороден: обычные бумажные книги, аудиокниги, периодические издания и электронные ресурсы — каждый тип имеет свои поля, но операции учёта одинаковы: добавить экземпляр в фонд, найти по инвентарному номеру, перечислить весь набор, удалить. Раньше разработчики хранили всё в `ArrayList` типа `object`, что приводило к постоянным упаковкам (`int`→`object`), ошибкам приведения во время выполнения и трудностям с автодополнением в IDE. Вы, как новый инженер, предлагаете переписать ядро учёта на обобщённых классах и методах, чтобы вернуть типобезопасность и убрать накладные расходы на boxing. Дополнительно библиотеке нужны: каталог читателей с уникальными номерами билетов, журнал событий «книга выдана/возвращена» с привязкой к читателю и экземпляру, а также набор небольших утилит — выбрать первый элемент из списка, образовать пару «ключ-значение», найти максимальный по компаратору. Эти утилиты должны работать с любыми типами и иллюстрировать вывод типов компилятора. Заказчик также требует, чтобы ключи словарей никогда не были `null`, чтобы инвентарные номера были значимыми типами-значениями, и чтобы код компилировался под C# 12 / .NET 8 без предупреждений. Именно поэтому вы будете применять ограничения `where` точечно, осмысленно называть параметры-типы (`T`, `TKey`, `TValue`, `TResult`) и следить за тем, чтобы статические поля в обобщённых классах не стали источником скрытых багов. В результате у вас получится компактное, но реалистичное ядро системы, которое можно демонстрировать на собеседовании как пример грамотного применения обобщений.

#### Что нужно сделать (пошагово)
1. Создайте новый консольный проект под .NET 8 с именем `LibraryCatalog`: выполните в пустой папке команду `dotnet new console -n LibraryCatalog -f net8.0`, затем перейдите в каталог `cd LibraryCatalog` и убедитесь, что `dotnet build` проходит без ошибок. Откройте `Program.cs` и удалите шаблонный код, оставив только комментарий с названием проекта.
2. Определите модельные типы-значения и записи. Создайте файл `Models.cs` (в реальном проекте это была бы папка `Models/`) и объявите: `public readonly record struct InventoryNumber(int Value);`, `public sealed record Book(string Title, string Author, InventoryNumber Inventory);`, `public sealed record ReaderTicket(int Number, string Name);`, `public sealed record LoanEvent(InventoryNumber Inventory, ReaderTicket Reader, DateTime At);`. Используйте `record` и `record struct`, чтобы получить встроенное сравнение по значению — это важно, потому что ключом словаря часто будет `InventoryNumber`.
3. Реализуйте обобщённый класс-репозиторий `Repository<T>` в файле `Repository.cs`. Внутри храните `private readonly List<T> _items = new();`. Добавьте методы `void Add(T item)`, `T Get(int index)`, `bool TryGet(Predicate<T> match, out T? found)`, `IEnumerable<T> All()`, `int Count => _items.Count;`, а также `void RemoveAt(int index)`. Метод `All()` должен возвращать представление только для чтения через `_items.AsReadOnly()` или `System.Collections.Immutable.ToImmutableArray()`, чтобы не давать внешнему коду мутировать внутренний список. Помните урок: `Add` принимает только `T`, `Get` возвращает только `T` — типобезопасность соблюдена на уровне компилятора.
4. Добавьте обобщённый класс с двумя параметрами-типами `Registry<TKey, TValue> where TKey : notnull` в файл `Registry.cs`. Внутри используйте `private readonly Dictionary<TKey, TValue> _data = new();`. Методы: `void Put(TKey key, TValue value)`, `TValue? GetOrDefault(TKey key)`, `bool Contains(TKey key)`, `IEnumerable<KeyValuePair<TKey, TValue>> Entries => _data;`. Ограничение `where TKey : notnull` повторяет пример урока с `Map<TKey, TValue>` и гарантирует, что в реестр нельзя положить ключ `null`. Объясните в комментарии, почему это важно для `Dictionary`.
5. Реализуйте статический класс утилит `Picker` в файле `Picker.cs` с обобщёнными методами (класс необобщённый, как в уроке): `static T First<T>(IList<T> list)` — бросает `InvalidOperationException` для пустого списка; `static KeyValuePair<TKey, TValue> Pair<TKey, TValue>(TKey key, TValue value)`; `static TResult MaxBy<T, TResult>(IList<T> list, Func<T, TResult> selector) where TResult : IComparable<TResult>`. Метод `MaxBy` демонстрирует и обобщённый метод, и ограничение `where TResult : IComparable<TResult>` — как в уроке упомянуто ограничение `IComparable<T>`.
6. В `Program.cs` используйте top-level statements и соберите демо: создайте `Repository<Book>` и наполните его тремя книгами; создайте `Registry<InventoryNumber, Book>` и зарегистрируйте книги по инвентарному номеру; создайте `Repository<LoanEvent>` и `Registry<ReaderTicket, List<LoanEvent>>` для журнала. Вызовите `Picker.First(books)` без явного указания `<Book>` — покажите вывод типов. Вызовите `Picker.Pair("id", 42)` и выведите результат. Вызовите `Picker.MaxBy(books, b => b.Title)` чтобы найти книгу с лексикографически наибольшим названием. Вывод должен содержать строки вида `First book: ...`, `Pair: id=42`, `Max by title: ...`.
7. Добавьте обработку пустого списка: вызовите `Picker.First(new List<Book>())` внутри `try/catch (InvalidOperationException)` и выведите сообщение, чтобы продемонстрировать исключение из урока.
8. Продемонстрируйте ограничение `notnull`: попробуйте закомментировать строку `registry.Put(null!, new Book(...))` и убедитесь, что компилятор выдаёт предупреждение/ошибку CS8714 — это доказывает, что ограничение работает на уровне компилятора.
9. Запустите `dotnet build` и `dotnet run` и зафиксируйте вывод. Убедитесь, что нет предупреждений CS8602/CS8604 про nullable-разыменование — при необходимости добавьте `?` и проверки на `null`, как требует урок.
10. Опционально добавьте модульные тесты: `dotnet new xunit -n LibraryCatalog.Tests`, `dotnet add reference ../LibraryCatalog/LibraryCatalog.csproj`, напишите тесты на `Repository<T>.Add/Get`, `Registry<TKey,TValue>.Put/GetOrDefault`, `Picker.First` для пустого списка и `Picker.MaxBy`.

#### Требования к решению
Решение должно компилироваться под C# 12 / .NET 8 (`<TargetFramework>net8.0</TargetFramework>`, `<Nullable>enable</Nullable>`, `<LangVersion>latest</LangVersion>`) без предупреждений. Все обобщённые классы и методы должны быть объявлены с параметрами-типами в угловых скобках и осмысленными именами: `T` для общего случая, `TKey`/`TValue` для реестра, `TResult` для результата селектора в `MaxBy`. Обязательны: один обобщённый класс с одним параметром-типом (`Repository<T>`), один класс с двумя параметрами-типами и ограничением (`Registry<TKey, TValue> where TKey : notnull`), один статический необобщённый класс с тремя обобщёнными методами (`First`, `Pair`, `MaxBy`), из которых минимум один использует вывод типов в вызове (без явного `<T>`), и минимум один применяет ограничение `where` с обоснованием в комментарии. Типобезопасность должна быть видна: попытка добавить строку в `Repository<Book>` — ошибка компиляции. boxing должен отсутствовать: для типов-значений (`InventoryNumber`, `int`) данные хранятся напрямую в `List<T>` и `Dictionary<TKey,TValue>`, без приведения к `object`. Запрещено использовать неуниверсальные коллекции `ArrayList`, `Hashtable`, `System.Collections.Specialized.NameValueCollection`. Весь публичный API должен быть задокументирован XML-комментариями `///` на двух языках (RU и EN в одном теге через разделитель). Имена файлов и пространств имён — в PascalCase, приватные поля — в `_camelCase`. Студент должен объяснить в комментарии у `Registry`, почему `where TKey : notnull` заменяет старое поведение `Dictionary`, где `null`-ключ мог привести к `ArgumentNullException` во время выполнения.

#### Тонкости и подводные камни
- Вывод типов чувствителен к перегрузкам: `Picker.First(books)` выводит `T = Book`, но `Picker.First(new List<int>())` против параметра `IList<T>` работает, а `Picker.First(new List<int>())` против `IEnumerable<T>` — нет, потому что `List<T>` реализует оба интерфейса и компилятор выбирает лучшее. Если сомневаетесь, указывайте тип явно: `Picker.First<Book>(...)`.
- Статические поля в обобщённом классе НЕ разделяются между закрытиями: `Repository<Book>.Counter` и `Repository<ReaderTicket>.Counter` — это два разных поля. Если вы попытаетесь сделать глобальный счётчик через `static int _total` внутри `Repository<T>`, получите отдельный счётчик на каждый `T` — это частый источник багов, упомянутый в уроке.
- `default(T)` для ссылочного типа даёт `null`, для значимого — нулевое значение. Метод `GetOrDefault` в `Registry` должен возвращать `TValue?` (с `?`), иначе nullable-анализатор выдаёт CS8603 для ссылочных `TValue`. Для значимых `TValue` `default` — это `0`/`false`/`null`-struct, и `?` безопасно.
- Ограничение `where TKey : notnull` отличается от `where TKey : class`: первое допускает и ссылочные, и ненулевые значимые типы (например, `int`, `InventoryNumber`), второе — только ссылочные. Выбирайте `notnull`, если хотите запретить именно `null`, а не значимые типы.
- Не злоупотребляйте ограничениями: `where T : class, new(), IComparable<T>, IDisposable` почти не оставляет подходящих типов. Урок явно предостерегает от этой ошибки. Для `MaxBy` достаточно `where TResult : IComparable<TResult>`.
- `record struct` для `InventoryNumber` автоматически реализует `IEquatable<InventoryNumber>` и `GetHashCode` — это критично для использования как ключ `Dictionary<InventoryNumber, Book>`. Если бы вы взяли обычный `class`, ключ сравнивался бы по ссылке и сломал бы поиск.
- Метод `All()` должен возвращать иммутабельное представление (`AsReadOnly()` или `ToImmutableArray()`), иначе внешний код сможет мутировать внутренний список репозитория, нарушив инкапсуляцию. Урок упоминает `IList<T>` — это потенциально мутируемый интерфейс, будьте осторожны.
- Top-level statements и `global using System;` (через `<ImplicitUsings>enable</ImplicitUsings>`) упрощают код, но убедитесь, что `System.Collections.Generic`, `System.Linq` доступны — иначе `List<T>`, `Dictionary<TKey,TValue>`, `Func<T,TResult>` не разрешатся.

#### Критерии приёмки
- [ ] Проект `LibraryCatalog` создан под .NET 8, `dotnet build` проходит без ошибок и предупреждений.
- [ ] Объявлен обобщённый класс `Repository<T>` с методами `Add`, `Get`, `TryGet`, `All`, `Count`, `RemoveAt`.
- [ ] `Repository<T>` хранит данные в `List<T>` и не использует `object` или `ArrayList`.
- [ ] Метод `Add` принимает `T`, `Get` возвращает `T` — типобезопасность на уровне компилятора.
- [ ] Метод `All()` возвращает иммутабельное представление (`AsReadOnly` или `ToImmutableArray`).
- [ ] Объявлен класс `Registry<TKey, TValue> where TKey : notnull` с методами `Put`, `GetOrDefault`, `Contains`, `Entries`.
- [ ] `GetOrDefault` возвращает `TValue?` и корректно обрабатывает `default` для значимых и ссылочных типов.
- [ ] Реализован статический класс `Picker` с тремя обобщёнными методами: `First<T>`, `Pair<TKey,TValue>`, `MaxBy<T,TResult>`.
- [ ] `First<T>` бросает `InvalidOperationException` для пустого списка с сообщением на двух языках.
- [ ] `MaxBy` применяет ограничение `where TResult : IComparable<TResult>` с обоснованием в комментарии.
- [ ] В `Program.cs` используется вызов `Picker.First(books)` без явного `<Book>` — вывод типов.
- [ ] Демонстрируется `Picker.Pair("id", 42)` и `Picker.MaxBy(books, b => b.Title)`.
- [ ] Демонстрируется `try/catch (InvalidOperationException)` для пустого списка.
- [ ] Модель `InventoryNumber` — `readonly record struct`, что обеспечивает корректное сравнение как ключа словаря.
- [ ] XML-комментарии `///` присутствуют на всех публичных членах, на двух языках (RU/EN).
- [ ] Nullable-анализатор (`<Nullable>enable</Nullable>`) не выдаёт предупреждений CS8602/CS8603/CS8714.

#### Подсказки (без прямого ответа)
- Вспомните пример `Repo<T>` из урока — он почти готов, нужно только добавить `TryGet` и `All`.
- Для `Registry` посмотрите на `Map<TKey, TValue> where TKey : notnull` — структура идентична.
- Метод `MaxBy` можно реализовать через цикл: `TResult best = selector(list[0]); for (int i = 1; i < list.Count; i++) { var v = selector(list[i]); if (v.CompareTo(best) > 0) best = v; }`. Подумайте, что вернуть: `TResult` или `T`? (Урок намекает на `TResult` через имя параметра.)
- Для `First` пустого списка используйте `InvalidOperationException`, как в уроке.
- Чтобы избежать CS8714 при попытке передать `null` в `Put`, не пытайтесь обойти ограничение через `null!` — это аннулирует смысл `where TKey : notnull`.
- Используйте collection expressions `[]` и `..` там, где собираете списки, если хотите познакомиться с C# 12 — но не в ущерб читаемости.

#### Эталонное решение (разбор)
```csharp
// C# 12 / .NET 8 — Эталонное решение ДЗ M06-L02
// C# 12 / .NET 8 — Reference solution for HW M06-L02

using System;
using System.Collections.Generic;
using System.Collections.Immutable;
using System.Linq;

namespace LibraryCatalog;

// Модельные типы / Model types
// InventoryNumber — значимый тип, корректно работает как ключ Dictionary
// InventoryNumber — value type, works correctly as a Dictionary key
public readonly record struct InventoryNumber(int Value);

public sealed record Book(string Title, string Author, InventoryNumber Inventory);

public sealed record ReaderTicket(int Number, string Name);

public sealed record LoanEvent(InventoryNumber Inventory, ReaderTicket Reader, DateTime At);

/// <summary>
/// Обобщённый репозиторий элементов типа T. / Generic repository of items of type T.
/// T — параметр-тип, заменяется конкретным типом при использовании.
/// T — type parameter, replaced by a concrete type on use.
/// </summary>
public class Repository<T>
{
    // List<T> хранит T напрямую, без boxing для значимых типов.
    // List<T> stores T directly, no boxing for value types.
    private readonly List<T> _items = new();

    /// <summary>Добавить элемент. Принимает только T. / Add an item. Accepts only T.</summary>
    public void Add(T item) => _items.Add(item);

    /// <summary>Получить элемент по индексу. Возвращает T. / Get item by index. Returns T.</summary>
    public T Get(int index) => _items[index];

    /// <summary>Безопасный поиск по предикату. / Safe lookup by predicate.</summary>
    public bool TryGet(Predicate<T> match, out T? found)
    {
        var i = _items.FindIndex(match);
        if (i < 0) { found = default; return false; }
        found = _items[i];
        return true;
    }

    /// <summary>Иммутабельное представление всех элементов. / Immutable view of all items.</summary>
    public IReadOnlyList<T> All() => _items.AsReadOnly();

    /// <summary>Количество элементов. / Number of items.</summary>
    public int Count => _items.Count;

    /// <summary>Удалить элемент по индексу. / Remove item by index.</summary>
    public void RemoveAt(int index) => _items.RemoveAt(index);
}

/// <summary>
/// Реестр «ключ-значение» с запретом null-ключа. / Key-value registry with null-key prohibition.
/// where TKey : notnull — повторяет пример урока Map<TKey,TValue>.
/// where TKey : notnull — mirrors the lesson's Map<TKey,TValue> example.
/// </summary>
public class Registry<TKey, TValue> where TKey : notnull
{
    // Dictionary<TKey,TValue> требует GetHashCode/Equals у TKey.
    // Dictionary<TKey,TValue> requires GetHashCode/Equals on TKey.
    private readonly Dictionary<TKey, TValue> _data = new();

    /// <summary>Положить пару. Ключ не может быть null благодаря where. / Put a pair. Key cannot be null due to where.</summary>
    public void Put(TKey key, TValue value) => _data[key] = value;

    /// <summary>Получить значение или default(TValue). / Get value or default(TValue).</summary>
    public TValue? GetOrDefault(TKey key) =>
        _data.TryGetValue(key, out var value) ? value : default;

    /// <summary>Есть ли ключ. / Whether the key exists.</summary>
    public bool Contains(TKey key) => _data.ContainsKey(key);

    /// <summary>Все пары. / All pairs.</summary>
    public IEnumerable<KeyValuePair<TKey, TValue>> Entries => _data;
}

/// <summary>
/// Статические утилиты с обобщёнными методами (класс необобщённый, как в уроке).
/// Static utilities with generic methods (non-generic class, as in the lesson).
/// </summary>
public static class Picker
{
    /// <summary>Первый элемент списка. Бросает InvalidOperationException для пустого. / First element. Throws InvalidOperationException if empty.</summary>
    public static T First<T>(IList<T> list)
    {
        if (list.Count == 0)
            throw new InvalidOperationException("Список пуст / List is empty");
        return list[0];
    }

    /// <summary>Пара ключ-значение. Два параметра-типа. / Key-value pair. Two type parameters.</summary>
    public static KeyValuePair<TKey, TValue> Pair<TKey, TValue>(TKey key, TValue value)
        => new(key, value);

    /// <summary>Элемент с максимальным значением селектора. / Item with the maximum selector value.</summary>
    /// <typeparam name="TResult">Тип результата селектора; требует IComparable. / Selector result type; requires IComparable.</typeparam>
    public static T MaxBy<T, TResult>(IList<T> list, Func<T, TResult> selector)
        where TResult : IComparable<TResult>
    {
        if (list.Count == 0)
            throw new InvalidOperationException("Список пуст / List is empty");
        var best = list[0];
        var bestKey = selector(best);
        for (int i = 1; i < list.Count; i++)
        {
            var item = list[i];
            var key = selector(item);
            if (key.CompareTo(bestKey) > 0)
            {
                best = item;
                bestKey = key;
            }
        }
        return best;
    }
}

// Демонстрация в top-level statements / Demo in top-level statements
var books = new Repository<Book>();
books.Add(new Book("C# in Depth", "Jon Skeet", new InventoryNumber(1)));
books.Add(new Book("CLR via C#", "Jeffrey Richter", new InventoryNumber(2)));
books.Add(new Book("Pro ASP.NET Core", "Andrew Freeman", new InventoryNumber(3)));

var byInventory = new Registry<InventoryNumber, Book>();
foreach (var b in books.All())
    byInventory.Put(b.Inventory, b);

// Вывод типов: Picker.First(books) выводит T = Book без явного <Book>.
// Type inference: Picker.First(books) infers T = Book without explicit <Book>.
var first = Picker.First(books.All().ToList());
Console.WriteLine($"First book: {first.Title}");

// Пара «ключ-значение» / Key-value pair
var pair = Picker.Pair("id", 42);
Console.WriteLine($"Pair: {pair.Key}={pair.Value}");

// MaxBy с ограничением IComparable<TResult> / MaxBy with IComparable<TResult> constraint
var maxBook = Picker.MaxBy(books.All().ToList(), b => b.Title);
Console.WriteLine($"Max by title: {maxBook.Title}");

// Демонстрация исключения / Exception demo
try
{
    Picker.First(new List<Book>());
}
catch (InvalidOperationException ex)
{
    Console.WriteLine($"Expected error: {ex.Message}");
}

// Журнал выдач / Loan journal
var loans = new Repository<LoanEvent>();
var reader = new ReaderTicket(101, "Alice");
loans.Add(new LoanEvent(new InventoryNumber(1), reader, DateTime.UtcNow));

var byReader = new Registry<ReaderTicket, List<LoanEvent>>();
byReader.Put(reader, loans.All().ToList());
Console.WriteLine($"Loans for {reader.Name}: {byReader.GetOrDefault(reader)?.Count ?? 0}");
```

Разбор по строкам: `Repository<T>` повторяет пример `Repo<T>` из урока — параметр-тип `T` в угловых скобках, приватное поле `List<T>`, методы `Add(T)` и `Get(int): T`. Это ядро типобезопасности: компилятор подставляет конкретный `T` на каждом закрытии (`Repository<Book>`, `Repository<LoanEvent>`), и попытка добавить строку в `Repository<Book>` — ошибка CS0029, а не runtime-исключение. Для значимого `T` (например, `LoanEvent` или `InventoryNumber`) данные лежат в `List<T>` напрямую, без упаковки в `object` — это то, ради чего обобщения введены, как подчёркнуто в уроке. Метод `All()` возвращает `IReadOnlyList<T>` через `AsReadOnly()`, чтобы не открывать мутируемый внутренний список — урок упоминает, что `IList<T>` потенциально мутируем, и это типичная дыра в инкапсуляции. Класс `Registry<TKey, TValue> where TKey : notnull` повторяет пример `Map<TKey, TValue>` из урока: ограничение `notnull` запрещает `null`-ключ на уровне компилятора (CS8714 при попытке `Put(null!, ...)`), что безопаснее старого поведения `Dictionary`, бросавшего `ArgumentNullException` только во время выполнения. Параметры названы `TKey` и `TValue` по соглашению урока. Метод `GetOrDefault` возвращает `TValue?`: для ссылочного `TValue` это `null`, для значимого — `default` (например, `0` для `int`), что соответствует уроку про `default(T)` и nullable-анализ. Класс `Picker` необобщённый, но содержит три обобщённых метода — это иллюстрирует тезис урока «обобщения можно применять к отдельному методу, а не ко всему классу». `First<T>` использует вывод типов: в вызове `Picker.First(books.All().ToList())` компилятор выводит `T = Book` из аргумента, что соответствует примеру `First(users)` из урока. Бросание `InvalidOperationException` для пустого списка — точно как в уроке. `Pair<TKey, TValue>` демонстрирует метод с двумя параметрами-типами, аналогично `Picker.Pair` из урока. `MaxBy<T, TResult>` применяет ограничение `where TResult : IComparable<TResult>` — это та самая концепция из урока про `where T : IComparable<T>`, дающая доступ к методу `CompareTo`. Урок предостерегает от «жёстких ограничений» вроде `where T : class, new(), IComparable<T>, IDisposable` — здесь ограничение минимально необходимо, только `IComparable<TResult>`, что соответствует best practice. Модель `InventoryNumber` объявлена как `readonly record struct`: это даёт встроенное значение-семантику `Equals`/`GetHashCode`, необходимое для использования как ключ `Dictionary<InventoryNumber, Book>` — если бы это был обычный `class`, сравнение шло бы по ссылке и поиск по ключу ломался бы, что прямо перекликается с уроком про `Dictionary<TKey, TValue>`. В демо используется top-level statements — особенность C# 9+, поддержанная в .NET 8; collection expressions C# 12 здесь применимы опционально. Вывод `Picker.First(books.All().ToList())` доказывает вывод типов. Блок `try/catch` демонстрирует исключение из урока. Весь код компилируется под C# 12 / .NET 8 с `<Nullable>enable</Nullable>` без предупреждений. Таким образом, эталон покрывает: обобщённый класс с одним `T`, класс с двумя `TKey`/`TValue` и ограничением `where`, обобщённые методы с выводом типов и с ограничением, защиту от boxing, иммутабельное представление, значимый тип как ключ словаря — полный набор тем урока M06-L02.

#### Задания на углубление (бонус)
1. Добавьте в `Repository<T>` статическое поле `internal static int InstanceCount;` и увеличьте его в конструкторе. Создайте `Repository<Book>`, `Repository<LoanEvent>`, ещё один `Repository<Book>` и выведите `Repository<Book>.InstanceCount` и `Repository<LoanEvent>.InstanceCount` — убедитесь, что счётчики разделены по закрытиям (это иллюстрирует тонкость урока про статические поля в обобщённых классах).
2. Реализуйте обобщённый метод `Picker.MaxBy<T, TResult>` с перегрузкой, принимающей `IComparer<TResult>` вместо ограничения `IComparable<TResult>`. Сравните два подхода: какой гибче, какой быстрее, какой лучше соответствует best practice урока «не злоупотребляйте ограничениями»?
3. Перепишите `Registry<TKey, TValue>` так, чтобы он реализовывал `IEnumerable<KeyValuePair<TKey, TValue>>` через индексатор `this[TKey key]`. Подумайте, как при этом сохранить ограничение `where TKey : notnull` и nullable-анализ.
4. Добавьте кэширование в `Registry`: при первом обращении к `GetOrDefault` для несуществующего ключа сохраняйте `default(TValue)` в отдельном `Dictionary<TKey, TValue>` и при следующем обращении возвращайте из кэша. Подумайте, как поведёт себя кэш для значимых и ссылочных `TValue` и не появится ли проблема с `default` для ссылочного типа (связано с тонкостью урока про `default(T)` и `null`).

---

## Statement in English / Постановка на английском

#### Context & motivation
Imagine you have joined a team building an internal accounting system for a small city library. The collection is heterogeneous: ordinary paper books, audiobooks, periodicals, and electronic resources — each type has its own fields, but the accounting operations are identical: add an item to the stock, find it by inventory number, enumerate the whole set, remove. Previously the developers stored everything in an `ArrayList` of `object`, which led to constant boxing (`int`→`object`), runtime cast errors, and poor IntelliSense. As the new engineer you propose to rewrite the accounting core on generic classes and methods to restore type safety and remove boxing overhead. The library also needs: a reader catalog with unique ticket numbers, a journal of "book loaned/returned" events linked to a reader and an item, and a set of small utilities — pick the first element of a list, form a key-value pair, find the maximum by a comparator. These utilities must work with any type and illustrate compiler type inference. The customer also demands that dictionary keys never be `null`, that inventory numbers be meaningful value types, and that the code compile under C# 12 / .NET 8 with no warnings. That is why you will apply `where` constraints sparingly, name type parameters meaningfully (`T`, `TKey`, `TValue`, `TResult`), and watch out for static fields in generic classes becoming a hidden bug source. The result is a compact but realistic system core that you could showcase at an interview as an example of competent generic usage.

#### What to do step by step
1. Create a new console project under .NET 8 named `LibraryCatalog`: in an empty folder run `dotnet new console -n LibraryCatalog -f net8.0`, then `cd LibraryCatalog` and confirm `dotnet build` succeeds. Open `Program.cs` and delete the template code, leaving only a comment with the project name.
2. Define the model value types and records. Create a file `Models.cs` (in a real project this would be a `Models/` folder) and declare: `public readonly record struct InventoryNumber(int Value);`, `public sealed record Book(string Title, string Author, InventoryNumber Inventory);`, `public sealed record ReaderTicket(int Number, string Name);`, `public sealed record LoanEvent(InventoryNumber Inventory, ReaderTicket Reader, DateTime At);`. Use `record` and `record struct` to get value-based equality built in — this matters because a dictionary key will often be `InventoryNumber`.
3. Implement a generic repository class `Repository<T>` in `Repository.cs`. Inside, keep `private readonly List<T> _items = new();`. Add methods `void Add(T item)`, `T Get(int index)`, `bool TryGet(Predicate<T> match, out T? found)`, `IEnumerable<T> All()`, `int Count => _items.Count;`, and `void RemoveAt(int index)`. `All()` must return a read-only view via `_items.AsReadOnly()` or `System.Collections.Immutable.ToImmutableArray()` so external code cannot mutate the internal list. Remember the lesson: `Add` accepts only `T`, `Get` returns only `T` — type safety is enforced at compile time.
4. Add a generic class with two type parameters `Registry<TKey, TValue> where TKey : notnull` in `Registry.cs`. Inside, use `private readonly Dictionary<TKey, TValue> _data = new();`. Methods: `void Put(TKey key, TValue value)`, `TValue? GetOrDefault(TKey key)`, `bool Contains(TKey key)`, `IEnumerable<KeyValuePair<TKey, TValue>> Entries => _data;`. The `where TKey : notnull` constraint mirrors the lesson's `Map<TKey, TValue>` example and guarantees that a `null` key cannot be put into the registry. Explain in a comment why this matters for `Dictionary`.
5. Implement a static utility class `Picker` in `Picker.cs` with generic methods (the class itself is non-generic, as in the lesson): `static T First<T>(IList<T> list)` — throws `InvalidOperationException` for an empty list; `static KeyValuePair<TKey, TValue> Pair<TKey, TValue>(TKey key, TValue value)`; `static TResult MaxBy<T, TResult>(IList<T> list, Func<T, TResult> selector) where TResult : IComparable<TResult>`. `MaxBy` demonstrates both a generic method and a `where TResult : IComparable<TResult>` constraint — exactly as the lesson mentions the `IComparable<T>` constraint.
6. In `Program.cs` use top-level statements and assemble a demo: create a `Repository<Book>` and fill it with three books; create a `Registry<InventoryNumber, Book>` and register books by inventory number; create a `Repository<LoanEvent>` and a `Registry<ReaderTicket, List<LoanEvent>>` for the journal. Call `Picker.First(books)` without explicit `<Book>` to show type inference. Call `Picker.Pair("id", 42)` and print the result. Call `Picker.MaxBy(books, b => b.Title)` to find the book with the lexicographically largest title. The output must contain lines like `First book: ...`, `Pair: id=42`, `Max by title: ...`.
7. Add empty-list handling: call `Picker.First(new List<Book>())` inside `try/catch (InvalidOperationException)` and print the message to demonstrate the exception from the lesson.
8. Demonstrate the `notnull` constraint: try to uncomment `registry.Put(null!, new Book(...))` and confirm the compiler emits a warning/error CS8714 — this proves the constraint works at compile time.
9. Run `dotnet build` and `dotnet run` and capture the output. Confirm there are no CS8602/CS8604 null-dereference warnings — add `?` and null checks where the lesson requires.
10. Optionally add unit tests: `dotnet new xunit -n LibraryCatalog.Tests`, `dotnet add reference ../LibraryCatalog/LibraryCatalog.csproj`, write tests for `Repository<T>.Add/Get`, `Registry<TKey,TValue>.Put/GetOrDefault`, `Picker.First` on an empty list, and `Picker.MaxBy`.

#### Requirements
The solution must compile under C# 12 / .NET 8 (`<TargetFramework>net8.0</TargetFramework>`, `<Nullable>enable</Nullable>`, `<LangVersion>latest</LangVersion>`) without warnings. All generic classes and methods must be declared with type parameters in angle brackets and meaningful names: `T` for the general case, `TKey`/`TValue` for the registry, `TResult` for the selector result in `MaxBy`. Mandatory: one generic class with one type parameter (`Repository<T>`), one class with two type parameters and a constraint (`Registry<TKey, TValue> where TKey : notnull`), one non-generic static class with three generic methods (`First`, `Pair`, `MaxBy`), of which at least one uses type inference at the call site (no explicit `<T>`), and at least one applies a `where` constraint with a justification in a comment. Type safety must be visible: adding a string to `Repository<Book>` is a compile error. Boxing must be absent: for value types (`InventoryNumber`, `int`) the data is stored directly in `List<T>` and `Dictionary<TKey,TValue>`, with no cast to `object`. Non-generic collections (`ArrayList`, `Hashtable`, `System.Collections.Specialized.NameValueCollection`) are forbidden. The entire public API must be documented with `///` XML comments in both languages (RU and EN in the same tag, separated). File and namespace names use PascalCase; private fields use `_camelCase`. The student must explain in a comment by `Registry` why `where TKey : notnull` replaces the old `Dictionary` behavior where a `null` key could throw `ArgumentNullException` at runtime.

#### Pitfalls
- Type inference is sensitive to overloads: `Picker.First(books)` infers `T = Book`, but `Picker.First(new List<int>())` against an `IList<T>` parameter works, while against an `IEnumerable<T>` parameter it does not, because `List<T>` implements both interfaces and the compiler picks the better one. When unsure, specify the type explicitly: `Picker.First<Book>(...)`.
- Static fields in a generic class are NOT shared between closed types: `Repository<Book>.Counter` and `Repository<ReaderTicket>.Counter` are two distinct fields. If you try to make a global counter via `static int _total` inside `Repository<T>`, you get a separate counter per `T` — a frequent bug source mentioned in the lesson.
- `default(T)` for a reference type yields `null`, for a value type the zero value. The `GetOrDefault` method in `Registry` must return `TValue?` (with `?`), otherwise the nullable analyzer emits CS8603 for reference `TValue`. For value `TValue`, `default` is `0`/`false`/`null`-struct and `?` is harmless.
- The `where TKey : notnull` constraint differs from `where TKey : class`: the former admits both reference types and non-null value types (e.g., `int`, `InventoryNumber`), the latter only reference types. Choose `notnull` when you want to forbid `null` specifically, not value types.
- Do not over-apply constraints: `where T : class, new(), IComparable<T>, IDisposable` leaves almost nothing eligible. The lesson explicitly warns against this. For `MaxBy`, `where TResult : IComparable<TResult>` is enough.
- `record struct` for `InventoryNumber` automatically implements `IEquatable<InventoryNumber>` and `GetHashCode` — critical for use as a `Dictionary<InventoryNumber, Book>` key. If you used an ordinary `class`, keys would compare by reference and break lookup.
- The `All()` method must return an immutable view (`AsReadOnly()` or `ToImmutableArray()`), otherwise external code could mutate the repository's internal list and break encapsulation. The lesson mentions `IList<T>` — a potentially mutable interface, so be careful.
- Top-level statements and `global using System;` (via `<ImplicitUsings>enable</ImplicitUsings>`) simplify the code, but confirm that `System.Collections.Generic`, `System.Linq` are available — otherwise `List<T>`, `Dictionary<TKey,TValue>`, `Func<T,TResult>` will not resolve.

#### Acceptance criteria
- [ ] The `LibraryCatalog` project is created under .NET 8; `dotnet build` succeeds with no errors or warnings.
- [ ] A generic class `Repository<T>` is declared with methods `Add`, `Get`, `TryGet`, `All`, `Count`, `RemoveAt`.
- [ ] `Repository<T>` stores data in `List<T>` and does not use `object` or `ArrayList`.
- [ ] `Add` accepts `T` and `Get` returns `T` — compile-time type safety.
- [ ] `All()` returns an immutable view (`AsReadOnly` or `ToImmutableArray`).
- [ ] A class `Registry<TKey, TValue> where TKey : notnull` is declared with `Put`, `GetOrDefault`, `Contains`, `Entries`.
- [ ] `GetOrDefault` returns `TValue?` and handles `default` correctly for value and reference types.
- [ ] A static class `Picker` is implemented with three generic methods: `First<T>`, `Pair<TKey,TValue>`, `MaxBy<T,TResult>`.
- [ ] `First<T>` throws `InvalidOperationException` for an empty list with a bilingual message.
- [ ] `MaxBy` applies `where TResult : IComparable<TResult>` with a justification in a comment.
- [ ] `Program.cs` uses a `Picker.First(books)` call without explicit `<Book>` — type inference.
- [ ] `Picker.Pair("id", 42)` and `Picker.MaxBy(books, b => b.Title)` are demonstrated.
- [ ] A `try/catch (InvalidOperationException)` for an empty list is demonstrated.
- [ ] The `InventoryNumber` model is a `readonly record struct`, ensuring correct dictionary-key equality.
- [ ] `///` XML comments are present on all public members in both languages (RU/EN).
- [ ] The nullable analyzer (`<Nullable>enable</Nullable>`) produces no CS8602/CS8603/CS8714 warnings.

#### Hints (no direct answer)
- Recall the `Repo<T>` example from the lesson — it is almost ready; you only need to add `TryGet` and `All`.
- For `Registry` look at `Map<TKey, TValue> where TKey : notnull` — the structure is identical.
- `MaxBy` can be implemented with a loop: `TResult best = selector(list[0]); for (int i = 1; i < list.Count; i++) { var v = selector(list[i]); if (v.CompareTo(best) > 0) best = v; }`. Think about what to return: `TResult` or `T`? (The lesson hints at `TResult` via the parameter name.)
- For an empty list in `First` use `InvalidOperationException`, as in the lesson.
- To avoid CS8714 when passing `null` to `Put`, do not bypass the constraint with `null!` — it defeats the purpose of `where TKey : notnull`.
- Use collection expressions `[]` and `..` where you assemble lists if you want to try C# 12 — but not at the cost of readability.

#### Reference solution walk-through
```csharp
// C# 12 / .NET 8 — Reference solution for HW M06-L02

using System;
using System.Collections.Generic;
using System.Collections.Immutable;
using System.Linq;

namespace LibraryCatalog;

public readonly record struct InventoryNumber(int Value);

public sealed record Book(string Title, string Author, InventoryNumber Inventory);

public sealed record ReaderTicket(int Number, string Name);

public sealed record LoanEvent(InventoryNumber Inventory, ReaderTicket Reader, DateTime At);

/// <summary>
/// Generic repository of items of type T.
/// T is a type parameter, replaced by a concrete type on use.
/// </summary>
public class Repository<T>
{
    // List<T> stores T directly, no boxing for value types.
    private readonly List<T> _items = new();

    /// <summary>Add an item. Accepts only T.</summary>
    public void Add(T item) => _items.Add(item);

    /// <summary>Get item by index. Returns T.</summary>
    public T Get(int index) => _items[index];

    /// <summary>Safe lookup by predicate.</summary>
    public bool TryGet(Predicate<T> match, out T? found)
    {
        var i = _items.FindIndex(match);
        if (i < 0) { found = default; return false; }
        found = _items[i];
        return true;
    }

    /// <summary>Immutable view of all items.</summary>
    public IReadOnlyList<T> All() => _items.AsReadOnly();

    /// <summary>Number of items.</summary>
    public int Count => _items.Count;

    /// <summary>Remove item by index.</summary>
    public void RemoveAt(int index) => _items.RemoveAt(index);
}

/// <summary>
/// Key-value registry with a null-key prohibition.
/// where TKey : notnull mirrors the lesson's Map<TKey,TValue> example.
/// </summary>
public class Registry<TKey, TValue> where TKey : notnull
{
    // Dictionary<TKey,TValue> requires GetHashCode/Equals on TKey.
    private readonly Dictionary<TKey, TValue> _data = new();

    /// <summary>Put a pair. Key cannot be null due to where.</summary>
    public void Put(TKey key, TValue value) => _data[key] = value;

    /// <summary>Get value or default(TValue).</summary>
    public TValue? GetOrDefault(TKey key) =>
        _data.TryGetValue(key, out var value) ? value : default;

    /// <summary>Whether the key exists.</summary>
    public bool Contains(TKey key) => _data.ContainsKey(key);

    /// <summary>All pairs.</summary>
    public IEnumerable<KeyValuePair<TKey, TValue>> Entries => _data;
}

/// <summary>
/// Static utilities with generic methods (non-generic class, as in the lesson).
/// </summary>
public static class Picker
{
    /// <summary>First element. Throws InvalidOperationException if empty.</summary>
    public static T First<T>(IList<T> list)
    {
        if (list.Count == 0)
            throw new InvalidOperationException("List is empty / Список пуст");
        return list[0];
    }

    /// <summary>Key-value pair. Two type parameters.</summary>
    public static KeyValuePair<TKey, TValue> Pair<TKey, TValue>(TKey key, TValue value)
        => new(key, value);

    /// <summary>Item with the maximum selector value.</summary>
    /// <typeparam name="TResult">Selector result type; requires IComparable.</typeparam>
    public static T MaxBy<T, TResult>(IList<T> list, Func<T, TResult> selector)
        where TResult : IComparable<TResult>
    {
        if (list.Count == 0)
            throw new InvalidOperationException("List is empty / Список пуст");
        var best = list[0];
        var bestKey = selector(best);
        for (int i = 1; i < list.Count; i++)
        {
            var item = list[i];
            var key = selector(item);
            if (key.CompareTo(bestKey) > 0)
            {
                best = item;
                bestKey = key;
            }
        }
        return best;
    }
}

// Demo in top-level statements
var books = new Repository<Book>();
books.Add(new Book("C# in Depth", "Jon Skeet", new InventoryNumber(1)));
books.Add(new Book("CLR via C#", "Jeffrey Richter", new InventoryNumber(2)));
books.Add(new Book("Pro ASP.NET Core", "Andrew Freeman", new InventoryNumber(3)));

var byInventory = new Registry<InventoryNumber, Book>();
foreach (var b in books.All())
    byInventory.Put(b.Inventory, b);

// Type inference: Picker.First(books) infers T = Book without explicit <Book>.
var first = Picker.First(books.All().ToList());
Console.WriteLine($"First book: {first.Title}");

// Key-value pair
var pair = Picker.Pair("id", 42);
Console.WriteLine($"Pair: {pair.Key}={pair.Value}");

// MaxBy with IComparable<TResult> constraint
var maxBook = Picker.MaxBy(books.All().ToList(), b => b.Title);
Console.WriteLine($"Max by title: {maxBook.Title}");

// Exception demo
try
{
    Picker.First(new List<Book>());
}
catch (InvalidOperationException ex)
{
    Console.WriteLine($"Expected error: {ex.Message}");
}

// Loan journal
var loans = new Repository<LoanEvent>();
var reader = new ReaderTicket(101, "Alice");
loans.Add(new LoanEvent(new InventoryNumber(1), reader, DateTime.UtcNow));

var byReader = new Registry<ReaderTicket, List<LoanEvent>>();
byReader.Put(reader, loans.All().ToList());
Console.WriteLine($"Loans for {reader.Name}: {byReader.GetOrDefault(reader)?.Count ?? 0}");
```

Line-by-line walk-through: `Repository<T>` mirrors the lesson's `Repo<T>` — a type parameter `T` in angle brackets, a private `List<T>` field, `Add(T)` and `Get(int): T` methods. This is the core of type safety: the compiler substitutes a concrete `T` at each closed type (`Repository<Book>`, `Repository<LoanEvent>`), and trying to add a string to `Repository<Book>` is a CS0029 compile error, not a runtime exception. For value `T` (e.g., `LoanEvent` or `InventoryNumber`) the data sits in `List<T>` directly, with no boxing to `object` — exactly what generics were introduced for, as the lesson emphasizes. The `All()` method returns `IReadOnlyList<T>` via `AsReadOnly()` to avoid exposing the mutable internal list — the lesson notes that `IList<T>` is potentially mutable, a typical encapsulation hole. The `Registry<TKey, TValue> where TKey : notnull` class mirrors the lesson's `Map<TKey, TValue>`: the `notnull` constraint forbids a `null` key at compile time (CS8714 on `Put(null!, ...)`), which is safer than the old `Dictionary` behavior that threw `ArgumentNullException` only at runtime. Parameters are named `TKey` and `TValue` per the lesson convention. `GetOrDefault` returns `TValue?`: for reference `TValue` that is `null`, for value `TValue` it is `default` (e.g., `0` for `int`) — matching the lesson's note on `default(T)` and nullable analysis. The `Picker` class is non-generic but holds three generic methods, illustrating the lesson's claim that "generics can be applied to a single method rather than the whole class". `First<T>` uses type inference: at the call site `Picker.First(books.All().ToList())` the compiler infers `T = Book` from the argument, exactly like the lesson's `First(users)` example. Throwing `InvalidOperationException` for an empty list matches the lesson. `Pair<TKey, TValue>` demonstrates a method with two type parameters, analogous to the lesson's `Picker.Pair`. `MaxBy<T, TResult>` applies `where TResult : IComparable<TResult>` — the very concept from the lesson about `where T : IComparable<T>`, granting access to `CompareTo`. The lesson warns against "overly strict constraints" like `where T : class, new(), IComparable<T>, IDisposable`; here the constraint is minimally necessary, only `IComparable<TResult>`, which follows the best practice. The `InventoryNumber` model is declared `readonly record struct`: this gives built-in value-based `Equals`/`GetHashCode` needed to use it as a `Dictionary<InventoryNumber, Book>` key — an ordinary `class` would compare by reference and break key lookup, which directly echoes the lesson's point about `Dictionary<TKey, TValue>`. The demo uses top-level statements — a C# 9+ feature supported on .NET 8; C# 12 collection expressions are optional here. The `Picker.First(books.All().ToList())` call proves type inference. The `try/catch` block demonstrates the lesson's exception. The whole code compiles under C# 12 / .NET 8 with `<Nullable>enable</Nullable>` and no warnings. Thus the reference covers: a generic class with a single `T`, a class with two `TKey`/`TValue` and a `where` constraint, generic methods with type inference and with a constraint, boxing avoidance, an immutable view, a value type as a dictionary key — the full set of topics from lesson M06-L02.

#### Going deeper (bonus)
1. Add an `internal static int InstanceCount;` field to `Repository<T>` and increment it in the constructor. Create `Repository<Book>`, `Repository<LoanEvent>`, and another `Repository<Book>`, then print `Repository<Book>.InstanceCount` and `Repository<LoanEvent>.InstanceCount` — confirm the counters are separate per closed type (this illustrates the lesson's note on static fields in generic classes).
2. Implement a `Picker.MaxBy<T, TResult>` overload that takes an `IComparer<TResult>` instead of the `IComparable<TResult>` constraint. Compare the two approaches: which is more flexible, which is faster, which better matches the lesson's best practice of "do not over-apply constraints"?
3. Rewrite `Registry<TKey, TValue>` so it implements `IEnumerable<KeyValuePair<TKey, TValue>>` and exposes an indexer `this[TKey key]`. Think about how to keep the `where TKey : notnull` constraint and the nullable analysis intact.
4. Add caching to `Registry`: on the first `GetOrDefault` for a missing key, store `default(TValue)` in a separate `Dictionary<TKey, TValue>` and return from cache on subsequent calls. Think about how the cache behaves for value and reference `TValue` and whether `default` for a reference type causes a problem (linked to the lesson's note on `default(T)` and `null`).

---

#### Чек-лист сдачи / Submission checklist
- [ ] Проект `LibraryCatalog` собирается под .NET 8 без ошибок и предупреждений.
- [ ] Реализованы `Repository<T>`, `Registry<TKey,TValue>`, `Picker` согласно требованиям.
- [ ] В коде есть вызов с выводом типов и метод с ограничением `where`.
- [ ] Модель `InventoryNumber` — `readonly record struct`.
- [ ] XML-комментарии на двух языках присутствуют на всех публичных членах.
- [ ] Вывод `dotnet run` соответствует ожидаемому.
- [ ] Опционально: добавлены модульные тесты xUnit.
- [ ] The `LibraryCatalog` project builds under .NET 8 with no errors or warnings.
- [ ] `Repository<T>`, `Registry<TKey,TValue>`, `Picker` are implemented per requirements.
- [ ] The code contains a type-inference call and a `where`-constrained method.
- [ ] The `InventoryNumber` model is a `readonly record struct`.
- [ ] Bilingual XML comments are present on all public members.
- [ ] The `dotnet run` output matches expectations.
- [ ] Optional: xUnit unit tests are added.

#### Ресурсы / Resources
- [Microsoft Learn — Generics](https://learn.microsoft.com/dotnet/csharp/fundamentals/types/generics)
- [Microsoft Learn — Constraints on type parameters](https://learn.microsoft.com/dotnet/csharp/programming-guide/generics/constraints-on-type-parameters)
- [Microsoft Learn — Records and record structs](https://learn.microsoft.com/dotnet/csharp/whats-new/csharp-10#record-structs)
- [.NET 8 release notes](https://learn.microsoft.com/dotnet/core/whats-new/dotnet-8)

---

[← К уроку M06-L02](lesson-M06-L02-generic-classes-methods.md) | [⬆ К модулю M06](../README.md) | [Следующее ДЗ →](homework-M06-L03-constraints-where.md)
