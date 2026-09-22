[⬆ К модулю M06](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M06-L05: Queue<T>, Stack<T>, SortedList, LinkedList / Queue<T>, Stack<T>, SortedList, LinkedList

**Модуль / Module:** M06
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

В предыдущих уроках мы работали со списками и словарями — коллекциями, где порядок определяется индексом или ключом. Но в реальных задачах часто важна не позиция элемента, а **порядок его обработки**. Здесь на сцену выходят `Queue<T>` и `Stack<T>` — специальные коллекции с дисциплиной доступа.

**Queue<T> — очередь (FIFO: First In, First Out).** Представьте обычную очередь в кассу: кто первым встал, тот первым обслуживается. Новый элемент добавляется в конец (`Enqueue`), а извлекается тот, что стоит в начале (`Dequeue`). Очередь незаменима в обработке задач: буфер сообщений, планировщик заданий, обход графа в ширину (BFS). Важно: `Dequeue` не только возвращает, но и **удаляет** элемент; если нужно только подсмотреть — используйте `Peek`.

**Stack<T> — стек (LIFO: Last In, First Out).** Аналогия — стопка тарелок: вы берёте ту, что положили последней. Добавление (`Push`) и извлечение (`Pop`) происходят с одного конца — «вершины» стека. Стек — естественная структура для отмены действий (Undo), разбора выражений, обратной рекурсии, обхода дерева в глубину (DFS). Внутри стек и очередь реализованы поверх массивов с циклическими буферами, поэтому `Enqueue`/`Dequeue`/`Push`/`Pop` работают за **амортизированное O(1)**.

**SortedList<TKey, TValue> — отсортированный словарь по ключу.** Это гибрид словаря и массива: пары «ключ-значение» хранятся в массивах, **отсортированных по ключу**. Благодаря этому ключи всегда идут по возрастанию, а перебор через `foreach` возвращает элементы в порядке ключей. Доступ по ключу — **O(log n)** (бинарный поиск), но вставка и удаление — **O(n)** из-за сдвига элементов. Выбирайте `SortedList`, когда коллекция относительно стабильна по размеру и вам важен отсортированный порядок перебора. У него есть «родственник» `SortedDictionary<TKey, TValue>` на основе красно-чёрного дерева: вставка/удаление за O(log n), но больший расход памяти. Правило: данные в основном читаете и перебираете по порядку → `SortedList`; данные часто меняются → `SortedDictionary`.

**LinkedList<T> — двусвязный список.** Каждый элемент (узел `LinkedListNode<T>`) хранит ссылки на предыдущий и следующий узлы. Главные плюсы: вставка и удаление **в известной позиции за O(1)** — нужно лишь иметь ссылку на узел. Минусы: нет индексатора (доступ по индексу — O(n)), больше памяти на ссылки, плохая локальность кэша. `LinkedList<T>` стоит брать, когда нужно частое удаление из середины при наличии итератора, или реализация очереди/дека с дешёвыми операциями на обоих концах (`AddFirst`, `AddLast`, `RemoveFirst`, `RemoveLast`).

**Когда какую коллекцию выбрать?** Задайте себе вопросы:
- Важен порядок поступления, а не позиция? → `Queue<T>` (FIFO) или `Stack<T>` (LIFO).
- Нужен словарь с отсортированными ключами и перебором по порядку? → `SortedList` (редкие изменения) или `SortedDictionary` (частые изменения).
- Нужны частые вставки/удаления в середину и есть ссылка на узел? → `LinkedList<T>`.
- Нужен индексный доступ и редкие вставки в середину? → `List<T>` (из прошлых уроков).
- Нужен быстрый поиск по ключу без порядка? → `Dictionary<TKey, TValue>`.

Помните: в 90% случаев правильный выбор — это `List<T>` или `Dictionary<TKey, TValue>`. Остальные коллекции — инструменты для специфических сценариев. Не берите `LinkedList` «на всякий случай» — чаще всего `List<T>` с индексами работает быстрее благодаря кэшу процессора. Измеряйте производительность реальными данными, а не догадками.

#### Theory (EN)

In previous lessons we worked with lists and dictionaries — collections where order is driven by an index or a key. But in many real tasks what matters is not the position of an element, but **the order in which it is processed**. This is where `Queue<T>` and `Stack<T>` come in — specialized collections with a strict access discipline.

**Queue<T> — a FIFO structure (First In, First Out).** Imagine a line at a ticket counter: whoever arrives first is served first. A new element is added to the back (`Enqueue`), and the element at the front is taken out (`Dequeue`). Queues are indispensable for task processing: message buffers, job schedulers, breadth-first graph traversal (BFS). Note: `Dequeue` both returns and **removes** the element; if you only want to look at the front without removing it, use `Peek`.

**Stack<T> — a LIFO structure (Last In, First Out).** The analogy is a stack of plates: you take the one you placed last. Both adding (`Push`) and removing (`Pop`) happen at one end — the “top” of the stack. Stacks are the natural shape for undo functionality, expression parsing, recursion backtracking, and depth-first tree traversal (DFS). Internally, both stack and queue are built on arrays with circular buffers, so `Enqueue`/`Dequeue`/`Push`/`Pop` run in **amortized O(1)**.

**SortedList<TKey, TValue> — a dictionary sorted by key.** It is a hybrid of a dictionary and an array: key/value pairs are stored in two arrays, **sorted by key**. Because of that, keys always appear in ascending order, and `foreach` iteration yields items in key order. Lookup by key is **O(log n)** (binary search), but insertion and removal are **O(n)** because elements must be shifted. Choose `SortedList` when the collection is fairly stable in size and you need a sorted iteration order. It has a sibling, `SortedDictionary<TKey, TValue>`, built on a red-black tree: insert/remove run in O(log n), but with higher memory overhead. Rule of thumb: you mostly read and iterate in order → `SortedList`; data changes frequently → `SortedDictionary`.

**LinkedList<T> — a doubly linked list.** Each element (a `LinkedListNode<T>`) stores references to the previous and next nodes. The main advantage: insertion and removal **at a known position are O(1)** — you only need a reference to the node. Downsides: no indexer (index-based access is O(n)), more memory for the links, and poor cache locality. Reach for `LinkedList<T>` when you need frequent removals from the middle while holding an iterator, or when you implement a queue/deque with cheap operations on both ends (`AddFirst`, `AddLast`, `RemoveFirst`, `RemoveLast`).

**Which collection should you choose?** Ask yourself:
- Does arrival order matter more than position? → `Queue<T>` (FIFO) or `Stack<T>` (LIFO).
- Do you need a dictionary with sorted keys and ordered iteration? → `SortedList` (rare changes) or `SortedDictionary` (frequent changes).
- Frequent insertions/removals in the middle with a node reference? → `LinkedList<T>`.
- Index access and rare middle insertions? → `List<T>` (from earlier lessons).
- Fast key lookup without ordering? → `Dictionary<TKey, TValue>`.

Remember: in 90% of cases the right answer is `List<T>` or `Dictionary<TKey, TValue>`. The other collections are tools for specific scenarios. Do not pick `LinkedList` “just in case” — `List<T>` with indexes is usually faster thanks to the CPU cache. Measure performance with real data, not guesses.

#### Пример кода / Code Example

```csharp
// C# 12+ / .NET 8+ — примеры Queue, Stack, SortedList, LinkedList
// Examples of Queue, Stack, SortedList, LinkedList

using System;
using System.Collections.Generic;

// === Queue<T> — FIFO: очередь задач / FIFO: task queue ===
var tasks = new Queue<string>();
tasks.Enqueue("Загрузить файл / Load file");        // добавляем в конец / add to back
tasks.Enqueue("Распарсить / Parse");
tasks.Enqueue("Сохранить / Save");

Console.WriteLine("Следующая задача / Next task: " + tasks.Peek()); // только глянуть / peek
while (tasks.Count > 0)
{
    string task = tasks.Dequeue(); // извлекаем из начала / take from front
    Console.WriteLine($"Обрабатываю / Processing: {task}");
}

// === Stack<T> — LIFO: история Undo / LIFO: Undo history ===
var undoStack = new Stack<string>();
undoStack.Push("Ввод текста / Type text");
undoStack.Push("Жирный шрифт / Bold");
undoStack.Push("Курсив / Italic");

// Откатываем в обратном порядке / Roll back in reverse order
while (undoStack.Count > 0)
{
    string action = undoStack.Pop();
    Console.WriteLine($"Отмена / Undo: {action}");
}

// === SortedList<TKey, TValue> — ключи всегда по возрастанию ===
// Keys always kept sorted
var prices = new SortedList<string, decimal>
{
    ["Banana"] = 1.20m,
    ["Apple"] = 2.50m,
    ["Cherry"] = 5.00m,
};

// Перебор идёт в порядке ключей: Apple, Banana, Cherry
// Iteration is in key order
foreach (var kv in prices)
{
    Console.WriteLine($"{kv.Key}: {kv.Value:C}");
}

Console.WriteLine($"Цена Apple (lookup O(log n)) / Price of Apple: {prices["Apple"]:C}");

// === LinkedList<T> — быстрые вставки/удаления с узлом ===
// Fast inserts/removes when you hold a node
var route = new LinkedList<string>();
route.AddLast("Москва / Moscow");
route.AddLast("Казань / Kazan");
route.AddLast("Владивосток / Vladivostok");

LinkedListNode<string> kazanNode = route.Find("Казань / Kazan")!;
route.AddBefore(kazanNode, "Нижний Новгород / Nizhny Novgorod"); // O(1) вставка / O(1) insert

foreach (string city in route)
{
    Console.WriteLine($"Остановка / Stop: {city}");
}

route.Remove("Казань / Kazan"); // удаление по значению O(n), по узлу — O(1)
// Removal by value is O(n); by node reference is O(1)
```

#### Best Practices
- Выбирайте коллекцию под сценарий доступа (FIFO/LIFO/сортировка/частые вставки), а не «на всякий случай». — **RU**
- Предпочитайте `Peek` вместо `Dequeue`/`Pop`, если элемент ещё нужен. — **RU**
- Для отсортированных данных, которые редко меняются, берите `SortedList`; для частых изменений — `SortedDictionary`. — **RU**
- Не используйте `LinkedList<T>`, если нужен индексный доступ — там это O(n). — **RU**
- Измеряйте производительность на реальных данных перед оптимизацией коллекции. — **RU**
- Pick the collection for the access pattern (FIFO/LIFO/sorted/frequent inserts), not “just in case.” — **EN**
- Prefer `Peek` over `Dequeue`/`Pop` when you still need the element. — **EN**
- Use `SortedList` for sorted data that rarely changes; `SortedDictionary` when it changes often. — **EN**
- Avoid `LinkedList<T>` when you need index access — it is O(n) there. — **EN**
- Benchmark with realistic data before optimizing your collection choice. — **EN**

#### Частые ошибки / Common Mistakes
- Использовать `List<T>` с `RemoveAt(0)` как очередь → квадратичная сложность. → Используйте `Queue<T>`, `Dequeue` работает за O(1). — **RU**
- Забывать, что `Dequeue`/`Pop` удаляют элемент. → Если нужен только просмотр, вызывайте `Peek`. — **RU**
- Ожидать O(1) вставку в `SortedList` при больших данных. → Для частых вставок берите `SortedDictionary`. — **RU**
- Перебирать `LinkedList` по индексу через `ElementAt(i)` в цикле → O(n²). → Храните узел или используйте `foreach`. — **RU**
- Использовать `Queue`/`Stack` в многопоточной среде без синхронизации. → Берите `ConcurrentQueue`/`ConcurrentStack`. — **RU**
- Using `List<T>` with `RemoveAt(0)` as a queue → quadratic cost. → Use `Queue<T>`; `Dequeue` is O(1). — **EN**
- Forgetting that `Dequeue`/`Pop` remove the element. → Use `Peek` when you only want to inspect. — **EN**
- Expecting O(1) insertion in `SortedList` on large data. → Use `SortedDictionary` for frequent inserts. — **EN**
- Iterating a `LinkedList` by index via `ElementAt(i)` in a loop → O(n²). → Hold a node or use `foreach`. — **EN**
- Using `Queue`/`Stack` across threads without synchronization. → Use `ConcurrentQueue`/`ConcurrentStack`. — **EN**

#### Чек-лист самопроверки / Self-check Checklist
- [ ] Я могу объяснить разницу между FIFO и LIFO своими словами. — **RU**
- [ ] Я знаю, что `Enqueue`/`Dequeue`/`Push`/`Pop` работают за амортизированное O(1). — **RU**
- [ ] Я выбираю между `SortedList` и `SortedDictionary` по частоте изменений. — **RU**
- [ ] Я понимаю, когда `LinkedList<T>` выгоднее `List<T>`, а когда наоборот. — **RU**
- [ ] Я не забываю про потокобезопасные аналоги в многопоточных сценариях. — **RU**
- [ ] I can explain FIFO vs LIFO in my own words. — **EN**
- [ ] I know `Enqueue`/`Dequeue`/`Push`/`Pop` run in amortized O(1). — **EN**
- [ ] I choose between `SortedList` and `SortedDictionary` based on change frequency. — **EN**
- [ ] I understand when `LinkedList<T>` beats `List<T>` and when it does not. — **EN**
- [ ] I remember thread-safe variants for multi-threaded scenarios. — **EN**

#### Ресурсы / Resources
- [Microsoft Learn — Collections](https://learn.microsoft.com/dotnet/standard/collections/)
- [Microsoft Learn — Queue<T>](https://learn.microsoft.com/dotnet/api/system.collections.generic.queue-1)
- [Microsoft Learn — Stack<T>](https://learn.microsoft.com/dotnet/api/system.collections.generic.stack-1)
- [Microsoft Learn — SortedList<TKey,TValue>](https://learn.microsoft.com/dotnet/api/system.collections.generic.sortedlist-2)
- [Microsoft Learn — LinkedList<T>](https://learn.microsoft.com/dotnet/api/system.collections.generic.linkedlist-1)

---

[⬆ К модулю M06](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
