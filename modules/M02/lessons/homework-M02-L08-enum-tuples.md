---
[← К уроку M02-L08](lesson-M02-L08-enum-tuples.md) | [⬆ К модулю M02](../README.md) | [Следующее ДЗ →](none)
---

### Домашнее задание M02-L08: enum и кортежи (кратко) / Homework M02-L08: enum and tuples (brief)

**Урок / Lesson:** M02-L08
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 3/5
**Цель / Goal:** (RU) Научиться объявлять `enum` с произвольным базовым типом, корректно использовать `[Flags]` со степенями двойки, проверять флаги через `HasFlag` и `&`, возвращать из методов несколько значений через кортежи с именованными полями, деконструировать кортежи и обоснованно выбирать `tuple` vs `record` vs `class`. (EN) Learn to declare an `enum` with a custom underlying type, correctly use `[Flags]` with powers of two, check flags via `HasFlag` and `&`, return several values from methods via tuples with named fields, deconstruct tuples, and justify choosing `tuple` vs `record` vs `class`.

#### Связь с уроком / Connection to the lesson
(RU) Домашнее задание напрямую опирается на примеры урока: объявление `PowerMode` и `AccessRights`, метод `CreateUser`, возвращающий кортеж, деконструкция `var (userName, userRights) = CreateUser(...)`, сравнение кортежей `pointA == pointB` и контраст кортежа с `record Point`. Здесь вы повторите все четыре «частые ошибки» — неявное приведение `enum` к `int`, забытые степени двойки в `[Flags]`, проверку флага через `==` и упаковку кортежа в `object`.
(EN) This homework directly builds on the lesson examples: declaring `PowerMode` and `AccessRights`, the tuple-returning `CreateUser` method, deconstruction `var (userName, userRights) = CreateUser(...)`, tuple comparison `pointA == pointB`, and the contrast between a tuple and `record Point`. You will reproduce all four "common mistakes" — implicit `enum`-to-`int` cast, forgotten powers of two in `[Flags]`, checking a flag with `==`, and boxing a tuple as `object`.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация
Вы разрабатываете маленькую утилиту командной строки `AccessManager`, которая моделирует права доступа к файлам в учебном «хранилище». Хранилище хранит файлы разных категорий (конфиги, логи, данные, исполняемые файлы), и у каждой категории есть «права по умолчанию». Пользователь утилиты запрашивает операцию (чтение, запись, исполнение, удаление), а утилита должна ответить, разрешена ли операция, и если нет — объяснить причину понятным текстом.

Эта задача идеально ложится на темы урока, потому что здесь есть и закрытое множество категорий (это `enum`), и комбинации прав (это `[Flags]`), и необходимость вернуть из метода сразу несколько разнородных значений: разрешено/запрещено и причину (это кортеж). В отличие от `out`-параметров, кортеж позволяет вернуть результат одной строкой и сразу деконструировать его в переменные, что делает код линейным и читаемым. Также в задаче есть «журналирование событий»: каждой ошибке сопоставляется уровень серьёзности, причём значений всего четыре и они компактны — это повод потренировать `enum` с базовым типом `byte`, как в примере `enum PowerMode : byte` из урока.

Наконец, файлы в хранилище — это сущность со стабильной семантикой: у файла есть путь, категория и текущие права, и вы захотите сравнивать файлы по значению, создавать копии с изменёнными правами через `with`. Это ровно тот случай, когда урок предписывает выбрать `record`, а не кортеж. В конце задания вы должны уметь объяснить, почему для «результата проверки доступа» выбран кортеж, а для «файла в хранилище» — `record`. Такое обоснование — часть приёмки.

#### Что нужно сделать (пошагово)
1. Создайте новый консольный проект .NET 8. Выполните в терминале команду `dotnet new console -n AccessManager -o AccessManager --framework net8.0`. Перейдите в папку проекта `cd AccessManager` и откройте `Program.cs`. Удалите шаблонный `Console.WriteLine("Hello, World!");`.
2. Используйте top-level statements (программа без явного `class Program` и `static void Main`), как в примере урока.
3. Объявите обычный `enum FileKind { Config, Log, Data, Executable }` с базовым типом по умолчанию (`int`). Это аналог `PowerMode` из урока.
4. Объявите перечисление с кастомным базовым типом: `enum Severity : byte { Info = 0, Warning = 1, Error = 2, Critical = 3 }`. Повторите приём из урока (`enum PowerMode : byte`).
5. Объявите `[Flags] enum Permission { None = 0, Read = 1, Write = 2, Execute = 4, Delete = 8 }`. Обязательно степени двойки и значение `None = 0` — это best practice из урока.
6. Объявите `record FileEntry(string Path, FileKind Kind, Permission Rights);` — стабильная семантика, сравнение по значению, `with`-выражения.
7. Напишите метод `(Permission Default, string Description) GetDefaultRights(FileKind kind)`, который по категории возвращает кортеж «права по умолчанию + описание». Например, для `FileKind.Log` — `Permission.Read` и описание «логи только для чтения». Реализуйте выбор через `switch`-выражение (pattern matching).
8. Напишите метод `(bool Allowed, string Reason) TryAccess(Permission current, Permission required)`, который возвращает кортеж «разрешено ли + причина». Внутри используйте `HasFlag` для проверки каждого бита `required` в `current`. Если не хватает хотя бы одного права, верните `false` и строку вида `"missing: Write, Delete"` со списком недостающих прав.
9. Напишите метод `(Severity Severity, string Message) ClassifyEvent(int errorCode)`, который по коду ошибки возвращает кортеж «уровень серьёзности + сообщение». Используйте pattern matching по диапазонам: коды 0 — `Info`, 1–99 — `Warning`, 100–199 — `Error`, 200+ — `Critical`. Подтвердите, что `Severity` имеет базовый тип `byte`, выведя `(byte)severity` в консоль.
10. Напишите метод `(FileEntry Entry, Permission Granted) Grant(FileEntry entry, Permission extra)`, который возвращает кортеж из обновлённого `FileEntry` (через `with { Rights = entry.Rights | extra }`) и фактически выданных прав.
11. В главной программе создайте несколько `FileEntry`, вызовите `GetDefaultRights`, `TryAccess`, `ClassifyEvent`, `Grant`. Деконструируйте результаты через `var (allowed, reason) = TryAccess(...)` и печатайте их.
12. Продемонстрируйте работу с `[Flags]`: создайте `Permission mine = Permission.Read | Permission.Write;`, напечатайте `mine.ToString()` (должно быть `Read, Write`), проверьте `mine.HasFlag(Permission.Write)` и побитовую проверку `(mine & Permission.Execute) == Permission.Execute`.
13. Продемонстрируйте явное приведение: `int kindAsInt = (int)FileKind.Log;` — убедитесь, что неявное приведение `int x = FileKind.Log;` не компилируется, и закомментируйте эту строку с пояснением.
14. Продемонстрируйте сравнение кортежей: `var a = (Code: 1, Msg: "ok"); var b = (Code: 1, Msg: "ok"); Console.WriteLine(a == b);` — должно быть `True`.
15. Продемонстрируйте контраст с `record`: создайте два `FileEntry` с одинаковыми полями, сравните через `==`, создайте копию через `with`.
16. (Бонус для углубления, но не обязательно для сдачи) Покажите, что имя поля кортежа теряется при упаковке в `object`: `object boxed = (Code: 1, Msg: "x");` и попробуйте обратиться к полю — объясните в комментарии, почему это антипаттерн.
17. Соберите и запустите проект: `dotnet build`, затем `dotnet run`. Убедитесь, что вывод соответствует ожидаемому.

Ожидаемый вывод (ключевые строки):
```
mode default for Log = Read (логи только для чтения)
mine = Read, Write
can write? True
can exec? False
access Read+Write to Read-only: False — missing: Write
severity for code 150 = Error (byte=2)
files equal by value? True
updated = FileEntry { Path = app.log, Kind = Log, Rights = Read, Write }
```

#### Требования к решению
Решение должно компилироваться под C# 12 / .NET 8 без предупреждений уровня error и запускаться через `dotnet run`. Все перечисления объявлены явно, с понятными именами; для `[Flags]` соблюдены степени двойки и присутствует `None = 0`. Кортежи используются только там, где это уместно по уроку: для возврата 2–4 значений из метода внутри одного вызова; для сущности «файл» выбран `record`, и это обосновано комментарием.

Приведение `enum` к числу должно быть явным (`(int)`, `(byte)`); неявное приведение должно отсутствовать или быть закомментированным с пояснением «не компилируется». Проверка флагов — через `HasFlag` или `(x & flag) == flag`, но не через `==` для комбинированных значений. Кортежи не должны упаковываться в `object` (кроме демонстрационного бонуса с пояснением, что это антипаттерн). Деконструкция `var (...) = Method(...)` используется минимум в трёх местах. Код оформлен через top-level statements, содержит двуязычные комментарии (RU + EN), как в примере урока. Файл `Program.cs` должен быть самодостаточным — один файл, без внешних зависимостей, кроме стандартной библиотеки.

#### Тонкости и подводные камни
- **Неявное приведение `enum` → `int` не компилируется.** Это первая «частая ошибка» из урока. Пишите `(int)FileKind.Log`, а не `int x = FileKind.Log;`. Базовый тип `byte` аналогично требует `(byte)severity`.
- **Степени двойки в `[Flags]` обязательны.** Если вы случайно зададите `Read = 1, Write = 2, Execute = 3` (не степень двойки), то `Execute` «перекроет» `Read` и `Write`, и `HasFlag` начнёт давать ложные результаты. Всегда `1, 2, 4, 8, 16...` и обязательно `None = 0`.
- **Проверка флага через `==` теряет комбинации.** `mine == Permission.Write` даст `False`, если `mine = Read | Write`. Правильно — `mine.HasFlag(Permission.Write)` или `(mine & Permission.Write) == Permission.Write`. Урок явно предупреждает об этом.
- **`ToString()` без `[Flags]` показывает число.** Если забыть атрибут, то для `Read | Write` вы получите `3`, а не `Read, Write`. В задании атрибут обязателен.
- **Имя поля кортежа теряется при упаковке в `object`.** `(x as object).Code` не скомпилируется, потому что тип поля известен только статически. Передавайте кортежи типизированно, без `object`.
- **Кортеж vs `record`.** Кортеж — «временная упаковка» для 2–4 значений в пределах вызова; `record` — когда есть стабильная семантика (`FileEntry`), нужно сравнение по значению и `with`. Не используйте кортеж вместо типа.
- **Базовый тип `byte` экономит память**, но не забывайте, что приведение всё равно явное. В `ClassifyEvent` верните `Severity` и покажите `(byte)severity`, чтобы убедиться, что значения 0–3 действительно уложились в `byte`.
- **`HasFlag` боксирует аргумент** в старых версиях; в .NET 8 это оптимизировано, но при очень горячих циклах предпочитайте `(x & flag) == flag`. Для учебного задания разница несущественна, но знать полезно.

#### Критерии приёмки
- [ ] Проект создаётся командой `dotnet new console` под .NET 8 и собирается без ошибок.
- [ ] Объявлен `enum FileKind` с базовым типом по умолчанию `int`.
- [ ] Объявлен `enum Severity : byte` с кастомным базовым типом и значениями 0–3.
- [ ] Объявлен `[Flags] enum Permission` со степенями двойки и `None = 0`.
- [ ] Метод `GetDefaultRights` возвращает кортеж `(Permission, string)` и реализован через `switch`-выражение.
- [ ] Метод `TryAccess` возвращает кортеж `(bool, string)` и внутри использует `HasFlag`.
- [ ] Метод `ClassifyEvent` возвращает кортеж `(Severity, string)` и использует pattern matching по диапазонам.
- [ ] Метод `Grant` возвращает кортеж `(FileEntry, Permission)` и использует `with` для обновления `record`.
- [ ] Объявлен `record FileEntry` со стабильной семантикой; выбор `record` обоснован комментарием.
- [ ] В главной программе минимум три деконструкции кортежа через `var (...) = ...`.
- [ ] Продемонстрировано явное приведение `(int)` и `(byte)`; неявное закомментировано с пояснением.
- [ ] Продемонстрирована проверка флага и через `HasFlag`, и через `&`.
- [ ] `Permission.Read | Permission.Write` печатается как `Read, Write` (атрибут `[Flags]` присутствует).
- [ ] Продемонстрировано сравнение кортежей по значению (`a == b` → `True`).
- [ ] Продемонстрирован контраст с `record`: сравнение и `with`.
- [ ] Код содержит двуязычные комментарии RU + EN, как в примере урока.
- [ ] Вывод `dotnet run` соответствует ожидаемым ключевым строкам.

#### Подсказки (без прямого ответа)
- Для `switch`-выражения по `FileKind` используйте синтаксис `kind switch { FileKind.Log => (Permission.Read, "логи только чтения"), ... }`.
- Список недостающих прав можно собрать, проверяя каждый флаг по очереди и добавляя имя в `List<string>`, затем `string.Join(", ", list)`.
- `HasFlag` возвращает `bool` для одного флага; чтобы найти ВСЕ недостающие, переберите `Read, Write, Execute, Delete`.
- Для pattern matching по диапазонам в `ClassifyEvent` используйте `errorCode switch { 0 => ..., >= 1 and < 100 => ..., ... }`.
- Чтобы обновить права в `record`, примените `entry with { Rights = entry.Rights | extra }` — побитовое ИЛИ добавляет биты.
- Если `int x = FileKind.Log;` не компилируется — это правильно, так и должно быть; закомментируйте и поясните.

#### Эталонное решение (разбор)
```csharp
// C# 12 / .NET 8 — AccessManager: enum + tuples + record
// Top-level statements / Top-level инструкции

using System;
using System.Collections.Generic;

// --- enum: категории файлов (базовый тип int по умолчанию) ---
// File categories, default underlying type int
enum FileKind
{
    Config,      // 0
    Log,         // 1
    Data,        // 2
    Executable   // 3
}

// --- enum с кастомным базовым типом byte (экономия памяти) ---
// Custom underlying type byte to save memory
enum Severity : byte
{
    Info     = 0,
    Warning  = 1,
    Error    = 2,
    Critical = 3
}

// --- [Flags]: комбинации через побитовое ИЛИ, степени двойки + None=0 ---
// [Flags]: bitwise OR combinations, powers of two + None=0
[Flags]
enum Permission
{
    None    = 0,
    Read    = 0b0001, // 1
    Write   = 0b0010, // 2
    Execute = 0b0100, // 4
    Delete  = 0b1000  // 8
}

// --- record: стабильная семантика, сравнение по значению, with-выражения ---
// record: stable semantics, value equality, with-expressions
record FileEntry(string Path, FileKind Kind, Permission Rights);

// --- Метод, возвращающий кортеж (права по умолчанию + описание) ---
// Method returning a tuple (default rights + description)
(Permission Default, string Description) GetDefaultRights(FileKind kind) =>
    kind switch
    {
        FileKind.Config    => (Permission.Read | Permission.Write, "конфиги: чтение и запись"),
        FileKind.Log       => (Permission.Read, "логи только для чтения"),
        FileKind.Data      => (Permission.Read | Permission.Write, "данные: чтение и запись"),
        FileKind.Executable=> (Permission.Read | Permission.Execute, "исполняемые: чтение и запуск"),
        _                  => (Permission.None, "неизвестная категория")
    };

// --- Метод с несколькими результатами через кортеж ---
// Multiple-return method via tuple
(bool Allowed, string Reason) TryAccess(Permission current, Permission required)
{
    var missing = new List<string>();
    if (required.HasFlag(Permission.Read)    && !current.HasFlag(Permission.Read))    missing.Add("Read");
    if (required.HasFlag(Permission.Write)   && !current.HasFlag(Permission.Write))   missing.Add("Write");
    if (required.HasFlag(Permission.Execute) && !current.HasFlag(Permission.Execute)) missing.Add("Execute");
    if (required.HasFlag(Permission.Delete)  && !current.HasFlag(Permission.Delete))  missing.Add("Delete");
    return missing.Count == 0
        ? (true, "доступ разрешён")
        : (false, $"missing: {string.Join(", ", missing)}");
}

// --- Классификация события: кортеж (Severity, Message) + pattern matching ---
// Event classification: tuple (Severity, Message) + pattern matching
(Severity Severity, string Message) ClassifyEvent(int errorCode) =>
    errorCode switch
    {
        0                 => (Severity.Info, "всё в порядке"),
        >= 1 and < 100    => (Severity.Warning, "некритичная проблема"),
        >= 100 and < 200  => (Severity.Error, "ошибка"),
        _                 => (Severity.Critical, "критический сбой")
    };

// --- Выдача прав: кортеж (обновлённый FileEntry, фактически выданные права) ---
// Grant rights: tuple (updated FileEntry, actually granted rights)
(FileEntry Entry, Permission Granted) Grant(FileEntry entry, Permission extra) =>
    (entry with { Rights = entry.Rights | extra }, entry.Rights | extra);

// === Демонстрация / Demo ===
FileKind kind = FileKind.Log;
var (defRights, defDesc) = GetDefaultRights(kind);
Console.WriteLine($"mode default for {kind} = {defRights} ({defDesc})");

// [Flags]: комбинируем права / combine rights
Permission mine = Permission.Read | Permission.Write;
Console.WriteLine($"mine = {mine}");                                  // Read, Write
Console.WriteLine($"can write? {mine.HasFlag(Permission.Write)}");    // True
bool canExec = (mine & Permission.Execute) == Permission.Execute;
Console.WriteLine($"can exec? {canExec}");                            // False

// Попытка доступа: деконструкция кортежа / access attempt: tuple deconstruction
FileEntry log = new("app.log", FileKind.Log, Permission.Read);
var (allowed, reason) = TryAccess(log.Rights, Permission.Read | Permission.Write);
Console.WriteLine($"access Read+Write to Read-only: {allowed} — {reason}");

// Классификация события + кастомный базовый тип byte
var (sev, msg) = ClassifyEvent(150);
Console.WriteLine($"severity for code 150 = {sev} (byte={(byte)sev}) — {msg}");

// Явное приведение enum к int; неявное не компилируется
int kindAsInt = (int)FileKind.Log;
// int bad = FileKind.Log; // ОШИБКА компиляции: нельзя неявно / compile error: no implicit cast
Console.WriteLine($"FileKind.Log as int = {kindAsInt}");

// Сравнение кортежей по значению / tuples equal by value
var a = (Code: 1, Msg: "ok");
var b = (Code: 1, Msg: "ok");
Console.WriteLine($"tuples equal by value? {a == b}");               // True

// Контраст с record: сравнение и with / contrast with record
FileEntry f1 = new("x", FileKind.Data, Permission.Read);
FileEntry f2 = new("x", FileKind.Data, Permission.Read);
Console.WriteLine($"files equal by value? {f1 == f2}");             // True
var (updated, granted) = Grant(log, Permission.Write);
Console.WriteLine($"updated = {updated}");                          // Rights = Read, Write
```

Разбор по строкам. Объявление `enum FileKind` повторяет пример `PowerMode` из урока: базовый тип `int` по умолчанию, значения начинаются с нуля, каждое имя сопоставлено числу. `enum Severity : byte` тренирует приём смены базового типа из урока (`enum PowerMode : byte`) — это экономит память, а в выводе мы убеждаемся, что значение уложилось в `byte` через `(byte)sev`. `[Flags] enum Permission` повторяет `AccessRights` из урока: степени двойки заданы бинарными литералами `0b0001...0b1000`, присутствует `None = 0` — это обязательный best practice, иначе комбинации «слипаются», как предупреждает урок.

`record FileEntry` — это прямая отсылка к `record Point` из урока: стабильная семантика, сравнение по значению, `with`-выражения. Здесь применена концепция «кортеж vs record»: для файла выбран `record`, потому что у него есть постоянная бизнес-семантика и мы хотим `with`. Метод `GetDefaultRights` возвращает кортеж `(Permission, string)` — два значения внутри одного вызова, как `CreateUser` в уроке; реализован через `switch`-выражение (pattern matching, C# 12). Метод `TryAccess` возвращает `(bool, string)` и внутри использует `HasFlag` для проверки каждого бита — это та самая правильная проверка флагов, которую урок противопоставляет ошибочной проверке через `==`. Сбор недостающих прав в `List<string>` с последующим `string.Join` даёт читаемую причину отказа.

`ClassifyEvent` возвращает `(Severity, string)` и использует pattern matching по диапазонам (`>= 1 and < 100`), закрепляя и кастомный базовый тип `byte`, и кортеж как носитель нескольких значений. `Grant` возвращает `(FileEntry, Permission)` и применяет `with { Rights = ... | extra }` — побитовое ИЛИ добавляет биты, как `AccessRights.Read | AccessRights.Write` в уроке. В демонстрации три деконструкции (`var (defRights, defDesc)`, `var (allowed, reason)`, `var (sev, msg)`, `var (updated, granted)`) — это `var (code, msg) = GetResult()` из урока, применённый многократно. Явное приведение `(int)FileKind.Log` и закомментированная попытка неявного `int bad = FileKind.Log;` воспроизводят первую «частую ошибку» урока. Сравнение `a == b` для кортежей повторяет `pointA == pointB` из урока и подтверждает, что кортежи сравниваются по значению. Контраст с `record` (`f1 == f2`, `Grant` через `with`) замыкает тему выбора типа.

#### Задания на углубление (бонус)
1. Добавьте парсинг прав из строки: метод `Permission ParsePermission(string s)` принимает `"read,write"` и возвращает комбинацию через `Enum.Parse<Permission>(s, ignoreCase: true)`. Обработайте случай, когда передано число `5` — `Enum.Parse` должен разобрать его как `Read, Write`. Подумайте, почему это работает именно с `[Flags]`.
2. Реализуйте метод `(Permission Recommended, Permission Forbidden) Analyze(FileKind kind)`, который возвращает два набора прав в одном кортеже. Подумайте, не пора ли здесь вместо кортежа ввести `record AccessPolicy` — обоснуйте выбор.
3. Покажите антипаттерн из урока: упакуйте кортеж в `object boxed = (Code: 1, Msg: "x");` и попробуйте обратиться к `Code`. Объясните в комментарии, почему имя поля теряется и почему урок рекомендует передавать кортежи типизированно.
4. Измерьте (мысленно) разницу между `HasFlag` и `(x & flag) == flag` в горячем цикле `for (int i = 0; i < 1_000_000; i++)`. В .NET 8 `HasFlag` не боксирует, но знание этого отличия — часть культуры C#.

---

## Statement in English / Постановка на английском

#### Context & motivation
You are building a small command-line utility called `AccessManager` that models file access rights in an educational "store". The store keeps files of different categories (configs, logs, data, executables), and each category has "default rights". The user of the utility requests an operation (read, write, execute, delete), and the utility must answer whether the operation is allowed and, if not, explain the reason in plain text.

This task maps perfectly onto the lesson topics, because here you have both a closed set of categories (an `enum`) and combinations of rights (a `[Flags]` enum), and you need to return several heterogeneous values from a method at once: allowed/denied and the reason (a tuple). Unlike `out` parameters, a tuple lets you return the result in a single line and immediately deconstruct it into variables, which keeps the code linear and readable. The task also includes "event logging": every error is associated with a severity level, and since there are only four values and they are compact, this is a good opportunity to practice an `enum` with an underlying type of `byte`, exactly like the `enum PowerMode : byte` example from the lesson.

Finally, files in the store are an entity with stable semantics: a file has a path, a category, and current rights, and you will want to compare files by value and create copies with modified rights via `with`. This is precisely the case where the lesson prescribes choosing a `record` over a tuple. By the end of the task you must be able to explain why a tuple was chosen for the "access check result" while a `record` was chosen for the "file in the store". That justification is part of the acceptance criteria.

#### What to do step by step
1. Create a new .NET 8 console project. In the terminal run `dotnet new console -n AccessManager -o AccessManager --framework net8.0`. Change into the project folder `cd AccessManager` and open `Program.cs`. Remove the template `Console.WriteLine("Hello, World!");`.
2. Use top-level statements (no explicit `class Program` and `static void Main`), as in the lesson example.
3. Declare a plain `enum FileKind { Config, Log, Data, Executable }` with the default underlying type (`int`). This mirrors `PowerMode` from the lesson.
4. Declare an enum with a custom underlying type: `enum Severity : byte { Info = 0, Warning = 1, Error = 2, Critical = 3 }`. Repeat the technique from the lesson (`enum PowerMode : byte`).
5. Declare `[Flags] enum Permission { None = 0, Read = 1, Write = 2, Execute = 4, Delete = 8 }`. Powers of two and a `None = 0` member are mandatory — this is the best practice from the lesson.
6. Declare `record FileEntry(string Path, FileKind Kind, Permission Rights);` — stable semantics, value equality, `with`-expressions.
7. Write a method `(Permission Default, string Description) GetDefaultRights(FileKind kind)` that returns a tuple "default rights + description" for a category. For example, for `FileKind.Log` return `Permission.Read` and the description "logs are read-only". Implement the selection with a `switch` expression (pattern matching).
8. Write a method `(bool Allowed, string Reason) TryAccess(Permission current, Permission required)` that returns a tuple "allowed + reason". Inside, use `HasFlag` to check each bit of `required` against `current`. If at least one right is missing, return `false` and a string like `"missing: Write, Delete"` listing the missing rights.
9. Write a method `(Severity Severity, string Message) ClassifyEvent(int errorCode)` that returns a tuple "severity + message" for an error code. Use pattern matching over ranges: code 0 → `Info`, 1–99 → `Warning`, 100–199 → `Error`, 200+ → `Critical`. Confirm that `Severity` has an underlying type of `byte` by printing `(byte)severity`.
10. Write a method `(FileEntry Entry, Permission Granted) Grant(FileEntry entry, Permission extra)` that returns a tuple of the updated `FileEntry` (via `with { Rights = entry.Rights | extra }`) and the rights actually granted.
11. In the main program create several `FileEntry` values, call `GetDefaultRights`, `TryAccess`, `ClassifyEvent`, and `Grant`. Deconstruct the results with `var (allowed, reason) = TryAccess(...)` and print them.
12. Demonstrate `[Flags]`: create `Permission mine = Permission.Read | Permission.Write;`, print `mine.ToString()` (should be `Read, Write`), check `mine.HasFlag(Permission.Write)` and the bitwise check `(mine & Permission.Execute) == Permission.Execute`.
13. Demonstrate explicit casting: `int kindAsInt = (int)FileKind.Log;` — confirm that an implicit cast `int x = FileKind.Log;` does not compile, and comment that line out with an explanation.
14. Demonstrate tuple comparison: `var a = (Code: 1, Msg: "ok"); var b = (Code: 1, Msg: "ok"); Console.WriteLine(a == b);` — should be `True`.
15. Demonstrate the contrast with `record`: create two `FileEntry` values with the same fields, compare them with `==`, and create a copy with `with`.
16. (Bonus for going deeper, not required for submission) Show that a tuple field name is lost when boxed as `object`: `object boxed = (Code: 1, Msg: "x");` and try to access the field — explain in a comment why this is an anti-pattern.
17. Build and run the project: `dotnet build`, then `dotnet run`. Make sure the output matches the expected values.

Expected output (key lines):
```
mode default for Log = Read (logs are read-only)
mine = Read, Write
can write? True
can exec? False
access Read+Write to Read-only: False — missing: Write
severity for code 150 = Error (byte=2)
files equal by value? True
updated = FileEntry { Path = app.log, Kind = Log, Rights = Read, Write }
```

#### Requirements
The solution must compile under C# 12 / .NET 8 without error-level warnings and run via `dotnet run`. All enums are declared explicitly with readable names; for `[Flags]` powers of two are respected and `None = 0` is present. Tuples are used only where the lesson deems them appropriate: to return 2–4 values from a method within a single call; for the "file" entity a `record` is chosen and this choice is justified by a comment.

Casting an `enum` to a number must be explicit (`(int)`, `(byte)`); an implicit cast must be absent or commented out with the explanation "does not compile". Flag checks go through `HasFlag` or `(x & flag) == flag`, never through `==` for combined values. Tuples must not be boxed as `object` (except for the demonstration bonus with a comment marking it as an anti-pattern). Deconstruction `var (...) = Method(...)` is used in at least three places. The code uses top-level statements, contains bilingual comments (RU + EN) as in the lesson example. `Program.cs` must be self-contained — a single file with no external dependencies beyond the standard library.

#### Pitfalls
- **An implicit `enum` → `int` cast does not compile.** This is the first "common mistake" from the lesson. Write `(int)FileKind.Log`, not `int x = FileKind.Log;`. The `byte` underlying type likewise requires `(byte)severity`.
- **Powers of two in `[Flags]` are mandatory.** If you accidentally set `Read = 1, Write = 2, Execute = 3` (not a power of two), then `Execute` "covers" `Read` and `Write`, and `HasFlag` starts giving wrong results. Always use `1, 2, 4, 8, 16...` and always include `None = 0`.
- **Checking a flag with `==` loses combinations.** `mine == Permission.Write` yields `False` when `mine = Read | Write`. The correct forms are `mine.HasFlag(Permission.Write)` or `(mine & Permission.Write) == Permission.Write`. The lesson warns about this explicitly.
- **`ToString()` without `[Flags]` shows a number.** If you forget the attribute, `Read | Write` prints as `3` instead of `Read, Write`. The attribute is mandatory in this task.
- **A tuple field name is lost when boxed as `object`.** `(x as object).Code` does not compile, because the field type is only known statically. Pass tuples typed, never as `object`.
- **Tuple vs `record`.** A tuple is a "temporary wrapper" for 2–4 values within a call; a `record` is for stable semantics (`FileEntry`), value equality, and `with`. Do not use a tuple in place of a type.
- **The `byte` underlying type saves memory**, but the cast is still explicit. In `ClassifyEvent` return `Severity` and print `(byte)severity` to confirm the values 0–3 really fit in a `byte`.
- **`HasFlag` boxed its argument** in older versions; in .NET 8 this is optimized away, but for very hot loops prefer `(x & flag) == flag`. For a learning task the difference is negligible, but it is good to know.

#### Acceptance criteria
- [ ] The project is created with `dotnet new console` on .NET 8 and builds without errors.
- [ ] `enum FileKind` is declared with the default underlying type `int`.
- [ ] `enum Severity : byte` is declared with a custom underlying type and values 0–3.
- [ ] `[Flags] enum Permission` is declared with powers of two and `None = 0`.
- [ ] `GetDefaultRights` returns a tuple `(Permission, string)` and is implemented with a `switch` expression.
- [ ] `TryAccess` returns a tuple `(bool, string)` and uses `HasFlag` internally.
- [ ] `ClassifyEvent` returns a tuple `(Severity, string)` and uses pattern matching over ranges.
- [ ] `Grant` returns a tuple `(FileEntry, Permission)` and uses `with` to update the `record`.
- [ ] `record FileEntry` is declared with stable semantics; the choice of `record` is justified by a comment.
- [ ] The main program contains at least three tuple deconstructions via `var (...) = ...`.
- [ ] Explicit casts `(int)` and `(byte)` are demonstrated; the implicit cast is commented out with an explanation.
- [ ] A flag check is demonstrated both via `HasFlag` and via `&`.
- [ ] `Permission.Read | Permission.Write` prints as `Read, Write` (the `[Flags]` attribute is present).
- [ ] Tuple value equality is demonstrated (`a == b` → `True`).
- [ ] The contrast with `record` is demonstrated: equality and `with`.
- [ ] The code contains bilingual RU + EN comments, as in the lesson example.
- [ ] The output of `dotnet run` matches the expected key lines.

#### Hints (no direct answer)
- For a `switch` expression over `FileKind` use the syntax `kind switch { FileKind.Log => (Permission.Read, "logs read-only"), ... }`.
- To list missing rights, check each flag in turn, add the name to a `List<string>`, then `string.Join(", ", list)`.
- `HasFlag` returns a `bool` for a single flag; to find ALL missing ones, iterate over `Read, Write, Execute, Delete`.
- For range-based pattern matching in `ClassifyEvent` use `errorCode switch { 0 => ..., >= 1 and < 100 => ..., ... }`.
- To update rights in a `record`, apply `entry with { Rights = entry.Rights | extra }` — bitwise OR adds bits.
- If `int x = FileKind.Log;` does not compile — that is correct and expected; comment it out and explain why.

#### Reference solution walk-through
```csharp
// C# 12 / .NET 8 — AccessManager: enum + tuples + record
// Top-level statements

using System;
using System.Collections.Generic;

// --- enum: file categories (default underlying type int) ---
enum FileKind
{
    Config,      // 0
    Log,         // 1
    Data,        // 2
    Executable   // 3
}

// --- enum with a custom underlying type byte (saves memory) ---
enum Severity : byte
{
    Info     = 0,
    Warning  = 1,
    Error    = 2,
    Critical = 3
}

// --- [Flags]: bitwise OR combinations, powers of two + None=0 ---
[Flags]
enum Permission
{
    None    = 0,
    Read    = 0b0001, // 1
    Write   = 0b0010, // 2
    Execute = 0b0100, // 4
    Delete  = 0b1000  // 8
}

// --- record: stable semantics, value equality, with-expressions ---
record FileEntry(string Path, FileKind Kind, Permission Rights);

// --- Method returning a tuple (default rights + description) ---
(Permission Default, string Description) GetDefaultRights(FileKind kind) =>
    kind switch
    {
        FileKind.Config    => (Permission.Read | Permission.Write, "configs: read and write"),
        FileKind.Log       => (Permission.Read, "logs are read-only"),
        FileKind.Data      => (Permission.Read | Permission.Write, "data: read and write"),
        FileKind.Executable=> (Permission.Read | Permission.Execute, "executables: read and run"),
        _                  => (Permission.None, "unknown category")
    };

// --- Multiple-return method via tuple ---
(bool Allowed, string Reason) TryAccess(Permission current, Permission required)
{
    var missing = new List<string>();
    if (required.HasFlag(Permission.Read)    && !current.HasFlag(Permission.Read))    missing.Add("Read");
    if (required.HasFlag(Permission.Write)   && !current.HasFlag(Permission.Write))   missing.Add("Write");
    if (required.HasFlag(Permission.Execute) && !current.HasFlag(Permission.Execute)) missing.Add("Execute");
    if (required.HasFlag(Permission.Delete)  && !current.HasFlag(Permission.Delete))  missing.Add("Delete");
    return missing.Count == 0
        ? (true, "access granted")
        : (false, $"missing: {string.Join(", ", missing)}");
}

// --- Event classification: tuple (Severity, Message) + pattern matching ---
(Severity Severity, string Message) ClassifyEvent(int errorCode) =>
    errorCode switch
    {
        0                 => (Severity.Info, "all good"),
        >= 1 and < 100    => (Severity.Warning, "non-critical issue"),
        >= 100 and < 200  => (Severity.Error, "error"),
        _                 => (Severity.Critical, "critical failure")
    };

// --- Grant rights: tuple (updated FileEntry, actually granted rights) ---
(FileEntry Entry, Permission Granted) Grant(FileEntry entry, Permission extra) =>
    (entry with { Rights = entry.Rights | extra }, entry.Rights | extra);

// === Demo ===
FileKind kind = FileKind.Log;
var (defRights, defDesc) = GetDefaultRights(kind);
Console.WriteLine($"mode default for {kind} = {defRights} ({defDesc})");

// [Flags]: combine rights
Permission mine = Permission.Read | Permission.Write;
Console.WriteLine($"mine = {mine}");                                  // Read, Write
Console.WriteLine($"can write? {mine.HasFlag(Permission.Write)}");    // True
bool canExec = (mine & Permission.Execute) == Permission.Execute;
Console.WriteLine($"can exec? {canExec}");                            // False

// Access attempt: tuple deconstruction
FileEntry log = new("app.log", FileKind.Log, Permission.Read);
var (allowed, reason) = TryAccess(log.Rights, Permission.Read | Permission.Write);
Console.WriteLine($"access Read+Write to Read-only: {allowed} — {reason}");

// Event classification + custom byte underlying type
var (sev, msg) = ClassifyEvent(150);
Console.WriteLine($"severity for code 150 = {sev} (byte={(byte)sev}) — {msg}");

// Explicit enum-to-int cast; implicit cast does not compile
int kindAsInt = (int)FileKind.Log;
// int bad = FileKind.Log; // COMPILE ERROR: no implicit cast
Console.WriteLine($"FileKind.Log as int = {kindAsInt}");

// Tuple value equality
var a = (Code: 1, Msg: "ok");
var b = (Code: 1, Msg: "ok");
Console.WriteLine($"tuples equal by value? {a == b}");               // True

// Contrast with record: equality and with
FileEntry f1 = new("x", FileKind.Data, Permission.Read);
FileEntry f2 = new("x", FileKind.Data, Permission.Read);
Console.WriteLine($"files equal by value? {f1 == f2}");             // True
var (updated, granted) = Grant(log, Permission.Write);
Console.WriteLine($"updated = {updated}");                          // Rights = Read, Write
```

Line-by-line walk-through. The `enum FileKind` declaration mirrors the lesson's `PowerMode`: the default underlying type is `int`, values start at zero, and each name maps to a number. `enum Severity : byte` trains the lesson's technique of changing the underlying type (`enum PowerMode : byte`) — it saves memory, and in the output we confirm the value fits in a `byte` via `(byte)sev`. `[Flags] enum Permission` mirrors `AccessRights` from the lesson: powers of two are written as binary literals `0b0001...0b1000`, and `None = 0` is present — a mandatory best practice, otherwise combinations "collide" as the lesson warns.

`record FileEntry` is a direct echo of `record Point` from the lesson: stable semantics, value equality, `with`-expressions. Here the "tuple vs record" concept is applied: a `record` is chosen for the file because it has permanent business semantics and we want `with`. The `GetDefaultRights` method returns a tuple `(Permission, string)` — two values within a single call, just like `CreateUser` in the lesson; it is implemented with a `switch` expression (pattern matching, C# 12). The `TryAccess` method returns `(bool, string)` and uses `HasFlag` internally to check each bit — this is exactly the correct flag check the lesson contrasts with the erroneous `==` check. Collecting missing rights into a `List<string>` and then `string.Join`-ing them produces a readable denial reason.

`ClassifyEvent` returns `(Severity, string)` and uses pattern matching over ranges (`>= 1 and < 100`), reinforcing both the custom `byte` underlying type and the tuple as a carrier of multiple values. `Grant` returns `(FileEntry, Permission)` and applies `with { Rights = ... | extra }` — the bitwise OR adds bits, exactly like `AccessRights.Read | AccessRights.Write` in the lesson. The demo contains four deconstructions (`var (defRights, defDesc)`, `var (allowed, reason)`, `var (sev, msg)`, `var (updated, granted)`) — this is the lesson's `var (code, msg) = GetResult()` applied repeatedly. The explicit cast `(int)FileKind.Log` together with the commented-out attempt at an implicit `int bad = FileKind.Log;` reproduces the lesson's first "common mistake". The `a == b` comparison for tuples echoes `pointA == pointB` from the lesson and confirms that tuples compare by value. The contrast with `record` (`f1 == f2`, `Grant` via `with`) closes the topic of type selection.

#### Going deeper (bonus)
1. Add permission parsing from a string: a method `Permission ParsePermission(string s)` that accepts `"read,write"` and returns the combination via `Enum.Parse<Permission>(s, ignoreCase: true)`. Handle the case where a number `5` is passed — `Enum.Parse` should interpret it as `Read, Write`. Think about why this works precisely because of `[Flags]`.
2. Implement a method `(Permission Recommended, Permission Forbidden) Analyze(FileKind kind)` that returns two sets of rights in a single tuple. Consider whether this is the moment to introduce an `record AccessPolicy` instead of a tuple — justify the choice.
3. Demonstrate the lesson's anti-pattern: box a tuple as `object boxed = (Code: 1, Msg: "x");` and try to access `Code`. Explain in a comment why the field name is lost and why the lesson recommends passing tuples typed.
4. Measure (mentally) the difference between `HasFlag` and `(x & flag) == flag` in a hot loop `for (int i = 0; i < 1_000_000; i++)`. In .NET 8 `HasFlag` no longer boxes, but knowing this distinction is part of C# culture.

---

#### Чек-лист сдачи / Submission checklist
- [ ] Создан проект `AccessManager` под .NET 8 (`dotnet new console`).
- [ ] Объявлены `FileKind`, `Severity : byte`, `[Flags] Permission` со степенями двойки.
- [ ] Объявлен `record FileEntry`; выбор `record` обоснован комментарием.
- [ ] Реализованы методы `GetDefaultRights`, `TryAccess`, `ClassifyEvent`, `Grant` с возвратом кортежей.
- [ ] Минимум три деконструкции кортежа в главной программе.
- [ ] Демонстрируются `HasFlag`, `&`, явные `(int)`/`(byte)`, сравнение кортежей, контраст с `record`.
- [ ] Код содержит двуязычные комментарии RU + EN.
- [ ] `dotnet build` и `dotnet run` проходят без ошибок, вывод соответствует ожидаемому.
- [ ] Project `AccessManager` created on .NET 8 (`dotnet new console`).
- [ ] `FileKind`, `Severity : byte`, `[Flags] Permission` with powers of two declared.
- [ ] `record FileEntry` declared; the `record` choice is justified by a comment.
- [ ] `GetDefaultRights`, `TryAccess`, `ClassifyEvent`, `Grant` implemented returning tuples.
- [ ] At least three tuple deconstructions in the main program.
- [ ] `HasFlag`, `&`, explicit `(int)`/`(byte)`, tuple equality, and contrast with `record` demonstrated.
- [ ] Code contains bilingual RU + EN comments.
- [ ] `dotnet build` and `dotnet run` succeed without errors; output matches expectations.

#### Ресурсы / Resources
- Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/language-reference/builtin-types/enum — Типы перечислений (RU) / Enumeration types (EN)
- Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/language-reference/builtin-types/value-tuples — Кортежи значений (RU) / Value tuples (EN)
- Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/fundamentals/functional/pattern-matching — Сопоставление шаблонов (RU) / Pattern matching (EN)
- Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/language-reference/builtin-types/record — Записи (RU) / Records (EN)

---
[← Предыдущее ДЗ: M02-L07](homework-M02-L07-string-stringbuilder.md) | [⬆ К модулю M02](../README.md) | [Следующее ДЗ →](none)
