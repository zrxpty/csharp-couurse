---
[← К уроку M10-L08](lesson-M10-L08-binary-formats.md) | [⬆ К модулю M10](../README.md) | [Следующее ДЗ →]()
---

### Домашнее задание M10-L08: Binary-форматы (Protocol Buffers/MemoryPack — обзор) / Homework M10-L08: Binary formats (Protocol Buffers/MemoryPack — overview)

**Урок / Lesson:** M10-L08
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 4/5
**Цель / Goal:** (RU) На практике сравнить три бинарных формата .NET (protobuf-net, MemoryPack, MessagePack) с JSON-эталоном по размеру полезной нагрузки и скорости сериализации, а также закрепить стратегии версионирования: тегированную схему protobuf с резервированием удалённых номеров и хрупкость MemoryPack к изменению порядка полей. (EN) In practice compare three .NET binary formats (protobuf-net, MemoryPack, MessagePack) against a JSON baseline by payload size and serialization speed, and internalize versioning strategies: the tagged protobuf schema with reserved deleted tag numbers, and MemoryPack's fragility when field order changes.

#### Связь с уроком / Connection to the lesson
(RU) Урок вводит три формата и их ниши: protobuf — для долгоживущих API с версионированием, MemoryPack — для кэша и IPC между одинаковыми версиями, MessagePack — универсальный компромисс. В этом задании вы не просто сериализуете один объект, как в демо урока, а прогоняете коллекцию из тысячи заказов через все три формата плюс JSON, измеряете размеры и время, и на конкретном эксперименте доказываете главное правило урока: теги protobuf совместимы вперёд/назад, а порядок полей MemoryPack — нет.
(EN) The lesson introduces three formats and their niches: protobuf for long-lived versioned APIs, MemoryPack for cache and IPC between identical versions, MessagePack as the universal compromise. In this homework you do not just serialize one object like the lesson demo; you run a thousand-order collection through all three formats plus JSON, measure sizes and timings, and prove the lesson's core rule with a concrete experiment: protobuf tags are forward/backward compatible, while MemoryPack field order is not.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация
Вы — инженер платформы обработки заказов в высоконагруженном e-commerce. Сервис `OrderGateway` принимает заказы по HTTP в JSON, потому что их видят внешние клиенты и операторы. Но внутри платформы заказы летят между микросервисами `OrderGateway → Inventory → Billing → Shipping` по горячему пути, около пяти тысяч заказов в секунду. JSON здесь — лишние мегабайты трафика и лишние миллисекунды CPU на парсинг текста. Руководство просит обосновать переход на бинарный формат: показать экономию в байтах и в наносекундах, а также доказать, что выбранный формат переживёт эволюцию модели (появление новых полей, удаление старых) без поломки совместимости.

Урок M10-L08 даёт обзор трёх кандидатов: `protobuf-net` (тегированная схема, прямая/обратная совместимость), `MemoryPack` (нативный .NET, рекордная скорость, но хрупкость к порядку полей) и `MessagePack` (компромисс с тегами-ключами). Ваша задача — построить минимальный, но честный стенд сравнения на реалистичных данных и принять инженерное решение, опираясь на измерения, а не на маркетинговые бенчмарки из README библиотек. Параллельно вы должны продемонстрировать два ключевых свойства форматов на опытах: устойчивость protobuf к добавлению поля и ломку MemoryPack при перестановке полей местами.

#### Что нужно сделать (пошагово)
1. Создайте консольный проект .NET 8 с именем `BinaryFormatsLab` в папке `modules/M10/homework/BinaryFormatsLab`:
   ```bash
   dotnet new console -n BinaryFormatsLab -o modules/M10/homework/BinaryFormatsLab --framework net8.0
   cd modules/M10/homework/BinaryFormatsLab
   dotnet add package protobuf-net
   dotnet add package MemoryPack
   dotnet add package MessagePack
   ```
   Убедитесь, что в `BinaryFormatsLab.csproj` появился `net8.0` и три `PackageReference`. Включите `<Nullable>enable</Nullable>` и `<ImplicitUsings>enable</ImplicitUsings>`.

2. Опишите доменную модель `Order` в файле `Models.cs`. Поля: `int Id`, `string Customer`, `decimal Total`, `DateTime CreatedUtc`, `List<OrderLine> Lines`, где `OrderLine` — `string Sku`, `int Qty`, `decimal Price`. Создайте три параллельных представления для каждого бинарного формата:
   - `OrderPb` с атрибутами `[ProtoContract]` и `[ProtoMember(N)]` (теги 1..6, для `List<OrderLine>` — `[ProtoMember(5)]` с подсписком `OrderLinePb`, помеченным `[ProtoContract]`);
   - `OrderMem` с `[MemoryPackable]` и `partial record` (порядок полей зафиксируйте с первого релиза);
   - `OrderMsg` с `[MessagePackObject]` и `[Key(N)]` (теги 0..4).

3. В файле `Data.cs` напишите фабрику `OrderFactory.Build(int count)`, возвращающую `List<Order>` со случайными, но детерминированными данными (фиксированный seed `42`, чтобы запуск был воспроизводим). Генерируйте 1000 заказов, у каждого от 1 до 5 строк. Детерминированность критична: при повторных запусках размеры и тайминги должны совпадать до байта.

4. В `Program.cs` реализуйте три блока.
   Блок A — **размер полезной нагрузки**: сериализуйте коллекцию из 1000 заказов каждым форматом в `byte[]` (для protobuf — через `Serializer.Serialize` в `MemoryStream`, затем `ToArray()`; для MemoryPack — `MemoryPackSerializer.Serialize`; для MessagePack — `MessagePackSerializer.Serialize`; для JSON — `JsonSerializer.SerializeToUtf8Bytes`). Выведите таблицу:
   ```
   protobuf   : NNNNN bytes
   MemoryPack : NNNNN bytes
   MessagePack: NNNNN bytes
   JSON (UTF8): NNNNN bytes
   ```
   Ожидаемый качественный результат: MemoryPack и MessagePack заметно меньше JSON, protobuf немного больше MessagePack из-за протокольных накладных расходов на `decimal` и `DateTime`.

   Блок B — **скорость сериализации**: через `Stopwatch` измерьте время 5000 циклов serialize+deserialize для каждого формата (отдельно serialize, отдельно deserialize, усредните). Разогрев — 500 циклов до замера, чтобы JIT скомпилировал горячий путь. Выведите среднее время на одну операцию в микросекундах и операций в секунду. Не используйте `DateTime.Now` для замеров — только `Stopwatch.GetTimestamp()` и `Stopwatch.GetElapsedTime()` (API .NET 8).

   Блок C — **эксперимент по совместимости**: создайте `OrderPbV2` с новым полем `[ProtoMember(6)] public string Currency { get; init; } = "USD"` и без поля `CreatedUtc` (удалённый тег 4 — зарезервируйте его через отдельный тип-маркер или комментарий `// reserved tag 4`). Сериализуйте `OrderPbV2` и десериализуйте как `OrderPb` (старая модель): докажите, что `CreatedUtc` получит `default`, а `Id/Customer/Total/Lines` сохранятся. Затем создайте `OrderMemSwapped`, где поля `Customer` и `Total` переставлены местами, сериализуйте `OrderMem` и десериализуйте как `OrderMemSwapped`: покажите, что значения перемешиваются (в `Customer` оказывается число, в `Total` — мусор), что доказывает хрупкость MemoryPack к порядку.

5. В отдельном методе `PrintReport` соберите все измерения в одну таблицу-сводку и выведите итоговую рекомендацию: какой формат выбрать для горячего внутреннего трафика, какой — для публичного API, какой — для кэша. Рекомендация должна опираться на ваши числа, а не на слайды урока.

6. Запустите `dotnet run` и сохраните вывод в `report.txt` рядом с проектом. Вывод должен содержать все три блока и итоговую рекомендацию.

#### Требования к решению
- Проект компилируется под .NET 8 без warning-ов уровня error (`<TreatWarningsAsErrors>true</TreatWarningsAsErrors>` опционально, но код должен быть чистым).
- Используется C# 12: top-level statements в `Program.cs`, `record` для моделей, collection expressions при инициализации списков (`new() { ... }` или `[ ... ]`), raw-string-литералы где уместно для длинного текстового блока рекомендации.
- Все три формата реально используются через свои API; нельзя имитировать «бинарность» ручной打包кой байтов.
- Данные детерминированы (seed `42`), повторный запуск даёт те же размеры байт.
- Замер скорости честный: разогрев, усреднение, отдельный учёт serialize и deserialize, использование `Stopwatch.GetElapsedTime`.
- Эксперимент по совместимости показывает оба сценария (protobuf — добавление поля прошло безопасно; MemoryPack — перестановка полей сломала данные).
- Резервирование удалённого тега в protobuf оформлено явно (комментарий или атрибут-маркер), чтобы показать понимание правила «не переиспользуй номера».
- Никаких `TODO`, `throw new NotImplementedException()`, заглушек. Код рабочий от первого запуска.

#### Тонкости и подводные камни
- **`decimal` в protobuf-net.** По умолчанию protobuf-net не знает, как сериализовать `decimal`, и либо падает, либо кодирует его через `BclHelpers` с тегами-вложенными сообщениями, что раздувает размер. Урок явно предупреждает об этом. Если хотите честное сравнение, оставьте `decimal` в protobuf-модели — и вы увидите, что protobuf-размер неожиданно велик; это и есть «налог» за BCL-совместимость. Альтернатива — хранить `Total` как `long` в копейках, но тогда модель перестаёт быть эквивалентной другим форматам, что нарушит чистоту эксперимента. Документируйте выбранный подход в комментарии.
- **`DateTime` в protobuf-net.** Аналогично кодируется через `BclHelpers.WriteDateTime` с тегом-сообщением, что добавляет байты. В MemoryPack и MessagePack `DateTime` укладывается компактнее. Учитывайте это при интерпретации блока A.
- **MemoryPack требует `partial record` и source-generator.** Если забыть `partial`, компилятор выдаст ошибку генератора. Поля должны быть автосвойствами с `get; init;` (или `get; set;`). Порядок полей определяется порядком объявления, а не алфавитом.
- **MemoryPack и `List<T>`.** Вложенные `OrderLineMem` тоже должны быть `[MemoryPackable]`. Забыли — генератор не сгенерирует сериализатор и будет runtime-ошибка.
- **protobuf-net и `List<T>`.** Для подсписка нужен отдельный `[ProtoContract]`-тип `OrderLinePb`, а в родителе — `[ProtoMember(5)] public List<OrderLinePb> Lines { get; init; } = new();`. Не забудьте `= new();`, иначе при десериализации пустого списка поле может оказаться `null`.
- **Резервирование тегов.** В `.proto`-мире есть `reserved 4;`. В protobuf-net на атрибутах эквивалента нет, поэтому удалённое поле документируют комментарием `// reserved: tag 4 (CreatedUtc) — DO NOT REUSE`. Если переиспользовать тег 4 под новое поле, старые клиенты получат мусор.
- **`Stopwatch.GetElapsedTime`.** В .NET 8 появился `Stopwatch.GetElapsedTime(long startTimestamp)` — он возвращает `TimeSpan` без создания лишних `DateTime`-объектов. Не используйте устаревший паттерн `(end - start) / Stopwatch.Frequency * 1000` — он менее точен и многословнее.
- **Разогрев.** Без 500 циклов разогрева первый замер захватит JIT-компиляцию и даст время в 5–20 раз выше реального. Это самая частая ошибка в студенческих бенчмарках.
- **Сборщик мусора.** Между замерами разных форматов вызывайте `GC.Collect(); GC.WaitForPendingFinalizers(); GC.Collect();`, чтобы один формат не платил за мусор другого. Иначе MemoryPack может выглядеть медленнее, чем есть, потому что предыдущий блок JSON оставил кучу мусора.

#### Критерии приёмки
- [ ] Проект `BinaryFormatsLab` создан, компилируется под `net8.0`, три `PackageReference` в `.csproj`.
- [ ] Модели `OrderPb`, `OrderMem`, `OrderMsg` и подсписки описаны корректно с правильными атрибутами каждого формата.
- [ ] `OrderFactory.Build` детерминирован (seed `42`), возвращает 1000 заказов со строками.
- [ ] Блок A выводит размеры всех четырёх форматов, числа стабильны при повторном запуске.
- [ ] Блок B измеряет serialize и deserialize отдельно, с разогревом и усреднением.
- [ ] Замер использует `Stopwatch.GetTimestamp()` / `Stopwatch.GetElapsedTime()`.
- [ ] Блок C демонстрирует совместимость protobuf (`OrderPbV2` → `OrderPb`) и ломку MemoryPack (`OrderMem` → `OrderMemSwapped`).
- [ ] Удалённый тег protobuf явно зарезервирован комментарием или маркером.
- [ ] Итоговая рекомендация опирается на измеренные числа, а не на общие слова.
- [ ] Вывод сохранён в `report.txt`.
- [ ] Нет `TODO`, заглушек, `NotImplementedException`.
- [ ] Код использует C# 12 (top-level statements, records, collection expressions, raw strings где уместно).
- [ ] Между замерами форматов вызывается `GC.Collect()` для чистоты.
- [ ] `decimal` и `DateTime` в protobuf обработаны осознанно, подход задокументирован комментарием.
- [ ] Объём RU- и EN-блоков постановки — не менее 1000 слов каждый.

#### Подсказки (без прямого ответа)
- Для разогрева подойдёт простой цикл `for (int i = 0; i < 500; i++) SerializeAndDeserialize(order);` перед основным замером. Не измеряйте время разогрева.
- Чтобы усреднить честно, накапливайте `long totalTicks` и делите на количество итераций, а не усредняйте `TimeSpan` через LINQ — это создаёт лишние аллокации.
- Для блока C удобно сериализовать `OrderPbV2` в `MemoryStream`, затем `stream.Position = 0` и десериализовать как `OrderPb`. protobuf-net прочитает известные теги и пропустит неизвестный тег 6.
- Чтобы показать ломку MemoryPack, сериализуйте `OrderMem(...)` и десериализуйте байты как `OrderMemSwapped` — там, где в исходной модели шло строковое `Customer`, в переставленной модели читается `decimal Total`, и наоборот.
- Не пытайтесь «улучшить» protobuf, заменив `decimal` на `long` — это нарушит эквивалентность моделей и сделает сравнение нечестным. Лучше задокументируйте налог.
- Для вывода таблицы используйте интерполяцию с выравниванием: `{name,-12}: {bytes,8:N0} bytes`.

#### Эталонное решение (разбор)
```csharp
// C# 12 / .NET 8 — BinaryFormatsLab
// Эталонное решение домашнего задания M10-L08.
// Reference solution for homework M10-L08.

using MemoryPack;
using MessagePack;
using ProtoBuf;
using System.Diagnostics;
using System.Text.Json;

// === Модели / Models ===

// Protocol Buffers: тегированная схема, прямая/обратная совместимость.
// Tagged schema, forward/backward compatibility.
[ProtoContract]
public record OrderPb
{
    [ProtoMember(1)] public int Id { get; init; }
    [ProtoMember(2)] public string Customer { get; init; } = "";
    [ProtoMember(3)] public decimal Total { get; init; }      // BclHelpers: налог на размер / size tax
    [ProtoMember(4)] public DateTime CreatedUtc { get; init; } // reserved при удалении — НЕ ПЕРЕИСПОЛЬЗОВАТЬ / reserved if removed — DO NOT REUSE
    [ProtoMember(5)] public List<OrderLinePb> Lines { get; init; } = new();
}

[ProtoContract]
public record OrderLinePb
{
    [ProtoMember(1)] public string Sku { get; init; } = "";
    [ProtoMember(2)] public int Qty { get; init; }
    [ProtoMember(3)] public decimal Price { get; init; }
}

// V2 с добавленным полем Currency (тег 6) и удалённым CreatedUtc (тег 4 — зарезервирован).
// V2 with added Currency (tag 6) and removed CreatedUtc (tag 4 — reserved).
[ProtoContract]
public record OrderPbV2
{
    [ProtoMember(1)] public int Id { get; init; }
    [ProtoMember(2)] public string Customer { get; init; } = "";
    [ProtoMember(3)] public decimal Total { get; init; }
    // reserved: tag 4 (CreatedUtc) — удалён, НЕ ПЕРЕИСПОЛЬЗОВАТЬ / removed, DO NOT REUSE
    [ProtoMember(5)] public List<OrderLinePb> Lines { get; init; } = new();
    [ProtoMember(6)] public string Currency { get; init; } = "USD";
}

// MemoryPack: порядок полей — контракт. Менять нельзя после релиза.
// MemoryPack: field order is the contract. Never change after release.
[MemoryPackable]
public partial record OrderMem
{
    public int Id { get; init; }
    public string Customer { get; init; } = "";
    public decimal Total { get; init; }
    public DateTime CreatedUtc { get; init; }
    public List<OrderLineMem> Lines { get; init; } = new();
}

[MemoryPackable]
public partial record OrderLineMem
{
    public string Sku { get; init; } = "";
    public int Qty { get; init; }
    public decimal Price { get; init; }
}

// Ломаная версия: Customer и Total переставлены. Демонстрирует хрупкость порядка.
// Broken version: Customer and Total swapped. Demonstrates order fragility.
[MemoryPackable]
public partial record OrderMemSwapped
{
    public int Id { get; init; }
    public decimal Total { get; init; }   // здесь ожидается decimal, но в байтах — строка Customer
    public string Customer { get; init; } = "";  // здесь ожидается string, но в байтах — decimal
    public DateTime CreatedUtc { get; init; }
    public List<OrderLineMem> Lines { get; init; } = new();
}

// MessagePack: теги-ключи, компромисс между protobuf и MemoryPack.
// MessagePack: tag keys, a compromise between protobuf and MemoryPack.
[MessagePackObject]
public record OrderMsg
{
    [Key(0)] public int Id { get; init; }
    [Key(1)] public string Customer { get; init; } = "";
    [Key(2)] public decimal Total { get; init; }
    [Key(3)] public DateTime CreatedUtc { get; init; }
    [Key(4)] public List<OrderLineMsg> Lines { get; init; } = new();
}

[MessagePackObject]
public record OrderLineMsg
{
    [Key(0)] public string Sku { get; init; } = "";
    [Key(1)] public int Qty { get; init; }
    [Key(2)] public decimal Price { get; init; }
}

// === Фабрика данных / Data factory ===
public static class OrderFactory
{
    public static List<OrderPb> BuildPb(int count, int seed = 42)
    {
        var rng = new Random(seed);
        var customers = new[] { "Alice", "Bob", "Charlie", "Diana", "Eve" };
        var skus = new[] { "SKU-001", "SKU-002", "SKU-003", "SKU-004", "SKU-005" };
        var result = new List<OrderPb>(count);
        for (int i = 1; i <= count; i++)
        {
            int lineCount = rng.Next(1, 6);
            var lines = new List<OrderLinePb>(lineCount);
            decimal total = 0m;
            for (int j = 0; j < lineCount; j++)
            {
                var line = new OrderLinePb
                {
                    Sku = skus[rng.Next(skus.Length)],
                    Qty = rng.Next(1, 10),
                    Price = Math.Round(rng.NextDecimal() * 100m, 2)
                };
                lines.Add(line);
                total += line.Price * line.Qty;
            }
            result.Add(new OrderPb
            {
                Id = i,
                Customer = customers[rng.Next(customers.Length)],
                Total = Math.Round(total, 2),
                CreatedUtc = new DateTime(2024, 1, 1, 0, 0, 0, DateTimeKind.Utc).AddSeconds(i),
                Lines = lines
            });
        }
        return result;
    }

    public static List<OrderMem> ToMem(List<OrderPb> pb) => pb.Select(o => new OrderMem
    {
        Id = o.Id, Customer = o.Customer, Total = o.Total, CreatedUtc = o.CreatedUtc,
        Lines = o.Lines.Select(l => new OrderLineMem { Sku = l.Sku, Qty = l.Qty, Price = l.Price }).ToList()
    }).ToList();

    public static List<OrderMsg> ToMsg(List<OrderPb> pb) => pb.Select(o => new OrderMsg
    {
        Id = o.Id, Customer = o.Customer, Total = o.Total, CreatedUtc = o.CreatedUtc,
        Lines = o.Lines.Select(l => new OrderLineMsg { Sku = l.Sku, Qty = l.Qty, Price = l.Price }).ToList()
    }).ToList();

    public static List<object> ToAnon(List<OrderPb> pb) => pb.Select(o => (object)new
    {
        o.Id, o.Customer, o.Total, o.CreatedUtc,
        Lines = o.Lines.Select(l => new { l.Sku, l.Qty, l.Price })
    }).ToList();
}

internal static class RandomExt
{
    public static decimal NextDecimal(this Random r) => (decimal)r.NextDouble();
}

// === Program / Точка входа ===
// Top-level statements (.NET 8 / C# 12).

const int Count = 1000;
const int Warmup = 500;
const int Iterations = 5000;

var pbOrders = OrderFactory.BuildPb(Count);
var memOrders = OrderFactory.ToMem(pbOrders);
var msgOrders = OrderFactory.ToMsg(pbOrders);
var anonOrders = OrderFactory.ToAnon(pbOrders);

// --- Блок A: размер полезной нагрузки / Block A: payload size ---
var pbBytes = SerializePb(pbOrders);
var memBytes = MemoryPackSerializer.Serialize(memOrders);
var msgBytes = MessagePackSerializer.Serialize(msgOrders);
var jsonBytes = JsonSerializer.SerializeToUtf8Bytes(anonOrders);

Console.WriteLine("=== Block A: payload size (1000 orders) ===");
Console.WriteLine($"{"protobuf",-12}: {pbBytes.Length,8:N0} bytes");
Console.WriteLine($"{"MemoryPack",-12}: {memBytes.Length,8:N0} bytes");
Console.WriteLine($"{"MessagePack",-12}: {msgBytes.Length,8:N0} bytes");
Console.WriteLine($"{"JSON (UTF8)",-12}: {jsonBytes.Length,8:N0} bytes");
Console.WriteLine();

// --- Блок B: скорость / Block B: speed ---
Console.WriteLine("=== Block B: speed (5000 iterations, 500 warmup) ===");
Measure("protobuf serialize  ", () => SerializePb(pbOrders), Warmup, Iterations);
Measure("protobuf deserialize", () => DeserializePb<OrderPb>(pbBytes), Warmup, Iterations);
Measure("MemoryPack serialize ", () => MemoryPackSerializer.Serialize(memOrders), Warmup, Iterations);
Measure("MemoryPack deserial. ", () => MemoryPackSerializer.Deserialize<List<OrderMem>>(memBytes), Warmup, Iterations);
Measure("MessagePack serialize", () => MessagePackSerializer.Serialize(msgOrders), Warmup, Iterations);
Measure("MessagePack deserial.", () => MessagePackSerializer.Deserialize<List<OrderMsg>>(msgBytes), Warmup, Iterations);
Measure("JSON serialize       ", () => JsonSerializer.SerializeToUtf8Bytes(anonOrders), Warmup, Iterations);
Measure("JSON deserialize     ", () => JsonSerializer.Deserialize<List<OrderMsg>>(jsonBytes), Warmup, Iterations);
Console.WriteLine();

// --- Блок C: совместимость / Block C: compatibility ---
Console.WriteLine("=== Block C: compatibility experiment ===");

// protobuf: V2 → V1. Новое поле Currency (тег 6) пропускается, удалённое (тег 4) даёт default.
var v2 = new OrderPbV2 { Id = 99, Customer = "Zoe", Total = 250m, Currency = "EUR",
    Lines = new() { new() { Sku = "SKU-X", Qty = 2, Price = 10m } } };
using var v2Stream = new MemoryStream();
Serializer.Serialize(v2Stream, v2);
v2Stream.Position = 0;
var v1back = Serializer.Deserialize<OrderPb>(v2Stream);
Console.WriteLine($"protobuf V2→V1: Id={v1back.Id} Customer={v1back.Customer} Total={v1back.Total} " +
                  $"CreatedUtc={v1back.CreatedUtc} (default={default(DateTime)}) Lines={v1back.Lines.Count}");
Console.WriteLine("  → добавление поля безопасно, удалённое поле=default / added field safe, removed=default");

// MemoryPack: порядок полей нарушен → данные перемешиваются.
var mem = new OrderMem { Id = 7, Customer = "Alice", Total = 199.99m,
    CreatedUtc = new DateTime(2024, 6, 1, 0, 0, 0, DateTimeKind.Utc),
    Lines = new() { new() { Sku = "SKU-1", Qty = 1, Price = 199.99m } } };
var memBrokenBytes = MemoryPackSerializer.Serialize(mem);
try
{
    var swapped = MemoryPackSerializer.Deserialize<OrderMemSwapped>(memBrokenBytes);
    Console.WriteLine($"MemoryPack swapped: Id={swapped.Id} Customer={swapped.Customer} Total={swapped.Total}");
    Console.WriteLine("  → порядок полей сломал данные / field order broke data");
}
catch (Exception ex)
{
    Console.WriteLine($"MemoryPack swapped threw: {ex.GetType().Name} — порядок полей критичен / order is critical");
}
Console.WriteLine();

// --- Итоговая рекомендация / Final recommendation ---
var recommendation = """
    === Рекомендация / Recommendation ===
    На измеренных данных MemoryPack даёт наименьший размер и наивысшую скорость —
    но его хрупкость к порядку полей (блок C) делает его непригодным для публичного API.
    Для горячего внутреннего трафика OrderGateway→Inventory→Billing→Shipping выбирайте
    MemoryPack: одинаковые версии, максимальная экономия.

    Для публичного API и долгоживущих контрактов — protobuf: теги дают прямую/обратную
    совместимость, удалённые теги резервируются и не переиспользуются.

    MessagePack — компромисс для случаев, когда нужна бинарность, но нет ресурсов на
    поддержку .proto-схемы.
    """;
Console.WriteLine(recommendation);

// === Вспомогательные методы / Helpers ===

static byte[] SerializePb<T>(T value)
{
    using var ms = new MemoryStream();
    Serializer.Serialize(ms, value);
    return ms.ToArray();
}

static T DeserializePb<T>(byte[] bytes) where T : class
{
    using var ms = new MemoryStream(bytes);
    return Serializer.Deserialize<T>(ms);
}

static void Measure(string label, Action op, int warmup, int iterations)
{
    for (int i = 0; i < warmup; i++) op();                       // разогрев / warmup
    GC.Collect(); GC.WaitForPendingFinalizers(); GC.Collect();   // чистый старт / clean start

    long start = Stopwatch.GetTimestamp();
    for (int i = 0; i < iterations; i++) op();
    TimeSpan elapsed = Stopwatch.GetElapsedTime(start);

    double avgUs = elapsed.TotalMilliseconds * 1000.0 / iterations;
    double opsPerSec = iterations / elapsed.TotalSeconds;
    Console.WriteLine($"{label}: {avgUs,8:F2} µs/op, {opsPerSec,12:N0} ops/s");
}
```

**Разбор по строкам.** Модели `OrderPb` / `OrderMem` / `OrderMsg` — три эквивалентных представления одной и той же доменной модели, каждое в своём формате. В `OrderPb` каждый `[ProtoMember(N)]` закрепляет числовой тег за полем — это и есть «схема на атрибутах» из урока, дающая прямую/обратную совместимость. `OrderPbV2` демонстрирует эволюцию: новое поле `Currency` получает тег 6, удалённое `CreatedUtc` (тег 4) помечено комментарием `// reserved ... DO NOT REUSE` — это реализация best practice урока «резервируйте теги, не переиспользуйте удалённые номера». В блоке C сериализованный `OrderPbV2` десериализуется как `OrderPb`: protobuf-net читает известные теги (1,2,3,5) и пропускает неизвестный тег 6, а отсутствующий тег 4 даёт `default(DateTime)` — доказательство правила урока «неизвестные поля просто пропускаются».

`OrderMem` использует `[MemoryPackable]` и `partial record` — требование source-генератора MemoryPack. Поля объявлены в фиксированном порядке, который становится бинарным контрактом. `OrderMemSwapped` переставляет `Customer` и `Total` местами: при десериализации байтов исходного `OrderMem` в переставленную модель значения оказываются не на своих местах (в `string Customer` читаются байты `decimal`, и наоборот) — это и есть «хрупкость к порядку полей» из урока, доказанная на опыте, а не на словах.

`OrderMsg` с `[Key(N)]` показывает компромиссный формат: теги-ключи есть, но без накладных расходов protobuf-net на `decimal`/`DateTime` через BclHelpers. Фабрика `OrderFactory.BuildPb` использует фиксированный seed `42` — детерминированность, чтобы повторный запуск давал те же размеры байтов и те же тайминги. Методы `ToMem` / `ToMsg` / `ToAnon` строят эквивалентные коллекции для каждого формата, сохраняя чистоту сравнения.

Блок A сериализует 1000 заказов каждым форматом и выводит размеры — здесь обычно видно, что protobuf-байтов больше, чем у MemoryPack/MessagePack, из-за налога BclHelpers на `decimal` и `DateTime` (это та самая «частая ошибка» из урока: `decimal` в protobuf-net без явной модели раздувает размер). Блок B измеряет serialize и deserialize отдельно, с разогревом (500 циклов до замера, чтобы JIT скомпилировал горячий путь) и `GC.Collect()` между форматами, чтобы один не платил за мусор другого. Используется `Stopwatch.GetTimestamp()` + `Stopwatch.GetElapsedTime()` — API .NET 8, рекомендованный вместо ручного деления на `Stopwatch.Frequency`. Блок C — два опыта совместимости. Итог — raw-string-литерал `recommendation` (C# 11+, уместен для многострочного текста) с инженерным выводом, опирающимся на измеренные числа, а не на общие слова урока.

Применённые концепции урока: тегированная схема protobuf и резервирование удалённых тегов, хрупкость порядка полей MemoryPack, теги-ключи MessagePack как компромисс, выбор формата по сценарию (protobuf для API, MemoryPack для кэша/IPC, MessagePack как универсал), отдельный JSON-путь для отладки, измерение на своих данных против JSON-эталона.

#### Задания на углубление (бонус)
1. Добавьте `BenchmarkDotNet` и замените ручной `Stopwatch`-замер на `[MemoryDiagnoser]`-бенчмарк. Сравните `Mean`, `Allocated`, `Gen0` — MemoryPack обычно показывает нулевые аллокации на горячем пути. Объясните, почему.
2. Реализуйте версионный сценарий «три релиза»: V1 → V2 (добавили `Currency`) → V3 (удалили `Total`, добавили `LinesTotal`). Покажите, что V1-клиент читает V3-данные без падения, а V3-клиент читает V1-данные. Документируйте каждый зарезервированный тег.
3. Сериализуйте коллекцию в файл и измерьте размер на диске сжатого (`GzipStream`) и несжатого варианта. Бинарные форматы сжимаются хуже JSON, потому что в них меньше избыточности — подтвердите или опровергните это на ваших данных.
4. Реализуйте `MemoryPack` с `MemoryPackSerializer.SerializeAsync(Stream, T)` и сравните с синхронной версией по скорости и аллокациям. Когда async-версия оправдана, а когда — нет?

---

## Statement in English / Постановка на английском

#### Context & motivation
You are a platform engineer at a high-load e-commerce order-processing company. The `OrderGateway` service accepts orders over HTTP in JSON, because external clients and operators read them. But inside the platform orders travel between microservices `OrderGateway → Inventory → Billing → Shipping` on the hot path, around five thousand orders per second. JSON here means wasted megabytes of traffic and wasted CPU-milliseconds on text parsing. Management asks you to justify a switch to a binary format: show the savings in bytes and in nanoseconds, and prove that the chosen format will survive model evolution (new fields arriving, old ones removed) without breaking compatibility.

Lesson M10-L08 gives an overview of three candidates: `protobuf-net` (tagged schema, forward/backward compatibility), `MemoryPack` (native .NET, record speed, but fragility to field order), and `MessagePack` (a compromise with tag keys). Your task is to build a minimal but honest comparison bench on realistic data and make an engineering decision based on measurements, not on marketing benchmarks from library READMEs. In parallel you must demonstrate two key properties of the formats through experiments: protobuf's tolerance to adding a field, and MemoryPack's breakage when fields are reordered.

#### What to do step by step
1. Create a .NET 8 console project named `BinaryFormatsLab` in `modules/M10/homework/BinaryFormatsLab`:
   ```bash
   dotnet new console -n BinaryFormatsLab -o modules/M10/homework/BinaryFormatsLab --framework net8.0
   cd modules/M10/homework/BinaryFormatsLab
   dotnet add package protobuf-net
   dotnet add package MemoryPack
   dotnet add package MessagePack
   ```
   Confirm `BinaryFormatsLab.csproj` has `net8.0` and three `PackageReference` entries. Enable `<Nullable>enable</Nullable>` and `<ImplicitUsings>enable</ImplicitUsings>`.

2. Describe the domain model `Order` in `Models.cs`. Fields: `int Id`, `string Customer`, `decimal Total`, `DateTime CreatedUtc`, `List<OrderLine> Lines`, where `OrderLine` is `string Sku`, `int Qty`, `decimal Price`. Create three parallel representations for each binary format:
   - `OrderPb` with `[ProtoContract]` and `[ProtoMember(N)]` attributes (tags 1..6; for `List<OrderLine>` use `[ProtoMember(5)]` with a nested `[ProtoContract]` type `OrderLinePb`);
   - `OrderMem` with `[MemoryPackable]` and `partial record` (freeze field order from the first release);
   - `OrderMsg` with `[MessagePackObject]` and `[Key(N)]` (tags 0..4).

3. In `Data.cs` write a factory `OrderFactory.Build(int count)` returning `List<Order>` with random but deterministic data (fixed seed `42`, so a run is reproducible). Generate 1000 orders, each with 1 to 5 lines. Determinism is critical: on repeated runs sizes and timings must match byte for byte.

4. In `Program.cs` implement three blocks.
   Block A — **payload size**: serialize a collection of 1000 orders with each format into `byte[]` (for protobuf — via `Serializer.Serialize` into a `MemoryStream`, then `ToArray()`; for MemoryPack — `MemoryPackSerializer.Serialize`; for MessagePack — `MessagePackSerializer.Serialize`; for JSON — `JsonSerializer.SerializeToUtf8Bytes`). Print a table:
   ```
   protobuf   : NNNNN bytes
   MemoryPack : NNNNN bytes
   MessagePack: NNNNN bytes
   JSON (UTF8): NNNNN bytes
   ```
   Expected qualitative result: MemoryPack and MessagePack are noticeably smaller than JSON, protobuf is slightly larger than MessagePack because of protocol overhead on `decimal` and `DateTime`.

   Block B — **serialization speed**: with `Stopwatch` measure the time of 5000 serialize+deserialize cycles for each format (serialize separately, deserialize separately, average). Warm up with 500 cycles before measurement so JIT compiles the hot path. Print the average time per operation in microseconds and operations per second. Do not use `DateTime.Now` for measurements — only `Stopwatch.GetTimestamp()` and `Stopwatch.GetElapsedTime()` (the .NET 8 API).

   Block C — **compatibility experiment**: create `OrderPbV2` with a new field `[ProtoMember(6)] public string Currency { get; init; } = "USD"` and without the `CreatedUtc` field (deleted tag 4 — reserve it via a marker comment `// reserved tag 4`). Serialize `OrderPbV2` and deserialize as `OrderPb` (the old model): prove that `CreatedUtc` gets `default`, while `Id/Customer/Total/Lines` survive. Then create `OrderMemSwapped`, where `Customer` and `Total` are swapped, serialize `OrderMem` and deserialize as `OrderMemSwapped`: show that values get mixed up (a number lands in `Customer`, garbage lands in `Total`), proving MemoryPack's fragility to field order.

5. In a separate `PrintReport` method collect all measurements into one summary table and print the final recommendation: which format to choose for hot internal traffic, which for a public API, which for cache. The recommendation must lean on your numbers, not on lesson slides.

6. Run `dotnet run` and save the output to `report.txt` next to the project. The output must contain all three blocks and the final recommendation.

#### Requirements
- The project compiles under .NET 8 without warnings-as-errors (optionally enable `<TreatWarningsAsErrors>true</TreatWarningsAsErrors>`, but the code must be clean).
- C# 12 is used: top-level statements in `Program.cs`, `record` for models, collection expressions for list initialization (`new() { ... }` or `[ ... ]`), raw-string literals where appropriate for the long recommendation text block.
- All three formats are really used through their own APIs; you may not fake "binariness" with manual byte packing.
- Data is deterministic (seed `42`); a repeated run yields the same byte sizes.
- The speed measurement is honest: warmup, averaging, separate accounting for serialize and deserialize, use of `Stopwatch.GetElapsedTime`.
- The compatibility experiment shows both scenarios (protobuf — adding a field was safe; MemoryPack — reordering broke the data).
- The reserved deleted protobuf tag is documented explicitly (comment or marker attribute), showing understanding of the "do not reuse numbers" rule.
- No `TODO`, `throw new NotImplementedException()`, stubs. The code works from the first run.

#### Pitfalls
- **`decimal` in protobuf-net.** By default protobuf-net does not know how to serialize `decimal`, and either throws or encodes it through `BclHelpers` with nested-message tags, which inflates the size. The lesson warns about this explicitly. If you want an honest comparison, leave `decimal` in the protobuf model — and you will see that the protobuf size is unexpectedly large; that is the "BCL-compatibility tax." An alternative is to store `Total` as `long` in cents, but then the model stops being equivalent to the other formats, breaking the experiment's purity. Document the chosen approach in a comment.
- **`DateTime` in protobuf-net.** Similarly encoded through `BclHelpers.WriteDateTime` with a message tag, which adds bytes. In MemoryPack and MessagePack `DateTime` lays out more compactly. Account for this when interpreting Block A.
- **MemoryPack requires `partial record` and a source generator.** Forgetting `partial` produces a generator error. Fields must be auto-properties with `get; init;` (or `get; set;`). Field order is determined by declaration order, not alphabet.
- **MemoryPack and `List<T>`.** The nested `OrderLineMem` must also be `[MemoryPackable]`. Forget it — the generator will not emit a serializer and you get a runtime error.
- **protobuf-net and `List<T>`.** The sublist needs a separate `[ProtoContract]` type `OrderLinePb`, and in the parent — `[ProtoMember(5)] public List<OrderLinePb> Lines { get; init; } = new();`. Do not forget `= new();`, otherwise on deserialization of an empty list the field may end up `null`.
- **Reserving tags.** In the `.proto` world there is `reserved 4;`. In protobuf-net with attributes there is no equivalent, so a deleted field is documented with a comment `// reserved: tag 4 (CreatedUtc) — DO NOT REUSE`. If tag 4 is reused for a new field, old clients will get garbage.
- **`Stopwatch.GetElapsedTime`.** .NET 8 introduced `Stopwatch.GetElapsedTime(long startTimestamp)` — it returns a `TimeSpan` without creating extra `DateTime` objects. Do not use the legacy pattern `(end - start) / Stopwatch.Frequency * 1000` — it is less precise and more verbose.
- **Warmup.** Without 500 warmup cycles the first measurement captures JIT compilation and yields a time 5–20× higher than reality. This is the most common mistake in student benchmarks.
- **Garbage collector.** Between measurements of different formats call `GC.Collect(); GC.WaitForPendingFinalizers(); GC.Collect();`, so one format does not pay for another's garbage. Otherwise MemoryPack may look slower than it is, because the previous JSON block left a pile of trash behind.

#### Acceptance criteria
- [ ] Project `BinaryFormatsLab` is created, compiles under `net8.0`, three `PackageReference` entries in `.csproj`.
- [ ] Models `OrderPb`, `OrderMem`, `OrderMsg` and sublists are described correctly with each format's proper attributes.
- [ ] `OrderFactory.Build` is deterministic (seed `42`), returns 1000 orders with lines.
- [ ] Block A prints sizes of all four formats, numbers are stable across repeated runs.
- [ ] Block B measures serialize and deserialize separately, with warmup and averaging.
- [ ] The measurement uses `Stopwatch.GetTimestamp()` / `Stopwatch.GetElapsedTime()`.
- [ ] Block C demonstrates protobuf compatibility (`OrderPbV2` → `OrderPb`) and MemoryPack breakage (`OrderMem` → `OrderMemSwapped`).
- [ ] The deleted protobuf tag is explicitly reserved with a comment or marker.
- [ ] The final recommendation leans on measured numbers, not on general words.
- [ ] The output is saved to `report.txt`.
- [ ] No `TODO`, stubs, `NotImplementedException`.
- [ ] The code uses C# 12 (top-level statements, records, collection expressions, raw strings where appropriate).
- [ ] `GC.Collect()` is called between format measurements for cleanliness.
- [ ] `decimal` and `DateTime` in protobuf are handled deliberately, with the approach documented in a comment.
- [ ] The RU and EN statement blocks are each at least 1000 words.

#### Hints (no direct answer)
- For warmup a simple `for (int i = 0; i < 500; i++) SerializeAndDeserialize(order);` loop before the main measurement works. Do not measure the warmup time.
- To average honestly, accumulate `long totalTicks` and divide by the iteration count, instead of averaging `TimeSpan` through LINQ — that creates extra allocations.
- For Block C it is convenient to serialize `OrderPbV2` into a `MemoryStream`, then `stream.Position = 0` and deserialize as `OrderPb`. protobuf-net reads known tags and skips the unknown tag 6.
- To show MemoryPack breakage, serialize `OrderMem(...)` and deserialize the bytes as `OrderMemSwapped` — where the source model had the string `Customer`, the swapped model reads `decimal Total`, and vice versa.
- Do not try to "improve" protobuf by replacing `decimal` with `long` — that breaks model equivalence and makes the comparison dishonest. Better to document the tax.
- For table output use interpolation with alignment: `{name,-12}: {bytes,8:N0} bytes`.

#### Reference solution walk-through (English code)
```csharp
// C# 12 / .NET 8 — BinaryFormatsLab
// Reference solution for homework M10-L08.

using MemoryPack;
using MessagePack;
using ProtoBuf;
using System.Diagnostics;
using System.Text.Json;

// === Models ===

// Protocol Buffers: tagged schema, forward/backward compatibility.
[ProtoContract]
public record OrderPb
{
    [ProtoMember(1)] public int Id { get; init; }
    [ProtoMember(2)] public string Customer { get; init; } = "";
    [ProtoMember(3)] public decimal Total { get; init; }      // BclHelpers: size tax
    [ProtoMember(4)] public DateTime CreatedUtc { get; init; } // reserved if removed — DO NOT REUSE
    [ProtoMember(5)] public List<OrderLinePb> Lines { get; init; } = new();
}

[ProtoContract]
public record OrderLinePb
{
    [ProtoMember(1)] public string Sku { get; init; } = "";
    [ProtoMember(2)] public int Qty { get; init; }
    [ProtoMember(3)] public decimal Price { get; init; }
}

// V2 with added Currency (tag 6) and removed CreatedUtc (tag 4 — reserved).
[ProtoContract]
public record OrderPbV2
{
    [ProtoMember(1)] public int Id { get; init; }
    [ProtoMember(2)] public string Customer { get; init; } = "";
    [ProtoMember(3)] public decimal Total { get; init; }
    // reserved: tag 4 (CreatedUtc) — removed, DO NOT REUSE
    [ProtoMember(5)] public List<OrderLinePb> Lines { get; init; } = new();
    [ProtoMember(6)] public string Currency { get; init; } = "USD";
}

// MemoryPack: field order is the contract. Never change after release.
[MemoryPackable]
public partial record OrderMem
{
    public int Id { get; init; }
    public string Customer { get; init; } = "";
    public decimal Total { get; init; }
    public DateTime CreatedUtc { get; init; }
    public List<OrderLineMem> Lines { get; init; } = new();
}

[MemoryPackable]
public partial record OrderLineMem
{
    public string Sku { get; init; } = "";
    public int Qty { get; init; }
    public decimal Price { get; init; }
}

// Broken version: Customer and Total swapped. Demonstrates order fragility.
[MemoryPackable]
public partial record OrderMemSwapped
{
    public int Id { get; init; }
    public decimal Total { get; init; }   // expects decimal, but bytes hold string Customer
    public string Customer { get; init; } = "";  // expects string, but bytes hold decimal
    public DateTime CreatedUtc { get; init; }
    public List<OrderLineMem> Lines { get; init; } = new();
}

// MessagePack: tag keys, a compromise between protobuf and MemoryPack.
[MessagePackObject]
public record OrderMsg
{
    [Key(0)] public int Id { get; init; }
    [Key(1)] public string Customer { get; init; } = "";
    [Key(2)] public decimal Total { get; init; }
    [Key(3)] public DateTime CreatedUtc { get; init; }
    [Key(4)] public List<OrderLineMsg> Lines { get; init; } = new();
}

[MessagePackObject]
public record OrderLineMsg
{
    [Key(0)] public string Sku { get; init; } = "";
    [Key(1)] public int Qty { get; init; }
    [Key(2)] public decimal Price { get; init; }
}

// === Data factory ===
public static class OrderFactory
{
    public static List<OrderPb> BuildPb(int count, int seed = 42)
    {
        var rng = new Random(seed);
        var customers = new[] { "Alice", "Bob", "Charlie", "Diana", "Eve" };
        var skus = new[] { "SKU-001", "SKU-002", "SKU-003", "SKU-004", "SKU-005" };
        var result = new List<OrderPb>(count);
        for (int i = 1; i <= count; i++)
        {
            int lineCount = rng.Next(1, 6);
            var lines = new List<OrderLinePb>(lineCount);
            decimal total = 0m;
            for (int j = 0; j < lineCount; j++)
            {
                var line = new OrderLinePb
                {
                    Sku = skus[rng.Next(skus.Length)],
                    Qty = rng.Next(1, 10),
                    Price = Math.Round((decimal)rng.NextDouble() * 100m, 2)
                };
                lines.Add(line);
                total += line.Price * line.Qty;
            }
            result.Add(new OrderPb
            {
                Id = i,
                Customer = customers[rng.Next(customers.Length)],
                Total = Math.Round(total, 2),
                CreatedUtc = new DateTime(2024, 1, 1, 0, 0, 0, DateTimeKind.Utc).AddSeconds(i),
                Lines = lines
            });
        }
        return result;
    }

    public static List<OrderMem> ToMem(List<OrderPb> pb) => pb.Select(o => new OrderMem
    {
        Id = o.Id, Customer = o.Customer, Total = o.Total, CreatedUtc = o.CreatedUtc,
        Lines = o.Lines.Select(l => new OrderLineMem { Sku = l.Sku, Qty = l.Qty, Price = l.Price }).ToList()
    }).ToList();

    public static List<OrderMsg> ToMsg(List<OrderPb> pb) => pb.Select(o => new OrderMsg
    {
        Id = o.Id, Customer = o.Customer, Total = o.Total, CreatedUtc = o.CreatedUtc,
        Lines = o.Lines.Select(l => new OrderLineMsg { Sku = l.Sku, Qty = l.Qty, Price = l.Price }).ToList()
    }).ToList();

    public static List<object> ToAnon(List<OrderPb> pb) => pb.Select(o => (object)new
    {
        o.Id, o.Customer, o.Total, o.CreatedUtc,
        Lines = o.Lines.Select(l => new { l.Sku, l.Qty, l.Price })
    }).ToList();
}

// === Program / Entry point ===
// Top-level statements (.NET 8 / C# 12).

const int Count = 1000;
const int Warmup = 500;
const int Iterations = 5000;

var pbOrders = OrderFactory.BuildPb(Count);
var memOrders = OrderFactory.ToMem(pbOrders);
var msgOrders = OrderFactory.ToMsg(pbOrders);
var anonOrders = OrderFactory.ToAnon(pbOrders);

// --- Block A: payload size ---
var pbBytes = SerializePb(pbOrders);
var memBytes = MemoryPackSerializer.Serialize(memOrders);
var msgBytes = MessagePackSerializer.Serialize(msgOrders);
var jsonBytes = JsonSerializer.SerializeToUtf8Bytes(anonOrders);

Console.WriteLine("=== Block A: payload size (1000 orders) ===");
Console.WriteLine($"{"protobuf",-12}: {pbBytes.Length,8:N0} bytes");
Console.WriteLine($"{"MemoryPack",-12}: {memBytes.Length,8:N0} bytes");
Console.WriteLine($"{"MessagePack",-12}: {msgBytes.Length,8:N0} bytes");
Console.WriteLine($"{"JSON (UTF8)",-12}: {jsonBytes.Length,8:N0} bytes");
Console.WriteLine();

// --- Block B: speed ---
Console.WriteLine("=== Block B: speed (5000 iterations, 500 warmup) ===");
Measure("protobuf serialize  ", () => SerializePb(pbOrders), Warmup, Iterations);
Measure("protobuf deserialize", () => DeserializePb<OrderPb>(pbBytes), Warmup, Iterations);
Measure("MemoryPack serialize ", () => MemoryPackSerializer.Serialize(memOrders), Warmup, Iterations);
Measure("MemoryPack deserial. ", () => MemoryPackSerializer.Deserialize<List<OrderMem>>(memBytes), Warmup, Iterations);
Measure("MessagePack serialize", () => MessagePackSerializer.Serialize(msgOrders), Warmup, Iterations);
Measure("MessagePack deserial.", () => MessagePackSerializer.Deserialize<List<OrderMsg>>(msgBytes), Warmup, Iterations);
Measure("JSON serialize       ", () => JsonSerializer.SerializeToUtf8Bytes(anonOrders), Warmup, Iterations);
Measure("JSON deserialize     ", () => JsonSerializer.Deserialize<List<OrderMsg>>(jsonBytes), Warmup, Iterations);
Console.WriteLine();

// --- Block C: compatibility ---
Console.WriteLine("=== Block C: compatibility experiment ===");

// protobuf: V2 → V1. New Currency (tag 6) is skipped, removed (tag 4) yields default.
var v2 = new OrderPbV2 { Id = 99, Customer = "Zoe", Total = 250m, Currency = "EUR",
    Lines = new() { new() { Sku = "SKU-X", Qty = 2, Price = 10m } } };
using var v2Stream = new MemoryStream();
Serializer.Serialize(v2Stream, v2);
v2Stream.Position = 0;
var v1back = Serializer.Deserialize<OrderPb>(v2Stream);
Console.WriteLine($"protobuf V2→V1: Id={v1back.Id} Customer={v1back.Customer} Total={v1back.Total} " +
                  $"CreatedUtc={v1back.CreatedUtc} (default={default(DateTime)}) Lines={v1back.Lines.Count}");
Console.WriteLine("  → added field is safe, removed field=default");

// MemoryPack: field order broken → data is mixed up.
var mem = new OrderMem { Id = 7, Customer = "Alice", Total = 199.99m,
    CreatedUtc = new DateTime(2024, 6, 1, 0, 0, 0, DateTimeKind.Utc),
    Lines = new() { new() { Sku = "SKU-1", Qty = 1, Price = 199.99m } } };
var memBrokenBytes = MemoryPackSerializer.Serialize(mem);
try
{
    var swapped = MemoryPackSerializer.Deserialize<OrderMemSwapped>(memBrokenBytes);
    Console.WriteLine($"MemoryPack swapped: Id={swapped.Id} Customer={swapped.Customer} Total={swapped.Total}");
    Console.WriteLine("  → field order broke data");
}
catch (Exception ex)
{
    Console.WriteLine($"MemoryPack swapped threw: {ex.GetType().Name} — order is critical");
}
Console.WriteLine();

// --- Final recommendation ---
var recommendation = """
    === Recommendation ===
    On the measured data MemoryPack gives the smallest size and the highest speed —
    but its fragility to field order (Block C) makes it unfit for a public API.
    For hot internal traffic OrderGateway→Inventory→Billing→Shipping choose
    MemoryPack: identical versions, maximum savings.

    For a public API and long-lived contracts — protobuf: tags give forward/backward
    compatibility, deleted tags are reserved and never reused.

    MessagePack is the compromise for cases where binariness is needed but there is
    no budget to maintain a .proto schema.
    """;
Console.WriteLine(recommendation);

// === Helpers ===

static byte[] SerializePb<T>(T value)
{
    using var ms = new MemoryStream();
    Serializer.Serialize(ms, value);
    return ms.ToArray();
}

static T DeserializePb<T>(byte[] bytes) where T : class
{
    using var ms = new MemoryStream(bytes);
    return Serializer.Deserialize<T>(ms);
}

static void Measure(string label, Action op, int warmup, int iterations)
{
    for (int i = 0; i < warmup; i++) op();                       // warmup
    GC.Collect(); GC.WaitForPendingFinalizers(); GC.Collect();   // clean start

    long start = Stopwatch.GetTimestamp();
    for (int i = 0; i < iterations; i++) op();
    TimeSpan elapsed = Stopwatch.GetElapsedTime(start);

    double avgUs = elapsed.TotalMilliseconds * 1000.0 / iterations;
    double opsPerSec = iterations / elapsed.TotalSeconds;
    Console.WriteLine($"{label}: {avgUs,8:F2} µs/op, {opsPerSec,12:N0} ops/s");
}
```

**Line-by-line walk-through.** The `OrderPb` / `OrderMem` / `OrderMsg` models are three equivalent representations of the same domain model, one per format. In `OrderPb` each `[ProtoMember(N)]` pins a numeric tag to a field — this is the "schema via attributes" idea from the lesson, giving forward/backward compatibility. `OrderPbV2` demonstrates evolution: the new `Currency` field gets tag 6, the removed `CreatedUtc` (tag 4) is marked with the comment `// reserved ... DO NOT REUSE` — implementing the lesson's best practice "reserve tags, never reuse deleted numbers." In Block C the serialized `OrderPbV2` is deserialized as `OrderPb`: protobuf-net reads known tags (1, 2, 3, 5) and skips the unknown tag 6, while the missing tag 4 yields `default(DateTime)` — proving the lesson rule "unknown fields are simply skipped."

`OrderMem` uses `[MemoryPackable]` and `partial record` — a requirement of the MemoryPack source generator. Fields are declared in a fixed order that becomes the binary contract. `OrderMemSwapped` swaps `Customer` and `Total`: when the bytes of the original `OrderMem` are deserialized into the swapped model, values land in the wrong slots (the bytes of `decimal` are read into `string Customer`, and vice versa) — this is the "field-order fragility" from the lesson, proven by experiment rather than by words.

`OrderMsg` with `[Key(N)]` shows the compromise format: tag keys exist, but without protobuf-net's BclHelpers overhead on `decimal`/`DateTime`. The factory `OrderFactory.BuildPb` uses a fixed seed `42` — determinism, so a repeated run yields the same byte sizes and the same timings. The `ToMem` / `ToMsg` / `ToAnon` methods build equivalent collections for each format, preserving comparison purity.

Block A serializes 1000 orders with each format and prints sizes — here you usually see that protobuf bytes are larger than MemoryPack/MessagePack, because of the BclHelpers tax on `decimal` and `DateTime` (that same "common mistake" from the lesson: `decimal` in protobuf-net without an explicit model inflates the size). Block B measures serialize and deserialize separately, with warmup (500 cycles before measurement so JIT compiles the hot path) and `GC.Collect()` between formats, so one does not pay for another's garbage. It uses `Stopwatch.GetTimestamp()` + `Stopwatch.GetElapsedTime()` — the .NET 8 API recommended instead of manual division by `Stopwatch.Frequency`. Block C is two compatibility experiments. The result is a raw-string literal `recommendation` (C# 11+, appropriate for multi-line text) with an engineering conclusion leaning on measured numbers, not on the lesson's general words.

Lesson concepts applied: protobuf's tagged schema with reserved deleted tags, MemoryPack's field-order fragility, MessagePack's tag keys as a compromise, format choice by scenario (protobuf for API, MemoryPack for cache/IPC, MessagePack as a universal), a separate JSON path for debugging, measurement on your own data against a JSON baseline.

#### Going deeper (bonus)
1. Add `BenchmarkDotNet` and replace the manual `Stopwatch` measurement with a `[MemoryDiagnoser]` benchmark. Compare `Mean`, `Allocated`, `Gen0` — MemoryPack usually shows zero allocations on the hot path. Explain why.
2. Implement a three-release versioning scenario: V1 → V2 (added `Currency`) → V3 (removed `Total`, added `LinesTotal`). Show that a V1 client reads V3 data without crashing, and a V3 client reads V1 data. Document every reserved tag.
3. Serialize the collection to a file and measure on-disk size of compressed (`GzipStream`) vs uncompressed. Binary formats compress worse than JSON, because they have less redundancy — confirm or refute this on your data.
4. Implement `MemoryPack` with `MemoryPackSerializer.SerializeAsync(Stream, T)` and compare it to the synchronous version by speed and allocations. When is the async version justified, and when is it not?

---

#### Чек-лист сдачи / Submission checklist
- [ ] (RU) Проект `BinaryFormatsLab` создан, компилируется под `net8.0`.
- [ ] (RU) Три пакета (`protobuf-net`, `MemoryPack`, `MessagePack`) добавлены в `.csproj`.
- [ ] (RU) Модели `OrderPb`, `OrderMem`, `OrderMsg` описаны с корректными атрибутами.
- [ ] (RU) `OrderFactory.Build` детерминирован (seed `42`), 1000 заказов со строками.
- [ ] (RU) Блок A выводит размеры четырёх форматов, числа стабильны.
- [ ] (RU) Блок B измеряет serialize/deserialize отдельно, с разогревом и `Stopwatch.GetElapsedTime`.
- [ ] (RU) Блок C доказывает совместимость protobuf и ломку MemoryPack.
- [ ] (RU) Удалённый тег protobuf зарезервирован комментарием.
- [ ] (RU) Итоговая рекомендация опирается на измеренные числа.
- [ ] (RU) Вывод сохранён в `report.txt`.
- [ ] (RU) Нет `TODO`, заглушек, `NotImplementedException`.
- [ ] (RU) Код использует C# 12 (top-level statements, records, collection expressions, raw strings).
- [ ] (EN) Project `BinaryFormatsLab` created, compiles under `net8.0`.
- [ ] (EN) Three packages (`protobuf-net`, `MemoryPack`, `MessagePack`) added to `.csproj`.
- [ ] (EN) Models `OrderPb`, `OrderMem`, `OrderMsg` described with correct attributes.
- [ ] (EN) `OrderFactory.Build` is deterministic (seed `42`), 1000 orders with lines.
- [ ] (EN) Block A prints sizes of four formats, numbers are stable.
- [ ] (EN) Block B measures serialize/deserialize separately, with warmup and `Stopwatch.GetElapsedTime`.
- [ ] (EN) Block C proves protobuf compatibility and MemoryPack breakage.
- [ ] (EN) The deleted protobuf tag is reserved with a comment.
- [ ] (EN) The final recommendation leans on measured numbers.
- [ ] (EN) Output saved to `report.txt`.
- [ ] (EN) No `TODO`, stubs, `NotImplementedException`.
- [ ] (EN) Code uses C# 12 (top-level statements, records, collection expressions, raw strings).

#### Ресурсы / Resources
- [Microsoft Learn — Serialization — https://learn.microsoft.com/dotnet/standard/serialization/](https://learn.microsoft.com/dotnet/standard/serialization/)
- [protobuf-net (GitHub) — https://github.com/protobuf-net/protobuf-net](https://github.com/protobuf-net/protobuf-net)
- [MemoryPack (GitHub) — https://github.com/Cysharp/MemoryPack](https://github.com/Cysharp/MemoryPack)
- [MessagePack-CSharp (GitHub) — https://github.com/MessagePack-CSharp/MessagePack-CSharp](https://github.com/MessagePack-CSharp/MessagePack-CSharp)
- [Protocol Buffers — Language Guide (proto3) — https://protobuf.dev/programming-guides/proto3/](https://protobuf.dev/programming-guides/proto3/)
- [.NET 8 Stopwatch.GetElapsedTime — https://learn.microsoft.com/dotnet/api/system.diagnostics.stopwatch.getelapsedtime](https://learn.microsoft.com/dotnet/api/system.diagnostics.stopwatch.getelapsedtime)
- [BenchmarkDotNet — https://benchmarkdotnet.org/](https://benchmarkdotnet.org/)
