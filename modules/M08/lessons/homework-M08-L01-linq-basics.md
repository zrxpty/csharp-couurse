---
[← К уроку M08-L01](lesson-M08-L01-linq-basics.md) | [⬆ К модулю M08](../README.md) | [Следующее ДЗ →](homework-M08-L02-where-select.md)
---

### Домашнее задание M08-L01: Основы LINQ, method vs query syntax / Homework M08-L01: LINQ basics, method vs query syntax

**Урок / Lesson:** M08-L01
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 3/5
**Цель / Goal:** (RU) Научиться строить LINQ-запросы обеими синтаксическими формами (method и query), понимать отложенное и немедленное выполнение, материализовать результаты и избегать типовых ошибок многократного перебора и мутации источника. (EN) Learn to build LINQ queries in both syntax forms (method and query), understand deferred vs eager execution, materialize results and avoid the common pitfalls of multiple enumeration and source mutation.

#### Связь с уроком / Connection to the lesson
(RU) Урок вводит два столпа LINQ — методы расширения на `IEnumerable<T>` и две синтаксические формы. ДЗ закрепляет именно эти концепции: вы пройдёте путь от тривиального `Where`/`Select` до демонстрации отложенного выполнения и сравнения с eager-операторами `Count`/`First`/`Any`. Особое внимание уделено частым ошибкам из урока — многократному перебору отложенного запроса и мутации источника.
(EN) The lesson introduces the two pillars of LINQ — extension methods on `IEnumerable<T>` and two syntax forms. This homework reinforces exactly those concepts: you go from a trivial `Where`/`Select` to demonstrating deferred execution and contrasting it with eager operators `Count`/`First`/`Any`. Special attention is paid to the lesson's common mistakes — multiple enumeration of a deferred query and source mutation.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация
Представьте, что вы работаете в стартапе «Читалка» — сервисе управления списком прочитанных книг. У вас есть плоский список объектов `Book` с полями `Title`, `Author`, `Pages`, `Year`, `Genre` и `Rating`. Продакт-менеджер регулярно просит разные срезы: «дай книги жанра Fantasy с рейтингом выше 4.5», «посчитай среднюю длину в страницах», «покажи топ-3 по рейтингу за этот год». Писать `foreach`-циклы для каждого запроса утомительно и нечитаемо: код раздувается, бизнес-логика тонет в индексах и временных списках. LINQ решает эту проблему — вы описываете *что* хотите получить, а не *как* перебирать. Кроме того, в кодовой базе уже есть легаси-запросы в query syntax (SQL-подобном), и вам нужно уверенно читать и ту, и другую форму, а также понимать, когда запрос действительно выполнился, а когда он лишь «рецепт». Отложенное выполнение — главный источник багов у новичков: запрос, который выглядит выполненным, на самом деле перевычисляется при каждом `foreach`, и если источник изменился, результат «плывёт». В этом задании вы построите несколько запросов обеими формами, намеренно столкнётесь с отложенностью и материализуете результат, чтобы «заморозить» его. Вы также используете eager-операторы, чтобы увидеть разницу: `Count`, `First`, `Any`, `Max` выполняются сразу и форсируют перебор. В конце вы сравните производительность и читаемость method vs query syntax и сформулируете собственное правило выбора. Это задание — фундамент для следующих уроков модуля M08, где появятся `Where`/`Select` в глубину, `Join`, `GroupBy` и LINQ to SQL/EF.

#### Что нужно сделать (пошагово)
1. Создайте консольный проект .NET 8 с C# 12. Выполните в терминале:
   ```bash
   dotnet new console -n LinqBasicsHomework -o LinqBasicsHomework --framework net8.0
   cd LinqBasicsHomework
   dotnet build
   ```
   Убедитесь, что сборка проходит без ошибок. Откройте `Program.cs` — там будет классический `Console.WriteLine("Hello, World!");`. Вы замените его полностью.

2. Определите модель `Book` как `record` (C# 12 поддерживает запись с позиционными параметрами и `init`-сеттерами). Добавьте файл `Book.cs`:
   ```csharp
   public record Book(string Title, string Author, int Pages, int Year, string Genre, double Rating);
   ```

3. В `Program.cs` создайте исходную коллекцию не менее чем из 10 книг разных жанров (Fantasy, Sci-Fi, Detective, History, Biography). Используйте collection expression (C# 12): `List<Book> library = [ ... ];`.

4. Реализуйте **Method syntax** блок: отфильтруйте книги жанра "Fantasy" с `Rating > 4.5`, отсортируйте по `Rating` по убыванию, спроецируйте в анонимный тип `{ Title, Author, Rating }` и материализуйте через `ToList()`. Выведите результат через `string.Join`.

5. Реализуйте **Query syntax** блок: тот же запрос через `from ... where ... orderby ... select`. Убедитесь, что вывод идентичен. Это докажет эквивалентность форм.

6. Продемонстрируйте **deferred execution**: сохраните `IEnumerable<Book> deferred = library.Where(b => b.Pages > 300);` НЕ материализуя. Выведите `deferred.Count()`. Затем добавьте в `library` ещё одну книгу с `Pages > 300` через `library.Add(...)`. Снова выведите `deferred.Count()`. Вы увидите, что число изменилось — запрос «увидел» новую книгу, потому что он перевычислился.

7. Материализуйте тот же запрос через `ToList()` в отдельную переменную `frozen`, добавьте ещё одну книгу и покажите, что `frozen.Count()` не изменился.

8. Примените eager-операторы и распечатайте результаты: `Any(b => b.Genre == "Sci-Fi")`, `First(b => b.Rating >= 5.0)` (с защитой через `FirstOrDefault`, если такого нет — объясните в комментарии, почему `First` бросит `InvalidOperationException`), `Max(b => b.Pages)`, `Count(b => b.Year >= 2020)`.

9. Сравните читаемость: перепишите один из запросов из method syntax в query syntax и наоборот. В комментарии RU+EN опишите, какая форма вам кажется читаемее и почему (это упражнение на осознанность выбора, а не на догму).

10. Запустите `dotnet run` и убедитесь, что вывод соответствует ожиданиям. Запушьте в репозиторий или сдайте архивом, как требует курс.

#### Требования к решению
- Целевая платформа: .NET 8, язык C# 12, `LangVersion` по умолчанию latest. Top-level statements в `Program.cs`.
- Используйте `record` для модели данных (неизменяемость по умолчанию — это best practice для DTO в LINQ-конвейерах).
- Все запросы должны быть типобезопасны: никаких `ArrayList`, `object`, рефлексии. Тип выводится компилятором.
- Минимум 5 различных операторов: `Where`, `Select`, `OrderBy`/`OrderByDescending`, `Take`, `Any`, `First`/`FirstOrDefault`, `Max`, `Count` — выберите не менее 5.
- Обе синтаксические формы должны присутствовать и давать одинаковый результат на одинаковых данных.
- Должна быть явная демонстрация отложенного выполнения (пункт 6) и материализации (пункт 7).
- Код компилируется без warning-ов уровня `CS8600` и подобных (nullable-контекст включён по умолчанию в .NET 8 templates).
- Вывод программы должен быть структурирован и подписан (например, `--- Method syntax ---`, `--- Deferred ---`), чтобы проверяющий легко orientировался.
- Запрещено использовать базы данных или EF Core — только in-memory коллекции, как в уроке.

#### Тонкости и подводные камни
- **Отложенное выполнение — главный подвох.** Запомните: `Where` и `Select` возвращают `IEnumerable<T>`, который хранит лишь *ссылку* на источник и делегат-предикат. Сам перебор произойдёт только в момент `foreach`/`ToList`/`Count`/`First`. Если между построением и перебором вы мутируете `library.Add(...)`, результат «увидит» новую запись. Это не баг, это контракт — но новички ловят сюрпризы.
- **Многократный перебор одного отложенного запроса** (например, два `foreach` подряд) выполнит фильтрацию дважды. Для тяжёлых источников это лишняя работа. Материализуйте один раз через `ToList()`, если намерены перебирать несколько раз.
- **`First` vs `FirstOrDefault`.** `First(predicate)` бросает `InvalidOperationException`, если ни один элемент не подошёл. `FirstOrDefault` вернёт `default(T)` (для ссылочных типов — `null`, для `Book` как record — `null`). Всегда думайте: пустая последовательность — ожидаемая ситуация или ошибка? От этого зависит выбор.
- **`Select` ≠ `Where`.** `Where` фильтрует (предикат возвращает `bool`), `Select` преобразует (проекция в новый тип). Перепутать легко, особенно если вы пишете `Select(b => b.Rating > 4.5)` — это скомпилируется (проекция в `bool`), но даст `IEnumerable<bool>`, а не отфильтрованные книги. Проверяйте сигнатуру.
- **Query syntax не покрывает все операторы.** `Take`, `Skip`, `Distinct`, `Max` в query syntax недоступны напрямую — приходится оборачивать в скобки и вызывать метод: `(from ... select ...).Take(3)`. Для тривиальных операций method syntax короче и не требует скобок.
- **`Count()` на отложенном запросе** форсирует полный перебор — это eager-оператор. Не вызываете `Count()` в условии цикла `for (int i = 0; i < query.Count(); i++)` — будет O(n²). Сохраните в переменную.
- **Materialize результат, если источник может измениться.** Это правило из урока: «зафиксируйте копию источника перед запросом».

#### Критерии приёмки
- [ ] Проект `LinqBasicsHomework` собирается командой `dotnet build` без ошибок и warning-ов.
- [ ] Используется .NET 8 / C# 12, top-level statements.
- [ ] Модель `Book` определена как `record` с минимум 6 полями.
- [ ] Исходная коллекция содержит не менее 10 книг минимум 5 жанров.
- [ ] Реализован запрос в **method syntax** с фильтром, сортировкой, проекцией и материализацией.
- [ ] Реализован эквивалентный запрос в **query syntax** с идентичным выводом.
- [ ] Демонстрация deferred execution: после `library.Add(...)` счётчик отложенного запроса изменился.
- [ ] Демонстрация материализации: `frozen.Count()` остался прежним после добавления книги.
- [ ] Использованы минимум 5 различных LINQ-операторов.
- [ ] Применены eager-операторы `Any`, `First`/`FirstOrDefault`, `Max`, `Count` с подписанным выводом.
- [ ] В комментариях RU+EN объяснён выбор method vs query syntax для конкретного запроса.
- [ ] Nullable-контекст включён, нет warning-ов `CS8602`/`CS8604`.
- [ ] Вывод программы структурирован заголовками (`--- ... ---`).
- [ ] Код не использует БД/EF — только in-memory.
- [ ] Программа запускается `dotnet run` и не падает.

#### Подсказки (без прямого ответа)
- Подумайте, какой тип возвращает `Where` — это подскажет, почему повторный `Count` «видит» новые данные.
- Если `First` бросает исключение, спросите себя: а что должно вернуться для пустой последовательности?
- Для projection в анонимный тип используйте `select new { ... }` — компилятор выведет тип, вам не нужно его называть.
- Query syntax начинается с `from`, заканчивается `select` или `group`. Запомните порядок — он отличается от SQL (там `SELECT` первый).
- Чтобы «заморозить» запрос, вызовите терминальный оператор — `ToList`/`ToArray`. До этого у вас лишь рецепт.

#### Эталонное решение (разбор)
```csharp
// C# 12 / .NET 8 — LinqBasicsHomework / Program.cs
// Двуязычные комментарии: RU — основная мысль, EN — краткое пояснение.
using System;
using System.Collections.Generic;
using System.Linq;

// Модель данных — неизменяемый record / Immutable record model
public record Book(string Title, string Author, int Pages, int Year, string Genre, double Rating);

// Точка входа — top-level statements / Entry point via top-level statements
List<Book> library =
[
    new("The Hobbit", "J.R.R. Tolkien", 310, 1937, "Fantasy", 4.7),
    new("Dune", "Frank Herbert", 412, 1965, "Sci-Fi", 4.6),
    new("1984", "George Orwell", 328, 1949, "Sci-Fi", 4.5),
    new("Sherlock Holmes", "Arthur Conan Doyle", 307, 1892, "Detective", 4.4),
    new("Sapiens", "Yuval Noah Harari", 443, 2011, "History", 4.5),
    new("Steve Jobs", "Walter Isaacson", 571, 2011, "Biography", 4.6),
    new("The Name of the Wind", "Patrick Rothfuss", 662, 2007, "Fantasy", 4.8),
    new("Foundation", "Isaac Asimov", 255, 1951, "Sci-Fi", 4.3),
    new("The Silent Patient", "Alex Michaelides", 336, 2019, "Detective", 4.2),
    new("Educated", "Tara Westover", 334, 2018, "Biography", 4.7),
    new("Mistborn", "Brandon Sanderson", 541, 2006, "Fantasy", 4.6),
];

// --- Method syntax --- / Синтаксис методов
// Фильтр + проекция + сортировка + материализация / Filter, project, sort, materialize
var methodTop = library
    .Where(b => b.Genre == "Fantasy" && b.Rating > 4.5)   // отложено / deferred
    .OrderByDescending(b => b.Rating)                       // отложено / deferred
    .Select(b => new { b.Title, b.Author, b.Rating })       // проекция в анонимный тип
    .ToList();                                              // eager! материализовано / eager!

Console.WriteLine("--- Method syntax ---");
foreach (var item in methodTop)
    Console.WriteLine($"{item.Rating:F1}  {item.Title} — {item.Author}");

// --- Query syntax --- эквивалент / equivalent
var queryTop = (from b in library
                where b.Genre == "Fantasy" && b.Rating > 4.5
                orderby b.Rating descending
                select new { b.Title, b.Author, b.Rating })
               .ToList();  // query syntax не умеет ToList напрямую — обёртка / wrap to call method

Console.WriteLine("--- Query syntax ---");
foreach (var item in queryTop)
    Console.WriteLine($"{item.Rating:F1}  {item.Title} — {item.Author}");

// --- Deferred execution --- / Отложенное выполнение
IEnumerable<Book> deferred = library.Where(b => b.Pages > 300);
Console.WriteLine($"deferred.Count() до добавления: {deferred.Count()}");  // форсирует перебор
library.Add(new Book("The Way of Kings", "Brandon Sanderson", 1007, 2010, "Fantasy", 4.9));
Console.WriteLine($"deferred.Count() после добавления: {deferred.Count()}");  // выросло!

// --- Materialization --- / Материализация «замораживает»
var frozen = library.Where(b => b.Pages > 300).ToList();
library.Add(new Book("Short Tale", "Anon", 120, 2024, "Detective", 3.9));
Console.WriteLine($"frozen.Count() (не должен измениться): {frozen.Count()}");

// --- Eager operators --- / Немедленные операторы
bool hasSciFi = library.Any(b => b.Genre == "Sci-Fi");
int maxPages = library.Max(b => b.Pages);
int recentCount = library.Count(b => b.Year >= 2020);
Book? topRated = library.FirstOrDefault(b => b.Rating >= 5.0);  // null, если нет — First бросил бы
Console.WriteLine($"hasSciFi={hasSciFi}, maxPages={maxPages}, recentCount={recentCount}");
Console.WriteLine($"topRated(>=5.0)={(topRated is null ? "none" : topRated.Title)}");
```

**Разбор по строкам.** `record Book` — неизменяемый DTO, идеален для LINQ-конвейеров: проекция не мутирует исходник, а создаёт новый тип. Collection expression `[ ... ]` — это C# 12, компилятор превращает его в `new List<Book> { ... }`. Блок **method syntax** строит цепочку: `Where` возвращает `IEnumerable<Book>` (отложено), `OrderByDescending` — тоже отложено, `Select` проецирует в анонимный тип (компилятор синтезирует имя класса), и только `.ToList()` форсирует перебор и фиксирует результат в `List<T>`. Блок **query syntax** семантически идентичен: компилятор переводит `from/where/orderby/select` в те же вызовы методов — поэтому вывод совпадает байт-в-байт. Обёртка `( ... ).ToList()` нужна потому, что в query syntax нет ключевого слова `ToList`. Демонстрация **deferred execution**: `deferred.Count()` вызывается дважды — до и после `library.Add(...)`. Поскольку `Where` хранит ссылку на `library`, второй `Count` перевычисляется и «видит» новую книгу. Это и есть отложенность: запрос — рецепт, перебор — готовка. Блок **materialization** через `ToList()` создаёт независимую копию; последующие `Add` на источник не влияют на `frozen`. Eager-блок показывает операторы, которые выполняются немедленно: `Any` перебирает до первого совпадения, `Max` — всю последовательность, `Count(predicate)` — тоже всю, `FirstOrDefault` — до первого совпадения или до конца. Использование `FirstOrDefault` вместо `First` — осознанный выбор: если ни одна книга не имеет рейтинг 5.0, мы не хотим краш программы, а хотим `null` и понятный вывод «none». Концепции урока, применённые здесь: методы расширения на `IEnumerable<T>`, две синтаксические формы, отложенное vs немедленное выполнение, материализация, разница `First`/`FirstOrDefault`, проекция в анонимный тип, типобезопасность компилятора.

#### Задания на углубление (бонус)
1. Добавьте оператор `Distinct` по `Genre` и соберите уникальный список жанров. Объясните, почему `Distinct` для `record` работает «из коробки» (подсказка: value equality).
2. Реализуйте запрос с `group by` в query syntax: сгруппируйте книги по жанру и выведите средний рейтинг каждой группы. Сравните читаемость с method-вариантом через `GroupBy(...)`.
3. Замерьте время многократного перебора отложенного запроса vs материализованного (`Stopwatch`) на коллекции из 100 000 элементов (сгенерируйте через `Enumerable.Range`). Объясните результат.
4. Перепишите модель как `class` с изменяемыми полями и покажите, как мутация поля `Rating` у объекта в коллекции влияет на уже построенный отложенный запрос. Сделайте вывод о роли неизменяемости.

---

## Statement in English / Постановка на английском

#### Context & motivation
Imagine you work at a startup called "Reader" — a service that manages a list of books a user has read. You have a flat list of `Book` objects with fields `Title`, `Author`, `Pages`, `Year`, `Genre` and `Rating`. The product manager regularly asks for different slices: "give me Fantasy books with rating above 4.5", "compute the average page count", "show the top 3 by rating this year". Writing `foreach` loops for each request is tedious and unreadable: the code balloons and the business intent drowns in indices and temporary lists. LINQ solves this — you describe *what* you want, not *how* to iterate. Moreover, the existing codebase already contains legacy queries written in query syntax (the SQL-like form), so you must confidently read both forms and know when a query has actually executed and when it is merely a "recipe". Deferred execution is the single biggest source of beginner bugs: a query that looks executed actually re-evaluates on every `foreach`, and if the source mutated, the result "drifts". In this assignment you will build several queries in both forms, deliberately encounter deferred execution, and materialize the result to "freeze" it. You will also use eager operators to see the contrast: `Count`, `First`, `Any`, `Max` run immediately and force enumeration. Finally you will compare the readability of method vs query syntax and articulate your own rule for choosing between them. This homework is the foundation for the rest of module M08, where `Where`/`Select` go deeper and `Join`, `GroupBy` and LINQ to SQL/EF appear.

#### What to do step by step
1. Create a .NET 8 console project with C# 12. In the terminal run:
   ```bash
   dotnet new console -n LinqBasicsHomework -o LinqBasicsHomework --framework net8.0
   cd LinqBasicsHomework
   dotnet build
   ```
   Confirm the build is clean. Open `Program.cs` — it contains the usual `Console.WriteLine("Hello, World!");`. You will replace it entirely.

2. Define the `Book` model as a `record` (C# 12 supports positional records with init-only setters). Add a file `Book.cs`:
   ```csharp
   public record Book(string Title, string Author, int Pages, int Year, string Genre, double Rating);
   ```

3. In `Program.cs` build a source collection of at least 10 books across different genres (Fantasy, Sci-Fi, Detective, History, Biography). Use a C# 12 collection expression: `List<Book> library = [ ... ];`.

4. Implement a **method syntax** block: filter books of genre "Fantasy" with `Rating > 4.5`, sort by `Rating` descending, project into an anonymous type `{ Title, Author, Rating }` and materialize with `ToList()`. Print the result via `string.Join` or a `foreach`.

5. Implement a **query syntax** block: the same query via `from ... where ... orderby ... select`. Confirm the output is identical. This proves the two forms are equivalent.

6. Demonstrate **deferred execution**: store `IEnumerable<Book> deferred = library.Where(b => b.Pages > 300);` WITHOUT materializing. Print `deferred.Count()`. Then add another book with `Pages > 300` via `library.Add(...)`. Print `deferred.Count()` again. You will see the number changed — the query "saw" the new book because it re-evaluated.

7. Materialize the same query with `ToList()` into a separate variable `frozen`, add yet another book and show that `frozen.Count()` did NOT change.

8. Apply eager operators and print the results: `Any(b => b.Genre == "Sci-Fi")`, `First(b => b.Rating >= 5.0)` (guarded with `FirstOrDefault` — explain in a comment why `First` would throw `InvalidOperationException` if nothing matches), `Max(b => b.Pages)`, `Count(b => b.Year >= 2020)`.

9. Compare readability: rewrite one of the queries from method syntax into query syntax and vice versa. In a RU+EN comment describe which form feels more readable to you and why (this is an exercise in conscious choice, not dogma).

10. Run `dotnet run` and confirm the output matches expectations. Push to your repo or submit as an archive, as the course requires.

#### Requirements
- Target platform: .NET 8, language C# 12, default `LangVersion` = latest. Top-level statements in `Program.cs`.
- Use a `record` for the data model (default immutability is a best practice for DTOs in LINQ pipelines).
- All queries must be type-safe: no `ArrayList`, no `object`, no reflection. The compiler infers types.
- At least 5 distinct operators used: `Where`, `Select`, `OrderBy`/`OrderByDescending`, `Take`, `Any`, `First`/`FirstOrDefault`, `Max`, `Count` — pick at least 5.
- Both syntax forms must be present and produce identical output on identical data.
- An explicit demonstration of deferred execution (step 6) and materialization (step 7) is required.
- The code must compile without `CS8600`-level warnings (nullable context is on by default in .NET 8 templates).
- The program output must be structured and labelled (e.g. `--- Method syntax ---`, `--- Deferred ---`) so a reviewer can orient easily.
- No databases or EF Core — in-memory collections only, as in the lesson.

#### Pitfalls
- **Deferred execution is the main trap.** Remember: `Where` and `Select` return an `IEnumerable<T>` that holds only a *reference* to the source and a predicate delegate. The actual iteration happens only at `foreach`/`ToList`/`Count`/`First`. If you mutate `library.Add(...)` between building and enumerating, the result "sees" the new entry. This is not a bug, it is the contract — but beginners get surprised.
- **Multiple enumeration of one deferred query** (e.g. two `foreach` in a row) runs the filter twice. For heavy sources this is wasted work. Materialize once with `ToList()` if you intend to iterate several times.
- **`First` vs `FirstOrDefault`.** `First(predicate)` throws `InvalidOperationException` when no element matches. `FirstOrDefault` returns `default(T)` (`null` for reference types, `null` for a `Book` record). Always ask: is an empty sequence expected or an error? That decides the choice.
- **`Select` ≠ `Where`.** `Where` filters (predicate returns `bool`), `Select` transforms (projection into a new type). It is easy to mix them up, especially `Select(b => b.Rating > 4.5)` — it compiles (projection into `bool`) but yields `IEnumerable<bool>`, not filtered books. Check the signature.
- **Query syntax does not cover all operators.** `Take`, `Skip`, `Distinct`, `Max` are not directly available in query syntax — you must wrap in parentheses and call the method: `(from ... select ...).Take(3)`. For trivial operations method syntax is shorter and needs no parens.
- **`Count()` on a deferred query** forces a full enumeration — it is eager. Do not call `Count()` inside a `for` loop condition `for (int i = 0; i < query.Count(); i++)` — that is O(n²). Cache it in a variable.
- **Materialize the result if the source may change.** This is the lesson rule: "snapshot the source before the query".

#### Acceptance criteria
- [ ] The project `LinqBasicsHomework` builds with `dotnet build` with no errors or warnings.
- [ ] .NET 8 / C# 12 is used, with top-level statements.
- [ ] The `Book` model is a `record` with at least 6 fields.
- [ ] The source collection has at least 10 books across at least 5 genres.
- [ ] A query in **method syntax** is implemented with filter, sort, projection and materialization.
- [ ] An equivalent query in **query syntax** is implemented with identical output.
- [ ] Deferred execution is demonstrated: after `library.Add(...)` the deferred query's count changes.
- [ ] Materialization is demonstrated: `frozen.Count()` stays the same after adding a book.
- [ ] At least 5 distinct LINQ operators are used.
- [ ] Eager operators `Any`, `First`/`FirstOrDefault`, `Max`, `Count` are applied with labelled output.
- [ ] A RU+EN comment explains the choice of method vs query syntax for a particular query.
- [ ] Nullable context is enabled; no `CS8602`/`CS8604` warnings.
- [ ] The program output is structured with headers (`--- ... ---`).
- [ ] The code uses no DB/EF — only in-memory.
- [ ] The program runs via `dotnet run` without crashing.

#### Hints (no direct answer)
- Think about what type `Where` returns — that explains why a repeated `Count` "sees" new data.
- If `First` throws, ask yourself: what should be returned for an empty sequence?
- For projection into an anonymous type use `select new { ... }` — the compiler infers the type, you do not name it.
- Query syntax starts with `from` and ends with `select` or `group`. Note the order — it differs from SQL (where `SELECT` comes first).
- To "freeze" a query call a terminal operator — `ToList`/`ToArray`. Before that you only have a recipe.

#### Reference solution walk-through
```csharp
// C# 12 / .NET 8 — LinqBasicsHomework / Program.cs
// Bilingual comments: RU main idea, EN brief note.
using System;
using System.Collections.Generic;
using System.Linq;

// Data model — immutable record / Immutable record model
public record Book(string Title, string Author, int Pages, int Year, string Genre, double Rating);

// Entry point — top-level statements / Entry point via top-level statements
List<Book> library =
[
    new("The Hobbit", "J.R.R. Tolkien", 310, 1937, "Fantasy", 4.7),
    new("Dune", "Frank Herbert", 412, 1965, "Sci-Fi", 4.6),
    new("1984", "George Orwell", 328, 1949, "Sci-Fi", 4.5),
    new("Sherlock Holmes", "Arthur Conan Doyle", 307, 1892, "Detective", 4.4),
    new("Sapiens", "Yuval Noah Harari", 443, 2011, "History", 4.5),
    new("Steve Jobs", "Walter Isaacson", 571, 2011, "Biography", 4.6),
    new("The Name of the Wind", "Patrick Rothfuss", 662, 2007, "Fantasy", 4.8),
    new("Foundation", "Isaac Asimov", 255, 1951, "Sci-Fi", 4.3),
    new("The Silent Patient", "Alex Michaelides", 336, 2019, "Detective", 4.2),
    new("Educated", "Tara Westover", 334, 2018, "Biography", 4.7),
    new("Mistborn", "Brandon Sanderson", 541, 2006, "Fantasy", 4.6),
];

// --- Method syntax --- / Method syntax
// Filter + projection + sort + materialization / Filter, project, sort, materialize
var methodTop = library
    .Where(b => b.Genre == "Fantasy" && b.Rating > 4.5)   // deferred / отложено
    .OrderByDescending(b => b.Rating)                       // deferred / отложено
    .Select(b => new { b.Title, b.Author, b.Rating })       // projection into anonymous type
    .ToList();                                              // eager! materialized / eager!

Console.WriteLine("--- Method syntax ---");
foreach (var item in methodTop)
    Console.WriteLine($"{item.Rating:F1}  {item.Title} — {item.Author}");

// --- Query syntax --- equivalent / эквивалент
var queryTop = (from b in library
                where b.Genre == "Fantasy" && b.Rating > 4.5
                orderby b.Rating descending
                select new { b.Title, b.Author, b.Rating })
               .ToList();  // query syntax cannot call ToList directly — wrap / wrap to call method

Console.WriteLine("--- Query syntax ---");
foreach (var item in queryTop)
    Console.WriteLine($"{item.Rating:F1}  {item.Title} — {item.Author}");

// --- Deferred execution --- / Отложенное выполнение
IEnumerable<Book> deferred = library.Where(b => b.Pages > 300);
Console.WriteLine($"deferred.Count() before add: {deferred.Count()}");  // forces enumeration
library.Add(new Book("The Way of Kings", "Brandon Sanderson", 1007, 2010, "Fantasy", 4.9));
Console.WriteLine($"deferred.Count() after add: {deferred.Count()}");  // grew!

// --- Materialization --- / Materialization "freezes"
var frozen = library.Where(b => b.Pages > 300).ToList();
library.Add(new Book("Short Tale", "Anon", 120, 2024, "Detective", 3.9));
Console.WriteLine($"frozen.Count() (should not change): {frozen.Count()}");

// --- Eager operators --- / Immediate operators
bool hasSciFi = library.Any(b => b.Genre == "Sci-Fi");
int maxPages = library.Max(b => b.Pages);
int recentCount = library.Count(b => b.Year >= 2020);
Book? topRated = library.FirstOrDefault(b => b.Rating >= 5.0);  // null if none — First would throw
Console.WriteLine($"hasSciFi={hasSciFi}, maxPages={maxPages}, recentCount={recentCount}");
Console.WriteLine($"topRated(>=5.0)={(topRated is null ? "none" : topRated.Title)}");
```

**Line-by-line walk-through.** `record Book` is an immutable DTO, ideal for LINQ pipelines: projection does not mutate the source but creates a new type. The collection expression `[ ... ]` is C# 12 — the compiler turns it into `new List<Book> { ... }`. The **method syntax** block builds a chain: `Where` returns `IEnumerable<Book>` (deferred), `OrderByDescending` is also deferred, `Select` projects into an anonymous type (the compiler synthesizes a class name), and only `.ToList()` forces enumeration and pins the result in a `List<T>`. The **query syntax** block is semantically identical: the compiler translates `from/where/orderby/select` into the same method calls — so the output matches byte-for-byte. The `( ... ).ToList()` wrapper is needed because query syntax has no `ToList` keyword. The **deferred execution** demo calls `deferred.Count()` twice — before and after `library.Add(...)`. Because `Where` holds a reference to `library`, the second `Count` re-evaluates and "sees" the new book. That is deferred execution: the query is a recipe, enumeration is the cooking. The **materialization** block via `ToList()` creates an independent copy; subsequent `Add` calls on the source do not affect `frozen`. The eager block shows operators that run immediately: `Any` scans until the first match, `Max` scans the whole sequence, `Count(predicate)` also scans the whole sequence, `FirstOrDefault` scans until the first match or the end. Using `FirstOrDefault` instead of `First` is a conscious choice: if no book has a rating of 5.0 we do not want a crash, we want `null` and a clear "none" output. Lesson concepts applied here: extension methods on `IEnumerable<T>`, the two syntax forms, deferred vs eager execution, materialization, the `First`/`FirstOrDefault` distinction, projection into an anonymous type, and compiler-enforced type safety.

#### Going deeper (bonus)
1. Add a `Distinct` on `Genre` and collect the unique genres. Explain why `Distinct` works "out of the box" for a `record` (hint: value equality).
2. Implement a `group by` query in query syntax: group books by genre and print the average rating of each group. Compare readability with the method-syntax version using `GroupBy(...)`.
3. Measure the time of multiple enumerations of a deferred query vs a materialized one (`Stopwatch`) on a collection of 100 000 elements (generate via `Enumerable.Range`). Explain the result.
4. Rewrite the model as a `class` with mutable fields and show how mutating the `Rating` field of an object in the collection affects an already-built deferred query. Draw a conclusion about the role of immutability.

---

#### Чек-лист сдачи / Submission checklist
- [ ] (RU) Проект собирается `dotnet build` без warning-ов.
- [ ] (RU) Использованы обе синтаксические формы LINQ с одинаковым выводом.
- [ ] (RU) Демонстрация deferred execution и материализации присутствует.
- [ ] (RU) Минимум 5 операторов LINQ применено.
- [ ] (RU) `First`/`FirstOrDefault` выбран осознанно, с комментарием.
- [ ] (RU) Вывод структурирован заголовками.
- [ ] (EN) The project builds with `dotnet build` with no warnings.
- [ ] (EN) Both LINQ syntax forms are used with identical output.
- [ ] (EN) Deferred execution and materialization are demonstrated.
- [ ] (EN) At least 5 LINQ operators are applied.
- [ ] (EN) `First`/`FirstOrDefault` is chosen deliberately, with a comment.
- [ ] (EN) Output is structured with headers.

#### Ресурсы / Resources
- [Microsoft Learn — LINQ overview](https://learn.microsoft.com/dotnet/csharp/linq/)
- [LINQ query syntax vs method syntax](https://learn.microsoft.com/dotnet/csharp/programming-guide/concepts/linq/query-syntax-and-method-syntax-in-linq)
- [Deferred execution vs immediate execution](https://learn.microsoft.com/dotnet/csharp/programming-guide/concepts/linq/deferred-execution-and-lazy-evaluation-in-linq-to-xml)
- [Enumerable methods reference](https://learn.microsoft.com/dotnet/api/system.linq.enumerable)
