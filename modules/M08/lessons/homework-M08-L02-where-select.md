---
[← К уроку M08-L02](lesson-M08-L02-where-select.md) | [⬆ К модулю M08](../README.md) | [Следующее ДЗ →](homework-M08-L03-orderby-thenby.md)
---

### Домашнее задание M08-L02: Where, Select, проекции / Homework M08-L02: Where, Select, projections

**Урок / Lesson:** M08-L02
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 3/5
**Цель / Goal:** (RU) Научиться осмысленно применять операторы `Where` и `Select`, строить цепочки запросов, использовать анонимные типы и кортежи для проекций, сплющивать вложенные коллекции через `SelectMany`, понимать отложенное исполнение и отличать его от материализации, а также свободно переходить между fluent- и query-синтаксисом. (EN) Learn to apply `Where` and `Select` deliberately, build query chains, use anonymous types and tuples for projections, flatten nested collections with `SelectMany`, understand deferred execution versus materialization, and move fluently between fluent and query syntax.

#### Связь с уроком / Connection to the lesson
(RU) Урок M08-L02 вводит два фундаментальных оператора LINQ — `Where` и `Select`, — а также анонимные типы, `SelectMany`, цепочки и отложенное исполнение. Это ДЗ закрепляет каждую из этих концепций через реалистичную задачу анализа каталога товаров и заказов: вы будете фильтровать, проектировать, сплющивать и материализовать данные, наступая на те же грабли, что описаны в разделе «Частые ошибки», и обходя их осознанно.
(EN) Lesson M08-L02 introduces the two foundational LINQ operators — `Where` and `Select` — plus anonymous types, `SelectMany`, chaining, and deferred execution. This homework anchors every one of those concepts through a realistic task of analysing a product catalogue and orders: you will filter, project, flatten, and materialize data, stepping on the same rakes described in the "Common Mistakes" section — and sidestepping them on purpose.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация
Представьте, что вы работаете в небольшой онлайн-платформе продажи цифровых товаров и подписок. У вас есть каталог продуктов (курсы, электронные книги, подписки), каждый из которых характеризуется идентификатором, названием, ценой, категорией, набором тегов и средним рейтингом покупателей. Поверх каталога лежит журнал заказов: каждый заказ содержит список позиций (ссылок на продукты и количеством), дату оформления и идентификатор покупателя.

Маркетинговая команда просит вас подготовить несколько аналитических выборок: какие товары дороги и популярны, какие теги встречаются чаще всего, какие позиции попадают в заказы определённой категории покупателей, какой будет налоговая нагрузка по каждой позиции. Всё это нужно получить в виде готовых к выводу структур, без ручных циклов и временных списков — только LINQ. При этом вы обязаны доказать команде, что понимаете разницу между отложенным исполнением и материализацией: каждый отчёт, который будет перечисляться больше одного раза или который может «протухнуть» из-за изменения исходных данных, должен быть заморожен через `ToList()` или `ToArray()`.

Именно в такой реальной работе ярко проявляются темы урока: порядок `Where` и `Select` в цепочке, выбор между анонимным типом и кортежем, осознанное применение `SelectMany` для «разворачивания» заказов в плоский поток позиций, эквивалентность fluent- и query-синтаксиса, и опасности повторного перечисления LINQ-запроса над изменяемым источником. Решая задачу, вы не просто напишете код — вы научитесь аргументировать, почему он написан именно так.

#### Что нужно сделать (пошагово)
1. Создайте консольный проект на .NET 8 командой `dotnet new console -n M08L02Homework -o M08L02Homework` из корня репозитория курса. Убедитесь, что в `M08L02Homework.csproj` указаны `<TargetFramework>net8.0</TargetFramework>` и `<LangVersion>latest</LangVersion>`. Откройте `Program.cs` и удалите шаблонный `Console.WriteLine("Hello, World")`.

2. В отдельном файле `Models.cs` (в том же проекте) объявите типы данных каталога с помощью `record` и collection expressions, где это уместно. Минимально нужны: `record Product(int Id, string Name, decimal Price, string Category, List<string> Tags, double Rating);` и `record Order(int OrderId, DateTime PlacedAt, string CustomerId, List<OrderLine> Lines);`, где `record OrderLine(int ProductId, int Qty);`. Добавьте вспомогательный метод-фабрику или статический класс `Seed` с методом `Catalog()` и `Orders()`, возвращающими исходные данные.

3. В `Program.cs` реализуйте метод-точку входа (top-level statements) и набор статических методов-отчётов в классе `Reports`. Каждый метод возвращает конкретный, готовый к печати результат (анонимный тип, кортеж или именованный record) и принимает `IEnumerable<Product>` и/или `IEnumerable<Order>`. Никаких `Console.WriteLine` внутри методов-отчётов, кроме специального демо-метода (см. шаг 6) — печать ведётся из `Program.cs`, чтобы отчёты оставались чистыми функциями.

4. Реализуйте следующие отчёты, строго соблюдая порядок операторов и материаллизуя результат там, где это нужно:
   - `ExpensivePopular()` — товары с ценой выше 100 и рейтингом не ниже 4.5; вернуть список анонимных объектов `{ Name, Price, Rating }`. Фильтр (`Where`) должен стоять раньше проекции (`Select`).
   - `PriceListWithTax()` — для каждого товара вернуть кортеж `(string Name, decimal Price, decimal Tax)`, где `Tax = Price * 0.20m`. Подумайте, почему здесь удобнее кортеж, а не анонимный тип.
   - `AllTags()` — все уникальные теги каталога через `SelectMany` + `Distinct`. Верните `IReadOnlyCollection<string>`.
   - `TopDeals(int n)` — товары с тегом `"sale"`, спроецированные в `{ Name, Price }`, отсортированные по цене (это будет формально тема следующего урока, но для целостности вывода разрешено использовать `OrderBy`), и ограниченные `Take(n)`. Сравните fluent- и query-запись.
   - `ItemsSoldByCategory(string category)` — через `SelectMany` по заказам собрать все `OrderLine`, отфильтровать те, чей `ProductId` относится к товару указанной категории, и вернуть суммарное количество проданных единиц. Здесь обязательно нужен «join» по `ProductId` — реализуйте через `from o in orders from line in o.Lines join p in products on line.ProductId equals p.Id where p.Category == category select line`.

5. Напишите метод `DeferredExecutionDemo()`, который наглядно показывает разницу между отложенным исполнением и материализацией. Алгоритм: постройте запрос `var q = products.Where(p => p.Price < 100m).Select(p => p.Name);`, затем добавьте в `products` новый товар ценой 50, затем перечислите `q` и выведите результат (новый товар должен появиться). После этого заморозьте второй запрос через `ToList()`, добавьте ещё один дешёвый товар и покажите, что замороженный список не изменился. Вывод программы должен явно это демонстрировать.

6. Добавьте метод `MutationSafetyDemo()`, который намеренно мутирует коллекцию во время `foreach` по живому LINQ-запросу — закомментируйте проблемную строку и оставьте комментарий, объясняющий, что произойдёт `InvalidOperationException`, и как `ToList()` перед мутацией решает проблему.

7. Скомпилируйте проект командой `dotnet build` из папки проекта. Запустите `dotnet run` и убедитесь, что вывод содержит все требуемые отчёты и оба демо. Зафиксируйте ожидаемый вывод в комментарии в начале `Program.cs`.

8. Покройте ключевые методы юнит-тестами в отдельном проекте `M08L02Homework.Tests` (создаётся `dotnet new xunit`). Тесты должны проверять: корректность фильтра `ExpensivePopular`, что `AllTags` действительно содержит только уникальные значения, что `ItemsSoldByCategory` правильно суммирует количества, и что `DeferredExecutionDemo` возвращает разные результаты до и после добавления товара при отложенном исполнении и одинаковые при материализованном.

#### Требования к решению
Решение должно использовать возможности C# 12 и .NET 8: top-level statements в `Program.cs`, `record` для моделей данных, collection expressions (`new() { ... }` или `[ ... ]`) для инициализации, pattern matching там, где он делает код яснее (например, `p is { Price: > 100, Rating: >= 4.5 }`), а также raw string literals (`"""..."""`) для многострочных текстовых блоков вывода, если они уместны. Версия языка должна быть явно указана в `.csproj`.

Все методы-отчёты обязаны соблюдать правило «`Where` раньше `Select`», если только проекция не выполняется раньше по принципиальной причине (такую причину нужно прокомментировать). Результаты, которые потенциально перечисляются больше одного раза или могут быть «отравлены» изменением источника, должны быть материализованы через `ToList()`/`ToArray()`. Возвращать анонимный тип из метода нельзя — если анонимный тип нужен как результат, метод должен либо возвращать кортеж, либо именованный `record`, либо принимать `Func` обратного вызова для печати; выберите подходящее решение и обоснуйте его.

Код должен быть разделён по файлам: `Models.cs` — типы и сид-данные, `Reports.cs` — методы-отчёты, `Program.cs` — точка входа и печать. Имена лямбда-параметров должны быть говорящими (`product`, `order`, `line`), а не однобуквенными, если внутри больше одного уровня вложенности. Каждое нетривиальное решение сопровождается комментарием на русском и английском (`// RU: ... / EN: ...`), объясняющим, какой концепции урока оно соответствует.

Проект должен собираться без предупреждений (`TreatWarningsAsErrors` можно включить опционально), а тесты — проходить. Запрещено использовать `.ForEach` над LINQ-результатом и явные циклы `for`/`foreach` внутри методов-отчётов (только LINQ). Циклы разрешены только в `Program.cs` для печати результатов.

#### Тонкости и подводные камни
Главная тонкость, которую подчёркивает урок — порядок `Where` и `Select`. Если поставить `Select` раньше и проекция выбросит поле, нужное для фильтра (например, вы спроецировали `{ Name, Price }`, а потом пытаетесь отфильтровать по `Rating`), код либо не скомпилируется, либо придётся переписывать запрос. Поэтому стандартное правило: фильтруем по «богатому» типу, а уже потом投影ируем в «тощую» форму. Это не только эстетика — это и производительность (не тратим работу на преобразование того, что будет отброшено), и безопасность (поля фильтра остаются в области видимости).

Вторая тонкость — отложенное исполнение. Новички часто удивляются: построили запрос, ничего не вызвав, потом изменили исходный список, потом перечислили — и видят изменённый результат. Урок прямо говорит: `Where` и `Select` лишь строят «рецепт». Если нужно зафиксировать снимок — обязательно `ToList()` или `ToArray()`. Особенно коварно это при повторном перечислении: вызов `Count()` и потом `foreach` по тому же `IEnumerable` дважды прогонят запрос. Для дорогих источников (файлы, сеть, базы) это скрытая потеря производительности.

Третья тонкость — `Select` vs `SelectMany`. Если ваша лямбда возвращает коллекцию (`p.Tags`, `o.Lines`), то обычный `Select` даст `IEnumerable<IEnumerable<T>>` — «последовательность последовательностей». Чтобы получить плоский поток, нужен `SelectMany`. Запомните эвристику: проекция возвращает коллекцию и вы хотите «высыпать всё на один стол» — берите `SelectMany`.

Четвёртая тонкость — анонимные типы и их ограничения. Анонимный тип удобен для локальных форм, но его нельзя вернуть из метода (имя типа неизвестно в сигнатуре). Возвращайте кортеж или именованный record. Также помните: анонимные типы имеют value-based `Equals`/`GetHashCode`, что делает их удобными для `Distinct`/`GroupBy` в рамках одного выражения, но跨-методами они бесполезны.

Пятая тонкость — мутация коллекции во время `foreach` по живому LINQ-запросу. Если источник поддерживает уведомления об изменениях (например, `List<T>` при перечислении через `List.Enumerator` проверяет `version`), вы получите `InvalidOperationException: Collection was modified`. Решение: сначала `ToList()`, потом мутируйте исходник.

Шестая тонкость — выбор синтаксиса. Query-синтаксис лаконичнее для нескольких `from` (они компилируются в `SelectMany`) и сложных `join`/`group`. Fluent-стиль читается легче для коротких цепочек. Урок рекомендует свободно владеть обоими и выбирать по читаемости. В ДЗ вы должны показать оба варианта хотя бы один раз.

#### Критерии приёмки
- [ ] Созданы проекты `M08L02Homework` и `M08L02Homework.Tests`, оба собираются на .NET 8 без ошибок и предупреждений.
- [ ] В `.csproj` явно указаны `<TargetFramework>net8.0</TargetFramework>` и `<LangVersion>latest</LangVersion>`.
- [ ] Модели объявлены через `record` с обязательными полями: `Product`, `Order`, `OrderLine`.
- [ ] Метод `ExpensivePopular` возвращает список анонимных объектов `{ Name, Price, Rating }`, фильтр `Where` стоит раньше `Select`.
- [ ] Метод `PriceListWithTax` возвращает кортежи `(string Name, decimal Price, decimal Tax)` с `Tax = Price * 0.20m`.
- [ ] Метод `AllTags` использует `SelectMany` + `Distinct` и возвращает уникальные теги.
- [ ] Метод `TopDeals` реализован дважды: fluent- и query-синтаксисом; результаты идентичны.
- [ ] Метод `ItemsSoldByCategory` реализует «join» по `ProductId` через query-синтаксис и корректно суммирует количества.
- [ ] Метод `DeferredExecutionDemo` наглядно показывает разницу: новый товар виден в отложенном запросе, но не виден в замороженном через `ToList()`.
- [ ] Метод `MutationSafetyDemo` содержит закомментированную проблемную строку и комментарий с объяснением `InvalidOperationException` и решения через `ToList()`.
- [ ] Внутри методов-отчётов нет явных циклов `for`/`foreach` и нет `.ForEach` — только LINQ; циклы только в `Program.cs` для печати.
- [ ] Использованы возможности C# 12: top-level statements, pattern matching (`is { ... }`), collection expressions или `new()` инициализаторы.
- [ ] Каждое нетривиальное решение сопровождается двуязычным комментарием `// RU: ... / EN: ...`, ссылающимся на концепцию урока.
- [ ] Юнит-тесты покрывают минимум 4 ключевых метода и проходят (`dotnet test` зелёный).
- [ ] В начале `Program.cs` зафиксирован ожидаемый вывод в комментарии.

#### Подсказки (без прямого ответа)
- Если фильтр не компилируется после `Select` — спросите себя: какое поле вы выбросили проекцией? Поднимите `Where` выше.
- Для кортежа с именованными полями используйте синтаксис `(Name: p.Name, Price: p.Price, Tax: p.Price * 0.20m)`.
- `SelectMany` принимает `Func<T, IEnumerable<U>>` — если ваша лямбда возвращает коллекцию, это сигнал.
- Для «join» по идентификатору в query-синтаксисе: `from o in orders from line in o.Lines join p in products on line.ProductId equals p.Id where ... select ...`.
- Чтобы доказать отложенное исполнение, меняйте источник между построением запроса и его перечислением — и наблюдайте, как меняется результат.
- Для материализации снимка вызовите `.ToList()` сразу после построения запроса, до любой мутации.

#### Эталонное решение (разбор)
```csharp
// C# 12 / .NET 8 — эталонное решение ДЗ M08-L02 / reference solution
// Файл: Models.cs
using System;
using System.Collections.Generic;

namespace M08L02Homework;

// RU: Модель продукта — record даёт value-равенство, удобно для LINQ-операций.
// EN: Product model — record gives value equality, handy for LINQ operators.
public record Product(
    int Id,
    string Name,
    decimal Price,
    string Category,
    List<string> Tags,
    double Rating);

// RU: Позиция заказа ссылается на ProductId, а не на сам Product — нормальная форма.
// EN: Order line references ProductId rather than Product itself — normal form.
public record OrderLine(int ProductId, int Qty);

// RU: Заказ содержит список позиций — это вложенная коллекция для SelectMany.
// EN: Order contains a list of lines — a nested collection for SelectMany.
public record Order(int OrderId, DateTime PlacedAt, string CustomerId, List<OrderLine> Lines);

// RU: Сид-данные в виде статического класса-фабрики — чистая функция, без побочных эффектов.
// EN: Seed data as a static factory class — pure function, no side effects.
public static class Seed
{
    public static List<Product> Catalog() =>
    [
        new(1, "Курс по C# / C# Course",       199m,  "Courses",    ["popular", "new"],      4.7),
        new(2, "Книга по LINQ / LINQ Book",     29m,   "Books",      ["sale"],                4.2),
        new(3, "Подписка Pro / Pro Subscription", 149m, "Subscriptions", ["popular"],        4.6),
        new(4, "Курс по F# / F# Course",         249m, "Courses",    ["new"],                4.3),
        new(5, "Книга по ASP.NET / ASP.NET Book", 39m, "Books",      ["sale", "popular"],    4.8),
        new(6, "Подписка Basic / Basic Subscription", 49m, "Subscriptions", ["sale"],        4.1),
    ];

    public static List<Order> Orders() =>
    [
        new(1001, new(2024, 5, 1), "alice",
            [new(1, 1), new(2, 2)]),
        new(1002, new(2024, 5, 3), "bob",
            [new(3, 1), new(5, 3)]),
        new(1003, new(2024, 5, 4), "alice",
            [new(4, 1), new(2, 1)]),
    ];
}
```

```csharp
// Файл: Reports.cs
using System;
using System.Collections.Generic;
using System.Linq;

namespace M08L02Homework;

public static class Reports
{
    // RU: Дорогие и популярные — Where раньше Select: не тратим работу на проекцию лишнего.
    // EN: Expensive and popular — Where before Select: do not waste work projecting extras.
    public static List<(string Name, decimal Price, double Rating)> ExpensivePopular(
        IEnumerable<Product> products)
        => products
            .Where(p => p is { Price: > 100m, Rating: >= 4.5 })   // pattern matching C# 12
            .Select(p => (p.Name, p.Price, p.Rating))             // кортеж — можно вернуть из метода
            .ToList();                                            // материализация: фиксируем снимок

    // RU: Прайс с налогом — кортеж удобнее анонимного типа, т.к. метод его возвращает.
    // EN: Price list with tax — tuple is handier than anonymous type since the method returns it.
    public static List<(string Name, decimal Price, decimal Tax)> PriceListWithTax(
        IEnumerable<Product> products)
        => products
            .Select(p => (p.Name, p.Price, Tax: p.Price * 0.20m))
            .ToList();

    // RU: Все теги каталога — SelectMany сплющивает List<List<string>> в плоский поток.
    // EN: All catalogue tags — SelectMany flattens List<List<string>> into a flat stream.
    public static IReadOnlyCollection<string> AllTags(IEnumerable<Product> products)
        => products
            .SelectMany(p => p.Tags)
            .Distinct()
            .OrderBy(t => t)
            .ToList();

    // RU: Топ скидок — fluent-стиль; OrderBy здесь для целостности вывода.
    // EN: Top deals — fluent style; OrderBy here is for output coherence.
    public static List<(string Name, decimal Price)> TopDealsFluent(IEnumerable<Product> products, int n)
        => products
            .Where(p => p.Tags.Contains("sale"))
            .Select(p => (p.Name, p.Price))
            .OrderBy(x => x.Price)
            .Take(n)
            .ToList();

    // RU: Топ скидок — query-синтаксис; компилируется в ту же цепочку, что и fluent-вариант.
    // EN: Top deals — query syntax; compiles to the same chain as the fluent version.
    public static List<(string Name, decimal Price)> TopDealsQuery(IEnumerable<Product> products, int n)
        => (from p in products
            where p.Tags.Contains("sale")
            orderby p.Price
            select (p.Name, p.Price))
            .Take(n)
            .ToList();

    // RU: Сумма проданных единиц по категории — SelectMany по заказам + join по ProductId.
    // EN: Sum of sold units per category — SelectMany over orders + join on ProductId.
    public static int ItemsSoldByCategory(
        IEnumerable<Product> products,
        IEnumerable<Order> orders,
        string category)
        => (from o in orders
            from line in o.Lines
            join p in products on line.ProductId equals p.Id
            where p.Category == category
            select line.Qty).Sum();

    // RU: Демо отложенного исполнения: новый товар виден в отложенном, но не в замороженном.
    // EN: Deferred execution demo: a new product is visible in deferred, not in frozen.
    public static (List<string> Deferred, List<string> Frozen) DeferredExecutionDemo(List<Product> products)
    {
        var deferred = products.Where(p => p.Price < 100m).Select(p => p.Name);
        products.Add(new Product(99, "Временный / Temp", 50m, "Books", [], 3.0));
        var deferredSnapshot = deferred.ToList();           // выполнится СЕЙЧАС, увидит временный товар

        var frozen = products.Where(p => p.Price < 100m).Select(p => p.Name).ToList();
        products.Add(new Product(98, "Ещё временный / Temp2", 30m, "Books", [], 3.0));
        return (deferredSnapshot, frozen);                  // frozen не содержит второй временный товар
    }
}
```

Разбор по строкам. В `ExpensivePopular` мы намеренно ставим `Where` раньше `Select`: фильтр работает по «богатому» `Product`, у которого доступны `Price` и `Rating`. Здесь же применён pattern matching `is { Price: > 100m, Rating: >= 4.5 }` — идиома C# 12, лаконично выражающая составное условие. Проекция в кортеж `(p.Name, p.Price, p.Rating)` выбрана потому, что метод должен вернуть результат наружу, а анонимный тип в сигнатуре объявить нельзя — это та самая «частая ошибка» из урока. `ToList()` материализует снимок, чтобы дальнейшие изменения `products` не «отравили» отчёт.

В `AllTags` ключевой момент — `SelectMany(p => p.Tags)`. Лямбда возвращает коллекцию (`List<string>`), обычный `Select` дал бы `IEnumerable<List<string>>`. `SelectMany` сплющивает все теги в один плоский поток, после чего `Distinct` оставляет уникальные. Порядок `SelectMany` → `Distinct` → `OrderBy` показывает каноническую цепочку: сначала сплющивание, потом дедупликация, потом сортировка для вывода.

В `TopDealsFluent` и `TopDealsQuery` показана эквивалентность двух синтаксисов. `Take(n)` навешивается снаружи query-выражения, потому что `orderby`/`where`/`select` — это «тело» query, а `Take` — оператор над результатом. Это типичный приём: query-синтаксис описывает декларативную часть, fluent-операторы навешиваются снаружи. Оба метода компилируются в одну и ту же цепочку вызовов расширений.

В `ItemsSoldByCategory` реализован классический «join» в query-синтаксисе: два `from` подряд (`from o in orders from line in o.Lines`) разворачиваются компилятором в `SelectMany`, а `join p in products on line.ProductId equals p.Id` выполняет соединение по ключу. Этот фрагмент нельзя лаконично переписать в fluent-стиле без `Join`-оператора и вспомогательного анонимного типа, поэтому query-синтаксис здесь оправдан — это та рекомендация из best practices урока.

В `DeferredExecutionDemo` первая часть строит отложенный запрос, но не выполняет его. Добавление товара в `products` после построения, но до `ToList()` означает, что при материализации `deferredSnapshot` временный товар попадёт в результат — это и есть «отложенное исполнение». Вторая часть сначала материализует `frozen` через `ToList()`, и только потом добавляется второй временный товар — `frozen` его не видит, потому что снимок уже зафиксирован. Этот метод — наглядная иллюстрация правила из урока: «если нужно зафиксировать — `ToList()`».

В `Models.cs` использованы collection expressions (`[ ... ]`) — новинка C# 12 для инициализации `List<T>` и массивов. `record` для моделей даёт value-равенство, что критично для корректной работы `Distinct`/`Contains`/`GroupBy`. `Seed` оформлен как статический класс-фабрика, возвращающий новые списки при каждом вызове, — это гарантирует, что тесты и демо работают с независимыми копиями данных и не «отравляют» друг друга.

#### Задания на углубление (бонус)
1. Реализуйте отчёт, который возвращает топ-3 покупателя по сумме стоимости их заказов. Потребуется `SelectMany` + `Join` + `GroupBy` + `Sum` + `OrderByDescending` + `Take`. Подумайте, где здесь материализовать результат.
2. Перепишите `ItemsSoldByCategory` в fluent-стиле через `SelectMany` + `Join`. Сравните читаемость. Сделайте вывод, когда query-синтаксис действительно лучше.
3. Добавьте метод `TagFrequency()`, возвращающий словарь `Dictionary<string, int>` частоты тегов во всём каталоге. Используйте `SelectMany` + `GroupBy` + `ToDictionary`.
4. Исследуйте производительность: измерьте через `Stopwatch` время повторного перечисления LINQ-запроса над большим списком (сгенерируйте 100 000 товаров) — один раз через живой `IEnumerable`, второй — через `ToList()`. Объясните разницу через концепцию отложенного исполнения.

---

## Statement in English / Постановка на английском

#### Context & motivation
Imagine you work at a small online platform selling digital goods and subscriptions. You have a catalogue of products (courses, e-books, subscriptions); each product has an identifier, a name, a price, a category, a set of tags, and an average customer rating. On top of the catalogue sits an order journal: every order carries a list of line items (references to products with a quantity), a placement date, and a customer identifier.

The marketing team asks you to prepare several analytical slices: which products are expensive and popular, which tags occur most often, which line items land in orders from a particular customer segment, what the tax burden per line item would be. All of this must be produced as ready-to-print structures, without manual loops or scratch lists — only LINQ. At the same time, you must prove to the team that you understand the difference between deferred execution and materialization: every report that will be enumerated more than once, or that could "go stale" because the source data changes, must be frozen with `ToList()` or `ToArray()`.

It is exactly in this kind of real work that the lesson's themes come to the surface: the order of `Where` and `Select` in a chain, the choice between an anonymous type and a tuple, the deliberate use of `SelectMany` to "unroll" orders into a flat stream of line items, the equivalence of fluent and query syntax, and the dangers of re-enumerating a LINQ query over a mutable source. By solving the task, you do not merely write code — you learn to argue why it is written exactly this way.

#### What to do step by step
1. Create a .NET 8 console project with `dotnet new console -n M08L02Homework -o M08L02Homework` from the course repository root. Make sure `M08L02Homework.csproj` contains `<TargetFramework>net8.0</TargetFramework>` and `<LangVersion>latest</LangVersion>`. Open `Program.cs` and remove the boilerplate `Console.WriteLine("Hello, World")`.

2. In a separate `Models.cs` file (in the same project), declare the catalogue data types using `record` and collection expressions where appropriate. At minimum you need: `record Product(int Id, string Name, decimal Price, string Category, List<string> Tags, double Rating);` and `record Order(int OrderId, DateTime PlacedAt, string CustomerId, List<OrderLine> Lines);` where `record OrderLine(int ProductId, int Qty);`. Add a static `Seed` class with `Catalog()` and `Orders()` methods returning the source data.

3. In `Program.cs`, implement the entry point (top-level statements) and a set of static report methods inside a `Reports` class. Each method returns a concrete, ready-to-print result (an anonymous type, a tuple, or a named record) and accepts `IEnumerable<Product>` and/or `IEnumerable<Order>`. No `Console.WriteLine` inside the report methods — except for the special demo method (see step 6). Printing is done from `Program.cs` so the reports stay pure functions.

4. Implement the following reports, strictly respecting operator order and materializing the result where needed:
   - `ExpensivePopular()` — products priced above 100 with a rating of at least 4.5; return a list of anonymous objects `{ Name, Price, Rating }`. The `Where` filter must come before the `Select` projection.
   - `PriceListWithTax()` — for every product return a tuple `(string Name, decimal Price, decimal Tax)` where `Tax = Price * 0.20m`. Think about why a tuple is handier here than an anonymous type.
   - `AllTags()` — all unique catalogue tags via `SelectMany` + `Distinct`. Return `IReadOnlyCollection<string>`.
   - `TopDeals(int n)` — products tagged `"sale"`, projected into `{ Name, Price }`, ordered by price (this is formally the next lesson's topic, but allowed here for output coherence), and limited with `Take(n)`. Compare the fluent and query-syntax versions.
   - `ItemsSoldByCategory(string category)` — via `SelectMany` over orders, collect all `OrderLine`s, filter those whose `ProductId` belongs to a product of the given category, and return the total quantity sold. A "join" on `ProductId` is required — implement it with `from o in orders from line in o.Lines join p in products on line.ProductId equals p.Id where p.Category == category select line`.

5. Write a `DeferredExecutionDemo()` method that vividly shows the difference between deferred execution and materialization. Algorithm: build a query `var q = products.Where(p => p.Price < 100m).Select(p => p.Name);`, then add a new product priced 50 to `products`, then enumerate `q` and print the result (the new product must appear). After that, freeze a second query with `ToList()`, add another cheap product, and show that the frozen list did not change. The program output must demonstrate this explicitly.

6. Add a `MutationSafetyDemo()` method that intentionally mutates a collection during a `foreach` over a live LINQ query — comment out the offending line and leave a comment explaining that an `InvalidOperationException` would be thrown, and how `ToList()` before mutation fixes it.

7. Build the project with `dotnet build` from the project folder. Run `dotnet run` and verify the output contains all the required reports and both demos. Pin the expected output as a comment at the top of `Program.cs`.

8. Cover the key methods with unit tests in a separate `M08L02Homework.Tests` project (created with `dotnet new xunit`). Tests must verify: correctness of the `ExpensivePopular` filter, that `AllTags` really contains only unique values, that `ItemsSoldByCategory` correctly sums quantities, and that `DeferredExecutionDemo` returns different results before and after adding a product under deferred execution, but identical results under materialization.

#### Requirements
The solution must use C# 12 and .NET 8 features: top-level statements in `Program.cs`, `record` for data models, collection expressions (`new() { ... }` or `[ ... ]`) for initialization, pattern matching where it clarifies the code (for instance `p is { Price: > 100, Rating: >= 4.5 }`), and raw string literals (`"""..."""`) for multi-line output text blocks where appropriate. The language version must be set explicitly in `.csproj`.

All report methods must follow the "`Where` before `Select`" rule, unless projection happens earlier for a principled reason (such a reason must be commented). Results that may be enumerated more than once or could be "poisoned" by source mutation must be materialized with `ToList()`/`ToArray()`. Returning an anonymous type from a method is not allowed — if an anonymous type is needed as a result, the method must either return a tuple, a named `record`, or take a printing callback; choose the right approach and justify it.

The code must be split across files: `Models.cs` for types and seed data, `Reports.cs` for report methods, `Program.cs` for the entry point and printing. Lambda parameter names must be meaningful (`product`, `order`, `line`), not single letters, when there is more than one level of nesting inside. Every non-trivial decision must carry a bilingual comment (`// RU: ... / EN: ...`) referencing the lesson concept it applies.

The project must build without warnings (you may optionally enable `TreatWarningsAsErrors`), and the tests must pass. Using `.ForEach` over a LINQ result and explicit `for`/`foreach` loops inside report methods is forbidden (LINQ only). Loops are allowed only in `Program.cs` for printing results.

#### Pitfalls
The main pitfall the lesson highlights is the order of `Where` and `Select`. If you put `Select` first and the projection drops a field the filter needs (for example you projected to `{ Name, Price }` and then try to filter by `Rating`), the code either fails to compile or forces a rewrite. So the standard rule is: filter on the "rich" type, then project to the "lean" shape. This is not just aesthetics — it is performance (no wasted work transforming what will be discarded) and safety (the filter fields stay in scope).

The second pitfall is deferred execution. Newcomers are often surprised: they build a query without running it, then mutate the source list, then enumerate — and see the mutated result. The lesson states plainly: `Where` and `Select` only build a "recipe". If you need a snapshot, you must call `ToList()` or `ToArray()`. This is especially treacherous on re-enumeration: calling `Count()` and then `foreach`-ing the same `IEnumerable` runs the query twice. For expensive sources (files, network, databases) this is a hidden performance trap.

The third pitfall is `Select` versus `SelectMany`. If your lambda returns a collection (`p.Tags`, `o.Lines`), a plain `Select` yields `IEnumerable<IEnumerable<T>>` — a "sequence of sequences". To get a flat stream, you need `SelectMany`. Remember the heuristic: if the projection returns a collection and you want to "dump everything onto one table", reach for `SelectMany`.

The fourth pitfall is anonymous types and their limits. An anonymous type is handy for local shapes, but it cannot be returned from a method (its name is unknown in the signature). Return a tuple or a named record instead. Also note: anonymous types have value-based `Equals`/`GetHashCode`, which makes them convenient for `Distinct`/`GroupBy` within a single expression, but useless across method boundaries.

The fifth pitfall is mutating a collection during a `foreach` over a live LINQ query. If the source tracks mutations (for example `List<T>` checks its `version` field during enumeration), you get `InvalidOperationException: Collection was modified`. Fix: `ToList()` first, then mutate the original.

The sixth pitfall is syntax choice. Query syntax is terser for multiple `from` clauses (they compile into `SelectMany`) and complex `join`/`group`. Fluent style reads better for short chains. The lesson recommends fluency in both and picking by readability. In this homework you must demonstrate both at least once.

#### Acceptance criteria
- [ ] Projects `M08L02Homework` and `M08L02Homework.Tests` are created and both build on .NET 8 with no errors or warnings.
- [ ] `.csproj` explicitly sets `<TargetFramework>net8.0</TargetFramework>` and `<LangVersion>latest</LangVersion>`.
- [ ] Models are declared with `record` and the required fields: `Product`, `Order`, `OrderLine`.
- [ ] Method `ExpensivePopular` returns a list of anonymous objects `{ Name, Price, Rating }`, with `Where` placed before `Select`.
- [ ] Method `PriceListWithTax` returns tuples `(string Name, decimal Price, decimal Tax)` with `Tax = Price * 0.20m`.
- [ ] Method `AllTags` uses `SelectMany` + `Distinct` and returns unique tags.
- [ ] Method `TopDeals` is implemented twice — fluent and query syntax — with identical results.
- [ ] Method `ItemsSoldByCategory` implements a `ProductId` "join" via query syntax and correctly sums quantities.
- [ ] Method `DeferredExecutionDemo` clearly shows the difference: a new product is visible in the deferred query but not in the `ToList()`-frozen one.
- [ ] Method `MutationSafetyDemo` contains the commented-out offending line and a comment explaining the `InvalidOperationException` and the `ToList()` fix.
- [ ] Inside report methods there are no explicit `for`/`foreach` loops and no `.ForEach` — LINQ only; loops live only in `Program.cs` for printing.
- [ ] C# 12 features are used: top-level statements, pattern matching (`is { ... }`), collection expressions or `new()` initializers.
- [ ] Every non-trivial decision carries a bilingual comment `// RU: ... / EN: ...` referencing a lesson concept.
- [ ] Unit tests cover at least 4 key methods and pass (`dotnet test` is green).
- [ ] Expected output is pinned as a comment at the top of `Program.cs`.

#### Hints (no direct answer)
- If the filter does not compile after `Select`, ask yourself: which field did the projection drop? Move `Where` higher.
- For a tuple with named fields, use the syntax `(Name: p.Name, Price: p.Price, Tax: p.Price * 0.20m)`.
- `SelectMany` takes a `Func<T, IEnumerable<U>>` — if your lambda returns a collection, that is the signal.
- For an id-based "join" in query syntax: `from o in orders from line in o.Lines join p in products on line.ProductId equals p.Id where ... select ...`.
- To prove deferred execution, mutate the source between building the query and enumerating it — and observe how the result changes.
- To freeze a snapshot, call `.ToList()` right after building the query, before any mutation.

#### Reference solution walk-through
```csharp
// C# 12 / .NET 8 — reference solution for homework M08-L02
// File: Models.cs
using System;
using System.Collections.Generic;

namespace M08L02Homework;

// EN: Product model — record gives value equality, handy for LINQ operators.
// RU: Модель продукта — record даёт value-равенство, удобно для LINQ-операций.
public record Product(
    int Id,
    string Name,
    decimal Price,
    string Category,
    List<string> Tags,
    double Rating);

// EN: Order line references ProductId rather than Product — normal form.
// RU: Позиция заказа ссылается на ProductId, а не на сам Product — нормальная форма.
public record OrderLine(int ProductId, int Qty);

// EN: Order contains a list of lines — a nested collection for SelectMany.
// RU: Заказ содержит список позиций — это вложенная коллекция для SelectMany.
public record Order(int OrderId, DateTime PlacedAt, string CustomerId, List<OrderLine> Lines);

// EN: Seed data as a static factory class — pure function, no side effects.
// RU: Сид-данные в виде статического класса-фабрики — чистая функция, без побочных эффектов.
public static class Seed
{
    public static List<Product> Catalog() =>
    [
        new(1, "C# Course",       199m,  "Courses",      ["popular", "new"],    4.7),
        new(2, "LINQ Book",        29m,   "Books",        ["sale"],              4.2),
        new(3, "Pro Subscription", 149m,  "Subscriptions",["popular"],          4.6),
        new(4, "F# Course",        249m,  "Courses",      ["new"],               4.3),
        new(5, "ASP.NET Book",     39m,   "Books",        ["sale", "popular"],   4.8),
        new(6, "Basic Subscription", 49m, "Subscriptions",["sale"],             4.1),
    ];

    public static List<Order> Orders() =>
    [
        new(1001, new(2024, 5, 1), "alice",
            [new(1, 1), new(2, 2)]),
        new(1002, new(2024, 5, 3), "bob",
            [new(3, 1), new(5, 3)]),
        new(1003, new(2024, 5, 4), "alice",
            [new(4, 1), new(2, 1)]),
    ];
}
```

```csharp
// File: Reports.cs
using System;
using System.Collections.Generic;
using System.Linq;

namespace M08L02Homework;

public static class Reports
{
    // EN: Expensive and popular — Where before Select: do not waste work projecting extras.
    // RU: Дорогие и популярные — Where раньше Select: не тратим работу на проекцию лишнего.
    public static List<(string Name, decimal Price, double Rating)> ExpensivePopular(
        IEnumerable<Product> products)
        => products
            .Where(p => p is { Price: > 100m, Rating: >= 4.5 })   // C# 12 pattern matching
            .Select(p => (p.Name, p.Price, p.Rating))             // tuple — returnable from method
            .ToList();                                            // materialize: freeze a snapshot

    // EN: Price list with tax — tuple is handier than anonymous type since the method returns it.
    // RU: Прайс с налогом — кортеж удобнее анонимного типа, т.к. метод его возвращает.
    public static List<(string Name, decimal Price, decimal Tax)> PriceListWithTax(
        IEnumerable<Product> products)
        => products
            .Select(p => (p.Name, p.Price, Tax: p.Price * 0.20m))
            .ToList();

    // EN: All catalogue tags — SelectMany flattens List<List<string>> into a flat stream.
    // RU: Все теги каталога — SelectMany сплющивает List<List<string>> в плоский поток.
    public static IReadOnlyCollection<string> AllTags(IEnumerable<Product> products)
        => products
            .SelectMany(p => p.Tags)
            .Distinct()
            .OrderBy(t => t)
            .ToList();

    // EN: Top deals — fluent style; OrderBy here is for output coherence.
    // RU: Топ скидок — fluent-стиль; OrderBy здесь для целостности вывода.
    public static List<(string Name, decimal Price)> TopDealsFluent(IEnumerable<Product> products, int n)
        => products
            .Where(p => p.Tags.Contains("sale"))
            .Select(p => (p.Name, p.Price))
            .OrderBy(x => x.Price)
            .Take(n)
            .ToList();

    // EN: Top deals — query syntax; compiles to the same chain as the fluent version.
    // RU: Топ скидок — query-синтаксис; компилируется в ту же цепочку, что и fluent-вариант.
    public static List<(string Name, decimal Price)> TopDealsQuery(IEnumerable<Product> products, int n)
        => (from p in products
            where p.Tags.Contains("sale")
            orderby p.Price
            select (p.Name, p.Price))
            .Take(n)
            .ToList();

    // EN: Sum of sold units per category — SelectMany over orders + join on ProductId.
    // RU: Сумма проданных единиц по категории — SelectMany по заказам + join по ProductId.
    public static int ItemsSoldByCategory(
        IEnumerable<Product> products,
        IEnumerable<Order> orders,
        string category)
        => (from o in orders
            from line in o.Lines
            join p in products on line.ProductId equals p.Id
            where p.Category == category
            select line.Qty).Sum();

    // EN: Deferred execution demo: a new product is visible in deferred, not in frozen.
    // RU: Демо отложенного исполнения: новый товар виден в отложенном, но не в замороженном.
    public static (List<string> Deferred, List<string> Frozen) DeferredExecutionDemo(List<Product> products)
    {
        var deferred = products.Where(p => p.Price < 100m).Select(p => p.Name);
        products.Add(new Product(99, "Temp",   50m, "Books", [], 3.0));
        var deferredSnapshot = deferred.ToList();           // runs NOW, sees the temp product

        var frozen = products.Where(p => p.Price < 100m).Select(p => p.Name).ToList();
        products.Add(new Product(98, "Temp2",  30m, "Books", [], 3.0));
        return (deferredSnapshot, frozen);                  // frozen does not contain the second temp
    }
}
```

Walk-through line by line. In `ExpensivePopular` we deliberately place `Where` before `Select`: the filter works on the "rich" `Product`, where `Price` and `Rating` are in scope. We also use the pattern match `is { Price: > 100m, Rating: >= 4.5 }` — a C# 12 idiom that expresses a compound condition compactly. The projection into a tuple `(p.Name, p.Price, p.Rating)` is chosen because the method must return the result outward and an anonymous type cannot appear in a signature — this is exactly the "common mistake" from the lesson. `ToList()` materializes a snapshot so later mutations of `products` do not "poison" the report.

In `AllTags` the key moment is `SelectMany(p => p.Tags)`. The lambda returns a collection (`List<string>`); a plain `Select` would have given `IEnumerable<List<string>>`. `SelectMany` flattens all tags into a single stream, after which `Distinct` keeps the unique ones. The order `SelectMany` → `Distinct` → `OrderBy` shows the canonical chain: flatten first, deduplicate next, sort for output last.

In `TopDealsFluent` and `TopDealsQuery` we show the equivalence of the two syntaxes. `Take(n)` is bolted onto the outside of the query expression because `orderby`/`where`/`select` are the body of the query, while `Take` is an operator over the result. This is a typical technique: query syntax describes the declarative part, fluent operators wrap it on the outside. Both methods compile to the same chain of extension calls.

In `ItemsSoldByCategory` we implement the classic "join" in query syntax: two consecutive `from` clauses (`from o in orders from line in o.Lines`) compile into a `SelectMany`, and `join p in products on line.ProductId equals p.Id` performs the keyed join. This fragment cannot be rewritten tersely in fluent style without the `Join` operator and a scratch anonymous type, so query syntax is justified here — exactly the lesson's best-practice recommendation.

In `DeferredExecutionDemo` the first half builds a deferred query but does not run it. Adding a product to `products` after building but before `ToList()` means that when `deferredSnapshot` is materialized, the temp product lands in the result — that is deferred execution. The second half materializes `frozen` with `ToList()` first, and only then is a second temp product added — `frozen` does not see it, because the snapshot was already taken. This method is a vivid illustration of the lesson's rule: "if you need a snapshot — `ToList()`".

In `Models.cs` we use collection expressions (`[ ... ]`) — a C# 12 feature for initializing `List<T>` and arrays. `record` for the models gives value equality, which is critical for `Distinct`/`Contains`/`GroupBy` to behave correctly. `Seed` is a static factory class returning fresh lists on each call — this guarantees that tests and demos operate on independent copies of the data and do not poison each other.

#### Going deeper (bonus)
1. Implement a report that returns the top-3 customers by total order value. You will need `SelectMany` + `Join` + `GroupBy` + `Sum` + `OrderByDescending` + `Take`. Decide where to materialize.
2. Rewrite `ItemsSoldByCategory` in fluent style with `SelectMany` + `Join`. Compare readability. Conclude when query syntax is genuinely better.
3. Add a `TagFrequency()` method returning a `Dictionary<string, int>` of tag frequency across the whole catalogue. Use `SelectMany` + `GroupBy` + `ToDictionary`.
4. Investigate performance: measure with `Stopwatch` the cost of re-enumerating a LINQ query over a large list (generate 100,000 products) — once over a live `IEnumerable`, once over a `ToList()`-frozen copy. Explain the gap via the concept of deferred execution.

---

#### Чек-лист сдачи / Submission checklist
- [ ] (RU) Проекты `M08L02Homework` и `M08L02Homework.Tests` созданы и собираются.
- [ ] (RU) В `Program.cs` используется top-level statements, в `.csproj` указан .NET 8.
- [ ] (RU) Все шесть методов-отчётов реализованы и протестированы.
- [ ] (RU) Демонстрация отложенного исполнения и безопасности мутаций присутствует.
- [ ] (RU) Каждый нетривиальный шаг снабжён комментарием `// RU: ... / EN: ...`.
- [ ] (RU) Тесты зелёные (`dotnet test`), сборка без предупреждений.
- [ ] (EN) Projects `M08L02Homework` and `M08L02Homework.Tests` are created and build.
- [ ] (EN) `Program.cs` uses top-level statements; `.csproj` targets .NET 8.
- [ ] (EN) All six report methods are implemented and tested.
- [ ] (EN) Deferred execution and mutation-safety demos are present.
- [ ] (EN) Every non-trivial step carries a `// RU: ... / EN: ...` comment.
- [ ] (EN) Tests are green (`dotnet test`); build is warning-free.

#### Ресурсы / Resources
- [Microsoft Learn — Standard query operators overview](https://learn.microsoft.com/dotnet/csharp/linq/standard-query-operators/)
- [Microsoft Learn — LINQ (C#)](https://learn.microsoft.com/dotnet/csharp/linq/)
- [Microsoft Learn — Anonymous types](https://learn.microsoft.com/dotnet/csharp/fundamentals/types/anonymous-types)
- [Microsoft Learn — LINQ query syntax](https://learn.microsoft.com/dotnet/csharp/language-reference/language-specification/expressions#query-expressions)
- [Microsoft Learn — Tuple types](https://learn.microsoft.com/dotnet/csharp/language-reference/builtin-types/reference-types#tuple-types)
- [Microsoft Learn — Collection expressions (C# 12)](https://learn.microsoft.com/dotnet/csharp/language-reference/operators/collection-expression)

---

[← К уроку M08-L02](lesson-M08-L02-where-select.md) | [⬆ К модулю M08](../README.md) | [Следующее ДЗ →](homework-M08-L03-orderby-thenby.md)
