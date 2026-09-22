---
[← К уроку M06-L08](lesson-M06-L08-span-memory.md) | [⬆ К модулю M06](../README.md) | [Следующее ДЗ →]()
---

### Домашнее задание M06-L08: Span<T>/Memory<T> (обзор, Optional) / Homework M06-L08: Span<T>/Memory<T> (overview, Optional)

**Урок / Lesson:** M06-L08
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 3/5
**Цель / Goal:** (RU) На практике освоить «оконную» модель памяти .NET 8: научиться открывать `Span<T>`/`ReadOnlySpan<T>` над массивами, строками и `stackalloc`-буферами, нарезать срезы без копий, выбрать `Memory<T>` там, где буфер должен пережить `await`, и научиться предугадывать ошибки компилятора, связанные с ограничениями `ref struct`. (EN) Gain hands-on command of .NET 8's "window" memory model: open `Span<T>`/`ReadOnlySpan<T>` over arrays, strings and `stackalloc` buffers, slice without copies, choose `Memory<T>` where a buffer must survive `await`, and learn to anticipate the compiler errors imposed by `ref struct` restrictions.

#### Связь с уроком / Connection to the lesson

(RU) Урок вводит `Span<T>` и `Memory<T>` через «аналогию с окном»: вместо копирования данных мы ставим раму над уже существующим участком памяти. Домашнее задание заставляет применить все четыре источника данных из урока — массив, строка, `stackalloc` и `Memory<T>` через `await` — в одном связном проекте, а также воспроизвести и объяснить типичные ошибки (CS4007, попытка сохранить `Span` в поле, возврат `stackalloc`-`Span` из метода).

(EN) The lesson introduces `Span<T>` and `Memory<T>` through the "window analogy": instead of copying data we place a frame over an existing memory region. The homework makes you apply all four data sources from the lesson — array, string, `stackalloc` and `Memory<T>` across `await` — inside one cohesive project, and to reproduce and explain the classic mistakes (CS4007, storing a `Span` in a field, returning a `stackalloc`-backed `Span` from a method).

---

## Постановка на русском / Russian statement

#### Контекст и мотивация

Ты разработчик мини-сервиса, который принимает «командные строки» сетевого протокола вида `host:port;host:port;...` и подготавливает к отправке бинарные пакеты. Каждая операция лежит на горячем пути: сервис обрабатывает тысячи строк в секунду, и каждая лишняя аллокация строки или массива байт напрямую режет пропускную способность. Команда уже заметила, что наивная реализация на `String.Split`, `Substring` и `new byte[N]` создаёт давление на сборщик мусора и увеличивает паузы.

Урок M06-L08 даёт тебе инструмент — `Span<T>` и `Memory<T>`, — который позволяет смотреть на уже существующую память как на массив, не копируя её. Это ровно то, что нужно: распарсить строку как серию срезов, просуммировать окно над целочисленным массивом без копии, отформатировать байт в hex прямо на стеке и переслать буфер через асинхронную границу `await`. В этом задании ты соберёшь все четыре сценария в один проект, чтобы увидеть, как «окна» заменяют копии, где граница между `Span` и `Memory` проходит физически (ошибка компилятора CS4007), и почему `stackalloc` нельзя возвращать наружу. Цель — не написать «ещё один парсер», а прочувствовать модель памяти на уровне, где компилятор становится твоим наставником.

#### Что нужно сделать (пошагово)

1. Создай консольный проект на .NET 8 через `dotnet new console -n SpanMemoryHomework -o SpanMemoryHomework` и перейди в него. Убедись, что в `SpanMemoryHomework.csproj` стоит `<TargetFramework>net8.0</TargetFramework>` и включёнNullable: `<Nullable>enable</Nullable>`. Используй C# 12 (top-level statements допустимы в `Program.cs`, но классы выноси в отдельные файлы).

2. Создай класс `ProtocolParser` со статическими методами, каждый из которых иллюстрирует отдельный источник данных для `Span`:
   - `int SumWindow(int[] numbers, int start, int length)` — берёт `numbers.AsSpan(start, length)` и возвращает сумму среза; продемонстрируй, что `foreach` по `Span<int>` работает как по массиву.
   - `int ParsePort(ReadOnlySpan<char> text)` — ищет `':'` через `IndexOf`, нарезает срез после двоеточия через `Slice` и парсит `int.Parse(portSpan)`. Верни `-1`, если двоеточия нет.
   - `int CountEndpoints(ReadOnlySpan<char> line)` — считает, сколько пар `host:port` разделено `';'`, используя только срезы (без `Split`, без `Substring`). Верни число пар.
   - `string FormatHex(byte value)` — использует `Span<byte> buffer = stackalloc byte[16];`, заполняет `"0x"` и две hex-цифры, возвращает строку через `Encoding.ASCII.GetString(buffer[..pos])`. Никакого `new byte[]`.

3. Создай класс `AsyncBufferOps` с методом `async Task<int> CountZerosAsync(Memory<byte> buffer)`, который внутри делает `await Task.Delay(10)` (имитация I/O) и затем читает `ReadOnlySpan<byte> view = buffer.Span;`, считая нули. Добавь также `async Task<int> CountZerosAsync(byte[] buffer)`, который делегирует в первый, передав `buffer` как `Memory<byte>` (покажи неявное преобразование массива в `Memory<T>`).

4. В `Program.cs` подготовь тестовые данные: массив `{1,2,3,4,5,6,7,8}`, строку `"localhost:8080;api.example.com:443;cache:6379"`, байт `0xAB` и буфер `byte[] {0, 5, 0, 7, 0}`. Выведи результаты каждого метода через `Console.WriteLine`. Ожидаемые выводы: сумма окна `[2..6]` = `3+4+5+6 = 18`, порт первой пары = `8080`, число пар = `3`, hex = `0xAB`, число нулей = `3`.

5. В отдельном файле `InvalidExamples.cs` помести закомментированный «плохой» код с пояснениями, почему он не компилируется: (а) метод `async Task<int> BadAsync(Span<byte> s)`, использующий `s` после `await` (CS4007); (б) класс с полем `private Span<byte> _cache;` (ref struct не может быть полем); (в) метод, возвращающий `Span<byte>` поверх `stackalloc`. Раскомментируй по очереди, убедись через `dotnet build`, что компилятор ругается именно так, как описано в уроке, и снова закомментируй.

6. Запусти `dotnet run` и проверь, что все пять выводов совпадают с ожидаемыми. Запусти `dotnet build -c Release` — он должен собираться без предупреждений, связанных с `Span`. Если появится предупреждение CA или анализатора про `Span` в `async` — разберись.

7. Напиши модульные тесты в проекте `SpanMemoryHomework.Tests` (`dotnet new xunit`), добавь ссылку на основной проект. Покрой `SumWindow`, `ParsePort`, `CountEndpoints`, `FormatHex` и `CountZerosAsync`. Используй `Assert.Equal`. Проверь краевые случаи: пустая строка, строка без двоеточия, окно длиной 0, буфер без нулей.

#### Требования к решению

Решение должно использовать ровно те конструкции, которые описаны в уроке: `AsSpan`, `Slice` (или оператор среза `..`), `IndexOf`, `int.Parse(ReadOnlySpan<char>)`, `stackalloc`, `Memory<T>.Span`. Запрещено использовать `String.Split`, `Substring`, `new string(char[])` для промежуточных значений и `new byte[N]` там, где достаточно `stackalloc`. Все публичные методы, принимающие текст, должны принимать `ReadOnlySpan<char>` (а не `string`), кроме точек входа в `Program.cs`, где `string` неизбежен — там вызывай `.AsSpan()`.

Код должен компилироваться под .NET 8 / C# 12 без ошибок и без предупреждений, связанных с `Span`. Типы `Span<T>`/`ReadOnlySpan<T>` должны передаваться по значению (это дешёвые структуры-окна), а не по `ref`, если только не нужна намеренная производительность. Для `async`-методов разрешён только `Memory<T>`; обращение к `.Span` должно происходить строго в синхронных сегментах после `await`. Структура проекта: библиотечный код в классах, `Program.cs` — только демонстрация. Тесты должны запускаться через `dotnet test` и проходить.

Имена файлов и классов должны совпадать с теми, что в пошаговом списке: `ProtocolParser.cs`, `AsyncBufferOps.cs`, `InvalidExamples.cs`, `Program.cs`. Не оставляй закомментированного рабочего кода в финальной сборке, кроме намеренно демонстрационного `InvalidExamples.cs`. Каждое использование `stackalloc` должно сопровождаться комментарием, почему буфер не возвращается наружу. Каждый переход `Span` → `Memory` (или обратный) — комментарием о границе `await`.

#### Тонкости и подводные камни

- `Span<T>` — это `ref struct`, поэтому его **нельзя** сохранить в поле класса или структуры, нельзя упаковать (boxing), нельзя захватить в лямбду/замыкание и нельзя использовать через `await`. Если попробуешь — получишь ошибку компиляции CS4007 (или близкую), а не рантайм-краш. Это защита памяти, а не неудобство.
- `stackalloc` размещает память на стеке метода. Возвращать `Span` над ним из метода нельзя: при выходе из метода стек очищается, и ссылка станет невалидной. Возвращай **копию** (например, строку через `Encoding.ASCII.GetString`), как в `FormatHex`.
- `int.Parse` в .NET имеет перегрузку, принимающую `ReadOnlySpan<char>` — это ключевая причина, почему парсинг на срезах быстрее `Substring` + `int.Parse(string)`: нет промежуточной аллокации строки.
- Срезы `span.Slice(start, length)` и оператор `span[start..end]` **не копируют** данные. Но следи за тем, чтобы индексы были валидны: выход за границы даст `ArgumentOutOfRangeException`. `AsSpan(start, length)` тоже проверяет границы.
- `Memory<T>` не «магически» переживает `await` безопаснее `Span` — он просто может храниться в куче. Получить `Span` из `Memory` можно через свойство `.Span`, но только синхронно. Если передашь `Memory` в другой `async`-метод, всё работает; если попытаешься удержать `.Span` через `await` — снова CS4007.
- Не путай `ReadOnlySpan<T>` и `Span<T>`: строка даёт только `ReadOnlySpan<char>` (строка неизменна). Массив даёт оба варианта в зависимости от перегрузки `AsSpan`. Выбирай `ReadOnly` по умолчанию — это выражает намерение и безопаснее.
- Boxing `Span` через обобщённые методы (например, `List<object>` или `IEnumerable<T>`) невозможен — компилятор запретит. Не пытайся положить `Span` в коллекцию.

#### Критерии приёмки

- [ ] Проект `SpanMemoryHomework` собирается под .NET 8 / C# 12 без ошибок и без предупреждений про `Span`.
- [ ] `ProtocolParser.SumWindow` использует `AsSpan(start, length)` и `foreach` по `Span<int>`, возвращает сумму окна.
- [ ] `ProtocolParser.ParsePort` принимает `ReadOnlySpan<char>`, использует `IndexOf(':')`, `Slice` и `int.Parse(ReadOnlySpan<char>)`, без `Substring`.
- [ ] `ProtocolParser.CountEndpoints` считает пары, разделённые `';'`, только срезами, без `Split`.
- [ ] `ProtocolParser.FormatHex` использует `stackalloc byte[16]`, не возвращает `Span` наружу, возвращает строку.
- [ ] `AsyncBufferOps.CountZerosAsync(Memory<byte>)` содержит `await` и обращается к `buffer.Span` строго после `await`, в синхронном сегменте.
- [ ] Перегрузка `CountZerosAsync(byte[])` делегирует в `Memory<byte>`-версию, демонстрируя неявное преобразование массива.
- [ ] `InvalidExamples.cs` содержит три закомментированных «плохих» примера с пояснениями; при раскомментировании каждого — соответствующая ошибка компиляции.
- [ ] `Program.cs` выводит ровно: `18`, `8080`, `3`, `0xAB`, `3`.
- [ ] `dotnet test` проходит; есть тесты на краевые случаи (пустая строка, нет двоеточия, окно длиной 0, буфер без нулей).
- [ ] Все публичные текстовые методы принимают `ReadOnlySpan<char>`, а не `string` (кроме точек входа в `Program.cs`).
- [ ] В коде нет `String.Split`, `Substring`, `new byte[N]` там, где нужен `stackalloc`, и `new string(char[])` для промежуточных значений.
- [ ] Каждый `stackalloc` снабжён комментарием о запрете возврата `Span` наружу.
- [ ] Каждый переход между `Span` и `Memory` снабжён комментарием о границе `await`.
- [ ] `dotnet build -c Release` не выдаёт предупреждений, связанных с `Span`/`Memory`.

#### Подсказки (без прямого ответа)

- Вспомни, что `IndexOf` у `ReadOnlySpan<char>` возвращает индекс первого вхождения символа или `-1`. Этого достаточно, чтобы найти и `':'`, и `';'`.
- Чтобы посчитать пары, разделённые `';'`, тебе не нужен `Split`: проходи по строке срезами, нарезая от текущей позиции до следующего `';'`.
- Для hex: старшая тетрада — это `value >> 4`, младшая — `value & 0x0F`. Символ для `0..9` — `'0' + n`, для `10..15` — `'A' + n - 10`.
- `Memory<byte>` получается из `byte[]` неявно; обратно — `memory.Span`. Думай о `Memory` как о «коробке, в которой лежит окно, но само окно можно достать только синхронно».
- Если компилятор ругается CS4007 в `async`-методе — это сигнал, что ты пытаешься удержать `Span` через `await`. Замени параметр на `Memory<T>`.
- Не пытайся вернуть `Span<byte>` из метода с `stackalloc`: верни `string` или `byte[]` (копию), как делает `FormatHex`.

#### Эталонное решение (разбор)

```csharp
// C# 12 / .NET 8 — Эталонное решение ДЗ M06-L08
// C# 12 / .NET 8 — Reference solution for Homework M06-L08
using System;
using System.Text;
using System.Threading.Tasks;

// 1) Синхронный парсер на срезах / Synchronous slice-based parser
public static class ProtocolParser
{
    // Окно над массивом без копии / Window over an array, no copy
    public static int SumWindow(int[] numbers, int start, int length)
    {
        Span<int> window = numbers.AsSpan(start, length); // окно, не копия / a window, not a copy
        int sum = 0;
        foreach (var n in window) sum += n;               // перебор как по массиву / iterate like an array
        return sum;
    }

    // Парсинг порта из "host:port" без Substring / Parse port from "host:port" without Substring
    public static int ParsePort(ReadOnlySpan<char> text)
    {
        int colon = text.IndexOf(':');                    // индекс ':' / index of ':'
        if (colon < 0) return -1;                         // нет двоеточия / no colon
        ReadOnlySpan<char> portSpan = text[(colon + 1)..]; // срез без аллокации / slice, no allocation
        return int.Parse(portSpan);                       // перегрузка под Span / Span overload
    }

    // Подсчёт пар "host:port" через ';' только срезами / Count "host:port" pairs by ';' using slices only
    public static int CountEndpoints(ReadOnlySpan<char> line)
    {
        int count = 0;
        while (!line.IsEmpty)
        {
            int semi = line.IndexOf(';');                 // ищем разделитель / find separator
            ReadOnlySpan<char> pair = semi < 0 ? line : line[..semi];
            if (pair.IndexOf(':') >= 0) count++;          // валидная пара / a valid pair
            line = semi < 0 ? default : line[(semi + 1)..]; // двигаем окно / advance the window
        }
        return count;
    }

    // Hex-форматирование через stackalloc / Hex formatting via stackalloc
    public static string FormatHex(byte value)
    {
        Span<byte> buffer = stackalloc byte[16];          // буфер на стеке, GC не участвует / stack buffer, no GC
        buffer[0] = (byte)'0';
        buffer[1] = (byte)'x';
        int pos = 2;
        buffer[pos++] = HexChar(value >> 4);              // старшая тетрада / high nibble
        buffer[pos++] = HexChar(value & 0x0F);            // младшая тетрада / low nibble
        return Encoding.ASCII.GetString(buffer[..pos]);   // копия только при возврате / copy only on return
        // ВАЖНО: не возвращаем Span над stackalloc — стек будет очищен / IMPORTANT: never return the stackalloc Span
    }

    private static byte HexChar(int n) => (byte)(n < 10 ? '0' + n : 'A' + n - 10);
}

// 2) Асинхронная обработка буфера через Memory<T> / Async buffer processing via Memory<T>
public static class AsyncBufferOps
{
    // Memory<T> переживает await / Memory<T> survives await
    public static async Task<int> CountZerosAsync(Memory<byte> buffer)
    {
        await Task.Delay(10);                             // имитация I/O — Span тут использовать нельзя / simulated I/O — Span cannot be used here
        ReadOnlySpan<byte> view = buffer.Span;            // синхронный доступ к окну / synchronous access to the window
        int zeros = 0;
        foreach (var b in view) if (b == 0) zeros++;
        return zeros;
    }

    // Перегрузка для массива: неявное преобразование byte[] -> Memory<byte> / Array overload: implicit byte[] -> Memory<byte>
    public static Task<int> CountZerosAsync(byte[] buffer) => CountZerosAsync((Memory<byte>)buffer);
}

// 3) Демонстрация / Demonstration
public static class Program
{
    public static void Main()
    {
        int[] data = { 1, 2, 3, 4, 5, 6, 7, 8 };
        Console.WriteLine(ProtocolParser.SumWindow(data, 2, 4));                 // 18

        Console.WriteLine(ProtocolParser.ParsePort("localhost:8080".AsSpan()));  // 8080

        ReadOnlySpan<char> line = "localhost:8080;api.example.com:443;cache:6379".AsSpan();
        Console.WriteLine(ProtocolParser.CountEndpoints(line));                  // 3

        Console.WriteLine(ProtocolParser.FormatHex(0xAB));                       // 0xAB

        byte[] buf = { 0, 5, 0, 7, 0 };
        Console.WriteLine(AsyncBufferOps.CountZerosAsync(buf).Result);           // 3
    }
}
```

Разбор по строкам. В `SumWindow` строка `numbers.AsSpan(start, length)` создаёт `Span<int>` — структуру из трёх полей (ссылка, индекс, длина), которая «смотрит» на участок массива без копирования. `foreach` по `Span<int>` компилируется в эффективный индексный доступ, эквивалентный циклу по массиву, но без аллокаций. Это иллюстрирует тезис урока: «окно» заменяет копию.

В `ParsePort` `text.IndexOf(':')` — это метод `ReadOnlySpan<char>`, возвращающий индекс без аллокации. Срез `text[(colon + 1)..]` использует оператор диапазона C# 8+, который для `Span` компилируется в `Slice` без копии. Ключевой момент — `int.Parse(portSpan)`: в уроке подчёркнуто, что `int.Parse` имеет перегрузку под `ReadOnlySpan<char>`, поэтому мы избегаем промежуточной строки, которую создал бы `Substring`. Это и есть «высокопроизводительный парсинг» из теории.

`CountEndpoints` расширяет идею: вместо `Split(';')` (который аллоцирует массив строк) мы вручную двигаем окно `line` через `line[(semi + 1)..]`. Каждая итерация — новый срез над той же памятью. Условие `pair.IndexOf(':') >= 0` фильтрует невалидные пары. Метод демонстрирует, что срезы можно «нарезать» рекурсивно, не создавая ни одной аллокации.

`FormatHex` — единственное место с `stackalloc`. Строка `Span<byte> buffer = stackalloc byte[16];` размещает 16 байт на стеке метода; GC в этом не участвует. Заполняем префикс `"0x"` и две тетрады. Возврат `Encoding.ASCII.GetString(buffer[..pos])` — единственная аллокация, и она вынужденная: мы возвращаем строку, а строка — это объект в куче. Комментарием подчеркнут запрет возвращать сам `Span` над `stackalloc`: при выходе стек очищается, ссылка станет невалидной, и это типичная ошибка из урока.

В `CountZerosAsync(Memory<byte>)` параметр — `Memory<byte>`, а не `Span<byte>`, потому что метод `async` и содержит `await Task.Delay`. Если бы параметр был `Span<byte>`, компилятор выдал бы CS4007 (нельзя использовать `Span` в `async`). После `await` мы в синхронном сегменте получаем `ReadOnlySpan<byte> view = buffer.Span;` и считаем нули обычным `foreach`. Перегрузка `CountZerosAsync(byte[])` показывает неявное преобразование массива в `Memory<T>`, описанное в уроке: `Memory<byte>` — это «обёртка над окном», которую можно хранить и передавать через `await`.

#### Задания на углубление (бонус)

1. Реализуй `bool TryParseEndpoint(ReadOnlySpan<char> text, out ReadOnlySpan<char> host, out int port)`, который нарезает `host` как срез (без копии) и парсит порт. Подумай: что мешает вернуть `ReadOnlySpan<char>` через `out` из метода, если исходный текст — `string`, живущий в куче? Почему это безопасно, а возврат `Span` над `stackalloc` — нет?
2. Добавь `Span<byte> ReverseInPlace(Span<byte> buffer)`, который разворачивает буфер на месте, и продемонстрируй, что изменения видны в исходном массиве. Объясни, почему `Span` позволяет мутировать исходные данные без `ref`.
3. Сравни производительность `CountEndpoints` на срезах vs на `String.Split` через `BenchmarkDotNet`. Измерь аллокации ( `dotnet-counters` или `GC.GetAllocatedBytesForCurrentThread()`). Опиши, где выигрыш становится существенным (длина строки, число пар).
4. Реализуй async-конвейер: метод `async Task ProcessAllAsync(Memory<byte>[] buffers)`, который обрабатывает несколько буферов через `await`, и покажи, что `Memory<T>` корректно переживает несколько `await` в цикле, тогда как `Span` не пережил бы ни одного.

---

## Statement in English / Постановка на английском

#### Context & motivation

You are a developer on a small service that accepts "command lines" of a network protocol shaped like `host:port;host:port;...` and prepares binary packets for sending. Every operation sits on a hot path: the service processes thousands of lines per second, and each unnecessary string or byte-array allocation directly cuts throughput. The team has already noticed that a naive implementation built on `String.Split`, `Substring` and `new byte[N]` puts pressure on the garbage collector and increases pause times.

Lesson M06-L08 gives you the tool — `Span<T>` and `Memory<T>` — that lets you look at already-existing memory as if it were an array, without copying it. This is exactly what you need: parse a string as a sequence of slices, sum a window over an integer array without copying, format a byte into hex directly on the stack, and ship a buffer across the asynchronous `await` boundary. In this assignment you will combine all four scenarios into one project, so that you can see how "windows" replace copies, where the boundary between `Span` and `Memory` is enforced physically (compiler error CS4007), and why a `stackalloc`-backed `Span` must never escape a method. The goal is not to write "yet another parser" but to internalise the memory model at the level where the compiler becomes your tutor.

#### What to do step by step

1. Create a console project on .NET 8 with `dotnet new console -n SpanMemoryHomework -o SpanMemoryHomework` and enter it. Make sure `SpanMemoryHomework.csproj` contains `<TargetFramework>net8.0</TargetFramework>` and enables Nullable: `<Nullable>enable</Nullable>`. Use C# 12 (top-level statements are allowed in `Program.cs`, but move classes into separate files).

2. Create a class `ProtocolParser` with static methods, each illustrating a separate data source for `Span`:
   - `int SumWindow(int[] numbers, int start, int length)` — takes `numbers.AsSpan(start, length)` and returns the sum of the slice; demonstrate that `foreach` over `Span<int>` works just like over an array.
   - `int ParsePort(ReadOnlySpan<char> text)` — finds `':'` via `IndexOf`, slices the part after the colon via `Slice`, and parses with `int.Parse(portSpan)`. Return `-1` when there is no colon.
   - `int CountEndpoints(ReadOnlySpan<char> line)` — counts how many `host:port` pairs are separated by `';'`, using only slices (no `Split`, no `Substring`). Return the number of pairs.
   - `string FormatHex(byte value)` — uses `Span<byte> buffer = stackalloc byte[16];`, fills in `"0x"` and the two hex digits, and returns a string via `Encoding.ASCII.GetString(buffer[..pos])`. No `new byte[]` allowed.

3. Create a class `AsyncBufferOps` with a method `async Task<int> CountZerosAsync(Memory<byte> buffer)` that performs `await Task.Delay(10)` (simulated I/O) and then reads `ReadOnlySpan<byte> view = buffer.Span;`, counting zero bytes. Add also `async Task<int> CountZerosAsync(byte[] buffer)` that delegates to the first one, passing `buffer` as `Memory<byte>` (demonstrating the implicit array-to-`Memory<T>` conversion).

4. In `Program.cs` prepare test data: the array `{1,2,3,4,5,6,7,8}`, the string `"localhost:8080;api.example.com:443;cache:6379"`, the byte `0xAB`, and the buffer `byte[] {0, 5, 0, 7, 0}`. Print each method's result with `Console.WriteLine`. Expected outputs: window sum `[2..6]` = `3+4+5+6 = 18`, first port = `8080`, pair count = `3`, hex = `0xAB`, zero count = `3`.

5. In a separate file `InvalidExamples.cs` place commented-out "bad" code with explanations of why it does not compile: (a) a method `async Task<int> BadAsync(Span<byte> s)` that uses `s` after `await` (CS4007); (b) a class with a field `private Span<byte> _cache;` (a `ref struct` cannot be a field); (c) a method returning a `Span<byte>` over `stackalloc`. Uncomment each one in turn, confirm via `dotnet build` that the compiler complains exactly as described in the lesson, and comment it back.

6. Run `dotnet run` and verify that all five outputs match the expected ones. Run `dotnet build -c Release` — it must build without `Span`-related warnings. If a CA warning or analyzer warning about `Span` in `async` appears, investigate it.

7. Write unit tests in a project `SpanMemoryHomework.Tests` (`dotnet new xunit`), add a reference to the main project. Cover `SumWindow`, `ParsePort`, `CountEndpoints`, `FormatHex`, and `CountZerosAsync`. Use `Assert.Equal`. Test edge cases: empty string, string without a colon, window of length 0, buffer with no zeros.

#### Requirements

The solution must use exactly the constructs described in the lesson: `AsSpan`, `Slice` (or the `..` slice operator), `IndexOf`, `int.Parse(ReadOnlySpan<char>)`, `stackalloc`, `Memory<T>.Span`. It is forbidden to use `String.Split`, `Substring`, `new string(char[])` for intermediate values, and `new byte[N]` where `stackalloc` would do. All public methods that accept text must accept `ReadOnlySpan<char>` (not `string`), except entry points in `Program.cs` where a `string` is unavoidable — there call `.AsSpan()`.

The code must compile under .NET 8 / C# 12 with no errors and no `Span`-related warnings. `Span<T>`/`ReadOnlySpan<T>` must be passed by value (they are cheap window structs), not by `ref`, unless intentional performance demands it. Only `Memory<T>` is allowed in `async` methods; access to `.Span` must happen strictly inside synchronous segments after `await`. Project layout: library code in classes, `Program.cs` is demonstration only. Tests must run via `dotnet test` and pass.

File and class names must match the step list: `ProtocolParser.cs`, `AsyncBufferOps.cs`, `InvalidExamples.cs`, `Program.cs`. Do not leave commented-out working code in the final build, except the intentionally demonstrational `InvalidExamples.cs`. Every `stackalloc` must carry a comment explaining why the `Span` is not returned. Every transition `Span` → `Memory` (or back) must carry a comment about the `await` boundary.

#### Pitfalls

- `Span<T>` is a `ref struct`, so it **cannot** be stored in a class or struct field, cannot be boxed, cannot be captured in a lambda/closure, and cannot be used across `await`. If you try, you get a compile-time error such as CS4007, not a runtime crash. This is memory-safety protection, not an inconvenience.
- `stackalloc` places memory on the method's stack. You must never return a `Span` over it from a method: when the method returns, the stack is reclaimed and the reference becomes invalid. Return a **copy** (for example a string via `Encoding.ASCII.GetString`), exactly as `FormatHex` does.
- `int.Parse` in .NET has an overload accepting `ReadOnlySpan<char>` — this is the key reason slice-based parsing beats `Substring` + `int.Parse(string)`: there is no intermediate string allocation.
- Slices `span.Slice(start, length)` and the `span[start..end]` operator **do not copy** data. But watch the indices: an out-of-range value throws `ArgumentOutOfRangeException`. `AsSpan(start, length)` also validates bounds.
- `Memory<T>` is not "magically" safer across `await` than `Span`; it simply can live on the heap. You obtain a `Span` from `Memory` via the `.Span` property, but only synchronously. Passing `Memory` into another `async` method is fine; trying to hold `.Span` across `await` brings back CS4007.
- Do not confuse `ReadOnlySpan<T>` with `Span<T>`: a string yields only `ReadOnlySpan<char>` (strings are immutable). An array yields both, depending on the `AsSpan` overload. Prefer `ReadOnly` by default — it expresses intent and is safer.
- Boxing a `Span` through generic methods (for example `List<object>` or `IEnumerable<T>`) is impossible — the compiler forbids it. Do not try to put a `Span` into a collection.

#### Acceptance criteria

- [ ] The `SpanMemoryHomework` project builds under .NET 8 / C# 12 with no errors and no `Span`-related warnings.
- [ ] `ProtocolParser.SumWindow` uses `AsSpan(start, length)` and `foreach` over `Span<int>`, returning the window sum.
- [ ] `ProtocolParser.ParsePort` accepts `ReadOnlySpan<char>`, uses `IndexOf(':')`, `Slice`, and `int.Parse(ReadOnlySpan<char>)`, with no `Substring`.
- [ ] `ProtocolParser.CountEndpoints` counts pairs separated by `';'` using slices only, with no `Split`.
- [ ] `ProtocolParser.FormatHex` uses `stackalloc byte[16]`, never returns the `Span`, and returns a string.
- [ ] `AsyncBufferOps.CountZerosAsync(Memory<byte>)` contains an `await` and accesses `buffer.Span` strictly after `await`, in a synchronous segment.
- [ ] The `CountZerosAsync(byte[])` overload delegates to the `Memory<byte>` version, demonstrating the implicit array conversion.
- [ ] `InvalidExamples.cs` contains three commented-out "bad" examples with explanations; uncommenting each one produces the matching compile error.
- [ ] `Program.cs` prints exactly: `18`, `8080`, `3`, `0xAB`, `3`.
- [ ] `dotnet test` passes; there are tests for edge cases (empty string, no colon, window of length 0, buffer with no zeros).
- [ ] All public text methods accept `ReadOnlySpan<char>`, not `string` (except entry points in `Program.cs`).
- [ ] No `String.Split`, `Substring`, `new byte[N]` where `stackalloc` is needed, or `new string(char[])` for intermediate values.
- [ ] Every `stackalloc` carries a comment about not returning the `Span`.
- [ ] Every `Span` ↔ `Memory` transition carries a comment about the `await` boundary.
- [ ] `dotnet build -c Release` emits no `Span`/`Memory`-related warnings.

#### Hints (no direct answer)

- Recall that `IndexOf` on `ReadOnlySpan<char>` returns the index of the first occurrence of a character, or `-1`. That is enough to find both `':'` and `';'`.
- To count pairs separated by `';'` you do not need `Split`: walk the string with slices, cutting from the current position to the next `';'`.
- For hex: the high nibble is `value >> 4`, the low nibble is `value & 0x0F`. The character for `0..9` is `'0' + n`, for `10..15` it is `'A' + n - 10`.
- `Memory<byte>` is obtained from `byte[]` implicitly; the reverse is `memory.Span`. Think of `Memory` as "a box that holds a window, but the window itself can only be taken out synchronously".
- If the compiler complains CS4007 in an `async` method, that is the signal that you are trying to hold a `Span` across `await`. Change the parameter to `Memory<T>`.
- Never try to return a `Span<byte>` from a method that used `stackalloc`: return a `string` or a `byte[]` (a copy), as `FormatHex` does.

#### Reference solution walk-through

```csharp
// C# 12 / .NET 8 — Reference solution for Homework M06-L08
using System;
using System.Text;
using System.Threading.Tasks;

// 1) Synchronous slice-based parser
public static class ProtocolParser
{
    // Window over an array, no copy
    public static int SumWindow(int[] numbers, int start, int length)
    {
        Span<int> window = numbers.AsSpan(start, length); // a window, not a copy
        int sum = 0;
        foreach (var n in window) sum += n;               // iterate like an array
        return sum;
    }

    // Parse port from "host:port" without Substring
    public static int ParsePort(ReadOnlySpan<char> text)
    {
        int colon = text.IndexOf(':');                    // index of ':'
        if (colon < 0) return -1;                         // no colon
        ReadOnlySpan<char> portSpan = text[(colon + 1)..]; // slice, no allocation
        return int.Parse(portSpan);                       // Span overload
    }

    // Count "host:port" pairs separated by ';' using slices only
    public static int CountEndpoints(ReadOnlySpan<char> line)
    {
        int count = 0;
        while (!line.IsEmpty)
        {
            int semi = line.IndexOf(';');                 // find separator
            ReadOnlySpan<char> pair = semi < 0 ? line : line[..semi];
            if (pair.IndexOf(':') >= 0) count++;          // a valid pair
            line = semi < 0 ? default : line[(semi + 1)..]; // advance the window
        }
        return count;
    }

    // Hex formatting via stackalloc
    public static string FormatHex(byte value)
    {
        Span<byte> buffer = stackalloc byte[16];          // stack buffer, no GC
        buffer[0] = (byte)'0';
        buffer[1] = (byte)'x';
        int pos = 2;
        buffer[pos++] = HexChar(value >> 4);              // high nibble
        buffer[pos++] = HexChar(value & 0x0F);            // low nibble
        return Encoding.ASCII.GetString(buffer[..pos]);   // copy only on return
        // IMPORTANT: never return the stackalloc Span — the stack will be reclaimed
    }

    private static byte HexChar(int n) => (byte)(n < 10 ? '0' + n : 'A' + n - 10);
}

// 2) Async buffer processing via Memory<T>
public static class AsyncBufferOps
{
    // Memory<T> survives await
    public static async Task<int> CountZerosAsync(Memory<byte> buffer)
    {
        await Task.Delay(10);                             // simulated I/O — Span cannot be used here
        ReadOnlySpan<byte> view = buffer.Span;            // synchronous access to the window
        int zeros = 0;
        foreach (var b in view) if (b == 0) zeros++;
        return zeros;
    }

    // Array overload: implicit byte[] -> Memory<byte>
    public static Task<int> CountZerosAsync(byte[] buffer) => CountZerosAsync((Memory<byte>)buffer);
}

// 3) Demonstration
public static class Program
{
    public static void Main()
    {
        int[] data = { 1, 2, 3, 4, 5, 6, 7, 8 };
        Console.WriteLine(ProtocolParser.SumWindow(data, 2, 4));                 // 18

        Console.WriteLine(ProtocolParser.ParsePort("localhost:8080".AsSpan()));  // 8080

        ReadOnlySpan<char> line = "localhost:8080;api.example.com:443;cache:6379".AsSpan();
        Console.WriteLine(ProtocolParser.CountEndpoints(line));                  // 3

        Console.WriteLine(ProtocolParser.FormatHex(0xAB));                       // 0xAB

        byte[] buf = { 0, 5, 0, 7, 0 };
        Console.WriteLine(AsyncBufferOps.CountZerosAsync(buf).Result);           // 3
    }
}
```

Line-by-line walk-through. In `SumWindow`, `numbers.AsSpan(start, length)` builds a `Span<int>` — a struct of three fields (reference, index, length) that "looks at" a region of the array without copying. `foreach` over `Span<int>` compiles to efficient indexed access, equivalent to an array loop but with no allocations. This illustrates the lesson's thesis: a "window" replaces a copy.

In `ParsePort`, `text.IndexOf(':')` is a `ReadOnlySpan<char>` method that returns an index with no allocation. The slice `text[(colon + 1)..]` uses the C# 8+ range operator, which for `Span` compiles to `Slice` with no copy. The key moment is `int.Parse(portSpan)`: the lesson stresses that `int.Parse` has an overload for `ReadOnlySpan<char>`, so we avoid the intermediate string that `Substring` would have created. This is precisely the "high-performance parsing" of the theory.

`CountEndpoints` extends the idea: instead of `Split(';')` (which allocates a string array) we manually advance the window `line` via `line[(semi + 1)..]`. Each iteration is a new slice over the same memory. The condition `pair.IndexOf(':') >= 0` filters invalid pairs. The method shows that slices can be carved recursively, producing zero allocations.

`FormatHex` is the only place with `stackalloc`. The line `Span<byte> buffer = stackalloc byte[16];` places 16 bytes on the method's stack; the GC is not involved. We fill the `"0x"` prefix and the two nibbles. The return `Encoding.ASCII.GetString(buffer[..pos])` is the only allocation, and it is forced: we return a string, and a string is a heap object. The comment stresses the ban on returning the `stackalloc` `Span` itself: on return the stack is reclaimed, the reference becomes invalid, and this is a classic lesson mistake.

In `CountZerosAsync(Memory<byte>)` the parameter is `Memory<byte>`, not `Span<byte>`, because the method is `async` and contains `await Task.Delay`. Had the parameter been `Span<byte>`, the compiler would emit CS4007 (a `Span` cannot be used in `async`). After `await`, inside a synchronous segment, we obtain `ReadOnlySpan<byte> view = buffer.Span;` and count zeros with a plain `foreach`. The `CountZerosAsync(byte[])` overload shows the implicit array-to-`Memory<T>` conversion described in the lesson: `Memory<byte>` is a "wrapper around a window" that can be stored and shipped across `await`.

#### Going deeper (bonus)

1. Implement `bool TryParseEndpoint(ReadOnlySpan<char> text, out ReadOnlySpan<char> host, out int port)` that slices `host` (no copy) and parses the port. Think: what stops you from returning a `ReadOnlySpan<char>` via `out` when the source text is a heap-resident `string`? Why is this safe while returning a `Span` over `stackalloc` is not?
2. Add `Span<byte> ReverseInPlace(Span<byte> buffer)` that reverses the buffer in place, and demonstrate that the changes are visible in the source array. Explain why `Span` lets you mutate the underlying data without `ref`.
3. Compare the performance of slice-based `CountEndpoints` against a `String.Split` version using `BenchmarkDotNet`. Measure allocations (with `dotnet-counters` or `GC.GetAllocatedBytesForCurrentThread()`). Describe where the win becomes meaningful (string length, number of pairs).
4. Implement an async pipeline: a method `async Task ProcessAllAsync(Memory<byte>[] buffers)` that processes several buffers through `await`, and show that `Memory<T>` correctly survives multiple `await`s in a loop, whereas `Span` would not survive even one.

---

#### Чек-лист сдачи / Submission checklist

- [ ] Проект собирается под .NET 8 / C# 12 без ошибок и предупреждений про `Span`.
- [ ] Реализованы `ProtocolParser`, `AsyncBufferOps`, `InvalidExamples`, `Program`.
- [ ] `dotnet run` выводит `18`, `8080`, `3`, `0xAB`, `3`.
- [ ] Тесты (`dotnet test`) проходят, включая краевые случаи.
- [ ] Никаких `Split`/`Substring`/`new byte[N]` вместо `stackalloc`.
- [ ] Каждый `stackalloc` и каждый `Span`↔`Memory` переход прокомментированы.
- [ ] Project builds under .NET 8 / C# 12 with no `Span`-related errors or warnings.
- [ ] `ProtocolParser`, `AsyncBufferOps`, `InvalidExamples`, `Program` are implemented.
- [ ] `dotnet run` prints `18`, `8080`, `3`, `0xAB`, `3`.
- [ ] Tests (`dotnet test`) pass, including edge cases.
- [ ] No `Split`/`Substring`/`new byte[N]` where `stackalloc` is expected.
- [ ] Every `stackalloc` and every `Span`↔`Memory` transition is commented.

#### Ресурсы / Resources

- [Microsoft Learn — Span<T>](https://learn.microsoft.com/dotnet/api/system.span-1)
- [Microsoft Learn — Memory<T>](https://learn.microsoft.com/dotnet/api/system.memory-1)
- [Microsoft Learn — ReadOnlySpan<T>](https://learn.microsoft.com/dotnet/api/system.readonlyspan-1)
- [Span<T> and Memory<T> usage guidelines](https://learn.microsoft.com/dotnet/standard/memory-and-spans/)
- [Stephen Toub — Span](https://learn.microsoft.com/archive/msdn-magazine/2018/january/csharp-all-about-span-exploring-a-new-net-mainstay)
