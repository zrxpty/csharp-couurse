[⬆ К модулю M04](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M04-L06: this, индексаторы / this, indexers

**Модуль / Module:** M04
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

Ключевое слово `this` в C# — это ссылка на текущий экземпляр класса или структуры. Представьте себе ресторан: официант говорит «это столик №5» — он указывает на конкретный объект, с которым работает прямо сейчас. Точно так же `this` указывает на тот конкретный экземпляр, внутри метода которого выполняется код. Если поле класса называется `name` и параметр метода тоже называется `name`, конструкция `this.name = name` однозначно говорит компилятору: «полю текущего экземпляра присвой значение параметра». Без `this` возникла бы неоднозначность, и компилятор решил бы, что вы присваиваете параметр самому себе.

`this` используется в нескольких случаях. Во-первых, для различения поля и параметра с одинаковым именем — классический приём в конструкторах. Во-вторых, для передачи текущего экземпляра другому методу: `printer.Print(this)`. В-третьих, для явного возврата самого объекта из метода, что enables fluent API, например `return this;` в паттерне Builder. В-четвёртых, для объявления индексаторов, о которых речь пойдёт ниже.

Особый случай — вызов другого конструктора через `this(...)`. Когда у класса несколько конструкторов с разным набором параметров, один конструктор может вызвать другой, чтобы не дублировать логику инициализации. Конструкция `public Book(string title) : this(title, "Unknown")` означает: сначала вызови конструктор с двумя параметрами, а потом выполни тело текущего конструктора. Вызов `this(...)` должен стоять в одной строке с объявлением, до тела. Это похоже на то, как старший мастер обучает подмаерья: основной конструктор делает всю черновую работу, а упрощённые конструкторы лишь подставляют значения по умолчанию и делегируют ему. Важное правило: вызов через `this(...)` выполняется до тела конструктора, а не после. В структурах `this()` также может инициализировать поля, но с ограничениями.

Индексаторы — это синтаксический сахар, позволяющий обращаться к объекту как к массиву через квадратные скобки `obj[index]`. Это удобно, когда класс логически представляет собой коллекцию или контейнер. Например, класс `Matrix` может иметь индексатор `matrix[i, j]`, а `WordList` — `words[0]`. Под капотом индексатор компилируется в свойство с параметром и методами `get`/`set`. Объявляется он с помощью `this` с параметрами в квадратных скобках: `public string this[int index] { get; set; }`.

Индексаторы можно перегружать по типу параметра. Один класс может иметь `this[int]` для доступа по числовому индексу и `this[string]` — по строковому ключу. Компилятор выберет нужный по типу аргумента в квадратных скобках. Это похоже на перегрузку методов, только применительно к «квадратным скобкам». Индексаторы могут быть только для чтения (только `get`), только для записи, или с асимметричными модификаторами доступа: `public int this[int i] { get; private set; }`. Индексаторы могут принимать несколько параметров, например для двумерной матрицы `this[int row, int col]`.

Важно помнить несколько тонкостей. Индексаторы не имеют имени, поэтому их нельзя вызывать по имени — только через `[]`. Они должны быть членами экземпляра (не `static`). Внутри `get`/`set` вы сами решаете, как хранить данные — во внутреннем массиве, словаре, списке или вычислять на лету. И, конечно, внутри индексатора `this` доступен и ссылается на текущий экземпляр, как и в любом методе.

#### Theory (EN)

The `this` keyword in C# is a reference to the current instance of a class or struct. Imagine a restaurant where a waiter says "this is table five" — he is pointing at the specific object he is working with right now. Likewise, `this` points at the exact instance whose method is currently executing. If a class field is named `name` and a method parameter is also named `name`, the expression `this.name = name` tells the compiler unambiguously: "assign the parameter value to the field of the current instance." Without `this` there would be ambiguity, and the compiler would assume you are assigning the parameter to itself.

There are several typical uses of `this`. First, distinguishing a field from a parameter that share the same name — the classic constructor pattern. Second, passing the current instance to another method, as in `printer.Print(this)`. Third, returning the object itself from a method to enable a fluent API, for example `return this;` in the Builder pattern. Fourth, declaring indexers, which we cover below.

A special case is invoking another constructor through `this(...)`. When a class has several constructors with different parameter sets, one constructor may call another to avoid duplicating initialization logic. The construct `public Book(string title) : this(title, "Unknown")` means: first call the two-parameter constructor, then execute the body of the current constructor. The `this(...)` call must appear on the same line as the declaration, before the body. It is like a senior craftsman training an apprentice: the main constructor does the heavy lifting, while simpler constructors supply default values and delegate to it. A key rule: the `this(...)` call runs before the constructor body, not after. In structs, `this()` can also initialize fields, but with restrictions.

Indexers are syntactic sugar that lets you treat an object like an array using square brackets: `obj[index]`. This is convenient when a class logically represents a collection or container. For example, a `Matrix` class can expose `matrix[i, j]`, and a `WordList` can expose `words[0]`. Under the hood, an indexer compiles to a parameterized property with `get` and `set` accessors. It is declared with `this` plus parameters in square brackets: `public string this[int index] { get; set; }`.

Indexers can be overloaded by parameter type. A single class may offer `this[int]` for numeric access and `this[string]` for string-key access. The compiler picks the right one based on the type inside the brackets — just like method overloading, but applied to "square brackets." Indexers can be read-only (`get` only), write-only, or have asymmetric access modifiers: `public int this[int i] { get; private set; }`. They can also take several parameters, for example a two-dimensional matrix `this[int row, int col]`.

A few subtleties matter. Indexers have no name, so they cannot be invoked by name — only through `[]`. They must be instance members (not `static`). Inside `get`/`set` you decide how data is stored: in an internal array, dictionary, list, or computed on the fly. And of course, inside an indexer `this` is available and refers to the current instance, just like in any method.

#### Пример кода / Code Example

```csharp
// C# 12 / .NET 8 — this, indexers
// Двуязычные комментарии RU+EN / Bilingual comments

using System;
using System.Collections.Generic;

// Класс демонстрирует: this для полей, this() вызов конструктора, индексаторы
// The class demonstrates: this for fields, this() constructor call, indexers
public class Bookshelf
{
    private readonly string _owner;     // владелец полки / shelf owner
    private readonly List<string> _books = new();

    // Основной конструктор с двумя параметрами — выполняет реальную работу
    // Main constructor with two parameters — does the real work
    public Bookshelf(string owner, IEnumerable<string> initialBooks)
    {
        _owner = owner;                          // this не нужен: имена различаются
                                                 // this not needed: names differ
        _books.AddRange(initialBooks);
    }

    // Упрощённый конструктор делегирует основному через this(...)
    // Simplified constructor delegates to the main one via this(...)
    public Bookshelf(string owner) : this(owner, Array.Empty<string>())
    {
        // тело выполняется ПОСЛЕ вызванного конструктора
        // body runs AFTER the delegated constructor
    }

    // Конструктор по умолчанию — fluent-стиль: возвращаем this из метода
    // Default constructor — fluent style: return this from a method
    public Bookshelf() : this("Anonymous") { }

    // Fluent-метод возвращает текущий экземпляр / Fluent method returns current instance
    public Bookshelf Add(string title)
    {
        _books.Add(title);
        return this;   // возвращаем себя, чтобы строить цепочку / return self for chaining
    }

    // Индексатор по целочисленному индексу — доступ как к массиву
    // Integer indexer — array-like access
    public string this[int index]
    {
        get
        {
            if (index < 0 || index >= _books.Count)
                throw new IndexOutOfRangeException($"Неверный индекс / Bad index: {index}");
            return _books[index];
        }
        set
        {
            if (index < 0 || index >= _books.Count)
                throw new IndexOutOfRangeException($"Неверный индекс / Bad index: {index}");
            _books[index] = value;   // value — неявный параметр setter / value is the implicit setter param
        }
    }

    // Перегрузка индексатора: доступ по строковому заголовку (только чтение)
    // Indexer overload: access by string title (read-only)
    public string this[string title]
    {
        get
        {
            int idx = _books.FindIndex(b => b.Equals(title, StringComparison.OrdinalIgnoreCase));
            if (idx < 0) throw new KeyNotFoundException($"Книга не найдена / Book not found: {title}");
            return _books[idx];
        }
    }

    public int Count => _books.Count;
    public string Owner => _owner;

    public override string ToString() => $"{Owner}: {Count} книг / books";
}

public static class Demo
{
    public static void Run()
    {
        // Цепочка вызовов благодаря return this / Chained calls thanks to return this
        var shelf = new Bookshelf("Alice")
            .Add("C# in Depth")
            .Add("Clean Code")
            .Add("Domain-Driven Design");

        Console.WriteLine(shelf);                       // Alice: 3 книг / books

        // Доступ по числовому индексу / Access by integer index
        Console.WriteLine(shelf[0]);                    // C# in Depth
        shelf[1] = "Refactoring";                       // set через индексатор / set via indexer
        Console.WriteLine(shelf[1]);                    // Refactoring

        // Доступ по строковому ключу — перегрузка индексатора
        // Access by string key — indexer overload
        Console.WriteLine(shelf["Clean Code"]);         // Clean Code

        // Упрощённый конструктор через this(...) / Simplified constructor via this(...)
        var empty = new Bookshelf("Bob");
        Console.WriteLine(empty);                       // Bob: 0 книг / books
    }
}
```

#### Best Practices
- Используйте `this` для полей только когда это устраняет неоднозначность имени; избыточный `this.` everywhere засоряет код. / Use `this` for fields only to resolve name ambiguity; redundant `this.` everywhere clutters the code.
- Держите один «главный» конструктор с полной логикой инициализации и делегируйте к нему из остальных через `this(...)`. / Keep one "main" constructor with full initialization logic and delegate to it from others via `this(...)`.
- Валидируйте индекс внутри индексаторов и выбрасывайте осмысленные исключения (`IndexOutOfRangeException`, `KeyNotFoundException`). / Validate the index inside indexers and throw meaningful exceptions (`IndexOutOfRangeException`, `KeyNotFoundException`).
- Предпочитайте индексатор с `private set` или только `get`, если изменение элементов извне не предусмотрено контрактом. / Prefer an indexer with `private set` or `get`-only if external mutation is not part of the contract.
- Не делайте индексаторы с побочными эффектами или тяжёлыми вычислениями — ожидание `obj[i]` должно быть дёшево, как обращение к массиву. / Avoid indexers with side effects or heavy computation — `obj[i]` should feel as cheap as array access.

#### Частые ошибки / Common Mistakes
- [Использование `this` в `static` методе] → [В `static` нет экземпляра; обращайтесь к статическим членам по имени класса.] (RU)
- [Вызов `this(...)` внутри тела конструктора вместо строки объявления] → [Ставьте `: this(...)` сразу после списка параметров, до тела.] (RU)
- [Отсутствие проверки границ в индексаторе] → [Проверяйте диапазон и выбрасывайте `IndexOutOfRangeException` с понятным сообщением.] (RU)
- [Слишком много перегрузок индексатора со схожими типами параметров] → [Это запутывает читателя; используйте именованные методы для неясных случаев.] (RU)
- [Изменение внутреннего состояния в `get` индексатора] → [`get` должен быть без побочных эффектов; side effects разрушают ожидания.] (RU)
- [Using `this` inside a `static` method] → [There is no instance in `static`; reach static members via the class name.] (EN)
- [Putting `this(...)` inside the constructor body instead of the declaration line] → [Place `: this(...)` right after the parameter list, before the body.] (EN)
- [No bounds check in the indexer] → [Validate the range and throw `IndexOutOfRangeException` with a clear message.] (EN)
- [Too many indexer overloads with similar parameter types] → [It confuses readers; use named methods for ambiguous cases.] (EN)
- [Mutating internal state in the indexer `get`] → [`get` must be side-effect free; side effects break expectations.] (EN)

#### Чек-лист самопроверки / Self-check Checklist
- [ ] Я понимаю, что `this` ссылается на текущий экземпляр и недоступен в `static`. / I understand that `this` refers to the current instance and is unavailable in `static`.
- [ ] Я умею различать поле и параметр через `this.field`. / I can distinguish a field from a parameter via `this.field`.
- [ ] Я могу вызвать один конструктор из другого через `: this(...)`. / I can call one constructor from another via `: this(...)`.
- [ ] Я могу объявить индексатор `this[int]` с `get` и `set`. / I can declare a `this[int]` indexer with `get` and `set`.
- [ ] Я умею перегружать индексаторы по типу параметра (например, `int` и `string`). / I can overload indexers by parameter type (e.g., `int` and `string`).
- [ ] Я проверяю границы и выбрасываю осмысленные исключения. / I validate bounds and throw meaningful exceptions.
- [ ] Я знаю, что индексаторы не имеют имени и не могут быть `static`. / I know indexers have no name and cannot be `static`.

#### Ресурсы / Resources
- [Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/indexers]

---

[⬆ К модулю M04](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
