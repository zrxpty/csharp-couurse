---
[← К уроку M08-L04](lesson-M08-L04-groupby-tolookup.md) | [⬆ К модулю M08](../README.md) | [Следующее ДЗ →](homework-M08-L05-join-zip.md)
---

### Домашнее задание M08-L04: GroupBy, ToLookup / Homework M08-L04: GroupBy, ToLookup

**Урок / Lesson:** M08-L04
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 3/5
**Цель / Goal:** (RU) На практике закрепить отложенную группировку `GroupBy` и немедленное построение индекса `ToLookup`, научиться выбирать между `IEnumerable<IGrouping<,>>` и `ILookup<,>` по сценарию, применять `elementSelector` и `IEqualityComparer<TKey>`, избегать многократного перечисления и ловушки группировки по типу без переопределённого равенства. (EN) Get hands-on practice with the deferred `GroupBy` operator and the eager `ToLookup` index builder, learn to choose between `IEnumerable<IGrouping<,>>` and `ILookup<,>` based on the scenario, apply `elementSelector` and `IEqualityComparer<TKey>`, avoid multiple enumeration and the trap of grouping by a type with no equality overridden.

#### Связь с уроком / Connection to the lesson
(RU) Урок объясняет «склад-и-полки» как модель группировки, показывает, что `GroupBy` отложен и возвращает `IEnumerable<IGrouping<TKey,TElement>>`, а `ToLookup` строит готовый неизменяемый `ILookup<TKey,TElement>` с безопасным индексатором. Это ДЗ заставит вас применить обе операции в одном связном проекте мини-аналитики заказов: вы построите конвейер `GroupBy → Select` для отчётов и параллельно `ToLookup` для частых запросов «заказы клиента», столкнётесь с многократным перечислением и компаратором, а также с ловушкой группировки по классу без `Equals`/`GetHashCode`. (EN) The lesson explains the “warehouse-and-shelves” model of grouping, shows that `GroupBy` is deferred and returns `IEnumerable<IGrouping<TKey,TElement>>`, while `ToLookup` builds a ready immutable `ILookup<TKey,TElement>` with a safe indexer. This homework makes you apply both operations in one cohesive mini order-analytics project: you will build a `GroupBy → Select` pipeline for reports and, in parallel, a `ToLookup` for frequent “orders of a customer” queries, encounter multiple enumeration and a comparer, plus the trap of grouping by a class without `Equals`/`GetHashCode`.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация
Вы присоединились к команде учебного сервиса аналитики интернет-магазина. Склад ежедневно присылает плоский список заказов — каждый заказ это идентификатор, код клиента, город доставки, категория товара и сумма. Бизнес-аналитикам нужны два совершенно разных представления одних и тех же данных. Во-первых, им нужен одноразовый отчёт: сколько заказов и на какую сумму пришло в each город и в each категорию, кто из клиентов оказался самым «дорогим», какие города прислали чётное и нечётное количество заказов. Это классическая задача для отложенного конвейера `GroupBy`: вы строите запрос, композируете его с `Select`, `OrderBy`, `Sum`, и только в момент `foreach` или `ToList()` группировка фактически выполняется. Во-вторых, бизнесу нужен постоянный индекс «код клиента → все его заказы», к которому они будут обращаться десятки раз за сессию — отвечать на вопрос «покажи заказы клиента C2» за `O(1+k)`, без повторного сканирования списка. Здесь отложенный `GroupBy` вреден: каждый запрос пересчитывал бы группы заново. Правильный инструмент — `ToLookup`, который немедленно строит неизменяемую мульти-словарную структуру с безопасным индексатором, не выбрасывающим `KeyNotFoundException`. Таким образом, в одном задании вы проживаете ключевую развилку урока: «дальше строим конвейер — `GroupBy`; нужна готовая структура для частых запросов по ключу — `ToLookup`». Дополнительно вы столкнётесь с тремя подводными камнями, которые урок выделяет особо: многократное перечисление `IEnumerable<IGrouping>` без `ToList`, регистронезависимая группировка по городу через `StringComparer.OrdinalIgnoreCase` и знаменитая ловушка «группировка по классу без переопределённого равенства», где каждая запись уходит в отдельную группу. В финале вы должны не просто написать рабочий код, а уметь обосновать, почему в каждой точке выбран именно `GroupBy` или именно `ToLookup` — это и есть осознанное владение темой.

#### Что нужно сделать (пошагово)
1. Создайте новый консольный проект на .NET 8:
   ```
   dotnet new console -n OrderAnalytics -f net8.0
   cd OrderAnalytics
   ```
   Убедитесь, что в `OrderAnalytics.csproj` стоит `<TargetFramework>net8.0</TargetFramework>` и язык C# 12 (при желании добавьте `<LangVersion>latest</LangVersion>`).

2. В файле `Order.cs` опишите модель заказа как `sealed record`:
   ```csharp
   public sealed record Order(int OrderId, string CustomerId, string City, string Category, decimal Total);
   ```
   Запись (`record`) автоматически даст корректные `Equals`/`GetHashCode` на основе значений — это пригодится в бонусном шаге, где вы намеренно покажете, что было бы с обычным `class`.

3. В `Program.cs` (top-level statements) задайте исходный список заказов через collection expression:
   ```csharp
   List<Order> orders =
   [
       new(1, "C1", "Moscow",      "Books",   100m),
       new(2, "C2", "Kazan",       "Toys",    250m),
       new(3, "C1", "moscow",      "Books",    75m),  // другой регистр города
       new(4, "C3", "Kazan",       "Books",   300m),
       new(5, "C2", "Moscow",      "Toys",     50m),
       new(6, "C1", "Novosibirsk", "Books",   400m),
       new(7, "C3", "kazan",       "Toys",    120m),  // другой регистр города
   ];
   ```
   Обратите внимание на намеренный разброс регистра в `City` — он понадобится для шага с компаратором.

4. Постройте отчёт «город → количество заказов и сумма» через `GroupBy` с `elementSelector`:
   ```csharp
   var byCity = orders
       .GroupBy(o => o.City, o => o.Total)
       .Select(g => new { City = g.Key, Count = g.Count(), Sum = g.Sum() })
       .OrderByDescending(r => r.Sum);
   ```
   Выведите результат через `foreach`. Объясните в комментарии, почему `elementSelector` (`o => o.Total`) здесь уместен: группа хранит только суммы, а не целые объекты `Order`, что экономит память и упрощает дальнейшую агрегацию.

5. Демонстрация многократного перечисления: сознательно «наступите на грабли» урока. Сначала покажите **плохой** вариант:
   ```csharp
   var grouped = orders.GroupBy(o => o.CustomerId); // ещё не выполнено
   if (grouped.Any()) { foreach (var g in grouped) { /* ... */ } } // группировка дважды!
   ```
   В комментарии поясните: `Any()` запускает группировку, затем `foreach` запускает её снова. Затем покажите **правильный** вариант с материализацией через `ToList()`. Выведите количество групп до и после — оно одинаково, но правильный вариант работает за одну группировку.

6. Постройте индекс «клиент → заказы» через `ToLookup`:
   ```csharp
   ILookup<string, Order> byCustomer = orders.ToLookup(o => o.CustomerId);
   ```
   В цикле обойдите `byCustomer` (он перечислим как `IGrouping`), а затем выполните три точечных запроса: `byCustomer["C1"]`, `byCustomer["C2"]` и `byCustomer["NO_SUCH_CUSTOMER"]`. Выведите количество заказов для каждого. Убедитесь и прокомментируйте, что последний запрос **не выбрасывает** `KeyNotFoundException` — возвращается пустая последовательность. Это ключевое свойство `ILookup` из урока.

7. Регистронезависимая группировка по городу через компаратор. Используя `StringComparer.OrdinalIgnoreCase`, постройте версию отчёта из шага 4, где `"Moscow"` и `"moscow"` сливаются в одну группу:
   ```csharp
   var byCityCi = orders
       .GroupBy(o => o.City, StringComparer.OrdinalIgnoreCase)
       .Select(g => (City: g.Key, Count: g.Count(), Sum: g.Sum()))
       .OrderBy(t => t.City);
   ```
   Выведите результат. Сравните с шагом 4: раньше `Moscow` и `moscow` были разными ключами, теперь это одна группа. В комментарии отметьте, что компаратор передаётся в `GroupBy` напрямую, без ручной нормализации `ToLower()`.

8. Демонстрация ловушки «группировка по классу без равенства». В отдельном файле `PlainOrder.cs` объявите обычный `class` (НЕ `record`) с теми же полями, но **без** переопределённых `Equals`/`GetHashCode`. Сгруппируйте список таких объектов по `CustomerId` и выведите количество групп. Вы увидите, что оно равно числу объектов, а не числу уникальных клиентов, — каждая запись ушла в свою группу, потому что ссылочное равенство не считает два разных экземпляра равными. В комментарии объясните механизм и предложите два исправления: переопределить равенство или использовать компаратор. Сравните с `record Order`, где этой проблемы нет.

9. Соберите и запустите проект:
   ```
   dotnet build
   dotnet run
   ```
   Ожидаемый вывод должен содержать: отчёт «город → count/sum», пояснение о двойной группировке и её устранении через `ToList`, отчёт по `ToLookup` с тремя точечными запросами (включая пустой результат для несуществующего клиента), регистронезависимый отчёт со слившимися городами, и демонстрацию развалившейся группировки по классу.

10. (Опционально, но желательно) Добавьте проверки-инварианты через `Debug.Assert` или `if`-проверки с понятным сообщением: например, что `byCustomer["NO_SUCH_CUSTOMER"].Count() == 0`, что в регистронезависимом отчёте ровно три города (Moscow, Kazan, Novosibirsk), что количество групп в правильном шаге 5 равно числу уникальных клиентов.

#### Требования к решению
- Целевая платформа — .NET 8, язык C# 12. Используйте top-level statements в `Program.cs`, pattern matching там, где он уместен, collection expressions для инициализации списка заказов и raw string literals для многострочных литералов в выводе, если они нужны.
- Модель `Order` — `sealed record` с позиционным синтаксисом; это автоматически обеспечивает корректные `Equals`/`GetHashCode` на основе значений. Для демонстрации ловушки (шаг 8) используется отдельный `class PlainOrder` **без** переопределённого равенства.
- В отчёте «город → сумма» обязательно применяется `elementSelector`: группа хранит только `decimal Total`, а не весь `Order`. Это best practice из урока — экономия памяти и упрощение downstream-кода.
- Демонстрация многократного перечисления (шаг 5) содержит **оба** варианта: плохой (двойная группировка) и хороший (`ToList`). Плохой вариант снабжён комментарием с объяснением. Допускается «спрятать» плохой вариант за флагом `const bool demonstrateBug = true;`, чтобы он не портил основной вывод.
- `ToLookup` используется именно там, где нужен многократный точечный доступ по ключу (шаг 6). Запрос к несуществующему ключу должен возвращать пустую последовательность без исключения — это проверяется явно.
- Регистронезависимая группировка (шаг 7) передаёт `StringComparer.OrdinalIgnoreCase` в `GroupBy` напрямую, без `ToLower()` в ключевой функции. Компаратор применяется именно к строковому ключу, а не к элементу.
- Демонстрация ловушки (шаг 8) честно показывает развалившуюся группировку: число групп равно числу объектов, а не числу уникальных клиентов. Комментарий объясняет причину (ссылочное равенство) и предлагает два исправления.
- Код компилируется без предупреждений компилятора (или с обоснованными) и запускается. Вывод `dotnet run` соответствует ожидаемому.
- Запрещено подменять `GroupBy` ручным `Dictionary<TKey, List<T>>` в основных шагах 4–6: цель — освоить именно операторы LINQ. Ручной словарь допустим только в бонусных задачах для сравнения производительности.

#### Тонкости и подводные камни
- **Отложенность — это не бесплатный подарок.** `GroupBy` ничего не делает до первого перечисления. Это даёт гибкость в композиции (`Select`, `OrderBy`, `Sum` навешиваются сверху и оптимизируются вместе), но превращается в ловушку, когда результат обходят дважды: `if (q.Any()) { foreach (var g in q) ... }` группирует дважды. Правило урока: собираетесь обходить больше одного раза — сразу `ToList()` или `ToDictionary()`/`ToLookup()`.
- **`IGrouping` — это «полка с этикеткой».** Он одновременно наследует `IEnumerable<TElement>` и несёт свойство `Key`. Поэтому внутри цикла вы пишете `g.Key` и перебираете `g` через `foreach`. Не пересчитывайте ключ заново из элемента — он уже есть на «этикетке».
- **`elementSelector` vs `resultSelector`.** `elementSelector` (второй аргумент-селектор у `GroupBy`) решает, **что хранить в группе**: весь объект или только нужное поле. `resultSelector` (перегрузка `GroupBy` с тремя аргументами) решает, **во что превратить каждую группу** в финальной проекции. Не путайте их: первый урезает содержимое корзины, второй — форму вывода.
- **`ToLookup` безопасен по индексатору.** `lookup[key]` для отсутствующего ключа возвращает пустую последовательность, а не бросает `KeyNotFoundException`. Это удобно, но означает, что вы не можете отличить «ключа нет» от «ключ есть, но значений ноль» через один индексатор — для проверки существования используйте `lookup.Contains(key)`.
- **`ILookup` неизменяем.** После построения его нельзя изменить: ни добавить, ни удалить пару. Если исходная коллекция изменилась — индекс этого не отразит. Для меняющихся данных `ToLookup` не подходит; там нужен пересчёт или ручной `Dictionary<TKey, List<T>>`.
- **Группировка по классу без равенства — главная ловушка.** `class` по умолчанию использует ссылочное равенство: два разных экземпляра с одинаковыми полями не равны, и каждый уходит в свою группу. `record` этой проблемы лишён (у него value-семантика из коробки). Для `class` либо переопределяйте `Equals`/`GetHashCode`, либо передавайте `IEqualityComparer<TKey>`.
- **Компаратор передаётся в оператор, а не вешается на ключ.** Для регистронезависимой группировки по строке правильное решение — `GroupBy(o => o.City, StringComparer.OrdinalIgnoreCase)`, а не `o => o.City.ToLower()`. Второй подход засоряет код, ломает исходный регистр в `g.Key` и порождает баги при смешивании культур.
- **Группировка — это не агрегация.** `GroupBy` лишь готовит корзины; сами агрегаты (`Count`, `Sum`, `Average`) вычисляются поверх элементов группы. Не ждите от `GroupBy` финального числа — это промежуточная структура.
- **`ToLookup` vs `Dictionary<TKey, List<T>>`.** Урок рекомендует `ToLookup`: он неизменяем, безопасно обрабатывает отсутствующие ключи и яснее выражает намерение «мульти-индекс по ключу». Ручной словарь списков уместен, только когда нужна изменяемость.

#### Критерии приёмки
- [ ] Проект `OrderAnalytics` собирается под .NET 8 / C# 12 без ошибок и предупреждений компилятора.
- [ ] `Order` объявлен как `sealed record` с позиционным синтаксисом; `PlainOrder` — обычный `class` без переопределённого равенства.
- [ ] Список заказов инициализирован через collection expression и содержит минимум два города в разном регистре.
- [ ] Отчёт «город → count/sum» построен через `GroupBy` с `elementSelector` (`o => o.Total`) и `Select`; результат отсортирован и выведен.
- [ ] В комментарии пояснено, почему `elementSelector` экономит память и упрощает downstream-код.
- [ ] Шаг 5 содержит **плохой** вариант (`Any()` + `foreach` без материализации) с комментарием о двойной группировке и **хороший** вариант с `ToList()`.
- [ ] `ToLookup` построен по `CustomerId` и обойдён через `foreach` как `IGrouping`.
- [ ] Точечные запросы `byCustomer["C1"]`, `byCustomer["C2"]` возвращают корректные количества заказов.
- [ ] Запрос `byCustomer["NO_SUCH_CUSTOMER"]` возвращает пустую последовательность без исключения; это явно проверено и прокомментировано.
- [ ] Регистронезависимая группировка передаёт `StringComparer.OrdinalIgnoreCase` в `GroupBy` напрямую (без `ToLower()`), и города `"Moscow"`/`"moscow"` сливаются в одну группу.
- [ ] Демонстрация ловушки показывает, что число групп по `PlainOrder` равно числу объектов, а не числу уникальных клиентов; комментарий объясняет причину и предлагает два исправления.
- [ ] Код компилируется без предупреждений и запускается; вывод `dotnet run` соответствует ожидаемому.
- [ ] (Опционально) Добавлены проверки-инварианты через `Debug.Assert` или `if` с понятным сообщением.
- [ ] В финальном комментарии или readme-блоке кратко обоснован выбор `GroupBy` vs `ToLookup` в каждой точке решения.

#### Подсказки (без прямого ответа)
- `elementSelector` — это второй аргумент `GroupBy` вида `o => o.Total`; он возвращает то, что попадёт в группу, а не ключ. Ключ остаётся первым аргументом `o => o.City`.
- Чтобы «наступить на грабли» многократного перечисления, достаточно обойти `IEnumerable<IGrouping>` сначала через `Any()`, а затем через `foreach`. Чтобы «отступить» — обернуть результат в `ToList()` сразу после `GroupBy`.
- `ILookup` перечислим: `foreach (var g in lookup)` даёт `IGrouping<TKey, TElement>`, у которого есть `g.Key` и сам `g` как `IEnumerable<TElement>`.
- `lookup[key]` для отсутствующего ключа возвращает пустую `IEnumerable<TElement>`; `lookup.Contains(key)` — булево наличие ключа.
- `StringComparer.OrdinalIgnoreCase` — это готовый `IEqualityComparer<string>`; его можно передать вторым аргументом в `GroupBy(o => key, comparer)` или `ToLookup(o => key, comparer)`.
- Для демонстрации ловушки не переопределяйте в `PlainOrder` ни `Equals`, ни `GetHashCode` — именно их отсутствие и должно вызвать развал.
- `record` автоматически получает value-based `Equals`/`GetHashCode`; поэтому `Order` из шага 2 группируется корректно, а `PlainOrder` из шага 8 — нет. Сравните их поведение в одном выводе.
- Чтобы сортировать группы, навесьте `OrderBy`/`OrderByDescending` **после** `Select` — так вы сортируете уже спроецированные строки отчёта, а не сами `IGrouping`.

#### Эталонное решение (разбор)
```csharp
// Order.cs — RU: модель-рекорд с value-семантикой; EN: record model with value semantics.
// RU: record автоматически реализует Equals/GetHashCode на основе значений — группировка работает корректно.
// EN: record auto-implements Equals/GetHashCode by value — grouping works correctly.
public sealed record Order(int OrderId, string CustomerId, string City, string Category, decimal Total);

// PlainOrder.cs — RU: намеренно «глупой» класс без равенства; EN: intentionally dumb class without equality.
// RU: ссылочное равенство по умолчанию — каждый экземпляр уникален, группировка разваливается.
// EN: default reference equality — every instance is unique, grouping falls apart.
public sealed class PlainOrder
{
    public int OrderId { get; init; }
    public string CustomerId { get; init; } = "";
    public string City { get; init; } = "";
    public string Category { get; init; } = "";
    public decimal Total { get; init; }
}

// Program.cs — top-level statements, C# 12 / .NET 8.
using System;
using System.Collections.Generic;
using System.Linq;

List<Order> orders =
[
    new(1, "C1", "Moscow",      "Books", 100m),
    new(2, "C2", "Kazan",       "Toys",  250m),
    new(3, "C1", "moscow",      "Books",  75m),  // RU: другой регистр города  EN: different case of city
    new(4, "C3", "Kazan",       "Books", 300m),
    new(5, "C2", "Moscow",      "Toys",   50m),
    new(6, "C1", "Novosibirsk", "Books", 400m),
    new(7, "C3", "kazan",       "Toys",  120m),  // RU: другой регистр города  EN: different case of city
];

// 1) GroupBy с elementSelector: группа хранит только Total, не весь Order.
//    GroupBy with elementSelector: the group stores only Total, not the whole Order.
var byCity = orders
    .GroupBy(o => o.City, o => o.Total)                      // RU: ключ — город, элемент — сумма
    .Select(g => new { City = g.Key, Count = g.Count(), Sum = g.Sum() })
    .OrderByDescending(r => r.Sum);

Console.WriteLine("GroupBy по городу (elementSelector = Total) / GroupBy by city:");
foreach (var row in byCity)
    Console.WriteLine($"  {row.City}: orders={row.Count}, sum={row.Sum}");

// 2) Многократное перечисление: плохой и хороший варианты.
//    Multiple enumeration: bad and good variants.
const bool demonstrateBug = true;
if (demonstrateBug)
{
    // RU: ПЛОХО — Any() и foreach дважды запускают группировку.
    // EN: BAD — Any() and foreach trigger grouping twice.
    var grouped = orders.GroupBy(o => o.CustomerId);        // ещё не выполнено / not executed yet
    Console.WriteLine($"\n[BUG] Any() => {grouped.Any()} (первая группировка)");
    foreach (var g in grouped)                               // вторая группировка
        Console.WriteLine($"  {g.Key}: {g.Count()} order(s)");
}

// RU: ХОРОШО — материализация через ToList фиксирует результат.
// EN: GOOD — materialization via ToList pins the result.
var pinned = orders.GroupBy(o => o.CustomerId).ToList();    // выполнено один раз / executed once
Console.WriteLine("\n[OK] ToList материализовал группы:");
foreach (var g in pinned)
    Console.WriteLine($"  {g.Key}: {g.Count()} order(s)");

// 3) ToLookup — немедленный индекс «клиент → заказы».
//    ToLookup — eager index "customer → orders".
ILookup<string, Order> byCustomer = orders.ToLookup(o => o.CustomerId);

Console.WriteLine("\nToLookup по CustomerId:");
foreach (var g in byCustomer)
{
    Console.WriteLine($"  {g.Key}: {g.Count()} order(s)");
    foreach (var o in g)
        Console.WriteLine($"    - #{o.OrderId}, {o.City}, {o.Total}");
}

// RU: индексатор не выбрасывает KeyNotFoundException — пустая последовательность.
// EN: the indexer never throws — empty sequence for a missing key.
foreach (var key in new[] { "C1", "C2", "NO_SUCH_CUSTOMER" })
    Console.WriteLine($"  lookup[{key}] => {byCustomer[key].Count()} order(s)");

// 4) Регистронезависимая группировка через компаратор.
//    Case-insensitive grouping via a comparer.
var byCityCi = orders
    .GroupBy(o => o.City, StringComparer.OrdinalIgnoreCase) // RU: компаратор прямо в GroupBy
    .Select(g => (City: g.Key, Count: g.Count(), Sum: g.Sum()))
    .OrderBy(t => t.City);

Console.WriteLine("\nGroupBy с OrdinalIgnoreCase (Москва == mOSCOW):");
foreach (var (city, count, sum) in byCityCi)
    Console.WriteLine($"  {city}: orders={count}, sum={sum}");

// 5) Ловушка: группировка по class без переопределённого равенства.
//    Trap: grouping by a class without overridden equality.
List<PlainOrder> plainOrders = orders
    .Select(o => new PlainOrder
    {
        OrderId = o.OrderId, CustomerId = o.CustomerId,
        City = o.City, Category = o.Category, Total = o.Total
    }).ToList();

var plainGroups = plainOrders.GroupBy(o => o.CustomerId);
Console.WriteLine($"\n[TRAP] PlainOrder: групп = {plainGroups.Count()}, объектов = {plainOrders.Count}");
Console.WriteLine("  RU: ссылочное равенство → каждая запись в своей группе. Исправление: record или IEqualityComparer.");
Console.WriteLine("  EN: reference equality → each record in its own group. Fix: record or IEqualityComparer.");

// 6) Инварианты.
//    Invariants.
System.Diagnostics.Debug.Assert(byCustomer["NO_SUCH_CUSTOMER"].Count() == 0, "missing key must yield empty");
System.Diagnostics.Debug.Assert(byCityCi.Count() == 3, "case-insensitive cities must merge to 3");
System.Diagnostics.Debug.Assert(pinned.Count == 3, "unique customers: C1, C2, C3");
```

Разбор по строкам. Модель `Order` объявлена `sealed record` с позиционным синтаксисом — это первое ключевое решение урока: `record` автоматически реализует `Equals`/`GetHashCode` на основе значений полей, поэтому две записи с одинаковыми `OrderId` (или, что важнее для группировки, с одинаковым ключом) считаются равными. Этому противопоставлен `PlainOrder` — обычный `class` без переопределённого равенства, у которого сравнение идет по ссылке; именно он позволит в шаге 5 честно показать ловушку «каждая запись в своей группе». Список `orders` задан через collection expression (C# 12): квадратные скобки вместо `new List<Order> { ... }` — это современная идиома, рекомендованная для .NET 8. В данные намеренно подмешаны города в разном регистре (`"moscow"`, `"kazan"`), чтобы шаг 4 (без компаратора) и шаг 7 (с компаратором) дали разные результаты.

Шаг 1 — классический конвейер `GroupBy → Select → OrderByDescending`. Здесь применены сразу две концепции урока. Во-первых, `elementSelector` (`o => o.Total`): группа хранит только суммы, а не целые объекты `Order`. Урок прямо рекомендует это как способ сэкономить память и упростить downstream-код — внутри `Select` мы вызываем `g.Count()` и `g.Sum()` уже над `IEnumerable<decimal>`, а не над `IEnumerable<Order>`. Во-вторых, отложенность: `GroupBy` ничего не делает до `foreach` в выводе. Если бы мы обошли `byCity` дважды, группировка выполнилась бы дважды — именно это демонстрирует шаг 2.

Шаг 2 — демонстрация многократного перечисления, одного из главных предупреждений урока. Плохой вариант `if (grouped.Any()) { foreach (var g in grouped) ... }` запускает группировку дважды: первый раз `Any()` ради проверки непустоты, второй раз — `foreach`. Хорошей альтернативой, согласно уроку, является `ToList()` сразу после `GroupBy`: он закрепляет результат, и последующие обходы бесплатны. Флаг `demonstrateBug` позволяет держать плохой вариант в коде как учебную демонстрацию, не портя основной вывод.

Шаг 3 — `ToLookup`. Урок определяет его как «`GroupBy`, который выполняется немедленно». В отличие от `GroupBy`, `ToLookup` сразу строит неизменяемый `ILookup<string, Order>`, который ведёт себя как мульти-словарь: одному ключу соответствует набор значений. Обход через `foreach` даёт `IGrouping<string, Order>` — ту же «полку с этикеткой», что и у `GroupBy`. Но главное свойство — индексатор `byCustomer[key]`: он **никогда** не выбрасывает `KeyNotFoundException`, для отсутствующего ключа возвращается пустая последовательность. Это проверяется в цикле по трём ключам, включая `"NO_SUCH_CUSTOMER"`. Выбор `ToLookup` здесь не случаен: бизнесу нужен многократный точечный доступ «заказы клиента», и пересчитывать группы на каждый запрос было бы расточительно. `GroupBy` тут был бы ошибкой.

Шаг 4 — регистронезависимая группировка через `StringComparer.OrdinalIgnoreCase`. Урок подчёркивает, что компаратор передаётся в оператор напрямую, а не «вешается» на ключ через `ToLower()`. Результат: `"Moscow"` и `"moscow"` сливаются в одну группу, и `g.Key` сохраняет исходный регистр первого встреченного элемента. Это и best practice (нет разбросанных `ToLower()` по коду), и требование корректности (нет риска рассинхронизации культур). Сравните с шагом 1, где без компаратора `Moscow` и `moscow` были разными ключами — наглядная разница.

Шаг 5 — ловушка «группировка по классу без равенства». `PlainOrder` — обычный `class`, у него нет переопределённых `Equals`/`GetHashCode`, поэтому по умолчанию используется ссылочное равенство. Два разных экземпляра с одинаковым `CustomerId` не равны, и `GroupBy` кладёт каждый в отдельную группу. Вывод `групп = 7, объектов = 7` (а не `групп = 3`) — это и есть «развал». Урок предлагает два исправления: сделать тип `record` (как `Order`) или передать `IEqualityComparer<TKey>`. Комментарий в коде фиксирует оба пути. Инварианты через `Debug.Assert` закрепляют ключевые свойства: пустой результат для отсутствующего ключа, ровно три города в регистронезависимом отчёте, ровно три уникальных клиента в `pinned`. Вместе все шаги проводят ученика по главной развилке урока: `GroupBy` — когда строим конвейер; `ToLookup` — когда нужен готовый индекс для частых запросов.

#### Задания на углубление (бонус)
1. Составной ключ. Постройте отчёт «год + месяц → сумма заказов», используя анонимный тип `new { o.OrderDate.Year, o.OrderDate.Month }` как ключ `GroupBy`. Добавьте к `Order` свойство `DateTime OrderDate` и заполните данные. Объясните, почему анонимный тип корректно работает как ключ (у него value-based `Equals`/`GetHashCode` из коробки).
2. Сравнение производительности. Измерьте через `BenchmarkDotNet` три подхода к многократному запросу «заказы клиента» на 100 000 заказов: (а) `orders.Where(o => o.CustomerId == id).ToList()` на каждый запрос; (б) `orders.GroupBy(o => o.CustomerId)` без материализации + обход; (в) `orders.ToLookup(o => o.CustomerId)` + индексатор. Объясните, почему (в) выигрывает на большом числе запросов.
3. Собственный компаратор. Реализуйте `IEqualityComparer<string>`, который считает города равными, если они совпадают после удаления trailing slash и приведения к верхнему регистру (моделируйте нормализацию маршрутов). Передайте его в `GroupBy` и проверьте слияние `"moscow"`/`"Moscow/"`.
4. `resultSelector`. Перепишите шаг 1, используя перегрузку `GroupBy` с `resultSelector` (три аргумента: keySelector, elementSelector, resultSelector), чтобы превратить каждую группу сразу в строку отчёта без отдельного `Select`. Сравните читаемость двух вариантов.

---

## Statement in English / Постановка на английском

#### Context & motivation
You have joined the team of a training e-commerce analytics service. The warehouse sends a flat list of orders every day — each order is an identifier, a customer code, a delivery city, a product category, and a total. The business analysts need two very different views of the same data. First, they need a one-off report: how many orders and what total amount arrived in each city and in each category, which customer turned out to be the most “expensive”, which cities sent an even or odd number of orders. This is a classic task for the deferred `GroupBy` pipeline: you build a query, compose it with `Select`, `OrderBy`, `Sum`, and only at the moment of `foreach` or `ToList()` does grouping actually execute. Second, the business needs a persistent index “customer code → all their orders”, which they will query dozens of times per session — answering “show me the orders of customer C2” in `O(1+k)` without rescanning the list. Here a deferred `GroupBy` is harmful: every query would recompute the groups from scratch. The right tool is `ToLookup`, which eagerly builds an immutable multi-dictionary structure with a safe indexer that never throws `KeyNotFoundException`. Thus, in a single assignment you live through the central fork of the lesson: “keep composing a pipeline — `GroupBy`; need a ready structure for frequent key lookups — `ToLookup`”. On top of that, you will run into three pitfalls the lesson singles out: multiple enumeration of `IEnumerable<IGrouping>` without `ToList`, case-insensitive grouping by city through `StringComparer.OrdinalIgnoreCase`, and the famous trap “grouping by a class without overridden equality” where every record lands in its own group. By the end you must not merely write working code, but be able to justify why `GroupBy` or `ToLookup` was chosen at each point — that is what deliberate mastery of the topic looks like.

#### What to do step by step
1. Create a new console project on .NET 8:
   ```
   dotnet new console -n OrderAnalytics -f net8.0
   cd OrderAnalytics
   ```
   Make sure `OrderAnalytics.csproj` contains `<TargetFramework>net8.0</TargetFramework>` and C# 12 (you may add `<LangVersion>latest</LangVersion>` to be safe).

2. In `Order.cs` describe the order model as a `sealed record`:
   ```csharp
   public sealed record Order(int OrderId, string CustomerId, string City, string Category, decimal Total);
   ```
   A `record` automatically gives correct `Equals`/`GetHashCode` based on values — this will help in the bonus step where you deliberately show what would happen with a plain `class`.

3. In `Program.cs` (top-level statements) define the source order list with a collection expression:
   ```csharp
   List<Order> orders =
   [
       new(1, "C1", "Moscow",      "Books",   100m),
       new(2, "C2", "Kazan",       "Toys",    250m),
       new(3, "C1", "moscow",      "Books",    75m),  // different case of city
       new(4, "C3", "Kazan",       "Books",   300m),
       new(5, "C2", "Moscow",      "Toys",     50m),
       new(6, "C1", "Novosibirsk", "Books",   400m),
       new(7, "C3", "kazan",       "Toys",    120m),  // different case of city
   ];
   ```
   Note the deliberate case variation in `City` — it is needed for the comparer step.

4. Build the “city → order count and total” report with `GroupBy` and an `elementSelector`:
   ```csharp
   var byCity = orders
       .GroupBy(o => o.City, o => o.Total)
       .Select(g => new { City = g.Key, Count = g.Count(), Sum = g.Sum() })
       .OrderByDescending(r => r.Sum);
   ```
   Print the result with `foreach`. In a comment explain why the `elementSelector` (`o => o.Total`) is appropriate here: the group stores only totals rather than whole `Order` objects, which saves memory and simplifies downstream aggregation.

5. Multiple-enumeration demonstration: deliberately step on the lesson’s rake. First show the **bad** variant:
   ```csharp
   var grouped = orders.GroupBy(o => o.CustomerId); // not executed yet
   if (grouped.Any()) { foreach (var g in grouped) { /* ... */ } } // grouping twice!
   ```
   In a comment explain: `Any()` triggers grouping, then `foreach` triggers it again. Then show the **good** variant with materialisation through `ToList()`. Print the group count before and after — it is the same, but the good variant runs grouping only once.

6. Build the “customer → orders” index with `ToLookup`:
   ```csharp
   ILookup<string, Order> byCustomer = orders.ToLookup(o => o.CustomerId);
   ```
   In a loop iterate `byCustomer` (it is enumerable as `IGrouping`), then issue three point lookups: `byCustomer["C1"]`, `byCustomer["C2"]`, and `byCustomer["NO_SUCH_CUSTOMER"]`. Print the order count for each. Confirm and comment that the last lookup **does not throw** `KeyNotFoundException` — an empty sequence is returned. This is the key property of `ILookup` from the lesson.

7. Case-insensitive grouping by city through a comparer. Using `StringComparer.OrdinalIgnoreCase`, build a version of the step-4 report where `"Moscow"` and `"moscow"` merge into a single group:
   ```csharp
   var byCityCi = orders
       .GroupBy(o => o.City, StringComparer.OrdinalIgnoreCase)
       .Select(g => (City: g.Key, Count: g.Count(), Sum: g.Sum()))
       .OrderBy(t => t.City);
   ```
   Print the result. Compare with step 4: previously `Moscow` and `moscow` were distinct keys, now they are one group. In a comment note that the comparer is passed straight to `GroupBy`, without manual `ToLower()` normalisation.

8. Demonstration of the trap “grouping by a class without equality”. In a separate file `PlainOrder.cs` declare an ordinary `class` (NOT a `record`) with the same fields but **without** overridden `Equals`/`GetHashCode`. Group a list of such objects by `CustomerId` and print the number of groups. You will see it equals the number of objects rather than the number of unique customers — every record went into its own group because reference equality does not consider two distinct instances equal. In a comment explain the mechanism and suggest two fixes: override equality, or use a comparer. Contrast this with `record Order`, where the problem does not exist.

9. Build and run the project:
   ```
   dotnet build
   dotnet run
   ```
   The expected output must contain: the “city → count/sum” report, an explanation of the double grouping and its elimination via `ToList`, the `ToLookup` report with three point lookups (including the empty result for a missing customer), the case-insensitive report with merged cities, and the demonstration of the collapsed grouping by a class.

10. (Optional but recommended) Add invariant checks via `Debug.Assert` or `if` checks with a clear message: for example, that `byCustomer["NO_SUCH_CUSTOMER"].Count() == 0`, that the case-insensitive report has exactly three cities (Moscow, Kazan, Novosibirsk), and that the group count in the good step-5 variant equals the number of unique customers.

#### Requirements
- Target .NET 8, language C# 12. Use top-level statements in `Program.cs`, pattern matching where appropriate, collection expressions to initialise the order list, and raw string literals for long multi-line output literals if needed.
- The `Order` model is a `sealed record` with positional syntax; this automatically ensures correct value-based `Equals`/`GetHashCode`. For the trap demonstration (step 8) a separate `class PlainOrder` **without** overridden equality is used.
- The “city → sum” report must use an `elementSelector`: the group stores only `decimal Total`, not the whole `Order`. This is a best practice from the lesson — memory savings and simpler downstream code.
- The multiple-enumeration demonstration (step 5) contains **both** variants: bad (double grouping) and good (`ToList`). The bad variant carries a comment with the explanation. It is acceptable to hide the bad variant behind a `const bool demonstrateBug = true;` flag so it does not pollute the main output.
- `ToLookup` is used precisely where repeated point lookups by key are needed (step 6). A lookup of a missing key must return an empty sequence without an exception — this is checked explicitly.
- The case-insensitive grouping (step 7) passes `StringComparer.OrdinalIgnoreCase` directly to `GroupBy`, without `ToLower()` in the key selector. The comparer is applied to the string key, not to the element.
- The trap demonstration (step 8) honestly shows the collapsed grouping: the group count equals the number of objects, not the number of unique customers. A comment explains the cause (reference equality) and suggests two fixes.
- The code compiles without compiler warnings (or with justified ones) and runs. The `dotnet run` output matches the expectation.
- Do not replace `GroupBy` with a hand-rolled `Dictionary<TKey, List<T>>` in the core steps 4–6: the goal is to master the LINQ operators themselves. A hand-rolled dictionary is allowed only in the bonus tasks for performance comparison.

#### Pitfalls
- **Deferred execution is not a free gift.** `GroupBy` does nothing until the first enumeration. That gives flexibility in composition (`Select`, `OrderBy`, `Sum` are layered on top and optimised together), but it turns into a trap when the result is iterated twice: `if (q.Any()) { foreach (var g in q) ... }` groups twice. The lesson’s rule: if you plan to iterate more than once, immediately `ToList()` or `ToDictionary()`/`ToLookup()`.
- **`IGrouping` is a “labeled shelf”.** It both inherits `IEnumerable<TElement>` and carries a `Key` property. That is why inside the loop you read `g.Key` and iterate `g` with `foreach`. Do not recompute the key from an element — it is already on the “label”.
- **`elementSelector` vs `resultSelector`.** `elementSelector` (the second selector argument of `GroupBy`) decides **what to store in the group**: the whole object or only the field you need. `resultSelector` (the three-argument `GroupBy` overload) decides **what to turn each group into** in the final projection. Do not confuse them: the first trims the bucket contents, the second shapes the output.
- **`ToLookup` is safe on the indexer.** `lookup[key]` for a missing key returns an empty sequence instead of throwing `KeyNotFoundException`. That is convenient, but it also means you cannot distinguish “key is absent” from “key is present but has zero values” through the indexer alone — use `lookup.Contains(key)` to check existence.
- **`ILookup` is immutable.** Once built, it cannot be changed: no add, no remove. If the source collection changes, the index will not reflect it. For mutating data `ToLookup` is the wrong choice; you need a recompute or a hand-rolled `Dictionary<TKey, List<T>>`.
- **Grouping by a class without equality is the chief trap.** A `class` uses reference equality by default: two distinct instances with identical fields are not equal, and each lands in its own group. A `record` does not have this problem (it has value semantics out of the box). For a `class` either override `Equals`/`GetHashCode` or pass an `IEqualityComparer<TKey>`.
- **The comparer goes into the operator, not onto the key.** For case-insensitive string grouping the right answer is `GroupBy(o => o.City, StringComparer.OrdinalIgnoreCase)`, not `o => o.City.ToLower()`. The latter litters the code, breaks the original case in `g.Key`, and breeds bugs when cultures are mixed.
- **Grouping is not aggregation.** `GroupBy` only prepares the buckets; the aggregates themselves (`Count`, `Sum`, `Average`) are computed over the group’s elements. Do not expect a final number from `GroupBy` — it is an intermediate structure.
- **`ToLookup` vs `Dictionary<TKey, List<T>>`.** The lesson recommends `ToLookup`: it is immutable, handles missing keys gracefully, and communicates the intent “multi-index by key” more clearly. A hand-rolled dictionary of lists is justified only when mutability is required.

#### Acceptance criteria
- [ ] The `OrderAnalytics` project builds under .NET 8 / C# 12 with no compiler errors or warnings.
- [ ] `Order` is declared as a `sealed record` with positional syntax; `PlainOrder` is an ordinary `class` without overridden equality.
- [ ] The order list is initialised through a collection expression and contains at least two cities in different cases.
- [ ] The “city → count/sum” report is built with `GroupBy` using an `elementSelector` (`o => o.Total`) and `Select`; the result is sorted and printed.
- [ ] A comment explains why the `elementSelector` saves memory and simplifies downstream code.
- [ ] Step 5 contains a **bad** variant (`Any()` + `foreach` without materialisation) with a comment on the double grouping and a **good** variant with `ToList()`.
- [ ] `ToLookup` is built by `CustomerId` and iterated through `foreach` as `IGrouping`.
- [ ] Point lookups `byCustomer["C1"]`, `byCustomer["C2"]` return the correct order counts.
- [ ] The lookup `byCustomer["NO_SUCH_CUSTOMER"]` returns an empty sequence without an exception; this is explicitly checked and commented.
- [ ] The case-insensitive grouping passes `StringComparer.OrdinalIgnoreCase` directly to `GroupBy` (no `ToLower()`), and the cities `"Moscow"`/`"moscow"` merge into a single group.
- [ ] The trap demonstration shows that the group count for `PlainOrder` equals the number of objects rather than the number of unique customers; a comment explains the cause and suggests two fixes.
- [ ] The code compiles without warnings and runs; the `dotnet run` output matches the expectation.
- [ ] (Optional) Invariant checks via `Debug.Assert` or `if` with a clear message are added.
- [ ] A final comment or readme block briefly justifies the choice of `GroupBy` vs `ToLookup` at each point of the solution.

#### Hints (no direct answer)
- `elementSelector` is the second argument of `GroupBy` of the form `o => o.Total`; it returns what goes into the group, not the key. The key remains the first argument, `o => o.City`.
- To “step on the rake” of multiple enumeration, it is enough to iterate `IEnumerable<IGrouping>` first with `Any()` and then with `foreach`. To “step off” — wrap the result in `ToList()` right after `GroupBy`.
- `ILookup` is enumerable: `foreach (var g in lookup)` yields an `IGrouping<TKey, TElement>` with `g.Key` and `g` itself as `IEnumerable<TElement>`.
- `lookup[key]` for a missing key returns an empty `IEnumerable<TElement>`; `lookup.Contains(key)` is the boolean presence check.
- `StringComparer.OrdinalIgnoreCase` is a ready-made `IEqualityComparer<string>`; pass it as the second argument to `GroupBy(o => key, comparer)` or `ToLookup(o => key, comparer)`.
- For the trap demonstration, do not override either `Equals` or `GetHashCode` on `PlainOrder` — their absence is exactly what causes the collapse.
- A `record` gets value-based `Equals`/`GetHashCode` automatically; that is why `Order` from step 2 groups correctly and `PlainOrder` from step 8 does not. Compare their behaviour in the same output.
- To sort the groups, attach `OrderBy`/`OrderByDescending` **after** `Select` — that way you sort the already-projected report rows, not the `IGrouping` objects themselves.

#### Reference solution walk-through
```csharp
// Order.cs — record model with value semantics.
// A record auto-implements Equals/GetHashCode by value — grouping works correctly.
public sealed record Order(int OrderId, string CustomerId, string City, string Category, decimal Total);

// PlainOrder.cs — intentionally dumb class without equality.
// Default reference equality — every instance is unique, grouping falls apart.
public sealed class PlainOrder
{
    public int OrderId { get; init; }
    public string CustomerId { get; init; } = "";
    public string City { get; init; } = "";
    public string Category { get; init; } = "";
    public decimal Total { get; init; }
}

// Program.cs — top-level statements, C# 12 / .NET 8.
using System;
using System.Collections.Generic;
using System.Linq;

List<Order> orders =
[
    new(1, "C1", "Moscow",      "Books", 100m),
    new(2, "C2", "Kazan",       "Toys",  250m),
    new(3, "C1", "moscow",      "Books",  75m),  // different case of city
    new(4, "C3", "Kazan",       "Books", 300m),
    new(5, "C2", "Moscow",      "Toys",   50m),
    new(6, "C1", "Novosibirsk", "Books", 400m),
    new(7, "C3", "kazan",       "Toys",  120m),  // different case of city
];

// 1) GroupBy with elementSelector: the group stores only Total, not the whole Order.
var byCity = orders
    .GroupBy(o => o.City, o => o.Total)                      // key is city, element is total
    .Select(g => new { City = g.Key, Count = g.Count(), Sum = g.Sum() })
    .OrderByDescending(r => r.Sum);

Console.WriteLine("GroupBy by city (elementSelector = Total):");
foreach (var row in byCity)
    Console.WriteLine($"  {row.City}: orders={row.Count}, sum={row.Sum}");

// 2) Multiple enumeration: bad and good variants.
const bool demonstrateBug = true;
if (demonstrateBug)
{
    // BAD — Any() and foreach trigger grouping twice.
    var grouped = orders.GroupBy(o => o.CustomerId);        // not executed yet
    Console.WriteLine($"\n[BUG] Any() => {grouped.Any()} (first grouping)");
    foreach (var g in grouped)                               // second grouping
        Console.WriteLine($"  {g.Key}: {g.Count()} order(s)");
}

// GOOD — materialization via ToList pins the result.
var pinned = orders.GroupBy(o => o.CustomerId).ToList();    // executed once
Console.WriteLine("\n[OK] ToList materialized the groups:");
foreach (var g in pinned)
    Console.WriteLine($"  {g.Key}: {g.Count()} order(s)");

// 3) ToLookup — eager index "customer → orders".
ILookup<string, Order> byCustomer = orders.ToLookup(o => o.CustomerId);

Console.WriteLine("\nToLookup by CustomerId:");
foreach (var g in byCustomer)
{
    Console.WriteLine($"  {g.Key}: {g.Count()} order(s)");
    foreach (var o in g)
        Console.WriteLine($"    - #{o.OrderId}, {o.City}, {o.Total}");
}

// The indexer never throws — empty sequence for a missing key.
foreach (var key in new[] { "C1", "C2", "NO_SUCH_CUSTOMER" })
    Console.WriteLine($"  lookup[{key}] => {byCustomer[key].Count()} order(s)");

// 4) Case-insensitive grouping via a comparer.
var byCityCi = orders
    .GroupBy(o => o.City, StringComparer.OrdinalIgnoreCase) // comparer straight into GroupBy
    .Select(g => (City: g.Key, Count: g.Count(), Sum: g.Sum()))
    .OrderBy(t => t.City);

Console.WriteLine("\nGroupBy with OrdinalIgnoreCase (Moscow == moscow):");
foreach (var (city, count, sum) in byCityCi)
    Console.WriteLine($"  {city}: orders={count}, sum={sum}");

// 5) Trap: grouping by a class without overridden equality.
List<PlainOrder> plainOrders = orders
    .Select(o => new PlainOrder
    {
        OrderId = o.OrderId, CustomerId = o.CustomerId,
        City = o.City, Category = o.Category, Total = o.Total
    }).ToList();

var plainGroups = plainOrders.GroupBy(o => o.CustomerId);
Console.WriteLine($"\n[TRAP] PlainOrder: groups = {plainGroups.Count()}, objects = {plainOrders.Count}");
Console.WriteLine("  Reference equality → each record in its own group. Fix: record or IEqualityComparer.");

// 6) Invariants.
System.Diagnostics.Debug.Assert(byCustomer["NO_SUCH_CUSTOMER"].Count() == 0, "missing key must yield empty");
System.Diagnostics.Debug.Assert(byCityCi.Count() == 3, "case-insensitive cities must merge to 3");
System.Diagnostics.Debug.Assert(pinned.Count == 3, "unique customers: C1, C2, C3");
```

Walk-through. The `Order` model is declared as a `sealed record` with positional syntax — this is the lesson’s first key decision: a `record` auto-implements `Equals`/`GetHashCode` based on field values, so two records with the same `OrderId` (or, more importantly for grouping, the same key) are considered equal. Contrasted with it is `PlainOrder` — a plain `class` without overridden equality, which compares by reference; it is exactly what lets step 5 honestly show the trap “every record in its own group”. The `orders` list uses a collection expression (C# 12): square brackets instead of `new List<Order> { ... }` — a modern idiom recommended for .NET 8. The data deliberately mixes cities in different cases (`"moscow"`, `"kazan"`) so that step 4 (without a comparer) and step 7 (with a comparer) produce different results.

Step 1 is a classic `GroupBy → Select → OrderByDescending` pipeline. Two lesson concepts are applied at once. First, the `elementSelector` (`o => o.Total`): the group stores only totals, not whole `Order` objects. The lesson explicitly recommends this as a way to save memory and simplify downstream code — inside `Select` we call `g.Count()` and `g.Sum()` over an `IEnumerable<decimal>`, not over an `IEnumerable<Order>`. Second, deferredness: `GroupBy` does nothing until the `foreach` in the output. If we iterated `byCity` twice, grouping would execute twice — exactly what step 2 demonstrates.

Step 2 is the demonstration of multiple enumeration, one of the lesson’s main warnings. The bad variant `if (grouped.Any()) { foreach (var g in grouped) ... }` runs grouping twice: once for `Any()` to check non-emptiness, once for `foreach`. The good alternative, per the lesson, is `ToList()` right after `GroupBy`: it pins the result, and subsequent iterations are free. The `demonstrateBug` flag keeps the bad variant in the code as an instructional demo without spoiling the main output.

Step 3 is `ToLookup`. The lesson defines it as “`GroupBy` that executes eagerly”. Unlike `GroupBy`, `ToLookup` immediately builds an immutable `ILookup<string, Order>` that behaves like a multi-dictionary: one key maps to a collection of values. Iteration via `foreach` yields an `IGrouping<string, Order>` — the same “labeled shelf” as `GroupBy`. But the headline property is the indexer `byCustomer[key]`: it **never** throws `KeyNotFoundException`, returning an empty sequence for a missing key. This is verified in the loop over three keys including `"NO_SUCH_CUSTOMER"`. The choice of `ToLookup` here is no accident: the business needs repeated point access “orders of a customer”, and recomputing groups per query would be wasteful. `GroupBy` would be a mistake here.

Step 4 is case-insensitive grouping through `StringComparer.OrdinalIgnoreCase`. The lesson stresses that the comparer is passed directly into the operator, not “bolted onto” the key via `ToLower()`. The result: `"Moscow"` and `"moscow"` merge into a single group, and `g.Key` keeps the original case of the first encountered element. This is both a best practice (no scattered `ToLower()` across the code) and a correctness requirement (no risk of culture desync). Contrast with step 1, where without a comparer `Moscow` and `moscow` were distinct keys — a vivid difference.

Step 5 is the trap “grouping by a class without equality”. `PlainOrder` is an ordinary `class` with no overridden `Equals`/`GetHashCode`, so it defaults to reference equality. Two distinct instances with the same `CustomerId` are not equal, and `GroupBy` puts each in its own group. The output `groups = 7, objects = 7` (rather than `groups = 3`) is the “collapse”. The lesson suggests two fixes: make the type a `record` (like `Order`) or pass an `IEqualityComparer<TKey>`. The code comment records both paths. The invariants via `Debug.Assert` lock down the key properties: an empty result for a missing key, exactly three cities in the case-insensitive report, exactly three unique customers in `pinned`. Together all steps walk the student through the lesson’s central fork: `GroupBy` when building a pipeline; `ToLookup` when a ready index for frequent lookups is needed.

#### Going deeper (bonus)
1. Composite key. Build a “year + month → order total” report using an anonymous type `new { o.OrderDate.Year, o.OrderDate.Month }` as the `GroupBy` key. Add an `DateTime OrderDate` property to `Order` and populate the data. Explain why an anonymous type works correctly as a key (it has value-based `Equals`/`GetHashCode` out of the box).
2. Performance comparison. Measure three approaches to repeated “orders of a customer” queries on 100,000 orders with `BenchmarkDotNet`: (a) `orders.Where(o => o.CustomerId == id).ToList()` per query; (b) `orders.GroupBy(o => o.CustomerId)` without materialisation plus iteration; (c) `orders.ToLookup(o => o.CustomerId)` plus the indexer. Explain why (c) wins on a large number of queries.
3. Custom comparer. Implement an `IEqualityComparer<string>` that considers cities equal when they match after stripping a trailing slash and upper-casing (modelling route normalisation). Pass it to `GroupBy` and verify the merge of `"moscow"`/`"Moscow/"`.
4. `resultSelector`. Rewrite step 1 using the `GroupBy` overload with a `resultSelector` (three arguments: keySelector, elementSelector, resultSelector) to turn each group directly into a report row without a separate `Select`. Compare the readability of the two variants.

---

#### Чек-лист сдачи / Submission checklist
- [ ] Создан проект `OrderAnalytics` на .NET 8 / C# 12, собирается без ошибок и предупреждений.
- [ ] `Order` — `sealed record`, `PlainOrder` — `class` без переопределённого равенства.
- [ ] Список заказов инициализирован через collection expression, есть города в разном регистре.
- [ ] Построен отчёт «город → count/sum» через `GroupBy` с `elementSelector` и `Select`.
- [ ] В комментарии пояснена польза `elementSelector` (память, downstream-код).
- [ ] Показаны плохой (двойная группировка) и хороший (`ToList`) варианты обхода `GroupBy`.
- [ ] `ToLookup` построен по `CustomerId` и обойдён через `foreach`.
- [ ] Точечные запросы по `C1`, `C2`, `NO_SUCH_CUSTOMER` работают; последний возвращает пустую последовательность без исключения.
- [ ] Регистронезависимая группировка использует `StringComparer.OrdinalIgnoreCase` напрямую в `GroupBy`.
- [ ] Демонстрация ловушки показывает развал группировки по `PlainOrder` с комментарием и двумя вариантами исправления.
- [ ] Вывод `dotnet run` соответствует ожидаемому.
- [ ] Project `OrderAnalytics` on .NET 8 / C# 12 created; builds without errors or warnings.
- [ ] `Order` is a `sealed record`; `PlainOrder` is a `class` without overridden equality.
- [ ] Order list initialised via a collection expression; cities in different cases are present.
- [ ] “city → count/sum” report built with `GroupBy` + `elementSelector` + `Select`.
- [ ] A comment explains the benefit of `elementSelector` (memory, downstream code).
- [ ] Both the bad (double grouping) and good (`ToList`) iteration variants are shown.
- [ ] `ToLookup` is built by `CustomerId` and iterated via `foreach`.
- [ ] Point lookups for `C1`, `C2`, `NO_SUCH_CUSTOMER` work; the last returns an empty sequence without an exception.
- [ ] The case-insensitive grouping uses `StringComparer.OrdinalIgnoreCase` directly in `GroupBy`.
- [ ] The trap demonstration shows the collapsed grouping by `PlainOrder` with a comment and two fix options.
- [ ] The `dotnet run` output matches the expectation.

#### Ресурсы / Resources
- [Microsoft Learn — Enumerable.GroupBy](https://learn.microsoft.com/dotnet/api/system.linq.enumerable.groupby)
- [Microsoft Learn — Enumerable.ToLookup](https://learn.microsoft.com/dotnet/api/system.linq.enumerable.tolookup)
- [Microsoft Learn — ILookup<TKey,TElement>](https://learn.microsoft.com/dotnet/api/system.linq.ilookup-2)
- [Microsoft Learn — IGrouping<TKey,TElement>](https://learn.microsoft.com/dotnet/api/system.linq.igrouping-2)
- [Microsoft Learn — StringComparer.OrdinalIgnoreCase](https://learn.microsoft.com/dotnet/api/system.stringcomparer.ordinalignorecase)
