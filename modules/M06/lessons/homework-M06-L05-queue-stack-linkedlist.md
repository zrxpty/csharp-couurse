---
[← К уроку M06-L05](lesson-M06-L05-queue-stack-linkedlist.md) | [⬆ К модулю M06](../README.md) | [Следующее ДЗ →](homework-M06-L06-ienumerable-ienumerator.md)
---

### Домашнее задание M06-L05: Queue<T>, Stack<T>, SortedList, LinkedList / Homework M06-L05: Queue<T>, Stack<T>, SortedList, LinkedList

**Урок / Lesson:** M06-L05
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 3/5
**Цель / Goal:** (RU) Научиться осознанно выбирать и применять `Queue<T>`, `Stack<T>`, `SortedList<TKey, TValue>` и `LinkedList<T>` в сценариях, где важна дисциплина доступа (FIFO/LIFO), отсортированный порядок перебора и быстрые вставки/удаления в середину по ссылке на узел. (EN) Learn to consciously pick and use `Queue<T>`, `Stack<T>`, `SortedList<TKey, TValue>` and `LinkedList<T>` in scenarios driven by access discipline (FIFO/LIFO), sorted iteration order, and fast middle insertions/removals through a node reference.

#### Связь с уроком / Connection to the lesson
(RU) Урок ввёл четыре специализированных коллекции и сформулировал правила выбора между ними по шаблону доступа. Это ДЗ закрепляет каждую из них на едином сквозном сценарии — мини-симуляторе печатного офиса — так, чтобы FIFO, LIFO, отсортированный каталог и конвейер с вставками в середину работали вместе, а не изолированно. Особое внимание уделено частым ошибкам из урока: `RemoveAt(0)` вместо очереди, `Dequeue`/`Pop` без `Peek`, ожидание O(1) у `SortedList` и индексация `LinkedList` через `ElementAt`.
(EN) The lesson introduced four specialized collections and framed the selection rules by access pattern. This homework anchors each of them in a single end-to-end scenario — a miniature print-office simulator — so that FIFO, LIFO, a sorted catalog, and a pipeline with middle insertions cooperate instead of being isolated. Special attention is paid to the lesson's common mistakes: `RemoveAt(0)` instead of a queue, `Dequeue`/`Pop` without `Peek`, expecting O(1) from `SortedList`, and indexing a `LinkedList` via `ElementAt`.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация
Вы прототипируете мини-симулятор печатного офиса «PrintFlow». В офисе параллельно живут четыре независимых процесса, и каждый из них естественно ложится на одну из коллекций, изученных в уроке M06-L05. Первая подсистема — приём заявок на печать: заявки поступают в порядке прибытия и должны обслуживаться строго в том же порядке, то есть это классический FIFO-буфер, которому соответствует `Queue<T>`. Вторая подсистема — журнал отмен: если оператор ошибся, он отменяет последнее действие, потом предпоследнее и так далее; это LIFO-история, идеально описываемая `Stack<T>`. Третья подсистема — каталог шаблонов документов: шаблоны должны перечисляться всегда по алфавиту названий, при этом каталог преимущественно читается, а изменяется редко; это типичный кейс для `SortedList<TKey, TValue>`. Четвёртая подсистема — конвейер постобработки: последовательность стадий (водяной знак, сжатие, шифрование, отправка), в которую оператор может динамически вставлять новые стадии перед или после уже известных узлов; здесь нужна `LinkedList<T>` с операциями `AddBefore`/`AddAfter` за O(1).

Ключевая мысль урока, которую должно закрепить задание: правильная коллекция определяется шаблоном доступа, а не «на всякий случай». В 90 % случаев выигрывают `List<T>` и `Dictionary<TKey, TValue>`, но в четырёх перечисленных подсистемах специализированные структуры дают как более честную сложность, так и более читаемый код. Дополнительно вы потренируетесь различать `SortedList` и `SortedDictionary` по частоте изменений, а также поймёте, почему индексация `LinkedList` через `ElementAt(i)` в цикле — это O(n²), и как этого избежать, храня ссылку на узел. Наконец, вы должны помнить о потокобезопасности: `Queue<T>` и `Stack<T>` не синхронизированы, и в многопоточной среде нужны `ConcurrentQueue`/`ConcurrentStack` — это тоже отражено в бонусном задании.

#### Что нужно сделать (пошагово)
1. Создайте консольный проект .NET 8 с именем `PrintFlow`:
   ```
   dotnet new console -n PrintFlow -o PrintFlow --framework net8.0
   cd PrintFlow
   dotnet build
   ```
   Убедитесь, что в `PrintFlow.csproj` указан `<TargetFramework>net8.0</TargetFramework>` и язык C# 12 (он включён по умолчанию для .NET 8). Используйте top-level statements в `Program.cs`.

2. Смоделируйте сущность `PrintJob` (record) с полями `Id` (int), `Title` (string), `Pages` (int). Сгенерируйте несколько заявок коллекцией-инициализатором или выражением коллекции C# 12: `PrintJob[] incoming = [ new(1, "Договор", 12), new(2, "Отчёт", 40), new(3, "Прайс", 4) ];`.

3. Подсистема «Приём заявок». Заведите `Queue<PrintJob> jobs = new(incoming);`. В цикле `while (jobs.Count > 0)` извлекайте заявки через `Dequeue` и печатайте строку вида `[done] #1 «Договор» 12 стр.`. Перед первой же итерацией покажите «следующую» заявку через `Peek`, не удаляя её, и убедитесь, что после `Peek` `Count` не изменился — это проверка понимания разницы между `Peek` и `Dequeue` из урока.

4. Подсистема «Отмены». Заведите `Stack<string> undo = new();` и последовательно `Push` три строковых действия: `"Ввод текста"`, `"Жирный"`, `"Курсив"`. Затем трижды `Pop` и выведите `Undo: …` — порядок должен быть обратным (`Курсив`, `Жирный`, `Ввод текста`). Подсчитайте, сколько раз `Pop` уменьшил `Count`, и выведите итог.

5. Подсистема «Каталог шаблонов». Заведите `SortedList<string, decimal> templates = new() { ["Invoice"] = 0.15m, ["Contract"] = 0.50m, ["Letter"] = 0.05m, ["Report"] = 0.30m };`. Переберите через `foreach` — вывод должен идти строго по алфавиту ключей: `Contract, Invoice, Letter, Report`. Сделайте lookup одного ключа через индексатор (`templates["Letter"]`) и убедитесь, что такой доступ — O(log n) благодаря бинарному поиску.

6. Подсистема «Конвейер». Заведите `LinkedList<string> pipeline = new();` и `AddLast` стадии: `"Watermark"`, `"Compress"`, `"Encrypt"`, `"Send"`. Найдите узел `"Compress"` через `pipeline.Find("Compress")!` и вставьте перед ним `"Grayscale"` через `AddBefore`, а после него — `"Sign"` через `AddAfter`. Выведите итоговую последовательность через `foreach`. Подумайте, почему вставка через узел — O(1), а `pipeline.Find` — O(n), и объясните это комментарием.

7. Демонстрация типичной ошибки из урока. В отдельном методе `BadQueueDemo()` заведите `List<int> bad = [1,2,3,4,5];` и в цикле сделайте `bad.RemoveAt(0)` пять раз, замерив количество операций сдвига в комментарии. Сравните с `Queue<int>` и явно напишите в выводе: «List.RemoveAt(0) — O(n) за вызов, итого O(n²); Queue.Dequeue — амортизированное O(1)».

8. Сборка и запуск:
   ```
   dotnet run
   ```
   Ожидаемый вывод содержит четыре блока: очередь заявок (в порядке поступления), стек отмен (в обратном порядке), каталог шаблонов (по алфавиту), конвейер с вставленными стадиями, плюс блок-предупреждение про `RemoveAt(0)`.

#### Требования к решению
- Целевые платформы и язык: .NET 8, C# 12. Разрешены top-level statements, записи (records), pattern matching, выражения коллекций `[ .. ]`, raw string literals для многострочного вывода. Запрещены сторонние пакеты — только BCL.
- Используйте именно `Queue<T>`, `Stack<T>`, `SortedList<TKey, TValue>` и `LinkedList<T>` из `System.Collections.Generic`. Замена любой из них на `List<T>`/`Dictionary` с ручной эмуляцией считается ошибкой выбора коллекции — это центральная идея урока.
- Каждый блок вывода должен сопровождаться заголовком-разделителем, чтобы проверяющий видел четыре подсистемы. Используйте `Peek` минимум один раз (для очереди), чтобы доказать понимание разницы с `Dequeue`.
- Для `LinkedList<T>` хотя бы одна вставка должна идти через ссылку на узел (`AddBefore`/`AddAfter` с `LinkedListNode<T>`), а не через `Find` + пересоздание. В комментарии укажите, что `Find` сам по себе O(n), но последующая вставка у узла — O(1).
- Код должен компилироваться без предупреждений (включая nullable). Приведите `null`-проверку или оператор `!` там, где `Find` теоретически может вернуть `null`.
- Вывод должен быть детерминированным: порядок заявок, отмен, ключей и стадий строго определён алгоритмом, а не случайным перемешиванием.
- Размер данных невелик, но в комментариях обязаны быть указания сложности: O(1), O(log n), O(n), O(n²) — там, где это релевантно. Это часть закрепления урока.

#### Тонкости и подводные камни
- `Dequeue` и `Pop` **не только возвращают, но и удаляют** элемент. Если в коде вы сначала вызываете `Peek`, а потом в той же итерации `Dequeue` — это две разные операции; `Peek` не двигает очередь. Частая ошибка — вызвать `Dequeue` дважды подряд, думая, что первый вызов «только посмотрел». Урок явно предостерегает от этого.
- `Queue<T>` и `Stack<T>` реализованы поверх массива с циклическим буфером, поэтому `Enqueue`/`Dequeue`/`Push`/`Pop` — **амортизированное O(1)**, но не «гарантированное». При росте внутреннего массива происходит копирование, и отдельный `Enqueue` может быть O(n). Если важна строгая гарантия, рассматривайте `ConcurrentQueue` (внутри сегментированный список) или `LinkedList`-на-основе очереди.
- `SortedList` хранит пары в двух параллельных массивах, отсортированных по ключу. Поэтому вставка/удаление — **O(n)** из-за сдвига. Если в вашей задаче каталог часто меняется, переходите на `SortedDictionary` (красно-чёрное дерево, O(log n) вставка/удаление). В ДЗ каталог меняется редко — `SortedList` уместен.
- `LinkedList<T>` **не имеет индексатора**. Любой `ElementAt(i)` в цикле — это O(n²). Правильный подход — хранить `LinkedListNode<T>` и двигаться через `node.Next`/`node.Previous`, либо просто `foreach`. Урок перечисляет это как частую ошибку.
- `LinkedList<T>.Find(value)` — O(n), потому что это линейный поиск. Но после того как узел найден, `AddBefore`/`AddAfter`/`Remove(node)` — O(1). Не путайте стоимость поиска со стоимостью вставки.
- В многопоточной среде `Queue<T>` и `Stack<T>` **не потокобезопасны**. Если несколько потоков кладут/забирают элементы, используйте `ConcurrentQueue<T>`/`ConcurrentStack<T>` из `System.Collections.Concurrent` либо внешнюю синхронизацию. В базовом ДЗ однопоточно, но в бонусе это учитывается.
- `SortedList` требует, чтобы ключи были сравнимы (`IComparable<TKey>` или переданный `IComparer<TKey>`). Для строк это работает «из коробки», но порядок зависит от культуры — для детерминированного вывода можно передать `StringComparer.Ordinal`.
- Nullable-анализ: `Find` возвращает `LinkedListNode<T>?`. Если вы пишете `var node = list.Find("x")!`, и элемента нет — будет `NullReferenceException`. Лучше явно проверить `if (node is not null)` через pattern matching C# 12.

#### Критерии приёмки
- [ ] Проект `PrintFlow` создаётся командой `dotnet new console`, собирается без ошибок и предупреждений на .NET 8 / C# 12.
- [ ] В коде присутствуют ровно четыре целевые коллекции: `Queue<T>`, `Stack<T>`, `LinkList<T>` нет — `LinkedList<T>`, `SortedList<TKey, TValue>`.
- [ ] Заявки печатаются строго в порядке поступления (FIFO), что доказано выводом `Peek` перед первой итерацией.
- [ ] Отмены печатаются в обратном порядке (LIFO), `Count` после каждого `Pop` уменьшается на 1.
- [ ] Каталог шаблонов перебирается по алфавиту ключей; сделан хотя бы один lookup через индексатор.
- [ ] В конвейер вставлены две новые стадии через `AddBefore`/`AddAfter` по ссылке на узел, итоговый порядок корректен.
- [ ] Присутствует метод `BadQueueDemo()`, демонстрирующий `List.RemoveAt(0)` и объясняющий сложность O(n²) против O(1) у `Queue.Dequeue`.
- [ ] В комментариях явно указаны сложности: амортизированное O(1) для очереди/стека, O(log n) lookup у `SortedList`, O(n) вставка у `SortedList`, O(1) вставка по узлу у `LinkedList`, O(n) у `Find`.
- [ ] Nullable-анализ чист: `Find` обработан через `if (node is not null)` или обоснованный `!`.
- [ ] Вывод детерминирован и разбит на четыре блока с заголовками.
- [ ] Использован хотя бы один элемент C# 12: record, выражение коллекции `[ .. ]` или pattern matching.
- [ ] Нет сторонних NuGet-зависимостей; только BCL.
- [ ] В комментарии к `SortedList` упомянут критерий выбора между `SortedList` и `SortedDictionary` (частота изменений).
- [ ] В комментарии к `Queue`/`Stack` упомянуты `ConcurrentQueue`/`ConcurrentStack` для многопоточного сценария.
- [ ] `Peek` вызван ровно там, где нужно «посмотреть, не удаляя», а не заменяет собой `Dequeue`.
- [ ] Код запускается командой `dotnet run` и выводит все ожидаемые блоки.

#### Подсказки (без прямого ответа)
- Вспомните, что `Queue<T>` имеет конструктор, принимающий `IEnumerable<T>`, — это позволяет инициализировать очередь сразу из массива.
- Для вывода валюты в каталоге попробуйте формат `"{key}: {value:C}"`, но учитывайте, что символ валюты зависит от текущей культуры; для детерминированности можно явно использовать `CultureInfo.InvariantCulture`.
- Чтобы показать разницу `Peek` vs `Dequeue`, выведите `Count` до и после `Peek` — он не должен измениться.
- Для вставки в `LinkedList` сначала получите узел через `Find`, сохраните в переменную типа `LinkedListNode<string>`, и только потом вызывайте `AddBefore`/`AddAfter` у самого списка, передавая узел.
- В `BadQueueDemo` вам не обязательно реально замерять время — достаточно пояснить в комментарии, что `RemoveAt(0)` сдвигает все оставшиеся элементы, поэтому пять вызовов на пяти элементах дают 4+3+2+1 = 10 сдвигов.
- Не путайте `SortedDictionary` и `SortedList`: оба отсортированы, но первая — дерево, второй — массивы.

#### Эталонное решение (разбор)
```csharp
// C# 12 / .NET 8 — PrintFlow: Queue, Stack, SortedList, LinkedList
// Двуязычные комментарии RU+EN

using System;
using System.Collections.Generic;
using System.Globalization;

// Модель заявки / Job record
public record PrintJob(int Id, string Title, int Pages);

// === 1. Queue<T>: приём заявок FIFO / FIFO intake ===
PrintJob[] incoming = [new(1, "Договор", 12), new(2, "Отчёт", 40), new(3, "Прайс", 4)];
var jobs = new Queue<PrintJob>(incoming);              // Enqueue уже выполнен конструктором / seeded

Console.WriteLine("=== Очередь заявок / Job queue (FIFO) ===");
Console.WriteLine($"Следующая без удаления / Peek: {jobs.Peek()}  (Count={jobs.Count})");
while (jobs.Count > 0)
{
    var job = jobs.Dequeue();                          // O(1) амортизированно / amortized O(1)
    Console.WriteLine($"[done] #{job.Id} «{job.Title}» {job.Pages} стр.");
}

// === 2. Stack<T>: отмены LIFO / Undo history ===
var undo = new Stack<string>();
undo.Push("Ввод текста");                              // Push на вершину / push to top
undo.Push("Жирный");
undo.Push("Курсив");

Console.WriteLine("\n=== История отмен / Undo (LIFO) ===");
while (undo.Count > 0)
{
    string action = undo.Pop();                        // Pop снимает вершину / pops top
    Console.WriteLine($"Undo: {action}  (осталось / left: {undo.Count})");
}

// === 3. SortedList<TKey,TValue>: каталог по алфавиту / sorted catalog ===
var templates = new SortedList<string, decimal>(StringComparer.Ordinal)
{
    ["Invoice"]  = 0.15m,
    ["Contract"] = 0.50m,
    ["Letter"]   = 0.05m,
    ["Report"]   = 0.30m,
};                                                     // ключи всегда отсортированы / keys sorted

Console.WriteLine("\n=== Каталог шаблонов / Templates (sorted by key) ===");
foreach (var kv in templates)                          // порядок ключей: Contract, Invoice, Letter, Report
{
    Console.WriteLine(string.Format(CultureInfo.InvariantCulture, "{0}: {1:F2}", kv.Key, kv.Value));
}
Console.WriteLine($"Lookup «Letter» (O(log n)): {templates["Letter"]:F2}");
// Вставка/удаление здесь O(n) из-за сдвига; для частых изменений — SortedDictionary.
// Insert/remove are O(n) due to shifting; use SortedDictionary for frequent changes.

// === 4. LinkedList<T>: конвейер с вставкой у узла / pipeline with node insert ===
var pipeline = new LinkedList<string>();
pipeline.AddLast("Watermark");
pipeline.AddLast("Compress");
pipeline.AddLast("Encrypt");
pipeline.AddLast("Send");

LinkedListNode<string>? compress = pipeline.Find("Compress"); // Find — O(n), но один раз
if (compress is not null)                              // pattern matching C# 12, nullable-safe
{
    pipeline.AddBefore(compress, "Grayscale");         // вставка у узла O(1) / O(1) at node
    pipeline.AddAfter(compress, "Sign");               // тоже O(1) / also O(1)
}

Console.WriteLine("\n=== Конвейер / Pipeline (LinkedList) ===");
for (var node = pipeline.First; node is not null; node = node.Next) // без ElementAt — без O(n²)
{
    Console.WriteLine($"- {node.Value}");
}

// === 5. Демонстрация частой ошибки / Common mistake demo ===
BadQueueDemo();

static void BadQueueDemo()
{
    Console.WriteLine("\n=== Антипаттерн / Anti-pattern ===");
    var bad = new List<int> { 1, 2, 3, 4, 5 };
    // RemoveAt(0) каждый раз сдвигает оставшиеся → O(n) за вызов, итого O(n²).
    // Queue.Dequeue — амортизированное O(1). Урок M06-L05 предостерегает от этого.
    while (bad.Count > 0)
    {
        int x = bad[0];
        bad.RemoveAt(0);                               // сдвиг / shift
        Console.WriteLine($"List.RemoveAt(0) → {x}");
    }
    Console.WriteLine("Вывод: используйте Queue<T>, а не List<T> + RemoveAt(0).");
}
```

Разбор по строкам. Строка `public record PrintJob(...)` использует синтаксис записей C# 12 — это уместно, потому что заявка — неизменяемое значение, и `record` даёт готовый `ToString`, который мы видим в выводе `Peek`. Конструктор `new Queue<PrintJob>(incoming)` сразу наполняет очередь: это эквивалентно `foreach` с `Enqueue`, но лаконичнее, и закрепляет, что `Queue<T>` принимает `IEnumerable<T>`. Вызов `jobs.Peek()` перед циклом — ключевая демонстрация урока: `Peek` возвращает голову, не удаляя её, что подтверждается неизменным `Count`. Цикл `while (jobs.Count > 0)` с `Dequeue` реализует классический FIFO-дренаж, и каждая операция — амортизированное O(1) благодаря внутреннему циклическому буферу.

Стековая часть `Push`/`Pop` симметрична: `Push` кладёт на вершину, `Pop` снимает её же, поэтому порядок вывода обратный (`Курсив`, `Жирный`, `Ввод текста`). Мы выводим `undo.Count` после каждого `Pop`, чтобы визуально доказать, что `Pop` именно удаляет элемент, — это та самая «частая ошибка» из урока, когда разработчик ждёт, что `Pop` только вернёт значение.

`SortedList<string, decimal>` с `StringComparer.Ordinal` гарантирует детерминированный лексикографический порядок независимо от культуры потока. Перебор `foreach` идёт по ключам `Contract, Invoice, Letter, Report` — это прямое следствие устройства `SortedList` (два отсортированных массива). Lookup `templates["Letter"]` — O(log n), потому что внутри бинарный поиск. Комментарий про `SortedDictionary` напоминает правило выбора: редкие изменения → `SortedList`, частые → `SortedDictionary` (дерево, O(log n) вставка).

`LinkedList<string>` наполняется через `AddLast`. `Find("Compress")` — O(n), но выполняется один раз; полученный узел `compress` используется для `AddBefore` и `AddAfter`, каждая из которых O(1), потому что нужно лишь переставить ссылки соседних узлов. Pattern matching `if (compress is not null)` — это nullable-безопасная проверка на C# 12, заменяющая уязвимый оператор `!`. Цикл `for (var node = pipeline.First; node is not null; node = node.Next)` намеренно избегает `ElementAt(i)`, который в цикле дал бы O(n²), — это та самая частая ошибка из урока, и эталон показывает правильную альтернативу.

Метод `BadQueueDemo` намеренно «нарушает» правило и использует `List<int>.RemoveAt(0)`, чтобы физически показать антипаттерн. Комментарий фиксирует сложность: каждый `RemoveAt(0)` сдвигает все оставшиеся элементы, итого O(n²); `Queue.Dequeue` — амортизированное O(1). Так замыкается круг: выбор коллекции продиктован шаблоном доступа, а не привычкой. Все четыре подсистемы вместе доказывают центральную мысль урока — каждая коллекция имеет свою нишу, и в 90 % случаев это всё-таки `List`/`Dictionary`, но в четырёх конкретных сценариях специализированные структуры дают честную сложность и читаемость.

#### Задания на углубление (бонус)
1. **Многопоточный приём заявок.** Перепишите подсистему очереди на `ConcurrentQueue<PrintJob>` и запустите два `Task`: один `Enqueue`, другой `Dequeue`. Выведите итоговое число обработанных заявок и убедитесь, что нет потерь. Объясните, почему `ConcurrentQueue` безопаснее обычного `Queue` без `lock`.
2. **Сравнение SortedList vs SortedDictionary.** Напишите `BenchmarkDemo`, который вставляет 10 000 случайных ключей сначала в `SortedList<int,int>`, потом в `SortedDictionary<int,int>`, и замерьте время через `Stopwatch`. Объясните, почему на больших данных `SortedDictionary` выигрывает при частых вставках, хотя память потребляет больше.
3. **Очередь на LinkedList.** Реализуйте собственную очередь `LinkedQueue<T>` поверх `LinkedList<T>` с методами `Enqueue`/`Dequeue`/`Peek`, все — O(1). Сравните с BCL `Queue<T>` по удобству и объясните, почему BCL использует массив с циклическим буфером, а не связный список (подсказка: локальность кэша).
4. **Обратная польская запись.** Используя `Stack<double>`, напишите вычислитель выражений в RPN (например, `"3 4 + 2 *"` → `14`). Это закрепит LIFO-семантику на реальной задаче разбора выражений из урока.

---

## Statement in English / Постановка на английском

#### Context & motivation
You are prototyping a miniature print-office simulator called "PrintFlow". The office runs four independent processes in parallel, and each of them maps naturally onto one of the collections studied in lesson M06-L05. The first subsystem is intake of print jobs: jobs arrive in arrival order and must be served in exactly that order — a classic FIFO buffer, which is exactly what `Queue<T>` models. The second subsystem is the undo log: when an operator makes a mistake, they undo the most recent action, then the one before it, and so on; that is a LIFO history, described perfectly by `Stack<T>`. The third subsystem is the document template catalog: templates must always enumerate in alphabetical order by name, while the catalog is mostly read and rarely mutated — a textbook case for `SortedList<TKey, TValue>`. The fourth subsystem is the post-processing pipeline: a sequence of stages (watermark, compress, encrypt, send) into which the operator can dynamically insert new stages before or after already known nodes; this is where `LinkedList<T>` shines with its `AddBefore`/`AddAfter` operations running in O(1).

The central lesson this homework must reinforce is that the right collection is chosen by access pattern, not "just in case." In 90 % of situations `List<T>` and `Dictionary<TKey, TValue>` win, but in the four subsystems above the specialized structures give both more honest complexity and more readable code. Along the way you will practice telling `SortedList` apart from `SortedDictionary` by mutation frequency, and you will understand why indexing a `LinkedList` through `ElementAt(i)` inside a loop is O(n²) and how to avoid it by holding a node reference. Finally, you must keep thread safety in mind: `Queue<T>` and `Stack<T>` are not synchronized, so in a multi-threaded setting you need `ConcurrentQueue`/`ConcurrentStack` — this is captured in the bonus task.

#### What to do step by step
1. Create a .NET 8 console project named `PrintFlow`:
   ```
   dotnet new console -n PrintFlow -o PrintFlow --framework net8.0
   cd PrintFlow
   dotnet build
   ```
   Make sure `PrintFlow.csproj` contains `<TargetFramework>net8.0</TargetFramework>`. C# 12 is enabled by default on .NET 8, so you may use top-level statements in `Program.cs`.

2. Model a `PrintJob` (a record) with fields `Id` (int), `Title` (string), `Pages` (int). Seed a few jobs with a C# 12 collection expression: `PrintJob[] incoming = [ new(1, "Contract", 12), new(2, "Report", 40), new(3, "PriceList", 4) ];`.

3. Intake subsystem. Create `Queue<PrintJob> jobs = new(incoming);`. In a `while (jobs.Count > 0)` loop, pull jobs with `Dequeue` and print a line like `[done] #1 «Contract» 12 p.`. Before the very first iteration, show the "next" job using `Peek` without removing it, and verify that after `Peek` the `Count` is unchanged — this checks your understanding of the difference between `Peek` and `Dequeue` from the lesson.

4. Undo subsystem. Create `Stack<string> undo = new();` and `Push` three string actions in order: `"Type text"`, `"Bold"`, `"Italic"`. Then `Pop` three times and print `Undo: …` — the order must be reversed (`Italic`, `Bold`, `Type text`). Count how many times `Pop` decremented `Count` and print the total.

5. Template catalog subsystem. Create `SortedList<string, decimal> templates = new() { ["Invoice"] = 0.15m, ["Contract"] = 0.50m, ["Letter"] = 0.05m, ["Report"] = 0.30m };`. Iterate with `foreach` — the output must be strictly alphabetical by key: `Contract, Invoice, Letter, Report`. Perform a lookup of one key via the indexer (`templates["Letter"]`) and note that such access is O(log n) thanks to binary search.

6. Pipeline subsystem. Create `LinkedList<string> pipeline = new();` and `AddLast` the stages: `"Watermark"`, `"Compress"`, `"Encrypt"`, `"Send"`. Find the `"Compress"` node via `pipeline.Find("Compress")!` and insert `"Grayscale"` before it with `AddBefore`, and `"Sign"` after it with `AddAfter`. Print the final sequence with `foreach`. Reason about why insertion via a node is O(1) while `pipeline.Find` is O(n), and explain it in a comment.

7. Demonstration of the typical lesson mistake. In a separate method `BadQueueDemo()`, create `List<int> bad = [1,2,3,4,5];` and in a loop call `bad.RemoveAt(0)` five times, noting the number of shift operations in a comment. Compare with `Queue<int>` and explicitly print: "List.RemoveAt(0) is O(n) per call, O(n²) total; Queue.Dequeue is amortized O(1)."

8. Build and run:
   ```
   dotnet run
   ```
   The expected output contains four blocks: the job queue (in arrival order), the undo stack (in reverse order), the template catalog (alphabetical), the pipeline with inserted stages, plus the `RemoveAt(0)` warning block.

#### Requirements
- Target platform and language: .NET 8, C# 12. Top-level statements, records, pattern matching, collection expressions `[ .. ]`, and raw string literals for multi-line output are all allowed. No third-party packages — BCL only.
- You must use exactly `Queue<T>`, `Stack<T>`, `SortedList<TKey, TValue>` and `LinkedList<T>` from `System.Collections.Generic`. Replacing any of them with `List<T>`/`Dictionary` plus manual emulation counts as a wrong collection choice — that is the core idea of the lesson.
- Each output block must be preceded by a header separator so the reviewer can see the four subsystems. Use `Peek` at least once (for the queue) to prove you understand the difference from `Dequeue`.
- For `LinkedList<T>`, at least one insertion must go through a node reference (`AddBefore`/`AddAfter` with `LinkedListNode<T>`), not through `Find` + re-creation. In a comment, note that `Find` itself is O(n) but the subsequent insert at the node is O(1).
- The code must compile without warnings (including nullable). Provide a null check or a justified `!` where `Find` could theoretically return `null`.
- The output must be deterministic: the order of jobs, undos, keys, and stages is strictly fixed by the algorithm, not by random shuffling.
- The dataset is small, but comments must mention complexity — O(1), O(log n), O(n), O(n²) — wherever relevant. This is part of reinforcing the lesson.

#### Pitfalls
- `Dequeue` and `Pop` **both return and remove** the element. If your code calls `Peek` and then, in the same iteration, `Dequeue`, those are two different operations; `Peek` does not advance the queue. A common mistake is to call `Dequeue` twice in a row, assuming the first call "just looked". The lesson warns about this explicitly.
- `Queue<T>` and `Stack<T>` are built on an array with a circular buffer, so `Enqueue`/`Dequeue`/`Push`/`Pop` are **amortized O(1)**, but not "guaranteed". When the internal array grows, elements are copied, so a single `Enqueue` may be O(n). If you need a strict guarantee, consider `ConcurrentQueue` (a segmented list inside) or a `LinkedList`-backed queue.
- `SortedList` stores pairs in two parallel arrays sorted by key, so insertion and removal are **O(n)** due to shifting. If your catalog mutates often, switch to `SortedDictionary` (a red-black tree, O(log n) insert/remove). In this homework the catalog changes rarely, so `SortedList` is appropriate.
- `LinkedList<T>` **has no indexer**. Any `ElementAt(i)` inside a loop is O(n²). The correct approach is to hold a `LinkedListNode<T>` and move via `node.Next`/`node.Previous`, or simply `foreach`. The lesson lists this as a frequent mistake.
- `LinkedList<T>.Find(value)` is O(n) because it is a linear search. But once the node is found, `AddBefore`/`AddAfter`/`Remove(node)` are O(1). Do not confuse the cost of the search with the cost of the insert.
- In a multi-threaded environment `Queue<T>` and `Stack<T>` are **not thread-safe**. If several threads enqueue/dequeue, use `ConcurrentQueue<T>`/`ConcurrentStack<T>` from `System.Collections.Concurrent` or external synchronization. The base homework is single-threaded, but the bonus task covers this.
- `SortedList` requires keys to be comparable (`IComparable<TKey>` or an `IComparer<TKey>`). For strings this works out of the box, but ordering depends on culture — for deterministic output pass `StringComparer.Ordinal`.
- Nullable analysis: `Find` returns `LinkedListNode<T>?`. If you write `var node = list.Find("x")!` and the element is missing, you get a `NullReferenceException`. Prefer an explicit `if (node is not null)` using C# 12 pattern matching.

#### Acceptance criteria
- [ ] The `PrintFlow` project is created with `dotnet new console`, builds without errors or warnings on .NET 8 / C# 12.
- [ ] Exactly four target collections appear in the code: `Queue<T>`, `Stack<T>`, `LinkedList<T>`, `SortedList<TKey, TValue>`.
- [ ] Jobs are printed strictly in arrival order (FIFO), proven by a `Peek` output before the first iteration.
- [ ] Undos are printed in reverse order (LIFO); `Count` decreases by 1 after each `Pop`.
- [ ] The template catalog iterates alphabetically by key; at least one lookup via the indexer is performed.
- [ ] Two new stages are inserted into the pipeline via `AddBefore`/`AddAfter` through a node reference; the final order is correct.
- [ ] A `BadQueueDemo()` method is present, demonstrating `List.RemoveAt(0)` and explaining O(n²) vs O(1) of `Queue.Dequeue`.
- [ ] Comments explicitly state complexities: amortized O(1) for queue/stack, O(log n) lookup in `SortedList`, O(n) insert in `SortedList`, O(1) insert at a node in `LinkedList`, O(n) for `Find`.
- [ ] Nullable analysis is clean: `Find` is handled via `if (node is not null)` or a justified `!`.
- [ ] The output is deterministic and split into four labeled blocks.
- [ ] At least one C# 12 feature is used: record, collection expression `[ .. ]`, or pattern matching.
- [ ] No third-party NuGet dependencies; BCL only.
- [ ] A comment on `SortedList` mentions the choice criterion between `SortedList` and `SortedDictionary` (mutation frequency).
- [ ] A comment on `Queue`/`Stack` mentions `ConcurrentQueue`/`ConcurrentStack` for multi-threaded scenarios.
- [ ] `Peek` is called exactly where "look without removing" is needed, not as a substitute for `Dequeue`.
- [ ] The code runs with `dotnet run` and prints all expected blocks.

#### Hints (no direct answer)
- Recall that `Queue<T>` has a constructor accepting `IEnumerable<T>`, so you can seed the queue directly from an array.
- For currency output in the catalog try `"{key}: {value:C}"`, but remember the currency symbol depends on the current culture; for determinism use `CultureInfo.InvariantCulture` explicitly.
- To show the `Peek` vs `Dequeue` difference, print `Count` before and after `Peek` — it must not change.
- For `LinkedList` insertion, first obtain the node via `Find`, store it in a `LinkedListNode<string>` variable, and only then call `AddBefore`/`AddAfter` on the list, passing the node.
- In `BadQueueDemo` you do not have to actually time it — it is enough to explain in a comment that `RemoveAt(0)` shifts all remaining elements, so five calls on five elements produce 4+3+2+1 = 10 shifts.
- Do not confuse `SortedDictionary` and `SortedList`: both are sorted, but the former is a tree and the latter is a pair of arrays.

#### Reference solution walk-through
```csharp
// C# 12 / .NET 8 — PrintFlow: Queue, Stack, SortedList, LinkedList
// Bilingual comments RU+EN

using System;
using System.Collections.Generic;
using System.Globalization;

public record PrintJob(int Id, string Title, int Pages);

// === 1. Queue<T>: FIFO intake ===
PrintJob[] incoming = [new(1, "Contract", 12), new(2, "Report", 40), new(3, "PriceList", 4)];
var jobs = new Queue<PrintJob>(incoming);              // seeded via IEnumerable ctor

Console.WriteLine("=== Job queue (FIFO) ===");
Console.WriteLine($"Peek (no removal): {jobs.Peek()}  (Count={jobs.Count})");
while (jobs.Count > 0)
{
    var job = jobs.Dequeue();                          // amortized O(1)
    Console.WriteLine($"[done] #{job.Id} «{job.Title}» {job.Pages} p.");
}

// === 2. Stack<T>: LIFO undo ===
var undo = new Stack<string>();
undo.Push("Type text");
undo.Push("Bold");
undo.Push("Italic");

Console.WriteLine("\n=== Undo (LIFO) ===");
while (undo.Count > 0)
{
    string action = undo.Pop();                        // pops the top
    Console.WriteLine($"Undo: {action}  (left: {undo.Count})");
}

// === 3. SortedList<TKey,TValue>: sorted catalog ===
var templates = new SortedList<string, decimal>(StringComparer.Ordinal)
{
    ["Invoice"]  = 0.15m,
    ["Contract"] = 0.50m,
    ["Letter"]   = 0.05m,
    ["Report"]   = 0.30m,
};                                                     // keys always sorted

Console.WriteLine("\n=== Templates (sorted by key) ===");
foreach (var kv in templates)                          // order: Contract, Invoice, Letter, Report
{
    Console.WriteLine(string.Format(CultureInfo.InvariantCulture, "{0}: {1:F2}", kv.Key, kv.Value));
}
Console.WriteLine($"Lookup «Letter» (O(log n)): {templates["Letter"]:F2}");
// Insert/remove are O(n) due to shifting; use SortedDictionary for frequent changes.

// === 4. LinkedList<T>: pipeline with node insert ===
var pipeline = new LinkedList<string>();
pipeline.AddLast("Watermark");
pipeline.AddLast("Compress");
pipeline.AddLast("Encrypt");
pipeline.AddLast("Send");

LinkedListNode<string>? compress = pipeline.Find("Compress"); // Find is O(n), once only
if (compress is not null)                              // C# 12 pattern, nullable-safe
{
    pipeline.AddBefore(compress, "Grayscale");         // O(1) at node
    pipeline.AddAfter(compress, "Sign");               // also O(1)
}

Console.WriteLine("\n=== Pipeline (LinkedList) ===");
for (var node = pipeline.First; node is not null; node = node.Next) // no ElementAt, no O(n²)
{
    Console.WriteLine($"- {node.Value}");
}

// === 5. Common mistake demo ===
BadQueueDemo();

static void BadQueueDemo()
{
    Console.WriteLine("\n=== Anti-pattern ===");
    var bad = new List<int> { 1, 2, 3, 4, 5 };
    // RemoveAt(0) shifts the remaining elements each time → O(n) per call, O(n²) total.
    // Queue.Dequeue is amortized O(1). Lesson M06-L05 warns against this.
    while (bad.Count > 0)
    {
        int x = bad[0];
        bad.RemoveAt(0);                               // shift
        Console.WriteLine($"List.RemoveAt(0) → {x}");
    }
    Console.WriteLine("Conclusion: use Queue<T>, not List<T> + RemoveAt(0).");
}
```

Line-by-line walk-through. The `public record PrintJob(...)` line uses C# 12 record syntax — appropriate because a job is an immutable value, and `record` gives a ready-made `ToString` that we see in the `Peek` output. The `new Queue<PrintJob>(incoming)` constructor seeds the queue in one step: this is equivalent to a `foreach` with `Enqueue`, but terser, and it reinforces that `Queue<T>` accepts `IEnumerable<T>`. The `jobs.Peek()` call before the loop is the key lesson demonstration: `Peek` returns the head without removing it, which we prove by showing `Count` unchanged. The `while (jobs.Count > 0)` loop with `Dequeue` performs the classic FIFO drain, each operation being amortized O(1) thanks to the internal circular buffer.

The stack section with `Push`/`Pop` is symmetric: `Push` adds to the top, `Pop` removes from the top, so the output order is reversed (`Italic`, `Bold`, `Type text`). We print `undo.Count` after each `Pop` to visually prove that `Pop` actually removes the element — addressing the lesson's common mistake where a developer expects `Pop` to merely return the value.

`SortedList<string, decimal>` with `StringComparer.Ordinal` guarantees a deterministic lexicographic order regardless of thread culture. The `foreach` iteration yields keys `Contract, Invoice, Letter, Report` — a direct consequence of how `SortedList` is built (two sorted arrays). The lookup `templates["Letter"]` is O(log n) because of internal binary search. The comment about `SortedDictionary` recalls the selection rule: rare mutations → `SortedList`, frequent mutations → `SortedDictionary` (a tree with O(log n) insert).

`LinkedList<string>` is populated with `AddLast`. `Find("Compress")` is O(n) but runs only once; the resulting `compress` node is then used for `AddBefore` and `AddAfter`, each O(1) because only neighbor links need rewiring. The pattern `if (compress is not null)` is a nullable-safe C# 12 check replacing the fragile `!` operator. The `for (var node = pipeline.First; node is not null; node = node.Next)` loop deliberately avoids `ElementAt(i)`, which in a loop would yield O(n²) — the very common mistake from the lesson, and the reference solution shows the correct alternative.

The `BadQueueDemo` method intentionally "breaks" the rule and uses `List<int>.RemoveAt(0)` to physically demonstrate the anti-pattern. The comment pins the complexity: every `RemoveAt(0)` shifts all remaining elements, totaling O(n²); `Queue.Dequeue` is amortized O(1). This closes the loop: the collection choice is dictated by the access pattern, not by habit. All four subsystems together prove the central lesson point — each collection has its niche, and in 90 % of cases it is still `List`/`Dictionary`, but in these four concrete scenarios the specialized structures give honest complexity and readability.

#### Going deeper (bonus)
1. **Multi-threaded intake.** Rewrite the queue subsystem using `ConcurrentQueue<PrintJob>` and launch two `Task`s: one enqueuing, one dequeuing. Print the total number of processed jobs and verify there are no losses. Explain why `ConcurrentQueue` is safer than a plain `Queue` without a `lock`.
2. **SortedList vs SortedDictionary.** Write a `BenchmarkDemo` that inserts 10 000 random keys first into `SortedList<int,int>`, then into `SortedDictionary<int,int>`, and measure the time with `Stopwatch`. Explain why on large data `SortedDictionary` wins on frequent inserts despite higher memory use.
3. **A LinkedList-backed queue.** Implement your own `LinkedQueue<T>` on top of `LinkedList<T>` with `Enqueue`/`Dequeue`/`Peek`, all O(1). Compare it with the BCL `Queue<T>` for ergonomics and explain why BCL uses an array with a circular buffer instead of a linked list (hint: cache locality).
4. **Reverse Polish Notation.** Using `Stack<double>`, write an RPN evaluator (e.g., `"3 4 + 2 *"` → `14`). This anchors LIFO semantics in a real expression-parsing task from the lesson.

---

#### Чек-лист сдачи / Submission checklist
- [ ] (RU) Проект `PrintFlow` собран на .NET 8 / C# 12 без предупреждений.
- [ ] (RU) Использованы `Queue<T>`, `Stack<T>`, `SortedList`, `LinkedList<T>` по назначению.
- [ ] (RU) Вывод разбит на 4 блока + антипаттерн; порядок детерминирован.
- [ ] (RU) В комментариях указаны сложности O(1)/O(log n)/O(n)/O(n²).
- [ ] (RU) `Peek` продемонстрирован отдельно от `Dequeue`.
- [ ] (EN) The `PrintFlow` project builds on .NET 8 / C# 12 with no warnings.
- [ ] (EN) `Queue<T>`, `Stack<T>`, `SortedList`, `LinkedList<T>` are used appropriately.
- [ ] (EN) Output is split into 4 blocks + the anti-pattern; order is deterministic.
- [ ] (EN) Comments state O(1)/O(log n)/O(n)/O(n²) complexities.
- [ ] (EN) `Peek` is demonstrated separately from `Dequeue`.

#### Ресурсы / Resources
- [Microsoft Learn — Collections](https://learn.microsoft.com/dotnet/standard/collections/)
- [Microsoft Learn — Queue<T>](https://learn.microsoft.com/dotnet/api/system.collections.generic.queue-1)
- [Microsoft Learn — Stack<T>](https://learn.microsoft.com/dotnet/api/system.collections.generic.stack-1)
- [Microsoft Learn — SortedList<TKey,TValue>](https://learn.microsoft.com/dotnet/api/system.collections.generic.sortedlist-2)
- [Microsoft Learn — SortedDictionary<TKey,TValue>](https://learn.microsoft.com/dotnet/api/system.collections.generic.sorteddictionary-2)
- [Microsoft Learn — LinkedList<T>](https://learn.microsoft.com/dotnet/api/system.collections.generic.linkedlist-1)
- [Microsoft Learn — ConcurrentQueue<T>](https://learn.microsoft.com/dotnet/api/system.collections.concurrent.concurrentqueue-1)
- [Microsoft Learn — ConcurrentStack<T>](https://learn.microsoft.com/dotnet/api/system.collections.concurrent.concurrentstack-1)
