[⬆ К модулю M10](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M10-L08: Binary-форматы (Protocol Buffers/MemoryPack — обзор) / Binary formats (Protocol Buffers/MemoryPack — overview)

**Модуль / Module:** M10
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

JSON — отличный формат по умолчанию: он читаемый, везде поддерживается и удобен для отладки. Но у читаемости есть цена. Текстовое представление тратит байты на кавычки, скобки, запятые и строковые имена полей. Числа кодируются десятичными цифрами. Парсинг текста — медленный, с ветвлениями и аллокациями. Когда данных много (логи, кэш, high-load API, межсервисный обмен), эти издержки становятся заметны.

Бинарные форматы решают три проблемы сразу: размер, скорость и детерминированность. Размер — потому что поля кодируются компактно, а имена заменяются числовыми тегами. Скорость — потому что парсер читает байты, а не разбирает грамматику. Детерминированность — потому что одинаковое значение всегда даёт одинаковые байты, что важно для хэширования, кэша и сигнатур.

**Protocol Buffers (protobuf)** от Google — де-факто стандарт межсервисного обмена. В экосистеме .NET его представляет библиотека `protobuf-net`. Ключевая идея — **схема**: вы описываете структуру данных (в `.proto`-файле или атрибутами прямо на C#-классах), и каждое поле получает числовой номер. По проводу летят не имена, а `(номер, значение)`. Это даёт прямой бонус — **прямую и обратную совместимость**: можно добавлять новые поля, не ломая старых клиентов. Неизвестные поля просто пропускаются. protobuf-net использует атрибуты `[ProtoContract]` и `[ProtoMember(N)]`, а данные сериализуются в `Stream` через `Serializer.Serialize`.

**MemoryPack** — современный .NET-нативный формат от автора `MessagePack-CSharp`. Он бьёт рекорды скорости, потому что использует `MemoryMarshal` и прямой доступ к памяти без рефлексии в горячем пути. Но его важная особенность — **формат зависит от порядка полей и бинарного layout**, а не от схемы с тегами. Это значит: MemoryPack идеален для хранения и передачи между одинаковыми версиями (кэш, IPC, сохранение состояния), но хуже подходит для долгоживущих API, где поля меняются со временем.

**MessagePack** занимает середину: бинарный, компактный, с поддержкой схем через `MessagePackObject` и тегами-ключами. Быстрее JSON в разы, медленнее MemoryPack, но устойчивее к изменениям.

**Когда выбирать binary?** Правило простое. Если данные читает человек или внешний клиент — JSON. Если данные идут между вашими сервисами в горячем пути, в кэш, в очередь или на диск в большом объёме — бинарный формат окупает себя. Для публичных API с долгой жизнью и версионированием — protobuf. Для внутреннего кэша и IPC — MemoryPack. MessagePack — универсальный компромисс.

Важный нюанс: бинарные форматы **не читаются глазами**. Это меняет подход к отладке — нужны инструменты и заранее продуманная стратегия версионирования. Зато вы получаете экономию трафика и CPU, которая на масштабе превращается в реальные деньги и секунды отклика.

#### Theory (EN)

JSON is the great default: readable, universally supported, easy to debug. But readability has a cost. Text representation spends bytes on quotes, braces, commas, and field names. Numbers are encoded as decimal digits. Parsing text is slow — full of branches and allocations. When data volume grows (logs, cache, high-load APIs, service-to-service traffic), these overheads become visible.

Binary formats solve three problems at once: size, speed, and determinism. Size, because fields are encoded compactly and names are replaced by numeric tags. Speed, because the parser reads bytes rather than parsing a grammar. Determinism, because the same value always produces the same bytes — important for hashing, caching, and signatures.

**Protocol Buffers (protobuf)** from Google is the de-facto standard for service-to-service exchange. In the .NET ecosystem it is represented by the `protobuf-net` library. The key idea is a **schema**: you describe the data structure (in a `.proto` file or with attributes directly on C# classes), and every field gets a numeric tag. What travels over the wire is not names but `(tag, value)` pairs. This gives a direct bonus — **forward and backward compatibility**: you can add new fields without breaking old clients. Unknown fields are simply skipped. protobuf-net uses `[ProtoContract]` and `[ProtoMember(N)]` attributes, and data is serialized into a `Stream` via `Serializer.Serialize`.

**MemoryPack** is a modern .NET-native format from the author of `MessagePack-CSharp`. It sets speed records because it uses `MemoryMarshal` and direct memory access with no reflection on the hot path. But its key trait is that the **format depends on field order and binary layout**, not on a tagged schema. That means MemoryPack is ideal for storing and transferring data between identical versions (cache, IPC, state persistence) but less suited for long-lived APIs where fields evolve over time.

**MessagePack** sits in the middle: binary, compact, with schema support through `MessagePackObject` and tag keys. Several times faster than JSON, slower than MemoryPack, but more tolerant to changes.

**When to go binary?** The rule is simple. If a human or an external client consumes the data — JSON. If data flows between your services on the hot path, into a cache, into a queue, or onto disk in large volume — a binary format pays off. For public APIs with a long lifetime and versioning — protobuf. For internal cache and IPC — MemoryPack. MessagePack is the universal compromise.

A crucial nuance: binary formats are **not readable by eye**. That changes the debugging approach — you need tools and a versioning strategy designed up front. In return you save traffic and CPU, which at scale turns into real money and response-time seconds.

#### Пример кода / Code Example

```csharp
// C# 12 / .NET 8 — обзор трёх бинарных форматов
// Install packages: protobuf-net, MemoryPack, MessagePack

using MemoryPack;           // MemoryPack — нативный .NET формат / native .NET format
using MessagePack;          // MessagePack — бинарный с тегами / binary with tags
using ProtoBuf;             // protobuf-net — Protocol Buffers / Protocol Buffers
using System.Text.Json;     // JSON — для сравнения / JSON for comparison

// === 1) Protocol Buffers (protobuf-net) ===
// Схема на атрибутах, полям присвоены числовые теги.
// Schema via attributes, fields get numeric tags.
[ProtoContract]
public record OrderPb
{
    [ProtoMember(1)] public int Id { get; init; }        // тег 1 — добавлять можно, удалять нельзя / tag 1
    [ProtoMember(2)] public string Customer { get; init; } = "";
    [ProtoMember(3)] public decimal Total { get; init; }
    [ProtoMember(4)] public DateTime Created { get; init; }
}

// === 2) MemoryPack ===
// Нет тегов — важен порядок полей. Быстро, но хрупко к изменению порядка.
// No tags — field order matters. Fast, but order-sensitive.
[MemoryPackable]
public partial record OrderMem
{
    public int Id { get; init; }
    public string Customer { get; init; } = "";
    public decimal Total { get; init; }
    public DateTime Created { get; init; }
}

// === 3) MessagePack ===
// Теги-ключи, компактно, устойчиво к добавлению полей.
// Tag keys, compact, tolerant to field additions.
[MessagePackObject]
public record OrderMsg
{
    [Key(0)] public int Id { get; init; }
    [Key(1)] public string Customer { get; init; } = "";
    [Key(2)] public decimal Total { get; init; }
    [Key(3)] public DateTime Created { get; init; }
}

public static class BinaryFormatsDemo
{
    public static void Run()
    {
        var order = new OrderPb(42, "Alice", 199.99m, DateTime.UtcNow);

        // Protocol Buffers → Stream / Protocol Buffers to Stream
        using var pbStream = new MemoryStream();
        Serializer.Serialize(pbStream, order);
        var pbBytes = pbStream.ToArray();
        pbStream.Position = 0;
        var pbBack = Serializer.Deserialize<OrderPb>(pbStream);

        // MemoryPack → byte[] / MemoryPack to byte[]
        var memBytes = MemoryPackSerializer.Serialize(new OrderMem(42, "Alice", 199.99m, DateTime.UtcNow));
        var memBack = MemoryPackSerializer.Deserialize<OrderMem>(memBytes);

        // MessagePack → byte[] / MessagePack to byte[]
        var msgBytes = MessagePackSerializer.Serialize(new OrderMsg(42, "Alice", 199.99m, DateTime.UtcNow));
        var msgBack = MessagePackSerializer.Deserialize<OrderMsg>(msgBytes);

        // JSON — эталон по размеру / JSON as size baseline
        var jsonBytes = JsonSerializer.SerializeToUtf8Bytes(new { Id = 42, Customer = "Alice", Total = 199.99m, Created = DateTime.UtcNow });

        Console.WriteLine($"protobuf   : {pbBytes.Length} bytes");
        Console.WriteLine($"MemoryPack : {memBytes.Length} bytes");
        Console.WriteLine($"MessagePack: {msgBytes.Length} bytes");
        Console.WriteLine($"JSON (UTF8): {jsonBytes.Length} bytes");
        Console.WriteLine($"Round-trip OK: {pbBack.Id == 42 && memBack.Id == 42 && msgBack.Id == 42}");
    }
}
```

#### Best Practices
- Используйте binary для горячего внутреннего трафика, кэша и очередей, а не для человекочитаемых конфигов / Use binary for hot internal traffic, cache, and queues — not for human-readable configs.
- Для публичных API с версионированием выбирайте protobuf: теги дают прямую/обратную совместимость / For versioned public APIs, prefer protobuf: tags give forward/backward compatibility.
- Не меняйте порядок полей в MemoryPack-типах после релиза — это ломает совместимость / Never change field order in MemoryPack types after release — it breaks compatibility.
- Резервируйте теги в protobuf: не переиспользуйте удалённые номера полей / Reserve tags in protobuf: never reuse deleted field numbers.
- Сравнивайте размер и скорость на ваших данных, а не на маркетинговых бенчмарках / Benchmark on your own data, not on marketing benchmarks.
- Храните схему в репозитории рядом с кодом — это контракт / Keep the schema in the repo next to code — it is the contract.

#### Частые ошибки / Common Mistakes
- Переиспользование номера удалённого поля в protobuf → всегда резервируйте через `[ProtoMember(...)]` и `reserved` в .proto / Reusing a deleted protobuf tag number → always reserve via `reserved` in .proto and never recycle numbers.
- Изменение порядка полей в MemoryPack-типе → зафиксируйте порядок с первого релиза, добавляйте только в конец / Changing field order in a MemoryPack type → freeze order from the first release, append only at the end.
- Сериализация `decimal` в protobuf-net без явной модели → используйте `decimal` через BclHelpers или строковое представление / Serializing `decimal` in protobuf-net without an explicit model → use `BclHelpers` or a string representation.
- Ожидание человекочитаемости бинарного вывода → добавляйте отдельный JSON-эндпоинт для отладки / Expecting binary output to be human-readable → add a separate JSON endpoint for debugging.
- Использование binary для конфигов, редактируемых руками → оставьте JSON/YAML/TOML / Using binary for hand-edited configs → keep JSON/YAML/TOML.

#### Чек-лист самопроверки / Self-check Checklist
- [ ] Я выбрал формат по сценарию: protobuf (API), MemoryPack (кэш/IPC), MessagePack (компромисс) / I chose the format by scenario: protobuf (API), MemoryPack (cache/IPC), MessagePack (compromise).
- [ ] Полям protobuf присвоены теги, удалённые номера зарезервированы / protobuf fields have tags, deleted numbers are reserved.
- [ ] Порядок полей в MemoryPack-типах зафиксирован и не меняется / Field order in MemoryPack types is fixed and unchanged.
- [ ] Я измерил размер и скорость на своих данных против JSON / I measured size and speed on my data against JSON.
- [ ] Есть путь отладки: JSON-эндпоинт или hex-дамп / A debug path exists: JSON endpoint or hex dump.
- [ ] Схема хранится в репозитории как контракт / The schema is stored in the repo as a contract.

#### Ресурсы / Resources
- [Microsoft Learn — Serialization — https://learn.microsoft.com/dotnet/standard/serialization/](https://learn.microsoft.com/dotnet/standard/serialization/)
- [protobuf-net (GitHub) — https://github.com/protobuf-net/protobuf-net](https://github.com/protobuf-net/protobuf-net)
- [MemoryPack (GitHub) — https://github.com/Cysharp/MemoryPack](https://github.com/Cysharp/MemoryPack)
- [MessagePack-CSharp (GitHub) — https://github.com/MessagePack-CSharp/MessagePack-CSharp](https://github.com/MessagePack-CSharp/MessagePack-CSharp)
- [Protocol Buffers — Language Guide — https://protobuf.dev/programming-guides/proto3/](https://protobuf.dev/programming-guides/proto3/)

---

[⬆ К модулю M10](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
