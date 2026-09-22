---
[← К уроку M08-L06](lesson-M08-L06-aggregates.md) | [⬆ К модулю M08](../README.md) | [Следующее ДЗ →](homework-M08-L07-first-single-elementat.md)
---

### Домашнее задание M08-L06: Агрегаты: Sum/Min/Max/Average/Count / Homework M08-L06: Aggregates: Sum/Min/Max/Average/Count

**Урок / Lesson:** M08-L06
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 4/5
**Цель / Goal:** (RU) Научиться безопасно применять агрегатные операторы LINQ (`Sum`, `Min`, `Max`, `Average`, `Count`, `LongCount`) к реальным данным, корректно обрабатывать пустые последовательности через nullable-проекции и `Any()`, а также осваивать универсальный `Aggregate` с `seed` для свёртки последовательности в составной объект. (EN) Learn to apply LINQ aggregate operators (`Sum`, `Min`, `Max`, `Average`, `Count`, `LongCount`) safely to real data, handle empty sequences correctly via nullable projections and `Any()`, and master the universal `Aggregate` with a `seed` to fold a sequence into a composite object.

#### Связь с уроком / Connection to the lesson
(RU) Урок M08-L06 вводит свёртку последовательности в скаляр и подчёркивает главную ловушку — неодинаковое поведение агрегатов на пустом вводе: `Sum`/`Count` возвращают `0`, а `Min`/`Max`/`Average` для не-nullable значимых типов бросают `InvalidOperationException`. Это ДЗ закрепляет все три формы `Aggregate`, nullable-семантику и связку `GroupBy` + агрегаты на данных, где пустые группы и `null`-значения встречаются реально.
(EN) Lesson M08-L06 introduces folding a sequence into a scalar and stresses the main trap — non-uniform behavior on empty input: `Sum`/`Count` return `0`, whereas `Min`/`Max`/`Average` on non-nullable value types throw `InvalidOperationException`. This homework reinforces all three `Aggregate` forms, nullable semantics, and the `GroupBy` + aggregates combination on data where empty groups and `null` values occur in practice.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация
Вы присоединились к команде аналитики интернет-магазина «Кофейная гуща». Склад данных отдаёт вам плоский список заказов за неделю: каждый заказ относится к одному клиенту, содержит одну позицию товара с количеством и ценой, а итоговая сумма заказа вычисляется как `Price * Quantity`. Часть полей может быть `null` — например, промокод мог не применяться, а у премиум-клиентов поле `DiscountPercent` может отсутствовать. Важно, что за ночь система могла не получить ни одного заказа в какой-то категории товаров (например, чайники не покупали), поэтому при группировке по категориям некоторые группы оказываются пустыми или содержат только заказы с `null`-ценой.

Ваша задача — построить консольное приложение, которое читает такие данные, считает ключевые метрики (общую выручку, средний чек, минимальный и максимальный заказ, количество заказов по категориям), безопасно обрабатывает пустые и `null`-содержащие группы, а также сворачивает всю неделю в один сводный объект через `Aggregate` с `seed`. Это типичная задача production-аналитики, где неаккуратное использование `Min`/`Max`/`Average` на пустых выборках регулярно приводит к падению отчётов в пятницу вечером. Цель ДЗ — выработать автоматический рефлекс: «агрегат по данным, которые могут быть пустыми — значит nullable-проекция или `Any()`».

#### Что нужно сделать (пошагово)
1. Создайте новый консольный проект на .NET 8 через `dotnet new console -n CoffeeAnalytics` в папке `modules/M08/lessons/code/M08-L06`. Убедитесь, что в `.csproj` указано `<LangVersion>latest</LangVersion>` и `<Nullable>enable</Nullable>`, чтобы можно было использовать C# 12 и nullable-аннотации.
2. В файле `Program.cs` опишите модель данных с помощью `record` и collection expressions. Минимальный набор: `record Order(int Id, string Customer, string Category, decimal? Price, int Quantity, decimal? DiscountPercent);`. Задайте тестовые данные на неделю — не менее 12 заказов, минимум 3 категории, минимум 2 клиента. Обязательно включите «крайние» случаи: заказ с `Price == null`, заказ с `Quantity == 0`, заказ с `DiscountPercent == null`, и категорию, по которой нет ни одного заказа (например, `"Teapots"`).
3. Реализуйте метод `decimal Total(Order o)`, который возвращает `Price * Quantity * (1 - DiscountPercent/100)` с учётом того, что `Price` и `DiscountPercent` могут быть `null`: если `Price` равен `null`, заказ не вносит вклад (возвращайте `0m`); если `DiscountPercent` равен `null`, скидка считается нулевой. Используйте pattern matching и null-coalescing.
4. Посчитайте и выведите общие агрегаты по всей неделе: `Count`, `LongCount`, `Sum` выручки, `Max` заказа, `Min` заказа, `Average` чека. Для `Min`/`Max`/`Average` обязательно используйте nullable-проекцию `Select(o => (decimal?)Total(o))`, чтобы пустые и `null`-содержащие выборки не бросали исключение. Вывод оформите через interpolated strings с форматированием валюты (`:C` или `:F2`).
5. Сгруппируйте заказы по `Category` через `GroupBy` и для каждой группы посчитайте `Count`, `Sum`, `Average`, `Max`, `Min` в одной LINQ-цепочке через анонимный объект. Для категорий, где все `Total` равны `null`/`0`, `Min`/`Max`/`Average` должны вернуть `null`, а не упасть. Выведите сводную таблицу по категориям.
6. Реализуйте свёртку всех заказов в один составной объект `WeeklyStats` через `Aggregate` с `seed`. Структура: `(int Count, decimal Revenue, decimal MaxOrder, decimal MinOrder, int MaxQuantity)`. В качестве `seed` используйте кортеж с нейтральными значениями. Внутри `func` обновляйте аккумулятор. Продемонстрируйте, что пустая коллекция (`Array.Empty<Order>()`) с тем же `Aggregate` возвращает `seed` без исключения.
7. Дополнительно реализуйте `Aggregate` с `resultSelector`: соберите строку с перечислением уникальных клиентов через запятую, начиная с `"Customers: "`, и обрежьте завершающие разделители через `TrimEnd(',', ' ')`, как в примере из урока.
8. Запустите приложение через `dotnet run` и убедитесь, что вывод содержит: общие метрики, таблицу по категориям (с корректными `null` для пустых групп), результат `Aggregate`-свёртки, строку клиентов и подтверждение, что пустая коллекция не бросает исключение. Сделайте скриншот или сохраните текст вывода.

#### Требования к решению
- Код должен компилироваться и запускаться под .NET 8 / C# 12 без предупреждений уровня error. Разрешены top-level statements, collection expressions, pattern matching, raw string literals для многострочного вывода.
- Все агрегаты `Min`/`Max`/`Average` по данным, которые могут быть пустыми или содержать `null`, обязаны использовать nullable-проекцию `Select(o => (T?)...)` или предварительную проверку `Any()`. Прямой вызов `Min()`/`Max()` на не-nullable проекции по потенциально пустой выборке считается ошибкой и должен быть явно прокомментирован как «небезопасно — демонстрация ловушки».
- `Aggregate` должен вызываться с `seed` во всех случаях, где возможна пустая последовательность. Использование формы `Aggregate(func)` без `seed` допускается только в демонстрационных целях с комментарием, почему это опасно.
- Имена переменных и методов — осмысленные. Дублирование логики вычисления `Total` недопустимо: выделите её в один метод и переиспользуйте. Код должен быть декларативным: предпочитайте перегрузки с селектором (`orders.Sum(o => Total(o))`) варианту `orders.Select(o => Total(o)).Sum()`.
- В выводе должны быть видны как успешные результаты, так и обработанные «крайние» случаи: заказ с `Price == null` не должен завышать `Sum`, категория без заказов должна отображаться с `null` метриками, а пустая коллекция должна возвращать `seed`.

#### Тонкости и подводные камни
- **Пустая последовательность — главный сюрприз.** `Min`/`Max` на не-nullable значимом типе (`int`, `double`, `decimal`) бросают `InvalidOperationException` на пустом вводе. Это самая частая причина падения production-отчётов. Если есть хоть малейший риск пустоты — nullable-проекция `Select(x => (decimal?)x)` или `Any()`.
- **`Sum` пустой последовательности равен `0`, а не `null`.** Не пишите лишних проверок `if (orders.Any())` перед `Sum` — это мёртвый код. Но помните: `Sum` по `null`-элементам в nullable-варианте ведёт себя как `0` для пропущенных элементов.
- **`Average` на пустой не-nullable коллекции бросает.** На nullable возвращает `null`. Различайте эти случаи осознанно: если бизнес-логика ожидает число, а не `null`, используйте `?? 0m` после nullable-`Average`.
- **`Count()` (метод LINQ, O(n)) vs свойство `Count` (у `List<T>`, O(1)).** Для `ICollection<T>` LINQ-`Count()` оптимизирован и падает на свойство, но на `IEnumerable` из `Where` он реально считает. Если нужна скорость на огромных данных — `LongCount()`, чтобы не переполнить `int`.
- **`Aggregate` без `seed` бросает на пустой или одноэлементной коллекции.** Всегда передавайте `seed`, если возможна пустота. Форма `Aggregate(seed, func, resultSelector)` удобна для финальной проекции, например очистки хвостовых разделителей.
- **Несколько проходов по одним данным.** Если производительность критична, сворачивайте одним `Aggregate` в кортеж вместо отдельных `Sum`+`Min`+`Max`+`Average`. Но в 95% случаев читаемость важнее микросекунд — не оптимизируйте преждевременно.
- **`null` в проекции селектора.** Если `selector` возвращает `null` для части элементов, nullable-агрегаты пропускают их автоматически. Не-nullable перегрузки в этом случае бросают `Nullable`-связанные исключения или ведут себя непредсказуемо — всегда используйте nullable-перегрузки для данных с `null`.
- **`GroupBy` + агрегаты по пустым группам.** Группа, в которой все элементы дали `null`-проекцию, при `Min`/`Max` через nullable вернёт `null`. Если в группе вообще нет элементов (такое бывает при объединении с эталонным списком категорий через `GroupJoin`), нужно отдельно обработать этот случай.

#### Критерии приёмки
- [ ] Проект создаётся командой `dotnet new console` и собирается без ошибок под .NET 8 / C# 12.
- [ ] В `.csproj` включены `<Nullable>enable</Nullable>` и актуальный `<LangVersion>`.
- [ ] Модель `Order` содержит не менее 12 записей с крайними случаями (`Price == null`, `Quantity == 0`, `DiscountPercent == null`, пустая категория).
- [ ] Метод `Total` корректно обрабатывает `null`-поля через pattern matching или null-coalescing.
- [ ] Общие агрегаты (`Count`, `LongCount`, `Sum`, `Max`, `Min`, `Average`) выведены и корректны.
- [ ] `Min`/`Max`/`Average` используют nullable-проекцию и не бросают исключение на пустых/`null` выборках.
- [ ] Таблица по категориям построена через `GroupBy` + анонимный объект с пятью агрегатами.
- [ ] Пустые или полностью-`null` категории отображаются с `null` метриками, без падения.
- [ ] `Aggregate` с `seed` сворачивает все заказы в `WeeklyStats` (кортеж из 5 полей).
- [ ] Демонстрируется, что `Aggregate` с `seed` на `Array.Empty<Order>()` возвращает `seed` без исключения.
- [ ] `Aggregate` с `resultSelector` строит строку клиентов через запятую с корректной обрезкой хвоста.
- [ ] В коде есть комментарии RU+EN, объясняющие, где применена nullable-защита и почему.
- [ ] Вывод `dotnet run` содержит все требуемые секции и читаем.
- [ ] Нет преждевременной оптимизации там, где она не нужна (отдельные агрегаты допустимы).
- [ ] Использованы возможности C# 12: collection expressions, top-level statements, pattern matching.
- [ ] Код не содержит TODO, заглушек и ссылок «см. выше».

#### Подсказки (без прямого ответа)
- Вспомните из урока: безопасный минимум выглядит как `orders.Where(...).Select(o => (decimal?)Total(o)).Min()`. Примените этот шаблон ко всем экстремумам и средним.
- Для `Aggregate` в кортеж начните с `seed: (Count: 0, Revenue: 0m, MaxOrder: decimal.MinValue, MinOrder: decimal.MaxValue, MaxQuantity: 0)` — подумайте, почему для `Max`/`Min` внутри аккумулятора лучше нейтральные экстремумы, а не `0`.
- Чтобы собрать уникальных клиентов, сначала `Select(o => o.Customer)`, затем `Distinct()`, затем `Aggregate` со строковым `seed`.
- Для вывода `null` используйте `value?.ToString() ?? "null"` или интерполяцию с pattern matching.
- Не забудьте продемонстрировать небезопасный вариант в комментарии: `// orders.Where(o => o.Category == "Teapots").Min(o => Total(o)); // InvalidOperationException!`

#### Эталонное решение (разбор)
```csharp
// C# 12 / .NET 8 — Аналитика заказов: агрегаты LINQ / Orders analytics: LINQ aggregates
using System.Linq;

// Модель данных / Data model
record Order(int Id, string Customer, string Category, decimal? Price, int Quantity, decimal? DiscountPercent);

// Тестовые данные за неделю через collection expressions / Week of test data via collection expressions
List<Order> orders =
[
    new(1,  "Alice",  "Coffee",   12.50m, 3, null),
    new(2,  "Bob",    "Coffee",   18.00m, 1, 5m),
    new(3,  "Alice",  "Tea",      9.00m,  4, null),
    new(4,  "Charlie","Coffee",   25.00m, 2, 10m),
    new(5,  "Bob",    "Syrup",    6.50m,  6, null),
    new(6,  "Alice",  "Coffee",   null,   2, null),   // Price == null — нет вклада / no contribution
    new(7,  "Charlie","Tea",      7.50m,  0, null),   // Quantity == 0
    new(8,  "Bob",    "Syrup",    6.50m,  3, null),
    new(9,  "Alice",  "Coffee",   15.00m, 5, null),
    new(10, "Dave",   "Tea",      9.00m,  2, 0m),
    new(11, "Eve",    "Syrup",    6.50m,  1, null),
    new(12, "Alice",  "Coffee",   30.00m, 1, 15m),
    // Категория "Teapots" намеренно отсутствует / "Teapots" category intentionally absent
];

// Итог заказа с учётом null-полей / Order total honoring null fields
decimal Total(Order o) =>
    o switch
    {
        { Price: null }          => 0m,                                   // нет цены — нет вклада
        { Price: var p, DiscountPercent: null, Quantity: var q } => p * q,
        { Price: var p, DiscountPercent: var d, Quantity: var q } => p * q * (1 - d / 100m),
    };

// --- Общие агрегаты / Overall aggregates ---
int    count       = orders.Count;                              // свойство List<T>, O(1)
long   longCount   = orders.LongCount();                        // для огромных выборок / for huge sets
decimal revenue    = orders.Sum(Total);                         // перегрузка с селектором — один проход
double avgItems    = orders.Average(o => o.Quantity);           // Quantity не nullable — безопасно

// Безопасные экстремумы через nullable-проекцию / Safe extremes via nullable projection
decimal? maxOrder  = orders.Select(o => (decimal?)Total(o)).Max();
decimal? minOrder  = orders
    .Where(o => o.Category == "Teapots")                        // пустая выборка / empty subset
    .Select(o => (decimal?)Total(o))
    .Min();                                                     // null — без исключения / null, no throw
decimal? avgCheck  = orders
    .Where(o => o.Category == "Teapots")
    .Select(o => (decimal?)Total(o))
    .Average();                                                 // null

Console.WriteLine(
    $"""
    --- Общие метрики недели / Weekly overall metrics ---
    Count      = {count}
    LongCount  = {longCount}
    Revenue    = {revenue:C}
    AvgItems   = {avgItems:F2}
    MaxOrder   = {maxOrder?.ToString("C") ?? "null"}
    MinOrder   = {minOrder?.ToString("C") ?? "null"}  (Teapots — пустая группа)
    AvgCheck   = {avgCheck?.ToString("C") ?? "null"}  (Teapots — пустая группа)
    """
);

// --- Сводка по категориям через GroupBy + агрегаты / Per-category summary via GroupBy + aggregates ---
// Эталонный список категорий, включая пустую "Teapots" / Reference category list incl. empty "Teapots"
string[] allCategories = ["Coffee", "Tea", "Syrup", "Teapots"];

var byCategory =
    from cat in allCategories
    join o in orders on cat equals o.Category into grp
    select new
    {
        Category  = cat,
        Count     = grp.Count(),
        Revenue   = grp.Sum(Total),
        AvgCheck  = grp.Select(o => (decimal?)Total(o)).Average(),   // null для пустой группы
        MaxCheck  = grp.Select(o => (decimal?)Total(o)).Max(),       // null для пустой группы
        MinCheck  = grp.Select(o => (decimal?)Total(o)).Min(),       // null для пустой группы
    };

Console.WriteLine("\n--- Сводка по категориям / Per-category summary ---");
Console.WriteLine("Category | Count | Revenue    | AvgCheck   | MaxCheck   | MinCheck");
foreach (var s in byCategory)
    Console.WriteLine(
        $"{s.Category,-9} | {s.Count,5} | {s.Revenue,10:C} | " +
        $"{(s.AvgCheck?.ToString("C") ?? "null"),10} | " +
        $"{(s.MaxCheck?.ToString("C") ?? "null"),10} | " +
        $"{(s.MinCheck?.ToString("C") ?? "null"),10}");

// --- Aggregate со seed: сворачиваем в составной объект / Aggregate with seed: fold into composite ---
var weekly = orders.Aggregate(
    seed: (Count: 0, Revenue: 0m, MaxOrder: decimal.MinValue, MinOrder: decimal.MaxValue, MaxQuantity: 0),
    func: (acc, o) => (
        Count:       acc.Count + 1,
        Revenue:     acc.Revenue + Total(o),
        MaxOrder:    Math.Max(acc.MaxOrder, Total(o)),
        MinOrder:    Total(o) == 0m ? acc.MinOrder : Math.Min(acc.MinOrder, Total(o)),
        MaxQuantity: Math.Max(acc.MaxQuantity, o.Quantity)));
Console.WriteLine(
    $"\n--- Aggregate-свёртка / Aggregate fold ---\n" +
    $"Count={weekly.Count}, Revenue={weekly.Revenue:C}, MaxOrder={weekly.MaxOrder:C}, " +
    $"MinOrder={weekly.MinOrder:C}, MaxQuantity={weekly.MaxQuantity}");

// --- Демонстрация безопасности seed на пустой коллекции / Seed safety on empty collection ---
var empty = Array.Empty<Order>();
var emptyFold = empty.Aggregate(
    seed: (Count: 0, Revenue: 0m, MaxOrder: decimal.MinValue, MinOrder: decimal.MaxValue, MaxQuantity: 0),
    func: (acc, o) => acc);  // не вызовется / never invoked
Console.WriteLine($"Empty fold = ({emptyFold.Count}, {emptyFold.Revenue}) — без исключения / no throw");

// --- Aggregate с resultSelector: строка клиентов / Aggregate with resultSelector: customer string ---
string customers = orders
    .Select(o => o.Customer)
    .Distinct()
    .Aggregate(
        seed: "Customers: ",
        func: (acc, name) => acc + name + ", ",
        resultSelector: acc => acc.TrimEnd(',', ' '));
Console.WriteLine(customers);

// --- Демонстрация небезопасного варианта (только комментарий!) / Unsafe variant (comment only!) ---
// orders.Where(o => o.Category == "Teapots").Min(o => Total(o)); // InvalidOperationException!
```

Разбор по строкам. Модель `Order` сделана `record`-ом с nullable-полями `Price` и `DiscountPercent` — это сразу моделирует реальную аналитику, где часть данных отсутствует. Тестовые данные через collection expression `[…]` включают четыре крайних случая: заказ с `Price == null` (№6), заказ с `Quantity == 0` (№7), несколько заказов с `DiscountPercent == null`, и намеренно отсутствующую категорию `"Teapots"`. Метод `Total` использует switch expression с property patterns: ветка `{ Price: null }` возвращает `0m`, ветка с `DiscountPercent: null` игнорирует скидку, общая ветка применяет скидку. Это позволяет переиспользовать `Total` во всех агрегатах без дублирования.

Общие метрики используют перегрузки с селектором (`orders.Sum(Total)`) — один проход и читаемость, как требует best practice из урока. `Count` берётся как свойство `List<T>` (O(1)), а `LongCount()` — как метод LINQ, чтобы продемонстрировать оба варианта. Ключевой момент — экстремумы `maxOrder`, `minOrder`, `avgCheck` для потенциально пустой выборки (категория `"Teapots"`) идут через `Select(o => (decimal?)Total(o))`: именно nullable-проекция делает `Min`/`Max`/`Average` безопасными и возвращающими `null` вместо `InvalidOperationException`. Сравните с закомментированной опасной строкой внизу — она бы упала.

Сводка по категориям использует `join … into grp` (это `GroupJoin`) по эталонному списку `allCategories`, чтобы пустая категория `"Teapots"` тоже попала в результат с `Count == 0` и `null` метриками. Все пять агрегатов (`Count`, `Sum`, `Average`, `Max`, `Min`) вычисляются в одной LINQ-цепочке через анонимный объект — это декларативный и эффективный способ, рекомендованный уроком. Обратите внимание: `Sum(Total)` здесь безопасен даже для пустой группы, потому что `Sum` пустой последовательности равен `0`, а не бросает.

`Aggregate` с `seed` сворачивает все заказы в кортеж из пяти полей за один проход. Выбор `MaxOrder: decimal.MinValue` и `MinOrder: decimal.MaxValue` в качестве нейтральных элементов — стандартный приём для экстремумов при fold-операции: первый реальный элемент неминуемо их перезапишет через `Math.Max`/`Math.Min`. Особое условие `Total(o) == 0m ? acc.MinOrder : Math.Min(...)` исключает заказы с нулевым вкладом (от `Price == null` или `Quantity == 0`) из расчёта минимума, чтобы `MinOrder` не оказался `0m` из-за «пустого» заказа. Демонстрация `empty.Aggregate(seed, func)` подтверждает безопасность формы с `seed` на пустой коллекции: `func` не вызывается ни разу, возвращается `seed` — это прямо та гарантия, которую урок называет главным преимуществом `Aggregate(seed, func)` над `Aggregate(func)`.

Наконец, `Aggregate` с `resultSelector` собирает строку клиентов: `seed "Customers: "`, накопление `acc + name + ", "`, финальная проекция `TrimEnd(',', ' ')` убирает хвостовой разделитель. Это точное повторение примера из урока, но на данных ДЗ. В целом решение покрывает все темы урока: базовые агрегаты, `LongCount`, nullable-семантику, пустые последовательности, три формы `Aggregate`, `GroupBy`/`GroupJoin` + агрегаты, выбор свойства `Count` vs метода `Count()`, и единственный проход через `Aggregate` как альтернативу цепочке отдельных агрегатов.

#### Задания на углубление (бонус)
1. **Один проход вместо четырёх.** Замените отдельные `Sum`+`Min`+`Max`+`Average` по одной категории одним `Aggregate` в кортеж, чтобы обойти группу за один проход. Измерьте разницу через `Stopwatch` на 1 000 000 заказов и обсудите, стоит ли оптимизация потери читаемости.
2. **Скользящее окно.** Реализуйте метод, который для каждого дня недели считает `Average` чека за этот и предыдущий день (окно шириной 2). Подумайте, как `Aggregate` с `seed`-аккумулятором-кортежем помогает хранить «предыдущий день».
3. **Кастомный агрегат-монада.** Через `Aggregate(seed, func, resultSelector)` постройте гистограмму распределения чеков по корзинам `[0–10), [10–50), [50–100), [100+)` в виде `Dictionary<string,int>`. Обсудите, почему здесь `Aggregate` уместнее цепочки `Where(...).Count()`.
4. **Потокобезопасная агрегация.** Исследуйте `ParallelEnumerable` и `Aggregate` с перегрузкой для параллельной свёртки (seed factory, updateAccumulatorFunc, combineAccumulatorsFunc, resultSelector). Объясните, зачем нужны отдельные локальные аккумуляторы и их объединение.

---

## Statement in English / Постановка на английском

#### Context & motivation
You have joined the analytics team of the "Coffee Grounds" online store. The data warehouse hands you a flat list of a week of orders: each order belongs to a single customer, contains a single product line with a quantity and a unit price, and the order total is computed as `Price * Quantity`. Several fields may be `null` — for example a promo code may not have been applied, and for premium customers the `DiscountPercent` field may be missing. Importantly, during the night the system may have received no orders in some product category (say, teapots were not purchased at all), so when you group by category some groups turn out to be empty or contain only orders with a `null` price.

Your task is to build a console application that reads such data, computes the key metrics (total revenue, average check, minimum and maximum order, number of orders per category), safely handles empty and `null`-containing groups, and folds the entire week into a single summary object via `Aggregate` with a `seed`. This is a typical production-analytics chore where careless use of `Min`/`Max`/`Average` on empty subsets regularly takes down the Friday evening report. The goal of this homework is to train an automatic reflex: "an aggregate over data that might be empty means a nullable projection or an `Any()` guard."

#### What to do step by step
1. Create a new .NET 8 console project via `dotnet new console -n CoffeeAnalytics` inside `modules/M08/lessons/code/M08-L06`. Make sure the `.csproj` sets `<LangVersion>latest</LangVersion>` and `<Nullable>enable</Nullable>` so you can use C# 12 and nullable annotations.
2. In `Program.cs` describe the data model with a `record` and collection expressions. The minimum set is `record Order(int Id, string Customer, string Category, decimal? Price, int Quantity, decimal? DiscountPercent);`. Provide a week of test data — at least 12 orders, at least 3 categories, at least 2 customers. Be sure to include edge cases: an order with `Price == null`, an order with `Quantity == 0`, an order with `DiscountPercent == null`, and a category that has no orders at all (for example `"Teapots"`).
3. Implement a method `decimal Total(Order o)` that returns `Price * Quantity * (1 - DiscountPercent/100)`, taking into account that `Price` and `DiscountPercent` may be `null`: if `Price` is `null` the order contributes nothing (return `0m`); if `DiscountPercent` is `null` the discount is treated as zero. Use pattern matching and null-coalescing.
4. Compute and print the overall aggregates for the whole week: `Count`, `LongCount`, revenue `Sum`, `Max` order, `Min` order, `Average` check. For `Min`/`Max`/`Average` you must use a nullable projection `Select(o => (decimal?)Total(o))` so that empty and `null`-containing subsets do not throw. Format the output with interpolated strings using currency formatting (`:C` or `:F2`).
5. Group the orders by `Category` using `GroupBy` and for each group compute `Count`, `Sum`, `Average`, `Max`, `Min` in a single LINQ chain through an anonymous object. For categories where every `Total` is `null` or `0`, the `Min`/`Max`/`Average` must return `null` instead of throwing. Print the per-category summary table.
6. Fold all orders into a single composite object `WeeklyStats` via `Aggregate` with a `seed`. The shape is `(int Count, decimal Revenue, decimal MaxOrder, decimal MinOrder, int MaxQuantity)`. Use a tuple with neutral values as the `seed`. Inside `func` update the accumulator. Demonstrate that an empty collection (`Array.Empty<Order>()`) with the same `Aggregate` returns the `seed` without throwing.
7. Additionally implement `Aggregate` with a `resultSelector`: build a string listing the unique customers comma-separated, starting with `"Customers: "`, and trim the trailing separators with `TrimEnd(',', ' ')`, exactly as in the lesson example.
8. Run the application with `dotnet run` and verify that the output contains: the overall metrics, the per-category table (with correct `null` values for empty groups), the result of the `Aggregate` fold, the customer string, and confirmation that the empty collection does not throw. Take a screenshot or save the output text.

#### Requirements
- The code must compile and run on .NET 8 / C# 12 with no error-level warnings. Top-level statements, collection expressions, pattern matching, and raw string literals for multi-line output are allowed and encouraged.
- Every `Min`/`Max`/`Average` over data that might be empty or contain `null` must use a nullable projection `Select(o => (T?)...)` or a prior `Any()` check. A direct `Min()`/`Max()` call on a non-nullable projection over a possibly-empty subset is considered a bug and, if it appears for demonstration, must be explicitly commented as "unsafe — trap demonstration".
- `Aggregate` must be called with a `seed` wherever the sequence might be empty. Using the `Aggregate(func)` form without a `seed` is allowed only for demonstration with a comment explaining why it is dangerous.
- Variable and method names must be meaningful. Duplicating the `Total` computation is not allowed: factor it into a single method and reuse it. The code must be declarative: prefer the selector overloads (`orders.Sum(o => Total(o))`) over `orders.Select(o => Total(o)).Sum()`.
- The output must show both successful results and handled edge cases: an order with `Price == null` must not inflate `Sum`; a category with no orders must display `null` metrics; and an empty collection must return the `seed`.

#### Pitfalls
- **Empty sequence — the big surprise.** `Min`/`Max` on a non-nullable value type (`int`, `double`, `decimal`) throw `InvalidOperationException` on empty input. This is the most common cause of production report crashes. Whenever emptiness is even possible — use a nullable projection `Select(x => (decimal?)x)` or `Any()`.
- **`Sum` of an empty sequence is `0`, not `null`.** Do not add redundant `if (orders.Any())` checks before `Sum` — that is dead code. Remember though: `Sum` over `null` elements in the nullable variant treats skipped elements as `0`.
- **`Average` on an empty non-nullable collection throws.** On the nullable variant it returns `null`. Choose consciously: if business logic expects a number rather than `null`, use `?? 0m` after a nullable `Average`.
- **`Count()` (LINQ method, O(n)) vs the `Count` property (`List<T>`, O(1)).** For `ICollection<T>` the LINQ `Count()` is optimized and falls back to the property, but on an `IEnumerable` produced by `Where` it really does count. If you need speed on huge data — `LongCount()`, to avoid `int` overflow.
- **`Aggregate` without a `seed` throws on an empty or single-element sequence.** Always pass a `seed` if emptiness is possible. The `Aggregate(seed, func, resultSelector)` form is convenient for a final projection, for example trimming trailing separators.
- **Multiple passes over the same data.** If performance is critical, fold in a single `Aggregate` into a tuple instead of separate `Sum`+`Min`+`Max`+`Average`. But in 95% of cases readability beats microseconds — do not optimize prematurely.
- **`null` in the selector projection.** If the `selector` returns `null` for some elements, the nullable aggregates skip them automatically. The non-nullable overloads in that case throw or behave unpredictably — always use the nullable overloads for `null`-bearing data.
- **`GroupBy` plus aggregates over empty groups.** A group in which every element projected to `null` will return `null` from `Min`/`Max` via the nullable variant. If a group has no elements at all (which happens when you join against a reference list of categories via `GroupJoin`), handle that case explicitly.

#### Acceptance criteria
- [ ] The project is created with `dotnet new console` and builds without errors on .NET 8 / C# 12.
- [ ] The `.csproj` enables `<Nullable>enable</Nullable>` and a current `<LangVersion>`.
- [ ] The `Order` model contains at least 12 records including edge cases (`Price == null`, `Quantity == 0`, `DiscountPercent == null`, an empty category).
- [ ] The `Total` method correctly handles `null` fields via pattern matching or null-coalescing.
- [ ] The overall aggregates (`Count`, `LongCount`, `Sum`, `Max`, `Min`, `Average`) are printed and correct.
- [ ] `Min`/`Max`/`Average` use a nullable projection and do not throw on empty or `null`-bearing subsets.
- [ ] The per-category table is built with `GroupBy` (or `GroupJoin`) and an anonymous object with five aggregates.
- [ ] Empty or all-`null` categories display `null` metrics without crashing.
- [ ] `Aggregate` with a `seed` folds all orders into `WeeklyStats` (a 5-field tuple).
- [ ] It is demonstrated that `Aggregate` with a `seed` on `Array.Empty<Order>()` returns the `seed` without throwing.
- [ ] `Aggregate` with a `resultSelector` builds the customer string with correct trailing-trim.
- [ ] The code has RU+EN comments explaining where nullable safety is applied and why.
- [ ] The `dotnet run` output contains every required section and is readable.
- [ ] There is no premature optimization where it is not needed (separate aggregates are acceptable).
- [ ] C# 12 features are used: collection expressions, top-level statements, pattern matching.
- [ ] The code contains no TODOs, stubs, or "see above" references.

#### Hints (no direct answer)
- Recall from the lesson: a safe minimum looks like `orders.Where(...).Select(o => (decimal?)Total(o)).Min()`. Apply this template to every extreme and average.
- For the tuple `Aggregate`, start with `seed: (Count: 0, Revenue: 0m, MaxOrder: decimal.MinValue, MinOrder: decimal.MaxValue, MaxQuantity: 0)` — think about why neutral extremes are better than `0` for `Max`/`Min` inside the accumulator.
- To gather unique customers, first `Select(o => o.Customer)`, then `Distinct()`, then `Aggregate` with a string `seed`.
- For printing `null`, use `value?.ToString() ?? "null"` or interpolation with pattern matching.
- Do not forget to demonstrate the unsafe variant in a comment: `// orders.Where(o => o.Category == "Teapots").Min(o => Total(o)); // InvalidOperationException!`

#### Reference solution walk-through
```csharp
// C# 12 / .NET 8 — Orders analytics: LINQ aggregates
using System.Linq;

// Data model
record Order(int Id, string Customer, string Category, decimal? Price, int Quantity, decimal? DiscountPercent);

// Week of test data via collection expressions
List<Order> orders =
[
    new(1,  "Alice",  "Coffee",   12.50m, 3, null),
    new(2,  "Bob",    "Coffee",   18.00m, 1, 5m),
    new(3,  "Alice",  "Tea",      9.00m,  4, null),
    new(4,  "Charlie","Coffee",   25.00m, 2, 10m),
    new(5,  "Bob",    "Syrup",    6.50m,  6, null),
    new(6,  "Alice",  "Coffee",   null,   2, null),   // Price == null — no contribution
    new(7,  "Charlie","Tea",      7.50m,  0, null),   // Quantity == 0
    new(8,  "Bob",    "Syrup",    6.50m,  3, null),
    new(9,  "Alice",  "Coffee",   15.00m, 5, null),
    new(10, "Dave",   "Tea",      9.00m,  2, 0m),
    new(11, "Eve",    "Syrup",    6.50m,  1, null),
    new(12, "Alice",  "Coffee",   30.00m, 1, 15m),
    // The "Teapots" category is intentionally absent
];

// Order total honoring null fields
decimal Total(Order o) =>
    o switch
    {
        { Price: null }          => 0m,                                   // no price — no contribution
        { Price: var p, DiscountPercent: null, Quantity: var q } => p * q,
        { Price: var p, DiscountPercent: var d, Quantity: var q } => p * q * (1 - d / 100m),
    };

// --- Overall aggregates ---
int    count       = orders.Count;                              // List<T> property, O(1)
long   longCount   = orders.LongCount();                        // for huge sets
decimal revenue    = orders.Sum(Total);                         // selector overload — one pass
double avgItems    = orders.Average(o => o.Quantity);           // Quantity is non-nullable — safe

// Safe extremes via nullable projection
decimal? maxOrder  = orders.Select(o => (decimal?)Total(o)).Max();
decimal? minOrder  = orders
    .Where(o => o.Category == "Teapots")                        // empty subset
    .Select(o => (decimal?)Total(o))
    .Min();                                                     // null — no throw
decimal? avgCheck  = orders
    .Where(o => o.Category == "Teapots")
    .Select(o => (decimal?)Total(o))
    .Average();                                                 // null

Console.WriteLine(
    $"""
    --- Weekly overall metrics ---
    Count      = {count}
    LongCount  = {longCount}
    Revenue    = {revenue:C}
    AvgItems   = {avgItems:F2}
    MaxOrder   = {maxOrder?.ToString("C") ?? "null"}
    MinOrder   = {minOrder?.ToString("C") ?? "null"}  (Teapots — empty group)
    AvgCheck   = {avgCheck?.ToString("C") ?? "null"}  (Teapots — empty group)
    """
);

// --- Per-category summary via GroupJoin + aggregates ---
// Reference category list including the empty "Teapots"
string[] allCategories = ["Coffee", "Tea", "Syrup", "Teapots"];

var byCategory =
    from cat in allCategories
    join o in orders on cat equals o.Category into grp
    select new
    {
        Category  = cat,
        Count     = grp.Count(),
        Revenue   = grp.Sum(Total),
        AvgCheck  = grp.Select(o => (decimal?)Total(o)).Average(),   // null for empty group
        MaxCheck  = grp.Select(o => (decimal?)Total(o)).Max(),       // null for empty group
        MinCheck  = grp.Select(o => (decimal?)Total(o)).Min(),       // null for empty group
    };

Console.WriteLine("\n--- Per-category summary ---");
Console.WriteLine("Category | Count | Revenue    | AvgCheck   | MaxCheck   | MinCheck");
foreach (var s in byCategory)
    Console.WriteLine(
        $"{s.Category,-9} | {s.Count,5} | {s.Revenue,10:C} | " +
        $"{(s.AvgCheck?.ToString("C") ?? "null"),10} | " +
        $"{(s.MaxCheck?.ToString("C") ?? "null"),10} | " +
        $"{(s.MinCheck?.ToString("C") ?? "null"),10}");

// --- Aggregate with seed: fold into a composite object ---
var weekly = orders.Aggregate(
    seed: (Count: 0, Revenue: 0m, MaxOrder: decimal.MinValue, MinOrder: decimal.MaxValue, MaxQuantity: 0),
    func: (acc, o) => (
        Count:       acc.Count + 1,
        Revenue:     acc.Revenue + Total(o),
        MaxOrder:    Math.Max(acc.MaxOrder, Total(o)),
        MinOrder:    Total(o) == 0m ? acc.MinOrder : Math.Min(acc.MinOrder, Total(o)),
        MaxQuantity: Math.Max(acc.MaxQuantity, o.Quantity)));
Console.WriteLine(
    $"\n--- Aggregate fold ---\n" +
    $"Count={weekly.Count}, Revenue={weekly.Revenue:C}, MaxOrder={weekly.MaxOrder:C}, " +
    $"MinOrder={weekly.MinOrder:C}, MaxQuantity={weekly.MaxQuantity}");

// --- Seed safety on an empty collection ---
var empty = Array.Empty<Order>();
var emptyFold = empty.Aggregate(
    seed: (Count: 0, Revenue: 0m, MaxOrder: decimal.MinValue, MinOrder: decimal.MaxValue, MaxQuantity: 0),
    func: (acc, o) => acc);  // never invoked
Console.WriteLine($"Empty fold = ({emptyFold.Count}, {emptyFold.Revenue}) — no throw");

// --- Aggregate with resultSelector: customer string ---
string customers = orders
    .Select(o => o.Customer)
    .Distinct()
    .Aggregate(
        seed: "Customers: ",
        func: (acc, name) => acc + name + ", ",
        resultSelector: acc => acc.TrimEnd(',', ' '));
Console.WriteLine(customers);

// --- Unsafe variant (comment only!) ---
// orders.Where(o => o.Category == "Teapots").Min(o => Total(o)); // InvalidOperationException!
```

Line-by-line walk-through. The `Order` model is a `record` with nullable `Price` and `DiscountPercent` fields, which immediately mirrors real analytics where some data is missing. The test data, built with a collection expression `[…]`, includes four edge cases: an order with `Price == null` (№6), an order with `Quantity == 0` (№7), several orders with `DiscountPercent == null`, and the deliberately absent `"Teapots"` category. The `Total` method uses a switch expression with property patterns: the `{ Price: null }` arm returns `0m`, the arm with `DiscountPercent: null` ignores the discount, and the general arm applies the discount. This lets every aggregate reuse `Total` without duplication.

The overall metrics use the selector overloads (`orders.Sum(Total)`) — a single pass and readability, exactly as the lesson's best practice demands. `Count` is taken as the `List<T>` property (O(1)), while `LongCount()` is the LINQ method, to demonstrate both. The crucial point is that the extremes `maxOrder`, `minOrder`, `avgCheck` for a possibly-empty subset (the `"Teapots"` category) go through `Select(o => (decimal?)Total(o))`: that nullable projection is precisely what makes `Min`/`Max`/`Average` safe and return `null` instead of `InvalidOperationException`. Contrast this with the commented unsafe line at the bottom — it would throw.

The per-category summary uses `join … into grp` (which is `GroupJoin`) against the reference list `allCategories`, so the empty `"Teapots"` category also appears in the result with `Count == 0` and `null` metrics. All five aggregates (`Count`, `Sum`, `Average`, `Max`, `Min`) are computed in a single LINQ chain through an anonymous object — the declarative and efficient style the lesson recommends. Note that `Sum(Total)` is safe even for an empty group, because `Sum` of an empty sequence is `0`, not a throw.

`Aggregate` with a `seed` folds all orders into a five-field tuple in a single pass. Choosing `MaxOrder: decimal.MinValue` and `MinOrder: decimal.MaxValue` as neutral elements is the standard technique for extremes during a fold: the first real element is guaranteed to overwrite them through `Math.Max`/`Math.Min`. The special condition `Total(o) == 0m ? acc.MinOrder : Math.Min(...)` excludes orders with zero contribution (from `Price == null` or `Quantity == 0`) from the minimum computation, so `MinOrder` does not end up as `0m` because of an "empty" order. The `empty.Aggregate(seed, func)` demonstration confirms the safety of the seeded form on an empty collection: `func` is never invoked and the `seed` is returned — exactly the guarantee the lesson names as the main advantage of `Aggregate(seed, func)` over `Aggregate(func)`.

Finally, `Aggregate` with a `resultSelector` builds the customer string: `seed "Customers: "`, accumulation `acc + name + ", "`, and the final projection `TrimEnd(',', ' ')` removes the trailing separator. This is a direct replay of the lesson example, but on the homework data. Overall the solution covers every lesson topic: the basic aggregates, `LongCount`, nullable semantics, empty sequences, all three `Aggregate` forms, `GroupBy`/`GroupJoin` plus aggregates, the choice between the `Count` property and the `Count()` method, and the single-pass `Aggregate` as an alternative to a chain of separate aggregates.

#### Going deeper (bonus)
1. **One pass instead of four.** Replace the separate `Sum`+`Min`+`Max`+`Average` over a single category with one `Aggregate` into a tuple, so the group is traversed once. Measure the difference with `Stopwatch` over 1,000,000 orders and discuss whether the optimization is worth the readability loss.
2. **Sliding window.** Implement a method that for each weekday computes the `Average` check for that day and the previous day (a window of width 2). Think about how `Aggregate` with a tuple `seed` accumulator helps carry "the previous day".
3. **A custom aggregate monad.** Through `Aggregate(seed, func, resultSelector)` build a histogram of check distribution across the buckets `[0–10), [10–50), [50–100), [100+)` as a `Dictionary<string,int>`. Discuss why `Aggregate` is more appropriate here than a chain of `Where(...).Count()` calls.
4. **Thread-safe aggregation.** Investigate `ParallelEnumerable` and the `Aggregate` overload for parallel folding (seed factory, updateAccumulatorFunc, combineAccumulatorsFunc, resultSelector). Explain why separate local accumulators and their combination are necessary.

---

#### Чек-лист сдачи / Submission checklist
- [ ] (RU) Проект собирается командой `dotnet build` без ошибок и предупреждений уровня error.
- [ ] (RU) В `.csproj` включены `<Nullable>enable</Nullable>` и `<LangVersion>latest</LangVersion>`.
- [ ] (RU) Модель `Order` содержит 12+ записей с крайними случаями.
- [ ] (RU) Метод `Total` корректно обрабатывает `null` через pattern matching.
- [ ] (RU) Общие агрегаты выведены: `Count`, `LongCount`, `Sum`, `Max`, `Min`, `Average`.
- [ ] (RU) `Min`/`Max`/`Average` используют nullable-проекцию и не падают на пустых выборках.
- [ ] (RU) Таблица по категориям построена через `GroupBy`/`GroupJoin` + 5 агрегатов.
- [ ] (RU) `Aggregate` с `seed` сворачивает заказы в кортеж из 5 полей.
- [ ] (RU) Демонстрируется безопасность `Aggregate(seed, func)` на пустой коллекции.
- [ ] (RU) `Aggregate` с `resultSelector` строит строку клиентов.
- [ ] (RU) В выводе `dotnet run` видны все секции и обработанные крайние случаи.
- [ ] (EN) The project builds with `dotnet build` with no errors or error-level warnings.
- [ ] (EN) The `.csproj` enables `<Nullable>enable</Nullable>` and `<LangVersion>latest</LangVersion>`.
- [ ] (EN) The `Order` model has 12+ records with edge cases.
- [ ] (EN) The `Total` method handles `null` correctly via pattern matching.
- [ ] (EN) Overall aggregates are printed: `Count`, `LongCount`, `Sum`, `Max`, `Min`, `Average`.
- [ ] (EN) `Min`/`Max`/`Average` use a nullable projection and do not throw on empty subsets.
- [ ] (EN) The per-category table is built with `GroupBy`/`GroupJoin` + 5 aggregates.
- [ ] (EN) `Aggregate` with a `seed` folds orders into a 5-field tuple.
- [ ] (EN) `Aggregate(seed, func)` safety on an empty collection is demonstrated.
- [ ] (EN) `Aggregate` with a `resultSelector` builds the customer string.
- [ ] (EN) The `dotnet run` output shows every section and handled edge case.

#### Ресурсы / Resources
- [Microsoft Learn — Enumerable.Sum](https://learn.microsoft.com/dotnet/api/system.linq.enumerable.sum)
- [Microsoft Learn — Enumerable.Min / Max](https://learn.microsoft.com/dotnet/api/system.linq.enumerable.min)
- [Microsoft Learn — Enumerable.Average](https://learn.microsoft.com/dotnet/api/system.linq.enumerable.average)
- [Microsoft Learn — Enumerable.Count / LongCount](https://learn.microsoft.com/dotnet/api/system.linq.enumerable.count)
- [Microsoft Learn — Enumerable.Aggregate](https://learn.microsoft.com/dotnet/api/system.linq.enumerable.aggregate)
- [Microsoft Learn — GroupBy](https://learn.microsoft.com/dotnet/api/system.linq.enumerable.groupby)

---
