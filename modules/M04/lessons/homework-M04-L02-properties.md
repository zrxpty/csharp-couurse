---
[← К уроку M04-L02](lesson-M04-L02-properties.md) | [⬆ К модулю M04](../README.md) | [Следующее ДЗ →](homework-M04-L03-constructors.md)
---

### Домашнее задание M04-L02: Поля, свойства (auto-properties, init-only, required) / Homework M04-L02: Fields, properties (auto-properties, init-only, required)

**Урок / Lesson:** M04-L02
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 3/5
**Цель / Goal:** (RU) Спроектировать доменную модель сервиса электронных книг на C# 12 / .NET 8, правильно комбинируя автосвойства, `init`-аксессоры, модификатор `required`, backing-field с валидацией в `set` и expression-bodied computed-свойства, и гарантировать, что объект нельзя привести в невалидное состояние ни при создании, ни позже. (EN) Design a domain model for an e-book service on C# 12 / .NET 8, correctly combining auto-properties, the `init` accessor, the `required` modifier, a backing field with validation in `set`, and expression-bodied computed properties, guaranteeing the object can never enter an invalid state — neither at construction nor afterwards.

#### Связь с уроком / Connection to the lesson
(RU) Задание напрямую закрепляет все шесть групп примеров из урока M04-L02: автосвойства (`get; set;` и `get;`-only с инициализатором), `init`-аксессоры (C# 9), `required`-свойства (C# 11), backing-field с валидацией в `set` и expression-bodied `get`, а также конструктор с `[SetsRequiredMembers]`. Особое внимание уделено частым ошибкам урока: публичным полям как антипаттерну, `private set` вместо `init`, открытому `List<T>` через `get; set;` и преждевременному использованию ключевого слова `field`, доступного лишь в C# 13+. (EN) The task directly reinforces all six groups of examples from lesson M04-L02: auto-properties (`get; set;` and `get;`-only with an initializer), the `init` accessor (C# 9), `required` properties (C# 11), a backing field with validation in `set` and an expression-bodied `get`, and a constructor decorated with `[SetsRequiredMembers]`. Special attention is paid to the lesson's common mistakes: public fields as an anti-pattern, `private set` instead of `init`, exposing `List<T>` through `get; set;`, and the premature use of the `field` keyword that is only available in C# 13+.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация
Вы присоединились к команде сервиса электронных книг «BookHub». Текущий прототип доменной модели написан наспех: сущности `Author`, `Book` и `ReaderProfile` состоят из публичных полей и свойств с `private set`, без какой-либо валидации. В результате в базу регулярно попадают объекты с отрицательной ценой, пустым или `null`-значением ISBN, нулевым количеством страниц и неинициализированным именем автора. Состояние объекта становится неконсистентным, баги всплывают только в рантайме, а многопоточный код страдает из-за того, что данные можно поменять в любой момент жизни объекта.

Техлид просит вас переписать доменную модель на C# 12 / .NET 8, используя современные механизмы свойств так, чтобы модель сама защищала свои инварианты. От вас требуется: для обычных изменяемых данных применять автосвойства `get; set;`; для данных, которые нельзя менять после создания, — `init`; для обязательных полей — `required`; для числовых значений с бизнес-ограничениями — backing-field с проверкой `value` в `set` и понятным исключением; для вычисляемых характеристик — expression-bodied свойства без `set`. Коллекции должны отдаваться как `IReadOnlyList<T>`, чтобы вызывающий код не мог их пересоздать или очистить. Ключевое слово `field` (semi-auto properties) использовать нельзя — это фича C# 13+/.NET 9, а проект собран на .NET 8.

Цель — получить модель, в которой компилятор ловит большинство ошибок ещё на этапе сборки (`required`), а оставшиеся инварианты проверяются в `set` в момент присвоения. Никаких «открытых сейфов в коридоре» — только «сейфы за стойкой администратора».

#### Что нужно сделать (пошагово)
1. Создайте новый консольный проект на .NET 8. Выполните в пустой папке команду `dotnet new console -n BookHub -o BookHub --framework net8.0`, затем перейдите внутрь: `cd BookHub`. Убедитесь, что в `BookHub.csproj` указано `<TargetFramework>net8.0</TargetFramework>` и при необходимости добавьте `<LangVersion>latest</LangVersion>`, чтобы включить все возможности C# 12.
2. Удалите автосгенерированный `Program.cs` и создайте вместо него четыре файла: `Author.cs`, `Book.cs`, `ReaderProfile.cs` и `Program.cs`. Разделение по файлам поможет видеть границы типов.
3. В `Author.cs` опишите класс `Author` с обязательным свойством `Name` (тип `string`, `required`, `init`), необязательным `Biography` (тип `string?`, `init`) и computed-свойством `DisplayName` — expression-bodied, без `set`: если `Biography` задан, вернуть `$"{Name} — {Biography}"`, иначе просто `Name`. Покажите, что попытка создать `new Author()` без `Name` не компилируется.
4. В `Book.cs` опишите класс `Book` со следующими свойствами: `required string Title { get; init; }`, `required string Isbn { get; init; }`, `int PageCount { get; init; }` со значением по умолчанию `1` (валидация диапазона — в бонусной части), `decimal Price` с backing-field `_price` и валидацией в `set`: значение не может быть отрицательным, иначе бросать `ArgumentOutOfRangeException(nameof(value), "Цена не может быть отрицательной")`, и computed-свойство `FormattedPrice => $"{_price:C}"`. Добавьте коллекцию `Tags` типа `IReadOnlyList<string>` с инициализатором `= Array.Empty<string>();` и `init`-аксессором — пересоздать коллекцию снаружи нельзя.
5. В `ReaderProfile.cs` опишите класс `ReaderProfile` со свойствами: `required string Email { get; init; }`, `string? DisplayName { get; init; }`, изменяемое `int Rating` с backing-field `_rating` и валидацией в `set` (допустимый диапазон 1–5, иначе `ArgumentOutOfRangeException`), и computed-свойство `Level` типа `string`, вычисляемое через pattern matching: `Rating >= 5 ? "Expert" : Rating >= 3 ? "Regular" : "Novice"`. Добавьте конструктор `[SetsRequiredMembers] ReaderProfile(string email)`, который присваивает `Email`, и покажите, что после него объектный инициализатор не требует повторного указания `Email`.
6. В том же файле или в `BadExample.cs` оставьте класс-антипример `BadBook` с публичным полем `public decimal Price;` и кратким комментарием, почему так делать нельзя (нет валидации, нет инкапсуляции, нет бинарной совместимости).
7. В `Program.cs` используйте top-level statements. Создайте легитимные объекты через инициализаторы, продемонстрируйте `init` (попытка `book.Price = -1m;` должна бросить исключение в `try/catch`), покажите `required` (закомментированная строка с ошибкой компиляции), проверьте `Level` для разных `Rating`, а вывод оформите через raw string literal `"""`
8. Соберите и запустите проект: `dotnet build`, затем `dotnet run`. Ожидаемый вывод должен содержать строки вида `Alice — bio`, `978-...`, `10,00 ₽` (или символ валюты текущей культуры), `Expert`/`Regular`/`Novice`, а также сообщение о пойманном исключении при попытке задать отрицательную цену. Если вывод отличается — отладьте.
9. В конце `Program.cs` оставьте закомментированные строки-ловушки: `// var bad = new Book { Title = "x" };` (нет `Isbn` — `required`), `// book.PageCount = 100;` (`init` после создания), `// book.Tags = new List<string>();` (`IReadOnlyList` нельзя переназначить). Раскомментирование каждой должно давать ошибку компиляции — проверьте это вручную.

#### Требования к решению
- Все поля классов должны быть приватными; публичный доступ открывается только через свойства. Публичные поля допускаются исключительно в классе-антипримере `BadBook`, снабжённом комментарием-объяснением.
- Обычные изменяемые данные используют автосвойство `get; set;`. Неизменяемые после создания данные используют `init`, а не `private set`. Обязательные свойства помечены `required` и обычно сочетаются с `init`.
- Для числовых свойств с бизнес-инвариантами (`Price`, `Rating`) реализован явный backing-field, а в `set` проверяется `value` и бросается `ArgumentOutOfRangeException` или `ArgumentException` с понятным сообщением, содержащим `nameof(value)`.
- Вычисляемые свойства (`DisplayName`, `FormattedPrice`, `Level`) объявлены expression-bodied (`=>`) и не имеют `set`.
- Коллекции отдаются как `IReadOnlyList<T>` с `init`-аксессором и инициализатором `= Array.Empty<T>();`, либо как `get;`-only автосвойство с инициализатором `= new()`. Никакого `public List<T> Tags { get; set; }`.
- В `ReaderProfile` присутствует конструктор с атрибутом `[System.Diagnostics.CodeAnalysis.SetsRequiredMembers]`, который присваивает все `required`-свойства, после чего объектный инициализатор может их не указывать.
- Проект собирается на `net8.0` без ошибок и предупреждений, связанных с моделью. Ключевое слово `field` не используется нигде — это фича C# 13+.
- В `Program.cs` используется top-level statements; pattern matching, collection expressions и raw string literals применяются там, где они уместны.

#### Тонкости и подводные камни
- **Не путайте `init` и `private set`.** `private set` формально «запрещает» запись снаружи, но инициализатор объекта `new Book { Title = "x" }` с `private set` не скомпилируется — а именно инициализаторы дают самый чистый синтаксис создания. `init` же разрешает запись в инициализаторе и в конструкторе, но запрещает её после. Для неизменяемых данных всегда выбирайте `init`.
- **`required` без `init` работает, но редко уместен.** `required string Email { get; set; }` заставит задать свойство в инициализаторе, но позволит поменять его позже — это редко то, что нужно для «обязательного при создании» поля. Чаще комбинация `required ... { get; init; }`.
- **`[SetsRequiredMembers]` не добавляет валидацию автоматически.** Атрибут лишь снимает требование компилятора указывать `required`-свойства в инициализаторе. Если в конструкторе вы присвоите `Email = null`, компилятор это пропустит. Поэтому либо добавляйте проверки в конструкторе, либо делайте `set`/`init` с валидацией (для `init` тоже можно писать тело с проверкой `value`).
- **Валидация только в конструкторе — недостаточна.** Если `set` публичный и без проверок, объект можно «испортить» позже: `book.Price = -1m;` сработает. Поэтому проверки должны жить именно в `set`, а конструктор лишь вызывает `set` косвенно через инициализатор.
- **Коллекции через `get; set;` — дыра.** `public List<string> Tags { get; set; }` позволяет вызывающему коду сделать `book.Tags = null;` или `book.Tags.Clear();`. Используйте `IReadOnlyList<T>` с `init` или `get;`-only автосвойство с инициализатором `= new()`, чтобы запретить переназначение, но разрешить рост через методы самой коллекции.
- **`field` в C# 12 не компилируется.** Semi-auto properties (`set => field = value;`) — это C# 13+/.NET 9. В .NET 8 используйте явный backing-field `private decimal _price;`. Не пытайтесь «сэкономить» — проект не соберётся.
- **`nameof(value)` в сообщении исключения.** Внутри `set` параметр всегда называется `value`; `nameof(value)` даёт строку `"value"`, что удобно для диагностики. Не пишите литерал `"value"` руками — при рефакторинге он рассинхронизируется.

#### Критерии приёмки
- [ ] Проект `BookHub` создаётся командой `dotnet new console --framework net8.0` и собирается без ошибок.
- [ ] Все поля доменных классов приватны; публичный доступ — только через свойства (кроме `BadBook`).
- [ ] Класс `Author` имеет `required string Name { get; init; }` и `string? Biography { get; init; }`.
- [ ] `Author.DisplayName` — expression-bodied computed-свойство без `set`.
- [ ] Класс `Book` имеет `required`-свойства `Title` и `Isbn` с `init`.
- [ ] `Book.Price` использует backing-field `_price` и бросает `ArgumentOutOfRangeException` при отрицательном значении.
- [ ] `Book.FormattedPrice` — expression-bodied computed-свойство, возвращающее значение в формате валюты.
- [ ] `Book.Tags` имеет тип `IReadOnlyList<string>` и инициализатор `Array.Empty<string>()`.
- [ ] Класс `ReaderProfile` имеет `required string Email { get; init; }` и изменяемое `int Rating` с валидацией 1–5.
- [ ] `ReaderProfile.Level` вычисляется через pattern matching и возвращает `"Expert"`/`"Regular"`/`"Novice"`.
- [ ] В `ReaderProfile` есть конструктор с `[SetsRequiredMembers]`, присваивающий `Email`.
- [ ] Класс `BadBook` содержит публичное поле `Price` и комментарий, объясняющий антипаттерн.
- [ ] В `Program.cs` используются top-level statements, raw string literal и хотя бы одна collection expression.
- [ ] `dotnet run` выводит данные книг, авторов и профилей читателей, а также сообщение о пойманном исключении при `Price = -1m`.
- [ ] Закомментированные строки-ловушки (`required`, `init` после создания, переназначение `IReadOnlyList`) при раскомментировании дают ошибку компиляции.
- [ ] Ключевое слово `field` нигде не используется (проект остаётся на C# 12 / .NET 8).

#### Подсказки (без прямого ответа)
- Подумайте, какое свойство «обязательно при создании, но неизменяемо после» — это прямой кандидат на `required ... { get; init; }`.
- Для `Price` вспомните аналог из урока: приватное поле + публичное свойство с телом в `set`. Expression-bodied `get` допустим, если тело состоит из одного выражения.
- Для `Level` используйте тернарный оператор или switch expression — оба варианта expression-bodied.
- Чтобы `Tags` нельзя было переназначить, но можно было наполнять, инициализируйте его конкретной коллекцией и откройте через интерфейс «только для чтения».
- В `Program.cs` оборачивайте опасные операции в `try/catch (ArgumentOutOfRangeException ex)` и печатайте `ex.Message` через raw string literal.
- Помните, что `init`-аксессор тоже может иметь тело с валидацией `value` — это пригодится для бонусной проверки `PageCount`.

#### Эталонное решение (разбор)
```csharp
// C# 12 / .NET 8 — доменная модель BookHub / BookHub domain model
using System;
using System.Collections.Generic;
using System.Diagnostics.CodeAnalysis;

// Антипример: публичное поле — нет валидации, нет инкапсуляции / Anti-example: public field
public class BadBook
{
    public decimal Price; // Любой может записать -5, и никто не возразит / Anyone can store -5 unchecked
}

// Автор: required + init + computed property / Author: required + init + computed property
public class Author
{
    public required string Name { get; init; }      // обязателен при создании / mandatory at construction
    public string? Biography { get; init; }         // необязателен / optional

    // Вычисляемое свойство без set / computed property, no set
    public string DisplayName => Biography is null ? Name : $"{Name} — {Biography}";
}

// Книга: required, init, backing-field с валидацией, IReadOnlyList / Book
public class Book
{
    public required string Title { get; init; }
    public required string Isbn { get; init; }
    public int PageCount { get; init; } = 1;        // значение по умолчанию / default value

    private decimal _price;                         // backing-field — приватное хранилище / private storage

    public decimal Price
    {
        get => _price;                              // expression-bodied get
        set
        {
            if (value < 0)
                throw new ArgumentOutOfRangeException(nameof(value),
                    "Цена не может быть отрицательной / Price cannot be negative");
            _price = value;
        }
    }

    public string FormattedPrice => $"{_price:C}";  // computed, expression-bodied

    // Коллекция только для чтения снаружи, с инициализатором / read-only collection, initialized
    public IReadOnlyList<string> Tags { get; init; } = Array.Empty<string>();
}

// Профиль читателя: required + init + mutable Rating с валидацией + конструктор / ReaderProfile
public class ReaderProfile
{
    public required string Email { get; init; }
    public string? DisplayName { get; init; }

    private int _rating;

    public int Rating
    {
        get => _rating;
        set
        {
            if (value is < 1 or > 5)
                throw new ArgumentOutOfRangeException(nameof(value),
                    "Рейтинг должен быть от 1 до 5 / Rating must be between 1 and 5");
            _rating = value;
        }
    }

    // Pattern matching для вычисляемого уровня / pattern matching for computed level
    public string Level => Rating switch
    {
        >= 5 => "Expert",
        >= 3 => "Regular",
        _    => "Novice"
    };

    // Конструктор снимает требование указывать Email в инициализаторе / ctor satisfies required
    [SetsRequiredMembers]
    public ReaderProfile(string email)
    {
        Email = email;
        _rating = 1; // дефолт через поле, минуя валидацию, т.к. 1 валиден / valid default via field
    }
}

// Program.cs — top-level statements / top-level statements
var author = new Author { Name = "Alice", Biography = "биография автора" };
var book = new Book
{
    Title = "C# in Depth",
    Isbn = "978-1617294532",
    Price = 10m,
    Tags = ["csharp", "programming"]   // collection expression → IReadOnlyList<string>
};
var reader = new ReaderProfile("alice@bookhub.io") { Rating = 4 };

try
{
    book.Price = -1m;  // бросит ArgumentOutOfRangeException / throws
}
catch (ArgumentOutOfRangeException ex)
{
    Console.WriteLine($"""
        Поймано исключение / Caught exception:
        {ex.Message}
        """);
}

Console.WriteLine($"""
    Автор / Author : {author.DisplayName}
    Книга / Book    : {book.Title} ({book.Isbn}), {book.FormattedPrice}, тегов: {book.Tags.Count}
    Читатель / Reader: {reader.Email} — {reader.Level} (рейтинг {reader.Rating})
    """);

// Ловушки компиляции (раскомментируйте для проверки) / Compile-time traps:
// var bad = new Book { Title = "x" };   // ошибка: нет required Isbn / error: missing required Isbn
// book.PageCount = 100;                 // ошибка: init после создания / error: init after construction
// book.Tags = new List<string>();       // ошибка: IReadOnlyList нельзя переназначить / error: cannot reassign
```

Разбор по строкам. Класс `BadBook` намеренно оставлен как антипример: публичное поле `Price` не даёт ни валидации, ни возможности позже заменить реализацию на свойство с проверкой без ломающих изменений — это прямая иллюстрация «открытого сейфа в коридоре» из урока. В `Author` свойство `Name` помечено `required` и `init`: компилятор не даст создать `new Author()` без `Name`, а после создания поменять его нельзя. `Biography` — `string?` с `init`, то есть необязательное неизменяемое поле. `DisplayName` — expression-bodied computed-свойство без `set`: здесь применён pattern matching `is null` из C# 7+, а само тело — одно выражение, что и даёт право на `=>`.

В `Book` сочетаются сразу несколько техник урока. `Title` и `Isbn` — `required ... { get; init; }`, то есть обязательны и неизменяемы. `PageCount` — `init`-свойство со значением по умолчанию `1`: в базовой версии валидация опущена, в бонусной её можно добавить через тело `init`-аксессора с проверкой `value`. `Price` — классический backing-field `_price` с валидацией в `set`: это та самая схема из урока, где проверка живёт именно в `set`, а не в конструкторе, потому что `set` вызывается и при инициализации, и при любом последующем присвоении. `FormattedPrice` — expression-bodied computed-свойство с форматом валюты `:C`. `Tags` имеет тип `IReadOnlyList<string>` с `init`-аксессором и инициализатором `Array.Empty<string>()`: это закрывает возможность переназначить коллекцию (`book.Tags = ...` не скомпилируется из-за `init` после создания) и при этом позволяет наполнять её через методы конкретной коллекции, если она будет передана.

В `ReaderProfile` показан сразу другой приём — конструктор с `[SetsRequiredMembers]`. Атрибут сообщает компилятору, что конструктор присваивает все `required`-свойства, поэтому в инициализаторе `new ReaderProfile("alice@bookhub.io") { Rating = 4 }` повторно указывать `Email` не нужно. Важно: атрибут лишь снимает требование инициализатора, он не валидирует значения — поэтому либо проверки ставят в `init`/`set`, либо в теле конструктора. `Rating` — изменяемое свойство с валидацией 1–5 через pattern matching `is < 1 or > 5` (C# 9 relational patterns). `Level` — expression-bodied switch expression (C# 8+), который вычисляет уровень по рейтингу. В конструкторе `_rating = 1` присваивается напрямую в поле, минуя `set`: это допустимо, потому что `1` заведомо валидно, но в более строгом коде лучше вызывать `Rating = 1`, чтобы проверка сработала. В `Program.cs` применены top-level statements, raw string literal `"""..."""` (C# 11) для многострочного вывода и collection expression `["csharp", "programming"]` (C# 12), который компилятор превращает в подходящую реализацию `IReadOnlyList<string>`. Блок `try/catch` демонстрирует, что валидация в `set` действительно бросает исключение, а не молча пропускает. Закомментированные строки-ловушки в конце — это «живые тесты» компилятора: их раскомментирование должно давать ошибку сборки, что подтверждает корректность использования `required`, `init` и `IReadOnlyList`.

#### Задания на углубление (бонус)
1. Добавьте валидацию в `init`-аксессор `PageCount`: значение должно быть положительным (`value < 1` → `ArgumentOutOfRangeException`). Убедитесь, что инициализатор `new Book { ..., PageCount = 0 }` бросает исключение в момент создания, а не позже. Подумайте, почему `init` с телом — это предпочтительнее проверки в конструкторе.
2. Сделайте `Author` неизменяемым через `record` вместо `class`, сохранив `required` и `init`. Сравните синтаксис: что стало короче, что осталось прежним. Наблюдение: `record` автоматически даёт value-равенство, но правила `required` и `init` не меняются.
3. Реализуйте `LibraryCatalog` — класс, который хранит `IReadOnlyList<Book>` и предоставляет computed-свойство `TotalValue => _books.Sum(b => b.Price);` (используйте `System.Linq`). Подумайте, должно ли `_books` быть `IReadOnlyList<Book>` или `List<Book>`, и почему внутреннее поле может быть изменяемым, а публичное свойство — нет.
4. Добавьте в `ReaderProfile` свойство-коллекцию `FavoriteTags` типа `IReadOnlyList<string>` с `init`-аксессором и инициализатором `= Array.Empty<string>();`. Покажите, что можно сделать `reader.FavoriteTags = ["csharp", "fsharp"];` в момент создания, но нельзя — после. Объясните, почему `init`-коллекция безопаснее `get; set;`-коллекции.

---

## Statement in English / Постановка на английском

#### Context & motivation
You have just joined the team of the e-book service "BookHub". The current prototype of the domain model was thrown together in a hurry: the `Author`, `Book`, and `ReaderProfile` entities are built from public fields and properties with `private set`, with no validation whatsoever. As a result, the database regularly receives objects with a negative price, an empty or `null` ISBN, zero page count, and an uninitialised author name. The state of an object becomes inconsistent, bugs surface only at runtime, and the multithreaded code suffers because the data can be mutated at any point in the object's lifetime.

The tech lead asks you to rewrite the domain model on C# 12 / .NET 8, using the modern property mechanisms so that the model protects its own invariants. You are required to: use plain auto-properties `get; set;` for ordinary mutable data; use `init` for data that must not change after construction; mark mandatory fields `required`; use a backing field with `value` validation in `set` and a clear exception for numeric values with business constraints; and use expression-bodied properties without `set` for computed characteristics. Collections must be exposed as `IReadOnlyList<T>` so that calling code cannot reassign or clear them. The `field` keyword (semi-auto properties) must not be used — it is a C# 13+/.NET 9 feature, while the project targets .NET 8.

The goal is a model in which the compiler catches most mistakes at build time (`required`), and the remaining invariants are checked in `set` at the moment of assignment. No "open safes in the hallway" — only "safes behind the reception desk".

#### What to do step by step
1. Create a new console project on .NET 8. Run `dotnet new console -n BookHub -o BookHub --framework net8.0` in an empty folder, then enter it with `cd BookHub`. Verify that `BookHub.csproj` contains `<TargetFramework>net8.0</TargetFramework>` and, if necessary, add `<LangVersion>latest</LangVersion>` to unlock all C# 12 features.
2. Delete the auto-generated `Program.cs` and create four files in its place: `Author.cs`, `Book.cs`, `ReaderProfile.cs`, and `Program.cs`. Splitting types across files makes the boundaries clearer.
3. In `Author.cs`, declare a class `Author` with a mandatory property `Name` (type `string`, `required`, `init`), an optional `Biography` (type `string?`, `init`), and a computed property `DisplayName` — expression-bodied, with no `set`: if `Biography` is set, return `$"{Name} — {Biography}"`, otherwise just `Name`. Demonstrate that `new Author()` without `Name` does not compile.
4. In `Book.cs`, declare a class `Book` with the following properties: `required string Title { get; init; }`, `required string Isbn { get; init; }`, `int PageCount { get; init; }` with a default of `1` (range validation goes in the bonus part), `decimal Price` with a backing field `_price` and validation in `set`: the value cannot be negative, otherwise throw `ArgumentOutOfRangeException(nameof(value), "Price cannot be negative")`, and a computed property `FormattedPrice => $"{_price:C}"`. Add a collection `Tags` of type `IReadOnlyList<string>` with an initializer `= Array.Empty<string>();` and an `init` accessor — the collection cannot be reassigned from the outside.
5. In `ReaderProfile.cs`, declare a class `ReaderProfile` with properties: `required string Email { get; init; }`, `string? DisplayName { get; init; }`, a mutable `int Rating` with a backing field `_rating` and validation in `set` (valid range 1–5, otherwise `ArgumentOutOfRangeException`), and a computed property `Level` of type `string` computed with pattern matching: `Rating >= 5 ? "Expert" : Rating >= 3 ? "Regular" : "Novice"`. Add a constructor `[SetsRequiredMembers] ReaderProfile(string email)` that assigns `Email`, and demonstrate that after it the object initializer no longer requires `Email`.
6. In the same file or in `BadExample.cs`, keep an anti-example class `BadBook` with a public field `public decimal Price;` and a short comment explaining why this is wrong (no validation, no encapsulation, no binary compatibility).
7. In `Program.cs`, use top-level statements. Build legitimate objects through initializers, demonstrate `init` (an attempt `book.Price = -1m;` should throw inside a `try/catch`), show `required` (a commented-out line that would be a compile error), check `Level` for different `Rating` values, and format the output with a raw string literal `"""`.
8. Build and run the project: `dotnet build`, then `dotnet run`. The expected output should contain lines such as `Alice — bio`, `978-...`, `10.00 $` (or the currency symbol of the current culture), `Expert`/`Regular`/`Novice`, plus a message about the exception caught when a negative price was attempted. If the output differs, debug.
9. At the end of `Program.cs`, leave commented trap lines: `// var bad = new Book { Title = "x" };` (missing `Isbn` — `required`), `// book.PageCount = 100;` (`init` after construction), `// book.Tags = new List<string>();` (`IReadOnlyList` cannot be reassigned). Uncommenting each one must produce a compile error — verify this manually.

#### Requirements
- All fields of the domain classes must be private; public access is exposed only through properties. Public fields are allowed solely in the `BadBook` anti-example class, with an explanatory comment.
- Ordinary mutable data uses an auto-property `get; set;`. Data immutable after construction uses `init`, not `private set`. Mandatory properties are marked `required` and are usually combined with `init`.
- For numeric properties with business invariants (`Price`, `Rating`), an explicit backing field is implemented, and `set` validates `value` and throws `ArgumentOutOfRangeException` or `ArgumentException` with a clear message containing `nameof(value)`.
- Computed properties (`DisplayName`, `FormattedPrice`, `Level`) are declared expression-bodied (`=>`) and have no `set`.
- Collections are exposed as `IReadOnlyList<T>` with an `init` accessor and the initializer `= Array.Empty<T>();`, or as a `get;`-only auto-property with the initializer `= new()`. No `public List<T> Tags { get; set; }`.
- `ReaderProfile` contains a constructor decorated with `[System.Diagnostics.CodeAnalysis.SetsRequiredMembers]` that assigns all `required` properties, after which the object initializer may omit them.
- The project builds on `net8.0` without model-related errors or warnings. The `field` keyword is not used anywhere — it is a C# 13+ feature.
- `Program.cs` uses top-level statements; pattern matching, collection expressions, and raw string literals are applied where appropriate.

#### Pitfalls
- **Do not confuse `init` with `private set`.** `private set` technically "forbids" writes from the outside, but the object initializer `new Book { Title = "x" }` will not compile with `private set` — and initializers give the cleanest construction syntax. `init` permits writes in the initializer and in the constructor, but forbids them afterwards. For immutable data, always choose `init`.
- **`required` without `init` works but is rarely appropriate.** `required string Email { get; set; }` forces the property to be set in the initializer, but allows it to be changed later — rarely what you want for a "mandatory at construction" field. The typical combination is `required ... { get; init; }`.
- **`[SetsRequiredMembers]` does not add validation.** The attribute only removes the compiler's requirement to specify `required` properties in the initializer. If inside the constructor you assign `Email = null`, the compiler will accept it. So either add checks in the constructor, or write a `set`/`init` body with `value` validation (an `init` accessor may also have a body that checks `value`).
- **Validation only in the constructor is insufficient.** If `set` is public and unchecked, the object can be "corrupted" later: `book.Price = -1m;` will succeed. Checks must live in `set`, while the constructor merely invokes `set` indirectly through the initializer.
- **Collections via `get; set;` are a hole.** `public List<string> Tags { get; set; }` lets the caller do `book.Tags = null;` or `book.Tags.Clear();`. Use `IReadOnlyList<T>` with `init`, or a `get;`-only auto-property with the initializer `= new()`, to forbid reassignment but still allow growth through the collection's own methods.
- **`field` does not compile in C# 12.** Semi-auto properties (`set => field = value;`) are C# 13+/.NET 9. On .NET 8, use an explicit backing field `private decimal _price;`. Do not try to "save space" — the project will not build.
- **`nameof(value)` in the exception message.** Inside `set`, the parameter is always named `value`; `nameof(value)` yields the string `"value"`, which is convenient for diagnostics. Do not write the literal `"value"` by hand — it will desynchronize during refactoring.

#### Acceptance criteria
- [ ] The `BookHub` project is created with `dotnet new console --framework net8.0` and builds without errors.
- [ ] All fields of the domain classes are private; public access is only through properties (except `BadBook`).
- [ ] The `Author` class has `required string Name { get; init; }` and `string? Biography { get; init; }`.
- [ ] `Author.DisplayName` is an expression-bodied computed property with no `set`.
- [ ] The `Book` class has `required` properties `Title` and `Isbn` with `init`.
- [ ] `Book.Price` uses a backing field `_price` and throws `ArgumentOutOfRangeException` for negative values.
- [ ] `Book.FormattedPrice` is an expression-bodied computed property returning the value as currency.
- [ ] `Book.Tags` has type `IReadOnlyList<string>` and the initializer `Array.Empty<string>()`.
- [ ] The `ReaderProfile` class has `required string Email { get; init; }` and a mutable `int Rating` validated to 1–5.
- [ ] `ReaderProfile.Level` is computed with pattern matching and returns `"Expert"`/`"Regular"`/`"Novice"`.
- [ ] `ReaderProfile` has a constructor with `[SetsRequiredMembers]` that assigns `Email`.
- [ ] The `BadBook` class contains a public `Price` field and a comment explaining the anti-pattern.
- [ ] `Program.cs` uses top-level statements, a raw string literal, and at least one collection expression.
- [ ] `dotnet run` prints data about books, authors, and reader profiles, plus a message about the exception caught at `Price = -1m`.
- [ ] The commented trap lines (`required`, `init` after construction, reassigning `IReadOnlyList`) produce compile errors when uncommented.
- [ ] The `field` keyword is not used anywhere (the project remains on C# 12 / .NET 8).

#### Hints (no direct answer)
- Think about which property is "mandatory at construction but immutable afterwards" — that is a direct candidate for `required ... { get; init; }`.
- For `Price`, recall the lesson's analog: a private field plus a public property with a `set` body. An expression-bodied `get` is allowed when the body is a single expression.
- For `Level`, use a ternary or a switch expression — both variants are expression-bodied.
- To make `Tags` non-reassignable but still fillable, initialize it with a concrete collection and expose it through a "read-only" interface.
- In `Program.cs`, wrap dangerous operations in `try/catch (ArgumentOutOfRangeException ex)` and print `ex.Message` through a raw string literal.
- Remember that the `init` accessor can also have a body that validates `value` — this will help with the bonus check on `PageCount`.

#### Reference solution walk-through
```csharp
// C# 12 / .NET 8 — BookHub domain model
using System;
using System.Collections.Generic;
using System.Diagnostics.CodeAnalysis;

// Anti-example: public field — no validation, no encapsulation
public class BadBook
{
    public decimal Price; // Anyone can store -5 unchecked
}

// Author: required + init + computed property
public class Author
{
    public required string Name { get; init; }      // mandatory at construction
    public string? Biography { get; init; }         // optional

    // Computed property, no set
    public string DisplayName => Biography is null ? Name : $"{Name} — {Biography}";
}

// Book: required, init, backing field with validation, IReadOnlyList
public class Book
{
    public required string Title { get; init; }
    public required string Isbn { get; init; }
    public int PageCount { get; init; } = 1;        // default value

    private decimal _price;                         // backing field — private storage

    public decimal Price
    {
        get => _price;                              // expression-bodied get
        set
        {
            if (value < 0)
                throw new ArgumentOutOfRangeException(nameof(value),
                    "Price cannot be negative");
            _price = value;
        }
    }

    public string FormattedPrice => $"{_price:C}";  // computed, expression-bodied

    // Read-only collection from the outside, with initializer
    public IReadOnlyList<string> Tags { get; init; } = Array.Empty<string>();
}

// ReaderProfile: required + init + mutable Rating with validation + constructor
public class ReaderProfile
{
    public required string Email { get; init; }
    public string? DisplayName { get; init; }

    private int _rating;

    public int Rating
    {
        get => _rating;
        set
        {
            if (value is < 1 or > 5)
                throw new ArgumentOutOfRangeException(nameof(value),
                    "Rating must be between 1 and 5");
            _rating = value;
        }
    }

    // Pattern matching for the computed level
    public string Level => Rating switch
    {
        >= 5 => "Expert",
        >= 3 => "Regular",
        _    => "Novice"
    };

    // Constructor satisfies the required members
    [SetsRequiredMembers]
    public ReaderProfile(string email)
    {
        Email = email;
        _rating = 1; // valid default via the field, bypassing validation
    }
}

// Program.cs — top-level statements
var author = new Author { Name = "Alice", Biography = "author biography" };
var book = new Book
{
    Title = "C# in Depth",
    Isbn = "978-1617294532",
    Price = 10m,
    Tags = ["csharp", "programming"]   // collection expression → IReadOnlyList<string>
};
var reader = new ReaderProfile("alice@bookhub.io") { Rating = 4 };

try
{
    book.Price = -1m;  // throws ArgumentOutOfRangeException
}
catch (ArgumentOutOfRangeException ex)
{
    Console.WriteLine($"""
        Caught exception:
        {ex.Message}
        """);
}

Console.WriteLine($"""
    Author : {author.DisplayName}
    Book   : {book.Title} ({book.Isbn}), {book.FormattedPrice}, tags: {book.Tags.Count}
    Reader : {reader.Email} — {reader.Level} (rating {reader.Rating})
    """);

// Compile-time traps (uncomment to verify):
// var bad = new Book { Title = "x" };   // error: missing required Isbn
// book.PageCount = 100;                 // error: init after construction
// book.Tags = new List<string>();       // error: cannot reassign IReadOnlyList
```

Line-by-line walk-through. The `BadBook` class is intentionally left as an anti-example: the public field `Price` offers neither validation nor the ability to later replace the implementation with a checked property without breaking changes — a direct illustration of the lesson's "open safe in the hallway". In `Author`, the `Name` property is marked `required` and `init`: the compiler will not let you create `new Author()` without `Name`, and after construction it cannot be changed. `Biography` is a `string?` with `init`, an optional immutable field. `DisplayName` is an expression-bodied computed property with no `set`: it uses the `is null` pattern from C# 7+, and because the body is a single expression, the `=>` syntax is justified.

`Book` combines several lesson techniques at once. `Title` and `Isbn` are `required ... { get; init; }`, that is, mandatory and immutable. `PageCount` is an `init` property with a default of `1`: validation is omitted in the base version and can be added in the bonus part through an `init` accessor body that checks `value`. `Price` is the classic backing-field `_price` with validation in `set` — exactly the lesson's scheme, where the check lives in `set` rather than in the constructor, because `set` is invoked both during initialization and on any subsequent assignment. `FormattedPrice` is an expression-bodied computed property with the currency format `:C`. `Tags` has type `IReadOnlyList<string>` with an `init` accessor and the initializer `Array.Empty<string>()`: this closes the possibility of reassigning the collection (`book.Tags = ...` will not compile because of `init` after construction) while still allowing it to be filled through the methods of the concrete collection that was supplied.

`ReaderProfile` demonstrates a different technique — a constructor with `[SetsRequiredMembers]`. The attribute tells the compiler that the constructor assigns all `required` properties, so the initializer `new ReaderProfile("alice@bookhub.io") { Rating = 4 }` does not need to specify `Email` again. It is important to understand that the attribute only removes the initializer requirement; it does not validate values — so checks go either into `init`/`set` or into the constructor body. `Rating` is a mutable property validated to 1–5 via the `is < 1 or > 5` pattern (C# 9 relational patterns). `Level` is an expression-bodied switch expression (C# 8+) that computes the level from the rating. In the constructor, `_rating = 1` is assigned directly to the field, bypassing `set`: this is acceptable because `1` is obviously valid, but in stricter code it is better to write `Rating = 1` so the check runs. `Program.cs` uses top-level statements, the raw string literal `"""..."""` (C# 11) for multi-line output, and the collection expression `["csharp", "programming"]` (C# 12), which the compiler turns into a suitable `IReadOnlyList<string>` implementation. The `try/catch` block demonstrates that validation in `set` really does throw rather than silently accept. The commented trap lines at the end are "live compiler tests": uncommenting them should produce a build error, confirming the correct use of `required`, `init`, and `IReadOnlyList`.

#### Going deeper (bonus)
1. Add validation to the `init` accessor of `PageCount`: the value must be positive (`value < 1` → `ArgumentOutOfRangeException`). Verify that the initializer `new Book { ..., PageCount = 0 }` throws at construction time, not later. Reflect on why an `init` body is preferable to a check in the constructor.
2. Make `Author` immutable through a `record` instead of a `class`, preserving `required` and `init`. Compare the syntax: what got shorter, what stayed the same. Observation: a `record` gives you value equality for free, but the rules of `required` and `init` do not change.
3. Implement a `LibraryCatalog` — a class that stores `IReadOnlyList<Book>` and exposes a computed property `TotalValue => _books.Sum(b => b.Price);` (use `System.Linq`). Decide whether `_books` should be `IReadOnlyList<Book>` or `List<Book>`, and explain why the internal field may be mutable while the public property is not.
4. Add a `FavoriteTags` collection property of type `IReadOnlyList<string>` to `ReaderProfile`, with an `init` accessor and the initializer `= Array.Empty<string>();`. Show that `reader.FavoriteTags = ["csharp", "fsharp"];` works at construction but not afterwards. Explain why an `init` collection is safer than a `get; set;` collection.

---

#### Чек-лист сдачи / Submission checklist
- [ ] (RU) Проект `BookHub` собирается на `net8.0` без ошибок и предупреждений модели.
- [ ] (RU) Все поля доменных классов приватны; публичный доступ — только через свойства (кроме `BadBook`).
- [ ] (RU) `Author`, `Book`, `ReaderProfile` используют `required`, `init`, backing-field с валидацией и expression-bodied computed-свойства.
- [ ] (RU) Коллекции отданы как `IReadOnlyList<T>`, без `public List<T> { get; set; }`.
- [ ] (RU) В `ReaderProfile` есть конструктор с `[SetsRequiredMembers]`.
- [ ] (RU) `Program.cs` использует top-level statements, raw string literal и collection expression; `dotnet run` выводит ожидаемые строки и сообщение о пойманном исключении.
- [ ] (RU) Ключевое слово `field` нигде не используется (C# 12 / .NET 8).
- [ ] (EN) The `BookHub` project builds on `net8.0` with no model-related errors or warnings.
- [ ] (EN) All fields of the domain classes are private; public access is only through properties (except `BadBook`).
- [ ] (EN) `Author`, `Book`, `ReaderProfile` use `required`, `init`, a backing field with validation, and expression-bodied computed properties.
- [ ] (EN) Collections are exposed as `IReadOnlyList<T>`, with no `public List<T> { get; set; }`.
- [ ] (EN) `ReaderProfile` has a constructor with `[SetsRequiredMembers]`.
- [ ] (EN) `Program.cs` uses top-level statements, a raw string literal, and a collection expression; `dotnet run` prints the expected lines and a message about the caught exception.
- [ ] (EN) The `field` keyword is not used anywhere (C# 12 / .NET 8).

#### Ресурсы / Resources
- [Microsoft Learn — Properties](https://learn.microsoft.com/dotnet/csharp/programming-guide/classes-and-structs/properties)
- [Microsoft Learn — init (C# 9)](https://learn.microsoft.com/dotnet/csharp/language-reference/keywords/init)
- [Microsoft Learn — required (C# 11)](https://learn.microsoft.com/dotnet/csharp/language-reference/keywords/required)
- [Microsoft Learn — Auto-implemented properties](https://learn.microsoft.com/dotnet/csharp/programming-guide/classes-and-structs/auto-implemented-properties)
- [Microsoft Learn — SetsRequiredMembersAttribute](https://learn.microsoft.com/dotnet/api/system.diagnostics.codeanalysis.setsrequiredmembersattribute)
- [Microsoft Learn — Collection expressions (C# 12)](https://learn.microsoft.com/dotnet/csharp/language-reference/operators/collection-expressions)
- [Microsoft Learn — Raw string literals (C# 11)](https://learn.microsoft.com/dotnet/csharp/programming-guide/strings/#raw-string-literals)
