---
[← К уроку M03-L05](lesson-M03-L05-methods.md) | [⬆ К модулю M03](../README.md) | [Следующее ДЗ →](homework-M03-L06-ref-out-in-params.md)
---

### Домашнее задание M03-L05: Объявление методов, сигнатура, возвращаемые значения / Homework M03-L05: Declaring methods, signature, return values

**Урок / Lesson:** M03-L05
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 3/5
**Цель / Goal:** (RU) Научиться объявлять методы C# 12 с корректной сигнатурой, осознанно выбирать `static`/instance и `void`/return, применять ранний возврат, перегрузку по сигнатуре и принцип CQS, а также грамотно обрабатывать `null` и крайние случаи. (EN) Learn to declare C# 12 methods with a correct signature, deliberately choose `static`/instance and `void`/return, apply early return, signature-based overloading and the CQS principle, and handle `null` and edge cases properly.

#### Связь с уроком / Connection to the lesson
(RU) ДЗ закрепляет пять обязательных частей объявления метода, понятие сигнатуры и перегрузки по списку типов параметров, различие `static` и экземплярных методов, разделение команд (`void`) и запросов (`return`) по принципу CQS, а также практику раннего возврата и проверок-стражей для невалидных входов и `null` — ровно те best practices и частые ошибки, которые разобраны в уроке.
(EN) The homework reinforces the five mandatory parts of a method declaration, the notion of signature and overload resolution by parameter type list, the distinction between `static` and instance methods, the CQS separation of commands (`void`) and queries (`return`), and the practice of early return with guard clauses for invalid inputs and `null` — exactly the best practices and common mistakes covered in the lesson.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация
Представьте, что вы присоединились к команде, которая разрабатывает мини-сервис учёта просроченных книг для муниципальной библиотеки. Сейчас в кодовой базе царит хаос: логика расчёта штрафа размазана по нескольким процедурам, имена методов то `Total()`, то `calculate_fee`, некоторые методы одновременно вычисляют значение и печатают в консоль, а проверки на `null` и отрицательные числа сделаны лишь кое-где. Библиотекарь жалуется, что при вводе нулевого или отрицательного количества дней просрочки сервис выдаёт отрицательный штраф, а для книг без даты возврата программа падает с `NullReferenceException`.

Ваша задача — отрефакторить и дописать сервис так, чтобы каждый метод имел чёткую сигнатуру, решал одну задачу, следовал принципу CQS и использовал ранний возврат для крайних случаев. Вы будете работать с методами обоих видов — `static` (чистые вычисления, не зависящие от состояния экземпляра) и экземплярными (действия над конкретным объектом `Book`). Вы реализуете перегрузку методов: одна и та же операция расчёта штрафа будет принимать то число дней, то объект книги вместе с текущей датой. Вы научитесь различать сигнатуру (имя + типы параметров) и полный заголовок метода, а также поймёте, почему два метода с одинаковым именем, но разными именами параметров одного и того же типа — это не перегрузка, а ошибка компиляции. Это задание напрямую моделирует ситуации из урока: аналогия с рецептом на кухне здесь превращается в «рецепт расчёта штрафа», где параметры — ингредиенты, тело — шаги, а возвращаемое значение — готовое блюдо (сумма штрафа или категория просрочки).

#### Что нужно сделать (пошагово)
1. Создайте консольный проект .NET 8 с именем `LibraryFees`. Выполните в терминале команду `dotnet new console -n LibraryFees -o LibraryFees --framework net8.0`, перейдите в папку проекта `cd LibraryFees` и убедитесь, что сборка проходит без ошибок: `dotnet build`. Ожидаемый вывод — `Build succeeded` с нулевыми предупреждениями.
2. Удалите шаблонный код из `Program.cs` и оставьте только пустой top-level файл. Вся логика будет в отдельных файлах классов, а `Program.cs` станет точкой входа с демонстрацией.
3. Создайте файл `Book.cs` с классом `Book`. У книги должны быть свойства: `Title` (строка, `init`), `DueDate` (`DateTime?`, `init`, может быть `null`, если дата возврата ещё не назначена), `DailyRate` (`decimal`, `init`, ставка штрафа за один день просрочки, по умолчанию `5m`). Используйте `record class` или обычный `class` — на ваше усмотрение, но обосновать выбор в комментарии.
4. Создайте перечисление `OverdueSeverity` в файле `OverdueSeverity.cs` со значениями `None`, `Mild`, `Moderate`, `Severe`. Это пригодится для метода категоризации.
5. Создайте статический класс `FeeCalculator` в файле `FeeCalculator.cs`. Реализуйте в нём следующие методы:
   - `public static decimal CalculateLateFee(int daysOverdue, decimal dailyRate)` — вычисляет штраф как `daysOverdue * dailyRate`, с ранним возвратом `0m` для неположительного количества дней и неположительной ставки.
   - `public static decimal CalculateLateFee(Book book, DateTime today)` — перегрузка по сигнатуре: если `book is null`, верните `0m`; если `book.DueDate is null`, верните `0m` (книга ещё не выдана); иначе вычислите количество дней просрочки как `(today - book.DueDate.Value).Days` и делегируйте первой перегрузке, передав `book.DailyRate`.
   - `public static bool IsOverdue(Book book, DateTime today)` — возвращает `true`, если книга просрочена; использует ранний возврат для `null` и `null`-даты.
   - `public static OverdueSeverity CategorizeOverdue(Book book, DateTime today)` — возвращает категорию просрочки с помощью pattern matching: `None` если не просрочена, `Mild` до 7 дней, `Moderate` до 30 дней, `Severe` свыше 30 дней.
   - `public static string FormatReport(Book book, DateTime today)` — возвращает строку отчёта с названием книги, статусом и суммой штрафа; это «запрос» по CQS, без побочных эффектов.
6. Создайте экземплярный класс `ReceiptPrinter` в файле `ReceiptPrinter.cs` с методом `public void PrintReceipt(Book book, decimal fee)`. Это «команда» по CQS: она только печатает в консоль и ничего не возвращает. Объясните в комментарии, почему именно `void` и почему метод экземплярный, а не статический.
7. В `Program.cs` создайте несколько книг с разными сценариями: книга просрочена на 10 дней, книга просрочена на 45 дней, книга не просрочена, книга с `DueDate = null`, книга — `null`. Вызовите все методы и распечатайте результаты через `ReceiptPrinter`. Ожидаемый вывод должен содержать пять строк с корректными суммами и категориями.
8. Проверьте, что для книги с `null`-датой и для `null`-книги программа не падает, а выдаёт безопасные значения `0m` и `None`.
9. Запустите проект командой `dotnet run` и убедитесь, что вывод соответствует ожидаемому.
10. Добавьте XML-комментарии `///` хотя бы к одному публичному методу в `FeeCalculator`, чтобы попробовать документирование сигнатуры.

#### Требования к решению
- Целевая платформа: C# 12 и .NET 8. Используйте top-level statements в `Program.cs`, `init`-свойства, pattern matching (включая `is` и switch-выражение), `decimal` для денежных сумм.
- Каждый метод должен быть назван глаголом в PascalCase: `CalculateLateFee`, `IsOverdue`, `CategorizeOverdue`, `FormatReport`, `PrintReceipt`. Имя-существительное вроде `Fee()` считается ошибкой.
- Модификатор доступа указывайте явно для каждого метода (`public` или `private`), даже если действует значение по умолчанию — так намерение очевидно, как требует best practice из урока.
- `static` используйте осознанно: чистые вычисления в `FeeCalculator` — `static`, действие над состоянием печати — экземплярный метод в `ReceiptPrinter`.
- Соблюдайте CQS: `CalculateLateFee`, `IsOverdue`, `CategorizeOverdue`, `FormatReport` — запросы (возвращают значение, не меняют состояние); `PrintReceipt` — команда (`void`, печатает).
- Все методы с типом возврата должны иметь `return` на каждом пути выполнения; используйте ранний возврат для невалидных входов и `null`.
- Методы должны быть короткими (до ~20 строк) и решать одну задачу — одну ответственность.
- Перегрузки `CalculateLateFee` должны различаться списком типов параметров, а не только их именами.
- Программа не должна падать с `NullReferenceException` ни при одном входе из демонстрации.

#### Тонкости и подводные камни
- Сигнатура метода включает имя и типы параметров, но не их имена. Два метода `CalculateLateFee(int days, decimal rate)` и `CalculateLateFee(int overdue, decimal daily)` — это не перегрузка, а ошибка компиляции «Type already defines a member with the same parameter types». Различать перегрузки можно только типами: `int` vs `Book`.
- Метод с типом возврата обязан вернуть значение на всех путях. Если вы пишете `if-else` без `return` в одной ветке, компилятор выдаст ошибку. Ранний возврат решает это: первые строки — проверки-стражи с `return`, последняя строка — счастливый путь.
- `void` не означает «нет return»: `return;` без значения допустим и полезен для раннего выхода из команды. Но возвращать значение из `void`-метода нельзя.
- Для ссылочных типов параметров проверяйте `null` через `is null` (а не `== null`) — это идиоматично и устойчиво к перегруженному оператору `==`. Для `DateTime?` используйте `is null` или `HasValue`.
- `decimal` для денег обязателен: `double` накапливает ошибку округления, и штраф `0.1 + 0.2` даст `0.30000000000000004`. Литералы `decimal` пишутся с суффиксом `m`: `5m`, `0m`, `0.1m`.
- Не путайте `static` и экземплярный метод. Если логика не использует `this` и не читает поля экземпляра — делайте метод `static`. Вызов `this.CalculateLateFee(...)` для чистой функции — запах кода, прямо упомянутый в частых ошибках урока.
- Разделение команд и запросов (CQS) не догма, но сильно упрощает тестирование: запрос можно вызывать сколько угодно раз без побочных эффектов. Не делайте метод, который и возвращает штраф, и печатает чек — разбейте на два.
- Pattern matching через switch-выражение компактен, но следите за тем, чтобы покрыть все случаи: добавьте ветку `_` или убедитесь, что компилятор подтвердит полноту (exhaustiveness). Для `OverdueSeverity` это особенно важно — забытая ветка даст неверный результат.
- Слишком много параметров (6+) — антипаттерн из урока. Если метод разрастается до пяти-шести параметров, сгруппируйте их в `record`. В этом задании параметры держим в разумных пределах.
- `DateTime` для «сегодня» передавайте параметром, а не берите `DateTime.Now` внутри метода — так метод остаётся чистым и детерминированным, его легко тестировать с фиктивной датой.

#### Критерии приёмки
- [ ] Проект `LibraryFees` собирается командой `dotnet build` без ошибок и предупреждений.
- [ ] `Program.cs` использует top-level statements и не содержит `class Program` / `static void Main`.
- [ ] Класс `Book` имеет свойства `Title`, `DueDate` (как `DateTime?`), `DailyRate` со значениями по умолчанию.
- [ ] Перечисление `OverdueSeverity` содержит `None`, `Mild`, `Moderate`, `Severe`.
- [ ] `FeeCalculator` — статический класс; все его методы `public static`.
- [ ] Есть две перегрузки `CalculateLateFee`, различающиеся списком типов параметров (`int, decimal` и `Book, DateTime`).
- [ ] `IsOverdue` возвращает `bool` и использует ранний возврат для `null` и `null`-даты.
- [ ] `CategorizeOverdue` использует pattern matching и возвращает все четыре значения перечисления.
- [ ] `FormatReport` — запрос: возвращает строку и не печатает в консоль.
- [ ] `ReceiptPrinter.PrintReceipt` — команда `void`, печатает в консоль, не возвращает значение.
- [ ] Модификаторы доступа указаны явно для каждого метода.
- [ ] Все методы названы глаголами в PascalCase.
- [ ] Программа не падает на `null`-книге и книге с `null`-датой, а выдаёт безопасные значения.
- [ ] `dotnet run` выводит минимум пять строк с корректными суммами и категориями.
- [ ] Хотя бы один публичный метод снабжён XML-комментариями `///`.

#### Подсказки (без прямого ответа)
- Для расчёта количества дней между датами используйте операцию вычитания `DateTime`: результат — `TimeSpan`, у которого есть свойство `.Days`.
- В перегрузке с `Book` сначала проверьте `book is null`, затем `book.DueDate is null`, затем вызовите первую перегрузку — это и есть каскад ранних возвратов.
- Для `CategorizeOverdue` сначала вызовите `IsOverdue`; если `false` — верните `None`. Затем используйте switch-выражение по количеству дней.
- Помните: `decimal` литералы требуют суффикса `m`. `0` без `m` в методе, возвращающем `decimal`, не скомпилируется.
- В `PrintReceipt` используйте интерполяцию строк `$"..."` и формат `{fee:C}` для вывода суммы в денежном формате.

#### Эталонное решение (разбор)
```csharp
// C# 12 / .NET 8 — эталонное решение ДЗ M03-L05
// Reference solution for homework M03-L05
// Темы урока: сигнатура, перегрузка, static/instance, void/return, CQS, ранний возврат, null-проверки.

using System;

// Модель книги. Используем record class — неизменяемый, удобен для значений.
// Book model. We use a record class — immutable, convenient for value-like data.
public record class Book
{
    public required string Title { get; init; }
    public DateTime? DueDate { get; init; }        // null — книга ещё не выдана / null — not checked out yet
    public decimal DailyRate { get; init; } = 5m;   // ставка штрафа за день / per-day fee
}

// Категории просрочки / Overdue categories.
public enum OverdueSeverity { None, Mild, Moderate, Severe }

// Статический класс с чистыми вычислениями — все методы static, без состояния экземпляра.
// Static class with pure computations — all methods static, no instance state.
public static class FeeCalculator
{
    /// <summary>Считает штраф по количеству дней и ставке. / Computes fee from days and rate.</summary>
    public static decimal CalculateLateFee(int daysOverdue, decimal dailyRate)
    {
        if (daysOverdue <= 0) return 0m;     // ранний возврат: нет просрочки / early return: no overdue
        if (dailyRate <= 0m) return 0m;      // ранний возврат: некорректная ставка / early return: bad rate

        return daysOverdue * dailyRate;      // счастливый путь / happy path
    }

    // Перегрузка по сигнатуре: список типов (Book, DateTime) отличается от (int, decimal).
    // Signature overload: (Book, DateTime) differs from (int, decimal).
    public static decimal CalculateLateFee(Book book, DateTime today)
    {
        if (book is null) return 0m;                  // страж от null / null guard
        if (book.DueDate is null) return 0m;          // книга не выдана / not checked out

        int days = (today - book.DueDate.Value).Days;
        return CalculateLateFee(days, book.DailyRate); // делегируем первой перегрузке / delegate to first overload
    }

    public static bool IsOverdue(Book book, DateTime today)
    {
        if (book is null) return false;
        if (book.DueDate is null) return false;

        return today > book.DueDate.Value;            // счастливый путь / happy path
    }

    // Pattern matching через switch-выражение — компактно и выразительно.
    // Pattern matching via switch expression — compact and expressive.
    public static OverdueSeverity CategorizeOverdue(Book book, DateTime today)
    {
        if (!IsOverdue(book, today)) return OverdueSeverity.None;   // ранний возврат / early return

        int days = (today - book!.DueDate!.Value).Days;
        return days switch
        {
            <= 7  => OverdueSeverity.Mild,
            <= 30 => OverdueSeverity.Moderate,
            _     => OverdueSeverity.Severe             // ветка-заглушка для полноты / discard for exhaustiveness
        };
    }

    // Запрос по CQS: возвращает строку, не имеет побочных эффектов.
    // CQS query: returns a string, no side effects.
    public static string FormatReport(Book book, DateTime today)
    {
        if (book is null) return "Книга не указана / No book provided";
        if (book.DueDate is null) return $"{book.Title}: не выдана / not checked out";

        bool overdue = IsOverdue(book, today);
        decimal fee = CalculateLateFee(book, today);
        var severity = CategorizeOverdue(book, today);
        string status = overdue ? "просрочена / overdue" : "в срок / on time";

        return $"{book.Title}: {status}, штраф / fee: {fee:C}, категория / severity: {severity}";
    }
}

// Экземплярный класс с командой печати — действие над конкретным принтером.
// Instance class with a print command — an action tied to a concrete printer instance.
public class ReceiptPrinter
{
    // Команда void по CQS: только побочный эффект (печать), ничего не возвращает.
    // CQS void command: only a side effect (printing), returns nothing.
    public void PrintReceipt(Book book, decimal fee)
    {
        if (book is null)
        {
            Console.WriteLine("Чек не печатается: книга null / No receipt: book is null");
            return;                       // ранний выход из void / early exit from void
        }

        Console.WriteLine($"=== ЧЕК / RECEIPT ===");
        Console.WriteLine($"Книга / Book: {book.Title}");
        Console.WriteLine($"Штраф / Fee: {fee:C}");
        Console.WriteLine($"====================");
    }
}

// Точка входа — top-level statements, демонстрация всех сценариев.
// Entry point — top-level statements, demo of all scenarios.
var today = new DateTime(2024, 6, 1);

var books = new Book[]
{
    new() { Title = "Война и мир / War and Peace", DueDate = new DateTime(2024, 5, 22), DailyRate = 5m },   // 10 дней
    new() { Title = "Справочник / Reference",       DueDate = new DateTime(2024, 4, 17), DailyRate = 3m },   // 45 дней
    new() { Title = "На руках / Checked out",       DueDate = new DateTime(2024, 6, 5),  DailyRate = 5m },   // не просрочена
    new() { Title = "Без даты / No date",           DueDate = null,                       DailyRate = 5m },   // null-дата
    null                                                                                // null-книга
};

var printer = new ReceiptPrinter();
foreach (var book in books)
{
    string report = FeeCalculator.FormatReport(book, today);
    Console.WriteLine(report);

    decimal fee = FeeCalculator.CalculateLateFee(book, today);
    printer.PrintReceipt(book, fee);
}
```

Разбор по строкам. Класс `Book` сделан `record class` с `init`-свойствами — это подчёркивает неизменяемость и удобство для значений, а `required` для `Title` гарантирует, что книга не будет создана без названия. `DueDate` объявлен как `DateTime?`: знак вопроса — синтаксис nullable-значимых типов, прямо из темы урока про `null` для параметров и свойств. `DailyRate` имеет значение по умолчанию `5m` с суффиксом `m`, что критично для `decimal` и упоминается в тонкостях. Перечисление `OverdueSeverity` задаёт четыре категории — это будет возвращаемое значение метода-запроса.

`FeeCalculator` объявлен `static class`: все методы в нём `public static`, что соответствует best practice «предпочитайте static для методов, не работающих с состоянием экземпляра». Первая перегрузка `CalculateLateFee(int, decimal)` использует два ранних возврата: для неположительных дней и неположительной ставки — это пример проверок-стражей из урока, читается сверху вниз как список правил без вложенности. Вторая перегрузка `CalculateLateFee(Book, DateTime)` отличается списком типов параметров — это и есть корректная перегрузка по сигнатуре, что закрепляет понятие «сигнатура = имя + типы параметров, но не их имена». Внутри она проверяет `book is null` и `book.DueDate is null` через оператор `is`, как рекомендовано в тонкостях, и делегирует первой перегрузке — повторное использование кода без дублирования логики.

`IsOverdue` — метод-запрос, возвращающий `bool`, с ранними возвратами `false` для крайних случаев и одним выражением счастливого пути в конце. `CategorizeOverdue` демонстрирует pattern matching через switch-выражение с веткой `_` для полноты — это прямо отвечает пункту тонкостей про exhaustiveness. Сначала вызывается `IsOverdue`, и если книга не просрочена, метод сразу возвращает `None` — повторное использование ранее написанного запроса, а не дублирование проверки.

`FormatReport` — запрос по CQS: он собирает строку, вызывает другие запросы и ничего не печатает. Это контраст с `ReceiptPrinter.PrintReceipt` — экземплярной `void`-командой, которая только печатает. Разделение этих двух обязанностей иллюстрирует принцип CQS из урока: один метод считает и форматирует, другой печатает. В `PrintReceipt` показан ранний выход из `void` через `return;` без значения — деталь, упомянутая в подводных камнях. Наконец, точка входа использует top-level statements, коллекционные выражения `new Book[] { ... }` и цикл `foreach`, проходя по пяти сценариям, включая `null`-книгу и книгу с `null`-датой, — программа не падает благодаря стражам, что закрывает требование приёмки.

#### Задания на углубление (бонус)
1. Добавьте третью перегрузку `CalculateLateFee`, принимающую массив книг `Book[]` и возвращающую суммарный штраф. Используйте LINQ `Sum` и убедитесь, что перегрузка корректно различается по сигнатуре.
2. Реализуйте метод `ApplyGracePeriod`, который уменьшает количество дней просрочки на заданный льготный период (например, 2 дня), но не ниже нуля. Подумайте, должен ли он быть запросом или командой, и обоснуйте.
3. Перепишите `CategorizeOverdue` без switch-выражения, используя каскад `if` с ранним возвратом. Сравните читаемость и обсудите, какой вариант предпочтительнее и почему.
4. Добавьте кэширование вычисленного штрафа внутри экземплярного класса `ReceiptPrinter` (поле `Dictionary<Book, decimal>`). Объясните, почему это превращает часть логики в команду с состоянием и как это соотносится с CQS.

---

## Statement in English / Постановка на английском

#### Context & motivation
Imagine you have joined a team building a small overdue-book tracking service for a municipal library. The codebase is a mess: the fee calculation logic is smeared across several procedures, method names alternate between `Total()` and `calculate_fee`, some methods both compute a value and print to the console at once, and null and negative-input checks appear only here and there. The librarian complains that entering zero or a negative number of overdue days makes the service emit a negative fee, and for books without a due date the program crashes with a `NullReferenceException`.

Your task is to refactor and extend the service so that every method has a clear signature, solves a single task, follows the CQS principle, and uses early return for edge cases. You will work with both kinds of methods — `static` (pure computations that do not depend on instance state) and instance methods (actions tied to a concrete `Book` object). You will implement method overloading: the very same fee computation will accept either a number of days or a book object together with today’s date. You will learn to distinguish the signature (name plus parameter types) from the full method header, and you will understand why two methods with the same name but parameters that differ only in names — not in types — is not an overload but a compile error. This assignment directly mirrors the situations from the lesson: the kitchen-recipe analogy turns into a “fee-computing recipe”, where parameters are ingredients, the body is the sequence of steps, and the return value is the finished dish (the fee amount or the overdue category).

The exercise is intentionally close to the lesson’s running example of orders and discounts, but re-cast in the library domain so that you cannot copy-paste and must re-derive every design decision. You will repeatedly face the choice between `void` and a concrete return type, between `static` and instance, between a guard clause and a nested `if`. By the end you should be able to justify, for each method, why its signature, access modifier, and return shape are what they are, and why each complies with the best practices and avoids the common mistakes enumerated in the lesson.

#### What to do step by step
1. Create a .NET 8 console project named `LibraryFees`. In the terminal run `dotnet new console -n LibraryFees -o LibraryFees --framework net8.0`, then `cd LibraryFees`, and verify the build is clean: `dotnet build`. The expected output is `Build succeeded` with zero warnings.
2. Remove the template code from `Program.cs` and leave only an empty top-level file. All logic will live in separate class files; `Program.cs` will be the entry point with a demonstration.
3. Create `Book.cs` with a `Book` class. The book must have properties: `Title` (string, `init`), `DueDate` (`DateTime?`, `init`, may be `null` if the book has not been checked out yet), `DailyRate` (`decimal`, `init`, the per-day fee, default `5m`). Use a `record class` or a plain `class` at your discretion, but justify the choice in a comment.
4. Create an enumeration `OverdueSeverity` in `OverdueSeverity.cs` with values `None`, `Mild`, `Moderate`, `Severe`. It will be used by the categorization method.
5. Create a static class `FeeCalculator` in `FeeCalculator.cs`. Implement the following methods in it:
   - `public static decimal CalculateLateFee(int daysOverdue, decimal dailyRate)` — computes the fee as `daysOverdue * dailyRate`, with early return `0m` for non-positive days and non-positive rate.
   - `public static decimal CalculateLateFee(Book book, DateTime today)` — a signature overload: if `book is null`, return `0m`; if `book.DueDate is null`, return `0m` (the book has not been checked out); otherwise compute the number of overdue days as `(today - book.DueDate.Value).Days` and delegate to the first overload, passing `book.DailyRate`.
   - `public static bool IsOverdue(Book book, DateTime today)` — returns `true` if the book is overdue; uses early return for `null` and a `null` date.
   - `public static OverdueSeverity CategorizeOverdue(Book book, DateTime today)` — returns the overdue category using pattern matching: `None` if not overdue, `Mild` up to 7 days, `Moderate` up to 30 days, `Severe` over 30 days.
   - `public static string FormatReport(Book book, DateTime today)` — returns a report string with the book title, status, and fee amount; this is a CQS “query” with no side effects.
6. Create an instance class `ReceiptPrinter` in `ReceiptPrinter.cs` with a method `public void PrintReceipt(Book book, decimal fee)`. This is a CQS “command”: it only prints to the console and returns nothing. Explain in a comment why it is `void` and why it is instance rather than static.
7. In `Program.cs` create several books covering different scenarios: a book 10 days overdue, a book 45 days overdue, a book that is not overdue, a book with `DueDate = null`, and a `null` book. Call every method and print the results via `ReceiptPrinter`. The expected output must contain five lines with correct fees and categories.
8. Verify that for the book with a `null` date and for the `null` book the program does not crash, but yields safe values `0m` and `None`.
9. Run the project with `dotnet run` and confirm the output matches expectations.
10. Add `///` XML documentation comments to at least one public method in `FeeCalculator` to practice documenting the signature.

#### Requirements
- Target platform: C# 12 and .NET 8. Use top-level statements in `Program.cs`, `init`-only properties, pattern matching (including `is` and switch expressions), and `decimal` for monetary amounts.
- Every method must be named with a verb in PascalCase: `CalculateLateFee`, `IsOverdue`, `CategorizeOverdue`, `FormatReport`, `PrintReceipt`. A noun name like `Fee()` counts as an error.
- State the access modifier explicitly for every method (`public` or `private`), even when the default applies — intent must be obvious, as the lesson’s best practice demands.
- Use `static` deliberately: pure computations in `FeeCalculator` are `static`; the printing action tied to state is an instance method in `ReceiptPrinter`.
- Obey CQS: `CalculateLateFee`, `IsOverdue`, `CategorizeOverdue`, `FormatReport` are queries (return a value, do not mutate state); `PrintReceipt` is a command (`void`, prints).
- Every method with a return type must have a `return` on each execution path; use early return for invalid inputs and `null`.
- Methods must be short (~20 lines) and focused on a single task — one responsibility.
- The two overloads of `CalculateLateFee` must differ by their parameter type list, not just by parameter names.
- The program must not crash with a `NullReferenceException` on any input from the demonstration.

#### Pitfalls
- A method’s signature includes its name and parameter types but not parameter names. Two methods `CalculateLateFee(int days, decimal rate)` and `CalculateLateFee(int overdue, decimal daily)` are not overloads — they produce the compile error “Type already defines a member with the same parameter types”. Overloads can differ only by types: `int` vs `Book`.
- A method with a return type must return a value on every path. If you write an `if-else` with no `return` in one branch, the compiler errors out. Early return fixes this: the first lines are guard clauses with `return`, the last line is the happy path.
- `void` does not mean “no return”: a bare `return;` without a value is allowed and useful for an early exit from a command. But returning a value from a `void` method is illegal.
- For reference-typed parameters, check `null` with `is null` (not `== null`) — it is idiomatic and robust against an overloaded `==` operator. For `DateTime?`, use `is null` or `HasValue`.
- `decimal` is mandatory for money: `double` accumulates rounding error, and a fee like `0.1 + 0.2` yields `0.30000000000000004`. Write `decimal` literals with the `m` suffix: `5m`, `0m`, `0.1m`.
- Do not confuse `static` with instance methods. If a method’s logic does not use `this` and does not read instance fields, make it `static`. Calling `this.CalculateLateFee(...)` for a pure function is a code smell explicitly listed in the lesson’s common mistakes.
- Separating commands and queries (CQS) is not dogma, but it greatly simplifies testing: a query can be called any number of times without side effects. Do not write a method that both returns a fee and prints a receipt — split it in two.
- Switch-expression pattern matching is compact, but make sure you cover every case: add a `_` branch or confirm that the compiler acknowledges exhaustiveness. For `OverdueSeverity` this matters — a forgotten branch yields a wrong result.
- Too many parameters (6+) is the anti-pattern from the lesson. If a method grows to five or six parameters, group them into a `record`. In this assignment the parameter counts stay reasonable.
- Pass “today” as a parameter rather than reading `DateTime.Now` inside a method — this keeps the method pure and deterministic and makes it trivial to test with a fixed date.

#### Acceptance criteria
- [ ] The `LibraryFees` project builds with `dotnet build` without errors or warnings.
- [ ] `Program.cs` uses top-level statements and contains no `class Program` / `static void Main`.
- [ ] The `Book` class has `Title`, `DueDate` (as `DateTime?`), and `DailyRate` with a default value.
- [ ] The `OverdueSeverity` enumeration contains `None`, `Mild`, `Moderate`, `Severe`.
- [ ] `FeeCalculator` is a static class; all its methods are `public static`.
- [ ] There are two overloads of `CalculateLateFee` that differ by parameter type list (`int, decimal` and `Book, DateTime`).
- [ ] `IsOverdue` returns `bool` and uses early return for `null` and a `null` date.
- [ ] `CategorizeOverdue` uses pattern matching and returns all four enumeration values.
- [ ] `FormatReport` is a query: it returns a string and does not print to the console.
- [ ] `ReceiptPrinter.PrintReceipt` is a `void` command: it prints to the console and returns no value.
- [ ] Access modifiers are stated explicitly for every method.
- [ ] All methods are named with verbs in PascalCase.
- [ ] The program does not crash on a `null` book or a book with a `null` date, but yields safe values.
- [ ] `dotnet run` prints at least five lines with correct fees and categories.
- [ ] At least one public method is documented with `///` XML comments.

#### Hints (no direct answer)
- To compute the number of days between dates, subtract one `DateTime` from another: the result is a `TimeSpan` with a `.Days` property.
- In the `Book` overload, check `book is null` first, then `book.DueDate is null`, then call the first overload — this is the cascade of early returns.
- In `CategorizeOverdue`, call `IsOverdue` first; if it returns `false`, return `None`. Then use a switch expression on the number of days.
- Remember: `decimal` literals need the `m` suffix. A bare `0` in a method that returns `decimal` will not compile.
- In `PrintReceipt`, use string interpolation `$"..."` and the `{fee:C}` format to render the amount as currency.

#### Reference solution walk-through
```csharp
// C# 12 / .NET 8 — reference solution for homework M03-L05
// Topics covered: signature, overloading, static/instance, void/return, CQS, early return, null checks.

using System;

// Book model. We use a record class — immutable, convenient for value-like data.
public record class Book
{
    public required string Title { get; init; }
    public DateTime? DueDate { get; init; }        // null — not checked out yet
    public decimal DailyRate { get; init; } = 5m;   // per-day fee
}

// Overdue categories.
public enum OverdueSeverity { None, Mild, Moderate, Severe }

// Static class with pure computations — all methods static, no instance state.
public static class FeeCalculator
{
    /// <summary>Computes the fee from the number of days and the daily rate.</summary>
    public static decimal CalculateLateFee(int daysOverdue, decimal dailyRate)
    {
        if (daysOverdue <= 0) return 0m;     // early return: no overdue
        if (dailyRate <= 0m) return 0m;      // early return: bad rate

        return daysOverdue * dailyRate;      // happy path
    }

    // Signature overload: (Book, DateTime) differs from (int, decimal).
    public static decimal CalculateLateFee(Book book, DateTime today)
    {
        if (book is null) return 0m;                  // null guard
        if (book.DueDate is null) return 0m;          // not checked out

        int days = (today - book.DueDate.Value).Days;
        return CalculateLateFee(days, book.DailyRate); // delegate to the first overload
    }

    public static bool IsOverdue(Book book, DateTime today)
    {
        if (book is null) return false;
        if (book.DueDate is null) return false;

        return today > book.DueDate.Value;            // happy path
    }

    // Pattern matching via switch expression — compact and expressive.
    public static OverdueSeverity CategorizeOverdue(Book book, DateTime today)
    {
        if (!IsOverdue(book, today)) return OverdueSeverity.None;   // early return

        int days = (today - book!.DueDate!.Value).Days;
        return days switch
        {
            <= 7  => OverdueSeverity.Mild,
            <= 30 => OverdueSeverity.Moderate,
            _     => OverdueSeverity.Severe             // discard for exhaustiveness
        };
    }

    // CQS query: returns a string, no side effects.
    public static string FormatReport(Book book, DateTime today)
    {
        if (book is null) return "No book provided";
        if (book.DueDate is null) return $"{book.Title}: not checked out";

        bool overdue = IsOverdue(book, today);
        decimal fee = CalculateLateFee(book, today);
        var severity = CategorizeOverdue(book, today);
        string status = overdue ? "overdue" : "on time";

        return $"{book.Title}: {status}, fee: {fee:C}, severity: {severity}";
    }
}

// Instance class with a print command — an action tied to a concrete printer instance.
public class ReceiptPrinter
{
    // CQS void command: only a side effect (printing), returns nothing.
    public void PrintReceipt(Book book, decimal fee)
    {
        if (book is null)
        {
            Console.WriteLine("No receipt: book is null");
            return;                       // early exit from void
        }

        Console.WriteLine($"=== RECEIPT ===");
        Console.WriteLine($"Book: {book.Title}");
        Console.WriteLine($"Fee: {fee:C}");
        Console.WriteLine($"================");
    }
}

// Entry point — top-level statements, demo of all scenarios.
var today = new DateTime(2024, 6, 1);

var books = new Book[]
{
    new() { Title = "War and Peace",  DueDate = new DateTime(2024, 5, 22), DailyRate = 5m },   // 10 days
    new() { Title = "Reference",      DueDate = new DateTime(2024, 4, 17), DailyRate = 3m },   // 45 days
    new() { Title = "Checked out",    DueDate = new DateTime(2024, 6, 5),  DailyRate = 5m },   // not overdue
    new() { Title = "No date",        DueDate = null,                       DailyRate = 5m },   // null date
    null                                                                                // null book
};

var printer = new ReceiptPrinter();
foreach (var book in books)
{
    string report = FeeCalculator.FormatReport(book, today);
    Console.WriteLine(report);

    decimal fee = FeeCalculator.CalculateLateFee(book, today);
    printer.PrintReceipt(book, fee);
}
```

Line-by-line walk-through. The `Book` class is a `record class` with `init`-only properties — this stresses immutability and value-like semantics, and `required` on `Title` guarantees a book is never created without a name. `DueDate` is declared as `DateTime?`: the question mark is the nullable value-type syntax, straight from the lesson’s topic on `null` for parameters and properties. `DailyRate` carries a default of `5m` with the `m` suffix, which is critical for `decimal` and is called out in the pitfalls. The `OverdueSeverity` enumeration defines four categories — this will be the return value of a query method.

`FeeCalculator` is declared `static class`: all its methods are `public static`, which matches the best practice “prefer static for methods that do not touch instance state”. The first overload `CalculateLateFee(int, decimal)` uses two early returns — for non-positive days and for a non-positive rate — which is exactly the guard-clause pattern from the lesson, reading top-to-bottom as a list of rules with no nesting. The second overload `CalculateLateFee(Book, DateTime)` differs by its parameter type list — this is a correct signature-based overload and reinforces the notion that “signature = name plus parameter types, but not their names”. Inside, it checks `book is null` and `book.DueDate is null` using the `is` operator, as recommended in the pitfalls, and delegates to the first overload — code reuse without duplicating the computation logic.

`IsOverdue` is a query returning `bool`, with early `false` returns for edge cases and a single happy-path expression at the end. `CategorizeOverdue` demonstrates pattern matching via a switch expression with a `_` discard branch for exhaustiveness — this directly answers the pitfalls item about exhaustiveness. It first calls `IsOverdue`, and if the book is not overdue it immediately returns `None` — reusing a previously written query rather than duplicating the check.

`FormatReport` is a CQS query: it assembles a string, calls other queries, and prints nothing. This contrasts with `ReceiptPrinter.PrintReceipt`, an instance `void` command that only prints. Splitting these two responsibilities illustrates the CQS principle from the lesson: one method computes and formats, the other prints. In `PrintReceipt` we see an early exit from `void` via a bare `return;` with no value — a detail mentioned in the pitfalls. Finally, the entry point uses top-level statements, the collection expression `new Book[] { ... }`, and a `foreach` loop that walks five scenarios including a `null` book and a book with a `null` date — the program does not crash thanks to the guards, which satisfies the acceptance criterion.

#### Going deeper (bonus)
1. Add a third overload of `CalculateLateFee` that accepts an array of books `Book[]` and returns the total fee. Use LINQ `Sum` and verify that the overload is correctly distinguished by signature.
2. Implement `ApplyGracePeriod`, which reduces the number of overdue days by a given grace period (e.g. 2 days) but never below zero. Decide whether it should be a query or a command and justify the choice.
3. Rewrite `CategorizeOverdue` without a switch expression, using a cascade of `if` statements with early return. Compare readability and discuss which variant you prefer and why.
4. Add caching of the computed fee inside the instance class `ReceiptPrinter` (a `Dictionary<Book, decimal>` field). Explain why this turns part of the logic into a stateful command and how it relates to CQS.

---

#### Чек-лист сдачи / Submission checklist
- [ ] Проект собирается без ошибок и предупреждений (RU).
- [ ] Все методы названы глаголами в PascalCase с явными модификаторами доступа (RU).
- [ ] Реализованы две перегрузки `CalculateLateFee`, различающиеся по сигнатуре (RU).
- [ ] `PrintReceipt` — команда `void`, остальные методы `FeeCalculator` — запросы (RU).
- [ ] Программа не падает на `null` и `null`-дате (RU).
- [ ] The project builds without errors or warnings (EN).
- [ ] All methods are named with verbs in PascalCase with explicit access modifiers (EN).
- [ ] Two overloads of `CalculateLateFee` differing by signature are implemented (EN).
- [ ] `PrintReceipt` is a `void` command; the rest of `FeeCalculator` are queries (EN).
- [ ] The program does not crash on `null` or a `null` date (EN).

#### Ресурсы / Resources
- Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/methods — Методы (C#) / Methods (C#)
- Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/programming-guide/classes-and-structs/methods — Методы (руководство по программированию) / Methods (Programming Guide)
- Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/methods/overloading — Перегрузка методов / Method overloading
- Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/language-reference/operators/patterns — Шаблоны и сопоставление шаблонов / Patterns and pattern matching
- Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/language-reference/builtin-types/nullable-value-types — Типы значений, допускающие null / Nullable value types
- Microsoft Learn — https://learn.microsoft.com/dotnet/api/system.datetime — Структура DateTime / DateTime struct

---
[← К уроку M03-L05](lesson-M03-L05-methods.md) | [⬆ К модулю M03](../README.md) | [Следующее ДЗ →](homework-M03-L06-ref-out-in-params.md)
---
