---
[← К уроку M08-L03](lesson-M08-L03-orderby-thenby.md) | [⬆ К модулю M08](../README.md) | [Следующее ДЗ →](homework-M08-L04-groupby-tolookup.md)
---

### Домашнее задание M08-L03: OrderBy/ThenBy, Reverse / Homework M08-L03: OrderBy/ThenBy, Reverse

**Урок / Lesson:** M08-L03
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 3/5
**Цель / Goal:** (RU) Научиться сортировать последовательности в LINQ с помощью `OrderBy`, `OrderByDescending`, `ThenBy`, `ThenByDescending` и `Reverse`, применять `IComparer<T>` и `StringComparer` для кастомных правил сравнения, понимать стабильность сортировки и отложенное выполнение. (EN) Learn to sort LINQ sequences with `OrderBy`, `OrderByDescending`, `ThenBy`, `ThenByDescending`, and `Reverse`, apply `IComparer<T>` and `StringComparer` for custom comparison rules, and understand sort stability and deferred execution.

#### Связь с уроком / Connection to the lesson

(RU) Урок M08-L03 вводит пять операторов сортировки LINQ и подробно разбирает их поведение: первичный и вторичный ключи, стабильность, отложенное выполнение, перегрузки с компаратором и роль `Reverse` как инверсии, а не сортировки по значению. Данное задание напрямую закрепляет все эти темы: вы построите многоуровневые цепочки, покажете стабильность экспериментально, реализуете собственный `IComparer<T>` и материализуете результаты, чтобы убедиться в отложенном выполнении. Также вы столкнётесь с типичными ошибками — вызовом `ThenBy` без `OrderBy` и серией `OrderBy` вместо `ThenBy` — и научитесь их избегать.

(EN) Lesson M08-L03 introduces the five LINQ sorting operators and examines their behavior in detail: primary and secondary keys, stability, deferred execution, comparer overloads, and the role of `Reverse` as an inversion rather than a value-based sort. This homework reinforces every one of those topics directly: you will build multi-level chains, demonstrate stability experimentally, implement your own `IComparer<T>`, and materialize results to observe deferred execution. You will also meet typical mistakes — calling `ThenBy` without `OrderBy`, and chaining several `OrderBy` calls instead of `ThenBy` — and learn to avoid them.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация

Представьте, что вы готовите отчёт для кафедры университета. У вас есть список студентов: каждый имеет фамилию, номер группы (например, `CS-101`), курс (1–4), средний балл (`Gpa`, типа `double`, например 4.5) и дату зачисления (`EnrolledOn`, `DateOnly`). Список небольшой, но он приходит из внешнего источника в произвольном порядке, а вам нужно представить его в нескольких видах: рейтинг по среднему баллу (от лучшего к худшему), алфавитный список с разбиением по группам, список «новичков» (моложе по курсу — на первом курсе), а также ретроспективу — от последних зачисленных к первым, чтобы оценить динамику набора. Одно и то же множество данных нужно отсортировать минимум пятью способами, и часть способов требует вторичных критериев: при равном `Gpa` сначала идёт фамилия по алфавиту, а при равной фамилии — имя; при сортировке по группе вторичным ключом выступает курс.

В реальных проектах сортировка почти никогда не бывает одномерной. Рейтинги, таблицы лидеров, очереди задач, лог-файлы — везде появляются составные ключи. LINQ даёт для этого удобный декларативный аппарат: вы описываете критерии как цепочку методов, а фреймворк берёт на себя механику сравнения. Но этот аппарат требует понимания: `ThenBy` не существует без `OrderBy`, несколько подряд `OrderBy` дают не составную сортировку, а только последнюю, `Reverse` — это не сортировка, а инверсия, а стабильность — не абстрактное свойство, а работающий инструмент, позволяющий сортировать по вторичному ключу раньше первичного. В этом задании вы пройдёте все эти ситуации на практике и亲手 убедитесь, как ведут себя операторы. Вы также столкнётесь с компараторами: часть строк нужно сравнивать без учёта регистра, а часть — по особому правилу (например, группа `CS-101` должна идти раньше `CS-20`, потому что логический порядок групп не совпадает с лексикографическим). Для этого придётся реализовать `IComparer<string>` самостоятельно и грамотно подать его в перегрузку `OrderBy(..., IComparer<TKey>)`.

#### Что нужно сделать (пошагово)

1. Создайте новый консольный проект .NET 8 с топ-level statements: выполните `dotnet new console -n SortingHomework -o SortingHomework --framework net8.0` в каталоге `modules/M08/homework` (если каталога нет — создайте). Перейдите в каталог проекта: `cd SortingHomework`. Убедитесь, что `dotnet build` проходит без предупреждений. Назначьте LangVersion `latest` в `SortingHomework.csproj`, добавив `<PropertyGroup><LangVersion>latest</LangVersion></PropertyGroup>`.

2. Замените содержимое `Program.cs` на собственный код. Определите модель студента с помощью `record`: `public record Student(string LastName, string FirstName, string Group, int Course, double Gpa, DateOnly EnrolledOn);`. Для удобной отладки переопределите `ToString` через синтаксис `record` (можно просто оставить автогенерируемый `ToString`, который уже даёт читаемый вывод).

3. Подготовьте исходный список из 12 студентов (переменная `students` типа `List<Student>`). Подберите данные так, чтобы они демонстрировали тонкости: минимум три пары с одинаковым `Gpa` (например, два студента с 4.5 и два с 3.8), минимум две пары с одинаковой фамилией, но разными именами, минимум одну пару в одной группе на одном курсе. Включите фамилии в разном регистре (например, `иванов` и `Иванов`), чтобы проверить `StringComparer.OrdinalIgnoreCase`.

4. Реализуйте и выведите на консоль пять запросов. Каждый запрос должен быть оформлен как отдельная локальная функция (например, `ByGpaRating`, `ByGroupThenCourse`, `FreshmenFirst`, `EnrollmentRetrospective`, `ByCustomGroupComparer`). Заголовок каждого раздела выводите через `Console.WriteLine` с чётким описанием критериев.

   - **Рейтинг по Gpa** (`ByGpaRating`): `OrderByDescending(s => s.Gpa)`, затем `ThenBy(s => s.LastName, StringComparer.OrdinalIgnoreCase)`, затем `ThenBy(s => s.FirstName, StringComparer.OrdinalIgnoreCase)`. Ожидается: лучшие студенты сверху; при равном балле — алфавит по фамилии; при равной фамилии — по имени.
   - **По группе, затем по курсу** (`ByGroupThenCourse`): `OrderBy(s => s.Group, StringComparer.OrdinalIgnoreCase)`, затем `ThenBy(s => s.Course)`. Внутри группы номера курсов идут по возрастанию.
   - **Сначала первокурсники** (`FreshmenFirst`): `OrderBy(s => s.Course)`, затем `ThenByDescending(s => s.Gpa)`, затем `ThenBy(s => s.LastName, StringComparer.OrdinalIgnoreCase)`. Первокурсники идут раньше старшекурсников; внутри курса — по убыванию балла.
   - **Ретросптива зачисления** (`EnrollmentRetrospective`): `OrderByDescending(s => s.EnrolledOn)`, а затем примените `.Reverse()` ко всей последовательности, чтобы показать, что `Reverse` инвертирует уже упорядоченный результат. Проверьте, что итоговый порядок совпадает с `OrderBy(s => s.EnrolledOn)` — и объясните в комментарии, почему `OrderBy(...).Reverse()` и `OrderByDescending(...)` в случае стабильной сортировки дают совпадающий результат лишь тогда, когда ключи уникальны.
   - **Кастомный компаратор групп** (`ByCustomGroupComparer`): реализуйте `sealed class GroupComparer : IComparer<string>`, который разбирает строку вида `CS-101` на префикс (`CS`) и число (`101`) и сравнивает сначала по префиксу лексикографически, а при равном префиксе — по числу численно (чтобы `CS-20` шёл после `CS-101`, а не перед ним, как было бы при строковом сравнении). Примените его через `OrderBy(s => s.Group, new GroupComparer())`, затем `ThenBy(s => s.LastName, StringComparer.OrdinalIgnoreCase)`.

5. Покажите стабильность сортировки экспериментально. Создайте отдельный раздел `// Stable sort demonstration`. Возьмите подсписок студентов с одинаковым `Gpa` (например, всех с `4.5`), отсортируйте их сначала по фамилии, потом по тому же `Gpa` (через `OrderBy`): убедитесь, что порядок фамилий «выживает» — это и есть стабильность. Выведите список до и после, добавьте комментарий с объяснением. Используйте метод расширения `Take`/`Where`, изученный в предыдущих уроках, чтобы отфильтровать подсписок.

6. Продемонстрируйте типичную ошибку: закомментируйте блок с двумя подряд `OrderBy` и пояснением, что второй `OrderBy` отменяет первый. Сравните результат `OrderBy(s => s.Course).OrderBy(s => s.Gpa)` с `OrderBy(s => s.Course).ThenBy(s => s.Gpa)`. Выведите оба и покажите, что первый вариант отсортирован только по `Gpa`, а второй — по курсу с уточнением по `Gpa`.

7. Покажите отложенное выполнение. Сохраните результат `var q = students.OrderBy(s => s.Gpa);` в переменную, затем добавьте в `students` ещё один элемент через `students.Add(...)`, затем перечислите `q` через `foreach`. Убедитесь, что новый элемент появляется в выводе — сортировка не «заморожена». Затем вызовите `.ToList()` сразу после `OrderBy`, повторите добавление и покажите, что снимок остался прежним. В комментарии объясните связь с deferred execution.

8. Вызовите `dotnet run` и сохраните вывод в файл `output.txt` (через перенаправление `dotnet run > output.txt`). Убедитесь, что все пять запросов выводятся корректно, а разделы с экспериментами читаются без ошибок. Запустите `dotnet build --no-incremental -warnaserror`, чтобы гарантировать отсутствие предупреждений.

#### Требования к решению

Решение должно представлять собой один файл `Program.cs` с top-level statements, без явного объявления `class Program` и `static void Main`. Используйте C# 12: collection expressions для инициализации `students` (`List<Student> students = [ ... ];` или `new(){ ... }`), pattern matching там, где он уместен (например, в `GroupComparer.Compare` можно разобрать строку через `string.Split` и `int.TryParse` с шаблоном `is [var prefix, var num]`), а также file-scoped namespaces не нужны — для файла `Program.cs` с top-level statements это излишне. Все LINQ-запросы должны быть написаны через method syntax (точечная нотация), как в уроке, а не через query syntax (`from ... orderby`). Каждая локальная функция должна возвращать `IEnumerable<Student>` или `IOrderedEnumerable<Student>` — возвращать `List<Student>` не нужно, чтобы подчеркнуть отложенный характер.

Компаратор `GroupComparer` должен быть реализован как `sealed class GroupComparer : IComparer<string>` с публичным методом `int Compare(string? x, string? y)`, корректно обрабатывающим `null` (по соглашению `null` идёт первым или последним — выберите одно и явно укажите в комментарии). Разбор строки группы должен быть устойчивым: если строка не содержит дефиса или числовая часть не парсится, используйте запасной порядок (например, лексикографическое сравнение всей строки). Никаких `throw` на «некорректный» ввод быть не должно — компаратор обязан быть полным.

Вывод должен быть читаемым: каждая строка — один студент с ключевыми полями, например `[CS-101, курс 2, Gpa 4.5] Иванов, Иван (зачислен 2023-09-01)`. Используйте интерполяцию строк и, при желании, raw string literals `"""..."""` для многострочных заголовков. Весь код должен компилироваться под `.NET 8` без предупреждений уровня `CS` и проходить `dotnet build -warnaserror`.

#### Тонкости и подводные камни

Главная ловушка — попытка вызвать `ThenBy` напрямую на `IEnumerable<T>` без предварительного `OrderBy`. Компилятор выдаст ошибку CS1929: `IOrderedEnumerable<T>` — это отдельный интерфейс, расширяющий `IEnumerable<T>`, и `ThenBy` определён именно на нём. Запомните: любой вторичный критерий начинается с `OrderBy` или `OrderByDescending`. Вторая ловушка — цепочка из нескольких `OrderBy`: `.OrderBy(s => s.Course).OrderBy(s => s.Gpa)` не даёт сортировку «сначала по курсу, потом по Gpa» — второй `OrderBy` полностью заменяет первый, и вы получаете сортировку только по `Gpa`, теряя порядок курсов. Это легко не заметить на данных, где `Gpa` случайно коррелирует с курсом.

Третья ловушка — уверенность, что `Reverse` сортирует. `Reverse` просто инвертирует текущий порядок за O(n) без вызова компаратора; это видно на примере `OrderBy(x => x.Gpa).Reverse()` — результат не равен `OrderByDescending(x => x.Gpa)`, если есть равные ключи: в первом случае равные сохранят перевёрнутый исходный порядок (стабильность работает «в обратную сторону»), во втором — исходный. Четвёртая тонкость — передача компаратора не того типа. Перегрузка `OrderBy<TSource, TKey>(..., IComparer<TKey>)` ожидает компаратор для типа ключа, а не для типа элемента. Если у вас `Student` и ключ `string Group`, то компаратор должен быть `IComparer<string>`, а не `IComparer<Student>`. Компилятор не даст передать неправильный тип, но легко ошибиться, когда ключ и элемент совпадают по типу (например, сортировка `List<string>`).

Пятая тонкость — отложенное выполнение. Результат `OrderBy` не хранится в памяти; это объект-запрос, который перевыполняется при каждом перечислении. Если вы модифицируете исходный список и снова перечислите — увидите новый снимок. Если же вам нужен «слепок на момент вызова», сразу вызывайте `.ToList()` или `.ToArray()`. Шестая тонкость — стабильность и порядок вызовов. Стабильность позволяет сортировать по вторичному ключу раньше первичного: `.OrderBy(s => s.LastName).OrderBy(s => s.Gpa)` сохранит фамильный порядок для равных `Gpa` только потому, что LINQ стабилен — но это хрупкая техника, лучше использовать `ThenBy`, который сам выражает намерение. Седьмая тонкость — производительность: каждый `ThenBy` добавляет стабильный проход, поэтому при большом числе критериев иногда лучше собрать составной ключ (кортеж) в одном `OrderBy`. На малых списках это незаметно, но привычка формировать «толстые» ключи полезна.

#### Критерии приёмки

- [ ] Проект `SortingHomework` создан под .NET 8 и компилируется без предупреждений (`dotnet build -warnaserror` проходит).
- [ ] `Program.cs` использует top-level statements и C# 12 (collection expressions, pattern matching в компараторе).
- [ ] Модель `Student` объявлена как `record` с шестью полями, включая `DateOnly EnrolledOn`.
- [ ] В списке есть минимум 12 студентов; данные намеренно подобраны для демонстрации равных ключей и регистра.
- [ ] Реализованы все пять локальных функций: `ByGpaRating`, `ByGroupThenCourse`, `FreshmenFirst`, `EnrollmentRetrospective`, `ByCustomGroupComparer`.
- [ ] `ByGpaRating` использует `OrderByDescending(Gpa).ThenBy(LastName, OrdinalIgnoreCase).ThenBy(FirstName, OrdinalIgnoreCase)`.
- [ ] `ByGroupThenCourse` использует `OrderBy(Group, OrdinalIgnoreCase).ThenBy(Course)`.
- [ ] `FreshmenFirst` использует `OrderBy(Course).ThenByDescending(Gpa).ThenBy(LastName, OrdinalIgnoreCase)`.
- [ ] `EnrollmentRetrospective` использует `OrderByDescending(EnrolledOn).Reverse()` с пояснением о связи с `OrderBy(EnrolledOn)`.
- [ ] `GroupComparer` реализует `IComparer<string>`, разбирает `CS-101` на префикс и число, сравнивает по префиксу лексикографически, затем по числу численно; устойчив к некорректному формату.
- [ ] Стабильность продемонстрирована экспериментально: сортировка подсписка по фамилии, затем по равному ключу сохраняет фамильный порядок.
- [ ] Ошибка «два `OrderBy` подряд» показана и объяснена в комментарии: второй отменяет первый.
- [ ] Отложенное выполнение показано: модификация `students` после `OrderBy` видна в перечислении; `.ToList()` фиксирует снимок.
- [ ] Вывод читаем, каждый студент — отдельная строка с ключевыми полями; разделы подписаны.
- [ ] `dotnet run > output.txt` формирует полный лог; в нём присутствуют все пять разделов и три эксперимента.

#### Подсказки (без прямого ответа)

- Чтобы получить подсписок «с одинаковым Gpa», используйте `students.Where(s => s.Gpa == 4.5).ToList()` — это создаст копию, которую безопасно сортировать.
- Для разбора группы в `GroupComparer` попробуйте `string.Split('-')` и pattern `is [var prefix, var numStr] when int.TryParse(numStr, out var num)` — это идиоматичный C# 12.
- Помните, что `OrderBy(...).Reverse()` и `OrderByDescending(...)` совпадают только когда ключи уникальны; на равных ключах стабильность даёт разный результат — используйте это для комментария.
- Если `ThenBy` не компилируется — проверьте, что вы вызываете его на `IOrderedEnumerable<Student>`, а не на `IEnumerable<Student>`; это значит, что перед ним не стоит `OrderBy`/`OrderByDescending`.
- Для `DateOnly` сравнение по умолчанию хронологическое, поэтому `OrderByDescending(s => s.EnrolledOn)` даёт «от новых к старым».
- Не забудьте `using System; using System.Collections.Generic; using System.Linq;` в начале файла (для top-level statements `using` можно ставить прямо в начале).

#### Эталонное решение (разбор)

```csharp
// C# 12 / .NET 8 — Домашнее задание M08-L03: OrderBy/ThenBy, Reverse
// Top-level statements. Bilingual comments RU+EN.

using System;
using System.Collections.Generic;
using System.Linq;

// Модель студента / Student model
public record Student(
    string LastName,
    string FirstName,
    string Group,    // формат "CS-101" / format "CS-101"
    int Course,      // 1..4
    double Gpa,      // 0.0..5.0
    DateOnly EnrolledOn);

// Кастомный компаратор групп: сначала по префиксу, потом по числу.
// Custom group comparer: prefix first, then numeric part.
public sealed class GroupComparer : IComparer<string>
{
    public int Compare(string? x, string? y)
    {
        // null-обработка: null считается «меньше» / null sorts first
        if (x is null && y is null) return 0;
        if (x is null) return -1;
        if (y is null) return 1;

        // Разбор "CS-101" → prefix="CS", num=101
        // Parse "CS-101" → prefix="CS", num=101
        if (TrySplit(x, out var px, out var nx) &&
            TrySplit(y, out var py, out var ny))
        {
            int byPrefix = string.Compare(px, py, StringComparison.OrdinalIgnoreCase);
            if (byPrefix != 0) return byPrefix;
            return nx.CompareTo(ny); // числовое сравнение / numeric compare
        }

        // Запасной порядок: лексикографически / Fallback: lexicographic
        return string.Compare(x, y, StringComparison.OrdinalIgnoreCase);
    }

    private static bool TrySplit(string s, out string prefix, out int num)
    {
        prefix = s;
        num = 0;
        var parts = s.Split('-');
        if (parts is [var p, var n] && int.TryParse(n, out num))
        {
            prefix = p;
            return true;
        }
        return false;
    }
}

// Исходный список / Source list
List<Student> students =
[
    new("Иванов",  "Иван",   "CS-101", 2, 4.5, new DateOnly(2023, 9, 1)),
    new("Петров",  "Пётр",   "CS-101", 2, 4.5, new DateOnly(2023, 9, 1)),
    new("Сидоров", "Сидор",  "CS-20",  1, 3.8, new DateOnly(2024, 9, 1)),
    new("Алексеев","Алексей","CS-101", 2, 3.8, new DateOnly(2023, 9, 1)),
    new("Борисов", "Борис",  "CS-20",  1, 4.2, new DateOnly(2024, 9, 1)),
    new("иванов",  "Игорь",  "CS-101", 3, 4.5, new DateOnly(2022, 9, 1)),
    new("Кузнецов","Никита", "CS-101", 3, 4.0, new DateOnly(2022, 9, 1)),
    new("Лебедев", "Лев",    "CS-20",  4, 4.5, new DateOnly(2021, 9, 1)),
    new("Морозов", "Михаил", "CS-20",  4, 3.8, new DateOnly(2021, 9, 1)),
    new("Новиков", "Николай","CS-101", 1, 4.2, new DateOnly(2024, 9, 1)),
    new("Орлов",   "Олег",   "CS-20",  2, 4.0, new DateOnly(2023, 9, 1)),
    new("Павлов",  "Павел",  "CS-101", 1, 4.0, new DateOnly(2024, 9, 1)),
];

// 1. Рейтинг по Gpa / Gpa rating
IOrderedEnumerable<Student> ByGpaRating(IEnumerable<Student> src) =>
    src.OrderByDescending(s => s.Gpa)
      .ThenBy(s => s.LastName, StringComparer.OrdinalIgnoreCase)
      .ThenBy(s => s.FirstName, StringComparer.OrdinalIgnoreCase);

// 2. По группе, затем по курсу / By group, then by course
IOrderedEnumerable<Student> ByGroupThenCourse(IEnumerable<Student> src) =>
    src.OrderBy(s => s.Group, StringComparer.OrdinalIgnoreCase)
       .ThenBy(s => s.Course);

// 3. Сначала первокурсники / Freshmen first
IOrderedEnumerable<Student> FreshmenFirst(IEnumerable<Student> src) =>
    src.OrderBy(s => s.Course)
       .ThenByDescending(s => s.Gpa)
       .ThenBy(s => s.LastName, StringComparer.OrdinalIgnoreCase);

// 4. Ретроспектива зачисления / Enrollment retrospective
IEnumerable<Student> EnrollmentRetrospective(IEnumerable<Student> src) =>
    src.OrderByDescending(s => s.EnrolledOn).Reverse();

// 5. Кастомный компаратор групп / Custom group comparer
IOrderedEnumerable<Student> ByCustomGroupComparer(IEnumerable<Student> src) =>
    src.OrderBy(s => s.Group, new GroupComparer())
       .ThenBy(s => s.LastName, StringComparer.OrdinalIgnoreCase);

// Вывод одной строки / Print a single row
static void Print(Student s) =>
    Console.WriteLine($"  [{s.Group}, курс {s.Course}, Gpa {s.Gpa:0.0}] " +
                      $"{s.LastName}, {s.FirstName} (зачислен {s.EnrolledOn:O})");

// Раздел / Section
static void Section(string title) =>
    Console.WriteLine($"\n=== {title} ===");

// Запуск всех разделов / Run all sections
Section("1. Рейтинг по Gpa / Gpa rating");
foreach (var s in ByGpaRating(students)) Print(s);

Section("2. По группе, затем по курсу / By group, then by course");
foreach (var s in ByGroupThenCourse(students)) Print(s);

Section("3. Сначала первокурсники / Freshmen first");
foreach (var s in FreshmenFirst(students)) Print(s);

Section("4. Ретроспектива зачисления / Enrollment retrospective");
foreach (var s in EnrollmentRetrospective(students)) Print(s);

Section("5. Кастомный компаратор групп / Custom group comparer");
foreach (var s in ByCustomGroupComparer(students)) Print(s);

// Эксперимент: стабильность / Stability demo
Section("Стабильность / Stability");
var sameGpa = students.Where(s => s.Gpa == 4.5).ToList();
Console.WriteLine("До сортировки / Before:");
foreach (var s in sameGpa) Print(s);
var stable = sameGpa
    .OrderBy(s => s.LastName, StringComparer.OrdinalIgnoreCase) // вторичный / secondary
    .OrderBy(s => s.Gpa);                                       // первичный / primary
Console.WriteLine("После OrderBy(LastName).OrderBy(Gpa) — фамилии «выживают» / After:");
foreach (var s in stable) Print(s);

// Эксперимент: два OrderBy подряд / Two OrderBy in a row
Section("Ошибка: два OrderBy подряд / Bug: two OrderBy");
var wrong = students.OrderBy(s => s.Course).OrderBy(s => s.Gpa);
var right = students.OrderBy(s => s.Course).ThenBy(s => s.Gpa);
Console.WriteLine("OrderBy(Course).OrderBy(Gpa) — только Gpa:");
foreach (var s in wrong.Take(4)) Print(s);
Console.WriteLine("OrderBy(Course).ThenBy(Gpa) — курс + Gpa:");
foreach (var s in right.Take(4)) Print(s);

// Эксперимент: отложенное выполнение / Deferred execution
Section("Отложенное выполнение / Deferred execution");
var deferred = students.OrderBy(s => s.Gpa);
students.Add(new("Зайцев", "Захар", "CS-101", 1, 5.0, new DateOnly(2024, 9, 1)));
Console.WriteLine("После добавления 5.0 в исходный список — он виден в отложенном:");
foreach (var s in deferred.Take(3)) Print(s);

var snapshot = students.OrderBy(s => s.Gpa).ToList();
students.Add(new("Яковлев","Яков","CS-20",4,5.0,new DateOnly(2021,9,1)));
Console.WriteLine("После .ToList() — снимок не меняется:");
foreach (var s in snapshot.Take(3)) Print(s);
```

Разбор по строкам. Файл начинается с `using` для `System`, `System.Collections.Generic`, `System.Linq` — все три нужны: `DateOnly` и `Console` из `System`, `IComparer<>` и `List<>` из `Collections.Generic`, операторы LINQ из `Linq`. Модель `Student` объявлена `record`: это даёт immutable-семантику и автогенерируемый `ToString`, удобный для отладки; поля выбраны так, чтобы покрыть все типы ключей урока — `int`, `double`, `DateOnly`, `string`. `GroupComparer` — центральная часть задания: он реализует `IComparer<string>`, потому что тип ключа `Group` — `string` (не `Student`), и именно этот тип ожидается перегрузкой `OrderBy<TSource, TKey>(..., IComparer<TKey>)`. Внутри `Compare` сначала обрабатываются `null` (по соглашению null меньше не-null — выбор, который нужно явно задокументировать), затем через вспомогательный `TrySplit` строка разбивается по дефису с помощью pattern matching `is [var p, var n]` — это идиома C# 12 для списковых шаблонов. Числовая часть парсится `int.TryParse`, что делает компаратор устойчивым: если формат не подходит, срабатывает запасной лексикографический путь без исключений. Сравнение префикса идёт через `StringComparison.OrdinalIgnoreCase`, числовой части — через `int.CompareTo`, что и даёт желанный порядок `CS-101 < CS-20` (лексикографически было бы наоборот).

Пять локальных функций иллюстрируют все операторы урока. `ByGpaRating` показывает `OrderByDescending` + двойной `ThenBy` с `StringComparer.OrdinalIgnoreCase` — это позволяет «иванов» (с маленькой буквы) встать рядом с «Иванов», демонстрируя полезность готового компаратора. `ByGroupThenCourse` — простейшая цепочка первичный/вторичный. `FreshmenFirst` комбинирует `OrderBy` (по возрастанию курса) с `ThenByDescending` (по убыванию балла внутри курса) — это типичный «топ внутри категории». `EnrollmentRetrospective` намеренно использует `.Reverse()` после `OrderByDescending`, чтобы показать, что `Reverse` инвертирует уже отсортированную последовательность: для уникальных ключей это совпадёт с прямым `OrderBy`, для равных — нет, что отмечено в комментарии. `ByCustomGroupComparer` применяет наш `GroupComparer` и показывает, как кастомное правило заменяет дефолтное. Все функции возвращают `IOrderedEnumerable<Student>` (или `IEnumerable<Student>` для `Reverse`), а не `List<Student>`, что подчёркивает отложенность. Эксперимент со стабильностью намеренно использует приём «вторичный `OrderBy` раньше первичного» — он работает только благодаря стабильности LINQ и наглядно показывает, как фамильный порядок выживает для равных Gpa; в реальном коде лучше писать `ThenBy`, но здесь приём полезен для демонстрации. Эксперимент с двумя `OrderBy` контрастирует ошибочный и правильный варианты, а эксперимент с `deferred`/`snapshot` материализует разницу между запросом и снимком: добавление элемента в исходный список меняет вывод отложенного запроса, но не снимок `.ToList()`. Конструкция `Take(3)` используется для компактности вывода. Все ключевые концепции урока — первичный/вторичный ключ, стабильность, отложенное выполнение, компаратор для типа ключа, `Reverse` как инверсия, `OrderBy` vs `OrderByDescending` — задействованы и видны в коде.

#### Задания на углубление (бонус)

1. Реализуйте `GroupComparer` через `IComparisonComparer<string>` на базе `Comparison<string>` (делегат) и используйте перегрузку `OrderBy` с `Comparison` через метод-помощник `Comparer<string>.Create`. Сравните читаемость двух подходов.
2. Добавьте шестой запрос: отсортируйте студентов по «длительности обучения» — вычислите `(DateOnly.FromDateTime(DateTime.Today) - EnrolledOn).Days` и отсортируйте по убыванию; при равных днях — по фамилии. Объясните, почему вычисление внутри key-selector не нарушает отложенное выполнение, но может привести к повторным вычислениям при многократном перечислении.
3. Сравните производительность `OrderBy(A).ThenBy(B).ThenBy(C)` и `OrderBy(s => (s.A, s.B, s.C))` (кортеж как составной ключ) на списке из 100 000 случайно сгенерированных студентов через `BenchmarkDotNet`. Объясните результат: сколько проходов сортировки делает каждый вариант.
4. Реализуйте `ThenByDescending` для ключа `EnrolledOn` в запросе `ByGpaRating` как четвёртый уровень и объясните, почему порядок уровней критичен: первый `ThenBy` «главнее» второго для равных первичных ключей.

---

## Statement in English / Постановка на английском

#### Context & motivation

Imagine you are preparing a report for a university department. You have a list of students: each has a last name, a group code (for example `CS-101`), a course year (1–4), a grade point average (`Gpa`, of type `double`, for instance 4.5), and an enrollment date (`EnrolledOn`, `DateOnly`). The list is small, but it arrives from an external source in arbitrary order, and you need to present it in several shapes: a ranking by grade point average (best first), an alphabetical list broken down by group, a list of "newcomers" (younger by course — first-year students first), and a retrospective view — from the most recently enrolled to the earliest, to assess admission dynamics. The same set of data must be sorted in at least five ways, and some of those ways require secondary criteria: when `Gpa` is equal, the last name comes first alphabetically, and when the last name is also equal, the first name; when sorting by group, the secondary key is the course year.

In real projects sorting is almost never one-dimensional. Rankings, leaderboards, task queues, log files — composite keys appear everywhere. LINQ provides a convenient declarative apparatus for this: you describe the criteria as a chain of methods, and the framework handles the comparison mechanics. But that apparatus demands understanding: `ThenBy` does not exist without `OrderBy`, several `OrderBy` calls in a row do not produce a composite sort but only the last one, `Reverse` is not a sort but an inversion, and stability is not an abstract property but a working tool that lets you sort by a secondary key before the primary one. In this assignment you will walk through all those situations in practice and see for yourself how the operators behave. You will also meet comparers: some strings must be compared case-insensitively, and some — by a special rule (for instance, group `CS-101` must come before `CS-20`, because the logical order of groups does not coincide with the lexicographic one). For that you will have to implement `IComparer<string>` yourself and feed it correctly into the `OrderBy(..., IComparer<TKey>)` overload.

#### What to do step by step

1. Create a new .NET 8 console project with top-level statements: run `dotnet new console -n SortingHomework -o SortingHomework --framework net8.0` inside the `modules/M08/homework` directory (create it if it does not exist). Move into the project folder: `cd SortingHomework`. Verify that `dotnet build` succeeds without warnings. Set `LangVersion` to `latest` in `SortingHomework.csproj` by adding `<PropertyGroup><LangVersion>latest</LangVersion></PropertyGroup>`.

2. Replace the contents of `Program.cs` with your own code. Define the student model using a `record`: `public record Student(string LastName, string FirstName, string Group, int Course, double Gpa, DateOnly EnrolledOn);`. For convenient debugging you can rely on the auto-generated `ToString` that records already provide, which gives a readable output.

3. Prepare a source list of 12 students (variable `students` of type `List<Student>`). Choose data that highlights the subtleties: at least three pairs with the same `Gpa` (for example, two students with 4.5 and two with 3.8), at least two pairs with the same last name but different first names, and at least one pair in the same group on the same course. Include last names in different cases (for example `ivánov` and `Ivanov`) to exercise `StringComparer.OrdinalIgnoreCase`.

4. Implement and print five queries. Each query should be written as a separate local function (for example `ByGpaRating`, `ByGroupThenCourse`, `FreshmenFirst`, `EnrollmentRetrospective`, `ByCustomGroupComparer`). Print a header for each section through `Console.WriteLine` with a clear description of the criteria.

   - **Gpa rating** (`ByGpaRating`): `OrderByDescending(s => s.Gpa)`, then `ThenBy(s => s.LastName, StringComparer.OrdinalIgnoreCase)`, then `ThenBy(s => s.FirstName, StringComparer.OrdinalIgnoreCase)`. Expected: best students on top; when the grade is equal — alphabetical by last name; when the last name is also equal — by first name.
   - **By group, then by course** (`ByGroupThenCourse`): `OrderBy(s => s.Group, StringComparer.OrdinalIgnoreCase)`, then `ThenBy(s => s.Course)`. Within a group, course numbers ascend.
   - **Freshmen first** (`FreshmenFirst`): `OrderBy(s => s.Course)`, then `ThenByDescending(s => s.Gpa)`, then `ThenBy(s => s.LastName, StringComparer.OrdinalIgnoreCase)`. First-year students come before seniors; within a course — by descending grade.
   - **Enrollment retrospective** (`EnrollmentRetrospective`): `OrderByDescending(s => s.EnrolledOn)`, then apply `.Reverse()` to the whole sequence, to demonstrate that `Reverse` inverts an already ordered result. Verify that the final order coincides with `OrderBy(s => s.EnrolledOn)` — and explain in a comment why `OrderBy(...).Reverse()` and `OrderByDescending(...)` coincide only when the keys are unique.
   - **Custom group comparer** (`ByCustomGroupComparer`): implement `sealed class GroupComparer : IComparer<string>` that parses a string like `CS-101` into a prefix (`CS`) and a number (`101`) and compares first by prefix lexicographically, and when the prefix is equal — by the number numerically (so that `CS-20` comes after `CS-101`, not before it as it would under string comparison). Apply it through `OrderBy(s => s.Group, new GroupComparer())`, then `ThenBy(s => s.LastName, StringComparer.OrdinalIgnoreCase)`.

5. Demonstrate sort stability experimentally. Create a separate section `// Stable sort demonstration`. Take a sublist of students with the same `Gpa` (for example all with `4.5`), sort them first by last name, then by the same `Gpa` (through `OrderBy`): confirm that the last-name order "survives" — that is stability. Print the list before and after, add a comment with the explanation. Use the `Take`/`Where` extension methods from earlier lessons to filter the sublist.

6. Demonstrate a typical mistake: comment out a block with two `OrderBy` calls in a row, with an explanation that the second `OrderBy` cancels the first. Compare the result of `OrderBy(s => s.Course).OrderBy(s => s.Gpa)` with `OrderBy(s => s.Course).ThenBy(s => s.Gpa)`. Print both and show that the first variant is sorted only by `Gpa`, while the second is sorted by course refined by `Gpa`.

7. Demonstrate deferred execution. Store the result `var q = students.OrderBy(s => s.Gpa);` in a variable, then add another element to `students` through `students.Add(...)`, then enumerate `q` with `foreach`. Confirm that the new element appears in the output — the sort is not "frozen". Then call `.ToList()` right after `OrderBy`, repeat the addition and show that the snapshot stayed the same. Explain the connection to deferred execution in a comment.

8. Run `dotnet run` and save the output to `output.txt` (via redirection `dotnet run > output.txt`). Verify that all five queries print correctly and that the experiment sections read without errors. Run `dotnet build --no-incremental -warnaserror` to guarantee no warnings.

#### Requirements

The solution must be a single `Program.cs` file with top-level statements, without an explicit `class Program` and `static void Main`. Use C# 12: collection expressions to initialize `students` (`List<Student> students = [ ... ];` or `new(){ ... }`), pattern matching where appropriate (for example, in `GroupComparer.Compare` you can split the string with `string.Split` and `int.TryParse` with the pattern `is [var prefix, var num]`), and no file-scoped namespaces — they are unnecessary for a `Program.cs` with top-level statements. All LINQ queries must be written with method syntax (dot notation), as in the lesson, not with query syntax (`from ... orderby`). Each local function should return `IEnumerable<Student>` or `IOrderedEnumerable<Student>` — returning `List<Student>` is unnecessary and would undermine the emphasis on deferred behavior.

The `GroupComparer` must be implemented as a `sealed class GroupComparer : IComparer<string>` with a public method `int Compare(string? x, string? y)` that handles `null` correctly (by convention `null` sorts either first or last — pick one and document it explicitly in a comment). Group-string parsing must be robust: if the string does not contain a hyphen or the numeric part does not parse, fall back to a lexicographic comparison of the whole string. There must be no `throw` on "invalid" input — the comparer must be total.

The output must be readable: one line per student with the key fields, for example `[CS-101, course 2, Gpa 4.5] Ivanov, Ivan (enrolled 2023-09-01)`. Use string interpolation and, if you like, raw string literals `"""..."""` for multi-line headers. The whole code must compile under `.NET 8` without any `CS`-level warnings and pass `dotnet build -warnaserror`.

#### Pitfalls

The main trap is trying to call `ThenBy` directly on `IEnumerable<T>` without a preceding `OrderBy`. The compiler will emit CS1929: `IOrderedEnumerable<T>` is a separate interface extending `IEnumerable<T>`, and `ThenBy` is defined on it. Remember: any secondary criterion starts with `OrderBy` or `OrderByDescending`. The second trap is a chain of several `OrderBy` calls: `.OrderBy(s => s.Course).OrderBy(s => s.Gpa)` does not give a sort "first by course, then by Gpa" — the second `OrderBy` fully replaces the first, and you end up sorted only by `Gpa`, losing the course order. This is easy to miss on data where `Gpa` happens to correlate with course.

The third trap is assuming that `Reverse` sorts. `Reverse` simply inverts the current order in O(n) without invoking any comparer; this is visible on `OrderBy(x => x.Gpa).Reverse()` — the result is not equal to `OrderByDescending(x => x.Gpa)` when there are equal keys: in the first case equal keys keep the reversed original order (stability "runs backwards"), in the second they keep the original. The fourth subtlety is passing a comparer of the wrong type. The overload `OrderBy<TSource, TKey>(..., IComparer<TKey>)` expects a comparer for the key type, not for the element type. If you have `Student` and a `string Group` key, the comparer must be `IComparer<string>`, not `IComparer<Student>`. The compiler will not let you pass the wrong type, but it is easy to slip when the key and the element share a type (for example, sorting a `List<string>`).

The fifth subtlety is deferred execution. The result of `OrderBy` is not stored in memory; it is a query object that re-executes on every enumeration. If you mutate the source list and enumerate again, you see the new snapshot. If you need a "snapshot at call time", call `.ToList()` or `.ToArray()` right away. The sixth subtlety is stability and the order of calls. Stability lets you sort by a secondary key before the primary one: `.OrderBy(s => s.LastName).OrderBy(s => s.Gpa)` keeps the last-name order for equal `Gpa` only because LINQ is stable — but this is a fragile technique, and it is better to use `ThenBy`, which expresses the intent directly. The seventh subtlety is performance: every `ThenBy` adds a stable pass, so with many criteria it may be better to pack a composite key (tuple) into a single `OrderBy`. On small lists this is imperceptible, but the habit of forming "thick" keys pays off.

#### Acceptance criteria

- [ ] The `SortingHomework` project targets .NET 8 and compiles without warnings (`dotnet build -warnaserror` passes).
- [ ] `Program.cs` uses top-level statements and C# 12 (collection expressions, pattern matching in the comparer).
- [ ] The `Student` model is declared as a `record` with six fields, including `DateOnly EnrolledOn`.
- [ ] The list contains at least 12 students; the data is deliberately chosen to demonstrate equal keys and case variation.
- [ ] All five local functions are implemented: `ByGpaRating`, `ByGroupThenCourse`, `FreshmenFirst`, `EnrollmentRetrospective`, `ByCustomGroupComparer`.
- [ ] `ByGpaRating` uses `OrderByDescending(Gpa).ThenBy(LastName, OrdinalIgnoreCase).ThenBy(FirstName, OrdinalIgnoreCase)`.
- [ ] `ByGroupThenCourse` uses `OrderBy(Group, OrdinalIgnoreCase).ThenBy(Course)`.
- [ ] `FreshmenFirst` uses `OrderBy(Course).ThenByDescending(Gpa).ThenBy(LastName, OrdinalIgnoreCase)`.
- [ ] `EnrollmentRetrospective` uses `OrderByDescending(EnrolledOn).Reverse()` with a note about the relation to `OrderBy(EnrolledOn)`.
- [ ] `GroupComparer` implements `IComparer<string>`, parses `CS-101` into prefix and number, compares by prefix lexicographically and then by number numerically, and is robust to malformed input.
- [ ] Stability is demonstrated experimentally: sorting a sublist by last name and then by an equal key keeps the last-name order.
- [ ] The "two `OrderBy` in a row" mistake is shown and explained in a comment: the second cancels the first.
- [ ] Deferred execution is shown: mutating `students` after `OrderBy` is visible in enumeration; `.ToList()` freezes the snapshot.
- [ ] The output is readable, one student per line with key fields; sections are labeled.
- [ ] `dotnet run > output.txt` produces a complete log with all five sections and the three experiments.

#### Hints (no direct answer)

- To get a sublist "with the same Gpa", use `students.Where(s => s.Gpa == 4.5).ToList()` — this creates a safe copy to sort.
- To parse the group in `GroupComparer`, try `string.Split('-')` with the pattern `is [var prefix, var numStr] when int.TryParse(numStr, out var num)` — an idiomatic C# 12 form.
- Remember that `OrderBy(...).Reverse()` and `OrderByDescending(...)` coincide only when the keys are unique; on equal keys stability gives different results — use that for the comment.
- If `ThenBy` does not compile — check that you are calling it on an `IOrderedEnumerable<Student>`, not on `IEnumerable<Student>`; that means there is no `OrderBy`/`OrderByDescending` before it.
- For `DateOnly`, the default comparison is chronological, so `OrderByDescending(s => s.EnrolledOn)` yields "newest to oldest".
- Do not forget `using System; using System.Collections.Generic; using System.Linq;` at the top of the file (with top-level statements, `using` directives go at the very top).

#### Reference solution walk-through

```csharp
// C# 12 / .NET 8 — Homework M08-L03: OrderBy/ThenBy, Reverse
// Top-level statements. Bilingual comments RU+EN.

using System;
using System.Collections.Generic;
using System.Linq;

// Student model / Student model
public record Student(
    string LastName,
    string FirstName,
    string Group,    // format "CS-101"
    int Course,      // 1..4
    double Gpa,      // 0.0..5.0
    DateOnly EnrolledOn);

// Custom group comparer: prefix first, then numeric part.
// Custom group comparer: prefix first, then numeric part.
public sealed class GroupComparer : IComparer<string>
{
    public int Compare(string? x, string? y)
    {
        // null handling: null sorts first / null sorts first
        if (x is null && y is null) return 0;
        if (x is null) return -1;
        if (y is null) return 1;

        // Parse "CS-101" → prefix="CS", num=101
        if (TrySplit(x, out var px, out var nx) &&
            TrySplit(y, out var py, out var ny))
        {
            int byPrefix = string.Compare(px, py, StringComparison.OrdinalIgnoreCase);
            if (byPrefix != 0) return byPrefix;
            return nx.CompareTo(ny); // numeric compare / numeric compare
        }

        // Fallback: lexicographic / Fallback: lexicographic
        return string.Compare(x, y, StringComparison.OrdinalIgnoreCase);
    }

    private static bool TrySplit(string s, out string prefix, out int num)
    {
        prefix = s;
        num = 0;
        var parts = s.Split('-');
        if (parts is [var p, var n] && int.TryParse(n, out num))
        {
            prefix = p;
            return true;
        }
        return false;
    }
}

// Source list / Source list
List<Student> students =
[
    new("Ivanov",  "Ivan",   "CS-101", 2, 4.5, new DateOnly(2023, 9, 1)),
    new("Petrov",  "Petr",   "CS-101", 2, 4.5, new DateOnly(2023, 9, 1)),
    new("Sidorov", "Sidor",  "CS-20",  1, 3.8, new DateOnly(2024, 9, 1)),
    new("Alekseev","Alexey", "CS-101", 2, 3.8, new DateOnly(2023, 9, 1)),
    new("Borisov", "Boris",  "CS-20",  1, 4.2, new DateOnly(2024, 9, 1)),
    new("ivanov",  "Igor",   "CS-101", 3, 4.5, new DateOnly(2022, 9, 1)),
    new("Kuznetsov","Nikita","CS-101", 3, 4.0, new DateOnly(2022, 9, 1)),
    new("Lebedev", "Lev",    "CS-20",  4, 4.5, new DateOnly(2021, 9, 1)),
    new("Morozov", "Mikhail","CS-20",  4, 3.8, new DateOnly(2021, 9, 1)),
    new("Novikov", "Nikolay","CS-101", 1, 4.2, new DateOnly(2024, 9, 1)),
    new("Orlov",   "Oleg",   "CS-20",  2, 4.0, new DateOnly(2023, 9, 1)),
    new("Pavlov",  "Pavel",  "CS-101", 1, 4.0, new DateOnly(2024, 9, 1)),
];

// 1. Gpa rating / Gpa rating
IOrderedEnumerable<Student> ByGpaRating(IEnumerable<Student> src) =>
    src.OrderByDescending(s => s.Gpa)
      .ThenBy(s => s.LastName, StringComparer.OrdinalIgnoreCase)
      .ThenBy(s => s.FirstName, StringComparer.OrdinalIgnoreCase);

// 2. By group, then by course / By group, then by course
IOrderedEnumerable<Student> ByGroupThenCourse(IEnumerable<Student> src) =>
    src.OrderBy(s => s.Group, StringComparer.OrdinalIgnoreCase)
       .ThenBy(s => s.Course);

// 3. Freshmen first / Freshmen first
IOrderedEnumerable<Student> FreshmenFirst(IEnumerable<Student> src) =>
    src.OrderBy(s => s.Course)
       .ThenByDescending(s => s.Gpa)
       .ThenBy(s => s.LastName, StringComparer.OrdinalIgnoreCase);

// 4. Enrollment retrospective / Enrollment retrospective
IEnumerable<Student> EnrollmentRetrospective(IEnumerable<Student> src) =>
    src.OrderByDescending(s => s.EnrolledOn).Reverse();

// 5. Custom group comparer / Custom group comparer
IOrderedEnumerable<Student> ByCustomGroupComparer(IEnumerable<Student> src) =>
    src.OrderBy(s => s.Group, new GroupComparer())
       .ThenBy(s => s.LastName, StringComparer.OrdinalIgnoreCase);

// Print one row / Print a single row
static void Print(Student s) =>
    Console.WriteLine($"  [{s.Group}, course {s.Course}, Gpa {s.Gpa:0.0}] " +
                      $"{s.LastName}, {s.FirstName} (enrolled {s.EnrolledOn:O})");

// Section header / Section header
static void Section(string title) =>
    Console.WriteLine($"\n=== {title} ===");

// Run all sections / Run all sections
Section("1. Gpa rating");
foreach (var s in ByGpaRating(students)) Print(s);

Section("2. By group, then by course");
foreach (var s in ByGroupThenCourse(students)) Print(s);

Section("3. Freshmen first");
foreach (var s in FreshmenFirst(students)) Print(s);

Section("4. Enrollment retrospective");
foreach (var s in EnrollmentRetrospective(students)) Print(s);

Section("5. Custom group comparer");
foreach (var s in ByCustomGroupComparer(students)) Print(s);

// Stability demo / Stability demo
Section("Stability");
var sameGpa = students.Where(s => s.Gpa == 4.5).ToList();
Console.WriteLine("Before sort:");
foreach (var s in sameGpa) Print(s);
var stable = sameGpa
    .OrderBy(s => s.LastName, StringComparer.OrdinalIgnoreCase) // secondary
    .OrderBy(s => s.Gpa);                                       // primary
Console.WriteLine("After OrderBy(LastName).OrderBy(Gpa) — last names survive:");
foreach (var s in stable) Print(s);

// Bug: two OrderBy / Bug: two OrderBy
Section("Bug: two OrderBy in a row");
var wrong = students.OrderBy(s => s.Course).OrderBy(s => s.Gpa);
var right = students.OrderBy(s => s.Course).ThenBy(s => s.Gpa);
Console.WriteLine("OrderBy(Course).OrderBy(Gpa) — only Gpa:");
foreach (var s in wrong.Take(4)) Print(s);
Console.WriteLine("OrderBy(Course).ThenBy(Gpa) — course + Gpa:");
foreach (var s in right.Take(4)) Print(s);

// Deferred execution / Deferred execution
Section("Deferred execution");
var deferred = students.OrderBy(s => s.Gpa);
students.Add(new("Zaytsev", "Zakhar", "CS-101", 1, 5.0, new DateOnly(2024, 9, 1)));
Console.WriteLine("After adding a 5.0 to the source — visible in the deferred query:");
foreach (var s in deferred.Take(3)) Print(s);

var snapshot = students.OrderBy(s => s.Gpa).ToList();
students.Add(new("Yakovlev","Yakov","CS-20",4,5.0,new DateOnly(2021,9,1)));
Console.WriteLine("After .ToList() — the snapshot does not change:");
foreach (var s in snapshot.Take(3)) Print(s);
```

Walk-through line by line. The file begins with `using` directives for `System`, `System.Collections.Generic`, and `System.Linq` — all three are needed: `DateOnly` and `Console` come from `System`, `IComparer<>` and `List<>` from `Collections.Generic`, and the LINQ operators from `Linq`. The `Student` model is declared as a `record`: this gives value semantics and an auto-generated `ToString` that is handy for debugging; the fields are chosen to cover every key type from the lesson — `int`, `double`, `DateOnly`, `string`. `GroupComparer` is the centerpiece of the assignment: it implements `IComparer<string>` because the key type `Group` is `string` (not `Student`), and that is exactly what the `OrderBy<TSource, TKey>(..., IComparer<TKey>)` overload expects. Inside `Compare`, `null` is handled first (by convention null sorts before non-null — a choice that must be documented explicitly), then through the helper `TrySplit` the string is split on the hyphen using the list pattern `is [var p, var n]` — an idiomatic C# 12 form. The numeric part is parsed with `int.TryParse`, which makes the comparer robust: if the format does not fit, the fallback lexicographic path runs without exceptions. Prefix comparison uses `StringComparison.OrdinalIgnoreCase`, the numeric part uses `int.CompareTo`, and that yields the desired order `CS-101 < CS-20` (lexicographically it would be the opposite).

The five local functions illustrate every operator from the lesson. `ByGpaRating` shows `OrderByDescending` plus a double `ThenBy` with `StringComparer.OrdinalIgnoreCase` — this lets `ivanov` (lowercase) stand next to `Ivanov`, demonstrating the usefulness of a ready-made comparer. `ByGroupThenCourse` is the simplest primary/secondary chain. `FreshmenFirst` combines `OrderBy` (ascending by course) with `ThenByDescending` (descending grade within a course) — a typical "top within a category". `EnrollmentRetrospective` deliberately applies `.Reverse()` after `OrderByDescending`, to show that `Reverse` inverts an already sorted sequence: for unique keys this coincides with a plain `OrderBy`, for equal keys it does not, which is noted in the comment. `ByCustomGroupComparer` applies our `GroupComparer` and shows how a custom rule replaces the default. All functions return `IOrderedEnumerable<Student>` (or `IEnumerable<Student>` for `Reverse`), not `List<Student>`, which emphasizes the deferred nature. The stability experiment deliberately uses the trick "secondary `OrderBy` before primary" — it works only thanks to LINQ stability and visibly shows how the last-name order survives for equal Gpa; in real code you would write `ThenBy`, but here the trick is useful as a demonstration. The two-`OrderBy` experiment contrasts the buggy and the correct variant, and the `deferred`/`snapshot` experiment materializes the difference between a query and a snapshot: adding an element to the source list changes the deferred query output but not the `.ToList()` snapshot. The `Take(3)` call keeps the output compact. Every key concept of the lesson — primary/secondary key, stability, deferred execution, comparer for the key type, `Reverse` as inversion, `OrderBy` vs `OrderByDescending` — is exercised and visible in the code.

#### Going deeper (bonus)

1. Implement `GroupComparer` through `IComparisonComparer<string>` based on the `Comparison<string>` delegate and use the `OrderBy` overload that accepts `Comparison` via the `Comparer<string>.Create` helper. Compare the readability of the two approaches.
2. Add a sixth query: sort students by "length of study" — compute `(DateOnly.FromDateTime(DateTime.Today) - EnrolledOn).Days` and sort descending; on equal days, by last name. Explain why computing inside the key-selector does not break deferred execution but may cause repeated computation on multiple enumerations.
3. Compare the performance of `OrderBy(A).ThenBy(B).ThenBy(C)` and `OrderBy(s => (s.A, s.B, s.C))` (a tuple as a composite key) on a list of 100 000 randomly generated students with `BenchmarkDotNet`. Explain the result: how many sorting passes each variant performs.
4. Add `ThenByDescending` for `EnrolledOn` to `ByGpaRating` as a fourth level and explain why the order of levels is critical: the first `ThenBy` is "more important" than the second for equal primary keys.

---

#### Чек-лист сдачи / Submission checklist

- [ ] Проект `SortingHomework` создан и собирается под .NET 8 без предупреждений.
- [ ] Файл `Program.cs` отправлен вместе с `SortingHomework.csproj`.
- [ ] Вывод `dotnet run` сохранён в `output.txt` и приложен к сдаче.
- [ ] Все пять разделов запросов присутствуют и корректны.
- [ ] Эксперименты со стабильностью, ошибкой «два OrderBy» и отложенным выполнением включены.
- [ ] Кастомный `GroupComparer` реализован и применён.
- [ ] Код использует C# 12 (collection expressions, pattern matching).
- [ ] Комментарии двуязычные (RU+EN) в ключевых местах.
- [ ] The `SortingHomework` project is created and builds under .NET 8 without warnings.
- [ ] The `Program.cs` file is submitted together with `SortingHomework.csproj`.
- [ ] The `dotnet run` output is saved to `output.txt` and attached.
- [ ] All five query sections are present and correct.
- [ ] The stability, "two OrderBy" bug, and deferred-execution experiments are included.
- [ ] The custom `GroupComparer` is implemented and applied.
- [ ] The code uses C# 12 (collection expressions, pattern matching).
- [ ] Comments are bilingual (RU+EN) at key points.

#### Ресурсы / Resources

- [Microsoft Learn — Enumerable.OrderBy](https://learn.microsoft.com/dotnet/api/system.linq.enumerable.orderby)
- [Microsoft Learn — Enumerable.OrderByDescending](https://learn.microsoft.com/dotnet/api/system.linq.enumerable.orderbydescending)
- [Microsoft Learn — Enumerable.ThenBy](https://learn.microsoft.com/dotnet/api/system.linq.enumerable.thenby)
- [Microsoft Learn — Enumerable.ThenByDescending](https://learn.microsoft.com/dotnet/api/system.linq.enumerable.thenbydescending)
- [Microsoft Learn — Enumerable.Reverse](https://learn.microsoft.com/dotnet/api/system.linq.enumerable.reverse)
- [Microsoft Learn — IComparer<T>](https://learn.microsoft.com/dotnet/api/system.collections.generic.icomparer-1)
- [Microsoft Learn — StringComparer](https://learn.microsoft.com/dotnet/api/system.stringcomparer)
- [Microsoft Learn — IOrderedEnumerable<T>](https://learn.microsoft.com/dotnet/api/system.linq.iorderedenumerable-1)

---

[← К уроку M08-L03](lesson-M08-L03-orderby-thenby.md) | [⬆ К модулю M08](../README.md) | [Следующее ДЗ →](homework-M08-L04-groupby-tolookup.md)
