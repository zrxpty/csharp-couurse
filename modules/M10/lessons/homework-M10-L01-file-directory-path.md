---
[← К уроку M10-L01](lesson-M10-L01-file-directory-path.md) | [⬆ К модулю M10](../README.md) | [Следующее ДЗ →](homework-M10-L02-stream-filestream.md)
---

### Домашнее задание M10-L01: File/FileInfo/Directory/Path / Homework M10-L01: File/FileInfo/Directory/Path

**Урок / Lesson:** M10-L01
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 3/5
**Цель / Goal:** (RU) Научиться выбирать между статическими (`File`/`Directory`) и инстансными (`FileInfo`/`DirectoryInfo`) классами под конкретную задачу, безопасно склеивать кроссплатформенные пути через `Path`, корректно работать с временными файлами и писать асинхронный, конкурентно-безопасный файловый I/O в .NET 8. (EN) Learn to choose between static (`File`/`Directory`) and instance (`FileInfo`/`DirectoryInfo`) classes for a given task, safely compose cross-platform paths with `Path`, handle temporary files correctly, and write asynchronous, concurrency-safe file I/O in .NET 8.

#### Связь с уроком / Connection to the lesson
(RU) Урок вводит четыре ключевых типа `System.IO` и объясняет, когда статический класс эффективнее инстансного, почему `Path` не трогает диск, в чём опасность хардкода `\\` и лимита `GetTempFileName()`, а также как `FileShare.Read` влияет на конкурентный доступ. Это ДЗ закрепляет каждую из этих тем на реальном CLI-инструменте.
(EN) The lesson introduces the four core `System.IO` types and explains when a static class beats an instance one, why `Path` never touches the disk, the danger of hardcoding `\\` and of the `GetTempFileName()` limit, and how `FileShare.Read` affects concurrent access. This homework cements every one of those topics on a real CLI tool.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация

Представьте, что вы junior-разработчик в команде, которая поддерживает внутренний инструмент для резервного копирования текстовых логов. Каждый вечер сервис складывает `.txt`-файлы в рабочую папку, и вашему менеджеру нужен простой, но надёжный CLI-утилита `dir-snapshot`, которая по команде делает «снимок» указанной директории: перечисляет файлы, считает суммарный размер, копирует все текстовые файлы в резервную подпапку и формирует человекочитаемый отчёт. Инструмент должен работать одинаково на Windows-сервере и в Linux-контейнере, потому что продакшен мигрирует на Linux. Это значит, что любая хардкод-привязка к обратному слешу моментально сломает запуск в Docker.

Именно здесь раскрываются темы урока M10-L01. Разовая проверка существования файла — это работа для статического `File.Exists`. А вот перечисление содержимого директории с чтением размера, времени создания и атрибутов каждого файла — это классический сценарий для `DirectoryInfo.GetFiles()`, возвращающего массив `FileInfo[]`, где каждый объект уже кэширует метаданные. Склейку путей мы обязаны делать через `Path.Combine`, а не через `+` и `"\"`, иначе кроссплатформенность потеряна. Отчёт нужно писать асинхронно, чтобы не блокировать поток, а временное имя для промежуточного файла — генерировать через `Guid`, а не через `Path.GetTempFileName()`, потому что у последнего на Windows есть неочевидный лимит имён, о котором прямо предупреждает урок.

Наконец, утилита должна логировать свои действия в общий лог-файл, к которому одновременно могут обращаться несколько запущенных экземпляров. Это подводит нас к concurrency-аспекту: файловые операции не атомарны, и параллельная запись без синхронизации приведёт к перемешанным строкам. Вы реализуете простую блокировку через `SemaphoreSlim`, что напрямую отрабатывает материал урока про синхронизацию писателей.

#### Что нужно сделать (пошагово)

1. **Создайте проект.** В папке `M10/homework/dir-snapshot` выполните `dotnet new console -n DirSnapshot -o DirSnapshot --framework net8.0`. Откройте `DirSnapshot.csproj` и убедитесь, что `LangVersion` соответствует C# 12 (можно добавить `<PropertyGroup><LangVersion>latest</LangVersion></PropertyGroup>` для надёжности). Запустите `dotnet build` — должно быть 0 ошибок и 0 предупреждений.

2. **Реализуйте top-level точку входа** в `Program.cs`. Разберите аргументы командной строки: первый аргумент — целевая директория (если не задан, используйте текущую `Directory.GetCurrentDirectory()`); необязательный флаг `--report <path>` задаёт путь к файлу отчёта (по умолчанию — `snapshot-report.txt` во временной папке системы). Выведите в консоль, какой именно путь будет использоваться, чтобы пользователь видел раскрытие относительного пути через `Path.GetFullPath`.

3. **Создайте класс `SnapshotEngine`** с методом `Task<SnapshotResult> RunAsync(string targetDir, string reportPath, CancellationToken ct)`. Внутри:
   - Проверьте существование директории через `Directory.Exists`; если нет — бросьте `DirectoryNotFoundException` с понятным сообщением.
   - Получите `DirectoryInfo` и вызовите `GetFiles(pattern: "*.txt", SearchOption.TopDirectoryOnly)`. Это вернёт `FileInfo[]` — каждый элемент уже кэширует `Length`, `CreationTime`, `LastWriteTime`, `Attributes`.
   - Посчитайте суммарный размер и количество файлов, используя LINQ (`files.Sum(f => f.Length)`). Объясните в комментарии, почему здесь выгодно работать с `FileInfo`, а не вызывать `new FileInfo(path).Length` повторно для каждого файла.

4. **Создайте резервную подпапку** `backup` внутри целевой директории через `Directory.CreateDirectory(Path.Combine(targetDir, "backup"))`. Скопируйте каждый текстовый файл туда, используя `FileInfo.CopyTo(destPath, overwrite: true)`. Логируйте каждую копию через вспомогательный метод `LogAsync`.

5. **Сгенерируйте отчёт.** Соберите строки вида `"<имя> | <размер> байт | создан <ISO-время> | <атрибуты>"`. Для временной метки используйте формат `"O"` (round-trip), как в примере урока. Запишите отчёт асинхронно через `await File.WriteAllTextAsync(reportPath, reportText, ct)`. Имя отчёта по умолчанию постройте как `Path.Combine(Path.GetTempPath(), $"dir-snapshot-{Guid.NewGuid():N}.txt")` — именно через `Guid`, чтобы обойти лимит `GetTempFileName()`.

6. **Реализуйте конкурентно-безопасный логгер** `ConcurrentFileLogger` с приватным `SemaphoreSlim(1, 1)`. Метод `LogAsync(string message)` должен: ожидать семафор, открывать лог-файл через `await File.AppendAllTextAsync(logPath, line)` под блокировкой, отпускать семафор в `finally`. В комментарии укажите, что `FileShare.Read` у `AppendAllText` позволяет другим процессам читать лог, но не писать — поэтому блокировка внутри процесса всё равно нужна.

7. **Добавьте обработку ошибок.** Если копирование одного файла падает с `IOException` или `UnauthorizedAccessException`, логируйте ошибку и продолжайте обработку остальных файлов; не роняйте весь процесс. В конце выведите краткую сводку: сколько файлов обработано успешно, сколько с ошибками, суммарный размер, путь к отчёту.

8. **Проверьте работу.** Создайте тестовую папку `M10/homework/dir-snapshot/test-data`, положите в неё три `.txt`-файла разного размера (например, через `echo` или вручную). Запустите `dotnet run --project DirSnapshot -- ./test-data`. Убедитесь, что: появилась папка `backup` с копиями; в консоли выведена статистика; во временной папке создан отчёт с GUID-именем; путь к отчёту напечатан. Запустите утилиту повторно — копии должны перезаписаться без ошибок благодаря `overwrite: true`.

9. **Проверьте кроссплатформенность локально.** Запустите `dotnet run -- ./test-data --report ./my-report.txt` и убедитесь, что `Path.GetFullPath` корректно раскрывает относительный путь относительно текущей рабочей директории, а не относительно `.exe`. Это ключевой момент урока: многие разработчики ошибочно ожидают раскрытие относительно сборки.

#### Требования к решению

- Целевая платформа: .NET 8, язык C# 12. Разрешены и поощряются top-level statements, pattern matching (`switch` выражения), collection expressions (`[]`), raw string literals (`"""..."""`) для шаблона отчёта, file-scoped namespaces.
- Все пути склеиваются исключительно через `Path.Combine` или `Path.Join`. Хардкод `\\` или `"\\"` в строковых литералах запрещён и должен быть обнаружен ревьюером как нарушение. Допускается `Path.DirectorySeparatorChar` в редких случаях.
- Для одноразовых действий (существует ли файл, удалить временный файл) используется статический `File`/`Directory`. Для серий операций над одним путём (перечисление + чтение метаданных каждого файла, копирование) используется `FileInfo`/`DirectoryInfo`.
- Весь файловый I/O — асинхронный: `WriteAllTextAsync`, `ReadAllTextAsync`, `AppendAllTextAsync`. Синхронные варианты допустимы только в холодном пути запуска (разбор аргументов), но не в горячем I/O.
- Все `FileStream`, `StreamReader`, `StreamWriter` (если появятся) оборачиваются в `using` или `await using`. Утечка дескрипторов недопустима.
- Временные имена файлов строятся через `Guid.NewGuid()` + `Path.Combine`, а не через `Path.GetTempFileName()`. В комментарии должно быть объяснено, почему (лимит на Windows).
- Логирование конкурентно-безопасно через `SemaphoreSlim`. Тест: два параллельных вызова `LogAsync` не должны перемешать строки.
- Обработка ошибок копирования не роняет весь процесс; ошибки логируются и учитываются в сводке.
- Код компилируется без предупреждений (`TreatWarningsAsErrors` опционально, но приветствуется), проходит `dotnet build` чисто.

#### Тонкости и подводные камни

- **`Path` не обращается к диску.** Вызов `Path.Combine`, `Path.GetFullPath`, `Path.GetFileName` только манипулирует строками. Это значит, что `Path.GetFullPath` не проверяет существование файла и раскрывает относительные пути относительно `Directory.GetCurrentDirectory()`, а не относительно исполняемой сборки. Если вы запускаете утилиту из другой рабочей директории, относительный путь развернётся иначе, чем вы ожидаете — это классический баг, прямо описанный в уроке.
- **Хардкод `\\` ломает Linux.** Строка `Path.Combine("data", "report.txt")` на Windows даст `data\report.txt`, на Linux — `data/report.txt`. Если вы напишете `"data\\report.txt"` вручную, на Linux получится буквально `data\report.txt` как одно имя файла, и `Directory.CreateDirectory` создаст папку с обратным слешем в имени. Всегда `Path.Combine`.
- **`FileInfo` кэширует метаданные.** После `new FileInfo(path)` свойства `Length`, `CreationTime`, `Attributes` берутся из кэша, сформированного в момент создания или последнего `Refresh()`. Если файл изменился между операциями, вызовите `fileInfo.Refresh()` перед повторным чтением. В нашем задании это не критично, потому что мы читаем метаданные сразу после `GetFiles()`, но знание этого факта защищает от тонких багов в логах.
- **`GetTempFileName()` имеет лимит.** На Windows метод генерирует имя из 8 шестнадцатеричных символов и исчерпывает ~65 535 уникальных имён в temp-папке, после чего бросает `IOException`. В серверных и циклических сценариях используйте `Guid.NewGuid():N` + `Path.Combine`. Урок явно об этом предупреждает, и ваше эталонное решение должно это демонстрировать.
- **`FileShare.Read` у `ReadAllText`.** Метод `File.ReadAllText`/`ReadAllTextAsync` открывает файл с `FileShare.Read`, что позволяет другим читателям читать одновременно, но блокирует писателей. И наоборот: пока вы пишете через `AppendAllTextAsync`, конкурентный читатель получит `IOException` о занятости файла. Поэтому для логов нужна внутрипроцессная блокировка (`SemaphoreSlim`), даже если файл открывается коротко.
- **Async vs sync в горячем пути.** Синхронный `File.ReadAllText` в цикле по сотням файлов заблокирует поток на дисковом I/O. В .NET 8 async-методы почти не имеют оверхеда и должны быть выбором по умолчанию для I/O. Но не злоупотребляйте `Task.Run` вокруг sync-вызовов — это антипаттерн; используйте настоящие `*Async`-методы BCL.
- **`Directory.CreateDirectory` идемпотентен.** Он не бросает исключение, если папка уже существует, и создаёт все промежуточные папки. Не нужно предварять его `Directory.Exists` — это лишний системный вызов, хотя в примере урока он есть для наглядности.

#### Критерии приёмки

- [ ] Проект `DirSnapshot` собирается командой `dotnet build` без ошибок и предупреждений.
- [ ] Целевая платформа — .NET 8, язык — C# 12 (top-level statements, file-scoped namespace).
- [ ] Ни одного хардкода `\\` или `"\\"` в строковых путях; все пути склеены через `Path.Combine`/`Path.Join`.
- [ ] Использован статический `File`/`Directory` для разовых проверок и `FileInfo`/`DirectoryInfo` для серий операций — выбор обоснован в комментариях.
- [ ] Весь горячий файловый I/O асинхронный (`*Async`-методы).
- [ ] Временное имя отчёта строится через `Guid.NewGuid():N` + `Path.Combine`, а не через `GetTempFileName()`.
- [ ] Реализован `ConcurrentFileLogger` с `SemaphoreSlim(1, 1)`, гарантирующий непересекающиеся строки.
- [ ] Копирование файлов использует `FileInfo.CopyTo(..., overwrite: true)`; повторный запуск не падает.
- [ ] Ошибка копирования одного файла логируется, обработка остальных продолжается, сводка отражает ошибки.
- [ ] В консоли печатается путь к отчёту и раскрытый через `Path.GetFullPath` целевой путь.
- [ ] Тестовый запуск `dotnet run -- ./test-data` создаёт папку `backup` и отчёт во временной папке.
- [ ] В коде есть комментарий, объясняющий, почему `Path.GetFullPath` раскрывает относительно `Directory.GetCurrentDirectory()`, а не относительно сборки.
- [ ] Нет утечки дескрипторов: все `FileStream`/`StreamReader`/`StreamWriter` в `using`/`await using`.
- [ ] Код проходит повторный запуск без побочных эффектов (перезапись, а не падение).
- [ ] В комментариях RU+EN указано, какой концепции урока соответствует каждый ключевой блок.

#### Подсказки (без прямого ответа)

- Подумайте, какой именно метод `DirectoryInfo` возвращает уже готовые `FileInfo[]` с кэшированными метаданными — это избавит вас от ручного `new FileInfo` в цикле.
- Для суммы размеров вспомните про LINQ `Sum`, но помните про возможное переполнение — на больших директориях `long` безопаснее `int`.
- Шаблон отчёта удобно собрать через `StringBuilder` или raw string literal с интерполяцией; для строки с ISO-временем используйте спецификатор `"O"`.
- Для конкурентной записи в лог подойдёт `SemaphoreSlim(1, 1)` с `await sem.WaitAsync(ct)` в `try` и `sem.Release()` в `finally`.
- Чтобы понять, почему `Guid`-имя безопаснее `GetTempFileName()`, перечитайте раздел «Временные файлы» в уроке.

#### Эталонное решение (разбор)

```csharp
// C# 12 / .NET 8 — DirSnapshot: снимок директории с резервным копированием и отчётом.
// Directory snapshot with backup and report. Закрепляет темы урока M10-L01.

using System.IO;
using System.Text;
using System.Threading;

// --- Точка входа / Entry point (top-level statements) ---
string targetDir = args.Length > 0 ? args[0] : Directory.GetCurrentDirectory();
// --report <path> — необязательный путь к отчёту / optional report path
string? reportOverride = null;
for (int i = 0; i < args.Length - 1; i++)
{
    if (args[i] == "--report")
    {
        reportOverride = args[i + 1];
    }
}

// Раскрываем относительный путь относительно Directory.GetCurrentDirectory(), а не .exe
// We resolve relative path against Directory.GetCurrentDirectory(), NOT against the .exe
string resolvedTarget = Path.GetFullPath(targetDir);
Console.WriteLine($"Целевая директория / Target: {resolvedTarget}");

// Имя отчёта по умолчанию: Guid во временной папке (обходим лимит GetTempFileName на Windows)
// Default report name: Guid in temp dir (avoids GetTempFileName limit on Windows)
string reportPath = reportOverride is null
    ? Path.Combine(Path.GetTempPath(), $"dir-snapshot-{Guid.NewGuid():N}.txt")
    : Path.GetFullPath(reportOverride);

var logger = new ConcurrentFileLogger(Path.Combine(Path.GetTempPath(), "dir-snapshot.log"));
var engine = new SnapshotEngine(logger);

try
{
    var result = await engine.RunAsync(resolvedTarget, reportPath, CancellationToken.None);
    Console.WriteLine($"""
        Готово / Done.
        Обработано файлов / Files processed: {result.ProcessedCount}
        С ошибками / With errors:          {result.ErrorCount}
        Суммарный размер / Total size:     {result.TotalBytes} байт / bytes
        Отчёт / Report:                    {result.ReportPath}
        """);
}
catch (Exception ex)
{
    Console.Error.WriteLine($"Фатальная ошибка / Fatal: {ex.Message}");
}

// --- SnapshotEngine: серия операций над директорией — используем DirectoryInfo/FileInfo ---
// Series of ops on one directory — use DirectoryInfo/FileInfo (cached metadata).

public sealed class SnapshotEngine(ConcurrentFileLogger logger)
{
    public async Task<SnapshotResult> RunAsync(string targetDir, string reportPath, CancellationToken ct)
    {
        // Разовая проверка существования — статический Directory.Exists / One-shot existence check
        if (!Directory.Exists(targetDir))
        {
            throw new DirectoryNotFoundException($"Директория не найдена / Directory not found: {targetDir}");
        }

        var dirInfo = new DirectoryInfo(targetDir);
        // GetFiles возвращает FileInfo[] с уже кэшированными Length/CreationTime/Attributes
        // GetFiles returns FileInfo[] with already-cached Length/CreationTime/Attributes
        FileInfo[] textFiles = dirInfo.GetFiles("*.txt", SearchOption.TopDirectoryOnly);

        long totalBytes = textFiles.Sum(f => f.Length);

        // Резервная подпапка через Path.Combine (кроссплатформенно) / Backup subdir via Path.Combine
        string backupDir = Path.Combine(targetDir, "backup");
        Directory.CreateDirectory(backupDir); // идемпотентен — создаёт все промежуточные / idempotent

        int processed = 0;
        int errors = 0;
        var reportLines = new List<string>(textFiles.Length + 4);

        reportLines.Add($"# Снимок директории / Directory snapshot: {targetDir}");
        reportLines.Add($"# Создан / Generated: {DateTimeOffset.Now:O}");
        reportLines.Add($"# Файлов / Files: {textFiles.Length}, размер / size: {totalBytes} bytes");
        reportLines.Add(new string('-', 60));

        foreach (var fi in textFiles)
        {
            string destPath = Path.Combine(backupDir, fi.Name);
            try
            {
                // FileInfo.CopyTo — серия операций над одним файлом (копирование + метаданные)
                // FileInfo.CopyTo — series of ops on one file (copy + metadata)
                fi.CopyTo(destPath, overwrite: true);
                fi.Refresh(); // обновляем кэш после копирования / refresh cache after copy
                reportLines.Add($"{fi.Name} | {fi.Length} bytes | created {fi.CreationTime:O} | {fi.Attributes}");
                await logger.LogAsync($"Скопирован / Copied: {fi.Name} -> {destPath}");
                processed++;
            }
            catch (IOException ioEx)
            {
                await logger.LogAsync($"Ошибка копирования / Copy error {fi.Name}: {ioEx.Message}");
                errors++;
            }
            catch (UnauthorizedAccessException uaEx)
            {
                await logger.LogAsync($"Нет доступа / Access denied {fi.Name}: {uaEx.Message}");
                errors++;
            }
        }

        // Асинхронная запись отчёта / Async report write
        string reportText = string.Join(Environment.NewLine, reportLines);
        await File.WriteAllTextAsync(reportPath, reportText, ct);

        return new SnapshotResult(processed, errors, totalBytes, reportPath);
    }
}

// --- Конкурентно-безопасный логгер: SemaphoreSlim защищает параллельную запись ---
// Concurrency-safe logger: SemaphoreSlim guards parallel writes.
public sealed class ConcurrentFileLogger(string logPath)
{
    private readonly SemaphoreSlim _gate = new(1, 1);

    public async Task LogAsync(string message)
    {
        string line = $"[{DateTimeOffset.Now:O}] {message}{Environment.NewLine}";
        await _gate.WaitAsync();
        try
        {
            // FileShare.Read у AppendAllTextAsync: другие читатели могут читать, но не писать
            // FileShare.Read on AppendAllTextAsync: other readers may read, but not write
            await File.AppendAllTextAsync(logPath, line);
        }
        finally
        {
            _gate.Release();
        }
    }
}

public sealed record SnapshotResult(int ProcessedCount, int ErrorCount, long TotalBytes, string ReportPath);
```

Разбор по строкам. Точка входа использует top-level statements C# 12: разбор аргументов через простой цикл, без тяжёлого парсера. Ключевой момент — `Path.GetFullPath(targetDir)`, который раскрывает относительный путь относительно `Directory.GetCurrentDirectory()`, а не относительно `.exe`; именно об этом предупреждает урок в разделе «Кроссплатформенность». Печать раскрытого пути в консоль делает поведение прозрачным для пользователя и помогает отладке.

Имя отчёта по умолчанию строится как `Path.Combine(Path.GetTempPath(), $"dir-snapshot-{Guid.NewGuid():N}.txt")`. Здесь сознательно отказываемся от `Path.GetTempFileName()`: урок предупреждает, что на Windows у него лимит ~65 535 уникальных имён и в плотном цикле он бросает `IOException`. `Guid`-имя не имеет такого лимита и предсказуемо уникально. Сам `Path.GetTempPath()` возвращает системную temp-папку (`%TEMP%` на Windows, `/tmp` на Linux) — это прямой перенос материала урока про временные файлы.

`SnapshotEngine` — это ровно тот случай, где инстансные классы эффективнее статических: мы делаем серию операций над одной директорией (существование, `GetFiles`, `CreateDirectory` для подпапки). `DirectoryInfo.GetFiles` возвращает `FileInfo[]`, где каждый элемент уже кэширует `Length`, `CreationTime`, `Attributes` — нам не нужно повторно вызывать `new FileInfo(path)` для чтения размера. Сумма размеров через `Sum(f => f.Length)` в `long` безопасна от переполнения. `fi.CopyTo(destPath, overwrite: true)` — снова `FileInfo`, потому что для каждого файла мы и копируем, и читаем метаданные; статический `File.Copy` потребовал бы повторного `new FileInfo`.

Обработка ошибок копирования ловит `IOException` и `UnauthorizedAccessException` отдельно, логирует и продолжает — это критично для надёжности утилиты, обрабатывающей десятки файлов: один залоченный файл не должен ронять весь снимок. `fi.Refresh()` перед чтением метаданных после копирования обновляет кэш `FileInfo` — это та самая тонкость, о которой говорит урок: инстансные классы кэшируют данные, и устаревший кэш даёт неверные размеры.

`ConcurrentFileLogger` использует `SemaphoreSlim(1, 1)` — прямой ответ на concurrency-аспект урока. Файловые операции не атомарны, и без блокировки две параллельные записи через `AppendAllTextAsync` могли бы перемешать строки или бросить `IOException` из-за конкуренции за дескриптор. `File.AppendAllTextAsync` открывает файл с `FileShare.Read`, что позволяет другим процессам читать лог, но не писать — поэтому внутрипроцессная блокировка всё равно необходима. Паттерн `try/finally` с `Release()` гарантирует, что семафор не «зависнет» при исключении.

Асинхронная запись отчёта через `await File.WriteAllTextAsync(reportPath, reportText, ct)` — это применение best practice урока: предпочитать `*Async`-методы для I/O, чтобы не блокировать поток. Raw string literal `"""..."""` в выводе итогов — современная возможность C# 11+, доступная в .NET 8. Запись `record SnapshotResult` — лаконичный способ вернутьimmutable-результат.

#### Задания на углубление (бонус)

1. **Рекурсивный снимок.** Добавьте флаг `--recursive`, который переключает `SearchOption` на `AllDirectories`. Обработайте `UnauthorizedAccessException` при заходе в системные папки так, чтобы пропускать их, но фиксировать в отчёте отдельной секцией «Skipped».
2. **Параллельное копирование.** Замените последовательный `foreach` на `Parallel.ForEachAsync` с ограничением степени параллелизма (например, `MaxDegreeOfParallelism = 4`). Убедитесь, что `SemaphoreSlim`-логгер корректно работает под параллельной нагрузкой, и измерьте ускорение через `Stopwatch`.
3. **Инкрементальный backup.** Сравнивайте `LastWriteTime` источника и копии в `backup`; копируйте только изменённые файлы. Сохраняйте состояние в JSON-файл `snapshot-state.json` через `System.Text.Json`.
4. **Хэширование.** Добавьте вычисление SHA-256 для каждого копируемого файла через `FileStream` + `SHA256.HashDataAsync` и записывайте хэш в отчёт. Это плавно подведёт к следующему уроку M10-L02 про `Stream`/`FileStream`.

---

## Statement in English / Постановка на английском

#### Context & motivation

Imagine you are a junior developer on a team that maintains an internal tool for backing up text logs. Every evening a service drops `.txt` files into a working folder, and your manager wants a simple but reliable CLI utility, `dir-snapshot`, that on demand takes a “snapshot” of a given directory: lists the files, computes the total size, copies every text file into a backup subfolder, and produces a human-readable report. The tool must run identically on a Windows server and in a Linux container, because production is migrating to Linux. That means any hardcoded backslash instantly breaks the run inside Docker.

This is exactly where the topics of lesson M10-L01 come into play. A one-off existence check is a job for the static `File.Exists`. Enumerating a directory while reading each file’s size, creation time, and attributes is the classic scenario for `DirectoryInfo.GetFiles()`, which returns a `FileInfo[]` where every element already caches the metadata. Path composition must go through `Path.Combine`, never through `+` and `"\"`, or cross-platform correctness is lost. The report must be written asynchronously so the thread is not blocked, and the intermediate file’s temporary name must be generated via `Guid`, not via `Path.GetTempFileName()`, because the latter has a non-obvious name limit on Windows that the lesson explicitly warns about.

Finally, the utility must log its actions to a shared log file that several running instances may touch at the same time. This brings us to the concurrency angle: file operations are not atomic, and parallel writes without synchronization produce garbled, interleaved lines. You will implement a simple lock with `SemaphoreSlim`, directly exercising the lesson material about synchronizing writers.

#### What to do step by step

1. **Create the project.** Inside `M10/homework/dir-snapshot` run `dotnet new console -n DirSnapshot -o DirSnapshot --framework net8.0`. Open `DirSnapshot.csproj` and make sure `LangVersion` targets C# 12 (you may add `<PropertyGroup><LangVersion>latest</LangVersion></PropertyGroup>` to be safe). Run `dotnet build` — it must report 0 errors and 0 warnings.

2. **Implement a top-level entry point** in `Program.cs`. Parse command-line arguments: the first argument is the target directory (if omitted, use `Directory.GetCurrentDirectory()`); an optional `--report <path>` flag sets the report file path (default: `snapshot-report.txt` in the system temp folder). Print the resolved path to the console so the user can see how `Path.GetFullPath` expands the relative path.

3. **Create a `SnapshotEngine` class** with a method `Task<SnapshotResult> RunAsync(string targetDir, string reportPath, CancellationToken ct)`. Inside:
   - Check the directory existence with `Directory.Exists`; if it is missing, throw `DirectoryNotFoundException` with a clear message.
   - Obtain a `DirectoryInfo` and call `GetFiles(pattern: "*.txt", SearchOption.TopDirectoryOnly)`. This returns a `FileInfo[]` where each element already caches `Length`, `CreationTime`, `LastWriteTime`, and `Attributes`.
   - Compute the total size and file count with LINQ (`files.Sum(f => f.Length)`). Explain in a comment why working with `FileInfo` is advantageous here instead of calling `new FileInfo(path).Length` repeatedly per file.

4. **Create the backup subfolder** named `backup` inside the target directory via `Directory.CreateDirectory(Path.Combine(targetDir, "backup"))`. Copy every text file into it using `FileInfo.CopyTo(destPath, overwrite: true)`. Log each copy through a helper `LogAsync` method.

5. **Generate the report.** Assemble lines of the form `"<name> | <size> bytes | created <ISO time> | <attributes>"`. Use the `"O"` (round-trip) format for the timestamp, as in the lesson example. Write the report asynchronously with `await File.WriteAllTextAsync(reportPath, reportText, ct)`. Build the default report name as `Path.Combine(Path.GetTempPath(), $"dir-snapshot-{Guid.NewGuid():N}.txt")` — through `Guid` precisely to avoid the `GetTempFileName()` limit.

6. **Implement a concurrency-safe logger** `ConcurrentFileLogger` with a private `SemaphoreSlim(1, 1)`. Its `LogAsync(string message)` method should: await the semaphore, open the log file with `await File.AppendAllTextAsync(logPath, line)` under the lock, and release the semaphore in `finally`. In a comment, note that `FileShare.Read` on `AppendAllText` lets other processes read the log but not write — that is why intra-process locking is still required.

7. **Add error handling.** If copying a single file fails with `IOException` or `UnauthorizedAccessException`, log the error and continue with the remaining files; do not crash the whole process. At the end, print a summary: how many files processed successfully, how many with errors, total size, report path.

8. **Verify the run.** Create a test folder `M10/homework/dir-snapshot/test-data`, drop three `.txt` files of different sizes into it (via `echo` or manually). Run `dotnet run --project DirSnapshot -- ./test-data`. Confirm that: a `backup` folder with copies appears; the console prints statistics; a report with a GUID name is created in the temp folder; the report path is printed. Run the utility again — copies must be overwritten without errors thanks to `overwrite: true`.

9. **Check cross-platform behavior locally.** Run `dotnet run -- ./test-data --report ./my-report.txt` and confirm that `Path.GetFullPath` correctly resolves the relative path against the current working directory, not against the `.exe`. This is a key point of the lesson: many developers wrongly expect resolution against the assembly.

#### Requirements

- Target: .NET 8, language C# 12. Top-level statements, pattern matching (`switch` expressions), collection expressions (`[]`), and raw string literals (`"""..."""`) for the report template are allowed and encouraged; file-scoped namespaces are preferred.
- All paths are joined exclusively via `Path.Combine` or `Path.Join`. Hardcoding `\\` or `"\\"` inside string literals is forbidden and must be flagged by the reviewer as a violation. `Path.DirectorySeparatorChar` is acceptable in rare cases.
- One-shot actions (does a file exist, delete a temp file) use the static `File`/`Directory`. Series of operations on one path (enumeration plus reading each file’s metadata, copying) use `FileInfo`/`DirectoryInfo`.
- All hot file I/O is asynchronous: `WriteAllTextAsync`, `ReadAllTextAsync`, `AppendAllTextAsync`. Synchronous variants are allowed only on the cold startup path (argument parsing), never on hot I/O.
- Any `FileStream`, `StreamReader`, `StreamWriter` that appears is wrapped in `using` or `await using`. Handle leaks are unacceptable.
- Temporary file names are built with `Guid.NewGuid()` + `Path.Combine`, not with `Path.GetTempFileName()`. A comment must explain why (the Windows name limit).
- Logging is concurrency-safe via `SemaphoreSlim`. Test: two parallel `LogAsync` calls must not interleave lines.
- Copy errors do not crash the whole process; errors are logged and reflected in the summary.
- The code compiles without warnings (`TreatWarningsAsErrors` optional but welcome); `dotnet build` is clean.

#### Pitfalls

- **`Path` never touches the disk.** Calls to `Path.Combine`, `Path.GetFullPath`, `Path.GetFileName` only manipulate strings. That means `Path.GetFullPath` does not verify the file exists and resolves relative paths against `Directory.GetCurrentDirectory()`, not against the executable assembly. If you launch the utility from a different working directory, a relative path expands differently than you might expect — the classic bug described in the lesson.
- **Hardcoded `\\` breaks Linux.** `Path.Combine("data", "report.txt")` yields `data\report.txt` on Windows and `data/report.txt` on Linux. If you write `"data\\report.txt"` by hand, Linux will treat `data\report.txt` as a single literal filename, and `Directory.CreateDirectory` will create a folder with a backslash in its name. Always use `Path.Combine`.
- **`FileInfo` caches metadata.** After `new FileInfo(path)`, the `Length`, `CreationTime`, and `Attributes` properties come from a cache formed at construction or at the last `Refresh()`. If the file changes between operations, call `fileInfo.Refresh()` before reading again. In this assignment it is not critical because we read metadata immediately after `GetFiles()`, but knowing this guards you from subtle log bugs.
- **`GetTempFileName()` has a limit.** On Windows the method generates an 8-character hex name and exhausts roughly 65,535 unique names in the temp folder, after which it throws `IOException`. In server and loop scenarios use `Guid.NewGuid():N` + `Path.Combine`. The lesson warns about this explicitly, and your reference solution must demonstrate the alternative.
- **`FileShare.Read` on `ReadAllText`.** `File.ReadAllText`/`ReadAllTextAsync` opens the file with `FileShare.Read`, which allows other readers to read concurrently but blocks writers. Conversely, while you write via `AppendAllTextAsync`, a concurrent reader will get an `IOException` about the file being in use. That is why logs need intra-process locking (`SemaphoreSlim`) even when the file is opened briefly.
- **Async vs sync on the hot path.** A synchronous `File.ReadAllText` inside a loop over hundreds of files will block the thread on disk I/O. In .NET 8 async methods have negligible overhead and should be the default for I/O. But do not abuse `Task.Run` around sync calls — that is an antipattern; use real BCL `*Async` methods instead.
- **`Directory.CreateDirectory` is idempotent.** It does not throw if the folder already exists and creates all intermediate folders. You do not need to precede it with `Directory.Exists` — that is a redundant system call, although the lesson example keeps it for clarity.

#### Acceptance criteria

- [ ] The `DirSnapshot` project builds with `dotnet build` with no errors and no warnings.
- [ ] Target is .NET 8, language C# 12 (top-level statements, file-scoped namespace).
- [ ] No hardcoded `\\` or `"\\"` in string paths; all paths joined via `Path.Combine`/`Path.Join`.
- [ ] Static `File`/`Directory` is used for one-shot checks and `FileInfo`/`DirectoryInfo` for series — the choice is justified in comments.
- [ ] All hot file I/O is asynchronous (`*Async` methods).
- [ ] The report’s temporary name is built via `Guid.NewGuid():N` + `Path.Combine`, not via `GetTempFileName()`.
- [ ] A `ConcurrentFileLogger` with `SemaphoreSlim(1, 1)` guarantees non-interleaved lines.
- [ ] File copying uses `FileInfo.CopyTo(..., overwrite: true)`; a second run does not crash.
- [ ] A copy error on one file is logged; other files keep being processed; the summary reflects errors.
- [ ] The report path and the `Path.GetFullPath`-resolved target path are printed to the console.
- [ ] A test run `dotnet run -- ./test-data` creates a `backup` folder and a report in the temp folder.
- [ ] A comment explains why `Path.GetFullPath` resolves against `Directory.GetCurrentDirectory()` and not against the assembly.
- [ ] No handle leaks: every `FileStream`/`StreamReader`/`StreamWriter` is in `using`/`await using`.
- [ ] A second run has no side effects (overwrite, not crash).
- [ ] Comments in RU+EN indicate which lesson concept each key block corresponds to.

#### Hints (no direct answer)

- Recall which `DirectoryInfo` method returns ready-made `FileInfo[]` with cached metadata — it spares you from a manual `new FileInfo` inside the loop.
- For the total size, remember LINQ `Sum`, but mind potential overflow — on large directories `long` is safer than `int`.
- The report template is convenient to build with a `StringBuilder` or a raw string literal with interpolation; for the ISO timestamp use the `"O"` specifier.
- For concurrent log writes, `SemaphoreSlim(1, 1)` with `await sem.WaitAsync(ct)` in `try` and `sem.Release()` in `finally` fits well.
- To understand why a `Guid` name is safer than `GetTempFileName()`, re-read the “Temporary files” section of the lesson.

#### Reference solution walk-through

```csharp
// C# 12 / .NET 8 — DirSnapshot: directory snapshot with backup and report.
// Reinforces lesson M10-L01 topics.

using System.IO;
using System.Text;
using System.Threading;

// --- Entry point (top-level statements) ---
string targetDir = args.Length > 0 ? args[0] : Directory.GetCurrentDirectory();
// --report <path> — optional report path
string? reportOverride = null;
for (int i = 0; i < args.Length - 1; i++)
{
    if (args[i] == "--report")
    {
        reportOverride = args[i + 1];
    }
}

// Resolve relative path against Directory.GetCurrentDirectory(), NOT against the .exe
string resolvedTarget = Path.GetFullPath(targetDir);
Console.WriteLine($"Target: {resolvedTarget}");

// Default report name: Guid in temp dir (avoids the GetTempFileName limit on Windows)
string reportPath = reportOverride is null
    ? Path.Combine(Path.GetTempPath(), $"dir-snapshot-{Guid.NewGuid():N}.txt")
    : Path.GetFullPath(reportOverride);

var logger = new ConcurrentFileLogger(Path.Combine(Path.GetTempPath(), "dir-snapshot.log"));
var engine = new SnapshotEngine(logger);

try
{
    var result = await engine.RunAsync(resolvedTarget, reportPath, CancellationToken.None);
    Console.WriteLine($"""
        Done.
        Files processed: {result.ProcessedCount}
        With errors:     {result.ErrorCount}
        Total size:      {result.TotalBytes} bytes
        Report:          {result.ReportPath}
        """);
}
catch (Exception ex)
{
    Console.Error.WriteLine($"Fatal: {ex.Message}");
}

// --- SnapshotEngine: a series of ops on one directory — use DirectoryInfo/FileInfo ---

public sealed class SnapshotEngine(ConcurrentFileLogger logger)
{
    public async Task<SnapshotResult> RunAsync(string targetDir, string reportPath, CancellationToken ct)
    {
        // One-shot existence check — static Directory.Exists
        if (!Directory.Exists(targetDir))
        {
            throw new DirectoryNotFoundException($"Directory not found: {targetDir}");
        }

        var dirInfo = new DirectoryInfo(targetDir);
        // GetFiles returns FileInfo[] with already-cached Length/CreationTime/Attributes
        FileInfo[] textFiles = dirInfo.GetFiles("*.txt", SearchOption.TopDirectoryOnly);

        long totalBytes = textFiles.Sum(f => f.Length);

        // Backup subdir via Path.Combine (cross-platform)
        string backupDir = Path.Combine(targetDir, "backup");
        Directory.CreateDirectory(backupDir); // idempotent — creates intermediate folders

        int processed = 0;
        int errors = 0;
        var reportLines = new List<string>(textFiles.Length + 4);

        reportLines.Add($"# Directory snapshot: {targetDir}");
        reportLines.Add($"# Generated: {DateTimeOffset.Now:O}");
        reportLines.Add($"# Files: {textFiles.Length}, size: {totalBytes} bytes");
        reportLines.Add(new string('-', 60));

        foreach (var fi in textFiles)
        {
            string destPath = Path.Combine(backupDir, fi.Name);
            try
            {
                // FileInfo.CopyTo — a series of ops on one file (copy + metadata)
                fi.CopyTo(destPath, overwrite: true);
                fi.Refresh(); // refresh the cache after copying
                reportLines.Add($"{fi.Name} | {fi.Length} bytes | created {fi.CreationTime:O} | {fi.Attributes}");
                await logger.LogAsync($"Copied: {fi.Name} -> {destPath}");
                processed++;
            }
            catch (IOException ioEx)
            {
                await logger.LogAsync($"Copy error {fi.Name}: {ioEx.Message}");
                errors++;
            }
            catch (UnauthorizedAccessException uaEx)
            {
                await logger.LogAsync($"Access denied {fi.Name}: {uaEx.Message}");
                errors++;
            }
        }

        // Async report write
        string reportText = string.Join(Environment.NewLine, reportLines);
        await File.WriteAllTextAsync(reportPath, reportText, ct);

        return new SnapshotResult(processed, errors, totalBytes, reportPath);
    }
}

// --- Concurrency-safe logger: SemaphoreSlim guards parallel writes ---

public sealed class ConcurrentFileLogger(string logPath)
{
    private readonly SemaphoreSlim _gate = new(1, 1);

    public async Task LogAsync(string message)
    {
        string line = $"[{DateTimeOffset.Now:O}] {message}{Environment.NewLine}";
        await _gate.WaitAsync();
        try
        {
            // FileShare.Read on AppendAllTextAsync: other readers may read, but not write
            await File.AppendAllTextAsync(logPath, line);
        }
        finally
        {
            _gate.Release();
        }
    }
}

public sealed record SnapshotResult(int ProcessedCount, int ErrorCount, long TotalBytes, string ReportPath);
```

Line-by-line walk-through. The entry point uses C# 12 top-level statements: argument parsing through a plain loop, without a heavy parser. The key moment is `Path.GetFullPath(targetDir)`, which resolves the relative path against `Directory.GetCurrentDirectory()` and not against the `.exe`; that is exactly what the lesson warns about in the “Cross-platform” section. Printing the resolved path makes the behavior transparent to the user and aids debugging.

The default report name is built as `Path.Combine(Path.GetTempPath(), $"dir-snapshot-{Guid.NewGuid():N}.txt")`. We deliberately avoid `Path.GetTempFileName()`: the lesson warns that on Windows it has a limit of roughly 65,535 unique names and throws `IOException` in a tight loop. A `Guid`-based name has no such limit and is predictably unique. `Path.GetTempPath()` itself returns the system temp folder (`%TEMP%` on Windows, `/tmp` on Linux) — a direct application of the lesson material on temporary files.

`SnapshotEngine` is precisely the case where instance classes beat static ones: we perform a series of operations on a single directory (existence, `GetFiles`, `CreateDirectory` for the subfolder). `DirectoryInfo.GetFiles` returns a `FileInfo[]` where each element already caches `Length`, `CreationTime`, and `Attributes` — we do not need to call `new FileInfo(path)` again to read the size. The size sum via `Sum(f => f.Length)` in `long` is safe from overflow. `fi.CopyTo(destPath, overwrite: true)` is again `FileInfo`, because for each file we both copy and read metadata; the static `File.Copy` would require a redundant `new FileInfo`.

Copy error handling catches `IOException` and `UnauthorizedAccessException` separately, logs, and continues — this is critical for the reliability of a utility processing dozens of files: a single locked file must not bring down the whole snapshot. `fi.Refresh()` before reading metadata after the copy refreshes the `FileInfo` cache — the very subtlety the lesson highlights: instance classes cache data, and a stale cache yields wrong sizes.

`ConcurrentFileLogger` uses `SemaphoreSlim(1, 1)` — a direct answer to the lesson’s concurrency angle. File operations are not atomic, and without a lock two parallel writes via `AppendAllTextAsync` could interleave lines or throw `IOException` from handle contention. `File.AppendAllTextAsync` opens the file with `FileShare.Read`, which lets other processes read the log but not write — that is why intra-process locking is still necessary. The `try/finally` pattern with `Release()` guarantees the semaphore does not get stuck on an exception.

The asynchronous report write via `await File.WriteAllTextAsync(reportPath, reportText, ct)` applies the lesson best practice: prefer `*Async` methods for I/O so the thread is not blocked. The raw string literal `"""..."""` in the final output is a modern C# 11+ feature available in .NET 8. The `record SnapshotResult` is a concise way to return an immutable result.

#### Going deeper (bonus)

1. **Recursive snapshot.** Add a `--recursive` flag that switches `SearchOption` to `AllDirectories`. Handle `UnauthorizedAccessException` when descending into system folders so they are skipped but recorded in a separate “Skipped” section of the report.
2. **Parallel copying.** Replace the sequential `foreach` with `Parallel.ForEachAsync` with a bounded degree of parallelism (say `MaxDegreeOfParallelism = 4`). Confirm the `SemaphoreSlim` logger behaves under parallel load, and measure the speedup with `Stopwatch`.
3. **Incremental backup.** Compare `LastWriteTime` of the source and the copy in `backup`; copy only changed files. Persist state in a `snapshot-state.json` file via `System.Text.Json`.
4. **Hashing.** Add SHA-256 computation for each copied file via `FileStream` + `SHA256.HashDataAsync` and record the hash in the report. This smoothly leads into the next lesson M10-L02 on `Stream`/`FileStream`.

---

#### Чек-лист сдачи / Submission checklist
- [ ] (RU) Проект `DirSnapshot` собирается без ошибок и предупреждений.
- [ ] (RU) Целевая платформа .NET 8 / C# 12, top-level statements.
- [ ] (RU) Все пути склеены через `Path.Combine`, без хардкода `\\`.
- [ ] (RU) Статические `File`/`Directory` для разовых операций, `FileInfo`/`DirectoryInfo` для серий.
- [ ] (RU) Асинхронный файловый I/O (`*Async`).
- [ ] (RU) Временное имя через `Guid`, не через `GetTempFileName()`.
- [ ] (RU) `ConcurrentFileLogger` с `SemaphoreSlim`.
- [ ] (RU) Обработка ошибок копирования не роняет процесс.
- [ ] (RU) В консоли печатается путь к отчёту и раскрытый целевой путь.
- [ ] (RU) Повторный запуск перезаписывает, а не падает.
- [ ] (EN) The `DirSnapshot` project builds with no errors or warnings.
- [ ] (EN) Target .NET 8 / C# 12, top-level statements.
- [ ] (EN) All paths joined via `Path.Combine`, no hardcoded `\\`.
- [ ] (EN) Static `File`/`Directory` for one-shot ops, `FileInfo`/`DirectoryInfo` for series.
- [ ] (EN) Asynchronous file I/O (`*Async`).
- [ ] (EN) Temp name via `Guid`, not via `GetTempFileName()`.
- [ ] (EN) `ConcurrentFileLogger` with `SemaphoreSlim`.
- [ ] (EN) Copy errors do not crash the process.
- [ ] (EN) The report path and resolved target path are printed.
- [ ] (EN) A second run overwrites instead of crashing.

#### Ресурсы / Resources
- [Microsoft Learn — File class](https://learn.microsoft.com/dotnet/api/system.io.file)
- [Microsoft Learn — FileInfo class](https://learn.microsoft.com/dotnet/api/system.io.fileinfo)
- [Microsoft Learn — Directory class](https://learn.microsoft.com/dotnet/api/system.io.directory)
- [Microsoft Learn — DirectoryInfo class](https://learn.microsoft.com/dotnet/api/system.io.directoryinfo)
- [Microsoft Learn — Path class](https://learn.microsoft.com/dotnet/api/system.io.path)
- [Microsoft Learn — Path.GetTempFileName](https://learn.microsoft.com/dotnet/api/system.io.path.gettempfilename)
- [Microsoft Learn — SemaphoreSlim](https://learn.microsoft.com/dotnet/api/system.threading.semaphoreslim)
- [Microsoft Learn — File.ReadAllTextAsync](https://learn.microsoft.com/dotnet/api/system.io.file.readalltextasync)
