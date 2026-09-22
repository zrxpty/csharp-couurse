[⬆ К модулю M10](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M10-L03: StreamReader/StreamWriter, кодировки / StreamReader/StreamWriter, encodings

**Модуль / Module:** M10
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

`StreamReader` и `StreamWriter` — это классы из пространства имён `System.IO`, которые превращают «сырой» поток байтов (`Stream`) в текст. Базовый `Stream` работает с байтами — он ничего не знает ни про буквы, ни про строки. Чтобы прочитать текст, нужно сказать платформе две вещи: **из каких байтов какая буква получается** (кодировка) и **где заканчивается одна строка и начинается другая**. Этим и занимаются `StreamReader`/`StreamWriter`.

**Аналогия.** Представь, что `Stream` — это конвейер с коробками, в каждой из которых лежит число от 0 до 255. `StreamReader` — это рабочий, который берёт несколько коробок подряд, складывает из них одну букву по заранее согласованному «рецепту» (кодировке) и подаёт тебе готовую строку. `StreamWriter` делает обратное: берёт твою строку, раскладывает её в коробки с числами и ставит на конвейер.

**Кодировки.** В .NET кодировка — это класс, унаследованный от `System.Text.Encoding`. Самая важная кодировка сегодня — **UTF-8**: именно её .NET использует по умолчанию, и именно она рекомендована для всех новых файлов. UTF-8 умеет кодировать любой символ Unicode (от латиницы до эмодзи), при этом ASCII-символы занимают 1 байт, а кириллица — 2 байта. Другие часто встречающиеся кодировки: `Encoding.Unicode` (UTF-16 LE, по 2 байта на символ), `Encoding.UTF32` (4 байта на символ — расточительно), `Encoding.ASCII` (только 7 бит, кириллицу не поддерживает), `Encoding.GetEncoding("windows-1251")` (устаревшая однобайтная кириллическая кодировка, встречается в старых файлах).

**BOM (Byte Order Mark).** В начало UTF-8/UTF-16 файла может быть записан специальный маркер — BOM. Для UTF-8 это байты `0xEF 0xBB 0xBF`. BOM помогает приложению-читателю понять: «это UTF-8». В .NET `Encoding.UTF8` по умолчанию пишет BOM, а `new UTF8Encoding(false)` и новый `Encoding.UTF8` в API файловых операций — **не пишет**. Поэтому `File.ReadAllText`/`File.WriteAllText` и конструкторы `StreamReader`/`StreamWriter` без явной кодировки создают файлы **без BOM**, что обычно и нужно. BOM полезен при обмене данными со старыми программами, но мешает, например, когда файл подключается как конфигурация или парсится инструментами, не ожидающими лишних байтов.

**Чтение.** `ReadLine()` читает одну строку до `\n` или `\r\n` и возвращает её без разделителя; `ReadToEnd()` читает весь файл целиком в одну строку; `Read()` — один символ. Есть асинхронные версии: `ReadLineAsync()`, `ReadToEndAsync()`, `ReadAsync()`, возвращающие `Task<string?>`. Для больших файлов лучше читать построчно в цикле `while ((line = await reader.ReadLineAsync()) is not null)` — это не загружает весь файл в память.

**Запись.** `Write()` и `WriteLine()` добавляют текст во внутренний буфер. Данные сбрасываются на диск при вызове `Flush()`, при `Dispose()` (через `using`) или при заполнении буфера. `AutoFlush = true` отключает буферизацию — удобно для логов, когда каждая запись должна немедленно попадать в файл.

**Culture.** Формат чисел и дат при преобразовании в строку зависит от `CultureInfo`. Если ты пишешь файл, который потом будут парсить другой программой на другой локали, всегда явно указывай инвариантную культуру: `value.ToString(CultureInfo.InvariantCulture)` и `double.Parse(s, CultureInfo.InvariantCulture)`. Иначе запятая/точка в десятичных дробях или формат даты сломают чтение.

**`using` и очистка.** Всегда оборачивай reader/writer в `using` (или `await using` для асинхронного dispose) — это гарантирует закрытие файла и сброс буфера даже при исключении. Незакрытый `StreamWriter` может оставить данные в буфере и потерять часть текста.

#### Theory (EN)

`StreamReader` and `StreamWriter` are classes in the `System.IO` namespace that turn a raw byte `Stream` into text. A base `Stream` works with bytes only — it knows nothing about letters or strings. To read text you must tell the platform two things: **which bytes map to which characters** (the encoding) and **where one line ends and the next begins**. That is exactly what `StreamReader`/`StreamWriter` do.

**Analogy.** Think of a `Stream` as a conveyor belt carrying boxes, each box holding one number from 0 to 255. `StreamReader` is the worker who takes several boxes in a row, assembles them into a single character using an agreed recipe (the encoding), and hands you a finished string. `StreamWriter` does the reverse: it takes your string, breaks it into numbered boxes, and puts them on the belt.

**Encodings.** In .NET an encoding is a class deriving from `System.Text.Encoding`. The most important one today is **UTF-8**: it is .NET's default and the recommended choice for every new file. UTF-8 can encode any Unicode character (from Latin letters to emoji); ASCII characters take 1 byte, Cyrillic takes 2 bytes. Other common encodings: `Encoding.Unicode` (UTF-16 LE, 2 bytes per char), `Encoding.UTF32` (4 bytes — wasteful), `Encoding.ASCII` (7 bits only, no Cyrillic), `Encoding.GetEncoding("windows-1251")` (legacy single-byte Cyrillic, found in old files).

**BOM (Byte Order Mark).** A file in UTF-8/UTF-16 may start with a special marker — the BOM. For UTF-8 these are the bytes `0xEF 0xBB 0xBF`. The BOM helps the reader application recognise "this is UTF-8". In .NET `Encoding.UTF8` emits a BOM by default, while `new UTF8Encoding(false)` and the file APIs do **not**. That is why `File.ReadAllText`/`File.WriteAllText` and the parameterless `StreamReader`/`StreamWriter` constructors produce files **without a BOM**, which is usually what you want. A BOM helps interop with older programs but gets in the way when a file is included as configuration or parsed by tools that do not expect extra bytes.

**Reading.** `ReadLine()` reads one line up to `\n` or `\r\n` and returns it without the delimiter; `ReadToEnd()` reads the whole file into a single string; `Read()` reads one character. There are async versions: `ReadLineAsync()`, `ReadToEndAsync()`, `ReadAsync()`, returning `Task<string?>`. For large files prefer the loop `while ((line = await reader.ReadLineAsync()) is not null)` — it does not load the entire file into memory.

**Writing.** `Write()` and `WriteLine()` append text to an internal buffer. Data is flushed to disk on `Flush()`, on `Dispose()` (via `using`), or when the buffer fills. `AutoFlush = true` disables buffering — handy for logs, where every record must reach the file immediately.

**Culture.** Number and date formatting depends on `CultureInfo`. When you write a file that another program on another locale will later parse, always specify the invariant culture explicitly: `value.ToString(CultureInfo.InvariantCulture)` and `double.Parse(s, CultureInfo.InvariantCulture)`. Otherwise the decimal separator (comma vs dot) or date format will break parsing.

**`using` and cleanup.** Always wrap a reader/writer in `using` (or `await using` for async dispose) — it guarantees the file is closed and the buffer flushed even on exception. A `StreamWriter` that is never closed may keep data in its buffer and lose part of the text.

#### Пример кода / Code Example

```csharp
// C# 12 / .NET 8 — StreamReader/StreamWriter, кодировки, async, culture
// Чтение и запись текстовых файлов с явным контролем кодировки и культуры.

using System.Globalization;
using System.Text;

// --- Запись файла в UTF-8 без BOM (по умолчанию для StreamWriter) ---
// Write a UTF-8 file without BOM (the default for StreamWriter).
var path = Path.Combine(Path.GetTempPath(), "m10-l03-demo.txt");

// using гарантирует Flush + закрытие файла даже при исключении.
// using guarantees Flush + file close even on exception.
using (var writer = new StreamWriter(path, append: false))
{
    writer.WriteLine("Привет, мир! / Hello, world!");
    // Инвариантная культура — чтобы число парсилось на любой локали.
    // Invariant culture so the number parses on any locale.
    writer.WriteLine(3.14.ToString(CultureInfo.InvariantCulture));
    writer.WriteLine(DateTime.UtcNow.ToString("O", CultureInfo.InvariantCulture));
}

// --- Чтение построчно, асинхронно, с явной кодировкой UTF-8 ---
// Read line by line, async, with explicit UTF-8 encoding (no BOM on read).
var utf8NoBom = new UTF8Encoding(encoderShouldEmitUTF8Identifier: false);
using (var reader = new StreamReader(path, utf8NoBom, detectEncodingFromByteOrderMarks: true))
{
    string? line;
    while ((line = await reader.ReadLineAsync()) is not null)
    {
        Console.WriteLine(line);
    }
}

// --- Весь файл сразу в одну строку ---
// The whole file at once into a single string.
string allText = await File.ReadAllTextAsync(path);
Console.WriteLine($"Всего символов / Total chars: {allText.Length}");

// --- Дописать в существующий файл (append) с автосбросом для лога ---
// Append to an existing file with AutoFlush for a log.
await using (var log = new StreamWriter(path, append: true, Encoding.UTF8))
{
    log.AutoFlush = true; // каждая строка сразу попадает на диск / each line reaches disk at once
    await log.WriteLineAsync($"Лог-запись / Log entry: {DateTime.UtcNow:O}");
}

// --- Парсинг чисел из файла с инвариантной культурой ---
// Parse numbers from the file with invariant culture.
using var parser = new StreamReader(path);
while ((parser.ReadLine()) is { } raw)
{
    if (double.TryParse(raw, NumberStyles.Float, CultureInfo.InvariantCulture, out var d))
    {
        Console.WriteLine($"Распарсено число / Parsed number: {d}");
    }
}

// --- Работа с устаревшей кодировкой windows-1251 (требует System.Text.Encoding.CodePages) ---
// Legacy windows-1251 encoding (requires System.Text.Encoding.CodePages).
Encoding.RegisterProvider(CodePagesEncodingProvider.Instance); // регистрируем поставщик / register provider
var win1251 = Encoding.GetEncoding("windows-1251");
string legacy = "Текст в кодировке windows-1251";
byte[] legacyBytes = win1251.GetBytes(legacy);
string restored = win1251.GetString(legacyBytes);
Console.WriteLine($"Legacy restored: {restored}");
```

#### Best Practices

- Всегда оборачивай `StreamReader`/`StreamWriter` в `using` (или `await using`), чтобы гарантированно закрыть файл и сбросить буфер.
- Используй UTF-8 без BOM как кодировку по умолчанию для всех новых текстовых файлов.
- Для больших файлов читай построчно через `ReadLineAsync()` в цикле, а не `ReadToEndAsync()` целиком — экономь память.
- Числа и даты записывай и парси только через `CultureInfo.InvariantCulture`, если файл предназначен для межсистемного обмена.
- Явно указывай кодировку при работе с унаследованными файлами (windows-1251, UTF-16) — не полагайся на автоопределение.

- Always wrap `StreamReader`/`StreamWriter` in `using` (or `await using`) to guarantee the file is closed and the buffer flushed.
- Use UTF-8 without BOM as the default encoding for every new text file.
- For large files, read line by line with `ReadLineAsync()` in a loop rather than `ReadToEndAsync()` at once — save memory.
- Write and parse numbers and dates only with `CultureInfo.InvariantCulture` when the file is meant for cross-system exchange.
- Specify the encoding explicitly when dealing with legacy files (windows-1251, UTF-16) — do not rely on autodetection.

#### Частые ошибки / Common Mistakes

- Не закрываешь `StreamWriter` (забыл `using`) → часть текста остаётся в буфере и теряется. Всегда используй `using`.
- Открываешь файл с `Encoding.UTF8` и получаешь лишний BOM, ломающий парсинг. Бери `new UTF8Encoding(false)` или дефолтный конструктор.
- Читаешь весь файл через `ReadToEnd()` в память — для гигабайтного лога это OOM. Используй `ReadLineAsync()` в цикле.
- Записываешь `double` через `ToString()` без культуры → на машине с запятой-разделителем файл не распарсится в США. Указывай `InvariantCulture`.
- Считаешь, что `StreamReader` без параметров «как-то сам» угадает кодировку старого windows-1251 файла. Не угадает — регистрируй `CodePagesEncodingProvider` и передавай кодировку явно.

- Leaving `StreamWriter` unclosed (forgot `using`) → part of the text stays in the buffer and is lost. Always use `using`.
- Opening a file with `Encoding.UTF8` and getting an unwanted BOM that breaks parsing. Use `new UTF8Encoding(false)` or the default constructor.
- Reading the whole file with `ReadToEnd()` into memory — for a gigabyte log this is OOM. Use `ReadLineAsync()` in a loop.
- Writing a `double` via `ToString()` without a culture → on a machine with a comma separator the file will not parse in the US. Specify `InvariantCulture`.
- Assuming the parameterless `StreamReader` will somehow "guess" the encoding of an old windows-1251 file. It will not — register `CodePagesEncodingProvider` and pass the encoding explicitly.

#### Чек-лист самопроверки / Self-check Checklist

- [ ] Я оборачиваю reader/writer в `using` / `await using`.
- [ ] Я знаю, что UTF-8 — кодировка по умолчанию и что она не пишет BOM в файловых API.
- [ ] Я могу объяснить, чем отличается `ReadLine` от `ReadToEnd` и когда применять каждый.
- [ ] Я использую `ReadLineAsync()` для больших файлов, чтобы не грузить всё в память.
- [ ] Я указываю `CultureInfo.InvariantCulture` для чисел и дат в межсистемных файлах.
- [ ] Я умею читать устаревшие кодировки через `CodePagesEncodingProvider`.

- [ ] I wrap reader/writer in `using` / `await using`.
- [ ] I know that UTF-8 is the default encoding and that the file APIs do not emit a BOM.
- [ ] I can explain the difference between `ReadLine` and `ReadToEnd` and when to use each.
- [ ] I use `ReadLineAsync()` for large files to avoid loading everything into memory.
- [ ] I specify `CultureInfo.InvariantCulture` for numbers and dates in cross-system files.
- [ ] I can read legacy encodings via `CodePagesEncodingProvider`.

#### Ресурсы / Resources

- [Microsoft Learn — https://learn.microsoft.com/dotnet/api/system.io.streamreader](https://learn.microsoft.com/dotnet/api/system.io.streamreader)
- [Microsoft Learn — StreamReader](https://learn.microsoft.com/dotnet/api/system.io.streamwriter)
- [Microsoft Learn — System.Text.Encoding](https://learn.microsoft.com/dotnet/api/system.text.encoding)

---

[⬆ К модулю M10](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
