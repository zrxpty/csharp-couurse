---
[← К уроку M03-L08](lesson-M03-L08-recursion.md) | [⬆ К модулю M03](../README.md) | [Следующее ДЗ →]()
---

### Домашнее задание M03-L08: Рекурсия и Tail-оптимизация (обзор) / Homework M03-L08: Recursion and tail-optimization (overview)

**Урок / Lesson:** M03-L08
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 3/5
**Цель / Goal:** (RU) Научиться осознанно применять рекурсию в C# 12 / .NET 8: строить корректные базовый и рекурсивный случаи, оценивать глубину стека, переписывать линейную рекурсию в итеративную форму с аккумулятором, реализовать хвостовую форму (понимая, что TCO в C# не гарантируется), и выбирать между рекурсивным обходом дерева и обходом с явным `Stack<T>`. (EN) Learn to apply recursion deliberately in C# 12 / .NET 8: build correct base and recursive cases, reason about stack depth, rewrite linear recursion into an iterative accumulator form, implement tail form (understanding that TCO is not guaranteed in C#), and choose between recursive tree traversal and traversal with an explicit `Stack<T>`.

#### Связь с уроком / Connection to the lesson
(RU) ДЗ напрямую закрепляет все ключевые темы урока: механику фреймов стека и `StackOverflowException`, наивный и итеративный факториал/Фибоначчи, хвостовую форму как стиль без гарантии TCO, уместную рекурсию по дереву и итеративный обход через `Stack<T>`, а также частые ошибки — забытым базовый случай, проверку `n == 0` вместо `n <= 0`, доверие к TCO и попытку поймать `StackOverflowException`.)
(EN) The homework directly reinforces every key topic of the lesson: call-stack frames and `StackOverflowException`, naive vs iterative factorial/Fibonacci, tail form as a style without a TCO guarantee, appropriate tree recursion and iterative traversal via `Stack<T>`, and the common mistakes — a missing base case, checking `n == 0` instead of `n <= 0`, trusting TCO, and trying to catch `StackOverflowException`.)

---

## Постановка на русском / Russian statement

#### Контекст и мотивация

В уроке M03-L08 вы увидели, что рекурсия в C# — это обычный вызов метода, и что у неё есть две грани. С одной стороны, она изящно повторяет структуру задачи: обход бинарного дерева «левое поддерево — корень — правое» буквально пересказывает определение дерева, а парсинг вложенных конструкций часто пишется в десяток строк. С другой стороны, каждый вызов занимает фрейм в стеке вызовов, глубина стека ограничена, и при превышении CLR выбрасывает `StackOverflowException` — фатальное, обычно неперехватываемое исключение, которое роняет процесс. Урок также показал тонкий, но критичный нюанс: «хвостовая форма» с аккумулятором в C# — это стиль написания кода, а не гарантия tail-call optimization. JIT-компилятор в общем случае не обязан превращать хвостовой вызов в переход, поэтому полагаться на TCO для безопасности стека нельзя.

Это ДЗ построено так, чтобы вы прожили обе грани руками. Вы реализуете один и тот же набор вычислений тремя способами — наивная рекурсия, хвостовая форма и итеративная версия — и убедитесь, что только итеративная версия безопасна при сколь угодно большом входе. Затем вы построите небольшое бинарное дерево и реализуете его обход двумя способами: естественной рекурсией (глубина = высота дерева, обычно безопасно) и итеративно с явным `Stack<T>` (безопасно всегда). Наконец, вы добавите «сторож» глубины — защитный счётчик, который бросает обычное, перехватываемое исключение до того, как рекурсия приблизится к пределу стека. Цель — выработать рефлекс: линейная рекурсия по числу почти всегда антипаттерн; рекурсия по дереву часто оправдана; TCO в C# — не опора, а повод переписать на цикл.

#### Что нужно сделать (пошагово)

1. Создайте проект консольного приложения. Выполните в терминале:
   ```bash
   dotnet new console -n RecursionLab -o RecursionLab --framework net8.0
   cd RecursionLab
   dotnet new sln && dotnet sln add RecursionLab.csproj
   ```
   Откройте `Program.cs` и удалите шаблонный код — вы будете писать на top-level statements, как в уроке.

2. Реализуйте факториал **тремя способами** в одном файле:
   - `FactorialRecursive(int n)` — наивная рекурсия `n * FactorialRecursive(n - 1)`;
   - `FactorialTail(int n, int acc = 1)` — хвостовая форма с аккумулятором;
   - `FactorialIterative(int n)` — цикл `for` с аккумулятором.
   Условие остановки во всех трёх — `n <= 0` (инклюзивное, чтобы отрицательный вход не ушёл в бесконечный спуск). Возвращаемый тип — `int` (переполнение `int` для больших `n` в данном задании допустимо, но отметьте это в комментарии).

3. Реализуйте Фибоначчи **двумя способами** и добавьте счётчик вызовов:
   - `FibRecursive(long n)` — наивная рекурсия `Fib(n-1) + Fib(n-2)`;
   - статическое поле `long _fibCallCount` — инкрементируется при каждом входе в `FibRecursive`;
   - `FibIterative(long n)` — цикл с двумя переменными и деконструкцией кортежа `(prev, curr) = (curr, prev + curr)` (как в уроке).
   В `Main` вызовите `FibRecursive(20)`, обнулите счётчик, и выведите его значение — вы увидите, сколько раз метод вызвал сам себя, что иллюстрирует экспоненциальный рост.

4. Определите тип дерева как `sealed record Node(int Value, Node? Left = null, Node? Right = null);` — ровно как в уроке. Соберите тестовое дерево:
   ```
           1
          / \
         2   3
        /     \
       4       5
   ```

5. Реализуйте обход InOrder **двумя способами**:
   - `List<int> InOrderRecursive(Node? node, List<int> acc)` — естественная рекурсия: левое поддерево, корень, правое поддерево; базовый случай `node is null` возвращает `acc`;
   - `List<int> InOrderIterative(Node? root)` — цикл с явным `Stack<Node>`, повторяющий логику урока: спуск влево с помещением узлов в стек, затем `Pop`, добавление значения, переход вправо.
   Ожидаемый вывод для дерева выше: `4, 2, 1, 3, 5`.

6. Реализуйте «сторож глубины» — метод `int SumTreeDepthBounded(Node? node, int depth, int maxDepth)`, который рекурсивно суммирует значения узлов, но при `depth > maxDepth` бросает `ArgumentException("max depth exceeded")`. Это перехватываемое исключение в отличие от `StackOverflowException`. В `Main` вызовите его с `maxDepth = 1` на тестовом дереве и оберните в `try/catch (ArgumentException)` — убедитесь, что падение контролируемое.

7. В секции демонстрации выведите:
   - `5!` всеми тремя способами (должно быть `120`);
   - `Fib(20)` итеративно и значение счётчика вызовов наивной рекурсии;
   - InOrder обеими реализациями;
   - результат `SumTreeDepthBounded` с `maxDepth = 10` (успех) и с `maxDepth = 1` (перехваченное исключение).

8. Запустите: `dotnet run`. Убедитесь, что вывод совпадает с ожидаемым. В комментарии в начале файла явно напишите, что `FactorialTail` — это хвостовая форма, но TCO в C# не гарантируется, поэтому для большой глубины надо использовать `FactorialIterative`.

9. (Опционально, для закрепления) Закомментированной строкой оставьте «плохой» вызов `FactorialRecursive(1_000_000);` с предупреждением, что он роняет процесс через `StackOverflowException`, который нельзя поймать. Реально не запускайте.

#### Требования к решению

Решение должно компилироваться без предупреждений уровня error под .NET 8 (C# 12) в конфигурации `Release`. Используйте top-level statements, pattern matching (`is null`, `is not null`), деконструкцию кортежа в Фибоначчи, `record` для узла дерева и `Stack<T>` из `System.Collections.Generic`. Возвращаемые типы: факториал — `int`, Фибоначчи — `long` (чтобы `Fib(90)` не переполнил). Все три версии факториала должны возвращать идентичный результат на одинаковых входах; обе версии InOrder — одинаковую последовательность.

Каждый рекурсивный метод обязан иметь корректный, достижимый базовый случай, и каждый рекурсивный шаг должен приближать задачу к базовому. Условие остановки факториала — `n <= 0`, а не `n == 0`, чтобы отрицательный вход не вызвал бесконечный спуск (прямая частая ошибка из урока). В комментариях RU+EN явно отметьте, что хвостовая форма в C# — стиль, а не гарантия TCO, и что безопасность по стеку обеспечивает только итеративная версия. Сторож глубины должен бросать `ArgumentException`, а не рассчитывать на перехват `StackOverflowException`.

Код должен быть протестирован на граничных входах: `Factorial(0)` → `1`, `Fib(0)` → `0`, `Fib(1)` → `1`, InOrder на пустом дереве (`null`) → пустой список. Запрещено полагаться на `try/catch (StackOverflowException)` как на механизм восстановления — такой код в решении появляться не должен; вместо этого глубина ограничивается инвариантом входа или сторожем.

#### Тонкости и подводные камни

- **Забытый или недостижимый базовый случай.** Первым делом в каждом рекурсивном методе пишите условие остановки и убеждайтесь, что шаг приближает к нему. Рекурсия без базового случая — это гарантированное переполнение стека.
- **Проверка `n == 0` вместо `n <= 0`.** Для факториала при отрицательном `n` проверка на равенство нулю даёт бесконечный спуск в отрицательную сторону. Используйте инклюзивное условие и валидируйте вход, если контракт метода запрещает отрицательные значения.
- **`StackOverflowException` неперехватываем.** Не пытайтесь обернуть глубокую рекурсию в `try/catch (StackOverflowException)` — в общем случае это не спасёт процесс. Проектируйте так, чтобы до переполнения не доходило: итеративная версия, явный `Stack<T>` или сторож глубины.
- **TCO в C# не гарантируется.** Хвостовая форма с аккумулятором читается хорошо, но JIT не обязан превращать `FactorialTail(n-1, n*acc)` в переход без нового фрейма. Не используйте её как аргумент безопасности стека — только `FactorialIterative` даёт реальную гарантию.
- **Наивный Фибоначчи экспоненциальен.** `FibRecursive(40)` уже ощутимо медленный, `FibRecursive(50)` — практически бесполезен. Счётчик вызовов в этом задании наглядно покажет масштаб повторных вычислений. Для production — итеративный вариант с двумя переменными или мемоизация.
- **Глубина обхода дерева = его высота.** Для сбалансированного дерева это `O(log n)` и безопасно; для вырожденного в связный список — `O(n)` и рискованно. В production для больших деревьев используйте `InOrderIterative` с явным стеком.
- **Тип счётчика вызовов.** Используйте `long`, а не `int`: для `Fib(40)` счётчик перевалит за миллиард и переполнит `int`.
- **Деконструкция кортежа `(prev, curr) = (curr, prev + curr)`.** Это корректно, потому что правая часть вычисляется целиком до присваивания; не нужно вводить временную переменную.

#### Критерии приёмки

- [ ] Проект `RecursionLab` создан через `dotnet new console --framework net8.0` и собирается без ошибок в `Release`.
- [ ] Используются top-level statements, `record Node`, pattern matching и деконструкция кортежа в Фибоначчи.
- [ ] `FactorialRecursive`, `FactorialTail` и `FactorialIterative` возвращают `120` для `n = 5` и `1` для `n = 0`.
- [ ] Условие остановки факториала — `n <= 0` (инклюзивное), в комментарии отмечена защита от отрицательного входа.
- [ ] `FibRecursive` и `FibIterative` возвращают одинаковые значения для `n` из `0..30`; `FibIterative(20)` = `6765`.
- [ ] Счётчик `_fibCallCount` корректно инкрементируется и его тип — `long`; для `Fib(20)` выводится ожидаемое число вызовов.
- [ ] `InOrderRecursive` и `InOrderIterative` возвращают `4, 2, 1, 3, 5` для тестового дерева и пустой список для `null`.
- [ ] `SumTreeDepthBounded` корректно суммирует дерево при `maxDepth = 10` и бросает `ArgumentException` при `maxDepth = 1`.
- [ ] В `Main` вызов с `maxDepth = 1` обёрнут в `try/catch (ArgumentException)` и выводит сообщение об ошибке, не роняя процесс.
- [ ] В комментарии явно сказано, что TCO в C# не гарантируется и что безопасность стека даёт только итеративная версия.
- [ ] В решении нет `try/catch (StackOverflowException)` как механизма восстановления.
- [ ] Закомментированная строка с `FactorialRecursive(1_000_000)` сопровождается предупреждением о `StackOverflowException`.
- [ ] `dotnet run` выводит все ожидаемые строки в указанном порядке.
- [ ] Код протестирован на граничных входах: `n = 0`, `n = 1`, пустое дерево.

#### Подсказки (без прямого ответа)

- Вспомните структуру `InOrderRecursive` из урока: левое поддерево → `acc.Add(node.Value)` → правое поддерево → `return acc`. Базовый случай — `node is null`.
- Для итеративного InOrder держите в голове два цикла: внутренний спускается влево, толкая узлы в `Stack<Node>`; внешний `Pop`-ает, добавляет значение и переходит вправо. Условие внешнего цикла — `current is not null || stack.Count > 0`.
- Сторож глубины передаёт `depth + 1` в рекурсивные вызовы и проверяет `depth > maxDepth` в начале метода — это аналог базового случая, только по «глубине», а не по структуре.
- Счётчик вызовов инкрементируйте первой же строкой тела `FibRecursive`, до вычисления суммы — так учитывается каждый вход, включая базовые случаи.

#### Эталонное решение (разбор)

```csharp
// C# 12 / .NET 8 — top-level statements
// Рекурсия, хвостовая форма (без гарантии TCO), итеративные альтернативы, обход дерева.
// Recursion, tail form (no TCO guarantee), iterative alternatives, tree traversal.

using System;
using System.Collections.Generic;

// ⚠️ Хвостовая форма в C# НЕ гарантирует tail-call optimization.
// ⚠️ Tail form in C# does NOT guarantee tail-call optimization.
// Безопасность стека даёт только FactorialIterative.
// Only FactorialIterative guarantees stack safety.

// 1. Наивная рекурсия — глубина стека = n. / Naive recursion — stack depth = n.
int FactorialRecursive(int n) =>
    n <= 0 ? 1 : n * FactorialRecursive(n - 1);

// 2. Хвостовая форма с аккумулятором — стиль, но не гарантия TCO.
//    Tail form with accumulator — a style, not a TCO guarantee.
int FactorialTail(int n, int acc = 1) =>
    n <= 0 ? acc : FactorialTail(n - 1, n * acc);

// 3. Итеративная версия — безопасна по стеку. / Iterative — stack-safe.
int FactorialIterative(int n)
{
    int result = 1;
    for (int i = 2; i <= n; i++)
        result *= i;
    return result;
}

// 4. Наивный Фибоначчи с подсчётом вызовов. / Naive Fibonacci with call counting.
long _fibCallCount = 0;
long FibRecursive(long n)
{
    _fibCallCount++;                       // каждый вход учитывается / every entry counted
    return n < 2 ? n : FibRecursive(n - 1) + FibRecursive(n - 2);
}

// 5. Итеративный Фибоначчи — O(n), без переполнения стека.
//    Iterative Fibonacci — O(n), no stack overflow.
long FibIterative(long n)
{
    if (n < 2) return n;
    long prev = 0, curr = 1;
    for (long i = 2; i <= n; i++)
        (prev, curr) = (curr, prev + curr); // деконструкция кортежа / tuple deconstruction
    return curr;
}

// 6. Дерево и два обхода. / Tree and two traversals.
sealed record Node(int Value, Node? Left = null, Node? Right = null);

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

// 7. Сторож глубины — перехватываемое исключение вместо StackOverflowException.
//    Depth guard — a catchable exception instead of StackOverflowException.
int SumTreeDepthBounded(Node? node, int depth, int maxDepth)
{
    if (node is null) return 0;
    if (depth > maxDepth)
        throw new ArgumentException("max depth exceeded", nameof(maxDepth));
    return node.Value
         + SumTreeDepthBounded(node.Left, depth + 1, maxDepth)
         + SumTreeDepthBounded(node.Right, depth + 1, maxDepth);
}

// Демонстрация / Demo
Console.WriteLine($"5! recursive = {FactorialRecursive(5)}");   // 120
Console.WriteLine($"5! tail      = {FactorialTail(5)}");         // 120
Console.WriteLine($"5! iterative = {FactorialIterative(5)}");   // 120
Console.WriteLine($"Fib(20) iter = {FibIterative(20)}");        // 6765
_fibCallCount = 0;
_ = FibRecursive(20);
Console.WriteLine($"Fib(20) naive call count = {_fibCallCount}");
Console.WriteLine($"InOrder rec  = {string.Join(", ", InOrderRecursive(tree, new()))}");
Console.WriteLine($"InOrder iter = {string.Join(", ", InOrderIterative(tree))}");
Console.WriteLine($"Sum(maxDepth=10) = {SumTreeDepthBounded(tree, 0, 10)}"); // 15
try
{
    _ = SumTreeDepthBounded(tree, 0, 1);
}
catch (ArgumentException ex)
{
    Console.WriteLine($"Guard triggered: {ex.Message}");
}

// ⚠️ НЕ запускать — StackOverflowException роняет процесс и не перехватывается.
// ⚠️ Do NOT run — StackOverflowException crashes the process and cannot be caught.
// _ = FactorialRecursive(1_000_000);
```

Разбор по строкам. Три версии факториала иллюстрируют прогрессию урока: `FactorialRecursive` — наивная форма, где каждый вызов ждёт результат следующего (`n * ...`), глубина стека равна `n`; `FactorialTail` — хвостовая форма с аккумулятором, где умножение выполнено до рекурсивного вызова, и вызов стоит последним действием — теоретически это кандидат на TCO, но в C# гарантии нет, что явно отмечено в комментарии; `FactorialIterative` — единственная версия, безопасная по стеку при любом `n`. Условие `n <= 0` вместо `n == 0` защищает от отрицательного входа — это прямая частая ошибка из урока.

`FibRecursive` с глобальным счётчиком `_fibCallCount` типа `long` (не `int`, чтобы не переполниться на больших `n`) делает наглядным экспоненциальный рост числа вызовов: для `Fib(20)` счётчик покажет десятки тысяч входов. `FibIterative` с деконструкцией `(prev, curr) = (curr, prev + curr)` — это O(n) и безопасно по стеку; кортеж вычисляется целиком до присваивания, поэтому временная переменная не нужна.

Обе версии InOrder возвращают одинаковую последовательность `4, 2, 1, 3, 5`. `InOrderRecursive` буквально повторяет определение дерева и безопасна, пока высота дерева невелика. `InOrderIterative` с явным `Stack<Node>` даёт тот же порядок, но не рискует переполнением стека CLR — это рекомендация урока для production.

`SumTreeDepthBounded` — ключевой «сторож»: при `depth > maxDepth` бросается `ArgumentException`, которое **можно** перехватить, в отличие от `StackOverflowException`. Это реализует best practice урока «ограничивайте максимальную глубину рекурсии входным инвариантом». В `Main` демонстрируется и успешный вызов (`maxDepth = 10`), и сработавший сторож (`maxDepth = 1`), обёрнутый в `try/catch (ArgumentException)` — процесс продолжает работать. Закомментированный `FactorialRecursive(1_000_000)` с предупреждением закрепляет, что `StackOverflowException` перехватывать бесполезно.

#### Задания на углубление (бонус)

1. Реализуйте `FibMemoized(long n)` с мемоизацией через `Dictionary<long, long>` и сравните число вызовов с наивной версией для `n = 40`.
2. Реализуйте обход дерева в ширину (BFS) с `Queue<Node>` и сравните порядок с InOrder.
3. Постройте «вырожденное» дерево (каждый узел имеет только правого ребёнка) высотой 10 000 и покажите, что `InOrderRecursive` всё ещё работает, но объясните, при какой высоте появился бы риск. Затем запустите `InOrderIterative` на том же дереве — он безопасен при любой высоте.
4. Добавьте метод `FactorialRecursiveBounded(int n, int maxDepth)`, который бросает `ArgumentException` при превышении глубины, и используйте его как «безопасную» обёртку над наивной рекурсией для недоверенного входа.

---

## Statement in English / Постановка на английском

#### Context & motivation

In lesson M03-L08 you saw that recursion in C# is just an ordinary method call and that it has two faces. On one hand, it elegantly mirrors the structure of a task: a binary-tree traversal "left subtree — root — right subtree" literally restates the definition of a tree, and parsing nested constructs often takes a dozen lines. On the other hand, every call occupies a frame on the call stack, the stack depth is bounded, and when it is exceeded the CLR throws `StackOverflowException` — a fatal, generally uncatchable exception that crashes the process. The lesson also highlighted a subtle but critical point: a "tail form" with an accumulator in C# is a coding style, not a guarantee of tail-call optimization. The JIT compiler is not, in general, obliged to turn a tail call into a jump, so relying on TCO for stack safety is a mistake.

This homework is structured so that you live through both faces with your own hands. You will implement the same set of computations three ways — naive recursion, tail form, and an iterative version — and confirm that only the iterative version is safe for arbitrarily large input. You will then build a small binary tree and implement its traversal in two ways: natural recursion (depth = tree height, usually safe) and an iterative version with an explicit `Stack<T>` (always safe). Finally, you will add a "depth guard" — a counter that throws an ordinary, catchable exception before the recursion gets anywhere near the stack limit. The goal is to build a reflex: linear recursion on a count is almost always an anti-pattern; recursion on a tree is often justified; TCO in C# is not a foundation to stand on, but a reason to rewrite as a loop.

#### What to do step by step

1. Create a console application project. In the terminal run:
   ```bash
   dotnet new console -n RecursionLab -o RecursionLab --framework net8.0
   cd RecursionLab
   dotnet new sln && dotnet sln add RecursionLab.csproj
   ```
   Open `Program.cs` and delete the template code — you will write top-level statements, as in the lesson.

2. Implement factorial **three ways** in a single file:
   - `FactorialRecursive(int n)` — naive recursion `n * FactorialRecursive(n - 1)`;
   - `FactorialTail(int n, int acc = 1)` — tail form with an accumulator;
   - `FactorialIterative(int n)` — a `for` loop with an accumulator.
   The stopping condition in all three is `n <= 0` (inclusive, so a negative input does not descend forever). The return type is `int` (overflow of `int` for large `n` is acceptable for this assignment, but note it in a comment).

3. Implement Fibonacci **two ways** and add a call counter:
   - `FibRecursive(long n)` — naive recursion `Fib(n-1) + Fib(n-2)`;
   - a static field `long _fibCallCount` — incremented on every entry into `FibRecursive`;
   - `FibIterative(long n)` — a loop with two variables and tuple deconstruction `(prev, curr) = (curr, prev + curr)` (as in the lesson).
   In `Main`, call `FibRecursive(20)`, reset the counter, and print its value — you will see how many times the method called itself, illustrating exponential growth.

4. Define the tree type as `sealed record Node(int Value, Node? Left = null, Node? Right = null);` — exactly as in the lesson. Assemble a test tree:
   ```
           1
          / \
         2   3
        /     \
       4       5
   ```

5. Implement InOrder traversal **two ways**:
   - `List<int> InOrderRecursive(Node? node, List<int> acc)` — natural recursion: left subtree, root, right subtree; the base case `node is null` returns `acc`;
   - `List<int> InOrderIterative(Node? root)` — a loop with an explicit `Stack<Node>` mirroring the lesson: descend left while pushing nodes onto the stack, then `Pop`, add the value, and move right.
   Expected output for the tree above: `4, 2, 1, 3, 5`.

6. Implement a "depth guard" — a method `int SumTreeDepthBounded(Node? node, int depth, int maxDepth)` that recursively sums node values but throws `ArgumentException("max depth exceeded")` when `depth > maxDepth`. This is a catchable exception, unlike `StackOverflowException`. In `Main`, call it with `maxDepth = 1` on the test tree and wrap the call in `try/catch (ArgumentException)` — confirm the failure is controlled.

7. In the demo section, print:
   - `5!` by all three methods (must be `120`);
   - `Fib(20)` iteratively and the call-count value of the naive recursion;
   - InOrder by both implementations;
   - the result of `SumTreeDepthBounded` with `maxDepth = 10` (success) and with `maxDepth = 1` (caught exception).

8. Run: `dotnet run`. Verify the output matches expectations. In a comment at the top of the file, state explicitly that `FactorialTail` is a tail form but that TCO is not guaranteed in C#, so for large depth you must use `FactorialIterative`.

9. (Optional, for reinforcement) Leave a commented-out "bad" call `FactorialRecursive(1_000_000);` with a warning that it crashes the process via `StackOverflowException`, which cannot be caught. Do not actually run it.

#### Requirements

The solution must compile without error-level warnings under .NET 8 (C# 12) in `Release` configuration. Use top-level statements, pattern matching (`is null`, `is not null`), tuple deconstruction in Fibonacci, a `record` for the tree node, and `Stack<T>` from `System.Collections.Generic`. Return types: factorial — `int`, Fibonacci — `long` (so `Fib(90)` does not overflow). All three factorial versions must return identical results on identical inputs; both InOrder versions must produce the same sequence.

Every recursive method must have a correct, reachable base case, and every recursive step must move the task toward the base case. The stopping condition for factorial is `n <= 0`, not `n == 0`, so that a negative input does not cause infinite descent (a direct common mistake from the lesson). In RU+EN comments, explicitly note that tail form in C# is a style, not a TCO guarantee, and that stack safety comes only from the iterative version. The depth guard must throw `ArgumentException`, not rely on catching `StackOverflowException`.

The code must be tested on boundary inputs: `Factorial(0)` → `1`, `Fib(0)` → `0`, `Fib(1)` → `1`, InOrder on an empty tree (`null`) → an empty list. Relying on `try/catch (StackOverflowException)` as a recovery mechanism is forbidden — such code must not appear in the solution; instead, depth is bounded by an input invariant or a depth guard.

#### Pitfalls

- **Missing or unreachable base case.** Always write the stopping condition first and confirm the step moves toward it. Recursion without a base case is guaranteed stack overflow.
- **Checking `n == 0` instead of `n <= 0`.** For factorial with a negative `n`, an equality check on zero causes infinite descent into the negatives. Use an inclusive condition and validate input if the contract forbids negatives.
- **`StackOverflowException` is uncatchable.** Do not wrap deep recursion in `try/catch (StackOverflowException)` — in general it will not save the process. Design so overflow is never reached: iterative version, explicit `Stack<T>`, or a depth guard.
- **TCO is not guaranteed in C#.** Tail form with an accumulator reads well, but the JIT is not obliged to turn `FactorialTail(n-1, n*acc)` into a frameless jump. Do not use it as a stack-safety argument — only `FactorialIterative` gives a real guarantee.
- **Naive Fibonacci is exponential.** `FibRecursive(40)` is already slow, `FibRecursive(50)` is practically useless. The call counter in this assignment will make the scale of redundant computation visible. For production, use the iterative two-variable version or memoization.
- **Tree-traversal depth equals tree height.** For a balanced tree this is `O(log n)` and safe; for one degenerated into a linked list it is `O(n)` and risky. In production, for large trees use `InOrderIterative` with an explicit stack.
- **Counter type.** Use `long`, not `int`: for `Fib(40)` the counter exceeds a billion and overflows `int`.
- **Tuple deconstruction `(prev, curr) = (curr, prev + curr)`.** This is correct because the right-hand side is fully evaluated before assignment; no temporary variable is needed.

#### Acceptance criteria

- [ ] The `RecursionLab` project is created via `dotnet new console --framework net8.0` and builds without errors in `Release`.
- [ ] Top-level statements, `record Node`, pattern matching, and tuple deconstruction in Fibonacci are used.
- [ ] `FactorialRecursive`, `FactorialTail`, and `FactorialIterative` return `120` for `n = 5` and `1` for `n = 0`.
- [ ] The factorial stopping condition is `n <= 0` (inclusive), with a comment noting protection against negative input.
- [ ] `FibRecursive` and `FibIterative` return identical values for `n` in `0..30`; `FibIterative(20)` = `6765`.
- [ ] The `_fibCallCount` counter is incremented correctly and its type is `long`; the expected call count for `Fib(20)` is printed.
- [ ] `InOrderRecursive` and `InOrderIterative` return `4, 2, 1, 3, 5` for the test tree and an empty list for `null`.
- [ ] `SumTreeDepthBounded` sums the tree correctly with `maxDepth = 10` and throws `ArgumentException` with `maxDepth = 1`.
- [ ] In `Main`, the `maxDepth = 1` call is wrapped in `try/catch (ArgumentException)` and prints an error message without crashing.
- [ ] A comment explicitly states that TCO is not guaranteed in C# and that only the iterative version is stack-safe.
- [ ] The solution contains no `try/catch (StackOverflowException)` as a recovery mechanism.
- [ ] The commented-out line with `FactorialRecursive(1_000_000)` is accompanied by a warning about `StackOverflowException`.
- [ ] `dotnet run` prints all expected lines in the specified order.
- [ ] The code is tested on boundary inputs: `n = 0`, `n = 1`, an empty tree.

#### Hints (no direct answer)

- Recall the structure of `InOrderRecursive` from the lesson: left subtree → `acc.Add(node.Value)` → right subtree → `return acc`. The base case is `node is null`.
- For iterative InOrder, keep two loops in mind: the inner one descends left, pushing nodes onto a `Stack<Node>`; the outer one `Pop`s, adds the value, and moves right. The outer loop condition is `current is not null || stack.Count > 0`.
- The depth guard passes `depth + 1` into the recursive calls and checks `depth > maxDepth` at the start of the method — it is an analog of the base case, but on "depth" rather than on structure.
- Increment the call counter as the very first line of `FibRecursive`'s body, before computing the sum — this counts every entry, including the base cases.

#### Reference solution walk-through

```csharp
// C# 12 / .NET 8 — top-level statements
// Recursion, tail form (no TCO guarantee), iterative alternatives, tree traversal.

using System;
using System.Collections.Generic;

// ⚠️ Tail form in C# does NOT guarantee tail-call optimization.
// Only FactorialIterative guarantees stack safety.

// 1. Naive recursion — stack depth = n.
int FactorialRecursive(int n) =>
    n <= 0 ? 1 : n * FactorialRecursive(n - 1);

// 2. Tail form with accumulator — a style, not a TCO guarantee.
int FactorialTail(int n, int acc = 1) =>
    n <= 0 ? acc : FactorialTail(n - 1, n * acc);

// 3. Iterative version — stack-safe.
int FactorialIterative(int n)
{
    int result = 1;
    for (int i = 2; i <= n; i++)
        result *= i;
    return result;
}

// 4. Naive Fibonacci with call counting.
long _fibCallCount = 0;
long FibRecursive(long n)
{
    _fibCallCount++;                       // every entry counted
    return n < 2 ? n : FibRecursive(n - 1) + FibRecursive(n - 2);
}

// 5. Iterative Fibonacci — O(n), no stack overflow.
long FibIterative(long n)
{
    if (n < 2) return n;
    long prev = 0, curr = 1;
    for (long i = 2; i <= n; i++)
        (prev, curr) = (curr, prev + curr); // tuple deconstruction
    return curr;
}

// 6. Tree and two traversals.
sealed record Node(int Value, Node? Left = null, Node? Right = null);

var tree = new Node(1,
    new Node(2, new Node(4), null),
    new Node(3, null, new Node(5)));

List<int> InOrderRecursive(Node? node, List<int> acc)
{
    if (node is null) return acc;          // base case
    InOrderRecursive(node.Left, acc);      // left subtree
    acc.Add(node.Value);                   // root
    InOrderRecursive(node.Right, acc);     // right subtree
    return acc;
}

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

// 7. Depth guard — a catchable exception instead of StackOverflowException.
int SumTreeDepthBounded(Node? node, int depth, int maxDepth)
{
    if (node is null) return 0;
    if (depth > maxDepth)
        throw new ArgumentException("max depth exceeded", nameof(maxDepth));
    return node.Value
         + SumTreeDepthBounded(node.Left, depth + 1, maxDepth)
         + SumTreeDepthBounded(node.Right, depth + 1, maxDepth);
}

// Demo
Console.WriteLine($"5! recursive = {FactorialRecursive(5)}");   // 120
Console.WriteLine($"5! tail      = {FactorialTail(5)}");         // 120
Console.WriteLine($"5! iterative = {FactorialIterative(5)}");   // 120
Console.WriteLine($"Fib(20) iter = {FibIterative(20)}");        // 6765
_fibCallCount = 0;
_ = FibRecursive(20);
Console.WriteLine($"Fib(20) naive call count = {_fibCallCount}");
Console.WriteLine($"InOrder rec  = {string.Join(", ", InOrderRecursive(tree, new()))}");
Console.WriteLine($"InOrder iter = {string.Join(", ", InOrderIterative(tree))}");
Console.WriteLine($"Sum(maxDepth=10) = {SumTreeDepthBounded(tree, 0, 10)}"); // 15
try
{
    _ = SumTreeDepthBounded(tree, 0, 1);
}
catch (ArgumentException ex)
{
    Console.WriteLine($"Guard triggered: {ex.Message}");
}

// ⚠️ Do NOT run — StackOverflowException crashes the process and cannot be caught.
// _ = FactorialRecursive(1_000_000);
```

Line-by-line walk-through. The three factorial versions illustrate the lesson's progression: `FactorialRecursive` is the naive form, where each call waits for the next result (`n * ...`) and the stack depth equals `n`; `FactorialTail` is the tail form with an accumulator, where the multiplication is performed before the recursive call and the call is the last action — theoretically a TCO candidate, but C# gives no guarantee, which is stated explicitly in the comment; `FactorialIterative` is the only version that is stack-safe for any `n`. The condition `n <= 0` instead of `n == 0` protects against negative input — a direct common mistake from the lesson.

`FibRecursive` with a global counter `_fibCallCount` of type `long` (not `int`, to avoid overflow at large `n`) makes the exponential growth of call count visible: for `Fib(20)` the counter shows tens of thousands of entries. `FibIterative` with deconstruction `(prev, curr) = (curr, prev + curr)` is O(n) and stack-safe; the tuple is fully evaluated before assignment, so no temporary variable is needed.

Both InOrder versions return the same sequence `4, 2, 1, 3, 5`. `InOrderRecursive` literally restates the definition of a tree and is safe as long as the tree height is modest. `InOrderIterative` with an explicit `Stack<Node>` produces the same order without risking CLR stack overflow — this is the lesson's recommendation for production.

`SumTreeDepthBounded` is the key "guard": when `depth > maxDepth` it throws `ArgumentException`, which **can** be caught, unlike `StackOverflowException`. This implements the lesson's best practice "bound the maximum recursion depth by an input invariant." `Main` demonstrates both a successful call (`maxDepth = 10`) and a triggered guard (`maxDepth = 1`), wrapped in `try/catch (ArgumentException)` — the process keeps running. The commented-out `FactorialRecursive(1_000_000)` with a warning reinforces that catching `StackOverflowException` is futile.

#### Going deeper (bonus)

1. Implement `FibMemoized(long n)` with memoization via `Dictionary<long, long>` and compare the call count with the naive version for `n = 40`.
2. Implement a breadth-first traversal (BFS) of the tree with `Queue<Node>` and compare the order with InOrder.
3. Build a "degenerate" tree (each node has only a right child) of height 10 000 and show that `InOrderRecursive` still works, then explain at what height a risk would appear. Then run `InOrderIterative` on the same tree — it is safe at any height.
4. Add a method `FactorialRecursiveBounded(int n, int maxDepth)` that throws `ArgumentException` when the depth is exceeded, and use it as a "safe" wrapper over naive recursion for untrusted input.

---

#### Чек-лист сдачи / Submission checklist
- [ ] Проект собирается через `dotnet build -c Release` без ошибок. / The project builds via `dotnet build -c Release` without errors.
- [ ] Все три версии факториала возвращают `120` для `5!`. / All three factorial versions return `120` for `5!`.
- [ ] Условие остановки факториала — `n <= 0`, с комментарием о защите от отрицательного входа. / The factorial stopping condition is `n <= 0`, with a comment on negative-input protection.
- [ ] `FibIterative(20)` = `6765`, счётчик вызовов наивной версии выведен и имеет тип `long`. / `FibIterative(20)` = `6765`, the naive call counter is printed and is of type `long`.
- [ ] Оба обхода InOrder возвращают `4, 2, 1, 3, 5`. / Both InOrder traversals return `4, 2, 1, 3, 5`.
- [ ] `SumTreeDepthBounded` бросает перехватываемое `ArgumentException` при `maxDepth = 1`. / `SumTreeDepthBounded` throws a catchable `ArgumentException` at `maxDepth = 1`.
- [ ] В комментарии явно указано, что TCO в C# не гарантируется. / A comment explicitly states that TCO is not guaranteed in C#.
- [ ] В решении нет `try/catch (StackOverflowException)`. / The solution contains no `try/catch (StackOverflowException)`.
- [ ] `dotnet run` выводит все ожидаемые строки. / `dotnet run` prints all expected lines.
- [ ] Код протестирован на граничных входах (`0`, `1`, пустое дерево). / The code is tested on boundary inputs (`0`, `1`, an empty tree).

#### Ресурсы / Resources
- Microsoft Learn — Методы (рекурсия) / Methods (recursion) — https://learn.microsoft.com/dotnet/csharp/methods#recursion
- Microsoft Learn — StackOverflowException (API) / StackOverflowException (API) — https://learn.microsoft.com/dotnet/api/system.stackoverflowexception
- Microsoft Learn — `Stack<T>` — https://learn.microsoft.com/dotnet/api/system.collections.generic.stack-1
- Microsoft Learn — Records (C#) — https://learn.microsoft.com/dotnet/csharp/whats-new/csharp-9#record-types
