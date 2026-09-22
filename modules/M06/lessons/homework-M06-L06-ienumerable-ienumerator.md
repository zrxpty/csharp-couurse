---
[← К уроку M06-L06](lesson-M06-L06-ienumerable-ienumerator.md) | [⬆ К модулю M06](../README.md) | [Следующее ДЗ →](homework-M06-L07-yield.md)
---

### Домашнее задание M06-L06: IEnumerable<T>/IEnumerator<T>, итерация / Homework M06-L06: IEnumerable<T>/IEnumerator<T>, iteration

**Урок / Lesson:** M06-L06
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 4/5
**Цель / Goal:** (RU) На практике пройти весь путь от ручной реализации `IEnumerable<T>`/`IEnumerator<T>` до ленивых генераторов на `yield return`, научиться различать отложенное и жадное выполнение LINQ, увидеть и устранить проблему множественного перечисления. (EN) Practically walk the whole path from a hand-written `IEnumerable<T>`/`IEnumerator<T>` to lazy `yield return` generators, learn to distinguish deferred from eager LINQ execution, and observe then fix the multiple-enumeration problem.

#### Связь с уроком / Connection to the lesson
(RU) Урок показывает, что `IEnumerable<T>` — это контракт на выдачу перечислителя, `IEnumerator<T>` — сам «указатель» с `MoveNext`/`Current`/`Reset`/`Dispose`, а `foreach` разворачивается компилятором именно в эту четвёрку вызовов. Домашнее задание заставляет вас реализовать эту четвёрку своими руками, затем переписать часть логики через `yield return` и на наглядном счётчике перечислений увидеть, что отложенные операторы лишь строят конвейер, а жадные — запускают его. (EN) The lesson shows that `IEnumerable<T>` is a contract for handing out an enumerator, `IEnumerator<T>` is the "pointer" itself with `MoveNext`/`Current`/`Reset`/`Dispose`, and `foreach` is expanded by the compiler into exactly these four calls. The homework makes you implement this quartet by hand, then rewrite part of the logic with `yield return`, and use a visible enumeration counter to confirm that deferred operators only build a pipeline while eager ones trigger it.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация

Вы присоединились к команде, которая пишет внутреннюю мини-библиотеку телеметрии `M06L06.Homework`. Библиотека должна уметь обходить потоки значений единообразно — будь то конечный диапазон целых чисел, бесконечная последовательность натуральных чисел или «скользящее окно» поверх массива показаний датчика. Команда сознательно отказалась от готовых коллекций там, где важна нулевая аллокация в горячих циклах, и хочет видеть, как именно устроена итерация «под капотом», прежде чем доверять её `yield return` и LINQ.

Ваш технический лидер поставил задачу: построить три ключевых компонента и один демонстрационный стенд. Первый компонент — `IntRange`, диапазон целых чисел, реализованный вручную через `IEnumerable<int>` и собственный `IEnumerator<int>` в виде `struct`, чтобы показать механику `MoveNext`/`Current`/`Reset`/`Dispose` без единой аллокации в куче. Второй компонент — `WindowSequence`, набор методов-расширений на `yield return`, которые строят ленивые последовательности: бесконечный генератор натуральных чисел и скользящее окно фиксированной ширины поверх произвольного источника. Третий компонент — `CountingEnumerable<T>`, диагностическая обёртка, считающая, сколько раз реальный источник перечислялся; она нужна, чтобы доказать себе и ревьюеру, что отложенные операторы LINQ не запускают вычисление, а жадные — запускают, и что повторный `foreach` по одному и тому же конвейеру каждый раз заново читает источник. Демонстрационный стенд в `Program.Main` должен последовательно показать все четыре сценария и напечатать значения счётчика перечислений в ключевых точках, чтобы вывод был самодокументируемым.

Эта задача ценна тем, что она соединяет три уровня понимания из урока: низкоуровневую механику перечислителей, высокоуровневый синтаксис `yield return` и операторную модель LINQ с её отложенностью. Вы не просто используете `foreach` — вы видите, во что он превращается, и учитесь принимать инженерные решения: когда достаточно `yield return`, когда нужен ручной `struct`-перечислитель ради производительности, и когда обязательно материализовать результат через `ToList()`, чтобы не перечитывать дорогой источник многократно.

#### Что нужно сделать (пошагово)

1. **Создайте проект.** В терминале выполните `dotnet new console -n M06L06.Homework -o M06L06.Homework -f net8.0`, затем перейдите в папку проекта `cd M06L06.Homework`. Откройте `M06L06.Homework.csproj` и убедитесь, что `TargetFramework` равен `net8.0` и включён `<Nullable>enable</Nullable>` и `<ImplicitUsings>enable</ImplicitUsings>`. Добавьте при необходимости `<LangVersion>latest</LangVersion>`, чтобы были доступны конструкции C# 12 (включая collection expressions `[1, 2, 3]`).

2. **Реализуйте `IntRange`.** В файле `IntRange.cs` объявите `public readonly struct IntRange : IEnumerable<int>`. Храните поля `_start` и `_count` типа `int`. Конструктор принимает `start` и `count`; если `count < 0`, бросайте `ArgumentOutOfRangeException`. Реализуйте публичный метод `GetEnumerator()`, возвращающий вложенную `public struct Enumerator : IEnumerator<int>`. Вложенный `Enumerator` хранит `_start`, `_count`, `_index` (инициализируется `-1`, что означает «до первого `MoveNext`») и `_current` (валиден только после успешного `MoveNext`). В `MoveNext` используйте беззнаковую проверку `(uint)(_index + 1) < (uint)_count`, чтобы одновременно покрыть и выход за границу, и потенциальное переполнение: если условие истинно, увеличьте `_index`, вычислите `_current = _start + _index` и верните `true`, иначе `false`. Реализуйте `Current` (типизированный и нетипизированный через `object IEnumerator.Current`), `Reset` (вернуть `_index` в `-1`) и пустой `Dispose` (неуправляемых ресурсов нет, но контракт `IEnumerator` требует метод). Дополнительно реализуйте явные реализации интерфейсов `IEnumerable<int>.GetEnumerator()` и `IEnumerable.GetEnumerator()`, делегирующие в типизированный `GetEnumerator()`.

3. **Реализуйте `WindowSequence`.** В файле `WindowSequence.cs` создайте `public static class WindowSequence`. Метод `Naturals(int start = 1)` возвращает `IEnumerable<int>` через `while (true) yield return start++;` — бесконечный генератор, безопасный только в связке с `Take`. Метод-расширение `SlidingWindows<T>(this IEnumerable<T> source, int windowSize)` возвращает `IEnumerable<IReadOnlyList<T>>`: валидирует аргументы, затем через `using var e = source.GetEnumerator()` обходит источник, поддерживает `List<T>`-буфер шириной `windowSize`, удаляет головной элемент при превышении ширины и через `yield return buffer.ToArray()` отдаёт копию окна. Копия обязательна: иначе внешний код, мутировавший возвращённый список, испортил бы внутренний буфер генератора.

4. **Реализуйте `CountingEnumerable<T>`.** В файле `CountingEnumerable.cs` объявьте `public sealed class CountingEnumerable<T> : IEnumerable<T>`. Конструктор принимает `IEnumerable<T> source` и бросает `ArgumentNullException` при `null`. Публичное свойство `int EnumerationCount { get; private set; }` инкрементируется в каждом `GetEnumerator()`. Оба `GetEnumerator` (типизированный и нетипизированный) делегируют в `_source.GetEnumerator()`, но только после инкремента счётчика. Это и есть видимый «датчик» перечислений: каждый вызов `GetEnumerator` означает один полный проход по источнику.

5. **Напишите демонстрационный стенд `Program.cs`** с top-level statements или классическим `Main`. Сценарий A: создайте `new IntRange(10, 5)`, обойдите его `foreach` (ожидаемый вывод `10 11 12 13 14`), затем вручную разверните `foreach` через `using var e = range.GetEnumerator(); while (e.MoveNext()) ...` и получите тот же вывод. Сценарий B: возьмите `int[] data = [1, 2, 3, 4, 5];` (collection expression C# 12), вызовите `data.SlidingWindows(3)` и напечатайте окна `1,2,3`, `2,3,4`, `3,4,5`; затем `WindowSequence.Naturals().Take(5).ToList()` и напечатайте `1,2,3,4,5`. Сценарий C: оберните `new[] { 1, 2, 3, 4, 5, 6 }` в `CountingEnumerable<int>`, постройте LINQ-конвейер `.Where(x => x % 2 == 0).Select(x => x * x)`, напечатайте `counting.EnumerationCount` (должно быть `0` — конвейер ещё не запущен), вызовите `pipeline.ToList()` (станет `1`), напечатайте результат `4,16,36`, затем вызовите `pipeline.Count()` (станет `2`) и снова напечатайте счётчик. Сценарий D: материализуйте конвейер один раз через `ToList()`, затем вызовите `.Sum()` и `.Max()` уже на списке — счётчик больше не должен расти.

6. **Запустите и проверьте.** Выполните `dotnet build`, убедитесь в отсутствии предупреждений (особенно `CA`-диагностик по nullable и `IDisposable`), затем `dotnet run`. Сверьте вывод со спецификацией выше. При необходимости добавьте `dotnet run --configuration Release` и заметьте, что поведение отложенности не зависит от конфигурации.

#### Требования к решению

- Проект на .NET 8 (`net8.0`), C# 12, `Nullable` и `ImplicitUsings` включены; используется как минимум одна collection expression и как минимум один `using`-statement для перечислителя.
- `IntRange` — `readonly struct`, реализующий `IEnumerable<int>` с типизированным `struct`-перечислителем `Enumerator`, реализующим `IEnumerator<int>` полностью (`MoveNext`, `Current`, `object IEnumerator.Current`, `Reset`, `Dispose`).
- `Enumerator.MoveNext` использует беззнаковую проверку границ и корректно возвращает `false` после последнего элемента; `Current` не бросает, если вызван до первого `MoveNext` (допускается возврат значения по умолчанию, поведение задокументировано в комментарии).
- `WindowSequence.SlidingWindows` построен на `yield return`, валидирует `null` и `windowSize <= 0`, возвращает копии окон (`ToArray()`), корректно освобождает перечислитель источника через `using`.
- `CountingEnumerable<T>` инкрементирует `EnumerationCount` строго один раз на каждый `GetEnumerator` и делегирует реальный перечислитель источника без буферизации.
- Демонстрационный стенд печатает счётчик перечислений в четырёх ключевых точках: до материализации, после `ToList`, после повторного `Count()`, и подтверждает, что при работе с материализованным списком счётчик не растёт.
- В коде присутствуют двуязычные комментарии (RU+EN) в ключевых местах: разворот `foreach`, беззнаковая проверка границ, причина `yield return`, причина материализации.
- Код компилируется без ошибок и предупреждений уровня `warning` в конфигурации `Debug` и работает одинаково в `Debug` и `Release`.

#### Тонкости и подводные камни

- **Беззнаковая проверка границ.** Условие `(uint)(_index + 1) < (uint)_count` безопасно против отрицательного `_count` и переполнений: при `_count < 0` правая часть становится огромным положительным числом, и сравнение ложно. Это идиома из исходников `Span<T>` и `Enumerable.Range` — не изобретайте свою арифметику.
- **`struct`-перечислитель и боксирование.** Типизированный `GetEnumerator()` возвращает `Enumerator` по значению, и `foreach` по `IntRange` не аллоцирует. Но если передать `IntRange` туда, где ожидается `IEnumerable<int>`, и вызвать `foreach`, сработает явная интерфейсная реализация — она тоже возвращает `Enumerator`, но уже упакованный в `IEnumerator<int>`, и аллокация появится. Это компромисс, о котором стоит знать.
- **`Dispose` обязателен.** Даже если ресурсов нет, `IEnumerator` наследует `IDisposable`, и `foreach` всегда вызывает `Dispose()` в блоке `finally`. Пустой `Dispose` — это честная реализация контракта, а не заглушка.
- **`yield return` и копия окна.** Если вернуть сам `buffer` вместо `buffer.ToArray()`, все окна будут ссылаться на один и тот же мутируемый список — классический баг ленивых генераторов. Возвращайте копию или `IReadOnlyList<T>` поверх копии.
- **Отложенность ловит на `Count()`.** Многие думают, что `.Count()` «дёшев», потому что у `List<T>` это свойство. Но у `IEnumerable<T>` это метод-расширение, который перечисляет весь источник. На отложенном конвейере `pipeline.Count()` запускает полный проход.
- **Множественное перечисление невидимо без счётчика.** Без `CountingEnumerable` вы бы не заметили, что `pipeline` перечитывает источник при каждом `foreach`. В реальном коде с `IQueryable` к БД это превращается в N одинаковых запросов.
- **`Reset` почти не используется.** Реализуйте его честно (вернуть `_index = -1`), но не полагайтесь на него: многие перечислители бросают `NotSupportedException`. `foreach` его не вызывает.
- **Изменение коллекции во время `foreach`.** Если бы `IntRange` хранил изменяемый массив, удаление элемента на обходе бросило бы `InvalidOperationException`. Наш `readonly struct` иммутабелен — это спасает.

#### Критерии приёмки

- [ ] Проект `M06L06.Homework` собирается под `net8.0` без ошибок и предупреждений.
- [ ] `IntRange` — `readonly struct`, реализующий `IEnumerable<int>` с вложенным `struct Enumerator : IEnumerator<int>`.
- [ ] `Enumerator.MoveNext` использует беззнаковую проверку `(uint)(_index + 1) < (uint)_count` и корректно завершает итерацию.
- [ ] Реализованы `Current`, `object IEnumerator.Current`, `Reset`, `Dispose`; `Dispose` пустой, но присутствует.
- [ ] `foreach` по `IntRange` и ручной разворот через `GetEnumerator`/`MoveNext`/`Current` дают идентичный вывод.
- [ ] `WindowSequence.Naturals()` — бесконечный генератор на `yield return`, безопасный в связке с `Take`.
- [ ] `SlidingWindows<T>` валидирует `null` и `windowSize <= 0`, возвращает копии окон через `ToArray()`.
- [ ] `CountingEnumerable<T>` инкрементирует `EnumerationCount` один раз на каждый `GetEnumerator`.
- [ ] После построения LINQ-конвейера `EnumerationCount == 0` (отложенность доказана).
- [ ] После `pipeline.ToList()` `EnumerationCount == 1` (материализация запустила один проход).
- [ ] После `pipeline.Count()` `EnumerationCount` увеличивается (множественное перечисление доказано).
- [ ] После перехода на материализованный `List<int>` счётчик перестаёт расти при `.Sum()`/`.Max()`.
- [ ] Вывод `dotnet run` соответствует спецификации: диапазон, окна, первые пять натуральных, ряд квадратов чётных.
- [ ] В коде есть двуязычные комментарии RU+EN в ключевых точках.
- [ ] Использована хотя бы одна collection expression C# 12 и хотя бы один `using`-statement.

#### Подсказки (без прямого ответа)

- Вспомните аналог из урока: «библиотекарь выдаёт карточку». `GetEnumerator` — это библиотекарь, `Enumerator` — карточка. Где в вашем коде «выдача карточки», а где «листание»?
- Для `MoveNext` посмотрите, как `Enumerable.Range` проверяет границы через `(uint)`. Подумайте, почему `int`-сравнение `_index + 1 < _count` было бы недостаточно при `_count < 0`.
- Для `SlidingWindows` задайте себе вопрос: что произойдёт, если вернуть `buffer` напрямую, а затем внешняя мутация поменяет `buffer[0]`? Как `yield return` переиспользует один и тот же объект между итерациями?
- Для счётчика перечислений помните, что `GetEnumerator()` вызывается не только `foreach`, но и любым LINQ-оператором, а также `ToList`/`ToArray`/`Count`. Каждый из них — отдельный проход.
- Чтобы «починить» множественное перечисление, спросите: где живёт результат после `ToList()`? В `List<T>`, у которого `Count` — свойство, а `Sum`/`Max` идут по уже материализованным данным без повторного обращения к источнику.

#### Эталонное решение (разбор)

```csharp
using System.Collections;
using System.Collections.Generic;
using System.Linq;

namespace M06L06.Homework;

// RU: Ручная реализация IEnumerable<int>/IEnumerator<int> — как Enumerable.Range, но своими руками.
// EN: Hand-written IEnumerable<int>/IEnumerator<int> — like Enumerable.Range, but built by hand.
public readonly struct IntRange : IEnumerable<int>
{
    private readonly int _start;
    private readonly int _count;

    public IntRange(int start, int count)
    {
        if (count < 0)
            throw new ArgumentOutOfRangeException(nameof(count), "Count must be non-negative.");
        _start = start;
        _count = count;
    }

    public int Count => _count;

    // RU: Типизированный GetEnumerator — путь без боксирования для foreach по IntRange.
    // EN: Typed GetEnumerator — the boxing-free path used by foreach over IntRange.
    public Enumerator GetEnumerator() => new Enumerator(_start, _count);

    IEnumerator<int> IEnumerable<int>.GetEnumerator() => GetEnumerator(); // здесь возможна аллокация
    IEnumerator IEnumerable.GetEnumerator() => GetEnumerator();

    // RU: struct-перечислитель: не аллоцируется в куче, эффективен в горячих циклах.
    // EN: struct enumerator: not heap-allocated, efficient in hot loops.
    public struct Enumerator : IEnumerator<int>
    {
        private readonly int _start;
        private readonly int _count;
        private int _index;   // -1 до первого MoveNext
        private int _current; // валиден только после успешного MoveNext

        internal Enumerator(int start, int count)
        {
            _start = start;
            _count = count;
            _index = -1;
            _current = 0;
        }

        public int Current => _current;
        object IEnumerator.Current => _current;

        public bool MoveNext()
        {
            // RU: Беззнаковая проверка покрывает и выход за границу, и отрицательный _count.
            // EN: Unsigned check covers both overrun and a negative _count.
            if ((uint)(_index + 1) < (uint)_count)
            {
                _index++;
                _current = _start + _index;
                return true;
            }
            return false;
        }

        // RU: Reset возвращает указатель «до первого элемента»; foreach его не вызывает.
        // EN: Reset rewinds to "before the first element"; foreach never calls it.
        public void Reset() => _index = -1;

        // RU: Неуправляемых ресурсов нет, но Dispose обязателен по контракту IEnumerator.
        // EN: No unmanaged resources, but Dispose is mandatory per the IEnumerator contract.
        public void Dispose() { }
    }
}

// RU: Ленивые генераторы на yield return — компилятор строит класс-конечный автомат.
// EN: Lazy generators via yield return — the compiler builds a state-machine class.
public static class WindowSequence
{
    // RU: Бесконечная последовательность; безопасна только с Take().
    // EN: Infinite sequence; safe only with Take().
    public static IEnumerable<int> Naturals(int start = 1)
    {
        while (true) yield return start++;
    }

    // RU: Скользящее окно фиксированной ширины поверх произвольного источника.
    // EN: Fixed-size sliding window over any source.
    public static IEnumerable<IReadOnlyList<T>> SlidingWindows<T>(
        this IEnumerable<T> source, int windowSize)
    {
        if (source is null) throw new ArgumentNullException(nameof(source));
        if (windowSize <= 0) throw new ArgumentOutOfRangeException(nameof(windowSize));

        using IEnumerator<T> e = source.GetEnumerator();
        var buffer = new List<T>(windowSize);

        while (e.MoveNext())
        {
            buffer.Add(e.Current);
            if (buffer.Count > windowSize)
                buffer.RemoveAt(0);

            if (buffer.Count == windowSize)
                // RU: Копия, чтобы внешняя мутация не испортила внутренний буфер.
                // EN: A copy so external mutation cannot corrupt the inner buffer.
                yield return buffer.ToArray();
        }
    }
}

// RU: Диагностическая обёртка: считает реальные проходы по источнику.
// EN: Diagnostic wrapper: tallies real passes over the source.
public sealed class CountingEnumerable<T> : IEnumerable<T>
{
    private readonly IEnumerable<T> _source;
    public int EnumerationCount { get; private set; }

    public CountingEnumerable(IEnumerable<T> source)
        => _source = source ?? throw new ArgumentNullException(nameof(source));

    public IEnumerator<T> GetEnumerator()
    {
        EnumerationCount++;             // каждый GetEnumerator = один проход
        return _source.GetEnumerator();
    }

    IEnumerator IEnumerable.GetEnumerator() => GetEnumerator();
}

public static class Program
{
    public static void Main()
    {
        // --- Сценарий A: ручная итерация vs foreach ---
        var range = new IntRange(10, 5); // 10,11,12,13,14
        foreach (int n in range) System.Console.Write(n + " ");
        System.Console.WriteLine();

        // Ручной разворот foreach — то же, что делает компилятор.
        using var e = range.GetEnumerator();
        while (e.MoveNext()) System.Console.Write(e.Current + " ");
        System.Console.WriteLine();

        // --- Сценарий B: yield return — окна и бесконечная последовательность ---
        int[] data = [1, 2, 3, 4, 5]; // collection expression C# 12
        foreach (var w in data.SlidingWindows(3))
            System.Console.WriteLine(string.Join(",", w)); // 1,2,3 / 2,3,4 / 3,4,5

        List<int> firstFive = WindowSequence.Naturals().Take(5).ToList();
        System.Console.WriteLine(string.Join(",", firstFive)); // 1,2,3,4,5

        // --- Сценарий C: отложенность и множественное перечисление ---
        var counting = new CountingEnumerable<int>(new[] { 1, 2, 3, 4, 5, 6 });
        IEnumerable<int> pipeline = counting.Where(x => x % 2 == 0).Select(x => x * x);

        System.Console.WriteLine($"Before materialization: {counting.EnumerationCount}"); // 0
        List<int> squares = pipeline.ToList(); // запуск -> 1
        System.Console.WriteLine($"After ToList: {counting.EnumerationCount} => {string.Join(",", squares)}");
        _ = pipeline.Count(); // повторный проход -> 2
        System.Console.WriteLine($"After Count(): {counting.EnumerationCount}");

        // --- Сценарий D: материализовать один раз, дальше не трогать источник ---
        List<int> materialized = pipeline.ToList(); // ещё один проход -> 3
        int sum = materialized.Sum();
        int max = materialized.Max();
        System.Console.WriteLine($"sum={sum}, max={max}, enumerations={counting.EnumerationCount}");
    }
}
```

Разбор по строкам. `IntRange` — `readonly struct`, что даёт две вещи: иммутабельность (поля нельзя изменить после создания) и возможность быть `in`-параметром без защитного копирования. Конструктор проверяет `count < 0` и бросает `ArgumentOutOfRangeException` — это та же валидация, что в `Enumerable.Range`. Типизированный `GetEnumerator()` возвращает `Enumerator` по значению: именно его использует `foreach` по `IntRange` напрямую (без приведения к интерфейсу), и именно поэтому перебор не аллоцирует. Две явные интерфейсные реализации делегируют в тот же метод, но при вызове через интерфейс `Enumerator` упакуется в `IEnumerator<int>` — это плата за совместимость с LINQ и обобщённым кодом.

Во вложенном `Enumerator` поле `_index = -1` означает состояние «до первого `MoveNext`» — точно как требует контракт `IEnumerator` (`Current` не валиден до первого успешного `MoveNext`). Условие `(uint)(_index + 1) < (uint)_count` — беззнаковая идиома проверки границ из исходников BCL: при `_count < 0` правая часть после приведения к `uint` становится очень большим числом, сравнение ложно, и метод сразу вернёт `false`, не выдав ни одного элемента. `Reset` возвращает `_index = -1`; он реализован честно, но, как отмечено в уроке, `foreach` его не вызывает, а многие перечислители вообще бросают `NotSupportedException`. `Dispose` пустой — это корректная реализация контракта `IDisposable`, который `IEnumerator` наследует; `foreach` всегда вызывает его в `finally`.

`WindowSequence.Naturals` — бесконечный генератор: `while (true) yield return start++`. Компилятор превращает этот метод в скрытый класс-конечный автомат, реализующий и `IEnumerable<int>`, и `IEnumerator<int>`; значения считаются по одному, по запросу. Поэтому `Naturals().Take(5).ToList()` не зацикливается и не переполняет память — берётся ровно пять значений. `SlidingWindows` обходит источник через `using IEnumerator<T> e = source.GetEnumerator()` (корректный `Dispose` в конце), поддерживает буфер ширины `windowSize` и через `yield return buffer.ToArray()` отдаёт копию каждого окна. Копия критична: верни мы сам `buffer`, все окна ссылались бы на один мутируемый список — классический баг ленивых последовательностей.

`CountingEnumerable<T>` инкрементирует `EnumerationCount` в каждом `GetEnumerator()`. Это видимый «датчик» отложенности: построение `pipeline` через `Where`/`Select` не вызывает `GetEnumerator` ни разу, поэтому счётчик равен `0`. `ToList()` вызывает один раз — счётчик `1`. Повторный `pipeline.Count()` вызывает ещё раз — счётчик `2`. Это и есть доказательство трёх ключевых тезисов урока: отложенные операторы строят конвейер, жадные его запускают, а каждый запуск — отдельный проход по источнику. Сценарий D материализует список один раз и дальше работает с ним: `.Sum()` и `.Max()` идут по `List<int>`, источник больше не трогается, счётчик заморожен.

#### Задания на углубление (бонус)

1. **Потокобезопасный счётчик.** Сделайте `EnumerationCount` атомарным через `Interlocked.Increment` и напишите комментарий, почему обычный `++` в многопоточной среде дал бы гонку. Подумайте, имеет ли смысл сам `IEnumerable<T>` в многопоточной среде (подсказка: контракт не гарантирует потокобезопасности).
2. **Демонстрация изменения источника.** Используйте `List<int>` как источник внутри `CountingEnumerable`, постройте конвейер, материализуйте его, затем измените исходный `List<int>` и материализуйте снова. Объясните, почему результат изменился, и как `ToList()` «замораживает» снимок.
3. **Сравнение производительности `struct` vs `class` перечислителя.** Напишите бенчмарк (можно через `System.Diagnostics.Stopwatch` в цикле на 10 миллионов итераций), сравните обход `IntRange` напрямую и через `IEnumerable<int>`. Объясните разницу аллокациями.
4. **Свой `IEnumerator<T>` с реальным `Dispose`.** Реализуйте перечислитель, читающий строки из `TextReader`, с осмысленным `Dispose`, закрывающим ридер. Покажите, что `foreach` корректно освобождает ресурс даже при исключении в теле цикла.

---

## Statement in English / Постановка на английском

#### Context & motivation

You are joining a team building an internal telemetry mini-library called `M06L06.Homework`. The library must traverse streams of values uniformly — whether the stream is a finite range of integers, an infinite sequence of natural numbers, or a sliding window over an array of sensor readings. The team has deliberately avoided ready-made collections where zero allocation in hot loops matters, and they want to see exactly how iteration works under the hood before trusting it to `yield return` and LINQ.

Your tech lead has set a task: build three key components and one demonstration harness. The first component is `IntRange`, a range of integers implemented by hand through `IEnumerable<int>` and a custom `IEnumerator<int>` exposed as a `struct`, to show the mechanics of `MoveNext`/`Current`/`Reset`/`Dispose` with no heap allocation. The second component is `WindowSequence`, a set of extension methods built on `yield return` that produce lazy sequences: an infinite generator of natural numbers and a sliding window of fixed width over an arbitrary source. The third component is `CountingEnumerable<T>`, a diagnostic wrapper that counts how many times the underlying source was actually enumerated; it exists to prove to yourself and your reviewer that deferred LINQ operators do not trigger computation, that eager operators do, and that re-running `foreach` over the same pipeline re-reads the source every time. The demonstration harness in `Program.Main` must walk through all four scenarios in order and print the enumeration counter at the key moments, so the output is self-documenting.

This task is valuable because it connects three levels of understanding from the lesson: the low-level mechanics of enumerators, the high-level syntax of `yield return`, and the operator model of LINQ with its deferred execution. You are not merely using `foreach` — you see what it compiles into, and you learn to make engineering decisions: when `yield return` is enough, when a hand-written `struct` enumerator is warranted for performance, and when materializing the result with `ToList()` is mandatory to avoid re-reading an expensive source multiple times.

#### What to do step by step

1. **Create the project.** In a terminal run `dotnet new console -n M06L06.Homework -o M06L06.Homework -f net8.0`, then `cd M06L06.Homework`. Open `M06L06.Homework.csproj` and confirm `TargetFramework` is `net8.0` with `<Nullable>enable</Nullable>` and `<ImplicitUsings>enable</ImplicitUsings>`. Add `<LangVersion>latest</LangVersion>` if needed so C# 12 features are available, including collection expressions `[1, 2, 3]`.

2. **Implement `IntRange`.** In `IntRange.cs` declare `public readonly struct IntRange : IEnumerable<int>`. Store `_start` and `_count` as `int` fields. The constructor takes `start` and `count`; throw `ArgumentOutOfRangeException` when `count < 0`. Add a public `GetEnumerator()` returning a nested `public struct Enumerator : IEnumerator<int>`. The nested `Enumerator` stores `_start`, `_count`, `_index` (initialized to `-1`, meaning "before the first `MoveNext`"), and `_current` (valid only after a successful `MoveNext`). In `MoveNext` use the unsigned check `(uint)(_index + 1) < (uint)_count` to cover both overrun and potential overflow at once: when the condition is true, increment `_index`, compute `_current = _start + _index`, and return `true`; otherwise return `false`. Implement `Current` (typed and untyped via `object IEnumerator.Current`), `Reset` (rewind `_index` to `-1`), and an empty `Dispose` (no unmanaged resources, but the `IEnumerator` contract requires the method). Add explicit interface implementations `IEnumerable<int>.GetEnumerator()` and `IEnumerable.GetEnumerator()` that delegate to the typed `GetEnumerator()`.

3. **Implement `WindowSequence`.** In `WindowSequence.cs` create `public static class WindowSequence`. The method `Naturals(int start = 1)` returns `IEnumerable<int>` via `while (true) yield return start++;` — an infinite generator, safe only in combination with `Take`. The extension method `SlidingWindows<T>(this IEnumerable<T> source, int windowSize)` returns `IEnumerable<IReadOnlyList<T>>`: validate arguments, then with `using var e = source.GetEnumerator()` walk the source, maintain a `List<T>` buffer of width `windowSize`, drop the head element when the width is exceeded, and via `yield return buffer.ToArray()` yield a copy of the window. The copy is mandatory: otherwise external code mutating the returned list would corrupt the generator's internal buffer.

4. **Implement `CountingEnumerable<T>`.** In `CountingEnumerable.cs` declare `public sealed class CountingEnumerable<T> : IEnumerable<T>`. The constructor takes `IEnumerable<T> source` and throws `ArgumentNullException` on `null`. The public property `int EnumerationCount { get; private set; }` is incremented in every `GetEnumerator()`. Both `GetEnumerator` methods (typed and untyped) delegate to `_source.GetEnumerator()`, but only after incrementing the counter. This is the visible enumeration "sensor": every `GetEnumerator` call means one full pass over the source.

5. **Write the demonstration harness `Program.cs`** with top-level statements or a classic `Main`. Scenario A: create `new IntRange(10, 5)`, iterate it with `foreach` (expected output `10 11 12 13 14`), then manually expand `foreach` via `using var e = range.GetEnumerator(); while (e.MoveNext()) ...` and obtain the same output. Scenario B: take `int[] data = [1, 2, 3, 4, 5];` (C# 12 collection expression), call `data.SlidingWindows(3)` and print windows `1,2,3`, `2,3,4`, `3,4,5`; then `WindowSequence.Naturals().Take(5).ToList()` and print `1,2,3,4,5`. Scenario C: wrap `new[] { 1, 2, 3, 4, 5, 6 }` in `CountingEnumerable<int>`, build the LINQ pipeline `.Where(x => x % 2 == 0).Select(x => x * x)`, print `counting.EnumerationCount` (should be `0` — the pipeline has not run yet), call `pipeline.ToList()` (becomes `1`), print the result `4,16,36`, then call `pipeline.Count()` (becomes `2`) and print the counter again. Scenario D: materialize the pipeline once via `ToList()`, then call `.Sum()` and `.Max()` on the list — the counter must not grow anymore.

6. **Run and verify.** Run `dotnet build`, ensure there are no warnings (especially `CA` diagnostics around nullable and `IDisposable`), then `dotnet run`. Compare the output with the specification above. If you wish, run `dotnet run --configuration Release` and note that deferred-execution behavior is independent of the build configuration.

#### Requirements

- The project targets .NET 8 (`net8.0`), C# 12, with `Nullable` and `ImplicitUsings` enabled; use at least one collection expression and at least one `using` statement for an enumerator.
- `IntRange` is a `readonly struct` implementing `IEnumerable<int>` with a typed `struct` enumerator `Enumerator` that fully implements `IEnumerator<int>` (`MoveNext`, `Current`, `object IEnumerator.Current`, `Reset`, `Dispose`).
- `Enumerator.MoveNext` uses the unsigned bounds check and correctly returns `false` after the last element; `Current` does not throw if called before the first `MoveNext` (returning the default value is acceptable, with the behavior documented in a comment).
- `WindowSequence.SlidingWindows` is built on `yield return`, validates `null` and `windowSize <= 0`, returns copies of windows (`ToArray()`), and correctly disposes the source enumerator via `using`.
- `CountingEnumerable<T>` increments `EnumerationCount` exactly once per `GetEnumerator` and delegates the real enumerator of the source without buffering.
- The demonstration harness prints the enumeration counter at four key moments: before materialization, after `ToList`, after the repeated `Count()`, and confirms that working with the materialized list no longer grows the counter.
- The code contains bilingual (RU+EN) comments at key points: the `foreach` expansion, the unsigned bounds check, the reason for `yield return`, and the reason for materialization.
- The code compiles without errors or `warning`-level diagnostics in `Debug` and behaves identically in `Debug` and `Release`.

#### Pitfalls

- **The unsigned bounds check.** The condition `(uint)(_index + 1) < (uint)_count` is safe against a negative `_count` and against overflow: when `_count < 0`, the right-hand side becomes a huge positive number and the comparison is false. This is the idiom from `Span<T>` and `Enumerable.Range` source — do not invent your own arithmetic.
- **The `struct` enumerator and boxing.** The typed `GetEnumerator()` returns `Enumerator` by value, so `foreach` over `IntRange` does not allocate. But if you pass `IntRange` where an `IEnumerable<int>` is expected and iterate it, the explicit interface implementation kicks in — it also returns `Enumerator`, but now boxed into `IEnumerator<int>`, and an allocation appears. This is a trade-off worth knowing.
- **`Dispose` is mandatory.** Even with no resources, `IEnumerator` inherits `IDisposable`, and `foreach` always calls `Dispose()` in a `finally` block. An empty `Dispose` is an honest contract implementation, not a stub.
- **`yield return` and the window copy.** If you return the `buffer` itself instead of `buffer.ToArray()`, all windows reference the same mutable list — a classic lazy-generator bug. Return a copy or an `IReadOnlyList<T>` over a copy.
- **Deferredness catches you at `Count()`.** Many assume `.Count()` is "cheap" because `List<T>` exposes it as a property. But on `IEnumerable<T>` it is an extension method that enumerates the whole source. On a deferred pipeline, `pipeline.Count()` triggers a full pass.
- **Multiple enumeration is invisible without a counter.** Without `CountingEnumerable` you would never notice that `pipeline` re-reads the source on every `foreach`. In real code over `IQueryable` against a database this turns into N identical queries.
- **`Reset` is almost never used.** Implement it honestly (rewind `_index = -1`), but never rely on it: many enumerators throw `NotSupportedException`. `foreach` does not call it.
- **Mutating a collection during `foreach`.** If `IntRange` held a mutable array, removing an element mid-iteration would throw `InvalidOperationException`. Our `readonly struct` is immutable — that saves us.

#### Acceptance criteria

- [ ] The `M06L06.Homework` project builds under `net8.0` with no errors or warnings.
- [ ] `IntRange` is a `readonly struct` implementing `IEnumerable<int>` with a nested `struct Enumerator : IEnumerator<int>`.
- [ ] `Enumerator.MoveNext` uses the unsigned check `(uint)(_index + 1) < (uint)_count` and terminates iteration correctly.
- [ ] `Current`, `object IEnumerator.Current`, `Reset`, and `Dispose` are implemented; `Dispose` is empty but present.
- [ ] `foreach` over `IntRange` and the manual expansion via `GetEnumerator`/`MoveNext`/`Current` produce identical output.
- [ ] `WindowSequence.Naturals()` is an infinite `yield return` generator, safe in combination with `Take`.
- [ ] `SlidingWindows<T>` validates `null` and `windowSize <= 0` and returns window copies via `ToArray()`.
- [ ] `CountingEnumerable<T>` increments `EnumerationCount` exactly once per `GetEnumerator`.
- [ ] After building the LINQ pipeline, `EnumerationCount == 0` (deferred execution proven).
- [ ] After `pipeline.ToList()`, `EnumerationCount == 1` (materialization triggered a single pass).
- [ ] After `pipeline.Count()`, `EnumerationCount` increases (multiple enumeration proven).
- [ ] After switching to the materialized `List<int>`, the counter stops growing on `.Sum()`/`.Max()`.
- [ ] The `dotnet run` output matches the spec: the range, the windows, the first five naturals, the squares of even numbers.
- [ ] The code has bilingual RU+EN comments at key points.
- [ ] At least one C# 12 collection expression and at least one `using` statement are used.

#### Hints (no direct answer)

- Recall the lesson analogy: "the librarian hands out a card". `GetEnumerator` is the librarian, `Enumerator` is the card. Where in your code is "handing out the card", and where is "flipping through"?
- For `MoveNext`, look at how `Enumerable.Range` checks bounds via `(uint)`. Ask yourself why an `int` comparison `_index + 1 < _count` would be insufficient when `_count < 0`.
- For `SlidingWindows`, ask: what happens if you return `buffer` directly and the caller then mutates `buffer[0]`? How does `yield return` reuse the same object instance across iterations?
- For the enumeration counter, remember that `GetEnumerator()` is called not only by `foreach` but by every LINQ operator, as well as `ToList`/`ToArray`/`Count`. Each is a separate pass.
- To "fix" multiple enumeration, ask: where does the result live after `ToList()`? In a `List<T>`, where `Count` is a property and `Sum`/`Max` walk already-materialized data without re-touching the source.

#### Reference solution walk-through

```csharp
using System.Collections;
using System.Collections.Generic;
using System.Linq;

namespace M06L06.Homework;

// EN: Hand-written IEnumerable<int>/IEnumerator<int> — like Enumerable.Range, but built by hand.
// RU: Ручная реализация — как Enumerable.Range, но своими руками.
public readonly struct IntRange : IEnumerable<int>
{
    private readonly int _start;
    private readonly int _count;

    public IntRange(int start, int count)
    {
        if (count < 0)
            throw new ArgumentOutOfRangeException(nameof(count), "Count must be non-negative.");
        _start = start;
        _count = count;
    }

    public int Count => _count;

    // EN: Typed GetEnumerator — the boxing-free path used by foreach over IntRange.
    public Enumerator GetEnumerator() => new Enumerator(_start, _count);

    IEnumerator<int> IEnumerable<int>.GetEnumerator() => GetEnumerator(); // allocation possible here
    IEnumerator IEnumerable.GetEnumerator() => GetEnumerator();

    // EN: struct enumerator: not heap-allocated, efficient in hot loops.
    public struct Enumerator : IEnumerator<int>
    {
        private readonly int _start;
        private readonly int _count;
        private int _index;   // -1 before the first MoveNext
        private int _current; // valid only after a successful MoveNext

        internal Enumerator(int start, int count)
        {
            _start = start;
            _count = count;
            _index = -1;
            _current = 0;
        }

        public int Current => _current;
        object IEnumerator.Current => _current;

        public bool MoveNext()
        {
            // EN: Unsigned check covers both overrun and a negative _count.
            if ((uint)(_index + 1) < (uint)_count)
            {
                _index++;
                _current = _start + _index;
                return true;
            }
            return false;
        }

        // EN: Reset rewinds to "before the first element"; foreach never calls it.
        public void Reset() => _index = -1;

        // EN: No unmanaged resources, but Dispose is mandatory per the IEnumerator contract.
        public void Dispose() { }
    }
}

// EN: Lazy generators via yield return — the compiler builds a state-machine class.
public static class WindowSequence
{
    // EN: Infinite sequence; safe only with Take().
    public static IEnumerable<int> Naturals(int start = 1)
    {
        while (true) yield return start++;
    }

    // EN: Fixed-size sliding window over any source.
    public static IEnumerable<IReadOnlyList<T>> SlidingWindows<T>(
        this IEnumerable<T> source, int windowSize)
    {
        if (source is null) throw new ArgumentNullException(nameof(source));
        if (windowSize <= 0) throw new ArgumentOutOfRangeException(nameof(windowSize));

        using IEnumerator<T> e = source.GetEnumerator();
        var buffer = new List<T>(windowSize);

        while (e.MoveNext())
        {
            buffer.Add(e.Current);
            if (buffer.Count > windowSize)
                buffer.RemoveAt(0);

            if (buffer.Count == windowSize)
                // EN: A copy so external mutation cannot corrupt the inner buffer.
                yield return buffer.ToArray();
        }
    }
}

// EN: Diagnostic wrapper: tallies real passes over the source.
public sealed class CountingEnumerable<T> : IEnumerable<T>
{
    private readonly IEnumerable<T> _source;
    public int EnumerationCount { get; private set; }

    public CountingEnumerable(IEnumerable<T> source)
        => _source = source ?? throw new ArgumentNullException(nameof(source));

    public IEnumerator<T> GetEnumerator()
    {
        EnumerationCount++;             // every GetEnumerator = one pass
        return _source.GetEnumerator();
    }

    IEnumerator IEnumerable.GetEnumerator() => GetEnumerator();
}

public static class Program
{
    public static void Main()
    {
        // --- Scenario A: manual iteration vs foreach ---
        var range = new IntRange(10, 5); // 10,11,12,13,14
        foreach (int n in range) System.Console.Write(n + " ");
        System.Console.WriteLine();

        // Manual foreach expansion — exactly what the compiler does.
        using var e = range.GetEnumerator();
        while (e.MoveNext()) System.Console.Write(e.Current + " ");
        System.Console.WriteLine();

        // --- Scenario B: yield return — windows and an infinite sequence ---
        int[] data = [1, 2, 3, 4, 5]; // C# 12 collection expression
        foreach (var w in data.SlidingWindows(3))
            System.Console.WriteLine(string.Join(",", w)); // 1,2,3 / 2,3,4 / 3,4,5

        List<int> firstFive = WindowSequence.Naturals().Take(5).ToList();
        System.Console.WriteLine(string.Join(",", firstFive)); // 1,2,3,4,5

        // --- Scenario C: deferred execution and multiple enumeration ---
        var counting = new CountingEnumerable<int>(new[] { 1, 2, 3, 4, 5, 6 });
        IEnumerable<int> pipeline = counting.Where(x => x % 2 == 0).Select(x => x * x);

        System.Console.WriteLine($"Before materialization: {counting.EnumerationCount}"); // 0
        List<int> squares = pipeline.ToList(); // trigger -> 1
        System.Console.WriteLine($"After ToList: {counting.EnumerationCount} => {string.Join(",", squares)}");
        _ = pipeline.Count(); // another pass -> 2
        System.Console.WriteLine($"After Count(): {counting.EnumerationCount}");

        // --- Scenario D: materialize once, never touch the source again ---
        List<int> materialized = pipeline.ToList(); // one more pass -> 3
        int sum = materialized.Sum();
        int max = materialized.Max();
        System.Console.WriteLine($"sum={sum}, max={max}, enumerations={counting.EnumerationCount}");
    }
}
```

Line-by-line walk-through. `IntRange` is a `readonly struct`, which gives two things: immutability (fields cannot be changed after construction) and the ability to be passed as an `in` parameter without defensive copying. The constructor checks `count < 0` and throws `ArgumentOutOfRangeException` — the same validation as in `Enumerable.Range`. The typed `GetEnumerator()` returns `Enumerator` by value: this is exactly what `foreach` over `IntRange` uses directly (without casting to the interface), and that is why iteration does not allocate. The two explicit interface implementations delegate to the same method, but when called through the interface the `Enumerator` gets boxed into `IEnumerator<int>` — that is the price of LINQ and generic-code compatibility.

In the nested `Enumerator`, the field `_index = -1` means the state "before the first `MoveNext`" — exactly as the `IEnumerator` contract requires (`Current` is not valid before the first successful `MoveNext`). The condition `(uint)(_index + 1) < (uint)_count` is the unsigned bounds-check idiom from the BCL sources: when `_count < 0`, the right-hand side becomes a very large number after the cast to `uint`, the comparison is false, and the method returns `false` immediately without yielding a single element. `Reset` returns `_index = -1`; it is implemented honestly, but as the lesson notes, `foreach` never calls it and many enumerators throw `NotSupportedException`. `Dispose` is empty — a correct implementation of the `IDisposable` contract that `IEnumerator` inherits; `foreach` always calls it in `finally`.

`WindowSequence.Naturals` is an infinite generator: `while (true) yield return start++`. The compiler turns this method into a hidden state-machine class implementing both `IEnumerable<int>` and `IEnumerator<int>`; values are produced one at a time, on demand. That is why `Naturals().Take(5).ToList()` neither loops forever nor exhausts memory — exactly five values are taken. `SlidingWindows` walks the source through `using IEnumerator<T> e = source.GetEnumerator()` (correct `Dispose` at the end), maintains a buffer of width `windowSize`, and via `yield return buffer.ToArray()` yields a copy of each window. The copy is critical: had we returned the `buffer` itself, every window would reference the same mutable list — a classic lazy-sequence bug.

`CountingEnumerable<T>` increments `EnumerationCount` in every `GetEnumerator()`. This is the visible "deferredness sensor": building the `pipeline` with `Where`/`Select` never calls `GetEnumerator`, so the counter stays `0`. `ToList()` calls it once — counter `1`. A repeated `pipeline.Count()` calls it again — counter `2`. This is the proof of the three key lesson theses: deferred operators build the pipeline, eager operators trigger it, and every trigger is a separate pass over the source. Scenario D materializes the list once and then works with it: `.Sum()` and `.Max()` walk the `List<int>`, the source is never touched again, and the counter is frozen.

#### Going deeper (bonus)

1. **Thread-safe counter.** Make `EnumerationCount` atomic via `Interlocked.Increment` and add a comment explaining why a plain `++` would race in a multithreaded environment. Consider whether `IEnumerable<T>` itself makes sense in a multithreaded context (hint: the contract does not guarantee thread safety).
2. **Source mutation demo.** Use a `List<int>` as the source inside `CountingEnumerable`, build a pipeline, materialize it, then mutate the underlying `List<int>` and materialize again. Explain why the result changed, and how `ToList()` "freezes" a snapshot.
3. **`struct` vs `class` enumerator benchmark.** Write a benchmark (e.g. `System.Diagnostics.Stopwatch` over a 10-million-iteration loop) comparing iteration over `IntRange` directly and via `IEnumerable<int>`. Explain the difference through allocations.
4. **A real `Dispose` enumerator.** Implement an enumerator reading lines from a `TextReader` with a meaningful `Dispose` that closes the reader. Show that `foreach` releases the resource correctly even when an exception is thrown inside the loop body.

---

#### Чек-лист сдачи / Submission checklist
- [ ] Проект `M06L06.Homework` собирается под `net8.0` без ошибок и предупреждений (RU).
- [ ] `IntRange` реализует `IEnumerable<int>` с ручным `struct`-перечислителем (RU).
- [ ] `WindowSequence` использует `yield return` и возвращает копии окон (RU).
- [ ] `CountingEnumerable<T>` считает перечисления; продемонстрированы `0`, `1`, `2` (RU).
- [ ] Вывод `dotnet run` соответствует спецификации (RU).
- [ ] The `M06L06.Homework` project builds under `net8.0` with no errors or warnings (EN).
- [ ] `IntRange` implements `IEnumerable<int>` with a hand-written `struct` enumerator (EN).
- [ ] `WindowSequence` uses `yield return` and returns window copies (EN).
- [ ] `CountingEnumerable<T>` counts enumerations; `0`, `1`, `2` are demonstrated (EN).
- [ ] The `dotnet run` output matches the specification (EN).

#### Ресурсы / Resources
- [Microsoft Learn — IEnumerable<T>](https://learn.microsoft.com/dotnet/api/system.collections.generic.ienumerable-1)
- [Microsoft Learn — IEnumerator<T>](https://learn.microsoft.com/dotnet/api/system.collections.generic.ienumerator-1)
- [Microsoft Learn — yield (C# reference)](https://learn.microsoft.com/dotnet/csharp/language-reference/statements/keywords/yield)
- [Microsoft Learn — Iterators (C#)](https://learn.microsoft.com/dotnet/csharp/iterators)
