---
[← К уроку M03-L01](lesson-M03-L01-if-ternary.md) | [⬆ К модулю M03](../README.md) | [Следующее ДЗ →](homework-M03-L02-switch-patterns.md)
---

### Домашнее задание M03-L01: if/else, тернарный оператор / Homework M03-L01: if/else, ternary operator

**Урок / Lesson:** M03-L01
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 3/5
**Цель / Goal:** (RU) Научиться применять `if/else if/else`, guard clauses (ранний возврат), короткое замыкание `&&`/`||`, тернарный оператор `?:` как выражение и pattern matching `is { ... }` для написания читаемого, безопасного и плоского условного кода на C# 12 / .NET 8. (EN) Learn to apply `if/else if/else`, guard clauses (early return), short-circuit `&&`/`||`, the ternary operator `?:` as an expression, and `is { ... }` pattern matching to write readable, safe, flat conditional code in C# 12 / .NET 8.

#### Связь с уроком / Connection to the lesson
(RU) ДЗ напрямую закрепляет ключевые концепции урока: guard clauses вместо вложенных `if`, тернарный оператор как выражение для выбора значения, короткое замыкание логических операторов для null-безопасности, pattern matching `is { ... }` вместо длинных цепочек проверок и правило «всегда фигурные скобки». Также отрабатываются частые ошибки: побитовое `&` вместо `&&`, висячий `else`, вложенный тернар и присваивание вместо сравнения.
(EN) The homework directly reinforces the lesson's key concepts: guard clauses instead of nested `if`, the ternary operator as a value-returning expression, short-circuit evaluation for null safety, `is { ... }` pattern matching instead of long check chains, and the "always braces" rule. It also drills common mistakes: bitwise `&` instead of `&&`, dangling `else`, nested ternary, and assignment instead of comparison.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация
Представьте, что вы junior-разработчик в команде интернет-магазина «Северный Двор». Команда пишет модуль расчёта стоимости доставки заказа. Сейчас этот модуль реализован стажёром в виде «лесенки» из вложенных `if` глубиной в четыре-пять уровней, с побитовым `&` вместо `&&`, без фигурных скобок на однострочных ветках и с вложенным тернарным оператором вида `a ? b ? c : d : e`. Код работает, но на код-ревью его называют «спагетти из условий»: тяжело читать, легко сломать, в нём дважды всплывал `NullReferenceException` из-за того, что правый операнд `&` вычислялся даже при `null`.

Ваша задача — переписать этот модуль с нуля, опираясь на лучшие практики из урока M03-L01. Вы должны продемонстрировать, что умеете: отсекать невалидные входы через guard clauses (ранний возврат), использовать взаимоисключающие ветки `if/else if/else` там, где это семантически верно, применять тернарный оператор только для выбора значения в одну строку, использовать короткое замыкание `&&`/`||` для null-безопасных проверок и заменять длинные цепочки `!= null && prop` на pattern matching `is { ... }`. Это реалистичная задача, которая каждый день встречается в production-коде: бизнес-логика с правилами скидок, порогами бесплатной доставки и надбавками за экспресс.

Цель — не просто «сделать, чтобы работало», а сделать код, который коллега прочитает за 30 секунд и поймёт без объяснений. Плоский, читаемый, безопасный. Каждый `if` должен быть обоснован, каждая ветка — взаимоисключающей там, где это нужно, а каждая фигурная скобка — на месте.

#### Что нужно сделать (пошагово)
1. Создайте новый консольный проект .NET 8 с C# 12. Откройте терминал в папке модуля и выполните команду `dotnet new console -n DeliveryCalc -o DeliveryCalc --framework net8.0`. Эта команда создаст проект с top-level statements и файлом `Program.cs`. Убедитесь, что в `DeliveryCalc.csproj` свойство `LangVersion` не зафиксировано на старой версии — по умолчанию для .NET 8 используется C# 12, но если явно указано `7.3`, удалите эту строку.
2. Откройте `Program.cs` и удалите шаблонный код `Console.WriteLine("Hello, World!");`. Вы будете писать решение с использованием top-level statements, как в примере урока.
3. Объявите модели данных с помощью `record` (C# 12): `Customer(string Name, int Age, bool IsMember, int YearsOfLoyalty)` и `Order(Customer? Buyer, decimal TotalAmount, double WeightKg, bool IsExpress, string Region)`. Регион — строка из множества `"RU"`, `"EU"`, `"US"` или любое другое значение (прочие регионы). Обратите внимание: `Buyer` помечен `?`, потому что заказ может прийти без данных покупателя.
4. Объявите `record DeliveryQuote(decimal Fee, decimal FinalPrice, string Label, List<string> Notes)` — это результат, который возвращает калькулятор. `Notes` — список строковых пояснений (например, `"loyal discount 20%"`, `"express +50%"`), полезных для отладки.
5. Реализуйте метод `public static DeliveryQuote CalculateDelivery(Order? order)`. Внутри метода:
   - Сначала идёт блок guard clauses: проверьте `order is null`, `order.Buyer is null`, `order.TotalAmount < 0m`, `order.WeightKg <= 0`. Для каждого невалидного случая верните `DeliveryQuote` с нулевой стоимостью и пояснительной меткой. Не выбрасывайте исключения — возвращайте «безопасный» результат, чтобы демонстрационный цикл не падал.
   - Затем вычислите базовую ставку доставки по региону через `if/else if/else`: `"RU"` → 300, `"EU"` → 1200, `"US"` → 1500, прочее → 2000. Это классический пример взаимоисключающих веток — обязательно с `else if`, не с независимыми `if`.
   - Вычислите экспресс-надбавку через тернарный оператор: если `IsExpress`, то `baseFee * 0.5m`, иначе `0m`. Это выбор значения — идеальный случай для `?:`.
   - Определите «премиум-клиента» через pattern matching: `order.Buyer is { IsMember: true, YearsOfLoyalty: >= 5 }`. Это заменяет длинную цепочку `order.Buyer != null && order.Buyer.IsMember && order.Buyer.YearsOfLoyalty >= 5`.
   - Определите «лояльного клиента» с коротким замыканием: `order.Buyer.IsMember && order.Buyer.YearsOfLoyalty >= 2`. Скидка лояльности — `baseFee * 0.2m` через тернар.
   - Бесплатная доставка при `TotalAmount >= 5000m`. Если бесплатная доставка — итоговая стоимость доставки `0`, иначе `baseFee + expressSurcharge - loyaltyDiscount`.
   - Защититесь от отрицательной стоимости (`if (fee < 0m) fee = 0m;`) — это страховка на случай, если скидка превысит базу.
   - Сформируйте метку статуса через `if/else if/else`: «Бесплатная», «Премиум», «Лояльный», «Экспресс», «Стандарт» — в этом порядке приоритета.
   - Соберите список `Notes`, добавляя пояснения через отдельные `if` (не взаимоисключающие).
6. Добавьте демонстрационный блок: создайте список заказов через collection expression или `new List<Order?> { ... }`, включающий валидные заказы разных категорий, заказ без покупателя (`Buyer = null`), заказ с отрицательной суммой и `null`-заказ. В цикле `foreach` вызовите `CalculateDelivery` и выведите результат в формате: `Label: fee=..., final=..., notes=[...]`.
7. Соберите и запустите: `dotnet build` (должно быть без ошибок и предупреждений уровня error), затем `dotnet run --project DeliveryCalc`. Зафиксируйте вывод для каждого тестового заказа.
8. Проверьте себя по чек-листу урока: все `if` в фигурных скобках (или однострочные guard clauses с `return`), нет вложенности глубже 2–3 уровней, используется `&&` а не `&`, тернар не вложён глубже одного уровня, применён pattern matching. Запрещено использовать `switch` expression — тема следующего урока, здесь тренируем именно `if/else if/else`.

#### Требования к решению
- Целевая платформа: .NET 8, язык C# 12. Используйте top-level statements, `record`, nullable-аннотации (`Customer?`, `Order?`), collection expressions или инициализаторы коллекций. Код должен компилироваться без ошибок и без предупреждений, связанных с null.
- Все условные конструкции должны следовать лучшим практикам урока: фигурные скобки `{}` обязательны для всех `if/else if/else`, даже однострочных (для guard clauses с немедленным `return` допустим компактный однострочный вид, но при малейшем сомнении ставьте скобки). Ветви `if/else if` должны быть действительно взаимоисключающими там, где этого требует семантика (выбор региона, выбор метки).
- Обязательно применение guard clauses (ранний возврат) для всех невалидных входов — никаких глубоко вложенных `if (order != null) { if (order.Buyer != null) { ... } }`. Основной сценарий должен читаться на верхнем уровне отступа.
- Тернарный оператор применяется только для выбора значения (экспресс-надбавка, скидка, итоговая цена), не вложён глубже одного уровня и не используется для побочных эффектов.
- Pattern matching `is { ... }` применён минимум один раз (определение премиум-клиента). Короткое замыкание `&&` использовано для null-безопасной проверки лояльности. Побитовое `&` / `|` в условиях запрещено.
- Код не должен содержать `TODO`, закомментированных блоков и заглушек. Демонстрационный вывод должен соответствовать расчётам для каждого тестового заказа.

#### Тонкости и подводные камни
- **Висячий else (dangling else).** Если вы пишете вложенные `if` без скобок, `else` привязывается к ближайшему `if` без своего `else`, что может привести к логике, отличной от задуманной. Всегда ставьте `{}` — это снимает неоднозначность и страхует при будущих правках, когда кто-то добавит строку в «однострочный» `if`.
- **Побитовое `&` вместо `&&`.** Классическая ошибка стажёра из контекста: `if (order.Buyer != null & order.Buyer.IsMember)` — побитовое `&` всегда вычисляет оба операнда, поэтому при `Buyer == null` правая часть бросает `NullReferenceException`. Используйте `&&` с коротким замыканием: если левый операнд ложен, правый не вычисляется. То же для `||` против `|`.
- **Тернарный — это выражение, не оператор.** `?:` возвращает значение, поэтому он идеален для инициализации переменной (`var fee = isExpress ? base * 0.5m : 0m;`). Не используйте его для побочных эффектов вроде `cond ? DoX() : DoY()` — это нечитаемо и нарушает ожидания читателя. И никогда не вкладывайте `a ? b ? c : d : e` — разбейте на `if` или промежуточные переменные.
- **Присваивание вместо сравнения.** `if (x = 5)` для `int` — ошибка компиляции (спасает), но для `bool` пройдёт бесшумно. Пишите `==`, а для сравнения с константой ставьте константу слева: `if (5 == x)` — тогда опечатка `if (5 = x)` не скомпилируется.
- **Взаимоисключающие ветки vs независимые `if`.** Выбор региона — взаимоисключающий, нужен `else if`. А сбор `Notes` — независимые условия, нужны отдельные `if` без `else`. Перепутать — значит либо пропустить нужную ветку, либо сработать дважды.
- **Pattern matching `is { ... }` уже включает null-проверку.** `order.Buyer is { IsMember: true }` безопасно вернёт `false` для `null` — не нужно предварять его `!= null`. Это компактнее и читается лучше, чем цепочка `&&`.
- **Глубина вложенности.** Если основной сценарий уходит на 4+ уровня отступа — это сигнал к guard clauses. После всех guard-ов метод должен быть «плоским»: основная логика на одном уровне.
- **Порядок веток.** В `if/else if/else` располагайте ветки от наиболее вероятных/приоритетных к менее вероятным. Для метки статуса важен приоритет: бесплатная доставка важнее премиум-статуса.

#### Критерии приёмки
- [ ] Проект `DeliveryCalc` создан через `dotnet new console` на .NET 8, собирается без ошибок.
- [ ] Использованы `record` для `Customer`, `Order`, `DeliveryQuote` с корректными nullable-аннотациями (`Customer?`, `Order?`).
- [ ] Реализован метод `CalculateDelivery(Order?)` с guard clauses для `null` заказа, `null` покупателя, отрицательной суммы и неположительного веса.
- [ ] Guard clauses используют ранний возврат, без вложенности; невалидные случаи возвращают `DeliveryQuote` с пояснительной меткой, а не выбрасывают исключение.
- [ ] Базовая ставка по региону реализована через `if/else if/else` с взаимоисключающими ветками и финальным `else`.
- [ ] Экспресс-надбавка и скидка лояльности вычислены через тернарный оператор (выбор значения), без вложенности глубже одного уровня.
- [ ] Премиум-клиент определён через pattern matching `is { IsMember: true, YearsOfLoyalty: >= 5 }`.
- [ ] Лояльность проверена с коротким замыканием `&&` (не `&`), null-безопасно.
- [ ] Метка статуса формируется через `if/else if/else` с правильным приоритетом веток.
- [ ] Сбор `Notes` выполнен отдельными независимыми `if` (не `else if`).
- [ ] Все `if/else if/else` обрамлены фигурными скобками (или компактный guard с `return`).
- [ ] Нет вложенности глубже 2–3 уровней; основной сценарий плоский.
- [ ] Нет побитового `&`/`|` в условиях; нет вложенного тернарного; нет `if (x = ...)` присваиваний.
- [ ] Демонстрационный список содержит валидные заказы всех категорий, заказ без покупателя, заказ с отрицательной суммой и `null`-заказ; вывод соответствует расчётам.
- [ ] Код не содержит `TODO`, закомментированных блоков и компилируется без предупреждений уровня error.

#### Подсказки (без прямого ответа)
- Начните метод с четырёх guard clauses подряд — каждый `return` сразу. Не оборачивайте основной сценарий в гигантский `if`.
- Для региона сначала проверьте `"RU"`, потом `"EU"`, потом `"US"`, а всё остальное — в `else`. Подумайте, почему `else` здесь обязателен.
- Чтобы вычислить «премиум», вспомните синтаксис `is { свойство: значение }` из урока — он заменяет три проверки разом и сам по себе null-безопасен.
- Для метки статуса продумайте приоритет: что важнее — бесплатная доставка или премиум? Первое совпадение в `if/else if` выигрывает.
- Если не уверены, нужен ли `else` — задайте вопрос: «Может ли одновременно быть истинно несколько условий?». Если да — независимые `if`; если нет — `else if`.
- Помните: тернар возвращает значение, поэтому его результат можно сразу присвоить `var`. Если хочется вызвать метод как побочный эффект — это сигнал «не используй тернар».

#### Эталонное решение (разбор)
```csharp
// C# 12 / .NET 8 — top-level statements
// Калькулятор стоимости доставки / Delivery cost calculator
// ДЗ M03-L01: if/else, тернарный оператор, guard clauses, pattern matching

using System;
using System.Collections.Generic;

// --- Модель данных / Data model ---
// Buyer помечен nullable: заказ может прийти без покупателя.
// Buyer is nullable: an order may arrive without a customer.
public record Customer(string Name, int Age, bool IsMember, int YearsOfLoyalty);
public record Order(Customer? Buyer, decimal TotalAmount, double WeightKg, bool IsExpress, string Region);
public record DeliveryQuote(decimal Fee, decimal FinalPrice, string Label, List<string> Notes);

// --- Калькулятор / Calculator ---
public static DeliveryQuote CalculateDelivery(Order? order)
{
    // Guard clauses (ранний возврат): отсекаем невалидные случаи сразу.
    // Guard clauses (early return): reject invalid cases up front.
    if (order is null)                                  // guard 1: null заказ / null order
        return new DeliveryQuote(0m, 0m, "Нет заказа / No order", new() { "order is null" });

    if (order.Buyer is null)                            // guard 2: null покупатель / null buyer
        return new DeliveryQuote(0m, 0m, "Нет покупателя / No buyer", new() { "buyer is null" });

    if (order.TotalAmount < 0m)                         // guard 3: отрицательная сумма / negative amount
        return new DeliveryQuote(0m, 0m, "Ошибка / Error", new() { "negative amount" });

    if (order.WeightKg <= 0)                            // guard 4: неположительный вес / non-positive weight
        return new DeliveryQuote(0m, 0m, "Ошибка / Error", new() { "non-positive weight" });

    // После guard-ов основной сценарий плоский — без вложенности.
    // After the guards the main scenario is flat — no nesting.

    // Базовая ставка по региону через if/else if/else (взаимоисключающие ветки + финальный else).
    // Base rate by region via if/else if/else (mutually exclusive branches + final else).
    decimal baseFee;
    if (order.Region == "RU")
        baseFee = 300m;
    else if (order.Region == "EU")
        baseFee = 1200m;
    else if (order.Region == "US")
        baseFee = 1500m;
    else
        baseFee = 2000m;                                // прочие регионы / other regions

    // Экспресс-надбавка через тернар (выбор значения, не побочный эффект).
    // Express surcharge via ternary (value selection, not a side effect).
    var expressSurcharge = order.IsExpress ? baseFee * 0.5m : 0m;

    // Pattern matching: премиум — членство AND стаж >= 5 (заменяет длинную &&-цепочку, null-безопасно).
    // Pattern matching: premium — member AND tenure >= 5 (replaces a long &&-chain, null-safe).
    var isPremium = order.Buyer is { IsMember: true, YearsOfLoyalty: >= 5 };

    // Короткое замыкание &&: если IsMember false, YearsOfLoyalty не вычисляется.
    // Short-circuit &&: if IsMember is false, YearsOfLoyalty is not evaluated.
    var isLoyal = order.Buyer.IsMember && order.Buyer.YearsOfLoyalty >= 2;
    var loyaltyDiscount = isLoyal ? baseFee * 0.2m : 0m;   // тернар для выбора скидки / ternary for discount

    // Бесплатная доставка при крупном заказе (порог 5000).
    // Free shipping for a large order (threshold 5000).
    var freeShipping = order.TotalAmount >= 5000m;

    // Итоговая стоимость с учётом всех правил.
    // Final fee accounting for all rules.
    decimal fee;
    if (freeShipping)
        fee = 0m;
    else
        fee = baseFee + expressSurcharge - loyaltyDiscount;

    // Страховка от отрицательной стоимости (guard).
    // Guard against a negative fee.
    if (fee < 0m)
        fee = 0m;

    // Финальная цена через тернар (выбор значения).
    // Final price via ternary (value selection).
    var finalPrice = freeShipping ? order.TotalAmount : order.TotalAmount + fee;

    // Метка статуса через if/else if/else (приоритет веток важен: первое совпадение выигрывает).
    // Status label via if/else if/else (branch priority matters: first match wins).
    string label;
    if (freeShipping)
        label = "Бесплатная / Free";
    else if (isPremium)
        label = "Премиум / Premium";
    else if (isLoyal)
        label = "Лояльный / Loyal";
    else if (order.IsExpress)
        label = "Экспресс / Express";
    else
        label = "Стандарт / Standard";

    // Сбор заметок — независимые if (НЕ взаимоисключающие), поэтому без else.
    // Notes collection — independent ifs (NOT mutually exclusive), so no else.
    var notes = new List<string>();
    if (isPremium) notes.Add("premium client");
    if (isLoyal && !isPremium) notes.Add("loyal discount 20%");
    if (order.IsExpress) notes.Add("express +50%");
    if (freeShipping) notes.Add("free shipping (amount >= 5000)");

    return new DeliveryQuote(fee, finalPrice, label, notes);
}

// --- Демо / Demo ---
var orders = new List<Order?>
{
    new(new("Анна", 30, true, 6), 6000m, 2.5, false, "RU"),   // премиум + бесплатная
    new(new("Борис", 25, true, 3), 1500m, 1.0, true, "EU"),   // лояльный + экспресс
    new(new("Виктор", 40, false, 0), 800m, 3.0, false, "US"), // стандарт
    new(new("Галина", 35, true, 8), 9000m, 5.0, true, "RU"),  // премиум + экспресс + бесплатная
    new(null, 1000m, 2.0, false, "RU"),                       // нет покупателя / no buyer
    new(new("Дмитрий", 50, true, 1), -100m, 2.0, false, "RU"),// отрицательная сумма / negative amount
    null,                                                      // null заказ / null order
};

foreach (var o in orders)
{
    var q = CalculateDelivery(o);
    Console.WriteLine($"{q.Label}: fee={q.Fee}, final={q.FinalPrice}, notes=[{string.Join(", ", q.Notes)}]");
}

// Ожидаемый вывод / Expected output:
// Бесплатная / Free: fee=0, final=6000, notes=[premium client, free shipping (amount >= 5000)]
// Лояльный / Loyal: fee=1560, final=3060, notes=[loyal discount 20%, express +50%]
// Стандарт / Standard: fee=1500, final=2300, notes=[]
// Бесплатная / Free: fee=0, final=9000, notes=[premium client, express +50%, free shipping (amount >= 5000)]
// Нет покупателя / No buyer: fee=0, final=0, notes=[buyer is null]
// Ошибка / Error: fee=0, final=0, notes=[negative amount]
// Нет заказа / No order: fee=0, final=0, notes=[order is null]
```

Разбор по строкам. Первые четыре `if` — это guard clauses (ранний возврат), ключевая техника урока: вместо вложенного «если всё хорошо — продолжаем» мы отсекаем невалидные случаи сразу и возвращаемся. Заметьте, что каждый guard сам по себе null-безопасен: `order is null` проверяет заказ до обращения к `order.Buyer`, а `order.Buyer is null` проверяется до обращения к свойствам покупателя — именно так short-circuit и порядок проверок защищают от `NullReferenceException`. После guard-ов основной сценарий идёт плоско, на одном уровне отступа, без «лесенки». Блок `if/else if/else` для региона демонстрирует взаимоисключающие ветки с финальным `else` — здесь `else if` обязателен, потому что регион один, и независимые `if` привели бы к перезаписи `baseFee`. Тернарный оператор для `expressSurcharge` и `loyaltyDiscount` — канонический случай: выбор значения в одну строку, без вложенности и без побочных эффектов; он возвращает значение, которое мы присваиваем `var`. Pattern matching `order.Buyer is { IsMember: true, YearsOfLoyalty: >= 5 }` — это звёздный момент урока: одна конструкция заменяет `order.Buyer != null && order.Buyer.IsMember && order.Buyer.YearsOfLoyalty >= 5`, причём `is { ... }` сам по себе возвращает `false` для `null`, так что дополнительная null-проверка не нужна. Строка `isLoyal` использует короткое замыкание `&&`: если `IsMember` ложно, правый операнд не вычисляется — это и оптимизация, и безопасность. Метка статуса через `if/else if/else` показывает приоритет веток: бесплатная доставка проверяется первой, потому что это сильнейший признак, и первое совпадение выигрывает. Сбор `Notes`, напротив, использует независимые `if` без `else`, потому что заметки не взаимоисключающие — несколько могут добавиться одновременно. Финальная цена и надбавки — снова тернар для выбора значения. Страховка `if (fee < 0m) fee = 0m;` — ещё один guard, защищающий от арифметической аномалии. Всё решение иллюстрирует главные принципы урока: всегда скобки, guard clauses вместо вложенности, тернар только для значения, `&&` не `&`, pattern matching вместо длинных цепочек.

#### Задания на углубление (бонус)
1. Добавьте поле `Customer.IsVip` и правило: VIP-клиент получает бесплатную доставку независимо от суммы заказа. Реализуйте через pattern matching `is { IsVip: true }` и скорректируйте приоритет метки статуса.
2. Вынесите ставки по регионам в `Dictionary<string, decimal>` и замените `if/else if/else` на поиск по словарю с дефолтным значением через `TryGetValue` и тернар. Сравните читаемость двух подходов и опишите, когда какой предпочтительнее.
3. Добавьте проверку возраста покупателя: если `Age < 18`, доставка алкоголя запрещена — верните специальную метку «Запрет / Forbidden» через guard clause. Подумайте, как это влияет на порядок guard-ов.
4. Напишите unit-тесты (xUnit) для `CalculateDelivery`, покрывающие все ветки: каждый регион, экспресс/не экспресс, премиум/лояльный/стандарт, все guard-случаи. Убедитесь, что тесты фиксируют порядок приоритета меток.

---

## Statement in English / Постановка на английском

#### Context & motivation
Imagine you are a junior developer on the team of the "Northern Yard" online store. The team is building the order delivery-cost module. Right now that module was written by an intern as a "ladder" of nested `if` four or five levels deep, using bitwise `&` instead of `&&`, omitting braces on one-line branches, and featuring a nested ternary of the form `a ? b ? c : d : e`. The code works, but on code review it is called "conditional spaghetti": hard to read, easy to break, and it has produced two `NullReferenceException` incidents because the right operand of `&` was evaluated even when the left was `null`.

Your task is to rewrite this module from scratch, guided by the best practices from lesson M03-L01. You must demonstrate that you can: reject invalid inputs with guard clauses (early return), use mutually exclusive `if/else if/else` branches where the semantics call for it, apply the ternary operator only for single-line value selection, rely on short-circuit `&&`/`||` for null-safe checks, and replace long `!= null && prop` chains with `is { ... }` pattern matching. This is a realistic task that shows up in production code every day: business logic with discount rules, free-shipping thresholds, and express surcharges.

The goal is not merely "make it work" but to produce code that a colleague can read in thirty seconds and understand without explanation. Flat, readable, safe. Every `if` must be justified, every branch mutually exclusive where needed, and every brace in place.

#### What to do step by step
1. Create a new .NET 8 console project targeting C# 12. Open a terminal in the module folder and run `dotnet new console -n DeliveryCalc -o DeliveryCalc --framework net8.0`. This creates a project with top-level statements and a `Program.cs` file. Verify that `DeliveryCalc.csproj` does not pin `LangVersion` to an old value — for .NET 8 the default is C# 12, but if you see `7.3` explicitly, remove that line.
2. Open `Program.cs` and delete the boilerplate `Console.WriteLine("Hello, World!");`. You will write the solution using top-level statements, exactly as in the lesson example.
3. Declare the data models using `record` (C# 12): `Customer(string Name, int Age, bool IsMember, int YearsOfLoyalty)` and `Order(Customer? Buyer, decimal TotalAmount, double WeightKg, bool IsExpress, string Region)`. The region is a string from the set `"RU"`, `"EU"`, `"US"`, or any other value (other regions). Note that `Buyer` is marked `?` because an order may arrive without customer data.
4. Declare `record DeliveryQuote(decimal Fee, decimal FinalPrice, string Label, List<string> Notes)` — the result the calculator returns. `Notes` is a list of explanatory strings (for example `"loyal discount 20%"`, `"express +50%"`) useful for debugging.
5. Implement the method `public static DeliveryQuote CalculateDelivery(Order? order)`. Inside the method:
   - First comes a block of guard clauses: check `order is null`, `order.Buyer is null`, `order.TotalAmount < 0m`, and `order.WeightKg <= 0`. For each invalid case return a `DeliveryQuote` with a zero fee and an explanatory label. Do not throw exceptions — return a "safe" result so the demo loop does not crash.
   - Then compute the base delivery rate by region using `if/else if/else`: `"RU"` → 300, `"EU"` → 1200, `"US"` → 1500, otherwise → 2000. This is the canonical example of mutually exclusive branches — make sure to use `else if`, not independent `if`s.
   - Compute the express surcharge with the ternary operator: if `IsExpress`, then `baseFee * 0.5m`, otherwise `0m`. This is value selection — the ideal case for `?:`.
   - Determine the "premium" client via pattern matching: `order.Buyer is { IsMember: true, YearsOfLoyalty: >= 5 }`. This replaces the long chain `order.Buyer != null && order.Buyer.IsMember && order.Buyer.YearsOfLoyalty >= 5`.
   - Determine the "loyal" client with short-circuit: `order.Buyer.IsMember && order.Buyer.YearsOfLoyalty >= 2`. The loyalty discount is `baseFee * 0.2m` via the ternary.
   - Free shipping when `TotalAmount >= 5000m`. If free shipping applies, the final fee is `0`; otherwise it is `baseFee + expressSurcharge - loyaltyDiscount`.
   - Guard against a negative fee (`if (fee < 0m) fee = 0m;`) — a safety net in case a discount exceeds the base.
   - Build the status label via `if/else if/else` in this priority order: "Free", "Premium", "Loyal", "Express", "Standard".
   - Collect the `Notes` list, appending explanations through separate independent `if`s (not mutually exclusive).
6. Add a demo block: build a list of orders via a collection expression or `new List<Order?> { ... }`, including valid orders of every category, an order without a buyer (`Buyer = null`), an order with a negative amount, and a `null` order. In a `foreach` loop call `CalculateDelivery` and print the result in the form: `Label: fee=..., final=..., notes=[...]`.
7. Build and run: `dotnet build` (must be free of errors and error-level warnings), then `dotnet run --project DeliveryCalc`. Record the output for each test order.
8. Check yourself against the lesson checklist: every `if` is braced (or a compact guard with `return`), no nesting deeper than two or three levels, `&&` is used instead of `&`, the ternary is not nested beyond one level, and pattern matching is applied. The `switch` expression is forbidden here — that is the next lesson; this homework trains `if/else if/else` specifically.

#### Requirements
- Target platform: .NET 8, language C# 12. Use top-level statements, `record`, nullable annotations (`Customer?`, `Order?`), and collection expressions or collection initializers. The code must compile without errors and without null-related warnings.
- All conditional constructs must follow the lesson's best practices: braces `{}` are mandatory for every `if/else if/else`, even one-line ones (for guard clauses with an immediate `return` a compact one-line form is acceptable, but when in doubt, add braces). `if/else if` branches must be genuinely mutually exclusive where the semantics require it (region choice, label choice).
- Guard clauses (early return) are mandatory for all invalid inputs — no deeply nested `if (order != null) { if (order.Buyer != null) { ... } }`. The main scenario must read at the top indentation level.
- The ternary operator is used only for value selection (express surcharge, discount, final price), is not nested beyond one level, and is never used for side effects.
- Pattern matching `is { ... }` is applied at least once (premium-client detection). Short-circuit `&&` is used for the null-safe loyalty check. Bitwise `&` / `|` in conditions is forbidden.
- The code must not contain `TODO`, commented-out blocks, or stubs. The demo output must match the calculations for each test order.

#### Pitfalls
- **Dangling else.** When you write nested `if` without braces, an `else` binds to the nearest `if` that lacks its own `else`, which can produce logic different from what you intended. Always use `{}` — it removes ambiguity and protects you during future edits when someone adds a line to a "one-line" `if`.
- **Bitwise `&` instead of `&&`.** The intern's classic mistake: `if (order.Buyer != null & order.Buyer.IsMember)` — bitwise `&` always evaluates both operands, so when `Buyer` is `null` the right side throws `NullReferenceException`. Use short-circuit `&&`: if the left operand is false, the right is never evaluated. The same applies to `||` versus `|`.
- **The ternary is an expression, not a statement.** `?:` returns a value, so it is perfect for initializing a variable (`var fee = isExpress ? base * 0.5m : 0m;`). Do not use it for side effects like `cond ? DoX() : DoY()` — that is unreadable and breaks the reader's expectations. And never nest `a ? b ? c : d : e` — break it into `if` or intermediate variables.
- **Assignment instead of comparison.** `if (x = 5)` for `int` is a compile error (which saves you), but for `bool` it slips through silently. Use `==`, and when comparing against a constant put the constant first: `if (5 == x)` — then a typo like `if (5 = x)` will not compile.
- **Mutually exclusive branches versus independent `if`s.** Region selection is mutually exclusive — it needs `else if`. Collecting `Notes` uses independent conditions — it needs separate `if` without `else`. Confusing them means either skipping a needed branch or firing twice.
- **Pattern matching `is { ... }` already includes a null check.** `order.Buyer is { IsMember: true }` safely returns `false` for `null` — you do not need to precede it with `!= null`. It is more compact and reads better than an `&&` chain.
- **Nesting depth.** If the main scenario sinks to four or more indentation levels, that is a signal to use guard clauses. After all guards the method should be "flat": the main logic on a single level.
- **Branch ordering.** In `if/else if/else` order branches from most likely / highest priority to least likely. For the status label, priority matters: free shipping outranks premium status, because the first match in `if/else if` wins.

#### Acceptance criteria
- [ ] The `DeliveryCalc` project is created via `dotnet new console` on .NET 8 and builds without errors.
- [ ] `record` is used for `Customer`, `Order`, and `DeliveryQuote` with correct nullable annotations (`Customer?`, `Order?`).
- [ ] The `CalculateDelivery(Order?)` method is implemented with guard clauses for a `null` order, a `null` buyer, a negative amount, and a non-positive weight.
- [ ] Guard clauses use early return without nesting; invalid cases return a `DeliveryQuote` with an explanatory label rather than throwing.
- [ ] The base rate by region is implemented with `if/else if/else` using mutually exclusive branches and a final `else`.
- [ ] The express surcharge and loyalty discount are computed with the ternary operator (value selection), not nested beyond one level.
- [ ] The premium client is detected via the `is { IsMember: true, YearsOfLoyalty: >= 5 }` pattern match.
- [ ] Loyalty is checked with short-circuit `&&` (not `&`), null-safe.
- [ ] The status label is built with `if/else if/else` using correct branch priority.
- [ ] The `Notes` collection uses separate independent `if`s (not `else if`).
- [ ] Every `if/else if/else` is wrapped in braces (or is a compact guard with `return`).
- [ ] No nesting deeper than two or three levels; the main scenario is flat.
- [ ] No bitwise `&`/`|` in conditions; no nested ternary; no `if (x = ...)` assignments.
- [ ] The demo list contains valid orders of every category, an order without a buyer, an order with a negative amount, and a `null` order; the output matches the calculations.
- [ ] The code contains no `TODO`, no commented-out blocks, and compiles without error-level warnings.

#### Hints (no direct answer)
- Start the method with four guard clauses in a row — each one `return`s immediately. Do not wrap the main scenario in a giant `if`.
- For the region, check `"RU"` first, then `"EU"`, then `"US"`, and put everything else in `else`. Think about why `else` is mandatory here.
- To compute "premium", recall the `is { property: value }` syntax from the lesson — it replaces three checks at once and is null-safe by itself.
- For the status label, think about priority: which is more important, free shipping or premium? The first match in `if/else if` wins.
- When unsure whether `else` is needed, ask: "Can several conditions be true at the same time?" If yes — independent `if`s; if no — `else if`.
- Remember that the ternary returns a value, so its result can be assigned directly to `var`. If you are tempted to call a method as a side effect — that is the signal "do not use the ternary."

#### Reference solution walk-through
```csharp
// C# 12 / .NET 8 — top-level statements
// Delivery cost calculator
// Homework M03-L01: if/else, ternary operator, guard clauses, pattern matching

using System;
using System.Collections.Generic;

// --- Data model ---
// Buyer is nullable: an order may arrive without a customer.
public record Customer(string Name, int Age, bool IsMember, int YearsOfLoyalty);
public record Order(Customer? Buyer, decimal TotalAmount, double WeightKg, bool IsExpress, string Region);
public record DeliveryQuote(decimal Fee, decimal FinalPrice, string Label, List<string> Notes);

// --- Calculator ---
public static DeliveryQuote CalculateDelivery(Order? order)
{
    // Guard clauses (early return): reject invalid cases up front.
    if (order is null)                                  // guard 1: null order
        return new DeliveryQuote(0m, 0m, "No order", new() { "order is null" });

    if (order.Buyer is null)                            // guard 2: null buyer
        return new DeliveryQuote(0m, 0m, "No buyer", new() { "buyer is null" });

    if (order.TotalAmount < 0m)                         // guard 3: negative amount
        return new DeliveryQuote(0m, 0m, "Error", new() { "negative amount" });

    if (order.WeightKg <= 0)                            // guard 4: non-positive weight
        return new DeliveryQuote(0m, 0m, "Error", new() { "non-positive weight" });

    // After the guards the main scenario is flat — no nesting.

    // Base rate by region via if/else if/else (mutually exclusive branches + final else).
    decimal baseFee;
    if (order.Region == "RU")
        baseFee = 300m;
    else if (order.Region == "EU")
        baseFee = 1200m;
    else if (order.Region == "US")
        baseFee = 1500m;
    else
        baseFee = 2000m;                                // other regions

    // Express surcharge via ternary (value selection, not a side effect).
    var expressSurcharge = order.IsExpress ? baseFee * 0.5m : 0m;

    // Pattern matching: premium — member AND tenure >= 5 (replaces a long &&-chain, null-safe).
    var isPremium = order.Buyer is { IsMember: true, YearsOfLoyalty: >= 5 };

    // Short-circuit &&: if IsMember is false, YearsOfLoyalty is not evaluated.
    var isLoyal = order.Buyer.IsMember && order.Buyer.YearsOfLoyalty >= 2;
    var loyaltyDiscount = isLoyal ? baseFee * 0.2m : 0m;   // ternary for discount

    // Free shipping for a large order (threshold 5000).
    var freeShipping = order.TotalAmount >= 5000m;

    // Final fee accounting for all rules.
    decimal fee;
    if (freeShipping)
        fee = 0m;
    else
        fee = baseFee + expressSurcharge - loyaltyDiscount;

    // Guard against a negative fee.
    if (fee < 0m)
        fee = 0m;

    // Final price via ternary (value selection).
    var finalPrice = freeShipping ? order.TotalAmount : order.TotalAmount + fee;

    // Status label via if/else if/else (branch priority matters: first match wins).
    string label;
    if (freeShipping)
        label = "Free";
    else if (isPremium)
        label = "Premium";
    else if (isLoyal)
        label = "Loyal";
    else if (order.IsExpress)
        label = "Express";
    else
        label = "Standard";

    // Notes collection — independent ifs (NOT mutually exclusive), so no else.
    var notes = new List<string>();
    if (isPremium) notes.Add("premium client");
    if (isLoyal && !isPremium) notes.Add("loyal discount 20%");
    if (order.IsExpress) notes.Add("express +50%");
    if (freeShipping) notes.Add("free shipping (amount >= 5000)");

    return new DeliveryQuote(fee, finalPrice, label, notes);
}

// --- Demo ---
var orders = new List<Order?>
{
    new(new("Anna", 30, true, 6), 6000m, 2.5, false, "RU"),   // premium + free
    new(new("Boris", 25, true, 3), 1500m, 1.0, true, "EU"),   // loyal + express
    new(new("Victor", 40, false, 0), 800m, 3.0, false, "US"), // standard
    new(new("Galina", 35, true, 8), 9000m, 5.0, true, "RU"),  // premium + express + free
    new(null, 1000m, 2.0, false, "RU"),                       // no buyer
    new(new("Dmitry", 50, true, 1), -100m, 2.0, false, "RU"), // negative amount
    null,                                                      // null order
};

foreach (var o in orders)
{
    var q = CalculateDelivery(o);
    Console.WriteLine($"{q.Label}: fee={q.Fee}, final={q.FinalPrice}, notes=[{string.Join(", ", q.Notes)}]");
}

// Expected output:
// Free: fee=0, final=6000, notes=[premium client, free shipping (amount >= 5000)]
// Loyal: fee=1560, final=3060, notes=[loyal discount 20%, express +50%]
// Standard: fee=1500, final=2300, notes=[]
// Free: fee=0, final=9000, notes=[premium client, express +50%, free shipping (amount >= 5000)]
// No buyer: fee=0, final=0, notes=[buyer is null]
// Error: fee=0, final=0, notes=[negative amount]
// No order: fee=0, final=0, notes=[order is null]
```

Line-by-line walk-through. The first four `if` statements are guard clauses (early return), the lesson's key technique: instead of a nested "if everything is fine, we continue," we reject invalid cases up front and return. Note that each guard is null-safe by itself: `order is null` checks the order before touching `order.Buyer`, and `order.Buyer is null` is checked before touching any buyer property — this is exactly how short-circuit behavior and the order of checks protect against `NullReferenceException`. After the guards the main scenario runs flat, at a single indentation level, with no "ladder." The `if/else if/else` block for the region demonstrates mutually exclusive branches with a final `else` — here `else if` is mandatory because there is exactly one region, and independent `if`s would overwrite `baseFee`. The ternary operator for `expressSurcharge` and `loyaltyDiscount` is the canonical case: single-line value selection, no nesting, no side effects; it returns a value that we assign to `var`. The pattern match `order.Buyer is { IsMember: true, YearsOfLoyalty: >= 5 }` is the star moment of the lesson: one construct replaces `order.Buyer != null && order.Buyer.IsMember && order.Buyer.YearsOfLoyalty >= 5`, and `is { ... }` returns `false` for `null` on its own, so an extra null check is unnecessary. The `isLoyal` line uses short-circuit `&&`: if `IsMember` is false, the right operand is never evaluated — both an optimization and a safety guard. The status label via `if/else if/else` shows branch priority: free shipping is checked first because it is the strongest signal, and the first match wins. The `Notes` collection, by contrast, uses independent `if`s without `else`, because notes are not mutually exclusive — several can be added at once. The final price and the surcharges are again ternaries for value selection. The safety net `if (fee < 0m) fee = 0m;` is another guard, protecting against an arithmetic anomaly. The whole solution illustrates the lesson's main principles: always braces, guard clauses over nesting, the ternary only for values, `&&` not `&`, and pattern matching over long chains.

#### Going deeper (bonus)
1. Add a `Customer.IsVip` field and a rule: a VIP client gets free shipping regardless of the order amount. Implement it via the `is { IsVip: true }` pattern match and adjust the status-label priority.
2. Move the regional rates into a `Dictionary<string, decimal>` and replace the `if/else if/else` with a dictionary lookup using `TryGetValue` and the ternary for a default value. Compare the readability of the two approaches and describe when each is preferable.
3. Add a buyer-age check: if `Age < 18`, alcohol delivery is forbidden — return a special "Forbidden" label via a guard clause. Think about how this affects the order of guards.
4. Write xUnit unit tests for `CalculateDelivery` covering every branch: each region, express/non-express, premium/loyal/standard, and all guard cases. Make sure the tests pin down the label-priority order.

---

#### Чек-лист сдачи / Submission checklist
- [ ] (RU) Проект `DeliveryCalc` создан на .NET 8 / C# 12 и собирается без ошибок.
- [ ] (RU) Использованы `record` с nullable-аннотациями `Customer?`, `Order?`.
- [ ] (RU) Реализованы guard clauses для всех невалидных входов через ранний возврат.
- [ ] (RU) Регионы реализованы через `if/else if/else` с финальным `else`.
- [ ] (RU) Тернарный оператор применён только для выбора значения, без вложенности.
- [ ] (RU) Pattern matching `is { ... }` использован для премиум-клиента.
- [ ] (RU) Лояльность проверена коротким замыканием `&&`, не `&`.
- [ ] (RU) Все `if/else if/else` в фигурных скобках; нет вложенности глубже 2–3 уровней.
- [ ] (RU) Демонстрационный вывод соответствует расчётам для всех тестовых заказов.
- [ ] (EN) The `DeliveryCalc` project targets .NET 8 / C# 12 and builds without errors.
- [ ] (EN) `record` types use nullable annotations `Customer?`, `Order?`.
- [ ] (EN) Guard clauses cover every invalid input via early return.
- [ ] (EN) Regions are implemented with `if/else if/else` and a final `else`.
- [ ] (EN) The ternary operator is used only for value selection, without nesting.
- [ ] (EN) `is { ... }` pattern matching is used for the premium client.
- [ ] (EN) Loyalty is checked with short-circuit `&&`, not `&`.
- [ ] (EN) Every `if/else if/else` is braced; no nesting deeper than two or three levels.
- [ ] (EN) The demo output matches the calculations for all test orders.

#### Ресурсы / Resources
- Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/language-reference/statements/selection-statements — Selection statements (if, switch) / Операторы выбора (if, switch)
- Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/language-reference/operators/conditional-operator — Conditional operator ?: / Условный оператор ?:
- Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/language-reference/operators/patterns — Patterns and pattern matching / Шаблоны и сопоставление шаблонов
- Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/language-reference/operators/boolean-logical-operators — Boolean logical operators (`&&`, `||`, `!`, `&`, `|`) / Логические операторы

---
[← К уроку M03-L01](lesson-M03-L01-if-ternary.md) | [⬆ К модулю M03](../README.md) | [Следующее ДЗ →](homework-M03-L02-switch-patterns.md)
---
