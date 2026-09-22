---
[← Предыдущий: M03-L05](lesson-M03-L05-methods.md) | [⬆ К модулю M03](../README.md) | [Следующий: M03-L07 →](lesson-M03-L07-overloading-scope.md)
---

### Урок M03-L06: Параметры: ref, out, in, params, значения по умолчанию / Parameters: ref, out, in, params, defaults

**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

В C# параметры по умолчанию передаются **по значению**. Это значит, что метод получает копию аргумента, а не саму переменную. Представьте, что вы даёте другу фотокопию документа: друг может рисовать на копии сколько угодно — оригинал остаётся нетронутым. Для значимых типов (int, double, struct) копируются сами данные, для ссылочных типов (class, массив) копируется ссылка, но оба указывают на один объект, поэтому изменения полей объекта видны вызывающей стороне (однако переназначение параметра на новый объект наружу не видно).

Ключевое слово **ref** передаёт переменную по ссылке. Аналогия: вы передаёте не копию, а сам оригинал документа с разрешением в нём писать. Переменная должна быть инициализирована до вызова, а в сигнатуре и в месте вызова пишется `ref`. Это двусторонний канал: метод может как читать, так и изменять значение, и изменения видны снаружи. Полезно для методов вроде `Swap` или `int.TryParse`-подобных сценариев, когда нужно вернуть несколько значений без кортежа.

**out** похож на `ref`, но с другими обязательствами. Переменная **не обязана быть инициализирована** до вызова — наоборот, метод **обязан** присвоить ей значение до возврата. Это «почтовый ящик»: вы даёте пустой конверт, метод кладёт туда результат. С C# 7 можно объявлять переменную прямо в вызове: `int.TryParse(s, out int result)`. Подходит, когда метод должен вернуть несколько значений или сигнализировать об успехе вместе с результатом.

**in** — это `ref readonly`: передача по ссылке без права изменения. Аналогия: вы даёте документ для чтения, но без права редактирования. Компилятор гарантирует, что метод не меняет значение, а вызывающая сторона экономит на копировании больших структур. Отличный выбор для больших readonly-структур, чтобы избежать дорогих копий без риска мутации. Переменная должна быть инициализирована до вызова.

**params** позволяет методу принимать переменное число аргументов одного типа. Аналогия: список покупок — можете передать 1, 3 или 10 пунктов, а метод соберёт их в массив. В сигнатуре: `void Log(params string[] messages)`. Вызов: `Log("a", "b", "c")`. Только один параметр `params` и только последний. Можно также передать массив напрямую.

**Значения по умолчанию** (optional parameters): параметру задаётся значение по умолчанию `void Print(int x = 0, string label = "n/a")`. Можно вызывать `Print()`, `Print(5)`, `Print(5, "count")`. Значения должны быть константами времени компиляции. Внимание: если изменить значение по умолчанию в библиотеке, перекомпилированные клиенты увидят новое значение, а старые бинарники — старое, закодированное в месте вызова.

**Именованные аргументы** позволяют передавать аргументы по имени, в любом порядке: `Print(label: "count", x: 5)`. Особенно полезны с параметрами по умолчанию, чтобы пропустить средние. Сочетание `params`, дефолтов и именованных аргументов даёт гибкие API, но злоупотребление снижает читаемость.

Выбирайте форму по намерению: нужно вернуть значение — `out`; нужно читать и менять существующее — `ref`; нужно только читать большую структуру без копии — `in`; переменное число однотипных аргументов — `params`; опциональные аргументы — значения по умолчанию + именованные аргументы.

#### Theory (EN)

In C#, parameters are passed **by value** by default. The method receives a copy of the argument, not the variable itself. Imagine handing a friend a photocopy of a document: they can scribble on the copy all they want — the original stays untouched. For value types (int, double, struct) the data itself is copied; for reference types (class, array) the reference is copied, but both point at the same object, so mutations to the object's fields are visible to the caller (while reassigning the parameter to a new object is not).

The **ref** keyword passes a variable by reference. Analogy: instead of a copy, you hand over the original document and allow writing on it. The variable must be initialized before the call, and `ref` appears both in the signature and at the call site. This is a two-way channel: the method can read and modify the value, and changes are visible outside. It is useful for methods like `Swap` or `int.TryParse`-like scenarios when you need to return several values without a tuple.

**out** resembles `ref` but with different obligations. The variable does **not** need to be initialized before the call — in fact, the method **must** assign it before returning. Think of it as a mailbox: you hand over an empty envelope and the method drops the result inside. Since C# 7 you can declare the variable inline: `int.TryParse(s, out int result)`. Use it when the method must return several values or signal success together with a result.

**in** is `ref readonly`: pass by reference without permission to modify. Analogy: you lend the document for reading only, no edits. The compiler guarantees the method will not change the value, and the caller saves the cost of copying a large struct. A great choice for large readonly structs to avoid expensive copies without mutation risk. The variable must be initialized before the call.

**params** lets a method accept a variable number of arguments of one type. Analogy: a shopping list — you can pass 1, 3, or 10 items, and the method gathers them into an array. Signature: `void Log(params string[] messages)`. Call: `Log("a", "b", "c")`. Only one `params` parameter is allowed and it must be the last. You can also pass an array directly.

**Default values** (optional parameters): a parameter gets a default `void Print(int x = 0, string label = "n/a")`. You can call `Print()`, `Print(5)`, `Print(5, "count")`. Values must be compile-time constants. Caveat: if you change a default in a library, recompiled clients see the new value, but old binaries keep the old value baked into their call sites.

**Named arguments** let you pass arguments by name, in any order: `Print(label: "count", x: 5)`. Especially handy with defaults to skip middle parameters. Combining `params`, defaults, and named arguments yields flexible APIs, but overuse hurts readability.

Pick the form by intent: need to return a value — `out`; need to read and mutate an existing one — `ref`; need to read a large struct without copying — `in`; variable number of same-type arguments — `params`; optional arguments — defaults plus named arguments.

#### Пример кода / Code Example

```csharp
// C# 12 / .NET 8 — рабочий демонстрационный файл / working demo file
// Запуск: dotnet run  /  Run: dotnet run

using System;
using System.Collections.Generic;

// 1) Передача по значению / Pass by value
//    int копируется; переназначение внутри метода не видно снаружи.
//    int is copied; reassignment inside the method is not visible outside.
int x = 10;
void IncrementByValue(int n) => n += 100;        // меняет копию / mutates a copy
IncrementByValue(x);
Console.WriteLine($"By value: {x}");             // 10 — оригинал не тронут / original untouched

// 2) ref — ссылка, двусторонний канал, переменная инициализирована.
//    ref — by reference, two-way, variable must be initialized.
void IncrementByRef(ref int n) => n += 100;
IncrementByRef(ref x);
Console.WriteLine($"By ref: {x}");               // 110 — изменение видно / change is visible

// 3) out — метод обязан присвоить; переменную можно объявить в вызове.
//    out — method must assign; variable can be declared inline.
bool TryParseInt(string s, out int result)
{
    result = 0;                                   // обязаны присвоить до возврата / must assign before return
    if (int.TryParse(s, out var v)) { result = v; return true; }
    return false;
}
if (TryParseInt("42", out int parsed))
    Console.WriteLine($"Parsed: {parsed}");       // 42

// 4) in — ref readonly, без копии и без изменения.
//    in — ref readonly, no copy, no mutation.
//    Большая readonly-структура / large readonly struct.
readonly struct BigPoint(double x, double y)
{
    public double X { get; } = x;
    public double Y { get; } = y;
}

double Distance(in BigPoint a, in BigPoint b)     // без копий, без прав на запись / no copies, read-only
{
    // a.X = 0; // Ошибка компиляции / compile error: cannot modify in-parameter
    double dx = a.X - b.X;
    double dy = a.Y - b.Y;
    return Math.Sqrt(dx * dx + dy * dy);
}

var p1 = new BigPoint(0, 0);
var p2 = new BigPoint(3, 4);
Console.WriteLine($"Distance: {Distance(in p1, in p2)}");   // 5

// 5) params — переменное число аргументов / variable number of arguments.
int Sum(params int[] numbers)
{
    var total = 0;
    foreach (var n in numbers) total += n;
    return total;
}
Console.WriteLine($"Sum(1,2,3): {Sum(1, 2, 3)}");          // 6
Console.WriteLine($"Sum(array): {Sum([10, 20, 30])}");     // 60 — коллекция-выражение / collection expr

// 6) Значения по умолчанию + именованные аргументы / defaults + named arguments.
void PrintReport(int page = 1, string title = "Untitled", bool landscape = false)
    => Console.WriteLine($"Report '{title}', page {page}, landscape={landscape}");

PrintReport();                                              // все по умолчанию / all defaults
PrintReport(5);                                             // page=5
PrintReport(title: "Sales Q4");                             // именованный / named
PrintReport(landscape: true, title: "Wide", page: 2);       // любой порядок / any order

// 7) Практический сценарий: обмен значений и множественный возврат.
//    Practical scenario: swap and multiple return values.
void Swap(ref int a, ref int b) => (a, b) = (b, a);        // деконструкция C# 12 / tuple deconstruction

int left = 1, right = 2;
Swap(ref left, ref right);
Console.WriteLine($"Swap: left={left}, right={right}");     // left=2, right=1

// Множественный возврат через out без кортежа / multiple return via out, no tuple.
void Analyze(string text, out int words, out int chars)
{
    words = text.Split(' ', StringSplitOptions.RemoveEmptyEntries | StringSplitOptions.TrimEntries).Length;
    chars = text.Length;
}
Analyze("C# parameters are powerful", out int w, out int c);
Console.WriteLine($"Words={w}, Chars={c}");
```

#### Best Practices

- Предпочитайте `out` возврат кортежа или нескольких `ref` только когда это улучшает читаемость; в новом коде часто лучше `(int, bool)`-кортеж или запись. (RU)
- Используйте `in` для больших readonly-структур (от ~16–24 байт), чтобы избежать копий; для маленьких и изменяемых структур это лишнее. (RU)
- Не злоупотребляйте параметрами по умолчанию в публичных API библиотек — изменение значения ломает бинарную совместимость. (RU)
- Prefer `out` over multiple `ref` or hand-rolled patterns only when it reads clearly; in modern code a tuple `(int, bool)` or a record often communicates intent better. (EN)
- Use `in` for large readonly structs (roughly 16–24 bytes and up) to skip copies; for small or mutable structs it just adds noise. (EN)
- Avoid default parameters in public library APIs — changing a default breaks binary compatibility for already-compiled callers. (EN)

#### Частые ошибки / Common Mistakes

- Забыли `ref`/`out` в месте вызова → компилятор требует модификатор и в сигнатуре, и при вызове; добавьте его в обе стороны. (RU)
- Передача неинициализированной переменной в `ref` → инициализируйте переменную до вызова; для неинициализированных используйте `out`. (RU)
- Попытка изменить `in`-параметр → `in` доступен только для чтения; если нужно менять, используйте `ref`. (RU)
- `params` не последним параметром → `params` обязан быть последним в списке параметров. (RU)
- Изменение значения по умолчанию в библиотеке и перекомпиляция → клиенты держат старое значение в месте вызова; версонируйте API или используйте перегрузки. (RU)
- Forgetting `ref`/`out` at the call site → the modifier must appear both in the signature and at the call site; add it on both sides. (EN)
- Passing an uninitialized variable to `ref` → initialize it before the call; use `out` when the variable is not yet initialized. (EN)
- Trying to modify an `in` parameter → `in` is read-only; switch to `ref` when mutation is required. (EN)
- Placing `params` anywhere but last → `params` must be the final parameter in the list. (EN)
- Changing a default in a library and recompiling → callers keep the old value baked into their call sites; version the API or use overloads instead. (EN)

#### Чек-лист самопроверки / Self-check Checklist

- [ ] Могу объяснить разницу между передачей по значению, `ref`, `out` и `in`. (RU)
- [ ] Знаю, когда переменная должна быть инициализирована до вызова (`ref`, `in`) и когда метод обязан присвоить значение (`out`). (RU)
- [ ] Умею использовать `params` для переменного числа аргументов и помню, что он последний. (RU)
- [ ] Применяю значения по умолчанию и именованные аргументы, понимаю риск бинарной совместимости. (RU)
- [ ] Пишу корректные модификаторы и в сигнатуре, и в месте вызова. (RU)
- [ ] I can explain the difference between pass-by-value, `ref`, `out`, and `in`. (EN)
- [ ] I know when a variable must be initialized before the call (`ref`, `in`) and when the method must assign it (`out`). (EN)
- [ ] I can use `params` for a variable number of arguments and remember it must be last. (EN)
- [ ] I apply default values and named arguments and understand the binary-compatibility risk. (EN)
- [ ] I write the correct modifiers in both the signature and the call site. (EN)

#### Ресурсы / Resources

- Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/language-reference/keywords/ref — Ключевое слово ref (передача по ссылке) / ref keyword (pass by reference)
- Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/language-reference/keywords/method-parameters — Параметры методов (ref, out, in, params, значения по умолчанию) / Method parameters (ref, out, in, params, defaults)
- Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/programming-guide/classes-and-structs/named-and-optional-arguments — Именованные и необязательные аргументы / Named and optional arguments

---
[← Предыдущий: M03-L05](lesson-M03-L05-methods.md) | [⬆ К модулю M03](../README.md) | [Следующий: M03-L07 →](lesson-M03-L07-overloading-scope.md)
---
