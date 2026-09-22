---
[← К уроку M10-L03](lesson-M10-L03-streamreader-writer-encoding.md) | [⬆ К модулю M10](../README.md) | [Следующее ДЗ →](homework-M10-L04-async-file-io.md)
---

### Домашнее задание M10-L03: StreamReader/StreamWriter, кодировки / Homework M10-L03: StreamReader/StreamWriter, encodings

**Урок / Lesson:** M10-L03
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 4/5
**Цель / Goal:** (RU) Научиться читать и записывать текстовые файлы через `StreamReader`/`StreamWriter` с явным контролем кодировки (UTF-8 без BOM, UTF-16, windows-1251), корректно управлять буфером и ресурсами через `using`/`await using`, обрабатывать большие файлы построчно и асинхронно, а также безопасно сериализовать числа и даты через `CultureInfo.InvariantCulture` для межсистемного обмена. (EN) Learn to read and write text files with `StreamReader`/`StreamWriter` while explicitly controlling the encoding (UTF-8 without BOM, UTF-16, windows-1251), correctly managing buffers and resources through `using`/`await using`, processing large files line by line and asynchronously, and safely serializing numbers and dates with `CultureInfo.InvariantCulture` for cross-system exchange.

#### Связь с уроком / Connection to the lesson
(RU) Урок вводит `StreamReader`/`StreamWriter` как мост между байтовым `Stream` и текстом, объясняет роль кодировок, BOM, асинхронного чтения и инвариантной культуры. ДЗ закрепляет все эти концепции на практике: вы построите конвертер форматов, который читает «грязные» исходные файлы в разных кодировках, нормализует данные и пишет чистый UTF-8-вывод без BOM, готовый к парсингу другой программой.
(EN) The lesson introduces `StreamReader`/`StreamWriter` as the bridge between a byte `Stream` and text, and explains the role of encodings, BOM, async reading and invariant culture. This homework puts every one of those concepts to work: you will build a format converter that reads "dirty" source files in different encodings, normalizes the data, and writes clean BOM-less UTF-8 output ready for parsing by another program.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация
Вы — инженер в компании, которая мигрирует старую систему отчётности на новую платформу .NET 8. Старая система десятилетиями копила текстовые файлы в самых разных кодировках: часть отчётов хранится в устаревшей однобайтной `windows-1251`, часть экспортировалась из Excel как UTF-16 LE с BOM, а самые свежие файлы уже приходят в UTF-8, причём иногда с BOM, иногда без. Новая платформа требует единый канонический формат: UTF-8 без BOM, числа в инвариантной культуре (десятичная точка), даты в формате ISO-8601 (`O`/`Round-trip`), каждая логическая запись на отдельной строке. Кроме того, отдельные исходные файлы достигают нескольких гигабайт — их нельзя загружать в память целиком, нужен потоковый построчный разбор. Ваша задача — написать консольную утилиту `EncodingNormalizer`, которая принимает на вход каталог исходных файлов, определяет их кодировку по расширению и BOM, читает данные построчно и асинхронно, преобразует каждую строку и записывает результат в новый каталог в канонической форме. Утилита должна быть устойчивой: некорректные числа и даты не должны ронять процесс, а должны попадать в отдельный лог ошибок с указанием строки и имени файла. Эта задача моделирует реальный сценарий миграции данных и заставляет вас одновременно применить работу с кодировками, async I/O, инвариантную культуру и корректное управление ресурсами — то есть все ключевые темы урока M10-L03.

#### Что нужно сделать (пошагово)
1. Создайте новый консольный проект .NET 8 с помощью команды `dotnet new console -n EncodingNormalizer -o EncodingNormalizer --framework net8.0`. Убедитесь, что в `EncodingNormalizer.csproj` установлен `<Nullable>enable</Nullable>` и `<ImplicitUsings>enable</ImplicitUsings>`.
2. Добавьте поддержку устаревших кодировок, установив пакет `dotnet add package System.Text.Encoding.CodePages`. Без него `Encoding.GetEncoding("windows-1251")` выбросит `ArgumentException` на платформах, не поддерживающих эту кодировку по умолчанию.
3. В точке входа программы (top-level statements, `Program.cs`) зарегистрируйте поставщик кодировок вызовом `Encoding.RegisterProvider(CodePagesEncodingProvider.Instance);` — это нужно сделать ровно один раз до первого обращения к `GetEncoding`.
4. Реализуйте метод `static Encoding DetectEncoding(string path)`, который открывает файл, читает первые три байта и проверяет наличие BOM UTF-8 (`0xEF 0xBB 0xBF`), UTF-16 LE (`0xFF 0xFE`) и UTF-16 BE (`0xFE 0xFF`). Если BOM есть, верните соответствующую кодировку через `Encoding.UTF8`, `Encoding.Unicode` или `Encoding.BigEndianUnicode`. Если BOM отсутствует, используйте эвристику по расширению: `.1251` → `windows-1251`, `.utf16` → `Encoding.Unicode`, всё остальное → UTF-8 без BOM (`new UTF8Encoding(false)`).
5. Реализуйте метод `static async Task NormalizeFileAsync(string src, string dst, StreamWriter errorLog, CancellationToken ct)`, который открывает исходный файл через `new StreamReader(src, DetectEncoding(src), detectEncodingFromByteOrderMarks: true)`, оборачивает его в `await using`, а целевой файл открывает через `await using var out = new StreamWriter(dst, append: false, new UTF8Encoding(false))`. Запись ведётся строго в UTF-8 без BOM.
6. Каждая строка исходного файла имеет формат `date|amount|label`, например `2024-03-14|1234.56|Отчёт по отделу`. При чтении разбивайте строку по символу `|` (не используйте тяжёлый `Regex` там, где хватит `Split` или `AsSpan`), парсите дату через `DateTime.ParseExact(..., "yyyy-MM-dd", CultureInfo.InvariantCulture)` и сумму через `double.Parse(..., CultureInfo.InvariantCulture)`. На выход записывайте каноническую строку `date=O;amount=Invariant;label` с помощью интерполяции и `CultureInfo.InvariantCulture`, например `date=2024-03-14T00:00:00.0000000Z;amount=1234.56;label=Отчёт по отделу`.
7. Если парсинг даты или суммы падает с исключением, перехватите его, запишите в `errorLog` строку вида `[src:line] message` через `await errorLog.WriteLineAsync(...)`, и продолжайте обработку следующих строк. Не прерывайте весь файл из-за одной битой строки.
8. Для больших файлов используйте именно цикл `while ((line = await reader.ReadLineAsync(ct)) is not null)`, а не `ReadToEndAsync()`. Передавайте `CancellationToken` во все async-вызовы, чтобы можно было прервать операцию по Ctrl+C.
9. Реализуйте метод `static async Task NormalizeDirectoryAsync(string srcDir, string dstDir, string errorLogPath, CancellationToken ct)`, который перебирает все файлы через `Directory.EnumerateFiles(srcDir, "*", SearchOption.TopDirectoryOnly)`, создаёт целевой каталог через `Directory.CreateDirectory(dstDir)`, открывает общий лог ошибок через `await using StreamWriter errorLog = new(errorLogPath, append: false, new UTF8Encoding(false))` и для каждого файла вызывает `NormalizeFileAsync`. Имя выходного файла получайте заменой расширения на `.normalized.txt`.
10. В `Program.cs` обработайте аргументы командной строки: если переданы `--src` и `--dst`, используйте их; иначе используйте каталоги `./input` и `./output` по умолчанию. Создайте `CancellationTokenSource`, подпишитесь на `Console.CancelKeyPress` для корректной отмены, и вызовите `NormalizeDirectoryAsync`. В конце выведите сводку: сколько файлов обработано, сколько строк записано, сколько ошибок зафиксировано.
11. Подготовьте тестовые данные: создайте каталог `input`, положите туда три файла — `report-utf8.txt` (UTF-8 без BOM), `legacy-1251.txt` (windows-1251) и `excel-utf16.txt` (UTF-16 LE с BOM). Каждый файл должен содержать не менее пяти корректных строк формата `date|amount|label` и одну заведомо битую строку (например, `bad|not-a-number|oops`) для проверки лога ошибок.
12. Запустите утилиту командой `dotnet run --project EncodingNormalizer -- --src input --dst output`. Проверьте, что в каталоге `output` появились три файла `.normalized.txt`, все в UTF-8 без BOM (проверьте первые три байта любым hex-просмотрщиком или `Get-Content -Encoding Byte -TotalCount 3`), а в файле `errors.log` зафиксированы три строки с ошибками парсинга.
13. Убедитесь, что программа не падает на пустых строках и строках с лишними пробелами — используйте `Trim()` и проверку `IsNullOrEmpty` перед разбором.
14. Сравните количество строк в исходных и выходных файлах: количество корректных строк должно совпадать, битые строки должны отсутствовать в выходном файле, но присутствовать в логе.

#### Требования к решению
- Целевая платформа: .NET 8, C# 12. Разрешены top-level statements, pattern matching (`is not null`, `is { }`), collection expressions, raw string literals для многострочных шаблонов.
- Все reader/writer должны быть обёрнуты в `using` или `await using`. Никаких «голых» `new StreamReader(...)` без dispose.
- Выходные файлы обязаны быть в UTF-8 без BOM. Используйте `new UTF8Encoding(encoderShouldEmitUTF8Identifier: false)` или дефолтный конструктор `StreamWriter` — но НЕ `Encoding.UTF8` напрямую, потому что он пишет BOM.
- Все числа и даты должны сериализоваться и парситься только через `CultureInfo.InvariantCulture`. Запрещено полагаться на текущую культуру потока.
- Крупные файлы читаются строго построчно через `ReadLineAsync(CancellationToken)`. Запрещено использовать `ReadToEnd`/`ReadToEndAsync` в основной логике нормализации.
- Поставщик `CodePagesEncodingProvider` регистрируется ровно один раз при старте программы, до любого вызова `GetEncoding`.
- Программа должна принимать `CancellationToken` и корректно завершать работу по `Console.CancelKeyPress`, не оставляя полуписанные файлы.
- Ошибки парсинга отдельных строк не должны ронять всю обработку; они пишутся в отдельный лог с указанием файла и номера строки.
- Код должен компилироваться без предупреждений (`TreatWarningsAsErrors` опционально, но желательно включить `<Nullable>enable</Nullable>` и не иметь `null`-предупреждений).
- Используйте `Path.Combine` для построения путей, не склеивайте строки через `+` или `\` вручную — это некроссплатформенно.

#### Тонкости и подводные камни
- Если вы откроете `StreamWriter` с `Encoding.UTF8` (а не с `new UTF8Encoding(false)`), в начало файла попадёт BOM из трёх байтов `0xEF 0xBB 0xBF`. Это сломает парсинг у программ, которые не ожидают BOM (например, у некоторых CSV-импортеров). Урок явно предостерегает от этой ошибки.
- `StreamReader` без явной кодировки использует UTF-8 и автоматически снимает BOM при чтении — это удобно, но не помогает с `windows-1251`. Для устаревших файлов обязательно регистрируйте `CodePagesEncodingProvider` и передавайте кодировку явно; иначе получите `ArgumentException` или «кракозябры».
- Не забывайте, что `StreamWriter` буферизует данные. Если вы забудете `using`/`Dispose`, последние строки могут остаться в буфере и не попасть на диск — типичная потеря данных, описанная в разделе «Частые ошибки».
- `ReadToEnd()` грузит весь файл в память. Для файла размером в несколько гигабайт это вызовет `OutOfMemoryException`. Урок рекомендует построчный цикл `while ((line = await reader.ReadLineAsync()) is not null)`.
- Параметр `detectEncodingFromByteOrderMarks: true` у `StreamReader` помогает снять BOM при чтении, но он НЕ определяет кодировку файла автоматически — он только съедает BOM, если кодировка уже подходит. Не путайте это с магическим автоопределением.
- `double.Parse` на машине с русской локалью по умолчанию ожидает запятую. Если в файле точка и вы не указали `InvariantCulture`, парсинг упадёт с `FormatException`. Симметрично — `ToString()` без культуры запишет запятую, и файл не откроется в США.
- `Encoding.UTF8` и `new UTF8Encoding(false)` — это разные экземпляры с разным поведением по BOM. Запомните: дефолтный `StreamWriter` и `File.WriteAllText` — без BOM; `Encoding.UTF8` напрямую — с BOM.
- `Append`-режим `StreamWriter` при открытии существующего файла не перезаписывает BOM, если он уже был. Если вы дописываете в UTF-8 файл с BOM, новый текст пойдёт после BOM — обычно это нормально, но будьте внимательны.
- При использовании `await using` с `StreamReader`/`StreamWriter` убедитесь, что метод помечен `async` и возвращает `Task`/`Task<T>`, иначе компилятор выдаст ошибку или предупреждение о `async`-методе без `await`.
- `Directory.EnumerateFiles` ленив: если вы начнёте обрабатывать файлы и параллельно кто-то добавит новый, он может попасть в перечисление. Для детерминированностиmaterialизуйте список через `.ToArray()` или `.ToList()`, если это критично.

#### Критерии приёмки
- [ ] Проект `EncodingNormalizer` создаётся командой `dotnet new console` и собирается без ошибок на .NET 8 / C# 12.
- [ ] Установлен пакет `System.Text.Encoding.CodePages` и вызван `Encoding.RegisterProvider(CodePagesEncodingProvider.Instance)`.
- [ ] Все `StreamReader`/`StreamWriter` обёрнуты в `using` или `await using`.
- [ ] Выходные файлы записываются в UTF-8 без BOM (первые три байта НЕ `0xEF 0xBB 0xBF`).
- [ ] Поддерживается чтение UTF-8, UTF-16 LE/BE (по BOM) и `windows-1251` (по расширению).
- [ ] Большие файлы читаются построчно через `ReadLineAsync(CancellationToken)`, без `ReadToEnd`.
- [ ] Числа парсятся и сериализуются через `CultureInfo.InvariantCulture`.
- [ ] Даты парсятся через `ParseExact` с инвариантной культурой и пишутся в формате ISO-8601 (`O`).
- [ ] Битые строки не роняют обработку; они попадают в `errors.log` с указанием файла и номера строки.
- [ ] Программа принимает `--src` и `--dst` и поддерживает значения по умолчанию `./input` и `./output`.
- [ ] Реализована отмена по `Ctrl+C` через `CancellationTokenSource` и `Console.CancelKeyPress`.
- [ ] В конце работы выводится сводка: обработано файлов, записано строк, зафиксировано ошибок.
- [ ] Тестовый набор из трёх файлов в разных кодировках корректно конвертируется в три UTF-8-файла без BOM.
- [ ] Пустые строки и строки с пробелами обрабатываются корректно (пропускаются или триммятся).
- [ ] Код не содержит `null`-предупреждений компилятора и предупреждений `CA` (по возможности).

#### Подсказки (без прямого ответа)
- Подумайте, какие первые байты у BOM для UTF-8, UTF-16 LE и UTF-16 BE — это «отпечаток пальца» кодировки.
- Для чтения первых байтов подойдёт `using var fs = new FileStream(path, FileMode.Open, FileAccess.Read, FileShare.Read, 4096, FileOptions.Asynchronous);` и `fs.Read(buffer, 0, 3)`.
- Не пытайтесь автоопределить `windows-1251` по содержимому — это ненадёжно; используйте расширение файла или конфигурацию.
- Для парсинга суммы с допуском пробелов и знака используйте `NumberStyles.Float | NumberStyles.AllowThousands` аккуратно — либо простой `double.Parse(s, CultureInfo.InvariantCulture)` после `Trim()`.
- Чтобы передать `CancellationToken` в `ReadLineAsync`, нужна перегрузка с `CancellationToken` (появилась в .NET 7+).
- Для построения выходной строки используйте не `string.Format`, а интерполяцию с инвариантной культурой: `FormattableString.Invariant($"date={date:O};amount={amount};label={label}")`.
- Лог ошибок тоже открывайте в UTF-8 без BOM — иначе сами нарушите собственный канонический формат.

#### Эталонное решение (разбор)
```csharp
// C# 12 / .NET 8 — EncodingNormalizer
// Конвертер текстовых файлов в канонический UTF-8 без BOM.
// Converter of text files into canonical BOM-less UTF-8.

using System.Globalization;
using System.Text;
using System.Text.Encodings.Web; // не используется здесь, но типично для Web; убрано ниже
// (оставлен только реально нужный using — System.Globalization и System.Text)

// --- Регистрация поставщика устаревших кодировок (ровно один раз) ---
// Register the legacy encoding provider exactly once.
Encoding.RegisterProvider(CodePagesEncodingProvider.Instance);

// --- Разбор аргументов командной строки ---
// Parse command-line arguments.
string srcDir = ArgsValue(args, "--src", "./input");
string dstDir = ArgsValue(args, "--dst", "./output");
string errorLogPath = Path.Combine(dstDir, "errors.log");

// --- Отмена по Ctrl+C ---
// Cancellation via Ctrl+C.
using var cts = new CancellationTokenSource();
Console.CancelKeyPress += (_, e) => { e.Cancel = true; cts.Cancel(); };

// --- Сводка результатов ---
// Result summary.
var summary = await NormalizeDirectoryAsync(srcDir, dstDir, errorLogPath, cts.Token);
Console.WriteLine($"Обработано файлов / Files processed: {summary.Files}");
Console.WriteLine($"Записано строк / Lines written: {summary.Lines}");
Console.WriteLine($"Ошибок парсинга / Parse errors: {summary.Errors}");

// === Методы / Methods ===

static string ArgsValue(string[] args, string key, string fallback)
{
    // Простая выборка значения по ключу из args.
    // Simple value lookup by key from args.
    for (int i = 0; i < args.Length - 1; i++)
    {
        if (args[i] == key)
        {
            return args[i + 1];
        }
    }
    return fallback;
}

static Encoding DetectEncoding(string path)
{
    // Читаем первые 3 байта и проверяем BOM.
    // Read the first 3 bytes and check the BOM.
    Span<byte> head = stackalloc byte[3];
    using var fs = new FileStream(path, FileMode.Open, FileAccess.Read,
        FileShare.Read, bufferSize: 4096, FileOptions.Asynchronous);
    int read = fs.Read(head);
    if (read >= 2)
    {
        if (head[0] == 0xFF && head[1] == 0xFE) return Encoding.Unicode;        // UTF-16 LE
        if (head[0] == 0xFE && head[1] == 0xFF) return Encoding.BigEndianUnicode; // UTF-16 BE
    }
    if (read >= 3 && head[0] == 0xEF && head[1] == 0xBB && head[2] == 0xBF)
    {
        return new UTF8Encoding(encoderShouldEmitUTF8Identifier: false); // UTF-8 с BOM → читаем как UTF-8
    }

    // Нет BOM — эвристика по расширению.
    // No BOM — heuristic by extension.
    return Path.GetExtension(path).ToLowerInvariant() switch
    {
        ".1251" => Encoding.GetEncoding("windows-1251"),
        ".utf16" => Encoding.Unicode,
        _ => new UTF8Encoding(encoderShouldEmitUTF8Identifier: false),
    };
}

static async Task<Summary> NormalizeDirectoryAsync(
    string srcDir, string dstDir, string errorLogPath, CancellationToken ct)
{
    Directory.CreateDirectory(dstDir);
    var files = Directory.EnumerateFiles(srcDir, "*", SearchOption.TopDirectoryOnly)
                         .ToArray(); // материализуем, чтобы перечисление было детерминированным

    int totalFiles = 0, totalLines = 0, totalErrors = 0;

    // Лог ошибок — тоже UTF-8 без BOM.
    // Error log — also BOM-less UTF-8.
    await using var errorLog = new StreamWriter(errorLogPath, append: false,
        new UTF8Encoding(encoderShouldEmitUTF8Identifier: false));

    foreach (var src in files)
    {
        var dst = Path.Combine(dstDir,
            Path.GetFileNameWithoutExtension(src) + ".normalized.txt");

        var (lines, errors) = await NormalizeFileAsync(src, dst, errorLog, ct);
        totalFiles++;
        totalLines += lines;
        totalErrors += errors;
        if (ct.IsCancellationRequested) break;
    }

    return new Summary(totalFiles, totalLines, totalErrors);
}

static async Task<(int Lines, int Errors)> NormalizeFileAsync(
    string src, string dst, StreamWriter errorLog, CancellationToken ct)
{
    int lines = 0, errors = 0;
    var enc = DetectEncoding(src);

    // await using гарантирует закрытие и Flush даже при исключении.
    // await using guarantees close and flush even on exception.
    await using var reader = new StreamReader(src, enc, detectEncodingFromByteOrderMarks: true);
    await using var writer = new StreamWriter(dst, append: false,
        new UTF8Encoding(encoderShouldEmitUTF8Identifier: false));

    int lineNo = 0;
    string? line;
    while ((line = await reader.ReadLineAsync(ct)) is not null)
    {
        lineNo++;
        if (string.IsNullOrWhiteSpace(line)) continue; // пустые строки пропускаем

        try
        {
            // Разбор формата date|amount|label через AsSpan — без аллокаций массива.
            // Parse date|amount|label via AsSpan — no array allocation.
            var span = line.AsSpan();
            int p1 = span.IndexOf('|');
            int p2 = span.LastIndexOf('|');
            if (p1 < 0 || p2 <= p1)
            {
                throw new FormatException("Ожидается формат date|amount|label / Expected date|amount|label");
            }

            var dateStr = span[..p1];
            var amountStr = span[(p1 + 1)..p2];
            var label = span[(p2 + 1)..].ToString();

            var date = DateTime.ParseExact(dateStr, "yyyy-MM-dd", CultureInfo.InvariantCulture);
            var amount = double.Parse(amountStr, CultureInfo.InvariantCulture);

            // Каноническая строка через FormattableString.Invariant — инвариантная культура гарантирована.
            // Canonical line via FormattableString.Invariant — invariant culture guaranteed.
            string canonical = FormattableString.Invariant($"date={date:O};amount={amount};label={label}");
            await writer.WriteLineAsync(canonical, ct);
            lines++;
        }
        catch (Exception ex)
        {
            errors++;
            await errorLog.WriteLineAsync($"[{Path.GetFileName(src)}:{lineNo}] {ex.Message}");
        }
    }

    return (lines, errors);
}

// Запись-сводка / Summary record.
record Summary(int Files, int Lines, int Errors);
```

Разбор по строкам. Регистрация `CodePagesEncodingProvider` в начале программы — обязательный шаг из урока: без него `windows-1251` недоступен. `ArgsValue` — простая утилита разбора аргументов; намеренно без тяжёлых библиотек, чтобы фокус остался на I/O. `DetectEncoding` читает первые байты через `stackalloc` (без аллокации массива на куче) и определяет BOM по отпечаткам `0xFF 0xFE` (UTF-16 LE) и `0xEF 0xBB 0xBF` (UTF-8); при отсутствии BOM применяется эвристика по расширению — это честная инженерная практика, потому что автоопределение `windows-1251` по содержимому ненадёжно. В `NormalizeDirectoryAsync` список файлов материализуется через `.ToArray()`, чтобы перечисление было детерминированным (урок предупреждает о ленивости `EnumerateFiles`). Лог ошибок открывается в UTF-8 без BOM через `new UTF8Encoding(false)` — это закрепляет правило «никакого BOM в выходных файлах» из урока. В `NormalizeFileAsync` оба потока обёрнуты в `await using`, что гарантирует Flush и закрытие даже при исключении — ключевая best practice урока. Цикл `while ((line = await reader.ReadLineAsync(ct)) is not null)` — асинхронный построчный разбор из урока, с токеном отмены; `ReadToEnd` намеренно не используется, чтобы не грузить большие файлы в память. Разбор строки идёт через `AsSpan` и `IndexOf('|')` — это экономит аллокации, но концептуально эквивалентно `Split`. Парсинг даты и числа происходит строго через `CultureInfo.InvariantCulture` — это прямое применение правила из урока о межсистемном обмене. Каноническая строка строится через `FormattableString.Invariant`, что гарантирует инвариантную культуру для всего выражения, включая формат даты `:O` (ISO-8601 round-trip). Ошибки парсинга перехватываются, логируются с указанием файла и номера строки, и обработка продолжается — это устойчивость к «грязным» данным. Запись `record Summary` использует синтаксис C# 12. Вся программа демонстрирует одновременное применение всех ключевых тем урока: кодировки, BOM, async I/O, инвариантная культура, `using`/`await using`, построчное чтение и устойчивость к ошибкам.

#### Задания на углубление (бонус)
1. Добавьте автоматическое автоопределение кодировки без BOM с помощью библиотеки `UTF.Unknown` (NuGet `UtfUnknown`) — пусть программа сама угадывает `windows-1251` против UTF-8 по частотам символов. Сравните точность с эвристикой по расширению.
2. Реализуйте параллельную обработку файлов через `Parallel.ForEachAsync` с ограничением `MaxDegreeOfParallelism = Environment.ProcessorCount`. Убедитесь, что `StreamWriter` лога ошибок не страдает от гонок — используйте `SemaphoreSlim` или отдельный `Channel<string>` для сериализации записей в лог.
3. Добавьте параметр `--dry-run`, при котором программа только проверяет, какие кодировки у файлов, и печатает отчёт, не записывая выходные файлы. Это полезно для аудита перед миграцией.
4. Реализуйте потоковую обработку архивов `.zip` через `System.IO.Compression.ZipArchive` в режиме `Read`: читайте текстовые файлы прямо из архива без распаковки на диск, нормализуйте и записывайте в новый архив. Это расширит задачу на работу с `Stream`-ами произвольной природы.

---

## Statement in English / Постановка на английском

#### Context & motivation
You are an engineer in a company that is migrating an old reporting system to a new .NET 8 platform. The legacy system has been accumulating text files in wildly different encodings for over a decade: some reports are stored in the legacy single-byte `windows-1251`, some were exported from Excel as UTF-16 LE with a BOM, and the newest files already arrive in UTF-8 — sometimes with a BOM, sometimes without. The new platform demands a single canonical format: UTF-8 without a BOM, numbers in the invariant culture (decimal point), dates in ISO-8601 form (`O` / round-trip), and one logical record per line. Moreover, individual source files can reach several gigabytes, so they cannot be loaded into memory wholesale — you need streaming, line-by-line parsing. Your task is to write a console utility called `EncodingNormalizer` that accepts a directory of source files, determines each file's encoding from its extension and BOM, reads the data line by line and asynchronously, transforms each line, and writes the result into a new directory in canonical form. The utility must be resilient: malformed numbers and dates must not crash the process; instead, they must go to a separate error log with the line number and file name. This task models a real data-migration scenario and forces you to apply encodings, async I/O, invariant culture, and correct resource management at the same time — that is, every key topic of lesson M10-L03.

#### What to do step by step
1. Create a new .NET 8 console project with the command `dotnet new console -n EncodingNormalizer -o EncodingNormalizer --framework net8.0`. Ensure that `EncodingNormalizer.csproj` has `<Nullable>enable</Nullable>` and `<ImplicitUsings>enable</ImplicitUsings>`.
2. Add support for legacy encodings by installing the package `dotnet add package System.Text.Encoding.CodePages`. Without it, `Encoding.GetEncoding("windows-1251")` will throw `ArgumentException` on platforms that do not support this encoding by default.
3. In the program entry point (top-level statements, `Program.cs`), register the encoding provider by calling `Encoding.RegisterProvider(CodePagesEncodingProvider.Instance);` — this must happen exactly once, before the first call to `GetEncoding`.
4. Implement a method `static Encoding DetectEncoding(string path)` that opens the file, reads the first three bytes, and checks for the UTF-8 BOM (`0xEF 0xBB 0xBF`), the UTF-16 LE BOM (`0xFF 0xFE`), and the UTF-16 BE BOM (`0xFE 0xFF`). If a BOM is present, return the matching encoding via `Encoding.UTF8`, `Encoding.Unicode`, or `Encoding.BigEndianUnicode`. If there is no BOM, use an extension-based heuristic: `.1251` → `windows-1251`, `.utf16` → `Encoding.Unicode`, everything else → UTF-8 without BOM (`new UTF8Encoding(false)`).
5. Implement a method `static async Task NormalizeFileAsync(string src, string dst, StreamWriter errorLog, CancellationToken ct)` that opens the source file with `new StreamReader(src, DetectEncoding(src), detectEncodingFromByteOrderMarks: true)`, wraps it in `await using`, and opens the target file with `await using var out = new StreamWriter(dst, append: false, new UTF8Encoding(false))`. Output is written strictly in BOM-less UTF-8.
6. Each source line has the format `date|amount|label`, for example `2024-03-14|1234.56|Sales report`. When reading, split the line on the `|` character (do not use a heavy `Regex` where `Split` or `AsSpan` is enough), parse the date with `DateTime.ParseExact(..., "yyyy-MM-dd", CultureInfo.InvariantCulture)`, and parse the amount with `double.Parse(..., CultureInfo.InvariantCulture)`. Write a canonical line `date=O;amount=Invariant;label` using interpolation and `CultureInfo.InvariantCulture`, for example `date=2024-03-14T00:00:00.0000000Z;amount=1234.56;label=Sales report`.
7. If parsing the date or amount throws, catch the exception, write a line of the form `[src:line] message` to `errorLog` via `await errorLog.WriteLineAsync(...)`, and continue processing the remaining lines. Do not abort the whole file because of a single bad line.
8. For large files, use the loop `while ((line = await reader.ReadLineAsync(ct)) is not null)`, not `ReadToEndAsync()`. Pass a `CancellationToken` to every async call so that the operation can be aborted with Ctrl+C.
9. Implement a method `static async Task NormalizeDirectoryAsync(string srcDir, string dstDir, string errorLogPath, CancellationToken ct)` that enumerates all files through `Directory.EnumerateFiles(srcDir, "*", SearchOption.TopDirectoryOnly)`, creates the target directory with `Directory.CreateDirectory(dstDir)`, opens a shared error log with `await using StreamWriter errorLog = new(errorLogPath, append: false, new UTF8Encoding(false))`, and calls `NormalizeFileAsync` for each file. The output file name is obtained by replacing the extension with `.normalized.txt`.
10. In `Program.cs`, handle command-line arguments: if `--src` and `--dst` are provided, use them; otherwise, fall back to `./input` and `./output`. Create a `CancellationTokenSource`, subscribe to `Console.CancelKeyPress` for graceful cancellation, and call `NormalizeDirectoryAsync`. At the end, print a summary: how many files were processed, how many lines were written, how many errors were logged.
11. Prepare test data: create an `input` directory and put three files in it — `report-utf8.txt` (UTF-8 without BOM), `legacy-1251.txt` (windows-1251), and `excel-utf16.txt` (UTF-16 LE with BOM). Each file should contain at least five valid lines of the form `date|amount|label` plus one intentionally broken line (for example, `bad|not-a-number|oops`) to exercise the error log.
12. Run the utility with `dotnet run --project EncodingNormalizer -- --src input --dst output`. Verify that the `output` directory contains three `.normalized.txt` files, all in UTF-8 without a BOM (check the first three bytes with a hex viewer or `Get-Content -Encoding Byte -TotalCount 3`), and that `errors.log` contains three parse-error entries.
13. Make sure the program does not crash on empty lines or lines with extra whitespace — use `Trim()` and check `IsNullOrEmpty` before parsing.
14. Compare the line counts in the source and output files: the number of valid lines should match, broken lines should be absent from the output file but present in the log.

#### Requirements
- Target platform: .NET 8, C# 12. Top-level statements, pattern matching (`is not null`, `is { }`), collection expressions, and raw string literals for multi-line templates are all allowed.
- Every reader/writer must be wrapped in `using` or `await using`. No bare `new StreamReader(...)` without disposal.
- Output files must be in UTF-8 without a BOM. Use `new UTF8Encoding(encoderShouldEmitUTF8Identifier: false)` or the default `StreamWriter` constructor — but do NOT use `Encoding.UTF8` directly, because it emits a BOM.
- All numbers and dates must be serialized and parsed only with `CultureInfo.InvariantCulture`. Relying on the current thread culture is forbidden.
- Large files are read strictly line by line via `ReadLineAsync(CancellationToken)`. Using `ReadToEnd`/`ReadToEndAsync` in the main normalization logic is forbidden.
- The `CodePagesEncodingProvider` is registered exactly once at startup, before any `GetEncoding` call.
- The program must accept a `CancellationToken` and shut down gracefully on `Console.CancelKeyPress`, leaving no half-written files.
- Parse errors of individual lines must not abort the whole run; they go to a separate log with the file name and line number.
- The code should compile without warnings (`TreatWarningsAsErrors` is optional but recommended, with `<Nullable>enable</Nullable>` and no null-state warnings).
- Use `Path.Combine` to build paths; do not concatenate with `+` or `\` manually — it is not cross-platform.

#### Pitfalls
- If you open a `StreamWriter` with `Encoding.UTF8` (instead of `new UTF8Encoding(false)`), three BOM bytes `0xEF 0xBB 0xBF` land at the start of the file. This breaks parsing in programs that do not expect a BOM (some CSV importers, for instance). The lesson explicitly warns against this mistake.
- A parameterless `StreamReader` uses UTF-8 and automatically strips the BOM on read — convenient, but useless for `windows-1251`. For legacy files you must register `CodePagesEncodingProvider` and pass the encoding explicitly; otherwise you get `ArgumentException` or mojibake.
- Remember that `StreamWriter` buffers data. If you forget `using`/`Dispose`, the last lines may stay in the buffer and never reach disk — a classic data loss bug described in the "Common Mistakes" section.
- `ReadToEnd()` loads the whole file into memory. For a multi-gigabyte file this will throw `OutOfMemoryException`. The lesson recommends the line-by-line loop `while ((line = await reader.ReadLineAsync()) is not null)`.
- The `detectEncodingFromByteOrderMarks: true` parameter on `StreamReader` helps strip the BOM during reading, but it does NOT auto-detect the file's encoding — it only eats the BOM if the encoding already matches. Do not confuse this with magic autodetection.
- `double.Parse` on a machine with a Russian locale expects a comma by default. If the file uses a dot and you did not specify `InvariantCulture`, parsing throws `FormatException`. Symmetrically, `ToString()` without a culture writes a comma, and the file will not open in the US.
- `Encoding.UTF8` and `new UTF8Encoding(false)` are different instances with different BOM behavior. Remember: the default `StreamWriter` and `File.WriteAllText` are BOM-less; `Encoding.UTF8` used directly emits a BOM.
- The `Append` mode of `StreamWriter` does not overwrite an existing BOM. If you append to a UTF-8 file that already has a BOM, the new text goes after the BOM — usually fine, but be aware.
- When using `await using` with `StreamReader`/`StreamWriter`, make sure the method is `async` and returns `Task`/`Task<T>`, otherwise the compiler emits an error or a warning about an async method without `await`.
- `Directory.EnumerateFiles` is lazy: if someone adds a file while you are iterating, it may appear in the enumeration. For determinism, materialize the list with `.ToArray()` or `.ToList()` when it matters.

#### Acceptance criteria
- [ ] The `EncodingNormalizer` project is created with `dotnet new console` and builds cleanly on .NET 8 / C# 12.
- [ ] The `System.Text.Encoding.CodePages` package is installed and `Encoding.RegisterProvider(CodePagesEncodingProvider.Instance)` is called.
- [ ] Every `StreamReader`/`StreamWriter` is wrapped in `using` or `await using`.
- [ ] Output files are written in UTF-8 without a BOM (the first three bytes are NOT `0xEF 0xBB 0xBF`).
- [ ] Reading supports UTF-8, UTF-16 LE/BE (by BOM), and `windows-1251` (by extension).
- [ ] Large files are read line by line via `ReadLineAsync(CancellationToken)`, without `ReadToEnd`.
- [ ] Numbers are parsed and serialized with `CultureInfo.InvariantCulture`.
- [ ] Dates are parsed with `ParseExact` and the invariant culture and written in ISO-8601 (`O`) format.
- [ ] Broken lines do not abort processing; they go to `errors.log` with the file name and line number.
- [ ] The program accepts `--src` and `--dst` and supports the `./input` / `./output` defaults.
- [ ] Cancellation via `Ctrl+C` is implemented through `CancellationTokenSource` and `Console.CancelKeyPress`.
- [ ] A summary is printed at the end: files processed, lines written, errors logged.
- [ ] The test set of three files in different encodings is correctly converted into three BOM-less UTF-8 files.
- [ ] Empty lines and lines with whitespace are handled correctly (skipped or trimmed).
- [ ] The code has no null-state compiler warnings and, ideally, no `CA` warnings.

#### Hints (no direct answer)
- Think about what the first bytes of the BOM are for UTF-8, UTF-16 LE, and UTF-16 BE — that is the encoding's "fingerprint".
- To read the first bytes, `using var fs = new FileStream(path, FileMode.Open, FileAccess.Read, FileShare.Read, 4096, FileOptions.Asynchronous);` and `fs.Read(buffer, 0, 3)` works well.
- Do not try to auto-detect `windows-1251` from content — it is unreliable; use the file extension or configuration.
- To parse an amount with tolerance for spaces and a sign, use `NumberStyles.Float | NumberStyles.AllowThousands` carefully — or simply `double.Parse(s, CultureInfo.InvariantCulture)` after `Trim()`.
- To pass a `CancellationToken` to `ReadLineAsync`, you need the overload that accepts one (added in .NET 7+).
- To build the output line, use interpolation with the invariant culture via `FormattableString.Invariant($"date={date:O};amount={amount};label={label}")` rather than `string.Format`.
- Open the error log in BOM-less UTF-8 too — otherwise you violate your own canonical format.

#### Reference solution walk-through
```csharp
// C# 12 / .NET 8 — EncodingNormalizer
// Converts text files into canonical BOM-less UTF-8.

using System.Globalization;
using System.Text;

// --- Register the legacy encoding provider exactly once ---
Encoding.RegisterProvider(CodePagesEncodingProvider.Instance);

// --- Parse command-line arguments ---
string srcDir = ArgsValue(args, "--src", "./input");
string dstDir = ArgsValue(args, "--dst", "./output");
string errorLogPath = Path.Combine(dstDir, "errors.log");

// --- Cancellation via Ctrl+C ---
using var cts = new CancellationTokenSource();
Console.CancelKeyPress += (_, e) => { e.Cancel = true; cts.Cancel(); };

// --- Run and print the summary ---
var summary = await NormalizeDirectoryAsync(srcDir, dstDir, errorLogPath, cts.Token);
Console.WriteLine($"Files processed: {summary.Files}");
Console.WriteLine($"Lines written:   {summary.Lines}");
Console.WriteLine($"Parse errors:     {summary.Errors}");

// === Methods ===

static string ArgsValue(string[] args, string key, string fallback)
{
    // Simple value lookup by key from args.
    for (int i = 0; i < args.Length - 1; i++)
    {
        if (args[i] == key)
        {
            return args[i + 1];
        }
    }
    return fallback;
}

static Encoding DetectEncoding(string path)
{
    // Read the first 3 bytes and check the BOM.
    Span<byte> head = stackalloc byte[3];
    using var fs = new FileStream(path, FileMode.Open, FileAccess.Read,
        FileShare.Read, bufferSize: 4096, FileOptions.Asynchronous);
    int read = fs.Read(head);
    if (read >= 2)
    {
        if (head[0] == 0xFF && head[1] == 0xFE) return Encoding.Unicode;            // UTF-16 LE
        if (head[0] == 0xFE && head[1] == 0xFF) return Encoding.BigEndianUnicode;   // UTF-16 BE
    }
    if (read >= 3 && head[0] == 0xEF && head[1] == 0xBB && head[2] == 0xBF)
    {
        return new UTF8Encoding(encoderShouldEmitUTF8Identifier: false); // UTF-8 with BOM → read as UTF-8
    }

    // No BOM — heuristic by extension.
    return Path.GetExtension(path).ToLowerInvariant() switch
    {
        ".1251" => Encoding.GetEncoding("windows-1251"),
        ".utf16" => Encoding.Unicode,
        _ => new UTF8Encoding(encoderShouldEmitUTF8Identifier: false),
    };
}

static async Task<Summary> NormalizeDirectoryAsync(
    string srcDir, string dstDir, string errorLogPath, CancellationToken ct)
{
    Directory.CreateDirectory(dstDir);
    var files = Directory.EnumerateFiles(srcDir, "*", SearchOption.TopDirectoryOnly)
                         .ToArray(); // materialize for deterministic enumeration

    int totalFiles = 0, totalLines = 0, totalErrors = 0;

    // Error log — also BOM-less UTF-8.
    await using var errorLog = new StreamWriter(errorLogPath, append: false,
        new UTF8Encoding(encoderShouldEmitUTF8Identifier: false));

    foreach (var src in files)
    {
        var dst = Path.Combine(dstDir,
            Path.GetFileNameWithoutExtension(src) + ".normalized.txt");

        var (lines, errors) = await NormalizeFileAsync(src, dst, errorLog, ct);
        totalFiles++;
        totalLines += lines;
        totalErrors += errors;
        if (ct.IsCancellationRequested) break;
    }

    return new Summary(totalFiles, totalLines, totalErrors);
}

static async Task<(int Lines, int Errors)> NormalizeFileAsync(
    string src, string dst, StreamWriter errorLog, CancellationToken ct)
{
    int lines = 0, errors = 0;
    var enc = DetectEncoding(src);

    // await using guarantees close and flush even on exception.
    await using var reader = new StreamReader(src, enc, detectEncodingFromByteOrderMarks: true);
    await using var writer = new StreamWriter(dst, append: false,
        new UTF8Encoding(encoderShouldEmitUTF8Identifier: false));

    int lineNo = 0;
    string? line;
    while ((line = await reader.ReadLineAsync(ct)) is not null)
    {
        lineNo++;
        if (string.IsNullOrWhiteSpace(line)) continue; // skip blank lines

        try
        {
            // Parse date|amount|label via AsSpan — no array allocation.
            var span = line.AsSpan();
            int p1 = span.IndexOf('|');
            int p2 = span.LastIndexOf('|');
            if (p1 < 0 || p2 <= p1)
            {
                throw new FormatException("Expected format date|amount|label");
            }

            var dateStr = span[..p1];
            var amountStr = span[(p1 + 1)..p2];
            var label = span[(p2 + 1)..].ToString();

            var date = DateTime.ParseExact(dateStr, "yyyy-MM-dd", CultureInfo.InvariantCulture);
            var amount = double.Parse(amountStr, CultureInfo.InvariantCulture);

            // Canonical line via FormattableString.Invariant — invariant culture guaranteed.
            string canonical = FormattableString.Invariant($"date={date:O};amount={amount};label={label}");
            await writer.WriteLineAsync(canonical, ct);
            lines++;
        }
        catch (Exception ex)
        {
            errors++;
            await errorLog.WriteLineAsync($"[{Path.GetFileName(src)}:{lineNo}] {ex.Message}");
        }
    }

    return (lines, errors);
}

// Summary record.
record Summary(int Files, int Lines, int Errors);
```

Walk-through. Registering `CodePagesEncodingProvider` at the start is the mandatory step from the lesson: without it, `windows-1251` is unavailable. `ArgsValue` is a minimal argument parser — intentionally lightweight so the focus stays on I/O. `DetectEncoding` reads the first bytes through `stackalloc` (no heap array allocation) and identifies the BOM by the fingerprints `0xFF 0xFE` (UTF-16 LE) and `0xEF 0xBB 0xBF` (UTF-8); when there is no BOM, an extension-based heuristic kicks in — honest engineering practice, because auto-detecting `windows-1251` from content is unreliable. In `NormalizeDirectoryAsync`, the file list is materialized via `.ToArray()` so the enumeration is deterministic (the lesson warns about the laziness of `EnumerateFiles`). The error log is opened in BOM-less UTF-8 via `new UTF8Encoding(false)` — reinforcing the lesson's "no BOM in output files" rule. In `NormalizeFileAsync`, both streams are wrapped in `await using`, which guarantees flush and close even on exception — the key best practice from the lesson. The loop `while ((line = await reader.ReadLineAsync(ct)) is not null)` is the async line-by-line reading pattern from the lesson, with a cancellation token; `ReadToEnd` is deliberately avoided so that large files do not flood memory. The line is parsed via `AsSpan` and `IndexOf('|')` — saving allocations, though conceptually equivalent to `Split`. Parsing the date and amount happens strictly through `CultureInfo.InvariantCulture` — a direct application of the lesson's cross-system-exchange rule. The canonical line is built through `FormattableString.Invariant`, which guarantees invariant culture for the whole expression, including the `:O` date format (ISO-8601 round-trip). Parse errors are caught, logged with the file name and line number, and processing continues — resilience to dirty data. The `record Summary` uses C# 12 syntax. The whole program demonstrates every key topic of the lesson simultaneously: encodings, BOM, async I/O, invariant culture, `using`/`await using`, line-by-line reading, and error resilience.

#### Going deeper (bonus)
1. Add automatic encoding detection for BOM-less files using the `UtfUnknown` library (NuGet `UtfUnknown`) — let the program guess `windows-1251` versus UTF-8 from character frequencies. Compare the accuracy with the extension heuristic.
2. Implement parallel file processing via `Parallel.ForEachAsync` with `MaxDegreeOfParallelism = Environment.ProcessorCount`. Make sure the error-log `StreamWriter` does not suffer from races — use a `SemaphoreSlim` or a separate `Channel<string>` to serialize log writes.
3. Add a `--dry-run` flag under which the program only detects the encoding of each file and prints a report without writing any output files. This is useful for auditing before a migration.
4. Implement streaming processing of `.zip` archives via `System.IO.Compression.ZipArchive` in `Read` mode: read text files directly from the archive without extracting to disk, normalize them, and write them into a new archive. This extends the task to working with streams of arbitrary nature.

---

#### Чек-лист сдачи / Submission checklist
- [ ] (RU) Проект `EncodingNormalizer` собирается на .NET 8 / C# 12 без ошибок и предупреждений.
- [ ] (RU) Установлен пакет `System.Text.Encoding.CodePages`, поставщик зарегистрирован.
- [ ] (RU) Все reader/writer обёрнуты в `using` / `await using`.
- [ ] (RU) Выходные файлы — UTF-8 без BOM.
- [ ] (RU) Поддержаны кодировки UTF-8, UTF-16 LE/BE, windows-1251.
- [ ] (RU) Большие файлы читаются через `ReadLineAsync(ct)`.
- [ ] (RU) Числа и даты используют `CultureInfo.InvariantCulture`.
- [ ] (RU) Битые строки логируются, обработка продолжается.
- [ ] (RU) Реализована отмена по Ctrl+C и сводка в конце.
- [ ] (RU) Тест из трёх файлов в разных кодировках проходит успешно.
- [ ] (EN) The `EncodingNormalizer` project builds on .NET 8 / C# 12 with no errors or warnings.
- [ ] (EN) `System.Text.Encoding.CodePages` is installed and the provider is registered.
- [ ] (EN) Every reader/writer is wrapped in `using` / `await using`.
- [ ] (EN) Output files are BOM-less UTF-8.
- [ ] (EN) UTF-8, UTF-16 LE/BE, and windows-1251 are supported.
- [ ] (EN) Large files are read via `ReadLineAsync(ct)`.
- [ ] (EN) Numbers and dates use `CultureInfo.InvariantCulture`.
- [ ] (EN) Broken lines are logged and processing continues.
- [ ] (EN) Ctrl+C cancellation and a final summary are implemented.
- [ ] (EN) The three-file, multi-encoding test passes successfully.

#### Ресурсы / Resources
- [Microsoft Learn — StreamReader](https://learn.microsoft.com/dotnet/api/system.io.streamreader)
- [Microsoft Learn — StreamWriter](https://learn.microsoft.com/dotnet/api/system.io.streamwriter)
- [Microsoft Learn — System.Text.Encoding](https://learn.microsoft.com/dotnet/api/system.text.encoding)
- [Microsoft Learn — CodePagesEncodingProvider](https://learn.microsoft.com/dotnet/api/system.text.codepagesencodingprovider)
- [Microsoft Learn — CultureInfo.InvariantCulture](https://learn.microsoft.com/dotnet/api/system.globalization.cultureinfo.invariantculture)
- [Microsoft Learn — UTF8Encoding](https://learn.microsoft.com/dotnet/api/system.text.utf8encoding)
- [Microsoft Learn — File and Stream I/O](https://learn.microsoft.com/dotnet/standard/io/)
