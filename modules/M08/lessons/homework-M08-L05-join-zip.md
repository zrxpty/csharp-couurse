---
[← К уроку M08-L05](lesson-M08-L05-join-zip.md) | [⬆ К модулю M08](../README.md) | [Следующее ДЗ →](homework-M08-L06-aggregates.md)
---

### Домашнее задание M08-L05: Join, GroupJoin, Zip / Homework M08-L05: Join, GroupJoin, Zip

**Урок / Lesson:** M08-L05
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 4/5
**Цель / Goal:** (RU) Научиться применять три оператора соединения LINQ — `Join`, `GroupJoin` и `Zip` — в реальных задачах, понимать разницу между внутренним соединением, соединением с группировкой и позиционным объединением, строить составные ключи через анонимные типы, превращать `GroupJoin` в плоский `LEFT JOIN` через `SelectMany` + `DefaultIfEmpty`, осознавать асимптотику `O(n+m)` и избегать типичных ошибок (порядок аргументов, ссылочные ключи без переопределённого равенства, соединение в цикле, молчаливая потеря данных в `Zip`). (EN) Learn to apply the three LINQ join operators — `Join`, `GroupJoin`, and `Zip` — in real tasks, understand the difference between inner join, grouped join, and positional pairing, build composite keys via anonymous types, flatten `GroupJoin` into a classic `LEFT JOIN` through `SelectMany` + `DefaultIfEmpty`, internalize the `O(n+m)` complexity, and avoid common mistakes (argument order, reference-type keys without overridden equality, joining inside loops, silent data loss in `Zip`).

#### Связь с уроком / Connection to the lesson
(RU) Урок M08-L05 вводит три оператора соединения с принципиально разной семантикой: `Join` (внутреннее соединение, аналог SQL `INNER JOIN`), `GroupJoin` (сохраняет все внешние элементы, формирует иерархию «мастер–детали») и `Zip` (позиционное объединение без ключей). В этом ДЗ вы закрепите каждый из них на едином наборе моделей «магазины – товары – продажи – рейтинги», построите составные ключи и плоский `LEFT JOIN`, и научитесь отличать ситуации, где нужен каждый оператор. Все частые ошибки из урока намеренно заложены в требования как точки проверки.
(EN) Lesson M08-L05 introduces three join operators with fundamentally different semantics: `Join` (inner join, the analog of SQL `INNER JOIN`), `GroupJoin` (preserves every outer element, forming a master–detail hierarchy), and `Zip` (positional pairing without keys). In this homework you will practice each of them on a single dataset of “stores – products – sales – ratings”, build composite keys and a flat `LEFT JOIN`, and learn to tell apart the situations where each operator fits. Every common mistake from the lesson is intentionally embedded in the requirements as a checkpoint.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация
Представьте, что вы инженер аналитики в небольшой розничной сети «Четыре сезона». У вас есть четыре источника данных, которые сейчас хранятся независимо друг от друга: справочник магазинов (`Store`), каталог товаров (`Product`), журнал продаж (`Sale`) и ежемесячные рейтинги товарных категорий (`CategoryRating`). Ни один источник в одиночку не даёт полной картины: чтобы посчитать выручку по магазинам, нужно связать продажи с товарами и магазинами; чтобы понять, какие товары «просели» относительно рейтинга категории, нужно соединить товары с рейтингами; чтобы построить параллельный отчёт «факт против плана» по позициям списка, нужно объединить два массива одинаковой длины по индексу. SQL дал бы вам `JOIN`, `LEFT JOIN` и… ничего для позиционного объединения. В LINQ же у вас есть ровно три оператора — `Join`, `GroupJoin` и `Zip`, — и умение выбрать правильный отличает中级-разработчика от начинающего. Это не академическое упражнение: те же модели и те же ошибки встречаются в реальных ETL-конвейерах, в отчётах EF Core и в обработке событий. Задача этого ДЗ — пройти путь от «я знаю, что `Join` существует» до «я знаю, когда какой оператор применить, почему он даёт `O(n+m)`, и какие подводные камни меня ждут».

#### Что нужно сделать (пошагово)
1. Создайте консольный проект .NET 8 командой `dotnet new console -n JoinZipHomework -o JoinZipHomework` и перейдите в папку `cd JoinZipHomework`. Убедитесь, что в `JoinZipHomework.csproj` стоит `<TargetFramework>net8.0</TargetFramework>` и язык C# 12 (для top-level statements и collection expressions).
2. В файле `Program.cs` опишите четыре модели данных через `record`: `Store(int Id, string City, string Region)`, `Product(int Id, string Name, string Category, decimal Price)`, `Sale(int Id, int StoreId, int ProductId, int Quantity, DateTime On)` и `CategoryRating(string Category, int Rating, int Month)`. Используйте `record`, чтобы равенство по значению работало «из коробки» — это важно, если вы решите использовать модели как ключи.
3. Заполните списки данными. Магазинов — 4 (Москва, Санкт-Петербург, Казань, Новосибирск), товаров — 6 (две категории по три товара), продаж — около 10 (включая магазин без продаж и товар без продаж), рейтингов — 2 строки (по одной категории). Намеренно добавьте «граничный» магазин без продаж, чтобы проверить поведение `GroupJoin` и плоского `LEFT JOIN`.
4. Реализуйте отчёт **A «Выручка по магазинам»** через `Join`: соедините `sales` с `stores` и с `products`, посчитайте `Quantity * Price` для каждой продажи, сгруппируйте и просуммируйте по магазину. Вывод должен содержать строки вида `Москва: 123 450 ₽`.
5. Реализуйте отчёт **B «Все магазины, даже без продаж»** через `GroupJoin` + `SelectMany` + `DefaultIfEmpty` (плоский `LEFT JOIN`). Магазин без продаж должен появиться со строкой `Новосибирск: 0 ₽ (нет продаж)`. Это ключевая проверка: классический `Join` здесь дал бы неверный результат, «потеряв» магазин.
6. Реализуйте отчёт **C «Иерархия мастер–детали»** через `GroupJoin` без выравнивания: для каждого магазина выведите список его продаж (город, количество продаж, суммарная выручка, перечень `ProductId`). Используйте `record` или анонимный тип как проекцию.
7. Реализуйте отчёт **D «Товары и рейтинги категорий»** через `Join` с **составным ключом** `new { Product.Category, Rating.Month }` — соедините товары с рейтингами так, чтобы каждый товар получил рейтинг своей категории за нужный месяц. Выведите `Товар — Категория — Рейтинг`.
8. Реализуйте отчёт **E «План против факта»** через `Zip`: у вас есть два массива одинаковой длины — `plannedNames` (плановые названия метрик) и `actualValues` (фактические числа). Соедините их по индексу и выведите пары. Намеренно сделайте длины разными хотя бы в одном пробном запуске, чтобы убедиться, что лишние элементы молча отбрасываются, и добавьте `Debug.Assert` или явную проверку `plannedNames.Length == actualValues.Length` с понятным сообщением об ошибке.
9. Добавьте микро-бенчмарк: для отчёта A замерьте время через `Stopwatch` на двух реализациях — наивный вложенный цикл `foreach + Where` против `Join`. Выведите оба времени. На малых данных разница незаметна, но это закрепит понимание асимптотики.
10. Проверьте программу командой `dotnet run` и убедитесь, что все пять отчётов выводятся корректно и магазин без продаж действительно появляется только в отчётах B и C, но не в A.

#### Требования к решению
- Целевой фреймворк — `net8.0`, язык — C# 12. Разрешены и поощряются top-level statements, collection expressions (`new()` / `[]`), pattern matching, `record`-модели, raw string literals для многострочных подсказок.
- Все три оператора — `Join`, `GroupJoin`, `Zip` — должны быть использованы минимум по одному разу, каждый в отчёте, где он семантически уместен (нельзя «натянуть» `Zip` на задачу с ключами).
- Плоский `LEFT JOIN` обязан использовать связку `GroupJoin` → `SelectMany` → `DefaultIfEmpty`, а не ручную проверку `if (orders.Any(...))`. Это закрепляет идиому из урока.
- Составной ключ обязан быть построен через анонимный тип `new { ... }` — не через строковую конкатенацию и не через кортеж (хотя кортеж тоже структурно равен, анонимный тип — канонический способ из урока).
- Запрещено использовать `Equals`/`GetHashCode` вручную: если ключ — это модель, используйте `record`; если составной — анонимный тип. Это проверяет знание частой ошибки «ссылочный ключ без переопределённого равенства → соединение пустое».
- Запрещено соединять внутри цикла: если вы заметили, что `Join` вызывается в `foreach`, вынесите соединение наружу или постройте `Lookup` заранее. Это закрепляет правило «каждый `Join` строит новый словарь → `O(n²)`».
- Код должен компилироваться без предупреждений уровня `error` и работать детерминированно (порядок строк в отчётах можно зафиксировать через `OrderBy`).
- Имена файлов: `Program.cs` (вся логика) или разбивка на `Models.cs` + `Reports.cs` + `Program.cs` на ваше усмотрение.

#### Тонкости и подводные камни
- **Порядок аргументов `Join`.** Сигнатура строго `(outer, inner, outerKey, innerKey, result)`. Начинающие часто путают `outerKey` и `innerKey`, особенно когда оба селектора выглядят похоже (`c => c.Id` и `o => o.CustomerId`). Результат — пустой вывод без ошибки. Всегда проговаривайте вслух: «внешний ключ — от outer, внутренний ключ — от inner».
- **`Join` ≠ `LEFT JOIN`.** Если вы ожидаете, что магазин без продаж появится в отчёте A, вы ошиблись оператором. `Join` молча отбрасывает элементы без совпадений. Для сохранения нужен `GroupJoin` или плоский `LEFT JOIN`.
- **Ссылочные ключи без переопределённого равенства.** Если вы используете класс с ссылочным равенством в качестве ключа, два «одинаковых» объекта не совпадут, и соединение вернёт пустоту. Решение — `record` (равенство по значению) или анонимный тип (структурное равенство). Никогда не используйте `class` без переопределённых `Equals`/`GetHashCode` как ключ.
- **Соединение в цикле.** Каждый вызов `Join` строит новый словарь из внутренней последовательности. Если вы вызываете `Join` внутри `foreach` по тысяче итераций, вы получаете `O(n²)` по памяти и времени. Выносите соединение за цикл или стройте `ToLookup` один раз.
- **`Zip` и разная длина.** `Zip` молча обрезает по более короткой последовательности. Если длины расходятся, вы теряете данные без исключения. В .NET 9+ появился трёхаргументный `Zip` с заполнителем, но в .NET 8 — только явная проверка `Length`/`Count`.
- **Тяжёлые вычисления в проекторе.** Если в лямбде `(c, o) => ...` вы делаете дорогой расчёт, он выполнится для каждой пары. Выносите проекцию с тяжёлой логикой в отдельный `Select` после соединения — так вы разделяете «соединить» и «вычислить».
- **`DefaultIfEmpty` без типа.** При плоском `LEFT JOIN` внутренний элемент может быть `null` (для внешнего элемента без совпадений). Используйте `o?.OrderId` и `o?.Amount`, а не `o.OrderId`, иначе получите `NullReferenceException`.
- **EF Core vs LINQ to Objects.** Если вы захотите перенести решение на EF Core, соединения транслируются в SQL JOIN, и правила оптимизации лежат на стороне СУБД. Старайтесь соединять на стороне базы, а в памяти только проектировать. Это ДЗ работает с LINQ to Objects, но помните про разницу.

#### Критерии приёмки
- [ ] Проект создаётся командой `dotnet new console`, фреймворк `net8.0`, C# 12.
- [ ] Описаны четыре `record`-модели: `Store`, `Product`, `Sale`, `CategoryRating`.
- [ ] В данных есть магазин без продаж и товар без продаж (граничные случаи).
- [ ] Отчёт A построен через `Join`, корректно считает выручку `Quantity * Price`.
- [ ] Отчёт B построен через `GroupJoin` + `SelectMany` + `DefaultIfEmpty` (плоский `LEFT JOIN`).
- [ ] Магазин без продаж появляется в отчётах B и C, но НЕ в отчёте A.
- [ ] Отчёт C — иерархия мастер–детали через `GroupJoin` без выравнивания.
- [ ] Отчёт D использует составной ключ через анонимный тип `new { ... }`.
- [ ] Отчёт E использует `Zip` по индексу, с явной проверкой длин.
- [ ] В коде нет соединений внутри `foreach` (проверка на `O(n²)`).
- [ ] Нет ручных `Equals`/`GetHashCode`; равенство ключей обеспечено `record` или анонимным типом.
- [ ] Микро-бенчмарк сравнивает вложенный цикл и `Join` через `Stopwatch`.
- [ ] `dotnet run` выводит все пять отчётов без исключений.
- [ ] Код компилируется без ошибок и без `warning` уровня `error`.
- [ ] В комментариях (RU+EN) объяснён выбор оператора в каждом отчёте.

#### Подсказки (без прямого ответа)
- Для отчёта A можно «дважды соединить»: сначала `sales.Join(products, ...)`, потом `.Join(stores, ...)`. Подумайте, какой селектор ключа использовать на каждом шаге.
- Для плоского `LEFT JOIN` шаблон: `outer.GroupJoin(inner, ok, ik, (o, g) => new { o, g }).SelectMany(x => x.g.DefaultIfEmpty(), (x, i) => ... )`. Обратите внимание, что второй аргумент `SelectMany` получает `i`, который может быть `null`.
- Для составного ключа помните, что анонимные типы сравниваются структурно: `new { a = 1, b = "x" }` равен другому `new { a = 1, b = "x" }`, но не равен `new { b = "x", a = 1 }` (порядок свойств важен).
- Для `Zip` проверяйте `plannedNames.Length == actualValues.Length` до вызова и выбрасывайте `InvalidOperationException` с понятным сообщением, если длины расходятся.
- Для микро-бенчмарка нагенерируйте 10 000 продаж и 1 000 товаров, чтобы разница между `O(n·m)` и `O(n+m)` стала заметна.

#### Эталонное решение (разбор)
```csharp
// C# 12 / .NET 8 — Join, GroupJoin, Zip: домашнее задание M08-L05
// Homework M08-L05: three join operators on a single dataset.

using System.Diagnostics;

// Модели / Models — record даёт равенство по значению / records give value equality
public record Store(int Id, string City, string Region);
public record Product(int Id, string Name, string Category, decimal Price);
public record Sale(int Id, int StoreId, int ProductId, int Quantity, DateTime On);
public record CategoryRating(string Category, int Rating, int Month);

// Данные / Data — collection expressions C# 12 / C# 12 collection expressions
List<Store> stores = [
    new(1, "Москва", "ЦФО"),
    new(2, "Санкт-Петербург", "СЗФО"),
    new(3, "Казань", "ПФО"),
    new(4, "Новосибирск", "СФО"), // магазин без продаж / store with no sales
];

List<Product> products = [
    new(10, "Ноутбук",   "Электроника", 80_000m),
    new(11, "Планшет",   "Электроника", 35_000m),
    new(12, "Наушники",  "Электроника",  5_000m),
    new(20, "Кофе",      "Бакалея",       1_200m),
    new(21, "Чай",       "Бакалея",         900m),
    new(22, "Сахар",     "Бакалея",         120m),
];

List<Sale> sales = [
    new(1, 1, 10, 2, new DateTime(2024, 5, 10)),
    new(2, 1, 12, 5, new DateTime(2024, 5, 11)),
    new(3, 1, 20, 10, new DateTime(2024, 5, 12)),
    new(4, 2, 11, 1, new DateTime(2024, 5, 12)),
    new(5, 2, 21, 8, new DateTime(2024, 5, 13)),
    new(6, 3, 10, 1, new DateTime(2024, 5, 14)),
    new(7, 3, 22, 30, new DateTime(2024, 5, 14)),
    // Новосибирск (Id=4) намеренно без продаж / intentionally no sales
];

List<CategoryRating> ratings = [
    new("Электроника", 9, 5),
    new("Бакалея",     7, 5),
];

// ── Отчёт A: Выручка по магазинам через Join / Report A: revenue via Join ──
// Внутреннее соединение: магазин без продаж НЕ попадёт / inner join: store with no sales drops out
var revenueByStore = sales
    .Join(products, s => s.ProductId, p => p.Id, (s, p) => new { s.StoreId, LineTotal = s.Quantity * p.Price })
    .Join(stores,   r => r.StoreId,  st => st.Id, (r, st) => new { st.City, r.LineTotal })
    .GroupBy(x => x.City)
    .Select(g => new { City = g.Key, Total = g.Sum(x => x.LineTotal) })
    .OrderByDescending(x => x.Total);

Console.WriteLine("A. Выручка по магазинам (Join, INNER):");
foreach (var r in revenueByStore)
    Console.WriteLine($"  {r.City}: {r.Total:N0} ₽");
// Новосибирска здесь нет — Inner join / Novosibirsk is absent here

// ── Отчёт B: Все магазины, даже без продаж / Report B: all stores incl. no sales ──
// GroupJoin + SelectMany + DefaultIfEmpty = плоский LEFT JOIN / flat LEFT JOIN
var leftJoin = stores
    .GroupJoin(
        sales,
        st => st.Id,
        s => s.StoreId,
        (st, salesGroup) => new { st, salesGroup })
    .SelectMany(
        x => x.salesGroup.DefaultIfEmpty(),
        (x, s) => new
        {
            x.st.City,
            LineTotal = s is null ? 0m : s.Quantity * products.First(p => p.Id == s.ProductId).Price,
            HasSales = s is not null,
        })
    .GroupBy(x => x.City)
    .Select(g => new
    {
        City = g.Key,
        Total = g.Sum(x => x.LineTotal),
        AnySales = g.Any(x => x.HasSales),
    })
    .OrderByDescending(x => x.Total);

Console.WriteLine("\nB. Все магазины (GroupJoin + SelectMany + DefaultIfEmpty, LEFT JOIN):");
foreach (var r in leftJoin)
    Console.WriteLine(r.AnySales
        ? $"  {r.City}: {r.Total:N0} ₽"
        : $"  {r.City}: 0 ₽ (нет продаж)");

// ── Отчёт C: Иерархия мастер–детали / Report C: master–detail hierarchy ──
var hierarchy = stores
    .GroupJoin(
        sales,
        st => st.Id,
        s => s.StoreId,
        (st, salesGroup) => new
        {
            Store = st,
            Sales = salesGroup.ToList(),
            Total = salesGroup.Sum(s => s.Quantity * products.First(p => p.Id == s.ProductId).Price)),
        });

Console.WriteLine("\nC. Иерархия мастер–детали (GroupJoin):");
foreach (var h in hierarchy)
{
    Console.WriteLine($"  {h.Store.City} (регион {h.Store.Region}): " +
                      $"продаж={h.Sales.Count}, сумма={h.Total:N0} ₽");
    foreach (var s in h.Sales)
        Console.WriteLine($"    – Sale#{s.Id}: товар={s.ProductId}, кол-во={s.Quantity}");
}

// ── Отчёт D: Товары и рейтинги категорий, составной ключ / Report D: composite key ──
// Составной ключ через анонимный тип — структурное равенство / composite key via anonymous type
var productsWithRating = products
    .Join(
        ratings,
        p => new { p.Category, Month = 5 },          // внешний составной ключ / outer composite key
        r => new { r.Category, r.Month },            // внутренний составной ключ / inner composite key
        (p, r) => new { p.Name, p.Category, r.Rating })
    .OrderBy(x => x.Category)
    .ThenBy(x => x.Name);

Console.WriteLine("\nD. Товары и рейтинги (Join, составной ключ):");
foreach (var r in productsWithRating)
    Console.WriteLine($"  {r.Name} [{r.Category}] — рейтинг {r.Rating}");

// ── Отчёт E: План против факта через Zip / Report E: plan vs actual via Zip ──
string[] plannedNames = ["Выручка", "Средний чек", "Конверсия"];
decimal[] actualValues = [1_250_000m, 4_300m, 3.2m];
// намеренная проверка длин — Zip молча обрежет иначе / explicit length check — Zip truncates silently
if (plannedNames.Length != actualValues.Length)
    throw new InvalidOperationException(
        $"Длины не совпадают: {plannedNames.Length} vs {actualValues.Length}");
var planVsActual = plannedNames.Zip(actualValues, (name, value) => new { name, value });

Console.WriteLine("\nE. План против факта (Zip по индексу):");
foreach (var r in planVsActual)
    Console.WriteLine($"  {r.name}: {r.value}");

// ── Микро-бенчмарк: наивный цикл vs Join / Micro-benchmark: naive loop vs Join ──
var rnd = new Random(42);
var bigSales  = Enumerable.Range(0, 20_000)
    .Select(i => new Sale(i, rnd.Next(1, 5), rnd.Next(10, 23), rnd.Next(1, 10), DateTime.Today)).ToList();
var bigProducts = Enumerable.Range(10, 13).Select(i => new Product(i, $"P{i}", "X", 100m)).ToList();

var sw = Stopwatch.StartNew();
// Наивный вложенный цикл — O(n·m) / naive nested loop — O(n·m)
decimal naive = 0;
foreach (var s in bigSales)
    foreach (var p in bigProducts)
        if (s.ProductId == p.Id)
            naive += s.Quantity * p.Price;
sw.Stop();
var naiveMs = sw.ElapsedMilliseconds;

sw.Restart();
// Join — O(n+m), строит словарь один раз / Join — O(n+m), builds dictionary once
decimal joined = bigSales
    .Join(bigProducts, s => s.ProductId, p => p.Id, (s, p) => s.Quantity * p.Price)
    .Sum();
sw.Stop();
var joinMs = sw.ElapsedMilliseconds;

Console.WriteLine($"\nБенчмарк: naive={naiveMs} мс ({naive:N0}), Join={joinMs} мс ({joined:N0})");
Console.WriteLine(naive == joined ? "✅ Суммы совпадают." : "❌ Суммы РАЗЛИЧАЮТСЯ — баг в логике.");
```

**Разбор по строкам.** Модели описаны как `record`, потому что ключевая частая ошибка из урока — ссылочные ключи без переопределённого равенства; `record` даёт структурное равенство автоматически, и если вы решите использовать саму модель как ключ (например, `Product` как ключ в каком-то соединении), она будет сравниваться по значениям свойств, а не по ссылке. Данные собраны через collection expressions C# 12 (`[ ... ]`), что компактнее `new List<T> { ... }`. Намеренно добавлен магазин Новосибирск (Id=4) без продаж — это «граничный» случай, на котором проверяется разница между `Join` и `GroupJoin`: в отчёте A его нет (inner join отбрасывает), в отчётах B и C он появляется (left join сохраняет). Отчёт A дважды применяет `Join`: сначала соединяет продажи с товарами по `ProductId`, чтобы получить `LineTotal = Quantity * Price`, затем соединяет результат с магазинами по `StoreId`, чтобы получить название города. Порядок аргументов строго `(outer, inner, outerKey, innerKey, result)` — это та самая сигнатура, которую новички путают; здесь она применена корректно. Финальная `GroupBy` по городу и `Sum` дают выручку. Отчёт B реализует плоский `LEFT JOIN` через каноническую идиому `GroupJoin` → `SelectMany` → `DefaultIfEmpty`: `GroupJoin` сохраняет все магазины и формирует группу продаж для каждого (возможно, пустую), `SelectMany` разворачивает группы в строки, а `DefaultIfEmpty` гарантирует, что для магазина без продаж появится одна строка с `null`-продажей. В проекторе мы используем `s is null` и `s is not null` (pattern matching C# 12), а не `s == null`, и `x => x.LineTotal` через тернарный оператор — это защищает от `NullReferenceException`, который был бы неизбежен при наивном `s.Quantity`. Отчёт C оставляет `GroupJoin` «как есть», без выравнивания — это иерархическая структура «магазин и его список продаж», которая читается естественнее для мастер–детали отчётов. Отчёт D использует составной ключ через анонимный тип `new { p.Category, Month = 5 }` и `new { r.Category, r.Month }`: это работает, потому что анонимные типы в C# реализуют структурное равенство — два анонимных объекта равны, если совпадают все свойства одного типа в одном порядке. Порядок свойств важен: `new { a, b }` ≠ `new { b, a }`. Отчёт E использует `Zip` для позиционного объединения двух массивов одинаковой длины; перед вызовом явно проверяется `plannedNames.Length == actualValues.Length`, потому что иначе `Zip` молча обрежет по более короткому — это отдельная частая ошибка из урока. Микро-бенчмарк сравнивает наивный вложенный `foreach` (`O(n·m)`) и `Join` (`O(n+m)`) на 20 000 продажах и 13 товарах; на таких объёмах разница становится заметной, и это закрепляет интуицию, почему `Join` предпочтительнее вложенных циклов с `Where`. Финальная проверка `naive == joined` — это не «доверяй, а проверяй»: две реализации должны дать одинаковую сумму, иначе где-то ошибка в логике.

#### Задания на углубление (бонус)
1. **Перенесите отчёты A–D на EF Core.** Создайте `DbContext` с `DbSet<Store>`, `DbSet<Product>`, `DbSet<Sale>`, `DbSet<CategoryRating>`, напишите те же запросы через LINQ поверх `DbSet`, и проверьте через логирование SQL, что соединения транслируются в `INNER JOIN`/`LEFT JOIN`. Сравните с поведением LINQ to Objects.
2. **Реализуйте «анти-соединение» (anti-join).** Выведите товары, у которых нет ни одной продажи, используя `GroupJoin` + `Where(g => !g.Any())` или `Where` + `Contains`. Объясните, почему `Join` здесь не подходит.
3. **`Zip` трёх последовательностей.** В .NET 9 появился трёхаргументный `Zip`. Реализуйте эквивалент «руками» на .NET 8: объедините три массива `names`, `values`, `units` по индексу в одну проекцию, с проверкой длин.
4. **Профилирование памяти.** Используйте `dotnet-counters` или `dotMemory` на микро-бенчмарке: покажите, что `Join` создаёт один словарь, а наивный цикл — не создаёт, но тратит больше CPU. Объясните компромисс «память vs время».

---

## Statement in English / Постановка на английском

#### Context & motivation
Imagine you are an analytics engineer at a small retail chain called “Four Seasons”. You have four data sources that currently live independently of each other: a store directory (`Store`), a product catalog (`Product`), a sales journal (`Sale`), and monthly category ratings (`CategoryRating`). No single source gives you the full picture on its own: to compute revenue per store you must link sales to products and stores; to understand which products “underperformed” relative to their category rating you must join products to ratings; to build a parallel “plan vs actual” report across a positional list you must zip two equal-length arrays by index. SQL would give you `JOIN`, `LEFT JOIN`, and… nothing for positional pairing. In LINQ you have exactly three operators — `Join`, `GroupJoin`, and `Zip` — and knowing which one to pick is what separates a mid-level developer from a junior. This is not an academic drill: the same models and the same mistakes show up in real ETL pipelines, in EF Core reporting code, and in event processing. The goal of this homework is to take you from “I know that `Join` exists” to “I know which operator to apply when, why it gives `O(n+m)`, and which pitfalls are waiting for me”.

#### What to do step by step
1. Create a .NET 8 console project with `dotnet new console -n JoinZipHomework -o JoinZipHomework` and enter the folder with `cd JoinZipHomework`. Verify that `JoinZipHomework.csproj` has `<TargetFramework>net8.0</TargetFramework>` and C# 12 enabled (for top-level statements and collection expressions).
2. In `Program.cs`, declare four data models as `record`: `Store(int Id, string City, string Region)`, `Product(int Id, string Name, string Category, decimal Price)`, `Sale(int Id, int StoreId, int ProductId, int Quantity, DateTime On)`, and `CategoryRating(string Category, int Rating, int Month)`. Using `record` is deliberate: value equality works out of the box, which matters if you later use a model as a join key.
3. Populate the lists. Four stores (Moscow, Saint Petersburg, Kazan, Novosibirsk), six products (two categories of three products each), about ten sales (including a store with no sales and a product with no sales), and two rating rows (one per category). Intentionally add a “boundary” store with no sales, so that the behavior of `GroupJoin` and the flat `LEFT JOIN` can be verified.
4. Implement report **A “Revenue by store”** using `Join`: join `sales` with `stores` and with `products`, compute `Quantity * Price` for each sale, group and sum by store. The output should contain lines like `Moscow: 123,450 ₽`.
5. Implement report **B “All stores, even without sales”** using `GroupJoin` + `SelectMany` + `DefaultIfEmpty` (a flat `LEFT JOIN`). The store with no sales must appear as `Novosibirsk: 0 ₽ (no sales)`. This is the key check: a plain `Join` would give the wrong answer here, silently dropping the store.
6. Implement report **C “Master–detail hierarchy”** using `GroupJoin` without flattening: for each store, print the list of its sales (city, sale count, total revenue, list of `ProductId`). Use a `record` or an anonymous type as the projection.
7. Implement report **D “Products and category ratings”** using `Join` with a **composite key** `new { Product.Category, Rating.Month }` — join products to ratings so that each product gets the rating of its own category for the right month. Print `Product — Category — Rating`.
8. Implement report **E “Plan vs actual”** using `Zip`: you have two arrays of equal length — `plannedNames` (planned metric names) and `actualValues` (actual numbers). Zip them by index and print the pairs. Intentionally make the lengths differ in at least one trial run to confirm that extra items are silently dropped, then add a `Debug.Assert` or an explicit `plannedNames.Length == actualValues.Length` check with a clear error message.
9. Add a micro-benchmark: for report A, measure time with `Stopwatch` on two implementations — a naive nested `foreach + Where` loop versus `Join`. Print both timings. On tiny data the difference is invisible, but the exercise cements the asymptotic intuition.
10. Run the program with `dotnet run` and confirm that all five reports print correctly and that the store with no sales appears only in reports B and C, never in A.

#### Requirements
- Target framework `net8.0`, language C# 12. Top-level statements, collection expressions (`new()` / `[]`), pattern matching, `record` models, and raw string literals for multi-line hints are all permitted and encouraged.
- All three operators — `Join`, `GroupJoin`, `Zip` — must be used at least once each, in a report where they are semantically appropriate (do not force `Zip` onto a keyed problem).
- The flat `LEFT JOIN` must use the `GroupJoin` → `SelectMany` → `DefaultIfEmpty` chain, not a manual `if (orders.Any(...))` check. This cements the idiom from the lesson.
- The composite key must be built with an anonymous type `new { ... }` — not with string concatenation and not with a tuple (a tuple is also structurally equal, but the anonymous type is the canonical form taught in the lesson).
- Manual `Equals`/`GetHashCode` overrides are forbidden: if the key is a model, use `record`; if it is composite, use an anonymous type. This checks the common mistake “reference-type key without overridden equality → empty join”.
- Joining inside a loop is forbidden: if you notice `Join` called inside a `foreach`, hoist the join out or precompute a `Lookup`. This cements the rule “every `Join` builds a new dictionary → `O(n²)`”.
- The code must compile without `error`-level warnings and run deterministically (you may fix the row order in reports with `OrderBy`).
- File names: `Program.cs` for all logic, or split into `Models.cs` + `Reports.cs` + `Program.cs` at your discretion.

#### Pitfalls
- **Argument order of `Join`.** The signature is strictly `(outer, inner, outerKey, innerKey, result)`. Beginners often swap `outerKey` and `innerKey`, especially when both selectors look similar (`c => c.Id` and `o => o.CustomerId`). The result is an empty output with no error. Always say it out loud: “outer key comes from outer, inner key comes from inner”.
- **`Join` is not `LEFT JOIN`.** If you expect the store with no sales to appear in report A, you picked the wrong operator. `Join` silently drops unmatched elements. To preserve them, use `GroupJoin` or the flat `LEFT JOIN`.
- **Reference-type keys without overridden equality.** If you use a class with reference equality as a key, two “equal” objects will not match, and the join returns nothing. The fix is `record` (value equality) or an anonymous type (structural equality). Never use a `class` without overridden `Equals`/`GetHashCode` as a key.
- **Joining inside a loop.** Each `Join` call builds a fresh dictionary from the inner sequence. If you call `Join` inside a `foreach` over a thousand iterations, you get `O(n²)` in time and memory. Hoist the join out of the loop or build a `ToLookup` once.
- **`Zip` and unequal lengths.** `Zip` silently truncates to the shorter sequence. If the lengths differ, you lose data with no exception. In .NET 9+ there is a three-argument `Zip` with a filler, but in .NET 8 you must check `Length`/`Count` explicitly.
- **Expensive computation in the projector.** If the lambda `(c, o) => ...` does heavy work, it runs for every matched pair. Move expensive projection into a separate `Select` after the join, so you separate “join” from “compute”.
- **`DefaultIfEmpty` without a type.** In a flat `LEFT JOIN`, the inner element can be `null` (for an outer element with no match). Use `o?.OrderId` and `o?.Amount`, not `o.OrderId`, or you will get a `NullReferenceException`.
- **EF Core vs LINQ to Objects.** If you later port this solution to EF Core, joins are translated to SQL JOINs, and optimization rules live on the database side. Prefer joining on the database and only projecting in memory. This homework uses LINQ to Objects, but keep the difference in mind.

#### Acceptance criteria
- [ ] The project is created with `dotnet new console`, framework `net8.0`, C# 12.
- [ ] Four `record` models are declared: `Store`, `Product`, `Sale`, `CategoryRating`.
- [ ] The data contains a store with no sales and a product with no sales (boundary cases).
- [ ] Report A is built with `Join` and correctly computes revenue `Quantity * Price`.
- [ ] Report B is built with `GroupJoin` + `SelectMany` + `DefaultIfEmpty` (flat `LEFT JOIN`).
- [ ] The store with no sales appears in reports B and C, but NOT in report A.
- [ ] Report C is a master–detail hierarchy via `GroupJoin` without flattening.
- [ ] Report D uses a composite key via an anonymous type `new { ... }`.
- [ ] Report E uses `Zip` by index, with an explicit length check.
- [ ] There are no joins inside `foreach` (the `O(n²)` check).
- [ ] There are no manual `Equals`/`GetHashCode`; key equality comes from `record` or anonymous types.
- [ ] The micro-benchmark compares the nested loop and `Join` using `Stopwatch`.
- [ ] `dotnet run` prints all five reports without exceptions.
- [ ] The code compiles with no errors and no `warning` at the `error` level.
- [ ] Comments (RU+EN) explain the choice of operator in each report.

#### Hints (no direct answer)
- For report A you can “join twice”: first `sales.Join(products, ...)`, then `.Join(stores, ...)`. Think about which key selector to use at each step.
- For the flat `LEFT JOIN` the pattern is: `outer.GroupJoin(inner, ok, ik, (o, g) => new { o, g }).SelectMany(x => x.g.DefaultIfEmpty(), (x, i) => ... )`. Note that the second argument of `SelectMany` receives `i`, which may be `null`.
- For the composite key remember that anonymous types are compared structurally: `new { a = 1, b = "x" }` equals another `new { a = 1, b = "x" }`, but does not equal `new { b = "x", a = 1 }` (property order matters).
- For `Zip`, check `plannedNames.Length == actualValues.Length` before the call and throw `InvalidOperationException` with a clear message if the lengths differ.
- For the micro-benchmark, generate 10,000 sales and 1,000 products so that the gap between `O(n·m)` and `O(n+m)` becomes visible.

#### Reference solution walk-through
```csharp
// C# 12 / .NET 8 — Join, GroupJoin, Zip: homework M08-L05
// Homework M08-L05: three join operators on a single dataset.

using System.Diagnostics;

// Models — record gives value equality out of the box
public record Store(int Id, string City, string Region);
public record Product(int Id, string Name, string Category, decimal Price);
public record Sale(int Id, int StoreId, int ProductId, int Quantity, DateTime On);
public record CategoryRating(string Category, int Rating, int Month);

// Data — C# 12 collection expressions
List<Store> stores = [
    new(1, "Moscow",          "Central"),
    new(2, "Saint Petersburg","North-West"),
    new(3, "Kazan",           "Volga"),
    new(4, "Novosibirsk",     "Siberia"), // store with no sales
];

List<Product> products = [
    new(10, "Laptop",    "Electronics", 80_000m),
    new(11, "Tablet",    "Electronics", 35_000m),
    new(12, "Headphones","Electronics",  5_000m),
    new(20, "Coffee",    "Grocery",      1_200m),
    new(21, "Tea",       "Grocery",        900m),
    new(22, "Sugar",     "Grocery",        120m),
];

List<Sale> sales = [
    new(1, 1, 10, 2, new DateTime(2024, 5, 10)),
    new(2, 1, 12, 5, new DateTime(2024, 5, 11)),
    new(3, 1, 20, 10, new DateTime(2024, 5, 12)),
    new(4, 2, 11, 1, new DateTime(2024, 5, 12)),
    new(5, 2, 21, 8, new DateTime(2024, 5, 13)),
    new(6, 3, 10, 1, new DateTime(2024, 5, 14)),
    new(7, 3, 22, 30, new DateTime(2024, 5, 14)),
    // Novosibirsk (Id=4) intentionally has no sales
];

List<CategoryRating> ratings = [
    new("Electronics", 9, 5),
    new("Grocery",     7, 5),
];

// ── Report A: revenue by store via Join (INNER) ──
var revenueByStore = sales
    .Join(products, s => s.ProductId, p => p.Id, (s, p) => new { s.StoreId, LineTotal = s.Quantity * p.Price })
    .Join(stores,   r => r.StoreId,  st => st.Id, (r, st) => new { st.City, r.LineTotal })
    .GroupBy(x => x.City)
    .Select(g => new { City = g.Key, Total = g.Sum(x => x.LineTotal) })
    .OrderByDescending(x => x.Total);

Console.WriteLine("A. Revenue by store (Join, INNER):");
foreach (var r in revenueByStore)
    Console.WriteLine($"  {r.City}: {r.Total:N0} RUB");
// Novosibirsk is absent here — inner join drops it

// ── Report B: all stores, even without sales (flat LEFT JOIN) ──
var leftJoin = stores
    .GroupJoin(
        sales,
        st => st.Id,
        s => s.StoreId,
        (st, salesGroup) => new { st, salesGroup })
    .SelectMany(
        x => x.salesGroup.DefaultIfEmpty(),
        (x, s) => new
        {
            x.st.City,
            LineTotal = s is null ? 0m : s.Quantity * products.First(p => p.Id == s.ProductId).Price,
            HasSales = s is not null,
        })
    .GroupBy(x => x.City)
    .Select(g => new { City = g.Key, Total = g.Sum(x => x.LineTotal), AnySales = g.Any(x => x.HasSales) })
    .OrderByDescending(x => x.Total);

Console.WriteLine("\nB. All stores (GroupJoin + SelectMany + DefaultIfEmpty, LEFT JOIN):");
foreach (var r in leftJoin)
    Console.WriteLine(r.AnySales ? $"  {r.City}: {r.Total:N0} RUB" : $"  {r.City}: 0 RUB (no sales)");

// ── Report C: master–detail hierarchy via GroupJoin ──
var hierarchy = stores
    .GroupJoin(
        sales,
        st => st.Id,
        s => s.StoreId,
        (st, salesGroup) => new
        {
            Store = st,
            Sales = salesGroup.ToList(),
            Total = salesGroup.Sum(s => s.Quantity * products.First(p => p.Id == s.ProductId).Price),
        });

Console.WriteLine("\nC. Master–detail hierarchy (GroupJoin):");
foreach (var h in hierarchy)
{
    Console.WriteLine($"  {h.Store.City} (region {h.Store.Region}): sales={h.Sales.Count}, total={h.Total:N0} RUB");
    foreach (var s in h.Sales)
        Console.WriteLine($"    - Sale#{s.Id}: product={s.ProductId}, qty={s.Quantity}");
}

// ── Report D: products and category ratings, composite key ──
var productsWithRating = products
    .Join(
        ratings,
        p => new { p.Category, Month = 5 },
        r => new { r.Category, r.Month },
        (p, r) => new { p.Name, p.Category, r.Rating })
    .OrderBy(x => x.Category)
    .ThenBy(x => x.Name);

Console.WriteLine("\nD. Products and ratings (Join, composite key):");
foreach (var r in productsWithRating)
    Console.WriteLine($"  {r.Name} [{r.Category}] — rating {r.Rating}");

// ── Report E: plan vs actual via Zip ──
string[] plannedNames = ["Revenue", "Average check", "Conversion"];
decimal[] actualValues = [1_250_000m, 4_300m, 3.2m];
if (plannedNames.Length != actualValues.Length)
    throw new InvalidOperationException(
        $"Lengths differ: {plannedNames.Length} vs {actualValues.Length}");
var planVsActual = plannedNames.Zip(actualValues, (name, value) => new { name, value });

Console.WriteLine("\nE. Plan vs actual (Zip by index):");
foreach (var r in planVsActual)
    Console.WriteLine($"  {r.name}: {r.value}");

// ── Micro-benchmark: naive loop vs Join ──
var rnd = new Random(42);
var bigSales = Enumerable.Range(0, 20_000)
    .Select(i => new Sale(i, rnd.Next(1, 5), rnd.Next(10, 23), rnd.Next(1, 10), DateTime.Today)).ToList();
var bigProducts = Enumerable.Range(10, 13).Select(i => new Product(i, $"P{i}", "X", 100m)).ToList();

var sw = Stopwatch.StartNew();
decimal naive = 0;
foreach (var s in bigSales)
    foreach (var p in bigProducts)
        if (s.ProductId == p.Id)
            naive += s.Quantity * p.Price;
sw.Stop();
var naiveMs = sw.ElapsedMilliseconds;

sw.Restart();
decimal joined = bigSales
    .Join(bigProducts, s => s.ProductId, p => p.Id, (s, p) => s.Quantity * p.Price)
    .Sum();
sw.Stop();
var joinMs = sw.ElapsedMilliseconds;

Console.WriteLine($"\nBenchmark: naive={naiveMs} ms ({naive:N0}), Join={joinMs} ms ({joined:N0})");
Console.WriteLine(naive == joined ? "✅ Sums match." : "❌ Sums DIFFER — bug in the logic.");
```

**Walk-through, line by line.** The models are declared as `record` because the key common mistake from the lesson is reference-type keys without overridden equality; `record` provides structural equality automatically, so if you decide to use a model itself as a join key (say, `Product` as a key in some join), it will be compared by property values, not by reference. The data is built with C# 12 collection expressions (`[ ... ]`), which is more compact than `new List<T> { ... }`. The Novosibirsk store (Id=4) is intentionally given no sales — this is the boundary case that exposes the difference between `Join` and `GroupJoin`: in report A it is absent (inner join drops it), while in reports B and C it appears (left join preserves it). Report A applies `Join` twice: first it joins sales to products on `ProductId` to obtain `LineTotal = Quantity * Price`, then it joins the result to stores on `StoreId` to get the city name. The argument order is strictly `(outer, inner, outerKey, innerKey, result)` — the very signature beginners trip over; here it is applied correctly. A final `GroupBy` by city and `Sum` produce the revenue. Report B implements the flat `LEFT JOIN` through the canonical idiom `GroupJoin` → `SelectMany` → `DefaultIfEmpty`: `GroupJoin` keeps every store and builds a group of sales for each (possibly empty), `SelectMany` flattens the groups into rows, and `DefaultIfEmpty` guarantees that a store with no sales still yields one row with a `null` sale. In the projector we use `s is null` and `s is not null` (C# 12 pattern matching) rather than `s == null`, and `x => x.LineTotal` via a ternary — this guards against the `NullReferenceException` that a naive `s.Quantity` would inevitably throw. Report C leaves `GroupJoin` “as is”, without flattening — this is the hierarchical “store and its list of sales” structure that reads more naturally for master–detail reports. Report D uses a composite key via the anonymous types `new { p.Category, Month = 5 }` and `new { r.Category, r.Month }`: this works because anonymous types in C# implement structural equality — two anonymous objects are equal when all their properties, with matching types, match in the same order. Property order matters: `new { a, b }` is not equal to `new { b, a }`. Report E uses `Zip` for positional pairing of two equal-length arrays; before the call we explicitly check `plannedNames.Length == actualValues.Length`, because otherwise `Zip` silently truncates to the shorter one — a separate common mistake from the lesson. The micro-benchmark compares a naive nested `foreach` (`O(n·m)`) with `Join` (`O(n+m)`) on 20,000 sales and 13 products; at that scale the difference becomes visible, and it cements the intuition for why `Join` is preferable to nested `Where` loops. The final `naive == joined` check is “trust but verify”: the two implementations must produce the same total, otherwise there is a bug in the logic somewhere.

#### Going deeper (bonus)
1. **Port reports A–D to EF Core.** Build a `DbContext` with `DbSet<Store>`, `DbSet<Product>`, `DbSet<Sale>`, `DbSet<CategoryRating>`, write the same queries as LINQ over `DbSet`, and verify through SQL logging that joins translate to `INNER JOIN`/`LEFT JOIN`. Compare with the LINQ to Objects behavior.
2. **Implement an anti-join.** Print products that have no sales at all, using `GroupJoin` + `Where(g => !g.Any())` or `Where` + `Contains`. Explain why `Join` is not a good fit here.
3. **Zip three sequences.** .NET 9 introduces a three-argument `Zip`. Implement the equivalent by hand on .NET 8: combine three arrays `names`, `values`, `units` by index into a single projection, with a length check.
4. **Memory profiling.** Run `dotnet-counters` or `dotMemory` on the micro-benchmark: show that `Join` allocates one dictionary, while the naive loop allocates none but burns more CPU. Explain the “memory vs time” trade-off.

---

#### Чек-лист сдачи / Submission checklist
- [ ] (RU) Проект `JoinZipHomework` создаётся и собирается без ошибок на .NET 8 / C# 12.
- [ ] (RU) Четыре `record`-модели описаны; есть магазин без продаж и товар без продаж.
- [ ] (RU) Все пять отчётов (A–E) выводятся корректно; магазин без продаж есть только в B и C.
- [ ] (RU) Использованы `Join`, `GroupJoin` (включая плоский `LEFT JOIN`) и `Zip` минимум по разу.
- [ ] (RU) Составной ключ — через анонимный тип; нет ручных `Equals`/`GetHashCode`; нет соединений в цикле.
- [ ] (RU) Микро-бенчмарк сравнивает вложенный цикл и `Join`; суммы совпадают.
- [ ] (RU) Комментарии на RU+EN объясняют выбор оператора.
- [ ] (EN) The `JoinZipHomework` project builds without errors on .NET 8 / C# 12.
- [ ] (EN) Four `record` models are declared; there is a store with no sales and a product with no sales.
- [ ] (EN) All five reports (A–E) print correctly; the store with no sales appears only in B and C.
- [ ] (EN) `Join`, `GroupJoin` (including flat `LEFT JOIN`), and `Zip` are each used at least once.
- [ ] (EN) The composite key uses an anonymous type; no manual `Equals`/`GetHashCode`; no joins inside loops.
- [ ] (EN) The micro-benchmark compares the nested loop and `Join`; the totals match.
- [ ] (EN) RU+EN comments explain the choice of operator.

#### Ресурсы / Resources
- [Microsoft Learn — Join operations](https://learn.microsoft.com/dotnet/csharp/linq/standard-query-operators/join-operations)
- [Microsoft Learn — Enumerable.Join](https://learn.microsoft.com/dotnet/api/system.linq.enumerable.join)
- [Microsoft Learn — Enumerable.GroupJoin](https://learn.microsoft.com/dotnet/api/system.linq.enumerable.groupjoin)
- [Microsoft Learn — Enumerable.Zip](https://learn.microsoft.com/dotnet/api/system.linq.enumerable.zip)
- [Microsoft Learn — C# 12 collection expressions](https://learn.microsoft.com/dotnet/csharp/language-reference/operators/collection-expressions)
- [Microsoft Learn — records](https://learn.microsoft.com/dotnet/csharp/fundamentals/types/records)

---
[← К уроку M08-L05](lesson-M08-L05-join-zip.md) | [⬆ К модулю M08](../README.md) | [Следующее ДЗ →](homework-M08-L06-aggregates.md)
