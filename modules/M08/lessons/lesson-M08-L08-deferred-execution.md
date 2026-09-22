[⬆ К модулю M08](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M08-L08: Отложенное vs немедленное выполнение, ToList/ToArray / Deferred vs immediate execution, ToList/ToArray

**Модуль / Module:** M08
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

Главное, что нужно понять про LINQ: многие операторы **не выполняются** в момент вызова. Они лишь собирают «рецепт» — описание того, что нужно сделать с данными. Реальная работа происходит позже, когда кто-то начинает перебирать результат. Это называется **отложенным выполнением** (deferred execution, lazy evaluation).

**Аналогия.** Представьте, что вы даёте повару заказ на торт. Повар записывает рецепт в блокнот, но не начинает печь. Торт появится только тогда, когда вы придёте и скажете: «Давай ешь». Каждый раз, когда вы приходите снова, повар печёт новый торт с нуля по тому же рецепту.

К отложенным операторам относятся `Where`, `Select`, `OrderBy`, `Skip`, `Take`, `GroupBy`, `Join`, `SelectMany`, `Distinct` и большинство других операторов «преобразования». Они возвращают `IEnumerable<T>`, который на самом деле является итератором: элементы вычисляются по одному, по мере вызова `MoveNext()`. Преимущества очевидны: можно строить цепочки запросов любой сложности без промежуточных коллекций, можно работать с бесконечными последовательностями через `Take`, и ресурсы тратятся только на то, что реально нужно.

Но отложенность порождает **ловушки**. Первая: **многократное перечисление** одной и той же переменной. Если источник — коллекция в памяти, это просто медленно. Но если источник — запрос к базе данных (Entity Framework), `IQueryable<T>` или итератор с побочными эффектами (например, чтение файла построчно), каждое перечисление заново выполняет всю работу, заново ходит в БД, заново открывает файл. Счетчик `foreach` дважды по одному `IEnumerable` — классический баг производительности.

Вторая ловушка: **захват переменных в замыкании**. Поскольку выполнение отложено, лямбды захватывают не значения, а переменные. Если вы измените переменную цикла после построения запроса, но до перечисления — запрос увидит уже новое значение.

Третья: **побочные эффекты внутри `Where`/`Select`**. Так как порядок и количество вызовов предсказать сложно (особенно с `OrderBy`, который может буферизовать), побочные эффекты становятся источником недетерминизма и багов.

Чтобы **заставить немедленное выполнение**, применяют операторы `ToList()`, `ToArray()`, `ToDictionary()`, `ToHashSet()`, а также `Count()`, `Sum()`, `First()`, `Single()`, `Any()`, `Max()`, `Average()`. Они форсируют полное (или до первого совпадения) перечисление прямо в момент вызова и возвращают уже не итератор, а конкретный результат — коллекцию или скаляр. `ToList` создаёт `List<T>` (мутируемая, удобная для дальнейшей работы), `ToArray` — массив (компактнее по памяти, чуть быстрее индексация, но фиксированный размер).

Правило большого пальца: если собираетесь перебирать результат **больше одного раза** — материализуйте через `ToList`/`ToArray` один раз и работайте с коллекцией. Если результат нужен один раз и помещается в память — тоже часто лучше материализовать, чтобы «заморозить» состояние источника на момент запроса (особенно для `IQueryable` к БД). Если же источник большой или бесконечный и нужен потоковый обход — оставляйте отложенным, но перебирайте строго один раз.

Понимание границы между «рецептом» и «тортом» — фундамент грамотной работы с LINQ.

#### Theory (EN)

The single most important fact about LINQ: many operators **do not run** when you call them. They only collect a "recipe" — a description of what should be done with the data. The real work happens later, when somebody starts enumerating the result. This is called **deferred execution** (lazy evaluation).

**Analogy.** You give a chef an order for a cake. The chef writes the recipe in a notebook but does not start baking. The cake appears only when you come back and say "serve it". Every time you come back, the chef bakes a brand-new cake from scratch using the same recipe.

Deferred operators include `Where`, `Select`, `OrderBy`, `Skip`, `Take`, `GroupBy`, `Join`, `SelectMany`, `Distinct` and most other "shaping" operators. They return an `IEnumerable<T>` that is really an iterator: elements are produced one by one as `MoveNext()` is called. The benefits are clear: you can compose arbitrarily long query chains without intermediate collections, you can deal with infinite sequences via `Take`, and resources are spent only on what is actually consumed.

But deferral creates **traps**. The first is **multiple enumeration** of the same variable. If the source is an in-memory collection, this is just slow. If the source is a database query (Entity Framework), an `IQueryable<T>`, or an iterator with side effects (say, reading a file line by line), every enumeration re-runs all the work, re-hits the database, re-opens the file. Iterating the same `IEnumerable` twice with two `foreach` loops is a classic performance bug — and ReSharper/Roslyn analyzers will warn you about it.

The second trap is **captured variables in closures**. Because execution is deferred, lambdas capture variables, not values. If you mutate a loop variable after building the query but before enumerating it, the query sees the mutated value.

The third is **side effects inside `Where`/`Select`**. Because order and number of invocations are hard to predict (especially with `OrderBy`, which may buffer the whole sequence), side effects become a source of nondeterminism and bugs. Keep predicates and projections pure.

To **force immediate execution**, you use `ToList()`, `ToArray()`, `ToDictionary()`, `ToHashSet()`, as well as `Count()`, `Sum()`, `First()`, `Single()`, `Any()`, `Max()`, `Average()`. These force a full (or up-to-first-match) enumeration right at the call site and return not an iterator but a concrete result — a collection or a scalar. `ToList` produces a `List<T>` (mutable, convenient for further work); `ToArray` produces an array (more memory-compact, slightly faster indexed access, but fixed size).

Rule of thumb: if you plan to iterate the result **more than once**, materialize it with `ToList`/`ToArray` once and work with that collection. If the result is needed once and fits in memory, materializing is often still preferable because it "freezes" the source state at query time — especially important for `IQueryable` over a database, where the underlying data may change between enumerations. If the source is huge or infinite and you need streaming, keep it deferred — but enumerate strictly once.

Understanding the border between the "recipe" and the "cake" is the foundation of using LINQ well.

#### Пример кода / Code Example

```csharp
// C# 12 / .NET 8 — Отложенное vs немедленное выполнение
// Deferred vs immediate execution

using System;
using System.Collections.Generic;
using System.Linq;

// Источник-итератор с побочным эффектом: считает, сколько раз создаётся элемент
// Iterator source with a side effect: counts how many times an element is produced
int produced = 0;
IEnumerable<int> Numbers()
{
    for (int i = 1; i <= 5; i++)
    {
        produced++;           // счётчик вызовов / call counter
        yield return i * 10;  // ленивый возврат / lazy yield
    }
}

// 1) ОТЛОЖЕННОЕ ВЫПОЛНЕНИЕ — на момент вызова Select ничего не вычисляется
// 1) DEFERRED — at the moment of Select, nothing is evaluated yet
var query = Numbers().Where(n => n > 20).Select(n => n / 10);
Console.WriteLine($"После построения запроса produced = {produced}"); // 0

// Каждое перечисление заново прогоняет весь конвейер
// Each enumeration re-runs the entire pipeline
foreach (var x in query) Console.Write($"{x} ");      // produced растёт до 5
Console.WriteLine($"\nПосле 1-го foreach produced = {produced}");

foreach (var x in query) Console.Write($"{x} ");      // produced снова растёт до 10
Console.WriteLine($"\nПосле 2-го foreach produced = {produced}");

// 2) НЕМЕДЛЕННОЕ ВЫПОЛНЕНИЕ через ToList — «замораживает» результат
// 2) IMMEDIATE via ToList — materializes and freezes the result
produced = 0;
var materialized = Numbers().Where(n => n > 20).Select(n => n / 10).ToList();
Console.WriteLine($"Сразу после ToList produced = {produced}"); // 5 — уже выполнено

// Повторные переборы materialized НЕ дёргают источник повторно
// Repeated iterations of materialized do NOT touch the source again
foreach (var x in materialized) Console.Write($"{x} ");
foreach (var x in materialized) Console.Write($"{x} ");
Console.WriteLine($"\nПосле двух переборов списка produced = {produced}"); // всё ещё 5

// 3) ToArray — то же немедленное выполнение, но результат — массив
// 3) ToArray — same immediate execution, but result is an array
produced = 0;
int[] array = Numbers().Where(n => n > 20).Select(n => n / 10).ToArray();
Console.WriteLine($"После ToArray produced = {produced}, длина = {array.Length}");

// 4) Скалярные операторы — тоже немедленные
// 4) Scalar operators are also immediate
produced = 0;
int count = Numbers().Count(n => n > 20);          // форсирует перебор / forces enumeration
int sum = Numbers().Sum();                          // ещё один полный перебор / another full pass
Console.WriteLine($"count = {count}, sum = {sum}, produced = {produced}"); // 10

// 5) ЛОВУШКА ЗАМЫКАНИЯ: лямбда захватывает переменную, а не значение
// 5) CLOSURE TRAP: lambda captures the variable, not the value
var actions = new List<Action>();
for (int i = 0; i < 3; i++)
    actions.Add(() => Console.WriteLine(i)); // захват переменной i / captures variable i
foreach (var a in actions) a(); // выведет 3 3 3, а не 0 1 2 / prints 3 3 3, not 0 1 2

// Исправление: копируем во локальную переменную на каждой итерации
// Fix: copy into a local variable on each iteration
var actionsFixed = new List<Action>();
for (int i = 0; i < 3; i++)
{
    int local = i; // своя копия / own copy
    actionsFixed.Add(() => Console.WriteLine(local));
}
foreach (var a in actionsFixed) a(); // 0 1 2

// 6) Практическое правило: материализуй, если перебираешь > 1 раза
// 6) Practical rule: materialize if you enumerate more than once
IEnumerable<int> source = Enumerable.Range(1, 1000);
var filtered = source.Where(n => n % 7 == 0).ToList(); // один расчёт, много использований
Console.WriteLine($"Кратных 7: {filtered.Count}, первый: {filtered[0]}");
```

#### Best Practices

- Материализуйте результат через `ToList`/`ToArray`, если планируете перебирать его больше одного раза или если источник — `IQueryable` к БД.
- Предпочитайте `ToArray`, когда размер известен и коллекция далее не мутируется: меньше накладных расходов по памяти.
- Предпочитайте `ToList`, когда нужен дальнейший `Add`/`Remove` или индексированный доступ с изменениями.
- Делайте предикаты и проекции чистыми функциями — без побочных эффектов и без изменения внешнего состояния.
- Перечисляйте отложенный запрос ровно один раз; если нужен повторный обход — материализуйте его заранее.
- Для `IQueryable` ставьте `ToList`/`ToArray` последним оператором, чтобы «зафиксировать» снимок данных на момент запроса.
- Включайте анализаторы IDE (IDE0060, CA1827 и др.) и предупреждения ReSharper о multiple enumeration.

- Materialize the result with `ToList`/`ToArray` if you plan to iterate it more than once or if the source is an `IQueryable` over a database.
- Prefer `ToArray` when the size is known and the collection will not be mutated afterwards: lower memory overhead.
- Prefer `ToList` when you need further `Add`/`Remove` or mutable indexed access.
- Keep predicates and projections pure functions — no side effects, no mutation of outer state.
- Enumerate a deferred query exactly once; if you need to traverse again, materialize first.
- For `IQueryable`, put `ToList`/`ToArray` as the last operator to freeze a snapshot of the data at query time.
- Enable IDE analyzers (IDE0060, CA1827, etc.) and ReSharper multiple-enumeration warnings.

#### Частые ошибки / Common Mistakes

- Двойной `foreach` по одному `IEnumerable` от дорогого источника → материализуйте через `ToList` один раз и перебирайте коллекцию.
- Изменение переменной цикла после построения, но до перечисления запроса → материализуйте сразу или копируйте значение в локальную переменную внутри лямбды.
- Побочные эффекты внутри `Where`/`Select` (логирование, счётчики, мутация) → Выносите побочные эффекты за пределы запроса; предикаты должны быть чистыми.
- Путаница `IQueryable` и `IEnumerable` при работе с EF → Помните, что операторы над `IQueryable` транслируются в SQL и выполняются на сервере; `AsEnumerable()` переключает на клиентское выполнение.
- Использование `Count()` для проверки «есть ли элементы» → Используйте `Any()` — он останавливается на первом совпадении и не перебирает всю последовательность.
- Вызов `ToList()` «на всякий случай» для огромной последовательности → материализуйте только то, что реально нужно повторно использовать, иначе теряете выгоды потоковой обработки.
- Ожидание, что `OrderBy` выполнится лениво покаскадно → `OrderBy` буферизует всю последовательность; последующие операторы работают уже над отсортированным снимком.

- Two `foreach` loops over the same `IEnumerable` from an expensive source → Materialize once with `ToList` and iterate the collection.
- Mutating a loop variable after building but before enumerating a query → Materialize immediately or copy the value into a local variable inside the lambda.
- Side effects inside `Where`/`Select` (logging, counters, mutation) → Move side effects out of the query; predicates must be pure.
- Mixing up `IQueryable` and `IEnumerable` with EF → Remember that operators on `IQueryable` are translated to SQL and run on the server; `AsEnumerable()` switches to client-side evaluation.
- Using `Count()` to check "are there any elements" → Use `Any()` — it stops at the first match and does not enumerate the whole sequence.
- Calling `ToList()` "just in case" on a huge sequence → Materialize only what you actually reuse, otherwise you lose streaming benefits.
- Expecting `OrderBy` to run lazily element-by-element → `OrderBy` buffers the whole sequence; subsequent operators work on the sorted snapshot.

#### Чек-лист самопроверки / Self-check Checklist

- [ ] Я могу назвать минимум 5 отложенных операторов и 5 немедленных.
- [ ] Я понимаю, что `Where`/`Select` не выполняются в момент вызова.
- [ ] Я знаю, что повторный `foreach` по `IEnumerable` запускает конвейер заново.
- [ ] Я использую `ToList`/`ToArray`, когда результат нужен больше одного раза.
- [ ] Я знаю разницу между `ToList` (мутируемый) и `ToArray` (фиксированный размер).
- [ ] Я использую `Any()` вместо `Count() > 0` для проверки наличия элементов.
- [ ] Я не допускаю побочных эффектов в предикатах и проекциях.
- [ ] Я понимаю ловушку замыкания с переменной цикла и умею её исправлять.
- [ ] Я отличаю клиентское (`IEnumerable`) и серверное (`IQueryable`) выполнение в EF.

- [ ] I can name at least 5 deferred and 5 immediate operators.
- [ ] I understand that `Where`/`Select` do not run at the call site.
- [ ] I know that a second `foreach` over an `IEnumerable` re-runs the pipeline.
- [ ] I use `ToList`/`ToArray` when the result is needed more than once.
- [ ] I know the difference between `ToList` (mutable) and `ToArray` (fixed size).
- [ ] I use `Any()` instead of `Count() > 0` to check for elements.
- [ ] I avoid side effects inside predicates and projections.
- [ ] I understand the closure trap with a loop variable and how to fix it.
- [ ] I distinguish client-side (`IEnumerable`) and server-side (`IQueryable`) execution in EF.

#### Ресурсы / Resources

- [Microsoft Learn — Deferred Execution Example — https://learn.microsoft.com/dotnet/csharp/programming-guide/concepts/linq/deferred-execution-example]

---

[⬆ К модулю M08](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
