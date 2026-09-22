[⬆ К модулю M10](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M10-L06: MemoryStream, PipeReader/PipeWriter (обзор) / MemoryStream, PipeReader/PipeWriter (overview)

**Модуль / Module:** M10
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

В модуле M10 мы уже работали с `FileStream` и буферизованными потоками. Но в реальных задачах часто нужно держать данные полностью в памяти — например, сериализовать объект в массив байт, собрать ответ веб-API или проксировать данные между двумя каналами. Для таких сценариев предназначен `MemoryStream` — это реализация `Stream`, чьим «хранилищем» служит обычный `byte[]` (или `ReadOnlySegment<byte>`), живущий в управляемой куче.

`MemoryStream` удобен: он поддерживает синхронные и асинхронные операции, позволяет читать и писать одновременно, дает прямой доступ к внутреннему буферу через `GetBuffer()` (если буфер не публичный — через `ToArray()`), а также умеет расти автоматически при переполнении. Однако у него есть цена. Каждый `Read`/`Write` копирует байты, а рост буфера сопровождается аллокацией нового массива и копированием старого содержимого. Аналогия: `MemoryStream` — это коробка, в которую вы складываете вещи; когда коробка заполняется, вы покупаете коробку побольше и перекладываете всё вручную.

Для большинства прикладных задач этого достаточно. Но высоконагруженные серверные сценарии (Kestrel, сетевые парсеры, крипто-конвейеры) требуют минимизации аллокаций и копирований. Здесь на сцену выходит `System.IO.Pipelines` — модель, построенная вокруг двух структур: `PipeWriter` (пишущий конец) и `PipeReader` (читающий конец), соединённых общим `Pipe`.

Ключевая идея `Pipelines` — **пул буферов** (`ArrayPool<byte>`) и **семантика аренды**. Писатель запрашивает у пула регион памяти через `GetSpan`/`GetMemory`, заполняет его, сообщает `Advance`, а затем `FlushAsync`. Читатель через `ReadAsync` получает `ReadResult` с `Buffer` (`ReadOnlySequence<byte>`) — это может быть несколько несмежных сегментов. После обработки читатель вызывает `AdvanceTo`, помечая, какую часть можно вернуть в пул. Аналогия: вместо одной большой коробки у вас лента конвейера с ячейками — вы берёте ячейку, кладёте деталь, отправляете дальше; получатель забирает деталь и возвращает ячейку на конвейер.

Важнейшее свойство конвейера — **backpressure (обратное давление)**. У `Pipe` есть параметры `pauseWriterThreshold` и `resumeWriterThreshold`. Если накоплено непрочитанных данных больше порога паузы, `FlushAsync` писателя «зависает» и не завершается, пока читатель не опустошит буфер ниже порога возобновления. Это естественным образом замедляет производителя, не давая ему захлебнуть потребителя — без ручного создания `SemaphoreSlim` или `Channel`. У классического `Stream` такого механизма нет: запись в медленный поток блокирует поток выполнения, но не координирует скорости читателя и писателя через явные пороги.

Главные отличия `Pipelines` от `Stream`:

- `Pipelines` работают с `Memory<byte>`/`Sequence`, а не с массивом — меньше копирований.
- Явный контроль потребления через `AdvanceTo(consumed, examined)` — читатель сообщает, до какого байта данные «обработаны» и до какого — «осмотрены», что позволяет корректно ждать новых данных при частичных сообщениях.
- Встроенный backpressure.
- `ReadOnlySequence` допускает несколько сегментов — не нужно склеивать данные в один массив при разборе протоколов.
- `PipeReader`/`PipeWriter` можно адаптировать к `Stream` через методы расширения `AsStream()`, а `Stream` — к конвейеру через `PipeReader.Create(stream)`.

Практическое правило: используйте `MemoryStream`, когда данные реально умещаются в памяти и важна простота API; переходите на `Pipelines`, когда парсите потоковый протокол по частям (HTTP, WebSocket, RESP, бинарные кадры) и хотите избежать лишних аллокаций в горячем пути.

#### Theory (EN)

Earlier in module M10 we worked with `FileStream` and buffered streams. Yet many real tasks require holding data entirely in memory — serializing an object to a byte array, assembling a web API response, or proxying bytes between two channels. For these scenarios the BCL offers `MemoryStream`, a `Stream` implementation whose backing store is a plain `byte[]` (or a `ReadOnlySegment<byte>`) living in the managed heap.

`MemoryStream` is convenient: it supports both synchronous and asynchronous operations, allows simultaneous reading and writing, exposes the internal buffer via `GetBuffer()` (or `ToArray()` when the buffer is not publicly visible), and grows automatically when it overflows. The cost, however, is real. Every `Read`/`Write` copies bytes, and growing the buffer allocates a new array and copies the old contents. Analogy: `MemoryStream` is a box into which you keep piling items; when the box is full you buy a bigger one and relocate everything by hand.

For most application-level work that is perfectly acceptable. But high-throughput server scenarios (Kestrel, network parsers, crypto pipelines) demand minimal allocations and copies. Enter `System.IO.Pipelines` — a model built around two types: `PipeWriter` (the writing end) and `PipeReader` (the reading end), connected by a shared `Pipe`.

The core idea of `Pipelines` is a **buffer pool** (`ArrayPool<byte>`) and **rental semantics**. The writer rents a region from the pool via `GetSpan`/`GetMemory`, fills it, calls `Advance`, then `FlushAsync`. The reader, through `ReadAsync`, receives a `ReadResult` whose `Buffer` is a `ReadOnlySequence<byte>` — possibly composed of several non-contiguous segments. After processing, the reader calls `AdvanceTo`, marking which portion may be returned to the pool. Analogy: instead of one big box you have a conveyor belt of slots — you take a slot, place a part, send it on; the receiver takes the part and returns the slot to the belt.

The most important property of a pipe is **backpressure**. A `Pipe` accepts `pauseWriterThreshold` and `resumeWriterThreshold`. If the amount of unread data exceeds the pause threshold, the writer's `FlushAsync` suspends and does not complete until the reader drains the buffer below the resume threshold. This naturally slows the producer, preventing it from drowning the consumer — with no manual `SemaphoreSlim` or `Channel` required. A classic `Stream` has no such mechanism: writing to a slow stream blocks the executing thread, but it does not coordinate reader/writer speeds through explicit thresholds.

Key differences between `Pipelines` and `Stream`:

- `Pipelines` operate on `Memory<byte>`/`Sequence` rather than arrays — fewer copies.
- Explicit consumption control via `AdvanceTo(consumed, examined)` — the reader reports up to which byte data is "processed" and up to which it is "examined", which lets it correctly await more data for partial messages.
- Built-in backpressure.
- `ReadOnlySequence` supports multiple segments — no need to concatenate data into a single array when parsing protocols.
- `PipeReader`/`PipeWriter` can be adapted to `Stream` via `AsStream()`, and a `Stream` can be wrapped as a pipe via `PipeReader.Create(stream)`.

Practical rule: use `MemoryStream` when the data genuinely fits in memory and API simplicity matters; move to `Pipelines` when you are parsing a streaming protocol in fragments (HTTP, WebSocket, RESP, binary frames) and want to avoid hot-path allocations.

#### Пример кода / Code Example

```csharp
// C# 12+ / .NET 8+
// Демонстрация MemoryStream и System.IO.Pipelines с backpressure.
// Demonstrates MemoryStream and System.IO.Pipelines with backpressure.

using System.Buffers;
using System.IO.Pipelines;
using System.Text;

// === 1) MemoryStream: простой сценарий «всё в памяти» ===
// === 1) MemoryStream: simple "everything in memory" scenario ===
using var ms = new MemoryStream();
ms.Write(Encoding.UTF8.GetBytes("Hello, "));        // пишем / write
ms.Write(Encoding.UTF8.GetBytes("world!"));
ms.Position = 0;                                     // перемотка к началу / rewind to start
byte[] bytes = ms.ToArray();                         // копия буфера / copy of buffer
Console.WriteLine(Encoding.UTF8.GetString(bytes));  // Hello, world!

// Прямой доступ к буферу без копирования (если buffer publicly visible):
// Direct buffer access without copying (when buffer is publicly visible):
using var ms2 = new MemoryStream(capacity: 128);
ms2.Write(bytes, 0, bytes.Length);
ArraySegment<byte> segment = ms2.GetBuffer();        // ссылка, не копия / reference, not copy
Console.WriteLine($"Buffer length: {segment.Count}");

// === 2) Pipelines: аренда буфера, partial-frame parsing, backpressure ===
// === 2) Pipelines: buffer rental, partial-frame parsing, backpressure ===
// Pipe с порогами backpressure / a pipe with backpressure thresholds.
var pipe = new Pipe(new PipeOptions(
    pool: ArrayPool<byte>.Shared,
    pauseWriterThreshold: 1024,    // писатель остановится здесь / writer pauses here
    resumeWriterThreshold: 512));  // писатель возобновится здесь / writer resumes here

// Протокол: кадр = [length:4 bytes BE][payload:length bytes].
// Protocol: frame = [length:4 bytes BE][payload:length bytes].
async Task WriterAsync(PipeWriter writer, string message, CancellationToken ct)
{
    byte[] payload = Encoding.UTF8.GetBytes(message);
    int length = payload.Length;

    // Запрашиваем у пула连续ную память / request contiguous memory from the pool.
    Span<byte> span = writer.GetSpan(4 + length);
    BitConverter.TryWriteBytes(span[..4], length);     // 4 байта длины / 4 length bytes (BE-agnostic demo)
    payload.CopyTo(span[4..]);
    writer.Advance(4 + length);                        // фиксируем запись / commit the write
    await writer.FlushAsync(ct);                       // может ждать при backpressure / may await on backpressure
}

async Task ReaderAsync(PipeReader reader, CancellationToken ct)
{
    while (!ct.IsCancellationRequested)
    {
        ReadResult result = await reader.ReadAsync(ct);
        ReadOnlySequence<byte> buffer = result.Buffer;

        while (TryReadFrame(ref buffer, out byte[]? payload))
        {
            Console.WriteLine($"Frame: {Encoding.UTF8.GetString(payload)}");
        }

        // Сообщаем пайпу, что обработано и осмотрено / tell the pipe what was consumed and examined.
        // Если кадр неполный — examined = buffer.Start, ждём ещё данных.
        // If the frame is partial — examined = buffer.Start, await more data.
        reader.AdvanceTo(buffer.Start, buffer.Start);

        if (result.IsCompleted)
        {
            reader.Complete();
            break;
        }
    }
}

// Чтение одного кадра из последовательности / read a single frame from the sequence.
bool TryReadFrame(ref ReadOnlySequence<byte> buffer, out byte[]? payload)
{
    if (buffer.Length < 4) { payload = null; return false; }           // мало данных для длины / not enough for length

    Span<byte> lenBytes = stackalloc byte[4];
    buffer.Slice(0, 4).CopyTo(lenBytes);
    int length = BitConverter.ToInt32(lenBytes);

    if (buffer.Length < 4 + length) { payload = null; return false; }  // тело не пришло целиком / body not fully here

    payload = buffer.Slice(4, length).ToArray();
    buffer = buffer.Slice(4 + length);                                 // продвигаем буфер / advance the buffer
    return true;
}

// === Запуск конвейера / run the pipeline ===
using var cts = new CancellationTokenSource();
Task readerTask = ReaderAsync(pipe.Reader, cts.Token);
await WriterAsync(pipe.Writer, "Ping", cts.Token);
await WriterAsync(pipe.Writer, "Pong", cts.Token);
await pipe.Writer.CompleteAsync(cts.Token);   // сигнализируем конец потока / signal end of stream
await readerTask;

// === 3) Адаптеры: Stream <-> Pipe ===
// === 3) Adapters: Stream <-> Pipe ===
await using var fileStream = new MemoryStream(Encoding.UTF8.GetBytes("\x00\x00\x00\x05Hello"));
PipeReader fileReader = PipeReader.Create(fileStream);
// Теперь fileReader читает из Stream как из пайпа — без ручных копирований.
// Now fileReader reads from the Stream as if it were a pipe — without manual copies.
ReadResult first = await fileReader.ReadAsync();
Console.WriteLine($"Stream-as-pipe bytes: {first.Buffer.Length}");
await fileReader.CompleteAsync();
```

#### Best Practices

- RU: Предпочитайте конструктор `new MemoryStream(capacity)`, если размер заранее известен — это avoids лишних перераспределений.
- EN: Prefer `new MemoryStream(capacity)` when the size is known up front — it avoids unnecessary reallocations.
- RU: Используйте `ArrayPool<byte>.Shared` при работе с временными буферами вместо `new byte[]`.
- EN: Use `ArrayPool<byte>.Shared` for transient buffers instead of `new byte[]`.
- RU: В `Pipelines` всегда вызывайте `AdvanceTo` ровно один раз на каждую `ReadAsync`, иначе буфер не вернётся в пул.
- EN: In `Pipelines` always call `AdvanceTo` exactly once per `ReadAsync`, otherwise the buffer is never returned to the pool.
- RU: Передавайте `examined` в `AdvanceTo` так, чтобы он указывал на последний байт частичного кадра — это даёт корректное ожидание данных.
- EN: Set `examined` in `AdvanceTo` to the last byte of a partial frame — this yields correct data-await behavior.
- RU: Завершайте и писателя, и читателя через `Complete`/`CompleteAsync`, чтобы освободить арендованную память пула.
- EN: Complete both writer and reader via `Complete`/`CompleteAsync` to release pooled memory back.
- RU: Настройте `pauseWriterThreshold`/`resumeWriterThreshold` осознанно —太小 приводит к зависаниям, слишком большой — к избыточной памяти.
- EN: Tune `pauseWriterThreshold`/`resumeWriterThreshold` deliberately — too small stalls throughput, too large wastes memory.

#### Частые ошибки / Common Mistakes

- RU: Забыли `Advance` после `GetSpan` → буфер не увеличивается, `FlushAsync` не отправляет данные. → Всегда вызывайте `writer.Advance(bytesWritten)` перед `FlushAsync`.
- EN: Forgot `Advance` after `GetSpan` → the buffer does not grow, `FlushAsync` sends nothing. → Always call `writer.Advance(bytesWritten)` before `FlushAsync`.
- RU: Вызывают `ReadAsync` в цикле без `AdvanceTo` → `Buffer` растёт бесконечно, пул не освобождается. → Продвигайте буфер в каждом проходе.
- EN: Calling `ReadAsync` in a loop without `AdvanceTo` → the `Buffer` grows forever, the pool is never released. → Advance the buffer on every iteration.
- RU: Используют `ToArray()` на `ReadOnlySequence` целиком → массивная аллокация и копирование. → Разбирайте сегменты итератором или `CopyTo` в арендованный буфер.
- EN: Using `ToArray()` on the whole `ReadOnlySequence` → a large allocation and copy. → Iterate segments or `CopyTo` into a rented buffer.
- RU: Не обрабатывают `IsCompleted` → цикл чтения висит навсегда. → Проверяйте `result.IsCompleted` и вызывайте `reader.Complete()`.
- EN: Ignoring `IsCompleted` → the read loop hangs forever. → Check `result.IsCompleted` and call `reader.Complete()`.
- RU: Держат ссылку на `Memory<byte>` после `AdvanceTo` → use-after-return в пул. → Считайте память невалидной после `AdvanceTo`.
- EN: Holding a `Memory<byte>` reference after `AdvanceTo` → use-after-return to the pool. → Treat the memory as invalid after `AdvanceTo`.
- RU: `GetBuffer()` на `MemoryStream` с непубличным буфером бросает `UnauthorizedAccessException`. → Используйте `ToArray()` или конструктор с `publiclyVisible: true`.
- EN: `GetBuffer()` on a `MemoryStream` with a non-public buffer throws `UnauthorizedAccessException`. → Use `ToArray()` or the constructor with `publiclyVisible: true`.

#### Чек-лист самопроверки / Self-check Checklist

- [ ] Я могу объяснить, когда выбрать `MemoryStream`, а когда `Pipelines` (RU).
- [ ] I can explain when to choose `MemoryStream` versus `Pipelines` (EN).
- [ ] Я знаю разницу между `consumed` и `examined` в `AdvanceTo`.
- [ ] I know the difference between `consumed` and `examined` in `AdvanceTo`.
- [ ] Я понимаю, как `pauseWriterThreshold` создаёт backpressure.
- [ ] I understand how `pauseWriterThreshold` creates backpressure.
- [ ] Я могу разобрать кадр из `ReadOnlySequence<byte>` без `ToArray()` на всю последовательность.
- [ ] I can parse a frame from `ReadOnlySequence<byte>` without `ToArray()` on the whole sequence.
- [ ] Я всегда вызываю `Complete` на писателе и читателе для освобождения пула.
- [ ] I always call `Complete` on both writer and reader to release the pool.
- [ ] Я могу адаптировать `Stream` к `PipeReader` и наоборот.
- [ ] I can adapt a `Stream` to a `PipeReader` and vice versa.

#### Ресурсы / Resources
- [Microsoft Learn — System.IO.Pipelines — https://learn.microsoft.com/dotnet/standard/io/pipelines]

---

[⬆ К модулю M10](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
