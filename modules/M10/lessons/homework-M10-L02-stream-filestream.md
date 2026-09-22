---
[← К уроку M10-L02](lesson-M10-L02-stream-filestream.md) | [⬆ К модулю M10](../README.md) | [Следующее ДЗ →](homework-M10-L03-streamreader-writer-encoding.md)
---

### Домашнее задание M10-L02: Stream, FileStream, буферизация / Homework M10-L02: Stream, FileStream, buffering

**Урок / Lesson:** M10-L02
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 4/5
**Цель / Goal:** (RU) Научиться открывать файлы через `FileStream` с `FileStreamOptions`, осознанно управлять буфером и режимом `FileOptions.Asynchronous`, корректно освобождать ресурсы через `await using`, проверять количество прочитанных байтов, позиционироваться через `Seek`/`Position` и компоновать декораторы (`BufferedStream`). (EN) Learn to open files through `FileStream` with `FileStreamOptions`, deliberately manage the buffer and the `FileOptions.Asynchronous` flag, correctly release resources via `await using`, validate the byte count returned by reads, position with `Seek`/`Position`, and compose decorator streams (`BufferedStream`).

#### Связь с уроком / Connection to the lesson
(RU) Урок вводит `Stream` как «трубу» с единым API и показывает `FileStream` как её файловую реализацию с буфером. Домашнее задание закрепляет всё это на практике: вы пишете утилиту копирования с измерением производительности, демонстрируете случайный доступ и собираете цепочку декораторов, чтобы увидеть компонуемость потоков своими глазами. (EN) The lesson introduces `Stream` as a "pipe" with a unified API and shows `FileStream` as its file-backed implementation with a buffer. This homework fixes all of that in practice: you write a measured copy utility, demonstrate random access, and assemble a decorator chain to see stream composability with your own eyes.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация

Вы — инженер в команде, которая разрабатывает утилиту резервного копирования небольших файлов конфигурации и логов. Команда хочет единый, честный и предсказуемый механизм копирования, который одинаково хорошо работает и на SSD-ноутбуке разработчика, и на медленном сетевом диске, и на классическом жёстком диске с вращающимися пластинами. Важно не просто «скопировать байты», а понять, где именно тратится время: в системных вызовах, в ожидании диска или в неоптимальном размере буфера. Утилита должна честно измерять производительность и выводить диагностику, по которой инженер может принять решение о размере буфера и режиме доступа.

В предыдущем уроке (M10-L01) вы работали с `File` и `Path` — высокоуровневыми хелперами, которые пряча детали. Теперь вы опускаетесь на уровень `Stream` и `FileStream`, чтобы осознанно управлять файловым дескриптором, буфером и режимом асинхронности. Это критический навык: вся последующая работа с `StreamReader`/`StreamWriter`, `GZipStream`, `CryptoStream` и сетевыми потоками строится на понимании базового `Stream`. Если вы научитесь грамотно открывать `FileStream` с правильными опциями, корректно освобождать его и проверять возвращаемое количество байтов, то любая более высокоуровневая обёртка станет для вас простой композицией декораторов поверх одной и той же абстракции.

#### Что нужно сделать (пошагово)

1. Создайте новый консольный проект .NET 8 с именем `StreamLab`:
   ```bash
   dotnet new console -n StreamLab -o StreamLab --framework net8.0
   cd StreamLab
   dotnet new sln --name StreamLab
   dotnet sln add StreamLab.csproj
   ```
   Включите в `.csproj` настройки `<Nullable>enable</Nullable>` и `<ImplicitUsings>enable</ImplicitUsings>`, используйте C# 12 (`<LangVersion>12</LangVersion>`).

2. В `Program.cs` реализуйте top-level приложение с тремя командами: `copy`, `seek`, `chain`. Команда определяется первым аргументом `args[0]`. Если аргументов нет — выведите справку.

3. Команда `copy <src> <dst>` должна копировать файл через `FileStream` с `FileStreamOptions`. Обязательно: `Mode = FileMode.Open`/`Create`, `Access = FileAccess.Read`/`Write`, `Share = FileShare.Read`, `BufferSize` configurable через переменную окружения `BUFFER_KB` (по умолчанию 64 КБ), `Options = FileOptions.Asynchronous`. Используйте `await using` для обоих потоков. Читайте в цикле `byte[] buffer` размером с `BufferSize`, проверяйте возвращаемое значение `await fs.ReadAsync(buffer, ...)` — оно может быть меньше запрошенного. Пишите ровно `read` байтов: `await dst.WriteAsync(buffer, 0, read)`.

4. Замерьте производительность через `Stopwatch`: выведите в консоль размер файла, время копирования в миллисекундах, пропускную способность в МБ/с и количество итераций цикла чтения. Запустите копирование одного и того же файла (создайте тестовый файл размером 50 МБ через `File.WriteAllBytes` или цикл `WriteAsync`) с буферами 4 КБ, 64 КБ и 1 МБ и сравните результаты. Запишите наблюдения в комментарий в коде.

5. Команда `seek <file> <offset>` должна открыть файл с `FileAccess.Read` и `FileShare.Read`, переместиться на указанный `offset` через `fs.Seek(offset, SeekOrigin.Begin)` и прочитать 16 байтов. Выведите их в hex и как ASCII (заменяя непечатные на `.`). Дополнительно покажите `fs.Position` до и после чтения, чтобы продемонстрировать, что `ReadAsync` двигает позицию.

6. Команда `chain <text>` должна продемонстрировать компонуемость декораторов: создайте `MemoryStream`, оберните его в `BufferedStream` с буфером 8 КБ, запишите `text` (UTF-8), вызовите `FlushAsync`, перемотайте в начало через `Position = 0` и прочитайте всё обратно. Выведите прочитанный текст. Затем соберите цепочку из трёх слоёв: `BufferedStream` поверх `MemoryStream` — это минимум; бонусом попробуйте обернуть ещё одним `BufferedStream`, чтобы показать, что декораторы вкладываются друг в друга без ошибок.

7. После каждого вызова `FlushAsync` для декоратора явно вызовите его `DisposeAsync` (через `await using`) раньше базового потока, чтобы убедиться, что последние байты действительно сброшены в нижележащий поток. Обратите внимание на порядок закрытия: декоратор закрывается первым, базовый — вторым.

8. Скомпилируйте и запустите:
   ```bash
   dotnet run -- copy sample.bin sample-copy.bin
   dotnet run -- seek sample.bin 100
   dotnet run -- chain "Hello, decorator world!"
   ```
   Убедитесь, что `sample-copy.bin` побайтово совпадает с `sample.bin` (можно сверить через `System.Security.Cryptography.SHA256`).

#### Требования к решению

- Целевая платформа: .NET 8, C# 12. Используйте top-level statements, file-scoped namespaces, collection expressions и pattern matching там, где это уместно (например, `args` switch expression для разбора команды).
- Каждый `FileStream` обязан создаваться через `FileStreamOptions` с явными `Mode`, `Access`, `Share`, `BufferSize` и `Options`. Никаких устаревших конструкторов вида `new FileStream(path, FileMode.Create)` без опций.
- Каждый открытый поток обязан быть обёрнут в `using` или `await using`. Для потоков с `FileOptions.Asynchronous` — обязательно `await using`. Никаких «голых» `new FileStream` без освобождения.
- Асинхронные операции (`ReadAsync`/`WriteAsync`) вызываются только на потоках, открытых с `FileOptions.Asynchronous`. Если поток синхронный — используйте синхронный `Read`/`Write` или предупреждайте в комментарии.
- В цикле копирования обязательно проверяется количество прочитанных байтов: переменная `read` может быть меньше `buffer.Length`. Никогда не пишите `buffer.Length` вместо `read` в `WriteAsync`.
- Код должен быть устойчивым к пустым и очень маленьким файлам, а также к файлам размером больше 2 ГБ (используйте `long`, а не `int`, для размеров и смещений).
- Все исключения ввода-вывода должны быть перехвачены и логированы с понятным сообщением и именем файла. Не «проглатывайте» исключения молча.

#### Тонкости и подводные камни

- `ReadAsync` не обязан вернуть ровно столько байтов, сколько вы запросили. Для файловых потоков обычно возвращает полный буфер, но это не контракт `Stream` — полагаться на это нельзя. В цикле копирования условие выхода — `read == 0`, а не `read < buffer.Length`.
- Без `FileOptions.Asynchronous` вызовы `ReadAsync`/`WriteAsync` под капотом используют синхронный I/O и блокируют поток пула через «overlapped»-эмуляцию. Для настоящей асинхронности через IOCP флаг обязателен. Урок прямо это подчёркивает.
- `Flush` на `FileStream` сбрасывает внутренний буфер в ядро ОС, но не гарантирует запись на физическое устройство. Если нужна durability (например, журнал транзакций) — вызывайте `fs.Flush(flushToDisk: true)`, что под капотом зовёт `FlushFileBuffers`. Это дорого, но необходимо для критичных данных.
- `Position` и `Seek` игнорируются при `FileMode.Append` — позиция всегда «в конец». Не пытайтесь дописывать файл с предварительным `Seek` — это не сработает. Для дописывания просто пишите.
- `BufferedStream` поверх `MemoryStream` не ускоряет работу (память и так быстрая), но полезен как демонстрация декоратора. Для реальных файлов `BufferedStream` поверх не буферизованного `FileStream` (с `BufferSize = 0`, если бы он был возможен) давал бы смысл, но в .NET `FileStream` уже буферизован по умолчанию, так что добавочный `BufferedStream` чаще избыточен.
- Не используйте поток после `Dispose`/`DisposeAsync` — будет `ObjectDisposedException`. Особенно коварно в коде с ветвлениями, где одна ветка закрыла поток раньше другой.
- Закрывайте декораторы раньше базовых потоков: при `await using` декоратор должен быть объявлен «снаружи» базового, чтобы порядок освобождения был корректным (декоратор `DisposeAsync` → базовый `DisposeAsync`). Иначе последние байты декоратора не допишутся.
- `FileShare.Read` разрешает другим процессам читать, но не писать. `FileShare.None` полностью блокирует файл. Выбирайте под сценарий: для копирования источника достаточно `FileShare.Read`, для назначения — `FileShare.None`.

#### Критерии приёмки

- [ ] Проект `StreamLab` собирается без предупреждений через `dotnet build -warnaserror`.
- [ ] Все `FileStream` создаются через `FileStreamOptions` с явными `Mode`, `Access`, `Share`, `BufferSize`, `Options`.
- [ ] Все потоки обёрнуты в `using` или `await using`; для `FileOptions.Asynchronous` — именно `await using`.
- [ ] В цикле копирования переменная `read` проверяется корректно: выход при `read == 0`, запись ровно `read` байтов.
- [ ] Команда `copy` работает для файлов 0 байт, 1 байт, 50 МБ и больше 2 ГБ без ошибок и без переполнения `int`.
- [ ] SHA256 копии совпадает с SHA256 оригинала для всех тестовых размеров.
- [ ] Команда `seek` выводит hex и ASCII 16 байтов, корректно преобразует непечатные символы в `.`.
- [ ] Команда `seek` показывает `Position` до и после `ReadAsync` и подтверждает движение позиции.
- [ ] Команда `chain` успешно собирает `BufferedStream` поверх `MemoryStream`, пишет, перематывает, читает и выводит исходный текст.
- [ ] Декоратор закрывается раньше базового потока (корректный порядок `await using`).
- [ ] Команда `copy` замеряет и выводит время, пропускную способность в МБ/с и количество итераций.
- [ ] Сравнение буферов 4 КБ / 64 КБ / 1 МБ выполнено и кратко прокомментировано в коде.
- [ ] Разбор команды реализован через pattern matching / switch expression C# 12.
- [ ] Исключения ввода-вывода логируются с именем файла, без «проглатывания».
- [ ] В коде нет `TODO`, нет закомментированного мусора, нет голых `new FileStream` без `using`.

#### Подсказки (без прямого ответа)

- Для сверки файлов используйте `SHA256.HashData` синхронно на `ReadOnlySpan<byte>` из `File.ReadAllBytes` — для тестов это допустимо; для больших файлов стримьте через `SHA256.Create()` и `CopyToAsync`.
- Переменную окружения читайте через `Environment.GetEnvironmentVariable("BUFFER_KB")` и парсите через `int.TryParse`; по умолчанию — 64.
- Для hex используйте `Convert.ToHexString(byte[])` (доступно в .NET 8), для ASCII — ручной цикл с проверкой `>= 32 && < 127`.
- В switch expression по `args` учитывайте случай `args.Length == 0` и печатайте справку через raw string literal `"""..."""`.
- Для генерации тестового файла 50 МБ не создавайте байты случайными по одному — используйте `Random.Shared.GetItems` или предварительно сгенерированный массив, записанный в цикле через `WriteAsync`.

#### Эталонное решение (разбор)

```csharp
// C# 12 / .NET 8 — Stream, FileStream, buffering — эталон StreamLab.
// Демонстрируем FileStreamOptions, FileOptions.Asynchronous, await using,
// проверку read-счёта, Seek/Position, компоновку декораторов.
// Demonstrates FileStreamOptions, true async, await using, read-count check,
// Seek/Position, decorator composition.

using System.Diagnostics;
using System.Security.Cryptography;
using System.Text;

const int DefaultBufferKb = 64;

return args.Length switch
{
    0 => PrintHelp(),
    _ => args[0].ToLowerInvariant() switch
    {
        "copy"  => await CopyAsync(args[1], args[2]),
        "seek"  => Seek(args[1], long.Parse(args[2])),
        "chain" => await ChainAsync(args.Length > 1 ? args[1] : "decorator demo"),
        _ => PrintHelp()
    }
};

static int PrintHelp()
{
    Console.WriteLine("""
        StreamLab — usage:
          copy <src> <dst>        Copy file with measured buffered FileStream.
          seek <file> <offset>    Read 16 bytes at offset, print hex+ASCII.
          chain <text?>           Decorator chain: BufferedStream over MemoryStream.
        Env: BUFFER_KB (default 64) — internal buffer size in KB.
        """);
    return 0;
}

static FileStreamOptions ReadOpts(long bufferSize) => new()
{
    Mode = FileMode.Open,
    Access = FileAccess.Read,
    Share = FileShare.Read,
    BufferSize = (int)bufferSize,
    Options = FileOptions.Asynchronous
};

static FileStreamOptions WriteOpts(long bufferSize) => new()
{
    Mode = FileMode.Create,
    Access = FileAccess.Write,
    Share = FileShare.None,
    BufferSize = (int)bufferSize,
    Options = FileOptions.Asynchronous
};

static long ResolveBufferBytes()
{
    string? kb = Environment.GetEnvironmentVariable("BUFFER_KB");
    return (int.TryParse(kb, out int k) && k > 0 ? k : DefaultBufferKb) * 1024L;
}

static async Task<int> CopyAsync(string src, string dst)
{
    long bufferSize = ResolveBufferBytes();
    var sw = Stopwatch.StartNew();
    long totalRead = 0;
    int iterations = 0;

    try
    {
        await using var input = new FileStream(src, ReadOpts(bufferSize));
        await using var output = new FileStream(dst, WriteOpts(bufferSize));

        byte[] buffer = new byte[bufferSize];
        int read;
        // ВАЖНО: выход при read == 0, запись ровно read байтов.
        // IMPORTANT: exit when read == 0, write exactly `read` bytes.
        while ((read = await input.ReadAsync(buffer)) != 0)
        {
            await output.WriteAsync(buffer, 0, read);
            totalRead += read;
            iterations++;
        }
        await output.FlushAsync();
    }
    catch (IOException ex)
    {
        Console.Error.WriteLine($"IO error ({src} → {dst}): {ex.Message}");
        return 1;
    }

    sw.Stop();
    double mb = totalRead / (1024.0 * 1024.0);
    double throughput = sw.Elapsed.TotalSeconds > 0 ? mb / sw.Elapsed.TotalSeconds : 0;
    Console.WriteLine(
        $"Copied {totalRead} bytes ({mb:F2} MiB) in {sw.Elapsed.TotalMilliseconds:F1} ms " +
        $"→ {throughput:F2} MiB/s, {iterations} iterations, buffer {bufferSize / 1024} KB");
    return 0;
}

static int Seek(string file, long offset)
{
    try
    {
        using var fs = new FileStream(file, new FileStreamOptions
        {
            Mode = FileMode.Open,
            Access = FileAccess.Read,
            Share = FileShare.Read,
            BufferSize = 4096,
            Options = FileOptions.None // синхронный доступ — демонстрация using / synchronous demo
        });

        Console.WriteLine($"Length = {fs.Length}");
        fs.Seek(offset, SeekOrigin.Begin);
        Console.WriteLine($"Position before read = {fs.Position}");

        byte[] buf = new byte[16];
        int read = fs.Read(buf, 0, buf.Length);
        Console.WriteLine($"Position after read  = {fs.Position} (read {read} bytes)");

        string hex = Convert.ToHexString(buf, 0, read);
        string ascii = new string([.. buf.Take(read).Select(b => (b >= 32 && b < 127) ? (char)b : '.')]);
        Console.WriteLine($"Hex:   {hex}");
        Console.WriteLine($"ASCII: {ascii}");
    }
    catch (IOException ex)
    {
        Console.Error.WriteLine($"IO error ({file}): {ex.Message}");
        return 1;
    }
    return 0;
}

static async Task<int> ChainAsync(string text)
{
    // Декоратор BufferedStream поверх MemoryStream — компонуемость Stream.
    // Decorator BufferedStream over MemoryStream — Stream composability.
    await using var mem = new MemoryStream();
    await using var buffered = new BufferedStream(mem, bufferSize: 8 * 1024);

    byte[] data = Encoding.UTF8.GetBytes(text);
    await buffered.WriteAsync(data);
    await buffered.FlushAsync(); // сброс декоратора в нижний поток / flush decorator into underlying
    buffered.Position = 0;

    byte[] all = new byte[buffered.Length];
    int read = await buffered.ReadAsync(all);
    Console.WriteLine($"Chain read {read} bytes: {Encoding.UTF8.GetString(all, 0, read)}");
    return 0;
}
```

Разбор по строкам. Top-level `return args.Length switch` — современный C# 12: вся программа — это одно switch-выражение, возвращающее `int` (код выхода). `ReadOpts` и `WriteOpts` — фабрики `FileStreamOptions`, которые централизуют все опции; это убирает дублирование и явно показывает, какие `Mode`/`Access`/`Share` мы ожидаем для чтения и записи. `FileOptions.Asynchronous` включён в обоих — поэтому дальше используются `ReadAsync`/`WriteAsync` и `await using`, что соответствует best practice урока: истинный async через IOCP, без блокировки потоков пула. В `CopyAsync` ключевой момент — цикл `while ((read = await input.ReadAsync(buffer)) != 0)`: мы пишем ровно `read` байтов через `WriteAsync(buffer, 0, read)`, а не `buffer.Length`, потому что `ReadAsync` может вернуть меньше запрошенного (контракт `Stream`). Это прямо из раздела «частые ошибки» урока. `await using var output` гарантирует, что поток закроется даже при исключении — это и есть правило «открыл — закрой». Порядок объявления `input` и `output` через `await using var` в одном блоке означает, что освобождение идёт в обратном порядке (output первым, input вторым), что для копирования безопасно. `ResolveBufferBytes` читает `BUFFER_KB`, давая возможность экспериментировать с размером буфера без перекомпиляции — это закрепляет идею, что `BufferSize` подбирается под паттерн доступа. В `Seek` намеренно показан синхронный поток с `FileOptions.None` и `using` (а не `await using`), чтобы продемонстрировать альтернативу для кратких операций и напомнить, что `Position` двигается после `Read`. Команда `chain` показывает декоратор `BufferedStream` поверх `MemoryStream` и обязательный `FlushAsync` перед перемоткой — без него последние байты остались бы в буфере декоратора и не попали в `MemoryStream`, что соответствует предупреждению урока про забытый flush декоратора. `await using` для `buffered` идёт раньше `mem` по синтаксису C# (переменные освобождаются в обратном порядке объявления), что обеспечивает корректный порядок закрытия.

#### Задания на углубление (бонус)

1. Добавьте команду `durability <text>`, которая пишет строку в файл через `FileStream` с `FileAccess.Write`, затем вызывает `fs.Flush(flushToDisk: true)` и замеряет разницу во времени по сравнению с обычным `FlushAsync`. Объясните в комментарии, почему `flushToDisk: true` дороже и когда он действительно нужен.
2. Реализуйте команду `benchmark <size_mb>`, которая генерирует файл указанного размера и копирует его с буферами 4 КБ, 16 КБ, 64 КБ, 256 КБ, 1 МБ, выводя таблицу пропускной способности. Сделайте вывод о том, после какого размера буфер перестаёт давать прирост.
3. Соберите трёхслойную цепочку декораторов: `BufferedStream` → `BufferedStream` → `MemoryStream` и убедитесь, что она работает. Объясните, почему вложенные `BufferedStream` обычно не имеют практического смысла, но корректны с точки зрения контракта `Stream`.
4. Добавьте поддержку `FileOptions.SequentialScan` и `FileOptions.RandomAccess` через флаг командной строки и замерьте, влияет ли это на пропускную способность для HDD-подобного паттерна (создайте файл, читайте его в случайном порядке через `Seek`).

---

## Statement in English / English statement

#### Context & motivation

You are an engineer on a team building a backup utility for small configuration files and logs. The team wants a single, honest, predictable copy mechanism that works equally well on a developer's SSD laptop, on a slow network share, and on a classic spinning hard drive. The goal is not just to "copy bytes" but to understand where time is actually spent: in syscalls, in disk wait, or in a suboptimal buffer size. The utility must measure performance honestly and print diagnostics from which an engineer can decide on the buffer size and the access mode.

In the previous lesson (M10-L01) you worked with `File` and `Path`, high-level helpers that hide the details. Now you drop down to the `Stream` and `FileStream` level to deliberately manage the file handle, the buffer, and the async mode. This is a critical skill: everything that follows — `StreamReader`/`StreamWriter`, `GZipStream`, `CryptoStream`, network streams — builds on the understanding of the base `Stream`. Once you can open a `FileStream` with the right options, release it correctly, and validate the byte count returned by reads, every higher-level wrapper becomes a simple composition of decorators over the very same abstraction. You stop seeing "file I/O" as a black box and start seeing a unified pipe you can wrap, layer, and measure.

#### What to do step by step

1. Create a new .NET 8 console project named `StreamLab`:
   ```bash
   dotnet new console -n StreamLab -o StreamLab --framework net8.0
   cd StreamLab
   dotnet new sln --name StreamLab
   dotnet sln add StreamLab.csproj
   ```
   In the `.csproj`, enable `<Nullable>enable</Nullable>` and `<ImplicitUsings>enable</ImplicitUsings>`, and target C# 12 (`<LangVersion>12</LangVersion>`).

2. In `Program.cs`, implement a top-level application with three commands: `copy`, `seek`, `chain`. The command is the first argument `args[0]`. With no arguments, print a help message.

3. The `copy <src> <dst>` command must copy the file through `FileStream` using `FileStreamOptions`. Required: `Mode = FileMode.Open`/`Create`, `Access = FileAccess.Read`/`Write`, `Share = FileShare.Read`, `BufferSize` configurable via the `BUFFER_KB` environment variable (default 64 KB), `Options = FileOptions.Asynchronous`. Use `await using` for both streams. Read into a `byte[] buffer` of size `BufferSize` in a loop; check the return value of `await fs.ReadAsync(buffer, ...)` — it may be less than requested. Write exactly `read` bytes: `await dst.WriteAsync(buffer, 0, read)`.

4. Measure performance with `Stopwatch`: print file size, copy time in milliseconds, throughput in MB/s, and the number of read iterations. Run the same copy (generate a 50 MB test file with `File.WriteAllBytes` or a `WriteAsync` loop) with buffers of 4 KB, 64 KB, and 1 MB, and compare. Record your observations as a comment in the code.

5. The `seek <file> <offset>` command must open the file with `FileAccess.Read` and `FileShare.Read`, move to the given `offset` via `fs.Seek(offset, SeekOrigin.Begin)`, and read 16 bytes. Print them in hex and as ASCII (replace non-printable bytes with `.`). Also show `fs.Position` before and after the read to demonstrate that `ReadAsync` advances the position.

6. The `chain <text>` command must demonstrate decorator composability: create a `MemoryStream`, wrap it in a `BufferedStream` with an 8 KB buffer, write `text` as UTF-8, call `FlushAsync`, rewind to the start via `Position = 0`, and read everything back. Print the recovered text. Then assemble a three-layer chain: `BufferedStream` over `MemoryStream` is the minimum; as a bonus, try wrapping it with another `BufferedStream` to show that decorators nest without errors.

7. After each `FlushAsync` on the decorator, make sure to dispose it (via `await using`) before the underlying stream, so that the final bytes are actually flushed down. Pay attention to closing order: the decorator closes first, the base stream second.

8. Build and run:
   ```bash
   dotnet run -- copy sample.bin sample-copy.bin
   dotnet run -- seek sample.bin 100
   dotnet run -- chain "Hello, decorator world!"
   ```
   Verify that `sample-copy.bin` is byte-for-byte identical to `sample.bin` (compare via `System.Security.Cryptography.SHA256`).

#### Requirements

- Target platform: .NET 8, C# 12. Use top-level statements, file-scoped namespaces, collection expressions, and pattern matching where appropriate (e.g. a switch expression on `args` to dispatch commands).
- Every `FileStream` must be created through `FileStreamOptions` with explicit `Mode`, `Access`, `Share`, `BufferSize`, and `Options`. No legacy constructors like `new FileStream(path, FileMode.Create)` without options.
- Every opened stream must be wrapped in `using` or `await using`. Streams opened with `FileOptions.Asynchronous` must use `await using`. No bare `new FileStream` without disposal.
- Async operations (`ReadAsync`/`WriteAsync`) are called only on streams opened with `FileOptions.Asynchronous`. If a stream is synchronous, use synchronous `Read`/`Write` or document the deviation in a comment.
- In the copy loop, the returned byte count must be checked: `read` may be less than `buffer.Length`. Never write `buffer.Length` instead of `read` into `WriteAsync`.
- The code must tolerate empty and very small files, as well as files larger than 2 GB (use `long`, not `int`, for sizes and offsets).
- All I/O exceptions must be caught and logged with a clear message and the file name. Never swallow exceptions silently.

#### Pitfalls

- `ReadAsync` is not obligated to return exactly the number of bytes you requested. For file streams it usually fills the buffer, but this is not part of the `Stream` contract — do not rely on it. The exit condition in the copy loop is `read == 0`, not `read < buffer.Length`.
- Without `FileOptions.Asynchronous`, the "async" calls use synchronous I/O under the hood and block a pool thread through overlapped emulation. For true IOCP-based async, the flag is mandatory. The lesson emphasizes this directly.
- `Flush` on `FileStream` empties the internal buffer into the OS kernel, but does not guarantee a write to the physical device. If you need durability (e.g. a transaction log), call `fs.Flush(flushToDisk: true)`, which under the hood calls `FlushFileBuffers`. This is expensive but required for critical data.
- `Position` and `Seek` are ignored under `FileMode.Append` — the position is always "end of file". Do not try to append with a prior `Seek`; it will not work. To append, just write.
- A `BufferedStream` over a `MemoryStream` does not speed anything up (memory is already fast), but it is a clean demonstration of the decorator pattern. For real files, `BufferedStream` over an unbuffered `FileStream` would matter, but .NET's `FileStream` is already buffered by default, so an extra `BufferedStream` is usually redundant.
- Do not use a stream after `Dispose`/`DisposeAsync` — you will get `ObjectDisposedException`. This is especially tricky in branching code where one path closes the stream earlier than another.
- Close decorators before their base streams: with `await using`, the decorator must be declared "outside" the base so disposal order is correct (decorator `DisposeAsync` → base `DisposeAsync`). Otherwise the decorator's final bytes never reach the base stream.
- `FileShare.Read` lets other processes read but not write. `FileShare.None` fully locks the file. Pick the right one per scenario: for copying a source, `FileShare.Read` is enough; for the destination, `FileShare.None` is safer.

#### Acceptance criteria

- [ ] The `StreamLab` project builds without warnings under `dotnet build -warnaserror`.
- [ ] All `FileStream` instances are created through `FileStreamOptions` with explicit `Mode`, `Access`, `Share`, `BufferSize`, `Options`.
- [ ] All streams are wrapped in `using` or `await using`; `FileOptions.Asynchronous` streams use `await using`.
- [ ] In the copy loop, `read` is checked correctly: exit on `read == 0`, write exactly `read` bytes.
- [ ] The `copy` command handles files of 0 bytes, 1 byte, 50 MB, and over 2 GB without errors and without `int` overflow.
- [ ] The SHA256 of the copy matches the SHA256 of the original for every test size.
- [ ] The `seek` command prints both hex and ASCII of 16 bytes, mapping non-printables to `.`.
- [ ] The `seek` command shows `Position` before and after `ReadAsync` and confirms the position advanced.
- [ ] The `chain` command assembles `BufferedStream` over `MemoryStream`, writes, rewinds, reads, and prints the original text.
- [ ] The decorator is disposed before the base stream (correct `await using` order).
- [ ] The `copy` command measures and prints time, throughput in MB/s, and iteration count.
- [ ] Buffer comparison of 4 KB / 64 KB / 1 MB is performed and briefly commented in the code.
- [ ] Command dispatch uses C# 12 pattern matching / a switch expression.
- [ ] I/O exceptions are logged with the file name and never swallowed.
- [ ] No `TODO`, no commented-out junk, no bare `new FileStream` without `using`.

#### Hints (no direct answer)

- To compare files, use `SHA256.HashData` synchronously on a `ReadOnlySpan<byte>` from `File.ReadAllBytes` for tests; for large files, stream through `SHA256.Create()` and `CopyToAsync`.
- Read the environment variable with `Environment.GetEnvironmentVariable("BUFFER_KB")` and parse with `int.TryParse`; default to 64.
- For hex use `Convert.ToHexString(byte[])` (available in .NET 8); for ASCII use a manual loop checking `>= 32 && < 127`.
- In the switch expression on `args`, handle `args.Length == 0` and print help using a raw string literal `"""..."""`.
- To generate a 50 MB test file, do not fill bytes one by one — use `Random.Shared.GetItems` or a pre-built array written in a `WriteAsync` loop.

#### Reference solution walk-through (English)

The reference solution mirrors the Russian code above. The top-level `return args.Length switch` is modern C# 12: the entire program is a single switch expression returning `int` (the exit code). `ReadOpts` and `WriteOpts` are `FileStreamOptions` factories that centralize every option and make the intent explicit; `FileOptions.Asynchronous` is set in both, so the rest of the code uses `ReadAsync`/`WriteAsync` and `await using`, matching the lesson's best practice of true IOCP-based async without blocking pool threads. In `CopyAsync`, the crucial line is `while ((read = await input.ReadAsync(buffer)) != 0)` followed by `await output.WriteAsync(buffer, 0, read)` — writing exactly `read` bytes, never `buffer.Length`, because `ReadAsync` may return fewer bytes than requested, which is the `Stream` contract. This directly addresses the "common mistakes" section of the lesson. `await using var output` guarantees disposal even on exceptions, honoring the "if you opened it, close it" rule. Declaring `input` and `output` in the same scope means disposal runs in reverse order (output first, then input), which is safe for copying. `ResolveBufferBytes` reads `BUFFER_KB`, enabling experimentation without recompilation, reinforcing the idea that `BufferSize` should match the access pattern. In `Seek`, a synchronous stream with `FileOptions.None` and `using` (not `await using`) is intentionally shown to demonstrate the alternative for short operations and to remind the reader that `Position` advances after `Read`. The `chain` command shows `BufferedStream` over `MemoryStream` with a mandatory `FlushAsync` before rewinding — without it, the last bytes stay in the decorator buffer and never reach the `MemoryStream`, which matches the lesson's warning about forgetting to flush a decorator. The `await using` for `buffered` precedes `mem` lexically, so C# disposes them in reverse declaration order, ensuring the correct close sequence.

```csharp
// C# 12 / .NET 8 — Stream, FileStream, buffering — reference StreamLab.
// Demonstrates FileStreamOptions, true async, await using, read-count check,
// Seek/Position, decorator composition.

using System.Diagnostics;
using System.Security.Cryptography;
using System.Text;

const int DefaultBufferKb = 64;

return args.Length switch
{
    0 => PrintHelp(),
    _ => args[0].ToLowerInvariant() switch
    {
        "copy"  => await CopyAsync(args[1], args[2]),
        "seek"  => Seek(args[1], long.Parse(args[2])),
        "chain" => await ChainAsync(args.Length > 1 ? args[1] : "decorator demo"),
        _ => PrintHelp()
    }
};

static int PrintHelp()
{
    Console.WriteLine("""
        StreamLab — usage:
          copy <src> <dst>        Copy file with measured buffered FileStream.
          seek <file> <offset>    Read 16 bytes at offset, print hex+ASCII.
          chain <text?>           Decorator chain: BufferedStream over MemoryStream.
        Env: BUFFER_KB (default 64) — internal buffer size in KB.
        """);
    return 0;
}

static FileStreamOptions ReadOpts(long bufferSize) => new()
{
    Mode = FileMode.Open,
    Access = FileAccess.Read,
    Share = FileShare.Read,
    BufferSize = (int)bufferSize,
    Options = FileOptions.Asynchronous
};

static FileStreamOptions WriteOpts(long bufferSize) => new()
{
    Mode = FileMode.Create,
    Access = FileAccess.Write,
    Share = FileShare.None,
    BufferSize = (int)bufferSize,
    Options = FileOptions.Asynchronous
};

static long ResolveBufferBytes()
{
    string? kb = Environment.GetEnvironmentVariable("BUFFER_KB");
    return (int.TryParse(kb, out int k) && k > 0 ? k : DefaultBufferKb) * 1024L;
}

static async Task<int> CopyAsync(string src, string dst)
{
    long bufferSize = ResolveBufferBytes();
    var sw = Stopwatch.StartNew();
    long totalRead = 0;
    int iterations = 0;

    try
    {
        await using var input = new FileStream(src, ReadOpts(bufferSize));
        await using var output = new FileStream(dst, WriteOpts(bufferSize));

        byte[] buffer = new byte[bufferSize];
        int read;
        // IMPORTANT: exit when read == 0, write exactly `read` bytes.
        while ((read = await input.ReadAsync(buffer)) != 0)
        {
            await output.WriteAsync(buffer, 0, read);
            totalRead += read;
            iterations++;
        }
        await output.FlushAsync();
    }
    catch (IOException ex)
    {
        Console.Error.WriteLine($"IO error ({src} → {dst}): {ex.Message}");
        return 1;
    }

    sw.Stop();
    double mb = totalRead / (1024.0 * 1024.0);
    double throughput = sw.Elapsed.TotalSeconds > 0 ? mb / sw.Elapsed.TotalSeconds : 0;
    Console.WriteLine(
        $"Copied {totalRead} bytes ({mb:F2} MiB) in {sw.Elapsed.TotalMilliseconds:F1} ms " +
        $"→ {throughput:F2} MiB/s, {iterations} iterations, buffer {bufferSize / 1024} KB");
    return 0;
}

static int Seek(string file, long offset)
{
    try
    {
        using var fs = new FileStream(file, new FileStreamOptions
        {
            Mode = FileMode.Open,
            Access = FileAccess.Read,
            Share = FileShare.Read,
            BufferSize = 4096,
            Options = FileOptions.None // synchronous access — demo of `using`
        });

        Console.WriteLine($"Length = {fs.Length}");
        fs.Seek(offset, SeekOrigin.Begin);
        Console.WriteLine($"Position before read = {fs.Position}");

        byte[] buf = new byte[16];
        int read = fs.Read(buf, 0, buf.Length);
        Console.WriteLine($"Position after read  = {fs.Position} (read {read} bytes)");

        string hex = Convert.ToHexString(buf, 0, read);
        string ascii = new string([.. buf.Take(read).Select(b => (b >= 32 && b < 127) ? (char)b : '.')]);
        Console.WriteLine($"Hex:   {hex}");
        Console.WriteLine($"ASCII: {ascii}");
    }
    catch (IOException ex)
    {
        Console.Error.WriteLine($"IO error ({file}): {ex.Message}");
        return 1;
    }
    return 0;
}

static async Task<int> ChainAsync(string text)
{
    // Decorator BufferedStream over MemoryStream — Stream composability.
    await using var mem = new MemoryStream();
    await using var buffered = new BufferedStream(mem, bufferSize: 8 * 1024);

    byte[] data = Encoding.UTF8.GetBytes(text);
    await buffered.WriteAsync(data);
    await buffered.FlushAsync(); // flush decorator into underlying stream
    buffered.Position = 0;

    byte[] all = new byte[buffered.Length];
    int read = await buffered.ReadAsync(all);
    Console.WriteLine($"Chain read {read} bytes: {Encoding.UTF8.GetString(all, 0, read)}");
    return 0;
}
```

#### Going deeper (bonus)

1. Add a `durability <text>` command that writes a string to a file via `FileStream` with `FileAccess.Write`, then calls `fs.Flush(flushToDisk: true)` and measures the time difference against a plain `FlushAsync`. Explain in a comment why `flushToDisk: true` is more expensive and when it is genuinely needed.
2. Implement a `benchmark <size_mb>` command that generates a file of the given size and copies it with buffers of 4 KB, 16 KB, 64 KB, 256 KB, and 1 MB, printing a throughput table. Conclude at which buffer size the gains plateau.
3. Build a three-layer decorator chain: `BufferedStream` → `BufferedStream` → `MemoryStream` and verify it works. Explain why nested `BufferedStream`s are generally pointless in practice but correct under the `Stream` contract.
4. Add support for `FileOptions.SequentialScan` and `FileOptions.RandomAccess` via a command-line flag and measure whether throughput changes for an HDD-like access pattern (build a file, then read it in random order via `Seek`).

---

#### Чек-лист сдачи / Submission checklist

- [ ] (RU) Проект `StreamLab` собирается без предупреждений.
- [ ] (RU) Все `FileStream` создаются через `FileStreamOptions`.
- [ ] (RU) Все потоки закрыты через `using`/`await using`.
- [ ] (RU) Команды `copy`/`seek`/`chain` работают и выводят ожидаемое.
- [ ] (RU) SHA256 копии совпадает с оригиналом.
- [ ] (RU) Сравнение буферов выполнено и прокомментировано.
- [ ] (RU) Нет `TODO`, нет закомментированного мусора, нет голых `new FileStream`.
- [ ] (EN) The `StreamLab` project builds without warnings.
- [ ] (EN) All `FileStream` instances use `FileStreamOptions`.
- [ ] (EN) All streams are disposed via `using`/`await using`.
- [ ] (EN) The `copy`/`seek`/`chain` commands work and print expected output.
- [ ] (EN) The SHA256 of the copy matches the original.
- [ ] (EN) Buffer comparison is performed and commented.
- [ ] (EN) No `TODO`, no commented-out junk, no bare `new FileStream`.

#### Ресурсы / Resources

- [Microsoft Learn — FileStream](https://learn.microsoft.com/dotnet/api/system.io.filestream)
- [Microsoft Learn — FileStreamOptions](https://learn.microsoft.com/dotnet/api/system.io.filestreamoptions)
- [Microsoft Learn — Stream](https://learn.microsoft.com/dotnet/api/system.io.stream)
- [Microsoft Learn — BufferedStream](https://learn.microsoft.com/dotnet/api/system.io.bufferedstream)
- [Microsoft Learn — FileOptions](https://learn.microsoft.com/dotnet/api/system.io.fileoptions)
- [.NET async I/O and FileOptions.Asynchronous — Stephen Toub](https://learn.microsoft.com/archive/blogs/pfxteam/faq-is-it-alright-to-use-filestream-in-async-code)

---
[← К уроку M10-L02](lesson-M10-L02-stream-filestream.md) | [⬆ К модулю M10](../README.md) | [Следующее ДЗ →](homework-M10-L03-streamreader-writer-encoding.md)
