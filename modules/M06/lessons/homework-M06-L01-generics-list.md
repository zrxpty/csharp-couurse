---
[← К уроку M06-L01](lesson-M06-L01-generics-list.md) | [⬆ К модулю M06](../README.md) | [Следующее ДЗ →](homework-M06-L02-generic-classes-methods.md)
---

### Домашнее задание M06-L01: Зачем дженерики, List<T> / Homework M06-L01: Why generics, List<T>

**Урок / Lesson:** M06-L01
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 3/5
**Цель / Goal:** (RU) На практике ощутить разницу между `ArrayList` (boxing, потеря типобезопасности) и `List<T>`, освоить базовые операции `List<T>`, научиться управлять `Count`/`Capacity` и проектировать публичный API на `IReadOnlyList<T>`. (EN) Feel the difference between `ArrayList` (boxing, loss of type safety) and `List<T>` in practice, master core `List<T>` operations, learn to manage `Count`/`Capacity`, and design a public API on top of `IReadOnlyList<T>`.

#### Связь с уроком / Connection to the lesson
(RU) Урок объясняет, почему дженерики заменили `ArrayList`: они убирают boxing, восстанавливают типобезопасность и делают код читаемым. Это ДЗ заставляет пройти весь путь — от демо `ArrayList` с runtime-ошибкой до аккуратного анализатора лога датчиков на `List<T>` с защищённым `IReadOnlyList<T>`-интерфейсом. Все ключевые операции урока (`Add`, `AddRange`, `Remove`, `RemoveAt`, `TrimExcess`, `IndexOf`, `Contains`) и частые ошибки (мутирование в `foreach`, путаница `Count`/`Capacity`, утечка изменяемой ссылки) здесь отрабатываются на реальном коде.
(EN) The lesson explains why generics replaced `ArrayList`: they remove boxing, restore type safety, and make code readable. This homework walks the whole path — from an `ArrayList` demo with a runtime error to a tidy sensor-log analyzer built on `List<T>` with a protected `IReadOnlyList<T>` surface. Every core operation from the lesson (`Add`, `AddRange`, `Remove`, `RemoveAt`, `TrimExcess`, `IndexOf`, `Contains`) and every common pitfall (mutating inside `foreach`, confusing `Count`/`Capacity`, leaking a mutable reference) is exercised on real code.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация

Вы присоединяетесь к команде, которая пишет мини-сервис мониторинга температуры теплицы. Сенсор раз в минуту шлёт вещественное число — градусы Цельсия. Окно наблюдения — сутки, то есть до 1440 значений. Исторически прототип хранил значения в `System.Collections.ArrayList`, потому что «так быстрее было набросать», и это решение уже начало мешать: при подсчёте среднего по суткам периодически вылетает `InvalidCastException`, потому что в тот же список случайно попали строковые сообщения вида `"sensor reboot"`. Кроме того, профилирование показывает необъяснимый рост аллокаций — тысячи мелких объектов в куче, которые приходится убирать сборщику мусора, хотя значения — простые `double`.

Ваша задача — переписать хранилище показаний на дженерики, продемонстрировать (на отдельном демо-модуле), почему старый подход ломался, и реализовать класс `SensorLog`, который накапливает показания, считает базовую статистику (минимум, максимум, среднее), умеет фильтровать выбросы (значения за пределами правдоподобного диапазона) и удалять дубликаты подряд идущих одинаковых значений. Публичный API должен отдавать данные только на чтение, чтобы внешние потребители не могли случайно изменить внутренний список. Эта задача — ровно то, ради чего в C# появились дженерики: типобезопасность без накладных расходов на boxing, читаемый код без приведений и защищённое состояние объекта.

#### Что нужно сделать (пошагово)

1. Создайте консольный проект .NET 8 с языковой версией C# 12:
   ```
   dotnet new console -n GreenhouseMonitor -o GreenhouseMonitor -f net8.0
   cd GreenhouseMonitor
   dotnet new langversion --set 12.0
   ```
   Убедитесь, что в `GreenhouseMonitor.csproj` указаны `<TargetFramework>net8.0</TargetFramework>` и `<LangVersion>12.0</LangVersion>` (или `latest`).

2. В файле `Program.cs` организуйте код через top-level statements, но саму логику вынесите в отдельные файлы: `BoxingDemo.cs` (демонстрация проблемы), `SensorLog.cs` (основной класс), `Reading.cs` (record для одного показания). Включите `Nullable` (по умолчанию для новых проектов).

3. В `BoxingDemo.cs` создайте метод `public static void Run()`, который: создаёт `ArrayList`, кладёт в него три `int` (1, 2, 3) и строку `"oops"`; пытается пройти `foreach (int n in boxed)` и поймать `InvalidCastException` в `try/catch`, выводя понятное сообщение; затем то же самое повторяет на `List<int>` и показывает, что ошибка невозможна — компилятор не даст положить строку. В комментариях (RU+EN) отметьте, где происходит boxing и unboxing.

4. Определите `Reading` как `readonly record struct` с полями `DateTime Offset` и `double Celsius`. Record-struct даёт структурное равенство, что пригодится для поиска дубликатов через `IndexOf`/`Remove`.

5. Реализуйте `SensorLog`:
   - приватное поле `private readonly List<Reading> _readings;`
   - конструктор `public SensorLog(int expectedCount = 1440)`, который передаёт `expectedCount` как `capacity` в `List<Reading>(capacity)`;
   - `public void Add(Reading r)` — добавить в конец;
   - `public void AddRange(IEnumerable<Reading> items)` — добавить пачку; используйте коллекционное выражение в демо-вызове, например `log.AddRange([r1, r2, r3]);`;
   - `public IReadOnlyList<Reading> Readings => _readings;` — публичное свойство только для чтения;
   - `public int Count => _readings.Count;`
   - `public Reading this[int index] => _readings[index];` — индексатор только на чтение (без setter);
   - `public (double Min, double Max, double Avg) Statistics()` — если список пуст, кидать `InvalidOperationException` с понятным сообщением; иначе вернуть кортеж;
   - `public int RemoveOutliers(double minValid, double maxValid)` — удалить все показания вне диапазона за один проход, **не мутируя список внутри `foreach`**; используйте `RemoveAll(predicate)` и верните число удалённых;
   - `public int CollapseDuplicates()` — удалить подряд идущие дубликаты (оставить первое из серии), тоже через `RemoveAll` с индексной проверкой или через построение нового списка; вернуть число удалённых;
   - `public bool RemoveAt(int index)` — безопасный RemoveAt: проверить `index >= 0 && index < _readings.Count`, иначе вернуть `false` (это отрабатывает урок «всегда проверяй индекс»);
   - `public void Trim()` — обёртка над `TrimExcess()`.

6. В `Program.cs` (top-level) соберите демо-сценарий:
   - создайте `var log = new SensorLog(expectedCount: 10);`
   - добавьте показания через `Add` и `AddRange` с коллекционным выражением;
   - намеренно добавьте пару выбросов (например, `-999.0` и `250.0`) и подряд идущие дубликаты;
   - выведите `Count` и `Capacity` до и после `Trim()`;
   - вызовите `RemoveOutliers(0, 100)`, затем `CollapseDuplicates()`;
   - распечатайте статистику;
   - продемонстрируйте, что внешняя попытка `log.Readings.Add(...)` **не компилируется** — закомментируйте строку с комментарием, почему она запрещена.

7. Соберите и запустите: `dotnet build`, затем `dotnet run`. Проверьте ожидаемый вывод: до `RemoveOutliers` `Count` больше, после — меньше; `Capacity` уменьшается после `Trim()`; статистика считает корректно.

8. Добавьте unit-тесты (опционально, через `dotnet new xunit`) на: пустой лог кидает `InvalidOperationException`; `RemoveOutliers` возвращает верное число; `CollapseDuplicates` не трогает неповторяющиеся; `RemoveAt(-1)` возвращает `false`.

#### Требования к решению

- Код должен компилироваться под C# 12 / .NET 8 без предупреждений, в том числе без `nullable`-предупреждений.
- Использовать `List<T>` и `IReadOnlyList<T>` из `System.Collections.Generic`; `ArrayList` — только в `BoxingDemo` для демонстрации проблемы.
- Задавать `capacity` в конструкторе `SensorLog` и вызывать `TrimExcess()` в `Trim()`.
- Запрещено мутировать `_readings` внутри `foreach` по нему; для удаления по условию применять `RemoveAll`.
- Публичные свойства и возвращаемые значения, отдающие коллекцию, — только `IReadOnlyList<Reading>` (или копия). Ни одна публичная сигнатура не возвращает сам `List<Reading>`.
- Индексатор на чтение не имеет `set`; `RemoveAt` публичный, но с защитой индекса.
- Код читаемый, с RU+EN комментариями в ключевых местах (boxing, capacity, read-only контракт).

#### Тонкости и подводные камни

- **Boxing прячется.** `ArrayList.Add(int)` упаковывает значение в `object` — это не видно в коде, но видно в аллокациях. В `BoxingDemo` явно отметьте строку с boxing и строку с unboxing в `foreach (int n in ...)`. Помните, что даже `boxed[0]` без приведения вернёт `object`, а не `int`.
- **`Count` ≠ `Capacity`.** В выводе `SensorLog` вы увидите, что `Capacity` растёт скачками (удвоение), а `Count` — по одному. Не ориентируйте логику на `Capacity` и не передавайте её как «размер» наружу.
- **Мутирование в `foreach` = `InvalidOperationException`.** Если внутри цикла по `_readings` вызвать `_readings.Remove(...)`, рантайм бросит исключение «Collection was modified». Решение — `RemoveAll`, который проходит список безопасно, либо сбор индексов и удаление с конца.
- **Утечка изменяемой ссылки.** Если бы свойство было `public List<Reading> Readings => _readings;`, любой внешний код мог бы вызвать `log.Readings.Clear()`. `IReadOnlyList<T>` закрывает `Add`/`Remove` на уровне компилятора — это и есть «договор о ненарушении» из урока.
- **`IndexOf` и `-1`.** Перед `RemoveAt` всегда проверяйте, что индекс найден. В `CollapseDuplicates` сравнивайте соседей через индексатор, аккуратно обходя границу `i == 0`.
- **`Remove(item)` и равенство.** Поскольку `Reading` — `record struct`, равенство структурное, и `Remove`/`IndexOf` работают по значениям, а не по ссылке. Если бы это был класс без переопределённого `Equals`, удаление шло бы по ссылке.
- **Capacity и `AddRange`.** Если `AddRange` превышает текущую `Capacity`, происходит одна реаллокация, а не серия — это эффективнее, чем вызывать `Add` в цикле.

#### Критерии приёмки

- [ ] Проект собирается под .NET 8 / C# 12 без ошибок и предупреждений.
- [ ] `BoxingDemo.Run()` падает в `InvalidCastException` на `ArrayList` с `string` и корректно отлавливает исключение.
- [ ] `BoxingDemo.Run()` демонстрирует, что `List<int>` не даёт положить `string` (ошибка на этапе компиляции).
- [ ] В комментариях отмечены строки с boxing и unboxing.
- [ ] `SensorLog` хранит данные в `List<Reading>` с заданной `capacity`.
- [ ] Публичный `Readings` имеет тип `IReadOnlyList<Reading>`.
- [ ] `Count` и индексатор только на чтение реализованы.
- [ ] `Statistics()` кидает `InvalidOperationException` для пустого лога.
- [ ] `RemoveOutliers` использует `RemoveAll` и возвращает число удалённых.
- [ ] `CollapseDuplicates` удаляет только подряд идущие дубликаты.
- [ ] `RemoveAt` безопасно обрабатывает `-1` и индекс за границей.
- [ ] `Trim()` вызывает `TrimExcess()`.
- [ ] В выводе видна разница `Count`/`Capacity` до и после `Trim()`.
- [ ] Строка `log.Readings.Add(...)` закомментирована с объяснением, почему она не компилируется.
- [ ] Нет мутаций списка внутри `foreach`.

#### Подсказки (без прямого ответа)

- Для `RemoveOutliers` предикат `RemoveAll` — это лямбда, возвращающая `true` для элементов, которые надо **удалить** (то есть тех, что вне диапазона). Не перепутайте направление.
- В `CollapseDuplicates` сравнивайте `i > 0 && _readings[i].Equals(_readings[i-1])` в предикате `RemoveAll`.
- `Statistics` можно посчитать за один проход через обычный `for` по `Count` и индексатору, без LINQ — чтобы не плодить лишних аллокаций.
- `IReadOnlyList<Reading>` неявно приводится из `List<Reading>` — отдельный `ToArray` не нужен.
- Для `record struct` равенство уже переопределено — вручную реализовывать `IEquatable<Reading>` не требуется.

#### Эталонное решение (разбор)

```csharp
// C# 12 / .NET 8
// BoxingDemo.cs — демонстрация проблемы с ArrayList
// BoxingDemo.cs — demo of the ArrayList problem

using System.Collections;

namespace GreenhouseMonitor;

public static class BoxingDemo
{
    public static void Run()
    {
        // --- Старый подход: ArrayList на базе object ---
        // --- Old approach: ArrayList backed by object ---
        ArrayList boxed = new();
        boxed.Add(1);          // boxing: int → object  /  boxing: int → object
        boxed.Add(2);
        boxed.Add(3);
        boxed.Add("oops");     // компилятор не возражает / compiler does not complain

        try
        {
            // unboxing в каждом приведении (int)  /  unboxing on each (int) cast
            int sum = 0;
            foreach (int n in boxed)   // InvalidCastException на "oops"
                sum += n;              // / InvalidCastException on "oops"
            Console.WriteLine($"sum = {sum}");
        }
        catch (InvalidCastException ex)
        {
            Console.WriteLine($"ArrayList сломался: {ex.Message}");
            // / ArrayList broke: {ex.Message}
        }

        // --- Современный подход: List<int> ---
        // --- Modern approach: List<int> ---
        List<int> numbers = new(capacity: 4) { 1, 2, 3 };
        // numbers.Add("oops");  // ошибка компиляции: типобезопасность
        // / compile-time error: type safety
        Console.WriteLine($"List<int> sum = {numbers.Sum()}");
    }
}
```

```csharp
// Reading.cs — одно показание датчика
// Reading.cs — a single sensor reading
namespace GreenhouseMonitor;

public readonly record struct Reading(DateTime Offset, double Celsius);
```

```csharp
// SensorLog.cs — основное хранилище на List<T> + IReadOnlyList<T>
// SensorLog.cs — main storage on List<T> + IReadOnlyList<T>
namespace GreenhouseMonitor;

public sealed class SensorLog
{
    private readonly List<Reading> _readings;

    // capacity задаём заранее, чтобы избежать лишних реаллокаций
    // set capacity up front to avoid extra reallocations
    public SensorLog(int expectedCount = 1440)
        => _readings = new List<Reading>(capacity: expectedCount);

    public void Add(Reading r) => _readings.Add(r);

    public void AddRange(IEnumerable<Reading> items) => _readings.AddRange(items);

    // Публичный API — только чтение: внешняя мутация запрещена на уровне компилятора.
    // Public API — read only: external mutation is forbidden by the compiler.
    public IReadOnlyList<Reading> Readings => _readings;

    public int Count => _readings.Count;

    // Индексатор без setter — наружу только чтение.
    // Indexer without setter — read-only to the outside.
    public Reading this[int index] => _readings[index];

    public (double Min, double Max, double Avg) Statistics()
    {
        if (_readings.Count == 0)
            throw new InvalidOperationException("Лог пуст / log is empty");

        double min = double.PositiveInfinity;
        double max = double.NegativeInfinity;
        double sum = 0;
        for (int i = 0; i < _readings.Count; i++)
        {
            double v = _readings[i].Celsius;
            if (v < min) min = v;
            if (v > max) max = v;
            sum += v;
        }
        return (min, max, sum / _readings.Count);
    }

    // Удаляем выбросы одним проходом через RemoveAll — без мутации в foreach.
    // Remove outliers in one pass via RemoveAll — no mutation inside foreach.
    public int RemoveOutliers(double minValid, double maxValid)
        => _readings.RemoveAll(r => r.Celsius < minValid || r.Celsius > maxValid);

    // Удаляем подряд идущие дубликаты: убираем элемент, равный предыдущему.
    // Remove consecutive duplicates: drop an element equal to the previous one.
    public int CollapseDuplicates()
        => _readings.RemoveAll(r =>
        {
            int i = _readings.IndexOf(r);
            return i > 0 && _readings[i - 1].Equals(r);
        });

    // Безопасный RemoveAt — всегда проверяем индекс.
    // Safe RemoveAt — always validate the index.
    public bool RemoveAt(int index)
    {
        if (index < 0 || index >= _readings.Count) return false;
        _readings.RemoveAt(index);
        return true;
    }

    public void Trim() => _readings.TrimExcess();
}
```

```csharp
// Program.cs — top-level entry
using GreenhouseMonitor;

BoxingDemo.Run();
Console.WriteLine("---");

var log = new SensorLog(expectedCount: 10);

var t0 = DateTime.Today;
log.Add(new Reading(t0.AddMinutes(1), 20.5));
log.AddRange([
    new Reading(t0.AddMinutes(2), 20.5),   // дубликат подряд / consecutive duplicate
    new Reading(t0.AddMinutes(3), 21.0),
    new Reading(t0.AddMinutes(4), -999.0), // выброс / outlier
    new Reading(t0.AddMinutes(5), 250.0),  // выброс / outlier
    new Reading(t0.AddMinutes(6), 22.0),
]);

Console.WriteLine($"До фильтра: Count={log.Count}, Capacity=? (через поле скрыто)");
int removedOut = log.RemoveOutliers(0, 100);
Console.WriteLine($"Удалено выбросов: {removedOut}");
int removedDup = log.CollapseDuplicates();
Console.WriteLine($"Удалено дубликатов подряд: {removedDup}");

var (min, max, avg) = log.Statistics();
Console.WriteLine($"Min={min}, Max={max}, Avg={avg:F2}");

Console.WriteLine($"log[0] = {log[0]}");
Console.WriteLine($"RemoveAt(-1) → {log.RemoveAt(-1)}"); // false

// log.Readings.Add(...);  // не компилируется: IReadOnlyList<T> не даёт Add
// / won't compile: IReadOnlyList<T> does not expose Add

log.Trim();
Console.WriteLine("После Trim() лишняя Capacity освобождена.");
```

**Разбор по строкам.** `BoxingDemo` воспроизводит ровно ту ошибку из урока, ради которой появились дженерики: `ArrayList` принимает `int` и `string` без жалоб компилятора, а `foreach (int n in boxed)` падает в рантайме с `InvalidCastException`, потому что на строке `"oops"` происходит неявный unboxing-каст. В `List<int>` тот же сценарий невозможен — попытка `numbers.Add("oops")` отклоняется компилятором: это и есть типобезопасность, обещанная дженериками. Поле `_readings` объявлено `private readonly List<Reading>`, а конструктор `SensorLog` передаёт `expectedCount` как `capacity` — это best practice из урока: если размер известен, реаллокаций и разрастания `Capacity` не будет. Свойство `Readings` возвращает `_readings` как `IReadOnlyList<Reading>`: неявное приведение работает, потому что `List<T>` реализует этот интерфейс; снаружи видны только `Count` и индексатор, `Add`/`Remove` скрыты — внешний код физически не может вызвать `log.Readings.Clear()`. Индексатор объявлен только с `get` (без `set`), что закрывает запись через `log[i] = ...`. `Statistics` проверяет пустоту и кидает `InvalidOperationException` — честная обработка граничного случая вместо `NaN`-ов. `RemoveOutliers` использует `RemoveAll` с предикатом «значение вне диапазона», что решает сразу две проблемы урока: нет мутации внутри `foreach` (которая дала бы `InvalidOperationException`) и удаление за один проход, без дорогостоящего `Remove(item)` в цикле. `CollapseDuplicates` через `IndexOf` находит позицию элемента и сравнивает с предыдущим — здесь важно, что `Reading` это `record struct` со структурным равенством, поэтому `IndexOf` находит именно то значение, а не какую-то другую ссылку. `RemoveAt` публичный, но с проверкой границ и возвратом `bool` — это прямой ответ на частую ошибку из урока «игнорирование `-1` от `IndexOf` и индексов за границей». `Trim()` — тонкая обёртка над `TrimExcess()`, освобождающая лишнюю `Capacity` после заполнения. В `Program.cs` используется коллекционное выражение `[...]` для `AddRange` — современная синтаксическая возможность C# 12, рекомендованная вместо `new List<T>{...}` или `Add` в цикле. Закомментированная строка `log.Readings.Add(...)` — наглядное доказательство, что контракт `IReadOnlyList<T>` работает на уровне компилятора, а не только на честном слове.

#### Задания на углубление (бонус)

1. Добавьте метод `public IReadOnlyList<Reading> Snapshot()` через `ToArray()`, который возвращает копию данных, и сравните по производительности с возвратом `IReadOnlyList<T>` без копии (используйте `BenchmarkDotNet`).
2. Реализуйте `CollapseDuplicates` без `RemoveAll`, построив новый `List<Reading>` и заменив `_readings` через `Clear()` + `AddRange` — объясните, почему так делать менее эффективно, и в чём подвох с `readonly` полем.
3. Расширьте `SensorLog` до обобщённого `SeriesLog<T>` (заготовка для следующего урока M06-L02), где `T : struct` — параметр типа, а логика `Statistics` делегируется внешнему `IStatCalculator<T>`.
4. Покажите через `dotnet-counters`, что при использовании `ArrayList<double>` объём аллокаций в `Gen 0` заметно выше, чем для `List<double>` на 10000 элементов, и объясните связь с boxing.

---

## Statement in English / Постановка на английском

#### Context & motivation

You are joining a team that is writing a small greenhouse temperature-monitoring service. A sensor sends one floating-point number every minute — degrees Celsius. The observation window is a full day, which means up to 1440 values. Historically the prototype stored readings in a `System.Collections.ArrayList`, because it was "quicker to sketch", and that decision has already started to bite: when the daily average is computed, an `InvalidCastException` occasionally flies out, because string messages such as `"sensor reboot"` slipped into the same list by accident. On top of that, profiling shows unexplained allocation growth — thousands of small heap objects that the garbage collector has to chase, even though the values are plain `double`.

Your job is to rewrite the reading store on generics, demonstrate (in a separate demo module) why the old approach broke, and implement a `SensorLog` class that accumulates readings, computes basic statistics (minimum, maximum, average), can filter outliers (values outside a plausible range), and removes consecutive duplicate readings. The public API must hand data out read-only, so external consumers cannot accidentally mutate the internal list. This task is exactly what generics were invented for in C#: type safety without boxing overhead, readable code without casts, and a protected object state.

#### What to do step by step

1. Create a .NET 8 console project with the C# 12 language version:
   ```
   dotnet new console -n GreenhouseMonitor -o GreenhouseMonitor -f net8.0
   cd GreenhouseMonitor
   dotnet new langversion --set 12.0
   ```
   Verify that `GreenhouseMonitor.csproj` contains `<TargetFramework>net8.0</TargetFramework>` and `<LangVersion>12.0</LangVersion>` (or `latest`).

2. In `Program.cs` use top-level statements, but move the logic into separate files: `BoxingDemo.cs` (the problem demo), `SensorLog.cs` (the main class), `Reading.cs` (a record for a single reading). Keep `Nullable` enabled (the default for new projects).

3. In `BoxingDemo.cs` create a method `public static void Run()` that: creates an `ArrayList`, adds three `int` values (1, 2, 3) and a string `"oops"`; tries to iterate with `foreach (int n in boxed)` and catches `InvalidCastException` in a `try/catch`, printing a clear message; then repeats the same with `List<int>` and shows the error is impossible — the compiler will not let you add a string. In comments (EN+RU) mark exactly where boxing and unboxing happen.

4. Define `Reading` as a `readonly record struct` with fields `DateTime Offset` and `double Celsius`. A record struct gives structural equality, which matters for duplicate search via `IndexOf`/`Remove`.

5. Implement `SensorLog`:
   - private field `private readonly List<Reading> _readings;`;
   - constructor `public SensorLog(int expectedCount = 1440)` that passes `expectedCount` as `capacity` to `List<Reading>(capacity)`;
   - `public void Add(Reading r)` — append;
   - `public void AddRange(IEnumerable<Reading> items)` — append a batch; in the demo call use a collection expression, e.g. `log.AddRange([r1, r2, r3]);`;
   - `public IReadOnlyList<Reading> Readings => _readings;` — a read-only public property;
   - `public int Count => _readings.Count;`;
   - `public Reading this[int index] => _readings[index];` — a read-only indexer (no setter);
   - `public (double Min, double Max, double Avg) Statistics()` — throw `InvalidOperationException` with a clear message when the list is empty; otherwise return a tuple;
   - `public int RemoveOutliers(double minValid, double maxValid)` — remove every reading outside the range in a single pass, **without mutating the list inside `foreach`**; use `RemoveAll(predicate)` and return the number removed;
   - `public int CollapseDuplicates()` — remove consecutive duplicates (keep the first of each run), also via `RemoveAll` with an index-aware check or by building a new list; return the number removed;
   - `public bool RemoveAt(int index)` — a safe RemoveAt: check `index >= 0 && index < _readings.Count`, otherwise return `false` (this exercises the lesson's "always validate the index" rule);
   - `public void Trim()` — a wrapper over `TrimExcess()`.

6. In `Program.cs` (top-level) build a demo scenario:
   - create `var log = new SensorLog(expectedCount: 10);`;
   - add readings through `Add` and `AddRange` with a collection expression;
   - deliberately add a couple of outliers (e.g. `-999.0` and `250.0`) and a few consecutive duplicates;
   - print `Count` and `Capacity` before and after `Trim()`;
   - call `RemoveOutliers(0, 100)`, then `CollapseDuplicates()`;
   - print the statistics;
   - demonstrate that an external attempt `log.Readings.Add(...)` **does not compile** — comment the line out with an explanation of why it is forbidden.

7. Build and run: `dotnet build`, then `dotnet run`. Verify the expected output: before `RemoveOutliers` `Count` is larger, after — smaller; `Capacity` shrinks after `Trim()`; statistics are computed correctly.

8. Add unit tests (optional, via `dotnet new xunit`) for: an empty log throws `InvalidOperationException`; `RemoveOutliers` returns the correct count; `CollapseDuplicates` leaves non-repeating values untouched; `RemoveAt(-1)` returns `false`.

#### Requirements

- The code must compile under C# 12 / .NET 8 without warnings, including `nullable` warnings.
- Use `List<T>` and `IReadOnlyList<T>` from `System.Collections.Generic`; `ArrayList` may appear only in `BoxingDemo` to demonstrate the problem.
- Set `capacity` in the `SensorLog` constructor and call `TrimExcess()` in `Trim()`.
- Mutating `_readings` inside a `foreach` over it is forbidden; for conditional removal use `RemoveAll`.
- Public properties and collection-returning members are `IReadOnlyList<Reading>` (or a copy). No public signature returns the bare `List<Reading>`.
- The read-only indexer has no `set`; `RemoveAt` is public but index-guarded.
- The code is readable, with EN+RU comments in key places (boxing, capacity, read-only contract).

#### Pitfalls

- **Boxing hides.** `ArrayList.Add(int)` boxes the value into `object` — you cannot see it in the code, but you can see it in allocations. In `BoxingDemo` explicitly mark the boxing line and the unboxing line in `foreach (int n in ...)`. Remember that even `boxed[0]` without a cast returns `object`, not `int`.
- **`Count` is not `Capacity`.** In the `SensorLog` output you will see `Capacity` jump in steps (doubling) while `Count` grows one at a time. Do not drive logic off `Capacity` and do not expose it as a "size".
- **Mutating inside `foreach` = `InvalidOperationException`.** If you call `_readings.Remove(...)` while iterating `_readings`, the runtime throws "Collection was modified". The fix is `RemoveAll`, which walks the list safely, or collecting indices and removing from the end.
- **Leaking a mutable reference.** If the property were `public List<Reading> Readings => _readings;`, any external code could call `log.Readings.Clear()`. `IReadOnlyList<T>` closes `Add`/`Remove` at the compiler level — that is the "do not touch" contract from the lesson.
- **`IndexOf` and `-1`.** Always check that an index was found before `RemoveAt`. In `CollapseDuplicates` compare neighbours through the indexer, carefully handling the `i == 0` boundary.
- **`Remove(item)` and equality.** Because `Reading` is a `record struct`, equality is structural and `Remove`/`IndexOf` work by value, not by reference. If it were a class without an overridden `Equals`, removal would go by reference.
- **Capacity and `AddRange`.** If `AddRange` exceeds the current `Capacity`, a single reallocation happens instead of a series — that is more efficient than calling `Add` in a loop.

#### Acceptance criteria

- [ ] The project builds under .NET 8 / C# 12 with no errors or warnings.
- [ ] `BoxingDemo.Run()` throws `InvalidCastException` on the `ArrayList` with a `string` and catches it correctly.
- [ ] `BoxingDemo.Run()` shows that `List<int>` refuses a `string` at compile time.
- [ ] Comments mark the boxing and unboxing lines.
- [ ] `SensorLog` stores data in a `List<Reading>` with `capacity` set.
- [ ] The public `Readings` is typed `IReadOnlyList<Reading>`.
- [ ] `Count` and a read-only indexer are implemented.
- [ ] `Statistics()` throws `InvalidOperationException` for an empty log.
- [ ] `RemoveOutliers` uses `RemoveAll` and returns the number removed.
- [ ] `CollapseDuplicates` removes only consecutive duplicates.
- [ ] `RemoveAt` safely handles `-1` and out-of-range indices.
- [ ] `Trim()` calls `TrimExcess()`.
- [ ] The output shows the `Count`/`Capacity` difference before and after `Trim()`.
- [ ] The line `log.Readings.Add(...)` is commented out with an explanation of why it does not compile.
- [ ] No list mutation happens inside `foreach`.

#### Hints (no direct answer)

- For `RemoveOutliers` the `RemoveAll` predicate is a lambda returning `true` for elements you want to **remove** (i.e. those outside the range). Do not invert the direction by mistake.
- In `CollapseDuplicates` compare `i > 0 && _readings[i].Equals(_readings[i-1])` inside the `RemoveAll` predicate.
- `Statistics` can be computed in a single `for` loop over `Count` and the indexer, without LINQ — to avoid extra allocations.
- `IReadOnlyList<Reading>` is implicitly convertible from `List<Reading>` — no separate `ToArray` is needed.
- For a `record struct` equality is already implemented — you do not need `IEquatable<Reading>` by hand.

#### Reference solution walk-through

```csharp
// C# 12 / .NET 8
// BoxingDemo.cs — demo of the ArrayList problem

using System.Collections;

namespace GreenhouseMonitor;

public static class BoxingDemo
{
    public static void Run()
    {
        // --- Old approach: ArrayList backed by object ---
        ArrayList boxed = new();
        boxed.Add(1);          // boxing: int → object
        boxed.Add(2);
        boxed.Add(3);
        boxed.Add("oops");     // compiler does not complain

        try
        {
            // unboxing on each (int) cast
            int sum = 0;
            foreach (int n in boxed)   // InvalidCastException on "oops"
                sum += n;
            Console.WriteLine($"sum = {sum}");
        }
        catch (InvalidCastException ex)
        {
            Console.WriteLine($"ArrayList broke: {ex.Message}");
        }

        // --- Modern approach: List<int> ---
        List<int> numbers = new(capacity: 4) { 1, 2, 3 };
        // numbers.Add("oops");  // compile-time error: type safety
        Console.WriteLine($"List<int> sum = {numbers.Sum()}");
    }
}
```

```csharp
// Reading.cs — a single sensor reading
namespace GreenhouseMonitor;

public readonly record struct Reading(DateTime Offset, double Celsius);
```

```csharp
// SensorLog.cs — main storage on List<T> + IReadOnlyList<T>
namespace GreenhouseMonitor;

public sealed class SensorLog
{
    private readonly List<Reading> _readings;

    // set capacity up front to avoid extra reallocations
    public SensorLog(int expectedCount = 1440)
        => _readings = new List<Reading>(capacity: expectedCount);

    public void Add(Reading r) => _readings.Add(r);

    public void AddRange(IEnumerable<Reading> items) => _readings.AddRange(items);

    // Public API — read only: external mutation is forbidden by the compiler.
    public IReadOnlyList<Reading> Readings => _readings;

    public int Count => _readings.Count;

    // Indexer without setter — read-only to the outside.
    public Reading this[int index] => _readings[index];

    public (double Min, double Max, double Avg) Statistics()
    {
        if (_readings.Count == 0)
            throw new InvalidOperationException("log is empty");

        double min = double.PositiveInfinity;
        double max = double.NegativeInfinity;
        double sum = 0;
        for (int i = 0; i < _readings.Count; i++)
        {
            double v = _readings[i].Celsius;
            if (v < min) min = v;
            if (v > max) max = v;
            sum += v;
        }
        return (min, max, sum / _readings.Count);
    }

    // Remove outliers in one pass via RemoveAll — no mutation inside foreach.
    public int RemoveOutliers(double minValid, double maxValid)
        => _readings.RemoveAll(r => r.Celsius < minValid || r.Celsius > maxValid);

    // Remove consecutive duplicates: drop an element equal to the previous one.
    public int CollapseDuplicates()
        => _readings.RemoveAll(r =>
        {
            int i = _readings.IndexOf(r);
            return i > 0 && _readings[i - 1].Equals(r);
        });

    // Safe RemoveAt — always validate the index.
    public bool RemoveAt(int index)
    {
        if (index < 0 || index >= _readings.Count) return false;
        _readings.RemoveAt(index);
        return true;
    }

    public void Trim() => _readings.TrimExcess();
}
```

```csharp
// Program.cs — top-level entry
using GreenhouseMonitor;

BoxingDemo.Run();
Console.WriteLine("---");

var log = new SensorLog(expectedCount: 10);

var t0 = DateTime.Today;
log.Add(new Reading(t0.AddMinutes(1), 20.5));
log.AddRange([
    new Reading(t0.AddMinutes(2), 20.5),   // consecutive duplicate
    new Reading(t0.AddMinutes(3), 21.0),
    new Reading(t0.AddMinutes(4), -999.0), // outlier
    new Reading(t0.AddMinutes(5), 250.0),  // outlier
    new Reading(t0.AddMinutes(6), 22.0),
]);

Console.WriteLine($"Before filter: Count={log.Count}");
int removedOut = log.RemoveOutliers(0, 100);
Console.WriteLine($"Outliers removed: {removedOut}");
int removedDup = log.CollapseDuplicates();
Console.WriteLine($"Consecutive duplicates removed: {removedDup}");

var (min, max, avg) = log.Statistics();
Console.WriteLine($"Min={min}, Max={max}, Avg={avg:F2}");

Console.WriteLine($"log[0] = {log[0]}");
Console.WriteLine($"RemoveAt(-1) → {log.RemoveAt(-1)}"); // false

// log.Readings.Add(...);  // won't compile: IReadOnlyList<T> does not expose Add

log.Trim();
Console.WriteLine("After Trim() spare Capacity is released.");
```

**Line-by-line walk-through.** `BoxingDemo` reproduces exactly the lesson's motivating error: `ArrayList` accepts `int` and `string` without compiler complaints, and `foreach (int n in boxed)` fails at runtime with `InvalidCastException`, because on the `"oops"` string an implicit unboxing cast happens. With `List<int>` the same scenario is impossible — `numbers.Add("oops")` is rejected by the compiler: that is exactly the type safety generics promise. The field `_readings` is declared `private readonly List<Reading>`, and the `SensorLog` constructor passes `expectedCount` as `capacity` — a best practice from the lesson: when the size is known, there are no reallocations and no `Capacity` growth. The `Readings` property returns `_readings` as `IReadOnlyList<Reading>`: the implicit conversion works because `List<T>` implements that interface; from the outside only `Count` and the indexer are visible, `Add`/`Remove` are hidden — external code physically cannot call `log.Readings.Clear()`. The indexer is declared with `get` only (no `set`), which blocks writes through `log[i] = ...`. `Statistics` checks emptiness and throws `InvalidOperationException` — an honest boundary handling instead of NaNs. `RemoveOutliers` uses `RemoveAll` with a "value out of range" predicate, which solves two lesson problems at once: no mutation inside `foreach` (which would throw `InvalidOperationException`) and removal in a single pass, without an expensive `Remove(item)` in a loop. `CollapseDuplicates` uses `IndexOf` to find the element's position and compares it with the previous one — here it matters that `Reading` is a `record struct` with structural equality, so `IndexOf` finds exactly that value, not some other reference. `RemoveAt` is public, but with bounds checking and a `bool` return — a direct response to the lesson's common mistake "ignoring `-1` from `IndexOf` and out-of-range indices". `Trim()` is a thin wrapper over `TrimExcess()` that releases spare `Capacity` after filling. In `Program.cs` a collection expression `[...]` is used for `AddRange` — a modern C# 12 syntactic feature recommended over `new List<T>{...}` or `Add` in a loop. The commented line `log.Readings.Add(...)` is a tangible proof that the `IReadOnlyList<T>` contract works at the compiler level, not merely on a gentleman's agreement.

#### Going deeper (bonus)

1. Add a method `public IReadOnlyList<Reading> Snapshot()` via `ToArray()` that returns a copy of the data, and benchmark it against returning `IReadOnlyList<T>` without a copy (use `BenchmarkDotNet`).
2. Implement `CollapseDuplicates` without `RemoveAll`, by building a new `List<Reading>` and replacing `_readings` through `Clear()` + `AddRange` — explain why this is less efficient and what the catch is with the `readonly` field.
3. Extend `SensorLog` to a generic `SeriesLog<T>` (prep for the next lesson M06-L02), where `T : struct` is a type parameter and the `Statistics` logic is delegated to an external `IStatCalculator<T>`.
4. Use `dotnet-counters` to show that with `ArrayList<double>` the `Gen 0` allocation volume is noticeably higher than with `List<double>` over 10000 elements, and explain the connection to boxing.

---

#### Чек-лист сдачи / Submission checklist
- [ ] Проект собирается под .NET 8 / C# 12 без предупреждений.
- [ ] `BoxingDemo` ловит `InvalidCastException` и показывает типобезопасность `List<int>`.
- [ ] `SensorLog` использует `List<Reading>` с `capacity` и `TrimExcess()`.
- [ ] Публичный API — только `IReadOnlyList<Reading>`.
- [ ] `RemoveOutliers` и `CollapseDuplicates` используют `RemoveAll`.
- [ ] `RemoveAt` проверяет границы и возвращает `bool`.
- [ ] Вывод показывает разницу `Count`/`Capacity` до и после `Trim()`.
- [ ] Закомментирована попытка `log.Readings.Add(...)` с объяснением.
- [ ] Нет мутаций списка внутри `foreach`.
- [ ] The project builds under .NET 8 / C# 12 with no warnings.
- [ ] `BoxingDemo` catches `InvalidCastException` and demonstrates `List<int>` type safety.
- [ ] `SensorLog` uses `List<Reading>` with `capacity` and `TrimExcess()`.
- [ ] Public API is `IReadOnlyList<Reading>` only.
- [ ] `RemoveOutliers` and `CollapseDuplicates` use `RemoveAll`.
- [ ] `RemoveAt` validates bounds and returns `bool`.
- [ ] The output shows the `Count`/`Capacity` difference before and after `Trim()`.
- [ ] The `log.Readings.Add(...)` attempt is commented out with an explanation.
- [ ] No list mutation inside `foreach`.

#### Ресурсы / Resources
- [Microsoft Learn — Generics — https://learn.microsoft.com/dotnet/csharp/fundamentals/types/generics]
- [Microsoft Learn — List<T> — https://learn.microsoft.com/dotnet/api/system.collections.generic.list-1]
- [Microsoft Learn — IReadOnlyList<T> — https://learn.microsoft.com/dotnet/api/system.collections.generic.ireadonlylist-1]
- [Microsoft Learn — Boxing and Unboxing — https://learn.microsoft.com/dotnet/csharp/programming-guide/types/boxing-and-unboxing]
- [Microsoft Learn — Collection expressions (C# 12) — https://learn.microsoft.com/dotnet/csharp/language-reference/operators/collection-expressions]
