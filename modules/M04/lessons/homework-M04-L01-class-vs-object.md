---
[← К уроку M04-L01](lesson-M04-L01-class-vs-object.md) | [⬆ К модулю M04](../README.md) | [Следующее ДЗ →](homework-M04-L02-properties.md)
---

### Домашнее задание M04-L01: Класс vs объект, объявление класса / Homework M04-L01: Class vs object, declaring a class

**Урок / Lesson:** M04-L01
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 3/5
**Цель / Goal:** (RU) Научиться объявлять классы C# 12 / .NET 8 с полями, свойствами, методами и конструктором, создавать независимые объекты через `new` с объектными инициализаторами, отличать экземплярные члены от статических и закрепить соглашения об именовании PascalCase / camelCase на сквозном примере. (EN) Learn to declare C# 12 / .NET 8 classes with fields, properties, methods and a constructor, create independent objects via `new` with object initializers, tell instance members apart from static ones, and internalize the PascalCase / camelCase naming conventions through one end-to-end example.

#### Связь с уроком / Connection to the lesson

(RU) Урок вводит фундаментальную пару ООП — класс как чертёж и объект как построенный по нему экземпляр — и показывает объявление класса, оператор `new`, объектные инициализаторы и разницу между статическими и экземплярными членами. Это ДЗ переводит теорию в практику: вы спроектируете два связанных класса (`Book` и корзину `ReadingList`), создадите несколько независимых объектов, покажете, что каждый хранит собственное состояние, и используете статический счётчик так, как в уроке использовался `House.TotalBuilt`.

(EN) The lesson introduces the foundational OOP pair — a class as a blueprint and an object as an instance built from it — and demonstrates class declaration, the `new` operator, object initializers, and the difference between static and instance members. This homework turns theory into practice: you will design two related classes (`Book` and a `ReadingList` cart), create several independent objects, demonstrate that each holds its own state, and use a static counter exactly the way the lesson used `House.TotalBuilt`.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация

Представьте, что вы разрабатываете небольшой модуль учёта книг для читательского клуба. В клубе есть много книг, и каждую книгу можно описать одним и тем же набором данных: название, автор, число страниц, год издания и признак «прочитана ли». Это идеальный кандидат на класс: класс `Book` станет чертежом, по которому вы создадите столько конкретных книг-объектов, сколько нужно. Каждый объект будет хранить собственное состояние (своё название, свой год), но делить с другими объектами общее поведение — методы `MarkAsRead()`, `Describe()` и т. д.

Параллельно клуб хочет вести списки чтения — подборки книг на месяц. Список тоже естественно смоделировать классом: `ReadingList` хранит название подборки и сам набор книг. Один список можно наполнять разными объектами-книгами, и несколько списков существуют независимо друг от друга, хотя построены по одному и тому же чертежу `ReadingList`.

Наконец, руководству клуба интересно знать, сколько всего книг зарегистрировано в системе. Это число относится не к конкретной книге, а ко всему классу `Book` целиком — ровно та ситуация, где в уроке использовалось статическое поле `House.TotalBuilt`. Сравнивая экземплярное поле (например, `Pages` — у каждой книги своё) и статическое (`TotalRegistered` — общее для класса), вы на собственном коде почувствуете границу между «принадлежит объекту» и «принадлежит классу», которая является ядром урока.

#### Что нужно сделать (пошагово)

1. Создайте новый консольный проект .NET 8 с именем `M04.L01.Homework`:
   ```
   dotnet new console -n M04.L01.Homework -f net8.0
   cd M04.L01.Homework
   ```
   Убедитесь, что в `M04.L01.Homework.csproj` указан `<TargetFramework>net8.0</TargetFramework>` и язык C# 12 (для .NET 8 это умолчание). Откройте `Program.cs` — он будет точкой входа.

2. Объявите класс `Book` в файле `Book.cs` в пространстве имён `M04.L01.Homework`. Имя класса — строго `PascalCase`. Внутри класса опишите: публичные поля `Title`, `Author`, `Pages`, `YearPublished` подходящих типов (`string`, `int`), публичное поле `IsRead` типа `bool`, статическое поле `TotalRegistered` типа `int`, конструктор `Book(string title, string author, int pages, int yearPublished)`, который заполняет поля и увеличивает `TotalRegistered`, а также методы `MarkAsRead()` (ставит `IsRead = true`) и `Describe()` (печатает сводку книги через `Console.WriteLine`).

3. Объявите класс `ReadingList` в файле `ReadingList.cs` в том же пространстве имён. Поля: публичное `Name` типа `string` и публичное поле `Books` — массив `Book[]` (или `List<Book>`, если вы уже знакомы с ним). Добавьте конструктор `ReadingList(string name)`, который задаёт `Name` и инициализирует пустой массив/список, метод `Add(Book book)`, добавляющий книгу, и метод `Print()`, печатающий название подборки и `Describe()` каждой книги.

4. В `Program.cs` создайте минимум четыре объекта `Book` через конструктор, например книги Толстого, Оруэлла, Кларка и Сапковского. Покажите, что это независимые объекты: поменяйте `IsRead` у одной книги вызовом `MarkAsRead()` и убедитесь, что остальные книги остались непрочитанными — выведите `IsRead` каждой книги в консоль.

5. Создайте два объекта `ReadingList` с разными именами (например, «Классика» и «Фантастика»). Добавьте в первый список две книги, во второй — другие две. Вызовите `Print()` у каждого списка и убедитесь, что они независимы.

6. Покажите статический счётчик: после создания всех книг напечатайте `Book.TotalRegistered`. Обратитесь к нему через имя класса, а не через объект. Создайте ещё одну книгу и снова напечатайте `TotalRegistered` — значение должно увеличиться.

7. Запустите проект командой `dotnet run` из папки проекта. Зафиксируйте ожидаемый вывод: для каждой книги — строка `Describe()`, для списков — заголовок и описания книг, для счётчика — итоговое число.

8. Осознанно продемонстрируйте одну частую ошибку из урока в виде комментария в коде (не запускайте её): закомментированную строку вида `// var x = Book.Title; // Ошибка: обращение к экземплярному полю через имя класса` с пояснением, почему так делать нельзя и как правильно (`someBook.Title`).

#### Требования к решению

Решение должно компилироваться без предупреждений уровня error под .NET 8 / C# 12 и запускаться командой `dotnet run`. Используйте `new` для каждого объекта: переменная без `new` (например, `Book b;`) не создаёт объект и при обращении к членам приведёт к `NullReferenceException` — таких мест в рабочем коде быть не должно. Классы должны жить в пространстве имён `M04.L01.Homework` и быть разнесены по файлам `Book.cs`, `ReadingList.cs`, `Program.cs`.

Все имена классов, публичных полей и методов — в `PascalCase` (`Book`, `ReadingList`, `MarkAsRead`, `TotalRegistered`); имена локальных переменных и параметров конструктора — в `camelCase` (`title`, `firstList`, `fantasyBook`). Методы называйте глаголами или глагольными фразами (`MarkAsRead`, `Add`, `Print`, `Describe`), классы — существительными (`Book`, `ReadingList`).

Каждый объект `Book` должен демонстрировать собственное состояние: разные `Title`, `Author`, `Pages`, `YearPublished`, независимый `IsRead`. Статическое поле `TotalRegistered` должно изменяться только при создании нового объекта через конструктор и быть доступным через `Book.TotalRegistered`. В выводе должно быть видно, что экземплярное поле своё у каждого объекта, а статическое — одно на весь класс.

#### Тонкости и подводные камни

- **Не обращайтесь к экземплярному полю через имя класса.** `Book.Title` не скомпилируется или не имеет смысла — `Title` принадлежит конкретному объекту. Правильно: `someBook.Title`. Через имя класса доступны только `static`-члены, такие как `Book.TotalRegistered`.
- **Не забывайте `new`.** `Book b;` без `new` не создаёт объект. В классе-ссылочном типе переменная остаётся `null`, и `b.Describe()` бросит `NullReferenceException`. Каждый объект должен рождаться через `new Book(...)` или объектный инициализатор `new Book { ... }`.
- **Различайте класс и объект.** Фраза «класс хранит название» концептуально неверна: класс — чертёж, данные хранят объекты. У класса нет своего `Title`; есть поле `Title`, значение которого живёт в каждом объекте отдельно.
- **Статическое поле — одно на класс.** `TotalRegistered` инкрементируется в конструкторе, поэтому при создании каждого объекта оно растёт. Если случайно объявить его без `static`, у каждого объекта будет свой независимый счётчик, и идея «общего для класса» сломается.
- **Имена в lowerCase — нарушение конвенций.** `class book` скомпилируется, но противоречит стилю C# и мешает команде. Используйте `PascalCase` для классов и публичных членов, `camelCase` для локальных переменных и параметров.
- **Объекты независимы.** Изменение `IsRead` у одной книги не должно влиять на другие. Если вы случайно сделаете поле `static`, изменение «прочитана» у одной книги поменяет флаг у всех — это частый и коварный баг, прямо иллюстрирующий опасность путаницы static/instance.
- **SRP.** Класс `Book` описывает книгу, `ReadingList` — подборку. Не пытайтесь в `Book` добавить логику печати всего списка — это ответственность `ReadingList`.

#### Критерии приёмки

- [ ] Проект `M04.L01.Homework` создаётся командой `dotnet new console -f net8.0` и собирается без ошибок.
- [ ] `dotnet run` выводит ожидаемые строки без исключений.
- [ ] Класс `Book` объявлен в `Book.cs` в пространстве имён `M04.L01.Homework` с именем в `PascalCase`.
- [ ] У `Book` есть поля `Title`, `Author`, `Pages`, `YearPublished`, `IsRead` подходящих типов.
- [ ] У `Book` есть статическое поле `TotalRegistered` и конструктор, увеличивающий его.
- [ ] Методы `MarkAsRead()` и `Describe()` реализованы и вызываются.
- [ ] Класс `ReadingList` объявлен в `ReadingList.cs` с полями `Name` и коллекцией книг.
- [ ] `ReadingList` имеет конструктор и методы `Add(Book)` и `Print()`.
- [ ] Создано не менее четырёх объектов `Book` через `new`.
- [ ] Продемонстрирована независимость объектов: `MarkAsRead()` меняет только одну книгу.
- [ ] Создано не менее двух объектов `ReadingList` с разными книгами и показана их независимость.
- [ ] `Book.TotalRegistered` печатается до и после создания дополнительной книги и корректно растёт.
- [ ] К статическому полю обращаются через имя класса, к экземплярным — через переменную объекта.
- [ ] Соблюдены соглашения: `PascalCase` для классов и публичных членов, `camelCase` для локальных переменных и параметров.
- [ ] В коде есть закомментированный пример типичной ошибки с пояснением.

#### Подсказки (без прямого ответа)

- Подумайте, какой тип лучше всего подходит для `Books` в `ReadingList`: фиксированный массив требует заранее знать размер, а `List<Book>` из `System.Collections.Generic` растёт динамически. Если вы ещё не проходили дженерики глубоко — можно начать с массива и индекса «следующая свободная позиция».
- Чтобы показать независимость объектов, создайте книгу, вызовите `MarkAsRead()`, и тут же выведите `IsRead` у этой и у другой книги. Разница значений и есть доказательство.
- Статический счётчик увеличивайте прямо в теле конструктора — это гарантирует, что каждый `new Book(...)` учтётся.
- Для `Describe()` используйте интерполяцию строк: `$"Книга \"{Title}\" — {Author}, {Pages} стр., {YearPublished} г."`.
- Не пытайтесь печатать весь список из класса `Book` — это нарушает SRP; печать списка принадлежит `ReadingList.Print()`.

#### Эталонное решение (разбор)

```csharp
// Book.cs — C# 12 / .NET 8
// Класс как чертёж книги; объекты — конкретные книги / Class as blueprint; objects are concrete books
namespace M04.L01.Homework;

public class Book
{
    // Экземемплярные поля — у каждого объекта свои / Instance fields, one set per object
    public string Title;
    public string Author;
    public int Pages;
    public int YearPublished;
    public bool IsRead;

    // Статическое поле — одно на весь класс / Static field, shared by the whole class
    public static int TotalRegistered;

    // Конструктор вызывается при каждом new Book(...) / Constructor runs on every new
    public Book(string title, string author, int pages, int yearPublished)
    {
        Title = title;              // camelCase параметр -> PascalCase поле
        Author = author;
        Pages = pages;
        YearPublished = yearPublished;
        IsRead = false;             // явная инициализация / explicit default
        TotalRegistered++;          // общий счётчик растёт / shared counter grows
    }

    public void MarkAsRead() => IsRead = true;   // поведение, меняющее состояние объекта

    public void Describe()
    {
        string status = IsRead ? "прочитана" : "не прочитана";
        Console.WriteLine($"Книга \"{Title}\" — {Author}, {Pages} стр., {YearPublished} г. ({status})");
    }
}
```

```csharp
// ReadingList.cs
namespace M04.L01.Homework;

public class ReadingList
{
    public string Name;
    private readonly List<Book> _books = new();   // инкапсуляция коллекции

    public ReadingList(string name) => Name = name;

    public void Add(Book book) => _books.Add(book);

    public void Print()
    {
        Console.WriteLine($"Подборка «{Name}», книг: {_books.Count}");
        foreach (var book in _books)   // поведение общее, но работает с данными конкретных книг
            book.Describe();
    }
}
```

```csharp
// Program.cs — top-level statements, C# 12
using M04.L01.Homework;

var warAndPeace = new Book("Война и мир", "Л. Толстой", 1225, 1869);
var nineteen84  = new Book("1984", "Дж. Оруэлл", 328, 1949);
var spaceOdyssey = new Book("2001: Космическая одиссея", "А. Кларк", 312, 1968);
var witcher     = new Book("Последнее желание", "А. Сапковский", 336, 1993);

// Демонстрация независимости объектов / Demonstrating object independence
nineteen84.MarkAsRead();
Console.WriteLine($"1984 прочитана: {nineteen84.IsRead}");   // True
Console.WriteLine($"Война и мир прочитана: {warAndPeace.IsRead}"); // False

// Статический счётчик через имя класса / Static counter via the class name
Console.WriteLine($"Зарегистрировано книг: {Book.TotalRegistered}"); // 4

var extra = new Book("Дюна", "Ф. Герберт", 688, 1965);
Console.WriteLine($"Зарегистрировано книг: {Book.TotalRegistered}"); // 5

// Два независимых списка / Two independent lists
var classics = new ReadingList("Классика");
classics.Add(warAndPeace);
classics.Add(nineteen84);

var fantasy = new ReadingList("Фантастика");
fantasy.Add(spaceOdyssey);
fantasy.Add(witcher);

classics.Print();
fantasy.Print();

// Частая ошибка (закомментировано намеренно) / Common mistake, intentionally commented
// var bad = Book.Title; // Ошибка: экземплярное поле через имя класса. Правильно: someBook.Title
```

Разбор по строкам. Класс `Book` объявлен с `public class Book` — `PascalCase`, как требует урок; тело в фигурных скобках содержит данные (поля) и поведение (методы). Поля `Title`, `Author`, `Pages`, `YearPublished`, `IsRead` — экземплярные: у каждого объекта свой набор значений, что напрямую иллюстрирует тезис урока «объект — построенный дом с собственным состоянием». Поле `TotalRegistered` помечено `static`, поэтому оно одно на весь класс — это аналог `House.TotalBuilt` из урока, и обращение к нему идёт через `Book.TotalRegistered`, а не через переменную объекта. Конструктор заполняет поля и инкрементирует счётчик: каждый `new Book(...)` гарантированно учтён, что моделирует идею «при построении дома счётчик растёт». Параметры конструктора — `camelCase` (`title`, `author`), поля — `PascalCase` (`Title`, `Author`): это соответствие конвенциям из урока. Метод `MarkAsRead()` меняет состояние конкретного объекта, а `Describe()` печатает его; поведение общее для класса, но оперирует данными своего объекта — ровно как `OpenDoor()` у `House`. В `Program.cs` создаются четыре независимых объекта; вызов `MarkAsRead()` у `nineteen84` оставляет `warAndPeace.IsRead == false`, что доказывает независимость состояния. Статический счётчик печатается до и после создания пятой книги и растёт с 4 до 5. Два объекта `ReadingList` независимы, хотя построены по одному чертежу. Закомментированная строка `Book.Title` фиксирует частую ошибку из урока — обращение к экземплярному полю через имя класса — с правильной альтернативой.

#### Задания на углубление (бонус)

1. Добавьте статический метод `Book.HowManyRegistered()`, возвращающий `TotalRegistered`, и вызовите его вместо прямого доступа к полю. Подумайте, почему метод предпочтительнее публичного статического поля с точки зрения инкапсуляции.
2. Добавьте в `Book` поле `Rating` типа `int` и метод `SetRating(int value)`, который rejects значения вне диапазона 1–5 (печатает предупреждение и не меняет поле). Это мостик к свойствам из следующего урока.
3. Перепишите `ReadingList` так, чтобы он хранил книги в массиве фиксированной длины с ручным отслеживанием индекса следующего свободного слота. Сравните с вариантом на `List<Book>`: что проще, где больше шансов на ошибку.
4. Добавьте класс `Library` со статическим списком всех книг и продемонстрируйте, что статическое состояние переживает создание/удаление локальных переменных в `Program.cs`. Объясните связь с идеей «статическое принадлежит классу».

---

## Statement in English / Постановка на английском

#### Context & motivation

Imagine you are building a small book-tracking module for a reading club. The club owns many books, and every book can be described with the same set of data: title, author, page count, publication year, and a "have I read it" flag. This is a textbook candidate for a class: the `Book` class becomes a blueprint from which you create as many concrete book objects as you need. Each object stores its own state (its own title, its own year), while sharing common behavior — the `MarkAsRead()`, `Describe()` methods and so on — with the other objects.

In parallel, the club wants to keep reading lists — monthly book selections. A list is also naturally modeled as a class: `ReadingList` holds the selection name and the collection of books themselves. One list can be filled with different book objects, and several lists exist independently of one another even though they are all built from the same `ReadingList` blueprint.

Finally, the club board wants to know how many books are registered in the system in total. That number does not belong to any single book — it belongs to the whole `Book` class. This is exactly the situation in the lesson where the static field `House.TotalBuilt` was used. By contrasting an instance field (for example `Pages`, different for every book) with a static one (`TotalRegistered`, shared by the class), you will feel the boundary between "belongs to the object" and "belongs to the class" in your own code — the very core of the lesson.

#### What to do step by step

1. Create a fresh .NET 8 console project named `M04.L01.Homework`:
   ```
   dotnet new console -n M04.L01.Homework -f net8.0
   cd M04.L01.Homework
   ```
   Verify that `M04.L01.Homework.csproj` contains `<TargetFramework>net8.0</TargetFramework>` and that the language is C# 12 (the default for .NET 8). Open `Program.cs` — it will be your entry point.

2. Declare a `Book` class in a `Book.cs` file under the `M04.L01.Homework` namespace. The class name must be strictly `PascalCase`. Inside, describe: public fields `Title`, `Author`, `Pages`, `YearPublished` with appropriate types (`string`, `int`), a public `bool` field `IsRead`, a static `int` field `TotalRegistered`, a constructor `Book(string title, string author, int pages, int yearPublished)` that populates the fields and increments `TotalRegistered`, and methods `MarkAsRead()` (sets `IsRead = true`) and `Describe()` (prints a book summary through `Console.WriteLine`).

3. Declare a `ReadingList` class in `ReadingList.cs` under the same namespace. Fields: a public `Name` of type `string` and a public `Books` field — either a `Book[]` array or a `List<Book>` if you are already comfortable with it. Add a constructor `ReadingList(string name)` that sets `Name` and initializes an empty array/list, a method `Add(Book book)` that adds a book, and a method `Print()` that prints the list name and calls `Describe()` on every book.

4. In `Program.cs`, create at least four `Book` objects via the constructor — for example books by Tolstoy, Orwell, Clarke, and Sapkowski. Demonstrate that they are independent objects: flip `IsRead` on one book by calling `MarkAsRead()` and confirm that the other books remain unread — print `IsRead` for each book to the console.

5. Create two `ReadingList` objects with different names (for example "Classics" and "Sci-Fi"). Add two books to the first list and two different books to the second. Call `Print()` on each list and confirm they are independent.

6. Show the static counter: after creating all books, print `Book.TotalRegistered`. Access it through the class name, not through an object. Create one more book and print `TotalRegistered` again — the value must increase.

7. Run the project with `dotnet run` from the project folder. Capture the expected output: for each book — a `Describe()` line; for the lists — a header and the descriptions of their books; for the counter — the final number.

8. Deliberately demonstrate one common mistake from the lesson as a comment in the code (do not run it): a commented line such as `// var x = Book.Title; // Error: accessing an instance field via the class name` with an explanation of why it is wrong and what the correct form is (`someBook.Title`).

#### Requirements

The solution must compile without error-level warnings under .NET 8 / C# 12 and run with `dotnet run`. Use `new` for every object: a variable without `new` (for example `Book b;`) does not create an object, and accessing its members would throw a `NullReferenceException` — there must be no such spots in the working code. Classes must live in the `M04.L01.Homework` namespace and be split across `Book.cs`, `ReadingList.cs`, and `Program.cs`.

All class names, public fields, and methods use `PascalCase` (`Book`, `ReadingList`, `MarkAsRead`, `TotalRegistered`); constructor parameter and local variable names use `camelCase` (`title`, `firstList`, `fantasyBook`). Name methods with verbs or verb phrases (`MarkAsRead`, `Add`, `Print`, `Describe`) and classes with nouns (`Book`, `ReadingList`).

Each `Book` object must demonstrate its own state: a distinct `Title`, `Author`, `Pages`, `YearPublished`, and an independent `IsRead`. The static field `TotalRegistered` must change only when a new object is created through the constructor and must be reachable as `Book.TotalRegistered`. The output must make it visible that an instance field is unique per object while a static field is shared across the whole class.

#### Pitfalls

- **Do not access an instance field through the class name.** `Book.Title` will not compile or makes no sense — `Title` belongs to a specific object. The correct form is `someBook.Title`. Only `static` members such as `Book.TotalRegistered` are reachable through the class name.
- **Do not forget `new`.** `Book b;` without `new` does not create an object. For a reference type the variable stays `null`, and `b.Describe()` throws `NullReferenceException`. Every object must be born via `new Book(...)` or an object initializer `new Book { ... }`.
- **Tell the class and the object apart.** The phrase "the class stores the title" is conceptually wrong: a class is a blueprint, objects store data. The class has no `Title` of its own; it has a `Title` field whose value lives separately in each object.
- **A static field is one per class.** `TotalRegistered` is incremented in the constructor, so it grows with every object created. If you accidentally drop the `static` keyword, every object gets its own independent counter and the "shared by the class" idea breaks.
- **Lowercase names violate conventions.** `class book` compiles but contradicts the C# style and hurts the team. Use `PascalCase` for classes and public members, `camelCase` for locals and parameters.
- **Objects are independent.** Changing `IsRead` on one book must not affect the others. If you accidentally make the field `static`, marking one book as read flips the flag on every book — a common and nasty bug that directly illustrates the danger of confusing static and instance.
- **SRP.** The `Book` class describes a book; `ReadingList` describes a selection. Do not push list-printing logic into `Book` — that is the responsibility of `ReadingList`.

#### Acceptance criteria

- [ ] The `M04.L01.Homework` project is created with `dotnet new console -f net8.0` and builds without errors.
- [ ] `dotnet run` prints the expected lines without exceptions.
- [ ] The `Book` class is declared in `Book.cs` under the `M04.L01.Homework` namespace with a `PascalCase` name.
- [ ] `Book` has fields `Title`, `Author`, `Pages`, `YearPublished`, `IsRead` with appropriate types.
- [ ] `Book` has a static field `TotalRegistered` and a constructor that increments it.
- [ ] The methods `MarkAsRead()` and `Describe()` are implemented and invoked.
- [ ] The `ReadingList` class is declared in `ReadingList.cs` with `Name` and a book collection.
- [ ] `ReadingList` has a constructor and `Add(Book)` and `Print()` methods.
- [ ] At least four `Book` objects are created via `new`.
- [ ] Object independence is demonstrated: `MarkAsRead()` changes only one book.
- [ ] At least two `ReadingList` objects with different books are created and shown to be independent.
- [ ] `Book.TotalRegistered` is printed before and after creating an additional book and grows correctly.
- [ ] The static field is accessed via the class name; instance fields are accessed via an object variable.
- [ ] Naming conventions are followed: `PascalCase` for classes and public members, `camelCase` for locals and parameters.
- [ ] The code contains a commented example of a typical mistake with an explanation.

#### Hints (no direct answer)

- Consider which type fits `Books` in `ReadingList` best: a fixed-size array requires knowing the size up front, while `List<Book>` from `System.Collections.Generic` grows on demand. If you have not studied generics in depth yet, start with an array and a "next free slot" index.
- To prove object independence, create a book, call `MarkAsRead()`, and immediately print `IsRead` for that book and for another one. The difference in values is the proof.
- Increment the static counter right inside the constructor body — that guarantees every `new Book(...)` is counted.
- For `Describe()`, use string interpolation: `$"Book \"{Title}\" — {Author}, {Pages} pp., {YearPublished}"`.
- Do not try to print the whole list from the `Book` class — that violates SRP; list printing belongs to `ReadingList.Print()`.

#### Reference solution walk-through

```csharp
// Book.cs — C# 12 / .NET 8
// Class as a book blueprint; objects are concrete books
namespace M04.L01.Homework;

public class Book
{
    // Instance fields — one set per object
    public string Title;
    public string Author;
    public int Pages;
    public int YearPublished;
    public bool IsRead;

    // Static field — one for the whole class
    public static int TotalRegistered;

    // Constructor runs on every new Book(...)
    public Book(string title, string author, int pages, int yearPublished)
    {
        Title = title;              // camelCase param -> PascalCase field
        Author = author;
        Pages = pages;
        YearPublished = yearPublished;
        IsRead = false;             // explicit default
        TotalRegistered++;          // shared counter grows
    }

    public void MarkAsRead() => IsRead = true;   // behavior that changes object state

    public void Describe()
    {
        string status = IsRead ? "read" : "unread";
        Console.WriteLine($"Book \"{Title}\" — {Author}, {Pages} pp., {YearPublished} ({status})");
    }
}
```

```csharp
// ReadingList.cs
namespace M04.L01.Homework;

public class ReadingList
{
    public string Name;
    private readonly List<Book> _books = new();   // encapsulated collection

    public ReadingList(string name) => Name = name;

    public void Add(Book book) => _books.Add(book);

    public void Print()
    {
        Console.WriteLine($"List \"{Name}\", books: {_books.Count}");
        foreach (var book in _books)   // shared behavior operating on each book's data
            book.Describe();
    }
}
```

```csharp
// Program.cs — top-level statements, C# 12
using M04.L01.Homework;

var warAndPeace = new Book("War and Peace", "L. Tolstoy", 1225, 1869);
var nineteen84  = new Book("1984", "G. Orwell", 328, 1949);
var spaceOdyssey = new Book("2001: A Space Odyssey", "A. Clarke", 312, 1968);
var witcher     = new Book("The Last Wish", "A. Sapkowski", 336, 1993);

// Demonstrating object independence
nineteen84.MarkAsRead();
Console.WriteLine($"1984 read: {nineteen84.IsRead}");        // True
Console.WriteLine($"War and Peace read: {warAndPeace.IsRead}"); // False

// Static counter via the class name
Console.WriteLine($"Books registered: {Book.TotalRegistered}"); // 4

var extra = new Book("Dune", "F. Herbert", 688, 1965);
Console.WriteLine($"Books registered: {Book.TotalRegistered}"); // 5

// Two independent lists
var classics = new ReadingList("Classics");
classics.Add(warAndPeace);
classics.Add(nineteen84);

var fantasy = new ReadingList("Sci-Fi");
fantasy.Add(spaceOdyssey);
fantasy.Add(witcher);

classics.Print();
fantasy.Print();

// Common mistake (intentionally commented) — accessing an instance field via the class name
// var bad = Book.Title; // Error. Correct: someBook.Title
```

Line-by-line walk-through. The `Book` class is declared as `public class Book` — `PascalCase`, as the lesson requires — and its body inside the curly braces holds both data (fields) and behavior (methods). The fields `Title`, `Author`, `Pages`, `YearPublished`, and `IsRead` are instance fields: each object gets its own set of values, which directly illustrates the lesson's claim that "an object is a built house with its own state". The field `TotalRegistered` is marked `static`, so there is exactly one copy for the whole class — the analog of `House.TotalBuilt` from the lesson — and it is accessed as `Book.TotalRegistered`, never through an object variable. The constructor populates the fields and increments the counter: every `new Book(...)` is accounted for, modeling the idea that "the counter grows each time a house is built". Constructor parameters are `camelCase` (`title`, `author`) while fields are `PascalCase` (`Title`, `Author`): this matches the naming conventions from the lesson. The method `MarkAsRead()` mutates a specific object's state, and `Describe()` prints it; the behavior is shared by the class but operates on its own object's data — exactly like `OpenDoor()` on `House`. In `Program.cs`, four independent objects are created; calling `MarkAsRead()` on `nineteen84` leaves `warAndPeace.IsRead == false`, proving state independence. The static counter is printed before and after the fifth book is created and grows from 4 to 5. The two `ReadingList` objects are independent even though built from the same blueprint. The commented line `Book.Title` captures the common mistake from the lesson — accessing an instance field through the class name — alongside the correct alternative.

#### Going deeper (bonus)

1. Add a static method `Book.HowManyRegistered()` that returns `TotalRegistered` and call it instead of touching the field directly. Reflect on why a method is preferable to a public static field from an encapsulation standpoint.
2. Add an `int` field `Rating` to `Book` and a method `SetRating(int value)` that rejects values outside 1–5 (prints a warning and leaves the field unchanged). This is a bridge to the properties topic of the next lesson.
3. Rewrite `ReadingList` so it stores books in a fixed-length array with manual tracking of the next free slot index. Compare it with the `List<Book>` version: which is simpler, where is there more room for error.
4. Add a `Library` class with a static list of all books and demonstrate that static state outlives the creation and disposal of local variables in `Program.cs`. Explain the connection to the "static belongs to the class" idea.

---

#### Чек-лист сдачи / Submission checklist

- [ ] Проект собирается командой `dotnet build` без ошибок.
- [ ] `dotnet run` выводит ожидаемый результат.
- [ ] Файлы `Book.cs`, `ReadingList.cs`, `Program.cs` лежат в `M04.L01.Homework`.
- [ ] Классы в `PascalCase`, локальные переменные и параметры — в `camelCase`.
- [ ] Создано ≥4 объектов `Book` и ≥2 объекта `ReadingList`.
- [ ] Демонстрируется независимость объектов и рост `Book.TotalRegistered`.
- [ ] В коде есть закомментированный пример типичной ошибки с пояснением.
- [ ] The project builds with `dotnet build` without errors.
- [ ] `dotnet run` prints the expected output.
- [ ] `Book.cs`, `ReadingList.cs`, `Program.cs` live in `M04.L01.Homework`.
- [ ] Classes use `PascalCase`; locals and parameters use `camelCase`.
- [ ] At least 4 `Book` objects and 2 `ReadingList` objects are created.
- [ ] Object independence and the growth of `Book.TotalRegistered` are demonstrated.
- [ ] The code contains a commented example of a typical mistake with an explanation.

#### Ресурсы / Resources

- [Microsoft Learn — Object-oriented programming (C#)](https://learn.microsoft.com/dotnet/csharp/fundamentals/object-oriented/)
- [Microsoft Learn — Classes (C# Programming Guide)](https://learn.microsoft.com/dotnet/csharp/fundamentals/types/classes)
- [Microsoft Learn — Objects and classes](https://learn.microsoft.com/dotnet/csharp/fundamentals/object-oriented/objects)
- [Microsoft Learn — Static and instance members](https://learn.microsoft.com/dotnet/csharp/programming-guide/classes-and-structs/static-classes-and-static-class-members)
- [Microsoft Learn — `dotnet new console`](https://learn.microsoft.com/dotnet/core/tools/dotnet-new-console)

---
[← К уроку M04-L01](lesson-M04-L01-class-vs-object.md) | [⬆ К модулю M04](../README.md) | [Следующее ДЗ →](homework-M04-L02-properties.md)
