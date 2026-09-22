---
[← Предыдущий: M03-L07](lesson-M03-L07-overloading-scope.md) | [⬆ К модулю M03](../README.md) | [Следующий: README →](../README.md)
---

### Урок M03-L08: Рекурсия и Tail-оптимизация (обзор) / Recursion and tail-optimization (overview)

**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

Рекурсия — это приём, при котором метод вызывает сам себя для решения более мелкой версии той же задачи. В C# это обычный вызов метода: ничто не мешает функции `Factorial` снова вызвать `Factorial`, если у компилятора есть корректная сигнатура. Красота рекурсии в том, что она повторяет структуру задачи: если задача естественно разлагается на «то же, но меньше», код получается коротким и читаемым.

Любая корректная рекурсия состоит из двух частей. **Базовый случай (base case)** — условие остановки, где метод возвращает ответ без нового вызова: факториал от 0 равен 1, пустое дерево не имеет дочерних узлов. **Рекурсивный случай (recursive case)** сводит задачу к более близкой к базовой: `n! = n * (n-1)!`. Если базовый случай отсутствует или недостижим, метод будет вызывать себя бесконечно — пока не кончится память.

Каждый вызов метода занимает **фрейм в стеке вызовов (call stack)**: туда попадают аргументы, локальные переменные и адрес возврата. При `Factorial(5)` стек растёт: `Factorial(5)` ждёт `Factorial(4)`, тот — `Factorial(3)`, и так до `Factorial(0)`. Только потом значения «сворачиваются» обратно. Глубина стека ограничена (порядка сотен тысяч фреймов для простых методов), поэтому для очень глубоких рекурсий CLR выбрасывает `StackOverflowException` — фатальное, неперехватываемое (обычно) исключение, которое роняет процесс. Это не «ошибка в логике», которую можно поймать в `catch`; это физическое переполнение стека.

Классические примеры — факториал и числа Фибоначчи. Факториал линейный по глубине (`n` фреймов), Фибоначчи «наивный» (`Fib(n) = Fib(n-1) + Fib(n-2)`) экспоненциальный по времени и линейный по глубине — но с огромным количеством повторных вычислений. Оба примера хорошо иллюстрируют механику, но на практике их лучше считать итеративно.

Когда рекурсия уместна? Когда структура данных сама рекурсивна: деревья (DOM, AST, файловая система), графы с обходом в глубину, парсинг вложенных конструкций. Обход бинарного дерева «посети левое поддерево, затем корень, затем правое» буквально повторяет определение дерева. Альтернатива — явный стек (`Stack<T>`) в куче — даёт тот же порядок, но не рискует переполнением стека CLR и часто читается ничуть не хуже.

Итеративная альтернатива почти всегда существует: цикл `for`/`while` с аккумулятором заменяет хвостовую рекурсию, а `Stack<T>` — глубокий обход. Для факториала и Фибоначчи итеративная версия короче, быстрее и безопаснее. Правило простое: линейная рекурсия по числу — почти всегда антипаттерн; рекурсия по дереву — часто оправдана.

**Tail-оптимизация (TCO, tail-call optimization)** — это техника компилятора, при которой вызов, стоящий последним в методе (после него нет работы), не создаёт новый фрейм, а переиспользует текущий. Теоретически это превращает рекурсию в цикл и убирает риск переполнения. **Важно: C# и .NET CLR не гарантируют TCO.** JIT-компилятор в отдельных случаях может применить хвостовой вызов (особенно в F#, или при `tail.` IL-префиксе), но в обычном C#-коде полагаться на это нельзя. Если глубина может быть большой — переписывайте на цикл или явный стек явно. «Хвостовая форма» кода в C# — это стиль, а не гарантия производительности.

#### Theory (EN)

Recursion is a technique where a method calls itself to solve a smaller version of the same problem. In C# this is just an ordinary method call: nothing prevents `Factorial` from calling `Factorial` again, provided the signature is correct. The beauty of recursion is that it mirrors the structure of the task: when a problem naturally decomposes into “the same thing, but smaller,” the code stays short and readable.

Every correct recursion has two parts. The **base case** is the stopping condition where the method returns an answer without a further call: factorial of 0 is 1, an empty tree has no children. The **recursive case** reduces the task toward the base case: `n! = n * (n-1)`. If the base case is missing or unreachable, the method calls itself forever — until memory runs out.

Each method call occupies a **frame on the call stack**: arguments, local variables, and the return address live there. With `Factorial(5)` the stack grows: `Factorial(5)` waits for `Factorial(4)`, which waits for `Factorial(3)`, down to `Factorial(0)`. Only then do the values roll back up. Stack depth is bounded (on the order of hundreds of thousands of frames for simple methods), so for very deep recursion the CLR throws `StackOverflowException` — a fatal, generally uncatchable exception that crashes the process. It is not a logic error you can trap in `catch`; it is a physical overflow of the stack.

Classic examples are factorial and Fibonacci numbers. Factorial is linear in depth (`n` frames); naive Fibonacci (`Fib(n) = Fib(n-1) + Fib(n-2)`) is exponential in time and linear in depth — with a huge amount of repeated work. Both illustrate the mechanics well, but in production you usually compute them iteratively.

When is recursion appropriate? When the data structure is itself recursive: trees (DOM, AST, file system), graphs traversed depth-first, parsing nested constructs. A binary-tree traversal “visit left subtree, then root, then right” literally restates the definition of a tree. The alternative — an explicit `Stack<T>` on the heap — gives the same order without risking CLR stack overflow and often reads just as clearly.

An iterative alternative almost always exists: a `for`/`while` loop with an accumulator replaces tail recursion, and a `Stack<T>` replaces deep traversal. For factorial and Fibonacci the iterative version is shorter, faster, and safer. The rule of thumb: linear recursion on a count is almost always an anti-pattern; recursion on a tree is often justified.

**Tail-call optimization (TCO)** is a compiler technique where a call that is the last action in a method (with no work after it) does not create a new frame but reuses the current one. In theory this turns recursion into a loop and removes overflow risk. **Crucially: C# and the .NET CLR do not guarantee TCO.** The JIT may apply a tail call in specific cases (notably in F#, or with the `tail.` IL prefix), but you cannot rely on it in ordinary C# code. If depth may be large, rewrite to a loop or an explicit stack deliberately. Writing “tail form” in C# is a style, not a performance guarantee.

#### Пример кода / Code Example
```csharp
// C# 12 / .NET 8 — top-level statements
// Демонстрация рекурсии, её рисков и итеративной альтернативы.
// Demonstrates recursion, its risks, and the iterative alternative.

using System;
using System.Collections.Generic;

// 1. Наивная (не хвостовая) рекурсия — факториал.
//    Naive (non-tail) recursion — factorial.
//    Каждый вызов ждёт результата следующего: глубина стека = n.
//    Each call waits for the next result: stack depth = n.
int FactorialRecursive(int n) =>
    n <= 0 ? 1 : n * FactorialRecursive(n - 1);

// 2. Хвостовая форма — аккумулятор передаётся вперёд.
//    Tail form — an accumulator is passed forward.
//    В C# это НЕ гарантирует TCO, но стиль полезен для чтения.
//    C# does NOT guarantee TCO here, but the style is readable.
int FactorialTail(int n, int acc = 1) =>
    n <= 0 ? acc : FactorialTail(n - 1, n * acc);

// 3. Итеративная версия — безопасна по стеку, обычно быстрее.
//    Iterative version — stack-safe, usually faster.
int FactorialIterative(int n)
{
    int result = 1;
    for (int i = 2; i <= n; i++)
        result *= i;
    return result;
}

// 4. Наивный Фибоначчи — экспоненциальное время, иллюстрация механики.
//    Naive Fibonacci — exponential time, illustrates mechanics.
long FibRecursive(long n) =>
    n < 2 ? n : FibRecursive(n - 1) + FibRecursive(n - 2);

// 5. Итеративный Фибоначчи — O(n), без риска переполнения стека.
//    Iterative Fibonacci — O(n), no stack-overflow risk.
long FibIterative(long n)
{
    if (n < 2) return n;
    long prev = 0, curr = 1;
    for (long i = 2; i <= n; i++)
        (prev, curr) = (curr, prev + curr); // pattern: tuple deconstruction
    return curr;
}

// 6. Уместная рекурсия — обход бинарного дерева.
//    Appropriate recursion — binary-tree traversal.
//    Глубина = высота дерева, что обычно безопасно.
//    Depth = tree height, usually safe.
var tree = new Node(1,
    new Node(2, new Node(4), null),
    new Node(3, null, new Node(5)));

List<int> InOrderRecursive(Node? node, List<int> acc)
{
    if (node is null) return acc;          // базовый случай / base case
    InOrderRecursive(node.Left, acc);      // левое поддерево / left subtree
    acc.Add(node.Value);                   // корень / root
    InOrderRecursive(node.Right, acc);     // правое поддерево / right subtree
    return acc;
}

// 7. Итеративный обход с явным стеком — для очень глубоких деревьев.
//    Iterative traversal with an explicit stack — for very deep trees.
List<int> InOrderIterative(Node? root)
{
    var result = new List<int>();
    var stack = new Stack<Node>();
    var current = root;
    while (current is not null || stack.Count > 0)
    {
        while (current is not null)
        {
            stack.Push(current);
            current = current.Left;
        }
        current = stack.Pop();
        result.Add(current.Value);
        current = current.Right;
    }
    return result;
}

// Демонстрация / Demo
Console.WriteLine($"5! (recursive) = {FactorialRecursive(5)}");   // 120
Console.WriteLine($"5! (tail)      = {FactorialTail(5)}");         // 120
Console.WriteLine($"5! (iterative) = {FactorialIterative(5)}");   // 120
Console.WriteLine($"Fib(10) iter   = {FibIterative(10)}");        // 55
Console.WriteLine($"InOrder rec    = {string.Join(", ", InOrderRecursive(tree, new()))}");
Console.WriteLine($"InOrder iter   = {string.Join(", ", InOrderIterative(tree))}");

// ⚠️ НЕ делайте так в продакшене — StackOverflowException роняет процесс.
// ⚠️ Do NOT do this in production — StackOverflowException crashes the process.
// try { FactorialRecursive(int.MaxValue); } catch (StackOverflowException) { }
// — перехватить всё равно не получится / you cannot catch it anyway.

// Вспомогательный тип дерева / Helper tree type
sealed record Node(int Value, Node? Left = null, Node? Right = null);
```

#### Best Practices
- Предпочитайте итеративную версию для линейной рекурсии по числу (факториал, Фибоначчи) — она безопасна по стеку и быстрее. / Prefer the iterative version for linear numeric recursion (factorial, Fibonacci) — it is stack-safe and faster.
- Используйте рекурсию там, где структура данных рекурсивна по природе: деревья, графы, вложенные конструкции. / Use recursion where the data structure is naturally recursive: trees, graphs, nested constructs.
- Всегда проверяйте достижимость базового случая и его корректность до написания рекурсивного шага. / Always verify the base case is reachable and correct before writing the recursive step.
- В C# не полагайтесь на tail-call optimization — если глубина может быть большой, явно переписывайте на цикл или `Stack<T>`. / In C#, do not rely on tail-call optimization — if depth may be large, explicitly rewrite to a loop or `Stack<T>`.
- Ограничивайте максимальную глубину рекурсии входным инвариантом (например, высотой дерева ≤ N), чтобы гарантировать отсутствие переполнения. / Bound the maximum recursion depth by an input invariant (e.g. tree height ≤ N) to guarantee no overflow.
- Для глубокой обработки деревьев в продакшене используйте явный стек в куче, а не неявный стек вызовов. / For deep tree processing in production, use an explicit heap stack rather than the implicit call stack.

#### Частые ошибки / Common Mistakes
- Забыт или недостижим базовый случай → бесконечная рекурсия и `StackOverflowException`. Как избежать: первым делом проверяйте условие остановки и убеждайтесь, что каждый шаг приближает к нему. / Missing or unreachable base case → infinite recursion and `StackOverflowException`. Avoid: check the stopping condition first and ensure each step moves toward it.
- Линейная рекурсия по большому `n` (например, `FactorialRecursive(1_000_000)`) → переполнение стека. Как избежать: переписывайте на цикл с аккумулятором. / Linear recursion over a large `n` (e.g. `FactorialRecursive(1_000_000)`) → stack overflow. Avoid: rewrite as a loop with an accumulator.
- Наивный Фибоначчи для `n ≥ 40` → экспоненциальное время. Как избежать: используйте итеративный вариант с двумя переменными или мемоизацию. / Naive Fibonacci for `n ≥ 40` → exponential time. Avoid: use the iterative two-variable version or memoization.
- Попытка `catch (StackOverflowException)` для восстановления → обычно неперехватываемо, процесс падает. Как избежать: не доводите до переполнения; проектируйте с запасом по глубине. / Trying `catch (StackOverflowException)` to recover → usually uncatchable, process crashes. Avoid: never reach the overflow; design with depth headroom.
- Ожидание, что «хвостовая форма» в C# даст TCO и уберёт переполнение → ложная гарантия. Как избежать: тестируйте на реальной глубине и при необходимости переходите на явный стек. / Expecting “tail form” in C# to give TCO and remove overflow → false guarantee. Avoid: test at real depth and switch to an explicit stack if needed.
- Проверка `n == 0` вместо `n <= 0` для факториала при отрицательном входе → бесконечный спуск. Как избежать: используйте инклюзивное условие и валидируйте вход. / Checking `n == 0` instead of `n <= 0` for factorial with negative input → infinite descent. Avoid: use an inclusive condition and validate input.

#### Чек-лист самопроверки / Self-check Checklist
- [ ] В каждом рекурсивном методе есть корректный, достижимый базовый случай. / Each recursive method has a correct, reachable base case.
- [ ] Я доказал, что рекурсивный шаг приближает задачу к базовому случаю. / I have shown the recursive step moves the task toward the base case.
- [ ] Максимальная глубина рекурсии ограничена входом и не грозит переполнением стека. / The maximum recursion depth is bounded by the input and poses no stack-overflow risk.
- [ ] Для линейной рекурсии по числу выбран итеративный вариант. / For linear numeric recursion, an iterative variant was chosen.
- [ ] Я не полагаюсь на TCO в C# для безопасности стека. / I do not rely on TCO in C# for stack safety.
- [ ] Глубокий обход деревьев в продакшене использует явный `Stack<T>`. / Deep tree traversal in production uses an explicit `Stack<T>`.
- [ ] Код протестирован на граничных и больших входах без `StackOverflowException`. / The code was tested on boundary and large inputs without `StackOverflowException`.

#### Ресурсы / Resources
- Microsoft Learn — Методы (рекурсия) / Methods (recursion) — https://learn.microsoft.com/dotnet/csharp/methods#recursion
- Microsoft Learn — StackOverflowException (API) / StackOverflowException (API) — https://learn.microsoft.com/dotnet/api/system.stackoverflowexception

---
[← Предыдущий: M03-L07](lesson-M03-L07-overloading-scope.md) | [⬆ К модулю M03](../README.md) | [Следующий: README →](../README.md)
