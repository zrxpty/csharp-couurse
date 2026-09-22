[⬆ К модулю M06](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M06-L08: Span<T>/Memory<T> (обзор, Optional) / Span<T>/Memory<T> (overview, Optional)

**Модуль / Module:** M06
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

`Span<T>` и `Memory<T>` — это типы для работы с непрерывными областями памяти без создания лишних копий. Главная идея: дать единое «окно» поверх уже существующих данных — массива, строки, участка стека или неуправляемого буфера — и работать с ним как с обычным массивом, но без аллокаций в управляемой куче.

**Аналогия с окном.** Представь длинный дом (массив на 1000 элементов). Тебе нужен взгляд только на комнаты с 10 по 30. Вместо того чтобы строить новый домик и переносить туда мебель (копировать массив), ты ставишь раму-окно — `Span<T>`, которая «смотрит» ровно на нужный участок. Данные те же, копий нет, окна можно двигать и сужать.

**Span<T> — это ref struct.** Это ключевое ограничение. `ref struct` живёт только на стеке и не может быть полем обычного класса или структуры, не может быть захвачен в замыкание, не может участвовать в `async/await` и не может быть упакован (boxed). Почему так строго? Потому что `Span<T>` внутри хранит управляемые ссылки (`ref`), и сборщик мусора должен корректно обновлять их при перемещении объектов в куче. Если бы `Span` мог «убежать» в кучу или пережить await, ссылки могли бы стать невалидными. Компилятор физически запрещает такие сценарии.

**Основные источники данных для Span:**
- `array.AsSpan()` — окно над массивом или его частью (`AsSpan(start, length)`).
- `"hello".AsSpan()` — окно над строкой без копирования символов (строка в .NET уже лежит в памяти непрерывно как UTF-16).
- `stackalloc byte[64]` — небольшой буфер прямо на стеке, без аллокации в куче. Идеален для коротких промежуточных операций (например, форматирование небольшого числа в символы).
- `new Span<T>(ptr, length)` — окно над неуправляемой памятью.

**Зачем stackalloc?** Когда нужно 16–64 байта на короткое время, вызывать `new byte[64]` значит дёргать сборщик мусора. `stackalloc` размещает память на стеке метода, она автоматически освобождается при выходе, и `Span<byte>` по ней работает как обычный массив. Главное — не возвращать такой `Span` из метода и не отдавать в асинхронный код.

**Memory<T> для async.** `Span<T>` нельзя использовать внутри `async`-методов (он не может пережить `await`). Для асинхронных сценариев придумали `Memory<T>` — это «обёртка над окном», которая может храниться в куче, быть полем класса, захватываться в замыкания. Чтобы поработать с данными синхронно в конкретный момент, ты вызываешь `memory.Span` и получаешь `Span<T>` — но только синхронно, без пересечения `await`.

**Срезы и ReadOnlySpan.** `ReadOnlySpan<T>` — неизменяемая версия, её дают строки и массивы, которые ты не планируешь менять. Срезы (`span.Slice(start, length)`) не копируют данные, они создают новое окно над тем же участком. Это основа高性能 парсинга: вместо `Substring` (копирование) используют `AsSpan().Slice()`.

**Когда применять.** Высоконагруженный парсинг, обработка буферов ввода-вывода, текстовые операции на горячих путях, работа с неуправляемой памятью. На обычном бизнес-коде разница незаметна — не усложняй без необходимости. Помни правило: `Span<T>` — для синхронного горячего пути, `Memory<T>` — когда нужно передать буфер через асинхронную границу.

#### Theory (EN)

`Span<T>` and `Memory<T>` are types for working with contiguous regions of memory without producing extra copies. The core idea is to provide a single "window" over already existing data — an array, a string, a stack buffer, or unmanaged memory — and operate on it like a regular array, but with no allocations on the managed heap.

**The window analogy.** Imagine a long building (an array of 1000 elements). You only need to look at rooms 10 through 30. Instead of constructing a new building and carrying the furniture over (copying the array), you place a window frame — `Span<T>` — that looks exactly at the slice you need. The data is the same, there are no copies, and the window can be moved and narrowed.

**Span<T> is a ref struct.** This is the key restriction. A `ref struct` lives only on the stack and cannot be a field of an ordinary class or struct, cannot be captured in a closure, cannot participate in `async/await`, and cannot be boxed. Why so strict? Because `Span<T>` internally holds managed references (`ref`), and the garbage collector must correctly update them when objects move inside the heap. If a `Span` could "escape" into the heap or survive an `await`, those references could become invalid. The compiler physically forbids such scenarios.

**Main data sources for Span:**
- `array.AsSpan()` — a window over the whole array or a part of it (`AsSpan(start, length)`).
- `"hello".AsSpan()` — a window over a string without copying characters (a .NET string already lives in memory as contiguous UTF-16).
- `stackalloc byte[64]` — a small buffer directly on the stack, with no heap allocation. Ideal for short intermediate operations, like formatting a small number into characters.
- `new Span<T>(ptr, length)` — a window over unmanaged memory.

**Why stackalloc?** When you need 16–64 bytes for a brief moment, calling `new byte[64]` means putting pressure on the garbage collector. `stackalloc` places the memory on the method's stack; it is released automatically on return, and a `Span<byte>` over it behaves like a normal array. The rule: never return such a `Span` from a method and never hand it to asynchronous code.

**Memory<T> for async.** `Span<T>` cannot be used inside `async` methods (it cannot survive an `await`). For asynchronous scenarios, `Memory<T>` was introduced — it is a "wrapper around a window" that can be stored on the heap, be a field of a class, and be captured in closures. When you need to process the data synchronously at a specific moment, you call `memory.Span` and get a `Span<T>` — but only synchronously, without crossing an `await`.

**Slices and ReadOnlySpan.** `ReadOnlySpan<T>` is the immutable version; strings and arrays you do not intend to mutate return it. Slices (`span.Slice(start, length)`) do not copy data — they create a new window over the same region. This is the foundation of high-performance parsing: instead of `Substring` (a copy) you use `AsSpan().Slice()`.

**When to use.** High-throughput parsing, I/O buffer processing, text operations on hot paths, and unmanaged-memory work. On ordinary business code the difference is invisible — do not add complexity without reason. Remember the rule: `Span<T>` is for the synchronous hot path; `Memory<T>` is for passing a buffer across an asynchronous boundary.

#### Пример кода / Code Example

```csharp
// C# 12 / .NET 8 — Span<T> и Memory<T>: обзор
// C# 12 / .NET 8 — Span<T> and Memory<T>: overview
using System;
using System.Text;
using System.Threading.Tasks;

public static class SpanMemoryDemo
{
    // 1) Окно над массивом без копирования / Window over an array without copying
    public static int SumWindow(int[] numbers)
    {
        // Берём срез 2..6 — копии нет / Take slice 2..6 — no copy
        Span<int> window = numbers.AsSpan(start: 2, length: 4);

        int sum = 0;
        foreach (var n in window)   // Перебираем «окно», как массив / Iterate the "window" like an array
            sum += n;
        return sum;
    }

    // 2) Окно над строкой: парсинг без Substring / Window over a string: parsing without Substring
    public static int ParsePort(ReadOnlySpan<char> text)
    {
        // Формат "host:port", ищем ':' / Format "host:port", find ':'
        int colon = text.IndexOf(':');
        if (colon < 0) return -1;

        ReadOnlySpan<char> portSpan = text.Slice(colon + 1); // Срез без аллокации / Slice without allocation
        return int.Parse(portSpan);                          // int.Parse умеет принимать Span / int.Parse accepts Span
    }

    // 3) stackalloc: маленький буфер на стеке / stackalloc: a small buffer on the stack
    public static string FormatHex(byte value)
    {
        // 16 байт на стеке — GC не участвует / 16 bytes on the stack — GC is not involved
        Span<byte> buffer = stackalloc byte[16];

        // Заполняем символами hex / Fill with hex characters
        buffer[0] = (byte)'0';
        buffer[1] = (byte)'x';

        int pos = 2;
        buffer[pos++] = HexChar(value >> 4);   // Старшая тетрада / High nibble
        buffer[pos++] = HexChar(value & 0x0F); // Младшая тетрада / Low nibble

        // Копируем в строку только при возврате / Copy to a string only on return
        return Encoding.ASCII.GetString(buffer[..pos]);
    }

    private static byte HexChar(int n) => (byte)(n < 10 ? '0' + n : 'A' + n - 10);

    // 4) Memory<T> для async: буфер переживает await / Memory<T> for async: the buffer survives await
    public static async Task<int> CountZerosAsync(Memory<byte> buffer)
    {
        // Представим, что данные читаются асинхронно / Imagine the data is read asynchronously
        await Task.Delay(10); // имитация I/O — Span здесь использовать НЕЛЬЗЯ / simulated I/O — Span CANNOT be used here

        // Синхронный доступ к окну через .Span / Synchronous access to the window via .Span
        ReadOnlySpan<byte> view = buffer.Span;
        int zeros = 0;
        foreach (var b in view)
            if (b == 0) zeros++;
        return zeros;
    }

    // 5) НЕЛЬЗЯ: так компилятор не даст собрать / INVALID: the compiler will reject this
    // public static async Task<int> BadAsync(Span<byte> s)
    // {
    //     await Task.Delay(1);
    //     return s.Length; // Ошибка CS4007: Span нельзя использовать в async / Error: Span cannot be used in async
    // }
}

// Демонстрация / Demonstration
public class Program
{
    public static void Main()
    {
        int[] data = { 1, 2, 3, 4, 5, 6, 7, 8 };
        Console.WriteLine(SpanMemoryDemo.SumWindow(data));      // 3+4+5+6 = 18

        Console.WriteLine(SpanMemoryDemo.ParsePort("localhost:8080".AsSpan())); // 8080

        Console.WriteLine(SpanMemoryDemo.FormatHex(0xAB));      // 0xAB

        byte[] buf = { 0, 5, 0, 7, 0 };
        Console.WriteLine(SpanMemoryDemo.CountZerosAsync(buf).Result); // 3
    }
}
```

#### Best Practices

- Используй `Span<T>` на горячих путях парсинга и обработки буферов; на обычном бизнес-коде это избыточно.
- Предпочитай `ReadOnlySpan<T>`, когда данные менять не нужно, — это выражает намерение и безопаснее.
- Бери `stackalloc` только для небольших буферов (десятки байт), никогда не возвращай такой `Span` из метода.
- Для `async`-границ используй `Memory<T>` и только внутри синхронных сегментов получай `.Span`.
- Передавай `Span` по значению (это дешёвая структура-окно) и не храни его в полях.

- Use `Span<T>` on hot parsing and buffer-processing paths; on ordinary business code it is overkill.
- Prefer `ReadOnlySpan<T>` when you do not need to mutate the data — it expresses intent and is safer.
- Use `stackalloc` only for small buffers (tens of bytes) and never return such a `Span` from a method.
- Across `async` boundaries use `Memory<T>` and obtain `.Span` only inside synchronous segments.
- Pass `Span` by value (it is a cheap window struct) and never store it in fields.

#### Частые ошибки / Common Mistakes

- Попытка сохранить `Span<T>` в поле класса или структуры → `ref struct` не может быть полем; используй `Memory<T>` для хранения.
- Использование `Span<T>` внутри `async`-метода → ошибка компиляции CS4007; переведи буфер в `Memory<T>`.
- Возврат `Span` поверх `stackalloc` из метода → стек будет очищен, ссылка станет невалидной; возвращай копию (например, строку).
- Вызов `Substring` там, где достаточно `AsSpan().Slice()` → лишняя аллокация строки; используй срезы.
- Случайное упаковка (`boxing`) `Span` через обобщённые методы/`object` → компилятор это запретит; не передавай `Span` в обобщённые контейнеры.

- Trying to store a `Span<T>` in a class or struct field → a `ref struct` cannot be a field; use `Memory<T>` for storage.
- Using a `Span<T>` inside an `async` method → compiler error CS4007; convert the buffer to `Memory<T>`.
- Returning a `Span` over `stackalloc` from a method → the stack is reclaimed and the reference becomes invalid; return a copy (e.g., a string).
- Calling `Substring` where `AsSpan().Slice()` would suffice → an unnecessary string allocation; use slices.
- Accidentally boxing a `Span` via generic methods or `object` → the compiler will forbid it; do not pass `Span` into generic containers.

#### Чек-лист самопроверки / Self-check Checklist

- [ ] Я могу объяснить, почему `Span<T>` не аллоцирует в куче.
- [ ] Я знаю три источника данных для `Span`: массив, строка, `stackalloc`.
- [ ] Я понимаю, что `ref struct` нельзя хранить в полях и использовать в `async`.
- [ ] Я выбираю `Memory<T>`, когда буфер должен пересечь границу `await`.
- [ ] Я использую `ReadOnlySpan<T>` для неизменяемых данных.
- [ ] Я не возвращаю `Span` поверх `stackalloc` из метода.

- [ ] I can explain why `Span<T>` does not allocate on the heap.
- [ ] I know three data sources for `Span`: array, string, `stackalloc`.
- [ ] I understand that a `ref struct` cannot be stored in fields or used in `async`.
- [ ] I choose `Memory<T>` when a buffer must cross an `await` boundary.
- [ ] I use `ReadOnlySpan<T>` for immutable data.
- [ ] I never return a `Span` over `stackalloc` from a method.

#### Ресурсы / Resources

- [Microsoft Learn — https://learn.microsoft.com/dotnet/api/system.span-1]

---

[⬆ К модулю M06](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
