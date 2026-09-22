---
[← К уроку M07-L02](lesson-M07-L02-try-catch-finally.md) | [⬆ К модулю M07](../README.md) | [Следующее ДЗ →](homework-M07-L03-throw-rethrow.md)
---

### Домашнее задание M07-L02: try/catch/finally, порядок catch / Homework M07-L02: try/catch/finally, catch order

**Урок / Lesson:** M07-L02
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 3/5
**Цель / Goal:** (RU) Научиться грамотно структурировать блоки `try/catch/finally`, соблюдать порядок `catch` от конкретных типов к общим, корректно освобождать ресурсы через `finally`, применять фильтры исключений `when (...)` и осознанно использовать вложенные `try` для частичного восстановления. (EN) Learn to structure `try/catch/finally` blocks correctly, keep the `catch` order from specific to general, release resources through `finally`, apply `when (...)` exception filters, and use nested `try` intentionally for partial recovery.

#### Связь с уроком / Connection to the lesson
(RU) Урок M07-L02 вводит конструкцию `try/catch/finally` как основной механизм обработки исключений, объясняет правило «от конкретных к общим», демонстрирует вложенные `try` и фильтры `when (...)`, а также перечисляет частые ошибки: пустые `catch`, `throw ex;` вместо `throw;`, утечки ресурсов из-за отсутствия `finally`. ДЗ закрепляет эти темы на практической задаче загрузки и разбора конфигурационного файла с несколькими типами ошибок.
(EN) Lesson M07-L02 introduces `try/catch/finally` as the primary exception-handling mechanism, explains the «specific to general» rule, demonstrates nested `try` and `when (...)` filters, and lists common mistakes: empty `catch`, `throw ex;` instead of `throw;`, and resource leaks from missing `finally`. This homework reinforces those topics through a practical config-file loading and parsing task with multiple failure types.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация
Представьте, что вы пишете утилиту командной строки `configcheck` для DevOps-команды. Утилита читает конфигурационный файл `.ini`-подобного формата `key=value`, парсит каждую строку, преобразует значения в числа и булевы флаги и формирует отчёт о готовности окружения к деплою. Файл может отсутствовать, быть заблокированным другим процессом, содержать строки без знака равенства, иметь нечисловые значения там, где ожидаются числа, или флаги в неожиданном формате. Каждая из этих ситуаций — отдельный тип исключения или логическая ошибка, требующая своей стратегии восстановления. Глобальная ошибка (файл недоступен) должна прерывать работу, а локальная (одна кривая строка) — лишь пропустить запись и продолжить разбор остальных, накопив предупреждения. Так вы моделируете реалистичный сценарий: частичное восстановление через вложенные `try`, гарантированное освобождение файловых дескрипторов через `finally`, точечная обработка известных типов через упорядоченные `catch`, и фильтрация по содержимому исключения через `when (...)`. Эта задача типична для backend-сервисов, импортёров данных и конвейеров обработки логов, где пропуск одной записи дешевле, чем падение всего процесса.

#### Что нужно сделать (пошагово)
1. Создайте новый проект консольного приложения: `dotnet new console -n ConfigCheck -o ConfigCheck -f net8.0`, перейдите в папку: `cd ConfigCheck`. Убедитесь, что в `ConfigCheck.csproj` указаны `<LangVersion>latest</LangVersion>` и `<TargetFramework>net8.0</TargetFramework>`.
2. В файле `Program.cs` реализуйте топ-level программу с классом `ConfigLoader`, содержащим метод `public static LoadResult Load(string path)`. Тип `LoadResult` определите как `record` с полями `bool Success`, `IReadOnlyList<string> Warnings`, `IReadOnlyDictionary<string, string> Entries`.
3. Подготовьте тестовый файл `sample.cfg` в папке проекта со строками: `port=8080`, `retry=3`, `debug=true`, `broken-line`, `timeout=not-a-number`, `enabled=1`, `name=`, `=missing-key`, `retries=5`.
4. В методе `Load` откройте файл через `File.OpenRead` и оберните в `StreamReader`. Реализуйте ручное освобождение через `finally` (не `using`-объявление) — это намеренно, чтобы отработать шаблон «`finally` для не-`IDisposable`-подобных сценариев и для демонстрации гарантированной очистки».
5. Внешний `try` должен содержать `catch`-блоки в строгом порядке: сначала `FileNotFoundException` с фильтром `when (string.IsNullOrEmpty(ex.FileName) is false)`, затем `FileNotFoundException` без условия, затем `IOException`, затем `UnauthorizedAccessException`, и только в конце `Exception` как страховку. Каждый блок должен логировать через `Console.Error.WriteLine` и формировать `LoadResult` с `Success=false`.
6. Внутри цикла чтения строк организуйте вложенный `try` для разбора одной строки. Локальные `catch` для `FormatException` и `ArgumentException` должны добавлять сообщение в коллекцию `warnings` и продолжать цикл. Не используйте здесь `catch (Exception)` — пусть неизвестные ошибки всплывают во внешний `try`.
7. Для парсинга чисел применяйте `int.Parse` в `try`-блоке, чтобы выбрасывалось `FormatException` при нечисловых значениях. Для булевых флагов используйте `bool.Parse` (он бросает `FormatException` на значениях вроде `1`, поэтому оборачивайте логику в собственную проверку с `throw new FormatException(...)`).
8. Добавьте фильтр `when (...)` хотя бы в один из внешних `catch`-блоков: например, перехватывайте `IOException` только если сообщение содержит слово «lock», иначе позволяйте исключению всплывать.
9. Запустите программу с разными аргументами: существующий файл, несуществующий файл (`nonexistent.cfg`), файл без прав доступа (на Windows создайте файл и снимите read- permisии через `icacls sample.cfg /deny "%USERNAME%:R"` во временном каталоге). Зафиксируйте вывод для каждого случая.
10. В отчёте `LoadResult` количество предупреждений для `sample.cfg` должно быть равно числу проблемных строк (`broken-line`, `timeout=not-a-number`, `name=`, `=missing-key`). Проверьте это assertion-ом или выводом.
11. Соберите проект в Release: `dotnet build -c Release` и убедитесь, что нет предупреждений компилятора CS1057, CS0162 (недостижимый код) или CA1031 (перехват слишком общего исключения без логирования). При необходимости добавьте `[System.Diagnostics.CodeAnalysis.SuppressMessage]` с обоснованием.
12. Запустите `dotnet run -- sample.cfg` и `dotnet run -- nonexistent.cfg`, скопируйте вывод в файл `output.txt` и приложите к решению.

#### Требования к решению
- Код компилируется под .NET 8 с C# 12 без ошибок и без предупреждений уровня error.
- Используются top-level statements для точки входа; остальная логика в статическом классе.
- Порядок `catch` строго от конкретных к общим: `FileNotFoundException` → `IOException` → `UnauthorizedAccessException` → `Exception`. Нарушение порядка — ошибка компиляции CS1057, которую вы должны продемонстрировать как отрицательный пример (создайте ветку `bad-catch-order`, покажите ошибку, затем исправьте).
- Минимум один `when (...)`-фильтр применяется осмысленно (не пустое `when (true)`).
- `finally` присутствует и освобождает `StreamReader` и `FileStream` с проверкой `?.Dispose()`; ресурсные переменные объявлены до `try` и проинициализированы `null`.
- Вложенный `try` используется для локального разбора строки; локальные `catch` не «глатают» исключения молча — добавляют в `warnings`.
- Для повторной генерации (если потребуется) используется `throw;`, а не `throw ex;`.
- `LoadResult` — immutable `record`; коллекции возвращаются как `IReadOnlyList`/`IReadOnlyDictionary`.
- Программа читает путь к файлу из `args[0]` с проверкой `args.Length`; при отсутствии аргумента выводит usage и возвращает код `1` через `Environment.Exit(1)`.

#### Тонкости и подводные камни
- `catch (Exception)` первым приведёт к ошибке CS1057 и сделает специфичные обработчики недостижимыми — компилятор C# требует строгий порядок от конкретных к общим.
- `finally` выполняется даже при `return` внутри `try` и `catch`, но НЕ при `StackOverflowException`, `ExecutionEngineException` или принудительном `Environment.FailFast`/убийстве процесса — не полагайтесь на `finally` для критичной персистентности.
- `bool.Parse("1")` выбрасывает `FormatException` — это частая ловушка при миграции кода с других языков; вручную нормализуйте `1`/`0`/`yes`/`no` к `true`/`false`.
- Фильтр `when (...)` не разматывает стек, пока условие вычисляется — это позволяет логировать и продолжить всплытие, сохраняя исходный стек для отладки; используйте это для условного перехвата `IOException` по сообщению.
- `throw ex;` обнуляет `StackTrace` в точке повторной генерации — всегда `throw;` для сохранения стека. В этом ДЗ вы не должны повторно выбрасывать через `throw ex;`.
- Пустой `catch { }` или `catch (Exception) { }` маскирует баги — как минимум пишите в `Console.Error` или лог. Чекер качества CA1031 ругается на общий `catch (Exception)` без логирования.
- Переменная ресурса должна быть объявлена до `try`, иначе `finally` её «не видит» и возникает CS0165 (использование неприсвоенной переменной). Инициализируйте `null` и используйте `?.Dispose()`.
- Вложенные `try` без нужды усложняют код; здесь они оправданы частичным восстановлением — локальная ошибка одной строки не должна валить весь файл.
- `File.OpenRead` открывает файл в режиме `Share=Read`, поэтому попытка записи из другого процесса всё равно может дать `IOException` — учитывайте это при тестах с блокировкой.
- При чтении строк кодировка по умолчанию UTF-8; для файлов с BOM используйте `new StreamReader(stream, Encoding.UTF8, detectEncodingFromByteOrderMarks: true)`.

#### Критерии приёмки
- [ ] Проект `ConfigCheck` собирается под .NET 8 / C# 12 без ошибок и без warning-as-error.
- [ ] Порядок `catch` от конкретных к общим соблюдён; продемонстрирован отрицательный пример CS1057 в ветке `bad-catch-order`.
- [ ] `finally` присутствует, освобождает `StreamReader` и `FileStream`, выполняется при всех путях выполнения (включая `return`).
- [ ] Минимум один осмысленный `when (...)`-фильтр в `catch`.
- [ ] Вложенный `try` обрабатывает локальные `FormatException`/`ArgumentException` и накапливает `warnings`.
- [ ] Нет пустых `catch { }`; каждый `catch` либо логирует, либо формирует результат.
- [ ] `bool.Parse` обёрнут нормализацией `1`/`0`/`yes`/`no`/`on`/`off`.
- [ ] `LoadResult` — `record` с immutable коллекциями.
- [ ] Запуск с `sample.cfg` даёт ровно 4 предупреждения и `Success=true`.
- [ ] Запуск с `nonexistent.cfg` даёт `Success=false` и сообщение о ненайденном файле.
- [ ] Запуск без аргументов выводит usage и возвращает код `1`.
- [ ] Нет `throw ex;` — только `throw;` (если повторная генерация вообще используется).
- [ ] Вывод для всех сценариев сохранён в `output.txt`.
- [ ] Код использует top-level statements и C# 12 (можно `collection expressions`, `required`, `init`).
- [ ] README или комментарий в `Program.cs` объясняет выбор порядка `catch` и место `finally`.

#### Подсказки (без прямого ответа)
- Вспомните иерархию: `FileNotFoundException` → `IOException` → `Exception`. Где должен стоять `UnauthorizedAccessException`? Он наследуется от `SystemException`, не от `IOException`.
- Для сбора предупреждений используйте `List<string>` внутри метода и возвращайте как `IReadOnlyList<string>` через `.AsReadOnly()` или `List.AsReadOnly()`.
- Фильтр `when` может вызывать вспомогательный метод — например, `when (IsTransientLock(ex))`. Это улучшает читаемость и тестируемость.
- Чтобы продемонстрировать CS1057, поставьте `catch (Exception ex)` перед `catch (FileNotFoundException)` — компилятор сразу укажет на недостижимый код.
- `args` в top-level программе доступен напрямую; проверяйте `args.Length == 0` до обращения к `args[0]`.
- Для возврата кода выхода используйте `Environment.ExitCode` или `Environment.Exit(1)` — `return` из top-level не задаёт код процесса.

#### Эталонное решение (разбор)
```csharp
// C# 12 / .NET 8 — Эталон ДЗ M07-L02
// Демонстрирует: порядок catch, finally, вложенный try, фильтр when, LoadResult record

using System.Collections.Generic;
using System.IO;

namespace ConfigCheck;

// Immutable результат загрузки / Immutable load result
public sealed record LoadResult(
    bool Success,
    IReadOnlyList<string> Warnings,
    IReadOnlyDictionary<string, string> Entries);

internal static class ConfigLoader
{
    // Ресурсные переменные объявлены ДО try, чтобы finally их видел
    // Resource variables declared BEFORE try so finally can see them
    public static LoadResult Load(string path)
    {
        FileStream? stream = null;
        StreamReader? reader = null;
        var warnings = new List<string>();
        var entries = new Dictionary<string, string>();

        try
        {
            stream = File.OpenRead(path);
            reader = new StreamReader(stream);

            string? line;
            int lineNo = 0;
            while ((line = reader.ReadLine()) is not null)
            {
                lineNo++;

                // Вложенный try: локальная ошибка одной строки не валит весь файл
                // Nested try: a single line's local error does not crash the whole file
                try
                {
                    var parts = line.Split('=', 2);
                    if (parts.Length != 2 || string.IsNullOrWhiteSpace(parts[0]))
                        throw new FormatException($"Строка {lineNo} не вида key=value / Line {lineNo} is not key=value");

                    var (key, value) = (parts[0].Trim(), parts[1].Trim());

                    // bool-нормализация, так как bool.Parse("1") бросает FormatException
                    // bool normalization because bool.Parse("1") throws FormatException
                    if (key.StartsWith("debug") || key.StartsWith("enabled"))
                    {
                        var norm = value.ToLowerInvariant() switch
                        {
                            "1" or "yes" or "on" or "true" => "true",
                            "0" or "no" or "off" or "false" => "false",
                            _ => throw new FormatException($"Некорректный флаг / Bad flag: {value}")
                        };
                        entries[key] = norm;
                    }
                    else if (key.StartsWith("port") || key.StartsWith("retry") || key.StartsWith("timeout"))
                    {
                        // int.Parse выбрасывает FormatException при нечисловом значении
                        // int.Parse throws FormatException on non-numeric value
                        _ = int.Parse(value);
                        entries[key] = value;
                    }
                    else
                    {
                        entries[key] = value;
                    }
                }
                catch (FormatException ex)
                {
                    warnings.Add($"Строка {lineNo}: {ex.Message}");
                }
                catch (ArgumentException ex)
                {
                    warnings.Add($"Строка {lineNo}: аргумент / argument: {ex.Message}");
                }
            }

            return new LoadResult(true, warnings.AsReadOnly(), entries);
        }
        // Порядок catch: от конкретных к общим / Catch order: specific to general
        catch (FileNotFoundException ex) when (string.IsNullOrEmpty(ex.FileName) is false)
        {
            Console.Error.WriteLine($"Файл не найден / File not found: {ex.FileName}");
            return new LoadResult(false, warnings, entries);
        }
        catch (FileNotFoundException)
        {
            Console.Error.WriteLine("Файл не найден (имя неизвестно) / File not found (name unknown)");
            return new LoadResult(false, warnings, entries);
        }
        catch (UnauthorizedAccessException ex)
        {
            Console.Error.WriteLine($"Нет доступа / Access denied: {ex.Message}");
            return new LoadResult(false, warnings, entries);
        }
        // Фильтр по содержимому: перехватываем только «lock»-ошибки, остальное всплывает
        // Content filter: catch only "lock" errors, the rest propagates
        catch (IOException ex) when (ex.Message.Contains("lock", StringComparison.OrdinalIgnoreCase))
        {
            Console.Error.WriteLine($"Файл заблокирован / File locked: {ex.Message}");
            return new LoadResult(false, warnings, entries);
        }
        catch (IOException ex)
        {
            Console.Error.WriteLine($"Ошибка ввода-вывода / I/O error: {ex.Message}");
            return new LoadResult(false, warnings, entries);
        }
        catch (Exception ex)
        {
            // Универсальная страховка — всегда последняя / Universal safety net — always last
            Console.Error.WriteLine($"Непредвиденная ошибка / Unexpected error: {ex.Message}");
            return new LoadResult(false, warnings, entries);
        }
        finally
        {
            // finally выполняется ВСЕГДА: успех, исключение, return
            // finally runs ALWAYS: success, exception, return
            reader?.Dispose();
            stream?.Dispose();
            Console.Error.WriteLine("Ресурсы освобождены / Resources disposed");
        }
    }
}

// Точка входа top-level / Top-level entry point
if (args.Length == 0)
{
    Console.Error.WriteLine("Использование / Usage: configcheck <path-to-cfg>");
    Environment.Exit(1);
}

var result = ConfigLoader.Load(args[0]);
Console.WriteLine($"Success={result.Success}, Warnings={result.Warnings.Count}, Entries={result.Entries.Count}");
foreach (var w in result.Warnings)
    Console.WriteLine($"WARN: {w}");
foreach (var (k, v) in result.Entries)
    Console.WriteLine($"{k} = {v}");
```

Разбор по строкам. `FileStream? stream` и `StreamReader? reader` объявлены до `try` и равны `null` — это необходимо, чтобы блок `finally` гарантированно видел переменные даже при исключении в `File.OpenRead`: если бы они были объявлены внутри `try`, компилятор выдал бы CS0165. Конструкция `?.Dispose()` в `finally` безопасна для `null` и закрывает ресурсы в обратном порядке (сначала reader, затем stream). Внешний `try` открывает файл и читает строки; `File.OpenRead` может выбросить `FileNotFoundException`, `IOException`, `UnauthorizedAccessException` или их подтипы. Порядок `catch` строго подчинён правилу «от конкретных к общим»: сначала два `FileNotFoundException` (с фильтром на `FileName` и без), затем `UnauthorizedAccessException`, затем два `IOException` (с фильтром `when` по слову «lock» и без), и только в самом конце `Exception`. Этот порядок диктуется иерархией типов: `FileNotFoundException` наследуется от `IOException`, поэтому если бы `IOException` стоял раньше, `FileNotFoundException` стал бы недостижимым (CS1057). Фильтр `when (ex.Message.Contains("lock", ...))` не разматывает стек, пока вычисляется, поэтому несовпадающие `IOException` продолжают всплывать с сохранённым стеком — это ценное свойство для отладки. Вложенный `try` внутри цикла отвечает за локальный разбор строки: `FormatException` от `int.Parse` и от нашей ручной проверки `key=value` перехватываются локально, запись добавляется в `warnings`, и цикл продолжается. Так реализуется частичное восстановление — одна кривая строка не валит весь файл. Локальные `catch` не используют `catch (Exception)`, чтобы неизвестные ошибки всплывали во внешний обработчик, где их поймает страховка. `bool.Parse` здесь не вызывается напрямую: вместо этого `switch`-выражение нормализует `1`/`0`/`yes`/`no` к `true`/`false`, избегая типичной ловушки `bool.Parse("1") → FormatException`. `LoadResult` — immutable `record` с `IReadOnlyList` и `IReadOnlyDictionary`, что гарантирует неизменяемость результата. В точке входа используется проверка `args.Length == 0` и `Environment.Exit(1)` — `return` из top-level не задаёт код процесса. `AsReadOnly()` оборачивает `List<string>` в `ReadOnlyCollection`, предотвращая дальнейшую мутацию. Наконец, в `finally` есть диагностический вывод «Ресурсы освобождены» — он должен появиться во всех сценариях запуска, что подтверждает безусловность `finally`.

#### Задания на углубление (бонус)
1. Замените ручной `finally` на `using`-объявление C# 8+ и сравните читаемость; объясните, почему в уроке говорится о предпочтительности `using`, но `finally` остаётся для не-`IDisposable`-ресурсов.
2. Добавьте таймер производительности и сравните время разбора файла на 100 000 строк с локальным `try/catch` на каждую строку против одного `try` с накоплением ошибок в коллекцию и одиночной проверкой после цикла.
3. Реализуйте перегрузку `LoadAsync(string path, CancellationToken ct)` с `StreamReader.ReadLineAsync` и корректной обработкой `OperationCanceledException` в `finally` — обратите внимание, что `ct` должен проверяться в `when`-фильтре.
4. Добавьте юнит-тесты на xUnit, проверяющие каждый путь `catch` (несуществующий файл, нет доступа, заблокированный файл, кривая строка), и убедитесь, что `finally` вызывается всегда — например, через мок `IDisposable` со счётчиком вызовов `Dispose`.

---

## Statement in English / Постановка на английском

#### Context & motivation
Imagine you are writing a command-line utility `configcheck` for a DevOps team. The utility reads an `.ini`-style configuration file in `key=value` form, parses every line, converts values into integers and boolean flags, and produces a readiness report for the deployment environment. The file may be missing, locked by another process, contain lines without an equals sign, hold non-numeric values where numbers are expected, or carry flags in an unexpected format. Each of these situations is a distinct exception type or a logical error that demands its own recovery strategy. A global failure (the file is unavailable) must abort the run, while a local one (a single malformed line) should merely skip the record and continue parsing the rest, accumulating warnings. This is a realistic scenario that models partial recovery through nested `try`, guaranteed release of file handles through `finally`, precise handling of known types through ordered `catch`, and content-based filtering through `when (...)`. The task is typical for backend services, data importers, and log-processing pipelines, where skipping one record is cheaper than crashing the whole process.

#### What to do step by step
1. Create a new console application: `dotnet new console -n ConfigCheck -o ConfigCheck -f net8.0`, then `cd ConfigCheck`. Ensure `ConfigCheck.csproj` contains `<LangVersion>latest</LangVersion>` and `<TargetFramework>net8.0</TargetFramework>`.
2. In `Program.cs` implement a top-level program with a `ConfigLoader` class that exposes `public static LoadResult Load(string path)`. Define `LoadResult` as a `record` with fields `bool Success`, `IReadOnlyList<string> Warnings`, `IReadOnlyDictionary<string, string> Entries`.
3. Prepare a test file `sample.cfg` in the project folder with the following lines: `port=8080`, `retry=3`, `debug=true`, `broken-line`, `timeout=not-a-number`, `enabled=1`, `name=`, `=missing-key`, `retries=5`.
4. In `Load`, open the file with `File.OpenRead` and wrap it in a `StreamReader`. Implement manual release through `finally` (not a `using` declaration) — this is deliberate, to practise the «`finally` for non-`IDisposable`-like scenarios and guaranteed cleanup» pattern.
5. The outer `try` must contain `catch` blocks in strict order: first `FileNotFoundException` with the filter `when (string.IsNullOrEmpty(ex.FileName) is false)`, then `FileNotFoundException` without a condition, then `IOException`, then `UnauthorizedAccessException`, and only at the end `Exception` as a safety net. Every block must log through `Console.Error.WriteLine` and produce a `LoadResult` with `Success=false`.
6. Inside the line-reading loop, organise a nested `try` for parsing a single line. Local `catch` blocks for `FormatException` and `ArgumentException` must append a message to the `warnings` collection and continue the loop. Do not use `catch (Exception)` here — let unknown errors bubble up to the outer `try`.
7. For number parsing use `int.Parse` inside a `try` so that a non-numeric value produces a `FormatException`. For boolean flags use `bool.Parse` (it throws `FormatException` on values like `1`), so wrap the logic in your own normalisation check that throws a new `FormatException(...)` instead.
8. Add a `when (...)` filter to at least one of the outer `catch` blocks: for example, catch `IOException` only when the message contains the word «lock», otherwise let the exception propagate.
9. Run the program with different arguments: an existing file, a non-existing file (`nonexistent.cfg`), and a file with no read permissions (on Windows, create a file and deny read access with `icacls sample.cfg /deny "%USERNAME%:R"` in a temporary folder). Record the output for each case.
10. The `LoadResult` report for `sample.cfg` must contain a number of warnings equal to the number of problematic lines (`broken-line`, `timeout=not-a-number`, `name=`, `=missing-key`). Verify it with an assertion or by printing.
11. Build the project in Release: `dotnet build -c Release` and make sure there are no CS1057, CS0162 (unreachable code), or CA1031 (catching a too-general exception without logging) warnings. If needed, add a `[System.Diagnostics.CodeAnalysis.SuppressMessage]` with a justification.
12. Run `dotnet run -- sample.cfg` and `dotnet run -- nonexistent.cfg`, copy the output into a file `output.txt`, and attach it to your solution.

#### Requirements
- The code compiles under .NET 8 with C# 12 without errors and without warning-as-error.
- Top-level statements are used for the entry point; the rest of the logic lives in a static class.
- The `catch` order is strictly specific-to-general: `FileNotFoundException` → `IOException` → `UnauthorizedAccessException` → `Exception`. A violation is a CS1057 compile error, which you must demonstrate as a negative example (create a branch `bad-catch-order`, show the error, then fix it).
- At least one `when (...)` filter is applied meaningfully (not an empty `when (true)`).
- `finally` is present and disposes of `StreamReader` and `FileStream` with `?.Dispose()`; the resource variables are declared before `try` and initialised to `null`.
- A nested `try` is used for local line parsing; local `catch` blocks do not silently swallow exceptions — they append to `warnings`.
- If rethrowing is needed, use `throw;`, never `throw ex;`.
- `LoadResult` is an immutable `record`; collections are returned as `IReadOnlyList`/`IReadOnlyDictionary`.
- The program reads the file path from `args[0]` with an `args.Length` check; if the argument is missing, it prints usage and returns exit code `1` via `Environment.Exit(1)`.

#### Pitfalls
- Placing `catch (Exception)` first triggers CS1057 and makes specific handlers unreachable — the C# compiler demands strict specific-to-general ordering.
- `finally` runs even on `return` inside `try` or `catch`, but NOT on `StackOverflowException`, `ExecutionEngineException`, or forced `Environment.FailFast`/process kill — do not rely on `finally` for critical persistence.
- `bool.Parse("1")` throws `FormatException` — a common trap when porting code from other languages; manually normalise `1`/`0`/`yes`/`no` to `true`/`false`.
- The `when (...)` filter does not unwind the stack while the condition is evaluated — this lets you log and continue propagation, preserving the original stack for debugging; use it for conditional interception of `IOException` by message.
- `throw ex;` resets the `StackTrace` at the rethrow point — always use `throw;` to preserve the stack. In this homework you must not rethrow via `throw ex;`.
- An empty `catch { }` or `catch (Exception) { }` masks bugs — at minimum write to `Console.Error` or a log. The CA1031 analyser flags a general `catch (Exception)` without logging.
- The resource variable must be declared before `try`, otherwise `finally` cannot see it and CS0165 (use of an unassigned variable) occurs. Initialise it to `null` and use `?.Dispose()`.
- Unnecessary nested `try` complicates code; here it is justified by partial recovery — a single line’s local error must not crash the whole file.
- `File.OpenRead` opens the file with `Share=Read`, so a write attempt from another process may still raise `IOException` — keep this in mind when testing file locking.
- The default encoding is UTF-8; for files with a BOM use `new StreamReader(stream, Encoding.UTF8, detectEncodingFromByteOrderMarks: true)`.

#### Acceptance criteria
- [ ] The `ConfigCheck` project builds under .NET 8 / C# 12 without errors and without warning-as-error.
- [ ] The `catch` order is specific-to-general; a negative CS1057 example is shown in the `bad-catch-order` branch.
- [ ] `finally` is present, disposes of `StreamReader` and `FileStream`, and runs on every execution path (including `return`).
- [ ] At least one meaningful `when (...)` filter is used in a `catch`.
- [ ] A nested `try` handles local `FormatException`/`ArgumentException` and accumulates `warnings`.
- [ ] There are no empty `catch { }` blocks; every `catch` either logs or produces a result.
- [ ] `bool.Parse` is wrapped with `1`/`0`/`yes`/`no`/`on`/`off` normalisation.
- [ ] `LoadResult` is a `record` with immutable collections.
- [ ] A run with `sample.cfg` yields exactly 4 warnings and `Success=true`.
- [ ] A run with `nonexistent.cfg` yields `Success=false` and a file-not-found message.
- [ ] A run with no arguments prints usage and returns exit code `1`.
- [ ] No `throw ex;` — only `throw;` (if rethrowing is used at all).
- [ ] The output for all scenarios is saved in `output.txt`.
- [ ] The code uses top-level statements and C# 12 features (`collection expressions`, `required`, `init` where applicable).
- [ ] A README or a comment in `Program.cs` explains the choice of `catch` order and the role of `finally`.

#### Hints (without the direct answer)
- Recall the hierarchy: `FileNotFoundException` → `IOException` → `Exception`. Where should `UnauthorizedAccessException` sit? It derives from `SystemException`, not from `IOException`.
- For warnings, use a `List<string>` inside the method and return it as `IReadOnlyList<string>` via `.AsReadOnly()` or `List.AsReadOnly()`.
- The `when` filter may call a helper method — e.g., `when (IsTransientLock(ex))`. This improves readability and testability.
- To demonstrate CS1057, put `catch (Exception ex)` before `catch (FileNotFoundException)` — the compiler immediately flags unreachable code.
- In a top-level program `args` is available directly; check `args.Length == 0` before touching `args[0]`.
- To set the exit code use `Environment.ExitCode` or `Environment.Exit(1)` — a `return` from top-level does not set the process exit code.

#### Reference solution walk-through
```csharp
// C# 12 / .NET 8 — Reference solution for Homework M07-L02
// Demonstrates: catch order, finally, nested try, when filter, LoadResult record

using System.Collections.Generic;
using System.IO;

namespace ConfigCheck;

// Immutable load result
public sealed record LoadResult(
    bool Success,
    IReadOnlyList<string> Warnings,
    IReadOnlyDictionary<string, string> Entries);

internal static class ConfigLoader
{
    // Resource variables declared BEFORE try so finally can see them
    public static LoadResult Load(string path)
    {
        FileStream? stream = null;
        StreamReader? reader = null;
        var warnings = new List<string>();
        var entries = new Dictionary<string, string>();

        try
        {
            stream = File.OpenRead(path);
            reader = new StreamReader(stream);

            string? line;
            int lineNo = 0;
            while ((line = reader.ReadLine()) is not null)
            {
                lineNo++;

                // Nested try: a single line's local error does not crash the whole file
                try
                {
                    var parts = line.Split('=', 2);
                    if (parts.Length != 2 || string.IsNullOrWhiteSpace(parts[0]))
                        throw new FormatException($"Line {lineNo} is not in key=value form");

                    var (key, value) = (parts[0].Trim(), parts[1].Trim());

                    // bool normalization because bool.Parse("1") throws FormatException
                    if (key.StartsWith("debug") || key.StartsWith("enabled"))
                    {
                        var norm = value.ToLowerInvariant() switch
                        {
                            "1" or "yes" or "on" or "true" => "true",
                            "0" or "no" or "off" or "false" => "false",
                            _ => throw new FormatException($"Bad flag value: {value}")
                        };
                        entries[key] = norm;
                    }
                    else if (key.StartsWith("port") || key.StartsWith("retry") || key.StartsWith("timeout"))
                    {
                        // int.Parse throws FormatException on non-numeric value
                        _ = int.Parse(value);
                        entries[key] = value;
                    }
                    else
                    {
                        entries[key] = value;
                    }
                }
                catch (FormatException ex)
                {
                    warnings.Add($"Line {lineNo}: {ex.Message}");
                }
                catch (ArgumentException ex)
                {
                    warnings.Add($"Line {lineNo}: argument: {ex.Message}");
                }
            }

            return new LoadResult(true, warnings.AsReadOnly(), entries);
        }
        // Catch order: specific to general
        catch (FileNotFoundException ex) when (string.IsNullOrEmpty(ex.FileName) is false)
        {
            Console.Error.WriteLine($"File not found: {ex.FileName}");
            return new LoadResult(false, warnings, entries);
        }
        catch (FileNotFoundException)
        {
            Console.Error.WriteLine("File not found (name unknown)");
            return new LoadResult(false, warnings, entries);
        }
        catch (UnauthorizedAccessException ex)
        {
            Console.Error.WriteLine($"Access denied: {ex.Message}");
            return new LoadResult(false, warnings, entries);
        }
        // Content filter: catch only "lock" errors, the rest propagates
        catch (IOException ex) when (ex.Message.Contains("lock", StringComparison.OrdinalIgnoreCase))
        {
            Console.Error.WriteLine($"File locked: {ex.Message}");
            return new LoadResult(false, warnings, entries);
        }
        catch (IOException ex)
        {
            Console.Error.WriteLine($"I/O error: {ex.Message}");
            return new LoadResult(false, warnings, entries);
        }
        catch (Exception ex)
        {
            // Universal safety net — always last
            Console.Error.WriteLine($"Unexpected error: {ex.Message}");
            return new LoadResult(false, warnings, entries);
        }
        finally
        {
            // finally runs ALWAYS: success, exception, return
            reader?.Dispose();
            stream?.Dispose();
            Console.Error.WriteLine("Resources disposed");
        }
    }
}

// Top-level entry point
if (args.Length == 0)
{
    Console.Error.WriteLine("Usage: configcheck <path-to-cfg>");
    Environment.Exit(1);
}

var result = ConfigLoader.Load(args[0]);
Console.WriteLine($"Success={result.Success}, Warnings={result.Warnings.Count}, Entries={result.Entries.Count}");
foreach (var w in result.Warnings)
    Console.WriteLine($"WARN: {w}");
foreach (var (k, v) in result.Entries)
    Console.WriteLine($"{k} = {v}");
```

Line-by-line walk-through. `FileStream? stream` and `StreamReader? reader` are declared before `try` and set to `null` — this is mandatory so the `finally` block can reliably see the variables even if `File.OpenRead` throws: had they been declared inside `try`, the compiler would emit CS0165. The `?.Dispose()` calls in `finally` are null-safe and close the resources in reverse order (reader first, then stream). The outer `try` opens the file and reads lines; `File.OpenRead` may throw `FileNotFoundException`, `IOException`, `UnauthorizedAccessException`, or their subtypes. The `catch` order strictly follows the «specific to general» rule: first two `FileNotFoundException` handlers (with a `FileName` filter and without), then `UnauthorizedAccessException`, then two `IOException` handlers (with a `when` filter on the word «lock» and without), and only at the very end `Exception`. This order is dictated by the type hierarchy: `FileNotFoundException` derives from `IOException`, so had `IOException` come first, `FileNotFoundException` would become unreachable (CS1057). The `when (ex.Message.Contains("lock", ...))` filter does not unwind the stack while it evaluates, so non-matching `IOException` instances keep propagating with the original stack intact — a valuable property for debugging. The nested `try` inside the loop handles local line parsing: `FormatException` from `int.Parse` and from our manual `key=value` check is caught locally, a record is appended to `warnings`, and the loop continues. This is partial recovery in action — a single malformed line does not crash the whole file. The local `catch` blocks deliberately avoid `catch (Exception)` so that unknown errors bubble up to the outer handler, where the safety net catches them. `bool.Parse` is not called directly: instead, a `switch` expression normalises `1`/`0`/`yes`/`no` to `true`/`false`, sidestepping the classic `bool.Parse("1") → FormatException` trap. `LoadResult` is an immutable `record` with `IReadOnlyList` and `IReadOnlyDictionary`, which guarantees result immutability. At the entry point an `args.Length == 0` check and `Environment.Exit(1)` are used — a `return` from top-level does not set the process exit code. `AsReadOnly()` wraps the `List<string>` into a `ReadOnlyCollection`, preventing further mutation. Finally, the diagnostic «Resources disposed» line in `finally` should appear in every run scenario, confirming the unconditional nature of `finally`.

#### Going deeper (bonus)
1. Replace the manual `finally` with a C# 8+ `using` declaration and compare readability; explain why the lesson prefers `using` yet keeps `finally` for non-`IDisposable` resources.
2. Add a performance timer and compare parsing of a 100 000-line file with a local `try/catch` per line against a single `try` that accumulates errors in a collection and validates after the loop.
3. Implement an `LoadAsync(string path, CancellationToken ct)` overload with `StreamReader.ReadLineAsync` and proper `OperationCanceledException` handling in `finally` — note that `ct` should be checked inside a `when` filter.
4. Add xUnit unit tests covering every `catch` path (missing file, no access, locked file, malformed line) and verify that `finally` is always invoked — for instance, via an `IDisposable` mock with a `Dispose` call counter.

---

#### Чек-лист сдачи / Submission checklist
- [ ] Проект `ConfigCheck` собирается под .NET 8 / C# 12.
- [ ] Порядок `catch` от конкретных к общим; показан отрицательный пример CS1057.
- [ ] `finally` освобождает `StreamReader` и `FileStream` на всех путях.
- [ ] Минимум один осмысленный `when (...)`-фильтр.
- [ ] Вложенный `try` накапливает `warnings`.
- [ ] Нет пустых `catch { }`.
- [ ] `bool.Parse` обёрнут нормализацией флагов.
- [ ] `LoadResult` — immutable `record`.
- [ ] `sample.cfg` → 4 предупреждения, `Success=true`.
- [ ] `nonexistent.cfg` → `Success=false`.
- [ ] Нет аргументов → usage и код `1`.
- [ ] Нет `throw ex;`.
- [ ] `output.txt` приложен.
- [ ] README/комментарий объясняет порядок `catch` и `finally`.
- [ ] Project `ConfigCheck` builds under .NET 8 / C# 12.
- [ ] `catch` order is specific-to-general; CS1057 negative example shown.
- [ ] `finally` disposes `StreamReader` and `FileStream` on every path.
- [ ] At least one meaningful `when (...)` filter.
- [ ] Nested `try` accumulates `warnings`.
- [ ] No empty `catch { }`.
- [ ] `bool.Parse` is wrapped with flag normalisation.
- [ ] `LoadResult` is an immutable `record`.
- [ ] `sample.cfg` → 4 warnings, `Success=true`.
- [ ] `nonexistent.cfg` → `Success=false`.
- [ ] No arguments → usage and exit code `1`.
- [ ] No `throw ex;`.
- [ ] `output.txt` attached.
- [ ] README/comment explains `catch` order and `finally`.

#### Ресурсы / Resources
- [Microsoft Learn — try-catch-finally](https://learn.microsoft.com/dotnet/csharp/language-reference/keywords/try-catch-finally)
- [Microsoft Learn — Exception filters (`when`)](https://learn.microsoft.com/dotnet/csharp/language-reference/keywords/when)
- [Microsoft Learn — `using` statement and declarations](https://learn.microsoft.com/dotnet/csharp/language-reference/statements/using)
- [Microsoft Learn — Exceptions and exception handling](https://learn.microsoft.com/dotnet/csharp/fundamentals/exceptions)
- [.NET GitHub — runtime exception hierarchy](https://github.com/dotnet/runtime)
