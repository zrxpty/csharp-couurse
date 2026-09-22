---
[← К уроку M02-L07](lesson-M02-L07-string-stringbuilder.md) | [⬆ К модулю M02](../README.md) | [Следующее ДЗ →](homework-M02-L08-enum-tuples.md)
---

### Домашнее задание M02-L07: string основы, интерполяция, StringBuilder / Homework M02-L07: string basics, interpolation, StringBuilder

**Урок / Lesson:** M02-L07
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 3/5
**Цель / Goal:** (RU) Научиться выбирать правильный инструмент работы со строками под каждую подзадачу — интерполяцию для сборки из значений, `StringBuilder` для циклов, verbatim для путей, raw string literals для JSON, `IsNullOrEmpty`/`IsNullOrWhiteSpace` для проверок пустоты — и обосновывать выбор производительностью и читаемостью. (EN) Learn to pick the correct string tool for each sub-task — interpolation for value assembly, `StringBuilder` for loops, verbatim for paths, raw string literals for JSON, `IsNullOrEmpty`/`IsNullOrWhiteSpace` for emptiness checks — and justify each choice by performance and readability.

#### Связь с уроком / Connection to the lesson
(RU) ДЗ напрямую закрепляет все ключевые темы урока M02-L07: неизменяемость `string` и её следствия (почему `+=` в цикле — это квадратичная аллокация), `$`-интерполяцию со спецификаторами формата и ширины, verbatim-строки `@"..."` для путей Windows, raw string literals `"""..."""` для встраивания JSON без экранирования, проверки пустоты через `IsNullOrEmpty`/`IsNullOrWhiteSpace` и `StringBuilder` с правилом «5+ итераций → StringBuilder».
(EN) This homework directly reinforces every key topic of lesson M02-L07: `string` immutability and its consequences (why `+=` in a loop is quadratic allocation), `$`-interpolation with format and width specifiers, verbatim strings `@"..."` for Windows paths, raw string literals `"""..."""` for embedding JSON without escaping, emptiness checks via `IsNullOrEmpty`/`IsNullOrWhiteSpace`, and `StringBuilder` with the "5+ iterations → StringBuilder" rule.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация
Представьте, что вы джун-разработчик в команде, которая пишет бэкенд онлайн-магазина «Камень и Текст». Ваш тимлид поручил вам реализовать модуль формирования текстовых отчётов по заказам за сутки. Отчёт идёт двумя потребителями: во-первых, он выводится в консоль оператора склада, который читает его построчно и отгружает товар; во-вторых, он вкладывается в JSON-конверт и отправляется во внешнюю систему аналитики. Из-за этого один и тот же набор данных нужно уметь представить и как аккуратно выровненный моноширинный текст, и как валидный JSON без экранирования кавычек вручную.

Урок M02-L07 дал вам всё, что нужно для такой работы. Неизменяемость `string` объясняет, почему наивный `+=` в цикле по сотням заказов «убьёт» память и процессорное время: каждое сложение создаёт новую строку в куче и копирует оба содержимого, а старые строки повисают до сборки мусора. Интерполяция `$` позволяет вписывать форматы валюты и ширины колонок прямо в литерал, делая код самодокументируемым. Verbatim-строки `@"..."` избавляют от двойных слэшей в путях Windows и от шумного экранирования в регулярных выражениях. Raw string literals `"""..."""` дают возможность встроить многострочный JSON, XML или HTML без танцев с кавычками и `\`. Методы `string.IsNullOrEmpty` / `IsNullOrWhiteSpace` защищают от падения на «грязных» входных данных, где поле комментария может прийти пустым, состоять из одних пробелов или равняться `null`. Это задание проверяет, что вы не просто заучили синтаксис, а умеете выбрать правильный инструмент под каждую подзадачу и обосновать выбор ссылкой на конкретное свойство строк из урока.

#### Что нужно сделать (пошагово)
1. Создайте новый проект консольного приложения командой `dotnet new console -n OrderReportGenerator -o OrderReportGenerator` в каталоге модуля. Убедитесь, что в `OrderReportGenerator.csproj` указан `<TargetFramework>net8.0</TargetFramework>` и язык C# 12 (можно явно прописать `<LangVersion>12</LangVersion>`, хотя для .NET 8 это значение по умолчанию). Выполните `dotnet build` и убедитесь, что сборка проходит без ошибок и предупреждений.
2. Откройте `Program.cs`, удалите шаблонный `Console.WriteLine("Hello, World!");` и организуйте код в top-level statements. Модели данных можно объявить прямо в файле через `record` — например, `record Order(int Id, string Customer, decimal Total, DateTime At, string? Note);`. Использование `record` здесь принципиально: оно идиоматично для C# 12, даёт неизменяемую модель со структурным равенством и согласуется с философией неизменяемости из урока.
3. Задайте исходные данные: список из 6–8 заказов `List<Order>`, среди которых намеренно есть заказ с пустой `Note` (`""`), заказ с `Note`, состоящим из одних пробелов (`"   "`), и заказ с `Note == null`. Это нужно, чтобы продемонстрировать разницу между `IsNullOrEmpty` и `IsNullOrWhiteSpace` — как раз тот частый случай, который разбирается в разделе «Частые ошибки» урока.
4. Реализуйте метод `string BuildPlainTextReport(IEnumerable<Order> orders)`, который возвращает многострочный отчёт: заголовок с датой сборки, разделительную линию, шапку таблицы с колонками `Id`, `Customer`, `Total` (валютный формат `C`, ширина 10), `At` (формат `yyyy-MM-dd HH:mm`) и `Note`. Сборку строк таблицы делайте через `StringBuilder` с `Append`/`AppendLine`, потому что строк много и это классический случай «5+ итераций» из урока. Пустую или пробельную `Note` заменяйте на литерал `—`.
5. Реализуйте метод `string BuildJsonSummary(IEnumerable<Order> orders)`, который возвращает JSON-строку со сводкой: числовое поле `count`, `totalRevenue` (сумма по всем заказам), `topCustomer` (клиент с максимальной суммой — удобно через `MaxBy` из .NET 6+) и массив `orders` с полями `id`, `customer`, `total`, `note`. JSON собирайте через raw string literal `"""..."""` с правильным числом `$` для интерполяции — не экранируйте кавычки вручную и не используйте оператор `+`.
6. Реализуйте классификацию заказов по сумме через `switch` expression с pattern matching и интерполяцией: до 1000 — `$"Small: {o.Customer}"`, от 1000 до 9999.99 — `$"Regular: {o.Customer}"`, от 10000 и выше — `$"Wholesale: {o.Customer}"`. Выведите классификацию для каждого заказа отдельной строкой.
7. Определите путь к файлу сохранения отчёта через verbatim-строку: `@"C:\Temp\reports\daily.txt"`. В реальном проекте путь брался бы из конфигурации (и помните предупреждение урока: не хранить секреты в строковых литералах, потому что они интернируются и живут в памяти), но здесь мы тренируем именно синтаксис `@"..."` против обычного `"C:\\Temp\\reports\\daily.txt"`.
8. Выведите в консоль в следующем порядке: текстовый отчёт, пустую строку-разделитель, JSON-сводку, пустую строку, список классификаций, ещё одну пустую строку и в конце — путь сохранения. Запустите `dotnet run` и убедитесь, что вывод не содержит ошибок форматирования, а JSON валиден (вставьте его в любой онлайн-валидатор JSON).
9. Добавьте комментарии-обоснования к каждому нетривиальному выбору: почему здесь `StringBuilder`, а не `+=`; почему raw string с двумя `$`, а не одним; почему `IsNullOrWhiteSpace`, а не `== ""`; почему `@"..."`, а не экранированный литерал. Эти комментарии — часть приёмки.

#### Требования к решению
- Целевой платформой должен быть .NET 8, язык C# 12; top-level statements обязательны — никаких `class Program { static void Main }`.
- Используйте `record` для модели `Order` (не класс с полями) — это идиоматично для C# 12 и даёт структурное равенство, согласующееся с неизменяемостью строк из урока.
- В `BuildPlainTextReport` обязательно применение `StringBuilder` и методов `Append`/`AppendLine`; использовать конкатенацию `+=` для сборки строк таблицы запрещено. Окончательный `ToString()` вызывается ровно один раз в конце метода — не после каждого `Append`.
- В `BuildJsonSummary` обязательно применение raw string literal `"""..."""` с интерполяцией. Число `$` должно быть согласовано с числом фигурных скобок в шаблоне JSON: при наличии одиночных `{`/`}` (границы объекта) используйте `$$"""..."""` и удваивайте литеральные скобки `{{`/`}}`. Использование оператора `+` или ручного экранирования кавычек через `\"` запрещено.
- Путь к файлу задавайте через verbatim-строку `@"..."` — обычный литерал с `\\` запрещён, потому что именно от этого шума предостерегает урок.
- Все проверки на пустоту `Note` должны идти через `string.IsNullOrEmpty` или `string.IsNullOrWhiteSpace` — ни одного `== ""` или `.Length == 0` без guard на `null`, как требует best practice урока.
- Классификация сумм реализована через `switch` expression с pattern matching (`<`, `>= ... and < ...`, discard `_`), а не через каскад `if/else`.
- Форматирование валюты и даты должно задаваться спецификаторами прямо в интерполяции (`{total,12:C}`, `{at:yyyy-MM-dd HH:mm}`), а не через отдельные вызовы `.ToString(...)` с последующей склейкой.
- Код компилируется без предупреждений и проходит `dotnet run` без исключений на тестовых данных с пустыми, пробельными и `null` комментариями.

#### Тонкости и подводные камни
- **Число `$` в raw string literal — самая частая ошибка.** В JSON много одиночных `{` и `}` (скобки границ объекта). Если поставить один `$`, компилятор попытается интерполировать каждую пару скобок и сломается. Ставьте `$$"""..."""` и экранируйте литеральные скобки удвоением `{{`/`}}` — тогда интерполируются только двойные скобки. Проверьте, что в местах вставки значений у вас именно `{{o.Id}}`, а в местах, где скобки принадлежат самому JSON (например, границы объекта), — тоже удвоены. Это прямой аналог best practice из урока про raw strings.
- **`IsNullOrEmpty` vs `IsNullOrWhiteSpace` — почувствуйте разницу.** Для поля `Note`, состоящего из пробелов, `IsNullOrEmpty` вернёт `false` (длина ненулевая), и в отчёт попадёт «невидимая» строка. Если бизнес-логика считает пробельный комментарий эквивалентным отсутствию комментария — берите `IsNullOrWhiteSpace`. В этом ДЗ намеренно есть заказ с пробельной `Note`, чтобы вы не прошли мимо этой тонкости. Урок прямо предостерегает от `if (s == "")` и `if (s.Length == 0)` без guard на `null`.
- **StringBuilder — не панацея.** Не вызывайте `sb.ToString()` после каждого `Append`, «чтобы посмотреть промежуточный результат» — это создаёт новую строку каждый раз и обнуляет выигрыш от изменяемого буфера. Вызывайте один раз в конце метода и возвращайте результат. Также не оборачивайте в `StringBuilder` разовую сборку строки из 2–3 полей — там интерполяция быстрее и читаемее, как подчёркнуто в правиле урока.
- **`+=` в цикле — квадратичная сложность.** Если вы случайно оставите `reportLine += order...` внутри `foreach`, на 8 заказах вы этого не заметите, но на 8000 заказов память и CPU взлетят. Воспринимайте `+=` для строк в цикле как code smell уровня «никогда не делай так». Урок объясняет это через копирование содержимого обеих строк при каждом сложении.
- **Формат `C` зависит от культуры.** `dotnet run` под русской локалью Windows может вывести `1 234,56 ₽`, а под инвариантной — `$1,234.56`. Это нормально, но если нужен детерминированный вывод, передавайте `CultureInfo.InvariantCulture` через `string.Format(...)` или `FormattableString.Invariant`. В этом ДЗ достаточно дефолтной культуры, но знайте о подводном камне.
- **Verbatim не экранирует кавычки через `\`.** Внутри `@"..."` кавычка вставляется удвоением `""`, а не `\"`. Смешивание двух подходов даёт ошибку компиляции. Запомните: `@` отключает escape-последовательности, поэтому `\` трактуется буквально — это и плюс (пути), и источник путаницы.
- **Интернирование литералов и секреты.** Литералы вроде `"—"` или `""` интернируются и общие для всего процесса — это нормально и даже экономит память. Но из-за того же интернирования урок предостерегает: не храните в строковых литералах секреты (пароли, токены), выносите их в конфигурацию.
- **`string.Empty` vs `""`.** В местах, где вы явно присваиваете «пустую строку по смыслу», предпочтительнее `string.Empty` — он читается яснее и подчёркивает намерение, а не случайно набранный пустой литерал.

#### Критерии приёмки
- [ ] Проект создан через `dotnet new console`, `TargetFramework=net8.0`, компилируется без ошибок и предупреждений.
- [ ] Использованы top-level statements, нет `class Program`/`Main`.
- [ ] `Order` объявлен как `record`, не как класс с полями.
- [ ] В тестовых данных есть заказы с `null`, пустой и пробельной `Note`.
- [ ] `BuildPlainTextReport` собирает строки через `StringBuilder.Append`/`AppendLine`, без `+=`.
- [ ] `ToString()` у `StringBuilder` вызывается ровно один раз в конце метода.
- [ ] Колонка `Total` отформатирована как валюта с шириной через спецификатор в интерполяции/`string.Format`.
- [ ] Колонка `At` отформатирована как `yyyy-MM-dd HH:mm` прямо в формате.
- [ ] Пустая или пробельная `Note` заменена на `—`, проверка через `IsNullOrEmpty` или `IsNullOrWhiteSpace` (не `== ""`).
- [ ] `BuildJsonSummary` использует raw string literal `"""..."""` с правильным числом `$`.
- [ ] JSON-вывод валиден (открывается в JSON-валидаторе) и не содержит ручного экранирования кавычек.
- [ ] Классификация сумм реализована через `switch` expression с pattern matching и интерполяцией.
- [ ] Путь к файлу задан через `@"..."` verbatim-строку, без `\\`.
- [ ] Вывод `dotnet run` содержит: текстовый отчёт, разделитель, JSON, классификации, путь — без ошибок.
- [ ] В коде есть комментарии-обоснования выбора StringBuilder / raw / IsNullOrWhiteSpace.

#### Подсказки (без прямого ответа)
- Вспомните метафору урока про «каменные таблички»: каждая `+=` — это новая табличка, выбитая заново. Где в вашем коде таблички плодятся быстрее всего?
- Для JSON со многими одиночными `{`/`}` начните с `$$"""..."""` и удваивайте скобки там, где они должны остаться литералом.
- `string.IsNullOrWhiteSpace(o.Note) ? "—" : o.Note` — компактный способ обработки пустого комментария.
- Чтобы найти «топ-клиента», используйте LINQ `OrderByDescending(o => o.Total).First().Customer` или более лаконичный `MaxBy(o => o.Total)` из .NET 6+.
- Ширина колонки `{value,10}` выравнивает по правому краю; для левого выравнивания используйте отрицательную ширину `{value,-10}`.
- Помните, что `sb.Append("- ").AppendLine(line)` можно вызывать цепочкой — это эффективнее и читаемее, чем два отдельных оператора.

#### Эталонное решение (разбор)

```csharp
// C# 12 / .NET 8 — top-level statements / top-level statements
using System.Globalization;
using System.Text;

// Модель данных — record, неизменяемая, со структурным равенством
// Data model — record, immutable, with structural equality
record Order(int Id, string Customer, decimal Total, DateTime At, string? Note);

// Тестовые данные с намеренно «грязными» Note: "", null, "   "
// Test data with intentionally dirty Notes: "", null, "   "
List<Order> orders =
[
    new(1, "Anna Petrova", 12500.00m, new DateTime(2024, 5, 1, 9, 15, 0),  "Доставка лифтом / elevator delivery"),
    new(2, "Ivan Sidorov",   450.50m, new DateTime(2024, 5, 1, 10, 5, 0),  ""),
    new(3, "Elena K.",      3200.75m, new DateTime(2024, 5, 1, 11, 0, 0),  null),
    new(4, "Pavel M.",      9800.00m, new DateTime(2024, 5, 1, 12, 30, 0), "   "),
    new(5, "Olga N.",      15000.00m, new DateTime(2024, 5, 1, 13, 45, 0), "Опт / wholesale"),
    new(6, "Dmitry O.",      750.00m, new DateTime(2024, 5, 1, 14, 10, 0), "Самовывоз / pickup"),
];

// 1) Текстовый отчёт через StringBuilder — цикл по строкам, нужен изменяемый буфер
// Plain-text report via StringBuilder — loop over rows, need a mutable buffer
string BuildPlainTextReport(IEnumerable<Order> rows)
{
    var sb = new StringBuilder();
    sb.AppendLine($"Отчёт по заказам / Orders report — {DateTime.Now:yyyy-MM-dd HH:mm}");
    sb.AppendLine(new string('-', 72));
    sb.AppendLine(string.Format(CultureInfo.InvariantCulture,
        "{0,-4} {1,-18} {2,12} {3,-17} {4}", "ID", "Customer", "Total", "At", "Note"));
    sb.AppendLine(new string('-', 72));

    foreach (var o in rows)
    {
        // IsNullOrWhiteSpace ловит и null, и "", и "   " — best practice из урока
        // IsNullOrWhiteSpace catches null, "", and "   " — lesson best practice
        string note = string.IsNullOrWhiteSpace(o.Note) ? "—" : o.Note;
        sb.Append(string.Format(CultureInfo.InvariantCulture, "{0,-4} ", o.Id));
        sb.Append(string.Format(CultureInfo.InvariantCulture, "{0,-18} ", o.Customer));
        sb.Append(string.Format(CultureInfo.InvariantCulture, "{0,12:C} ", o.Total));
        sb.Append(string.Format(CultureInfo.InvariantCulture, "{0:yyyy-MM-dd HH:mm} ", o.At));
        sb.AppendLine(note);
    }

    sb.AppendLine(new string('-', 72));
    return sb.ToString();   // единственный финальный аллок / single final allocation
}

// 2) JSON-сводка через raw string literal — никакого ручного экранирования кавычек
// JSON summary via raw string literal — no manual quote escaping
string BuildJsonSummary(IEnumerable<Order> rows)
{
    var list = rows.ToList();
    int count = list.Count;
    decimal totalRevenue = list.Sum(o => o.Total);
    string topCustomer = list.MaxBy(o => o.Total)!.Customer;

    // Массив элементов собираем через StringBuilder, каждый элемент — через интерполяцию
    // Build the orders array via StringBuilder; each element is interpolated
    var sb = new StringBuilder();
    for (int i = 0; i < list.Count; i++)
    {
        var o = list[i];
        string note = string.IsNullOrWhiteSpace(o.Note) ? "" : o.Note;
        string comma = i < list.Count - 1 ? "," : "";
        // Два $: литеральные { } JSON удваиваем, интерполируем {{ }}
        // Two $: double literal JSON braces, interpolate {{ }}
        sb.Append($$"""    { "id": {{o.Id}}, "customer": "{{o.Customer}}", "total": {{o.Total}}, "note": "{{note}}" }{{comma}}
""");
    }
    string ordersJson = sb.ToString();

    // Сырая строка с двумя $: одиночные { } удваиваем, интерполируем двойные {{ }}
    // Raw string with two $: double single braces, interpolate double braces
    return $$"""
{
  "count": {{count}},
  "totalRevenue": {{totalRevenue}},
  "topCustomer": "{{topCustomer}}",
  "orders": [
{{ordersJson}}  ]
}
""";
}

// 3) Классификация через switch expression + pattern matching + интерполяцию
// Classification via switch expression + pattern matching + interpolation
string Classify(Order o) => o.Total switch
{
    < 1000m            => $"Small: {o.Customer}",
    >= 1000m and < 10000m => $"Regular: {o.Customer}",
    _                  => $"Wholesale: {o.Customer}"
};

// 4) Путь к файлу через verbatim — без двойных слэшей
// File path via verbatim — no doubled backslashes
string reportPath = @"C:\Temp\reports\daily.txt";

// Вывод / Output
Console.WriteLine(BuildPlainTextReport(orders));
Console.WriteLine(BuildJsonSummary(orders));
Console.WriteLine();
foreach (var o in orders)
{
    Console.WriteLine(Classify(o));
}
Console.WriteLine();
Console.WriteLine($"Saved to: {reportPath}");
```

Разбор по строкам. Модель `record Order` даёт неизменяемую модель данных с автоматической реализацией равенства и `ToString()` — это идиоматично для C# 12 и отражает философию неизменяемости из урока: данные заказа не «мутируют» после создания, как и сами строки. Список `orders` задан через коллекционное выражение `[...]` (новинка C# 12), а среди `Note` намеренно присутствуют `""`, `null` и `"   "` — это три разных «плохих» случая, которые проверяют, выбрали ли вы правильный метод проверки пустоты.

`BuildPlainTextReport` использует `StringBuilder`, потому что количество строк в таблице растёт с числом заказов — это классический случай «5+ итераций» из урока, где `+=` дал бы квадратичную аллокацию. Каждый `Append`/`AppendLine` пишет в изменяемый внутренний буфер за амортизированное O(1), а единственный `ToString()` в конце делает один финальный аллок. Если бы вы написали `report += line` в `foreach`, на каждой итерации создавалась бы новая строка длиной `len(report)+len(line)` с копированием обоих содержимых — именно то, от чего предостерегает урок. Ширины колонок и форматы (`{0,12:C}`, `{0:yyyy-MM-dd HH:mm}`) заданы прямо в `string.Format`, что функционально эквивалентно интерполяции `$"{o.Total,12:C}"` и отражает best practice урока про форматирование в скобках. Проверка `string.IsNullOrWhiteSpace(o.Note)` ловит все три «плохих» случая, в отличие от `== ""` или `IsNullOrEmpty`, которые пропустили бы пробельную `Note` и вывели бы в таблицу невидимую строку.

`BuildJsonSummary` использует raw string literal `$$"""..."""`. Число `$` равно двум, потому что в JSON много одиночных `{`/`}` (границы объекта), и мы хотим интерполировать только двойные скобки `{{ }}`. Это напрямую применяет best practice урока: не экранировать кавычки вручную, а выбрать правильное число `$`. Внутренний цикл собирает массив элементов тоже через `StringBuilder`, а каждый элемент — через интерполяцию внутри raw literal с удвоенными скобками в местах, где нужны реальные `{`/`}` JSON. `MaxBy` из .NET 6+ находит топ-клиента без ручной сортировки. `Classify` — это `switch` expression с pattern matching и интерполяцией: компактно, самодокументируемо, без каскада `if/else`. Путь `@"C:\Temp\reports\daily.txt"` — verbatim, без двойных слэшей, как требует best practice урока для путей Windows. Итог: каждый выбор обоснован конкретной темой урока — неизменяемость, интерполяция, verbatim, raw strings, проверки пустоты, StringBuilder, pattern matching.

#### Задания на углубление (бонус)
1. Сравните производительность `+=` и `StringBuilder` на 50 000 итераций: измерьте время через `Stopwatch` и число аллокаций (через `GC.GetAllocatedBytesForCurrentThread()`). Выведите оба числа и объясните разницу O(n) vs O(n²) своими словами, опираясь на объяснение копирования содержимого строк из урока.
2. Добавьте поддержку культуры отчёта: параметризуйте `BuildPlainTextReport` через `CultureInfo` и проверьте, что валюта и дата меняются при `ru-RU` и `en-US`. Используйте `FormattableString.Invariant` для JSON, чтобы числа в JSON всегда были с точкой и не ломали парсер аналитики.
3. Перепишите `BuildJsonSummary` через `System.Text.Json` (`JsonSerializer.Serialize`) и сравните читаемость, производительность и подверженность ошибкам с raw string literal. Объясните, когда каждый подход уместен (быстрый прототип vs продакшен с произвольными данными).
4. Реализуйте метод `BuildCsvReport`, который экранирует запятые и кавычки в `Customer`/`Note` по правилам RFC 4180 (удвоение кавычек, оборачивание поля в кавычки при наличии спецсимволов). Используйте `StringBuilder` и `string.Replace`. Подумайте, почему здесь verbatim не помогает, а raw string literal — тем более.

---

## Statement in English / Постановка на английском

#### Context & motivation
Imagine you are a junior developer on the team building the back end of the "Stone and Text" online store. Your tech lead has asked you to implement a module that produces daily textual order reports. The report has two consumers: first, it is printed to the warehouse operator's console, who reads it line by line and ships goods; second, it is wrapped in a JSON envelope and pushed to an external analytics system. Because of this, the same dataset must be rendered both as a neatly aligned monospaced text table and as valid JSON without hand-escaped quotes.

Lesson M02-L07 gave you everything required for such work. The immutability of `string` explains why a naive `+=` inside a loop over hundreds of orders will kill memory and CPU: every addition creates a brand-new string on the heap and copies both contents, while the old strings linger until garbage collection. `$` interpolation lets you embed currency formats and column widths directly in the literal, making the code self-documenting. Verbatim strings `@"..."` remove the need for doubled backslashes in Windows paths and for noisy escaping in regular expressions. Raw string literals `"""..."""` let you embed multi-line JSON, XML or HTML without juggling quotes and backslashes. The methods `string.IsNullOrEmpty` / `IsNullOrWhiteSpace` guard against crashes on dirty input where a comment field may arrive empty, whitespace-only, or `null`. This homework verifies that you have not merely memorized the syntax but can pick the right tool for each sub-task and justify the choice by referring to a specific string property from the lesson.

#### What to do step by step
1. Create a new console application with `dotnet new console -n OrderReportGenerator -o OrderReportGenerator` inside the module folder. Confirm `OrderReportGenerator.csproj` targets `<TargetFramework>net8.0</TargetFramework>` and uses C# 12 (you may set `<LangVersion>12</LangVersion>` explicitly, although it is the default for .NET 8). Run `dotnet build` and make sure it succeeds with no errors or warnings.
2. Open `Program.cs`, delete the boilerplate `Console.WriteLine("Hello, World!");`, and organize the code as top-level statements. You may declare data models directly in the file using `record` — for example, `record Order(int Id, string Customer, decimal Total, DateTime At, string? Note);`. Using a `record` here is deliberate: it is idiomatic for C# 12, gives an immutable model with structural equality, and aligns with the lesson's philosophy of immutability.
3. Provide source data: a `List<Order>` of 6–8 orders, deliberately including one with an empty `Note` (`""`), one with a whitespace-only `Note` (`"   "`), and one with `Note == null`. This exercises the difference between `IsNullOrEmpty` and `IsNullOrWhiteSpace` — exactly the common mistake discussed in the lesson's "Common Mistakes" section.
4. Implement `string BuildPlainTextReport(IEnumerable<Order> orders)` returning a multi-line report: a header with the build date, a separator line, a table header with columns `Id`, `Customer`, `Total` (currency format `C`, width 10), `At` (format `yyyy-MM-dd HH:mm`), and `Note`. Build table rows with `StringBuilder` via `Append`/`AppendLine`, because there are many rows and this is the textbook "5+ iterations" case from the lesson. Replace an empty or whitespace `Note` with the literal `—`.
5. Implement `string BuildJsonSummary(IEnumerable<Order> orders)` returning a JSON string with a summary: a numeric `count` field, `totalRevenue` (the sum across all orders), `topCustomer` (the customer with the largest total — conveniently via `MaxBy` from .NET 6+), and an `orders` array with `id`, `customer`, `total`, `note`. Build the JSON with a raw string literal `"""..."""` using the correct number of `$` for interpolation — do not escape quotes manually and do not use the `+` operator.
6. Implement order classification by total using a `switch` expression with pattern matching and interpolation: below 1000 → `$"Small: {o.Customer}"`, 1000–9999.99 → `$"Regular: {o.Customer}"`, 10000 and above → `$"Wholesale: {o.Customer}"`. Print the classification for every order on a separate line.
7. Define the report file path using a verbatim string: `@"C:\Temp\reports\daily.txt"`. In a real project the path would come from configuration (and remember the lesson's warning: do not store secrets in string literals, because they are interned and persist in memory), but here we are training the `@"..."` syntax against the plain `"C:\\Temp\\reports\\daily.txt"`.
8. Print to the console in this order: the plain-text report, a blank separator line, the JSON summary, another blank line, the list of classifications, another blank line, and finally the save path. Run `dotnet run` and verify the output has no formatting errors and the JSON is valid (paste it into any online JSON validator).
9. Add justification comments to every non-trivial choice: why `StringBuilder` here and not `+=`; why a raw string with two `$` and not one; why `IsNullOrWhiteSpace` and not `== ""`; why `@"..."` and not an escaped literal. These comments are part of acceptance.

#### Requirements
- Target .NET 8 and C# 12; top-level statements are mandatory — no `class Program { static void Main }`.
- Use a `record` for the `Order` model (not a class with fields) — this is idiomatic C# 12 and gives structural equality, consistent with the lesson's immutability theme.
- In `BuildPlainTextReport` you must use `StringBuilder` and `Append`/`AppendLine`; building table rows with `+=` is forbidden. The final `ToString()` is called exactly once at the end of the method — not after every `Append`.
- In `BuildJsonSummary` you must use a raw string literal `"""..."""` with interpolation. The count of `$` must match the brace count in the JSON template: when single `{`/`}` are present (object boundaries) use `$$"""..."""` and double literal braces `{{`/`}}`. Using the `+` operator or manual quote escaping with `\"` is forbidden.
- The file path must be a verbatim string `@"..."` — a plain literal with `\\` is forbidden, because this is exactly the noise the lesson warns against.
- All emptiness checks on `Note` must go through `string.IsNullOrEmpty` or `string.IsNullOrWhiteSpace` — no `== ""` or `.Length == 0` without a null guard, as the lesson's best practice requires.
- Total classification must use a `switch` expression with pattern matching (`<`, `>= ... and < ...`, discard `_`), not an `if/else` cascade.
- Currency and date formatting must be specified inline in the interpolation (`{total,12:C}`, `{at:yyyy-MM-dd HH:mm}`), not through separate `.ToString(...)` calls followed by concatenation.
- The code compiles without warnings and `dotnet run` succeeds without exceptions on test data with empty, whitespace, and `null` comments.

#### Pitfalls
- **The count of `$` in a raw string literal — the most common mistake.** JSON has many single `{` and `}` (object boundaries). With a single `$` the compiler will try to interpolate every brace pair and fail. Use `$$"""..."""` and double literal braces `{{`/`}}` so only double braces are interpolated. Verify that value insertion sites use `{{o.Id}}`, and braces belonging to the JSON itself are also doubled. This is a direct application of the lesson's raw-strings best practice.
- **`IsNullOrEmpty` vs `IsNullOrWhiteSpace` — feel the difference.** For a whitespace-only `Note`, `IsNullOrEmpty` returns `false` (length is non-zero), and an "invisible" string reaches the report. If business logic treats a whitespace comment as equivalent to no comment, use `IsNullOrWhiteSpace`. This homework deliberately includes a whitespace `Note` so you cannot skip this subtlety. The lesson explicitly warns against `if (s == "")` and `if (s.Length == 0)` without a null guard.
- **StringBuilder is not a panacea.** Do not call `sb.ToString()` after every `Append` "to peek" — each call allocates a new string and negates the benefit of the mutable buffer. Call it once at the end of the method and return the result. Also, do not wrap a one-off assembly of two or three fields in `StringBuilder` — interpolation is faster and more readable there, as the lesson's rule of thumb emphasizes.
- **`+=` in a loop is quadratic.** If you accidentally leave `reportLine += order...` inside `foreach`, you will not notice it on 8 orders, but on 8000 memory and CPU will spike. Treat `+=` for strings in a loop as a "never do this" code smell. The lesson explains this through copying both strings' contents on every addition.
- **The `C` format depends on culture.** `dotnet run` under a Russian Windows locale may print `1 234,56 ₽`, while under the invariant culture it prints `$1,234.56`. That is fine, but for deterministic output pass `CultureInfo.InvariantCulture` via `string.Format(...)` or `FormattableString.Invariant`. This homework accepts the default culture, but be aware of the pitfall.
- **Verbatim does not escape quotes with `\`.** Inside `@"..."` a quote is inserted by doubling `""`, not `\"`. Mixing the two produces a compile error. Remember: `@` disables escape sequences, so `\` is treated literally — this is both a benefit (paths) and a source of confusion.
- **Literal interning and secrets.** Literals like `"—"` or `""` are interned and shared across the process — that is normal and even saves memory. But because of the same interning, the lesson warns: do not keep secrets (passwords, tokens) in string literals — move them to configuration.
- **`string.Empty` vs `""`.** Where you intentionally assign "an empty string by meaning," prefer `string.Empty` — it reads clearer and signals intent, rather than an accidentally typed empty literal.

#### Acceptance criteria
- [ ] Project created via `dotnet new console`, `TargetFramework=net8.0`, compiles without errors or warnings.
- [ ] Top-level statements used, no `class Program`/`Main`.
- [ ] `Order` declared as a `record`, not a class with fields.
- [ ] Test data includes orders with `null`, empty, and whitespace `Note`.
- [ ] `BuildPlainTextReport` builds rows via `StringBuilder.Append`/`AppendLine`, without `+=`.
- [ ] `StringBuilder.ToString()` is called exactly once at the end of the method.
- [ ] The `Total` column is formatted as currency with a width via an interpolation/`string.Format` specifier.
- [ ] The `At` column is formatted as `yyyy-MM-dd HH:mm` inline in the format string.
- [ ] Empty or whitespace `Note` is replaced with `—`, checked via `IsNullOrEmpty` or `IsNullOrWhiteSpace` (not `== ""`).
- [ ] `BuildJsonSummary` uses a raw string literal `"""..."""` with the correct count of `$`.
- [ ] The JSON output is valid (parses in a JSON validator) and contains no manual quote escaping.
- [ ] Total classification is implemented with a `switch` expression, pattern matching, and interpolation.
- [ ] The file path uses a verbatim string `@"..."`, with no `\\`.
- [ ] `dotnet run` output contains: plain-text report, separator, JSON, classifications, path — with no errors.
- [ ] Code contains justification comments for the choice of StringBuilder / raw / IsNullOrWhiteSpace.

#### Hints (no direct answer)
- Recall the lesson's "stone tablet" metaphor: every `+=` is a new tablet, chiseled from scratch. Where in your code do tablets multiply fastest?
- For JSON with many single `{`/`}`, start with `$$"""..."""` and double braces where they must remain literal.
- `string.IsNullOrWhiteSpace(o.Note) ? "—" : o.Note` is a compact way to handle an empty comment.
- To find the top customer, use LINQ `OrderByDescending(o => o.Total).First().Customer` or the more concise `MaxBy(o => o.Total)` from .NET 6+.
- Column width `{value,10}` right-aligns; for left alignment use a negative width `{value,-10}`.
- Remember that `sb.Append("- ").AppendLine(line)` can be chained — it is more efficient and readable than two separate statements.

#### Reference solution walk-through

```csharp
// C# 12 / .NET 8 — top-level statements
using System.Globalization;
using System.Text;

// Data model — record, immutable, with structural equality
record Order(int Id, string Customer, decimal Total, DateTime At, string? Note);

// Test data with intentionally dirty Notes: "", null, "   "
List<Order> orders =
[
    new(1, "Anna Petrova", 12500.00m, new DateTime(2024, 5, 1, 9, 15, 0),  "Elevator delivery"),
    new(2, "Ivan Sidorov",   450.50m, new DateTime(2024, 5, 1, 10, 5, 0),  ""),
    new(3, "Elena K.",      3200.75m, new DateTime(2024, 5, 1, 11, 0, 0),  null),
    new(4, "Pavel M.",      9800.00m, new DateTime(2024, 5, 1, 12, 30, 0), "   "),
    new(5, "Olga N.",      15000.00m, new DateTime(2024, 5, 1, 13, 45, 0), "Wholesale"),
    new(6, "Dmitry O.",      750.00m, new DateTime(2024, 5, 1, 14, 10, 0), "Pickup"),
];

// 1) Plain-text report via StringBuilder — loop over rows, need a mutable buffer
string BuildPlainTextReport(IEnumerable<Order> rows)
{
    var sb = new StringBuilder();
    sb.AppendLine($"Orders report — {DateTime.Now:yyyy-MM-dd HH:mm}");
    sb.AppendLine(new string('-', 72));
    sb.AppendLine(string.Format(CultureInfo.InvariantCulture,
        "{0,-4} {1,-18} {2,12} {3,-17} {4}", "ID", "Customer", "Total", "At", "Note"));
    sb.AppendLine(new string('-', 72));

    foreach (var o in rows)
    {
        // IsNullOrWhiteSpace catches null, "", and "   " — lesson best practice
        string note = string.IsNullOrWhiteSpace(o.Note) ? "—" : o.Note;
        sb.Append(string.Format(CultureInfo.InvariantCulture, "{0,-4} ", o.Id));
        sb.Append(string.Format(CultureInfo.InvariantCulture, "{0,-18} ", o.Customer));
        sb.Append(string.Format(CultureInfo.InvariantCulture, "{0,12:C} ", o.Total));
        sb.Append(string.Format(CultureInfo.InvariantCulture, "{0:yyyy-MM-dd HH:mm} ", o.At));
        sb.AppendLine(note);
    }

    sb.AppendLine(new string('-', 72));
    return sb.ToString();   // single final allocation
}

// 2) JSON summary via raw string literal — no manual quote escaping
string BuildJsonSummary(IEnumerable<Order> rows)
{
    var list = rows.ToList();
    int count = list.Count;
    decimal totalRevenue = list.Sum(o => o.Total);
    string topCustomer = list.MaxBy(o => o.Total)!.Customer;

    // Build the orders array via StringBuilder; each element is interpolated
    var sb = new StringBuilder();
    for (int i = 0; i < list.Count; i++)
    {
        var o = list[i];
        string note = string.IsNullOrWhiteSpace(o.Note) ? "" : o.Note;
        string comma = i < list.Count - 1 ? "," : "";
        // Two $: double literal JSON braces, interpolate {{ }}
        sb.Append($$"""    { "id": {{o.Id}}, "customer": "{{o.Customer}}", "total": {{o.Total}}, "note": "{{note}}" }{{comma}}
""");
    }
    string ordersJson = sb.ToString();

    // Raw string with two $: double single braces, interpolate double braces
    return $$"""
{
  "count": {{count}},
  "totalRevenue": {{totalRevenue}},
  "topCustomer": "{{topCustomer}}",
  "orders": [
{{ordersJson}}  ]
}
""";
}

// 3) Classification via switch expression + pattern matching + interpolation
string Classify(Order o) => o.Total switch
{
    < 1000m               => $"Small: {o.Customer}",
    >= 1000m and < 10000m => $"Regular: {o.Customer}",
    _                     => $"Wholesale: {o.Customer}"
};

// 4) File path via verbatim — no doubled backslashes
string reportPath = @"C:\Temp\reports\daily.txt";

// Output
Console.WriteLine(BuildPlainTextReport(orders));
Console.WriteLine(BuildJsonSummary(orders));
Console.WriteLine();
foreach (var o in orders)
{
    Console.WriteLine(Classify(o));
}
Console.WriteLine();
Console.WriteLine($"Saved to: {reportPath}");
```

Walk-through. The `record Order` model gives immutable data with automatic equality and `ToString()` — idiomatic for C# 12 and aligned with the lesson's philosophy of immutability: order data does not mutate after creation, just like strings themselves. The `orders` list uses a collection expression `[...]` (new in C# 12), and among the `Note` values there are deliberately `""`, `null`, and `"   "` — three distinct "bad" cases that test whether you chose the correct emptiness check.

`BuildPlainTextReport` uses `StringBuilder` because the number of rows grows with the order count — the textbook "5+ iterations" case from the lesson where `+=` would yield quadratic allocation. Each `Append`/`AppendLine` writes into the mutable internal buffer at amortized O(1), and the single `ToString()` at the end makes one final allocation. If you had written `report += line` in the `foreach`, every iteration would create a new string of length `len(report)+len(line)` and copy both contents — exactly what the lesson warns against. Column widths and formats (`{0,12:C}`, `{0:yyyy-MM-dd HH:mm}`) are specified directly in `string.Format`, functionally equivalent to `$"{o.Total,12:C}"` and reflecting the lesson's best practice of in-brace formatting. The `string.IsNullOrWhiteSpace(o.Note)` check catches all three "bad" cases, unlike `== ""` or `IsNullOrEmpty`, which would pass the whitespace `Note` through and print an invisible line into the table.

`BuildJsonSummary` uses a raw string literal `$$"""..."""`. The count of `$` is two because JSON has many single `{`/`}` (object boundaries), and we want to interpolate only double braces `{{ }}`. This directly applies the lesson's best practice: do not escape quotes by hand — pick the correct count of `$`. The inner loop also assembles the element array through `StringBuilder`, while each element is interpolated inside a raw literal with doubled braces where real JSON `{`/`}` are needed. `MaxBy` from .NET 6+ finds the top customer without manual sorting. `Classify` is a `switch` expression with pattern matching and interpolation: compact, self-documenting, with no `if/else` cascade. The path `@"C:\Temp\reports\daily.txt"` is verbatim, with no doubled backslashes, as the lesson's best practice for Windows paths requires. The bottom line: every choice maps to a specific lesson topic — immutability, interpolation, verbatim, raw strings, emptiness checks, StringBuilder, pattern matching.

#### Going deeper (bonus)
1. Benchmark `+=` versus `StringBuilder` over 50 000 iterations: measure time with `Stopwatch` and allocation bytes with `GC.GetAllocatedBytesForCurrentThread()`. Print both numbers and explain the O(n) vs O(n²) difference in your own words, leaning on the lesson's explanation of copying both strings' contents.
2. Add culture support: parameterize `BuildPlainTextReport` with `CultureInfo` and verify that currency and dates change under `ru-RU` and `en-US`. Use `FormattableString.Invariant` for the JSON so numbers always use a decimal point and do not break the analytics parser.
3. Rewrite `BuildJsonSummary` with `System.Text.Json` (`JsonSerializer.Serialize`) and compare readability, performance, and error-proneness against the raw string literal. Explain when each approach is appropriate (quick prototype vs production with arbitrary data).
4. Implement `BuildCsvReport` that escapes commas and quotes in `Customer`/`Note` per RFC 4180 (doubling quotes, wrapping a field in quotes when special characters are present). Use `StringBuilder` and `string.Replace`. Think about why verbatim does not help here, and a raw string literal even less so.

---

#### Чек-лист сдачи / Submission checklist
- [ ] Проект `OrderReportGenerator` создан и собирается под .NET 8 / C# 12.
- [ ] Использованы top-level statements и `record Order`.
- [ ] `BuildPlainTextReport` использует `StringBuilder`, `ToString()` один раз.
- [ ] `BuildJsonSummary` использует raw string literal с правильным числом `$`.
- [ ] Все проверки пустоты — через `IsNullOrEmpty`/`IsNullOrWhiteSpace`.
- [ ] Классификация — через `switch` expression с pattern matching.
- [ ] Путь задан через verbatim `@"..."`.
- [ ] `dotnet run` выводит отчёт, JSON, классификации и путь без ошибок.
- [ ] Добавлены комментарии-обоснования ключевых решений.
- [ ] Project `OrderReportGenerator` created and builds under .NET 8 / C# 12.
- [ ] Top-level statements and `record Order` used.
- [ ] `BuildPlainTextReport` uses `StringBuilder`, `ToString()` once.
- [ ] `BuildJsonSummary` uses a raw string literal with the correct count of `$`.
- [ ] All emptiness checks go through `IsNullOrEmpty`/`IsNullOrWhiteSpace`.
- [ ] Classification uses a `switch` expression with pattern matching.
- [ ] Path uses verbatim `@"..."`.
- [ ] `dotnet run` prints the report, JSON, classifications, and path with no errors.
- [ ] Justification comments added for key decisions.

#### Ресурсы / Resources
- Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/tour-of-csharp/program-building-blocks#strings — Строки (C#): основы / Strings (C#): basics
- Microsoft Learn — https://learn.microsoft.com/dotnet/api/system.text.stringbuilder — Класс StringBuilder: изменяемые строки / StringBuilder class: mutable strings
- Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/language-reference/tokens/raw-string — Raw string literals (C# 11)
- Microsoft Learn — https://learn.microsoft.com/dotnet/api/system.string.isnullorwhitespace — `String.IsNullOrWhiteSpace`
- Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/language-reference/operators/patterns — Pattern matching (C#)

---
[← К уроку M02-L07](lesson-M02-L07-string-stringbuilder.md) | [⬆ К модулю M02](../README.md) | [Следующее ДЗ →](homework-M02-L08-enum-tuples.md)
---
