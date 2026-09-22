---
[← Предыдущее ДЗ](homework-M10-L05-system-text-json.md) | [← К уроку M10-L06](lesson-M10-L06-memorystream-pipes.md) | [⬆ К модулю M10](../README.md) | [Следующее ДЗ →](homework-M10-L07-xml-serialization.md)
---

### Домашнее задание M10-L06: MemoryStream, PipeReader/PipeWriter (обзор) / Homework M10-L06: MemoryStream, PipeReader/PipeWriter (overview)

**Урок / Lesson:** M10-L06
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 3/5
**Цель / Goal:** (RU) Научиться осознанно выбирать между `MemoryStream` и `System.IO.Pipelines`, реализовать бинарный протокол кадров с префиксом длины поверх обоих API, корректно управлять арендой буферов, обратным давлением и жизненным циклом `PipeReader`/`PipeWriter`, а также адаптировать `Stream` к конвейеру и обратно. (EN) Learn to choose deliberately between `MemoryStream` and `System.IO.Pipelines`, implement a length-prefixed binary frame protocol on top of both APIs, correctly manage buffer rental, backpressure and the `PipeReader`/`PipeWriter` lifecycle, and adapt a `Stream` to a pipe and back.

#### Связь с уроком / Connection to the lesson
(RU) Урок вводит `MemoryStream` как простое in-memory хранилище на `byte[]` и переходит к `System.IO.Pipelines` с пулом `ArrayPool<byte>`, явным контролем потребления через `AdvanceTo(consumed, examined)` и встроенным backpressure через `pauseWriterThreshold`/`resumeWriterThreshold`. ДЗ закрепляет все эти концепции на едином сквозном протоколе «кадр = [длина:4 BE][payload]», который мы реализуем сначала через `MemoryStream`, затем через `Pipe`, и, наконец, через адаптер `Stream`→`PipeReader`.
(EN) The lesson introduces `MemoryStream` as a simple in-memory `byte[]`-backed store and moves on to `System.IO.Pipelines` with an `ArrayPool<byte>` pool, explicit consumption control via `AdvanceTo(consumed, examined)`, and built-in backpressure through `pauseWriterThreshold`/`resumeWriterThreshold`. This homework cements all of those concepts on a single end-to-end "frame = [length:4 BE][payload]" protocol that we first implement over `MemoryStream`, then over a `Pipe`, and finally through a `Stream`→`PipeReader` adapter.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация
Вы прототипируете компактный in-process брокер сообщений для учебного сервиса логирования. Сообщения нужно уметь упаковывать в байты, передавать через канал и разбирать на приёмной стороне. Канал может доставлять данные произвольными порциями: иногда целый кадр приходит за один раз, иногда — только первые несколько байт заголовка, а тело приезжает позже. Такая ситуация типична для сетевых протоколов (HTTP, WebSocket, RESP, бинарные кадры) и требует парсера, устойчивого к частичным кадрам.

В модуле M10 вы уже работали с `FileStream` и буферизованными потоками, а в этом уроке увидели два in-memory инструмента: классический `MemoryStream` и современный `System.IO.Pipelines`. `MemoryStream` проще: его внутреннее хранилище — обычный `byte[]` в управляемой куче, он умеет автоматически расти, поддерживает синхронные и асинхронные операции и даёт прямой доступ к буферу через `GetBuffer()` (или копию через `ToArray()`). Но за простоту платят копированиями: каждый `Read`/`Write` копирует байты, а рост буфера аллоцирует новый массив и копирует старое содержимое. Для высоконагруженных горяих путей (Kestrel, сетевые парсеры, крипто-конвейеры) это дорого.

`System.IO.Pipelines` устроен иначе: писатель арендует регион у `ArrayPool<byte>` через `GetSpan`/`GetMemory`, заполняет его, фиксирует записанное через `Advance` и отправляет через `FlushAsync`; читатель через `ReadAsync` получает `ReadResult` с `Buffer` типа `ReadOnlySequence<byte>` (возможно, из нескольких несмежных сегментов), а после обработки вызывает `AdvanceTo`, возвращая память в пул. У конвейера есть встроенное обратное давление: при превышении `pauseWriterThreshold` метод `FlushAsync` писателя приостанавливается, пока читатель не опустошит буфер ниже `resumeWriterThreshold`. Цель ДЗ — прочувствовать обе модели на одном и том же протоколе и научиться выбирать правильный инструмент осознанно.

#### Что нужно сделать (пошагово)
1. Создайте консольный проект на .NET 8: `dotnet new console -n M10L06.Homework -o M10L06.Homework --framework net8.0`, перейдите в папку `cd M10L06.Homework` и откройте `Program.cs`. Убедитесь, что в `.csproj` включён `<Nullable>enable</Nullable>` и `<ImplicitUsings>enable</ImplicitUsings>`.
2. Определите модель данных: перечисление `LogLevel { Info, Warn, Error }` и запись `record LogEntry(DateTimeOffset Timestamp, LogLevel Level, string Message);`. Используйте коллекционное выражение для инициализации массива из трёх записей.
3. **Часть A — MemoryStream.** Реализуйте два метода: `byte[] SerializeWithMemoryStream(LogEntry[] entries)` и `LogEntry[] DeserializeWithMemoryStream(byte[] blob)`. Сериализация: откройте `new MemoryStream(capacity: 1024)` (подсказка размера, чтобы избежать лишних перераспределений), оберните его в `BinaryWriter`, запишите счётчик записей (`int`), затем для каждой записи — `Timestamp.ToUnixTimeSeconds()` (`long`), `(int)Level`, длину сообщения (`int`) и UTF-8 байты сообщения. Верните `ms.ToArray()`. Десериализация: откройте `new MemoryStream(blob, writable: false)`, оберните в `BinaryReader`, прочитайте счётчик и в цикле восстановите записи. Выведите в консоль размер blob и количество восстановленных записей.
4. **Часть B — Pipelines.** Создайте `Pipe` с `PipeOptions(pool: ArrayPool<byte>.Shared, pauseWriterThreshold: 64, resumeWriterThreshold: 32)`. Реализуйте `async Task FrameWriterAsync(PipeWriter writer, string message, CancellationToken ct)`: получите `payload = Encoding.UTF8.GetBytes(message)`, запросите `writer.GetSpan(4 + payload.Length)`, запишите длину big-endian через `BinaryPrimitives.WriteInt32BigEndian(span[..4], payload.Length)`, скопируйте payload в `span[4..]`, вызовите `writer.Advance(4 + payload.Length)` и `await writer.FlushAsync(ct)`. Реализуйте `async Task FrameReaderAsync(PipeReader reader, CancellationToken ct)`: в цикле `await reader.ReadAsync(ct)`, получите `result.Buffer`, в цикле вызывайте `TryReadFrame(ref buffer, out byte[]? payload)`, печатая каждый кадр, затем вызовите `reader.AdvanceTo(buffer.Start, buffer.End)` (важно: `examined` указывает на конец доступных данных, чтобы корректно ждать ещё данные при неполном кадре), проверьте `result.IsCompleted` и завершите через `reader.Complete()`. Реализуйте `bool TryReadFrame(ref ReadOnlySequence<byte> buffer, out byte[]? payload)`: если `< 4` байт — неполный заголовок; иначе прочитайте длину через `stackalloc byte[4]` и `BinaryPrimitives.ReadInt32BigEndian`, проверьте `buffer.Length < 4 + length`, вырежьте payload через `buffer.Slice(4, length).ToArray()` и продвиньте буфер `buffer = buffer.Slice(4 + length)`.
5. Запустите читатель в фоне (`Task readerTask = FrameReaderAsync(pipe.Reader, cts.Token);`), отправьте четыре кадра (`"alpha"`, `"beta"`, `"gamma"`, `"delta"`) коллекционным циклом `foreach (string m in [...])`, завершите писателя `await pipe.Writer.CompleteAsync(ct)` и дождитесь читателя `await readerTask;`. Ожидаемый вывод: четыре строки `B: frame = ...`.
6. **Часть C — адаптер Stream→PipeReader.** Соберите один кадр `byte[] oneFrame = BuildFrame("adapter-demo");`, оберните `new MemoryStream(oneFrame, writable: false)`, создайте `PipeReader pr = PipeReader.Create(adapterStream);`, вызовите `ReadAsync`, разберите кадр через `TryReadFrame` и напечатайте `C: adapter frame = ...`, затем `await pr.CompleteAsync()`.
7. Запустите `dotnet run` и убедитесь, что все три части выполняются без исключений, а вывод совпадает с ожидаемым.

#### Требования к решению
- Целевой фреймворк — `net8.0`, язык C# 12: top-level statements, nullable-контекст, коллекционные выражения для массивов, range-операторы (`[..4]`, `[4..]`), `record` для модели. Сборка `dotnet build` должна проходить без предупреждений.
- Протокол един для всех трёх частей: кадр = 4 байта длины в big-endian + `length` байт UTF-8 payload. Длина читается как знаковое `int`; добавьте защиту `if (length < 0) return false;` от malicious-кадра с отрицательной длиной.
- В части A используйте конструктор `new MemoryStream(capacity: ...)` для записи и `new MemoryStream(byte[], writable: false)` для чтения; не вызывайте `GetBuffer()` на потоке с непубличным буфером (это бросило бы `UnauthorizedAccessException`) — используйте `ToArray()` для копии.
- В части B обязателен вызов `writer.Advance(...)` перед `FlushAsync` и ровно один `reader.AdvanceTo(...)` на каждый `ReadAsync`. Параметр `examined` должен указывать на `buffer.End`, чтобы при неполном кадре конвейер дождался новых данных, а не входил в busy-loop.
- Читатель обязан обрабатывать `result.IsCompleted` и вызывать `reader.Complete()`; писатель завершается через `pipe.Writer.CompleteAsync(ct)`. Используйте `CancellationTokenSource` и передавайте токен во все асинхронные вызовы.
- Никаких `ToArray()` на всей `ReadOnlySequence<byte>` целиком вне разбора одного кадра; разбирайте данные через `Slice` и продвигайте буфер. После `AdvanceTo` считайте любую `Memory<byte>` из буфера невалидной (use-after-return в пул).

#### Тонкости и подводные камни
- **Забытый `Advance`**: если вызвать `GetSpan`, заполнить его, но не вызвать `writer.Advance(bytesWritten)`, то `FlushAsync` не отправит ничего — конвейер не знает, сколько байт актуально. Всегда парный вызов `GetSpan`→`Advance`.
- **Отсутствие `AdvanceTo`**: цикл `ReadAsync` без продвижения буфера заставляет `Buffer` расти бесконечно, а пул — не освобождаться. На каждом проходе обязательно `AdvanceTo(consumed, examined)` ровно один раз.
- **`consumed` vs `examined`**: `consumed` — до какого байта данные обработаны и можно вернуть в пул; `examined` — до какого байта данные осмотрены. Если при неполном кадре поставить `examined = consumed`, конвейер увидит «неосмотренные» данные и тут же перезапустит `ReadAsync` → CPU-спин. Поэтому ставьте `examined = buffer.End` (последний осмотренный байт), чтобы корректно ждать новые данные — это прямо соответствует best practice из урока.
- **`ToArray()` на всей последовательности**: массовая аллокация и копирование; разбирайте сегменты через `Slice`/`CopyTo` в арендованный или `stackalloc`-буфер.
- **`IsCompleted` игнорируется**: цикл чтения висит вечно. Проверяйте флаг и завершайте `reader.Complete()`.
- **`GetBuffer()` на непубличном буфере** бросает `UnauthorizedAccessException` — используйте `ToArray()` или конструктор `new MemoryStream(..., publiclyVisible: true)`.
- **Backpressure**: слишком маленький `pauseWriterThreshold` роняет throughput (писатель постоянно засыпает), слишком большой — расходует память. В демо с быстрым читателем пауза не наблюдается, но конфигурация должна быть осмысленной.
- **use-after-return**: держать ссылку на `Memory<byte>` после `AdvanceTo` — использовать память, уже отданную в пул; считайте её невалидной сразу после продвижения.

#### Критерии приёмки
- [ ] Проект `M10L06.Homework` создаётся командой `dotnet new console` на `net8.0` и собирается без warning'ов.
- [ ] Модель содержит `enum LogLevel` и `record LogEntry`; массив инициализирован коллекционным выражением.
- [ ] Часть A: `SerializeWithMemoryStream` использует `new MemoryStream(capacity: 1024)` и `BinaryWriter`.
- [ ] Часть A: `DeserializeWithMemoryStream` использует `new MemoryStream(blob, writable: false)` и `BinaryReader`.
- [ ] Часть A: round-trip восстанавливает все записи; выводится размер blob и количество записей.
- [ ] Часть B: `Pipe` создан с `ArrayPool<byte>.Shared` и порогами `pauseWriterThreshold: 64`, `resumeWriterThreshold: 32`.
- [ ] Часть B: `FrameWriterAsync` вызывает `GetSpan`→запись→`Advance`→`FlushAsync` в правильном порядке.
- [ ] Часть B: длина записывается big-endian через `BinaryPrimitives.WriteInt32BigEndian`.
- [ ] Часть B: `FrameReaderAsync` вызывает `AdvanceTo` ровно один раз за `ReadAsync` с `examined = buffer.End`.
- [ ] Часть B: `TryReadFrame` корректно обрабатывает неполный заголовок (`< 4`), неполное тело и отрицательную длину.
- [ ] Часть B: цикл чтения завершается по `result.IsCompleted` через `reader.Complete()`.
- [ ] Часть C: `MemoryStream` оборачивается в `PipeReader.Create` и кадр разбирается без ручного копирования.
- [ ] Часть C: вызывается `pr.CompleteAsync()` для освобождения ресурсов адаптера.
- [ ] `dotnet run` выводит строки частей A, B, C без исключений.
- [ ] В коде нет `ToArray()` на всей `ReadOnlySequence` вне разбора одного кадра и нет ссылок на буфер после `AdvanceTo`.

#### Подсказки (без прямого ответа)
- Подумайте, почему `new MemoryStream(capacity: 1024)` лучше `new MemoryStream()` при известном порядке размера данных: вспомните аналогию из урока про «коробку, которую приходится менять на бо́льшую».
- Для big-endian длины используйте `BinaryPrimitives` из `System.Buffers.Binary` — это избавит от ручных сдвигов и ошибок порядка байт.
- В `TryReadFrame` параметр `buffer` передавайте по `ref` и переприсваивайте `buffer = buffer.Slice(4 + length)` — так «наружу» автоматически продвигается позиция потребления.
- Если в части B цикл чтения «съедает» CPU на 100%, проверьте, что `examined` указывает на `buffer.End`, а не на `buffer.Start`.
- Адаптер `PipeReader.Create(stream)` читает из `Stream` как из пайпа — вам не нужно вручную вызывать `stream.Read` и копировать в `PipeWriter`.

#### Эталонное решение (разбор)
```csharp
// C# 12 / .NET 8 — Homework M10-L06 reference solution
// Бинарный протокол кадров с префиксом длины поверх MemoryStream + Pipelines.
// Length-prefixed binary frame protocol over MemoryStream + Pipelines.

using System.Buffers;
using System.Buffers.Binary;
using System.IO;
using System.IO.Pipelines;
using System.Text;

// === Часть A / Part A: MemoryStream round-trip of LogEntry records ===
LogEntry[] entries =
[
    new(DateTimeOffset.UtcNow, LogLevel.Info,  "Service started"),
    new(DateTimeOffset.UtcNow, LogLevel.Warn,  "Queue depth high"),
    new(DateTimeOffset.UtcNow, LogLevel.Error, "Db connection lost"),
];

byte[] blob = SerializeWithMemoryStream(entries);
LogEntry[] roundTripped = DeserializeWithMemoryStream(blob);
Console.WriteLine($"A: serialized {blob.Length} bytes, round-trip {roundTripped.Length} entries");
foreach (LogEntry e in roundTripped)
    Console.WriteLine($"  {e.Timestamp:O} [{e.Level}] {e.Message}");

static byte[] SerializeWithMemoryStream(LogEntry[] entries)
{
    // Подсказываем capacity, чтобы избежать лишних перераспределений.
    // Hint capacity to avoid repeated reallocations.
    using var ms = new MemoryStream(capacity: 1024);
    using var w = new BinaryWriter(ms, Encoding.UTF8, leaveOpen: false);
    w.Write(entries.Length);                              // счётчик / count
    foreach (LogEntry e in entries)
    {
        w.Write(e.Timestamp.ToUnixTimeSeconds());        // 8 байт / bytes
        w.Write((int)e.Level);                           // 4 байта / bytes
        byte[] msg = Encoding.UTF8.GetBytes(e.Message);
        w.Write(msg.Length); w.Write(msg);               // [len][payload]
    }
    return ms.ToArray();                                 // копия, буфер не публичный / copy
}

static LogEntry[] DeserializeWithMemoryStream(byte[] blob)
{
    using var ms = new MemoryStream(blob, writable: false);
    using var r = new BinaryReader(ms, Encoding.UTF8, leaveOpen: false);
    int count = r.ReadInt32();
    var result = new LogEntry[count];
    for (int i = 0; i < count; i++)
    {
        long ts = r.ReadInt64();
        LogLevel level = (LogLevel)r.ReadInt32();
        int len = r.ReadInt32();
        string msg = Encoding.UTF8.GetString(r.ReadBytes(len));
        result[i] = new(DateTimeOffset.FromUnixTimeSeconds(ts), level, msg);
    }
    return result;
}

// === Часть B / Part B: Pipelines frame protocol with backpressure ===
var pipe = new Pipe(new PipeOptions(
    pool: ArrayPool<byte>.Shared,
    pauseWriterThreshold: 64,      // маленький порог — наблюдаем паузу / small to observe pause
    resumeWriterThreshold: 32));

using var cts = new CancellationTokenSource();
CancellationToken ct = cts.Token;

Task readerTask = FrameReaderAsync(pipe.Reader, ct);
foreach (string m in ["alpha", "beta", "gamma", "delta"])
    await FrameWriterAsync(pipe.Writer, m, ct);
await pipe.Writer.CompleteAsync(ct);
await readerTask;

static async Task FrameWriterAsync(PipeWriter writer, string message, CancellationToken ct)
{
    byte[] payload = Encoding.UTF8.GetBytes(message);
    int needed = 4 + payload.Length;
    Span<byte> span = writer.GetSpan(needed);
    BinaryPrimitives.WriteInt32BigEndian(span[..4], payload.Length); // 4 байта BE / 4 BE bytes
    payload.CopyTo(span[4..]);
    writer.Advance(needed);                                // фиксируем запись / commit
    await writer.FlushAsync(ct);                           // может ждать при backpressure
}

static async Task FrameReaderAsync(PipeReader reader, CancellationToken ct)
{
    while (true)
    {
        ReadResult result = await reader.ReadAsync(ct);
        ReadOnlySequence<byte> buffer = result.Buffer;
        while (TryReadFrame(ref buffer, out byte[]? payload))
            Console.WriteLine($"B: frame = {Encoding.UTF8.GetString(payload)}");
        // consumed = начало остатка, examined = конец: ждём ещё данные при неполном кадре.
        // consumed = start of remainder, examined = end: await more data on partial frame.
        reader.AdvanceTo(buffer.Start, buffer.End);
        if (result.IsCompleted) { reader.Complete(); break; }
    }
}

static bool TryReadFrame(ref ReadOnlySequence<byte> buffer, out byte[]? payload)
{
    payload = null;
    if (buffer.Length < 4) return false;                  // неполный заголовок / partial header
    Span<byte> lenBytes = stackalloc byte[4];
    buffer.Slice(0, 4).CopyTo(lenBytes);
    int length = BinaryPrimitives.ReadInt32BigEndian(lenBytes);
    if (length < 0 || buffer.Length < 4 + length) return false; // неполное тело / partial body
    payload = buffer.Slice(4, length).ToArray();
    buffer = buffer.Slice(4 + length);                    // продвигаем буфер / advance
    return true;
}

// === Часть C / Part C: Stream -> PipeReader adapter ===
byte[] oneFrame = BuildFrame("adapter-demo");
await using var adapterStream = new MemoryStream(oneFrame, writable: false);
PipeReader pr = PipeReader.Create(adapterStream);
ReadResult rr = await pr.ReadAsync();
if (TryReadFrame(ref rr.Buffer, out byte[]? p))
    Console.WriteLine($"C: adapter frame = {Encoding.UTF8.GetString(p)}");
await pr.CompleteAsync();

static byte[] BuildFrame(string message)
{
    byte[] payload = Encoding.UTF8.GetBytes(message);
    byte[] frame = new byte[4 + payload.Length];
    BinaryPrimitives.WriteInt32BigEndian(frame.AsSpan(..4), payload.Length);
    payload.CopyTo(frame.AsSpan(4..));
    return frame;
}

enum LogLevel { Info, Warn, Error }
record LogEntry(DateTimeOffset Timestamp, LogLevel Level, string Message);
```

Разбор по строкам. Часть A использует `new MemoryStream(capacity: 1024)` — это best practice урока: размер известен приблизительно, подсказка capacity спасает от лишних перераспределений (аналогия «коробка большего размера»). `BinaryWriter` поверх потока удобен для записи примитивов; `leaveOpen: false` позволяет `using` закрыть и поток, и writer. Возврат `ms.ToArray()` делает копию, потому что буфер `MemoryStream`, созданный через конструктор с capacity, не публично видим — вызов `GetBuffer()` здесь бросил бы `UnauthorizedAccessException` (частая ошибка из урока). Десериализация открывает `new MemoryStream(blob, writable: false)` — read-only обёртка над существующим массивом без копирования.

Часть B — ядро урока. `PipeOptions` явно задаёт `ArrayPool<byte>.Shared` и пороги backpressure. `FrameWriterAsync` строго следует протоколу `GetSpan`→запись→`Advance`→`FlushAsync`: без `Advance` `FlushAsync` отправил бы ноль байт (типичная ошибка). Длина пишется big-endian через `BinaryPrimitives.WriteInt32BigEndian(span[..4], ...)` с range-оператором C# 12. `FlushAsync` принимает `CancellationToken` и может приостановиться при превышении `pauseWriterThreshold`. В `FrameReaderAsync` ключевой момент — `reader.AdvanceTo(buffer.Start, buffer.End)`: `buffer` уже продвинут внутри `TryReadFrame` (через `ref` и `Slice`), поэтому `buffer.Start` указывает на начало непрочитанного остатка (всё разобранное помечено потреблённым), а `buffer.End` — на последний осмотренный байт. Это правильное применение best practice «examined = последний осмотренный байт»: при неполном кадре конвейер дождётся новых данных, а не войдёт в busy-loop (как было бы при `examined = buffer.Start`). Проверка `result.IsCompleted` и вызов `reader.Complete()` предотвращают вечное зависание цикла. `TryReadFrame` обрабатывает три случая: меньше 4 байт (неполный заголовок), отрицательная длина (защита от malicious-кадра) и неполное тело — везде возвращается `false`, буфер не продвигается, и читатель ждёт ещё данных.

Часть C демонстрирует адаптер `PipeReader.Create(stream)`: `MemoryStream` читается как пайп, тот же `TryReadFrame` работает без изменений — код разбора переиспользуется. `pr.CompleteAsync()` освобождает ресурсы адаптера. Всё решение — top-level statements, коллекционные выражения, range-операторы, `record`, nullable-аннотации `byte[]?` — соответствует C# 12 / .NET 8.

#### Задания на углубление (бонус)
1. Сделайте читатель медленным: добавьте `await Task.Delay(50, ct)` перед `ReadAsync` и уменьшите `pauseWriterThreshold` до 16. Логируйте моменты, когда `FrameWriterAsync` «зависает» на `FlushAsync`, и объясните, как `resumeWriterThreshold` возобновляет писателя.
2. Замените `ToArray()` в `TryReadFrame` на аренду `ArrayPool<byte>.Shared.Rent(length)` с последующим `Return` после печати — и измерьте allocations через `dotnet-counters` в сценарии из 10 000 кадров.
3. Реализуйте `Stream`-адаптер в обратную сторону: оберните `PipeWriter` в `Stream` через `writer.AsStream()` и запишите в него кадр обычным `stream.Write`.
4. Добавьте второй кадр в часть C и убедитесь, что один `ReadAsync` может вернуть сразу несколько кадров; объясните, почему `ReadOnlySequence` из нескольких сегментов не требует склейки в один массив.

---

## Statement in English / Постановка на английском

#### Context & motivation
You are prototyping a compact in-process message broker for a training logging service. Messages must be packed into bytes, transported over a channel, and parsed on the receiving side. The channel may deliver data in arbitrary chunks: sometimes a whole frame arrives at once, sometimes only the first few bytes of the header show up and the body arrives later. This situation is typical for network protocols (HTTP, WebSocket, RESP, binary frames) and demands a parser that is resilient to partial frames.

Earlier in module M10 you worked with `FileStream` and buffered streams, and in this lesson you met two in-memory tools: the classic `MemoryStream` and the modern `System.IO.Pipelines`. `MemoryStream` is simpler: its backing store is a plain `byte[]` in the managed heap, it grows automatically, supports both synchronous and asynchronous operations, and exposes the internal buffer through `GetBuffer()` (or a copy through `ToArray()`). But simplicity is paid for in copies: every `Read`/`Write` copies bytes, and growing the buffer allocates a new array and copies the old contents. For high-throughput hot paths (Kestrel, network parsers, crypto pipelines) that is expensive.

`System.IO.Pipelines` is different: the writer rents a region from `ArrayPool<byte>` via `GetSpan`/`GetMemory`, fills it, commits the written bytes through `Advance`, and pushes them through `FlushAsync`; the reader, via `ReadAsync`, receives a `ReadResult` whose `Buffer` is a `ReadOnlySequence<byte>` (possibly composed of several non-contiguous segments), and after processing calls `AdvanceTo` to return memory to the pool. The pipe has built-in backpressure: when `pauseWriterThreshold` is exceeded, the writer's `FlushAsync` suspends until the reader drains the buffer below `resumeWriterThreshold`. The goal of this homework is to feel both models on the very same protocol and learn to choose the right tool deliberately.

#### What to do step by step
1. Create a .NET 8 console project: `dotnet new console -n M10L06.Homework -o M10L06.Homework --framework net8.0`, then `cd M10L06.Homework` and open `Program.cs`. Make sure the `.csproj` has `<Nullable>enable</Nullable>` and `<ImplicitUsings>enable</ImplicitUsings>`.
2. Define the data model: an enum `LogLevel { Info, Warn, Error }` and a record `record LogEntry(DateTimeOffset Timestamp, LogLevel Level, string Message);`. Use a collection expression to initialize an array of three records.
3. **Part A — MemoryStream.** Implement two methods: `byte[] SerializeWithMemoryStream(LogEntry[] entries)` and `LogEntry[] DeserializeWithMemoryStream(byte[] blob)`. Serialization: open `new MemoryStream(capacity: 1024)` (a size hint to avoid extra reallocations), wrap it in a `BinaryWriter`, write the record count (`int`), then for each record write `Timestamp.ToUnixTimeSeconds()` (`long`), `(int)Level`, the message length (`int`), and the UTF-8 bytes of the message. Return `ms.ToArray()`. Deserialization: open `new MemoryStream(blob, writable: false)`, wrap in a `BinaryReader`, read the count, and restore the records in a loop. Print the blob size and the number of restored records.
4. **Part B — Pipelines.** Create a `Pipe` with `PipeOptions(pool: ArrayPool<byte>.Shared, pauseWriterThreshold: 64, resumeWriterThreshold: 32)`. Implement `async Task FrameWriterAsync(PipeWriter writer, string message, CancellationToken ct)`: get `payload = Encoding.UTF8.GetBytes(message)`, request `writer.GetSpan(4 + payload.Length)`, write the length big-endian via `BinaryPrimitives.WriteInt32BigEndian(span[..4], payload.Length)`, copy the payload into `span[4..]`, call `writer.Advance(4 + payload.Length)` and `await writer.FlushAsync(ct)`. Implement `async Task FrameReaderAsync(PipeReader reader, CancellationToken ct)`: loop `await reader.ReadAsync(ct)`, get `result.Buffer`, in a loop call `TryReadFrame(ref buffer, out byte[]? payload)` printing each frame, then call `reader.AdvanceTo(buffer.Start, buffer.End)` (important: `examined` points to the end of available data so the pipe correctly awaits more data on a partial frame), check `result.IsCompleted` and finish via `reader.Complete()`. Implement `bool TryReadFrame(ref ReadOnlySequence<byte> buffer, out byte[]? payload)`: if fewer than 4 bytes — partial header; otherwise read the length via `stackalloc byte[4]` and `BinaryPrimitives.ReadInt32BigEndian`, check `buffer.Length < 4 + length`, slice the payload via `buffer.Slice(4, length).ToArray()` and advance the buffer with `buffer = buffer.Slice(4 + length)`.
5. Start the reader in the background (`Task readerTask = FrameReaderAsync(pipe.Reader, cts.Token);`), send four frames (`"alpha"`, `"beta"`, `"gamma"`, `"delta"`) with a collection loop `foreach (string m in [...])`, complete the writer with `await pipe.Writer.CompleteAsync(ct)` and await the reader `await readerTask;`. Expected output: four lines `B: frame = ...`.
6. **Part C — Stream→PipeReader adapter.** Build one frame `byte[] oneFrame = BuildFrame("adapter-demo");`, wrap `new MemoryStream(oneFrame, writable: false)`, create `PipeReader pr = PipeReader.Create(adapterStream);`, call `ReadAsync`, parse the frame via `TryReadFrame` and print `C: adapter frame = ...`, then `await pr.CompleteAsync()`.
7. Run `dotnet run` and make sure all three parts execute without exceptions and the output matches expectations.

#### Requirements
- Target framework `net8.0`, language C# 12: top-level statements, nullable context, collection expressions for arrays, range operators (`[..4]`, `[4..]`), `record` for the model. `dotnet build` must pass without warnings.
- The protocol is identical across all three parts: a frame is 4 bytes of big-endian length followed by `length` bytes of UTF-8 payload. The length is read as a signed `int`; add a guard `if (length < 0) return false;` against a malicious frame with a negative length.
- In Part A use `new MemoryStream(capacity: ...)` for writing and `new MemoryStream(byte[], writable: false)` for reading; do not call `GetBuffer()` on a stream with a non-public buffer (it would throw `UnauthorizedAccessException`) — use `ToArray()` for a copy.
- In Part B a `writer.Advance(...)` before `FlushAsync` and exactly one `reader.AdvanceTo(...)` per `ReadAsync` are mandatory. The `examined` argument must point to `buffer.End` so that, on a partial frame, the pipe waits for more data instead of entering a busy-loop.
- The reader must handle `result.IsCompleted` and call `reader.Complete()`; the writer is completed through `pipe.Writer.CompleteAsync(ct)`. Use a `CancellationTokenSource` and pass the token to every asynchronous call.
- No `ToArray()` on the whole `ReadOnlySequence<byte>` outside parsing a single frame; parse data with `Slice` and advance the buffer. After `AdvanceTo`, treat any `Memory<byte>` from the buffer as invalid (use-after-return to the pool).

#### Pitfalls
- **Forgotten `Advance`**: if you call `GetSpan`, fill it, but skip `writer.Advance(bytesWritten)`, then `FlushAsync` sends nothing — the pipe does not know how many bytes are actual. Always pair `GetSpan` with `Advance`.
- **Missing `AdvanceTo`**: a `ReadAsync` loop without advancing the buffer makes `Buffer` grow forever and the pool never gets released. On every iteration call `AdvanceTo(consumed, examined)` exactly once.
- **`consumed` vs `examined`**: `consumed` is up to which byte data is processed and can be returned to the pool; `examined` is up to which byte data has been inspected. If on a partial frame you set `examined = consumed`, the pipe sees "unexamined" data and immediately restarts `ReadAsync` → CPU spin. Therefore set `examined = buffer.End` (the last examined byte) so the pipe correctly awaits new data — this matches the lesson's best practice directly.
- **`ToArray()` on the whole sequence**: a large allocation and copy; parse segments with `Slice`/`CopyTo` into a rented or `stackalloc` buffer.
- **`IsCompleted` ignored**: the read loop hangs forever. Check the flag and call `reader.Complete()`.
- **`GetBuffer()` on a non-public buffer** throws `UnauthorizedAccessException` — use `ToArray()` or the constructor with `publiclyVisible: true`.
- **Backpressure**: too small a `pauseWriterThreshold` kills throughput (the writer keeps sleeping), too large wastes memory. With a fast reader the demo will not pause, but the configuration must be deliberate.
- **use-after-return**: holding a `Memory<byte>` reference after `AdvanceTo` means using memory already returned to the pool; treat it as invalid immediately after advancing.

#### Acceptance criteria
- [ ] The `M10L06.Homework` project is created with `dotnet new console` on `net8.0` and builds without warnings.
- [ ] The model contains `enum LogLevel` and `record LogEntry`; the array is initialized with a collection expression.
- [ ] Part A: `SerializeWithMemoryStream` uses `new MemoryStream(capacity: 1024)` and a `BinaryWriter`.
- [ ] Part A: `DeserializeWithMemoryStream` uses `new MemoryStream(blob, writable: false)` and a `BinaryReader`.
- [ ] Part A: the round-trip restores all records; the blob size and record count are printed.
- [ ] Part B: the `Pipe` is created with `ArrayPool<byte>.Shared` and thresholds `pauseWriterThreshold: 64`, `resumeWriterThreshold: 32`.
- [ ] Part B: `FrameWriterAsync` calls `GetSpan`→write→`Advance`→`FlushAsync` in the correct order.
- [ ] Part B: the length is written big-endian via `BinaryPrimitives.WriteInt32BigEndian`.
- [ ] Part B: `FrameReaderAsync` calls `AdvanceTo` exactly once per `ReadAsync` with `examined = buffer.End`.
- [ ] Part B: `TryReadFrame` correctly handles a partial header (`< 4`), a partial body, and a negative length.
- [ ] Part B: the read loop terminates on `result.IsCompleted` via `reader.Complete()`.
- [ ] Part C: the `MemoryStream` is wrapped with `PipeReader.Create` and the frame is parsed without manual copying.
- [ ] Part C: `pr.CompleteAsync()` is called to release the adapter's resources.
- [ ] `dotnet run` prints the lines of parts A, B, C without exceptions.
- [ ] The code has no `ToArray()` on the whole `ReadOnlySequence` outside parsing a single frame and no references to the buffer after `AdvanceTo`.

#### Hints (no direct answer)
- Think about why `new MemoryStream(capacity: 1024)` is better than `new MemoryStream()` when the rough size is known: recall the lesson's analogy of "a box you have to replace with a bigger one".
- For the big-endian length use `BinaryPrimitives` from `System.Buffers.Binary` — it removes manual shifts and byte-order bugs.
- In `TryReadFrame` pass `buffer` by `ref` and reassign `buffer = buffer.Slice(4 + length)` — that automatically advances the consumption position "outside".
- If in Part B the read loop eats 100% CPU, check that `examined` points to `buffer.End`, not to `buffer.Start`.
- The `PipeReader.Create(stream)` adapter reads from a `Stream` as if it were a pipe — you do not need to call `stream.Read` manually and copy into a `PipeWriter`.

#### Reference solution walk-through
```csharp
// C# 12 / .NET 8 — Homework M10-L06 reference solution
// Length-prefixed binary frame protocol over MemoryStream + Pipelines.

using System.Buffers;
using System.Buffers.Binary;
using System.IO;
using System.IO.Pipelines;
using System.Text;

// === Part A: MemoryStream round-trip of LogEntry records ===
LogEntry[] entries =
[
    new(DateTimeOffset.UtcNow, LogLevel.Info,  "Service started"),
    new(DateTimeOffset.UtcNow, LogLevel.Warn,  "Queue depth high"),
    new(DateTimeOffset.UtcNow, LogLevel.Error, "Db connection lost"),
];

byte[] blob = SerializeWithMemoryStream(entries);
LogEntry[] roundTripped = DeserializeWithMemoryStream(blob);
Console.WriteLine($"A: serialized {blob.Length} bytes, round-trip {roundTripped.Length} entries");
foreach (LogEntry e in roundTripped)
    Console.WriteLine($"  {e.Timestamp:O} [{e.Level}] {e.Message}");

static byte[] SerializeWithMemoryStream(LogEntry[] entries)
{
    // Hint capacity to avoid repeated reallocations.
    using var ms = new MemoryStream(capacity: 1024);
    using var w = new BinaryWriter(ms, Encoding.UTF8, leaveOpen: false);
    w.Write(entries.Length);                              // count
    foreach (LogEntry e in entries)
    {
        w.Write(e.Timestamp.ToUnixTimeSeconds());        // 8 bytes
        w.Write((int)e.Level);                           // 4 bytes
        byte[] msg = Encoding.UTF8.GetBytes(e.Message);
        w.Write(msg.Length); w.Write(msg);               // [len][payload]
    }
    return ms.ToArray();                                 // copy, buffer not publicly visible
}

static LogEntry[] DeserializeWithMemoryStream(byte[] blob)
{
    using var ms = new MemoryStream(blob, writable: false);
    using var r = new BinaryReader(ms, Encoding.UTF8, leaveOpen: false);
    int count = r.ReadInt32();
    var result = new LogEntry[count];
    for (int i = 0; i < count; i++)
    {
        long ts = r.ReadInt64();
        LogLevel level = (LogLevel)r.ReadInt32();
        int len = r.ReadInt32();
        string msg = Encoding.UTF8.GetString(r.ReadBytes(len));
        result[i] = new(DateTimeOffset.FromUnixTimeSeconds(ts), level, msg);
    }
    return result;
}

// === Part B: Pipelines frame protocol with backpressure ===
var pipe = new Pipe(new PipeOptions(
    pool: ArrayPool<byte>.Shared,
    pauseWriterThreshold: 64,      // small threshold to observe the pause
    resumeWriterThreshold: 32));

using var cts = new CancellationTokenSource();
CancellationToken ct = cts.Token;

Task readerTask = FrameReaderAsync(pipe.Reader, ct);
foreach (string m in ["alpha", "beta", "gamma", "delta"])
    await FrameWriterAsync(pipe.Writer, m, ct);
await pipe.Writer.CompleteAsync(ct);
await readerTask;

static async Task FrameWriterAsync(PipeWriter writer, string message, CancellationToken ct)
{
    byte[] payload = Encoding.UTF8.GetBytes(message);
    int needed = 4 + payload.Length;
    Span<byte> span = writer.GetSpan(needed);
    BinaryPrimitives.WriteInt32BigEndian(span[..4], payload.Length); // 4 BE bytes
    payload.CopyTo(span[4..]);
    writer.Advance(needed);                                // commit the write
    await writer.FlushAsync(ct);                           // may await on backpressure
}

static async Task FrameReaderAsync(PipeReader reader, CancellationToken ct)
{
    while (true)
    {
        ReadResult result = await reader.ReadAsync(ct);
        ReadOnlySequence<byte> buffer = result.Buffer;
        while (TryReadFrame(ref buffer, out byte[]? payload))
            Console.WriteLine($"B: frame = {Encoding.UTF8.GetString(payload)}");
        // consumed = start of remainder, examined = end: await more data on a partial frame.
        reader.AdvanceTo(buffer.Start, buffer.End);
        if (result.IsCompleted) { reader.Complete(); break; }
    }
}

static bool TryReadFrame(ref ReadOnlySequence<byte> buffer, out byte[]? payload)
{
    payload = null;
    if (buffer.Length < 4) return false;                  // partial header
    Span<byte> lenBytes = stackalloc byte[4];
    buffer.Slice(0, 4).CopyTo(lenBytes);
    int length = BinaryPrimitives.ReadInt32BigEndian(lenBytes);
    if (length < 0 || buffer.Length < 4 + length) return false; // partial body
    payload = buffer.Slice(4, length).ToArray();
    buffer = buffer.Slice(4 + length);                    // advance the buffer
    return true;
}

// === Part C: Stream -> PipeReader adapter ===
byte[] oneFrame = BuildFrame("adapter-demo");
await using var adapterStream = new MemoryStream(oneFrame, writable: false);
PipeReader pr = PipeReader.Create(adapterStream);
ReadResult rr = await pr.ReadAsync();
if (TryReadFrame(ref rr.Buffer, out byte[]? p))
    Console.WriteLine($"C: adapter frame = {Encoding.UTF8.GetString(p)}");
await pr.CompleteAsync();

static byte[] BuildFrame(string message)
{
    byte[] payload = Encoding.UTF8.GetBytes(message);
    byte[] frame = new byte[4 + payload.Length];
    BinaryPrimitives.WriteInt32BigEndian(frame.AsSpan(..4), payload.Length);
    payload.CopyTo(frame.AsSpan(4..));
    return frame;
}

enum LogLevel { Info, Warn, Error }
record LogEntry(DateTimeOffset Timestamp, LogLevel Level, string Message);
```

Line-by-line walk-through. Part A uses `new MemoryStream(capacity: 1024)` — this is the lesson's best practice: the rough size is known, and the capacity hint saves extra reallocations (the "bigger box" analogy). `BinaryWriter` over the stream is convenient for writing primitives; `leaveOpen: false` lets `using` close both the writer and the stream. Returning `ms.ToArray()` makes a copy because the buffer of a `MemoryStream` created through the capacity constructor is not publicly visible — calling `GetBuffer()` here would throw `UnauthorizedAccessException` (a common mistake from the lesson). Deserialization opens `new MemoryStream(blob, writable: false)` — a read-only wrapper over an existing array with no copying.

Part B is the core of the lesson. `PipeOptions` explicitly sets `ArrayPool<byte>.Shared` and the backpressure thresholds. `FrameWriterAsync` strictly follows the `GetSpan`→write→`Advance`→`FlushAsync` protocol: without `Advance`, `FlushAsync` would send zero bytes (a typical mistake). The length is written big-endian via `BinaryPrimitives.WriteInt32BigEndian(span[..4], ...)` using a C# 12 range operator. `FlushAsync` takes a `CancellationToken` and may suspend when `pauseWriterThreshold` is exceeded. In `FrameReaderAsync` the key line is `reader.AdvanceTo(buffer.Start, buffer.End)`: `buffer` has already been advanced inside `TryReadFrame` (via `ref` and `Slice`), so `buffer.Start` points to the start of the unread remainder (everything parsed is marked consumed), and `buffer.End` is the last examined byte. This is the correct application of the best practice "examined = last examined byte": on a partial frame the pipe waits for more data instead of busy-looping (as it would with `examined = buffer.Start`). The `result.IsCompleted` check and `reader.Complete()` prevent the loop from hanging forever. `TryReadFrame` handles three cases: fewer than 4 bytes (partial header), a negative length (guard against a malicious frame), and a partial body — in all of them it returns `false`, does not advance the buffer, and the reader waits for more data.

Part C demonstrates the `PipeReader.Create(stream)` adapter: the `MemoryStream` is read as a pipe, and the very same `TryReadFrame` works unchanged — the parsing code is reused. `pr.CompleteAsync()` releases the adapter's resources. The whole solution uses top-level statements, collection expressions, range operators, a `record`, and nullable `byte[]?` annotations — fully aligned with C# 12 / .NET 8.

#### Going deeper (bonus)
1. Make the reader slow: add `await Task.Delay(50, ct)` before `ReadAsync` and lower `pauseWriterThreshold` to 16. Log the moments when `FrameWriterAsync` "hangs" on `FlushAsync` and explain how `resumeWriterThreshold` resumes the writer.
2. Replace `ToArray()` in `TryReadFrame` with an `ArrayPool<byte>.Shared.Rent(length)` rental and a `Return` after printing — then measure allocations with `dotnet-counters` over a 10 000-frame scenario.
3. Implement the adapter in the opposite direction: wrap a `PipeWriter` into a `Stream` via `writer.AsStream()` and write a frame with a plain `stream.Write`.
4. Add a second frame in Part C and confirm that a single `ReadAsync` may return several frames at once; explain why a multi-segment `ReadOnlySequence` does not require concatenation into a single array.

---

#### Чек-лист сдачи / Submission checklist
- [ ] (RU) Проект `M10L06.Homework` создан на `net8.0` и собирается без предупреждений.
- [ ] (RU) Реализованы все три части: MemoryStream round-trip, Pipelines-протокол, адаптер Stream→PipeReader.
- [ ] (RU) В части B `AdvanceTo` вызывается с `examined = buffer.End`, `IsCompleted` обрабатывается.
- [ ] (RU) Код использует коллекционные выражения, range-операторы, `record`, top-level statements.
- [ ] (RU) `dotnet run` выводит ожидаемые строки без исключений.
- [ ] (EN) The `M10L06.Homework` project is created on `net8.0` and builds without warnings.
- [ ] (EN) All three parts are implemented: MemoryStream round-trip, the Pipelines protocol, and the Stream→PipeReader adapter.
- [ ] (EN) In Part B `AdvanceTo` is called with `examined = buffer.End` and `IsCompleted` is handled.
- [ ] (EN) The code uses collection expressions, range operators, a `record`, and top-level statements.
- [ ] (EN) `dotnet run` prints the expected lines without exceptions.

#### Ресурсы / Resources
- [Microsoft Learn — System.IO.Pipelines — https://learn.microsoft.com/dotnet/standard/io/pipelines](https://learn.microsoft.com/dotnet/standard/io/pipelines)
- [Microsoft Learn — MemoryStream — https://learn.microsoft.com/dotnet/api/system.io.memorystream](https://learn.microsoft.com/dotnet/api/system.io.memorystream)
- [Microsoft Learn — ArrayPool\<T\> — https://learn.microsoft.com/dotnet/api/system.buffers.arraypool-1](https://learn.microsoft.com/dotnet/api/system.buffers.arraypool-1)
- [Microsoft Learn — ReadOnlySequence\<T\> — https://learn.microsoft.com/dotnet/api/system.buffers.readonlysequence-1](https://learn.microsoft.com/dotnet/api/system.buffers.readonlysequence-1)
- [.NET GitHub — Pipelines samples — https://github.com/dotnet/aspnetcore/tree/main/src/Servers](https://github.com/dotnet/aspnetcore/tree/main/src/Servers)

---

[⬆ К модулю M10](../README.md) | [Следующее ДЗ →](homework-M10-L07-xml-serialization.md)
