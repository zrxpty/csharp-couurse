[⬆ К модулю M06](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M06-L07: yield return, ленивые последовательности / yield return, lazy sequences

**Модуль / Module:** M06
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

`yield return` — это механизм C#, который позволяет создавать *ленивые* (отложенные) последовательности. Когда вы пишете метод с `yield return`, компилятор не выполняет тело метода сразу от начала до конца. Вместо этого он превращает метод в специальный объект — итератор, который возвращает элементы по одному, по требованию.

Представьте ресторан, где повар готовит блюда не все сразу, а только когда клиент заказывает следующее. В обычном методе вы бы наполнили весь поднос едой и отдали его целиком (`List<T>` с заранее вычисленными значениями). С `yield return` повар ждёт заказа: «дай следующее блюдо» — готовит одно, отдаёт, снова ждёт. Это и есть ленивость: вычисления происходят ровно тогда, когда они нужны, и ровно в нужном объёме.

**Как это работает внутри.** Компилятор C# превращает метод с `yield` в конечный автомат (state machine). Генерируется скрытый класс, реализующий `IEnumerable<T>` и `IEnumerator<T>`. Каждый вызов `MoveNext()` переводит автомат в следующее состояние, а поле `Current` получает очередное значение. Локальные переменные метода «замораживаются» между вызовами — они становятся полями этого скрытого класса. Поэтому итератор помнит, где остановился.

**Ленивость и выгоды.** Главное преимущество — экономия памяти и времени. Бесконечная последовательность `Fibonacci()` не может поместиться в `List<long>`, но через `yield return` её можно перебирать сколько угодно: каждый следующий элемент вычисляется на лету. Ленивость также позволяет строить конвейеры: `numbers.Where(...).Select(...).Take(10)`. Каждый этап — итератор, и данные проходят через цепочку по одному элементу, без создания промежуточных массивов.

**`yield break`.** Оператор `yield break` досрочно завершает итерацию — аналог `return` для обычного метода, но для итератора. Удобен, когда нужно остановиться по условию, не дойдя до конца. Достижение конца метода тоже неявно делает `yield break`.

**Бесконечные последовательности.** С `yield return` можно писать `while (true)` и никогда не возвращаться — потребитель сам решит, когда остановиться через `Take(n)` или `First(...)`. Это безопасно, потому что ничего не вычисляется заранее.

**Производительность и подводные камни.**
- Каждый `yield`-метод создаёт объект итератора (аллокация на куче) — для горячих путей это может быть дорого.
- Многократный перебор `IEnumerable` заново запускает всю логику: дорогой источник данных лучше материализовать через `.ToList()`.
- `yield` нельзя использовать в методах с `ref`/`out`-параметрами, в `unsafe`-блоках, в `finally`-блоках `try`, а также в лямбдах и анонимных методах.
- Побочные эффекты в итераторе выполняются лениво — если вызвать метод, но не перебирать его, ничего не произойдёт. Это частая причина неожиданного поведения.
- Итераторы *не поддерживают* многопоточный перебор: один и тот же объект `IEnumerable` нельзя безопасно использовать из нескольких потоков одновременно.

Используйте `yield return` там, где важна ленивость: потоки данных, генераторы, парсинг больших файлов по строкам, конвейеры LINQ-to-objects. Там, где нужна готовая коллекция для многократного доступа или измерения размера, — материализуйте в `List`/`Array`.

#### Theory (EN)

`yield return` is a C# feature for building *lazy* (deferred) sequences. When you write a method with `yield return`, the compiler does not run the method body from start to finish in one pass. Instead, it transforms the method into a special iterator object that produces elements one at a time, on demand.

Imagine a restaurant where the chef prepares dishes one by one, only when a customer asks for the next one. A regular method would be like loading an entire tray with food and handing it over all at once (a `List<T>` with precomputed values). With `yield return`, the chef waits: "give me the next dish" — cooks one, hands it over, waits again. That is laziness: work happens exactly when it is needed, and exactly in the amount requested.

**How it works inside.** The C# compiler turns a `yield` method into a state machine. A hidden class implementing `IEnumerable<T>` and `IEnumerator<T>` is generated. Each call to `MoveNext()` advances the machine to the next state, and the `Current` field receives the next value. The method's local variables are "frozen" between calls — they become fields of the hidden class. That is why the iterator remembers where it stopped.

**Laziness and benefits.** The main advantage is saving memory and time. An infinite `Fibonacci()` sequence cannot fit in a `List<long>`, but with `yield return` you can iterate it as long as you want: each next element is computed on the fly. Laziness also enables pipelines: `numbers.Where(...).Select(...).Take(10)`. Each stage is an iterator, and data flows through the chain one element at a time, without intermediate arrays.

**`yield break`.** The `yield break` statement ends iteration early — it is the iterator's equivalent of a plain `return`. Useful when you need to stop by a condition before reaching the end. Reaching the end of the method also implicitly performs a `yield break`.

**Infinite sequences.** With `yield return` you can write `while (true)` and never return — the consumer decides when to stop via `Take(n)` or `First(...)`. This is safe because nothing is computed in advance.

**Performance and pitfalls.**
- Each `yield` method allocates an iterator object on the heap — this can be costly on hot paths.
- Iterating an `IEnumerable` multiple times re-runs the whole logic: for an expensive data source, materialize with `.ToList()`.
- `yield` cannot be used in methods with `ref`/`out` parameters, inside `unsafe` blocks, inside a `finally` block of `try`, or inside lambdas and anonymous methods.
- Side effects in an iterator run lazily — if you call the method but never iterate it, nothing happens. This is a common source of surprises.
- Iterators do *not* support multithreaded enumeration: a single `IEnumerable` object cannot be safely consumed from multiple threads at once.

Use `yield return` where laziness matters: data streams, generators, line-by-line parsing of large files, LINQ-to-objects pipelines. Where you need a ready collection for repeated access or for measuring size — materialize into a `List` or `Array`.

#### Пример кода / Code Example

```csharp
using System;
using System.Collections.Generic;
using System.Linq;

// Бесконечная ленивая последовательность Фибоначчи
// Infinite lazy Fibonacci sequence
static IEnumerable<long> Fibonacci()
{
    long prev = 0;
    long curr = 1;
    while (true)                 // никогда не завершается сама / never ends on its own
    {
        yield return prev;       // отдаём текущее / yield current value
        (prev, curr) = (curr, prev + curr);   // шаг вперёд / advance
    }
}

// Ленивое чтение строк файла по одной — память не переполняется
// Lazy line-by-line file reading — memory stays flat
static IEnumerable<string> ReadLinesLazy(string path)
{
    foreach (var line in System.IO.File.ReadLines(path))
    {
        if (string.IsNullOrWhiteSpace(line))
            yield break;         // стоп на первой пустой строке / stop at first blank line
        yield return line.Trim();
    }
}

// Генератор диапазона с шагом / Range generator with a step
static IEnumerable<int> Range(int from, int to, int step)
{
    for (int i = from; step > 0 ? i <= to : i >= to; i += step)
        yield return i;
}

class Program
{
    static void Main()
    {
        // Берём первые 10 чисел Фибоначчи — бесконечность безопасна
        // Take first 10 Fibonacci numbers — infinity is safe
        var first10 = Fibonacci().Take(10).ToArray();
        Console.WriteLine(string.Join(", ", first10));
        // 0, 1, 1, 2, 3, 5, 8, 13, 21, 34

        // Ленивый конвейер: ничего не материализуется до ToArray
        // Lazy pipeline: nothing materializes until ToArray
        var evens = Range(1, 100, 1)
                    .Where(n => n % 2 == 0)
                    .Select(n => n * n)
                    .Take(5);
        Console.WriteLine(string.Join(", ", evens));
        // 4, 16, 36, 64, 100

        // Демонстрация ленивости: побочный эффект виден только при переборе
        // Laziness demo: side effect appears only during enumeration
        var query = Range(1, 5, 1).Select(n => { Console.WriteLine($"computing {n}"); return n * 10; });
        Console.WriteLine("query построен, но ничего не вычислено / query built, nothing computed yet");
        foreach (var v in query) { /* перебор запускает вычисление / iteration triggers computation */ }
    }
}
```

#### Best Practices
- Используйте `yield return` для ленивых потоков и конвейеров; материализуйте (`ToList`/`ToArray`) там, где нужен многократный доступ или подсчёт размера. (RU)
- Избегайте побочных эффектов внутри итераторов — ленивое исполнение делает их непредсказуемыми во времени. (RU)
- Не перебирайте один и тот же дорогой `IEnumerable` дважды; кэшируйте результат в коллекцию. (RU)
- Use `yield return` for lazy streams and pipelines; materialize (`ToList`/`ToArray`) where repeated access or sizing is needed. (EN)
- Avoid side effects inside iterators — deferred execution makes their timing unpredictable. (EN)
- Do not enumerate the same expensive `IEnumerable` twice; cache the result into a collection. (EN)

#### Частые ошибки / Common Mistakes
- Несколько переборов одного `IEnumerable`, который ходит в БД/файл → повторное выполнение всей работы. Кэшируйте через `.ToList()`. (RU)
- Побочные эффекты в итераторе, рассчитанные на немедленный запуск → ничего не выполнится до перебора. Разделяйте построение и исполнение. (RU)
- Использование `yield` в лямбде или методе с `ref`/`out` → ошибка компиляции. Вынесите логику в отдельный метод. (RU)
- Многопоточный перебор одного итератора → гонки данных. Создавайте новый `IEnumerable` на поток или материализуйте. (RU)
- Multiple enumerations of a DB/file-backed `IEnumerable` → work runs again each time. Cache via `.ToList()`. (EN)
- Side effects in an iterator expected to fire immediately → nothing runs until enumeration. Separate building from execution. (EN)
- Using `yield` in a lambda or a `ref`/`out` method → compile error. Extract the logic into its own method. (EN)
- Multithreaded enumeration of one iterator → data races. Create a fresh `IEnumerable` per thread or materialize. (EN)

#### Чек-лист самопроверки / Self-check Checklist
- [ ] Я понимаю, что `yield return` создаёт итератор, а не выполняет тело сразу. (RU)
- [ ] Я могу объяснить, почему бесконечная последовательность безопасна с `yield`. (RU)
- [ ] Я знаю, когда нужен `yield break` и чем он отличается от обычного `return`. (RU)
- [ ] Я помню про аллокацию итератора и риск повторного перебора. (RU)
- [ ] Я знаю ограничения: лямбды, `ref/out`, `unsafe`, `finally`, многопоточность. (RU)
- [ ] I understand that `yield return` creates an iterator and does not run the body eagerly. (EN)
- [ ] I can explain why an infinite sequence is safe with `yield`. (EN)
- [ ] I know when `yield break` is needed and how it differs from a plain `return`. (EN)
- [ ] I remember the iterator allocation cost and the multiple-enumeration risk. (EN)
- [ ] I know the restrictions: lambdas, `ref/out`, `unsafe`, `finally`, threading. (EN)

#### Ресурсы / Resources
- [Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/language-reference/statements/yield](https://learn.microsoft.com/dotnet/csharp/language-reference/statements/yield)

---

[⬆ К модулю M06](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
