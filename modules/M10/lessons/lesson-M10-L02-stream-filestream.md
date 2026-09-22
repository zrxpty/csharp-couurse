[⬆ К модулю M10](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M10-L02: Stream, FileStream, буферизация / Stream, FileStream, buffering

**Модуль / Module:** M10
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

**Stream как абстракция.** В .NET `Stream` — это базовый абстрактный класс, описывающий последовательность байтов, которую можно читать, писать и по которой можно перемещаться (seek). Главная идея: поток — это «труба» между источником данных (файл, память, сеть, криптопровайдер) и вашим кодом. Не важно, что лежит по ту сторону трубы — вы работаете с одним и тем же API: `Read`, `Write`, `Seek`, `Flush`, `Length`, `Position`. Эта унификация позволяет, например, сжать данные через `GZipStream`, не зная, куда они потом попадут — в файл, в память или в сетевой сокет.

**FileStream — поток для файлов.** `FileStream` — конкретная реализация `Stream`, работающая поверх файлового дескриптора операционной системы. Он умеет открывать файл на чтение, запись или чтение-запись, может работать в синхронном и асинхронном режимах, поддерживает позиционирование (`Seek`) и сообщает длину (`Length`). Каждый вызов `Read`/`Write` без буферизации — это системный вызов (syscall) в ядро ОС, а это дорого. Поэтому по умолчанию `FileStream` держит внутренний буфер в памяти и копит байты, прежде чем обратиться к диску.

**Буферизация.** Аналогия: вы носите воду из колодца. Без буфера вы бегаете с кружкой за каждым глотком — сто поездок, сто обращений к колодцу. С буфером вы берёте ведро, наполняете его один раз, а потом разливаете по чашкам дома. `FileStream` делает то же самое: читает «ведро» байтов за один syscall и потом раздаёт их из памяти. Аналогично на запись: накапливает в буфере и выливает на диск одним заходом. Метод `Flush` принудительно сбрасывает буфер туда, куда он предназначен. Важно понимать: `Flush` на `FileStream` пишет данные на диск на уровне ОС, но не гарантирует, что физическое устройство их уже зафиксировало (для этого есть `Flush(flushToDisk: true)` — он вызывает `FlushFileBuffers`).

**FileStreamOptions.** Начиная с .NET 6 появился тип `FileStreamOptions`, который собрал все параметры открытия файла в одном месте: `Mode` ( FileMode), `Access` (FileAccess), `Share` (FileShare), размер буфера `BufferSize`, флаг `Options` (для `FileOptions.Asynchronous`, `RandomAccess`, `SequentialScan`, `Encrypted`), `PreallocationSize` и `UnmanagedCreateOptions`. Это делает конструкторы `FileStream` читаемее и даёт доступ к новым возможностям. Флаг `FileOptions.Asynchronous` особенно важен: он переключает поток в «истинно асинхронный» режим, при котором `ReadAsync`/`WriteAsync` используют IOCP вместо блокировки потоков пула.

**Dispose и using/await using.** `Stream` реализует `IDisposable`, а многие его наследники — ещё и `IAsyncDisposable`. Файл — это неуправляемый ресурс ОС, и его нужно закрывать. Правило: если открыли, обязаны закрыть. В C# 12 есть два способа: `using` (синхронный, вызывает `Dispose`, блокирует до завершения I/O) и `await using` (асинхронный, вызывает `DisposeAsync`, не блокирует поток). Для `FileStream` с `FileOptions.Asynchronous` предпочтителен `await using` — закрытие тоже пойдёт через пул без блокировки. В новых проектах по умолчанию используйте `await using` для потоков, поддерживающих асинхронную очистку. Никогда не оставляйте `FileStream` без `using` — сборщик мусора не гарантирует своевременного освобождения дескриптора, и вы получите «файл занят другим процессом» в самый неподходящий момент.

#### Theory (EN)

**Stream as an abstraction.** In .NET, `Stream` is an abstract base class describing a sequence of bytes you can read, write, and navigate (seek). The key idea: a stream is a "pipe" between a data source (file, memory, network, crypto provider) and your code. It does not matter what sits on the far end of the pipe — you work with the same API: `Read`, `Write`, `Seek`, `Flush`, `Length`, `Position`. This unification lets you, for example, compress data through `GZipStream` without knowing whether it later lands in a file, in memory, or in a network socket. Decorator streams wrap other streams, so `BufferedStream`, `CryptoStream`, `GZipStream` compose cleanly on top of any base stream.

**FileStream — a stream for files.** `FileStream` is a concrete `Stream` implementation built on top of an OS file handle. It can open a file for reading, writing, or read-write; it works in both synchronous and asynchronous modes; it supports positioning (`Seek`) and reports its length (`Length`). Every unbuffered `Read`/`Write` call is a syscall into the kernel, and syscalls are expensive. So by default `FileStream` keeps an internal in-memory buffer and accumulates bytes before touching the disk.

**Buffering.** Analogy: you carry water from a well. Without a buffer you run with a cup for every sip — a hundred trips, a hundred well visits. With a buffer you take a bucket, fill it once, and pour into cups at home. `FileStream` does the same: it reads a "bucket" of bytes in one syscall, then hands them out from memory. The same happens on writes — bytes accumulate in the buffer and are flushed to disk in one shot. The `Flush` method forces the buffer to its destination. Important nuance: `Flush` on `FileStream` writes to the OS-level file, but does not guarantee the physical device has persisted it; for that use `Flush(flushToDisk: true)`, which calls `FlushFileBuffers`.

**FileStreamOptions.** Starting with .NET 6, the `FileStreamOptions` type gathers every file-opening parameter in one place: `Mode` (FileMode), `Access` (FileAccess), `Share` (FileShare), `BufferSize`, `Options` (for `FileOptions.Asynchronous`, `RandomAccess`, `SequentialScan`, `Encrypted`), `PreallocationSize`, and `UnmanagedCreateOptions`. This makes `FileStream` constructors far more readable and exposes new capabilities. The `FileOptions.Asynchronous` flag is especially important: it switches the stream into "true async" mode, where `ReadAsync`/`WriteAsync` use IOCP instead of blocking pool threads. Without that flag, "async" calls still tie up a thread.

**Dispose and using/await using.** `Stream` implements `IDisposable`, and many of its subclasses also implement `IAsyncDisposable`. A file is an unmanaged OS resource and must be closed. Rule: if you opened it, you must close it. In C# 12 there are two tools: `using` (synchronous — calls `Dispose`, blocks until I/O finishes) and `await using` (asynchronous — calls `DisposeAsync`, does not block the thread). For a `FileStream` opened with `FileOptions.Asynchronous`, prefer `await using` so even the close goes through the pool without blocking. In modern projects default to `await using` for streams that support async disposal. Never leave a `FileStream` without a `using` — the garbage collector does not guarantee timely handle release, and you will get "file in use by another process" at the worst possible moment.

#### Пример кода / Code Example

```csharp
// C# 12 / .NET 8 — Stream, FileStream, buffering
// Демонстрируем: FileStreamOptions, буферизацию, async I/O, await using.

using System.Text;

const string filePath = "demo.bin";

// 1) Запись через FileStreamOptions + true async + буфер 64 КБ.
//    Write through FileStreamOptions + true async + 64 KB buffer.
var writeOpts = new FileStreamOptions
{
    Mode = FileMode.Create,
    Access = FileAccess.Write,
    Share = FileShare.None,
    BufferSize = 64 * 1024,                 // размер внутреннего буфера / internal buffer size
    Options = FileOptions.Asynchronous      // истинный async через IOCP / true async via IOCP
};

await using (var fs = new FileStream(filePath, writeOpts))
{
    // Пишем строку как UTF-8 байты.
    // Write a string as UTF-8 bytes.
    byte[] data = Encoding.UTF8.GetBytes("Hello, FileStream! Привет, поток!");
    await fs.WriteAsync(data);              // буфер копит, потом сбросит / buffer accumulates then flushes
    await fs.FlushAsync();                  // принудительный сброс / explicit flush
}

// 2) Чтение с positional Seek — демонстрация случайного доступа.
//    Read with positional Seek — random access demo.
var readOpts = new FileStreamOptions
{
    Mode = FileMode.Open,
    Access = FileAccess.Read,
    Share = FileShare.Read,
    BufferSize = 64 * 1024,
    Options = FileOptions.Asynchronous
};

await using (var fs = new FileStream(filePath, readOpts))
{
    Console.WriteLine($"Length = {fs.Length} bytes"); // длина файла / file length

    // Перемещаемся к 7-му байту (пропускаем "Hello, ").
    // Move to the 7th byte (skip "Hello, ").
    fs.Seek(offset: 7, SeekOrigin.Begin);
    fs.Position = 7; // эквивалент Seek / equivalent to Seek

    var buf = new byte[fs.Length - 7];
    int read = await fs.ReadAsync(buf);     // асинхронно читаем в буфер / async read into buffer
    string text = Encoding.UTF8.GetString(buf, 0, read);
    Console.WriteLine($"Read: {text} ({read} bytes)");
}

// 3) Обёртка декоратором BufferedStream поверх MemoryStream —
//    показать единый API Stream и компонуемость декораторов.
//    Wrap a decorator BufferedStream over MemoryStream —
//    show the unified Stream API and decorator composability.
using var mem = new MemoryStream();
using var buffered = new BufferedStream(mem, bufferSize: 8 * 1024);
await buffered.WriteAsync(Encoding.UTF8.GetBytes("decorator chain"));
await buffered.FlushAsync();
buffered.Position = 0;
var all = new byte[buffered.Length];
_ = await buffered.ReadAsync(all);
Console.WriteLine($"Chain: {Encoding.UTF8.GetString(all)}");
```

#### Best Practices

- Открывайте `FileStream` с `FileOptions.Asynchronous`, если планируете `ReadAsync`/`WriteAsync` — иначе «асинхронные» вызовы блокируют поток пула. / Open `FileStream` with `FileOptions.Asynchronous` if you plan to use `ReadAsync`/`WriteAsync` — otherwise "async" calls block a pool thread.
- Используйте `await using` для потоков с `IAsyncDisposable`; для синхронного кода — `using`. / Use `await using` for streams that implement `IAsyncDisposable`; for synchronous code, use `using`.
- Подбирайте `BufferSize` под паттерн доступа: 4–64 КБ для последовательного I/O, больший — для больших файлов. / Choose `BufferSize` for the access pattern: 4–64 KB for sequential I/O, larger for big files.
- Сбрасывайте буфер (`Flush`/`FlushAsync`) перед передачей файла другому процессу или закрытием связанного потока-декоратора. / Flush the buffer (`Flush`/`FlushAsync`) before handing a file to another process or closing a related decorator stream.
- Предпочитайте `FileOptions.SequentialScan` для линейного чтения и `FileOptions.RandomAccess` для скачкообразного — ОС подстроит кэширование. / Prefer `FileOptions.SequentialScan` for linear reads and `FileOptions.RandomAccess` for jump access — the OS tunes caching.
- Задавайте `PreallocationSize`, если пишете большой файл по частям — избегаете фрагментации. / Set `PreallocationSize` when writing a large file in chunks — avoids fragmentation.

#### Частые ошибки / Common Mistakes

- Не закрываете `FileStream` (нет `using`) → файл остаётся заняты до GC; используйте `using`/`await using`. / Not closing `FileStream` (no `using`) → file stays locked until GC; use `using`/`await using`.
- Зываете `Read`/`Write` без проверки возвращаемого числа байтов → получаете «обрезанные» данные; всегда сверяйте `read` с запрошенной длиной. / Calling `Read`/`Write` without checking the returned byte count → you get truncated data; always compare `read` with the requested length.
- Используете `ReadAsync` на потоке без `FileOptions.Asynchronous` → синхронный I/O блокирует поток пула; ставьте флаг при создании. / Using `ReadAsync` on a stream without `FileOptions.Asynchronous` → synchronous I/O blocks a pool thread; set the flag on creation.
- Забываете `FlushAsync` перед закрытием декоратора (`GZipStream`, `CryptoStream`) → последние блоки не дописываются; flush декоратор, затем закрывайте. / Forgetting `FlushAsync` before closing a decorator (`GZipStream`, `CryptoStream`) → final blocks are not written; flush the decorator, then close.
- Читаете из закрытого/освобождённого потока → `ObjectDisposedException`; не используйте поток после `Dispose`. / Reading from a disposed stream → `ObjectDisposedException`; never use a stream after `Dispose`.
- Путаете `Position` и `Seek` с `FileMode.Append` → позиция игнорируется при append; для дописывания просто пишите. / Confusing `Position`/`Seek` with `FileMode.Append` → position is ignored in append mode; just write to append.
- Считаете, что `Flush` = данные физически на диске → используйте `Flush(flushToDisk: true)` для durability. / Assuming `Flush` means data is physically on disk → use `Flush(flushToDisk: true)` for durability.

#### Чек-лист самопроверки / Self-check Checklist

- [ ] Я открываю `FileStream` через `FileStreamOptions` и указываю `Mode`, `Access`, `Share`, `BufferSize`, `Options`. / I open `FileStream` via `FileStreamOptions` and specify `Mode`, `Access`, `Share`, `BufferSize`, `Options`.
- [ ] Я ставлю `FileOptions.Asynchronous`, если работаю через `ReadAsync`/`WriteAsync`. / I set `FileOptions.Asynchronous` when using `ReadAsync`/`WriteAsync`.
- [ ] Каждый открытый поток обёрнут в `using` или `await using`. / Every opened stream is wrapped in `using` or `await using`.
- [ ] Я проверяю количество прочитанных байтов, возвращаемых `Read`/`ReadAsync`. / I check the byte count returned by `Read`/`ReadAsync`.
- [ ] Я понимаю, что `Flush` сбрасывает буфер, но не гарантирует запись на физический носитель. / I understand `Flush` clears the buffer but does not guarantee a write to the physical medium.
- [ ] Я выбираю `SequentialScan` или `RandomAccess` под паттерн доступа. / I choose `SequentialScan` or `RandomAccess` for the access pattern.
- [ ] Я знаю разницу между `Dispose` и `DisposeAsync` и применяю правильную форму. / I know the difference between `Dispose` and `DisposeAsync` and apply the correct form.
- [ ] Я могу объяснить, почему декораторы (`BufferedStream`, `GZipStream`) работают поверх любого `Stream`. / I can explain why decorators (`BufferedStream`, `GZipStream`) work over any `Stream`.

#### Ресурсы / Resources

- [Microsoft Learn — https://learn.microsoft.com/dotnet/api/system.io.filestream]

---

[⬆ К модулю M10](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
